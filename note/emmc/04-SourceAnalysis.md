# 04. eMMC 初始化流程源码分析

> 本文是 STM32MP257 eMMC 驱动深度分析系列的第 4 篇。
> 对应框架步骤 ④：从源码层面逐行追踪初始化路径。
>
> **前置:** [03-Architecture.md](03-Architecture.md) — 熟悉 mmc_host / mmc_card / ops 概念
> **下一篇:** [05-IO-Path.md](05-IO-Path.md)
>
> **字数：** 中文字数 27,925 + 英文单词 10,016 ≈ **37,941 字**（含代码段），阅读时间约 **120-150 分钟**

---

## 4.1 从一个问题开始

系统启动后，你在终端敲 `ls /dev/mmcblk*`，看到：

```
/dev/mmcblk1       # 主分区 14.6 GiB
/dev/mmcblk1boot0  # 启动分区 4 MiB
/dev/mmcblk1boot1  # 启动分区 4 MiB
/dev/mmcblk1rpmb   # RPMB 安全分区 4 MiB
```

问题是：**从一块 eMMC 闪存芯片到变成 `/dev/mmcblk1`，内核里经历了什么？**

04 篇的目的就是回答这个问题。我们会选一条具体的路径——**ATK 板上的 eMMC（接在 SDMMC2 控制器上）**——逐行追踪从系统启动到块设备就绪的完整代码流。

### 三个组成部分

先搞清楚代码里涉及的几个模块各自是什么角色。

MMC 子系统的软件栈分为三层：

```
mmc_block.ko       块设备驱动层    把 eMMC 包装成 /dev/mmcblk1
mmc_core.ko        核心框架层      卡探测、命令发送、时序切换等通用逻辑
mmci-pl18x         控制器驱动层    操作 SDMMC2 硬件寄存器、中断、DMA
```

- **`mmci-pl18x`**（`drivers/mmc/host/mmci*.c`）：这是 SDMMC2 **控制器的驱动**。它操作寄存器、处理中断、提交 DMA。它不知道什么是文件系统。它实现 `mmc_host_ops` 回调表，供 core 层调用。
- **`mmc_core.ko`**（`drivers/mmc/core/*.c`）：这是**核心框架**。它做的事包括：卡探测流程编排、命令重试、时序协商、电源管理。它不直接操作 SDMMC2 寄存器，而是通过 `host->ops->xxx()` 调控制器驱动。
- **`mmc_block.ko`**（`drivers/mmc/core/block.c` + `queue.c`）：这是**块设备驱动**。它负责分配 gendisk、注册 blk-mq 请求队列、处理 ioctl。它让 eMMC 看起来像一块硬盘。

### "MMC 子系统注册"指的是什么

从设备模型的角度看，一个子系统要能运转，需要注册两样基础结构：

1. **总线类型**（`bus_type`）——承载设备和驱动，负责匹配和 probe
2. **设备类**（`class`）——在 `/sys/class/` 下提供用户态视角的设备归类

MMC 子系统的注册在 `mmc_init()` 中完成：

- **`mmc_bus_type`**：名为 `mmc` 的总线。卡设备（`mmc_card`）挂在这条总线上，块设备驱动（`mmc_block`）也注册在这条总线上，由总线核心负责匹配。
- **`mmc_host_class`**：名为 `mmc_host` 的设备类。

**class 不是必须的——没有它控制器驱动照样工作、卡照样读写。** 它的唯一作用是提供一个**稳定的、不依赖硬件拓扑的入口**供用户态工具定位控制器。

为什么需要？看设备树。`mmc_host_class` 注册后，class device（名为 `mmc1`）以 AMBA 设备为 parent 注册，`mmc_card` 挂在它下面——但这发生在设备树内部，`class device` 的路径不是 `mmc_host/` 这个目录：

```
/sys/devices/platform/soc/48230000.sdmmc/        ← AMBA 设备
  └── mmc1/                                      ← class device (parent = AMBA 设备)
       └── mmc1:0001/                            ← mmc_card (parent = class device)
```

`mmc_host` 这个类名体现在 **class 类型的 symlink** 上，不体现在设备路径中：

```
/sys/class/mmc_host/mmc1/ → ../../devices/platform/soc/48230000.sdmmc/mmc1/
```

有了这条 symlink，所有控制器都集中在：

```
/sys/class/mmc_host/mmc0/    ← SDMMC1（SD 卡槽，如果有）
/sys/class/mmc_host/mmc1/    ← SDMMC2（eMMC）
/sys/class/mmc_host/mmc2/    ← SDMMC3（如果有）
```

用户态程序不需要知道控制器挂在哪条总线上（可能是 AMBA、platform、SDHCI 的 PCI 设备），直接遍历 `/sys/class/mmc_host/` 就行（`mmc-utils` 就是这样做的）。

**class 和 `/dev/xxx` 是两回事**——class 归类的是**控制器**，`/dev/mmcblk1` 是**块设备**的访问节点，由 `device_add_disk()` 创建，完全不依赖 `mmc_host_class`。两者的关系链是：

```
mmc_host_class  →  /sys/class/mmc_host/mmc1/         ← 控制器的 symlink
                        │ parent
                        ↓
mmc_add_card     →  mmc_card 挂在 mmc1 下面           ← 卡设备注册
                        │ 总线匹配
                        ↓
mmc_blk_probe   →  device_add_disk → /dev/mmcblk1    ← 块设备节点创建
```

class 只服务前两步，不参与块设备的创建设备节点的过程。

- `mmc_init()` 额外注册了一个字符设备和 `sdio_bus_type`。

**字符设备（`register_chrdev(0, "mmc", &mmc_fops)`）**：分配一个主号（可通过 `/proc/devices` 查到），只有 `.open` 和 `.unlocked_ioctl` 两个操作。它跟 `/dev/mmcblk1` 是两套完全独立的接口——字符设备负责**控制面**（发 CMD、读状态），块设备负责**数据面**（读写存储扇区）。比如 `mmc-utils` 读 EXT_CSD 的流程是：打开字符设备 → `ioctl` 下发 CMD8 → 取回 512 字节响应。这条路不经过块设备，也不发读写块数据。反过来，`dd if=/dev/mmcblk1` 走的是块设备，不会经过字符设备。两者彼此无关，同时存在。

**`sdio_bus_type`**：SDIO 是一种扩展接口，把 SD 总线协议扩展为外设总线，常用于 WiFi 模块、蓝牙等。它在 `mmc_init()` 中顺手注册，但因为 eMMC 不是 SDIO 设备，所以走不到这条分支。ATK 板不涉及，只提不展开。

入口是 `subsys_initcall`，优先级高于 `module_init`，保证 core 层在所有控制器驱动 probe 之前就绪。

### 初始化流程全景

```
时间 → 系统启动
         ↓
[1] Core 层注册
    mmc_init()
     ├─ 注册 mmc_bus_type
     ├─ 注册 mmc_host_class
     └─ 注册 sdio_bus_type
         ↓
[2] 控制器驱动 probe
    DTS 节点 compatible = "arm,primecell"
     └─ amba_match() → mmci_probe()
          ├─ mmc_alloc_host() → 创建 mmc_host 实例
          ├─ 填入 mmc->ops = &mmci_ops
          ├─ 解析 DTS 属性（解析后写入 mmc->caps 等字段）
          └─ mmc_add_host() → 移交 core 层管理
               ↓
[3] Core 层自动探测卡
    mmc_start_host() → mmc_rescan()
     └─ mmc_rescan_try_freq() → 尝试 400KHz
          └─ mmc_attach_mmc() → 识别为 eMMC
               └─ mmc_init_card() → CMD 序列初始化
                    ├─ CMD0  复位卡
                    ├─ CMD1  获取 OCR（电压信息）
                    ├─ CMD2  获取 CID（卡 ID）
                    ├─ CMD3  分配 RCA + 切推挽
                    ├─ CMD9  获取 CSD（卡参数）
                    ├─ CMD7  选卡，进入 Transfer State
                    ├─ CMD8  读取 EXT_CSD（512 字节扩展寄存器）
                    └─ CMD6  多次 SWITCH：时序/总线宽度/缓存
               ↓
[4] 块设备注册
    mmc_add_card() → device_add()
     └─ mmc_bus_type 匹配 → mmc_blk_probe()
          └─ /dev/mmcblk1 可用
```

后续各节按这 4 个阶段展开。

---

## 4.2 阶段一：Core 层注册 — `mmc_init()`

**入口文件：** `drivers/mmc/core/core.c`，`subsys_initcall(mmc_init)`

`subsys_initcall` 的链接优先级是 4（`module_init` 是 6），这意味着它在所有 `module_init` 之前执行。设计意图很清晰：卡探测依赖总线/类机制，core 层必须先于控制器驱动就绪。

`mmc_init()` 干了四件事：

```c
static int __init mmc_init(void)
{
    int ret;

    ret = register_chrdev(0, "mmc", &mmc_fops);         // ① 字符设备
    ret = mmc_register_bus();                            // ② mmc_bus_type
    ret = mmc_register_host_class();                     // ③ mmc_host_class
    ret = sdio_register_bus();                           // ④ sdio_bus_type
    return 0;
}
```

逐行看：

- **① `register_chrdev(0, "mmc", &mmc_fops)`**：注册一个主号动态分配的字符设备。这不是给用户读写用的——`mmc_fops` 只支持 `.open` 和 `.unlocked_ioctl`，用户态工具（如 `mmc-utils`）通过 `ioctl` 向内核发送 RAW 的 MMC 命令来调试或诊断。主号 0 让内核动态分配，成功后可通过 `/proc/devices` 查到。

- **② `mmc_register_bus()`**：调 `bus_register(&mmc_bus_type)`，在 sysfs 下创建 `/sys/bus/mmc/`。这条总线后续承载两类设备：`mmc_card`（卡设备）和 `mmc_driver`（块设备驱动）。

- **③ `mmc_register_host_class()`**：调 `class_register(&mmc_host_class)`，创建 `/sys/class/mmc_host/`。每个 SDMMC 控制器对应一个 `mmc_host` 实例，出现在此目录下（如 `mmc0`、`mmc1`）。

- **④ `sdio_register_bus()`**：注册 `sdio_bus_type`。ATK 板没有 SDIO 设备，不展开。

**阶段一结束时的状态：**

```
/sys/bus/mmc/              → mmc_bus_type，就绪
/sys/class/mmc_host/       → mmc_host_class，就绪
/proc/devices 中有 "mmc"   → 字符设备，就绪
```

但此时还没有具体的 mmc_host 实例——那是阶段二的事。

---

## 4.3 阶段二：控制器驱动 Probe — `mmci_probe()`

### 4.3.1 AMBA 总线：物理互连与 Linux 驱动框架

SDMMC2 走 AMBA 总线而非 platform 总线。理解这一点需要区分两个层面：物理片上互连和 Linux 驱动框架。

#### 物理 AMBA 总线——芯片内部的互连标准

**STM32MP257 里所有外设，包括 UART、I2C、SPI、SDMMC、GPU、以太网，物理上都连接在 AMBA 总线上。** AMBA（AXI/AHB/APB）是 ARM 芯片内部的互连标准——Cortex-A35 核心通过 AXI 总线访问 SDMMC2 控制器，也通过同一条 AXI 总线访问其他所有外设。这条总线在芯片物理上有实实在在的线路和时序协议。

```
Cortex-A35         Cortex-A35
  核心 0             核心 1
    │                 │
    └──────┬──────────┘
           │  AXI 总线（物理 AMBA 总线）
     ┌─────┼─────┬─────┬─────┐
     │     │     │     │     │
  SDMMC2  UART4  I2C3  SPI1  GPU  ← 所有外设都在这条物理总线上
```

所有外设都共享同一条物理 AMBA 互连。

#### Linux `amba_bustype`——PrimeCell 驱动框架

**Linux 里"走 AMBA 总线"指的则是 `amba_bustype` 这个驱动框架**，它只给 DTS 中 `compatible` 含有 `"arm,primecell"` 的设备使用。判断标准只有一条——**看 DTS 的 compatible 字符串**。

对比 STM32MP257 的两个外设：

```dts
/* SDMMC2 → compatible 含 "arm,primecell" → Linux 走 amba_bustype */
sdmmc2: sdmmc@48230000 {
    compatible = "st,stm32mp25-sdmmc2", "arm,pl18x", "arm,primecell";  // ← 触发 AMBA 路径
};

/* UART4 → compatible 不含 "arm,primecell" → Linux 走 platform_bus_type */
uart4: serial@40070000 {
    compatible = "st,stm32h7-uart";                                     // ← 走 platform
};
```

两个设备物理上都挂在同一条 AXI 总线上，但由于 compatible 字符串不同，在 Linux 里一个属于 `amba_bustype`，一个属于 `platform_bus_type`。

场景对比：

| 场景 | 物理层面 | Linux `bus_type` 层面 |
|------|---------|---------------------|
| SDMMC2 | 在 AXI 总线上 | `amba_bustype`（带 PrimeCell 硬件 ID） |
| UART4 | 在 AXI 总线上 | `platform_bus_type`（字符串匹配） |
| 两者的关系 | 同一条物理总线 | 不同的驱动框架 |

#### 为什么需要两套 bus_type？

```dts
/* ① 走 amba_bustype——有硬件 ID 可校验 */
sdmmc2: sdmmc@48230000 {
    compatible = "arm,primecell";   // → OF 框架识别，读 PeriphID 寄存器精确匹配
};

/* ② 走 platform_bus_type——只能靠 compatible 字符串匹配 */
uart4: serial@40070000 {
    compatible = "st,stm32h7-uart";  // → 匹配 drivers/tty/serial/stm32-usart.c
};

/* ③ 走 platform_bus_type——I2C 控制器 */
i2c3: i2c@40090000 {
    compatible = "st,stm32mp25-i2c"; // → 匹配 drivers/i2c/busses/i2c-stm32f7.c
};
```

所有外设的物理连接方式完全一样（都挂在 AXI 总线上），但 Linux 选用哪个驱动框架取决于该外设的 IP 是 ARM 设计的 PrimeCell 还是 ST 自己设计的。PrimeCell 是 ARM 的 IP 商标，遵循 ARM 的寄存器布局规范（含 PeriphID/CID 寄存器），所以可以用 `amba_bustype` 做硬件自校验；ST 自研的 UART、I2C 等 IP 没有这些寄存器，就用 platform 总线做字符串匹配。

#### 实际设备分布：AMBA 总线上有哪些设备？

在 STM32MP257 的 DTS 中搜索 `"arm,primecell"`，得到：

| 设备 | DTS 节点 | 用途 |
|------|---------|------|
| SDMMC1/2/3 | `sdmmc@48220000/48230000/...` | SD/eMMC 控制器 |
| ETM | `etm@...` | Coresight 跟踪（调试用） |
| Funnel | `funnel@...` | Coresight 跟踪数据合并 |
| TMC/TPIU | `tmc@...` / `tpiu@...` | Coresight 跟踪缓冲区/输出 |
| STM | `stm@...` | Coresight 系统跟踪 |
| CTI | `cti@...` | Coresight 交叉触发 |
| CPU Debug | `debug@...` | 调试接口 |

基本上是 **SDMMC + 调试跟踪外设**。UART、I2C、SPI、GPIO、以太网等常规外设都走 platform 总线——它们和 SDMMC 物理上共享同一条 AXI 总线，但 Linux 用不同的 `bus_type` 管理。

#### 物理 AMBA 总线与 Linux bus_type 的关系

```
                    STM32MP257 SoC
┌──────────────────────────────────────────────────────┐
│  Cortex-A35                                           │
│     │                                                 │
│     └────────── AXI 总线（物理 AMBA 互联）───────────│──── 物理层面
│           │        │         │          │             │   所有外设都在这
│       SDMMC2    UART4     I2C3      ETM              │
│           │        │         │          │             │
│      compatible  compatible  compatible  compatible   │
│      "arm,       "st,       "st,        "arm,        │
│      primecell"  stm32h7-   stm32mp25-  primecell"   │
│                  uart"      i2c"                     │
│           │        │         │          │             │
│           ↓        ↓         ↓          ↓             │
│      ┌─────────┐ ┌────────┐ ┌────────┐ ┌──────────┐  │
│      │amba_bus │ │platform│ │platform│ │amba_bus  │  │── Linux 层面
│      │_type    │ │_bus_typ│ │_bus_typ│ │_type     │  │   bus_type 不同
│      │(PrimeCell)│(ST UART)│(ST I2C) │(Coresight)│  │
│      └─────────┘ └────────┘ └────────┘ └──────────┘  │
└──────────────────────────────────────────────────────┘
```

**核心结论：Linux 的 `bus_type` 不反映物理连接方式，它只决定设备和驱动怎么匹配。** 物理上所有外设共享同一条 AXI 总线，但 Linux 对 PrimeCell 设备和普通外设用了两套不同的匹配机制。

#### 硬件自校验的实际意义

对比两种场景：

**普通平台总线设备（如 UART4）**：内核完全信任 DTS。DTS 说 `uart4@40070000` compatible = `"st,stm32h7-uart"`，驱动就信了，直接 probe。如果 DTS 写错了地址，比如把 `reg` 配成了 `0x40070000` 但硬件实际是 `0x40080000`，驱动读到的寄存器都是错的，可能挂死或静默工作异常——内核没办法发现这个错误。

**PrimeCell 设备（如 SDMMC2）**：内核不轻信 DTS。DTS 说这是 SDMMC2，内核先读硬件寄存器 0xFE0-0xFEC，确认 PeriphID = `0x??353180`，再读 0xFF0-0xFFC 确认 CID = `0xB105F00D`。两个都对上了，才相信 DTS 是对的。如果 DTS 的 `reg` 指向了错误的地址，读出来的 ID 不对，amba 匹配直接失败，驱动不会被 probe。

这不是一个"花哨功能"，而是一个**实质性的可靠性差异**：

| | 普通外设（platform） | PrimeCell（amba） |
|--|-------------------|-----------------|
| **设备识别靠什么** | 完全依赖 DTS 字符串 | DTS 指个路 + 硬件 ID 双重确认 |
| **DTS `reg` 写错会怎样** | 驱动 probe 后访问错误地址，可能挂死 | PeriphID 读出来不对，匹配失败，不 probe |
| **不同版本怎么区分** | DTS 换 compatible 字符串 | 同一个驱动读 PeriphID 低字节（版本号），自动适配 |
| **类似的设计** | 身份完全靠外部声明 | 硬件自带身份标识（类比 PCI Vendor ID、USB VID:PID）|

这就是 4.3.1 开头那张总线对比表中，`amba_bustype` 标注为"半虚拟（有硬件 ID）"的原因——它虽然也是软件框架，但它管的设备在硬件层面自带了身份标识，让总线匹配多了一层硬件验证。（PCI 和 USB 也类似：PCI 设备有 Vendor ID/Device ID，USB 设备有 VID/PID，都是硬件自枚举。SoC 集成外设里能做到这点的很少，PrimeCell 是其中之一。）

### 4.3.2 从 DTS 到 AMBA 匹配

SDMMC2 控制器在 DTS 中描述如下（来自 `stm32mp25-atk-bsp.dts` 引用的 SoC 通用 DTSI）：

```
sdmmc2: sdmmc@48230000 {
    compatible = "arm,primecell";
    reg = <0x48230000 0x1000>;
    interrupts = <GIC_SPI 126 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&rcc SDMMC2CLK>, <&rcc SDMMC2K>;
    clock-names = "apb_pclk", "sdmmc2";
    ...
};
```

**匹配入口：** `compatible = "arm,primecell"` 是触发 AMBA 匹配的关键标记。OF 框架看到这个值后不走 platform 总线路径，而是进入 AMBA 总线的识别流程：

```
DTS: compatible = "arm,primecell"
  → 触发 AMBA 总线识别流程
  → of_amba_device_create() 创建 amba_device
  → 分配虚拟地址 → 读取寄存器 0xFE0~0xFEC 处的 PeriphID 值
  → 组合成 periphid → 遍历 mmci_driver.id_table
  → mmci_ids[] 中的条目逐个比对（id & mask == periphid & mask）
  → 匹配 → 用对应的 .data 数据调用 mmci_probe()
```

这里最关键的区别是：**AMBA 的设备识别不靠 DTS 中的 compatible 字符串细分，而靠硬件寄存器中的固化 ID**。每个 PrimeCell 外设在其寄存器地址空间的 `0xFE0`~`0xFEC` 偏移处固化了 4 个 `PeriphID` 寄存器，存有外设的类型和版本号。内核只需要知道"这是一个 PrimeCell"，然后通过读寄存器的物理值就能区分是哪个变体。

**SDMMC 拥有硬件 ID 而 UART、I2C 没有，原因不在功能差异，而在 IP 供应商不同。** SDMMC 控制器是 ARM 设计的 PrimeCell PL18x，UART 和 I2C 是 ST 自研的 IP。ARM 定义 PrimeCell 标准时要求每个 IP 都包含 PeriphID/CID 寄存器；ST 设计自己的 UART 和 I2C 时没按这个标准来。如果 ST 当初选了 ARM 的 PL011 UART，那个 UART 也会有 PeriphID，也会走 `amba_bustype`。在 STM32MP257 上，另一组走 `amba_bustype` 的外设是 Coresight 调试组件（ETM、Funnel、TMC 等），这些也是 ARM 设计的 IP。分界线是"ARM 设计的 IP vs ST 设计的 IP"，不沿功能划分。

SDMMC2 对应的 `mmci_ids[]` 条目：

```c
/* mmci.c */
static struct amba_id mmci_ids[] = {
    {
        .id     = 0x00353180,        // 期望的硬件 ID 值
        .mask   = 0xf0ffffff,        // 逐位掩码，1=精确匹配，0=忽略
        .data   = &variant_stm32_sdmmcv3,  // 匹配后关联的数据
    },
    ...
};
```

`amba_lookup()` 的匹配逻辑是：**`(dev->periphid & mask) == (id & mask)`**。这里 `id = 0x00353180`，`mask = 0xf0ffffff`，含义是：

```
0x00353180  = 0x00  0x35  0x31  0x80
0xf0ffffff  = 0xf0  0xff  0xff  0xff

匹配条件：最低 3 字节完全一致 (0x353180)
         最高字节的低 4 位不关心（mask=0x0）
```

所以只要硬件读出的 periphid 值是 `0x??353180`（最高字节任意），就会匹配到这个条目，使用 `variant_stm32_sdmmcv3` 作为驱动数据。

`variant_stm32_sdmmcv3` 是 STM32MP2 专属的变体描述结构体（`drivers/mmc/host/mmci.c:312`）。它的每个字段都在告诉核心驱动：**这个芯片的 SDMMC 控制器有哪些与众不同之处**。

`variant_data` 结构体（`mmci.h:337`）约包含 40 个字段，大部分是寄存器位的差异标记，比如命令响应格式（长响应/短响应/CRC）、时钟分频算法、电源控制方式等。对于 STM32MP2，最关键的几个差异：

| 字段 | `variant_stm32_sdmmcv3` | 含义 |
|------|-------------------------|------|
| `f_max` | 267000000 | 最大时钟 267MHz（控制器硬件上限） |
| `fifosize` | 1024 (256×4) | FIFO 深度 1KB，v1/v2 只有 64B |
| `stm32_clkdiv` | true | 使用 STM32 特有的分频算法：`clkdiv = rate/mclk - 2` |
| `stm32_idmabsize_mask` | `GENMASK(16, 6)` | IDMA 描述符中 `idmasize` 字段的位宽 |
| `dma_lli` | true | 支持 DMA 链接列表（硬件自动遍历描述符链） |
| `datactrl_first` | true | 数据通道必须先于命令设置 |
| `busy_detect` | true | 支持 DAT0 忙信号检测 |
| `datalength_bits` | 25 | MMCIDATALENGTH 寄存器为 25 位（支持最大 32MB 传输） |
| `init` | `sdmmc_variant_init` | 初始化函数（DLYB regmap + IDMA 设置） |

对比 v1 和 v2 可以更清楚地看出 v3 的进化：

| 差异点 | `stm32_sdmmc` (v1) | `stm32_sdmmcv2` | `stm32_sdmmcv3` |
|-------|--------------------|----------------|----------------|
| FIFO | 64 B | 64 B | **1024 B** |
| `f_max` | 208 MHz | 267 MHz | 267 MHz |
| `dma_lli` | 无 | 无 | **支持** |
| IDMA 掩码 | `GENMASK(12,5)` | `GENMASK(12,5)` | `GENMASK(16,6)` |

三个版本的 FIFO 从 64B 跳到 1024B，是因为 STM32MP2 的 SDMMC 控制器内部 FIFO 硬件深度增加，减少了 PIO 模式下中断触发的频率。`dma_lli` 的支持意味着 IDMA 可以用描述符链实现 scatter-gather，无需 CPU 逐个搬运。

---

**从 DTS 到 `mmci_probe()` 的完整源码链路**

下面把前面流程图中的 6 步展开为源码，逐行走一遍。

**Step ①：DTS 遍历到 `compatible = "arm,primecell"`**

入口在 `drivers/of/platform.c:409`，OF 框架构建设备模型时遍历 DTS 节点：

```c
/* of_platform_bus_create() */
if (of_device_is_compatible(bus, "arm,primecell")) {
    of_amba_device_create(bus, bus_id, platform_data, parent);
    return 0;   // ← 關鍵：返回后不再创建 platform device
}
```

这一步决定了为什么 SDMMC2 不走 platform 总线：OF 框架先检查 `"arm,primecell"`，是就直接走 AMBA 分支并返回，后面的 `of_platform_device_create_pdata()` 不会执行。

**Step ②：`of_amba_device_create()` — 创建 `amba_device`**

```c
/* drivers/of/platform.c:253 */
static struct amba_device *of_amba_device_create(...)
{
    dev = amba_device_alloc(NULL, 0, 0);             // 分配 amba_device

    device_set_node(&dev->dev, of_fwnode_handle(node));
    dev->dev.parent = parent ? : &platform_bus;

    /* 可选：DTS 可以覆盖硬件 ID */
    of_property_read_u32(node, "arm,primecell-periphid", &dev->periphid);

    /* 提取 reg 中的地址和大小 */
    ret = of_address_to_resource(node, 0, &dev->res);
    // → dev->res.start = 0x48230000
    // → dev->res.end   = 0x48230FFF

    /* 注册到设备模型 */
    amba_device_add(dev, &iomem_resource);
}
```

**Step ③：`amba_device_add()` — 读出 PeriphID 再注册**

```c
/* drivers/amba/bus.c:547 */
int amba_device_add(struct amba_device *dev, struct resource *parent)
{
    request_resource(parent, &dev->res);

    /* 如果 DTS 没设 arm,primecell-periphid，就读硬件寄存器 */
    if (!dev->periphid) {
        if (amba_read_periphid(dev))
            dev_set_uevent_suppress(&dev->dev, true);
    }

    device_add(&dev->dev);    // 注册到设备模型，触发总线匹配
}
```

**Step ④：`amba_read_periphid()` — 读取寄存器 0xFE0~0xFEF**

```c
/* drivers/amba/bus.c:133 */
static int amba_read_periphid(struct amba_device *dev)
{
    /* 先开时钟 */
    amba_get_enable_pclk(dev);

    /* 映射寄存器地址空间：0x48230000 + 4KB */
    size = resource_size(&dev->res);    // = 0x1000
    tmp = ioremap(dev->res.start, size);

    /* 读 PeriphID0~3：位置在地址空间末尾 - 0x20 处 */
    for (pid = 0, i = 0; i < 4; i++)
        pid |= (readl(tmp + size - 0x20 + 4 * i) & 255) << (i * 8);
    /*  读 tmp+0xFE0 → PeriphID0 (bit[7:0])
     *  读 tmp+0xFE4 → PeriphID1 (bit[15:8])
     *  读 tmp+0xFE8 → PeriphID2 (bit[23:16])
     *  读 tmp+0xFEC → PeriphID3 (bit[31:24])
     */

    /* 读 CID0~3：位置在地址空间末尾 - 0x10 处 */
    for (cid = 0, i = 0; i < 4; i++)
        cid |= (readl(tmp + size - 0x10 + 4 * i) & 255) << (i * 8);

    /* CID 验证通过后才赋值 periphid */
    if (cid == AMBA_CID || cid == CORESIGHT_CID) {
        dev->periphid = pid;
        dev->cid = cid;
    }

    iounmap(tmp);
}
```

**CID 是 AMBA 的 Magic Number。** `AMBA_CID = 0xB105F00D`（读作 "bus food"），是 PrimeCell 外设固定的硬件 ID。内核读 CID 的目的是**验证当前映射的地址确实是一个有效的 PrimeCell 外设**——如果 DTS 的 `reg` 地址写错、或者硬件未上电，读出来的值不会是 `0xB105F00D`，periphid 不会被赋值，总线匹配会失败。形象地说：PeriphID 是"你是谁"，CID 是"你确实是你声称的那个东西"。

**注意不要和 eMMC 卡的 CID 混淆**——第 4.4.5 节的 CMD2 也读一个叫 CID 的东西，但那两个 CID 是两个完全独立的概念：

| CID | 所属 | 位置 | 值 | 用途 |
|-----|------|------|-----|------|
| **AMBA CID** | SDMMC2 控制器（PrimeCell 外设） | 寄存器偏移 0xFF0-0xFFC | `0xB105F00D` | 验证 PrimeCell 身份 |
| **eMMC CID** | eMMC 芯片（Kingston 58A43A） | 通过 CMD2 从卡读取 | 厂商/序列号等 | 识别卡身份 |

硬件寄存器布局（以 0x48230000 基址 + 4KB 空间为例）：

```
0x48230000                        0x48230FE0    0x48230FF0    0x48230FFF
    ├── 控制寄存器 ... ──┤              ├──PeriphID0~3──┤├──CID0~3──┤
    ↑                                    ↑               ↑
    ioremap                           size-0x20        size-0x10
    (tmp)
```

**Step ⑤：`amba_match()` — 总线匹配**

`device_add(&dev->dev)` 触发总线匹配，AMBA 总线的 match 函数：

```c
/* drivers/amba/bus.c:207 */
static int amba_match(struct device *dev, struct device_driver *drv)
{
    /* 如果上一步 periphid 没读到（注册时资源未就绪），这里再试 */
    if (!pcdev->periphid) {
        ret = amba_read_periphid(pcdev);
        if (ret)
            return -EPROBE_DEFER;   // 暂时 defer，等条件满足再试
    }

    /* 遍历驱动的 id_table */
    return amba_lookup(pcdrv->id_table, pcdev) != NULL;
}
```

`amba_lookup()` 遍历 `mmci_ids[]` 表：

```c
/* drivers/amba/bus.c:49 */
static const struct amba_id *
amba_lookup(const struct amba_id *table, struct amba_device *dev)
{
    while (table->mask) {
        if ((dev->periphid & table->mask) == table->id)
            return table;
        table++;
    }
    return NULL;
}
```

**Step ⑥：`amba_probe()` → 调用 `mmci_probe()`**

```c
/* drivers/amba/bus.c:277 */
static int amba_probe(struct device *dev)
{
    struct amba_device *pcdev = to_amba_device(dev);
    struct amba_driver *pcdrv = to_amba_driver(dev->driver);

    /* 再次 lookup 获取匹配的 amba_id 条目（含 .data） */
    const struct amba_id *id = amba_lookup(pcdrv->id_table, pcdev);

    /* 开时钟、pm_runtime... */

    /* 调用驱动的 probe，传入 id（含 variant_stm32_sdmmcv3） */
    ret = pcdrv->probe(pcdev, id);
}
```

