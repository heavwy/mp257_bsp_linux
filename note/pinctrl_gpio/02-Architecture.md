# 核心数据结构与设计原理

> 两个子系统的静态结构（不涉及代码流程，那是后面文档的事）。
>
> **字数**：约 15000 字 · **建议阅读时间**：50~80 分钟

## 1. 设计思想：为什么需要两套数据结构

### 1.1 同一个引脚，两种视角

Pinctrl 和 GPIO 是 Linux 对同一组物理引脚的**两个不同视角的抽象**。它们管的根本不是同一件事：

| 视角 | 管什么 | 典型接口 |
|------|--------|---------|
| **Pinctrl** | 这个引脚现在当什么用——UART_TX？SPI_SCLK？GPIO？ | `pinctrl_select_state()` |
| **GPIO** | 这个引脚当 GPIO 用时，电平是高是低？方向是输入还是输出？ | `gpiod_get_value()` / `gpiod_set_value()` |

### 1.2 为什么软件里必须拆成两套

拆成两套的根本原因不是"设计模式"，而是**硬件改了也不会通知你**。

以 STM32 的 MODER 寄存器为例：你在另一个核（M33）或者 bootrom 里把 PG14 的 MODER 改成了 0b10（Alternate Function），A35 核上跑的 Linux 并不知道。如果不做冲突检测，A35 的另一个驱动又去请求 PG14 当 GPIO 用，结果是两个驱动同时写了同一组寄存器——不可预测。

所以 Linux 的设计是：

1. **Pinctrl** 管"这组引脚当前配了什么功能"——通过 `pin_desc.mux_owner` 做冲突检测
2. **GPIO** 管"当 GPIO 用时数据怎么读写"——通过 `gpio_desc.flags` 标记占用状态
3. 两个子系统通过 **pinctrl_gpio_range** 交换编号信息，通过 **后门机制**（`gpio_chip.request` / `gpio_set_direction`）完成引脚模式切换

### 1.3 硬件差异对软件设计的决定性影响

不同 SoC 的硬件实现方式不同，决定了软件路径不同：

| 类型 | 代表 | Pinctrl 和 GPIO 硬件关系 | 软件影响 |
|------|------|-------------------------|---------|
| **合体架构** | STM32 | GPIO 寄存器兼任 Pinctrl（MODER/AFR + ODR/IDR 在同一地址空间） | 同一驱动注册 pinctrl 和 gpio_chip，后门可以直接写寄存器 |
| **分离架构** | i.MX6ULL | IOMUXC 和 GPIO 是两套独立硬件模块（不同地址空间） | 两个独立驱动，后门只能做冲突检测不能切模式 |

硬件决定了软件设计，但框架是统一的。§2 和 §3 将分别介绍两套独立的数据结构，§4 介绍连接它们的桥梁。

> 本章只讲数据结构（静态），不讲代码流程（动态）。probe 流程、状态切换、后门调用链留到 03-Source-Analysis.md。

---
## 2. Pinctrl 侧：数据结构与三大功能

Pinctrl 子系统管理引脚的三个核心问题：这个引脚能当什么用？当前被配成了哪个外设的功能？电气参数是多少？下面先介绍解决这三个问题所需的数据结构，再说明它们如何协作。

### 2.1 三大功能概述

Pinctrl 管理三件事，每个功能对应一个独立的 ops 结构体：

| # | 功能 | ops 结构体 | 管什么 |
|---|------|-----------|--------|
| 1 | **引脚枚举** | `struct pinctrl_ops` | 这个控制器管哪些引脚、怎么分组的 |
| 2 | **引脚复用** | `struct pinmux_ops` | 引脚当前配成了哪个外设功能 |
| 3 | **引脚配置** | `struct pinconf_ops` | 引脚的电气参数（上下拉、驱动强度、slew rate） |


### 2.2 分层架构：消费者层 → Core 层 → 控制器驱动层

Pinctrl 子系统和 MMC 子系统一样，采用**核心层 + 控制器驱动**的分离架构：

```
┌──────────────────────────────────────────────────────────────────────┐
│  消费者层 (Consumer / Client drivers)                   pinctrl API  │
│                                                                      │
│  ┌──────────────────────────────────────────────┐                   │
│  │  设备驱动（UART/I2C/MMC/LED 驱动等）            │                   │
│  │    ● pinctrl_bind_pins() — probe 时自动调用    │                   │
│  │    ● pinctrl_get / pinctrl_select_state       │                   │
│  │    ● gpiod_get — 后门机制调用 pinctrl          │                   │
│  └──────────────────────────────────────────────┘                   │
│          │ 通过 pinctrl_desc 中的 ops 间接调用                       │
│          ▼                                                          │
├──────────────────────────────────────────────────────────────────────┤
│  PINCTRL CORE 层 (通用逻辑)                      drivers/pinctrl/   │
│                                                                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────┐             │
│  │ core.c   │ │ pinmux.c │ │ pinconf.c│ │devicetree.c│             │
│  │          │ │          │ │          │ │            │             │
│  │ 注册管理  │ │ 复用切换  │ │ 参数配置  │ │ DTS 解析   │            │
│  │ 冲突检测  │ │ pin_req/ │ │ config   │ │ pinctrl_   │            │
│  │ pinctrl_ │ │ free     │ │ apply    │ │ map 生成   │            │
│  │ select   │ │          │ │          │ │            │            │
│  └──────────┘ └──────────┘ └──────────┘ └────────────┘             │
│          │                                                          │
│          │  调用 pinctrl_dev->desc->pctlops/pmxops/confops          │
│          ▼                                                          │
├──────────────────────────────────────────────────────────────────────┤
│  PIN CONTROLLER 驱动层 (硬件操作)                 drivers/pinctrl/  │
│                                                                      │
│  ┌──────────────────────────────────────────────┐                   │
│  │  pinctrl-stm32.c / pinctrl-imx.c / ...       │                   │
│  │    ● 实现 pinctrl_ops / pinmux_ops / confops │                   │
│  │    ● 操作硬件寄存器                             │                   │
│  │    ● stm32_pmx_set_mode() 写 MODER + AFR     │                   │
│  │    ● probe 时注册 pinctrl_desc 到 core 层     │                   │
│  └──────────────────────────────────────────────┘                   │
│          │                                                          │
│          ▼  写寄存器                                                 │
│  ┌──────────────────────────────────────────────┐                   │
│  │  STM32MP257 GPIO 硬件（44240000 地址空间）     │                   │
│  │  MODER · AFRL/H · OSPEEDR · PUPDR · OTYPER  │                   │
│  └──────────────────────────────────────────────┘                   │
└──────────────────────────────────────────────────────────────────────┘
```

各层定位：**消费者层**调用 pinctrl API 申请引脚，不碰底层硬件；**Core 层**管理注册、冲突检测、DTS 解析，不直接操作寄存器；**驱动层**实现三个 ops，操作硬件寄存器。三层之间不能越级调用，全部通过 `pinctrl_desc` 中的 ops 间接调用。

### 2.3 数据结构关系图：结构体之间的指针指向

上面是"架构层次"（软件模块的分层），这里是"数据结构关系"（每个结构体内部的字段怎么指向其他结构体）。Pinctrl 子系统的数据结构分为三个维度：**控制器的描述与注册**、**运行时的管理单元**、**操作接口与数据管道**。它们之间通过指针连接：

```
                   注册调用 devm_pinctrl_register()
                   ┌──────────────────────────────────────────────────┐
                   │            pinctrl_desc（只读模板）               │
                   │  .name = "44240000.pinctrl"                     │
                   │  .pins → pinctrl_pin_desc[176]  ← 引脚列表      │
                   │  .npins = 176                                    │
                   │  .pctlops → stm32_pctrl_ops  ← 三个 ops 入口     │
                   │  .pmxops  → stm32_pmx_ops                       │
                   │  .confops → stm32_pconf_ops                     │
                   └──────────┬───────────────────────────────────────┘
                              │ pinctrl_register() 内部根据 desc 创建
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      pinctrl_dev（运行时对象）                        │
│                                                                      │
│  .desc ───────────────────────────────────────────────→ pinctrl_desc │
│  .pin_desc_tree（红黑树，key = pin 号）                               │
│      ├── key=0   → pin_desc{name="PA0",  mux_owner, gpio_owner}    │
│      ├── key=1   → pin_desc{name="PA1",  mux_owner, gpio_owner}    │
│      └── key=110 → pin_desc{name="PG14", mux_owner="uart1", ...}   │
│  .gpio_ranges（链表）—— 桥梁 → 见 §4                               │
│      ├── range[0]: base=0,  pin_base=0,  gc=&gpioa_chip           │
│      ├── range[1]: base=16, pin_base=16, gc=&gpiob_chip           │
│      └── range[2]: base=32, pin_base=32, gc=&gpioc_chip           │
│  .groups[]（group_desc 数组）                                        │
│      ├── [0]: name="PA0",   pin=0                                │
│      ├── [1]: name="PA1",   pin=1                                │
│      └── [110]: name="PG14", pin=110                              │
│  .states（pinctrl 状态链表）                                        │
│      ├── state "default": settings[] → set_mux + pinconf          │
│      └── state "sleep":   settings[] → ANALOG                     │
│  .driver_data → stm32_pinctrl{groups, banks, clks, ...}           │
│  .dev → struct device{of_node, ...}                                │
└──────────────────────────────────────────────────────────────────────┘

```

各结构体的详细说明：

**`pinctrl_desc`**——厂商驱动的注册凭证。包含三个 ops 指针、引脚描述表、自定义参数等。填写后调用 `devm_pinctrl_register()` 提交给 core 层，core 层根据它创建运行时对象。（→ 详细分析：§2.4 pinctrl_desc）

**`pinctrl_dev`**——core 层注册时创建的运行时对象。包含 `pin_desc_tree`（冲突检测）、`gpio_ranges`（桥梁）、`groups[]`（引脚组）、`states`（状态链表）等运行时状态。驱动开发者通过它调 core 层辅助 API。（→ 详细分析：§2.5 pinctrl_dev）

**`pin_desc`**——每个物理引脚一个，记录 `mux_owner`（复用功能所有者）和 `gpio_owner`（GPIO 所有者）。冲突检测的核心——每次 `pin_request()` 查 `mux_usecount` 和 `owner` 判冲突。由 core 层在注册时自动创建。（→ 详细分析：§2.6 pin_desc）

**`group_desc`**——引脚组描述。包含组名和该组包含的 pin 号数组。由 pinctrl_ops 的三个回调在注册时创建，存储在 `pinctrl_dev->groups[]` 中。（→ 详细分析：§2.6 group_desc）

**`pinctrl_ops`**——引脚枚举操作集。三个回调回答：有多少组？第 n 组叫什么？第 n 组有哪些引脚？（→ 详细分析：§2.7 pinctrl_ops）

**`pinmux_ops`**——引脚复用操作集。核心回调是 `set_mux(func_sel, group_sel)`，写入 MODER/AFR 寄存器切换功能。还包含 GPIO 后门回调。（→ 详细分析：§2.7 pinmux_ops）

**`pinconf_ops`**——引脚配置操作集。`pin_config_group_set(group, configs)` 写入 OSPEEDR/PUPDR/OTYPER 配置电气参数。（→ 详细分析：§2.7 pinconf_ops）

**`pinctrl_map`**——DTS 解析结果。字符串形式的映射条目，包含 function 名、group 名、pinconf 参数数组。存储在 `pinctrl_maps` 全局链表中。（→ 详细分析：§2.8 数据管道）

**`pinctrl_setting`**——运行时从 map 转换来的可执行配置。用数字索引代替字符串，直接传给 ops 回调。（→ 详细分析：§2.8 数据管道）

### 2.4 pinctrl_desc：SoC 厂商的注册凭证

先看第一张骨架结构体——它是 pin controller 驱动向 pinctrl core 注册时传入的参数，描述了**这个 SoC 的 pin controller IP 的静态能力**。

pinctrl core 是通用的框架代码，**不知道也不关心**某个具体的 SoC 有多少个引脚、寄存器怎么操作。它通过 `pinctrl_desc` 来获取这些信息——`pinctrl_desc` 是 **SoC 厂商的 pin controller 驱动**（如 `pinctrl-stm32.c`）向 pinctrl core 注册时传入的参数，描述了**这个 SoC 的 pin controller IP 的静态能力**。

```c
/* include/linux/pinctrl/pinctrl.h */
struct pinctrl_desc {
    const char               *name;          /* 控制器名 */
    const struct pinctrl_pin_desc *pins;     /* 引脚描述表 */
    unsigned int              npins;         /* 引脚总数 */
    const struct pinctrl_ops  *pctlops;      /* 功能 1：引脚枚举 */
    const struct pinmux_ops   *pmxops;       /* 功能 2：引脚复用 */
    const struct pinconf_ops  *confops;      /* 功能 3：引脚配置 */
    struct module             *owner;        /* 所属模块（refcount）*/
#ifdef CONFIG_GENERIC_PINCONF
    unsigned int               num_custom_params;
    const struct pinconf_generic_params *custom_params;
    const struct pin_config_item *custom_conf_items;
#endif
    bool link_consumers;                     /* 是否与 consumer 建立设备链接 */
};
```

如果把 pinctrl 子系统比作一套框架，那么 `pinctrl_desc` 就是**挂钩**——SoC 厂商把自家 IP 的参数（引脚数、ops 实现）挂在上面，然后整个框架就能工作了。

下面逐个字段解释：

