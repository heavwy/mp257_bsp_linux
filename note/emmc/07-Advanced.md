# 07. 高级特性：CQHCI 与 Tuning

> 本文是系列第 7 篇，覆盖两个独立但都属于"高级特性"的 MMC 子系统主题。
>
> **前置：** [06-IO-Path.md](06-IO-Path.md) — 传统 IO 路径
> **下一篇：** [08-Interview.md](08-Interview.md)
>
> **字数：** 中文字数 9,798 + 英文单词 4,704 ≈ **14,502 字**（含代码段），**行数：1,108**
>
> **建议阅读时间：** 50–70 分钟

---

## Part 1：CQHCI — 硬件命令队列

---

### 1.1 传统路径的瓶颈

回顾 06-IO-Path 中传统 IO 路径的核心 loop：

```
CPU 准备 cmd/data 描述符
    → 写 MMCIDATACTRL (启动数据路径)
    → 写 MMCICMD  (启动命令)
    → 等待 命令完成中断 (CC)
    → 等待 数据完成中断 (DATAEND)
    → 检查错误
    → 重复下一笔
```

每一笔 IO 都需要 CPU 逐条发起命令、等待完成、再发起下一条。eMMC 和 host 之间没有任何排队机制——一笔请求必须完全结束后才能开始下一笔。

对于 HS200/HS400 这类高速模式，**命令和数据的交互开销** 越来越成为瓶颈。如果 host 能同时下发多笔请求让 eMMC 自己调度执行，CPU 就可以从逐条驱动的模式中解放出来。

> **CQHCI（Command Queue Host Controller Interface）** 正是为了解决这个问题而生的硬件卸载方案。它定义了一套"门铃（doorbell）模型"，host 把多笔请求的描述符一次性写入 DRAM，然后按门铃通知 eMMC，eMMC 自主调度执行。

---

### 1.2 CQHCI 的核心思想：门铃模型

CQHCI 的工作流程可以用一句话概括：

> **CPU 准备描述符 → 按门铃 → eMMC 自己干活 → 完成通知**

具体来说：

1. **准备阶段（CPU 负责）**：在 DRAM 中填充 Task Descriptor 和 Transfer Descriptor。Task Descriptor（16 字节）编码命令参数——相当于传统路径的 CMD23（块计数）+ CMD18/CMD25（读/写）的合并。Transfer Descriptor（8 字节）存放数据 buffer 物理地址——相当于 DMA 描述符。每个排队中的 IO 请求对应一个"slot"（最多 32 个），每个 slot 有自己的 Task Descriptor 和可选的 Transfer Descriptor。
2. **提交阶段（CPU 负责）**：CPU 写 `CQHCI_TDBR`（Task Doorbell Register）寄存器，将要提交的 slot 号对应的 bit 置 1。一个 4 字节 MMIO 写操作即完成提交流程。

3. **执行阶段（eMMC 自主）**：eMMC 硬件轮询检测到 `CQHCI_TDBR` 中某 bit 被置位后，通过 DMA 从 DRAM 读取对应的 Task Descriptor，解析 opcode、LBA、block count 等参数，在 MMC 总线上自主发起命令并完成数据传输。此过程无需 CPU 介入。

4. **完成阶段（eMMC 通知）**：eMMC 在请求完成后将完成状态写入 `CQHCI_TCN`（Task Completion Notification）寄存器，并置位 `CQHCI_IS.TCC` 中断位。CPU 在中断处理中读取 `CQHCI_TCN` 获取完成位图，遍历所有完成的 slot，调用 `mmc_cqe_request_done()` 通知 MMC core 层释放请求。

```
传统路径:   CPU → CMD23（块计数）→ 完成中断 → CMD18/CMD25（读/写）→ 完成中断 → 数据结束中断 → CPU 处理下一笔

CQHCI:     CPU → 写 DRAM 描述符 → 写 CQHCI_TDBR（一次性门铃）
                  ┌───────────────────────────────────┐
                  │  eMMC 自主解析描述符、发命令、传数据  │
                  │  多笔请求乱序执行，批量完成通知       │
                  └───────────────────────────────────┘
                  中断（CQHCI_IS.TCC）→ CPU 遍历 CQHCI_TCN 完成位图
```

---

### 1.3 关键数据结构和寄存器

#### Task Descriptor（128 位）

Task Descriptor 描述一笔请求的参数。它位于 DRAM 中，由 CPU 填充，eMMC 硬件读取。

```
位域布局 — 字 0 [63:0]（来源: cqhci.h，cqhci-core.c）:

数据命令（slot 0-30, ACT=0x5）:
  [0]     VALID         — 描述符有效标志
  [1]     END           — 队列结束标志
  [2]     INT           — 完成后触发中断
  [5:3]   ACT           — 操作类型 (3-bit, 0x5=数据命令/DCMD)
  [6]     FORCED_PROG   — 强制进度报告
  [10:7]  CONTEXT       — 上下文标识
  [11]    DATA_TAG      — 数据标签
  [12]    DATA_DIR      — 方向 (0=写, 1=读)
  [13]    PRIORITY      — 优先级
  [14]    QBAR          — 队列屏障
  [15]    REL_WRITE     — 可靠写入
  [31:16] BLK_COUNT     — 块数量 (对应 CMD23 的块计数)
  [63:32] BLK_ADDR      — 起始块地址 LBA (低 32 位)

DCMD（slot 31, ACT=0x5，无数据传输）共用字 0 的低 16 位，但 [31:16] 含义不同:
  [21:16] CMD_INDEX     — 命令索引 (如 CMD6=6, CMD13=13)
  [22]    CMD_TIMING    — 0=R1B(有busy), 1=普通
  [24:23] RESP_TYPE     — 响应类型 (0x2=R1, 0x3=R1B)

字 1 [63:0]（128-bit 模式下）:
  [63:0]  加密/密钥上下文（cqhci_crypto_prep_task_desc）
```

各字段详细说明：

- **VALID (bit 0)** —— CPU 填充描述符后必须置 1，表示这个描述符可供 eMMC 硬件读取。硬件处理完该 slot 后会将此位清零。一个典型 bug 是 CPU 写了描述符但忘记置 VALID，eMMC 检测到此位为 0 直接跳过，导致请求永远无法完成。

- **END (bit 1)** —— 标记这是队列中的最后一个描述符。但在 Linux CQHCI 驱动中，每个 request 独立提交到单独的 slot（事务性提交），不是批量提交一个链表，所以每次都置 1。

- **INT (bit 2)** —— 置 1 时，该请求完成后触发中断；置 0 时不触发，需要靠 interrupt coalescing（多个请求合并一次中断）或 poll 来获取完成通知。Linux CQHCI 驱动将此位固定置 1，每个请求都产生中断。

- **ACT (bits 5:3, 3-bit)** —— 操作类型。`0x5` 表示数据命令或 DCMD，`0x4` 表示 Transfer Descriptor（见下文）。注意区分：Task Desc 和 Transfer Desc 各有一个 ACT 字段，但值不同——Task Desc 用 `0x5`，Transfer Desc 用 `0x4`。

- **FORCED_PROG (bit 6)** —— 强制进度报告。置 1 时，eMMC 在执行命令过程中必须返回进度（如已写入百分比）。该位依赖 eMMC 固件支持，实际驱动很少使用，固定为 0。

- **CONTEXT (bits 10:7, 4-bit)** —— 透传字段，CQHCI 硬件不解析这 4 位，只原样传递。驱动开发者可用来携带调试信息、标记请求来源等。Linux 驱动没有使用，固定为 0。

- **DATA_TAG (bit 11)** —— 数据标签。置 1 表示这批数据是预取友好的（如顺序预读），eMMC 可以据此决定是否将数据放入内部缓存。Linux 块层下发 `REQ_FUA` 或 `REQ_PREFLUSH` 时可能涉及此位。

- **DATA_DIR (bit 12)** —— 数据传输方向。0 = 写（host → eMMC，对应 CMD25），1 = 读（eMMC → host，对应 CMD18）。此位直接影响 Transfer Descriptor 中 DMA 映射的方向（`DMA_TO_DEVICE` / `DMA_FROM_DEVICE`）。

- **PRIORITY (bit 13)** —— 优先级。置 1 表示高优先级，eMMC 可优先调度此请求。但 eMMC 调度策略是设备固件决定的，此位仅作为提示，实际效果取决于 eMMC 型号。

- **QBAR (bit 14)** —— 队列屏障。置 1 时，此请求之前所有已排队但未执行的请求必须全部完成后才能执行此请求。用于保证写顺序（如文件系统 journal 提交时，必须先写 journal 再写数据）。类似 Linux 块层的 `REQ_BARRIER`。

- **REL_WRITE (bit 15)** —— 可靠写入，eMMC 5.0+ 定义的原子写保证。置 1 时 eMMC 保证这次写入原子性：要么完整写入，要么完全不写，不会出现写入一半断电导致的数据损坏（torn write）。

  实现方式：eMMC 内部 FTL（Flash Translation Layer）对置了 REL_WRITE 的写入采用"写前备份 + 原子切换"策略——先把数据写入一个临时物理块，更新 FTL 映射表（这是原子操作）将 LBA 指向新块，再释放旧块。如果映射更新前断电，恢复后 LBA 仍指向旧数据；更新后断电，新数据已生效。代价是每次可靠写入至少多一次物理块写入，性能下降明显。

  上层与 `REQ_FUA` 配合使用的场景：`REQ_FUA` 保证数据真正落到 NAND（不留在 eMMC 内部缓存），`REL_WRITE` 保证落盘后数据的完整性。文件系统 journal 提交时通常同时携带这两个标志，确保日志块既落盘又防 torn write。

