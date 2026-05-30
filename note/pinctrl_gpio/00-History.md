# GPIO 与 Pinctrl 子系统的历史演化

> 为什么 Linux 要把同一个硬件抽象成两个子系统？它们各自从哪来，到哪去。
>
> **字数**：约 14000 字 · **建议阅读时间**：45~70 分钟

## 1. 前传：混沌时代（Linux 早期 ~ 2.6.17）

Linux 最初是为 x86 PC 设计的，没有"GPIO 子系统"这个概念。随着 ARM 等嵌入式架构的加入，GPIO 操作的需求出现了，但存在两个核心问题：**完全没有标准 API  → 引脚复用也各自为政**。

### 1.1 没有标准 API 的时期

在 gpiolib（2.6.18）出现前，GPIO 操作是架构私有的。每个 CPU 架构在 `asm/arch-*/` 目录下定义自己的 GPIO 接口。有的直接暴露硬件寄存器，有的封装了一层函数，但**没有任何跨平台的标准**。

以 Linux v2.6.12（2005 年发布的早期 git 版本）为例，ARM 架构中的 GPIO 实现方式截然不同：

**SA-1100（StrongARM）—— 驱动直接读写寄存器**

SA-1100 是 Linux 早期支持的 ARM 平台之一。它的 GPIO 控制就是一组内存映射寄存器，定义在 `include/asm-arm/arch-sa1100/SA-1100.h` 中：

```c
/* 寄存器地址宏（0x9004xxxx 是物理地址，__REG 通过 io_p2v 转虚拟地址） */
#define GPLR    __REG(0x90040000)   /* GPIO Pin Level Register      */
#define GPDR    __REG(0x90040004)   /* GPIO Pin Direction Register  */
#define GPSR    __REG(0x90040008)   /* GPIO Pin output Set Register */
#define GPCR    __REG(0x9004000C)   /* GPIO Pin output Clear Reg.   */
#define GAFR    __REG(0x9004001C)   /* GPIO Alternate Function Reg. */

#define GPIO_GPIO(Nb)  (0x00000001 << (Nb))  /* 位掩码，Nb 取值 0~27 */
```

Board 文件（`arch/arm/mach-sa1100/collie.c`）里直接对这组寄存器赋值：

```c
/* collie.c — 直接操作 GPIO 寄存器 */
static void __init collie_init(void)
{
    GPLR = GPIO_GPIO18;             /* 设置 GPIO18 电平 */
    GAFR = GPIO_SSP_TXD |           /* 配置复用功能 */
           GPIO_SSP_SCLK |
           GPIO_SSP_SFRM |
           GPIO_32_768kHz;
    GPDR = GPIO_LDD8 | GPIO_LDD9 |  /* 配置方向（设为输出） */
           GPIO_SSP_TXD |
           GPIO_SDLC_SCLK |
           GPIO_32_768kHz;
}
```

没有 `gpio_request()`、没有 `gpio_direction_output()`——驱动代码直接向物理地址写值。一个驱动如果要支持多个平台，就得用条件编译包裹截然不同的寄存器操作。

**S3C2410（Samsung）—— 架构私有函数封装**

Samsung S3C2410 是同期广泛应用的 SoC，它采用了不同的私有方案：在 `include/asm-arm/arch-s3c2410/hardware.h` 中声明了一组架构特有的 GPIO 函数：

```c
/* S3C2410 架构私有 GPIO API */
extern void s3c2410_gpio_cfgpin(unsigned int pin, unsigned int function);
extern void s3c2410_gpio_setpin(unsigned int pin, unsigned int value);
extern unsigned int s3c2410_gpio_getpin(unsigned int pin);
extern int s3c2410_gpio_pullup(unsigned int pin, unsigned int to);
```

这是比 SA-1100 进步了一点——至少是函数调用而非直接寄存器赋值——但这些函数只对 S3C2410 有效。驱动要支持多平台，仍然只能写条件编译：

```c
/* 伪代码：驱动里充斥着这样的代码 */
#ifdef CONFIG_ARCH_SA1100
    GPDR |= GPIO_GPIO(5);
    GPSR = GPIO_GPIO(5);
#elif defined(CONFIG_ARCH_S3C2410)
    s3c2410_gpio_cfgpin(S3C2410_GPA5, S3C2410_GPA5_OUT);
    s3c2410_gpio_setpin(S3C2410_GPA5, 1);
#endif
```

### 1.2 ARM SoC pinmux 的混乱

各架构私有 API 只是表面问题。更底层也更严重的问题是**引脚复用（pin multiplexing）完全无标准化**。

每个 ARM SoC 厂商都在自己的 `arch/arm/mach-*` 目录里实现私有的 pinmux 逻辑：

```
arch/arm/mach-imx/
arch/arm/mach-omap2/
arch/arm/mach-pxa/
arch/arm/mach-s3c24xx/
arch/arm/mach-tegra/
...
```

以 PXA 为例，它的引脚配置在 board 文件中通过数组硬编码完成（`arch/arm/mach-pxa/mainstone.c`，v2.6.30）：

```c
static unsigned long mainstone_pin_config[] __initdata = {
    GPIO15_nCS_1,                /* Chip Select */
    GPIO58_LCD_LDD_0,            /* LCD 数据位 */
    GPIO59_LCD_LDD_1,
    /* ... 数十个引脚逐一配置 */
    GPIO32_MMC_CLK,              /* MMC */
    GPIO112_MMC_CMD,
    GPIO92_MMC_DAT_0,
    GPIO88_USBH1_PWR,            /* USB Host */
    GPIO89_USBH1_PEN,
    /* ... 每个外设都要在 board 里配一轮 */
};
```

后来 PXA 引入了 MFP 框架的 `MFP_CFG` 宏（`arch/arm/mach-pxa/include/mach/mfp.h`），但仍然只是把赋值变得更紧凑，无法改变"每个平台各自为政"的本质：

```c
#define MFP_CFG(pin, af) \
    ((MFP_CFG_DEFAULT & ~MFP_AF_MASK) | \
     (MFP_PIN(MFP_PIN_##pin) | MFP_##af))
```

不同厂商的 pinmux API 天差地别，没有任何共性：

| 厂商 | Pinmux API | 风格 |
|------|-----------|------|
| NXP i.MX | `iomux_config` + board 函数 | 数组 + 宏 |
| TI OMAP | `omap_cfg_reg()` + padconf 寄存器 | 直接写寄存器 |
| Samsung S3C | `s3c_gpio_cfgpin()` | 函数调用 |
| Intel PXA | `MFP_CFG(pin, af)` / `pxa2xx_mfp_config()` | 配置表 |

芯片厂商每出一个新 SoC 就要重写一套 pinmux 代码。到了 2000 年代末期，随着 ARM SoC 种类爆发（手机、平板、工控），这种各自为政的方式变得不可持续。

> **根本问题**：GPIO 和 pinmux 操作散落在 `arch/arm/` 和 `drivers/` 的各个角落，没有统一的抽象层。驱动开发者要么用 `#ifdef` 写不可移植的代码，要么依赖 board 文件传递的魔法整数。这种"混沌"状态直接催生了 gpiolib 和 pinctrl 两个子系统的诞生。

## 2. GPIO 标准化的第一步：gpiolib（Linux 2.6.18 ~ 2.6.27）

### 2.1 整型 API 的诞生

Linux 2.6.18（2006 年）引入了 gpiolib，**David Brownell** 贡献。

Brownell 是嵌入式 Linux 领域的先驱，主要活跃在 USB、GPIO 和电源管理方向。他在 2005～2006 年间注意到 LKML 上反复出现的讨论：每个新 ARM SoC 都要重写一套 GPIO 操作，驱动代码中充斥着 `#ifdef CONFIG_ARCH_XXX`。他给出的解决方案很直接——定义一个通用的 `gpio_chip` 接口，让厂商实现底层操作，上层 API 统一。

#### API 定义

gpiolib 定义了第一套跨平台的 GPIO 接口——**整型 API（Integer-based GPIO API）**，声明在 `include/linux/gpio.h`：

```c
int gpio_request(unsigned gpio, const char *label);
void gpio_free(unsigned gpio);
int gpio_direction_input(unsigned gpio);
int gpio_direction_output(unsigned gpio, int value);
int gpio_get_value(unsigned gpio);
void gpio_set_value(unsigned gpio, int value);
```

这套 API 的设计哲学：

- **所有操作通过整数编号**：GPIO 用 `unsigned gpio` 标识，不再依赖架构特定的寄存器地址或函数名
- **引入标签（label）机制**：`gpio_request()` 的 `label` 参数记录谁在使用该引脚，方便调试和冲突排查
- **区分方向和数据操作**：`direction_input/output` 单独成函数，与 `get/set` 分离——这对于需要配置方向寄存器（DDR）的硬件是必要的抽象

#### gpio_chip：厂商的注册接口

芯片厂商（或 SoC 的 arch 代码）通过注册 `gpio_chip` 来接入 gpiolib：

```c
struct gpio_chip {
    int             base;        /* GPIO 全局起始编号 */
    int             ngpio;       /* 本控制器支持的引脚数 */
    const char      *label;      /* 调试标签（如 "GPIOA"） */

    /* 核心操作集 */
    int (*request)(struct gpio_chip *chip, unsigned offset);
    void (*free)(struct gpio_chip *chip, unsigned offset);
    int (*direction_input)(struct gpio_chip *chip, unsigned offset);
    int (*direction_output)(struct gpio_chip *chip, unsigned offset, int value);
    int (*get)(struct gpio_chip *chip, unsigned offset);
    void (*set)(struct gpio_chip *chip, unsigned offset, int value);

    /* 可选操作 */
    int (*to_irq)(struct gpio_chip *chip, unsigned offset);  /* GPIO → IRQ 映射 */

    struct module *owner;        /* 防止模块被卸载 */
    void *data;                  /* 私有数据指针 */
};
```

厂商只需要填充这个结构体，调用 `gpiochip_add()` 注册。以 PXA 平台为例（`arch/arm/mach-pxa/gpio.c`，v2.6.30）：

