# 04. 运行时情景分析

> 本文是 STM32MP257 Pinctrl & GPIO 深度分析系列的第 4 篇。
> 从实际使用场景出发，自上而下追踪关键路径从用户态（或内核驱动）直到硬件寄存器。
>
> **前置:** [03-SourceAnalysis.md](03-SourceAnalysis.md) — 初始化流程源码分析
> **下一篇:** [05-Client-Usage.md](05-Client-Usage.md) — GPIO 消费者驱动开发
>
> **字数：约 23,000 词（含代码段）**
> **建议阅读时间：45~70 分钟**

---

## 4.1 概述

03 篇追踪了系统从启动到 pinctrl & GPIO 子系统全部就绪的初始化流程。本篇则聚焦**初始化完成之后**——当 consumer 设备（UART、SDMMC、I2C 等）和用户态程序真正使用这些子系统时，代码路径是怎样的。

本篇选择两个最核心的场景：

| 场景 | 起点 | 终点 | 涉及的核心路径 |
|------|------|------|--------------|
| **场景一：Consumer probe 设定引脚复用** | consumer 设备 probe → `pinctrl_bind_pins()` | `stm32_pmx_set_mode()` 写 AFRL/AFRH + MODER | `pinctrl_select_state()` → `create_pinctrl()` → `dt_to_map()` → `add_setting()` → `set_mux` 回调 |
| **场景二：`/dev/gpiochip` 用户态 GPIO 控制** | `open("/dev/gpiochip0")` → `ioctl()` | `stm32_gpio_set/get()` 读写 BSRR/IDR<br>`stm32_gpio_to_irq()` → EXTI → GIC | gpiolib cdev → gpiolib core → `gpio_chip` 回调（含 `set`/`get`/`to_irq`/`direction_output`） |

## 4.2 场景一：Consumer 设备 probe 的引脚复用设定

### 4.2.1 从一个问题开始

ATK 板的控制台串口是 USART2，在 DTS 中这样定义：

```dts
// stm32mp257d-atk-bsp.dts:763
&usart2 {
    pinctrl-names = "default", "idle", "sleep";
    pinctrl-0 = <&usart2_pins_a>;
    pinctrl-1 = <&usart2_idle_pins_a>;
    pinctrl-2 = <&usart2_sleep_pins_a>;
    /delete-property/dmas;
    /delete-property/dma-names;
    status = "okay";
};
```

其中 `usart2_pins_a` 在 pinctrl DTSI 中定义：

```dts
// stm32mp25-pinctrl-atk-ddr-2GB.dtsi:508
usart2_pins_a: usart2-0 {
    pins1 {
        pinmux = <STM32_PINMUX('A', 4, AF6)>; /* USART2_TX */
        bias-disable;
        drive-push-pull;
        slew-rate = <0>;
    };
    pins2 {
        pinmux = <STM32_PINMUX('A', 8, AF8)>; /* USART2_RX */
        bias-pull-up;
    };
};
```

**问题是**：USART2 驱动 probe 时，驱动代码里没有任何显式的 pinctrl 调用——那 PA4 和 PA8 是怎么自动变成 USART2 收发引脚的？从 DTS 中 `pinctrl-0 = <&usart2_pins_a>` 到 GPIO 寄存器 MODER/AFRL/AFRH 的具体值，中间经历了哪些代码路径？

### 4.2.2 完整调用链全景

下面是整个场景一从 USART2 驱动 probe 触发到 GPIO 寄存器写入的完整路径。整体分三大步：

1. **pinctrl_bind_pins** 由设备核心层自动调用，触发获取 pinctrl 句柄
2. **create_pinctrl** 内部做了两次遍历——第一次把 DTS 中的 `pinctrl-0` 属性解析成通用的 map 条目（"PA4 要配成 af6"这种字符串形式），第二次把这些 map 条目转成当前 pin controller 能直接用的整数 setting（group=4, func=7）
3. **pinctrl_select_state** 分两阶段执行 setting——先调 `pinmux_enable_setting` 写 AFRx + MODER 选择复用功能，再调 `pinconf_apply_setting` 写 PUPDR/OTYPER/OSPEEDR 配置电气参数

```
USART2 设备驱动 probe
USART2 设备驱动 probe
  │
  │ [驱动核心层] driver_probe_device()  ← drivers/base/dd.c:636
  │
  ├── ① pinctrl_bind_pins(dev)          ← drivers/base/pinctrl.c:21
  │      └─ devm_pinctrl_get(dev)        ← drivers/pinctrl/core.c:1150
  │           └─ create_pinctrl(dev, NULL)  ← drivers/pinctrl/core.c:1043
  │                │
  │                ├─ ▲ 第一次遍历 ▲
  │                │  pinctrl_dt_to_map(p, pctldev)  ← drivers/pinctrl/devicetree.c:200
  │                │    ├─ 寻找 "pinctrl-0" 属性
  │                │    ├─ 解析 phandle → 找到 pin config 节点
  │                │    └─ dt_to_map_one_config(p, ...)  ← drivers/pinctrl/devicetree.c:109
  │                │         └─ ops->dt_node_to_map(pctldev, np, &map, &num_maps)
  │                │              └─ stm32_pctrl_dt_node_to_map()  ← pinctrl-stm32.c:947
  │                │                   └─ 逐个子节点:
  │                │                        stm32_pctrl_dt_subnode_to_map()  ← pinctrl-stm32.c:852
  │                │                         ├─ 解析 pinmux = <0x407> -> PA4 AF6
  │                │                         │              → STM32_PINMUX('A',4,AF6) = 0x407
  │                │                         │              → STM32_GET_PIN_NO(0x407) = 4
  │                │                         │              → STM32_GET_PIN_FUNC(0x407) = 7
  │                │                         │              → 查找 group
  │                │                         ├─ stm32_pctrl_dt_node_to_map_func()
  │                │                         │    → 创建 PIN_MAP_TYPE_MUX_GROUP map entry
  │                │                         └─ pinctrl_utils_add_map_configs() [可选]
  │                │                              → 创建 PIN_MAP_TYPE_CONFIGS_GROUP map entry
  │                │              dt_remember_or_free_map()  ← drivers/pinctrl/devicetree.c:65
  │                │               → pinctrl_register_mappings() → 挂入 pinctrl_maps 链表
  │                │
  │                ├─ ▲ 第二次遍历 ▲
  │                │  for_each_pin_map() → add_setting()
  │                │    ├─ pinmux_map_to_setting()  ← drivers/pinctrl/pinmux.c
  │                │    └─ pinconf_map_to_setting() ← drivers/pinctrl/pinconf.c
  │                │
  │                └─ setting 挂入 state->settings 链表
  │
  ├── ② pinctrl_lookup_state(p, "default") → 找到 default state
  │
  └── ③ pinctrl_select_state(p, default_state)  ← drivers/pinctrl/core.c:1358
       └─ pinctrl_commit_state(p, state)        ← drivers/pinctrl/core.c:1256
            │
            ├─ [Pinmux 阶段] 遍历 state->settings:
            │   for PIN_MAP_TYPE_MUX_GROUP:
            │     pinmux_enable_setting(setting)  ← drivers/pinctrl/pinmux.c:418
            │       ├─ pctlops->get_group_pins() → 获取组内 pin 列表
            │       ├─ 逐个 pin: pin_request(pctldev, pins[i], owner, NULL)
            │       │    → 检查冲突 → 设置 owner → module 引用
            │       └─ ops->set_mux(pctldev, func, group)
            │            └─ stm32_pmx_set_mux()  ← pinctrl-stm32.c:1098
            │                 ├─ pinctrl_find_gpio_range_from_pin() → 找 bank
            │                 ├─ mode = stm32_gpio_get_mode(function)    → 2 (AF)
            │                 └─ alt  = stm32_gpio_get_alt(function)     → 6
            │                      └─ stm32_pmx_set_mode(bank, pin, mode, alt)
            │                           └─ ① write AFRx: 设置 AF 选择
            │                           └─ ② write MODER: 设置模式 0b10
            │
            └─ [Pinconf 阶段] 遍历 state->settings:
                for PIN_MAP_TYPE_CONFIGS_GROUP:
                  pinconf_apply_setting(setting)
                    └─ stm32_pconf_set() → bias / drive / slew-rate 等
```

### 4.2.3 触发点：pinctrl_bind_pins 在 probe 时序中的位置

Consumer 设备的 probe 不是在代码里主动调用 `pinctrl_bind_pins` 的——它是内核设备核心在 probe 之前自动调用的。

```c
// drivers/base/dd.c:604-659 （简化摘录，省略 defer/links 检查）
static int really_probe(struct device *dev, struct device_driver *drv)
{
re_probe:
    dev->driver = drv;

    /* If using pinctrl, bind pins now before probing */
    ret = pinctrl_bind_pins(dev);           // ← 636: probe 前调 pinctrl
    if (ret)
        goto pinctrl_bind_failed;

    if (dev->bus->dma_configure) {
        ret = dev->bus->dma_configure(dev);
    }

    ret = driver_sysfs_add(dev);

    ret = call_driver_probe(dev, drv);      // ← 659: 最终调驱动 probe
}
```

**时序关系：**

```
really_probe(dev, drv)
  ├── ① pinctrl_bind_pins(dev)     ← ★ 先配置引脚复用
  ├── ② DMA 配置
  ├── ③ driver_sysfs_add
  └── ④ drv->probe(dev)            ← 然后才调驱动自己的 probe
```

### 4.2.4 Stage 1: pinctrl_bind_pins — 获取 pinctrl 句柄

**入口文件：** `drivers/base/pinctrl.c:21`

```c
// drivers/base/pinctrl.c:21
int pinctrl_bind_pins(struct device *dev)
{
    int ret;

    // of_node_reused 标记的节点（如由内核直接创建的虚拟设备）不需要 pinctrl
    // USART2 的 of_node 来自 DTS → 不走此分支
    if (dev->of_node_reused)
        return 0;

    // 在 dev->pins 中分配 dev_pin_info 结构体——之后 pinctrl_get 返回的句柄、
    // 查找到的 default/init/sleep 等 state 指针都挂在这里
    dev->pins = devm_kzalloc(dev, sizeof(*(dev->pins)), GFP_KERNEL);
    if (!dev->pins)
        return -ENOMEM;

    // 调用 devm_pinctrl_get → pinctrl_get → create_pinctrl
    // 这一步会触发完整的 DTS 解析（pinctrl_dt_to_map）+ setting 链表构建
    // 如果 pin controller 驱动还没 probe 完，返回 -EPROBE_DEFER，USART2 的 probe 被延后
    dev->pins->p = devm_pinctrl_get(dev);
    if (IS_ERR(dev->pins->p)) {
        dev_dbg(dev, "no pinctrl handle\n");
        ret = PTR_ERR(dev->pins->p);
        goto cleanup_alloc;
    }

    // 在刚刚构建好的 states 链表中按名查找 "default"
    // USART2 的 DTS 中有 pinctrl-0 = <&usart2_pins_a>，编译后的 state 名来自 pinctrl-names
    // 所以能匹配到 pinctrl-names[0] = "default"
    dev->pins->default_state = pinctrl_lookup_state(dev->pins->p,
                                PINCTRL_STATE_DEFAULT);
    if (IS_ERR(dev->pins->default_state)) {
        dev_dbg(dev, "no default pinctrl state\n");
        ret = 0;
        goto cleanup_get;
    }

    // 查找 "init" state——这是可选的，DTS 中没有定义 pinctrl-init 时返回 -ENOENT
    dev->pins->init_state = pinctrl_lookup_state(dev->pins->p,
                                PINCTRL_STATE_INIT);

    if (IS_ERR(dev->pins->init_state)) {
        // USART2 没配 pinctrl-init，所以走这里——直接应用 default state
        // 这一步会真正调用 pinctrl_commit_state 去写硬件寄存器
        ret = pinctrl_select_state(dev->pins->p,
                                    dev->pins->default_state);
    } else {
        // 如果有 init state（比如某些设备需要 probe 前先配成特定状态），则先选 init
        ret = pinctrl_select_state(dev->pins->p, dev->pins->init_state);
    }

    // 如果内核开启了 CONFIG_PM，还会另外查找 sleep 和 idle state
    // 它们不会现在应用，而是缓存在 dev_pin_info 中供系统休眠时切换
    // ...

    return 0;

cleanup_get:
    devm_pinctrl_put(dev->pins->p);
cleanup_alloc:
    devm_kfree(dev, dev->pins);
    dev->pins = NULL;
    return ret;
}
```

#### `struct dev_pin_info`

`dev_pin_info`（`include/linux/pinctrl/devinfo.h:36`）是嵌入在 `struct device` 中的引脚状态容器，通过 `dev->pins` 访问。它保存了该设备的所有 pinctrl state 句柄：

```c
struct dev_pin_info {
    struct pinctrl *p;                   // pinctrl 句柄（pinctrl_get 返回）
    struct pinctrl_state *default_state; // "default" state
    struct pinctrl_state *init_state;    // "init" state（可选）
#ifdef CONFIG_PM
    struct pinctrl_state *sleep_state;   // "sleep" state
    struct pinctrl_state *idle_state;    // "idle" state
#endif
};
```

各字段在 `pinctrl_bind_pins` 中的填充顺序与代码中的标注①②③...对应：

1. **`p`** ← `devm_pinctrl_get(dev)`（标注 ②）。指向 `create_pinctrl()` 创建的 `struct pinctrl`，内含 `states` 链表（每个 DTS 的 `pinctrl-N` 对应一个 `pinctrl_state` 节点），每个 state 下挂 `settings` 链表（具体的 MUX/CONFIGS setting）。

2. **`default_state`** / **`init_state`** ← `pinctrl_lookup_state()`（标注 ③④）。遍历 `p->states` 链表按名查找，找到后缓存到这些指针中供后续和 PM 切换使用。

3. **`sleep_state`** / **`idle_state`**（PM 使能时）← 标注 ⑥ 中查找。用于系统 suspend/resume 时的引脚状态切换。

**关键步骤：**

| 步骤 | 函数 | 行为 | 出错时的行为 |
|------|------|------|-------------|
| ① | `devm_kzalloc` | 分配 `dev_pin_info` | 返回 `-ENOMEM` |
| ② | `devm_pinctrl_get` | 获取 pinctrl 句柄（触发生成 map → setting） | `dev_dbg` 记录，`goto cleanup_alloc` |
| ③ | `pinctrl_lookup_state(..., "default")` | 在 pinctrl 句柄的 states 链表中按名查找 | `dev_dbg` 记录，`goto cleanup_get` |
| ④ | `pinctrl_lookup_state(..., "init")` | 查找 "init" state（可选） | `dev_dbg` 记录，走 default 路径 |
| ⑤ | `pinctrl_select_state` | **真正写硬件寄存器**，把引脚 mux 到目标功能 | `dev_dbg` 记录，`goto cleanup_get` |

#### devm_pinctrl_get → pinctrl_get → create_pinctrl

这节展开看 `pinctrl_bind_pins` 第 ② 步的调用链。`devm_pinctrl_get` 包了三层：

- **`devm_pinctrl_get`**：devres 管理，创建一个 devres 资源来保存 `struct pinctrl *` 指针，probe 失败或设备移除时自动释放
- **`pinctrl_get`**：遍历全局 `pinctrl_list` 链表，看 USART2 对应的 `struct pinctrl` 结构体是否已经创建过。有就直接返回，不用再走一遍 create_pinctrl
- **`create_pinctrl`**：创建一个 `struct pinctrl` 结构体，内部完成 DTS 解析 + setting 链表构建，**不写硬件**

还有一个关键点：`create_pinctrl(dev, NULL)` 第二个参数传的是 NULL——USART2 不知道自己的 pin controller 是谁，需要进去之后通过 DTS phandle 找到 `pinctrl@44240000`。如果 pin controller 还没 probe 完，这里返回 `-EPROBE_DEFER`。

```c
// drivers/pinctrl/core.c:1379
struct pinctrl *devm_pinctrl_get(struct device *dev)
{
    struct pinctrl **ptr, *p;

    ptr = devres_alloc(devm_pinctrl_release, sizeof(*ptr), GFP_KERNEL);
    if (!ptr)
        return ERR_PTR(-ENOMEM);

    p = pinctrl_get(dev);
    if (!IS_ERR(p)) {
        *ptr = p;
        devres_add(dev, ptr);
    } else {
        devres_free(ptr);
    }

    return p;
}
```

`pinctrl_get` 先遍历全局 `pinctrl_list` 链表，看 USART2 对应的 `struct pinctrl` 结构体是否已经创建过。USART2 是第一次 probe，没有匹配项，于是调用 `create_pinctrl(dev, NULL)`：

```c
// drivers/pinctrl/core.c:1131
struct pinctrl *pinctrl_get(struct device *dev)
{
    struct pinctrl *p;

    // 遍历全局 pinctrl_list：此时链表中只有 pinctrl 驱动自身 hog 时创建的条目，
    // 没有 p->dev == 当前 USART2 device 的条目 → 返回 NULL
    p = find_pinctrl(dev);
    if (p) {
        dev_dbg(dev, "obtain a copy of previously claimed pinctrl\n");
        return p;
    }

    // USART2 走这里：首次创建，第二个参数 NULL 表示非 hog 场景，
    // 进去之后需要通过 DTS phandle 找到 pinctrl@44240000
    return create_pinctrl(dev, NULL);
}
```

`find_pinctrl` 加锁遍历全局 `pinctrl_list` 链表，对每个节点比较 `p->dev == dev`——看这个 device 之前有没有创建过 `struct pinctrl` 结构体。如果没有，返回 NULL。

```c
// drivers/pinctrl/core.c:1026
static struct pinctrl *find_pinctrl(struct device *dev)
{
    struct pinctrl *p;

    mutex_lock(&pinctrl_list_mutex);
    list_for_each_entry(p, &pinctrl_list, node)
        if (p->dev == dev) {         // 匹配条件：pinctrl 句柄的 dev 指针 == 当前 device
            mutex_unlock(&pinctrl_list_mutex);
            return p;
        }

    mutex_unlock(&pinctrl_list_mutex);
    return NULL;                      // USART2 第一次调用，没有匹配项
}
```

