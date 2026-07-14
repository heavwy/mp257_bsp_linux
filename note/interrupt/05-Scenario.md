# 05. 运行时情景分析

> 本文是 STM32MP257 中断子系统深度分析系列的最后一篇。
> 用两个真实场景——gpio-keys KEY0 按键和 I2C 触摸屏中断——完整追踪
> 从硬件电平变化到用户态可见的中断全路径。
>
> **前置:** [04-BottomHalf.md](04-BottomHalf.md) — 下半部四种机制分析
> **下一篇:** 无（系列终篇）
>
> **字数：约 18,000 字**
> **建议阅读时间：60 分钟**

---

## 5.1 概述

### 5.1.1 与前面四篇的分工

前面的四篇分别覆盖了中断子系统的四个侧面：

| 篇目 | 覆盖内容 | 与 05 的关系 |
|------|---------|------------|
| **00-History** | 30 年演进史，每个机制解决什么问题 | 回答"为什么设计成这样" |
| **01-Usage** | DTS 编写、API 使用、调试接口 | 本篇的场景 DTS 和调试手段引用自此 |
| **02-Architecture** | 核心数据结构：irq_desc、irq_domain、irq_chip、irqaction | 本篇涉及的数据结构不再重复定义 |
| **03-SourceAnalysis** | 初始化流程：GIC→EXTI→GPIO domain→request_irq | 本篇从"全部就绪后"开始 |
| **04-BottomHalf** | 四种下半部机制的数据结构与调度路径 | 本篇引用其 workqueue/threaded IRQ 机制，不重复原理 |
| **05-Scenario** | ★ 完整运行时路径：硬件→用户态 | **当前本文** |

**本文的处理原则：**

1. **路径全过程追踪**，从电平变化到 `/dev/input/event0`
2. 02/03/04 已覆盖的数据结构、机制原理不再解释——只引用章节号
3. 每个断面（cross-section）给出 **源码函数名 + 文件行号**（符合源码唯一原则）
4. 涉及 EXTI/GIC 寄存器的部分给出 **具体的寄存器和位偏移**

### 5.1.2 两幕速览

| 幕数 | 场景 | 设备 | DTS 路径 | 下半部机制 | 用户态接口 |
|------|------|------|---------|-----------|-----------|
| **第一幕** | 按键 KEY0 按下 | gpio-keys | GPIO PH5 → EXTI → GIC | **workqueue**（`mod_delayed_work`） | `evtest KEY` |
| **第二幕** | 触摸屏点击 | FT6x36 (I2C) | I2C2 → GIC（直连） | **线程化 IRQ**（`IRQ_WAKE_THREAD`） | `evtest ABS_*` |

### 5.1.3 两幕的全路径分叉点

两幕从硬件到 GIC 到流控函数的路径**完全相同**。关键分叉点在顶半部 handler 的返回值：

```
硬件触发 → EXTI → GIC → CPU(vbar) → entry.S → gic_handle_irq
                                                              │
                                                              ▼
                                                  handle_fasteoi_irq
                                                              │
                                                              ▼
                                                  handle_irq_event
                                                              │
                                                              ▼
                                                  action->handler()
                                                              │
                                          ┌─────────────────────┐
                                          │                     │
                                    return IRQ_HANDLED    return IRQ_WAKE_THREAD
                                          │                     │
                                          ▼                     ▼
                                    mod_delayed_work      __irq_wake_thread
                                          │                     │
                                          ▼                     ▼
                                  (时间解耦) worker      irq_thread 立即调度
                                  线程调度执行                 │
                                          │           i2c_transfer (sleep)
                                  input_report_key      input_report_abs
                                          │                     │
                                          ▼                     ▼
                                      evtest /dev/input/event0
```

从 `gic_handle_irq` 到 `handle_fasteoi_irq` 到 `handle_irq_event` 到 `action->handler()`——这是两幕**共享的中断前半段**。我们在第一幕中详细走通前半段全路径，第二幕重点放在后半段的分叉和差异上。

---

## 第一幕：gpio-keys KEY0 按下（workqueue 路径）

### 5.2 场景设定

#### 5.2.1 DTS 配置

KEY0 在 ATK 板的 DTS 中定义如下（路径基于 stm32mp257d-atk-bsp.dts）：

```dts
gpio-keys {
    compatible = "gpio-keys";
    /*
     * pinctrl 子系统在 probe 前将 PH5 配置为 GPIO 输入模式
     * （非 AF，由 gpio-keys 驱动直接读取 GPIO 寄存器）
     */
    pinctrl-names = "default";
    pinctrl-0 = <&key0_pins>;

    key0 {
        label = "KEY0";
        linux,code = <KEY_0>;
        gpios = <&gpioh 5 GPIO_ACTIVE_LOW>;
        /* debounce-interval 传递给 bdata->software_debounce */
        debounce-interval = <10>;
    };
};
```

KEY0 对应的 GPIO pin 是 **PH5**（GPIOH bank 的第 5 脚）。中断信号路径：

```
PH5 按下（低电平有效）
    → GPIOH bank 检测到电平变化（配置为 GPIO 输入，polling by EXTI）
    → EXTI 边沿检测（RTSR/FTSR 配置的边沿）
    → EXTI 向 GIC 发送 SPI 中断信号
    → GIC Distributor → CPU Interface → CPU IRQ 引脚
```

PH5 对应的 EXTI event 编号和 GIC SPI 编号由 SoC 硬件连接决定。stm32mp251.dtsi 中 EXTI 到 GIC 的路由：

```dts
/* stm32mp251.dtsi — EXTI 节点的 interrupts 属性定义了每个 EXTI event 对应的 GIC SPI */
exti1: interrupt-controller@44220000 {
    compatible = "st,stm32mp1-exti";
    reg = <0x44220000 0x400>;
    interrupts-extended =
        <&intc GIC_SPI 268 IRQ_TYPE_LEVEL_HIGH>,   /* event 0 */
        <&intc GIC_SPI 269 IRQ_TYPE_LEVEL_HIGH>,   /* event 1 */
        ...
        <&intc GIC_SPI 273 IRQ_TYPE_LEVEL_HIGH>,   /* event 5 ← PH5 在此 */
        ...
};
```

GPIOH 的 `.to_irq()` 回调将 PH5 映射为 EXTI event 5（通过 pinctrl 的 `stm32_gpio_to_irq`），而 EXTI event 5 对应 GIC SPI 273。所以最终的中断编号链是：

```
PH5 → GPIOH domain hwirq=5 → EXTI domain hwirq=5 → GIC domain hwirq=273
```

#### 5.2.2 request_irq 的注册回顾

gpio-keys 驱动在 probe() 中为每个按键调用 `gpiod_to_irq()` 和 `devm_request_irq()`，详见 03-§3.6。注册完成后，`/proc/interrupts` 中可见：

```
~# cat /proc/interrupts
           CPU0
268:          0  stm32mp_exti   5  Edge      gpio-keys
```

其中 **virq=268**（Linux IRQ 号），hwirq=5（EXTI event），由 `irq_create_of_mapping` 分配。

```c
// gpio_keys.c:592 — 初始化 workqueue（probe 阶段）
INIT_DELAYED_WORK(&bdata->work, gpio_keys_gpio_work_func);

// gpio_keys.c:598 — 注册中断，handler = gpio_keys_gpio_isr
isr = gpio_keys_gpio_isr;
irqflags = IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING;
```

probe 阶段完成后，**硬件和软件状态**（来自 03-§3.7 的完整就绪图）：

| 组件 | 状态 |
|------|------|
| GICD_CTLR | 1 (Distributor 使能) |
| GICC_CTLR | 1 (CPU Interface 使能) |
| GICC_PMR | 0xF0 (优先级阈值，允许所有中断) |
| EXTI IMR[5] | 1 (PH5 中断未被屏蔽) |
| GICD_ISENABLER[spi=273] | 1 (SPI 使能) |
| irq_desc[268].handle_irq | `handle_fasteoi_irq` |
| irq_desc[268].action->handler | `gpio_keys_gpio_isr` |
| irq_desc[268].action->thread_fn | NULL (不是线程化 IRQ) |

**现在一切就绪。等待 PH5 被按下。**

---

### 5.3 断面 1.1：硬件层——GPIO PH5 → EXTI → GIC

#### 5.3.1 PH5 按下发生了什么

ATK 板上的 KEY0 是**低电平有效**的按键。按下前 PH5 被外部上拉电阻拉到高电平。按下后 PH5 被拉到 GND（低电平）。GPIOH bank 检测到这个电平变化。

GPIO bank 在 STM32MP257 中通过硬件连线连接到 EXTI 控制器。PH5 的 EXTI event 编号为 5（由 SoC 设计决定，在 pinctrl 的 `stm32_gpio_to_irq` 中编码为 GPIO pin 索引）。

EXTI 控制器检测到 PH5 的边沿跳变后：

1. **RPR（Rising Pending Register）或 FPR（Falling Pending Register）置位**——记录中断事件已发生
2. **IMR（Interrupt Mask Register）检查**——如果对应的 bit 为 1（未屏蔽），允许中断信号继续传递
3. **TRG（Trigger Register）决定 EOI 行为**

EXTI 寄存器的物理基址是 `0x44220000`（来自 `stm32mp251.dtsi`）。三个 bank 的寄存器偏移：

| Bank | IMR 偏移 | RPR 偏移 | FPR 偏移 | 覆盖 event 范围 |
|------|---------|---------|---------|----------------|
| Bank 1 | 0x80 | 0x0C | 0x10 | event 0–31 |
| Bank 2 | 0x90 | 0x2C | 0x30 | event 32–63 |
| Bank 3 | 0xA0 | 0x4C | 0x50 | event 64–95 |

PH5 的 event 5 落在 **Bank 1**（event 0–31）。IMR1 的 bit 5 用于屏蔽/使能，RPR1 的 bit 5 在上升沿置位，FPR1 的 bit 5 在下降沿置位。

```c
// irq-stm32mp-exti.c:93 — Bank 1 寄存器定义
static const struct stm32mp_exti_bank stm32mp_exti_b1 = {
    .imr_ofst   = 0x80,
    .rtsr_ofst  = 0x00,
    .ftsr_ofst  = 0x04,
    .rpr_ofst   = 0x0C,
    .fpr_ofst   = 0x10,
    .trg_ofst   = 0x3EC,
};
```

