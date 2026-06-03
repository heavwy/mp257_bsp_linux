# 04. 运行时情景分析

> 本文是 STM32MP257 Pinctrl & GPIO 深度分析系列的第 4 篇。
> 从实际使用场景出发，自上而下追踪关键路径从用户态（或内核驱动）直到硬件寄存器。
>
> **前置:** [03-SourceAnalysis.md](03-SourceAnalysis.md) — 初始化流程源码分析
> **下一篇:** [05-Client-Usage.md](05-Client-Usage.md) — GPIO 消费者驱动开发
>
> **字数：约 XXXX 词（含代码段）**
> **建议阅读时间：XX~XX 分钟**

---

## 4.1 概述

03 篇完成了从系统启动到 pinctrl & GPIO 子系统全部就绪的初始化跟踪。本篇考察**初始化之后发生的事情**——当内核中的 consumer 设备（UART、SDMMC、I2C 等）和用户态程序真正使用这些子系统时，经历了怎样的代码路径。

本篇选择两个最核心的场景：

| 场景 | 起点 | 终点 | 涉及的核心路径 |
|------|------|------|--------------|
| **场景一：Consumer probe 设定引脚复用** | consumer 设备 probe → `pinctrl_bind_pins()` | `stm32_pmx_set_mode()` 写 AFRL/AFRH | `pinctrl_select_state()` → `create_pinctrl()` → `dt_to_map()` → `add_setting()` → `set_mux` 回调 |
| **场景二：`/dev/gpiochip` 用户态 GPIO 控制** | `open("/dev/gpiochip0")` → `ioctl()` | `stm32_gpio_set/get()` 读写 BSRR/IDR | gpiolib cdev → gpiolib core → `gpio_chip` 回调 |

每个场景都按照**分层递进**的方式追踪：

```
场景起点
  │
  ├── ① 用户态 / 内核调用层     ← 从哪发起
  ├── ② 核心框架层               ← pinctrl core / gpiolib core
  ├── ③ 驱动回调层               ← stm32_xxx 回调函数
  └── ④ 硬件寄存器层             ← 操作了什么寄存器
```

---

## 4.2 场景一：Consumer 设备 probe 的引脚复用设定

### 4.2.1 问题

UART4 在 DTS 中定义了自己的引脚配置：

```dts
&uart4 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&uart4_pins_a>;
    pinctrl-1 = <&uart4_sleep_pins_a>;
    status = "okay";
};
```

当 UART4 驱动 probe 时，内核如何将这些 DTS 描述转化为 MODER 和 AFRL/AFRH 寄存器的具体值？

### 4.2.2 路径全貌

```
uart4 设备 probe
  │
  ├── driver → devm_platform_get_and_ioremap_resource()
  │
  ├── driver → devm_serial8250_register()  (或类似注册函数)
  │
  ├── ★ platform bus 自动调用 pinctrl_bind_pins(dev)
  │     └── devm_pinctrl_get(dev)
  │           └── pinctrl_get(dev)
  │                 └── create_pinctrl(dev, pctldev)
  │                       ├── 第一次遍历 pinctrl_maps: dt_to_map()
  │                       │     ├── pinctrl_dt_to_map()        ← 找 DTS pinctrl-N 属性
  │                       │     └── dt_to_map_one_config()     ← phandle 找 pinctrl 节点
  │                       │           └── dt_node_to_map()     ← STM32 回调创建 pinctrl_map[]
  │                       │                 └── MUX + CONFIGS 两个 map 条目
  │                       │                       → pinctrl_register_maps() → 挂入 pinctrl_maps 链表
  │                       │
  │                       └── 第二次遍历 pinctrl_maps: add_setting()
  │                             └── for_each_pin_map() → add_setting()
  │                                   └── pinmux_map_to_setting()  → 填充 settings 链表
  │                                       pinconf_map_to_setting() → 填充 settings 链表
  │
  └── ★ pinctrl_select_state(p->state)
        └── pinmux_enable_setting(setting)
              └── ops->set_mux(pctldev, setting->data.mux.func,
                               setting->data.mux.group)
                    └── stm32_pmx_set_mode(bank, pin, mode, alt)
                          └── writel(val, bank->base + AFRL/AFRH)
                              // 物理寄存器操作
```

