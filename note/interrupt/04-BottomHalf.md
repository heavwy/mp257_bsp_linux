# 04. 下半部机制分析

> 中断下半部的四种机制——softirq、tasklet、workqueue、threaded irq——的设计原理、核心数据结构、运行时调度路径与 STM32MP257 上的实际选择。
>
> **前置:** [03-SourceAnalysis.md](03-SourceAnalysis.md) — 中断子系统初始化流程（GIC → EXTI → GPIO domain → request_irq）
> **下一篇:** [05-Scenario.md](05-Scenario.md) — 消费者使用中断的情景分析
>
> **字数：约 24,000 词（含代码段）**
> **建议阅读时间：60~90 分钟**

---

## 1. 概述

### 1.1 从一个问题开始

03 篇追踪了中断子系统的初始化——GIC、EXTI、irq_domain、irq_desc 全部就绪。现在系统可以接收中断了。gpio-keys 驱动注册了一个 top half handler，当中断触发时它被执行：

```c
static irqreturn_t gpio_keys_irq_handler(int irq, void *dev_id)
{
    mod_delayed_work(system_wq, &bdata->work, 0);
    return IRQ_HANDLED;
}
```

handler 只做了一件事：调度一个 work。然后返回。**为什么顶半部不能直接处理按键上报？为什么需要推迟到下半部？**

00-History §1.1 交代了根本原因——IRQ context 中不可 sleep、IRQ off 时间过长会丢中断。但"下半部"这个笼统的概念，在 Linux 内核中实际上演进了四种不同的机制，各有各的数据结构、调度策略和适用场景。

**问题是：** 这四种机制分别是什么？它们怎么工作？什么时候该用哪一个？

### 1.2 四种机制总览

| 机制 | 上下文 | 能否 sleep | 并行度 | 引入版本 | 适用场景 |
|------|--------|-----------|--------|---------|---------|
| Softirq | softirq context | ❌ 否 | 多 CPU 同类型并行 | v2.3 | 高性能（网络 RX/TX、块设备）|
| Tasklet | softirq context | ❌ 否 | 同类型串行 | v2.5.3 | 驱动通用推迟（简单、不 sleep）|
| Workqueue | 进程 context | ✅ 能 | 多 worker 并行 | v2.5→v3.2 CMWQ | 可 sleep 的异步任务 |
| 线程化 IRQ | 进程 context | ✅ 能 | 每中断独享线程 | v2.6.30 | 中断处理本身需要 sleep |

注意其中的依赖关系：**tasklet 基于 softirq 实现**，workqueue 和线程化 IRQ 基于内核线程。softirq 是整个下半部体系的基石。

### 1.3 两条路径的分水岭：irq_exit_rcu

四种机制在调度时机上分为两条路径：

```
中断顶半部返回
  └─ __irq_exit_rcu()                    ← kernel/softirq.c:633
       ├─ 有 softirq pending
       │    └─ invoke_softirq()           ← 在中断返回路径上直接执行
       │         └─ __do_softirq()        ← softirq/tasklet 路径
       │              └─ softirq_vec[n].action()
       │                   ├─ tasklet_action()    ← TASKLET_SOFTIRQ
       │                   └─ tasklet_hi_action() ← HI_SOFTIRQ
       └─ 无 softirq pending
            └─ eret 返回被中断的上下文
                 └─ worker_thread() / irq_thread() ← workqueue/threaded irq 路径
                                                      （独立调度，与中断返回无关）
```

- softirq、tasklet：通过 `raise_softirq` 置位 pending，在 `irq_exit_rcu` 中直接执行
- workqueue：通过 `schedule_work` 唤醒 worker 线程，由调度器独立调度
- 线程化 IRQ：通过 `return IRQ_WAKE_THREAD` 唤醒 irq_thread，由调度器独立调度

两条路径的执行上下文完全不同——前者在 softirq context（仍然是原子的），后者在进程 context（可以 sleep）。这个区别决定了四种机制各自的约束和适用场景。

有了这个全景认识，下面从最基础的 softirq 开始，逐层深入。

---

## 2. Softirq

### 2.1 完整路径

softirq 的完整执行路径是这样的：

```
驱动/子系统调 raise_softirq(N)
  → or_softirq_pending(1UL << N)          ← 置位 pending bit
  → if (!in_interrupt())
      wakeup_softirqd()                   ← 不在中断中 → 唤醒 ksoftirqd

中断顶半部返回时：
  __irq_exit_rcu()
    → if (!in_interrupt() && local_softirq_pending())
        → invoke_softirq()                ← 在中断返回路径上直接执行
            → __do_softirq()
              → handle_softirqs(false)    ← 遍历 pending 并调 action

或 ksoftirqd 线程：
  run_ksoftirqd()
    → if (local_softirq_pending())
        → handle_softirqs(true)           ← 同一個函数，参数不同
```

所以 `handle_softirqs` 是真正的调度循环，它有**两个调用者**：
1. `__do_softirq`（通过 `irq_exit_rcu` 在中断返回路径上直接执行）
2. `run_ksoftirqd`（per-CPU 内核线程，在进程上下文中执行）

两者的区别：`handle_softirqs(false)` 直接在当前上下文执行（中断返回时），`handle_softirqs(true)` 在 ksoftirqd 线程中执行（可被抢占）。

理解了整体路径后，下面看各环节的数据结构和代码实现。

### 2.2 核心数据结构

softirq 的核心是一个全局函数指针数组和一个 per-CPU pending 位图。

**softirq_vec[] — 全局 action 数组**

```c
// kernel/softirq.c:59
static struct softirq_action softirq_vec[NR_SOFTIRQS] __cacheline_aligned_in_smp;

// include/linux/interrupt.h:588
struct softirq_action
{
    void (*action)(struct softirq_action *);
};
```

`NR_SOFTIRQS = 10`（`include/linux/interrupt.h:561`）。10 个槽位，每个槽位一个函数指针。这是一个**全局稀疏数组**——不是所有槽位都被注册，只有内核子系统需要的才填。

**10 个 softirq 向量：**

```c
// include/linux/interrupt.h:550
enum
{
    HI_SOFTIRQ = 0,      // 高优先级 tasklet
    TIMER_SOFTIRQ,       // 定时器
    NET_TX_SOFTIRQ,      // 网络 TX
    NET_RX_SOFTIRQ,      // 网络 RX
    BLOCK_SOFTIRQ,       // 块设备完成
    IRQ_POLL_SOFTIRQ,    // 中断轮询
    TASKLET_SOFTIRQ,     // 普通 tasklet
    SCHED_SOFTIRQ,       // 调度负载均衡
    HRTIMER_SOFTIRQ,     // 高精度定时器
    RCU_SOFTIRQ,         // RCU 回调
    NR_SOFTIRQS          // = 10
};
```

| nr | 名称 | 注册者 | action 函数 | 用途 |
|----|------|--------|------------|------|
| 0 | HI_SOFTIRQ | softirq_init | tasklet_hi_action | 高优先级 tasklet |
| 1 | TIMER_SOFTIRQ | timer_init | run_timer_softirq | 定时器 |
| 2 | NET_TX_SOFTIRQ | 网络子系统 | net_tx_action | 网络 TX |
| 3 | NET_RX_SOFTIRQ | 网络子系统 | net_rx_action | 网络 RX（NAPI）|
| 4 | BLOCK_SOFTIRQ | 块子系统 | blk_done_softirq | 块设备完成 |
| 5 | IRQ_POLL_SOFTIRQ | 块子系统 | irq_poll_softirq | 中断轮询 |
| 6 | TASKLET_SOFTIRQ | softirq_init | tasklet_action | 普通 tasklet |
| 7 | SCHED_SOFTIRQ | 调度器 | run_rebalance_domains | 负载均衡 |
| 8 | HRTIMER_SOFTIRQ | hrtimer_init | hrtimer_run_softirq | 高精度定时器 |
| 9 | RCU_SOFTIRQ | RCU | rcu_core_si | RCU 回调 |

**__softirq_pending — per-CPU pending 位图**

```c
// include/linux/interrupt.h
DECLARE_PER_CPU(struct irq_cpustat_t, irq_stat);
// include/asm-generic/softirq_stack.h
typedef struct {
    unsigned int __softirq_pending;
} ____cacheline_aligned irq_cpustat_t;
```

每个 CPU 一个 32bit 整数。bit N = 1 表示 softirq 向量 N 待处理。读写通过 `local_softirq_pending()` / `or_softirq_pending()` 访问。

### 2.3 注册：open_softirq

```c
// kernel/softirq.c:709
void open_softirq(int nr, void (*action)(struct softirq_action *))
{
    softirq_vec[nr].action = action;
}
```

简单到极致。`NR_SOFTIRQS` 个槽位的全局数组，注册就是填函数指针。

**但 `softirq_init` 只注册了 TASKLET 和 HI 两个**（§2.7），其余 8 个由各子系统自行注册：

