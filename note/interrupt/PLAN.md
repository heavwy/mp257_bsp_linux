# 中断子系统学习笔记 — 写作规划

> 本文档是 `note/interrupt/` 系列笔记的总体规划。
>
> 分析对象：STM32MP257D (Cortex-A35)，Linux v6.6.78 (stm32mp-r2)
> 中断控制器：GIC-400 (GICv2) + EXTI (st,stm32mp1-exti)
>
> 系列定位：**源码驱动，无需外部资料**。每一章的内容边界由源码文件划分决定。
>
> **总字数预计**：50,000–70,000 字
> **共 5 篇**：00-History → 01-Usage → 02-Architecture → 03-SourceAnalysis → 04-Scenario

---

## 核心决策：为什么没有独立的高级篇

中断子系统包含 PM、SMP 亲和性、虚假中断检测、RIF 安全等功能。但仔细分析源码可知：

- 这些功能的实现代码**分别挂载在** GIC 驱动、EXTI 驱动、IRQ 核心层的对应位置
- 不存在一个独立的、与其他模块解耦的"高级"代码段

因此将这部分内容**打散归入各正篇**：
- PM 功能 → EXTI 驱动分析（03-SourceAnalysis）及运行时路径（04-Scenario）
- SMP 亲和性 → GIC 驱动架构（02-Architecture）
- 虚假中断检测 → 核心层流程（04-Scenario）
- RIF 安全配置 → EXTI probe 流程（03-SourceAnalysis）
- Debug 技巧 → 使用篇（01-Usage）

**不纳入系列的内容**：
- `kernel/workqueue.c` 完整分析 — 通用内核机制，非中断子系统专属。只在 04-Scenario 中说明中断调用 workqueue 的接口边界
- `kernel/softirq.c` 完整分析 — 同上，限定在 `irq_exit_rcu()` → `invoke_softirq()` 的触发路径

---

## 源码文件总索引

以下是整个系列会涉及的全部源码文件及其在各篇中的分布：

| 文件 | 行数 | 覆盖篇目 | 核心函数/结构体 |
|------|------|---------|----------------|
| `arch/arm64/kernel/entry.S` | 1099 | 03, 04 | vectors、kernel_ventry、el1h_64_irq |
| `arch/arm64/kernel/irq.c` | — | 03, 04 | init_IRQ、el1h_64_irq_handler、handle_arch_irq |
| `init/main.c` | — | 03 | start_kernel → init_IRQ |
| `drivers/irqchip/irqchip.c` | — | 03 | irqchip_init |
| `drivers/of/irq.c` | — | 03 | of_irq_init |
| `drivers/irqchip/irq-gic.c` | 1754 | 02, 03, 04 | gic_chip_data、gic_of_init、gic_handle_irq |
| `drivers/irqchip/irq-stm32mp-exti.c` | 993 | 02, 03, 04 | stm32mp_exti_host_data、probe、domain_alloc |
| `drivers/pinctrl/stm32/pinctrl-stm32mp.c` | 见 pinctrl 系列 | 03 | stm32_gpio_to_irq |
| `kernel/irq/irqdesc.c` | — | 02, 04 | irq_desc、generic_handle_domain_irq |
| `kernel/irq/manage.c` | 2941 | 01, 04 | request_threaded_irq、setup_irq、irq_thread |
| `kernel/irq/chip.c` | 1619 | 02, 04 | handle_level_irq、handle_fasteoi_irq、handle_edge_irq |
| `kernel/irq/handle.c` | — | 02, 04 | handle_arch_irq、handle_irq_event |
| `kernel/irq/irqdomain.c` | 1975 | 02, 03 | irq_domain 层次管理、alloc/find/map |
| `kernel/irq/devres.c` | 284 | 01 | devm_request_irq |
| `kernel/irq/proc.c` | — | 01 | /proc/interrupts、/proc/irq/ |
| `kernel/irq/debugfs.c` | — | 01 | /sys/kernel/debug/irq/ |
| `kernel/irq/pm.c` | — | 03 | irq_pm_* |
| `kernel/irq/affinity.c` | — | 02 | irq_set_affinity |
| `kernel/irq/spurious.c` | — | 04 | note_interrupt、poll_spurious |
| `kernel/softirq.c` | 1010 | 02(简述), 04(路径) | irq_exit_rcu、do_softirq、tasklet_action |
| `kernel/workqueue.c` | 6858 | 04(接口边界) | queue_work、schedule_work |
| `include/linux/interrupt.h` | — | 01, 02 | irqaction、irq_chip、irq_handler_t |
| `include/linux/irqdesc.h` | — | 02 | irq_desc |
| `include/linux/irqdomain.h` | — | 02 | irq_domain、irq_domain_ops |
| `arch/arm64/boot/dts/st/stm32mp251.dtsi` | ~3000 | 01 | intc、exti1、v2m 节点 |