至此，`mmci_probe()` 被调用，传入的 `id->data = &variant_stm32_sdmmcv3` 就是这个匹配过程的最终输出——它携带了 STM32MP2 SDMMC 的所有硬件差异参数。

**这个 `pcdrv->probe` 为什么指向 `mmci_probe`？** 回到 `mmci.c` 的驱动定义：

```c
/* drivers/mmc/host/mmci.c:2688 */
static struct amba_driver mmci_driver = {
    .drv = {
        .name = "mmci-pl18x",
    },
    .probe      = mmci_probe,        // ← 驱动级别的 probe
    .id_table   = mmci_ids,
};

/* 注册：module_amba_driver() → amba_driver_register() */
drv->drv.bus = &amba_bustype;        // ← 告诉内核挂在 AMBA 总线上
driver_register(&drv->drv);           // ← 注册到设备模型
```

而 AMBA 总线框架在初始化时注册了**总线级别的 probe**：

```c
/* drivers/amba/bus.c:437 */
struct bus_type amba_bustype = {
    .name   = "amba",
    .match  = amba_match,    // 总线匹配函数
    .probe  = amba_probe,    // ← ★ 总线级别的 probe 入口
};
```

所以 `pcdrv->probe` 能调到 `mmci_probe`，是因为**两级 probe 的协作**：

```
mmci 驱动注册:
  module_amba_driver(mmci_driver)
    → amba_driver_register(&mmci_driver)
    → drv->drv.bus = &amba_bustype
    → driver_register(&drv->drv)       ← 注册到 AMBA 总线

设备匹配命中时，内核设备模型自动调用:
  总线 → bus->probe(dev)              ← 总线级别的入口
         ↓
      amba_probe(struct device *dev)       ← 参数是通用的 struct device
         ├─ pcdev = to_amba_device(dev)    ← 转回 amba_device
         ├─ pcdrv = to_amba_driver(dev->driver)  ← 从 dev->driver 找回 mmci_driver
         └─ pcdrv->probe(pcdev, id)        ← 调用驱动自己的 probe
               ↓
            mmci_probe(pcdev, id)          ← ★ 最终到达
```

两个 `probe` 的参数完全不同，不是同一个函数：

| `probe` 所在 | 位置 | 参数类型 | 职责 |
|-------------|------|----------|------|
| `amba_bustype.probe` = `amba_probe` | bus.c (总线框架) | `(struct device *)` | 类型转换、时钟使能、调度到驱动 |
| `mmci_driver.probe` = `mmci_probe` | mmci.c (驱动) | `(struct amba_device *, const struct amba_id *)` | 真正的硬件初始化 |

**板上验证结果**（ATK 板实际启动后）：

```
# 设备出现在 AMBA 总线下，不在 platform 总线
/sys/bus/amba/devices/48230000.mmc
/sys/bus/amba/devices/48220000.mmc           ← SDMMC1 也在 AMBA 总线下

# 驱动链接指向 amba 总线
ls -l /sys/devices/platform/soc@0/42080000.bus/48230000.mmc/driver
  → ../../../../../bus/amba/drivers/mmci-pl18x

# platform 总线下找不到 sdmmc
ls /sys/bus/platform/devices/ | grep sdmmc  → 无输出
```

这也解释了 sysfs 中的命名差异——DTS 节点名是 `sdmmc2`，但 AMBA 总线创建设备时用 `of_device_make_bus_id()` 生成 `48230000.mmc`（地址 + 驱动关联名）。如果走 platform 总线，名字会是 `48230000.sdmmc`。

### 4.3.3 Probe 逐段分析

`mmci_probe()` 的输入是 AMBA 匹配的结果，输出是一个可以管理 eMMC 卡的 `mmc_host` 实例，已配好时钟、中断、DMA，并注册到 core 层。

```
输入:                         执行:                                    输出:
  amba_device (硬件基址, IRQ)  →  mmc_alloc_host                   →  mmc_host
  amba_id     (variant 数据)      ├─ mmci_of_parse (DTS → caps)       (已注册到 core,
                                   ├─ pinctrl / clocks / ioremap      卡探测即将开始)
                                   ├─ variant->init (DLYB, IDMA)
                                   ├─ regulator / IRQ / DMA
                                   └─ mmc_add_host (移交 core)
```

按执行顺序分解为 10 步，可分为四个阶段：

| 阶段 | 步骤 | 做的事 |
|------|------|--------|
| **结构体分配** | ①② | 分配 `mmc_host` + `mmci_host`，解析 DTS 属性为 caps 位 |
| **硬件配置** | ③④⑤⑥⑦⑧ | pinctrl、时钟、ioremap、变体初始化、regulator、中断 |
| **数据通道** | ⑨ | DMA（IDMA 描述符链）设置 |
| **移交 Core** | ⑩ | `mmc_add_host()` → 卡探测流程启动 |

---

**① 分配 mmc_host — `mmc_alloc_host(sizeof(struct mmci_host), &dev->dev)`**

这是 MMC 子系统的核心分配函数。一次 `kzalloc` 分配连续的两块空间：

```c
struct mmc_host *mmc_alloc_host(int extra, struct device *dev)
{
    host = kzalloc(sizeof(struct mmc_host) + extra, GFP_KERNEL);
    // sizeof(struct mmc_host)   → 通用部分
    // extra = sizeof(struct mmci_host) → 驱动私有部分
    return host;
}
```

这两个结构体各管各的（见 03 篇 3.4 节、3.5 节）：

| 结构体 | 所属层 | 职责 | 关键字段 |
|--------|--------|------|----------|
| `mmc_host` | core 层通用 | 管理控制器、调度 I/O、协调卡生命周期 | `ops`、`caps`、`f_max`、`card` |
| `mmci_host` | mmci 驱动私有 | 操作 SDMMC2 硬件寄存器 | `base`（0x48230000）、`clk`、`variant`、`dlyb` |

core 层代码只访问 `mmc_host`，从不过问 `mmci_host` 的内容；host driver 通过 `mmc_priv()` 拿到自己的私有数据指针。这种"一次分配、两层隔离"的设计保证了 core 层对具体硬件的零依赖。

关键在 `struct mmc_host` 的末尾定义了一个**柔性数组成员**（flexible array member）：

```c
struct mmc_host {
    ...
    unsigned long private[] ____cacheline_aligned;  // 零长度，不占 sizeof
};
```

`kzalloc(sizeof(struct mmc_host) + extra)` 分配的总大小超出 `struct mmc_host` 自身，多出来的部分恰好被 `private[]` 柔性数组覆盖。`mmc_priv(mmc)` 宏返回 `(void *)host->private`，即柔性数组的起始地址——也就是 `struct mmci_host` 空间的起点。

```
<------ sizeof(struct mmc_host) -----><--- sizeof(struct mmci_host) --->
┌────────────────────────────────────┬──────────────────────────────────┐
│ struct mmc_host                    │ struct mmci_host                │
│ (core 层使用)                      │ (mmci 驱动私有)                  │
│ ops, caps, f_min, f_max ...        │ base, clk, dma, dlyb ...        │
│                                    │                                  │
│ unsigned long private[] ───────────→                                  │
│ (柔性数组，零开销)                  │                                  │
└────────────────────────────────────┴──────────────────────────────────┘
                                     ↑
                           mmc_priv(mmc) = host->private
```

`struct mmc_host` 和 `struct mmci_host` 在内存中紧挨着，`mmc_priv(mmc)` 返回的地址就是两者交界处。在 `mmci_probe` 中通过一次转换获得私有数据指针：

```c
/* mmci.c:2229-2234 */
mmc = mmc_alloc_host(sizeof(struct mmci_host), &dev->dev);
host = mmc_priv(mmc);    // ← host 指向 struct mmci_host 区域
host->mmc = mmc;          // ← 反向指针，私有数据也能访问通用部分
```

分配完成后，关键字段初始化：

```c
mmc->ops = &mmci_ops;     // core 层通过 ops 调 mmci
```

---

**② 解析 DTS 属性 — `mmci_of_parse()`**

这个函数读取设备树中的 MMC 通用属性，转换成 `mmc->caps` 和 `mmc->caps2` 的位标志。

ATK 板 DTS 属性与 caps 位对照表：

| DTS 属性 | Caps 位 | 含义 |
|----------|---------|------|
| `bus-width <8>` | caps |= MMC_CAP_8_BIT_DATA | 8 位总线 |
| `non-removable` | caps2 |= MMC_CAP2_NO_PRESCAN_POWERUP | 非可插拔（固定 eMMC） |
| `mmc-ddr-1_8v` | caps |= MMC_CAP_1_8V_DDR | 支持 DDR52 @ 1.8V |
| `cap-mmc-highspeed` | caps |= MMC_CAP_MMC_HIGHSPEED | 支持 HS 时序 |
| `max-frequency <166000000>` | mmc->f_max = 166000000 | 最大时钟频率 |

这些位标志在后续阶段三的时序选择中决定可用路径。可以回看 01 篇的 debugfs 输出来验证：

```
# cat /sys/kernel/debug/mmc1/ios
clock:          52000000 Hz
bus mode:       2 (push-pull)
bus width:      3 (8-bit)
timing spec:    7 (mmc-ddr52)
```

这里的 `timing spec: 7` 对应 `MMC_TIMING_MMC_DDR52`，`clock: 52000000 Hz` 对应 DDR52 速率。注意最终时序取决于卡的能力和主机能力的交集，不是 DTS 单方面决定的——见 4.4.6 节的分析。

回顾 probe 顺序：`mmc_of_parse()` 在步骤 ② 先执行，`variant->init()` 在步骤 ⑥ 才跑。所以 **变体初始化不设任何 `MMC_CAP_*` 位**（步骤 ⑥ 会验证这一点）。HS200/HS400 这些能力位如果存在，只能是 DTS 的功劳。

那 HS200 和 HS400 的能力位从哪里来？回到 `mmc_of_parse()`（`host.c:393-398`）：

```c
if (device_property_read_bool(dev, "mmc-hs200-1_8v"))
    host->caps2 |= MMC_CAP2_HS200_1_8V_SDR;
if (device_property_read_bool(dev, "mmc-hs400-1_8v"))
    host->caps2 |= MMC_CAP2_HS400_1_8V | MMC_CAP2_HS200_1_8V_SDR;
```

HS200 和 HS400 的能力位完全由 DTS 属性 `mmc-hs200-1_8v` 和 `mmc-hs400-1_8v` 控制。如果 DTS 中没有这两个属性，`caps2` 中就不会有对应的位。

ATK 板的 sdmmc2 节点（`stm32mp257d-atk-bsp.dts:715`）：

```
&sdmmc2 {
    non-removable;
    no-sd;
    no-sdio;
    bus-width = <8>;
    mmc-ddr-1_8v;
    // 没有 mmc-hs200-1_8v
    // 没有 mmc-hs400-1_8v
};
```

SoC 级节点（`stm32mp251.dtsi:2262`）同样没有定义这两个属性——它只有 `cap-mmc-highspeed` 和 `max-frequency = <166000000>`。

所以 ATK 板 `host->caps2` 中没有 `MMC_CAP2_HS200_1_8V_SDR` 或 `MMC_CAP2_HS400_1_8V`。这不是"硬件不支持"（Kingston 58A43A eMMC 芯片实际支持 HS200/HS400），而是板级设计没有在 DTS 中声明启用这些模式。

---

这就直接影响了阶段三的时序选择。进入 `mmc_init_card()` → `mmc_select_card_type()`（`mmc.c:194`），内核逐位计算可用模式 `avail_type`：

| `avail_type` 位 | 判断条件 | ATK 板的 `host->caps/caps2` | 结果 |
|----------------|---------|---------------------------|------|
| `HS_26` / `HS_52` | `caps & MMC_CAP_MMC_HIGHSPEED` ∩ `card_type & HS_xx` | DTS `cap-mmc-highspeed` ✅ | 生效 |
| `DDR_1_8V` | `caps & MMC_CAP_1_8V_DDR` ∩ `card_type & DDR_1_8V` | DTS `mmc-ddr-1_8v` ✅ | 生效 |
| `HS200_1_8V` | `caps2 & MMC_CAP2_HS200_1_8V_SDR` ∩ `card_type & HS200_1_8V` | **不存在** ❌ | 跳过 |
| `HS400_1_8V` | `caps2 & MMC_CAP2_HS400_1_8V` ∩ `card_type & HS400_1_8V` | **不存在** ❌ | 跳过 |

`avail_type` 最终只包含 `HS_26 | HS_52 | DDR_1_8V`。eMMC 芯片虽然硬件支持 HS200/HS400，但 host 侧没有声明，所以这些模式不进入候选集。

然后 `mmc_select_timing()`（`mmc.c:1538`）按优先级依次尝试：

```
avail_type 包含: HS_26, HS_52, DDR_1_8V    ← HS200/HS400 不在候选集
                              ↓
mmc_select_timing() 优先级：
  ① HS400ES → avail_type 无此位，跳过
  ② HS200   → avail_type 无此位，跳过
  ③ HS      → avail_type 有 HS_26|HS_52
               → mmc_select_hs()，CMD6 设 HS 时序，成功 ✓
```

回到 `mmc_init_card()`（`mmc.c:1828-1832`），此时卡处于 HS 模式：

```c
} else {
    /* Select the desired bus width optionally */
    err = mmc_select_bus_width(card);       // 尝试 8-bit，成功
    if (err > 0 && mmc_card_hs(card)) {
        err = mmc_select_hs_ddr(card);       // DDR52 降级路径
        if (err)
            goto free_card;
    }
}
```

`mmc_select_hs_ddr()` 检查 `avail_type & EXT_CSD_CARD_TYPE_DDR_52`（= `DDR_1_8V | DDR_1_2V`）——由于 DTS 声明了 `mmc-ddr-1_8v`，这个检查通过，DDR52 启用。最终时序就是 `MMC_TIMING_MMC_DDR52`。

如果从 DTS 中去掉 `mmc-ddr-1_8v`，则 `avail_type` 中只有 `HS_26 | HS_52`，DDR52 不会进入候选集，最终只能跑在 HS 52MHz 1-bit SDR（约 6.5 MB/s）。相比 DDR52 8-bit 带宽（约 104 MB/s），差距显著。

一句话总结：**DTS 属性 → `caps2` → `avail_type` → 时序选择**，这条链路上每一级都在收窄可选模式。ATK 板的 DTS 没有声明 HS200/HS400，这些模式从候选集中被排除，所以时序选择直接落到 HS → DDR52。这是正常的板级配置决策，与硬件能力是否达标无关。

---

**③ pinctrl 配置**

```c
devm_pinctrl_get_select_default(dev);
```

STM32MP2 的 sdmmc2 有三组 pinctrl 状态，在 DTS 中定义（`stm32mp257d-atk-bsp.dts:716-719`）：

```
pinctrl-names = "default", "opendrain", "sleep";
pinctrl-0 = <&sdmmc2_b4_pins_a &sdmmc2_d47_pins_a>;    // 推挽
pinctrl-1 = <&sdmmc2_b4_od_pins_a &sdmmc2_d47_pins_a>; // CMD 开漏
pinctrl-2 = <&sdmmc2_b4_sleep_pins_a &sdmmc2_d47_sleep_pins_a>; // 模拟
```

三组状态的引脚配置差异只在 CMD 引脚（E15）：

| 状态 | CMD 引脚（E15） | D0-D7 | 用途 |
|------|----------------|-------|------|
| `default` | `drive-push-pull` | `drive-push-pull` | 正常数据传输（推挽驱动） |
| `opendrain` | `drive-open-drain` | `drive-push-pull` | 卡识别阶段（开漏仲裁） |
| `sleep` | `ANALOG` | `ANALOG` | 低功耗休眠 |

Probe 阶段选择的 `default` 状态使所有引脚处于推挽模式。但在 probe 期间（`mmci.c:2246-2260`），驱动还会预先查找 `opendrain` 状态：

```c
if (!variant->opendrain) {                // STM32MP2 无开漏寄存器位
    host->pinctrl = devm_pinctrl_get(&dev->dev);
    host->pins_opendrain = pinctrl_lookup_state(host->pinctrl,
                                                MMCI_PINCTRL_STATE_OPENDRAIN);
}
```

这里的关键是 `variant_stm32_sdmmcv3` 没有定义 `.opendrain` 字段（为零）。对比其他变体：

| 变体 | `.opendrain` | 开漏实现方式 |
|------|-------------|-------------|
| ARM 参考设计 | `MCI_ROD`（寄存器位） | 写 PWR 寄存器 bit |
| STM32MP2 | 0（未定义） | pinctrl 切换引脚模式 |

因为 STM32MP2 的 SDMMC2 控制器没有硬件开漏控制位（PWR 寄存器中无 `MCI_ROD`），开漏切换只能通过 pinctrl 重配置 CMD 引脚的电气特性实现。

什么时候切换开漏？在阶段三的卡探测流程中（`mmc_init_card`，`mmc.c`）：

```
① CMD0 前: mmc_set_bus_mode(OPENDRAIN)  → pinctrl → "opendrain" 状态
      ↓                           CMD 引脚变为开漏
② CMD2 识别完成: RCA 分配成功
      ↓
③ RCA 后: mmc_set_bus_mode(PUSHPULL)    → pinctrl → "default" 状态
      ↓                           CMD 引脚恢复推挽
```

这个切换通过 `mmci_set_ios()`（`mmci.c:1994-1997`）实现：

```c
if (ios->bus_mode == MMC_BUSMODE_OPENDRAIN)
    pinctrl_select_state(host->pinctrl, host->pins_opendrain);
else
    pinctrl_select_default_state(mmc_dev(mmc));
```

**为什么需要开漏？** 参见 02 篇 **2.3 MMC 状态机** 和 **2.3.1 初始化流程中的状态变迁**——Step 0 明确了识别阶段必须工作在 Open Drain 模式，每个步骤都标明了时钟和总线模式。CMD1 和 CMD2 都涉及多个卡同时驱动 CMD 线：

| 命令 | 多卡如何共享 CMD 线 | 依赖开漏的原因 |
|------|-------------------|--------------|
| CMD1 | 所有卡同时回复 OCR，用**线与**取交集（busy 位、电压窗口） | 推挽下两卡输出相反电平会短路 |
| CMD2 | 所有卡同时发 CID，用**逐位仲裁**（bitwise arbitration）选出一张 | 失利卡停止输出、胜者独占，开漏是"低覆盖高"安全的前提 |

仲裁失利的卡停在 **Ready** 状态，下一轮 CMD2 重新参与。每轮只识别一张卡，CMD2→CMD3 循环直到所有卡都有 RCA。识别完成后，所有后续命令带 RCA 寻址——每次只有一个目标卡响应，不再需要开漏，切回推挽才能跑高速。

**为什么 RCA 分完后推挽就安全了？** CMD3 分配的 RCA 成为每张卡在总线上的唯一 ID。之后的命令（如 CMD7 SELECT/DESELECT、CMD18 READ_MULTIPLE_BLOCK）都带 RCA 参数：

```
CMD7(1)  →  只有 RCA=1 的卡响应           ← 一对一，推挽安全
CMD7(2)  →  只有 RCA=2 的卡响应           ← 一对一，推挽安全
CMD18(1) →  只有 RCA=1 的卡传输数据块     ← 一对一，推挽安全
```

> 以上多卡仲裁机制是 MMC 协议历史设计，现实中不存在多条 eMMC/SD 共享同一总线的情景（见 4.4.3 节说明）。

**回到 ATK 板**，物理上只有一片 eMMC，不会有真正的仲裁冲突。但协议层不管你有几张卡，流程必须走完。就像 TCP 三次握手——即使你知道网络没问题，协议也得做完 SYN→SYN-ACK→ACK 才能发数据。这是协议兼容性要求，不是运行时优化能跳过的。

---

**④ 时钟配置**

SDMMC2 在硬件层有两个时钟域，分别驱动不同功能：

| 信号名 | Linux 名称 | 来源 | 用途 |
|--------|-----------|------|------|
| ICN（interconnect） | `ck_icn_m_sdmmc2` | FLEXGEN 1 → `ck_icn_sdmmc` → gate | APB 寄存器访问 |
| KER（kernel） | `ck_ker_sdmmc2` | FLEXGEN 52 → gate → `CK_KER_SDMMC2` | SDMMC 通信位时钟 |

两者共用同一个硬件门控位 `GATE_SDMMC2`（`RCC_SDMMC2CFGR` bit 1），但来源不同——ICN 时钟来自 bus fabric（所有外设共享的互连时钟域），KER 时钟来自独立的 FLEXGEN 通道（可编程分频）。这意味着**它们不是同一个物理时钟**。

```
ck_icn_sdmmc (FLEXGEN_1) ──→ gate ─→ ck_icn_m_sdmmc2  (APB 接口，DTS 未声明)
                                  ↑
FLEXGEN_52 ──→ gate ─→ ck_ker_sdmmc2 ──→ CK_KER_SDMMC2  (位时钟，DTS 声明)
               共用 GATE_SDMMC2
```

DTS 只声明了 `CK_KER_SDMMC2`（`stm32mp251.dtsi:2258`）：

```c
clocks = <&rcc CK_KER_SDMMC2>;
clock-names = "apb_pclk";        // PrimeCell 惯例名，AMBA 框架要求
```

`apb_pclk` 是 AMBA PrimeCell 框架要求的时钟名——AMBA 框架在 `amba_probe()` 中用这个名字获取时钟并使能（`bus.c:66-70`）。`mmci_probe` 中同样获取这个时钟作为 SDMMC 通信时钟（`mmci.c:2268`）：

```c
host->clk = devm_clk_get(&dev->dev, NULL);        // CK_KER_SDMMC2
clk_prepare_enable(host->clk);
host->mclk = clk_get_rate(host->clk);              // 读实际频率
```

**关键结论：** `CK_KER_SDMMC2` 就是 SDMMC2 的 kernel 时钟，DTS 没有额外声明 APB 接口时钟。AMBA 框架拿到的 `apb_pclk` 实际上也是 `CK_KER_SDMMC2`——但硬件上存在独立的 ICN 时钟，它与 KER 时钟共用门控位，使能 `CK_KER_SDMMC2` 时 ICN 也被使能，所以寄存器访问能正常工作。

如果 `host->mclk` 超过变体的 `f_max` 上限（`variant_stm32_sdmmcv3.f_max = 267MHz`），还会调低频率：

```c
if (host->mclk > variant->f_max) {
    clk_set_rate(host->clk, variant->f_max);
    host->mclk = clk_get_rate(host->clk);
}
```

**时钟树：从 PLL 到 SDMMC2**

```
PLL4 ─┐
PLL5 ─┤
PLL6 ─┤        ┌──────────────────┐      ┌──────────────┐
PLL7 ─┼───mux──→ FLEXGEN_52       │      │ RCC_         │
PLL8 ─┤        │  (分频+门控)     ──→    │ SDMMC2CFGR   │
HSI  ─┤        └──────────────────┘      │ bit 1        │
HSE  ─┘                                   │ (GATE_       │
                                          │  SDMMC2)     │
ck_icn_sdmmc (FLEXGEN_1) ──→ gate[1] ────→              │
                                          └──┬───┬──────┘
                                             │   │
                                             │   └──→ RIFSC 验证（只读，不写）
                                             ↓
                                      ┌──────────────┐
                                      │ ck_ker_sdmmc2│ → SDMMC2 位时钟
                                      │ ck_icn_m_    │ → APB 寄存器访问
                                      │ sdmmc2       │
                                      └──────────────┘
```

FLEXGEN 通道 52 可以从 8 个源（PLL4~PLL8、HSI、HSE、MSI）中选择一个，经过可编程分频后输出。ATK 板上实际来自哪个 PLL 由 U-Boot 或 TF-A 的时钟初始化代码配置，Linux 一般不再改变 FLEXGEN 的 mux 选择，只通过 `clk_set_rate` 调整分频系数。

**RCC 防火墙（RIFSC）**

```c
STM32_GATE_CFG(CK_KER_SDMMC2, ck_ker_sdmmc2, SEC_RIFSC(77));
```

`SEC_RIFSC(77)` 标记该门控受 RIFSC（Resource Isolation Framework）保护。内核使能时钟前会读取 RIFSC 寄存器验证当前执行上下文（CA35 non-secure）是否有权限，未通过则 `clk_prepare_enable` 返回 `-EACCES`。RIFSC 的初始配置由底层固件完成（本文不展开）。

**内部的时钟分频：`mmci_sdmmc_set_clkreg()`**

CK_KER_SDMMC2 进入 SDMMC2 控制器后，还需要经过内部的分频器才能得到最终的 CMD/DAT 线频率。这个分频通过写 `MCI_CLKCR` 寄存器实现，逻辑封装在 `mmci_sdmmc_set_clkreg()`（`mmci_stm32_sdmmc.c:342`）：

```c
// cclk = mclk / (2 * clkdiv)  →  clkdiv = mclk / (2 * desired)
if (desired >= host->mclk && !ddr) {
    host->cclk = host->mclk;              // bypass: clkdiv=0
} else {
    clk = DIV_ROUND_UP(host->mclk, 2 * desired);
    host->cclk = host->mclk / (2 * clk);
}
```

| desired | clkdiv | 输出频率 | 场景 |
|---------|--------|---------|------|
| 400KHz | mclk / (2 × 400K) ≈ 207 | mclk / 414 ≈ 400KHz | 卡识别（开漏，限频） |
| 52MHz | mclk / (2 × 52M) ≈ 1 | mclk / 2 ≈ 52MHz | HS / DDR52 |
| 200MHz | desired 超 mclk 一半 | bypass，clkdiv=0 = mclk | HS200/HS400 |

**f_max / f_min 边界**

Probe 末尾确定频率上下限（`mmci.c:2316-2339`）：

```c
mmc->f_min = DIV_ROUND_UP(host->mclk, 2046);     // stm32_clkdiv：最大分频比
mmc->f_max = min(host->mclk, mmc->f_max);          // DTS max-frequency <166MHz>
```

ATK 板：`host->mclk ≈ 166MHz`，`f_min ≈ 166MHz / 2046 ≈ 81KHz`。

---

**⑤ 寄存器 ioremap**

```c
base = devm_ioremap_resource(dev, iobase);       // 物理地址 → 虚拟地址
```

SDMMC2 寄存器基址 `0x48230000`，映射范围 `0x1000`（4KB）。

---

**⑥ 变体初始化 — `variant->init()`**

`mmci_probe()` 在 ioremap 完成后调用（`mmci.c:2307-2308`）：

```c
if (variant->init)
    variant->init(host);
```

对于 `variant_stm32_sdmmcv3`，它指向 `sdmmc_variant_init()`（`mmci_stm32_sdmmc.c:779`）：

```c
void sdmmc_variant_init(struct mmci_host *host)
{
    struct device_node *np = host->mmc->parent->of_node;
    void __iomem *base_dlyb;
    struct sdmmc_dlyb *dlyb;

    host->ops = &sdmmc_variant_ops;           // 变体操作回调（clkdiv 计算等）
    host->pwr_reg = readl_relaxed(host->base + MMCIPOWER);

    /* 映射 DLYB 寄存器区域（reg 第二段: 0x44230800） */
    base_dlyb = devm_of_iomap(mmc_dev(host->mmc), np, 1, NULL);
    if (IS_ERR(base_dlyb))
        return;

    dlyb = devm_kzalloc(mmc_dev(host->mmc), sizeof(*dlyb), GFP_KERNEL);
    if (!dlyb)
        return;

    dlyb->base = base_dlyb;
    if (of_device_is_compatible(np, "st,stm32mp25-sdmmc2")) {
        dlyb->ops = &dlyb_tuning_mp25_ops;    // MP25 专用调谐操作
        dlyb->min_freq = 100000000;            // 100MHz 以下不需要调谐
    }
    /* ... 注册 DLYB 到 debugfs ... */
    host->dlyb = dlyb;
}
```

做了三件事：

1. **注册变体操作回调**：`sdmmc_variant_ops` 包含 STM32MP2 特有的 `clkdiv` 计算（`(rate / mclk) - 2`）、IDMA 设置等
2. **映射 DLYB 寄存器**：ioremap 第二段 `reg`（`0x44230800`），这是 STM32MP2 独有的延迟校准硬件
3. **选择 MP25 调谐操作集**：根据 compatible 区分 MP15 和 MP25 的 DLYB 配置

注意 `sdmmc_variant_init()` **没有设置任何 `MMC_CAP_*` 位**。HS200/HS400 的能力位不由它添加，它只关心 DLYB 调谐硬件——而调谐只在 HS200/HS400 模式才需要。

DLYB（Delay Block）是 STM32MP2 用于高频时序校准的关键硬件——通过调节数据线的延迟值来找到有效的采样窗口中心点。详见 03 篇的 3.7 节。

---

**⑦ 电源与 regulator**

```c
mmc_regulator_get_supply(mmc);                   // 获取 vmmc / vqmmc 供应
```

ATK 板上：
- `vmmc`：3.3V（eMMC 核心电源）
- `vqmmc`：1.8V（I/O 电源，由 `mmc-ddr-1_8v` 决定）

---

**⑧ 中断请求**

```c
devm_request_irq(dev, irq, mmci_irq, IRQF_SHARED, ...);
```

`mmci_irq()` 处理的中断事件掩码（`MCI_IRQENABLE` 中定义）：

| 中断标志 | 含义 |
|----------|------|
| `MCI_CMDONLY` | 命令发送完成 |
| `MCI_CMDRESPEND` | 命令响应接收完成 |
| `MCI_DATATIMEOUT` | 数据超时 |
| `MCI_DATACRCFAIL` | 数据 CRC 错误 |
| `MCI_DATAEND` | 数据传输完成 |
| `MCI_DATABLOCKEND` | 单个数据块传输完成 |
| `MCI_TXUNDERRUN` / `MCI_RXOVERRUN` | FIFO 溢出/空 |

---

**⑨ DMA 设置 — `mmci_dma_setup()`**

STM32MP2 不使用标准 DMAC，而是使用 IDMA（Internal DMA）：

```c
struct sdmmc_idma {
    struct sdmmc_lli_desc *sg_cpu;     // 描述符链 CPU 地址
    dma_addr_t sg_dma;                  // 描述符链 DMA 地址
};
```

每个描述符的结构（`sdmmc_lli_desc`）：

```c
struct sdmmc_lli_desc {
    u32 idmalar;        // 下一条描述符地址（链表指针）
    u32 idmabase;       // 数据缓冲区地址
    u16 idmasize;       // 传输长度（bytes）
};
```

IDMA 用描述符链实现 scatter-gather：每个描述符指向一块数据和下一个描述符，控制器自动遍历整个链，无需 CPU 逐块搬运。

---

**⑩ mmc_add_host — 移交 core 层**

```c
mmc_add_host(mmc);
```

这一步把 mmc_host 注册到 core 层，内部调用：

```c
int mmc_add_host(struct mmc_host *host)
{
    device_register(&host->class_dev);     // 出现在 /sys/class/mmc_host/mmcX/
    mmc_start_host(host);                   // 启动卡探测流程
    return 0;
}
```

`mmc_start_host()` 是阶段三的入口——probe 到这里结束，控制权从 mmci 驱动交回 core 层。

---

## 4.4 阶段三：Core 层自动探测卡

**状态机贯穿全程。** 整个卡探测过程是 MMC 状态机（02 篇 §2.3）的代码具象化——每条命令都在触发状态变迁。对照关系：

| 命令 | 状态迁移 | 发生位置 |
|------|---------|---------|
| CMD0 | 任意状态 → **Idle** | `mmc_go_idle()` — 复位总线 |
| CMD1 | Idle → **Ready** | `mmc_send_op_cond()` — 电压协商 |
| CMD2 | Ready → **Ident** | `mmc_send_cid()` — 多卡仲裁 |
| CMD3 | Ident → **Standby** | `mmc_set_relative_addr()` — 分配 RCA |
| CMD9 | Standby → Standby（不迁移） | `mmc_send_csd()` — 读 CSD |
| CMD7 | Standby → **Transfer** | `mmc_select_card()` — 选卡 |

