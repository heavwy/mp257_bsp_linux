# Linux 中断子系统 30 年演进史

> 为什么 irq_desc、irq_domain、irq_chip、handle_fasteoi_irq 是今天这个样子？
>
> 每个数据结构背后都对应一个真实的历史问题。本文从 Linux v1.0 到 v6.6，沿着中断子系统的关键节点，追溯**每个机制是在解决什么问题**。
>
> **字数**：约 25,000 字 · **建议阅读时间**：60~90 分钟

---

## 1. 前传：do_IRQ() 超级处理器的时代（v1.0 ~ v2.4）

### 1.1 中断是什么

中断是外设通知 CPU 有事件需要处理的硬件机制。以 STM32MP257 为例：按下开发板上的按键 PH5，GPIO 控制器检测到电平变化，通过 EXTI 边沿检测确认事件，然后经过 GIC 中断控制器向 Cortex-A35 核发送一个 IRQ 信号。CPU 在每条指令执行完毕后都会检查 IRQ 引脚是否有信号——这就是中断的"异步"特性：CPU 不知道何时会发生，但保证不会错过。

CPU 响应中断后自动完成三件事（ARM64 为例，参考 `arch/arm64/kernel/entry.S` 的 vectors）：

1. **保存现场**：将当前程序的 PC、PSTATE 和通用寄存器压入栈，构造 `struct pt_regs`
2. **查向量表**：从 `VBAR_EL1` 寄存器指向的异常向量表中，根据异常类型（IRQ/FIQ/Sync/Error）和来源异常级别（EL0/EL1）找到对应入口
3. **跳转执行**：跳转到 `kernel_ventry` 宏生成的入口代码，最终进入 C 函数 `handle_arch_irq`

这三步完成后，中断处理函数开始执行。但此时 CPU 处于一个特殊状态——**IRQ context（中断上下文）**，有两个硬约束：

**约束一：IRQ context 中不可睡眠。** 原因只有一条：中断 handler 执行期间 IRQ 是关着的。睡眠需要别人唤醒——但唤醒依赖中断（timer 中断、设备中断、IPI），IRQ 关了谁来触发？结果就是永远锁死。

从机制层面看，中断 handler 也没有自己的执行上下文。可调度实体都有自己的 `task_struct` 和独享的内核栈。中断 handler 则是"借用"了当前被打断进程的内核栈——`current` 宏返回的仍然是那个被中断的进程，不是中断自己的。中断发生时的 `pt_regs` 就保存在这个进程的内核栈顶，SP 指向它，中断返回时 CPU 从这个位置恢复寄存器然后 `eret`。

如果在中断中调用了 `schedule()`，调度器的 `switch_to` 会做栈切换：把当前 SP 保存到被中断进程的 `task_struct->thread.sp`，然后加载新进程的 `thread.sp` 到 SP。此时栈已经被换了。等中断 handler 执行到返回指令时，CPU 从 SP 指向的位置读取 pt_regs——但读到的是新进程栈上的随机数据，不是原来的 PC/寄存器。`eret` 跳到错误的地址，内核直接崩溃。

所以 IRQ context 中不能调用任何可能触发 `schedule()` 的函数：
- `copy_from_user()` —— 用户态地址可能未映射，缺页异常入口 `do_page_fault()` 在极少数路径下可能 `schedule()` 等待磁盘 I/O
- `mutex_lock()` —— 发生竞争时走 `__mutex_lock_slowpath()` → `schedule()`
- `kmalloc(GFP_KERNEL)` —— 内存紧张时触发直接回收（direct reclaim），`try_to_free_pages()` → `schedule()`

IRQ context 中唯一安全的锁是 `spinlock`（及其变体 `raw_spinlock`），因为它不睡眠、只忙等待。

**约束二：IRQ 关闭时间过长会丢中断。** IRQ 关闭期间，外设仍然在产生中断信号。GIC 的 Distributor 将对应中断标记为 Pending。问题出在 GIC 和外设层面各自只有有限的状态存储资源：

从 GIC 侧看（以 GIC-400 为例），每个中断的 Pending 状态位有且只有一位。IRQ 关闭期间，同一个中断源来了两次，Pending 位从 0→1（第一次）后保持为 1，第二次边沿无法让 Pending 位再翻转一次。等 CPU 打开 IRQ 后只看到一个 Pending——**第二个中断在 GIC 层面就已经丢掉了**。

从外设侧看更直接。以 UART 为例：串口接收 FIFO 通常只有 16 字节。在 115200 bps、8N1 格式下，每字节传输耗时约 87 µs。如果 IRQ 关闭时间超过 87 µs × 16 ≈ 1.4 ms，FIFO 满了之后新数据直接覆盖旧数据——这已经不是"丢中断"而是"丢数据"了。

**IRQ 关不关，硬件都在工作。你关了门，门外的事件不会停下来等你，而是直接超时消失。**

这两个约束指向同一个结论：**中断处理必须拆为两段。** 顶半部只做最少操作（ack、保存必要数据），立即返回恢复 IRQ 使能；耗时的实际工作在顶半部返回后的适当时机完成。

第 1 章标题中"超级处理器"指的就是 v2.4 内核中的 `__do_IRQ()`——它试图用一个函数包揽所有处理，在设计上就违背了"顶半部必须快"这个根本约束。

### 1.2 早期 Linux 的中断处理模型

Linux v1.0 ~ v2.4 时期的中断模型以 x86 为中心。当时的硬件格局是：

**x86 平台**：Intel 8259A PIC（可编程中断控制器），管理 16 条 IRQ 线（IRQ 0~15）。CPU 通过 INTR 引脚接收中断信号，从数据总线读取中断向量号，然后根据 IDTR 寄存器定位到 IDT（中断描述符表），跳转到对应的处理函数。IRQ 0（定时器）、IRQ 1（键盘）、IRQ 14（IDE 硬盘主通道）是当时最繁忙的中断源。

**SMP 出现后**：Intel 引入了 IO-APIC（I/O 高级可编程中断控制器），最多支持 24 条中断线，并且可以将中断分发到多个 CPU。但内核的中断处理模型并没有跟上硬件的变化——仍然使用一个全局的 `irq_desc` 数组，每个中断号对应一个处理函数。

**非 x86 架构**：ARM、PowerPC、MIPS 等架构各自实现自己的中断入口。ARM 使用 `arch/arm/kernel/irq.c` 中的 `asm_do_IRQ()` 作为中断入口，PowerPC 则使用 `__do_irq()`。这些函数名不同、参数不同、注册方式不同，但核心逻辑类似。

这个时期的中断处理流程可以概括为：

```
硬件中断 → CPU 保存现场 → 架构入口(do_IRQ) → irq_enter()
    → __do_IRQ(irq)    ← 统一中断处理函数
    → irq_exit()        ← 检查 BH
    → 恢复寄存器，返回
```

问题在于，各架构的 `irq_desc` 结构体定义不同、中断请求 API 不同（x86 有 `request_irq()`，ARM 早期用 `setup_arm_irq()`）、中断号分配方式不同（x86 用固定的 ISA 0~15，ARM 用板级 `#define`）。**没有统一的框架**意味着编写跨平台驱动需要在 `#ifdef` 中包裹架构特定的中断注册代码，这使得代码难以维护和移植。

### 1.3 __do_IRQ()：一个函数处理所有类型

v2.4 ~ v2.6 早期版本中，内核在 `arch/i386/kernel/irq.c` 中提供了一个名为 `__do_IRQ()` 的"超级处理器"函数。它的设计意图是**用一个函数处理所有类型的中断**（电平、边沿、简单），通过挂载不同的 `hw_interrupt_type` 结构体来适应不同的中断控制器硬件。

`__do_IRQ()` 的核心逻辑可以概括为三段式：

```c
/* 伪代码 — __do_IRQ() 的设计思路 */
unsigned int __do_IRQ(unsigned int irq, struct pt_regs *regs)
{
    struct irq_desc *desc = irq_desc + irq;

    /* 1. 调用芯片层的 ack 函数确认中断 */
    desc->chip->ack(irq);
    /* 2. 处理中断：遍历 action 链表，调用驱动注册的所有 handler */
    handle_irq_event(irq, regs);
    /* 3. 调用芯片层的 end 函数结束中断 */
    desc->chip->end(irq);

    return 1;
}
```

这个单一的 `__do_IRQ()` 通过 `irq_desc->chip` 中的函数指针（`ack`、`mask`、`mask_ack`、`end` 等）来区分不同的中断控制芯片。x86 的 IO-APIC 和 8259A PIC 挂上各自不同的 `hw_interrupt_type` 结构体，`__do_IRQ()` 在运行时根据挂载的 chip 调用对应的 ack/end 函数。

`hw_interrupt_type` 结构体（即今天的 `irq_chip` 的前身）包含一组函数指针：

```c
/* v2.4 中的 hw_interrupt_type — 每个中断控制器一套 */
struct hw_interrupt_type {
    const char *typename;
    unsigned int (*startup)(unsigned int irq);
    void (*shutdown)(unsigned int irq);
    void (*enable)(unsigned int irq);
    void (*disable)(unsigned int irq);
    void (*ack)(unsigned int irq);         /* ← 确认中断 */
    void (*end)(unsigned int irq);          /* ← 结束中断 */
    void (*set_affinity)(unsigned int irq, cpumask_t dest);
};
```

比较今天 v6.6.78 内核中的 `irq_chip` 结构体（`include/linux/irq.h` L501）：

```c
struct irq_chip {
    const char    *name;
    unsigned int  (*irq_startup)(struct irq_data *data);
    void          (*irq_shutdown)(struct irq_data *data);
    void          (*irq_enable)(struct irq_data *data);
    void          (*irq_disable)(struct irq_data *data);
    void          (*irq_ack)(struct irq_data *data);
    void          (*irq_mask)(struct irq_data *data);       /* ← NEW */
    void          (*irq_mask_ack)(struct irq_data *data);   /* ← NEW */
    void          (*irq_unmask)(struct irq_data *data);     /* ← NEW */
    void          (*irq_eoi)(struct irq_data *data);        /* ← NEW */
    int           (*irq_set_type)(...);
    int           (*irq_set_wake)(...);
    ...
};
```

