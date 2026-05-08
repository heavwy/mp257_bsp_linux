# Linux 驱动子系统深度学习框架

> **目标**: 定义可复用的驱动子系统学习方法论和文档产出规范  
> **适用平台**: STM32MP257 (Cortex-A35, Linux v6.6.78)  
> **前置知识**: [Linux-Driver-Base-Knowledge.md](Linux-Driver-Base-Knowledge.md) — 每次学习新子系统前先回顾  
> **核心理念**: 不逐行读代码，只做关键路径分析；不只看代码，必须结合硬件手册

---

## 一、框架总览

```
┌─────────────────────────────────────────────────────────────────┐
│                   驱动子系统深度学习                               │
├─────────────────────────────────────────────────────────────────┤
│  前置依赖                                                      │
│    ┌─ Linux 设备模型         ── bus/driver/device 三元组        │
│    ├─ 并发模型               ── 四种上下文 + 锁选择              │
│    └─ 内存管理与 DMA         ── cache coherency + API 选择      │
│    (以上详见 Linux-Driver-Base-Knowledge.md)                     │
│                                                                │
│  对一个子系统的完整分析 — 分 11 步                              │
│    ├─ ⓪ 发展历史             ── 30 年演进，有趣有料             │
│    ├─ ① 使用方法             ── 用户态/内核态/DTS              │
│    ├─ ② 架构与数据结构        ── 分层图 + 结构体关联 + 状态机    │
│    ├─ ③ Probe 源码分析       ── probe 逐行分析                  │
│    ├─ ④ 数据/中断路径分析     ── 传输/中断/电源管理              │
│    ├─ ⑤ 硬件手册关联         ── 寄存器 ↔ 代码对应              │
│    ├─ ⑥ 错误处理             ── EPROBE_DEFER + 回滚            │
│    ├─ ⑦ 情景分析             ── 完整链路走通                   │
│    ├─ ⑧ 面试现场             ── 核心问题深挖                   │
│    ├─ ⑨ 实验                ── 上手操作才能真掌握             │
│    └─ ⑩ 实践总结            ── 难点 + 避坑 + 技术债务          │
│                                                                │
│  输出 — 一个系列文档集合                                        │
│    └─ 每步产出一篇 Markdown，12 篇文档 + README 构成知识库      │
└─────────────────────────────────────────────────────────────────┘
```

**学习原则:**
- **先基础再实战**: 回顾 Base-Knowledge 中的设备模型和并发基础
- **关键路径驱动**: 不逐行读代码，只追踪 probe → open/read/write → 中断 三条主线
- **动手验证**: 每个知识点必须在开发板上验证
- **系列产出**: 每步产出一篇独立文档，组合成知识库

---

## 二、对每个子系统的分析步骤

每步完成后产出一篇 Markdown 文档，存放到 `note/<SubsystemName>/`。

---

### 步骤 ⓪ 发展历史

产出文档: `00-History.md`

> 一个子系统今天的模样，是过去 30 年打补丁打出来的。  
> 读懂历史 = 读懂"为什么代码长得这么丑"、"为什么有这么多层"、"为什么这个 API 有两个版本"。

**为什么需要历史？**

驱动子系统不是一天设计出来的。一个 1996 年为了 2.4 寸单色屏写的框架，到了 2024 年要支持 4K HDMI + 触摸 + 多图层合成，代码里全是历史的痕迹。不了解历史，你会觉得"_这个设计好蠢_"——其实它是为了兼容 20 年前的一个需求。

**输出格式:**

```
# 00. <子系统> 30 年演进史

> 本文是系列的第 0 篇，为后续的所有分析提供历史背景。
> 建议在阅读其他文档前先读这篇。

---

## 年代线

[年份] [内核版本] ── 发生了什么
  ┌─ 关键变化:
  └─ 为什么:

[年份] [内核版本] ── 发生了什么
  ...

---

## 关键转折点

转折 1: [事件]
  ┌─ 之前: 怎么做
  └─ 之后: 怎么做

转折 2: ...

---

## 遗留的技术债务

到今天仍然存在的"历史遗留问题":

| 问题 | 来源 | 为什么还没改 |
|------|------|------------|

---

## 有趣的故事

- [一个著名的 bug / Linus 的评论 / 维护者的争议]
```

**各子系统的历史关键节点参考：**