### ATK 板 eMMC 探测总览

以下全景图中每条命令旁标注了状态变迁，以便与上表对照：

```
mmc_start_host()
  └─ mmc_rescan()
       └─ mmc_rescan_try_freq(400K)
            ├─ mmc_power_up()           推挽/1-bit/Legacy
            ├─ mmc_go_idle()            CMD0, **任意→Idle**（总线复位）
            ├─ mmc_attach_sdio()        → 跳过 (NO_SDIO)
            ├─ mmc_attach_sd()          → 跳过 (NO_SD)
            └─ mmc_attach_mmc()
                 ├─ mmc_set_bus_mode(OPENDRAIN)      ← 切 开漏
                 ├─ CMD1 轮询                        **Idle→Ready**
                 ├─ mmc_attach_bus(&mmc_ops)
                 └─ mmc_init_card()
                      ├─ CMD0                        **任意→Idle**（二次复位）
                      ├─ CMD1                        **Idle→Ready**
                      ├─ CMD2                        **Ready→Ident**
                      ├─ CMD3 + 切推挽                **Ident→Standby**
                      ├─ CMD9                        Standby（不迁移）
                      ├─ CMD7                        **Standby→Transfer**
                      ├─ CMD8 / CMD6 ×N              Transfer（数据操作）
                      └─ mmc_add_card()              → mmc_block probe
```

> CMD0 调了两次：第一次在 `mmc_rescan_try_freq` 中（推挽模式，复位总线上所有卡）；第二次在 `mmc_init_card()` 中（开漏模式，启动 eMMC 标准初始化序列）。两次功能不同。

接下来的 4.4.1~4.4.6 节将逐层展开这个路径中的每个环节。

### 4.4.1 探测入口

`mmc_add_host()` → `mmc_start_host()`：

```c
void mmc_start_host(struct mmc_host *host)
{
    host->f_init = max(freqs[0], host->f_min);     // 400KHz 或 f_min
    mmc_power_up(host, host->ocr_avail);            // 上电
    _mmc_detect_change(host, 0, false);              // 触发探测
}
```

`_mmc_detect_change()` 调度一个工作队列：

```c
void _mmc_detect_change(struct mmc_host *host, unsigned long delay, bool cd)
{
    mmc_schedule_delayed_work(&host->detect, delay);
}
```

这个 `host->detect` 在何处初始化？回到阶段二的 `mmc_alloc_host()`（`host.c:566`）：

```c
struct mmc_host *mmc_alloc_host(int extra, struct device *dev)
{
    /* ... 分配和零初始化 struct mmc_host ... */
    INIT_DELAYED_WORK(&host->detect, mmc_rescan);   // ← 绑定工作函数
    INIT_WORK(&host->sdio_irq_work, sdio_irq_work);
    timer_setup(&host->retune_timer, mmc_retune_timer, 0);
    /* ... */
}
```

`INIT_DELAYED_WORK` 将 `host->detect` 这个 `delayed_work` 的工作函数绑定为 `mmc_rescan`。所以 `_mmc_detect_change` 只是触发调度的包装，真正的工作函数在 `mmc_alloc_host` 时就定死了。完整的触发链：

```
mmci_probe()
  └─ mmc_alloc_host(..., dev)          → INIT_DELAYED_WORK(&detect, mmc_rescan)
  └─ mmc_add_host(host)
       └─ mmc_start_host(host)
            └─ _mmc_detect_change(host, 0, false)
                 └─ mmc_schedule_delayed_work(&detect, 0)
                      ↓ 工作队列线程
                      mmc_rescan()     ← 卡探测的真正入口
```

此时卡还处于**未上电/未定义**状态，状态机尚未启动。`mmc_rescan()` 的任务就是让卡从上电开始，逐条命令驱动状态机走完 `Idle → Ready → Ident → Standby → Transfer` 的完整初始化链。

工作队列处理函数 `mmc_rescan()`（`core.c:2207`）：

```c
void mmc_rescan(struct work_struct *work)
{
    static const unsigned freqs[] = { 400000, 300000, 200000, 100000 };

    for (i = 0; i < ARRAY_SIZE(freqs); i++) {
        unsigned int freq = freqs[i];
        if (freq > host->f_max) {
            if (i + 1 < ARRAY_SIZE(freqs))
                continue;            // 超 f_max 跳过当前档
            freq = host->f_max;       // 最后一档直接用 f_max
        }
        if (!mmc_rescan_try_freq(host, max(freq, host->f_min)))
            break;                     // 找到卡就停
        if (freqs[i] <= host->f_min)
            break;                     // 低于 f_min 也停
    }
}
```

`mmc_rescan_try_freq()` 被调用时的实际频率受 `host->f_max` 和 `host->f_min` 约束。ATK 板 `f_max = 166MHz`，`freqs[0] = 400KHz < f_max(166MHz)`，不满足跳过条件。第一次调用即为 `mmc_rescan_try_freq(host, 400000)`，成功探测到 eMMC 后 break，不再尝试后续档位。


### 4.4.2 卡类型识别

`mmc_rescan_try_freq()`（`core.c:2054`）的完整流程：

```
① mmc_power_up(host, host->ocr_avail)       → 上电 + 设初态（推挽/1-bit/Legacy）
② mmc_hw_reset_for_init(host)                → 硬件复位
③ sdio_reset(host)                           → CMD52 复位 SDIO
④ mmc_go_idle(host)                          → CMD0，所有卡复位到 Idle State
⑤
⑥ ── caps2 & MMC_CAP2_NO_SDIO? ── 有 → 跳过 ↓
│   └─ mmc_attach_sdio()  → CMD5   ─── 有响应? → SDIO 初始化，返回
│
⑦ ── caps2 & MMC_CAP2_NO_SD? ──── 有 → 跳过 ↓
│   └─ mmc_attach_sd()    → ACMD41 ─── 有响应? → SD 卡初始化，返回
│
⑧ ── caps2 & MMC_CAP2_NO_MMC? ──── 有 → 跳过 ↓
    └─ mmc_attach_mmc()   → CMD1   ─── 有响应? → eMMC 初始化，返回
                                      全部无响应 → mmc_power_off()，返回 -EIO
```

状态机角度：此时所有卡都在 **Idle State**（上一步 CMD0 强制复位的结果）。接下来三种识别命令各自尝试让卡离开 Idle：

| 命令 | 状态迁移 | 说明 |
|------|---------|------|
| CMD5 | SDIO 检测，无状态迁移 | 非 SDIO 设备不应答，留在 Idle |
| ACMD41 | SD 检测，无状态迁移 | 非 SD 卡不应答，留在 Idle |
| CMD1 | Idle → **Ready**（BUSY=1 时） | eMMC 卡的初始化完成信号 |

前两条跳过时不改变状态，CMD1 成功后才真正触发从 Idle 到 Ready 的第一次状态变迁。

顺序固定为 **SDIO → SD → MMC**（`core.c:2087` 注释：`Order's important: probe SDIO, then SD, then MMC`）。因为 CMD1 对某些 SD 卡也会有响应，所以必须先用 CMD5（仅 SDIO 响应）和 ACMD41（仅 SD 响应）排除前两种，剩下的才是 MMC。

ATK 板的 DTS 有 `no-sd` 和 `no-sdio` 标记，`caps2` 中包含 `MMC_CAP2_NO_SDIO` 和 `MMC_CAP2_NO_SD`，前两条直接跳过，最终落到 `mmc_attach_mmc()`。

下面逐个分析探测流程中的每个函数。

#### 4.4.2.1 `mmc_power_up()` — 上电与初态设置（`core.c:1332`）

先把要理解这个函数拆成三段来看：**core 层准备** → **回调到 host 驱动** → **等待硬件稳定**。

```c
void mmc_power_up(struct mmc_host *host, u32 ocr)
{
    if (host->ios.power_mode == MMC_POWER_ON)
        return;                    // 已上电则跳过

    mmc_pwrseq_pre_power_on(host); // power sequence 前处理（如 GPIO 使能）

    /* ────────── 第一阶段：准备上电 ────────── */

    host->ios.vdd = fls(ocr) - 1;        // ① 电压选择
    host->ios.power_mode = MMC_POWER_UP; // ② 标记"正在上电"

    mmc_set_initial_state(host);         // ③ 设置 IO 初态（关键）
    mmc_set_initial_signal_voltage(host);// ④ 设置信号电压

    mmc_delay(host->ios.power_delay_ms); // ⑤ 等电压稳定

    mmc_pwrseq_post_power_on(host);      // power sequence 后处理

    /* ────────── 第二阶段：完成上电 ────────── */

    host->ios.clock = host->f_init;      // ⑥ 设初始时钟
    host->ios.power_mode = MMC_POWER_ON; // ⑦ 标记"已上电"
    mmc_set_ios(host);                   // ⑧ 一次性写入硬件（关键）

    mmc_delay(host->ios.power_delay_ms); // ⑨ 等 74 个时钟周期
}
```

**① `host->ios.vdd = fls(ocr) - 1` — 把电压位图转成索引**

`ocr` 是 **host 侧**能提供的电压位图（`host->ocr_avail`，来自 regulator 能力），不是卡端的 OCR。`fls()`（find last set）返回最高位序号，减 1 得到一个从 0 开始的索引。这个 `vdd` 索引在 `mmci_set_ios()` 中通过 `mmc_regulator_set_ocr()` 设置 **`vmmc` regulator**（即卡的主供电电压 VCC）：

```c
// mmci.c:1944-1946
case MMC_POWER_UP:
    mmc_regulator_set_ocr(mmc, mmc->supply.vmmc, ios->vdd);
    // → mmc_ocrbitnum_to_vdd(vdd, &min_uV, &max_uV)
    // → regulator_set_voltage(supply, min_uV, max_uV)
    // → regulator_enable(supply)
```

所以 `vdd` 控制的是**卡供电电压（VCC）**，通常是 3.3V。**信号电压（I/O 电压 VCCQ）** 由后面的 `mmc_set_initial_signal_voltage()` 通过 `vqmmc` regulator 单独配置。两者分工：

| `ios` 参数 | regulator | 物理引脚 | 典型值 | 作用 |
|-----------|----------|---------|--------|------|
| `vdd` | `vmmc` | eMMC VCC 引脚 | 3.3V | 卡内部核心供电 |
| （signal voltage） | `vqmmc` | eMMC VCCQ 引脚 | 1.8V | I/O 引脚电平 |

**② `power_mode = MMC_POWER_UP` — 两阶段上电设计**

MMC 的上电分两步：

| 阶段 | `power_mode` | 做的事 |
|------|-------------|--------|
| 第一阶段 | `MMC_POWER_UP` | 打开 regulator（vmmc/vqmmc），但不输出时钟 |
| 第二阶段 | `MMC_POWER_ON` | 时钟开始输出，I/O 线全部使能 |

这种设计保证在时钟开始之前电源已经完全稳定。如果两个阶段合并，卡可能在电压未达到最小值时就收到时钟，状态机行为未定义。

**③ `mmc_set_initial_state()` — 把所有 IO 参数打到安全默认值**

```c
void mmc_set_initial_state(struct mmc_host *host)
{
    if (host->cqe_on)
        host->cqe_ops->cqe_off(host);     // 关 CMDQ（如果有）
    mmc_retune_disable(host);              // 关调谐（初始不需要）

    host->ios.chip_select = MMC_CS_DONTCARE;  // CS 引脚不关心
    host->ios.bus_mode = MMC_BUSMODE_PUSHPULL; // ← 推挽，不是开漏
    host->ios.bus_width = MMC_BUS_WIDTH_1;     // ← 1-bit，不是 8-bit
    host->ios.timing = MMC_TIMING_LEGACY;      // ← Legacy，不是 DDR/HS200
    host->ios.drv_type = 0;                    // 驱动类型 A
    host->ios.enhanced_strobe = false;         // 关增强选通

    mmc_set_ios(host);  // ← 写入硬件（触发 mmci_set_ios）
    mmc_crypto_set_initial_state(host);
}
```

六个参数的物理含义：

| 参数 | 值 | 为什么初始是这个 |
|------|----|----------------|
| `bus_mode = PUSHPULL` | 推挽 | 探测阶段还没有多卡仲裁，不需要开漏。开漏只在 CMD1/CMD2 阶段需要 |
| `bus_width = 1-bit` | 单线 | 还不知道卡支持多宽，先用最低配置通信 |
| `timing = LEGACY` | ≤25MHz | 初始必须跑最慢、最兼容的时序，等 CMD6 再协商高速 |
| `chip_select = DONTCARE` | 忽略 | eMMC 没有 CS 引脚（SPI 模式才有），写 DONTCARE 即可 |
| `drv_type = 0` | 驱动类型 A | 默认驱动器强度 |
| `enhanced_strobe = false` | 关 | HS400 增强选通，初始不需要 |

设置完成后调 `mmc_set_ios(host)`，这个函数只有两行：

```c
static inline void mmc_set_ios(struct mmc_host *host)
{
    host->ops->set_ios(host, &host->ios);  // → 回调到 mmci_set_ios()
}
```

`host->ops->set_ios` 就是阶段二赋值的 `mmci_ops.set_ios = mmci_set_ios`。这一步的本质是：**core 层把 `host->ios` 里攒好的所有参数一次推给 host 驱动**，host 驱动负责把这些参数写入硬件寄存器。

**④ `mmc_set_initial_signal_voltage()` — 试信号电压**

```c
void mmc_set_initial_signal_voltage(struct mmc_host *host)
{
    if (!mmc_set_signal_voltage(host, MMC_SIGNAL_VOLTAGE_330))
        ...
    else if (!mmc_set_signal_voltage(host, MMC_SIGNAL_VOLTAGE_180))
        ...
    else if (!mmc_set_signal_voltage(host, MMC_SIGNAL_VOLTAGE_120))
        ...
}
```

从 3.3V 开始试，不行就降 1.8V，再不行就 1.2V。**成功/失败的判断逻辑在三级调用链的底层。**

三级调用链如下：

```
mmc_set_signal_voltage(host, signal_voltage)         // core.c
  └─ host->ops->start_signal_voltage_switch(host, ios) // 回调到 host 驱动
       └─ mmci_sig_volt_switch(mmc, ios)                // mmci.c
            └─ mmc_regulator_set_vqmmc(mmc, ios)        // regulator.c
                 └─ mmc_regulator_set_voltage_if_supported(
                        vqmmc, min_uV, target_uV, max_uV)
                      ├─ regulator_is_supported_voltage()  ← 关键判断
                      └─ regulator_set_voltage_triplet()   ← 实际设置
```

**成败的判定在 `mmc_regulator_set_voltage_if_supported()`：**

```c
static int mmc_regulator_set_voltage_if_supported(
    struct regulator *regulator,
    int min_uV, int target_uV, int max_uV)
{
    /* ① 先查 regulator 硬件是否支持这个电压范围 */
    if (!regulator_is_supported_voltage(regulator, min_uV, max_uV))
        return -EINVAL;                // ← 不支持，返回负值 = 失败

    /* ② 检查当前电压是否已经是目标值 */
    current_uV = regulator_get_voltage(regulator);
    if (current_uV == target_uV)
        return 1;                      // ← 已经是目标值，但返回 1≠0 表示"没实际切换"

    /* ③ 实际切换电压 */
    return regulator_set_voltage_triplet(regulator, min_uV, target_uV, max_uV);
    // 成功返回 0，失败返回负值
}
```

`mmc_set_signal_voltage()` 的返回值逻辑：

```c
int mmc_set_signal_voltage(struct mmc_host *host, int signal_voltage)
{
    host->ios.signal_voltage = signal_voltage;          // 先设新值
    if (host->ops->start_signal_voltage_switch)
        err = host->ops->start_signal_voltage_switch(host, &host->ios);

    if (err)
        host->ios.signal_voltage = old_signal_voltage;  // 失败则恢复旧值

    return err;
}
```

所以 `mmc_set_initial_signal_voltage()` 中的 `!mmc_set_signal_voltage(xxx)` 判断的是：**返回 0 视为成功**（正数 1 也表示切换已完成），**返回负值视为失败**。

**ATK 板的具体路径：**

ATK 板 sdmmc2 节点的 DTS 配置（`stm32mp257d-atk-bsp.dts`）：

```dts
&sdmmc2 {
    vmmc-supply  = <&scmi_vdd_emmc>;   // vmmc = 卡供电（3.3V）
    vqmmc-supply = <&scmi_vddio2>;     // vqmmc = I/O 信号电压（1.8V）
    mmc-ddr-1_8v;                       // 声明 DDR 模式使用 1.8V
};
```

`&scmi_vddio2` 是通过 SCMI 固件接口控制的 PMIC 输出，在 ATK 板上硬件设计固定为 1.8V。所以：

```
① 试 3.3V (2700000~3600000μV)
   → regulator_is_supported_voltage(scmi_vddio2, 2.7V, 3.6V) → false
   → 返回 -EINVAL
   → else if 走下一个

② 试 1.8V (1700000~1950000μV)
   → regulator_is_supported_voltage(scmi_vddio2, 1.7V, 1.95V) → true（匹配 1.8V）
   → regulator_set_voltage_triplet → 成功（硬件已固在 1.8V，无需实际切换）
   → 返回 0
   → if (!ret) 成立，打印 "Initial signal voltage of 1.8v"
```

如果板子上连 `vqmmc` regulator 都没有（DTS 没配 `vqmmc-supply`），`mmci_sig_volt_switch()` 走 fallback：

```c
/* No vqmmc regulator, assume fixed regulator at 3.3V */
if (mmc->ios.signal_voltage == MMC_SIGNAL_VOLTAGE_330)
    return 0;        // 3.3V 无条件成功（假定硬件默认就是 3.3V I/O）
return -EINVAL;      // 1.8V/1.2V 不可能支持 → 失败
```

**小结：** 成功/失败的判断不靠"发命令看卡是否响应"，而是靠 **regulator 框架查约束表**。`regulator_is_supported_voltage()` 查的是 regulator 驱动注册时建立的 **电压约束表**（`n_voltages` + `min_uV`/`max_uV`）——这些约束在设备树中定义。`scmi_vddio2` 是 SCMI 管理的固定输出 regulator，它的约束就是"只能输出 1.8V，不可调"。请求 3.3V 时 regulator 框架看到约束表中没有这个档位，直接返回 `-EINVAL`，不会去操作 PMIC 硬件。

**⑤ 第一次 `mmc_delay(power_delay_ms)` — 等电压升到最小值**

`power_delay_ms` 默认 10ms（由 `mmc_alloc_host` 设为 10）。注释原文：

> This delay should be sufficient to allow the power supply to reach the minimum voltage.

这 10ms 里硬件在做的是：

```
t=0     regulator_enable 被调用，vmmc 开始从 0V 上升
         ↑
t=~2ms   vmmc 达到 3.3V（上升时间取决于电容负载）
         ↑
t=10ms  （安全余量）保证任何瞬态都已过去
         ↓
         mmc_delay 返回，电压完全稳定
```

**⑥ `host->ios.clock = host->f_init` — 设初始时钟频率**

`host->f_init = max(freqs[0], host->f_min)`。首次探测时为 400KHz。之所以初始用 400KHz 而不是直接跳到 52MHz：识别阶段必须工作在低速（协议要求 ≤400KHz），因为此时总线处于 Legacy timing + 开漏模式，信号上升沿受 RC 时间常数限制，跑不了高频。

**⑦ `power_mode = MMC_POWER_ON` — 第二阶段**

标记电源已稳定，准备让 host 驱动开启完整 I/O。

**⑧ `mmc_set_ios(host)` — 把时钟和 POWER_ON 写入硬件**

第二次回调 `mmci_set_ios()`，此时 host 驱动收到的 `ios` 包含：

```
power_mode = MMC_POWER_ON    // ← 新值
clock = 400000               // ← 新值
bus_mode = PUSHPULL          // ← initial_state 设的
bus_width = 1-bit            // ← initial_state 设的
timing = LEGACY              // ← initial_state 设的
```

`mmci_set_ios()` 收到 `MMC_POWER_ON` 后：

```c
// mmci.c:1956-1966
case MMC_POWER_ON:
    if (!IS_ERR(mmc->supply.vqmmc) && !host->vqmmc_enabled) {
        ret = regulator_enable(mmc->supply.vqmmc);  // 开 vqmmc（1.8V）
        host->vqmmc_enabled = true;
    }
    pwr |= MCI_PWR_ON;   // = 0x03，之后 writel(pwr, base + MMCIPOWER)
    break;
```

`MCI_PWR_ON = 0x03` 写入 **MMCIPOWER 寄存器（基址 + 0x000，RM 中称 SDMMC_POWER）**。这个寄存器的 **PWRCTRL[1:0] 字段（bits 1:0）** 控制 SDMMC IP 核本身的功能状态。

根据参考手册（RM0457 §40.10.1），PWRCTRL[1:0] 定义如下：

| PWRCTRL[1:0] | 状态 | SDMMC 行为 |
|:---:|------|-----------|
| 00 | Reset / Power-off | SDMMC 禁用，时钟停止，D[7:0] 和 CMD 高阻，CK 拉低 |
| 01 | Reserved | 写入不生效，值不变 |
| 10 | Power-cycle | SDMMC 禁用，时钟停止，D[7:0]、CMD、CK 全部拉低 |
| **11** | **Power-on** | **卡被时钟驱动。前 74 个 SDMMC_CK 周期 SDMMC 仍处于禁用状态；74 周期后 SDMMC 使能，D[7:0]、CMD、CK 按 SDMMC 操作正常控制** |

> 写入 11 后，该寄存器被硬件锁定，再次写入被忽略（PWRCTRL 保持 11）。

**`MCI_PWR_ON = 0x03` 就是设置 PWRCTRL[1:0] = 11，让 SDMMC 进入 Power-on 状态。** 此时 SDMMC 开始输出时钟给卡，但前 74 个时钟周期控制器本身保持禁用——这正是代码中第二个 `mmc_delay()` 的原因（之后详述）。

PWRCTRL 控制的是 SDMMC IP 核内部的电源状态机——相当于 SDMMC 模块的"使能开关"。外部电源（vmmc/vqmmc）由 regulator 框架独立管理，与此无关。

> 注：这个寄存器的高位（bit 2~8）还包含了 VSWITCH、VSWITCHEN、DIRPOL 及方向控制位，用于 1.8V 信号电压切换流程，将在 ④ 中详述。

**理顺三个独立的电源管理层次：**

| 控制对象 | 机制 | 实现位置 |
|---------|------|---------|
| vmmc（卡供电 3.3V） | regulator 框架 | `mmc_power_up()` → `mmc_regulator_set_ocr(vmmc)` |
| vqmmc（I/O 电压 1.8V） | regulator 框架 | `mmci_set_ios()` → `regulator_enable(vqmmc)` |
| SDMMC IP 使能（PWRCTRL） | SDMMC_POWER 寄存器 | `mmci_set_ios()` → `pwr |= MCI_PWR_ON` |
| CMD 引脚电气模式 | pinctrl 框架 | `mmci_set_ios()` → `pinctrl_select_state()` |

**`MCI_PWR_ON` 的作用可以总结为：**

在 vmmc 和 vqmmc 都已供电的前提下，写 PWRCTRL = 11 通知 SDMMC 控制器："电源已经稳定，开始输出时钟驱动卡"。这不是开 regulator，而是让 SDMMC IP 核退出复位状态、开始工作。

**⑨ 第二次 `mmc_delay()` — 等 74 个时钟周期**

注释原文：

> This delay must be at least 74 clock sizes, or 1 ms, or the time required to reach a stable voltage.

为什么是 74 个周期？参考手册（RM0457 §40.10.1）给出了更直接的解释：

> PWRCTRL = 11 (Power-on): the card is clocked, **The first 74 SDMMC_CK cycles the SDMMC is still disabled**. After the 74 cycles the SDMMC is enabled and the SDMMC_D[7:0], SDMMC_CMD and SDMMC_CK are controlled according the SDMMC operation.

也就是说，写 PWRCTRL = 11 之后，**SDMMC 控制器本身在前 74 个周期处于"自禁用"状态**——它在等待自己的内部逻辑稳定（时钟使能、输出驱动准备）。74 个周期后控制器才完全使能，此时才能发第一条命令。

从卡的角度，JEDEC 规范也要求 **上电后 host 必须提供至少 74 个时钟周期** 让卡内部 POR 完成。两侧的需求一致：host 控制器需要 74 周期自稳定，卡也需要 74 周期完成 POR。

以 400KHz 计，74 周期 ≈ 185μs，`power_delay_ms` 默认 10ms，实际延时远超最小值——安全余量。

---

**小结：`mmc_power_up()` 的执行效果**

```
调用前：                mmc_power_up 执行后：
SDMMC2 寄存器            SDMMC2 寄存器
  PWRCTRL[1:0] = 00       PWRCTRL[1:0] = 11 (Power-on)
  MCI_CLKCR = 0           MCI_CLKCR = 400KHz 分频值
  CMD/DAT: 高阻            CMD/DAT: 推挽输出, 1-bit, 1.8V
  vmmc: 0V                 vmmc: 3.3V（稳定）
  vqmmc: 0V                vqmmc: 1.8V（稳定）

卡状态：无电                卡状态：Idle State（等待 CMD0）
```

**状态机：** 上电完成后卡自动进入 **Idle State**，时钟已经开始输出（400KHz），CMD/DAT 线处于推挽模式。但此时还没有发任何命令——卡在 Idle State 里等着接收第一条命令（CMD0）。

#### 4.4.2.2 `mmc_hw_reset_for_init()` — 硬件复位（`core.c:2004`）

```c
static void mmc_hw_reset_for_init(struct mmc_host *host)
{
    mmc_pwrseq_reset(host);                 // power seq（DTS pwrseq 属性）

    if (!(host->caps & MMC_CAP_HW_RESET) || !host->ops->card_hw_reset)
        return;                             // 不支持硬件复位 → 跳过
    host->ops->card_hw_reset(host);         // 拉 RST_n 引脚
}
```

注释说明了必要性：

> Some eMMCs (with VCCQ always on) may not be reset after power up, so do a hardware reset if possible.

某些 eMMC 的 VCCQ 常供电，单纯上下电 VCC 不一定会复位内部状态机，需要拉 RST_n 引脚。ATK 板的 mmci 驱动没有实现 `.card_hw_reset`，此步实际为空。

**状态机：** 硬件复位将卡强制拉回 **Idle State**，不受之前状态影响（RST_n 是异步复位信号，优先级高于任何命令）。

#### 4.4.2.3 `sdio_reset()` — CMD52 复位 SDIO（`sdio_ops.c:202`）

```c
int sdio_reset(struct mmc_host *host)
{
    u8 abort;
    ret = mmc_io_rw_direct_host(host, 0, 0, SDIO_CCCR_ABORT, 0, &abort);
    if (ret)
        abort = 0x08;
    else
        abort |= 0x08;                       // 置 RESET 位
    return mmc_io_rw_direct_host(host, 1, 0, SDIO_CCCR_ABORT, abort, NULL);
}
```

CMD52 是 SDIO 独有的 I/O 读写命令（非 SDIO 设备忽略）。向 CCCR 寄存器 `0x06`（ABORT）写 bit 3 复位 SDIO 卡。注释：

> sdio_reset sends CMD52 to reset card. Since we do not know if the card is being re-initialized, just send it. CMD52 should be ignored by SD/eMMC cards.

ATK 板有 `MMC_CAP2_NO_SDIO`，此步被外层 `if (!(host->caps2 & MMC_CAP2_NO_SDIO))` 跳过。

#### 4.4.2.4 `mmc_go_idle()` — CMD0 总线复位（`mmc_ops.c:147`）

```c
int mmc_go_idle(struct mmc_host *host)
{
    struct mmc_command cmd = {};

    if (!mmc_host_is_spi(host)) {
        mmc_set_chip_select(host, MMC_CS_HIGH);   // 拉高 DAT3/CS
        mmc_delay(1);
    }

    cmd.opcode = MMC_GO_IDLE_STATE;                // = 0
    cmd.arg = 0;
    cmd.flags = MMC_RSP_SPI_R1 | MMC_RSP_NONE | MMC_CMD_BC;
    // MMC_CMD_BC = 广播命令，无响应

    err = mmc_wait_for_cmd(host, &cmd, 0);
    mmc_delay(1);

    if (!mmc_host_is_spi(host))
        mmc_set_chip_select(host, MMC_CS_DONTCARE);
}
```

**CMD0 的双重含义：**

CMD0（GO_IDLE_STATE）是 MMC/SD 协议中最基础的复位命令——总线上所有卡收到后无条件回到 Idle State。但 CMD0 还有一个隐藏的第二重含义，取决于 **DAT3 线的电平**：

| DAT3 电平 | CMD0 的含义 |
|-----------|-----------|
| HIGH | 正常复位到 Idle State |
| LOW | 切换到 SPI 模式（SD 卡规范定义） |

原因是 SD 卡的 DAT3 引脚在 SPI 模式下被复用为 **nCS（片选）**。如果卡在收到 CMD0 时检测到 DAT3 为 LOW，就会认为主机想切换到 SPI 模式。

**CS_HIGH 的作用：**

```c
if (!mmc_host_is_spi(host)) {
    mmc_set_chip_select(host, MMC_CS_HIGH);
```

内核注释原文：

> Non-SPI hosts need to prevent chipselect going active during GO_IDLE; that would put chips into SPI mode. Remind them of that in case of hardware that won't pull up DAT3/nCS otherwise.

这段代码只在非 SPI 模式下执行。其含义是：**我们现在以 native 模式运行，要确保 DAT3 拉高，让卡也留在 native 模式，不会因为 DAT3 浮空 LOW 而误入 SPI 模式。**

有些控制器没有在 DAT3 上做内部上拉，如果不显式设 CS HIGH，DAT3 可能浮空为 LOW。CMD0 一到，卡就认为主机要求切换 SPI 模式——显然不是我们想要的。

SPI host 不走这个分支，因为 SPI 模式下 DAT3/nCS 由 SPI 协议层管理（每个帧都要拉低再拉高），core 层不去干涉。

**CMD0 发出后：**

```c
if (!mmc_host_is_spi(host))
    mmc_set_chip_select(host, MMC_CS_DONTCARE);
```

复位完成后 CS 恢复 DONTCARE，把 DAT3 的控制权交还给 host 驱动管理器。

**对 STM32MP2 / SDMMC2 的影响：**

`mmci_set_ios()` 不处理 `ios.chip_select`（查过源码确认无相关代码），所以这个操作对 STM32MP2 没有硬件效果——可能 SDMMC2 硬件内部已做了 DAT3 上拉，不需要软件干预。但 MMC 核心层作为通用代码，对所有非 SPI host 都做这个保护，无论底层控制器是否需要。

#### 4.4.2.5 ⑤⑥⑦ 三路卡类型识别

核心逻辑是顺序调用三个 `mmc_attach_*()` 函数，每个内部发一条识别命令：

```c
// core.c:2087-2098
/* Order's important: probe SDIO, then SD, then MMC */
if (!(host->caps2 & MMC_CAP2_NO_SDIO))
    if (!mmc_attach_sdio(host)) return 0;    // CMD5

if (!(host->caps2 & MMC_CAP2_NO_SD))
    if (!mmc_attach_sd(host))   return 0;    // ACMD41

if (!(host->caps2 & MMC_CAP2_NO_MMC))
    if (!mmc_attach_mmc(host))  return 0;    // CMD1
```