---

## 第 0 篇：00-History.md — 中断子系统演进史

**核心问题**：Linux 中断子系统为什么设计成今天这样？每个关键机制是在解决什么历史问题？

**源码线索**：`include/linux/irqdomain.h` 中 `enum irq_domain_bus_token` 的版本演化、`kernel/irq/` 目录的文件创建顺序、`include/linux/interrupt.h` 中 `irq_chip` 结构体的字段增长

### 大纲

| 章节 | 时间 | 核心变化 | 涉及源码 |
|------|------|---------|---------|
| 1.1 前传：无统一框架时代 | v1.0–v2.4 | `do_IRQ()` 架构私有，`arch/*/kernel/irq.c` 各自实现 | 无通用 `kernel/irq/` |
| 1.2 统一中断框架诞生 | v2.6.8–v2.6.9 | `kernel/irq/` 目录创建，`request_irq()`/`free_irq()` 标准化 | `manage.c`、`irqdesc.c` |
| 1.3 irq_chip 分离 | v2.6.18–v2.6.24 | `kernel/irq/chip.c` 引入，`handle_level_irq`/`handle_edge_irq` 流控 | `chip.c`、`handle.c` |
| 1.4 irq_domain 的诞生 | v2.6.38 | 解决设备树中断映射，PowerPC MPIC 首次使用 | `irqdomain.c` |
| 1.5 层级 irq_domain | v3.10+ | 解决中断控制器级联，`irq_domain_create_hierarchy()` | `irqdomain.c` `struct irq_domain` 的 `parent` 字段 |
| 1.6 ARM64 中断入口 | v3.11+ | `entry.S` 引入 `VBAR_EL1` 机制、`handle_arch_irq` 函数指针 | `entry.S`、`irq.c`(arm64) |
| 1.7 下半部演进 | v2.6–v3.2 | softirq → tasklet → CMWQ (v3.2) → 线程化 IRQ | `softirq.c`、`workqueue.c` |

**写作要点**：
- 每个阶段用一个实际 commit 或版本号锚定
- 展示旧代码 vs 新代码的对比
- 不堆砌细节，重点回答"为什么"

---

## 第 1 篇：01-Usage.md — 中断使用方法和 DTS 配置

**核心问题**：驱动开发者在 STM32MP257 上怎么使用中断？有哪些入口、怎么调试？

### 源码依据

| 功能 | 源码文件 | 核心函数/接口 |
|------|---------|--------------|
| 标准请求 | `kernel/irq/manage.c` | `request_threaded_irq()` |
| 自动管理 | `kernel/irq/devres.c` | `devm_request_irq()` |
| 中断控制 | `kernel/irq/manage.c` | `disable_irq/enable_irq/synchronize_irq` |
| 获取中断号 | `driver/base/platform.c` | `platform_get_irq()` |
| 线程化 IRQ | `kernel/irq/manage.c` | `request_threaded_irq(handler, thread_fn)` |
| GPIO to IRQ | `gpiolib.c` + pinctrl-stm32mp | `gpiod_to_irq()` |
| 下半部(Tasklet) | `kernel/softirq.c` | `tasklet_init/schedule/kill` |
| 下半部(Work) | `kernel/workqueue.c` | `INIT_WORK`/`schedule_work`/`queue_work` |
| /proc 接口 | `kernel/irq/proc.c` | `/proc/interrupts` 生成逻辑 |
| DebugFS | `kernel/irq/debugfs.c` | `/sys/kernel/debug/irq/irqs/N` 字段含义 |
| DTS 定点 | `stm32mp251.dtsi` | intc 节点(三 cell)、exti1 节点 |