| 子系统 | 诞生 | 重大重构 | 关键人物/事件 |
|--------|------|---------|-------------|
| **中断** | v2.0 (1996) genirq | v2.6.24 (2007) 通用层 + irq_domain v3.4 (2012) | Thomas Gleixner 重写, INTC 类型泛滥 |
| **GPIO** | v2.4 (2001) 整数 API | v3.10 (2013) 描述符 API, v4.5 (2016) 字符设备 | Linus Walleij 推动描述符化 |
| **I2C** | v2.4 (2001) | v2.6.34 (2010) 原子操作修复 | 从 bit-bang 到多主控 |
| **SPI** | v2.4 (2001) | v2.6.25 (2008) 队列化 | David Brownell 初始实现 |
| **UART** | v1.0 (1994) | v2.4.10 (2001) tty 层, v3.7 (2012) 多核安全 | Alan Cox 长期维护 |
| **DMA** | v2.3 (1999) | v2.6.17 (2006) dmaengine, v4.2 通用 API | 从 arch 专属到通用 |
| **ETH** | v1.0 (1994) | v2.5 (2002) NAPI, v4.10 (2017) XDP | NAPI 解决活锁, eBPF |
| **AUDIO** | v2.4 (2001) OSS | v2.6 (2003) ALSA, v2.6.18 (2006) ASoC | OSS→ALSA 是大撕裂 |
| **DISPLAY** | v2.1 (1998) fbdev | v3.3 (2012) DRM 成为主流, v4.12 (2017) atomic API | fbdev 到 DRM/KMS 转型 |
| **USB** | v2.3 (1999) | v2.6 (2003) USB Gadget, v3.2 (2011) xHCI | OHCI/UHCI/EHCI/xHCI 四代 |
| **MMC** | v2.4 (2000) | v2.6.25 (2008) 分层, v4.14 (2017) CQHCI | 从 SD 到 eMMC 到 UFS |
| **PCIe** | v2.0 (1996) PCI | v2.6 (2003) MSI, v3.23 (2012) PCIe 热插拔 | BIOS→ACPI→DT 枚举 |
| **PINCTRL** | v2.6.38 (2011) | — | GPIO 和 pinmux 从纠缠到分离 |
| **CLK** | v2.6.39 (2011) | v3.0 (2011) clk framework | 之前每个 arch 自己管 |

**参考来源：** Linux 内核邮件列表 (LKML)、内核 git log、kernelnewbies.org、LWN.net

---

### 步骤 ① 使用方法 (Usage)

产出文档: `01-Usage.md`

列出该子系统的三个维度:

**1. 用户态接口**
- sysfs 路径，实际可操作的命令
- 专用工具（如 i2c-tools、evtest、gpiod、ethtool 等）
- 验证方法——在开发板上能跑什么命令确认功能正常

**2. 内核态核心 API**
- 分配/注册/注销函数
- 数据收发函数（同步/异步）
- 中断/DMA API（如果涉及）
- 优先展示 `devm_` 系列版本

**3. DTS 配置**
- SoC 级 dtsi 中的控制器节点
- 板级 dts 中的外设节点
- pinctrl/pinmux 配置
- 时钟、中断、DMA 的引用方式
- status 属性

---

### 步骤 ② 架构与数据结构分析

产出文档: `02-Architecture.md`

**2.1 分层架构图**

按六层模型画出该子系统：

```
Layer 1: 用户空间     — ——工具名称, 设备节点路径
Layer 2: VFS/系统调用 — ——file_operations 接口
Layer 3: 核心框架     — ——框架名，核心结构体
Layer 4: 具体驱动     — ——驱动名，probe 函数位置
Layer 5: 寄存器操作   — ——readl/writel 访问
Layer 6: 物理硬件     — ——外设基址，中断号
```

**2.2 核心数据结构关联图**

列出 3-5 个核心结构体，画清楚:

```
struct core_frame {
    ── 职责: 核心框架数据，管理所有实例
    ── 关键字段: dev, ops, id, list_head
    ── 关联: container_of 关系, 指针指向, 链表连接
}

struct drv_instance {
    ── 职责: 一个硬件实例的数据
    ── 关键字段: base(MMIO), irq, clk, lock, state
    ── 关联: 通过 ops 被 core 调用，通过 container_of 自反
}

struct hw_ops {
    ── 职责: 驱动的回调函数表
    ── 回调: .init, .transfer, .cleanup
}
```