对比两者可以清楚看到演进方向：v2.4 只有 7 个回调函数（ack 和 end 承担了多重职责），v6.6 扩展到了 20+ 个回调，且每个函数职责单一。

### 1.4 __do_IRQ() 的根本问题

先区分两个概念：

**硬件操作（chip details）** 是操作中断控制器寄存器的动作，例如"往 0x20 端口写 0x20"（向 8259A 发 EOI）、"把 GICD_ICFGR 的 bit N 置 1"（配置触发类型）。这些操作由芯片手册定义，芯片不同操作就不同。

**流控逻辑（flow control）** 是中断处理的策略：什么时候需要屏蔽中断、什么时候需要应答、什么时候结束。电平触发的流控是"先屏蔽 → 应答 → 处理 → 解屏蔽"，因为电平信号一直保持，不屏蔽会连续触发。边沿触发的流控是"只应答 → 处理"，因为边沿信号只存在一瞬间，不会重复触发。

`__do_IRQ()` 的问题在于把流控逻辑硬塞进了 `ack()` 和 `end()` 这两个硬件操作函数里。同一个 `ack()` 在不同芯片实现中做的事情完全不同：

- 对 8259A PIC 的 ack：发 EOI（纯硬件操作，不需要 mask）
- 对 IO-APIC 边沿触发的 ack：清除 IRR 位（纯硬件操作）
- 对 IO-APIC 电平触发的 ack：**先屏蔽中断，再清除 IRR**（这里是电平流控需要的 mask，被塞进了 ack 里）

`chip->end()` 也同样混乱：
- 对 8259A PIC：什么也不做
- 对 IO-APIC 边沿触发：什么也不做（硬件自动完成）
- 对 IO-APIC 电平触发：**解屏蔽**（之前 ack 里 mask 了，end 里必须 unmask——这里藏着电平流控需要的 unmask）

关键问题在于：**电平中断和边沿中断的处理差异不是芯片差异，而是流控差异**。一个芯片（如 IO-APIC）既支持电平也支持边沿，两种触发方式下的 ack/end 行为完全不同。但 `__do_IRQ()` 把流控策略写在芯片驱动的 ack/end 里，导致同一套代码里流控逻辑和硬件操作纠缠不清：

```
__do_IRQ() 内部充满了 if/else 判断：
    if (type == LEVEL) {
        chip->mask(irq);       ← 在 ack 中隐式做了
        chip->ack(irq);
        handle_irq_event(irq);
        chip->unmask(irq);     ← 在 end 中隐式做了
    } else if (type == EDGE) {
        chip->ack(irq);        ← 只 ack，不 mask
        handle_irq_event(irq);
        /* 不需要 unmask */
    }
```

这种设计导致：
- 每个中断控制器的驱动代码中，实际流控逻辑和硬件操作代码交织在一起
- 不同驱动之间重复实现相同的电平/边沿处理模式
- 新增一种流控方式（如 Fast EOI）需要修改所有驱动

内核文档 `Documentation/core-api/genericirq.rst` 中如此评价这段历史：

> The original implementation of interrupt handling in Linux uses the __do_IRQ() super-handler, which is able to deal with every type of interrupt logic.
>
> — Documentation/core-api/genericirq.rst

"能处理所有类型"的另一面是"每个类型都处理得不够干净"。

### 1.5 下半部机制的起源：Bottom Half (BH) 的诞生

1.1 节的两个约束（不可睡眠、丢中断）在 v2.1 之前一直没有一个机制化的解决方案。顶半部里的工作全在 IRQ 关闭的上下文中同步完成，中断一密集系统就扛不住。

v2.1 引入了第一个下半部机制，直接命名为 **BH (Bottom Half)**。它的思路是：顶半部只做最少操作（确认中断、把数据搬到内存队列），然后标记一个 BH 为"待处理"；中断返回前，内核检查到有 BH 待处理就执行它，完成剩余工作后再返回被中断的程序。这样 IRQ 可以尽早恢复使能，解决丢中断问题。

BH 的执行时机在 `irq_exit()` → `do_bottom_half()`，走的是中断返回路径。此时 CPU 仍然在被打断进程的内核栈上，`pt_regs`（保存着被中断程序的 PC、PSTATE 和寄存器）就压在这个栈顶，SP 指向它，等着最后被恢复然后 `eret` 返回。

如果 BH 里调用了 `schedule()`，`switch_to` 会把 SP 切换到新进程的栈——原来保存 pt_regs 的那个栈被抛弃了。等 BH 执行完回到中断返回路径，CPU 从 SP 读取 pt_regs 时读到的全是新进程栈上的随机数据，`eret` 跳到错误地址，内核崩溃。

所以 **BH 不可睡眠**——栈上还压着中断返回的现场，一睡就没人能回来了。

BH 的核心数据结构是全局的，所有 CPU 共享：

```c
/* 全局 BH 状态 */
static unsigned long bh_active;          /* 位图：标记哪些 BH 待处理 */
static unsigned long bh_mask;            /* 位图：标记哪些 BH 已注册 */
static void (*bh_base[32])(void);        /* 最多 32 个处理函数指针 */
```

注册一个 BH：在头文件里枚举一个新编号，调用 `init_bh(nr, handler)` 填入 `bh_base`。
触发一个 BH：在顶半部末尾调用 `mark_bh(nr)`，将 `bh_active` 对应位置 1。
执行 BH：每次中断返回时在 `irq_exit()` 中检查：

```c
__do_IRQ(irq, regs)
    → ...
    → irq_exit()
        → if (bh_active & bh_mask)
            do_bottom_half();
```

v2.4 内核预定义的 BH 枚举（`include/linux/interrupt.h`）：

```c
enum {
    TIMER_BH,       /* 定时器 */
    CONSOLE_BH,     /* 控制台输出 */
    SERIAL_BH,      /* 串口接收 */
    BLOCK_BH,       /* 块设备 I/O 完成 */
    IMMEDIATE_BH,   /* 最高优先级 BH */
    KEYBOARD_BH,    /* 键盘输入 */
    ...
};
```

BH 的致命缺陷使它在 SMP 时代迅速成为瓶颈：

| 问题 | 表现 | 后果 |
|------|------|------|
| **固定 32 个** | 槽位在头文件里枚举，新增需改内核头文件并重新编译 | 子系统开发者无法动态注册新的 BH |
| **全局互斥** | `do_bottom_half()` 在全局锁 `global_bh_lock` 保护下执行，同一时刻只有一个 CPU 可以执行 | SMP 扩展性极差，多核系统上 BH 完全串行化 |
| **不可嵌套** | BH 执行期间禁止被另一个 BH 抢占 | 高优先级 BH 必须等待低优先级 BH 完成 |

SMP 普及后，BH 的全局锁成了最大性能瓶颈——**所有 CPU 都在抢一把锁，但同一时刻只能有一个 CPU 干活。** 这个锁的争用在中断密集场景下可以吃掉 30% 以上的 CPU 时间。BH 被 softirq 取代已是必然。

> 今天的 v6.6.78 内核代码中，仍然可以找到 BH 机制的遗存：`include/media/drv-intf/saa7146.h` 中的注释依然写着 `/* for IMMEDIATE_BH */`，尽管 `IMMEDIATE_BH` 这个宏本身早已随 BH 机制一同被移除。

---

## 2. Generic IRQ Layer 的诞生（v2.6.8 ~ v2.6.24）

### 2.1 分裂的源头：Russell King 对 ARM 中断的归纳

ARM 架构是嵌入式设备的代表，中断控制器种类繁多（有简单的 GPIO 中断控制器，有复杂的 GIC）。**2005 年前后**，Linux ARM 维护者 Russell King 在处理 ARM 平台的中断支持时，发现不同硬件的中断流控规律可以归纳为几种基本类型：

| 类型 | 行为特征 | 典型硬件 |
|------|---------|---------|
| **Level type** | 中断信号保持有效电平，直到 CPU 处理完 | 传统 PCI 中断、GPIO 电平检测 |
| **Edge type** | 中断在信号边沿触发一次，不会保持 | GPIO 边沿检测、老式 ISA 中断 |
| **Simple type** | 无硬件操作需求，CPU 检测到中断即可 | 部分内联中断控制器 |

Russell King 为每种类型定义了不同的处理路径，这就是后来 `handle_level_irq()`、`handle_edge_irq()`、`handle_simple_irq()` 的原型。

### 2.2 Thomas Gleixner 的改造：流控与芯片操作分离

2005~2006 年，Thomas Gleixner 和 Ingo Molnar 开始对中断子系统进行系统性的重构（这就是 `kernel/irq/` 目录的起源）。他们的核心思想是：**把「中断流控 (flow handling)」和「芯片操作 (chip details)」拆成两个独立层次**。

**改造前的代码结构（v2.6.12）**：

```
__do_IRQ()                     ← 超级处理器
  ├── desc->chip->ack()        ← 混合了应答 + 屏蔽
  ├── handle_irq_event()       ← 调用驱动 handler
  └── desc->chip->end()        ← 混合了解屏蔽 + EOI
```

**改造后的代码结构（v2.6.18 ~ v2.6.24）**：

```
desc->handle_irq()             ← 流控函数（顶层）
  ├── desc->irq_data.chip      ← irq_chip（底层硬件操作）
  │     ├── irq_ack()          ← 应答中断
  │     ├── irq_mask()         ← 屏蔽中断
  │     ├── irq_unmask()       ← 解屏蔽
  │     └── irq_eoi()          ← 中断结束
  └── handle_irq_event()       ← 调用驱动 handler
```

这个拆分的本质变化是：

- 每个 `irq_desc` 通过 `handle_irq` 函数指针挂载一个**流控函数**（`handle_level_irq`、`handle_edge_irq`、`handle_fasteoi_irq` 等）
- 流控函数只使用 `irq_data.chip` 中的底层原语（`mask/unmask/ack/eoi`）来完成硬件操作
- 驱动开发者不需要关心流控细节，只需调用 `request_irq()` 注册 handler

