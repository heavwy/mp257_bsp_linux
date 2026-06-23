# 03. 初始化流程源码分析

> 本文是 STM32MP257 中断子系统深度分析系列的第 3 篇。
> 从源码层面逐行追踪中断子系统的初始化路径。
>
> **前置:** [02-Architecture.md](02-Architecture.md) — 熟悉核心结构体和数据结构关系
> **下一篇:** ——
>
> **字数：约 30,000 词（含代码段）**
> **建议阅读时间：70~90 分钟**

---

## 3.1 从一个问题开始

你在 ATK 板上设置好按键 PH5 对应的 gpio-keys 节点，加载内核后打开 retroarch 游戏。按一下按键，马里奥跳了起来。

问题是：**从系统上电到按下按键触发马里奥跳跃，内核里经历了什么？**

中断信号的物理路径是清晰的：

```
PH5 按键按下 → GPIOH bank → EXTI 边沿检测 → GIC 中断控制器 → CPU 异常入口
```

但这背后依赖四层软件基础设施在初始化时正确就位：

1. **CPU 异常向量表** — 发生中断时 CPU 跳到哪里
2. **GIC 中断控制器** — 根中断控制器，负责中断仲裁、优先级、路由
3. **EXTI 边沿检测** — GPIO 中断的入口，负责边沿检测、屏蔽、转发
4. **GPIO irq_domain** — 将 GPIO pin 号翻译为 Linux 中断号

**本文沿着初始化时间线，逐层追踪这些环节的代码路径。**

### 初始化流程全景

```
时间 →
       │
       │  [0] 编译期 — __irqchip_of_table 段静态布局
       │      所有 IRQCHIP_DECLARE 条目集中在 __irqchip_of_table 段
       │
       ▼
       │  [1] 上电 → head.S
       │      └─ msr vbar_el1, x0    ← 安装异常向量表
       │
       ▼
       │  [2] start_kernel()
       │      └─ init_IRQ()
       │           └─ irqchip_init()
       │                └─ of_irq_init(__irqchip_of_table)
       │                     │
       │                     ├─ 第 1 轮: parent=NULL
       │                     │   → gic_of_init()        ← GIC (根)
       │                     │     ├─ set_handle_irq()
       │                     │     ├─ gic_dist_init()
       │                     │     ├─ gic_cpu_init()
       │                     │     └─ irq_domain 创建
       │                     │
       │                     └─ (EXTI 不在 of_irq_init 中，
       │                         走 module_platform_driver)
       ▼
       │  [3] do_initcalls() → device_initcall
       │      └─ stm32mp_exti_probe()
       │           ├─ irq_domain 创建
       │           ├─ generic irq_chip 分配
       │           └─ irq_set_chained_handler() ← 链式串联
       ▼
       │  [4] pinctrl probe → stm32_pctl_probe()
       │      └─ stm32_pctrl_get_irq_domain()
       │           └─ irq_domain_create_hierarchy(exti_domain)
       ▼
       │  [5] 消费者设备 probe
       │      └─ driver_probe_device()
       │           ├─ irq_create_of_mapping()  ← 第一次引用时分配 virq
       │           │    └─ irq_domain_alloc_irqs()
       │           │         [穿越三层 domain: GPIO→EXTI→GIC]
       │           │
       │           └─ request_irq(virq, handler, ...)
       │                └─ irqaction 挂入 irq_desc.action 链表
       ▼
       全部就绪，等待中断
```


---

## 3.2 阶段零：ARM64 异常入口

中断子系统能工作的先决条件：**CPU 必须知道发生中断时跳到哪里**。这个"哪里"由异常向量表定义。本阶段在 `start_kernel()` 之前就已就绪——CPU 启动汇编 `head.S` 中设置了 `VBAR_EL1`。

### 3.2.1 向量表的定义

ARM64 的异常向量表定义在 `arch/arm64/kernel/entry.S`：

```asm
// arch/arm64/kernel/entry.S:529
.pushsection ".entry.text", "ax"
.align  11                                    // 2KB 对齐——ARMv8 硬性要求
SYM_CODE_START(vectors)
    kernel_ventry   1, t, 64, sync            // Synchronous EL1t
    kernel_ventry   1, t, 64, irq             // IRQ EL1t
    kernel_ventry   1, t, 64, fiq             // FIQ EL1t
    kernel_ventry   1, t, 64, error           // Error EL1t

    kernel_ventry   1, h, 64, sync            // Synchronous EL1h
    kernel_ventry   1, h, 64, irq             // IRQ EL1h   ← 内核态中断走这里
    kernel_ventry   1, h, 64, fiq             // FIQ EL1h
    kernel_ventry   1, h, 64, error           // Error EL1h

    kernel_ventry   0, t, 64, sync            // Synchronous 64-bit EL0
    kernel_ventry   0, t, 64, irq             // IRQ 64-bit EL0
    kernel_ventry   0, t, 64, fiq             // FIQ 64-bit EL0
    kernel_ventry   0, t, 64, error           // Error 64-bit EL0

    kernel_ventry   0, t, 32, sync            // Synchronous 32-bit EL0
    kernel_ventry   0, t, 32, irq             // IRQ 32-bit EL0
    kernel_ventry   0, t, 32, fiq             // FIQ 32-bit EL0
    kernel_ventry   0, t, 32, error           // Error 32-bit EL0
SYM_CODE_END(vectors)
```

**`.align 11`** — ARMv8 的 `VBAR_EL1` 寄存器低 11 位被硬件忽略，所以基地址必须 $2^{11}=2048$ 字节对齐。向量表 16 个条目 × 128 字节/条目 = 2048 字节，正好 2KB。

#### 向量表的硬件偏移

发生异常时，CPU 按异常类型在 `VBAR_EL1` 上加固定偏移：

```
偏移      异常
0x000     Synchronous EL1t (SP_EL0)
0x080     IRQ EL1t
0x100     FIQ EL1t
0x180     Error EL1t

0x200     Synchronous EL1h (SP_EL1)
0x280     IRQ EL1h          ← 内核态 IRQ 入口
0x300     FIQ EL1h
0x380     Error EL1h

0x400     Synchronous 64-bit EL0
0x480     IRQ 64-bit EL0
...
```

`kernel_ventry` 的 4 个参数：
- `el`: 0=EL0（用户态），1=EL1（内核态）
- `ht`: `h`=使用 SP_ELx，`t`=使用 SP_EL0
- `regsize`: 32 或 64 位
- `label`: sync/irq/fiq/error

### 3.2.2 kernel_ventry 宏：保存现场

```asm
.macro kernel_ventry, el, ht, regsize, label
    .align 7                                   // 128 字节对齐
.L\@:
    sub     sp, sp, #S_FRAME_SIZE              // S_FRAME_SIZE = 296
    stp     x0, x1, [sp, #16 * 0]
    stp     x2, x3, [sp, #16 * 1]
    // ... x4~x29
    stp     x30, xzr, [sp, #16 * 14]
    mrs     x30, elr_el1
    stp     x30, xzr, [sp, #S_FRAME_SIZE - 16]
    mrs     x30, spsr_el1
    stp     x30, xzr, [sp, #S_FRAME_SIZE - 32]
    mov     x0, sp
    bl      el1h_64_irq_handler                // ← 跳入 C 代码
.endm
```

#### pt_regs 栈帧布局

```
中断发生前的 SP
   │  sub sp, sp, #296
   ▼
┌──────────────────────┐ ← SP
│ pt_regs               │
│  ┌────────────────┐   │
│  │ x0, x1         │   │  0x000
│  │ x2, x3         │   │  0x010
│  │ ...            │   │
│  │ x28, x29       │   │  0x0E0
│  │ x30            │   │  0x0F0
│  │ sp_el0          │   │  0x100
│  │ pc (ELR_EL1)   │   │  0x110  ← 中断返回地址
│  │ pstate         │   │  0x120  ← 被中断时的 PSTATE
│  └────────────────┘   │
└──────────────────────┘ ← 中断前的 SP
```

### 3.2.3 VBAR_EL1 设置

向量表由链接器固定在 `.entry.text` 段，`head.S` 在 `start_kernel()` 之前设置 `VBAR_EL1`：

```asm
// arch/arm64/kernel/head.S
SYM_CODE_START(__primary_switched)
    adr_l   x8, vectors
    msr     vbar_el1, x8
    isb
```

从此任何 IRQ 到达 CPU 都会自动跳转到 `vectors[0x280]`。

### 3.2.4 C 入口：el1h_64_irq_handler → gic_handle_irq

```c
// arch/arm64/kernel/entry-common.c
asmlinkage void noinstr el1h_64_irq_handler(struct pt_regs *regs)
{
    el1_interrupt(regs, handle_arch_irq);
}
```

```c
static void noinstr el1_interrupt(struct pt_regs *regs,
                                  void (*handler)(struct pt_regs *))
{
    write_sysreg(DAIF_PROCCTX_NOIRQ, daif);   // 关 IRQ
    if (IS_ENABLED(CONFIG_ARM64_PSEUDO_NMI) && !interrupts_enabled(regs))
        __el1_pnmi(regs, handler);
    else
        __el1_irq(regs, handler);
}
```

`__el1_irq` 是中断核心入口：

```c
static __always_inline void __el1_irq(struct pt_regs *regs,
                                      void (*handler)(struct pt_regs *))
{
    enter_from_kernel_mode(regs);   // ① 通知 lockdep/RCU/trace：我进中断了

    irq_enter_rcu();
    do_interrupt_handler(regs, handler); // ② → gic_handle_irq(regs)
    irq_exit_rcu();

    arm64_preempt_schedule_irq();   // ③ 抢占检查

    exit_to_kernel_mode(regs);      // ④ 恢复
}
```

**`enter_from_kernel_mode`** 的三行簿记：

```c
lockdep_hardirqs_off(CALLER_ADDR0);   // lockdep：IRQ 已关
rcu_irq_enter_check_tick();            // RCU：进入读端临界区
trace_hardirqs_off_finish();           // ftrace：记录 IRQ 状态
```

| 行 | 通知对象 | 不做的后果 |
|----|---------|-----------|
| `lockdep_hardirqs_off()` | lockdep | 误判 spinlock 死锁 |
| `rcu_irq_enter_check_tick()` | RCU | RCU 误回收中断 handler 引用的数据 |
| `trace_hardirqs_off_finish()` | ftrace | trace 记录错误的 IRQ 状态 |

`handle_arch_irq` 是一个全局函数指针：

```c
// arch/arm64/kernel/irq.c:100
void (*handle_arch_irq)(struct pt_regs *) __ro_after_init = default_handle_irq;

static void default_handle_irq(struct pt_regs *regs)
{
    panic("IRQ taken without a root IRQ handler\n");
}
```

GIC 初始化前任何 IRQ 都会 panic。GIC 驱动的第一件事就是用 `set_handle_irq(gic_handle_irq)` 替换它。

---

## 3.3 中断控制器注册机制：IRQCHIP_DECLARE 与 of_irq_init

GIC 初始化不走标准平台驱动路径，而是通过 `IRQCHIP_DECLARE` 宏注册，在 `of_irq_init()` 中早期初始化。

