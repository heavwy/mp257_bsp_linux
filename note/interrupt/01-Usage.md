# 01. 中断子系统使用方法

> 看完就能在 STM32MP257 上配中断 DTS、调注册 API、用调试接口排查问题。不深入设计原理。
>
> **字数**：字数 **13368 字** · **行数**：1,683 · **建议阅读时间**：40 分钟

---

## 1.1 DTS 配置

### 场景引导：你的中断信号走哪条路

STM32MP257 上，一个外设的中断信号到达 CPU 有三条可能的路径。DTS 怎么写取决于你的设备用哪条：

| 场景 | 信号路径 | 适用设备 | DTS 写法 |
|------|---------|---------|---------|
| **A：直连 GIC** | 设备 → GIC | SoC 内部外设（USART、I2C、SDMMC、SPI、TIM） | `interrupts = <GIC_SPI N type>` |
| **B：经 EXTI 转发** | 设备 → EXTI → GIC | 需要低功耗唤醒、或中断输出连到 EXTI 输入的外设 | `interrupts-extended = <&exti1 N type>` |
| **C：经 GPIO 再经 EXTI** | 设备 → GPIO bank → EXTI → GIC | 外部设备的中断线连到了 SoC 的某个 GPIO pin 上 | `interrupt-parent = <&gpioX>; interrupts = <pin type>` |

三种场景最终都落到 GIC。区别在于**中断信号从哪个节点进入这条链**——是直接进 GIC、先经 EXTI、还是先经 GPIO bank。

---

### 1.1.1 场景 A：设备直连 GIC

大部分 SoC 内部外设（USART、I2C、SDMMC、SPI、TIM）的中断输出直接连到 GIC 的 SPI 输入线上。SoC dtsi 中已经写好了：

```dts
/* stm32mp251.dtsi — SoC 级默认定义 */
sdmmc1: mmc@48220000 {
    interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>;
};

usart1: serial@4c000000 {
    interrupts = <GIC_SPI 45 IRQ_TYPE_LEVEL_HIGH>;
};

i2c3: i2c@4c008000 {
    interrupts = <GIC_SPI 73 IRQ_TYPE_LEVEL_HIGH>;
};
```

板级 dts 中不需要重复写 `interrupts`，除非你想覆盖 SoC 的默认配置。

**三个 cell 的含义：**

```dts
interrupts = <GIC_SPI   42   IRQ_TYPE_LEVEL_HIGH>;
              ───┬──   ─┬─     ───────┬────────
                 │      │             └─ cell 2：触发类型
                 │      └─ cell 1：中断编号（SPI 编号从 0 开始）
                 └─ cell 0：中断类型（GIC_SPI=0 / GIC_PPI=1）
```

**cell 0**——中断类型。这是 GIC 在 ARM 架构层面的分类，两种类型的路由方式完全不同：

- **`GIC_SPI`（Shared Peripheral Interrupt，共享外设中断）**：外设的中断输出线直接连到 GIC 的 SPI 输入。GIC 可以把这个中断**路由到任意一个 CPU**，板级代码（U-Boot/ATF）可以配置亲和性。SoC 内部几乎所有外设（SDMMC、USART、I2C、TIM 等）都走这条路。DTS 中的 SPI 编号范围是 0~987，对应 GIC 硬件 ID 32~1019。

- **`GIC_PPI`（Private Peripheral Interrupt，私有外设中断）**：每个 CPU 核独享一条中断线，不参与路由。典型例子是 `arch_timer`——每个 CPU 都有自己的通用定时器，中断直接发给绑定的那个核，不需要 GIC 做分发。PPI 编号固定 0~15，与 GIC 硬件 ID 一致。

| 宏 | 值 | 类型（中文） | 路由方式 | 典型设备 |
|----|----|-------------|---------|---------|
| `GIC_SPI` | 0 | 共享外设中断 | GIC 可路由到任意 CPU | SDMMC、USART、I2C、TIM |
| `GIC_PPI` | 1 | 私有外设中断 | 直接发给绑定的 CPU，不路由 | arch_timer |

**cell 1**——中断编号。做驱动开发只需要记住一句话：

> DTS 里 SPI 从 0 编号，GIC 硬件从 32 编号，差 32。

| 在哪儿看 | 看到的值 | 说明 |
|---------|---------|------|
| DTS 里写 | `GIC_SPI 42` | 你写的是"第 42 个 SPI" |
| GIC 硬件寄存器 | bit **74** | 硬件管这个中断叫 ID 74（= 42 + 32） |

为什么差 32？因为 GIC 把 ID 0~31 固定给了 SGI（核间中断）和 PPI（每核私有中断），SPI 从 32 开始排。DTS 约定 SPI 编号从 0 写起，内核在 `gic_irq_domain_translate()` 里自动加 32。

PPI 没有这个偏移：DTS 写 `GIC_PPI 13`，硬件 ID 就是 13，完全一致。

**cell 2**——触发类型（`include/dt-bindings/interrupt-controller/irq.h`）：

| 宏 | 值 | 含义 | 典型用途 |
|----|----|------|---------|
| `IRQ_TYPE_EDGE_RISING` | 1 | 上升沿触发 | USB、ETH 某些情况 |
| `IRQ_TYPE_EDGE_FALLING` | 2 | 下降沿触发 | 少见 |
| `IRQ_TYPE_EDGE_BOTH` | 3 | 双沿触发 | GPIO 按键 |
| `IRQ_TYPE_LEVEL_HIGH` | 4 | 高电平触发 | 大部分 SoC 外设 |
| `IRQ_TYPE_LEVEL_LOW` | 8 | 低电平触发 | 唤醒源 |

STM32MP257 上 SoC 内部外设的 `interrupts` 几乎全部是 `IRQ_TYPE_LEVEL_HIGH`。这里需要理解电平触发在处理过程中是怎么避免"反复 irq"的，它依赖 GIC 内部的状态机：

```
初始状态：中断线低 → GIC 认为 Inactive
          │
事件发生：外设拉高中断线 → GIC 检测到电平变高
          │
          ▼
GIC 状态变为 Pending → 向 CPU 发出 IRQ
          │
CPU 读取 GIC_ICC_IAR（Interrupt Acknowledge Register）
          │  ┌──────────────────────────────────────┐
          ├─→│ GIC 把该中断状态标记为 Active          │
          │  │ 此时 GIC 不会因为同一个中断线还高着    │
          │  │ 就重复向 CPU 发 IRQ（相当于"屏蔽"了） │
          │  └──────────────────────────────────────┘
          ▼
CPU 执行 handler：
  1. 读外设状态寄存器，确定中断原因
  2. 处理数据（如读 RX FIFO）
  3. 写外设寄存器清除中断标志 → 外设检测到标志被清 → 拉低中断线
          ▼
CPU 写 GIC_ICC_EOIR（End Of Interrupt Register）
          │  ┌──────────────────────────────────────┐
          ├─→│ GIC 把该中断状态变为 Inactive         │
          │  │ 检查中断线：                          │
          │  │  ● 已拉低 → 结束，无事发生            │
          │  │  ● 还高着 → GIC 立即再设 Pending      │
          │  │     → CPU 再进一次 handler             │
          │  │     （这就是"中断风暴"，说明驱动忘了  │
          │  │      清设备端的中断标志）              │
          │  └──────────────────────────────────────┘
```

所以电平触发本质上是 **GIC 通过 Active 状态自动屏蔽了该中断线**，不需要 CPU 显式去 mask/unmask。驱动开发者的责任就是确保 handler 中**清掉了外设的中断标志**，否则 EOI 后中断线还高着就会无限循环。

注：驱动端怎么拿中断号、注册 handler，见 1.2 节。怎么从 `/proc/interrupts` 确认中断生效，见 1.4 节。

---

### 1.1.2 场景 B：设备经 EXTI 转发到 GIC

EXTI 控制器像是一个"中断转发站"。它检测输入信号的边沿，然后把信号转发到 GIC 的某个 SPI 输入。

**什么时候用这个路径：**

- 设备的中断输出线物理上连到了 EXTI 的输入引脚（而不是直连 GIC）
- 外设需要在系统 suspend 时唤醒 CPU——EXTI 可以在低功耗模式下保持检测能力
- STM32MP2 上 GPIO 中断默认就是走这个路径（见场景 C）

**怎么写——两种写法等价，推荐第一种：**

写法一（设 interrupt-parent，清晰）：
```dts
my_device {
    interrupt-parent = <&exti1>;          /* 指定中断控制器为 EXTI */
    interrupts = <47 IRQ_TYPE_LEVEL_HIGH>; /* event 号 47 */
};
```

写法二（interrupts-extended，entry 自带控制器 phandle）：
```dts
my_device {
    interrupts-extended = <&exti1 47 IRQ_TYPE_LEVEL_HIGH>;
};
```

两种写法效果完全一样。写法一更常见，写法二还有一个场景：**一个设备有多个中断源、分别来自不同的中断控制器**——此时每个 `interrupts-extended` entry 可以指定不同的 phandle：

```dts
my_device {
    interrupts-extended =
        <&intc  GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>,   /* 功能中断 → GIC */
        <&exti1 47     IRQ_TYPE_EDGE_RISING>;       /* 唤醒中断 → EXTI */
};
```

这要求设备物理上有两根独立的中断输出线分别接到不同的中断控制器（例如某些桥接芯片或以太网 PHY），普通外设不需要。

**EXTI event 到 GIC SPI 的映射**

EXTI 有 96 个 event 输入（EXTI_0 ~ EXTI_95），每个 event 在 SoC 内部有**一根独立的逻辑信号线**接到 GIC 的一个 SPI 输入。这块映射是 SoC 设计固定的，不需要软件干预。

EXTI 节点通过自己的 `interrupts-extended` 属性声明这 96 根线的接法——每个 entry 是一个 `<&intc GIC_SPI N type>` 三元组，entry 的索引位置对应 EXTI event 号：

