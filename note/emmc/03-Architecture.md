# 03. eMMC 核心架构：分层设计与核心数据结构

> 本文是 STM32MP257 eMMC 驱动深度分析系列的第 3 篇。  
> 对应框架步骤 ③：从架构层面理解 MMC 子系统，建立数据结构关系图。
>
> **前置:** [01-Usage.md](01-Usage.md) — sysfs/debugfs 基本操作  
> [02-Hardware.md](02-Hardware.md) — MMC 协议与传输模式  
> **下一篇:** [04-SourceAnalysis.md](04-SourceAnalysis.md)
>
> **字数：** 约 10,000 字  
> **建议阅读时间：** 25–35 分钟（含代码和图表）

---

## 3.1 分层架构全景

MMC 子系统是 Linux 内核中典型的"核心层 + 控制器驱动"分离架构。Core 层封装了卡探测、时序切换、请求排队等通用逻辑，host driver 只负责操作寄存器。二者的接口由 `struct mmc_host_ops` 定义——**只要实现这组回调，任何控制器都能接入 MMC 子系统**。比如把 SDMMC2 换成 DW-MMC，core 层的卡探测、块设备注册代码一行都不用改，但 host driver 本身仍要重新实现数千行：寄存器接口、DMA 映射、中断处理、调谐算法、时钟电源——这些全部是控制器相关的。

### 三层架构总图

MMC 子系统从设计上分为明确的三层，每层通过接口隔开：

```
┌──────────────────────────────────────────────────────────────────────┐
│  BLOCK 层 (I/O 调度 + 块设备)                    mmc_block.ko       │
│                                                                      │
│  ┌──────────────────────────────────────┐                           │
│  │  block.c                             │                           │
│  │    ● gendisk 注册 / 分区 / ioctl     │                           │
│  │    ● I/O 下发入口 (mmc_blk_issue_rq) │                           │
│  ├──────────────────────────────────────┤                           │
│  │  queue.c                             │                           │
│  │    ● blk-mq 请求队列管理              │                           │
│  │    ● mmc_queue_rq → mmc_start_req    │                           │
│  └──────────────────────────────────────┘                           │
│          │ 提交 mmc_request                                         │
│          ▼                                                          │
├──────────────────────────────────────────────────────────────────────┤
│  CORE 层 (框架 + 通用逻辑)                      mmc_core.ko         │
│                                                                      │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐            │
│  │ core.c │ │ host.c │ │ bus.c  │ │ mmc.c  │ │ mmc_ops│  ...       │
│  │        │ │        │ │        │ │        │ │ .c     │            │
│  │ 主机   │ │ host   │ │ 总线   │ │ eMMC   │ │ CMD    │            │
│  │ 生命   │ │ class  │ │ 管理   │ │ 初始化  │ │ 封装    │           │
│  │ 周期   │ │ 管理   │ │        │ │        │ │        │            │
│  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘            │
│          │                                                          │
│          │  host->ops->request/set_ios/execute_tuning...            │
│          ▼                                                          │
├──────────────────────────────────────────────────────────────────────┤
│  HOST 层 (寄存器操作 + 中断 + DMA)             mmci-pl18x           │
│                                                                      │
│  ┌──────────────────────────────────────────────┐                   │
│  │ mmci.c  (ARM PrimeCell PL180/181 通用部分)    │                   │
│  │   ● AMBA 总线 probe/resume                    │                   │
│  │   ● 寄存器读写 (MCI_CMD/MCI_DAT/MCI_MASK0)    │                   │
│  │   ● 中断处理 / PIO 传输                        │                   │
│  ├──────────────────────────────────────────────┤                   │
│  │ mmci_stm32_sdmmc.c  (STM32MP2 变体)           │                   │
│  │   ● SDMMC 延迟线校准 (DLYB)                    │                   │
│  │   ● IDMA 链表描述符 (sdmmc_lli_desc)           │                   │
│  │   ● STM32MP25 特定配置 (DTS compatible)        │                   │
│  └──────────────────────────────────────────────┘                   │
│          │                                                          │
│          ▼  写寄存器                                                 │
│  ┌──────────────────────────────────────────────┐                   │
│  │ STM32MP2 SDMMC2 控制器硬件                     │                   │
│  │   时钟 166MHz · 8-bit · DDR52 · IDMA          │                   │
│  └──────────────────────────────────────────────┘                   │
└──────────────────────────────────────────────────────────────────────┘
```

从上往下每层的定位和作用如下：

---

**BLOCK 层**（`mmc_block.ko`）—— 把 eMMC 变成 `/dev/mmcblk1` 的驱动。

在裸机编程中读写 eMMC 就是调用一个 `ReadSector(addr, buf)` 函数，直接往 SDMMC 控制器寄存器里填命令、等中断、读数据。

Linux 不这么干。它把"能按块（sector）随机读写的设备"抽象成一类——**块设备**。和字符设备（如串口 `/dev/ttySTM0`，一个字节一个字节流式读写）不同，块设备的特点是：
- 按固定大小块访问（eMMC 是 512 字节一块）
- 可以随机寻址（读第 1000 块不依赖第 999 块）
- 有缓存层和调度层（多个进程同时读写时排队合并）

一个块设备驱动在内核里主要由两个东西组成：

```
块设备驱动 = gendisk + request_queue
              │            └ 存放 I/O 请求的队列
              └ 代表"一块磁盘"的内核对象
                  含分区表、容量、设备号等
```

`mmc_block.ko` 就是这样一个块设备驱动。它在 ATK 板上注册的 `gendisk` 就是 `/dev/mmcblk1`，每个分区对应 `/dev/mmcblk1p1`、`/dev/mmcblk1p2` 等。它的工作原理如下：

```
文件系统（ext4 / fat32 …）
  ↓  读/写某个文件 → 换算成 sector 号
blk-mq（通用块 I/O 调度层）
  ↓  分发请求（struct request）
┌─ mmc_block.ko ──────────────────┐
│  queue.c：           block.c：   │
│  接收 blk-mq        注册 gendisk│
│  分发的 request，    （含分区扫  │
│  包装成 MMC 层       描、ioctl） │
│  能读懂的            负责统计   │
│  mmc_request         I/O 计数   │
└────────┬────────────────────────┘
         ↓ 提交 mmc_request 到 core 层
```

所以 `mmc_block` **是 MMC 子系统内的模块**——它认得 `mmc_request`，因为这是它和 core 层之间的通信接口。但它**不需要知道**卡是 eMMC 还是 SD 卡，也不需要知道控制器是 SDMMC2 还是 DW-MMC。

---

**CORE 层**（`mmc_core.ko`）—— 子系统的"大脑"。

卡探测、命令序列编排、时序协商（跑 400KHz 还是 52MHz）、电压切换、命令重试——所有通用逻辑都在这里。core.c 是主入口，管理主机生命周期和 I/O 路径；host.c 管理 host class 设备（sysfs）；bus.c 管理 mmc 总线；mmc.c 实现了 eMMC 卡完整的初始化序列（CMD1→CMD2→CMD3→…）。这些代码**一次写好，所有控制器通用**。