```c
// kernel/softirq.c:915 — softirq_init
open_softirq(TASKLET_SOFTIRQ, tasklet_action);
open_softirq(HI_SOFTIRQ, tasklet_hi_action);

// kernel/time/timer.c:2310 — 时钟子系统 init
open_softirq(TIMER_SOFTIRQ, run_timer_softirq);

// kernel/time/hrtimer.c:2278 — 高精度定时器 init
open_softirq(HRTIMER_SOFTIRQ, hrtimer_run_softirq);

// kernel/sched/fair.c:13236 — 调度器 init
open_softirq(SCHED_SOFTIRQ, run_rebalance_domains);

// kernel/rcu/tree.c:5078 — RCU 子系统 init
open_softirq(RCU_SOFTIRQ, rcu_core_si);

// net/core/dev.c:11661 — 网络子系统 initcall
open_softirq(NET_TX_SOFTIRQ, net_tx_action);
open_softirq(NET_RX_SOFTIRQ, net_rx_action);

// block/blk-mq.c:5020 — 块子系统 initcall
open_softirq(BLOCK_SOFTIRQ, blk_done_softirq);
```

| 向量 | 注册代码位置 | 初始化阶段 |
|------|------------|-----------|
| TASKLET_SOFTIRQ | `kernel/softirq.c:915`（softirq_init） | start_kernel |
| HI_SOFTIRQ | `kernel/softirq.c:916`（softirq_init） | start_kernel |
| TIMER_SOFTIRQ | `kernel/time/timer.c:2310`（timer_init） | start_kernel |
| HRTIMER_SOFTIRQ | `kernel/time/hrtimer.c:2278`（hrtimer_init） | start_kernel |
| SCHED_SOFTIRQ | `kernel/sched/fair.c:13236`（sched_init） | start_kernel |
| RCU_SOFTIRQ | `kernel/rcu/tree.c:5078`（rcu_init） | start_kernel |
| NET_TX_SOFTIRQ | `net/core/dev.c:11661`（net_dev_init） | do_initcalls → subsys_initcall |
| NET_RX_SOFTIRQ | `net/core/dev.c:11662`（net_dev_init） | do_initcalls → subsys_initcall |
| BLOCK_SOFTIRQ | `block/blk-mq.c:5020`（blk_mq_init） | do_initcalls → subsys_initcall |

前 6 个在 `start_kernel` 中注册，此时内存管理、调度器、中断系统正在逐步就绪。后 3 个在 `do_initcalls` 阶段注册——此时所有核心子系统已完全就绪，网络和块设备驱动可以正常加载。

ksoftirqd 线程也不是 `softirq_init` 创建的，而是通过 `smpboot_register_percpu_thread` 在 `early_initcall(spawn_ksoftirqd)` 中创建（`kernel/softirq.c:970`），每个 CPU 一个。这意味着在 `early_initcall` 之前，系统中不存在 ksoftirqd 线程——此时 softirq 只能在 `irq_exit_rcu` 路径中直接执行。

### 2.4 触发：raise_softirq

当一个子系统需要触发 softirq 时，调用 `raise_softirq`：

```c
// kernel/softirq.c:693
void raise_softirq(unsigned int nr)
{
    unsigned long flags;

    local_irq_save(flags);
    raise_softirq_irqoff(nr);
    local_irq_restore(flags);
}

// kernel/softirq.c:676
inline void raise_softirq_irqoff(unsigned int nr)
{
    __raise_softirq_irqoff(nr);

    if (!in_interrupt() && should_wake_ksoftirqd())
        wakeup_softirqd();
}

// kernel/softirq.c:702
void __raise_softirq_irqoff(unsigned int nr)
{
    lockdep_assert_irqs_disabled();
    trace_softirq_raise(nr);
    or_softirq_pending(1UL << nr);
}
```

最底层就是 `or_softirq_pending(1UL << nr)` — 在当前 CPU 的 `__softirq_pending` 中置位对应 bit。

**关键决策：`in_interrupt()` 分支**

- 当前在中断上下文中（`in_interrupt() = true`）：只置位 pending。不唤醒 ksoftirqd。因为 `irq_exit_rcu` 会在中断返回时检查并处理 pending。
- 当前不在中断上下文中：置位 pending 并唤醒 ksoftirqd。因为没有 irq_exit_rcu 路径来消费 pending，必须靠 ksoftirqd 线程。

这个分支保证：**从中断中 raise 的 softirq 在中断返回时立即执行，从进程上下文 raise 的 softirq 由 ksoftirqd 在适当时机处理。**

### 2.5 调度循环：handle_softirqs

`__do_softirq()` 是软中断的入口（`kernel/softirq.c:592`），它直接调用 `handle_softirqs(false)`。ksoftirqd 线程则调用 `handle_softirqs(true)`。两者调用的是同一个函数，参数 `ksirqd` 只在 CONFIG_PREEMPT_RT 下有区别，非 RT 内核中未使用：

```c
// kernel/softirq.c:517
static void handle_softirqs(bool ksirqd)
{
    unsigned long end = jiffies + MAX_SOFTIRQ_TIME;   // 软中断最长允许执行 2ms
    int max_restart = MAX_SOFTIRQ_RESTART;             // 最多重入 10 次
    __u32 pending;
    int softirq_bit;

    /* ── ① 取当前 CPU 的 pending 位图 ── */
    pending = local_softirq_pending();
    // pending 是一个 32bit 整数，bit N = 1 表示 softirq 向量 N 待处理
    // 例如 pending=0b1000001 → bit 0 (HI) 和 bit 6 (TASKLET) 有请求

    softirq_handle_begin();   // softirq context 进入计数 +1

    /* ── ② 重置 pending，然后开中断执行 ── */
restart:
    set_softirq_pending(0);   // 清空 pending 位图
    // ★ 为什么必须先清空？
    //    pending 局部变量保存了"本次要处理的位图快照"。
    //    清掉实际寄存器后，action 执行期间新 raise 的 softirq
    //    （通过 or_softirq_pending）会从一个干净的 0 开始置位。
    //
    //    如果不清，第 ⑥ 步读到的 pending 会是：
    //      旧 bit（处理前就有的） + 新 bit（处理期间新来的）
    //    → 无法区分"这个 bit 是已经处理过的旧事件还是新事件"
    //    → goto restart 后会重复处理已经处理过的 softirq 向量
    //
    //    清了之后，第 ⑥ 步读到的 pending 全部是处理期间新来的，
    //    不存在"旧 bit 残留"的问题，每次 restart 都是针对新事件。

    local_irq_enable();       // ★ 开中断！softirq 执行时允许硬件中断嵌套
    //    这是 softirq context 与 hardirq context 的关键区别：
    //    hardirq 中 IRQ 是关闭的，softirq 中 IRQ 是打开的。
    //    高优先级硬件中断可以抢占正在执行的 softirq。

    h = softirq_vec;          // 指向 softirq_vec[0] = HI_SOFTIRQ

    /* ── ③ 遍历 pending 位图 ── */
    while ((softirq_bit = ffs(pending))) {
    // ffs = find first set，从最低位(bit 0)开始找第一个 1
    // ffs(0b100100000) = 6 → TASKLET_SOFTIRQ
    // ffs(0b000000001) = 1 → HI_SOFTIRQ
    // ★ bit 0 (HI_SOFTIRQ) 总是最先执行——这是高优先级的实现机制
        unsigned int vec_nr;
        h += softirq_bit - 1;  // 定位到对应的 softirq_vec 条目
        vec_nr = h - softirq_vec;  // 计算向量编号（用于 trace）

        trace_softirq_entry(vec_nr);
        h->action(h);          // ← 执行 softirq 回调函数！
        // 例：如果 vec_nr=6 (TASKLET_SOFTIRQ)，这里调 tasklet_action()
        // tasklet_action 会摘下当前 CPU 的 tasklet_vec 链表并遍历执行
        trace_softirq_exit(vec_nr);

        h++;                   // 指向下一个 softirq_vec 条目
        pending >>= softirq_bit;  // 移除已处理的 bit，准备检查下一个
        // 例：pending=0b100100000, softirq_bit=6
        //     pending >>= 6 → 0b100 (只剩下 bit 2 = NET_TX)
    }

    /* ── ④ 关中断，检查执行期间新产生的 pending ── */
    local_irq_disable();

    pending = local_softirq_pending();  // 读新 pending（第 ② 步之后新置位的）
    if (pending) {
    // 有新的 softirq 在刚才的执行过程中被 raise 了
    // 例如：tasklet_action 中执行了 t->callback(t)，callback 又调了
    //       tasklet_schedule 再次触发同一个 softirq
        if (time_before(jiffies, end)      // 还在 2ms 时限内
            && !need_resched()              // 没有更高优先级的任务要运行
            && --max_restart)               // 还没超过 10 次重入限制
            goto restart;                   // 继续执行（在中断返回路径上）
        // 三个条件任一不满足：
        wakeup_softirqd();                  // 交给 ksoftirqd 慢慢处理
        // ksoftirqd 是一个可被抢占的内核线程，不会饿死用户态进程
    }

    account_softirq_exit(current);
    lockdep_softirq_end(in_hardirq);
    softirq_handle_end();     // softirq context 退出计数 -1
    current_restore_flags(old_flags, PF_MEMALLOC);
}
```