```dts
exti1: interrupt-controller@44220000 {
    interrupts-extended =
        /* EXTI_0  */  <&intc GIC_SPI 268 IRQ_TYPE_LEVEL_HIGH>,
        /* EXTI_1  */  <&intc GIC_SPI 269 IRQ_TYPE_LEVEL_HIGH>,
        /* EXTI_2  */  <&intc GIC_SPI 270 IRQ_TYPE_LEVEL_HIGH>,
        /* EXTI_3  */  <&intc GIC_SPI 271 IRQ_TYPE_LEVEL_HIGH>,
        /* EXTI_4  */  <&intc GIC_SPI 272 IRQ_TYPE_LEVEL_HIGH>,
        /* EXTI_5  */  <&intc GIC_SPI 273 IRQ_TYPE_LEVEL_HIGH>,
        /* EXTI_6  */  <&intc GIC_SPI 274 IRQ_TYPE_LEVEL_HIGH>,
        /* EXTI_7  */  <&intc GIC_SPI 275 IRQ_TYPE_LEVEL_HIGH>,
        /* EXTI_8  */  <&intc GIC_SPI 276 IRQ_TYPE_LEVEL_HIGH>,
        /* EXTI_9  */  <&intc GIC_SPI 277 IRQ_TYPE_LEVEL_HIGH>,
        /* EXTI_10 */  <&intc GIC_SPI 278 IRQ_TYPE_LEVEL_HIGH>,
        ...
        /* EXTI_20 */  <0>,                         /* 未使用 */
        ...
    ;
};
```

每个 entry 包含 `<phandle  cell0  cell1  cell2>`。第一项是中断控制器 phandle（这里 `&intc` 指向 GIC），后面三个 cell 是 GIC 要求的 `#interrupt-cells = <3>`（中断类型、编号、触发类型）。entry 总数 < 96，未使用的 event 填 `<0>`。EXTI 驱动在 probe 时遍历这些 entry，建立 event 号到 GIC SPI 的内部查找表。

---

### 1.1.3 场景 C：设备经 GPIO 再经 EXTI 到 GIC

这是最灵活、也是最常见的外部设备中断方式。外设的中断输出线连到 SoC 的某个 GPIO pin 上，GPIO bank 检测到电平变化后转发给 EXTI，再由 EXTI 送到 GIC。

**什么时候用：**

- 外部设备（按键、触控屏、传感器）的中断输出连到了 GPIO pin
- GPIO bank 在 DTS 中声明了 `interrupt-controller`

**ATK 板上的实际例子（PH5 按键）：**

```dts
/* stm32mp257d-atk.dts */
gpio-keys {
    compatible = "gpio-keys";
    button-user {
        label = "User-Key";
        linux,code = <BTN_1>;
        gpios = <&gpioh 5 GPIO_ACTIVE_HIGH>;
        gpio-key,wakeup;
    };
};
```

这里没有出现 `interrupts`——因为 `gpio-keys` 驱动自己调 `gpiod_to_irq()` 从 GPIO 描述符转成了中断号。不会被别的设备抢走这个中断，因为 `gpios = <&gpioh 5 ...>` 已经把 PH5 这个 pin claim 掉了，同一个 pin 不可能同时被两个设备使用。等效的 DTS 写法（如果设备不是 gpio-keys 而是一个普通外设）：

```dts
touch@14 {
    interrupt-parent = <&gpioh>;          /* GPIOH 是中断控制器 */
    interrupts = <5 IRQ_TYPE_EDGE_RISING>; /* pin 5 (PH5)，上升沿触发 */
};
```

**为什么 gpioh 能当 interrupt-controller？**

```dts
/* stm32mp251.dtsi */
gpioh: gpio@442b0000 {
    gpio-controller;        /* 这是 GPIO 控制器 */
    #gpio-cells = <2>;      /* GPIO 引用用 2 cell：pin + flags */
    interrupt-controller;   /* 这也是中断控制器 */
    #interrupt-cells = <2>; /* 中断引用用 2 cell：pin + 触发类型 */
    st,bank-name = "GPIOH";
};
```

`interrupt-controller` 这个属性的意思是：这个 GPIO bank 内部有中断检测逻辑（输入电平变化时产生中断信号），可以作为下级设备的中断控制器。

**完整路径：信号从 PH5 到 CPU**

以 PH5 按键为例，中断信号经过：

```
GPIO pin → GPIO bank → 映射到 EXTI event → EXTI 映射到 GIC SPI → CPU
```

关键映射关系就一个：**GPIO pin N 固定映射到 EXTI1 event N**。PH5 是 pin 5，所以走 EXTI1 event 5。

但有个问题：所有 bank 的 pin 5（GPA5、GPB5、...、GPH5）都只能走 EXTI1 event 5。同一时刻只能有一个 bank 的 pin 5 做中断源。查看下面"冲突检测"小节了解如何仲裁。

EXTI1 event 到 GIC SPI 的映射在 EXTI1 节点中固定：

| EXTI1 event | 信号来自 | GIC SPI（DTS 编号） | GIC 硬件 ID |
|:---:|:---|:---:|:---:|
| 5 | 当前启用中断的 bank 的 pin 5 | 273 | 305 |
| 4 | 当前启用中断的 bank 的 pin 4 | 272 | 304 |
| 6 | 当前启用中断的 bank 的 pin 6 | 274 | 306 |

完整路径：**PH5 电平变化 → GPIOH 检测到 → pin 5 映射到 EXTI1 event 5 → EXTI1 查表 event 5=GIC SPI 273 → CPU 收到中断**。

**DTS 只需要写 pin 号：** `interrupts = <5>`。EXTI 驱动内部负责 GPIO bank 到 event 的仲裁。

#### 冲突检测：同一个 pin 号被不同 bank 抢中断

所有 bank 的 pin 5（GPA5、GPB5、...、GPH5）都只能走 EXTI1 event 5，但任一时刻只能有一个 bank 的 pin 5 作为中断源。如果 GPA5 已经启用了中断，PH5 又来申请，会发生什么？

EXTI 驱动自己维护了 16 个 event 的占用表（`drivers/irqchip/irq-stm32mp-exti.c`）：

```c
/* 每个 event 对应哪路 GPIO 的 EXTI_CR 寄存器 */
DECLARE_BITMAP(gpio_mux_used, 16);   /* event 是否已被占用 */
u8 gpio_mux_pos[16];                 /* 被哪个 bank 占用 */
```

申请时，会写 EXTI_CR 寄存器选择对应的 GPIO bank，同时记录当前占用：

```c
stm32mp_exti_gpio_bank_alloc() {
    /* 如果 event N 已被占用且不是同一个 bank → 冲突 */
    if (gpio_mux_used[N] && gpio_mux_pos[N] != current_bank)
        return -EBUSY;

    /* 写入 EXTI_CR 寄存器，选择 GPIO bank 连到 event N */
    writel_relaxed(cr | (gpio_bank << EXTI_CR_SHIFT(hwirq)), base + EXTI_CR(hwirq));

    set_bit(hwirq, gpio_mux_used);      /* 标记 event 已用 */
    gpio_mux_pos[hwirq] = gpio_bank;    /* 记录占用者 */
}
```

所以你配 DTS 时不需担心冲突——代码里有保护。如果两个设备错误地配了不同 bank 的同一个 pin 号，后申请的那个会收到 `-EBUSY`。

---

### 1.1.4 参考：三个中断控制器节点的完整定义

开发中不需要写这些（SoC dtsi 中已定义），但理解了每个节点才能看懂路径：

```dts
/* ① GIC 节点 — 根中断控制器 */
intc: interrupt-controller@4ac00000 {
    compatible = "st,stm32mp2-cortex-a7-gic", "arm,cortex-a7-gic";
    #interrupt-cells = <3>;           /* cell 格式：类型(SPI/PPI) + 编号 + 触发类型 */
    interrupt-controller;
    interrupt-parent = <&intc>;       /* GIC 是根，interrupt-parent 指向自己 */
    reg = <0x0 0x4ac10000 0x0 0x1000>,   /* Distributor 寄存器基址 */
          <0x0 0x4ac20000 0x0 0x2000>,   /* CPU Interface 基址 */
          <0x0 0x4ac40000 0x0 0x2000>,   /* 虚拟 CPU 接口 */
          <0x0 0x4ac60000 0x0 0x2000>;   /* 虚拟控制 */
};

/* ② EXTI 节点 — 转发层 */
exti1: interrupt-controller@44220000 {
    compatible = "st,stm32mp1-exti";
    #interrupt-cells = <2>;           /* cell 格式：event 号 + 触发类型 */
    interrupt-controller;
    reg = <0x44220000 0x400>;
    /* 96 个 EXTI event 到 GIC SPI 的映射表 */
    interrupts-extended =
        <&intc GIC_SPI 268 IRQ_TYPE_LEVEL_HIGH>,   /* EXTI_0 */
        <&intc GIC_SPI 269 IRQ_TYPE_LEVEL_HIGH>,   /* EXTI_1 */
        ...
        <&intc GIC_SPI 273 IRQ_TYPE_LEVEL_HIGH>,   /* EXTI_5 — GPIO pin 5 */
        ...
    ;
};

/* ③ GPIOH bank — 第三层中断控制器 */
gpioh: gpio@442b0000 {
    gpio-controller;
    #gpio-cells = <2>;
    interrupt-controller;             /* 作为中断控制器 */
    #interrupt-cells = <2>;           /* cell 格式：pin 号 + 触发类型 */
    reg = <0x70000 0x400>;
    st,bank-name = "GPIOH";
};
```

**层级关系：**

```
intc (GIC, root domain, #interrupt-cells = 3)
  ↑ parent domain
exti1 (EXTI, child domain, #interrupt-cells = 2)
  ↑ parent domain
gpioh (GPIO bank, #interrupt-cells = 2)
```