---

**`name`——控制器名称**

核心层用它来标识这个 pin controller，主要用于日志输出和 debugfs 目录名。

```c
// STM32MP257 上实际填的就是 DTS 节点名
pctldesc->name = "44240000.pinctrl";
```

debugfs 中这个值显示为目录名：

```shell
$ ls /sys/kernel/debug/pinctrl/
44240000.pinctrl/           # 主域 pinctrl
46200000.pinctrl/           # 安全域 pinctrl_z
```

---

**`pins` + `npins`——引脚描述表及其数量**

`pins` 指向一个 `pinctrl_pin_desc` 数组，`npins` 是这个数组的长度。这是 core 层建立 `pin_desc` 数组（冲突检测用）的基础——core 层会遍历这个数组，为每个引脚创建一个 `pin_desc`。

```c
// 伪代码：core 层注册时做的事
for (i = 0; i < npins; i++) {
    struct pin_desc *d = pin_desc + i;
    d->name = pins[i].name;       // 如 "PG14"
    d->number = pins[i].number;   // 如 110
}
```

STM32MP257 主域 176 个引脚，数组部分内容：

```c
pins[0]   = {.number = 0,   .name = "PA0"};
pins[1]   = {.number = 1,   .name = "PA1"};
...
pins[110] = {.number = 110, .name = "PG14"};
...
pins[175] = {.number = 175, .name = "PK7"};
```

**为什么需要这两个字段？** 因为 core 层只有知道"这个 SoC 有多少个引脚、每个引脚叫什么"，才能做三件事：创建冲突检测用的 `pin_desc[]`、在 debugfs 中显示引脚名、在 `pin_request()` 时校验引脚的合法性（请求的 pin 号是否在 0~npins-1 范围内）。

---

**`pctlops` / `pmxops` / `confops`——三大功能的入口**

这是 SoC 厂商实现硬件操作的地方。每个 ops 是一个结构体指针，指向厂商实现的一组回调函数。

| 字段 | 类型 | 管什么 | STM32 的实现 |
|------|------|--------|-------------|
| `pctlops` | `struct pinctrl_ops *` | 引脚枚举——告诉 core 层有多少组、每组有哪些引脚 | `stm32_pctrl_ops` |
| `pmxops` | `struct pinmux_ops *` | 引脚复用——切换外设功能、GPIO 后门 | `stm32_pmx_ops` |
| `confops` | `struct pinconf_ops *` | 引脚配置——上下拉/驱动强度/slew rate | `stm32_pconf_ops` |

**为什么是三个指针而不是一个？** 因为 SoC 可能只实现了部分功能。一个只有 pinmux 没有 pinconf 的控制器，`confops = NULL`，core 层不会调用它。这是内核一贯的"按需实现"设计——不需要的 ops 不用填充。

这三个字段是 `pinctrl_desc` 中**最重要的字段**，§2.5~§2.7 逐个展开。

---

**`owner`——模块引用计数**

防止 pin controller 驱动被 rmmod 卸载时还有设备在使用它。通常填 `THIS_MODULE`。

```c
.owner = THIS_MODULE,
```

如果驱动被编译进内核（非模块），这个字段值为 NULL，相当于不检查。

---

**`link_consumers`——是否与消费者建立 device link**

`bool` 值。为 `true` 时，pinctrl core 会在 pin controller（作为 supplier）和**使用这个引脚的设备**（作为 consumer）之间建立 device link。作用：保证 suspend/resume 时序正确。

pin controller 是 supplier（提供服务者），consumer（如 UART 驱动）是依赖方。device link 强制：

| 阶段 | 顺序 | 原因 |
|------|------|------|
| suspend | consumer **先** → supplier **后** | UART 可能要在 suspend 时调 pinctrl 切 sleep 状态，所以 pinctrl 必须在 UART suspend 之后再断电 |
| resume | supplier **先** → consumer **后** | 恢复时 pinctrl 必须先恢复寄存器访问能力，UART 才能调 pinctrl 恢复引脚配置 |

如果没有这个 device link，suspend 时可能出现 pinctrl 先断电，UART 再去调 `pinctrl_select_state("sleep")`——寄存器访问不了，系统挂死。

```c
.link_consumers = true,   /* STM32 使用的是 true */
```

如果一个设备（比如 UART）用的引脚由这个 pinctrl 管理，没有 device link 的话，suspend 时可能 pinctrl 先断电了，UART 驱动才去调 pinctrl 切 sleep 状态——这时寄存器已经无法访问了。device link 强制了顺序。

如果不需要这种依赖管理（简单控制器），设为 `false` 可以跳过 device link 的开销。

---

**`num_custom_params` / `custom_params` / `custom_conf_items`——SoC 专有参数扩展**

标准 pinconf 参数（上下拉、slew rate 等）由内核定义在 `include/linux/pinctrl/pinconf-generic.h` 中。但 STM32MP257 有 `st,io-retime`、`st,io-clk-edge`、`st,io-delay` 这些**标准 pinconf 没有的参数**。SoC 厂商通过这三个字段向 pinctrl core 注册扩展：

| 字段 | 类型 | 作用 |
|------|------|------|
| `num_custom_params` | `unsigned int` | 数组长度，告诉 core 层有几个自定义参数 |
| `custom_params` | `const struct pinconf_generic_params *` | 自定义参数的**定义**：DTS 属性名 → 参数 ID |
| `custom_conf_items` | `const struct pin_config_item *` | 自定义参数的**显示**：参数 ID → debugfs 可读字符串 |

**`custom_params`**——`struct pinconf_generic_params` 的定义：

```c
struct pinconf_generic_params {
    const char * const property;    /* DTS 属性名（如 "st,io-retime"）*/
    enum pin_config_param param;    /* 参数 ID（自定义枚举值）*/
    u32 default_value;              /* DTS 没写时用的默认值 */
};
```

核心层解析 DTS 时，遇到一个不认识（不在标准参数列表中）的属性名，就去 `custom_params` 数组中逐条匹配 `property` 字段。匹配上了就按自定义参数处理，正常下发到 `confops->pin_config_set()`。没有这个扩展，所有 `st,*` 属性都会被 DTS 解析器忽略并警告。

**`custom_conf_items`**——`struct pin_config_item` 的定义：

```c
struct pin_config_item {
    const enum pin_config_param param;    /* 参数 ID，与 custom_params 一致 */
    const char * const display;           /* debugfs 显示名（如 "io-retime"）*/
    const char * const format;            /* 值的打印格式（"%s" / "%u"）*/
    bool has_arg;                         /* 该参数是否带有值 */
};
```

这个字段的作用是让 debugfs 能显示自定义参数的当前值。内核在 `pinconf_generic_dump_config()`（`drivers/pinctrl/pinconf-generic.c` 第 143 行）中判断：如果没提供 `custom_conf_items`，自定义参数的信息**直接跳过，不在 debugfs 中输出**。

`PCONFDUMP` 是 `struct pin_config_item` 的初始化宏（`include/linux/pinctrl/pinconf-generic.h` 第 177 行）：

```c
#define PCONFDUMP(a, b, c, d) { .param = a, .display = b, .format = c, .has_arg = d }
```

一般用法（StarFive JH7100 驱动为例）：

```c
static const struct pin_config_item starfive_pinconf_custom_conf_items[] = {
    PCONFDUMP(PIN_CONFIG_STARFIVE_STRONG_PULL_UP, "input bias strong pull-up", NULL, false),
};
```

**STM32 驱动没有提供此项（`custom_conf_items = NULL`）**，所以 `st,io-retime` 等自定义参数的当前值不会出现在 debugfs pinconf 输出中——不是因为没生效，只是因为内核不知道该怎么显示它们。

简单说：**`custom_params` 是给内核解析 DTS 用的（必须），`custom_conf_items` 是给人看 debugfs 用的（可选）。**

STM32 上的注册（`drivers/pinctrl/stm32/pinctrl-stm32.c`，第 123 行）：

```c
static const struct pinconf_generic_params stm32_gpio_bindings[] = {
    {"st,io-delay-path",    STM32_GPIO_PIN_CONFIG_DELAY_PATH,  0},
    {"st,io-clk-edge",      STM32_GPIO_PIN_CONFIG_CLK_EDGE,   0},
    {"st,io-clk-type",      STM32_GPIO_PIN_CONFIG_CLK_TYPE,   0},
    {"st,io-retime",        STM32_GPIO_PIN_CONFIG_RETIME,     0},
    {"st,io-delay",         STM32_GPIO_PIN_CONFIG_DELAY,       0},
};
```

核心层解析 DTS 遇到 `st,io-retime` 时，在标准参数中找不到匹配，就在 `custom_params` 数组中查找——找到了就把它当成一个合法的 pinconf 参数，正常走解析路径。如果没有这个扩展机制，所有不认识的自定义属性都会被忽略。

以上是 `pinctrl_desc` 所有字段的逐个说明。将这些字段全部填充后，驱动调用 `devm_pinctrl_register()` 将描述信息提交给 pinctrl core，core 层在注册过程中会做以下几步：

```
struct pinctrl_desc 在驱动中填充完毕
  → devm_pinctrl_register(dev, pinctrl_desc, driver_data)
    → core 层内部创建 pinctrl_dev
    → 根据 pinctrl_desc.pins/npins 创建 pin_desc[] 数组（每个引脚一个）
    → 调 pctlops->get_groups_count() 获取分组数
    → 对每组调 pctlops->get_group_pins() 建立 group_desc[] 数组
    → 返回 pinctrl_dev *——注册后的运行时对象
```

**为什么 pinctrl_desc 和 pinctrl_dev 要分开？** `pinctrl_desc` 是**静态描述**（驱动代码中编译时定义），`pinctrl_dev` 是**运行时对象**（包含 pin_desc 数组、gpio_ranges 链表等动态状态）。一个 `pinctrl_desc` 模板可以注册多次（如 MP257 的两个 pinctrl 实例共用同一个 desc 类型），但每次注册生成的 `pinctrl_dev` 是独立的。


### 2.5 pinctrl_dev：core 层创建的运行时对象

`pinctrl_desc` 是 SoC 厂商填写的"静态描述"。pinctrl core 收到后，在 `pinctrl_register()` 内部**基于它创建一个运行时对象**——这就是 `pinctrl_dev`。两者关系：**厂商提供模具（desc），core 层铸造出实例（dev）**。

`pinctrl_dev` 定义在 `drivers/pinctrl/core.h`（不是公开头文件，驱动开发者不需要直接访问它的字段）：

```c
/* drivers/pinctrl/core.h */
struct pinctrl_dev {
    struct list_head        node;            /* 全局链表中的节点 */
    struct pinctrl_desc    *desc;            /* 指向注册时填写的描述信息 */
    struct radix_tree_root pin_desc_tree;    /* pin_desc 的红黑树索引 */
    struct list_head        gpio_ranges;     /* gpio-ranges 映射链表 */
    struct device          *dev;             /* 对应的 struct device */
    struct module          *owner;
    void                   *driver_data;     /* 厂商私有数据 */
    struct pinctrl         *hog;             /* hog pin 映射 */
    struct list_head        states;          /* 设备的所有引脚状态 */
};
```

下面逐个字段解释。

---

**`desc`——指向注册时的描述信息**

记录这个 pinctrl_dev 是由哪一个 `pinctrl_desc` 创建的。关系是：`pinctrl_dev->desc = 注册时传入的 pinctrl_desc`。

**为什么需要这个字段？** core 层在运行时需要知道"这个控制器的三个 ops 是什么"——`pctldev->desc->pmxops->set_mux()` 就是这样调用的。

---

**`node`——全局链表节点**

所有的 pinctrl_dev 通过这个字段串在 `pinctrl_list` 全局链表中（`drivers/pinctrl/core.c`）：

```c
static LIST_HEAD(pinctrl_list);
```

**为什么需要这个链表？** 当解析 DTS 的 `pinctrl-0 = <&xxx>` 时，DTS 中只写了 phandle（如 `&pinctrl`）。内核需要根据这个 phandle 找到对应的 pinctrl_dev——遍历 `pinctrl_list`，匹配每个 `pinctrl_dev->dev` 的 of_node 与 phandle 指向的节点，找到匹配的那个。

---

**`pin_desc_tree`——最重要的运行时数据**

pinctrl core 在注册时根据 `pinctrl_desc.pins` 数组为每个引脚创建一个 `pin_desc`（定义见 §2.6 冲突检测），以 pin 号为 key 插入红黑树。后续所有 `pin_request()` 调用都通过这棵树查找引脚状态、做冲突检测。

```
pinctrl_dev->pin_desc_tree     radix_tree_lookup(key=110) → pin_desc
                                  ├── mux_owner = "uart1"
                                  ├── mux_usecount = 3
                                  └── gpio_owner = NULL
```

debugfs 中对应的输出：

```
pin 110 (PG14): function usart1 group usart1-0  (3 users)
```

---

**`gpio_ranges`——与 GPIO 子系统的桥梁**

存储 `pinctrl_gpio_range` 节点的链表，每个节点描述一个 GPIO bank 的编号映射关系：

```c
pinctrl_dev->gpio_ranges
    range[0] → {base=0,  npins=16, pin_base=0,  gc=&gpioa_chip}
    range[1] → {base=16, npins=16, pin_base=16, gc=&gpiob_chip}
    range[2] → {base=32, npins=14, pin_base=32, gc=&gpioc_chip}
    ...
```

