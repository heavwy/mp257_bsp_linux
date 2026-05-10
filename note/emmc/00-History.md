# 00. eMMC/MMC 子系统 30 年演进史

> 本文是 [STM32MP257 eMMC 驱动深度分析](README.md) 系列的第 0 篇。  
> MMC 子系统是 Linux 内核中最古老的块设备子系统之一，同时也是"胶水代码"密度最高的子系统之一。  
> 它要同时支持 eMMC、SD、SDIO、MMC 卡四种设备、三种总线模式（SPI/1-bit/4-bit/8-bit）、五档电压、九种时序模式。这背后的每一个 if/else，都对应一段历史。
>
> **字数：** 约 9,400 字  
> **建议阅读时间：** 30–40 分钟（含代码和图表）
>
> **理解历史 = 理解为什么代码长这样。**

---

## 一、1990 年代的存储格局（Pre-MMC）

在 1997 年 MMC 问世之前，便携式设备的存储选择是这样的：

```
软盘 (FDD)
  优点: 几乎每台 PC 都有
  缺点: 1.44MB、慢、机械结构脆弱、耗电大、100mm×100mm 没法放进手机

CompactFlash (CF)
  优点: 速度快、容量大（当时可达 256MB）、IDE 协议兼容
  缺点: 42 个引脚、50mm×43mm、成本高、功耗大
  典型设备: 早期数码单反（Nikon D1, Canon EOS D30）

SmartMedia (SSFDC)
  优点: 引脚少（22 pin）、薄、东芝主推
  缺点: 控制器在主机侧、兼容性差、最大只到 128MB
  典型设备: 数码相机（Olympus, Fuji）

PCMCIA
  优点: 接口标准、能外接网卡/硬盘
  缺点: 85mm×54mm（和信用卡一样大）、功耗高、贵
  典型设备: 笔记本扩展卡
```

**关键矛盾：手机和 PDA 市场在爆炸式增长，但没有一个"足够小、足够省电、足够便宜"的存储卡标准。** 诺基亚 6110（1997）和 Palm Pilot（1996）只能靠内置内存，用户无法扩展存储。

这时的 Linux：v2.0 系列，基本没有可插拔存储设备的概念。软驱用 floppy 驱动，硬盘用 IDE 驱动，没有"热插拔"这个机制。

---

## 二、1997–1999: MMC 诞生与第一版 Linux 驱动

### MMC 标准诞生

**1997 年 11 月**，西门子（Siemens）和 SanDisk 联合发布了 MMC（MultiMediaCard）标准 v1.0。

技术上的关键设计决策：

```
物理接口: 7 个引脚（电源 2 + 地 2 + 时钟 1 + 命令 1 + 数据 1）
  对比: CF 卡 42 针，PCMCIA 68 针 → MMC 是当时引脚最少的存储卡

尺寸: 24mm × 32mm × 1.4mm
  对比: CF 卡 43mm × 36mm × 3.3mm → MMC 厚度不到 CF 的一半

重量: <2 克
  对比: CF 卡 ~12 克

协议: 兼容 SPI 模式
  这意味着任何有 SPI 接口的 MCU 都可以读写 MMC 卡
  → 这是它被嵌入式系统广泛采用的关键原因

功耗: 工作 <100mW，休眠 <50μA
  对比: 微型硬盘（Microdrive，IBM 的同类产品）~2W

容量: 最初 2MB / 4MB / 8MB，很快到 64MB

数据线: 1-bit（只有 1 根数据线 DAT0）
  对比: CF 卡 16-bit PIO，PCMCIA 16/32-bit → MMC 是串行总线

时钟频率: 0-20MHz
  对比: CF 卡最多 16.6MB/s (PIO Mode 4) → MMC 每个时钟传 1 bit

传输速率: ~2.5MB/s (20MHz x 1-bit)
  对比: 软盘 ~0.06MB/s，CF ~16.6MB/s → MMC 比软盘快，但远慢于 CF

工作电压: 3.3V (2.7-3.6V)
  兼容 1.8V 的逻辑电平需要 10 年后才出现 → 此时只有 3.3V
```

**1998 年初**，MMC 协会成立（由西门子、SanDisk、诺基亚、摩托罗拉等 14 家公司组成），MMC v2.0 发布，增加了写保护、多段写入等功能。

### SD 卡横空出世

**1999 年 8 月**，SanDisk、松下（Panasonic）和东芝（Toshiba）联合发布了 SD（Secure Digital）卡标准。

SD 和 MMC 的关键区别：

| | MMC | SD | 后果 |
|--|-----|----|------|
| **引脚** | 7 pin | 9 pin（多 2 个引脚，4 条数据线） | SD 插槽能插 MMC，但反过来不行 |
| **加密** | 无 | CPRM（Content Protection for Recordable Media） | 音乐/视频行业愿意用 SD |
| **时钟** | 0-20MHz | 0-25MHz | SD 时钟频率更高 |
| **位宽** | 1-bit | 4-bit | SD 并行传 4 倍数据 |
| **速率** | ~2.5MB/s (20MHz 1-bit) | ~12.5MB/s (25MHz 4-bit) | SD 快 5 倍 |
| **电压** | 3.3V only (2.7-3.6V) | 3.3V only (2.7-3.6V) | 两者一致，1.8V 未出现 |
| **许可** | 开放标准 | 需要授权费 | SD 协会收钱，限制开源实现 |

**最让内核开发者头疼的决策：** SD 卡的 SPI 模式下用了和 MMC 不同的命令集。这意味着一个驱动要处理两套初始化序列，而且没法在 SPI 模式下通过命令区分你插的是 MMC 还是 SD——必须试完一套再试另一套。

### Linux 第一个 MMC 驱动

**1999 年 12 月，Linux v2.2.13**。David Brownell 提交了内核第一个 MMC 驱动 `drivers/block/mmc.c`。

这个驱动在当时属于"最小可工作产品"：

```c
// 1999 年 mmc.c 的核心逻辑（伪代码）
static int mmc_read_sector(struct mmc_device *dev, u32 sector, u8 *buf)
{
    // 只有 SPI 模式
    spi_write(dev->spi, READ_SINGLE_BLOCK_CMD, 4);
    spi_read(dev->spi, buf, 512);          // 一次只能读 1 个 sector
    return 0;                               // 没有错误处理
}
```

当时的状态：
- **只支持 SPI 模式**——因为 SPI 子系统和 David Brownell 本人都在 1999 年刚被合入
- **只读，不支持写**——写操作涉及块擦除和 FTL，比读复杂得多
- **固定 512 字节 sector**——硬编码
- **一次一个 sector**——队列？不存在的
- **没有热插拔**——要么 init 时检测到，要么重启后插
- **代码量**：~1800 行 C

这个驱动服务的使用场景：嵌入式 Linux 设备从 MMC 卡启动。（实际上，这也是今天很多开发板仍然在使用的方式——从 SD 卡启动。）

### 同时期的其他存储技术

| 技术 | 诞生 | 与 MMC/SD 的关系 |
|------|------|----------------|
| USB 1.0 | 1996 | 后来 USB 读卡器成了 PC 访问 SD/MMC 的主要方式 |
| USB 大容量存储类 | 2000 | 读卡器 + 存储卡 vs U 盘——存储卡市场比 U 盘大得多 |
| ATA Flash | 1994 | CF 卡使用 ATA 协议，这是 MMC/SD 采用 SPI 模式的原因之一 |

**历史的有趣之处：** MMC 最初的设计目标是"小型化存储"，但它后来发展出的 eMMC 标准反而回到了"焊在板子上"——绕了一圈，从可插拔回到固定设备，但总线接口比原始 SPI 快了 1000 倍。