每个下级节点通过 parent domain 指向上级，最终到达 GIC。`#interrupt-cells` 的值向下传递——消费者节点（如 `touch@14`）写 2 cell 是因为它直接连 GPIOH，GPIOH 的 `#interrupt-cells = <2>`。

---

### 1.1.5 DTS 配置速查

| 设备类型 | DTS 写法 | 说明 |
|---------|---------|------|
| SoC 内部外设（SDMMC/USART/I2C） | SoC dtsi 已配好，板级不用改 | 默认走场景 A，直连 GIC |
| 外部设备中断到 GPIO pin | `interrupt-parent = <&gpioX>;`<br>`interrupts = <pin type>;` | 走场景 C，GPIO→EXTI→GIC |
| 设备经 EXTI 不经过 GPIO | `interrupt-parent = <&exti1>;`<br>`interrupts = <event type>;` | 走场景 B，设备直接连 EXTI |
| GPIO 按键/中断唤醒 | `gpios = <&gpioX pin flags>;` | 驱动内调 `gpiod_to_irq()` 转中断号 |

---

## 1.2 内核 API

### 1.2.0 使用前须知：中断上下文

中断 handler 运行在 hardirq context，跟进程 context 有本质区别——不能睡眠、不能调可能 sleep 的 API。以下表格先贴在这里，后续每个会反复提到：

**各 API 能否在中断 context 中调：**

| 函数 | 中断 context | 进程 context |
|------|------------|------------|
| `platform_get_irq` | ❌（可能 sleep） | ✅ |
| `gpiod_to_irq` | ❌（可能 sleep） | ✅ |
| `devm_request_irq` | ❌（分配内存） | ✅ |
| `devm_request_threaded_irq` | ❌（分配内存） | ✅ |
| `disable_irq` | ❌（等待同步） | ✅ |
| `disable_irq_nosync` | ✅ | ✅ |
| `enable_irq` | ✅ | ✅ |
| `synchronize_irq` | ❌（可能 sleep） | ✅ |
| `irq_set_irq_wake` | ❌ | ✅ |

### 1.2.1 获取中断号

#### platform_get_irq — 从 DTS 拿中断号

**头文件：** `<linux/platform_device.h>`

```c
int platform_get_irq(struct platform_device *pdev, unsigned int index);
```

| 参数 | 说明 |
|------|------|
| `pdev` | probe 传入的 `struct platform_device *` |
| `index` | `interrupts` 属性中的第几个（从 0 开始） |

**返回值：**
- `> 0`：有效的 Linux IRQ number（virq），可直接传给 `request_irq`
- `< 0`：错误码（`-ENXIO` 无中断、`-EPROBE_DEFER` 中断控制器未 probe）

**内部路径：**

```
platform_get_irq(pdev, 0)
  → of_irq_get(dev->of_node, 0)                ← 解析 DTS 的 interrupt 属性
    → irq_create_of_mapping(&of_irq)            ← 找中断控制器
      → irq_create_fwspec_mapping(&fwspec)      ← 在 irq_domain 中分配映射
        → irq_domain_alloc_irqs(domain, 1, hwirq) ← 分配一个 virq
        → 返回 virq                             ← probe 拿到的值
```

**注意事项：**
- 必须在进程 context 调（可能 sleep 等待分配 virq）
- 多中断设备：`platform_get_irq(pdev, 0)` 拿第 0 个，`platform_get_irq(pdev, 1)` 拿第 1 个

#### gpiod_to_irq — 从 GPIO 描述符拿中断号

**头文件：** `<linux/gpio/consumer.h>`

```c
int gpiod_to_irq(const struct gpio_desc *desc);
```

| 参数 | 说明 |
|------|------|
| `desc` | `devm_gpiod_get` 返回的 GPIO 描述符 |

**返回值：**
- `> 0`：有效的 virq
- `-ENXIO`：该 GPIO 不支持中断（GPIO 控制器没有实现 `to_irq` 回调）
- `-EPROBE_DEFER`：GPIO 的 irq_chip 还没准备好

**说明：** 从 GPIO 描述符反查对应的中断号。跟 `platform_get_irq` 的区别：

| 对比项 | `platform_get_irq` | `gpiod_to_irq` |
|--------|-------------------|----------------|
| DTS 信息来源 | `interrupts` 属性 | `gpios` 属性 |
| 适用场景 | 设备有独立中断线 | 设备中断复用在 GPIO pin 上 |

最终返回的都是 virq，行为没有区别。

---

### 1.2.2 注册中断 handler

#### devm_request_irq — 注册标准 handler（纯顶半部）

**头文件：** `<linux/interrupt.h>`

```c
int devm_request_irq(struct device *dev, unsigned int irq,
                     irq_handler_t handler,
                     unsigned long irqflags,
                     const char *devname, void *dev_id);
```

`devm_request_irq` 是 `request_threaded_irq` 的托管封装，`thread_fn = NULL` 表示纯顶半部模式：

```c
static inline int devm_request_irq(struct device *dev, unsigned int irq,
                                   irq_handler_t handler,
                                   unsigned long irqflags,
                                   const char *devname, void *dev_id)
{
    return devm_request_threaded_irq(dev, irq, handler, NULL, irqflags,
                                     devname, dev_id);
}
```

**参数说明：**

| 参数 | 说明 | 注意事项 |
|------|------|---------|
| `irq` | `platform_get_irq` 或 `gpiod_to_irq` 返回的 virq | 不能是负数，不能是 0 |
| `handler` | 顶半部函数指针 | 在 hardirq context 执行，不能睡眠 |
| `irqflags` | 触发类型 + 控制标志 | 见下表 |
| `devname` | 显示在 `/proc/interrupts` 最后一列 | 排查时反查设备用 |
| `dev_id` | 传给 handler 的 cookie | 共享中断时必须传非 NULL |

**irqflags 常用组合：**

| flags | 值 | 用途 |
|-------|----|------|
| `IRQF_TRIGGER_HIGH` | 0x04 | 高电平触发 |
| `IRQF_TRIGGER_LOW` | 0x08 | 低电平触发 |
| `IRQF_TRIGGER_RISING` | 0x01 | 上升沿触发 |
| `IRQF_TRIGGER_FALLING` | 0x02 | 下降沿触发 |
| `IRQF_SHARED` | 0x80 | 共享中断线 |
| `IRQF_ONESHOT` | 0x2000 | 线程化 IRQ 时用（见下文） |
| `IRQF_NO_SUSPEND` | 0x4000 | suspend 时保持使能（唤醒源） |

> **⚠️ 触发类型必须和硬件一致。** 设备的中断输出是高电平有效，你就不能用 `IRQF_TRIGGER_RISING`——边沿触发等一个上升沿，但电平信号一直高着，不会产生边沿。结果是中断要么完全不触发，要么反复触发风暴。
>
> **⚠️ DTS 中已经配了触发类型，request_irq 还写不写？** 多数情况不需要写 `IRQF_TRIGGER_*` flag，因为 `platform_get_irq` 从 DTS 读到的 cell 2 已经设好了。如果你 `devm_request_irq` 又传一次 `IRQF_TRIGGER_*`，内核会检查是否一致，不一致时打印 warning 甚至返回 `-EBUSY`。标准写法：**DTS 配了触发类型 → irqflags 只传 `IRQF_SHARED` 等控制类标志**（不传 `IRQF_TRIGGER_*`）。

**devm 特性：** 保证 probe 失败或设备移除时自动 `free_irq`，不需要写 remove 回调。

**handler 返回值规则：**

```c
static irqreturn_t my_handler(int irq, void *dev_id)
{
    if (!my_device_triggered(dev_id))
        return IRQ_NONE;      /* 0：不是我的中断，共享中断时继续调下一个 handler */

    /* ... 处理 ... */
    return IRQ_HANDLED;       /* 1：已处理 */
    return IRQ_WAKE_THREAD;   /* 2：唤醒 thread_fn（仅线程化 IRQ）*/
}
```

**`IRQ_HANDLED` / `IRQ_NONE` 不能乱写：**
- 该写 `IRQ_HANDLED` 写了 `IRQ_NONE`：内核认为虚假中断，`note_interrupt()` 累加计数，超过阈值后打印 `nobody cared` 并 `disable_irq()`，中断永远不再触发。
- 该写 `IRQ_NONE` 写了 `IRQ_HANDLED`：共享中断中另一个设备产生的中断被你的 handler "消费"了，对方永远收不到。

#### devm_request_threaded_irq — 注册线程化 handler（顶半部 + 底半部线程）

**头文件：** `<linux/interrupt.h>`

```c
int devm_request_threaded_irq(struct device *dev, unsigned int irq,
                              irq_handler_t handler,
                              irq_handler_t thread_fn,
                              unsigned long irqflags,
                              const char *devname, void *dev_id);
```

| 参数 | 说明 |
|------|------|
| `handler` | 顶半部，执行在 hardirq context，不能 sleep |
| `thread_fn` | 底半部线程，执行在进程 context，可以 sleep |
| `irqflags` | 触发类型 + `IRQF_ONESHOT`（见下方说明） |

**三种工作模式：**

| handler | thread_fn | 行为 |
|---------|-----------|------|
| `my_handler` | `my_thread_fn` | 顶半部确认中断源，返回 `IRQ_WAKE_THREAD` 唤醒线程 |
| `my_handler` | `NULL` | 等价于 `devm_request_irq`，纯顶半部 |
| `NULL` | `my_thread_fn` | 内核安装默认 handler，直接返回 `IRQ_WAKE_THREAD`，纯线程化 |

NULL handler 时内核安装默认实现，不检查直接唤醒线程：

```c
static irqreturn_t irq_default_primary_handler(int irq, void *dev_id)
{
    return IRQ_WAKE_THREAD;
}
```

**什么情况用 NULL handler：** 设备独享中断线（没有 `IRQF_SHARED`），所有工作在 `thread_fn` 里完成，不需要在顶半部确认。