**为什么需要 MAX_SOFTIRQ_TIME 和 MAX_SOFTIRQ_RESTART 两个限制？**

如果 softirq 处理中不断有新的 pending 被触发（例如网络高负载下 `NET_RX_SOFTIRQ` 执行中又有新包到达，handler 再次 raise 自己），`goto restart` 会一直执行，用户态进程得不到 CPU。

- `MAX_SOFTIRQ_TIME = 2ms`（`kernel/softirq.c:481`）：软中断总执行时间不超过 2ms
- `MAX_SOFTIRQ_RESTART = 10`（`kernel/softirq.c:482`）：最多重入 10 次

任一超限就不再继续，唤醒 ksoftirqd 在进程上下文中慢慢处理。ksoftirqd 可以被抢占，所以用户态进程仍然能得到 CPU 时间。

**`ffs` 从 bit 0 开始遍历的优先级效果：**

| softirq 向量 | bit 位置 | ffs 顺序 |
|-------------|---------|---------|
| HI_SOFTIRQ | 0 | 最先执行 |
| TIMER_SOFTIRQ | 1 | 第 2 |
| NET_TX_SOFTIRQ | 2 | 第 3 |
| NET_RX_SOFTIRQ | 3 | 第 4 |
| BLOCK_SOFTIRQ | 4 | 第 5 |
| IRQ_POLL_SOFTIRQ | 5 | 第 6 |
| TASKLET_SOFTIRQ | 6 | 第 7 |
| SCHED_SOFTIRQ | 7 | 第 8 |
| HRTIMER_SOFTIRQ | 8 | 第 9 |
| RCU_SOFTIRQ | 9 | 最后执行 |

bit 位置越小的向量越先执行。这就是 HI_SOFTIRQ（高优先级 tasklet）先于 TASKLET_SOFTIRQ（普通 tasklet）执行的原因。

### 2.6 ksoftirqd：per-CPU 守护线程

§2.5 分析了 `handle_softirqs` 的循环逻辑。它有**两个调用者**（§2.1 已介绍）：

| 调用者 | 调用方式 | 执行上下文 |
|--------|---------|-----------|
| `__do_softirq()` | `handle_softirqs(false)` | 中断返回路径（§2.5 分析的就是这个）|
| `run_ksoftirqd()` | `handle_softirqs(true)` | ksoftirqd 内核线程 |

`run_ksoftirqd` 的代码很简单：

两条路径的触发和执行不是孤立的，而是通过三个环节串联：

```
                    ┌──────────────────┐
                    │  raise_softirq()  │ ← 输入端：谁都可以调，置 pending
                    └────────┬─────────┘
                             │
              ┌──────────────┴──────────────┐
              │ in_interrupt() 的值         │
              │（调 raise_softirq 时的上下文）│
              ├──────────────┬──────────────┤
              │ true         │ false        │
              │（中断上下文中）│（进程上下文中）│
              └──────┬───────┴──────┬───────┘
                     │              │
             只置 pending    置 pending +
             不唤醒 ksoftirqd  唤醒 ksoftirqd
                     │              │
                     ▼              ▼
          ┌─────────────────┐   ┌──────────────────┐
          │ __irq_exit_rcu │   │ ksoftirqd 被调度  │
          │ → invoke_softirq│   │ → run_ksoftirqd  │
          └───────┬─────────┘   └────────┬─────────┘
                  │                      │
          ┌───────┴───────────┐     ┌────┴────┐
          │ force_irqthreads? │     │路径 B   │
          │ (invoke_softirq内)│     │handle_  │
          ├────────┬──────────┤     │softirqs │
          │ no     │ yes      │     │(true)   │
          │(默认)  │          │     └─────────┘
          ├────────┴──────────┤
          │ 路径 A            │
          │ __do_softirq      │
          │ → handle_softirqs │
          │   (false)         │
          └────────┬──────────┘
                   │
          ┌────────┴──────────────┐
          │ 处理完，检查新 pending │
          ├───────┬───────────────┤
          │ 有    │ 没有          │
          │ 且    │ → 返回        │
          │未超限 │               │
          │→继续  │               │
          │ 超限  │               │
          │→唤醒  │               │
          │ ksoft │               │
          │ irqd  │               │
          └───────┴───────────────┘
                   │ 超限时
                   ▼
          ┌──────────────────┐
          │ wakeup_softirqd()│ ← 路径 A 超限后也切到路径 B
          │ → ksoftirqd      │
          └──────────────────┘
```

**三个场景走哪个路径：**

```
场景一：顶半部中调 tasklet_schedule → raise_softirq(TASKLET_SOFTIRQ)
  → in_interrupt()=true → 只置 pending
  → 中断返回 → __irq_exit_rcu → invoke_softirq → __do_softirq（路径 A）
  → 紧贴顶半部执行 tasklet_action

场景二：进程上下文中调 raise_softirq
  → in_interrupt()=false → 置 pending + wakeup_softirqd
  → ksoftirqd 被调度时执行（路径 B）

场景三：路径 A 执行中不断有新 pending，2ms/10 次超限
  → wakeup_softirqd（切到路径 B）
  → handle_softirqs 返回 → eret 恢复被中断的进程
  → ksoftirqd 稍后被调度时继续处理
```

ksoftirqd 本身就是一个简单循环：

```c
// kernel/softirq.c:924
static void run_ksoftirqd(unsigned int cpu)
{
    ksoftirqd_run_begin();
    if (local_softirq_pending()) {
        handle_softirqs(true);           // 参数 true 表示在 ksoftirqd 中执行
        ksoftirqd_run_end();
        cond_resched();                  // 处理完让出 CPU
        return;
    }
    ksoftirqd_run_end();
}
```

ksoftirqd 通过 `smpboot_register_percpu_thread` 注册（`kernel/softirq.c:970`）。

**总结`invoke_softirq` 在整个路径中的角色：** 它是 `__irq_exit_rcu` → 执行 softirq 之间的**输出端决策点**——决定是直接执行（路径 A）还是唤醒 ksoftirqd（路径 B）：

```c
// kernel/softirq.c:425 (非 PREEMPT_RT)
static inline void invoke_softirq(void)
{
    if (!force_irqthreads() || !__this_cpu_read(ksoftirqd)) {
        __do_softirq();           // → 路径 A：直接执行（默认）
    } else {
        wakeup_softirqd();        // → 路径 B：交给 ksoftirqd
    }
}
```

`if (!force_irqthreads() || !__this_cpu_read(ksoftirqd))` 走路径 B 需要**两个条件同时为真**：

| `force_irqthreads()` | `__this_cpu_read(ksoftirqd)` | 走哪条路 |
|---------------------|-----------------------------|---------|
| false（默认） | 无所谓 | 路径 A：直接执行 |
| true | false（ksoftirqd 还没创建）| 路径 A：直接执行 |
| true | true（ksoftirqd 已创建）| **路径 B：唤醒 ksoftirqd** |

所以走路径 B 的唯一场景是：**内核参数 `threadirqs` 被设置，且当前 CPU 的 ksoftirqd 线程已创建**。默认情况下 `force_irqthreads()` 为 false，条件永远为 true，`invoke_softirq` 永远不会调 `wakeup_softirqd()`。

那靠什么触发路径 B？**不是靠 `invoke_softirq`**，而是另外两处直接调 `wakeup_softirqd()` 的入口：

1. `handle_softirqs` 超限（2ms/10次）→ 直接 `wakeup_softirqd()`
2. `raise_softirq_irqoff` 在进程上下文 → 直接 `wakeup_softirqd()`

这两处不经过 `invoke_softirq`。`invoke_softirq` 只在 `__irq_exit_rcu` 中调用（中断返回路径），它只是用 `threadirqs` 参数提供了一个**兜底开关**——强制把"中断返回时直接执行"也改为线程化。

与 `raise_softirq`（输入端）不同——`raise_softirq` 看的是"谁调的我"（中断/进程上下文），`invoke_softirq` 看的是"系统配置"（是否设了 `threadirqs` 内核参数）。两者独立决策，但共同决定了 softirq 最终在哪个上下文中执行。

### 2.7 初始化：softirq_init

softirq 机制在内核启动的哪个节点被初始化？答案在 `start_kernel` 中：

```c
// init/main.c
start_kernel()
  ├── init_IRQ()              ← 安装 GIC + EXTI（03 篇全文）
  ├── softirq_init()          ← 注册 TASKLET/HI_SOFTIRQ
  ├── time_init()             ← 注册 TIMER_SOFTIRQ
  ├── workqueue_init_early()  ← 创建 system_wq 等全局队列
  ├── ...（大量子系统初始化）...
  ├── workqueue_init()        ← 创建 kworker 线程
  ├── ...（更多初始化）...
  └── arch_call_rest_init()
       └── do_basic_setup()
            └── do_initcalls()
                 ├── spawn_ksoftirqd()    ← early_initcall，创建 ksoftirqd 线程
                 ├── 网络 initcall        ← 注册 NET_TX/NET_RX_SOFTIRQ
                 └── 块设备 initcall      ← 注册 BLOCK_SOFTIRQ
```

`softirq_init()` 本身的代码很简单：

