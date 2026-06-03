# 03. 初始化流程源码分析

> 本文是 STM32MP257 Pinctrl & GPIO 深度分析系列的第 3 篇。
> 从源码层面逐行追踪两个子系统的初始化路径。
>
> **前置:** [02-Architecture.md](02-Architecture.md) — 熟悉核心结构体和数据结构关系
> **下一篇:** [04-Scenario.md](04-Scenario.md) — 运行时情景分析
>
> **字数：约 20,500 词（含代码段）**
> **建议阅读时间：40~60 分钟**

---

## 3.1 从一个问题开始

系统启动后，你在终端敲：

```shell
$ ls /sys/kernel/debug/pinctrl/
44240000.pinctrl/

$ cat /sys/kernel/debug/gpio
gpiochip0: GPIOs 0-15, parent: platform/44240000.pinctrl, GPIOA
gpiochip1: GPIOs 16-31, parent: platform/44240000.pinctrl, GPIOB
...
gpiochip9: GPIOs 176-185, parent: platform/44240000.pinctrl, GPIOZ
```

问题是：**从系统上电到 pinctrl 注册完成、GPIO bank 全部就绪，内核里经历了什么？**

本文以 ATK 板的主域 pinctrl（`44240000.pinctrl`）为线索，追踪从内核启动到 `/sys/kernel/debug/gpio` 出现完整 GPIO 信息为止的全部代码路径。

### 初始化流程全景

```
时间 → 系统启动
        ↓
[1] Pinctrl Core 层注册
    core_initcall(pinctrl_init)
     └─ pinctrl_init_debugfs() — 创建 debugfs 根目录

[2] GPIO Core 层注册
    subsys_initcall(gpiolib_sysfs_init)
     └─ class_register(&gpio_class) — 创建 /sys/class/gpio/

[3] 驱动匹配与 probe 触发
    module_platform_driver(stm32mp257_pinctrl_driver)
     └─ platform bus 匹配 DTS → stm32_pctl_probe()
          │
          ├─ ① stm32_pctrl_get_irq_domain()     — IRQ domain
          ├─ ② stm32_pctrl_create_pins_tab()     — 引脚表（package 过滤）
          ├─ ③ stm32_pctrl_build_state()         — 分组状态
          ├─ ④ devm_pinctrl_register()           — 注册 pinctrl
          ├─ ⑤ 枚举 GPIO 子节点 → 取 clock/reset
          └─ ⑥ stm32_gpiolib_register_bank()     — 逐个注册 GPIO
               └─ gpiochip_add_data() → gpio_device
```

### 三部分代码的角色

| 代码 | 文件位置 | 作用 |
|------|---------|------|
| pinctrl core | `drivers/pinctrl/core.c` | 核心框架：pin_desc 管理、冲突检测、全局链表 |
| gpiolib core | `drivers/gpio/gpiolib.c` | GPIO 框架：gpio_device、字符设备、gpio-ranges 解析 |
| STM32 驱动 | `drivers/pinctrl/stm32/pinctrl-stm32.c` | 合体驱动：一个 probe 注册 pinctrl + GPIO |

---

## 3.2 阶段一：Core 层注册

### 3.2.1 Pinctrl Core — `core_initcall(pinctrl_init)`

**入口文件：** `drivers/pinctrl/core.c`

```c
static int __init pinctrl_init(void)
{
    pr_info("initialized pinctrl subsystem\n");
    pinctrl_init_debugfs();
    return 0;
}
core_initcall(pinctrl_init);
```

#### `core_initcall` 的展开过程

`core_initcall` 不是一个函数调用，而是一个**宏**，它把函数指针放在编译器生成的特定 ELF 段中：

```c
// include/linux/init.h:283, 301
#define core_initcall(fn)         __define_initcall(fn, 1)
#define __define_initcall(fn, id) ___define_initcall(fn, id, .initcall##id)
```

展开后相当于：

```c
// 编译器生成：
static initcall_entry_t __initcall_pinctrl_init1 __used
    __attribute__((__section__(".initcall1.init"))) = pinctrl_init;
```

**关键：** `pinctrl_init` 的函数指针被存放在了名为 `.initcall1.init` 的 ELF 段中。不是谁"调用了"它，而是编译器把地址写到了一个约定的位置。

#### 内核启动时如何找到它

内核的链接脚本（`arch/arm64/kernel/vmlinux.lds.S`）为每个 initcall 级别定义了起始和结束符号：

```
.initcall1.init : {
    __initcall1_start = .;
    *(.initcall1.init)    ← 所有 core_initcall 的函数指针集中在这里
    __initcall1_end = .;
}
```

启动时，`init/main.c` 中的 `do_initcalls()` 按级别遍历这些段：

```c
// init/main.c:1303
static void __init do_initcalls(void)
{
    int level;
    // 按级别 0→7 依次执行
    for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++)
        do_initcall_level(level, command_line);
}

// initcall_levels 数组存放每个段的起始地址
static initcall_entry_t *initcall_levels[] __initdata = {
    __initcall0_start,    // 早期
    __initcall1_start,    // core_initcall  ← pinctrl_init 在这里
    __initcall2_start,    // postcore_initcall
    __initcall3_start,    // arch_initcall
    __initcall4_start,    // subsys_initcall
    __initcall5_start,    // fs_initcall
    __initcall6_start,    // device_initcall (= module_init)
    __initcall7_start,    // late_initcall
};

// do_initcall_level 遍历该段的每个函数指针并调用
for (fn = initcall_levels[level]; fn < initcall_levels[level+1]; fn++)
    do_one_initcall(*fn);
```

所以 `core_initcall(pinctrl_init)` 的完整执行路径是：

```
内核启动
  → start_kernel()                     // init/main.c
    → kernel_init_freeable()
      → do_basic_setup()
        → do_initcalls()
          → 遍历 initcall_levels[1]（.initcall1.init 段）
            → 找到函数指针 pinctrl_init，调用它
              → pinctrl_init_debugfs()
              → return 0
```

`core_initcall` 的优先级是 1——数字越小执行越早。所有级别按 0→7 顺序执行，所以 `core_initcall`（级别 1）在 `module_init`（级别 6）之前执行。设计意图：pinctrl core 必须在任何 pin controller 驱动之前就绪。

#### `pinctrl_init` 实际做了什么

它只做了一件事：

```c
pinctrl_init_debugfs();  // debugfs_create_dir("pinctrl")
```

创建 `/sys/kernel/debug/pinctrl/` 目录。此时目录是空的——子目录（如 `44240000.pinctrl/`）要等 `pinctrl_enable()` 时逐个创建。

**pinctrl core 没有注册总线类型。** 不像 MMC 有 `mmc_bus_type`，pin controller 直接挂在 platform bus 上，消费者设备通过 DTS 中 `pinctrl-0` 属性的 phandle 间接引用它，不需要总线匹配机制。

### 3.2.2 GPIO Core — 初始化入口

与 pinctrl core 不同，GPIO core 在 `subsys_initcall` 阶段通过两个入口完成初始化：

**入口 1：`gpiolib_core_init()` — `fs_initcall`（优先级 5）**

```c
static int __init gpiolib_core_init(void)
{
    ...
}
fs_initcall(gpiolib_core_init);
```

这个 `fs_initcall` 展开为放在 `.initcall5.init` 段中：

```c
// 等价于编译器生成：
static initcall_entry_t __initcall_gpiolib_core_init5
    __attribute__((__section__(".initcall5.init"))) = gpiolib_core_init;
```

级别 5 在 `core_initcall`（级别 1）之后执行。这个入口做一些非常早期的内部初始化（如 spinlock 初始化），但不创建任何用户态可见的接口。

**入口 2：`gpiolib_sysfs_init()` — `subsys_initcall`（优先级 4）**

```c
// drivers/gpio/gpiolib-sysfs.c
static int __init gpiolib_sysfs_init(void)
{
    int status;
    struct gpio_device *gdev;

    class_register(&gpio_class);    // 创建 /sys/class/gpio/

    list_for_each_entry(gdev, &gpio_devices, list) {
        /* 对已注册的 GPIO chip 补注册 sysfs 接口 */
        status = gpiochip_sysfs_register(gdev);
    }
    return 0;
}
subsys_initcall(gpiolib_sysfs_init);
```

注意这里**级别 4 比级别 5 更早执行**——数字越小越早。所以 `gpiolib_sysfs_init`（级别 4）反而在 `gpiolib_core_init`（级别 5）之前执行。设计意图：sysfs 接口（/sys/class/gpio/）必须先就绪，因为后续任何 GPIO chip 注册时都需要 sysfs 路径。而 `gpiolib_core_init` 只做一些内部初始化，可以排在后面。

`gpiolib_sysfs_init` 干了两件事：

1. **`class_register(&gpio_class)`**：在内核设备模型中注册 `gpio_class`，在 sysfs 下创建 `/sys/class/gpio/`。这是 legacy sysfs GPIO 接口的根目录。

2. **遍历 `gpio_devices` 全局链表**：在 `gpiolib_sysfs_init` 执行之前，可能已有 GPIO chip 通过 `gpiochip_add_data()` 注册了（某些驱动在更早的 initcall 级别 probe）。当时 `gpio_class` 还没创建，所以这些 chip 的 sysfs 入口还没建立。这里补注册。

> **注意**：`gpio_class` 是 **sysfs legacy 接口**（`echo N > /sys/class/gpio/export`）用的，不是 `/dev/gpiochipN`。字符设备 `/dev/gpiochipN` 通过 `cdev_add()` 在 `gpiochip_add_data()` 内部注册，不依赖 `gpio_class`。

### 3.2.3 Core 层注册完成后的状态

```
内核启动完成 core_initcall 阶段后：
  pinctrl debugfs: /sys/kernel/debug/pinctrl/  （空目录）

内核启动完成 subsys_initcall 阶段后：
  GPIO sysfs:     /sys/class/gpio/              （空目录，尚无 gpiochipN）
  GPIO devices:   gpio_devices 链表              （空链表）
```

两个 core 层已经就绪，现在等待具体的 pin controller 驱动 probe。

---

## 3.3 阶段二：驱动注册与匹配

### 3.3.1 硬件描述：DTS 中的 pin controller 节点

先看硬件是怎么描述的。ATK 板主域 pin controller 在 DTS 中的定义：

```dts
// ATK 板主域 pin controller 节点（arch/arm64/boot/dts/st/stm32mp257.dtsi）
pinctrl: pinctrl@44240000 {
    compatible = "st,stm32mp257-pinctrl";
    reg = <0x44240000 0x400>;              // 寄存器基地址 + 长度
    ranges = <0 0x44240000 0x400>;          // 子节点地址映射
    st,syscfg = <&syscfg 0x700 0xff>;       // 系统配置寄存器（中断路由）

    // GPIO bank 子节点——每个 bank 一段独立的寄存器空间
    gpioa: gpio@44240000 {
        compatible = "st,stm32mp257-gpio";
        reg = <0x0 0x400>;                  // 相对父节点偏移
        gpio-ranges = <&pinctrl 0 0 16>;    // 桥梁：全局编号 → pin 号
        interrupts-extended = <&exti1 1 IRQ_TYPE_LEVEL_HIGH>;  // 中断到 EXTI
        ...
    };
    gpiob: gpio@44250000 {
        compatible = "st,stm32mp257-gpio";
        reg = <0x100 0x400>;
        gpio-ranges = <&pinctrl 16 16 16>;
        ...
    };
    // ... 直至 GPIOI (9 个 bank)
};
```

pin controller 节点包含两类信息：

| 属性 | 含义 | 用途 |
|------|------|------|
| `compatible` | 驱动匹配标识 | platform bus 据此找到 `stm32mp257_pinctrl_driver` |
| `reg` | 寄存器物理地址 | probe 时做 `ioremap`，将硬件寄存器映射到内核虚拟地址空间 |
| `ranges` | 子节点地址翻译 | 告诉内核子节点 `reg` 中的偏移如何转为物理地址 |
| `st,syscfg` | SYSCFG 系统配置寄存器 | GPIO → EXTI 中断信号路由选择。每个 GPIO pin 的中断信号需要选通到 EXTI 控制器，SYSCFG 中的 irqmux 寄存器决定"GPIOA pin 0"还是"GPIOB pin 0"连到 EXTI line 0。probe 中通过 `syscon_regmap_lookup_by_phandle(np, "st,syscfg")` 获取 regmap，然后为每组 16 个 IRQ line 分配 regmap_field |
| GPIO 子节点 | 各 bank 的寄存器空间 | probe 中通过 `for_each_gpiochip_node()` 遍历并注册 |

还有第二个 pin controller 实例——安全域 pinctrl_z：

```dts
pinctrl_z: pinctrl@46200000 {
    compatible = "st,stm32mp257-z-pinctrl";
    reg = <0x46200000 0x400>;

    gpioz: gpio@46200000 {
        compatible = "st,stm32mp257-gpio";
        gpio-ranges = <&pinctrl_z 0 176 10>;  // GPIOZ 全局编号以 176 为起点
        ...
    };
};
```

**关键点**：虽然 GPIO 子节点也有 `compatible = "st,stm32mp257-gpio"`，但它们**不会作为独立的 platform device 在 platform bus 上走一遍匹配→probe 流程**。它们由父节点 pinctrl 的 probe 内部通过 `for_each_gpiochip_node()` 遍历子节点、手动读取 `reg`/`gpio-ranges`/`clocks` 等属性、然后逐个 `gpiochip_add_data()` 注册。这就是"合体驱动"的含义——pinctrl 和 GPIO 共享同一个 `pinctrl@44240000` 节点，一个 probe 函数负责完成两个子系统的全部初始化。

### 3.3.2 平台驱动结构：如何描述"我能匹配哪个节点"

驱动代码中定义一个 `platform_driver` 结构体，声明自己能匹配哪个 compatible：

```c
// pinctrl-stm32mp257.c:2576
static struct platform_driver stm32mp257_pinctrl_driver = {
    .probe  = stm32_pctl_probe,                     // 匹配成功时调用
    .driver = {
        .name           = "stm32mp257-pinctrl",
        .of_match_table = stm32mp257_pctrl_match,    // 匹配表
        .pm             = &stm32_pinctrl_dev_pm_ops, // 电源管理回调
    },
    // 没有 .id_table——DT-only 驱动不需要
    // .id_table 用于非 DT 平台（如 ACPI、legacy platform_device），
    // 这里只依赖 DTS compatible 匹配
};
```

**注意 `platform_driver` 还有一个可选字段 `.id_table`，但这里没有提供。** `id_table` 用于非设备树平台（如 ACPI、Legacy platform device 的 name 匹配）。在 DT-only 的系统中，`of_match_table` 就足够了——platform bus 的匹配逻辑是：有 DT 节点时优先用 `of_match_table` 匹配 compatible，没有 DT 时才用 `id_table` 匹配 name。

`.of_match_table` 指向设备树匹配表：

```c
// pinctrl-stm32mp257.c:2559
static const struct of_device_id stm32mp257_pctrl_match[] = {
    {
        .compatible = "st,stm32mp257-pinctrl",
        .data = &stm32mp257_match_data,     // ← 匹配后传给 probe
    },
    {
        .compatible = "st,stm32mp257-z-pinctrl",
        .data = &stm32mp257_z_match_data,
    },
    { }                                      // 空条目终止
};
```

**`platform_driver.of_match_table` 指向了上面的 `stm32mp257_pctrl_match[]` 数组**。箭头关系如下：

```
stm32mp257_pinctrl_driver.driver.of_match_table
                │
                ▼
     stm32mp257_pctrl_match[]  ← 匹配表
        [0] { .compatible = "st,stm32mp257-pinctrl",
              .data = &stm32mp257_match_data       }
        [1] { .compatible = "st,stm32mp257-z-pinctrl",
              .data = &stm32mp257_z_match_data       }
        [2] { /* 空终止 */ }
```

platform bus 匹配时遍历 `of_match_table` 指向的数组，将数组每个条目的 `.compatible` 与 DTS 节点的 `compatible` 属性逐条对比。匹配成功后，该条目的 `.data` 指针通过 `device_get_match_data(dev)` 传递给 probe 函数。

两个 `.data` 分别指向主域和安全域的匹配数据：

```c
// 主域（对应 pinctrl@44240000）
static struct stm32_pinctrl_match_data stm32mp257_match_data = {
    .pins            = stm32mp257_pins,            // 主域引脚描述表
    .npins           = ARRAY_SIZE(stm32mp257_pins), // 198（所有封装）
    .secure_control  = true,
    .io_sync_control = true,
    .rif_control     = true,
};

// 安全域（对应 pinctrl@46200000）
static struct stm32_pinctrl_match_data stm32mp257_z_match_data = {
    .pins            = stm32mp257_z_pins,            // 安全域引脚描述表
    .npins           = ARRAY_SIZE(stm32mp257_z_pins),// ~150
    .secure_control  = true,
    .io_sync_control = true,
    .rif_control     = true,
};
```

`stm32mp257_pins[]` 是 SoC 厂商在 `pinctrl-stm32mp257.c` 中写死的引脚描述表，每个引脚包含编号、名称和所有支持的 alternate function。以 PA0 为例：

