# 01. eMMC/MMC 子系统使用方法

> 本文是 [eMMC 驱动深度分析](README.md) 系列的第 1 篇。  
> 对应框架步骤 ①：从用户态、内核态、DTS 三个维度了解 eMMC 子系统。

---

## 1.1 用户态接口

### 查看 eMMC/SD 设备信息

```bash
# 查看块设备列表
ls -la /dev/mmcblk*
# 典型输出:
#   /dev/mmcblk0      ← eMMC 主设备
#   /dev/mmcblk0p1    ← 分区 1
#   /dev/mmcblk0p2    ← 分区 2
#   /dev/mmcblk0boot0 ← eMMC 启动分区 0
#   /dev/mmcblk0boot1 ← eMMC 启动分区 1
#   /dev/mmcblk0rpmb  ← RPMB 分区（安全区）

# 查看 eMMC 详细信息（最常用的调试命令）
cat /sys/kernel/debug/mmc0/ios
# 输出示例:
#   clock:          200000000 Hz    ← 当前时钟频率
#   vdd:            21 (3.3V)       ← 电压
#   bus mode:       2 (push-pull)
#   chip select:    0 (don't care)
#   power mode:     2 (on)
#   bus width:      3 (8 bits)      ← 总线位宽
#   timing spec:    9 (mmc hs200)    ← 当前时序模式
#   signal voltage: 1 (1.8V)        ← 信号电压

# 查看 eMMC 内部寄存器信息（CID/CSD/ECSD）
cat /sys/kernel/debug/mmc0/registers
# 或
mmc extcsd read /dev/mmcblk0
```

**`mmc` 工具**（mmc-utils 包）是调试 eMMC 的瑞士军刀：

```bash
# 安装（Buildroot 中选 BR2_PACKAGE_MMC_UTILS）
mmc extcsd read /dev/mmcblk0     # 读取 ECSD 寄存器（最有用）
mmc status get /dev/mmcblk0       # 获取设备状态
mmc writeprotect get /dev/mmcblk0 # 写保护状态
mmc hwreset enable /dev/mmcblk0   # 硬件复位使能

# 查看 eMMC 的寿命/健康度（eMMC v5.0+）
mmc extcsd read /dev/mmcblk0 | grep -i "life\|pre_eol\|user"
#   PRE_EOL_INFO: 0x01           ← 0=正常, 1=消耗 >80%, 2=消耗 >90%, 3=已到寿命
#   LIFE_TIME_EST: 0x01 0x01     ← [0-10] 估计寿命，10 为耗尽
```

### 性能测试

```bash
# 顺序读
dd if=/dev/mmcblk0 of=/dev/null bs=1M count=100

# 顺序写（注意：会写数据！）
dd if=/dev/zero of=/tmp/test.bin bs=1M count=100

# 用 hdparm 看缓存策略
hdparm -W /dev/mmcblk0           # 查看写缓存是否开启
hdparm -W 1 /dev/mmcblk0         # 开启写缓存

# 查看队列深度
cat /sys/block/mmcblk0/device/queue_depth
```

### 排查 `sync` 问题

```bash
# 1. 查看当前 I/O 状态
cat /sys/block/mmcblk0/stat
# 格式: read-issued read-completed ... write-issued write-completed ... io-in-flight io-time
# 重点关注: io-in-flight != 0 说明 I/O 卡住
#          io-time 很大说明请求延迟异常

# 2. 查看 I/O 错误计数
cat /sys/block/mmcblk0/device/err stats
# 输出格式: 读错误数 写错误数 校验错误数 ...

# 3. 查看 eMMC 设备状态
mmc status get /dev/mmcblk0
# 返回 SWITCH_ERROR 或 ECC_FAILED 说明内部有问题

# 4. 查看内核日志
dmesg | grep -iE "mmc|mmcblk|sdhci|sync"
# 关注:
#   - "mmc0: Timeout waiting for hardware interrupt" ← 超时
#   - "mmcblk0: error -110 sending status command"   ← -110 = ETIMEDOUT
#   - "mmc0: card never left busy state"              ← eMMC 忙状态卡死
#   - "blk_update_request: I/O error"                 ← 块层 IO 错误

# 5. 往 mmc 模块开动态 debug
echo 'file drivers/mmc/core/core.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/mmc/host/*.c +p' > /sys/kernel/debug/dynamic_debug/control
```

