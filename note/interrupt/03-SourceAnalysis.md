# 03. 初始化流程源码分析

> 本文是 STM32MP257 中断子系统深度分析系列的第 3 篇。
> 从源码层面逐行追踪中断子系统的初始化路径。
>
> **前置:** [02-Architecture.md](02-Architecture.md) — 熟悉核心结构体和数据结构关系
> **下一篇:** ——
>
> **字数：约 28,000 词（含代码段）**
> **建议阅读时间：60~80 分钟**

---

## 3.1 从一个问题开始

系统启动后，你在终端敲：

```shell
$ cat /proc/interrupts
           CPU0
 16:        131      GIC-400  29  Level     arch_timer
 29:          0      GIC-400 273  Level     stm32mp_exti
 30:          0      GIC-400 274  Level     stm32mp_exti
...
268:          0  stm32mp_exti   5  Edge      gpio-keys
```

问题是：**从系统上电到 `/proc/interrupts` 中出现完整的三层中断控制器（GIC → EXTI → GPIO），内核里经历了什么？**

以 ATK 板按键 PH5 为例，从中断信号的全路径来看，初始化必须按依赖顺序完成四层基础设施：

| 层次 | 组件 | 初始化时机 | 依赖 |
|------|------|-----------|------|
| ① | CPU 异常向量表 | `head.S`，`start_kernel()` 之前 | 无 |
| ② | GIC（根中断控制器） | `start_kernel()` → `init_IRQ()` | 无 |
| ③ | EXTI（第一级子控制器） | `do_initcalls()` → `module_init` | 需要 GIC irq_domain |
| ④ | GPIO irq_domain（第二级子域） | pinctrl probe 内部 | 需要 EXTI irq_domain |

**本文沿着初始化时间线，逐层追踪这些环节的代码路径。**

---

## 3.2 初始化流程全景

```
时间 →
       │
       │  [编译期] — 注册 GIC 初始化函数
       │     目的：GIC 的初始化函数 gic_of_init 必须能在 start_kernel 中被找到。
       │           IRQCHIP_DECLARE 宏把它注册到 __irqchip_of_table 特殊 ELF 段，
       │           链接后形成连续数组。这样无需设备模型，汇编级就能遍历调用。
       │
       ▼
       │  [1] head.S → 安装异常向量表（汇编）
       │     目的：CPU 收到 IRQ 信号时必须知道跳到哪里处理。
       │           VBAR_EL1 指向 vectors 表，中断到来时 CPU 硬件自动跳转。
       │           此时根 handler 还是占位函数——会 panic。
       │      └─ msr vbar_el1, x0
       │
       ▼
       │  [2] start_kernel() → 初始化根中断控制器 GIC
       │     目的：建立整个中断系统的根。GIC 是中断的"总机"——所有中断信号
       │           都经过它路由到 CPU。必须在其他子控制器之前初始化。
       │      └─ init_IRQ() → irqchip_init() → of_irq_init(__irqchip_of_table)
       │           │
       │           ├─ GIC 是根节点（无 parent），在第一轮被初始化
       │           │   → gic_of_init()
       │           │     ├─ set_handle_irq()    → 替换 panic 占位函数
       │           │     ├─ gic_dist_init()     → 配置 Distributor（路由/优先级）
       │           │     ├─ gic_cpu_init()      → 配置 CPU Interface（门槛/使能）
       │           │     ├─ irq_domain_create_linear() → 创建 hwirq→virq 映射表
       │           │     └─ gic_smp_init()      → 核间中断 IPI
       │           │
       │           └─ EXTI 不在这里（它在 __irqchip_of_table 之外）
       ▼
       │  [3] do_initcalls() → 初始化 EXTI（第一级子控制器）
       │     目的：GPIO 中断信号必须先经过 EXTI 的边沿检测才能到达 GIC。
       │           EXTI 负责触发方式选择（上升/下降沿）、中断屏蔽、挂起标志。
       │           它走标准 platform 驱动路径，在内存就绪后初始化。
       │      └─ module_platform_driver(stm32mp_exti_driver)
       │           → stm32mp_exti_probe()
       │             ├─ 为每个 bank 分配 chip_data + ioremap
       │             ├─ 初始化硬件寄存器（EXTI_CR 选通 GPIO 输入）
       │             ├─ irq_domain_add_hierarchy(parent=GIC_domain, 64)
       │             │   → 创建层级域：EXTI 是 GIC 的子域
       │             └─ 设置两种 irq_chip（EXTI 事件 / 直通事件）
       │                 → mask/unmask/set_type/eoi  操作 EXTI 自身寄存器
       │                 → ack/set_affinity          委托给父域 GIC
       ▼
       │  [4] pinctrl probe → 创建 GPIO irq_domain（第二级子域）
       │     目的：每个 GPIO pin 可以产生中断，但 GPIO 本身没有中断寄存器。
       │           需要创建一个层次化 irq_domain，把 GPIO pin 号映射到 EXTI line，
       │           并通过 parent 指针将 mask/unmask 操作委托给 EXTI。
       │      └─ stm32_pctl_probe() → stm32_pctrl_get_irq_domain()
       │           └─ irq_domain_create_hierarchy(parent=exti_domain, 16)
       ▼
       │  [5] 消费者设备 probe → 注册中断 handler
       │     目的：具体设备（如按键）在 probe 时通过 request_irq 将业务 handler
       │           注册到系统中。这个过程中，中断号通过三层 domain 分配，
       │           irqaction 挂入 irq_desc，三级 irq_chip 使能中断。
       │      └─ driver_probe_device()
       │           ├─ irq_create_of_mapping() → 穿越三层 domain 分配 virq
       │           └─ request_irq(virq, handler) → irqaction 挂入 irq_desc
       ▼
       全部就绪，等待中断
```

### 四个代码模块的角色

| 代码 | 文件位置 | 作用 | 初始化方式 |
|------|---------|------|-----------|
| ARM64 entry | `arch/arm64/kernel/entry.S` | 异常向量表、中断入口（汇编） | 上电时 head.S 直接设置 VBAR_EL1 |
| GIC 驱动 | `drivers/irqchip/irq-gic.c` | 根中断控制器初始化 + irq_domain | `IRQCHIP_DECLARE` → `of_irq_init()` |
| STM32 EXTI 驱动 | `drivers/irqchip/irq-stm32mp-exti.c` | 第一级子域（边沿检测/屏蔽/转发） | `module_platform_driver` → `do_initcalls()` |
| STM32 Pinctrl 驱动 | `drivers/pinctrl/stm32/pinctrl-stm32.c` | GPIO irq_domain（第二级子域） | `module_platform_driver` → pinctrl probe 内部 |

全景图把整个流程分成了 5 个阶段，每个阶段解决一个问题。**下面按照时间线，从 GIC 开始逐层追踪。**

---

## 3.3 阶段一：GIC 初始化（根中断控制器）

全景图的时间起点是 `start_kernel()` → `init_IRQ()` → `irqchip_init()`。但在此之前，CPU 已经完成了一件中断子系统能工作的先决条件：**安装异常向量表**（全景图 [1] 对应的工作）。

> **前置条件：启动汇编中安装的异常向量表**
>
> `head.S` 在进入 `start_kernel()` 之前就设置了 `VBAR_EL1`，指向 `vectors` 表：
> ```asm
> adr_l x8, vectors
> msr vbar_el1, x8
> ```
> 这个表定义了 CPU 收到 IRQ 信号时跳转到哪个入口。中断到来时，CPU 自动跳到 `vectors + 0x280`，经过 `kernel_ventry` 宏保存现场，最终进入 C 函数 `el1h_64_irq_handler()`，它调用了全局函数指针 `handle_arch_irq`。此时 `handle_arch_irq` 指向 `default_handle_irq`——一个直接 panic 的占位函数。**GIC 初始化要做的第一件事就是替换它。**

回到全景图 [2]。`start_kernel()` 执行到 `init_IRQ()` → `irqchip_init()` → `of_irq_init(__irqchip_of_table)`，这是 GIC 初始化的入口。GIC 不走标准 platform 驱动路径，而是通过 `IRQCHIP_DECLARE` 宏提前注册，在 `mm_init()` 之前就完成初始化。之所以这么早，是因为 GIC 是中断系统的根——所有其他中断控制器都依赖它。

### 3.3.0 GIC 初始化施工图

先总览 `gic_of_init()` 做了什么——它把 02 章中定义的关键数据结构从"抽象概念"变成"运行时实例"。

`gic_of_init()` 的输入只有两个参数：DTS 中 `intc` 节点的 `device_node` 指针，以及 `parent` 指针（GIC 是根节点所以为 `NULL`）。函数返回时，GIC 的硬件寄存器已配置、irq_domain 已创建、`handle_arch_irq` 已从 panic 占位替换为真正的 `gic_handle_irq`。

```
gic_of_init(node, parent=NULL)
   │
   │  [准备工作：寄存器映射]
   │
   ├── ① gic_of_setup()
   │     → gic_data[0].raw_dist_base = ioremap(0x4ac10000, 0x1000)
   │     → gic_data[0].raw_cpu_base  = ioremap(0x4ac20000, 0x2000)
   │      02-§2: 让 GIC 的 irq_chip 回调（mask/unmask/eoi）能访问硬件寄存器
   │      此时硬件寄存器还是复位值——GIC 处于"全关"状态
   │
   ├── ② __gic_init_bases()
   │    │
   │    ├── set_handle_irq(gic_handle_irq)
   │    │   → handle_arch_irq 从 default_handle_irq（会 panic）替换为 gic_handle_irq
   │    │   → 从此 CPU 的中断有了真正的入口
   │    │   02-§2.5: 安装根中断控制器的分发函数
   │    │
   │    └── gic_init_bases()
   │         │
   │         ├── 读 GIC_DIST_CTR → gic_irqs = 1020
   │         │     从硬件寄存器读出 GIC 支持 32 组中断 × 32 = 1024 → 截断到 1020
   │         │     这决定了 irq_domain 的大小
   │         │
   │         ├── irq_domain_create_linear(handle, 1020, &ops, gic_data)
   │         │   → gic_data[0].domain
   │         │   → 分配 irq_domain 结构体 + revmap[1020] 柔性数组
   │         │   → revmap[0..1019] = {全 NULL}
   │         │   02-§3.2: 创建线性映射域——hwirq → virq 的翻译表
   │         │   ★ 此时是"空网"——没有任何中断被映射
   │         │
   │         ├── gic_dist_init()
   │         │   → GIC_DIST_CTRL = 0      临时关闭
   │         │   → GIC_DIST_TARGET = 0x01010101...  所有 SPI 路由到 CPU0
   │         │   → GIC_DIST_PRI = 0xa0a0a0a0...     优先级 0xa0
   │         │   → GIC_DIST_CONFIG = 0               默认电平触发
   │         │   → GIC_DIST_ENABLE = 0               全部禁用
   │         │   → GIC_DIST_CTRL = 1      重新使能
   │         │   02-§1.3: Distributor 硬件初始化——路由/优先级/触发类型
   │         │
   │         ├── gic_cpu_init()
   │         │   → GICC_PMR  = 0xf0    优先级阈值
   │         │   → GICC_CTLR = 0x1     使能 CPU Interface
   │         │   02-§1.4: CPU Interface 初始化——让中断能到达 CPU
   │         │
   │         └── gic_pm_init()     — 电源管理准备
   │
   └── ③ gic_smp_init()     — SGI (核间中断) 支持，仅第一次调用
```

