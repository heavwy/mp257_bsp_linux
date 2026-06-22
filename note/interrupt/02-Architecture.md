# 核心数据结构与设计原理

> 中断子系统的静态结构（不涉及代码流程，那是后面文档的事）。
>
> **字数**：预估约 25000 字 · **建议阅读时间**：60~90 分钟

---

## 1. 设计思想：从信号路径看四层架构与四大数据结构

### 1.1 先看中断信号的完整路径

把"中断"想象成一个物理信号——它从外设出发，经过中断控制器，最终到达 CPU，触发一段软件代码执行。我们可以沿着这条路径走一遍，看内核每一步需要解决什么问题。

以 STM32MP257 开发板上的按键 PH5 按下为例：

```
PH5 按键按下（GPIO 电平变化）
    │
    │  步骤 1：中断信号从哪条路进来？
    │  GPIO bank 检测到电平变化，输出信号给 EXTI
    │
    ▼
EXTI 边沿检测 → 设置 pending 位（RPR/FPR）
    │
    │  步骤 2：这个信号对应 GIC 的哪个中断号？
    │  EXTI 通过硬件连线把事件送给 GIC 的某个 SPI
    │
    ▼
GIC Distributor 接收中断信号
    │  步骤 3：GIC 知道这个中断该发给哪个 CPU 吗？
    │  查配置的 affinity——发给 CPU0
    │
    ▼
CPU0 的 GIC CPU Interface 收到中断
    │  步骤 4：CPU 怎么知道自己该处理什么？
    │  读 GICC_IAR 拿到中断号，查 irq_domain 转成 virq
    │
    ▼
irq_desc[virq].handle_irq 被调用
    │  步骤 5：怎么开关这个中断、怎么确认它？
    │  调 irq_chip 的回调——mask/ack/eoi
    │
    ▼
irqaction->handler() 被执行
    │  步骤 6：最终谁处理这个中断？
    │  设备驱动注册的 handler——是按键消抖还是马里奥跳跃
    │
    ▼
真正的"处理"完成
```

把这些步骤背后的技术问题**归纳为四个，每个对应一个核心数据结构**：

| # | 问题 | 管什么 | 核心数据结构 | 所属层 | 本章§ |
|---|------|--------|-------------|-------|-------|
| 1 | **识别** | 这个硬件中断号对应哪个 Linux IRQ 号？ | `struct irq_domain` | Core 层 | §3 |
| 2 | **控制** | 怎么 mask/unmask/ack/eoi 这个中断？ | `struct irq_chip` | 控制器驱动层（core 定义接口，驱动实现） | §2 |
| 3 | **流转** | 什么时候 mask、什么时候 ack、什么时候 EOI？ | `struct irq_desc`（flow handler） | Core 层 | §4 |
| 4 | **处理** | 最终谁来做真正的业务处理？ | `struct irqaction` | 消费者层 | §5 |

这四个数据结构分别属于中断子系统的**四个软件层**。下面用分层架构图展示每层的定位和各数据结构在其中的位置：

```
┌─────────────────────────────────────────────────────────────────────┐
│  消费者层 (Consumer Layer)                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  struct irqaction      ← 设备驱动调 request_irq 注册进来       │   │
│  │  .handler              顶半部回调（IRQ off）                   │   │
│  │  .thread_fn            底半部回调（IRQ on，可睡眠）            │   │
│  └──────────────────────────────────────────────────────────────┘   │
│  管什么：问题 4——中断最终谁处理                                       │
├─────────────────────────────────────────────────────────────────────┤
│  Core 层 (通用中断逻辑, kernel/irq/)                                 │
│                                                                      │
│  ┌────────────────────────────┐  ┌──────────────────────────────┐   │
│  │  struct irq_domain（问题 1）  │  │  struct irq_desc（问题 3）     │   │
│  │  翻译: (ctrlr, hwirq)→virq  │  │  .handle_irq → flow handler  │   │
│  │  .xlate / .alloc / .map    │  │  .action → irqaction 链表     │   │
│  │  .linear_revmap[] 快速查找  │  │  .lock / .depth / .kstat     │   │
│  └────────────────────────────┘  └──────────────────────────────┘   │
│                                                                      │
│  管什么：问题 1（编号翻译）+ 问题 3（流程控制）                        │
├─────────────────────────────────────────────────────────────────────┤
│  中断控制器驱动层 (Controller Driver Layer, drivers/irqchip/)        │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  struct irq_chip（问题 2）                                     │   │
│  │  → GIC 实现: gic_chip { .irq_mask, .irq_unmask, .irq_eoi... }  │   │
│  │  → EXTI 实现: stm32mp_exti_chip { .irq_mask, .irq_eoi... }     │   │
│  │  同时注册 irq_domain: gic_irq_domain_hierarchy_ops             │   │
│  └──────────────────────────────────────────────────────────────┘   │
│  管什么：问题 2——操作中断控制器寄存器                                 │
├─────────────────────────────────────────────────────────────────────┤
│  硬件层 (Hardware Layer)                                            │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  GIC-400 寄存器: GICD_ICENABLER / GICD_ISENABLER / GICC_EOIR │   │
│  │  EXTI 寄存器: IMR / RPR / FPR / RTSR / FTSR / TRG            │   │
│  └──────────────────────────────────────────────────────────────┘   │
│  管什么：真正产生和传递中断信号的物理电路                             │
└─────────────────────────────────────────────────────────────────────┘
```

各层之间的关系：**消费者层**调 `request_irq` 注册 irqaction，不碰底层寄存器；**Core 层**管理 irq_desc 和 irq_domain，定义 irq_chip 接口，不直接操作硬件；**控制器驱动层**实现 irq_chip 回调并注册 irq_domain，操作寄存器；**硬件层**是物理电路。

**核心理念**：四个问题不可分割但必须分开实现。换 GIC 到 GICv3 只需要换控制器驱动层（`irq-gic-v3.c`）；加 EXTI 这样的新控制器只需要写新驱动（新的 irq_chip + irq_domain ops），挂到 Core 层的层级域链上。分开的收益在于**每一层的修改不影响其他层**。

### 1.2 四层架构分层详解

和 pinctrl 子系统一样，中断子系统也采用 **核心层 + 控制器驱动** 的分离架构。但中断子系统更复杂——它的「核心层」内部还按功能分成了三组职责：

```
┌─────────────────────────────────────────────────────────────────────┐
│  消费者层 (Consumer / Device Drivers)                  kernel/irq/  │
│                                                                      │
│  ┌───────────────────────────────────────────────────┐              │
│  │  设备驱动（UART/I2C/SDMMC/GPIO按键驱动等）          │              │
│  │    ● request_irq / request_threaded_irq            │              │
│  │    ● devm_request_irq / gpiod_to_irq              │              │
│  │    ● enable_irq / disable_irq / synchronize_irq   │              │
│  │    ● handler(): 从 irqaction->handler 回调入口      │              │
│  └───────────────────────────────────────────────────┘              │
│          │                                                          │
│          │  通过 irq_desc 中的 handle_irq 和 action 间接调用         │
│          ▼                                                          │
├─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤
│  CORE 层 (通用逻辑)                               kernel/irq/     │
│                                                                      │
│  ┌───────────────────────────────────────────────────┐              │
│  │  irq_desc（流程控制）| irq_domain（编号映射）        │              │
│  │  irq_chip 接口定义（core 声明，驱动实现）            │              │
│  │                                                     │              │
│  │  核心文件: chip.c / irqdomain.c / manage.c          │              │
│  └───────────────────────────────────────────────────┘              │
│          │                                                          │
│          │  通过 irq_chip 中的回调间接操作                           │
│          ▼                                                          │
├─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤
│  中断控制器驱动层 (Hardware Operations)           drivers/irqchip/ │
│                                                                      │
│  ┌──────────────────────────────────┐  ┌──────────────────────────┐ │
│  │  GICv2 (irq-gic.c)              │  │  STM32MP EXTI            │ │
│  │  gic_chip                       │  │  (irq-stm32mp-exti.c)    │ │
│  │    .irq_mask  = gic_mask_irq    │  │  stm32mp_exti_chip       │ │
│  │    .irq_unmask = gic_unmask_irq │  │    .irq_mask = ..._mask  │ │
│  │    .irq_eoi = gic_eoi_irq      │  │    .irq_eoi = ..._eoi    │ │
│  │    .irq_set_type = gic_set_type │  │    +parent callback 链   │ │
│  │  gic_irq_domain_hierarchy_ops   │  │  stm32mp_exti_domain_ops│ │
│  └──────────────────────────────────┘  └──────────────────────────┘ │
│          │                                                          │
│          ▼  写寄存器                                                │
│  ┌──────────────────────────────────────────────────────────────────┐│
│  │  STM32MP257 硬件寄存器                                          ││
│  │  GIC: GICD_ICENABLER / GICD_ISENABLER / GICC_EOIR / GICC_IAR  ││
│  │  EXTI: IMR / RPR / FPR / RTSR / FTSR                          ││
│  └──────────────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────────────┘
```

各层的定位：

| 层 | 管什么 | 不做什么 |
|----|--------|---------|
| **消费者层** | 调用 `request_irq` 注册 handler，调 API 控制中断状态 | 不碰硬件寄存器 |
| **Core 层** | 流程控制（flow handler）+ 编号翻译（irq_domain）+ irq_chip 接口定义 | 不直接操作硬件寄存器（通过 irq_chip 回调） |
| **控制器驱动层** | 实现 irq_chip 回调，注册 irq_domain，写硬件寄存器 | 不决定中断处理的时序（由 Core 层 flow handler 决定） |
| **硬件层** | GIC/EXTI 物理电路，产生和传递中断信号 | 不参与软件决策 |

> **与 pinctrl 的类比**：pinctrl 的三层架构（Consumer → Core → Controller）在这里变成了四层，因为中断子系统把 Core 层拆成了**流程控制（irq_desc/flow handlers）** 和**编号映射（irq_domain）** 两个独立职责。pinctrl 的核心层不需要独立的编号映射层，因为它管理的是「引脚号」这个静态命名空间；而中断子系统的 hwirq → virq 映射是**动态分配**的，多个中断控制器的中断号可能重叠，必须有一个独立的翻译层。

### 1.3 为什么拆成四层——拆和不拆的区别

如果不拆，会怎么样？回到 v2.4 之前的「超级处理器 `__do_IRQ()`」时代（见 00-History §1），一个中断来了，所有事情在一个函数里完成：

- 查中断类型（电平/边沿）
- ack（写 PIC 寄存器）
- mask / unmask
- 决定是否向下传递
- 调用设备 handler

这个模型的问题：**你在一个地方耦合了「怎么操作硬件」和「什么时候做什么操作」**。

四层架构的核心哲学是：

```
问题 = "做什么"  →  由程序员在 DTS + 驱动代码中 配置
方案 = "怎么做"  →  由对应的 layer 提供回调
流程 = "什么时候做" → 由 flow handler 在 irq_desc 中 固定
```

- 驱动开发者只需要管**做什么**（DTS 写 trigger type，代码中设 affinity）
- irq_chip 回调实现**怎么做**（mask 是写哪个寄存器）
- flow handler 决定了**什么时候做什么**（入口 mask 还是出口 unmask）

简单说：**一个人想知道怎么开车，不需要了解内燃机的工作原理；但造车的人必须把油门、刹车、离合分开设计。**

### 1.4 本章结构对应关系

下面这张表帮你定位本章每个部分在四层架构中的位置：

| 层 | 核心数据结构 | § 章节 | 实际操作 |
|----|--------------|--------|---------|
| **消费者层** | `struct irqaction` | §5 | 设备驱动注册和销毁 |
| **Core 层** | `struct irq_desc` + `struct irq_domain` + `struct irq_chip`（接口） | §2(irq_chip) + §3(domain) + §4(desc) | flow control / 编号翻译 / chip 接口定义 |
| **控制器驱动层** | `struct irq_chip`（实现） + 注册 irq_domain | §2 | mask/unmask/ack/eoi/set_type 的实际硬件操作 |
| **硬件层** | GIC/EXTI 寄存器 | §6 | 物理中断信号的传递 |

全文以 STM32MP257 上的 GIC + EXTI 为实际参考平台。

---

## 2. struct irq_chip：封装中断控制器的硬件差异

中断子系统的第一层叫 irq_chip。它的职责非常简单：**把「操作中断控制器的寄存器」封装成统一回调接口**。

每个中断控制器虽然都提供 mask/unmask/ack/eoi 等功能，但寄存器的地址、位域、操作顺序各不同。irq_chip 的作用就是把这些差异挡住，让上层（flow handler）只需调用 `irq_data->chip->irq_mask(d)`，不用关心底下是 GIC 还是 EXTI。

### 2.1 irq_chip 结构体全景

`struct irq_chip` 定义在 `include/linux/irq.h`，是中断子系统中**回调函数最多**的结构体（超过 30 个字段）。但驱动开发者只实现其中一小部分——其余可以 NULL（core 层会用默认行为代替）。

完整定义如下：

```c
struct irq_chip {
    const char        *name;                           /* 芯片名（/proc/interrupts 显示）*/

    /* 控制组 */
    unsigned int    (*irq_startup)(struct irq_data *d);
    void            (*irq_shutdown)(struct irq_data *d);
    void            (*irq_enable)(struct irq_data *d);
    void            (*irq_disable)(struct irq_data *d);
    void            (*irq_ack)(struct irq_data *d);        /* 确认中断 */
    void            (*irq_mask)(struct irq_data *d);        /* 屏蔽中断 */
    void            (*irq_mask_ack)(struct irq_data *d);   /* 屏蔽+确认一起 */
    void            (*irq_unmask)(struct irq_data *d);      /* 取消屏蔽 */
    void            (*irq_eoi)(struct irq_data *d);         /* 结束中断 */

    /* 配置组 */
    int             (*irq_set_affinity)(struct irq_data *d,        /* SMP 亲和性 */
                                         const struct cpumask *dest, bool force);
    int             (*irq_retrigger)(struct irq_data *d);           /* 重新触发 */
    int             (*irq_set_type)(struct irq_data *d, unsigned int flow_type);

    /* 电源组 */
    int             (*irq_set_wake)(struct irq_data *d, unsigned int on);

    /* 特殊组 */
    int             (*irq_get_irqchip_state)(struct irq_data *d,   /* 查询硬件状态 */
                                              enum irqchip_irq_state which, bool *state);
    int             (*irq_set_irqchip_state)(struct irq_data *d,   /* 设置硬件状态 */
                                              enum irqchip_irq_state which, bool state);
    void            (*irq_print_chip)(struct irq_data *d, struct seq_file *p);
    int             (*irq_request_resources)(struct irq_data *d);
    void            (*irq_release_resources)(struct irq_data *d);
    void            (*irq_compose_msi_msg)(struct irq_data *d, struct msi_msg *msg);
    void            (*irq_write_msi_msg)(struct irq_data *d, struct msi_msg *msg);
    void            (*ipi_send_single)(struct irq_data *d, unsigned int cpu);
    void            (*ipi_send_mask)(struct irq_data *d, const struct cpumask *dest);

    unsigned long    flags;                              /* IRQCHIP_* 标志 */
};
```

所有 irq_chip 回调的第一个参数都是 **`struct irq_data *d`**。它是什么？它是"本次中断操作的目标信息"——记录了当前正在操作哪个中断、这个中断关联哪个控制器、硬件中断号是多少。

```c
struct irq_data {
    u32                 mask;           /* 预计算位掩码（加速寄存器访问）*/
    unsigned int        irq;            /* Linux IRQ 号（virq）*/
    unsigned long       hwirq;          /* 硬件中断号 */
    struct irq_common_data *common;     /* 共享数据（affinity 等）*/
    struct irq_chip     *chip;          /* 指向这个中断对应的 irq_chip */
    struct irq_domain   *domain;        /* 所属的 irq_domain */
    struct irq_data     *parent_data;   /* 层级域中的父层 irq_data */
    void                *chip_data;     /* 芯片私有数据 */
};
```

只看最关键的几个字段：