`pin_desc_tree` 和 `gpio_ranges` 是两个独立结构，解决两个不同方向的问题：

| 字段 | 方向 | 谁查 | 查什么 |
|------|------|------|--------|
| `pin_desc_tree` | Pinctrl 内部 | core 层的 `pin_request()` | pin 号 → pin_desc（冲突检测） |
| `gpio_ranges` | GPIO → Pinctrl | GPIO core 的 `pinctrl_gpio_request()` | 全局编号 → (pctldev, pin_offset) |

---

**`dev`——对应的 struct device**

指向 pin controller 在内核设备模型中的 `struct device`。通过它可以找到对应的 DTS 节点（`dev->of_node`）、power domain、PM 回调等。

```c
/* 日志输出时也用这个字段 */
dev_dbg(pctldev->dev, "pin %d already requested\n", pin);
```

---

**`owner`——模块引用计数**

与 `pinctrl_desc.owner` 相同，防止驱动模块被卸载时还有引用。通常填 `THIS_MODULE`。

---

**`driver_data`——厂商私有数据**

驱动在 `pinctrl_register()` 时传入的第三个参数，core 层原样保存、不碰它。厂商通过 `pinctrl_dev_get_drvdata(pctldev)` 获取。

STM32 上指向的结构体：

```c
struct stm32_pinctrl {
    struct device *dev;
    struct pinctrl_dev *pctl_dev;           /* 指回自己 */
    struct stm32_pinctrl_group *groups;     /* 分组信息，在 build_state 中创建 */
    struct stm32_gpio_bank *banks;          /* 所有 GPIO bank */
    struct stm32_desc_pin *pins;            /* 所有引脚描述 */
    struct clk_bulk_data *clks;             /* 各 bank 的时钟 */
    int nbanks;                              /* bank 数量 */
    ...
};
```

**为什么 core 层要保存这个字段？** 因为 pinctrl_ops 的实现需要访问驱动私有数据——比如 `set_mux` 回调中要操作寄存器，就需要通过 `stm32_pinctrl` 找到 bank 基地址。

---

**`hog`——hog pin 映射**

"hog"意思是"霸占"。某些引脚需要在注册时就强制 pin_request，不允许其他驱动申请。`hog` 存储这些"霸占"引脚的映射状态。STM32 上没有使用（值为 NULL），主要用于调试场景。

---

**`states`——已解析的 pinctrl state 链表**

设备驱动通过 `pinctrl_get()` 请求引脚状态时，core 层解析 DTS 后将结果（多个 `pinctrl_setting` 的集合）存储在 `states` 链表中。之后 `pinctrl_select_state()` 从中查找匹配的 state 并应用。

状态切换时三个字段的协作路径：

```
驱动调 pinctrl_select_state("sleep")
  → 遍历 pctldev->states 链表，找到 name="sleep" 的 state
    → 遍历 state 中的所有 pinctrl_setting[]
      → 如果 type=MUX_GROUP：
          dep->pmxops->set_mux(func_sel, group_sel)
          内部调 pin_request() → 到 pin_desc_tree 做冲突检测
      → 如果 type=CONFIGS_GROUP：
          dep->confops->pin_config_group_set(group, configs)
```

---

**pinctrl_dev 与 pinctrl_desc 的对应关系总图**：

```
pinctrl_dev（运行时，core 层创建）         pinctrl_desc（静态，厂商填写）
    ├── desc ───────────────────────────  .name / .pins / .npins
    ├── pin_desc_tree（红黑树）              .pctlops / .pmxops / .confops
    ├── gpio_ranges（链表）                  .owner / .link_consumers
    ├── dev（设备模型）
    └── driver_data（厂商私有数据）
```


### 2.6 pin_desc + group_desc：引脚与引脚组的管理单元

#### pin_desc：每个引脚一张"身份证"

`pin_desc` 在 `pinctrl_register()` 注册时由 core 层自动创建。core 层遍历 `pinctrl_desc.pins[]` 数组（这是 SoC 厂商填写的写死引脚表），为每个 `pinctrl_pin_desc` 创建一个 `pin_desc`，以 pin 号为 key 插入红黑树：

```c
/* drivers/pinctrl/core.c — pinctrl_register_one_pin() */
pindesc = kzalloc(sizeof(*pindesc), GFP_KERNEL);
pindesc->pctldev = pctldev;
pindesc->name = pin->name;       /* 如 "PG14"，来自 SoC 写死的引脚表 */
mutex_init(&pindesc->mux_lock);
radix_tree_insert(&pctldev->pin_desc_tree, pin->number, pindesc);
/*                          key = pin 号 ↑         */
```

所以 pin_desc 中的数据**不是从 DTS 解析的**，而是来自 SoC 驱动代码中写死的 `stm32_pinctrl_match_data.pins[]` 表。它记录这个引脚当前被谁占着、做什么用——**冲突检测的核心依赖**。

定义在 `drivers/pinctrl/core.h`：

```c
struct pin_desc {
    struct pinctrl_dev *pctldev;     /* 所属 pin controller */
    const char        *name;         /* 引脚名（如 "PG14"）*/
#ifdef CONFIG_PINMUX
    unsigned int       mux_usecount; /* 引用计数 */
    const char        *mux_owner;    /* 复用功能所有者 */
    const struct pinctrl_setting_mux *mux_setting; /* 当前复用配置（debugfs 用）*/
    const char        *gpio_owner;   /* GPIO 所有者 */
    struct mutex       mux_lock;     /* 保护并发访问的锁 */
#endif
};
```

逐字段说明：

**`pctldev`**——指向所属的 pin controller 的运行时对象。每个引脚被哪个 pinctrl 管理，通过这个指针回溯。当 core 层在 `pin_request()` 中需要操作寄存器时（实际由 ops 完成），通过 `pctldev->desc->pmxops->set_mux()` 调用厂商的实现。注意这是一个**反向指针**——pin_desc 是 pinctrl_dev 的 `pin_desc_tree` 红黑树中的节点，但每个节点又指回 pinctrl_dev。

**`name`**——引脚名称（如 "PG14"）。来自 `pinctrl_desc.pins[].name`，注册时由 pinctrl core 复制过来。用于 debugfs 输出，方便开发者定位物理引脚。

**`mux_usecount`**——引用计数。每次成功调用 `pin_request()` 就 +1，每次 `pin_free()` 就 -1。减到 0 时该引脚才算真正释放。为什么需要计数而不是 bool？因为同一个驱动可能对同一组引脚多次调 `pinctrl_get()`（例如设备树中多个 pinctrl-X 引用同一个 group），每次都需要 increment，释放时 decrement，不能直接清零。

**`mux_owner`**——当前占用这个引脚复用的设备名（如 `"uart1"`）。当另一个驱动请求同一引脚时，核心层对比 `mux_owner` 和请求者的名字：名字相同表示同一驱动复用（usecount++），不同则冲突（返回 -EBUSY）。

```
pin_request(pctldev, pin, owner="spi2")
  → pin_desc.mux_owner = "uart1"
  → strcmp("spi2", "uart1") != 0 → mux_usecount > 0 → -EBUSY
```

**`mux_setting`**——指向这个引脚当前的复用配置。指向 `pinctrl_setting` 中的 `pinctrl_setting_mux` 部分，记录了 `func_selector` 和 `group_selector`。主要用于 debugfs 输出——`cat /sys/kernel/debug/pinctrl/*/pinmux-pins` 中显示的 `function usart1` 就是从这个指针读出来的。

**`gpio_owner`**——以 GPIO 方式占用这个引脚的所有者。复用（pinmux）和 GPIO 是两个独立的请求通道——一个引脚可以同时有 `mux_owner`（被配成外设功能）和 `gpio_owner`（被 GPIO 子系统申请）。这是 pinctrl 特有的"双 owner"设计。当 `strict = 0`（STM32）时，两者可以共存；`strict = 1` 时不允许共存。

**`mux_lock`**——保护这个 pin_desc 的并发访问。多核系统上两个 CPU 可能同时调 `pin_request()` 操作同一个引脚，`mux_lock` 保证 `mux_usecount` 的修改是原子的。


上面 pin_desc 中每个字段的值，都可以通过 debugfs 中的 pinmux-pins 文件直接查看。以 PG14 被 usart1 占用为例：

```
pin_desc[110]:                 ↔ debugfs:
    pctldev     → pinctrl_dev   pin 110 (PG14): function usart1 group usart1-0  (3 users)
    name        = "PG14"
    mux_owner   = "uart1"
    mux_usecount = 3
    gpio_owner  = NULL
```

#### group_desc：引脚组的运行时存储

group_desc 描述"一组物理引脚"（如 USART1 的 TX+RX）。它和 `pinctrl_ops` 的三个回调的关系是：**回调是查询接口，group_desc 是背后的存储**。

```
pinctrl_ops                                           group_desc[]
  get_groups_count() → 返回 176                        ┌──────────────┐
  get_group_name(0)  → 返回 "PA0"                    → │ name="PA0"   │
  get_group_pins(0)  → 返回 pin={0}                    │ pins={0}     │
                                                       ├──────────────┤
  get_group_name(1)  → 返回 "PA1"                    → │ name="PA1"   │
  get_group_pins(1)  → 返回 pin={1}                    │ pin={1}      │
                                                       └──────────────┘
```

pinctrl core 在注册时调用这三个回调获取分组信息，存入 `pinctrl_dev->groups[]` 数组。group_desc 的具体内容因 SoC 而异。STM32 采用 **"一个 pin 就是一个 group"** 的策略——每组只包含一个引脚，组名就是引脚名。这些数据来自 SoC 驱动中写死的引脚表（`pinctrl-stm32mp257.c` 的 match_data），不是从 DTS 解析的。

```c
struct group_desc {
    const char *name;        /* 组名（如 "PA0"，STM32 为一 pin 一组）*/
    int *pins;               /* 包含的 pin 号数组 */
    int num_pins;            /* 数组长度 */
    void *data;              /* 驱动私有数据 */
};
```

逐字段说明：

**`name`**——引脚组名称。STM32 采用 **"一个 pin 就是一个 group"**——groups[0].name = "PA0"、groups[1].name = "PA1"、groups[110].name = "PG14"。数据来自 SoC 驱动写死的引脚表，不是 DTS 解析的。

> **注意**：debugfs 中 `function: usart1, groups: [ usart1-0 ]` 显示的 `usart1-0` 是 DTS 中定义的 function+group 映射名，属于 `pinmux_ops->get_function_groups()` 的返回内容，与 `group_desc` 是两套概念。STM32 的 `get_function_groups()` 会返回**所有** group 给每个 function——因为 function↔group 的映射关系完全由 DTS 解析时动态决定，不在驱动中预定义。

**`pins`**——指向 pin 号数组。每个数字对应一个物理引脚（如 `{110, 111}` 表示 PG14 和 PG15）。`set_mux()` 回调中通过这个数组遍历所有引脚，逐个写入 MODER/AFR。

**`num_pins`**——数组的长度。告诉 set_mux 有多少个引脚需要配。

**`data`**——驱动私有数据。pinctrl core 不碰这个字段，保留给厂商驱动使用。STM32 上可以存放额外的电气参数。

ATK 板上的分组实例（STM32 "一个 pin 就是一个 group"，存储在 `pinctrl_dev->groups[]` 中）：

| group_selector | group 名 | 包含的 pin 号 |
|---------------|---------|--------------|
| 0 | `PA0` | 0 |
| 1 | `PA1` | 1 |
| ... | ... | ... |
| 110 | `PG14` | 110 |
| ... | ... | ... |
| 175 | `PK7` | 175 |

---

### 2.7 三个 ops 结构体：操作接口的定义

三个 ops 由 pin controller 驱动实现，是 pinctrl core 操作硬件的唯一入口。

#### pinctrl_ops（引脚枚举）

```c
struct pinctrl_ops {
    int (*get_groups_count)(struct pinctrl_dev *pctldev);
    const char *(*get_group_name)(struct pinctrl_dev *pctldev,
                                   unsigned group_selector);
    int (*get_group_pins)(struct pinctrl_dev *pctldev,
                           unsigned group_selector,
                           const unsigned **pins,
                           unsigned *npins);
};
```

三个回调都由 pinctrl core 在注册时调用，目的是**让 core 层知道这个控制器有哪些引脚组**。它们都用 `group_selector`（数字索引）作为参数——从 0 开始，依次询问每一组。

**`get_groups_count()`**——返回这个控制器有多少个 group。Core 层根据这个返回值分配 `group_desc[]` 数组。STM32 上返回 `pctl->ngroups`，即引脚总数（176 个）。每个 group 只有一个 pin。

**`get_group_name(group_selector)`**——返回第 N 个 group 的名称。Core 层用这个名字在调试输出、日志中标识这个组。STM32 上返回引脚名，如 `"PA0"`、`"PG14"`。

**`get_group_pins(group_selector, &pins, &npins)`**——返回第 N 个 group 包含哪些 pin 号。`pins` 输出一个 pin 号数组，`npins` 输出数组长度。之后 `set_mux()` 回调会用到这些 pin 号来逐个操作硬件寄存器。STM32 上每个 group 只有一个 pin，所以 `*pins = &groups[N].pin`，`*npins = 1`。

这三个回调的调用时机（发生在 `pinctrl_register()` 内部）：

