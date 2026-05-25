# 08. MMC 子系统面试指南

> 本文是系列第 8 篇。将 00-07 的核心知识提炼为结构化面试题库，每道题标注难度星级和对应文档索引。
>
> **前置：** 全系列 00-07
>
> **字数：** 中文字数 6,248 + 英文单词 1,731 ≈ **7,979 字**，**行数：296**
>
> **建议阅读时间：** 40–60 分钟

---

## 2.1 历史与演进

对应：00-History — MMC 子系统演进历史和设计决策。

**Q1：MMC 和 SD 卡的协议差异是什么？为什么 Linux 驱动要用"多轮试探"？** ★★

同一物理总线（CLK/CMD/DAT）上挂着不同类型的卡，协议不兼容。MMC 用 CMD1 获取卡信息，SD 用 ACMD41，SDIO 用 CMD5。Linux 驱动不知道插槽上是什么卡，只能每种协议都试一遍——先试 MMC（CMD1 静默），再试 SD（ACMD41），最后试 SDIO（CMD5）。`mmc_rescan` 中的试探顺序决定了初始化的延迟。

**Q2：eMMC v4.4 / v5.0 各自的关键特性是什么？** ★★

| 版本 | 关键特性 |
|------|---------|
| v4.4 (2011) | HS200（200MHz SDR），CMD21 tuning |
| v5.0 (2013) | HS400（200MHz DDR），CQHCI（硬件命令队列） |

v4.4 引入了 tuning 机制解决高速采样的信号完整性问题。v5.0 引入 CQHCI 将命令调度卸载到硬件。v5.0 之后 eMMC 物理层基本到头，高速场景被 UFS 分流。

**Q3：blk-mq 什么时候合入 MMC 子系统的？解决了什么问题？** ★★★

Linux v4.13（2017 年）。之前 MMC 使用单队列 bio 路径（`mmc_queue_fn`），所有 IO 请求串行通过一个工作队列处理。blk-mq 带来了 per-CPU 无锁提交（`mmc_mq_queue_rq` 每个 CPU 各自提交流程无需全局锁）、tag 系统提供背压、多队列深度（虽然 MMC 在硬件上仍是单队列）。提交者是 ST 工程师 Ulf Hansson。

---

## 2.2 硬件与协议

对应：02-Hardware — eMMC 芯片工作原理、总线协议、时序。

**Q4：MMC 命令有哪几种类型？R1B 响应和 R1 有何不同？** ★★

四种类型：广播命令（CMD0/CMD1）、点对点命令（CMD7 选卡）、带数据命令（CMD18/CMD25）、无数据命令（CMD13 查询状态）。

R1 是 48-bit 标准响应（含 CRC 和状态标志），无 busy 信号。R1B 在 R1 基础上多了 DAT0 busy 信号——卡完成操作前持续拉低 DAT0。写操作（CMD24/CMD25）和 CMD6 使用 R1B，驱动必须等待 DAT0 释放才能发下一条命令。`MMC_CAP_WAIT_WHILE_BUSY` 使能此等待。

**Q5：DDR52 和 HS200 的本质区别是什么？** ★★★

```
DDR52:  时钟 52MHz，DDR（双沿采样）→ 8-bit × 52MHz × 2 = 104MB/s
HS200:  时钟 200MHz，SDR（单沿采样）→ 8-bit × 200MHz = 200MB/s
```

DDR52 不需要 tuning（频率低，数据有效窗口宽），HS200 必须 tuning（5ns 周期下信号延迟可能超过采样窗口）。DDR52 是 eMMC 的"稳妥模式"，HS200 是"性能模式"。

**Q6：eMMC 的 Boot 分区如何工作？Boot ROM 怎么读？** ★★★

上电或复位后 Host 发 CMD0 带 `BOOT_MODE` 标志（`CMD0_OP_BOOT_MODE`），eMMC 从 Boot 分区 1 自动发送数据。Boot 分区在 LBA 地址空间中不可见——不能像普通分区一样 `read/write`，只能通过 CMD0 触发读取或通过 CMD6 `PARTITION_ACCESS` 切换后访问。Boot 分区大小由 EXT_CSD byte 226（`BOOT_SIZE_MULT`）定义，一般 4MB。