---

## 三、2001–2005: SD 的大爆发、第一个"四设备困境"、Pierre Ossman 时代

### SD 卡来了，驱动跟不上

回顾第 2 节：SD 卡标准 1999 年就已发布，引脚比 MMC 多 2 个（9 vs 7），通信协议从一开始就不一样。1999 年的 Linux 驱动（David Brownell 的 mmc.c）只支持 SPI 模式下的 MMC 卡，压根不认识 SD。

问题在 2001–2002 年开始爆发：SD 卡大规模上市，用户买了数码相机送的 SD 卡，插到 Linux 设备的 MMC 插槽上——**内核不识别**。硬件上是兼容的（SD 插槽能插 MMC，反之亦然），但软件层面的命令集完全不同。

SD 卡在 SPI 模式下使用自己的一套命令：
- MMC 用 `CMD1`（SEND_OP_COND）协商电压和容量
- SD 用 `ACMD41`（SD_SEND_OP_COND）做同样的事，参数格式不同

于是驱动不得不变成"试错法"——先按 MMC 试，不行再按 SD 试：

```c
/* 2002 年驱动的初始化逻辑（伪代码）—— 两轮试探 */
static int mmc_detect_card(struct mmc_device *dev)
{
    /* 第一轮: 按 MMC 协议 */
    mmc_send_cmd(dev, CMD0, 0);
    mmc_send_cmd(dev, CMD1, 0x40100000);     // MMC 的 SEND_OP_COND
    if (mmc_rsp(dev) & MMC_CARD_BUSY) {
        dev->type = MMC_TYPE_MMC;
        goto init_done;
    }
    
    /* 第二轮: 按 SD 协议 */
    mmc_send_cmd(dev, CMD0, 0);              // 重新复位
    mmc_send_cmd(dev, ACMD41, 0x40100000);   // SD 的 APP_SEND_OP_COND
    if (mmc_rsp(dev) & MMC_CARD_BUSY) {
        dev->type = MMC_TYPE_SD;
        goto init_done;
    }
    
    dev->type = MMC_TYPE_UNKNOWN;
    return -ENODEV;
}
```

这种"多轮试探"模式至今仍在 `drivers/mmc/core/mmc_ops.c` 中使用——不是不想改，是硬件上没法通过电气信号区分卡的类型，只能靠发命令试。

### 2002: SDIO 来了——不存数据的 SD 卡

**2002 年**，SD 卡协会发布了 SDIO 标准，允许 SD 插槽连接非存储外设：WiFi 卡（最典型）、蓝牙、GPS、摄像头。**SDIO 设备走同样的 SD 总线协议，传的不是数据块，而是 I/O 控制命令。**

这对 Linux MMC 驱动意味着灾难性的复杂度增长：
- 一个总线驱动要同时处理**存储设备**（MMC/SD）和**功能设备**（SDIO）
- SDIO 设备用 `CMD5`（IO_SEND_OP_COND）初始化，和前两个都不一样
- 于是试探逻辑变成三轮而不是两轮
- 运行时还要区分 `card->type == MMC_TYPE_SDIO` 走不同的 I/O 路径

2005 年 Linux 加入 `CONFIG_MMC_SDIO` 配置选项，SDIO 支持成为可选。今天你在 `drivers/mmc/core/` 里看到的 `if (card->type == MMC_TYPE_SDIO)` 分散在各个函数中——每个判断都对应着一类设备的特殊行为。

### 2004-2005: 高速 SD 与双电压的出现

2004 年，SD 协会发布了 SD High-Speed 标准（后称 SDHS），这是 SD 卡第一次重大速度升级：

```
SD 标准演进（截至 2005）:
  SD 1.0 (1999):   25MHz 4-bit → 12.5MB/s, 3.3V only
  SD 1.1 (2004):   50MHz 4-bit → 25MB/s, 引入双电压 (3.3V/1.8V)
                    ↑ 时钟翻倍，速率翻倍

  MMC 同期规格:
  MMC v2.0 (1998): 20MHz 1-bit → 2.5MB/s, 3.3V only
  MMC v3.0 (2001): 20MHz 4-bit → 10MB/s, 3.3V only
  MMC v4.0 (2005): 26MHz 4-bit/8-bit → 26MB/s (8-bit), 3.3V only
                    ↑ eMMC 的前身
```

**双电压切换机制**是这个时期一个重要的技术细节。之前所有卡都是 3.3V I/O，但高速信号在 3.3V 下信号完整性变差（摆幅大、上升沿慢）。SD 协会的解决方案：

```
双电压切换流程 (CMD0 + CMD1):
  ① 主机发 CMD0 带 arg=0xFFFFFF  → 进入"电压切换"模式
  ② 卡将 DAT 线拉低表示就绪
  ③ 主机将 vqmmc 供电从 3.3V 切到 1.8V
  ④ 卡检测电压变化，切换内部 I/O 电平
  ⑤ 主机发 CMD1 确认切换完成

  失败处理:
  如果卡不响应或切换失败 → 主机恢复 3.3V，回退到低速模式
  这个回退逻辑至今仍在 drivers/mmc/core/core.c 中
```

这个机制的遗留影响：今天你在 DTS 中看到的 `vqmmc-supply` 就是管 I/O 电压切换的。eMMC 的 HS200/HS400 模式必须用 1.8V I/O，`vqmmc` 稳压器必须支持动态切换。

### Pierre Ossman 接管与写支持

**2004 年**，Pierre Ossman（又名 pjrm）接管了 MMC 子系统的维护。他是当时社区中少数真正理解 MMC 协议的人之一。

Pierre 面临的现状：内核 mmc 驱动自 1999 年以来只有读功能。嵌入式用户要的是**在 Linux 上往 SD 卡写数据**。他干了三件大事：

**1. CMD24/CMD25 写支持**

写操作比读复杂得多，核心问题是 R1B 响应：

```
读操作 (CMD17):
  主机发 CMD17 → 卡返回数据 → 完成
  总线空闲时间: ~100μs-1ms

写操作 (CMD24):
  主机发 CMD24 + 数据 → 卡拉低 DAT0 (R1B busy) → 内部擦除+编程 → 拉高 DAT0
  总线空闲时间: ~10ms-500ms
```

R1B 是 MMC 协议中的一种响应类型，**卡通过拉低 DAT0 线表示自己在忙，主机在此期间不能发送任何命令**。对于写操作，卡内部在完成 NAND 擦除和编程之前不会释放 DAT0。

```c
/* 2004 年 Pierre 增加的 CMD24 写路径（伪代码） */
static int mmc_write_block(struct mmc_card *card, u32 sector, u8 *buf)
{
    struct mmc_command cmd = {0};
    
    cmd.opcode = MMC_WRITE_BLOCK;    // CMD24
    cmd.arg    = sector;
    cmd.flags  = MMC_RSP_R1B;        // ⚠️ 关键: 卡会拉低 DAT0
    cmd.busy_timeout = 500;          // 硬编码 500ms 超时
    
    mmc_wait_for_req(card->host, &cmd, &data);
    
    /* 等 DAT0 释放 */
    while (mmc_card_busy(card)) {
        if (time_after(jiffies, timeout))
            return -ETIMEDOUT;
        cpu_relax();
    }
    return 0;
}
```

**2. 块层集成**：他写了 `mmc_block.c` 和 `mmc_queue.c`，将 MMC 设备接入内核的请求队列（request_fn），使 MMC 设备能像普通块设备一样使用 `mount`、`dd`、`mkfs`。

**3. 多块写 (CMD25)**：2005 年增加，允许一次发送多个扇区，大幅提升顺序写性能。

### FTL 对块设备层的真正影响