USART2 是第一次调用，`pinctrl_list` 中没有 `p->dev` 指向 USART2 device 的条目，所以返回 NULL，接着走 `create_pinctrl(dev, NULL)`。

### 4.2.5 Stage 2: create_pinctrl — 两次遍历核心机制

`create_pinctrl` 是这个场景中最核心的函数。它分两次遍历完成两件事：

**第一次遍历**：读 USART2 的 DTS 节点，找到 `pinctrl-0 = <&usart2_pins_a>`，顺着 phandle 找到 `usart2_pins_a` 节点，把 PA4 和 PA8 的 pinmux 配置解析成 4 个通用的 map 条目——"PA4 要配成 af6"、"PA4 要配 bias-disable"这种字符串形式

**第二次遍历**：把这 4 个 map 条目转成当前 pin controller 能直接用的 setting——原来存的是字符串 "PA4""af6"，现在换成整数索引 group=4、func=7

最终 USART2 的 `struct pinctrl` 结构体里就有了一个叫 "default" 的 state，下面挂着 4 个 setting 节点。**注意到这里还没有写硬件寄存器。**

```c
// drivers/pinctrl/core.c:1043
static struct pinctrl *create_pinctrl(struct device *dev,
                      struct pinctrl_dev *pctldev)
{
    struct pinctrl *p;
    const char *devname;
    struct pinctrl_maps *maps_node;
    const struct pinctrl_map *map;
    int ret;

    p = kzalloc(sizeof(*p), GFP_KERNEL);
    if (!p)
        return ERR_PTR(-ENOMEM);
    p->dev = dev;
    INIT_LIST_HEAD(&p->states);
    INIT_LIST_HEAD(&p->dt_maps);

    // ═══ 第一次遍历：DTS → pinctrl_map ═══
    ret = pinctrl_dt_to_map(p, pctldev);
    if (ret < 0) {
        kfree(p);
        return ERR_PTR(ret);
    }

    devname = dev_name(dev);

    mutex_lock(&pinctrl_maps_mutex);

    // ═══ 第二次遍历：pinctrl_map → pinctrl_setting ═══
    for_each_pin_map(maps_node, map) {
        if (strcmp(map->dev_name, devname))
            continue;

        /* hog 场景下跳过不属于本 pctldev 的 map */
        if (pctldev &&
            strcmp(dev_name(pctldev->dev), map->ctrl_dev_name))
            continue;

        ret = add_setting(p, pctldev, map);
        if (ret)
            goto free_state;
    }

    mutex_unlock(&pinctrl_maps_mutex);

    list_add_tail(&p->node, &pinctrl_list);

    return p;
}
```

**两次遍历的视角差异：**

| 遍历 | 函数 | 输入 | 输出 | 目的地 |
|------|------|------|------|--------|
| **第一次** | `pinctrl_dt_to_map` | DTS 节点（`pinctrl-0` 属性） | `pinctrl_map[]`（通用描述） | 全局 `pinctrl_maps` 链表 + `p->dt_maps` |
| **第二次** | `for_each_pin_map → add_setting` | 全局 `pinctrl_maps` 链表 | `pinctrl_setting`（具体描述） | `state->settings` 链表 |

#### 第一次遍历的深入：pinctrl_dt_to_map

这个函数将 DTS 节点中的 `pinctrl-0`、`pinctrl-1` 等属性解析成内核通用的 `pinctrl_map`。代码涉及的每个内核概念，我们先用 USART2 的 DTS 实际值把它说明白。

**函数处理的 DTS 数据（以 USART2 为例）：**

```dts
usart2 {                                                  // ← p->dev->of_node 指向的节点
    pinctrl-names = "default", "idle", "sleep";
    pinctrl-0 = <&usart2_pins_a>;
    pinctrl-1 = <&usart2_idle_pins_a>;
    pinctrl-2 = <&usart2_sleep_pins_a>;
    /delete-property/dmas;
    /delete-property/dma-names;
    status = "okay";
};
```

##### DTS → DTB 编译时发生了什么

你写的 DTS：

```dts
usart2 {
    pinctrl-0 = <&usart2_pins_a>;   // 人类可读的符号引用
};
```

`dtc` 编译器处理时：
1. 扫描所有节点，发现 `usart2_pins_a` 节点被引用
2. 给 `usart2_pins_a` 节点分配一个唯一编号，例如 `phandle = 0x00001234`
3. 在节点 `usart2_pins_a` 的二进制数据中嵌入这个编号
4. 把引用 `&usart2_pins_a` 替换成这个数字

**编译后 DTB 中 `pinctrl-0` 属性的内容就是 4 个字节：**

```
pinctrl-0 = <0x00001234>;   // 不再是 &usart2_pins_a，而是纯数字
```

DTB 中的实际二进制布局：

```
内存地址       内容 (十六进制)         含义
──────────    ───────────────      ─────────────────
0xXXXX        00 00 12 34          "pinctrl-0" 属性的值：1 个 u32 = 0x00001234
```

DTB 格式规定数值用**大端序**（big-endian）存储，即最高位字节排在最前面。所以 `0x00001234` 在内存中表现为 `00 00 12 34`（而不是小端序的 `34 12 00 00`）。

##### `prop->value` 指向哪里

内核从 DTB 中解析出 `pinctrl-0` 这个属性后，在内存中创建一个 `struct property`，其中的 `value` 指针直接指向 DTB 中的那 4 个字节：

```
prop->value  ────────→  内存地址 0xXXXX
                         [00] [00] [12] [34]    ← 4 字节，大端序
```

##### `list = prop->value` 和 `be32_to_cpup`

`list` 被声明为 `const __be32 *`，意思是"指向大端 32 位整数的指针"。它指向的地址和 `prop->value` 相同，但类型不同：

```c
const __be32 *list = prop->value;
// list 指向： 00 00 12 34 ，但告诉编译器"这 4 个字节是一个大端 u32"
```

ARM CPU（小端序）不能直接读取 `00 00 12 34` 为 `u32`——如果直接 `*list` 读取，小端 CPU 会把它解释为 `0x34120000`，这是个错误的值。所以必须用 `be32_to_cpup`（big-endian to CPU pointer）：

```c
phandle = be32_to_cpup(list);
// list 指向的 4 字节:  00 00 12 34  (大端)
// be32_to_cpup 执行:
//   字节 0 (地址+0): 0x00 → 左移 24 位 → 0x00000000
//   字节 1 (地址+1): 0x00 → 左移 16 位 → 0x00000000
//   字节 2 (地址+2): 0x12 → 左移 8 位  → 0x00001200
//   字节 3 (地址+3): 0x34 → 左移 0 位  → 0x00000034
//   按位或结果:                                  0x00001234  ✓
```

为什么需要 `list++`？因为属性值可以包含多个 phandle：

```dts
pinctrl-0 = <&usart2_pins_a>, <&some_other_pins>;
```

编译后 DTB 中是 8 个字节：`00 00 12 34 00 00 56 78`。`list++` 将指针后移 4 字节，指向下一个大端 u32。

```
初始:  list → [00][00][12][34][00][00][56][78]  ← prop->value 指向的 8 字节
               0   1   2   3   4   5   6   7
               ↑
第一次 be32_to_cpup(list):    phandle = 0x00001234
      list++ → list 指向下一个 4 字节

第二次: list → [00][00][12][34][00][00][56][78]
                                     ↑
               be32_to_cpup(list):    phandle = 0x00005678
```

##### phandle 数字 → 节点

现在有了 `phandle = 0x00001234`。怎么找到节点？

```c
np_config = of_find_node_by_phandle(0x1234);
```

这个函数做什么？内核在 DTB 解析阶段（`unflatten_device_tree`）就已经为每个节点分配了 `struct device_node`，并且把节点的 phandle 号码存在 `np->phandle` 字段。`of_find_node_by_phandle(0x1234)` 做的事情就是：

1. 维护一个全局的 phandle 哈希表（`phandle_cache`）
2. 用 `0x1234` 做哈希查找
3. 返回 `usart2_pins_a` 节点的 `struct device_node *`

```
phandle 缓存（内核内部维护的哈希表）:
  0x00001000 → &{/soc/pinctrl@44240000/usart2-0/pins1}
  0x00001234 → &{/soc/pinctrl@44240000/usart2_pins_a}  ← 找到
  0x00005678 → &{/...}
```

这个 `struct device_node *` 指针 `np_config`，就是 `usart2_pins_a` 节点在内核内存中的表示。它的结构（简化）：

```
np_config (struct device_node *)
  ├── name = "usart2"
  ├── full_name = "/soc/pinctrl@44240000/usart2_pins_a"
  ├── phandle = 0x00001234
  ├── properties → [pinmux] → [bias-pull-up] → ...
  ├── parent → 指向父节点（pin controller 节点）
  └── children → [pins1] → [pins2] → ...
```

##### 几个初学者容易卡住的概念

- **`struct device_node`**：设备树中一个节点 `xxx { ... };` 在内核中的表示。节点名字、属性列表（properties）、父节点指针、子节点链表、phandle 等都存在这里。`p->dev->of_node` 就是 USART2 这个节点。

- **`struct property`**：节点内一个属性 `pinctrl-0 = <...>;` 在内核中的表示。包含属性名（`name`）、值（`value` 指针，指向原始字节）、长度（`length`）。

- **`phandle`**：DTS 编译时，每个被其他节点引用的节点会被分配一个唯一的 32 位数字编号。`<&usart2_pins_a>` 在 DTS 中是符号引用，编译成 DTB 时变成 `phandle = 0x00001234`。内核通过 `of_find_node_by_phandle(0x1234)` 就能找到 `usart2_pins_a` 节点在内存中的 `struct device_node *`。

- **`__be32`**：`__be32` = "big-endian 32-bit"。DTB 格式规定所有多字节数值以**大端序**（高位字节在低地址）存储。ARM 内核通常是小端序运行，所以不能直接用 `*list` 读，必须用 `be32_to_cpup(list)` 将大端转为 CPU 本地字节序。

##### 全过程总结

```
DTS 源文件:
  usart2 { pinctrl-0 = <&usart2_pins_a>; };     ← 人类写的符号
                          │
                          ▼ dtc 编译
DTB 二进制:
  地址 X:  00 00 12 34                            ← phandle 编号
                          │
                          ▼ of_find_property("pinctrl-0")
内核:
  prop->value → 指向地址 X 的 4 字节              ← const __be32 *list
                          │
                          ▼ be32_to_cpup(list)
  phandle = 0x00001234                            ← u32 纯数字
                          │
                          ▼ of_find_node_by_phandle(0x1234)
  np_config = struct device_node *                ← 内存中的节点
              → name = "usart2_pins_a"
              → 包含子节点 pins1 { pinmux = <0x407>; ... }
                                                   ← 下一节展开
```

##### 函数执行流程（对照 USART2）

`pinctrl_dt_to_map` 的主循环遍历 state 编号 0、1、2……，对每个编号：
1. 构造属性名 `"pinctrl-N"`，从节点中查找该属性
2. 如果属性存在，读出属性值中的 phandle 列表
3. 查 `pinctrl-names` 属性获取这个 state 的名称（"default"/"idle"/"sleep"）
4. 对每个 phandle，调用 `dt_to_map_one_config` 解析目标节点

下面用 USART2 的实际值逐步跟踪：

```
第 1 次循环 (state=0):
  propname = "pinctrl-0"
  of_find_property(np, "pinctrl-0", &size) → 找到 ✓
    prop->value 指向 DTB 中该属性的数据区
    size = 4 (1 个 u32 = 4 字节)
    size / 4 = 1 → 1 个 phandle
  pinctrl-names[0] = "default" → statename = "default"
  list 指向 [phandle_of_usart2_pins_a]
  phandle = be32_to_cpup(list) → 如 0x00001234
  np_config = of_find_node_by_phandle(0x1234) → 找到 usart2_pins_a 节点 ✓
  dt_to_map_one_config(p, pctldev, "default", np_config)  → 生成 map

第 2 次循环 (state=1):
  propname = "pinctrl-1"
  of_find_property(np, "pinctrl-1", &size) → 找到 ✓
  pinctrl-names[1] = "idle" → statename = "idle"

第 3 次循环 (state=2):
  propname = "pinctrl-2"
  of_find_property(np, "pinctrl-2", &size) → 找到 ✓
  pinctrl-names[2] = "sleep" → statename = "sleep"

第 4 次循环 (state=3):
  propname = "pinctrl-3"
  of_find_property(np, "pinctrl-3", &size) → 找不到 ✗
  state = 3 > 0 → break (正常结束)
```

**入口文件：** `drivers/pinctrl/devicetree.c:200`

```c
int pinctrl_dt_to_map(struct pinctrl *p, struct pinctrl_dev *pctldev)
{
    // USART2 的 of_node 指向 &usart2 节点，里面有 pinctrl-0/pinctrl-1/pinctrl-2 属性
    struct device_node *np = p->dev->of_node;
    int state, ret;
    char *propname;
    struct property *prop;
    const char *statename;
    const __be32 *list;
    int size, config;
    phandle phandle;
    struct device_node *np_config;

    if (!np) { return 0; }

    of_node_get(np);

    // 循环 0, 1, 2, ... 分别对应 pinctrl-0, pinctrl-1, pinctrl-2, ...
    for (state = 0; ; state++) {
        // 第 1 次: propname = "pinctrl-0", 在 USART2 节点中找到该属性 ✓
        // 第 2 次: propname = "pinctrl-1", 找到 ✓
        // 第 3 次: propname = "pinctrl-2", 找到 ✓
        // 第 4 次: propname = "pinctrl-3", 找不到 ✗ → break
        propname = kasprintf(GFP_KERNEL, "pinctrl-%d", state);
        prop = of_find_property(np, propname, &size);
        kfree(propname);
        if (!prop) {
            // state=0 就找不到 pinctrl-0 → 说明设备没有 pinctrl 配置，返回错误
            // state>0 找不到 → 正常结束循环
            if (state == 0) {
                ret = -ENODEV;
                goto err;
            }
            break;
        }

        // prop->value 指向 DTB 中 pinctrl-0 的二进制数据：如 [00 00 12 34]
        list = prop->value;
        size /= sizeof(*list);     // 除以 4 得到 phandle 个数，pinctrl-0 只有 1 个 phandle

        // 从 pinctrl-names 属性中取这个 state 的名字
        // pinctrl-names = "default", "idle", "sleep"
        // state=0 → statename = "default"
        // state=1 → statename = "idle"
        // state=2 → statename = "sleep"
        ret = of_property_read_string_index(np, "pinctrl-names",
                            state, &statename);
        if (ret < 0)
            statename = prop->name + strlen("pinctrl-");

        // 遍历每个 phandle：pinctrl-0 中只有一个 phandle（usart2_pins_a）
        // 如果 pinctrl-0 = <&usart2_pins_a &some_other_pins>，size 就是 2
        for (config = 0; config < size; config++) {
            phandle = be32_to_cpup(list++);        // 如 0x00001234

            // 通过 phandle 找到 usart2_pins_a 节点在内存中的 device_node
            np_config = of_find_node_by_phandle(phandle);
            if (!np_config) { ret = -EINVAL; goto err; }

            // 把 usart2_pins_a 节点交给下一级处理，生成 map 条目
            ret = dt_to_map_one_config(p, pctldev, statename, np_config);
            of_node_put(np_config);
            if (ret < 0) goto err;
        }
    }

    return 0;

err:
    pinctrl_dt_free_maps(p);
    return ret;
}
```

##### dt_to_map_one_config — 找到 pin controller 并调用回调

这个函数从 pin config 节点出发，**向上找父节点**，直到找到已注册的 pin controller。

**pin config 节点在 DTS 中的定义位置：**

```dts
// stm32mp25-pinctrl-atk-ddr-2GB.dtsi —— 这整个文件就是 &pinctrl { ... }; 的展开
&pinctrl {                                                 // ← pinctrl@44240000 节点
    usart2_pins_a: usart2-0 {                              // ← np_config 指向这里
        pins1 {                                            // ← 子节点（不是父节点！）
            pinmux = <STM32_PINMUX('A', 4, AF6)>;
            bias-disable;
            drive-push-pull;
            slew-rate = <0>;
        };
        pins2 {
            pinmux = <STM32_PINMUX('A', 8, AF8)>;
            bias-pull-up;
        };
    };
};
```

`usart2_pins_a` 被定义在 `&pinctrl {}` 块内，所以它的**父节点就是 `pinctrl@44240000`**。`pins1`/`pins2` 是它的**子节点**，不是父节点。

**向上遍历路径：**

```
np_config = usart2_pins_a 节点
              │
              │ of_get_next_parent()  → 获取该节点的父节点
              ▼
            父节点: pinctrl@44240000
              │
              │ get_pinctrl_dev_from_of_node(pinctrl@44240000)
              │   → 遍历全局 pinctrldev_list
              │   → 找到 stm32_pctl_probe 时注册的 pctldev
              ▼
            找到 pctldev ✅ → break 跳出循环
```

**如果找不到 pin controller（比如驱动还没 probe）：**

```
usart2_pins_a → parent → pinctrl@44240000
                              │
                              │ get_pinctrl_dev_from_of_node()
                              │   → pinctrldev_list 中没有匹配项
                              │   → 返回 NULL
                              ▼
                           继续向上 → 根节点 → -EPROBE_DEFER
```

**入口文件：** `drivers/pinctrl/devicetree.c:109`