```
pinctrl_register(pinctrl_desc)
  → pinctrl_register_pins()              ← 创建 pin_desc[]
  → count = pctlops->get_groups_count()  ← 有几组？
  → 分配 group_desc[count] 数组
  → for (i = 0; i < count; i++)
      name = pctlops->get_group_name(i)       ← 第 i 组叫什么？
      pins = pctlops->get_group_pins(i, ...)  ← 第 i 组有哪些 pin？
      填充 group_desc[i] = {name, pins, npins}
```

#### pinmux_ops（引脚复用）

```c
struct pinmux_ops {
    int (*set_mux)(struct pinctrl_dev *pctldev,
                   unsigned func_selector, unsigned group_selector);
    int (*gpio_request_enable)(struct pinctrl_dev *pctldev,
                                struct pinctrl_gpio_range *range,
                                unsigned offset);
    int (*gpio_set_direction)(struct pinctrl_dev *pctldev,
                               struct pinctrl_gpio_range *range,
                               unsigned offset, bool input);
    int (*gpio_disable_free)(struct pinctrl_dev *pctldev,
                              struct pinctrl_gpio_range *range,
                              unsigned offset);
    int (*get_functions_count)(struct pinctrl_dev *pctldev);
    const char *(*get_function_name)(struct pinctrl_dev *pctldev,
                                      unsigned selector);
    int (*get_function_groups)(struct pinctrl_dev *pctldev,
                                unsigned selector,
                                const char *const **groups,
                                unsigned *const num_groups);
    bool strict;
};
```

**`set_mux(func_selector, group_selector)`**——**引脚复用的核心入口**。所有 DTS 中 `pinctrl-0 = <&xxx>` 的配置最终都落在这个回调上。core 层在 `pinctrl_select_state()` 中遍历 settings，遇到 `PIN_MAP_TYPE_MUX_GROUP` 类型时调用它。两个参数：

| 参数 | 含义 | 例子 |
|------|------|------|
| `func_selector` | 第几个 function（数字索引） | function "uart1"（= AF6）→ selector = 2 |
| `group_selector` | 第几个 group（数字索引） | group "PG14" → selector = 110 |

set_mux 在 STM32 上的实现路径：

```
set_mux(func_sel, group_sel)
  → 根据 group_sel 找到 pctl->groups[group_sel] 得到 pin 号
  → 根据 func_sel 找到 AF 值
  → stm32_pmx_set_mode(bank, pin, mode=AF, alt=func_num)
    → 写 MODER[pin*2:pin*2+1] = 0b10（Alternate Function）
    → 写 AFRL/H[pin*4:pin*4+3] = alt（AF 编号）
```

**`gpio_request_enable(range, offset)`**——当驱动以 GPIO 方式通过后门机制请求引脚时，由 `pinctrl_gpio_request()` 调用。如果 SoC 实现了此回调，GPIO request 阶段就会直接写 MODER 把引脚切到 GPIO 模式。**STM32 未实现此回调**（选择在 direction 阶段切 MODER）。

**`gpio_set_direction(range, offset, input)`**——驱动调 `gpiod_direction_output/input` 时，由 `pinctrl_gpio_direction_output()` 调用。**STM32 实现了此回调**，在此写入 MODER 把引脚从外设功能切回 GPIO：

```c
stm32_pmx_gpio_set_direction(pctldev, range, offset, input)
  → stm32_pmx_set_mode(bank, pin, mode=!input, alt=0)
    → 写 MODER = 0b01（output）或 0b00（input）
    → 写 AFRL/H = 0（清空外设功能编号）
```

**`gpio_disable_free(range, offset)`**——GPIO 释放时的反向操作。STM32 未实现。

**`get_functions_count()`**——返回这个控制器上有多少种不同的 function。用于 core 层分配 function 索引空间。

**`get_function_name(selector)`**——返回第 N 个 function 的字符串名。主要用于 debugfs 显示。

**`get_function_groups(selector, &groups, &num_groups)`**——返回第 N 个 function 关联了哪些 group。STM32 将**所有 group 都返回给每个 function**——因为 function↔group 的关联完全由 DTS 动态决定，不在驱动中预定义。

**`strict`**——`bool` 值。`true` 时启用严格模式：引脚被 pinmux 占用后 GPIO 请求被拒绝。`false`（STM32）时允许共存——GPIO 读电平不受影响，但写 ODR 不会驱动到引脚上（MODER 在外设模式）。

#### pinconf_ops（引脚配置）

```c
struct pinconf_ops {
    int (*pin_config_set)(struct pinctrl_dev *pctldev,
                          unsigned pin, unsigned long *configs,
                          unsigned num_configs);
    int (*pin_config_group_set)(struct pinctrl_dev *pctldev,
                                unsigned group_selector,
                                unsigned long *configs,
                                unsigned num_configs);
    int (*pin_config_get)(struct pinctrl_dev *pctldev,
                          unsigned pin, unsigned long *config);
};
```

**`pin_config_set(pin, configs, num_configs)`**——配置单个引脚的电气参数。`pin` 是 pin 号，`configs` 是参数数组。STM32 采用"一 pin 一组"，所以此回调通过 `pin_config_group_set` 实现。

**`pin_config_group_set(group_selector, configs, num_configs)`**——配置一组引脚的电气参数。这是 core 层最常调用的 pinconf 回调。`configs` 数组中的每个元素编码为 `(参数类型 << 16) | 参数值`。STM32 的实现是一个大的 switch dispatch：

```c
/* 简化的 STM32 实现 */
switch (pinconf_to_config_param(config)) {
    case PIN_CONFIG_BIAS_DISABLE:
        stm32_pconf_set_bias(bank, offset, 0);       // PUPDR = 0b00
        break;
    case PIN_CONFIG_BIAS_PULL_UP:
        stm32_pconf_set_bias(bank, offset, 1);       // PUPDR = 0b01
        break;
    case PIN_CONFIG_DRIVE_PUSH_PULL:
        stm32_pconf_set_drive(bank, offset, 0);      // OTYPER = 0
        break;
    case PIN_CONFIG_SLEW_RATE:
        stm32_pconf_set_speed(bank, offset, arg);    // OSPEEDR = arg
        break;
    /* 以及 STM32 自定义的 st,io-retime 等 */
}
```

**`pin_config_get(pin, &config)`**——读取单个引脚的当前配置。主要用于 debugfs——`cat /sys/kernel/debug/pinctrl/*/pinconf-pins` 时遍历每个引脚调用此回调。

`configs` 的编码与解码：

```c
/* DTS 中 slew-rate = <2> 被解析为 */
unsigned long config = (PIN_CONFIG_SLEW_RATE << 16) | 2;

/* 在 pinconf 回调内部分解 */
enum pin_config_param param = pinconf_to_config_param(config);  // PIN_CONFIG_SLEW_RATE
u32 arg = pinconf_to_config_argument(config);                   // 2
```

编码成一个 unsigned long 而不是结构体，是因为 pinconf 参数经常批量传递（数组方式），单个 unsigned long 可以用一次赋值完成拷贝，比结构体更高效。

---

### 2.8 数据管道：pinctrl_map → pinctrl_setting

DTS 中的引脚配置是静态文本，运行时需要转化为内核可执行的配置。转化路径涉及两个数据结构：**`pinctrl_map`**（DTS 解析结果，字符串形式）和 **`pinctrl_setting`**（运行时配置，数字索引形式）。

```
DTS                                    内核运行时

pinctrl-0 = <&usart1_pins_a>           pinctrl_maps 全局链表
  ├─ pinmux = <STM32_PINMUX>           ├─ pinctrl_map[0]: type=MUX_GROUP
  │                                      .function = "uart1"
  │                                      .group = "PG14"
  ├─ bias-disable                      ├─ pinctrl_map[1]: type=CONFIGS_GROUP
  │                                      .group = "PG14"
  │                                      .configs = {BIAS_DISABLE}
  └─ slew-rate = <0>                   └─ pinctrl_map[2]: type=CONFIGS_GROUP
                                         .group = "PG14"
                                         .configs = {SLEW_RATE=2}
                                                 ↓ pinctrl_select_state()
                                         pinctrl_setting[]
                                         ├─ setting[0]: type=MUX_GROUP
                                         │   .data.mux.func_sel = 2    ← 数字索引
                                         │   .data.mux.group_sel = 110
                                         ├─ setting[1]: type=CONFIGS_GROUP
                                         │   .data.configs.group_sel = 110
                                         │   .data.configs.configs = {BIAS_DISABLE}
                                         └─ setting[2]: type=CONFIGS_GROUP
                                             .data.configs.group_sel = 110
                                             .data.configs.configs = {SLEW_RATE=2}
```

**为什么一个 DTS 节点会生成多个 map 条目？** 因为 `pinmux` 和 `bias-disable`、`slew-rate` 是**两种不同类型的映射**——`PIN_MAP_TYPE_MUX_GROUP` 和 `PIN_MAP_TYPE_CONFIGS_GROUP`。一个 `pinctrl_map` 的 union 中要么存 `{function, group}` 要么存 `{configs}`，不能同时包含两者。

#### pinctrl_map：DTS 解析结果的静态存储

`pinctrl_map` 由 `pinctrl_dt_to_map()`（`drivers/pinctrl/devicetree.c`）在设备 probe 时解析 DTS 生成，存储在 `pinctrl_maps` 全局链表中。

```c
struct pinctrl_map {
    const char *dev_name;       /* 使用该映射的设备 */
    const char *name;           /* state 名 */
    enum pinctrl_map_type type; /* 映射类型 */
    const char *ctrl_dev_name;  /* pin controller 设备名 */
    /* type = PIN_MAP_TYPE_MUX_GROUP 时 */
    const char *function;       /* function 名（如 "uart1"）*/
    const char *group;          /* group 名（如 "PG14"）*/
    /* type = PIN_MAP_TYPE_CONFIGS_GROUP 时 */
    unsigned long *configs;     /* 配置编码数组 */
    unsigned num_configs;       /* 数组长度 */
};
```

逐字段说明：

**`dev_name`**——使用这个映射的消费者设备名。对应 DTS 中引用 `pinctrl-0` 的节点名（如 `"usart1"`）。pinctrl core 在 `pinctrl_select_state(dev, ...)` 时通过这个字段匹配消费者——只应用与传入 dev 同名的映射。如果为 NULL，表示这是 pin controller **自身**专用的 hog 映射，在 `pinctrl_register()` 时自动应用，不依赖任何消费者设备。

**`name`**——状态名，对应 `pinctrl-names` 中的字符串（如 `"default"`、`"sleep"`）。当 `pinctrl_select_state(dev, "sleep")` 被调用时，core 层筛选出 `name = "sleep"` 的条目。

**`type`**——映射类型，决定这个 map 是复用配置还是电气配置：

| 枚举值 | 含义 | 对应的 data 字段 |
|--------|------|-----------------|
| `PIN_MAP_TYPE_MUX_GROUP` | 引脚复用配置 | `.function` + `.group` |
| `PIN_MAP_TYPE_CONFIGS_GROUP` | 引脚电气配置 | `.configs[]` |

**`ctrl_dev_name`**——pin controller 设备名。DTS 中通过 phandle（`&pinctrl`）指向 pin controller 节点，`pinctrl_dt_to_map()` 在解析时将 phandle 转换为对应的 pin controller 设备名。后续通过 `get_pinctrl_dev_from_devname()` 根据这个字符串查找 `pinctrl_dev`。

以下字段仅当 `type = PIN_MAP_TYPE_MUX_GROUP` 时有效：

**`function`**——function 名，字符串形式（如 `"uart1"`）。来自 DTS 中 `STM32_PINMUX()` 解析后的 AF 编号对应的逻辑名。运行时会通过 `pinmux_ops.get_function_name()` 查找对应的数字索引（`func_selector`）。

**`group`**——group 名，字符串形式（如 `"PG14"`）。来自 DTS 中引脚配置节点指定的引脚。运行时会通过 `pinctrl_ops.get_group_name()` 查找对应的数字索引（`group_selector`）。

以下字段仅当 `type = PIN_MAP_TYPE_CONFIGS_GROUP` 时有效：

**`configs`**——指向电气参数编码数组。每个元素是 `(pinconf参数类型 << 16) | 参数值`。如 `slew-rate = <2>` 编码为 `(PIN_CONFIG_SLEW_RATE << 16) | 2`。

**`num_configs`**——`configs` 数组的元素个数。有多少个电气属性就有多少个元素。

存储位置——全局链表 `pinctrl_maps`（`drivers/pinctrl/core.c`）。每个设备 probe 时 DTS 解析生成自己的 map 数组，用 `struct pinctrl_maps` 包装后挂入全局链表：

```c
struct pinctrl_maps {
    struct list_head node;
    const struct pinctrl_map *maps;  /* 指向 map 数组 */
    unsigned num_maps;               /* 数组长度 */
};
static LIST_HEAD(pinctrl_maps);      /* 全局链表头 */
```

**为什么用 `struct pinctrl_maps` 包装一层？** 因为 `pinctrl_map` 本身就是个数组，不能直接把数组挂到链表上。`struct pinctrl_maps` 作为一个包装节点，存了数组指针和长度，串入全局链表。链表节点数 = 设备数，不是 map 条目数，查找时逐节点扫描。

**ATK 板实例**——全局链表中的内容（DTS 解析后的结果）：