按下 KEY0（下降沿）时，硬件自动将 `EXTI_FPR1` 的 bit 5 置 1。如果 `EXTI_IMR1` 的 bit 5 = 1（未被屏蔽），EXTI 向 GIC 发送中断信号。

#### 5.3.2 EXTI 到 GIC 的信号传递

EXTI 通过硬件连线将每个 event 映射为一个 GIC SPI。与 PH5 对应的 GIC SPI ID 是 **273**（从上面 DTS 可知 event 5 对应 GIC SPI 273）。

GIC-400 收到 SPI 273 的中断信号后：

1. **Distributor（GICD）检查**：GICD_ISENABLER 中 SPI 273 对应的 bit 是否为 1（使能）
2. **优先级比较**：SPI 273 的优先级（在 GICD_IPRIORITYR 中配置）是否高于 GICC_PMR 中设置的阈值
3. **目标 CPU 检查**：SPI 273 的 affinity（在 GICD_ITARGETSR 中配置）是否包含当前 CPU
4. **转发**：如果三个条件都满足，Distributor 将中断转发给 CPU Interface

GIC-400 的寄存器基址在 `0x4ac00000`（Distributor）和 `0x4ac10000`（CPU Interface）。关键的 CPU Interface 寄存器：

| 寄存器 | 偏移 | 用途 |
|--------|------|------|
| GICC_IAR | 0x0C | 读取中断 ID（ACK 中断） |
| GICC_EOI | 0x10 | 标记中断处理完成 |
| GICC_PMR | 0x04 | 优先级掩码 |

CPU Interface 收到中断后，向 Cortex-A35 核的 nIRQ 引脚断言一个 IRQ 信号。CPU 在每条指令执行完后采样 nIRQ 引脚——如果为有效电平，进入 IRQ 处理模式。

---

### 5.4 断面 1.2：ARM64 异常入口——entry.S

#### 5.4.1 从 VBAR_EL1 到 vectors

Cortex-A35 在 EL1 模式下运行时，异常向量表基址由 `VBAR_EL1` 寄存器指定。系统启动时 `head.S` 将 `entry.S` 中的 `vectors` 标签写入 VBAR_EL1。

当 CPU 检测到 nIRQ 引脚有效时，自动完成三件事：

1. 将返回地址（当前 PC）保存到 `ELR_EL1`
2. 将当前 PSTATE 保存到 `SPSR_EL1`
3. 根据异常类型查 `VBAR_EL1` 指向的向量表

异常向量表在 `entry.S:520`：

```asm
// arch/arm64/kernel/entry.S:520 — 异常向量表
SYM_CODE_START(vectors)
    kernel_ventry   1, t, 64, sync      // Synchronous EL1t
    kernel_ventry   1, t, 64, irq       // IRQ EL1t
    kernel_ventry   1, t, 64, fiq       // FIQ EL1t
    kernel_ventry   1, t, 64, error     // Error EL1t

    kernel_ventry   1, h, 64, irq       // ← ★ IRQ EL1h (我们的路径)
    kernel_ventry   1, h, 64, sync      // Synchronous EL1h
    kernel_ventry   1, h, 64, fiq       // FIQ EL1h
    kernel_ventry   1, h, 64, error     // Error EL1h
    ...
```

表中共 16 个条目（4 异常类型 × 4 来源模式）。我们的中断来自 **EL1h 模式**（kernel 在 EL1 运行，使用 SP_EL1 栈指针），所以对应的条目是第 5 个——`kernel_ventry 1, h, 64, irq`。

#### 5.4.2 kernel_ventry 宏——入口前置

`kernel_ventry` 宏（`entry.S:38`）做三件事：

```asm
// arch/arm64/kernel/entry.S:38
.macro kernel_ventry, el:req, ht:req, regsize:req, label:req
    .align 7                        // 128 字节对齐
.Lventry_start\@:
    sub sp, sp, #PT_REGS_SIZE       // ★ 在栈上分配 pt_regs 空间 (288 字节)
    ...
    b   el\el\ht\()_\regsize\()_\label    // 跳转到 el1h_64_irq
.org .Lventry_start\@ + 128         // 不超过 128 字节
.endm
```

`PT_REGS_SIZE`（`struct pt_regs` 的大小）是 **288 字节**（36 个 64 位寄存器 × 8）。`sub sp, sp, #PT_REGS_SIZE` 在栈顶分配 pt_regs 结构体的空间——中断 handler 的寄存器现场马上要保存到这里。

#### 5.4.3 el1h_64_irq 入口处理

跳转到 `el1h_64_irq` 后，执行的是 `entry_handler` 宏生成的代码：

```asm
// arch/arm64/kernel/entry.S:573 — entry_handler 宏
.macro entry_handler el:req, ht:req, regsize:req, label:req
SYM_CODE_START_LOCAL(el\el\ht\()_\regsize\()_\label)
    kernel_entry \el, \regsize       // ★ 保存所有通用寄存器到 pt_regs
    mov x0, sp                       // x0 = pt_regs 指针（第一个参数）
    bl  el\el\ht\()_\regsize\()_\label\()_handler  // 调 C 函数
    .if \el == 0
    b   ret_to_user
    .else
    b   ret_to_kernel                 // ★ EL1 中断返回走这里
    .endif
SYM_CODE_END(el\el\ht\()_\regsize\()_\label)
.endm

// arch/arm64/kernel/entry.S:595 — 实际生成 el1h_64_irq 符号
entry_handler   1, h, 64, irq
```

所以流程清晰了：

```
el1h_64_irq:
    kernel_entry 1, 64    ← 保存现场
    mov x0, sp            ← pt_regs 指针作为参数
    bl el1h_64_irq_handler ← C 函数
    b ret_to_kernel       ← 返回路径
```

#### 5.4.4 kernel_entry——保存现场

`kernel_entry` 宏（`entry.S:198`）的主要工作就是**将所有通用寄存器压入栈**，构成 `struct pt_regs`：

```asm
// arch/arm64/kernel/entry.S:198
.macro kernel_entry, el, regsize=64
    stp x0, x1, [sp, #16 * 0]
    stp x2, x3, [sp, #16 * 1]
    ...
    stp x26, x27, [sp, #16 * 13]
    stp x28, x29, [sp, #16 * 14]

    // 对于 EL1 中断：
    add x21, sp, #PT_REGS_SIZE        // x21 = 中断发生前的 SP 值
    get_current_task tsk               // tsk (x28) = current

    mrs x22, elr_el1                  // x22 = 返回地址（被打断的 PC）
    mrs x23, spsr_el1                 // x23 = 被打断的 PSTATE
    stp lr, x21, [sp, #S_LR]          // 保存 LR 和 SP
    stp x29, x22, [sp, #S_STACKFRAME] // 保存 FP 和 PC（回溯链）
    add x29, sp, #S_STACKFRAME
    stp x22, x23, [sp, #S_PC]         // 保存 PC 和 PSTATE（中断返回用）
.endm
```

中断返回时要靠 `pt_regs` 中的 PC（`elr_el1`）和 PSTATE（`spsr_el1`）恢复执行现场——用 `eret` 指令从 `ELR_EL1` + `SPSR_EL1` 恢复。这也是为什么中断 handler 绝对不能破坏栈上 pt_regs 的原因。

#### 5.4.5 el1h_64_irq_handler——C 函数入口

`el1h_64_irq_handler` 是 C 函数，在 `arch/arm64/kernel/entry-common.c:520`：

```c
// arch/arm64/kernel/entry-common.c:520
asmlinkage void noinstr el1h_64_irq_handler(struct pt_regs *regs)
{
    el1_interrupt(regs, handle_arch_irq);
}

// 同文件:509 — el1_interrupt
static void noinstr el1_interrupt(struct pt_regs *regs,
                  void (*handler)(struct pt_regs *))
{
    write_sysreg(DAIF_PROCCTX_NOIRQ, daif);

    if (IS_ENABLED(CONFIG_ARM64_PSEUDO_NMI) && !interrupts_enabled(regs))
        __el1_pnmi(regs, handler);
    else
        __el1_irq(regs, handler);       // ← 正常中断路径
}

// 同文件:496 — __el1_irq：真正的核心
static __always_inline void __el1_irq(struct pt_regs *regs,
                      void (*handler)(struct pt_regs *))
{
    enter_from_kernel_mode(regs);

    irq_enter_rcu();                    // ★ 标记进入硬中断上下文
    do_interrupt_handler(regs, handler); // ★ 调 gic_handle_irq
    irq_exit_rcu();                     // ★ 退出，检查 softirq

    arm64_preempt_schedule_irq();       // 中断退出后可抢占检查

    exit_to_kernel_mode(regs);
}
```

`irq_enter_rcu()` 和 `irq_exit_rcu()` 是**下半部分水岭**的第一道门——前者递增 `preempt_count` 中的 `HARDIRQ_OFFSET` 标记当前在硬中断上下文，后者递减并检查是否有 softirq 待处理。这个分水岭在 04-§6 有详细分析。

`do_interrupt_handler()` 负责将控制权从架构相关层交接到中断控制器驱动层：

```c
// arch/arm64/kernel/entry-common.c:270
static void do_interrupt_handler(struct pt_regs *regs,
                 void (*handler)(struct pt_regs *))
{
    struct pt_regs *old_regs = set_irq_regs(regs);

    if (on_thread_stack())
        call_on_irq_stack(regs, handler);  // 如果在内核线程栈上，切到 IRQ 栈
    else
        handler(regs);                     // 已在中段栈上，直接调

    set_irq_regs(old_regs);
}
```

`handle_arch_irq` 是一个函数指针，在 GIC 初始化时由 `set_handle_irq(gic_handle_irq)` 设置（`irq-gic.c:1266`）。在 STM32MP257 上，它就是 `gic_handle_irq`。

`do_interrupt_handler()` 内部有一个关键判断——**中断栈切换**：

```c
// arch/arm64/kernel/entry-common.c:270
static void do_interrupt_handler(struct pt_regs *regs,
                 void (*handler)(struct pt_regs *))
{
    struct pt_regs *old_regs = set_irq_regs(regs);

    if (on_thread_stack())
        call_on_irq_stack(regs, handler);  // 在任务栈上 → 切到 IRQ 栈
    else
        handler(regs);                     // 已在中段栈上 → 直接调

    set_irq_regs(old_regs);
}
```