| 字段 | 含义 | 在回调中怎么用 |
|------|------|--------------|
| `irq` | Linux 虚拟中断号 | 芯片不需要它，供 core 层在回调路径中查表 |
| `hwirq` | 硬件中断号 | GIC/EXTI 根据它算出寄存器 bit 位：`BIT(hwirq % 32)` |
| `chip` | irq_chip 指针 | `d->chip->irq_mask(d)`——core 层就是通过这个字段找到回调函数 |
| `domain` | 所属 irq_domain | 用于反向查找 `irq_find_mapping()` |
| `parent_data` | 父层 irq_data | 层级域跨控制器传递——`irq_chip_mask_parent(d)` 内部调 `d->parent_data->chip->irq_mask(d->parent_data)` |
| `chip_data` | 驱动私有数据 | GIC 的 chip_data 指向 `gic_chip_data *`（基址、中断数），EXTI 的 chip_data 指向 `stm32mp_exti_chip_data *`（bank、IMR cache） |

简单说：**irq_data 是 irq_chip 回调的"上下文"**——它告诉回调函数"你现在要操作哪个中断、它的控制器是谁、硬件编号是多少"。`struct irq_chip` 本身是全局的（gic_chip 只有一份），不同中断通过不同的 irq_data 来区分。

irq_data 更完整的解析（含 `parent_data` 在层级域中的链式结构）在 §4.2，那里结合 irq_desc 一起展开。目前你只需要知道：**所有的 irq_chip 回调都通过 `d` 拿到中断的硬件信息，然后操作寄存器。**

所有回调可以按功能分组：

| 分组 | 回调 | 作用 | 是否必须 |
|------|------|------|---------|
| **控制组** | `irq_startup` / `irq_shutdown` | 启动/关闭中断（非必须，有默认实现） | 否 |
| | `irq_enable` / `irq_disable` | 使能/禁止（默认就是 unmask/mask） | 否 |
| | `irq_mask` / `irq_unmask` | 屏蔽/取消屏蔽（核心回调） | **是** |
| | `irq_ack` | 确认（告诉硬件 CPU 已收到） | 看 flow handler |
| | `irq_eoi` | 结束中断（GIC 特有，priority drop + deactivate） | 看 flow handler |
| | `irq_mask_ack` | mask + ack 一起做（优化用，减少寄存器访问） | 否 |
| **配置组** | `irq_set_type` | 设置触发类型（电平/边沿） | 建议实现 |
| | `irq_set_affinity` | 设置中断分发到哪个 CPU（SMP） | 否 |
| **电源组** | `irq_set_wake` | 配置为唤醒源 | 否 |
| | `irq_suspend` / `irq_resume` | 芯片级休眠/恢复回调 | 否 |
| **特殊组** | `irq_retrigger` | 软件触发中断重新发送（模拟硬件中断） | 否 |
| | `irq_get/set_irqchip_state` | 查询/设置硬件中断状态 | 否 |
| | `irq_request/release_resources` | 资源管理（request 时调用） | 否 |

`irq_retrigger` 的作用是软件请求 GIC 重新触发一次中断——往 `GICD_ISPENDR` 写 1，对应的中断就会从 Inactive 变为 Pending。它被两个路径调用：
- `check_irq_resend()`（`kernel/irq/resend.c`）——`handle_fasteoi_irq` 末尾检查 `IRQS_PENDING` 位，如果有 pending 需要重发时调它
- `core.c` 中的 `irq_retrigger()` 通用接口

GIC 和 EXTI 都实现了这个回调：`gic_retrigger()` 直接写 GICD_ISPENDR；`stm32mp_exti_retrigger()` 写 SWIER 寄存器软件触发 EXTI event。

从 $1 的四个核心问题来看，irq_chip 主要回答**问题 2：怎么控制硬件**。

### 2.2 控制组：mask / unmask / ack / eoi

GIC 和 EXTI 的硬件设计完全不同，分开来看。

---
#### 2.2.1 GIC：四状态状态机驱动

GIC 为每个中断维护四种硬件状态，mask/ack/eoi 都是对状态机的操作：

| 状态 | Pending | Active | 含义 |
|------|---------|--------|------|
| Inactive | 0 | 0 | 中断线空闲 |
| Pending | 1 | 0 | 外设触发了中断，等待 CPU 处理 |
| Active | 0 | 1 | CPU 正在处理该中断 |
| Active & Pending | 1 | 1 | CPU 在处理期间，同一中断源再次触发 |

GICv2 有两种工作模式，由 GICC_CTLR 的 **EoImode** 位（bit[1]）控制，决定 EOI 时状态如何转换：

- **EOI mode 0（标准模式）**：写 GICC_EOIR = Priority Drop + Deactivate，Active → Inactive 一步到位
- **EOI mode 1（Split mode）**：写 GICC_EOIR 只做 Priority Drop（Active → Pending），之后再写 GICC_DIR 做 Deactivate（Pending → Inactive）。为虚拟化设计

**状态转换图（三组线性路径）：**

```
路径 A——标准 EOI mode 0：
 Inactive ──①──► Pending ──②──► Active ──③──► Inactive
 (P=0,A=0)      (P=1,A=0)      (P=0,A=1)     (P=0,A=0)

路径 B——再触发分支（两种 mode 均适用）：
 Active ──①'──► Active&Pending ──③''──► Pending ──①──► Active（再次触发）
 (P=0,A=1)      (P=1,A=1)         (P=1,A=0)  (立即)

路径 C——Split mode 代替路径 A 的 ③：
 Active ──④──► Pending ──④'──► Inactive
 (P=0,A=1)  (P=1,A=0)  (P=0,A=0)
```

| 编号 | 操作 | 转换 | 说明 |
|------|------|------|------|
| ① | 外设中断信号有效 | Inactive → Pending | 硬件自动 |
| ② | CPU 读 GICC_IAR | Pending → Active | `gic_handle_irq()` 入口，硬件自动 ack |
| ③ | 写 GICC_EOIR（mode 0） | Active → **Inactive** | Priority Drop + Deactivate 一步完成 |
| ④ | 写 GICC_EOIR（mode 1） | Active → **Pending** | 只 Priority Drop |
| ④' | 写 GICC_DIR（mode 1） | Pending → **Inactive** | 真正的 Deactivate |
| ①' | 外设信号再次有效 | Active → Active & Pending | 电平保持或新边沿 |
| ③'' | 写 GICC_EOIR（任一种 mode） | Active & Pending → **Pending** | Active 清 0，Pending 保留 |

**mask/unmask 不在状态机之内**——它是状态机入口的阀门。mask 时外设信号被拦住，状态机收不到①的触发信号。

##### GIC 的 mask

写 GICD_ICENABLER（WO 寄存器），写 1 关中断，不依赖当前值，不需 RMW：

```c
static void gic_mask_irq(struct irq_data *d)
{
    gic_poke_irq(d, GIC_DIST_ENABLE_CLEAR);
}
static void gic_unmask_irq(struct irq_data *d)
{
    gic_poke_irq(d, GIC_DIST_ENABLE_SET);
}
```

GICD_ICENABLER 偏移 0x180，每个 bit 对应一个中断号。

##### GIC 的 ack

ack = 读 GICC_IAR，硬件自动完成 Pending → Active，**没有软件回调**：

```c
/* gic_handle_irq 入口 */
irqstat = readl_relaxed(cpu_base + GIC_CPU_INTACK);
irqnr = irqstat & GICC_IAR_INT_ID_MASK;
```

所以 `gic_chip.irq_ack = NULL`。

##### GIC 的 eoi

**EOI mode 0**——写 GICC_EOIR，Active → Inactive：

```c
static void gic_eoi_irq(struct irq_data *d)
{
    u32 hwirq = gic_irq(d);
    if (hwirq < 16) hwirq = this_cpu_read(sgi_intid);
    writel_relaxed(hwirq, gic_cpu_base(d) + GIC_CPU_EOI);
}
```

**EOI mode 1（Split mode）**——分两步：

第 1 步：`gic_handle_irq()` 入口写 GICC_EOIR 只做 Priority Drop（Active → Pending）：

```c
if (static_branch_likely(&supports_deactivate_key))
    writel_relaxed(irqstat, cpu_base + GIC_CPU_EOI);
```

第 2 步：flow handler 返回时调 `chip->irq_eoi()` → `gic_eoimode1_eoi_irq()` 写 GICC_DIR（或 STM32MP2 客制的 GIC_STM32_CPU_DEACTIVATE），做真正的 Deactivate（Pending → Inactive）。

---
#### 2.2.2 EXTI：没有状态机，只有 pending 位

EXTI 跟 GIC 完全不同——它没有"Active"这个概念，没有四状态机。它的硬件只有三个要素：

- **RPR/FPR**：上升沿/下降沿 pending 位（W1C，写 1 清除）
- **IMR**：中断屏蔽寄存器（R/W，bit=1 使能，bit=0 屏蔽）
- **输出**：一根电平线——有中断 pending 时拉高，清 pending 后拉低

所以 EXTI 的三个回调跟 GIC 的对应关系不是一一对应的：

##### EXTI 的 mask

写 IMR 的 RMW + 必须向上调 GIC 的 mask：

```c
static void stm32mp_exti_mask(struct irq_data *d)
{
    raw_spin_lock(&chip_data->rlock);
    chip_data->mask_cache &= ~stm32mp_exti_clr_bit(d, bank->imr_ofst);
    raw_spin_unlock(&chip_data->rlock);

    irq_chip_mask_parent(d);   /* 向上调 GIC 的 mask */
}
```

为什么比 GIC 多了这么多东西？
- IMR 是 R/W 寄存器，必须 RMW。`mask_cache` 记录驱动认为 IMR 的值，`raw_spinlock` 防并发覆盖
- `mask_cache` 在 suspend/resume 中用来区分"驱动使能的位"和"唤醒要保留的位"
- mask EXTI 的 IMR 只阻止新事件，但之前送出的高电平还在 GIC 那边 pending。所以必须两端都 mask

##### EXTI 的 ack：不会被调用

EXTI 不需要做 Pending → Active 的转换（它根本没有 Active 状态）。它的 pending 位在 EOI 时由 `stm32mp_exti_eoi()` 顺带清除——写 RPR/FPR 写 1 清除。

`stm32mp_exti_chip` 虽然赋值了 `.irq_ack = irq_chip_ack_parent`，但这条路径**永远不会被执行**——因为 `handle_fasteoi_irq`（EXTI 所有中断最终用的 flow handler）从不调 chip->irq_ack，只调 chip->irq_eoi。这是 fasteoi 的特性：它假设硬件自己管 ack，软件只需在结束时写 EOI。

##### EXTI 的 eoi：ack + unmask + eoi 三位一体

```c
static void stm32mp_exti_eoi(struct irq_data *d)
{
    raw_spin_lock(&chip_data->rlock);

    stm32mp_exti_write_bit(d, bank->rpr_ofst);   /* ① 清 pending（相当于 ack）*/
    stm32mp_exti_write_bit(d, bank->fpr_ofst);
    chip_data->mask_cache |= stm32mp_exti_set_bit(d, bank->imr_ofst); /* ② 恢复 IMR（相当于 unmask）*/

    raw_spin_unlock(&chip_data->rlock);

    irq_chip_eoi_parent(d);  /* ③ 向上调 GIC 的 EOI */
}
```

EXTI 的清 pending 拉低输出电平 → GIC 的电平信号消失 → GIC 状态机前进。

**EXTI 输出电平与 GIC 状态机的联动：**

```
外设触发 → EXTI 设 pending → EXTI 输出高电平
    │  GIC 状态机: Inactive ──①──► Pending
    ▼  GIC 状态机: Pending ──②──► Active（CPU 读 IAR）
    ▼  handle_fasteoi_irq → action->handler()
    ▼  chip->irq_eoi()
         ├─ stm32mp_exti_eoi: 清 RPR/FPR（输出回低）+ 恢复 IMR
         └─ irq_chip_eoi_parent → gic_eoi_irq / gic_eoimode1_eoi_irq
              → GIC 状态机: Active ──③──► Inactive
                或 Active ──④──► Pending ──④'──► Inactive
```

EXTI 没有自己的状态机，它通过输出电平高低来驱动 GIC 状态机。这就是为什么 EXTI 不需要单独做 ack——它的 pending 清除就是让 GIC 的电平信号消失，GIC 的 EOI 管理剩下的所有状态转换。
### 2.3 配置组：set_type / set_affinity

#### irq_set_type

配置中断触发类型：电平（高/低）或边沿（上升/下降）。

**GICv2 实现**（`irq-gic.c`，第 302 行）：

```c
static int gic_set_type(struct irq_data *d, unsigned int type)
{
    /* SGI (0~15) 只能是边沿上升 */
    if (gicirq < 16)
        return type != IRQ_TYPE_EDGE_RISING ? -EINVAL : 0;

    /* SPI (>= 32) 只支持电平高或边沿上升 */
    if (gicirq >= 32 && type != IRQ_TYPE_LEVEL_HIGH &&
                        type != IRQ_TYPE_EDGE_RISING)
        return -EINVAL;

    ret = gic_configure_irq(gicirq, type, base + GIC_DIST_CONFIG, NULL);
    ...
}
```

GICv2 的约束：
- SGI（0~15）：固定为边沿上升，不可更改
- PPI（16~31）：支持电平或边沿
- SPI（32+）：只支持 `IRQ_TYPE_LEVEL_HIGH` 或 `IRQ_TYPE_EDGE_RISING`

`gic_configure_irq()` 最终操作的是 `GIC_DIST_CONFIG`（GICD_ICFGR）寄存器——每个中断用 2 bit 配置触发类型。

**EXTI 实现**（`irq-stm32mp-exti.c`，第 400 行）：

```c
static int stm32mp_exti_set_type(struct irq_data *d, unsigned int type)
{
    ...
    rtsr = readl_relaxed(base + bank->rtsr_ofst);  /* 上升沿触发寄存器 */
    ftsr = readl_relaxed(base + bank->ftsr_ofst);  /* 下降沿触发寄存器 */

    err = stm32mp_exti_convert_type(d, type, &rtsr, &ftsr);
    ...
    writel_relaxed(rtsr, base + bank->rtsr_ofst);
    writel_relaxed(ftsr, base + bank->ftsr_ofst);
    ...
    /* 关键：传递给 GIC 的一律配成 LEVEL_HIGH */
    return irq_chip_set_type_parent(d, IRQ_TYPE_LEVEL_HIGH);
}
```

这里有一个重要的设计模式：**EXTI 把所有外部触发类型翻译后，向 GIC 注册的永远是 `IRQ_TYPE_LEVEL_HIGH`**。因为 EXTI 作为 GIC 的父域，它的输出是一根"有线"信号——边沿检测、电平检测都在 EXTI 内部完成了，EXTI 的输出就是一根线，有事件时拉高，EOI 时清除。对 GIC 来说，看到的就是电平高信号。

#### irq_set_affinity

设置中断分发到哪个 CPU。只对 SPI 有意义，PPI 是 per-CPU 的，SGI 是软件触发的。

GICv2 的 affinity 配置通过 `GIC_DIST_TARGET` 寄存器实现——每 8 个中断一组，每个中断占用 1 字节，每 bit 对应一个 CPU。

EXTI 的 irq_chip 中 `irq_set_affinity` 直接指向 `irq_chip_set_affinity_parent`——不做自己的 affinity 逻辑，委托给 GIC。

### 2.4 电源组：irq_set_wake

```c
static int stm32mp_exti_set_wake(struct irq_data *d, unsigned int on)
{
    struct stm32mp_exti_chip_data *chip_data = irq_data_get_irq_chip_data(d);
    u32 mask = BIT(d->hwirq % IRQS_PER_BANK);

    raw_spin_lock(&chip_data->rlock);
    if (on)
        chip_data->wake_active |= mask;
    else
        chip_data->wake_active &= ~mask;
    raw_spin_unlock(&chip_data->rlock);

    return irq_chip_set_wake_parent(d, on);
}
```

`wake_active` 记录哪些 EXTI event 被配置为唤醒源。它不直接操作硬件，而是为 suspend/resume 流程提供输入——在 §2.2.2 的 `stm32mp_chip_suspend()` 中，`mask_cache` 和 `wake_active` 配合实现"suspend 时关掉所有非唤醒中断，保留唤醒源"：

```c
val = readl_relaxed(base + bank->imr_ofst);
val &= ~mask_cache;        /* 关掉驱动原本使能的所有中断 */
val |= wake_active;         /* 但保留标记为唤醒源的位 */
writel_relaxed(val, base + bank->imr_ofst);
```

如果某个 GPIO 按键中断被设为唤醒源（`device_may_wakeup`），驱动调 `irq_set_irq_wake()` 触发这里的 `stm32mp_exti_set_wake(d, 1)`，将对应 bit 记入 `wake_active`。系统 suspend 时，这个 bit 就会在 IMR 中被保留，中断信号能通过 EXTI 穿透到 GIC 唤醒 SoC。

