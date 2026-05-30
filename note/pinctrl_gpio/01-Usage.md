# Pinctrl / GPIO 使用示例

> 看完就能上手写 DTS 和调 API，不深入原理。
>
> **字数**：约 8100 字 · **建议阅读时间**：35~50 分钟

## 1. DTS 配置

### 1.1 场景引导：什么时候该用什么

在开始写 DTS 之前，先回答一个问题：**我的设备要怎么用引脚？** 三种情况：

#### 场景 A——你的设备是 UART/I2C/MMC/SPI 等外设

这种设备的引脚是一个**固定功能组**——比如 USART1 需要 TX + RX 两个引脚，它们在板子设计时就已经确定连接到某个 SoC pin 上，运行期间不会改变。

**做法**：在 DTS 中写 `pinctrl-0`，指向 pin controller 节点中预定义的引脚组 phandle。内核在 probe 时自动把引脚配置为对应的外设功能。

```dts
&usart1 {
    pinctrl-names = "default";
    pinctrl-0 = <&usart1_pins_a>;  /* 指向下面的引脚组定义 */
};

&pinctrl {
    usart1_pins_a: usart1-0 {
        pins1 {
            pinmux = <STM32_PINMUX('G', 14, AF6)>; /* TX: PG14 → USART1_TX */
            bias-disable;
            drive-push-pull;
            slew-rate = <0>;
        };
        pins2 {
            pinmux = <STM32_PINMUX('G', 15, AF6)>; /* RX: PG15 → USART1_RX */
            bias-disable;
        };
    };
};
```

**驱动代码**：不需要调任何 pinctrl API。

#### 场景 B——你的设备需要控制一个 GPIO（LED/按键/复位）

这种场景不涉及"功能复用"——引脚就是当作 GPIO 用。不需要 `pinctrl-0`（STM32 上后门机制自动处理 MODER，但 i.MX 可能需要）。

**做法**：在 DTS 中写 `*-gpios` 属性，驱动中调 `devm_gpiod_get()` 拿到描述符后读写。

```dts
led {
    gpios = <&gpioh 4 GPIO_ACTIVE_LOW>;  /* PH4，低电平有效 */
};
```

```c
desc = devm_gpiod_get(dev, NULL, GPIOD_OUT_LOW);
gpiod_set_value(desc, 1);  /* 硬件引脚上输出低电平（ACTIVE_LOW 自动反转）*/
```

#### 场景 C——你的设备既需要固定功能又需要 GPIO

很多设备是混合的——比如 SDMMC 的数据线是固定功能（走 `pinctrl-0`），卡检测引脚是 GPIO（走 `cd-gpios`）。两者互不冲突，在同一个设备节点中同时声明：

```dts
&sdmmc1 {
    pinctrl-names = "default", "opendrain", "sleep";
    pinctrl-0 = <&sdmmc1_b4_pins_a>;           /* 数据线：pinctrl 配为外设功能 */
    cd-gpios = <&gpioi 8 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;  /* 卡检测：GPIO */
};
```

#### 三种路径的对比

| 路径 | 声明方式 | 谁在管 | 驱动要做什么 |
|------|---------|--------|-------------|
| pinctrl 外设功能 | `pinctrl-0 = <&xxx>` | pinctrl core 自动配 | 什么都不用做 |
| GPIO 功能 | `*-gpios` | gpiolib 后门机制 | 调 `gpiod_get/set` |
| 混合 | 两者都有 | pinctrl + gpiolib | 各管各的 |

> 如果你看到这里还不确定自己的设备属于哪种场景，先确认硬件设计：检查原理图上的引脚是连到外设模块（UART/I2C/SPI）还是连到 LED/按键。前者是场景 A，后者是场景 B。

下面详细展开每种路径的具体写法。

### 1.2 外设功能：pinctrl 配置

#### 属性格式

```dts
&usart1 {
    pinctrl-names = "default", "idle", "sleep";  /* 状态名列表 */
    pinctrl-0 = <&usart1_pins_a>;                /* 状态 0：default */
    pinctrl-1 = <&usart1_idle_pins_a>;           /* 状态 1：idle */
    pinctrl-2 = <&usart1_sleep_pins_a>;          /* 状态 2：sleep */
};
```

- `pinctrl-names`：给状态起名字（可选。不写时默认 `pinctrl-0` 为 "default"）
- `pinctrl-X`：指向 pin controller 节点中定义的引脚配置 phandle
- 一个 phandle 对应一个 function+group 的组合

ATK 板上常见的外设配置模式：

```dts
/* 只有 default 状态 */
&adc_12 {
    pinctrl-names = "default";
    pinctrl-0 = <&adc1_in15_pins_a>;
};

/* default + sleep 双状态，用于电源管理 */
&m_can1 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&m_can1_pins_a>;
    pinctrl-1 = <&m_can1_sleep_pins_a>;
};

/* 有 idle 状态（用于半关闭场景）*/
&usart1 {
    pinctrl-names = "default", "idle", "sleep";
    pinctrl-0 = <&usart1_pins_a>;
    pinctrl-1 = <&usart1_idle_pins_a>;
    pinctrl-2 = <&usart1_sleep_pins_a>;
};
```

#### 引脚配置节点的写法

引脚配置在 pin controller 的 `&pinctrl` 节点下定义，分为多个子节点，每个子节点对应一个引脚组（group）：

```dts
&pinctrl {
    /* 命名规则：<外设名>_<功能>_pins_<变体> */
    usart1_pins_a: usart1-0 {
        pins1 {
            pinmux = <STM32_PINMUX('G', 14, AF6)>; /* USART1_TX */
            bias-disable;
            drive-push-pull;
            slew-rate = <0>;
        };
        pins2 {
            pinmux = <STM32_PINMUX('G', 15, AF6)>; /* USART1_RX */
            bias-disable;
            /* RX 不需要配置驱动能力（输入模式）*/
        };
    };
};
```

**为什么分成 pins1、pins2？** 在同一组引脚中，TX 是推挽输出，RX 是输入——电气属性不同，所以分在不同的子节点中。pin controller 驱动在解析时会分别应用各自的配置。

#### 更复杂的例子：RGMII 以太网

```dts
eth2_rgmii_pins_a: eth2-rgmii-0 {
    pins1 {
        pinmux = <STM32_PINMUX('C', 7, AF10)>,  /* ETH_RGMII_TXD0 */
                 <STM32_PINMUX('C', 8, AF10)>,  /* ETH_RGMII_TXD1 */
                 <STM32_PINMUX('C', 9, AF10)>,  /* ETH_RGMII_TXD2 */
                 <STM32_PINMUX('C', 10, AF10)>, /* ETH_RGMII_TXD3 */
                 <STM32_PINMUX('C', 4, AF10)>;  /* ETH_RGMII_TX_CTL */
        bias-disable;
        drive-push-pull;
        slew-rate = <3>;
        st,io-retime = <1>;
        st,io-clk-edge = <1>;
    };
    pins2 {
        pinmux = <STM32_PINMUX('F', 8, AF10)>,  /* ETH_RGMII_CLK125 */
                 <STM32_PINMUX('F', 7, AF10)>;  /* ETH_RGMII_GTX_CLK */
        bias-disable;
        drive-push-pull;
        slew-rate = <3>;
        /* 时钟引脚没有 st,io-retime */
    };
    pins3 {
        pinmux = <STM32_PINMUX('C', 5, AF10)>;  /* ETH_MDIO */
        bias-disable;
        drive-push-pull;
        slew-rate = <0>;
    };
    pins4 {
        pinmux = <STM32_PINMUX('G', 0, AF10)>,  /* ETH_RGMII_RXD0 */
                 <STM32_PINMUX('C', 12, AF10)>, /* ETH_RGMII_RXD1 */
                 <STM32_PINMUX('F', 9, AF10)>,  /* ETH_RGMII_RXD2 */
                 <STM32_PINMUX('C', 11, AF10)>, /* ETH_RGMII_RXD3 */
                 <STM32_PINMUX('C', 3, AF10)>;  /* ETH_RGMII_RX_CTL */
        bias-disable;
        st,io-retime = <1>;
        st,io-clk-edge = <1>;
    };
    pins5 {
        pinmux = <STM32_PINMUX('F', 6, AF10)>;  /* ETH_RGMII_RX_CLK */
        bias-disable;
    };
};
```