### 2.3 核心数据结构定型

这个时期的改造定型了沿用至今的两个核心结构体：

**struct irq_chip** — 硬件操作函数表（当前内核 `include/linux/irq.h` L501）：

```c
struct irq_chip {
    const char    *name;
    void    (*irq_ack)(struct irq_data *data);         /* 应答中断 */
    void    (*irq_mask)(struct irq_data *data);         /* 屏蔽中断 */
    void    (*irq_mask_ack)(struct irq_data *data);     /* 屏蔽+应答合并 */
    void    (*irq_unmask)(struct irq_data *data);       /* 解屏蔽 */
    void    (*irq_eoi)(struct irq_data *data);          /* 中断结束(EOI) */
    int     (*irq_set_type)(struct irq_data *data, unsigned int flow_type);
    int     (*irq_set_wake)(struct irq_data *data, unsigned int on);
    int     (*irq_set_affinity)(...);                   /* SMP 路由 */
    ...
};
```

`irq_chip` 的设计模式是 **vtable（虚函数表）**——每个中断控制器提供一个实例，填充自己硬件对应的回调。以 STM32MP257 为例：GIC 的 `chip` 调用 `gic_eoi_irq()` 写 `GICC_EOIR`，GPIO 控制器的 `chip` 则调用 `stm32_gpio_irq_unmask()` 操作 `GPIO_ICR`。上层的 `handle_fasteoi_irq()` 只调用 `chip->irq_eoi()`，完全不知道底层是什么硬件。

相比上一代的 `hw_interrupt_type`（只定义了 `ack`、`end` 两个模糊回调），`irq_chip` 有两大改进：

- **细粒度原语**：mask/unmask/ack/eoi 各司其职。`mask_ack` 合并操作是可选优化——如果不提供，流控函数会分别调用 `mask` 和 `ack`
- **扩展接口**：`set_type` 配置触发方式（电平/边沿），`set_wake` 配置中断能否唤醒系统，`set_affinity` 配置 SMP 路由——这些都是 `hw_interrupt_type` 时代没有的

**struct irq_desc** — 中断描述符（当前内核 `include/linux/irqdesc.h` L55）：

```c
struct irq_desc {
    struct irq_common_data  irq_common_data;
    struct irq_data         irq_data;           /* 核心数据：chip、hwirq、domain */
    irq_flow_handler_t      handle_irq;         /* ← 流控函数指针（关键改动） */
    struct irqaction        *action;            /* 驱动 handler 链表 */
    unsigned int            depth;              /* disable_irq 嵌套深度 */
    unsigned int            wake_depth;         /* enable_irq_wake 嵌套深度 */
    ...
};
```

`irq_desc` 是中断子系统的**中心控制块**。系统中有多少个 IRQ 号，就有多少个 `irq_desc`。关键字段的设计意图：

- **`handle_irq`**：这是 Gleixner 改造的最终落地成果。每个 IRQ 在 setup 时确定流控函数（电平中断挂 `handle_level_irq`，GIC 中断挂 `handle_fasteoi_irq`），运行时直接通过函数指针调用，不再需要 `__do_IRQ()` 那样的超级判断器
- **`action` 链表**：支持中断共享（`IRQF_SHARED`）。同一条 IRQ 线上挂多个设备时，所有 `irqaction` 串成链表依次执行，通过 `action->dev_id` 参数区分具体是哪个设备触发了中断
- **`depth`**：`disable_irq()` 嵌套计数。内核代码中可能存在路径重叠调用 `disable_irq()` 的情况，depth 保证只有等所有调用者都调用了 `enable_irq()` 之后才真正解屏蔽
- **`wake_depth`**：与 `depth` 对称，管理中断作为系统唤醒源的引用计数，`enable_irq_wake()` 和 `disable_irq_wake()` 必须配对调用

**一个重要的演进细节**：v2.6 时代 `irq_desc` 存储在固定大小的 `irq_desc[NR_IRQS]` 数组中。到了 v3.x（commit `9c58bae` 附近），内核改为 radix tree（`irq_desc_tree`）动态分配，不再预占内存。但 `irq_desc` 本身的核心结构维持不变——这本身就说明了它在 v2.6 的设计已足够成熟。

### 2.4 流控函数的定型

Generic IRQ Layer 定义了五种标准流控函数，每种对应一类中断处理场景。以下是从当前 `kernel/irq/chip.c` 中提取的核心逻辑，并绑定了实际硬件场景。

**handle_level_irq（电平触发）**：

```c
void handle_level_irq(struct irq_desc *desc)
{
    raw_spin_lock(&desc->lock);
    mask_ack_irq(desc);             /* ① 屏蔽 + 清 pending */
    handle_irq_event(desc);         /* ② 调 handler（内部释放/重取锁） */
    cond_unmask_irq(desc);          /* ③ 解屏蔽 */
    raw_spin_unlock(&desc->lock);
}
```

电平信号会**持续保持有效**直到外设撤销。ack 虽然清除了 CPU 接口侧的 Pending 状态，但控制器硬件仍然检测到电平有效，立即再次将中断标记为 Pending——**在 handler 还没机会让设备撤销电平之前，下一个中断已经触发了**。如果不 mask，此循环会无限重复，形成中断风暴。所以流程必须是 mask → handle → unmask。而 `raw_spin_lock` 保护的是对 `irq_desc` 的并发访问——同一中断可能在另一个 CPU 上同时操作（如 affinity 迁移），需要保证 `istate`、`action` 链表的修改不被竞争。典型场景：

- **PCI INTx（传统 PCI 中断线）**：PCI 设备通过 INTA/INTB/INTC/INTD 线拉低电平通知中断，直到驱动读写设备寄存器清除中断条件后信号才撤销。多个 `pcie-*` 主机控制器驱动（如 `drivers/pci/controller/pcie-xilinx-nwl.c`）将其子中断注册为 `handle_level_irq`
- **GPIO 电平检测**（如 `drivers/gpio/gpio-brcmstb.c`）：GPIO 引脚配置为电平触发（高电平或低电平有效）时，信号电平持续保持有效，直到外设条件解除。如果不 mask，从 handler 返回后同一电平立即再次触发，形成无限重入——所以必须用 `handle_level_irq` 的 mask/unmask 模式处理

**handle_edge_irq（边沿触发）**：

```c
void handle_edge_irq(struct irq_desc *desc)
{
    raw_spin_lock(&desc->lock);

    desc->irq_data.chip->irq_ack(&desc->irq_data);   /* 清 pending，允许检测下一个边沿 */

    do {
        handle_irq_event(desc);                       /* 调 handler（内部释放/重取锁） */
    } while (desc->istate & IRQS_PENDING);            /* 处理期间又触发了？再来一次 */

    raw_spin_unlock(&desc->lock);
}
```

说明：边沿信号跳变后不会保持，所以只需 ack 清除硬件锁存，不需要像 level 那样 mask。但如果 handler 正在执行时又来了一个同源边沿，内核会在嵌套路径中将 `IRQS_PENDING` 置位，外层 do-while 检测到后立即再处理一次，直到没有新的边沿到达。`raw_spin_lock` 保护 `desc->istate` 的 `IRQS_PENDING` 位操作和 `action` 链表的并发访问。

这个简化的版本省去了异常保护分支（中断禁用、无 action、suspend 等），保留了 edge 流控的核心差异。典型场景：

- **GPIO 边沿检测**（如 `drivers/gpio/gpio-mpc8xxx.c`、`drivers/gpio/gpio-sa1100.c`）：按键按下产生下降沿，松开产生上升沿。如果驱动在顶半部做较长的处理（如去抖），第二次按键可能在 handler 还没返回时就到达，pending 机制保证不丢失
- **PMIC 子中断**（如 `drivers/mfd/max8998-irq.c`, `drivers/mfd/wm831x-irq.c`）：电源管理芯片通过一个 IRQ 线向 SoC 报告多种事件，芯片内部的中断控制器将各路子中断注册为 `handle_edge_irq`

**handle_fasteoi_irq（Fast EOI）——现代中断控制器的标准**：

```c
void handle_fasteoi_irq(struct irq_desc *desc)
{
    raw_spin_lock(&desc->lock);

    handle_irq_event(desc);                        /* ① 调 handler（内部释放/重取锁） */
    desc->irq_data.chip->irq_eoi(&desc->irq_data); /* ② 写 EOI 寄存器 */

    raw_spin_unlock(&desc->lock);
}
```

这是最简洁的模式——**不需要 mask/unmask**。因为高级中断控制器（GIC、MSI）硬件内部维护着中断状态机：CPU 读中断后，GIC 自动将 Distributor 侧对应中断标记为 Pending-Inservice；写 `GICC_EOIR` 后硬件自动清除，允许下次触发。handler 还在跑时同一中断再次到达，GIC 会再次标记 Pending，不会像 level 那样无限递归。`raw_spin_lock` 在此保护的是 affinity 迁移等场景下对 `irq_desc` 的并发访问。

典型场景：
- **ARM GIC**（`drivers/irqchip/irq-gic-v3.c` L598）：`desc->handle_irq = handle_fasteoi_irq` — 所有通过 GIC 路由的中断都使用此模式，包括 STM32MP257 的 GPIO 按键中断、UART 中断、timer 中断等
- **sun4i 中断控制器**（`drivers/irqchip/irq-sun4i.c` L96）

以下是你可以在开发板上验证的实际数据：

```
root@buildroot:~# cat /sys/kernel/debug/irq/irqs/70
handler:  handle_fasteoi_irq
```

每个中断源（包括 GPIO 按键、UART、timer）的 handler 都是 `handle_fasteoi_irq`——因为 STM32MP257 的所有外设中断都通过 GIC 路由。

**handle_simple_irq（简单模式）——无硬件操作需求**：

