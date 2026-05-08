# Linux 驱动开发基础知识

> 本文是驱动子系统学习的**前置知识**，在学习任何一个具体子系统之前，  
> 或在学习过程中遇到障碍时，随时回来查阅。  
> **适用**: STM32MP257 (Cortex-A35, Linux v6.6.78)

---

## 一、设备模型：总线/设备/驱动三元组

### 1.1 核心问题

MCU 的思维：直接操作地址 `*(unsigned long *)0x40005400 = val;`  
Linux 的思维：设备在 DTS 中描述，驱动声明 `compatible`，内核做匹配，然后回调你的 `probe()`。

```
设备树 (DTS)
  │  unflatten_device_tree()
  ▼
device_node 树 (内存中的 DTS 展开)
  │  of_platform_populate()
  ▼
platform_device (注册到 platform bus)
  │  bus->match(drv, dev)
  ▼
platform_driver → really_probe() → driver->probe()
```

### 1.2 三元组关系

```
         ┌─────────────────────────────────────┐
         │          bus_type                     │
         │  .match    : 匹配 driver 和 device    │
         │  .probe    : really_probe            │
         │  .uevent   : 发送消息给 udev          │
         └────────────┬────────────────────────┘
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
 ┌─────────────────┐    ┌──────────────────────┐
 │    device        │    │    driver             │
 │  .init_name      │    │  .of_match_table     │
 │  .bus            │    │  .probe()             │
 │  .driver         │    │  .remove()            │
 │  .driver_data    │    │  .pm = xxx_pm_ops    │
 └─────────────────┘    └──────────────────────┘
```

不同总线的匹配方式：

| 总线类型 | 匹配依据 | 典型设备 |
|---------|---------|---------|
| **platform** | DTS `compatible` 属性 | SoC 片上外设 |
| **I2C** | `id_table` 中的 name | I2C 从设备 (传感器、RTC) |
| **SPI** | `id_table` 或 DT `compatible` | SPI 从设备 (ADC、显示) |
| **PCI** | `vendor:device` ID 号 | PCIe 外设卡 |
| **USB** | `vendor:product` ID 号 | USB 外设 |

### 1.3 device_register 完整流程

```
device_add()
  ├── dev_set_name()              ← 确定 sysfs 名字
  ├── bus_add_device()            ← 挂到 bus 的 device 链表
  ├── bus_probe_device()
  │     └── device_initial_probe()
  │           └── __device_attach()
  │                 └── bus_for_each_drv(drv)
  │                       └── driver_match_device(drv, dev)
  │                             └── bus->match()
  │                           匹配成功↓
  │                             driver_probe_device(drv, dev)
  │                               └── really_probe()
  │                                     ├── dma_configure()
  │                                     └── driver->probe(dev)   ← 你的 probe 被调用
  ├── device_create_file()        ← 创建 sysfs 属性
  └── kobject_uevent()            ← 通知 udev 创建设备节点
```

### 1.4 uevent / sysfs 本质

**sysfs 原理**：每个 `device` 包含一个 `kobject`，在 sysfs 中表现为一个目录。`ktype` 定义了默认的属性文件（show/store 回调）。

**uevent 原理**：`device_add()` → 内核通过 netlink socket 发送消息 → 用户空间的 `udevd` 接收 → 匹配规则 → 创建 `/dev/` 节点、加载固件、启动监控任务。

```bash
# 查看 udev 规则
ls /etc/udev/rules.d/
# 触发规则重载
udevadm control --reload
# 监控 uevent
udevadm monitor
```

### 1.5 EPROBE_DEFER 机制

驱动 probe 时如果依赖的资源尚未就绪，返回 `-EPROBE_DEFER`（-517）。

```
really_probe()
  ├── driver->probe() 返回 -EPROBE_DEFER
  ├── 设备加入 deferred_probe_pending_list
  └── 每次新驱动注册 → 触发 deferred_probe_work_func
        └── 重试 pending 列表里的 probe
```

```c
/* 标准模式 */
supply = devm_regulator_get(dev, "vcc");
if (IS_ERR(supply)) {
    if (PTR_ERR(supply) == -EPROBE_DEFER)
        return -EPROBE_DEFER;    /* 等依赖就绪后重试 */
    return dev_err_probe(dev, PTR_ERR(supply), "get regulator failed");
}
```

```bash
# 调试 deferred probe
cat /sys/kernel/debug/devices_deferred    # 哪些设备在等
cat /sys/kernel/debug/devices_probed      # 哪些 probe 成功
dmesg | grep -i "probe defer"
```

---

## 二、并发模型

### 2.1 Linux 的四种执行上下文