```c
static int dt_to_map_one_config(struct pinctrl *p,
                struct pinctrl_dev *hog_pctldev,
                const char *statename,
                struct device_node *np_config)       // ← usart2_pins_a 节点
{
    struct pinctrl_dev *pctldev = NULL;
    struct device_node *np_pctldev;
    const struct pinctrl_ops *ops;
    int ret;
    struct pinctrl_map *map;
    unsigned num_maps;

    // ① 从 np_config 向上找父节点，找 pin controller
    np_pctldev = of_node_get(np_config);
    for (;;) {
        np_pctldev = of_get_next_parent(np_pctldev);
        if (!np_pctldev || of_node_is_root(np_pctldev)) {
            ret = -EPROBE_DEFER;
            return ret;
        }

        pctldev = get_pinctrl_dev_from_of_node(np_pctldev);
        if (pctldev)
            break;
    }
    of_node_put(np_pctldev);

    // ② 调用驱动的 dt_node_to_map 回调
    ops = pctldev->desc->pctlops;
    ret = ops->dt_node_to_map(pctldev, np_config, &map, &num_maps);
    // → stm32_pctrl_dt_node_to_map(pctldev, usart2_pins_a, &map, &num_maps)

    // ③ 将生成的 map 挂入全局链表
    return dt_remember_or_free_map(p, statename, pctldev, map, num_maps);
}
```

> **`get_pinctrl_dev_from_of_node` 怎么通过 DTS 节点找到 pctldev？** 03 篇 probe 流程结束时，`pinctrldev_list` 的状态：
>
> ```
> pinctrldev_list 全局链表（内核启动完成后）:
>   ┌──────────────────────────────────────────────────────────────────────┐
>   │ pctldev[0]: 主域                                                    │
>   │   ├── dev = platform_device "44240000.pinctrl"                     │
>   │   └── dev->of_node = DTS 节点 pinctrl@44240000                     │
>   │                        (compatible = "st,stm32mp257-pinctrl")       │
>   │                        ├── 子节点 gpioa { reg=<0x0> }              │
>   │                        └── ... 到 gpioi                            │
>   ├──────────────────────────────────────────────────────────────────────┤
>   │ pctldev[1]: 安全域                                                  │
>   │   ├── dev = platform_device "46200000.pinctrl"                     │
>   │   └── dev->of_node = DTS 节点 pinctrl@46200000                     │
>   │                        (compatible = "st,stm32mp257-z-pinctrl")     │
>   │                        └── 子节点 gpioz                             │
>   └──────────────────────────────────────────────────────────────────────┘
> ```
>
> 在 `dt_to_map_one_config` 中，`np_pctldev` 是 `usart2_pins_a` 的父节点——即 `pinctrl@44240000`。
>
> ```c
> // drivers/pinctrl/core.c:122
> list_for_each_entry(pctldev, &pinctrldev_list, node) {
>     if (device_match_of_node(pctldev->dev, np_pctldev)) {
>         return pctldev;
>     }
> }
> return NULL;
> ```
>
> 第 1 轮: pctldev[0]（主域, "44240000.pinctrl"），`dev->of_node == pinctrl@44240000` → 匹配！返回 pctldev[0]。
>
> 为什么不是安全域的 pctldev[1]？因为 `usart2_pins_a` 定义在 `&pinctrl {}` 块（主域）中，`np_pctldev` 指向的是 `pinctrl@44240000` 节点而不是 `pinctrl@46200000` 节点。

##### STM32 的回调：stm32_pctrl_dt_node_to_map

找到 pctldev 后，调用 `ops->dt_node_to_map`。对于 STM32 驱动，这个函数遍历 pin config 节点的所有**子节点**（即 `pins1`、`pins2`），每个子节点调一次 `stm32_pctrl_dt_subnode_to_map` 生成 map 条目。

**处理的 DTS 结构：**

```dts
usart2_pins_a: usart2-0 {          // ← np_config（phandle 指向的节点）
    pins1 {                         // ← 子节点 1
        pinmux = <STM32_PINMUX('A', 4, AF6)>;  // DTB: 0x00000407
        bias-disable;
        drive-push-pull;
        slew-rate = <0>;
    };
    pins2 {                         // ← 子节点 2
        pinmux = <STM32_PINMUX('A', 8, AF8)>;  // DTB: 0x00000809
        bias-pull-up;
    };
};
```

```c
// drivers/pinctrl/stm32/pinctrl-stm32.c:947
static int stm32_pctrl_dt_node_to_map(struct pinctrl_dev *pctldev,
                     struct device_node *np_config,
                     struct pinctrl_map **map, unsigned *num_maps)
{
    struct device_node *np;
    unsigned reserved_maps;
    int ret;

    *map = NULL;
    *num_maps = 0;
    reserved_maps = 0;

    // 遍历 np_config 的子节点：pins1 → pins2
    for_each_child_of_node(np_config, np) {
        // 第 1 次循环: np=pins1, 生成 2 个 map (MUX + CONFIGS)
        // 第 2 次循环: np=pins2, 生成 2 个 map (MUX + CONFIGS)
        ret = stm32_pctrl_dt_subnode_to_map(pctldev, np, map,
                &reserved_maps, num_maps);
        if (ret < 0) {
            pinctrl_utils_free_map(pctldev, *map, *num_maps);
            return ret;
        }
    }

    return 0;
}
```

##### stm32_pctrl_dt_subnode_to_map — 解析 pinmux 属性

这个函数读取一个子节点（如 `pins1`）中的 `pinmux` 属性，提取 pin 号和 function 号，查 group，然后创建 `pinctrl_map[]` 条目。

**处理的单个 DTS 子节点（以 `pins1` 为例）：**

```dts
pins1 {                                                    // ← node 参数
    pinmux = <0x00000407>;                                 // ← DTB 中的实际值（大端）
    bias-disable;
    drive-push-pull;
    slew-rate = <0>;
};
```

**`of_find_property(node, "pinmux", NULL)` 读取了什么？**

```
node→properties 链表:
  "pinmux" → name="pinmux", value→[00 00 04 07], length=4
  "bias-disable" → name="bias-disable", value=NULL (空属性)
  "drive-push-pull" → name="drive-push-pull", value=NULL (空属性)
  "slew-rate" → name="slew-rate", value→[00 00 00 00]
```

`pins->length = 4`（4 字节），`sizeof(u32) = 4`，所以 `num_pins = 4/4 = 1`——这个节点中有 1 个 pinmux 值。

**`of_property_read_u32_index(node, "pinmux", 0, &pinfunc)` 提取的值：**

`pinfunc = 0x407`（大端转小端后）

**解码：**

```c
pin  = STM32_GET_PIN_NO(0x407)   = 0x407 >> 8 = 4   → PA4 ✓
func = STM32_GET_PIN_FUNC(0x407) = 0x407 & 0xff = 7  → AF6 编码值 ✓
```

**`stm32_pctrl_find_group_by_pin(pctl, 4)` 查找 group：**

在 probe 时（03-§3.4.5），`stm32_pctrl_build_state` 为每个 pin 创建了一个 group：
```
pctl->groups[4] = { .name = "PA4", .pin = 4 }
```
按 pin 号 4 遍历 groups[]，找到 `groups[4]`，返回 `grp = &groups[4]`。

**然后创建 MUX map 条目：**

```c
stm32_pctrl_dt_node_to_map_func(pctl, 4, 7, grp, &map, &reserved, &num_maps);
// 等价于：
map[0].type               = PIN_MAP_TYPE_MUX_GROUP;
map[0].data.mux.group     = "PA4";                    // group 名
map[0].data.mux.function  = "af6";                    // stm32_gpio_functions[7]
num_maps++;  // → 1
```

**如果还有 pinconf 配置（`bias-disable` 等），创建第二个 map 条目：**

```c
pinctrl_utils_add_map_configs(..., "PA4",
    configs, num_configs, PIN_MAP_TYPE_CONFIGS_GROUP);
// 等价于：
map[1].type                 = PIN_MAP_TYPE_CONFIGS_GROUP;
map[1].data.configs.group   = "PA4";
map[1].data.configs.configs = [PIN_CONFIG_BIAS_DISABLE,
                                PIN_CONFIG_DRIVE_PUSH_PULL,
                                PIN_CONFIG_SLEW_RATE];
map[1].data.configs.num_configs = 3;
num_maps++;  // → 2
```

**处理 `pins2` 时同理：**

```
pinfunc = 0x809
pin  = 0x809 >> 8 = 8       → PA8
func = 0x809 & 0xff = 9     → AF8 编码值
grp  = groups[8]            → { .name = "PA8", .pin = 8 }

map[2]: MUX_GROUP,  group="PA8", function="af8"
map[3]: CONFIGS_GROUP, group="PA8", configs={bias-pull-up}
```

**入口文件：** `pinctrl-stm32.c:852`

node 参数：第一次调用时指向 pins1（PA4），第二次调用时指向 pins2（PA8）

```c
// drivers/pinctrl/stm32/pinctrl-stm32.c:852
static int stm32_pctrl_dt_subnode_to_map(struct pinctrl_dev *pctldev,
                      struct device_node *node,            // ← pins1 或 pins2
                      struct pinctrl_map **map,
                      unsigned *reserved_maps,
                      unsigned *num_maps)
{
    struct stm32_pinctrl *pctl;
    struct stm32_pinctrl_group *grp;
    struct property *pins;
    u32 pinfunc, pin, func;
    unsigned long *configs;
    unsigned int num_configs;
    bool has_config = 0;
    unsigned reserve = 0;
    int num_pins, num_funcs, maps_per_pin, i, err = 0;

    pctl = pinctrl_dev_get_drvdata(pctldev);

    // ↓ 找到这个子节点的 "pinmux" 属性
    //   对于 pins1：pinmux = <0x00000407>，pins->length = 4
    //   对于 pins2：pinmux = <0x00000809>，pins->length = 4
    pins = of_find_property(node, "pinmux", NULL);
    if (!pins) {
        dev_err(pctl->dev, "missing pins property in node %pOFn .\n", node);
        return -EINVAL;
    }

    // ↓ 读取 pinconf 配置（bias, drive, slew-rate 等）
    //   对于 pins1：有 bias-disable、drive-push-pull、slew-rate → num_configs=3
    //   对于 pins2：只有 bias-pull-up → num_configs=1
    err = pinconf_generic_parse_dt_config(node, pctldev, &configs, &num_configs);
    if (err) return err;
    if (num_configs) has_config = 1;

    // ↓ 每个子节点只有 1 个 pinmux 值，所以 num_pins = 1
    //   pins1 和 pins2 各只有 1 个 pin
    num_pins = pins->length / sizeof(u32);
    // ↓ num_pins = 1（每个子节点只有 1 个 pin）
    //   所以 maps_per_pin 计算如下：
    //   pins1：有 pinmux（MUX map） + 有 pinconf（CONFIGS map）= 2 个 map
    //   pins2：有 pinmux（MUX map） + 有 pinconf（CONFIGS map）= 2 个 map
    num_funcs = num_pins;
    maps_per_pin = 0;
    if (num_funcs) maps_per_pin++;
    if (has_config && num_pins >= 1) maps_per_pin++;

    if (!num_pins || !maps_per_pin) {
        err = -EINVAL;
        goto exit;
    }

    // ↓ 给 map 数组扩容——确保至少有 reserve 个空位放即将生成的 map 条目
    //   第一次调（pins1）：*reserved_maps=0, *num_maps=0, reserve=2
    //     → krealloc 从 0 扩到 2
    //   第二次调（pins2）：*reserved_maps=2, *num_maps=2, reserve=2
    //     → old_num(2) < new_num(4)，krealloc 扩到 4
    //   最后 map[] 数组里一共 4 个条目：PA4_MUX, PA4_CONF, PA8_MUX, PA8_CONF
    err = pinctrl_utils_reserve_map(pctldev, map,
            reserved_maps, num_maps, reserve);
    if (err) goto exit;

    // ↓ num_pins = 1，所以只循环 1 次
    for (i = 0; i < num_pins; i++) {
        // 对 pins1：pinfunc = 0x407（PA4 AF6 编码）
        // 对 pins2：pinfunc = 0x809（PA8 AF8 编码）
        err = of_property_read_u32_index(node, "pinmux", i, &pinfunc);
        if (err) goto exit;

        pin  = STM32_GET_PIN_NO(pinfunc);     // 0x407 >> 8 = 4  → PA4
        func = STM32_GET_PIN_FUNC(pinfunc);   // 0x407 & 0xff = 7 → AF6 编码值

        // 检查 PA4 是否真的支持 function 7（af6）
        // 查 pctl->match_data->pins[4] 中声明的 AF 列表
        if (!stm32_pctrl_is_function_valid(pctl, pin, func)) {
            err = -EINVAL;
            goto exit;
        }

        grp = stm32_pctrl_find_group_by_pin(pctl, pin);
        if (!grp) {
            dev_err(pctl->dev, "unable to match pin %d to group\n", pin);
            err = -EINVAL;
            goto exit;
        }

        err = stm32_pctrl_dt_node_to_map_func(pctl, pin, func, grp,
                         map, reserved_maps, num_maps);
        if (err) goto exit;

        // ↓ pinctrl_utils_add_map_configs(... grp->name, configs, num_configs, ...)
        //   先 kmemdup 拷贝 pinconf 数组（因为 map 条目需要独立的内存所有权）
        //   然后填入：
        //     map[1].type = PIN_MAP_TYPE_CONFIGS_GROUP
        //     map[1].data.configs.group_or_pin = "PA4"
        //     map[1].data.configs.configs = {bias-disable, drive-push-pull, slew-rate=0}
        //     map[1].data.configs.num_configs = 3
        //   然后 *num_maps += 1
        // 对 pins1：configs = {bias-disable, drive-push-pull, slew-rate=0}
        // 对 pins2：configs = {bias-pull-up}
        if (has_config) {
            err = pinctrl_utils_add_map_configs(pctldev, map,
                    reserved_maps, num_maps, grp->name,
                    configs, num_configs,
                    PIN_MAP_TYPE_CONFIGS_GROUP);
            if (err) goto exit;
        }
    }

exit:
    kfree(configs);
    return err;
}
```

**三个内部调用的源码分析**

**① `stm32_pctrl_find_group_by_pin`**（pinctrl-stm32.c:789）
遍历 `groups[]` 数组，按 pin 号匹配。对 PA4（pin=4）来说，找到 `groups[4]`：

```c
stm32_pctrl_find_group_by_pin(struct stm32_pinctrl *pctl, u32 pin)
{
    int i;

    for (i = 0; i < pctl->ngroups; i++) {
        struct stm32_pinctrl_group *grp = pctl->groups + i;

        if (grp->pin == pin)        // pin=4 -> groups[4] 匹配
            return grp;
    }

    return NULL;
}
```

**② `stm32_pctrl_dt_node_to_map_func`**（pinctrl-stm32.c:832）
往 `map[]` 数组当前位置填入 MUX 条目：

```c
static int stm32_pctrl_dt_node_to_map_func(struct stm32_pinctrl *pctl,
        u32 pin, u32 fnum, struct stm32_pinctrl_group *grp,
        struct pinctrl_map **map, unsigned *reserved_maps,
        unsigned *num_maps)
{
    if (*num_maps == *reserved_maps)
        return -ENOSPC;

    (*map)[*num_maps].type = PIN_MAP_TYPE_MUX_GROUP;
    (*map)[*num_maps].data.mux.group = grp->name;

    if (!stm32_pctrl_is_function_valid(pctl, pin, fnum))
        return -EINVAL;

    (*map)[*num_maps].data.mux.function = stm32_gpio_functions[fnum];
    (*num_maps)++;
    return 0;
}
```

**③ `pinctrl_utils_add_map_configs`**（pinctrl-utils.c:60）
拷贝 pinconf 数组，填入 CONFIGS 条目：

```c
int pinctrl_utils_add_map_configs(struct pinctrl_dev *pctldev,
        struct pinctrl_map **map, unsigned *reserved_maps,
        unsigned *num_maps, const char *group,
        unsigned long *configs, unsigned num_configs,
        enum pinctrl_map_type type)
{
    unsigned long *dup_configs;

    if (WARN_ON(*num_maps == *reserved_maps))
        return -ENOSPC;

    dup_configs = kmemdup(configs, num_configs * sizeof(*dup_configs),
                          GFP_KERNEL);
    if (!dup_configs)
        return -ENOMEM;

    (*map)[*num_maps].type = type;
    (*map)[*num_maps].data.configs.group_or_pin = group;
    (*map)[*num_maps].data.configs.configs = dup_configs;
    (*map)[*num_maps].data.configs.num_configs = num_configs;
    (*num_maps)++;
    return 0;
}
```

##### 关键：`STM32_PINMUX` 宏的编码规则

```c
// include/dt-bindings/pinctrl/stm32-pinfunc.h:34
#define STM32_PINMUX(port, line, mode) (((PIN_NO(port, line)) << 8) | (mode))

#define PIN_NO(port, line)  (((port) - 'A') * 0x10 + (line))
```

**关键：DTS binding 中的 AF 宏定义**

```c
// include/dt-bindings/pinctrl/stm32-pinfunc.h:11-28
#define GPIO    0x0
#define AF0     0x1
#define AF1     0x2
...
#define AF5     0x6
#define AF6     0x7     ← AF6 = 0x7（不是 6！）
#define AF7     0x8
#define AF8     0x9     ← AF8 = 0x9（不是 8！）
...
#define ANALOG  0x11
```

因为 `GPIO = 0x0` 占用了 0 号位置，所以 **AFn 的编码值 = n + 1**：
- `AF6 = 0x7`（编码值 7，对应硬件 AF 值 6）
- `AF8 = 0x9`（编码值 9，对应硬件 AF 值 8）

**PA4 AF6 的编码过程：**

```
PIN_NO('A', 4) = ('A' - 'A') × 16 + 4 = 0 × 16 + 4 = 4
STM32_PINMUX('A', 4, AF6) = (4 << 8) | 0x7 = 0x400 | 0x7 = 0x407
```

**PA8 AF8 的编码过程：**

```
PIN_NO('A', 8) = ('A' - 'A') × 16 + 8 = 8
STM32_PINMUX('A', 8, AF8) = (8 << 8) | 0x9 = 0x800 | 0x9 = 0x809
```

**解码时用：**