三条识别命令的对比如下：

| | CMD5 (SDIO) | ACMD41 (SD) | CMD1 (MMC) |
|--|------------|-------------|------------|
| 封装函数 | `mmc_send_io_op_cond()` | `mmc_send_app_op_cond()` | `mmc_send_op_cond()` |
| opcode | 5 | 41（需前缀 CMD55） | 1 |
| flags | `MMC_RSP_R4 \| MMC_CMD_BCR` | `MMC_RSP_R3 \| MMC_CMD_BCR` | `MMC_RSP_R3 \| MMC_CMD_BCR` |
| 响应格式 | R4（OCR + 功能数） | R3（OCR） | R3（OCR） |
| 超时机制 | 100 次 × 10ms = 1s | 100 次 × 10ms = 1s | `__mmc_poll_for_busy` 1s |
| 探测模式 | `ocr==0` 只发一次 | `ocr==0` 只发一次 | `ocr==0` 一直轮询到 busy |

**为什么顺序是固定的？** 因为 CMD1 对某些 SD 卡也会有响应。放入注释的原话：

> Order's important: probe SDIO, then SD, then MMC

如果先发 CMD1 得到正响应，内核会误判为 MMC 卡，后续发 eMMC 初始化序列（CMD0/CMD2/CMD3/CMD9/CMD7/CMD8/CMD6）给 SD 卡，可能导致状态紊乱。正确的顺序是利用协议的排他性递进：

```
CMD5 → 仅 SDIO 设备响应      → 排他性最强，优先
ACMD41 → 仅 SD 卡响应        → 排他性次之
CMD1 → SD 卡也可能错误响应    → 最不排他，兜底
```
#### 4.4.2.6 从 CMD 到硬件寄存器

所有命令最终通过 `host->ops->request()` 下发到控制器驱动：

```
mmc_wait_for_cmd(host, &cmd, retries)
  └─ mmc_wait_for_req(host, &mrq)
       └─ __mmc_start_req(host, mrq)
            └─ __mmc_start_request(host, mrq)
                 └─ host->ops->request(host, mrq)
                      └─ mmci_request(host, mrq)         = mmci.c
                           └─ mmci_start_command(host, cmd, ...)
                                └─ writel(···, base + MCI_ARGUMENT)  = 寄存器 0x08
                                └─ writel(···, base + MCI_COMMAND)   = 寄存器 0x0C
```

硬件完成命令后触发中断（`MCI_CMDRESPEND`），`mmci_irq()` 唤醒等待的线程。


### 4.4.3 eMMC 初始化入口 — `mmc_attach_mmc()`

**入口：** `drivers/mmc/core/mmc.c:2293`

这是探测流程的最后一步——前面已经通过 CMD0 让所有卡回到 Idle State，又通过 CMD1 确认了总线上有 MMC 卡。现在要正式初始化这张卡。

```c
int mmc_attach_mmc(struct mmc_host *host)
{
    int err;
    u32 ocr, rocr;

    WARN_ON(!host->claimed);

    /* ① 切开漏——CMD1 需要多卡线与仲裁 */
    if (!mmc_host_is_spi(host))
        mmc_set_bus_mode(host, MMC_BUSMODE_OPENDRAIN);

    /* ② CMD1 快速探测：arg=0 仅检查有无卡响应 */
    err = mmc_send_op_cond(host, 0, &ocr);
    if (err)
        return err;

    /* ③ 确认是 MMC → 挂上 mmc_bus_ops，标记总线类型 */
    mmc_attach_bus(host, &mmc_ops);
    if (host->ocr_avail_mmc)
        host->ocr_avail = host->ocr_avail_mmc;

    /* SPI 模式走不同路径读 OCR */
    if (mmc_host_is_spi(host)) {
        err = mmc_spi_read_ocr(host, 1, &ocr);
        if (err)
            goto err;
    }

    /* ④ 电压协商：主机 ∩ 卡的交集 */
    rocr = mmc_select_voltage(host, ocr);
    if (!rocr) {
        err = -EINVAL;
        goto err;
    }

    /* ⑤ 完整 CMD 序列初始化卡 */
    err = mmc_init_card(host, rocr, NULL);
    if (err)
        goto err;

    /* ⑥ 创建设备 → 触发 mmc_block probe */
    mmc_release_host(host);
    err = mmc_add_card(host->card);
    if (err)
        goto remove_card;

    mmc_claim_host(host);
    return 0;

remove_card:
    mmc_remove_card(host->card);
    mmc_claim_host(host);
    host->card = NULL;
err:
    mmc_detach_bus(host);
    pr_err("%s: error %d whilst initialising MMC card\n",
           mmc_hostname(host), err);
    return err;
}
```

**① 开漏切换 — MMC 协议规范要求**

这是 MMC 协议规范对识别阶段（CMD1、CMD2）的硬性要求：两条命令都必须工作在开漏模式。开漏模式下，总线上无论是一张卡还是多张卡，CMD 线的电气特性都是安全的——卡只拉低不拉高，线与仲裁不会造成短路；推挽模式下，如果多个设备同时驱动相反电平会导致电气冲突。

> **多卡仲裁是协议的历史遗产，实践中不存在。** MMC 协议在设计时考虑了多卡共享总线（CMD3 的 RCA 机制即是为此），但实际产品中一颗 SoC 的一个 SDMMC 控制器只接入一个设备——SD 卡槽、eMMC 或 SDIO 模块。STM32MP257 的 3 个 SDMMC 控制器各管一路就是典型做法。开漏模式之所以必须走，不是因为真的要仲裁多卡，而是协议兼容性的硬约束——就像 TCP 三次握手，即使你确定只有一个客户端，也必须做完 SYN → SYN-ACK → ACK。

回顾之前的时序：`mmc_power_up()` 设了推挽（`mmc_set_initial_state`），之后 `mmc_go_idle()` 保持推挽。现在 `mmc_attach_mmc()` 要把总线切回开漏，因为接下来 CMD1、CMD2 都是多卡仲裁命令。

这一步只在非 SPI 模式下执行。SPI 模式没有开漏概念。

**② CMD1 快速探测 — `mmc_send_op_cond(host, 0, &ocr)`**

调用 `MMC_SEND_OP_COND`（CMD1），参数 `ocr = 0`。这个值的意义是：

> ocr = 0 表示"不要协商电压，只探测是否存在 MMC 卡"

如果总线上没有 MMC 卡，CMD1 无响应，函数超时返回 `-ETIMEDOUT`。`mmc_attach_mmc` 是三步识别的最后一路，它失败后 `mmc_rescan_try_freq()` 返回错误，`mmc_rescan()` 用下一档频率（300KHz→200KHz→100KHz）重试整个流程。

如果有响应，`ocr` 返回卡支持的电压窗口（如 `bit 27~15` 表示支持电压范围），`bit 31` 为 BUSY 标志（卡是否初始化完毕）。

注意这只是快速探测，真正的 CMD1 全参数调用在 `mmc_init_card()` 内部。

##### 4.4.3.1 CMD1 轮询机制详解

`mmc_send_op_cond()` 内部通过 `__mmc_poll_for_busy()` 实现 CMD1 轮询：

```c
int mmc_send_op_cond(struct mmc_host *host, u32 ocr, u32 *rocr)
{
    struct mmc_command cmd = {};

    cmd.opcode = MMC_SEND_OP_COND;         // CMD1
    cmd.arg = ocr;
    cmd.flags = MMC_RSP_SPI_R1 | MMC_RSP_R3 | MMC_CMD_BCR;

    err = __mmc_poll_for_busy(host,
        MMC_OP_COND_PERIOD_US,             // 4ms
        MMC_OP_COND_TIMEOUT_MS,            // 1000ms
        &__mmc_send_op_cond_cb, &cb_data);
}
```

忙等循环（`mmc_ops.c:502`）：

```c
timeout = jiffies + msecs_to_jiffies(timeout_ms) + 1;
do {
    expired = time_after(jiffies, timeout);
    err = (*busy_cb)(cb_data, &busy);      // 发 CMD1 检查 busy
    if (err) return err;                   // 超时 = 没有 MMC 卡
    if (expired && busy) return -ETIMEDOUT;
    if (busy) {
        usleep_range(udelay, udelay * 2);  // 指数退避 32us→32ms
        if (udelay < udelay_max)
            udelay *= 2;
    }
} while (busy);
```

每轮回调的关键逻辑：

```c
static int __mmc_send_op_cond_cb(void *cb_data, bool *busy)
{
    err = mmc_wait_for_cmd(host, cmd, 0);

    if (cmd->resp[0] & MMC_CARD_BUSY) {    // bit 31
        *busy = false;                      // 卡就绪，停止轮询
        return 0;
    }
    *busy = true;                           // 卡还忙，继续轮询

    /* ★ 第一次 ocr=0 探测时，用卡的返回值设置后续参数 */
    if (!ocr && !mmc_host_is_spi(host))
        cmd->arg = cmd->resp[0] | BIT(30);  // 返回值 + HCS 位
    return 0;
}
```

轮询流程总结：

```
首次 CMD1(arg=0)         → 卡响应 OCR，BUSY=0（卡还在初始化）
用 resp[0] 设置下次 arg  → arg = OCR 返回值 | HCS(bit30)
再次 CMD1(arg=OCR|HCS)   → 卡继续初始化，重复轮询...
...
某次 CMD1                → BUSY=1（卡就绪）→ 退出循环
```

**状态机：** CMD1 是状态迁移的启动信号。当 BUSY=1 时，卡从 **Idle State** 进入 **Ready State**，标志着初始化已完成、可以接收 CMD2 进行识别。在此之前卡一直留在 Idle，重复响应 CMD1 的轮询。

---

**③ `mmc_attach_bus(host, &mmc_ops)` — 贴标签**

```c
void mmc_attach_bus(struct mmc_host *host, const struct mmc_bus_ops *ops)
{
    host->bus_ops = ops;    // 挂上 mmc_ops
}
```

本质上是给 `host` 打个"这是 MMC 总线"的标签。`mmc_ops` 提供了一组生命周期回调：

```c
static const struct mmc_bus_ops mmc_ops = {
    .remove = mmc_remove,
    .detect = mmc_detect,
    .suspend = mmc_suspend,
    .resume = mmc_resume,
    .runtime_suspend = mmc_runtime_suspend,
    .runtime_resume = mmc_runtime_resume,
    .alive = mmc_alive,
    .shutdown = mmc_shutdown,
    .hw_reset = _mmc_hw_reset,
    .cache_enabled = _mmc_cache_enabled,
    .flush_cache = _mmc_flush_cache,
};
```

每个回调的职责（从 `drivers/mmc/core/mmc.c` 源码整理）：

| 回调 | 触发时机 | 核心逻辑 |
|------|---------|---------|
| `remove` | 卡拔出/驱动卸载 | 调 `mmc_remove_card()` 释放卡设备，清 `host->card` |
| `detect` | host 层调度检测 | 发 CMD13 检查卡是否健在，不在了走 remove + detach_bus + power_off |
| `alive` | 检测前快速确认 | 发 CMD13（SEND_STATUS），有响应即认定存活 |
| `suspend` | 系统 suspend | 刷缓存 → power-off notify / sleep 通知卡 → 断电，标记 suspended |
| `resume` | 系统 resume | 重新使能 pm_runtime，卡状态恢复交给 runtime_resume 或块设备层 |
| `runtime_suspend` | 运行时 autosuspend | 仅在 `MMC_CAP_AGGRESSIVE_PM` 时真正挂起，否则空操作 |
| `runtime_resume` | 运行时 autoresume | 重新上电 + `mmc_init_card()` 恢复卡状态 |
| `shutdown` | 系统关机/重启 | 先保证卡醒着（poweroff notify 场景），再执行 long 模式 suspend + 断电 |
| `hw_reset` | 错误恢复/调谐失败 | 刷缓存 → 若支持 RST_n 引脚则拉复位，否则暴力上下电 → 重跑 `mmc_init_card` |
| `cache_enabled` | 查询接口 | 返回 `cache_size > 0 && cache_ctrl & 1` |
| `flush_cache` | 刷缓存 | CMD6 写 EXT_CSD FLUSH_CACHE，卡将内部缓存刷入 NAND |

**设计意图：** 这组回调把卡的整个生命周期（添加→检测→挂起→恢复→复位→移除）抽象为函数指针表。core 层在确认卡类型后挂上这个 ops，后续所有生命周期事件通过这里分发——不直接操作硬件，也不关心具体命令细节。

另外 `host->ocr_avail_mmc` 是一个可选覆写——如果平台定义了 MMC 专用的电压限制（例如某些板子对 eMMC 和 SD 卡使用不同的调节器），可以覆盖 `host->ocr_avail`。

**④ `mmc_select_voltage` — 主机侧电压定档**

```c
/* core.c:1115 */
u32 mmc_select_voltage(struct mmc_host *host, u32 ocr)
{
    /* ① 过滤无效低电压位 */
    if (ocr & 0x7F) {
        ocr &= ~0x7F;          // bit 0-6 对应 <1.7V，MMC 规范不支持
    }

    /* ② 主机 ∩ 卡：双方都支持的电压窗口 */
    ocr &= host->ocr_avail;
    if (!ocr) {
        dev_warn(mmc_dev(host), "no support for card's volts\n");
        return 0;               // 无交集 → 返回 0，调用方报错
    }

    /* ③ 定档：选一个 2-bit 窗口 */
    if (host->caps2 & MMC_CAP2_FULL_PWR_CYCLE) {
        bit = ffs(ocr) - 1;            // 取最低支持位
        ocr &= 3 << bit;
        mmc_power_cycle(host, ocr);    // 重新上下电
    } else {
        bit = fls(ocr) - 1;            // 取最高支持位
        ocr &= 3 << (bit - 1);         // 保留最高位及低一位
    }
    return ocr;
}
```

三步逻辑：

1. **过滤无效位（`ocr & 0x7F`）**：OCR 的 bit 0~6 代表 1.7V 以下的电压档位，MMC 规范不支持这么低的电压。如果卡宣称支持这些位（某些不规范卡），内核直接清零。

2. **取交集（`ocr &= host->ocr_avail`）**：`host->ocr_avail` 在 probe 阶段由 `mmc_regulator_get_supply()` 从 `vmmc-supply` regulator 的电压约束表解析得到。两叠取与，只剩双方都支持的电压位。结果为零则返回 `0`，`mmc_attach_mmc()` 走 `goto err` 路径打印 "error whilst initialising MMC card" 并 detach 总线。

3. **定档为 2-bit 窗口**：CMD1 协议要求主机向卡宣告一个连续位段作为工作电压。取交集后剩下若干连续位，以 3 个位为例：

   ```
   bit 22 (3.20~3.30V) = 1  ← 最低位
   bit 23 (3.30~3.40V) = 1
   bit 24 (3.40~3.50V) = 1  ← 最高位
   ```

   两个路径选哪一对：

   - **默认路径**：`fls` 取最高位 → bit = 24，`ocr &= 3 << (24-1) = bit22 | bit23` → 宣告 **3.20~3.40V**
   - **`MMC_CAP2_FULL_PWR_CYCLE`**：`ffs` 取最低位 → bit = 22，`ocr &= 3 << 22 = bit22 | bit23` → 同样宣告 **3.20~3.40V**，但先 `mmc_power_cycle` 再继续

   两者的实质差异只在**是否做完整上下电**。选哪一对取决于取最高还是最低位，但最终结果在连续 3 位时恰巧一样。2-bit 宽度是因为 CMD1 要求连续位段，单一位的电压范围太窄（如 3.30~3.40V），难以保证 regulator 的负载波动余量。

返回的 `rocr` 不参与 regulator 控制。电压设置分两阶段：

- **物理上电（`mmc_power_up`）**：`host->ios.vdd = fls(host->ocr_avail) - 1`，调 `mmc_regulator_set_ocr(vmmc)` **真实写入 regulator 硬件**，输出 3.3V 给卡供电。
- **协议握手（`mmc_select_voltage` → CMD1）**：只做位运算，结果 `rocr` 作为 CMD1 参数。CMD1 的核心是**轮询卡就绪**——主机反复发 CMD1，卡在内部初始化完成前响应 BUSY=0，完成后 BUSY=1。`rocr` 中的电压信息是主机告诉卡"我用这个电压范围"。卡的逻辑：
  - 电压匹配 → 正常应答 CMD1，BUSY=0/1 表示初始化进度
  - 电压不匹配 → **卡不应答**，主机等到超时返回 `-ETIMEDOUT`，`mmc_attach_mmc` 报错退出

**⑤ `mmc_init_card()` — 正式 CMD 序列**

这是 eMMC 初始化的重头戏。内部会执行完整的 CMD 状态机迁移：

```
CMD0  → Idle State（第二次复位，用开漏模式）
CMD1  → Idle → Ready（带电压参数 + 轮询 BUSY）
CMD2  → Ready → Ident（全卡仲裁发 CID）
CMD3  → Ident → Standby（分配 RCA）
CMD9  → Standby（读 CSD 获取卡属性）
CMD7  → Standby → Transfer（选中卡，进入数据传输态）
CMD8  → Transfer（读 ext_csd 获取完整能力）
CMD6  → Transfer（时序协商：HS→HS200→HS400 阶梯上升）
```

详细分析见下节 4.4.4。

**⑥ `mmc_add_card()` — 注册到设备模型，触发块设备创建**

```c
mmc_release_host(host);
err = mmc_add_card(host->card);
```

`mmc_add_card()`（`bus.c:300`）做三件事：

1. **命名**：`dev_set_name(&card->dev, "%s:%04x", mmc_hostname(host), card->rca)` → 设备名如 `mmc1:0001`
2. **打 log**：`pr_info("mmc1: new high speed DDR MMC card at address 0001\n")` → 这是内核启动时你看到的那行 MMC 消息
3. **注册设备**：`device_add(&card->dev)` → 把 card 挂到 MMC 总线上，触发驱动匹配

第 3 步是关键。`device_add` 触发内核设备模型遍历 `mmc_bus_type` 上的驱动，找到 `mmc_block` 驱动后调其 probe 函数 `mmc_blk_probe()`，后者创建 gendisk、调 `device_add_disk`，最终在 `/dev/` 下出现 `mmcblk1`。

为什么先 `mmc_release_host`？因为 `device_add` → `mmc_blk_probe` → `device_add_disk` 会触发**分区表读取**。分区读走块设备层 submit_bio，最终由 `mmc_blk_mq_issue_rq` 处理——这个函数跑在 **blk-mq 的线程上下文**（`kblockd`），和跑 `mmc_rescan` 的 kworker **不是同一个线程**。

如果不提前释放，两个线程争 host 锁：

```
kworker（持锁）→ mmc_add_card → device_add_disk → 等分区读完才返回
kblockd（等锁）→ 分区读 → MMC 请求 → mmc_claim_host → schedule() 睡眠
```

kworker 等 kblockd 读完分区才能返回，kblockd 等 kworker 释放锁才能发命令——互相死等。所以 `mmc_release_host` 在 `mmc_add_card` 前先让出锁，让 kblockd 能拿锁完成分区读，读完后 kworker 再通过 `mmc_claim_host` 拿回来。

### 4.4.4 CMD 序列逐条分析 — `mmc_init_card()`

**入口：** `drivers/mmc/core/mmc.c:1600`

这是 eMMC 初始化最核心的函数。先看整体骨架，再逐条分析：

```c
static int mmc_init_card(struct mmc_host *host, u32 ocr,
                         struct mmc_card *oldcard)
{
    /* ① 开漏 + CMD0 复位 */
    mmc_set_bus_mode(host, MMC_BUSMODE_OPENDRAIN);
    mmc_go_idle(host);

    /* ② CMD1 电压协商 + 轮询 BUSY */
    err = mmc_send_op_cond(host, ocr | BIT(30), &rocr);

    /* ③ CMD2 读 CID → 分配 RCA → 切推挽 */
    err = mmc_send_cid(host, cid);
    err = mmc_set_relative_addr(card);     // CMD3
    mmc_set_bus_mode(host, MMC_BUSMODE_PUSHPULL);

    /* ④ CMD9 读 CSD + CMD7 选卡 */
    err = mmc_send_csd(card, card->raw_csd);
    err = mmc_select_card(card);

    /* ⑤ CMD8 读 EXT_CSD → 获取卡全部能力 */
    err = mmc_read_ext_csd(card);

    /* ⑥ CMD6 多次 SWITCH（时序/宽度/缓存/CMDQ） */
    err = mmc_select_timing(card);         // 时序协商阶梯
    mmc_select_bus_width(card);           // 总线宽度
    mmc_switch(card, ..., EXT_CSD_CACHE_CTRL, 1); // 使能缓存
    mmc_cmdq_enable(card);                // 使能 CMDQ
}
```

下面按命令逐条分析，每条标注**状态机**的状态变迁。

**第 1 步：CMD0 — GO_IDLE_STATE**

```c
/* line 1611-1612: 设置开漏总线模式 */
if (!mmc_host_is_spi(host))
    mmc_set_bus_mode(host, MMC_BUSMODE_OPENDRAIN);

/* line 1621: 发送 CMD0 复位卡 */
mmc_go_idle(host);
```

内核注释（`mmc.c:1614-1620`）解释了为什么在这还要发一次 CMD0：

> Since we're changing the OCR value, we seem to need to tell some cards to go back to the idle state. mmc_go_idle is needed for eMMC that are asleep.

两个原因：
- **更换 OCR**：之前 `mmc_attach_mmc` 中的 CMD1 只是快速探测（arg=0），现在 `mmc_init_card` 要发完整的 CMD1（带 `ocr | HCS`）。某些卡在收到新的 OCR 值之前需要先复位到 Idle State 才能重新协商。
- **唤醒休眠的 eMMC**：如果卡之前进入了 Sleep State（CMD5），CMD0 是唯一能把它唤醒到 Idle 的命令。

**为什么这里设开漏？** 不是给 CMD0 用的（CMD0 是广播命令，无响应，总线模式无所谓），而是为接下来的 CMD1/CMD2 做准备——这两条命令需要开漏模式做多卡仲裁。而切 pinctrl 的开漏状态必须在发任何命令之前完成。

**与 4.4.2.4 节第一次 CMD0 的区别：**

| 位置 | 总线模式 | 目的 |
|------|---------|------|
| `mmc_rescan_try_freq` 中的 CMD0（4.4.2.4） | **推挽** | 探测前复位总线上所有卡 |
| `mmc_init_card` 中的 CMD0（此处） | **开漏** | 更换 OCR 前复位 + 唤醒睡眠卡 |

```
暗线状态机: 任意状态 → CMD0 → Idle State
```

---

**第 2 步：CMD1 — SEND_OP_COND**

```c
/* line 1624: CMD1，带电压参数 + HCS 位 */
err = mmc_send_op_cond(host, ocr | (1 << 30), &rocr);
```

参数 `ocr` 是前一步 `mmc_select_voltage` 的返回值（host ∩ card 的 2-bit 电压窗口）。`BIT(30)` 是 HCS（High Capacity Support），告诉卡主机支持 >2GB 的块寻址。

内部实现（`mmc_ops.c:223`）——`mmc_send_op_cond` 通过 `__mmc_poll_for_busy` 做轮询，每轮回调 `__mmc_send_op_cond_cb`：

```c
static int __mmc_send_op_cond_cb(void *cb_data, bool *busy)
{
    err = mmc_wait_for_cmd(host, cmd, 0);   // 发 CMD1

    if (cmd->resp[0] & MMC_CARD_BUSY) {      // bit 31
        *busy = false;                        // 卡就绪，停止轮询
        return 0;
    }
    *busy = true;                             // 卡还忙，继续轮询
    return 0;
}
```

轮询过程：

```
时间 → 主机发 CMD1(arg=ocr|HCS) → 卡响应 OCR，BUSY=0（还在初始化）
          usleep(指数退避 32us→32ms)
       主机发 CMD1(arg=ocr|HCS) → 卡响应 OCR，BUSY=0
          usleep(...)
       主机发 CMD1(arg=ocr|HCS) → 卡响应 OCR，BUSY=1（就绪了！）
          → 退出轮询
```

- **卡就绪前**：BUSY=0，卡还在做内部初始化（加载固件、设置默认状态），每次 CMD1 返回后主机等一段时间再试
- **卡就绪后**：BUSY=1，卡从 **Idle → Ready State**，可以接收 CMD2 了
- **超时**：`MMC_OP_COND_TIMEOUT_MS = 1000ms`，1 秒内 BUSY 仍为 0 则返回 `-ETIMEDOUT`

对比 4.4.3 节中 `mmc_attach_mmc` 的第一步 CMD1（arg=0 只探测有无卡），此处是有完整电压参数的 CMD1，真正完成电压协商 + 等待卡就绪。

```
暗线状态机: Idle State → CMD1 → Ready State
```

---

**第 3 步：CMD2 — ALL_SEND_CID**

```c
/* line 1640: 获取 CID */
err = mmc_send_cid(host, cid);
```

这是整个初始化过程中最特殊的一条命令——它与控制器寄存器无关，纯粹利用了开漏总线的电气特性做"硬件级仲裁"。

先看内核源码是怎么发的：

```c
// mmc_ops.c, line 362
int mmc_send_cid(struct mmc_host *host, u32 *cid)
{
    int ret;
    /* MMC_ALL_SEND_CID (CMD2), arg=0, R2（136-bit 长响应） */
    ret = mmc_send_cxd_native(host, 0, cid, MMC_ALL_SEND_CID);

    return ret;
}
```

```c
// mmc_ops.c, line 261
static int mmc_send_cxd_native(struct mmc_host *host, u32 arg,
                                u32 *cxd, int opcode)
{
    int err;
    struct mmc_command cmd = {0};

    cmd.opcode = opcode;     // MMC_ALL_SEND_CID (CMD2)
    cmd.arg = arg;           // 0
    cmd.flags = MMC_RSP_R2 | MMC_CMD_AC;  // R2 = 136-bit, 应用类命令

    err = mmc_wait_for_cmd(host, &cmd, 0);  // 同步发送
    if (err)
        return err;

    memcpy(cxd, cmd.resp, sizeof(cmd.resp));  // 4 × 32-bit CID
    return 0;
}
```

关键参数：
- **`arg = 0`**：CMD2 是广播命令，无地址参数，所有处于 Ready State 的卡都会参与。
- **`MMC_RSP_R2`**：136-bit 长响应（128-bit CID + 8-bit CRC），全部走 **CMD 线**（非 DAT 线），占用 136 个 CMD 线时钟周期。初始化阶段总线宽度为 1-bit（DAT0 只有一根线），但响应走 CMD 线不受 DAT 宽度影响——代价只是频率低时多等几个周期而已。
- **`MMC_CMD_AC`**：应用类命令（Application-specific Command），不是控制类命令（BC/BCR）。这一步验证了协议细节——CMD2 虽然是广播性质的，但会因为多条卡同时响应而产生数据冲突，所以被归为 AC 类，表示每张卡可以独立响应。

**逐位仲裁（Bitwise Arbitration）过程：**

CMD2 的时序与普通命令不同——普通命令是"主机发指令 → 指定卡返回响应"，但 CMD2 是"主机发指令 → 所有卡同时返回 CID"：

```
主机发送 CMD2 命令头 ───→ 所有卡同时驱动 DAT 线 ──→ 线与（低覆盖高）

                         卡 A CID:  1 0 1 0 0 1 1 0 ...
                         卡 B CID:  1 0 0 1 1 1 0 0 ...
                         ──────────────────────────────
                         DAT 线:    1 0 0 0 0 1 0 0 ...
                         仲裁解析:  前 2 位相同（11），
                                    第 3 位卡 A 拉低→卡 B 发现冲突→退让
```

仲裁规则：
1. 每张卡同时从 MSB 开始逐位驱动 CID 到 DAT 线。
2. **开漏模式下**，"低覆盖高"——如果一张卡驱动 0、另一张驱动 1，DAT 线电平 = 0。
3. 每驱动一位，每张卡都"回读"DAT 线实际电平。
4. 如果某位卡驱动的是 1 但读到的是 0（发现被其他卡拉低了），说明有卡在这一位竞争，**该卡立即停止驱动并退出**。
5. 唯一没有退出的卡 = 最大 CID 的卡 → **赢得仲裁**。

**仲裁结果：两分叉的状态迁移**

```
                ┌────────────────────┐
                │   Ready State      │
                │   (多卡等待仲裁)     │
                └────────┬───────────┘
                         │ CMD2 发出
                         ▼
               ┌─────────────────────┐
               │   逐位仲裁开始        │
               │   所有卡逐位驱动 CID  │
               └────────┬─────────────┘
                        │
              ┌─────────┴──────────┐
              ▼                    ▼
   ┌──────────────────┐  ┌──────────────────┐
   │   赢家（Winner）  │  │ 输家（Loser）    │
   │   → Ident State  │  │ → 留在 Ready     │
   │   完整发送 CID    │  │    State         │
   │   等待 CMD3 分配  │  │   等待下一轮     │
   │   RCA            │  │   CMD2 再次仲裁  │
   └──────────────────┘  └──────────────────┘
```

**内核态如何处理仲裁输赢？**

内核层面**完全不处理**。`mmc_send_cid()` 只是发送 CMD2 并回收响应——它不知道也不关心总线上有几张卡、谁赢了谁输了。原因很简单：

- **实际场景只有一张 eMMC 卡**：eMMC 是固定焊接的，没有热插拔，不存在多卡共享总线。CMD2 的仲裁逻辑对所有卡都一样——总线上只有一张卡时，它自然就是赢家。
- **多仲裁处理是协议的遗留能力**：MMC 协议设计之初考虑了多卡环境（如数码相机插多张卡），但 eMMC 场景下从未用到。内核的 `mmc_init_card()` 整个流程都假设只有一张卡，不需要循环调用 CMD2。

**CMD2 → CMD3 之间做的事**

先看源码（首次探测路径，`oldcard = NULL`）：

```c
/* line 1640: CMD2 — 获取 CID */
err = mmc_send_cid(host, cid);

/* =========== 分配 card 结构体 =========== */
card = mmc_alloc_card(host, &mmc_type);            // (1)
if (IS_ERR(card)) {
    err = PTR_ERR(card);
    goto err;
}
card->ocr = ocr;                                   // (2)
card->type = MMC_TYPE_MMC;                         // (3)
card->rca = 1;                                     // (4)
memcpy(card->raw_cid, cid, sizeof(card->raw_cid)); // (5)

/* =========== 控制器 quirk 回调 =========== */
if (host->ops->init_card)
    host->ops->init_card(host, card);              // (6)

/* =========== 发 CMD3 分配 RCA =========== */
err = mmc_set_relative_addr(card);                 // 见第 4 步展开
mmc_set_bus_mode(host, MMC_BUSMODE_PUSHPULL);     // 见第 4 步展开
```

逐条解释：

**(1) `mmc_alloc_card()` — 分配 card 对象**

```c
/* bus.c:277 */
card = kzalloc(sizeof(struct mmc_card), GFP_KERNEL);  // 分配内存
card->host = host;
device_initialize(&card->dev);                         // 初始化 device 模型
card->dev.parent = mmc_classdev(host);                // parent = mmc host class 设备
card->dev.bus = &mmc_bus_type;                        // bus = mmc_bus_type
card->dev.release = mmc_release_card;
card->dev.type = type;                                 // type = &mmc_type
```

`kzalloc` 分配 `struct mmc_card`（包含 CID/CSD/ext_csd 等所有字段的缓冲区），`device_initialize` 初始化内核设备模型的内部数据结构（kobject、锁、引用计数）。注意这**只是创建对象，还没注册到系统**——注册是后面 `mmc_add_card()` 做的。