```c
// stm32mp257_pins[] 的第一项
STM32_PIN_PKG(
    PINCTRL_PIN(0, "PA0"),     // pin 号 = 0, 名称 = "PA0"
    STM32MP_PKG_AI | STM32MP_PKG_AJ | STM32MP_PKG_AK | STM32MP_PKG_AL,  // 支持的封装
    STM32_FUNCTION(0, "GPIOA0"),         // AF0: GPIO
    STM32_FUNCTION(2, "LPTIM1_CH2"),     // AF2: LPTIM1 通道 2
    STM32_FUNCTION(4, "UART8_CTS"),      // AF4: UART8 CTS
    STM32_FUNCTION(6, "UART5_TX"),       // AF6: UART5 TX
    STM32_FUNCTION(7, "USART3_TX"),      // AF7: USART3 TX
    STM32_FUNCTION(15, "DCMI_D9 PSSI_D9 DCMIPP_D9"), // AF15: 数字摄像头
    STM32_FUNCTION(16, "EVENTOUT"),      // AF16: 事件输出
    STM32_FUNCTION(17, "ANALOG")         // AF17: 模拟模式
),
```

`STM32MP_PKG_AI` / `AJ` / `AK` / `AL` 是封装类型标志位。STM32MP257 有多种 LBGA 封装规格，引脚总数不同：

| 封装 | 说明 |
|------|------|
| `STM32MP_PKG_AI` | LBGA 448 引脚（完整封装）|
| `STM32MP_PKG_AJ` | LBGA 292 引脚（ATK 板使用的封装）|
| `STM32MP_PKG_AK` | LBGA 289 引脚 |
| `STM32MP_PKG_AL` | LBGA 272 引脚 |

PA0 在四种封装上都存在，所以四个标志全标。如果某个引脚只在特定封装上存在（如大封装的 PK7 不在 AJ 封装上），就只标对应封装的标志。probe 时 `stm32_pctrl_create_pins_tab()` 根据 DTS 中 `st,package` 属性（ATK 板为 `STM32MP_PKG_AJ`）只保留当前封装实际存在的引脚。

`stm32mp257_z_pins[]` 结构相同，但引脚的 pinctrl pin 号从 400 开始：

```c
// stm32mp257_z_pins[] 的第一项
STM32_PIN_PKG(
    PINCTRL_PIN(400, "PZ0"),     // pin 号 = 400（安全域），名称 = "PZ0"
    STM32MP_PKG_AI | STM32MP_PKG_AJ | STM32MP_PKG_AK | STM32MP_PKG_AL,
    STM32_FUNCTION(0, "GPIOZ0"),         // AF0: GPIO
    STM32_FUNCTION(4, "SPI8_MOSI"),      // AF4: SPI8 MOSI
    STM32_FUNCTION(7, "LPUART1_TX"),     // AF7: LPUART1 TX
    STM32_FUNCTION(9, "I2C8_SDA"),       // AF9: I2C8 SDA
    STM32_FUNCTION(16, "EVENTOUT"),
    STM32_FUNCTION(17, "ANALOG")
),
```

`.pins` 指向这些数据，`.npins` 是数组长度。probe 时 `stm32_pctrl_create_pins_tab()` 会遍历这个数组，根据 `st,package` 属性过滤掉当前封装不存在的引脚，只保留可用引脚。主域过滤后 ATK 板上为 176 个（PA0~PK7）。

两个实例的 `.secure_control` / `.rif_control` / `.io_sync_control` 全为 `true`——安全控制标志相同，区别只在 `pins` 指向不同的物理引脚集合。两个条目对应两个硬件实例：

| DTS compatible | 硬件实例 | 地址 | 管理引脚 | 安全域 |
|---------------|---------|------|---------|--------|
| `st,stm32mp257-pinctrl` | 主域 pinctrl | 0x44240000 | GPIOA~I (176 个) | 非安全（Non-secure）|
| `st,stm32mp257-z-pinctrl` | 安全域 pinctrl_z | 0x46200000 | GPIOZ (10 个) | 安全（Secure，TZCD 保护）|

**为什么需要两个独立的 pinctrl 实例？** 不是"一种驱动注册了两个 chip"，而是硬件上就有两块完全独立的 IP 块，各自的寄存器基地址不同（0x44240000 vs 0x46200000），各自有自己的 compatible 字符串和 DTS 节点，各自触发一次 `stm32_pctl_probe`。GPIOZ 所在的 pinctrl_z 属于 TZCD（TrustZone Controller Device）安全域，通常仅由 OP-TEE 或 TF-A 控制，Linux 能否访问取决于固件启动时的安全配置。ATK 板上 GPIOZ 被配置为非安全可访问，所以 Linux 也能看到它。安全域分离的目的是防止非安全世界（Linux）操作关键安全引脚（如 tamper 检测、RTC 备份域信号）。

### 3.3.3 `module_platform_driver`：驱动如何注册到内核

```c
// pinctrl-stm32mp257.c:2584
module_platform_driver(stm32mp257_pinctrl_driver);
```

这个宏在 `include/linux/platform_device.h` 中展开为：

```c
#define module_platform_driver(__platform_driver) \
    module_driver(__platform_driver, platform_driver_register, \
                  platform_driver_unregister)
```

最终生成两个 initcall 函数，放在 `.initcall6.init`（`module_init` 级别）：

```c
// 初始化函数——放在 .initcall6.init 段
static int __init stm32mp257_pinctrl_init(void)
{
    return platform_driver_register(&stm32mp257_pinctrl_driver);
}
module_init(stm32mp257_pinctrl_init);
// → __attribute__((__section__(".initcall6.init"))) = stm32mp257_pinctrl_init

// 退出函数（模块卸载时）
static void __exit stm32mp257_pinctrl_exit(void)
{
    platform_driver_unregister(&stm32mp257_pinctrl_driver);
}
module_exit(stm32mp257_pinctrl_exit);
```

`module_init` 是级别 6，而之前介绍的 pinctrl core 的 `core_initcall` 是级别 1，GPIO core 的 `subsys_initcall` 是级别 4。所以它们在内核启动时的执行顺序是：

```
内核启动 → do_initcalls()
  level 1 (.initcall1):  pinctrl_init()              ← pinctrl core 先就绪
  level 4 (.initcall4):  gpiolib_sysfs_init()        ← GPIO sysfs 类注册
  level 6 (.initcall6):  stm32mp257_pinctrl_init()   ← 驱动注册
    └─ platform_driver_register()
```

### 3.3.4 从 initcall 到 probe 的完整链路

`stm32mp257_pinctrl_init()` 调 `platform_driver_register()`，内核内部发生以下匹配过程：

```
platform_driver_register(&stm32mp257_pinctrl_driver)   ← 驱动只注册一次
  └─ driver_register(&drv->driver)                    // 注册到 platform bus
       └─ bus_add_driver()                            // 挂入总线
            └─ driver_attach(drv)                     // 遍历已注册设备，找匹配
                 │
                 for each device on platform bus:
                   │
                   ├─ "44240000.pinctrl"
                   │    of_match_device(dev->of_node)
                   │      → compatible "st,stm32mp257-pinctrl"      ← 匹配！
                   │      → match_entry = &stm32mp257_pctrl_match[0]
                   │      → match_data  = &stm32mp257_match_data
                   │      → driver->probe(dev) → stm32_pctl_probe(pdev)  ①
                   │
                   └─ "46200000.pinctrl_z"  (安全域)
                        of_match_device(dev->of_node)
                          → compatible "st,stm32mp257-z-pinctrl"    ← 匹配！
                          → match_entry = &stm32mp257_pctrl_match[1]
                          → match_data  = &stm32mp257_z_match_data
                          → driver->probe(dev) → stm32_pctl_probe(pdev)  ②
```

所以 **一个驱动注册、两个 DTS 节点、两次 probe**：

| 调用 | pdev 实例 | DTS 节点 | match_data |
|------|-----------|----------|------------|
| ① 主域 | `44240000.pinctrl` | `pinctrl@44240000` | `stm32mp257_match_data` |
| ② 安全域 | `46200000.pinctrl_z` | `pinctrl@46200000` | `stm32mp257_z_match_data` |

**关键细节：`platform_driver_register()` 只注册驱动，不直接调用 probe。** Probe 触发是 `driver_register()` 内部的 `driver_attach()` 遍历已注册设备时发生的。如果设备被编译为模块（`.ko`），probe 在 `insmod` 时触发。如果设备编译进内核，probe 在 `do_initcalls()` 执行到级别 6 时触发。匹配成功后，`of_match_table` 条目中的 `.data` 指针通过 `device_get_match_data()` 传递给 probe（见 3.3.2 中 `stm32mp257_match_data` / `stm32mp257_z_match_data` 的定义）。

至此，probe 函数被调用，`match_data` 传入。下一节开始进入 `stm32_pctl_probe()` 的逐段源码分析。

---

## 3.4 阶段三：Probe 逐段分析 — `stm32_pctl_probe()`

现在进入 `stm32_pctl_probe()` 的逐段分析。在逐行跟踪代码之前，先总览这个函数做了什么——它实际上是把 02 章中定义的所有关键数据结构从"静态描述"变成"运行时实例"的过程。

### 3.4.0 Probe 总览：数据结构施工图

输入 `stm32_pctl_probe()` 时，只有两个东西：一个 `platform_device` 指针（描述 DTS 节点），一个 `match_data` 指针（指向 `stm32_pinctrl_match_data`，即 SoC 写死的引脚描述表）。退出时，pinctrl 和 GPIO 两个子系统的全部运行时数据结构都注册完成。

下面是 **probe 步骤与 02 核心结构体** 的对应关系——每步都对应着创建或填充 02 中某个关键结构体。probe 流程按时间顺序如下——

```
stm32_pctl_probe(pdev, match_data)
  │
  │  [准备工作]
  │
  ├── ① stm32_pinctrl *pctl = devm_kzalloc()
  │     02-§2.5: 分配"总管"结构体，目前全空
  │
  ├── ② stm32_pctrl_get_irq_domain()
  │     → pctl->domain = irq_domain_create_hierarchy()
  │     02-§3.6: GPIO 中断的 irq_domain（parent = EXTI 控制器）
  │
  ├── ③ hwspinlock（可选）
  │     → pctl->hwlock
  │
  │  [Pinctrl 侧注册]
  │
  ├── ④ stm32_pctrl_create_pins_tab()
  │     → pctl->pins[] = stm32_desc_pin[]（拷贝 + 过滤）
  │     02-§2.5: 从 match_data.pins 复制到可写内存，只保留当前封装的引脚
  │
  ├── ⑤ stm32_pctrl_build_state()
  │     → pctl->groups[176] = stm32_pinctrl_group[]
  │     02-§2.5: 一 pin 一组，group 名即引脚名
  │
  ├── ⑥ 填充 pinctrl_desc → devm_pinctrl_register()
  │     ├── pinctrl_desc.pins = 从 stm32_desc_pin[] 提取的 pinctrl_pin_desc[]
  │     ├── pinctrl_desc.pctlops = stm32_pctrl_ops   （02-§2.7）
  │     ├── pinctrl_desc.pmxops  = stm32_pmx_ops     （02-§2.7）
  │     ├── pinctrl_desc.confops = stm32_pconf_ops   （02-§2.7）
  │     │
  │     └── pinctrl_register() 内部（02-§2.5, §2.6）:
  │           ├── pinctrl_init_controller()
  │           │     └── pinctrl_register_pins()
  │           │           02-§2.6: 为每个引脚创建 pin_desc[]
  │           │                     → 插入 pinctrl_dev->pin_desc_tree（radix tree）
  │           └── pinctrl_enable()
  │                 02-§2.5: 挂入 pinctrldev_list 全局链表
  │                 02-§2.5: pinctrl_dev->gpio_ranges 链表初始化（空）
  │                 02-§2.4: debugfs 目录创建
  │
  │  [GPIO 侧注册]
  │
  ├── ⑦ 枚举 GPIO bank 子节点
  │     → for_each_gpiochip_node() 取 9 个 bank 的 clock + reset
  │     → clk_bulk_prepare_enable()
  │
  └── ⑧ for_each GPIO bank → stm32_gpiolib_register_bank()
         └── gpiochip_add_data(&bank->gpio_chip, bank)
               02-§3.2, §3.3:
                 ├── 分配 gpio_device（02-§3.3）
                 ├── 分配 gpio_desc[16]（02-§3.5）
                 ├── gpiochip_find_base() → 动态编号
                 ├── gpiodev_add_to_list()
                 │
                 └── of_gpiochip_add()
                       ├── of_gpiochip_add_pin_range()
                       │     02-§4.3, §4.4:
                       │       gpio-ranges → pinctrl_gpio_range
                       │         ├── .node → pinctrl_dev->gpio_ranges
                       │         └── .node2 → gpio_device->pin_ranges
                       └── of_gpiochip_scan_gpios()
```

**两个子系统之间的桥梁**：注意到步骤⑥创建了 `pinctrl_dev->gpio_ranges` 链表（此时空链），步骤⑧中 `of_gpiochip_add_pin_range()` 逐个创建 `pinctrl_gpio_range` 并挂入。这正是 02-§4.4 描述的"双链表"安装流程——每个 range 同时挂在 pinctrl_dev 和 gpio_device 上，实现 pinctrl pin 号 ↔ GPIO 全局编号 的双向翻译。

`stm32_pinctrl`（02-§2.5 `driver_data`）是这个函数的"施工场地"，probe 期间逐一填充它的各个字段，最终得到一张完整的数据关系网：

```
stm32_pinctrl（pctl）—— 02-§2.5
├── dev         → &pdev->dev
├── pctl_dev    → pinctrl_dev *（⑥ 注册后返回）—— 02-§2.5
├── pctl_desc   → pinctrl_desc（填充后用来注册）—— 02-§2.4
├── groups[]    → stm32_pinctrl_group[176]（⑤ 时创建）—— 02-§2.5
├── banks[]     → stm32_gpio_bank[9]（⑧ 时创建）—— 02-§2.5
├── pins[]      → stm32_desc_pin[]（④ 时创建）—— 02-§2.5
├── domain      → irq_domain *（② 时创建）
├── match_data  → stm32_pinctrl_match_data（① 传入）—— 02-§2.6
└── nbanks      → bank 计数器
```

每个 GPIO bank 又是独立的结构体：

```
stm32_gpio_bank[n]—— 02-§2.5
├── base       → 寄存器虚拟地址（ioremap）
├── gpio_chip  → gpio_chip（从 stm32_gpio_template 拷贝）—— 02-§3.4
│                 ├── .request / .free / .get / .set / ...
│                 └── .parent = &pdev->dev
├── range      → pinctrl_gpio_range（内嵌，注册时填入双链表）—— 02-§4.3
└── lock       → spinlock_t（保护本 bank 并发）
```

### 3.4.1 获取平台数据与分配主结构体

```c
// pinctrl-stm32.c:2011
int stm32_pctl_probe(struct platform_device *pdev)
{
    const struct stm32_pinctrl_match_data *match_data;
    struct fwnode_handle *child;
    struct device *dev = &pdev->dev;
    struct stm32_pinctrl *pctl;
    struct pinctrl_pin_desc *pins;
    int i, ret, hwlock_id;
    unsigned int banks;

    // ① 获取平台数据（来自 of_match_table 的 .data 字段）
    match_data = device_get_match_data(dev);
    if (!match_data)
        return -EINVAL;

    // ② 分配主结构体 stm32_pinctrl（驱动私有数据）
    pctl = devm_kzalloc(dev, sizeof(*pctl), GFP_KERNEL);
    if (!pctl)
        return -ENOMEM;

    platform_set_drvdata(pdev, pctl);
    pctl->dev = dev;
```

此时 `pctl` 的各个字段状态（`devm_kzalloc` 初始化为全 0/NULL）：

```
pctl->dev         → &pdev->dev                     // 已赋值
pctl->match_data  → &stm32mp257_match_data          // 来自 device_get_match_data()
                     ├── .pins = stm32mp257_pins[]  // 约 200 条静态引脚描述（写死）
                     ├── .npins = 198               // AI 完整封装引脚数
                     ├── .secure_control = true
                     ├── .io_sync_control = true
                     └── .rif_control = true
pctl->pctl_dev    → NULL                            // 未注册
pctl->pctl_desc   → {0}                             // 未填充
pctl->groups      → NULL                            // 未创建
pctl->banks       → NULL                            // 未创建
pctl->pins        → NULL                            // 未创建
pctl->domain      → NULL                            // 未创建
pctl->nbanks      → 0
```

`match_data` 指向 `stm32mp257_match_data`，这是 of_match_table 中 `compatible = "st,stm32mp257-pinctrl"` 对应的数据。`match_data->pins` 指向 `stm32mp257_pins[]`，一个约 200 条目的静态引脚描述表，写死在 `pinctrl-stm32mp257.c` 中，描述主域每个引脚的各种 alternate function（如 PA0 支持 AF2_LPTIM1_CH2、AF4_UART8_CTS、AF7_USART3_TX 等）。注意这些数据目前是 const 只读的——后面需要拷贝到可写内存才能按封装过滤。

两个返回值检查：
- `if (!match_data) return -EINVAL`：如果 of_match_table 中匹配到的条目没有 `.data`（或 DTS 中没有 compatible），直接返回无效参数——这种情况在编译正确的 DTB + 驱动中不会发生，但作为防御性编程保留。
- `if (!pctl) return -ENOMEM`：内存不足。`devm_kzalloc` 是 devres 管理的分配，不需要手动 free。

### 3.4.2 前置：获取 IRQ domain

```c
// pinctrl-stm32.c:2033
pctl->domain = stm32_pctrl_get_irq_domain(pctl);
if (IS_ERR(pctl->domain))
    return PTR_ERR(pctl->domain);
if (!pctl->domain)
    dev_warn(dev, "pinctrl without interrupt support\n");
```