### 大纲

| 章节 | 内容 | 关键点 |
|------|------|--------|
| 1.1 用户态接口 | `/proc/interrupts` 各列含义、`/proc/irq/N/` 目录结构 | `proc.c` 如何统计和格式化 |
| 1.2 DebugFS 调试 | `/sys/kernel/debug/irq/domains/`、`/irqs/70` 全部字段解读 | 三层 domain 拓扑（实测数据） |
| 1.3 内核态 API | | |
| 1.3.1 标准中断请求 | `request_threaded_irq()` — handler(顶半部) + thread_fn(底半部) | `manage.c` L2145 参数详解 |
| 1.3.2 devm 自动管理 | `devm_request_irq()` — 设备移除自动释放 | `devres.c` 实现原理 |
| 1.3.3 获取中断号 | `platform_get_irq()` 从 DTS 获取 virq | 与 `of_irq_parse_and_map` 的关系 |
| 1.3.4 中断控制 | `enable_irq/disable_irq/synchronize_irq` | `depth` 计数器的含义，同步 vs 异步 |
| 1.3.5 GPIO 中断路径 | `gpiod_get` → `gpiod_to_irq` → `devm_request_irq` | pinctrl 的 `stm32_gpio_to_irq` 实现 |
| 1.4 下半部 API | | |
| 1.4.1 Tasklet | `DECLARE_TASKLET`/`tasklet_schedule`/`tasklet_kill` | 注册在 `TASKLET_SOFTIRQ` 向量上 |
| 1.4.2 Workqueue | `INIT_WORK` + `schedule_work()` / `alloc_workqueue()` + `queue_work()` | system_wq 等全局队列的适用场景 |
| 1.4.3 线程化 IRQ | `request_threaded_irq(handler, thread_fn)` — 顶半部返回 `IRQ_WAKE_THREAD` | `irq_thread()` 的实现 |
| 1.5 DTS 配置 | | |
| 1.5.1 GIC 三 cell | `<GIC_SPI 120 IRQ_TYPE_LEVEL_HIGH>` — type/编号/trigger | `stm32mp251.dtsi` 中 intc 节点 |
| 1.5.2 EXTI 路由 | `interrupts-extended = <&exti1 47 IRQ_TYPE_LEVEL_HIGH>` | EXTI event 号到 GIC SPI 的映射 |
| 1.5.3 GPIO 中断 | `interrupt-parent = <&gpioh>` / `interrupts = <5 IRQ_TYPE_EDGE_BOTH>` | PH5 的实际 DTS 配置 |
| 1.6 实战调试技巧 | devmem 查看 EXTI/GIC 寄存器、trace-cmd、dynamic_debug | 三板斧解决中断问题 |

---

## 第 2 篇：02-Architecture.md — 核心数据结构与设计模式

**核心问题**：中断子系统的骨架是什么？irq_desc、irq_domain、irq_chip、irqaction 如何配合工作？

### 数据结构关系总览

```
irq_desc[virq]
  ├── handle_irq        ← 流控函数（由 irq_domain 设置）
  ├── action            ← irqaction 链表（由 request_irq 注册）
  │     ├── handler     ← 驱动顶半部
  │     ├── thread_fn   ← 驱动底半部线程
  │     └── thread      ← 内核线程（线程化 IRQ）
  ├── irq_data          ← irq_data 链表（层级关键）
  │     ├── irq_chip    ← mask/unmask/ack/eoi 操作
  │     ├── irq_domain  ← 该层所属的 domain
  │     └── parent_data ← 父级 irq_data（链表节点）
  └── depth/wake_depth  ← 嵌套深度控制

irq_domain
  ├── ops: {alloc, free, xlate, match, select}
  ├── host_data          ← 驱动私有数据（EXTI: host_data → chip_data）
  └── parent             ← 父 domain（NULL 表示根，即 GIC）

GIC 侧:
  gic_chip_data
    ├── domain       ← GIC irq_domain
    ├── chip         ← gic_chip (mask/unmask/eoi)
    ├── dist_base    ← Distributor 寄存器基址
    └── cpu_base     ← CPU Interface 寄存器基址

EXTI 侧:
  stm32mp_exti_host_data
    ├── drv_data     ← exti_banks[3] + desc_irqs[96]
    ├── chips_data[] ← per-bank chip_data（内含 mask_cache）
    └── base         ← EXTI MMIO 基址
```

