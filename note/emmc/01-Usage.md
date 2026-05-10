# 01. eMMC 子系统使用方法

> 本文是 [STM32MP257 eMMC 驱动深度分析](README.md) 系列的第 1 篇。  
> 对应框架步骤 ①：从 DTS、用户态、内核态三个视角了解 eMMC 子系统。
>
> **字数：** 约 10,800 字  
> **建议阅读时间：** 40–55 分钟（含代码和图表）

---

## 1.1 DTS 配置

DTS 决定了 MMC 子系统如何初始化 SDMMC 控制器。配置是否正确，直接体现在启动日志中——每一条 DTS 属性，在日志里都有对应的结果。

### SoC 级定义

STM32MP25 在 `stm32mp251.dtsi` 中定义了三个 SDMMC 控制器。eMMC 接在 SDMMC2 上：

```dts
sdmmc2: mmc@48230000 {                          /* eMMC */
    compatible = "st,stm32mp25-sdmmc2", "arm,pl18x", "arm,primecell";
    arm,primecell-periphid = <0x00353180>;
    reg = <0x48230000 0x400>, <0x44230800 0x8>;
    interrupts = <GIC_SPI 197 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&rcc CK_KER_SDMMC2>;
    clock-names = "apb_pclk";
    resets = <&rcc SDMMC2_R>;
    cap-sd-highspeed;
    cap-mmc-highspeed;
    max-frequency = <166000000>;
    access-controllers = <&rifsc 77>;
    power-domains = <&d1_pd>;
    status = "disabled";
};
```

SoC 级定义的几个缺省值，后续会被板级 DTS 覆盖或补充：

| 属性 | 缺省值 | 说明 |
|------|--------|------|
| `max-frequency` | 166 MHz | 板级可以设更高（如 200MHz 对应 HS200），但 ATK 板没有覆盖 |
| `cap-mmc-highspeed` | 已使能 | 只允许 26/52MHz SDR 模式，更高模式需要板级追加属性 |
| `clocks` | 单时钟 | 只有一个 `CK_KER_SDMMC2`，名字叫 `apb_pclk` |

---

### 电源配置

```dts
vmmc-supply  = <&scmi_vdd_emmc>;   /* VCC 核心供电，3.3V，来自 PMIC LDO2 */
vqmmc-supply = <&scmi_vddio2>;     /* VCCQ I/O 供电，固定 1.8V */
```

这两个属性是 eMMC 正常工作的基础。

| 属性 | 连接 eMMC 引脚 | 电压 | 不配的后果 |
|------|---------------|------|-----------|
| `vmmc-supply` | VCC | 3.3V | 卡不上电，日志中完全不出现 mmc 相关消息 |
| `vqmmc-supply` | VCCQ | 1.8V（固定） | eMMC 无法进入 DDR52 及以上模式 |

ATK 板的 eMMC VCCQ 固定接 1.8V，不需要电压切换。所以 `vqmmc-supply` 对应的 regulator（`scmi_vddio2`）是固定输出，不像某些设计需要动态切 3.3V/1.8V。

---

### 模式使能

```dts
mmc-ddr-1_8v;        /* 允许 DDR52 @1.8V，52MHz DDR = 104MB/s */
```

eMMC 有多种模式可选，由低到高：

| 模式 | DTS 属性 | 速率（8-bit） | 说明 |
|------|---------|-------------|------|
| LEGACY | 无（默认） | ~25 MB/s | 400kHz 初始模式 |
| HS SDR | `cap-mmc-highspeed`（SoC 默认） | 52 MB/s | 52MHz 单沿采样 |
| DDR52 | `mmc-ddr-1_8v` | 104 MB/s | 52MHz 双沿采样，ATK 板当前配置 |
| HS200 | `mmc-hs200-1_8v` | 200 MB/s | 200MHz 单沿采样，需 tuning |
| HS400 | `mmc-hs400-1_8v` | 400 MB/s | 200MHz 双沿采样，需 Enhanced Strobe |

内核按照从高到低的顺序尝试模式，失败后自动回退。尝试的结果直接体现在启动日志中：

```
HS400 成功: mmc1: new HS400 MMC card at address 0001
HS200 成功: mmc1: new HS200 MMC card at address 0001
DDR52 成功: mmc1: new DDR MMC card at address 0001     ← ATK 板的实际情况
HS SDR 成功: mmc1: new high speed MMC card at address 0001
```

ATK 板的实际日志：

```
[    4.052657] mmc1: new DDR MMC card at address 0001
```

说明 DTS 没有配 HS200/HS400，eMMC 运行在 DDR52（104MB/s）。如果产品需要更高带宽，在板级 DTS 追加 `mmc-hs200-1_8v` 即可，前提是信号完整性达标。

---

### 设备类型限制

```dts
non-removable;     /* eMMC 焊在板上，不需要轮询 */
no-sd;             /* 禁止尝试 SD 协议 */
no-sdio;           /* 禁止尝试 SDIO 协议 */
```

不加这些属性，内核会额外做无用功：

- 没有 `non-removable`：内核每隔一两秒发 CMD13 轮询卡是否还在
- 没有 `no-sd`：内核多花几百 ms 发 ACMD41 尝试 SD 协议
- 没有 `no-sdio`：同上，发 CMD5 尝试 SDIO 协议

对于焊死的 eMMC，这三条都应该配齐。

---

### st,neg-edge

```dts
st,neg-edge;    /* 下降沿采样 */
```

STM32MP2 的 SDMMC 控制器默认在时钟上升沿采样数据。但在高速模式下，信号延迟可能导致上升沿采样点不在数据眼图中心。`st,neg-edge` 让控制器改用下降沿采样，等效于将采样点推迟半个时钟周期。DDR52 及以上模式建议配这个属性。

---

### pinctrl：引脚复用与信号完整性

SDMMC2 的引脚定义分为两组，共 8-bit 数据线：

```dts
/* 低 4 位数据线 + 控制线 */
sdmmc2_b4_pins_a: sdmmc2-b4-0 {
    pins1 {
        pinmux = <STM32_PINMUX('E', 13, AF12)>, /* SDMMC2_D0 */
                 <STM32_PINMUX('E', 11, AF12)>, /* SDMMC2_D1 */
                 <STM32_PINMUX('E', 8,  AF12)>, /* SDMMC2_D2 */
                 <STM32_PINMUX('E', 12, AF12)>, /* SDMMC2_D3 */
                 <STM32_PINMUX('E', 15, AF12)>; /* SDMMC2_CMD */
        slew-rate = <1>;
        drive-push-pull;
        bias-pull-up;
    };
    pins2 {
        pinmux = <STM32_PINMUX('E', 14, AF12)>; /* SDMMC2_CK */
        slew-rate = <2>;                         /* CK 驱动强度更高 */
        drive-push-pull;
        bias-pull-up;
    };
};

/* 高 4 位数据线（8-bit 模式需要） */
sdmmc2_d47_pins_a: sdmmc2-d47-0 {
    pins {
        pinmux = <STM32_PINMUX('E', 10, AF12)>, /* SDMMC2_D4 */
                 <STM32_PINMUX('E',  9, AF12)>, /* SDMMC2_D5 */
                 <STM32_PINMUX('E',  6, AF12)>, /* SDMMC2_D6 */
                 <STM32_PINMUX('E',  7, AF12)>; /* SDMMC2_D7 */
        slew-rate = <1>;
        drive-push-pull;
        bias-pull-up;
    };
};
```

三个要点：

- **CK 单独分组**：时钟引脚 `slew-rate = <2>` 比数据线的 `<1>` 高。时钟是所有数据线的时序基准，驱动强度要更大。
- **`bias-pull-up`**：所有 SDMMC 信号都用了内部上拉，确保总线空闲时电平确定。
- **AF12**：SDMMC2 的功能复用是 AF12，不是 AF10。

---

### 板级完整配置

```dts
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

启动日志：

```
[    0.819824] mmci-pl18x 48230000.mmc: mmc1: PL180 manf 53 rev3 at 0x48230000 irq 65,0 (pio)
[    4.052657] mmc1: new DDR MMC card at address 0001       ← mmc-ddr-1_8v
[    4.053347] mmcblk1: mmc1:0001 58A43A 14.6 GiB
[    4.059543]  mmcblk1: p1 p2 p3 p4 p5 p6
[    4.061229] mmcblk1boot0: mmc1:0001 58A43A 4.00 MiB
[    4.066532] mmcblk1boot1: mmc1:0001 58A43A 4.00 MiB
[    4.070984] mmcblk1rpmb: mmc1:0001 58A43A 4.00 MiB, chardev (240:0)
```

日志与 DTS 属性的对应关系：

| DTS 属性 | 日志体现 |
|----------|---------|
| `pinctrl-*` | `mmci-pl18x 48230000.mmc: mmc1: PL180 manf 53 rev3` |
| `mmc-ddr-1_8v` | `new DDR MMC card` |
| `bus-width = <8>` | boot0/boot1/rpmb 各 4MB 正确枚举 |
| `non-removable` | 无 `card removed` 消息 |
| `no-sd` + `no-sdio` | 无 ACMD41/CMD5 相关日志 |

---

### 属性速查表

| 属性 | 作用 | 日志中如何体现 |
|------|------|---------------|
| `non-removable` | 不轮询卡检测 | 无 `card removed` 消息 |
| `no-sd` / `no-sdio` | 跳过 SD/SDIO 协议初始化 | 无 ACMD41/CMD5 超时日志 |
| `bus-width` | 数据线数量（1/4/8） | boot0/boot1/rpmb 正确枚举 |
| `mmc-ddr-1_8v` | 允许 DDR52 @1.8V | `DDR MMC card` |
| `mmc-hs200-1_8v` | 允许 HS200 @1.8V | `HS200 MMC card` |
| `mmc-hs400-1_8v` | 允许 HS400 @1.8V | `HS400 MMC card` |
| `st,neg-edge` | 下降沿采样 | 无直接日志，影响稳定性 |
| `vmmc-supply` | VCC 核心供电（3.3V） | 不配则无日志 |
| `vqmmc-supply` | VCCQ I/O 供电（1.8V） | 不配则 DDR52 无法使能 |

---

## 1.2 用户态操作

### 1.2.1 准备工作

操作 eMMC 需要以下工具。确认板子的根文件系统中已包含它们：

| 工具包 | 提供什么命令 | 为什么要用它 | Buildroot 配置项 |
|--------|-------------|-------------|----------------|
| `mmc-utils` | `mmc` | 读写 eMMC 内部寄存器（EXT_CSD）、查寿命、切启动分区 | `BR2_PACKAGE_MMC_UTILS` |
| `hdparm` | `hdparm` | 查看 eMMC 的写缓存是否开启 | `BR2_PACKAGE_HDPARM` |
| `e2fsprogs` | `mkfs.ext4` | 给新分区创建 ext4 文件系统 | `BR2_PACKAGE_E2FSPROGS`（无需子选项，`mkfs.ext4` 默认编译） |
| `dosfstools` | `mkfs.vfat` | 给新分区创建 FAT 文件系统（boot 分区常用）| `BR2_PACKAGE_DOSFSTOOLS` + `BR2_PACKAGE_DOSFSTOOLS_MKFS_FAT` |

在板子上验证各命令可用：

```bash
# mmc-utils
mmc --help 2>&1 | head -3
# hdparm（注意 --version 不支持，需要用 -V）
hdparm -V
# mkfs.ext4
mkfs.ext4 -V 2>&1 | head -1
```

实际输出：
```
# mmc --help 2>&1 | head -3
Usage:
        mmc extcsd read <device>
                Print extcsd data from <device>.