施工图画出了 GIC 要做哪些事。第一步 `gic_of_setup()` 要映射 GIC 的寄存器，寄存器地址从 DTS 的 `reg` 属性中来。**先看 GIC 的 DTS 节点——它描述了硬件的位置和能力，后面的代码分析会反复引用它。**

### 3.3.1 GIC DTS 节点

ATK 板的 GIC 在 SoC dtsi 中定义：

```dts
// arch/arm64/boot/dts/st/stm32mp251.dtsi
intc: interrupt-controller@4ac00000 {
    compatible = "st,stm32mp2-cortex-a7-gic", "arm,cortex-a7-gic";
    reg = <0x0 0x4ac10000 0x0 0x1000>,     // [0] Distributor
          <0x0 0x4ac20000 0x0 0x2000>,     // [1] CPU Interface
          <0x0 0x4ac40000 0x0 0x2000>,     // [2] Virtual CPU (KVM)
          <0x0 0x4ac60000 0x0 0x2000>;     // [3] Virtual Ctrl (KVM)
    interrupt-controller;
    #interrupt-cells = <3>;
};
```

| idx | 区 | 地址 | 长度 | 谁映射 |
|-----|-----|------|------|-------|
| 0 | Distributor | 0x4ac10000 | 4KB | GIC 驱动 |
| 1 | CPU Interface | 0x4ac20000 | 8KB | GIC 驱动 |
| 2 | Virtual CPU | 0x4ac40000 | 8KB | KVM（驱动不映射） |
| 3 | Virtual Ctrl | 0x4ac60000 | 8KB | KVM（驱动不映射） |

`reg[2]` 和 `reg[3]` 不由 GIC 驱动映射，仅由 KVM 在 `gic_of_setup_kvm_info()` 中提取物理地址。

硬件描述（DTS）是静态的，**它不会主动触发代码执行**。下一个问题是：谁来调用 `gic_of_init()`？GIC 不走 `module_platform_driver` 那种 platform bus 匹配，而是通过 **`IRQCHIP_DECLARE` 宏**在编译期注册，在 `start_kernel()` 中被 `of_irq_init()` 遍历调用。下面看这个机制。

### 3.3.2 IRQCHIP_DECLARE 注册机制

GIC 驱动通过一个宏把自己注册到内核中：

```c
// drivers/irqchip/irq-gic.c:1569
IRQCHIP_DECLARE(stm32mp2_cortex_a7_gic, "st,stm32mp2-cortex-a7-gic", gic_of_init);
```

**这不是函数调用，不产生执行代码。** 它告诉编译器：在 `__irqchip_of_table` 这个特殊 ELF 段中放一条记录。

宏展开过程：

```c
// 第一层：irqchip.h
#define IRQCHIP_DECLARE(name, compat, fn) \
    OF_DECLARE_2(irqchip, name, compat, typecheck_irq_init_cb(fn))

// 第二层：of.h
#define OF_DECLARE_2(table, name, compat, fn)                    \
    static const struct of_device_id __of_table_##name           \
        __used __section("__irqchip_of_table")                   \
        = { .compatible = compat, .data = fn }
```

完全展开后，编译器生成：

```c
static const struct of_device_id __of_table_stm32mp2_cortex_a7_gic
    __used __attribute__((__section__("__irqchip_of_table")))
    = {
        .compatible = "st,stm32mp2-cortex-a7-gic",
        .data       = gic_of_init,
    };
```

**关键：** 这不是一个数组，而是一个 `of_device_id` 结构体，被放在名为 `__irqchip_of_table` 的 ELF 段中。**链接时，所有 `.o` 文件中同名段的内容被合并成一个连续数组。**

`__irqchip_of_table` 段在内存中的布局：

```
┌──────────────────────────────────────────────┐
│ .compatible = "arm,gic-400"                   │
│ .data       = gic_of_init                     │
├──────────────────────────────────────────────┤
│ .compatible = "arm,cortex-a7-gic"             │
│ .data       = gic_of_init                     │
├──────────────────────────────────────────────┤
│ .compatible = "st,stm32mp2-cortex-a7-gic"    │ ← ATK 板匹配这个
│ .data       = gic_of_init                     │
├──────────────────────────────────────────────┤
│ ... 其他驱动（如 EXTI 不在这里）              │
├──────────────────────────────────────────────┤
│ { }  /* 空终止 */                             │
└──────────────────────────────────────────────┘
```

**为什么 GIC 要用 IRQCHIP_DECLARE 而不是 module_platform_driver？**

| 机制 | 初始化阶段 | 可用内存分配 | 适合场景 |
|------|-----------|-------------|---------|
| `IRQCHIP_DECLARE` → `of_irq_init()` | `start_kernel()` 内部，`mm_init()` **之前** | 只能用静态分配（`gic_data[]` 编译期数组） | 根中断控制器、必须最早初始化的控制器 |
| `module_platform_driver` → `do_initcalls()` | `mm_init()` **之后** | 可用 `kzalloc` / `devm_kzalloc` | 普通外设驱动、依赖 GIC 存在的子控制器 |

GIC 必须在 `mm_init()` 之前初始化（kmalloc 后端还没就绪），所以只能用编译期就存在的 `gic_data[]` 静态数组。而 EXTI 可以等内存系统就绪后用 `devm_kzalloc`。

回到时间线。`IRQCHIP_DECLARE` 把 `gic_of_init` 的地址放入了 `__irqchip_of_table` 段。但只有段还不够——`of_irq_init()` 要做的是**将段中的条目与 DTS 中标记了 `interrupt-controller` 的节点逐一匹配**。它遍历 DTS，找到 `compatible = "st,stm32mp2-cortex-a7-gic"` 且声明了 `interrupt-controller` 的节点（即 `intc@4ac00000`），然后从 `__irqchip_of_table` 中取出对应的 `gic_of_init` 函数指针并调用。**下面进入 `gic_of_init` 的代码。**

### 3.3.3 gic_of_init() — 入口

`start_kernel()` 的执行路径：

```
start_kernel()
  → init_IRQ()                        // arch/arm64/kernel/irq.c
    → irqchip_init()                  // drivers/irqchip/irqchip.c
      → of_irq_init(__irqchip_of_table)
```

`of_irq_init()` 的核心逻辑分为两步：

**第 1 步：收集 DTS 中所有匹配的中断控制器节点**

```c
// kernel/irq/irqdomain.c
for_each_matching_node_and_match(np, matches, &match) {
    if (!of_property_read_bool(np, "interrupt-controller"))
        continue;
    if (!of_device_is_available(np))
        continue;

    desc = kzalloc(sizeof(*desc), GFP_KERNEL);
    desc->irq_init_cb = match->data;            // = gic_of_init
    desc->dev = of_node_get(np);                // = intc 节点

    // 确定 parent：没有 interrupts 属性的节点为根
    desc->interrupt_parent = of_irq_find_parent(np);
    if (desc->interrupt_parent == np)
        desc->interrupt_parent = NULL;           // 根

    list_add_tail(&desc->list, &intc_desc_list);
}
```

**第 2 步：按 parent→child 的 BFS 拓扑排序初始化**

```c
while (!list_empty(&intc_desc_list)) {
    // 找出所有 parent == NULL 的节点（根），初始化它们
    list_for_each_entry_safe(desc, ...) {
        if (desc->interrupt_parent != parent)
            continue;
        desc->irq_init_cb(desc->dev, desc->interrupt_parent);
        // 对 GIC: → gic_of_init(intc_node, NULL)
    }
    // 已初始化的节点成为下一轮的 parent
    parent = next_initialized_node();
}
```

GIC 没有 parent（`interrupts` 属性为空），所以在第一轮被初始化。**同一轮可能有多个根节点**（如 GIC + 其他 IRQCHIP_DECLARE 的根控制器），所有根都初始化完后，它们的子节点在下一轮初始化。

在 ATK 板上，`of_irq_init` **只初始化了 GIC**。EXTI 不在 `__irqchip_of_table` 段中，所以不被处理。

`of_irq_init` 调用了 `gic_of_init`。进入函数后，**第一件实际工作是映射 GIC 的硬件寄存器**——`gic_of_setup`。GIC 的 Distributor 和 CPU Interface 位于不同的物理地址，需要分别 ioremap。

### 3.3.4 gic_of_setup() — 寄存器映射

```c
// irq-gic.c:1495
static int __init gic_of_setup(struct gic_chip_data *gic,
                                struct device_node *node)
{
    // reg[0]: Distributor — 直接 ioremap
    gic->raw_dist_base = of_iomap(node, 0);
    if (WARN(!gic->raw_dist_base, "unable to map gic dist registers\n"))
        goto error;

    // reg[1]: CPU Interface — 先取物理地址（KVM 需要），再 ioremap
    struct resource cpuif_res;
    of_address_to_resource(node, 1, &cpuif_res);
    gic->cpu_phys_base = cpuif_res.start;
    gic->raw_cpu_base = of_iomap(node, 1);
    if (WARN(!gic->raw_cpu_base, "unable to map gic cpu registers\n"))
        goto error;

    // 非 banked GIC 才需要 per-CPU 偏移
    if (of_property_read_u32(node, "cpu-offset", &gic->percpu_offset))
        gic->percpu_offset = 0;

    gic_enable_of_quirks(node, gic_quirks, gic);
    return 0;
}
```