这里分 5 组引脚的原因：TX 数据线需要额外的 `st,io-retime`、时钟线速度最高（slew-rate=3）、MDIO 速度最低（slew-rate=0）、RX 数据线也需要 `st,io-retime`——每组引脚电气配置不同，无法合并。

#### Sleep 状态的作用

suspend 时，内核自动将引脚切到 sleep 状态，把引脚设置为 Analog 模式以省电：

```dts
usart1_sleep_pins_a: usart1-sleep-0 {
    pins {
        pinmux = <STM32_PINMUX('G', 14, ANALOG)>,  /* USART1_TX → 模拟 */
                 <STM32_PINMUX('G', 15, ANALOG)>;  /* USART1_RX → 模拟 */
    };
    /* ANALOG 模式下不需要电气配置 */
};
```

如果 DTS 中不提供 sleep 状态，suspend 时引脚保持 default 状态。

### 1.3 STM32_PINMUX 宏详解

定义在 `include/dt-bindings/pinctrl/stm32-pinfunc.h`：

```c
#define PIN_NO(port, line)     (((port) - 'A') * 0x10 + (line))
#define STM32_PINMUX(port, line, mode) (((PIN_NO(port, line)) << 8) | (mode))
```

参数含义：

| 参数 | 示例 | 含义 |
|------|------|------|
| `port` | `'G'` | GPIO 字母（A~K, Z） |
| `line` | `14` | 引脚编号（0~15） |
| `mode` | `AF6` | 功能模式编号 |

`STM32_PINMUX('G', 14, AF6)` 展开计算：

```
PIN_NO('G', 14) = ('G' - 'A') * 16 + 14 = 6 * 16 + 14 = 110
最终值 = (110 << 8) | 0x7 = 0x6E07
```

内核解析时拆分：高 8 位 = 引脚号 110（即 PG14），低 8 位 = AF6。

可用 mode 值：

| 宏 | 值 | 含义 |
|----|----|------|
| `GPIO` | 0x0 | GPIO 模式 |
| `AF0` ~ `AF15` | 0x1 ~ 0x10 | 外设复用功能（编号由 SoC 手册定义） |
| `ANALOG` | 0x11 | 模拟模式 |
| `RSVD` | 0x12 | 保留 |

**注意**：`GPIO` 和 `ANALOG` 的区别——`GPIO` 表示让 pinctrl 把引脚配置为 GPIO 功能（MODER=0b00/0b01），`ANALOG` 表示配置为模拟模式（MODER=0b11）。两者不是同一个东西。

### 1.4 电气配置属性

#### 标准属性（所有 SoC 通用）

| 属性 | 值 | 说明 |
|------|----|------|
| `bias-disable` | 无 | 禁止上下拉 |
| `bias-pull-up` | 无 | 上拉 |
| `bias-pull-down` | 无 | 下拉 |
| `drive-push-pull` | 无 | 推挽输出 |
| `drive-open-drain` | 无 | 开漏输出 |
| `slew-rate` | 0~3 | 输出翻转率（见下文） |

**slew-rate 详解**：控制 GPIO 输出驱动器的翻转速率，对应 OSPEEDR 寄存器的 2-bit 编码。两个工作电压范围各有独立的数据：

| 值 | 3.3V Fmax（30pF） | 1.8V Fmax（30pF） | 驱动电流 | 适用场景 |
|----|------------------|-----------------|---------|---------|
| 0 | 45 MHz | 45 MHz | 6.5 mA | I2C、UART 等低速接口 |
| 1 | 70 MHz | 70 MHz | 10 mA | CAN、MDIO、SPI 等中速接口 |
| 2 | 100 MHz | 100 MHz | 13 mA | MMC 数据线、SPI 高速 |
| 3 | 120 MHz | 120 MHz（10pF 下 250 MHz） | 20 mA | RGMII 时钟、MMC 时钟线（HS400/SDR104 需要此档） |

> 1.8V / 10pF 下 slew-rate=3 可达 **250 MHz**，满足 MMC SDR104（208 MHz）和 HS200 的时序要求。3.3V 下最高 120 MHz，覆盖 RGMII（125 MHz）的需求。数据来源：DS14285 Table 63（3.3V）和 Table 64（1.8V）。

**选型原则**：不是越快越好。slew-rate 越高，信号边沿越陡，电磁辐射（EMI）越大。在满足时序要求的前提下尽量用低值。ATK 板 DTS 中：UART 用 0、MDIO 用 0、CAN TX 用 1、MMC 时钟用 2~3、RGMII 数据/时钟用 3。

#### STM32MP257 专有属性：高速 IO 时序控制

MP257 在标准 GPIO 寄存器（MODER/OTYPER/OSPEEDR/SPEEDR/PUPDR/IDR/ODR/AFR）之外，新增了一个 **ADVCFGR 寄存器（Advanced Configuration Register，偏移 0x48）**，专门用于高速接口的 IO 时序微调。以下三个属性都写在这个寄存器中。

| 属性 | 寄存器位 | 说明 |
|------|---------|------|
| `st,io-retime` | ADVCFGR bit 3 | IO 重定时（D 触发器同步） |
| `st,io-clk-edge` | ADVCFGR bit 1 | IO 时钟沿选择 |
| `st,io-delay` | 独立延时链寄存器 | 粗粒度模拟延时调整 |

---

**st,io-retime**：控制 IO 路径上是否插入一个重定时触发器（D flip-flop）。

```
没有 retime (st,io-retime = <0>)：

  IO pad ──────────────→ 内部总线
                 ↑
          延迟不确定（与 PCB 走线长度相关）

有 retime (st,io-retime = <1>)：

  IO pad ─→ [D 触发器] ─→ 内部总线
                  ↑
             内核时钟同步
          延迟确定 = 1 个时钟周期
```

- `0`：关闭。数据直接通过，延迟与 PCB 走线有关，不固定
- `1`：打开。数据经过 D 触发器再用内核时钟同步输出，延迟确定为一个时钟周期

**用途**：RGMII 接口有多条数据线和时钟线同步传输。PCB 走线不可能完全等长，数据到达时间有偏差。打开 retime 后每个数据用同一个内核时钟重新同步，消除走线偏差。ATK 板 RGMII 的 TX/RX 数据线都打开了 retime。

---

**st,io-clk-edge**：选择 IO 数据在时钟的哪个沿被采样/输出。

- `0`：上升沿
- `1`：下降沿

**用途**：调整时钟相位对齐。比如 RGMII 的 TX 数据线和 GTX_CLK 之间，如果默认上升沿采样有建立时间/保持时间违例，将 `st,io-clk-edge` 设为 `1` 改用下降沿，可能解决时序问题。

在 ATK 板 DTS 中，RGMII 接口同时使用了 retime + clk-edge 组合：