2005 年，开发者的认知开始深化：**FTL（Flash Translation Layer）的存在意味着 MMC/SD 卡不是一个"可预测延迟"的块设备。**

这不是简单科普 FTL 是什么，而是它对 Linux 块设备层的具体影响：

| 问题 | 表现 | 根因 |
|------|------|------|
| **写延迟方差极大** | 典型 ~10ms，最差 ~500ms+ | 垃圾回收可能在后台触发 |
| **写放大 (WAF)** | 主机写 4KB，卡内部可能搬 512KB | 擦除块 > 写入页，需要合并 |
| **忙等待不可接受** | CPU 在 while 循环中空转 | 块层期望 completion callback |
| **超时是伪命题** | 设 500ms 但 GC 可能需要 2s | R1B 不是"预计完成时间" |

这些都是上面写操作延迟不可预测的三个原因（GC 写放大、busy_timeout 硬编码、忙等待）。后面在 02-Architecture.md 和 07-Scenario-Analysis.md 中会看到这些因素在块层到硬件层的完整路径上如何叠加。

### 2005 年子系统快照

```
drivers/mmc/
├── mmc.c               ← 单体驱动 ~3000 行 (MMC + SD + 部分 SDIO)
├── mmc_block.c         ← 块设备操作 (Pierre 2004)
├── mmc_queue.c         ← 请求队列 (Pierre 2004)
├── mmc_sysfs.c         ← sysfs 接口
└── mmc_sdio.c          ← SDIO 支持 (新增)

核心能力: MMC/SD 探测(两轮), CMD17 读, CMD24 单写, CMD25 多写
限制:   无 DMA(PIO)、无热插拔、无 eMMC、固定 20MHz、固定 1-bit SPI
当前速度规格:
  MMC:  20MHz 1-bit  →  ~2.5MB/s, 3.3V only
  SD:   25MHz 4-bit  → ~12.5MB/s, 3.3V only (SDHS 50MHz 尚未软件支持)
  SDIO: 同 SD 25MHz 4-bit
  eMMC: 还不存在
维护者: Pierre Ossman (pjrm)
技术债务:
  - 多轮初始化试探（至今未改，硬件限制）
  - R1B busy_timeout 硬编码 500ms
  - mmc_card_busy() 忙等待
```

---

## 四、2006–2007: eMMC 诞生、分层重构、第一个复杂控制器

### eMMC v4.0 标准（2006）

**2006 年 9 月**，JEDEC 发布了 eMMC 标准 v4.0。eMMC 不是一种新卡，而是**把 MMC 总线协议和 NAND Flash 封装在同一个 BGA 封装中，焊在 PCB 上**。它保留了 MMC 的命令集（CMD1/CMD2/CMD3/CMD9/CMD17/CMD24...）和总线协议，但增加了面向嵌入式场景的关键功能：

| 新增功能 | 说明 | 对 Linux 驱动的意义 |
|----------|------|-------------------|
| **双 Boot 分区** | Boot1 + Boot2, 128KB~32MB | 需要 CMD0 带参进入 BOOT 模式，`mmc` 工具可操作 |
| **RPMB** (v4.1) | Replay Protected Memory Block, 128KB~4MB | 需要认证访问，`/dev/mmcblk0rpmb` 字符设备 |
| **可靠写 (Reliable Write)** | CMD23 + CMD25 标记原子写 | 文件系统 metadata 的关键依赖，掉电安全 |
| **擦除/修剪** | CMD38 参数扩展: ERASE / TRIM / DISCARD | 通知卡释放 block，提升 GC 效率，`blkdiscard` 命令 |
| **增强分区** | 指定区域用 SLC 模式，寿命更长 | 适合日志/数据库等高写区域，DTS 中配置 |

eMMC v4.0 的速度规格：

| 模式 | 时钟 | 位宽 | 速率 | 电压 |
|------|------|------|------|------|
| 默认 (Backward) | 0-26MHz | 1/4/8-bit | ~26MB/s | Vcc=3.3V, Vccq=3.3V |
| 高速 (HS) 模式 | 0-52MHz | 1/4/8-bit | ~52MB/s | Vcc=3.3V, Vccq=1.8V/3.3V |

关键变化：**HS 模式将时钟从 26MHz 翻倍到 52MHz（SDR，单沿采样），8-bit 下达到 52MB/s**，这比最早的 MMC（2.5MB/s）快了 20 倍。（DDR 双沿采样在 MMC/eMMC 协议中要到 2009 年 v4.4 才引入；SD 协议也在 2010 年 UHS-I 中加入了 DDR50 模式。）同时 Vccq 开始支持 1.8V——高速信号在 3.3V 下摆幅大、上升沿慢，1.8V 是信号完整性的必然选择。

**eMMC 和 SD 的分水岭：** SD 是消费电子标准（相机/手机可插拔），eMMC 是嵌入式标准（焊在板子上）。两者共享同一套总线物理层，但 eMMC 多了嵌入式场景需要的分区、可靠写、RPMB 等特性。这也是 Linux 驱动中 `if (mmc_card_mmc(card))` 判断的来源——eMMC 走 MMC 协议但需要特殊处理。

### eMMC 分区模型的演进

上面表格提到了"双 Boot 分区"和"RPMB"，但 eMMC 的分区模型不是一次性设计出来的，它经历了三个阶段：

#### 阶段一：v4.0（2006）— 分区模型的诞生

eMMC v4.0 从传统 MMC 卡继承来的只有**一个用户数据区（UDA）**。为了满足嵌入式启动和安全需求，JEDEC 在 v4.0 引入了三种新硬件分区：

```
v4.0 新增的硬件分区:
  Boot Partition 1/2  → 脱离文件系统，Boot ROM 可裸读
  RPMB (v4.1)         → 独立认证，防止重放攻击
  Enhanced Partition  → 指定区域用 SLC 模式，延长寿命
```

为什么是 Boot + RPMB 这两个？

| 需求 | 之前怎么做的（SD 卡方案） | eMMC 的方案 |
|------|--------------------------|-------------|
| 存储 bootloader | SD 卡插槽 + SD 卡 MBR 引导 | 硬件 boot 分区，ROM 直接读裸分区，不需要文件系统栈 |
| 安全计费/DRM 密钥 | 不存在（SD 的 CPRM 是内容加密不是防重放） | RPMB 分区 + HMAC-SHA256 认证，写次数受保护 |

**Boot 分区的设计体现了嵌入式系统的核心矛盾**：CPU 上电时连内存控制器都没初始化，SRAM 通常只有几百 KB。Boot ROM 要在这几百 KB 里完成 MMC 初始化 + 读取固件，不能有多余的协议开销。Boot ROM 读启动分区的流程是：

```
Boot ROM 读 eMMC 启动分区:
  ① 发送 CMD0(arg=0xFFFFFFFA)  → CMD0 带 GO_IDLE 参数 + BOOT_MODE 标志
  ② eMMC 硬件自动从 Boot Partition 的 0 地址开始输出数据
  ③ Boot ROM 连续读取，直到收够所需的字节数
  ④ 发送 CMD0(带 STOP_BOOT 标志) 终止启动模式
  ⑤ 跳转到加载的固件

整个过程中 eMMC 充当了一个"地址不可见、连续输出的 ROM"。
不需要发送 CMD17(READ_SINGLE_BLOCK)，不需要指定 LBA 地址。
```

#### 阶段二：v4.3–v4.4（2008–2009）— 分区控制的完善

v4.3/v4.4 在 EXT_CSD 寄存器中逐步完善了分区的运行时控制能力：