# hdparm -V
hdparm v9.65

# mkfs.ext4 -V 2>&1 | head -1
mke2fs 1.47.2 (1-Jan-2025)
```

---

### 1.2.2 识别 eMMC 设备

上电后，eMMC 和 SD 卡都表现为 `/dev/mmcblk*` 块设备。用 `ls -la` 可以列出所有 MMC 设备节点：

```bash
ls -la /dev/mmcblk*
```

实际输出：
```
brw-rw----    1 root     disk      179,  32 Jan  1  1970 /dev/mmcblk0
brw-rw----    1 root     disk      179,  33 Jan  1 01:47 /dev/mmcblk0p1
brw-rw----    1 root     disk      179,  34 Jan  1 01:47 /dev/mmcblk0p2
brw-rw----    1 root     disk      179,  35 Jan  1 01:47 /dev/mmcblk0p3
brw-rw----    1 root     disk      179,   0 Jan  1  1970 /dev/mmcblk1
brw-rw----    1 root     disk      179,  64 Jan  1  1970 /dev/mmcblk1boot0
brw-rw----    1 root     disk      179,  96 Jan  1  1970 /dev/mmcblk1boot1
brw-rw----    1 root     disk      179,   1 Jan  1 01:47 /dev/mmcblk1p1
brw-rw----    1 root     disk      179,   2 Jan  1 01:47 /dev/mmcblk1p2
brw-rw----    1 root     disk      179,   3 Jan  1 01:47 /dev/mmcblk1p3
brw-rw----    1 root     disk      179,   4 Jan  1 01:47 /dev/mmcblk1p4
brw-rw----    1 root     disk      179,   5 Jan  1 01:47 /dev/mmcblk1p5
brw-rw----    1 root     disk      179,   6 Jan  1 01:47 /dev/mmcblk1p6
crw-------    1 root     root      240,   0 Jan  1  1970 /dev/mmcblk1rpmb
```

如何区分哪个是 eMMC？对照启动日志：

```
[    4.053347] mmcblk1: mmc1:0001 58A43A 14.6 GiB
```

`mmcblk1` 就是 eMMC（14.6 GiB，host mmc1 = sdmmc2）。`mmcblk0` 是 SD 卡（59.5 GiB，host mmc0 = sdmmc1）。

eMMC 设备节点说明：

| 设备节点 | 作用 | 主次设备号 |
|---------|------|-----------|
| `/dev/mmcblk1` | eMMC 原始块设备（操作整个芯片） | 179, 0 |
| `/dev/mmcblk1p1` ~ `mmcblk1p6` | 硬件分区 1~6 | 179, 1~6 |
| `/dev/mmcblk1boot0` | 启动分区 0 | 179, 64 |
| `/dev/mmcblk1boot1` | 启动分区 1 | 179, 96 |
| `/dev/mmcblk1rpmb` | RPMB 安全分区（字符设备） | 240, 0 |

> `brw` = block device（块设备），`crw` = character device（字符设备）。RPMB 因为需要安全认证，不走普通块 I/O 路径，所以是字符设备。

注意事项：
- **eMMC 是 mmcblk1，不是 mmcblk0** — 初学者常在这搞混，写错设备会误伤 SD 卡
- 主设备号 179 是 `mmc_block` 驱动，240 是 `rpmb` 驱动
- 出现 `mmcblk1rpmb` 说明 eMMC 支持 RPMB 功能（不是所有 eMMC 都有）

---

### 1.2.3 查看容量与分区表

**总容量** — 两个途径查看：

```bash
# 方法一：启动日志（最直观）
dmesg | grep mmcblk1

# 方法二：sysfs（程序读容量的标准路径）
cat /sys/block/mmcblk1/size
```

实际输出：
```
# dmesg | grep mmcblk1
[    4.053347] mmcblk1: mmc1:0001 58A43A 14.6 GiB

# cat /sys/block/mmcblk1/size
30576640
```

`size` 的单位是**扇区数**（1 扇区 = 512 字节），换算：30576640 × 512 ÷ 1024³ = 14.58 GiB，与日志一致。

> **为什么不用 `fdisk -l` 看总容量？** 这里有个坑——ATK 板 eMMC 的 `fdisk -l` 输出的总容量数值有误（显示 2642M 与实际 14.6 GiB 不符），这是 fdisk 版本的小 bug。**以 `dmesg` 和 `/sys/block/mmcblk1/size` 为准。**

**分区表**：

```bash
fdisk -l /dev/mmcblk1
```

实际输出：
```
Found valid GPT with protective MBR; using GPT

Disk /dev/mmcblk1: 30576640 sectors, 2642M
Logical sector size: 512
Disk identifier (GUID): 4fbdfe23-ebed-42a7-a1f6-ef7cf23190cc
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 30576606

Number  Start (sector)    End (sector)  Size Name
     1            1024            2047  512K metadata1
     2            2048            3071  512K metadata2
     3            3072           11263 4096K fip-a
     4           11264           20479 4608K fip-b
     5           20480          413695  192M bootfs
     6          413696        30576606 14.3G rootfs
```

这个分区表是典型的 A/B 双槽启动布局：

| 分区 | 大小 | 内容 | 说明 |
|------|------|------|------|
| `metadata1` | 512K | TF-A metadata | A/B 启动状态跟踪，标记当前活动槽位 |
| `metadata2` | 512K | metadata 备份 | 冗余备份，防止 metadata 损坏导致变砖 |
| `fip-a` | 4M | FIP 镜像 A | FIP = TF-A BL2 + OP-TEE + U-Boot，槽位 A |
| `fip-b` | 4608K | FIP 镜像 B | 槽位 B，略大（可能包含额外 firmware） |
| `bootfs` | 192M | 内核 + DTB + extlinux | 存放 Linux 内核镜像和设备树 |
| `rootfs` | 14.3G | 根文件系统 | ext4，系统主体 |

> `metadata` 和 `fip` 的分区是不能直接 `mount` 的——它们存的是裸二进制镜像，不是文件系统。只有 `bootfs` 和 `rootfs` 是可挂载的块分区。

---

### 1.2.4 重新分区

开发现场通常不需要 eMMC 的 A/B 启动布局，我们把它清空，做成一个单分区数据盘。

> **⚠️ 以下操作会完全擦除 eMMC 上的所有数据（包括出厂带的 A/B 启动分区布局）。**
> 操作前确认 eMMC 上没有需要保留的数据。

ATK 板的 `fdisk` 是 BusyBox 版（不支持 GPT），所以使用 MBR 分区表：

```
# fdisk /dev/mmcblk1

Found valid GPT with protective MBR; using GPT

Command (m for help): o                    ← 新建 DOS(MBR) 分区表
Building a new DOS disklabel. Changes will remain in memory only,
until you decide to write them. After that the previous content
won't be recoverable.

The number of cylinders for this disk is set to 1895.
...

Command (m for help): n                    ← 新建分区
Partition type
   p   primary partition (1-4)
   e   extended
p                                          ← 主分区
Partition number (1-4): 1                  ← 分区号 1
First sector (63-30576639, default 63):
Using default value 63
Last sector ... (63-30576639, default 30576639):
Using default value 30576639

Command (m for help): w                    ← 写入并退出
The partition table has been altered.
Calling ioctl() to re-read partition table
[ 1476.799716]  mmcblk1: p1
```

> BusyBox `fdisk` 不用 GPT 的 `g` 命令，而是用 `o` 创建 MBR。MBR 的第一个扇区从 63 开始（不是 GPT 的 2048），这是历史兼容原因，对性能基本无影响。

确认分区：

```bash
fdisk -l /dev/mmcblk1
```

实际输出：
```
Disk /dev/mmcblk1: 15 GB, 15655239680 bytes, 30576640 sectors
1895 cylinders, 256 heads, 63 sectors/track
Units: sectors of 1 * 512 = 512 bytes

Device       Boot StartCHS    EndCHS        StartLBA     EndLBA    Sectors  Size Id Type
/dev/mmcblk1p1    0,1,1       1023,255,63         63   30576639   30576577 14.5G 83 Linux
```

> `Id 83` = Linux native 分区类型，`StartLBA = 63` 是 MBR 的起始扇区。

---

### 1.2.5 格式化与挂载

创建 ext4 文件系统：

```bash
mkfs.ext4 -L emmc-data /dev/mmcblk1p1
```

实际输出：
```
mke2fs 1.47.2 (1-Jan-2025)
Discarding device blocks: done
Creating filesystem with 3822072 4k blocks and 956592 inodes
Filesystem UUID: c3bcb21d-09d9-4c9e-ae9d-fc78844251d3
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
```

挂载到系统：

```bash
mkdir -p /mnt/emmc-data
mount /dev/mmcblk1p1 /mnt/emmc-data
df -h | grep mmcblk1
```

实际输出：
```
[ 1624.913471] EXT4-fs (mmcblk1p1): mounted filesystem ... r/w with ordered data mode.
/dev/mmcblk1p1           14.2G      2.0M     13.5G   0% /mnt/emmc-data
```

现在 eMMC 就是一个普通的数据盘了：

```bash
echo "hello eMMC" > /mnt/emmc-data/test.txt
cat /mnt/emmc-data/test.txt
df -h /mnt/emmc-data
```

实际输出：
```
# echo "hello eMMC" > /mnt/emmc-data/test.txt
# cat /mnt/emmc-data/test.txt
hello eMMC
# df -h /mnt/emmc-data
Filesystem                Size      Used Available Use% Mounted on
/dev/mmcblk1p1           14.2G      2.0M     13.5G   0% /mnt/emmc-data
```

取消挂载：

```bash
umount /mnt/emmc-data
```

实际输出（内核日志）：
```
[ 1814.826913] EXT4-fs (mmcblk1p1): unmounting filesystem ...
```

> 重启后挂载点会消失，在 TFTP+NFS 开发环境下，每次重启后重新 `mount` 即可。

---

### 1.2.6 性能测试

eMMC 性能测试需要关注三个指标：顺序读、顺序写、写缓存影响。

**准备**：重新挂载 eMMC（如果已经 umount 了）

```bash
mount /dev/mmcblk1p1 /mnt/emmc-data
```

#### 顺序读（raw device）

```bash
time dd if=/dev/mmcblk1 of=/dev/null bs=1M count=500
```

```
500+0 records in
500+0 records out
real    0m 6.04s
user    0m 0.00s
sys     0m 1.16s
```

速率 = 500MB ÷ 6.04s ≈ **82.8 MB/s**

#### 顺序写（通过文件系统，直写绕过缓存）

```bash
time dd if=/dev/zero of=/mnt/emmc-data/test.bin bs=1M count=200 oflag=direct
```

```
200+0 records in
200+0 records out
real    0m 3.02s
user    0m 0.00s
sys     0m 0.18s
```

速率 = 200MB ÷ 3.02s ≈ **66.2 MB/s**

> `oflag=direct` 绕过内核 page cache，直接写 eMMC，测得的是真实写性能。不加这个参数测的是内存写速度，不是 eMMC 的。

#### 读回（同一文件）

```bash
time dd if=/mnt/emmc-data/test.bin of=/dev/null bs=1M
```

```
200+0 records in
200+0 records out
real    0m 2.38s
user    0m 0.00s
sys     0m 0.39s
```

速率 = 200MB ÷ 2.38s ≈ **84.0 MB/s**

#### 两个缓存层，先说清楚

写 eMMC 时数据穿过两个独立的缓存层，顺序是：

```
应用程序 dd
  │ write()
  ↓
