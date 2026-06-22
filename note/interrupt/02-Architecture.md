# 核心数据结构与设计原理

> 中断子系统的静态结构（不涉及代码流程，那是后面文档的事）。
>
> **字数**：预估约 12000 字 · **建议阅读时间**：40~70 分钟

---

## 1. 设计思想：四个核心问题与四层架构

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

沿着这条路径，中断子系统要解决的问题可以归纳为四个层次：

| # | 问题 | 管什么 | 对应层次 |
|---|------|--------|---------|
| 1 | **识别** | 刚才收到的是哪个中断？ | irq_domain（hwirq → virq 映射） |
| 2 | **控制** | 怎么关/开/确认/结束这个中断？ | irq_chip（硬件寄存器操作） |
| 3 | **路由** | 这个中断发给哪个 CPU？ | irq_chip + GIC 分发逻辑 |
| 4 | **分发** | 哪个驱动的 handler 来处理？ | irq_desc + irqaction 链表 |

下面用 ASCII 图展示这四个问题的层级关系——**从硬件向软件逐层映射**：

```
硬件中断信号
    │
    ▼
┌──────────────────────────────────────┐
│  问题 1：这是什么中断？              │
│  irq_domain: (中断控制器, hwirq) → virq │
└──────────────────────┬───────────────┘
                       │ 得到 Linux IRQ 号
                       ▼
┌──────────────────────────────────────┐
│  问题 2：怎么控制硬件？              │
│  irq_chip: mask/unmask/ack/eoi/set_type │
└──────────────────────┬───────────────┘
                       │ 操作寄存器
                       ▼
┌──────────────────────────────────────┐
│  问题 3：发给哪个 CPU？              │
│  GIC Distributor + irq_set_affinity    │
└──────────────────────┬───────────────┘
                       │ 中断到达 CPU
                       ▼
┌──────────────────────────────────────┐
│  问题 4：谁处理这个中断？            │
│  irq_desc.handle_irq → irqaction.handler │
│  (flow control + 设备驱动 handler)       │
└──────────────────────────────────────┘
```

### 1.2 四层架构总览

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
│  │  1. irq_chip 层 — 硬件操作接口                      │              │
│  │     chip.c:  mask/unmask/ack/eoi → 各 irq_chip 回调 │              │
│  │                                                     │              │
│  │  2. irq_desc 层 — 流程控制                          │              │
│  │     handle_level_irq / handle_fasteoi_irq           │              │
│  │     handle_edge_irq / handle_simple_irq             │              │
│  │     handle_percpu_irq / handle_percpu_devid_irq    │              │
│  │                                                     │              │
│  │  3. irq_domain 层 — 中断编号映射                    │              │
│  │     irqdomain.c:  (controller, hwirq) → virq        │              │
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
| **irq_chip 层** | 封装硬件差异——mask 是写 GICD_ICENABLER 还是写 EXTI_IMR | 不决定何时调用这些操作 |
| **irq_desc 层** | 决定 flow control 逻辑——入口是否 mask、何时 ack、何时 unmask | 不直接操作硬件 |
| **irq_domain 层** | 建立从硬件中断号到 Linux IRQ 号的映射 | 不参与中断处理路径 |

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
| **irq_chip 层** | `struct irq_chip` + `struct irq_data` | §2 + §4.2 | mask/unmask/ack/eoi/set_type |
| **irq_desc 层** | `struct irq_desc` | §4 | flow control（5 种 handler） |
| **irq_domain 层** | `struct irq_domain` | §3 | hwirq → virq 翻译 |

全文以 STM32MP257 上的 GIC + EXTI 为实际参考平台。

---

## 2. irq_chip 层：封装中断控制器的硬件差异

中断子系统的第一层叫 irq_chip。它的职责非常简单：**把「操作中断控制器的寄存器」封装成统一回调接口**。

每个中断控制器虽然都提供 mask/unmask/ack/eoi 等功能，但寄存器的地址、位域、操作顺序各不同。irq_chip 层的作用就是把这些差异挡住，让上层（flow handler）只需调用 `irq_data->chip->irq_mask(d)`，不用关心底下是 GIC 还是 EXTI。

### 2.1 irq_chip 结构体全景

`struct irq_chip` 定义在 `include/linux/irq.h`，是中断子系统中**回调函数最多**的结构体（超过 30 个字段）。但驱动开发者只实现其中一小部分——其余可以 NULL（core 层会用默认行为代替）。

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
| **特殊组** | `irq_retrigger` | 重新触发中断（SW 模拟用） | 否 |
| | `irq_get/set_irqchip_state` | 查询/设置硬件中断状态 | 否 |
| | `irq_request/release_resources` | 资源管理（request 时调用） | 否 |

从 $1 的四个核心问题来看，irq_chip 主要回答**问题 2：怎么控制硬件**。

### 2.2 控制组：mask / unmask / ack / eoi

这是 irq_chip 最核心的四个回调。它们的时序关系决定了 flow handler 的类型：

```
硬件中断到达
    │
    ▼
┌────────────────┐
│  mask?         │ ← 是否需要立即屏蔽（防止同一中断重复触发）
└───────┬────────┘
        │
┌───────▼────────┐
│  ack?          │ ← 是否需要确认收悉（清除硬件 pending 位）
└───────┬────────┘
        │
  处理 handler（设备驱动的回调）
        │
┌───────▼────────┐
│  unmask?       │ ← 何时重新打开中断
│  eoi?          │ ← 是否写 EOI 寄存器（GIC 需要）
└────────────────┘
```

#### irq_mask / irq_unmask

这两个回调是最核心的控制操作——相当于中断的"开关"。