| v4.3 新增 | 作用 |
|-----------|------|
| `PARTITION_CONFIG` (byte 179) | 选择哪个分区映射到用户空间，以及上电从哪个 boot 分区加载 |
| `BOOT_BUS_COND` (byte 177) | 配置启动时的总线宽度（1/4/8-bit）和时序模式 |
| `RST_N_FUNC` (byte 162) | 配置 RST_n 引脚功能（普通复位 / 硬件复位） |
| `BOOT_CONFIG_PROT` (byte 178) | **永久锁定**启动配置（写 0x01 后 byte 177/162/179 再也不能改） |

这些寄存器的设计理念很有趣——**先试错再锁定**。开发阶段 `BOOT_CONFIG_PROT = 0x00`，随意修改启动配置。量产时锁定，防止固件被篡改后修改启动路径。

#### 阶段三：v5.0（2013）— RPMB 完善 + CQHCI

v5.0 在分区模型上没有新增硬件分区类型，但加强了 RPMB 和 CQHCI：
- RPMB 支持更大的分区（最多 64 MiB）
- CQHCI 的队列管理也考虑了分区切换的场景

#### 分区模型的演变总结

```
2006 eMMC v4.0:   Boot1 + Boot2 + RPMB + Enhanced → 4 种新硬件分区诞生
2008 eMMC v4.3:   PARTITION_CONFIG, BOOT_BUS_COND, BOOT_CONFIG_PROT → 运行时控制
2009 eMMC v4.4:   分区切换流程完善，与 HS200 的启动时序配合
2013 eMMC v5.0:   RPMB 容量扩展，CQHCI 支持分区切换
现在              → 分区模型基本固化，后续只增加特性不做结构变更
```

对比 01-Usage.md 中 1.2.8 节的寄存器操作可以体会到：**eMMC 分区操作的本质就是读写 EXT_CSD 中的几个配置字节，触发 eMMC 内部的硬件路由切换。** 没有文件系统元数据的开销，没有块层的参与，这就是硬件分区和软件分区的根本区别。

### Pierre 的分层重构（2006–2007）

2006 年是 MMC 子系统历史上最重要的重构。之前的 `drivers/mmc/mmc.c` 是一个约 4000 行的单体文件，混合了协议解析、块设备、和控制器操作。Pierre 的拆分方案成为今天架构的基石：

```
重构前 (2005):
  drivers/mmc/mmc.c  ← 4000 行单体: 协议 + 块设备 + 控制器
  drivers/mmc/mmc_block.c

重构后 (2007, v2.6.25):
  drivers/mmc/
  ├── core/              ← MMC 核心层（协议无关）
  │   ├── core.c         ← 总线框架、card 管理
  │   ├── mmc.c          ← MMC/eMMC 协议操作
  │   ├── sd.c           ← SD 协议操作（新增）
  │   ├── sdio.c         ← SDIO 协议操作（新增）
  │   ├── mmc_ops.c      ← 底层命令发送
  │   ├── sdio_ops.c
  │   ├── bus.c          ← mmc bus 类型
  │   ├── host.c         ← host 注册管理
  │   └── block.c        ← 块设备驱动（从 mmc_block.c 迁入）
  │
  └── host/              ← 具体控制器驱动
      ├── mmc_spi.c      ← 原来的 SPI 模式驱动
      ├── omap.c         ← OMAP1 控制器
      ├── omap_hsmmc.c   ← OMAP2+ 高速度控制器
      ├── at91_mci.c     ← Atmel
      ├── imxmmc.c       ← i.MX
      ├── s3cmci.c       ← Samsung S3C
      ├── pxamci.c       ← Intel PXA
      └── ...            ← 更多平台
```

这次重构的设计思想：

| 问题 | 解决方案 | 结果 |
|------|---------|------|
| 协议和设备类型混杂 | core/ 层按协议分文件（mmc.c / sd.c / sdio.c） | 新增设备类型只需加文件 |
| 控制器驱动和协议耦合 | host/ 层通过 `mmc_host_ops` 回调解耦 | 换 SoC 只需换 host 驱动 |
| 块设备代码在主机驱动内 | block.c 统一管理，通过 mmc_queue 桥接 | 块层和协议层分离 |

### mmc_host_ops——关键抽象

2006 年引入的 `mmc_host_ops` 是 MMC 子系统最核心的接口抽象：

```c
struct mmc_host_ops {
    /* 发送一个 mmc_request（核心） */
    void (*request)(struct mmc_host *host, struct mmc_request *mrq);
    
    /* 设置 IO 电压、时钟、总线宽度 */
    void (*set_ios)(struct mmc_host *host, struct mmc_ios *ios);
    
    /* 卡检测和写保护 */
    int (*get_ro)(struct mmc_host *host);
    int (*get_cd)(struct mmc_host *host);
    
    /* SDIO 中断 */
    void (*enable_sdio_irq)(struct mmc_host *host, int enable);
};
```

这个设计的意义：**core/ 层负责 MMC/SD/SDIO/eMMC 的协议差异，host/ 层只关心"收发 MMC 总线命令"这个原子操作。** host 驱动不需要知道发的是 CMD17 还是 CMD24，core 层不需要知道寄存器怎么配。

### OMAP hsmmc——第一个"现代"控制器驱动

**2007 年**，德州仪器为 OMAP2/3 平台提交了 `omap_hsmmc.c`。这是第一个真正成熟的 MMC 主机控制器驱动：

```
omap_hsmmc.c 的关键能力（2007）:

  [DMA]    sG DMA 传输          ← 之前都是 PIO
  [4-bit]  数据线模式           ← 之前只有 1-bit SPI
  [8-bit]  eMMC 需要的宽度      ← eMMC 标准要求
  [CLK]    时钟动态调整         ← 省电
           最高时钟 ~52MHz (HS 模式) ← 之前固定 20MHz
  [VOLT]   I/O 电压配置 (1.8V/3.3V)  ← 通过 set_ios 回调设置 vqmmc
  [PM]     电源管理 suspend/resume
  [HOTPLUG]  card detect IRQ    ← 不用轮询
  [FLOW]   硬件流控 DTO 中断     ← 防止 FIFO 溢出
```

它的架构成为后来几乎所有 MMC 主机控制器驱动的模板：配置 DMA 通道、配时钟、配中断、注册 `mmc_host_ops`。STM32MP25 的 SDMMC 控制器驱动 (`drivers/mmc/host/mmci.c`，基于 ARM PrimeCell PL18x) 虽然来自 ARM Ltd 而不是 TI，但整体架构设计与此一致。

### 2007 年子系统快照

```
drivers/mmc/
├── core/            ← ~20,000 行
│   ├── core.c       ← 总线框架
│   ├── mmc.c        ← MMC/eMMC 协议
│   ├── sd.c         ← SD 协议
│   ├── sdio.c       ← SDIO 协议
│   ├── block.c      ← 块设备
│   └── host.c       ← host 注册
└── host/            ← 每个控制器 ~3000-6000 行
    ├── omap_hsmmc.c ← TI OMAP (最成熟)
    ├── mmc_spi.c    ← SPI 模式（兼容古董）
    ├── atmel-mci.c
    ├── imxmmc.c
    ├── s3cmci.c
    ├── pxamci.c
    └── mmci.c       ← ARM PrimeCell PL18x (STM32MP2 的祖先!)

核心能力:
  + 分层架构 (core ↔ host)
  + DMA 传输 (sG)
  + 多设备类型支持 (MMC/SD/SDIO/eMMC)
  + 热插拔 IRQ
  + 时钟动态缩放
  + 电源管理

当前速度规格:
  MMC/eMMC: 26-52MHz 8-bit → 26-52MB/s, 3.3V/1.8V I/O
  SD:        25MHz 4-bit → 12.5MB/s (SDHS 50MHz 部分支持)
  SDIO:      同 SD, 25MHz 4-bit
  SPI:       20MHz 1-bit → 2.5MB/s（古董模式）

维护者: Pierre Ossman (pjrm) — 重构核心
        linux-mmc 邮件列表成立
        mmc 子系统和 networking、block 一样有了独立维护
```