① 内核 Page Cache（DDR 内存）   ← sync 管的是这一层
  │ 内核后台回写 (flush 线程)
  ↓
② eMMC 内部 Cache（SLC NAND）  ← mmc cache disable 关的是这一层
  │ eMMC 固件后台搬运
  ↓
   NAND 主存储区（TLC/MLC）
```

**内核 Page Cache**：所有文件 I/O（不指定 `O_DIRECT`）都先写 DDR 内存，内核标记为脏页（dirty pages），后台异步刷到块设备。`sync` 命令做的就是：**立即把所有脏页刷到块设备，等刷完才返回。**

**eMMC 内部 Cache**：数据到达 eMMC 芯片后，如果内部缓存开启（`CACHE_CTRL=0x01`），eMMC 先收到 SLC 缓存就返回完成信号，后台再搬运到主存储区。

#### 不带 oflag=direct 的 dd + sync

```bash
time dd if=/dev/zero of=/mnt/emmc-data/test2.bin bs=1M count=200
time sync
```

```
# dd：只写内存，不落盘
200+0 records in
200+0 records out
real    0m 0.75s        ← 0.75s 是因为数据只写到内核 page cache（DDR），没到 eMMC

# sync：强制刷脏页到 eMMC
real    0m 0.01s        ← 0.01s 说明内核后台回写线程已经提前把脏页刷走了
```

关键理解：

| 操作 | 数据到了哪 | 耗时 |
|------|-----------|------|
| `dd`（不带 direct） | 内核 page cache（DDR 内存） | 0.75s |
| 内核后台回写线程 | page cache → eMMC 芯片 | 在 dd 执行期间已自动完成 |
| `sync` | 确认所有脏页已到 eMMC | 0.01s（只剩少量残留） |

> `sync` 返回后数据**肯定已经离开了 DDR 到了 eMMC 芯片**。但到没到 NAND 主存储区，取决于 eMMC 内部缓存是否开启（`CACHE_CTRL`）。这是两个独立的问题：`sync` 管的是第①层，`CACHE_CTRL` 管的是第②层。`oflag=direct` 只跳过了第①层（page cache），数据仍然会经过第②层（eMMC 内部缓存）。测真实 NAND 速度需要先确认 eMMC 内部缓存状态。

#### 汇总

| 场景 | 速率 |
|------|------|
| 顺序读（raw device） | 82.8 MB/s |
| 顺序读（文件系统） | 84.0 MB/s |
| 顺序写（`oflag=direct`） | 66.2 MB/s |
| DDR52 理论极限 | 104 MB/s |

清理测试文件：

```bash
rm /mnt/emmc-data/test.bin /mnt/emmc-data/test2.bin
```

> 以上数据只代表顺序大块传输。实际文件系统性能还受 4K 随机读写影响，那些需要 `fio` 才能测准。

---

### 1.2.7 健康诊断

eMMC 内部有一块 EXT_CSD 寄存器区域，记录了芯片的寿命、配置、特性支持等信息。用 `mmc extcsd read` 读取：

```bash
mmc extcsd read /dev/mmcblk1
```

输出较长，以下是关键字段的分析。

#### 芯片基本信息

```
Extended CSD rev 1.8 (MMC 5.1)
Sector Count [SEC_COUNT: 0x01d29000]   = 30576640 扇区 = 14.6 GiB
Cache Size [CACHE_SIZE] is 8192 KiB    = 8 MiB 缓存
```

#### 寿命与健康状态

```
eMMC Life Time Estimation A [EXT_CSD_DEVICE_LIFE_TIME_EST_TYP_A]: 0x01
  0x01 = 0%~10% lifetime used  ← 几乎全新
eMMC Life Time Estimation B [EXT_CSD_DEVICE_LIFE_TIME_EST_TYP_B]: 0x00
  0x00 = 未定义（仅适用于 TLC/MLC 的 B 类）
eMMC Pre EOL information [EXT_CSD_PRE_EOL_INFO]: 0x01
  0x01 = Normal          ← 正常
  0x02 = Warning（寿命消耗 > 90%）
  0x03 = Critical（濒死）
```

> 这块 eMMC 几乎还是全新的，寿命消耗 < 10%。

#### 缓存

```
Control to turn the Cache ON/OFF [CACHE_CTRL]: 0x01
  0x00 = OFF     0x01 = ON ✓
```

eMMC 内部有 8 MiB 的 SLC 缓存，当前已开启。但要注意：**这是 eMMC 芯片内部的缓存，不是内核的文件系统缓存（page cache）。** 两者是独立的两层：

| 缓存层 | 位置 | 谁管理 | 掉电风险 | 用户态控制方式 |
|--------|------|--------|---------|-------------|
| 内核 page cache | DDR 内存 | Linux 内核 | 低（内核回写策略成熟） | `oflag=direct` 绕过 |
| eMMC 内部 Cache | eMMC 芯片内部 SLC | eMMC 控制器固件 | **中高**（芯片缓存易失） | `mmc cache disable/enable` |

内核 page cache 是操作系统层面的缓存——读文件时数据留在 DDR 内存中加速下次访问，写文件时先写内存再异步刷盘。`dd` 的 `oflag=direct` 跳过的是这一层，让数据直接发给 eMMC。

eMMC 内部 Cache 是芯片层面的缓存——数据到达 eMMC 后先写入 SLC 缓存就返回 ACK，芯片后台异步搬到大容量 TLC/MLC 区域。如果掉电时还没搬完，数据就丢了。

> 实测 `hdparm -W /dev/mmcblk1` 在 ATK 板上返回空输出——`hdparm` 是 ATA 命令集工具，对 eMMC 不生效。判断 eMMC 缓存状态以 `mmc extcsd read` 为准。

#### 支持的速率（硬件能力 vs 当前配置）

```
Card Type [CARD_TYPE: 0x57]
  HS400 Dual Data Rate eMMC @200MHz 1.8VI/O    ← 硬件支持 HS400
  HS200 Single Data Rate eMMC @200MHz 1.8VI/O   ← 硬件支持 HS200
  HS Dual Data Rate eMMC @52MHz 1.8V or 3VI/O   ← DDR52
  HS eMMC @52MHz
  HS eMMC @26MHz

High-speed interface timing [HS_TIMING: 0x01]   ← 当前运行在 HS 模式
```

当前跑的是 DDR52（`HS_TIMING: 0x01`），但硬件支持 HS200/HS400——DTS 没配所以没用上。

#### 启动分区

```
Boot partition size [BOOT_SIZE_MULTI: 0x20]
  = 32 × 128 KiB = 4 MiB  ← boot0/boot1 各 4MB

Boot configuration bytes [PARTITION_CONFIG: 0x50]
  Boot Partition 2 enabled                      ← 当前从 boot1 启动
  No access to boot partition                   ← 启动分区未映射到用户空间
```

#### RPMB

```
RPMB Size [RPMB_SIZE_MULT]: 0x20
  = 32 × 128 KiB = 4 MiB
```

#### 命令队列

```
Command Queue Support [CMDQ_SUPPORT]: 0x01      ← 支持
Command Queue Depth [CMDQ_DEPTH]: 32            ← 队列深度 32
Command Enabled [CMDQ_MODE_EN]: 0x00            ← 当前未启用
```

> CMDQ 是 eMMC 5.0 引入的硬件命令队列，当前内核未启用。

#### 关键问题速查

用这条命令可以快速提取健康状态：

```bash
mmc extcsd read /dev/mmcblk1 | grep -E 'LIFE_TIME|PRE_EOL|CACHE_CTRL'
```

```
eMMC Life Time Estimation A [...]: 0x01
eMMC Life Time Estimation B [...]: 0x00
eMMC Pre EOL information [...]: 0x01
Control to turn the Cache ON/OFF [CACHE_CTRL]: 0x01
```

---

### 1.2.8 启动分区操作

#### eMMC 内部硬件分区模型

eMMC 芯片在 NAND 闪存内部划分了多个**硬件分区**（Hardware Partitions），由 eMMC 控制器硬件独立管理，相互之间擦除和写保护都是隔离的：

```
eMMC 芯片内部布局（概念图）:

┌─────────────────────────────────────┐
│  Boot Partition 1  (boot0)          │  4 MiB  ← CPU 上电从这里读固件
├─────────────────────────────────────┤
│  Boot Partition 2  (boot1)          │  4 MiB  ← 冗余备份 / A/B 槽位
├─────────────────────────────────────┤
│  RPMB Partition                     │  4 MiB  ← 安全认证（字符设备）
├─────────────────────────────────────┤
│  User Data Area                     │  ~14.5 GiB  ← 我们看到的 mmcblk1
│  ├─ GPA (General Purpose Area) 1~4  │          ← 可选，一般不用
│  └─ UDA (User Data Area)            │          ← GPT 分区表 + 文件系统
└─────────────────────────────────────┘
```

##### 为什么需要独立的启动分区？

SoC（如 STM32MP257）上电时，**CPU 核心还没有任何软件在运行**。片上 Boot ROM 负责从某个外设加载第一级 bootloader。如果 bootloader 放在用户分区的 ext4 文件系统中，Boot ROM 需要：

1. 初始化 MMC 总线 → ② 解析 GPT 分区表 → ③ 识别 ext4 超级块 → ④ 遍历目录找到 bootloader → ⑤ 读取到 SRAM

这对片上 ROM 来说**太复杂了**（ROM 不能有 bug，不能更新）。独立启动分区的设计把流程简化为：

```
Boot ROM 从 eMMC boot 分区加载固件:
  ① 根据 BOOT 引脚电平决定从哪个外设启动
  ② 向 eMMC 发 CMD0(参数 = BOOT_MODE)
  ③ eMMC 硬件将 Boot Partition 的内容连续输出到 DAT 线
  ④ Boot ROM 读取固定字节数，校验，跳转

不需要文件系统，不需要分区表，不需要驱动栈。
```

启动分区的内容就是**裸二进制镜像**——ATK 板上的 `tf-a-bl2.bin` 最终就放在这里。

##### 实践中的角色

| 场景 | 启动分区的角色 |
|------|--------------|
| SD 卡启动（ATK 板当前方式） | 不涉及 eMMC 启动分区，由 SD 卡的 MBR 引导 |
| eMMC 启动（产品量产） | TF-A BL2 存在 eMMC boot1，Boot ROM 直接裸读 |
| A/B 安全启动 | boot0 存 A 版本固件，boot1 存 B 版本，OTA 切换 |

ATK 板的完整启动链（未来从 eMMC 启动时）：

```
Boot ROM → eMMC boot 分区 → TF-A BL2 → OP-TEE → U-Boot → Linux
             (tf-a-bl2.bin)      |          |        |
                              BL32      BL33     Kernel + DTB
                                                    |
                                                 存用户分区 bootfs