```c
static struct gpio_chip pxa_gpio_chip = {
    .label            = "PXA_GPIO",
    .direction_input  = pxa_gpio_direction_input,
    .direction_output = pxa_gpio_direction_output,
    .get              = pxa_gpio_get,
    .set              = pxa_gpio_set,
    .request          = pxa_gpio_request,
    .to_irq           = pxa_gpio_to_irq,
};
```

对比之前 SA-1100 直接写寄存器的方式，这已经是巨大的进步——现在驱动写 `gpio_set_value(5, 1)` 统一调用，不再关心底层是 ARM、MIPS 还是 SuperH。

#### 核心实现：gpio_desc 数组和编号查找

gpiolib 内部维护了一个全局的 GPIO 描述符数组：

```c
/* drivers/gpio/gpiolib.c (v2.6.25 首次引入，简化结构) */
struct gpio_desc {
    struct gpio_chip *chip;      /* 所属控制器 */
    unsigned long    flags;      /* 状态位：是否已被申请、方向等 */
#ifdef CONFIG_DEBUG_FS
    const char       *label;     /* 请求者标签（仅 debugfs 启用时存在） */
#endif
};

static struct gpio_desc gpio_desc[ARCH_NR_GPIOS];  /* 默认 256 */
```

当驱动调用 `gpio_request(N, ...)` 时：

```
gpio_request(N, "led")
  └─ 检查 N < ARCH_NR_GPIOS（防止越界）
  └─ 检查 gpio_desc[N].flags 是否已被申请（冲突检测）
  └─ 设置 flags |= FLAG_REQUESTED
  └─ 记录标签到 gpio_desc[N].label
  └─ 调用 chip->request(chip, offset)（厂商的引脚级初始化）
```

`gpio_set_value(N, v)` 的查找路径更简短：

```
gpio_set_value(N, v)
  └─ gpio_desc[N].chip->set(chip, N - chip->base, v)
```

这里的关键是把一个全局整数 `N` 拆解为 `(chip, offset)` 二元组：`chip` 是控制器指针，`offset = N - chip->base` 是控制器内部的引脚偏移。这样驱动始终只跟一个整数打交道，所有"路由"都由 gpiolib 在内部完成。

#### gpio_chip 注册流程

当厂商调用 `gpiochip_add()` 时，内核做三件事：

```
gpiochip_add(&pxa_gpio_chip)
  └─ 1. 分配编号空间：从 base 开始，连续 ngpio 个编号
      └─ 如果 base < 0，自动分配空闲区间（动态编号）
  └─ 2. 遍历 gpio_desc[base..base+ngpio-1]
      └─ 设置每个 desc 的 chip 指针指向本 gpio_chip
  └─ 3. 将 chip 加入全局链表 gpio_chips（遍历查找用）
```

动态编号意味着驱动可以不用关心具体编号——它在 `gpiochip_add` 时传入 `base = -1`，系统自动分配，通过 `chip->base` 取出结果。这在多 GPIO 控制器的系统上很有用（避免手动协调编号）。

#### 一个典型的转换案例

以 PXA SPI 片选控制为例。gpiolib 出现前，board 文件里直接调用架构函数：

```c
/* 旧方式：架构私有调用 */
pxa_gpio_setpin(SPITZ_GPIO_LCDCON_CS, 1);
```

gpiolib 出现后，SPI 通用框架的片选操作改为调用 `gpio_set_value()`：

```c
/* 新方式：gpiolib 统一调用 */
gpio_set_value(spi->cs_gpio, 1);
```

`spi->cs_gpio` 由 board 文件或 DT 提供，不再依赖 PXA 特定的函数。同一套 SPI 驱动代码，在 PXA 上跑是 `gpio_set_value`，在 OMAP 上跑也是 `gpio_set_value`——底层路由由 gpiolib 完成。

#### 初始版本的限制（2.6.18）

最初的 gpiolib 有几个值得一提的局限：

1. **编译时可选**：`CONFIG_GPIOLIB` 是一个开关，未开启时 `gpio_request()` 等函数为空操作或返回错误。不使用 gpiolib 的平台（如 x86）不受影响。
2. **GPIO 编号范围固定**：`ARCH_NR_GPIOS` 默认为 256，超出此范围的 GPIO 无法使用。后续版本逐步增大这个限制（2.6.38 改为 512，4.x 改为 1024）。
3. **只支持整数 API**：描述符 API（`gpiod_*`）要到 3.10 才出现，这里还是 `gpio_*`。
4. **无设备树支持**：设备树在当时尚未在 ARM 上普及（ARM DT 支持始于 3.0 左右），GPIO 编号通过平台数据传递。

尽管如此，2.6.18 的 gpiolib 已经解决了最核心的问题——**驱动开发者终于可以写一次 GPIO 操作代码，然后在不同 SoC 之间复用**。

### 2.2 sysfs 用户态接口（2.6.27）

Linux 2.6.27（2008 年 7 月）在 gpiolib 中增加了 sysfs 用户态接口，代码**内联**在 `drivers/gpio/gpiolib.c` 中（当时未拆分为独立文件）。通过注册 `gpio_class`（`struct class`），在 `/sys/class/gpio/` 下创建文件层次。文档见 `Documentation/gpio.txt`：

```shell
echo N > /sys/class/gpio/export              # 导出 GPIO N 到用户态
echo out > /sys/class/gpio/gpioN/direction   # 设为输出
echo 1 > /sys/class/gpio/gpioN/value         # 输出高电平
```

**sysfs 目录结构**：

```
/sys/class/gpio/
├── export                   # 写 GPIO 编号 → 导出
├── unexport                 # 写 GPIO 编号 → 取消导出
├── gpioN/                   # 每个已导出的 GPIO
│   ├── value                # 电平值（r/w，input 只读）
│   ├── direction            # 方向（in/out/high/low）
│   └── edge                 # 中断触发沿（none/rising/falling/both）
└── gpiochipN/               # 每个 GPIO 控制器
    ├── base                 # 起始编号
    ├── label                # 控制器名
    └── ngpio                # 引脚数量
```

**内核导出 API**：驱动调用 `gpio_export(gpio, direction_may_change)` 将指定 GPIO 导出到 sysfs，`gpio_unexport()` 撤销导出。

**实现演变**：

| 版本 | 变更 |
|------|------|
| v2.6.27（2008） | sysfs 代码内联在 `drivers/gpio/gpiolib.c`，**David Brownell** 贡献 |
| v3.17（2014） | 拆分为独立的 `drivers/gpio/gpiolib-sysfs.c` |
| v4.6（2016） | 标记为 **obsolete**（`Documentation/ABI/obsolete/sysfs-gpio`），由 `CONFIG_GPIO_SYSFS` 控制 |
| 替代方案 | GPIO chardev（`/dev/gpiochipN`，见第 6.1 节） |

尽管 2016 年就已标记 obsolete，sysfs 接口至今（v6.6）未被删除，由 `CONFIG_GPIO_SYSFS` 控制编译。但新设计不应再依赖它，应使用 GPIO chardev（见第 6.1 节）。

### 2.3 整型 API 的局限

虽然 gpiolib 解决了"统一接口"的问题，但四个缺陷制约了它的可用性。

#### 1. 全局编号冲突

GPIO 控制器注册时需要指定 `base`（全局起始编号），多个芯片组共存时**必须人工协调编号空间**。以典型的嵌入式系统为例：

```c
/* 假设系统中有两个 GPIO 控制器 */
gpiochip_add(&gpio_chip_a);    /* base=0,  ngpio=32 → 占用 0~31  */
gpiochip_add(&gpio_chip_b);    /* base=32, ngpio=32 → 占用 32~63 */
gpiochip_add(&i2c_gpio_expander); /* ?? base 不能重叠 */
```

如果 `i2c_gpio_expander` 手动指定 base=16，不会报错，但会静默覆盖 `gpio_chip_a` 的编号空间——`gpio_request(16, ...)` 申请的是 expander 的引脚，驱动却以为是 `gpio_chip_a` 的引脚。这类 bug 只在运行时暴露，无编译期检查。

动态编号（base < 0）在 v2.6.27 中尚不可靠，`gpiochip_add()` 的自动分配逻辑简单粗暴，在热插拔场景下编号不固定，调试困难。

#### 2. 无 DT 感知

在设备树（ARM DT 支持始于 v3.0 左右）普及前，GPIO 编号通过 board 文件的 `platform_data` 硬编码传递。以 Sharp Zaurus sl-C3000（Spitz，PXA270）为例：

```c
/* arch/arm/mach-pxa/spitz.h — 魔法整数 */
#define SPITZ_GPIO_ON_KEY           (95)
#define SPITZ_GPIO_SWA              (97)
#define SPITZ_GPIO_CF_CD            (94)
#define SPITZ_GPIO_AC_IN            (115)
```

这些编号以 `platform_data` 结构体传入驱动：

```c
/* arch/arm/mach-pxa/spitz.c */
static struct corgissp_machinfo spitz_ssp_machinfo = {
    .cs_lcdcon    = SPITZ_GPIO_LCDCON_CS,       /* 53 */
    .cs_ads7846   = SPITZ_GPIO_ADS7846_CS,      /* 14 */
    .cs_max1111   = SPITZ_GPIO_MAX1111_CS,      /* 20 */
};

/* 驱动内使用：依赖 board 传过来的魔法数字 */
static int corgi_ssp_probe(struct platform_device *dev)
{
    struct corgissp_machinfo *machinfo = dev->dev.platform_data;
    gpio_request(machinfo->cs_lcdcon, "LCD CS");  /* 53 在 PXA 上是什么？ */
    gpio_set_value(machinfo->cs_lcdcon, 1);
}
```

这些编号只对 PXA 的 GPIO 空间有效——`95` 在 PXA 上是 ON_KEY，换一颗 SoC，95 号可能完全不存在，甚至属于另一个功能模块。驱动开发者必须时刻知道"我在哪颗芯片上跑"，与 gpiolib 的跨平台目标背道而驰。