```c
void handle_simple_irq(struct irq_desc *desc)
{
    raw_spin_lock(&desc->lock);
    handle_irq_event(desc);      /* 唯一做的事：调用驱动 handler */
    raw_spin_unlock(&desc->lock);
}
```

`handle_simple_irq` **不调用任何 `chip->irq_ack/mask/unmask/eoi`**——它只需要执行 handler，不需要操作硬件寄存器。但 `action` 链表和 `istate` 状态的并发保护仍然需要，所以同样持有 `desc->lock`。适用于"子中断"展开场景：物理中断已经被父驱动处理了，子中断只是软件层面的事件。

典型场景——**I2C 接的 GPIO 扩展芯片**（`gpio-pca953x.c`）：

这类芯片有多个 GPIO 引脚可以产生中断，但芯片本身只有**一条 IRQ 线**连到 SoC。芯片驱动注册中断时没有顶半部，只有 threaded handler（`devm_request_threaded_irq(..., NULL, pca953x_irq_handler, ...)`）。发生中断时的流程：

```
PCA9535 某个引脚电平变化
  → 芯片拉低 IRQ 线 → SoC 收到物理中断（硬中断）
      → 内核唤醒 pca953x 驱动的 threaded handler（内核线程）
          → pca953x_irq_handler（线程上下文，可以睡眠）
              → 通过 I2C 读 PCA9535 的输入寄存器
              → 知道哪个(些)引脚发生了变化
              → 对每个变化的引脚，调 handle_nested_irq(该引脚的虚拟 IRQ 号)
                  → 查 irq_desc → handle_simple_irq → 驱动的 handler
```

整个流程从 threaded handler 开始都在内核线程中执行。`handle_nested_irq` 将子中断的 handler 在当前线程上下文中同步调用，不需要走 GIC 硬件。`gpio-pca953x.c`（`drivers/gpio/gpio-pca953x.c` L956-958, L901, L952）正是这样用的。

为什么不需要 ack/mask？因为 **I2C 读寄存器这一步已经把芯片硬件上的中断状态清除了**——PCA9535 的 IRQ 线在寄存器被读取后自动撤销。子中断没有独立的硬件控制器，不需要再操作任何寄存器。

**handle_percpu_irq（每 CPU 中断）——SMP 专用**：

```c
void handle_percpu_irq(struct irq_desc *desc)
{
    if (chip->irq_ack)  chip->irq_ack(&desc->irq_data);   /* 可选 ack */
    handle_irq_event_percpu(desc);                         /* 无锁遍历 action */
    if (chip->irq_eoi)  chip->irq_eoi(&desc->irq_data);   /* 可选 EOI */
}
```

最关键的特征是**无锁设计**：per-CPU 中断只发给一个固定的 CPU，不会跨核心迁移，因此不需要 spinlock 保护——这是 GIC PPI（Private Peripheral Interrupt）的硬件保证。典型场景：

- **CPU 本地 timer 中断**（`drivers/irqchip/irq-mips-cpu.c`、`drivers/irqchip/irq-loongarch-cpu.c`）
- ARM64 架构中 GIC 的 PPI（如 `__IPI`、`arch_timer`）也使用类似的无锁处理方式

---

注意看代码中的 `raw_spin_lock(&desc->lock)`。为什么前四种需要加锁而 percpu 不需要？

`irq_desc` 是共享数据结构，系统中所有中断共用一个 `irq_desc` 数组/树。两个 CPU 可能同时处理不同的中断——但**同一中断**也可能被两个 CPU 同时操作（比如 affinity 迁移期间、或 level 中断在 SMP 上的竞争）。`desc->lock` 保护的是对这个 `irq_desc` 实例的并发访问：修改 `istate` 状态位、遍历/修改 `action` 链表、操作 `depth` 计数等。

per-CPU 中断（PPI）的"免锁"特权来自硬件保证：GIC 的每个 PPI 只发送给一个指定的 CPU，不会跨核心。既然**只有一个 CPU 会访问这个 `irq_desc`**，spinlock 就是多余的。

**五种流控函数的横向对比**：

| 函数 | 是否 spinlock | 是否 mask/unmask | 是否需要 ack | 是否需要 EOI | 典型硬件 |
|------|:---:|:---:|:---:|:---:|---------|
| `handle_level_irq` | ✅ 需要 | ✅ mask_ack + unmask | ✅ | ❌ | PCI INTx、GPIO 电平模式 |
| `handle_edge_irq` | ✅ 需要 | 处理中才 mask | ✅ ack 即可 | ❌ | GPIO 边沿、PMIC 子中断 |
| `handle_fasteoi_irq` | ✅ 需要 | ❌ | ❌ | ✅ EOI | ARM GIC、MSI |
| `handle_simple_irq` | ✅ 需要 | ❌ | ❌ | ❌ | 软件模拟中断、I2C 子中断 |
| `handle_percpu_irq` | ❌ **不需要** | ❌ | 可选 | 可选 | CPU 本地 timer、GIC PPI |

此外，在 v4.x/v5.x 内核中又增加了两种变体：`handle_edge_eoi_irq`（边沿触发 + EOI，用于需要边沿语义的 GICv3 层次域）和 `handle_fasteoi_mask_irq`（需要在 EOI 前 mask，用于部分需要 mask+EOI 顺序的 irq_chip）。这两种变体是对五种基本流控函数的补充，不改变核心分层设计。

**总结**：五种流控函数的差异归结为一个问题——**中断控制器硬件需要软件做什么才能安全地处理下一个中断？** 有的需要 mask/unmask（level），有的只需 ack（edge），有的只需 EOI（fasteoi），有的什么都不需要（simple），有的每个 CPU 各自独立处理（percpu）。流控函数将这种需求差异封装为可替换的函数指针，上层驱动统一调用 `request_irq()`，不需要关心底层差异。

### 2.5 __do_IRQ() 的弃用与移除

Generic IRQ Layer 完成后，`__do_IRQ()` 被标记为 DEPRECATED。各架构逐步迁移到新的流控机制：

- v2.6.24：大多数 x86 中断控制器迁移完成
- v2.6.29：ARM 基本完成迁移
- v3.x 系列：`__do_IRQ()` 最终被移除

当前 v6.6.78 内核的 `kernel/irq/` 目录结构体现了这次改造的全部成果：

```
kernel/irq/
├── manage.c         — 顶层 API（request_irq/free_irq/enable_irq/disable_irq）
├── chip.c           — 流控函数（handle_level_irq/handle_edge_irq/handle_fasteoi_irq）
├── handle.c         — handle_irq_event（action 链表遍历 + 线程唤醒）
├── irqdesc.c        — irq_desc 分配/释放
├── irqdomain.c      — irq_domain（hwirq→virq 映射）
├── devres.c         — devm_request_irq 自动管理
├── proc.c           — /proc/interrupts 接口
├── debugfs.c        — /sys/kernel/debug/irq/ debug 接口
├── affinity.c       — SMP 亲和性管理
├── pm.c             — 休眠/唤醒中断管理
├── spurious.c       — 虚假中断检测
├── resend.c         — 中断重发
├── autoprobe.c      — 中断自动探测
├── timings.c        — 中断时间统计
├── msi.c            — MSI 中断
└── cpuhotplug.c     — CPU 热插拔中断迁移
```

这个结构一直保持到今天的 v6.6.78，19 个 `.c` 文件共约 15,000 行代码，共同构成了 Linux 通用中断处理框架。

---

## 3. 下半部机制的演进（v2.1 ~ v3.2）

中断处理的核心约束是**顶半部必须快**——它在 IRQ 关闭的上下文中运行，任何耗时操作都会阻塞其他中断甚至导致内核延迟。因此，耗时的工作（协议处理、数据拷贝、事件上报）必须推迟到"下半部"执行。

下半部机制在 Linux 历史中经历了四次重大迭代：

### 3.1 BH（Bottom Half）— v2.1 引入

第 1 节已经介绍了 BH 的设计思路和它为什么不可睡眠。这里补充 BH 的具体实现机制和驱动使用方式。

**数据结构**（位于 `kernel/softirq.c`，全局变量，所有 CPU 共享）：

```c
static unsigned long bh_active;          /* 位图：标记哪些 BH 待处理 */
static unsigned long bh_mask;            /* 位图：标记哪些 BH 已注册 */
static void (*bh_base[32])(void);        /* 最多 32 个处理函数指针 */
```

BH 的处理函数被编号为 0~31，编号越大优先级越高（注意：不是越小越高）。`bh_active` 的 bit N 为 1 表示编号 N 的 BH 有待处理工作，`bh_mask` 的 bit N 为 1 表示编号 N 的 BH 已注册了 handler。

**驱动使用 BH 的标准 API 和流程**：

```c
/* 1) 在 include/linux/interrupt.h 中枚举 BH 编号 */
enum {
    TIMER_BH = 0,
    CONSOLE_BH,
    SERIAL_BH,
    BLOCK_BH,
    IMMEDIATE_BH = 28,
    KEYBOARD_BH = 29,
    ...
};

/* 2) 驱动初始化时注册 handler — 填入 bh_base[nr] 并设 bh_mask 对应位 */
init_bh(SERIAL_BH, serial_bh_handler);

/* 3) 顶半部末尾触发 BH */
irqreturn_t serial_irq_handler(int irq, void *dev_id)
{
    /* 从硬件接收 FIFO 搬运数据到内存缓冲区，硬件操作越短越好 */
    ... 
    mark_bh(SERIAL_BH);  /* bh_active |= BIT(SERIAL_BH) — 告诉内核：SERIAL_BH 有待处理工作 */
    return IRQ_HANDLED;
}

/* 4) 可选：临时禁用/重新启用某个 BH（带嵌套计数 bh_mask_count[nr]） */
disable_bh(SERIAL_BH);   /* bh_mask &= ~BIT(nr) */
enable_bh(SERIAL_BH);    /* 减计数到 0 后 bh_mask |= BIT(nr) */

/* 5) 驱动卸载时移除 */
remove_bh(SERIAL_BH);    /* bh_base[nr] = NULL; bh_mask &= ~BIT(nr) */
```