```

##### 硬件分区 vs 软件分区（GPT）

新手最容易混淆这两个概念：

| | 硬件分区（eMMC 内置） | 软件分区（GPT/MBR） |
|--|---------------------|-------------------|
| 管理方 | eMMC 控制器硬件 | 操作系统（内核 + fdisk） |
| 配置位置 | EXT_CSD 寄存器 | 用户数据区的前 34~128 个扇区 |
| 数量 | 固定：Boot×2 + RPMB + GPA×4 + UDA | 最多 128 个 |
| 可见性 | 独立 mmcblk 设备节点 | mmcblk1pN |
| 用途 | 启动、安全存储 | 文件系统分区 |

有了这个基础，再来看实际寄存器操作就清晰了。

#### 读取启动相关寄存器

所有启动配置集中在 EXT_CSD 的几个字节中，一条命令全拉出来：

```bash
mmc extcsd read /dev/mmcblk1 | grep -E 'BOOT_CONFIG_PROT|PARTITION_CONFIG|BOOT_BUS_COND|BOOT_SIZE_MULTI|RST_N_FUNC|BOOT_WP'
```

实际输出：
```
Boot configuration bytes [PARTITION_CONFIG: 0x50]
Boot configuration protection [BOOT_CONFIG_PROT: 0x00]
Boot bus conditions [BOOT_BUS_COND: 0x01]
Boot write protection status registers [BOOT_WP_STATUS]: 0x00
Boot partition size [BOOT_SIZE_MULTI: 0x20]
RST_n function [RST_N_FUNC: 0x01]
```

所有值都是 `0x00` 或合理的默认值——这块 eMMC 还没被改动过启动配置。

---

#### PARTITION_CONFIG 寄存器拆解（byte 179）

`0x50` = `0b01010000`，这是整个启动控制的核心字节：

```
Byte 179 = [PARTITION_CONFIG]

Bit[7]     Bit[6]       Bit[5:3]                  Bit[2:0]
rsvd       BOOT_ACK     PARTITION_ACCESS          BOOT_PARTITION_ENABLE
  0          1            000 = 无访问              000 = 禁用启动
                          001 = R/W boot1           001 = boot1
                          010 = R/W boot2           010 = boot2
                          011 = R/W RPMB            111 = 用户分区

0x50 = 0b01010000:
  BOOT_ACK              = 1   ← 上电时 eMMC 发送 boot acknowledge
  PARTITION_ACCESS      = 000 ← 当前无分区映射（boot0/boot1 未暴露）
  BOOT_PARTITION_ENABLE = 000 ← 从用户分区启动，非 boot 分区
```

四个要点：

| bit 区域 | 职责 | 可逆？ |
|----------|------|--------|
| `BOOT_PARTITION_ENABLE` (bit[2:0]) | 告诉 eMMC 上电后从哪读固件 | ✅ 随时改 |
| `BOOT_ACK` (bit[6]) | 上电后 eMMC 是否先发 ACK 再传输数据 | ✅ 随时改 |
| `PARTITION_ACCESS` (bit[5:3]) | 把 boot0/boot1/RPMB 映射到 `/dev/mmcblk1` 用户空间 | ✅ 随时改，访问完改回 000 |
| 整个 byte 179 | 以上三项的容器 | ⚠️ 受 `BOOT_CONFIG_PROT` 约束 |

---

#### ⚠️ BOOT_CONFIG_PROT — 永久锁定 byte 178

```
[BOOT_CONFIG_PROT: 0x00]  ← 当前未锁定

0x00 = 可自由修改 byte 177/179/162
0x01 = 永久锁定（非 OTP，Write Once）

一旦写 0x01：
  - byte 177 (BOOT_BUS_COND)    ← 冻结
  - byte 162 (RST_N_FUNC)       ← 冻结
  - byte 179 (PARTITION_CONFIG) ← 冻结
  - 以上三个寄存器再也不能改，包括 0x01 → 0x00 也不行
```

> **⚠️ 永远不要对 `BOOT_CONFIG_PROT` 写 `0x01`。** ATK 板上的这块 eMMC 当前是 `0x00`（未锁定），这是可逆操作的最后防线。一旦锁定，启动配置就焊死在芯片里了，再也改不了。

---

#### 映射 boot 分区到用户空间（参考命令）

有些场景需要读/写 boot 分区内容（如烧录 U-Boot SPL）。通过 `PARTITION_ACCESS` 将 boot 分区映射到 `/dev/mmcblk1`：

```
PARTITION_ACCESS = 000 → /dev/mmcblk1  = 用户分区（上电默认）
PARTITION_ACCESS = 001 → /dev/mmcblk1  = boot1
PARTITION_ACCESS = 010 → /dev/mmcblk1  = boot2
PARTITION_ACCESS = 011 → /dev/mmcblk1  = RPMB
```

对应关系决定了 `/dev/mmcblk1` 实际指向哪个物理分区。映射期间，`dd` 读写的就是映射后的分区，不是用户分区。

使能访问的等效 EXT_CSD 命令（**参考，不实操**）：

```
# 映射 boot1 到 mmcblk1（用户分区消失，mmcblk1 = boot1）
mmc bootpart enable 1 1 /dev/mmcblk1

# 等效于 EXT_CSD 写入:
#   PARTITION_ACCESS[5:3] ← 001
#   需要 mmc-utils 编译时开启 ENABLE_DANGEROUS_COMMANDS
```

`bootpart enable` 的两个参数：
- 第一个 `1` → BOOT_PARTITION_ENABLE = boot1（上电从 boot1 启动）
- 第二个 `1` → PARTITION_ACCESS = boot1（当前映射 boot1）

操作后验证：

```bash
mmc extcsd read /dev/mmcblk1 | grep PARTITION_CONFIG
# Boot configuration bytes [PARTITION_CONFIG: 0x49]
# 0x49 = 0b01001001 → ACCESS = boot1, ENABLE = boot1
```

> **操作完记得改回 `0x00`**——把 `PARTITION_ACCESS` 恢复为 000，否则 `/dev/mmcblk1p1` 等用户分区全部"消失"。

---

#### 从 eMMC 启动的概念流程（理论）

ATK 板当前从 SD 卡启动，不走 eMMC 的 boot 分区。如果想改为从 eMMC 启动，需要三步：

```
① 烧录固件到 boot 分区:
   mmc bootpart enable 1 1 /dev/mmcblk1
   dd if=tf-a-bl2.bin of=/dev/mmcblk1 bs=512
   → 固件写入 boot1

② 配置启动参数:
   BOOT_PARTITION_ENABLE ← boot1 (bit[2:0] = 001)
   BOOT_BUS_COND ← 设置启动时的总线宽度和时钟模式

③ 切换板子拨码开关:
   BOOT0=ON, BOOT1=ON, BOOT2=OPEN, BOOT3=OPEN
   → 从 eMMC 启动
```

这是固件烧录的概念流程，不是 ATK 板的当前操作。ATK 板现阶段保持 SD 卡启动。

---

#### 启动分区操作安全总结

| 操作 | 风险 | 能否改回 |
|------|------|---------|
| `mmc extcsd read` 查看启动配置 | 无 | — |
| 修改 `PARTITION_ACCESS` 映射分区 | 低（改回即可） | ✅ |
| 修改 `BOOT_PARTITION_ENABLE` | 低（ATK 从 SD 卡启动，不受影响） | ✅ |
| 修改 `BOOT_BUS_COND` | 低 | ✅（除非 byte 178 已锁） |
| 对 boot 分区 `dd` 写入 | 低（只影响 boot 分区数据） | ✅ |
| 写 `BOOT_CONFIG_PROT` = `0x01` | **永久** | ❌ |

一条底线：**`BOOT_CONFIG_PROT` 永远保持 `0x00`**，其他启动配置都可以自由探索。

---

### 1.2.9 RPMB 安全分区

#### 为什么需要 RPMB？

eMMC 存储的数据有两类安全威胁是普通块设备防不了的：

| 攻击方式 | 什么意思 | 例子 |
|---------|---------|------|
| **篡改** | 攻击者直接改掉存储的数据 | 改掉安全启动状态，加载旧版有漏洞的固件 |
| **重放** | 攻击者记录下某时刻的有效数据，之后再写回去 | 计费场景：写入"还剩 100 次" → 用完 100 次 → 重新写入"还剩 100 次"（应该是 0） |

普通块设备没有认证机制，谁拿到硬件谁就能读写。RPMB 的解决方案是**认证写 + 单调递增计数器**：

- 每次写入必须附带 HMAC-SHA256 签名（密钥在 eMMC 出厂前烧录，不可读）
- 每次写入后 Write Counter +1，读的时候也会返回当前 Counter 值
- 攻击者即使捕获了写入的报文，重放时 Counter 不匹配，eMMC 拒绝

#### 在启动链中的角色

STM32MP2 使用 RPMB 存储安全启动的信任状态：

```
Boot ROM → TF-A BL2 → OP-TEE → U-Boot → Linux
                         │
                 OP-TEE 通过 RPMB 存储:
                   - 安全启动状态 (滚回防护)
                   - OEM 密钥
                   - 计费/授权计数
```

RPMB 的操作不经过 Linux 内核的块层，而是由 OP-TEE 安全内核通过 eMMC 的 RPMB 协议直接操作。Linux 用户空间看到的 `/dev/mmcblk1rpmb` 只是一个 MMIO 映射接口，真正的认证逻辑在 TEE 侧。

#### 查看 RPMB 配置

RPMB 的大小在 EXT_CSD 中：

```bash
mmc extcsd read /dev/mmcblk1 | grep RPMB_SIZE_MULT
```

```
RPMB Size [RPMB_SIZE_MULT]: 0x20    ← 32 × 128 KiB = 4 MiB
```

单位是 128 KiB。`0x20 = 32`，ATK 板这块 eMMC 的 RPMB 分区是 4 MiB。

设备节点（来自 1.2.2 节）：

```
crw-------    1 root     root      240,   0 Jan  1  1970 /dev/mmcblk1rpmb
```

`crw` 表示字符设备（不是块设备），主设备号 240 是内核 RPMB 驱动。所以不能用 `dd` 或 `mount` 直接操作它。

#### 认证密钥（Authentication Key）

RPMB 使用前需要先写入认证密钥，这个操作**只能做一次**（除非 Security Partition 被销毁重新配置）：

```
流程:
  ① 生成 32 字节 HMAC-SHA256 密钥
  ② mmc rpmb write-key /dev/mmcblk1rpmb <key-file>
  ③ eMMC 内部永久存储密钥，计数器从 0 开始

  之后:
  密钥不可读，RPMB 进入 "已认证" 状态
```

ATK 板没有执行过这一步——这块 eMMC 的 RPMB 当前是未认证状态，不可写。

#### 读写操作概念

用户空间通过 `mmc rpmb` 子命令操作 RPMB：

```bash
# 不实操，仅展示命令格式
mmc rpmb read-block /dev/mmcblk1rpmb <addr> <size> <out-file>
mmc rpmb write-block /dev/mmcblk1rpmb <addr> <size> <in-file>
```

每条 RPMB 命令内部自动附带：
- 当前 Write Counter
- HMAC-SHA256 签名（使用已写入的密钥）
- 帧类型标记（Request/Response、Read/Write/Get Counter 等）

下图是一次 RPMB 写操作的协议交互：

```
Host                                    eMMC
  │                                       │
  │── RPMB Write Request ────────────────→│  数据 + Counter + HMAC
  │                                       │  验证 HMAC、检查 Counter
  │                                       │  写入 NAND、Counter++、生成响应 HMAC
  │←── RPMB Write Response ──────────────│  状态 + Counter + HMAC
  │                                       │
  │── RPMB Read Counter Request ─────────→│  读验证：用写入的密钥签名请求
  │←── RPMB Read Counter Response ───────│  返回当前 Counter 值