### 大纲

| 章节 | 源码结构体/文件 | 内容 |
|------|----------------|------|
| 2.1 四层中断处理模型 | — | 硬件层 → GIC 层 → 核心层 → 驱动层 |
| 2.2 irq_desc | `include/linux/irqdesc.h` | 55 行完整结构体分析、每个字段含义 |
| 2.3 irqaction | `include/linux/interrupt.h` L118 | handler/thread_fn/thread/thread_flags/IRQF_* |
| 2.4 irq_data 与 irq_chip | `include/linux/irq.h` | `irq_data` 链表实现层级传递、`irq_chip` 回调表 |
| 2.5 irq_domain | `include/linux/irqdomain.h` L150 | ops/host_data/parent、三种映射方式 |
| 2.6 GIC 数据结构 | `irq-gic.c: struct gic_chip_data` L70 | 全局 `gic_data[]` 数组、dist/cpu 基址、domain/chip |
| 2.7 EXTI 数据结构 | `irq-stm32mp-exti.c` 四个结构体 L52-92 | host_data → chip_data → exti_bank 三层容器 |
| 2.8 mask_cache 优化 | `stm32mp_exti_mask/unmask` | 写缓存减少 MMIO 读-改-写延迟 |
| 2.9 两个 irq_chip 的设计 | `stm32mp_exti_chip` vs `_direct` L520/L534 | TRG 寄存器位决定 EOI 行为 |
| 2.10 流控函数 | `kernel/irq/chip.c` | `handle_level_irq` L628 / `handle_fasteoi_irq` L687 / `handle_edge_irq` L787 |
| 2.11 链式 vs 层级 | `kernel/irq/chip.c` + `irqdomain.c` | `handle_nested_irq` vs `irq_chip_*_parent` |
| 2.12 软中断向量表 | `kernel/softirq.c` | 10 个 `softirq` 向量（HI→TIMER→NET_TX→NET_RX→BLOCK→IRQ_POLL→TASKLET→SCHED→HRTIMER→RCU） |
| 2.13 SMP 中断分发 | `kernel/irq/affinity.c` + `irq-gic.c` | GICD_ITARGETSR 寄存器、`/proc/irq/N/smp_affinity` |

---

## 第 3 篇：03-SourceAnalysis.md — 初始化流程源码分析

**核心问题**：系统启动后，中断子系统是怎么从零到全部就绪的？

### 调用链全景

```
start_kernel
  ├── setup_arch()
  │     └── 解析 DTB、初始化页表
  ├── init_IRQ()                         ← arch/arm64/kernel/irq.c
  │     ├── init_irq_stacks()            ← per-CPU 中断栈分配
  │     └── irqchip_init()               ← drivers/irqchip/irqchip.c
  │           └── of_irq_init(__irqchip_of_table)  ← drivers/of/irq.c
  │                 ├── GIC: gic_of_init()          ← irq-gic.c (interrupt-controller@4ac00000)
  │                 └── EXTI: stm32mp_exti_probe()  ← irq-stm32mp-exti.c (platform_driver)
  ├── time_init()                        ← 初始化时钟中断
  ├── softirq_init()                     ← kernel/softirq.c（注册 TASKLET/HI 软中断）
  └── workqueue_init()                   ← kernel/workqueue.c（创建 worker 线程）
        └── workqueue_init_early()       ← 创建 system_wq 等全局队列
```

### 大纲