这是 probe 中第一件实际工作——创建中断控制器 domain。pinctrl 同时也是 GPIO 中断控制器，需要在注册 GPIO bank 之前建立 IRQ 基础设施（因为 `gpiochip_add_data` 时会配置中断）。

`stm32_pctrl_get_irq_domain()` 为每个 GPIO bank 创建一个层次化 irq domain。ATK 板走的是 `interrupts-extended` 路径（`pinctrl-stm32.c:1823`），因为每个 GPIO bank 的 DTS 节点中是这样声明的：

```dts
gpioa: gpio@44240000 {
    interrupts-extended = <&exti1 1 IRQ_TYPE_LEVEL_HIGH>;
};
```

函数解析这个属性，找到父 domain（EXTI 控制器），然后调用 `irq_domain_create_hierarchy(parent, 0, 16, fwnode, &stm32_gpio_domain_ops, pctl)`，为这个 bank 创建一个能处理 16 个中断源 (STM32_GPIO_IRQ_LINE) 的子 domain。

注册后的 `pctl->domain` 指向以下层次链：

```
pctl->domain  ─→ pinctrl_irq_domain（子 domain）
                     parent ─→ exti_domain（EXTI 控制器）
                                   parent ─→ irq_domain（GICv3）
```

如果 parent domain（EXTI）尚未注册，`irq_find_host()` 返回 NULL，probe 返回 `-EPROBE_DEFER`——这是为什么 IRQ domain 必须最先创建的原因之一：如果先做了 pinctrl 注册，再因为 IRQ 问题退回去，前面的工作就白费了。

### 3.4.3 硬件自旋锁（可选）

```c
// pinctrl-stm32.c:2041
hwlock_id = of_hwspin_lock_get_id(pdev->dev.of_node, 0);
if (hwlock_id < 0) {
    if (hwlock_id == -EPROBE_DEFER)
        return hwlock_id;
} else {
    pctl->hwlock = hwspin_lock_request_specific(hwlock_id);
}
```

硬件自旋锁用于在**多核系统**中保护 GPIO 寄存器访问的原子性。A35 的两个核可能同时操作同一个 GPIO bank 的寄存器，硬件自旋锁可以防止竞争。如果 DTS 中没定义（`hwlock_id < 0` 且不是 EPROBE_DEFER），`pctl->hwlock = NULL`，后续寄存器操作不加硬件锁，只使用 spin_lock。

ATK 板主域 pinctrl 的 DTS 节点没有声明 `hwlocks` 属性，所以 `of_hwspin_lock_get_id()` 返回 `-ENOSYS`，`pctl->hwlock = NULL`。这意味着后续所有 GPIO 寄存器操作（如 `stm32_gpio_set()`）只使用 `spin_lock_irqsave(&bank->lock, flags)` 保护——这是纯软件自旋锁，适用于 A35 两个核间同一时间只有一个访问 GPIO 寄存器的情况。如果板级方案需要 A35 + M33 协处理器共享 GPIO，可以在 DTS 中添加 `hwlocks = <&hsem 1>` 让驱动获取硬件信号量（HSEM）实现核间同步。

### 3.4.4 创建引脚表（package 过滤）

```c
// pinctrl-stm32.c:2051-2064
pctl->match_data = match_data;

// 可选：获取封装信息
if (!device_property_read_u32(dev, "st,package", &pctl->pkg))
    dev_dbg(pctl->dev, "package detected: %x\n", pctl->pkg);

// 分配引脚表空间（最大 npins = 198）
pctl->pins = devm_kcalloc(pctl->dev, pctl->match_data->npins,
                          sizeof(*pctl->pins), GFP_KERNEL);

// 创建引脚表（package 过滤）
ret = stm32_pctrl_create_pins_tab(pctl, pctl->pins);
```

传递 `pctl->pins` 作为第二实参，函数内部的形参 `pins` 就指向这块已分配的内存——后续 `pins->pin = p->pin` 和 `pins++` 都是往 `pctl->pins` 缓冲区中逐个写入。

**为什么需要 package 过滤？**

STM32MP257 有多种封装（LBGA448、LBGA292 等），不同封装引出的 GPIO 引脚数量不同：

| 封装 | 主域可用引脚数 | 安全域 |
|------|--------------|--------|
| LBGA448 (完整) | 176 (PA0~PK7) | 10 (GPIOZ) |
| LBGA292 (精简) | 约 120 | 10 |

如果在 LBGA292 封装上注册了 PK7 这个不存在的引脚，驱动请求它时 pinctrl core 不会拒绝（因为 pin_desc[175] 存在），但写寄存器时地址不对——PK7 的寄存器位段在实际硬件上不存在。

**`stm32_pctrl_create_pins_tab()` 核心逻辑：**

```c
// pinctrl-stm32.c:1989
static int stm32_pctrl_create_pins_tab(struct stm32_pinctrl *pctl,
                                        struct stm32_desc_pin *pins)
{
	const struct stm32_desc_pin *p;
	int i, nb_pins_available = 0;

	for (i = 0; i < pctl->match_data->npins; i++) {
		p = pctl->match_data->pins + i;
		if (pctl->pkg && !(pctl->pkg & p->pkg))
			continue;
		pins->pin = p->pin;
		memcpy((struct stm32_desc_pin *)pins->functions, p->functions,
		       STM32_CONFIG_NUM * sizeof(struct stm32_desc_function));
		pins++;
		nb_pins_available++;
	}

	pctl->npins = nb_pins_available;

	return 0;
}
```

执行完 `create_pins_tab()` 后，`pctl->pins` 数组中每个元素是 `struct stm32_desc_pin`，包含引脚编号/名称、alternate function 表、封装掩码。几个典型条目在内存中的实际值：

```
pctl->pins[0]                      pctl->pins[1]                      pctl->pins[110]
  .pin.number = 0                    .pin.number = 1                    .pin.number = 110
  .pin.name   = "PA0"               .pin.name   = "PA1"               .pin.name   = "PG14"
  .pkg = AI|AJ|AK|AL                .pkg = AI|AJ|AK|AL                 .pkg = AI|AJ|AK|AL
  .functions:                       .functions:                        .functions:
    [0]  = "GPIOA0"                   [0]  = "GPIOA1"                   [0]  = "GPIOG14"
    [2]  = "LPTIM1_CH2"               [3]  = "SPI6_MISO"                [1]  = "TRACED12"
    [3]  = "SPI5_RDY"                 [5]  = "SAI3_SD_A"                [2]  = "HDP4"
    [4]  = "UART8_CTS"                [6]  = "USART1_RTS"               [3]  = "SPI7_RDY"
    [5]  = "SAI2_MCLK_B"              [7]  = "USART6_CK"                [6]  = "MDF1_CKI5"
    [6]  = "UART5_TX"                 [8]  = "TIM4_CH2"                 [7]  = "USART1_TX"
    [7]  = "USART3_TX"                [9]  = "I2C4_SDA"                 [8]  = "(null)"
    [8]  = "TIM3_ETR"                 [10] = "I2C6_SDA"                 [9]  = "TIM8_BKIN2"
    [9]  = "TIM5_CH2"                 [11] = "(null)"                   [10] = "(null)"
    [10] = "(null)"                   [12] = "LCD_R3"                   [11] = "(null)"
    [11] = "ETH2_MII_RXD2..."         [13] = "(null)"                   [12] = "(null)"
    [12] = "(null)"                   [14] = "DCMI_D5..."               [13] = "(null)"
    [13] = "FMC_NL"                   [15] = "ETH3_PHY_INTN"            [14] = "LCD_B1"
    [14] = "(null)"                   [16] = "EVENTOUT"                 [15] = "DCMI_D9..."
    [15] = "DCMI_D9..."               [17] = "ANALOG"                   [16] = "EVENTOUT"
    [16] = "EVENTOUT"                                                        [17] = "ANALOG"
    [17] = "ANALOG"
```

未指定的 AF 编号在内存中为 `{ .num = 0, .name = NULL }`，图中标为 `(null)`。过滤结果：`match_data->npins = 198`（AI 完整封装）→ `pctl->npins = 176`（ATK 板 AJ 封装实际可用数）。被跳过的 22 个引脚在 debugfs 中不可见，consumer 设备也无法请求它们。

### 3.4.5 构建分组状态（一 pin 一组）

```c
// pinctrl-stm32.c:2066
ret = stm32_pctrl_build_state(pdev);
```

```c
// pinctrl-stm32.c:1958
static int stm32_pctrl_build_state(struct platform_device *pdev)
{
    struct stm32_pinctrl *pctl = platform_get_drvdata(pdev);
    int i;

    pctl->ngroups = pctl->npins;  // 组数 = 引脚数

    pctl->groups = devm_kcalloc(&pdev->dev, pctl->ngroups,
                                sizeof(*pctl->groups), GFP_KERNEL);
    if (!pctl->groups)
        return -ENOMEM;

    pctl->grp_names = devm_kcalloc(&pdev->dev, pctl->ngroups,
                                   sizeof(*pctl->grp_names), GFP_KERNEL);
    if (!pctl->grp_names)
        return -ENOMEM;

    for (i = 0; i < pctl->npins; i++) {
        const struct stm32_desc_pin *pin = pctl->pins + i;
        struct stm32_pinctrl_group *group = pctl->groups + i;

        group->name = pin->pin.name;       // 如 "PG14"
        group->pin  = pin->pin.number;      // 如 110
        pctl->grp_names[i] = pin->pin.name;
    }
    return 0;
}
```

`stm32_pctrl_build_state()` 执行后，`pctl` 中多了两个数组：

```
pctl->ngroups = 176                ← 等于过滤后的引脚数
pctl->groups  = stm32_pinctrl_group[176]
                ├── [0]   { .name = "PA0",  .pin = 0   }
                ├── [1]   { .name = "PA1",  .pin = 1   }
                ├── ...
                ├── [110] { .name = "PG14", .pin = 110 }
                └── [175] { .name = "PK7",  .pin = 175 }
pctl->grp_names = const char*[176]   ← 每个元素指向对应的 group name
                ├── [0]   = "PA0"
                ├── [110] = "PG14"
                └── [175] = "PK7"
```

这些数据供 pinctrl_ops 的枚举回调使用：

| 数据 | 类型 | 谁使用 |
|------|------|--------|
| `pctl->groups[i].pin` | `unsigned int` | `get_group_pins(selector)` 返回 pin 号 |
| `pctl->grp_names[i]` | `const char *` | `get_group_name(selector)` 返回组名 |

三个枚举回调的具体实现：

| 回调 | 返回 | 数据来源 |
|------|------|---------|
| `get_groups_count()` | `pctl->ngroups`（176） | 遍历次数 |
| `get_group_name(i)` | `pctl->grp_names[i]`（如 "PG14"） | 字符串匹配，DTS 中的 group 名 → 数字索引 |
| `get_group_pins(i)` | `&groups[i].pin` + `num_pins=1` | 返回 pin 号 |

**为什么 groups[i] 只有 "pin" 而没有 "config"？** 其他 SoC 驱动（如 pinctrl-imx）会在 group 中预配置电气参数。STM32 采用"一 pin 一组"简化策略，电气参数全部由 DTS 的 `pinctrl-0` 属性动态下发，不在驱动中预定义。

### 3.4.6 填充 pinctrl_desc 并注册

#### 3.4.6.1 填充 desc

```c
// pinctrl-stm32.c:2083-2104
pins = devm_kcalloc(&pdev->dev, pctl->npins, sizeof(*pins), GFP_KERNEL);
for (i = 0; i < pctl->npins; i++)
    pins[i] = pctl->pins[i].pin;    // 从 stm32_desc_pin 提取 pinctrl_pin_desc

pctl->pctl_desc.name           = dev_name(&pdev->dev);  // "44240000.pinctrl"
pctl->pctl_desc.owner          = THIS_MODULE;
pctl->pctl_desc.pins           = pins;                   // pinctrl_pin_desc[176]
pctl->pctl_desc.npins          = pctl->npins;            // 176
pctl->pctl_desc.link_consumers = true;
pctl->pctl_desc.confops        = &stm32_pconf_ops;
pctl->pctl_desc.pctlops        = &stm32_pctrl_ops;
pctl->pctl_desc.pmxops         = &stm32_pmx_ops;
pctl->pctl_desc.num_custom_params = ARRAY_SIZE(stm32_gpio_bindings);
pctl->pctl_desc.custom_params    = stm32_gpio_bindings;
```

| pinctrl_desc 字段 | 实际值 | 含义 |
|-------------------|--------|------|
| `.name` | `"44240000.pinctrl"` | debugfs 目录名、日志标识 |
| `.pins` | pinctrl_pin_desc[176] | 引脚描述表 |
| `.npins` | 176 | 可用引脚数 |
| `.pctlops` | `stm32_pctrl_ops` | 引脚枚举回调 |
| `.pmxops` | `stm32_pmx_ops` | 引脚复用回调 |
| `.confops` | `stm32_pconf_ops` | 引脚配置回调 |
| `.link_consumers` | `true` | 建立 supplier → consumer device link |
| `.custom_params` | `stm32_gpio_bindings` | 自定义 DTS 属性（st,io-retime 等）|

填充后，`pctl->pctl_desc` 的内容如下：

```
pctl_desc.name           = "44240000.pinctrl"        ← dev_name(&pdev->dev)
pctl_desc.pins           = pinctrl_pin_desc[176]     ← 从 stm32_desc_pin[].pin 提取
                          ├── { .number=0,  .name="PA0"  }
                          ├── { .number=110,.name="PG14" }
                          └── { .number=175,.name="PK7"  }
pctl_desc.npins          = 176                       ← ATK 板 AJ 封装可用引脚数
pctl_desc.link_consumers = true                      ← 启用 supplier-consumer device link
pctl_desc.pctlops        = &stm32_pctrl_ops          ← 引脚枚举 + DTS map
pctl_desc.pmxops         = &stm32_pmx_ops            ← 引脚复用
pctl_desc.confops        = &stm32_pconf_ops          ← 引脚配置
pctl_desc.custom_params  = stm32_gpio_bindings       ← st,io-retime 等厂商私有属性
```

注意 `pctl_desc.pins[]` 只包含 `{number, name}` 两个字段——pinctrl core 层用不到 alternate function 表（`stm32_desc_pin.functions[]`），那些数据留在 `pctl->pins[i].functions` 中，仅在驱动内部 `set_mux()` 回调时才会被查找。

#### 3.4.6.2 `devm_pinctrl_register()` — 提交给 core 层

```c
pctl->pctl_dev = devm_pinctrl_register(&pdev->dev, &pctl->pctl_desc, pctl);
```

`devm_pinctrl_register()` 是资源管理版本——设备 remove 时自动 unregister。

```
devm_pinctrl_register()
  └─ pinctrl_register()
       ├── pinctrl_init_controller()
       │     ├── 分配 pinctrl_dev
       │     ├── 初始化关键字段
       │     ├── 检查 ops 完整性
       │     └── pinctrl_register_pins()     ← 创建 pin_desc[]
       └── pinctrl_enable()
             ├── pinctrl_claim_hogs()        ← 处理 hog 引脚
             ├── 挂入 pinctrldev_list 链表    ← 全局可见
             └── pinctrl_init_device_debugfs() ← 创建 debugfs
```

##### 3.4.6.2.1 `pinctrl_init_controller()` 内部

`devm_pinctrl_register()` → `pinctrl_register()` 的第一步：分配 `pinctrl_dev` 实例、初始化内部数据结构、检查 ops 完整性、创建 `pin_desc[]` 数组。

```c
// core.c:2008
static struct pinctrl_dev *pinctrl_init_controller(
	struct pinctrl_desc *pctldesc, struct device *dev, void *driver_data)
{
	struct pinctrl_dev *pctldev;
	int ret;

	// ── 阶段 A：参数校验 ──
	if (!pctldesc)
		return ERR_PTR(-EINVAL);
	if (!pctldesc->name)
		return ERR_PTR(-EINVAL);

	// 分配 pinctrl_dev 实例（driver_data 指向 stm32_pinctrl）
	pctldev = kzalloc(sizeof(*pctldev), GFP_KERNEL);
	if (!pctldev)
		return ERR_PTR(-ENOMEM);

	// ── 阶段 B：初始化内部数据结构 ──
	pctldev->owner = pctldesc->owner;
	pctldev->desc = pctldesc;                               // 指向 pinctrl_desc
	pctldev->driver_data = driver_data;                      // → stm32_pinctrl
	INIT_RADIX_TREE(&pctldev->pin_desc_tree, GFP_KERNEL);    // pin_desc 索引（key=pin号）
	INIT_LIST_HEAD(&pctldev->gpio_ranges);                   // pinctrl_gpio_range 链表（桥梁）
	INIT_LIST_HEAD(&pctldev->node);                          // 挂入 pinctrldev_list 的节点
	pctldev->dev = dev;
	mutex_init(&pctldev->mutex);                             // 保护 pinctrl_dev 并发访问

	// ── 阶段 C：ops 完整性检查 + 引脚注册 ──
	// pinctrl_check_ops：检查 pctlops 是否提供（至少需要 dt_node_to_map）
	ret = pinctrl_check_ops(pctldev);
	if (ret)
		goto out_err;

	// pinmux_check_ops / pinconf_check_ops：只有驱动实现了对应 ops 才检查
	// 如果实现了 pmxops，必须提供 set_mux（否则无法配置引脚复用）
	if (pctldesc->pmxops) {
		ret = pinmux_check_ops(pctldev);
		if (ret)
			goto out_err;
	}
	// 如果实现了 confops，必须提供 pin_config_set（否则无法配置电气参数）
	if (pctldesc->confops) {
		ret = pinconf_check_ops(pctldev);
		if (ret)
			goto out_err;
	}

	// ★ 注册所有引脚：遍历 pinctrl_desc.pins[0..npins-1]，
	//    为每个调用 pinctrl_register_one_pin() → 创建 pin_desc
	ret = pinctrl_register_pins(pctldev, pctldesc->pins, pctldesc->npins);
	if (ret) {
		pinctrl_free_pindescs(pctldev, pctldesc->pins, pctldesc->npins);
		goto out_err;
	}

	return pctldev;

out_err:
	mutex_destroy(&pctldev->mutex);
	kfree(pctldev);
	return ERR_PTR(ret);
}
```