**Q7：RPMB 的 HMAC 认证流程是什么？** ★★★

RPMB（Replay Protected Memory Block）是 eMMC 中需要认证才能访问的安全区域。流程：Host 生成 HMAC-SHA256（使用 eMMC 芯片生产时烧录的密钥）→ CMD25 写入数据 + HMAC → eMMC 校验 HMAC → 校验通过才写入。密钥在芯片生产时写入 OTP（一次性可编程），不可读回。Linux 中通过 `/dev/mmcblkXrpmb` 访问，只能 ioctl，不能 open/read/write。

---

## 2.3 架构与数据结构

对应：03-Architecture — MMC 子系统三层架构设计。

**Q8：画出 MMC 子系统的三层架构图，说明每层的职责。** ★★

```
块设备层（block层）   mmc_block.c      —— 注册块设备、处理 bio、翻译为 MMC 请求
MMC 核心层（core层）  mmc.c/mmc_ops.c   —— 卡探测、协议处理、命令封装
主机控制器层（host层） mmci.c           —— 实际的寄存器操作、中断处理、DMA
```

每层通过结构体指针联系：`mmc_host` 持有 `mmc_host_ops`（core→host 的回调），`mmc_card` 挂载在 `mmc_host` 上。host 层不关心卡是什么类型，只执行 core 层下发的命令。

**Q9：`mmc_host` 和 `mmc_card` 的关系是什么？生命周期谁管理？** ★★★

`mmc_host` 由 host 控制器驱动在 probe 时通过 `mmc_alloc_host()` 分配，代表一个控制器硬件实例。`mmc_card` 在 `mmc_rescan` → `mmc_init_card` 成功后由 core 层分配，代表一张具体的 eMMC/SD 卡。host 可能没有 card（空插槽），但 card 一定有 host 指针。生命周期：host 控制器驱动控制 host 的创建和销毁（`mmc_add_host` / `mmc_remove_host`），card 的生命周期由 core 层管理（探测到插入时创建，探测到移除时销毁）。

**Q10：`mmc_host_ops` 中有哪些关键回调？** ★★

- `.request()` —— 下发命令请求（最核心，MMC 核心层发 CMD 时调用）
- `.set_ios()` —— 设置 IO 状态（时钟频率、总线宽度、电压）
- `.execute_tuning()` —— 执行 HS200 采样调相
- `.card_busy()` —— 查询 MMC 卡是否忙（用于 R1B 等待）
- `.hw_reset()` —— 硬件复位 eMMC

**Q11：`mmc_request` 包含哪三个组成部分？** ★★★

```c
struct mmc_request {
    struct mmc_command *sbc;     // 前设命令（CMD23，设置块数）
    struct mmc_command *cmd;     // 主命令（CMD18/CMD25 等）
    struct mmc_data *data;       // 数据传输描述（SG 表、块数、方向）
};
```

三个组成部分的典型关系：`sbc` 设块数 → `cmd` 发读/写命令 → `data` 描述数据 buffer 位置。在 CQHCI 模式下 `sbc` 和 `cmd` 合并编码到 Task Descriptor 中，不再单独出现在 MMC 总线上。

---

## 2.4 初始化流程

对应：04-SourceAnalysis — MMC 卡探测、设备注册、分区创建。

**Q12：从 DTS 匹配到 `/dev/mmcblk1` 出现，经历了哪几个阶段？** ★★★

```
① AMBA 匹配 → mmci_probe() → mmc_alloc_host + mmc_add_host
② mmc_rescan() → 检测卡插入状态
③ mmc_init_card() → CMD 序列探测卡能力（CMD0→CMD1→CMD2→CMD3→CMD9→CMD7→CMD6）
④ mmc_add_card() → 注册到 MMC 总线
⑤ mmc_blk_probe() → 注册块设备 → /dev/mmcblk1 出现
```

关键点：`mmc_rescan` 每 2 秒由 delayed_work 触发一次（`MMC_SCAN_DELAY`），检测到卡后执行完整的初始化链。

**Q13：`mmc_init_card` 的 CMD 序列是什么？** ★★★