```c
// drivers/pinctrl/stm32/pinctrl-stm32.h:14-15
#define STM32_GET_PIN_NO(x)   ((x) >> 8)        // 右移 8 位取 pin 号
#define STM32_GET_PIN_FUNC(x) ((x) & 0xff)       // 低 8 位取 function

// 对 0x407: pin = 4 (PA4), func = 0x7 (AF6 的 DTS 编码值)
// 对 0x809: pin = 8 (PA8), func = 0x9 (AF8 的 DTS 编码值)
```

**`stm32_gpio_functions[]` 数组：**

```c
// drivers/pinctrl/stm32/pinctrl-stm32.c:113
static const char * const stm32_gpio_functions[] = {
    "gpio", "af0", "af1", "af2", "af3", "af4",
    "af5", "af6", "af7", "af8", "af9", "af10",
    "af11", "af12", "af13", "af14", "af15",
    "analog", "reserved",
};
```

所以 `fnum=7` 对应 `stm32_gpio_functions[7] = "af6"`。

##### dt_remember_or_free_map — 将 map 挂入全局链表

**入口文件：** `drivers/pinctrl/devicetree.c:65`

```c
static int dt_remember_or_free_map(struct pinctrl *p, const char *statename,
                   struct pinctrl_dev *pctldev,
                   struct pinctrl_map *map, unsigned num_maps)
{
    int i;
    struct pinctrl_dt_map *dt_map;

    // ① 为每个 map 条目设置 dev_name（即 consumer 设备名）和 state 名
    for (i = 0; i < num_maps; i++) {
        map[i].dev_name = kstrdup_const(dev_name(p->dev), GFP_KERNEL);
        // → "usart2"
        map[i].name = statename;
        // → "default" (或 "idle", "sleep")
        if (pctldev)
            map[i].ctrl_dev_name = dev_name(pctldev->dev);
            // → "44240000.pinctrl"
    }

    // ② 创建 dt_map 包裹结构体，挂入 dt_maps 链表
    dt_map = kzalloc(sizeof(*dt_map), GFP_KERNEL);
    dt_map->pctldev = pctldev;
    dt_map->map = map;
    dt_map->num_maps = num_maps;
    list_add_tail(&dt_map->node, &p->dt_maps);

    // ③ 注册到全局 pinctrl_maps 链表
    return pinctrl_register_mappings(map, num_maps);
}
```

`pinctrl_dt_map` 是包裹结构体：

```c
struct pinctrl_dt_map {
    struct list_head node;         // 挂入 p->dt_maps 链表
    struct pinctrl_dev *pctldev;  // 分配者（用于释放时调 dt_free_map）
    struct pinctrl_map *map;      // 实际的 map 条目数组
    unsigned num_maps;            // 条目数
};
```

调用后 `pinctrl_maps` 链表（全局）中的 USART2 相关条目：

```
pinctrl_maps 全局链表（每个 map 条目是一个 pinctrl_map）
  ┌──────────────────────────────────────────────────────────┐
  │ map[0]: dev_name="usart2", name="default", type=MUX_GROUP│
  │         .data.mux.group = "PA4"                          │
  │         .data.mux.function = "af6"                       │
  │         .ctrl_dev_name = "44240000.pinctrl"             │
  ├──────────────────────────────────────────────────────────┤
  │ map[1]: dev_name="usart2", name="default", type=CONFIGS  │
  │         .data.configs.group = "PA4"                      │
  │         (bias-disable, drive-push-pull, slew-rate=<0>)   │
  ├──────────────────────────────────────────────────────────┤
  │ map[2]: dev_name="usart2", name="default", type=MUX_GROUP│
  │         .data.mux.group = "PA8"                          │
  │         .data.mux.function = "af8"                       │
  ├──────────────────────────────────────────────────────────┤
  │ map[3]: dev_name="usart2", name="default", type=CONFIGS  │
  │         .data.configs.group = "PA8"                      │
  │         (bias-pull-up)                                   │
  ├──────────────────────────────────────────────────────────┤
  │ ... map[4]~[7] 同样结构，但 name="idle"                  │
  │ ... map[8]~[11] 同样结构，但 name="sleep"               │
  └──────────────────────────────────────────────────────────┘
```

每个子节点（`pins1`/`pins2`）都生成了 2 个 map 条目（MUX + CONFIGS）。USART2 有三个 state（default/idle/sleep），每个 state 有两个子节点，所以总共生成 3 × 2 × 2 = 12 个 map 条目。

#### 第二次遍历的深入：for_each_pin_map → add_setting

回到 `create_pinctrl`，第一次遍历生成 map 并挂入全局链表后，第二次遍历从全局链表捞出属于自己的 map：

```c
// create_pinctrl() 中的第二次遍历 (core.c:1074)
for_each_pin_map(maps_node, map) {
    if (strcmp(map->dev_name, devname)) continue;
    // → 只处理 dev_name == "usart2" 的 map

    ret = add_setting(p, map);
}
```

`add_setting` 的核心工作：**将 map 中的字符串（"PA4"、"af6"）转换为 setting 中的整数选择器（4、7）**。

##### add_setting — 从 map 字符串到 setting 整数

看一个具体的 MUX map 条目怎么转换成 setting：

```
输入 pinctrl_map（字符串形式）：
  map[0].type               = PIN_MAP_TYPE_MUX_GROUP
  map[0].name               = "default"
  map[0].dev_name           = "usart2"
  map[0].ctrl_dev_name      = "44240000.pinctrl"
  map[0].data.mux.group     = "PA4"           ← 字符串
  map[0].data.mux.function  = "af6"           ← 字符串
```

**入口文件：** `drivers/pinctrl/core.c:961`

```c
static int add_setting(struct pinctrl *p, struct pinctrl_dev *pctldev,
                       const struct pinctrl_map *map)
{
    struct pinctrl_state *state;
    struct pinctrl_setting *setting;
    int ret;

    // ↓ 根据 map->name 找到或创建对应的 pinctrl_state
    //   传入的 map->name = "default"（在 dt_remember_or_free_map 中写入的 statename）
    //   第一次遇到 "default" 时 create_state 创建一个新 state，之后第 2~4 个 map 直接 find_state 复用
    //   对 USART2：state "default" 最终下挂 4 个 setting（PA4_MUX/PA4_CONF/PA8_MUX/PA8_CONF）
    state = find_state(p, map->name);
    if (!state)
        state = create_state(p, map->name);
    if (IS_ERR(state))
        return PTR_ERR(state);

    // 跳过 dummy state（USART2 的 map 类型是 MUX_GROUP 或 CONFIGS_GROUP，不会走这里）
    if (map->type == PIN_MAP_TYPE_DUMMY_STATE)
        return 0;

    // ↓ 分配一个 pinctrl_setting 节点
    setting = kzalloc(sizeof(*setting), GFP_KERNEL);
    if (!setting)
        return -ENOMEM;

    setting->type = map->type;
    // 第一次调：type = PIN_MAP_TYPE_MUX_GROUP
    // 第二次调：type = PIN_MAP_TYPE_CONFIGS_GROUP

    // ↓ 找到这个 map 条目对应的 pin controller 设备
    //   create_pinctrl 传进来的 pctldev = NULL（USART2 非 hog 场景）
    //   所以走 else 分支：通过 map->ctrl_dev_name = "44240000.pinctrl" 查找
    //   get_pinctrl_dev_from_devname 遍历 pinctrldev_list 匹配设备名
    if (pctldev)
        setting->pctldev = pctldev;
    else
        setting->pctldev =
            get_pinctrl_dev_from_devname(map->ctrl_dev_name);
    if (!setting->pctldev) {
        kfree(setting);
        // 如果是 hog 条目（ctrl_dev_name == dev_name），返回 ENODEV
        if (!strcmp(map->ctrl_dev_name, map->dev_name))
            return -ENODEV;
        // 否则返回 EPROBE_DEFER——"44240000.pinctrl" 还没注册，USART2 probe 延后
        dev_info(p->dev, "unknown pinctrl device %s, deferring probe\n",
                 map->ctrl_dev_name);
        return -EPROBE_DEFER;
    }
    setting->dev_name = map->dev_name;   // "usart2"

    // ↓ 根据 map 类型做对应的转换
    //   对 MUX_GROUP：调 pinmux_map_to_setting，把 "PA4" → group_sel=4, "af6" → func=7
    //   对 CONFIGS_GROUP：调 pinconf_map_to_setting，把 "PA4" → group_sel=4，configs 数组直接拷贝
    switch (map->type) {
    case PIN_MAP_TYPE_MUX_GROUP:
        ret = pinmux_map_to_setting(map, setting);
        break;
    case PIN_MAP_TYPE_CONFIGS_PIN:
    case PIN_MAP_TYPE_CONFIGS_GROUP:
        ret = pinconf_map_to_setting(map, setting);
        break;
    default:
        ret = -EINVAL;
        break;
    }
    if (ret < 0) {
        kfree(setting);
        return ret;
    }

    // ↓ 把这个 setting 挂到 "default" state 的 settings 链表末尾
    //   第一次调后：state->settings = [PA4_MUX setting]
    //   第二次调后：state->settings = [PA4_MUX, PA4_CONF]
    //   第三次调后：state->settings = [PA4_MUX, PA4_CONF, PA8_MUX]
    //   第四次调后：state->settings = [PA4_MUX, PA4_CONF, PA8_MUX, PA8_CONF]
    list_add_tail(&setting->node, &state->settings);

    return 0;
}
```

> **注意**：`pctldev` 参数的存在是为了让 `create_pinctrl` 在已知 pctldev 的情况下直接传递（避免二次查找）。当从全局 `pinctrl_maps` 链表遍历时（`for_each_pin_map`），`pctldev` 传入 NULL，走 `get_pinctrl_dev_from_devname` 按名查找分支。

##### pinmux_map_to_setting — 字符串到整数选择器的转换

这个函数将 group 名和 function 名翻译成 pin controller 可以理解的整数选择器。

```
输入: map（字符串）                          输出: setting（整数）
  .data.mux.group    = "PA4"      ──────→    .data.mux.group = 4
  .data.mux.function = "af6"     ──────→    .data.mux.func   = 7
```

**入口文件：** `drivers/pinctrl/pinmux.c:353`

```c
// 输入 map: group="PA4"(字符串), function="af6"(字符串)
// 输出 setting: group_sel=4(整数), func_sel=7(整数)
int pinmux_map_to_setting(const struct pinctrl_map *map,
              struct pinctrl_setting *setting)
{
    struct pinctrl_dev *pctldev = setting->pctldev;
    const struct pinmux_ops *pmxops = pctldev->desc->pmxops;
    char const * const *groups;
    unsigned num_groups;
    int ret;
    const char *group;

    // ↓ 把 function 字符串 "af6" 转成整数 7
    //   内部遍历 pctldev 的 functions[] 数组（stm32_gpio_functions）
    //   逐个比较字符串，找到 "af6" 在索引 7 处，返回 selector = 7
    ret = pinmux_func_name_to_selector(pctldev, map->data.mux.function);
    if (ret < 0) {
        dev_err(pctldev->dev, "invalid function %s in map table\n",
            map->data.mux.function);
        return ret;
    }
    setting->data.mux.func = ret;       // → 7

    // ↓ 拿到 function 7 关联的所有 group 名列表
    //   STM32 驱动中每个 function 的 group 列表是 build_state 时建的
    //   function 7(af6)：groups = {"PA4", "PA5", ...} 这类 pin 名
    ret = pmxops->get_function_groups(pctldev, setting->data.mux.func,
                      &groups, &num_groups);
    if (ret < 0) return ret;

    if (!num_groups) {
        dev_err(pctldev->dev,
            "function %s can't be selected on any group\n",
            map->data.mux.function);
        return -EINVAL;
    }

    // ↓ 在 groups[] 中找 "PA4" 的索引（用于后续校验）
    //   match_string 遍历 groups[] 比对字符串，返回 "PA4" 在数组中的位置
    if (map->data.mux.group) {
        group = map->data.mux.group;         // → "PA4"
        ret = match_string(groups, num_groups, group);
        if (ret < 0) {
            dev_err(pctldev->dev,
                "invalid group \"%s\" for function \"%s\"\n",
                group, map->data.mux.function);
            return -EINVAL;
        }
    } else {
        group = groups[0];
    }

    // ↓ 把 group 字符串 "PA4" 转成整数 4
    //   内部遍历 pinctrl_dev->desc->pins[] 按 name 匹配
    //   找到 pin_desc.name == "PA4" 的条目，返回其索引
    ret = pinctrl_get_group_selector(pctldev, group);
    if (ret < 0) {
        dev_err(pctldev->dev, "invalid group %s in map table\n",
            map->data.mux.group);
        return ret;
    }
    setting->data.mux.group = ret;      // → 4

    return 0;
}
```

**CONFIGS map 的转换（`pinconf_map_to_setting`）同样需要查表：** group 名→整数，但 configs 数组直接拷贝。

```
输入 map（字符串）                          输出 setting（整数 + 直接拷贝）
  .data.configs.group_or_pin = "PA4"  ──→    .data.configs.group_or_pin = 4 (查表)
  .data.configs.configs = [...]       ──→   .data.configs.configs = [...] (原样)
  .data.configs.num_configs = 3       ──→   .data.configs.num_configs = 3
```

##### 第二次遍历完成后的状态

对于 USART2 的 12 个 map 条目，`add_setting` 执行 12 次，生成：

```
p (struct pinctrl *，USART2 设备的 pinctrl 句柄)
  │
  ├── states 链表:
  │     │
  │     ├── state "default"                  ← pinctrl_find_or_add_state 创建
  │     │    │                                   map->name="default"
  │     │    ├── setting (链表节点)
  │     │    │    type=MUX_GROUP             ← pins1 的 pinmux 属性
  │     │    │    .data.mux.func=7           (af6)
  │     │    │    .data.mux.group=4          (PA4)
  │     │    ├── setting (链表节点)
  │     │    │    type=CONFIGS_GROUP         ← pins1 的 bias/slew-rate 等
  │     │    │    .data.configs={bias-disable,
  │     │    │        drive-push-pull, slew-rate=0}
  │     │    │    .data.configs.group_sel=4
  │     │    ├── setting (链表节点)
  │     │    │    type=MUX_GROUP             ← pins2 的 pinmux 属性
  │     │    │    .data.mux.func=9           (af8)
  │     │    │    .data.mux.group=8          (PA8)
  │     │    └── setting (链表节点)
  │     │         type=CONFIGS_GROUP         ← pins2 的 bias-pull-up
  │     │         .data.configs={bias-pull-up}
  │     │         .data.configs.group_sel=8
  │     │
  │     ├── state "idle"
  │     │    ├── setting (链表节点)
  │     │    │    type=MUX_GROUP             (analog, PA4)
  │     │    │    .data.mux.func=17
  │     │    │    .data.mux.group=4
  │     │    ├── setting (链表节点)
  │     │    │    type=MUX_GROUP             (af8, PA8)
  │     │    │    .data.mux.func=9
  │     │    │    .data.mux.group=8
  │     │    └── setting (链表节点)
  │     │         type=CONFIGS_GROUP         (bias-pull-up, PA8)
  │     │
  │     └── state "sleep"
  │          ├── setting (链表节点)
  │          │    type=MUX_GROUP             (analog, PA4)
  │          │    .data.mux.func=17
  │          │    .data.mux.group=4
  │          └── setting (链表节点)
  │               type=MUX_GROUP             (analog, PA8)
  │               .data.mux.func=17
  │               .data.mux.group=8
  │
  ├── dt_maps 链表（struct pinctrl_dt_map 的链表）
  │    ├── dt_map[0]  → 对应 usart2_pins_a 的 4 个 map
  │    │    ├── .map = pinctrl_map[] [MUX/PA4, CONFIGS/PA4, MUX/PA8, CONFIGS/PA8]
  │    │    ├── .num_maps = 4
  │    │    └── .pctldev = "44240000.pinctrl" (释放时调 dt_free_map 回调)
  │    ├── dt_map[1]  → 对应 usart2_idle_pins_a
  │    ├── dt_map[2]  → 对应 usart2_sleep_pins_a
  │    └── 释放时 pinctrl_dt_free_maps 遍历此链表:
  │          ① pinctrl_unregister_mappings() → 从全局链表移除
  │          ② dt_free_map() → 释放 map[i].dev_name + 驱动私有数据
  │
  ├── dev     → USART2 device
  └── node    → 挂入全局 pinctrl_list 链表
```

> **关键理解：** map 和 setting 的区别就在**字符串 vs 整数**。map 是跨设备的通用格式（用字符串命名 group/function），setting 是特定 pinctrl_dev 的优化格式（用整数索引）。这就是"两次遍历"设计的核心——第一次将 DTS 翻译为通用 map，第二次将 map 适配到特定 pin controller 的 setting。这两步之间允许插入运行时修改 map 的机制（board 级别的 `pinctrl_map` 覆盖）。

至此，`create_pinctrl` 完成，返回填充好的 `pinctrl` 句柄。此时**还没有写任何硬件寄存器**——所有工作都是 DTS 解析和链表构建。真正的寄存器操作在 `pinctrl_select_state`。

#### 此时的状态

```
create_pinctrl 返回后:
  p->states     → [default, idle, sleep] 三个 state，每个下挂 settings
  p->dt_maps    → 保存 3 个 dt_map，各包裹一组 pinctrl_map[]
  p->dev        → USART2 device
  pinctrl_list  → 新节点：usart2

硬件寄存器: 未修改
  GPIOA MODER:    复位值（默认 0xFFFFFFFF → analog 模式，或 0x00000000 → 输入模式）
  GPIOA AFRL:     复位值（默认为 0x00000000 → AF0）
```

### 4.2.6 Stage 3: pinctrl_select_state — 最终执行

回到 `pinctrl_bind_pins`，在 `create_pinctrl` 和 `pinctrl_lookup_state` 之后：