`pinctrl_register_pins()` 是遍历函数，对每个引脚调用 `pinctrl_register_one_pin()`：

```c
// core.c:254
static int pinctrl_register_pins(struct pinctrl_dev *pctldev,
				 const struct pinctrl_pin_desc *pins,
				 unsigned num_descs)
{
	unsigned i;
	int ret = 0;

	for (i = 0; i < num_descs; i++) {
		ret = pinctrl_register_one_pin(pctldev, &pins[i]);
		if (ret)
			return ret;
	}
	return 0;
}
```

这里有一个关键区别需要理解：`pinctrl_register_pins()` 的参数 `pins` 指向的是驱动提供的 **`pinctrl_pin_desc`**（轻量级结构，只含 `number`、`name`、`drv_data` 三个字段），而它创建的是 **`pin_desc`**（core 层的运行时结构，除了 identity 外还包含 `mux_usecount`、`mux_owner`、`gpio_owner`、`mux_setting` 等所有权跟踪字段）。前者是静态注册凭证，后者是运行时身份证——`pin_desc` 从 `pinctrl_pin_desc` 读取 identity 信息，再附加上自己的运行时字段。

真正的注册逻辑在 `pinctrl_register_one_pin()` 中——为单个引脚分配 `pin_desc`、填入身份信息、以 pin 号为 key 插入 radix tree：

```c
// core.c:204
static int pinctrl_register_one_pin(struct pinctrl_dev *pctldev,
                                    const struct pinctrl_pin_desc *pin)
{
    struct pin_desc *pindesc;

    // ① 查重：pin 号不能重复注册
    //    如果重复注册，说明驱动有 bug（如 pinctrl_desc.pins[] 有重复编号）
    pindesc = pin_desc_get(pctldev, pin->number);
    if (pindesc) {
        dev_err(pctldev->dev, "pin %d already registered\n", pin->number);
        return -EINVAL;
    }

    // ② 分配 pin_desc
    pindesc = kzalloc(sizeof(*pindesc), GFP_KERNEL);

    // ③ 设置基本信息
    pindesc->pctldev = pctldev;
    pindesc->name    = pin->name;           // "PG14"（来自 SoC 写死的引脚表）
    pindesc->drv_data = pin->drv_data;      // 驱动私有数据
    mutex_init(&pindesc->mux_lock);

    // ④ ★ 以 pin 号为 key 插入 radix tree
    radix_tree_insert(&pctldev->pin_desc_tree, pin->number, pindesc);
    //  key = pin 号 ↑

    return 0;
}
```

注册完成后，`pinctrl_dev` 内部的红黑树：

```
pinctrl_dev->pin_desc_tree（radix tree，key = pin 号）
  ├── key=0   → pin_desc{name="PA0",   mux_usecount=0, mux_owner=NULL, gpio_owner=NULL}
  ├── key=1   → pin_desc{name="PA1",   mux_usecount=0, mux_owner=NULL, gpio_owner=NULL}
  ├── ...
  ├── key=110 → pin_desc{name="PG14",  mux_usecount=0, mux_owner=NULL, gpio_owner=NULL}
  └── key=175 → pin_desc{name="PK7",   mux_usecount=0, mux_owner=NULL, gpio_owner=NULL}
```

**所有 `mux_usecount`、`owner` 字段此时均为 0/NULL**——pin_desc 在注册时只建立了"身份信息"，不记录"谁在用"。这些字段在后续 `pin_request()` / `pin_free()` 被 consumer 设备调用时才被修改。

##### 3.4.6.2.2 `pinctrl_enable()` 内部

`pinctrl_init_controller()` 完成后调用 `pinctrl_enable()`，完成三件事：处理 pinctrl 节点自身的 hog 引脚（通常为空）、将 `pinctrl_dev` 加入全局链表使其对 consumer 可见、创建 debugfs 调试接口。

```c
// core.c:2124
int pinctrl_enable(struct pinctrl_dev *pctldev)
{
    // ① 处理 hog 引脚（自身 DTS 中的 pinctrl-0 状态）
    //    如果 pinctrl 节点有 pinctrl-0 属性，这里应用它
    pinctrl_claim_hogs(pctldev);

    // ② 挂入 pinctrldev_list 全局链表
    //    consumer 设备通过 for_each_pinctrl() 遍历此链表查找对应的 pinctrl_dev
    list_add_tail(&pctldev->node, &pinctrldev_list);

    // ③ 创建 debugfs 目录
    //    在 /sys/kernel/debug/pinctrl/ 下创建 44240000.pinctrl/
    pinctrl_init_device_debugfs(pctldev);

    return 0;
}
```

**`pinctrl_claim_hogs()`** — 处理 pin controller 自身节点声明的 `pinctrl-0` 引脚。以假设的调试场景为例：

```dts
pinctrl: pinctrl@44240000 {
    compatible = "st,stm32mp257-pinctrl";
    pinctrl-0 = <&debug_pins>;
    pinctrl-names = "default";

    debug_pins: debug-pins {
        pins {
            pinmux = <STM32_PINMUX('G', 14, AF6)>;  /* USART1_TX */
            bias-disable;
        };
    };
};
```

如果 DTS 中定义了如上 `pinctrl-0`，`pinctrl_claim_hogs()` 的代码执行逻辑如下：

```c
// core.c:2086
static int pinctrl_claim_hogs(struct pinctrl_dev *pctldev)
{
    // 为 pin controller 自身设备创建一个 pinctrl 句柄
    // pctldev->dev 就是 platform_device 的 struct device
    pctldev->p = create_pinctrl(pctldev->dev, pctldev);
    //  ↑ 解析 pinctrl-0 → 生成 pinctrl_map[2]:
    //    map[0]: MUX_GROUP,  function="af6",  group="PG14"
    //    map[1]: CONFIGS_GROUP,  group="PG14",  configs={BIAS_DISABLE}

    // 查找并应用 "default" 状态
    pctldev->hog_default = pinctrl_lookup_state(pctldev->p, "default");
    if (!IS_ERR(pctldev->hog_default))
        pinctrl_select_state(pctldev->p, pctldev->hog_default);
    //  ↑ 执行 set_mux → 写 MODER/PUPDR/OTYPER
    //    PG14 被配置为 USART1_TX（AF6），无上下拉

    // 尝试 sleep 状态（可选）
    pctldev->hog_sleep = pinctrl_lookup_state(pctldev->p, "sleep");
    ...
}
```

应用后 PG14 的寄存器已被硬件配置好，但 `pin_desc->mux_owner` 依然为 NULL——hog 机制只做硬件配置，不走 `pin_request()` 所有权注册。

**ATK 板不触发此路径。** pinctrl 节点没有声明 `pinctrl-0`，`create_pinctrl()` 返回 `-ENODEV`，`pinctrl_claim_hogs()` 直接 return 0。这个机制的用途是：调试 UART 或 JTAG 引脚可以通过 hog 在 pinctrl probe 时立即配置，无需等待具体驱动 probe。

**`list_add_tail()`** — 将 pinctrl_dev 加入全局链表 `pinctrldev_list`：

```c
// core.c:48-52
static DEFINE_MUTEX(pinctrldev_list_mutex);
static LIST_HEAD(pinctrldev_list);
```

执行后链表只有一个节点：

```
pinctrldev_list → pinctrl_dev{ desc.name="44240000.pinctrl" }
```

这个链表的用途是让 consumer 设备能通过设备名或 of_node 找到对应的 pinctrl_dev。以 USART1 为例，它 probe 时通过 `create_pinctrl()` 来完成两件事：

1. **解析 DTS 生成 map** — 将 DTS 中 `pinmux = <STM32_PINMUX('G', 14, AF6)>` 转为 `pinctrl_map[]`。这一步需要找到 pinctrl_dev，因为它要调用 `pinctrl_ops.dt_node_to_map()` 回调。
2. **将 map 转为 setting** — 把 map 写入 `pinctrl_setting`，供后续 `pinctrl_select_state()` 写寄存器。这一步也需要 pinctrl_dev，因为 setting 要指向具体的 pinctrl_dev。

两件事都需要 pinctrl_dev，但查找方式不同：

| 步骤 | 查找方式 | 为什么不同 |
|------|---------|-----------|
| ① 解析 DTS | 用 of_node 指针匹配（`device_match_of_node`）| 从 DTS 子节点上溯找到 pin controller 节点，直接比对 device_node 指针 |
| ② 生成 setting | 用设备名字符串匹配（`strcmp(dev_name)`）| map 中保存的是 `ctrl_dev_name` 字符串，以它为 key 遍历链表 |

`create_pinctrl()` 内部的完整调用流程如下（两次遍历用 ★ 标注）：

```
create_pinctrl(dev=usart1_dev, pctldev=NULL)
  │
  │  ── 第一阶段：解析 DTS 生成 map ──
  │
  ├─ pinctrl_dt_to_map(p, NULL)
  │    │
  │    └─ dt_to_map_one_config(p, NULL, "default", np_config)
  │         │                          ↑ pins1 节点
  │         │
  │         │ ★ for 循环：np_pctldev = of_get_next_parent(np_pctldev)
  │         │     每上溯一级拿到的 device_node 指针都传给
  │         │     get_pinctrl_dev_from_of_node()，遍历
  │         │     pinctrldev_list 逐项比对 dev->of_node 指针
  │         │     （比指针地址，不是比字符串）
  │         │     上溯路径：
  │         │       usart1_pins_a     → 不匹配 → 继续
  │         │       pinctrl@44240000  → 匹配！→ 返回
  │         │
  │         ├─ ops->dt_node_to_map(pctldev, pins1, &map, &num_maps)
  │         │     → 驱动回调解析 pinmux，生成 map[0]=MUX, map[1]=CONFIGS
  │         │
  │         └─ dt_remember_or_free_map(p, "default", pctldev, map, 2)
  │               → map[i].dev_name = "usart1"
  │               → map[i].ctrl_dev_name = "44240000.pinctrl"
  │               → pinctrl_register_mappings()
  │                 → 分配 struct pinctrl_maps{node, maps→map[], num_maps=2}
  │                 → list_add_tail() → 挂入全局链表 pinctrl_maps
  │
  │  ── 第二阶段：遍历 map 生成 setting ──
  │
  ├─ for_each_pin_map(maps_node, map)        ← 遍历全局 pinctrl_maps
  │    │  里面包含所有设备的 map（USART1、ETH、MMC 等）
  │    │  通过 strcmp(map->dev_name, "usart1") 筛选
  │    │  只处理属于本设备的 map 条目
  │    │
  │    └─ add_setting(p, pctldev=NULL, map)
  │         │
  │         ├─ ★ get_pinctrl_dev_from_devname("44240000.pinctrl")
  │         │     → 遍历 pinctrldev_list                    第二次遍历
  │         │     → 匹配 dev_name → 返回 pinctrl_dev
  │         │
  │         └─ 分配 setting，填入数据：
  │            setting->type = PIN_MAP_TYPE_MUX_GROUP
  │            setting->pctldev = pinctrl_dev
  │            setting->data.mux.function = "af6"
  │            setting->data.mux.group = 110
  │            → list_add_tail(&setting->node, &state->settings)
  │
  └─ pinctrl_select_state(p, "default")
        → 遍历 p->states 链表 → 找到名为 "default" 的 state
        → 遍历 state->settings 链表
        → 对每个 setting 调对应回调：
          setting(MUX_GROUP)     → set_mux()  写 MODER
          setting(CONFIGS_GROUP) → pinconf_set() 写 PUPDR/OTYPER
        → 完成硬件配置

三者之间的数据结构关系：
  struct pinctrl(p)
  └── states 链表头
       └── state("default")
            └── settings 链表头
                 ├── setting{type=MUX_GROUP,     pctldev=..., data.mux.function="af6"}
                 └── setting{type=CONFIGS_GROUP, pctldev=..., data.configs={...}}
```

以下面 USART1 的 DTS 为例，展开这两步：

```dts
&usart1 {
    pinctrl-0 = <&usart1_pins_a>;
};

usart1_pins_a: usart1-0 {
    pins1 {
        pinmux = <STM32_PINMUX('G', 14, AF6)>;  /* USART1_TX on PG14 */
        bias-disable;
        drive-push-pull;
    };
};
```

**① 解析 DTS（按 of_node 查找）**— 进入 `create_pinctrl()` → `pinctrl_dt_to_map()`，需要找到 pinctrl_dev 来调用 `dt_node_to_map()` 回调。但 `pinmux` 属性写在 `pins1` 子节点里，`pins1` 不是 pin controller，需要沿 parent 链上溯：

```
pins1                         ← np_config 的起始节点
  parent → usart1_pins_a      ← 不是 pin controller，继续
             parent → pinctrl@44240000  ← 是 pin controller！
```

  找到后得到该 pin controller 的 `pinctrl_dev`。接下来两步将 DTS 属性转换成内核内部的 `pinctrl_map`：
  1. **`ops->dt_node_to_map()`** — 回调驱动解析 `pinmux` 属性，生成 `pinctrl_map[]`：
      ```
      map[0]: type=MUX_GROUP,      data: {group="PG14", function="af6"}
      map[1]: type=CONFIGS_GROUP,  data: {configs={BIAS_DISABLE, PUSH_PULL}}
      ```
  2. **`dt_remember_or_free_map()`** — 遍历所有 map 条目，填入 `dev_name` 和 `ctrl_dev_name`，然后注册到全局链表：
     ```c
     for (i = 0; i < num_maps; i++) {
         map[i].dev_name = "usart1";
         map[i].ctrl_dev_name = "44240000.pinctrl";
     }
     pinctrl_register_mappings(map, num_maps);
     ```

**② 生成 setting（按设备名查找）** — 回到 `create_pinctrl()`，此时 map 已注册到 `pinctrl_maps` 全局链表。遍历该链表找到属于 USART1 的 map，调 `add_setting()`：

```c
// core.c:create_pinctrl() → 遍历 pinctrl_maps → add_setting()
// map->ctrl_dev_name 就是从第一次遍历中写入的 "44240000.pinctrl"
setting->pctldev = get_pinctrl_dev_from_devname(map->ctrl_dev_name);
```

```c
// core.c:100
struct pinctrl_dev *get_pinctrl_dev_from_devname(const char *devname)
{
    list_for_each_entry(pctldev, &pinctrldev_list, node)
        if (!strcmp(dev_name(pctldev->dev), devname))
            // strcmp(dev_name(pctldev->dev), "44240000.pinctrl") == 0 ?
            // 是 → 返回这个 pinctrl_dev
            return pctldev;
    return NULL;
}
```

找到后将 pin controller 存入 setting，后续 `pinctrl_select_state()` 即可通过 `setting->pctldev` 直接操作硬件。

**`pinctrl_init_device_debugfs()`** — 创建 debugfs 目录：

```c
// core.c:1912
static void pinctrl_init_device_debugfs(struct pinctrl_dev *pctldev)
{
    // 目录名 = pinctrl_desc.name = "44240000.pinctrl"
    pctldev->device_dir = debugfs_create_dir(dev_name(pctldev->dev),
                                              debugfs_root);
    // 子文件
    debugfs_create_file("pinctrl-devices", ...);
    debugfs_create_file("pinmux-pins", ...);
    debugfs_create_file("pinmux-select", ...);
    debugfs_create_file("pinconf-pins", ...);
    debugfs_create_file("pinconf-groups", ...);
}
```

创建后：

```
/sys/kernel/debug/pinctrl/
  └── 44240000.pinctrl/
       ├── pinctrl-devices    ← 该 controller 管理的设备列表
       ├── pinmux-pins        ← 每个引脚的复用状态
       ├── pinmux-select      ← 手动切状态（写操作）
       ├── pinconf-pins       ← 每个引脚的配置参数
       └── pinconf-groups     ← 每组引脚的配置参数
```

创建后的 debugfs 目录结构：

```
/sys/kernel/debug/pinctrl/
  └── 44240000.pinctrl/            ← 目录名 = pinctrl_desc.name
       ├── pinctrl-devices         ← 该 controller 管理的设备列表
       ├── pinmux-pins             ← 每个引脚的复用状态（最常用）
       ├── pinmux-select           ← 手动切状态（写操作）
       ├── pinconf-pins            ← 每个引脚的配置参数
       └── pinconf-groups          ← 每组引脚的配置参数
```

此时 `cat /sys/kernel/debug/pinctrl/44240000.pinctrl/pinmux-pins` 的输出：