**(2) `card->ocr = ocr`**

保存 CMD1 协商的 OCR 值。后续电源管理操作需要它——比如休眠恢复后重新上电要知道用多少电压。

**(3) `card->type = MMC_TYPE_MMC`**

标记卡类型。`mmc_init_card()` 虽然放在 `mmc.c`（MMC 专属），但被调用的路径不止一条——显式赋值确保类型正确。

**(4) `card->rca = 1`**

预设 RCA 为 1。MMC 协议中主机通过 CMD3 的 arg[31:16] 指定 RCA。eMMC 只有一张卡，内核固定用 1。SD 协议不同——卡自分配 RCA 并在响应中返回（详见第 4 步）。

**(5) `memcpy(card->raw_cid, cid, sizeof(card->raw_cid))`**

把 CMD2 返回的 4 × 32-bit CID 保存到 card 结构中。后面 `mmc_decode_cid()` 会把它解析成可读的厂商/产品名/序列号，sysfs 节点 `/sys/block/mmcblk1/device/cid` 也展示原始值。

**(6) `init_card` 回调 — 控制器 quirk 介入点**

```c
if (host->ops->init_card)
    host->ops->init_card(host, card);
```

`mmc_host_ops` 的可选回调。时机在"卡对象刚创建完、还没进入高带宽配置之前"，给控制器驱动一个早期介入机会。实际用法举例：

| 控制器 | 实现 | 作用 |
|--------|------|------|
| `sdhci-esdhc-imx` | `usdhc_init_card` | 保存 `card->type`，后续 tuning 根据卡类型做不同校准 |
| `mxcmmc` | `mxcmci_init_card` | 检测 MX3 SoC CRC bug + SDIO 卡 → 限制为 1-bit |
| `sdhci-xenon` | `xenon_init_card` | 保存 `card->type` 供 PHY 时序调整 |

**CID 数据结构**

CMD2 返回的 CID 由 4 个 32 位字组成（R2 响应），包含卡的身份信息：

| 字段 | 缩写 | 位数 | 说明 |
|------|------|------|------|
| Manufacturer ID | MID | 8 | 厂商代码（Kingston = 0x70） |
| OEM/Application ID | OID | 2 | OEM 标识 |
| Product Name | PNM | 40 | 产品名称（ASCII） |
| Product Revision | PRV | 8 | 产品版本（BCD 编码） |
| Product Serial Number | PSN | 32 | 序列号 |
| Manufacturing Date | MDT | 8 | 生产日期（年/月编码） |
| CRC | — | 7 | 校验 |
| Reserved | — | 2 | 保留位 |

> **协议 vs 实际：** CMD2 在协议层面是完备的多卡仲裁机制，但 eMMC 场景下它等价于"读卡 ID 的专用命令"。仲裁逻辑完全由卡硬件自动完成，内核驱动只是坐等赢家返回 CID。

```
暗线状态机: Ready State → CMD2 → Ident State
          (赢家进 Ident，输家留在 Ready)
```

**第 4 步：CMD3 — SET_RELATIVE_ADDR + 切换推挽模式**

```c
/* line 1679: 分配 RCA */
err = mmc_set_relative_addr(card);
/* card->rca = 1 */

/* line 1683: 立即切换为推挽模式 */
mmc_set_bus_mode(host, MMC_BUSMODE_PUSHPULL);
```

看实现：

```c
/* mmc_ops.c:249 */
int mmc_set_relative_addr(struct mmc_card *card)
{
    struct mmc_command cmd = {};

    cmd.opcode = MMC_SET_RELATIVE_ADDR;        // CMD3
    cmd.arg = card->rca << 16;                 // 1 << 16 = 0x10000
    cmd.flags = MMC_RSP_R1 | MMC_CMD_AC;

    return mmc_wait_for_cmd(card->host, &cmd, MMC_CMD_RETRIES);
}
```

参数细节：

- **`arg = card->rca << 16`**：MMC 协议规定 CMD3 的 arg[31:16] 存放 RCA，arg[15:0] 保留。`card->rca = 1` → arg = `0x10000`。RCA 是一个 16-bit 的短地址，后续命令（CMD7/CMD9/CMD13 等）都通过 arg 携带 RCA 来指定目标卡。
- **`MMC_RSP_R1`**：48-bit 短响应（仅含卡状态 bits）。CMD3 只是写一个寄存器，不需要长回复。
- **`MMC_CMD_AC`**：应用类命令（Application-specific），支持独立响应。
- **`MMC_CMD_RETRIES = 3`**：失败最多重试 3 次。重试由 `mmc_wait_for_req_done()` 的循环实现：

```c
/* core.c:401 */
while (1) {
    wait_for_completion(&mrq->completion);
    cmd = mrq->cmd;

    if (!cmd->error || !cmd->retries || mmc_card_removed(host->card))
        break;

    cmd->retries--;
    cmd->error = 0;
    __mmc_start_request(host, mrq);         // 不换参数重新发
}
```

每次重试发送完全相同的 CMD3（arg 不变），直到成功或耗尽重试次数。

**MMC vs SD 的 CMD3 差异：**

| | **MMC（eMMC）** | **SD** |
|---|---|---|
| 命令名 | `MMC_SET_RELATIVE_ADDR` | `SD_SEND_RELATIVE_ADDR` |
| arg | `card->rca << 16`（host 指定） | `0`（卡自分配） |
| 响应 | R1（仅状态，不含 RCA） | R6（包含卡自分配的 RCA） |
| RCA 来源 | 内核 `card->rca = 1` | 从 cmd.resp[0] >> 16 提取 |
| 命令类 | AC（应用类） | BCR（广播 + 响应） |

SD 的实现——卡自分配 RCA，主机从响应中读取：

```c
/* sd_ops.c:238 */
int mmc_send_relative_addr(struct mmc_host *host, unsigned int *rca)
{
    cmd.opcode = SD_SEND_RELATIVE_ADDR;
    cmd.arg = 0;
    cmd.flags = MMC_RSP_R6 | MMC_CMD_BCR;

    err = mmc_wait_for_cmd(host, &cmd, MMC_CMD_RETRIES);

    *rca = cmd.resp[0] >> 16;               // 卡在响应里返回 RCA
    return err;
}
```

两种设计出自不同假设：MMC 协议考虑多卡场景，host 分配地址集中管理；SD 面向单卡消费电子，卡自己生成更简单。eMMC 继承 MMC 的 host 分配方式，单卡场景下简化为固定值 1。

**状态迁移：Ident State → Standby State**

CMD3 成功后卡进入 Standby State。Standby 意味着：
- 卡已拥有 RCA，可以接收以该 RCA 为目标的命令（CMD9 读 CSD、CMD13 查状态、CMD7 选卡）
- 卡**未选中**（Not Selected），不参与数据传输
- 只有收到 CMD7 带自身 RCA 后，才会进入 Transfer State 准备读写

**CMD3 之后立即切换推挽模式的协议根源：**

```
CMD3 之前的场景：多卡仲裁 → 必须开漏（防止总线冲突）
CMD3 之后：     单卡已寻址   → 立即切推挽，为高速传输准备
```

在 STM32MP2 上，切换通过 **pinctrl** 实现：

```c
/* mmci_set_ios() 内部 */
if (ios->bus_mode == MMC_BUSMODE_OPENDRAIN)
    pinctrl_select_state(host->pins_opendrain);
else
    pinctrl_select_state(host->pins_default);
```

```
状态机:  Ident State → CMD3 → Standby State
         Open-Drain  →       → Push-Pull
```

---

**第 5 步：CMD9 — SEND_CSD（读取卡配置信息）**

```c
/* line 1690-1696: 获取并解码 CSD */
err = mmc_send_csd(card, card->raw_csd);  // CMD9
err = mmc_decode_csd(card);               // 解码 CSD 寄存器
err = mmc_decode_cid(card);               // 解码 CID 寄存器内容
```

这一步的目的是**读取卡的 CSD 寄存器**。CSD（Card Specific Data）是 MMC 协议定义的最基础的配置寄存器——卡支持什么 MMC 版本、基础时钟多快、块大小多少、擦除粒度多大，都在 CSD 里。这些信息是驱动后续决策的输入：比如知道 `mmca_vsn` 后才能决定怎么解析 CID，知道 `max_dtr` 后才能设定初始时钟。

注意 CSD 对 eMMC 5.x 来说已经"不够用了"。现代卡的高速能力（HS200/HS400、Cache、CMDQ）都存放在 EXT_CSD 中（后面 CMD8 才读），CSD 只提供最基本的兼容性信息。但它是协议兼容的根基——所有 MMC/eMMC 卡都必须支持 CMD9。

另外这是 CMD2 之后第一条**带 RCA 寻址**的命令。CMD2 是广播（arg=0），CMD9 用 `card->rca << 16` 指定具体卡——标志着总线从多卡模式切入了单卡寻址。

**① `mmc_send_csd()` — 发 CMD9 读裸数据**

```c
/* mmc_ops.c:353 */
int mmc_send_csd(struct mmc_card *card, u32 *csd)
{
    return mmc_send_cxd_native(card->host,
                card->rca << 16, csd, MMC_SEND_CSD);
}
```

与 CMD2 共用 `mmc_send_cxd_native`，差异对比：

| | CMD2（ALL_SEND_CID） | CMD9（SEND_CSD） |
|---|---|---|
| arg | 0（广播） | `card->rca << 16`（寻址） |
| 响应 | CID（你是谁） | CSD（你有什么能力） |
| 参与卡 | 所有 Ready 的卡 | 只有指定 RCA 的卡 |

**② `mmc_decode_csd()` — 解析 CSD 获取版本和能力**

`mmc_decode_csd` 用 `UNSTUFF_BITS` 宏从裸位中提取字段（`mmc.c:142`），几个关键项：

```c
csd->mmca_vsn  = UNSTUFF_BITS(resp, 122, 4);  // ← 最重要：MMC 协议版本
csd->max_dtr   = tran_exp[e] * tran_mant[m];   // 基础最大时钟（kHz）
csd->cmdclass  = UNSTUFF_BITS(resp, 84, 12);   // 支持的命令类别
csd->read_blkbits = UNSTUFF_BITS(resp, 80, 4); // 读块大小（2^n）
```

`mmca_vsn` 是最关键的字段——多个后续操作依赖它：

| mmca_vsn | MMC 版本 | 被谁用 |
|----------|---------|--------|
| 0-1 | v1.x | CID 解析（v1 格式） |
| 2-3 | v2.x-v3.x | CID 解析（v2+ 格式） |
| **4** | **v4.x** | CID 解析 + 决定是否读 EXT_CSD |
| 5 | v4.41 | 同上 |
| **6** | **v5.x** | 同上（ATK 板 Kingston eMMC = 6） |

`mmca_vsn >= 4` 的含义是"卡支持 EXT_CSD"，后续的第 7 步（CMD8）才会执行。如果 `mmca_vsn < 4`（理论上早期 MMC 卡），内核跳过 EXT_CSD 读取，仅靠 CSD 的基础能力运行。

**③ `mmc_decode_cid()` — 解析 CID（依赖 mmca_vsn）**

CID 虽然在 CMD2 阶段就已获取并存到 `card->raw_cid`，但一直没被解析——因为不知道用 v1 还是 v2+ 格式来解释。直到 CSD 解析出 `mmca_vsn` 后，`mmc_decode_cid` 才能做位布局选择：

```c
switch (card->csd.mmca_vsn) {
case 0: case 1:
    /* v1 格式：24-bit MID, 6-byte prod_name */
    card->cid.manfid = UNSTUFF_BITS(resp, 104, 24);
    break;
case 2: case 3: case 4: case 5: case 6:
    /* v2+ 格式：8-bit MID, 16-bit OEMID, 32-bit serial */
    card->cid.manfid = UNSTUFF_BITS(resp, 120, 8);
    card->cid.oemid  = UNSTUFF_BITS(resp, 104, 16);
    break;
}
```

这就是 `mmc_decode_csd` 必须在 `mmc_decode_cid` 之前调用的原因——先读版本，再决定 CID 解析格式。

```
暗线状态机: Standby State → CMD9 → 仍在 Standby State
          CMD9 只读不写，不改变卡状态
```

---

**第 6 步：CMD7 — SELECT/DESELECT_CARD（选卡，进入 Transfer State）**

```c
/* line 1712-1716: 选卡 */
if (!mmc_host_is_spi(host)) {
    err = mmc_select_card(card);     // CMD7
}
```

这一步的目的就一个：**把卡从 Standby 切到 Transfer State**。只有 Transfer State 才能发数据类命令——接下来第 7 步的 CMD8 要用块读取传 512 字节 EXT_CSD，卡必须处于 Transfer State。

看实现：

```c
/* mmc_ops.c:99 */
static int _mmc_select_card(struct mmc_host *host, struct mmc_card *card)
{
    struct mmc_command cmd = {};

    cmd.opcode = MMC_SELECT_CARD;                 // CMD7

    if (card) {
        cmd.arg = card->rca << 16;                // 选中：arg = RCA
        cmd.flags = MMC_RSP_R1 | MMC_CMD_AC;      // 期望 R1 响应
    } else {
        cmd.arg = 0;                              // 取消选中：arg = 0
        cmd.flags = MMC_RSP_NONE | MMC_CMD_AC;    // 无响应
    }

    return mmc_wait_for_cmd(host, &cmd, MMC_CMD_RETRIES);
}
```

两个关键细节：

**① 选卡 vs 取消选卡是同一个命令**

| 操作 | arg | 响应 | 效果 |
|------|-----|------|------|
| `mmc_select_card(card)` | `card->rca << 16` | R1（卡返回状态） | Standby → **Transfer** |
| `mmc_deselect_cards(host)` | 0 | 无响应 | Transfer → **Standby** |

`arg=0` 时协议层含义是"不指定任何卡"，卡回到 Standby State。内核在休眠和硬件复位路径中调 `mmc_deselect_cards`：

```c
/* mmc.c:1972 — SLEEP_AWAKE 前先取消选卡 */
err = mmc_deselect_cards(host);
```

**② 为什么不在 CMD3 后立即选卡？**

从 CMD3（Standby）到 CMD7（Transfer）之间插了一个 CMD9（读 CSD）——因为 CMD9 在 Standby State 也可以执行，不需要 Transfer State。CMD9 是信息命令（只读不写），协议允许它在 Standby 下执行。而 CMD8（EXT_CSD）是数据命令，必须 Transfer State 才能发。所以 CMD7 被刻意推迟到了 CMD9 之后、CMD8 之前，最小化状态切换次数。

```
暗线状态机: Standby State → CMD7 → Transfer State
```

---

**第 7 步：CMD8 — SEND_EXT_CSD（读取全部高级能力）**

```c
/* line 1719-1721: 读取 EXT_CSD */
err = mmc_read_ext_csd(card);
```

这一步是整个初始化流程中信息量最大的一步。CSD 只给基本兼容性信息（版本、基础时钟），而 **EXT_CSD 包含了 eMMC 的所有高级能力**——从分区布局到高速时序模式到命令队列。内核要靠它决定后面怎么做时序协商和总线配置。

**进入前的检查：**

```c
/* mmc.c:676 */
static int mmc_read_ext_csd(struct mmc_card *card)
{
    /* 只有 mmca_vsn > 3（MMC v4+）的卡才有 EXT_CSD */
    if (!mmc_can_ext_csd(card))
        return 0;

    err = mmc_get_ext_csd(card, &ext_csd);       // 发 CMD8 读 512 字节
    ...
    err = mmc_decode_ext_csd(card, ext_csd);     // 解码
    kfree(ext_csd);                               // 解码后释放原始数据
    return err;
}
```

**① `mmc_get_ext_csd()` — 读裸数据（块传输）**

```c
/* mmc_ops.c:370 */
ext_csd = kzalloc(512, GFP_KERNEL);              // 分配 512 字节缓冲区

err = mmc_send_adtc_data(card, card->host,
            MMC_SEND_EXT_CSD, 0, ext_csd, 512);  // ← 真正的 CMD8
```

内核注释说得很直白：*As the ext_csd is so large and mostly unused, we don't store the raw block in mmc_card.*（EXT_CSD 很大且大部分字段用不上，不保留原始块）。所以 `kzalloc` 临时分配、传给解码器、解码完马上释放。

`mmc_send_adtc_data` 设置了 ADTC（Address Data Transfer Command）类单块读取：

```c
cmd.opcode = MMC_SEND_EXT_CSD;    // CMD8
cmd.flags = MMC_RSP_SPI_R1 | MMC_RSP_R1 | MMC_CMD_ADTC;

data.blksz = 512;                  // 块大小
data.blocks = 1;                   // 单块读取
data.flags = MMC_DATA_READ;        // 读方向
```

ADTC 是协议中对"带数据传输命令"的分类。CMD8（EXT_CSD 读取）虽然名字叫 SEND_EXT_CSD，实际用了块传输机制——和 CMD17（READ_SINGLE_BLOCK）在传输层完全一致，区别只在于读取的目标寄存器不同。

**② `mmc_decode_ext_csd()` — 大量字段提取**

这是一个约 200 行的庞大解析函数，逐字节从 512 字节数组中提取信息。下面是它对内核至关重要的几个输出：

| 字段 | EXT_CSD 偏移 | 作用 |
|------|-------------|------|
| `raw_card_type` | 196 | 卡支持的时序模式（HS26/HS52/DDR/HS200/HS400），决定了第 8 步的协商阶梯能走到哪一级 |
| `rev` | 192 | EXT_CSD 版本号，决定后续字段是否存在（rev >= 4 才有分区管理） |
| `sectors` | 212-215 | 用户数据区总扇区数（4 字节），>2GB 标记块寻址 |
| `raw_boot_mult` | 226 | Boot 分区大小（×128KB），ATK 板 4MB |
| `rpmb_size_mult` | 168 | RPMB 分区大小（×128KB），ATK 板 4MB |
| `raw_partition_support` | 160 | 分区支持能力（boot/ RPMB/ GP 分区） |
| `cache_size` | 249(2B) 或 249-252(4B) | 缓存大小（byte），为后续缓存使能提供依据 |
| `cmdq_support` | 308 | CMDQ 队列深度掩码，决定是否使能命令队列 |
| `generic_cmd6_time` | 148 | CMD6 SWITCH 操作超时（10ms 单位），内核用它设置 CMD6 等待时间 |
| `strobe_support` | 184 | HS400 增强选通（Enhanced Strobe）支持 |
| `ffu_capable` | 261 | 固件升级能力 |

**③ `mmc_select_card_type()` — 可用时序筛选**

`mmc_decode_ext_csd` 内部调用了 `mmc_select_card_type`（`mmc.c:194`），这是为下一步时序协商做的预处理——用 host 能力和卡能力做与运算，得到双方的可用交集：

```c
static void mmc_select_card_type(struct mmc_card *card)
{
    u8 card_type = card->ext_csd.raw_card_type;
    u32 caps = host->caps, caps2 = host->caps2;
    unsigned int avail_type = 0;

    if (caps & MMC_CAP_MMC_HIGHSPEED &&
        card_type & EXT_CSD_CARD_TYPE_HS_26)
        avail_type |= EXT_CSD_CARD_TYPE_HS_26;     // HS 26MHz

    if (caps2 & MMC_CAP2_HS200_1_8V_SDR &&
        card_type & EXT_CSD_CARD_TYPE_HS200_1_8V)
        avail_type |= EXT_CSD_CARD_TYPE_HS200_1_8V; // HS200 200MHz

    if (caps2 & MMC_CAP2_HS400_1_8V &&
        card_type & EXT_CSD_CARD_TYPE_HS400_1_8V)
        avail_type |= EXT_CSD_CARD_TYPE_HS400_1_8V; // HS400 200MHz DDR
    ...
    card->mmc_avail_type = avail_type;              // ← 保存交集供后续使用
}
```

`card->mmc_avail_type` 就是第 8 步（CMD6 时序协商阶梯）的输入——从高到低尝试，走通就停。

**错误处理：不致命，降级也能跑**

`mmc_read_ext_csd` 的错误处理很宽松——只有 `-EINVAL` / `-ENOSYS` / `-EFAULT` 是"真的不行"，其他错误可以容忍：

```c
if ((err != -EINVAL) && (err != -ENOSYS) && (err != -EFAULT))
    return err;    // 真的失败，整个初始化终止

/* 降级：没有 EXT_CSD，卡只能靠 CSD 的基础能力运行 */
pr_warn("unable to read EXT_CSD, performance might suffer\n");
err = 0;           // 不阻断流程
```

对 ATK 板来说这一步不会出错——Kingston eMMC 5.1 完整支持 EXT_CSD，512 字节数据一次性块读取成功。

```
暗线状态机: Transfer State → CMD8 → Transfer State
          CMD8 是数据读取命令，不改变卡状态
```

---

**第 8 步：CMD6 — SWITCH（多次执行，时序/宽度/缓存）**

CMD6 是 eMMC 协议中唯一一条可以**修改 EXT_CSD 寄存器**的命令。内核在 `mmc_init_card()` 中多次调用 CMD6，写入不同的 EXT_CSD 偏移，完成时序切换、总线宽度、缓存使能和功能配置。详见下一节（4.4.5）。

---

### 4.4.5 时序协商 — 高速模式配置

> **状态机上下文：** 卡已处于 Transfer State（CMD7 选中）。所有 CMD6 操作都在此状态下完成，不改变卡状态。

从 400KHz 基线到 200MHz HS200/HS400——本节是 mmc_init_card 中**最关键的一段**：利用第 7 步读回来的 EXT_CSD 信息，通过多次 CMD6 写入卡内寄存器，把卡从默认的低速模式切换到高速模式。

#### 4.4.5.1 CMD6 协议基础

CMD6（SWITCH）是 eMMC 协议中**唯一能修改卡内部寄存器**的命令：

```
CMD6 arg 格式:
  [31:26] [25:24]   [23:16]      [15:8]      [7:0]
  保留     命令集     寄存器索引     写入值      保留
           (0=Normal)
```

内核用 `__mmc_switch()` 封装发送。例如写 `EXT_CSD[185] = 2`（HS200 时序）：

```c
__mmc_switch(card, EXT_CSD_CMD_SET_NORMAL,   // 命令集 = Normal
             EXT_CSD_HS_TIMING,               // 偏移   = 185
             EXT_CSD_TIMING_HS200,            // 值     = 2
             card->ext_csd.generic_cmd6_time, // 超时   = ~10ms
             0, false, true, MMC_CMD_RETRIES);
```

时序协商本质就是：**用 CMD6 反复写 EXT_CSD 中的时序寄存器和宽度寄存器，配合控制器的时钟/电压调节**，找到双方最优配置。

#### 4.4.5.2 协商入口 — `mmc_select_timing()`：高→低优先级

```c
/* mmc.c:1538 */
static int mmc_select_timing(struct mmc_card *card)
{
    int err = 0;

    if (!mmc_can_ext_csd(card))          // mmca_vsn <= 3，老卡没有 EXT_CSD
        goto bus_speed;                   // → 跳过，用默认时序

    /* 优先级 1: HS400ES — 最高速 */
    if (card->mmc_avail_type & EXT_CSD_CARD_TYPE_HS400ES) {
        err = mmc_select_hs400es(card);
        goto out;
    }

    /* 优先级 2: HS200 */
    if (card->mmc_avail_type & EXT_CSD_CARD_TYPE_HS200) {
        err = mmc_select_hs200(card);
        if (err == -EBADMSG)              // 失败 → 清除标志，降级
            card->mmc_avail_type &= ~EXT_CSD_CARD_TYPE_HS200;
        else
            goto out;                     // 成功，跳过后续降级
    }

    /* 优先级 3: HS 52MHz */
    if (card->mmc_avail_type & EXT_CSD_CARD_TYPE_HS)
        err = mmc_select_hs(card);

out:
    if (err && err != -EBADMSG)
        return err;

bus_speed:
    mmc_set_bus_speed(card);       // 根据最终选中的 timing 设时钟频率
    return 0;
}
```

`card->mmc_avail_type` 是上一节 `mmc_select_card_type()` 的计算结果——`ext_csd.raw_card_type`（卡能力）与 `host->caps / caps2`（控制器能力）的按位与。

**需要特别注意：`mmc_select_timing()` 不是完整的时序协商过程。** 它只负责选择**基础时序**（base timing），后续还有升级步骤在 `mmc_init_card()` 中完成：

```
mmc_select_timing()                      ← 内核优先级顺序
  ├─ HS400ES → 成功即止                 (最高速，需要增强选通)
  ├─ HS200   → 成功即止                 (中速，SDR 200MHz)
  └─ HS      → 保底                      (低速，SDR 52MHz)

mmc_init_card() 后续处理                 ← 在基础时序上进一步升级
  ├─ 如果当前是 HS200:
  │    调谐 → 尝试 mmc_select_hs400()   (HS200→HS→DDR→HS400 三段式)
  └─ 如果当前是 HS:
       设宽度 → 尝试 mmc_select_hs_ddr() (HS→DDR52 升级)
```

所以本节按 DDR52 → 其他高速模式的顺序组织：**4.4.5.4 DDR52 详细流程**（ATK 板实际使用）→ **4.4.5.5 其他高速模式**（HS200/HS400/HS400ES）→ **4.4.5.6 对比** → **4.4.5.7 其他 CMD6 用途**。

#### 4.4.5.3 时序三要素：电压、宽度、时序

高速模式配置涉及三个维度，**必须按特定顺序操作**：

| 要素 | 控制方式 | 可选值 |
|------|---------|--------|
| **信号电压** | `mmc_set_signal_voltage()` → 调 VCCQ | 3.3V / 1.8V / 1.2V |
| **总线宽度** | CMD6 写 `EXT_CSD[183]`（BUS_WIDTH） | 0=1bit, 1=4bit SDR, 2=8bit SDR, 5=4bit DDR, 6=8bit DDR |
| **时序模式** | CMD6 写 `EXT_CSD[185]`（HS_TIMING） | 0=Legacy, 1=HS, 2=HS200, 3=HS400 |

通用原则：**先切电压 → 再设宽度 → 再切时序 → 最后升时钟**。其中任何一步失败则整条路径降级。

#### 4.4.5.4 DDR52 详细流程 — ATK 板实际使用模式

> **ATK 板走 DDR52 的原因：** DTS 只声明了 `mmc-ddr-1_8v`，没有 `mmc-hs200-1_8v` 或 `mmc-hs400-1_8v`。因此 `mmc_select_card_type()` 的计算结果 `card->mmc_avail_type` 中不含 HS200/HS400 位，`mmc_select_timing()` 跳过前两个分支，直接进入 HS 路径线。

DDR52 是 ATK 板上 Kingston 58A43A eMMC 实际协商出来的最终高速模式。下面以代码路径 + 时序操作的对应关系来组织，展示每段代码对应的硬件行为：

```
mmc_init_card 中的 DDR52 完整路径:

mmc_select_timing()
  ├─ mmc_select_hs()          ① CMD6: 卡切 HS 时序, host 同步切 HS 时序
  └─ mmc_set_bus_speed()      ② mmc_set_clock(52MHz): 升频

mmc_init_card 后续处理
  ├─ mmc_select_bus_width()   ③ CMD6: 写 EXT_CSD[183], 8-bit SDR → 验证
  └─ mmc_select_hs_ddr()      ④ CMD6: 写 EXT_CSD[183], 8-bit DDR
```

##### ① CMD6 切 HS 时序 — `mmc_select_hs()`

```c
/* mmc.c:1076-1087 */
static int mmc_select_hs(struct mmc_card *card)
{
    err = __mmc_switch(card, EXT_CSD_CMD_SET_NORMAL,
                       EXT_CSD_HS_TIMING, EXT_CSD_TIMING_HS,    // [185] = 1
                       card->ext_csd.generic_cmd6_time,          // ~10ms 超时
                       MMC_TIMING_MMC_HS,                        // timing=1 → 同步改 host
                       true, true, MMC_CMD_RETRIES);             // send_status, crc_err_fatal
    return err;
}
```

`__mmc_switch()` 内部做了三件事（`mmc_ops.c:595`）：

```
① mmc_wait_for_cmd → CMD6: 写 EXT_CSD[185] = 1   ← 改卡侧: 卡内部切到 HS 模式
② mmc_poll_for_busy → 等待卡完成内部切换              ← 卡释放总线
③ timing != 0 → mmc_set_timing(host, MMC_TIMING_MMC_HS)  ← 改 host 侧
```

**关键认识：CMD6 只改卡侧，`mmc_set_timing` 只改 host 侧。** `__mmc_switch` 的 `timing` 参数只是把后者"顺带做了"。DDR52 路径中 `timing=MMC_TIMING_MMC_HS`，所以 CMD6 写完后 host 侧立即同步配置。这与 HS200 不同（HS200 传 `timing=0`，host 侧要等升频后再配，详见 4.4.5.5 节对比）。

CMD6 发送后，卡内部的行为（参考 eMMC 协议 Figure 135 — SWITCH 操作时序）：

```
CMD6 参数: [23:16]=185(HS_TIMING) [15:8]=1(HS)
  → 卡解析 arg，定位到 EXT_CSD byte 185
  → 写入值 1（HS_TIMING_ENABLE）
  → 卡内部 PLL/时序电路重新配置:
       时钟模式从 Legacy SDR 切换到 HS SDR
       内部逻辑频率上限从 26MHz 提高到 52MHz
  → 卡拉低 DAT0 表示 Busy（~10ms）
  → 释放总线，准备接收下一命令
```

`mmc_select_hs()` 的返回值没有校验 `SWITCH_ERROR` 位——判断卡是否真正支持 HS 是在 `mmc_select_timing()` 之前就通过 `card->mmc_avail_type` 完成了。这里假设 CMD6 必然成功。

##### ② 升频到 52MHz — `mmc_set_bus_speed()`

```c
/* mmc.c:1538, at out: label after mmc_select_timing */
mmc_set_bus_speed(card);
```

`mmc_set_bus_speed()` 根据当前 `host->ios.timing` 设置时钟：

```c
/* mmc.c:974-992 */
static void mmc_set_bus_speed(struct mmc_card *card)
{
    /* timing=MMC_TIMING_MMC_HS → max_dtr = card->ext_csd.hs_max_dtr */
    unsigned int max_dtr = mmc_card_max_dtr(card);

    /* ...钳位逻辑... */
    if (mmc_card_hs(card) && max_dtr > card->ext_csd.hs_max_dtr)
        max_dtr = card->ext_csd.hs_max_dtr;         // 不超过卡声明的 HS 上限

    mmc_set_clock(host, max_dtr);                   // → 写 MCI_CLKCR
}
```

`card->ext_csd.hs_max_dtr` 来自 EXT_CSD 第 46-49 字节（4 字节），Kingston 58A43A 声明为 52MHz。`mmc_set_clock` 的流程：

```c
/* core.c:914 */
void mmc_set_clock(struct mmc_host *host, unsigned int hz)
{
    if (hz > host->f_max)
        hz = host->f_max;            // DTS max-frequency=166MHz, 52MHz < 166MHz → 不钳位
    host->ios.clock = hz;            // 更新 ios 状态
    mmc_set_ios(host);               // → mmci_set_ios → 写 MCI_CLKCR
}
```

**此时总线状态：**

| 维度 | 卡侧 | host 侧 |
|------|------|---------|
| 时序 | HS (内部 PLL 锁定 52MHz) | `MMC_TIMING_MMC_HS`（寄存器位） |
| 时钟 | 接收 52MHz | 输出 52MHz（MCI_CLKCR 分频比已校准） |
| 宽度 | 1-bit（默认，尚未设置） | 1-bit（默认） |
| 电压 | VCCQ=1.8V（上电即固定） | 1.8V（scmi_vddio2） |