#### 5.4.6 中断栈切换：三级栈系统

ARM64 内核为每个 CPU 分配了三类栈：

| 栈 | 大小 | 用途 |
|----|------|------|
| **Thread stack**（任务栈） | 16KB | 进程在内核态执行（系统调用、page fault、文件系统路径） |
| **IRQ stack**（中断栈） | 8KB | 中断处理专用，每个 CPU 独立一份 |
| **Overflow stack**（溢出栈） | — | 检测栈溢出时的紧急救援栈 |

**为什么中断需要独立栈？**

进程在内核态执行时（比如 ext4 文件系统在遍历深度目录树），可能已经用掉了 12KB 的任务栈。此时如果发生中断，handler 还需要几百字节栈空间。如果中断和任务共享同一个栈，在任务栈快用光时来中断——必然溢出。

每个 CPU 的 IRQ 栈在 `init_IRQ()` 时分配，地址存在 per-CPU 变量 `irq_stack_ptr` 中。`on_thread_stack()` 检查当前 SP 是否在任务栈范围内：

```c
// arch/arm64/include/asm/stacktrace.h
#define on_thread_stack()  on_task_stack(current, current_stack_pointer, 1)
```

- **`on_thread_stack() = true`** → 这是**首次中断**，CPU 在用任务栈。需要切到 IRQ 栈上执行 handler
- **`on_thread_stack() = false`** → 已经在 IRQ 栈上了。说明发生了**中断嵌套**（一个中断 handler 执行中，又被另一个中断打断）。直接在当前栈上调用 handler，不需要再切

`call_on_irq_stack` 的汇编实现（`entry.S:875`）：

```asm
SYM_FUNC_START(call_on_irq_stack)
    stp     x29, x30, [sp, #-16]!       // ① 在任务栈上保存 FP/LR
    mov     x29, sp

    ldr_this_cpu x16, irq_stack_ptr, x17  // ② 加载本 CPU 的 IRQ 栈底指针

    add     sp, x16, #IRQ_STACK_SIZE    // ③ SP 切到 IRQ 栈顶
    blr     x1                          // ④ 调用 handler(regs)

    mov     sp, x29                     // ⑤ SP 切回任务栈
    ldp     x29, x30, [sp], #16        // ⑥ 恢复 FP/LR
    ret
SYM_FUNC_END(call_on_irq_stack)
```

三步完成栈切换：

```
中断到达 → CPU 在任务栈上保存 pt_regs（kernel_entry）
         → on_thread_stack() = true
         → 在任务栈上保存 FP/LR
         → SP 指向 irq_stack_ptr + IRQ_STACK_SIZE（切栈）
         → handler 在 IRQ 栈上执行
         → handler 返回，SP 切回任务栈
         → 恢复 FP/LR，继续中断返回路径（irq_exit_rcu → kernel_exit → eret）
```

**需要特别说明：ARM64 的硬中断 handler 执行期间，不允许硬中断嵌套。** 原因在硬件层面——CPU 响应 IRQ 时自动将 PSTATE.I 置 1（屏蔽所有 IRQ），直到 `eret` 指令恢复 SPSR_EL1 中保存的旧 PSTATE 时才解除。整个 `kernel_entry → handler → kernel_exit → eret` 路径上 IRQ 始终是关的。所以 "handler 执行中被另一个硬中断打断" 这件事**不会发生**。

那 `on_thread_stack() = false` 是给谁准备的？

**大部分时候用不上——它防的是三种一辈子碰不到几次的极端情况：**

**情况 1：NMI（不可屏蔽中断）。** 你可以把 ARM64 的异常分成两档：普通 IRQ 用 PSTATE.I 屏蔽，NMI 用 PSTATE.F 屏蔽。如果一个中断被配置成 NMI，它连普通 IRQ handler 也能打断——就像急诊医生可以打断普通门诊医生一样。此时 `on_thread_stack()` 会发现 SP 正在 IRQ 栈上（被普通中断占着呢），就不切栈了。

**情况 2：栈溢出了。** 如果内核代码的栈递归太深（比如 `printk → 驱动 → printk` 死循环），SP 可能冲出栈底。`kernel_ventry` 宏在给 pt_regs 分配空间后会检查 SP 是否越界——如果越界就切到 overflow 栈（一个小型备用栈），然后调 `handle_bad_stack` 打 `kernel panic`。这种情况下 SP 不在任务栈上，`on_thread_stack()` 返回 false。

不妨把 overflow 栈理解成楼里的**消防通道**——正常情况下没人用，但一旦主楼梯垮了（栈溢出），你至少有条路逃出去报信（打 panic）。

**情况 3：固件事件。** ARM 的 SDEI 机制允许固件（跑在 EL3 的信任固件）在发生特定硬件事件时（比如内存 CE 错误）回调 Linux。SDEI 有自己专用的 per-CPU 栈——`sdei_stack_normal_ptr` 和 `sdei_stack_critical_ptr`（《sdei.c:34-35》），与任务栈、IRQ 栈完全独立。SDEI 事件通过独立的入口向量进入，会切到 SDEI 栈上执行，不碰中断栈或任务栈。如果在 SDEI handler 中又发生了需要调 `do_interrupt_handler` 的极端情况（实际上几乎不可能），此时 SP 在 SDEI 栈上，`on_thread_stack()` 返回 false。

**一句话**：普通用户一辈子碰不到 `on_thread_stack() = false`。这个 `else` 分支是给内核开发者做安全兜底的。

这个机制也用于 softirq：内核在 `irq_exit_rcu` 中检测到有 softirq 待处理时，在 `irq.c:86` 同样调用 `call_on_irq_stack(NULL, ____do_softirq)`——softirq handler 也是在 IRQ 栈上执行。这也是为什么 softirq context 被认为是原子的（它在中断栈上运行，与硬中断共享相同的上下文约束）。

此时我们从汇编到 C 的完整调用链是：

```
el1h_64_irq_handler(regs)
  └─ __el1_irq(regs, handle_arch_irq)
       ├─ irq_enter_rcu()              ← 标记硬中断上下文
       ├─ do_interrupt_handler(regs, gic_handle_irq)
       │    ├─ on_thread_stack()?       ← 判断是否切 IRQ 栈
       │    └─ gic_handle_irq(regs)     ← ★ 进入 GIC 主循环
       └─ irq_exit_rcu()               ← 退出硬中断上下文
```

---

### 5.5 断面 1.3：GIC 中断主循环——gic_handle_irq

#### 5.5.1 主循环代码

`gic_handle_irq` 在 `irq-gic.c:345`：

```c
// drivers/irqchip/irq-gic.c:345
static void __exception_irq_entry gic_handle_irq(struct pt_regs *regs)
{
    u32 irqstat, irqnr;
    struct gic_chip_data *gic = &gic_data[0];
    void __iomem *cpu_base = gic_data_cpu_base(gic);

    do {
        // ★ 步骤 1：读取 GICC_IAR，获取中断 ID（ACK 中断）
        irqstat = readl_relaxed(cpu_base + GIC_CPU_INTACK);
        irqnr = irqstat & GICC_IAR_INT_ID_MASK;

        // ★ 步骤 2：ID >= 1020 → 伪中断（spurious），退出循环
        if (unlikely(irqnr >= 1020))
            break;

        // ★ 步骤 3：如果支持 split EOI（deactivate），先写 EOI
        if (static_branch_likely(&supports_deactivate_key))
            writel_relaxed(irqstat, cpu_base + GIC_CPU_EOI);
        isb();

        // ★ 步骤 4：IPI（SGI，irqnr <= 15）需要特殊处理
        if (irqnr <= 15) {
            smp_rmb();
            this_cpu_write(sgi_intid, irqstat);
        }

        // ★ 步骤 5：将 hwirq 传递给核心层分发
        generic_handle_domain_irq(gic->domain, irqnr);
    } while (1);
}
```

这是一个 **do/while(1) 循环**——每次处理一个中断，直到读出 spurious ID（>=1020）才退出。这意味着在高频中断场景下（如网卡），一次读出来的多个中断可以在同一个入口中连续处理，不需要反复走 entry.S 的保存/恢复路径。

#### 5.5.2 逐步骤拆解——以 KEY0 按下为例

**步骤 1：读 GICC_IAR（ACK）**

CPU Interface 的 GICC_IAR 寄存器（偏移 0x0C）是一个关键寄存器——**读它即 ACK**。读操作返回当前最高优先级等待中断的 ID，同时告诉 GIC：CPU 已经接收了这个中断，GIC 可以降低这个中断的电平信号。

对于 KEY0 按下，GICC_IAR 返回的 ID 是 **273**（GIC SPI 编号）。这个 ID 被称为 **hwirq**（硬件中断号），它在 GIC-400 的地址空间中是一个全局编号——SPI 范围是 32–1019。

我们的 `irqstat = readl_relaxed(cpu_base + 0x0C)` 读到的值是：`irqstat = 273`（最低 10 bit 是中断 ID，bit 10+ 编码了源 CPU——对 SGI 有意义，对 SPI 无用）。

**步骤 2：spurious 检查**

`irqnr = irqstat & GICC_IAR_INT_ID_MASK` 得到 273，远小于 1020，所以不是 spurious，继续处理。

**步骤 3：split EOI**

`supports_deactivate_key` 静态分支在 GIC-400 上通常为 **false**——GIC-400 不支持 GICv3 风格的分离 EOI/deactivate 机制。所以在 STM32MP257 上，EOI 不会在这里写，而是延迟到流控函数（handle_fasteoi_irq）的 `irq_chip_eoi_parent()` 中通过 GIC 的 `irq_eoi()` 回调写 GICC_EOI。

**步骤 5：generic_handle_domain_irq**

这是将 hwirq 转给 Linux 核心层的关键调用：

```c
generic_handle_domain_irq(gic->domain, irqnr);
```

它内部做了：

1. `irq_find_mapping(domain, hwirq)` → 在 GIC irq_domain 的 revmap 树中查找 hwirq=273 对应的 **virq**（Linux IRQ 号）。这个映射是在请求中断时（`irq_create_fwspec_mapping`）建立的。

2. `generic_handle_irq(virq)` → 从 `irq_desc[virq]` 中找到 irq_desc，然后调用 `desc->handle_irq(desc)`。

