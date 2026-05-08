# STM32MP257 中断子系统分析

> 📌 **SoC**: STM32MP257D (Cortex-A35 × 1)  
> **内核**: Linux v6.6.78 (stm32mp-r2)  
> **GIC**: GIC-400 (GICv2) + v2m MSI  
> **EXTI**: STM32 外部中断控制器 × 2（`st,stm32mp1-exti`）  
> **参考手册**: STM32MP25xx 参考手册 (RM0456)

---

## 一、使用方法 (Usage)

### 1.1 用户态接口

> 💡 `cat /proc/interrupts` 是调试中断的第一站，显示每个中断号、CPU 分布、触发次数和设备名。

```bash
# 查看所有中断及 CPU 分布
cat /proc/interrupts

# 查看中断控制器类型
cat /proc/irq/*/chip_name

# 查看中断设备名称
cat /proc/irq/*/name

# 查看中断域层次结构
ls /sys/kernel/debug/irq/domains/
cat /sys/kernel/debug/irq/domains/stm32mp-exti

# 查看设备中断号（以某个设备为例）
cat /sys/devices/platform/soc/*/irq

# 调试 GPIO 中断使用
cat /sys/kernel/debug/gpio
```

### 1.2 内核态 API

> 💡 STM32MP2 全部中断 API 都是标准 Linux 内核接口，与 ARM GIC 架构完全兼容。

```c
#include <linux/interrupt.h>

/* ========== 标准中断请求 ========== */

/**
 * request_irq - 请求中断线
 * @irq:  中断号（通过 platform_get_irq 等获取）
 * @handler: 中断处理函数（返回 IRQ_HANDLED 或 IRQ_NONE）
 * @flags:  触发类型标志
 * @name:   中断名称（显示在 /proc/interrupts）
 * @dev:    设备标识（共享中断时区分）
 */
int request_irq(unsigned int irq, irq_handler_t handler,
                unsigned long flags, const char *name, void *dev);

/* 常用 flags:
 *   IRQF_TRIGGER_RISING   - 上升沿触发
 *   IRQF_TRIGGER_FALLING  - 下降沿触发
 *   IRQF_TRIGGER_HIGH     - 高电平触发
 *   IRQF_TRIGGER_LOW      - 低电平触发
 *   IRQF_SHARED           - 共享中断线
 *   IRQF_NO_SUSPEND       - 休眠时保持使能
 */

/* ========== 推荐：devm_ 自动管理版本 ========== */

int devm_request_irq(struct device *dev, unsigned int irq,
                     irq_handler_t handler, unsigned long flags,
                     const char *name, void *dev_id);
/* 设备移除时自动释放 */

/* ========== 获取中断号 ========== */

int platform_get_irq(struct platform_device *pdev, unsigned int num);
int platform_get_irq_byname(struct platform_device *pdev, const char *name);

/* ========== 线程化中断（避免在中断上下文做耗时操作） ========== */

int request_threaded_irq(unsigned int irq, irq_handler_t handler,
                         irq_handler_t thread_fn, unsigned long flags,
                         const char *name, void *dev);
/* handler 返回 IRQ_WAKE_THREAD → thread_fn 在进程上下文运行 */

/* ========== GPIO 中断 ========== */

#include <linux/gpio/consumer.h>

struct gpio_desc *gpiod_get(struct device *dev, const char *con_id);
int gpiod_to_irq(struct gpio_desc *desc);
/* 典型用法: gpiod_get → gpiod_to_irq → devm_request_irq */

/* ========== 中断控制 ========== */

void enable_irq(unsigned int irq);          /* 使能中断 */
void disable_irq(unsigned int irq);         /* 同步禁用（等处理完） */
void disable_irq_nosync(unsigned int irq);  /* 异步禁用 */
void synchronize_irq(unsigned int irq);     /* 等待中断处理结束 */
```

### 1.3 DTS 配置

> 💡 STM32MP2 中断 DTS 使用 `<GIC_SPI/PPI 编号 触发类型>` 三单元格格式，通过 `#include <dt-bindings/interrupt-controller/arm-gic.h>` 获取宏定义。