- **BLK_COUNT (bits 31:16, 16-bit)** —— 块数量，最大 65535 块（即 32MB，按 512 字节/块）。**这就是传统路径中 CMD23 的块计数值**，直接被编码进描述符，所以 CQHCI 模式下不需要在 MMC 总线上单独发送 CMD23。`cqhci_prep_task_desc()` 中写此字段的值来自 `mrq->data->blocks`。

- **BLK_ADDR (bits 63:32, 32-bit)** —— 起始 LBA（扇区地址）。eMMC 只支持 32-bit LBA（不是 48-bit LBA，那是 SD 卡的协议），所以描述符中只分配了 32 位。32 位 LBA 最大可寻址 2TB（2^32 × 512 字节），远超 eMMC 实际容量。值来自 `mrq->data->blk_addr`。

**DCMD 特殊字段**（仅当该 slot 用作 DCMD 时有效，数据命令忽略这些位）：

- **CMD_INDEX (bits 21:16, 6-bit)** —— MMC 命令号。存放具体命令编号，如 CMD6 存 6、CMD13 存 13、CMD5（SLEEP/AWAKE）存 5。eMMC 硬件读取此字段后在总线上发出对应命令，不需要 Transfer Descriptor。

- **CMD_TIMING (bit 22)** —— 命令时序，控制 host 侧是否等待 DAT0 busy 信号。0 = 有 busy 等待（CMD6/CMD5 这类长耗时命令），eMMC 执行完后拉低 DAT0 表示忙，host 需等 DAT0 释放；1 = 普通（无 busy，如 CMD13 查询状态）。

- **RESP_TYPE (bits 24:23, 2-bit)** —— 响应类型。0x2 = R1（48 位 CRC 响应，无 busy），0x3 = R1B（R1 + DAT0 busy 信号）。

  这两个字段在信息上重叠（R1B 总是有 busy），但分工不同：RESP_TYPE 告诉 eMMC 以什么格式返回响应数据，CMD_TIMING 告诉 host 是否需要等 DAT0 释放才认为命令结束。代码中两者同步设定——R1B 配 `RESP_TYPE=0x3 + CMD_TIMING=0`，R1 配 `RESP_TYPE=0x2 + CMD_TIMING=1`，无响应的情况（如 CMD0 复位）配 `RESP_TYPE=0x0 + CMD_TIMING=1`。

**字 1（128-bit 描述符模式）**：
字 1 仅在配置寄存器启用了 128-bit 描述符时才存在（`CQHCI_CFG` 的 `CQHCI_TASK_DESC_SZ` 位）。通常用于存放加密/密钥上下文（`cqhci_crypto_prep_task_desc`），非加密场景此字无用。

> **关键设计思路**：数据命令的 BLK_COUNT 替代了 CMD23 的角色，DCMD 的 CMD_INDEX 替代了传统 CMD 命令号的角色。**整个 CQHCI 模式下，MMC 总线上看不到 CMD23**，块计数被"消化"进了描述符。

#### Transfer Descriptor（数据地址 + 长度）

Transfer Descriptor 描述数据 buffer 的物理地址和长度。由 CPU 填充，eMMC 的 DMA 引擎读取。

```
位域布局（来源: cqhci-core.c cqhci_set_tran_desc）:

属性字 (32-bit，描述符起始 4 字节):
  [0]     VALID         — 有效标志
  [1]     END           — 最后一个描述符（SG 链表结束）
  [2]     INT           — 完成后触发中断
  [5:3]   ACT           — 操作类型 (3-bit, 0x4=数据传输)
  [31:16] DAT_LENGTH    — 数据长度 (字节)

地址字 (描述符偏移 4 字节，dma64 为 8 字节，否则 4 字节):
          DAT_ADDR      — 数据 buffer 物理地址
```

各字段说明：

- **VALID (bit 0)** —— 同 Task Desc，置 1 表示该描述符有效。每个 SG 段对应一个 Transfer Desc，硬件依次处理，遇到 VALID=0 停止。

- **END (bit 1)** —— 标记这是 SG 链表的最后一个描述符。CPU 遍历 SG 表时，最后一个元素设 END=1，前面的 SG 元素设 END=0。硬件处理到 END=1 的描述符后停止读取后续描述符。

- **INT (bit 2)** —— 此 Transfer Desc 处理完后是否触发中断。`cqhci_set_tran_desc()` 中固定置 0，因为 CQHCI 的完成通知走的是 Task Desc 的 INT 位 + TCC 中断机制，不需要每个 SG 段都产生中断。

- **ACT (bits 5:3, 3-bit)** —— 操作类型。Task Desc 的 ACT=0x5 表示"这是一笔数据命令"，Transfer Desc 的 ACT=0x4 表示"这是一段数据传输描述"。两者分工明确：Task Desc 描述命令参数，Transfer Desc 描述数据位置。

- **DAT_LENGTH (bits 31:16, 16-bit)** —— 当前 SG 段的数据长度，单位字节。最大值 65535 字节（64KB - 1）。如果一笔 IO 的数据 buffer 使用了多个 SG 段，每个段有自己独立的 DAT_LENGTH。值来自 `sg_dma_len(sg)`。

- **DAT_ADDR（地址字）** —— 数据 buffer 的物理地址。如果 host 控制器支持 64 位 DMA（`cq_host->dma64`），地址字为 8 字节，否则为 4 字节。值来自 `sg_dma_address(sg)`。这个地址必须是物理地址（不是虚拟地址），因为 eMMC 的 DMA 引擎直接寻址系统总线。

**与 Task Desc 的关系**：一笔数据请求的完整信息分布在一个 Task Desc + 一到多个 Transfer Desc 中。Task Desc 说"发什么命令"（CMD18/CMD25, LBA, 块数），Transfer Desc 说"数据放哪"（物理地址、长度）。对于 DCMD，只有 Task Desc，没有 Transfer Desc。

#### DCMD（Direct Command）

CQHCI 有一个特殊的 slot——**slot 31（DCMD_SLOT）**。这个 slot 用于发送**没有数据传输**的命令（如 CMD6 切换、CMD13 查询状态）。

DCMD 的 Task Descriptor 中 `ACT=0x5`，不需要 Transfer Descriptor。它的命令参数通过 Task Descriptor 的 `CMD_INDEX` 字段指定。

#### 关键寄存器（CQHCI MMIO 空间）

来源：`drivers/mmc/host/cqhci.h` 第 17–114 行。

| 寄存器 | 偏移 | 功能 |
|--------|------|------|
| CQHCI_VER | 0x00 | 版本寄存器（Major[11:8], Minor1[7:4], Minor2[3:0]） |
| CQHCI_CAP | 0x04 | 能力寄存器（加密支持、ITCF 倍率） |
| CQHCI_CFG | 0x08 | 配置寄存器（使能、DCMD 支持、Task Desc 大小） |
| CQHCI_CTL | 0x0C | 控制寄存器（暂停 HALT、清除全部任务） |
| CQHCI_IS | 0x10 | **中断状态** — TCC(完成)、RED(可恢复错误)、GCE(加密错误) |
| CQHCI_TDBR | 0x28 | **Doorbell** — 写 bit X 表示提交 slot X |
| CQHCI_TCN | 0x2C | **完成通知** — 哪些 slot 完成了 |
| CQHCI_TERRI | 0x54 | 错误详情 — 哪个 tag 出错（TERRI_C_TASK[12:8]），什么错误 |

> **注意区分：** `CQHCI_IS` 在 0x10 而**不是** 0x30。0x30 是 `CQHCI_DQS`（设备队列状态寄存器），常用于 poll 等待队列空闲。用错偏移是 CQHCI 驱动移植中最隐蔽的错误之一。

---


理解了数据结构后，回到 1.2 节的门铃模型，看两个完整的硬件操作过程。

#### 情景 1：读请求（slot 3，起始 LBA 0x8000，连续 8 块）

**① 准备** —— CPU 在 DRAM 中 slot 3 的位置写入 16 字节 Task Descriptor，打包以下参数：
- `VALID=1`：描述符有效
- `ACT=0x5`：操作类型为数据命令（区别于后面讲的 Transfer Desc 的 `ACT=0x4`）
- `BLK_ADDR=0x8000`：起始 LBA 为 0x8000（第 32768 个 512 字节扇区，即 16MB 偏移处）
- `BLK_COUNT=8`：连续读 8 个块（8 × 512 字节 = 4KB 数据）
- `DATA_DIR=1`：方向为读
- `QBAR=0, PRIORITY=0`：不设屏障，普通优先级