对于 KEY0（PH5），virq 是 **268**（`/proc/interrupts` 中看到的编号）。所以 `desc->handle_irq` 就是我们在 02-§4 和 03-§3.7 中看到的 **handle_fasteoi_irq**。

---

### 5.6 断面 1.4：流控——handle_fasteoi_irq

`handle_fasteoi_irq` 是 GIC 风格的流控函数——中断是"EOI 驱动"的：硬件自动 mask，软件只需要在恰当的时候调用 EOI。其完整代码在 `kernel/irq/chip.c:687`：

```c
// kernel/irq/chip.c:687
void handle_fasteoi_irq(struct irq_desc *desc)
{
    struct irq_chip *chip = desc->irq_data.chip;

    raw_spin_lock(&desc->lock);

    // ★ 安全检查：affinity 竞争导致的中断重发
    if (!irq_may_run(desc)) {
        if (irqd_needs_resend_when_in_progress(&desc->irq_data))
            desc->istate |= IRQS_PENDING;
        goto out;
    }

    desc->istate &= ~(IRQS_REPLAY | IRQS_WAITING);

    // ★ 没注册 handler 或已被禁用 → mask 掉
    if (unlikely(!desc->action || irqd_irq_disabled(&desc->irq_data))) {
        desc->istate |= IRQS_PENDING;
        mask_irq(desc);
        goto out;
    }

    kstat_incr_irqs_this_cpu(desc);  // 递增 /proc/interrupts 计数

    // ★ 如果标志了 IRQS_ONESHOT（线程化 IRQ），先 mask
    if (desc->istate & IRQS_ONESHOT)
        mask_irq(desc);

    handle_irq_event(desc);          // ★ 调用 action 链表

    cond_unmask_eoi_irq(desc, chip); // ★ 条件性 unmask + EOI

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

#### 5.6.1 EXTI 的 mask/unmask/eoi 回调

`mask_irq(desc)` 和随后的 `unmask_irq()`、`eoi_irq()` 最终调用的是 irq_chip 中的回调。对于经过 EXTI 的中断，这段 irq_chip 是 `stm32mp_exti_chip`：

```c
// irq-stm32mp-exti.c:520
static struct irq_chip stm32mp_exti_chip = {
    .name           = "stm32mp-exti",
    .irq_eoi        = stm32mp_exti_eoi,
    .irq_mask       = stm32mp_exti_mask,
    .irq_unmask     = stm32mp_exti_unmask,
    .irq_set_type   = stm32mp_exti_set_type,
    .irq_set_affinity = irq_chip_set_affinity_parent,
    ...
};
```

**stm32mp_exti_mask** — 写 mask_cache 并委托父级（GIC）：

```c
// irq-stm32mp-exti.c:376
static void stm32mp_exti_mask(struct irq_data *d)
{
    struct stm32mp_exti_chip_data *chip_data = irq_data_get_irq_chip_data(d);
    const struct stm32mp_exti_bank *bank = chip_data->reg_bank;

    raw_spin_lock(&chip_data->rlock);
    chip_data->mask_cache &= ~stm32mp_exti_clr_bit(d, bank->imr_ofst); // 清除 cache bit
    raw_spin_unlock(&chip_data->rlock);

    irq_chip_mask_parent(d);  // 委托 GIC 进行真实的 mask
}
```

注意 **mask_cache**：EXTI 驱动不直接写 IMR 寄存器，而是先在内存的 `mask_cache` 中将对应位清 0，然后通过 `irq_chip_mask_parent` 委托给 GIC——GIC 的 `gic_chip` 会写 GICD_ICENABLER（中断清除使能寄存器）。EXTI 的真正 IMR 写入推迟到 resume 或 EOI 时批量同步。这种"写缓存"模式在 02-§2.8 有详细分析。

**stm32mp_exti_unmask** — 恢复 mask_cache 并委托父级：

```c
// irq-stm32mp-exti.c:388
static void stm32mp_exti_unmask(struct irq_data *d)
{
    ...
    chip_data->mask_cache |= stm32mp_exti_set_bit(d, bank->imr_ofst);
    ...
    irq_chip_unmask_parent(d);
}
```

**stm32mp_exti_eoi** — 清 pending + 恢复 mask_cache + 委托 EOI：

```c
// irq-stm32mp-exti.c:359
static void stm32mp_exti_eoi(struct irq_data *d)
{
    struct stm32mp_exti_chip_data *chip_data = irq_data_get_irq_chip_data(d);
    const struct stm32mp_exti_bank *bank = chip_data->reg_bank;

    raw_spin_lock(&chip_data->rlock);

    // ★ 写 RPR 清上升沿 pending（寄存器写 1 清 0）
    stm32mp_exti_write_bit(d, bank->rpr_ofst);
    // ★ 写 FPR 清下降沿 pending
    stm32mp_exti_write_bit(d, bank->fpr_ofst);

    // ★ 恢复 mask_cache（之前被 mask 清掉的 bit 重新置 1）
    chip_data->mask_cache |= stm32mp_exti_set_bit(d, bank->imr_ofst);

    raw_spin_unlock(&chip_data->rlock);

    // ★ 委托 GIC 写 GICC_EOI
    irq_chip_eoi_parent(d);
}
```

EOI 做了三件事：
1. **清 RPR/FPR** —— 告诉 EXTI：这个中断事件已经处理完了，可以接收下一次边沿检测
2. **恢复 mask_cache** —— mask 阶段清除的 cache bit 重新置 1（使能下次中断）
3. **irq_chip_eoi_parent** —— 去调 GIC 的 EOI 回调，写 GICC_EOI 寄存器

在 STM32MP257 上，因为 GIC-400 不支持 split EOI（`supports_deactivate_key = false`），GIC 的 `irq_eoi` 回调就是写 `GICC_EOI`（偏移 0x10）：

```c
// irq-gic.c:231
static void gic_eoi_irq(struct irq_data *d)
{
    writel_relaxed(irq_data_get_irq_chip_data(d), GIC_CPU_EOI);
}
```

#### 5.6.2 KEY0 在 handle_fasteoi_irq 中的实际流控序列

对于 gpio-keys 驱动（**非 ONESHOT，非线程化**），handle_fasteoi_irq 中的序列是：

```
① irq_may_run 检查          → 通过
② 无 action 或 disabled 检查 → 通过（action 已注册）
③ kstat_incr_irqs_this_cpu  → /proc/interrupts 计数+1
④ IRQS_ONESHOT 检查         → 否，跳过 mask
⑤ handle_irq_event(desc)    → ★ 调用 action->handler
⑥ cond_unmask_eoi_irq(desc, chip):
     └─ !IRQS_ONESHOT        → 直接 chip->irq_eoi() ← EXTI EOI
⑦ 检查 IRQS_PENDING         → 否，结束
```

**步骤 ⑥ 的 cond_unmask_eoi_irq 是关键**——对于非 ONESHOT 中断，它只调 `chip->irq_eoi()`，不需要 unmask（因为 handle_fasteoi 假设硬件会自动屏蔽，EOI 后自动解除屏蔽）。

---

### 5.7 断面 1.5：handler 执行——gpio_keys_gpio_isr

`handle_irq_event(desc)` 的实际工作是遍历 action 链表，逐一调用 handler。我们看代码：

```c
// kernel/irq/handle.c:202
irqreturn_t handle_irq_event(struct irq_desc *desc)
{
    desc->istate &= ~IRQS_PENDING;
    irqd_set(&desc->irq_data, IRQD_IRQ_INPROGRESS);
    raw_spin_unlock(&desc->lock);      // ★ 解锁！handler 在锁外执行

    ret = handle_irq_event_percpu(desc);

    raw_spin_lock(&desc->lock);
    irqd_clear(&desc->irq_data, IRQD_IRQ_INPROGRESS);
    return ret;
}
```

注意关键设计：**在调用 handler 之前，desc->lock 已经被释放**。这意味着 handler 执行期间，其他 CPU 可以访问同一个 irq_desc（例如，另一个 CPU 可以修改亲和性）。`IRQD_IRQ_INPROGRESS` 标志防止了并发释放中断等操作。

```c
// kernel/irq/handle.c:189
irqreturn_t handle_irq_event_percpu(struct irq_desc *desc)
{
    retval = __handle_irq_event_percpu(desc);

    add_interrupt_randomness(desc->irq_data.irq);  // 用中断时间增加熵池

    if (!irq_settings_no_debug(desc))
        note_interrupt(desc, retval);  // ★ 虚假中断检测
    return retval;
}
```

`__handle_irq_event_percpu` 才是真正的 handler 循环：

```c
// kernel/irq/handle.c:139
irqreturn_t __handle_irq_event_percpu(struct irq_desc *desc)
{
    irqreturn_t retval = IRQ_NONE;
    struct irqaction *action;

    for_each_action_of_desc(desc, action) {
        irqreturn_t res;

        trace_irq_handler_entry(irq, action);
        res = action->handler(irq, action->dev_id);  // ★ 调用驱动注册的 handler
        trace_irq_handler_exit(irq, action, res);

        switch (res) {
        case IRQ_WAKE_THREAD:
            __irq_wake_thread(desc, action);  // → 线程化 IRQ 路径
            break;
        default:
            break;
        }

        retval |= res;
    }

    return retval;
}
```

对于 KEY0 按下，`action->handler` 是 `gpio_keys_gpio_isr`。

#### gpio_keys_gpio_isr——顶半部的实际工作

```c
// drivers/input/keyboard/gpio_keys.c:417
static irqreturn_t gpio_keys_gpio_isr(int irq, void *dev_id)
{
    struct gpio_button_data *bdata = dev_id;

    BUG_ON(irq != bdata->irq);

    // 如果是唤醒源，阻止系统进入睡眠
    if (bdata->button->wakeup) {
        pm_stay_awake(bdata->input->dev.parent);
        ...
    }

    if (bdata->debounce_use_hrtimer) {
        // ★ 使用 hrtimer 消抖（比 workqueue 更精确）
        hrtimer_start(&bdata->debounce_timer,
                  ms_to_ktime(bdata->software_debounce),
                  HRTIMER_MODE_REL);
    } else {
        // ★ 使用 workqueue 消抖（默认路径，software_debounce=10ms）
        mod_delayed_work(system_wq,
                 &bdata->work,
                 msecs_to_jiffies(bdata->software_debounce));
    }

    return IRQ_HANDLED;   // ← 返回 IRQ_HANDLED，不走 IRQ_WAKE_THREAD
}
```

handler 只做了一件事：**调度一个 10ms 后执行的 delayed work，然后立即返回**。它没有读 GPIO 引脚状态，没有上报 input 事件——这些工作全部推迟到下半部的 workqueue 中。

为什么不在顶半部直接上报 input 事件？原因有二：

1. **IRQ context 中不可 sleep**（00-History §1.1）：`gpiod_get_value_cansleep` 和 `input_event` 虽然不会主动 sleep，但 input 子系统的某些路径（如 `EV_FF` 力反馈）可能牵涉到 `mutex_lock`，顶半部没有进程上下文去处理这些等待。

2. **消抖需要定时器**：机械按键按下后的 10–20ms 内会有多次反弹，需要消抖。这个延迟意味着"推迟处理"是必然的——顶半部也不可能自旋等待 10ms（`irq_disabled = true`）。

所以 gpio-keys 驱动天然适合 workqueue 模式：顶半部只调度 work，worker 线程在 10ms 后运行，此时处于**进程上下文**，可以 sleep、可以持有 mutex、可以调用各种可能阻塞的 API。

---

### 5.8 断面 1.6：中断退出——irq_exit_rcu

handler 返回 IRQ_HANDLED 后，`__handle_irq_event_percpu` 将结果累加到 `retval`（`IRQ_HANDLED | IRQ_NONE = IRQ_HANDLED`），然后一路返回到 `handle_fasteoi_irq`，执行 `cond_unmask_eoi_irq`（调 EXTI EOI）后释放 `desc->lock`。

控制权回到 `__el1_irq`（`entry-common.c:496`）：

```c
static __always_inline void __el1_irq(struct pt_regs *regs,
                      void (*handler)(struct pt_regs *))
{
    enter_from_kernel_mode(regs);

    irq_enter_rcu();                    // 已进入
    do_interrupt_handler(regs, handler); // 已完成
    irq_exit_rcu();                     // ★ 现在退出

    arm64_preempt_schedule_irq();
    exit_to_kernel_mode(regs);
}
```

`irq_exit_rcu()` 调用 `__irq_exit_rcu()`（`softirq.c:633`）：

```c
// kernel/softirq.c:633
static inline void __irq_exit_rcu(void)
{
    local_irq_disable();                    // 关本地 IRQ
    account_hardirq_exit(current);
    preempt_count_sub(HARDIRQ_OFFSET);      // 递减硬中断计数

    if (!in_interrupt() && local_softirq_pending())
        invoke_softirq();                   // ★ 有 softirq pending 就执行

    tick_irq_exit();
}
```

`preempt_count_sub(HARDIRQ_OFFSET)` 将 `preempt_count` 中的 `HARDIRQ_SHIFT` 位递减（回到 0，表示不再在硬中断上下文中）。此时 `in_interrupt()` 检查整个 `preempt_count`——如果 softirq 计数也为 0（没有被嵌套的 softirq），且 `local_softirq_pending()` 非空，才调用 `invoke_softirq()`。

对于 gpio-keys 的 workqueue 路径：`mod_delayed_work` **不会置位任何 softirq pending bit**——workqueue 通过 `wake_up_worker` 唤醒内核线程，不走 softirq。所以 `local_softirq_pending()` 返回 0，`invoke_softirq()` 不被调用。

**这就是 04-§1.3 所说的"下半部分水岭"**：

```
irq_exit_rcu()
  └─ __irq_exit_rcu()
       ├─ softirq pending? → 否（workqueue 不置位 softirq）
       └─ eret → 返回被中断的上下文