**GIC 节点** — 定义在 `stm32mp251.dtsi`：

```dts
intc: interrupt-controller@4ac00000 {
    compatible = "st,stm32mp2-cortex-a7-gic", "arm,cortex-a7-gic";
    #interrupt-cells = <3>;
    interrupt-controller;
    interrupt-parent = <&intc>;
    reg = <0x0 0x4ac10000 0x0 0x1000>,    /* GICD: Distributor */
          <0x0 0x4ac20000 0x0 0x2000>,    /* GICC: CPU Interface */
          <0x0 0x4ac40000 0x0 0x2000>,    /* GICH: Virtual I/F */
          <0x0 0x4ac60000 0x0 0x2000>;    /* GICV: Virtual CPU I/F */
    interrupts = <GIC_PPI 9 (GIC_CPU_MASK_SIMPLE(1) | IRQ_TYPE_LEVEL_LOW)>;

    #address-cells = <2>;
    #size-cells = <2>;
    ranges;

    /* MSI 中断支持（PCIe 等设备） */
    v2m0: v2m@48090000 {
        compatible = "arm,gic-v2m-frame";
        reg = <0x0 0x48090000 0x0 0x1000>;
        msi-controller;
    };
};
```

**设备引用中断（三种形式）：**

```dts
/* 形式一：直接 GIC SPI（外设中断） */
/*   GIC_SPI = 0, 编号范围 32-987, 触发类型 */
interrupts = <GIC_SPI 120 IRQ_TYPE_LEVEL_HIGH>;

/* 形式二：GIC PPI（CPU 私有中断） */
/*   GIC_PPI = 1, 编号范围 16-31, 包含 CPU 掩码 */
interrupts = <GIC_PPI 14 (GIC_CPU_MASK_SIMPLE(1) | IRQ_TYPE_LEVEL_HIGH)>;

/* 形式三：通过 EXTI 路由（GPIO 中断/唤醒源） */
interrupts-extended = <&exti1 47 IRQ_TYPE_LEVEL_HIGH>;
```

---

## 二、设计原理与数据结构 (Design & Structures)

### 2.1 整体架构

> 💡 STM32MP257 采用**两级级联中断控制器架构**：EXTI 作为底层中断收集器汇总 GPIO 和系统事件，GIC-400 作为顶层中断控制器分发到 CPU 核。PCIe/MSI 中断通过 v2m frame 桥接到 GIC。

```
                    +-----------+
   GPIO 中断 ────→ |           |     SPI     +-----------+
   外设事件 ────→ | EXTI1/EXTI2 | ────────→ |           |
   SW 触发  ────→ |           |   68 条线   | GIC-400   | ────→ CPU0 (Cortex-A35)
                    +-----------+             | (GICv2)   |
   PCIe/MSI  ────→ | v2m frame | ────────→ |           |
                    +-----------+     SPI    +-----------+
                                                   │
                                              +----------+
                                              | CPU I/F  |
                                              | GICC_*   |
                                              +----------+
```

**中断类型划分：**

| 类型 | 说明 | STM32MP257 范围 | 典型设备 |
|------|------|----------------|---------|
| SGI | 核间中断 (PPI 0-15) | ID0-15 | IPI (SMP) |
| PPI | 私有外设中断 | ID16-31 | Generic Timer, PMU, GIC maintenance |
| SPI | 共享外设中断 | ID32-987 | 全部外设 (UART, DMA, ETH, SDMMC...) |

### 2.2 核心数据结构关系

> 💡 数据结构分四层：irq_domain 是映射框架，host_data 是驱动全局数据，chip_data 是每个 bank 的实例数据，exti_bank 是寄存器偏移描述。