紧接着在后面写入 8 字节 Transfer Descriptor：`DAT_ADDR=0x48000000`（数据 buffer 在系统内存 0x48000000 处）、`DAT_LENGTH=4096`（buffer 容量 4KB）。此时两个描述符都在 DRAM 中，eMMC 尚未感知。

**② 门铃** —— CPU 执行 `writel(BIT(3), CQHCI_TDBR)`，将 doorbell 寄存器 bit 3 置 1。一次 4 字节 MMIO 写后，CQHCI 提交流程的 CPU 投入到此结束。

**③ 执行** —— eMMC 检测到 doorbell bit 3 被置位，通过 DMA 从 DRAM 读取 Task Descriptor，解析出"CMD23(块数=8) + CMD18(读, LBA=0x8000)"。注意这两条命令在传统路径中需 CPU 逐条发送，现在编码在 16 字节描述符中由硬件自行解析执行。eMMC 向 NAND Flash 发读命令，数据从 Flash 读到内部 SRAM，通过 8-bit DDR 总线送出。host 端 IDMA 自动将数据从 SDMMC2 FIFO 搬到 DRAM 0x48000000。

**④ 完成** —— eMMC 写 `CQHCI_TCN` bit 3，置位 `CQHCI_IS.TCC` 触发中断。CPU 在中断处理中读取 `CQHCI_TCN` 看到 bit 3 完成，调用 `mmc_cqe_request_done()` 通知 MMC core 层这 4KB 数据已到位。

> **传统路径做同样的事**：CMD23(8块) → 完成中断 → CMD18(读, LBA=0x8000) → 数据结束中断，每笔 4 次 CPU 参与。**CQHCI**：CPU 只写一次 DRAM 描述符 + 一次门铃 MMIO。

#### 情景 2：无数据命令（CMD6 切换分区，DCMD slot 31）

DCMD 不走 Transfer Descriptor，因为不需要数据传输。

**① 准备** —— CPU 在 slot 31 写 16 字节 Task Descriptor，参数如下：
- `ACT=0x5`：同为数据命令类型的操作（DCMD 和数据命令共用 `ACT=0x5`，区别在于 DCMD 无 Transfer Desc）
- `CMD_INDEX=6`：命令号为 6，即 CMD6（SWITCH 命令，用于切换 eMMC 分区或设置扩展寄存器）
- `RESP_TYPE=0x3`：响应类型 R1B（带 busy 信号）
- `CMD_TIMING=0`：0 表示有 busy 等待，此时 eMMC 会拉低 DAT0 表示正忙

不写 Transfer Desc。

**② 门铃** —— `writel(BIT(31), CQHCI_TDBR)`。

**③ 执行** —— eMMC 读 Task Desc 后在 MMC 总线上发 CMD6，内部切换分区，拉低 DAT0 表示 busy。上层调用者阻塞在 `mmc_blk_cmd_complete` 等 busy 结束。

**④ 完成** —— busy 结束后 eMMC 写 CQHCI_TCN bit 31 触发中断，`cqhci_finish_mrq` 唤醒等待进程。

---

### 1.4 命令提交流程：cqhci_request

现在从源码层面看 CQHCI 的提交过程。函数入口在 `drivers/mmc/host/cqhci-core.c` 第 592 行的 `cqhci_request()`。

#### 流程总览

```
cqhci_request(mmc, mrq)                              /* cqhci-core.c:592 */
  │
  ├─ [状态检查] !cq_host->enabled → 返回 -EINVAL
  ├─ [状态检查] !cq_host->activated → __cqhci_enable()
  ├─ [状态恢复] HALT 状态 → 清除 HALT, cqe_on = true
  │
  ├─ [填充描述符] 有数据? → cqhci_prep_task_desc() + cqhci_prep_tran_desc()
  │                     → cqhci_prep_dcmd_desc()
  │
  ├─ [加锁] spin_lock_irqsave
  ├─ [恢复检查] recovery_halt? → 返回 -EBUSY
  ├─ [绑定] slot[tag].mrq = mrq
  ├─ [屏障] wmb()    ← 保证描述符先于门铃被 DMA 看到
  ├─ [门铃] writel(BIT(tag), CQHCI_TDBR)
  ├─ [确认] 回读 TDBR 验证 bit 仍为 1
  └─ [解锁] spin_unlock_irqrestore
```

#### 第一步：状态检查（启用态、激活态、运行态）

CQHCI 有三个层次的状态位，各自管不同的事情：

| 状态位 | 检查位置 | 检查目的 | 什么时候触发 |
|--------|---------|---------|-------------|
| `cq_host->enabled` | `cqhci_request()` 入口 | CQHCI 基本功能是否可用 | `cqhci_init()` 成功后置 true |
| `cq_host->activated` | `cqhci_request()` 入口 | 是否从 suspend 中恢复 | `__cqhci_enable()` 后置 true，suspend 时置 false |
| `mmc->cqe_on` | `cqhci_request()` 内部 | CQE 当前是否在运行状态 | tuning/电源管理时 cqe_off，完成后 cqe_on |

具体来说：

- **`!cq_host->enabled`** —— CQHCI 初始化失败或从未使能，直接拒绝请求。`cqhci_init()` 在 probe 时调用，成功后 `cq_host->enabled = true`。如果 probe 时 CQHCI 初始化失败（如寄存器验证失败），此位不会置 true，所有后续请求都会被挡在这一步。

- **`!cq_host->activated`** —— 系统从 suspend 唤醒后，CQHCI 硬件丢失了寄存器和描述符基地址配置，需要重新使能。`!activated` 分支调用 `__cqhci_enable(cq_host)`（第 606 行），该函数会：
  1. 重新写 `CQHCI_TDLBA/U`（任务描述符基地址）
  2. 重新写 `CQHCI_CFG`（配置寄存器）
  3. 重新写 `CQHCI_ISTE/ISGE`（中断使能）
  4. 清除 `CQHCI_HALT`
  5. 置 `cq_host->activated = true`

- **`mmc->cqe_on`** —— 这个标志表示 CQE 当前正在运行。`tuning` 操作会调用 `cqe_off()` 临时关闭 CQE（因为 CMD21 不能在 CQE 模式下执行），完成后调用 `cqe_on()` 恢复。`cqhci_request()` 发现 `!cqe_on` 时执行恢复流程：写 `CQHCI_CTL` 清除 HALT 状态，置 `cqe_on = true`。

  这里要注意：**恢复和激活是两回事**。激活（activated）发生在 suspend/resume 后，涉及所有寄存器的重配；而 cqe_on 恢复只清除 HALT 状态，不重新配置寄存器。

#### 第二步：填充描述符

- **有数据（`mrq->data != NULL`）** —— 调用 `cqhci_prep_task_desc()` 填充 Task Descriptor，将 `mrq` 中的块数、起始 LBA、方向、优先级等写入描述符位域。然后调用 `cqhci_prep_tran_desc()` 遍历 `data->sg` 链表，用 `cqhci_set_tran_desc()` 为每个 SG 元素创建 Transfer Descriptor。最后一个 SG 段标记 `END=1`。

- **无数据（`mrq->data == NULL`）** —— 调用 `cqhci_prep_dcmd_desc()` 填充 DCMD Descriptor，该函数将 `mrq->cmd->opcode` 写入 Task Desc 的 `CMD_INDEX` 字段，根据 `MMC_RSP_R1B` 标志设 `RESP_TYPE` 和 `CMD_TIMING`。DCMD 没有 Transfer Descriptor。

#### 第三步：tag 分配规则

`cqhci_tag()`（第 587 行）根据 `mrq->cmd` 区分数据命令和 DCMD：

```c
static inline int cqhci_tag(struct mmc_request *mrq)
{
    return mrq->cmd ? DCMD_SLOT : mrq->tag;
}
```

- `mrq->cmd == NULL` → 这是数据命令 → tag 复用 blk-mq 的 tag（0～30）
- `mrq->cmd != NULL` → 这是 DCMD（无数据命令） → tag 固定为 31

为什么可以这样判断？在 CQHCI 模式下，MMC core 通过 CQE 接口下发请求时做了区分：数据命令的 `mrq->cmd` 设为 NULL（因为命令参数已经编码到 Task Descriptor 中），而 DCMD 需要 `mrq->cmd` 来携带命令号。所以 `mrq->cmd` 是否为 NULL 恰好能区分两种请求类型。

这样一来，数据命令使用 tag 0～30（共 31 个队列深度），DCMD 独占 slot 31。blk-mq 的 tag 被直接映射为 CQHCI slot，省去了 tag→slot 转换表。

#### 第四步：加锁与恢复检查

填充好描述符后，加 `spin_lock_irqsave`（关本地中断 + 获取自旋锁）。在锁内检查 `cq_host->recovery_halt`：

- 如果 `recovery_halt == true`，说明此时正在执行错误恢复流程，CQE 暂停中，不能提交新请求，返回 `-EBUSY`
- 上层（MMC 块层）收到 -EBUSY 后会稍后重试

恢复检查通过后执行 `slot[tag].mrq = mrq`，将 tag 和请求绑定。这个 slot 指针在中断完成时由 `cqhci_finish_mrq()` 取出并调用 `mmc_cqe_request_done(mmc, slot->mrq)`，所以必须在这里就绑定好。

#### 第五步：内存屏障 wmb()