```c
// pinctrl_bind_pins():53
dev->pins->init_state = pinctrl_lookup_state(dev->pins->p, PINCTRL_STATE_INIT);
if (IS_ERR(dev->pins->init_state)) {
    // ATK 板的 USART2 没有 init state
    ret = pinctrl_select_state(dev->pins->p,
                                dev->pins->default_state);
} else {
    ret = pinctrl_select_state(dev->pins->p, dev->pins->init_state);
}
```

USART2 在 DTS 中没有定义 `pinctrl-init`，所以 `pinctrl_lookup_state` 返回 `-ENOENT`，直接 select "default" state。

`pinctrl_select_state` 又调 `pinctrl_commit_state`：

```c
// drivers/pinctrl/core.c:1358
int pinctrl_select_state(struct pinctrl *p, struct pinctrl_state *state)
{
    if (p->state == state)
        return 0;
    return pinctrl_commit_state(p, state);
}
```

> **注意**：如果设备有多个 state（如 default/sleep/idle），`pinctrl_select_state` 可以在运行时被多次调用。比如系统休眠时调用 `pinctrl_select_state(p, p->sleep_state)`，激活时调用 `pinctrl_select_state(p, p->default_state)`。每次切换都会先释放旧 state 的 mux 设置，再应用新 state 的。

#### pinctrl_commit_state — 两阶段应用

这个函数接收 USART2 的 "default" state，将 state 中的 settings 逐个应用到硬件。

**输入 state 的内容（USART2 "default"）：**

```
state "default" 的 settings 链表有 4 个节点:
  setting[0]: type=MUX_GROUP,    func=7, group=4   ← 先执行
  setting[1]: type=CONFIGS_GROUP, configs={bias-disable,...}
  setting[2]: type=MUX_GROUP,    func=9, group=8
  setting[3]: type=CONFIGS_GROUP, configs={bias-pull-up}
```

**执行流程分两个 phase 遍历这 4 个 setting：**

```
Phase 1 (MUX 阶段): 遍历 4 个 setting
  setting[0]: MUX_GROUP → pinmux_enable_setting()  ✓ 执行
  setting[1]: CONFIGS_GROUP → ret=0 (跳过，等 Phase 2)
  setting[2]: MUX_GROUP → pinmux_enable_setting()  ✓ 执行
  setting[3]: CONFIGS_GROUP → ret=0 (跳过)

Phase 2 (CONFIGS 阶段): 再次遍历 4 个 setting
  setting[0]: MUX_GROUP → ret=0 (Phase 1 已处理)
  setting[1]: CONFIGS_GROUP → pinconf_apply_setting() ✓ 执行
  setting[2]: MUX_GROUP → ret=0 (Phase 1 已处理)
  setting[3]: CONFIGS_GROUP → pinconf_apply_setting() ✓ 执行
```

**为什么必须分两阶段？** 假设 PA4 当前是 analog 模式（复位默认），此时写 PUPDR 配置上拉是无效的——必须先通过 pinmux 切到 AF 模式，再配置 AF 模式下的偏置。Phase 1 先切功能，Phase 2 再配参数。

```c
// drivers/pinctrl/core.c:1256
static int pinctrl_commit_state(struct pinctrl *p, struct pinctrl_state *state)
{
    struct pinctrl_setting *setting, *setting2;
    struct pinctrl_state *old_state = READ_ONCE(p->state);
    int ret;

    // Phase 0: 如果之前有旧 state（如从 sleep 切回 default），
    //          先释放旧 state 中的 MUX 设置
    if (old_state) {
        list_for_each_entry(setting, &old_state->settings, node) {
            if (setting->type != PIN_MAP_TYPE_MUX_GROUP)
                continue;
            pinmux_disable_setting(setting);
        }
    }

    p->state = NULL;

    // ═══ Phase 1: 应用 MUX 设置 ═══
    // 只处理 type=MUX_GROUP，跳过 CONFIGS
    list_for_each_entry(setting, &state->settings, node) {
        switch (setting->type) {
        case PIN_MAP_TYPE_MUX_GROUP:
            ret = pinmux_enable_setting(setting);
            // → 第 1 次: setting[0], func=7, group=4 (PA4 af6) ✓
            // → 第 2 次: setting[2], func=9, group=8 (PA8 af8) ✓
            break;
        case PIN_MAP_TYPE_CONFIGS_PIN:
        case PIN_MAP_TYPE_CONFIGS_GROUP:
            ret = 0;    // 延迟到 Phase 2
            break;
        }
        if (ret < 0) goto unapply_new_state;

        /* 非 hog 设备 → 建立 pinctrl 链路（用于 probe 顺序） */
        if (p != setting->pctldev->p)
            pinctrl_link_add(setting->pctldev, p->dev);
    }

    // ═══ Phase 2: 应用配置 ═══
    // 只处理 type=CONFIGS_GROUP，跳过 MUX
    list_for_each_entry(setting, &state->settings, node) {
        switch (setting->type) {
        case PIN_MAP_TYPE_MUX_GROUP:
            ret = 0;    // Phase 1 已处理
            break;
        case PIN_MAP_TYPE_CONFIGS_PIN:
        case PIN_MAP_TYPE_CONFIGS_GROUP:
            ret = pinconf_apply_setting(setting);
            // → 第 1 次: setting[1], PA4, {bias-disable,...} ✓
            // → 第 2 次: setting[3], PA8, {bias-pull-up}   ✓
            break;
        }
        if (ret < 0) goto unapply_new_state;

        /* 同上 — 非 hog 设备建立链路 */
        if (p != setting->pctldev->p)
            pinctrl_link_add(setting->pctldev, p->dev);
    }

    p->state = state;
    return 0;
}
```

### 4.2.7 Stage 4: pinmux_enable_setting — pin 申请 + set_mux

**入口文件：** `drivers/pinctrl/pinmux.c:418`

```c
int pinmux_enable_setting(const struct pinctrl_setting *setting)
{
    struct pinctrl_dev *pctldev = setting->pctldev;
    const struct pinctrl_ops *pctlops = pctldev->desc->pctlops;
    const struct pinmux_ops *ops = pctldev->desc->pmxops;
    int ret = 0;
    const unsigned *pins = NULL;
    unsigned num_pins = 0;
    int i;
    struct pin_desc *desc;

    // ① 通过 group 获取该组包含的所有 pin 号
    //    STM32 驱动中：每个 group = 一个 pin，所以 num_pins = 1
    ret = pctlops->get_group_pins(pctldev, setting->data.mux.group,
                      &pins, &num_pins);
    // → pins = [4]  (PA4 的 pinctrl pin 号)
    //   num_pins = 1

    // ═══ 子步骤 1: 逐个 pin 申请（检查冲突 + 设置 owner）═══
    for (i = 0; i < num_pins; i++) {
        ret = pin_request(pctldev, pins[i], setting->dev_name, NULL);
        // → pin_request(pctldev, 4, "usart2", NULL)
    }

    // ═══ 子步骤 2: 记录 mux_setting 到 pin_desc ═══
    for (i = 0; i < num_pins; i++) {
        desc = pin_desc_get(pctldev, pins[i]);
        guard(mutex)(&desc->mux_lock);
        desc->mux_setting = &(setting->data.mux);
    }

    // ═══ 子步骤 3: 调用驱动的 set_mux 回调 ═══
    ret = ops->set_mux(pctldev, setting->data.mux.func,
                        setting->data.mux.group);
    // → ops->set_mux(pctldev, 7, 4)
    // → stm32_pmx_set_mux(pctldev, func_selector=7, group_selector=4)

    return 0;
}
```

#### pin_request — 冲突检测

```c
// gpio_range 只是一个类型标记，不是查表结果：
//   NULL  → mux 请求（场景一 USART2），检查/设置 desc->mux_owner
//   非空 → GPIO 请求（场景二 GPIOH5），检查/设置 desc->gpio_owner
// 同一 pin 不能同时被两种模式占用
static int pin_request(struct pinctrl_dev *pctldev,
               int pin, const char *owner,
               struct pinctrl_gpio_range *gpio_range)
{
    struct pin_desc *desc;
    const struct pinmux_ops *ops = pctldev->desc->pmxops;

    desc = pin_desc_get(pctldev, pin);

    guard(mutex)(&desc->mux_lock);

    // 冲突检查 1: 如果引脚已被其他设备 mux 占用
    if ((!gpio_range || ops->strict) &&
        desc->mux_usecount && strcmp(desc->mux_owner, owner)) {
        dev_err(pctldev->dev,
            "pin %s already requested by %s\n",
            desc->name, desc->mux_owner);
        goto out;
    }

    // 冲突检查 2: 如果引脚已被其他设备 GPIO 占用
    if ((gpio_range || ops->strict) && desc->gpio_owner) {
        dev_err(pctldev->dev,
            "pin %s already requested by %s\n",
            desc->name, desc->gpio_owner);
        goto out;
    }

    if (gpio_range) {
        desc->gpio_owner = owner;
    } else {
        desc->mux_usecount++;
        if (desc->mux_usecount > 1)
            return 0;
        desc->mux_owner = owner;
    }

    // ...
    return 0;
}
```

**USART2 PA4 调用 `pin_request` 时的实际跟踪：**

```
pin_request(pctldev, pin=4, owner="usart2", gpio_range=NULL)
  │
  ├── desc = pin_desc_get(pctldev, 4)      → 找到 PA4 的 pin_desc
  │
  ├── 冲突检查 1: (!NULL || strict) && mux_usecount && owner不同?
  │       gpio_range=NULL → !gpio_range=true
  │       ops->strict 由驱动决定（STM32MP2 是否启用 strict）
  │       mux_usecount == 0 → 条件 false → 通过 ✓
  │
  ├── 冲突检查 2: (NULL || strict) && gpio_owner?
  │       gpio_range=NULL → false（因 gpio_range 为 NULL 时跳过此检查）
  │
  └── gpio_range == NULL → 走 mux 分支:
        desc->mux_usecount++  → 0 → 1
        if (mux_usecount > 1) → false (第一次申请)
        desc->mux_owner = "usart2"   ✓ 申请成功
```

如果 PA4 已经被其他设备（如 SPI2）申请为 mux 引脚，则 `pin_request` 返回错误，`pinctrl_bind_pins` 返回 `-EPROBE_DEFER`，USART2 的 probe 被延迟。设备核心会在 SPI2 probe 释放引脚后重新尝试。

#### stm32_pmx_set_mux — 找到 bank 和寄存器参数

**入口文件：** `pinctrl-stm32.c:1098`

```c
static int stm32_pmx_set_mux(struct pinctrl_dev *pctldev,
                unsigned function,              // 7 = "af6"
                unsigned group)                 // 4 = PA4
{
    struct stm32_pinctrl *pctl = pinctrl_dev_get_drvdata(pctldev);
    struct stm32_pinctrl_group *g = pctl->groups + group;
    // → g->pin = 4, g->name = "PA4"

    struct pinctrl_gpio_range *range;
    struct stm32_gpio_bank *bank;
    u32 mode, alt;
    int pin;

    // ① 检查 function 对 pin 是否合法
    if (!stm32_pctrl_is_function_valid(pctl, g->pin, function))
        return -EINVAL;

    // ② 跳过 reserved pin（硬件保留，不操作寄存器）
    if (function == STM32_PIN_RSVD)
        return 0;

    // ↓ ③ pinctrl pin 号(4) → GPIO bank 映射（跨子系统查找）
    //   pinctrl_find_gpio_range_from_pin 遍历 pctldev->gpio_ranges 链表，
    //   找哪个 GPIO range 包含了 pinctrl pin 4。
    //   对 PA4：GPIOA 的 range->pin_base=0, npins=16，4 在 [0,15] 范围内 → 匹配
    //   返回的 range->gc 是 GPIOA 的 gpio_chip，
    //   gpiochip_get_data(range->gc) 得到 stm32_gpio_bank，bank->base = 0x44240000
    range = pinctrl_find_gpio_range_from_pin(pctldev, g->pin);
    if (!range)
        return -EINVAL;
    bank = gpiochip_get_data(range->gc);

    // ↓ ④ pinctrl pin 号(4) → GPIO bank 内偏移(4)
    //   stm32_gpio_pin(g->pin) = g->pin % 16 = 4 % 16 = 4
    //   这是写寄存器时需要的位域偏移——MODER bits[9:8], AFRL bits[19:16]
    pin = stm32_gpio_pin(g->pin);

    // ↓ ⑤ function selector(7) → mode(2) + alt(6)
    //   stm32_gpio_get_mode(7)  → 7 ∈ [1,16] → 返回 2 (AF 模式)
    //   stm32_gpio_get_alt(7)   → 7-1 → 返回 6 (硬件 AF6 值)
    mode = stm32_gpio_get_mode(function);
    alt  = stm32_gpio_get_alt(function);

    // ↓ ⑥ 写寄存器：先写 AFRL（选 AF6），再写 MODER（切到 AF 模式）
    return stm32_pmx_set_mode(bank, pin, mode, alt);
}
```

> **两个子系统的 pin 映射**：pinctrl 层（`g->pin=4`）和 GPIO 层（`bank->base + offset=4`）是两套编号。`pinctrl_find_gpio_range_from_pin` 通过 probe 时建立的 `pinctrl_gpio_range` 链表，把 pinctrl pin 号翻译到 GPIO chip + bank 结构体；`stm32_gpio_pin` 算出 bank 内的位偏移。两者加起来才能定位到具体的硬件寄存器位域。

#### stm32_pmx_set_mode — 最终写寄存器

**入口文件：** `pinctrl-stm32.c:1035`

这是整个调用链的终点——直接读写 GPIO 寄存器的函数。

```c
static int stm32_pmx_set_mode(struct stm32_gpio_bank *bank,
                  int pin, u32 mode, u32 alt)
{
    struct stm32_pinctrl *pctl = dev_get_drvdata(bank->gpio_chip.parent);
    u32 val;
    int alt_shift = (pin % 8) * 4;
    int alt_offset = STM32_GPIO_AFRL + (pin / 8) * 4;
    unsigned long flags;
    int err = 0;

    spin_lock_irqsave(&bank->lock, flags);

    if (pctl->hwlock) {
        err = hwspin_lock_timeout_in_atomic(pctl->hwlock, HWSPNLCK_TIMEOUT);
        if (err) goto unlock;
    }

    // ═══ 第一步：写 AFRL 或 AFRH 寄存器 ═══
    val = readl_relaxed(bank->base + alt_offset);   // 读当前值
    val &= ~GENMASK(alt_shift + 3, alt_shift);       // 清除目标引脚的 4 位
    val |= (alt << alt_shift);                        // 写入 alt 值
    writel_relaxed(val, bank->base + alt_offset);    // 写回硬件

    // ═══ 第二步：写 MODER 寄存器 ═══
    val = readl_relaxed(bank->base + STM32_GPIO_MODER);
    val &= ~GENMASK(pin * 2 + 1, pin * 2);
    val |= mode << (pin * 2);
    writel_relaxed(val, bank->base + STM32_GPIO_MODER);

    // 备份到内存（用于 get_direction 查询）
    stm32_gpio_backup_mode(bank, pin, mode, alt);

    if (pctl->hwlock)
        hwspin_unlock_in_atomic(pctl->hwlock);

unlock:
    spin_unlock_irqrestore(&bank->lock, flags);
    return err;
}
```

**寄存器偏移定义：**

```c
// pinctrl-stm32.c:41-49
#define STM32_GPIO_MODER    0x00   // 模式寄存器（2 bits per pin）
#define STM32_GPIO_IDR      0x10   // 输入数据寄存器
#define STM32_GPIO_BSRR     0x18   // 位设置/复位寄存器
#define STM32_GPIO_AFRL     0x20   // 复用功能低寄存器（引脚 0~7）
#define STM32_GPIO_AFRH     0x24   // 复用功能高寄存器（引脚 8~15）
```

**`stm32_gpio_get_mode` 和 `stm32_gpio_get_alt` 的映射表：**

```c
// pinctrl-stm32.c:186 (get_mode) / 200 (get_alt)
static inline u32 stm32_gpio_get_mode(u32 function)
{
    switch (function) {
    case STM32_PIN_GPIO:           // 0     → GPIO 模式
        return 0;
    case STM32_PIN_AF(0) ... STM32_PIN_AF(15):  // 1~16 → AF 模式
        return 2;
    case STM32_PIN_ANALOG:         // 17    → 模拟模式
        return 3;
    }
    return 0;
}

static inline u32 stm32_gpio_get_alt(u32 function)
{
    switch (function) {
    case STM32_PIN_GPIO:
        return 0;
    case STM32_PIN_AF(0) ... STM32_PIN_AF(15):
        return function - 1;   // AF(6)=7 → 返回 6
    case STM32_PIN_ANALOG:
        return 0;
    }
    return 0;
}
```

**数组索引与硬件 AF 值的关系：**

| 数组索引 (selector) | 函数名 | 对应 AF | `stm32_gpio_get_alt(selector)` |
|:---:|:---:|:---:|:---:|
| 0 | "gpio" | GPIO | 0 |
| 1 | "af0" | AF0 | 0 |
| 2 | "af1" | AF1 | 1 |
| ... | ... | ... | ... |
| **7** | **"af6"** | **AF6** | **6** |
| 8 | "af7" | AF7 | 7 |
| ... | ... | ... | ... |
| 16 | "af15" | AF15 | 15 |
| 17 | "analog" | ANALOG | 0 |

DTS 中 `AF6 = 0x7`，解码后 fnum=7 → `stm32_gpio_functions[7] = "af6"` → pinctrl selector=7 → `stm32_gpio_get_alt(7) = 6` → **AFRL 写入 6（硬件 AF6）**。

#### 针对 PA4 AF6 的寄存器操作展开

PA4 是 GPIOA 的第 5 个引脚（偏移 4，从 0 计数），处于 AFRL 范围内（引脚 0~7）。

**进入 `stm32_pmx_set_mode` 时的参数：**

```
bank = GPIOA bank  (base = 0x44240000)
pin  = 4
mode = 2    (alternate function mode)
alt  = 6    (AF6 = USART2_TX)
```

**第一步：计算寄存器地址和位偏移**