| 阶段 | 源码函数 | 行号 | 关键作用 |
|------|---------|------|---------|
| **3.1 异常向量表安装** | | | |
| 3.1.1 | `head.S: __primary_switched` → `msr vbar_el1, x8` | — | 将 `entry.S` 的 `vectors` 写入 VBAR_EL1 |
| 3.1.2 | `entry.S: vectors` 16 条目, `kernel_ventry` 宏 | — | 128B 对齐、栈溢出检测机制、`el1h_64_irq` 的定义 |
| **3.2 IRQ 核心层初始化** | | | |
| 3.2.1 | `init_IRQ()` | — | 分配中断栈、调用 irqchip_init |
| 3.2.2 | `irqchip_init()` → `of_irq_init()` | — | 扫描 `__irqchip_of_table` 段，按 parent 依赖排序 |
| 3.2.3 | `of_irq_init()` DFS 初始化策略 | `drivers/of/irq.c` | parent=NULL→GIC→EXTI 的层级顺序 |
| **3.3 GIC 初始化** | | | |
| 3.3.1 | `IRQCHIP_DECLARE(gic_400, ...)` | `irq-gic.c` | 编译时放入 `__irqchip_of_table` 段 |
| 3.3.2 | `gic_of_init()` | L1174 | `gic_of_setup()` + `__gic_init_bases()` |
| 3.3.3 | `gic_of_setup()` | — | ioremap Distributor/CPU Interface/Virt 寄存器 |
| 3.3.4 | `__gic_init_bases()` | L1249 | `set_handle_irq(gic_handle_irq)` + `gic_dist_init()` + `gic_cpu_init()` |
| 3.3.5 | `gic_dist_init()` | L475 | 初始化 GICD_CTLR、配置 SPI 优先级/目标 CPU |
| 3.3.6 | `gic_cpu_init()` | L498 | 配置 GICC_PMR(优先级掩码)、GICC_CTLR(使能)、写 CPU 接口 |
| **3.4 EXTI 初始化** | | | |
| 3.4.1 | `stm32mp_exti_probe()` 完整流程 | L836 | 11 步顺序详解 |
| 3.4.2 | host_data 分配 + hwspinlock | L840-850 | 多客隔离的可选锁 |
| 3.4.3 | ioremap + drv_data 获取 | L852-860 | `stm32mp1_drv_data` → 3 banks + desc_irqs[96] |
| 3.4.4 | `stm32mp_exti_chip_init()` per bank | — | 清 IMR、保存 SeCCFGR(安全事件掩码) |
| 3.4.5 | `stm32mp_exti_check_rif()` | L804 | CIDCFGR 寄存器遍历，非 CID1 事件标记 reserved |
| 3.4.6 | 查找 GIC parent domain | — | `irq_find_host(of_irq_find_parent(np))` |
| 3.4.7 | `irq_domain_add_hierarchy()` 注册层级 domain | — | 将 EXTI domain 挂载为 GIC domain 的子域 |
| 3.4.8 | 可选 wakeup domain + PM 使能 | — | 唤醒中断特殊路由 |
| **3.5 Pinctrl/GPIO 中断准备** | | | |
| 3.5.1 | pinctrl probe → irq_domain 注册 | 链接 pinctrl 系列 | `stm32_pctrl_get_irq_domain()` |
| 3.5.2 | `gpiochip_add_data()` → irq_chip 注册 | 链接 pinctrl 系列 | `stm32_gpio_to_irq()` 映射 GPIO pin → EXTI event |
| **3.6 软中断初始化** | | | |
| 3.6.1 | `softirq_init()` | `softirq.c` L904 | 注册 `TASKLET_SOFTIRQ`/`HI_SOFTIRQ` |
| 3.6.2 | ksoftirqd 内核线程 | `softirq.c` L919 | per-CPU 守护进程上下文 |

---

## 第 4 篇：04-Scenario.md — 运行时情景分析

**核心问题**：GPIO 按键按下后，从电平变化到马里奥跳跃，代码路径是什么样的？

### 完整调用链