**GICv2 实现**（`drivers/irqchip/irq-gic.c`，第 199 行）：

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

`gic_poke_irq()` 内部根据中断号算出在 `GIC_DIST_ENABLE_CLEAR` 或 `GIC_DIST_ENABLE_SET` 寄存器中的 bit 偏移，然后写寄存器。GICv2 的 Distributor 寄存器布局：

```
GICD_ICENABLER（0x180 ~ 0x1FC）— 写 1 关闭对应中断
GICD_ISENABLER（0x100 ~ 0x17C）— 写 1 打开对应中断

每个 bit 对应一个中断号：
  bit 0  = IRQ 0 (SGI0)
  bit 16 = IRQ 16 (PPI0)
  bit 32 = IRQ 32 (SPI0, 第一个 SPI)
```

**EXTI 实现**（`drivers/irqchip/irq-stm32mp-exti.c`，第 376 行）：

```c
static void stm32mp_exti_mask(struct irq_data *d)
{
    struct stm32mp_exti_chip_data *chip_data = irq_data_get_irq_chip_data(d);
    const struct stm32mp_exti_bank *bank = chip_data->reg_bank;

    raw_spin_lock(&chip_data->rlock);
    chip_data->mask_cache &= ~stm32mp_exti_clr_bit(d, bank->imr_ofst);
    raw_spin_unlock(&chip_data->rlock);

    irq_chip_mask_parent(d);   /* ← 层级域链：向上调 GIC 的 mask */
}
```

关键区别：
1. EXTI 有自己的 **mask_cache**（软件缓存），因为 EXTI IMR 是 R/W 寄存器，mask/unmask 不能简单写 1——必须 read-modify-write
2. EXTI mask 之后，还要调 `irq_chip_mask_parent(d)`——因为 EXTI 是 GIC 的子域，mask EXTI 之后 GIC 那边也需要 mask，否则 GIC 认为中断还在 pending
3. 加了 `raw_spinlock` 保护，因为 cache + RMW 操作需要原子性

为什么 EXTI 需要 mask_cache 而 GIC 不需要？看 GIC 的寄存器——`GIC_DIST_ENABLE_CLEAR` 是**写 1 清除**的寄存器（write-only），不需要先读后写。而 EXTI 的 IMR 是**直接读写**的寄存器，必须 RMW。RMW 天然需要缓存和锁。

#### irq_ack

ack 的作用是告诉中断控制器"CPU 已经收到这个中断了"，通常是清除硬件中的 pending 位。

对 GICv2 来说，ack 不是通过 irq_chip 回调完成的——GICv2 的 ack 就是读 `GICC_IAR` 寄存器，这一步在入口汇编代码或 `gic_handle_irq()` 中已经完成了（`readl_relaxed(cpu_base + GIC_CPU_INTACK)`）。所以 GIC 的 irq_chip **不实现** `irq_ack`——它是通过读 IAR 隐式 ack 的。

对 EXTI 来说，ack 操作就是写 RPR/FPR 寄存器来清除 pending：

```c
/* EXTI 没有显式的 irq_ack 回调 */
/* 但是 stm32mp_exti_eoi 中隐含了 pending 清除操作 */
```

实际上，EXTI 的 irq_chip 中 `irq_ack = irq_chip_ack_parent`——把 ack 传给了父域（GIC）。EXTI 自己的 pending 清除在 EOI 中完成（见下文）。

**这一点是理解 GIC + EXTI 层级域的关键**：EXTI 的 ack 通过 `irq_chip_ack_parent()` 转发给 GIC，但 GIC 的 ack 是读 IAR——在入口路径中已经完成了。所以层级域中 ack 的传递其实是**空操作**（GIC 侧不需要软件再做什么）。

#### irq_eoi

EOI（End Of Interrupt）是 GIC 特有的操作。写 `GICC_EOIR` 寄存器通知 GIC"这个中断已经处理完了"，GIC 才会降低当前中断优先级、允许相同或更低优先级的中断再次送达。

**GICv2 标准 EOI**（`irq-gic.c`，第 224 行）：

```c
static void gic_eoi_irq(struct irq_data *d)
{
    u32 hwirq = gic_irq(d);

    /* SGI (hwirq < 16) 需要带上 source CPU ID */
    if (hwirq < 16)
        hwirq = this_cpu_read(sgi_intid);

    writel_relaxed(hwirq, gic_cpu_base(d) + GIC_CPU_EOI);
}
```

**STM32MP2 特有的 EOI 模式**（`irq-gic.c`，第 236 行）：

```c
#define GIC_STM32_CPU_DEACTIVATE 0x10000

static void gic_eoimode1_eoi_irq(struct irq_data *d)
{
    ...
    if (static_branch_unlikely(&gic_stm32_gicc_dir_access))
        writel_relaxed(hwirq, gic_cpu_base(d) + GIC_STM32_CPU_DEACTIVATE);
    else
        writel_relaxed(hwirq, gic_cpu_base(d) + GIC_CPU_DEACTIVATE);
}
```

STM32MP2 把 GICC 的地址偏移 `0x10000` 当成 Deactivate 寄存器使用。这是 ST 对 GICv2 IP 的客制化——让 Priority Drop 和 Deactivate 可以分开控制。

#### Split EOI and Deactivate：为什么需要两步走

GICv2 允许编程模式将 EOI 和 Deactivate 拆成两步（Split mode）。传统模式下写 `GICC_EOIR` 同时完成两件事：
1. **优先级降级（Priority Drop）**——告诉 GIC 当前中断处理完了，CPU 可以接收更低优先级的中断
2. **停用中断（Deactivate）**——将中断状态从 Active 变回 Inactive，允许该中断再次触发

分开的意义在于**虚拟化场景**：