#### 3. Consumer/Provider 未分离

整型 API 时代，**Consumer（使用 GPIO 的驱动）和 Provider（提供 GPIO 的芯片驱动）的接口声明在同一个头文件中**，没有分层：

```c
/* include/asm-generic/gpio.h (v2.6.27) — 全部混在一起 */

/* Provider 接口：芯片厂商需要调用的 */
extern int gpiochip_add(struct gpio_chip *chip);
extern int gpiochip_remove(struct gpio_chip *chip);

/* Consumer 接口：驱动开发者日常使用的 */
extern int gpio_request(unsigned gpio, const char *label);
extern void gpio_free(unsigned gpio);
extern int gpio_direction_input(unsigned gpio);
extern int gpio_direction_output(unsigned gpio, int value);

/* 用户态接口 */
extern int gpio_export(unsigned gpio, bool direction_may_change);
```

头文件分工的混乱反映出设计层面的问题：没有谁该包含哪个头文件的约定。一个 LED 驱动开发者本不需要看到 `struct gpio_chip` 和 `gpiochip_add()`，但它们就躺在同一个头文件里。这不是文档问题，而是**接口边界模糊**——任何人都可能在驱动里直接调用 `gpiochip_add()`，这在内核里曾真实发生过。

对比现代设计：

| 角色 | 现代头文件 | 使用者 |
|------|-----------|--------|
| Consumer | `#include <linux/gpio/consumer.h>` | 驱动开发者（如 LED 驱动） |
| Provider | `#include <linux/gpio/driver.h>` | 芯片厂商（如 GPIO 控制器驱动） |
| 遗留 | `#include <linux/gpio.h>` | 兼容旧代码（包含两者） |

#### 4. 引脚复用无管理

这是最致命的问题。整型 GPIO API 对**引脚复用（pinmux）完全没有感知**，驱动可以随意申请已经被其他外设占用的引脚，没有任何冲突检测。

假设一个 I2C 控制器正在使用 GPIOB 的 pin 6 和 7 作为 SCL/SDA，此时另一个驱动可以在不通知 I2C 子系统的情况下执行：

```c
gpio_request(38, "my-device");   /* 38 = GPIOB[6]，I2C 总线正在使用！ */
gpio_direction_output(38, 1);    /* 静默覆盖 MODER 寄存器 → I2C 总线崩了 */
```

pinmux 配置散落在各平台代码中，毫无标准化：

| 厂商 | Pinmux 配置方式 | 问题 |
|------|---------------|------|
| PXA | `MFP_CFG(pin, af)` 宏数组 | 纯板级，无内核校验 |
| i.MX | `iomux_config()` 调用 | 平台私有的配置函数 |
| OMAP | `omap_cfg_reg()` 寄存器写入 | 直接操作物理地址 |
| Samsung | `s3c_gpio_cfgpin()` 私有函数 | 只在本架构内有效 |

没有子系统来回答一个基本问题：**"这个引脚当前被配置成了什么功能？"**。结果是，GPIO 和 pinmux 这两套逻辑虽然操作的是同一组硬件寄存器，却没有任何协调机制来防止冲突。

---

这四个局限的本质可以总结为一句话：**gpiolib 只解决了"怎么读写 GPIO 数据"，但没解决"该引脚当前能不能被当 GPIO 用"**。后者需要 dt-bindings、pinctrl 子系统和描述符 API 的协同解决，这就是后续章节要讲的内容。

---

## 3. Pinctrl 子系统诞生（Linux 3.2 ~ 3.9，2012~2013）

### 3.1 pinctrl core 首次合入（v3.2）

Linus Walleij（ST-Ericsson / Linaro）于 2011~2012 年开发了 pinctrl 子系统，**Linux 3.2（2012 年 1 月）首次合入**，核心代码为 `drivers/pinctrl/core.c`：

```
Copyright (C) 2011 ST-Ericsson SA
Written on behalf of Linaro for ST-Ericsson
Author: Linus Walleij <linus.walleij@linaro.org>
```

设计目标：把 ARM SoC（以及其他架构）的引脚管理从散落的 `mach-*` 集中到一个统一框架中。

**v3.2 初始版本的文件结构**（共 7 个文件）：

```
drivers/pinctrl/
├── Kconfig / Makefile     — 编译框架
├── core.c  / core.h       — 核心层：pin controller 注册、pin desc 管理
├── pinmux.c / pinmux.h    — 引脚复用逻辑
└── pinmux-u300.c          — 首个硬件驱动（U300 平台）
    pinmux-sirf.c          — 第二个硬件驱动（SiRF 平台）
```

**注意**：v3.2 的初始版本只包含 **引脚枚举（pin enumeration）** 和 **引脚复用（pinmux）** 两个功能。电气配置（pinconf，如上下拉、驱动强度）尚未加入——这是 v3.3 才合入的。

#### pinctrl_desc：控制器的注册描述

```c
/* drivers/pinctrl/core.c — v3.2 首次亮相的结构 */
struct pinctrl_desc {
    const char *name;                        /* 控制器名 */
    struct pinctrl_pin_desc const *pins;     /* 该控制器管理的所有引脚描述 */
    unsigned int npins;                      /* 引脚数量 */
    unsigned int maxpin;                     /* 最大引脚号（允许稀疏编号空间） */
    struct pinctrl_ops *pctlops;             /* 引脚枚举操作 */
    struct pinmux_ops *pmxops;              /* 复用操作 */
    struct module *owner;                    /* 模块引用 */
};
```

注册方式：

```c
struct pinctrl_dev *pinctrl_register(struct pinctrl_desc *pctldesc,
                                     struct device *dev, void *driver_data);
```

#### 核心 API：驱动侧

```c
/* include/linux/pinctrl/pinctrl.h (v3.2) */
struct pinctrl_dev *pinctrl_register(struct pinctrl_desc *pctldesc,
                                     struct device *dev, void *driver_data);
void pinctrl_unregister(struct pinctrl_dev *pctldev);
void pinctrl_add_gpio_range(struct pinctrl_dev *pctldev,
                            struct pinctrl_gpio_range *range);
```

```c
/* include/linux/pinctrl/pinmux.h (v3.2) — Consumer 接口 */
struct pinmux *pinmux_get(struct device *dev, const char *name);
void pinmux_put(struct pinmux *pmx);
int pinmux_enable(struct pinmux *pmx);
void pinmux_disable(struct pinmux *pmx);
```

```c
/* include/linux/pinctrl/machine.h (v3.2) — Board 映射表 */
struct pinmux_map {
    const char *name;          /* 映射名 */
    const char *ctrl_dev_name; /* pin controller 设备名 */
    const char *function;      /* 复用功能（如 "spi0"） */
    const char *group;         /* 引脚组（如 "spi0_grp"） */
    const char *dev_name;      /* 使用该映射的设备名 */
    bool hog_on_boot;          /* 是否在启动时自动启用 */
};
```

#### pinmux_ops：厂商需实现的复用操作

```c
struct pinmux_ops {
    int (*request)(struct pinctrl_dev *pctldev, unsigned offset);
    int (*free)(struct pinctrl_dev *pctldev, unsigned offset);
    int (*list_functions)(struct pinctrl_dev *pctldev, unsigned selector);
    const char *(*get_function_name)(struct pinctrl_dev *pctldev,
                                     unsigned selector);
    int (*get_function_groups)(struct pinctrl_dev *pctldev,
                               unsigned selector,
                               const char *const **groups,
                               unsigned *const num_groups);
    int (*enable)(struct pinctrl_dev *pctldev,
                  unsigned func_selector, unsigned group_selector);
    void (*disable)(struct pinctrl_dev *pctldev,
                    unsigned func_selector, unsigned group_selector);
    int (*gpio_request_enable)(struct pinctrl_dev *pctldev,
                               struct pinctrl_gpio_range *range,
                               unsigned offset);
};
```

关键看最后一个回调 `gpio_request_enable`——它在初始版本中就已存在，说明 **GPIO 与 pinctrl 的交互从 pinctrl 诞生之初就在设计中**，而不是后来补的。`pinmux_request_gpio()` 就是 GPIO 核心层调用的入口，最终会落到这个回调上。

#### v3.2 时的 GPIO 与 pinctrl 交互流程

```
gpio_request(N, "led")
  └─ gpio_desc[N].chip->request(chip, offset)    ← gpio_chip.request
      └─ pinctrl_request_gpio(chip->base + offset)  ← 新加入的桥接
          └─ pinmux_request_gpio(gpio)
              └─ pin_request(pctldev, pin, ...)
                  └─ ops->gpio_request_enable(pctldev, range, offset)
```

这是当时加入的 pinctrl 对 GPIO 的"后门"钩子。注意此时（v3.2）还没有 `drivers/base/pinctrl.c`——驱动核心与 pinctrl 的自动绑定要等到 v3.9。

### 3.2 pinconf 加入（v3.3）

Linux 3.3（2012 年 3 月）新增了电气配置支持，`struct pinctrl_desc` 扩展为三个 ops：

```c
struct pinctrl_desc {
    ...
    struct pinctrl_ops *pctlops;    /* 引脚枚举 */
    struct pinmux_ops  *pmxops;     /* 引脚复用 */
    struct pinconf_ops *confops;    /* 引脚配置 ← v3.3 新增 */
};
```

新增 `include/linux/pinctrl/pinconf.h`，提供：

- **上下拉（pull-up/pull-down）**
- **驱动强度（drive strength）**
- **slew rate**
- **open drain / open source**

厂商实现的配置操作：

```c
struct pinconf_ops {
    int (*pin_config_get)(struct pinctrl_dev *pctldev, unsigned pin,
                          unsigned long *config);
    int (*pin_config_set)(struct pinctrl_dev *pctldev, unsigned pin,
                          unsigned long config);
    int (*pin_config_group_get)(struct pinctrl_dev *pctldev,
                                unsigned selector, unsigned long *config);
    int (*pin_config_group_set)(struct pinctrl_dev *pctldev,
                                unsigned selector, unsigned long config);
};
```