### /sys 下的 eMMC 属性

```bash
# /sys/block/mmcblk0/ 下的关键文件
cat /sys/block/mmcblk0/size                   # 设备大小（sector 数）
cat /sys/block/mmcblk0/queue/logical_block_size  # 逻辑块大小（通常 512）
cat /sys/block/mmcblk0/queue/physical_block_size # 物理块大小（eMMC 通常 512B 或 4KB）
cat /sys/block/mmcblk0/queue/max_hw_sectors_kb   # 最大硬件传输大小
cat /sys/block/mmcblk0/queue/write_cache        # 写缓存策略

# /sys/bus/mmc/devices/ 下的关键文件
ls /sys/bus/mmc/devices/mmc0:0001/              # eMMC 设备目录
cat /sys/bus/mmc/devices/mmc0:0001/cid          # Card Identification
cat /sys/bus/mmc/devices/mmc0:0001/csd          # Card Specific Data
cat /sys/bus/mmc/devices/mmc0:0001/ecsd         # Extended CSD
cat /sys/bus/mmc/devices/mmc0:0001/date         # 生产日期
cat /sys/bus/mmc/devices/mmc0:0001/manfid       # 制造商 ID
cat /sys/bus/mmc/devices/mmc0:0001/name         # 型号名
cat /sys/bus/mmc/devices/mmc0:0001/ocr          # 电压范围
cat /sys/bus/mmc/devices/mmc0:0001/fwrev        # 固件版本
```

---

## 1.2 内核态 API

### 核心数据结构

```c
struct mmc_host;     /* MMC 主机控制器抽象 —— 每个 SDMMC 控制器一个 */
struct mmc_card;     /* MMC/SD/SDIO 卡抽象 —— 代表一个具体的卡/eMMC */
struct mmc_request;  /* 一个 MMC 协议请求（包含 CMD + 可选 DATA + 可选 STOP） */
struct mmc_command;  /* 一条 MMC 总线命令 */
struct mmc_data;     /* 数据段 */
```

### 主机控制器驱动注册

```c
/* 分配和注册 mmc_host */
struct mmc_host *mmc_alloc_host(int extra, struct device *dev);
int mmc_add_host(struct mmc_host *host);

/* 主机控制器必须实现的 ops */
struct mmc_host_ops {
    /* 发送命令（必须） */
    void (*request)(struct mmc_host *host, struct mmc_request *mrq);
    
    /* 设置总线的 I/O 电压和时序 */
    int (*set_ios)(struct mmc_host *host, struct mmc_ios *ios);
    
    /* 获取卡检测和写保护信号 */
    int (*get_ro)(struct mmc_host *host);
    int (*get_cd)(struct mmc_host *host);
    
    /* 中断相关 */
    void (*enable_sdio_irq)(struct mmc_host *host, int enable);
    
    /* 初始化卡片（可选，mmc/core 层有默认实现） */
    int (*init_card)(struct mmc_host *host, struct mmc_card *card);
    
    /* 高速调相 (tuning) */
    int (*execute_tuning)(struct mmc_host *host, u32 opcode);
};
```

### 块设备操作（blk-mq 路径）

```c
/* mmc/core/block.c — 块设备请求处理 */
static blk_status_t mmc_blk_mq_rw_rq(struct mmc_queue *mq,
                                       struct request *req);
// 每个 req 被转换为一个 mmc_request，通过 host->ops->request() 发送
// 然后等待 completion 完成
```

---

## 1.3 DTS 配置

### STM32MP2 SDMMC 控制器（stm32mp251.dtsi）