它不直接操作寄存器，而是通过 `host->ops->xxx()` 调 host driver。如果某步操作失败，core 层决定是否重试、是否降速、是否报错——host driver 只管"执行"不负责"决策"。

---

**HOST 层**（`mmci-pl18x`）—— 干"体力活"的。

core 层说"发 CMD1"，host driver 就写 SDMMC2 的 `MCI_CMD` 寄存器；core 层说"查 DAT0 忙不忙"，它就读 `MCI_STATUS`。它操作的是物理地址 `0x48230000`（ATK 板上 SDMMC2 的寄存器基址），处理中断、搬运 DMA 数据。

mmci.c 是 ARM PrimeCell PL180/181 的通用部分——所有使用这个 IP 核的 SoC 都能用。`mmci_stm32_sdmmc.c` 是 STM32MP2 的变体文件，专门处理该 SoC 独有的两个硬件特性。

**DLYB（延迟线校准）**：eMMC 跑到 DDR52（52MHz 双倍数据率）或更高频率时，数据信号往返极快，控制器必须在一个很小的窗口内正确采样信号。DLYB 是一个硬件校准模块，它能动态调整采样延迟，找到最稳定的采样点。这就像调整相机快门时机来抓拍高速运动物体——偏一点就花了。

**IDMA（内部 DMA）**：如果没有 DMA，CPU 要从 SDMMC 控制器的 FIFO 里一个字节一个字节地读数据（PIO 模式），DDR52 下 8-bit 总线每秒产生约 100MB 的数据，CPU 根本忙不过来。IDMA 是 SDMMC 控制器内置的 DMA 引擎，能直接在控制器 FIFO 和内存之间搬运数据，不需要 CPU 逐字节复制。驱动只需准备好一份"链表"（`sdmmc_lli_desc`），描述每个数据块在内存中的位置和长度，剩下的硬件自动完成。

---

**分层的价值**：三层之间不能越级调用。BLOCK 层不直接调 HOST 层，HOST 层也不直接向 BLOCK 层发数据——全靠 CORE 层中转。

从 Makefile 可以验证这种依赖关系：

```makefile
# drivers/mmc/core/Makefile
obj-$(CONFIG_MMC)        += mmc_core.o
mmc_core-y               := core.o bus.o host.o mmc.o mmc_ops.o \
                             sd.o sd_ops.o sdio.o sdio_ops.o ...
obj-$(CONFIG_MMC_BLOCK)  += mmc_block.o     # ← 独立模块，链接 mmc_core
mmc_block-objs           := block.o queue.o

# drivers/mmc/host/Makefile
obj-$(CONFIG_MMC_ARMMMCI) += mmci.o         # ← 独立模块，链接 mmc_core
mmci-y                     := mmci.o mmci_stm32_sdmmc.o
```

### 两层回调：子系统设计的核心

分层架构不能靠约定来保证——需要一套机制强制 BLOCK、CORE、HOST 之间不越级调用。MMC 子系统的做法是：**core 层不直接调用其他模块的函数，全部通过 ops 结构体间接调用**。

而且这里有两张 ops 表，分管两件不同的事：

| | `mmc_host_ops` | `mmc_bus_ops` |
|--|----------------|---------------|
| **管什么** | 操作硬件寄存器 | 管理卡的生命周期 |
| **谁实现** | host driver（不同控制器不一样） | 卡类型驱动（eMMC/SD/SDIO 不一样） |
| **什么时候填入** | probe 时，一次填好 | 卡探测到后才填 |
| **换个实现会怎样** | 换一个控制器（如 SDMMC2 → DW-MMC） | 换一种卡（如 eMMC → SD 卡） |

**为什么是两张而不是一张？** 因为"操作寄存器"和"管理卡"是两件独立的事。一张卡插在不同的控制器上，卡管理逻辑不用改；同一个控制器换了不同卡，寄存器操作逻辑也不用改。分成两张表，core 层组合使用，互不耦合：

```
core 层调 host driver（操作硬件）：         core 层调卡类型（管理卡）：
  host->ops->request(host, mrq)  \\ 发命令   host->bus_ops->detect(host)  \\ 检测卡是否存在
  host->ops->set_ios(host, ios)  \\ 调时钟   host->bus_ops->remove(host)  \\ 移除卡
  host->ops->execute_tuning()    \\ 调信号   host->bus_ops->suspend(host) \\ 卡休眠
  host->ops->card_busy(host)     \\ 查忙     host->bus_ops->resume(host)  \\ 卡唤醒
```

左边四个函数解决的是"怎么写寄存器"的问题：

- `request` —— core 层要发一条命令（比如 CMD18 读数据），于是调 request()。host driver 把命令写到 `MCI_CMD` 寄存器，把数据参数写到 `MCI_DAT` 寄存器，然后启动传输。不同的控制器写寄存器的方式完全不同，所以归 host_ops。
- `set_ios` —— core 层决定把时钟从 400KHz 升到 52MHz，于是调 set_ios()。host driver 重新计算分频系数，写 `MCI_CLKCR`。电压切换也在这里做。这也是硬件相关的。
- `execute_tuning` —— eMMC 进入 DDR52 模式后，core 层要求微调采样延迟。host driver 操作 DLYB 模块的寄存器，不断试不同的延迟值，找一个最佳的。
- `card_busy` —— 发完命令后，core 层需要知道卡是否还在忙（DAT0 信号是否被拉低）。host driver 读 `MCI_STATUS` 寄存器的相应位。

右边四个函数解决的是"卡发生了什么"的问题：

- `detect` —— core 层想知道卡还在不在。对于 eMMC（不可移除），直接返回"还在"。对于 SD 卡，可能需要读卡检测 GPIO。
- `remove` —— 卡被拔出时，core 层调 remove() 清理卡相关的资源。不同的卡类型清理方式不同（SDIO 还要断开 function 驱动）。
- `suspend/resume` —— 系统进入休眠时，不同的卡类型需要发不同的休眠命令、保存不同的上下文，所以归 bus_ops。

总结：**host_ops = 怎么操作硬件，bus_ops = 怎么管理卡。** core 层既不碰寄存器也不直接管卡——它通过这两张表调度两端的工作。

这两张 ops 表的定义位置和填充时机：

| Ops 表 | 定义位置 | 谁实现 | 谁来填 |
|--------|----------|--------|--------|
| `struct mmc_host_ops` | `include/linux/mmc/host.h:115` | 主机控制器驱动 | host driver probe 时填入 `mmc->ops` |
| `struct mmc_bus_ops` | `drivers/mmc/core/core.h:20` | 卡设备类型（mmc/sd/sdio） | `mmc_attach_xxx()` 时调用 `mmc_attach_bus()` 填入 |

`mmc_host_ops` 的典型实现者（以 STM32MP2 为例）：