```
时间轴 →
                    Host 收到中断          Guest 处理完中断
                         │                      │
                         ▼                      ▼
   ┌───────────────────────────────────────────────────────┐
   │ 中断状态: Pending → Active           Active → Inactive   │
   │                  ▲                      ▲              │
   │                  │                      │              │
   │             读 GICC_IAR             写 GICC_DIR        │
   │             （隐式 ack + 置 Active）   （Deactivate）      │
   │                                                       │
   │                  ───── EOI 写 GICC_EOIR ─────          │
   │                （只做 Priority Drop，不做 Deactivate）    │
   └───────────────────────────────────────────────────────┘
```

关键场景：
1. 物理设备产生中断，GIC 将中断标记为 Pending，发给 CPU
2. Host（Hypervisor）读 `GICC_IAR`，状态变为 Active，开始处理
3. Host 写 `GICC_EOIR`（**只 Priority Drop，不 Deactivate**），允许 CPU 继续接收其他中断
4. Host 切回 VM，VM 中的 Guest OS 处理这个虚拟中断
5. Guest 处理完后，Host 写 `GICC_DIR`（**真正的 Deactivate**），中断状态回到 Inactive

如果 EOI 和 Deactivate 不分开，第 3 步写 EOI 时就把中断 deactivate 了——物理中断消失，VM 里看不到中断。

STM32MP2 的客制化 GIC IP 中，GICC 的 `0x10000` 偏移被映射为 Deactivate 寄存器（`GIC_STM32_CPU_DEACTIVATE`）。ST 在驱动中通过 `gic_stm32_gicc_dir_access` 静态分支判断是否使用这个客制化路径（`irq-gic.c`，第 247 行）：

```c
if (static_branch_unlikely(&gic_stm32_gicc_dir_access))
    writel_relaxed(hwirq, gic_cpu_base(d) + GIC_STM32_CPU_DEACTIVATE);
else
    writel_relaxed(hwirq, gic_cpu_base(d) + GIC_CPU_DEACTIVATE);
```

静态分支 `gic_stm32_gicc_dir_access` 在 `gic_of_setup()` 中根据 DTS 信息初始化。支持分段模式的 GIC，`GICC_DIR` 寄存器（偏移 `0x1000`）可用；否则只能写标准 `GICC_EOIR` 同时完成 Priority Drop + Deactivate。

这个功能在 Notion 笔记中被称为"中断处理的分段式管理"——将"可以接收新中断"（Priority Drop）和"关闭本次中断"（Deactivate）在时间上解耦。

**EXTI 的 EOI**（`irq-stm32mp-exti.c`，第 359 行）：

```c
static void stm32mp_exti_eoi(struct irq_data *d)
{
    struct stm32mp_exti_chip_data *chip_data = irq_data_get_irq_chip_data(d);
    const struct stm32mp_exti_bank *bank = chip_data->reg_bank;

    raw_spin_lock(&chip_data->rlock);

    /* 清除 RPR（上升沿 pending）和 FPR（下降沿 pending）*/
    stm32mp_exti_write_bit(d, bank->rpr_ofst);
    stm32mp_exti_write_bit(d, bank->fpr_ofst);

    /* 恢复 IMR（EOI = 清除 pending + 重新使能）*/
    chip_data->mask_cache |= stm32mp_exti_set_bit(d, bank->imr_ofst);

    raw_spin_unlock(&chip_data->rlock);

    irq_chip_eoi_parent(d);   /* 向上调 GIC 的 EOI */
}
```

EXTI 的 EOI 做了三件事：
1. **写 RPR + FPR**：清除边沿检测的 pending 状态（相当于 ack）
2. **写 IMR**：重新使能中断（EXTI 的 EOI 同时包含了 unmask 的语义）
3. **调 parent EOI**：向上传递给 GIC

**所以对 EXTI 来说，EOI 扮演了 "ack + unmask + eoi" 的三个角色。**

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

EXTI 在休眠时如果某个中断配置了唤醒功能，IMR 中对应的位必须保持使能，否则中断信号无法穿透到 GIC。`wake_active` 记录了哪些 EXTI event 配置了唤醒，`stm32mp_exti_suspend()` 中据此决定保留哪些 IMR 位。

### 2.5 irq_chip flags

每个 irq_chip 还有一个 `flags` 字段，指示 core 层"这个芯片有什么特性"。GICv2 的 gic_chip：

```c
static const struct irq_chip gic_chip = {
    ...
    .flags = IRQCHIP_SET_TYPE_MASKED |
             IRQCHIP_SKIP_SET_WAKE |
             IRQCHIP_MASK_ON_SUSPEND,
};
```

| 标志位 | 含义 | GIC 用了？ | EXTI 用了？ |
|--------|------|-----------|------------|
| `IRQCHIP_SET_TYPE_MASKED` | 改触发类型前先 mask 中断（防误触发） | ✅ | ❌ |
| `IRQCHIP_SKIP_SET_WAKE` | 不支持 wake（GIC 自己不能唤醒 SoC） | ✅ | ❌（EXTI 支持唤醒） |
| `IRQCHIP_MASK_ON_SUSPEND` | suspend 时自动 mask 非唤醒中断 | ✅ | ✅（在 suspend 回调中处理） |
| `IRQCHIP_EOI_THREADED` | 线程化 handler 返回后才 EOI | ❌ | ❌ |

`IRQCHIP_SET_TYPE_MASKED` 的含义：在调用 `irq_set_type()` 之前，core 层会先 mask 这个中断。防止切换触发类型的过程中，旧的配置还没清除、新的还没生效时，中间状态产生误中断。

### 2.6 层级域中的 parent 回调链