```c
// kernel/softirq.c:904
void __init softirq_init(void)
{
    int cpu;

    for_each_possible_cpu(cpu) {
        per_cpu(tasklet_vec, cpu).tail =
            &per_cpu(tasklet_vec, cpu).head;
        per_cpu(tasklet_hi_vec, cpu).tail =
            &per_cpu(tasklet_hi_vec, cpu).head;
    }

    open_softirq(TASKLET_SOFTIRQ, tasklet_action);
    open_softirq(HI_SOFTIRQ, tasklet_hi_action);
}
```

它做了三件事：

**① 初始化 per-CPU tasklet 链表头。** `tasklet_vec` 和 `tasklet_hi_vec` 是每个 CPU 一个的链表，存放该 CPU 上被调度但尚未执行的 tasklet。初始态下 head = NULL，tail = &head（空链表的标准表示法）。

**② 注册 TASKLET_SOFTIRQ → tasklet_action。** 当 `tasklet_schedule()` 触发 `TASKLET_SOFTIRQ` 时，`handle_softirqs` 在 ffs 遍历到 bit 6 时会调 `tasklet_action`，它摘下当前 CPU 的 tasklet_vec 链表，逐个执行回调。

**③ 注册 HI_SOFTIRQ → tasklet_hi_action。** 与上面相同，但对应高优先级 tasklet（bit 0，先执行）。

**关键观察：softirq_init 只注册了 TASKLET 和 HI 两个向量。** 其余 8 个向量什么时候注册？

| 向量 | 谁注册 | 什么时候 |
|------|--------|---------|
| TIMER_SOFTIRQ | `timer_init()` | `start_kernel` 中，`softirq_init` 之后 |
| HRTIMER_SOFTIRQ | `hrtimer_init()` | `start_kernel` 中 |
| SCHED_SOFTIRQ | `sched_init()` | `start_kernel` 中 |
| RCU_SOFTIRQ | `rcu_init()` | `start_kernel` 中 |
| NET_TX/RX_SOFTIRQ | 网络子系统 | `do_initcalls` → `subsys_initcall` |
| BLOCK_SOFTIRQ | 块子系统 | `do_initcalls` → `subsys_initcall` |
| IRQ_POLL_SOFTIRQ | 块子系统 | `do_initcalls` |

前四个在 `start_kernel` 中注册——它们在 `mm_init` 之前就已经可用。后三个在 `do_initcalls` 阶段注册——此时内存管理、调度器、中断系统已经完全就绪。

**ksoftirqd 线程的创建时机：**

ksoftirqd 线程不是 `softirq_init` 创建的，而是在 `early_initcall(spawn_ksoftirqd)` 中（`kernel/softirq.c:977`）：

```c
static __init int spawn_ksoftirqd(void)
{
    cpuhp_setup_state_nocalls(CPUHP_SOFTIRQ_DEAD, "softirq:dead", NULL,
                  takeover_tasklets);
    BUG_ON(smpboot_register_percpu_thread(&softirq_threads));
    return 0;
}
early_initcall(spawn_ksoftirqd);
```

这意味着在 `start_kernel` 执行期间（`do_initcalls` 之前），**ksoftirqd 线程还不存在**。此时如果触发 softirq，raise_softirq 中的 `should_wake_ksoftirqd()` 返回 false，所以 softirq 只能在 `irq_exit_rcu` 路径中直接执行（`invoke_softirq` → `__do_softirq`）。直到 `early_initcall` 阶段 `spawn_ksoftirqd` 执行后，ksoftirqd 线程才上线，`should_wake_ksoftirqd` 才开始返回 true。

softirq 机制的来龙去脉清楚了。但驱动开发者一般不直接使用 softirq——它太底层（需要考虑并发、不能 sleep）。驱动用得最多的是 **tasklet**，它是 softirq 之上的一层封装。下面分析 tasklet。

---

## 3. Tasklet

Tasklet 基于 softirq 的 `TASKLET_SOFTIRQ` 向量实现。它的完整执行路径：

```
驱动调 tasklet_schedule(&t)
  → test_and_set_bit(SCHED, &t->state)   ← 防重入
  → 将 t 挂入当前 CPU 的 tasklet_vec 链表
  → raise_softirq_irqoff(TASKLET_SOFTIRQ) ← 置 pending bit 6

中断返回或 ksoftirqd 执行时：
  handle_softirqs()
    → ffs(pending) 遍历到 bit 6
    → softirq_vec[6].action = tasklet_action
      → tasklet_action_common()
        → 摘下当前 CPU 的 tasklet_vec 链表
        → 遍历每个 tasklet_struct：
            1. tasklet_trylock() 获取 RUN 锁
            2. 检查 count==0（使能状态）
            3. 调 t->callback(t) 或 t->func(t->data)
            4. tasklet_unlock() 释放 RUN 锁
        → 被其他 CPU 占用的重入链表末尾
```

tasklet 与直接使用 softirq 的核心区别：**串行化**。`tasklet_trylock` 保证同一个 tasklet 在同一时刻最多在一个 CPU 上执行——驱动开发者不需要考虑并发重入问题。

### 3.1 核心数据结构

```c
// include/linux/interrupt.h:642
struct tasklet_struct
{
    struct tasklet_struct *next;       // 链表节点
    unsigned long state;               // bit 0: SCHED, bit 1: RUN
    atomic_t count;                    // 0=使能，>0=禁用
    bool use_callback;                 // true=新接口 callback, false=旧接口 func
    union {
        void (*func)(unsigned long data);
        void (*callback)(struct tasklet_struct *t);
    };
    unsigned long data;                // func 的参数
};
```

**state 字段的两个标志位：**

```c
// include/linux/interrupt.h:684
enum
{
    TASKLET_STATE_SCHED,    /* Tasklet is scheduled for execution */
    TASKLET_STATE_RUN       /* Tasklet is running (SMP only) */
};
```

| 标志 | bit | 谁设 | 谁清 | 含义 |
|------|-----|------|------|------|
| TASKLET_STATE_SCHED | 0 | tasklet_schedule | tasklet_action 执行前 | 已调度，等待执行 |
| TASKLET_STATE_RUN | 1 | tasklet_trylock | tasklet_unlock | 正在执行（SMP） |

**count 字段控制使能/禁用：**

```c
// include/linux/interrupt.h:723
static inline void tasklet_disable_nosync(struct tasklet_struct *t)
{
    atomic_inc(&t->count);
    smp_mb__after_atomic();
}

// include/linux/interrupt.h:733
static inline void tasklet_disable(struct tasklet_struct *t)
{
    tasklet_disable_nosync(t);
    tasklet_unlock_wait(t);           // 若 count 从 0→1，等待 RUN 结束
}

// include/linux/interrupt.h:740
static inline void tasklet_enable(struct tasklet_struct *t)
{
    smp_mb__before_atomic();
    atomic_dec(&t->count);
}
```

`count = 0` 表示使能，`count > 0` 表示禁用（可嵌套禁用）。`tasklet_disable` 如果发现 count 从 0 变为 1，会同步等待当前正在执行的 tasklet 完成——这保证 disable 返回后，tasklet 一定不会再执行。

**per-CPU 链表头：**

```c
// kernel/softirq.c:717
struct tasklet_head {
    struct tasklet_struct *head;
    struct tasklet_struct **tail;
};

static DEFINE_PER_CPU(struct tasklet_head, tasklet_vec);       // 普通优先级
static DEFINE_PER_CPU(struct tasklet_head, tasklet_hi_vec);    // 高优先级
```

每个 CPU 维护两个链表，分别对应 `TASKLET_SOFTIRQ` 和 `HI_SOFTIRQ`。

### 3.2 调度：tasklet_schedule

```c
// include/linux/interrupt.h:709
static inline void tasklet_schedule(struct tasklet_struct *t)
{
    if (!test_and_set_bit(TASKLET_STATE_SCHED, &t->state))
        __tasklet_schedule(t);
}

// kernel/softirq.c:741
void __tasklet_schedule(struct tasklet_struct *t)
{
    __tasklet_schedule_common(t, &tasklet_vec, TASKLET_SOFTIRQ);
}

// kernel/softirq.c:725
static void __tasklet_schedule_common(struct tasklet_struct *t,
                      struct tasklet_head __percpu *headp,
                      unsigned int softirq_nr)
{
    struct tasklet_head *head;
    unsigned long flags;

    local_irq_save(flags);
    head = this_cpu_ptr(headp);
    t->next = NULL;
    *head->tail = t;
    head->tail = &(t->next);
    raise_softirq_irqoff(softirq_nr);       // → 置 pending TASKLET_SOFTIRQ bit
    local_irq_restore(flags);
}
```

**三个要点：**

1. **SCHED 标志防重入**：`test_and_set_bit` 检查 SCHED 位——如果已经置位，说明 tasklet 已经在等待执行，第二次调度被忽略。这就是**事件合并**——连续触发多次只执行一次。

2. **per-CPU 链表**：挂入当前 CPU 的 tasklet_vec，不需要跨 CPU 锁。

3. **raise_softirq**：置位 `TASKLET_SOFTIRQ` 的 pending 位，触发 softirq 路径。