**两个地址区的处理其实是一样的**——都用了 `of_iomap()` 做映射。区别在于 CPU Interface 多做了一个步骤：`of_address_to_resource()` 读出它的物理地址并保存到 `gic->cpu_phys_base`。原因是 KVM 虚拟化需要这个物理地址来配置客户机的中断控制器（`gic_of_setup_kvm_info()` 在 `gic_of_init` 末尾调用）。Distributor 的物理地址没有单独保存，因为 KVM 不需要它——KVM 只关心 CPU Interface（用于注入中断给虚拟机）。

执行后 `gic_data[0]` 的寄存器映射状态：

```
gic_data[0]:
  .raw_dist_base → 0x4ac10000  (ioremap'd, 4KB)
  .raw_cpu_base  → 0x4ac20000  (ioremap'd, 8KB)
  .cpu_phys_base  = 0x4ac20000  (物理地址，KVM 用)
  .percpu_offset  = 0
```

此时驱动已经能读写 GIC 的硬件寄存器，但 GIC 还没有被配置——所有中断仍然被硬件屏蔽。

寄存器映射只解决了"能访问硬件"的问题，接下来才做真正的初始化。**`__gic_init_bases()` 干两件本质不同的事：** 一是安装根中断入口 `set_handle_irq(gic_handle_irq)`，从此 CPU 中断不会 panic 了；二是调用 `gic_init_bases()` 完成硬件配置和 irq_domain 创建。

### 3.3.5 __gic_init_bases() — 安装根 handler + 启动硬件

```c
// irq-gic.c:1249
static int __init __gic_init_bases(struct gic_chip_data *gic,
                                    struct fwnode_handle *handle)
{
    // ★ 关键：安装顶层 IRQ handler
    if (gic == &gic_data[0]) {
        set_handle_irq(gic_handle_irq);
    }

    ret = gic_init_bases(gic, handle);   // 硬件初始化 + irq_domain

    if (gic == &gic_data[0])
        gic_smp_init();                   // IPI 支持（SGIs）

    return ret;
}
```

**`set_handle_irq(gic_handle_irq)`** 替换之前那个会 panic 的 `default_handle_irq`：

```c
// arch/arm64/kernel/irq.c
int __init set_handle_irq(void (*handle_irq)(struct pt_regs *))
{
    if (handle_arch_irq != default_handle_irq)
        return -EBUSY;          // 防止重复设置
    handle_arch_irq = handle_irq;
    return 0;
}
```

**从这里开始，CPU 有了真正的根中断入口。** 在这之前任何中断都会 panic。

入口问题解决后，`__gic_init_bases` 调用 `gic_init_bases` 做真正的硬件初始化。**`gic_init_bases` 依次做 5 件事：读硬件中断数、创建 irq_domain、配置 Distributor、配置 CPU Interface、电源管理。** 下面逐个展开。

### 3.3.6 gic_init_bases() — 硬件初始化 + irq_domain 创建

```c
// irq-gic.c:1175
static int __init gic_init_bases(struct gic_chip_data *gic,
                                  struct fwnode_handle *handle)
{
    int gic_irqs, ret;

    // ① 读 GIC_DIST_CTR 寄存器 — 确定硬件支持的中断数
    gic_irqs = readl_relaxed(gic_data_dist_base(gic) + GIC_DIST_CTR) & 0x1f;
    gic_irqs = (gic_irqs + 1) * 32;
    if (gic_irqs > 1020)
        gic_irqs = 1020;

    // ② 创建 irq_domain（线性映射）
    gic->domain = irq_domain_create_linear(handle, gic_irqs,
                                            &gic_irq_domain_hierarchy_ops,
                                            gic);

    // ③ Distributor 硬件初始化
    gic_dist_init(gic);

    // ④ CPU Interface 初始化
    ret = gic_cpu_init(gic);

    // ⑤ 电源管理
    ret = gic_pm_init(gic);
    return 0;
}
```

上面是 `gic_init_bases` 的完整代码。下面按顺序展开每一步的内部实现。

**代码第①步——读 `GIC_DIST_CTR` 寄存器确定硬件支持的中断数量。** GICv2 用这个寄存器报告它实现了多少个中断线。

#### 3.3.6.1 GIC_DIST_CTR — 读出硬件支持的中断数

```c
gic_irqs = readl_relaxed(gic_data_dist_base(gic) + GIC_DIST_CTR) & 0x1f;
```

`GIC_DIST_CTR`（偏移 `0x0004`）的 `bit[4:0]` 表示支持的中断线组数（每组 32 个），返回值 = 组数 − 1：

```
寄存器: 0x4ac10004
bit[4:0] = 0x1f → 32 组 → 32 × 32 = 1024 → 上限 1020
```

GICv2 规范规定中断 ID 最大为 1019（1020~1023 保留），所以取 `min(1024, 1020) = 1020`。

中断 ID 范围：

| ID 范围 | 类型 | 数量 | 用途 |
|---------|------|------|------|
| 0~15 | SGI | 16 | 核间中断（IPI） |
| 16~31 | PPI | 16 | 每核私有中断（Timer, PMU） |
| 32~1019 | SPI | 988 | 外设共享中断 |
| 1020~1023 | — | 4 | 保留 |

知道了中断数量（1020）和它们的分类（SGI/PPI/SPI），**代码第②步创建 irq_domain 来管理这些中断的 hwirq → virq 映射关系。** GIC 使用线性映射域——因为 hwirq 0~1019 是连续的。

#### 3.3.6.2 irq_domain_create_linear — 创建"空网"

```c
gic->domain = irq_domain_create_linear(handle, gic_irqs,
                                        &gic_irq_domain_hierarchy_ops,
                                        gic);
```

展开后等价于调用 `__irq_domain_add()`：

```c
// include/linux/irqdomain.h
static inline struct irq_domain *irq_domain_create_linear(
    struct fwnode_handle *fwnode, unsigned int size,
    const struct irq_domain_ops *ops, void *host_data)
{
    return __irq_domain_add(fwnode, size, size, 0, ops, host_data);
}
```

`__irq_domain_add()` 分配了一段连续内存，包含 `irq_domain` 结构体本身和 `revmap[]` 柔性数组：

```c
// kernel/irq/irqdomain.c
struct irq_domain *__irq_domain_add(struct fwnode_handle *fwnode,
                                     unsigned int size,        // = 1020
                                     irq_hw_number_t hwirq_max, // = 1020
                                     int direct_max,            // = 0
                                     const struct irq_domain_ops *ops,
                                     void *host_data)           // = gic_data[0]
{
    domain = kzalloc_node(struct_size(domain, revmap, size), GFP_KERNEL, ...);
    // struct_size(domain, revmap, 1020) ≈ sizeof(irq_domain) + 1020 * 8

    domain->ops         = ops;          // → gic_irq_domain_hierarchy_ops
    domain->host_data   = host_data;    // → gic_data[0]
    domain->hwirq_max   = hwirq_max;    // → 1020
    domain->revmap_size = size;         // → 1020（走数组不走 radix tree）
    domain->fwnode      = fwnode;

    list_add(&domain->link, &irq_domain_list);  // 全局可见！
    debugfs_add_domain_dir(domain);

    return domain;
}
```

分配后 `revmap` 数组的内容（全是 NULL——此时没有任何中断被映射）：

```
revmap[0]       = NULL   ← hwirq  0  (SGI #0)
revmap[1]       = NULL   ← hwirq  1  (SGI #1)
...
revmap[15]      = NULL   ← hwirq 15  (SGI #15)
revmap[16]      = NULL   ← hwirq 16  (PPI #0 - Timer)
revmap[25]      = NULL   ← hwirq 25  (PPI #9 - 架构 Timer)
...
revmap[31]      = NULL   ← hwirq 31  (PPI #15)
revmap[32]      = NULL   ← hwirq 32  (SPI #0)
...
revmap[273]     = NULL   ← hwirq 273 (EXTI 的 SPI#268+5, 将来填充)
revmap[300]     = NULL   ← hwirq 300 (SPI #268, EXTI 的 GIC 中断)
...
revmap[1019]    = NULL   ← hwirq 1019
```

**为什么叫"线性"映射？** `revmap` 是一个数组，`hwirq` 就是数组下标。查找 `irq_find_mapping(domain, hwirq)` 直接返回 `domain->revmap[hwirq]`——O(1)。对立面是 radix tree 域（`irq_domain_create_tree`），用于 hwirq 稀疏的场景（如 MSI），但查找是 O(log n)。

**GIC 为什么用线性域？** GIC 的 hwirq 范围是连续的 0~1019，几乎每个号都可能被用到。用数组正好一一对应，查找零开销。如果改用树形域，1020 个 slot 的 radix tree 反而更慢更复杂。

**这张空网什么时候被填充？** 当设备第一次引用中断时，`irq_create_of_mapping()` → `irq_domain_alloc_irqs()` → `irq_domain_insert_irq()` 会填充：

```c
static void irq_domain_insert_irq(int virq)
{
    struct irq_data *data;
    for (data = irq_get_irq_data(virq); data; data = data->parent_data) {
        if (data->hwirq < data->domain->revmap_size)
            rcu_assign_pointer(data->domain->revmap[data->hwirq], data);
    }
}
```

```
revmap[300] ← irq_data     // 当 EXTI 的 SPI#268 被映射时
```

irq_domain 建好了（虽然 revmap 里全是 NULL），**代码第③步开始配置 Distributor 硬件。** Distributor 负责接收所有中断信号、判断优先级、路由到指定的 CPU。这里要关闭→配置→重新使能。

#### 3.3.6.3 gic_dist_init() — Distributor 硬件初始化

```c
// irq-gic.c:475
static void gic_dist_init(struct gic_chip_data *gic)
{
    unsigned int i;
    u32 cpumask;
    unsigned int gic_irqs = gic->gic_irqs;
    void __iomem *base = gic_data_dist_base(gic);

    // ① 全局关闭 Distributor
    writel_relaxed(0, base + GIC_DIST_CTRL);    // GICD_CTLR = 0

    // ② 所有 SPI 路由到 CPU0
    cpumask = gic_get_cpumask(gic);             // = 0x01
    cpumask |= cpumask << 8;                    // = 0x0101
    cpumask |= cpumask << 16;                   // = 0x01010101
    for (i = 32; i < gic_irqs; i += 4)
        writel_relaxed(cpumask, base + GIC_DIST_TARGET + i * 4 / 4);

    // ③ 配置优先级和触发类型
    gic_dist_config(base, gic_irqs, NULL);

    // ④ 全局开启 Distributor
    writel_relaxed(1, base + GIC_DIST_CTRL);    // GICD_CTLR = 1
}
```