```
CMD0  GO_IDLE_STATE          → 复位卡到空闲态
CMD1  SEND_OP_COND           → 获取卡工作电压范围（OCR 寄存器）
CMD2  ALL_SEND_CID           → 获取卡唯一标识（CID，所有卡同时响应）
CMD3  SET_RELATIVE_ADDR      → 分配 RCA（相对地址，后续用 RCA 寻址）
CMD9  SEND_CSD               → 获取 CSD 寄存器（卡规格）
CMD7  SELECT_CARD            → 选中卡（切换到 Transfer 状态）
CMD8  SEND_EXT_CSD           → 获取 EXT_CSD（eMMC 特有扩展信息）
CMD6  SWITCH                 → 设置时序模式和工作参数
```

`CMD1` 和 `CMD8` 之间可能经历多轮电压协商，host 逐级尝试支持的电压。CMD6 之后再发 CMD13 确认模式切换成功。

**Q14：EXT_CSD 的哪些字段决定了块设备分区？** ★★

| EXT_CSD 字节 | 名称 | 意义 |
|-------------|------|------|
| 212-215 | `SEC_COUNT` | eMMC 总扇区数（32-bit），决定了 `mmcblk1` 的块设备容量 |
| 226 | `BOOT_SIZE_MULT` | Boot 分区大小倍数（×128KB），一般 4MB（32 × 128KB） |
| 168 | `RPMB_SIZE_MULT` | RPMB 分区大小倍数（×128KB），一般 512KB~4MB |

**Q15：RPMB 为什么不是块设备？** ★★★

RPMB 不能像普通块设备一样 open/read/write——它需要 HMAC 认证，认证密钥在芯片生产时烧录到 OTP，操作系统不可读。Linux 通过 `/dev/mmcblkXrpmb` 字符设备（不是块设备）暴露，只支持 `ioctl` 指令（`MMC_IOC_RPMB_CMD`），应用程序必须自己计算 HMAC 并封装为特定的帧结构。

---

## 2.5 数据通路

对应：05-VFS-to-Block（系统调用到 submit_bio）+ 06-IO-Path（submit_bio 到 eMMC NAND）。

**Q16：`dd if=/dev/mmcblk1 bs=4k count=1` 的数据流经过哪些内核层？** ★★★★

```
read() → VFS (blkdev_read_iter) → Page Cache (filemap_read)
  → 缺页 (filemap_get_pages) → submit_bio
  → blk-mq (mmc_mq_queue_rq) → 分配 tag
  → MMC core (mmc_start_request) → 发 CMD23 + CMD18
  → host 驱动 (mmci_request) → IDMA setup → dma_map_sg
  → 硬件 SDMMC2 发命令 → eMMC 返回数据 → IDMA 搬 DDR
  → DATAEND 中断 → mmci_cmd_irq → mmc_request_done
  → folio_unlock → copy_to_user → read() 返回
```

关键瓶颈在 Page Cache 缺页（如果 cache miss → 等 IO），如果 cache hit 直接 memcpy 返回，不经过块层。

**Q17：Buffered write 为什么比 read 快？** ★★★

Buffered write 数据写到 Page Cache 就返回了（`__folio_mark_dirty`），真正的 IO 由后台 `wb_workfn` 异步刷盘。从 write() 返回到落盘通常间隔几十毫秒甚至更久。Read 在 cache miss 时必须同步等 IO 完成——进程睡眠在 `folio_wait_locked` 上，直到 DATAEND 中断唤醒。

**Q18：`sync` 保证数据落盘吗？** ★★★★★

不保证。`sync` 只保证脏页从 Page Cache 写入 eMMC（通过 CMD25 写回），但 eMMC 内部有 Cache（SRAM），数据可能还在 eMMC Cache 中没写到 NAND。只有 `CMD6 FLUSH_CACHE`（EXT_CSD byte 32）发出并等 DAT0 busy 结束，eMMC Cache 中的数据才真正刷入 NAND。`sync` + 立即断电可能丢失最后几秒的数据。`REQ_FUA`（Force Unit Access）在单次 IO 中同时完成写入和刷 Cache。