```
alt_shift = (pin % 8) * 4 = (4 % 8) * 4 = 16
   → PA4 的 AFRL 位段从 bit 16 开始，占 4 位 (bits 16-19)

alt_offset = STM32_GPIO_AFRL + (pin / 8) * 4
           = 0x20 + (4 / 8) * 4 = 0x20 + 0 = 0x20
   → AFRL 寄存器（0x20），因为 pin < 8
```

**第二步：写 AFRL 寄存器**

```
读: val = readl(0x44240000 + 0x20)                    // 读当前 AFRL
   = 0xXXXXXXXX

清: val &= ~GENMASK(16+3, 16)  =  val & ~0x000F0000  // 清 bits 16-19

设: val |= (6 << 16) = val | 0x00060000              // bits 16-19 = 0110

写: writel(0x00060000, 0x44240000 + 0x20)            // 写回
```

**第三步：写 MODER 寄存器**

```
pin * 2 = 4 * 2 = 8
GENMASK(8+1, 8) = 0x00000300  (bits 8-9)

读: val = readl(0x44240000 + 0x00)                    // 读当前 MODER
清: val &= ~0x00000300                                // 清 bits 8-9
设: val |= (2 << 8) = val | 0x00000200                // bits 8-9 = 10b
写: writel(0x00000200, 0x44240000 + 0x00)             // 写回
```

> **MODER 编码：**
> - `0b00` = 输入模式（Input）
> - `0b01` = 输出模式（Output）
> - `0b10` = 复用功能模式（Alternate Function）
> - `0b11` = 模拟模式（Analog）

**最终硬件状态：**

```
GPIOA MODER (@ 0x44240000):
  bits 9:8 = 0b10  → PA4 = Alternate Function mode

GPIOA AFRL  (@ 0x44240020):
  bits 19:16 = 0b0110  → PA4 Alternate Function = AF6
```

#### 再验证 PA8 AF8

**进入 `stm32_pmx_set_mode` 时的参数：**

```
bank = GPIOA bank
pin  = 8
mode = 2    (alternate function mode)
alt  = 8    (AF8 = USART2_RX)
```

**第一步：计算**

```
alt_shift = (8 % 8) * 4 = 0          ← PA8 是 AFRH 的第 0 个 4 位段
alt_offset = 0x20 + (8 / 8) * 4 = 0x24  ← AFRH 寄存器（0x24）
```

**第二步：写 AFRH（0x24）**

```
读: val = readl(0x44240000 + 0x24)
清: val &= ~0x0000000F
设: val |= (8 << 0) = 0x00000008    // AF8 = 1000b
写: writel(0x00000008, 0x44240000 + 0x24)
```

**第三步：写 MODER**

```
pin * 2 = 8 * 2 = 16
GENMASK(16+1, 16) = 0x00030000  (bits 16-17)

读: val = readl(0x44240000 + 0x00)
清: val &= ~0x00030000
设: val |= (2 << 16) = 0x00020000
写: writel(0x00020000, 0x44240000 + 0x00)
```

**MODER 最终值（只关心 PA4 和 PA8）：**

```
GPIOA MODER (@ 0x44240000)
  bit  9:8 = 0b10  → PA4 = AF mode (USART2_TX)
  bit 17:16 = 0b10 → PA8 = AF mode (USART2_RX)
```

### 4.2.8 下一步：pinconf 阶段

MUX 配置完成后，`pinctrl_commit_state` 进入 Phase 2，遍历剩余的 `PIN_MAP_TYPE_CONFIGS_GROUP` setting，调用 `pinconf_apply_setting`。

对 USART2 的 `pins1` 子节点，有以下配置：

```dts
pins1 {                          /* PA4 */
    pinmux = <STM32_PINMUX('A', 4, AF6)>;
    bias-disable;                /* 无上下拉 */
    drive-push-pull;             /* 推挽输出 */
    slew-rate = <0>;             /* 低速 */
};
```

`pinconf_apply_setting` 最终调 STM32 驱动的 `stm32_pconf_set` 函数，该函数操作 GPIO 的以下寄存器：

| 配置 | 寄存器 | 偏移 | 位域 |
|------|--------|------|------|
| bias-disable/pull-up/pull-down | PUPDR | 0x0C | 每 pin 2 bits |
| drive-push-pull/open-drain | OTYPER | 0x04 | 每 pin 1 bit |
| slew-rate | OSPEEDR | 0x08 | 每 pin 2 bits |

对于 PA4：

```
PUPDR (@ 0x4424000C):
  bits 9:8 = 0b00 → no pull-up/pull-down (bias-disable)

OTYPER (@ 0x44240004):
  bit 4 = 0 → push-pull (drive-push-pull)

OSPEEDR (@ 0x44240008):
  bits 9:8 = 0b00 → low speed (slew-rate = <0>)
```

对于 PA8：

```dts
pins2 {                          /* PA8 */
    pinmux = <STM32_PINMUX('A', 8, AF8)>;
    bias-pull-up;                /* 上拉 */
};
```

```
PUPDR (@ 0x4424000C):
  bits 17:16 = 0b01 → pull-up (bias-pull-up)
```

**pinconf_apply_setting 源码分析**（pinconf.c:148）

这是 pinctrl core 层的通用分发函数——根据 setting 类型调驱动回调，看不到硬件细节：

```c
int pinconf_apply_setting(const struct pinctrl_setting *setting)
{
    struct pinctrl_dev *pctldev = setting->pctldev;
    const struct pinconf_ops *ops = pctldev->desc->confops;

    // STM32 驱动注册了 confops，不会走这里
    if (!ops) { dev_err(...); return -EINVAL; }

    switch (setting->type) {
    case PIN_MAP_TYPE_CONFIGS_GROUP:      // USART2 走这分支
        ret = ops->pin_config_group_set(pctldev,
                setting->data.configs.group_or_pin,   // group_sel=4 (PA4)
                setting->data.configs.configs,        // {bias-disable, drive-push-pull, slew-rate=0}
                setting->data.configs.num_configs);   // 3
        break;
    }
    return ret;
}
```

**stm32_pconf_group_set**（pinctrl-stm32.c:1539） 和 **stm32_pconf_set**（pinctrl-stm32.c:1561）

`ops->pin_config_group_set` 对应 STM32 驱动的 `stm32_pconf_group_set`，它遍历 group 内的每个 pin 调 `stm32_pconf_parse_conf`。因为 STM32 一 pin 一 group，每次只处理一个 pin。`stm32_pconf_set` 是单 pin 版本，逻辑相同：

```c
// stm32_pconf_set：遍历 3 个 pinconf 配置，逐个解析+写寄存器
static int stm32_pconf_set(struct pinctrl_dev *pctldev, unsigned int pin,
               unsigned long *configs, unsigned int num_configs)
{
    int i, ret;

    for (i = 0; i < num_configs; i++) {
        // pinconf_to_config_param 取配置类型（如 BIAS_DISABLE）
        // pinconf_to_config_argument 取参数值（如 slew-rate 的数值）
        ret = stm32_pconf_parse_conf(pctldev, pin,
                pinconf_to_config_param(configs[i]),
                pinconf_to_config_argument(configs[i]));
        if (ret < 0)
            return ret;
    }
    return 0;
}
```

对 PA4，循环 3 次：
```
i=0: param=BIAS_DISABLE, arg=0      → stm32_pconf_set_bias → PUPDR bits[9:8]=00
i=1: param=DRIVE_PUSH_PULL, arg=0   → stm32_pconf_set_driving → OTYPER bit4=0
i=2: param=SLEW_RATE, arg=0         → stm32_pconf_set_speed → OSPEEDR bits[9:8]=00
```

**stm32_pconf_parse_conf**（pinctrl-stm32.c:1456）

这是真正的寄存器操作分发器——根据 param 类型调对应的写寄存器函数：

```c
static int stm32_pconf_parse_conf(struct pinctrl_dev *pctldev,
        unsigned int pin, enum pin_config_param param,
        enum pin_config_param arg)
{
    // 从 pinctrl pin 号找到 GPIO bank + 偏移（和 stm32_pmx_set_mux 中的查找一样）
    range = pinctrl_find_gpio_range_from_pin_nolock(pctldev, pin);
    bank = gpiochip_get_data(range->gc);
    offset = stm32_gpio_pin(pin);    // pinctrl pin 号 → bank 内偏移

    switch (param) {
    case PIN_CONFIG_BIAS_DISABLE:
        ret = stm32_pconf_set_bias(bank, offset, 0);  // PUPDR[9:8] = 00
        break;
    case PIN_CONFIG_BIAS_PULL_UP:
        ret = stm32_pconf_set_bias(bank, offset, 1);  // PUPDR[17:16] = 01
        break;
    case PIN_CONFIG_DRIVE_PUSH_PULL:
        ret = stm32_pconf_set_driving(bank, offset, 0); // OTYPER[4] = 0
        break;
    case PIN_CONFIG_SLEW_RATE:
        ret = stm32_pconf_set_speed(bank, offset, arg);  // OSPEEDR[9:8] = arg
        break;
    // 还有 BIAS_PULL_DOWN、DRIVE_OPEN_DRAIN 等
    }
}
```

### 4.2.9 ATK 板实例完全验证

**设备：** USART2（console serial），ATK 板 `stm32mp257d-atk-bsp.dts`

**引脚：** PA4 = USART2_TX, PA8 = USART2_RX

**DTS 中 pinmux 值：**

| 引脚 | DTS 配置 | PINMUX 编码 | pinctrl pin# | GPIO 全局# |
|------|---------|------------|-------------|-----------|
| PA4 | `STM32_PINMUX('A', 4, AF6)` | 0x407 | 4 | 4 |
| PA8 | `STM32_PINMUX('A', 8, AF8)` | 0x809 | 8 | 8 |

**执行 `pinctrl_select_state` 后，GPIOA 的寄存器最终状态：**

```
GPIOA 寄存器基址: 0x44240000

MODER  (0x00):  0x00020200
  位 9:8   = 0b10 → PA4 = Alternate Function
  位 17:16 = 0b10 → PA8 = Alternate Function

AFRL   (0x20):  0x00060000
  位 19:16 = 0b0110 → PA4 = AF6 (USART2_TX)

AFRH   (0x24):  0x00000008
  位 3:0  = 0b1000 → PA8 = AF8 (USART2_RX)

PUPDR  (0x0C):  0x00010000
  位 9:8   = 0b00 → PA4 = no pull (bias-disable)
  位 17:16 = 0b01 → PA8 = pull-up (bias-pull-up)

OTYPER (0x04):  0x00000000
  位 4     = 0 → PA4 = push-pull

OSPEEDR(0x08):  0x00000000
  位 9:8   = 0b00 → PA4 = low speed
```

### 4.2.10 场景一总结

```
DTS 描述                    内核运行时                   硬件寄存器
──────────────────────────────────────────────────────────────────
STM32_PINMUX('A',4,AF6)    pinctrl_map→setting          MODER[9:8]=0b10
  │                        →pinmux_enable_setting()     AFRL[19:16]=0110
  │                        →stm32_pmx_set_mode()        PUPDR[9:8]=00
  └── bias-disable         →pinconf_apply_setting()      OTYPER[4]=0
     drive-push-pull       →stm32_pconf_set()            OSPEEDR[9:8]=00
     slew-rate=<0>
```

关键要点总结：

1. **两次遍历设计**：`create_pinctrl` 先通过 `pinctrl_dt_to_map` 将 DTS 解析为通用 map，再通过 `add_setting` 将 map 转为内核 setting。DTS 解析是 pin controller 驱动的回调（STM32 的 `stm32_pctrl_dt_node_to_map`），setting 转换是 pinctrl core 的逻辑。

2. **两阶段提交**：`pinctrl_commit_state` 分两阶段应用设置——先 MUX（`pinmux_enable_setting`）后配置（`pinconf_apply_setting`）。这个顺序不是随意的：引脚必须先切到目标功能才能应用该功能的电气配置。

3. **两次硬件写入**：`stm32_pmx_set_mode` 先写 AFRx（选择 alternate function 编号），后写 MODER（确认切换模式）。即使 MODER 当前是 0（输入），AFRx 也必须先准备好，因为模式切换发生在一瞬间。

4. **每个引脚是一个 group**：STM32 驱动中每个引脚单独成组（`stm32_pctrl_build_state` 时一 pin 一组），所以 `get_group_pins` 总是返回 1 个引脚。

## 4.3 场景二：`/dev/gpiochip` 用户态 GPIO 控制

### 4.3.1 从一个问题开始

在 ATK 板终端上执行：

```bash
$ gpioset gpiochip7 5=1        # 点亮用户 LED（GPIOH5 → bank 7, offset 5）
$ gpioget gpiochip0 3          # 读取 GPIOA3 的电平
$ gpiomon gpiochip6 5          # 监控 GPIOG5 的中断事件
```

这些命令如何最终操作到 GPIO 寄存器的 BSRR、IDR 等硬件？

**ATK 板 DTS 中定义的硬件：**

```dts
// stm32mp257d-atk-bsp.dts:78
gpio-leds {
    compatible = "gpio-leds";
    led {
        function = LED_FUNCTION_HEARTBEAT;
        color = <LED_COLOR_ID_RED>;
        gpios = <&gpioh 5 GPIO_ACTIVE_LOW>;  // GPIOH5
        linux,default-trigger = "heartbeat";
    };
};
```

**GPIOH 的硬件信息（来自 ATK 板 DTSI）：**

```dts
// stm32mp25xxak-pinctrl.dtsi:52
gpioh: gpio@442b0000 {
    status = "okay";
    ngpios = <12>;                              // 只有 12 个引脚
    gpio-ranges = <&pinctrl 2 114 12>;          // bank 内偏移 2~13
};
```

**GPIOH bank 参数：**

| 参数 | 值 | 来源 |
|------|-----|------|
| bank_nr | 7 | 114 / 16 = 7 |
| 寄存器基址 | 0x442b0000 | DTS reg = <0x70000 0x400> + ranges |
| bank 内引脚数 | 12 | ngpios |
| pinctrl pin 号范围 | 114~125 | gpio-ranges: pin_base=114, npins=12 |

### 4.3.2 场景总览

**场景在做什么**

用户态程序（如 `gpioset`/`gpioget`/`gpiomon`）通过 `/dev/gpiochipN` 字符设备操作 GPIO——打开设备、ioctl 请求 line、然后读写电平或监听中断。整个过程从用户态 system call 一路下沉到 GPIO 寄存器：

| 层 | 做了什么 | 输出 |
|---|---------|------|
| **用户态** | libgpiod 封装 `open/ioctl` 系统调用 | 系统调用陷入内核 |
| **gpiolib cdev** | 处理 `ioctl` 路由，分配 `linereq`，管理 line 请求 | 返回 fd，设置方向/中断 |
| **gpiolib core** | 管理 `gpio_desc`，方向/电平操作的通用逻辑 | 调用 `gpio_chip` 回调 |
| **STM32 驱动** | 实现 `gpio_chip` 回调：`set`→写 BSRR，`get`→读 IDR，`direction_output`→写 MODER | 物理寄存器被操作 |

**最终结果**

```
输出模式:  MODER bits[11:10] = 0b01 → GPIOH5 为输出模式
写电平:    BSRR @ 0x442b0000 写 BIT(5) → GPIOH5 输出高电平
读电平:    IDR @ 0x442b0000 读 BIT(5) → 返回 0 或 1
中断:      gpiod_to_irq() → irq_create_fwspec_mapping() → Linux IRQ 号
```

用户态到硬件的完整路径可以概括为 5 步：

```
gpioset gpiochip7 5=1
  → ① open("/dev/gpiochip7")     ← 字符设备
    → ② ioctl(GET_LINE_IOCTL)    ← 请求 line 5
      → ③ gpiod_direction_output ← 设方向
        → ④ gpiod_set_value      ← 设电平（或 ioctl 设值）
          → ⑤ writel(BSRR)       ← 写硬件寄存器
```

### 4.3.3 路径全貌

```
用户态 (gpioset/gpioget/gpiomon 或自定义程序)
  │
  ├── open("/dev/gpiochip0")
  │     └─ gpio_chrdev_open()          ← gpiolib-cdev.c:2881
  │          → 分配 gpio_chardev_data
  │          → file->private_data = cdev
  │
  ├── ioctl(GPIO_V2_GET_LINE_IOCTL)    ← 请求一行 GPIO（v2 API）
  │     └─ gpio_ioctl_unlocked()
  │          → linereq_create(gdev, ip) ← gpiolib-cdev.c:1812
  │               ├─ gpiod_request_user(desc, label) → gpiolib 申请
  │               ├─ gpiod_direction_output(desc, val)  [输出方式]
  │               │  └─ stm32_gpio_direction_output()
  │               │       → __stm32_gpio_set() 写 BSRR   ← 设置初始电平
  │               │       → stm32_pmx_gpio_set_direction() 写 MODER
  │               └─ gpiod_direction_input(desc)   [输入方式]
  │                 └─ stm32_pmx_gpio_set_direction() 写 MODER
  │
  ├── 返回 fd → 通过 fd 操作 GPIO
  │
  ├── ioctl(fd, GPIO_V2_LINE_SET_VALUES_IOCTL, {mask, bits})  ← 设电平
  │     └─ linereq_set_values()
  │          → gpiod_set_raw_value_commit()
  │               → gc->set() → stm32_gpio_set()
  │                    → writel_relaxed(BIT(offset), BSRR)   ← ★
  │
  ├── ioctl(fd, GPIO_V2_LINE_GET_VALUES_IOCTL, {mask})     ← 读电平
  │     └─ linereq_get_values()
  │          → gpiod_get_raw_value_commit()
  │               → gc->get() → stm32_gpio_get()
  │                    → readl_relaxed(IDR) & BIT(pin)   ← ★
  │
  └── ioctl(fd, GPIO_V2_LINE_SET_CONFIG_IOCTL, ...)  ← 配置/中断
       └─ edge_detector_setup()
            → gpiod_to_irq() → stm32_gpio_to_irq()
            → request_threaded_irq() → EXTI → GIC
```