```
Pin 0 (PA0):  (MUX UNCLAIMED) (GPIO UNCLAIMED)
Pin 1 (PA1):  (MUX UNCLAIMED) (GPIO UNCLAIMED)
...
Pin 110 (PG14):  (MUX UNCLAIMED) (GPIO UNCLAIMED)
...
Pin 175 (PK7):  (MUX UNCLAIMED) (GPIO UNCLAIMED)
```

所有引脚 `UNCLAIMED`——pin_desc 注册时只建立了"身份信息"，`mux_owner` 和 `gpio_owner` 全部为 NULL。这些字段将在后续 consumer 设备 probe 时通过 `pin_request()` 被写入。注意安全域 pinctrl_z 此时还未 probe，所以 debugfs 中只有 `44240000.pinctrl/` 这一个目录。

### 3.4.7 pinctrl_ops 回调实现

`stm32_pctrl_ops` 是 pinctrl core 在注册时和运行时调用的入口，定义如下：

```c
static const struct pinctrl_ops stm32_pctrl_ops = {
    .dt_node_to_map     = stm32_pctrl_dt_node_to_map,
    .dt_free_map        = pinctrl_utils_free_map,
    .get_groups_count   = stm32_pctrl_get_groups_count,
    .get_group_name     = stm32_pctrl_get_group_name,
    .get_group_pins     = stm32_pctrl_get_group_pins,
};
```

#### 3.4.7.1 引脚枚举回调 — 注册时调用

三个枚举回调在 `pinctrl_register()` 内部由 core 层调用，用于建立 `pinctrl_dev->groups[]` 数组：

```c
static int stm32_pctrl_get_groups_count(struct pinctrl_dev *pctldev)
{
    struct stm32_pinctrl *pctl = pinctrl_dev_get_drvdata(pctldev);
    return pctl->ngroups;       // 176
}

static const char *stm32_pctrl_get_group_name(struct pinctrl_dev *pctldev,
                                              unsigned group)
{
    struct stm32_pinctrl *pctl = pinctrl_dev_get_drvdata(pctldev);
    return pctl->groups[group].name;    // "PA0", "PG14", ...
}

static int stm32_pctrl_get_group_pins(struct pinctrl_dev *pctldev,
                                      unsigned group,
                                      const unsigned **pins,
                                      unsigned *num_pins)
{
    struct stm32_pinctrl *pctl = pinctrl_dev_get_drvdata(pctldev);
    *pins = (unsigned *)&pctl->groups[group].pin;
    *num_pins = 1;                      // STM32 一 pin 一组
    return 0;
}
```

核心层注册时调用这三个回调：

```
pinctrl_register()
  → pinctrl_init_controller()
    → pinctrl_register_pins()        // 创建 pin_desc[]
    → (此时不调 pinctrl_ops)
  → pinctrl_enable()
    → (也不调 pinctrl_ops 枚举回调)
```

**关键点：** 这三个枚举回调**不是在 pinctrl_register() 内部调用的**——它们只在 consumer 设备通过 `pinctrl_get()` / `pinctrl_select_state()` 时需要查找 group 时才被调用。core 层在注册时只通过 `pinctrl_desc.pins[]` 创建 `pin_desc[]`，不通过 `pctlops` 枚举 group。这也解释了为什么 pinctrl core 不强制 `pctlops` 提供这些回调——如果 SoC 驱动不需要 consumer 查找 group，可以不实现。

但实际上 STM32 实现了它们，因为 consumer 设备需要"字符串名 → 数字索引"的转换。以 USART1 为例，其 DTS 中声明 `pinctrl-0 = <&usart1_pins_a>`，usart1_pins_a 的子节点通过 `pinmux = <STM32_PINMUX('G', 14, AF6)>` 指定了引脚。DTS 解析后得到 group 名字符串 `"PG14"`。当 core 层需要查找 PG14 对应的 group 索引时，调用链如下：

```
pinctrl_select_state("default")
  → 需要将 DTS 中的 group 名 "PG14" 转为数字索引
    → get_group_count()           → 返回 176
    → for (i=0; i<176; i++)
        → get_group_name(i)
          → 遍历 pctl->grp_names[]
            → i=0  返回 "PA0"  → 不匹配
            → i=1  返回 "PA1"  → 不匹配
            → ...
            → i=110 返回 "PG14" → 匹配！
    → 找到 group index = 110
```

这就是枚举回调的唯一用途——将 DTS 中的字符串名解析为数字索引，然后通过 `get_group_pins(110)` 获取该 group 对应的 pin 号（110），最终在 `set_mux()` 中操作 PG14 的硬件寄存器。

#### 3.4.7.2 `dt_node_to_map` — DTS 解析入口

这是最重要的 pinctrl_ops 回调——它将 DTS 的 `pinctrl-0` 节点解析为 `pinctrl_map[]` 数组。调用时机不是 probe 阶段，而是 consumer 设备 probe 时 `pinctrl_dt_to_map()` 调用的：

```c
// consumer 设备 probe 顺序：
driver_probe_device()
  → pinctrl_bind_pins(dev)                // ← 这是 04 情景分析的重点
    → pinctrl_dt_to_map(dev)              // 解析 DTS
      → pinctrl_ops.dt_node_to_map()      // ← 这里调用
```

**`stm32_pctrl_dt_node_to_map()` 的工作流程：**

```c
static int stm32_pctrl_dt_node_to_map(struct pinctrl_dev *pctldev,
                     struct device_node *np_config,
                     struct pinctrl_map **map, unsigned *num_maps)
{
    // 遍历 pinctrl-x 子节点（如 usart1_pins_a 下的 pins1, pins2）
    for_each_child_of_node(np_config, np) {
        stm32_pctrl_dt_subnode_to_map(pctldev, np, map, ...);
    }
}
```

**`stm32_pctrl_dt_subnode_to_map()` 对每个子节点做三件事（`pinctrl-stm32.c:852`）：**

```
① 读 "pinmux" 属性（必须）
   pins = of_find_property(node, "pinmux", NULL);
   每个 STM32_PINMUX('G', 14, AF6) 编码为 u32: (pin << 16) | func

② 读电气参数（可选）
   pinconf_generic_parse_dt_config(node, pctldev, &configs, &num_configs);
   解析 bias-disable、slew-rate、drive-push-pull 等属性

③ 对每个 pinmux 值，生成 pinctrl_map 条目
   for (i = 0; i < num_pins; i++) {
       pin   = STM32_GET_PIN_NO(pinfunc);   // 提取 pin 号
       func  = STM32_GET_PIN_FUNC(pinfunc);  // 提取 AF 编号
       grp   = stm32_pctrl_find_group_by_pin(pctl, pin);  // pin → group 名
       
       // 生成 MUX 条目
       map[].type = PIN_MAP_TYPE_MUX_GROUP;
       map[].data.mux.group    = grp->name;          // "PG14"
       map[].data.mux.function = stm32_gpio_functions[func];  // "af6"
       
       // 如果有电气参数，生成 CONFIGS 条目
       map[].type = PIN_MAP_TYPE_CONFIGS_GROUP;
       map[].data.configs.configs = configs;         // {BIAS_DISABLE, ...}
   }
```

以 USART1 的 DTS 配置为例：

```dts
&usart1 {
    pinctrl-0 = <&usart1_pins_a>;
};

usart1_pins_a: usart1-0 {
    pins1 {
        pinmux = <STM32_PINMUX('G', 14, AF6)>;  /* TX */
        bias-disable;
        drive-push-pull;
    };
};
```

解析后生成 2 个 `pinctrl_map`：

```
map[0]: type=MUX_GROUP,  function="af6",  group="PG14"
map[1]: type=CONFIGS_GROUP,  group="PG14",  configs={BIAS_DISABLE, PUSH_PULL}
```

如果 USART1 还有 RX 引脚（PG15）：

```
map[2]: type=MUX_GROUP,  function="af6",  group="PG15"
map[3]: type=CONFIGS_GROUP,  group="PG15",  configs={BIAS_DISABLE}
```

这些 map 最终被挂入 `pinctrl_maps` 全局链表，供后续 `pinctrl_select_state()` 使用。

### 3.4.8 准备 GPIO bank 资源

#### 3.4.8.1 枚举子节点数量

```c
// pinctrl-stm32.c:2111
banks = gpiochip_node_count(dev);
if (!banks) {
    dev_err(dev, "at least one GPIO bank is required\n");
    return -EINVAL;
}
pctl->banks = devm_kcalloc(dev, banks, sizeof(*pctl->banks), GFP_KERNEL);
pctl->clks  = devm_kcalloc(dev, banks, sizeof(*pctl->clks), GFP_KERNEL);
```

`gpiochip_node_count()` 遍历 `dev` 的所有子 fwnode 节点，统计声明了 `gpio-controller` 属性的节点数。其内部通过 `for_each_gpiochip_node` 宏实现：每遇到一个包含 `gpio-controller;` 的子节点就计数 +1（不是靠 compatible 字符串前缀匹配）。ATK 板上主域共 9 个 bank：

```dts
pinctrl@44240000 {
    gpioa: gpio@44240000 { gpio-controller; ... };  // 1
    gpiob: gpio@44250000 { gpio-controller; ... };  // 2
    gpioc: gpio@44260000 { gpio-controller; ... };  // 3
    gpiod: gpio@44270000 { gpio-controller; ... };  // 4
    gpioe: gpio@44280000 { gpio-controller; ... };  // 5
    gpiof: gpio@44290000 { gpio-controller; ... };  // 6
    gpiog: gpio@442a0000 { gpio-controller; ... };  // 7
    gpioh: gpio@442b0000 { gpio-controller; ... };  // 8
    gpioi: gpio@442c0000 { gpio-controller; ... };  // 9
};
```

执行后，`pctl` 中多出以下字段：

```
pctl->banks = stm32_gpio_bank[9]    ← 9 个 bank 的运行时数据
              ├── [0] = GPIOA 的 bank 数据（后续逐个填入）
              ├── [1] = GPIOB 的 bank 数据
              ├── ...
              └── [8] = GPIOI 的 bank 数据
pctl->clks  = clk_bulk_data[9]      ← 每个 bank 一个时钟句柄
```

`clk_bulk_data` 是 clock 框架提供的批量操作结构体，把"名称 → 时钟句柄"打包成数组，供 `clk_bulk_prepare_enable()` 等批量 API 统一操作。结构体定义如下（`include/linux/clk.h:87`）：

```c
struct clk_bulk_data {
    const char *id;    // 标识名（调试用途）
    struct clk *clk;   // 时钟句柄，指向 struct clk 封装体
};
```

`struct clk` 是对 clock core 的轻量封装（`drivers/clk/clk.c:102`）：

```c
struct clk {
    struct clk_core *core;  // 指向真正的时钟控制逻辑
    struct device *dev;
    const char *dev_id;
    const char *con_id;
    ...
};
```

驱动中每个 bank 对应 `clks[]` 中的一个元素，由以下代码填充：

```c
// pinctrl-stm32.c:2139
pctl->clks[i].clk = of_clk_get_by_name(np, NULL);  // 取 DTS clocks 属性第 0 个
pctl->clks[i].id = "pctl";                           // 所有 bank 固定写死为 "pctl"
```

注意两点：
1. **`id` 全是 `"pctl"`**——不是各 bank 独立的时钟名，只是批量操作的统一标识
2. **`clk` 指针指向 `struct clk`**——它是 clock 框架的封装对象，内部 `core` 字段指向硬件时钟控制逻辑

DTS 中每个 bank 的 `clocks` 属性引用的是 **SCMI 时钟**（`stm32mp251.dtsi`）：

| 元素 | bank | DTS clocks 属性 | clk 句柄 | clock core 对应的硬件门控 |
|------|------|----------------|----------|--------------------------|
| `clks[0]` | GPIOA | `&scmi_clk CK_SCMI_GPIOA` | `struct clk *` | `RCC_GPIOACFGR` bit 1 (GPIOAEN) |
| `clks[1]` | GPIOB | `&scmi_clk CK_SCMI_GPIOB` | `struct clk *` | `RCC_GPIOBCFGR` bit 1 (GPIOBEN) |
| ... | ... | ... | ... | ... |
| `clks[8]` | GPIOI | `&scmi_clk CK_SCMI_GPIOI` | `struct clk *` | `RCC_GPIOICFGR` bit 1 (GPIOIEN) |

硬件本质（`drivers/clk/stm32/clk-stm32mp25.c:2936`）：

```
ck_icn_ls_mcu  (ICN LS MCU 总线时钟)
    │
    ├── RCC_GPIOACFGR[1] → GPIOA PCLK
    ├── RCC_GPIOBCFGR[1] → GPIOB PCLK
    └── ...
```

每个 GPIO bank 的 **PCLK（外设总线时钟）** 在 RCC 中有一个独立的门控寄存器（`GPIOxCFGR`），bit 1 为使能位。但这个时钟的使能/关闭并非由内核直接操作 RCC，而是通过 **SCMI 协议** 委托给 OP-TEE 安全固件——DTS 中 `&scmi_clk` 即此通道。驱动调用 `clk_prepare_enable()` 时，内部通过 SCMI 消息让 OP-TEE 置位对应寄存器位。

**为什么需要这个时钟？** GPIO 的寄存器（MODER、ODR、IDR、BSRR 等）都在 PCLK 域下，读写前必须使能该 bank 的 PCLK。后面 `stm32_gpiolib_register_bank()` 注册每个 bank 时，会通过 `clk_bulk_prepare_enable(pctl->nbanks, pctl->clks)` 一次性使能所有 bank 的时钟。

注意 GPIOZ 不在这里。GPIOZ 的初始化流程与主域 GPIOA~I **框架相同**（同属 `stm32_pctl_probe()`，同样 6 个阶段），但有两个根本性差异：

1. **两次 probe，输入数据不同**——GPIOZ 位于 `pinctrl_z@46200000` 节点（安全岛），compatible 为 `"st,stm32mp257-z-pinctrl"`，与主域 `"st,stm32mp257-pinctrl"` 是两个不同的 DTS 节点。platform bus 匹配两次，`stm32_pctl_probe()` 被调用两次，分别持有 `stm32mp257_match_data`（主域，9 bank）和 `stm32mp257_z_match_data`（安全域，仅 GPIOZ 1 个 bank，10 个引脚 PZ0~PZ9）。

2. **Runtime 安全过滤效果不同**——两边虽然 `secure_control = true` 相同，但效果取决于不同物理地址上的寄存器值。驱动在 `gpiochip_init_valid_mask()` 回调中读取硬件状态做两层过滤：

```c
// pinctrl-stm32.c:460 — gpiochip_init_valid_mask 回调
bitmap_fill(valid_mask, ngpios);

if (bank->secure_control) {
    /* 第一层：SECCFGR 标记的安全引脚 */
    sec = readl_relaxed(bank->base + STM32_GPIO_SECCFGR);
    // 对主域 GPIOA：读 0x44240030
    // 对安全域 GPIOZ：读 0x46200030  ← 物理地址不同！
    for (i = 0; i < ngpios; i++) {
        if (sec & BIT(i)) {
            clear_bit(i, valid_mask);  // 标记为不可用
            dev_dbg(pctl->dev, "No access to gpio %d - %d\n",
                    bank->bank_nr, i);
        }
    }
}

if (bank->rif_control) {
    /* 第二层：RIF 防火墙 — CIDCFGR/SEMCR 寄存器 */
    for (i = 0; i < ngpios; i++) {
        if (!test_bit(i, valid_mask))
            continue;
        if (stm32_gpio_rif_valid(bank, i))
            continue;          // 当前 cid 有权访问 → 保留
        dev_dbg(pctl->dev, "RIF semaphore ownership conflict, GPIO %u", i);
        clear_bit(i, valid_mask);
    }
}
```

`stm32_gpio_rif_valid()` 内部的 CIDCFGR 寄存器逻辑（`pinctrl-stm32.c:265`）：

```c
static bool stm32_gpio_rif_valid(struct stm32_gpio_bank *bank, unsigned int gpio_nr)
{
    cid = readl_relaxed(bank->base + STM32_GPIO_CIDCFGR(gpio_nr));

    if (!(cid & STM32_GPIO_CIDCFGR_CFEN))
        return true;              // CFEN=0：防火墙关闭，谁都能访问

    if (!(cid & STM32_GPIO_CIDCFGR_SEMEN)) {
        // 信号量未使能：检查静态 CID 归属
        if (FIELD_GET(STM32_GPIO_CIDCFGR_SCID_MASK, cid) == STM32_GPIO_CID1)
            return true;          // 归属 CID1（Linux），可以访问
        return false;             // 归属其他 CID，拒绝访问
    }

    // 信号量使能：检查写名单中是否包含 CID1
    if (cid & STM32_GPIO_CIDCFGR_SEMWL_CID1)
        return true;
    return false;
}
```

OP-TEE 在安全岛 `0x46200030` 写入的 SECCFGR 值通常比主域 `0x44240030` 更严格（更多 bit 为 1），同时安全岛的每个 GPIOZ 引脚还有独立的 CIDCFGR 配置决定是否允许 Linux 访问。两层过滤叠加，导致 Linux 实际可用的 GPIOZ 引脚数少于硬件引脚数。