```

没有 softirq 要处理，中断直接退出。控制权从 `ret_to_kernel` 到 `kernel_exit` 到 `eret`，**CPU 恢复执行被 KEY0 按下时打断的代码**——可能是某个进程的用户态代码，也可能是内核的 idle 循环。

与此同时，`system_wq` 的 worker 线程还在睡眠——它将在 **10ms 后**被 `mod_delayed_work` 的定时器唤醒。

---

### 5.9 断面 1.7：下半部——workqueue 执行

#### 5.9.1 delayed_work 的定时触发

`mod_delayed_work(system_wq, &bdata->work, msecs_to_jiffies(10))` 做了三件事：

1. 将 `bdata->work` 加入 `system_wq` 的某个 `pool_workqueue`
2. 设置一个内核定时器（`timer`），10ms 后到期
3. worker 线程在定时器到期、且被调度器选中后运行

关于 CMWQ（Concurrency Managed Workqueue）的完整数据结构（`work_struct → pool_workqueue → worker_pool → worker`），已经在 04-§4 中详细分析过，这里不再重复。关键是：**workqueue 下半部与中断返回在时间上完全解耦**——irq_exit_rcu 执行 eret 后，CPU 已经回去做别的事了，worker 线程在 10ms 后的某个时刻才被调度。

#### 5.9.2 gpio_keys_gpio_work_func

当 worker 线程被唤醒并取得 `bdata->work` 后，执行 `gpio_keys_gpio_work_func`：

```c
// drivers/input/keyboard/gpio_keys.c:399
static void gpio_keys_gpio_work_func(struct work_struct *work)
{
    struct gpio_button_data *bdata =
        container_of(work, struct gpio_button_data, work.work);

    gpio_keys_debounce_event(bdata);
}
```

`gpio_keys_debounce_event` 调用 `gpio_keys_gpio_report_event` 和 `input_sync`：

```c
// drivers/input/keyboard/gpio_keys.c:360
static void gpio_keys_gpio_report_event(struct gpio_button_data *bdata)
{
    const struct gpio_keys_button *button = bdata->button;
    struct input_dev *input = bdata->input;
    unsigned int type = button->type ?: EV_KEY;
    int state;

    // ★ 读 GPIO 引脚电平（此时在进程上下文，可以 sleep）
    state = bdata->debounce_use_hrtimer ?
            gpiod_get_value(bdata->gpiod) :
            gpiod_get_value_cansleep(bdata->gpiod);
    ...

    // ★ 上报按键事件到 input 子系统
    if (type == EV_ABS) {
        if (state)
            input_event(input, type, button->code, button->value);
    } else {
        input_event(input, type, *bdata->code, state);
    }
}

// 同文件:390
static void gpio_keys_debounce_event(struct gpio_button_data *bdata)
{
    gpio_keys_gpio_report_event(bdata);
    input_sync(bdata->input);     // ★ 同步——让事件被消费者读取

    if (bdata->button->wakeup)
        pm_relax(bdata->input->dev.parent);
}
```

**关键对比**：如果在顶半部直接读 GPIO、上报 input 事件，`gpiod_get_value_cansleep` 和 `input_event` 虽然通常不会阻塞，但：

- `gpiod_get_value` 在非 cansleep 模式下涉及 `spin_lock` + 寄存器读，本身是安全的
- 但 input 子系统的 `input_event → input_handle_event` 可能触发 `EV_FF` 力反馈等复杂路径

gpio-keys 驱动选择 workqueue 的根本原因还是**消抖需要定时器**——没有 10ms 延迟，机械按键的反弹会导致一次按压上报多次事件。

#### 5.9.3 input 子系统的下发路径

`input_event(dev, type, code, value)` 将事件放入 `dev->evdev` 的缓冲区，`input_sync()` 将 `EV_SYN/SYN_REPORT` 标记写入缓冲区并唤醒等待的读取进程。

对于 `evtest` 或 `retroarch` 这类用户态程序，它们的读取路径是：

```
read(/dev/input/event0)
  └─ evdev_read()
       └─ evdev_fetch_next_event()
            └─ 从循环缓冲区读取 struct input_event
```

用户态得到的 `struct input_event` 结构：

```c
struct input_event {
    struct timeval time;   // 事件发生的时间戳
    __u16 type;            // EV_KEY = 0x01
    __u16 code;            // KEY_0 = 11
    __s32 value;           // 0 (释放) 或 1 (按下)
};
```

---

### 5.10 断面 1.8：用户态可见

用户态通过 `evtest` 观察：

```
~# evtest /dev/input/event0
Input driver version is 1.0.1
Input device ID: bus 0x19 vendor 0x1 product 0x1 version 0x100
Input device name: "gpio-keys"

Event: time 1623456789.123456, type 1 (EV_KEY), code 11 (KEY_0), value 1  ← 按下
Event: time 1623456789.123456, type 0 (EV_SYN), code 0 (SYN_REPORT), value 0

Event: time 1623456799.654321, type 1 (EV_KEY), code 11 (KEY_0), value 0  ← 释放
Event: time 1623456799.654321, type 0 (EV_SYN), code 0 (SYN_REPORT), value 0
```

至此第一幕的完整路径全部打通。

---

### 5.11 第一幕路径全景总结（见本章末尾的完整对比表）

---

## 第二幕：I2C 触摸屏中断（threaded IRQ 路径）

### 5.12 场景设定

#### 5.12.1 第二个问题

ATK 板上有一块 **FT6336**（或类似 FT6x36 系列）电容触摸屏，通过 **I2C2** 总线与 SoC 通信。触摸屏的中断引脚连接到 SoC 的某个 GPIO（通常是 EXTI 事件），中断触发后驱动需要通过 I2C 读取触摸坐标。

**问题是**：同样是"外设触发中断→驱动读取数据→上报 input 事件"的模式，为什么触摸屏驱动不使用 gpio-keys 那样的 workqueue，而必须使用**线程化 IRQ**？

答案在 04-§5.1 已经说过：**I2C 传输需要 sleep**。`i2c_transfer()` 内部需要等待硬件 ACK、等待 DMA 完成，这些等待机制都基于调度——意味着调用者必须处于进程上下文。workqueue 虽然也是进程上下文，但 workqueue 是共享线程池，而线程化 IRQ 为每路中断创建**专用内核线程**（`irq/N-xxx`），提供更好的实时性和确定性。

所以触摸屏驱动的选择是：

| 需求 | 方案 |
|------|------|
| 顶半部不能 sleep，但需要快速确认中断来源 | 轻量级顶半部（读 GPIO 状态寄存器）→ return IRQ_WAKE_THREAD |
| 底半部需要 i2c_transfer（sleep） | 专用内核线程 irq_thread → thread_fn |
| 中断再次触发时，必须等待前一次 I2C 传输完成 | IRQF_ONESHOT 标志（传输完成前不 unmask） |

#### 5.12.2 I2C 触摸屏的中断路径

触摸屏的中断连接与 KEY0 不同。KEY0 是 GPIO 按键，中断经过 GPIO→EXTI→GIC 三级。I2C 触摸屏的中断连接方式取决于硬件设计。两种可能：

- **方式 A：触摸屏 INT 引脚直接连到 SoC GPIO** → 与 KEY0 一样走 GPIO→EXTI→GIC
- **方式 B：触摸屏 INT 引脚连到 I2C 控制器的中断输出** → I2C 控制器直连 GIC

ATK 板上的具体连接取决于原理图，但无论哪种方式，**中断的前半段路径**（硬件→GIC→entry.S→gic_handle_irq→handle_fasteoi_irq→handle_irq_event）与第一幕完全相同。

**关键区别在顶半部和返回路径上**。

#### 5.12.3 request_threaded_irq 的注册

触摸屏驱动在 probe() 中调用：

```c
// 典型触摸屏驱动的 request_threaded_irq 调用
error = devm_request_threaded_irq(&client->dev, client->irq,
                  ft6x36_irq_handler,   // top half
                  ft6x36_irq_thread_fn, // thread fn
                  IRQF_TRIGGER_FALLING | IRQF_ONESHOT,
                  "ft6x36", client);