```dts
pins1 {
    pinmux = <STM32_PINMUX('A', 15, AF10)>; /* ETH_RGMII_TXD0 */
    slew-rate = <3>;
    st,io-retime = <1>;    /* 打开重定时 */
    st,io-clk-edge = <1>;  /* 配合下降沿 */
};
```

---

**st,io-delay**：控制一个粗粒度模拟延时链，将信号整体往后推。步长约几百皮秒（具体值由工艺和电压决定，需查 SoC 数据手册）。

```
输入信号 ───┬───→ 正常路径（延迟 ~0）
            │
            └───→ [延时链] ──→ 输出到内部总线
                    ↑
               st,io-delay 控制延迟量
```

**与 st,io-retime 的区别**：

| 手段 | 原理 | 精度 | 适用场景 |
|------|------|------|---------|
| `st,io-retime` | D 触发器 + 时钟同步 | 1 个时钟周期（~ns 级） | RGMII 数据线与时钟对齐 |
| `st,io-delay` | 模拟延迟链 | ~几百 ps/步 | 微调 PCB 走线差异 |

**实际应用**：如果测试发现某一组 RGMII 数据线总是比时钟早到或晚到几个百皮秒，在该引脚的 `st,io-delay` 上加一个步长微调，让所有数据在同一时钟沿到达内部逻辑。

---

**这些属性在谁身上用？** 只在高速接口（RGMII、FMC、DDR 等）中有意义。低速接口（UART、I2C）的信号周期在 µs 级，几 ns 的延迟完全无所谓，不需要配这些属性。

### 1.5 GPIO 功能配置

#### 控制器节点（pin controller 内部子节点）

每个 GPIO bank 是 pinctrl 的子节点，通过属性声明自己是 GPIO 控制器：

```dts
gpioa: gpio@44240000 {
    gpio-controller;           /* 声明：这是一个 GPIO 控制器 */
    #gpio-cells = <2>;         /* 每个引用需要 2 个 cell */
    interrupts = <GIC_SPI 139 IRQ_TYPE_LEVEL_HIGH>;
    gpio-ranges = <&pinctrl 0 0 16>;
    /*          pinctrl  基偏移  基 pin  数量
                       ↓       ↓       ↓
                 GPIOA 的 pin 0→pinctrl pin 0，共 16 个 */
    st,bank-name = "GPIOA";
};
```

`gpio-ranges` 是连接 GPIO 编号与 pinctrl pin 号的桥梁（00-History §5.2 已详述）。

#### Consumer 节点的 GPIO 属性

消费者节点通过 `*-gpios` 属性引用 GPIO：

```dts
/* 格式：<&gpio_controller pin_offset flags> */
led {
    gpios = <&gpioh 4 GPIO_ACTIVE_LOW>;   /* PH4，低电平有效 */
};

/* 多个 GPIO 的场景 */
es8388: es8388@11 {
    spk-con-gpios = <&gpiod 12 GPIO_ACTIVE_LOW>;  /* PD12 */
    hp-det-gpios  = <&gpiod 13 GPIO_ACTIVE_LOW>;  /* PD13 */
};
```

**属性命名规则**：`*-gpios` 中的 `*` 是 `con_id`，驱动层调 `gpiod_get(dev, "spk-con")` 来匹配。如果属性名以 `-gpios` 结尾，驱动传去掉 `-gpios` 的字符串。

**`#gpio-cells = <2>` 的两个 cell**：

| cell | 含义 | 示例 |
|------|------|------|
| 第一 cell | 引脚偏移（bank 内编号） | `4` = GPIOH 的第 4 个引脚 |
| 第二 cell | GPIO 标志（见下面表格） | `GPIO_ACTIVE_LOW` |

**flags 取值**（`include/dt-bindings/gpio/gpio.h`）：

| 宏 | 值 | 含义 |
|----|----|------|
| `GPIO_ACTIVE_HIGH` | 0 | 高电平有效（默认） |
| `GPIO_ACTIVE_LOW` | 1 | 低电平有效 |
| `GPIO_PUSH_PULL` | (1<<2) | 推挽模式 |
| `GPIO_OPEN_DRAIN` | (1<<3) | 开漏模式 |
| `GPIO_PULL_UP` | (1<<4) | 上拉 |
| `GPIO_PULL_DOWN` | (1<<5) | 下拉 |

标志可以通过或运算组合，下面逐个说明每个标志的实际含义。

---

**ACTIVE_HIGH / ACTIVE_LOW（逻辑高低 vs 物理高低）**

这是最常见也最容易搞错的一个。它描述的是**软件上的"有效"状态**对应到硬件引脚上的**实际电平**。

```
逻辑视角：            gpiod_set_value(desc, 1)  →  灯亮
                               ↑ 或 ↓
                      ACTIVE_HIGH / ACTIVE_LOW 控制反转
                               ↑ 或 ↓
硬件视角：            写 ODR 寄存器 → 引脚实际电压

举例：
  GPIO_ACTIVE_HIGH：gpiod_set_value(1) → 引脚输出 3.3V → LED 亮
  GPIO_ACTIVE_LOW： gpiod_set_value(1) → 引脚输出 0V   → LED 亮（反转了！）
```

看 ATK 板上的 LED：

```dts
led {
    gpios = <&gpioh 4 GPIO_ACTIVE_LOW>;
};
```

原理图上这个 LED 的接法是：正极接 3.3V，负极经电阻到 PH4。当 PH4 输出低电平时，电流从 3.3V → LED → 电阻 → PH4，LED 亮。所以"有效"（亮）对应的是低电平，用 `GPIO_ACTIVE_LOW`。

驱动中这样控制：

```c
gpiod_set_value(desc, 1);   /* 亮：gpiolib 看到 ACTIVE_LOW，硬件写 0 */
gpiod_set_value(desc, 0);   /* 灭：gpiolib 看到 ACTIVE_LOW，硬件写 1 */
```

**如果配反了会怎样？** GPIO_ACTIVE_HIGH 和 GPIO_ACTIVE_LOW 弄反，最直接的症状是：`gpiod_set_value(1)` 灯灭，`gpiod_set_value(0)` 灯亮。驱动里找半天 bug，最后发现 DTS 里的标志反了。

另一个例子：按键的 ACTIVE_HIGH 用法：

```dts
gpio-keys {
    button-user {
        gpios = <&gpioh 5 GPIO_ACTIVE_HIGH>;
    };
};
```

按键按下时引脚被拉到 3.3V，所以有效电平是高，用 ACTIVE_HIGH。

**结论**：ACTIVE_HIGH/LOW 不改变硬件电平极性，只改变 gpiod_set_value/get_value 的内部反转逻辑。raw 版本（`gpiod_set_raw_value`）不经过反转，直接写 ODR。

---

**PUSH_PULL / OPEN_DRAIN（输出驱动方式）**

描述 GPIO 作为输出时的驱动器结构：

```
推挽（PUSH_PULL）：
  输出 1 → PMOS 导通，NMOS 截止 → 引脚拉到 VDD
  输出 0 → PMOS 截止，NMOS 导通 → 引脚拉到 GND

开漏（OPEN_DRAIN）：
  输出 1 → NMOS 截止 → 引脚高阻（由外部上拉电阻拉到 VDD）
  输出 0 → NMOS 导通 → 引脚拉到 GND
```

**推挽**：标准输出方式，可以主动驱动高电平和低电平。适合大多数场景（LED、芯片复位、普通 GPIO）。

**开漏**：引脚只能主动拉低，不能主动拉高。适用于以下场景：