### 2.5 irq_chip flags

`struct irq_chip` 的 `flags` 字段是**硬件特性声明**——不同厂家的中断控制器硬件行为不同，flags 告诉 IRQ core 层"我这颗芯片的硬件是怎么工作的"，core 层据此决定处理方式。

```c
static const struct irq_chip gic_chip = {
    ...
    .flags = IRQCHIP_SET_TYPE_MASKED |
             IRQCHIP_SKIP_SET_WAKE |
             IRQCHIP_MASK_ON_SUSPEND,
};
```

| 标志位 | 作用 | GIC 用了？ | EXTI 用了？ |
|--------|------|-----------|------------|
| `IRQCHIP_SET_TYPE_MASKED` | 调 `irq_set_type()` 前先 mask 中断，防止切换触发类型时误触发 | ✅ | ❌ |
| `IRQCHIP_SKIP_SET_WAKE` | 不支持 wake，core 层不用调 `irq_set_wake`（GIC 是 CPU 内部模块，唤醒不了 SoC） | ✅ | ❌（EXTI 是唤醒源控制器，必须支持 wake） |
| `IRQCHIP_MASK_ON_SUSPEND` | suspend 前自动 mask 非唤醒中断 | ✅ | ❌ |
| `IRQCHIP_EOI_THREADED` | 线程化 handler 执行完才调 irq_eoi | ❌ | ❌ |

各标志位的硬件含义：

**`IRQCHIP_SET_TYPE_MASKED`**——芯片的触发类型配置寄存器**在改写过程中可能产生假中断**。GICD_ICFGR 每个中断用 2 bit 配置触发类型，从一种类型改到另一种时，中间过渡态 (old → new) 可能短暂匹配某个非法电平，GIC 硬件因此触发一次不该有的中断。设了这个标志后，core 层在调 `irq_set_type()` 前先调 `irq_mask()` 关掉中断，改完再 unmask。EXTI 没设这个标志——因为它改的是 RTSR（上升沿选择）和 FTSR（下降沿选择），两个寄存器互相独立，不存在中间态误触发的问题。

**`IRQCHIP_SKIP_SET_WAKE`**——芯片的硬件设计**不支持作为唤醒源**。GIC 的电源域跟 CPU 核绑在一起——CPU 休眠 = GIC 断电，GIC 本身没有办法在系统休眠时保持工作来唤醒 SoC。所以设了这个标志，core 层就不会去调 `irq_set_wake()`（调了也没用）。EXTI 没设——EXTI 是专门设计来处理唤醒事件的，它的电源域是 always-on 域（`ret_pd`），休眠时 IMR 可以保持使能，外部信号进来就能唤醒 SoC。

**`IRQCHIP_MASK_ON_SUSPEND`**——芯片**自己不管理 suspend/resume 的中断使能状态**，需要 core 层 suspend 时自动 mask 非唤醒中断、resume 时 unmask。GIC 设了——GIC 没有自己的 suspend/resume 回调，全靠 core 层帮它关。EXTI 没设——EXTI 有自己的 `stm32mp_exti_suspend()`/`stm32mp_exti_resume()` 回调（§2.4），自己控制哪个中断在休眠时保持使能。

**`IRQCHIP_EOI_THREADED`**——芯片要求**等到底半部线程执行完才能 EOI**。这是一个极少数芯片才需要的特性——大部分芯片（包括 GIC 和 EXTI）在顶半部返回后就可以 EOI，不需要等线程化的底半部。

### 2.6 层级域中的 parent 回调链

前面多次提到一个模式：EXTI 的 mask/eoi/set_type 回调在完成自己的硬件操作后，还会调 `irq_chip_mask_parent(d)`、`irq_chip_eoi_parent(d)` 这类 `irq_chip_*_parent` 辅助函数。这是层级域的关键机制——**内核在调用链上自动把中断请求从子控制器传递到父控制器**。

以 mask 为例的调用链：

```
EXTI 层: stm32mp_exti_mask(d)
    │
    ├─ 写 EXTI_IMR（清除对应 bit）
    │
    └─ irq_chip_mask_parent(d)       ← core 层提供的辅助函数
         │                              内核自动找到当前中断对应的父控制器
         ▼                              并调它的 mask 回调
GIC 层:  gic_mask_irq(parent_data)
    │
    └─ gic_poke_irq(parent_data, GIC_DIST_ENABLE_CLEAR)
         │
         ▼ 写 GICD_ICENABLER 寄存器
```

为什么两端都要 mask？mask 了 EXTI 的 IMR 只阻止了新事件进入，但之前已经送出的高电平信号还在 GIC 端 pending。GIC 看到 pending 位为 1 就会继续向 CPU 发中断。所以必须两端都关。

这个链式结构是 00-History §5.2 描述的层级域（Hierarchical Domain）在代码层面的体现。内核通过 `irq_data` 中的 `parent_data` 指针实现这个自动传递（`irq_data` 的细节见 §4.2）。

**为什么需要同时 mask EXTI 和 GIC？** 因为两个控制器之间没有硬件级的 mask 传递机制。你 mask 了 EXTI 的 IMR，GIC 那边检测到这个中断的 pending 位仍然是 1（因为 EXTI 之前已经送出了一个电平高信号给 GIC）。GIC 不知道 EXTI 内部已经 mask 了，它看到 pending=1 就会继续往 CPU 发中断。所以必须两端都 mask。

### 2.7 两个 irq_chip 实例对比：GIC vs EXTI

| 特性 | GICv2 (gic_chip) | EXTI (stm32mp_exti_chip) |
|------|-----------------|-------------------------|
| name | `"GICv2"` | `"stm32mp-exti"` |
| mask | 写 GICD_ICENABLER（WO，无缓存） | 写 EXTI_IMR + mask_cache + 调 parent |
| unmask | 写 GICD_ISENABLER | 写 EXTI_IMR + mask_cache + 调 parent |
| ack | 不实现（读 IAR 隐式 ack） | 委托 parent（`irq_chip_ack_parent`） |
| eoi | 写 GICC_EOIR 或 GICC_DIR | 清 RPR/FPR + 恢复 IMR + 调 parent |
| set_type | 配置 GICD_ICFGR（Level/Edge 限制） | 配置 RTSR/FTSR，向 GIC 报 LEVEL_HIGH |
| set_affinity | 配置 GICD_TARGET | 委托 parent |
| set_wake | 不支持（SKIP_SET_WAKE） | 记录 wake_active + 委托 parent |
| flags | SET_TYPE_MASKED \| SKIP_SET_WAKE \| MASK_ON_SUSPEND | 无特殊 flags |

---

## 3. struct irq_domain：硬件中断号到 Linux IRQ 号的映射

从 §1 的四个核心问题来看，irq_domain 回答**问题 1：识别——这个中断是哪来的**。

硬件中断控制器（GIC、EXTI）用 **hwirq**（硬件中断号）标识中断。例如 GIC 的 SPI 42 就是 hwirq 74（SPI 从 32 开始编号）。但 Linux 内核用一个全局的 **virq**（虚拟中断号，即 Linux IRQ 号）来标识中断——这是 `request_irq(irq, handler)` 的第一个参数。

**为什么需要翻译层？** 因为两块中断控制器的 hwirq 编号空间是重叠的——GIC 有 hwirq 0~1019，EXTI 也有自己的 0~95。没有 irq_domain，内核无法区分 `hwirq=42` 是 GIC 的 SPI 还是 EXTI 的 event。

### 3.1 irq_domain 结构体全字段解析

`struct irq_domain` 定义在 `include/linux/irqdomain.h`，第 150 行。它描述了一个中断控制器管理的整个"中断编号空间"，以及如何在这个空间中将硬件中断号（hwirq）翻译为 Linux 虚拟中断号（virq）。每个中断控制器实例对应一个 irq_domain（GIC 一个、EXTI 一个）。

```c
struct irq_domain {
    struct list_head            link;           /* 全局链表节点 */
    const char                  *name;          /* 域名 */
    const struct irq_domain_ops *ops;           /* 操作回调 */
    void                        *host_data;     /* 控制器私有数据 */
    unsigned int                flags;          /* 域标志 */
    unsigned int                mapcount;       /* 已映射的中断数 */
    struct mutex                mutex;
    struct irq_domain           *root;          /* 根域指针 */

    /* 可选数据 */
    struct fwnode_handle        *fwnode;        /* 固件节点 */
    enum irq_domain_bus_token   bus_token;
    struct irq_domain_chip_generic *gc;

#ifdef CONFIG_IRQ_DOMAIN_HIERARCHY
    struct irq_domain           *parent;        /* 父域指针 */
#endif

    /* 反向映射数据 */
    irq_hw_number_t             hwirq_max;      /* 最大 hwirq 号 */
    unsigned int                revmap_size;    /* 线性映射表大小 */
    struct radix_tree_root      revmap_tree;    /* 树映射基数树根 */
    struct irq_data __rcu       *revmap[];      /* 线性映射表（柔性数组） */
};
```

各字段按功能分为四组，每组逐个说明：

---
**`link`——全局链表节点**

所有 irq_domain 通过这个字段串在 `irq_domain_list` 全局链表中：

```c
/* kernel/irq/irqdomain.c */
static DEFINE_MUTEX(irq_domain_mutex);
static LIST_HEAD(irq_domain_list);
```

**为什么需要这个链表？** 当内核解析 DTS 中的 `interrupt-parent = <&exti1>` 时，它拿到的是设备节点 `np`，但并不知道对应的 irq_domain 在哪里。内核遍历 `irq_domain_list`，对每个 domain 调用 `ops->match(d, np)` 来匹配。找到了，就用它的 `ops->xlate()` 和 `ops->alloc()` 来给这个中断分配 virq。

```c
/* irq_find_host(np) — 遍历全局链表，逐个匹配 */
struct irq_domain *irq_find_host(struct device_node *node)
{
    list_for_each_entry(d, &irq_domain_list, link) {
        if (d->ops->match(d, node, DOMAIN_BUS_ANY))
            return d;
    }
    return NULL;
}
```

GIC 域和 EXTI 域都在这个链表上，各自有不同的 fwnode。`irq_find_host` 根据 DTS 节点的 phandle 找到对应域。

---
**`name`——域名**

用于日志输出和 debugfs 目录名。创建时传入的字符串，通常由驱动根据 DTS 节点名生成。

debugfs 中可以看到：

```shell
# ls /sys/kernel/debug/irq/domains/
stm32mp-exti
```

这个 name 不会直接参与中断处理，纯粹给人看的。

---
**`ops`——操作回调入口**

指向 `struct irq_domain_ops`，是 irq_domain 的**功能核心**。它包含以下几个关键回调：

| 回调 | 作用 | GIC 实现了？ | EXTI 实现了？ |
|------|------|------------|--------------|
| `match` | 根据 device_node 匹配 domain | ❌（用默认） | ✅ |
| `select` | 根据 fwspec 匹配 domain | ❌ | ✅ |
| `xlate` | 从 DTS intspec 解析出 (hwirq, type) | ❌（用 translate） | ✅（`irq_domain_xlate_twocell`） |
| `translate` | 从 fwspec 解析出 (hwirq, type) | ✅ | ❌ |
| `alloc` | 分配 virq、设置 irq_chip 和 flow handler | ✅ | ✅ |
| `map` | 旧式分配，逐级映射 | ❌ | ❌ |
| `free` | 释放 virq | ✅ | ✅ |

GIC 的 ops：

```c
static const struct irq_domain_ops gic_irq_domain_hierarchy_ops = {
    .translate  = gic_irq_domain_translate,
    .alloc      = gic_irq_domain_alloc,
    .free       = irq_domain_free_irqs_top,
};
```

EXTI 的 ops：

```c
static const struct irq_domain_ops stm32mp_exti_domain_ops = {
    .match  = stm32mp_exti_domain_match,
    .select = stm32mp_exti_domain_select,
    .alloc  = stm32mp_exti_domain_alloc,
    .free   = stm32mp_exti_domain_free,
    .xlate  = irq_domain_xlate_twocell,
};
```

`ops` 是连接 irq_domain 和具体控制器驱动的桥梁——core 层不知道 GIC 怎么翻译中断号，但它知道调 `domain->ops->translate()`。

---
**`host_data`——控制器私有数据**

core 层不碰这个字段，原样保存。驱动在创建 domain 时传入，alloc 回调中通过 `domain->host_data` 取出来用。

- GIC：`host_data = gic`（`struct gic_chip_data *`，包含 GICD/GICC 基址、中断数、domain 指针）
- EXTI：`host_data = host_data`（`struct stm32mp_exti_host_data *`，包含 base 地址、chips_data 数组、hwlock 等）

```c
/* gic_irq_domain_map 中取 host_data */
struct gic_chip_data *gic = d->host_data;

/* stm32mp_exti_domain_alloc 中取 host_data */
struct stm32mp_exti_host_data *host_data = dm->host_data;
```

---
**`flags`——域标志**

声明这个 irq_domain 的特性，目前只有两个常用标志：

| 标志 | 含义 | 谁用 |
|------|------|------|
| `IRQ_DOMAIN_FLAG_HIERARCHY` | 该域是层级域，有 parent | `irq_domain_add_hierarchy()` 自动设置 |
| `IRQ_DOMAIN_FLAG_IPI_PER_CPU` / `IPI_SINGLE` | IPI 域 | GIC 的 SGI |
| `IRQ_DOMAIN_FLAG_MSI` | MSI 中断域 | PCIe |

GIC 的 domain 创建（`irq-gic.c`，第 1221 行）：

```c
gic->domain = irq_domain_create_linear(handle, gic_irqs,
                                       &gic_irq_domain_hierarchy_ops, gic);
```

`irq_domain_create_linear` 内部不设 `HIERARCHY` 标志，因为 GIC 是根域（没有 parent）。

EXTI 的 domain 创建（`irq-stm32mp-exti.c`，第 905 行）：

```c
domain = irq_domain_add_hierarchy(parent_domain, 0,
                                   drv_data->bank_nr * IRQS_PER_BANK,
                                   np, &stm32mp_exti_domain_ops, host_data);
```

`irq_domain_add_hierarchy()` 内部自动设置 `flags |= IRQ_DOMAIN_FLAG_HIERARCHY`。

---
**`mapcount` / `mutex` / `root`——运行时管理字段**

| 字段 | 作用 |
|------|------|
| `mapcount` | 已经映射的中断数，统计用 |
| `mutex` | 保护 domain 的并发访问（分配/释放 virq 时加锁） |
| `root` | 指向**根域**的指针。在层级域中，子域的 root 指向最顶层的域（GIC），根域的 root 指向自己 |

非层级域的 root = NULL，不启用额外保护。层级域的 root 在创建子域时自动关联。

---
**`fwnode`——固件节点**

关联的固件节点（在 DTS 系统中就是 device_node）。它是 irq_domain 的"身份证"——`irq_find_host()` 遍历链表时，最终通过比较 device_node 找到对应的 domain。

GIC 的 fwnode：从 DTS 中 intc 节点的 device_node 转换而来。

```c
/* irq-gic.c 中创建 domain 时传入的 handle 就是 of_node_to_fwnode(np) */
gic->domain = irq_domain_create_linear(handle, ...);
```

EXTI 的 fwnode：同样是 `np = dev->of_node`。

```c
/* irq-stm32mp-exti.c，第 905 行 */
domain = irq_domain_add_hierarchy(parent_domain, 0, ...,
                                  np, &stm32mp_exti_domain_ops, host_data);
/* 传入的 np（device_node *）在 irq_domain_add_hierarchy 内部转成 fwnode */
```

**为什么 irq_domain 用 fwnode 而不是 device_node？** `struct fwnode_handle` 是内核的固件节点抽象层，可以代表 DT 的 device_node、ACPI 的 acpi_device、或者其他固件接口的节点。irq_domain 用 fwnode 是为了固件无关——在 DTS 系统中，`domain->fwnode` 就是通过 `of_node_to_fwnode(np)` 从 device_node 转换来的；需要取出 device_node 时调用 `to_of_node(domain->fwnode)` 反向转换。

---
**`parent`——父域指针（层级域专用）**

#ifdef CONFIG_IRQ_DOMAIN_HIERARCHY 保护的字段，只有在开启了层级域支持时才存在。指向当前域的上层域。

GIC domain：`parent = NULL`（根域，没有父域）。

EXTI domain：`parent → GIC domain`（EXTI 挂载在 GIC 下）。