if (error)
    dev_err(&client->dev, "request irq failed: %d\n", error);
```

这里与 gpio-keys 注册的区别：

| 参数 | gpio-keys | 触摸屏 |
|------|----------|--------|
| handler | `gpio_keys_gpio_isr` | `ft6x36_irq_handler` |
| thread_fn | NULL | `ft6x36_irq_thread_fn` |
| irqflags | `IRQF_TRIGGER_RISING \| IRQF_TRIGGER_FALLING` | `IRQF_TRIGGER_FALLING \| IRQF_ONESHOT` |

`request_threaded_irq` 在 `kernel/irq/manage.c` 中实现的完整流程已经在 04-§5.2 分析过，这里只关注与运行路径直接相关的内核线程创建。

当 `thread_fn != NULL` 时，`__setup_irq()` 调用 `setup_irq_thread()` 创建一个内核线程：

```c
// kernel/irq/manage.c:1561（简化）
if (new->thread_fn && !nested) {
    struct task_struct *t;
    t = kthread_create(irq_thread, new, "irq/%d-%s", irq, new->name);
    ...
    new->thread = t;
}
```

创建后的线程在 `/proc` 中可见：

```
~# ps | grep irq/
  125 root       0:00 [irq/269-ft6x36]
```

---

### 5.13 断面 2.1：顶半部——handler 的执行

触摸屏被点击后，中断硬件路径走完，进入 `__handle_irq_event_percpu`。`action->handler` 是触摸屏驱动注册的顶半部。

典型的触摸屏顶半部（伪代码简化真实驱动）：

```c
static irqreturn_t ft6x36_irq_handler(int irq, void *dev_id)
{
    struct ft6x36_data *data = dev_id;

    /*
     * 顶半部只做一件事：确认中断来自本设备。
     * 真正的 I2C 读取在 thread_fn 中完成（因为 i2c_transfer 需要 sleep）。
     *
     * 某些实现可能通过 GPIO 口直接读一个状态引脚来判断——这不会 sleep。
     */
    if (!(readl_relaxed(data->reg_base + FT6X36_REG_STATUS) & FT6X36_TP_INT))
        return IRQ_NONE;  // 不是我们的中断

    return IRQ_WAKE_THREAD;   // ← ★ 关键：唤醒线程处理
}
```

这个 handler 与 gpio-keys 的 handler 有三个关键区别：

1. **返回值不是 IRQ_HANDLED，而是 IRQ_WAKE_THREAD**——告诉核心层"我需要唤醒一个内核线程来处理"
2. **顶半部尽可能轻量**——只确认中断来源，不做数据读取
3. **没有 workqueue/hrtimer 调度**——不延迟，立即唤醒线程

`__handle_irq_event_percpu` 检测到 `IRQ_WAKE_THREAD` 后：

```c
// kernel/irq/handle.c:165
switch (res) {
case IRQ_WAKE_THREAD:
    __irq_wake_thread(desc, action);   // ★ 唤醒线程
    break;
}
```

---

### 5.14 断面 2.2：__irq_wake_thread——线程唤醒

`__irq_wake_thread` 在 `kernel/irq/handle.c:61`：

```c
// kernel/irq/handle.c:61
void __irq_wake_thread(struct irq_desc *desc, struct irqaction *action)
{
    // 线程已退出？跳过
    if (action->thread->flags & PF_EXITING)
        return;

    // 已标记为待运行？跳过（避免重复唤醒）
    if (test_and_set_bit(IRQTF_RUNTHREAD, &action->thread_flags))
        return;

    // ★ 设置 ONESHOT mask（线程处理完成前，同中断不再次触发）
    desc->threads_oneshot |= action->thread_mask;

    // 递增 thread 活跃计数（供 synchronize_irq 等待用）
    atomic_inc(&desc->threads_active);

    // ★ 唤醒内核线程！
    wake_up_process(action->thread);
}
```

这个函数做了四件关键的事：

1. **设置 IRQTF_RUNTHREAD 标志**——标记"有工作要处理"。如果这个标志已经被设置（前一次中断的线程还没处理完），`test_and_set_bit` 返回 1 并直接 return。这意味着**一次线程处理周期中，多次中断触发只会唤醒一次**。

2. **设置 threads_oneshot mask**——这个 mask 与 `IRQF_ONESHOT` 标志配合：当线程正在运行时，同中断的再次触发**不会调用 handler**（因为硬件层面中断线被硬件自动屏蔽了），直到线程处理完成后才在 `irq_finalize_oneshot` 中清除这个 bit 并 unmask。

3. **递增 threads_active**——用于 `synchronize_irq()` 等待所有活跃线程处理完成。

4. **wake_up_process**——将 irq 线程放入运行队列，调度器会在合适的时机选中它执行。

**与第一幕的关键分叉对比**：

| | 第一幕（workqueue） | 第二幕（threaded IRQ） |
|--|-------------------|----------------------|
| 下半部触发方式 | `mod_delayed_work` 设置定时器 | `__irq_wake_thread` 直接唤醒线程 |
| 触发时机 | 在 handler 中 | 在 `__handle_irq_event_percpu` 中（调用 handler 后） |
| 延迟 | 10ms（software_debounce） | 0（立即唤醒） |
| 执行线程 | system_wq 的共享 worker | 专用 irq/N-xxx 线程 |
| 时间解耦程度 | 完全解耦（定时器到期 → 调度） | 部分解耦（立即唤醒 → 调度器决策） |

---

### 5.15 断面 2.3：中断退出——irq_exit_rcu

`__irq_wake_thread` 执行完毕后，`__handle_irq_event_percpu` 返回给 `handle_irq_event_percpu`，后者调用 `note_interrupt` 进行虚假中断检测（见 §5.16），然后返回给 `handle_fasteoi_irq`。

但对线程化 IRQ 来说，`handle_fasteoi_irq` 中的 `cond_unmask_eoi_irq` 行为**与第一幕不同**：

```c
// kernel/irq/chip.c:657
static void cond_unmask_eoi_irq(struct irq_desc *desc, struct irq_chip *chip)
{
    if (!(desc->istate & IRQS_ONESHOT)) {
        chip->irq_eoi(&desc->irq_data);       // ← 第一幕：直接 EOI
        return;
    }

    // IRQS_ONESHOT 路径（触摸屏）：
    if (!irqd_irq_disabled(&desc->irq_data) &&
        irqd_irq_masked(&desc->irq_data) && !desc->threads_oneshot) {
        // 线程已处理完（threads_oneshot=0）→ EOI + unmask
        chip->irq_eoi(&desc->irq_data);
        unmask_irq(desc);
    } else if (!(chip->flags & IRQCHIP_EOI_THREADED)) {
        // 线程还在运行 → 只 EOI，不 unmask
        chip->irq_eoi(&desc->irq_data);
    }
}
```

对于带 `IRQF_ONESHOT` 的触摸屏中断：

- `handle_fasteoi_irq` 执行到步骤 ④ 时：`desc->istate & IRQS_ONESHOT = true` → **先 mask_irq**
- `cond_unmask_eoi_irq` 中：`desc->threads_oneshot != 0`（被 `__irq_wake_thread` 设置了）→ **只 EOI，不 unmask**

这意味着：**线程处理期间，中断线保持 mask 状态**——同触摸屏的再次点击不会触发新的中断，直到线程完成 I2C 传输并调用 `irq_finalize_oneshot`。

回到 `__el1_irq` 后，`irq_exit_rcu()` 的执行与第一幕相同——`preempt_count_sub(HARDIRQ_OFFSET)`，检查 softirq pending（没有），eret 返回。

但与第一幕不同的是：**虽然中断处理函数已经返回，但 irq 线程正在运行队列中等待被调度执行**。

---

### 5.16 断面 2.4：下半部——irq_thread 的生命周期

#### 5.16.1 irq_thread 主循环

`irq_thread` 在 `kernel/irq/manage.c:1298`：

```c
// kernel/irq/manage.c:1298
static int irq_thread(void *data)
{
    struct irqaction *action = data;
    struct irq_desc *desc = irq_to_desc(action->irq);
    irqreturn_t (*handler_fn)(struct irq_desc *, struct irqaction *);

    irq_thread_set_ready(desc, action);
    sched_set_fifo(current);            // ★ 设为 SCHED_FIFO 实时优先级

    if (force_irqthreads() && ...)
        handler_fn = irq_forced_thread_fn;
    else
        handler_fn = irq_thread_fn;      // 正常路径

    irq_thread_check_affinity(desc, action);

    // ★ 主循环：等待中断、处理、再等待
    while (!irq_wait_for_interrupt(action)) {
        irqreturn_t action_ret;

        irq_thread_check_affinity(desc, action);
        action_ret = handler_fn(desc, action);
        if (action_ret == IRQ_WAKE_THREAD)
            irq_wake_secondary(desc, action);

        wake_threads_waitq(desc);
    }
    return 0;
}
```

`irq_wait_for_interrupt` 检查 `IRQTF_RUNTHREAD` 标志——如果没有被置位，线程在 `wait_event` 上睡眠。一旦 `__irq_wake_thread` 设置了 `IRQTF_RUNTHREAD` 并调了 `wake_up_process`，线程被唤醒，`irq_wait_for_interrupt` 返回 0，进入处理循环。

线程被创建时通过 `sched_set_fifo(current)` 将其调度策略设为 **SCHED_FIFO**（实时 FIFO）。这意味着相比普通 CFS 线程（`system_wq` 的 worker），irq 线程具有更高的响应优先级——**触摸屏中断的线程会比 gpio-keys 的 workqueue worker 更早被调度**。

#### 5.16.2 irq_thread_fn——调用 thread_fn

`handler_fn(desc, action)` 对于非 force_irqthreads 的情况是 `irq_thread_fn`：

```c
// kernel/irq/manage.c:1212
static irqreturn_t irq_thread_fn(struct irq_desc *desc,
        struct irqaction *action)
{
    irqreturn_t ret;