关键设计：**一个 irq_data 的 chip 不一定只对应一个物理芯片**。

在层级域（hierarchical domain）中，每个 irq_data 通过 `parent_data` 指针连到上一层。当 EXTI 的 chip 调用 `irq_chip_mask_parent(d)` 时，实际调用的是 `d->parent_data->chip->irq_mask(d->parent_data)`——即 GIC 的 mask。

<img src="https://via.placeholder.com/1x1" alt="层级域调用链" />

```
EXTI 层: stm32mp_exti_mask(d)
    │
    ├─ 写 EXTI_IMR（清除对应 bit）
    │
    └─ irq_chip_mask_parent(d)
         │
         ▼ d->parent_data->chip->irq_mask(d->parent_data)
         │
GIC 层:  gic_mask_irq(d->parent_data)
         │
         └─ gic_poke_irq(parent_data, GIC_DIST_ENABLE_CLEAR)
              │
              ▼ 写 GICD_ICENABLER 寄存器
```

这个链式结构是 00-History §5.2 描述的层级域（Hierarchical Domain）在代码层面的实际体现。

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

## 3. irq_domain 层：硬件中断号到 Linux IRQ 号的映射

从 §1 的四个核心问题来看，irq_domain 回答**问题 1：识别——这个中断是哪来的**。

硬件中断控制器（GIC、EXTI）用 **hwirq**（硬件中断号）标识中断。例如 GIC 的 SPI 42 就是 hwirq 74（SPI 从 32 开始编号）。但 Linux 内核用一个全局的 **virq**（虚拟中断号，即 Linux IRQ 号）来标识中断——这是 `request_irq(irq, handler)` 的第一个参数。

**为什么需要翻译层？** 因为两块中断控制器的 hwirq 编号空间是重叠的——GIC 有 hwirq 0~1019，EXTI 也有自己的 0~95。没有 irq_domain，内核无法区分 `hwirq=42` 是 GIC 的 SPI 还是 EXTI 的 event。

### 3.1 irq_domain 结构体全字段解析

`struct irq_domain` 定义在 `include/linux/irqdomain.h`，第 150 行：

```c
struct irq_domain {
    struct list_head            link;           /* 全局 domain 链表节点 */
    const char                  *name;          /* 域名（debugfs 显示用） */
    const struct irq_domain_ops *ops;           /* 操作回调（map/alloc/xlate） */
    void                        *host_data;     /* 控制器私有数据（如 gic_chip_data） */
    unsigned int                flags;          /* 域标志 */
    unsigned int                mapcount;       /* 已映射的中断数 */
    struct mutex                mutex;
    struct irq_domain           *root;          /* 根域指针 */

    /* 可选数据 */
    struct fwnode_handle        *fwnode;         /* 固件节点（DTS 的 device_node） */
    enum irq_domain_bus_token   bus_token;

    struct irq_domain_chip_generic *gc;

#ifdef CONFIG_IRQ_DOMAIN_HIERARCHY
    struct irq_domain           *parent;        /* 父域指针 */
#endif

    /* 反向映射数据 */
    irq_hw_number_t             hwirq_max;      /* 最大 hwirq 号 */
    unsigned int                revmap_size;    /* 线性映射表大小 */
    struct radix_tree_root      revmap_tree;    /* 树映射的红黑树根 */
    struct irq_data __rcu       *revmap[];      /* 线性映射表（柔性数组） */
};
```

各字段的职责：

| 字段 | 类型 | 作用 | 谁填的 |
|------|------|------|--------|
| `link` | `list_head` | 串在全局 `irq_domain_list` 链表中，供 `irq_find_host()` 遍历查找 | core 层创建时自动 |
| `name` | `const char *` | 域名，debugfs 中显示 | 创建时传入的参数 |
| `ops` | `const struct irq_domain_ops *` | 翻译、映射、分配回调 | 驱动提供（如 `stm32mp_exti_domain_ops`） |
| `host_data` | `void *` | 厂商私有数据，`irq_domain_alloc()` 中用 | 驱动提供（如 GIC 指向 `gic_chip_data`） |
| `flags` | `unsigned int` | `IRQ_DOMAIN_FLAG_HIERARCHY` 等 | 创建时传入 |
| `fwnode` | `struct fwnode_handle *` | 关联的 DTS 节点，用于 `irq_find_host(np)` 匹配 | 创建时传入 |
| `parent` | `struct irq_domain *` | 父域指针，层级域的上下级关系 | 创建时传入（非根域） |
| `revmap[]` | `struct irq_data *[]` | 线性映射表，`idx = hwirq` → 直接取 `irq_data *` | core 层管理 |
| `revmap_tree` | `radix_tree_root` | 树映射的红黑树根，用于稀疏 hwirq | core 层管理 |

### 3.2 irq_domain_ops：核心回调

```c
struct irq_domain_ops {
    int (*match)(struct irq_domain *d, struct device_node *node,
                 enum irq_domain_bus_token bus_token);
    int (*select)(struct irq_domain *d, struct irq_fwspec *fwspec,
                  enum irq_domain_bus_token bus_token);
    int (*map)(struct irq_domain *d, unsigned int virq, irq_hw_number_t hw);
    void (*unmap)(struct irq_domain *d, unsigned int virq);
    int (*xlate)(struct irq_domain *d, struct device_node *node,
                 const u32 *intspec, unsigned int intsize,
                 unsigned long *out_hwirq, unsigned int *out_type);
    /* 层级域 V2 接口 */
    int (*alloc)(struct irq_domain *d, unsigned int virq,
                 unsigned int nr_irqs, void *arg);
    void (*free)(struct irq_domain *d, unsigned int virq,
                 unsigned int nr_irqs);
    int (*activate)(struct irq_domain *d, struct irq_data *irqd, bool reserve);
    void (*deactivate)(struct irq_domain *d, struct irq_data *irq_data);
    int (*translate)(struct irq_domain *d, struct irq_fwspec *fwspec,
                     unsigned long *out_hwirq, unsigned int *out_type);
};
```

