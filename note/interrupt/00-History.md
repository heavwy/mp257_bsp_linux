# Linux 中断子系统 30 年演进史

> 为什么 irq_desc、irq_domain、irq_chip、handle_fasteoi_irq 是今天这个样子？
>
> 每个数据结构背后都对应一个真实的历史问题。本文从 Linux v1.0 到 v6.6，沿着中断子系统的关键节点，追溯**每个机制是在解决什么问题**。
>
> **字数**：约 12,000 字 · **建议阅读时间**：40~60 分钟

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

### 2.4 流控函数的定型

Generic IRQ Layer 定义了五种标准流控函数，每种对应一类中断处理场景。以下是从当前 `kernel/irq/chip.c` 中提取的核心逻辑对比：

**handle_level_irq（电平触发）**：

```c
void handle_level_irq(struct irq_desc *desc)
{
    mask_ack_irq(desc);            /* 屏蔽中断 + 应答 */
    handle_irq_event(desc);        /* 调用驱动 handler */
    unmask_irq(desc);              /* 处理完后解屏蔽 */
}
```

电平信号保持有效，所以处理前必须屏蔽以防止同中断再次触发，处理完后才解屏蔽。

**handle_edge_irq（边沿触发）**：

```c
void handle_edge_irq(struct irq_desc *desc)
{
    if (desc->status & IRQ_INPROGRESS) {
        mask_ack_irq(desc);        /* 正在处理中，屏蔽并标记 pending */
        desc->status |= IRQ_PENDING;
        return;
    }
    desc->irq_data.chip->irq_ack();      /* 仅应答，不屏蔽 */
    desc->status |= IRQ_INPROGRESS;
    do {
        handle_irq_event(desc);
    } while (desc->status & IRQ_PENDING); /* 处理期间又触发了，继续处理 */
    desc->status &= ~IRQ_INPROGRESS;
}
```

边沿触发是"一次性"的，不会保持。如果上一中断还没处理完又来一个，必须记住（pending）。这是边沿中断最复杂的地方。

**handle_fasteoi_irq（Fast EOI）**：

```c
void handle_fasteoi_irq(struct irq_desc *desc)
{
    handle_irq_event(desc);              /* 直接处理 */
    desc->irq_data.chip->irq_eoi(desc);  /* 处理完后发 EOI */
}
```

Fast EOI 模式是最简洁的——不需要 mask/unmask，因为中断控制器（如 GIC）有自己的 EOI 寄存器，写一次 EOI 硬件自动完成状态清除。这是现代高级中断控制器（GIC、MSI）的标准模式。

**v6.6.78 内核中 STM32MP257 的 GPIO 按键中断正是使用 `handle_fasteoi_irq`**，这一点可以通过你的实测数据验证：

```
root@buildroot:~# cat /sys/kernel/debug/irq/irqs/70
handler:  handle_fasteoi_irq
```

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

第 1 节已经介绍了 BH 的基本问题。这里补充其核心数据结构：

```c
/* 全局 BH 状态 — 一次只能由一个 CPU 处理 */
static unsigned long bh_active;          /* 标记哪些 BH 待处理 */
static unsigned long bh_mask;            /* 标记哪些 BH 已注册 */
static void (*bh_base[32])(void);        /* 32 个 BH 处理函数指针 */

/* 唯一入口 */
void do_bottom_half(void);
```

BH 的局限：

| 问题 | 表现 | 后果 |
|------|------|------|
| 固定 32 个 | 头文件里枚举，新增需改内核 | 子系统开发者无法动态注册 |
| 全局互斥 | `global_bh_lock` 串行化所有 CPU | SMP 扩展性极差 |
| 不可嵌套 | 执行中禁止被另一个 BH 抢占 | 响应延迟不可控 |

### 3.2 Softirq — v2.3 重写（1999 年）

1999 年，Alexey Kuznetsov（ANK）几乎重写了整个下半部系统，这是 softirq.c 头部注释的由来：

```c
/* linux/kernel/softirq.c
 * Rewritten. Old one was good in 2.2, but in 2.3 it was immoral. --ANK (990903)
 */
```

**Softirq 的核心设计原则**（来自 `softirq.c` 头部注释）：

> - No shared variables, all the data are CPU local.
> - If a softirq needs serialization, let it serialize itself by its own spinlocks.
> - Even if softirq is serialized, only local cpu is marked for execution.

翻译过来：
- **CPU 本地数据**：不再有全局 `bh_active`，每个 CPU 有独立的 `__softirq_pending` 位图
- **自串行化**：框架不提供互斥，需要串行化的软中断自己加锁
- **每个 CPU 独立触发**：一个 CPU 上的软中断密集不会阻塞另一个 CPU

v6.6.78 内核中 softirq 的 10 个向量（`include/linux/interrupt.h` L550）：