```
硬件触发
  GPIO PH5 电平变化
    → EXTI 检测边沿（RPR/FPR 置位）
    → EXTI IMR 允许（mask_cache）
    → EXTI 向 GIC 发送 SPI 中断信号

GIC 分发
    → GIC Distributor (GICD_ISENABLER 使能)
    → CPU Interface (GICC_IAR 可读)
    → 向 CPU 核断言 nIRQ 信号

ARM64 异常入口（entry.S）
    → CPU 查 VBAR_EL1 → vectors 表
    → kernel_ventry 1, h, 64, irq（EL1h 模式）
    → kernel_entry 保存 pt_regs
    → el1h_64_irq_handler()
      → __el1_irq()
        → irq_enter_rcu()
        → do_interrupt_handler(regs, handle_arch_irq)

GIC 中断处理主循环（irq-gic.c）
    → gic_handle_irq():
      do {
        irqstat = readl(GICC_IAR)       ← 获取中断 ID (hwirq)
        if (irqnr >= 1020) break        ← spurious ID
        if (split EOI) writel(GICC_EOI)
        generic_handle_domain_irq(gic->domain, irqnr)
                                        ← hwirq → virq 映射
      } while (1)

IRQ 核心层分发（kernel/irq/irqdesc.c + chip.c）
    → generic_handle_domain_irq(domain, hwirq)
    → irq_find_mapping(domain, hwirq) → virq
    → generic_handle_irq_desc(virq)
      → desc->handle_irq() → handle_fasteoi_irq()

流控处理（chip.c）
    → handle_fasteoi_irq(desc):
      mask_irq(desc)                     ← stm32mp_exti_mask()
      handle_irq_event(desc)
      unmask_irq(desc)                   ← stm32mp_exti_unmask()
      eoi_irq(desc)                      ← stm32mp_exti_eoi()
        → 写 RPR/FPR（清 EXTI 挂起）
        → irq_chip_eoi_parent() → GIC EOI

handle_irq_event（handle.c）
    → handle_irq_event_percpu(desc)
    → __handle_irq_event_percpu(desc):
      do {
        ret = action->handler(irq, dev_id) ← 驱动顶半部
        if (ret == IRQ_WAKE_THREAD)
          __irq_wake_thread(desc, action) ← 唤醒线程
      } while (action)

gpio-keys 驱动层
    → gpio_keys_irq_handler()
      → gpiod_get_value() 读 GPIO 电平
      → input_event() 上报按键事件
      → input_sync()

中断退出 + 下半部触发
    → irq_exit_rcu()
    → __irq_exit_rcu():
      if (!in_interrupt() && local_softirq_pending())
        invoke_softirq() 或 wakeup_ksoftirqd()

下半部执行
    → do_softirq():
      pending = __softirq_pending(cpu)
      for each set bit:
        softirq_vec[nr].action()
    → tasklet_action()  ← TASKLET_SOFTIRQ
    → 或 worker_thread() ← 如果驱动用了 workqueue

用户态可见
    → evtest /dev/input/event0 读取 input 事件
    → retroarch 响应跳跃
```

### 大纲

| 阶段 | 源码函数 | 文件:行号 |
|------|---------|-----------|
| **4.1 场景设定** | DTS + 原理图 + 板级数据 | `stm32mp257d-atk.dts` |
| **4.2 硬件触发** | EXTI RPR/FPR 置位 | `irq-stm32mp-exti.c` |
| **4.3 GIC 分发** | Distributor → CPU Interface | `irq-gic.c` GICD 寄存器 |
| **4.4 ARM64 异常入口** | vectors → kernel_ventry → el1h_64_irq | `entry.S` |
| **4.5 gic_handle_irq 主循环** | GICC_IAR → generic_handle_domain_irq | `irq-gic.c` L345 |
| **4.6 流控：handle_fasteoi_irq** | mask/event/eoi 三步 | `chip.c` L687 |
| **4.7 handle_irq_event** | 遍历 action 链表 | `handle.c` L139 |
| **4.8 驱动顶半部** | gpio-keys → input_event | `gpio-keys.c` |
| **4.9 中断退出与下半部触发** | irq_exit_rcu → invoke_softirq | `softirq.c` L600-665 |
| **4.10 下半部执行** | do_softirq → tasklet_action | `softirq.c` L517 |
| **4.11 线程化 IRQ 路径** | 返回 IRQ_WAKE_THREAD → irq_thread | `manage.c` L1298 |
| **4.12 Workqueue 路径** | schedule_work → worker_thread | `workqueue.c` |
| **4.13 虚假中断检测** | note_interrupt → spurious 计数 | `spurious.c` |
| **4.14 中断绑定与迁移** | gic_set_affinity → GICD_ITARGETSR | `irq-gic.c` `affinity.c` |