```
全局链表 pinctrl_maps（链表头）
  │
  ├── node（来自 &usart1 的 DTS 解析）
  │     .num_maps = 4
  │     .maps[0]: dev="usart1", name="default",  type=MUX,    func="uart1", group="PG14"
  │     .maps[1]: dev="usart1", name="default",  type=CONFIGS, group="PG14", configs={BIAS_DISABLE}
  │     .maps[2]: dev="usart1", name="sleep",    type=MUX,    func="ANALOG", group="PG14"
  │     .maps[3]: dev="usart1", name="sleep",    type=CONFIGS, group="PG14", configs={ANALOG}
  │
  ├── node（来自 &i2c3 的 DTS 解析）
  │     .num_maps = 2
  │     .maps[0]: dev="i2c3", name="default", type=MUX,    func="i2c3", group="PG1"
  │     .maps[1]: dev="i2c3", name="default", type=CONFIGS, group="PG1", configs={BIAS_DISABLE}
  │
  └── node（来自板级 platform_data，非 DT）
        .num_maps = 1
        .maps[0]: dev="pcie", name="default", type=MUX, func="pcie", group="PB8"
```

当 `pinctrl_select_state(usart1_dev, "default")` 调用时，遍历 `pinctrl_maps` 链表，匹配 `dev="usart1" && name="default"`，取出 map[0] 和 map[1]，生成 setting，调用 `set_mux()` 和 `pin_config_group_set()`。

#### pinctrl_setting：运行时可执行的配置

`pinctrl_setting` 由 `pinctrl_select_state()` 内部从 `pinctrl_map` 转换而来。转换过程：**将字符串形式的 function/group 名称解析为数字索引**，然后直接传递给 ops 回调。

```c
struct pinctrl_setting {
    struct list_head node;           /* 链表节点 */
    enum pinctrl_map_type type;      /* 映射类型 */
    struct pinctrl_dev *pctldev;     /* 目标 pin controller */
    const char *dev_name;            /* 消费者设备名 */
    union {
        struct pinctrl_setting_mux {
            unsigned func_sel;       /* function 的数字索引 */
            unsigned group_sel;      /* group 的数字索引 */
        } mux;
        struct pinctrl_setting_configs {
            unsigned group_sel;      /* group 的数字索引 */
            unsigned long *configs;  /* 电气参数编码数组 */
            unsigned num_configs;    /* 数组长度 */
        } configs;
    } data;
};
```

逐字段说明：

**`node`**——链表节点。多个 `pinctrl_setting` 组成一个 state，串在 `struct pinctrl_state.settings` 链表中。

**`type`**——与 `pinctrl_map.type` 相同，决定 union 中使用哪个成员。

**`pctldev`**——指向目标 pin controller 的运行时对象。直接从 `pinctrl_map.ctrl_dev_name` 转换而来——`get_pinctrl_dev_from_devname(ctrl_dev_name)` 查找得到。

**`dev_name`**——消费者设备名（与 `pinctrl_map.dev_name` 相同）。保留用于调试输出。

以下字段仅当 `type = PIN_MAP_TYPE_MUX_GROUP` 时有效：

**`data.mux.func_sel`**——function 的数字索引。从 `pinctrl_map.function` 字符串转换而来——`pinmux_ops.get_function_name(i)` 返回的字符串与 `pinctrl_map.function` 匹配时，i 就是 `func_sel`。之后直接传给 `set_mux(func_sel, group_sel)`。

**`data.mux.group_sel`**——group 的数字索引。转换方式相同——`pinctrl_ops.get_group_name(i)` 匹配后得到 i。

以下字段仅当 `type = PIN_MAP_TYPE_CONFIGS_GROUP` 时有效：

**`data.configs.group_sel`**——group 的数字索引。转换方式与 mux 的 `group_sel` 相同。

**`data.configs.configs`**——直接继承自 `pinctrl_map.configs`。不需要转换，因为电气参数本身就是数字编码。

**`data.configs.num_configs`**——配置数组长度。

**为什么 mux 部分要转成数字索引？** `set_mux()` 只接受 `func_selector` 和 `group_selector`（数字索引），不接受字符串。map 中存的是字符串（`"uart1"`、`"PG14"`），setting 在生成时一次性完成字符串→数字索引的查找——`pinmux_ops.get_function_name(i)` 匹配 `"uart1"` 找到 `i=2`，`pinctrl_ops.get_group_name(i)` 匹配 `"PG14"` 找到 `i=110`。之后 `set_mux(2, 110)` 直接用数字，不再查字符串。

**为什么 configs 部分不需要转换？** 电气参数本身就是编码后的数字值（`(PIN_CONFIG_SLEW_RATE << 16) | 2`），map 和 setting 中都是同一份数字，直接从 map 复制到 setting，不需要任何查找。

```
map（字符串） →  pinctrl_select_state()  →  setting（数字） →  ops 回调
 "uart1"               查找 func_sel              func_sel=2      set_mux(2, 110)
 "PG14"                查找 group_sel             group_sel=110
```

---

### 2.9 功能实现一：引脚枚举

数据结构准备好之后，再看每个功能是怎么通过它们实现的。

引脚枚举回答的是"这个控制器管哪些引脚、怎么分组的"。pinctrl core 在注册时调用三个回调来建立自己的分组信息。以上面介绍的 `pinctrl_ops`、`group_desc` 和 `pctldev` 为背景，实际的枚举过程就是填充 `pinctrl_dev->groups[]` 数组的过程：

```
pinctrl_register(pinctrl_desc)
  → 1. pinctrl_register_pins()
       根据 desc->pins 数组创建 pin_desc[0..175]，插入红黑树
  → 2. count = pctlops->get_groups_count()    ← 返回 176
  → 3. 分配 pctldev->groups = group_desc[176]
  → 4. for (i = 0; i < 176; i++)
        name  = pctlops->get_group_name(i)
        pins  = pctlops->get_group_pins(i, ...)
        group_desc[i] = {name, pins, npins}
```

枚举完成后，`pinctrl_dev` 内部的 `groups[]` 数组如下（ATK 板实际数据）：

```
pinctrl_dev->groups[] 数组（共 176 个元素）
  │
  ├── groups[0]   → { .name = "PA0",   .pins = {0},   .num_pins = 1 }
  ├── groups[1]   → { .name = "PA1",   .pins = {1},   .num_pins = 1 }
  ├── groups[2]   → { .name = "PA2",   .pins = {2},   .num_pins = 1 }
  ├── ...         
  ├── groups[110] → { .name = "PG14",  .pins = {110}, .num_pins = 1 }
  ├── groups[111] → { .name = "PG15",  .pins = {111}, .num_pins = 1 }
  ├── ...
  └── groups[175] → { .name = "PK7",   .pins = {175}, .num_pins = 1 }
```

每个 `group` 只包含一个 pin，因为 STM32 采用"一 pin 一组"的策略。`groups[i]` 的下标就是 `group_selector`，`set_mux(func_sel, group_sel)` 中的 `group_sel=110` 就对应 `groups[110]`，即 `pinctrl pin 110`（PG14）。

> 注意：pinctrl pin 号和 GPIO 全局编号是两套概念。这里 `pins = {110}` 是 pinctrl 内部的 pin 号。在 STM32 上，因 `gpio-ranges` 为一对一连续映射（GPIOA base=0 → pinctrl pin_base=0），pinctrl pin 号恰好等于 GPIO 全局编号。但不是所有 SoC 都如此。

枚举过程中三个回调与填入 `group_desc` 的对应关系（以第 0 组和第 110 组为例）：

```
i=0: get_group_name(0) → "PA0"           → groups[0].name = "PA0"
     get_group_pins(0) → pins={0}, npins=1 → groups[0].pins = {0}
                                             groups[0].num_pins = 1

i=110: get_group_name(110) → "PG14"        → groups[110].name = "PG14"
       get_group_pins(110) → pins={110}, npins=1 → groups[110].pins = {110}
                                                   groups[110].num_pins = 1
```

这些数据全部来自 `pinctrl_desc.pins[]` 中写死的 SoC 引脚表，**不是从 DTS 解析的**。

> 注意：DTS 中看到的 `usart1-0` 等节点名不是 `group_desc`，它们是 `pinctrl_map` 中的 group 字段，由 `pinctrl_dt_to_map()` 解析时生成。`group_desc` 是 STM32 写死的"一 pin 一组"——`groups[0]="PA0"`、`groups[110]="PG14"`。

### 2.10 功能实现二：引脚复用

引脚复用回答的是"把 PG14 从当前功能切到 AF6（USART1_TX）"。从 DTS 到寄存器写入经过 4 个阶段。

#### 阶段 1：DTS 解析——`pinctrl_dt_to_map()`

在设备 probe 时，`pinctrl_bind_pins()` 调 `pinctrl_dt_to_map()`，解析 `pinctrl-0 = <&usart1_pins_a>` 指向的 DTS 节点：

```dts
usart1_pins_a: usart1-0 {
    pins1 {
        pinmux = <STM32_PINMUX('G', 14, AF6)>;  /* TX */
        bias-disable;
        drive-push-pull;
    };
};
```

DTS 解析代码遍历 `pinmux` 属性中的每个 `STM32_PINMUX` 值，分解为 pin 号和 function 号：

```
STM32_PINMUX('G', 14, AF6)
  → STM32_GET_PIN_NO   → pin  = 110   （即 PG14）
  → STM32_GET_PIN_FUNC → func = 7     （即 AF6）
  → find_group_by_pin(110)       → groups[110].name = "PG14"
  → stm32_gpio_functions[7]      → function 名 = "af6"
```

每个 pin 独立生成 MUX 和 CONFIGS 两组 map（因为 STM32 一 pin 一组）：

```
usart1 的 pinctrl_maps 节点：
  map[0]: type=MUX_GROUP,     function="af6", group="PG14"
  map[1]: type=CONFIGS_GROUP, group="PG14", configs={BIAS_DISABLE, PUSH_PULL}
  map[2]: type=MUX_GROUP,     function="af6", group="PG15"
  map[3]: type=CONFIGS_GROUP, group="PG15", configs={BIAS_DISABLE}
```

#### 阶段 2：pinctrl_select_state()——字符串→数字转换

`pinctrl_bind_pins()` 调 `pinctrl_select_state(dev, "default")`。对每个 MUX 类型的 map，将字符串转为数字索引：

```
map: function="af6", group="PG14"

查 func_sel:  "af6" 在 stm32_gpio_functions[] 中的下标  → func_sel = 7
查 group_sel: "PG14" 在 groups[] 中的下标               → group_sel = 110

生成的 setting: func_sel=7, group_sel=110  ← 数字，直接传给 set_mux
```

CONFIGS 类型不用转换（configs 本身是数字编码），直接复制到 setting。

#### 阶段 3：调用 `pmxops->set_mux(func_sel, group_sel)`

```
stm32_pmx_set_mux(pctldev, function=7, group=110)   // 源码，drivers/pinctrl/stm32/pinctrl-stm32.c
{
    g = pctl->groups[110];           // 根据 group_sel 找到 group
    g->pin = 110;                     // pinctrl pin 号

    pinctrl_find_gpio_range_from_pin(pctldev, 110);  // 查 gpio-ranges → GPIOG bank
    bank = gpiochip_get_data(range->gc);             // GPIOG 的寄存器基地址
    pin  = stm32_gpio_pin(110);      // 转换 pinctrl pin → bank 内偏移: 110-96=14

    mode = stm32_gpio_get_mode(7);   // function=7(AF6) → mode=2(MODER=0b10, AF 模式)
    alt  = stm32_gpio_get_alt(7);    // function=7(AF6) → alt=6(AFR 寄存器写入 6)

    stm32_pmx_set_mode(bank, pin=14, mode=2, alt=6);  // 写寄存器
}
```

#### 阶段 4：写寄存器——`stm32_pmx_set_mode()`

写入两个寄存器：
1. **AFRH**——写入外设功能编号 AF6
2. **MODER**——写入 Alternate Function 模式（0b10）

写入后 PG14 由 UART 控制器控制，GPIO 写 ODR 不影响引脚电平。

---

**function + group 对应关系**：一个 function 对应多个 group（sdmmc2 有 b4 和 d47 两组），一个 group 只能属于一个 function。

**冲突检测**：`set_mux` 前调 `pin_request()`，通过 `pin_desc.mux_usecount` 和 `mux_owner` 对比（逻辑见 §2.6）。

### 2.11 功能实现三：引脚配置

引脚配置的流程与 §2.10（引脚复用）完全一致——同样经过 DTS 解析 → map → setting → ops 回调 → 写寄存器。**仅有的区别**是：

1. **无需字符串→数字转换**：电气参数在 DTS 解析时已被 `pinconf_generic_parse_dt_config()` 编码为 `(参数类型 << 16) | 参数值` 的数字格式，map 和 setting 中都是同一份编码，直接复制
2. **调用的 ops 不同**：`confops->pin_config_group_set(group_sel, configs)` 而非 `set_mux`
3. **操作的寄存器不同**：写 OSPEEDR/PUPDR/OTYPER，而非 MODER/AFR

#### DTS 属性 → pinconf 参数 → 硬件寄存器映射

| DTS 属性 | pinconf 参数 | STM32 寄存器操作 |
|---------|-------------|-----------------|
| `slew-rate = <N>` | `PIN_CONFIG_SLEW_RATE` | OSPEEDR = N |
| `bias-disable` | `PIN_CONFIG_BIAS_DISABLE` | PUPDR = 0b00 |
| `bias-pull-up` | `PIN_CONFIG_BIAS_PULL_UP` | PUPDR = 0b01 |
| `bias-pull-down` | `PIN_CONFIG_BIAS_PULL_DOWN` | PUPDR = 0b10 |
| `drive-push-pull` | `PIN_CONFIG_DRIVE_PUSH_PULL` | OTYPER = 0 |
| `drive-open-drain` | `PIN_CONFIG_DRIVE_OPEN_DRAIN` | OTYPER = 1 |