```c
enum {
    HI_SOFTIRQ = 0,        /* 高优先级 tasklet */
    TIMER_SOFTIRQ,          /* 定时器 */
    NET_TX_SOFTIRQ,         /* 网络发送 */
    NET_RX_SOFTIRQ,         /* 网络接收 */
    BLOCK_SOFTIRQ,          /* 块设备 I/O 完成 */
    IRQ_POLL_SOFTIRQ,       /* 中断轮询 */
    TASKLET_SOFTIRQ,        /* tasklet */
    SCHED_SOFTIRQ,          /* 调度器 */
    HRTIMER_SOFTIRQ,        /* 高精度定时器 */
    RCU_SOFTIRQ,            /* RCU 回调 */
    NR_SOFTIRQS             /* = 10 */
};
```

**Softirq 的执行时机**（`kernel/softirq.c` L653-665）：

```c
void irq_exit_rcu(void)
{
    __irq_exit_rcu();      /* ← 中断出口检查并执行软中断 */
    lockdep_hardirq_exit();
}

static void __irq_exit_rcu(void)
{
    if (!in_interrupt() && local_softirq_pending())
        invoke_softirq();  /* 直接执行，或唤醒 ksoftirqd */
}
```

这意味着：**每次硬中断处理完毕后，在 `irq_exit_rcu()` 中检查是否有待处理的 softirq**。如果有，直接在当前上下文执行（`do_softirq()`），或者唤醒内核线程 `ksoftirqd/N` 在进程上下文中执行。

### 3.3 Tasklet — 基于 Softirq 的简化封装

Tasklet 是 softirq 之上最常用的封装，它的全部代码在 `kernel/softirq.c` 中，基于 `TASKLET_SOFTIRQ` 和 `HI_SOFTIRQ` 两个向量实现：

```c
/* v6.6.78 — kernel/softirq.c L717-723 */
struct tasklet_head {
    struct tasklet_struct *head;
    struct tasklet_struct **tail;
};

static DEFINE_PER_CPU(struct tasklet_head, tasklet_vec);    /* 普通优先级 */
static DEFINE_PER_CPU(struct tasklet_head, tasklet_hi_vec); /* 高优先级 */
```

Tasklet 的调度入口 `tasklet_schedule()` 做的事情很简单：
1. 将 tasklet 加入当前 CPU 的 `tasklet_vec` 链表（如果还没加入）
2. 触发 `TASKLET_SOFTIRQ` 软中断

在 `irq_exit_rcu()` → `do_softirq()` → `tasklet_action()` 路径中，`TASKLET_SOFTIRQ` 的处理函数遍历链表执行已调度的 tasklet：

```c
static void tasklet_action_common(struct softirq_action *a,
                                  struct tasklet_head *tl_head, ...)
{
    struct tasklet_struct *list;

    /* 把链表从 per-CPU 摘下来，这样调度进来的新 tasklet 不会立即执行 */
    list = tl_head->head;
    tl_head->head = NULL;
    tl_head->tail = &tl_head->head;

    /* 遍历链表 */
    while (list) {
        struct tasklet_struct *t = list;
        list = list->next;

        if (tasklet_trylock(t)) {          /* 防止同 tasklet 多 CPU 并行 */
            if (!atomic_read(&t->count)) { /* count=0 表示未被禁用 */
                t->func(t->data);          /* ← 执行 tasklet 回调函数 */
                tasklet_unlock(t);
            }
        }
    }
}
```

**Tasklet 与 Softirq 的关系**：

| | Softirq | Tasklet |
|---|---|---|
| 注册方式 | 编译时静态 | 运行时动态 |
| 同一个 handler 可以并行吗 | 可（多 CPU 同时执行） | 不可（同一类型串行） |
| 使用难度 | 高（需自己处理并发） | 低（内核保证串行） |
| 典型用户 | 网络(NET_RX)、块设备(BLOCK) | GPIO 按键、I2C 消息处理 |

### 3.4 Workqueue — v2.5 引入 → v3.2 CMWQ 重写

Workqueue 与 tasklet 的最大区别是：**tasklet 在 softirq 上下文中执行（不可睡眠），workqueue 在进程上下文（可睡眠）**。

**原始 workqueue（v2.5 ~ v3.1）**：

每个 workqueue 对应一个内核线程（`events/N`），线程从队列中取出 work 执行。问题是：所有 work 共享一个线程池，一个 work 阻塞会影响其他 work。

**CMWQ（Concurrency Managed Workqueue）— v3.2 由 Tejun Heo 重写**：

CMWQ 的核心思想是**将"工作队列"和"工作线程"解耦**：

```
workqueue_struct              ← 对外接口（每个子系统一个）
    ↓ queue_work()
pool_workqueue (pwq)          ← 从 workqueue 到线程池的桥接
    ↓
worker_pool                   ← 线程池（per-CPU 或 unbound）
    ├── worker                 ← 实际干活的内核线程
    ├── worker
    └── ...
```

CMWQ 的关键设计：

- **`system_wq`**：标准共享队列，适合大部分 work（`schedule_work()` 默认使用）
- **`system_highpri_wq`**：高优先级队列
- **`system_unbound_wq`**：非绑定队列（work 可在任意 CPU 上执行）
- **`alloc_workqueue()`**：创建专用队列，可设置 WQ_UNBOUND / WQ_HIGHPRI / WQ_FREEZABLE 等标志

v6.6.78 内核中预定义的 7 个全局 workqueue（`kernel/workqueue.c` L423-435）：