这些回调的职责清晰分层：

```
DTS 解析阶段:
  xlate / translate — 把 DTS 的 intspec（如 <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>）
                      翻译成 (hwirq, type)

中断分配阶段:
  map  — 传统方式：为单个 hwirq 分配 virq，设置 irq_chip 和 flow handler
  alloc — 层级域方式：分配一组 virq，调 parent 的 alloc 逐层向上（V2 接口）

中断释放阶段:
  unmap / free — 反向清理

匹配阶段:
  match — 用 device_node 匹配 domain（传统）
  select — 用 fwspec 匹配 domain（层级域中用于区分多个 domain）
```

**GIC 的 irq_domain_ops**（`irq-gic.c`，第 1168 行）：

```c
static const struct irq_domain_ops gic_irq_domain_hierarchy_ops = {
    .translate  = gic_irq_domain_translate,
    .alloc      = gic_irq_domain_alloc,
    .free       = irq_domain_free_irqs_top,
};
```

GIC 实现了 `translate` + `alloc` + `free`。`alloc` 内部调 `translate` 解析参数，然后调 `gic_irq_domain_map` 为每个 virq 设置 irq_chip 和 flow handler。

**EXTI 的 irq_domain_ops**（`irq-stm32mp-exti.c`，第 796 行）：

```c
static const struct irq_domain_ops stm32mp_exti_domain_ops = {
    .match  = stm32mp_exti_domain_match,
    .select = stm32mp_exti_domain_select,
    .alloc  = stm32mp_exti_domain_alloc,
    .free   = stm32mp_exti_domain_free,
    .xlate  = irq_domain_xlate_twocell,
};
```

EXTI 使用旧式 `xlate`（而非 `translate`），因为它只解析 DTS 的 `<&exti1 event_id type>` 这种双 cell 绑定。`irq_domain_xlate_twocell` 是内核提供的通用实现。

### 3.3 四种映射方式深度对比

irq_domain 的核心任务是将 hwirq 翻译为 virq。翻译结果需要缓存起来，下次再遇到同一个 hwirq 时直接查到。四种缓存方式：

| 方式 | 数据结构 | 查找复杂度 | 适用场景 | 谁在用 |
|------|---------|-----------|---------|--------|
| **Linear** | 数组 `revmap[hwirq]` | O(1) | hwirq 连续、总数少（<1024） | GIC（~1020 个中断，连续编号） |
| **Tree** | 红黑树 `revmap_tree` | O(log N) | hwirq 范围大且稀疏 | MSI、PCIe 中断（动态编号） |
| **Legacy** | 预分配 virq，查 `irq_map[]` | O(1) 但浪费 | 少量静态 IRQ，旧驱动兼容 | 旧代码（已淘汰） |
| **NoMap** | 无缓存 | N/A | MSI 等不需要反向查找的场景 | MSI 域 |

**线性映射（Linear）** 是最常用的方式。创建时指定 `size`，内核分配 `sizeof(struct irq_data *) * size` 的数组做 `revmap[]`。反向映射时：

```c
/* kernel/irq/irqdomain.c — 线性映射查找 */
static struct irq_data *linear_revmap(struct irq_domain *domain,
                                       irq_hw_number_t hwirq)
{
    return rcu_dereference(domain->revmap[hwirq]);
}
```

就是一个直接的数组下标访问。GIC 用这种方式：

```c
/* irq-gic.c 第 1221 行 */
gic->domain = irq_domain_create_linear(handle, gic_irqs,
                                       &gic_irq_domain_hierarchy_ops, gic);
```

`gic_irqs` 是通过 `GIC_DIST_CTR` 寄存器查到的实际硬件支持的中断数（最大 1020）。

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

## 4. irq_desc 层：中断描述符与 Flow Control

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

flow handler 是挂在 `irq_desc->handle_irq` 上的函数指针。它决定了**中断从硬件触发到软件 handler 执行之间，mask/ack/eoi/unmask 这些操作的顺序**。

五种标准 flow handler（都在 `kernel/irq/chip.c` 中）：

| Handler | mask 时机 | ack 时机 | unmask 时机 | EOI | 适用场景 |
|---------|----------|---------|------------|-----|---------|
| **`handle_level_irq`** | 入口即 mask | 入口即 ack | handler 返回后（cond_unmask） | 不需要 | 电平触发（需保持 mask 防重复触发） |
| **`handle_edge_irq`** | 入口即 mask | 入口即 ack（芯片的 irq_ack） | 无 pending 且有 action 时 unmask | 不需要 | 边沿触发（防丢失边沿） |
| **`handle_fasteoi_irq`** | 从不 mask（特殊情况除外） | 不 ack（硬件自动） | 不 unmask | 入口调 irq_eoi | GIC 等硬件自动管理的控制器；**STM32MP257 上 GIC 和 EXTI 的所有中断都走此 handler** |
| **`handle_simple_irq`** | 不 mask | 不 ack | 不 unmask | 不需要 | 非标准/简单控制器 |
| **`handle_percpu_irq`** | 不 mask | 可能不 ack | 不 unmask | 入口 EOI | 每个 CPU 独享中断 |

#### handle_level_irq（kernel/irq/chip.c，第 628 行）