**① GIC_DIST_CTRL (0x0000)** — 写 0 禁止所有中断转发；写 1 使能。

**② GIC_DIST_TARGET (0x8000)** — 每个 SPI 有 8 位 CPU mask：

```
GIC_DIST_TARGET + 0x00: byte[0]=CPU mask for IRQ 32
                          byte[1]=CPU mask for IRQ 33
                          byte[2]=CPU mask for IRQ 34
                          byte[3]=CPU mask for IRQ 35
```

`cpumask = 0x01010101` 将 IRQ 32~35 都路由到 CPU0。循环覆盖所有 SPI（32~1019）。

**③ gic_dist_config()** — 配置优先级、触发类型、初始禁用：

```c
static void gic_dist_config(void __iomem *base, int gic_irqs, ...)
{
    unsigned int i;

    // 优先级寄存器 GIC_DIST_PRI (0x400)
    // 每 byte 一个中断的优先级，32bit 寄存器控制 4 个中断
    for (i = 32; i < gic_irqs; i += 4)
        writel_relaxed(0xa0a0a0a0, base + GIC_DIST_PRI + i);

    // 触发类型寄存器 GIC_DIST_CONFIG (0xC00)
    // 每 2 bit 一个中断: 00=电平, 01=边沿
    for (i = 32; i < gic_irqs; i += 16)
        writel_relaxed(0, base + GIC_DIST_CONFIG + i);  // 默认电平触发

    // 禁用所有中断（写 GIC_DIST_ENABLE_CLEAR 寄存器）
    for (i = 32; i < gic_irqs; i += 4)
        writel_relaxed(0, base + GIC_DIST_ENABLE + i);
}
```

**优先级 0xa0**：ARM 优先级数值越小优先级越高，0x00 最高，0xFF 最低。0xa0 是 Linux 给非安全 SPI 的默认值。

Distributor 就绪后，**代码第④步初始化 CPU Interface——这是中断到达 CPU 的最后一关。** 它的工作是设置优先级阈值（只有足够高优先级的中断才能通过）和使能自身。

#### 3.3.6.4 gic_cpu_init() — CPU Interface 使能

```c
// irq-gic.c:498
static int gic_cpu_init(struct gic_chip_data *gic)
{
    void __iomem *base = gic_data_cpu_base(gic);
    void __iomem *dist_base = gic_data_dist_base(gic);

    // 配置 PPI（IRQ 16~31）的优先级
    gic_cpu_config(dist_base, 32, NULL);

    // 写优先级阈值寄存器 — 只允许优先级高于 0xf0 的中断通过
    writel_relaxed(0xf0, base + GIC_CPU_PRIMASK);   // GICC_PMR

    // 使能 CPU Interface
    writel_relaxed(1, base + GIC_CPU_CTRL);          // GICC_CTLR = 1
    return 0;
}
```

| 寄存器 | 偏移 | 值 | 含义 |
|--------|------|---|------|
| GICC_PMR | 0x0004 | 0xf0 | 优先级阈值：只允许优先级数值 < 0xf0（即优先级更高）的中断通过 |
| GICC_CTLR | 0x0000 | 0x1 | CPU Interface 使能 |

到此 GIC 的软硬件全部初始化完成。**下面汇总 GIC 初始化后的完整状态**——包括内存中的 `gic_data[0]`、irq_domain、硬件寄存器值。

### 3.3.7 GIC 初始化完成后的状态

```
┌─────────────────────────────────────────────────┐
│ handle_arch_irq = gic_handle_irq                  │  ← 3.2 中的入口
├─────────────────────────────────────────────────┤
│ gic_data[0]:                                      │
│   .domain      → irq_domain (linear, 1020 items) │
│   .raw_dist_base → 0x4ac10000 (ioremap'd)         │
│   .raw_cpu_base  → 0x4ac20000 (ioremap'd)         │
│   .gic_irqs     → 1020                            │
├─────────────────────────────────────────────────┤
│ Distributor 硬件：                                 │
│   GICD_CTLR        = 1                            │
│   GICD_ISENABLER   = 0（所有中断禁用）              │
│   GICD_ITARGETSR   = 0x01010101...（全路由 CPU0）   │
│   GICD_IPRIORITYR  = 0xa0...（SPI 优先级 0xa0）    │
├─────────────────────────────────────────────────┤
│ CPU Interface 硬件：                               │
│   GICC_CTLR        = 1                            │
│   GICC_PMR         = 0xf0                         │
├─────────────────────────────────────────────────┤
│ irq_domain (gic_data[0].domain):                  │
│   revmap[0..1019] = 全 NULL                       │
│   ops = gic_irq_domain_hierarchy_ops              │
│     .translate → gic_irq_domain_translate          │
│     .alloc     → gic_irq_domain_alloc              │
│   host_data = gic_data[0]                          │
│   parent = NULL (根)                               │
└─────────────────────────────────────────────────┘
```

此时 GIC 全部就绪。**但系统还在启动过程中。** `start_kernel()` 完成 `init_IRQ()` 后继续执行，经过 `mm_init()`（内存初始化——kmalloc 开始可用）→ `do_basic_setup()` → `do_initcalls()`。在这里，`module_init` 级别的驱动被加载。

GIC 虽然就绪了，但它只知道原始中断号（SPI 42、SPI 268...），不知道这些 SPI 对应什么设备。对于 GPIO 中断，中间有一个 **EXTI（Extended Interrupt Controller）** 负责边沿检测和转发。EXTI 走标准 `platform_driver` 路径，在 `do_initcalls()` 中被 probe。

---

## 3.4 阶段二：EXTI 初始化（第一级子域）

### 3.4.0 对比：为什么 EXTI 不走 of_irq_init

| | GIC | EXTI |
|--|-----|------|
| 注册机制 | `IRQCHIP_DECLARE` | `module_platform_driver` |
| 初始化阶段 | `start_kernel()` → `init_IRQ()` | `do_initcalls()` → `.initcall6` |
| 内存分配 | 静态数组 `gic_data[]` | `devm_kzalloc` |
| 对 kmalloc 的依赖 | 无（`mm_init()` 之前） | 有（`mm_init()` 之后） |
| 能否使用设备模型 | 否 | 是（`platform_device`） |
| 谁找到 DTS 节点 | `of_irq_init()` 遍历 | platform bus 匹配 |

EXTI 的初始化需要 GIC 的 irq_domain 已经存在（EXTI 需要将它的 SPI 中断号注册到 GIC domain 中），所以它必须在 GIC 之后初始化。而且 EXTI 使用 `devm_kzalloc` 分配内存、使用 `platform_driver` 的设备模型，这些都依赖 `mm_init()` 之后的运行环境。

和 GIC 的流程一样，**先看 EXTI 的 DTS 节点——它和 GIC 的有什么不同？** 最明显的区别是多了 `interrupts-extended` 属性（EXTI 本身占用 GIC 的 SPI），而且使用双 cell 而不是 GIC 的三 cell。

### 3.4.1 EXTI DTS 节点

```dts
// stm32mp251.dtsi
exti1: interrupt-controller@44220000 {
    compatible = "st,stm32mp1-exti";            // 匹配 stm32mp1-exti 驱动
    interrupt-controller;
    #interrupt-cells = <2>;                     // 双 cell: <line type>
    reg = <0x44220000 0x400>;                   // 寄存器基址
    st,irqs = <268 269 270 271 272 273 274 275 276 277>;
    interrupts-extended =
        <&intc GIC_SPI 268 IRQ_TYPE_LEVEL_HIGH>,   // EXTI line group 0
        <&intc GIC_SPI 269 IRQ_TYPE_LEVEL_HIGH>,   // EXTI line group 1
        ...
        <&intc GIC_SPI 277 IRQ_TYPE_LEVEL_HIGH>;   // EXTI line group 9
};
```

两个 EXTI 实例：`exti1`（主域，0x44220000）和 `exti2`（安全域，0x46230000）。GPIO 中断走 `exti1`。

**`interrupts-extended`** 中的每一行对应 EXTI 输出到 GIC 的一个 SPI 中断。EXTI 有 64 个内部中断线（2 bank × 32 lines），但只有 10 根输出线连到了 GIC（SPI 268~277），每根输出线处理一组内部中断线。

看完 DTS，下面看 `stm32mp_exti_probe` 的完整代码——它和 GIC 的 `gic_of_init` 结构类似，但有几个关键区别。

### 3.4.2 stm32mp_exti_probe() 施工图

```
stm32mp_exti_probe(pdev)
   │
   ├── ① 分配主结构体 + chips_data 数组
   │     目的：EXTI 驱动不使用通用框架。它手动分配 host_data
   │           （寄存器基址、domain 指针、驱动信息）和 chips_data[]
   │           （每 bank 一个 chip_data，保存寄存器偏移表 + 运行时状态）。
   │     → host_data = devm_kzalloc()
   │     → host_data->chips_data = devm_kcalloc(bank_nr)
   │     → host_data->base = devm_platform_ioremap_resource(pdev, 0)
   │
   ├── ② 初始化每个 EXTI bank 的硬件
   │     目的：配置 EXTI_CR 寄存器，选通 GPIO 信号到对应的 EXTI line。
   │           上电后的 EXTI 不知道 GPIO 信号走哪条路进来，必须显式配置。
   │     → for_each bank: stm32mp_exti_chip_init()
   │
   ├── ③ 创建层级 irq_domain（parent = GIC）
   │     目的：EXTI 是 GIC 的子控制器。它的 domain 通过 parent 指针
   │           挂在 GIC domain 下。中断分配时，core 层自动穿越两层。
   │     → parent_domain = irq_find_host(GIC_node)
   │     → domain = irq_domain_add_hierarchy(parent_domain, 0, 64,
   │                                          &stm32mp_exti_domain_ops,
   │                                          host_data)
   │     与 GIC domain 的区别：
   │       - 大小 64（2 bank × 32 lines）
   │       - host_data 非空（传 host_data，用于 alloc 回调）
   │       - 有 parent（GIC domain）
   │
   └── ④ alloc 回调动态选择 irq_chip
        目的：EXTI 有两种中断事件类型——需要边沿检测的（通过 EXTI）
             和不需要的（直通 GIC）。alloc 时读硬件寄存器判断类型，
             为每个中断线选择合适的 irq_chip。
         → stm32mp_exti_alloc():
             ├─ event_trg = readl(TRG register)
             ├─ if (event_trg & BIT): 用 stm32mp_exti_chip（EXTI 事件）
             │     .irq_mask   = stm32mp_exti_mask   写 IMR
             │     .irq_unmask = stm32mp_exti_unmask 写 IMR
             │     .irq_set_type = stm32mp_exti_set_type 写 RTSR/FTSR
             │     .irq_eoi    = stm32mp_exti_eoi   清 pending
             │     .irq_ack    = irq_chip_ack_parent  委托 GIC
             └─ else: 用 stm32mp_exti_chip_direct（直通 GIC）
                   .irq_set_type = irq_chip_set_type_parent  全委托 GIC
```