    ret = action->thread_fn(action->irq, action->dev_id);  // ★ 调驱动注册的 thread_fn
    if (ret == IRQ_HANDLED)
        atomic_inc(&desc->threads_handled);

    irq_finalize_oneshot(desc, action);  // ★ 清除 ONESHOT 状态、unmask
    return ret;
}
```

最终调用的是驱动注册的 `ft6x36_irq_thread_fn`。

#### 5.16.3 thread_fn——真正的 I2C 读取

```c
// 典型触摸屏驱动的 thread_fn（简化伪代码）
static irqreturn_t ft6x36_irq_thread_fn(int irq, void *dev_id)
{
    struct ft6x36_data *data = dev_id;
    struct i2c_client *client = data->client;
    u8 buf[6];
    int ret;

    /*
     * ★ i2c_transfer 可能 sleep！
     * I2C 控制器需要等待从设备 ACK，期间 CPU 可以调度做别的事。
     * 如果在硬中断上下文中调用 i2c_transfer，会触发 kernel BUG。
     */
    ret = i2c_master_recv(client, buf, sizeof(buf));
    if (ret < 0) {
        dev_err(&client->dev, "read touch data failed: %d\n", ret);
        return IRQ_NONE;
    }

    // 解析触摸坐标
    int x = ((buf[0] & 0x0F) << 8) | buf[1];
    int y = ((buf[2] & 0x0F) << 8) | buf[3];

    // 上报到 input 子系统
    input_report_abs(data->input, ABS_X, x);
    input_report_abs(data->input, ABS_Y, y);
    input_sync(data->input);

    return IRQ_HANDLED;
}
```

**为什么 i2c_transfer 需要 sleep？**

I2C 协议是同步串行协议：主机发送地址后，从设备需要在每个字节后发送 ACK 信号。如果使用 `spin_lock_irqsave` 忙等，CPU 会在等待期间空转。对于低速 I2C（标准模式 100kHz，一个字节需 80μs），不可接受。

内核的 I2C 核心层在 `i2c_transfer` 内部使用 `wait_for_completion` 等待 DMA 或中断完成——这必然导致调用进程进入睡眠等待状态。

在 IRQ context 中调用 `i2c_transfer` 会触发：

```
BUG: sleeping function called from invalid context at kernel/i2c/i2c-core-base.c:...
in_atomic(): 1, irqs_disabled(): 1, ...
```

这就是线程化 IRQ 存在的意义。

#### 5.16.4 irq_finalize_oneshot——处理完成后的收尾

`thread_fn` 返回后，`irq_thread_fn` 调用 `irq_finalize_oneshot`：

```c
// kernel/irq/manage.c — irq_finalize_oneshot 的简化逻辑
static void irq_finalize_oneshot(struct irq_desc *desc, struct irqaction *action)
{
    raw_spin_lock(&desc->lock);

    // 清除当前 action 在 threads_oneshot 中的 bit
    desc->threads_oneshot &= ~action->thread_mask;

    // 如果没有其他活跃 thread，且没有被禁用，则 unmask
    if (!desc->threads_oneshot && !irqd_irq_disabled(&desc->irq_data)) {
        unmask_irq(desc);              // ★ 终于 unmask！
        // 如果 PENDING 位被设置了（处理期间又有中断发生），重新触发
        if (desc->istate & IRQS_PENDING)
            check_irq_resend(desc, false);
    }

    raw_spin_unlock(&desc->lock);
}
```

到这一步，EXTI 和 GIC 的 mask 终于被解除，触摸屏的中断线可以再次触发。

#### 5.16.5 irq_thread 的等待循环

处理完成后，`irq_thread` 回到 `while (!irq_wait_for_interrupt(action))` 循环的顶部——再次检查 `IRQTF_RUNTHREAD` 标志。如果线程处理期间没有新中断发生（`IRQTF_RUNTHREAD` 没有被再次置位），线程回到睡眠状态，等待下一次 `__irq_wake_thread`。

---

### 5.17 第二幕路径全景总结

```
触摸屏点击
  │
  ├─ 硬件层（同第一幕 §5.3）
  │    INT 引脚电平变化 → EXTI/GIC → CPU
  │
  ├─ ARM64 入口（同第一幕 §5.4） + GIC 主循环（同第一幕 §5.5）
  │
  ├─ 流控：handle_fasteoi_irq                           §5.6
  │    IRQS_ONESHOT=true → mask_irq → handle_irq_event
  │    → cond_unmask_eoi_irq: 只 EOI，不 unmask
  │
  ├─ 顶半部：ft6x36_irq_handler                          §5.13
  │    读状态寄存器确认中断来源
  │    return IRQ_WAKE_THREAD
  │
  ├─ __irq_wake_thread                                   §5.14
  │    set_bit(IRQTF_RUNTHREAD)
  │    desc->threads_oneshot |= action->thread_mask
  │    wake_up_process(action->thread)
  │
  ├─ 中断退出：irq_exit_rcu（同第一幕 §5.8）
  │
  ├─ 【分叉点】调度器决策                                §5.16
  │    irq/N-ft6x36 被调度（SCHED_FIFO 优先级）
  │
  ├─ irq_thread 主循环                                   §5.16
  │    irq_wait_for_interrupt → 检查 IRQTF_RUNTHREAD
  │    → irq_thread_fn → action->thread_fn()
  │
  ├─ thread_fn：ft6x36_irq_thread_fn                     §5.16
  │    i2c_master_recv（sleep 等待 I2C 传输）
  │    → 解析坐标 → input_report_abs → input_sync
  │
  ├─ irq_finalize_oneshot                                §5.16
  │    threads_oneshot &= ~action->thread_mask
  │    → unmask_irq（恢复中断线）
  │
  └─ 用户态                                            §5.10
       evtest 收到 ABS_X / ABS_Y / SYN_REPORT 事件