- **多设备共享总线**：如 I2C 的 SDA/SCL，多个设备都可拉低总线，没有设备能强拉高——谁先抢到谁就输了。开漏 + 外部上拉实现"线与"（wire-AND）
- **电平转换**：开漏输出加不同电压的上拉电阻，可以让 3.3V 器件和 1.8V 器件在一条总线上通信

ATK 板上的 SDMMC CMD 线就用开漏，因为 CMD 线允许多个设备（SD 卡 + 控制器）驱动：

```dts
sdmmc1_b4_od_pins_a: sdmmc1-b4-od-0 {
    pins3 {
        pinmux = <STM32_PINMUX('E', 2, AF10)>; /* SDMMC1_CMD */
        drive-open-drain;
    };
};
```

而普通 GPIO 输出（LED、复位信号）都用推挽，不需要开漏。

**如果不配这个标志会怎样？** 默认 GPIO 驱动使用推挽模式。如果实际硬件需要开漏（如 I2C）但没配 `GPIO_OPEN_DRAIN`，驱动强行输出高电平时会通过 PMOS 直接拉 VDD，与外部设备想拉低的信号冲突，严重时可能损坏 IO 引脚。

---

**PULL_UP / PULL_DOWN（引脚默认电平）**

当引脚配置为输入（或高阻状态）时，内部上下拉电阻将引脚电平钳制在一个确定的电压，避免"浮空"。

- `GPIO_PULL_UP`：内部 40kΩ 电阻连到 VDD，引脚默认高电平
- `GPIO_PULL_DOWN`：内部 40kΩ 电阻连到 GND，引脚默认低电平
- 不配：引脚浮空（外部有驱动时不需要上下拉）

从数据手册 Table 61 看，上下拉等效电阻约 40kΩ（3.3V/25°C 条件下）。

**实际应用**：

```dts
/* 卡检测引脚：未插入 SD 卡时内部上拉确保高电平，不会被噪声误触发 */
cd-gpios = <&gpioi 8 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;

/* 按键引脚：未按下时上拉为高电平，按下被拉低 */
button {
    gpios = <&gpioh 5 GPIO_ACTIVE_HIGH>;
    /* 外部原理图已有上拉电阻，不需要内部上拉。如果外部没有上拉，配 GPIO_PULL_DOWN */
};
```

**什么时候不配？** 当引脚有外部上拉/下拉电阻时，内部上下拉可以不配（用 `bias-disable`）。内外都配上拉，等效电阻变小（~20kΩ），功耗略增但功能正常。

---

**标志组合示例**

```dts
/* SD 卡检测：低电平有效 + 内部上拉确保浮空时不被误触 */
cd-gpios = <&gpioi 8 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;

/* 摄像头复位：高电平有效 + 推挽输出 */
reset-gpios = <&gpiog 4 (GPIO_ACTIVE_HIGH | GPIO_PUSH_PULL)>;
```

#### 既有 pinctrl 功能又用 GPIO 的混合场景

```dts
&sdmmc1 {
    pinctrl-names = "default", "opendrain", "sleep";
    pinctrl-0 = <&sdmmc1_b4_pins_a>;           /* 数据/时钟线：pinctrl 配为外设功能 */
    pinctrl-1 = <&sdmmc1_b4_od_pins_a>;        /* CMD 线开漏状态 */
    pinctrl-2 = <&sdmmc1_b4_sleep_pins_a>;     /* 休眠 */
    cd-gpios = <&gpioi 8 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;
    /*          ↑ 卡检测引脚：GPIO 功能          */
    bus-width = <4>;
};
```

同一设备节点中，MMC 数据线走 `pinctrl-0`（由 pinctrl 管理），卡检测走 `cd-gpios`（由 gpiolib 管理），互不冲突。

## 2. 内核 API

所有 GPIO 相关 API 声明在 `#include <linux/gpio/consumer.h>` 中，pinctrl 相关在 `#include <linux/pinctrl/consumer.h>`。

### 2.1 GPIO API 速览

#### 获取和释放

| API | 说明 |
|-----|------|
| `devm_gpiod_get(dev, con_id, flags)` | 获取单个 GPIO 描述符（推荐：自动释放） |
| `devm_gpiod_get_optional(dev, con_id, flags)` | 同上，但 con_id 不存在时返回 NULL 而非错误 |
| `devm_gpiod_get_index(dev, con_id, index, flags)` | 获取第 index 个 GPIO（一个属性有多个 phandle 时） |
| `devm_gpiod_get_array(dev, con_id, flags)` | 一次获取一组 GPIO |
| `gpiod_put(desc)` | 释放（非托管版本用） |
| `devm_gpiod_put(dev, desc)` | 释放（devm 版本，一般不需要手动调） |

`con_id` 对应 DTS 属性名去掉 `-gpios` 后缀。例如 `spk-con-gpios` → `con_id = "spk-con"`。当属性名为 `gpios`（无前缀）时传 `NULL`。

`flags` 决定获取后是否立即设置方向和初始值：

| flags | 含义 |
|-------|------|
| `GPIOD_ASIS` | 不改变方向和输出值 |
| `GPIOD_IN` | 设为输入 |
| `GPIOD_OUT_LOW` | 设为输出，初始低电平 |
| `GPIOD_OUT_HIGH` | 设为输出，初始高电平 |
| `GPIOD_OUT_LOW_OPEN_DRAIN` | 开漏输出，初始低 |
| `GPIOD_OUT_HIGH_OPEN_DRAIN` | 开漏输出，初始高 |

#### devm_ 自动释放机制

`devm_` 前缀（device-managed）表示资源由内核接管生命周期。不需要手动释放，**也不需要写 remove 回调或 error path 清理**。

```c
static int my_led_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;

    /* devm_kzalloc：设备 remove 时自动 kfree */
    led = devm_kzalloc(dev, sizeof(*led), GFP_KERNEL);

    /* devm_gpiod_get：设备 remove 时自动 gpiod_put */
    led->gpiod = devm_gpiod_get(dev, NULL, GPIOD_OUT_LOW);
    if (IS_ERR(led->gpiod))
        return dev_err_probe(dev, PTR_ERR(led->gpiod), "get gpio failed\n");
    /* ↑ probe 失败，devm_kzalloc 分配的内存也会自动回滚 */
    /* 不需要写 goto err_free_led */
    return 0;
}
/* 不需要 .remove 回调 */
```

内核的实现原理：`devm_` 函数将资源和释放函数注册到设备的 **devres 链表** 上。设备 remove 时（`devres_release_all`），内核遍历链表逐个释放。probe 中途失败也会触发同样的回滚。

对比非 devm 版本：

```c
/* 非 devm：每个错误路径都要手动 goto 清理 */
static int my_led_probe(...)
{
    led = kzalloc(...);
    if (!led) return -ENOMEM;
    gpiod = gpiod_get(dev, ...);
    if (IS_ERR(gpiod)) {
        ret = PTR_ERR(gpiod);
        goto err_free_led;   /* 手动跳转 */
    }
    return 0;
err_free_led:
    kfree(led);
    return ret;
}
static int my_led_remove(...)
{
    gpiod_put(led->gpiod);   /* 手动释放 */
    kfree(led);
}
```

**规则**：能用 `devm_` 就用 `devm_`。非托管版本（`gpiod_get`/`gpiod_put`）只在某些需要手动控制资源生命周期的场景下使用。

#### 普通获取 vs _optional 获取

`devm_gpiod_get` 和 `devm_gpiod_get_optional` 的唯一区别是 DTS 中属性不存在时的行为：

```c
// devm_gpiod_get：属性不存在 → 返回 ERR_PTR(-ENOENT)
// devm_gpiod_get_optional：属性不存在 → 返回 NULL
```