**`do_bottom_half()` 的执行逻辑**——优先级最高的 BH 最先执行：

```c
void do_bottom_half(void)
{
    unsigned long active, pending;
    int i;

    /* 只取已注册且有挂起的 BH */
    active = get_active_bhs();          /* bh_active & bh_mask */

    raw_spin_lock_irq(&global_bh_lock);  /* ← 全局锁：同一时刻只有一个 CPU 能进来 */

    do {
        /* 找到最高优先级的待处理 BH（编号最大的） */
        i = 31 - __builtin_clz(active);   /* fls(active) - 1 */
        
        /* 清除该位，防止重复执行 */
        bh_active &= ~BIT(i);
        
        /* 释放锁 — 注意：执行 BH 期间其他 CPU 可以竞争这把锁 */
        raw_spin_unlock_irq(&global_bh_lock);
        
        /* 执行 BH handler */
        bh_base[i]();                      /* 同步调用，执行完才返回 */
        
        /* 重新上锁检查是否还有待处理工作 */
        raw_spin_lock_irq(&global_bh_lock);
        active = bh_active & bh_mask;
    } while (active);

    raw_spin_unlock_irq(&global_bh_lock);
}
```

这个设计有两个关键行为：

1. **优先级反转隐患**：执行 BH[N] 期间，如果有更高优先级的 BH[M]（M > N）被 `mark_bh` 触发，必须等当前 BH[N] 返回后才能重新进入 `do_bottom_half()` 扫描到 BH[M]——因为 `global_bh_lock` 是互斥的
2. **释放锁窗口**：执行 `bh_base[i]()` 前释放了锁，其他 CPU 可以 `mark_bh` 并竞争锁，这是 SMP 下唯一的并发通道——但同一时刻仍然只能有一个 CPU 在执行 BH 代码

**真实世界的 BH 使用——SERIAL_BH 例子**：

串口驱动是 BH 的典型用户。以 16550 UART 为例，接收中断到来时顶半部只做一件事：从 `RBR` 寄存器读取收到的字节，放入一个内存缓冲区，然后 `mark_bh(SERIAL_BH)`。真正的协议处理（行规程、tty 层分发）在 SERIAL_BH 中完成：

```
串口接收中断（顶半部，IRQ 关闭）
  → read RBR → push to flip buffer → mark_bh(SERIAL_BH) → return
中断返回前 irq_exit() 检查 bh_active
  → do_bottom_half()
      → lock global_bh_lock
      → 扫描最高优先级 BH（可能是 SERIAL_BH，也可能是 TIMER_BH）
      → serial_bh_handler()
          → 遍历 flip buffer → 按 tty 行规程处理 → wake up 读进程
      → unlock global_bh_lock
```

这个模式的缺点是：串口收到的所有字节都在一个"原子上下文"中处理（不可睡眠），所以 tty 层不能做内存分配、不能等待——一旦 flip buffer 满而读进程还没来取走，数据就会丢失。

**SMP 下的性能灾难——global_bh_lock 实测数据**：

当你有一个 4 核 CPU 同时处理网络和磁盘中断时，`global_bh_lock` 的争用场景：

| CPU 0 | CPU 1 | CPU 2 | CPU 3 |
|-------|-------|-------|-------|
| NET_BH 执行中 | 等待 global_bh_lock | 等待 global_bh_lock | 等待 global_bh_lock |
| 持有锁 | 自旋 | 自旋 | 自旋 |

三个 CPU 在空转等一把锁。更糟的是，顶半部 (`mark_bh`) 也需要竞争 `global_bh_lock`，这意味着**顶半部也在等锁**——中断延迟因此大幅增加。

**BH 的完整局限回顾**：

| 问题 | 表现 | 后果 |
|------|------|------|
| 固定 32 个 | 头文件里枚举，新增需改内核 | 子系统开发者无法动态注册新的下半部 |
| 全局互斥 | `global_bh_lock` 串行化所有 CPU | SMP 扩展性极差，多核上 BH 完全串行 |
| 不可嵌套 | 一次只执行一个 BH | 高优先级 BH 必须等低优先级的执行完 |
| 不可睡眠 | `pt_regs` 在中断栈顶 | 无法处理需要等待 I/O 或内存分配的逻辑 |

**BH 机制的遗产**：

BH 本身在 v2.5 开发周期中被 softirq 取代，但它的概念留下了几个至今可见的影响：

1. 软中断使用的 `__softirq_pending` 位图机制直接继承了 BH 的位图标记思路
2. `HI_SOFTIRQ` 和 `TASKLET_SOFTIRQ` 这两个软中断向量就是为兼容旧的 BH 语义而保留的（高优先级下半部和普通下半部）
3. 即使在 v6.6.78 内核中，仍有驱动通过头文件注释引用 `IMMEDIATE_BH`（如 `include/media/drv-intf/saa7146.h`），显示 BH 枚举多年来曾是内核 ABI 的一部分

### 3.2 Softirq — v2.3 重写（1999 年）

1999 年，Alexey Kuznetsov（ANK）几乎重写了整个下半部系统。这次重写的核心目标是**彻底甩掉 BH 的全局锁瓶颈**。softirq.c 头部注释毫不留情：

```c
/* linux/kernel/softirq.c
 * Rewritten. Old one was good in 2.2, but in 2.3 it was immoral. --ANK (990903)
 */
```

**对比 BH 解决的核心问题**：

| | BH (v2.1) | Softirq (v2.3) |
|--|-----------|----------------|
| pending 位图 | 全局 `bh_active`，所有 CPU 共享 | 每 CPU 的 `irq_stat.__softirq_pending` |
| handler 表 | 全局 `bh_base[32]` | 全局 `softirq_vec[NR_SOFTIRQS]` |
| 互斥 | `global_bh_lock` 全局锁 | **无锁** —— 每个 CPU 独立执行 |
| 调用 context | 中断返回路径，IRQ 关闭 | 中断返回路径，**IRQ 打开** |
| 数量限制 | 32 个硬编码 | 10 个（可通过 `NR_SOFTIRQS` 扩展） |

**核心数据结构**（当前 v6.6.78 `kernel/softirq.c` L59, `include/linux/interrupt.h` L520-528）：

```c
/* 全局函数表：一个向量一个 handler */
static struct softirq_action softirq_vec[NR_SOFTIRQS] __cacheline_aligned_in_smp;

/* 每 CPU pending 位图 —— 不再有全局 bh_active */
DECLARE_PER_CPU(struct irq_stat, irq_stat);
/* → 实际通过 local_softirq_pending() 访问 */

#define local_softirq_pending()     __this_cpu_read(irq_stat.__softirq_pending)
#define set_softirq_pending(x)      __this_cpu_write(irq_stat.__softirq_pending, (x))
#define or_softirq_pending(x)       __this_cpu_or(irq_stat.__softirq_pending, (x))
```

对比 BH：BH 所有 CPU 共享一个 `bh_active`，操作它需要 `global_bh_lock`。Softirq 把它改为每 CPU 一份，**写自己的位图不需要锁**——这就把 BH 时代最严重的 SMP 瓶颈直接消除了。

**注册一个 softirq handler**（`kernel/softirq.c` L709-712）：

```c
void open_softirq(int nr, void (*action)(struct softirq_action *))
{
    softirq_vec[nr].action = action;
}
```

对比 BH 的 `init_bh(nr, handler)` 还需要操作 `bh_mask`。Softirq 的 `open_softirq` 只填一个函数指针，**没有 enable/disable 机制**——softirq 一旦注册就是永远可用的（对应 BH 的 `open_bh` 不存在了）。

**触发一个 softirq**（`kernel/softirq.c` L676-707）：

```c
/* 核心操作：在 per-CPU 位图上设一个 bit */
void __raise_softirq_irqoff(unsigned int nr)
{
    trace_softirq_raise(nr);
    or_softirq_pending(1UL << nr);     /* per-CPU 位图操作，无需锁 */
}

/* 如果在中断上下文外触发，还需要唤醒 ksoftirqd */
inline void raise_softirq_irqoff(unsigned int nr)
{
    __raise_softirq_irqoff(nr);
    if (!in_interrupt() && should_wake_ksoftirqd())
        wakeup_softirqd();             /* 当前不在中断中 → 唤醒线程来跑 */
}

/* 从非原子上下文调用的安全版本 */
void raise_softirq(unsigned int nr)
{
    unsigned long flags;
    local_irq_save(flags);             /* 关 IRQ，因为 __raise 要求 IRQ disabled */
    raise_softirq_irqoff(nr);
    local_irq_restore(flags);
}
```

**重点理解 `raise_softirq` 的惰性调度策略**：

- 如果在中断 handler 或另一个 softirq 中调用 → 只设 bit 就返回。因为中断/softirq 返回前会检查 pending，自然会执行
- 如果在进程上下文中调用（如驱动在系统调用中触发）→ 设 bit + 唤醒 `ksoftirqd/N`。因为进程上下文没有 `irq_exit` 那种"每次中断返回后自动检查"的机制，只能靠 ksoftirqd 来兜底

这就是 softirq 相比 BH 的关键改进：BH 只有 `irq_exit → do_bottom_half` 一条执行路径（如果错过了就等下一次中断），而 softirq 有两条互补路径——中断返回时在 `irq_exit` 中直接处理，加上进程上下文也可以通过 `ksoftirqd` 来兜底。

**Softirq 的执行路径：irq_exit → invoke_softirq**

在 ARM64 架构上，每次硬件中断处理完毕后会调用 `irq_exit()`，路径如下（`kernel/softirq.c` L597-691）：

```c
/* irq_exit_rcu — 中断出口门卫 */
void irq_exit_rcu(void)
{
    __irq_exit_rcu();                   /* ← 检查并执行 softirq */
    lockdep_hardirq_exit();
}

static void __irq_exit_rcu(void)
{
    if (!in_interrupt() && local_softirq_pending())
        invoke_softirq();               /* 有 pending → 处理 */
}

/* invoke_softirq — 决定是直接执行还是唤醒线程 */
static inline void invoke_softirq(void)
{
    if (!force_irqthreads()) {
        __do_softirq();                 /* ← 直接在当前栈上执行 */
    } else {
        wakeup_softirqd();              /* CONFIG_IRQ_FORCED_THREADING */
    }
}
```