**历史注脚：** 这个分层架构是今天所有 MMC 控制器驱动的基石。从 2007 年到现在（2024），`mmc_host_ops` 只增加了 `execute_tuning`、`enhanced_strobe`、`hw_reset` 少数几个回调——接口本身 17 年没变。

---

## 五、2008–2012: 速度竞赛——HS200、DDR52、tuning、UHS-I

这一时期的核心主线：**MMC/SD 协议从并行（提高位宽）转向串行（提高时钟频率），并通过 DDR 采样再翻倍。** 信号完整性成为主导问题，tuning 机制应运而生。

### eMMC 标准速度演进一览

先纵向看这 5 年 eMMC 标准的速度飞跃：

```
eMMC 标准演进 (2006-2012):
  v4.0 (2006):  26MHz SDR 8-bit →  26MB/s  ← 基本模式
                52MHz SDR 8-bit →  52MB/s  ← HS 模式 (SDR)
  v4.3 (2008):  52MHz SDR 8-bit →  52MB/s  ← +
                新增 Sleep/唤醒、EXT_CSD 访问
  v4.4 (2009):  52MHz DDR 8-bit → 104MB/s  ← DDR52 模式
               200MHz SDR 8-bit → 200MB/s  ← HS200 模式 🚀
  v4.41(2010):  HS200 勘误, 调相(tuning) 具体化
  v4.5 (2011):  200MHz SDR 8-bit → 200MB/s  ← 新增 Cache/CMD Queue/Packed CMD
                高速模式没提，但命令队列降低软件开销

  对比 2006 v4.0 → 2009 v4.4:  52MB/s → 200MB/s, 4 年翻 4 倍
```

关键转折点：**v4.4 引入的 HS200 模式（200MHz SDR, 8-bit）将速率推到 200MB/s。** 这是通过将时钟从 52MHz 提升到 200MHz 实现的，代价是信号完整性变得极难保证。

### DDR52 模式——不升时钟提速率

2009 年 eMMC v4.4 同时引入了 DDR52 模式：

| | DDR52 | HS200 |
|--|-------|-------|
| 时钟 | 52MHz | 200MHz |
| 采样方式 | DDR（双沿） | SDR（单沿） |
| 8-bit 速率 | 104MB/s | 200MB/s |
| 信号电压 | 1.8V 或 3.3V | 必须 1.8V |
| Tuning | 不需要 | 必须 |
| 复杂度 | 低 | 高 |

DDR52 的意义：不升时钟、不 tuning，靠 DDR 翻倍，适合不需要极致性能的嵌入式设备。

DDR52 至今仍是很多嵌入式设备的"甜点模式"——不需要 tuning，不需要 1.8V（可选 3.3V），52MB/s→104MB/s 足够大多数应用。你在 01-Usage.md 中看到的 `mmc-ddr-3_3v` DTS 属性就是为这个模式准备的。

### HS200 与 tuning 的引入

HS200 的核心挑战：**200MHz 下，信号在 PCB 上的传播延迟已经接近一个时钟周期。** 主机不知道卡返回的数据什么时候稳定到达采样寄存器。

```
HS200 的信号完整性问题:

  200MHz 时钟周期 = 5ns
  PCB 走线延迟 ≈ 1ns/15cm (FR4 介质)
  
  问题: 卡发出的数据和时钟到达主机的时间差是未知的
        取决于 PCB 布线长度、温度、电压、芯片 corner
  
  解决方案: tuning (调相)
    主机在初始化阶段发 CMD21 (TUNING) 给卡
    卡返回 64 字节已知模式
    主机尝试不同的采样相位延迟 (delay line taps)
    找出"采样窗口"——所有能正确采到的相位
    取窗口中间作为工作相位（留 margin）
```

**Tuning 在代码中的体现：**

```c
/* mmc_host_ops 中 2009 年新增的回调 */
struct mmc_host_ops {
    /* ... 之前的回调 ... */
    
    /* 执行 tuning（HS200/SDR104 需要） */
    int (*execute_tuning)(struct mmc_host *host, u32 opcode);
    /* opcode = MMC_SEND_TUNING_BLOCK (CMD21) */
};
```

这个回调在 `drivers/mmc/core/mmc.c` 的 `mmc_hs200_tuning()` 中调用。主机控制器驱动需要实现几十到几百个延迟步进的扫描——这也是 STM32MP25 SDMMC 控制器驱动中 `mmci_execute_tuning()` 的核心逻辑。

### 电压的最终分化：1.8V 成为高速的必要条件

到 2009-2010 年，MMC/SD 的电压演进已经完成分化：

```
电压分档:
  3.3V (2.7-3.6V):  Legacy 模式, 低速 (≤52MHz SDR, ≤26MB/s)
  1.8V (1.7-1.95V): 高速模式 (HS200/DDR52/UHS-I)
  1.2V (1.15-1.3V): eMMC v5.0+ 可选 (HS400 增强)

为什么高速必须降电压:
  3.3V: 摆幅 3.3V, 上升时间 ~2ns → 200MHz (5ns 周期) 时信号还没稳就下一个周期
  1.8V: 摆幅 1.8V, 上升时间 ~1ns → 200MHz 下可用
  核心矛盾: 电压越低信号越干净, 但噪声容限越小
```

这也是 DTS 中 `vmmc-supply`（Core 3.3V）和 `vqmmc-supply`（I/O 1.8V/3.3V）分立的根本原因——核心供电始终 3.3V，I/O 电压根据模式切换。

### 2011: eMMC v4.5——Cache 和命令队列

eMMC v4.5 没有提速，但解决了影响实际性能的两个问题：

**1. 写缓存 (Cache):**
在此之前，每个写操作都要等 R1B busy 结束。v4.5 允许 eMMC 内部缓存写数据，主机不必等待每个写完成。`/sys/block/mmcblk0/queue/write_cache` 控制的就是这个功能。开启后写性能大幅提升，但掉电丢数据的风险也增加了（需要 CMD FLUSH 保证）。

**2. 命令队列 (CMD Queue):**
v4.5 引入了 CMD44/QUEUED_TASK_PARAMS + CMD45/QUEUED_TASK_ADDR，允许给每个命令分配一个"任务 ID"（0-31）。但这只是让 CPU 在发命令时多带一个编号，**每条命令还是要 CPU 走 MMC 总线一个个发**：

```
一次 I/O 在 v4.5 "队列"下的总线事务:
  CPU → CMD44(slot=0, lba=100, prio=high)   走 MMC 总线
  CPU → CMD45(slot=0, addr=0x40000000)       走 MMC 总线
  CPU → DATA[512B]                           走 MMC 总线
  CPU ← R1B(busy 10ms)                       等 DAT0 释放
  CPU → CMD44(slot=1, lba=200)               走 MMC 总线
  ...

每个命令还是 CPU 一条条发的, 编个号而已
```

这和 CQHCI（v5.0）有本质区别——CQHCI 不用 CPU 逐条发命令，硬件自己去 DMA 取：
- **v5.0 CQHCI 硬件队列**：CPU 一次性写 32 个任务描述符到内存 → 写门铃 → 硬件自动处理队列 → 独立中断通知每个完成

v4.5 的队列是 CQHCI 的概念原型，但两者实现完全不同。

### 2012: SD UHS-I 与 SD 3.0

SD 卡阵营也在提速：2012 年 SD 3.0 / UHS-I 标准，达到 104MB/s。