对每个结构体解释**为什么需要它**（它解决了什么问题）。

**2.3 状态机**

画出关键的状态迁移：

```
PROBE → READY → SUSPEND → RESUME → REMOVE
                → TRANSFER (数据传输中状态)
```

**2.4 设计模式**

识别并说明该子系统的设计模式（分层分离、注册回调、container_of、生产者-消费者、缓存优化等）。

---

### 步骤 ③ Probe 流程源码分析

产出文档: `03-Probe-Analysis.md`

**3.1 入口定位**

```
函数: xxx_probe(struct platform_device *pdev)
文件: drivers/xxx/xxx-core.c:1234
```

**3.2 逐行分析**

每步说明 "为什么"：

```
Step 1: 获取 SoC 差异数据
  ┌─ of_device_get_match_data(dev)
  └─ 为什么？同一个驱动可能支持多个 SoC 版本，寄存器偏移不同

Step 2: ioremap
  ┌─ devm_platform_ioremap_resource(pdev, 0)
  └─ 物理地址 → 虚拟地址，为什么用 devm_？自动释放

Step 3: 时钟使能
  ┌─ devm_clk_get() + clk_prepare_enable()
  └─ 为什么 prepare/enable 分开？prepare 可睡眠配 PLL，enable 原子操作开 gate

Step 4: 复位
  ┌─ reset_control_assert/deassert
  └─ 对应手册中 RCC 复位操作

Step 5: 中断
  ┌─ platform_get_irq() + devm_request_irq()
  └─ irq 号从 DTS 解析，handler 在硬中断上下文执行

Step 6: DMA（可选）
  ┌─ dma_request_chan() + dmaengine_slave_config()

Step 7: 注册核心框架
  ┌─ xxx_register(dev, &inst, &xxx_ops)
  └─ → core 将实例加入链表，创建 sysfs，发送 uevent
```

**3.3 数据结构状态变化**

```
probe 前: 全 0
probe 中: base → ioremap, clk → 已使能, irq → 已注册, state → PROBING
probe 后: state → READY, 已加入 core 链表, 设备节点已创建
```

---

### 步骤 ④ 数据/中断路径分析

产出文档: `04-DataPath-Analysis.md`

如果子系统涉及数据收发或中断处理，追踪以下路径：

**路径 A: 数据传输**

```
硬件 FIFO 收到数据
  → ① 中断 handler 读状态寄存器确认源
  → ② 从 FIFO 读数据到内存 buffer
  → ③ 底半部/线程化处理
  → ④ 唤醒等待队列
  → ⑤ read() → copy_to_user()
```

**路径 B: 中断处理**

```
硬件触发 (EXTI/GIC 寄存器变化)
  → 异常向量表 → gic_handle_irq()
  → irq_domain 映射 → irq_desc
  → 调用 chip->irq_mask() → 屏蔽
  → handle_level_irq() → generic_handle_irq()
  → 驱动注册的 handler()
  → chip->irq_unmask() → chip->irq_eoi()
```

**路径 C: 电源管理**

```
xxx_suspend(): 保存寄存器上下文，关时钟，选 sleep pinctrl 状态
xxx_resume(): 恢复寄存器，开时钟，恢复 default pinctrl 状态
```

---

### 步骤 ⑤ 硬件手册关联

产出文档: `05-Hardware-Registers.md`

**寄存器速查表**

| 偏移 | 寄存器名 | 功能 | 位域说明 | 代码引用 |
|------|---------|------|---------|---------|
| 0x00 | XXX_CR1 | 控制寄存器1 | [0]EN, [3:1]MODE, [4]IE | `writel(val, base + CR1)` |
| 0x04 | XXX_SR | 状态寄存器 | [0]RXNE, [1]TXE | `readl(base + SR)` |
| 0x08 | XXX_DR | 数据寄存器 | [7:0]数据 | `readl(base + DR)` |

每行对应代码中具体的 `readl`/`writel` 调用和手册中的章节号。

---

### 步骤 ⑥ 错误处理与健壮性

产出文档: `06-Error-Handling.md`