下一步 `mmc_select_bus_width()` 将在 52MHz 下执行，而不是 400KHz。这是 DDR52 与 HS200 路径的一个重要区别——HS200 的宽度切换在 400KHz 下做，DDR52 的宽度切换在已经升频后的 52MHz 下做。

##### ③ CMD6 设总线宽度 — `mmc_select_bus_width()`

从 `mmc_select_timing()` 返回后，`mmc_init_card` 进入 else 分支（非 HS200/HS400ES）：

```c
/* mmc.c:1828-1836 */
} else {
    /* Select the desired bus width optionally */
    err = mmc_select_bus_width(card);
    if (err > 0 && mmc_card_hs(card)) {
        err = mmc_select_hs_ddr(card);
        if (err)
            goto free_card;
    }
}
```

`mmc_select_bus_width()` 发送 CMD6 写 `EXT_CSD[183] BUS_WIDTH`。优先级顺序：**8-bit → 4-bit → 1-bit**，成功即返回：

```c
/* mmc.c:1005 */
static int mmc_select_bus_width(struct mmc_card *card)
{
    static const int ext_csd_bits[] = {           // SDR 宽度编码
        EXT_CSD_BUS_WIDTH_8,                      // = 2
        EXT_CSD_BUS_WIDTH_4,                      // = 1
        EXT_CSD_BUS_WIDTH_1,                      // = 0
    };
    static const int bus_widths[] = {
        MMC_BUS_WIDTH_8,
        MMC_BUS_WIDTH_4,
        MMC_BUS_WIDTH_1,
    };

    for (idx = 0; idx < 3; idx++) {
        /* ① CMD6 写 EXT_CSD[183] = ext_csd_bits[idx] */
        err = mmc_switch(card, ..., EXT_CSD_BUS_WIDTH,
                         ext_csd_bits[idx], ...);
        if (err)
            continue;                         // CMD6 失败 → 尝试更窄宽度

        /* ② host 侧同步设置总线宽度（写控制器寄存器） */
        mmc_set_bus_width(host, bus_widths[idx]);

        /* ③ 验证宽度是否真的生效了 */
        if (!(host->caps & MMC_CAP_BUS_WIDTH_TEST))
            err = mmc_compare_ext_csds(card, bus_width);
        else
            err = mmc_bus_test(card, bus_width);

        if (!err)
            return bus_widths[idx];           // 验证通过 → 返回正数（实际宽度）
    }
    return 0;                                   // 全失败 → 返回 0（1-bit 兜底）
}
```

**ATK 板**使用 `mmc_compare_ext_csds()` 验证（因为 `mmci` 没有设 `MMC_CAP_BUS_WIDTH_TEST` 位）。

宽度验证的原理详见 4.4.5.5 节（两种验证方法在 HS200/DDR52 路径中完全一样）。一句话总结：**CMD6 改宽度 → 在新宽度下重新读 EXT_CSD → 与基准值（1-bit 下读的）对比只读字段 → 一致则通过**。

`err > 0` 表示找到了非 1-bit 宽度（8-bit 或 4-bit）。此时判断 `mmc_card_hs(card)` 为真（第 ① 步设的 `MMC_TIMING_MMC_HS`），进入 `mmc_select_hs_ddr()`。

##### ④ CMD6 切换到 DDR — `mmc_select_hs_ddr()`

```c
/* mmc.c:1094-1157 */
static int mmc_select_hs_ddr(struct mmc_card *card)
{
    /* 检查卡是否支持 DDR52 */
    if (!(card->mmc_avail_type & EXT_CSD_CARD_TYPE_DDR_52))
        return 0;                          // 不支持 → 保持在 SDR HS

    /* ① DDR 宽度编码: 8-bit → 6, 4-bit → 5 */
    ext_csd_bits = (bus_width == MMC_BUS_WIDTH_8) ?
        EXT_CSD_DDR_BUS_WIDTH_8 : EXT_CSD_DDR_BUS_WIDTH_4;

    /* ② CMD6 写 EXT_CSD[183] = DDR 编码 */
    __mmc_switch(card, ..., EXT_CSD_BUS_WIDTH, ext_csd_bits,
                 card->ext_csd.generic_cmd6_time,
                 MMC_TIMING_MMC_DDR52,          // ← timing=DDR52, 同步改 host
                 1, 1, MMC_CMD_RETRIES);

    /* ③ 电压确认（如果需要） */
    if (card->mmc_avail_type & EXT_CSD_CARD_TYPE_DDR_1_2V)
        mmc_set_signal_voltage(host, MMC_SIGNAL_VOLTAGE_120);
    else if (card->mmc_avail_type & EXT_CSD_CARD_TYPE_DDR_1_8V &&
             host->caps & MMC_CAP_1_8V_DDR)
        mmc_set_signal_voltage(host, MMC_SIGNAL_VOLTAGE_180);

    return 0;
}
```

**SDR 与 DDR 宽度编码的区别：**

```
SDR 8-bit:  EXT_CSD[183] = 2 → 8 条数据线, 单沿采样（每个时钟沿采 1 bit）
DDR 8-bit:  EXT_CSD[183] = 6 → 8 条数据线, 双沿采样（每个时钟沿采 2 bit）
                        bit 3 = 1 表示 DDR 模式
                        编码: b'110 = DDR 8-bit, b'101 = DDR 4-bit
```

第 ② 步的 `__mmc_switch(timing=MMC_TIMING_MMC_DDR52)` 同时做了：

```
CMD6: 写 EXT_CSD[183] = 6     ← 卡侧: 切到 DDR 采样模式
                               （卡内部改变 I/O 缓冲区的采样时钟——
                                 从单个时钟沿改为双边沿采样）
                               host 侧: mmc_set_timing(MMC_TIMING_MMC_DDR52)
                                → 写控制器寄存器，通知 host 启用 DDR 模式
                                → host 侧改变数据采样策略（双边沿）
```

第 ③ 步的电压确认：ATK 板上 `scmi_vddio2` 固定 1.8V，效果同 HS200 路径中的"确认"而非"切换"（见 4.4.5.5 节分析）。

##### DDR52 完整状态迁移

```
初始 (mmc_init_card 入口):   200KHz, 1-bit, Legacy
    │
    ├── mmc_power_up → mmc_set_initial_signal_voltage
    │    VCCQ=1.8V (scmi_vddio2 fixed)
    │
    ├── 识别阶段 (CMD1, CMD2, CMD3, CMD9, CMD7)
    │    400KHz, 1-bit, Legacy
    │
    ├── EXT_CSD 读取 (CMD8)
    │    400KHz, 1-bit, Legacy
    │
    ├── ① mmc_select_hs: CMD6 EXT_CSD[185] = 1
    │    400KHz, 1-bit, HS        ← 卡已切，host 已切，但时钟还没升
    │
    ├── ② mmc_set_bus_speed: mmc_set_clock(52MHz)
    │    52MHz, 1-bit, HS         ← 双方都在 HS 52MHz
    │
    ├── ③ mmc_select_bus_width: CMD6 EXT_CSD[183] = 2, 验证
    │    52MHz, 8-bit, HS         ← 总线变宽，速率翻 8 倍
    │
    └── ④ mmc_select_hs_ddr: CMD6 EXT_CSD[183] = 6
         52MHz, 8-bit DDR, DDR52  ← 最终 ATK 板 DDR52 模式
```

##### 性能计算

| 参数 | 值 | 公式 |
|------|----|------|
| 时钟 | 52MHz | `ext_csd.hs_max_dtr` |
| 总线宽度 | 8-bit | DTS `bus-width=<8>` |
| 采样方式 | DDR（双沿） | CMD6 `EXT_CSD[183]=6` |
| **理论吞吐率** | **104MB/s** | `52MHz × 8bit × 2(DDR) ÷ 8` |

实际吞吐率因命令开销、DMA 延迟、文件系统元数据等因素大约为理论值的 60-80%。用 `dd if=/dev/mmcblk1 of=/dev/null bs=1M count=100` 实测通常在 **60-80MB/s** 范围。

#### 4.4.5.5 其他高速模式

> **定位：** 以下三种模式（HS400ES / HS200 / HS400）在 ATK 板上均不被触发。本节仅为完整性和对比而写，内容比 DDR52 节简洁。

##### HS200 路径（单沿 200MHz）

**入口条件：** `mmc_select_timing()` 第二优先级检查 `avail_type & EXT_CSD_CARD_TYPE_HS200`。ATK 板因 DTS 无 `mmc-hs200-1_8v`，此条件不成立。

`mmc_select_hs200()`（`mmc.c:1461-1533`）分 5 步：

| 步骤 | 操作 | 说明 |
|------|------|------|
| ① 驱动强度 | `mmc_select_driver_type()` | 配两边：card→CMD6 [185]:[7:4], host→`mmc_set_driver_type()` |
| ② 电压确认 | `mmc_set_signal_voltage()` | 1.2V→1.8V 优先级；ATK 固定 1.8V 只是"确认" |
| ③ 宽度 | `mmc_select_bus_width()` | 8→4→1-bit，`mmc_compare_ext_csds()` 验证 |
| ④ 时序 | `__mmc_switch(timing=0)` | CMD6 [185]=2，**timing=0** 不配 host（要等升频后） |
| ⑤ 升频+验证 | `mmc_set_timing`+`mmc_set_clock`+`mmc_switch_status(false)` | host 升频后再发 CMD13，CRC 错误被吞掉 |

五个要点：

**1) 驱动强度是双向配置。** 读操作卡驱动总线，写操作 host 驱动总线。卡侧通过 CMD6 写入 EXT_CSD[185] 的 [7:4] 位；host 侧通过 `mmc_set_driver_type()` → `mmci_set_ios()` 写控制器寄存器。`mmc_select_drive_strength()` 是分发函数（`core.c:1288`），回调 `host->ops->select_drive_strength`；mmci 未实现该回调，ATK 回退 Type 0。

```c
int mmc_select_drive_strength(struct mmc_card *card, unsigned int max_dtr,
                              int card_drv_type, int *drv_type)
{
    *drv_type = 0;
    if (!host->ops->select_drive_strength)
        return 0;     // mmci 未实现 → Type 0
    /* 否则: 取 host_drv_type ∩ card_drv_type 最优值 */
}
```

**2) 电压确认走 regulator，不发 CMD。** eMMC 的 VCCQ 完全由主机侧 power management IC 控制。ATK 板的 `scmi_vddio2` 固定 1.8V，`mmc_regulator_set_vqmmc()` 内部检查发现已在目标电压，直接返回"已匹配"。详见 4.4.5.4 第 ① 步的电压分析（同样适用于此场景）。

**3) HS200 传 `timing=0`，与 DDR52 的 `__mmc_switch` 行为不同。** 原因是 CMD6 写完后卡已切到 HS200（200MHz 内部 PLL），但 host 还在 400KHz。如果此时发 CMD13 验证，两者的频率不匹配会导致 CRC 错误。所以正确顺序是：CMD6 写完→亲自升 host 频率→双方匹配后再验证。

| 路径 | `__mmc_switch` timing 参数 | host 侧配置时机 |
|------|---------------------------|---------------|
| DDR52 (HS) | `MMC_TIMING_MMC_HS` (≠0) | `__mmc_switch` 内部顺带 |
| HS200 | `0` | 调用者手动 `mmc_set_timing` + `mmc_set_clock` |

**4) 验证用 `crc_err_fatal=false`。** 第一次 CMD13 可能因时序刚切换不稳定出现 CRC 错误（`-EILSEQ`），但内核选择忽略——因为 CRC 错误不代表切换失败，调谐会最终验证。这是 `mmc_switch_status(card, false)` 的逻辑（`mmc_ops.c:446`）：

```c
if (!crc_err_fatal && err == -EILSEQ)
    return 0;       // 吞掉 CRC 错误，不是重试 —— 就是忽略
```

只有 `-EBADMSG`（卡状态寄存器中的 `SWITCH_ERROR` 位）触发错误恢复（恢复旧时钟/时序/电压）。

**5) 时钟被 `host->f_max` 钳位。** DTS `max-frequency = <166000000>`，`mmc_set_clock()` 内部 `hz = min(hz, host->f_max)`。即使 `MMC_HS200_MAX_DTR = 200MHz`，实际 HS200 时钟仅为 166MHz → **166MB/s**。

##### HS400 升级路径（HS200 调谐后尝试）

> **不是 `mmc_select_timing()` 直接选择的模式。** HS400 在 `mmc_init_card()` 中 HS200 调谐成功后额外尝试（`mmc.c:1814-1816`），所以它实质上是 HS200 的"高级附加选项"。

HS200→HS400 不能直接切——协议规定宽度寄存器只能在 HS 时序下修改。HS400 升级走"三段式"（降→DDR 宽→升）：

```
① CMD6: EXT_CSD[185]=1 (HS)         — 降回 HS，频率同步降 200→52MHz
② CMD6: EXT_CSD[183]=6 (DDR 8-bit) — 改 DDR 宽度编码
③ CMD6: EXT_CSD[185]=3 (HS400)      — 升到 HS400，频率恢复 200MHz
```

HS400 理论吞吐率 = 200MHz × 8bit × 2(DDR) = **400MB/s**。

##### HS400ES（增强选通，最高优先级）

**三个条件同时满足才触发**（`mmc.c:250-253`）：

```c
if ((caps2 & MMC_CAP2_HS400_ES) &&              // 主机支持
    card->ext_csd.strobe_support &&               // 卡声明支持
    (avail_type & EXT_CSD_CARD_TYPE_HS400))       // 基础 HS400 能力
    avail_type |= EXT_CSD_CARD_TYPE_HS400ES;
```

与普通 HS400 的区别：增加专用 **DS（Data Strobe）线**由 eMMC 驱动，数据与选通信号在芯片内对齐，不受 PCB 延迟影响，不需要 CMD21 调谐。ATK 板的 `host->caps2` 中没有 `MMC_CAP2_HS400_ES`，永不触发。

#### 4.4.5.6 完整路径对比

| 路径 | 步骤 | 操作 | CMD6 写入 | 时钟 | 宽度 | 吞吐率 |
|------|------|------|-----------|------|------|--------|
| **DDR52** | ① | 切 HS 时序 | EXT_CSD[185]=1 | 400KHz | 1-bit | — |
| → | ② | 升频 | — | **52MHz** | 1-bit | — |
| → | ③ | 切宽度 | EXT_CSD[183]=2/1 | 52MHz | 8/4-bit | 52/26MB/s |
| → | ④ | 切 DDR | EXT_CSD[183]=6/5 | 52MHz | DDR | **104/52MB/s** |
| **HS200** | ① | 切电压 1.8V | — | 400KHz | 1-bit | — |
| → | ② | 切宽度 | EXT_CSD[183]=2 (8-bit) | 400KHz | 8-bit | — |
| → | ③ | 切时序 | EXT_CSD[185]=2 (HS200) | 400KHz | 8-bit | — |
| → | ④ | 升频+调谐 | — | **200MHz** | 8-bit | **200MB/s** |
| ↓ **HS400** | ⑤ | 降级 HS | EXT_CSD[185]=1 | 200→52MHz | 8-bit | — |
| → | ⑥ | 切 DDR 宽度 | EXT_CSD[183]=6 (DDR8) | 52MHz | DDR 8-bit | — |
| → | ⑦ | 升 HS400 | EXT_CSD[185]=3 | 52MHz | DDR 8-bit | — |
| → | ⑧ | 升频+调谐 | — | **200MHz** | DDR 8-bit | **400MB/s** |

#### 4.4.5.7 mmc_init_card 中的其他 CMD6 用法

| 用途 | EXT_CSD 偏移 | 说明 |
|------|-------------|------|
| **擦除组定义** | [181] ERASE_GROUP_DEF | 启用 HC 擦除分组 |
| **电源关闭通知** | [192] POWER_OFF_NOTIFICATION | 使能休眠电源通知 |
| **HPI 使能** | [165] HPI_MGMT | 允许中断长时间操作 |
| **缓存使能** | [33] CACHE_CTRL | 启用内部写缓存 |
| **CMDQ 使能** | 通过 `mmc_cmdq_enable()` | 启用命令队列 |

#### 4.4.5.8 CMD6 使能缓存（Cache Enable）

> **状态机上下文：** Transfer State，CMD6 修改 EXT_CSD 寄存器，不改变卡状态。

时序协商和总线宽度完成后，`mmc_init_card()` 的最后一步是使能 eMMC 内部写缓存：

```c
/* mmc.c:1861-1887 */
/*
 * If cache size is higher than 0, this indicates the existence of cache
 * and it can be turned on. Note that some eMMCs from Micron has been
 * reported to need ~800 ms timeout, while enabling the cache after
 * sudden power failure tests. Let's extend the timeout to a minimum of
 * DEFAULT_CACHE_EN_TIMEOUT_MS and do it for all cards.
 */
if (card->ext_csd.cache_size > 0) {
    unsigned int timeout_ms = MIN_CACHE_EN_TIMEOUT_MS;   // = 1600ms

    timeout_ms = max(card->ext_csd.generic_cmd6_time, timeout_ms);
    err = mmc_switch(card, EXT_CSD_CMD_SET_NORMAL,
                     EXT_CSD_CACHE_CTRL, 1, timeout_ms);
    if (err && err != -EBADMSG)
        goto free_card;

    if (err) {
        /* -EBADMSG: 卡声明有缓存但实际不支持 */
        pr_warn("%s: Cache is supported, but failed to turn on (%d)\n",
                mmc_hostname(card->host), err);
        card->ext_csd.cache_ctrl = 0;
    } else {
        card->ext_csd.cache_ctrl = 1;
    }
}
```

逐行解读：

**① `card->ext_csd.cache_size > 0` — 卡是否有缓存？**

`cache_size` 来自 EXT_CSD[249:252]（rev >= 6 取 4 字节，否则取 [249:250] 2 字节），单位是字节。如果为 0，说明卡内部没有写缓存硬件，整个流程跳过。

**② `MIN_CACHE_EN_TIMEOUT_MS = 1600ms` — 超时陷阱**

注释说明背景：**Micron 某些 eMMC 在断电测试后需要约 800ms 才能完成缓存使能**。内核保守地把所有卡的缓存使能超时设为至少 1600ms，再和 `generic_cmd6_time`（EXT_CSD[148]，通常 ~10ms）取最大值。对比普通 CMD6 时序切换的超时通常仅 ~10ms，此处长了两个数量级。

**③ `EXT_CSD_CACHE_CTRL` — 字节 33**

写 1 使能缓存，写 0 禁用缓存（R/W）。与 FLUSH（字节 32，WO）是两个独立寄存器——CACHE_CTRL 控制开关，FLUSH_CACHE 写任意值触发刷写。

**④ 错误处理**

| 返回值 | 含义 | 处理 |
|--------|------|------|
| `0` | 成功 | `cache_ctrl = 1` |
| `-EBADMSG` | 卡 SWITCH_ERROR | 卡声称有缓存但实际不支持，仅警告不阻断 |
| 其他错误 | CMD6 传输异常 | `goto free_card`，初始化失败 |

**⑤ `cache_ctrl` 标志的后续用途**

`card->ext_csd.cache_ctrl = 1` 在后续多处被检查：
- `_mmc_cache_enabled()`：`return cache_size > 0 && cache_ctrl & 1`
- `mmc_blk_alloc_req()` 第⑤步：通过 `mmc_cache_enabled()` 设置 `QUEUE_FLAG_WC`
- `_mmc_flush_cache()`：只在 `cache_enabled` 为真时发 FLUSH
- `Suspend` 路径：休眠前刷缓存，唤醒后重新使能

**⑥ 与块层的关系**

卡初始化中 CMD6 写字节 33=1 完成**硬件使能**；后续 `mmc_blk_probe()` 在块层通过 `blk_queue_write_cache()` 设置 `QUEUE_FLAG_WC`，告诉文件系统"该设备支持写缓存"。两个层面独立——硬件使能 + 软件通告，详见 4.5.4 节第⑤步。

### 4.4.6 Tuning — DLYB 延迟线校准

**状态机上下文：** Tuning 在 **Transfer State** 中执行。CMD21（SEND_TUNING_BLOCK）是数据传输命令，卡在 Transfer State 内接收调谐块并返回——每发一次 CMD21 卡短暂进入 **Data State** 传输数据，传完回到 **Transfer State**。状态迁移模式：`Transfer ↔ Data`。

> **ATK 板不执行 tuning。** `sdmmc_execute_tuning()` 入口有频率门槛：`clock <= dlyb->min_freq` 时直接 `return 0`（`mmci_stm32_sdmmc.c:684-687`）。MP25 的 `min_freq = 100MHz`，而 DDR52 只跑 52MHz，所以 tuning 整个被跳过。以下分析针对 HS200/HS400 场景。

#### 入口守卫：谁需要 tuning

```c
/* mmci_stm32_sdmmc.c:677 */
static int sdmmc_execute_tuning(struct mmc_host *mmc, u32 opcode)
{
    /* ── 只对 HS200/SDR104 且频率够高时执行 ── */
    if ((host->mmc->ios.timing != MMC_TIMING_UHS_SDR104 &&
         host->mmc->ios.timing != MMC_TIMING_MMC_HS200) ||
        host->mmc->actual_clock <= dlyb->min_freq)
        return 0;                    // ← ATK 板 DDR52 52MHz < 100MHz，跳过

    /* ── 使能延迟线硬件 ── */
    ret = dlyb->ops->dlyb_enable(dlyb);
    ...
    /* ── 切换时钟源为 SDMMC_FBCK（反馈时钟）─ */
    clk = host->clk_reg;
    clk &= ~MCI_STM32_CLK_SEL_MSK;
    clk |= MCI_STM32_CLK_SELFBCK;    // 时钟源从 CK 切到 FBCK
    mmci_write_clkreg(host, clk);
    ...
    /* ── 延迟线初始化 ── */
    ret = dlyb->ops->tuning_prepare(host);
    ...
    /* ── 相位扫描 ── */
    return sdmmc_dlyb_phase_tuning(host, opcode);
}
```

三件事：使能 DLYB → 切时钟源（FBCK 使能反馈路径以补偿总延迟） → 相位扫描。

#### 硬件差异：MP15 DLYB vs MP25 DLYBSD

| 特性 | MP15（DLYB） | MP25（DLYBSD） |
|------|-------------|---------------|
| 延迟线类型 | 模拟延迟单元链（SEL+UNIT+LNG） | 数字抽头选择（RXTAPSEL） |
| 可调相位数 | `__fls(lng)` ~11 步 | 32 级抽头 |
| 校准方式 | 需要自校准确定 UNIT 值 | 直接选择抽头，无需校准 |
| 对应文件 | `mmci_stm32_sdmmc.c` DLYB 寄存器 | `mmci_stm32_sdmmc.c` SYSCFG 寄存器 |

MP15 的 DLYB 有一个自校准过程（`sdmmc_dlyb_mp15_prepare`）：遍历 UNIT 值写入 `DLYB_CFGR`，等待 `LNGF` 位置位，读到 `LNG` 值后取 `__fls(lng)` 作为有效步进数。MP25 的 DLYBSD 则简单得多——`sdmmc_dlyb_mp25_prepare` 直接设置 `dlyb->max = 32`，不需要校准。

#### CMD21 通过/失败判定 — `mmc_send_tuning()`

每个相位点调用一次 `mmc_send_tuning()`（`mmc_ops.c:668`），它在协议层和数据层两级检查：

```c
int mmc_send_tuning(struct mmc_host *host, u32 opcode, int *cmd_error)
{
    /* 准备 64 字节接收缓冲区 */
    data_buf = kzalloc(size, GFP_KERNEL);

    /* 发 CMD21：读 1 块调谐数据 */
    cmd.opcode = opcode;              // CMD21 (MMC) 或 CMD19 (SD)
    cmd.flags = MMC_RSP_R1 | MMC_CMD_ADTC;

    data.blksz = size;                // 8-bit: 64 bytes, 4-bit: 64 bytes
    data.blocks = 1;
    data.flags = MMC_DATA_READ;
    data.timeout_ns = 150 * NSEC_PER_MSEC;

    mmc_wait_for_req(host, &mrq);     // 执行命令

    /* ① 协议层检查：命令/数据传输是否出错 */
    if (cmd.error) { err = cmd.error; goto out; }   // CRC 错误、超时等
    if (data.error) { err = data.error; goto out; }  // 数据阶段错误

    /* ② 数据层检查：读回的数据模式是否匹配 */
    if (memcmp(data_buf, tuning_block_pattern, size))
        err = -EIO;                   // 数据不匹配 → 采样点不对

out:
    kfree(data_buf);
    return err;                       // 0 = 通过，非零 = 失败
}
```

调谐数据块是一个已知的固定模式（`tuning_blk_pattern_8bit`，64 字节），设计上包含丰富的比特跳变（`0xff`/`0x00` 边界、`0xcc`/`0x33` 相邻位翻转），使得任何采样偏移都会导致模式不匹配：

```
tuning_blk_pattern_8bit[0..15]:
  ff ff 00 ff ff ff 00 00 ff ff cc cc cc 33 cc cc
  ↑        ↑                    ↑        ↑
  全 1→全 0  边界               相邻位翻转   相邻位翻转
```

**通过条件（两者都必须满足）：**

| 检查层级 | 条件 | 失败含义 |
|---------|------|---------|
| 协议层 | `cmd.error == 0 && data.error == 0` | 卡没有响应 CRC 错误，数据 CRC 校验通过 |
| 数据层 | `memcmp == 0` | 采到的 64 字节与预期完全一致 |

**如果协议层报错（cmd.error 或 data.error）：** 说明连数据都没读回来——可能是相位偏移太大，卡根本不响应或响应 CRC 校验失败。这种情况在 `sdmmc_dlyb_phase_tuning` 中视为"该相位不通"，记一次失败。

**如果 memcmp 失败（返回 -EIO）：** 命令和数据 CRC 都通过了，但采样点在错误的边沿附近，导致 1 个或多个比特采样错误。这是"眼图刚好闭上"的阶段——CRC 还没错（因为 CRC 码本身也以相同偏移被采样，也能凑巧通过校验），但数据内容已经不对了。

#### 相位扫描算法 — `sdmmc_dlyb_phase_tuning()`

```c
/* mmci_stm32_sdmmc.c:632 */
static int sdmmc_dlyb_phase_tuning(struct mmci_host *host, u32 opcode)
{
    int cur_len = 0, max_len = 0, end_of_len = 0;

    /* ── 遍历所有相位 ── */
    for (phase = 0; phase <= dlyb->max; phase++) {
        dlyb->ops->set_cfg(dlyb, dlyb->unit, phase, false);
        // 对 MP25: 写 SYSCFG_DLYBSD_CR.RXTAPSEL[6:1] = phase

        if (mmc_send_tuning(host->mmc, opcode, NULL)) {
            cur_len = 0;              // 失败 → 连续窗口断裂
        } else {
            cur_len++;                // 通过 → 延长窗口
            if (cur_len > max_len) {
                max_len = cur_len;    // 更新最长窗口
                end_of_len = phase;   // 记录窗口结束位置
            }
        }
    }

    if (!max_len) {
        dev_err(..., "no tuning point found\n");
        return -EINVAL;               // 全不通 → 调谐失败
    }

    /* ── 取最长窗口中心 ── */
    phase = end_of_len - max_len / 2;
    dlyb->ops->set_cfg(dlyb, dlyb->unit, phase, false);

    return 0;
}
```

算法核心：**找最长的连续通过窗口，取中心点。**

```
相位:     0   1   2   3   4   5   6   7  ...  31
结果:     ✗   ✓   ✓   ✓   ✓   ✓   ✗   ✗       ✗
                 |<---  max_len=5  --->|
                                     end_of_len=5
                                     中心 = 5 - 5/2 = 3
```

为什么取窗口中心？因为眼图中心是采样裕量最大的位置。温度变化或电压漂移会使眼图左右收缩，如果选在边缘，收缩一点就进误码区；选在中心则容忍度最大。

| 返回码 | 含义 | 结果 |
|--------|------|------|
| `0` | 找到至少一个可通过的相位 | 延迟值写入 DLYB/SYSCFG，调谐完成 |
| `-EINVAL` | 所有相位全部失败（`max_len == 0`） | HS200 初始化失败 → 整卡 free_card |

#### 为什么 ATK 板 DDR52 不需要 tuning

DDR52 的工作频率是 52MHz，信号周期 ~19.2ns。MB 级 PCB 走线延迟在纳秒量级（FR4 上 1 inch ≈ 160ps），即使不考虑 DLYB 补偿，采样裕量也足够大。tuning 的必要性与频率成正比：

| 模式 | 频率 | 周期 | 数据有效窗口 | 需要 tuning？ |
|------|------|------|-------------|-------------|
| Legacy / HS | ≤52MHz | ≥19.2ns | ~10ns | 不需要 |
| DDR52 | 52MHz DDR | 19.2ns/双沿 | ~5ns/沿 | 不需要 |
| HS200 | 200MHz | 5ns | ~2-3ns | **必须** |
| HS400 | 200MHz DDR | 5ns/双沿 | ~1.5ns/沿 | **必须** |

52MHz 下一个 bit 的采样窗口约 10ns，PCB 走线延迟通常 < 1ns，只占窗口的 10%，不需要精细补偿。200MHz 下一个 bit 的采样窗口仅 2-3ns，走线延迟就会占 30-50%，必须通过 tuning 找到最佳采样点。

---

## 4.5 阶段四：块设备注册 — `mmc_blk_probe()`

> **核心问题：** 阶段三结束时 eMMC 卡已经跑在 DDR52 模式、CMD 序列全部完成、`mmc_card` 结构体填充了完整的 EXT_CSD 信息。但此时还没有 `/dev/mmcblk1`——块设备节点的创建是阶段四的任务。
>
> 阶段四的输入是 `mmc_card`（卡设备），输出是 3 个块设备节点 + 1 个字符设备节点（`mmcblk1`、`mmcblk1boot0`、`mmcblk1boot1`、`mmcblk1rpmb`——其中 RPMB 是字符设备）。背后的操作是：**MMC 总线匹配 → mmc_block 驱动 probe → gendisk 分配与注册 + RPMB 字符设备创建**。

### 4.5.1 从 mmc_card 到块设备 — 总览

阶段四涉及两个独立的总线/设备模型层次：

```
   AMBA 总线                    MMC 总线
  (阶段二)                     (阶段四)
┌──────────────────┐        ┌──────────────────────┐
│   amba_device    │  probe  │    mmc_card          │  ← 阶段三的输出
│   (SDMMC2 控制器) │───→    │   (卡设备)            │
│   mmci-pl18x     │        │   dev.bus = mmc_bus   │
└──────────────────┘        │        ↓ match        │
                             │   mmc_block 驱动      │
                             │   mmc_blk_probe()     │
                             │        ↓              │
                             │   gendisk × 4         │
                             │   /dev/mmcblk1 ...    │
                             └──────────────────────┘
```

关键认识：**阶段二的 AMBA 匹配找到了控制器驱动（mmci-pl18x），阶段四的 MMC 总线匹配找到了块设备驱动（mmc_block）。** 到阶段三完成时，mmc_block 还没参与——所有 CMD 序列都是 mmc_core 编排的，mmc_block 是事后才被 probe 的。

### 4.5.2 MMC 总线匹配 — `mmc_add_card()` → `device_add()`

CMD 序列完成后 `mmc_attach_mmc()` 的最后一步：

```c
/* mmc_attach_mmc() → mmc.c:1878 */
mmc_add_card(host->card);
```

`mmc_add_card()`（`bus.c:300`）初始化 `mmc_card.dev` 并注册到设备模型：