```c
// drivers/mmc/host/mmci.c
static const struct mmc_host_ops mmci_ops = {
    .request        = mmci_request,         // 发一条命令并传输数据
    .set_ios        = mmci_set_ios,         // 改时钟/电压/总线宽度
    .get_ro         = mmci_get_ro,          // 卡是否写保护（SD 卡用）
    .get_cd         = mmci_get_cd,          // 卡是否插着（eMMC 固定返回 1）
    .card_busy      = mmci_card_busy,       // DAT0 线是否忙
    .execute_tuning = mmci_execute_tuning,  // 微调采样延迟
    ...
};
```

每个回调的触发场景如下：

| 回调 | core 层什么时候调它 | 它做什么 |
|------|-------------------|----------|
| `request` | 每次读写 eMMC 时 | 把 `mmc_request` 中的命令填到 `MCI_CMD`、数据参数填到 `MCI_DAT`，然后启动传输 |
| `set_ios` | 初始化阶段改时钟（400KHz→52MHz），切电压（3.3V→1.8V）时 | 重新计算分频系数写 `MCI_CLKCR`，写 `MCI_POWER` 控制上电/电压 |
| `get_ro` | 用户执行 `mount` 或 `open` 时检查 | 读 GPIO 或寄存器判断 SD 卡写保护开关。eMMC 没有写保护，固定返回 0（可写） |
| `get_cd` | 系统启动时、热插拔事件时 | 卡是否物理存在。eMMC 焊在板子上，固定返回 1（存在） |
| `card_busy` | 发完带有 R1B 响应的命令后，core 层需要等卡就绪 | 读 `MCI_STATUS` 寄存器的 DAT0 位——低电平表示卡还在忙 |
| `execute_tuning` | 进入 DDR52/HS200 等高倍速模式后 | 操作 DLYB 模块：发一系列测试数据，试不同的延迟值，选误码率最低的那个 |

---

`mmc_bus_ops` 的典型实现者（以 eMMC 为例）：

```c
// drivers/mmc/core/mmc.c
static const struct mmc_bus_ops mmc_ops = {
    .remove         = mmc_remove,           // 卡被移除时清理
    .detect         = mmc_detect,           // 检查卡是否还在
    .suspend        = mmc_suspend,          // 系统休眠→卡休眠
    .resume         = mmc_resume,           // 系统唤醒→卡唤醒
    .runtime_suspend = mmc_runtime_suspend, // 空闲时关卡时钟
    .runtime_resume = mmc_runtime_resume,   // 要读写时重新开时钟
    .alive          = mmc_alive,            // 发 CMD13 看卡是否响应
    .shutdown       = mmc_shutdown,         // 关机前通知卡
    .hw_reset       = mmc_hw_reset,         // 拉硬件复位引脚
    .sw_reset       = mmc_sw_reset,         // 发 CMD0 软复位
    .cache_enabled  = mmc_cache_enabled,    // eMMC 内部缓存开了没
    .flush_cache    = mmc_flush_cache,      // 写回 eMMC 内部缓存
};
```

| 回调 | core 层什么时候调它 | 它做什么 |
|------|-------------------|----------|
| `remove` | 卡被拔出时 | 清理设备模型中的卡设备、释放资源 |
| `detect` | 周期性重扫描时 | 发 CMD13 读卡状态，如果没响应则认为卡已移除 |
| `suspend/resume` | `echo mem > /sys/power/state` 时 | 发 CMD5 让卡进入休眠，唤醒时重新初始化 |
| `runtime_suspend/resume` | 系统空闲时自动触发 | 关闭 eMMC 的时钟节省功耗，读写前重新打开 |
| `alive` | quick 检测卡是否还活着 | 发 CMD13 读状态寄存器，不报错就算活着 |
| `shutdown` | `poweroff` 关机时 | 发 CMD6 关闭 eMMC 写保护等功能后通知卡断电 |
| `hw_reset` | 卡死机时（需要硬件支持） | 拉低 eMMC 的 RST_n 引脚再释放 |
| `sw_reset` | 卡协议异常时 | 发 CMD0（GO_IDLE），让卡回到初始状态 |
| `cache_enabled` | core 层决定是否发 flush 前 | 检查 EXT_CSD 的缓存使能位 |
| `flush_cache` | 关机或同步（sync）时 | 发 CMD6 触发 eMMC 内部缓存写回，防止丢数据 |

### 跨层初始化流程

从系统启动到 eMMC 可用的完整调用链：