这是 CQHCI 提交流程中最关键也最容易出错的一步。处理器的内存写入和 MMIO 写入的可见性顺序不同：

```
CPU 写描述符到 DRAM      ← store 指令，写入内存子系统
CPU 写 CQHCI_TDBR 寄存器  ← MMIO 写，走外设总线到 eMMC
```

在 ARM 弱一致性内存模型中，**这两个写入可能被重排** —— eMMC 可能在看到 doorbell 被置位时，描述符还在 CPU 的 store buffer 里没有真正到达 DRAM，于是读到陈旧数据。

`wmb()`（Write Memory Barrier）强制保证：**所有在 wmb() 之前的 store 操作，在 wmb() 之后的所有 store 操作被执行之前，已经全局可见**。翻译成 CQHCI 场景就是：执行 wmb() 后，门铃 writel 到达 CQHCI 硬件时，描述符一定已经在 DRAM 中了。

没有 wmb() 的结果：eMMC 读到错误的命令参数，可能发错 LBA，或者读到 BLK_COUNT=0 导致传 0 字节。这类 bug 在调试时表现出"随机性"——有时对有时错，取决于 CPU 和总线的时序。

在 x86 上这个 bug 不太可能出现（x86 采用较强的内存序），但在 ARM 上**一定会出问题**。这也是 CQHCI 在 ARM 上移植时最常见的坑。

#### 第六步：写门铃并回读确认

`writel(BIT(tag), CQHCI_TDBR)` —— 将 tag 对应的 bit 写 1。

写门铃后，**host 控制器**（而非 eMMC 设备）开始工作。整个"硬件自主执行"的过程分三层：

```
1. CPU 写门铃 → CQHCI 寄存器 TDBR bit X = 1
                       │
2. CQHCI 引擎（host 控制器内部逻辑）检测到 TDBR bit X
       ├─ 控制 DMA 引擎从 DRAM 读取 slot X 的 Task Descriptor
       ├─ 解析描述符：BLK_COUNT=8, BLK_ADDR=0x8000, DATA_DIR=1
       ├─ 通过 MMC 总线向 eMMC 设备发命令：
       │    ├─ CMD23（设置块数为 8）        ← 这条命令在 MMC 总线上可见
       │    └─ CMD18（从 LBA 0x8000 读）    ← 这条命令在 MMC 总线上可见
       └─ DMA 引擎将数据从 SDMMC2 FIFO 搬到 DRAM 0x48000000
                       │
3. 传输完成后 CQHCI 逻辑写 CQHCI_TCN bit X，触发 TCC 中断
```

注意：**eMMC 设备（NAND Flash）只负责在 MMC 总线上响应命令和传输数据**，它没有能力也不需要通过 DMA 去读系统的 DRAM。读 DRAM、解析描述符、在总线上发命令，这些全部由 host 控制器内的 CQHCI 逻辑+DMA 引擎完成。这就是 1.2 节中说"硬件自主执行"的"硬件"的实际含义——不是 eMMC 设备主动，而是 host 控制器自动。

---

写门铃后立即读回 `CQHCI_TDBR` 验证刚写入的 bit 是否仍为 1。如果 doorbell 没有成功置位（比如硬件处于暂停状态未完全恢复），读回值中该 bit 会是 0。这是防御性检查——硬件故障时尽早暴露，而不是等请求超时后再排查。这里只做调试输出，不阻塞流程。

---

**把整个过程串起来**：当一个读请求到达 `cqhci_request()` 时，先过三个状态检查（enabled / activated / cqe_on），确保 CQHCI 处于可用状态。然后填充描述符到 DRAM，根据请求类型走不同路径（数据走 Task Desc + Transfer Desc，DCMD 走 DCMD Desc）。最后在锁保护下，通过 wmb() 确保描述符可见后，一次 MMIO 写门铃。**从 CPU 的角度看，一笔请求从 cqhci_request 进入到返回，只需要一次 MMIO 写（门铃）**，其余都是内存操作。之后 host 控制器的 CQHCI 引擎自动完成读描述符、命令解析、总线命令收发、DMA 数据传输的全部工作。

---

### 1.5 中断完成流程：cqhci_irq

请求完成后，host 控制器内的 CQHCI 逻辑触发中断。中断处理函数 `cqhci_irq()`（`cqhci-core.c:812`）负责读取完成状态、定位完成的请求并通知上层。

#### 涉及的寄存器

- **CQHCI_IS (0x10)** —— 中断状态寄存器。每个 bit 表示一类中断事件，W1C（写 1 清除）语义。各 bit 含义：
  - `IS_TCC` (BIT1)：传输完成（Task Completion Notification）。一个或多个 slot 完成了，需读 CQHCI_TCN 查看具体哪些。
  - `IS_RED` (BIT2)：可恢复错误（Recoverable Error）。CRC 错误或超时，走恢复流程。
  - `IS_TCL` (BIT3)：任务清除完成（Task Clear）。CQHCI_TCLR 寄存器的清除命令已执行完毕。
  - `IS_GCE` (BIT4)：通用加密错误（General Crypto Error）。
  - `IS_ICCE` (BIT5)：无效加密配置错误（Invalid Crypto Config Error）。
  - `IS_HAC` (BIT0)：Host 适配完成（Host Adapter Completion）。

- **CQHCI_TCN (0x2C)** —— 完成通知寄存器。bit X = 1 表示 slot X 的请求已完成。同样 W1C 语义。

#### 中断处理流程

```
cqhci_irq(mmc, intmask, cmd_error, data_error)       /* cqhci-core.c:812 */
  │
  ├─ [读 + 清除 IS] status = readl(CQHCI_IS)
  │    writel(status, CQHCI_IS)                        /* W1C：写 1 清除 */
  │
  ├─ [错误分支] status & (RED|GCE|ICCE) || cmd/data_error?
  │    └─ cqhci_error_irq()                            /* cqhci-core.c:690 */
  │         ├─ 读 CQHCI_TERRI 获取错误详情            /* 定位出错 tag */
  │         └─ cqhci_recovery_needed()                 /* 设 recovery_halt */
  │
  ├─ [完成分支] status & IS_TCC?
  │    ├─ comp_status = readl(CQHCI_TCN)               /* 读完成位图 */
  │    │    writel(comp_status, CQHCI_TCN)              /* W1C 清除 */
  │    │
  │    ├─ for_each_set_bit(tag, &comp_status)          /* 遍历每个完成的 tag */
  │    │    └─ cqhci_finish_mrq(mmc, tag)              /* cqhci-core.c:778 */
  │    │         ├─ 设 bytes_xfered（实际传输字节数）
  │    │         └─ mmc_cqe_request_done(mmc, mrq)     /* 回调 MMC core */
  │    │
  │    └─ qcnt==0 && waiting_for_idle → wake_up        /* 唤醒等待空闲的线程 */
  │
  ├─ IS_TCL → wake_up(&wait_queue)                     /* 任务清除完成 */
  └─ IS_HAC → wake_up(&wait_queue)                     /* Host 适配完成 */
```

#### 关键行为

**1. W1C 清除避免竞态** —— `writel(status, CQHCI_IS)` 将读到的值原样写回，已置 1 的位被写 1 清除。如果在读 IS 和写 IS 之间来了新的中断，其对应 bit 不会被清除（W1C 只清除写 1 的位，新来的中断位在读到的 status 中是 0，不会被写 1 清除）。

**2. TCN 位图与批量完成** —— CQHCI_TCN 是一个 32 位位图，每一位表示一个 slot 的完成状态。一次 TCC 中断可能对应多个 slot 同时完成，CPU 通过 `for_each_set_bit` 遍历所有完成的 tag，对每个调用 `cqhci_finish_mrq()`。这是 CQHCI 减少中断次数的核心原因——**一次中断携带多个完成通知**：

```
传统路径: 每笔请求 → 1 次 CC 中断 + 1 次 DATAEND 中断 = 2 次中断
CQHCI:    多笔请求同时完成 → 1 次 TCC 中断 → 从 TCN 位图中逐个取出
```

**3. qcnt 完成计数** —— `cq_host->qcnt` 在 `cqhci_request()` 中递增（发出请求时），在 `cqhci_finish_mrq()` 中递减（完成时）。当 qcnt 降为 0 且 `waiting_for_idle` 被置位时，唤醒等待队列。这个机制用于 `cqhci_wait_for_idle()`——错误恢复时需要等所有未完成请求结束才能清除任务。

**4. 错误优先于完成** —— 中断处理先处理错误分支再处理完成分支。如果一笔请求同时触发了 RED 错误和 TCC 完成，错误处理先执行，优先进入恢复流程。

#### 传统路径对比

| 方面 | 传统路径 | CQHCI |
|------|---------|-------|
| 每笔请求中断次数 | 至少 2 次（CMD 完成 + 数据结束） | 通常 1 次（批量 TCC） |
| 完成通知携带量 | 单次中断对应单笔请求 | 单次 TCC 可携带最多 32 个 tag 完成 |
| 错误定位 | 检查 cmd->error / data->error 字段值 | 读 CQHCI_TERRI 寄存器，`TERRI_C_TASK[12:8]` 直接给出出错 tag |
| qcnt 跟踪 | 无，host->claimed 互斥 | `cq_host->qcnt` 实时跟踪未完成请求数 |