这个检查中的 `!in_interrupt()` 是关键保护：如果当前已经嵌套在另一个 softirq 中（`in_interrupt()` 为 true），就不递归执行，而是让外层返回后再处理。

**`handle_softirqs()` — 实际处理函数**（`kernel/softirq.c` L517-590，v6.6.78）：

```c
static void handle_softirqs(bool ksirqd)
{
    unsigned long end = jiffies + MAX_SOFTIRQ_TIME;       /* 超时：2ms */
    unsigned long old_flags = current->flags;
    int max_restart = MAX_SOFTIRQ_RESTART;                 /* 最大重试：10 次 */
    struct softirq_action *h;
    __u32 pending;
    int softirq_bit;

    pending = local_softirq_pending();                     /* 快照当前 pending */

restart:
    /* 清空 pending 位图，然后打开 IRQ */
    set_softirq_pending(0);
    local_irq_enable();                                    /* ← IRQ 使能！ */

    h = softirq_vec;
    /* ffs: find first set — 最低编号的软中断优先处理（0=HI_SOFTIRQ 优先级最高） */
    while ((softirq_bit = ffs(pending))) {
        unsigned int vec_nr = (h + (softirq_bit - 1)) - softirq_vec;
        h += softirq_bit - 1;

        h->action(h);                                      /* ← 调用软中断 handler */
        h++;
        pending >>= softirq_bit;
    }

    /* 处理完后，关 IRQ 检查是否又有新的 pending 了 */
    local_irq_disable();
    pending = local_softirq_pending();
    if (pending) {
        /* 没超时、没人请求 resched、还没重试满 10 次 → 再来一轮 */
        if (time_before(jiffies, end) && !need_resched() && --max_restart)
            goto restart;
        /* 否则交给 ksoftirqd 兜底 */
        wakeup_softirqd();
    }
}
```

这个设计中有几个关键决策：

1. **ffs（find first set）编号越小优先级越高**。`HI_SOFTIRQ=0` 优先于 `TIMER_SOFTIRQ=1` 优先于 `NET_TX_SOFTIRQ=2`……这和 BH 的"编号越大优先级越高"正好相反
2. **`local_irq_enable()` — softirq 在 IRQ 打开下执行**。这点 BH 和 softirq 一致（BH 在调 handler 前也释放了锁），但两者在并发模型上有根本区别：

   - **BH**：`do_bottom_half()` 扫描位图和调 handler 期间持有 `global_bh_lock`，同一时刻全局只有一个 CPU 在执行 BH。handler 执行期间 IRQ 打开的，但其他 CPU 进不来
   - **Softirq**：各 CPU 独立处理自己的 pending 位图，**完全无锁并行**。`handle_softirqs()` 先将位图清空（快照），再遍历处理，执行完后重新读取 pending——如果 handler 执行期间同类型 softirq 又被触发（如新网络中断调了 `__raise_softirq_irqoff(NET_RX`），会在下一轮 restart 或 ksoftirqd 中处理，不会在当前 handler 中间嵌套重入
3. **重启限制**：最多重试 10 次或 2ms，超限后唤醒 ksoftirqd 接管——防止 softirq 占着 CPU 不放导致用户态进程饿死

**ksoftirqd — per-CPU 内核线程兜底**（`kernel/softirq.c` L74-81）：

```c
DEFINE_PER_CPU(struct task_struct *, ksoftirqd);

static void wakeup_softirqd(void)
{
    struct task_struct *tsk = __this_cpu_read(ksoftirqd);
    if (tsk)
        wake_up_process(tsk);         /* 唤醒当前 CPU 的 ksoftirqd/N 线程 */
}
```

每个 CPU 有一个名为 `ksoftirqd/N`（N=CPU 编号）的内核线程。当 softirq 负载过高（10 次重试都处理不完）或不在中断上下文中触发时，由这个线程来执行剩余的 softirq：

```
ksoftirqd/N 线程的主循环：
  ksoftirqd_run_begin()     → 标记入 softirq context
  handle_softirqs(true)     → 处理 pending 的 softirq
  ksoftirqd_run_end()       → 退出 softirq context
  schedule()                → 让出 CPU 给其他进程
```

因为 ksoftirqd 运行在进程上下文中，它可以被调度——如果 softirq 做不完，下次时间片到了继续做。这样就不会像 BH 那样只要还有 pending 就卡在中断返回路径上不走。

**Softirq 的 10 个向量**（当前 v6.6.78 `include/linux/interrupt.h` L550）：

```c
enum {
    HI_SOFTIRQ = 0,         /* 高优先级 tasklet */
    TIMER_SOFTIRQ,          /* 定时器回调 */
    NET_TX_SOFTIRQ,         /* 网络发包完成 */
    NET_RX_SOFTIRQ,         /* 网络收包 */
    BLOCK_SOFTIRQ,          /* 块设备 I/O 完成 */
    IRQ_POLL_SOFTIRQ,       /* 中断驱动的轮询（NAPI 相关） */
    TASKLET_SOFTIRQ,        /* 普通 tasklet */
    SCHED_SOFTIRQ,          /* 调度器负载均衡 */
    HRTIMER_SOFTIRQ,        /* 高精度定时器到期 */
    RCU_SOFTIRQ,            /* RCU 回调处理 */
    NR_SOFTIRQS             /* = 10 */
};
```

以典型的网络收包路径为例，展示完整的 softirq 触发和处理流程：

```
网卡硬件中断
  → 顶半部（关 IRQ，极短）
      → napi_schedule() → __raise_softirq_irqoff(NET_RX_SOFTIRQ)  ← 只设一个 bit
      → 返回
  → irq_exit_rcu()
      → invoke_softirq()
          → __do_softirq()
              → handle_softirqs(false)
                  → local_irq_enable()                              ← 打开 IRQ！
                  → softirq_vec[NET_RX_SOFTIRQ].action() 即 net_rx_action()
                      → 从 ring buffer 批量收包
                      → 协议栈处理（IP → TCP/UDP）
                      → wake up 用户态 socket 等待进程
                  → 检查是否还有 pending
                  → 没有 → 返回
```

这个路径的关键改进：**整个收包过程（从驱动到协议栈）都在 IRQ 打开下执行**。BH 时代这个过程是 IRQ 关闭的，UART FIFO 满覆盖的问题在 softirq 中被彻底消除。

**总结 softirq 相对 BH 的核心改进**：

1. **每 CPU pending 位图** → 消除了 `global_bh_lock`，SMP 性能大幅提升
2. **IRQ 打开执行** → 不再有"关 IRQ 丢中断"的风险
3. **ksoftirqd 兜底** → softirq 不限制死 CPU，高负载下能优雅降级
4. **ffs 优先级** → 编号越小优先级越高，比 BH 的 fls 更直观（和硬件中断优先级编号习惯一致）

但 softirq 也有一个重要约束：**同一个 softirq 在同一个 CPU 上不会嵌套执行**。防重入的机制在 `__irq_exit_rcu` 中的 `!in_interrupt()` 检查：

```c
static void __irq_exit_rcu(void)
{
    if (!in_interrupt() && local_softirq_pending())
        invoke_softirq();
}
```

当 CPU 0 正在 `handle_softirqs()` 中执行 `net_rx_action` 时，`preempt_count` 中设置了 `SOFTIRQ_OFFSET`，`in_interrupt()` 返回 true。此时新网络中断的顶半部处理完后调用 `irq_exit`，因为 `in_interrupt()` 为 true，会**跳过** `invoke_softirq`，直接返回到被中断的 `net_rx_action` 中。新触发的中断只设置了 pending bit，等当前这轮处理完后下轮 restart 再处理。

这意味着 **softirq 没有重入问题（同 CPU 不嵌套），但有并发问题（多核并行）**。解决并发问题的策略就是 ANK 原则的前两条：用 per-CPU 数据避免共享，必须共享时用锁保护。这正是 ***No shared variables, all the data are CPU local*** 原则的体现。

### 3.3 Tasklet — 基于 Softirq 的简化封装

Tasklet 是 softirq 之上的简化封装，解决 softirq 的一个痛点：softirq 必须在编译时静态注册（在 `enum` 里加一项），驱动无法动态创建新的下半部。

另一个区别在于并发模型的选择：softirq 的同一个 handler 可以在多个 CPU 上同时执行——这对网络、块设备等高性能子系统是优势，但驱动开发者需要为共享数据做保护。Tasklet 选择了相反的模型：通过 `tasklet_trylock(RUN)` 保证同一个 tasklet 实例不会被两个 CPU 同时执行。这么设计的出发点是因为 tasklet 的目标用户是普通驱动（如 GPIO 按键、I2C 消息），这类场景不需要多核并行，开发便利性更重要

**设计思路**：用两个 softirq 向量（`TASKLET_SOFTIRQ` 和 `HI_SOFTIRQ`）来调度一个 per-CPU 链表，驱动只需向链表中添加节点，无需关心并发细节：

```
TASKLET_SOFTIRQ 软中断 handler → tasklet_action()
  → 遍历当前 CPU 的 tasklet_vec 链表
    → 对每个节点执行回调函数
    → tasklet_trylock(RUN bit) 保证同实例不并行
```

**核心串行化机制**：每个 tasklet 实例的 `state` 字段有两个标记位——`SCHED`（已入队）和 `RUN`（正在执行）。两者的配合方式如下：