```
┌────────────────────────────────────────────────────────────────────┐
│ 阶段 1: Core 层注册 (subsys_initcall)                              │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│ subsys_initcall(mmc_init)          ← drivers/mmc/core/core.c:2365 │
│  ├─ mmc_register_bus()             ← 注册 mmc_bus_type            │
│  ├─ mmc_register_host_class()      ← 注册 /sys/class/mmc_host     │
│  └─ sdio_register_bus()            ← 注册 sdio_bus_type           │
│                                                                    │
├────────────────────────────────────────────────────────────────────┤
│ 阶段 2: Host driver probe (module_init)                            │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│ module_amba_driver(mmci_driver)    ← drivers/mmc/host/mmci.c:2699 │
│  └─ mmci_probe(amba_device)        ← 匹配 DTS 中 arm,pl18x       │
│       ├─ mmc = mmc_alloc_host(...) ← 分配 mmc_host，含私有数据     │
│       ├─ mmc->ops = &mmci_ops      ← 填充 mmc_host_ops（关键！）  │
│       ├─ mmc_of_parse(mmc)         ← 解析 DTS 属性                │
│       │   (bus-width, cap-*, max-frequency...)                    │
│       ├─ 初始化时钟、IRQ、DMA...                                   │
│       └─ mmc_add_host(mmc)         ← 注册到 core 层              │
│            └─ mmc_start_host()                                   │
│                 ├─ mmc_power_up()  ← 上电 (vmmc/vqmmc)           │
│                 └─ _mmc_detect_change() → 调度 host->detect 队列  │
│                      └─ mmc_rescan() ← 核心探测入口              │
│                                                                    │
├────────────────────────────────────────────────────────────────────┤
│ 阶段 3: 卡探测 (mmc_rescan → mmc_attach_mmc)                      │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│ mmc_rescan()                        ← drivers/mmc/core/core.c    │
│  └─ mmc_rescan_try_freq(host, 400000)  ← 从 400KHz 开始扫描      │
│       └─ mmc_attach_mmc(host)        ← 识别为 eMMC 卡             │
│            ├─ mmc_attach_bus(host, &mmc_ops)  ← 填入 bus_ops      │
│            ├─ mmc_init_card(host, &card)                          │
│            │   ├─ CMD1  ← mmc_send_op_cond() 获取 OCR            │
│            │   ├─ CMD2  ← mmc_all_send_cid()  获取 CID            │
│            │   ├─ CMD3  ← mmc_set_relative_addr() 设置 RCA       │
│            │   ├─ CMD9  ← mmc_send_csd()  获取 CSD                │
│            │   ├─ CMD7  ← mmc_select_card() 选卡                  │
│            │   ├─ CMD8  ← mmc_send_ext_csd()  获取 EXT_CSD        │
│            │   ├─ 时序协商 (DDR52/HS200/HS400)                    │
│            │   └─ 电压切换 (3.3V → 1.8V 如需)                     │
│            └─ mmc_add_card(host->card)    ← 注册到 mmc_bus_type   │
│                 └─ device_add(&card->dev) → 触发 mmc_blk_probe    │
│                                                                    │
├────────────────────────────────────────────────────────────────────┤
│ 阶段 4: 块设备注册 (mmc_blk_probe)                                │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│ mmc_blk_probe(card)                 ← drivers/mmc/core/block.c   │
│  ├─ alloc_disk()                    ← 分配 gendisk                │
│  ├─ blk_mq_alloc_queue()            ← 分配 blk-mq 请求队列       │
│  ├─ mmc_init_queue()                ← 初始化队列                  │
│  └─ device_add_disk()               ← 注册 /dev/mmcblk1          │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

### 各阶段涉及的设备与驱动

这个流程涉及两条总线和三个设备/驱动：

| 阶段 | 总线 | 设备 | 驱动 | 创建的软件对象 |
|------|------|------|------|---------------|
| 1 | — | — | — | `mmc_bus_type`、`mmc_host_class` |
| 2 | **AMBA 总线**（DTS compatible `"arm,primecell"`） | SDMMC2 控制器 | `mmci_driver`（`mmci-pl18x`） | `mmc_host` |
| 3 | — | eMMC 芯片 | —（core 层主动探测） | `mmc_card` |
| 4 | **MMC 总线**（`mmc_bus_type`） | `mmc_card` | `mmc_block`（`mmc_blk_probe`） | `gendisk` = `/dev/mmcblk1` |

> ⚠️ **同一个物理芯片，三层抽象：**
> 
> ```
> 物理 eMMC 芯片       硬件（板子上焊的）
>     ↓  软件抽象
> mmc_card            MMC 子系统的设备对象，挂在 mmc_bus_type 上
>                     → /sys/bus/mmc/devices/mmc1:0001/
>     ↓  块设备封装
> /dev/mmcblk1        用户态接口，block 类设备
>                     → /sys/block/mmcblk1/
> ```
> 
> `mmc_card` 不是硬件——它和 `/dev/mmcblk1` 都是软件，只是处于不同层次：`mmc_card` 是"驱动层面"的设备（驱动匹配的对象），`/dev/mmcblk1` 是"用户层面"的接口（open/read/write 的对象）。`mmc_blk_probe` 负责在这两层之间搭桥：收到一个 `mmc_card`，创建一个 `gendisk`（即 `/dev/mmcblk1`）。

下面逐一解释每个阶段为什么是必要的。

---

**阶段 1：Core 层必须先注册**

Linux 设备模型要求：一个总线类型必须先注册，然后设备才能在上面注册、驱动才能在上面匹配。所以 `mmc_init` 必须最先执行——它被标记为 `subsys_initcall`，优先级高于 `module_init`。

为什么注册三样东西？

- `mmc_bus_type`：一条名叫 `mmc` 的总线。卡设备（`mmc_card`）将注册在这条总线上，`mmc_block` 也注册在这条总线上，后续靠总线匹配机制让 `mmc_block` 驱动识别到卡设备。
- `mmc_host_class`：一个设备类，出现在 `/sys/class/mmc_host/` 下。每个 SDMMC 控制器对应一个 `mmcX` 目录，用户可以通过 sysfs 查看控制器状态。
- `sdio_bus_type`：另一条总线，专给 SDIO 设备用。SDIO 的协议流程和 eMMC/SD 不同，需要独立的总线管理。

**如果 core 层没注册好，host driver 去调 `mmc_alloc_host()` 时系统里根本没有 MMC 相关的设备模型基础设施，会直接失败。**

---

**阶段 2：Host driver probe 时不能只分配 mmc_host，还要填 ops**

`mmci_probe` 里做的 5 件事有严格顺序：

1. **`mmc_alloc_host` 分配 `mmc_host`** —— 这个结构体由 core 层管理，但 host driver 需要自己的私有数据（寄存器基址、时钟句柄、IDMA 状态等）。内核用一种"一次分配"的技巧：`mmc_alloc_host(sizeof(struct mmci_host), ...)` 把 `mmc_host` 和 `mmci_host` 分配在连续内存里。这样释放时只需 `mmc_free_host`，core 层自动释放私有数据，不会泄漏。

2. **`mmc->ops = &mmci_ops` 填入回调表** —— 这是最关键的一步。之后 core 层所有"操作硬件"的需求都通过这张表调回来。**如果在填 ops 之前调 `mmc_add_host`，core 层启动卡探测时调用 `host->ops->request()` 会拿到 NULL 指针，内核直接 panic。**

3. **`mmc_of_parse` 读 DTS** —— 从设备树解析 `bus-width`、`non-removable`、`max-frequency` 等属性，写入 `mmc->caps` 和 `mmc->caps2`。这些能力位决定后续 core 层在卡探测时尝试哪些时序模式。

4. **初始化时钟、IRQ、DMA** —— 使能外设时钟、注册中断处理函数、准备 DMA 通道。这些是硬件能工作的前提条件。

5. **`mmc_add_host` —— 把控制权交给 core 层**：这个函数会调用 `mmc_start_host()` → `mmc_power_up()` 给卡上电（`vmmc` 和 `vqmmc` 两个电源域），然后通过 `_mmc_detect_change()` 调度一个工作队列，最终触发 `mmc_rescan()`。从这里开始，host driver 不再主动做事——它只等 core 层调它的 ops 回调。

**为什么 `mmc_add_host` 之前就要上电？** 因为 `mmc_rescan` 需要发命令检测卡，而卡需要供电才能响应。`mmc_power_up` 就是开 `vmmc`（核心电源 3.3V）和 `vqmmc`（信号电压 3.3V 或 1.8V）两个 regulator。

---

**阶段 3：卡探测——一个"先问是不是，再问是谁，再协商速度"的过程**

**为什么从 400KHz 开始？** MMC 协议规定所有卡上电后都必须能工作在 400KHz（这是最低兼容频率）。不管是什么卡、什么质量，400KHz 一定能通信。确认卡存在之后，core 层再逐步拉升频率。

进入 `mmc_attach_mmc` 后的每一步都有明确目的：

- **`mmc_attach_bus(host, &mmc_ops)` 填入 bus_ops** —— 告诉 core 层："这张卡是 eMMC 类型，以后卡生命周期的事（detect、remove、suspend）请调 `mmc_ops` 里的函数。"

- **CMD1（`mmc_send_op_cond`）获取 OCR** —— OCR 寄存器告诉 host：卡工作在哪个电压范围、是否支持高容量（sector 寻址）。如果电压不兼容，后续不用继续了，直接报错。**这是"先问是不是"——电压都不匹配，后面别谈了。**

- **CMD2（`mmc_all_send_cid`）获取 CID** —— CID 是卡的唯一序列号，包含制造商 ID、产品名、序列号等。这个信息供调试和 quirks 匹配用（比如发现某个型号的卡有 bug，内核可以针对它的 CID 做特殊处理）。

- **CMD3（`mmc_set_relative_addr`）设置 RCA** —— 给卡一个"短地址"（RCA），后续命令用这个地址而不是完整的 CID 来指代这张卡。**这对应"再问是谁"——先知道是谁，再给个短名，后面好称呼。**

- **CMD9（`mmc_send_csd`）获取 CSD** —— CSD 包含卡的物理参数：容量、块大小、最大时钟、命令类等。core 层读到这些信息后才能判断"这个卡支持什么操作"。

- **CMD7（`mmc_select_card`）选卡** —— 把卡从待机状态切换到传输状态。进入传输状态后才能发数据相关的命令。**这是一个状态切换，不是数据传输。**

- **CMD8（`mmc_send_ext_csd`）获取 EXT_CSD** —— EXT_CSD 是 512 字节的扩展寄存器，包含时序协商所需的所有信息：支持哪些模式（HS/DDR/HS200/HS400）、缓存大小、分区配置、健康状态等。**时序协商必须读完 EXT_CSD 才能进行。**

- **时序协商——双方对表**：core 层检查 `card->ext_csd` 里卡支持的模式，再对照 `host->caps/caps2` 里控制器支持的模式，取双方交集。比如卡支持 HS400，但 DTS 里没配 `mmc-hs400-1_8v`（即 `caps2` 没有 `MMC_CAP2_HS400`），那就降到 HS200 或 DDR52。协商完毕后，core 层发 CMD6（SWITCH）让卡切换到选定的模式，同时调 `host->ops->set_ios()` 让 host driver 调整控制器寄存器的时钟和时序设置。

- **电压切换（如需要）** —— DDR52 之后的模式（HS200/HS400）要求信号电压从 3.3V 降到 1.8V。这一步涉及信号级转换，不能只是写寄存器——core 层先切 host 端电压，然后发 CMD6 告诉卡也切，最后确认双方都在 1.8V。

全部完成后，`mmc_add_card` 把 `card->dev` 注册到 `mmc_bus_type` 总线上。这一注册触发 Linux 设备模型的核心机制：总线有新设备，总线上的驱动（`mmc_block`）有机会 claim 这个设备——于是 `mmc_blk_probe` 被调用。

---

**阶段 4：块设备注册——为什么最后一步才是 /dev/mmcblk1**

`mmc_blk_probe` 的顺序也有讲究：

1. **`alloc_disk` 分配 gendisk** —— gendisk 代表一块磁盘。必须先有它，才能在上面挂队列和分区。
2. **`blk_mq_alloc_queue` 分配请求队列** —— blk-mq 是 Linux 最新的块 I/O 框架，支持多队列并发。队列是"文件系统→块设备驱动"之间的管道。
3. **`mmc_init_queue` 绑定队列和 gendisk** —— 指定队列的参数（最大请求数、最大段数等）来自 `mmc_host` 的 `max_segs`、`max_blk_size` 等字段。这些在阶段 2 的 `mmc_of_parse` 时就已经设定好了。
4. **`device_add_disk` 让 /dev/mmcblk1 出现** —— 这个函数创建块设备文件、扫描分区表、为每个分区创建 `/dev/mmcblk1pX`。**只有这步之后用户才能用 `mount` 访问 eMMC。**

**为什么 `device_add_disk` 最后？** 因为在此之前，内核还没有准备好处理 I/O 请求。如果在队列没有初始化完成时就暴露设备文件，用户进程的读写请求会落到一个未就绪的驱动上，导致内核报错或数据损坏。

### core/ 目录下各文件的角色

| 文件 | 角色 | 参与阶段 |
|------|------|----------|
| `core.c` | 模块入口、主机生命周期、I/O 路径、电源时序 | 阶段 1, 3 |
| `host.c` | host class 设备管理、retune 框架 | 阶段 1, 2 |
| `bus.c` | mmc_bus_type 注册、卡设备 sysfs | 阶段 1, 3 |
| `mmc.c` | eMMC 卡探测/初始化、时序选择 | 阶段 3 |
| `mmc_ops.c` | eMMC 命令封装 (CMD6/CMD8/send_ext_csd 等) | 阶段 3 |
| `sd.c` | SD 卡探测/初始化 | SD 卡探测 |
| `sdio.c` | SDIO 卡探测/初始化 | SDIO 卡探测 |
| `block.c` | gendisk 注册、ioctl、I/O 下发 | 阶段 4 |
| `queue.c` | blk-mq 队列管理、并发控制 | 阶段 4 |
| `debugfs.c` | debugfs 接口 | 运行时 |
| `regulator.c` | 电源管理辅助 | 阶段 2, 3 |
| `slot-gpio.c` | CD/WP GPIO 检测 | 阶段 2 |

### 两条总线与各组件的协作关系

> ⚠️ **两条总线是平行独立的**，不存在谁挂在谁下面。sysfs 里看到 `.../48230000.mmc/mmc_host/mmc1/mmc1:0001/` 这个长路径是**设备父子链**（物理拓扑：SDMMC2 控制器是 SoC 片上总线的一个外设），不是总线嵌套。AMBA 总线和 MMC 总线是同一层级的两种总线类型，各自管理自己的设备和驱动匹配，互不隶属。

MMC 子系统的初始化过程涉及两条独立的平行总线，各自承载不同的设备和驱动：

```
┌─────────────────────────────────────────────────────────────────────────┐
│  AMBA 总线                              MMC 总线                         │
│  (匹配 DTS compatible="arm,primecell")  (mmc_bus_type)                   │
│                                                                          │
│  设备: SDMMC2 控制器                     设备: mmc_card                   │
│  驱动: mmci_driver (mmci-pl18x)          驱动: mmc_block                  │
│                                                                          │
│  职责: 操作 SDMMC2 寄存器                 职责: 处理 MMC 协议              │
│        request / set_ios / tuning              把块请求转 mmc_request    │
│                                                                          │
│         ↓ probe                                       ↑                 │
│  创建 mmc_host ────────── core 层中转 ──────── 注册 mmc_card              │
│                      (mmc_core.ko)                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