**这条路径没有链式 handler，没有 generic irq_chip。** EXTI 通过`irq_domain_add_hierarchy` 成为 GIC 的子域，中断信号通过 irq_data 的 parent_data 链自动传递。



### 3.4.3 Probe 逐段分析

EXTI probe 函数源码（`irq-stm32mp-exti.c:836`）：

```c
static int stm32mp_exti_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct device_node *np = dev->of_node;
    struct stm32mp_exti_host_data *host_data;
    const struct stm32mp_exti_drv_data *drv_data;
    struct irq_domain *parent_domain, *domain;
    int ret, i;

    // ── ① 分配主结构体 ──
    host_data = devm_kzalloc(dev, sizeof(*host_data), GFP_KERNEL);
    dev_set_drvdata(dev, host_data);
    host_data->dev = dev;
    raw_spin_lock_init(&host_data->gpio_mux_lock);

    // 可选硬件自旋锁（ATK 板 DTS 没配，跳过）
    ret = of_hwspin_lock_get_id(np, 0);
    if (ret >= 0)
        host_data->hwlock = devm_hwspin_lock_request_specific(dev, ret);

    // ── ② 获取匹配数据 ──
    drv_data = of_device_get_match_data(dev);
    host_data->drv_data = drv_data;

    // ── ③ 分配 per-bank chip_data ──
    host_data->chips_data = devm_kcalloc(dev, drv_data->bank_nr,
                              sizeof(*host_data->chips_data), GFP_KERNEL);

    // ── ④ 寄存器映射 ──
    host_data->base = devm_platform_ioremap_resource(pdev, 0);

    // ── ⑤ 硬件初始化 ──
    for (i = 0; i < drv_data->bank_nr; i++)
        stm32mp_exti_chip_init(host_data, i, np);

    // ── ⑥ 安全隔离检查 ──
    stm32mp_exti_check_rif(host_data);

    // ── ⑦ 找 GIC domain，创建层级子域 ──
    parent_domain = irq_find_host(of_irq_find_parent(np));
    domain = irq_domain_add_hierarchy(parent_domain, 0,
                        drv_data->bank_nr * IRQS_PER_BANK,
                        np, &stm32mp_exti_domain_ops, host_data);

    // ── ⑧ 标记 dt_has_irqs_desc（用于 alloc） ──
    if (of_property_read_bool(np, "interrupts-extended"))
        host_data->dt_has_irqs_desc = true;

    return 0;
}
```

#### ① host_data — 主结构体

```c
host_data = devm_kzalloc(dev, sizeof(*host_data), GFP_KERNEL);
dev_set_drvdata(dev, host_data);
host_data->dev = dev;
raw_spin_lock_init(&host_data->gpio_mux_lock);
```

`stm32mp_exti_host_data` 是 EXTI 驱动的顶层结构体，类似 GIC 的 `gic_data[0]`：

```c
struct stm32mp_exti_host_data {
    struct device *dev;
    void __iomem *base;                          // 寄存器基址
    struct irq_domain *domain;                   // irq_domain
    struct stm32mp_exti_chip_data *chips_data;   // 每 bank 一个
    const struct stm32mp_exti_drv_data *drv_data;
    struct hwspinlock *hwlock;                   // 可选硬件自旋锁
    raw_spinlock_t gpio_mux_lock;
    u32 gpio_mux_available[STM32MP_GPIO_BANK_MAX];
    bool dt_has_irqs_desc;                       // 是否从 DTS 读 GIC SPI 映射
    bool has_syscon;
};
```

分配后各字段初始状态：

```
host_data:
  .dev      = &pdev->dev
  .base     = NULL      ← 第④步才填充
  .domain   = NULL      ← 第⑦步才填充
  .chips_data = NULL    ← 第③步才填充
  .drv_data = NULL      ← 第②步才填充
```

#### ② drv_data — 匹配到的硬件参数

```c
drv_data = of_device_get_match_data(dev);
```

匹配表（`irq-stm32mp-exti.c:970`）：

```c
static const struct of_device_id stm32mp_exti_ids[] = {
    { .compatible = "st,stm32mp1-exti",  .data = &stm32mp1_drv_data  },
    { .compatible = "st,stm32mp13-exti", .data = &stm32mp13_drv_data },
};
```

ATK 板 DTS 声明 `compatible = "st,stm32mp1-exti"`，拿到 `stm32mp1_drv_data`：

```c
static const struct stm32mp_exti_drv_data stm32mp1_drv_data = {
    .bank_nr    = ARRAY_SIZE(stm32mp_exti_banks),  // 2
    .bank       = stm32mp_exti_banks,               // bank 寄存器偏移表
    .desc_irqs  = stm32mp1_desc_irq,                // EXTI line → GIC SPI 映射表
    .irq_nb     = &stm32mp1_irq_nb,
};
```

填充后：

```
host_data.drv_data → stm32mp1_drv_data
  .bank_nr    = 2
  .bank       → [stm32mp1_exti_bank1, stm32mp1_exti_bank2]
  .desc_irqs  → stm32mp1_desc_irq[STM32MP_DESC_IRQ_SIZE]
```

#### ③ chips_data — per-bank 运行时数据

```c
host_data->chips_data = devm_kcalloc(dev, drv_data->bank_nr,
                          sizeof(*host_data->chips_data), GFP_KERNEL);
```

分配 2 个 `stm32mp_exti_chip_data`，每 bank 一个：

```c
struct stm32mp_exti_chip_data {
    const struct stm32mp_exti_bank *reg_bank;  // 本 bank 的寄存器偏移表
    raw_spinlock_t rlock;                       // 保护寄存器并发
    u32            wake_active;                 // 唤醒中断缓存
    u32            mask_cache;                  // IMR 软件缓存
    unsigned long  event_reserved;              // 安全保留的事件位
    struct stm32mp_exti_host_data *host_data;   // 回指 host_data
};
```

填充前全是 0：

```
host_data.chips_data[0]:
  .reg_bank   = NULL     ← 第⑤步填充
  .rlock      = 未初始化
  .wake_active = 0
  .mask_cache = 0
  .event_reserved = 0

host_data.chips_data[1]: 同上
```

#### ④ base — 寄存器映射

```c
host_data->base = devm_platform_ioremap_resource(pdev, 0);
```

将 DTS `reg = <0x44220000 0x400>` 映射到内核虚拟地址空间：

```
host_data.base → 0x44220000 (ioremap'd, 0x400 bytes)
  0x44220000 + 0x080 = IMR     寄存器
  0x44220000 + 0x0C0 = RTSR   上升沿触发选择
  0x44220000 + 0x0E0 = FTSR   下降沿触发选择
  0x44220000 + 0x100 = RPR    上升沿挂起
  0x44220000 + 0x120 = FPR    下降沿挂起
  0x44220000 + 0x200 = TRG    触发类型（边沿/电平）
```

#### ⑤ stm32mp_exti_chip_init — 硬件初始化

```c
for (i = 0; i < drv_data->bank_nr; i++)
    stm32mp_exti_chip_init(host_data, i, np);
```

对每个 bank 调用。函数内部（`irq-stm32mp-exti.c:767`）：

```c
stm32mp_exti_chip_data *stm32mp_exti_chip_init(
    struct stm32mp_exti_host_data *h_data,
    unsigned int bank_idx, struct device_node *np)
{
    chip_data = &h_data->chips_data[bank_idx];

    // ① 填充 reg_bank：指向 bank 对应的寄存器偏移表
    chip_data->reg_bank = h_data->drv_data->bank[bank_idx];

    // ② 初始化自旋锁
    raw_spin_lock_init(&chip_data->rlock);

    // ③ 初始化 mask_cache
    chip_data->mask_cache = readl_relaxed(h_data->base
                              + chip_data->reg_bank->imr_ofst);

    // ④ 配置 gpio_mux_available（GPIO 输入选通）
    stm32mp_exti_cur_cr_update(h_data, np, true);

    return chip_data;
}
```

第④步读取 EXTI_CR 寄存器，确认哪些 EXTI line 被配置为 GPIO 输入模式。ATK 板启动时 CR 寄存器由 bootloader 配置，驱动只读取验证，不做修改。

初始化后 chips_data 状态：

```
host_data.chips_data[0]:
  .reg_bank   → stm32mp1_exti_bank1 { imr_ofst=0x080, rtsr_ofst=0x0C0,
                                       ftsr_ofst=0x0E0, rpr_ofst=0x100,
                                       fpr_ofst=0x120, trg_ofst=0x200 }
  .mask_cache = 读取 IMR 寄存器当前值（启动时为 0）
  .wake_active = 0
  .event_reserved = 0

host_data.chips_data[1]:
  .reg_bank   → stm32mp1_exti_bank2
```

#### ⑥ stm32mp_exti_check_rif — 安全隔离检查

```c
stm32mp_exti_check_rif(host_data);
```

遍历所有 EXTI line，读取 EXTI_CIDCFGR 寄存器检查哪些 line 被 TZCD 安全隔离（仅 OP-TEE/TF-A 可访问）。被隔离的 line 在 `chip_data->event_reserved` 中置位，Linux alloc 时会拒绝访问。

#### ⑦ irq_domain_add_hierarchy — 创建层级子域

```c
parent_domain = irq_find_host(of_irq_find_parent(np));
domain = irq_domain_add_hierarchy(parent_domain, 0,
                    drv_data->bank_nr * IRQS_PER_BANK,
                    np, &stm32mp_exti_domain_ops, host_data);
```