```
irq_domain (层级)
  ├── parent: GIC irq_domain
  ├── ops: stm32mp_exti_domain_ops { .alloc, .free, .xlate, .match, .select }
  └── host_data: stm32mp_exti_host_data  ◄──── 全局宿主数据
        ├── base        : void __iomem*       // EXTI 寄存器 MMIO 基地址
        ├── dev         : struct device*
        ├── hwlock      : struct hwspinlock*   // 可选的硬件自旋锁（多客隔离）
        ├── dt_has_irqs_desc : bool            // true = 从 DT 解析父中断映射
        ├── drv_data    : stm32mp_exti_drv_data*  // 驱动级配置
        │     ├── exti_banks : bank[]          // 3 个 bank 的寄存器描述
        │     ├── bank_nr    : u32 = 3
        │     └── desc_irqs  : u8[96]          // event→GIC SPI 静态映射表
        ├── chips_data[3] : stm32mp_exti_chip_data[]  ◄──── 每 bank 一个
        │     ├── reg_bank     : exti_bank*    // 该 bank 的寄存器偏移表
        │     ├── rlock        : raw_spinlock  // 保护 MMIO 寄存器并发访问
        │     ├── mask_cache   : u32           // IMR 缓存 ← 关键优化
        │     ├── rtsr_cache   : u32           // 上升沿触发保存（PM 恢复用）
        │     ├── ftsr_cache   : u32           // 下降沿触发保存
        │     ├── wake_active  : u32           // 已使能的唤醒源掩码
        │     └── event_reserved : u32         // 安全世界保留的事件位
        └── gpio_mux_used[16]  : bitmap        // GPIO mux 占用状态（event 0-15）
        └── gpio_mux_pos[16]   : u8[]          // GPIO mux 所在的 bank 号
```

### 2.3 EXTI 寄存器组 (Bank)

> 💡 EXTI 有三个 bank，每个 bank 管理 32 个事件。寄存器偏移在不同 bank 间规律错开（间隔 0x10）。

| 寄存器 | 功能 | B1 偏移 | B2 偏移 | B3 偏移 |
|--------|------|---------|---------|---------|
| **RTSR** | 上升沿触发选择 | 0x00 | 0x20 | 0x40 |
| **FTSR** | 下降沿触发选择 | 0x04 | 0x24 | 0x44 |
| **SWIER** | 软件中断触发 | 0x08 | 0x28 | 0x48 |
| **RPR** | 上升沿挂起 | 0x0C | 0x2C | 0x4C |
| **FPR** | 下降沿挂起 | 0x10 | 0x30 | 0x50 |
| **SeCCFGR** | 安全配置 | 0x14 | 0x34 | 0x54 |
| **IMR** | 中断掩码 | 0x80 | 0x90 | 0xA0 |
| **TRG** | 触发类型选择 | 0x3EC | 0x3E8 | 0x3E4 |

```c
/* 驱动中 bank 的静态定义 */
static const struct stm32mp_exti_bank stm32mp_exti_b1 = {
    .imr_ofst     = 0x80,
    .rtsr_ofst    = 0x00,
    .ftsr_ofst    = 0x04,
    .swier_ofst   = 0x08,
    .rpr_ofst     = 0x0C,
    .fpr_ofst     = 0x10,
    .trg_ofst     = 0x3EC,
    .seccfgr_ofst = 0x14,
};
```

### 2.4 设计模式总结

> 💡 五种设计模式相互配合，实现了安全可扩展的中断控制器驱动。

| 模式 | 实现位置 | 说明 |
|------|---------|------|
| **层级 irq_domain** | `irq_domain_add_hierarchy()` | EXTI 注册为 GIC domain 的子域，中断分配先 EXTI 映射再递交给 GIC |
| **写缓存优化** | `chip_data->mask_cache` | 避免频繁 IMR 读-改-写，mask/unmask 直接在缓存操作后写寄存器 |
| **GPIO Mux 仲裁** | `gpio_mux_used[]` bitmap | event 0-15 可被多个 GPIO bank 共享，通过仲裁选择当前路由 |
| **RIF 硬件隔离** | `EXTI_EnCIDCFGR` 寄存器 | RIF 机制实现多客安全隔离，非 CID1 的事件自动标记 reserved |
| **PM 上下文保存/恢复** | `rtsr_cache` / `ftsr_cache` | noirq suspend/resume 时保存和恢复触发配置，保证唤醒后状态一致 |

---

## 三、源码分析 (Source Code Analysis)

### 3.1 关键文件