```

---

## 第三幕：综合话题

### 5.18 两幕路径对比

#### 5.18.1 纵向对比

| 阶段 | 第一幕（gpio-keys KEY0） | 第二幕（I2C 触摸屏） |
|------|------------------------|--------------------|
| **DTS 路径** | GPIO PH5 → EXTI → GIC | 触摸 INT → EXTI(或直连) → GIC |
| **硬件触发** | 按键机械电平 → EXTI FPR 置位 | 电容感应电平 → EXTI RPR/FPR 置位 |
| **GIC 处理** | gic_handle_irq 循环读 GICC_IAR → ID=273 | 同上，ID 不同 |
| **流控函数** | handle_fasteoi_irq | handle_fasteoi_irq |
| **IRQS_ONESHOT** | 否 | **是**（线程化 IRQ 标志） |
| **mask_irq 时机** | 不由 handle_fasteoi 触发 | 由 handle_fasteoi 在 event 前触发 |
| **EOI 时机** | event 后立即 EOI | event 后 EOI，但 mask 保留 |
| **handler** | `gpio_keys_gpio_isr` | `ft6x36_irq_handler` |
| **handler 返回值** | `IRQ_HANDLED` | **`IRQ_WAKE_THREAD`** |
| **下半部触发** | `mod_delayed_work`（定时 10ms） | `__irq_wake_thread`（立即唤醒） |
| **下半部机制** | system_wq 共享 worker | 专用 `irq/N-ft6x36` 内核线程 |
| **调度优先级** | CFS（普通公平调度） | SCHED_FIFO（实时优先级） |
| **下半部执行** | 进程上下文，可调用 `gpiod_get_value_cansleep` | 进程上下文，可调用 `i2c_transfer` |
| **中断恢复** | cond_unmask_eoi_irq 直接 unmask+EOI | irq_finalize_oneshot 延迟 unmask |
| **用户态** | evtest → EV_KEY / KEY_0 | evtest → EV_ABS / ABS_X / ABS_Y |

#### 5.18.2 下半部分水岭的实时表现

从顶半部返回到下半部开始执行的**时间差**：

| 场景 | 延迟来源 | 典型延迟 |
|------|---------|---------|
| gpio-keys | ① `mod_delayed_work` 定时器 10ms + ② worker 被调度时间 | 10~30ms |
| 触摸屏 | irq 线程（SCHED_FIFO）几乎立即被调度 | <1ms |
| softirq/tasklet | 在 `irq_exit_rcu` 中直接执行，不经过调度器 | 0 |

**为什么触摸屏需要比按键更高的响应速度？**

人的触觉反馈要求触摸延迟 <10ms 才感觉不到滞后。按键（如 KEY0）用于开关功能，100ms 级延迟人也不会有明显不适。

---

### 5.19 虚假中断检测

#### 5.19.1 note_interrupt 的触发

在 `handle_irq_event_percpu` 中，每次 handler 执行完毕后调用 `note_interrupt`：

```c
// kernel/irq/handle.c:189
irqreturn_t handle_irq_event_percpu(struct irq_desc *desc)
{
    retval = __handle_irq_event_percpu(desc);
    add_interrupt_randomness(desc->irq_data.irq);

    if (!irq_settings_no_debug(desc))
        note_interrupt(desc, retval);  // ★ 虚假中断检测入口
    return retval;
}
```

`note_interrupt` 在 `kernel/irq/spurious.c:272`。它的核心逻辑是**统计连续未处理的中断次数**：

```c
// kernel/irq/spurious.c:272（简化）
void note_interrupt(struct irq_desc *desc, irqreturn_t action_ret)
{
    if (action_ret == IRQ_NONE) {
        // ★ handler 返回 IRQ_NONE → 可能是虚假中断
        desc->irqs_unhandled++;
    }

    if (action_ret == IRQ_HANDLED)
        // ★ 成功处理 → 降低计数（但不能直接清零，防止波动）
        desc->irqs_unhandled--;

    if (desc->irqs_unhandled > THRESHOLD) {
        // ★ 连续未处理超过阈值 → 禁用中断
        __report_bad_irq(desc, action_ret);
        mask_irq(desc);
    }
}
```

在实际运行中，如果某个中断的 handler 连续多次返回 `IRQ_NONE`（或根本没有注册 handler），内核会认为这个中断是**虚假中断（spurious interrupt）**，将其屏蔽并打印：

```
Disabling IRQ #268
```

#### 5.19.2 在 KEY0 场景中的虚假中断

GPIO 按键因为机械弹片的物理特性，**按下和释放在短时间内可能产生多次边沿跳变**（反弹/bounce）。虽然 gpio-keys 驱动内部做了 10ms 的 software debounce，但对 EXTI 控制器来说，每次跳变都会触发一次中断。

如果 debounce 不当，可能出现以下情况：

1. PH5 电平从高→低（按键按下）→ EXTI FPR 置位 → 中断触发
2. handler 执行，`mod_delayed_work`，返回 IRQ_HANDLED
3. 10ms 内 PH5 反弹回高又到低（机械弹片振动）→ 再次中断
4. handler 再次执行，但此时 bdata->work 已经被前一次调度，`mod_delayed_work` 只是**重新设置定时器**，不会导致多次上报

但如果 PH5 的硬件设计有问题——如产生了频繁的噪声跳变——EXTI 检测到的边沿频率可能超过 handler 的处理能力。每次 handler 返回 IRQ_HANDLED，但 EXTI 的 pending 位已经被清掉后又立即被置位——`note_interrupt` 不会认为这是虚假中断（因为它确实被处理了），但 CPU 时间被大量浪费。

#### 5.19.3 在触摸屏场景中的虚假中断

触摸屏的 INT 引脚通常是**电平触发**（而非边沿触发）。如果触摸屏在初始化阶段尚未完成 I2C 通信配置，INT 引脚可能处于不确定状态，导致中断触发后顶半部读状态寄存器返回"无中断"——返回 IRQ_NONE。

连续多次 IRQ_NONE 后，`note_interrupt` 会禁用该中断。这就是为什么触摸屏驱动在 probe() 中需要先完成 I2C 初始化、再 request_irq 的原因。

---

### 5.20 SMP 中断分发

#### 5.20.1 默认 affinity

在 STM32MP257（双核 Cortex-A35）上，所有中断默认的 SMP affinity 是 **CPU0 和 CPU1**（`cpu_possible_mask` 的所有位）。这意味着 GIC Distributor 可以将中断分发给任意核心。

查看当前分配：

```
~# cat /proc/irq/268/smp_affinity
ff
```

`0xff` (bit 0 + bit 1) 表示 CPU0 和 CPU1 都允许接收此中断。

#### 5.20.2 GIC 的硬件分发

GIC-400 通过 **GICD_ITARGETSR** 寄存器配置每个 SPI 的目标 CPU。每个 SPI 对应一个 `GICD_ITARGETSR` 寄存器（4 个字节，每个 CPU 占一个字节）：

```
SPI 273 对应的 GICD_ITARGETSR 偏移 = 0x1800 + (273 - 32) * 4 = 0x1C24
```

`gic_set_affinity` 在 `irq-gic.c` 中通过写这个寄存器实现：

```c
// irq-gic.c — gic_set_affinity（简化）
static int gic_set_affinity(struct irq_data *d, const struct cpumask *mask_val, ...)
{
    // 将 cpumask 转换为 GICD_ITARGETSR 的字节值
    unsigned int cpu = cpumask_any_and(mask_val, cpu_online_mask);
    u32 val = cpu << (cpu * 8);  // 每个 CPU 对应一个字节

    // 写 GICD_ITARGETSR
    writel_relaxed(val, base + GIC_DIST_ITARGETSR + offset);
    ...
}
```

#### 5.20.3 在场景中的实际表现

对于 gpio-keys 的 KEY0 中断（SPI 273），如果两个 CPU 都在空闲状态，GIC 会将中断分发给**先 IAR 读的那个 CPU**——通常是先读到 GICC_IAR 的。这是一种自然的"负载均衡"，不需要软件干预。

但对于需要频繁 I2C 传输的触摸屏，如果中断在两个 CPU 间频繁迁移，可能导致 cache 亲和性下降（irq 线程的数据在 CPU0 的 cache 中，但被分发到 CPU1）。典型优化是将触摸屏中断绑定到指定 CPU：

```shell
echo 1 > /proc/irq/269/smp_affinity    # 只让 CPU0 处理
```

`/proc/irq/N/smp_affinity` 的写入会触发 `gic_set_affinity`，更新 GICD_ITARGETSR 寄存器。之后所有 SPI 273 的中断只发送给 CPU0。

---

### 5.21 调试回顾

#### 5.21.1 第一幕场景的调试手段

当 KEY0 按下后没有响应时，排查路径（基于 01-Usage §1.5）：

```
~# cat /proc/interrupts       ← 检查 virq=268 的计数
  268:          0  stm32mp_exti   5  Edge      gpio-keys
                           ↑ 计数为 0 → 中断没到 CPU

~# devmem 0x4422000C          ← 读 EXTI RPR1（Bank 1 的 RPR）
  按 KEY0 后看 bit 5 是否置位
  0x00000020                  ← bit 5=1 → EXTI 收到了边沿信号

~# devmem 0x44220080          ← 读 EXTI IMR1
  0x00000020                  ← bit 5=1 → 没有被 EXTI 屏蔽

~# devmem 0x4AC10C00          ← 读 GICC_IAR
  (会阻塞等待中断，不推荐在脚本中用)

~# devmem 0x4AC10224          ← 读 GICD_ISPENDR9（SPI 273 对应 bit 17）
  按 KEY0 后看 bit 17 是否变化
```

#### 5.21.2 trace-cmd 观察整条路径

```shell
# 跟踪 KEY0 中断完整路径
trace-cmd record -e irq:irq_handler_entry \
                 -e irq:irq_handler_exit \
                 -e workqueue:workqueue_execute_start \
                 -e workqueue:workqueue_execute_end \
                 -e syscalls:sys_enter_read \
                 sleep 10

# 按 KEY0，然后 Ctrl+C 查看结果
trace-cmd report
```

典型输出：

```
    irq_handler_entry: irq=268 name=gpio-keys
    irq_handler_exit:  irq=268 ret=handled
    workqueue_execute_start: work=gpio_keys_gpio_work_func
    workqueue_execute_end:   work=gpio_keys_gpio_work_func
    sys_enter_read:     fd=3 buf=7f... count=24  ← evtest 读到 input 事件
```

---

### 5.22 全文总结：中断子系统五篇的脉络

至此中断子系统系列全部五篇完成。贯穿所有篇章的一条主线是：**信号路径——中断从外设到 CPU 到用户态的全路径**。

| 篇章 | 主要问题 | 回答 |
|------|---------|------|
| **00-History** | 中断子系统为什么设计成这样？ | 每个机制解决一个历史问题 |
| **01-Usage** | 驱动开发者怎么用中断？ | DTS + API + 调试三板斧 |
| **02-Architecture** | 中断子系统的骨架是什么？ | 四个数据结构、四层架构 |
| **03-SourceAnalysis** | 系统启动后中断怎么初始化？ | GIC→EXTI→GPIO domain 的初始化链 |
| **04-BottomHalf** | 下半部的四种机制怎么选？ | softirq/tasklet/workqueue/threaded IRQ 的数据结构与调度 |
| **05-Scenario** | 从按键按下到 evtest 代码路径是什么？ | 两幕全景追踪 + 综合话题 |

**一条隐线贯穿始终**：从 00 的"为什么有下半部"，到 02 的 irq_chip 和流控函数，到 04 的四机制详解，到 05 的路径贯通——**中断子系统是一个层层抽象的杰作**。每一层做自己该做的事：硬件层产生信号，GIC 层分发中断，核心层管理生命周期，流控层处理时序，驱动层使用数据，下半部层推迟耗时处理。这种分层使得：换一个中断控制器（GIC-400 → GIC-500），换一种 CPU 架构（ARM64 → RISC-V），甚至换一种下半部机制（tasklet → workqueue），都不影响其他层。

如果你能完整理解 05 篇中 KEY0 按下后从 EXTI FPR 到 evtest 的每一级路径，你就是真正理解了 Linux 中断子系统是如何工作的。


```
PH5 按下（电平变化）
  │
  ├─ 硬件层                                    §5.3
  │    GPIOH pin 5 → EXTI FPR1 bit5 置位 → GIC SPI 273
  │
  ├─ GIC Distributor → CPU Interface           §5.3
  │    GICD_ISENABLER 检查 → GICC_IAR 可读 → nIRQ 断言
  │
  ├─ ARM64 异常入口                            §5.4
  │    VBAR_EL1 → vectors[5] → kernel_ventry → kernel_entry
  │    → el1h_64_irq_handler → __el1_irq → irq_enter_rcu
  │    → do_interrupt_handler(gic_handle_irq)
  │
  ├─ GIC 主循环：gic_handle_irq                §5.5
  │    do { readl(GICC_IAR) → ID=273
  │         generic_handle_domain_irq(domain, 273)
  │    } while(1) 直到 spurious
  │
  ├─ 流控：handle_fasteoi_irq                  §5.6
  │    kstat_incr → handle_irq_event
  │    → cond_unmask_eoi_irq → EXTI EOI(RPR/FPR) → GIC EOI
  │
  ├─ 顶半部：gpio_keys_gpio_isr                §5.7
  │    mod_delayed_work(system_wq, work, 10ms)
  │    return IRQ_HANDLED
  │
  ├─ 中断退出：irq_exit_rcu                    §5.8
  │    preempt_count_sub(HARDIRQ_OFFSET)
  │    no softirq pending → eret
  │
  ├─ 下半部：workqueue（10ms 后）              §5.9
  │    worker 线程被调度
  │    gpio_keys_gpio_work_func
  │    → gpiod_get_value_cansleep(PH5)
  │    → input_event(EV_KEY, KEY_0, state)
  │    → input_sync()
  │
  └─ 用户态：evtest read /dev/input/event0     §5.10
       struct input_event { EV_KEY, KEY_0, 1/0 }
```