整个初始化链中涉及的主要组件：

| 组件 | 类型 | 所在总线/类 | 职责 |
|------|------|-----------|------|
| SDMMC2 控制器 | 硬件外设 | AMBA 总线 | 操作寄存器、中断、DMA |
| mmci_driver | 驱动 | AMBA 总线 | 驱动 SDMMC2 控制器 |
| mmc_host | core 层管理对象 | mmc_host_class | 调度 I/O、管理卡生命周期 |
| mmc_card | MMC 子系统设备 | MMC 总线（mmc_bus_type） | 代表 eMMC 芯片 |
| mmc_block | 驱动 | MMC 总线（mmc_bus_type） | 驱动 mmc_card，创建块设备 |
| /dev/mmcblk1 | 块设备接口 | block 类 | 用户态块设备访问 |

其中真正的设备/驱动匹配关系有两组：AMBA 总线上 `mmci_driver` → SDMMC2 控制器，MMC 总线上 `mmc_block` → `mmc_card`。

**完整的 I/O 路径**（从用户读写到寄存器）：

```
应用程序:  read("/mnt/file", buf)
               ↓
VFS:        找到文件对应 inode → 底层块设备是 /dev/mmcblk1
               ↓
文件系统:   ext4 把文件偏移换算成 sector 号 → 提交 bio
               ↓
通用块层:   bio 排队 → blk-mq 生成 struct request
           放到 mmcblk1 的 request_queue 中
               ↓
mmc_block:  queue.c 的 .queue_rq 回调被触发
           把 struct request 转为 mmc_request
           → mmc_start_req()
               ↓
MMC core:   mmc_start_request(host, mrq)
           → host->ops->request(host, mrq)
               ↓
mmci-pl18x: mmci_request()
           写 MCI_CMD(命令号) / MCI_DAT(数据参数) 寄存器
           IDMA 开始搬运数据
               ↓
SDMMC2 控制器硬件 → eMMC 芯片
```