```c
void handle_level_irq(struct irq_desc *desc)
{
    raw_spin_lock(&desc->lock);
    mask_ack_irq(desc);             /* ① 立即 mask + ack */

    if (!irq_may_run(desc))         /* ② 检查是否可运行 */
        goto out_unlock;

    desc->istate &= ~(IRQS_REPLAY | IRQS_WAITING);

    if (unlikely(!desc->action || irqd_irq_disabled(&desc->irq_data))) {
        desc->istate |= IRQS_PENDING;
        goto out_unlock;            /* ③ 无 handler → 保持 mask，标记 pending */
    }

    kstat_incr_irqs_this_cpu(desc);
    handle_irq_event(desc);         /* ④ 执行 action 链表的 handler */

    cond_unmask_irq(desc);          /* ⑤ handler 完成后 unmask */

out_unlock:
    raw_spin_unlock(&desc->lock);
}
```

电平中断的核心问题：**只要设备保持电平为高，GIC 就会一直看到 pending。** 如果不 mask，`handle_irq_event()` 一返回，中断立即再次触发——形成死循环。所以 `handle_level_irq` 在第一步就 mask 了中断，直到 handler 执行完再 unmask。

时序（以 STM32MP2 的 EXTI level 中断为例）：

```
时间轴 →
                    ① mask_ack_irq()               ⑤ cond_unmask_irq()
                         │                               │
                         ▼                               ▼
   ┌───────────────────────────────────────────────────────────┐
   │  中断信号                                                          │
   │  ┌─────────────────────────────────────────────────────┐       │
   │  │  设备保持电平为高             │       │
   │  └─────────────────────────────────────────────────────┘       │
   │                                                                    │
   │  EXTI_IMR ────┐                                       ──── │
   │               │                                            │
   │  mask=True    │     mask=False                              │
   │               │                                            │
   │  handle_irq_event()                                         │
   │  ┌─────────────────────────────┐                           │
   │  │  action->handler() 执行      │                           │
   │  └─────────────────────────────┘                           │
   └───────────────────────────────────────────────────────────┘
```

#### handle_edge_irq（kernel/irq/chip.c，第 787 行）

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

    desc->irq_data.chip->irq_ack(&desc->irq_data);   /* ① 先 ack（不等事件结束）*/

    do {
        if (unlikely(!desc->action)) {
            mask_irq(desc);
            goto out_unlock;
        }

        if (unlikely(desc->istate & IRQS_PENDING)) {   /* ② 处理期间新中断到了？ */
            if (!irqd_irq_disabled(&desc->irq_data) &&
                irqd_irq_masked(&desc->irq_data))
                unmask_irq(desc);                       /* ③ unmask 重新使能 */
        }

        handle_irq_event(desc);                         /* ④ 执行 handler */

    } while ((desc->istate & IRQS_PENDING) &&           /* ⑤ 还有 pending？继续循环 */
             !irqd_irq_disabled(&desc->irq_data));