### 3.3.1 IRQCHIP_DECLARE 宏完全展开

```c
// drivers/irqchip/irq-gic.c:1569
IRQCHIP_DECLARE(stm32mp2_cortex_a7_gic, "st,stm32mp2-cortex-a7-gic", gic_of_init);
```

**第一层展开**（`irqchip.h`）：

```c
#define IRQCHIP_DECLARE(name, compat, fn) \
    OF_DECLARE_2(irqchip, name, compat, typecheck_irq_init_cb(fn))
```

**第二层展开**（`of.h`）：

```c
#define OF_DECLARE_2(table, name, compat, fn)                    \
    static const struct of_device_id __of_table_##name           \
        __used __section("__irqchip_of_table")                   \
        = { .compatible = compat, .data = fn }
```

**完全展开后**，编译器生成：

```c
static const struct of_device_id __of_table_stm32mp2_cortex_a7_gic
    __used __attribute__((__section__("__irqchip_of_table")))
    = {
        .compatible = "st,stm32mp2-cortex-a7-gic",
        .data       = gic_of_init,    // 函数指针
    };
```

**关键：这不是函数调用，不产生执行代码。** 它只在 `.init.rodata` 的 `__irqchip_of_table` 段中放置一个 `of_device_id` 结构体。链接器将所有 `.o` 文件的同名段合并成连续数组。

`__irqchip_of_table` 段内容：

```
┌──────────────────────────────────────────────┐
│ .compatible = "arm,gic-400"                   │
│ .data       = gic_of_init                     │
├──────────────────────────────────────────────┤
│ .compatible = "arm,cortex-a7-gic"             │ ← 匹配这个
│ .data       = gic_of_init                     │
├──────────────────────────────────────────────┤
│ .compatible = "st,stm32mp2-cortex-a7-gic"    │ ← 或这个（ATK 板用）
│ .data       = gic_of_init                     │
├──────────────────────────────────────────────┤
│ ... 其他 IRQCHIP_DECLARE 条目 ...            │
├──────────────────────────────────────────────┤
│ .compatible = ""  (末端哨兵)                   │
│ .data       = NULL                           │
└──────────────────────────────────────────────┘
```

### 3.3.2 irqchip_init() — 触发入口

`start_kernel()` → `init_IRQ()` → `irqchip_init()`：

```c
// init/main.c:978
start_kernel()
    → ...
    → init_IRQ()           // ← 中断初始化
    → ...
    → mm_init()             // ← kmalloc 后端在这之后才就绪
    → ...
    → do_basic_setup()      // ← 普通驱动（EXTI 等）
```

```c
// arch/arm64/kernel/irq.c
void __init init_IRQ(void)
{
    init_irq_stacks();
    init_irq_scs();
    irqchip_init();          // ← 关键
}
```

```c
// drivers/irqchip/irqchip.c
void __init irqchip_init(void)
{
    of_irq_init(__irqchip_of_table);
    acpi_probe_device_table(irqchip);   // x86 相关，忽略
}
```

### 3.3.3 of_irq_init() — Parent-first BFS 拓扑排序

```c
void __init of_irq_init(const struct of_device_id *matches)
{
    LIST_HEAD(intc_desc_list);      // 待初始化
    LIST_HEAD(intc_parent_list);    // 已初始化暂存

    // 第 1 步：收集 DTS 中所有匹配的中断控制器
    for_each_matching_node_and_match(np, matches, &match) {
        if (!of_property_read_bool(np, "interrupt-controller"))
            continue;
        if (!of_device_is_available(np))
            continue;
        if (!match->data)
            continue;

        desc = kzalloc(sizeof(*desc), GFP_KERNEL);
        desc->irq_init_cb = match->data;
        desc->dev = of_node_get(np);

        // 确定 parent
        desc->interrupt_parent = of_parse_phandle(np, "interrupts-extended", 0);
        if (!desc->interrupt_parent)
            desc->interrupt_parent = of_irq_find_parent(np);
        if (desc->interrupt_parent == np) {
            of_node_put(desc->interrupt_parent);
            desc->interrupt_parent = NULL;   // 根
        }
        list_add_tail(&desc->list, &intc_desc_list);
    }

    // 第 2 步：BFS 分层初始化
    while (!list_empty(&intc_desc_list)) {
        // 处理所有 interrupt_parent == parent 的节点（同一层）
        list_for_each_entry_safe(desc, temp_desc, &intc_desc_list, list) {
            if (desc->interrupt_parent != parent)
                continue;
            list_del(&desc->list);
            of_node_set_flag(desc->dev, OF_POPULATED);
            ret = desc->irq_init_cb(desc->dev, desc->interrupt_parent);
            if (ret) {
                of_node_clear_flag(desc->dev, OF_POPULATED);
                kfree(desc);
                continue;
            }
            list_add_tail(&desc->list, &intc_parent_list);
        }
        // 取下一个已初始化为新的 parent
        desc = list_first_entry_or_null(&intc_parent_list, ...);
        if (!desc) break;
        list_del(&desc->list);
        parent = desc->dev;
        kfree(desc);
    }
}
```

**对 ATK 板，of_irq_init 只初始化了 GIC**（根节点）。EXTI 不是 IRQCHIP_DECLARE，在后续 `module_platform_driver` 中初始化。

### 3.3.4 GIC DTS 节点

```dts
intc: interrupt-controller@4ac00000 {
    compatible = "st,stm32mp2-cortex-a7-gic", "arm,cortex-a7-gic";
    reg = <0x0 0x4ac10000 0x0 0x1000>,     // [0] Distributor
          <0x0 0x4ac20000 0x0 0x2000>,     // [1] CPU Interface
          <0x0 0x4ac40000 0x0 0x2000>,     // [2] Virtual CPU
          <0x0 0x4ac60000 0x0 0x2000>;     // [3] Virtual Ctrl
};
```

| idx | 区 | 地址 | 长度 | 谁映射 |
|-----|-----|------|------|-------|
| 0 | Distributor | 0x4ac10000 | 4KB | GIC 驱动 |
| 1 | CPU Interface | 0x4ac20000 | 8KB | GIC 驱动 |
| 2 | Virtual CPU | 0x4ac40000 | 8KB | KVM（GIC 不映射） |
| 3 | Virtual Ctrl | 0x4ac60000 | 8KB | KVM（GIC 不映射） |

---

## 3.4 阶段一：GIC 初始化

这是真正的初始化起点。of_irq_init 调用 gic_of_init()，完成 GIC 的所有配置。

### 3.4.1 gic_data[] 静态数组

```c
static struct gic_chip_data gic_data[CONFIG_ARM_GIC_MAX_NR] __read_mostly;
static int gic_cnt __initdata;
```

GIC 初始化时不能用 kzalloc——kmalloc 后端（slab）在 `mm_init()` 之后才就绪，而 GIC 在 `init_IRQ()` 中初始化，远早于 `mm_init()`。所以使用编译期就分配好的静态数组。

`__read_mostly`：中断热路径每次中断都要读 gic_data[]（取 dist_base、cpu_base、domain），放在只读段中减少 cache miss。

`gic_data[]` 支持多 GIC 级联（`CONFIG_ARM_GIC_MAX_NR` 默认为 1）。STM32MP257 只有一个 GIC，`gic_cnt` 始终为 0。

### 3.4.2 gic_of_init() — 入口

```c
// drivers/irqchip/irq-gic.c:1513
static int __init gic_of_init(struct device_node *node,
                               struct device_node *parent)
{
    struct gic_chip_data *gic;
    int ret;

    gic = &gic_data[gic_cnt];        // 取静态数组槽位

    // ① 寄存器映射
    ret = gic_of_setup(gic, node);
    if (ret) return ret;

    // ② 检查 EOI 模式
    if (gic_cnt == 0 && !gic_check_eoimode(node, &gic->raw_cpu_base))
        static_branch_disable(&supports_deactivate_key);

    // ③ 核心初始化：set_handle_irq + 硬件配置 + irq_domain
    ret = __gic_init_bases(gic, &node->fwnode);
    if (ret) return ret;

    // ④ 第一颗 GIC 才需要：KVM 支持 + MSI
    if (!gic_cnt) {
        gic_init_physaddr(node);
        gic_of_setup_kvm_info(node);
    }

    if (IS_ENABLED(CONFIG_ARM_GIC_V2M))
        gicv2m_init(&node->fwnode, gic_data[gic_cnt].domain);  // MSI

    gic_cnt++;
    return 0;
}
```

### 3.4.3 gic_of_setup() — 寄存器映射

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

    // 厂商 quirks
    gic_enable_of_quirks(node, gic_quirks, gic);
    return 0;

error:
    gic_teardown(gic);
    return -ENOMEM;
}
```

**of_iomap(node, index) 内部**：读 DTS reg 属性的第 index 个地址/大小对，调用 ioremap。等价于 `of_address_to_resource(node, index, &res)` + `ioremap(res.start, resource_size(&res))`。

ioremap 后内核虚拟地址空间中的布局：

```
─────────────────────────────────────
  gic->raw_dist_base   → 0x4ac10000  (Distributor, 4KB)
  gic->raw_cpu_base    → 0x4ac20000  (CPU Interface, 8KB)
  gic->cpu_phys_base   = 0x4ac20000  (KVM 用物理地址)
─────────────────────────────────────
```

reg[2] 和 reg[3]（Virtual CPU / Virtual Ctrl）不由 `gic_of_setup` 映射。它们仅在 KVM 路径中由 `gic_of_setup_kvm_info()` 提取物理地址：

```c
static void __init gic_of_setup_kvm_info(struct device_node *node)
{
    // reg[2]: VCTRL → gic_v2_kvm_info.vctrl
    of_address_to_resource(node, 2, &gic_v2_kvm_info.vctrl);
    // reg[3]: VCPU  → gic_v2_kvm_info.vcpu
    of_address_to_resource(node, 3, &gic_v2_kvm_info.vcpu);
}
```

### 3.4.4 __gic_init_bases() — 安装根 handler + 启动硬件

```c
// irq-gic.c:1249
static int __init __gic_init_bases(struct gic_chip_data *gic,
                                    struct fwnode_handle *handle)
{
    // === 关键：安装顶层 IRQ handler（仅第一次）===
    if (gic == &gic_data[0]) {
        set_handle_irq(gic_handle_irq);
        if (static_branch_likely(&supports_deactivate_key))
            pr_info("GIC: Using split EOI/Deactivate mode\n");
    }

    ret = gic_init_bases(gic, handle);   // 硬件初始化 + irq_domain
    if (gic == &gic_data[0])
        gic_smp_init();                   // IPI 支持

    return ret;
}
```

**`set_handle_irq(gic_handle_irq)`** — 用 `gic_handle_irq` 替换 `default_handle_irq`：

```c
// arch/arm64/kernel/irq.c:103
int __init set_handle_irq(void (*handle_irq)(struct pt_regs *))
{
    if (handle_arch_irq != default_handle_irq)
        return -EBUSY;          // 防止重复设置——只能装一次
    handle_arch_irq = handle_irq;
    return 0;
}
```

在这之前任何中断都会 panic。之后 CPU 的 IRQ 信号有了真正的入口。

### 3.4.5 gic_init_bases() — 硬件初始化 + irq_domain 创建

```c
// irq-gic.c:1175
static int __init gic_init_bases(struct gic_chip_data *gic,
                                  struct fwnode_handle *handle)
{
    int gic_irqs, ret;