这个字段是层级域链式调用的硬件基础——`irq_chip_mask_parent(d)` 内部找到 `d->domain->parent`，调它的 chip 回调。`irq_domain_alloc_irqs_parent()` 调 `domain->parent->ops->alloc()` 逐层向上分配 virq。

---
**反向映射数据——`hwirq_max` / `revmap_size` / `revmap_tree` / `revmap[]`**

这四个字段是 irq_domain 的**核心功能实现**——从 hwirq 快速找到对应的 irq_data（以及它关联的 virq）。它们之间的关系：

```
中断发生时，GIC 告诉 CPU "hwirq = 74"
    │
    ▼
irq_find_mapping(domain, hwirq)   ← 查反向映射表
    │
    ├─ 如果 hwirq < revmap_size（在线性映射范围内）
    │    → 直接取 revmap[hwirq]，O(1)
    │
    └─ 否则（hwirq 超出了线性映射范围）
         → 在 revmap_tree（基数树）中查找
```

**`revmap_size`**——线性映射表的大小。如果 domain 支持 1020 个中断（GIC），`revmap_size = 1020`。如果 hwirq 的编号是连续的（GIC 的 SPI/PPI/SGI 就是从 0~1019），那么每个 hwirq 都可以直接作为数组下标，在 `revmap[]` 中 O(1) 找到对应的 irq_data。

**`hwirq_max`**——最大 hwirq 号 + 1。它 >= revmap_size，防止超界访问。

**`revmap[]`**——柔性数组，`revmap[hwirq] = &irq_data`。线性映射的核心。如下是创建线性映射域的过程（`irq_domain_create_linear` 内部）：

```c
domain->revmap_size = size;
domain->revmap = kcalloc(size, sizeof(struct irq_data *), GFP_KERNEL);
```

当 `irq_domain_set_hwirq_and_chip()` 将一个 (hwirq, virq) 映射关系注册到 domain 时：

```c
/* irq_domain_set_mapping — 将 virq 对应的 irq_data 存入 revmap */
domain->revmap[hwirq] = irq_data;
```

**`revmap_tree`**——基数树根，用于 hwirq 范围大且稀疏的场景（如 MSI 中断）。当 hwirq >= revmap_size 时，在树中查找。查找函数 `radix_tree_lookup(&domain->revmap_tree, hwirq)`。

两种查找方式的适用场景对比：

| 方式 | 数据结构 | 复杂度 | 适用场景 |
|------|---------|--------|---------|
| Linear（`revmap[]`） | 数组 | O(1) | hwirq 连续、总数少（GIC 的 1020 个中断就是连续编号） |
| Tree（`revmap_tree`） | 基数树 | O(k)（树高固定） | hwirq 范围大且稀疏（MSI、PCIe 动态分配的中断号） |

**为什么线性映射是 O(1) 但 GIC 能用？** GIC 的中断号 0~1019 是硬件固定的——SPI 从 32 开始连续编号到 1019。每个外设的中断号都落在这个连续范围内。数组下标直接就是 hwirq，不需要查找。而 MSI 中断的编号是运行时动态分配的，可能分布在很大的区间内（0~65535），但实际使用的只有几十个——浪费几万个数组条目太奢侈，用基数树更省内存。

### 3.2 irq_domain_ops：核心回调

`struct irq_domain_ops` 定义了 domain 的**全部能力**——从匹配 domain 到解析中断参数、分配 virq、设置硬件。所有回调按调用时序分为三组：

```
阶段 1：匹配——从 DTS 的 phandle 找到对应 domain
  → match / select

阶段 2：解析——把 DTS 的 intspec 翻译成 (hwirq, type)
  → xlate / translate

阶段 3：分配——分配 virq，设置 irq_chip 和 flow handler
  → alloc / map / free / unmap
```

完整定义（`include/linux/irqdomain.h`，第 88 行）：

```c
struct irq_domain_ops {
    /* 阶段 1：匹配 */
    int (*match)(struct irq_domain *d, struct device_node *node,
                 enum irq_domain_bus_token bus_token);
    int (*select)(struct irq_domain *d, struct irq_fwspec *fwspec,
                  enum irq_domain_bus_token bus_token);

    /* 阶段 2：解析 */
    int (*map)(struct irq_domain *d, unsigned int virq, irq_hw_number_t hw);
    void (*unmap)(struct irq_domain *d, unsigned int virq);
    int (*xlate)(struct irq_domain *d, struct device_node *node,
                 const u32 *intspec, unsigned int intsize,
                 unsigned long *out_hwirq, unsigned int *out_type);

    /* 阶段 3：分配（V2 层级域接口）*/
    int (*alloc)(struct irq_domain *d, unsigned int virq,
                 unsigned int nr_irqs, void *arg);
    void (*free)(struct irq_domain *d, unsigned int virq,
                 unsigned int nr_irqs);
    int (*activate)(struct irq_domain *d, struct irq_data *irqd, bool reserve);
    void (*deactivate)(struct irq_domain *d, struct irq_data *irq_data);

    /* 现代解析接口（替代 xlate）*/
    int (*translate)(struct irq_domain *d, struct irq_fwspec *fwspec,
                     unsigned long *out_hwirq, unsigned int *out_type);
#ifdef CONFIG_GENERIC_IRQ_DEBUGFS
    void (*debug_show)(struct seq_file *m, struct irq_domain *d,
                       struct irq_data *irqd, int ind);
#endif
};
```

下面逐个回调展开。

---
#### 阶段 1：match / select——找到你的 domain

**`match`——根据 device_node 匹配 domain**

当内核解析 `interrupt-parent = <&exti1>` 时，拿到 exti1 的 device_node `np`，调 `irq_find_host(np)`。它遍历全局 `irq_domain_list`，对每个 domain 调 `ops->match(d, np)`，返回 1 的就是匹配的。

```c
/* kernel/irq/irqdomain.c */
struct irq_domain *irq_find_host(struct device_node *node)
{
    struct irq_domain *d, *found = NULL;
    ...
    list_for_each_entry(d, &irq_domain_list, link) {
        if (d->ops->match(d, node, DOMAIN_BUS_ANY)) {
            found = d;
            break;
        }
    }
    return found;
}
```

**谁实现了？** GIC **没有实现**——用内核默认的匹配逻辑（直接比较 `d->fwnode` 和 `np` 指向的 fwnode）。EXTI **实现了**，因为 STM32MP2 上 exti1 的 DTS 节点既要匹配自己 `interrupt-controller@44220000` 这个节点，又要兼容旧 DTS 的 `syscon` 形式，默认比较逻辑不够用：

```c
/* irq-stm32mp-exti.c，第 562 行 */
static int stm32mp_exti_domain_match(struct irq_domain *dm,
                                     struct device_node *node,
                                     enum irq_domain_bus_token bus_token)
{
    struct stm32mp_exti_host_data *host_data = dm->host_data;

    if (!node || (bus_token != DOMAIN_BUS_ANY && dm->bus_token != bus_token))
        return 0;

    if (!host_data->dt_has_irqs_desc)
        return (to_of_node(dm->fwnode) == node);  /* 直接比较 fwnode */

    /* 兼容 interrupts-extended 的路径：比较 parent 的 fwnode */
    if (node != host_data->dev->of_node)
        return 0;
    return (to_of_node(dm->parent->fwnode) ==
            of_irq_find_parent(host_data->dev->of_node));
}
```

**`select`——根据 fwspec 匹配 domain**

跟 `match` 类似，但它匹配的是 `irq_fwspec`（固件中断描述符）而不是 device_node。用在多 domain 共享同一个 device_node 的场景——同一个 DTS 节点可能对应多个 irq_domain（如 exti1 的主域和 wakeup 域），通过 fwspec 参数进一步区分。

EXTI 实现了它来做 GPIO mux 的选路检查：

```c
/* irq-stm32mp-exti.c，第 582 行 */
static int stm32mp_exti_domain_select(struct irq_domain *dm,
                                      struct irq_fwspec *fwspec,
                                      enum irq_domain_bus_token bus_token)
{
    ...
    /* 检查 GPIO mux 是否可用——同一 EXTI 线不能同时被两个不同 GPIO bank 占用 */
    if (!stm32mp_exti_test_gpio_mux_available(host_data, gpio_bank, hwirq))
        return 0;
    return 1;
}
```

---
#### 阶段 2：xlate / translate——从 DTS 描述翻译为硬件参数

**`xlate`——旧式 DTS 参数解析**

DTS 中写 `interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>`，内核解析器拿到三个 32 位整数的数组。`xlate` 负责把他翻译成 `(hwirq, type)`。

```c
int (*xlate)(struct irq_domain *d, struct device_node *node,
             const u32 *intspec, unsigned int intsize,
             unsigned long *out_hwirq, unsigned int *out_type);
```

参数 `intspec` 就是 DTS 中的 cell 数组，`intsize` 是 cell 个数。`xlate` 把结果存入 `out_hwirq` 和 `out_type`。

EXTI 使用了内核提供的通用实现 **`irq_domain_xlate_twocell`**（`kernel/irq/irqdomain.c`，第 998 行）：

```c
int irq_domain_xlate_twocell(struct irq_domain *d, struct device_node *node,
                              const u32 *intspec, unsigned int intsize,
                              unsigned long *out_hwirq, unsigned int *out_type)
{
    if (intsize < 2)
        return -EINVAL;
    *out_hwirq = intspec[0];          /* 第一个 cell：EXTI event 号 */
    *out_type  = intspec[1] & IRQ_TYPE_SENSE_MASK;  /* 第二个 cell：触发类型 */
    return 0;
}
```

DTS 中的 `<&exti1 42 IRQ_TYPE_EDGE_RISING>` → `xlate` 返回 `hwirq=42, type=IRQ_TYPE_EDGE_RISING`。

**`translate`——现代 fwspec 解析**

跟 `xlate` 做的事情一样（intspec → hwirq + type），但参数类型变成了 `struct irq_fwspec *`（不再是裸露的 `u32 *intspec`）。这是 V2 接口，用在层级域中。

GIC 的 **`gic_irq_domain_translate`**（`irq-gic.c`，第 1092 行）：

```c
static int gic_irq_domain_translate(struct irq_domain *d,
                                    struct irq_fwspec *fwspec,
                                    unsigned long *hwirq, unsigned int *type)
{
    if (is_of_node(fwspec->fwnode)) {
        if (fwspec->param_count < 3)
            return -EINVAL;

        switch (fwspec->param[0]) {
        case 0:  /* GIC_SPI */
            *hwirq = fwspec->param[1] + 32;   /* SPI 编号偏移 32 */
            break;
        case 1:  /* GIC_PPI */
            *hwirq = fwspec->param[1] + 16;   /* PPI 编号偏移 16 */
            break;
        default:
            return -EINVAL;
        }
        *type = fwspec->param[2] & IRQ_TYPE_SENSE_MASK;
        ...
    }
    ...
}
```

关键翻译规则——GIC 的 DTS cell 在编号上做了**抽象层**：

| DTS 写法 | cell[0] | cell[1] | hwirq 结果 |
|----------|---------|---------|-----------|
| `<GIC_SPI 42 LEVEL_HIGH>` | 0（GIC_SPI） | 42 | **42 + 32 = 74** |
| `<GIC_PPI 9 LEVEL_LOW>` | 1（GIC_PPI） | 9 | **9 + 16 = 25** |

DTS 中的 SPI 编号从 0 开始（方便阅读：SPI 42 = 第 42 个 SPI），但 GIC 硬件的中断号分配是 SGI 0~15、PPI 16~31、SPI 32+。所以 translate 要做这个偏移。同理 EXTI 不需要偏移（它的 event 号直接就是 hwirq）。

**为什么 xlate 和 translate 同时存在？** 历史原因。`xlate` 是旧接口（只有 DTS），`translate` 是 V2 接口（支持任何固件类型）。新驱动应该用 `translate`，旧驱动兼容 `xlate`。内核在 `irq_create_of_mapping()` 中优先调 `translate`，没有就调 `xlate`。

---
#### 阶段 3：alloc / map / free——分配和释放 virq

**`map`——旧式单中断分配**

```c
int (*map)(struct irq_domain *d, unsigned int virq, irq_hw_number_t hw);
```

在传统的非层级域驱动中使用。它直接为单个 hwirq 分配 virq，设置 irq_chip 和 flow handler。每分配一个中断调一次。

**`alloc`——现代批量分配（V2 接口）**

```c
int (*alloc)(struct irq_domain *d, unsigned int virq,
             unsigned int nr_irqs, void *arg);
```

`alloc` 是层级域时代的分配方式。它**一次分配多个中断**（nr_irqs），并且会调 `irq_domain_alloc_irqs_parent()` 逐层向上传递分配请求。

GIC 的 `gic_irq_domain_alloc`（`irq-gic.c`，第 1147 行）：

```c
static int gic_irq_domain_alloc(struct irq_domain *domain, unsigned int virq,
                                unsigned int nr_irqs, void *arg)
{
    int i, ret;
    irq_hw_number_t hwirq;
    unsigned int type = IRQ_TYPE_NONE;
    struct irq_fwspec *fwspec = arg;

    ret = gic_irq_domain_translate(domain, fwspec, &hwirq, &type);
    if (ret)
        return ret;

    for (i = 0; i < nr_irqs; i++) {
        ret = gic_irq_domain_map(domain, virq + i, hwirq + i);
        if (ret)
            return ret;
    }
    return 0;
}
```

流程：先调 `translate` 把 fwspec 翻译成 (hwirq, type)，然后逐个调 `gic_irq_domain_map` 为每个 virq 设置 irq_chip 和 flow handler。

EXTI 的 `stm32mp_exti_domain_alloc`（第 677 行）做了更多事——它不仅要设 chip，还要检查安全域预留、分配 GPIO mux、构造 parent fwspec 向上传递：

```c
static int stm32mp_exti_domain_alloc(struct irq_domain *dm, unsigned int virq,
                                     unsigned int nr_irqs, void *data)
{
    struct irq_fwspec *fwspec = data;
    irq_hw_number_t hwirq = fwspec->param[0];

    /* ① 校验范围 */
    if (hwirq >= host_data->drv_data->bank_nr * IRQS_PER_BANK)
        return -EINVAL;

    /* ② 检查安全域预留（Secure 世界独占的 event 不能分配给 Normal 世界） */
    if (chip_data->event_reserved & BIT(hwirq % IRQS_PER_BANK))
        return -EPERM;

    /* ③ 分配 GPIO mux 选路 */
    stm32mp_exti_gpio_bank_alloc(dm, fwspec);

    /* ④ 设置 chip：根据 TRG 寄存器选 stm32mp_exti_chip 或 direct 变体 */
    chip = (event_trg & BIT(hwirq % IRQS_PER_BANK)) ?
           &stm32mp_exti_chip : &stm32mp_exti_chip_direct;
    irq_domain_set_hwirq_and_chip(dm, virq, hwirq, chip, chip_data);

    /* ⑤ 构造 parent fwspec，向上调 GIC 的 alloc */
    p_fwspec.param[0] = GIC_SPI;
    p_fwspec.param[1] = desc_irq;
    p_fwspec.param[2] = IRQ_TYPE_LEVEL_HIGH;
    return irq_domain_alloc_irqs_parent(dm, virq, 1, &p_fwspec);
}
```

**`free`——释放 virq**

反向操作。GIC 用内核通用 `irq_domain_free_irqs_top`，EXTI 额外调 `stm32mp_exti_gpio_bank_free` 释放 GPIO mux 选路：

```c
static void stm32mp_exti_domain_free(struct irq_domain *dm, unsigned int virq,
                                     unsigned int nr_irqs)
{
    struct irq_data *irq_data = irq_domain_get_irq_data(dm, virq);
    stm32mp_exti_gpio_bank_free(dm, irq_data->hwirq);
    irq_domain_free_irqs_common(dm, virq, nr_irqs);
}
```

---
#### GIC 和 EXTI 的 ops 实现对比总结

| 回调 | GIC | EXTI | 作用 |
|------|-----|------|------|
| `match` | ❌（默认 fwnode 比较）| ✅ `stm32mp_exti_domain_match` | 根据 device_node 找 domain |
| `select` | ❌ | ✅ `stm32mp_exti_domain_select` | 根据 fwspec 精确选 domain |
| `xlate` | ❌ | ✅ `irq_domain_xlate_twocell` | 解析 DTS intspec → (hwirq, type) |
| `translate` | ✅ `gic_irq_domain_translate` | ❌ | 从 fwspec 解析 (hwirq, type) |
| `alloc` | ✅ `gic_irq_domain_alloc` | ✅ `stm32mp_exti_domain_alloc` | 分配 virq + 设 chip + 向上传递 |
| `free` | ✅ `irq_domain_free_irqs_top` | ✅ `stm32mp_exti_domain_free` | 释放 virq + 资源清理 |
| `map` | ❌ | ❌ | 旧式单中断分配（已弃用）|