**什么情况需要自定义 handler + thread_fn：**
- 共享中断（`IRQF_SHARED`），需要在顶半部先确认中断是否属于自己
- 顶半部需要从硬件读少量数据保存下来再唤醒线程

**IRQF_ONESHOT 的作用：**

一句话：**电平触发 + handler = NULL（纯线程化），必须加 IRQF_ONESHOT，否则死机。**

#### 为什么电平触发必须加 ONESHOT？

电平触发时，只要外设中断线上电平有效，GIC 就一直看到中断：

```
不加 ONESHOT：
  中断触发 → handler → IRQ_WAKE_THREAD
    → 内核 unmask                     ← handler 返回，GIC 立即释放中断线
    → 电平还高着！                     ← thread_fn 还没跑，没人清设备中断
    → GIC 又触发中断
    → 无限循环：handler → unmask → handler → unmask → ...
    → thread_fn 永远没机会执行           ← 系统死机
```

加了 ONESHOT 后，handler 返回时内核保持 mask，给 thread_fn 时间清中断：

```
加 ONESHOT：
  中断触发 → handler → IRQ_WAKE_THREAD
    → 内核保持 interrupt MASKED       ← ONESHOT 保证
    → thread_fn 跑，读/写外设寄存器     ← 可以 sleep
    → 清中断标志，外设拉低电平
    → thread_fn 返回
    → 内核 unmask                     ← 电平已经低了，安全
    → 下一次中断正常进来
```

#### 为什么边沿触发不需要 ONESHOT？

边沿触发只在跳变瞬间触发一次，跳完就过去了。handler 返回后 unmask，电平已经恢复正常，不会立即再触发。

#### 什么时候可以不加 ONESHOT？

- **边沿触发**的中断不需要
- **你在自定义 handler 中已经关了设备中断**（如写 INT_EN=0），从设备侧就不会再触发了

但稳妥做法仍然加上 ONESHOT——设备级屏蔽失效时 GIC 侧还有一层保护。

#### 共享中断能用 ONESHOT 吗？

共享中断（IRQF_SHARED）加 IRQF_ONESHOT 会导致：你的 thread_fn 跑着，整条中断线被 mask，**共享这条线的其他设备也收不到中断**。所以共享中断尽量别用纯线程化 handler，或者自己在 handler 中做设备级 mask/unmask。

---

### 1.2.3 中断下半部机制

Linux 中断处理分上半部（hardirq）和下半部。上半部做最紧急的事（读硬件状态、清中断标志），耗时操作推迟到下半部。

有四种推迟机制：

```
                短延迟               长延迟
  softirq   ── 软中断          ❌ 不能 sleep
  tasklet   ── 小任务          ❌ 不能 sleep
  threaded  ── 线程化 IRQ      ✅ 可以 sleep
  workqueue ── 工作队列        ✅ 可以 sleep
```

#### softirq — 最轻量，驱动不直接调

内核内部用，驱动不直接注册。典型使用者：网络栈（`NET_TX_SOFTIRQ`）、块设备层（`BLOCK_SOFTIRQ`）、RCU。

#### tasklet — softirq 驱动的轻量推迟机制

构建在 softirq 之上（内核为 tasklet 保留了 `TASKLET_SOFTIRQ`），驱动可以直接用：

> **注意：** v6.6 中 tasklet 回调签名已改为 `void callback(struct tasklet_struct *t)`，通过 `from_tasklet()` 宏反拿自己的结构体。旧版 API（`DECLARE_TASKLET_OLD`、`tasklet_init`）仍然存在但不建议在新代码中使用。

```c
#include <linux/interrupt.h>

struct my_dev {
    struct tasklet_struct irq_tasklet;  /* tasklet 嵌入到驱动结构体中 */
    void __iomem *base;
    /* ... 其他成员 */
};

/* 新 API：回调参数传 tasklet_struct 指针，用 from_tasklet 反拿宿主结构体 */
void my_tasklet_fn(struct tasklet_struct *t)
{
    struct my_dev *dev = from_tasklet(dev, t, irq_tasklet);
    /* dev 就是包含这个 tasklet 的 my_dev 结构体，base 等成员都可以用 */
    /* 处理中断数据 ... */
}

/* 方式一：静态声明（编译时初始化）—— 2 个参数，无 data */
DECLARE_TASKLET(my_tasklet, my_tasklet_fn);

/* 方式二：运行时初始化（probe 中调）*/
tasklet_setup(t, my_tasklet_fn);     /* 新 API，只有 2 个参数 */

/* 顶半部中触发 */
tasklet_schedule(&my_tasklet);    /* 正常优先级 */
tasklet_hi_schedule(t);           /* 高优先级 */
```

| API | 说明 |
|-----|------|
| `DECLARE_TASKLET(name, callback)` | 静态声明（新 API，回调参数为 `struct tasklet_struct *`） |
| `tasklet_setup(t, callback)` | 运行时初始化（新 API） |
| `tasklet_schedule(&t)` | 触发 tasklet |
| `tasklet_hi_schedule(&t)` | 高优先级触发 |
| `tasklet_kill(&t)` | 卸载时确保 tasklet 不再运行，在 remove 回调中调 |

**用法注意：** `tasklet_kill` 在驱动 remove 回调中调，等当前 tasklet 跑完才返回。确保卸载时不会有 tasklet 还在访问即将释放的内存。如果 tasklet 还没有被 schedule，直接返回 0。

**特点：**

- **不能 sleep。** tasklet 运行在 softirq context（硬中断关闭后、进程调度恢复前），本质还是原子上下文。不能调 `schedule()`、不能调 `mutex_lock()`、不能调 `copy_from_user()`。
- **同一 tasklet 不会并行。** 同一个 tasklet 不会同时在两个 CPU 上跑。如果 CPU0 上正在执行 tasklet A，CPU1 触发了 tasklet A，要等 CPU0 跑完才执行。这保证了你不需要在 tasklet 函数内部加锁保护自己的数据（只有一个执行流）。
- **不同 tasklet 可以并行。** tasklet A 在 CPU0 上跑、tasklet B 在 CPU1 上跑，是允许的。如果 A 和 B 访问同一份全局数据，你得自己加 spinlock。
- **软中断级别的 tasklet 不能抢占另一个软中断。** 同一个 CPU 上，softirq/tasklet 执行期间，进程调度被禁止，中断 handler 可以抢占 tasklet（中断优先级更高）。如果 tasklet 和顶半部分享数据，需要 `spin_lock_irqsave`。

**现状说明（v6.6.x LTS）：** tasklet 依然是完整支持的核心机制，未在代码层面打上 `__deprecated` 标签。社区在 v6.9（2024 年 5 月）引入了 `system_bh_wq`（BH workqueue）作为替代方向，但 v6.6 中不存在此机制。社区演进方向是不鼓励新驱动盲目使用 tasklet，原因在于 softirq context 中长时间执行会拖累 CPU 的进程调度。

**替代首选：** 线程化 IRQ（`devm_request_threaded_irq`）——底半部跑在内核线程中，可以 sleep，可被调度，对实时性更友好。

**适用场景：** 只有对底半部响应延迟要求极高（硬中断一完立马执行）、数据量极小、不涉及任何 sleep 操作时，v6.6 下使用 tasklet 依然是合理选择。

#### 线程化 IRQ（threaded IRQ）

见 1.2.2 的 `devm_request_threaded_irq`。新驱动优先选择底半部机制。

#### workqueue — 可以 sleep、延迟、取消

workqueue 将工作交给 `kworker` 内核线程执行。与线程化 IRQ 的关键区别：

- **线程化 IRQ**：`thread_fn` 只能被中断触发，没有中断就不会跑。生命周期完全跟着中断走。
- **workqueue**：可以在任何 context（中断、进程、定时器）调 `schedule_work()` 提交 work，不需要中断触发。可以延迟执行、可以取消、可以在卸载时安全等待 work 跑完。

```c
#include <linux/workqueue.h>

void my_work_fn(struct work_struct *work);

/* 方式一：静态声明（编译时初始化）*/
DECLARE_WORK(my_work, my_work_fn);
DECLARE_DELAYED_WORK(my_dwork, my_work_fn);

/* 方式二：运行时初始化（probe 中调）*/
struct work_struct *w;
struct delayed_work *dw;
INIT_WORK(w, my_work_fn);
INIT_DELAYED_WORK(dw, my_work_fn);

schedule_work(&my_work);                          /* 尽快执行 */
schedule_delayed_work(&my_dwork, msecs_to_jiffies(100)); /* 延迟 100ms */
```

| API | 说明 |
|-----|------|
| `DECLARE_WORK(name, func)` / `INIT_WORK(work, func)` | 声明 / 初始化普通 work |
| `DECLARE_DELAYED_WORK(name, func)` / `INIT_DELAYED_WORK(work, func)` | 声明 / 初始化延迟 work |
| `schedule_work(&w)` | 提交到系统 workqueue，尽快执行 |
| `schedule_delayed_work(&dw, delay)` | 延迟指定 jiffies 后执行 |
| `cancel_work_sync(&w)` / `cancel_delayed_work_sync(&dw)` | 取消，等跑完再返回 |
| `flush_work(&w)` / `flush_delayed_work(&dw)` | 等 work 执行完 |

**适用场景：**
- 中断中收了数据，需要提交到文件系统或网络栈
- 需要延迟处理（去抖、超时检查）
- 卸载时需要安全取消未执行的工作

#### 场景选型：线程化 IRQ 还是 workqueue？

两者都可以在中断下半部 sleep，最本质的区别：