> STM32MP257 的 `st,io-retime` 等专有属性通过 `pinctrl_desc.custom_params` 扩展（§2.4），走相同解析路径，最终由 `stm32_pconf_set_advcfgr()` 写入 ADVCFGR 寄存器。

#### 执行顺序

同一 `pinctrl-0` 节点中的 `pinmux` 和电气属性在解析时拆为独立的 map，`pinctrl_select_state()` 执行时**先配复用功能，再配电气参数**：

```
pinctrl_select_state()
  ├─ set_mux(func_sel, group_sel)         → 写 MODER + AFR（先切功能）
  └─ pin_config_group_set(group, configs)  → 写 OSPEEDR + PUPDR + OTYPER（再配电气）
```


## 3. GPIO 侧：三层架构与数据结构

### 3.1 GPIO 三大功能 + 分层架构

GPIO 子系统管理引脚的**数据读写**（Pinctrl 管功能选择，GPIO 管数据）。驱动开发者通过三类操作与 GPIO 交互：

| # | 功能 | API | 作用 |
|---|------|-----|------|
| 1 | **输出** | `gpiod_set_value()` / `gpiod_direction_output()` | 控制引脚电平（LED 亮灭、复位信号） |
| 2 | **输入** | `gpiod_get_value()` / `gpiod_direction_input()` | 读取引脚电平（按键状态、检测信号） |
| 3 | **中断** | `gpiod_to_irq()` + `request_irq()` | 等待引脚电平变化（按键中断、边沿触发） |

和 Pinctrl 一样，GPIO 子系统也采用三层分离架构：

```
┌──────────────────────────────────────────────────────────────────────┐
│  消费者层 (GPIO Consumer)                                gpiolib API │
│                                                                      │
│  ┌──────────────────────────────────────────────┐                   │
│  │  设备驱动（LED/按键/MMC/传感器驱动等）          │                   │
│  │    ● devm_gpiod_get() / gpiod_set_value()     │                   │
│  │    ● gpiod_direction_output() / to_irq()     │                   │
│  │    ● 不直接访问硬件寄存器，只看 gpio_desc *     │                   │
│  └──────────────────────────────────────────────┘                   │
│          │ 通过 gpio_chip 中的 ops 间接调用                         │
│          ▼                                                          │
├──────────────────────────────────────────────────────────────────────┤
│  GPIO CORE 层 (通用逻辑)                           drivers/gpio/    │
│                                                                      │
│  ┌──────────┐ ┌───────────┐ ┌──────────┐ ┌────────────┐            │
│  │ gpiolib  │ │ gpiolib-  │ │ gpiolib- │ │ gpiolib-   │            │
│  │ .c       │ │ of.c      │ │ sysfs.c  │ │ cdev.c     │            │
│  │          │ │           │ │          │ │            │            │
│  │ 描述符   │ │ DT phandle│ │ sysfs    │ │ chardev    │            │
│  │ 管理     │ │ 解析      │ │ 接口     │ │ /dev/      │            │
│  │ 冲突检测 │ │ gpio-rngs │ │ (已废弃) │ │ gpiochipN  │            │
│  └──────────┘ └───────────┘ └──────────┘ └────────────┘            │
│          │                                                          │
│          │  调用 gpio_chip->request/set/direction...                │
│          ▼                                                          │
├──────────────────────────────────────────────────────────────────────┤
│  GPIO CHIP 驱动层 (硬件操作)                      drivers/gpio/     │
│                                                                      │
│  ┌──────────────────────────────────────────────┐                   │
│  │  stm32_gpio.c / gpio-mxc.c / gpio-pca953x.c  │                   │
│  │    ● 实现 gpio_chip 中的回调                    │                   │
│  │    ● 操作硬件寄存器或 I2C/SPI 命令               │                   │
│  │    ● STM32：写 BSRR/ODR/IDR/MODER 寄存器      │                   │
│  └──────────────────────────────────────────────┘                   │
│          │                                                          │
│          ▼  写寄存器/发 I2C 命令                                     │
│  ┌──────────────────────────────────────────────┐                   │
│  │  STM32MP257 GPIO 硬件 / PCA9535 扩展器       │                   │
│  │  BSRR · ODR · IDR · MODER                   │                   │
│  └──────────────────────────────────────────────┘                   │
└──────────────────────────────────────────────────────────────────────┘
```

各层定位：**消费者层**看到的是 `struct gpio_desc *`（不透明指针），通过 `gpiod_*` API 操作；**GPIO Core 层**管理描述符分配、冲突检测、DTS 解析、chardev；**gpio_chip 驱动层**实现硬件操作回调。三层之间通过 `gpio_chip` 的 ops 间接调用。

### 3.2 数据结构关系图：结构体之间的指针指向

GPIO 子系统有三个核心结构体，它们之间通过指针连接：

```
                    gpiochip_add_data()
                    ┌──────────────────────────────────────────────┐
                    │              gpio_device                     │
                    │  .dev     → struct device{of_node, ...}      │
                    │  .chrdev  → /dev/gpiochip0                   │
                    │  .id      = 0                                │
                    │  .base    = 0   （全局编号起点）               │
                    │  .ngpio   = 16  （本 bank 引脚数）             │
                    │  .label   = "GPIOA"                          │
                    │  .chip    ──────────────────────┐             │
                    │  .descs   ──────┐                │             │
                    │  .pin_ranges    │                │             │
                    └─────────────────│────────────────│─────────────┘
                                      │                │
                    ┌─────────────────▼────┐  ┌────────▼──────────┐
                    │    gpio_desc[0..15]  │  │    gpio_chip       │
                    │                     │  │                   │
                    │  descs[0]:           │  │  .label="GPIOA"   │
                    │    .gdev → gpio_dev  │  │  .base=0          │
                    │    .flags=0          │  │  .ngpio=16        │
                    │    .label="pa5-led"  │  │  .request → stm32 │
                    │  descs[1]:           │  │  .set     → stm32 │
                    │    ...               │  │  .get     → stm32 │
                    │  descs[4]:           │  │  .to_irq  → stm32 │
                    │    .flags=REQUESTED  │  │  .direction_input │
                    │    .label="heartbeat"│  │  .direction_out   │
                    └─────────────────────┘  └───────────────────┘

```

各结构体速览：

| 结构体 | 作用 | 谁实现的 |
|--------|------|---------|
| `gpio_device` | GPIO 控制器的完整设备抽象 | core 层创建 |
| `gpio_chip` | 硬件操作接口（回调函数集） | 厂商驱动实现 |
| `gpio_desc` | 单个引脚的描述符（不透明指针） | core 层创建 |

### 3.3 gpio_device：控制器的完整设备抽象

`gpio_device` 是 GPIO 子系统对一个"GPIO 控制器"的完整描述。它在 `gpiochip_add_data()` 时由 core 层创建。每个 bank（GPIO 端口）一个——这里的"bank"指的是一组共用相同寄存器组的引脚，在 STM32 上每个 bank 映射到一段独立的寄存器地址空间（如 GPIOA 的 MODER/ODR/IDR 等寄存器都在 0x44240000 开始的区间），一般包含 16 个引脚。ATK 板上共 10 个 bank：主域 GPIOA~GPIOI（各 16/14/12 个引脚不等）+ 安全域 GPIOZ（10 个引脚）。

定义在 `drivers/gpio/gpiolib.h`：

```c
struct gpio_device {
    struct device     dev;          /* 内核设备模型 */
    struct cdev       chrdev;       /* 字符设备（/dev/gpiochipN）*/
    int               id;           /* chip 编号（0=gpiochip0）*/
    struct device     *mockdev;     /* 用于 sysfs 的设备 */
    struct module     *owner;       /* 模块引用计数 */
    struct gpio_chip  *chip;        /* → 硬件操作接口 */
    struct gpio_desc  *descs;       /* → 描述符数组 */
    int               base;         /* 全局编号起点 */
    u16               ngpio;        /* 引脚数 */
    const char        *label;       /* 名称（如 "GPIOA"）*/
    void              *data;        /* 厂商私有数据 */
    struct list_head  list;         /* 全局链表节点 */
#ifdef CONFIG_PINCTRL
    struct list_head  pin_ranges;   /* gpio-ranges 映射链表 */
#endif
};
```

逐字段说明：

`gpio_device` 中有三个与"设备"相关的字段，分别服务于三个不同的接口路径：

| 字段 | 设备路径 | 谁用 | 用途 |
|------|---------|------|------|
| `dev` | 设备模型树（如 `/sys/devices/platform/soc/...`） | 内核驱动框架 | 电源管理、DTS of_node 关联、probe/remove 生命周期 |
| `chrdev` | `/dev/gpiochipN` | 用户态程序 | 通过 ioctl 操作 GPIO（libgpiod 方式） |
| `mockdev` | `/sys/class/gpio/gpiochipN/` | 用户态 shell 脚本 | legacy sysfs 接口（`echo N > export`，已 deprecated） |

**`dev`**——内核设备模型的 `struct device`。指向这个 GPIO 控制器的设备节点。`dev->of_node` 指向 DTS 节点（如 `gpio@44240000`），用于电源管理、of 解析等。

**`chrdev`**——字符设备。每个 `gpio_device` 对应一个 `/dev/gpiochipN`，用户态通过它发起 ioctl 操作（libgpiod 的工具链就是通过这个接口与内核通信）。由 gpiolib core 在注册时自动创建。

**`id`**——控制器编号。`gpiochip0` 的 id = 0，`gpiochip7`（GPIOH）的 id = 7。对应 `/dev/gpiochip7`。

**`mockdev`**——为 legacy sysfs 接口（`/sys/class/gpio/gpiochipN/`）创建的 class device，让用户态通过 `echo N > export` 等方式导出 GPIO 操作。`CONFIG_GPIO_SYSFS` 禁用时此字段为 NULL。

**`owner`**——模块引用计数。防止 GPIO 控制器驱动被 rmmod 卸载时还有消费者在使用。通过 `try_module_get()` / `module_put()` 维护。

**`chip`**——指向 `gpio_chip` 的指针。**关键字段**。`gpio_device` 是"控制器管理单位"（设备模型、chrdev、pin_ranges），而 `gpio_chip` 是"硬件操作单位"（set/get/direction）。同一个 `gpio_chip` 可能被多个 `gpio_device` 共享（但通常一对一）。所有 `gpiod_*` API 最终都通过 `gdev->chip->xxx()` 调用到硬件。

**`descs`**——指向 `gpio_desc` 数组。数组长度为 `ngpio`。每个引脚对应一个 `gpio_desc`。驱动调 `devm_gpiod_get()` 时返回的就是 `descs[offset]` 的指针。

**`base`**——全局编号起点。**deprecated**，但仍在广泛使用。PA0 = base + 0 = 0，PB0 = base + 16 = 16，PG14 = 96 + 14 = 110。新驱动应传 -1 让系统自动分配。

**`ngpio`**——本控制器的引脚数。GPIOA = 16，GPIOC = 14，GPIOH = 14，GPIOZ = 10。不同的封装（package）数值不同，通过 DTS 的 `st,package` 属性确定。

**`label`**——控制器名称。STM32 上填 `st,bank-name` 属性值，如 `"GPIOA"`、`"GPIOB"`。用于 debugfs 和日志输出。

**`data`**——厂商私有数据指针。与 `gpio_chip` 的私有数据约定相同。STM32 上指向 `struct stm32_gpio_bank`，包含寄存器基地址等信息。

**`list`**——全局链表节点。所有 `gpio_device` 通过这个字段串在 `gpio_devices` 全局链表中，供 `gpio_device_find()` 遍历查找。

**`pin_ranges`**——gpio-ranges 映射链表。与 pinctrl 的桥梁（§4 详述），存储 `pinctrl_gpio_range` 节点。注册时通过 `of_gpiochip_add()` 解析 DTS 的 `gpio-ranges` 属性后填充。

### 3.4 gpio_chip：厂商的硬件操作接口

`gpio_chip` 是 SoC 厂商实现的硬件操作接口。它的回调函数被 GPIO core 层在 `gpiod_*` API 中间接调用。

**为什么需要 `gpio_chip`？** 不同的 GPIO 控制器硬件差异巨大：

- STM32 内置 GPIO：直接读写寄存器（几十 ns）
- I2C GPIO 扩展器（PCA9535）：通过 I2C 命令读写（几百 µs）
- SPI GPIO 扩展器：通过 SPI 命令读写（几十 µs）

`gpio_chip` 把这些差异封装成统一的回调接口，core 层只调回调，不关心底层。