**Q19：CMD6 FLUSH_CACHE 在哪个 EXT_CSD 字节？调用链是什么？** ★★★★★

EXT_CSD byte 32（`FLUSH_CACHE`）。调用链：

```
sync() → wb_workfn → blkdev_issue_flush
  → submit_bio(REQ_PREFLUSH) → mmc_blk_flush
    → _mmc_flush_cache(card)
      → mmc_switch(card, EXT_CSD_CMD_SET_NORMAL, FLUSH_CACHE, 1)
        → 发 CMD6 → 卡内部刷 Cache → 等待 DAT0 busy 结束
```

`mmc_switch` 内部包含 `MMC_CAP_WAIT_WHILE_BUSY` 等待逻辑。如果此 flag 没设（或硬件不支持 busy 检测），驱动的 busy timeout 可能覆盖不了完整刷 Cache 耗时（通常 100-200ms，大写入时可到秒级）。

**Q20：DMA 一致性在 MMC 读写中如何保证？** ★★★★

`dma_map_sg()` 的参数方向决定了 cache 维护操作：

```
读 (FROM_DEVICE): dma_map_sg 隐含 invalidate CPU Cache
                  → IDMA 写入 DDR → CPU 读时必须从 DDR 取（不读 stale cache）

写 (TO_DEVICE):   dma_map_sg 隐含 clean CPU Cache
                  → CPU 写入 DDR 的最新数据必须先刷出 cache
                  → IDMA 才能从 DDR 拿到正确数据
```

漏掉 cache 维护会出现"数据时对时错"的随机 bug——数据在 DDR 中是新的，但 CPU 读到的是 cache 中的旧值。`st,stm32mp25-sdmmc2` 的 variant_data 中 `dma_lli = true` 使用 DMA LLI 链表模式，每个 SG 段由一个 LLI 描述，`dma_map_sg` 在 setup 时统一完成 mapping。

**Q21：为什么 blk-mq 在 MMC 上是"伪多队列"？** ★★★

blk-mq 允许多个 submission queue（per-CPU 无锁提交），但 MMC 的 hardware queue 数量是 `nr_hw_queues = 1`。所有请求最终通过 `mmc_get_card` 串行化（`host->claimed` 互斥锁），同一时刻只能有一个请求在 MMC 总线上执行。blk-mq tag 系统仍然提供了背压和并发管理（最多 256 个 tag），但硬件执行层面永远是单队列。

**Q22：MMC 默认 IO 调度器是什么？** ★★

`mq-deadline`。MMC 是单队列块设备，blk-mq 模式下默认使用 mq-deadline（不是 CFQ 或 NOOP）。可以通过 `/sys/block/mmcblk1/queue/scheduler` 查看和切换。

---

## 2.6 CQHCI

对应：07-Advanced Part1 — 硬件命令队列。

**Q23：CQHCI 和传统软件路径的核心区别是什么？** ★★★

传统路径：CPU 逐条发 CMD23（块计数）→ 等待完成 → CMD18/CMD25（读/写）→ 等待数据结束，每笔请求至少 3 次 MMIO 写 + 2 次中断。

CQHCI：CPU 在 DRAM 中填充 Task Descriptor（编码 CMD23 + CMD18/25 参数）→ `wmb()` 保证描述符可见 → 一次 MMIO 写门铃寄存器（`CQHCI_TDBR`）。host 控制器自动读描述符、发命令、搬数据。完成时一次 TCC 中断携带多个完成通知。

**Q24：`dma_wmb()` 在 CQHCI 命令提交中的作用？** ★★★★

ARM 弱内存模型下，描述符写入 DRAM 和门铃写入 MMIO 可能被 CPU 重排——eMMC 可能在看到门铃时描述符还在 store buffer 中。`wmb()`（实际是 DSB 指令）强制所有之前的 store 全局可见后才执行之后的 store。没有这个屏障：描述符中 BLK_ADDR=0（还没写出去），eMMC 读到错误的 LBA，数据写到错误的扇区。x86 不需要这个屏障（TSO 强模型），这是 CQHCI 移植最容易被忽视的坑。

**Q25：CQHCI 最多支持多少个排队任务？** ★