| 文件 | 行数 | 功能 |
|------|------|------|
| `drivers/irqchip/irq-stm32mp-exti.c` | 993 | STM32MP2 EXTI 中断控制器驱动 |
| `drivers/irqchip/irq-gic.c` | 1754 | ARM GICv2 通用驱动 |
| `drivers/irqchip/irq-gic-v2m.c` | — | GICv2m MSI 支持 |
| `arch/arm64/boot/dts/st/stm32mp251.dtsi` | ~3000 | SoC 级设备树（GIC + EXTI 定义） |

### 3.2 Probe 流程

> 💡 `stm32mp_exti_probe()` 负责初始化 EXTI 硬件、检测 RIF 安全配置、注册层级 irq_domain。

```c
// irq-stm32mp-exti.c:836
static int stm32mp_exti_probe(struct platform_device *pdev)
```

```
① 分配 host_data (devm_kzalloc)
    └── 初始化 gpio_mux_lock
    └── host_data->has_syscon = of_device_is_compatible(np, "syscon")

② 获取 hwspinlock（可选，用于多客间原子操作）
    └── of_hwspin_lock_get_id(np, 0)
    └── 失败返回 EPROBE_DEFER（框架未就绪时延迟重试）

③ 获取驱动配置数据 + ioremap 寄存器
    └── drv_data = of_device_get_match_data(dev)
         └── 当前 compatible "st,stm32mp1-exti" → stm32mp1_drv_data
              ├── exti_banks = stm32mp_exti_banks[3]
              ├── bank_nr = 3
              └── desc_irqs = stm32mp1_desc_irq (event→GIC SPI 映射表)
    └── host_data->base = devm_platform_ioremap_resource(pdev, 0)
         └── EXTI1 → 0x44220000, EXTI2 → 0x46230000

④ 初始化每个 bank
    for i in 0..2:
        stm32mp_exti_chip_init(host_data, i, np)
            ├── 写 0 到 IMR（热重启后清零残留）
            └── 读取 SeCCFGR 保存 event_reserved（安全事件掩码）

⑤ 检查 RIF（Resource Isolation Framework）
    stm32mp_exti_check_rif(host_data)
        ├── 读 EXTI_HWCFGR1 → 检查 CIDWIDTH（是否支持 CID）
        └── 遍历每个 event 的 EXTI_EnCIDCFGR
             └── CID != 1（非 Cortex-A35）→ 标记 event_reserved

⑥ 查找 GIC parent irq_domain
    parent_domain = irq_find_host(of_irq_find_parent(np))
    └── 必须存在，否则返回 -EINVAL

⑦ 注册层级 irq_domain
    domain = irq_domain_add_hierarchy(parent_domain, 0,
                                      bank_nr * 32,
                                      np, &stm32mp_exti_domain_ops,
                                      host_data)

⑧ 可选：注册 wakeup domain（用于唤醒中断路由）
    if (np 有 wakeup-parent 属性 && 有 interrupts-extended)
        → 创建第二个层级 domain，parent 为 wakeup 控制器

⑨ 使能运行时 PM
    devm_pm_runtime_enable(dev)
```

**硬件寄存器要点（RM0456）：**

| 地址/偏移 | 寄存器 | 说明 |
|-----------|--------|------|
| `0x44220000` | EXTI1 基址 | VDD_NS_UC 域，保留电源域不掉电 |
| `0x46230000` | EXTI2 基址 | VDD_NS_UC 域 |
| EXTI 偏移 `0x3F0` | HWCFGR1 | 硬件配置寄存器，CIDWIDTH[27:24] 指示 CID 位宽 |
| EXTI 偏移 `0x180 + n×4` | EnCIDCFGR(n) | 每个 event 的 CID 过滤:CID[6:4], CFEN[0] 使能 |

### 3.3 中断分配流程

> 💡 `stm32mp_exti_domain_alloc()` 在设备 request_irq 时被调用，负责在 EXTI domain 分配中断资源并链接到 GIC parent。

```c
// irq-stm32mp-exti.c:677
static int stm32mp_exti_domain_alloc(struct irq_domain *dm,
                                     unsigned int virq,
                                     unsigned int nr_irqs, void *data)
```