```c
struct workqueue_struct *system_wq __read_mostly;
struct workqueue_struct *system_highpri_wq __read_mostly;
struct workqueue_struct *system_long_wq __read_mostly;
struct workqueue_struct *system_unbound_wq __read_mostly;
struct workqueue_struct *system_freezable_wq __read_mostly;
struct workqueue_struct *system_power_efficient_wq __read_mostly;
struct workqueue_struct *system_freezable_power_efficient_wq __read_mostly;
```

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
     ├─ 需要立刻再次触发中断？→ ThREADED IRQ（handler 返回 IRQ_WAKE_THREAD）
     │
     ├─ 需要睡眠（mutex、copy_to_user）？→ Workqueue
     │    ├─ 偶尔执行 → schedule_work()（用 system_wq）
     │    └─ 频繁执行 → alloc_workqueue(name, 0, 0) 创建专用队列
     │
     ├─ 不可睡眠，耗时短？→ Tasklet
     │    ├─ DECLARE_TASKLET(name, fn)
     │    └─ tasklet_schedule(&name)
     │
     └─ 不可睡眠，性能密集型 → Softirq（直接注册）
          └─ open_softirq(NET_RX_SOFTIRQ, handler);
               raise_softirq(NET_RX_SOFTIRQ);
```

在 STM32MP257 的中断路径中，**你实测的 GPIO 按键中断使用的是哪种下半部**？查一下 `gpio-keys` 驱动的实现就会发现：它使用 `input_event()` 上报给 input 子系统，最终在 `TASKLET_SOFTIRQ` 的上下文中完成事件的分发——tasklet 是中断到用户态之间的关键桥梁。

---

## 4. irq_domain 的诞生（v2.6.38）

### 4.1 问题：中断号不够用了

Generic IRQ Layer 定型后，中断使用方式是这样的：

1. 驱动程序调用 `request_irq(irq_number, handler, ...)`，其中 `irq_number` 是一个**全局分配的 Linux IRQ 号**
2. 内核通过 `irq_desc[irq_number]` 数组查找对应的 `irq_desc`
3. 硬件中断触发时，架构代码调用 `generic_handle_irq(irq_number)`

在单一中断控制器（如 x86 的 8259A PIC 或 IO-APIC）的系统中，Linux IRQ 号可以直接对应硬件中断号，或者通过简单的偏移量转换。但**随着设备树（Device Tree）在 PowerPC 上的推广，以及 SoC 内部多重中断控制器的出现，问题暴露了**：

| 场景 | 问题 |
|------|------|
| GPIO 控制器作为中断控制器 | 32 个 GPIO 引脚映射到 30 个 Linux 中断号 |
| SoC 内部多个中断控制器 | 每个控制器的 hwirq 空间从 0 开始，冲突 |
| 设备树中断描述 | 需要一种方式将 `interrupts = <&gpio 5 IRQ_TYPE_EDGE_BOTH>` 翻译为 Linux IRQ 号 |

**核心矛盾**：硬件用 `(interrupt_controller, hwirq)` 二元组标识一个中断源，而内核需要一个**全局整数** `irq_no`。需要一个映射层解决这个转换。

内核文档 `Documentation/core-api/irq/irq-domain.rst` 这样描述：

> Here the interrupt number loose all kind of correspondence to hardware interrupt numbers: whereas in the past, IRQ numbers could be chosen so they matched the hardware IRQ line into the root interrupt controller, nowadays this number is just a number.

### 4.2 irq_domain 的设计

Grant Likely 在 **v2.6.38**（2011 年）合入了 `irq_domain` 框架。它在 `irq_alloc_desc()` 之上增加了一层 `hwirq → Linux IRQ` 的映射：

```
设备树: interrupts = <&gpio 5 IRQ_TYPE_EDGE_BOTH>
                             │
                             ▼
  irq_domain (由 gpio 控制器注册):
      ops.xlate(hwirq=5)         ← 从 fwspec 解析出 hwirq 和 type
      irq_create_mapping(domain, hwirq=5)
          └── irq_alloc_desc()   ← 分配 Linux IRQ 号（假设是 70）
          └── irq_domain_associate(domain, virq=70, hwirq=5)
              └── ops.map(domain, virq, hwirq)  ← 设置 irq_desc
                                                  irq_desc[70].irq_data.hwirq = 5
                                                  irq_desc[70].irq_data.domain = &gpio_domain
```

关键结构体——`struct irq_domain`（`include/linux/irqdomain.h` L150）：

```c
struct irq_domain {
    struct list_head        link;           /* domain 全局链表 */
    const char              *name;          /* domain 名称（debugfs 显示） */
    const struct irq_domain_ops *ops;       /* 操作函数表 */

    /* 映射数据 */
    irq_hw_number_t         hwirq_max;      /* 最大 hwirq 数 */
    unsigned int            revmap_direct_max_irq;
    struct radix_tree_root  revmap_tree;    /* radix tree（tree 模式使用） */
    struct mutex            revmap_mutex;
    struct fwnode_handle    *fwnode;        /* 关联的设备树 fwnode */
    void                    *host_data;     /* 驱动私有数据（用于存放 chip_data 指针） */
    unsigned int            flags;