三种返回值的处理模式：

```c
desc = devm_gpiod_get_optional(dev, "enable", GPIOD_ASIS);

if (IS_ERR(desc))       // ← 错误：属性存在但解析失败（如 phandle 错误）
    return PTR_ERR(desc);

if (!desc)              // ← 正常：这块板子没有这个引脚
    /* 跳过，不做 GPIO 相关操作 */
else                    // ← 正常：拿到 GPIO
    gpiod_direction_output(desc, 1);
```

**实际场景**：同一套驱动支持多个硬件变体，有些板子有使能引脚，有些没有。用 `_optional` 一套代码兼容两块板子：

```dts
/* 板子 A */                         /* 板子 B */
mydev {                              mydev {
    enable-gpios = <&gpioa 5 0>;          /* 不写这个属性 */
};                                   };
```

```c
desc = devm_gpiod_get_optional(dev, "enable", GPIOD_ASIS);
if (IS_ERR(desc))
    return PTR_ERR(desc);

if (desc) {
    /* 板子 A：有使能引脚 */
} else {
    /* 板子 B：没有使能引脚，默认一直使能 */
}
```

`IS_ERR` / `PTR_ERR` / `ERR_PTR` 是内核用指针返回错误码的技巧：

| 函数 | 作用 |
|------|------|
| `IS_ERR(ptr)` | 判断指针是否编码了错误码（不是合法指针也不是 NULL） |
| `PTR_ERR(ptr)` | 从错误指针中提取错误码（如 -ENOMEM） |
| `ERR_PTR(err)` | 将错误码编码为指针返回 |

#### 设置方向

| API | 说明 |
|-----|------|
| `gpiod_direction_input(desc)` | 设为输入 |
| `gpiod_direction_output(desc, value)` | 设为输出并设初始值 |

`gpiod_direction_output` 在 STM32 上会触发后门机制（见 00-History §5.3），通过 pinctrl 写入 MODER 寄存器。

#### 读写值

| API | 说明 |
|-----|------|
| `gpiod_get_value(desc)` | 读电平（已反转 ACTIVE_LOW） |
| `gpiod_set_value(desc, value)` | 写电平（已反转 ACTIVE_LOW） |
| `gpiod_get_raw_value(desc)` | 读电平（不反转，直接读 IDR） |
| `gpiod_set_raw_value(desc, value)` | 写电平（不反转，直接写 ODR） |
| `gpiod_get_value_cansleep(desc)` | 可睡眠上下文中读（I2C GPIO 扩展器等） |
| `gpiod_set_value_cansleep(desc, value)` | 可睡眠上下文中写 |

**ACTIVE_LOW 处理**：设 `gpios = <&gpioh 4 GPIO_ACTIVE_LOW>` 时，`gpiod_set_value(desc, 1)` 在硬件引脚上输出 0（低电平有效，gpiolib 自动反转）。raw 版本不经过反转，直接写 ODR。

**为什么要有 cansleep 版本？**

普通版本（`gpiod_get_value/set_value`）内部使用**自旋锁（spinlock）**保护，不能睡眠。这意味着它可以在中断上下文等关中断的环境中被调用——几十 ns 就读写完寄存器。

但有的 GPIO 控制器不在 SoC 内部，而是挂在 I2C 或 SPI 总线上（如 PCA9535 扩展器）。读写这种 GPIO 需要等 I2C 传输完成，**必然要睡眠**。此时必须用 `_cansleep` 版本，它内部改用**互斥锁（mutex）**，允许睡眠。

```c
/* 如果你不确定 GPIO 是 SoC 内置还是外挂扩展器：*/
if (gpiod_cansleep(desc))
    gpiod_set_value_cansleep(desc, 1);
else
    gpiod_set_value(desc, 1);
```

STM32MP257 上大多数 GPIO 是内置的（直接读写 IDR/ODR 寄存器），用普通版本即可。

**为什么读写 GPIO 需要锁？**

GPIO 控制器寄存器被所有 CPU 共享。以 `gpiod_set_value(desc, 1)` 为例，它的底层操作是**读-改-写**（read-modify-write）：

```
读 ODR 当前值 → 改目标 bit → 写回 ODR
```

没有锁的话，两个 CPU 同时操作同一组寄存器时会出现：

```
CPU0：set PA0=1                     | CPU1：set PA1=0
  read ODR → 0x0000FFFF             |   read ODR → 0x0000FFFF
  modify bit0 → 0x0000FFFF          |   modify bit1 → 0x0000FFFE
  write ODR = 0x0000FFFF            |   write ODR = 0x0000FFFE
                                    ↑
                       CPU0 写的 bit0 被 CPU1 的写操作覆盖了！
                       结果：PA0=0（应该为 1），PA1=0（正确）
```

spinlock 的作用是保证**一个完整的 RMW 序列不被其他执行流打断**——拿到锁的人读完→改完→写完，才轮到下一个人。这在 MCU 上不是一个问题（单核，不会被真正的并行打断），但在 Cortex-A35 双核的 MP257 上是必须的。

#### 中断

| API | 说明 |
|-----|------|
| `gpiod_to_irq(desc)` | GPIO 描述符转中断号 |

**为什么需要这个函数？**

CPU 的中断控制器（GIC）不认识"GPIO 47 号引脚"。它认识的是**中断号**（IRQ number）。要从 GPIO 引脚触发中断，你需要在 GPIO 控制器和中断控制器之间建立映射。

每个 GPIO bank 内部有一个中断控制器（STM32 上通过 EXTI 机制实现）。`gpiod_to_irq` 做的就是：**找到这个 GPIO 引脚对应的中断号**，让你可以用标准的 `request_irq` / `devm_request_threaded_irq` 来注册中断处理函数。

```c
/* 没有 gpiod_to_irq，你只能轮询——但那是低效的 */
while (1) {
    if (gpiod_get_value(key))
        handle_key();
    udelay(1000);  /* 浪费 CPU */
}

/* 有 gpiod_to_irq，你可以等中断——CPU 闲着，不浪费 */
int irq = gpiod_to_irq(key_desc);
devm_request_threaded_irq(dev, irq, NULL, key_isr,
                           IRQF_TRIGGER_RISING, "key", dev);
/* 按键按下 → 硬件触发中断 → key_isr 被调用 → 执行完后 CPU 继续休眠 */
```

**STM32 上的实际路径**：

```
gpiod_to_irq(desc)
  → chip->to_irq(chip, offset)          ← stm32_gpio_to_irq()
    → irq_create_fwspec_mapping(&fwspec)  ← 在 GIC 中分配中断号
      → 返回 Linux IRQ number              ← 比如 48
```

GPIO 控制器的 DTS 中声明了它的中断控制器能力：

```dts
gpioa: gpio@44240000 {
    gpio-controller;
    interrupt-controller;           /* 这个 bank 同时也是中断控制器 */
    #interrupt-cells = <2>;         /* 每个引用需要 2 个 cell */
    interrupts = <GIC_SPI 139 IRQ_TYPE_LEVEL_HIGH>;  /* 到 GIC 的连接 */
};
```

- `interrupt-controller`：声明这个 GPIO bank 是一个中断控制器
- `interrupts`：这个 bank 的中断输出线连到 GIC 的哪个中断号（139）
- `#interrupt-cells = <2>`：两个 cell，分别是 GPIO 偏移和触发类型

```dts
/* 设备节点中直接引用 GPIO 作为中断控制器 */
touch@14 {
    interrupt-parent = <&gpiob>;          /* 中断通过 GPIOB 路由 */
    interrupts = <2 IRQ_TYPE_EDGE_RISING>; /* GPIOB 的 pin 2，上升沿触发 */
};
```