至此，pinctrl 的三大职责（枚举、复用、配置）全部到位。

### 3.3 驱动核心绑定（v3.9）

`drivers/base/pinctrl.c`（2013 年 4 月，v3.9-rc1）将 pinctrl 与驱动核心绑定：

```
Copyright (C) 2012 ST-Ericsson SA
Author: Linus Walleij <linus.walleij@linaro.org>
```

这个文件的核心作用：在设备 `probe` 时自动调用 `pinctrl_bind_pins()`，解析设备节点中的 `pinctrl-names` / `pinctrl-X` 属性并自动应用引脚状态。

```c
/* drivers/base/pinctrl.c —— v3.9 核心逻辑 */
int pinctrl_bind_pins(struct device *dev)
{
    /* 解析 dev 的 DT 节点中 pinctrl-names / pinctrl-X 属性 */
    /* 找到对应的 pinmux map，调用 pinmux_get / pinmux_enable */
    /* 设备 probe 完成 → 应用 "default" 状态 */
    /* 设备 suspend → 应用 "sleep" 状态 */
}
```

这使得驱动开发者不需要在 probe 里手动调用 pinmux 函数——只需在 DTS 中声明 `pinctrl-0 = <&uart1_pins_a>`，内核自动完成配置。

### 3.4 各厂商 pinmux 迁入时间线

| 厂商 | 原始位置 | pinctrl 驱动 | 迁入版本 |
|------|---------|-------------|---------|
| ST-Ericsson U300 |（首款实验平台） | `pinmux-u300.c` | v3.2 |
| SiRF | `arch/arm/mach-prima2/` | `pinmux-sirf.c` | v3.2 |
| NXP i.MX | `arch/arm/mach-imx/` | `pinctrl-imx.c` | v3.6~3.7 |
| Samsung Exynos | `arch/arm/mach-exynos/` | `pinctrl-exynos.c` | v3.10 |
| Generic (DT) | `arch/arm/mach-omap2/` | `pinctrl-single.c` | v3.11 |
| STM32 |（新平台） | `pinctrl-stm32.c` | v4.5 |

### 3.5 DT 绑定标准化

伴随 pinctrl 子系统，设备树绑定同步标准化，将引脚配置从 board 文件迁移到 DTS。

#### DTS 中的 pinctrl 属性

client 设备在 DTS 中通过 `pinctrl-names` / `pinctrl-X` 引用引脚状态：

```dts
&uart1 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&uart1_pins_a>;
    pinctrl-1 = <&uart1_sleep_pins_a>;
};
```

- `pinctrl-names`：状态名列表，对应 `pinctrl-0`、`pinctrl-1`……
- `pinctrl-0`：指向 pin controller 节点中定义的引脚配置 phandle
- 驱动核心在 probe/suspend/resume 时自动切换状态

pin controller 节点中定义具体的引脚复用和电气配置：

```dts
uart1_pins_a: uart1-0 {
    pins1 {
        pinmux = <STM32_PINMUX('A', 9, AF7)>;  /* TX */
        bias-disable;
        drive-push-pull;
        slew-rate = <0>;
    };
};
```

#### gpio-ranges：GPIO 与 Pinctrl 的编号映射

GPIO 与 pinctrl 通过 `gpio-ranges` 属性建立编号映射：

```dts
gpioa: gpio@44240000 {
    gpio-controller;
    #gpio-cells = <2>;
    gpio-ranges = <&pinctrl 0 0 16>;
    /*           pinctrl  芯片内部引脚号  数量
                      ↓       ↓           ↓
                GPIOA 的 0 号对应 pinctrl 的 0 号，共 16 个引脚 */
};
```

内核解析 `gpio-ranges` 后调用 `pinctrl_add_gpio_range()`，将 GPIO 编号空间注册到 pinctrl 子系统的查询表中，使得 `pinctrl_request_gpio()` 能根据 GPIO 编号反向查找对应的 pin controller 和 pin 号。

### 3.6 引脚复用的冲突检测机制

这是 pinctrl 相比旧方案最关键的改进——**每个引脚都有 `pin_desc` 来追踪当前使用者**，防止冲突。

#### 核心数据结构

```c
/* drivers/pinctrl/core.h */
struct pin_desc {
    struct pinctrl_dev *pctldev;    /* 所属 pin controller */
    const char        *name;        /* 引脚名（如 "PA9"） */
    spinlock_t         lock;        /* 保护并发访问 */
#ifdef CONFIG_PINMUX
    unsigned int       mux_usecount; /* 引用计数（可多个 mapping 共享） */
    const char        *mux_owner;   /* 当前使用者的名称 */
#endif
#ifdef CONFIG_GPIOLIB
    const char        *gpio_owner;  /* GPIO 所有者（与 mux 不同） */
#endif
};
```

每个物理引脚对应一个 `pin_desc`，记录谁在用、用作什么功能。

#### 检测流程

当驱动申请一个引脚时（无论是通过 pinmux 还是 GPIO 请求），最终进入 `pin_request()` 函数，以下游 STM32MP257 为例展开流程：

```
pinctrl_get(dev_name)                   ← 驱动侧：请求引脚控制权
  → pinctrl_dt_to_map()                 ← 解析设备树的 pinctrl-X 属性
  → create_pinctrl()
    → add_map_mux()
      → pinmux_map_to_setting()
        → pin_request(pctldev, pin, func, gpio_range)
          └─ spin_lock(&desc->lock)
          └─ 检查 desc->mux_usecount 是否为 0
              ├── 是 → 可以申请，记录 mux_owner
              └── 否 → 检查 mux_owner 是否等于当前请求者
                  ├── 相同 → usecount++，复用（同一驱动多次请求同一引脚）
                  └── 不同 → "pin already requested" — 拒绝
          └─ spin_unlock(&desc->lock)
```

#### 三种冲突场景

**场景 1：两个驱动争用同一引脚**

```
驱动 A 请求引脚 PA9 作为 UART1_TX  → 成功，mux_owner="uart1"
驱动 B 请求引脚 PA9 作为 SPI2_SCK  → 失败！
  内核日志: "pin PA9 already requested by uart1; cannot claim for spi2"
```

**场景 2：GPIO 与复用功能的冲突**

```
驱动 A 使用 PA9 作为 I2C1_SCL（复用功能）
驱动 B 调用 gpio_request(9, "mydev")  → 通过 pinctrl_gpio_request() 检查
  → pin_request() 发现 mux_usecount > 0
  → 检查 ops->strict 标志
     ├── strict=1 → 拒绝（-EBUSY）
     └── strict=0 → 软 claim 放行（非真正"并存"）
```

`strict` 标志由 pin controller 驱动设置，理解它的含义需要澄清一个重要概念。

**strict=0 不是允许一个引脚同时跑两个功能，而是只做 soft claim（记录所有权），不做 hard block（硬件强制隔离）。** 驱动 B 虽然能拿到 gpio_desc，但实际能做什么受硬件限制。

以 STM32MP257 为例，PA9 已配为 I2C1_SCL（MODER=10, AFR=4），此时驱动 B 的操作结果：

| 操作 | 能否生效 | 原因 |
|------|---------|------|
| `gpiod_get_value()` | 能 | 读 IDR 寄存器无需 MODER 配合——它始终反映物理电平 |
| `gpiod_set_value()` | 可写但被"吞掉" | 写 BSRR/ODR 确实会改寄存器位，但 MODER 未切到 output，**寄存器值不会驱动到引脚上** |
| `gpiod_direction_output()` | 最终受 pinctrl 拦截 | 在方向切换时仍会通过 `pinmux_gpio_direction()` 校验，不会偷偷改 MODER |

所以 strict=0 的实际含义是：**pinctrl 让请求通过，但硬件自动限制了你能做什么**。实践中这用于 debug/monitor 场景（如另一个驱动偷看 I2C SCL 波形），而非真正同时使用引脚。

STM32 通常不设 strict，原因是其 GPIO 读寄存器（IDR）在 Alternate Function 模式下始终有效——这是硬件特性决定的，与严格性配置无关。

**场景 3：同一驱动多次请求同一组引脚**

```
pinctrl_get("mmc0")  → 请求 MMC 的所有引脚
pinctrl_get("mmc0")  → 再次请求同一组引脚
  → mux_usecount 从 1 → 2，不报错
pinctrl_put("mmc0")  → usecount--，仍为 1
pinctrl_put("mmc0")  → usecount=0，释放引脚
```

#### v3.2 初版 vs 现代版本（v6.6）的差异

| 特性 | v3.2 初版 | v6.6 |
|------|----------|------|
| 冲突检测 | `if (desc->mux_function)` 二进制 | `mux_usecount` 引用计数 + 所有者对比 |
| 锁 | `spinlock` | `mutex`（scoped_guard 语法） |
| GPIO 所有者 | 无单独字段 | `gpio_owner` 独立追踪 |
| strict 模式 | 不支持 | 支持，由 `ops->strict` 控制 |
| 错误信息 | `"pin already requested"` | `"pin %s already requested by %s"` |

### 3.7 影响

pinctrl 子系统上线后，各厂商的 pinmux 代码逐渐从 `mach-*` 迁出，归入 `drivers/pinctrl/`。**GPIO 和 pinmux 之间有了协调机制**——再也不会出现一个驱动把 I2C 总线引脚偷偷改成 GPIO 输出的情况。

## 4. GPIO 描述符革命（Linux 3.13 ~ 4.x，2013~2015）

### 4.1 核心矛盾：整数编号的不可移植性

整型 GPIO API 的问题在 2.3 节已详细说明，其根源可以归结为一句话：**全局整数编号无法携带设备拓扑信息**。

驱动中写 `gpio_request(47, "led")`，编号 47 的含义完全取决于 board 文件或平台代码中 `gpiochip_add()` 的注册顺序。换一颗 SoC、换一个板子、甚至只是调整了 Kconfig，47 号就可能指向不同的引脚。这在设备树普及前是常态——驱动开发者必须知道自己在哪块板上跑。