    /* ← 父 domain（v3.10+ 层级支持，见第 5 节） */
    struct irq_domain       *parent;
};
```

### 4.3 四种映射方式

irq_domain 提供了四种反向映射方式（`hwirq → virq`）：

| 类型 | 分配函数 | 数据结构 | 查询复杂度 | 适用场景 |
|------|---------|---------|-----------|---------|
| **Linear** | `irq_domain_add_linear()` | 固定大小数组 | O(1) | hwirq 范围小且固定的控制器（推荐） |
| **Tree** | `irq_domain_add_tree()` | radix tree | O(log N) | hwirq 范围大或稀疏的控制器 |
| **Legacy** | `irq_domain_add_legacy()` | 预分配数组 | O(1) | 旧平台固定 IRQ 映射（已废弃） |
| **NoMap** | `irq_domain_add_nomap()` | 无映射 | — | 硬件可编程（极少使用） |

**Linear 模式**是绝大多数驱动控制器的选择。它以 hwirq 为索引直接查表，结构简单、延迟确定。v6.6.78 中 STM32MP257 的 pinctrl 驱动正是使用 Linear 模式注册 GPIO 中断。

**Tree 模式**适合 hwirq 空间巨大（如 GICv3 的 SPI 中断可达 1020 个）但实际使用很少的场景，避免为所有可能的 hwirq 预分配条目。

以 **STM32MP257 的 PROCBUS 设备树**（`stm32mp251.dtsi`）为例，中断引用的三种形式都由 irq_domain 解析：

```dts
/* 形式一：直接引用 GIC SPI — virq 由 GIC domain 分配 */
&usart2 {
    interrupts = <GIC_SPI 146 IRQ_TYPE_LEVEL_HIGH>;
};

/* 形式二：引用 GPIO 中断 — virq 由 GPIO domain 分配，parent 是 EXTI，再 parent 是 GIC */
&gpioh {
    interrupts = <5 IRQ_TYPE_EDGE_BOTH>;
};

/* 形式三：引用 EXTI 唤醒中断 — virq 由 EXTI domain 分配 */
interrupts-extended = <&exti1 47 IRQ_TYPE_LEVEL_HIGH>;
```

### 4.4 irq_domain 的初始化顺序

irq_domain 的另一个重要设计是**依赖排序初始化**。中断控制器有层级关系（GIC → EXTI → GPIO），子控制器的 irq_domain 需要父 domain 已存在。

内核的 `drivers/of/irq.c` 中的 `of_irq_init()` 实现了这个排序：

```c
/* v6.6.78 — drivers/of/irq.c */
void __init of_irq_init(const struct of_device_id *matches)
{
    /* 第一阶段：扫描设备树，收集所有 interrupt-controller 节点 */
    for_each_matching_node_and_match(np, matches, &match) {
        desc = kzalloc(sizeof(*desc), GFP_KERNEL);
        desc->irq_init_cb = match->data;         /* 驱动初始化函数 */
        desc->dev = of_node_get(np);
        /* 找到该控制器的父节点 */
        desc->interrupt_parent = of_irq_find_parent(np);
        list_add_tail(&desc->list, &intc_desc_list);
    }

    /* 第二阶段：按 parent 依赖排序初始化 */
    while (!list_empty(&intc_desc_list)) {
        list_for_each_entry_safe(desc, ...) {
            if (desc->interrupt_parent != parent)
                continue;                        /* 爹还没初始化，跳过 */
            /* 初始化当前控制器（parent=null 的是根 GIC，先初始化） */
            desc->irq_init_cb(desc->dev, desc->interrupt_parent);
        }
        /* 取新初始化的控制器作为下一轮的"父节点" */
        parent = next->dev;
    }
}
```

这个"先父亲后儿子"的初始化策略确保了在 STM32MP257 上：
1. 先初始化 **GIC（根 domain）**：`gic_of_init()` → 创建 GIC irq_domain
2. 再初始化 **EXTI（子 domain）**：`stm32mp_exti_probe()` → `irq_domain_add_hierarchy(gic_domain, ...)`
3. 最后初始化 **Pinctrl/GPIO**：注册 Linear domain，parent 指向 EXTI domain

这正是你实测 `/sys/kernel/debug/irq/irqs/70` 看到的三层 domain 拓扑：

```
domain: :soc@0:pinctrl@44240000-0   ← GPIO domain (leaf)
  parent: :soc@0:intc@44220000       ← EXTI domain (middle)
    parent: :intc@4ac00000           ← GIC domain (root)