---

### 1.6 错误恢复：recovery 流程

CQHCI 错误处理分两级触发：中断检测错误 → 停止接收新请求 → 暂停 CQE → 清空未完成任务 → 重新使能。

#### 触发路径

错误检测点在 `cqhci_irq()` 和 `cqhci_error_irq()` 两处：

```
触发方式 1（硬件检测）:
  中断状态寄存器 CQHCI_IS.RED = 1
    → cqhci_error_irq() 读 CQHCI_TERRI
    → 从 TERRI_C_TASK[12:8] 定位出错 tag
    → cqhci_recovery_needed()

触发方式 2（软件检测）:
  host 驱动在命令/数据传输中检测到 cmd_error / data_error
    → 以错误码调用 cqhci_irq()
    → cmd_error/data_error 非 0 → cqhci_error_irq()
    → cqhci_recovery_needed()
```

`cqhci_recovery_needed()`（第 662 行）实际只做三件事：

```c
/* cqhci-core.c:662 */
static void cqhci_recovery_needed(struct mmc_host *mmc, struct mmc_request *mrq,
                                  bool notify)
{
    cq_host->recovery_halt = true;       /* ① 关门：新请求的检查点（cqhci_request 中检查此位） */
    wake_up(&cq_host->wait_queue);        /* ② 唤醒：可能有人在等 idle */
    mrq->recovery_notifier(mrq);          /* ③ 通知：告知块层停止下发新请求 */
}
```

三个动作的含义：
- `recovery_halt = true` —— `cqhci_request()` 入口锁内检查此位，发现已设则返回 -EBUSY。这是**从源头拦截新请求**，防止在恢复过程中有新的请求进入 CQHCI。
- `wake_up` —— 如果有线程在 `cqhci_wait_for_idle()` 上等待，此时被唤醒，进入恢复流程。
- `recovery_notifier` —— 回调到 MMC 块层，通知块层不要再提交新的 CQE 请求到这个 host。

#### 两阶段恢复接口

CQHCI 通过 `cqhci_cqe_ops`（第 1121 行）向 MMC core 注册了 `cqe_recovery_start` 和 `cqe_recovery_finish`，形成标准的恢复流程：

**阶段 1 — `cqe_recovery_start`（把现场冻结）:**

```
cqhci_recovery_start()                                /* cqhci-core.c:1094 */
  │
  ├─ writel(CQHCI_HALT, CQHCI_CTL)                    /* 暂停 CQE 硬件 */
  │    暂停后 CQHCI 不再处理 doorbell，正在传输的请求继续完成
  │
  ├─ cqhci_wait_for_idle()                            /* 等待 qcnt 降为 0 */
  │    循环检查 qcnt，直到所有正在执行的请求自然完成
  │    超时则直接继续
  │
  ├─ cqhci_clear_all_tasks()                          /* 清除 CQE 中的所有任务 */
  │    写 CQHCI_TCLR 寄存器，等待 IS_TCL 中断确认清除完成
  │
  └─ cqhci_recover_mrqs()                             /* 回收未完成的请求 */
       └─ for_each slot → 有 mrq 未完成 → mmc_cqe_request_done(mmc, mrq)
           以错误状态回调 MMC core，释放仍挂在 slot 上的请求
```

**阶段 2 — `cqe_recovery_finish`（重新初始化）:**

```
cqhci_recovery_finish()                               /* cqhci-core.c:1099 */
  │
  ├─ writel(0, CQHCI_CTL)                             /* 清除 HALT */
  ├─ __cqhci_enable()                                 /* 重新使能 CQHCI */
  │    重新配置 TDLBA、CFG、ISTE/ISGE 等寄存器
  │
  └─ recovery_halt = false                            /* 开门：允许新请求进入 */
```

两阶段中间的过程由 MMC core 层控制，确保所有等待的请求在恢复完成后才重新下发。

---

### 1.7 在 STM32MP2 上的情况

**一句话结论：STM32MP2 的 SDMMC 控制器硬件上有 CQHCI 能力（SDMMC 内建），但 mmci 驱动层完全没有实现 CQHCI 支持。ATK 板使用 DDR52 模式，CQHCI 即使实现也收益有限。**

#### 硬件能力 vs 驱动实现

先看 SoC 级硬件能力。SDMMC2 控制器在 `stm32mp251.dtsi` 中声明：

```dts
/* arch/arm64/boot/dts/st/stm32mp251.dtsi:2253 */
sdmmc2: mmc@48230000 {
    compatible = "st,stm32mp25-sdmmc2", "arm,pl18x", "arm,primecell";
    arm,primecell-periphid = <0x00353180>;
    reg = <0x48230000 0x400>, <0x44230800 0x8>;
    interrupts = <GIC_SPI 197 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&rcc CK_KER_SDMMC2>;
    clock-names = "apb_pclk";
    max-frequency = <166000000>;
    ...
};
```

`max-frequency = <166000000>` 说明控制器最高支持 166MHz。从硬件规格看：
- 166MHz SDR → HS200 模式（~166MB/s 8-bit）
- 166MHz DDR → HS400 模式（~332MB/s 8-bit DDR），但需要 DQS 信号线支持

SDMMC2 的 "st,stm32mp25-sdmmc2" 兼容字符串匹配 `mmci.c` 中的 `variant_stm32_sdmmcv3`，该 variant 定义了 `f_max = 267000000`（SoC DTS 中限为 166MHz），但 `variant_data` 结构体没有 `caps` 或 `caps2` 字段，也没有任何 CQHCI 相关字段。

再看 ATK 板的 DTS 覆盖：

```dts
/* stm32mp257d-atk-ddr-2GB.dts:661 */
&sdmmc2 {
    pinctrl-names = "default", "opendrain", "sleep";
    pinctrl-0 = <&sdmmc2_b4_pins_a &sdmmc2_d47_pins_a>;
    pinctrl-1 = <&sdmmc2_b4_od_pins_a &sdmmc2_d47_pins_a>;
    pinctrl-2 = <&sdmmc2_b4_sleep_pins_a &sdmmc2_d47_sleep_pins_a>;
    non-removable;
    no-sd;
    no-sdio;
    st,neg-edge;
    bus-width = <8>;
    vmmc-supply = <&scmi_vdd_emmc>;
    vqmmc-supply = <&scmi_vddio2>;
    mmc-ddr-1_8v;
    status = "okay";
};
```

关键点：**只声明了 `mmc-ddr-1_8v`，没有 `mmc-hs200-1_8v` 和 `mmc-hs400-1_8v`。** 这意味着内核不会尝试 HS200/HS400，直接在 DDR52 模式下运行。对比 ST 官方 MX 板：

```dts
/* stm32mp257f-ev1-mx.dts:1146 — MX 板增加了 HS200 */
&sdmmc2 {
    mmc-ddr-1_8v;
    mmc-hs200-1_8v;     /* ← ATK 板没有这行 */
};
```

#### 驱动缺少的 CQHCI 支持链

整个 mmci 驱动栈（`mmci.c` + `mmci_stm32_sdmmc.c`）没有任何 CQHCI 代码，具体缺口如下：

| 层面 | 需要什么 | 现状 |
|------|---------|------|
| **Kconfig** | `MMC_STM32_SDMMC` 需 `select MMC_CQHCI` | 未选择，`MMC_CQHCI` 是独立选项 |
| **variant_data** | 需添加 `caps2 = MMC_CAP2_CQE \| MMC_CAP2_CQE_DCMD` | 无 caps/caps2 字段 |
| **probe** | 调用 `cqhci_init()` 分配 DMA 描述符内存 | 无调用 |
| **IRQ** | `cqhci_irq()` 处理 TCC/TE 中断 | mmci 只处理传统 MCI 中断 |
| **ops** | 注册 `cqhci_cqe_ops` 到 `mmc_host.cqe_ops` | `cqe_ops` 为 NULL |
| **DMA** | CQHCI 需要自己的 DMA 描述符区域 | IDMA 只能处理单笔传输，无队列能力 |

#### ATK 板使用 DDR52 的合理性

ATK 板跑在 DDR52 8-bit 50MHz，理论带宽约 200MB/s，实际顺序读写 ~100-150MB/s。这个速度下：

- 每笔请求的传输时间远大于命令交互开销，CPU 逐条发 CMD 的瓶颈不突出
- CQHCI 的优势体现在**命令密集**场景（小 IO、高并发），而 DDR52 上单笔大 IO 居多
- DDR52 的信号完整性要求低，不需要 DLYB tuning（`min_freq = 100000000`，DDR52 50MHz 低于 tuning 阈值）

如果后续切换到 HS200 或 HS400，CQHCI 就变得必要了。那时需要补齐上述整个支持链，工作量包括：
1. Kconfig 增加 `select MMC_CQHCI`
2. `mmci_stm32_sdmmc.c` 中实现 `cqhci_host_ops`（`cqhci_requests` 需要将 CQHCI 请求映射到 SDMMC 总线的门铃写）
3. 修改中断处理共享（CQHCI 中断和传统 MCI 中断共用同一个 IRQ 号）
4. DTS 中增加 CQHCI 相关属性（如 `mmc-cqe`，但 CQHCI 通常不需要 DTS 属性，寄存器基地址直接复用 SDMMC 的 reg 区域）