```c
struct gpio_chip {
    const char        *label;       /* bank 名（如 "GPIOA"）*/
    struct gpio_device *gpiodev;    /* → 指回 gpio_device */
    struct device     *parent;      /* 父设备 */
    struct fwnode_handle *fwnode;   /* fwnode 句柄 */
    struct module     *owner;       /* 模块引用计数 */

    /* 核心操作回调 */
    int (*request)(struct gpio_chip *gc, unsigned int offset);
    void (*free)(struct gpio_chip *gc, unsigned int offset);
    int (*get_direction)(struct gpio_chip *gc, unsigned int offset);
    int (*direction_input)(struct gpio_chip *gc, unsigned int offset);
    int (*direction_output)(struct gpio_chip *gc, unsigned int offset, int value);
    int (*get)(struct gpio_chip *gc, unsigned int offset);
    int (*get_multiple)(struct gpio_chip *gc, unsigned long *mask, unsigned long *bits);
    void (*set)(struct gpio_chip *gc, unsigned int offset, int value);
    void (*set_multiple)(struct gpio_chip *gc, unsigned long *mask, unsigned long *bits);
    int (*set_config)(struct gpio_chip *gc, unsigned int offset, unsigned long config);
    int (*to_irq)(struct gpio_chip *gc, unsigned int offset);

    /* 编号与引脚数 */
    int   base;                     /* 全局编号起点 */
    u16   ngpio;                    /* 本 bank 引脚数 */
    u16   offset;                   /* 多 chip 共享同一设备时做偏移 */
    const char *const *names;       /* 引脚别名（可 NULL）*/
    bool  can_sleep;                /* 访问时需要睡眠（I2C/SPI 扩展器设为 true）*/

    /* 通用 GPIO (CONFIG_GPIO_GENERIC) */
    unsigned long (*read_reg)(void __iomem *reg);
    void (*write_reg)(void __iomem *reg, unsigned long data);
    void __iomem *reg_dat;
    void __iomem *reg_set;
    void __iomem *reg_clr;
    void __iomem *reg_dir_out;
    void __iomem *reg_dir_in;

    /* 中断 (CONFIG_GPIOLIB_IRQCHIP) */
    struct gpio_irq_chip irq;

    /* 有效引脚掩码 */
    unsigned long *valid_mask;
};
```

逐字段说明：

**`label`**——名称。用于 debugfs 和日志标识。ATK 板上：`"GPIOA"`、`"GPIOB"` 等。

**`gpiodev`**——指回 `gpio_device` 的反向指针。`gpio_device.chip` 指向 `gpio_chip`，`gpio_chip.gpiodev` 指回 `gpio_device`，构成双向引用。

**`parent`**——父设备。通常是 pin controller 的设备（STM32 上指向 `pinctrl` 节点对应的 `struct device`）。用于设备模型中的父子关系、PM 回调传递。

**`fwnode`**——firmware node。DT 系统上指向 `of_node`。用于 DTS 属性解析（如 `gpio-ranges`、`ngpios` 等）。

**`owner`**——模块引用计数。填 `THIS_MODULE`。

**核心回调（`request` / `free`）**——后门机制入口。驱动调 `gpiod_get()` 时，core 层分配 `gpio_desc` 后调用 `chip->request(chip, offset)`。STM32 上：
```
stm32_gpio_request(chip, offset)
  → pinctrl_find_gpio_range_from_pin_nolock()  // 验证引脚在范围内
  → stm32_gpio_rif_acquire_semaphore()          // MP257 安全权限检查
  → pinctrl_gpio_request(chip->base + offset)    // 进入 pinctrl 冲突检测
```

**核心回调（`direction_input` / `direction_output`）**——配置引脚方向。STM32 上 `direction_output` 内部会调用 pinctrl 后门写 MODER：

```
stm32_gpio_direction_output(chip, offset, 0)
  → __stm32_gpio_set(bank, offset, 0)          // 先写 BSRR 设电平
  → pinctrl_gpio_direction_output(gpio_num)    // 通过 pinctrl 写 MODER
```

**核心回调（`get` / `set`）**——读写电平。STM32 上直接操作寄存器：

| 驱动调 | gpio_chip 回调 | STM32 硬件操作 |
|--------|---------------|---------------|
| `gpiod_set_value(desc, 1)` | `chip->set(chip, offset, 1)` | `writel(BIT(offset), bank->base + BSRR)` |
| `gpiod_get_value(desc)` | `chip->get(chip, offset)` | `readl(bank->base + IDR) & BIT(offset)` |

**`get_direction`**——读取当前方向。STM32 通过读取 MODER 寄存器判断。

**`get_multiple` / `set_multiple`**——批量读写。一次调用操作多个引脚，提高效率。

**`set_config`**——配置电气参数。STM32 上通过此回调处理 `PIN_CONFIG_*` 参数（上下拉等）。

**`to_irq`**——GPIO → 中断号映射。`gpiod_to_irq(desc)` 的底层实现。STM32 上调用 `irq_create_fwspec_mapping()` 映射到 GIC。

**`base`**——全局编号起点。**已废弃**，新驱动应传 -1 自动分配。ATK 板上 GPIOA.base=0，GPIOB.base=16，GPIOH.base=112。

**`ngpio`**——本 bank 的引脚数。

**`can_sleep`**——`bool` 值。为 `true` 时，访问这个 GPIO 需要睡眠（如 I2C 扩展器）。此时 GPIO core 会强制使用 `gpiod_*_cansleep()` 变体，在可睡眠上下文中调用。

**`valid_mask`**——有效引脚掩码。某些 SoC 的特定 package 可能只有部分引脚可用，通过此掩码禁用不可用的引脚。

**`irq`**——`struct gpio_irq_chip`，将 GPIO bank 包装为一个中断控制器，实现 GPIO offset → Linux IRQ 号的映射。填充后 gpiolib core 在 `gpiochip_add_data()` 时自动完成中断控制器注册。对于不依赖 gpio_irq_chip 快捷路径的驱动（如 STM32），此字段保持零值，中断相关的内容由驱动自行管理，留待中断子系统章节详述。

ATK 板上的实例（每个 bank 一个 gpio_chip）：

```
gpiochip0 → GPIOA（base=0,  ngpio=16）
gpiochip1 → GPIOB（base=16, ngpio=16）
...
gpiochip7 → GPIOH（base=112, ngpio=14）
gpiochip9 → GPIOZ（base=176, ngpio=10）
```

### 3.5 gpio_desc：驱动手中的"不透明指针"

`gpio_desc` 是驱动开发者最终拿到的结构体。它被设计为**不透明（opaque）**——驱动只能通过 `gpiod_*` API 操作，不能直接访问内部字段。

定义在 `drivers/gpio/gpiolib.h`：

```c
struct gpio_desc {
    struct gpio_device  *gdev;   /* → 所属 gpio_device */
    unsigned long       flags;   /* 状态标志位 */
#ifdef CONFIG_DEBUG_FS
    const char          *label;  /* 申请者的标签（debugfs 显示用）*/
#endif
};
```

`gpio_desc[]` 数组在 `gpiochip_add_data()` 注册时由 core 层创建。core 层根据 `gpio_chip.ngpio` 分配数组，初始化每个描述的 `gdev` 回指指针（`drivers/gpio/gpiolib.c`）：

```c
// gpiochip_add_data_with_key() 中的简化流程
gdev->ngpio = gc->ngpio;                                       // 记录引脚数
gdev->descs = kcalloc(gc->ngpio, sizeof(*gdev->descs), GFP_KERNEL);  // 分配数组
for (i = 0; i < gc->ngpio; i++)
    gdev->descs[i].gdev = gdev;                                // 每个 desc 指回 gdev
```

与 `pin_desc`（§2.6）在注册时即完成全部初始化不同，`gpio_desc` 注册时只设置 `gdev` 字段。`flags` 和 `label` 在后续 `gpiod_get()` 调用时才被填充——`FLAG_REQUESTED` 在请求成功时设置，`label` 记录申请者名称，其他标志位由 DTS 中的 GPIO 标志（`GPIO_ACTIVE_LOW` 等）解析后写入。

逐字段说明：

**`gdev`**——指向所属 `gpio_device`。这是 gpio_desc 最关键的字段——通过它找到 `gpio_chip`，再调回调函数：

```
gpiod_set_value(desc, 1)
  → desc->gdev->chip->set(chip, offset, 1)
```

**`flags`**——状态标志位。记录了当前 GPIO 的状态，通过位运算管理：

| 标志位 | 含义 | 设置时机 |
|--------|------|---------|
| `FLAG_REQUESTED` | 已被申请 | `gpiod_get()` 成功时 |
| `FLAG_IS_OUT` | 方向为输出 | `gpiod_direction_output()` 时 |
| `FLAG_ACTIVE_LOW` | 低电平有效 | 由 DTS 中 `GPIO_ACTIVE_LOW` 标志设置 |
| `FLAG_OPEN_DRAIN` | 开漏输出 | 由 DTS 中 `GPIO_OPEN_DRAIN` 标志设置 |
| `FLAG_OPEN_SOURCE` | 开源输出 | 由 DTS 设置 |
| `FLAG_USED_AS_IRQ` | 被用作中断 | `gpiod_to_irq()` 时 |
| `FLAG_PULL_UP` | 内部上拉 | 由 DTS 标志设置 |
| `FLAG_PULL_DOWN` | 内部下拉 | 由 DTS 标志设置 |
| `FLAG_BIAS_DISABLE` | 禁止上下拉 | 由 DTS 标志设置 |

**`label`**——申请者的标识字符串（仅 debugfs 编译时有效）。来自 `gpiod_get(dev, con_id, flags)` 中的 `con_id`。在 `/sys/kernel/debug/gpio` 中显示的标签就是此字段。

**为什么是不透明的？** 旧版整型 API 时代，驱动直接传 GPIO 编号：

```c
gpio_request(47, "led");   // 47 在 GPIOA 还是 GPIOB？靠猜
gpio_set_value(47, 1);
```

描述符 API 不再暴露编号：

```c
struct gpio_desc *desc = devm_gpiod_get(dev, NULL, GPIOD_OUT_LOW);
// 驱动只知道这是个"描述符"，不知道是哪个 bank 的哪个引脚
gpiod_set_value(desc, 1);
```

gpio_desc 与 gpio_device.descs[] 数组的关系（以 GPIOA 为例）：

```
gpio_device（GPIOA）
  .descs[0]  → 描述 PA0，.gdev → GPIOA, .flags=0
  .descs[1]  → 描述 PA1，.gdev → GPIOA, .flags=0
  ...
  .descs[4]  → 描述 PA4，.gdev → GPIOA, .flags=REQUESTED|IS_OUT, .label="uart1_tx"
  .descs[5]  → 描述 PA5，.gdev → GPIOA, .flags=0
```

当 `devm_gpiod_get(dev, NULL, GPIOD_OUT_LOW)` 解析 DTS 中的 `gpios = <&gpioa 5 GPIO_ACTIVE_LOW>` 时，core 层找到 GPIOA 的 gpio_device，返回 `descs[5]` 的指针。

### 3.6 功能实现：从 API 到硬件寄存器

#### 输出场景：gpiod_set_value(desc, 1) 完整路径

```c
gpiod_set_value(desc, 1)
  → desc->gdev->chip->set(chip, offset, 1)
  → STM32：writel(BIT(5), bank->base + BSRR)
    // BSRR（Bit Set/Reset Register）：写 bit 5 = 1 将 PA5 设为高电平
```

ACTIVE_LOW 的处理由 gpiolib core 在调用 `chip->set()` 之前完成：

```c
int gpiod_set_value(struct gpio_desc *desc, int value)
{
    // 如果 FLAG_ACTIVE_LOW 被设置，反转 value
    if (test_bit(FLAG_ACTIVE_LOW, &desc->flags))
        value = !value;
    // 再调硬件操作
    desc->gdev->chip->set(chip, offset, value);
}
```

DTS 中 `GPIO_ACTIVE_LOW` 标志在 `devm_gpiod_get()` 解析时自动设置到 `desc->flags`，驱动不用关心。

#### 输入场景：gpiod_get_value(desc) 完整路径

```c
gpiod_get_value(desc)
  → desc->gdev->chip->get(chip, offset)
  → STM32：readl(bank->base + IDR) & BIT(5)
    // IDR（Input Data Register）：读取 PA5 的当前电平
```

同样的 ACTIVE_LOW 反转在返回值上也适用。

#### 中断场景：gpiod_to_irq(desc) + request_irq 完整路径

```c
gpiod_to_irq(desc)
  → desc->gdev->chip->to_irq(chip, offset)
  → STM32：stm32_gpio_to_irq(chip, offset)
    → irq_create_fwspec_mapping(&fwspec)
    → 返回 IRQ number（Linux 中断号）
```

返回的中断号可以用于 `devm_request_threaded_irq()` 注册中断处理函数。GPIO 控制器 DTS 中需声明 `interrupt-controller`：

```dts
gpioa: gpio@44240000 {
    interrupt-controller;           /* 这个 bank 也是中断控制器 */
    #interrupt-cells = <2>;         /* <偏移 触发类型> */
    interrupts = <GIC_SPI 139 IRQ_TYPE_LEVEL_HIGH>;  /* 到 GIC 的连接 */
};
```

#### ATK 板实例：LED（输出）完整路径

```dts
led {
    gpios = <&gpioh 4 GPIO_ACTIVE_LOW>;  /* PH4 */
};
```

```c
desc = devm_gpiod_get(dev, NULL, GPIOD_OUT_LOW);
// 1. of_find_gpiochip_by_node(GPIOH 的 of_node) → gpio_device(id=7)
// 2. desc = gpio_device.descs[4]
// 3. 设置 FLAG_ACTIVE_LOW（从 DTS GPIO_ACTIVE_LOW 解析得到）
// 4. desc->gdev->chip->direction_output(chip, 4, 0) → 写 BSRR + pinctrl 后门
// 5. 返回 desc

gpiod_set_value(desc, 1);
// 1. 发现 FLAG_ACTIVE_LOW，反转 value = 0
// 2. desc->gdev->chip->set(chip, 4, 0) → 写 BSRR bit 4 为 0 → 硬件输出低电平
// 3. 原理图：LED 正极接 3.3V，负极经电阻到 PH4 → PH4=0 时 LED 亮
```