设备树在 ARM 上的普及（v3.0+）为解决这个问题创造了条件：DT 提供了从 **phandle + 偏移** 描述 GPIO 的能力，但内核 API 还是接收整数。这中间缺少一层转换——**把 DT 中的"哪个控制器第几脚"自动翻译成 gpio_desc，而不经过全局编号**。

Linus Walleij 在 David Brownell 退休后于 2011 年左右接管 GPIO 子系统维护，主导了这场转换。

### 4.2 描述符 API 的引入历程

描述符 API 不是一次合入完成的，而是跨多个内核版本逐步完善的：

| 版本 | 变更 | 说明 |
|------|------|------|
| v3.5（2012） | `of_get_named_gpio_flags()` 加入 | DT → GPIO 编号的解析函数，但**返回仍是整数** |
| v3.10（2013） | `gpiod_*` 内部基础设施 | 函数已创建但 **static 未导出** |
| v3.13（2014.1） | `include/linux/gpio/consumer.h` 创建 | 描述符 API 首次成为**公开接口** |
| v3.14（2014.3） | `include/linux/gpio/driver.h` 拆分 | Consumer 与 Provider 头文件正式分离 |
| v3.13+ | `devm_gpiod_get()` 等托管版本 | 设备生命周期管理的自动释放版本 |

**关键的转折点是 v3.13**：新增 `include/linux/gpio/` 目录，GPIO 头文件从此有了分层结构：

```
include/linux/gpio/           ← v3.13 新增目录
├── consumer.h                ← Consumer（驱动开发者用）
└── driver.h                  ← Provider（芯片厂商用，v3.14 加入）
```

### 4.3 描述符 API 的外在形式

```c
/* include/linux/gpio/consumer.h (v3.13 首次公开) */
#include <linux/gpio/consumer.h>

/* 获取和释放 GPIO 描述符 */
struct gpio_desc *gpiod_get(struct device *dev, const char *con_id);
struct gpio_desc *gpiod_get_index(struct device *dev,
                                  const char *con_id, unsigned int idx);
void gpiod_put(struct gpio_desc *desc);

/* 方向和数据操作 */
int gpiod_direction_input(struct gpio_desc *desc);
int gpiod_direction_output(struct gpio_desc *desc, int value);
int gpiod_get_value(const struct gpio_desc *desc);
void gpiod_set_value(struct gpio_desc *desc, int value);
```

**关键变化**：`gpiod_get()` 的第一个参数是 `struct device *dev`，不传整数编号。核心层通过 `dev` 的设备树节点自动查找对应的 GPIO 属性：

```dts
myled {
    compatible = "mycompany,led";
    led-gpios = <&gpioa 5 GPIO_ACTIVE_LOW>;
    /*            ↑控制器  ↑偏移 ↑标志      */
    /* 驱动说 "我要 led-gpios"，内核查 gpioa→5  */
};
```

```c
/* 驱动代码：完全不涉及 GPIO 编号 */
struct gpio_desc *led = gpiod_get(dev, "led");
if (IS_ERR(led))
    return PTR_ERR(led);

gpiod_direction_output(led, 0);
/* 释放由 gpiod_put() 或 devm 自动管理 */
```

**命名规则**：`gpiod_get(dev, "led")` 查找的是 DT 属性名 **`led-gpios`**（去掉 `-gpios` 后缀，对应 `con_id` = "led"）。这是约定：DT 属性以 `-gpios` 结尾，驱动中传去掉后缀的名字。

### 4.4 Consumer/Provider 分层：头文件革命

描述符 API 最深远的设计不是"用指针替代整数"，而是 **Consumer/Provider 接口的物理分离**。

#### 旧世界：一个头文件管所有

v3.13 之前，`include/linux/gpio.h`（或架构对应的头文件）同时包含以下内容，不分角色：

```c
/* include/asm-generic/gpio.h (v3.10) — 角色不分 */
struct gpio_chip { ... };              /* 芯片厂商需要知道的结构 */
extern int gpiochip_add(struct gpio_chip *chip);  /* 厂商的注册函数 */
extern int gpio_request(unsigned gpio, const char *label);  /* 驱动开发者的 API */
extern void gpio_set_value(unsigned gpio, int value);       /* 驱动开发者的 API */
extern int gpio_export(unsigned gpio, bool direction_may_change);  /* sysfs */
```

一个 LED 驱动本不需要看到 `struct gpio_chip`，但它被迫包含了。接口边界模糊，职责不清。

#### 新世界：三个层次，各司其职

```
应用层驱动（LED、按键、MMC、传感器...）
         │
         ▼
Consumer 接口                     include/linux/gpio/consumer.h
  gpiod_get() / gpiod_set_value() / gpiod_direction_output()
         │
         ▼
GPIO Core                         drivers/gpio/gpiolib.c
  查找 desc → 找到对应的 gpio_chip → 调用 ops
  管理 gpio_desc 数组、冲突检测、标签记录
         │
         ▼
Provider 接口                     include/linux/gpio/driver.h
  struct gpio_chip { ... }        ← 厂商只需要包含这个头文件
  gpiochip_add_data()
         │
         ▼
硬件层（芯片厂商实现）             drivers/gpio/gpio-*.c
  stm32_gpio.c / pxa_gpio.c / imx_gpio.c ...
```

每个角色只包含自己需要的头文件，不越界。

### 4.5 gpiod_get 的内部查找路径

`gpiod_get()` 的核心工作：从设备树中解析出 GPIO 控制器和偏移量，返回描述符。

```
gpiod_get(dev, "led")
  └─ gpiod_get_index(dev, "led", 0, flags)
      └─ gpiod_find_and_request(dev, fwnode, "led", 0, &flags)
          ├─ 1. 从 dev->fwnode 获取设备树节点
          ├─ 2. 在节点中查找属性名为 "led-gpios" 的属性
          ├─ 3. 解析 phandle → 找到对应的 gpio_chip（如 &gpioa）
          ├─ 4. 解析偏移量 → 确定引脚号（如偏移 5）
          ├─ 5. 解析 GPIO 标志（如 GPIO_ACTIVE_LOW）
          ├─ 6. 调用 gpiod_request(desc, con_id) → 标记为已申请
          └─ 7. 返回 struct gpio_desc *（不透明指针）
```

#### 关键细节：从 phandle 到 gpio_chip 的映射

GPIO 控制器在设备树中有两个关键属性：

```dts
gpioa: gpio@44240000 {
    gpio-controller;          /* 声明：我是一个 GPIO 控制器 */
    #gpio-cells = <2>;        /* 每个引用需要 2 个 cell：<偏移 标志> */
};

myled {
    led-gpios = <&gpioa 5 GPIO_ACTIVE_LOW>;
    /*             ↑     ↑       ↑
             phandle   偏移    标志
    */
};
```

内核解析流程：

```
of_parse_phandle(np, "led-gpios", 0)    → 获取 phandle 指向的节点（gpioa）
of_find_gpiochip_by_node(gpioa_node)     → 找到对应的 gpio_chip 指针
of_gpio_flags_parse(gpioa_node, ...)      → 解析偏移和标志
gpiochip_request_own_desc(chip, offset)  → 从芯片中分配一个描述符
```

完成时，系统中没有出现任何整数 GPIO 编号——全程通过 phandle 和偏移操作。

### 4.6 devm_ 托管版本（Linux 3.13+）

Linux 3.13 同时引入了 `devm_` 系列的托管 GPIO API：

```c
struct gpio_desc *devm_gpiod_get(struct device *dev, const char *con_id);
/* 设备注销时自动释放，无需显式调用 gpiod_put */
```

对比非托管版本，错误处理大幅简化：

```c
/* 非托管版本：每个错误路径都要手动 gpiod_put，容易遗漏 */
int probe(struct device *dev)
{
    struct gpio_desc *desc = gpiod_get(dev, "led");
    if (IS_ERR(desc))
        return PTR_ERR(desc);

    int ret = some_other_init(dev);
    if (ret) {
        gpiod_put(desc);    /* 容易遗漏！ */
        return ret;
    }
    return 0;
}
```

```c
/* 托管版本：devm_gpiod_get 自动处理释放 */
int probe(struct device *dev)
{
    struct gpio_desc *desc = devm_gpiod_get(dev, "led");
    if (IS_ERR(desc))
        return PTR_ERR(desc);
    /* 无需手动释放，dev->driver 注销时自动 gpiod_put */
    return 0;
}
```

`devm_` 的实现原理：在 `devres` 链表中注册回调，设备注销时内核遍历链表自动调用 `gpiod_put`。

### 4.7 描述符 API 解决了什么

| 旧问题 | 描述符 API 的解法 |
|--------|------------------|
| 全局编号冲突 | **不暴露编号**——驱动通过 `dev` + `con_id` 获取 gpio_desc，全程不碰整数 |
| 无 DT 感知 | **自动 phandle 解析**——`gpiod_get()` 内部完成 DT 节点到 gpio_chip 的映射 |
| Consumer/Provider 未分离 | **头文件分层**——consumer.h 和 driver.h 从物理上隔离了两种角色 |
| 运行时类型安全 | **gpio_desc 是不透明指针**——不能被当作整数运算或误用 |

但描述符 API **本身**没有内置冲突检测逻辑——`gpiod_get()` 走到最后一步时会调用 `gpiochip_request_own_desc()` → `chip->request()`，通过第 3.6 节的 `pin_request()` 机制落到 pinctrl 的冲突检测上。换句话说，描述符 API **复用了 pinctrl 的后端校验**，自己不另起炉灶。

所以这个问题的正确答案是：
- **谁解决了冲突检测**：§3 的 pinctrl 子系统（`pin_desc.mux_usecount` + `gpio_owner` + `strict`）
- **描述符 API 做了什么**：在调用链末端接入 pinctrl 的检测入口，不重复实现
- **两者的桥梁**：`gpio-ranges` 映射 + pinctrl 后门机制（`gpio_request_enable` 等），见第 5 节

---

## 5. GPIO 与 Pinctrl 的融合