这条路径的关键点是：**文件系统和 block 层不知道底层是 eMMC**，它们只认 `gendisk` 和 `request_queue`。`mmc_block` 负责把 block 世界的 `struct request` 翻译成 MMC 世界的 `struct mmc_request`。然后 core 层通过 `host->ops` 调 host driver 操作寄存器，最终把数据送到物理 eMMC 芯片。

---

## 3.2 核心数据结构

MMC 子系统在 I/O 路径上有两层抽象的三种结构体：block 层用 `mmc_request` 描述一次读写；core 层用 `mmc_host` 代表控制器、`mmc_card` 代表卡。它们的使用层级关系如下：

```
Block Layer (block.c / queue.c)
   │  每次读写创建一个 mmc_request
   │  mmc_request.host = 目标控制器
   ▼
┌───────────────────────────────────┐
│  mmc_request                      │
│  ├── cmd    → opcode + arg + resp │
│  ├── data   → sg表 + blksz + 方向 │
│  └── stop   → CMD12 (多块传输)    │
│  host  → mmc_host                 │
│  card  → mmc_card                 │
└────────────┬──────────────────────┘
             │ host->ops->request(host, mrq)
             ▼
┌──────────────────────┐     ┌──────────────────────┐
│  mmc_host (控制器)    │     │  mmc_card (卡)        │
│                      │     │                       │
│  ops → mmci_request  │     │  host → mmc_host      │
│  ios → 时钟/电压/时序 │     │  cid/csd/ext_csd      │
│  caps → 能力位        │     │  type → MMC/SD/SDIO   │
│  card → mmc_card ────│────►│  part[7] → 分区       │
│  class_dev → sysfs    │     └──────────────────────┘
│  detect → 工作队列    │
└──────────────────────┘
```

`mmc_host` 和 `mmc_card` 互相指向：`host->card = card`，`card->host = host`。一个 host 同一时间服务一张卡，所以这是一个单指针。

下面逐个分析关键设计点，不做字段罗列。

### mmc_host：控制器的软件抽象

`mmc_host` 是 MMC core 层的核心句柄，代表一个物理的主机控制器（ATK 板上就是 SDMMC2）。结构体定义在 `include/linux/mmc/host.h:320`，约 200 行。其字段按用途可分为以下几组。

#### 设备模型与接口

```c
struct device       *parent;        // 父设备（amba_device）
struct device       class_dev;      // 类设备（出现在 /sys/class/mmc_host/mmcX/）
int                 index;          // 主机编号（mmc0/mmc1）
const struct mmc_host_ops *ops;     // → 调 host driver 的函数表
```

`ops` 是 core 层向下调用 host driver 的唯一通道。host driver 在 probe 时填入自己的实现：

```c
// mmci.c:2236
mmc->ops = &mmci_ops;
```

core 层发一条读写命令时，调用链为：

```
mmc_start_request(host, mrq)          // core.c
  → host->ops->request(host, mrq)    // 进入 host driver
    → mmci_request(host, mrq)        // mmci.c：写 MCI_CMD 寄存器
```

`mmci_ops` 实现了最常用的回调：

| 回调 | 用途 |
|------|------|
| `request(host, mrq)` | 提交一次 I/O 请求 → 写 MCI_CMD/MCI_DAT 寄存器 |
| `set_ios(host, ios)` | 设置时钟/电压/时序 → 写 MCI_CLKCR/MCI_POWER |
| `card_busy(host)` | 检查 DAT0 忙信号 → 读 MCI_STATUS |
| `execute_tuning(host)` | 延迟线调谐 → DLYB 校准 |

更换控制器（如 SDMMC2 → DW-MMC），只需换一个 host driver 填不同的 ops。

#### 能力配置

```c
unsigned int    f_min;          // 最小时钟频率（通常 400KHz）
unsigned int    f_max;          // 最大时钟频率（DTS max-frequency 属性）
unsigned int    f_init;         // 初始化频率（默认 400KHz）
u32             ocr_avail;      // 支持的电压范围
u32             caps;           // 能力位 1（32 位掩码）
u32             caps2;          // 能力位 2（32 位掩码）
```

为什么有两个 32 位字段而不是一个 `u64`？历史原因：`caps` 最早就是 32 位，后来新能力位超过 32 个时，内核没有把 `caps` 改成 `u64`（涉及大量位操作兼容性改动），而是直接加了 `caps2`。逻辑上的大致分工是 `caps` 放基础能力（总线宽度、DDR 电压、SD UHS 模式），`caps2` 放后来加入的扩展能力（HS200/HS400、CMDQ、设备排除）。

`caps` 和 `caps2` 由 `mmc_of_parse()` 从 DTS 属性解析而来。ATK 板 `sdmmc2` 节点中的对应关系：

| DTS 属性 | cap 位 | 含义 |
|----------|--------|------|
| `non-removable` | `MMC_CAP_NONREMOVABLE` | eMMC 不可移除 |
| `bus-width <8>` | `MMC_CAP_8_BIT_DATA` | 支持 8-bit 总线 |
| `cap-mmc-highspeed` | `MMC_CAP_MMC_HIGHSPEED` | 支持高速模式 |
| `mmc-ddr-1_8v` | `MMC_CAP_1_8V_DDR` | 支持 DDR52 @1.8V |

这些 cap 位在卡探测阶段被 `mmc_select_timing()` 读取，决定尝试哪种时序模式——如果 `caps2` 没有 `MMC_CAP2_HS400`，就不会尝试 HS400。

#### 运行时状态

```c
struct mmc_ios  ios;    // 当前 I/O 设置（时钟/电压/时序/总线宽度）
```

core 层通过辅助函数修改 `ios`，然后调 `set_ios` 同步到硬件：