高优先级版本走 `HI_SOFTIRQ` 向量：

```c
// include/linux/interrupt.h:717
static inline void tasklet_hi_schedule(struct tasklet_struct *t)
{
    if (!test_and_set_bit(TASKLET_STATE_SCHED, &t->state))
        __tasklet_hi_schedule(t);
}
```

### 3.3 执行：tasklet_action

`tasklet_action` 是 `TASKLET_SOFTIRQ` 向量的 action 回调，`tasklet_hi_action` 是 `HI_SOFTIRQ` 向量的回调。两者调用同一个函数 `tasklet_action_common`，只是传入的链表头和 softirq 编号不同：

```c
// kernel/softirq.c:814
static __latent_entropy void tasklet_action(struct softirq_action *a)
{
    tasklet_action_common(a, this_cpu_ptr(&tasklet_vec), TASKLET_SOFTIRQ);
}

// kernel/softirq.c:819
static __latent_entropy void tasklet_hi_action(struct softirq_action *a)
{
    tasklet_action_common(a, this_cpu_ptr(&tasklet_hi_vec), HI_SOFTIRQ);
}
```

核心逻辑在 `tasklet_action_common`：

```c
// kernel/softirq.c:769
static void tasklet_action_common(struct softirq_action *a,
                  struct tasklet_head *tl_head,
                  unsigned int softirq_nr)
{
    struct tasklet_struct *list;

    /* ── ① 摘下当前 CPU 的整个 tasklet 链表 ── */
    local_irq_disable();
    list = tl_head->head;              // 取链表头
    tl_head->head = NULL;              // 清空链表
    tl_head->tail = &tl_head->head;    // tail 指向 head（空链表状态）
    local_irq_enable();
    // ★ 为什么摘下整个链表而非逐个取？
    //    ① 开中断后新调度的 tasklet 会挂入空的链表，不会和当前正在处理的混在一起
    //    ② 加锁粒度最小——只在摘链表时关中断，处理过程中中断是开的
    //    ③ 未执行完的 tasklet（RUN 锁被占）可以重入空链表末尾

    /* ── ② 遍历链表 ── */
    while (list) {
        struct tasklet_struct *t = list;
        list = list->next;

        if (tasklet_trylock(t)) {      // 尝试获取 RUN 锁（SMP 串行化关键）
            // test_and_set_bit(TASKLET_STATE_RUN)
            // 返回 0 → 获取锁成功，继续
            // 返回 1 → 其他 CPU 正在执行 → 走 else 分支

            if (!atomic_read(&t->count)) {
                // count=0：使能状态，可以执行
                if (tasklet_clear_sched(t)) {
                    // 清除 SCHED 标志，然后执行回调
                    if (t->use_callback)
                        t->callback(t);         // 新接口
                    else
                        t->func(t->data);       // 旧接口
                }
                tasklet_unlock(t);    // 释放 RUN 锁
                continue;
            }
            // count>0：被 tasklet_disable 禁用了
            // 不解锁 SCHED，保持"已调度"状态，等 enable 后重新触发
            tasklet_unlock(t);
        }
        /* ── ③ RUN 锁被其他 CPU 持有 ── */
        // 同一个 tasklet 正在另一个 CPU 上执行
        // 不能跳过——这个 tasklet 还需要被执行
        // 将其重入链表尾部，等下次 softirq 触发
        local_irq_disable();
        t->next = NULL;
        *tl_head->tail = t;            // 挂入链表末尾
        tl_head->tail = &t->next;
        __raise_softirq_irqoff(softirq_nr);  // 重新触发 softirq
        local_irq_enable();
        // ★ 这个重新触发保证：被其他 CPU 占用的 tasklet 不会丢，
        //    等 RUN 锁释放后，下次 softirq 会再次尝试执行它
    }
}
```

**串行化原理**（这也是 tasklet 与直接使用 softirq 的核心区别）：

tasklet 的串行化靠的是 `tasklet_struct` 的 `state` 字段中的两个标志位配合：

```c
// include/linux/interrupt.h:684
enum
{
    TASKLET_STATE_SCHED,      // bit 0：已调度，等待执行
    TASKLET_STATE_RUN         // bit 1：正在执行（SMP）
};
```

**`tasklet_trylock` — 原子的 test_and_set_bit：**

```c
// include/linux/interrupt.h:691
static inline int tasklet_trylock(struct tasklet_struct *t)
{
    return !test_and_set_bit(TASKLET_STATE_RUN, &(t)->state);
}
```

`test_and_set_bit` 是 CPU 提供的一条原子指令（ARM64 上对应 `LDX`/`STX` 指令对）：
1. **原子地**读取 `t->state` 的 bit 1
2. **原子地**将 bit 1 置为 1
3. 返回 bit 1 的**旧值**

| 旧值 | 返回值 | `!返回值` | 含义 |
|------|--------|----------|------|
| 0（没人在执行）| 0 | **1 = true** | 获取锁成功——可以执行 |
| 1（别人在执行）| 1 | **0 = false** | 获取锁失败——重入链表 |

因为是**原子操作**，两个 CPU 同时调 `tasklet_trylock(t)` 时：

```
CPU0：test_and_set_bit(RUN)          CPU1：test_and_set_bit(RUN)
  读 t->state bit 1 = 0                读 t->state bit 1 = 1
  写 t->state bit 1 = 1（成功）          写 t->state bit 1 = 1（已=1，不变）
  返回 0 → 获取锁                       返回 1 → 失败
  → 执行 callback                      → 重入链表
```

`test_and_set_bit` 保证了**读和写之间不会被其他核打断**——不可能两个 CPU 同时读到 0。这就是串行化的硬件基础。

**`tasklet_clear_sched` — SCHED 标志的清除：**

```c
// kernel/softirq.c:755
static bool tasklet_clear_sched(struct tasklet_struct *t)
{
    if (test_and_clear_bit(TASKLET_STATE_SCHED, &t->state)) {
        wake_up_var(&t->state);      // 唤醒 tasklet_kill 的等待者
        return true;
    }
    ...
    return false;
}
```

同样用原子的 `test_and_clear_bit` 清除 SCHED 标志。`tasklet_kill` 通过等待 SCHED 标志清除来确认 tasklet 不会再执行。

**`tasklet_unlock` — RUN 锁的释放：**

```c
void tasklet_unlock(struct tasklet_struct *t)
{
    smp_mb__before_atomic();         // 保证 callback 的写操作在解锁前对其他核可见
    clear_and_wake_up_bit(TASKLET_STATE_RUN, &t->state);
}
```

清除 RUN 位后，其他 CPU 上等待该 tasklet 的 `tasklet_trylock` 才能成功，从而在下次 softirq 中执行重入链表中的这个 tasklet。

```
CPU0                              CPU1
───                               ───
tasklet_schedule(t)               tasklet_schedule(t)
  → SCHED=1，挂入 CPU0 链表         → SCHED 已置位 → 跳过
↓                                 ↓
softirq 触发                       softirq 触发
  → tasklet_action                  → tasklet_action
    → 摘下链表                        → 摘下自己的链表（不同 CPU）
    → tasklet_trylock(t)             → tasklet_trylock(t)
      → RUN=0，取锁成功               → RUN=1，取锁失败！
      → 执行 callback                  → 重入链表末尾，重新触发 softirq
      → clear RUN+SCHED
```

`tasklet_trylock` 基于 `test_and_set_bit(TASKLET_STATE_RUN)`，保证**同一个 tasklet 在同一时刻最多在一个 CPU 上执行**。不同类型的 tasklet（不同的 tasklet_struct 实例）不互斥——它们只是在同一个链表上的不同节点，可以分别在 CPU0 和 CPU1 上并行执行。

**两种优先级：**

| 优先级 | softirq 向量 | nr | 函数 | 链表 |
|--------|-------------|----|------|------|
| 高 | HI_SOFTIRQ | 0 | tasklet_hi_action | tasklet_hi_vec |
| 普通 | TASKLET_SOFTIRQ | 6 | tasklet_action | tasklet_vec |

高优先级 tasklet 在 `handle_softirqs` 的 ffs 遍历中先执行（bit 0），普通 tasklet 后执行（bit 6）。

### 3.4 生命周期与使用

```c
// 静态声明
DECLARE_TASKLET(my_tasklet, my_callback);           // 使能状态 (count=0)
DECLARE_TASKLET_DISABLED(my_tasklet, my_callback);  // 禁用状态 (count=1)

// 动态初始化
void tasklet_setup(struct tasklet_struct *t,
                   void (*callback)(struct tasklet_struct *));

// 调度
tasklet_schedule(&my_tasklet);      // 普通优先级
tasklet_hi_schedule(&my_tasklet);   // 高优先级

// 控制
tasklet_disable(&my_tasklet);       // 禁用 + 同步等待执行完
tasklet_enable(&my_tasklet);        // 恢复

// 销毁
tasklet_kill(&my_tasklet);          // 确保不再调度并等待完成
```

tasklet 适用于：**不需要 sleep、执行时间短（<1ms）、不需要延迟/定时、希望自动串行化避免并发问题**的推迟任务。它是驱动开发中最常用的下半部机制之一。