| 对比项 | 主域 pinctrl（GPIOA~I） | 安全域 pinctrl_z（GPIOZ） |
|--------|-----------------------|--------------------------|
| 物理地址 | `0x44240000 ~ 0x44A03FF` | `0x46200000 ~ 0x462003FF` |
| compatible | `st,stm32mp257-pinctrl` | `st,stm32mp257-z-pinctrl` |
| match_data | `stm32mp257_match_data` | `stm32mp257_z_match_data` |
| bank 数 | 9 | 1 |
| 引脚总数 | 数百 | 10（PZ0~PZ9） |
| 中断路由 | `exti1` | `exti1`，但 `bank_ioport_nr = 11` 区分 |
| SECCFGR 效果 | OP-TEE 标记少量安全引脚 | 安全岛上更多引脚被标记 |
| ATK 板实际用途 | UART/SDMMC/ETH/I2C/SPI 等 | SPI8_CS、VBUS 检测、传感器 |

#### 3.4.8.2 取所有 bank 的 clock 和 reset

```c
// pinctrl-stm32.c:2127
for_each_gpiochip_node(dev, child) {
    struct stm32_gpio_bank *bank = &pctl->banks[i];
    struct device_node *np = to_of_node(child);

    bank->rstc = of_reset_control_get_exclusive(np, NULL);   // 取 reset
    if (PTR_ERR(bank->rstc) == -EPROBE_DEFER) {
        fwnode_handle_put(child);
        return -EPROBE_DEFER;
    }

    pctl->clks[i].clk = of_clk_get_by_name(np, NULL);        // 取 clock
    if (IS_ERR(pctl->clks[i].clk)) {
        fwnode_handle_put(child);
        return dev_err_probe(dev, PTR_ERR(pctl->clks[i].clk),
                             "failed to get clk\n");
    }
    pctl->clks[i].id = "pctl";
    i++;
}

clk_bulk_prepare_enable(banks, pctl->clks);  // 使能所有 bank 时钟
```

以 GPIOA 的 DTS 子节点为例：

```dts
gpioa: gpio@44240000 {
    clocks = <&scmi_clk CK_SCMI_GPIOA>;
    resets = <&scmi_reset RST_SCMI_GPIOA>;
};
```

遍历 9 个 bank 后，`pctl->clks[]` 的内容变为：

```
pctl->clks[0].clk = GPIOA 时钟句柄（来自 SCMI 协议，clock ID = CK_SCMI_GPIOA）
pctl->clks[0].id  = "pctl"                ← 驱动自定义标签，供 bulk API 调试用
pctl->clks[1].clk = GPIOB 时钟句柄
pctl->clks[1].id  = "pctl"
...
pctl->clks[8].clk = GPIOI 时钟句柄
pctl->clks[8].id  = "pctl"
```

同时，`pctl->banks[i].rstc` 保存每个 bank 的复位控制器句柄。`reset` 在硬件上对应 RCC 中每个 GPIO bank 的独立复位控制位：

```c
// drivers/clk/stm32/stm32mp25_rcc.h
#define RCC_GPIOACFGR        0x52C
#define RCC_GPIOACFGR_GPIOARST   BIT(0)   // ← 复位控制位，1=复位状态
#define RCC_GPIOACFGR_GPIOAEN    BIT(1)   // 时钟使能位
```

STM32MP25 上电时 RCC 默认将 **所有 GPIO bank 保持在复位状态**（`GPIOxRST = 1`），此时：
- GPIO 寄存器不可读写（`readl` 可能返回 0，`writel` 写入无效）
- 引脚输出强制为默认复位电平

驱动初始化序列中，`reset_control_deassert()` 会通过 SCMI 协议委托 OP-TEE 将 `RCC_GPIOxCFGR` 的 bit 0 清 0，GPIO bank 才能正常工作：

```
                      ┌───────────────────────┐
  上电默认             │ RCC_GPIOxCFGR         │
  GPIOxRST = 1 ──────→│ bit 0 (RST) = 1       │ → GPIO 寄存器不可操作
                      │ bit 1 (EN)  = 0       │
                      └───────────────────────┘
                              │
  reset_control_deassert()    │ SCMI 协议 → OP-TEE
                              ▼
                      ┌───────────────────────┐
  解复位后             │ RCC_GPIOxCFGR         │
  GPIOxRST = 0 ──────→│ bit 0 (RST) = 0       │ → GPIO 寄存器可正常访问
                      │ bit 1 (EN)  = 1       │    （时钟已由 clk_bulk 使能）
                      └───────────────────────┘
```

注意时序：`clk_bulk_prepare_enable()` 先使能时钟，`reset_control_deassert()` 在后面的 `stm32_gpiolib_register_bank()` 中才调用。**必须先开时钟再解复位**，否则解复位操作本身也需要时钟。

如果 DTS 中没有 `resets` 属性，`of_reset_control_get_exclusive()` 返回 `-ENOENT`（不是错误），后续 `reset_control_deassert()` 会直接跳过。

`clk_bulk_prepare_enable(9, pctl->clks)` 一次性使能 9 个 bank 的 AHB 总线时钟——所有 GPIO 寄存器模块的时钟必须在后续寄存器访问前就绪。

**为什么在注册 GPIO 之前使能时钟？** `gpiochip_add_data()` 在注册时会调用 `gpiochip_init_valid_mask()`，后者可能会读取硬件寄存器（如 SECCFGR 寄存器检查安全状态）。如果时钟未使能，寄存器读取会挂死或返回 0。

### 3.4.9 逐个注册 GPIO bank

```c
// pinctrl-stm32.c:2153
for_each_gpiochip_node(dev, child) {
    ret = stm32_gpiolib_register_bank(pctl, child);
    if (ret) {
        fwnode_handle_put(child);
        goto err_register;
    }
    pctl->nbanks++;
}
```

`stm32_gpiolib_register_bank()` 完成单个 GPIO bank 的全部初始化工作。我们逐行分析这个函数。

#### 3.4.9.1 复位解除与 IO 映射

```c
// pinctrl-stm32.c:1722
static int stm32_gpiolib_register_bank(struct stm32_pinctrl *pctl,
                                        struct fwnode_handle *fwnode)
{
    struct stm32_gpio_bank *bank = &pctl->banks[pctl->nbanks];
    ...
    // 解除复位（可选）
    if (!IS_ERR(bank->rstc))
        reset_control_deassert(bank->rstc);

    // IO remap（物理地址 → 虚拟地址）
    if (of_address_to_resource(to_of_node(fwnode), 0, &res))
        return -ENODEV;
    bank->base = devm_ioremap_resource(dev, &res);
```

`reset_control_deassert()` 将 GPIO bank 从复位状态释放。如果 DTS 中没有定义 `resets` 属性，`bank->rstc` 为 `-ENOENT`，`IS_ERR` 检测到后会跳过。

`devm_ioremap_resource()` 将物理地址（如 GPIOA 的 `0x44240000`）映射到内核虚拟地址空间。后续所有寄存器操作（`readl`/`writel`）都通过 `bank->base + offset` 进行。

GPIO bank 的地址空间分配：

| Bank | 物理地址 | 长度 | 虚拟地址（映射后）|
|------|---------|------|-----------------|
| GPIOA | 0x44240000 | 0x400 | bank->base + 0x00 |
| GPIOB | 0x44250000 | 0x400 | bank->base + 0x00 |
| GPIOC | 0x44260000 | 0x400 | bank->base + 0x00 |
| ... | ... | ... | ... |
| GPIOI | 0x442C0000 | 0x400 | bank->base + 0x00 |

以 GPIOA 为例，`reg = <0x0 0x400>`（偏移 0x0），`of_address_to_resource()` 结合父节点的 `ranges` 翻译后得到物理地址 `0x44240000`，`devm_ioremap_resource()` 映射到内核虚拟地址空间。此后每个 bank 的寄存器基址 `bank->base` 记录如下：

```
bank (GPIOA) → base = ioremap(0x44240000, 0x400)    ← 映射后虚拟地址
bank (GPIOB) → base = ioremap(0x44250000, 0x400)
bank (GPIOC) → base = ioremap(0x44260000, 0x400)
...
bank (GPIOI) → base = ioremap(0x442C0000, 0x400)
```

所有 bank 的寄存器偏移布局相同，只是 base 不同：

| 偏移 | 寄存器 | 位宽 | 描述 |
|------|--------|------|------|
| `base+0x00` | MODER | 32 | 模式（输入/输出/复用/模拟） |
| `base+0x04` | OTYPER | 32 | 输出类型（推挽/开漏） |
| `base+0x08` | OSPEEDR | 32 | 输出速度 |
| `base+0x0C` | PUPDR | 32 | 上下拉 |
| `base+0x10` | IDR | 32 | 输入数据 |
| `base+0x14` | ODR | 32 | 输出数据 |
| `base+0x24` | AFRL | 32 | 复用功能选择低位 |
| `base+0x28` | AFRH | 32 | 复用功能选择高位 |
| `base+0x3C` | SECCFGR | 32 | 安全配置（MP257 新增） |

后续驱动调用 `readl(bank->base + 0x00)` 读取 GPIOA 的 MODER（物理地址 0x44240000），`writel(val, bank->base + 0x14)` 设 GPIOA ODR（物理 0x44240014）。

如果 DTS 中没有 `resets` 属性，`bank->rstc` 为 `-ENOENT`，`reset_control_deassert()` 直接跳过——GPIO bank 默认不在复位状态。

#### 3.4.9.2 gpio_chip 初始化和个性化

```c
// pinctrl-stm32.c:1745
bank->gpio_chip = stm32_gpio_template;  // 从模板复制

fwnode_property_read_string(fwnode, "st,bank-name", &bank->gpio_chip.label);
// label = "GPIOA"

bank->gpio_chip.base   = -1;           // 动态分配全局编号
bank->gpio_chip.ngpio  = npins;         // 本 bank 引脚数（来自 gpio-ranges）
bank->gpio_chip.fwnode = fwnode;        // 指向 DTS 子节点
bank->gpio_chip.parent = dev;           // 父设备 = pinctrl device
```

**`stm32_gpio_template` 静态模板**（`pinctrl-stm32.c:497`）：

```c
static const struct gpio_chip stm32_gpio_template = {
    .request          = stm32_gpio_request,
    .free             = stm32_gpio_free,
    .get              = stm32_gpio_get,
    .set              = stm32_gpio_set,
    .direction_input  = stm32_gpio_direction_input,
    .direction_output = stm32_gpio_direction_output,
    .to_irq           = stm32_gpio_to_irq,
    .get_direction    = stm32_gpio_get_direction,
    .set_config       = gpiochip_generic_config,
    .init_valid_mask  = stm32_gpio_init_valid_mask,
};
```

模板定义了所有 GPIO bank 共用的回调函数。每个 bank 从模板拷贝一份（`bank->gpio_chip = stm32_gpio_template`，不是指针，是结构体赋值），然后覆盖 `.label`、`.base`、`.ngpio`、`.fwnode`、`.parent` 等个性字段。

以 GPIOA 为例，拷贝覆盖后 `bank->gpio_chip` 的内容：

```
bank->gpio_chip  ← 从 stm32_gpio_template 拷贝
  ├── .request          = stm32_gpio_request          ← 模板回调
  ├── .free             = stm32_gpio_free             ← 模板回调
  ├── .get              = stm32_gpio_get              ← 模板回调
  ├── .set              = stm32_gpio_set              ← 模板回调
  ├── .direction_input  = stm32_gpio_direction_input  ← 模板回调
  ├── .direction_output = stm32_gpio_direction_output ← 模板回调
  ├── .to_irq           = stm32_gpio_to_irq           ← 模板回调
  ├── .get_direction    = stm32_gpio_get_direction    ← 模板回调
  ├── .set_config       = gpiochip_generic_config     ← 模板回调
  ├── .init_valid_mask  = stm32_gpio_init_valid_mask  ← 模板回调
  │
  ├── .label   = "GPIOA"          ← 来自 DTS st,bank-name
  ├── .base    = -1               ← 强制动态分配（后续 gpiochip_find_base 改）
  ├── .ngpio   = 16               ← 来自 gpio-ranges 解析
  ├── .fwnode  = &gpioa_fwnode    ← DTS 节点 fwnode_handle
  └── .parent  = &pdev->dev       ← 父设备 = 44240000.pinctrl
```

所有 9 个 bank 共用同一套回调函数——区别仅在于回调内部通过 `gpio_chip` → `container_of` 反算回 `stm32_gpio_bank`，进而拿到不同的 `bank->base` 地址。例如 `stm32_gpio_set()` 内部：

```c
bank = container_of(gc, struct stm32_gpio_bank, gpio_chip);
// 用 bank->base 写 ODR 寄存器
writel(val, bank->base + reg_offset);
```

#### 3.4.9.3 gpio-ranges 解析

```c
// pinctrl-stm32.c:1749
if (!fwnode_property_get_reference_args(fwnode, "gpio-ranges", NULL, 3, i, &args)) {
    // 解析第一个 gpio-ranges 条目
    bank_nr = args.args[1] / STM32_GPIO_PINS_PER_BANK;
    bank->gpio_chip.base = args.args[1];   // 临时赋值为 pin_base

    // 遍历所有条目，取最大引脚范围
    npins = args.args[0] + args.args[2];
    while (!fwnode_property_get_reference_args(fwnode, "gpio-ranges", NULL, 3, ++i, &args))
        npins = max(npins, (int)(args.args[0] + args.args[2]));
} else {
    // 无 gpio-ranges 属性：手动构建 range
    ...
}
...
bank->gpio_chip.base = -1;  // ★ 改写为 -1，强制动态分配
bank->gpio_chip.ngpio = npins;
```

这段代码只做了一个事：**提取本 bank 的引脚数 `npins`**。例如 `gpio-ranges = <&pinctrl 0 0 16>` 的三个参数中，`args.args[0] + args.args[2] = 0 + 16 = 16`，驱动就知道 GPIOA 有 16 个引脚。

它**没有**创建翻译表。`bank->gpio_chip.base` 虽然临时赋值为 `args.args[1]`（pin_base），但最后改为 `-1`，意思是"让 gpiolib core 动态分配全局编号"。真正创建 `pinctrl_gpio_range` 翻译表的代码在 gpiolib core 层——`gpiochip_add_data()` 内部。

#### 3.4.9.4 of_gpiochip_add_pin_range — 从 DTS 创建翻译表

这是 gpiolib-of.c 的核心函数，功能是：**解析 `gpio-ranges` 属性，为每个条目创建一个 `pinctrl_gpio_range` 实例，同时挂入 pinctrl 和 GPIO 两侧的链表**。

```c
// gpiolib-of.c:1012
static int of_gpiochip_add_pin_range(struct gpio_chip *chip)
{
    struct of_phandle_args pinspec;
    struct pinctrl_dev *pctldev;
    int index = 0, ret;

    for (;; index++) {
        // 逐条解析 gpio-ranges
        // 格式：gpio-ranges = <&phandle gpio_offset pin_offset npins>;
        ret = of_parse_phandle_with_fixed_args(np, "gpio-ranges", 3,
                                                index, &pinspec);
        if (ret)
            break;  // 没有后续条目了

        // pinspec.np    = &pinctrl 节点（通过 phandle 引用）
        // pinspec.args[0] = gpio_offset  ← bank 内偏移起点（如 GPIOH 为 2）
        // pinspec.args[1] = pin_offset   ← pinctrl pin 号起点（如 GPIOH 为 114）
        // pinspec.args[2] = npins        ← 连续映射的引脚数（如 GPIOH 为 12）

        // 通过 phandle 找到对应的 pinctrl_dev
        pctldev = of_pinctrl_get(pinspec.np);
        if (!pctldev)
            return -EPROBE_DEFER;  // pinctrl 还没注册，待会重试

        if (pinspec.args[2]) {  // npins != 0
            ret = gpiochip_add_pin_range(chip,
                        pinctrl_dev_get_devname(pctldev),  // pinctrl 设备名
                        pinspec.args[0],    // gpio_offset
                        pinspec.args[1],    // pin_offset
                        pinspec.args[2]);   // npins
        }
    }
    return 0;
}
```

`gpiochip_add_pin_range()` 是真正创建翻译表的地方。注意它分配的不是直接的 `pinctrl_gpio_range`，而是它的包装器 `gpio_pin_range`：

```c
// gpiolib.c:2022 — 完整源码，非缩略
int gpiochip_add_pin_range(struct gpio_chip *gc, const char *pinctl_name,
                           unsigned int gpio_offset, unsigned int pin_offset,
                           unsigned int npins)
{
    // ★ 分配的是 gpio_pin_range（包装器），不是直接的 pinctrl_gpio_range
    struct gpio_pin_range *pin_range;
    struct gpio_device *gdev = gc->gpiodev;

    pin_range = kzalloc(sizeof(*pin_range), GFP_KERNEL);

    // ── ① 填充翻译表数据 ──
    pin_range->range.id       = gpio_offset;          // bank 内偏移起点
    pin_range->range.gc       = gc;                   // 回指 gpio_chip
    pin_range->range.name     = gc->label;            // "GPIOA"
    pin_range->range.base     = gdev->base + gpio_offset;  // ★ GPIO 全局编号
    pin_range->range.pin_base = pin_offset;           // pinctrl pin 号起点
    pin_range->range.npins    = npins;                // 引脚数

    // ── ② 挂入 pinctrl 侧链表 ──
    pin_range->pctldev = pinctrl_find_and_add_gpio_range(pinctl_name,
                                                          &pin_range->range);
    // pinctrl_find_and_add_gpio_range() 内部：
    //   (a) 根据 pinctl_name 遍历 pinctrldev_list 找到 pctldev
    //   (b) pinctrl_add_gpio_range(pctldev, &pin_range->range)
    //       → list_add_tail(&pin_range->range.node, &pctldev->gpio_ranges)

    // ── ③ 挂入 GPIO 侧链表 ──
    list_add_tail(&pin_range->node, &gdev->pin_ranges);
    //          ↑ 注意这里是 pin_range->node，不是 range.node

    return 0;
}
```