```
CPU 0                                   CPU 1
────                                    ────
tasklet_schedule(t)
  test_and_set_bit(SCHED) → 0→1 (成功)
  入队 + raise TASKLET_SOFTIRQ
                                        tasklet_schedule(t)
                                          test_and_set_bit(SCHED)
                                          → 已经是 1（失败，跳过）
                                          → 同一个 tasklet 不会重复入队

[软中断处理]
tasklet_action_common():
  摘下链表 → 遍历 → 遇到 t
  tasklet_trylock(t):
    test_and_set_bit(RUN) → 0→1 (成功)
  清除 SCHED → 执行 t->func()
                                          [另一个 CPU 尝试调度 t]
                                          tasklet_schedule(t)
                                            test_and_set_bit(SCHED) → 0→1 (成功)
                                            入队
                                          [同 CPU 的软中断处理]
                                          tasklet_action_common():
                                            遇到 t
                                            tasklet_trylock(t):
                                              test_and_set_bit(RUN)
                                              → 还是 1（CPU 0 没释放）
                                              → 失败！
                                            重新入队 + 再触发 softirq

  t->func() 返回
  tasklet_unlock(t):
    clear_bit(RUN)
                                          [下次 softirq 处理]
                                          tasklet_trylock(RUN) → 1→0 后成功
                                          执行 t->func()
```

关键点：如果 `RUN` 竞争失败，当前 CPU 不是自旋等待（不像 spinlock），而是**重新入队 + 触发 softirq 下次再试**。这样不会浪费 CPU 时间，也给了另一个 CPU 上的 tasklet 完成执行的机会。

**Tasklet 与 Softirq 的对比**：

| | Softirq | Tasklet |
|--|---------|---------|
| 注册方式 | 编译时静态 | 运行时动态 |
| 并发模型 | 多核并行 | 同实例串行（`tasklet_trylock`） |
| 性能 | 高（无额外同步） | 稍低（原子操作开销） |

**演进地位**：Tasklet 是 BH → Softirq 路线上的中间产物——它保留了 softirq 的不可睡眠约束（运行在 softirq context），但提供了运行时动态注册能力。下一个机制 workqueue 将彻底解决"不可睡眠"这个根本限制。

### 3.4 Workqueue — v2.5 引入 → v3.2 CMWQ 重写

Workqueue 与 tasklet 的最大区别是可睡眠能力：**tasklet 在 softirq 上下文中执行（不可睡眠），workqueue 在进程上下文（可睡眠）**。驱动中那些需要等待 I/O、申请内存、拿 mutex 的操作，只能用 workqueue 完成。

#### 原始 workqueue（v2.5 ~ v3.1）——一个队列一个线程

最早的 workqueue 设计很简单：每个 workqueue 有自己的内核线程，线程从队列中取 work 依次执行。

```
workqueue_a → 内核线程 events/0 → work1 → work2 → [work3 阻塞了] → ...（后面全等）
workqueue_b → 内核线程 events/1 → work4 → work5 → work6 → ...
```

问题一：**相互阻塞**。work1 调用 `msleep(100)` 或等某个 mutex，同一个队列里的 work2、work3 全被堵住。

问题二：**线程浪费**。`events/N` 线程池固定创建 `NR_CPUS` 个线程（通常是 4 或 8）。但实际只有少数几个 work 同时在跑，多余的线程占着栈空间（每个内核线程至少 4KB 栈）什么都不做。

#### v3.2 CMWQ——将"队列"和"线程池"解耦

Tejun Heo 的思路是区分两个概念：

- **`workqueue_struct`**：你创建的"工作队列"，是对外接口。内核有 300 多个子系统各自创建了自己的 workqueue
- **`worker_pool`**：实际干活的线程池。系统只有少数几个 pool（每个 CPU 两个：普通优先级 + 高优先级）

一个 workqueue 不拥有自己的线程，而是通过 `pool_workqueue` 这个桥接结构，**借用**线程池里的 worker 来执行 work：

```
驱动 A 调用               驱动 B 调用
queue_work(wq_a, ...)     queue_work(wq_b, ...)
       │                         │
       ▼                         ▼
   wq_a ──── pwq ────┐      wq_b ──── pwq ────┐
                     │                        │
                     ▼                        ▼
               worker_pool[CPU0]          worker_pool[CPU1]
               ├── worker                 ├── worker
               ├── worker                 ├── worker
               └── ...                    └── ...
```

这样设计的好处：

1. **线程复用**：300 个 workqueue 不再需要 300 个线程，全部共享 per-CPU 的 pool（每个 CPU 只有 2 个 pool：普通 + 高优先级）。系统只创建真正需要的 worker 数量

2. **并发管理（Concurrency Managed）**：pool 根据当前正在执行的 work 数量自动调整 worker 数量。如果一个 work 阻塞了，pool 检测到活跃 worker 减少，会创建新的 worker 来接替——这样阻塞不会卡住整个队列

3. **work 和线程解耦**：驱动 A 的 work 阻塞了，只阻塞它借用的那个 worker。pool 中的其他 worker 可以继续执行其他 workqueue 的任务

#### 实际如何使用

对于大多数驱动而言，最常用的方式是用内核预定义的共享 workqueue：

```c
/* 最常用：借用 system_wq */
schedule_work(&my_work);          /* ≡ queue_work(system_wq, &my_work) */
schedule_delayed_work(&my_dwork, delay);  /* 延迟执行 */

/* 性能关键或需要隔离：创建专用 workqueue */
static struct workqueue_struct *my_wq;
my_wq = alloc_workqueue("my_driver", WQ_UNBOUND | WQ_HIGHPRI, 0);
queue_work(my_wq, &my_work);
```

v6.6.78 内核预定义了 7 个全局 workqueue（`kernel/workqueue.c` L423-435）：

```c
struct workqueue_struct *system_wq __read_mostly;
struct workqueue_struct *system_highpri_wq __read_mostly;
struct workqueue_struct *system_long_wq __read_mostly;
struct workqueue_struct *system_unbound_wq __read_mostly;
struct workqueue_struct *system_freezable_wq __read_mostly;
struct workqueue_struct *system_power_efficient_wq __read_mostly;
struct workqueue_struct *system_freezable_power_efficient_wq __read_mostly;
```

它们的含义和对应的快捷调度函数：

| workqueue | 快捷调度函数 | 场景 |
|-----------|-----------|------|
| `system_wq` | `schedule_work()` | 通用短 work，80% 的情况用这个 |
| `system_highpri_wq` | `schedule_work_highpri()` | 需要低延迟执行（如音频） |
| `system_long_wq` | `schedule_long_work()` | 执行时间可能很长（如固件加载） |
| `system_unbound_wq` | `schedule_unbound_work()` | 不绑定固定 CPU，可跨核心迁移 |
| `system_freezable_wq` | `schedule_freezable_work()` | 系统休眠时自动暂停 |
| `system_power_efficient_wq` | 无 | 省电优先，不紧急 |
| `system_freezable_power_efficient_wq` | 无 | 休眠暂停 + 省电 |

大部分驱动只需要 `schedule_work()`，由 `system_wq` 承接。只有特殊需求才需要选其他队列或自己 `alloc_workqueue()`。

### 3.5 线程化 IRQ — v2.6.30 引入

2008 年，Thomas Gleixner 引入了 `request_threaded_irq()` API（v2.6.30）。它的核心思想是**在 request_irq 时直接提供一个进程上下文的处理函数**：

```c
/* v6.6.78 — kernel/irq/manage.c L2145 */
int request_threaded_irq(unsigned int irq,
                         irq_handler_t handler,        /* 顶半部（可 NULL） */
                         irq_handler_t thread_fn,       /* 线程化处理函数 */
                         unsigned long irqflags,
                         const char *devname, void *dev_id);
```

执行流程：

```c
/* 简化后的执行路径 */
irq_handler_t handler(int irq, void *dev_id)
{
    /* 做少量必要的硬件操作 */
    return IRQ_WAKE_THREAD;    /* ← 返回此值唤醒内核线程 */
}

/* 内核线程在进程上下文执行 thread_fn */
static int irq_thread(void *data)    /* kernel/irq/manage.c L1298 */
{
    struct irqaction *action = data;

    while (!irq_wait_for_interrupt(action)) {
        irq_thread_fn(desc, action);  /* → action->thread_fn(irq, dev_id) */
    }
    return 0;
}
```

**触发时机**：handler 返回 `IRQ_WAKE_THREAD` 后，`__irq_wake_thread()` 将对应线程放入运行队列。在 ARM64 上，这个过程发生在 `handle_irq_event_percpu()` 返回后：

```c
/* kernel/irq/handle.c L61 */
void __irq_wake_thread(struct irq_desc *desc, struct irqaction *action)
{
    if (action->thread->flags & PF_EXITING)
        return;
    if (test_and_set_bit(IRQTF_RUNTHREAD, &action->thread_flags))
        return;
    wake_up_process(action->thread);   /* 唤醒内核线程 */
}
```

### 3.6 下半部选择指南

中断发生时，顶半部执行完后需要决定用哪种下半部机制递送工作。以下是选择依据：

```
顶半部处理完毕
    │
    ├─ 需要睡眠（mutex、I2C 通信、copy_to_user）？
    │    ├─ Threaded IRQ（handler 返回 IRQ_WAKE_THREAD）
    │    │    每个中断有独立的内核线程，优先级高、响应快
    │    │    适合每次中断都要处理固定逻辑（如 GPIO 按键上报）
    │    │
    │    └─ Workqueue（schedule_work）
    │         借用共享线程池（system_wq），和其他驱动共用 worker
    │         适合非紧急、零散的任务
    │
    ├─ 不可睡眠，耗时短？→ Tasklet（tasklet_schedule）
    │    └─ 面向普通驱动，运行时动态注册
    │
    └─ 不可睡眠，性能密集型？→ Softirq（open_softirq + raise_softirq）
         └─ 面向网络、块设备等高性能子系统，编译时静态注册
```

**Threaded IRQ vs Workqueue**：Threaded IRQ 的中断线程是每个中断独立创建的（`request_threaded_irq` 时框架自动创建），命名 `irq/N-name`，优先级高，适合中断处理有确定性延迟要求的场景。Workqueue 的 worker 是共享的，适合不紧急的批处理任务——顶半部先把数据收下来，workqueue 里慢慢处理。**如果拿不准，优先用 Threaded IRQ**——它更简单，`request_threaded_irq(irq, handler, thread_fn, ...)` 一步到位。