## 4. 桥梁：gpio_ranges

> 一个数据结构、两个链表、一条总线——Pinctrl 和 GPIO 就是这样拼在一起的。

### 4.1 概述：两个独立子系统的编号翻译问题

前面两节讲了两个子系统各有各的编号体系：

- **Pinctrl** 用 pin 号（0~npins-1）索引 `pin_desc[]`
- **GPIO** 用全局编号（如 47）索引 `gpio_desc[]`

当 GPIO core 调 `pinctrl_gpio_request(47)` 到 pinctrl 做冲突检测时，它传的是全局编号 47，但 pinctrl 不认识这个数字——它只认识自己的 pin 号（如 pin 15）。**gpio-ranges 就是连接这两套编号体系的翻译表。**

### 4.2 桥梁的数据结构关系

```
                  pinctrl_dev                          
              ┌─────────────────┐                     
              │  .gpio_ranges   │                     
              │  （链表头）      │                     
              └──────┬──────────┘                     
                     │                                
       ┌─────────────┼──────────────────────────┐     
       │             │                          │     
       │    pinctrl_gpio_range              pinctrl_gpio_range
       │  ┌──────────────────────┐       ┌──────────────────────┐
       ├──│ .node (pinctrl 链表) │   ├──│ .node (pinctrl 链表) │
       │  │ .node2 (GPIO 链表)   │───→│ .node2 (GPIO 链表)   │───→ ...
       │  │ .base=0, .npins=16  │   │  │ .base=16, .npins=16 │
       │  │ .gc → gpio_chip(GPIOA)│  │  │ .gc → gpio_chip(GPIOB)│
       │  └──────────────────────┘   │  └──────────────────────┘
       │                              │
       ▼                              ▼
  gpio_device(GPIOA)            gpio_device(GPIOB)
  ┌─────────────────┐          ┌─────────────────┐
  │ .pin_ranges     │          │ .pin_ranges     │
  │ （链表头）       │          │ （链表头）       │
  └─────────────────┘          └─────────────────┘
```

**关键点：** 一个 pinctrl_dev 管理多个 GPIO bank，每个 bank 对应一个独立的 `pinctrl_gpio_range`。Pinctrl 侧所有 range 挂在同一个 `pinctrl_dev.gpio_ranges` 链表上；GPIO 侧每个 bank 各自持有自己的 range（挂在各自的 `gpio_device.pin_ranges` 链表上）。翻译时两端各自遍历自己的链表。

### 4.3 pinctrl_gpio_range：翻译表

```c
struct pinctrl_gpio_range {
    struct list_head node;           /* 挂在 pinctrl_dev->gpio_ranges 中 */
    struct list_head node2;          /* 挂在 gpio_device->pin_ranges 中 */
    const char *name;                /* 名称（如 "GPIOA"）*/
    unsigned int id;                 /* ID */
    unsigned int base;               /* GPIO 全局编号起点 */
    unsigned int pin_base;           /* pinctrl pin 号起点 */
    unsigned int npins;              /* 范围长度 */
    unsigned const *pins;            /* 具体引脚列表（通常为 NULL，使用连续映射）*/
    struct gpio_chip *gc;            /* → 指向对应的 gpio_chip */
};
```

逐字段说明：

**`node` / `node2`——双链表节点**

这是桥梁最关键的设计——一个 range 同时挂在两个子系统的链表上：

| 字段 | 挂在哪个链表 | 谁遍历 |
|------|-------------|--------|
| `node` | `pinctrl_dev->gpio_ranges` | Pinctrl core 的 `pinctrl_find_gpio_range_from_pin()` |
| `node2` | `gpio_device->pin_ranges` | GPIO core 的 `gpiochip_find_pin_range()` |

**为什么需要两个链表节点？** 因为两个子系统都有"从自己的编号翻译到对方编号"的需求，但各自使用的遍历入口不同。Pinctrl 侧需要一个链表来"给定全局编号，找对应 range"，GPIO 侧需要一个链表来"给定 pin 号，找对应 bank"。同一个 range 数据，两套索引。

**`base`——GPIO 全局编号起点**

这个 range 对应的 GPIO 全局编号的起始值。例如 GPIOA 的 `base = 0` 表示 PA0 的全局编号是 0。这个值来自 DTS 中 `gpio-ranges` 的第二个参数，或者在 probe 时由驱动自动分配。

**`pin_base`——pinctrl pin 号起点**

这个 range 对应的 pinctrl pin 号的起始值。STM32 上由于 `gpio-ranges` 一对一连续映射，`pin_base` 等于 `base`。但在不连续映射的 SoC 上，两者可以不同。

**`npins`——范围长度**

这个 range 覆盖的引脚数。遍历链表时使用 `[base, base + npins)` 判断一个全局编号是否属于这个 range。

**`pins`——非连续映射的引脚列表**

当引脚映射不是连续的一段时使用——`pins` 指向一个具体引脚号的数组，数组长度是 `npins`。STM32 上为 NULL（因为映射是连续的，直接用 `pin_base + offset` 计算）。

**`gc`——指向对应的 gpio_chip**

通过这个字段，翻译完成后就能直接找到对应的 GPIO 控制器进行操作。这个指针在 `pinctrl_add_gpio_range()` 时设置。

### 4.4 映射安装流程：从 DTS 到链表节点

`gpio-ranges` 属性在 DTS 中声明，由 GPIO core 在 `gpiochip_add_data()` 时解析并创建 `pinctrl_gpio_range`，然后挂入两个链表中：

```
DTS:  gpio-ranges = <&pinctrl 0 0 16>;
                                │  │  └─ count（16 个引脚）
                                │  └──── pin_base（pinctrl pin 号起点 0）
                                └─────── base（GPIO 全局编号起点 0）
                                                 ↓ of_gpiochip_add()
    创建 pinctrl_gpio_range{base=0, pin_base=0, npins=16, gc=&gpioa_gpio_chip}
       ├── 挂入 pinctrl_dev->gpio_ranges 链表  ← 后门冲突检测时遍历
       └── 挂入 gpio_device->pin_ranges 链表    ← 验证 GPIO 请求时遍历
```

### 4.5 翻译过程：全局编号 → pinctrl pin 号

当 GPIO core 调用 `pinctrl_gpio_request(global_gpio_num)` 进入后门时，翻译过程如下：

```
入参：global_gpio_num = 47

1. 遍历 pinctrl_dev->gpio_ranges 链表
   找到第一个满足 range->base <= 47 < range->base + range->npins 的 range
   → 找到 range: base=32, npins=14, pin_base=32

2. 计算 pinctrl pin 号
   pin = range->pin_base + (47 - range->base) = 32 + 15 = 47

3. 在 pinctrl 的 pin_desc[47] 中做冲突检测
   ├─ mux_usecount > 0 且 mux_owner 不是当前请求者？→ -EBUSY
   ├─ gpio_owner != NULL？→ -EBUSY
   └─ 都空闲 → 标记 gpio_owner，返回成功
```

### 4.6 STM32MP257 的实际映射

从 ATK 板的 DTS 看：

```dts
pinctrl: pinctrl@44240000 {
    gpioa: gpio@44240000 { gpio-ranges = <&pinctrl 0 0 16>; };
    gpiob: gpio@44250000 { gpio-ranges = <&pinctrl 0 16 16>; };
    gpioc: gpio@44260000 { gpio-ranges = <&pinctrl 0 32 14>; };
    ...
};
```

`gpio-ranges` 的四元组含义：`<&pinctrl GPIO_offset pin_base count>`。

实际映射关系：

```
GPIO bank    pinctrl pin 范围    GPIO 全局编号范围    翻译公式
GPIOA        0~15               0~15                 pin = gpio
GPIOB        16~31              16~31                pin = gpio
GPIOC        32~45              32~45                pin = gpio
...
GPIOI        144~155            144~155              pin = gpio
GPIOZ        176~185            176~185              pin = gpio（安全域）
```

STM32 上 `pin_base == base`，是因为 `gpio-ranges` 配置为一一对应。这不是必然的——在其他 SoC 上可能 GPIO 全局编号 0~15 对应 pinctrl pin 32~47，即 `base=0, pin_base=32`。

### 4.7 后门机制的完整调用链

桥梁不仅做翻译，还串联了两个子系统的操作：

```
gpiod_get(dev, con_id, flags)
  → gpiochip_find_pin_range(base + offset)      // 查 pin_ranges 验证有效性
  → chip->request(chip, offset)                   // 调 stm32_gpio_request
    → pinctrl_gpio_request(gpio_num)               // 进入 pinctrl
      → 遍历 gpio_ranges 翻译 pin 号               // 全局编号 → pin 号
      → pin_request(pin)                           // 冲突检测 + 标记 owner

gpiod_direction_output(desc, 0)
  → desc->gdev->chip->set(chip, offset, 0)         // 先写 BSRR 设电平
  → pinctrl_gpio_direction_output(gpio_num)        // 进入 pinctrl 写 MODER
    → 遍历 gpio_ranges 翻译 pin 号
    → pmxops->gpio_set_direction(pctldev, range, offset, output)
      → stm32_pmx_set_mode(bank, pin, mode=OUTPUT) // 写 MODER 切 GPIO 输出
```

这就是"后门机制"的完整路径——GPIO 的操作最终通过桥梁翻译后，由 pinctrl 完成对 MODER 寄存器的配置。

---

## 5. 两种硬件架构对比（STM32 vs i.MX6ULL）

两个子系统设计得再好，最终落到硬件上时，不同的 SoC 实现方式决定了软件路径不同。

### 5.1 STM32（合体架构）

```
GPIO bank 地址空间（如 gpioa: gpio@44240000）：
  偏移 0x00:  MODER    ← 功能选择（Pinctrl）+ 方向（GPIO）
  偏移 0x04:  OTYPER   ← 输出类型
  偏移 0x08:  OSPEEDR  ← 输出速度
  偏移 0x0C:  PUPDR    ← 上下拉
  偏移 0x10:  IDR      ← 输入数据
  偏移 0x14:  ODR      ← 输出数据
  偏移 0x20:  AFRL     ← 外设功能选择
  偏移 0x24:  AFRH     ← 外设功能选择
  偏移 0x48:  ADVCFGR  ← 高级配置（MP257 新增）
```

**关键特征**：没有独立的 Pinctrl 硬件模块。"Pinctrl"是对 MODER/AFR 中"功能配置"位段的软件抽象。

**后果**：Pinctrl 和 GPIO 必须合体。`pinctrl_desc` 和 `gpio_chip` 由同一个驱动注册（`stm32_pctl_probe`）。GPIO 请求时，后门机制能自动写 MODER，因为 pinctrl 和 GPIO 是同一段代码、同一组寄存器。

### 5.2 i.MX6ULL（分离架构）

```
独立地址空间：
  IOMUXC（引脚复用控制器）:       GPIO 控制器:
    SW_MUX_CTL（功能选择）          DR（数据寄存器）
    SW_PAD_CTL（电气配置）          GDIR（方向寄存器）
                                    PSR（状态寄存器）
```

**关键特征**：IOMUXC 和 GPIO 是两套独立的硬件模块，有独立的地址空间。

**后果**：Pinctrl 和 GPIO 可以拆成不同的驱动（`pinctrl-imx6ul.c` + `gpio-mxc.c`）。GPIO 驱动写不了 IOMUXC 的寄存器，所以后门机制**只做冲突检测，不能自动配 GPIO 模式**。DTS 中必须写 `pinctrl-0 = <&led_pin>` 让 pinctrl 在 probe 时配好 IOMUXC。

### 5.3 两种架构的决定性影响

| 特性 | STM32（合体） | i.MX6ULL（分离） |
|------|-------------|-----------------|
| 硬件 Pinctrl 模块 | **无**（GPIO 寄存器兼任） | **有**（独立 IOMUXC 地址空间） |
| Pinctrl ops 操作哪些寄存器 | MODER / AFR / OSPEEDR / PUPDR | IOMUXC_SW_MUX_CTL / SW_PAD_CTL |
| GPIO chip 操作哪些寄存器 | MODER / ODR / IDR / BSRR | DR / GDIR / PSR |
| 后门自动配 GPIO？ | ✅ `gpio_set_direction` 写 MODER | ❌ 只能冲突检测 |
| DTS pinctrl-0 是否必须 | 可选（后门可代劳） | **必须**（不配 IOMUXC 不工作） |
| GPIO 与 Pinctrl 驱动 | 同一个驱动（pinctrl-stm32.c） | 两个独立驱动 |
| gpio_chip 注册时机 | 在 pinctrl probe 中扫描子节点注册 | 独立 platform_driver probe |

---

## 6. 总结

本章围绕"同一个引脚，两套视角"，按四个层次展开：

- **§2 Pinctrl 侧**：引脚枚举（pinctrl_ops）、复用功能（pinmux_ops）、电气配置（pinconf_ops）三件事各自的数据结构与 ops 设计
- **§3 GPIO 侧**：gpio_device 管设备抽象、gpio_chip 管硬件操作、gpio_desc 管描述符的三层分离架构
- **§4 桥梁**：pinctrl_gpio_range 通过双链表连接两个子系统，完成全局编号 ↔ Pinctrl pin 号的翻译
- **§5 硬件对比**：两种 SoC 架构（合体 vs 分离）对框架路径的决定性影响