- EPROBE_DEFER 依赖分析：这个驱动依赖哪些资源？哪些可能 defer？
- devm_ 释放顺序：probe 中的申请顺序确认 remove 时不会乱序
- 部分初始化回滚：probe 每步失败时要能 clean up
- ioctl/read/write：参数校验、设备状态检查、copy_to/from_user 错误处理

---

### 步骤 ⑦ 情景分析

产出文档: `07-Scenario-Analysis.md`

构造 2-3 个代表情景，完整走通从硬件到用户空间的链路。格式如下：

```
=== 情景: [名称] ===
一句话描述

触发者: [用户操作 / 硬件事件]
硬件入口: [哪个外设/哪个中断/哪个寄存器]
软件入口: [第一个被调用的函数]

━━━━ 完整链路 ━━━━

第①站: 硬件层
  发生了什么:
  寄存器变化:
  手册验证: RM0456 §X.X

第②站: CPU 异常 / 中断入口
  → [函数名]
  关键代码:
    代码片段

第③站: 驱动处理
  → [函数名]

第④站: 数据传递到用户空间
  → [函数名]

第⑤站: 用户空间
  应用程序看到什么

━━━━ 全程耗时分析 ━━━━

各阶段延迟估计   瓶颈在哪里
```

---

### 步骤 ⑧ 面试现场

产出文档: `08-Interview.md`

提炼该子系统的核心面试问题。格式如下：

```
### Q: [问题]
级别: 初级/中级/高级
考察点: 面试官想考察什么

答案: [2-3 句话直击核心]

深入追问:
追问 1: → 回答
追问 2: → 回答

避坑提示: ⚠️ 常见错误
```

每个子系统至少提炼 3-5 个问题，覆盖:
- 基本概念（初级）
- 数据结构与 API 选择（中级）
- 并发处理 / 异常路径（中级 → 高级）
- 设计决策（高级）

---

### 步骤 ⑨ 实验

产出文档: `09-Lab.md`

> 前面 8 步都是"看"和"分析"，这一步是**动手操作**。  
> 实验是对前面所有理解的验证，每个子系统必须提供完整的实验指导。

**实验设计要求：**
- 能在 STM32MP257 ATK 开发板上独立完成
- 不需要额外的硬件（除非该子系统必须——如 I2C 外设）
- 步骤清晰，每一步都有**期望结果**
- 提供验证命令和预期输出
- 对常见失败给出排查指引

实验按以下模板组织：

```
# 09. 实验 — STM32MP257 <子系统>

> 前置: [01-Usage.md](01-Usage.md), [07-Scenario-Analysis.md](07-Scenario-Analysis.md)

---

## 实验 1：[名称]

### 目标
用一句话说明这个实验验证什么。

### 硬件要求
需要哪些引脚、跳线、外设。

### 操作步骤

Step 1: [操作说明]
  $ [命令]
  → 期望输出:
    [预期结果]

Step 2: [操作说明]
  $ [命令]
  → 期望输出:
    [预期结果]

### 验证方法
$ [验证命令]
→ 期望看到:
  [预期输出]

### 排查指引
| 现象 | 可能原因 | 检查点 |
|------|---------|--------|
| 错误 A | 原因1 | 怎么检查 |
| 错误 B | 原因2 | 怎么检查 |

---

## 实验 2：[名称]

...
```

**每个子系统至少 2 个实验：**
- **实验 1：基础验证** — 确认设备已注册、基本读写正常（"它能工作"）
- **实验 2：深入验证** — 压测、边界条件、错误注入（"它真的能工作"）

**实验类型参考（按子系统）：**

| 子系统 | 实验1（基础） | 实验2（深入） |
|--------|-------------|-------------|
| **GPIO** | libgpiod 读/写电平，验证电压 | 中断触发 + IRQ 计数验证 |
| **I2C** | i2c-tools 探测、读寄存器 | 大量读写验证 + 时钟拉伸测试 |
| **SPI** | spidev 收发测试 | DMA 传输 + 不同速率验证 |
| **UART** | echo/cat 环回测试 | 高波特率 + 流控验证 |
| **DMA** | dmatest 模块测试 | 内存到内存传输 + 性能测量 |
| **ETH** | ping + iperf 测吞吐 | 中断合并 + 不同帧长 |
| **AUDIO** | aplay 播放测试音频 | 多通道 + 采样率切换 |
| **DISPLAY** | modetest 显示颜色条 | 多图层叠加 + vblank 测试 |
| **MMC** | dd 读写 + hdparm 测速 | 热插拔 + 异常掉电 |
| **USB** | lsusb 识别 + 简单传输 | 批量传输 + 速率切换 |