但 tasklet 仍然受 softirq context 的约束——不可 sleep。如果推迟任务需要执行 `i2c_transfer`、`copy_from_user`、`mutex_lock` 等可能调度（schedule）的操作，tasklet 就不够用了。这时候需要 workqueue 或线程化 IRQ。

---

## 4. Workqueue

### 4.1 完整路径

workqueue 的完整执行路径：

```
驱动调 schedule_work(&work) 或 queue_work(wq, &work)
  → test_and_set_bit(WORK_STRUCT_PENDING)   ← 防重入
  → 找到当前 CPU 对应的 pool_workqueue
  → insert_work(pwq, work, &pool->worklist)  ← 挂入线程池
  → wake_up_worker(pool)                     ← 唤醒空闲 worker

worker 线程被调度时：
  worker_thread()
    → process_one_work(worker, work)
      → 从 pool->worklist 取出 work
      → work->func(work)                    ← 执行回调！

无 work 时：
  → schedule()                              ← 睡眠等待
```

tasklet 的问题——它在 softirq context 中执行，不能 sleep。workqueue 把执行上下文从 softirq context 换成了**进程 context**——worker 线程可以 sleep、可以被抢占、持有 mutex、调用 `copy_from_user`。

### 4.2 从 tasklet 到 workqueue

v3.2 之前的旧式 workqueue：每个 workqueue 一个内核线程。

v3.2 之前的旧式 workqueue：每个 workqueue 一个内核线程。

```
旧式：workqueue = 内核线程
  workqueue_1 → kthread_worker_1
  workqueue_2 → kthread_worker_2      ← N 个 workqueue = N 个线程
  ...                                    ← 大量线程空转浪费内存
```

假设系统有 50 个驱动各自创建 workqueue，就是 50 个常驻内核线程，大部分时间在睡眠。v3.2 引入 CMWQ（Concurrency Managed Workqueue），核心思想：**将 workqueue（对外接口）和 worker 线程（执行者）解耦。**

```
CMWQ（v3.2+）:

workqueue_struct（接口层：驱动看到的东西）
  │ wq->pwqs 链表
  └── pool_workqueue（per-CPU 桥梁）
        │ pwq->pool
        └── worker_pool（线程池）
              ├── worker_1 ← kworker/0:0    ← 线程池共享
              ├── worker_2 ← kworker/0:1
              └── ...

所有 workqueue 共享同一个 worker_pool
```

每个 CPU 两个 worker_pool：`normal`（普通优先级）和 `highpri`（高优先级）。系统上所有 workqueue（无论多少个）都共享这两个线程池。

### 4.3 核心数据结构

workqueue 体系涉及四个结构体，从高到低四层：

```c
// include/linux/workqueue.h

// —— 第 1 层：工作节点（具体任务）——
struct work_struct {
    atomic_long_t data;              // 状态标志 + pool ID
    struct list_head entry;          // 链表节点
    work_func_t func;                // 回调函数
};

// —— 第 2 层：workqueue（驱动注册的接口）——
struct workqueue_struct {
    struct list_head pwqs;           // per-CPU pool_workqueue 链表
    char name[WQ_NAME_LEN];          // 名称（ps 可见）
    unsigned int flags;              // WQ_UNBOUND / WQ_FREEZABLE 等
};

// —— 第 3 层：pool_workqueue（桥梁）——
struct pool_workqueue {
    struct worker_pool *pool;        // 关联的线程池
    struct workqueue_struct *wq;     // 关联的 workqueue
    struct list_head delayed_works;  // 延迟 work 链表
    ...
};

// —— 第 4 层：worker_pool（线程池）——
struct worker_pool {
    spinlock_t lock;
    struct list_head worklist;       // 待处理 work 链表
    struct list_head workers;        // 所有 worker 线程
    struct list_head idle_list;      // 空闲 worker
    int nr_running;                  // 正在运行的 worker 数
};

struct worker {
    struct task_struct *task;        // 内核线程
    struct worker_pool *pool;        // 所属线程池
    struct list_head scheduled;      // 正在执行的 work 链表
};
```

**四层关系图：**

```
workqueue_struct（你注册的接口：system_wq、my_wq）
  │ wq->pwqs
  │
  ├── pool_workqueue[CPU0]
  │     │ pwq->pool
  │     └── worker_pool[0]（CPU0 普通优先级）
  │           │ pool->worklist    ← 待处理的 work_struct
  │           │ pool->workers     ← kworker/0:0, kworker/0:1
  │           │ pool->idle_list   ← 空闲 worker
  │
  └── pool_workqueue[CPU1]
        │
        └── worker_pool[0]（CPU1 普通优先级）
              │ ...
```

当驱动调 `queue_work(system_wq, work)` 时，work 被放入当前 CPU 对应的 `worker_pool` 的 `worklist`。然后同一个 pool 中的某个空闲 worker 被唤醒，从 worklist 中取出 work 执行。

### 4.4 全局 workqueue

```c
// kernel/workqueue.c:423
struct workqueue_struct *system_wq __read_mostly;
struct workqueue_struct *system_highpri_wq __read_mostly;
struct workqueue_struct *system_unbound_wq __read_mostly;
struct workqueue_struct *system_freezable_wq __read_mostly;
struct workqueue_struct *system_power_efficient_wq __read_mostly;
```

| 全局 workqueue | 特性 | 适用场景 |
|---------------|------|---------|
| `system_wq` | per-CPU，普通优先级 | 默认 `schedule_work()` |
| `system_highpri_wq` | per-CPU，高优先级 | 需要低延迟 |
| `system_unbound_wq` | 不绑定 CPU | 避免缓存跳跃 |
| `system_freezable_wq` | 休眠时排空 | PM 相关 |
| `system_power_efficient_wq` | 非 CPU 绑定的省电版 | 移动设备 |

驱动开发者一般只用 `system_wq`——`schedule_work()` 就是挂入 `system_wq`。

### 4.5 调度：schedule_work → queue_work

```c
// include/linux/workqueue.h:621
static inline bool schedule_work(struct work_struct *work)
{
    return queue_work(system_wq, work);
}

// kernel/workqueue.c:1835
bool queue_work(struct workqueue_struct *wq, struct work_struct *work)
{
    return queue_work_on(WORK_CPU_UNBOUND, wq, work);
}
```

`queue_work_on` 最终调用 `__queue_work`（`kernel/workqueue.c:1703`）：

```c
static void __queue_work(int cpu, struct workqueue_struct *wq,
             struct work_struct *work)
{
    struct pool_workqueue *pwq;
    struct worker_pool *pool;

    // ① WORK_STRUCT_PENDING 位检查——防重入
    if (unlikely(!test_and_set_bit(WORK_STRUCT_PENDING, work_data_bits(work))))
        goto out;  // 已经在队列中

    // ② 找到当前 CPU 对应的 pool_workqueue 和 worker_pool
    pwq = get_work_pwq(work, wq);
    pool = pwq->pool;

    // ③ 将 work 插入线程池的 worklist
    insert_work(pwq, work, &pool->worklist, work_flags);

    // ④ 唤醒空闲 worker
    wake_up_worker(pool);
}
```

**`mod_delayed_work` — gpio-keys 使用的带延迟版本：**

```c
// kernel/workqueue.c
bool mod_delayed_work(struct workqueue_struct *wq, struct delayed_work *dwork,
                      unsigned long delay)
{
    if (delay == 0)
        return queue_work(wq, &dwork->work);
    // delay > 0: 启动定时器，到期后调 delayed_work_timer_fn
    // → 到期时将 work 移入 pool->worklist
    return queue_delayed_work(wq, dwork, delay);
}
```

`mod_delayed_work` 的 "mod"（modify）语义：如果 work 已经在定时器等待中，重置定时器时间。gpio-keys 正是用这个实现按键防抖——连续按键时每次 reset 定时器，只有停止按键超过 debounce 时间后才真正执行 work。

### 4.6 执行：worker_thread → process_one_work

```c
// kernel/workqueue.c:2737
static int worker_thread(void *__worker)
{
    struct worker *worker = __worker;
    struct worker_pool *pool = worker->pool;

    set_pf_worker(true);
woke_up:
    raw_spin_lock_irq(&pool->lock);

    worker_leave_idle(worker);
recheck:
    if (!need_more_worker(pool))
        goto sleep;

    if (unlikely(!may_start_working(pool)) && manage_workers(worker))
        goto recheck;

    // 取第一个 work 执行
    process_one_work(worker, work);
    goto woke_up;

sleep:
    worker_enter_idle(worker);
    __set_current_state(TASK_IDLE);
    raw_spin_unlock_irq(&pool->lock);
    schedule();                       // 没有 work → 睡眠
    goto woke_up;
}
```

`process_one_work`（`kernel/workqueue.c:2537`）负责执行 work 回调：

```c
static void process_one_work(struct worker *worker, struct work_struct *work)
{
    struct pool_workqueue *pwq = get_work_pwq(work);
    struct worker_pool *pool = worker->pool;

    // 将 work 从 pool->worklist 移到 worker->scheduled
    list_del_init(&work->entry);

    // 执行 work 回调
    trace_workqueue_execute_start(work);
    worker->current_func = work->func;
    worker->current_func(work);          // ← 执行回调！
    worker->current_func = NULL;
    trace_workqueue_execute_end(work);
    ...
}
```