### 3.3 两种反向映射方式

irq_domain 翻译 hwirq → virq 后，需要把结果存起来供下次快速查找。内核提供两种缓存方式：

| 方式 | 数据结构 | 查找复杂度 | 适用场景 | 谁在用 |
|------|---------|-----------|---------|--------|
| **Linear** | 数组 `revmap[hwirq]` | O(1) | hwirq 连续、总数少（<1024） | GIC（~1020 个中断，连续编号） |
| **Tree** | 基数树 `revmap_tree` | O(k)（树高固定） | hwirq 范围大且稀疏 | MSI、PCIe 中断（动态编号） |
| **NoMap** | 无 | N/A | 不需要反向查找 | MSI 域（virq 通过 MSI 消息直达 CPU） |

**线性映射（Linear）** 是最常用的方式。创建时指定 `size`，内核分配 `sizeof(struct irq_data *) * size` 的数组做 `revmap[]`。反向映射就是直接的数组下标访问：

```c
/* kernel/irq/irqdomain.c */
static struct irq_data *linear_revmap(struct irq_domain *domain,
                                       irq_hw_number_t hwirq)
{
    return rcu_dereference(domain->revmap[hwirq]);
}
```

GIC 用这种方式：

```c
/* irq-gic.c 第 1221 行 */
gic->domain = irq_domain_create_linear(handle, gic_irqs,
                                       &gic_irq_domain_hierarchy_ops, gic);
```

`gic_irqs` 通过 `GIC_DIST_CTR` 寄存器读出（IT_LINES_NUMBER 字段），最大 1020。

为什么 GIC 能用线性映射？因为 GIC 的 hwirq 0~1019 是硬件固定且连续的——每个 SPI 都对应一个唯一编号，数组下标直接就是 hwirq。不需要复杂查找。

**树映射（Tree）** 用在 hwirq 稀疏或数量很大的场景。例如 MSI 中断在 0~65535 范围内随机分配，只用几十个——数组浪费太大，用基数树更省内存。查找时：

```c
/* kernel/irq/irqdomain.c */
radix_tree_lookup(&domain->revmap_tree, hwirq);
```

创建方式是通过 `irq_domain_create_tree()`。

旧内核还有一种 **Legacy** 方式（预分配固定数量的 virq，hwirq 和 virq 一一对应），但 v6.6 中已被移除，所有驱动都改用 Linear 或 Tree。

**NoMap** 是第三种形式——它不做反向映射缓存。使用 `irq_domain_create_nomap()` 创建，`irq_find_mapping()` 永远返回错误。它不是为了查找 hwirq → virq，而是给**不需要反向查找**的场景用的——典型例子是 MSI 域，中断号通过 MSI 消息直接送达 CPU，不需要通过 domain 查表。

### 3.4 STM32MP257 上的 domain 拓扑

在 STM32MP257 上，中断控制器之间形成了三层层级域：

```
┌──────────────────────────────────────────────┐
│     GIC domain (根域)                          │
│  compatible = "arm,cortex-a7-gic"             │
│  hwirq: 0~1019                               │
│                                              │
│  ● SPI 32~1019: 外设直连（USART/SDMMC/I2C等） │
│  ● PPI 16~31:   每个 CPU 私有（Timer、PMU等） │
│  ● SGI 0~15:    核间中断（IPI）                │
│                                              │
│  映射函数: handle_fasteoi_irq (SPI)           │
│  irq_chip: gic_chip / gic_chip_mode1         │
└────────────────────┬─────────────────────────┘
                     │ parent
                     ▼
┌──────────────────────────────────────────────┐
│     EXTI domain (子域，挂 GIC 下)              │
│  compatible = "st,stm32mp1-exti"             │
│  hwirq: 0~95 (3 banks × 32)                  │
│                                              │
│  ● event 0~95: 外部中断/唤醒事件                │
│  ● GPIO mux:  16 个 GPIO 中断共享 EXTI 线     │
│                                              │
│  映射函数: handle_fasteoi_irq（由 GIC 设置） │
│  irq_chip: stm32mp_exti_chip                 │
│  （或 direct，不含 event_trg 的线路）           │
└────────────────────┬─────────────────────────┘
                     │ parent（非 ADIN）
                     ▼
┌──────────────────────────────────────────────┐
│     GPIO mux domain（虚拟选路层）               │
│  ⚠ 这不是独立的 irq_domain                     │
│  是 EXTI 驱动内部通过 gpio_mux_used 管理的      │
│  选路逻辑：哪个 GPIO bank 的哪个 pin            │
│  连接到了哪个 EXTI event 线                     │
└──────────────────────────────────────────────┘
```

#### 层级域创建流程

GIC domain 是在 `gic_init_bases()` 中创建的——根域（无 parent）。

EXTI domain 是在 `stm32mp_exti_probe()` 中创建的——通过 `irq_domain_add_hierarchy()` 挂到 GIC domain 下：

```c
/* irq-stm32mp-exti.c，第 905 行 */
parent_domain = irq_find_host(of_irq_find_parent(np));

domain = irq_domain_add_hierarchy(parent_domain, 0,
                                  drv_data->bank_nr * IRQS_PER_BANK,
                                  np, &stm32mp_exti_domain_ops,
                                  host_data);
```

`irq_domain_add_hierarchy()` 内部做了三件事：
1. 创建新的 irq_domain，设置 `domain->parent = parent_domain`
2. 设置 `domain->flags |= IRQ_DOMAIN_FLAG_HIERARCHY`
3. 将 parent_domain 和当前 domain 关联

### 3.5 层级域的关键机制

#### alloc 的自底向上传递

当设备驱动（或内核 core）调用 `irq_domain_alloc_irqs()` 时，分配过程是从底层域向根域传递的：

```
irq_create_of_mapping() — 解析 DTS 的 interrupts 属性
    │
    ▼
irq_find_host() — 找到对应的 irq_domain（匹配 fwnode）
    │
    ▼ 调 domain->ops->alloc()
    │
EXTI 的 stm32mp_exti_domain_alloc()
    │
    ├─ 设置 irq_chip = &stm32mp_exti_chip（或 direct）
    ├─ 设置 irq_domain_set_hwirq_and_chip()
    │
    ├─ 构造 parent_fwspec（GIC_SPI + desc_irq + LEVEL_HIGH）
    │
    └─ irq_domain_alloc_irqs_parent()
         │
         ▼ 调 parent->ops->alloc()
         │
         GIC 的 gic_irq_domain_alloc()
              │
              ├─ gic_irq_domain_translate() — 解析 fwspec
              ├─ gic_irq_domain_map() — 设置 irq_chip = gic_chip
              │   └─ irq_domain_set_info() — 设置 flow handler
              └─ 完成
```

关键点：**每一层只做属于自己的事情**。EXTI 分配时设置 EXTI 的 chip 和 hwirq，然后构造 GIC 能理解的 fwspec（把 EXTI 事件号翻译成 GIC SPI 号），传递给 parent 继续分配。最终的 virq 被两层 chip 共享——`irq_data->chip` 是 EXTI 的 chip，`irq_data->parent_data->chip` 是 GIC 的 chip。

#### xlate / translate 的链式传递

当 DTS 中写 `interrupts-extended = <&exti1 42 IRQ_TYPE_EDGE_RISING>` 时：

1. 内核解析器根据 phandle `&exti1` 找到 EXTI 的 irq_domain
2. 调 EXTI domain 的 `xlate`（`irq_domain_xlate_twocell`）→ `hwirq=42, type=EDGE_RISING`
3. 调 EXTI domain 的 `alloc` → `stm32mp_exti_domain_alloc()`
4. `alloc` 内部：设置 EXTI chip，然后构造 `p_fwspec = {GIC_SPI, desc_irq, LEVEL_HIGH}`
5. 调 `irq_domain_alloc_irqs_parent()` → GIC 的 `translate` → 得到 GIC 的 hwirq
6. GIC 分配 virq，设置 `gic_chip` + `handle_fasteoi_irq`
7. 返回最终的 virq——下层是 EXTI 的 chip，上层（parent_data）是 GIC 的 chip

#### 链式（Chained）vs 层级（Hierarchy）：两种中断控制器架构对比

链式和层级是 Linux 中断子系统中"下级中断控制器"的两种实现架构。它们在教材中有详细图解，核心区别是**多个下级中断信号如何映射到 GIC 的中断线**：

| 特性 | 链式（Chained） | 层级（Hierarchy） |
|------|----------------|-----------------|
| **拓扑** | 多个下级中断 → 共用 1 条 GIC SPI | 每个下级中断 → 各占 1 条 GIC SPI |
| **映射方式** | 多对一 | 一对一 |
| **irq_desc 数量** | 多个（每层独立） | 一个（层级共用） |
| **入口函数** | 两级 flow handler：GIC handler → 下级 handler → action | 一级 flow handler（GIC 的），下级 chip 回调嵌入 parent_data 链 |
| **中断源细分** | 下级驱动读寄存器细分（需要软件分辨） | GIC 硬件直接分辨（不需要软件参与） |
| **数据结构** | 两个 irq_desc 各自独立，无 parent_data 链 | 一个 irq_desc，irq_data 通过 parent_data 形成链表 |
| **适用场景** | 传统 GPIO 控制器（如 i.MX6ULL 的 GPIO） | 现代 SoC 的中断控制器（如 EXTI、MSI 控制器） |

**链式架构**的处理流程（以 GPIO 作为下级控制器为例）：

```
外设触发中断
    │
    ▼
GIC 检测到 SPI 33 → hwirq=33 → virq=17（GIC domain 映射）
    │
    ▼
irq_desc[17].handle_irq（handleB——由 GPIO 驱动设置）
    │
    ├─ mask/ack: 调 irq_dataA->irq_chip（GIC 的回调）
    ├─ 读取 GPIO 寄存器，确定是 GPIO 2 号引脚
    ├─ GPIO domain 翻译：hwirq=2 → virq=102
    ├─ 调 irq_desc[102].handle_irq（handleC——另一个 handler）
    │     ├─ mask/ack: 调 irq_dataB->irq_chip（GPIO 的回调）
    │     ├─ 调 action->handler()（用户注册的 handler）
    │     └─ unmask: 调 irq_dataB->irq_chip
    └─ unmask: 调 irq_dataA->irq_chip
```

**层级架构**的处理流程（以 STM32MP2 EXTI 为例）：

```
外设触发中断
    │
    ▼
GIC 检测到 SPI 153 → hwirq=153 → virq=N（GIC domain 映射）
    │
    ▼
irq_desc[N].handle_irq（handle_fasteoi_irq——由 GIC 设置）
    │
    ├─ mask/ack:
    │    ├─ 调 irq_data->chip->irq_mask = stm32mp_exti_mask
    │    │    └─ 写 EXTI_IMR + irq_chip_mask_parent()
    │    │         └─ 调 parent_data->chip->irq_mask = gic_mask_irq
    │    └─（ack 通过 parent 链转发）
    ├─ 调 action->handler()（用户注册的 handler）
    └─ unmask/eoi:
         ├─ 调 irq_data->chip->irq_eoi = stm32mp_exti_eoi
         │    ├─ 清 RPR/FPR + 恢复 IMR
         │    └─ irq_chip_eoi_parent() → gic_eoi_irq
```

**两种架构的历史定位**（与 00-History §5 呼应）：

| | 链式 | 层级 |
|--|------|------|
| **引入时间** | v2.6.x，GPIO 控制器时代 | v3.14+，MSI/VFIO 时代 |
| **典型代表** | i.MX6ULL GPIO → `gpio-mxc.c` | STM32MP EXTI → `irq-stm32mp-exti.c` |
| **设计思想** | "软件分辨"——下级驱动读寄存器判断哪个子中断触发 | "硬件分辨"——每个子中断独占一条中断线 |
| **扩展性** | 子中断数量受限于下级驱动 | 支持随意扩展，parent_data 链无长度限制 |
| **Debug 难度** | 两级 irq_desc，`/proc/interrupts` 看到两个中断号 | 单级 irq_desc，`/proc/interrupts` 只看到一个 |

STM32MP2 的 EXTI 采用层级架构（通过 `irq_domain_add_hierarchy()` 注册）。各 GPIO bank 的中断分别对应不同的 EXTI event 和 GIC SPI，硬件上已经分辨好了，不需要在软件中遍历 GPIO 寄存器来细分中断源。

---

## 4. struct irq_desc：中断描述符与 Flow Control

从 §1 的四个核心问题来看，irq_desc 主要回答**问题 4：分发——谁处理这个中断**。同时 irq_desc 通过嵌入的 irq_data 连接了 irq_chip（问题 2：控制）和 irq_domain（问题 1：识别）。

`irq_desc` 是整个中断子系统的**核心枢纽**——它把硬件操作（irq_chip）、编号映射（irq_domain）、设备驱动 handler（irqaction）、流程控制（flow handler）全部串联在一起。

### 4.1 irq_desc 结构体全字段解析

`struct irq_desc` 定义在 `include/linux/irqdesc.h`，第 55 行：

```c
struct irq_desc {
    struct irq_common_data  irq_common_data;  /* 共享数据（affinity、msi_desc） */
    struct irq_data         irq_data;          /* 核心连接器（chip、domain、hwirq） */
    unsigned int __percpu   *kstat_irqs;       /* 每个 CPU 的中断计数 */
    irq_flow_handler_t      handle_irq;        /* flow handler 函数指针 */
    struct irqaction        *action;           /* irqaction 链表头 */
    unsigned int            status_use_accessors;
    unsigned int            core_internal_state__do_not_mess_with_it;
    unsigned int            depth;             /* disable 嵌套深度 */
    unsigned int            wake_depth;        /* set_wake 嵌套深度 */
    unsigned int            tot_count;         /* 总中断次数（统计用） */
    unsigned int            irq_count;         /* 防止假中断 */
    unsigned long           last_unhandled;
    unsigned int            irqs_unhandled;
    atomic_t                threads_handled;
    int                     threads_handled_last;
    raw_spinlock_t          lock;              /* 中断描述符自旋锁 */
    struct cpumask          *percpu_enabled;
    /* SMP 相关 */
#ifdef CONFIG_SMP
    const struct cpumask    *affinity_hint;
    struct irq_affinity_notify *affinity_notify;
    cpumask_var_t           pending_mask;
#endif
    unsigned long           threads_oneshot;   /* 哪些 action 触发了 ONESHOT */
    atomic_t                threads_active;    /* 当前正在运行的 thread 数 */
    wait_queue_head_t       wait_for_threads;  /* sync_irq 等待队列 */
    struct mutex            request_mutex;     /* request/free 保护锁 */
    int                     parent_irq;        /* 级联时的父中断号 */
    const char              *name;             /* flow handler 名（/proc/interrupts） */
} ____cacheline_internodealigned_in_smp;
```

各核心字段的职责：

| 字段 | 作用 | 谁使用 |
|------|------|--------|
| `irq_data` | **核心连接器**——指向 chip、domain，持有 hwirq 和 virq | 每个地方都用到 |
| `handle_irq` | flow handler 指针——决定中断处理的时序逻辑 | core 层的入口 |
| `action` | irqaction 链表——设备驱动的 handler 列表 | 中断发生时遍历执行 |
| `lock` | 自旋锁——保护这个 desc 的所有字段 | 所有路径 |
| `depth` | `disable_irq()` 嵌套深度——0 表示使能 | 用户调 `enable_irq/disable_irq` |
| `kstat_irqs` | 每个 CPU 的中断计数 | `/proc/interrupts` 显示 |
| `threads_oneshot` | 位图——哪些 action 触发了 IRQF_ONESHOT | flow handler 判断是否保持 mask |
| `threads_active` | 正在运行的线程化 handler 数 | `synchronize_irq()` 等待 |

### 4.2 irq_data：整个子系统中最关键的连接器

为什么 `irq_data` 要从 `irq_desc` 中独立出来？原因只有一条：**层级域**。

在层级域中，一个 virq 同时绑定了多个控制器的信息——EXTI 的 chip 和 GIC 的 chip。如果 `irq_data` 嵌在 `irq_desc` 里，就只能存一份 chip 和 domain 指针。通过 `parent_data` 指针链，每个 virq 可以带一整串"嵌套的 irq_data"：