`of_irq_find_parent(np)` 从 EXTI 节点的 `interrupt-parent`（或 `interrupts-extended`）属性找到 GIC 节点。`irq_find_host` 返回 GIC domain。

`irq_domain_add_hierarchy` 内部分两步：

```c
// __irq_domain_add: 分配 irq_domain 结构体 + revmap[64] 柔性数组
domain = kzalloc(struct_size(domain, revmap, 64), GFP_KERNEL);
domain->ops        = stm32mp_exti_domain_ops;
domain->host_data  = host_data;
domain->hwirq_max  = 64;
domain->revmap_size = 64;    // 线性映射

// 设置 parent 指针
domain->parent = parent_domain;  // ← 关键：链接到 GIC domain
```

创建后 domain 状态：

```
host_data.domain → irq_domain
  .revmap_size = 64
  .revmap[0..63] = 全 NULL    ← 空网，request_irq 时填充
  .parent = GIC_domain
  .ops    = stm32mp_exti_domain_ops
  .host_data = host_data
```

#### ⑧ dt_has_irqs_desc

```c
if (of_property_read_bool(np, "interrupts-extended"))
    host_data->dt_has_irqs_desc = true;
```

ATK 板 DTS 有 `interrupts-extended`，此标志置位。alloc 回调会根据此标志选择路径 A（从 DTS 查 GIC SPI 映射）而非路径 B（从驱动内部 `desc_irqs` 表查）。

#### probe 完成后的完整状态

```
host_data (stm32mp_exti_host_data)
├── .dev        → &pdev->dev
├── .drv_data   → &stm32mp1_drv_data { bank_nr=2, desc_irqs=stm32mp1_desc_irq[] }
├── .base       → ioremap(0x44220000, 0x400)
├── .domain     → irq_domain [linear, 64, parent=GIC_domain]
│                  .revmap[0..63] = 全 NULL
│                  .ops = stm32mp_exti_domain_ops
│                  .host_data = host_data
├── .dt_has_irqs_desc = true
├── .chips_data[0]:
│     .reg_bank   → stm32mp1_exti_bank1 { imr_ofst=0x080, ... }
│     .mask_cache = 0
│     .wake_active = 0
│     .rlock      = (initialized)
│
└── .chips_data[1]:
      .reg_bank   → stm32mp1_exti_bank2 { imr_ofst=... }
      .mask_cache = 0
```

硬件状态：EXTI_CR 已配置 GPIO 输入选通，IMR=0（全屏蔽），RTSR/FTSR=0（无触发选择）。

**alloc 回调分析见下节（链式 vs 层级模型已在 02-Architecture 中详细对比，不重复）。**

### 3.4.4 EXTI 寄存器布局

```c
static const struct stm32mp_exti_bank stm32mp1_exti_bank1 = {
    .imr_ofst    = 0x080,    // Interrupt Mask Register
    .rtsr_ofst   = 0x0C0,    // Rising Trigger Selection
    .ftsr_ofst   = 0x0E0,    // Falling Trigger Selection
    .rpr_ofst    = 0x100,    // Rising Pending Register
    .fpr_ofst    = 0x120,    // Falling Pending Register
    .trg_ofst    = 0x200,    // Trigger Register (edge/level)
    .seccfgr_ofst = 0x140,   // Secure Config Register
};
```

以 PH5 按键中断为例，DTS 配置 `interrupts = <5 IRQ_TYPE_EDGE_BOTH>` 时：

```
EXTI_CR[5] (0x060 + 5/4*4 = 0x064)  bit[7:0] = 0x07
  └─ 选择 GPIOH 作为输入源（0x00=GPIOA, 0x01=GPIOB, ..., 0x07=GPIOH）

EXTI_RTSR (0x0C0)                    bit[5] = 1  ← 上升沿使能
EXTI_FTSR (0x0E0)                    bit[5] = 1  ← 下降沿使能
EXTI_IMR  (0x080)                    bit[5] = 1  ← 取消屏蔽

触发后：
  EXTI_RPR (0x100)  bit[5] = 1  ← 上升沿挂起
  EXTI_FPR (0x120)  bit[5] = 1  ← 下降沿挂起
```

寄存器的配置是静态的——**中断到来时，mask/unmask/eoi/set_type 这些回调操作的就是上述寄存器。** 下面看 EXTI 的 `stm32mp_exti_chip` 中几个关键回调的实现。

### 3.4.5 irq_chip 回调分析

EXTI 驱动定义了两个 irq_chip 变体，它们的 mask/unmask 共享同一套实现，区别主要在 set_type 和 eoi。下面是三个核心回调：

**① `stm32mp_exti_mask` / `stm32mp_exti_unmask` — 写 IMR**

```c
static void stm32mp_exti_mask(struct irq_data *d)
{
    struct stm32mp_exti_chip_data *chip_data = irq_data_get_irq_chip_data(d);
    raw_spin_lock(&chip_data->rlock);

    chip_data->mask_cache &= ~BIT(d->hwirq % IRQS_PER_BANK);
    writel_relaxed(chip_data->mask_cache,
                   chip_data->host_data->base + chip_data->reg_bank->imr_ofst);

    raw_spin_unlock(&chip_data->rlock);
    irq_chip_mask_parent(d);        // ★ 也要 mask GIC 侧
}
```

`irq_chip_mask_parent(d)` 通过 `d->parent_data` 找到 GIC 层的 irq_data，调用 `gic_chip.irq_mask`。这意味着 **mask 一个 EXTI 中断需要在两个地方同时屏蔽**：EXTI 的 IMR 和 GIC 的 ICENABLER。只 mask 一边都不够——EXTI 侧的 IMR 阻止新事件进入，但之前已经 pending 的信号还在 GIC 侧等处理。

```c
static void stm32mp_exti_unmask(struct irq_data *d)
{
    // 与 mask 相反：写 IMR 置位 + irq_chip_unmask_parent(d)
}
```

**② `stm32mp_exti_set_type` — 配 RTSR/FTSR**

```c
static int stm32mp_exti_set_type(struct irq_data *d, unsigned int type)
{
    struct stm32mp_exti_chip_data *chip_data = irq_data_get_irq_chip_data(d);
    u32 mask = BIT(d->hwirq % IRQS_PER_BANK);
    u32 rtsr, ftsr;

    raw_spin_lock(&chip_data->rlock);

    // 读当前 RTSR/FTSR
    rtsr = readl_relaxed(base + chip_data->reg_bank->rtsr_ofst);
    ftsr = readl_relaxed(base + chip_data->reg_bank->ftsr_ofst);

    // 根据 type 设置/清除对应位
    switch (type) {
    case IRQ_TYPE_EDGE_RISING:
        rtsr |= mask;  ftsr &= ~mask;  break;
    case IRQ_TYPE_EDGE_FALLING:
        rtsr &= ~mask; ftsr |= mask;   break;
    case IRQ_TYPE_EDGE_BOTH:
        rtsr |= mask;  ftsr |= mask;   break;
    case IRQ_TYPE_LEVEL_HIGH:
        rtsr &= ~mask; ftsr &= ~mask;  break;
    }

    writel_relaxed(rtsr, base + chip_data->reg_bank->rtsr_ofst);
    writel_relaxed(ftsr, base + chip_data->reg_bank->ftsr_ofst);

    raw_spin_unlock(&chip_data->rlock);
    return 0;
}
```

以 PH5 按键配双边沿触发为例：
```
调用前: RTSR[5]=0 FTSR[5]=0
调用:   stm32mp_exti_set_type(d, IRQ_TYPE_EDGE_BOTH)
执行后: RTSR[5]=1 FTSR[5]=1
```

**③ 完整的中断路径（层级模型下）：**

```
PH5 按键按下 → GPIOH 电平变化 → EXTI 边沿检测
  → EXTI_RPR[5] = 1（挂起位被硬件置位）
    → EXTI 输出信号到 GIC SPI#(268+n)
      → GIC Distributor → CPU Interface → CPU IRQ 引脚
        → el1h_64_irq_handler → gic_handle_irq
          → 读 GICC_IAR → hwirq
            → irq_find_mapping(gic_domain, hwirq) → virq
              → irq_desc[virq].handle_irq = handle_fasteoi_irq
                → stm32mp_exti_mask(d)          写 EXTI_IMR 清位 + GICD_ICENABLER
                → handle_irq_event(desc)
                  → action->handler()           消费者 handler（gpio_keys）
                → stm32mp_exti_eoi(d)           写 EXTI_RPR 清 pending
                  → irq_chip_eoi_parent(d)      写 GICC_EOIR
```

**层级模型下不需要独立的链式 handler：** 核心层通过 irq_data 的 parent_data 链自动在 EXTI 和 GIC 之间委派操作。mask 一个中断时，`stm32mp_exti_mask` 写 IMR，再调 `irq_chip_mask_parent` 写 GICD_ICENABLER——全部自动完成。

EXTI 的初始化到此完成。**汇总当前状态**——和 GIC 一样，在进入下一阶段前先看看我们有了什么。

### 3.4.6 EXTI 初始化完成后的状态

```
┌──────────────────────────────────────────────┐
│ exti1 (0x44220000):                            │
│   .domain    → irq_domain (hierarchy, 64)     │
│     .parent  = GIC_domain                      │
│     revmap[0..63] = 全 NULL                    │
│     ops = stm32mp_exti_domain_ops              │
│   .base      → 0x44220000 (ioremap'd)          │
│   .chips_data→ [bank0 chip_data, bank1 ...]   │
├──────────────────────────────────────────────┤
│ stm32mp_exti_chip（EXTI 事件型）:               │
│   .irq_mask   = stm32mp_exti_mask   写 IMR    │
│   .irq_unmask = stm32mp_exti_unmask            │
│   .irq_set_type = stm32mp_exti_set_type       │
│   .irq_eoi    = stm32mp_exti_eoi   清 pending │
│   .irq_ack    = irq_chip_ack_parent → GIC      │
├──────────────────────────────────────────────┤
│ stm32mp_exti_chip_direct（直通型，无 EXTI）：   │
│   全部委托给 parent（GIC），除 mask/unmask 外  │
├──────────────────────────────────────────────┤
│ 硬件寄存器初始状态：                             │
│   EXTI_CR    = 已配置（GPIO 输入选通）           │
│   EXTI_IMR   = 0x00000000（所有中断屏蔽）        │
│   EXTI_RTSR  = 0x00000000（无上升沿触发）        │
│   EXTI_FTSR  = 0x00000000（无下降沿触发）        │
│   EXTI_RPR   = 0x00000000（无挂起）             │
├──────────────────────────────────────────────┤
│ GIC domain revmap 未新增填充                    │
│ （EXTI 不占用 GIC SPI，层级域通过 parent_data   │
│   链连接，不走独立 SPI 分配）                    │
└──────────────────────────────────────────────┘
```