| 特性 | 线程化 IRQ | workqueue |
|------|-----------|-----------|
| 执行线程 | 独立实时线程（`SCHED_FIFO`，优先级 50） | 共享 `kworker` 线程池（`SCHED_NORMAL`） |
| 来源 | 只能由特定硬件中断触发 | 任意 context（中断、timer、进程） |
| 串行保证 | 配合 `IRQF_ONESHOT` 自动串行 | 无，需自加锁 |
| 延迟/取消 | 不支持原生延迟/取消 | `schedule_delayed_work` / `cancel_work_sync` |
| 同步注销 | `free_irq` 内部自动调 `synchronize_irq` 等 thread 退出 | 卸载时需手动调 `cancel_work_sync` |

**选型判断：**

- **强实时、随硬件触发、不需要延迟/取消 → 线程化 IRQ**。自带实时调度优先级、`IRQF_ONESHOT` 保证串行、代码最简洁。
- **需要延迟触发（去抖）、需要主动取消、或从多个非中断地方触发 → workqueue**。灵活性高。

**实际例子：**

**线程化 IRQ — I2C 触控屏：** 中断来了就要读一次坐标，不存在"取消这次读数"。每个中断通过 IRQF_ONESHOT 保证串行处理。

```c
devm_request_threaded_irq(dev, irq, NULL, touch_thread_fn,
                           IRQF_ONESHOT, "touch", touch);
```

**workqueue — 按键机械去抖：** 按键按下后延迟 50ms 再读 GPIO 确认电平。`mod_delayed_work` 原生支持延迟：

```c
static irqreturn_t btn_handler(int irq, void *d)
{
    struct btn_dev *btn = d;
    mod_delayed_work(system_wq, &btn->debounce_work, msecs_to_jiffies(50));
    return IRQ_HANDLED;
}
/* 线程化 IRQ 无法原生延迟处理 */
```

**workqueue — 异步固件加载：** 设备启动中断触发后，需要从文件系统加载 10MB 固件到外设。耗时长、可被取消、不要求实时性：

```c
static int fw_remove(struct platform_device *pdev)
{
    cancel_work_sync(&dev->fw_work);  /* 确保 work 不再访问释放的内存 */
    return 0;
}
```

**一句话：需要延迟/取消 → workqueue；不需要 → 线程化 IRQ 更简单。**

#### 四种机制对比

| 特性 | softirq | tasklet | threaded IRQ | workqueue |
|------|---------|---------|-------------|-----------|
| 驱动直接使用？ | ❌ 内核核心专属 | ✅ 可以用 | ✅ 推荐 | ✅ 需要时用 |
| 能否 sleep？ | ❌ 绝对不可 | ❌ 绝对不可 | ✅ 可以 | ✅ 可以 |
| 执行 context | 软中断上下文 | 软中断上下文 | 进程上下文（`irq_thread`） | 进程上下文（`kworker`） |
| 调度优先级 | 极高（紧跟硬中断） | 极高（紧跟硬中断） | **高（实时 `SCHED_FIFO`）** | 普通（`SCHED_NORMAL`） |
| 典型场景 | 网卡收包、timer | 历史遗留驱动 | I2C/SPI/UART 外设驱动 | 延迟任务、低速 IO、异步加载 |

### 1.2.4 运行时控制

#### disable_irq / enable_irq — 开关中断

```c
void disable_irq(unsigned int irq);          /* 屏蔽中断，并等待当前 handler 退出（忙等待） */
void disable_irq_nosync(unsigned int irq);   /* 屏蔽中断，不等当前 handler 直接返回 */
void enable_irq(unsigned int irq);
```

**核心概念：depth 计数器**

`disable_irq` 不直接操作硬件屏蔽寄存器，它操作 `irq_desc->depth`：

```
初始：depth = 0（中断使能）

disable_irq(irq):                     第二次 disable_irq(irq):
  depth++ → depth = 1                   depth++ → depth = 2
  if (depth == 1) ← 第一次才调 mask     ↑ 不调 mask, 只计数
    chip->irq_mask()

enable_irq(irq):                      第二次 enable_irq(irq):
  depth-- → depth = 1                   depth-- → depth = 0
  ↑ 还不到 unmask                       if (depth == 0) ← 回到 0 才 unmask
                                          chip->irq_unmask()
```

`disable_irq` / `enable_irq` 必须成对出现。少一个中断永远打不开，多一个永远关不掉。

| 函数 | 是否等待 | 能否在硬中断 context 调 | 典型场景 |
|------|---------|----------------------|---------|
| `disable_irq` | 🔄 同步等待（自旋忙等） | ❌ **禁止**（关闭本中断会自锁死锁） | 进程 context 临时关中断 |
| `disable_irq_nosync` | ⚡ 不等待，直接计数 | ✅ 可以 | 硬中断 handler 中临时关本中断 |

> **为什么 `disable_irq` 不能在硬中断 context 调？** 不是因为可能 sleep，而是因为它是**忙等待**——通过自旋等待当前 handler 执行完。如果你在 CPU0 的 handler 中调 `disable_irq(本中断)`，handler 自己等自己结束，CPU0 永久死锁。

#### synchronize_irq — 等 handler 全部退出

```c
void synchronize_irq(unsigned int irq);
```

等待该中断线的所有 handler（包括线程化 IRQ 的 thread_fn）执行完毕。

**驱动卸载的正确做法：**

`free_irq()` 内部第一步就会屏蔽硬件中断并调用 `synchronize_irq()` 等待所有 handler 安全退出。所以 **不需要**在 free_irq 前手动调 `disable_irq` + `synchronize_irq`，直接调 `free_irq` 即可：

```c
static int my_remove(struct platform_device *pdev)
{
    struct my_dev *dev = platform_get_drvdata(pdev);

    /* 让硬件停止上报中断（可选：外设通过 I2C/SPI 关中断）*/
    my_hardware_disable_int(dev);

    /* 直接释放，内核内部自动 mask + synchronize_irq */
    free_irq(dev->irq, dev);
    return 0;
}
```

> 使用 `devm_request_irq` 时，内核在 devres 释放时自动调 `free_irq`。如果**只需要释放中断**，可以省略 remove 回调。但如果还要释放非 devm 内存、注销字符设备等，仍需写 remove。

#### irq_set_irq_wake — 配置唤醒源

```c
int irq_set_irq_wake(unsigned int irq, unsigned int on);
```

| 参数 | 说明 |
|------|------|
| `irq` | virq |
| `on` | 1 = 使能唤醒，0 = 取消唤醒 |

**内部链路（STM32MP257）：**

系统进入深度休眠（Standby）时，GIC 通常会被断电。真正负责唤醒的是 Always-On 电源域中的 EXTI 控制器：

```
irq_set_irq_wake(irq, 1)
  → 内核标记 IRQD_WAKEUP_STATE         ← 休眠时跳过对此中断的 disable 屏蔽
  → stm32mp_exti_set_wake()
    → 配置 EXTI_IMR/EMR 寄存器          ← 休眠期保持 event 使能
    → 联动 PWR 电源管理单元              ← 配置为 AON 域唤醒源
```

> 休眠时，内核默认会调 `suspend_device_irqs()` 关闭所有中断。
> `irq_set_irq_wake(irq, 1)` 会把该中断标记为唤醒源，**使它不被 suspend_device_irqs 屏蔽**，从而能在休眠状态下来电唤醒 CPU。

**注意事项：**
- 内部维护 `wake_depth` 计数器，调用 1 次 `(irq, 1)` 必须配对 1 次 `(irq, 0)`，否则系统无法进入深度休眠
- DTS 中标准写法声明唤醒源，子系统驱动自动处理：

```dts
gpio-keys {
    compatible = "gpio-keys";
    button-user {
        gpios = <&gpioh 5 GPIO_ACTIVE_HIGH>;
        linux,code = <BTN_1>;
        wakeup-source;       /* 标准 DT 唤醒源声明，推荐使用 */
        /* 兼容旧写法：gpio-key,wakeup（ATK 板 DTS 实际使用此传统写法）*/
    };
};
```

## 1.3 使用场景示例

> 本节覆盖 1.2 中介绍的核心 API，给出完整的驱动代码模板。

### 1.3.1 场景一：标准外设中断（platform_get_irq + devm_request_irq）

**适用 DTS 配置：** 场景 A，SoC 内部外设直连 GIC。

**ATK 板实际外设：** SDMMC2（eMMC）、USART1、I2C3 等。

**完整驱动 probe 模板：**

```c
#include <linux/interrupt.h>
#include <linux/platform_device.h>
#include <linux/io.h>

struct my_dev {
    void __iomem *base;
    int irq;
    /* 设备自定义数据 */
};

static irqreturn_t my_handler(int irq, void *dev_id)
{
    struct my_dev *dev = dev_id;
    u32 sr;

    sr = readl(dev->base + REG_SR);
    if (!(sr & INT_PENDING))
        return IRQ_NONE;

    /* 清中断标志 */
    writel(sr, dev->base + REG_SR);
    /* 处理数据... */

    return IRQ_HANDLED;
}

static int my_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct my_dev *mydev;
    int irq, ret;

    mydev = devm_kzalloc(dev, sizeof(*mydev), GFP_KERNEL);
    if (!mydev)
        return -ENOMEM;
    platform_set_drvdata(pdev, mydev);

    /* 拿中断号（DTS 已配 interrupts = <GIC_SPI N type>）*/
    irq = platform_get_irq(pdev, 0);
    if (irq < 0)
        return dev_err_probe(dev, irq, "get irq failed\n");

    /* irqflags = 0：DTS 已指定触发类型，不需要再传 IRQF_TRIGGER_*
     *             否则内核会做一致性检查，不一致时打印 warning */
    ret = devm_request_irq(dev, irq, my_handler, 0, "mydev", mydev);
    if (ret)
        return dev_err_probe(dev, ret, "request irq failed\n");

    mydev->irq = irq;
    return 0;
}

static int my_remove(struct platform_device *pdev)
{
    /* devm_request_irq 已自动释放中断，无需手动 free_irq */
    return 0;
}

static const struct of_device_id my_of_match[] = {
    { .compatible = "vendor,my-device" },
    { }
};
MODULE_DEVICE_TABLE(of, my_of_match);

static struct platform_driver my_driver = {
    .probe  = my_probe,
    .remove = my_remove,
    .driver = {
        .name = "mydev",
        .of_match_table = my_of_match,
    },
};
module_platform_driver(my_driver);
```