### 5.1 三个需要解决的问题

前两章分别讲述了 GPIO 和 Pinctrl 两个子系统的演化。它们在代码上是**独立的**——不同维护者、不同目录、不同 API。但在 STM32 这样的 SoC 上，它们面对的是**同一组硬件寄存器**：GPIO 的 MODER 既控制数据方向（GPIO 视角），也控制功能模式选择（Pinctrl 视角）。

这就产生了三个实际问题，按"从基础到上层"排列：

---

**问题 1（编号翻译）：GPIO core 说"pin 9"和 Pinctrl 说"pin 9"是同一个东西吗？**

两个子系统各自维护一套编号，**互不认识**：

- **GPIO core 内部**维护一个全局数组 `gpio_desc[ARCH_NR_GPIOS]`（v6.6 默认 1024），数组下标就是**全局编号**。驱动看不到它——驱动只跟 `struct gpio_desc *` 打交道，全程不碰数字。
- **Pinctrl 内部**每个控制器维护自己的 `pin_desc[0..npins-1]`，下标是**控制器内 pin 号**。

这里有一个容易被忽略的不对称：**GPIO bank 有多个（GPIOA、GPIOB…各是一个 gpio_chip），但 pinctrl 通常只有一个控制器**，它把所有 bank 的引脚拍平到一个 pin_desc 数组里。所以 GPIOA 的 pin 0~15 对应 pinctrl 的 pin 0~15，GPIOB 的 pin 0~15 对应 pinctrl 的 pin 16~31，以此类推。

当 GPIO core 调用 pinctrl 做冲突检测时，它传的是自己的全局编号，但 pinctrl 不认识这个数字——需要一个翻译表把它转成 `(pinctrl_dev, pin_offset)`。

```
GPIO core 内部：gpio_desc[9]                     ← 全局编号 9
         ↓
Pinctrl 需要知道：查这个 9 号对应哪个控制器、哪个 pin
         ↓
需要查表：全局 9 号 → 属于 GPIOA → pinctrl 的 pin 9

        ↑——如果 gpio-ranges 是 1:1 映射，两边编号碰巧相等
            但这不是必然的，见 §5.2
```

> **没有这个映射 → GPIO core 不知道全局编号对应哪个 pinctrl 引脚 → 冲突检测调用不出去 → 等于没检测。**

---

**问题 2（运行时代理）：驱动调 gpiod_get() 时，谁来把引脚切到 GPIO 功能？**

一个典型的 GPIO 使用流程：

```c
gpiod_get(dev, "led");             // 拿到描述符
gpiod_direction_output(led, 0);    // 设为输出低电平
```

驱动没有调用任何 pinctrl API，但硬件上需要把对应引脚的 MODER 从"复位值"切到"output"模式。**谁来干这个活？**

方案 A：DTS 里每个 GPIO 使用者都写 `pinctrl-0`。可行但烦人——每个 LED、每个按键都要配。

方案 B：GPIO 核心层在操作硬件时"顺手"调用 pinctrl，自动把引脚切换为 GPIO 功能。这就是内核实际采用的方案，叫做**后门机制（backdoor）**。

> **没有这个代理 → 驱动以为配好了 GPIO，实际上 MODER 还是复位值，引脚不工作。**

---

**问题 3（驱动合体）：STM32 上 GPIO 和 Pinctrl 是同一个硬件模块，驱动怎么写？**

在 STM32 上，没有独立的 Pinctrl 硬件模块——"Pinctrl"这个概念是软件对 GPIO 寄存器中复用/配置功能的抽象。MODER、AFR、PUPDR、OSPEEDR 这些寄存器全在 GPIO 的地址空间里。

如果 GPIO 和 Pinctrl 各写一个驱动，它们就得**争同一组寄存器**——谁先 probe、谁后 probe、谁读 MODER 时另一个正在写，都是问题。

> **没有统一驱动 → 两个驱动抢同一组寄存器 → 时序问题 + 竞态。**

---

这三个问题分别对应三个解决方案：**gpio-ranges 映射**（问题 1）、**后门调用链**（问题 2）、**合体 probe**（问题 3）。下面逐一展开。

### 5.2 gpio-ranges：编号映射表（解决问题 1）

#### 为什么需要它

前文说过，GPIO core 内部用全局编号（`gpio_desc[9]`），Pinctrl 用控制器内 pin 号（`pin_desc[9]`）。两者互不认识——GPIO core 在调用 `pinctrl_gpio_request()` 时，需要把"全局编号 9"翻译成"pinctrl 的 pin 9"。这个翻译表就是 `gpio-ranges`。

#### 在 DTS 中怎么声明

```dts
gpioa: gpio@44240000 {
    gpio-controller;
    #gpio-cells = <2>;
    gpio-ranges = <&pinctrl 0 0 16>;
    /*           ↑        ↑ ↑  ↑
                 引脚控制器  │  │
                         GPIOA 的 0 号  │
                          对应 pinctrl 的 0 号
                                   一共 16 个引脚 */
};
```

含义：GPIOA bank 的 `offset 0` → Pinctrl 的 `pin 0`，GPIOA 的 `offset 1` → Pinctrl 的 `pin 1`，……共 16 个。这是一对一连续映射，也是最常见的配置。

再看完整的多 bank 场景——每个 GPIO bank 各有自己的 DT 节点和 `gpio-ranges`，但都指向同一个 `&pinctrl`：

```dts
pinctrl: pinctrl@44240000 {
    compatible = "st,stm32mp257-pinctrl";
    /* pinctrl 只有一个，管理所有 bank 的引脚 */

    gpioa: gpio@44240000 {
        gpio-controller;
        #gpio-cells = <2>;
        gpio-ranges = <&pinctrl 0 0 16>;   /* GPIOA 的 pin 0→pinctrl pin 0 */
    };
    gpiob: gpio@44250000 {
        gpio-controller;
        #gpio-cells = <2>;
        gpio-ranges = <&pinctrl 0 16 16>;  /* GPIOB 的 pin 0→pinctrl pin 16 */
    };
    gpioc: gpio@44260000 {
        gpio-controller;
        #gpio-cells = <2>;
        gpio-ranges = <&pinctrl 0 32 16>;  /* GPIOC 的 pin 0→pinctrl pin 32 */
    };
    /* ... 所有 bank 都指向 &pinctrl，偏移依次递增 */
};
```

pinctrl 内部有一个拍平的 `pin_desc[0..npins-1]` 数组：

```
pin_desc[0]  = PA0    pin_desc[16] = PB0    pin_desc[32] = PC0
pin_desc[1]  = PA1    pin_desc[17] = PB1    pin_desc[33] = PC1
...                   ...                    ...
pin_desc[15] = PA15   pin_desc[31] = PB15   pin_desc[47] = PC15
```

这就是"**多个 GPIO chip，一个 pinctrl 控制器**"的硬件布局在 DTS 中的体现。

这条属性也回答了常见问题：**"为什么 GPIO 编号和 Pinctrl 引脚号不直接相等？"**——因为每个 GPIO bank 只管理 16 个引脚，但 pinctrl 管理所有 bank 拍平后的全部引脚，`gpio-ranges` 就是用 `(gpio_offset, pin_offset, count)` 三元组把它们连起来的翻译表。

#### 内核如何解析

`gpio-ranges` 在 GPIO 控制器注册时解析，时机很关键——它发生在 `gpiochip_add_data()` 中，早于任何 GPIO 请求：

```
gpiochip_add_data_with_key(gc, ...)
  └─ of_gpiochip_add(gc)                    ← DT 路径
      └─ of_gpiochip_scan_gpios(gc)          ← 扫描 gpio-ranges 属性
          └─ of_gpiochip_add_pin_range(chip)
              └─ 解析 DT 的 "gpio-ranges" 属性
                  └─ for_each entry:
                      gpiochip_add_pin_range(...)
                        → pinctrl_add_gpio_range(pctldev, range)
                          /* 注册到 pinctrl 的查找表 */
```

注册完成后，pinctrl 内部维护了一张映射表。当 `pinctrl_gpio_request(gpio_num)` 被调用时：

```
pinctrl_gpio_request(gpio_num)
  └─ pinctrl_find_gpio_range_from_pin()     ← 查表：gpio_num → (pctldev, offset)
      └─ 遍历 pctldev->gpio_ranges 链表
          └─ 找到 range->base <= gpio_num < range->base + range->npins
              └─ 返回 range
  └─ pin_request(pctldev, gpio_num - range->base, ...)  ← 用偏移请求 pinctrl
```

这就是"翻译"的全过程：**全局编号 → 查 gpio_ranges → 找到 pctldev + 偏移 → 调用 pin_request()**。

#### 这条属性解决的本质问题

`gpio-ranges` 解决的不仅仅是"编号转换"——它的存在使得 GPIO 核心层能够**在运行时反向查找** pinctrl 的引脚信息。没有它，GPIO 子系统就不知道一个全局编号对应哪个 pinctrl 控制器、哪个引脚，**冲突检测和后门机制都无法工作**。

### 5.3 后门机制：运行时自动配置（解决问题 2）

#### 为什么需要它

有了 `gpio-ranges`，GPIO 核心层能查表找到正确的 pinctrl 引脚。但问题 2 依然是：**驱动调 `gpiod_get()` 时，谁负责把引脚真正切换为 GPIO 功能？**

这个问题的答案**因 SoC 硬件架构而异**，不是统一的。

**情况 1：Pinctrl 和 GPIO 操作同一组寄存器（如 STM32）**

在 STM32 上，pinctrl 和 GPIO 面对的是同一个 MODER 寄存器（§5.1 矛盾所述）。驱动调 `gpiod_get()` → `gpiod_direction_output()` 时，GPIO core 需要在写 MODER 的同时让 pinctrl 知道状态变更——否则 pinctrl 的记录和硬件状态就会不一致。后门机制在此充当"同步桥"：GPIO 的 `direction_output` 内部调用 pinctrl 的 `gpio_set_direction` 完成 MODER 写入，两边状态一致。