```

> 实际产品中 RPMB 的操作由安全组件完成（TF-A/OP-TEE），不直接在 Linux 用户空间调 `mmc rpmb`。这里列出的命令用于工厂烧录和生产测试。

#### 一句话总结

| 状态 | 说明 |
|------|------|
| ATK 板 | RPMB 存在（4 MiB），无认证密钥，Linux 用户空间不可写 |
| 量产场景 | 工厂烧录阶段写入密钥，OP-TEE 使用 RPMB 存储安全启动状态 |
| 调试 | `mmc extcsd read` 查看大小，`mmc rpmb` 操作需要密钥才可用 |

RPMB 是 eMMC 中"用不上但在产品里很重要的那部分"——开发阶段用不到，产品量产时必须配。

---

### 1.2.10 eMMC 内部写缓存

#### 为什么有这个东西？

eMMC 内部 NAND 的写操作比读慢得多（~10-500ms vs ~1ms），因为写需要擦除再编程。为了吸收这个延迟差距，eMMC 芯片内部有一个 SLC 缓存（实测 8 MiB），写入的数据先进入 SLC 缓存就返回，芯片后台异步搬到大容量 TLC/MLC 区域。

这个机制对主机完全透明——内核看到的只有"写完返回了"，不知道数据还没落盘。

#### 检查当前状态

通过 EXT_CSD 查看（实际硬件输出）：

```bash
mmc extcsd read /dev/mmcblk1 | grep -E 'CACHE_CTRL|CACHE_SIZE'
```

```
Cache Size [CACHE_SIZE] is 8192 KiB
Control to turn the Cache ON/OFF [CACHE_CTRL]: 0x01
```

`0x01` 表示缓存**已开启**。

> 注意：`hdparm -W` 是 ATA 命令，对 MMC 设备不生效。实测 ATK 板上 `hdparm -W /dev/mmcblk1` 返回空输出，不要用这个命令判断 eMMC 的缓存状态。

#### 开关与使用场景

通过 `mmc` 工具写入 EXT_CSD byte 183 来开关：

```bash
# 关闭 eMMC 内部缓存
mmc cache disable /dev/mmcblk1

# 开启
mmc cache enable /dev/mmcblk1
```

关闭后所有写操作直接到 NAND 主存储区，延迟变长但掉电安全。不用重启，立即生效。

| 场景 | 建议 | 原因 |
|------|------|------|
| 开发调试 | 保持开启 | 性能好就够了，数据丢了也无所谓 |
| 普通嵌入式产品 | 保持开启 | 关机有 flush 流程，掉电概率低 |
| **工业/车载/关键设备** | **关闭** | 意外掉电时缓存数据丢失可能导致文件系统损坏 |

> ATK 板是开发板，保持默认开启即可。缓存关闭是产品阶段（尤其是工业认证）才需要考虑的事。标准兼容的 eMMC 芯片掉电后会自动丢弃写缓存内容且不通知主机，这是文件系统损坏的常见隐蔽原因。

---

## 1.3 sysfs 接口

sysfs 是内核向用户空间暴露运行时状态的主要途径。eMMC 相关的节点分布在三个路径：

| 路径 | 有什么 | 典型用途 |
|------|--------|---------|
| `/sys/block/mmcblk1/` | 块设备层：I/O 统计、队列参数、调度器 | 看读写负载、调队列策略 |
| `/sys/bus/mmc/devices/` | MMC 总线层：寿命、厂家、CID、日期 | 快速健康检查 |
| `/sys/class/mmc_host/mmc1/` | host 控制器层：CMDQ 状态 | 查硬件队列是否启用 |

### 1.3.1 块设备层统计

`/sys/block/mmcblk1/stat` 记录所有已完成 I/O 的累计计数，是 `iostat` 的数据源。实测 ATK 板 eMMC 当前状态：

```bash
cat /sys/block/mmcblk1/stat
```

```
   395    0    21165    317    0    0    0    0    0    196    317    0    0    0    0    0    0
```

前 11 个字段含义：

| # | 字段 | 实测值 | 单位 | 说明 |
|---|------|--------|------|------|
| 1 | 读完成次数 | 395 | 次 | 成功完成的读请求数 |
| 2 | 读合并次数 | 0 | 次 | 相邻请求合并的次数 |
| 3 | 读扇区数 | 21165 | 扇区（512B） | 总共读了多少数据，÷2 = KB |
| 4 | 读耗时 | 317 | ms | | 
| 5 | 写完成次数 | 0 | 次 | ATK 板当前未对 eMMC 做写操作 |
| 6 | 写合并次数 | 0 | 次 |
| 7 | 写扇区数 | 0 | 扇区 |
| 8 | 写耗时 | 0 | ms |
| 9 | 排队中 I/O | 0 | 个 | 当前正在处理的请求数 |
| 10 | I/O 总耗时 | 196 | ms | 含排队时间的总耗时 |
| 11 | 加权 I/O 总耗时 | 317 | ms | |

> 写次数全 0 说明板子从 SD 卡启动后 eMMC 只被读了启动日志，没有被写过。这个文件是理解 block layer 行为的第一手来源，`iostat -x 1` 的数据就来自这里。

### 1.3.2 队列参数

队列参数在 `/sys/block/mmcblk1/queue/` 下，这里只讲两个最常用的：I/O 调度器和写缓存策略。

#### I/O 调度器（scheduler）

先理解**为什么需要 I/O 调度器**：多个进程同时读写块设备时，请求会到达 block layer。调度器决定这些请求的执行顺序。

```
机械硬盘时代（核心矛盾：磁头寻道）:
  进程 A 读 LBA=100, 进程 B 读 LBA=90000
  调度器做的: 把 A 和 B 的请求按 LBA 排序
  效果: 磁头从 100 读到 90000，只需要移动一次，而不是来回跑

eMMC/SD 时代（核心矛盾：NAND 不能原地更新）:
  eMMC 内部 FTL 自己会管逻辑地址到物理地址的映射
  调度器能做的: 合并相邻扇区的请求（减少命令数量），但"电梯算法"省寻道的优化没用
```

查看当前调度器（实测 ATK 板）：

```bash
cat /sys/block/mmcblk1/queue/scheduler
```

```
none [mq-deadline] kyber bfq
```

方括号里是当前生效的——`mq-deadline`。共有四个候选项，各有用处：

| 调度器 | 核心思路 | 适合场景 | eMMC 上值不值？ |
|--------|---------|---------|----------------|
| `mq-deadline`（默认） | 按 LBA 排序 + 设置读写截止时间，避免写饿死读 | **通用默认**，不知道选什么就这个 | ✅ 安全，基本无脑用 |
| `none` | 直通，进来就发，不做排序不做合并 | **测试基准**，测设备极限性能 | ✅ 适合，eMMC 自带 FTL 不需要内核排序 |
| `kyber` | 监控每个进程的 I/O 延迟，动态调整并发数，保证延迟公平 | **延迟敏感场景**（音频、车机） | ⚠️ 看需求 |
| `bfq` | 按进程权重分配磁盘带宽，类似 CPU 的 CFQ | **多进程并发且要公平** | ❌ 计算开销大，eMMC 没必要 |

运行时切换：

```bash
echo none > /sys/block/mmcblk1/queue/scheduler
cat /sys/block/mmcblk1/queue/scheduler
```

```
[none] mq-deadline kyber bfq
```

这个切换是在线热切换，不影响正在跑的 I/O。重启后恢复为内核默认值。

> 简单结论：**ATK 板用默认 `mq-deadline` 就行**，不用折腾调度器。上面列这些只是为了让你知道有这个东西以及它存在的理由。

#### 写缓存策略（write_cache）

```bash
cat /sys/block/mmcblk1/queue/write_cache
```

```
write back
```

这个值来自 eMMC 初始化时内核读取的 `CACHE_CTRL`。但它**只是一个初始化快照，不会随运行时变化而更新**——你用 `mmc cache disable` 关了内部缓存后，`write_cache` 仍然显示 `write back`。

那它会有什么影响？如果 `write cache = write back`，内核在文件系统 journal 提交等关键时刻会发 CMD FLUSH 给 eMMC，确保数据落盘。但如果 `CACHE_CTRL` 实际已关闭（0x00），eMMC 收到 FLUSH 后检查一下就直接返回——**无害，只是多余**。

```
初始化时:
  内核读 CACHE_CTRL = 0x01
  → write_cache = write back
  → 运行时发 FLUSH

你手动改了之后:
  mmc cache disable → CACHE_CTRL = 0x00
  write_cache 不变，仍显示 write back
  内核继续发 FLUSH → eMMC 收到后直接返回，什么也不做
```

> **实际上这个属性对 ATK 板意义不大。** 想看缓存真实状态，直接看 `mmc extcsd read | grep CACHE_CTRL`。`write_cache` 只是告诉你"内核初始化时认为这个 eMMC 有缓存能力"，不代表当前运行时状态。

### 1.3.3 MMC 总线设备层

先看目录结构。`/sys/bus/mmc/devices/` 下是系统所有 MMC 总线上的设备，命名规则是 `mmc{host编号}:{RCA地址}`：

```bash
ls /sys/bus/mmc/devices/
```

```
mmc0:aaaa  mmc1:0001
```

ATK 板上有两个设备：
- `mmc0:aaaa` — SD 卡（host mmc0 = sdmmc1），RCA = 0xaaaa
- `mmc1:0001` — eMMC（host mmc1 = sdmmc2），RCA = 0x0001

> RCA（Relative Card Address）是 MMC/SD 协议初始化阶段主机分配给每张卡的 16 位地址，用于后续命令寻址。SD 卡的 RCA 由主机随机分配（这里是 `aaaa`），eMMC 通常固定为 `0001`。

进入 eMMC 设备目录可以看到内核暴露的所有属性：

```bash
ls /sys/bus/mmc/devices/mmc1:0001/
```

```
block/           csd              enhanced_rpmb_supported*  hwrev*        mmcblk1rpmb/   oemid*         pre_eol_info*      rca*             subsystem/
cid*             date*            erase_size*               life_time*    name*          power/         preferred_erase_size*  rel_sectors*    type*
cmdq_en*         driver@          ffu_capable*               manfid*       ocr*           prv*           raw_rpmb_size_mult*  removable*      uevent*
                 dsr*             fwrev*                     