**`gpiod_to_irq` 的返回值**：

- `> 0`：有效的 Linux IRQ number（可以使用 request_irq 注册）
- `-ENXIO`：这个 GPIO 不支持中断（没有 `to_irq` 回调）
- `-EPROBE_DEFER`：GPIO 的 irqchip 还没注册好，需要等 probe 重试

### 2.2 Pinctrl Consumer API 速览

| API | 说明 |
|-----|------|
| `devm_pinctrl_get(dev)` | 获取 pinctrl 句柄 |
| `pinctrl_lookup_state(p, name)` | 查找某个状态（对应 DTS 的 `pinctrl-names`） |
| `pinctrl_select_state(p, state)` | 应用状态 |
| `devm_pinctrl_get_select(dev, name)` | 便捷函数：get + lookup + select 一次完成 |

大多数驱动不需要调这些 API——`pinctrl_bind_pins()` 在 probe 时自动解析 `pinctrl-0` 并应用 "default" 状态。只有需要在运行时切换引脚功能时才手动调用。

使用场景：

```c
/* 拿到 pinctrl 句柄 */
struct pinctrl *p = devm_pinctrl_get(dev);

/* 查找 DTS 中定义的 "sleep" 状态 */
struct pinctrl_state *s = pinctrl_lookup_state(p, "sleep");

/* 应用这个状态（引脚立刻切到该配置）*/
pinctrl_select_state(p, s);
```

### 2.3 场景一：控制一个 LED（GPIO 输出）

```dts
led {
    gpios = <&gpioh 4 GPIO_ACTIVE_LOW>;
};
```

```c
#include <linux/gpio/consumer.h>
#include <linux/platform_device.h>
#include <linux/module.h>

struct my_led {
    struct gpio_desc *gpiod;
};

static int my_led_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct my_led *led;

    led = devm_kzalloc(dev, sizeof(*led), GFP_KERNEL);
    if (!led)
        return -ENOMEM;
    platform_set_drvdata(pdev, led);

    /* 步骤 1+2：拿描述符 + 设为输出低电平（一次完成）*/
    led->gpiod = devm_gpiod_get(dev, NULL, GPIOD_OUT_LOW);
    if (IS_ERR(led->gpiod))
        return dev_err_probe(dev, PTR_ERR(led->gpiod), "get gpio failed\n");

    /* 之后随时可以控制亮灭 */
    gpiod_set_value(led->gpiod, 1);  /* 亮 */
    gpiod_set_value(led->gpiod, 0);  /* 灭 */

    return 0;
    /* devm 自动释放 gpiod，不需要手动清理 */
}
```

### 2.4 场景二：读一个按键（GPIO 输入 + 中断）

```dts
gpio-keys {
    compatible = "gpio-keys";
    button-user {
        gpios = <&gpioh 5 GPIO_ACTIVE_HIGH>;
        linux,code = <BTN_1>;
    };
};
```

```c
/* probe 中：拿描述符（设输入）→ 转中断 → 注册中断处理 */
struct gpio_desc *key;
int irq, ret;

key = devm_gpiod_get(dev, NULL, GPIOD_IN);
if (IS_ERR(key))
    return PTR_ERR(key);

/* 读当前电平 */
int val = gpiod_get_value(key);

/* 转中断号 */
irq = gpiod_to_irq(key);
if (irq < 0)
    return irq;

/* 注册中断 */
ret = devm_request_threaded_irq(dev, irq, NULL, button_isr,
                                 IRQF_TRIGGER_RISING | IRQF_ONESHOT,
                                 "user-key", dev);

static irqreturn_t button_isr(int irq, void *data)
{
    struct device *dev = data;
    struct gpio_desc *key = dev_get_drvdata(dev);
    dev_info(dev, "key value=%d\n", gpiod_get_value(key));
    return IRQ_HANDLED;
}
```

**`_optional` 变体**：当 GPIO 在有些板子上存在、有些板子上不存在时：

```c
desc = devm_gpiod_get_optional(dev, "enable", GPIOD_ASIS);
if (IS_ERR(desc))
    return PTR_ERR(desc);
if (!desc)
    /* 这个板子没有 enable 引脚，跳过即可 */
```

### 2.5 场景三：suspend/resume 切换 pin state（Pinctrl）

```dts
&sdmmc1 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&sdmmc1_b4_pins_a>;        /* 正常工作 */
    pinctrl-1 = <&sdmmc1_b4_sleep_pins_a>;  /* suspend（ANALOG 省电）*/
};
```

```c
#include <linux/pinctrl/consumer.h>

struct mmc_priv {
    struct pinctrl *p;
    struct pinctrl_state *pins_default;
    struct pinctrl_state *pins_sleep;
};

static int mmc_probe(struct platform_device *pdev)
{
    struct mmc_priv *priv = ...;

    /* "default" 由 pinctrl_bind_pins() 自动配好 */
    /* 这里只保存 sleep 句柄，供 suspend 时切换 */

    priv->p = devm_pinctrl_get(dev);
    priv->pins_default = pinctrl_lookup_state(priv->p, "default");
    priv->pins_sleep   = pinctrl_lookup_state(priv->p, "sleep");
}

static int mmc_suspend(struct device *dev)
{
    struct mmc_priv *priv = dev_get_drvdata(dev);
    return pinctrl_select_state(priv->p, priv->pins_sleep);
}

static int mmc_resume(struct device *dev)
{
    struct mmc_priv *priv = dev_get_drvdata(dev);
    return pinctrl_select_state(priv->p, priv->pins_default);
}
```

## 3. 用户态操作

驱动开发者通过内核 API 控制 GPIO（§2），但调试时需要临时测一个引脚、写一个简单的测试程序——不可能每次都写一个内核模块再编译烧录。用户态操作就是给这种场景用的：在 shell 里直接控制 GPIO，验证硬件是否正常。

### 3.1 libgpiod（推荐）

#### 为什么需要它

过去（Linux 4.4 之前），用户态 GPIO 只有 sysfs 接口。核心问题是：**导出 → 设方向 → 写值** 是三步独立的文件操作。

```shell
# sysfs 方式：三步操作，中间可能被其他进程打断
echo 47 > /sys/class/gpio/export          # 步骤 1
echo out > /sys/class/gpio/gpio47/direction  # 步骤 2
echo 1 > /sys/class/gpio/gpio47/value       # 步骤 3
# 如果步骤 1 和 2 之间有别的进程 unexport 了同一个 GPIO → 崩溃
```

Linux 4.4 引入了 GPIO chardev（`/dev/gpiochipN`），将所有操作封装为一次 ioctl 调用——原子、安全、支持中断和批量操作。

libgpiod 是对 chardev ioctl 的封装，提供了**命令行工具**（快速测试）和 **C API**（写用户态程序）。

> **Buildroot 中启用 libgpiod 工具**：默认配置只编译库，不安装命令行工具。需要在 `make menuconfig` 中手动开启：
>
> ```
> Target packages → Libraries → Hardware handling → libgpiod
>   → [*] libgpiod
>   → [*]   install tools       ← 选上才有 gpiodetect/gpioset/gpiomon 等命令
> ```
>
> 配置后执行 `make libgpiod-rebuild && make` 重新编译烧录。

#### 常用命令