**情况 2：Pinctrl 和 GPIO 是独立硬件块（如 i.MX6ULL）**

i.MX6ULL 上，IOMUXC（引脚复用控制器）和 GPIO 寄存器是**两个独立的硬件模块**。IOMUXC 负责功能选择，GPIO 只负责数据读写。此时 GPIO 驱动根本没有能力切换功能模式——它根本访问不到 IOMUXC 的地址空间。

这种情况下，后门机制**仅做冲突检测**（验证该引脚没有被其他外设占用），不做自动配置。驱动必须在 DTS 中显式写 `pinctrl-0` 让 pinctrl 在 probe 时配好 IOMUXC：

```dts
leds {
    compatible = "gpio-leds";
    pinctrl-0 = <&led_pin>;    /* i.MX6ULL 必须写——不配 IOMUXC，引脚不工作 */
    led0 {
        gpios = <&gpio1 5 GPIO_ACTIVE_LOW>;
    };
};
```

**情况 3：完全独立的 GPIO 外设（如 PCA9535）**

I2C GPIO 扩展器与 pinctrl 毫无关系，后门机制完全不涉及。

---

**所以 "后门机制" 的核心价值不是 "帮驱动自动配 MODER"，而是 "提供一个统一的冲突检测入口"**——至于是否兼做自动配置，是每个 SoC 自己决定的。

| SoC | Pinctrl 与 GPIO 硬件关系 | pinctrl-0 必须？ | 后门自动配 GPIO？ |
|-----|------------------------|----------------|-----------------|
| STM32MP257 | **合体**：MODER 同一寄存器 | 可选 | ✅ `gpio_set_direction` 写 MODER |
| i.MX6ULL | **独立**：IOMUXC ≠ GPIO 寄存器 | **必须** | ❌ 仅冲突检测 |
| PCA9535 | **外设**：与 pinctrl 无关 | 不需要 | ❌ 不走 pinctrl |

后门机制让前两种 SoC 共用同一套 GPIO API，底层行为差异由 pin controller 驱动决定。

#### 后门 1：请求时（GPIO request）——冲突检测

当驱动调用 `gpiod_get()` 时，调用链最终会走到 pinctrl：

```
gpiod_get()
  → gpiod_get_index()
    → gpiod_find_and_request()
      → gpiod_request()
        → gpiod_request_commit()
          → chip->request(chip, offset)      ← gpio_chip.request（厂商实现）
            → pinctrl_gpio_request(gpio_num)  ← 进入 pinctrl 层
              → pinmux_request_gpio()
                → pin_request(pctldev, pin, func="gpio", ...)
                  └─ 检查 mux_usecount：该 pin 有没有被其他驱动以非 GPIO 方式申请？
                      ├─ 有则检查 strict 标志（见 §3.6 场景 2）
                      └─ 无则记录 mux_owner，放行
                  └─ ops->gpio_request_enable?  ← 部分 SoC 在此配置 GPIO
                  └─ 记录 gpio_owner
```

这个调用链的作用：**验证当前引脚是否可被 GPIO 使用**（即冲突检测）。其中 `ops->gpio_request_enable` 是可选回调——如果 SoC 实现了它，`gpiod_get()` 阶段就会直接写入 MODER 完成 GPIO 模式切换。

**STM32 的选择是不实现 `gpio_request_enable`**，后门 1 只做验证：
- `stm32_gpio_request()`：检查引脚在 gpio-ranges 范围内 + RIF 安全权限验证
- `stm32_pmx_request()`：检查引脚合法性
- **不读写任何硬件寄存器**

这意味着 STM32 上 `gpiod_get()` 确实是"拿走描述符，不碰 MODER"——但这只是 STM32 的设计决定，不是后门 1 的通用约束。

#### 后门 2：方向设置时（GPIO direction）——真正切复用

STM32 把 MODER 的配置放在方向设置阶段：

```
gpiod_direction_output(desc, 0)
  → gpiod_direction_output_raw_commit()
    → chip->direction_output(chip, offset, 0)  ← gpio_chip.direction_output
      → 厂商先写 BSRR（设置输出电平）
      → pinctrl_gpio_direction_output(gpio_num) ← 再进入 pinctrl
        → pinmux_gpio_direction()
          → ops->gpio_set_direction(pctldev, range, pin, output)
            ← 厂商在此写入 MODER（GPIO mode, output）
```

**为什么 STM32 把 MODER 配置放在 direction 阶段，而不是 request 阶段？**

这是 STM32 的设计选择——将"验证所有权"（request）和"配置硬件"（direction）分成两个独立阶段。驱动调用 `gpiod_direction_output()` 是明确表示"我要开始用这个引脚了"的信号，此时切 MODER 语义清晰。

部分 SoC 采用相反的策略：在 request 阶段通过 `ops->gpio_request_enable` 直接切好 MODER，用户拿到 desc 时引脚已就绪，direction 只改方向位。两种方案无优劣之分，是不同 SoC 的实现偏好。

**为什么方向操作要经过 pinctrl？**

对 STM32 而言，"方向"不是独立的寄存器位——它是 MODER 寄存器的四种功能模式之一：

| MODER 值 | 含义 |
|----------|------|
| 0b00 | Input（既是 GPIO 方向，也是功能） |
| 0b01 | Output（既是 GPIO 方向，也是功能） |
| 0b10 | Alternate Function（复用到外设） |
| 0b11 | Analog（模拟模式） |

从 Alternate Function 切到 Output，不是"改一个方向位"那么简单——是把整个功能模式从"I2C1_SCL"改成"GPIO Output"。而**功能模式是 pinctrl 管的领域**，所以方向设置不得不进入 pinctrl 完成。

对比 I2C GPIO 扩展器（如 PCA9535）就完全不同——它有独立的配置寄存器控制方向，不涉及功能 mux，所以 `direction_output` 直接写配置寄存器，不需要 pinctrl 介入。STM32 没有这个独立性，MODER 身兼二职（方向 + 功能选择），方向操作只能走到 pinctrl 里。

#### 如果不走后门，驱动需要做什么

没有后门机制，驱动就得手动调用 pinctrl API：

```c
/* 假想代码：没有后门机制时驱动要做的 */
int probe(struct device *dev)
{
    /* step 1: 手动请求引脚控制权 */
    struct pinctrl *p = pinctrl_get(dev);
    struct pinctrl_state *s = pinctrl_lookup_state(p, "gpio");
    pinctrl_select_state(p, s);

    /* step 2: 再通过 gpio 操作数据 */
    struct gpio_desc *desc = gpiod_get(dev, "led");
    gpiod_direction_output(desc, 0);

    /* step 3: 释放时还得反过来 */
    ...
}
```

后门机制让驱动只需做第 2 步——第 1 步由 GPIO 核心层在 `gpiod_get()` 和 `gpiod_direction_output()` 中代为完成。**驱动开发者不需要知道 pinctrl 的存在。**

#### 设计权衡

不同的 SoC 可以选择在不同时机做 GPIO 复用配置：

| 时机 | 函数 | 典型用途 |
|------|------|---------|
| request 阶段 | `ops->gpio_request_enable()` | SoC 需要在请求时立即切 MODE |
| direction 阶段 | `ops->gpio_set_direction()` | SoC 在方向确定后才切 MODE（如 STM32） |
| 两者都做 | 两者配合 | 复杂 SoC，request 只验证，direction 再配置 |

这种灵活性是后门设计的核心——pinctrl 核心层只提供调用点，具体行为由厂商的 `pinmux_ops` 实现决定。

### 5.4 STM32 合体实现（解决问题 3）

#### 为什么需要合体——硬件上没有独立的 Pinctrl

STM32 的硬件布局决定了软件必须合体。看一个 GPIO bank 的寄存器布局：

```
GPIO bank 地址空间（如 gpioa: gpio@44240000）：
  偏移 0x00:  MODER    ← 模式选择（输入/输出/外设/模拟）
  偏移 0x04:  OTYPER   ← 输出类型（推挽/开漏）
  偏移 0x08:  OSPEEDR  ← 输出速度
  偏移 0x0C:  PUPDR    ← 上下拉配置
  偏移 0x10:  IDR       ← 输入数据
  偏移 0x14:  ODR       ← 输出数据
  偏移 0x20:  AFRL      ← 外设功能选择（低 8 个 pin）
  偏移 0x24:  AFRH      ← 外设功能选择（高 8 个 pin）
```

这是**一个完整的硬件模块**。没有独立的 Pinctrl 硬件单元——不存在像 i.MX6ULL 的 IOMUXC 那样的独立外设。"Pinctrl" 只是软件对 MODER、AFR 这些寄存器中"功能配置"位段的抽象命名。

问题在于 Linux 把"功能配置"和"数据"分成了两个子系统，但在 STM32 上它们落到了同一个寄存器上——**MODER**：

```
MODER 每个 pin 的 2-bit 编码：
  0b00 = Input          ← GPIO 管它叫"方向"
  0b01 = Output         ← GPIO 管它叫"方向"
  0b10 = Alternate Func ← Pinctrl 管它叫"功能"
  0b11 = Analog         ← Pinctrl 管它叫"功能"
```

**一个寄存器，同时被两个子系统声称所有权。** 如果拆成两个独立驱动：

```
PINCTRL 驱动（platform_driver）            GPIO 驱动（platform_driver）
  ├── iomap GPIOA 地址空间                   ├── iomap GPIOA 地址空间
  ├── 读 MODER → 改 pin 5 = AF → 写 MODER   ├── 读 MODER → 改 pin 5 = Out → 写 MODER
  ├── readl_relaxed()                        ├── readl_relaxed()
  └── writel_relaxed()                       └── writel_relaxed()
                                              ↑
                                      同一个寄存器，两个驱动都在做 read-modify-write
                                      没有协调 → 竞态，互相覆盖
```

两个驱动都 `iomap` 同一段物理地址，没有锁、没有协调、没有谁该管哪个 bit 的划分——因为 MODER 的 2-bit 编码本质上就无法"按职责分区"。