```
irq_desc
  └── irq_data          ← EXTI 层（上层）
        ├── irq  = virq
        ├── hwirq = EXTI event 号（如 42）
        ├── chip → &stm32mp_exti_chip
        ├── domain → EXTI irq_domain
        └── parent_data
              └── irq_data    ← GIC 层（下层）
                    ├── hwirq = GIC SPI 号（如 desc_irq）
                    ├── chip → &gic_chip
                    ├── domain → GIC irq_domain
                    └── parent_data = NULL（根域）
```

**当你调 `irq_chip_mask_parent(d)` 时，内部就是 `d->parent_data->chip->irq_mask(d->parent_data)`。**

```c
/* include/linux/irq.h，第 179 行 */
struct irq_data {
    u32                 mask;           /* 预计算位掩码（加速寄存器访问） */
    unsigned int        irq;            /* Linux IRQ 号（virq） */
    unsigned long       hwirq;          /* 硬件中断号 */
    struct irq_common_data *common;     /* 指向共享数据（affinity 等） */
    struct irq_chip     *chip;          /* irq_chip 指针（哪个控制器的回调） */
    struct irq_domain   *domain;        /* 所属 irq_domain */
#ifdef CONFIG_IRQ_DOMAIN_HIERARCHY
    struct irq_data     *parent_data;   /* 父层的 irq_data */
#endif
    void                *chip_data;     /* 芯片私有数据（如 stm32mp_exti_chip_data *） */
};
```

各字段详解：

**`irq` + `hwirq`**——两套编号的桥梁。`irq` 是全局唯一的 Linux IRQ 号（`request_irq` 的第一个参数），`hwirq` 是这个中断控制器内部的中断号。翻译公式：

```
virq = domain->ops->map(domain, hwirq)  返回的 virq
hwirq = domain->ops->xlate(...)          从 DTS 解析出来的
```

**`chip`**——指向这个层级使用的 irq_chip。对于 EXTI 层，`chip = &stm32mp_exti_chip`；对于 GIC 层，`chip = &gic_chip`。

**`domain`**——指向创建这个 irq_data 的 irq_domain。通过 `irq_data->domain->parent` 可以找到父域。

**`parent_data`**——层级域的关键指针。`parent_data->chip` 可以调父层的回调。所有 `irq_chip_*_parent()` 系列辅助函数的核心逻辑就是：

```c
/* kernel/irq/chip.c */
void irq_chip_mask_parent(struct irq_data *d)
{
    d->parent_data->chip->irq_mask(d->parent_data);
}
```

**`chip_data`**——厂商私有数据指针。GIC 的 `chip_data` 指向 `struct gic_chip_data *`，EXTI 的 `chip_data` 指向 `struct stm32mp_exti_chip_data *`。

#### irq_common_data 与 irq_data 的关系

`irq_common_data` 是所有层级共享的数据（被各级 irq_data 共用），`irq_data` 是每层独立的（不同层级有不同的 chip、domain、parent_data）：

```
irq_desc
  ├── irq_common_data ← 所有层级的 irq_data.common 都指向它
  │     ├── affinity     （SMP 亲和性掩码）
  │     ├── handler_data （可选的私有数据）
  │     ├── msi_desc     （MSI 描述符，非 MSI 中断为 NULL）
  │     └── state_use_accessors （IRQD_* 状态位）
  └── irq_data（第 1 层——EXTI）
        ├── common ──────────────→ irq_common_data
        ├── chip → &stm32mp_exti_chip
        └── parent_data
              └── irq_data（第 2 层——GIC）
                    ├── common ──→ 同一个 irq_common_data
                    ├── chip → &gic_chip
                    └── parent_data = NULL
```

### 4.3 五种 flow handler 深度剖析

flow handler 是挂在 `irq_desc->handle_irq` 上的函数指针，决定了 **mask/ack/eoi/unmask 的时序**。内核提供了五种标准实现，分别对应不同的中断控制器硬件特性。

分类依据在于两个问题：

**问题 1：这个控制器是否在硬件上自动管理流程？**
- 如果硬件自动管理（如 GIC——写 EOI 就走完整个流程），用 `handle_fasteoi_irq`
- 如果硬件不做管理（如传统中断控制器——必须软件 mask 和 ack），用 `handle_level_irq` 或 `handle_edge_irq`

**问题 2：中断是电平触发还是边沿触发？**
- 电平：信号持续存在，必须保持 mask 直到 handler 完成，否则死循环 → `handle_level_irq`
- 边沿：信号是脉冲，ack 后马上消失，但可能丢边沿 → `handle_edge_irq`

下面逐个展开。

---
#### handle_level_irq：电平触发的"先关、处理、再开"

**解决什么问题？** 电平中断的外设信号保持为高，直到 CPU 处理完清掉。但这个"清掉"发生在 handler 中。如果不先 mask，CPU 读 IAR（ack）后状态从 Pending 变 Active，但外设电平还是高——GIC 立即重新触发，handler 再次被调用，形成死循环。

**思路**：入口先 mask（关阀门），等 handler 清掉中断条件后再 unmask（开阀门）。

```c
void handle_level_irq(struct irq_desc *desc)
{
    raw_spin_lock(&desc->lock);
    mask_ack_irq(desc);             /* ① 立即 mask + ack */

    if (!irq_may_run(desc))
        goto out_unlock;

    desc->istate &= ~(IRQS_REPLAY | IRQS_WAITING);

    if (unlikely(!desc->action || irqd_irq_disabled(&desc->irq_data))) {
        desc->istate |= IRQS_PENDING;
        goto out_unlock;
    }

    kstat_incr_irqs_this_cpu(desc);
    handle_irq_event(desc);         /* ④ 执行 action 链表的 handler */

    cond_unmask_irq(desc);          /* ⑤ handler 完成后 unmask */

out_unlock:
    raw_spin_unlock(&desc->lock);
}
```

逐行拆解：

| 行 | 做了什么 | 跟 GIC 状态机的关系 |
|----|---------|-------------------|
| ① `mask_ack_irq` | mask（关入口阀门）+ ack（Pending → Active） | 阀门关了，外设信号进不来；状态机停在 Active |
| ② `irq_may_run` | 检查其他核是否正在处理（`IRQD_IRQ_INPROGRESS`） | 防并发 |
| ③ `!desc->action` | 没有 handler？保持 mask，标记 pending 等以后 | 状态机卡在 Active |
| ④ `handle_irq_event` | 调 `action->handler()`——**这里清掉中断条件** | handler 写外设寄存器，让它电平变低 |
| ⑤ `cond_unmask_irq` | unmask（开阀门），允许下一轮中断 | 阀门开了，电平如果已变低就没问题 |

时序图：

```
时间   ─────────────────────────────────────────────────►
       │                                                  │
   中断信号：    ───── 持续为高（设备保持电平）──────────────
       │                                                  │
   阀门(mask)：   ── 关 ──── handler 执行 ──── 开 ────
       │                                                  │
   handler：                       ── 清中断条件 ──
       │
   ↑                               ↑                    ↑
   ① mask_ack_irq()               ④ handle_irq_event()  ⑤ cond_unmask_irq()
      (Pending→Active)             (action->handler())    (开阀门)
```

---
#### handle_edge_irq：边沿触发的"ack 后检测 pending 循环"

**解决什么问题？** 边沿信号的脉冲只持续一瞬间，ack 后信号就消失了。但如果边沿到来时 CPU 正在处理上一个中断，这个边沿可能被漏掉。

**思路**：先 ack 让出控制权，然后用 do-while 循环检查是否有新的边沿（`IRQS_PENDING`）。有就 unmask 重来一轮。

```c
void handle_edge_irq(struct irq_desc *desc)
{
    raw_spin_lock(&desc->lock);

    desc->istate &= ~(IRQS_REPLAY | IRQS_WAITING);

    if (!irq_may_run(desc)) {
        desc->istate |= IRQS_PENDING;
        mask_ack_irq(desc);
        goto out_unlock;
    }

    if (irqd_irq_disabled(&desc->irq_data) || !desc->action) {
        desc->istate |= IRQS_PENDING;
        mask_ack_irq(desc);
        goto out_unlock;
    }

    kstat_incr_irqs_this_cpu(desc);

    desc->irq_data.chip->irq_ack(&desc->irq_data);   /* ① 单独 ack */

    do {
        if (unlikely(!desc->action)) {
            mask_irq(desc);
            goto out_unlock;
        }

        if (unlikely(desc->istate & IRQS_PENDING)) {   /* ② 新的边沿？ */
            if (!irqd_irq_disabled(&desc->irq_data) &&
                irqd_irq_masked(&desc->irq_data))
                unmask_irq(desc);                       /* ③ 如果 masked，unmask */
        }

        handle_irq_event(desc);                         /* ④ 执行 handler */

    } while ((desc->istate & IRQS_PENDING) &&           /* ⑤ 还有 pending？继续 */
             !irqd_irq_disabled(&desc->irq_data));

out_unlock:
    raw_spin_unlock(&desc->lock);
}
```

关键设计在 do-while：

| 步骤 | 操作 | 说明 |
|------|------|------|
| ① `irq_ack()` | 单独一次 ack（Pending → Active） | 跟 `mask_ack_irq` 中那次重复 ack？不是——第二次 ack 是因为之前的 `mask_ack_irq` 只"预处理"了不可运行的情况，现在真的要运行了，需要确保状态准确 |
| ② 检测 `IRQS_PENDING` | 处理期间又来了新边沿？ | GIC 状态从 Active → Active&Pending，内核标记 pending |
| ③ `unmask_irq()` | 开阀门，允许新边沿触发 | 新边沿进来触发 ①，变成 Pending→Active |
| ④ `handle_irq_event` | 调 handler | 第二次调 handler |
| ⑤ while 条件 | pending 还在？再循环 | 只要还有 pending，说明边沿来了还没处理完，继续 |

边沿 vs 电平的对比表：

| 对比项 | level_irq | edge_irq | 原因 |
|--------|----------|---------|------|
| mask 时机 | 入口立即 mask，直到 handler 返回 | ack 前 mask，循环中会 unmask | level 关一次够，edge 需要循环 |
| ack 位置 | 在 `mask_ack_irq` 统一处理 | `mask_ack_irq` + 循环内再 `irq_ack` 一次 | edge 需要让出 pending 位给新边沿 |
| 新中断处理 | 不理（保持 mask） | do-while 检查 `IRQS_PENDING` | level 的电平还在，查了也没用 |
| unmask 时机 | handler 返回后 cond_unmask | pending 时 unmask 重入 | level 一次搞定，edge 可能多次 |

---
#### handle_fasteoi_irq：GIC 的"只写 EOI，其他不管"

**解决什么问题？** GIC 这类硬件自动管理的设备——读 IAR 就是 ack（Pending → Active），写 EOI 就是结束（Active → Inactive 或 Pending）。软件不需要 mask/ack。所以 flow handler 可以非常轻。

```c
void handle_fasteoi_irq(struct irq_desc *desc)
{
    struct irq_chip *chip = desc->irq_data.chip;

    raw_spin_lock(&desc->lock);

    if (!irq_may_run(desc)) {
        if (irqd_needs_resend_when_in_progress(&desc->irq_data))
            desc->istate |= IRQS_PENDING;
        goto out;
    }

    desc->istate &= ~(IRQS_REPLAY | IRQS_WAITING);

    if (unlikely(!desc->action || irqd_irq_disabled(&desc->irq_data))) {
        desc->istate |= IRQS_PENDING;
        mask_irq(desc);
        goto out;
    }

    kstat_incr_irqs_this_cpu(desc);
    if (desc->istate & IRQS_ONESHOT)
        mask_irq(desc);

    handle_irq_event(desc);

    cond_unmask_eoi_irq(desc, chip);

    if (unlikely(desc->istate & IRQS_PENDING))
        check_irq_resend(desc, false);

    raw_spin_unlock(&desc->lock);
    return;
out:
    if (!(chip->flags & IRQCHIP_EOI_IF_HANDLED))
        chip->irq_eoi(&desc->irq_data);
    raw_spin_unlock(&desc->lock);
}
```

**最重要的特征：不主动调 mask、不主动调 ack。**

只有两种情况下 mask：

1. **无 action 或 disabled**：中断没人要了，直接 mask + EOI 关掉它
2. **`IRQF_ONESHOT`**：线程化中断的顶半部返回后、底半部完成前保持 mask（防止中断风暴——详见 §5.3）

其他所有路径都不 mask，因为 GIC 硬件通过 Active 状态自己防止重复触发——状态机本身就是锁。

`cond_unmask_eoi_irq()` 的逻辑：如果非 ONESHOT，直接 EOI；如果 ONESHOT 且线程不在运行，EOI + unmask；如果线程还在跑，只 EOI 不 unmask（等底半部完成后再 unmask）。

---
#### handle_simple_irq：最轻量的"裸调 handler"

适用于**不需要任何 flow control** 的中断控制器（如 SoC 内部的中断聚合器）。它只做一件事：检查可运行性 → 调 handler → 返回。不 mask、不 ack、不 eoi。

```c
void handle_simple_irq(struct irq_desc *desc)
{
    raw_spin_lock(&desc->lock);
    if (!irq_may_run(desc))
        goto out_unlock;
    desc->istate &= ~(IRQS_REPLAY | IRQS_WAITING);
    if (unlikely(!desc->action || irqd_irq_disabled(&desc->irq_data))) {
        desc->istate |= IRQS_PENDING;
        goto out_unlock;
    }
    kstat_incr_irqs_this_cpu(desc);
    handle_irq_event(desc);
out_unlock:
    raw_spin_unlock(&desc->lock);
}
```

这类控制器通常有自己的 mask/ack 机制，由父控制器在链式调用中完成。

---
#### handle_percpu_devid_irq：per-CPU 中断的"不锁"

GIC 的 PPI（Private Peripheral Interrupt，hwirq 16~31）是每个核独享的——同一个中断号在不同核之间不冲突。所以不加锁、不用 kstat 全局统计、不 mask。

```c
void handle_percpu_devid_irq(struct irq_desc *desc)
{
    struct irq_chip *chip = irq_desc_get_chip(desc);
    struct irqaction *action = desc->action;
    unsigned int irq = irq_desc_get_irq(desc);
    irqreturn_t res;

    __kstat_incr_irqs_this_cpu(desc);

    if (chip->irq_ack)
        chip->irq_ack(&desc->irq_data);

    if (likely(action)) {
        trace_irq_handler_entry(irq, action);
        res = action->handler(irq, raw_cpu_ptr(action->percpu_dev_id));
        trace_irq_handler_exit(irq, action, res);
    }

    if (chip->irq_eoi)
        chip->irq_eoi(&desc->irq_data);
}
```

`percpu_dev_id` 是 per-CPU 中断特有的——每个核看到的设备 ID 不同。入口可能调 `irq_ack`，出口可能调 `irq_eoi`，具体取决于芯片。

---
#### 选型总结

| handler | 适用条件 | 本平台谁用 |
|---------|---------|-----------|
| `handle_level_irq` | 芯片不自动管理，电平触发，软件必须 mask 防死循环 | **不用**（EXTI 走 fasteoi） |
| `handle_edge_irq` | 芯片不自动管理，边沿触发，软件必须循环防丢边沿 | **不用** |
| `handle_fasteoi_irq` | 芯片硬件自动管理状态，软件只需 EOI | **所有 SPI + EXTI** |
| `handle_simple_irq` | 芯片不需要任何流程控制 | 极少用 |
| `handle_percpu_devid_irq` | per-CPU 中断，不加锁，不分发 | **GIC PPI**（Timer、PMU）

### 4.4 STM32MP257 上各中断的 flow handler 选择

在 GIC 的 `gic_irq_domain_map()`（`irq-gic.c`，第 1063 行）中，flow handler 根据中断类型选择：

```c
static int gic_irq_domain_map(struct irq_domain *d, unsigned int irq,
                              irq_hw_number_t hw)
{
    switch (hw) {
    case 0 ... 31:
        /* SGI (0~15) + PPI (16~31) → per-CPU handler */
        irq_domain_set_info(d, irq, hw, chip, d->host_data,
                            handle_percpu_devid_irq, NULL, NULL);
        break;
    default:
        /* SPI (32+) → handle_fasteoi_irq */
        irq_domain_set_info(d, irq, hw, chip, d->host_data,
                            handle_fasteoi_irq, NULL, NULL);
        break;
    }
    ...
}
```