```shell
# 查看系统上有哪些 GPIO 控制器
$ gpiodetect
gpiochip0 [GPIOA] (16 lines)
gpiochip1 [GPIOB] (16 lines)
gpiochip2 [GPIOC] (14 lines)
gpiochip3 [GPIOD] (16 lines)
gpiochip4 [GPIOE] (16 lines)
gpiochip5 [GPIOF] (16 lines)
gpiochip6 [GPIOG] (16 lines)
gpiochip7 [GPIOH] (14 lines)
gpiochip8 [GPIOI] (12 lines)
gpiochip9 [GPIOZ] (10 lines)
```

注意：**每个 GPIO bank 是一个独立的 chip**（不是所有 bank 合并为一个）。所以操作 PA5 要用 `gpiochip0`，操作 PH4 要用 `gpiochip7`。

```shell
# 查看某个控制器的所有 line 状态（取 GPIOH 为例，LED 和按键都在这里）
$ gpioinfo gpiochip7
gpiochip7 - 14 lines:
        line   0:      unnamed       unused   input  active-high
        line   1:      unnamed       unused   input  active-high
        line   2:        "PH2"       unused   input  active-high
        line   3:        "PH3"       unused   input  active-high
        line   4:        "PH4" "red:heartbeat" output active-low [used]
        line   5:        "PH5"   "User-Key"   input  active-high [used]
        line   6:        "PH6"       unused   input  active-high
        ...

# 各列含义：line 编号 | "引脚名" | 标签 | 方向 | 有效电平 | [占用者]
```

```shell
# 控制一个空闲的 GPIO 输出（以 PA5 为例，它当前是 unused）
$ gpioset gpiochip0 5=1            # 设置 PA5 输出高电平
$ gpioset gpiochip0 5=0            # 设置 PA5 输出低电平
$ gpioset gpiochip0 5=1 6=0 7=1   # 同时设置多路（原子操作）
```

```shell
# 读取一个 GPIO 输入
$ gpioget gpiochip0 3
1                                  # 返回 0 或 1
```

```shell
# 监听中断事件
$ gpiomon --num-events=5 --both-edges gpiochip0 5
event:  RISING EDGE  timestamp: 8231890974887   # 上升沿
event:  FALLING EDGE timestamp: 8231891001221   # 下降沿
```

**实际场景：验证 LED 硬件（PH4）**

LED 已被内核心跳驱动占用。想验证硬件，不能直接 `gpioset`（返回设备忙），需要通过 LED class 控制：

```shell
# 1. 查出 LED 在哪个 bank、line 几
$ gpioinfo gpiochip7
        line   4:        "PH4" "red:heartbeat" output active-low [used]
#                                        ↑ 标签是 "red:heartbeat"

# 2. 通过 LED class 控制亮灭
$ ls /sys/class/leds/
red:heartbeat

$ cat /sys/class/leds/red:heartbeat/trigger
none usb-gadget usb-host rfkill-any heartbeat [cpu] ...
#                                                ↑ 当前是 cpu trigger

$ echo heartbeat > /sys/class/leds/red:heartbeat/trigger   # 切到心跳
$ echo none > /sys/class/leds/red:heartbeat/trigger        # 关闭 trigger
$ echo 1 > /sys/class/leds/red:heartbeat/brightness        # 手动亮
$ echo 0 > /sys/class/leds/red:heartbeat/brightness        # 手动灭
```

如果想通过 `gpioset` 强制控制，必须先让内核释放：
```shell
$ echo none > /sys/class/leds/red:heartbeat/trigger   # 关闭自动 trigger
$ echo 0 > /sys/class/leds/red:heartbeat/brightness   # 先灭
# 然后 gpioset gpiochip7 4=... 就可以用了
```

**实际场景：调试按键中断（PH5）**

```shell
# 1. 查出按键在哪个 bank
$ gpioinfo gpiochip7
        line   5:        "PH5"   "User-Key"   input  active-high [used]

# 2. 先用 gpioget 确认电平变化正常
$ gpioget gpiochip7 5              # 不按 → 0？按 → 1？
#                                   如果都是 0 检查上拉或原理图

# 3. 再用 gpiomon 确认中断能触发
$ gpiomon --num-events=3 gpiochip7 5
event:  RISING EDGE  timestamp: ...  # 按下的瞬间
event:  FALLING EDGE timestamp: ...  # 松开的瞬间
```

#### C 语言 API（libgpiod v1）

ATK 板当前使用 libgpiod **v1.6.3**。C API 使用方法如下：

```c
#include <gpiod.h>

int main(void)
{
    struct gpiod_chip *chip;
    struct gpiod_line *line;

    /* 打开 GPIOA bank（PH4 在 gpiochip7，这里以 gpiochip0 为例）*/
    chip = gpiod_chip_open("/dev/gpiochip0");
    if (!chip)
        return -1;

    /* 获取 line 5（PA5） */
    line = gpiod_chip_get_line(chip, 5);
    if (!line)
        return -1;

    /* 请求为输出，初始高电平 */
    if (gpiod_line_request_output(line, "my-app",
                                  GPIOD_LINE_REQUEST_FLAG_OUTPUT) < 0)
        return -1;

    /* 控制 */
    gpiod_line_set_value(line, 1);   // 亮
    sleep(1);
    gpiod_line_set_value(line, 0);   // 灭

    /* 清理 */
    gpiod_line_release(line);
    gpiod_chip_close(chip);
    return 0;
}
```

编译：`gcc -o gpio-test gpio-test.c -lgpiod`

> **libgpiod v1 vs v2 差异**：ATK 板当前是 v1.6.3（Buildroot 2024.x 起默认 v2）。主要区别是 flags 宏名字不同——v1 用 `GPIOD_LINE_REQUEST_FLAG_OUTPUT`，v2 用 `GPIOD_LINE_ACTIVE_STATE_HIGH`。如果你的板子升级到了 v2，按 v2 语法改写即可。

### 3.2 sysfs 接口（已知会淘汰）

#### 为什么还有

在 libgpiod 出现之前（2008~2015），这是唯一的用户态 GPIO 接口。`Documentation/ABI/obsolete/sysfs-gpio` 中已标记为 obsolete，但考虑到兼容性，很多内核仍然通过 `CONFIG_GPIO_SYSFS` 保留它。

#### 怎么用

```shell
# 1. 导出 GPIO，告诉内核我要操作这个引脚
echo 47 > /sys/class/gpio/export

# 2. 设方向
echo out > /sys/class/gpio/gpio47/direction   # 输出
# echo in > /sys/class/gpio/gpio47/direction   # 输入

# 3. 写值/读值
echo 1 > /sys/class/gpio/gpio47/value          # 输出高
cat /sys/class/gpio/gpio47/value               # 读取当前电平

# 4. 中断监听（只能 poll，不能精确知道触发时间）
echo rising > /sys/class/gpio/gpio47/edge      # 配置上升沿触发

# 5. 用完释放
echo 47 > /sys/class/gpio/unexport
```

#### 为什么不推荐

| 问题 | 说明 |
|------|------|
| **非原子** | export→direction→value 三步独立，中间可能被抢占 |
| **中断弱** | 只能 `poll()`，无时间戳、无去抖、无法检测事件丢失 |
| **不安全** | 全局 export，任何进程都能导出任何 GPIO，无归属管理 |
| **批量低效** | 一次操作一个 GPIO，无法原子设置多路 |

**一句话**：新设计不要用 sysfs，已经用了尽快迁移到 libgpiod。

## 4. Debug 手段

硬件不工作是最常见的场景。Debug 就是搞清楚**这个引脚当前被谁占着、配成了什么功能**。debugfs 提供了两个门。

### 4.1 /sys/kernel/debug/gpio：查 GPIO 使用状态

#### 为什么需要它