### 4.3.4 字符设备的创建

`/dev/gpiochipN` 设备的创建发生在 03 中讲的 `gpiochip_add_data()` 流程内部。

```c
// drivers/gpio/gpiolib-cdev.c:2982
int gpiolib_cdev_register(struct gpio_device *gdev, dev_t devt)
{
    int ret;

    // ① 初始化 cdev，关联 file_operations
    cdev_init(&gdev->chrdev, &gpio_fileops);
    // → gpio_fileops 中定义了 .open = gpio_chrdev_open
    //                  .unlocked_ioctl = gpio_ioctl

    gdev->chrdev.owner = THIS_MODULE;
    gdev->dev.devt = MKDEV(MAJOR(devt), gdev->id);

    // ② 将 cdev 和 device 同时添加到内核
    ret = cdev_device_add(&gdev->chrdev, &gdev->dev);
    // → 内部创建 /dev/gpiochipN 节点

    chip_dbg(gdev->chip, "added GPIO chardev (%d:%d)\n",
         MAJOR(devt), gdev->id);

    return 0;
}
```

文件操作结构体：

```c
// drivers/gpio/gpiolib-cdev.c:2969
static const struct file_operations gpio_fileops = {
    .release = gpio_chrdev_release,
    .open    = gpio_chrdev_open,
    .poll    = lineinfo_watch_poll,
    .read    = lineinfo_watch_read,
    .owner   = THIS_MODULE,
    .llseek  = no_llseek,
    .unlocked_ioctl = gpio_ioctl,
#ifdef CONFIG_COMPAT
    .compat_ioctl   = gpio_ioctl_compat,
#endif
};
```

**注册后 `/dev` 下的设备：**

```
$ ls -l /dev/gpiochip*
crw------- 1 root root 254, 0 Jan 1  1970 /dev/gpiochip0   ← GPIOA
crw------- 1 root root 254, 1 Jan 1  1970 /dev/gpiochip1   ← GPIOB
...
crw------- 1 root root 254, 7 Jan 1  1970 /dev/gpiochip7   ← GPIOH
```

每个 GPIO bank 一个独立的字符设备。设备号主编号相同（254），次编号为 bank 的注册顺序（gpio_device.id）。

### 4.3.5 open("/dev/gpiochipN") — 初始化 chardev 数据

```c
// drivers/gpio/gpiolib-cdev.c:2881
static int gpio_chrdev_open(struct inode *inode, struct file *file)
{
    struct gpio_device *gdev = container_of(inode->i_cdev,
                        struct gpio_device, chrdev);
    struct gpio_chardev_data *cdev;
    int ret = -ENOMEM;

    // ① 检查 chip 是否已被移除
    if (!gdev->chip) { ret = -ENODEV; goto out_unlock; }

    // ② 分配 chardev 数据
    cdev = kzalloc(sizeof(*cdev), GFP_KERNEL);
    cdev->watched_lines = bitmap_zalloc(gdev->chip->ngpio, GFP_KERNEL);
    init_waitqueue_head(&cdev->wait);
    INIT_KFIFO(cdev->events);
    cdev->gdev = gpio_device_get(gdev);

    // ③ 注册通知链
    cdev->lineinfo_changed_nb.notifier_call = lineinfo_changed_notify;
    blocking_notifier_chain_register(&gdev->line_state_notifier,
                      &cdev->lineinfo_changed_nb);

    // ④ 保存到 file->private_data
    file->private_data = cdev;

    ret = nonseekable_open(inode, file);
    return ret;
}
```

**打开后的关键数据结构关系：**

```
file  (用户态 fd)
  └─ private_data → gpio_chardev_data
                       ├─ gdev → gpio_device → gpio_chip → stm32_gpio_bank
                       ├─ watched_lines: bitmap (监听哪些 line 的信息变更)
                       └─ events: event FIFO (line 状态变更事件)
```

### 4.3.6 ioctl 路由 — 从 gpio_ioctl 到 linereq_create

用户调用 `ioctl(fd, GPIO_V2_GET_LINE_IOCTL, &req)` 时，内核 VFS 层调用 `gpio_fileops.unlocked_ioctl`，即 `gpio_ioctl`。

```c
// drivers/gpio/gpiolib-cdev.c:2705
static long gpio_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
{
    struct gpio_chardev_data *cdev = file->private_data;
    return call_ioctl_locked(file, cmd, arg, cdev->gdev,
                  gpio_ioctl_unlocked);
}

static long gpio_ioctl_unlocked(struct file *file, unsigned int cmd,
                 unsigned long arg)
{
    struct gpio_chardev_data *cdev = file->private_data;
    struct gpio_device *gdev = cdev->gdev;
    void __user *ip = (void __user *)arg;

    if (!gdev->chip) return -ENODEV;

    switch (cmd) {
    case GPIO_GET_CHIPINFO_IOCTL:
        return chipinfo_get(cdev, ip);
    case GPIO_V2_GET_LINEINFO_IOCTL:
        return lineinfo_get(cdev, ip, false);
    case GPIO_V2_GET_LINEINFO_WATCH_IOCTL:
        return lineinfo_get(cdev, ip, true);
    case GPIO_V2_GET_LINE_IOCTL:
        return linereq_create(gdev, ip);              // ★ 请求 GPIO line
    case GPIO_GET_LINEINFO_UNWATCH_IOCTL:
        return lineinfo_unwatch(cdev, ip);
    default:
        return -EINVAL;
    }
}
```

#### linereq_create — 请求 GPIO line（v2 API）

这是用户态获取 GPIO 访问权限的核心函数。libgpiod 的 `gpioset`/`gpioget`/`gpiomon` 都通过这个 ioctl 请求 GPIO。

**用户执行 `gpioset gpiochip7 5=1` 时，libgpiod 构造的 ioctl 参数：**

```c
struct gpio_v2_line_request req = {
    .offsets     = { 5, 0, 0, 0, ... },    // ← GPIOH bank 内偏移 5
    .consumer    = "gpioset",               // ← 消费者名称
    .config = {
        .flags  = GPIO_V2_LINE_FLAG_OUTPUT, // ← 输出方向
    },
    .num_lines = 1,
};
```

**入口文件：** `gpiolib-cdev.c:1812`

```c
static int linereq_create(struct gpio_device *gdev, void __user *ip)
{
    struct gpio_v2_line_request ulr;
    struct linereq *lr;
    int fd, ret;
    unsigned int i;

    // ① 从用户态复制请求参数
    if (copy_from_user(&ulr, ip, sizeof(ulr)))
        return -EFAULT;
    // ulr.offsets[0]=5, ulr.consumer="gpioset", ulr.num_lines=1

    // ② 参数验证
    if ((ulr.num_lines == 0) || (ulr.num_lines > GPIO_V2_LINES_MAX))
        return -EINVAL;

    // ③ 分配 linereq 结构体
    lr = kzalloc(struct_size(lr, lines, ulr.num_lines), GFP_KERNEL);
    lr->gdev = gpio_device_get(gdev);
    lr->num_lines = 1;

    if (ulr.consumer[0] != '\0') {
        lr->label = kstrndup("gpioset", ...);
    }

    // ═══ ④ 逐个 line 请求 GPIO ═══
    for (i = 0; i < ulr.num_lines; i++) {
        u32 offset = ulr.offsets[i];              // → 5 (GPIOH5)

        desc = gpiochip_get_desc(gdev->chip, 5);
        // → desc = &gdev->descs[5]

        ret = gpiod_request_user(desc, "gpioset");
        if (ret) goto out_free_linereq;

        lr->lines[i].desc = desc;

        // 配置 flags 和方向
        flags = gpio_v2_line_config_flags(&ulr.config, 0);  // OUTPUT
        if (flags & GPIO_V2_LINE_FLAG_OUTPUT) {
            int val = gpio_v2_line_config_output_value(&ulr.config, 0);
            // → val = 1 (gpioset 5=1)
            ret = gpiod_direction_output(desc, 1);
        }
    }

    // ⑤ 分配匿名 fd 返回用户态
    fd = get_unused_fd_flags(O_RDONLY | O_CLOEXEC);
    fd_install(fd, file);
    return fd;
}
```

#### gpiod_request_user → gpiod_request → gpiod_request_commit

`gpiod_request_user` 内部调 `gpiod_request`，`gpiod_request` 调 `gpiod_request_commit`：

```c
// gpiolib.c:2181
int gpiod_request(struct gpio_desc *desc, const char *label)
{
    int ret = -EPROBE_DEFER;

    if (try_module_get(desc->gdev->owner)) {
        ret = gpiod_request_commit(desc, "gpioset");
        if (ret)
            module_put(desc->gdev->owner);
        else
            gpio_device_get(desc->gdev);
    }
    return ret;
}

static int gpiod_request_commit(struct gpio_desc *desc, const char *label)
{
    struct gpio_chip *gc = desc->gdev->chip;
    unsigned offset;

    // ① 标记为已请求（防止重复请求）
    if (test_and_set_bit(FLAG_REQUESTED, &desc->flags) == 0) {
        desc_set_label(desc, "gpioset");
    } else {
        return -EBUSY;          // GPIOH5 已被其他程序占用
    }

    // ② 调 chip 驱动的 .request 回调
    if (gc->request) {
        offset = gpio_chip_hwgpio(desc);      // offset = 5
        ret = gc->request(gc, 5);
        // → stm32_gpio_request(GPIOH_chip, 5)
    }

    return 0;
}
```

**GPIOH5 从用户 `gpioset gpiochip7 5=1` 到 `gpiod_request_commit` 的数据流：**

```
用户空间:
  gpioset gpiochip7 5=1
    │  参数解析: chip=gpiochip7(bank_nr=7), offset=5, value=1
    │  libgpiod 构造 gpio_v2_line_request{offsets={5}, consumer="gpioset", OUTPUT}
    ▼
  ioctl(GPIO_V2_GET_LINE_IOCTL, &req)
    │
    ▼ 内核
  linereq_create(gdev, ip)
    ├── copy_from_user → ulr.offsets[0] = 5, ulr.consumer = "gpioset"
    ├── gpiochip_get_desc(gdev->chip, 5) → desc = &gdev->descs[5]
    │   desc->label = NULL (未请求)
    │   desc->flags = 0
    │
    ├── gpiod_request_user(desc, "gpioset")
    │    └── gpiod_request_commit(desc, "gpioset")
    │         ├── test_and_set_bit(FLAG_REQUESTED) → 0→1 (设置已请求标记)
    │         ├── desc_set_label(desc, "gpioset")
    │         └── gc->request(gc, 5)
    │              └── stm32_gpio_request(chip, 5)  ← 进入 STM32 驱动
    │
    └── gpiod_direction_output(desc, 1)  → 写 BSRR + MODER（下节展开）
```

##### stm32_gpio_request — 跨子系统桥梁

**入口文件：** `pinctrl-stm32.c:350`

```c
static int stm32_gpio_request(struct gpio_chip *chip, unsigned offset)
{
    // chip = GPIOH 的 gpio_chip, offset = 5

    struct stm32_gpio_bank *bank = gpiochip_get_data(chip);
    // → bank = GPIOH, bank_nr = 7

    struct stm32_pinctrl *pctl = dev_get_drvdata(bank->gpio_chip.parent);
    struct pinctrl_gpio_range *range;

    // ① GPIO bank 内偏移 → pinctrl pin 号
    //    bank_nr = 7, STM32_GPIO_PINS_PER_BANK = 16
    //    pin = 5 + 7 * 16 = 117
    int pin = offset + (bank->bank_nr * STM32_GPIO_PINS_PER_BANK);

    // ② 确认这个 pin 在 gpio-range 中
    //    gpio-ranges = <&pinctrl 2 114 12>
    //    → bank 内偏移 2~13 对应 pinctrl pin 114~125
    //    → offset 5 在 range 内 → 有效
    range = pinctrl_find_gpio_range_from_pin_nolock(pctl->pctl_dev, pin);
    if (!range) {
        dev_err(pctl->dev, "pin %d not in range.\n", pin);
        return -EINVAL;
    }

    // ③ RIF 安全控制（STM32MP2 特有）
    if (bank->rif_control) {
        if (!stm32_gpio_rif_acquire_semaphore(bank, offset)) {
            dev_err(pctl->dev, "pin %d not available.\n", pin);
            return -EINVAL;
        }
    }

    // ④ 跨子系统：通知 pinctrl 该引脚被 GPIO 使用
    return pinctrl_gpio_request(chip->base + offset);
}
```

**`pinctrl_gpio_request` — 通知 pinctrl 子系统：**

```c
// drivers/pinctrl/core.c:795
int pinctrl_gpio_request(unsigned gpio)  // gpio = 全局 GPIO 号
{
    struct pinctrl_dev *pctldev;
    struct pinctrl_gpio_range *range;
    int pin;

    // ① 遍历 pctldev->gpio_ranges 链表，找 gpio 落在哪个 range 中
    ret = pinctrl_get_device_gpio_range(gpio, &pctldev, &range);

    // ② 全局 GPIO 号 → pinctrl pin 号
    //    gpio_to_pin: offset = gpio - range->base
    //                 pin = range->pin_base + offset
    pin = gpio_to_pin(range, gpio);

    // ③ 调用 pinmux 层申请（冲突检测 + 设置 gpio_owner）
    return pinmux_request_gpio(pctldev, range, pin, gpio);
}
```

**`pinmux_request_gpio`：**

```c
// drivers/pinctrl/pinmux.c:275
int pinmux_request_gpio(struct pinctrl_dev *pctldev,
            struct pinctrl_gpio_range *range,
            unsigned pin, unsigned gpio)
{
    const char *owner;

    // 构造 owner 字符串：如 "GPIOH:117"
    owner = kasprintf(GFP_KERNEL, "%s:%d", range->name, gpio);

    // 调用核心 pin_request（与场景一的 mux 申请相同函数）
    // 注意第四个参数 range != NULL → 表示是 GPIO 请求
    return pin_request(pctldev, pin, owner, range);
}
```

**关键点**：`pin_request` 被**两种场景**复用：
- 场景一：`pinmux_enable_setting` 调它时传递 `gpio_range = NULL` → 设置 `desc->mux_owner`
- 场景二：`pinmux_request_gpio` 调它时传递 `gpio_range != NULL` → 设置 `desc->gpio_owner`

这意味着一个引脚不能同时被 mux 为外设功能和 GPIO 使用——除非 pin controller 的 `strict` 标志为 false。

**GPIOH5 完整申请数据流：**

```
stm32_gpio_request(chip=GPIOH, offset=5)
  ├── bank->bank_nr = 7
  ├── pinctrl pin# = 5 + 7×16 = 117
  │
  ├── stm32_gpio_rif_acquire_semaphore(bank, 5)   // RIF 锁（STM32MP2 特有）
  │   └── 成功 → 该 pin 未被安全域锁定
  │
  └── pinctrl_gpio_request(chip->base + 5)
       └── 全局 GPIO 号 = chip->base + 5
           例: gpioh 的 base = 112 → 112 + 5 = 117
       │
       ├── pinctrl_get_device_gpio_range(117, ...)
       │   → 遍历 pctldev->gpio_ranges
       │   → gpio-ranges 中 base=112, npins=12
       │   → 117 ∈ [112, 123] → 找到 range ✓
       │
       ├── gpio_to_pin(range, 117)
       │   → offset = 117 - 112 = 5
       │   → pin = 114 + 5 = 119 (pinctrl pin#)
       │
       └── pinmux_request_gpio(pctldev, range, pin=119, gpio=117)
            └── owner = "gpioh:117"
            └── pin_request(pctldev, 119, "gpioh:117", range != NULL)
                 → gpio_range != NULL → 走 GPIO 分支
                 → desc->gpio_owner = "gpioh:117"  ✓ 申请成功
```

> **注意**：`chip->base` 在 `gpiochip_add_data` 时由系统分配（或 DTS 中 `gpio-ranges` 的首个编号），GPIOH 的 base = 112。

### 4.3.7 方向配置：从 gpiod_direction_output 到 MODER 寄存器

```c
if (flags & GPIO_V2_LINE_FLAG_OUTPUT) {
    val = gpio_v2_line_config_output_value(lc, i);
    ret = gpiod_direction_output(desc, val);     // ← 设置输出
}
```

调用链：

```
gpiod_direction_output(desc, val)           ← gpiolib.c
  └─ gpiod_direction_output_commit(desc, val)
       └─ gc->direction_output(gc, offset, val)
            └─ stm32_gpio_direction_output(chip, offset, val)
                 ├─ __stm32_gpio_set(bank, offset, value)  ← 先设电平
                 │    └─ writel_relaxed(BIT(offset), BSRR)  ← 写 BSRR
                 └─ pinctrl_gpio_direction_output(chip->base + offset)
                      └─ pinmux_gpio_set_direction()
                           └─ ops->gpio_set_direction()
                                └─ stm32_pmx_gpio_set_direction(bank, pin, input=false)
                                     └─ stm32_pmx_set_mode(bank, pin, mode=1, alt=0)
                                          → MODER bits = 0b01 (output)
```

**`stm32_pmx_gpio_set_direction`：**

```c
// pinctrl-stm32.c:1134
static int stm32_pmx_gpio_set_direction(struct pinctrl_dev *pctldev,
                struct pinctrl_gpio_range *range, unsigned gpio, bool input)
{
    struct stm32_gpio_bank *bank = gpiochip_get_data(range->gc);
    int pin = stm32_gpio_pin(gpio);

    // input=true  → mode=0 (输入模式)
    // input=false → mode=1 (输出模式)
    return stm32_pmx_set_mode(bank, pin, !input, 0);
}
```

### 4.3.8 写 GPIO：从 ioctl 到 BSRR 寄存器

用户在获得 line request 的 fd 后，通过 `GPIO_V2_LINE_SET_VALUES_IOCTL` 写值：

```c
// gpiolib-cdev.c:1621
case GPIO_V2_LINE_SET_VALUES_IOCTL:
    return linereq_set_values(lr, ip);
```