```

---

## 5. 从链式到层级：中断控制器的两种级联方案（v3.0+）

### 5.1 链式（Chained）—— 老方案

在多级中断控制器出现之前，中断控制器之间最常见的连接方式就是"链式"。它的硬件特征是**多对一**：子控制器的多个中断输出，汇总到父控制器的一个中断输入。

以 GPIO 中断为例：

```
GPIO 引脚 0 ─┐
GPIO 引脚 1 ─┤
GPIO 引脚 2 ─┤──→ GIC SPI 33 号
GPIO 引脚 3 ─┘
```

在链式方案中，需要两个 irq_desc：
- `irq_desc[17]` 对应 GIC SPI 33 号（父中断）
- `irq_desc[100~103]` 对应 GPIO 的 4 个引脚（子中断）

处理流程：

```c
/* GIC 检测到 SPI 33 → virq=17 */
generic_handle_domain_irq(gic_domain, 33);    /* irq_find_mapping → 17 */
    → irq_desc[17].handle_irq                 /* → gic_handle_cascade_irq() */

        /* GIC 驱动内的级联处理函数 */
        gic_handle_cascade_irq(desc):
            mask_ack_irq(desc)                  /* 屏蔽 SPI 33 */
            chip = irq_data_get_irq_chip(desc);
            /* 调用 GPIO 驱动注册的 handler → 读 GPIO 寄存器 */
            generic_handle_irq(virq=102);       /* 细分中断源 */
            chip->irq_unmask(desc);              /* 解屏蔽 SPI 33 */
```

链式方案的关键特征：
- **2 个 irq_desc**：父级一个、子级一群
- **父级 handler 做分发**：GIC SPI 33 的 handler 不是去调用设备的 action，而是去读 GPIO 寄存器，再调用子中断的 handler
- **`irq_set_chained_handler_and_data()`** 注册：专门用于链式场景，不创建 action，直接替换 handler

GIC 驱动中的 `gic_cascade_irq()` 就是典型实现（`irq-gic.c` L419）：

```c
void __init gic_cascade_irq(unsigned int gic_nr, unsigned int irq)
{
    irq_set_chained_handler_and_data(irq, gic_handle_cascade_irq,
                                     &gic_data[gic_nr]);
}
```

链式的局限：
- **父中断不可被其他设备共享**（已经被级联 handler 独占）
- **需要额外软件分发**（读 GPIO 寄存器区分来源），增加延迟
- **嵌套层级越多，软件开销越大**（每层都要手动分发）
- **扩展性差**（GPIO 每增加一个中断引脚，不需要改 GIC 配置，但分发逻辑变复杂）

### 5.2 层级（Hierarchy）—— 新方案

层级方案的前提是：**子控制器的每个中断在父控制器中有独立的中断线**（一对一映射）。

对于 STM32MP257 的 EXTI+GIC 组合，硬件连接是：

```
EXTI event 0  →  GIC SPI 268
EXTI event 1  →  GIC SPI 269
...
EXTI event 47 →  GIC SPI 315
```

每个 EXTI event 有**独立的 GIC SPI**，不需要软件分发。EXTI 控制器的作用是边沿检测、触发类型选择、唤醒管理——而不是"分辨谁触发了中断"。

流程完全不同：

```
request_irq(virq=70, handler)
    → irq_domain_alloc_irqs(gpio_domain, hwirq=5)
        → gpio_domain.alloc(virq=70, hwirq=5)    ← 设置 chip_data
        → irq_domain_alloc_irqs_parent()          ← 递归
            → exti_domain.alloc(virq=70, hwirq=5) ← 设置 chip_data
            → irq_domain_alloc_irqs_parent()       ← 递归
                → gic_domain.alloc(virq=70, hwirq=305)
                                                    ← 写入 GICD_ISENABLER
```

处理中断时：

```c
GIC 检测到 SPI 305 → virq=70
generic_handle_domain_irq(gic_domain, 305);    /* 直接找到 virq=70 */
    → irq_desc[70].handle_irq                   /* → handle_fasteoi_irq */
        → irq_desc[70].chip->irq_mask           /* → stm32mp_exti_mask → 
                                                    写 IMR 寄存器 + 
                                                    irq_chip_mask_parent()
                                                    → gic_mask_irq */
        → handle_irq_event(desc)                /* → action->handler */
        → irq_desc[70].chip->irq_unmask         /* → stm32mp_exti_unmask */
        → irq_desc[70].chip->irq_eoi            /* → stm32mp_exti_eoi →
                                                    写 RPR/FPR +
                                                    irq_chip_eoi_parent()
                                                    → gic_eoi_irq */