---

### 步骤 ⑩ 实践总结

产出文档: `10-Practice-Tips.md`

> 这一步不讲源码了。讲的是"真干活的时候会遇到什么"。  
> 每个模块学完，把学习难点、实践避坑、技术债务整理成文。

**三部分内容：**

#### 10.1 学习难点与克服方法

列出这个子系统最具挑战性的 3-5 个概念：

| # | 难点 | 为什么难 | 怎么克服 |
|---|------|---------|---------|
| 1 | [概念] | MCU背景的人没有这个认知，或者这个抽象层次很绕 | 用什么方法理解它（类比、画图、拆解、动手） |
| 2 | [概念] | ... | ... |

**例子（其实这个框架本身就可以这样用）：**

| 难点 | 为什么难 | 怎么克服 |
|------|---------|---------|
| 设备模型 | MCU 是"直接写地址控制"，Linux 是"注册回调等内核调用你" | 先读 Base-Knowledge 的 bus/driver/device 图，再用 /sys 验证 |
| 并发 | MCU 单核单线程，Linux 4个CPU同时跑你的代码 | 先用锁选择矩阵，再在实际驱动中标注每段代码的上下文类型 |
| DMA cache coherency | MCU 没有 cache 问题，不理解"CPU 看见的和硬件看见的不一样" | 在开发板上关 cache 跑一次 DMA，看数据读错的现象，再开 cache |
| irq_domain 层级映射 | MCU 中断是固定向量表，Linux 有三层映射和查找 | 用 /sys/kernel/debug/irq/domains/ 查看实际映射树 |
| DTS ≠ 运行时 DTS | 以为改了 .dts 就能生效，但 OP-TEE/U-Boot 会在启动时改 DTS | 运行时反编译 /sys/firmware/devicetree/base/ 确认真实情况 |

#### 10.2 实践中要注意的点 (避坑指南)

开发这个子系统的驱动时最容易出问题的地方：

```
=== 实践注意点 ===

🔴 坑 1: [具体问题]
  现象:
  原因:
  怎么避免:

🔴 坑 2: [具体问题]
  ...
```

每个子系统提炼 3-5 个"老司机才知道"的坑。这些坑来自：
- LKML 的讨论
- 内核文档中的 TODO/FIXME 注释
- 长期维护者的 commit message
- 实际开发中踩过

#### 10.3 技术债务 (Technical Debt)

这个子系统里哪些部分已经过时但还在用：

| 遗留物 | 来源 | 还在用的原因 | 新替代方案 |
|--------|------|------------|-----------|
| 旧 API / 旧框架 | 历史版本遗留 | 向后兼容 | 当前推荐的新 API |

---

## 三、系列文档结构

每个子系统的产出是以下文件集合，存于 `note/<SubsystemName>/`：

```
note/<SubsystemName>/
├── README.md                     ← 系列索引
├── 00-History.md                 ← 30 年演进史
├── 01-Usage.md                   ← 使用方法 + DTS
├── 02-Architecture.md            ← 分层架构 + 数据结构 + 状态机
├── 03-Probe-Analysis.md          ← Probe 流程逐行分析
├── 04-DataPath-Analysis.md       ← 数据/中断路径 (如适用)
├── 05-Hardware-Registers.md      ← 硬件寄存器速查
├── 06-Error-Handling.md          ← 错误处理 + 健壮性
├── 07-Scenario-Analysis.md       ← 情景分析
├── 08-Interview.md               ← 面试现场
├── 09-Lab.md                     ← 动手实验
└── 10-Practice-Tips.md           ← 难点 + 避坑 + 技术债务
```

### README.md 索引模板

```markdown
# STM32MP257 <子系统> 驱动深度分析

> SoC: STM32MP257D | 内核: v6.6.78 | RM0456 Chapter X

## 文档列表

| # | 文档 | 摘要 | 状态 |
|---|------|------|------|
| 1 | [01-Usage.md](01-Usage.md) | 用户态接口、内核API、DTS配置 | ✅ |
| 2 | [02-Architecture.md](02-Architecture.md) | 六层架构、核心结构、状态机 | ✅ |
| 3 | [03-Probe-Analysis.md](03-Probe-Analysis.md) | Probe 逐行分析 | ⏳ |
| ... | ... | ... | ... |

## 学习路线

先用 → 理解架构 → 追源码 → 对手册 → 情景走通 → 面试检验
```