**STM32MP257 上运行的 worker 线程：**

```bash
$ ps aux | grep kworker
root        11  0.0  0.0      0     0 ?        I    xx:xx   0:00 [kworker/0:0]
root        12  0.0  0.0      0     0 ?        I    xx:xx   0:00 [kworker/0:1H]
root        13  0.0  0.0      0     0 ?        I    xx:xx   0:00 [kworker/1:0]
root        14  0.0  0.0      0     0 ?        I    xx:xx   0:00 [kworker/1:1H]
                    │                                  │
                    │                                  └ H = highpri worker_pool
                    └ worker 序号（动态增加）
```

### 4.7 初始化：workqueue_init

workqueue 的初始化分两个阶段，分别在 `start_kernel` 的不同时间点调用。

#### 第一阶段：workqueue_init_early

在 `start_kernel` 早期（`mm_init` 刚完成、内存分配可用但 kthread 还不能创建时）调用：

```c
// init/main.c — start_kernel 中
workqueue_init_early();    // 早，~line 965
...
workqueue_init();          // 晚，~line 1538
```

`workqueue_init_early`（`kernel/workqueue.c:6566`）做了以下工作：

**① 分配 per-CPU worker_pool。** 每个 CPU 两个 pool（普通优先级 + 高优先级），调用 `init_worker_pool()` 初始化：

```c
// kernel/workqueue.c:6600
for_each_possible_cpu(cpu) {
    for_each_cpu_worker_pool(pool, cpu) {
        BUG_ON(init_worker_pool(pool));
        pool->cpu = cpu;
        pool->attrs->nice = std_nice[i++];  // 0 (normal) / -20 (highpri)
    }
}
```

此时 pool 的数据结构已分配好，但 pool 中没有 worker 线程（因为 kthread 还没就绪）。

**② 创建全局 workqueue：**

```c
// kernel/workqueue.c:6638
system_wq = alloc_workqueue("events", 0, 0);
system_highpri_wq = alloc_workqueue("events_highpri", WQ_HIGHPRI, 0);
system_long_wq = alloc_workqueue("events_long", 0, 0);
system_unbound_wq = alloc_workqueue("events_unbound", WQ_UNBOUND, WQ_MAX_ACTIVE);
system_freezable_wq = alloc_workqueue("events_freezable", WQ_FREEZABLE, 0);
system_power_efficient_wq = alloc_workqueue("events_power_efficient",
                          WQ_POWER_EFFICIENT, 0);
system_freezable_power_efficient_wq = alloc_workqueue(
    "events_freezable_power_efficient",
    WQ_FREEZABLE | WQ_POWER_EFFICIENT, 0);
```

每个 `alloc_workqueue` 创建一个 `workqueue_struct`，并为每个在线 CPU 创建对应的 `pool_workqueue`，关联到该 CPU 的 `worker_pool`。

此时驱动已经可以调用 `schedule_work()`——work 会被挂入 pool->worklist，但因为没有 worker 线程，**work 不会被执行**，只是排队。

#### 第二阶段：workqueue_init

在 `start_kernel` 晚期（kthread 已可以创建和调度时）调用（`kernel/workqueue.c:6704`）：

```c
void __init workqueue_init(void)
{
    wq_cpu_intensive_thresh_init();

    // ① 为每个 workqueue 创建 rescuer 线程（防止 ABBA 死锁）
    list_for_each_entry(wq, &workqueues, list) {
        WARN(init_rescuer(wq), "failed to create early rescuer for %s", wq->name);
    }

    // ② 为每个 per-CPU worker_pool 创建初始 worker 线程
    for_each_online_cpu(cpu) {
        for_each_cpu_worker_pool(pool, cpu) {
            pool->flags &= ~POOL_DISASSOCIATED;
            BUG_ON(!create_worker(pool));     // ← 创建第一个 kworker/N
        }
    }

    // ③ 为 unbound pool 创建初始 worker
    hash_for_each(unbound_pool_hash, bkt, pool, hash_node)
        BUG_ON(!create_worker(pool));

    wq_online = true;   // ← 从此 workqueue 开始真正执行 work
    wq_watchdog_init();
}
```

`create_worker` 内部调用 `kthread_create_on_node(worker_thread, worker, ...)`（`kernel/workqueue.c:2194`），创建内核线程并绑定到对应 worker_pool。

**初始化时序总结：**

```
时间 →

workqueue_init_early():
  └─ 分配 worker_pool 数据结构
  └─ 创建 system_wq 等全局 workqueue（但 worker 线程还没创建）
  └─ 此时 queue_work 可以调用，work 只排队不执行

[ 中间大量子系统初始化，包括 softirq_init、time_init 等 ]

workqueue_init():
  └─ create_worker(per-CPU pool)  → 创建 kworker/0:0, kworker/0:1, ...
  └─ create_worker(unbound pool)  → 创建 kworker/u4:0, ...
  └─ wq_online = true             → work 开始被执行
```

**"work 排队但不执行"这个中间状态很重要：** 在 `workqueue_init` 之前调用的 `schedule_work` 不会丢——work 挂在 pool->worklist 上，`workqueue_init` 创建 worker 后它们会被顺序执行。这保证了 `workqueue_init_early` 到 `workqueue_init` 之间所有排队的 work 都不会丢失。

workqueue 解决了"推迟执行且可以 sleep"的问题。但有一个场景 workqueue 不是最优的：**中断处理本身需要 sleep 且要求低延迟**。例如 I2C 触摸屏在中断中必须调 `i2c_transfer`（会 sleep），但如果把它放到共享的 workqueue 中，它要和其他 work 竞争线程池，调度延迟不可控。这时候需要线程化 IRQ——每个中断一个专用内核线程。

---

## 5. 线程化 IRQ

### 5.1 完整路径

线程化 IRQ 的执行路径与前三者完全不同，它的触发点不在 `raise_softirq`，而在 `handle_irq_event` 中：

```
驱动在 probe 中注册：
  request_threaded_irq(irq, handler=NULL, thread_fn=my_fn, IRQF_ONESHOT, ...)
    → action->handler = NULL（用默认 irq_default_primary_handler）
    → action->thread_fn = my_fn
    → action->thread = kthread_create(irq_thread, action, "irq/%d-%s", irq, name)
    → irq 线程创建但处于睡眠状态

中断触发时（顶半部）：
  handle_irq_event()
    → __handle_irq_event_percpu()
      → action->handler = irq_default_primary_handler
        → return IRQ_WAKE_THREAD
    → __irq_wake_thread(desc, action)
      → set_bit(IRQTF_RUNTHREAD, &action->thread_flags)
      → wake_up_process(action->thread)          ← 唤醒 irq 线程
    → do { } while (ret == IRQ_WAKE_THREAD)       ← 循环（见下文 IRQF_ONESHOT 解释）

irq 线程被调度时（下半部）：
  irq_thread()
    → irq_wait_for_interrupt()
      → test_and_clear_bit(IRQTF_RUNTHREAD) = true
      → 返回 0
    → handler_fn(desc, action)
      → action->thread_fn(irq, dev_id)            ← 执行驱动 thread_fn！
        → 可以 sleep（i2c_transfer、mutex_lock 等）
    → irq_finalize_oneshot(desc, action)          ← 电平触发重新使能
    → 回到 irq_wait_for_interrupt 等待下一次中断
```

### 5.2 与前三者的本质区别

线程化 IRQ 不是排队机制（不像 workqueue 有 worklist 排队），它是 `request_irq` 的一个可选特性。当驱动调用 `request_threaded_irq(irq, handler=NULL, thread_fn, ...)` 时，内核为这个中断创建一个**专用的内核线程**（SCHED_FIFO 实时优先级）。顶半部执行完毕后，handler 返回 `IRQ_WAKE_THREAD`，唤醒该线程执行 `thread_fn`。

### 5.3 irq_thread 执行流

```c
// kernel/irq/manage.c:1298
static int irq_thread(void *data)
{
    struct callback_head on_exit_work;
    struct irqaction *action = data;
    struct irq_desc *desc = irq_to_desc(action->irq);
    irqreturn_t (*handler_fn)(struct irq_desc *desc,
            struct irqaction *action);

    irq_thread_set_ready(desc, action);

    sched_set_fifo(current);                       // SCHED_FIFO 实时优先级

    if (force_irqthreads() && ...)
        handler_fn = irq_forced_thread_fn;
    else
        handler_fn = irq_thread_fn;

    while (!irq_wait_for_interrupt(action)) {       // 等待 IRQTF_RUNTHREAD
        irq_thread_check_affinity(desc, action);

        action_ret = handler_fn(desc, action);       // 调 thread_fn
        if (action_ret == IRQ_WAKE_THREAD)
            irq_wake_secondary(desc, action);

        wake_threads_waitq(desc);
    }
    return 0;
}
```

**irq_wait_for_interrupt：**