这里涉及两个不同的链表节点，注意区分：

```c
// ── 每个翻译表对应一个包装器 ──
struct gpio_pin_range {               // include/linux/gpio/driver.h:720
    struct list_head node;              // node #1 → 挂入 gpio_device->pin_ranges
    struct pinctrl_dev *pctldev;        // 指向 pinctrl_dev，方便 GPIO 侧找回 pinctrl
    struct pinctrl_gpio_range range;    // 内嵌实际翻译表
};

// ── 内嵌的翻译表数据 ──
struct pinctrl_gpio_range {            // include/linux/pinctrl/pinctrl.h:78
    struct list_head node;              // node #2 → 挂入 pinctrl_dev->gpio_ranges
    const char *name;                   // 名称，如 "GPIOA"
    unsigned int base;                  // ★ GPIO 全局编号起点
    unsigned int pin_base;              // ★ pinctrl pin 号起点
    unsigned int npins;                 // 连续映射数
    struct gpio_chip *gc;               // 回指 gpio_chip，方便 pinctrl 侧找回 GPIO
};
```

**"双链表"的实质：** 一个 `gpio_pin_range` 包装器内嵌 `pinctrl_gpio_range`，后者的 `.node` 挂入 pinctrl 侧链表，前者的 `.node` 挂入 GPIO 侧链表——**同一块内存，两套索引**：

```
        ┌──────────────────────────────────────────────────┐
        │  gpio_pin_range  (kzalloc 一次，单块内存)         │
        │                                                    │
        │  ┌────────────────────────────────────────────┐    │
        │  │  .node ──────────────────→ gdev->pin_ranges │    │
        │  │  .pctldev → pinctrl_dev                     │    │
        │  │                                              │    │
        │  │  .range (pinctrl_gpio_range) {               │    │
        │  │    .node ───────────────→ pctldev->gpio_ranges│    │
        │  │    .name     = "GPIOA"                       │    │
        │  │    .base     = 0    (GPIO 全局编号起点)      │    │
        │  │    .pin_base = 0    (pinctrl pin 号起点)     │    │
        │  │    .npins    = 16                            │    │
        │  │    .gc       = &GPIOA_chip                   │    │
        │  │  }                                            │    │
        │  └────────────────────────────────────────────┘    │
        └──────────────────────────────────────────────────┘
```

这种分离设计的目的是：**数据是一份，但两个子系统各自从自己的入口遍历**——pinctrl 侧遍历 `pctldev->gpio_ranges`（通过 `pinctrl_gpio_range.node`），GPIO 侧遍历 `gdev->pin_ranges`（通过 `gpio_pin_range.node`）。

#### 3.4.9.5 翻译表怎么用：双向翻译举例

**场景一：GPIO → pinctrl**（GPIO core 已知 GPIO 全局号，找对应的 pinctrl pin）

例如驱动调用 `gpio_request(628, "mydev")`，GPIO core 通过 gpio_ranges 检查这个引脚是否被 pinctrl 管理，需要找到对应的 pinctrl pin 号：

```
① GPIO core 遍历 gdev->pin_ranges 链表
   逐个检查：range.base ≤ GPIO号 < range.base + range.npins？

② GPIOG range：base=606, npins=16 → 606 ≤ 628 < 622？→ ❌

③ GPIOH range：base=624, npins=12 → 624 ≤ 628 < 636？→ ✅ 命中！
   range.pin_base = 114

④ 计算 pinctrl pin 号：
   pin = range.pin_base + (GPIO号 - range.base)
       = 114 + (628 - 624)
       = 118
   → 这是 pinctrl 全局 pin 118（即 PH6），GPIO core 可以通知 pinctrl
      "我要用 pin 118，你检查下有没有冲突"
```

**场景二：Pinctrl → GPIO**（pinctrl 已知 pin 号，找对应的 GPIO chip 回调）

例如 pinctrl 要把 pin 118 从复用功能切回 GPIO 模式（如 `pinmux_disable_setting()` 时），它只知道这是 pinctrl 全局 pin 118，但切 GPIO 需要调用 `gpio_chip` 的回调——必须找到它属于哪个 bank：

```
① pinctrl core 持有 pin 号 118（即 PH4）
   需要调用 gpio_chip.direction_input() 或 gpio_chip.set()，
   但不知道 118 对应哪个 gpio_chip

② 遍历 pctldev->gpio_ranges 链表
   逐个检查：range.pin_base ≤ pin号 < range.pin_base + range.npins？

③ GPIOG range：pin_base=96, npins=16 → 96 ≤ 118 < 112？→ ❌
   注意这里比较的是 pin_base，不是 base——因为输入是 pinctrl pin 号

④ GPIOH range：pin_base=114, npins=12 → 114 ≤ 118 < 126？→ ✅ 命中！
   range.gc = &GPIOH_chip    ← 拿到了 gpio_chip，可以调用回调

⑤ 至此 pinctrl 知道了两件事：
   - 这个 pin 在 GPIOH bank（`range.gc = &GPIOH_chip`）
   - 它在 bank 内的引脚名 = PH4（通过 pin 118 - pin_base 114 = 4，即 PH2+4）
   → 可以调用 GPIOH_chip.direction_input(gc, 4) 来切回 GPIO 模式
```

#### 3.4.9.6 ATK 板完整的翻译表

ATK 板 DTS（`stm32mp25xxak-pinctrl.dtsi`）中主域 9 个 bank 的 `gpio-ranges`：

```dts
&pinctrl {
    gpioa: gpio@44240000 { gpio-ranges = <&pinctrl 0   0  16>; };
    gpiob: gpio@44250000 { gpio-ranges = <&pinctrl 0  16  16>; };
    gpioc: gpio@44260000 { gpio-ranges = <&pinctrl 0  32  14>; };
    gpiod: gpio@44270000 { gpio-ranges = <&pinctrl 0  48  16>; };
    gpioe: gpio@44280000 { gpio-ranges = <&pinctrl 0  64  16>; };
    gpiof: gpio@44290000 { gpio-ranges = <&pinctrl 0  80  16>; };
    gpiog: gpio@442a0000 { gpio-ranges = <&pinctrl 0  96  16>; };
    gpioh: gpio@442b0000 { gpio-ranges = <&pinctrl 2 114  12>; };  // PH0~1 不存在
    gpioi: gpio@442c0000 { gpio-ranges = <&pinctrl 0 128  12>; };
};
```

`gpio-ranges = <&pinctrl gpio_offset pin_offset npins>`：

| 参数 | 含义 | 例如 GPIOH |
|------|------|-----------|
| `gpio_offset` | bank 内从第几个引脚开始映射 | 2 → PH0/PH1 不存在，从 PH2 开始 |
| `pin_offset` | 对应 pinctrl 全局 pin 号起点 | 114 → PH2 是全局第 114 号 |
| `npins` | 连续映射多少个 | 12 → PH2 ~ PH13 |

执行 `gpiochip_add_pin_range()` 后，每个 bank 生成一个翻译表：

| bank | range.base | range.pin_base | npins | 映射关系 |
|------|-----------|---------------|-------|---------|
| GPIOA | `512+0=512` | 0 | 16 | 512~527 → pin 0~15 |
| GPIOB | `528+0=528` | 16 | 16 | 528~543 → pin 16~31 |
| GPIOC | `544+0=544` | 32 | 14 | 544~557 → pin 32~45 |
| GPIOD | `558+0=558` | 48 | 16 | 558~573 → pin 48~63 |
| GPIOE | `574+0=574` | 64 | 16 | 574~589 → pin 64~79 |
| GPIOF | `590+0=590` | 80 | 16 | 590~605 → pin 80~95 |
| GPIOG | `606+0=606` | 96 | 16 | 606~621 → pin 96~111 |
| GPIOH | `622+2=624` | 114 | 12 | 624~635 → pin 114~125 |
| GPIOI | `634+0=634` | 128 | 12 | 634~645 → pin 128~139 |

`base = gdev->base + gpio_offset`。`GPIO_DYNAMIC_BASE = 512`（`include/linux/gpio.h:76`），`gpiochip_find_base()` 从 512 开始分配。GPIOA 获得 base=512，之后每个 bank 连续递增。GPIOH 的 `gdev->base=622`，加 `gpio_offset=2` 得到 range.base=624。

#### 3.4.9.7 引脚别名

这段代码给 debugfs 显示用的，让 `cat /sys/kernel/debug/gpio` 能看到 `PA0`、`PH4` 这样的引脚名而非编号。

```c
// pinctrl-stm32.c:1785
for (i = 0; i < npins; i++) {
    // 找：bank 内偏移 i 对应哪个 pinctrl 引脚描述
    stm32_pin = stm32_pctrl_get_desc_pin_from_gpio(pctl, bank, i);
    if (stm32_pin && stm32_pin->pin.name)
        names[i] = devm_kasprintf(dev, GFP_KERNEL, "%s", stm32_pin->pin.name);
    else
        names[i] = NULL;
}
bank->gpio_chip.names = (const char * const *)names;
```

内核里引脚名存在 `pctl->pins[]` 数组中，每个元素是一个 `stm32_desc_pin`，其中有 `.pin.name`（如 `"PA0"`）和 `.pin.number`（pin 号）。问题是：**给了一个 bank 和内部偏移，怎么从 `pctl->pins[]` 中找到对的条目？**

核心函数 `stm32_pctrl_get_desc_pin_from_gpio()`（`pinctrl-stm32.c:1698`）：

```c
static struct stm32_desc_pin *
stm32_pctrl_get_desc_pin_from_gpio(struct stm32_pinctrl *pctl,
                                    struct stm32_gpio_bank *bank,
                                    unsigned int offset)
{
    // ── 先按 "bank_nr × 16 + offset" 直接算 pin 号 ──
    unsigned int stm32_pin_nb = bank->bank_nr * STM32_GPIO_PINS_PER_BANK + offset;

    // 策略一（快速路径）：假设 pctl->pins[pin号] = 该 pin 的描述
    // 适合绝大多数 bank——PA0 的 pin.number=0，就在 pctl->pins[0]
    if (stm32_pin_nb < pctl->npins) {
        pin_desc = pctl->pins + stm32_pin_nb;  // 直接数组索引
        if (pin_desc->pin.number == stm32_pin_nb)
            return pin_desc;  // 校验通过（编号 = 索引），直接返回
    }

    // 策略二（慢速路径）：线性遍历整个 pins 数组
    // 当快速路径失败时——因为某些 pin 被 package 过滤掉了，
    // 导致数组索引不再等于 pin 号
    for (i = 0; i < pctl->npins; i++) {
        pin_desc = pctl->pins + i;
        if (pin_desc->pin.number == stm32_pin_nb)
            return pin_desc;
    }
    return NULL;  // 这个引脚在当前 package 上不存在
}
```

**为什么需要两种策略？** 因为 `pctl->pins[]` 是 **稠密数组**。`stm32_pctrl_create_pins_tab()` 建表时会跳过当前 package 不存在的 pin：

```
原始静态表 match_data->pins[]          package 过滤 (PKG_AK) →  pctl->pins[]
┌───────────────────────────────┐                              ┌─────────────────┐
│ PIN(0,   "PA0")   PKG_AK ✓   │                              │ [0] = PA0  (0)  │
│ PIN(1,   "PA1")   PKG_AK ✓   │   跳过不匹配 package 的条目   │ [1] = PA1  (1)  │
│ ...                           │   ────────────────────────→  │ ...              │
│ PIN(112, "PH0")   PKG_AK ✗   │                              │ [111] = (shift) │
│ PIN(113, "PH1")   PKG_AK ✗   │                              │ [112] = PH2(114)│
│ PIN(114, "PH2")   PKG_AK ✓   │                              │ [113] = PH3(115)│
│ ...                           │                              │ ...              │
└───────────────────────────────┘                              └─────────────────┘
```

结果：`pctl->pins[]` 的索引不再等于 pin 号（PH2 的 pin 号 114，但在数组中的索引是 112）。

以 GPIOH 为例——`bank_nr = 7`（`114/16=7`），`STM32_GPIO_PINS_PER_BANK = 16`：

| bank 内偏移 | 期望引脚 | stm32_pin_nb | 快速路径（数组索引） | 校验结果 | 慢速路径 |
|-----------|---------|-------------|-------------------|---------|---------|
| 0 | PH0（不存在） | 112 | `pctl->pins[112]` = 114 | `114≠112` → ❌ | 线查找 112 → 无 → NULL ✅ |
| 1 | PH1（不存在） | 113 | `pctl->pins[113]` = 115 | `115≠113` → ❌ | 线查找 113 → 无 → NULL ✅ |
| 2 | PH2 | 114 | `pctl->pins[114]` = 116 | `116≠114` → ❌ | 线查找 114 → **找到** ✅ |
| 3 | PH3 | 115 | `pctl->pins[115]` = 117 | `117≠115` → ❌ | 线查找 115 → **找到** ✅ |
| ... | ... | ... | ... | ... | ... |

> 注：PH0/PH1 在 AK 封装上不存在（`~STM32MP_PKG_AK`），所以原始表中这两个条目被跳过。`pctl->pins[112]` 存的是 PH2（pin 114），导致偏移+2。

快速路径对前几个 bank（A~G，无过滤）完全命中，效率最高。慢速路径负责兜底 GPIOH 这种因 package 过滤产生偏移的 bank，以及 GPIOZ（pin 号 400+，完全不在主域数组范围内）。

最终效果——debugfs 中看到的是名字而非编号：

```
gpiochip0: GPIOs 0-15, parent: platform/44240000.pinctrl, GPIOA:
 gpio-0  (                    |PA0                   ) in  lo
 gpio-1  (                    |PA1                   ) in  lo
 ...
```

#### 3.4.9.8 gpiochip_add_data — 注册全过程

`stm32_gpiolib_register_bank()` 最后调用 `gpiochip_add_data(&bank->gpio_chip, bank)`，进入 gpiolib core 层。完整的注册流程如下：

```
gpiochip_add_data(&bank->gpio_chip, bank)
  │
  ├── gpiochip_find_base()          ← 因为 base = -1，动态分配全局编号
  │                                    GPIOA→0, GPIOB→16, GPIOC→32...
  │
  ├── gpio_device_add_to_list()     ← 挂入全局链表 gpio_devices
  │
  ├── of_gpiochip_add()             ← ★ of 层初始化，核心在此
  │     ├── of_gpiochip_add_pin_range()   ← 解析 gpio-ranges，创建翻译表
  │     └── of_gpiochip_scan_gpios()      ← gpio-hog 处理（DTS 预置输出）
  │
  ├── cdev_device_add()             ← 创建 /dev/gpiochipN 字符设备
  │
  └── gpiochip_sysfs_register()     ← 创建 /sys/class/gpio/gpiochipN/
```

`gpiochip_add_data` 不是一个独立函数，而是一个**宏**（`include/linux/gpio/driver.h:580`）：

```c
#define gpiochip_add_data(gc, data) \
    gpiochip_add_data_with_key(gc, data, &lock_key, &request_key)
```

它直接展开为 `gpiochip_add_data_with_key()`，多了两个 lockdep 调试参数（生产环境时为 NULL）。所以上面概览图中的每一步，实际执行的都是 `gpiochip_add_data_with_key`。下面逐步骤分析：

```
gpiochip_add_data_with_key(gc, bank, &lock_key, &request_key)
  │
  ├── ① 分配 gpio_device
  │     gdev = kzalloc(sizeof(*gdev), GFP_KERNEL);
  │
  ├── ② 填充基本字段
  │     gdev->dev     → 初始化 gpio_device 的设备
  │     gdev->chip    = gc       （互指）
  │     gdev->owner   = gc->owner
  │     gc->gpiodev   = gdev     （互指）
  │
  ├── ③ 获取 ngpio（来自 DTS 或 gc->ngpio）
  │     gpiochip_get_ngpios(gc, &gdev->dev);
  │
  ├── ④ ★ 分配 gpio_desc[] 数组
  │     gdev->ngpio  = gc->ngpio;
  │     gdev->descs  = kcalloc(gc->ngpio, sizeof(struct gpio_desc), GFP_KERNEL);
  │     for (i = 0; i < gc->ngpio; i++)
  │         gdev->descs[i].gdev = gdev;   // 每个 desc 指回 gdev
  │     gdev->label  = kstrdup(gc->label, GFP_KERNEL);
  │
  ├── ★ spin_lock_irqsave 保护以下⑤⑥操作
  │
  ├── ⑤ 分配 GPIO 全局编号
  │     base = gc->base;               // 读取驱动设置的值（STM32 设为 -1）
  │     if (base < 0) {                // 驱动要求动态分配
  │         base = gpiochip_find_base(gc->ngpio);  // 找空闲编号区间
  │         if (base < 0)
  │             goto err_free_label;    // 编号空间耗尽
  │         gc->base = base;           // 改写 gc 中的 base
  │     } else {
  │         dev_warn("静态分配已废弃"); // 不覆盖，但打印警告
  │     }
  │     gdev->base = base;             // gdev 记录最终 base
  │
  ├── ⑥ 挂入 gpio_devices 全局链表
  │     ret = gpiodev_add_to_list(gdev);  // 将 gdev 加入 gpio_devices 链表
  │     if (ret) {                        // 编号区间重叠 → 失败
  │         spin_unlock_irqrestore(&gpio_lock, flags);
  │         goto err_free_label;
  │     }
  │     // 链表挂入成功，再填充 desc 互指
  │     for (i = 0; i < gc->ngpio; i++)
  │         gdev->descs[i].gdev = gdev;
  │
  │     spin_unlock_irqrestore(&gpio_lock, flags);
  │
  ├── ⑦ ★ 解析 gpio-ranges → pinctrl_gpio_range 创建
  │     of_gpiochip_add(gc);
  │       ├── of_gpiochip_add_pin_range()   // gpio-ranges → range
  │       └── of_gpiochip_scan_gpios()       // GPIO hog 处理
  │
  ├── ⑧ 有效引脚掩码
  │     gpiochip_init_valid_mask(gc);
  │
  ├── ⑨ 注册字符设备 /dev/gpiochipN
  │     dev_set_name(&gdev->dev, "gpiochip%d", gdev->id);
  │     cdev_init(&gdev->chrdev, &gpio_fileops);
  │     cdev_add(&gdev->chrdev, gdev->dev.devt, 1);
  │
  └── ⑩ 注册 sysfs 接口（如果 gpio_class 已注册）
        gpiochip_sysfs_register(gdev);
```