**验证：** 启动后 `cat /proc/interrupts | grep mydev` 应能看到对应行。

---

### 1.3.2 场景二：GPIO 中断（gpiod_to_irq + devm_request_threaded_irq）

**适用 DTS 配置：** 场景 C，外设中断连到 GPIO pin，DTS 中用 `gpios` 属性。

**ATK 板实际设备：** PH5 按键（gpio-keys）。

```c
#include <linux/gpio/consumer.h>
#include <linux/interrupt.h>
#include <linux/module.h>
#include <linux/platform_device.h>

struct button_dev {
    struct gpio_desc *key_gpio;
    int irq;
};

static irqreturn_t button_thread_fn(int irq, void *dev_id)
{
    struct button_dev *btn = dev_id;
    int val;

    /* 读 GPIO 当前电平 */
    val = gpiod_get_value(btn->key_gpio);
    dev_info(btn->dev, "key value = %d\n", val);

    return IRQ_HANDLED;
}

static int button_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct button_dev *btn;
    int irq, ret;

    btn = devm_kzalloc(dev, sizeof(*btn), GFP_KERNEL);
    if (!btn)
        return -ENOMEM;
    btn->dev = dev;
    platform_set_drvdata(pdev, btn);

    /* 拿 GPIO 描述符 */
    btn->key_gpio = devm_gpiod_get(dev, NULL, GPIOD_IN);
    if (IS_ERR(btn->key_gpio))
        return dev_err_probe(dev, PTR_ERR(btn->key_gpio), "get gpio failed\n");

    /* GPIO 描述符 → 中断号 */
    irq = gpiod_to_irq(btn->key_gpio);
    if (irq < 0)
        return dev_err_probe(dev, irq, "gpio to irq failed\n");

    /* 注册线程化中断 */
    ret = devm_request_threaded_irq(dev, irq, NULL, button_thread_fn,
                                     IRQF_TRIGGER_RISING | IRQF_ONESHOT,
                                     "button", btn);
    if (ret)
        return dev_err_probe(dev, ret, "request irq failed\n");

    return 0;
}

static const struct of_device_id button_of_match[] = {
    { .compatible = "vendor,gpio-button" },
    { }
};
MODULE_DEVICE_TABLE(of, button_of_match);

static struct platform_driver button_driver = {
    .probe  = button_probe,
    .driver = {
        .name = "gpio-button",
        .of_match_table = button_of_match,
    },
};
module_platform_driver(button_driver);
```

> 与 `gpio-keys` 驱动的区别：`gpio-keys` 是 input 子系统驱动，上报 `input_event`。这里简化成了直接 `dev_info` 打印，只展示最核心的"GPIO 中断"流程。

---

### 1.3.3 场景三：线程化 IRQ + workqueue 混合使用

**适用场景：** 设备每次中断都要处理（用线程化 IRQ），但还需要延迟去抖或定期轮询（用 workqueue）。

**ATK 板实际场景举例：** 按键去抖。中断来时先不立刻上报，等 50ms 后再确认电平。

```c
#include <linux/interrupt.h>
#include <linux/workqueue.h>
#include <linux/gpio/consumer.h>

struct debounce_dev {
    struct gpio_desc *gpio;
    int irq;
    struct delayed_work dwork;
};

static void debounce_work_fn(struct work_struct *work)
{
    struct delayed_work *dwork = to_delayed_work(work);
    struct debounce_dev *dev = container_of(dwork, struct debounce_dev, dwork);
    int val;

    val = gpiod_get_value(dev->gpio);
    if (val)
        dev_info(dev, "key pressed (debounced)\n");
    else
        dev_info(dev, "key released (debounced)\n");
}

static irqreturn_t debounce_handler(int irq, void *dev_id)
{
    struct debounce_dev *dev = dev_id;

    /* 每次中断都推迟 50ms，抖动产生的快速连续中断只会刷新定时器 */
    mod_delayed_work(system_wq, &dev->dwork, msecs_to_jiffies(50));

    return IRQ_HANDLED;
}

static int debounce_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct debounce_dev *db;
    int irq, ret;

    db = devm_kzalloc(dev, sizeof(*db), GFP_KERNEL);
    platform_set_drvdata(pdev, db);

    db->gpio = devm_gpiod_get(dev, NULL, GPIOD_IN);
    if (IS_ERR(db->gpio))
        return PTR_ERR(db->gpio);

    irq = gpiod_to_irq(db->gpio);
    if (irq < 0)
        return irq;

    INIT_DELAYED_WORK(&db->dwork, debounce_work_fn);

    ret = devm_request_irq(dev, irq, debounce_handler,
                           IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING,
                           "debounce", db);
    if (ret)
        return ret;

    return 0;
}
```

**关键点：** `mod_delayed_work` 重置定时器，抖动产生的多次中断不会导致多次上报。

---

### 1.3.4 场景四：tasklet 快速处理（不 sleep 的底半部）

**适用场景：** 中断只需要快速拷贝数据或操作寄存器，不需要 sleep。例如 SPI 接收中断，handler 把数据搬到一个缓冲区，然后唤醒 tasklet 做校验处理。

```c
#include <linux/interrupt.h>

struct spi_dev {
    void __iomem *base;
    struct tasklet_struct rx_tasklet;
    u32 rx_buf[64];
    unsigned int rx_count;
};

/* tasklet 回调：在 softirq context 执行，不能 sleep */
static void spi_rx_tasklet(struct tasklet_struct *t)
{
    struct spi_dev *spi = from_tasklet(spi, t, rx_tasklet);

    /* 校验数据、提交流到上层（如 tty 或 input 子系统）*/
    for (int i = 0; i < spi->rx_count; i++)
        pr_debug("spi rx[%d] = 0x%08x\n", i, spi->rx_buf[i]);

    /* 重新使能设备中断 */
    writel(1, spi->base + REG_INT_EN);
}

static irqreturn_t spi_handler(int irq, void *dev_id)
{
    struct spi_dev *spi = dev_id;

    /* 快速拷贝数据到临时缓冲区 */
    while (readl(spi->base + REG_SR) & RX_READY)
        spi->rx_buf[spi->rx_count++] = readl(spi->base + REG_RX);

    /* 关闭设备中断（tasklet 处理完再开），防止暴风 */
    writel(0, spi->base + REG_INT_EN);

    /* 唤醒 tasklet 做后续处理 */
    tasklet_schedule(&spi->rx_tasklet);

    return IRQ_HANDLED;
}

static int spi_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct spi_dev *spi;
    int irq, ret;

    spi = devm_kzalloc(dev, sizeof(*spi), GFP_KERNEL);
    platform_set_drvdata(pdev, spi);
    spi->base = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(spi->base))
        return PTR_ERR(spi->base);

    /* 运行时初始化 tasklet */
    tasklet_setup(&spi->rx_tasklet, spi_rx_tasklet);

    irq = platform_get_irq(pdev, 0);
    if (irq < 0)
        return irq;

    ret = devm_request_irq(dev, irq, spi_handler, 0, "spi", spi);
    if (ret)
        return ret;

    return 0;
}

/* remove 回调：卸载时确保 tasklet 已停止 */
static int spi_remove(struct platform_device *pdev)
{
    struct spi_dev *spi = platform_get_drvdata(pdev);

    tasklet_kill(&spi->rx_tasklet);
    return 0;
}
```

**为什么这里用 tasklet 而不是线程化 IRQ？** handler 中只做了"memcpy 级别"的快速操作，不需要 sleep。tasklet 足够处理，延迟比线程化 IRQ 更低（softirq context，紧跟硬中断）。但如果校验逻辑涉及锁或文件操作，就应该用 workqueue 或线程化 IRQ。

**为什么要在 handler 中关设备中断？** 防止 tasklet 还没跑、又来了新中断导致缓冲区覆盖。tasklet 处理完再重新开中断。

---

## 1.4 调试接口

### 1.4.1 /proc/interrupts — 查触发次数和中断源

**你要做什么：** 怀疑中断没触发，或触发频率不对。第一件事就是在 `/proc/interrupts` 中确认计数是否增加、中断是谁注册的。

**怎么看：**

```bash
$ cat /proc/interrupts | grep " 70:"
 70:          5          3     stm32mp-exti   5  Edge      gpio-keys
```

| 列 | 值 | 这列告诉你什么 |
|----|----|---------------|
| `70` | virq=70 | 软件用的中断号，`request_irq(70, handler)` |
| `5` | CPU0 触发 5 次 | 中断是否到了 CPU0 |
| `3` | CPU1 触发 3 次 | 中断是否均衡到了两个核 |
| `stm32mp-exti` | chip name | 中断控制器名称（GPIO 中断显示 EXTI chip；直连 GIC 的外设显示 GIC-400） |
| `5` | hwirq=5 | EXTI event 5（GPIO pin 5），不是 GIC 的硬件 ID |
| `Edge` | 触发类型 | `Edge` 边沿 / `Level` 电平 |
| `gpio-keys` | 设备名 | 注册 `devm_request_irq` 时传入的 `devname` |

**第 5 列显示的是叶节点 irq_domain 的 hwirq，取决于中断路径：**

- **GPIO/EXTI 路径**（如 gpio-keys）：显示 EXTI 域的 hwirq = GPIO pin 号（如 `5`）。不是 GIC 的硬件 ID。
- **直连 GIC 路径**（如 SDMMC）：显示 GIC 的硬件 hwirq = DTS SPI 号 + 32：
  ```
  DTS <GIC_SPI 273> → /proc/interrupts 第 5 列 = 305
                    → GICD_ISENABLER bit 305
  ```