out_unlock:
    raw_spin_unlock(&desc->lock);
}
```

边沿中断 vs 电平中断的关键区别：

| 特性 | level | edge |
|------|-------|------|
| mask 时机 | 入口立即 mask（防重复） | 入口 mask（防丢失边沿） |
| ack 时机 | 在 mask_ack_irq 中统一步骤 | 在 `irq_ack()` 中单独一步 |
| ack 后中断信号 | 仍然保持有效（电平没变） | 马上消失（边沿已过） |
| 新中断到达 | 不检测（保持 mask） | 检查 `IRQS_PENDING` 并重入 |
| unmask 条件 | handler 返回后 cond_unmask | pending 位清除或无 pending |

边沿中断的 do-while 循环很关键：handler 执行期间新的边沿来了，GIC 会把它记录到 pending 位。`handle_edge_irq` 检测到 pending 后，unmask 再跑一轮 handler——**不会丢边沿**。

#### handle_fasteoi_irq（kernel/irq/chip.c，第 687 行）

```c
void handle_fasteoi_irq(struct irq_desc *desc)
{
    struct irq_chip *chip = desc->irq_data.chip;

    raw_spin_lock(&desc->lock);

    if (!irq_may_run(desc)) {
        if (irqd_needs_resend_when_in_progress(&desc->irq_data))
            desc->istate |= IRQS_PENDING;
        goto out;                   /* ① 不可运行 → 发 EOI 返回 */
    }

    desc->istate &= ~(IRQS_REPLAY | IRQS_WAITING);

    if (unlikely(!desc->action || irqd_irq_disabled(&desc->irq_data))) {
        desc->istate |= IRQS_PENDING;
        mask_irq(desc);             /* ② 无 handler → mask + EOI */
        goto out;
    }

    kstat_incr_irqs_this_cpu(desc);
    if (desc->istate & IRQS_ONESHOT)
        mask_irq(desc);             /* ③ ONESHOT 模式 → mask（防止中断风暴） */

    handle_irq_event(desc);         /* ④ 执行 handler */

    cond_unmask_eoi_irq(desc, chip); /* ⑤ unmask（如果需要） + EOI */

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

"fasteoi" 的名字从何而来？传统 GICv1 需要软件 mask + ack 再 unmask，GICv2 引入一个寄存器 `GICC_EOIR`——只需写一次 EOI，硬件自己处理剩下的。所以叫"快 EOI"。

`handle_fasteoi_irq` 的特性：
- **不主动 mask**：因为 GIC 硬件自动管理 pending/active 状态
- **不手动 ack**：因为 ack 在入口读 IAR 时已经隐式完成
- **入口只做 EOI**：在 `out` 路径（不可运行）调 `chip->irq_eoi()`，降低优先级
- **ONESHOT 才 mask**：只有线程化中断（`IRQF_ONESHOT`）才需要 mask 防止中断风暴

**GIC 的 SPI 全部使用 `handle_fasteoi_irq`**，这就是 GICv2 作为"透明控制器"的特点——硬件自动管理流程。

#### handle_simple_irq（kernel/irq/chip.c，第 538 行）

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

最简单的 handler——不 mask、不 ack、不 EOI。适用于那些"不需要 flow control"的中断控制器（如一些 SoC 内部的中断聚合器）。

#### handle_percpu_devid_irq（kernel/irq/chip.c，第 924 行）

```c
void handle_percpu_devid_irq(struct irq_desc *desc)
{
    struct irq_chip *chip = irq_desc_get_chip(desc);
    struct irqaction *action = desc->action;
    ...
    if (chip->irq_eoi)
        chip->irq_eoi(&desc->irq_data);
    ...
    __handle_irq_event_percpu(desc);
    ...
}
```

这种 handler 不拿锁（因为 per-CPU 中断只到指定的 CPU，不会冲突），不需要 mask（每个核独享，没有竞争）。GIC 的 PPI 中断（IRQ 16~31）使用此 handler。

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

**注意：EXTI 的中断也走 `handle_fasteoi_irq`，不是 `handle_level_irq` 或 `handle_edge_irq`。**

原因在于层级域的 alloc 时序：EXTI domain 的 `alloc` 只设 chip（`irq_domain_set_hwirq_and_chip`），然后调 `irq_domain_alloc_irqs_parent()` 进入 GIC domain 的 alloc。GIC 的 `gic_irq_domain_map()` 调用 `irq_domain_set_info()` 时，**覆盖了 `desc->handle_irq`**，设为 `handle_fasteoi_irq`。最终生效的是最后设的那一个——GIC 设的 `handle_fasteoi_irq`。

所以无论 DTS 中给 EXTI 配了什么触发类型，**运行时 `desc->handle_irq` 都是 `handle_fasteoi_irq`**。那 EXTI 的边沿/电平差异谁来管？**EXTI 的 chip 回调自身**：`stm32mp_exti_set_type()` 根据 DTS 配 RTSR/FTSR，`stm32mp_exti_eoi()` 清 RPR/FPR 并恢复 IMR。EXTI 边沿检测的结果通过输出电平高传给 GIC，GIC 全部当 level high 处理。

`handle_fasteoi_irq` 对 EXTI 也能工作的原因：它不主动 mask，只调 chip->irq_eoi()。EXTI 的 eoi 清除了 RPR/FPR（pending 位），EXTI 输出电平回低。GIC 看到电平变低，知道中断结束。整个过程在 fasteoi 框架中天然成立。

实际执行路径是一个"双层"的 flow control：

```
外设触发中断
    │
    ▼
EXTI 检测 RTSR/FTSR → 设置 pending (RPR/FPR)
    │
    ▼
EXTI 输出电平高给 GIC
    │
    ▼
GIC 检测到电平高 → Distributor 设置 pending
    │
    ▼
CPU 读 GICC_IAR（隐式 ack）
    │
    ▼
GIC 的 handle_fasteoi_irq 被调用
    │  ├─ 不 mask（GIC 硬件管理）
    │  └─ 调 handle_irq_event()
    │       └─ 调 action->handler()
    │            │
    │            ▼
    │       handler 返回
    │
    ▼
cond_unmask_eoi_irq() → 调 chip->irq_eoi()
    │
    ▼
stm32mp_exti_eoi()
    ├─ 清除 RPR/FPR（ack EXTI）
    ├─ 恢复 IMR（unmask EXTI）
    └─ irq_chip_eoi_parent() → gic_eoi_irq（写 GICC_EOIR）
```

如果此处需要继续追踪更长的中断处理链（包含 `irq_may_run`、`handle_irq_event`、`__handle_irq_event_percpu` 等内部函数），那是 03-Source-Analysis.md 的内容，本章只到结构体层面为止。

---

## 5. irqaction 层：设备驱动的中断注册入口

从 §1 的四个核心问题来看，irqaction 位于**分发**环节的最末端——它的 `handler` 函数就是最终被调用的设备驱动中断处理函数。

irqaction 层是中断子系统中**最靠近设备驱动**的部分。它不像 irq_chip 那样操作硬件，也不像 irq_domain 那样做编号翻译。它的职责只有一个：**把设备驱动的 handler 挂到 irq_desc 上，在中断发生时被调用**。

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
    unsigned long       thread_flags;    /* 线程状态标志 */
    unsigned long       thread_mask;     /* 共享中断时线程唤醒位掩码 */
    const char          *name;           /* 设备名（/proc/interrupts 显示）*/
    struct proc_dir_entry *dir;          /* /proc/irq/NN/name 目录项 */
};
```

| 字段 | 作用 | 谁使用 |
|------|------|--------|
| `handler` | **顶半部回调**——运行在中断上下文中（IRQ 关闭） | flow handler 调 `handle_irq_event()` 时调用 |
| `thread_fn` | **底半部回调**——运行在内核线程中（IRQ 打开） | 线程化中断，由 `irq_thread()` 调用 |
| `thread` | 内核线程的 `task_struct *` | 线程化中断创建时由 `kthread_create()` 初始化 |
| `dev_id` | **共享中断的区分标识**——`free_irq()` 和 `synchronize_irq()` 用它匹配 | 设备驱动传入，core 层原样保存 |
| `next` | 链表指针——多个设备可以共享同一个物理中断线 | 共享中断注册时插入 |
| `flags` | IRQF_* 标志（见下一节） | 注册时传入，flow handler 检查 |
| `irq` | Linux IRQ 号，对应哪个 irq_desc | request_irq 时传入 |

### 5.2 irqaction 的注册与销毁

设备驱动通过 `request_irq()` 或 `request_threaded_irq()` 注册中断 handler。`request_irq()` 是 `request_threaded_irq(irq, handler, NULL, flags, name, dev)` 的封装——`thread_fn = NULL` 表示纯顶半部。

注册流程（数据结构层面的流程，不是逐行代码）：

```
request_threaded_irq(irq, handler, thread_fn, flags, name, dev_id)
    │
    └─ __setup_irq(irq, desc, new_action)    ← 核心函数
         │
         ├─ ① 根据 handler/thread_fn 设置 action 类型
         │
         ├─ ② IRQF_SHARED 检查
         │    └─ 检查已有 handler 的触发类型是否兼容
         │    └─ 检查硬件是否支持共享
         │
         ├─ ③ 将 action 插入 desc->action 链表
         │    └─ 如果 desc->action == NULL → 新建链表
         │    └─ 如果已存在 → 追加到链表末尾（共享中断）
         │
         ├─ ④ 调用 irq_chip 的设置
         │    └─ irqd_set_trigger_type() — 根据 flags 设置触发类型
         │    └─ 可能调 chip->irq_set_type()
         │
         ├─ ⑤ 创建线程（如果 thread_fn 不为 NULL）
         │    └─ kthread_create(irq_thread, action, "irq/%d-%s", irq, name)
         │    └─ action->thread = kthread 的 task_struct
         │
         └─ ⑥ 启用中断
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
#### GIC 中断的四种状态