---

### 1.8 CQHCI 的关键约束

#### 约束 1：最多 32 个排队任务

CQHCI 硬件定义 `NUM_SLOTS=32`（0–31），门铃寄存器 `CQHCI_TDBR` 是 32 位位图，每一位对应一个 slot。这是硬件设计上限——`TDBR` 寄存器位宽决定的，软件无法扩充。

实际可用数据队列深度不是 32，是 **31**：

```
slot 0–30：数据命令（read/write），对应 blk-mq tag 0–30
slot 31：  DCMD（CMD6/CMD13 等无数据命令），固定预留
```

31 个数据队列深度对于 eMMC 场景完全够用。HS400 下单笔读延迟 ~100μs（含 NAND 寻址 + 数据传输），31 笔并发可以覆盖 ~3ms 的延迟隐藏需求。对比 NVMe 有 64K 队列，eMMC 不需要那么深——eMMC 的延迟瓶颈在 NAND 介质，队列深了收益递减。

#### 约束 2：VALID bit 和 doorbell 写入顺序

CPU 的写入顺序必须是：

```
1. 写 Task Descriptor 到 DRAM  ← 普通 store
2. 写 VALID=1 到 Descriptor     ← 标记描述符可用
3. wmb() 内存屏障               ← 保证 1+2 先于 4 可见
4. writel(BIT(tag), TDBR)      ← 门铃
```

**如果顺序错了**（先写门铃再写 VALID），eMMC/host 读到 VALID=0 认为描述符无效，这个 slot 永远不会被执行。而且这个 bug 是**间歇性**出现的——取决于 CPU store buffer 刷新的时机，调试时极难复现。

注意：描述符填充和门铃之间有 `wmb()`，但描述符内部的 VALID 位和其他字段之间没有单独屏障。这是因为 CPU 对同一 cacheline 的写入是保序的——Task Descriptor 只有 16 字节，在一个 cacheline（ARM64 通常是 64 字节）内，CPU 按程序顺序提交 store。所以先写 `BLK_ADDR/BLK_COUNT` 再写 `VALID=1` 没有问题。**但如果描述符跨越两个 cacheline（比如启用了 128-bit 描述符模式），就需要额外的 DMA 屏障**，因为跨 cacheline 的顺序不被硬件保证。

#### 约束 3：wmb() 内存屏障的硬件原理

ARM64 是弱一致性内存模型。处理器可以重排对不同地址的写入：

```c
// 没有 wmb() 的情况 — ARM64 允许硬件重排
*(desc_blk_addr) = 0x8000;   // store 1: 写 LBA 到 DRAM
*(desc_valid)    = 1;        // store 2: 写 VALID bit
writel(BIT(3), TDBR);       // store 3: 写门铃（MMIO）
```

ARM64 的内存排序规则：
- **同一个地址**（同一 cacheline）：保序
- **不同地址**（DRAM vs MMIO）：**不保序**

所以没有 `wmb()` 时，store 3（门铃 MMIO）可能先于 store 1+2（DRAM 描述符）到达目的地。host CQHCI 引擎看到门铃后立刻去 DRAM 读描述符，可能读到 `BLK_ADDR=0`（store 1 还没刷出去），发出 CMD18 读 LBA 0，数据全错。

`wmb()` 插入了一条 **DSB（Data Synchronization Barrier）** 指令，强制流水线中所有之前的 store 完成后才能执行之后的 store。代价是 ~几十纳秒的停顿（取决于总线拓扑），但对于 CQHCI 的正确性必不可少。

在 x86 上通常不需要这个屏障，因为 x86 的 TSO（Total Store Order）模型保证不同地址的 store 按程序顺序对外可见。这也是为什么 CQHCI 驱动在 x86 模拟器上测试没问题、部署到 ARM 上就概率性出错。

#### 约束 4：CQE on/off 切换

CQHCI 硬件在运行时，MMC 总线上的命令流完全由 CQHCI 控制器接管。传统路径通过 `MMCI_CMD` 和 `MMCI_DATA` 寄存器发命令，CQHCI 模式下这些寄存器不再被使用。这意味着：

- **CMD21（tuning）不能通过 CQHCI 发出**——因为 CMD21 是特殊命令，不走 Task Descriptor 流程
- **电源管理命令（CMD5 SLEEP/AWAKE）不能通过 CQHCI 发出**——需要直接控制总线
- **CMD0（复位）不能通过 CQHCI 发出**——硬复位必须在 CQHCI 停止后做

所以 `mmc_execute_tuning()` 的流程是标准模板：

```c
/* core/core.c — mmc_execute_tuning() */
if (host->cqe_on)
    host->cqe_ops->cqe_off(host);   /* 关 CQE：停止硬件队列 */
    
err = host->ops->execute_tuning(host, opcode);  /* 发 CMD21 遍历延迟步进 */

if (err)
    host->cqe_ops->cqe_on(host);    /* 开 CQE：恢复硬件队列 */
```

**关 CQE（`cqe_off`）** 做的事情：
1. 写 `CQHCI_CTL` 置 HALT 位 → 暂停 CQHCI 引擎
2. 等待正在传输的请求自然完成（`cqhci_wait_for_idle`）
3. 置 `mmc->cqe_on = false`

**开 CQE（`cqe_on`）** 做的事情：
1. 写 `CQHCI_CTL` 清 HALT 位 → 恢复 CQHCI 引擎
2. 置 `mmc->cqe_on = true`

关键点：`cqe_off` 不是掉电，只是暂停。已提交到卡但未完成的请求会在 CQHCI 恢复后继续。如果是在错误恢复场景，则走 `cqhci_recovery_start/finish`（见 1.6 节），那会清空所有未完成任务。

#### 约束 5：DCMD slot 31 的单点瓶颈

所有无数据命令（CMD6 切换分区、CMD13 查状态、CMD5 SLEEP/AWAKE）共用同一个 slot 31。这带来了几个实际限制：

**串行化** —— DCMD 在同一时刻只能有一个在排队。当 `cqhci_request()` 收到一个 DCMD 时，如果 slot 31 上还有未完成的 DCMD，`cqhci_tag()` 返回 31，`cqhci_request()` 发现 slot 31 的 `mrq` 指针非空，直接返回 `-EBUSY`。

**阻塞场景** —— 如果文件系统在写数据的同时频繁执行 `CMD6` 切换分区（比如 GBA 模拟器访问 eMMC 的不同分区），DCMD 的串行化带来的延迟可能在 100μs–1ms 级别。对于 50MHz DDR52 模式，1ms 大约浪费 ~25KB 吞吐量。

**为什么只有一个 DCMD slot？** CQHCI 硬件设计者认为无数据命令是低频的（状态查询、分区切换），不需要深度队列。实际上也的确如此——在正常 IO 路径中，DCMD 远少于数据命令（通常 <1% 的比例）。

#### 约束 6：Transfer Descriptor 的 SG 页面边界限制

`DAT_ADDR` 字段没有要求物理地址对齐到页边界，`DAT_LENGTH` 最大 65535 字节（16-bit 字段）。但实际使用中 Linux 的块层下发 IO 时通常以 4KB 为单位对齐，一个 SG 段恰好对应一个 4KB 物理页。一笔大 IO 被拆成多个 4KB 段，每个段一个 Transfer Descriptor。

对于大 IO（比如 512KB 读），`cqhci_set_tran_desc()` 需要遍历 `data->sg` 链表，为每个 sg 段创建一个 Transfer Descriptor。SG 链表的 DMA 地址来自 `sg_dma_address(sg)`，由 `dma_map_sg()` 在上层完成映射。如果 DMA 映射失败（如 IOMMU 翻译出错），`sg_dma_address` 返回无效地址，CQHCI 传出的数据全部写入错误的物理内存——这不是 CQHCI 本身的问题，但排查时很容易怀疑到 CQHCI 头上。

---

## Part 2：Tuning — 调相与信号完整性

---

### 2.1 问题的起源

当 eMMC 运行在 **HS200** 模式时，时钟频率达到 200MHz，周期仅 5ns：

```
周期 = 1 / 200MHz = 5ns
数据有效窗口 ≈ 周期 / 2 = 2.5ns (SDR)
```

在 5ns 的尺度下，PCB 走线长度、I/O pad 延迟、温度漂移、电压波动等因素造成的总延迟可以轻松逼近甚至超过 2.5ns 的采样窗口。比如：

- PCB 走线每 1cm 引入 ~60-70ps 延迟
- I/O pad 缓冲器引入 ~1-2ns 延迟
- 温度每变化 50°C 引入 ~几百 ps 的漂移

结果是：**host 控制器不知道应该在什么时刻采样数据线上的信号**。采样点太早或太晚都会读到错误的数据。

> **Tuning 就是要找那个正确的采样点。** 而且这个正确采样点会随温度电压变化，所以不能只在启动时调一次——需要定期重调（`mmc_retune`）。

---

### 2.2 CMD21 Tuning 流程

Tuning 的核心机制是 **CMD21（`MMC_SEND_TUNING_BLOCK_HS200`）**。