```
UHS-I 模式（SD 卡 4-bit 总线）:
  SDR12:   25MHz 4-bit  →  12.5MB/s (1.8V)
  SDR25:   50MHz 4-bit  →  25MB/s   (1.8V)
  SDR50:  100MHz 4-bit  →  50MB/s   (1.8V, 需要 tuning)
  DDR50:   50MHz 4-bit DDR → 50MB/s (1.8V)
  SDR104: 208MHz 4-bit  → 104MB/s   (1.8V, 需要 tuning)

  注意: SD 卡最大 4-bit (不像 eMMC 有 8-bit)
       所以同频率下 SD 速率只有 eMMC 的一半
```

UHS-I 的一个重要变化：**引入了 CMD11（VOLTAGE_SWITCH）作为专用的电压切换命令**。切换前先发 CMD8（SEND_IF_COND）确认卡版本与供电电压匹配，在 ACMD41 中通过 S18R/S18A 位进行 1.8V 协商。如果协商成功，则发送 CMD11 触发物理电压从 3.3V 切换至 1.8V。这与 eMMC 通过 CMD6 选模式 + vqmmc 切电压的两步方案在概念上相似（都是先协商再切换），但协议细节完全不同。

### 维护者变更

2008 年左右，Pierre Ossman 逐渐淡出。MMC 子系统的维护由 **Chris Ball** 和后续的 **Ulf Hansson** 接管。到现在（2024），Ulf Hansson 仍然是 MMC 子系统的维护者（同时也是 PM 子系统的维护者）。稳定的维护者让 MMC 子系统的 API 保持了罕见的向后兼容性。

### 2009 和 2012 子系统快照对比

```
2009 (v2.6.31):
  drivers/mmc/
  ├── core/ (25,000 行)      ← HS200, DDR52, tuning 支持
  └── host/ (每个 ~6000 行)
      ├── omap_hsmmc.c       ← 最先支持 HS200
      ├── dw_mmc.c           ← Synopsys DesignWare (新增)
      ├── sdhci.c            ← SD Host Controller Interface (新增 🚀)
      ├── mmci.c             ← ARM PrimeCell
      └── ...

  SDHCI 的出现: Intel 主导的 SD 主机控制器标准
  后续多数 SoC (包括 STM32MP2 的 SDMMC) 都受 SDHCI 规范影响
  但 STM32MP2 走的是 ARM PrimeCell PL18x + ST 自家封装, 没直接用 SDHCI

2012 (v3.6):
  drivers/mmc/
  ├── core/ (35,000 行)
  │   ├── mmc.c              ← HS200 tuning, eMMC v4.5 cache/queue
  │   ├── sd.c               ← UHS-I tuning
  │   ├── sdio.c
  │   ├── block.c
  │   └── ...
  └── host/ (每个 ~8000 行)
      ├── sdhci.c            ← 成为事实标准
      ├── dw_mmc.c
      ├── omap_hsmmc.c
      ├── mmci.c
      └── ...

速度规格:
  eMMC:  200MHz SDR 8-bit → 200MB/s, 1.8V I/O
  SD:    104MHz SDR → 104MB/s (UHS-I), 1.8V I/O
  SDIO:  同 SD UHS-I
  SPI:   20MHz 1-bit → 2.5MB/s（维持不变）

维护者: Chris Ball → Ulf Hansson (2012 至今)
```

**历史注脚：** HS200 模式（200MHz SDR）是 eMMC 速度提升的转折点。此后 10 年（2012-2022），eMMC 的速度增长明显放缓——HS400（v5.0, 2013）只再翻倍到 400MB/s，而物理层（MIPI M-PHY）的极限已经触到了。这也是为什么业界开始转向 UFS——它不是 MMC 协议的演进，而是一个全新的体系。

---

## 六、2013–2017: HS400、CQHCI、blk-mq——从协议层到块层的重构

这个时期，eMMC 协议本身的发展开始放缓，但 Linux MMC 子系统经历了从块层到核心的重大重构。

### eMMC v5.0 (2013): HS400 + CQHCI

eMMC v5.0 是 eMMC 历史上最后一个重大速度升级，此后速度停留在 400MB/s。

```
eMMC 标准演进 (2013-2015):
  v5.0 (2013):  200MHz DDR 8-bit → 400MB/s  ← HS400 🚀
                新增 CQHCI (硬件命令队列)
                新增 Enhanced Strobe (HS400 调相)
                新增 1.2V I/O 可选
  
  v5.1 (2015):  400MB/s 不变
                新增 BKOPS 控制 (Background Ops)
                新增健康报告 (PRE_EOL, LIFE_TIME_EST)
                写缓存增强

  对比:  2009 v4.4 → 2013 v5.0:  200MB/s → 400MB/s, 4 年翻倍
         2013 v5.0 → 至今:       停留在 400MB/s, 已有 12 年未提速
```

为什么 HS400 之后不再提速？三个原因：
- **物理层到头了**：200MHz DDR 的时序裕量已经极小，再提频需要换物理层（MIPI M-PHY）
- **UFS 分流**：追求性能的应用转向 UFS（基于 SCSI 模型，不是 MMC 协议）
- **eMMC 的定位固化**：变成"够用就好"的嵌入式存储，焊在板子上不追求极致速度

### HS400 与 Enhanced Strobe

HS400 = HS200 的时钟翻倍 + DDR 采样：

```
HS200 → HS400:
  HS200 (v4.4, 2009):  200MHz SDR  8-bit →  200MB/s
  HS400 (v5.0, 2013):  200MHz DDR  8-bit →  400MB/s
                        ↑ 时钟不变，DDR 让速率翻倍
```

但 HS400 引入了一个关键创新：**Enhanced Strobe（增强选通）**。

```
HS200 的 tuning 问题:
  HS200 靠 host 调相 (delay line) 找采样点
  但 200MHz DDR 下，数据有效窗口只有 ~2.5ns
  delay line 的精度 (通常 ~100ps 步进) 不够用了

Enhanced Strobe 方案:
  eMMC 设备发送数据时，同时发一个差分选通信号 (DQS)
  host 用 DQS 而不是自己的时钟来锁存数据
  DQS 和数据是同步从 eMMC 发出的 → 自动对齐, 不需要 tuning
  → 代码中的 mmc_host_ops->enhanced_strobe 回调
  
  代价: host 控制器必须支持 DQS 信号接收
       STM32MP25 SDMMC 支持 Enhanced Strobe
```

这是 eMMC 和 SD 在高速路径上的一个关键分岔——SD 卡没有 Enhanced Strobe 的等价物。

### CQHCI——硬件命令队列

eMMC v4.5 (2011) 已经有命令队列能力，但那是**软件队列**。v5.0 的 CQHCI 是**硬件队列**。两者的本质区别只有一句话：

> **v4.5 的"排队"走的是 MMC 总线，CQHCI 的"排队"走的是内存总线。**

什么意思？对比一次 I/O 的操作路径：