```

层级方案的关键特征：

| 特性 | 链式 (Chained) | 层级 (Hierarchy) |
|------|---------------|-----------------|
| irq_desc 数量 | 父 1 + 子 N = N+1 | 子 N（共享父的 desc） |
| 分发方式 | handler 手动读寄存器分发 | irq_chip 递归调用父级 |
| 父中断是否可共享 | 否 | 是（每个子中断有独立 SPI） |
| 硬件要求 | 多对一 | 一对一 |
| 软件开销 | 每层 handler 调用 | 函数指针递归 |
| 典型实现 | `gic_cascade_irq()` | EXTI `irq_domain_add_hierarchy()` |

### 5.3 层级域的内核实现

层级域的核心是 `irq_domain_create_hierarchy()`（`kernel/irq/irqdomain.c` L1130）：

```c
struct irq_domain *irq_domain_create_hierarchy(struct irq_domain *parent,
                                               unsigned int flags,
                                               unsigned int size,
                                               struct fwnode_handle *fwnode,
                                               const struct irq_domain_ops *ops,
                                               void *host_data)
{
    struct irq_domain *domain;

    domain = __irq_domain_create(fwnode, size, size, 0, ops, host_data);
    if (domain) {
        domain->parent = parent;        /* ← 关键：记录父 domain */
        ...
        __irq_domain_publish(domain);
    }
    return domain;
}
```

当子 domain 的 `alloc()` 中调用 `irq_domain_alloc_irqs_parent()` 时，它递归进入父 domain：

```c
int irq_domain_alloc_irqs_parent(struct irq_domain *domain,
                                 unsigned int irq_base, unsigned int nr_irqs,
                                 void *arg)
{
    if (!domain->parent)
        return -ENOSYS;
    /* 递归到父 domain 的 alloc */
    return irq_domain_alloc_irqs_hierarchy(domain->parent, irq_base,
                                           nr_irqs, arg);
}
```

**STM32MP257 的 EXTI 正是使用层级域**（`irq-stm32mp-exti.c` L836 probe 中）：

```c
/* 创建层级 domain，parent 指向 GIC domain */
domain = irq_domain_add_hierarchy(parent_domain, 0,
                                  bank_nr * 32,
                                  np, &stm32mp_exti_domain_ops,
                                  host_data);
```

而 EXTI domain ops 的 `alloc()` 在设置好自身 chip_data 后，调用 `irq_domain_alloc_irqs_parent()` 让 GIC 完成剩余工作（`irq-stm32mp-exti.c` L677）。

**irq_chip 的递归调用**：层级域下，mask/unmask/eoi 等操作也需要递归到父级。EXTI 驱动的 irq_chip 中使用了 `irq_chip_*_parent()` 系列函数：

```c
static struct irq_chip stm32mp_exti_chip = {
    .name          = "stm32mp-exti",
    .irq_eoi       = stm32mp_exti_eoi,         /* 写 RPR+FPR 清挂起 + 
                                                   irq_chip_eoi_parent(d) */
    .irq_mask      = stm32mp_exti_mask,         /* 写 IMR + 
                                                   irq_chip_mask_parent(d) */
    .irq_unmask    = stm32mp_exti_unmask,       /* 写 IMR + 
                                                   irq_chip_unmask_parent(d) */
    .irq_set_type  = stm32mp_exti_set_type,     /* 写 RTSR/FTSR */
    .irq_set_wake  = stm32mp_exti_set_wake,     /* 标记唤醒源 +
                                                   irq_chip_set_wake_parent(d) */
    .irq_ack       = irq_chip_ack_parent,       /* 直通 GIC */
};
```

其中 `irq_chip_eoi_parent()` 等函数的实现就是通过 `irq_data->parent_data` 找到父级的 irq_chip 并调用对应函数——这就是"层级"的本质：**每个操作可以穿透到父级，形成一条完整的硬件操作链**。

---

## 6. ARM64 中断入口（v3.11+）

### 6.1 ARM64 在 Linux 中的支持

ARM64（AArch64）架构从 Linux **v3.7**（2012 年 10 月）开始获得官方支持。与 ARM32 不同，ARM64 的中断入口完全重写——使用 `VBAR_EL1` 寄存器定位异常向量表，由 `entry.S` 中的 `vectors` 符号定义。

### 6.2 异常向量表结构

ARM64 的异常向量表由 16 个条目组成，分为 4 组（每组 4 个异常类型），每组的基地址由当前的异常级别（EL）和使用的堆栈指针（SP）决定：

```
向量表基址 (VBAR_EL1)
    ├── 第 1 组：EL1, 使用 SP_EL0 (同步/IRQ/FIQ/Error)
    ├── 第 2 组：EL1, 使用 SP_EL1 ← 内核态中断走这里
    ├── 第 3 组：EL0, 64 位程序  ← 用户态中断走这里
    └── 第 4 组：EL0, 32 位程序
```

每组 4 种异常类型：同步（sync）、普通中断（irq）、快速中断（fiq）、系统错误（error）。

`arch/arm64/kernel/entry.S` 中使用 `kernel_ventry` 宏定义所有 16 个向量：

```asm
.align 11                     /* 2KB 对齐 — ARM64 硬性要求 */
SYM_CODE_START(vectors)
    kernel_ventry  1, t, 64, sync         /* EL1t Synchronous */
    kernel_ventry  1, t, 64, irq          /* EL1t IRQ */
    kernel_ventry  1, t, 64, fiq          /* EL1t FIQ */
    kernel_ventry  1, t, 64, error        /* EL1t Error */

    kernel_ventry  1, h, 64, sync         /* EL1h Synchronous */
    kernel_ventry  1, h, 64, irq          /* EL1h IRQ  ← 内核态中断入口 */
    kernel_ventry  1, h, 64, fiq          /* EL1h FIQ */
    kernel_ventry  1, h, 64, error        /* EL1h Error */

    kernel_ventry  0, t, 64, sync         /* EL0 64-bit Synchronous */
    kernel_ventry  0, t, 64, irq          /* EL0 64-bit IRQ  ← 用户态中断入口 */
    kernel_ventry  0, t, 64, fiq          /* EL0 64-bit FIQ */
    kernel_ventry  0, t, 64, error        /* EL0 64-bit Error */

    kernel_ventry  0, t, 32, sync         /* EL0 32-bit Synchronous */
    kernel_ventry  0, t, 32, irq          /* EL0 32-bit IRQ */
    kernel_ventry  0, t, 32, fiq          /* EL0 32-bit FIQ */
    kernel_ventry  0, t, 32, error        /* EL0 32-bit Error */