#### 卡返回已知模式

eMMC 收到 CMD21 后，返回预定义的已知模式。8-bit 总线返回 128 字节（`tuning_blk_pattern_8bit`），4-bit 总线返回 64 字节（`tuning_blk_pattern_4bit`）。host 已知这个模式的期望值，所以通过"发送 CMD21 → 读取返回数据 → 和期望值比较"来判断当前采样点是否正确。

```c
// mmc_ops.c - mmc_send_tuning()
cmd.opcode = MMC_SEND_TUNING_BLOCK_HS200;  // CMD21
cmd.flags  = MMC_RSP_R1 | MMC_CMD_ADTC;

data.blksz    = size;       // 8-bit: 128 bytes, 4-bit: 64 bytes
data.blocks   = 1;
data.flags    = MMC_DATA_READ;
data.timeout_ns = 150 * NSEC_PER_MSEC;  // 150ms timeout

// 等待命令完成
mmc_wait_for_req(host, &mrq);

// 比较返回数据和期望模式
if (memcmp(data_buf, tuning_block_pattern, size))
    err = -EIO;
```

关键点：**CMD21 返回数据正确性完全取决于采样点的位置**。如果采样点不对，读到的数据会错位，`memcmp` 返回非零，这步 tuning 就失败了。

#### 上层调用链

```
mmc_init_card()
  └─ mmc_select_hs200()          ← 尝试切换到 HS200
       └─ mmc_hs200_tuning()     ← HS200 必须调相
            └─ mmc_execute_tuning()
                 └─ host->ops->execute_tuning(mmc, opcode)
                      └─ sdmmc_execute_tuning()    ← 主机控制器具体实现
```

`mmc_execute_tuning()` 中还有一个关键细节：如果 CQE 正在运行，需要先关闭它：

```c
// core.c - mmc_execute_tuning()
if (host->cqe_on)
    host->cqe_ops->cqe_off(host);  // ← 调相前必须先关 CQE
```

因为 CQHCI 硬件会接管命令流程，tuning 这种特殊命令无法在 CQE 运行期间正常执行。

---

### 2.3 采样窗口搜索算法

Tuning 的核心算法在 `sdmmc_dlyb_phase_tuning()` 中。它实现了一个 **线性扫描 + 中值选取** 的策略：

```c
// mmci_stm32_sdmmc.c - sdmmc_dlyb_phase_tuning()
for (phase = 0; phase <= dlyb->max; phase++) {
    // 1. 设置延迟步进
    dlyb->ops->set_cfg(dlyb, dlyb->unit, phase, false);

    // 2. 发 CMD21 试这个相位
    if (mmc_send_tuning(host->mmc, opcode, NULL)) {
        cur_len = 0;                // 失败 → 清零连续计数
    } else {
        cur_len++;                  // 成功 → 累加连续窗口长度
        if (cur_len > max_len) {
            max_len = cur_len;      // 记录最大窗口
            end_of_len = phase;     // 记录窗口结束位置
        }
    }
}

// 3. 取窗口中间值（留 margin）
phase = end_of_len - max_len / 2;
dlyb->ops->set_cfg(dlyb, dlyb->unit, phase, false);
```

一个直观的理解：

```
延迟步进:  0  1  2  3  4  5  6  7  8  9  10  11  12  13  14
CMD21 结果: ✗  ✗  ✓  ✓  ✓  ✓  ✓  ✗  ✗  ✗   ✓   ✓   ✗   ✗   ✗
                    |_________|           |____|
                    窗口1 (len=5)         窗口2 (len=2)
                    
max_len=5, end_of_len=7  →  选 phase = 7 - 5/2 = 7 - 2 = 5
```

算法会记录整个扫描过程中**最长的连续成功窗口**，然后取这个窗口的中间值作为最终采样相位。这样做的目的是留出余量（margin）：如果窗口边缘因为温度变化而收缩，采样点还在窗口内。

#### 为什么不用二分搜索？

你可能会想：这遍历几十上百步不就太慢了吗？为什么不用二分？

答案是：**采样窗口可能不连续**。上例中就有两个分离的窗口（可能由信号反射、串扰等问题导致）。二分搜索可能只找到一个窗口就停了，而线性扫描能发现所有窗口并选择质量最好的（最长的）。

> 实际步数视平台而定：MP25 固定 32 步（`DLYBSD_TAPSEL_NB = 32`），MP1 则取决于 `sdmmc_dlyb_mp15_prepare` 中测得的延迟线长度（`dlyb->max = __fls(lng)`），通常在 8–11 步。所以 tuning 实际耗时在微秒级，不是一个性能敏感的操作。

---

### 2.4 DLYB 硬件：延迟线如何移相

DLYB（Delay Line Block）是调节采样相位的硬件。它的本质是 **把采样时钟延迟一段时间再触发采样**，从而对齐数据眼图的中心。

#### 物理模型：为什么要"移相"

```
数据信号眼图:  
              ┌──┐  ┌──┐  ┌──┐  ┌──┐
              │  │  │  │  │  │  │  │  ← eMMC 输出的数据
              └──┘  └──┘  └──┘  └──┘
采样时钟:   ▲      ▲      ▲      ▲        ← 默认采样点（可能在数据跳变处）
              ↑
          眼图中心 ← 正确采样点
```

eMMC 发出的数据和时钟之间有固定的相位偏移（由 PCB 走线、I/O pad 延迟等决定）。默认的采样点可能正好落在数据跳变处（眼图边缘），读到不确定的值。DLYB 做的事情就是**把采样时钟往后推一段时间**，让采样点落在数据稳定的窗口中。

#### 延迟链的物理结构

DLYB 内部是一串首尾相连的缓冲器（buffer），称为**延迟链**。每个缓冲器引入一个固定的微小延迟（称为一个 tap）：

```
                tap 0     tap 1     tap 2          tap N
输入时钟:  ──→ [BUF] ──→ [BUF] ──→ [BUF] ──→ ... ──→ [BUF] ──→
                    │         │         │                │
                  输出0      输出1      输出2            输出N
                                              ↑
                                        多路选择器 → 选哪个 tap 输出作为采样时钟
```

每个 tap 的输出就是输入时钟延迟了 N × 单个缓冲器延迟 后的波形。多路选择器从 N+1 个 tap 中选一个作为最终采样时钟。选择的 tap 编号就是代码的 `phase` 参数。

以 MP25 为例：`DLYBSD_TAPSEL_NB = 32`，所以有 32 个 tap（phase 0–31）。单个 tap 延迟约 50–100ps（取决于工艺/电压/温度）。最大可延迟范围 = 32 × 100ps = 3.2ns——覆盖 HS200 的 2.5ns 数据有效窗口绰绰有余。

#### 具体例子：phase=5 代表什么

假设 MP25 的每个 tap 延迟 ~80ps。`set_cfg(dlyb, 0, 5, false)` 将采样时钟延迟 5 × 80ps = 400ps。

```
tap 编号:     0     5      10      15      20      25      30
累加延迟(ps): 0    400    800    1200    1600    2000    2400    3200
                          ↑
采样窗口:     └──────✓──────✓──────✓──────✓──────┘
                        相位 10-25 返回正确数据
                          ↑
                         选 phase=17（窗口中心，留余量）
```

tuning 算法遍历 phase 0→31，对每个 phase 发 CMD21 检查数据正确性。上例中 phase 10–25 返回正确数据，最长连续窗口 16 步，选中间值 17。这样温度变化时窗口左右收缩，采样点仍在窗口内。

#### 结构体设计

```c
// mmci_stm32_sdmmc.c:79
struct sdmmc_dlyb {
    void __iomem *base;       // 寄存器基地址（MP1→DLYB 模块，MP25→SYSCFG）
    u32 unit;                 // 延迟线单元编号（MP1 多路选择用，MP25 固定）
    u32 max;                  // 最大 phase 值（MP25=31，MP1 动态 8-11）
    unsigned int min_freq;    // 需要 tuning 的最低频率（MP25=100MHz）
    struct sdmmc_tuning_ops *ops;
};
```

- **base** —— MP1 指向 DLYB 独立外设的寄存器基地址；MP25 指向 SYSCFG 块中 DLYBSD 相关寄存器的基地址
- **max** —— `for (phase = 0; phase <= dlyb->max; phase++)` 的上限。MP25 固定为 31（`DLYBSD_TAPSEL_NB - 1`，共 32 步）；MP1 需要运行 `sdmmc_dlyb_mp15_prepare` 测量各条延迟线的实际长度后确定
- **min_freq** —— 低于此频率不需要 tuning。信号频率低时数据有效窗口大，默认采样点就足够可靠。MP25 为 100MHz，MP1 为 50MHz

```c
// mmci_stm32_sdmmc.c:71 — ops 回调屏蔽 MP1/MP25 硬件差异
struct sdmmc_tuning_ops {
    int (*dlyb_enable)(struct sdmmc_dlyb *dlyb);
    void (*set_input_ck)(struct sdmmc_dlyb *dlyb);
    int (*tuning_prepare)(struct mmci_host *host);
    int (*set_cfg)(struct sdmmc_dlyb *dlyb, int unit,
                   int phase, bool sampler);
};
```