### 四个典型路径对比

| 场景 | 流控函数 | 下半部 | 文件 |
|------|---------|--------|------|
| GPIO 按键 | `handle_fasteoi_irq` | softirq → tasklet → input 事件 | `chip.c` + `softirq.c` |
| 网卡 RX | `handle_fasteoi_irq` | `NET_RX_SOFTIRQ` (NAPI) | `chip.c` + `softirq.c` |
| 慢速设备(I2C) | `handle_level_irq` | 线程化 IRQ | `chip.c` + `manage.c` |
| 块设备 | `handle_edge_irq` | `BLOCK_SOFTIRQ` | `chip.c` + `softirq.c` |

---

## 章节内容分配矩阵

```
                  00-History  01-Usage  02-Arch  03-Source  04-Scenario
                    演进史     API/调试   数据结构   初始化     运行时路径
                                                             
异常向量表历史       ✅                                      
irq_desc 演进        ✅                                      
irq_domain 诞生      ✅                                      
层级 domain 演进     ✅                                      
                                                                                
request_irq API                 ✅                              
platform_get_irq                ✅                              
devm_request_irq                ✅                              
gpiod_to_irq                    ✅                              
/proc/interrupts                ✅                              
debugfs irqs/N                  ✅                              
DTS intc/exti 格式              ✅                              
Tasklet API                     ✅                              
Workqueue API                   ✅                              
线程化 IRQ API                  ✅                              
                                                                                
irq_desc 结构体                        ✅                       
irq_data + irq_chip                     ✅                       
irqaction                                ✅                       
irq_domain                               ✅                       
GIC 数据结构                             ✅                       
EXTI 数据结构                            ✅                       
mask_cache 优化                          ✅                       
两个 irq_chip 设计                        ✅                       
流控函数(level/edge/fasteoi)              ✅                       
链式 vs 层级                              ✅                       
SMP 亲和性                               ✅                       
软中断向量                                ✅                       
                                                                                
head.S VBAR_EL1                                   ✅                
init_IRQ → irqchip_init                            ✅                
of_irq_init DFS 策略                               ✅                
gic_of_init → __gic_init_bases                      ✅                
gic_dist_init + gic_cpu_init                        ✅                
stm32mp_exti_probe 11步                             ✅                
EXTI RIF 安全检测                                   ✅                
irq_domain_add_hierarchy                            ✅                
GPIO 中断准备(pinctrl)                              ✅                
softirq_init                                         ✅                
workqueue_init                                       ✅                
EXTI PM(suspend/resume)                              ✅                
                                                                                
完整中断路径：硬件→CPU                                        ✅                       
gic_handle_irq 主循环                                       ✅                       
handle_fasteoi_irq 流控                                      ✅                       
handle_irq_event action 遍历                                 ✅                       
irq_exit → do_softirq                                        ✅                       
tasklet_action                                                ✅                       
线程化 IRQ 路径                                               ✅                       
workqueue 路径                                                ✅                       
虚假中断检测                                                   ✅                       
中断绑定迁移                                                   ✅                       
```

---

## 写作原则

1. **源码唯一**：每个结论必须对应 `.c`/`.h` 文件的函数或行号，参考资料仅作理解参考不直接引用
2. **不堆代码**：代码块不超过全文 30%，关键路径用流程图/表格辅助
3. **场景驱动**：04 篇用真实的马里奥按键场景串联，其他篇的示例贴近 STM32MP257 实际硬件
4. **分层写作**：先写 00 热身，然后 01/02 并行，最后 03/04 需要完整的源码阅读

---

*规划完成日期：2026-06-08*
*预计开始写作：00-History.md 第一版*