```c
int mmc_add_card(struct mmc_card *card)
{
    device_initialize(&card->dev);

    /* parent = MMC host class device (/sys/class/mmc_host/mmc1/) */
    card->dev.parent = &card->host->class_dev;
    card->dev.release = mmc_release_card;

    /* bus = mmc_bus_type — 挂到 MMC 总线上 */
    card->dev.bus = &mmc_bus_type;

    /* 设备名 = "mmc1:0001"（host名:卡地址） */
    dev_set_name(&card->dev, "%s:%04x",
                 mmc_hostname(card->host), card->rca);

    /* 注册到 Linux 设备模型 — 触发总线匹配 */
    ret = device_add(&card->dev);
}
```

`device_add()` 内部的核心过程（Linux 设备模型标准流程）：

```
device_add(card->dev)
  ├─ dev->bus = &mmc_bus_type              ← 标记总线归属
  ├─ kobject_add → /sys/devices/.../mmc1:0001/  ← sysfs 入口
  ├─ bus_add_device                         ← 在总线上注册
  └─ bus_probe_device                       ← 触发匹配 probe
        └─ device_initial_probe
              └─ __device_attach
                    └─ bus_for_each_drv(mmc_bus_type, ...)  ← 遍历所有注册的驱动
                         └─ __device_attach_driver(drv, dev)
                              ├─ driver_match_device(drv, dev)
                              ├─ driver_probe_device(drv, dev)
                              │    └─ __driver_probe_device
                              │         └─ really_probe → drv->probe(card)
                              └─ 返回值决定是继续循环还是中断
```

#### 关键一：`mmc_bus_type` 没有 `.match` 回调

`mmc_bus_type` 的定义（`bus.c:219`）：

```c
static struct bus_type mmc_bus_type = {
    .name       = "mmc",
    .dev_groups = mmc_dev_groups,
    .uevent     = mmc_bus_uevent,
    .probe      = mmc_bus_probe,    // 直接调用 drv->probe(card)
    .remove     = mmc_bus_remove,
    .shutdown   = mmc_bus_shutdown,
    .pm         = &mmc_bus_pm_ops,
    /* ⚠️ 没有 .match 成员！ */
};
```

没有 `.match` 时，`driver_match_device()` 的默认行为（`base.h:164`）：

```c
static inline int driver_match_device(struct device_driver *drv,
                                      struct device *dev)
{
    return drv->bus->match ? drv->bus->match(dev, drv) : 1;
    //                        ↑ 没有 match → 默认返回 1（匹配）
}
```

**所有注册到 `mmc_bus_type` 上的驱动都会收到"匹配成功"**，批量的 probe 尝试即将开始。

#### 关键二：`mmc_driver` 结构体也没有 id_table

`mmc_driver`（`bus.h:33`）：

```c
struct mmc_driver {
    struct device_driver drv;
    int (*probe)(struct mmc_card *card);
    void (*remove)(struct mmc_card *card);
    void (*shutdown)(struct mmc_card *card);
    /* ⚠️ 没有 id_table */
};
```

MMC 总线既不靠 vendor/device ID，也不靠 id_table 匹配。**匹配条件是"全部放行"，过滤由 probe 函数自己决定。**

#### 关键三：内核的 probe 批处理逻辑

`__device_attach_driver()` 是每个驱动被调用时的回调（`dd.c:922-962`）：

```c
static int __device_attach_driver(struct device_driver *drv, void *_data)
{
    struct device_attach_data *data = _data;
    struct device *dev = data->dev;
    int ret;

    /* ① 匹配阶段 */
    ret = driver_match_device(drv, dev);
    if (ret == 0) return 0;              // 不匹配 → 跳过，继续下一个
    if (ret == -EPROBE_DEFER) return ret; // 延迟 → 中断循环
    if (ret < 0) return ret;             // 错误 → 中断循环
    /* ret > 0: 匹配成功 */

    /* ② probe 阶段 — 注意注释！ */
    /*
     * Ignore errors returned by ->probe so that the next driver can try
     * its luck.
     */
    ret = driver_probe_device(drv, dev);
    if (ret < 0)
        return ret;        // -EBUSY/-EPROBE_DEFER → 中断循环
    return ret == 0;       // ★ 0(成功)→1中断, 正值(失败)→0继续
}
```

`driver_probe_device()` 内部调用 `__driver_probe_device()`（`dd.c:780-800`）：

```c
static int __driver_probe_device(struct device_driver *drv, struct device *dev)
{
    if (dev->p->dead || !device_is_registered(dev))
        return -ENODEV;

    if (dev->driver)        // ← 关键门卫：该设备已经有驱动了？
        return -EBUSY;      //   有 → 拒绝

    /* 首次 probe */
    ret = really_probe(dev, drv);
    ...
}
```

`really_probe()`（`dd.c:604`）是最终执行 probe 的地方：

```
really_probe(dev, drv)
  ├─ dev->driver = drv;                    ← 先把驱动临时挂上 (dd.c:633)
  ├─ call_driver_probe(dev, drv)           ← 调 bus.probe() → drv->probe(card)
  │    └─ mmc_blk_probe(card)
  │         ├─ 检查 CCC_BLOCK_READ         ← 自身过滤
  │         ├─ 分配资源、创建设备
  │         └─ return 0;                   ← 成功
  │
  ├─ [成功路径]                            ← probe 返回 0
  │    └─ driver_bound(dev)                ← dev->driver 正式生效
  │    └─ return 0
  │
  └─ [失败] probe 返回 -ENODEV 等
       ├─ ret = -ret;                      ← 转为正值 (dd.c:674)
       ├─ device_unbind_cleanup(dev)
            └─ dev->driver = NULL;         ← 清除驱动标记 (dd.c:554)
       └─ return 正值
```

#### 完整的遍历决策逻辑

把返回值链拼起来（注意 `return ret == 0` 的布尔语义）：

```
                           │ driver_probe_device  │ __device_attach_driver  │ bus_for_each_drv
                           │ 返回值               │ return ret == 0         │ if (error) break;
───────────────────────────┼──────────────────────┼─────────────────────────┼─────────────────
probe 成功 (返回 0)        │ 0                    │ 0==0 → true → 1        │ break！✓ 中断
probe 失败 (如 -ENODEV)    │ 正数(19)             │ 19==0 → false → 0      │ continue 🠷
dev->driver 已被占 (-EBUSY)│ -EBUSY               │ ret<0 → return -EBUSY   │ break！中断
```

**三个分支的正确理解：**

| 场景 | `driver_probe_device` 返回 | `__device_attach_driver` 返回 | `bus_for_each_drv` | 说明 |
|------|---------------------------|-------------------------------|--------------------|------|
| ✅ probe 成功 | `0` | `return 0==0` → **1** | **中断** | **本驱动绑定成功，不再尝试后续驱动** |
| ❌ probe 失败（如 -ENODEV） | 正数（`really_probe` 转换后） | `return 正数==0` → **0** | **继续** | 忽略错误，给下一个驱动机会 |
| 🚫 dev->driver 已占用 | `-EBUSY` | `return -EBUSY` → **负值** | **中断** | 安全门卫，防止二次绑定 |

**总结：如果某个驱动 probe 成功，循环立即停止。只有 probe 失败的驱动才会被跳过（继续循环）。**

但这是否意味着"先注册的驱动一定赢"？不完全。详见下面的分析。

#### ATK 板实际情况：只存在一个驱动

检查 ATK BSP 内核的 `.config`：

```
CONFIG_MMC_BLOCK=y
# CONFIG_MMC_TEST is not set    ← mmc_test 未编译
```

**MMC 总线上只注册了 `mmc_block` 一个驱动。没有竞争、没有顺序依赖。** `mmc_add_card` → `device_add` → `bus_for_each_drv` 遍历的唯一驱动就是 `mmc_block`，匹配 → probe → 绑定，完成。

#### 拓展：如果存在多个 MMC 驱动呢？

MMC 子系统中除了 `mmc_block`，还有 `mmc_test` 驱动（调试用，`CONFIG_MMC_TEST` 开启时编译）。两者都用 `module_init()` 注册到 `mmc_bus_type`。

在 Makefile（`drivers/mmc/core/Makefile`）中：

```makefile
obj-$(CONFIG_MMC_BLOCK)  += mmc_block.o    # block.o + queue.o
obj-$(CONFIG_MMC_TEST)   += mmc_test.o     # 调试驱动
```

`mmc_block.o` 链接在前，因此 `mmc_blk_init` 的 `module_init` 先于 `mmc_test_init` 执行，`mmc_block` 驱动先被注册到总线。

但 `mmc_test` 是否可能抢在 `mmc_block` 之前 probe？可以但不会发生，因为：

- 两个驱动的 probe 函数对 eMMC 卡都会返回 0（都支持）
- 但 `mmc_block` 先注册 → 先被 `bus_for_each_drv` 遍历到 → probe 成功 → `__device_attach_driver` 返回 1 → **循环中断** → `mmc_test` 不会被轮到

如果反过来（`mmc_test` 先注册），它确实会先绑定设备，然后循环中断，`mmc_block` 根本没机会被遍历到——结果就是没有 `/dev/mmcblk1`。**这正是为什么生产系统中只让一个驱动（mmc_block）注册到 MMC 总线的原因**——内核不依靠"刚好链接顺序对"来保证正确性。

手动 bind 也是如此。通过 sysfs 手动绑定：

```bash
echo mmc1:0001 > /sys/bus/mmc/drivers/mmc_test/bind
```

底层走 `device_driver_attach()` → `__driver_probe_device()` → 同样检查 `dev->driver` → **`-EBUSY`**。必须先 unbind：

```bash
echo mmc1:0001 > /sys/bus/mmc/drivers/mmcblk/unbind   # 清空 dev->driver
echo mmc1:0001 > /sys/bus/mmc/drivers/mmc_test/bind    # 再绑 mmc_test
```

**总结：MMC 总线没有 vendor/device ID 表，也没有 match 回调。所有驱动都被"放行"尝试 probe，一旦某个驱动 probe 成功，`__device_attach_driver` 返回 1 中断 `bus_for_each_drv` 循环。`dev->driver` 门卫是防止二次绑定的辅助安全机制。** 过滤由 probe 函数自检完成——`mmc_blk_probe` 检查 `CCC_BLOCK_READ`。ATK 板只有一个 `mmc_block` 驱动，所以匹配过程只遍历一次、直接通过、绑定完成。

### 4.5.3 `mmc_blk_probe()` — 逐步骤分析

**入口：** `drivers/mmc/core/block.c:3007`

```c
static int mmc_blk_probe(struct mmc_card *card)
{
    struct mmc_blk_data *md;
    int ret = 0;

    // ── ① 安全检查 ──────────────────────────────────
    if (!(card->csd.cmdclass & CCC_BLOCK_READ))
        return -ENODEV;

    mmc_fixup_device(card, mmc_blk_fixups);

    // ── ② 工作队列 ──────────────────────────────────
    card->complete_wq = alloc_workqueue("mmc_complete",
                            WQ_MEM_RECLAIM | WQ_HIGHPRI, 0);

    // ── ③ 主分区块设备 ───────────────────────────────
    md = mmc_blk_alloc(card);           // → /dev/mmcblk1

    // ── ④ 硬件子分区 ────────────────────────────────
    ret = mmc_blk_alloc_parts(card, md); // → boot0, boot1, rpmb

    // ── ⑤ DebugFS 诊断接口 ──────────────────────────
    mmc_blk_add_debugfs(card, md);

    // ── ⑥ 运行时电源管理 ────────────────────────────
    pm_runtime_set_autosuspend_delay(&card->dev, 3000);
    pm_runtime_use_autosuspend(&card->dev);
    pm_runtime_set_active(&card->dev);
    pm_runtime_enable(&card->dev);

    return 0;
}
```

下面对不在其他节详细展开的步骤（①、②、⑤、⑥）做细致分析。③ 和 ④ 的展开见 4.5.4 和 4.5.5。

---

#### ① `CCC_BLOCK_READ` + `mmc_fixup_device`

**`CCC_BLOCK_READ` 过滤器：**

`card->csd.cmdclass` 是从 CSD 寄存器读取的位图，每位表示卡支持的一组命令：

```c
#define CCC_BLOCK_READ   (1 << 5)   /* CMD17: READ_SINGLE_BLOCK */
                                     /* CMD18: READ_MULTIPLE_BLOCK */
```

eMMC 芯片必须支持块读命令，否则 `mmc_block` 驱动拒绝绑定。这本质上就是 4.5.2 节讨论的 probe 自检过滤——`mmc_blk_probe` 用 `CCC_BLOCK_READ` 确认卡是可寻址的块设备。

**`mmc_fixup_device()` — 怪癖补丁：**

```c
/* quirks.h:218 */
static inline void mmc_fixup_device(struct mmc_card *card,
                                    const struct mmc_fixup *table)
{
    for (f = table; f->vendor_fixup; f++) {
        /* CID 匹配：制造商、OEM、产品名、版本 */
        if (f->manfid != CID_MANFID_ANY &&
            f->manfid != card->cid.manfid) continue;
        if (f->name != CID_NAME_ANY &&
            strncmp(f->name, card->cid.prod_name, ...)) continue;
        if (rev < f->rev_start || rev > f->rev_end) continue;
        ...

        /* 匹配成功 → 调用回调设置怪癖标志 */
        f->vendor_fixup(card, f->data);
    }
}
```

`mmc_blk_fixups` 表（`quirks.h:40`）包含针对特定 eMMC/SD 芯片的硬件 bug 补偿。例如：

```c
static const struct mmc_fixup __maybe_unused mmc_blk_fixups[] = {
    /* SanDisk iNAND: CMD38 参数通过 EXT_CSD[113] 传递 */
    MMC_FIXUP("SEM02G", CID_MANFID_SANDISK, 0x100, add_quirk,
              MMC_QUIRK_INAND_CMD38),
    ...
    END_FIXUP
};
```

`mmc_fixup_device()` 在 CID 维度（制造商+产品名+版本号）匹配，命中后设置 `card->quirks` 标志位。后续 MMC 核心代码在发出特定命令时检查这些标志，采用替代流程绕过硬件 bug。

ATK 板的 eMMC（制造商 ID 通常为 0x15 的 Samsung 或 0x45 的 SanDisk）如果不在 `mmc_blk_fixups` 表中，则跳过。

---

#### ② `card->complete_wq` — 块请求完成工作队列

先想一个问题：**为什么 `mmc_blk_probe()` 需要创建一个工作队列？**

块设备 I/O 的典型路径：

```
应用程序 write(buf)
  ① syscall → VFS → 文件系统 → block layer
  ② mmc 驱动发送 CMD25 (多块写)
  ③ eMMC 卡执行写入，此时 CPU 可以干别的事
  ④ 卡完成后触发中断
  ⑤ 中断处理程序 → mmc_request_done()
  ⑥ 唤醒应用程序 → write() 返回
```

问题在 ⑤→⑥ 之间：**中断处理程序运行在原子上下文中**。在 Linux 中，硬中断处理程序不能调用会睡眠的函数——不能获取信号量、不能调 `wake_up()`、不能调 `complete()`。但 ⑥ 恰恰需要这些操作。

所以需要一个**工作队列**（workqueue）：中断处理程序把"唤醒应用程序"这件事打包成一个 work item，扔到队列里，然后退出中断。内核线程（kworker）会在进程上下文中取出这个 item 执行——此时可以安全地调用 `wake_up()`。

```
中断上下文                  进程上下文 (kworker)
┌──────────────┐          ┌──────────────────┐
│  mmc_request_done()     │  complete_wq 线程  │
│    queue_work(↑) ──────→│    wake_up()      │
│    return IRQ_HANDLED   │    ...            │
└──────────────┘          └──────────────────┘
```

但这里有两个潜在问题，决定了 `alloc_workqueue()` 的两个 flag：

**问题 1：内存回收死锁（`WQ_MEM_RECLAIM`）**

假设系统内存不足，内核要回收内存。回收的一种方式是写回脏页：

```
kswapd (内存回收)
  └─ 写回脏页 → 提交块 I/O → eMMC 开始写
        └─ I/O 完成 → 中断 → queue_work(complete_wq, item)
             └─ 分配内存给 work item  → 但系统正在回收内存！
                  └─ 等内存回收完成 → 等 I/O 完成 → 死锁！
```

`WQ_MEM_RECLAIM` 解决这个死锁：它告诉内核"这个工作队列参与内存回收路径"。内核会为这类队列预留一个"内存回收专用"的 worker 线程，该线程有保留的内存池，即使在内存紧张时也能正常运行。没有这个 flag，上述死锁就会发生。

**问题 2：完成延迟（`WQ_HIGHPRI`）**

工作队列中的 item 由 kworker 线程池执行。默认情况下，kworker 和其他普通线程一样参与调度。如果系统中有大量其他工作项排队，块 I/O 的完成处理可能会被延迟。

`WQ_HIGHPRI` 让这个队列的 work item 以高优先级调度——它们不会被普通工作项堵塞在队列里。这对块设备很重要：应用程序的 `read()`/`write()` 等待 I/O 完成才能返回，延迟直接体现为应用的响应时间。

所以：

```c
card->complete_wq = alloc_workqueue("mmc_complete",
                        WQ_MEM_RECLAIM | WQ_HIGHPRI, 0);
```

| flag | 解决什么问题 |
|------|------------|
| `WQ_MEM_RECLAIM` | 防止内存回收 → 等 I/O → 等 workqueue → 死锁 |
| `WQ_HIGHPRI` | 防止 I/O 完成被其他工作项排队耽误 |

---

#### ⑤ `mmc_blk_add_debugfs()` — 诊断接口

```c
/* block.c:2958 */
static void mmc_blk_add_debugfs(struct mmc_card *card, struct mmc_blk_data *md)
{
    if (!card->debugfs_root)
        return;                             // debugfs 未挂载

    if (mmc_card_mmc(card) || mmc_card_sd(card)) {
        md->status_dentry = debugfs_create_file_unsafe(
                "status", 0400, root, card, &mmc_dbg_card_status_fops);
    }
    if (mmc_card_mmc(card)) {
        md->ext_csd_dentry = debugfs_create_file(
                "ext_csd", S_IRUSR, root, card, &mmc_dbg_ext_csd_fops);
    }
}
```

创建两个 debugfs 节点：

- **`/sys/kernel/debug/mmc1/status`** — 读取时通过 CMD13 获取卡状态寄存器（`SEND_STATUS`），输出 4-bit 状态码 + 就绪/忙标志
- **`/sys/kernel/debug/mmc1/ext_csd`** — 读取时通过 CMD8 重新读取 512 字节 EXT_CSD，以 hex dump 形式输出。可用于运行时检查当前时序模式、擦除计数、寿命估计等

这些接口在调试阶段非常有用——不需要 `mmc-utils`，直接 `cat` 即可查看卡状态。

---

#### ⑥ 运行时电源管理

4 行调用配置了 MMC 卡设备（`card->dev`）的 runtime PM：

```c
pm_runtime_set_autosuspend_delay(&card->dev, 3000);   // ① 3 秒延迟
pm_runtime_use_autosuspend(&card->dev);                // ② 启用自动暂停
pm_runtime_set_active(&card->dev);                     // ③ 初始状态：激活
pm_runtime_enable(&card->dev);                         // ④ 启用 runtime PM
```

执行后，MMC 卡在无 I/O 请求 **3 秒后**自动进入低功耗状态：

```
  最后一次 I/O 完成
        │
        ▼
  计时器开始计时 (3s)
        │
        ▼ 3 秒内无新 I/O
  pm_runtime_put → suspend callback → 卡进入休眠
        │
        ▼ 新 I/O 到来
  pm_runtime_get → resume callback → 卡唤醒 → 执行 I/O → 重启计时器
```

这背后的回调（runtime PM 操作函数）定义在 `mmc_bus_type.pm`（`bus.c:214`）：

```c
static const struct dev_pm_ops mmc_bus_pm_ops = {
    SET_RUNTIME_PM_OPS(mmc_runtime_suspend, mmc_runtime_resume, NULL)
    ...
};
```

`mmc_runtime_suspend` / `mmc_runtime_resume`（`bus.c:197-211`）调用 `host->bus_ops` 中定义的回调，最终由 `mmc_set_ios()` 调整时钟和电压。暂停状态下 eMMC 时钟被关闭，VCCQ 可降低到 1.2V（如果控制器支持）。

---

**小结：** `mmc_blk_probe()` 的 6 个步骤覆盖了从"卡是否符合块设备条件"到"块设备创建"到"运行时电源策略"的完整初始化。其中最核心的数据结构创建在 ③ 和 ④，详见后两节。

### 4.5.4 `mmc_blk_alloc()` → `mmc_blk_alloc_req()` — gendisk 的创建

#### `mmc_blk_alloc()` — 容量的来源

```c
/* block.c:2586 */
static struct mmc_blk_data *mmc_blk_alloc(struct mmc_card *card)
{
    sector_t size;

    if (!mmc_card_sd(card) && mmc_card_blockaddr(card)) {
        size = card->ext_csd.sectors;   // = 30777344 → 14.6 GiB
    } else {
        size = (sector_t)card->csd.capacity << (card->csd.read_blkbits - 9);
    }

    return mmc_blk_alloc_req(card, &card->dev, size, false, NULL,
                             MMC_BLK_DATA_AREA_MAIN, 0);
}
```

这段代码的核心问题是：**eMMC 卡的容量存在两个地方，用哪个？**

条件 `!mmc_card_sd(card) && mmc_card_blockaddr(card)` 决定了走哪条路：

- `!mmc_card_sd(card)` — 不是 SD 卡（即 eMMC 卡）
- `mmc_card_blockaddr(card)` — 使用块寻址（而非字节寻址）

`mmc_card_blockaddr()` 检查 `card->state & MMC_STATE_BLOCKADDR`，这个标志在 `mmc_decode_ext_csd()` 中设置（`mmc.c:416`）：

```c
/* Cards with density > 2GiB are sector addressed */
if (card->ext_csd.sectors > (2u * 1024 * 1024 * 1024) / 512)
    mmc_card_set_blockaddr(card);
```

**容量超过 2 GiB 的卡才使用块寻址**。ATK 板 14.6 GiB 的 eMMC 显然满足。

```
                       容量 > 2GiB？
                      /            \
                     YES            NO
                      │              │
              块寻址 eMMC        SD 卡或小容量卡
                      │              │
                      ▼              ▼
            size = ext_csd.sectors    size = csd.capacity
                                       << (read_blkbits - 9)
                      │              │
                      ▼              ▼
              30777344 扇区        从 CSD 字段计算
```

**路径 1 — 大容量 eMMC（> 2 GiB）：**

```c
size = card->ext_csd.sectors;   // = 30777344
```

`ext_csd.sectors` 来自 EXT_CSD[212:215] **SEC_COUNT** 寄存器，4 字节小端整数，单位是 512 字节扇区：

```c
/* mmc.c:408-413 */
card->ext_csd.sectors =
    ext_csd[EXT_CSD_SEC_CNT + 0] << 0 |
    ext_csd[EXT_CSD_SEC_CNT + 1] << 8 |
    ext_csd[EXT_CSD_SEC_CNT + 2] << 16 |
    ext_csd[EXT_CSD_SEC_CNT + 3] << 24;
```

30777344 扇区 × 512 字节 = 15757928448 字节 ≈ 14.6 GiB。这是最直接的容量来源，不需要任何换算。

**路径 2 — SD 卡或小容量 eMMC（≤ 2 GiB）：**

```c
size = (sector_t)card->csd.capacity << (card->csd.read_blkbits - 9);
```

`card->csd.capacity` 和 `card->csd.read_blkbits` 在 `mmc_decode_csd()` 中从 CSD 寄存器解析（`mmc.c:173`）：

```c
csd->capacity     = (1 + m) << (e + 2);   // C_SIZE + C_SIZE_MULT → 块数
csd->read_blkbits = UNSTUFF_BITS(resp, 80, 4);  // READ_BL_LEN，通常为 10（1024 字节/块）
```

`<< (read_blkbits - 9)` 把块数转换为 512 字节扇区数：对于 `read_blkbits=10`（每块 1024 字节），需要每块折算为 2 个 512 字节扇区，即 `capacity << 1`。

内核源码本身的注释（`block.c:2591-2600`）说得很清楚：

```c
if (!mmc_card_sd(card) && mmc_card_blockaddr(card)) {
    /* The EXT_CSD sector count is in number of 512 byte sectors. */
    size = card->ext_csd.sectors;
} else {
    /* The CSD capacity field is in units of read_blkbits.
     * set_capacity takes units of 512 bytes. */
    size = ...;
}
```

**ATK 板走路径 1**。路径 2 主要是对 SD 卡和历史遗留的小容量 eMMC 的兼容。

`mmc_blk_alloc_req()`（`block.c:2458`）是真正的 gendisk 创建点。它把 `mmc_blk_alloc()` 传进来的扇区数变成用户空间看到的 `/dev/mmcblk1`。

**它在干什么？** 一个块设备在内核中有三个层级：

```
                  应用程序                         用户空间
               ┌──────────┐
               │ /dev/mmcblk1  │  ← 设备节点（通过文件系统访问）
               └─────┬─────┘
             ┌───────┴────────┐
             │   gendisk      │   ← 通用磁盘层（管理分区、IO 调度）
             │ (block_device) │
             └───────┬────────┘
             ┌───────┴────────┐
             │   blk-mq queue │   ← 请求队列（IO 路径的核心）
             │ (request_queue)│
             └───────┬────────┘
                     │
              ┌──────┴──────┐
              │  MMC 子系统  │
              │  (mmc_wait_for_req → CMD) │
```

`mmc_blk_alloc_req()` 负责创建前三层（blk-mq 队列 + gendisk + 设备节点），并把它们串起来。它也负责配置写策略（缓存、FUA、可靠写）。

完整源码如下，然后按执行顺序分 6 步解读：

```c
/* block.c:2458 */
static struct mmc_blk_data *mmc_blk_alloc_req(struct mmc_card *card,
                                              struct device *parent,
                                              sector_t size,
                                              bool default_ro,
                                              const char *subname,
                                              int area_type,
                                              unsigned int part_type)
{
    struct mmc_blk_data *md;
    int devidx, ret;
    bool cache_enabled = false;
    bool fua_enabled = false;

    /* ① 分配全局设备编号 */
    devidx = ida_simple_get(&mmc_blk_ida, 0, max_devices, GFP_KERNEL);
    if (devidx < 0)
        return ERR_PTR(devidx);

    /* ② 分配 mmc_blk_data 结构体 */
    md = kzalloc(sizeof(struct mmc_blk_data), GFP_KERNEL);
    if (!md)
        goto out;

    md->area_type = area_type;
    md->read_only = mmc_blk_readonly(card);

    /* ③ 创建 blk-mq 队列 + gendisk */
    md->disk = mmc_init_queue(&md->queue, card);
    if (IS_ERR(md->disk))
        goto err_kfree;

    INIT_LIST_HEAD(&md->part);
    INIT_LIST_HEAD(&md->rpmbs);
    kref_init(&md->kref);
    md->queue.blkdata = md;
    md->part_type = part_type;

    /* ④ 填充 gendisk 属性 */
    md->disk->major      = MMC_BLOCK_MAJOR;      // 179
    md->disk->first_minor = devidx * perdev_minors;
    md->disk->minors     = perdev_minors;          // 8
    md->disk->fops       = &mmc_bdops;
    md->disk->private_data = md;
    md->parent = parent;
    set_disk_ro(md->disk, md->read_only || default_ro);

    if (area_type & (MMC_BLK_DATA_AREA_RPMB | MMC_BLK_DATA_AREA_BOOT))
        md->disk->flags |= GENHD_FL_NO_PART;

    snprintf(md->disk->disk_name, sizeof(md->disk->disk_name),
             "mmcblk%u%s", card->host->index, subname ? subname : "");

    set_capacity(md->disk, size);                  // 容量 = 扇区数

    /* ⑤ CMD23 可靠写检测 */
    if (mmc_host_cmd23(card->host)) {
        if ((mmc_card_mmc(card) &&
             card->csd.mmca_vsn >= CSD_SPEC_VER_3) || ...)
            md->flags |= MMC_BLK_CMD23;
    }

    if (md->flags & MMC_BLK_CMD23 &&
        ((card->ext_csd.rel_param & EXT_CSD_WR_REL_PARAM_EN) ||
         card->ext_csd.rel_sectors)) {
        md->flags |= MMC_BLK_REL_WR;
        fua_enabled = true;
        cache_enabled = true;
    }
    if (mmc_cache_enabled(card->host))
        cache_enabled = true;
    blk_queue_write_cache(md->queue.queue, cache_enabled, fua_enabled);

    /* ⑥ 块设备节点创建 */
    if (area_type == MMC_BLK_DATA_AREA_MAIN)
        dev_set_drvdata(&card->dev, md);
    ret = device_add_disk(md->parent, md->disk, mmc_disk_attr_groups);
    if (ret)
        goto err_put_disk;

    /* 打印内核日志: "mmcblk1: 29.1 GiB" */
    string_get_size((u64)size, 512, STRING_UNITS_2, cap_str, sizeof(cap_str));
    pr_info("%s: %s %s %s%s\n", md->disk->disk_name, ...);

    return md;

err_put_disk:
    put_disk(md->disk);
    blk_mq_free_tag_set(&md->queue.tag_set);
err_kfree:
    kfree(md);
out:
    ida_simple_remove(&mmc_blk_ida, devidx);
    return ERR_PTR(ret);
}
```

---

#### ① 分配设备编号（devidx）

**为什么需要这一步？** Linux 块设备层使用**主次设备号**（major/minor）唯一标识每个块设备。MMC 块设备使用静态主号 `MMC_BLOCK_MAJOR`（179），但系统可能挂载多个 `mmcblk` 设备（`mmcblk0`、`mmcblk1`、...），需要给每个设备分配不同的次号区间。`devidx` 就是一个全局递增的序号，用来计算次号偏移：

```
                同一个主号 179 下的次号空间
  ┌─────────┬─────────┬─────────┬─────────┬─────────┐
  │mmcblk0  │mmcblk1  │mmcblk2  │mmcblk3  │...      │  ← 每个占 8 个次号
  │0 1...7  │8 9...15 │16...23  │24...31  │         │
  └─────────┴─────────┴─────────┴─────────┴─────────┘
               ↑
          devidx = 1
```

```c
devidx = ida_simple_get(&mmc_blk_ida, 0, max_devices, GFP_KERNEL);
```

`mmc_blk_ida` 是内核 IDA（ID Allocation）分配器，保证并发安全下分配不重复的整数索引。`devidx` 的值决定了第 ③ 步中的 `first_minor = devidx × perdev_minors`。

注意磁盘名 `mmcblk1` 中的 `1` 并非来自 `devidx`——它来自 `card->host->index`，后者是 `mmc_alloc_host()` 中从 `mmc_host_ida` 分配的控制器序号。两个编号来自独立的 IDA 空间，数值上可能巧合相等（比如系统中只有一个 MMC 控制器时），但概念和用途完全不同：`devidx` 决定块设备次号，`host->index` 决定磁盘名前缀。

#### ② `mmc_init_queue()` — blk-mq 请求队列初始化

**为什么需要请求队列？** 应用程序读写文件（`dd if=/dev/mmcblk1`）产生的不是 "CMD17 读扇区 X" 这样的 MMC 命令——经过 VFS → 文件系统 → 块设备层，IO 被封装成 `struct request`。请求队列（request queue）是块设备层的核心抽象，承担三个职责：

1. **合并与排序**：将相邻扇区的多个小 IO 合并成一个，减少 MMC 命令数量
2. **异步提交**：请求提交后不阻塞调用者，完成后通过回调通知
3. **背压（backpressure）**：当队列深度已满时，阻塞上层继续提交，防止 IO 堆积