`set_cfg` 是核心操作：将第 `unit` 条延迟链上的第 `phase` 个 tap 输出选为采样时钟。MP25 忽略 `unit`，直接将 `phase` 写入 `SYSCFG_DLYBSD_CR.RXTAPSEL`。

#### MP25 的寄存器操作

MP25 的 DLYB 集成在 SYSCFG（系统配置寄存器）中，不是独立外设。涉及两个寄存器：

```
SYSCFG_DLYBSD_CR:
  bit 0    EN       — 延迟链使能。写 1 启动，内部自动完成延迟校准并锁定
  bits 6:1 RXTAPSEL — RX 采样 tap 选择。6-bit 字段（硬件支持 0–63），
                      软件限制最大 31（dlyb->max = DLYBSD_TAPSEL_NB - 1）

SYSCFG_DLYBSD_SR:
  bit 0    LOCK     — 锁定状态。EN=1 且内部 PLL 稳定后自动置 1
  bit 16   RXTAPSEL_ACK — tap 写入确认。每次写 RXTAPSEL 后，硬件切换完成时置 1
```

`set_cfg` 的 MP25 实现（伪代码）：

```c
// 写 phase 到 RXTAPSEL 字段
reg = readl(SYSCFG_DLYBSD_CR);
reg = (reg & ~RXTAPSEL_MASK) | (phase << RXTAPSEL_SHIFT);
writel(reg, SYSCFG_DLYBSD_CR);

// poll RXTAPSEL_ACK 确认硬件已切换到新 tap
readl_poll_timeout(SYSCFG_DLYBSD_SR, sr, sr & RXTAPSEL_ACK, 1, 1000);

// W1C 清除 ACK 位
writel(RXTAPSEL_ACK, SYSCFG_DLYBSD_SR);
```

每次 tuning 循环迭代（phase 0→1→2→...→31）都执行上述操作：写 RXTAPSEL → poll ACK → 发 CMD21 验证。

#### MP1 对比：为什么多一个扫描步骤

MP1 的 DLYB 是独立外设 IP，不是集成在 SYSCFG 中的。独立 IP 意味着芯片设计者对延迟线的 PVT（工艺-电压-温度）特性没有系统级保证——同一芯片上不同延迟线的长度可能不同。

```
MP1 DLYB 内部结构：
              UNIT 0:  [BUF]─[BUF]─[BUF]─...─[BUF]    ← 128 个 tap
              UNIT 1:  [BUF]─[BUF]─[BUF]─...─[BUF]
              UNIT 2:  [BUF]─[BUF]─[BUF]─...─[BUF]
              ...
              UNIT 127: [BUF]─[BUF]─[BUF]─...─[BUF]
```

每个 UNIT 是一条独立的延迟链，有 128 个 tap。但不同 UNIT 的延迟特性不同（工艺分布决定）。`sdmmc_dlyb_mp15_prepare` 遍历 UNIT 0–127，对每个 UNIT 测量 LNG（自动测得的延迟线总长度），选出满足 `lng < BIT(DLYB_NB_DELAY)` 的 UNIT。最终 `dlyb->max = __fls(lng)`——用 LNG 的最高 bit 位置作为最大 phase，通常是 8–11。

这就是为什么 MP1 的 prepare 比 MP25 复杂得多。MP1 要**先找一条能用的延迟线**（UNIT 0–127 里碰运气），再测它的长度，然后才能开始 tuning。MP25 的延迟链直接固化在 SYSCFG 中，32 tap 全部可用，不需要扫描。

```
MP25:  32 tap 固定延迟链 → 写 RXTAPSEL 选 phase → 发 CMD21
MP1:   扫描 128 条延迟链 → 选一条可用的 → 测其长度 → 遍历 phase 发 CMD21
         ↑ MP1 多这一步是因为 DLYB 是独立 IP，PVT 覆盖无法保证
```

---

### 2.5 Tuning 失败的回退机制

Tuning 失败是板上调试时最常见的问题之一。Linux MMC 子系统设计了**自动降级**机制：

```
HS200 tuning 失败
    ↓
内核自动降级到 DDR52（不需要 tuning）
    ↓
卡仍然可以工作，但速度从 ~200MB/s 降到 ~104MB/s
```

实现位置在 `mmc_init_card()` 中：

```
mmc_init_card()
  ├─ mmc_select_hs200()       ← 尝试 HS200
  │    └─ mmc_hs200_tuning()  ← 包含 tuning
  │         └─ 失败返回 -EIO/-ETIMEDOUT
  │
  └─ mmc_select_hs_ddr52()    ← 回退到 DDR52
       └─ 不需要 tuning
```

这解释了为什么 ATK 板的 DTS 没有声明 `mmc-hs200-1_8v`，eMMC 仍然能正常工作——它在 DDR52 模式下运行。

#### Tuning 失败的原因排查

1. **vqmmc 电压不足**：HS200 需要 1.8V vqmmc。用万用表量实际电压。
2. **时钟质量差**：200MHz 时钟抖动过大。示波器看眼图。
3. **DLYB 延迟线范围不够**：DLYB 的最大延迟步进需要覆盖至少一个完整的数据 bit 周期。如果物理设计上限不够，采样点不可能正确。
4. **PCB 走线不等长**：8 条数据线之间的长度差过大，每条线的采样窗口位置不同，没有通用的延迟步进能覆盖所有线。

#### 调试手段

```
# 查看当前 timing spec
cat /sys/kernel/debug/mmc1/ios

# timing spec 数值含义:
# 6 = MMC_TIMING_MMC_DDR52 (DDR52)
# 7 = MMC_TIMING_MMC_HS200
# 8 = MMC_TIMING_MMC_HS400

# 查看 tuning 是否成功
dmesg | grep mmc1
  mmc1: hs200 tuning failed  ← 需要排查
  mmc1: DDR52 mode selected  ← 降级后的模式
```

---

### 2.6 Enhanced Strobe（HS400 特有）

HS400 模式下有一个特殊的 tuning 机制——**Enhanced Strobe**。

#### 原理

HS200 用**时钟信号（CLK）**来采样数据，所以需要 tuning 来找到正确的采样点。

HS400 改用 **DQS（Data Strobe）差分信号** 来采样数据。关键区别是：

> **DQS 由 eMMC 发出，与数据信号同源同路径。** 所以 DQS 和数据之间的相对延迟很小——无论温度怎么变，DQS 和数据一起漂移，相对位置不变。

这就是 Enhanced Strobe 不需要 tuning 的根本原因。

```
HS200 采样:  主机发 CLK → 卡返回数据 → 主机用 CLK 采样数据
             ↑ 主机时钟和数据不同源，需要 tuning 对齐

HS400 采样:  卡返回数据 + DQS → 主机用 DQS 采样数据
             ↑ DQS 和数据同源，自动对齐
```

#### 软件流程

如果实现了 `host->ops->enhanced_strobe()`，在切换到 HS400 时会调用：

```c
// mmc.c 中的 mmc_select_hs400()
mmc_set_timing(host, MMC_TIMING_MMC_HS400);
mmc_set_bus_speed(card);

if (host->ops->execute_hs400_tuning) {
    // 即使 HS400 不需要 tuning，有些主机仍需要微量调相
    host->ops->execute_hs400_tuning(host, card);
}
```

---

### 2.7 Tuning 与 CQHCI 的交互

当一个系统同时使用了 CQHCI 和 HS200/HS400（虽然 ATK 板没有），tuning 和 CQHCI 之间有重要的交互：

1. **Tuning 前必须关 CQE**：如 `mmc_execute_tuning()` 中的 `cqe_off()`。因为 CMD21 是一条特殊命令，CQHCI 不会自动处理它。

2. **重调（Retune）**：温度变化导致采样窗口漂移后，内核需要重新 tuning。retune 时先停 CQE，再发 CMD21，找到新采样点，再重新使能 CQE。

3. **retune 触发时机**：`mmc_host` 中有一个 `need_retune` 标志，在每次请求完成后检查。如果发现需要 retune，下一个请求会先做 retune 再执行。

---

---

## 总结

1. **门铃模型将命令提交从 O(n) 降为 O(1)**：传统每笔需 3 次 MMIO，CQHCI 只需一次内存写 + 一次门铃。代价是 ARM 弱内存下 `wmb()` 不可或缺。

2. **host 控制器才是真正的执行者**：读 DRAM 描述符、解析参数、发 MMC 命令——全部由 host CQHCI 引擎完成，eMMC 只被动响应总线。

3. **Tuning 的本质是给采样时钟"找位置"**：DLYB 用 N 个 buffer 逐步延迟时钟，找到数据眼图中心。MP25 固定 32 tap，MP1 需先扫 128 条延迟线。

4. **ATK 板实际情况**：DDR52 50MHz 下 CQHCI 收益有限。DTS 只声明 `mmc-ddr-1_8v`，mmci 驱动完全无 CQHCI 代码。

5. **HS400 的 DQS 让 tuning 成为过去**：Enhanced Strobe 的 DQS 与数据同源同路径，延迟一起漂移，不需要调相。

---

> **下一篇：** [08-Interview.md](08-Interview.md) — 将全书内容整理为面试题库。