SYM_CODE_END(vectors)
```

异常向量表的安装时机在 `head.S` 的 `__primary_switched` 函数中：

```asm
/* arch/arm64/kernel/head.S */
SYM_FUNC_START_LOCAL(__primary_switched)
    ...
    adr_l   x8, vectors            /* 加载向量表地址 */
    msr vbar_el1, x8               /* 写入 VBAR_EL1 寄存器 */
    isb                            /* 指令同步屏障，保证立即生效 */
    ...
    bl      start_kernel           /* 跳入 C 语言初始化 */
```

### 6.3 kernel_ventry 宏的工作流程

`kernel_ventry` 宏（`entry.S`）为每个向量条目生成 128 字节的入口代码，该代码：

1. **在栈上分配 `pt_regs` 空间**（约 288 字节，保存 31 个通用寄存器 + 其他上下文）
2. **检查栈溢出**（通过 `CONFIG_VMAP_STACK` 检测 SP 是否越界）
3. **跳转到对应的处理函数**（如 `el1h_64_irq`）

以 `kernel_ventry 1, h, 64, irq` 为例，最终跳转到 `el1h_64_irq`。这个函数通过 `entry_handler` 宏定义：

```asm
/* entry.S: entry_handler 宏扩展为完整函数 */
SYM_CODE_START_LOCAL(el1h_64_irq)
    kernel_entry 1, 64             /* 保存所有寄存器到栈上 pt_regs */
    mov     x0, sp                 /* x0 = regs 指针 */
    bl      el1h_64_irq_handler    /* 调用 C 函数 */
    b       ret_to_kernel          /* 恢复寄存器、返回 */
SYM_CODE_END(el1h_64_irq)
```

### 6.4 C 层面的处理入口

`el1h_64_irq_handler()` 定义在 `arch/arm64/kernel/entry-common.c` L520：

```c
asmlinkage void noinstr el1h_64_irq_handler(struct pt_regs *regs)
{
    el1_interrupt(regs, handle_arch_irq);   /* ← 传入函数指针 */
}

static void noinstr el1_interrupt(struct pt_regs *regs,
                                  void (*handler)(struct pt_regs *))
{
    write_sysreg(DAIF_PROCCTX_NOIRQ, daif);  /* 关闭 IRQ（防止嵌套） */

    if (IS_ENABLED(CONFIG_ARM64_PSEUDO_NMI) && !interrupts_enabled(regs))
        __el1_pnmi(regs, handler);           /* Pseudo-NMI 路径 */
    else
        __el1_irq(regs, handler);            /* 标准路径 */
}

static __always_inline void __el1_irq(struct pt_regs *regs,
                                      void (*handler)(struct pt_regs *))
{
    enter_from_kernel_mode(regs);            /* 记录上下文信息（lockdep/trace） */

    irq_enter_rcu();                         /* 进入中断上下文 */
    do_interrupt_handler(regs, handler);     /* ← 最终调用 gic_handle_irq */
    irq_exit_rcu();                          /* ← 退出中断，触发软中断 */

    arm64_preempt_schedule_irq();            /* 检查是否可抢占 */
    exit_to_kernel_mode(regs);
}
```

用户态中断路径类似：`el0t_64_irq` → `el0t_64_irq_handler()` → `__el0_irq_handler_common()` → `el0_interrupt(regs, handle_arch_irq)`。

### 6.5 handle_arch_irq：将架构代码与 GIC 驱动解耦

`handle_arch_irq` 是一个**函数指针**，定义在 `arch/arm64/kernel/irq.c` L100：

```c
static void default_handle_irq(struct pt_regs *regs)
{
    panic("IRQ taken without a root IRQ handler\n");
}

void (*handle_arch_irq)(struct pt_regs *) __ro_after_init = default_handle_irq;