```
  进程上下文
  ┌─────────────────────────────────────────────┐
  │ 入口: read(), write(), ioctl(), sysfs 属性   │
  │ 特点: 可睡眠、可持有 mutex、current 指向用户态 │
  │ 检测: current->mm != NULL (或 !in_interrupt())│
  │ 睡眠: msleep(), wait_event_interruptible()   │
  └─────────────────────────────────────────────┘

  软中断 / tasklet
  ┌─────────────────────────────────────────────┐
  │ 入口: 中断底半部、timer 回调、NAPI poll      │
  │ 特点: 不可睡眠、不可持有 mutex               │
  │ 锁:   spinlock                              │
  │ 检测: in_softirq() 为真                     │
  └─────────────────────────────────────────────┘

  硬中断
  ┌─────────────────────────────────────────────┐
  │ 入口: request_irq() 注册的 handler          │
  │ 特点: 不可睡眠、不可持有 mutex、时间必须极短  │
  │ 锁:   spinlock_irqsave (关本地 CPU 中断)    │
  │ 检测: in_irq() 为真                         │
  └─────────────────────────────────────────────┘

  原子上下文 (Atomic = 硬中断 + 软中断 + 持 spinlock)
  ┌─────────────────────────────────────────────┐
  │ 检测: might_sleep() 会触发 WARN             │
  │ 原则: 不确定是否能睡眠？假设不能！           │
  └─────────────────────────────────────────────┘
```

### 2.2 锁的选择

```
                  mutex       spinlock     spinlock_irq      RCU
                  ─────       ────────     ────────────     ───
  进程上下文       ✅          ✅            ✅              ✅
  软中断          ❌死锁       ✅            ✅              ✅
  硬中断          ❌死锁       ❌死锁         ✅              ✅
  可睡眠          ✅          ❌            ❌              ❌
  临界区大小      任意         极短          极短            任意
  读多写少        普通         普通          普通            ✅最佳
```

**选择规则（优先级）：**
1. 进程上下文且临界区可能睡眠 → `mutex`
2. 原子上下文必须保护 → `spinlock`
3. 数据在硬中断 handler 中也会被访问 → `spin_lock_irqsave()`（关本地 CPU 中断，防死锁）
4. 只在 softirq 中访问 → `spin_lock_bh()`（只关 softirq）
5. 读远多于写 >100:1 → 考虑 `RCU`

### 2.3 经典死锁场景：spin_lock 没有 irqsave

```
CPU0: 进程上下文 spin_lock(&lock)      → 获得锁
CPU1: 硬中断 → my_isr → spin_lock(&lock) → 自旋等待
CPU0: ...持有锁时，本 CPU 中断来了...
CPU0: → 也跑去执行 my_isr → spin_lock(&lock) → 自己锁自己 = 死锁！
```

**解决**：只要锁可能被硬中断中的代码访问，就必须用 `spin_lock_irqsave()`。

```c
/* ✅ 正确 */
unsigned long flags;
spin_lock_irqsave(&dev->lock, flags);
/* ... */
spin_unlock_irqrestore(&dev->lock, flags);

/* ❌ 危险 —— 如果 #ifdef 可能在硬中断中使用 */
spin_lock(&dev->lock);
```

### 2.4 常见函数是否可以睡眠速查

| 函数 | 可睡眠？ | 说明 |
|------|---------|------|
| `kmalloc(GFP_KERNEL, ...)` | ✅ | 进程上下文 |
| `kmalloc(GFP_ATOMIC, ...)` | ❌ | 原子上下文使用 |
| `mutex_lock()` | ✅ | |
| `spin_lock()` | ❌ | |
| `msleep()` | ✅ | |
| `udelay()` | ❌ | 忙等 |
| `copy_to_user()` | ✅ | 可能触发缺页 |
| `kfree()` | ❌ | 可在中断中调用 |
| `wake_up()` | ❌ | 可在中断中调用 |
| `printk()` | ❌ | 可在中断中调用 |
| `dev_err()` | ❌ | 可在中断中调用 |

---

## 三、内存管理与 DMA

### 3.1 三种地址的认知

```
  CPU 视角            MMU                 物理内存           DMA 控制器
┌──────────┐      ┌──────────┐       ┌──────────────┐      ┌──────────┐
│ 虚拟地址 │ ──→  │ 页表映射  │ ──→  │  物理地址     │ ←─→ │ 总线地址 │
│ 0xFFFF...│      │ VA→PA    │       │ 0x4xxx_xxxx  │      │ 0x4xxx.. │
└──────────┘      └──────────┘       └──────────────┘      └──────────┘
                     MMU               物理内存             DMA 引擎
```

- `ioremap(物理地址)` → 虚拟地址（CPU 访问外设寄存器）
- `dma_alloc_coherent()` → 返回两个值：虚拟地址(CPU 用) + DMA 地址(DMA 控制器用)
- 物理地址 ≠ 总线地址（除非 `dma_pfn_offset=0`）