你的驱动调了 `devm_gpiod_get` 没报错，但 GPIO 不工作。第一个问题：**这个 GPIO 被谁占着？方向对吗？**

```shell
$ cat /sys/kernel/debug/gpio
GPIOs 0-15, platform/44240000.gpio, gpio@44240000:
 gpio-2   ( |uart1_rx              ) in  hi
 gpio-3   ( |uart1_tx              ) out hi
 gpio-5   ( |heartbeat-led         ) out lo

GPIOs 16-31, platform/44250000.gpio, gpio@44250000:
 gpio-18  ( |reset                 ) out lo

GPIOs 176-183, platform/46200000.gpio, gpio@46200000:
 gpio-176 ( |spi8 CS               ) out hi
```

各列含义：

| 列 | 示例 | 说明 |
|------|------|------|
| `gpio-N` | `gpio-5` | GPIO core 内部全局编号 |
| `(标签)` | `heartbeat-led` | 申请时传入的 label（可反查是哪个驱动） |
| `in`/`out` | `out` | 当前方向 |
| `hi`/`lo` | `lo` | 当前读到的电平 |

#### 实际排查

**LED 不亮，查 GPIO debugfs：**

```shell
$ cat /sys/kernel/debug/gpio | grep gpio-5
 gpio-5   ( |heartbeat-led         ) out lo
```

正常。`GPIO_ACTIVE_LOW` 的 LED，`out lo` 表示软件写 0，硬件输出 3.3V。如果显示 `out hi` 反而可能是 ACTIVE_LOW 配反了。

**标签不对：**

```shell
$ cat /sys/kernel/debug/gpio | grep gpio-5
 gpio-5   ( |touch-reset           ) out hi
#          ↑ 你的 LED 驱动申请的 gpio-5，但标签是 touch-reset
#          说明触控驱动先占了这个引脚，你的 LED 驱动应该申请失败
#          去 dmesg 查 "pin already requested"
```

**查空闲引脚：**

```shell
$ cat /sys/kernel/debug/gpio | grep unused
 gpio-12 ( |"PA12"                ) unused
#          ↑ "unused" 表示没人占。可以用 libgpiod 直接测试
```

### 4.2 /sys/kernel/debug/pinctrl/：查引脚复用状态

#### 为什么需要它

debugfs GPIO 只看到 GPIO 层面的占用。但引脚可能被**复用功能**占着——比如 PG14 配成了 `usart1_tx`（AF6），此时 `gpiod_get` 可能不报错（STM32 strict=0），但写 ODR 不产生效果（MODER 是 AF 模式）。

debugfs pinctrl 回答：**这个引脚当前被配成了什么功能？**

```shell
# 查看系统上有哪些 pinctrl 控制器
$ ls /sys/kernel/debug/pinctrl/
44240000.pinctrl          # 主域（GPIOA~GPIOK）
46200000.pinctrl          # 安全域（GPIOZ）
```

```shell
# 查看某个控制器的全部引脚复用状态
$ cat /sys/kernel/debug/pinctrl/44240000.pinctrl/pinmux-pins
Pinmux table:

group: usart1-0
  pin 110 (PG14): function usart1 group usart1-0  (3 users)
  pin 111 (PG15): function usart1 group usart1-0  (3 users)
group: sdmmc1-b4-0
  pin 100 (PE4):  function sdmmc1 group sdmmc1-b4-0  (2 users)
  ...
```

```shell
# 查看已注册的所有功能组
$ cat /sys/kernel/debug/pinctrl/44240000.pinctrl/pinmux-functions
function: usart1, groups: [ usart1-0 ]           # USART1 有 1 组引脚
function: i2c3, groups: [ i2c3-0 ]               # I2C3 有 1 组
function: sdmmc1, groups: [ sdmmc1-b4-0 ]         # SDMMC1 有 1 组 4bit
function: sdmmc2, groups: [ sdmmc2-b4-0 sdmmc2-d47-0 ]  # SDMMC2 有 2 组
```

#### 实际排查

**I2C 不工作，怀疑引脚被 GPIO 误配：**

```shell
$ cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep -E "PG1|PG2"
  pin 97 (PG1): function i2c3 group i2c3-0  (1 users)
  pin 98 (PG2): function i2c3 group i2c3-0  (1 users)
#                ↑ 正常：PG1/PG2 配为 i2c3 功能
#                如果这里不是 function i2c3，说明 DTS 配错了外设
```

**两个驱动争同一个引脚：**

```shell
$ cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep "PG14"
  pin 110 (PG14): function usart1 group usart1-0  (3 users)
#                ↑ 如果你以为 PG14 应该是 SPI2，但实际显示 usart1
#                说明 DTS 中两个外设都配了同个引脚，另一个在 dmesg 里有冲突日志
```

### 4.3 完整排查流程

```
引脚不工作
  │
  ├─ 1. /sys/kernel/debug/gpio（查 GPIO 占用）
  │     $ cat /sys/kernel/debug/gpio | grep <你的引脚>
  │     │
  │     ├─ "unused"            → 驱动没申请到，或 DTS 没配 gpios 属性
  │     ├─ "(你的驱动标签)"     → gpiolib 正常，查 pinctrl
  │     └─ "(其他驱动)"         → 被别的驱动占了，检查 dmesg 冲突
  │
  └─ 2. /sys/kernel/debug/pinctrl（查引脚复用功能）
        $ cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep <引脚名>
        │
        ├─ "function gpio"     → 正常（后门机制切成了 GPIO 功能）
        ├─ "function usart1"   → 被配成了外设功能（如果期望 GPIO 就是问题）
        └─ 无输出              → 引脚没配，保持复位 ANALOG 模式
```

**实际例子：UART1 不工作**

```shell
# 第 1 步：查 GPIO
$ cat /sys/kernel/debug/gpio | grep PG14
（无输出）
# PG14 没被 GPIO 占用 → 问题不在 gpiolib，在 pinctrl

# 第 2 步：查 pinctrl
$ cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep PG14
  pin 110 (PG14): function spi2 group spi2-0  (1 users)
#                    ↑ 不是 usart1！是 spi2！
# DTS 里 PG14 被配成了 SPI2 功能，但 UART 驱动以为它是 USART1
# 解决：检查 DTS 中 uart1 和 spi2 的 pinctrl-0 配置
```

**dmesg 中的冲突提示：**

```
pin PG14 already requested by usart1; cannot claim for spi2
# 内核启动时出现这个 → PG14 在两处 pinctrl-0 中被引用
# 一个驱动（usart1）先占，另一个（spi2）在 probe 时被 pinctrl 拦截
```
## 5. 总结

写 DTS 还是写代码？记住三条原则：

1. **外设功能走 pinctrl-0**：UART/I2C/MMC/SPI 等在 DTS 配 `pinctrl-0`，驱动里不用碰 pinctrl API，probe 时自动配置。
2. **GPIO 功能走 gpios**：LED/按键/复位等写 `*-gpios` 属性，驱动里调 `devm_gpiod_get` + `gpiod_set_value`，后门自动处理 MODER（STM32）。
3. **调试用 libgpiod + debugfs**：先 `gpioset` 排除硬件问题，再查 `/sys/kernel/debug/gpio` 和 `/sys/kernel/debug/pinctrl/*/pinmux-pins` 定位软件问题。

三种报错最常见：

```
pin XX already requested by YY     → DTS 中两个外设配了同一个引脚
gpiod_get 返回 -EPROBE_DEFER      → GPIO 控制器还没 probe，内核会重试
GPIO_ACTIVE_HIGH/LOW 配反         → gpiod_set_value(1) 灯灭，gpiod_set_value(0) 灯亮
```