```c
void mmc_set_timing(struct mmc_host *host, unsigned int timing)
{
    host->ios.timing = timing;               // 改软件状态
    host->ops->set_ios(host, &host->ios);    // 写寄存器
}
```

ATK 板 eMMC 初始化过程中 `ios` 的演变：

| 阶段 | ios.timing | ios.clock | bus_width | signal_voltage |
|------|-----------|-----------|-----------|---------------|
| 初始上电 | LEGACY | 400KHz | 1 | 3.3V |
| 识别卡后 | LEGACY | 400KHz | 1 | 3.3V |
| 选卡后 | MMC_HS | 52MHz | 1 | 3.3V |
| 设置总线宽度 | MMC_HS | 52MHz | 8 | 3.3V |
| 完成初始化 | DDR52 | 52MHz | 8 | 1.8V |

每次 `ios` 变化最终反映到 SDMMC2 的 `MCI_CLKCR`（时钟分频）、`MCI_POWER`（上电与电压）等寄存器中。

#### 卡连接

```c
struct mmc_card         *card;      // 当前连接的卡
const struct mmc_bus_ops *bus_ops;  // 卡类型的操作表
```

`card` 指向关联的 `mmc_card` 实例。`bus_ops` 在卡识别阶段由 `mmc_attach_mmc()` 填入 `&mmc_ops`（eMMC 类型）。core 层通过 `host->bus_ops->detect()` 等调卡类型生命周期函数。

#### 并发控制

```c
spinlock_t          lock;       // 自旋锁，保护 claim 和 bus_ops
struct mmc_ctx      *claimer;   // 当前持有 host 的上下文
int                 claim_cnt;  // claim 嵌套计数
```

MMC 子系统的并发控制模型是"谁 claim 谁操作"：

```c
mmc_claim_host(host);       // 申请 host 所有权（拿不到则阻塞）
// ... 操作 host 和 card ...
mmc_release_host(host);     // 释放
```

卡探测线程和 I/O 提交线程通过这个机制互斥。为什么基于 host 而非 card？因为一个 host 同一时刻只能执行一条命令，不管这命令发给哪张卡——实际上一个 host 通常只接一张卡。

#### 卡检测

```c
struct delayed_work   detect;          // 卡检测/重扫描工作队列
int                   rescan_disable;  // 是否禁止重扫描
unsigned int          need_retune;     // 需要重新调谐
struct timer_list     retune_timer;    // 周期性调谐定时器
```

`detect` 工作队列在 `mmc_add_host()` 时初始化。`mmc_rescan()` 从这里触发，执行卡探测或重扫描。对于 eMMC（`non-removable`），`rescan_disable` 控制只扫描一次。

#### I/O 参数

```c
unsigned int    max_segs;       // 最大 DMA 段数
unsigned int    max_seg_size;   // 每段最大字节数
unsigned int    max_blk_size;   // 最大块大小
unsigned int    max_blk_count;  // 一次请求最大块数
unsigned int    max_req_size;   // 一次请求最大字节数
```

这些参数由 host driver 在 probe 时设定（基于硬件能力），block 层通过 `blk_queue_max_*()` 设置队列参数。

#### 命令队列（可选）

```c
const struct mmc_cqe_ops  *cqe_ops;     // CMDQ 操作表
void                      *cqe_private; // CMDQ 私有数据
int                        cqe_qdepth;  // 队列深度
bool                       cqe_enabled; // 是否启用
```

如果控制器支持硬件命令队列（CMDQ），`cqe_ops` 提供 `cqe_request()`、`cqe_recovery()` 等回调。ATK 板的 eMMC（Kingston 58A43A）支持 CMDQ，但需要 `caps2` 有 `MMC_CAP2_CQE` 位。

#### 私有数据

```c
unsigned long   private[];  // 变长数组，host driver 扩展使用
```

`mmc_alloc_host` 时一次分配 host + private：

```c
// mmci.c:2229
mmc = mmc_alloc_host(sizeof(struct mmci_host), &dev->dev);
```

`struct mmci_host` 包含 SDMMC2 的硬件依赖：

```c
struct mmci_host {
    struct mmc_host      *mmc;       // 反向指向宿主
    void __iomem         *base;      // 寄存器基址 0x48230000
    struct clk           *clk;       // 时钟句柄（166MHz）
    struct variant_data  *variant;   // STM32MP25 变体参数
    struct sdmmc_idma    *idma;      // IDMA 状态
    // ...
};
```

core 层通过 `mmc_priv(mmc)` 宏拿到指针，但不会访问其字段——私有数据只属于 host driver。

### mmc_card：卡的软件抽象

`mmc_card` 代表一张物理卡设备。它在 `mmc_init_card` 中被创建，在 `mmc_add_card` 中被注册到设备模型。

#### 创建过程

```c
// mmc.c:1657
card = mmc_alloc_card(host, &mmc_type);
card->ocr = ocr;
card->type = MMC_TYPE_MMC;
card->rca = 1;                          // eMMC 固定 RCA=1
memcpy(card->raw_cid, cid, sizeof(card->raw_cid));
```

与 `mmc_host` 不同，`mmc_card` 不是一次分配定型的。其大部分字段是在 `mmc_init_card` 执行过程中**逐步填充**的：

```
mmc_alloc_card()
  → 分配 card，设置 host 指针
mmc_send_csd() + mmc_decode_csd()
  → 填充 card->csd（容量、块大小等）
mmc_decode_cid()
  → 填充 card->cid（制造商、序列号等）
mmc_read_ext_csd()
  → 填充 card->ext_csd（512 字节扩展数据）
mmc_set_erase_size()
  → 填充 erase_size、pref_erase 等
mmc_select_timing()
  → 卡进入最终时序模式
```

这种"逐步填充"的设计是因为：早期命令（CMD2/CMD3）获取的信息很少，后续命令（CMD8/CMD6）才逐步获取更多信息。

#### 三个寄存器数据结构的用途差异

`mmc_card` 包含三个从硬件寄存器解码而来的数据结构：`cid`、`csd`、`ext_csd`。它们不是在同一个时间点获取的，用途也不同：

| 数据结构 | 获取命令 | 填充时机 | 用途 |
|----------|---------|---------|------|
| `cid` | CMD2 | 卡识别阶段 | 制造商标识 + 序列号，用于 quirks 匹配和 sysfs 显示 |
| `csd` | CMD9 | 卡识别阶段 | 卡物理参数（容量、块大小、命令类），用于初始化决策 |
| `ext_csd` | CMD8 | 选卡之后 | 512 字节扩展寄存器，时序协商 + 分区配置 + 健康监控 |

三者中 `ext_csd` 最为关键。eMMC 初始化的大部分决策（选择哪种时序模式、如何配置分区、是否启用缓存）都要读取 `ext_csd` 后才能做出。这也是内核在 CMD7 选卡之后才发 CMD8 的原因——选卡之前只能做基本的识别。

设备健康信息也来自 `ext_csd`。例如 `pre_eol_info`（寿命预终点）和 `device_life_time_est`（使用寿命估算）可以通过 debugfs 读取：