```
写一次数据 (以 slot=0, lba=100, 数据在 DRAM 0x40000000 为例):

  v4.5 软件队列:
    MMC 总线事务 1:  CPU → CMD44(slot=0, lba=100)        ← 走 MMC 总线
    MMC 总线事务 2:  CPU → CMD45(slot=0, addr=0x40000000) ← 走 MMC 总线
    MMC 总线事务 3:  CPU → DATA(512B)                      ← 走 MMC 总线
    MMC 总线事务 4:  CPU ← R1B(等 10ms)                   ← DAT0 忙
    ... 然后下一个命令 ...
    总计: 每个命令 4 次 MMC 总线事务, 全是 CPU 控制

  CQHCI 硬件队列:
    [CPU 准备阶段, 走内存总线, 不占用 MMC 总线]
    CPU 写 DRAM: task_desc[0] = {op=WRITE, lba=100, addr=0x40000000}
    CPU 写 DRAM: task_desc[1] = {op=WRITE, lba=200, addr=0x40001000}
    CPU 写 DRAM: task_desc[2] = {op=WRITE, lba=300, addr=0x40002000}
    CPU MMIO:    CQHCI_SQTDBR = BIT(0) | BIT(1) | BIT(2)   ← 一个 4 字节 MMIO 写

    [CQHCI 硬件接管, 通过 DMA 访问 DRAM, 自动发 MMC 命令]
    硬件 DMA:    ← 读 task_desc[0] (走内存总线, 不走 MMC 总线)
    硬件 → MMC: ← CMD25(lba=100) + DATA(512B)              ← 走 MMC 总线
    硬件 ← MMC: → R1B
    硬件 DMA:    ← 读 task_desc[1]
    硬件 → MMC: ← CMD25(lba=200) + DATA(512B)
    ...
    硬件 → CPU:  中断: "slot 0 完成! slot 1 完成! slot 2 完成!"
    总计: CPU 只写了 3 次 DRAM + 1 次 MMIO, 剩下硬件自己干
```

**关键区别总结：**

| | v4.5 软件队列 | v5.0 CQHCI |
|--|-------------|-----------|
| 队列存在哪里 | CPU 内存 | CPU 内存 (准备阶段) + 硬件寄存器 (执行阶段) |
| 队列怎么提交 | 每个命令走 MMC 总线发 CMD44+CMD45 | 写 DRAM (内存总线) + 写 1 次 MMIO 门铃 |
| 队列谁执行 | CPU 逐条控制 | 硬件通过 DMA 自动取任务、发命令 |
| 通知完成方式 | CPU 发 CMD13 轮询 | 硬件产生完成中断 |
| MMC 总线额外开销 | 每条 I/O 多 2 个 CMD44/CMD45 | 零 — 队列管理不走 MMC 总线 |

CQHCI 的硬件执行流程：

```
主机提交命令:
  ① 分配一个空闲 queue slot (0-31)
  ② 在内存中填写 Task Descriptor (命令参数、数据地址、DMA 链表)
  ③ 写门铃寄存器 CQHCI_SQTDBR: 置位 bit[slot]
  ④ 硬件 DMA 读取 Task Descriptor，自动发命令、传数据
  ⑤ 每个 slot 完成 → 硬件产生中断
```

### blk-mq 集成

**先理解"块层"（block layer）是干什么的：**

```
VFS (read/write/open/close)
  │
  ↓  文件系统 (ext4, ubifs...)
  │   将文件读写翻译成"扇区偏移 + 长度"
  ↓
  block layer  ← 本节的主角
  │   收到 file system 发来的 struct request
  │   负责排队、合并、调度、发送给驱动
  ↓
  mmc_block.c
  │   将 struct request 拆成 struct mmc_request
  ↓
  mmc core (mmc_wait_for_req)
  │
  ↓
  host 控制器
```

块层就是 VFS/文件系统和设备驱动之间的**调度器**。它解决两个核心问题：
- **合并 (merge)**：相邻扇区的请求合并成一个，减少发命令次数
- **排队 (queue)**：多个进程同时读写时，决定谁先谁后

**单队列（request_fn）的问题：**

2014 年之前，块层只有一个全局队列（一个 `request_queue` 一把锁）。这在单核 CPU 上没问题，但多核下就变成了：

```
双核 SoC (如 STM32MP2: 2×Cortex-A35):

  CPU 0                    CPU 1
    │                         │
    │ 进程 A: write()         │
    │   → 锁定 request_queue  │ 进程 B: write()
    │   → 加入请求            │   → 等待锁 (空转!)
    │   → 解锁                │   → 拿到锁
    │   → 等 mmc 发完命令      │   → 加入请求
    │                        │   → 等 mmc 发完命令
    │     大部分时间一个核干活, 另一个核等锁
```

这就是 **lock contention（锁竞争）**。两个 CPU 不能同时提交 I/O。

**blk-mq（multi-queue）的解决方案：**

blk-mq 把全局一把锁拆成了两层队列：

```
blk-mq 架构:

                   进程 A 提交           进程 B 提交
                      │                    │
                      ▼                    ▼
  [软件队列层]   per-CPU 队列 0      per-CPU 队列 1
                (无锁, 只本 CPU 用)   (无锁, 只本 CPU 用)
                      │                    │
                      ▼                    ▼
  [硬件队列层]   ──────────── 分发 ────────────
                      │
                      ▼
                  驱动 (mmc_block)
                      │
                      ▼
                  发送命令
```

**per-CPU 队列 = 每个 CPU 自己的收件箱。** CPU 0 来的请求放进 CPU 0 的队列，CPU 1 的放进 CPU 1 的队列。各写各的，不需要锁。然后块层从所有 per-CPU 队列中取请求、合并、发给驱动。

```
关键变化:

           单队列 (request_fn)      blk-mq
           ─────────────────      ───────
  锁策略    一把全局锁               per-CPU 无锁提交
  多核行为   一个核干活, 其他等锁      多核并行提交
  CQHCI 配合 不支持                  每个 slot 映射到硬件队列
  代码    block/genhd.c            block/blk-mq.c
```

**MMC 在 2017 年（v4.13）切换到了 blk-mq。** 这意味着：

```
对于 STM32MP2 (2×A35, eMMC 用户数据分区):

  多核并发写场景:
    CPU 0: 进程 A 写文件 → per-CPU queue 0
    CPU 1: 进程 B 写文件 → per-CPU queue 1
                                   ↓
                           blk-mq 合并分发
                                   ↓
                            mmc_block + CQHCI
                            (32 个 slot, 硬件调度)
```

这就是 blk-mq 对多核 SoC 的意义——每个核独立提交，不抢同一把锁。

### 2017 年的 MMC 子系统快照

```
drivers/mmc/
├── core/ (~45,000 行)
│   ├── mmc.c           ← HS400, Enhanced Strobe
│   ├── sd.c            ← UHS-I 全部模式
│   ├── block.c         ← blk-mq + CQHCI 集成
│   ├── queue.c         ← blk-mq 封装
│   ├── cqhci.c         ← CQHCI 核心 (新增 ~1500 行)
│   └── ...
│
└── host/ (每个 ~8000-12,000 行)
    ├── sdhci.c         ← SDHCI 标准 (支持 CQHCI)
    ├── sdhci-of-*.c    ← 各 SoC 的 SDHCI 封装
    ├── dw_mmc.c        ← Synopsys DesignWare
    ├── mmci.c          ← ARM PrimeCell PL18x (STM32MP2 的祖先)
    └── ...

速度规格:
  eMMC:  200MHz DDR 8-bit → 400MB/s, 1.8V/1.2V I/O
  SD:    208MHz 4-bit → 104MB/s (UHS-I SDR104), 1.8V I/O
  SDIO:  同 UHS-I (WiFi 等)
  SPI:   20MHz 1-bit → 2.5MB/s

维护者: Ulf Hansson (2012 至今)
```

**历史注脚：** eMMC 从 v5.0 (2013) 到现在的 v5.2 (2024)，HS400 的 400MB/s 上限一直没变。这不是 Linux 的问题，是 JEDEC 标准放弃了物理层升级，把"高速存储"这个赛道拱手让给了 UFS。但 eMMC 以其低成本、小封装、够用的性能，在嵌入式领域（尤其工业、IoT、汽车）还会存在至少 10 年。

---

## 七、2017–2024: STM32MP2 SDMMC 与当前格局