此时 EXTI 已就绪，通过层级域挂在 GIC 之下。**但系统还在继续启动。** GIC 知道 SPI，EXTI 知道 EXTI line，但它们还缺少最后一层翻译：**GPIO pin 号 → EXTI line 号**。这一层在稍后的 `do_initcalls()` 中由 pinctrl 驱动的 probe 创建。

---

## 3.5 阶段三：GPIO irq_domain（第二级子域）

这是初始化路径的第三层——GPIO bank 为每个 pin 创建层次化 irq_domain，挂在 EXTI domain 之下。

### 3.5.1 注册时机：pinctrl probe 内部

和 EXTI 一样，GPIO irq_domain 也在 `do_initcalls()` 期间创建。但和 EXTI 不同，它**不单独占用一个驱动**——而是作为 pinctrl 合体驱动 probe 的一部分完成的。

GPIO 中断 domain 不单独占用一个 initcall 阶段，而是在 pinctrl 驱动 probe 中完成：

```c
// pinctrl-stm32.c:2033
int stm32_pctl_probe(struct platform_device *pdev)
{
    // ① 创建 IRQ domain（必须在 pinctrl 注册和 GPIO chip 添加之前）
    pctl->domain = stm32_pctrl_get_irq_domain(pctl);
    if (IS_ERR(pctl->domain))
        return PTR_ERR(pctl->domain);

    // ② 后续步骤：注册 pinctrl + GPIO bank
    ...
}
```

**为什么要在 pinctrl 注册之前？** 因为之后 `gpiochip_add_data()` 需要一个已经就绪的 irq_domain 来支持 `gpio_to_irq()` 转换。如果 pinctrl 先注册了，再因为 IRQ 问题 `-EPROBE_DEFER` 退回，之前的工作就白费了。

创建 irq_domain 的具体代码在 `stm32_pctrl_get_irq_domain()` 中。它遍历 GPIO 子节点，为每个 bank 创建一个**层次化 domain**，以 EXTI domain 为 parent。

### 3.5.2 层次化 domain 创建

```c
// pinctrl-stm32.c:1823
static struct irq_domain *stm32_pctrl_get_irq_domain(struct stm32_pinctrl *pctl)
{
    for_each_gpiochip_node(np) {
        // 解析 interrupts-extended 属性找到 parent domain（EXTI）
        // gpioa: gpio@44240000 {
        //     interrupts-extended = <&exti1 1 IRQ_TYPE_LEVEL_HIGH>;
        // };

        of_parse_phandle_with_args(np, "interrupts-extended",
                                   "#interrupt-cells", 0, &args);
        parent = irq_find_host(args.np);        // 找 EXTI domain
        if (!parent)
            return ERR_PTR(-EPROBE_DEFER);       // EXTI 还没好

        // 创建层次化 domain
        domain = irq_domain_create_hierarchy(parent, 0,
                                              STM32_GPIO_IRQ_LINE,  // 16
                                              fwnode,
                                              &stm32_gpio_domain_ops,
                                              pctl);
        return domain;
    }
    return NULL;
}
```

**`irq_domain_create_hierarchy()`** 与 `irq_domain_create_linear()` 的区别在 `parent` 参数：

```c
struct irq_domain *irq_domain_create_hierarchy(struct irq_domain *parent, ...)
{
    struct irq_domain *domain = __irq_domain_add(fwnode, size, size, 0, ops, host_data);
    if (domain)
        domain->parent = parent;       // ← 关键：链接到 parent domain
    return domain;
}
```

这意味着 GPIO domain 和 EXTI domain 之间通过 `parent` 指针关联。当 `request_irq` 触发 domain 分配时，GPIO domain 的 `.alloc` 会递归调用 `irq_domain_alloc_irqs_parent()`，逐级向上穿越 EXTI → GIC。

**到此三层 domain 全部就位**——从 GIC（根）到 EXTI（第一级子域）到 GPIO（第二级子域）。下面用一张总图展示它们的完整关系。

### 3.5.3 三层 domain 层级总图

初始化完成后的三层 domain 层级关系：

```
GIC domain (root, parent=NULL)             02-§3: 根中断控制器
  .ops = gic_irq_domain_hierarchy_ops
  .host_data = gic_data[0]
  revmap[0..1019]
      ↑ parent
      │
  EXTI domain (hierarchy, parent=GIC)      02-§3: 边沿检测、屏蔽
  .ops = stm32mp_exti_domain_ops
  .host_data = host_data (EXTI 主结构体)
  revmap[0..63]
      ↑ parent
      │
  GPIO domain (第二级子域，每个 pinctrl 一个)  02-§3: GPIO pin → EXTI line 映射
  .ops = stm32_gpio_domain_ops
  .host_data = stm32_pinctrl
  revmap[0..15]  (16 GPIO IRQ lines)
```

从图中可以看到三层之间**全部使用层级模型连接**——EXTI 是 GIC 的子域，GPIO 是 EXTI 的子域。

### 3.5.4 三层层级在 STM32MP257 上的具体体现

STM32MP257 的三层中断控制器全部用层级模型连接：

| 关系 | 连接方式 | 层级中的作用 |
|------|---------|-------------|
| GIC（根） | `parent = NULL` | 中断的最终目的地，管理路由/优先级/亲和性 |
| EXTI → GIC | `irq_domain_add_hierarchy(parent=GIC_domain)` | 扩展 GIC：增加边沿检测、触发方式选择、独立屏蔽 |
| GPIO → EXTI | `irq_domain_create_hierarchy(parent=exti_domain)` | 扩展 EXTI：将 GPIO pin 号翻译为 EXTI line 号 |

**每层 chip 回调的分工：**

```
                     mask/unmask        set_type         ack/eoi        affinity
GPIO 层      → 全部 irq_chip_mask_parent() → 委托给 EXTI（GPIO 没有中断寄存器）
                  ↓
EXTI 层      → stm32mp_exti_mask()     → stm32mp_exti_set_type()  委托 GIC   委托 GIC
                  写 IMR 寄存器          写 RTSR/FTSR 寄存器
                  ↓
GIC 层       → gic_mask_irq()          → gic_set_type()          写 GICC   写 GICD
                  写 GICD_ICENABLER      写 GICD_CONFIG
```

为什么是这样的分工？因为每层硬件的实际能力不同：

| 硬件 | 能做什么 | 不能做什么 |
|------|---------|-----------|
| GPIO bank | 检测引脚电平变化 | 没有中断控制寄存器，不能 mask/ack/配置触发类型 |
| EXTI | 边沿检测、独立 IMR、触发类型配置 | 不知道中断应该路由到哪个 CPU（这是 GIC 的事） |
| GIC | 路由、优先级、亲和性、CPU Interface | 不知道 GPIO pin 号和 EXTI line 号的对应关系 |

层级模型让每层只做自己硬件能做的事，不能做的通过 `irq_chip_*_parent()` 委托给上一层：

```
GPIO 的 irq_mask:
  → irq_chip_mask_parent(d)          ← "我不会，找我爸"
    → EXTI 的 stm32mp_exti_mask(d):  ← "我会，我来了"
        写 EXTI_IMR 清除对应位
      → irq_chip_mask_parent(d)      ← "我的部分做完了，再找我爸"
        → GIC 的 gic_mask_irq(d):    ← "我会，我来了"
            写 GICD_ICENABLER
```

**这就是层级模型的核心价值：** 每层的 irq_chip 只实现自己硬件支持的操作，不支持的自动委派给 parent。不需要像链式模型那样写一个手动分发函数来管理嵌套。

三层中断基础设施全部就绪后，**系统继续启动，进入消费者设备的 probe 阶段。** 以 gpio-keys 驱动为例，它在 probe 时调用 `request_irq()` 将按键中断 handler 注册到系统中。下面看这个"运行时注册"过程如何利用前面三层 domain 完成中断号的分配和 handler 的挂载。

---

## 3.6 运行时注册：request_irq 流程（以 gpio-keys 为例）

### 3.6.1 触发时机

以 gpio-keys（PH5 按键）为例，DTS：

```dts
gpio-keys {
    compatible = "gpio-keys";
    button-jump {
        gpios = <&gpioh 5 GPIO_ACTIVE_LOW>;
        interrupts = <5 IRQ_TYPE_EDGE_BOTH>;   // PH5 在 GPIOH 中编号 5
    };
};
```

gpio-keys 驱动 probe 时调 `gpiod_to_irq()` 或 `platform_get_irq()`，触发中断号的分配。**这个分配过程穿越三层 domain**，下面展开看。

### 3.6.2 irq_create_of_mapping — 三层分配

```c
// drivers/of/irq.c
unsigned int irq_create_of_mapping(struct of_phandle_args *irq_data)
{
    struct irq_domain *domain;

    domain = irq_find_host(irq_data->np);   // ① 找 domain
    if (!domain) return 0;

    return irq_create_fwspec_mapping(&fwspec);  // ② 分配 virq
}
```

**① `irq_find_host(gpioh_of_node)`** — 遍历 `irq_domain_list` 全局链表，比对 of_node → 返回 **GPIO domain**（stm32_pinctrl 创建的层次域）。

**② `irq_create_fwspec_mapping()`**：

```c
unsigned int irq_create_fwspec_mapping(struct irq_fwspec *fwspec)
{
    struct irq_domain *domain = irq_find_host(fwspec->fwnode);
    irq_hw_number_t hwirq;
    unsigned int type, virq;

    // ① 通过 .xlate 回调翻译 DTS cell → (hwirq, type)
    domain->ops->xlate(domain, fwspec, &hwirq, &type);
    // GPIO domain 的 xlate 是 irq_domain_xlate_twocell
    // hwirq = 5, type = IRQ_TYPE_EDGE_BOTH

    // ② 分配 virq
    virq = irq_domain_alloc_irqs(domain, hwirq, 1, NUMA_NO_NODE, &fwspec);
    // ③ 配置触发类型
    irq_set_irq_type(virq, type);

    return virq;
}
```