```bash
cat /sys/kernel/debug/mmc1/mmc1:0001/ext_csd | grep -E "PRE_EOL|LIFE_TIME"
```

`ext_csd.sectors` 的值直接决定块设备的大小——`cat /sys/block/mmcblk1/size` 读的就是这个值。

#### 分区模型：part_config 切换机制

`mmc_card` 的分区模型不是通过"字段包含"设计的，而是通过**配置切换**实现的。eMMC 的内部存储被划分为多个分区区域，但同一时刻只有一个是"活跃"的。通过 CMD6（SWITCH）修改 `EXT_CSD[179]`（`PART_CONFIG`）来切换当前访问的分区：

```
part_config 寄存器 (EXT_CSD[179]) 的 3:0 位：
  0 = 主分区（用户数据区）        → /dev/mmcblk1
  1 = 启动分区 0                  → /dev/mmcblk1boot0
  2 = 启动分区 1                  → /dev/mmcblk1boot1
  4 = 通用目的分区 0              → /dev/mmcblk1gp0
  5 = 通用目的分区 1              → /dev/mmcblk1gp1
  6 = 通用目的分区 2              → /dev/mmcblk1gp2
  7 = 通用目的分区 3              → /dev/mmcblk1gp3
```

RPMB（Replay Protected Memory Block）是独立处理的——它通过专门的 CMD25/CMD23/CMD24 序列访问，不是通过 `part_config` 切换。所以 `card->part` 数组只包含 7 个元素（主 + 2boot + 4gp），RPMB 单独记录在 `card->nr_parts` 中。

### mmc_request：I/O 操作单元

`mmc_request` 是 I/O 路径的核心。一次读写操作被包装成一个 request：

```c
struct mmc_request {
    struct mmc_command  *cmd;    // 命令（必选）
    struct mmc_data     *data;   // 数据段（可选）
    struct mmc_command  *stop;   // 停止命令（多块传输时用）
    struct mmc_command  *sbc;    // 设置块计数命令（CMD23）
    struct mmc_card     *card;   // 目标卡
    struct mmc_host     *host;   // 目标控制器
    bool                cap_cmd_during_tfr; // 传输中允许发命令
    int                 tag;     // CQE 标签
    void (*done)(struct mmc_request *);  // 完成回调
};
```

`cmd`、`data`、`stop` 三个指针构成一个"命令三元组"：

```
mmc_request
├── cmd  → mmc_command (opcode + arg + flags + resp[4])
│          一次读写需要一条 READ/WRITE 命令
├── data → mmc_data (sg 表 + blksz + blocks + 方向)
│          数据在内存和卡之间通过 DMA 或 PIO 传输
└── stop → mmc_command (CMD12 停止传输)
          多块读写完成后需要发停止命令
```

命令结构体：

```c
struct mmc_command {
    u32         opcode;     // 命令号：CMD18 等
    u32         arg;        // 参数
    u32         resp[4];    // 响应（R1/R1B=1 字，R2=4 字）
    unsigned int flags;     // 响应类型 + 命令类型
    unsigned int retries;   // 重试次数
    int         error;      // 错误码
    struct mmc_data *data;  // 关联的数据段
    struct mmc_request *mrq; // 所属请求
};
```

数据段结构体：

```c
struct mmc_data {
    unsigned int    blksz;      // 块大小（通常 512 字节）
    unsigned int    blocks;     // 块数
    unsigned int    bytes_xfered; // 已传输字节数
    int             error;      // 传输错误码
    unsigned int    flags;      // 方向：MMC_DATA_READ / MMC_DATA_WRITE
    struct scatterlist *sg;     // DMA 散列表
    unsigned int    sg_len;     // 散列表长度
    struct mmc_command *stop;   // 关联的停止命令
    struct mmc_request *mrq;    // 所属请求
};
```

> 💡 **STM32MP2 硬件联动**：  
> `mmc_data->sg` 包含的内存分散列表，在进入 Host 层（`mmci_stm32_sdmmc.c`）后，会被驱动程序遍历并转换为 STM32MP257 SDMMC 控制器所认识的 IDMA 链表描述符（`sdmmc_lli_desc`）。控制器随后通过硬件直接读取这些描述符完成高速数据传输，无需 CPU 干预。

请求的完整生命周期：

```
block 层组装 mmc_request
        ↓
mmc_start_request(host, mrq)        → core.c
  → mmc_mrq_prep()                   → DMA 映射
    → host->ops->request(host, mrq)  → mmci_request()
        ↓
host driver 写寄存器，启动传输
        ↓
中断到达 → mmci_irq()
  → mmc_request_done(host, mrq)      → core.c
    → mrq->done(mrq)                 → block 层完成回调
```

> 💡 **CQE 异步 I/O 预告**：  
> 上图中的 I/O 路径是**同步阻塞**模式——`mmc_wait_for_req()` 会阻塞等待完成信号。支持 CMDQ（命令队列）的 eMMC（如 Kingston 58A43A）可以通过 `cqe_ops->cqe_request()` 实现异步 I/O：同时下发多条命令，每条命令用 `tag` 追踪，完成顺序与下发顺序无关。这能显著提升随机读写吞吐，后续 I/O 路径分析文章中会展开。

### 三个结构体在初始化中的协作

以 3.1 节的 init 流程为背景，三个结构体的填充顺序：

```
阶段 1-2: mmc_host 分配并配置
  mmc_alloc_host() → host 创建
  mmci_probe() → ops / caps / f_min/f_max / 私有数据 填入
  mmc_add_host() → detect 工作队列启动

阶段 3: mmc_card 创建并填充
  mmc_alloc_card(host, ...)  → card 创建，host 指针关联
  mmc_init_card()             → 逐步填充 cid/csd/ext_csd
  mmc_add_card(card)          → card->dev 注册

阶段 4: mmc_request 开始出现
  mmc_blk_probe() → 初始化 blk-mq 队列
  之后每次读写操作创建 mmc_request
```

三个阶段分别对应三个结构体的"主角期"：初始化开始时只有 `mmc_host`；卡探测过程中 `mmc_card` 被填充；块设备就绪后 `mmc_request` 开始流转。

---

## 本文总结

1. **三层架构**：BLOCK（块设备）、CORE（协议编排）、HOST（寄存器操作）。block 层通过 `mmc_request` 与 core 层通信，core 层通过 `host->ops` 调 host driver。
2. **两条独立总线**：AMBA 总线（`mmci_driver` + SDMMC2 控制器）和 MMC 总线（`mmc_block` + `mmc_card`），平行运行，互不隶属。
3. **两张 ops 表**：`mmc_host_ops`（操作寄存器）和 `mmc_bus_ops`（管理卡），core 层通过它们调度两端。
4. **两组设备/驱动关系**：AMBA 总线上 `mmci_driver` 驱动 SDMMC2 控制器；MMC 总线上 `mmc_block` 驱动 `mmc_card`。

下一篇 [04-SourceAnalysis.md](04-SourceAnalysis.md) 将进入源码分析。