```

按用途分组说明关键文件：

**设备标识（排查硬件型号、批次）：**

| 文件 | 实测值 | 说明 |
|------|--------|------|
| `manfid` | `0x0000d6` | 制造商 ID |
| `oemid` | `0x0103` | OEM ID，来自 CID 寄存器 |
| `name` | `58A43A` | 产品名，与启动日志 `mmc1:0001 58A43A` 一致 |
| `date` | `11/2023` | 生产日期 |
| `serial` | `0x3be31092` | 序列号 |
| `hwrev` | `0x0` | 硬件版本号 |
| `fwrev` | `0x0203030000000000` | 固件版本号 |
| `cid` | `d60103353841343341103be31092ba4e` | CID 寄存器完整内容 |

**CID 寄存器解析：**

原始的 `cid` 是 128 位（16 字节）的 hex dump，上面的 manfid、oemid、name、serial 都来自这里。拆解 `d60103353841343341103be31092ba4e`：

```
字节偏移  值             字段
[0]      d6           = 制造商 ID (0xd6)           → manfid
[1-2]    01 03        = OEM ID (0x0103)            → oemid
[3-8]    35 38 41 34 33 41 = 产品名 (ASCII: 58A43A) → name
[9]      10           = 产品版本 (1.0)              → hwrev = 0x0
[10-13] 3b e3 10 92   = 序列号                     → serial
[14-15] ba 4e         = CRC + 生产日期
```

生产日期编码在字节 [14-15] 中，需要按 MMC 协议标准解析，sysfs 的 `date` 文件已经帮你翻译好了，直接读那个就行。

> 日常工作中你基本不需要手动解析 CID 原始值——sysfs 已经把所有字段拆成单独文件了。`cid` 的原始值主要用于和生产厂商核对批次信息。

### 1.3.4 host 控制器与 CMDQ

`/sys/class/mmc_host/mmc1/` 是 host 控制器的类目录，主要看两个东西：

```bash
ls -la /sys/class/mmc_host/mmc1/
```

```
device -> ../../../48230000.mmc        ← 平台设备，MMIO 地址 0x48230000
mmc1:0001/                             ← symlink 到总线设备目录
power/ subsystem/ uevent               ← 电源管理和属性
```

`device` symlink 指向 sdmmc2 的平台设备，从这里可以去看控制器的资源信息：

```bash
ls /sys/class/mmc_host/mmc1/device/
```

`mmc1:0001/` 就是 `/sys/bus/mmc/devices/mmc1:0001/` 的另一个入口——两个路径指向同一个设备。

#### CMDQ 状态

eMMC 支持硬件命令队列（CMDQ）时，可以通过 sysfs 确认是否启用：

```bash
cat /sys/devices/platform/soc@0/42080000.bus/48230000.mmc/mmc_host/mmc1/mmc1:0001/cmdq_en
```

```
0
```

`0` 表示 CMDQ 当前**未启用**。这个路径很长，因为要从平台设备树逐级往下走。也可以从总线设备目录找到它：

```bash
find /sys -name cmdq_en 2>/dev/null
```

| 值 | 含义 |
|----|------|
| `0` | CMDQ 未启用（ATK 板当前状态） |
| `1` | CMDQ 已启用 |

ATK 板 CMDQ 未启用，和 extcsd 中 `CMDQ_MODE_EN = 0x00` 一致。启用 CMDQ 需要内核配置 `CONFIG_MMC_CQHCI`（STM32MP2 defconfig 默认已开启）以及 eMMC 硬件支持（`CMDQ_SUPPORT = 0x01`，已确认支持）。实际由内核在初始化时自动决定，不需要 DTS 属性。

> CMDQ 不是本节重点，它是 MMC 核心层和块设备层的交互机制，在 02-Architecture.md 中会结合 CQHCI 驱动详细分析。

---

## 1.4 debugfs 接口

debugfs 提供比 sysfs 更底层的 MMC 运行时状态，是排查时序问题的第一站。需要内核开启 `CONFIG_MMC_DEBUG=y`（ATK BSP defconfig 已开启）。

挂载后查看：

```bash
mount -t debugfs none /sys/kernel/debug
ls /sys/kernel/debug/mmc1/
```

```
caps       caps2      clock      err_state  err_stats  ios        mmc1:0001
```

### 1.4.1 ios — 当前时序参数

`ios` 是 debugfs 中最有价值的节点，直接读出当前控制器的实际配置：

```bash
cat /sys/kernel/debug/mmc1/ios
```

```
clock:          52000000 Hz
actual clock:   50000000 Hz
vdd:            21 (3.3 ~ 3.4 V)
bus mode:       2 (push-pull)
chip select:    0 (don't care)
power mode:     2 (on)
bus width:      3 (8 bits)
timing spec:    8 (mmc DDR52)
signal voltage: 1 (1.80 V)
driver type:    0 (driver type B)
```

与 DTS 配置的对照关系：

| ios 字段 | 对应 DTS | 实测值 | 说明 |
|---------|---------|--------|------|
| `clock` | `max-frequency` | 52 MHz | 请求值 |
| `actual clock` | 时钟树分频 | **50 MHz** | 实际输出，存在偏差 |
| `bus width` | `bus-width` | 8 bits | |
| `timing spec` | 模式属性（如 `mmc-ddr-1_8v`） | mmc DDR52 | |
| `signal voltage` | **`vqmmc-supply`** | **1.80 V** | **I/O 供电电压，决定 MMC 总线信号电平** |
| `vdd` | **`vmmc-supply`** | **3.3 ~ 3.4 V** | **核心供电电压，始终固定，与模式无关** |

> 最容易混淆的就是 `vdd` 和 `signal voltage`：**`vdd`（VCC）是 eMMC 核心/NAND 阵列的供电，永远 3.3V，不管你是 LEGACY 还是 HS400。`signal voltage`（VCCQ）是 MMC 总线信号电平，DDR52/HS200/HS400 必须用 1.8V。** DTS 中 `vmmc-supply` 管 VCC，`vqmmc-supply` 管 VCCQ，两者独立配置。

VCCQ 的电压决定了 eMMC 能跑多快，对比两种配置：

| | VCCQ = 1.8V（ATK 板） | VCCQ = 3.3V |
|--|---------------------|------------|
| `signal voltage` | 1.80 V | 3.3 V |
| 可选模式 | DDR52 ✅ / HS200 ✅ / HS400 ✅ | DDR52 ❌ / HS200 ❌ / HS400 ❌ |
| 最高速率 | 可达 400 MB/s（HS400） | 上限 52 MB/s（HS SDR） |
| 场景 | 需要高性能或 DDR52 以上模式 | 成本敏感、低性能即可 |

ATK 板选 1.8V 是正确的——即使 DTS 当前只开了 DDR52（104 MB/s），将来要切到 HS200/HS400 也不需要改硬件。如果 VCCQ 接了 3.3V，想升速就得改 PCB。

**注意 `actual clock` 和 `clock` 的差异**：DTS 中 `max-frequency = <166000000>`（166 MHz），但实际协商结果是 52 MHz（DDR52 模式），而且由于 STM32MP2 时钟树 PLL 分频不能精确产生 52 MHz，最终实际输出是 **50 MHz**。这属于正常现象，时钟树的分频器只能输出接近目标值的频率。

> 如果怀疑 eMMC 没有按预期模式运行（比如 DTS 配了 `mmc-hs200-1_8v` 但日志显示 DDR52），读 `ios` 的 `timing spec` 一眼就能确认实际跑在哪个模式。

### 1.4.2 err_stats — 错误统计与排查方法

`err_stats` 是 debugfs 中**排查硬件问题最重要的节点**。它记录了 eMMC 通信过程中发生的各类错误次数，是判断信号完整性、时序稳定性、硬件可靠性的直接依据。

```bash
cat /sys/kernel/debug/mmc1/err_stats
```

```
# ATK 板实测（所有计数器为 0，说明信号完整性良好）:
```

先看 err_stats 包含哪些错误类型：

| 错误字段 | 全称 | 计数原理 |
|---------|------|---------|
| `data_crc_err` | Data CRC Error（数据阶段） | 每次数据读/写传输完成后，CRC-16 校验失败就 +1 |
| `data_end_err` | Data CRC Error（Data End 阶段） | Data End 标志未到/超时 |
| `cmd_timeout_err` | Command Timeout | 命令发出后 64 个时钟周期内无响应 |
| `cmd_crc_err` | Command CRC Error | eMMC 返回的 CMD 响应 CRC7 校验失败 |
| `cmd_end_err` | Command End Bit Error | CMD 响应结束位格式错误 |
| `data_timeout_err` | Data Timeout | 数据传输超出硬件超时计数器 |
| `tuning_err` | Tuning Error | HS200/HS400 Tuning 过程失败 |
| `ack_timeout_err` | Boot ACK Timeout | 从 boot 分区启动时 ACK 超时 |
| `adma_err` | ADMA Error | ADMA 描述符错误或超出范围 |
| `erase_timeout_err` | Erase Timeout | 擦除操作超时 |

> ATK 板所有计数为 0，说明当前 DDR52 模式下信号完整性良好，没有通信错误。

---

#### 各错误的硬件原因与排查方法

##### data_crc_err — 数据 CRC 错误

**硬件原因：** 数据线上的比特在传输过程中被干扰，eMMC 或 host 计算出的 CRC-16 与发送方不匹配。**这是信号质量问题最常见的表现形式。**

```
正常时序:
  Host CLK █▁█▁█▁█▁█▁█▁█▁█▁
  Host DAT ———[数据位 + CRC16]———→ eMMC ✓

CRC 错误:
  Host CLK █▁█▁█▁█▁█▁█▁█▁█▁
  Host DAT ———[数据位 + CRC16]———→ eMMC ✗   ← 某位被噪声翻转
```

**排查路径（从上到下，概率从高到低）：**

| 步骤 | 检查什么 | 怎么查 | 修复 |
|------|---------|--------|------|
| 1 | clock 频率是否过高 | `cat /sys/kernel/debug/mmc1/ios` 看 `actual clock` | 降低 `max-frequency` 测试 |
| 2 | 数据线驱动强度 | DTS pinctrl `slew-rate` | 提高到 `<2>` 或 `<3>` |
| 3 | 信号走线长度 | PCB 布局 | DDR52 下 DAT 线偏差应 < 500mil |
| 4 | 参考电压噪声 | 示波器测 VCCQ 纹波 | 增大去耦电容 |
| 5 | eMMC 时钟相位 | DTS `st,neg-edge` | 尝试去掉/加上 `st,neg-edge` |

**实际案例：** 某产品从 DDR52 升级到 HS200 后，高温老化测试中大量 `data_crc_err`。排查步骤：
1. 读 err_stats 发现 `data_crc_err` 每分钟增长数百次，`cmd_crc_err` 为 0 → 判断是**数据线**问题，不是命令线
2. 降频到 100MHz 测试，CRC 错误消失 → 确定是高频信号完整性问题
3. 用示波器测量 DAT 线，发现 D7 线有过冲 > 200mV
4. 在 D7 线上加 22Ω 串联电阻抑制过冲，错误归零

---

##### data_end_err — Data End 错误

**硬件原因：** MMC 协议中，每次数据传输完成后，eMMC 会在 DAT0 线上发送一个 Data End 标志位。如果 host 在预期时间内没有收到这个标志，计数器 +1。

**与 data_crc_err 的区别：** Data CRC 表示数据内容错了，Data End 表示传输结束标志丢失。后者通常更严重——意味着数据根本没传完，eMMC 可能还在处理中。

**排查方法：**
- 如果 `data_end_err` 和 `data_crc_err` 同时增长 → 信号整体质量差，先按 data_crc_err 排查
- 如果只有 `data_end_err` 增长 → 怀疑 eMMC 内部处理超时，降低频率或增大超时 timeout

---

##### cmd_timeout_err — 命令超时

**硬件原因：** host 向 eMMC 发出命令后，eMMC 在 64 个时钟周期内没有拉低 DAT0 线（CMD 响应窗口开始）。

**什么情况下会出现（不一定代表故障）：**

| 场景 | 预期行为 | 是否需要处理 |
|------|---------|------------|
| 初始化阶段发送 CMD8（接口条件检查） | 不支持的卡不会响应，超时正常 | ❌ 正常流程 |
| 尝试 SD 协议时 ACMD41 超时 | 对 eMMC 发 ACMD41 必然超时 | ❌ 配 `no-sd` 后可避免 |
| 运行时 CMD25（多块写）超时 | **异常**，eMMC 内部忙但没有拉 DAT0 | ✅ 需要排查 |

**排查方法：**
- 大概率是正常初始化过程中的超时（如上面两种场景）。**先看 dmesg 中错误发生的时间点**：如果集中在 probe 阶段，忽略；如果发生在运行时，排查硬件。
- 查看 dmesg 是否有 `mmc1: Timeout waiting for hardware interrupt` 消息
- 检查电源稳定性，VCC 下降到 3.0V 以下时 eMMC 不响应命令

**实际案例：** 某板低温测试（-20°C）时偶发 `cmd_timeout_err`，每次卡在 eMMC 初始化阶段。原因是低温导致 PMIC 输出 VCC 电压下降至 2.9V（eMMC 要求 3.1~3.6V）。更换低温特性更好的 PMIC 后解决。

---

##### cmd_crc_err — 命令响应 CRC 错误

**硬件原因：** host 发出命令后，eMMC 返回了响应，但响应中的 CRC7 校验失败。与 `data_crc_err` 的区别在于：问题出在**命令线（CMD）**而不是数据线（DAT）。

**排查方法：**
- `cmd_crc_err` 增长 → 先检查 CMD 线的 PCB 走线
- 检查 CMD 线上的上拉电阻（eMMC 规范要求 CMD 线必须有上拉）
- 检查 `cmd_timeout_err` 是否同时增长：如果两者都增长，可能是 eMMC 进入了不稳定状态

**实际案例：** 某开发板用杜邦线飞接 eMMC 测试，CMD 线没有上拉电阻，`cmd_crc_err` 在启动阶段就累计到几十次。在 CMD 线与 VCCQ 之间焊接 10kΩ 上拉电阻后清零。

---

##### cmd_end_err — 命令结束位错误

**硬件原因：** MMC 协议规定 CMD 响应的最后一位必须是特定电平。如果 host 检测到结束位格式不正确，说明 CMD 线上的时序有问题。

**排查方法：**
- 与 `cmd_crc_err` 联动排查，如果两者同时出现，大概率是 CMD 线上的共性问题（走线过长、干扰、上拉不足）
- 用示波器抓 CMD 线波形，看响应结束位是否清晰、无抖动

---

##### data_timeout_err — 数据超时

**硬件原因：** eMMC 启动数据传输后，host 内部硬件超时计数器归零前数据没有传完。超时时间由 host 控制器设定（STM32MP2 的 SDMMC 控制器有可配的 DTIMER 寄存器）。

**与 cmd_timeout_err 的区别：** CMD 超时是**命令阶段**（CMD 线）。Data 超时是**数据阶段**（DAT 线），通常发生在 eMMC 内部忙（long write/erase）。

**排查方法：**
- 伴随大量写操作时出现 → 检查 eMMC 是否处于繁忙状态（没有及时拉低 DAT0）
- 检查 `CACHE_CTRL`：缓存开启时写操作返回快，理论上不容易超时。如果关闭了缓存，大块写入时可能因 eMMC 内部擦除慢而超时
- 检查 eMMC 是否接近寿命终点（`pre_eol_info` == 0x03）：濒死 eMMC 写操作极慢，容易超时

---

##### tuning_err — Tuning 错误

**硬件原因：** HS200/HS400 模式下，host 需要通过 tuning 过程找到最佳采样点。Tuning 时 host 发送固定数据模式，eMMC 返回已知内容，host 通过比较找到数据眼图中心。

```
正确 tuning（眼图中心采样）:
  CLK █▁█▁█▁█▁█▁█▁█▁█▁
  DAT ──╲___/‾‾‾╲___/‾‾‾─   ← 数据眼图
         ↑ 采样点在这里

Tuning 失败:
  CLK █▁█▁█▁█▁█▁█▁█▁█▁
  DAT ──╲___/‾‾‾╲___/‾‾‾─   ← 数据眼图
               ↑ 采样点在眼图边缘甚至外面
```

**什么时候会出现：**
- 频率过高，信号眼图太小没有足够的采样窗口
- VCCQ 电压偏低，信号幅值不够
- PCB 走线过长，信号衰减严重

**排查方法：**
- ATK 板 DDR52 模式不需要 tuning，所以 `tuning_err` 永远是 0
- 如果从 DDR52 升级到 HS200 后出现，示波器测眼图，调整 `st,neg-edge` 或硬件匹配

---

##### ack_timeout_err — Boot ACK 超时

**硬件原因：** 从 boot 分区启动时，eMMC 需要在收到 CMD0（参数 = BOOT_MODE）后发送 ACK。如果 ACK 没到就 +1。

**排查方法：**
- 这是 boot 流程相关，ATK 板从 SD 卡启动所以永远为 0
- 如果从 eMMC 启动时出现，检查 `BOOT_ACK` 配置和启动分区内容是否完整

---

##### adma_err — ADMA 错误

**硬件原因：** SDMMC 控制器使用 ADMA（Advanced DMA）在内存和 MMC 总线之间搬数据。ADMA 描述符表配置错误（地址不对齐、长度越界）时触发。

**排查方法：**
- 通常是内核驱动 bug，不是硬件问题
- 第一次出现就伴随内核 oops → 驱动或者内存分配问题
- 如果 ADMA 错误伴随内存压力出现 → DMA 缓冲区的物理地址不连续或超出了控制器寻址范围

---

##### erase_timeout_err — 擦除超时

**硬件原因：** eMMC 擦除大块数据时耗时较长（数百 ms），如果擦除操作超过 host 设定的超时时间就触发。

**排查方法：**
- 伴随 `blkdiscard`、`fstrim` 或文件系统丢弃操作出现 → 正常现象，可以增大超时值
- 单纯 mount 一个 ext4 文件系统就出现 → ext4 格式化时默认开启了 discard，挂载时加 `nodiscard` 选项可关闭

---

#### 排查流程图

```
发现 err_stats 有非零值
  │
  ├── 发生在初始化阶段（dmesg 时间戳 < 5s） → 大概率正常
  │      └─ 确认: dmesg | grep mmc1 | head -10
  │              看是否有 "mmc1: new ... card" 消息
  │              有 → 正常，忽略初始化阶段的超时
  │              无 → 排查 DTS 和电源
  │
  ├── data_crc_err 持续增长
  │      └─ 信号完整性排查优先级最高
  │           1. 降频测试
  │           2. 检查 pinctrl slew-rate
  │           3. 示波器测波形
  │
  ├── cmd_crc_err + cmd_end_err 同时增长
  │      └─ CMD 线问题
  │           1. 检查上拉电阻
  │           2. 检查 CMD 线走线长度
  │
  ├── cmd_timeout_err 运行时增长
  │      └─ 电源不稳定或 eMMC 进入异常状态
  │           1. 测 VCC/VCCQ 电压稳定性
  │           2. 尝试 mmc 重新初始化
  │
  └── erase_timeout_err 增长
         └─ 伴随 discard 操作 → 正常，忽略
           否则检查 eMMC 寿命 (pre_eol_info)
```

---

#### 一句话总结

| 发现 | 结论 |
|------|------|
| 所有计数器为 0 | 信号完整性良好，不需要任何处理（ATK 板当前状态） |
| `data_crc_err` 增长 | **信号完整性问题的第一信号**，优先排查 |
| `cmd_*_err` 增长 | CMD 线问题（上拉、走线、噪声） |
| `tuning_err` 增长 | HS200/HS400 需要重新 tuning 或改 DTS |
| `erase_timeout_err` 增长 | 大概率正常（discard 操作），先看 dmesg |

#### 配套节点：err_state — 快速健康检查

err_state 是 err_stats 的**汇总版**：内核遍历所有错误计数器，有任何一个非零就返回 1，否则返回 0（源码见 `drivers/mmc/core/debugfs.c` 的 `mmc_err_state_get`）。

```bash
cat /sys/kernel/debug/mmc1/err_state
```

```
0
```

ATK 板输出为 0，与 err_stats 全零一致。

| 值 | 含义 | 操作 |
|----|------|------|
| 0 | 自启动以来未发生过错误 | 无需操作 |
| 1 | 至少发生过一次错误 | 读 err_stats 看详情 |

err_state 与 err_stats 共享同一组计数器，关系如下：

```
err_stats 逐条明细:          err_state 汇总:
  data_crc_err = 0                   0  ← 所有计数器都是 0
  cmd_timeout_err = 0       ⟹
  tuning_err = 0                   (1  ← 假如有任何一个 > 0)
  ...
```

> 这个设计主要是为自动化监控服务的——监控脚本先读 err_state 这个整数，发现变成 1 了再读 err_stats 取详情，避免每次都全量解析多行文本。err_state 不会自动归零，和 err_stats 一样是累计值。它也不区分错误严重程度——一次 CMD8 超时（初始化阶段的正常流程）也会把它置为 1。

---

### 1.4.3 caps / caps2 — host 控制器能力

caps 和 caps2 是 SDMMC 控制器向 MMC 核心层声明的硬件能力位图，反映的是**控制器硬件的能力**，不是 eMMC 芯片的能力（后者看 EXT_CSD 的 CARD_TYPE）。

内核初始化时，驱动在 `stm32_sdmmc2_probe()` 中调用 `mmc_of_parse()` 解析 DTS 属性，将 `non-removable`、`no-sd`、`bus-width`、`cap-mmc-highspeed` 等翻译成 `mmc->caps` / `mmc->caps2` 位标志。

```bash
cat /sys/kernel/debug/mmc1/caps
```

```
0x40401347
```

```bash
cat /sys/kernel/debug/mmc1/caps2
```

```
0x00280000
```

把 hex 值拆成位图来看，caps 的每一位都对应一个 DTS 属性或驱动隐式设置。以下是完整解析（所有 32 位逐一列出，包括未置位的），**所有位偏移已经过内核源码 `include/linux/mmc/host.h` 验证**：

**caps = 0x40401347 位解析：**

```
0x40401347 = 0100 0000 0100 0000 0001 0011 0100 0111
              ↑ bit 31                      ↑ bit 0
```

| 位 | 掩码 | 宏名称 | 值 | 来源 | 含义 |
|----|------|--------|----|------|------|
| 0 | 0x00000001 | `MMC_CAP_4_BIT_DATA` | **1** ✅ | `bus-width` | 支持 4-bit 总线宽度 |
| 1 | 0x00000002 | `MMC_CAP_MMC_HIGHSPEED` | **1** ✅ | `cap-mmc-highspeed` | 支持 MMC HS 26/52MHz SDR |
| 2 | 0x00000004 | `MMC_CAP_SD_HIGHSPEED` | **1** ✅ | SoC 默认 | 支持 SD 高速模式 |
| 3 | 0x00000008 | `MMC_CAP_SDIO_IRQ` | 0 | — | SDIO 异步中断支持 |
| 4 | 0x00000010 | `MMC_CAP_SPI` | 0 | — | 仅 SPI 协议 |
| 5 | 0x00000020 | `MMC_CAP_NEEDS_POLL` | 0 | — | 需要轮询卡检测 |
| 6 | 0x00000040 | `MMC_CAP_8_BIT_DATA` | **1** ✅ | `bus-width = <8>` | **支持 8-bit 总线宽度** |
| 7 | 0x00000080 | `MMC_CAP_AGGRESSIVE_PM` | 0 | — | 空闲时挂起 eMMC 时钟 |
| 8 | 0x00000100 | `MMC_CAP_NONREMOVABLE` | **1** ✅ | `non-removable` | **焊死不可移除** |
| 9 | 0x00000200 | `MMC_CAP_WAIT_WHILE_BUSY` | **1** ✅ | 驱动隐式 | **等待 DAT0 busy 信号** |
| 10 | 0x00000400 | *(reserved)* | 0 | — | — |
| 11 | 0x00000800 | `MMC_CAP_3_3V_DDR` | 0 | — | eMMC DDR52 @3.3V |
| 12 | 0x00001000 | `MMC_CAP_1_8V_DDR` | **1** ✅ | `mmc-ddr-1_8v` | **eMMC DDR52 @1.8V** |
| 13 | 0x00002000 | `MMC_CAP_1_2V_DDR` | 0 | — | eMMC DDR52 @1.2V |
| 14 | 0x00004000 | `MMC_CAP_POWER_OFF_CARD` | 0 | — | 支持完全断电 |
| 15 | 0x00008000 | `MMC_CAP_BUS_WIDTH_TEST` | 0 | — | CMD14/CMD19 总线宽度测试 |
| 16 | 0x00010000 | `MMC_CAP_UHS_SDR12` | 0 | — | UHS-I SDR12（SD 卡用） |
| 17 | 0x00020000 | `MMC_CAP_UHS_SDR25` | 0 | — | UHS-I SDR25 |
| 18 | 0x00040000 | `MMC_CAP_UHS_SDR50` | 0 | — | UHS-I SDR50 |
| 19 | 0x00080000 | `MMC_CAP_UHS_SDR104` | 0 | — | UHS-I SDR104 |
| 20 | 0x00100000 | `MMC_CAP_UHS_DDR50` | 0 | — | UHS-I DDR50 |
| 21 | 0x00200000 | `MMC_CAP_SYNC_RUNTIME_PM` | 0 | — | 同步运行时 PM 挂起 |
| 22 | 0x00400000 | `MMC_CAP_NEED_RSP_BUSY` | **1** ✅ | 驱动隐式 | **R1B 响应需要检测 DAT0 busy** |
| 23 | 0x00800000 | `MMC_CAP_DRIVER_TYPE_A` | 0 | — | Driver Type A |
| 24 | 0x01000000 | `MMC_CAP_DRIVER_TYPE_C` | 0 | — | Driver Type C |
| 25 | 0x02000000 | `MMC_CAP_DRIVER_TYPE_D` | 0 | — | Driver Type D |
| 26 | 0x04000000 | *(reserved)* | 0 | — | — |
| 27 | 0x08000000 | `MMC_CAP_DONE_COMPLETE` | 0 | — | request_done 中完成回调 |
| 28 | 0x10000000 | `MMC_CAP_CD_WAKE` | 0 | — | 卡检测唤醒 |
| 29 | 0x20000000 | `MMC_CAP_CMD_DURING_TFR` | 0 | — | 数据传输中可发命令 |
| 30 | 0x40000000 | `MMC_CAP_CMD23` | **1** ✅ | 驱动隐式 | **支持 CMD23 多块写** |
| 31 | 0x80000000 | `MMC_CAP_HW_RESET` | 0 | — | 硬件复位 RST_n |

置位位共 9 个，按功能分组：

| 分组 | 置位的位 | 来自 DTS |
|------|---------|---------|
| 总线宽度 | 0 (4-bit)、6 (8-bit) | `bus-width = <8>` |
| 高速模式 | 1 (MMC HS)、12 (DDR 1.8V) | `cap-mmc-highspeed`、`mmc-ddr-1_8v` |
| 设备特性 | 8 (non-removable) | `non-removable` |
| 命令协议 | 9 (wait busy)、22 (need rsp busy)、30 (CMD23) | 驱动隐式设置 |
| SD 兼容 | 2 (SD HS) | SoC 默认（不影响 eMMC） |

**caps2 = 0x00280000 位解析：**

```
0x00280000 = 0000 0000 0010 1000 0000 0000 0000 0000
              ↑ bit 31                      ↑ bit 0
```

| 位 | 掩码 | 宏名称 | 值 | 来源 | 含义 |
|----|------|--------|----|------|------|
| 0 | 0x00000001 | `MMC_CAP2_BOOTPART_NOACC` | 0 | — | 启动分区不可访问 |
| 1 | 0x00000002 | *(reserved)* | 0 | — | — |
| 2 | 0x00000004 | `MMC_CAP2_FULL_PWR_CYCLE` | 0 | — | 支持完整上电循环 |
| 3 | 0x00000008 | `MMC_CAP2_FULL_PWR_CYCLE_IN_SUSPEND` | 0 | — | suspend 中上电循环 |
| 4 | 0x00000010 | *(reserved)* | 0 | — | — |
| 5 | 0x00000020 | `MMC_CAP2_HS200_1_8V_SDR` | 0 | — | HS200 @1.8V（DTS 未配） |
| 6 | 0x00000040 | `MMC_CAP2_HS200_1_2V_SDR` | 0 | — | HS200 @1.2V |
| 7 | 0x00000080 | `MMC_CAP2_SD_EXP` | 0 | — | SD Express |
| 8 | 0x00000100 | `MMC_CAP2_SD_EXP_1_2V` | 0 | — | SD Express 1.2V |
| 9 | 0x00000200 | *(reserved)* | 0 | — | — |
| 10 | 0x00000400 | `MMC_CAP2_CD_ACTIVE_HIGH` | 0 | — | 卡检测高电平有效 |
| 11 | 0x00000800 | `MMC_CAP2_RO_ACTIVE_HIGH` | 0 | — | 写保护高电平有效 |
| 12 | 0x00001000 | *(reserved)* | 0 | — | — |
| 13 | 0x00002000 | *(reserved)* | 0 | — | — |
| 14 | 0x00004000 | `MMC_CAP2_NO_PRESCAN_POWERUP` | 0 | — | 扫描前不上电 |
| 15 | 0x00008000 | `MMC_CAP2_HS400_1_8V` | 0 | — | HS400 @1.8V（DTS 未配） |
| 16 | 0x00010000 | `MMC_CAP2_HS400_1_2V` | 0 | — | HS400 @1.2V |
| 17 | 0x00020000 | `MMC_CAP2_SDIO_IRQ_NOTHREAD` | 0 | — | SDIO IRQ 非线程 |
| 18 | 0x00040000 | `MMC_CAP2_NO_WRITE_PROTECT` | 0 | — | 无写保护引脚 |
| 19 | **0x00080000** | **`MMC_CAP2_NO_SDIO`** | **1** ✅ | `no-sdio` | **跳过 SDIO 协议初始化** |
| 20 | 0x00100000 | `MMC_CAP2_HS400_ES` | 0 | — | HS400 Enhanced Strobe |
| 21 | **0x00200000** | **`MMC_CAP2_NO_SD`** | **1** ✅ | `no-sd` | **跳过 SD 协议初始化** |
| 22 | 0x00400000 | `MMC_CAP2_NO_MMC` | 0 | — | 跳过 MMC 协议 |
| 23 | 0x00800000 | `MMC_CAP2_CQE` | 0 | — | CMDQ 队列引擎（未启用） |
| 24 | 0x01000000 | `MMC_CAP2_CQE_DCMD` | 0 | — | CQE 直接命令 |
| 25 | 0x02000000 | `MMC_CAP2_AVOID_3_3V` | 0 | — | 避免 3.3V 信号 |
| 26 | 0x04000000 | `MMC_CAP2_MERGE_CAPABLE` | 0 | — | 段合并 |
| 27 | 0x08000000 | `MMC_CAP2_CRYPTO` | 0 | — | 内联加密（未配置） |
| 28 | 0x10000000 | `MMC_CAP2_ALT_GPT_TEGRA` | 0 | — | Tegra 专用 |
| 29~31 | — | *(unused)* | 0 | — | — |

> caps2 只置了 2 位（NO_SD + NO_SDIO），说明 DTS 中确实没有配任何 HS200/HS400 属性。

需要特别注意区分三层"能力"：

| 层 | 查询方式 | 反映什么 |
|----|---------|---------|
| caps / caps2（debugfs） | `cat /sys/kernel/debug/mmc1/caps` | host 控制器的能力（DTS 配了什么） |
| CARD_TYPE（EXT_CSD byte 196） | `mmc extcsd read /dev/mmcblk1 \| grep CARD_TYPE` | eMMC 芯片自身的能力 |
| ios（debugfs） | `cat /sys/kernel/debug/mmc1/ios` | 当前实际协商结果 |

> **三层能力可能不一致**：eMMC 芯片支持 HS400（`CARD_TYPE`），但 DTS 没配（caps2 无 HS400 位），所以 ios 显示 DDR52。排查问题时要区分：是 eMMC 不支持、还是 DTS 没配、还是当前没协商上。

---

### 1.4.4 clock — 运行时调频

`clock` 节点可以**读写**——读返回当前请求频率，写可以运行时改变频率，不需要改 DTS 重启。

先测默认值：

```bash
cat /sys/kernel/debug/mmc1/clock
```

```
52000000
```

单位 Hz，即 52 MHz。这是内核向时钟树请求的目标频率，与 ios 中的 `clock` 字段值一致。

实测降频（不需要重启）：

```bash
echo 26000000 > /sys/kernel/debug/mmc1/clock
cat /sys/kernel/debug/mmc1/clock
```

```
26000000
```

验证 ios 同步更新：

```bash
cat /sys/kernel/debug/mmc1/ios | grep clock
```

```
clock:          26000000 Hz
actual clock:   25000000 Hz
```

关键观察：**请求值和实际输出差约 2 MHz**（52→50、26→25），这是 STM32MP2 时钟树 PLL 分频器的精度限制——PLL 的分频系数是整数分频，不一定能精确合成目标频率。52 MHz 和 26 MHz 是请求值，硬件输出的是最接近的可达频率。

降频测试的主要用途：

| 场景 | 操作 | 目的 |
|------|------|------|
| 排查 data_crc_err | 降频到 26MHz | 确认错误是否与频率相关 |
| 排查初始化失败 | 降频到 400kHz | 排除高频时序问题 |
| 找稳定上限 | 从 52MHz 逐档降频 | 确定信号裕量 |
| 极端温度测试 | 降频后高/低温运行 | 验证温度对信号的影响 |

---

## 总结

01 篇从四个层面走通了 eMMC 子系统的"使用方法"：

| 层面 | 核心内容 | 对应章节 |
|------|---------|---------|
| **设备树** | DTS 属性 → 硬件配置 → 启动日志 | 1.1 |
| **用户空间** | `mmc` 工具、`dd` 性能测试、分区/格式化、RPMB | 1.2 |
| **sysfs** | 块设备统计、I/O 调度器、总线设备属性、健康状态 | 1.3 |
| **debugfs** | 时序参数、错误统计、能力位图、运行时调频 | 1.4 |

开发板实测结论：
- ATK 板 eMMC 运行在 DDR52 模式（104 MB/s 理论峰值），实测顺序读 ~83 MB/s、顺序写 ~66 MB/s
- 信号完整性良好（err_stats 全零），芯片几乎全新（寿命消耗 < 10%）
- 硬件支持 HS200/HS400，但 DTS 未配置

继续阅读：[02-Hardware.md](02-Hardware.md) — MMC 协议、命令集、PIO/SDMA/ADMA 传输模式、STM32MP2 SDMMC2 控制器寄存器。