---

## 4. irq_domain 的诞生（v2.6.38）

### 4.1 问题：多个中断控制器的命名空间冲突

Generic IRQ Layer 定型后，中断使用方式是这样的：

1. 驱动程序调用 `request_irq(irq_number, handler, ...)`，其中 `irq_number` 是一个**全局分配的 Linux IRQ 号**
2. 内核通过 `irq_desc[irq_number]` 查找对应的 `irq_desc`
3. 硬件中断触发时，架构代码调用 `generic_handle_irq(irq_number)`

在单一中断控制器（如 x86 的 8259A PIC 或 IO-APIC）的系统中，这一套能工作——hwirq 可以直接映射到 Linux IRQ 号，因为所有中断源属于同一个控制器。但 **SoC 内部通常有多个中断控制器**，以 STM32MP257 为例：

- GICv3：SPI 空间 hwirq=32~1019
- EXTI：hwirq=0~79（外部中断线）
- GPIOA~GPIOZ 每个 16 个引脚：各有自己的 hwirq=0~15

**每个控制器都有 hwirq=0**。内核收到中断号 5，它来自 GPIOA 的 PA5 还是 GIC 的 SPI 5？这个问题在单一控制器时代不存在，在多控制器 SoC 上必须解决。

**核心矛盾**：硬件用 `(interrupt_controller, hwirq)` 二元组唯一标识中断源，而内核需要一个**全局整数** `virq`。需要一个映射层来完成这次转换。

内核文档对此有一段经典的描述：

> Here the interrupt number loose all kind of correspondence to hardware interrupt numbers: whereas in the past, IRQ numbers could be chosen so they matched the hardware IRQ line into the root interrupt controller, nowadays this number is just a number.

> "中断号已经失去了与硬件中断号之间的任何对应关系：过去 IRQ 号可以选到与根中断控制器的硬件中断线匹配，如今这个数字仅仅是一个数字而已。"

### 4.2 irq_domain 的设计——`(controller, hwirq)` → `virq`

Grant Likely 在 **v2.6.38**（2011 年）合入了 `irq_domain` 框架。核心思路是新增一层映射，将 `(interrupt_controller, hwirq)` 转为全局 Linux IRQ 号（virq）。

#### 核心结构体

```c
struct irq_domain {
    const struct irq_domain_ops *ops;   /* 操作函数：翻译 hwirq、映射等 */
    struct fwnode_handle    *fwnode;    /* 绑定到哪个设备树节点 */
    void                    *host_data; /* 驱动私有数据 */
    struct irq_domain       *parent;    /* 父 domain（层级域用） */
    irq_hw_number_t         hwirq_max;
    struct radix_tree_root  revmap_tree; /* Tree 模式用 */
};
```

#### 映射过程

以设备树 `interrupts = <&gpio 5 IRQ_TYPE_EDGE_BOTH>` 为例：

```
设备树描述 → irq_domain 查找 → 翻译 hwirq
  → irq_alloc_desc() 从全局 irq_desc 树中分配一个空 virq
  → irq_desc[virq].irq_data.hwirq = hwirq
  → irq_desc[virq].irq_data.domain = domain
  → domain->ops->map() 设置 irq_chip 等
```

`irq_alloc_desc()` 是自动分配编号的，virq 与 hwirq 之间没有固定对应关系。`(gpio_domain, hwirq=5)` → `virq=70` 的映射完全由 irq_domain 维护。

### 4.3 四种反向映射方式

硬件中断触发时，内核需要由 hwirq 快速找到 virq。根据控制器的特点选择不同方式：

| 模式 | 数据结构 | 适用场景 |
|------|---------|---------|
| **Linear** | 固定数组 `revmap[hwirq] = virq` | hwirq 连续且数量不大（如 GPIO 的 16 个引脚） |
| **Tree** | radix tree | hwirq 空间大但使用稀疏（如 GICv3 SPI 可达 1020） |
| **Legacy** | 预分配固定数组 | 旧平台兼容（已废弃） |
| **NoMap** | 无映射（hwirq = virq） | 极少使用 |

Linear 以 hwirq 为下标直接查数组，O(1)，STM32MP257 的 pinctrl 就是用的 Linear。GIC 用 Tree，因为 SPI 空间大（0~1019）但实际只用其中一部分，没必要预分配全部数组。

### 4.4 irq_domain 的历史意义

irq_domain 解决了一个根本问题：**将 Linux IRQ 号与硬件中断号解耦**。在此之前，驱动里硬编码中断号是常态，改用设备树后，中断描述从"写死整型"变成了 `(controller_phandle, hwirq)` 二元组，驱动不再关心自己拿到的是几号 IRQ。

这个解耦带来的三个直接变化：

1. **驱动可移植**：同一份驱动代码在不同 SoC 上编译，不需要改中断号
2. **多控制器共存**：每个控制器有自己的 domain，hwirq=0 在各 domain 内独立解析，不再冲突
3. **为层级域铺路**：irq_domain 的 `parent` 指针在 v3.10 后被复用，实现了 GIC→EXTI→GPIO 这样的多层中断路径——这正是下一章的主题

---

## 5. 中断控制器级联：链式与层级域

irq_domain 框架落地后，开发者很快遇到了一个新问题：**当系统中存在多个中断控制器时，它们之间的级联关系如何用软件表达？** 硬件上有两种连接方式，对应了两种软件方案。链式方案最先出现（解决多对一），层级域后来补充（解决一对一）。

### 5.1 链式（Chained）—— 多对一拓扑

最早的级联方案。硬件特征是**子控制器的多个中断信号共享父控制器的一个中断输入**。以 GPIO 直接接 GIC 为例——GPIO 的 16 个引脚共用一个 GIC SPI：

```
GPIO 引脚 0 ─┐
GPIO 引脚 1 ─┤
GPIO 引脚 2 ─┤──→ GIC SPI 33（共享）
GPIO 引脚 3 ─┘
```

这种连接方式在嵌入式 SoC 中很常见，因为 GIC 的 SPI 是有限资源。处理中断时，GIC SPI 33 的 handler 不是直接调设备 action，而是**手动分发**——读 GPIO 寄存器区分是哪个引脚触发了，再调对应子中断：

```
GIC SPI 33 触发
  → 链式 handler 运行（取代常规的 action handler）
  → 读 GPIO 中断状态寄存器
  → 发现引脚 2 触发了
  → generic_handle_irq(gpio_pin_2_virq)
```

链式方案通过 `irq_set_chained_handler_and_data()` 注册，不创建 action，用自定义 handler 覆盖了父中断的整个处理流程。

### 5.2 层级域（Hierarchy）—— 一对一拓扑

硬件设计的演进催生了另一种级联场景：**子控制器的每个中断源都有独立的父控制器中断线**。

以 STM32MP257 的 EXTI + GIC 为例——EXTI event 0~47 各占一个 GIC SPI：

```
EXTI event 0  →  GIC SPI 268
EXTI event 1  →  GIC SPI 269
...
EXTI event 47 →  GIC SPI 315
```

这里每个 EXTI event 有独立的 SPI，不需要软件分发。但 EXTI 本身是**功能性控制器**——它要做边沿检测、触发类型选择、唤醒管理。所以 mask 一个中断时，EXTI 驱动操作完自己的 IMR 寄存器后，还需要递归调用 GIC 的 mask 函数。层级域将这个"递归操作"抽象为 `irq_domain->parent` 指针和 `irq_chip_*_parent()` 系列函数。

```
virq 的 mask 操作：
  → exti_chip->irq_mask(data)        ← EXTI 写 IMR 寄存器
  → irq_chip_mask_parent(data)       ← 递归到父级
      → gic_chip->irq_mask(data)     ← GIC 写 GICD_ICENABLER
```

层级域不是用来替代链式的——它解决的是**链式无法处理的问题**：当中间层控制器需要参与每一级硬件操作时，链式的"父 handler 手动分发"无法满足递归操作的需求。

### 两种方案对比

| | 链式（Chained） | 层级域（Hierarchy） |
|--|---------------|-----------------|
| 硬件连接 | 多对一（共享一个父中断） | 一对一（各占独立父中断） |
| 软件分发 | 手动读寄存器 | 不需要（硬件已决定） |
| 操作递归 | 不需要 | 需要（mask/eoi 逐层传递） |
| 资源消耗 | 省父中断 | 耗父中断 |
| 典型场景 | GPIO 直连 GIC | EXTI 在 GPIO 和 GIC 之间 |

---

## 结语：中断子系统的设计哲学

回顾 30 年演进史，Linux 中断子系统的每一个核心机制都对应一个明确的历史问题：

| 机制 | 解决的问题 | 引入版本 |
|------|-----------|---------|
| **BH** | 顶半部必须快，耗时工作推迟 | v2.1 |
| **irq_chip + 流控函数** | `__do_IRQ()` 把流控和硬件操作混在一起 | v2.6.18 |
| **Softirq** | BH 的 `global_bh_lock` 导致 SMP 扩展性差 | v2.3 重写 |
| **Tasklet** | Softirq 静态注册，驱动无法动态创建 | v2.4 |
| **Workqueue** | Softirq/Tasklet 不可睡眠 | v2.5 |
| **Threaded IRQ** | 为每个中断创建专用内核线程，简化驱动开发 | v2.6.30 |
| **irq_domain** | 多控制器时代 hwirq 命名空间冲突 | v2.6.38 |
| **层级域** | 多对一/一对一两种级联拓扑需要不同的软件方案 | v3.10+ |

**一条隐线贯穿始终**：每一次演进都是因为"硬件变复杂了"或"并发要求变高了"，迫使内核将抽象的层次再提高一层。从单一的 `__do_IRQ()` 到可替换的流控函数，从全局的 BH 位图到 per-CPU 的 softirq，从固定的 hwirq→virq 映射到 irq_domain——每一层抽象都让驱动开发者离硬件更远一步，但离"写一次到处跑"的目标也更近一步。