```
① 解析硬件中断号
    hwirq = fwspec->param[0]       // EXTI event 号 (0-95)
    bank  = hwirq / IRQS_PER_BANK  // bank 索引
    chip_data = &host_data->chips_data[bank]

② 安全检查：拒绝安全世界保留的事件
    if (chip_data->event_reserved & BIT(hwirq % 32))
        return -EPERM
    /* 安全事件来自 SeCCFGR + CIDCFGR 配置 */

③ 分配 GPIO mux（仅 event < 16）
    stm32mp_exti_gpio_bank_alloc(dm, fwspec)
        └── 解析 fwspec->param[1] 获取 GPIO bank 编号
        └── test_and_set_bit 跟踪占用

④ 选择 irq_chip 类型（关键步骤）
    event_trg = readl(base + bank->trg_ofst)
    chip = (event_trg & BIT(hwirq % 32)) ?
           &stm32mp_exti_chip : &stm32mp_exti_chip_direct;
    /* TRG=1: 需要 EOI 的标准 chip（边沿/电平触发事件）*/
    /* TRG=0: 直通 chip，EOI 直接传递到父域 */

⑤ 设置 irq_chip + chip_data
    irq_domain_set_hwirq_and_chip(dm, virq, hwirq, chip, chip_data)

⑥ 查找父中断（GIC SPI 号）并分配
    路径 A: dt_has_irqs_desc == true
        └── of_irq_parse_one(dev->of_node, hwirq) → 从 DT "interrupts-extended" 解析
    路径 B: 使用 drv_data->desc_irqs 静态映射表
        └── desc_irq = host_data->drv_data->desc_irqs[hwirq]
        └── param[1] = desc_irq  →  GIC SPI 中断号

⑦ 在 GIC domain 完成分配
    irq_domain_alloc_irqs_parent(dm, virq, 1, &p_fwspec)
```

### 3.4 中断处理完整路径

> 💡 从硬件触发到驱动 handler 的完整链路：EXTI 检测 → GIC 路由 → CPU 异常 → irq_desc 分发。

```
GPIO pin 电平变化 / 外设中断条件满足
        │
        ▼
① EXTI 检测事件
   └── 边沿触发 → RPR/FPR 对应位置 1
   └── 电平触发 → 电平通过 TRG 配置直通
        │
        ▼
② EXTI IMR 检查（mask_cache）
   └── IMR & BIT(event) != 0   ← 未被掩码
        │
        ▼
③ EXTI 向 GIC 发送 SPI 中断
   └── EXTI1_SPI_0  → GIC SPI 268
   └── EXTI1_SPI_1  → GIC SPI 269
   └── ...（见 DTS interrupts-extended 映射）
        │
        ▼
④ GIC-400 Distributor 处理
   └── 根据 GICD_ISENABLER 检查使能
   └── 根据 GICD_ICFGR 配置选择触发类型
   └── 根据 GICD_ITARGETSR 选择目标 CPU
        │
        ▼
⑤ CPU Interface (GICC)
   └── 读取 GICC_IAR 获得中断 ID
   └── 向 CPU 核断言 IRQ 信号
        │
        ▼
⑥ ARMv8 异常向量
   └── 跳转 el1_irq → handle_arch_irq
   └── gic_handle_irq() → 调用 generic_handle_irq()
        │
        ▼
⑦ irq_desc 和 irq_chip 回调
   └── handle_level_irq() 或 handle_edge_irq()
   └── 调用 stm32mp_exti_mask() → 写 IMR 屏蔽
        │
        ▼
⑧ 设备中断处理函数执行
   └── 驱动注册的 handler() 运行（顶半部）
   └── 或 wakeup 线程（request_threaded_irq）
        │
        ▼
⑨ EOI（End Of Interrupt）
   stm32mp_exti_eoi()
       ├── 写 RPR: writel_relaxed(BIT(event), base + bank->rpr_ofst)
       ├── 写 FPR: writel_relaxed(BIT(event), base + bank->fpr_ofst)
       ├── 恢复 IMR: chip_data->mask_cache |= set_bit(...)
       └── irq_chip_eoi_parent(d) → 写 GICC_EOI 寄存器
```