直连 GIC 的外设查 `/proc/interrupts` 时注意第 5 列是 hwirq（加了 32），不是 DTS 里的原始 SPI 号。

**排查手段一：两次对比，看计数是否增长**

```bash
cat /proc/interrupts > /tmp/a
sleep 5
cat /proc/interrupts > /tmp/b
diff /tmp/a /tmp/b
```

- **计数增加了** → 中断到了 CPU，问题在 driver handler 内部（handler 没正确处理或返回了 `IRQ_NONE`）
- **计数不变** → 中断没到 CPU。原因可能是：硬件没触发、EXTI IMR 屏蔽了、GIC 没使能该 SPI

**排查手段二：排除虚假中断**

```bash
# 看 dmesg 有没有 "nobody cared" 或 "Disabling IRQ"
dmesg | grep -i "irq.*disable\|nobody cared"
```

如果有输出，说明 handler 返回了太多次 `IRQ_NONE`，内核检测到"虚假中断"后自动关了该中断。这时先去 `/proc/irq/N/spurious` 确认计数。

**排查手段三：判断中断均衡**

双核 Cortex-A35 上正常情况两个 CPU 列的计数接近。如果某行只有 CPU0 有计数，说明 `smp_affinity` 被固定到了 CPU0（见 1.4.2 节）。

> 注意：PPI（私有外设中断，如 `arch_timer`）在 `/proc/interrupts` 中**绝对会出现**，且各 CPU 列都有独立的高计数，不是"不出现在 proc"的中断。

**什么情况下看不到某行或行异常：**

| 现象 | 原因 | 背后机制 |
|------|------|---------|
| 整行不出现 | 没有驱动成功申请该中断 | driver probe 失败、没调 `devm_request_irq`、DTS `status = "disabled"` |
| 行存在，计数永远卡住不动 | 中断被内核强制关闭 | 因连续返回 `IRQ_NONE` 触发虚假中断保护，dmesg 报 `Disabling IRQ #N`。**此时行不会消失，只是计数不再增长** |
| 行存在，计数始终 0 | 链式中继中断（Nested IRQ） | 由 `handle_nested_irq()` 转发处理，不走 `kstat_irqs` 统计 |

---

### 1.4.2 /proc/irq/N/ — 改亲和性、查虚假中断计数

**你要做什么：**  
- 场景 A：mmc1 中断在 CPU0 上太多，想分一些到 CPU1  
- 场景 B：怀疑某个中断被内核判为虚假中断，看计数确认

**目录结构：**

```bash
$ ls /proc/irq/70/          # virq=70 的按键中断
affinity_hint  default_smp_affinity  node         spurious
effective_aff  effective_affinity_list  smp_affinity
```

**场景 A：把中断绑到特定 CPU**

现代内核推荐使用更直观的 `smp_affinity_list`（直接写 CPU 编号），无需做十六进制位运算：

```bash
# 方式一：smp_affinity（十六进制掩码）
$ cat /proc/irq/70/smp_affinity
1                # bit 0 = CPU0，bit 1 = CPU1

$ echo 2 > /proc/irq/70/smp_affinity         # 改为 CPU1

# 方式二：smp_affinity_list（CPU 列表，更直观）
$ echo 1 > /proc/irq/70/smp_affinity_list    # 直接指定 CPU1

# 确认实际生效
$ cat /proc/irq/70/effective_affinity
2                # 实际只发到 CPU1 了
```

`smp_affinity` 写的是期望值，`effective_affinity` 是实际生效值。两者不同的原因：
- GIC-400 可能不支持指定的 CPU 子集
- 中断类型是 PPI，本来就只能发给一个 CPU

**场景 B：查虚假中断**

```bash
$ cat /proc/irq/70/spurious
count 100050           # 该中断总计触发次数
unhandled 0            # 返回 IRQ_NONE 的次数
last_unhandled 0 ms    # 上一次发生 IRQ_NONE 的时间点
```

内核在 `note_interrupt()`（`kernel/irq/spurious.c` L272）中检测虚假中断。关键逻辑：

- 每次 handler 返回 `IRQ_NONE`，比较当前时间与 `last_unhandled` 是否超过 100ms：
  - **超过 100ms**：`irqs_unhandled = 1`（重置，不是归零）——防止低频偶发中断被累加误杀
  - **未超过 100ms**：`irqs_unhandled++`（密集中断正常累加）
- 当累计 `irq_count` 达到 100000 次，且其中 `irqs_unhandled > 99900`（即 >99.9% 未被处理）时：

```
irq 70: nobody cared (try booting with the "irqpoll" option)
Disabling IRQ #70
```

**排查方法：** 观察 `unhandled` 行是否在触发事件时稳定增长：

```bash
watch -n 2 'cat /proc/irq/70/spurious | grep unhandled'
```

如果 `unhandled` 随每次按键 +1 → handler 返回了 `IRQ_NONE`，去检查 `devm_request_irq` 注册的 handler 函数。

**其他文件：**

| 文件 | 内容 | 用途 |
|------|------|------|
| `node` | NUMA 节点（STM32MP257 单节点 = 0） | 确认没有跨 NUMA 访问 |
| `default_smp_affinity` | 驱动或 irqchip 设置的默认值 | 对比当前值判断是否被人改过 |
| `affinity_hint` | 驱动建议的亲和性 | 一般跟 default 一致 |

---

### 1.4.3 DebugFS — 查 irq_domain 拓扑和完整字段

**你要做什么：** `/proc/interrupts` 只告诉你触发次数和注册者。DebugFS 可以看到更多——这个中断的 irq_chip 是什么、domain 层级是什么样的、流控函数是什么。

```bash
$ mount -t debugfs none /sys/kernel/debug       # 挂载 debugfs
$ ls /sys/kernel/debug/irq/
domains/   irqs/                                 # 两个子目录
```

**`/sys/kernel/debug/irq/irqs/N` — 单个中断的全部字段**

以 virq=70（按键 PH5）为例：

```bash
$ cat /sys/kernel/debug/irq/irqs/70
```

输出分三段：

**① handler — 流控函数**

```
handler:  handle_fasteoi_irq
```

| 流控函数 | 对应什么 | STM32MP257 上谁在用 |
|---------|---------|-------------------|
| `handle_fasteoi_irq` | GIC/EOI 模式 | GIC SPI 中断、EXTI 转发的 SPI |
| `handle_edge_irq` | 边沿触发 | GPIO 边沿中断、某些 USB |
| `handle_level_irq` | 电平触发（非 EOI） | 极少见 |

**② chip — irq_chip 和 domain 层级**

```
irq_data.chip: stm32mp-exti
    domain:  stm32mp-exti-0
    parent chip: GIC-400
    parent domain: irqchip@0
```

这四行直接证明了 STM32MP257 的两层 domain 拓扑：

```
domain: stm32mp-exti-0    ← EXTI domain（子域）
    ↑ parent domain
parent domain: irqchip@0  ← GIC domain（父域，根）
    ↑ parent chip
parent chip: GIC-400      ← GIC 的 irq_chip
```

**③ action — 驱动注册的回调**

```
handler:  gpio_keys_irq_handler    ← devm_request_irq 注册的 handler
devname:  gpio-keys                ← /proc/interrupts 最后一列
dev_id:   ffffff810012abc0         ← 传给 handler 的 cookie（设备结构体地址）
```

如果中断是共享的（`IRQF_SHARED`），这里会有多组 `handler/devname/dev_id`。

**`/sys/kernel/debug/irq/domains/` — domain 拓扑和映射表**

```bash
$ ls /sys/kernel/debug/irq/domains/
irqchip@0  stm32mp-exti-0
```

每个 domain 一个目录：

```bash
$ ls /sys/kernel/debug/irq/domains/stm32mp-exti-0/
name        size        hwirqs      mappings

$ cat /sys/kernel/debug/irq/domains/stm32mp-exti-0/name
stm32mp-exti-0

$ cat /sys/kernel/debug/irq/domains/stm32mp-exti-0/size
96                  # EXTI domain 支持 96 个 event
```

**`mappings` 是最有用的文件：**

```bash
$ cat /sys/kernel/debug/irq/domains/stm32mp-exti-0/mappings
5 -> 0x48 0x00000001 0x00000002      # EXTI1 event 5 → virq=72 (0x48)
6 -> 0x49 0x00000001 0x00000002      # EXTI1 event 6 → virq=73
```

格式：

```
hwirq → virq(hex)  revmap  flags
```

**跨 domain 查整条路径：**

```
GPIOH pin 5 → EXTI1 event 5 → GIC 硬件 ID 305 (DTS GIC_SPI 273 + 32)

查法：
  ① EXTI domain mappings 看 hwirq=5 → virq=72（假设值）
  ② /proc/interrupts 找 virq=72 → 列 4 显示 chip=stm32mp-exti, 列 5 显示 hwirq=5（EXTI 域的 hwirq，不是 GIC 的）
  ③ GIC domain mappings 中 hwirq=305 对应的 virq 也是 72 → 确认
```

> **实际排查中 debugfs 不常用**——`/proc/interrupts` 解决 80% 的问题，`/proc/irq/N/` 解决 15%，debugfs 只在查 domain 拓扑、确认 chip 层级时才需要。

---

### 1.4.4 调试接口速查

| 工具 | 解决什么问题 | 关键命令 |
|------|------------|---------|
| `/proc/interrupts` | 中断触发了几次？谁注册的？均衡了吗？ | `watch -n 1 'cat /proc/interrupts \| grep your_irq'` |
| `/proc/irq/N/smp_affinity` | 怎么把中断绑到指定 CPU？ | `echo 2 > /proc/irq/N/smp_affinity` |
| `/proc/irq/N/spurious` | 被内核判为虚假中断了吗？ | `cat /proc/irq/N/spurious` |
| `debugfs irqs/N` | irq_chip 是什么？domain 层级？流控函数？ | `cat /sys/kernel/debug/irq/irqs/N` |
| `debugfs domains/` | hwirq 到 virq 的映射表？ | `cat /sys/kernel/debug/irq/domains/*/mappings` |