```dts
/* SoC 级定义 */
sdmmc1: mmc@50110000 {
    compatible = "st,stm32mp25-sdmmc2", "arm,pl18x", "arm,primecell";
    reg = <0x50110000 0x1000>;
    interrupts = <GIC_SPI 134 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&rcc CK_KER_SDMMC1>,
             <&rcc CK_BUS_SDMMC1>,
             <&rcc CK_BUS_SDMMC1>;
    clock-names = "sw", "apb", "ahb";
    resets = <&rcc SDMMC1_R>;
    reset-names = "reset";
    status = "disabled";
};

sdmmc2: mmc@50120000 {
    compatible = "st,stm32mp25-sdmmc2", "arm,pl18x", "arm,primecell";
    reg = <0x50120000 0x1000>;
    interrupts = <GIC_SPI 135 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&rcc CK_KER_SDMMC2>,
             <&rcc CK_BUS_SDMMC2>,
             <&rcc CK_BUS_SDMMC2>;
    clock-names = "sw", "apb", "ahb";
    resets = <&rcc SDMMC2_R>;
    reset-names = "reset";
    status = "disabled";
};
```

### 板级配置：eMMC 固定不可移除

```dts
/* eMMC 通常接在 SDMMC2，SD 卡接在 SDMMC1 */
&sdmmc2 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&sdmmc2_b4_pins_a &sdmmc2_d47_pins_a>;
    pinctrl-1 = <&sdmmc2_b4_sleep_pins_a &sdmmc2_d47_sleep_pins_a>;
    
    /* eMMC 关键配置 */
    non-removable;              /* ⚠️ 焊在板上，不是可插拔设备 */
    bus-width = <8>;            /* 8-bit 数据线 */
    vmmc-supply = <&v3v3>;     /* 核心电压 3.3V */
    vqmmc-supply = <&vdd_io>;  /* I/O 电压 1.8V 或 3.3V */
    max-frequency = <200000000>; /* 最大时钟 200MHz */
    no-sd;                      /* 不是 SD 卡 */
    no-sdio;                    /* 不是 SDIO */
    cap-mmc-highspeed;          /* 支持 MMC 高速模式 */
    mmc-ddr-3_3v;               /* DDR 模式 3.3V */
    mmc-hs200-1_8v;             /* HS200 @1.8V */
    mmc-hs400-1_8v;             /* HS400 @1.8V（如果支持）*/
    
    /* STM32MP2 特定配置 */
    st,neg-edge;                /* 使用时钟下降沿采样 */
    bus-width = <8>;
    status = "okay";
};
```

### 关键 DTS 属性说明

| 属性 | 作用 | 不设的后果 |
|------|------|-----------|
| `non-removable` | 不从设备检测引脚读状态 | 内核会不断轮询卡检测，浪费资源，行为异常 |
| `bus-width` | 数据线数量（1/4/8） | 默认 1-bit 模式，性能极差 |
| `max-frequency` | 最高时钟频率 | 默认可能很低（~400kHz），或太高导致信号不稳 |
| `no-sd` / `no-sdio` | 限制设备类型 | 不加的话内核会尝试 SD/SDIO 初始化，浪费时间 |
| `cap-mmc-highspeed` | 允许 26MHz 高速模式 | 可能被限制在 20MHz 低速率 |
| `mmc-hs200-1_8v` | 允许 HS200 模式 | 最高只能到 DDR52，带宽减半 |
| `st,neg-edge` | STM32MP2 时钟相位 | 不设可能导致数据采样错误 |

---

## 1.4 与 `sync` 问题直接相关的检查点

先提供一个快速排查命令，后续架构和分析中会深入解释原理：

```bash
# 第一步：确认设备状态
cat /sys/block/mmcblk0/device/er rstts
mmc status get /dev/mmcblk0
dmesg | grep -iE "mmc0\|mmcblk0\|error\|timeout\|fail"

# 第二步：看电源和信号（如果有 devmem）
devmem 0x50120000 32                # SDMMC2 电源控制寄存器
devmem 0x50120008 32                # SDMMC2 时钟控制寄存器

# 第三步：看写缓存
cat /sys/block/mmcblk0/queue/write_cache

# 第四步：强制刷缓存看报错
sync
echo 3 > /proc/sys/vm/drop_caches
sync
# 观察 dmesg 输出
```