GICv2 规范定义了中断的四种硬件状态，它们在 Distributor 的寄存器中通过 Active 和 Pending 位组合表示：

| 状态 | Pending 位 | Active 位 | 含义 |
|------|-----------|----------|------|
| **Inactive（非活动）** | 0 | 0 | 未触发，中断线空闲 |
| **Pending（挂起）** | 1 | 0 | 中断已触发，等待 CPU 处理 |
| **Active（活动）** | 0 | 1 | CPU 正在处理该中断 |
| **Active and Pending（活动且挂起）** | 1 | 1 | CPU 正在处理，同一中断源再次触发 |

四种状态的转换路径如下：

```
                   外设触发中断
                         │
                         ▼
     ┌──────────┐    读 GICC_IAR    ┌──────────┐
     │ Pending  │ ─────────────►   │  Active  │
     │          │                  │          │
     └────┬─────┘                  └────┬─────┘
          │                             │
          │ 外设在处理期间            │ 写 GICC_EOIR
          │ 再次触发                  │ （或 GICC_DIR）
          ▼                             ▼
     ┌──────────┐                  ┌──────────┐
     │ Active & │                  │ Inactive │
     │ Pending  │                  │          │
     └──────────┘                  └──────────┘
```

四种状态的核心价值在于 **Active and Pending** 状态。它解决了电平/边沿中断的一个本质问题：**CPU 正在处理一个中断时，同一中断源再次触发怎么办？**

- 对电平中断，外设保持电平为高，GIC 看到 pending 位本来就是 1。但 Active 位也是 1（因为 CPU 正在处理），所以状态是 Active and Pending。等 CPU 完成 EOI/Deactivate 后，状态回到 Pending，中断立即再次触发——这就是 `handle_level_irq` 必须在入口 mask 的原因，否则死循环

- 对边沿中断，第二个边沿使 pending 位再次置 1（Active 位仍为 1），状态进入 Active and Pending。`handle_edge_irq` 在处理完一轮后检查 pending 位，有 pending 就再跑一轮 handler——不会丢边沿

用户可以通 `irq_chip` 的 `irq_get_irqchip_state` 回调读取任一种状态：

```c
/* irq-gic.c，第 279 行 */
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

`GIC_DIST_PENDING_SET`（GICD_ISPENDR）和 `GIC_DIST_ACTIVE_SET`（GICD_ISACTIVER）分别对应 Pending 和 Active 位的读取。

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

中断子系统的核心数据结构可以用四层模型来概括：

| 层 | 数据结构 | 职责 | 类比 |
|----|---------|------|------|
| **irq_chip** | `struct irq_chip` | 封装硬件差异（mask/unmask/ack/eoi/set_type） | 汽车的内燃机——怎么产生动力 |
| **irq_domain** | `struct irq_domain` | 翻译编号（hwirq → virq） | 导航地图——哪个按钮对应哪个功能 |
| **irq_desc** | `struct irq_desc` | 中断描述符 + 流程控制 | 变速箱——什么时候做什么操作 |
| **irqaction** | `struct irqaction` | 连接设备驱动 handler | 方向盘——最终操作的人 |

每一层解决一个核心问题：

1. **irq_domain**：中断来了是哪个（hwirq → virq 映射）
2. **irq_chip**：怎么开关和确认这个中断（mask/unmask/ack/eoi）
3. **irq_desc**：什么时机做开关确认（flow handler 的时序）
4. **irqaction**：谁来做最终处理（设备驱动的 handler）

这四个问题不可分割但必须分开实现。分开的收益：

- 换 GIC 到 GICv3 时，只需要改 irq_chip 的实现（`irq-gic-v3.c`），flow handler 的 logic 不变
- 增加 EXTI 这样的新控制器时，只需要写新的 irq_domain 和 irq_chip，挂到 GIC 的 domain 下
- 驱动开发者不需要关心底下有几层控制器，只需要 `request_irq(irq, handler)`——层级之间通过 `parent_data` 链自动传递

下一章 03-Source-Analysis.md 将沿着具体代码路径，逐行走一遍 probe、request_irq、中断触发、flow handler 执行、threaded handler 唤醒的完整流程。

---

*本章完 · 下一章：03-Source-Analysis.md — 中断源码分析*