| 中断类型 | hwirq 范围 | Flow Handler | 原因 |
|----------|-----------|-------------|------|
| SGI（核间中断） | 0~15 | `handle_percpu_devid_irq` | 软件触发，每个 CPU 单独处理 |
| PPI（私有外设中断） | 16~31 | `handle_percpu_devid_irq` | 每个 CPU 独享，不需同步 |
| SPI（共享外设中断） | 32~1019 | `handle_fasteoi_irq` | GIC 硬件自动管理，只需 EOI |

**注意：EXTI 的中断也走 `handle_fasteoi_irq`。** 原因在层级域的 alloc 时序：

```
EXTI domain alloc（第 712 行）:
  irq_domain_set_hwirq_and_chip()    ← 设 chip = stm32mp_exti_chip
  irq_domain_alloc_irqs_parent()     ← 调 parent (GIC) 的 alloc
    → GIC domain alloc（第 1160 行）:
      gic_irq_domain_map()
        irq_domain_set_info(d, virq, hwirq, chip, host_data,
                            handle_fasteoi_irq, NULL, NULL)   ← 设 handle_irq
```

EXIT 设 **chip**（操作硬件寄存器），GIC 设 **handle_irq**（flow handler 函数指针）。最终 `desc->handle_irq = handle_fasteoi_irq`，`desc->irq_data.chip = &stm32mp_exti_chip`。

以 PH5 按键为例：DTS 配 `IRQ_TYPE_EDGE_FALLING`：

- `stm32mp_exti_set_type()` 写 FTSR 配置下降沿检测，然后调 `irq_chip_set_type_parent(d, IRQ_TYPE_LEVEL_HIGH)`——告诉 GIC 这是电平高触发
- 按键按下 → EXTI 检测下降沿 → 设 FPR pending → 输出电平高给 GIC
- `handle_fasteoi_irq` 被调用 → 不 mask、不 ack（GIC 入口读 IAR 已完成）→ 调 `action->handler()`
- handler 返回 → `cond_unmask_eoi_irq()` → `stm32mp_exti_eoi()` 清 FPR + 恢复 IMR + `irq_chip_eoi_parent()` 写 GICC_EOIR

EXIT 的边沿/电平差异不体现在 flow handler 上（一律 fasteoi），而是体现在 **chip 回调的参数配置**中——`set_type` 配 RTSR 还是 FTSR，`eoi` 清 RPR 还是 FPR。

---

## 5. struct irqaction：设备驱动的中断注册入口

从 §1 的四个核心问题来看，irqaction 位于**分发**环节的最末端——它的 `handler` 函数就是最终被调用的设备驱动中断处理函数。

irqaction 是中断子系统中**最靠近设备驱动**的部分。它不像 irq_chip 那样操作硬件，也不像 irq_domain 那样做编号翻译。它的职责只有一个：**把设备驱动的 handler 挂到 irq_desc 上，在中断发生时被调用**。

### 5.1 irqaction 结构体全字段解析

`struct irqaction` 定义在 `include/linux/interrupt.h`，第 118 行：

```c
struct irqaction {
    irq_handler_t       handler;         /* 硬中断 handler（顶半部）*/
    void                *dev_id;         /* 设备标识，共享中断时区分设备 */
    void __percpu       *percpu_dev_id;  /* per-CPU 设备标识 */
    struct irqaction    *next;           /* 共享中断链表的下一个节点 */
    irq_handler_t       thread_fn;      /* 线程化 handler（底半部）*/
    struct task_struct  *thread;         /* 内核线程的 task_struct */
    struct irqaction    *secondary;      /* 强制线程化时的次级 action */
    unsigned int        irq;             /* Linux IRQ 号 */
    unsigned int        flags;           /* IRQF_* 标志 */
    unsigned long       thread_flags;    /* 线程状态标记 */
    unsigned long       thread_mask;     /* 共享中断时线程唤醒位掩码 */
    const char          *name;           /* 设备名（/proc/interrupts 显示）*/
    struct proc_dir_entry *dir;          /* /proc/irq/NN/name 目录项 */
};
```

下面按功能分组逐字段说明。

---
**`handler` 与 `thread_fn`——顶半部和底半部**

这是 irqaction 最重要的两个字段，它们决定了中断在哪里执行：

```c
typedef irqreturn_t (*irq_handler_t)(int, void *);
```

- **`handler`（顶半部）**：在硬中断上下文中执行（IRQ 关闭）。只能做最少操作——读状态、保存数据、清中断条件。需要睡眠的操作（I2C 读写、内存分配、互斥锁）绝对不能放在这里。

- **`thread_fn`（底半部）**：在内核线程中执行（IRQ 打开，可睡眠）。当 handler 返回 `IRQ_WAKE_THREAD` 时，core 层唤醒 `action->thread`，线程执行 thread_fn。

两者至少有一个不为 NULL：
- `handler != NULL, thread_fn == NULL` → 纯顶半部中断
- `handler != NULL, thread_fn != NULL` → 线程化中断（handler 做最少处理，thread_fn 做主要工作）
- `handler == NULL, thread_fn != NULL` → 纯线程化中断（`IRQF_ONESHOT` 常用模式，参见 §5.3）

---
**`thread`——内核线程的 task_struct**

当 `thread_fn` 不为 NULL 时，`__setup_irq()` 在注册时创建内核线程：

```c
action->thread = kthread_create(irq_thread, action, "irq/%d-%s", irq, name);
```

`irq_thread()` 是 `kernel/irq/manage.c` 中的线程主函数。它的循环逻辑：

```c
static int irq_thread(void *data)
{
    struct irqaction *action = data;
    ...
    while (!irq_wait_for_interrupt(action)) {    /* 等待中断触发 */
        if (action->thread_fn)                   /* 调底半部 */
            action->thread_fn(action->irq, action->dev_id);
        ...
    }
}
```

`threads_active` 字段（在 irq_desc 中）跟踪当前正在运行的 thread 数，`synchronize_irq()` 等待这个计数归零才返回。

---
**`dev_id`——共享中断的唯一身份证**

`dev_id` 是 `void *`，设备驱动传什么，core 层就保存什么，只做两个用途：

1. **`free_irq(irq, dev_id)` 时匹配释放**——只释放 dev_id 相同的那个 irqaction
2. **共享中断的 handler 中区分来源**——多个设备共享同一根中断线，handler 先查自己的 dev_id 判断是否自己的设备产生了中断

```c
/* 共享中断 handler 的典型写法 */
static irqreturn_t my_handler(int irq, void *dev_id)
{
    struct my_dev *dev = dev_id;
    if (!readl(dev->reg + INT_STATUS))
        return IRQ_NONE;          /* 不是我的中断 */
    /* 处理中断 */
    return IRQ_HANDLED;
}
```

如果没有共享，dev_id 可以传 NULL（但最好传设备结构体，方便调试读 `/proc/interrupts` 时看到关联）。

---
**`percpu_dev_id`——per-CPU 中断的 dev_id**

per-CPU 中断（PPI）的 `dev_id` 是 `void __percpu *`——每个核看到不同的设备 ID。例如，每个 CPU 都有自己的 Timer，Timer 中断 handler 需要知道当前 CPU 的 Timer 寄存器基址，通过 `raw_cpu_ptr(percpu_dev_id)` 获取。

```c
/* handle_percpu_devid_irq 中调用 */
res = action->handler(irq, raw_cpu_ptr(action->percpu_dev_id));
```

---
**`next`——共享中断链表**

当两个设备共用一根中断线（`IRQF_SHARED`）时，`desc->action` 指向第一个 irqaction，第一个的 `next` 指向第二个，形成单向链表。`handle_irq_event()` 遍历整个链表，逐个调用 handler，直到某个返回 `IRQ_HANDLED`。

```
desc->action → irqaction(devA) → irqaction(devB) → NULL
```

---
**`flags`——IRQF_* 注册标志**

驱动调 `request_irq` 时传入的标志，保存在 irqaction 中供 flow handler 和 core 层决策。

| 标志 | 含义 | 使用场景 |
|------|------|---------|
| `IRQF_TRIGGER_*` | 触发类型（高/低/上升/下降） | 配外设中断类型 |
| `IRQF_SHARED` | 允许共享中断线 | 多个设备共用一条中断 |
| `IRQF_ONESHOT` | handler 返回后不 unmask，等 thread_fn 完成 | 线程化中断防中断风暴 |
| `IRQF_NO_SUSPEND` | suspend 时不关中断 | 唤醒源 |
| `IRQF_NO_THREAD` | 禁止强制线程化 | 实时性要求高的中断 |

---
**`irq`——Linux IRQ 号**

`request_irq(irq, handler)` 的第一个参数，保存在 irqaction 中供日志和 debug 使用。`/proc/interrupts` 第一列就是这个号。

---
**`thread_flags` 与 `thread_mask`**

共享中断中多个线程化 handler 之间的同步。`thread_mask` 是位掩码——每个共享的 irqaction 分配一个 bit。flow handler 调 `handle_irq_event()` 返回后，根据 `thread_mask` 唤醒对应的线程。`thread_flags` 标记线程是否需要被唤醒。

---
**`name`——设备名**

显示在 `/proc/interrupts` 中，方便排查哪个中断是哪个设备。

```shell
$ cat /proc/interrupts
           CPU0       CPU1
 29:       1234       5678      GICv2  29  timer
 74:          0          0      GICv2  74  mmc1
```

---
**`dir`——procfs 目录项**

每个注册的 irqaction 在 `/proc/irq/N/` 下有一个以其 name 命名的子目录。例如 `irq 74` 关联了 `mmc1` 驱动，`/proc/irq/74/mmc1/` 下有 node 等调试文件。这个目录在 action 注册时由 proc 文件系统自动创建，free_irq 时删除。

### 5.2 irqaction 的注册与销毁

设备驱动通过 `request_irq()` 或 `request_threaded_irq()` 注册中断 handler。`request_irq()` 是 `request_threaded_irq(irq, handler, NULL, flags, name, dev)` 的封装——`thread_fn = NULL` 表示纯顶半部。

注册流程（数据结构层面的流程，不是逐行代码）：

```
request_threaded_irq(irq, handler, thread_fn, flags, name, dev_id)
    │
    ├─ ① 分配 struct irqaction（new_action）
    │    填充: .handler = handler, .thread_fn = thread_fn,
    │          .flags = flags, .name = name, .dev_id = dev_id
    │
    └─ __setup_irq(irq, desc, new_action)
         │
         ├─ ② 根据 handler/thread_fn 设置 action 类型
         │    （纯顶半部 / 线程化 / 纯线程化）
         │
         ├─ ③ IRQF_SHARED 检查
         │    └─ 检查已有 handler 的触发类型是否兼容
         │    └─ 检查硬件是否支持共享
         │
         ├─ ④ 将 new_action 插入 desc->action 链表
         │    └─ 如果 desc->action == NULL → 它就是链表头
         │    └─ 如果已存在 → 追加到链表末尾（共享中断）
         │
         ├─ ⑤ 调用 irq_chip 的设置
         │    └─ irqd_set_trigger_type() — 根据 flags 设置触发类型
         │    └─ 可能调 chip->irq_set_type()
         │
         ├─ ⑥ 创建线程（如果 thread_fn 不为 NULL）
         │    └─ kthread_create(irq_thread, new_action, ...)
         │    └─ new_action->thread = kthread 的 task_struct
         │
         └─ ⑦ 启用中断
              └─ 最终调 chip->irq_unmask()
```

**共享中断（IRQF_SHARED）**：多个设备物理上共用一条中断线。注册时：

- `dev_id` 是共享中断的唯一区分方法——`free_irq()` 和 handler 中都需要用 `dev_id` 区分是哪个设备产生了中断
- 所有共享 handler 执行完，如果返回都是 `IRQ_NONE`，core 层认为这是假中断

### 5.3 中断线程化机制

中断线程化是内核将**底半部处理**从硬中断上下文移到内核线程中执行的方法。

```
request_threaded_irq(irq, handler, thread_fn, flags, name, dev_id)
    │
    │  handler != NULL, thread_fn != NULL
    │
    ▼
action->handler   ← 顶半部（硬中断上下文，IRQ 关闭）
action->thread_fn ← 底半部（内核线程，IRQ 打开，可睡眠）
```

执行流程：

```
中断触发
    │
    ├─ flow handler（如 handle_fasteoi_irq）
    │    └─ handle_irq_event(desc)
    │         └─ __handle_irq_event_percpu(desc)
    │              ├─ 调 action->handler()     ← 顶半部（IRQ off）
    │              │    └─ 返回 IRQ_WAKE_THREAD → 标记需要唤醒线程
    │              │
    │              └─ action->thread_flags 标记
    │
    └─ 顶半部返回
         │
         ├─ cond_unmask_eoi_irq()—根据 IRQF_ONESHOT 决定是否 unmask
         │
         └─ 唤醒 action->thread
              └─ irq_thread()
                   ├─ set_current_state(TASK_RUNNING)
                   └─ 调 action->thread_fn()   ← 底半部（IRQ on，可睡眠）
```

#### IRQF_ONESHOT 的设计原理

`IRQF_ONESHOT` 标志的含义：**中断处理期间（从顶半部开始到底半部结束），中断线保持 mask 状态**。

为什么需要这个标志？考虑一个场景：

1. 外设触发中断，电平变高
2. 硬件自动将 pending 位置 1
3. 顶半部 `handler()` 执行，写外设寄存器清除了中断条件
4. 但电平下降需要时间（µs 级），电平还没回到低
5. 如果此时 unmask，GIC 检测到电平仍高 → 立即再次触发中断
6. CPU 再次进入中断 → 再次跑顶半部 → **中断风暴**

`IRQF_ONESHOT` 在第 4 步阻止 unmask：**只有底半部 `thread_fn()` 执行完成后才能 unmask**。底半部返回时，外设已经完成了电平切换，安全了。

在代码中体现（`handle_fasteoi_irq`，第 717 行）：

```c
if (desc->istate & IRQS_ONESHOT)
    mask_irq(desc);        /* ONESHOT → 执行 handler 之前就 mask */

handle_irq_event(desc);    /* 执行 action->handler */

cond_unmask_eoi_irq(desc, chip);   /* thread_fn 完成后才 unmask */
```

`cond_unmask_eoi_irq()`（第 657 行）的逻辑：

```c
static void cond_unmask_eoi_irq(struct irq_desc *desc, struct irq_chip *chip)
{
    if (!(desc->istate & IRQS_ONESHOT)) {
        chip->irq_eoi(&desc->irq_data);         /* 无 ONESHOT → 直接 EOI */
        return;
    }
    /* ONESHOT 模式：只在 thread 未运行或已完成时 unmask */
    if (!irqd_irq_disabled(&desc->irq_data) &&
        irqd_irq_masked(&desc->irq_data) && !desc->threads_oneshot) {
        chip->irq_eoi(&desc->irq_data);
        unmask_irq(desc);
    } else if (!(chip->flags & IRQCHIP_EOI_THREADED)) {
        chip->irq_eoi(&desc->irq_data);
    }
}
```

核心判断：`!desc->threads_oneshot`——没有线程正在运行，才算安全。

---

## 6. STM32MP257 中断拓扑总览

前面四章分别讲了 irq_chip、irq_domain、irq_desc、irqaction 四个层次。本章将它们套回到 STM32MP257 的具体硬件上，画出一张完整的拓扑图。

### 6.1 GIC-400（GICv2）在 STM32MP257 上的实现

STM32MP257 集成了 ARM GIC-400，属于 GICv2 架构。DTS 定义（`stm32mp251.dtsi`，第 200 行）：

```dts
intc: interrupt-controller@4ac00000 {
    compatible = "st,stm32mp2-cortex-a7-gic", "arm,cortex-a7-gic";
    #interrupt-cells = <3>;
    interrupt-controller;
    reg = <0x0 0x4ac10000 0x0 0x1000>,    /* GICD — Distributor */
          <0x0 0x4ac20000 0x0 0x2000>,    /* GICC — CPU Interface */
          <0x0 0x4ac40000 0x0 0x2000>,    /* GICH — Hypervisor */
          <0x0 0x4ac60000 0x0 0x2000>;    /* GICV — Virtual CPU */
    interrupts = <GIC_PPI 9 (GIC_CPU_MASK_SIMPLE(1) | IRQ_TYPE_LEVEL_LOW)>;

    v2m0: v2m@48090000 {
        compatible = "arm,gic-v2m-frame";  /* MSI 支持 */
        reg = <0x0 0x48090000 0x0 0x1000>;
        msi-controller;
    };
};
```