```c
// kernel/irq/manage.c:1056
static int irq_wait_for_interrupt(struct irqaction *action)
{
    for (;;) {
        set_current_state(TASK_INTERRUPTIBLE);

        if (kthread_should_stop()) {
            if (test_and_clear_bit(IRQTF_RUNTHREAD, &action->thread_flags)) {
                __set_current_state(TASK_RUNNING);
                return 0;
            }
            __set_current_state(TASK_RUNNING);
            return -1;          // kthread_stop 退出
        }

        if (test_and_clear_bit(IRQTF_RUNTHREAD, &action->thread_flags)) {
            __set_current_state(TASK_RUNNING);
            return 0;           // 有中断要处理
        }
        schedule();              // 睡眠等待
    }
}
```

线程化 IRQ 的执行路径：

```
中断触发
  → handle_irq_event
    → action->handler = NULL（驱动注册时设的）
      → irq_default_primary_handler()     ← kernel/irq/manage.c:1035
        return IRQ_WAKE_THREAD

handle_irq_event:
  ret = IRQ_WAKE_THREAD
  → __irq_wake_thread(desc, action)
    → set_bit(IRQTF_RUNTHREAD, &action->thread_flags)
    → wake_up_process(action->thread)       ← 唤醒 irq/xxx 线程
  → do { } while (ret == IRQ_WAKE_THREAD)  ← 循环（为什么？见 5.4）

irq_thread 被唤醒：
  → irq_wait_for_interrupt 返回 0
  → handler_fn(desc, action)
    → action->thread_fn(irq, dev_id)        ← 驱动 thread_fn 执行
```

### 5.4 IRQF_ONESHOT 设计原理

```c
// kernel/irq/manage.c:1086
static void irq_finalize_oneshot(struct irq_desc *desc,
                 struct irqaction *action)
{
    if (!(desc->istate & IRQS_ONESHOT))
        return;
    if (!irqd_irq_disabled(&desc->irq_data) && (desc->istate & IRQS_ONESHOT))
        unmask_irq(desc);     // ← thread_fn 完成后才 unmask！
}
```

不加 ONESHOT 的问题：

```
handle_fasteoi_irq                    ← 流控
  → mask_irq                          ← 关（EXTI IMR + GICD_ICENABLER）
  → handle_irq_event
    → handler 返回 IRQ_WAKE_THREAD
  → unmask_irq                        ← 开！电平设备信号线仍然有效
  → GIC 立即又收到相同中断
  → handle_fasteoi_irq 再次被调用       ← 无限循环
```

ONESHOT 保证：**mask 一直保持到 thread_fn 执行完成后才 unmask**。中间 GIC 和 EXTI 的 mask 位保持，防止电平触发的设备在顶半部返回后立即再次触发同一中断。

这就是为什么**电平触发的中断必须加 `IRQF_ONESHOT`**（01 §1.2.3 解释了 API 层面的要求，这里是从内核实现层面说明原因）。

### 5.5 与 workqueue 的对比

| 对比项 | 线程化 IRQ | Workqueue |
|--------|-----------|-----------|
| 线程性质 | 每中断专用线程 | 共享线程池 |
| 优先级 | SCHED_FIFO（实时） | 普通优先级 |
| 调度延迟 | 低 | 较高（需竞争线程池） |
| 适合场景 | 中断处理本身必须 sleep | 通用的异步推迟 |
| 资源开销 | 每个中断一个常驻线程 | 共享线程池按需增减 |
| 创建方式 | request_threaded_irq 自动创建 | schedule_work 挂入 system_wq |

选择标准：**如果中断处理需要 sleep（i2c_transfer、mutex_lock 等），且对延迟敏感 → 线程化 IRQ。如果只是通用推迟执行 → workqueue。**

---

## 6. irq_exit_rcu — 下半部调度总控

### 6.1 调度路径

四种下半部机制的调度入口在 `__irq_exit_rcu`：

```c
// kernel/softirq.c:633
static inline void __irq_exit_rcu(void)
{
    local_irq_disable();
    account_hardirq_exit(current);
    preempt_count_sub(HARDIRQ_OFFSET);
    if (!in_interrupt() && local_softirq_pending())
        invoke_softirq();               // ← 下半部调度入口
    tick_irq_exit();
}
```

**两个条件：**

| 条件 | 含义 | 为什么需要 |
|------|------|-----------|
| `!in_interrupt()` | 不在中断嵌套中（硬/软） | 嵌套中不触发——上层中断返回时会处理 |
| `local_softirq_pending()` | 有 pending | 没有 softirq 就不做 |

**invoke_softirq 的完整决策：**

```c
// kernel/softirq.c:425（非 PREEMPT_RT 版本）
static inline void invoke_softirq(void)
{
    if (!force_irqthreads() || !__this_cpu_read(ksoftirqd)) {
        __do_softirq();           // 默认路径：中断返回时直接执行
    } else {
        wakeup_softirqd();        // threadirqs 参数：强制线程化
    }
}
```

### 6.2 四种机制的调度时机对比

```
中断顶半部返回
  │
  ├─ softirq/tasklet 路径
  │    raise_softirq() → pending 置位
  │    → irq_exit_rcu() 检查 pending
  │      → invoke_softirq()
  │        → __do_softirq()           ← 中断返回路径上直接执行
  │         → softirq_vec[n].action()
  │           → tasklet_action()      ← TASKLET_SOFTIRQ
  │              → t->callback(t)
  │
  ├─ workqueue 路径
  │    mod_delayed_work(system_wq, work, delay)
  │    → __queue_work() + wake_up_worker()
  │    → worker_thread() 被调度时执行  ← 独立调度，与中断返回无关
  │      → process_one_work()
  │        → work->func()
  │
  └─ threaded IRQ 路径
       handler 返回 IRQ_WAKE_THREAD
       → __irq_wake_thread() + wake_up_process()
       → irq_thread() 被调度时执行     ← 独立调度，与中断返回无关
         → action->thread_fn()
```

| 机制 | 谁触发 | 何时执行 | 上下文 |
|------|--------|---------|-------|
| Softirq | `raise_softirq` → pending | `irq_exit_rcu` 中直接执行 | softirq context |
| Tasklet | `tasklet_schedule` → pending TASKLET_SOFTIRQ | 同上 | softirq context |
| Workqueue | `schedule_work` → wake_up_worker | worker 线程被调度时 | 进程 context |
| 线程化 IRQ | handler 返回 `IRQ_WAKE_THREAD` | irq 线程被调度时 | 进程 context |

**softirq/tasklet 紧贴中断返回路径执行**，workqueue/threaded IRQ 则与中断返回在时间上解耦。

---

## 7. STM32MP257 实际场景中的下半部选择

### 7.1 ATK 板上各设备的下半部使用

| 设备 | 下半部选择 | 原因 |
|------|-----------|------|
| gpio-keys（KEY0） | workqueue（`mod_delayed_work`）| 需要 debounce 延迟 |
| I2C 触摸屏 | 线程化 IRQ（`request_threaded_irq`）| 需要 `i2c_transfer`——必须 sleep |
| 网卡 | softirq（NAPI → `NET_RX_SOFTIRQ`）| 高性能要求 |
| SDMMC | softirq（`BLOCK_SOFTIRQ`）| 块设备完成处理 |
| 串口 | 线程化 IRQ | tty 层 flip buffer 用线程化处理 |

### 7.2 选择指南

```
中断推迟的任务需要 sleep？
  ├─ 是 → 需要专用线程（对实时性要求高）？
  │       ├─ 是 → 线程化 IRQ（SCHED_FIFO，专用，低延迟）
  │       └─ 否 → workqueue（共享线程池，够用）
  │
  └─ 否 → 顶半部执行时间？
           ├─ <10μs → 直接在顶半部完成
           ├─ 10~100μs → tasklet（串行，不 sleep，简单）
           └─ >100μs 或高性能 → softirq（NAPI 等自定义向量）
```

---

## 总结

| 机制 | 核心数据结构 | 执行上下文 | 能否 sleep | 并行 | 调度入口 |
|------|------------|-----------|-----------|------|---------|
| Softirq | softirq_vec[10] + per-CPU pending | softirq context | ❌ | 多 CPU 并行 | irq_exit_rcu → invoke_softirq |
| Tasklet | tasklet_struct + per-CPU 链表 | softirq context | ❌ | 同类型串行 | softirq TASKLET_SOFTIRQ |
| Workqueue | work_struct → pool_workqueue → worker_pool | 进程 context | ✅ | 多 worker 并行 | schedule_work → wake_up_worker |
| 线程化 IRQ | irqaction->thread（内核线程）| 进程 context | ✅ | 独享线程 | IRQ_WAKE_THREAD → wake_up_process |

04 篇分析了下半部的四种机制。下一篇 05-Scenario 将用 gpio-keys 按键中断和 I2C 触摸屏中断两个实际场景，把这些机制放入完整的中断处理路径中追踪。其中 gpio-keys 的 workqueue 下半部、I2C 触摸屏的线程化 IRQ 路径会引用本章的分析——届时不再重复机制细节，只关注"这个场景中下半部是怎么被触发的"。