**`irq_domain_alloc_irqs()`** 是分配的核心，它穿越三层 domain：

```
irq_domain_alloc_irqs(GPIO_domain, hwirq=5, nr=1, ...)
   │
   ├─ irq_domain_alloc_irq_data(GPIO_domain, virq, 1)
   │   为 GPIO 层分配 irq_data（hwirq=5）
   │
   ├─ GPIO_domain->ops->alloc(virq, 1, &fwspec)
   │   → stm32_gpio_irq_domain_alloc()
   │        │
   │        ├─ irq_domain_set_hwirq_and_chip(virq, GPIO_domain, hwirq=5,
   │        │      &stm32_gpio_irq_chip, pctl)
   │        │    ← virq 的 irq_data[0] 指向 GPIO 层的 chip 和 hwirq
   │        │
   │        └─ irq_domain_alloc_irqs_parent(domain, virq, 1, &fwspec)
   │             │
   │             ├─ → 递归进入 EXTI_domain->ops->alloc()
   │             │   → stm32mp_exti_alloc(virq, 1, &fwspec)
   │             │        │
   │             │        ├─ irq_domain_set_hwirq_and_chip(virq, EXTI_domain,
   │             │        │      hwirq_line, &gc->chip_types->chip, gc)
   │             │        │    ← irq_data[1] 指向 EXTI 层的 generic chip
   │             │        │
   │             │        └─ irq_domain_alloc_irqs_parent(domain, virq, 1, ...)
   │             │             │
   │             │             ├─ → 递归进入 GIC_domain->ops->alloc()
   │             │             │   → gic_irq_domain_alloc()
   │             │             │        │
   │             │             │        ├─ irq_domain_set_hwirq_and_chip(virq,
   │             │             │        │      GIC_domain, hwirq_spi,
   │             │             │        │      &gic_chip, gic_data)
   │             │             │        │    ← irq_data[2] 指向 GIC 层的 chip
   │             │             │        │
   │             │             │        ├─ gic_set_affinity() → 路由到 CPU0
   │             │             │        └─ gic_irq_domain_map() → 设置 flow handler
   │             │             │              → irq_desc->handle_irq = handle_fasteoi_irq
   │             │             │
   │             │             └─ 返回 (分配完成)
   │             │
   │             └─ 返回
   │
   ├─ irq_domain_insert_irq(virq)
   │   填充每层 domain 的 revmap：
   │   → GIC_domain->revmap[spi_hwirq]   = irq_data[2]
   │   → EXTI_domain->revmap[exti_line]  = irq_data[1]
   │   → GPIO_domain->revmap[pin]        = irq_data[0]
   │
   └─ 返回 virq
```

分配完成后，一个 irq_desc 对应三层的 irq_data：

```
irq_desc[virq]                             02-§4: irq_desc
  .handle_irq = handle_fasteoi_irq         02-§4: flow handler
  .action = NULL                           02-§5: 尚未注册 handler
  .irq_data (内嵌):
    .hwirq = 5                             02-§3: GPIO pin 5
    .chip = &stm32_gpio_irq_chip           02-§2: GPIO 层的 chip
    .domain = GPIO_domain                  02-§3: GPIO domain
    .parent_data → irq_data (alloc):
      .hwirq = exti_line                   02-§3: EXTI line 号
      .chip = &stm32mp_exti_chip          02-§2: EXTI irq_chip
      .domain = EXTI_domain                02-§3: EXTI domain
      .parent_data → irq_data (alloc):
        .hwirq = 268 + n                   ← GIC SPI 号
        .chip = &gic_chip                  02-§2: GIC chip
        .domain = GIC_domain               02-§3: GIC domain
        .parent_data = NULL                ← 根
```

virq 分配完成后，还要把业务 handler 注册到这个 virq 上。**`request_irq` 将设备的 irqaction 挂入 irq_desc->action 链表，并最终通过三层的 irq_chip 使能中断。**

### 3.6.3 request_irq — handler 注册

```c
// drivers/input/keyboard/gpio_keys.c
error = request_any_context_irq(irq, gpio_keys_irq_isr,
                                 irqflags, desc, bdata->input);
```

`request_irq` → `setup_irq()` → `__setup_irq()`：

```
request_irq(virq, gpio_keys_irq_isr, IRQF_TRIGGER_FALLING, ...)
   │
   └─ __setup_irq(virq, desc, new_action)
        │
        ├─ ① 分配 irqaction
        │   → new_action->handler = gpio_keys_irq_isr
        │   → new_action->name = "gpio-keys"
        │   → new_action->dev_id = bdata->input
        │
        ├─ ② IRQF_SHARED 检查
        │   (本 DTS 每个按键独立中断，跳过)
        │
        ├─ ③ 将 action 插入 irq_desc->action 链表
        │   → desc->action = new_action
        │
        ├─ ④ 调用 irq_chip 使能中断
        │   → __irq_setup_irq() 内部
        │   → chip->irq_unmask(data)     ← 三级委派
        │     → GPIO layer: irq_chip_unmask_parent()
        │       → EXTI: stm32mp_exti_mask() 写 IMR
        │         → GIC: gic_unmask_irq() 写 GICD_ISENABLER
        │
        ├─ ⑤ 创建线程（如果 thread_fn 不为 NULL）
        │   (gpio_keys 使用 request_any_context_irq, 可能在 thread 中)
        │
        └─ 返回 0
```

注册完成后：

```
irq_desc[virq]
  .handle_irq = handle_fasteoi_irq
  .action → irqaction
    .handler = gpio_keys_irq_isr
    .name    = "gpio-keys"
    .dev_id  = input_device
    .next    = NULL
```

注册完成后，**回顾从按键按下到 handler 执行的完整信号路径**——把本文前三阶段建立的"基础设施"和第四阶段的"运行时注册"串起来。

### 3.6.4 全路径回顾

从按键按下到 handler 执行：

```
PH5 按下 → GPIOH 检测到电平变化
  → EXTI 边沿检测 → EXTI_RPR bit[5] = 1
    → EXTI 输出信号到 GIC SPI#(268+n)
      → GIC Distributor 识别 pending
        → CPU Interface 向 CPU0 发送 IRQ
          → CPU 查 VBAR_EL1 → vectors[0x280]
            → kernel_ventry 保存 pt_regs
              → __el1_irq → gic_handle_irq()
                → 读 GICC_IAR → hwirq = 300
                  → irq_find_mapping(GIC_domain, 300) → virq
                    → irq_desc[virq].handle_irq = handle_fasteoi_irq
                      → mask (GICD_ICENABLER)
                        → stm32mp_exti_mask()    ← EXTI mask IMR
                          → 读 EXTI_RPR → bit[5] = 1
                            → irq_find_mapping(EXTI_domain, 5) → virq2
                              → generic_handle_irq(virq2)
                                → irq_desc[virq2].handle_irq
                                  → mask/handler/unmask
                                     → ... gpio_keys_irq_isr
                      → eoi (GICC_EOIR)
```

信号路径看完了。**最后汇总整个系统全部就绪后的完整状态**——从 CPU 入口到三层 irq_domain 到硬件寄存器，回答 3.1 开头的问题。

---

## 3.7 全部就绪后的完整状态

三层中断控制器全部初始化完成后：

```
┌─ handle_arch_irq = gic_handle_irq ──────────────────────────┐
│                                                               │
│  GIC domain (root, parent=NULL)            02-§3: 根中断控制器  │
│  .ops = gic_irq_domain_hierarchy_ops                             │
│  revmap[0..1019]  （消费者设备引用中断时填充）                     │
│      ↑ parent                                                    │
│      │                                                           │
│  EXTI domain (hierarchy, parent=GIC)      02-§3: 第一级子域      │
│  .ops = stm32mp_exti_domain_ops                                   │
│  .parent = GIC_domain                                             │
│  revmap[0..63]   （消费者设备引用中断时填充）                      │
│      ↑ parent                                                    │
│      │                                                           │
│  GPIO domain (hierarchy, parent=EXTI)    02-§3: 第二级子域       │
│  .ops = stm32_gpio_domain_ops                                     │
│  revmap[0..15]   （消费者设备引用中断时填充）                      │
│                                                               │
│  GIC chip                 02-§2: irq_chip  写 GICD/GICC 寄存器 │
│  EXTI generic chip        02-§2: irq_chip  写 IMR/RTSR/FTSR   │
│  GPIO irq_chip            (委托给 EXTI)                       │
│                                                               │
│  irq_desc[virq]          02-§4: irq_desc                     │
│  ├─ handle_irq = handle_fasteoi_irq                           │
│  └─ action → gpio_keys_irq_isr  02-§5: irqaction             │
│                                                               │
│  硬件寄存器状态：                                               │
│  GICD_CTLR     = 1           Distributor 使能                  │
│  GICC_CTLR     = 1           CPU Interface 使能                │
│  GICC_PMR      = 0xf0       优先级阈值                         │
│  EXTI_IMR[5]   = 1          GPIOH pin 5 取消屏蔽               │
│  GICD_ISENABLER[spi] = 1   SPI 使能                          │
└─────────────────────────────────────────────────────────────┘
```

回到 3.1 的问题：**从系统上电到 `/proc/interrupts` 中出现完整的三层中断控制器（GIC → EXTI → GPIO），内核经历了什么？**

| 阶段 | 做了什么 | 位于哪个文件 | init 级别 |
|------|---------|------------|----------|
| 编译期 | `IRQCHIP_DECLARE` 把 `gic_of_init` 地址放进 `__irqchip_of_table` 段 | `irq-gic.c` | 链接时 |
| head.S | 设置 `VBAR_EL1` → CPU 知道中断跳哪里 | `head.S` | 上电 |
| `init_IRQ()` | `of_irq_init()` 调用 `gic_of_init()` → GIC 全部就绪 | `irq-gic.c` | `start_kernel()` 内 |
| `do_initcalls()` | `stm32mp_exti_probe()` → EXTI 层级 domain + irq_chip | `irq-stm32mp-exti.c` | `.initcall6` |
| pinctrl probe | `stm32_pctrl_get_irq_domain()` → GPIO 层次域 | `pinctrl-stm32.c` | `.initcall6` |
| 设备 probe | `irq_create_of_mapping()` + `request_irq()` → 三层映射 + handler | `gpio_keys.c` | 设备驱动 probe |