### 4.2.3 第 1 阶段：pinctrl_bind_pins 触发

（待展开，详见后续写作）

### 4.2.4 第 2 阶段：create_pinctrl 两次遍历

（待展开，详见后续写作）

### 4.2.5 第 3 阶段：pinctrl_select_state 执行

（待展开，详见后续写作）

### 4.2.6 第 4 阶段：set_mux 回调写硬件寄存器

（待展开，详见后续写作）

### 4.2.7 ATK 板实例：UART4 的 pinmux 配置

（待展开，详见后续写作）

---

## 4.3 场景二：`/dev/gpiochip` 用户态 GPIO 控制

### 4.3.1 问题

用户在终端执行：

```bash
$ gpioset gpiochip0 5=1
$ gpioget gpiochip0 3
$ gpiomon gpiochip0 3
```

这些命令如何最终操作到 GPIO 寄存器的 BSRR、IDR、EXTI 等硬件？

### 4.3.2 路径全貌

```
用户态 (gpioset/gpioget/gpiomon 或自定义程序)
  │
  ├── open("/dev/gpiochip0")
  │     └── gpiolib cdev: gpio_open()
  │
  ├── ioctl(GPIO_GET_LINE_IOCTL)           ← 请求一行 GPIO
  │     └── gpiolib cdev: gpio_ioctl()
  │           └── gpiolib core: gpiod_request()
  │                 └── gpio_chip.request() → stm32_gpio_request()
  │                       └── 检查引脚状态 + pinctrl 引脚申请
  │
  ├── ioctl(GPIOHANDLE_SET_LINE_VALUES)    ← 设电平 (输出)
  │     └── gpiolib cdev: linehandle_ioctl()
  │           └── gpiolib core: gpiod_set_value_cansleep()
  │                 └── gpio_chip.set() → stm32_gpio_set()
  │                       └── writel(BIT(pin), bank->base + BSRR)
  │
  ├── ioctl(GPIO_GET_LINE_VALUES)          ← 读电平 (输入)
  │     └── gpiolib cdev: linehandle_ioctl()
  │           └── gpiolib core: gpiod_get_value_cansleep()
  │                 └── gpio_chip.get() → stm32_gpio_get()
  │                       └── readl(bank->base + IDR) & BIT(pin)
  │
  └── ioctl(GPIO_GET_LINE_EVENT_IOCTL)     ← 等待中断
        └── gpiolib cdev: 注册 event 监听
              └── gpiolib core: gpiod_to_irq()
                    └── gpio_chip.to_irq() → stm32_gpio_to_irq()
                          └── irq_create_fwspec_mapping(EXIT)

//
// 内核中各层接口
//
//   用户态 ioctl
//     → cdev 层 (gpio_ioctl / linehandle_ioctl)    — 内核态入口，校验 fd/权限
//       → gpiolib core (gpiod_get_value/set, gpiod_request) — desc 操作
//         → gpio_chip 回调 (.get/.set/.request/.to_irq)  — driver 注册
//           → STM32 硬件寄存器 (BSRR, IDR, MODER, AFRL/AFRH)
```

### 4.3.3 字符设备回顾

（待展开，详见后续写作）

### 4.3.4 ioctl 路由：从文件操作到 gpiolib

（待展开，详见后续写作）

### 4.3.5 gpio_chip 回调：写 BSRR、读 IDR

（待展开，详见后续写作）

### 4.3.6 中断：从 GPIO 到 EXTI 再到 GIC

（待展开，详见后续写作）

---

## 总结

（待写完各节后统一总结）

---

**参考文件索引**

| 路径 | 内容 |
|------|------|
| `drivers/pinctrl/core.c` | pinctrl core：create_pinctrl, pinctrl_select_state |
| `drivers/pinctrl/pinmux.c` | pinmux 核心逻辑 |
| `drivers/pinctrl/stm32/pinctrl-stm32.c` | STM32 驱动：set_mux, gpio 回调 |
| `drivers/gpio/gpiolib.c` | gpiolib core：gpiod_get/set/request |
| `drivers/gpio/gpiolib-cdev.c` | gpiolib 字符设备：ioctl 路由 |
| `drivers/gpio/gpiolib-of.c` | OF 解析：gpio-ranges, hog |