`mmc_init_queue()` 初始化 **blk-mq**（Block Multi-Queue）——Linux v4.0+ 引入的新一代块层 IO 框架：

```c
md->disk = mmc_init_queue(&md->queue, card);
```

```
mmc_init_queue()
  ├─ blk_mq_alloc_tag_set(&md->queue.tag_set)      ← 分配标签集（跟踪飞行中的请求）
  │    tag_set->ops = &mmc_mq_ops                    ← 块层请求 → MMC 命令的转换层
  │    tag_set->nr_hw_queues = 1                     ← 单硬件队列（MMC 无需多队列）
  │    tag_set->queue_depth = num_tags               ← 队列深度（有 CMDQ 时 = 32）
  ├─ md->queue.queue = blk_mq_init_queue(&tag_set)   ← 创建请求队列实例
  ├─ blk_queue_prep_rq                               ← 注册请求准备回调
  └─ md->disk = alloc_disk(perdev_minors)            ← 分配 gendisk 结构体
```

blk-mq 相比旧版单队列（blk-sq）的核心改进是**标记调度（tag-based scheduling）**：每个飞行中的请求分配一个唯一标记（tag），块层和驱动通过 tag 跟踪完成状态，避免了旧版遍历队列的锁竞争。MMC 只用单队列（`nr_hw_queues = 1`），但受益于 blk-mq 的异步完成和标记管理。

**`mmc_mq_ops`** 是连接块层和 MMC 驱动的桥梁——它将 `blk_mq_ops.queue_rq()` 的通用块请求转化为 `mmc_blk_mq_issue_rq()`，后者根据请求类型（读/写/DISCARD/FLUSH）分发到对应处理函数，最终调用 `mmc_wait_for_req()` 执行 CMD + 数据传输。

注意到 `mmc_init_queue()` 同时做了两件事：创建 blk-mq 队列 **和** 分配 gendisk 结构体（`alloc_disk`）。但此时 gendisk 还是个空壳——属性填充是第 ③ 步的工作。

#### ③ gendisk 属性设置

**为什么需要这一步？** 第 ② 步的 `alloc_disk()` 只分配了 `struct gendisk` 的空壳（一块内存）。但内核还不知道这个磁盘的名字、容量、操作函数。这一步填充 gendisk 的元数据字段，为第 ⑥ 步的 `device_add_disk()` 正式注册做准备。

```c
md->disk->major       = MMC_BLOCK_MAJOR;          // 179
md->disk->first_minor = devidx * perdev_minors;    // 每个磁盘 8 个次设备号
md->disk->minors      = perdev_minors;              // = 8
md->disk->fops        = &mmc_bdops;                // open/release/ioctl
md->disk->private_data = md;                        // 反向引用

snprintf(md->disk->disk_name, sizeof(md->disk->disk_name),
         "mmcblk%u%s", card->host->index, subname ? subname : "");
// → "mmcblk1"（subname=NULL, host->index=1）

set_capacity(md->disk, size);   // 设置块设备容量 = 30777344 扇区

/* 主分区设备 → 注册驱动数据（用于 RPMB ioctl 回溯） */
if (area_type == MMC_BLK_DATA_AREA_MAIN)
    dev_set_drvdata(&card->dev, md);
```

逐字段说明：

| 字段 | 值 | 作用 |
|------|-----|------|
| `major` | `MMC_BLOCK_MAJOR`（179） | MMC 块设备静态主号，所有 `mmcblk*` 共享 |
| `first_minor` | `devidx × 8` | 次号起始偏移，8 为步长 |
| `minors` | `perdev_minors`（8） | 每个磁盘占 8 个次号（磁盘本身 + 最多 6 分区 + 保留） |
| `fops` | `&mmc_bdops` | 块设备操作表 |
| `private_data` | `md` | IO 路径中通过 `disk->private_data` 回溯到 `mmc_blk_data` |
| `disk_name` | `"mmcblk1"` | 命名规则 `mmcblk<host_index>` |
| `capacity` | `set_capacity(disk, size)` | 通告块层容量 = 30777344 扇区 = 14.6 GiB |

对于 boot/RPMB 分区，还设置 `GENHD_FL_NO_PART` 标志避免分区表扫描（它们不是可分区介质）。

`mmc_bdops` 定义了块设备操作表：

```c
/* block.c:862 */
static const struct block_device_operations mmc_bdops = {
    .open        = mmc_blk_open,
    .release     = mmc_blk_release,
    .ioctl       = mmc_blk_ioctl,         // 主设备 ioctl
    .compat_ioctl= mmc_blk_compat_ioctl,  // 32-bit 兼容
    .getgeo      = mmc_blk_getgeo,        // 磁盘几何信息（CHS 参数）
    .owner       = THIS_MODULE,
};
```

其中 `mmc_blk_ioctl` 是控制面入口——`mmc-utils` 等工具通过它发送 RAW 的 MMC 命令（如 CMD8 读 EXT_CSD、CMD13 读状态）。这个 ioctl 走的是块设备主设备号（179），和 4.2 节中注册的字符设备（主号动态分配）是两条独立的控制路径，都可以发 RAW 命令。

#### ④ CMD23 可靠写检测

**为什么需要可靠写？** 多块写（CMD25）期间如果发生电源故障或总线错误，卡可能只写了部分块——静默的数据损坏。传统的多块写流程是 "CMD25 + CMD12（停止）"，没有校验机制。**CMD23（SET_BLOCK_COUNT）** 在 CMD25 之前告诉卡"这次要写 N 块"，卡可以对比实际收到的数据块数是否匹配——不匹配则放弃本次写入。这就是"可靠写"（Reliable Write）的基本思想。

```c
if (mmc_host_cmd23(card->host)) {
    if ((mmc_card_mmc(card) && card->csd.mmca_vsn >= CSD_SPEC_VER_3) || ...)
        md->flags |= MMC_BLK_CMD23;
}
```

两个条件缺一不可：
- `mmc_host_cmd23(card->host)`：host 侧控制器支持 CMD23（`host->caps & MMC_CAP_CMD23`）
- `card->csd.mmca_vsn >= CSD_SPEC_VER_3`：卡侧协议版本 ≥ MMC v4.3（此版本起引入可靠写功能）

如果都满足，设置 `MMC_BLK_CMD23` 标志。这个标志在第 ⑤ 步中用于判断是否启用可靠写和 FUA——注意它只是个前提条件，最终的可靠写启用还依赖 EXT_CSD 中的其他参数。

#### ⑤ 缓存与 FUA — 硬件操作在哪里？

**为什么需要这一步？** 应用层调 `sync()` 或者文件系统写 journal，最终都需要确保数据真正落在了 Flash 上，而不是停留在 eMMC 内部的缓存里。这一步就是告诉块层"这个 eMMC 支持什么写保证策略"，让块层在需要时能发出正确的命令。

先看三个概念在 eMMC 硬件层面的真实操作：

| 概念 | 块层抽象 | eMMC 硬件实际执行 |
|------|---------|-----------------|
| **Write Cache** | `QUEUE_FLAG_WC` | eMMC 内部 SRAM 缓存，CMD6 写 EXT_CSD 字节 33=1 开启 |
| **FLUSH** | `REQ_PREFLUSH` | CMD6 写 EXT_CSD 字节 32（写任意值触发卡内部刷写），eMMC 将缓存刷入 Flash |
| **FUA** | `REQ_FUA` | CMD23 bit 31=1，eMMC 保证数据直写非易失介质 |

```c
if (md->flags & MMC_BLK_CMD23 &&
    ((card->ext_csd.rel_param & EXT_CSD_WR_REL_PARAM_EN) ||
     card->ext_csd.rel_sectors)) {
    md->flags |= MMC_BLK_REL_WR;
    fua_enabled = true;
    cache_enabled = true;
}
if (mmc_cache_enabled(card->host))
    cache_enabled = true;

blk_queue_write_cache(md->queue.queue, cache_enabled, fua_enabled);
```

##### Write Cache 到底在硬件哪里开启的？

eMMC 的写缓存在**卡初始化阶段**（`mmc_init_card()`，详见 4.4.5.8 节）就已经通过 CMD6 写 `EXT_CSD_CACHE_CTRL`（字节 33）= 1 开启了，不在这一步。

第 ⑤ 步中的 `mmc_cache_enabled()` 只是检查 `card->ext_csd.cache_ctrl == 1`——读一下初始化阶段已经设好的标志，**不是在这里开缓存**。

那 FLUSH 写的是哪个？**字节 32（`EXT_CSD_FLUSH_CACHE`）**，和字节 33 是两个独立的寄存器。写任意值到字节 32 触发 eMMC 内部缓存刷写，不改变缓存开关状态。详见下面的 FLUSH 分析。

##### FLUSH 操作 eMMC 缓存吗？

**是的**。应用 `sync()` / `fsync()` → 文件系统发 journal commit → 块层收到 `REQ_OP_FLUSH` → `mmc_blk_issue_flush()`：

```c
/* block.c:1276 */
static void mmc_blk_issue_flush(struct mmc_queue *mq, struct request *req)
{
    ret = mmc_flush_cache(card->host);
    blk_mq_end_request(req, ret ? BLK_STS_IOERR : BLK_STS_OK);
}
```

最终执行 `_mmc_flush_cache()`（`mmc.c:2087`）：

```c
static int _mmc_flush_cache(struct mmc_host *host)
{
    if (_mmc_cache_enabled(host)) {
        err = mmc_switch(host->card, EXT_CSD_CMD_SET_NORMAL,
                         EXT_CSD_FLUSH_CACHE, 1,   // ← 字节 32，值任意
                         CACHE_FLUSH_TIMEOUT_MS);
        // ↑ CMD6 写 EXT_CSD[32]（FLUSH_CACHE），卡收到后刷内部缓存
    }
    return err;
}
```

**FLUSH 的硬件操作就是 CMD6**，写 EXT_CSD[32]（FLUSH_CACHE 寄存器，写任意值触发刷写）。eMMC 收到后启动内部 FSM，把 SRAM 缓存中的脏数据写入 Flash 阵列。

##### sync() 到底做了什么？

很多人以为 `sync()` 就是"把 DDR 里的数据刷到 eMMC Flash 上"，这是错觉。`sync()` 实际做了**两件事**，数据经历了**两级缓存**：

```
                         DDR                        eMMC 芯片内部
应用程序 write(fd, buf)
    │
    ▼  ① 数据复制到 page cache，write 立即返回
 ┌──────────┐
 │ page cache│  ← DDR 中的文件缓存页（脏页）
 └─────┬────┘
       │  ② writeback（sync 前半段）
       │  ⚡ DMA 传输
       ▼
 ┌──────────┐
 │ SRAM Cache│  ← eMMC 芯片内部的硬件写缓存
 └─────┬────┘
       │  ③ FLUSH（sync 后半段）
       │  CMD6 写 EXT_CSD[32]
       ▼
 ┌──────────┐
 │ Flash 阵列│  ← 真正持久化存储
 └──────────┘
```

**第一级：DDR 中的 page cache。** 应用程序调 `write()` 时，数据先被复制到内核的 page cache（DDR 中），`write` 立即返回。此时如果掉电，数据丢失。`sync()` 的前半段通过 writeback 机制把这些脏页提交到块设备层，经 DMA 传输到 eMMC 内部的 SRAM 写缓存。

**第二级：eMMC 内部的 SRAM 写缓存。** 数据到达这里时，对 `sync()` 来说"写入"已经完成了，但 eMMC 还没把数据搬到 Flash 介质上。此时掉电同样丢数据。`sync()` 的后半段通过 FLUSH 命令（CMD6 FLUSH_CACHE）让 eMMC 把 SRAM 中的数据刷入 Flash 阵列。

整个过程 `sync()` 不会刷"所有 DDR 内存"——匿名页（堆、栈、匿名 mmap）不属于任何文件，page cache 不涉及它们。`sync()` 只提交文件对应的脏 page cache，然后发 FLUSH。

回到 `fsync(fd)`：它比 `sync()` 多一步——先通过 `filemap_fdatawrite()` 把指定文件的脏 page cache 提交到块层，再调 `blkdev_issue_flush()` 发 FLUSH。区别在于 `sync()` 全局，`fsync()` 只搞一个文件。

##### FUA 在 eMMC 上怎么实现的？

FUA 不靠 FLUSH，靠 **CMD23 可信写（Reliable Write）**。写路径中（`mmc_blk_mq_issue_rw_rq`，`block.c:1709`）：

```c
brq->sbc.opcode = MMC_SET_BLOCK_COUNT;        // CMD23
brq->sbc.arg    = brq->data.blocks |
                  (do_rel_wr ? (1 << 31) : 0); // bit 31 = 可靠写
brq->mrq.sbc = &brq->sbc;
```

其中 `do_rel_wr`（`block.c:1377`）：

```c
do_rel_wr = (req->cmd_flags & REQ_FUA) &&
            rq_data_dir(req) == WRITE &&
            (md->flags & MMC_BLK_REL_WR);
```

FUA 完整链路：

```
REQ_FUA → do_rel_wr → CMD23 bit 31=1 →
CMD25(多块写) 前卡收到 CMD23 →
卡保证所有数据块在返回前已写入非易失存储

注意 CMD23 是**一次性**的——它只对紧随其后的那一次 CMD25 生效。eMMC 协议中，CMD23 设置的块数和可靠写位（bit 31）在每次数据传输命令完成后自动清除，不会跨命令保持。这就是为什么代码中每个写请求都独立判断 `do_rel_wr` 并重新设置 CMD23 参数。没有 CMD23 的 CMD25 就是普通多块写，不会继承上一次的可靠写状态。
```

##### 总结：数据完整性链路

```
应用层               块层                          MMC 硬件
sync()/fsync()
  ↓
REQ_OP_FLUSH ─────> mmc_blk_issue_flush()
                         ↓
                    mmc_switch(CMD6, FLUSH_CACHE=0)
                         ↓
                    eMMC 刷内部缓存 → Flash 阵列

REQ_FUA
  ↓
do_rel_wr ─────────> CMD23 bit 31 = 1 → CMD25
                         ↓
                    eMMC 保证直写非易失介质
```

`blk_queue_write_cache()` 向块层注册 `QUEUE_FLAG_WC` 和 `QUEUE_FLAG_FUA`。文件系统在挂载时读取这些标志，在 journal commit 等关键路径上自动附加 `REQ_PREFLUSH` 或 `REQ_FUA`——从应用 `sync()` 到 eMMC Flash 阵列的整条数据完整性链路至此打通。


#### ⑥ `device_add_disk()` — 块设备节点创建

**为什么最后一步是 `device_add_disk()`？** 前面五步（①~⑤）创建了所有内核数据结构——IDA 编号、blk-mq 请求队列、gendisk 属性填充——这些都藏在内核内存里，用户空间看不到。没有这个调用，就没有 `/dev/mmcblk1`，没有 `/sys/block/mmcblk1/`，udev 不会收到事件，文件系统无法挂载。`device_add_disk()` 是"把磁盘暴露给用户空间"的最后一步。

```c
ret = device_add_disk(md->parent, md->disk, mmc_disk_attr_groups);
```

`device_add_disk()`（`genhd.c:394`）内部按以下步骤执行（先看完整时序概览，再分步解读）：
##### 完整时序

```
device_add_disk(mmclbk1)
  │
  ├─ elevator_init_mq()          ← IO 调度器初始化
  ├─ dev_set_uevent_suppress(1)  ← 抑制 uevent
  ├─ device_add(ddev)            ← /sys/block/mmcblk1/ 出现
  ├─ blk_register_queue()        ← /sys/.../queue/ 出现
  ├─ bdev_add(part0, 179:0)      ← 块设备层注册
  ├─ disk_scan_partitions()      ← 读 MBR/GPT，注册分区
  ├─ dev_set_uevent_suppress(0)  ← 释放 uevent
  ├─ disk_uevent(KOBJ_ADD)       ← udev → /dev/mmcblk1 创建
  │
  ▼
用户空间看到 /dev/mmcblk1 和 /dev/mmcblk1p1...
```

**`elevator_init_mq()` — IO 调度器初始化**

为 blk-mq 队列选择一个默认 IO 调度器（如 mq-deadline）。对 eMMC 这类非机械介质效果有限，但保留框架允许用户切换：

```bash
echo none > /sys/block/mmcblk1/queue/scheduler
```

**主次设备号校验**

MMC 使用静态主号 `MMC_BLOCK_MAJOR=179`，校验第 ③ 步设置的 `first_minor` 和 `minors` 是否越界（次号空间 0-255，每个磁盘 8 个）。

**抑制 uevent + `device_add()`**

```c
dev_set_uevent_suppress(ddev, 1);     // 先抑制 uevent

ddev->parent  = parent;               // → card->dev
ddev->groups  = groups;               // → mmc_disk_attr_groups
dev_set_name(ddev, "%s", disk->disk_name);  // → "mmcblk1"
ddev->devt    = MKDEV(179, first_minor);    // → dev_t(179, 0)

ret = device_add(ddev);               // 注册到驱动模型
```

设计意图：**先抑制 uevent（不让 udev 看见），再创建设备目录**。`device_add()` 创建 `/sys/devices/virtual/block/mmcblk1/` 和 `/sys/block/mmcblk1/` symlink，但由于 uevent 被抑制，udev 不会收到 `KOBJ_ADD`，`/dev/` 下还没有节点。

**`blk_register_queue()` — 请求队列注册**

在 `/sys/block/mmcblk1/queue/` 下暴露 IO 调度器、写缓存策略、IO 统计等参数。第 ⑤ 步设置的 `blk_queue_write_cache()` 在此刻生效——用户可通过 `write_cache` 文件看到当前策略是 "write back" 还是 "write through"。

**`bdev_add()` — 块设备层注册**

```c
bdev_add(disk->part0, ddev->devt);
```

将 `disk->part0` 关联到 `dev_t(179, 0)`，加入块设备 inode 哈希表。此后内核可通过 `dev_t` 找到该块设备——但用户空间的 `/dev/` 节点还未出现（uevent 仍被抑制）。

**分区扫描**

```c
if (get_capacity(disk))
    disk_scan_partitions(disk, BLK_OPEN_READ);
```

读取 0 号扇区的 MBR/GPT。如果找到有效分区：

- 为每个分区分配次设备号（`179:1`、`179:2`、...）
- 创建分区的 `block_device`，调 `bdev_add()` 注册到块设备层

如果 eMMC 是空白的（无 MBR/GPT），分区扫描 0 个分区，直接跳过。

**释放 uevent — `/dev/` 节点创建**

```c
dev_set_uevent_suppress(ddev, 0);   // 释放抑制
disk_uevent(disk, KOBJ_ADD);        // 发出 ADD 事件
```

udev 收到 `KOBJ_ADD`：

```
→ mknod /dev/mmcblk1 b 179 0
→ 对所有已扫描的分区分别发 KOBJ_ADD
→ /dev/mmcblk1p1, /dev/mmcblk1p2 ...
```

**关键时序：分区扫描在 uevent 之前完成。** 保证 udev 看到设备时，分区已就绪，不会出现"设备有了但分区还在创建"的竞态。

**收尾**

```c
disk_update_readahead(disk);      // 设置预读大小
disk_add_events(disk);            // 启用事件轮询
set_bit(GD_ADDED, &disk->state);  // 标记已添加
```

##### 关于 `mmc_disk_attr_groups`

```c
/* block.c:2952 */
static const struct attribute_group *mmc_disk_attr_groups[] = {
    &mmc_disk_attr_group,   // force_ro
    NULL,
};
```

这些属性通过上面第 ③ 步的 `ddev->groups = groups` 传给 `device_add()`，sysfs 框架自动添加。最终出现在：

```
/sys/block/mmcblk1/force_ro
/sys/block/mmcblk1boot0/force_ro
```

典型场景——更新 bootloader：

```bash
echo 0 > /sys/block/mmcblk1boot0/force_ro
dd if=u-boot.stm32 of=/dev/mmcblk1boot0 bs=1M
```

需要 eMMC 的 `boot_ro_lock` 未被锁定（一次性熔丝位，锁定后无法逆转）。


此时内核日志出现：

```
mmcblk1: mmc1:0001 H28U74301DMFPR-EG 14.6 GiB
mmcblk1: p1  p2
```

这段日志来自 `mmc_blk_alloc_req()` 的最后几行（`block.c:4196`），打印磁盘名、CID 厂商信息、容量和分区列表——标志整个 eMMC 初始化流程完成，用户空间可通过 `/dev/mmcblk1` 正常访问 eMMC。


### 4.5.5 多硬件分区的生成 — `mmc_blk_alloc_parts()`

eMMC 内部有多个硬件分区（boot0、boot1、RPMB），它们**不是通过 MBR/GPT 分区表定义的**，而是 eMMC 芯片物理上就有独立的内存阵列。内核通过 `mmc_decode_ext_csd()` 从 EXT_CSD 中读出分区信息，在 `mmc_part_add()` 中填充 `card->part[]` 数组。

分区信息的填充发生在 `mmc_decode_ext_csd()`（`mmc.c:370`）中，按类型分三步：

```c
/* mmc.c:455 — Boot 分区 */
if (ext_csd[EXT_CSD_BOOT_MULT] && mmc_boot_partition_access(card->host)) {
    for (idx = 0; idx < MMC_NUM_BOOT_PARTITION; idx++) {      // 2 个
        part_size = ext_csd[EXT_CSD_BOOT_MULT] << 17;          // × 128KB
        mmc_part_add(card, part_size,
                     EXT_CSD_PART_CONFIG_ACC_BOOT0 + idx,      // 0x1, 0x2
                     "boot%d", idx, true,                       // 只读
                     MMC_BLK_DATA_AREA_BOOT);
    }
}

/* mmc.c:574 — RPMB 分区 */
if (ext_csd[EXT_CSD_RPMB_MULT] && mmc_host_cmd23(card->host)) {
    mmc_part_add(card, ext_csd[EXT_CSD_RPMB_MULT] << 17,       // × 128KB
                 EXT_CSD_PART_CONFIG_ACC_RPMB,                  // 0x3
                 "rpmb", 0, false,
                 MMC_BLK_DATA_AREA_RPMB);
}

/* mmc.c:339 — GP（通用目的）分区（可能有 4 个，ATK 板无） */
mmc_manage_gp_partitions(card, ext_csd);
```

`mmc_part_add()` 将信息存到 `card->part[]`（`mmc.c:309`）：

```c
static void mmc_part_add(struct mmc_card *card, u64 size,
                         unsigned int part_cfg, char *name, int idx,
                         bool ro, int area_type)
{
    card->part[card->nr_parts].size = size;         // 字节数
    card->part[card->nr_parts].part_cfg = part_cfg; // 分区配置码
    sprintf(card->part[card->nr_parts].name, name, idx);
    card->part[card->nr_parts].force_ro = ro;        // boot 分区只读
    card->part[card->nr_parts].area_type = area_type;
    card->nr_parts++;
}
```

`mmc_blk_alloc_parts()`（`block.c:2785`）遍历这个数组，对每个分区创建一个 gendisk：

```c
static int mmc_blk_alloc_parts(struct mmc_card *card, struct mmc_blk_data *md)
{
    for (idx = 0; idx < card->nr_parts; idx++) {
        if (card->part[idx].area_type & MMC_BLK_DATA_AREA_RPMB) {
            /* RPMB：特殊处理，没有块队列 */
            ret = mmc_blk_alloc_rpmb_part(card, md, ...);
        } else if (card->part[idx].size) {
            /* boot/GP：标准 gendisk 创建 */
            ret = mmc_blk_alloc_part(card, md, ...);
        }
    }
}
```

每个硬件分区对应一个独立的 `mmc_blk_alloc_req()` 调用，生成独立的 gendisk：

| 硬件分区 | 磁盘名称 | 设备号 | area_type | 设备类型 | 访问方式 |
|---------|---------|--------|-----------|---------|---------|
| 用户数据区 | `mmcblk1` | 179:0-7 | `MAIN` | 块设备 | 常规块读写 |
| Boot 0 | `mmcblk1boot0` | 179:8-15 | `BOOT` | 块设备 | 块读写（默认只读） |
| Boot 1 | `mmcblk1boot1` | 179:16-23 | `BOOT` | 块设备 | 块读写（默认只读） |
| RPMB | `mmcblk1rpmb` | 独立字符设备号 | `RPMB` | **字符设备** | ioctl 专用 |

三个关键设计差异：

**1) Boot 分区默认只读：** `mmc_part_add` 时传 `force_ro = true`，`mmc_blk_alloc_req()` 中：

```c
md->read_only = mmc_blk_readonly(card);   // 检查硬件写保护引脚
set_disk_ro(md->disk, md->read_only || default_ro);
```

需要写入 boot 分区时，用户必须先 `echo 0 > /sys/block/mmcblk1boot0/force_ro`——这需要 eMMC 自身支持 boot 分区写（`boot_ro_lock` 未锁定）。

**2) Boot 和 RPMB 分区不扫描分区表：**

```c
if (area_type & (MMC_BLK_DATA_AREA_RPMB | MMC_BLK_DATA_AREA_BOOT))
    md->disk->flags |= GENHD_FL_NO_PART;
```

这些分区上不会出现 `p1/p2` 子分区。只有主分区（`MMC_BLK_DATA_AREA_MAIN`）会参与分区表扫描。

**3) RPMB 没有块队列：** `mmc_blk_alloc_rpmb_part()` 创建的不是完整 gendisk，而是一个字符设备接口：

```c
/* block.c:2643 */
static int mmc_blk_alloc_rpmb_part(...)
{
    /* RPMB 分区分配字符设备号 */
    ret = alloc_chrdev_region(&rpmb->chrdev, 0, 1, ...);
    cdev_init(&rpmb->cdev, &mmc_rpmb_file_operations);
    cdev_add(&rpmb->cdev, rpmb->chrdev, 1);

    /* 创建 /sys/block/mmcblk1rpmb/ 目录 */
    rpmb->dev = device_create(mmc_rpmb_class, parent_dev, rpmb->chrdev, ...);
}
```

所以虽然 `lsblk` 显示 `mmcblk1rpmb` 像是块设备，它的 `open`/`read`/`write` 操作函数 `mmc_rpmb_file_operations` 中，`read`/`write` 返回 `-EINVAL`——只能通过 ioctl 访问。RPMB 的每次读写都带上 HMAC（Keyed-Hash Message Authentication Code），密钥在 eMMC 芯片生产时烧录，用于安全启动和防回滚。

### 4.5.6 数据流回溯

从 EXT_CSD 的原始字节到 `/dev/mmcblk1` 的完整数据流：

```
EXT_CSD[212-215] sector count (4 bytes)
    │
    ▼
mmc_decode_ext_csd()
    └─ card->ext_csd.sectors = 30777344
    └─ card->ext_csd.boot_size_mult = 32  → 分区条目 "boot0"/"boot1"
    └─ card->ext_csd.rpmb_size_mult = 32  → 分区条目 "rpmb"
    ▼
mmc_blk_alloc()
    └─ size = card->ext_csd.sectors
    ▼
mmc_blk_alloc_req()
    ├─ devidx = ida_simple_get(...)
    ├─ md->disk = mmc_init_queue(...)     ← blk-mq 队列
    ├─ md->disk->major = MMC_BLOCK_MAJOR (179)
    ├─ md->disk->first_minor = devidx × 8
    ├─ md->disk->disk_name = "mmcblk1"
    ├─ set_capacity(disk, 30777344)
    └─ device_add_disk(...)
         └─ /dev/mmcblk1  ← 这就是 ls 看到的设备节点
    ▼
mmc_blk_alloc_parts()
    ├─ mmc_blk_alloc_req(... "boot0" ...)  → /dev/mmcblk1boot0
    ├─ mmc_blk_alloc_req(... "boot1" ...)  → /dev/mmcblk1boot1
    └─ mmc_blk_alloc_rpmb_part(...)        → /dev/mmcblk1rpmb
```

### 4.5.7 为什么有 4 个设备而不是 1 个

回到 4.1 节的问题：`ls /dev/mmcblk*` 显示 4 个设备。现在可以完整回答：

```
                     ┌──────────────────────┐
                     │  mmc_init_card()     │
                     │  CMD 序列完成         │
                     │  DDR52 @ 52MHz, 8-bit │
                     └──────────┬───────────┘
                                │ mmc_add_card() → __device_attach
                                ▼
                     ┌──────────────────────┐
                     │  mmc_blk_probe()     │
                     │  mmc_blk_alloc()     │
                     │    → mmcblk1        │  ← EXT_CSD.sectors = 30777344
                     │  mmc_blk_alloc_parts()│
                     │    → mmcblk1boot0   │  ← EXT_CSD.boot_size_mult × 128K
                     │    → mmcblk1boot1   │  ← 同上
                     │    → mmcblk1rpmb    │  ← EXT_CSD.rpmb_size_mult × 128K
                     └──────────────────────┘
```

| 设备 | 大小 | 来自 | 用途 |
|------|------|------|------|
| `/dev/mmcblk1` | 14.6 GiB | `EXT_CSD[212:215]` sectors | 用户数据（rootfs、APP） |
| `/dev/mmcblk1boot0` | 4 MiB | `EXT_CSD[226]` boot_size_mult=32 | 引导程序（FSBL、SSBL） |
| `/dev/mmcblk1boot1` | 4 MiB | `EXT_CSD[226]` boot_size_mult=32 | 引导备份 |
| `/dev/mmcblk1rpmb` | 4 MiB | `EXT_CSD[168]` rpmb_size_mult=32 | 安全存储（HMAC 认证） |

eMMC 的硬件分区设计是为了满足**安全启动和防回滚**的需求：boot 分区存储不可篡改的引导加载程序，RPMB 存储密钥和计数器。普通文件系统写入不影响这些分区。

---

## 4.6 总结

### 四条架构主线回顾

本文追踪了从系统启动到 `/dev/mmcblk1` 就绪的完整路径，涉及四条层次分明的主线：

**阶段一 — Core 层框架注册**（`mmc_init()`）

注册 `mmc_bus_type` 和 `mmc_host_class`，在 sysfs 中创建骨架。此时没有设备也没有驱动，只是为后续的设备注册准备好总线和类。

**阶段二 — AMBA 总线匹配控制器**（`mmci_probe()`）

DTS 节点 `compatible = "arm,primecell"` 触发 AMBA 总线匹配，`mmci-pl18x` 驱动被 probe。`mmc_alloc_host` 一次性分配 `mmc_host + mmc_priv` 两个结构体，`mmc_add_host` 将控制权移交给 core 层。这个阶段产生的是**控制器**（`mmc_host`），不是卡设备。

**阶段三 — Core 层卡探测**（`mmc_rescan` → `mmc_init_card`）

没有驱动参与，全是 core 层代码通过 CMD 序列与 eMMC 芯片交互。核心成果：
- CMD1 轮询确定卡就绪
- CMD2/CMD3/CMD9 完成识别和 CSD 读取
- CMD7 将卡切入 Transfer State
- CMD8 读取 512 字节 EXT_CSD，填充 `mmc_card` 的全部能力信息
- CMD6 序列完成时序协商：ATK 板走 **DDR52 @ 52MHz**（DTS 无 HS200/HS400 声明）

**阶段四 — MMC 总线匹配块设备**（`mmc_blk_probe()`）

`mmc_add_card` 将 `mmc_card` 注册到 `mmc_bus_type` 总线，`mmc_block` 驱动匹配成功后被 probe。`mmc_blk_alloc_req` 创建 gendisk 并通过 `device_add_disk` 生成 `/dev/mmcblk1`。`mmc_blk_alloc_parts` 根据 EXT_CSD 中的分区大小创建 boot0/boot1/rpmb。