### 3.2 GFP 标志

| 标志 | 适用上下文 | 说明 |
|------|-----------|------|
| `GFP_KERNEL` | 进程上下文 | 标准分配，可睡眠 |
| `GFP_ATOMIC` | 中断/持锁中 | 紧急分配，不可睡眠 |
| `GFP_DMA` | 需要 DMA 地址 | 分配低端 DMA 区域内存 |
| `GFP_DMA32` | 32-bit DMA | 32-bit 地址空间内的内存 |
| `GFP_KERNEL \| GFP_DMA` | 驱动中最常用 | 可睡眠 + DMA 区域 |

### 3.3 DMA API 选择

```
                dma_alloc_coherent       dma_map_single
                ──────────────────       ──────────────
 cache 一致性    硬件维护                 需要手动 sync
 性能            低（每次刷 cache）       高（按需 sync）
 分配时机        probe 时分配             每次传输前
 释放时机        remove 时释放            每次传输后
 返回            虚拟+DMA 双地址          只返回 DMA 地址
 场景            长时间共享 buffer        短时传输
                 (音频、显示 buffer)       (SPI、I2C 传输)
```

### 3.4 Cache Coherency 问题

STM32MP257 Cortex-A35: L1 (32KB I + 32KB D) + L2 (256KB)

```
CPU 写内存:               DMA 控制器写内存:
  CPU → Cache → 物理内存       DMA → 物理内存
       ↓ 数据在 cache 中               ↑ DMA 写完内存
       还没写回                        CPU cache 是脏的

问题: DMA 把新数据写入物理内存，CPU 读的却是 cache 中旧的值！
```

**方案 1 - `dma_alloc_coherent()`**：分配非缓存内存（页表设 cache-off），CPU 和 DMA 看到同一份。简单但性能低（每次访问都走总线）。

**方案 2 - `dma_map_single()` + sync**：
```c
/* 从设备到 CPU（DMA 写完后 CPU 读） */
dma_addr = dma_map_single(dev, buf, len, DMA_FROM_DEVICE);
/* 启动 DMA... */
dma_unmap_single(dev, dma_addr, len, DMA_FROM_DEVICE);
/* unmap 自动做 cache invalidate，CPU 读到的就是新数据 */

/* 从 CPU 到设备（CPU 写完 DMA 读） */
dma_addr = dma_map_single(dev, buf, len, DMA_TO_DEVICE);
/* map 自动做 cache flush，确保 DMA 看到最新数据 */
```

### 3.5 devm_ 自动资源管理

```c
/* devm_ 分配的释放顺序 = LIFO（栈） */

static int my_probe(struct platform_device *pdev)
{
    devm_clk_get(dev, NULL);        /* ① 先分配 */
    devm_request_irq(dev, ...);     /* ② 后分配 */
    misc_register(&dev->misc);      /* ③ 非 devm_ */

    /* remove 时释放顺序: */
    /*  ③ misc_deregister() — 手动 */
    /*  ② devm 释放中断 */
    /*  ① devm 释放时钟 */
}

/* 为什么是 LIFO？因为后注册的依赖先注册的。
   中断 handler 依赖时钟——所以中断先释放，时钟后释放。 */
```

---

## 四、调试基础

```bash
# 第一板斧：静态检查
cat /sys/bus/platform/devices/xxx/              # 设备在不在
cat /sys/kernel/debug/devices_deferred           # 谁 deferred
dmesg | grep -iE "xxx|probe|error"               # 日志

# 第二板斧：tracepoint
trace-cmd record -e irq:irq_handler_entry -e irq:irq_handler_exit sleep 1
trace-cmd report | grep xxx

# 第三板斧：perf probe 动态插桩
perf probe --add 'xxx_probe+0'
perf record -e probe:xxx_probe -a -- sleep 5
perf script
perf probe --del probe:xxx_probe

# 第四板斧：dynamic debug
echo 'file drivers/xxx/xxx.c +p' > /sys/kernel/debug/dynamic_debug/control

# 第五板斧：硬件级
devmem 0x44250000 32                             # 读 GPIO 寄存器
cat /proc/interrupts | grep xxx                  # 看中断
cat /sys/kernel/debug/irq/irqs/<irq>             # 单个中断详情
```

---

## 附录：进阶阅读

- Linux 设备模型: `Documentation/driver-api/driver-model/`
- DMA API: `Documentation/core-api/dma-api.rst`
- 并发/锁: `Documentation/locking/`
- FTrace: `Documentation/trace/ftrace.rst`
- STM32MP2 参考手册: RM0456
- ARM Cortex-A35: ARM ARM (Architecture Reference Manual)