这一节把前面 30 年的演进收束到**你手上的平台**——STM32MP257 的 SDMMC 控制器及其驱动 `mmci.c`。

### STM32MP2 SDMMC 控制器是什么？

STM32MP2 的 SDMMC 控制器不是一般的 SDHCI 标准控制器，而是基于 **ARM PrimeCell MCI (PL18x)** 加上 ST 的扩展。这意味着：

```
SDMMC 控制器的家族树:

  SDHCI 阵营                          ARM PrimeCell 阵营
  (Intel 主导的标准)                   (ARM 自主研发)
  ┌───────────┐                       ┌───────────┐
  │ sdhci.c   │                       │ mmci.c    │ ← STM32MP2 在这里
  ├───────────┤                       ├───────────┤
  │ sdhci-of-*│ (各 SoC 封装)          │ stm32mp25 │ ← ST 扩展
  │ dw_mmc    │ (Synopsys)            │ qcom      │ (高通扩展)
  │ sdhci-xenon│ (Marvell)            │ ux500     │ (ST-Ericsson)
  └───────────┘                       └───────────┘

  STM32MP2 的兼容性字符串:
  compatible = "st,stm32mp25-sdmmc2", "arm,pl18x", "arm,primecell";
                ↑  ST 扩展              ↑  ARM IP       ↑ AMBA 总线
```

### mmci.c 驱动的架构

`drivers/mmc/host/mmci.c` 是 ARM PrimeCell MCI 的通用驱动，通过 `variant_data` 结构体区分不同厂商和版本的差异：

```c
/* 每个 SoC 变体的差异通过这个结构体描述 */
struct variant_data {
    unsigned int    clkdiv_max;       /* 最大分频系数 */
    unsigned int    datactrl_mask_ddr;/* DDR 模式掩码 */
    unsigned int    datactrl_mask_buswidth; /* 位宽掩码 */
    bool            st_clkdiv;        /* ST 的分频器 */
    bool            stm32_idmabs;     /* STM32 IDMA 支持 */
    bool            dma_lli;          /* DMA 链表支持 */
    /* ... */
};
```

STM32MP25 的平台数据定义（在 `drivers/mmc/host/mmci_stm32_sdmmc.c` 中）：

| 特性 | STM32MP25 SDMMC | 说明 |
|------|----------------|------|
| 最大时钟 | 200MHz | HS200/HS400 |
| 位宽 | 1/4/8-bit | eMMC 8-bit |
| HS400 | 支持 | 带 Enhanced Strobe |
| CQHCI | 支持 | 硬件命令队列 |
| IDMA | 支持 | 内部 DMA (无需外部 DMA 控制器) |
| 电压 | 3.3V/1.8V | 通过 vqmmc-supply 配置 |
| 时钟边沿 | `st,neg-edge` | 下降沿采样 (DTS 配置) |

### Probe 流程前瞻

这是 03-Probe-Analysis.md 的预告，先看整体流程：

```
STM32MP2 SDMMC probe 流程:

  ① platform_driver_register("st,stm32mp25-sdmmc2")
     → mmci_probe()
  
  ② 获取 variant_data
     → 根据 compatible 匹配 stm32mp256_variant_data
  
  ③ ioremap 寄存器
  ④ 时钟: clk_get + clk_prepare_enable
     → 三个时钟: sw(内核), apb(总线), ahb(DMA)
  
  ⑤ 复位: reset_control_get + reset_control_deassert
  
  ⑥ 中断: devm_request_irq()
     → handler: mmci_irq()
     → 处理 11 种中断源 (CMD/DATO/TX/RX/...)
  
  ⑦ DMA: 配置 IDMA 或外部 DMA
  
  ⑧ mmc_alloc_host() + mmc_add_host()
     → 注册到 MMC 核心层
     → 核心层开始卡检测/初始化流程
```

### 2024 年 MMC 子系统全景回顾

```
drivers/mmc/ 总览 (~60,000 行代码, 30 年积累):

  core/ (~30,000 行)
    core.c           ─ 总线管理, 热插拔, 电源管理
    mmc.c            ─ eMMC 协议 (mmc_attach_bus → init → HS200 → HS400)
    sd.c             ─ SD 协议 (UHS-I, SDR104)
    sdio.c           ─ SDIO 协议
    block.c          ─ blk-mq 块设备, CQHCI 集成
    queue.c          ─ blk-mq 队列封装
    cqhci.c          ─ CQHCI 硬件队列 (~1500 行)
    mmc_ops.c        ─ 底层命令发送 mmc_wait_for_cmd()

  host/ (~30,000 行)
    sdhci.c          ─ SDHCI 标准主机 (最主流)
    dw_mmc.c         ─ Synopsys DesignWare
    mmci.c           ─ ARM PrimeCell PL18x (STM32MP2)
    mmci_stm32_sdmmc.c ─ STM32MP2 扩展 (STM 维护)
    omap_hsmmc.c     ─ TI OMAP (历史贡献者)
    ...

速度规格 (2024):
  eMMC:  200MHz DDR 8-bit → 400MB/s  ← HS400 (2013 年至今未变)
  SD:    208MHz 4-bit SDR → 104MB/s (SDR104)
          50MHz 4-bit DDR →  50MB/s (DDR50)
  SDIO:  同 SD (WiFi 6 网卡 ~1.2Gbps 但通过 PCIe 不是 SDIO)
  SPI:   20MHz 1-bit → 2.5MB/s (兼容古董)
```

### 30 年演进总结

```
1997   MMC v1.0         7-pin, 20MHz 1-bit, ~2.5MB/s, 3.3V
1999   SD 卡出现         9-pin, 25MHz 4-bit, ~12.5MB/s
       Linux 第一个驱动  David Brownell, 只读 SPI 模式

2004   Pierre Ossman     写支持 (CMD24/CMD25), R1B busy, 块层
2005   四设备类型          MMC + SD + SDIO + (eMMC 即将到来)

2006   eMMC v4.0         BGA 封装, 分区, 可靠写, 52MB/s HS
2007   分层重构           core/ + host/ 分离, mmc_host_ops

2009   eMMC v4.4         HS200: 200MHz SDR 8-bit → 200MB/s
                         1.8V I/O 成为高速必要条件
                         Tuning 引入 (CMD21)

2011   eMMC v4.5         写缓存, 软件命令队列 (CMD44/CMD45)
2012   SD UHS-I          104MB/s (SDR104), CMD8 电压切换

2013   eMMC v5.0         HS400: 200MHz DDR 8-bit → 400MB/s
                         CQHCI 硬件命令队列
                         Enhanced Strobe (DQS)

2015   eMMC v5.1         健康报告, BKOPS, 写缓存增强
2016   blk-mq 集成       MMC 块设备切换到多队列

2017-2024                 400MB/s 上限未变
                          UFS 接手"高速"赛道
                          eMMC 固守嵌入式/工业/汽车市场
```

### 从历史到源码——后续文档的起点

这 30 年演进留下的分层架构是后续分析的路线图：

| 历史层 | 对应文档 | 核心内容 |
|--------|---------|---------|
| 物理层 (总线/电压/时序) | 05-Hardware-Registers.md | STM32MP2 SDMMC 寄存器 |
| 主机控制器 (mmci.c) | 03-Probe-Analysis.md | probe 逐行分析 |
| MMC 核心层 (core/) | 02-Architecture.md | 分层 + 数据结构 + 状态机 |
| 块设备层 (block.c) | 04-DataPath-Analysis.md | blk-mq → mmc_request 路径 |
| 用户空间接口 | 01-Usage.md | sysfs, debugfs, mmc 工具 |

继续阅读：[01-Usage.md](01-Usage.md) — 使用方法与 DTS 配置

---