四个寄存器区域的映射：

| 区域 | 基址 | 大小 | 作用 |
|------|------|------|------|
| GICD（Distributor） | 0x4ac10000 | 4KB | 中断分发——使能/类型/优先级/affinity |
| GICC（CPU Interface） | 0x4ac20000 | 8KB | CPU 接口——读 IAR 取中断号、写 EOI 完成处理 |
| GICH（Hypervisor） | 0x4ac40000 | 8KB | 虚拟化支持 |
| GICV（Virtual CPU） | 0x4ac60000 | 8KB | 虚拟机 CPU 接口 |

GIC-400 的中断容量通过 `GIC_DIST_CTR` 寄存器读取（`irq-gic.c`，第 1215 行）：

```c
gic_irqs = readl_relaxed(gic_data_dist_base(gic) + GIC_DIST_CTR) & 0x1f;
gic_irqs = (gic_irqs + 1) * 32;
if (gic_irqs > 1020)
    gic_irqs = 1020;
```

对于 STM32MP257，`GICD_CTR.IT_LINES_NUMBER` 字段指示的中断线数量决定了 GIC domain 的 `revmap` 大小。

GIC 的四状态硬件状态机（Inactive / Pending / Active / Active & Pending）以及各操作对应的状态转换，已在 §2.2 的"从 GIC 状态机理解 irq_ack、irq_eoi、mask 的关系"中详细阐述。这里不再重复。

作为补充，开发者可以通过 `gic_irq_get_irqchip_state()` 在运行时读取某个中断的当前硬件状态（`irq-gic.c`，第 279 行）：

```c
static int gic_irq_get_irqchip_state(struct irq_data *d,
                                      enum irqchip_irq_state which, bool *val)
{
    switch (which) {
    case IRQCHIP_STATE_PENDING:
        *val = gic_peek_irq(d, GIC_DIST_PENDING_SET);
        break;
    case IRQCHIP_STATE_ACTIVE:
        *val = gic_peek_irq(d, GIC_DIST_ACTIVE_SET);
        break;
    case IRQCHIP_STATE_MASKED:
        *val = !gic_peek_irq(d, GIC_DIST_ENABLE_SET);
        break;
    ...
    }
}
```

对应的寄存器：`GIC_DIST_PENDING_SET`（GICD_ISPENDR）和 `GIC_DIST_ACTIVE_SET`（GICD_ISACTIVER）分别映射 Pending 和 Active 位。这在调试中断卡住的问题时很有用——查一下是不是中断卡在了 Pending 状态而无人处理。

### 6.2 EXTI 控制器内部结构

EXTI（Extended Interrupt Controller）是 STM32 系列 SoC 的特色模块。在 STM32MP257 上，它管理着 **96 个外部/唤醒事件**，分为 3 个 bank，每个 bank 32 个事件。

DTS 定义（`stm32mp251.dtsi`，第 2661 行）：

```dts
exti1: interrupt-controller@44220000 {
    compatible = "st,stm32mp1-exti";
    interrupt-controller;
    #interrupt-cells = <2>;
    reg = <0x44220000 0x400>;
    interrupts-extended =
        <&intc GIC_SPI 268 IRQ_TYPE_LEVEL_HIGH>,    /* EXTI_0 */
        <&intc GIC_SPI 269 IRQ_TYPE_LEVEL_HIGH>,    /* EXTI_1 */
        ...
        <0>,                                         /* EXTI_20 — 未连到 GIC */
        ...
        <&intc GIC_SPI 134 IRQ_TYPE_LEVEL_HIGH>;    /* EXTI_70 */
};
```

#### 寄存器布局

每个 bank 有相同的一组寄存器（偏移定义在 `irq-stm32mp-exti.c`，第 93 行）：

| 寄存器 | 偏移（Bank1） | 作用 |
|--------|-------------|------|
| RTSR | 0x00 | 上升沿触发选择（Rising Trigger Selection） |
| FTSR | 0x04 | 下降沿触发选择（Falling Trigger Selection） |
| SWIER | 0x08 | 软件中断事件寄存器（SW Interrupt/Event） |
| RPR | 0x0C | 上升沿 pending 寄存器（Rising Pending） |
| FPR | 0x10 | 下降沿 pending 寄存器（Falling Pending） |
| SECCFGR | 0x14 | 安全配置寄存器 |
| IMR | 0x80 | 中断屏蔽寄存器（Interrupt Mask） |
| TRG | 0x3EC | 触发类型寄存器（configurable event vs direct） |

> 这个 TRG 寄存器很关键——它决定了这个 event 是走**可配置通道**（`stm32mp_exti_chip`，含 EOI/mask/unmask/set_type）还是**直通通道**（`stm32mp_exti_chip_direct`，委托 parent）。驱动在 `stm32mp_exti_domain_alloc()` 中根据 TRG 位选择 chip 实例。
> **GPIO mux 的硬件选路约束**：STM32MP2 的 EXTI 控制器有 16 个外部中断线（EXTI0~EXTI15），每个 EXTIx **只能从 PAx、PBx、……、PKx 中选择一个 GPIO 引脚作为中断源**。也就是说，PA0 使用了 EXTI0，PB0 就不能再使用 EXTI0。这与很多 SoC 的"任一 GPIO 引脚都可以独立产生中断"不同——STM32MP2 的 GPIO 中断是**通过 EXTI mux 共享 16 条中断线的**。
>
> 驱动中 `stm32mp_exti_test_gpio_mux_available()`（第 548 行）实现这一选路约束检查：
>
> ```c
> static bool stm32mp_exti_test_gpio_mux_available(
>     struct stm32mp_exti_host_data *host_data,
>     unsigned int bank_nr, unsigned int gpio_nr)
> {
>     if (gpio_nr >= STM32MP_GPIO_IRQ_LINES)  /* 只支持 EXTI0~EXTI15 */
>         return false;
>     if (!test_bit(gpio_nr, host_data->gpio_mux_used) ||
>         bank_nr == host_data->gpio_mux_pos[gpio_nr])  /* 同 bank 可以复用 */
>         return true;
>     return false;  /* 冲突！不同 bank 的相同 GPIO 号 */
> }
> ```
>
> 逻辑：每个 `gpio_nr`（即 EXTI 线号）第一次被使用时记录 `gpio_mux_pos[gpio_nr] = bank_nr`，后续再分配时只允许同一个 bank 使用这条线。例如 PH5（bank=7, pin=5）用了 EXTI5，同一 bank 的另一个 PH 引脚还可以用 EXTI5（通过 `EXTI_CR` 的 mux 重新选路），但 PA5 就不能再用 EXTI5。
>
> 硬件上这个选路是通过 `EXTI_CR(n)` 寄存器配置的（`irq-stm32mp-exti.c`，第 35 行）：
>
> ```c
> #define EXTI_CR(n)         (0x060 + ((n) / 4) * 4)
> #define EXTI_CR_SHIFT(n)   (((n) % 4) * 8)
> #define EXTI_CR_MASK(n)    (GENMASK(7, 0) << EXTI_CR_SHIFT(n))
> ```
>
> 每个 EXTI_CR 寄存器管理 4 个 EXTI 线的 GPIO bank 选路，每个线占 8 bit。值 `0x07` 表示 PH bank（bank 7）。

每个 event 的硬件中断号映射关系（`stm32mp_exti_domain_alloc` 中 `interrupts-extended` 路径）：

```
EXTI event 号（DTS 中 param[0]）→ 查 DT interrupts-extended 属性
                                  → 得到 parent (GIC) 的 fwspec
                                  → irq_domain_alloc_irqs_parent() 传递给 GIC
```

### 6.3 完整中断信号拓扑

把 GIC 和 EXTI 整合到一张图上：

```
                      ┌─────────────────────────────────────────────────┐
                      │              Cortex-A35 CPU0                     │
                      │  ┌──────────────────────────────────────────┐   │
                      │  │  GICC — CPU Interface                    │   │
                      │  │  读 GICC_IAR → 取中断号                     │   │
                      │  │  写 GICC_EOIR → EOI（priority drop）       │   │
                      │  └──────────────┬───────────────────────────┘   │
                      └─────────────────┼───────────────────────────────┘
                                        │
                      ┌─────────────────▼───────────────────────────────┐
                      │        GICD — Distributor                       │
                      │                                                │
                      │  ● 使能/屏蔽：GICD_ISENABLER/ICENABLER        │
                      │  ● 触发类型：GICD_ICFGR（2bit/中断）           │
                      │  ● 优先级：GICD_IPRIORITYR                     │
                      │  ● Affinity：GICD_ITARGETSR（每中断 1 字节）    │
                      │                                                │
                      │  中断范围：hwirq 0~1019                        │
                      │  ● SGI 0~15:    核间中断（IPI）                │
                      │  ● PPI 16~31:   私有外设中断（Timer, PMU）     │
                      │  ● SPI 32~1019: 共享外设中断                   │
                      └──────────┬────────────────┬────────────────────┘
                                 │                │
                    ┌────────────▼────┐    ┌──────▼───────────────┐
                    │  直连 SPI 中断    │    │  EXTI 转发的中断      │
                    │  USART1/SDMMC/  │    │  （通过 interrupts-   │
                    │  I2C/TIMER 等   │    │   extended 映射）     │
                    │  handle_fasteoi │    │                       │
                    └─────────────────┘    └──────────┬────────────┘
                                                      │
                    ┌─────────────────────────────────▼─────────────────┐
                    │  EXTI — 外部中断控制器 (0x44220000)               │
                    │                                                   │
                    │  3 banks × 32 event = 96 外部事件                 │
                    │                                                   │
                    │  EXTI Event 信号路径（以按键 PH5 为例）：          │
                    │                                                   │
                    │  GPIO PH5                                        │
                    │      │                                           │
                    │      ▼                                           │
                    │  GPIO bank（Pinctrl → GPIO）                      │
                    │      │ 通过 GPIO mux 选择连到哪个 EXTI event      │
                    │      ▼                                           │
                    │  EXTI event N（例如 event 42）                    │
                    │      │                                           │
                    │      ▼                                           │
                    │  EXTI 边沿检测 → RPR/FPR pending                  │
                    │      │                                           │
                    │      ▼                                           │
                    │  读 TRG: configurable?                            │
                    │   ├── Yes → 走 stm32mp_exti_chip                 │
                    │   │         (含 EOI/set_type/mask/unmask)         │
                    │   └── No  → 走 stm32mp_exti_chip_direct          │
                    │                     (委托 parent 所有操作)         │
                    │      │                                           │
                    │      ▼                                           │
                    │  输出电平高给 GIC                                 │
                    └──────────────────────────────────────────────────┘
```

### 6.4 中断号空间分配

结合 DTS 和驱动代码，可以总结 STM32MP257 上的 IRQ 号分配：

| 中断类型 | hwirq 范围 | Virq 分配方式 | Flow Handler |
|----------|-----------|-------------|-------------|
| GIC SGI（核间中断） | 0~15 | irq_domain_create_linear，预分配 | `handle_percpu_devid_irq` |
| GIC PPI（私有外设） | 16~31 | irq_domain_create_linear，预分配 | `handle_percpu_devid_irq` |
| GIC SPI（共享外设） | 32~1019 | irq_domain_create_linear，按需分配 | `handle_fasteoi_irq` |
| EXTI event | 0~95 | 子域（层级域），按需分配 | `handle_fasteoi_irq`（由 GIC 设置） |

SPI 的典型分配示例（从 DTS 中摘录）：

| 设备 | hwirq | DTS 写法 | 谁管 |
|------|-------|---------|------|
| SDMMC1 | 42+32=74 | `<GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>` | GIC 直连 |
| USART1 | 45+32=77 | `<GIC_SPI 45 IRQ_TYPE_LEVEL_HIGH>` | GIC 直连 |
| I2C3 | 73+32=105 | `<GIC_SPI 73 IRQ_TYPE_LEVEL_HIGH>` | GIC 直连 |
| EXTI event 0 | EXTI→GIC SPI 268 | `interrupts-extended = <&exti1 0 ...>` | 经 EXTI |
| EXTI event 42 | EXTI→GIC SPI 153 | `interrupts-extended = <&exti1 42 ...>` | 经 EXTI |

### 6.5 数据结构全景（PH5 按键中断示例）

以 ATK 板上 PH5 按键（GPIO 中断）为例，从最底层到最上层的数据结构连接：

```
设备树中配置（见 01-Usage.md §1.1.3）:
    gpio-keys {
        key-ph5 {
            gpios = <&gpiop 5 GPIO_ACTIVE_LOW>;
        };
    };

驱动代码中使用:
    irq = gpiod_to_irq(gpio);          ← 拿到 virq
    devm_request_threaded_irq(dev, irq, NULL, ph5_key_handler,
                              IRQF_TRIGGER_FALLING | IRQF_ONESHOT,
                              "ph5-key", dev);

完整数据结构链:
┌─────────────────────────────────────────────────────────────────────┐
│  irq_desc[N]                                                        │
│    .irq_data                                                        │
│      .irq       = N           ← virq（gpiod_to_irq 返回值）          │
│      .hwirq     = EXTI event 号（如 42）                            │
│      .chip      → &stm32mp_exti_chip    ← EXTI 的 irq_chip         │
│      .chip_data → stm32mp_exti_chip_data ← EXTI 私有数据（IMR 等）    │
│      .domain    → EXTI irq_domain       ← 子域                      │
│      .parent_data → irq_data[1]                                     │
│                      .irq       = N                                  │
│                      .hwirq     = 153  ← GIC SPI (268-32+? wait)    │
│                      .chip      → &gic_chip ← GIC 的 irq_chip        │
│                      .domain    → GIC irq_domain ← 根域              │
│                      .parent_data = NULL                              │
│    .handle_irq = handle_fasteoi_irq          ← 由 GIC 设置的 flow handler │
│      注：EXTI 的芯片回调（eoi/mask）负责其自身流逻辑                   │
│    .action → irqaction{                                             │
│        .handler   = NULL                 ← 纯线程化中断             │
│        .thread_fn = ph5_key_handler      ← 底半部回调               │
│        .thread    = kthread "irq/N-ph5-key"                         │
│        .flags     = IRQF_TRIGGER_FALLING | IRQF_ONESHOT            │
│        .dev_id    = dev                                              │
│        .next      = NULL                   ← 非共享中断             │
│    }                                                                 │
│    .depth       = 0                     ← 使能中                    │
│    .lock        = raw_spinlock_t                                      │
│    .kstat_irqs  = per-CPU 计数器                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. 总结

中断子系统采用**四层架构**，每层包含对应的核心数据结构：

| 层 | 主要数据结构 | 职责 | 类比 |
|----|---------|------|------|
| **消费者层** | `struct irqaction` | 设备驱动注册 handler，中断最终落在此处 | 方向盘——最终操作的人 |
| **Core 层** | `struct irq_desc`（流程控制）+ `struct irq_domain`（编号翻译）+ `struct irq_chip`（接口定义） | 决定何时 mask/ack/eoi，翻译 hwirq → virq | 变速箱——什么时候做什么操作 |
| **控制器驱动层** | `struct irq_chip`（实现） | 封装硬件差异（mask/unmask/ack/eoi/set_type） | 内燃机——怎么产生动力 |
| **硬件层** | GIC/EXTI 寄存器 | 产生和传递中断信号 | 车轮——物理执行 |

四层解决四个核心问题：

1. **irq_domain**：这个中断是哪来的（hwirq → virq 映射）
2. **irq_chip**：怎么开关和确认这个中断（mask/unmask/ack/eoi）
3. **irq_desc**：什么时机做开关确认（flow handler 的时序）
4. **irqaction**：谁来做最终处理（设备驱动的 handler）

分开的收益：

- 换 GIC 到 GICv3 时，只需要换控制器驱动层（`irq-gic-v3.c`），Core 层的 flow handler 逻辑不变
- 增加 EXTI 这样的新控制器时，只需要写新的 irq_domain 和 irq_chip，挂到 GIC 的 domain 下
- 驱动开发者不需要关心底下有几层控制器，只需要 `request_irq(irq, handler)`——层级之间通过 parent 域自动传递

下一章 03-Source-Analysis.md 将沿着具体代码路径，逐行走一遍 probe、request_irq、中断触发、flow handler 执行、threaded handler 唤醒的完整流程。

---

*本章完 · 下一章：03-Source-Analysis.md — 中断源码分析*