### 3.5 mask/unmask 实现细节

> 💡 mask_cache 的设计避免了每次操作都从硬件读取 IMR，减少 MMIO 访问延迟。

```c
// irq-stm32mp-exti.c:376
static void stm32mp_exti_mask(struct irq_data *d)
{
    struct stm32mp_exti_chip_data *chip_data = irq_data_get_irq_chip_data(d);
    const struct stm32mp_exti_bank *bank = chip_data->reg_bank;

    raw_spin_lock(&chip_data->rlock);
    /* 硬件：IMR &= ~BIT(hwirq % 32) → 掩码中断 */
    chip_data->mask_cache &= ~stm32mp_exti_clr_bit(d, bank->imr_ofst);
    raw_spin_unlock(&chip_data->rlock);

    irq_chip_mask_parent(d);  /* 同步掩码 GIC 侧 */
}

// irq-stm32mp-exti.c:388
static void stm32mp_exti_unmask(struct irq_data *d)
{
    struct stm32mp_exti_chip_data *chip_data = irq_data_get_irq_chip_data(d);
    const struct stm32mp_exti_bank *bank = chip_data->reg_bank;

    raw_spin_lock(&chip_data->rlock);
    /* 硬件：IMR |= BIT(hwirq % 32) → 使能中断 */
    chip_data->mask_cache |= stm32mp_exti_set_bit(d, bank->imr_ofst);
    raw_spin_unlock(&chip_data->rlock);

    irq_chip_unmask_parent(d); /* 同步使能 GIC 侧 */
}
```

### 3.6 中断触发类型设置

> 💡 EXTI 的触发类型配置与 GIC 配合：RTSR/FTSR 控制边沿检测，但电平触发也在内部映射为双边沿检测。

```c
// irq-stm32mp-exti.c:256
static int stm32mp_exti_convert_type(struct irq_data *d,
                                     unsigned int type, u32 *rtsr, u32 *ftsr)
{
    u32 mask = BIT(d->hwirq % IRQS_PER_BANK);

    switch (type & IRQ_TYPE_SENSE_MASK) {
    case IRQ_TYPE_EDGE_RISING:
        *rtsr |= mask;   *ftsr &= ~mask;   // RTSR=1, FTSR=0
        break;
    case IRQ_TYPE_EDGE_FALLING:
        *rtsr &= ~mask;  *ftsr |= mask;    // RTSR=0, FTSR=1
        break;
    case IRQ_TYPE_EDGE_BOTH:
        *rtsr |= mask;   *ftsr |= mask;    // RTSR=1, FTSR=1
        break;
    case IRQ_TYPE_LEVEL_HIGH:
    case IRQ_TYPE_LEVEL_LOW:
        /* 电平触发在 EXTI 内部映射为双边沿检测 */
        *rtsr |= mask;   *ftsr |= mask;    // RTSR=1, FTSR=1
        break;
    default:
        return -EINVAL;
    }
    return 0;
}
```

### 3.7 电源管理：Suspend/Resume

> 💡 通过 `NOIRQ_SYSTEM_SLEEP_PM_OPS` 注册，在系统完全停止中断前的 noirq 阶段执行，确保唤醒源配置不会被意外清零。

**挂起流程** (`stm32mp_exti_suspend` @L459):

```c
for each bank:
    stm32mp_chip_suspend(chip_data, mask_cache, wake_active)
        1️⃣ 保存 RTSR/FTSR → chip_data->rtsr_cache / ftsr_cache
        2️⃣ IMR = (IMR & ~mask_cache) | wake_active
           └── 关闭所有非唤醒中断，保留唤醒源
        3️⃣ 如果有 wake_active → device_set_wakeup_path(dev)
```

**恢复流程** (`stm32mp_exti_resume` @L493):