硬件设计限制：`CQHCI_TDBR` 门铃寄存器是 32-bit 位图，所以 32 个 slot。DCMD 独占 slot 31，留给数据命令的 slot 是 0-30（共 31 个）。排队任务上限就是 31 个数据命令 + 1 个 DCMD。

---

## 2.7 Tuning

对应：07-Advanced Part2 — 高速模式调相与信号完整性。

**Q26：HS200 为什么需要 tuning？** ★★★

HS200 时钟 200MHz，周期 5ns，SDR（单沿采样）的有效采样窗口约 2.5ns。PCB 走线、I/O pad 延迟、温度漂移的总和可以轻松超过 2.5ns。host 不知道应该在什么时刻采样——采样点太早或太晚都读到错误数据。tuning 通过 DLYB 延迟链逐步移相，找到眼图中心的正确采样点。

**Q27：Tuning 失败后内核怎么做？** ★★★

```
mmc_init_card() → mmc_select_hs200() → mmc_hs200_tuning() → CMD21 扫描失败
  → 返回 -EIO → mmc_init_card 捕获错误
  → mmc_select_hs_ddr52() → 降级到 DDR52（104MB/s）
  → 卡正常工作，但速度减半
```

Tuning 失败后不会让卡无法使用，而是自动回退到不需要 tuning 的 DDR52 模式。可以在 `dmesg` 中看到 `mmc1: hs200 tuning failed` 和 `mmc1: DDR52 mode selected` 两条日志。

**Q28：Enhanced Strobe 为什么不需要 tuning？** ★★★

HS200 用 CLK 时钟采样数据——时钟由 host 发出，数据由 eMMC 发出，不同源。温度/电压变化时两者相对漂移，需要 tuning 重新对齐。

HS400 改用 DQS（Data Strobe）差分信号采样数据——DQS 和数据都由 eMMC 发出，走相同的物理路径。温度变化时 DQS 和数据一起漂移，相对位置不变。这是物理架构决定了 Enhanced Strobe 不需要 tuning。

---

## 3. 现场调试场景题

> 开放题，考察排查思路的完整性。没有标准答案，以下给出预期排查路径。

**场景 1：新板子启动后 `ls /dev/mmcblk*` 什么也没有，起点在哪？**

```
① dmesg | grep mmc — 卡是否被识别？"Card did not respond to voltage select"？
② cat /sys/kernel/debug/mmc1/ios — 时钟 0？电压 0？说明没检测到卡
③ DTS 检查：&sdmmc2 的 status="okay"? bus-width? vmmc/vqmmc supply?
   实际案例：vqmmc-supply 配的是 3.3V 但 eMMC 要求 1.8V
④ 硬件量信号：示波器看 SDMMC_CLK 有无时钟、CMD 线有无命令波形
⑤ 引脚 mux：检查 pinctrl 配置是否正确，特别是 d47_pins（数据线高 4 位）
```

**场景 2：eMMC 写性能只有预期的一半，怎么定位？**

```
① cat /sys/kernel/debug/mmc1/ios — 看 timing spec 是不是 DDR52？期望是 HS200？
② dmesg | grep mmc — 搜 hs200 tuning failed（卡回退到低速模式）
③ 区分读写：分别测读和写，如果读正常写慢 → 检查 write cache 配置
④ cat /sys/block/mmcblk1/queue/write_cache — "write back" 还是 "write through"
⑤ cat /sys/block/mmcblk1/queue/scheduler — 确认不是用了错误的调度器
```

**场景 3：突然掉电后 rootfs 损坏，软件层面怎么降低概率？**

```
① 确认 eMMC Cache 是否开启（默认开启）。开启时掉电可能丢最后几秒数据
② 文件系统挂载参数是否有 barrier？ext4 默认开 barrier，但旧内核可能没开
③ 关键数据是否使用 FUA 写入？REL_WRITE + FUA 组合防止 torn write
④ 检查硬件电路：eMMC 的 VCC 是否有足够大的退耦电容，掉电时能否维持到刷 Cache 完成
⑤ 方案：sync 间隔缩短 + 关键文件 O_SYNC + 内核配置 CONFIG_MMC_BLOCK_DEFERRED_RESUME
```