int __init set_handle_irq(void (*handle_irq)(struct pt_regs *))
{
    if (handle_arch_irq != default_handle_irq)
        return -EBUSY;
    handle_arch_irq = handle_irq;             /* GIC 驱动调用此函数注册自己 */
    pr_info("Root IRQ handler: %ps\n", handle_irq);
    return 0;
}
```

GIC 驱动在初始化时（`__gic_init_bases()`）调用 `set_handle_irq(gic_handle_irq)` 将自己注册为根中断处理函数。如果系统中更换了中断控制器（如从 GICv2 换成 GICv3），只需在 GICv3 驱动中调用同样的接口——**架构代码不需要任何修改**。

### 6.6 gic_handle_irq：中断处理主循环

`gic_handle_irq()`（`irq-gic.c` L345）是中断处理的"主循环"：

```c
static void __exception_irq_entry gic_handle_irq(struct pt_regs *regs)
{
    u32 irqstat, irqnr;
    struct gic_chip_data *gic = &gic_data[0];
    void __iomem *cpu_base = gic_data_cpu_base(gic);

    do {
        irqstat = readl_relaxed(cpu_base + GIC_CPU_INTACK);  /* 读 GICC_IAR */
        irqnr = irqstat & GICC_IAR_INT_ID_MASK;              /* 获取中断 ID */

        if (unlikely(irqnr >= 1020))
            break;              /* ID 1020-1023 是伪中断(spurious)，结束循环 */

        /* 如果硬件支持 split EOI/Deactivate，立即写 EOI */
        if (static_branch_likely(&supports_deactivate_key))
            writel_relaxed(irqstat, cpu_base + GIC_CPU_EOI);

        /* SGI (IPI) 需要处理源 CPU 信息 */
        if (irqnr <= 15) {
            /* ... 保存 SGI 源 CPU 信息 ... */
        }

        generic_handle_domain_irq(gic->domain, irqnr);  /* hwirq → virq → handler */
    } while (1);                    /* 循环直到所有待处理中断完成 */
}
```

这个 `do {} while(1)` 循环意味着：**GIC 可以在一次异常入口中处理完所有待处理的中断**，避免多次异常进出。当所有中断处理完毕后，`GICC_IAR` 返回 1023（spurious ID），循环退出。

### 6.7 ARM64 中断入口的完整路径

总结从硬件触发到驱动 handler 的完整路径：

```
GPIO 引脚电平变化
  → EXTI 检测边沿，向 GIC 发送 SPI 信号
  → GIC Distributor 路由到 CPU Interface
  → CPU Interface 向核断言 nIRQ 信号
  → CPU 查 VBAR_EL1 → vectors 表
  → kernel_ventry 1, h, 64, irq            ← entry.S 汇编入口
  → kernel_entry: 保存 pt_regs 到栈上
  → el1h_64_irq_handler(regs)              ← entry-common.c C 入口
  → el1_interrupt(regs, handle_arch_irq)    ← 关闭本地 IRQ
  → __el1_irq(regs, handler)
  → irq_enter_rcu()                         ← 记录中断上下文
  → do_interrupt_handler(regs, gic_handle_irq) ← 调用 GIC 主循环
  → gic_handle_irq:
      do {
          readl(GICC_IAR) → hwirq
          generic_handle_domain_irq(gic->domain, hwirq)
              → irq_find_mapping → virq
              → generic_handle_irq_desc(virq)
                  → desc->handle_irq()  /* handle_fasteoi_irq */
      } while(1)
  → irq_exit_rcu()                          ← 中断退出，触发软中断
  → do_softirq() / wakeup_ksoftirqd()       ← 下半部执行
  → exit_to_kernel_mode(regs)               ← 恢复寄存器，返回
```

**这就是 ARM64 上从硬件中断到驱动 handler 的完整路径**。STM32MP257（Cortex-A35）正是采用这条路径处理所有的 GPIO 按键中断。

---

## 7. 总结

30 年来，Linux 中断子系统经历了 4 次关键重构：

| 时间 | 版本 | 变革 | 核心人物 | 解决什么问题 |
|------|------|------|---------|------------|
| 2003~2006 | v2.6.8~v2.6.24 | Generic IRQ Layer：流控与芯片操作分离 | Thomas Gleixner, Ingo Molnar, Russell King | `__do_IRQ()` 混合流控和硬件操作，无法扩展 |
| 1999~2012 | v2.3~v3.2 | 下半部：BH→Softirq→Tasklet→Workqueue→Threaded IRQ | Alexey Kuznetsov, Tejun Heo, Thomas Gleixner | BH 全局锁瓶颈、SMP 扩展性差 |
| 2011 | v2.6.38 | irq_domain：hwirq→virq 映射 | Grant Likely | 设备树多重中断控制器的编号冲突 |
| 2013+ | v3.7+ | 层级 irq_domain + ARM64 支持 | ARM 社区, Thomas Gleixner | 链式分发延迟、ARM64 新架构 |

每个数据结构和 API 都是**对特定历史问题的回应**，不是凭空设计出来的。

对于 STM32MP257 开发来说，理解这些历史能帮你快速定位：
- 为什么需要理解和区分链式 vs 层级？→ EXTI 是层级域，pinctrl 也是层级域
- 为什么需要 irq_domain？→ 否则 `/proc/interrupts` 中的中断号没有任何结构
- 为什么需要 `irq_chip_*_parent()` 系列函数？→ 层级域下 irq_chip 操作需要递归到父级
- 为什么下半部有 4 种选择？→ 不同的性能/延迟/睡眠需求

回到你实测的按键中断数据：

```
/sys/kernel/debug/irq/irqs/70:
  handler:  handle_fasteoi_irq         ← Generic IRQ Layer 的流控函数
  chip:     stm32gpio → stm32mp-exti → GICv2   ← 层级 irq_chip 递归
  domain:   pinctrl → exti → gic                 ← 层级 irq_domain 三级
```

这三行数据浓缩了中断子系统 30 年的演进结果。