调用链：

```
gpiod_set_array_value_complex(false, true, ...)
  └─ gpiod_set_value_nocheck(desc, value)
       └─ gpiod_set_raw_value_commit(desc, value)
            └─ gc->set(gc, gpio_chip_hwgpio(desc), value)
                 └─ stm32_gpio_set(chip, offset, value)
                      └─ __stm32_gpio_set(bank, offset, value)
```

#### __stm32_gpio_set — BSRR 写操作

**入口文件：** `pinctrl-stm32.c:339`

```c
static inline void __stm32_gpio_set(struct stm32_gpio_bank *bank,
    unsigned offset, int value)
{
    // ① 备份值到内存
    stm32_gpio_backup_value(bank, offset, value);

    // ② BSRR 技巧：低 16 位 = set，高 16 位 = reset
    if (!value)
        offset += STM32_GPIO_PINS_PER_BANK;  // 置 0 → 使用高 16 位

    // ③ 写 BSRR 寄存器
    writel_relaxed(BIT(offset), bank->base + STM32_GPIO_BSRR);
}
```

**BSRR（Bit Set/Reset Register）设计原理：**

```
BSRR 寄存器布局 (@ 0x18):
  位 [15:0]  = BSx (Set):   写 1 使对应引脚输出高电平
  位 [31:16] = BRx (Reset): 写 1 使对应引脚输出低电平

  如果同时写 BSx 和 BRx 对应同一位 → BRx 优先 (Reset)
```

**对于 GPIOH5 置 1（value=1）：**

```
offset = 5
value = 1 → 不移位（使用低 16 位）
writel_relaxed(BIT(5), 0x442b0000 + 0x18)
  → 写 0x00000020 到 BSRR
  → GPIOH 的第 5 位输出高电平
```

**对于 GPIOH5 置 0（value=0）：**

```
offset = 5 + 16 = 21
writel_relaxed(BIT(21), 0x442b0000 + 0x18)
  → 写 0x00200000 到 BSRR
  → 写 BR5 (bit 21)，GPIOH5 输出低电平
```

> **为什么不用写 ODR？** BSRR 是**原子操作**——写 1 的位生效，写 0 的位不影响。而 ODR 需要读-改-写，在多线程/中断上下文中可能存在竞态问题。BSRR 的设计确保了单次写入即可安全地 set 或 reset 一个引脚，无需额外的锁保护。

### 4.3.9 读 GPIO：从 ioctl 到 IDR 寄存器

```c
// gpiolib-cdev.c:1435
static long linereq_get_values(struct linereq *lr, void __user *ip)
{
    // ... 解析 mask 参数
    ret = gpiod_get_array_value_complex(false, true, num_get,
                        descs, NULL, vals);
}

// gpiolib.c:3635
int gpiod_get_value_cansleep(const struct gpio_desc *desc)
{
    int value;
    value = gpiod_get_raw_value_commit(desc);
    if (test_bit(FLAG_ACTIVE_LOW, &desc->flags))
        value = !value;
    return value;
}

// gpiolib.c:2816
static int gpiod_get_raw_value_commit(const struct gpio_desc *desc)
{
    value = gpio_chip_get_value(gc, desc);
    return value < 0 ? value : !!value;
}

// gpiolib.c:2789
static int gpio_chip_get_value(struct gpio_chip *gc, const struct gpio_desc *desc)
{
    return gc->get ? gc->get(gc, gpio_chip_hwgpio(desc)) : -EIO;
    // → stm32_gpio_get(chip, offset)
}
```

#### stm32_gpio_get — 读 IDR

```c
// pinctrl-stm32.c:392
static int stm32_gpio_get(struct gpio_chip *chip, unsigned offset)
{
    struct stm32_gpio_bank *bank = gpiochip_get_data(chip);

    return !!(readl_relaxed(bank->base + STM32_GPIO_IDR) & BIT(offset));
    // → IDR @ 0x10, 读取 offset 位
}
```

**IDR（Input Data Register）特性：**

```
IDR 寄存器 (@ 0x10):
  位 [15:0] = 对应引脚 15~0 的当前输入电平
  无论引脚配置为什么模式（输入/输出/AF），IDR 都反映引脚的实际物理电平
```

**对于 GPIOH5：**

```c
val = readl_relaxed(0x442b0000 + 0x10)    // 读取 IDR 所有 16 位
val & BIT(5)                               // 如果高电平→非0→return 1
                                           // 如果低电平→0→return 0
```

### 4.3.10 中断：GPIO → EXTI → GIC（gpiomon 路径）

当用户执行 `gpiomon gpiochip6 5` 时，libgpiod 调用 ioctl 请求 line 并配置边沿检测：

```c
// linereq_create() 中:
if (flags & GPIO_V2_LINE_FLAG_INPUT) {
    ret = gpiod_direction_input(desc);
    if (!ret)
        ret = edge_detector_setup(&lr->lines[i], lc, i, edflags);
}
```

#### edge_detector_setup — 注册中断 handler

```c
// gpiolib-cdev.c:1185
static int edge_detector_setup(struct line *line,
               struct gpio_v2_line_config *lc,
               unsigned int line_idx, u64 edflags)
{
    unsigned long irqflags = 0;
    int irq, ret;

    // ① 分配事件 FIFO
    if (eflags && !kfifo_initialized(&line->req->events)) {
        ret = kfifo_alloc(&line->req->events, ...);
    }

    // ② 配置软件去抖（可选）

    // ③ 获取 IRQ 号
    irq = gpiod_to_irq(line->desc);     // ← 关键：GPIO → IRQ

    // ④ 设置触发方式
    if (eflags & GPIO_V2_LINE_FLAG_EDGE_RISING)
        irqflags |= IRQF_TRIGGER_RISING;
    if (eflags & GPIO_V2_LINE_FLAG_EDGE_FALLING)
        irqflags |= IRQF_TRIGGER_FALLING;

    // ⑤ 注册线程化中断
    ret = request_threaded_irq(irq, edge_irq_handler, edge_irq_thread,
                   irqflags, label, line);
}
```

#### gpiod_to_irq → stm32_gpio_to_irq

```c
// gpiolib.c (简化)
int gpiod_to_irq(const struct gpio_desc *desc)
{
    struct gpio_chip *gc = desc->gdev->chip;
    return gc->to_irq ? gc->to_irq(gc, gpio_chip_hwgpio(desc)) : -ENXIO;
    // → stm32_gpio_to_irq(chip, offset)
}
```

```c
// pinctrl-stm32.c:423
static int stm32_gpio_to_irq(struct gpio_chip *chip, unsigned int offset)
{
    struct stm32_gpio_bank *bank = gpiochip_get_data(chip);
    struct irq_fwspec fwspec;

    // 如果已缓存，直接返回
    if (bank->virq[offset])
        return bank->virq[offset];

    // 构造 IRQ fwspec
    fwspec.fwnode = bank->gpio_chip.fwnode;
    fwspec.param_count = 2;
    fwspec.param[0] = offset;
    fwspec.param[1] = IRQ_TYPE_NONE;

    // 通过 irq domain 映射
    return irq_create_fwspec_mapping(&fwspec);
    // → bank domain → EXTI domain → GIC domain
}
```

**中断路径总结：**

```
GPIO pin 物理电平变化
  → GPIO 模块检测到边沿
    → EXTI 多路选择器（SYSCFG 配置）路由到 EXTI 控制器
      → EXTI line 检测到中断
        → GIC（Generic Interrupt Controller）接收 CPU 中断
          → edge_irq_handler() → 将事件写入 kfifo
            → 用户态通过 poll() / read() 读取事件
```

**gpiomon gpiochip6 5（监听 GPIOG5 边沿事件）的实际跟踪：**

```
用户执行 gpiomon gpiochip6 5 -r -f    ← 监听 GPIOG5 上升沿+下降沿
  │ gpiomon 构造 GPIO_V2_GET_LINE_IOCTL {offsets={5}, INPUT, EDGE_BOTH}
  │
  ├── linereq_create() → gpiod_direction_input(desc)
  │   → stm32_pmx_gpio_set_direction(bank=GPIOG, pin=5, input=true)
  │     → stm32_pmx_set_mode(bank, 5, mode=0, alt=0)
  │     → GPIOG MODER bits[11:10] = 0b00 (input mode)
  │
  └── edge_detector_setup()
        ├── eflags = RISING | FALLING
        ├── gpiod_to_irq(desc)
        │   └── stm32_gpio_to_irq(chip=GPIOG, offset=5)
        │        ├── bank->virq[5] 是否缓存？→ 首次调用为 0
        │        ├── 构造 fwspec: {fwnode, param[0]=5, param[1]=IRQ_TYPE_NONE}
        │        ├── irq_create_fwspec_mapping(&fwspec)
        │        │   → bank irq_domain → EXTI irq_domain → GIC
        │        │   → 分配 Linux IRQ 号（如 64）
        │        └── bank->virq[5] = 64 (缓存)
        │
        ├── irqflags = IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING
        └── request_threaded_irq(irq=64,
               edge_irq_handler, edge_irq_thread,
               IRQF_TRIGGER_RISING | IRQF_TRIGGER_FALLING,
               "gpiomon", line)
```

此后 GPIOG5 每产生一次电平跳变：
```
物理跳变 → EXTI → GIC → edge_irq_handler()
  → 将 {timestamp, rising/falling} 写入 kfifo
    → 用户态 gpiomon: poll() 检测到事件 → read() 读取
      → 打印 "     5     1    rising" 之类的事件记录
```

> GPIO 中断在 03 的 probe 流程中已经配置好 irq_domain 层次结构：`stm32_pctrl_get_irq_domain()` 在 probe 早期创建每个 bank 的 irq_domain，`stm32_gpio_to_irq()` 就是在这个层次化的 domain 中查找映射关系。

### 4.3.11 场景二完全验证

**设备：** ATK 板用户 LED 连接 GPIOH5。

```dts
gpio-leds {
    led {
        gpios = <&gpioh 5 GPIO_ACTIVE_LOW>;
        linux,default-trigger = "heartbeat";
    };
};
```

**GPIOH5（LED）的操作路径：**

```
用户执行: gpioset gpiochip7 5=1
  │
  ├── open("/dev/gpiochip7")  → gpio_chrdev_open(), chrdev 基于 gpioh 实例
  │
  ├── ioctl(GPIO_V2_GET_LINE_IOCTL) → linereq_create()
  │    ├── gpiod_request_user() → stm32_gpio_request()
  │    │    → pinctrl_gpio_request() → pinmux_request_gpio()
  │    │    → pin_request() — 检查冲突，设 gpio_owner
  │    │
  │    └── gpiod_direction_output(desc, 1)
  │         ├── __stm32_gpio_set(bank, 5, 1)
  │         │    → writel_relaxed(BIT(5), 0x442b0000 + 0x18)  // BSRR set
  │         └── stm32_pmx_gpio_set_direction(bank, 5, input=false)
  │              → stm32_pmx_set_mode(bank, 5, mode=1, alt=0)
  │              → MODER bits [11:10] = 0b01 (output)
  │
  └── GPIO_V2_LINE_SET_VALUES_IOCTL {mask=1<<0, bits=1<<0}
       └── linereq_set_values()
            → stm32_gpio_set(chip, 5, 1)
                 → __stm32_gpio_set(bank, 5, 1)
                      → writel_relaxed(BIT(5), 0x442b0000 + 0x18)
                          // BSRR 写入 0x00000020
                          // → GPIOH5 输出高电平
```

**GPIOH 寄存器在操作前后的状态：**

```
操作前 (复位状态):
  MODER @ 0x442b0000:  0x00000000  (复位：输入模式)
  BSRR  @ 0x442b0000:  无影响 (写1生效)
  IDR   @ 0x442b0000:  取决于外部电平

gpiod_direction_output 后:
  MODER @ 0x442b0000:  0x00000400  (bits [11:10] = 0b01)

gpiod_set_value(desc, 1) 后:
  BSRR @ 0x442b0000:  0x00000020  → 引脚 5 输出高电平

gpiod_set_value(desc, 0) 后:
  BSRR @ 0x442b0000:  0x00200000  (0x20 << 16) → 引脚 5 输出低电平

读取电平 (gpiod_get_value):
  val = readl(0x442b0000 + 0x10)  // IDR
  val & BIT(5) → 0x20 (高) 或 0x00 (低)
```

### 4.3.12 两种 GPIO 使用路径的互通性

场景一是"外设功能 mux"（consumer probe），场景二是"GPIO 作为 GPIO 使用"。它们操作的是**同一套寄存器**，但视角不同：

| 操作 | 场景一（USART2） | 场景二（LED） |
|------|-----------------|--------------|
| **MODER** | 写入 `0b10`（AF 模式） | 写入 `0b01`（输出模式） |
| **AFRx** | 写入 AF 编号（6/8） | 写入 0（不重要） |
| **BSRR** | 由 UART 外设硬件控制 | CPU 通过 gpiolib 写入 |
| **IDR** | 由 UART 外设硬件驱动 | CPU 通过 gpiolib 读取 |
| **冲突检测** | `pin_desc.mux_owner` 检查 | `pin_desc.gpio_owner` 检查 |

两者**不能同时**使用同一个引脚——`pin_request` 中的冲突检测确保了一个引脚要么被 mux 为外设功能，要么被用作 GPIO，不能共存（strict 模式下）。

---

## 总结

### 场景一（Consumer probe pin muxing）

从 DTS 中的 `pinctrl-0` 到硬件寄存器的完整路径：

```
DTS: pinctrl-0 = <&usart2_pins_a>          // phandle 引用
  → pinctrl_dt_to_map()                    // 解析 DTS 属性
    → dt_to_map_one_config()               // 找 pin controller
      → stm32_pctrl_dt_node_to_map()       // 驱动回调，生成 map
        → dt_remember_or_free_map()        // 挂入全局链表
  → for_each_pin_map() → add_setting()    // map → setting
    → pinctrl_select_state()               // 执行
      → pinmux_enable_setting()            // pin 申请 + set_mux
        → stm32_pmx_set_mux()             // 找 bank，算 mode/alt
          → stm32_pmx_set_mode()           // 写 AFRx + MODER
      → pinconf_apply_setting()            // 写 PUPDR/OTYPER/OSPEEDR
```

### 场景二（用户态 GPIO）

从 `gpioset` 命令行到硬件寄存器的完整路径：

```
用户态: gpioset gpiochip7 5=1
  → ioctl(GPIO_V2_GET_LINE_IOCTL)          // 请求 line
    → linereq_create()
      → gpiod_request_user()               // 申请 GPIO
        → stm32_gpio_request()             // 通知 pinctrl
      → gpiod_direction_output()           // 设方向
        → stm32_pmx_gpio_set_direction()   // 写 MODER
  → ioctl(GPIO_V2_LINE_SET_VALUES_IOCTL)   // 设电平
    → gpiod_set_raw_value_commit()
      → stm32_gpio_set()                   // 写 BSRR
```

### 两个场景的共性与差异

| 维度 | 场景一 | 场景二 |
|------|--------|--------|
| **触发方式** | 设备 probe 时自动触发 | 用户态程序显式调用 |
| **核心框架** | pinctrl core | gpiolib core + gpiolib cdev |
| **最终函数** | `stm32_pmx_set_mode` | `stm32_gpio_set`/`get` |
| **写入的寄存器** | MODER + AFRx + PUPDR/OTYPER/OSPEEDR | BSRR（写）IDR（读） |
| **关键结构体** | `pinctrl_map` → `pinctrl_setting` → `pin_desc` | `gpio_desc` → `gpio_chip` |
| **跨子系统协作** | pinctrl 驱动内部完成 | gpio_chip 回调 → pinctrl_gpio_request |
| **冲突检测** | `pin_desc.mux_usecount` + `mux_owner` | `pin_desc.gpio_owner` |

两个场景最终都在 STM32 GPIO 寄存器层面交汇——无论你是通过 pinctrl 子系统将引脚 mux 为 UART 功能，还是通过 gpiolib 将引脚当作 GPIO 输出，最终操作的都是同一组物理寄存器（MODER、AFRx、BSRR、IDR 等）。pinctrl 和 GPIO 子系统在逻辑上分离，但在硬件层面共享操作目标——这正是 02 中所说的"合体驱动"在运行时的体现。

---

**参考文件索引**

| 路径 | 内容 |
|------|------|
| `drivers/pinctrl/core.c` | pinctrl core：`create_pinctrl`, `pinctrl_select_state`, `pinctrl_commit_state` |
| `drivers/pinctrl/devicetree.c` | DTS 解析：`pinctrl_dt_to_map`, `dt_to_map_one_config`, `dt_remember_or_free_map` |
| `drivers/base/pinctrl.c` | `pinctrl_bind_pins`——consumer 设备 pinctrl 绑定入口 |
| `drivers/base/dd.c` | `really_probe`——设备驱动 probe 流程 |
| `drivers/pinctrl/pinmux.c` | `pinmux_enable_setting`, `pin_request`, `pinmux_request_gpio` |
| `drivers/pinctrl/stm32/pinctrl-stm32.c` | STM32 驱动：`stm32_pmx_set_mux`, `stm32_pmx_set_mode`, `stm32_gpio_set/get/request/to_irq` |
| `drivers/gpio/gpiolib.c` | gpiolib core：`gpiod_request`, `gpiod_get_value`, `gpiod_set_value` |
| `drivers/gpio/gpiolib-cdev.c` | gpiolib 字符设备：`gpio_chrdev_open`, `gpio_ioctl`, `linereq_create` |
| `include/dt-bindings/pinctrl/stm32-pinfunc.h` | `STM32_PINMUX` 宏定义 |
| `include/linux/pinctrl/devinfo.h` | `struct dev_pin_info` 定义 |
| `arch/arm64/boot/dts/st/stm32mp25-pinctrl-atk-ddr-2GB.dtsi` | ATK 板 pinctrl 配置 |
| `arch/arm64/boot/dts/st/stm32mp257d-atk-bsp.dts` | ATK BSP DTS |