```c
1️⃣ stm32mp_exti_resume_gpio_mux(host_data)
   └── 根据 gpio_mux_used 位图重建 EXTI_CR 寄存器
   └── 每个 event 对应 8-bit CR 域，写入正确的 GPIO bank 号

2️⃣ for each bank:
    stm32mp_chip_resume(chip_data, mask_cache, wake_active)
        1️⃣ 恢复 RTSR = rtsr_cache
        2️⃣ 恢复 FTSR = ftsr_cache
        3️⃣ IMR = (IMR & ~wake_active) | mask_cache
           └── 关闭唤醒源，恢复原始掩码状态
```

### 3.8 两个 irq_chip 的区别

> 💡 EXTI 驱动注册了两个 irq_chip：标准 chip 处理需要 EOI 的事件，direct chip 将 EOI 直通到 GIC。

```c
// irq-stm32mp-exti.c:520
static struct irq_chip stm32mp_exti_chip = {
    .name          = "stm32mp-exti",
    .irq_eoi       = stm32mp_exti_eoi,         // 写 RPR+FPR 清挂起
    .irq_ack       = irq_chip_ack_parent,       // 传递 GIC
    .irq_mask      = stm32mp_exti_mask,
    .irq_unmask    = stm32mp_exti_unmask,
    .irq_set_type  = stm32mp_exti_set_type,     // 配置 RTSR/FTSR
    .irq_set_wake  = stm32mp_exti_set_wake,
    .irq_retrigger = stm32mp_exti_retrigger,    // 写 SWIER
    .irq_set_affinity = irq_chip_set_affinity_parent,
};

// irq-stm32mp-exti.c:534 — 用于 TRG=0 的直通事件
static struct irq_chip stm32mp_exti_chip_direct = {
    .name          = "stm32mp-exti-direct",
    .irq_eoi       = irq_chip_eoi_parent,        // 直接传递 GIC，不清 EXTI
    .irq_ack       = irq_chip_ack_parent,
    .irq_mask      = stm32mp_exti_mask,
    .irq_unmask    = stm32mp_exti_unmask,
    .irq_set_type  = irq_chip_set_type_parent,    // 直通 GIC，不配 RTSR/FTSR
    .irq_set_wake  = stm32mp_exti_set_wake,
    .irq_retrigger = irq_chip_retrigger_hierarchy,
    .irq_set_affinity = irq_chip_set_affinity_parent,
};
```

**两者的区别：**

| 特性 | stm32mp_exti_chip | stm32mp_exti_chip_direct |
|------|-------------------|--------------------------|
| TRG 值 | 1 | 0 |
| EOI 行为 | 写 RPR+FPR 清挂起 + 恢复 IMR | 直接调用 irq_chip_eoi_parent |
| set_type | 配置 RTSR/FTSR 寄存器 | 直接传递到 GIC |
| retrigger | 写 SWIER 软件触发 | 层级传递 retrigger |
| 适用场景 | EXTI 检测的事件（边沿/电平） | 直通事件（不需要 EXTI 检测） |

---

## 四、硬件寄存器参考 (RM0456)

> 💡 STM32MP257 参考手册 RM0456 相关章节。以下寄存器位于 EXTI 外设地址空间。

### 4.1 EXTI 寄存器总表