    // ① 读 GIC_DIST_CTR 寄存器——确定硬件支持的中断数
    gic_irqs = readl_relaxed(gic_data_dist_base(gic) + GIC_DIST_CTR) & 0x1f;
    gic_irqs = (gic_irqs + 1) * 32;
    if (gic_irqs > 1020)
        gic_irqs = 1020;

    // ② 创建 irq_domain
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

#### 3.4.5.1 GIC_DIST_CTR 寄存器

```c
#define GIC_DIST_CTR   0x0004   // Distributor 基址 + 0x04
```

`GIC_DIST_CTR` 的 `bit[4:0]` 表示支持的中断线组数（每组 32 个），其值 = (组数 - 1)：

```
寄存器: 0x4ac10004
bit[4:0] = 0x1f → 32 组 → 32 × 32 = 1024 个中断 → 上限 1020
```

STM32MP257 的 GIC-400 通常返回 0x1f，代表 32 组。GICv2 规范规定中断 ID 最大为 1019（1020~1023 保留），所以取 min(1024, 1020) = 1020。

中断 ID 范围：

| ID 范围 | 类型 | 数量 | 用途 |
|---------|------|------|------|
| 0~15 | SGI | 16 | 核间中断（IPI） |
| 16~31 | PPI | 16 | 每核私有中断（Timer, PMU） |
| 32~1019 | SPI | 988 | 外设共享中断 |
| 1020~1023 | — | 4 | 保留 |

#### 3.4.5.2 irq_domain_create_linear() — 线性映射域

```c
gic->domain = irq_domain_create_linear(handle, gic_irqs,
                                        &gic_irq_domain_hierarchy_ops,
                                        gic);
```

展开为：

```c
// include/linux/irqdomain.h:375
static inline struct irq_domain *irq_domain_create_linear(
    struct fwnode_handle *fwnode, unsigned int size,
    const struct irq_domain_ops *ops, void *host_data)
{
    return __irq_domain_add(fwnode, size, size, 0, ops, host_data);
}
```

**`__irq_domain_add()`** 的分配：

```c
struct irq_domain *__irq_domain_add(struct fwnode_handle *fwnode,
                                     unsigned int size,   // = gic_irqs = 1020
                                     irq_hw_number_t hwirq_max,  // = 1020
                                     int direct_max,             // = 0
                                     const struct irq_domain_ops *ops,
                                     void *host_data)            // = gic_data[0]
{
    // 分配 domain + revmap 数组（一次 kzalloc，连续内存）
    domain = kzalloc_node(struct_size(domain, revmap, size), GFP_KERNEL, ...);
    // struct_size(domain, revmap, 1020)
    // = sizeof(struct irq_domain) + 1020 * sizeof(struct irq_data *)

    // 初始化
    domain->ops         = ops;          // → gic_irq_domain_hierarchy_ops
    domain->host_data   = host_data;    // → gic_data[0]
    domain->hwirq_max   = hwirq_max;    // → 1020
    domain->revmap_size = size;         // → 1020 ← 关键：决定了走数组还是 radix tree
    domain->fwnode      = fwnode;

    // 挂入全局链表 irq_domain_list（使 irq_find_host() 能查到）
    list_add(&domain->link, &irq_domain_list);
    debugfs_add_domain_dir(domain);

    return domain;
}
```

**struct_size 宏的展开**：

```c
struct_size(domain, revmap, 1020)
    = sizeof(struct irq_domain) + sizeof(struct irq_data *) * 1020
    ≈ 120 + 8 * 1020 = 8280 字节
```

**`struct irq_domain` 的 revmap 字段**：

```c
struct irq_domain {
    ...
    unsigned int            revmap_size;       // = 1020
    struct radix_tree_root  revmap_tree;       // 空 radix tree（本 domain 不用）
    struct irq_data __rcu   *revmap[];         // 柔性数组，末尾对齐
};
```

**分配后 revmap 数组的内容**（全是 NULL——此时没有任何中断被映射）：

```
revmap[0]      = NULL   ← hwirq  0  (SGI #0)
revmap[1]      = NULL   ← hwirq  1  (SGI #1)
...
revmap[15]     = NULL   ← hwirq 15  (SGI #15)
revmap[16]     = NULL   ← hwirq 16  (PPI #0)
...
revmap[31]     = NULL   ← hwirq 31  (PPI #15)
revmap[32]     = NULL   ← hwirq 32  (SPI #0)
...
revmap[273]    = NULL   ← hwirq 273 (EXTI 中断，将来填充)
...
revmap[1019]   = NULL   ← hwirq 1019
```

**这张空网什么时候被填充？** 当设备第一次引用中断时，`irq_create_of_mapping()` → `irq_domain_alloc_irqs()` → `irq_domain_insert_irq()` 执行：

```c
static void irq_domain_insert_irq(int virq)
{
    struct irq_data *data;
    for (data = irq_get_irq_data(virq); data; data = data->parent_data) {
        if (data->hwirq < data->domain->revmap_size)
            rcu_assign_pointer(data->domain->revmap[data->hwirq], data);
        // 之后 irq_find_mapping(domain, hwirq) 直接返回 revmap[hwirq]
    }
}
```

**为什么叫"线性"映射？** revmap 是一个数组，`hwirq` 就是数组下标。查找是 O(1) 的：`irq_find_mapping(domain, hwirq) = domain->revmap[hwirq]`。对立面是 radix tree 的域（`irq_domain_create_tree`），用于 hwirq 稀疏的场景（如 MSI），以节省内存，但查找是 O(log n)。

**GIC 为什么用线性域？** GIC 的 hwirq 范围是连续的 0~1019，几乎每个号都可能被用到（虽然实际用的不多，但空间是连续的）。用数组正好一一对应，查找零开销。如果改用树形域，1020 个 slot 的 radix tree 反而更慢更复杂。

#### 3.4.5.3 irq_domain_ops — 驱动必须提供的翻译回调

```c
static const struct irq_domain_ops gic_irq_domain_hierarchy_ops = {
    .translate = gic_irq_domain_translate,    // DTS cell → (hwirq, type)
    .alloc     = gic_irq_domain_alloc,         // 分配 virq + 设置 flow handler
    .free      = irq_domain_free_irqs_top,
};
```

**`.translate`** — 将 DTS 的 `interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>` 三单元组翻译为 (hwirq, type)：

```c
// irq-gic.c:1092
static int gic_irq_domain_translate(struct irq_domain *d,
                                     struct irq_fwspec *fwspec,
                                     unsigned long *hwirq,
                                     unsigned int *type)
{
    if (is_of_node(fwspec->fwnode)) {
        if (fwspec->param_count < 3)
            return -EINVAL;

        switch (fwspec->param[0]) {
        case 0:  // GIC_SPI
            *hwirq = fwspec->param[1] + 32;   // DTS SPI 0 → hwirq 32
            break;
        case 1:  // GIC_PPI
            *hwirq = fwspec->param[1] + 16;   // DTS PPI 0 → hwirq 16
            break;
        default:
            return -EINVAL;
        }
        *type = fwspec->param[2] & IRQ_TYPE_SENSE_MASK;
        return 0;
    }
    // ... ACPI 路径
}
```

翻译结果：

| DTS 写法 | hwirq |
|----------|-------|
| `GIC_SPI 0` | 32 |
| `GIC_SPI 42` | 74 |
| `GIC_SPI 268` | 300 |
| `GIC_PPI 9` | 25 |

**为什么 SPI 从 32 开始？** GIC 硬件把 ID 0~31 固定给了 SGI 和 PPI，SPI 自然从 32 开始排。DTS 约定 SPI 编号从 0 写起（这样看到 `GIC_SPI 42` 就知道是"第 42 个 SPI"），translate 负责加 32。

### 3.4.6 gic_dist_init() — Distributor 硬件初始化

```c
// irq-gic.c:475
static void gic_dist_init(struct gic_chip_data *gic)
{
    unsigned int i;
    u32 cpumask;
    unsigned int gic_irqs = gic->gic_irqs;
    void __iomem *base = gic_data_dist_base(gic);

    // ① 全局关闭 Distributor
    writel_relaxed(GICD_DISABLE, base + GIC_DIST_CTRL);

    // ② 所有 SPI 路由到 CPU0
    cpumask = gic_get_cpumask(gic);
    cpumask |= cpumask << 8;
    cpumask |= cpumask << 16;
    for (i = 32; i < gic_irqs; i += 4)
        writel_relaxed(cpumask, base + GIC_DIST_TARGET + i * 4 / 4);

    // ③ 配置优先级和触发类型
    gic_dist_config(base, gic_irqs, NULL);

    // ④ 全局开启 Distributor
    writel_relaxed(GICD_ENABLE, base + GIC_DIST_CTRL);
}
```

#### ① GIC_DIST_CTRL (0x0000)

```c
#define GICD_DISABLE    0
#define GICD_ENABLE     1
```

写 0 → 所有中断不转发到 CPU Interface；写 1 → 重新使能。

#### ② GIC_DIST_TARGET (0x8000) — 路由

每个 SPI 有 8 位 CPU mask（实际只用低 4 位）：

```
GIC_DIST_TARGET + 0x00: IRQ 32~35 的 CPU mask
  byte[0] = CPU mask for IRQ 32
  byte[1] = CPU mask for IRQ 33
  byte[2] = CPU mask for IRQ 34
  byte[3] = CPU mask for IRQ 35
```

`cpumask = 0x01`（仅 CPU0），重复到 32 位：`0x01010101`。循环写入每组 4 个 SPI：
- `DIST_TARGET[32..35] = 0x01010101`
- `DIST_TARGET[36..39] = 0x01010101`
- ...

所有 SPI 默认路由到 CPU0。

#### ③ gic_dist_config() — 优先级 + 触发类型 + 初始禁用

```c
static void gic_dist_config(void __iomem *base, int gic_irqs, ...)
{
    unsigned int i;

    // 优先级寄存器 GIC_DIST_PRI (0x400)
    // 每 4 个 SPI 共享一个 32 位寄存器
    for (i = 32; i < gic_irqs; i += 4)
        writel_relaxed(0xa0a0a0a0, base + GIC_DIST_PRI + i);

    // 触发类型寄存器 GIC_DIST_CONFIG (0xC00)
    // bit[1:0] 控制一个 SPI: 0=电平, 1=边沿
    for (i = 32; i < gic_irqs; i += 16)
        writel_relaxed(0, base + GIC_DIST_CONFIG + i);

    // 禁用所有 SPI（写 GIC_DIST_ENABLE_CLEAR）
    for (i = 32; i < gic_irqs; i += 4)
        writel_relaxed(0, base + GIC_DIST_ENABLE + i);
}
```

寄存器详细：

| 寄存器 | 偏移 | 每个 SPI 的位域 |
|--------|------|----------------|
| GIC_DIST_PRI | 0x400 + (irq * 4 / 4) | byte[0..3]：IRQ N..N+3 的优先级，0xa0 表示非安全世界默认值 |
| GIC_DIST_CONFIG | 0xC00 + (irq * 4 / 16) | bit[1:0]：00=电平触发，01=边沿触发（每 16 个 IRQ 共享） |
| GIC_DIST_ENABLE_CLEAR | 0x180 + (irq * 4 / 32) | 每位对应一个 IRQ，写 1 禁用 |

**优先级 0xa0**：ARM 优先级数值越小优先级越高，0x00 最高，0xFF 最低。0xa0 是 Linux 给非安全 SPI 的默认值。

### 3.4.7 gic_cpu_init() — CPU Interface 初始化

```c
// irq-gic.c:498
static int gic_cpu_init(struct gic_chip_data *gic)
{
    void __iomem *dist_base = gic_data_dist_base(gic);
    void __iomem *base = gic_data_cpu_base(gic);

    if (gic == &gic_data[0]) {
        // 更新 gic_cpu_map[当前 CPU] = CPU mask
        gic_check_cpu_features();
        cpu_mask = gic_get_cpumask(gic);
        gic_cpu_map[cpu] = cpu_mask;
    }

    // 配置 PPI（IRQ 16~31）的优先级
    gic_cpu_config(dist_base, 32, NULL);

    // 设置优先级阈值寄存器
    writel_relaxed(GICC_INT_PRI_THRESHOLD, base + GIC_CPU_PRIMASK);

    // 使能 CPU Interface
    writel_relaxed(GICC_ENABLE, base + GIC_CPU_CTRL);
    return 0;
}
```

**GICC_PMR (0x0004)** — 优先级掩码寄存器：

```c
#define GICC_INT_PRI_THRESHOLD  0xf0
// PMR[7:3]：优先级阈值
// 只有优先级高于此值（即数值更小）的中断才能到达 CPU
// 0xf0 = 允许所有优先级 < 0xf0 的中断
```

**GICC_CTRL (0x0000)** — CPU Interface 控制寄存器：

```c
#define GICC_ENABLE     1        // bit[0]: 使能 CPU Interface
// bit[9]: 如果支持，启用 EOImode=1（split EOI/Deactivate）
```

### 3.4.8 GIC 初始化完成后的状态

```
┌─────────────────────────────────────────────────┐
│ handle_arch_irq = gic_handle_irq                  │
├─────────────────────────────────────────────────┤
│ gic_data[0]:                                      │
│   .domain      → irq_domain (linear, 1020 items) │
│   .raw_dist_base → 0x4ac10000 (ioremap'd)         │
│   .raw_cpu_base  → 0x4ac20000 (ioremap'd)         │
│   .gic_irqs     → 1020                            │
├─────────────────────────────────────────────────┤
│ Distributor 硬件：                                 │
│   GICD_CTLR        = ENABLE                        │
│   GICD_ISENABLER   = 0（所有中断禁用）              │
│   GICD_ITARGETSR   = 0x01010101...（全路由 CPU0）   │
│   GICD_IPRIORITYR  = 0xa0...（全优先级 0xa0）       │
├─────────────────────────────────────────────────┤
│ CPU Interface 硬件：                               │
│   GICC_CTLR        = ENABLE                        │
│   GICC_PMR         = 0xf0                          │
├─────────────────────────────────────────────────┤
│ irq_domain:                                       │
│   revmap[] 1020 个 NULL                           │
│   ops = { .translate, .alloc, .free }             │
│   host_data = gic_data[0]                         │
└─────────────────────────────────────────────────┘
```

此时 GIC 全部就绪。如果有外设中断到来，CPU 可以正确响应（不会 panic），但因为 revmap 全空且所有中断硬件禁用，中断号无法翻译，中断被 GIC 忽略。

### 3.4.9 gic_handle_irq() — 中断发生时 GIC 的入口

先提前分析，因为中断发生时的第一条 C 代码就是它：

```c
// irq-gic.c:362
static asmlinkage void __exception_irq_entry gic_handle_irq(struct pt_regs *regs)
{
    u32 irqstat, irqnr;

    do {
        // ① 读 GICC_IAR 获取当前最高优先级的中断 ID
        irqstat = readl_relaxed(gic_cpu_base + GIC_CPU_INTACK);
        irqnr = irqstat & ~0x1c00;          // bit[9:0] = 中断 ID

        if (likely(irqnr > 15 && irqnr < 1020)) {
            // ② 通过 irq_find_mapping 找 virq
            int irq = irq_find_mapping(gic->domain, irqnr);
            if (likely(irq))
                generic_handle_irq(irq);    // ③ 调 flow handler
        } else if (irqnr < 16) {
            // SGI（核间中断 → IPI）
            handle_IPI(irqnr, regs);
        }
        // ④ 写 GICC_EOIR 确认
    } while (irqnr != 1023);  // 1023 = "没有更多中断"
}
```

**GICC_IAR (0x000c)**：读操作返回当前优先级最高的待处理中断 ID。

数据流：

```
GICC_IAR → hwirq → irq_find_mapping(gic_domain, hwirq)
  → revmap[hwirq] → irq_data → irq_to_desc(virq) → irq_desc
  → irq_desc->handle_irq(desc) → flow handler
    → (如果是链式) mask/ack → 细分 → 调子域 handler
    → (如果是最终设备) mask/ack → 调 action->handler
    → eoi (写 GICC_EOIR)
```

---

## 3.5 阶段二：EXTI 初始化

GIC 初始化完成后，`handle_arch_irq` 就位、GIC 硬件使能。但 GIC 只知道原始中断号——它不知道一个 SPI 对应的是 GPIO 还是 TIM。对于 GPIO 中断，中间有一个 **EXTI（Extended Interrupt Controller）** 负责边沿检测和转发。

EXTI 的初始化机制与 GIC 不同——它走标准平台驱动路径，不在 `of_irq_init()` 中。

### 3.5.1 注册机制：module_platform_driver

```c
// drivers/irqchip/irq-stm32mp-exti.c:980
static struct platform_driver stm32mp_exti_driver = {
    .probe      = stm32mp_exti_probe,
    .driver     = {
        .name           = "stm32mp_exti",
        .of_match_table = stm32mp_exti_ids,
        .pm             = &stm32mp_exti_dev_pm_ops,
    },
};
module_platform_driver(stm32mp_exti_driver);
```

`module_platform_driver` 展开为 `module_init` → `.initcall6.init`（`device_initcall`，优先级 6）：

```c
// 编译器生成（等价于）：
static int __init stm32mp_exti_init(void)
    __attribute__((__section__(".initcall6.init")));
static int __init stm32mp_exti_init(void)
{
    return platform_driver_register(&stm32mp_exti_driver);
}
```

**对比 GIC 和 EXTI 的初始化时机：**

| 中断控制器 | 注册机制 | 执行阶段 | 调用路径 |
|-----------|---------|---------|---------|
| GIC | `IRQCHIP_DECLARE` → `of_irq_init()` | `start_kernel()` 内部（**最早**） | `init/main.c` → `init_IRQ()` → `irqchip_init()` |
| EXTI | `module_platform_driver` | `do_initcalls()` 级别 6 | `do_basic_setup()` → `do_initcalls()` → 级别 6 |

GIC 在 `start_kernel()` 内部就绪，EXTI 在 `do_initcalls()` 中才初始化——远晚于 GIC。这是合理的，因为 EXTI 的初始化需要 GIC 的 irq_domain 已经存在（EXTI 需要将它的 SPI 中断号注册到 GIC domain 中）。

匹配表：

```c
static const struct of_device_id stm32mp_exti_ids[] = {
    { .compatible = "st,stm32mp1-exti", .data = &stm32mp1_drv_data},
    { .compatible = "st,stm32mp13-exti", .data = &stm32mp13_drv_data},
    {},
};
```

ATK 板使用 `"st,stm32mp1-exti"`。

### 3.5.2 EXTI DTS 节点

```dts
// stm32mp251.dtsi
exti1: interrupt-controller@44220000 {
    compatible = "st,stm32mp1-exti";
    interrupt-controller;
    #interrupt-cells = <2>;
    reg = <0x44220000 0x400>;
    interrupts-extended =
        <&intc GIC_SPI 268 IRQ_TYPE_LEVEL_HIGH>,   // EXTI_0
        <&intc GIC_SPI 269 IRQ_TYPE_LEVEL_HIGH>,   // EXTI_1
        ...
        <&intc GIC_SPI 277 IRQ_TYPE_LEVEL_HIGH>;   // EXTI_9
};

exti2: interrupt-controller@46230000 {
    // 安全域 EXTI，ATK 板不用于 GPIO 中断
};
```

两个 EXTI 实例：`exti1`（主域，`0x44220000`）和 `exti2`（安全域，`0x46230000`）。GPIO 中断走 `exti1`，位于低功耗域（`ret_pd`），支持唤醒。

**`interrupts-extended`** 中的每一行对应 EXTI 输出到 GIC 的一个 SPI 中断。EXTI 本身有 64 个内部中断线（`IRQS_PER_BANK × bank_nr`），但只有 10 根输出线连到了 GIC（SPI 268~277）。这意味着 EXTI 内部有 64 个 possible 中断源，但在硬件上最多 10 个能同时上报给 GIC。

### 3.5.3 stm32mp_exti_probe() 入口

```c
// irq-stm32mp-exti.c:895
static int stm32mp_exti_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct stm32mp_exti_host_data *host_data;
    const struct stm32mp_exti_drv_data *drv_data;

    drv_data = of_device_get_match_data(dev);
    if (!drv_data) return -EINVAL;

    // ① 主结构体分配 + 寄存器映射
    host_data = devm_kzalloc(dev, sizeof(*host_data), GFP_KERNEL);
    host_data->dev = dev;
    host_data->drv_data = drv_data;
    host_data->base = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(host_data->base))
        return PTR_ERR(host_data->base);

    // ② 硬件初始化
    host_data->hwspinlock = ...;  // 硬件自旋锁（可选）
    if (hwlock_id >= 0)
        host_data->hwlock = hwspin_lock_request_specific(hwlock_id);

    // 读取 HWCFGR1 确认硬件特性
    hwcfgr1 = readl_relaxed(host_data->base + EXTI_HWCFGR1);

    // ③ 创建 irq_domain
    host_data->domain = irq_domain_add_linear(dev->of_node,
                                               drv_data->bank_nr * IRQS_PER_BANK,
                                               &stm32mp_exti_domain_ops,
                                               NULL);
    // ④ 分配 generic irq_chip
    ret = irq_alloc_domain_generic_chips(host_data->domain, IRQS_PER_BANK,
                                          1, "exti", handle_edge_irq,
                                          clr, 0, 0);
    // ⑤ 初始化每个 EXTI bank
    for (i = 0; i < drv_data->bank_nr; i++)
        stm32mp_exti_chip_init(host_data, i, np);

    // ⑥ 安装链式 handler
    nr_irqs = of_irq_count(node);
    for (i = 0; i < nr_irqs; i++) {
        unsigned int irq = irq_of_parse_and_map(node, i);
        irq_set_handler_data(irq, host_data->domain);
        irq_set_chained_handler(irq, stm32_irq_handler);
    }

    return 0;
}
```

### 3.5.4 EXTI irq_domain

```c
host_data->domain = irq_domain_add_linear(dev->of_node,
                                           drv_data->bank_nr * IRQS_PER_BANK,
                                           &stm32mp_exti_domain_ops,
                                           NULL);
```

`IRQS_PER_BANK = 32`，`bank_nr` 从驱动数据中读取（STM32MP1 为 2 个 bank，共 64 个中断线）。

与 GIC 的 domain ops 不同，EXTI 使用：

```c
static const struct irq_domain_ops stm32mp_exti_domain_ops = {
    .alloc  = stm32mp_exti_alloc,
    .free   = stm32mp_exti_free,
    .xlate  = irq_domain_xlate_twocell,
};
```

**`irq_domain_xlate_twocell`** — 标准的双 cell 翻译（`#interrupt-cells = <2>`）：

```c
int irq_domain_xlate_twocell(struct irq_domain *d, struct device_node *ctrlr,
                              const u32 *intspec, unsigned int intsize,
                              irq_hw_number_t *out_hwirq, unsigned int *out_type)
{
    *out_hwirq = intspec[0];    // 第一个 cell = EXTI line 编号
    *out_type  = intspec[1];    // 第二个 cell = 触发类型
    return 0;
}
```

### 3.5.5 Generic irq_chip 配置

`irq_alloc_domain_generic_chips()` 为核心层提供的通用 irq_chip 分配函数，避免了每个驱动重复编写 mask/unmask 回调。后续配置：

```c
gc = irq_get_domain_generic_chip(domain, i * IRQS_PER_BANK);

gc->reg_base = host_data->base;                         // EXTI 寄存器基址
gc->chip_types->chip.irq_ack    = stm32_irq_ack;        // ACK：清挂起位
gc->chip_types->chip.irq_mask   = irq_gc_mask_clr_bit;  // 写 IMR 清位
gc->chip_types->chip.irq_unmask = irq_gc_mask_set_bit;  // 写 IMR 置位
gc->chip_types->chip.irq_set_type = stm32_irq_set_type; // 配 RTSR/FTSR
gc->chip_types->chip.irq_set_wake  = irq_gc_set_wake;
gc->chip_types->regs.mask = stm32_bank->imr_ofst;       // IMR 偏移
```

**`irq_gc_mask_clr_bit`** 内部：

```c
void irq_gc_mask_clr_bit(struct irq_data *d)
{
    struct irq_chip_generic *gc = irq_data_get_irq_chip_data(d);
    irq_gc_lock(gc);
    u32 val = irq_reg_readl(gc, gc->chip_types->regs.mask);
    val &= ~d->mask;        // 清对应位 → 屏蔽中断
    irq_reg_writel(gc, val, gc->chip_types->regs.mask);
    irq_gc_unlock(gc);
}
```

`irq_gc_mask_set_bit` 同理，将 `val &= ~mask` 改为 `val |= mask` 即可。

### 3.5.6 EXTI 寄存器布局

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

以 PH5（EXTI line 5）为例，按键触发时的寄存器配置：

```
EXTI_CR[5] (0x060 + 5/4*4 = 0x064)  bit[7:0] = 0x07
  └─ 选择 GPIOH 作为输入源
     （0x00=GPIOA, 0x01=GPIOB, ..., 0x07=GPIOH）

EXTI_RTSR (0x0C0)                    bit[5] = 1
  └─ 使能上升沿触发检测

EXTI_FTSR (0x0E0)                    bit[5] = 1
  └─ 使能下降沿触发检测

EXTI_IMR  (0x080)                    bit[5] = 1
  └─ 取消屏蔽（允许中断通过到 GIC）

触发后的挂起：
  EXTI_RPR (0x100)  bit[5] = 1  ← 上升沿挂起
  EXTI_FPR (0x120)  bit[5] = 1  ← 下降沿挂起
```

### 3.5.7 链式 Handler — stm32_irq_handler()

EXTI 使用**链式模型**：EXTI 的 GIC SPI handler 负责"分发"，在 GIC 的 flow handler 内嵌套执行：

```c
// irq-stm32mp-exti.c:118
static void stm32_irq_handler(struct irq_desc *desc)
{
    struct irq_domain *domain = irq_desc_get_handler_data(desc);
    struct irq_chip *chip = irq_desc_get_chip(desc);
    unsigned long pending;
    int n;

    // ① 锁住 GIC 层（防止嵌套）
    chained_irq_enter(chip, desc);

    // ② 读取挂起寄存器
    pending = stm32_exti_pending(gc);

    // ③ 遍历所有挂起的 EXTI line
    for_each_set_bit(n, &pending, IRQS_PER_BANK) {
        unsigned int irq = irq_find_mapping(domain, irq_base + n);
        generic_handle_irq(irq);
    }

    // ④ 解锁 GIC 层
    chained_irq_exit(chip, desc);
}
```

完整的中断嵌套路径：

```
GIC Distributor 收到 SPI#268+n
  → CPU 响应用户 IRQ
  → el1h_64_irq_handler → __el1_irq
    → gic_handle_irq()
      → 读 GICC_IAR → hwirq = 268 + n
      → irq_find_mapping(gic_domain, hwirq) → virq
      → irq_desc[virq].handle_irq = handle_fasteoi_irq
        → mask_irq()           ← GICD_ICENABLER
        → action->handler()    ← stm32_irq_handler()
          → chained_irq_enter()  ← 防止嵌套
          → 读 EXTI_RPR | EXTI_FPR
          → for_each_set_bit:
              irq_find_mapping(exti_domain, line) → virq2
              generic_handle_irq(virq2)
          → chained_irq_exit()
        → eoi_irq()             ← GICC_EOIR
```

**链式模型的特征**：EXTI 的 GIC SPI 对应一个 irq_desc，handle_irq 是 `handle_fasteoi_irq`，action 链表中是 `stm32_irq_handler`。这个 handler 读取 EXTI 内部寄存器确定是哪个 line 触发了，再通过 `generic_handle_irq` 调用子域（GPIO）的 handler。两层 irq_desc，两次 flow handler 调用。

### 3.5.8 EXTI 初始化完成后的状态

```
┌──────────────────────────────────────────────┐
│ exti1 (0x44220000):                            │
│   .domain    → irq_domain (linear, 64 items)  │
│   .base      → 0x44220000 (ioremap'd)          │
├──────────────────────────────────────────────┤
│ Generic irq_chip (bank0, 32 lines):            │
│   .irq_ack    = stm32_irq_ack                  │
│   .irq_mask   = irq_gc_mask_clr_bit            │
│   .irq_unmask = irq_gc_mask_set_bit            │
│   .irq_set_type = stm32_irq_set_type           │
│   flow handler = handle_edge_irq               │
├──────────────────────────────────────────────┤
│ 链式 handler 已安装：                           │
│   GIC SPI 268 → stm32_irq_handler              │
│   GIC SPI 269 → stm32_irq_handler              │
│   ... (SPI 277)                                │
└──────────────────────────────────────────────┘
```

---

## 3.6 阶段三：GPIO 中断 domain

这是初始化路径的第三层——GPIO bank 为每个 pin 创建层次化 irq_domain，挂在 EXTI domain 之下。

### 3.6.1 注册时机：pinctrl probe 内部

GPIO 中断 domain 不单独占用一个 initcall 阶段，而是在 pinctrl 驱动 probe 中完成：

```c
// pinctrl-stm32.c:2033
int stm32_pctl_probe(struct platform_device *pdev)
{
    // ① 创建 IRQ domain（必须在 pinctrl 注册之前）
    pctl->domain = stm32_pctrl_get_irq_domain(pctl);
    if (IS_ERR(pctl->domain))
        return PTR_ERR(pctl->domain);

    // ② 注册 pinctrl + GPIO bank（gpiochip_add_data 需要 irq_domain）
    ...
}
```

这个步骤必须早于 `gpiochip_add_data()`，因为 GPIO chip 注册时需要 irq_domain 已经就绪来支持中断请求。

### 3.6.2 stm32_pctrl_get_irq_domain() — 层次化 domain

```c
// pinctrl-stm32.c:1823
static struct irq_domain *stm32_pctrl_get_irq_domain(struct stm32_pinctrl *pctl)
{
    struct device_node *np;
    struct irq_domain *domain;
    struct of_phandle_args args;

    for_each_gpiochip_node(np) {
        // 解析 interrupts-extended 属性找到 parent domain（EXTI）
        // gpioa: gpio@44240000 {
        //     interrupts-extended = <&exti1 1 IRQ_TYPE_LEVEL_HIGH>;
        // };

        // 找 parent domain
        of_parse_phandle_with_args(np, "interrupts-extended",
                                   "#interrupt-cells", 0, &args);
        parent = irq_find_host(args.np);
        if (!parent) {
            // 如果 EXTI 还没初始化，返回 EPROBE_DEFER
            return ERR_PTR(-EPROBE_DEFER);
        }

        // 创建层次化 domain
        domain = irq_domain_create_hierarchy(parent, 0,
                                              STM32_GPIO_IRQ_LINE,  // 16
                                              fwnode,
                                              &stm32_gpio_domain_ops,
                                              pctl);
        if (domain)
            return domain;
    }
    return NULL;
}
```

**`irq_domain_create_hierarchy()`** 与 `irq_domain_create_linear()` 的区别在 `parent` 参数：

```c
// kernel/irq/irqdomain.c
struct irq_domain *irq_domain_create_hierarchy(struct irq_domain *parent,
                                                unsigned int flags,
                                                unsigned int size,
                                                struct fwnode_handle *fwnode,
                                                const struct irq_domain_ops *ops,
                                                void *host_data)
{
    struct irq_domain *domain = __irq_domain_add(fwnode, size, size, 0, ops, host_data);
    if (domain)
        domain->parent = parent;       // ← 关键：链接到 parent domain
    return domain;
}
```

这意味着 GPIO domain 和 EXTI domain 之间通过 `parent` 指针关联。当 `request_irq` 触发 domain 分配时，GPIO domain 的 `.alloc` 会递归调用 `irq_domain_alloc_irqs_parent()`，逐级向上穿越 EXTI → GIC。

### 3.6.3 GPIO 与 EXTI/GIC 的三种模型

必须区分清楚 STM32MP257 上中断控制器的三种交互模型：

| 关系 | 模型 | 实现方式 |
|------|------|---------|
| GIC → CPU | 硬件信号 | CPU 中断引脚 + 向量表 |
| EXTI → GIC | **链式（Chained）** | EXTI 占用 GIC SPI，`irq_set_chained_handler()` 安装分发函数 |
| GPIO → EXTI | **层级（Hierarchy）** | `irq_domain_create_hierarchy()`，irq_data 链委派 |

**链式模型**：EXTI 有自己的 irq_desc（由 GIC 的 SPI 中断号创建），flow handler 是 `handle_fasteoi_irq`，action 中是 `stm32_irq_handler`——这个 handler 作为分发员，读取 EXTI 内部寄存器后调用 `generic_handle_irq` 继续分发。

**层级模型**：GPIO bank 不创建独立的 irq_desc。GPIO pin 的中断号在分配时穿越三层 domain，形成一个 irq_desc 对应多个 irq_data（每层一个）。mask/unmask 操作通过 `irq_chip_mask_parent()` 沿 parent_data 链向上委派：

```
irq_desc[virq]
  .irq_data (内嵌) → domain = gpio_domain, chip = stm32_gpio_irq_chip
    .parent_data → irq_data (alloc) → domain = exti_domain, chip = generic(IMR)
      .parent_data → irq_data (alloc) → domain = gic_domain, chip = gic_chip
        .parent_data = NULL
```

当调用 `mask()` 时：

```
stm32_gpio_irq_mask(data)
  → irq_chip_mask_parent(data)       ← GPIO 层直接上传
    → exti generic chip mask         ← 写 EXTI_IMR 清 bit
      → irq_chip_mask_parent(data)   ← EXTI 层也上传？
         → gic_chip.mask             ← 写 GICD_ICENABLER
```

实际上 STM32 的 GPIO irq_chip 的 mask 回调就是直接调用 `irq_chip_mask_parent()`——因为 GPIO bank 本身没有中断控制寄存器，所有控制都在 EXTI 中。mask 一个 GPIO 中断相当于 mask 对应的 EXTI line。

### 3.6.4 完整的 irq_domain 层级

```
GIC domain (root, parent = NULL)
  .ops = gic_irq_domain_hierarchy_ops
  .host_data = gic_data[0]
  revmap[0..1019]
      ↑ parent
      │
  EXTI domain (第一级子域)
  .ops = stm32mp_exti_domain_ops
  .host_data = NULL (使用 generic chip)
  revmap[0..63]
      ↑ parent
      │
  GPIO domain (第二级子域，每个 pin controller 一个)
  .ops = stm32_gpio_domain_ops
  .host_data = stm32_pinctrl
  revmap[0..15]  (16 GPIO IRQ lines)
```

---

## 3.7 阶段四：request_irq 流程

前三阶段完成了基础设施的初始化。**本阶段是消费者设备将自己的中断 handler 注册到系统中的过程。**

以 gpio-keys（PH5）为例，DTS 中声明：

```dts
gpio-keys {
    compatible = "gpio-keys";
    button-jump {
        gpios = <&gpioh 5 GPIO_ACTIVE_LOW>;
        interrupt-parent = <&gpioh>;
        interrupts = <5 IRQ_TYPE_EDGE_BOTH>;
    };
};
```

中断号的分配在设备驱动 probe 时由 DTS 解析自动触发。

### 3.7.1 中断号的创建：irq_create_of_mapping

```c
// drivers/of/irq.c
unsigned int irq_create_of_mapping(struct of_phandle_args *irq_data)
{
    struct irq_domain *domain;

    // ① 根据 of_node 找到 domain
    domain = irq_find_host(irq_data->np);
    if (!domain) return 0;

    // ② 创建或查找 virq
    return irq_create_fwspec_mapping(&fwspec);
}
```

`irq_find_host(irq_data->np)` 遍历 `irq_domain_list` 全局链表，比对 DTS 节点的 `of_node` 指针与 domain 的 `fwnode`，找到对应的 domain。

对 gpio-keys，`interrupt-parent = <&gpioh>` → `irq_find_host(gpioh_of_node)` → 返回 GPIO domain。

```c
unsigned int irq_create_fwspec_mapping(struct irq_fwspec *fwspec)
{
    struct irq_domain *domain = irq_find_host(fwspec->fwnode);
    irq_hw_number_t hwirq;
    unsigned int type, virq;

    // ① 通过 .xlate 回调翻译 (cell[0], cell[1]) → (hwirq, type)
    domain->ops->xlate(domain, ...);

    // ② 查询是否已分配——查 linear_revmap[]
    virq = irq_find_mapping(domain, hwirq);
    if (virq) {
        irq_set_irq_type(virq, type);       // 更新触发类型
        return virq;
    }

    // ③ 新分配——穿越 domain 层级
    virq = irq_domain_alloc_irqs(domain, 1, NUMA_NO_NODE, fwspec);
    if (virq)
        return virq;
}
```

**第二次调用同一个中断源时**，`irq_find_mapping(domain, hwirq)` 会从 revmap 中返回已分配的 virq——这是共享中断不会重复分配的依据。

### 3.7.2 irq_domain_alloc_irqs — 分四步

```c
// kernel/irq/irqdomain.c — 简化后的核心逻辑
int __irq_domain_alloc_irqs(struct irq_domain *domain, int irq_base,
                             unsigned int nr_irqs, int node, void *arg)
{
    return irq_domain_alloc_irqs_locked(domain, irq_base, nr_irqs,
                                         node, arg, false, NULL);
}

static int irq_domain_alloc_irqs_locked(domain, irq_base, nr_irqs, node, arg, ...)
{
    int virq;

    // ═══════════════════════════════════════════════
    // 第 1 步：分配 virq 号 + irq_desc
    // ═══════════════════════════════════════════════
    virq = irq_domain_alloc_descs(irq_base, nr_irqs, 0, node, affinity);
    // 从全局 allocated_irqs 位图中找到一个空闲 bit
    // bit 的索引就是 virq 号（如 42）

    // ═══════════════════════════════════════════════
    // 第 2 步：创建层级 irq_data 链
    // ═══════════════════════════════════════════════
    irq_domain_alloc_irq_data(domain, virq, nr_irqs);
    // irq_desc[virq] 中内嵌的 irq_data 归传入 domain 所有
    // 为每个 parent domain 分配额外的 irq_data（kzalloc），通过 parent_data 指针串联

    // ═══════════════════════════════════════════════
    // 第 3 步：调用每层的 .alloc 回调（从叶子域到根域）
    // ═══════════════════════════════════════════════
    irq_domain_alloc_irqs_hierarchy(domain, virq, nr_irqs, arg);
    // 递归：domain->ops->alloc() → 内部调 irq_domain_alloc_irqs_parent()
    //   → parent->ops->alloc() → 内部调 irq_domain_alloc_irqs_parent()
    //     → ...

    // ═══════════════════════════════════════════════
    // 第 4 步：填充各层的 revmap
    // ═══════════════════════════════════════════════
    irq_domain_insert_irq(virq);
    // 遍历 irq_data 链，为每层写入 domain->revmap[hwirq] = irq_data

    return virq;
}
```

#### 第 1 步：分配 virq

```c
int irq_domain_alloc_descs(int virq, unsigned int cnt, irq_hw_number_t hwirq,
                            int node, const struct irq_affinity_desc *affinity)
{
    // 一般传入 virq = -1，表示自动分配
    // 从 irq_alloc_descs() → 在 allocated_irqs 位图中找一个空闲区间
    // 找到后分配 irq_desc（如果之前还没分配过）
    return irq_alloc_descs(virq, 0, cnt, node);
}
```

virq 号由内核全局统一分配，与 hwirq 没有固定关系。第一次分配通常从 16 开始（0~15 是 legacy IRQ 保留）。

#### 第 2 步：创建 irq_data 链

```c
static int irq_domain_alloc_irq_data(struct irq_domain *domain,
                                      unsigned int virq, unsigned int nr_irqs)
{
    for (i = 0; i < nr_irqs; i++) {
        irq_data = irq_get_irq_data(virq + i);   // 取 irq_desc 内嵌的 irq_data
        irq_data->domain = domain;               // 设置最外层 domain

        for (parent = domain->parent; parent; parent = parent->parent) {
            irq_data = irq_domain_insert_irq_data(parent, irq_data);
            // 为每层 parent 分配新的 irq_data，链接到 parent_data
        }
    }
    return 0;
}
```

以 GPIO → EXTI → GIC 为例，创建的 irq_data 链：

```
irq_desc[42]
  .irq_data (内嵌) → domain = gpio_domain
    .parent_data → irq_data (kzalloc) → domain = exti_domain
      .parent_data → irq_data (kzalloc) → domain = gic_domain
        .parent_data = NULL
```

#### 第 3 步：递归调用 .alloc

```c
int irq_domain_alloc_irqs_hierarchy(struct irq_domain *domain,
                                     unsigned int irq_base,
                                     unsigned int nr_irqs, void *arg)
{
    return domain->ops->alloc(domain, irq_base, nr_irqs, arg);
}
```

**层级域的各层 .alloc 实现：**

**GIC 域**（叶子域，无 parent）的 `.alloc` 是最底层实现：

```c
static int gic_irq_domain_alloc(struct irq_domain *domain, unsigned int virq,
                                 unsigned int nr_irqs, void *arg)
{
    struct irq_fwspec *fwspec = arg;
    irq_hw_number_t hwirq;
    unsigned int type;

    ret = gic_irq_domain_translate(domain, fwspec, &hwirq, &type);
    if (ret) return ret;

    for (i = 0; i < nr_irqs; i++) {
        ret = gic_irq_domain_map(domain, virq + i, hwirq + i);
    }
    return 0;
}
```

**`.map` 回调——最关键的一步**：

```c
static int gic_irq_domain_map(struct irq_domain *d, unsigned int irq,
                               irq_hw_number_t hw)
{
    struct gic_chip_data *gic = d->host_data;
    const struct irq_chip *chip;

    chip = (static_branch_likely(&supports_deactivate_key) &&
            gic == &gic_data[0]) ? &gic_chip_mode1 : &gic_chip;

    switch (hw) {
    case 0 ... 31:   // SGI + PPI
        irq_set_percpu_devid(irq);
        irq_domain_set_info(d, irq, hw, chip, d->host_data,
                            handle_percpu_devid_irq, NULL, NULL);
        break;
    default:          // SPI (hw ≥ 32)
        irq_domain_set_info(d, irq, hw, chip, d->host_data,
                            handle_fasteoi_irq, NULL, NULL);
        irq_set_probe(irq);
        irqd_set_single_target(irqd);
        break;
    }
    return 0;
}
```

`irq_domain_set_info()` 填充 irq_desc 的关键字段：

```c
void irq_domain_set_info(struct irq_domain *domain, unsigned int virq,
                          irq_hw_number_t hwirq, struct irq_chip *chip,
                          void *chip_data, irq_flow_handler_t handler,
                          void *handler_data, const char *handler_name)
{
    irq_domain_set_hwirq_and_chip(virq, hwirq, chip, chip_data);
    //   → irq_desc[virq].irq_data.hwirq = hwirq
    //   → irq_desc[virq].irq_data.chip  = chip
    //   → irq_desc[virq].irq_data.chip_data = gic_data[0]

    __irq_set_handler(virq, handler);
    //   → irq_desc[virq].handle_irq = handle_fasteoi_irq
}
```

以 SPI 42（hwirq=74）为例，分配完成后 irq_desc[42] 的关键字段：

```
irq_desc[42]
  ├── irq_data (GIC 层):
  │     ├── domain   = gic_domain
  │     ├── hwirq    = 74
  │     ├── chip     = &gic_chip
  │     └── chip_data = gic_data[0]
  ├── handle_irq    = handle_fasteoi_irq
  ├── kstat:        清零
  ├── depth:        1           ← 硬件未使能
  └── action:       NULL        ← 还没有 handler
```

**EXTI 域**的 `.alloc`：

```c
static int stm32mp_exti_alloc(struct irq_domain *d, unsigned int virq,
                                unsigned int nr_irqs, void *fwspec)
{
    struct irq_chip_generic *gc;
    struct stm32mp_exti_chip_data *chip_data;

    gc = irq_get_domain_generic_chip(d, virq);
    chip_data = gc->private;

    irq_domain_set_hwirq_and_chip(d, virq, fwspec->param[0],
                                   &gc->chip_types->chip, chip_data);
    return 0;
}
```

**GPIO 域**的 `.alloc` 类似，但不涉及硬件寄存器操作——它只设置 domain 映射，实际的硬件使能在 EXTI 层完成。

#### 第 4 步：写入 revmap

```c
static void irq_domain_insert_irq(int virq)
{
    struct irq_data *data;
    for (data = irq_get_irq_data(virq); data; data = data->parent_data) {
        struct irq_domain *domain = data->domain;
        domain->mapcount++;
        if (data->hwirq < domain->revmap_size)
            rcu_assign_pointer(domain->revmap[data->hwirq], data);
        else
            radix_tree_insert(&domain->revmap_tree, data->hwirq, data);
    }
}
```

以 GIC domain 为例，`rcu_assign_pointer(domain->revmap[74], irq_data)`——之后中断发生时 `irq_find_mapping(gic_domain, 74)` 直接返回 `revmap[74]`。

### 3.7.3 完整的分配流程（gpio-keys 按键 PH5）

```
DTS: interrupt-parent = <&gpioh>
     interrupts = <5 IRQ_TYPE_EDGE_BOTH>

→ irq_create_of_mapping()
  → irq_find_host(gpioh_of_node)  → GPIO domain
  → irq_create_fwspec_mapping()
    → .xlate 翻译: hwirq=5, type=EDGE_BOTH

    → irq_domain_alloc_irqs(gpio_domain, 1, ...)
      → irq_domain_alloc_descs()     → virq = 42（从位图中分配）
      → irq_domain_alloc_irq_data()  → 创建 3 层 irq_data 链

      → irq_domain_alloc_irqs_hierarchy(gpio_domain)
        → gpio_domain.ops->alloc()
          → irq_domain_alloc_irqs_parent(exti_domain)
            → exti_domain.ops->alloc()
              → irq_domain_set_hwirq_and_chip(exti_domain, ...)
              → irq_domain_alloc_irqs_parent(gic_domain)
                → gic_domain.ops->alloc()
                  → gic_irq_domain_translate()  → hwirq = 268+5=273
                  → gic_irq_domain_map()
                    → irq_domain_set_info()
                      → handle_irq = handle_fasteoi_irq
                      → chip = gic_chip
                      → 写 GICD_ISENABLER[273] = 1（硬件使能）

      → irq_domain_insert_irq(virq)
        → gpio_domain.revmap[5]    = irq_data
        → exti_domain.revmap[5]    = irq_data
        → gic_domain.revmap[273]   = irq_data

  ← 返回 virq = 42
```

### 3.7.4 request_irq — 挂入 handler

`request_irq(42, handler, flags, name, dev)` 是驱动开发者最常用的 API：

```c
// include/linux/interrupt.h
static inline int __must_check
request_irq(unsigned int irq, irq_handler_t handler,
            unsigned long flags, const char *name, void *dev)
{
    return request_threaded_irq(irq, handler, NULL, flags, name, dev);
}
```

`request_threaded_irq`：

```c
// kernel/irq/manage.c
int request_threaded_irq(unsigned int irq, irq_handler_t handler,
                          irq_handler_t thread_fn, unsigned long irqflags,
                          const char *devname, void *dev_id)
{
    struct irqaction *action;
    struct irq_desc *desc;

    // ① 找到 irq_desc（第 1 步已分配，这里直接取）
    desc = irq_to_desc_alloc(irq);
    if (!desc) return -EINVAL;

    // ② 分配 irqaction
    action = kzalloc(sizeof(struct irqaction), GFP_KERNEL);
    action->handler   = handler;       // 顶半部回调
    action->thread_fn = thread_fn;     // 底半部回调（NULL = 纯顶半部）
    action->flags     = irqflags;
    action->name      = devname;
    action->dev_id    = dev_id;

    // ③ 挂入 irq_desc 的 action 链表
    retval = __setup_irq(irq, desc, action);
    return retval;
}
```

`__setup_irq` 的核心：

```c
static int __setup_irq(unsigned int irq, struct irq_desc *desc,
                        struct irqaction *action)
{
    // 确保 flow handler 已经设置（第 3 步已完成）
    if (!desc->action) {
        if (!desc->handle_irq)
            __irq_set_handler(irq, handle_bad_irq);
    }

    // 将 action 挂入链表
    action->irq = irq;
    action->next = NULL;
    if (desc->action) {
        // 共享中断：追加到链表末尾
        ...
    } else {
        desc->action = action;          // 第一个 action
    }

    // 通过 dev_id 检查是否重复注册
    ...

    // 设置线程（如果 thread_fn 不为 NULL）
    if (action->thread_fn) {
        // 创建内核线程 irq/42-button-jump
        action->thread = kthread_create(irq_thread, action, "irq/%d-%s", irq, name);
    }

    // 使能硬件中断（除非 IRQF_NO_AUTOEN）
    if (!(action->flags & IRQF_NO_AUTOEN))
        irq_enable(desc);               // unmask
}
```

`irq_enable` → `__irq_enable` → 遍历 irq_data 链，调用每层的 `irq_chip.irq_unmask`：

```
irq_enable(desc)
  → __irq_enable(desc)
    → 对 irq_data 链的每一层：
      → chip->irq_unmask(data)
        → (GPIO 层)  irq_chip_unmask_parent(d)
          → (EXTI 层) 写 EXTI_IMR bit[5] = 1
            → (GIC 层) 写 GICD_ISENABLER[273] = 1
```

### 3.7.5 完成后的 irq_desc 状态

```
irq_desc[42]  (virq = 42)
├── handle_irq    = handle_fasteoi_irq   ← GIC .map 时设置
├── depth:        0                      ← 已使能
│
├── irq_data 链：
│   ┌────────────────────────────┐
│   │ irq_data (内嵌)            │ ← GPIO 层
│   │   domain  = gpio_domain    │
│   │   hwirq   = 5              │
│   │   chip    = stm32_gpio_    │
│   │            irq_chip        │
│   │   parent_data ────────────┼──→
│   └────────────────────────────┘  │ ┌──────────────────────────┐
│                                    │ │ irq_data (kzalloc)       │ ← EXTI 层
│                                    │ │   domain = exti_domain    │
│                                    │ │   hwirq  = 5              │
│                                    │ │   chip   = generic_chip   │
│                                    │ │   parent_data ──────────┼──→
│                                    │ └──────────────────────────┘ │ ┌────────────────────────┐
│                                    │                                │ │ irq_data (kzalloc)     │ ← GIC 层
│                                    │                                │ │   domain = gic_domain   │
│                                    │                                │ │   hwirq  = 273          │
│                                    │                                │ │   chip   = gic_chip     │
│                                    │                                │ │   parent_data = NULL    │
│                                    │                                │ └────────────────────────┘
├── action:
│   ├── handler  = gpio_keys_gpio_isr  ← 驱动注册的顶半部
│   ├── name     = "button-jump"
│   ├── dev_id   = button 设备结构体
│   ├── thread   = NULL               ← 纯顶半部，无底半部线程
│   └── next     = NULL               ← 非共享中断
│
├── revmap 索引：
│   gpio_domain.revmap[5]   = irq_data
│   exti_domain.revmap[5]   = irq_data
│   gic_domain.revmap[273]  = irq_data
│
├── 硬件状态：
│   EXTI_IMR bit[5] = 1     ← 允许中断通过 EXTI
│   GICD_ISENABLER[273] = 1 ← GIC 使能
│   GICC_PMR = 0xf0         ← 优先级阈值
```

### 3.7.6 request_irq 与 alloc 的分工对比

| | `irq_domain_alloc_irqs()`（第 1~3 步） | `request_irq()`（第 4 步） |
|---|---|---|
| 调用时机 | DTS 解析时（**一次**） | 驱动 probe 时（**每次**） |
| 做什么 | 分配 virq、创建 irq_desc、设 flow handler、填充 revmap | 分配 irqaction、挂入 action 链表 |
| 参数 | fwspec（hwirq + type） | virq + handler + flags |
| 硬件操作 | 写 GICD_ISENABLER、EXTI_IMR | 无（除非 IRQF_NO_AUTOEN 已设） |

---

## 3.8 完整的中断触发路径回顾

所有初始化完成后，按下按键 PH5：

```
PH5 电平变化（高→低）
  → GPIOH bank 输出到 EXTI
    → EXTI_CR[5] 选择 GPIOH
    → EXTI_FTSR bit[5] 下降沿检测匹配
    → EXTI_FPR bit[5] = 1（挂起）
    → EXTI_IMR bit[5] = 1（已使能）
      → EXTI 向 GIC 发送中断信号（SPI#273）

    → GIC Distributor SPI#273 Pending
    → GIC CPU Interface 仲裁（优先级判断）
    → CPU IRQ 引脚触发

    → 查 VBAR_EL1 → vectors[0x280]（EL1h_IRQ）
    → kernel_ventry: sub sp, #296, 保存 pt_regs
    → el1h_64_irq_handler(regs)
      → __el1_irq(regs, handle_arch_irq)
        → enter_from_kernel_mode(regs)    ← lockdep/RCU/trace
        → irq_enter_rcu()
        → gic_handle_irq(regs)
          → 读 GICC_IAR → hwirq = 273
          → irq_find_mapping(gic_domain, 273)
            → gic_domain.revmap[273] → irq_data → irq_desc[42]
          → irq_desc[42].handle_irq = handle_fasteoi_irq
            → mask: gic_chip.irq_mask(data)
              → 写 GICD_ICENABLER[273] = 0
            → action->handler(irq, desc)
              = stm32_irq_handler()       ← EXTI 链式 handler
              → chained_irq_enter(chip, desc)
              → 读 EXTI_RPR|FPR → bit[5] = 1
              → irq_find_mapping(exti_domain, 5)
                → exti_domain.revmap[5] → irq_data → irq_desc[42]
                (注意：这里的 virq2 也是 42？不对——EXTI 是链式模型，
                有自己的 irq_desc。让我们重新分析)
```

**等一下——这里需要区分两个场景：**

**场景 A：设备直接走 EXTI（无 GPIO 层）** — 设备 DTS 中 `interrupt-parent = <&exti1>`，中断直接挂在 EXTI domain 下。此时分配一个独立的 virq（如 43），`irq_desc[43]` 存在于 EXTI domain 中。

**场景 B：设备走 GPIO → EXTI（gpio-keys 场景）** — 设备 DTS 中 `interrupt-parent = <&gpioh>`，中断在 GPIO domain 中分配。此时只有 **一个** irq_desc，但 irq_data 链有三层。当 GIC 分发时，`handle_fasteoi_irq` 调用 action→`stm32_irq_handler()`（如果这个 GIC SPI 对应的是 EXTI 的链式 handler），但 gpio-keys 的 path 并不经过 `stm32_irq_handler`——因为 GPIO 用的是层级模型，不是链式模型。

对 gpio-keys 的正确路径：

```
PH5 → EXTI（边沿检测 + 挂起）
  → GIC SPI#273 Pending
  → gic_handle_irq()
    → irq_find_mapping(gic_domain, 273) → virq = 42
    → irq_desc[42].handle_irq = handle_fasteoi_irq
      → mask: gic_chip
      → action:
          → 这里要看 GIC SPI#273 这个 irq_desc 的 action 是什么
          → 它是 EXTI 通过 irq_set_chained_handler() 安装的 stm32_irq_handler
          → stm32_irq_handler() 读 EXTI 挂起寄存器
            → 发现 line 5 触发了
            → irq_find_mapping(exti_domain, 5) → virq_ex = 43
            → generic_handle_irq(43)
              → irq_desc[43].handle_irq = handle_edge_irq
                → mask: exti_generic_chip（写 EXTI_IMR 清 bit[5]）
                  → 如果此 virq 是层级域（GPIO），需要委派给 parent
                → action->handler()
                  = gpio_keys_gpio_isr()  ← 最终用户 handler
                  → 上报 input 事件
                  → 马里奥跳跃
                → unmask
      → eoi: gic_chip
```

这揭示了一个关键事实：**STM32MP257 上 GPIO 中断实际上走了两层 irq_desc**：

1. **第一层 irq_desc**（virq = 42）：GIC SPI#273 对应的 irq_desc，`handle_irq = handle_fasteoi_irq`，action 中是 `stm32_irq_handler`（EXTI 链式 handler）
2. **第二层 irq_desc**（virq = 43）：EXTI line 5 对应的 irq_desc，`handle_irq = handle_edge_irq`，action 中是 `gpio_keys_gpio_isr`

**但这里 GPIO 的层级模型体现在哪里？** GPIO domain 的层级是相对于 EXTI 而言的——当 GPIO pin 通过 `gpioh` domain 请求中断时，分配在 EXTI domain 下。此时 virq 的 irq_data 链有两层（GPIO → EXTI），但 flow handler 仍由 EXTI domain 的 generic chip 提供（`handle_edge_irq`），GPIO 层的 irq_chip 仅做 mask/unmask 委派。

---

## 3.9 数据结构施工总图

中断初始化完成后，所有关键数据结构之间的关系：

```
┌──────────────────────────────────────────────────────────────────┐
│                    ARM64 CPU                                      │
│  VBAR_EL1 = → vectors[]                                          │
│    el1h_64_irq_handler → __el1_irq → handle_arch_irq            │
│  handle_arch_irq = gic_handle_irq  (由 set_handle_irq 安装)      │
└──────────────────────────┬───────────────────────────────────────┘
                           │ 读 GICC_IAR
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│  GIC (intc@4ac00000)    GIC-400                                 │
│  ┌────────────────┐   ┌──────────────────┐                       │
│  │ gic_data[0]     │   │ irq_domain       │                       │
│  │  .dist_base     │   │  .revmap[0..1019] │ (线性)              │
│  │  .cpu_base      │   │  .ops = gic_irq_  │                     │
│  │  .gic_irqs=1020 │   │    domain_ops     │                      │
│  │  .domain ───────┼──→│  .parent = NULL   │                      │
│  └────────────────┘   │  .host_data = gic  │                     │
│                       └────────┬───────────┘                      │
│  寄存器: 0x4ac10000           │ parent                           │
│   GICD_CTLR, GICD_ISENABLER   │                                  │
│   GICD_ITARGETSR, GICD_IPRIO  │                                  │
│   0x4ac20000                  │                                  │
│   GICC_IAR, GICC_EOIR, PMR   │                                  │
└───────────────────────────────┼──────────────────────────────────┘
                                │ parent
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│  EXTI (exti1@44220000)    stm32mp1-exti                          │
│  ┌────────────────────┐   ┌──────────────────────┐               │
│  │ stm32mp_exti_       │   │ irq_domain (EXTI)    │               │
│  │  host_data          │   │  .revmap[0..63]      │               │
│  │  .base       ───────┤──→│  .ops = stm32mp_exti_ │              │
│  │  .domain ───────────┤──→│    domain_ops         │              │
│  │  .chips_data[2]     │   │  .parent = GIC       │              │
│  └────────────────────┘   └──────────┬───────────┘               │
│  寄存器: 0x44220000                  │ parent                    │
│   IMR(0x080), RTSR(0x0C0)           │                            │
│   FTSR(0x0E0), RPR(0x100)          ▼                            │
│   FPR(0x120), CR(0x060)                                        │
│  ┌────────────────────────────────────────────────────┐          │
│  │ Generic irq_chip (bank0, 32 lines)                  │          │
│  │  irq_ack = stm32_irq_ack                           │          │
│  │  irq_mask = irq_gc_mask_clr_bit                    │          │
│  │  irq_unmask = irq_gc_mask_set_bit                  │          │
│  │  irq_set_type = stm32_irq_set_type                 │          │
│  │  flow handler = handle_edge_irq                    │          │
│  ├────────────────────────────────────────────────────┤          │
│  │ 链式 handler: stm32_irq_handler()                   │          │
│  │   → 读 RPR/FPR → 遍历 → generic_handle_irq(exti_virq)│        │
│  └────────────────────────────────────────────────────┘          │
└───────────────────────────────┬──────────────────────────────────┘
                                │ parent
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│  Pinctrl (44240000.pinctrl) — GPIO domain                        │
│  ┌────────────────────┐   ┌─────────────────────────┐            │
│  │ stm32_pinctrl      │   │ irq_domain (GPIO)        │            │
│  │  .domain ──────────┤──→│  (层次化, 16 items)      │            │
│  │  .banks[9]         │   │  .parent = EXTI          │            │
│  └────────────────────┘   │  .ops = stm32_gpio_      │            │
│                           │    domain_ops            │            │
│  GPIOA irq_chip:          └──────────────────────────┘            │
│    irq_mask → irq_chip_mask_parent → EXTI → GIC                  │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│  irq_desc 实例 — 消费者中断                                       │
│                                                                  │
│  EXTI SPI#273 的 irq_desc:                                       │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ virq = 42                                                   │ │
│  │ handle_irq = handle_fasteoi_irq                             │ │
│  │ irq_data: GIC domain, hwirq=273                             │ │
│  │ action → stm32_irq_handler()  ← EXTI 链式                   │ │
│  │   → 读 EXTI RPR, 分发到 virq=43                             │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  EXTI line 5 的 irq_desc:                                        │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ virq = 43                                                   │ │
│  │ handle_irq = handle_edge_irq                                │ │
│  │ irq_data 链: GPIO → EXTI                                    │ │
│  │  (无 GIC 层——EXTI 已经分担了这个中断)                         │ │
│  │ action → gpio_keys_gpio_isr()  ← 用户 handler               │ │
│  │   → 上报 input 事件 → 马里奥跳跃                              │ │
│  └─────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

---

## 总结

本文追踪了从 CPU 上电到按键中断响应就绪的完整初始化路径，覆盖五个阶段：

**阶段零——ARM64 异常入口**：异常向量表在编译期由 `.entry.text` 段固定，启动时 `head.S` 设置 `VBAR_EL1`。硬件 IRQ 信号到达时，CPU 自动执行 `kernel_ventry` 宏，保存 pt_regs，调 `el1h_64_irq_handler` → `__el1_irq` → `handle_arch_irq`。这是从硬件到软件的"最后一公里"。

**阶段一——GIC 初始化**：通过 `IRQCHIP_DECLARE` 在 `__irqchip_of_table` 段中静态注册，`of_irq_init()` 作为根控制器最先初始化。完成四件事：
1. `set_handle_irq(gic_handle_irq)` — 替换默认 panic handler
2. `gic_dist_init()` — Distributor 使能、路由、优先级初始化
3. `gic_cpu_init()` — CPU Interface 使能、PMR 阈值设置
4. `irq_domain_create_linear()` — 创建 1020 项的线性 revmap 数组

**阶段二——EXTI 初始化**：通过 `module_platform_driver` 在 `device_initcall` 阶段初始化。创建独立 irq_domain、分配 generic irq_chip、通过 `irq_set_chained_handler()` 安装 `stm32_irq_handler() ` 作为链式分发 handler。

**阶段三——GPIO 中断 domain**：在 pinctrl probe 内部通过 `irq_domain_create_hierarchy()` 创建层次化 domain，挂在 EXTI 之下。GPIO 层的 irq_chip mask/unmask 沿 parent_data 链委派给 EXTI/GIC。

**阶段四——request_irq 流程**：消费者驱动 probe 时，DTB 解析触发 `irq_create_of_mapping()` → `irq_domain_alloc_irqs()`。分配过程分四步：分配 virq → 创建 irq_data 链 → 每层 .alloc 回调 → 填充 revmap。完成后，驱动调 `request_irq()` 将 handler 挂入 irq_desc->action 链表。

### 关键设计要点

| # | 要点 | 说明 |
|---|------|------|
| 1 | **Parent-first BFS 拓扑排序** | `of_irq_init()` 的两层循环保证根（GIC）先于子节点初始化 |
| 2 | **早期初始化的约束** | `IRQCHIP_DECLARE` 用静态段注册（无法 kzalloc），适用于 slab 就绪前的早期阶段 |
| 3 | **线性 vs 树形 revmap** | GIC 用线性域（1020 项数组，O(1)），因为 hwirq 连续；MSI 用树形域（O(log n)），因为稀疏 |
| 4 | **链式（Chained）模型** | EXTI 对 GIC：多个 EXTI line 共享一组 GIC SPI，EXTI 的 handler 自己分发（一对多，两层 irq_desc） |
| 5 | **层级（Hierarchy）模型** | GPIO 对 EXTI：每个 GPIO pin 映射到 EXTI line，irq_data 链委派操作（一对一，单层 irq_desc） |
| 6 | **Generic irq_chip** | 标准 mask/unmask 回调（`irq_gc_mask_clr_bit`/`irq_gc_mask_set_bit`）避免每个驱动重复实现 |
| 7 | **分配与注册分离** | `irq_domain_alloc_irqs()` 在 DTS 解析时完成（分配 virq + 硬件使能），`request_irq()` 在 driver probe 时完成（挂 action） |
| 8 | **寄存器映射延迟** | GIC 的 reg[2]/reg[3]（虚拟化寄存器）不由 GIC 驱动映射，仅在 KVM 使能时由 vgic 自行 ioremap |