`gpiochip_add_data()` 执行过程中，gpiolib core 层为这个 bank 创建完整的运行时数据结构。以第一个注册的 GPIOA 为例，注册过程中各步骤的数据变化如下：

**步骤 ①~②**：分配 `gpio_device` 并建立互指关系

```
gdev = kzalloc(sizeof(gpio_device))
gdev->chip  = &bank->gpio_chip        ← 互指
bank->gpio_chip.gpiodev = gdev        ← 互指
```

**步骤 ④**：分配 `gpio_desc[ngpio]` 数组

```
gdev->ngpio = 16
gdev->descs = gpio_desc[16]  ← 每个 desc.flags = 0, desc.gdev = gdev
              ├── [0]  { .flags=0, .gdev=gdev }
              ├── [1]  { .flags=0, .gdev=gdev }
              └── [15] { .flags=0, .gdev=gdev }
```

每个 `gpio_desc` 的 `flags` 此时为 0——还没有任何驱动通过 `devm_gpiod_get()` 请求这些 GPIO。后续消费者驱动调用 `gpiod_get()` 时，core 层会在 `gpiod_find_and_request()` 中设置 `FLAG_REQUESTED` 和 `label`。

**步骤 ⑤**：动态编号分配

```c
base = gc->base;                     // = -1（驱动设为 -1 表示要动态分配）
if (base < 0) {                      // 进入动态分配路径
    base = gpiochip_find_base(16);   // 查找空闲区间
    gc->base = base;                 // = 512（GPIOA 获得 512~527）
}
gdev->base = base;                   // gdev 记录最终 base
```

GPIOA 获得 base=512（全局编号 512~527），GPIOB 获得 base=528（528~543），以此类推。`GPIO_DYNAMIC_BASE = 512`（`include/linux/gpio.h:76`），动态分配从 512 开始。GPIO 全局编号与 pinctrl pin 号是两套独立编号系统——前者从 512 开始，后者从 0 开始，通过 gpio-ranges 翻译表关联。

**步骤 ⑥**：挂入 gpio_devices 全局链表

```c
// 在 spin_lock_irqsave 保护下
ret = gpiodev_add_to_list(gdev);
if (ret) {                              // 编号区间与已有 chip 重叠
    spin_unlock_irqrestore(&gpio_lock, flags);
    goto err_free_label;
}
// 挂入成功后，补全 desc 互指（之前只分配了数组，没填 gdev 指针）
for (i = 0; i < gc->ngpio; i++)
    gdev->descs[i].gdev = gdev;

spin_unlock_irqrestore(&gpio_lock, flags);
```

`gpiodev_add_to_list()` 将 `gdev` 插入全局链表 `gpio_devices`。之后 `gpiochip_find_base()` 分配编号时就知道哪些区间已被占用。如果编号重叠（例如两个 chip 都试图分配同一编号区间），函数返回错误，probe 回滚。

同时 `gdev->descs[i].gdev = gdev` 补全了 step ④ 中 `kcalloc` 分配的 `gpio_desc[]` 数组——之前每个 desc 的 `gdev` 还是 NULL。

解开 spinlock 后继续后续初始化。

**步骤 ⑦**：`of_gpiochip_add()` 内部完成两件事

```c
// gpiolib-of.c:1098
int of_gpiochip_add(struct gpio_chip *chip)
{
    if (!chip->of_xlate) {
        chip->of_gpio_n_cells = 2;
        chip->of_xlate = of_gpio_simple_xlate;
    }
    ret = of_gpiochip_add_pin_range(chip);   // gpio-ranges → range
    ret = of_gpiochip_scan_gpios(chip);       // GPIO hog 处理
    return ret;
}
```

`of_gpiochip_add_pin_range()` 读取 `gpio-ranges` 属性，创建 `pinctrl_gpio_range` 并挂入双链表。

**步骤 ⑨**：字符设备注册

```c
dev_set_name(&gdev->dev, "gpiochip%d", gdev->id);  // "gpiochip0"
cdev_init(&gdev->chrdev, &gpio_fileops);
cdev_add(&gdev->chrdev, gdev->dev.devt, 1);         // → /dev/gpiochip0
```

逐个 bank 注册完成后，`gpio_devices` 全局链表和 `/dev/` 的内容如下：

```
gpio_devices 全局链表：
  ├── gdev{id=0, base=512, descs[16], label="GPIOA"}  → /dev/gpiochip0
  ├── gdev{id=1, base=528, descs[16], label="GPIOB"}  → /dev/gpiochip1
  ├── gdev{id=2, base=544, descs[14], label="GPIOC"}  → /dev/gpiochip2
  ├── gdev{id=3, base=558, descs[16], label="GPIOD"}  → /dev/gpiochip3
  ├── gdev{id=4, base=574, descs[16], label="GPIOE"}  → /dev/gpiochip4
  ├── gdev{id=5, base=590, descs[16], label="GPIOF"}  → /dev/gpiochip5
  ├── gdev{id=6, base=606, descs[16], label="GPIOG"}  → /dev/gpiochip6
  ├── gdev{id=7, base=622, descs[12], label="GPIOH"}  → /dev/gpiochip7
  └── gdev{id=8, base=634, descs[12], label="GPIOI"}  → /dev/gpiochip8
```

注意 GPIOC 的 ngpio 是 14、GPIOH 和 GPIOI 是 12，而非 16——因为这些 bank 的部分引脚在当前封装上不存在（如 PH0/PH1、PI4~PI7），差异反映在 DTS 的 `ngpios` 属性中。

同时，每个 bank 注册后在 `pinctrl_dev` 上留下一个 `pinctrl_gpio_range` 条目：

```
pinctrl_dev->gpio_ranges（链表头）
  ├── range: GPIOA base=512, pin_base=0,  npins=16, gc=&gpioa_gpio_chip
  │     .node  → 在 pinctrl_dev->gpio_ranges 链表中
  │     .node2 → 在 gdev(GPIOA)->pin_ranges 链表中
  │
  ├── range: GPIOB base=528, pin_base=16, npins=16, gc=&gpiob_gpio_chip
  │     .node  → 在 pinctrl_dev->gpio_ranges 链表中
  │     .node2 → 在 gdev(GPIOB)->pin_ranges 链表中
  │
  ├── range: GPIOC base=544, pin_base=32, npins=14, gc=&gpioc_gpio_chip
  │
  └── ...
```

每个 bank 注册时 `dev_info` 打印一条日志：

```shell
[    0.123456] 44240000.pinctrl GPIOA bank added
[    0.123567] 44240000.pinctrl GPIOB bank added
...
[    0.124567] 44240000.pinctrl GPIOI bank added
```

### 3.4.10 错误处理

```c
// pinctrl-stm32.c:2166
err_register:
    for (i = 0; i < pctl->nbanks; i++) {
        struct stm32_gpio_bank *bank = &pctl->banks[i];
        gpiochip_remove(&bank->gpio_chip);
    }
    clk_bulk_disable_unprepare(banks, pctl->clks);
    return ret;
```

假设 GPIOI（第 9 个 bank）注册时内存不足，`gpiochip_add_data()` 返回 `-ENOMEM`，for_each 循环中断。此时 `pctl->nbanks = 8`（GPIOA~GPIOH 已成功注册），`err_register` 段遍历这 8 个 bank，依次调用 `gpiochip_remove()` 将它们从 `gpio_devices` 链表和字符设备系统中移除。然后 `clk_bulk_disable_unprepare(9, pctl->clks)` 关闭所有 bank 的时钟。probe 返回 `-ENOMEM`，内核将 `44240000.pinctrl` 标记为 `probe_failed`。

但 pinctrl 注册没有被回滚——`devm_pinctrl_register()` 申请的 `pinctrl_dev` 和 `pin_desc[]` 仍在内存中，它们的释放由 devm 框架在设备 remove 时自动处理。这意味着 pinctrl debugfs 目录和 `pinctrldev_list` 中的条目会残留在系统中，但事实上不会有任何 consumer 设备能成功引用它（因为 device link 会检测到供应商 probe 失败而阻止消费者 probe）。

---

## 3.5 初始化完成后的状态

### 3.5.1 Pinctrl 侧

```
pinctrldev_list 全局链表
  └── pinctrl_dev "44240000.pinctrl"
        ├── pin_desc_tree [0..175]（176 个 pin_desc，全部空闲）
        ├── gpio_ranges 链表（9 个 range，指向各 bank）
        ├── groups[0..175]（176 个 group，一 pin 一组）
        ├── states（空链表——尚未有任何 consumer 请求状态）
        ├── hogs（空——pinctrl 自身没有 pinctrl-0）
        └── debugfs: /sys/kernel/debug/pinctrl/44240000.pinctrl/
```

### 3.5.2 GPIO 侧

```
gpio_devices 全局链表
  ├── gdev0  (GPIOA, id=0)  → /dev/gpiochip0,  descs[0..15],  base=512
  ├── gdev1  (GPIOB, id=1)  → /dev/gpiochip1,  descs[0..15],  base=528
  ├── ...
  ├── gdev8  (GPIOI, id=8)  → /dev/gpiochip8,  descs[0..11],  base=634
  └── gdev9  (GPIOZ, id=9)  → /dev/gpiochip9,  descs[0..9],   base=646
```

### 3.5.3 用户态可见的接口

```shell
# debugfs：pinctrl 状态
$ cat /sys/kernel/debug/pinctrl/44240000.pinctrl/pinmux-pins
Pinmux settings per pin
Pin 0 (PA0):  (MUX UNCLAIMED) (GPIO UNCLAIMED)
Pin 1 (PA1):  (MUX UNCLAIMED) (GPIO UNCLAIMED)
...

# debugfs：GPIO 状态
$ cat /sys/kernel/debug/gpio
gpiochip0: GPIOs 0-15, parent: platform/44240000.pinctrl, GPIOA
gpiochip1: GPIOs 16-31, parent: platform/44240000.pinctrl, GPIOB
...

# 字符设备
$ ls /dev/gpiochip*
/dev/gpiochip0  /dev/gpiochip1  ...  /dev/gpiochip9
```

### 3.5.4 尚未做的事

初始化完成后，以下运行时行为**还没有发生**——它们将在后续 consumer 设备 probe 时触发：

1. **`pinctrl_select_state()` 未被调用**——只有 UART/MMC/I2C 等 consumer 设备 probe 时，`pinctrl_bind_pins()` 才会调用它
2. **`pin_desc` 的 `mux_owner` / `gpio_owner` 全为空**——还没有任何驱动请求引脚
3. **`gpio_desc` 的 `flags` 全为 0**——还没有任何驱动通过 `devm_gpiod_get()` 请求 GPIO

这些"运行时"行为将在下一篇 **04-情景分析** 中详细追踪。

---

## 3.6 完整初始化流程图

```
系统启动
  │
  ├── [core_initcall] pinctrl_init()
  │     └─ debugfs_create_dir("pinctrl")
  │
  ├── [subsys_initcall] gpiolib_sysfs_init()
  │     └─ class_register(&gpio_class)
  │
  └── [module_init] stm32mp257_pinctrl_init()
        └─ platform_driver_register(&stm32mp257_pinctrl_driver)
              │
              │  DTS 匹配 "st,stm32mp257-pinctrl"
              ▼
        stm32_pctl_probe(pdev)
          │
          ├── ① stm32_pctrl_get_irq_domain()
          │     层次化 irq domain（parent = EXTI/GIC）
          │
          ├── ② stm32_pctrl_create_pins_tab()
          │     根据 st,package 过滤 → 176 引脚
          │
          ├── ③ stm32_pctrl_build_state()
          │     groups[176] 建立
          │
          ├── ④ 填充 pinctrl_desc
          │     name="44240000.pinctrl", 三个 ops, link_consumers=true
          │     custom_params=stm32_gpio_bindings
          │
          ├── ④' devm_pinctrl_register()
          │     ├── pinctrl_init_controller()
          │     │     ├── 分配 pinctrl_dev
          │     │     ├── ops 完整性检查
          │     │     └── pinctrl_register_pins()
          │     │           └── 遍历 176 个 pins，为每个创建 pin_desc
          │     │                → radix_tree_insert(pin_desc_tree, pin#)
          │     │
          │     └── pinctrl_enable()
          │           ├── pinctrl_claim_hogs()（一般不做事）
          │           ├── list_add_tail(&pctldev->node, &pinctrldev_list)
          │           └── debugfs 目录创建
          │
          ├── ⑤ 枚举 GPIO bank 子节点
          │     取 9 个 bank 的 clock 和 reset
          │     clk_bulk_prepare_enable() 使能所有时钟
          │
          └── ⑥ for_each GPIO bank → stm32_gpiolib_register_bank()
                │
                ├── reset_control_deassert()（解除复位）
                ├── devm_ioremap_resource()（地址映射）
                ├── bank->gpio_chip = stm32_gpio_template（模板拷贝）
                ├── label/ngpio/fwnode/parent 个性化设置
                ├── names[] 引脚别名设置
                │
                └── gpiochip_add_data(&bank->gpio_chip, bank)
                      │
                      ├── 分配 gpio_device
                      ├── 分配 gpio_desc[ngpio]
                      │     └── 每个 desc->gdev = gdev
                      ├── gpiochip_find_base() 动态编号
                      ├── gpiodev_add_to_list() 全局链表
                      │
                      ├── of_gpiochip_add()
                      │     ├── of_gpiochip_add_pin_range()
                      │     │     └── gpio-ranges → pinctrl_gpio_range
                      │     │           ├── pinctrl_add_gpio_range()
                      │     │           └── list_add(&range->node2, ...)
                      │     │
                      │     └── of_gpiochip_scan_gpios()
                      │
                      ├── gpiochip_init_valid_mask()
                      ├── cdev_add("/dev/gpiochipN")
                      └── gpiochip_sysfs_register()
        │
        └── 返回 0 → 两个子系统全部就绪
```

---

## 总结

本文追踪了从内核启动到 GPIO 全量就绪的完整初始化路径，覆盖三个层面的代码：

**Core 层（pinctrl core + gpiolib core）**：两个 `initcall` 函数分别创建 pinctrl debugfs 根目录和 GPIO sysfs 类，为驱动的注册搭建好框架。这些 infra 层代码不依赖具体硬件，因此可以在早期阶段直接运行。

**驱动层（STM32 pinctrl + GPIO 合体驱动）**：`stm32_pctl_probe()` 是本文的核心——承担了"初始化一个 pinctrl + 批量注册多个 GPIO bank"的双重职责。6 个阶段的顺序至关重要：必须先有 IRQ domain（中断路由）→ 再有引脚表（package 过滤）→ 然后注册 pinctrl（提供 pinmux/pinconf 能力）→ 最后才能逐个注册 GPIO bank，因为 `gpio_ranges` 解析时需要 pinctrl_dev 已就绪。

**GPIO core 层（gpiolib）**：`gpiochip_add_data()` 经由宏展开为 `gpiochip_add_data_with_key()`，为每个 bank 分配 `gpio_device`、`gpio_desc[]` 数组、动态全局编号（从 `GPIO_DYNAMIC_BASE=512` 开始），并通过 `of_gpiochip_add_pin_range()` 建立 GPIO 编号 ←→ pinctrl pin 号的双向翻译表。9 个 bank 依次注册，整个过程在 spinlock 保护下串行完成。

**关键设计要点回顾：**
- **合体驱动**：一个 pinctrl 节点（`44240000.pinctrl`）驱动 9 个 GPIO bank，所有 bank 共享模板回调，差异仅在于 `bank->base` 寄存器基址
- **两次 probe**：GPIOZ 位于独立的安全岛节点，第二个 probe 用不同的 match_data 注册
- **双链表翻译**：`gpio_pin_range.range.node → pctldev->gpio_ranges` / `gpio_pin_range.node → gdev->pin_ranges`，同一个 range 两套索引
- **安全过滤**：SECCFGR 和 CIDCFGR 寄存器在初始化时决定哪些引脚对 Linux 可见，GPIOZ 因安全岛位址通常更受限

初始化完成后，pinctrl 和 GPIO 子系统已全部就绪。接下来的运行时行为——consumer 设备通过 `pinctrl_select_state()` 请求 pinmux 配置、通过 `gpiod_get()` 请求 GPIO 操作——将在下一篇 **04-情景分析** 中逐一展开。`