| 偏移 | 寄存器 | Bank1 | Bank2 | Bank3 |
|------|--------|-------|-------|-------|
| `0x000` | **RTSR1** | 上升沿触发 | — | — |
| `0x004` | **FTSR1** | 下降沿触发 | — | — |
| `0x008` | **SWIER1** | 软件中断 | — | — |
| `0x00C` | **RPR1** | 上升沿挂起 | — | — |
| `0x010` | **FPR1** | 下降沿挂起 | — | — |
| `0x014` | **SeCCFGR1** | 安全配置 | — | — |
| `0x020` | **RTSR2** | — | 上升沿触发 | — |
| `0x024` | **FTSR2** | — | 下降沿触发 | — |
| `0x028` | **SWIER2** | — | 软件中断 | — |
| `0x02C` | **RPR2** | — | 上升沿挂起 | — |
| `0x030` | **FPR2** | — | 下降沿挂起 | — |
| `0x034` | **SeCCFGR2** | — | 安全配置 | — |
| `0x040` | **RTSR3** | — | — | 上升沿触发 |
| `0x044` | **FTSR3** | — | — | 下降沿触发 |
| `0x048` | **SWIER3** | — | — | 软件中断 |
| `0x04C` | **RPR3** | — | — | 上升沿挂起 |
| `0x050` | **FPR3** | — | — | 下降沿挂起 |
| `0x054` | **SeCCFGR3** | — | — | 安全配置 |
| `0x060` | **CR(0)~CR(15)** | GPIO MUX 配置（每事件 8-bit × 4 事件/寄存器） |
| `0x080` | **IMR1** | 中断掩码（Bank1 event 0-31） |
| `0x090` | **IMR2** | 中断掩码（Bank2 event 0-31） |
| `0x0A0` | **IMR3** | 中断掩码（Bank3 event 0-31） |
| `0x180` | **EnCIDCFGR(n)** | 各 event 的 CID 配置（n=0..95） |
| `0x3E4` | **TRG3** | Bank3 触发类型选择 |
| `0x3E8` | **TRG2** | Bank2 触发类型选择 |
| `0x3EC` | **TRG1** | Bank1 触发类型选择 |
| `0x3F0` | **HWCFGR1** | 硬件配置（CIDWIDTH[27:24]） |

### 4.2 EXTI↔GIC SPI 映射（EXTI1 关键路由）

> 📌 详见 `stm32mp251.dtsi` 中 `exti1` 节点的 `interrupts-extended` 属性

| EXTI Event | GIC SPI | 典型用途 |
|------------|---------|---------|
| EXTI_0 | SPI 268 | GPIO wakeup event 0 |
| EXTI_1 | SPI 269 | GPIO wakeup event 1 |
| ... | ... | ... |
| EXTI_16 | SPI 0 | — |
| EXTI_17 | SPI 1 | — |
| EXTI_18 | SPI 260 | — |
| EXTI_19 | SPI 259 | — |
| EXTI_20 | 未连接 | — |
| EXTI_21 | SPI 108 | — |
| ... | ... | ... |

### 4.3 GIC-400 寄存器

| 地址 | 寄存器 | 功能 |
|------|--------|------|
| `0x4ac10000` | **GICD_CTLR** | Distributor 控制 |
| `0x4ac10004` | **GICD_TYPER** | 类型（中断线数、CPU 数） |
| `0x4ac10100` | **GICD_ISENABLERn** | 中断使能设置 |
| `0x4ac10180` | **GICD_ICENABLERn** | 中断使能清除 |
| `0x4ac10800` | **GICD_ICFGRn** | 中断触发配置 |
| `0x4ac11000` | **GICD_ITARGETSRn** | CPU 目标 |
| `0x4ac20000` | **GICC_CTLR** | CPU 接口控制 |
| `0x4ac2000C` | **GICC_IAR** | 中断确认（读取获得中断 ID） |
| `0x4ac20010` | **GICC_EOIR** | 中断结束（写入通知 GIC） |
| `0x4ac20014` | **GICC_RPR** | 当前运行优先级 |
| `0x4ac2001C` | **GICC_PMR** | 优先级掩码 |

---

## 附录：调试技巧

> 💡 三板斧调试 STM32MP2 中断问题。

```bash
# 1. 确认中断触发
cat /proc/interrupts | grep <device_name>
watch -n1 'cat /proc/interrupts | grep <device_name>'  # 实时监控

# 2. 确认中断域映射
cat /sys/kernel/debug/irq/domains/stm32mp-exti

# 3. 确认 EXTI 寄存器状态（devmem2 或 busybox devmem）
# 查看 EXTI1 IMR1 (0x44220080)
devmem 0x44220080
# 查看 RTSR1 (0x44220000)
devmem 0x44220000
# 查看挂起位 RPR1 (0x4422000C)
devmem 0x4422000C

# 4. trace 中断处理延迟
trace-cmd record -e irq_handler_entry -e irq_handler_exit
trace-cmd report | grep <device_name>

# 5. 动态调试 EXTI 驱动
echo 'file irq-stm32mp-exti.c +p' > /sys/kernel/debug/dynamic_debug/control
```