## 1.5 实战排查：PH5 按键中断不工作

### 场景

ATK 板上的 User-Key（PH5）按了没反应。`evtest` 没输出。已知：
- DTS 中 `gpio-keys` 配了 `gpios = <&gpioh 5 GPIO_ACTIVE_HIGH>`
- 内核启动日志中 `gpio-keys` probe 成功（无报错）

### 排查步骤

**第一步：确认中断计数（软件层——最快判断问题范围）**

```bash
$ cat /proc/interrupts | grep gpio-keys
 70:          0          0     stm32mp-exti   5  Edge      gpio-keys
```

- 第 4 列 chip = `stm32mp-exti`（不是 GIC-400，因为 GPIO 中断的 irq_chip 是 EXTI）
- 第 5 列 hwirq = `5`（EXTI event 5 = GPIO pin 5，不是 GIC SPI 305）
- 触发类型 = `Edge`（按键必须是边沿触发，电平触发会卡死）
- 计数是 0/0——说明从未触发

**第二步：连续两次对比，确认计数不变**

```bash
$ cat /proc/interrupts | grep " 70:" > /tmp/before
# 按 10 次按键
$ cat /proc/interrupts | grep " 70:" > /tmp/after
$ diff /tmp/before /tmp/after
```

计数完全没变 → **中断没到 CPU**。问题在硬件路径（GPIO → EXTI → GIC），不是 driver 的 handler。

**第三步：devmem 查 EXTI 寄存器**

PH5 → GPIO pin 5 → EXTI1 event 5。event 5 在 Bank 1（events 0~31）：

| EXTI 寄存器 | 偏移 | bank 1 | 关注 bit |
|------------|------|--------|---------|
| RPR (Rising Pending) | 0x00 | RPR1 在 0x00 | bit 5 |
| FPR (Falling Pending) | 0x10 | FPR1 在 0x10 | bit 5 |
| IMR (Interrupt Mask) | 0x20 | IMR1 在 0x20 | bit 5 |

```bash
# EXTI 基址 = 0x44220000
# 读 IMR1，先确认 event 5 没有被屏蔽
$ devmem 0x44220020   # IMR1
0xFFFFFFFF            # bit 5 = 1，mask 正常

# 读 RPR1，看是否有上升沿
$ devmem 0x44220000   # RPR1
0x00000000            # 没有挂起

# 按一下按键，再读 RPR1
$ devmem 0x44220000
0x00000000            # 还是 0！
```

EXTI RPR1 的 bit 5 没有置位。说明 GPIO pin 5 的电平变化**没有到达 EXTI**。

可能原因：
- 硬件层面：GPIO 输入没有使能，IRQ MUX 没有配置
- 软件层面：`stm32_gpio_domain_activate` 没有正确写 IRQ MUX 寄存器

**第四步：查 GIC 寄存器（验证 EXTI → GIC 转发）**

> STM32MP257 使用 **GIC-400（GICv2）**，GICD（Distributor）基址 = 0x4ac10000（见 DTS `intc: interrupt-controller@4ac00000 { reg = <... 0x4ac10000 ...>; }`）。

EXTI1 event 5 映射到 GIC SPI 273（DTS 编号），GIC 硬件 ID = 273 + 32 = 305。hwirq=305 在 ISPENDR9（bits 288~319），bit offset = 305 - 288 = 17：

```bash
# ISPENDR9 = GICD + 0x200 + 9*4 = 0x4ac10000 + 0x224 = 0x4ac10224
$ devmem 0x4ac10224   # GICD_ISPENDR9
0x00000000            # bit 17 = 0，GIC 没收到任何信号
```

GIC 也没收到。结合第三步 EXTI RPR 为 0，说明**信号在 GPIO 到 EXTI 之间就断了**。

**第五步：检查 IRQ MUX 配置**

查看 GPIO domain 是否正确激活了 PH5 的中断路由：

```bash
$ echo 'module pinctrl-stm32 +p' > /sys/kernel/debug/dynamic_debug/control
$ dmesg | tail -10
```

如果能正常激活，问题可能在 IRQ MUX 的配置。检查 EXTI 驱动 probe 是否成功：

```bash
$ dmesg | grep -i exti
[    1.234] stm32mp-exti 44220000.interrupt-controller: EXTI driver registered
[    1.235] stm32mp-exti 44220000.interrupt-controller: Failed to get hwspinlock
```

**第六步：查 hwspinlock**

`Failed to get hwspinlock`！STM32MP257 是多核 SoC（A35 + M33），EXTI 寄存器可能被多核共享。在需要访问 IRQ MUX 寄存器时，EXTI 驱动尝试获取 hwspinlock（硬件自旋锁）。如果获取失败，驱动**跳过**了 IRQ MUX 的选择器配置（`stm32mp_exti_gpio_bank_alloc` 不会写 EXTI_CR 寄存器），导致 GPIO bank 引脚到 EXTI event 的路由从未建立。

```bash
$ dmesg | grep hwspinlock
[    0.567] hwspinlock 0: no hwspinlock registered
```

**结论：** DTS 中没有为 `hwspinlock` 节点配置 `status = "okay"`，或驱动未编译进内核。EXTI 获取 hwspinlock 失败后跳过了 IRQ MUX 配置，导致 GPIO→EXTI 路由不通。

> **延伸：hwspinlock 是什么？**
>
> 软件 spinlock 保护同一 OS 内多核访问共享内存，但 STM32MP257 是**多核异构 SoC**（A35 跑 Linux，M33 跑 RTOS/裸机），两侧不同软件、不同地址空间，无法共享一个内存中的 spinlock 变量。
>
> hwspinlock 是 SoC 内部的**硬件锁模块**——一组专用寄存器，所有核心（A35、M33）都通过相同物理地址访问。读某个锁寄存器：
> - 返回 `0`（空闲）：该核拿到锁
> - 返回非 `0`（已被锁）：别的核在用
>
> 硬件保证读操作是原子的，只有一个核能读到 `0`。EXTI 的 IRQ MUX 寄存器（EXTI_CR）是 A35 和 M33 共享的，所以在写它之前必须拿 hwspinlock。拿不到时驱动跳过 EXTI_CR 配置，路由就断了。

**修复后正常状态：**

```
 70:        127        103     stm32mp-exti   5  Edge      gpio-keys
```

CPU0/CPU1 都有计数，触发类型为 Edge，说明 GPIO→EXTI→GIC→CPU 路径全部打通。

---

### 1.5.1 完整排查流程速查

```
中断不工作
  │
  ├─ /proc/interrupts 搜 virq 行
  │    ├─ 无此行 → 中断没注册（去 dmesg 查 driver probe）
  │    └─ 有行，计数 0 → 中断没到 CPU
  │
  ├─ 两次对比计数
  │    ├─ 增长 → 中断到了 CPU，问题在 handler 内
  │    └─ 不变 → 中断没到 CPU
  │          │
  │          ├─ devmem EXTI RPR（EXTI 收到信号吗？）
  │          │    ├─ 没置位 → GPIO→EXTI 不通
  │          │    │             └─ 查 IRQ MUX、查 EXTI hwspinlock
  │          │    │
  │          │    └─ 置位了 → EXTI 收到了
  │          │         ├─ devmem EXTI IMR → 0 被屏蔽
  │          │         └─ devmem GICD_ISPENDR → 查 GIC 是否收到
  │          │
  │          └─ dmesg / dynamic_debug
  │               ├─ "Failed to get hwspinlock" → EXTI 跳过 IRQ MUX
  │               └─ "Disabling IRQ" → 虚假中断
  │
  └─ 修复后验证
       /proc/interrupts 计数增长
       evtest 能收到事件
```

**各步命令速查（基于 STM32MP2 + GIC-400）：**

| 步骤 | 命令 | 查到什么 |
|------|------|---------|
| 查中断注册 | `grep gpio-keys /proc/interrupts` | 无行→没注册 |
| 查计数增长 | 两次 `cat` 对比 | 不变→中断没到 CPU |
| 查 EXTI 收到 | `devmem 0x44220000`（RPR1），按按键 | bit 置位→EXTI 收到 |
| 查 EXTI 屏蔽 | `devmem 0x44220020`（IMR1） | 0→被屏蔽 |
| 查 GIC 收到 | `devmem 0x4ac10224`（ISPENDR9，hwirq 305） | bit 变化→GIC 收到 |
| 查虚假中断 | `cat /proc/irq/N/spurious` 或 `dmesg | grep "Disabling IRQ"` | spurious 增长 |
| 查 EXTI 驱动 | `dmesg | grep -i exti` | "Failed"→probe 部分失败 |
| 查 hwspinlock | `dmesg | grep hwspinlock` | "no hwspinlock"→IRQ MUX 跳过 |

---

## 总结

01 篇从 DTS 到 API 到排查实战，覆盖了 STM32MP257 上中断子系统的主要使用方法。

| 层面 | 核心内容 | 对应章节 |
|------|---------|---------|
| **设备树** | 三种中断路径 + GIC/EXTI/GPIO 三级控制器 | 1.1 |
| **内核 API** | 获取中断号 / 注册 handler / 下半部机制 / 运行时控制 / 唤醒源 | 1.2 |
| **使用场景** | 标准外设中断 / GPIO 中断 / 线程化 IRQ + workqueue / tasklet | 1.3 |
| **调试接口** | `/proc/interrupts` / `/proc/irq/N/` / DebugFS | 1.4 |
| **实战排查** | 从 `/proc/interrupts` 到 devmem 到 hwspinlock 的完整流程 | 1.5 |

继续阅读：[02-Architecture.md](02-Architecture.md) — 中断子系统的核心数据结构（irq_desc、irq_domain、irq_chip）与设计模式。