### 单篇模板

```markdown
# 0X. 标题 — STM32MP257 <子系统>

> 本文是 [系列](README.md) 的第 X 篇。  
> 前置: [01-Usage.md](01-Usage.md)  
> 下一篇: [03-xxx.md](03-xxx.md)

---

## 适用版本

内核 v6.6.78，文件 `drivers/xxx/xxx.c` @ commit <hash>

## 正文

...
```

---

## 四、子系统适用性矩阵

各子系统在学习时可以调整上述 8 步的侧重：

| 子系统 | Usage | 架构/数据结构 | Probe | 数据路径 | 中断 | 寄存器 | 错误处理 | 情景 | 面试 |
|--------|-------|-------------|-------|---------|------|-------|---------|------|------|
| **GPIO** | libgpiod | gpio_chip/desc | 简单 | 无 | 可选 | 简单 | 简单 | 按键→输入 | 3题 |
| **PINCTRL** | debugfs | pinctrl_dev | 简单 | 配置 | 无 | 中 | 简单 | 引脚复用 | 3题 |
| **CLK** | debugfs | clk_hw/core | 中 | 无 | 无 | 中 | 中 | — | 2题 |
| **I2C** | i2c-tools | adapter/client | 简单 | ✦✦✦✦ | 可选 | 简单 | 简单 | 读写 EEPROM | 4题 |
| **SPI** | spidev | controller/device | 简单 | ✦✦✦✦ | 可选 | 简单 | 简单 | DMA 传输 | 4题 |
| **UART** | stty/minicom | uart_port/kfifo | 简单 | ✦✦✦✦ | ✦✦✦✦ | 简单 | 中 | 串口收发 | 4题 |
| **DMA** | — | dma_chan/sg | 中 | ✦✦✦✦ | ✦✦✦✦ | 复杂 | 中 | 内存拷贝 | 5题 |
| **INPUT** | evtest | input_dev/handler | 低 | ✦✦✦ | ✦✦✦✦ | 无 | 中 | 按键上报 | 4题 |
| **ETH** | ifconfig | net_device/skb | 复杂 | ✦✦✦✦✦ | ✦✦✦✦✦ | 复杂 | 复杂 | 收发包 | 6题 |
| **AUDIO** | aplay/arecord | snd_soc_card | 复杂 | DMA循环 | 周期性 | 复杂 | 复杂 | 播放/录音 | 6题 |
| **DISPLAY** | modetest | drm_device/fb | 复杂 | vblank | vsync | 复杂 | 复杂 | 刷新屏幕 | 7题 |
| **MMC** | mount/dd | mmc_host/block | 复杂 | ✦✦✦✦ | 复杂 | 复杂 | 中 | 读写 SD | 5题 |
| **USB** | lsusb | usb_host/urb | 复杂 | ✦✦✦✦✦ | ✦✦✦✦ | 复杂 | 复杂 | 枚举/传输 | 6题 |
| **PCIe** | lspci | pci_bus/dev | 复杂 | BAR/DMA | MSI | 复杂 | 复杂 | 配置空间 | 5题 |

---

## 五、BSP 开发上下文

### 5.1 DTS 修改后的现实

> 你编译的 DTS 不等于运行时内核看到的 DTS。  
> OP-TEE 和 U-Boot 会在启动时修改 DTS（加入 secure 内存、RIF 配置、bootargs）。

```bash
# 查看内核真正收到的 DTS
dtc -I fs -O dts /sys/firmware/devicetree/base/ | less
```

### 5.2 编译验证路径

```bash
# 最快: 只编译一个 .o
make output/build/linux-custom/drivers/xxx/xxx.o

# 模块部署
scp xxx.ko root@<board>:/tmp/
ssh root@<board> insmod /tmp/xxx.ko

# 只编译 DTB
make dtbs

# 完整编译
make linux-rebuild
```

### 5.3 内核配置

```bash
# 通过 Buildroot 配置界面
make linux-menuconfig

# 或用 config fragment 持久化
echo "CONFIG_XXX=y" > board/stmicroelectronics/common/linux-enable-xxx.config
```