**合体是硬件设计迫使软件做的选择**：硬件上没有独立 Pinctrl → 寄存器边界不清晰 → 拆成两个驱动人为制造竞态 → 合体是最直接、最简单的解法。i.MX6ULL 能拆是因为 IOMUXC 和 GPIO 是两段独立的地址空间，硬件已经分好了。

#### 代码流程

```
stm32_pctl_probe()                   /* drivers/pinctrl/stm32/pinctrl-stm32.c */
  → stm32_pctrl_create_pins_tab()    /* 创建引脚软件描述表（PA0~PA15、PB0~PB15…），不碰硬件 */
  → stm32_pctrl_build_state()        /* 构建引脚功能组状态 */
  → devm_pinctrl_register()          /* 注册 pinctrl_dev（纯软件注册） */
  → clk_bulk_prepare_enable()        /* 使能所有 GPIO bank 时钟 ← 此时硬件才通电 */
  → 遍历 DT 子节点
    → for_each_gpiochip_node(dev, child)
      → of_address_to_resource()     /* 读 child 的 reg 属性，获得地址 */
      → devm_ioremap_resource()      /* 映射 GPIO 寄存器地址 ← 第一个硬件操作（但只映射，不写） */
      → stm32_gpiolib_register_bank()
        → bank->gpio_chip = stm32_gpio_template
        → gpiochip_add_data()        /* 注册 GPIO chip */
```

**注意**：probe 全程不读写 MODER/AFR 等控制寄存器。所有引脚的 MODER 保持复位值（Analog 模式 0b11），直到后门机制在 `gpiod_direction_output()` 中第一次写入。

**一个 probe 函数，同时完成了 Pinctrl 和 GPIO 两个子系统的注册。**

#### 请求阶段 vs 方向阶段的角色分工

STM32 明确将两个阶段分开：

**请求阶段（`stm32_gpio_request`）：只验证，不写寄存器**

```
stm32_gpio_request(chip, offset)
  → pinctrl_find_gpio_range_from_pin_nolock()  /* 验证该 pin 在 gpio-ranges 内 */
  → stm32_pmx_request()                         /* 只检查引脚合法性 */
  → RIF 安全权限检查                             /* MP257 新增：安全域隔离 */
```

不操作 MODER，不涉及任何硬件状态变更。

**方向阶段（`stm32_gpio_direction_output` / `stm32_gpio_direction_input`）：真正操作硬件**

```
stm32_gpio_direction_output(chip, offset, value)
  → __stm32_gpio_set(bank, offset, value)        /* 先写 BSRR 设置输出电平 */
  → pinctrl_gpio_direction_output(gpio_num)       /* 进入 pinctrl 层 */
    → stm32_pmx_gpio_set_direction()
      → stm32_pmx_set_mode(bank, pin, mode=1, alt=0)
        → 写 MODER[offset*2:offset*2+1] = 0b01    /* GPIO output mode */
        → 写 AFR[offset*4:offset*4+3] = 0         /* 清空复用选择 */
```

这就是后门机制中 `pinmux_gpio_direction()`→`ops->gpio_set_direction()` 在 STM32 上的落地——**修改 MODER 寄存器**。

#### 合体带来的好处

1. **DTS 简洁**：GPIO bank 节点不需要 compatible 属性，完全由 pinctrl 驱遍历子节点自动注册
2. **数据共享**：bank 结构体中的私有数据（如 `st,io-retime` 属性解析结果）和 pinctrl 的 pin table 在同一个驱动内自然可访问，无需跨驱动通信
3. **寄存器操作统一**：`stm32_pmx_set_mode()` 读写 MODER/AFR 和 GPIO 的数据操作在同一个代码单元中，虽然没有竞态问题（硬件寄存器本身就是所有 bank 共享的），但维护者只需要理解一个驱动的逻辑

多出的 RIF（Resource Isolation Framework）检查是 MP257 相比 MP157 的新增特性，用于安全域（Cortex-A35 Secure/Non-Secure）之间的引脚隔离。

## 6. 最新演进（Linux 4.4 ~ 6.x，2015~至今）

### 6.1 GPIO chardev 取代 sysfs

sysfs GPIO 接口（§2.2）虽然简单，但有几个无法修复的结构性缺陷：操作非原子（三步操作各自独立）、中断支持弱（只有 poll，无时间戳）、批量操作低效（一次一个文件写）、安全问题（全局 export 无归属管理）。

Linux 4.4（2015 年）引入 GPIO chardev，用 `/dev/gpiochipN` 替代 `/sys/class/gpio/`：

```
sysfs 时代：                          chardev 时代：
  echo 47 > /sys/class/gpio/export       open("/dev/gpiochip0")
  echo out > /sys/class/gpio/gpio47/...  ioctl(request line)
  echo 1 > /sys/class/gpio/gpio47/value  ioctl(set value)
```

chardev 的设计核心是 **line request 模型**——用户态通过 ioctl 向内核请求一组 GPIO line 的所有权，返回一个专用 fd。所有操作（读、写、监听事件）通过这个 fd 完成，不再有 export/unexport 的全局竞争。

对应的用户态工具 libgpiod（Linus Walleij / Bartosz Golaszewski 维护）封装了 ioctl 调用：

```shell
gpioset gpiochip0 5=1        # 设置 GPIO5 输出高电平
gpioget gpiochip0 5          # 读取 GPIO5 电平
gpioinfo                     # 查看所有 GPIO 信息
gpiomon gpiochip0 5          # 监听 GPIO5 中断事件
```

### 6.2 GPIO ABI v1 → v2

GPIO chardev 经历了两次 ABI：

| 版本 | 引入内核 | 核心改进 |
|------|---------|---------|
| v1 | Linux 4.4（2015） | 初始 chardev，line request + event fd |
| v2 | Linux 5.10（2020） | 方向可重配、去抖内建、line watch 监控、批量查询 |

v1 到 v2 的演进体现了实践中的需求积累：用户需要在 request 后动态切换方向而不释放 fd、需要硬件去抖而非软件 poll、需要知道其他进程何时占用了某个 GPIO。v2 是对 v1 四年实践的补齐。

v1 和 v2 的 ioctl 命令号不同，可在一个内核中共存。libgpiod v1（对应 v1 ABI）于 2022 年被 libgpiod v2 取代，两者 API 不兼容。

### 6.3 STM32MP257 新特性

相比 STM32MP157，MP257 的 pinctrl/GPIO 有以下变化：

**更多引脚**：GPIOA~GPIOK（11 bank × 16 pin）+ GPIOZ（1 bank × 8 pin），总计 184 个可编程引脚，比 MP157 多出约 30%。

**双 pinctrl 实例**：`pinctrl`（主域 @44240000）管理 GPIOA~K；`pinctrl_z`（安全域 @46200000）仅管理 GPIOZ。后者是 MP157 没有的，GPIOZ 的引脚由安全世界（OP-TEE）管理，Linux 需经过 RIF 防火墙授权才能访问。

**RIF 安全隔离**：MP257 新增的硬件防火墙机制。每个引脚可在 RIF 中配置为安全域独占 / 非安全域独占 / 共享（硬件信号量同步）。`stm32_gpio_request` 中新增了 RIF 权限检查，是 MP157 没有的步骤。

**高速接口时序控制**：新增 `st,io-retime`、`st,io-clk-edge`、`st,io-delay` 三个 pinctrl 属性，用于高速接口（FMC、DDR）的 IO 延迟调整，步长约几百 ps。

这些变化共同指向一个趋势：**随着 SoC 越来越复杂，pinctrl/GPIO 子系统正在从单纯的"引脚管理"向"安全隔离 + 高速时序控制"扩展。**

## 7. API 演化总览

| 时期 | 内核版本 | GPIO API | Pinctrl API | 用户态 | 关键局限 |
|------|---------|---------|-------------|--------|---------|
| 混沌时代 | 2.6.18 以前 | 架构私有（寄存器/函数） | 无（mach-* 散落） | 无 | 完全不可移植 |
| gpiolib 诞生 | 2.6.18 ~ 2.6.27 | `gpio_request()` 整型 | 无 | sysfs（2.6.27） | 全局编号无管理 |
| pinctrl 创建 | 3.2 ~ 3.3 | 整型 API | `pinctrl_register()` + pinmux | sysfs | 无 DT 绑定，pinconf 初期 |
| 驱动核心绑定 | 3.9 ~ 3.10 | 整型 API | `pinctrl_bind_pins()` 自动应用 | sysfs | GPIO 与 pinctrl 未联动 |
| 描述符革命 | 3.13 ~ 4.x | `gpiod_get()` 描述符 | DT 绑定标准化 | sysfs | 用户态仍薄弱 |
| chardev 时代 | 4.4 ~ 5.10 | 描述符 API | pinctrl 成熟 + gpio-ranges + 后门 | chardev v1（4.4） | 需学 libgpiod |
| 当前主流 | 5.10+ ~ 6.x | 描述符 API | pinctrl + gpio-ranges | libgpiod v2（5.10+） | — |

## 8. 总结

GPIO 和 Pinctrl 两个子系统的演化史，本质上是 Linux 在**通用性**和**硬件差异**之间寻找平衡的过程：

1. **从无到有**：gpiolib 解决了 GPIO 接口统一问题
2. **从散到聚**：pinctrl 解决了引脚复用/配置的标准化问题
3. **从分到合**：gpio-ranges + 后门机制 + 合体 probe 建立了两套子系统共管同一硬件时的协调桥梁

对于嵌入式驱动开发者，理解这段历史比背 API 更有价值——**当你看到一个 `gpiod_get()`，你应该知道它背后不只是拿到一个 GPIO 描述符，还通过后门机制完成了冲突检测，并在 `gpiod_direction_output()` 时由 pinctrl 把引脚配置为 GPIO 功能（STM32 这类合体 SoC 上）。** 对于 pinctrl 和 GPIO 硬件独立的 SoC（如 i.MX6ULL），DTS 中的 `pinctrl-0` 仍然是 GPIO 正常工作的前提。

