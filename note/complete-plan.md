# 嵌入式全栈筑基完整计划

> 刘海卫 | 32岁（1994.01）| 2026-06 起
> 背景：8 年 MCU 开发 + 电力电子硕士 + 自学 Linux 驱动
> 最终方向：嵌入式 Linux BSP 驱动工程师 → 35 岁具备全栈能力
> 最终目标：Batocera/Rocknix 移植 + NES 模拟器 + 自己画载板跑起来

---

## 一、背景评估与现实定位

### 已有资产
- 8 年 MCU 深度开发（RTOS、协议栈、驱动、芯片 bring-up）
- 电力电子硕士学历 + SCI 一区论文
- 自学能力已验证（eMMC 系列文档质量）
- 开发板 + Buildroot 全链路已打通
- 鼎阳 SDS804 示波器已购

### 当前短板
- Linux 内核框架缺少体系化理解（中断子系统、DMA 框架等）
- 无 Linux 驱动开发的工作经历
- 缺乏 Linux 驱动面试的"技术话术"

### 求职定位
- **目标岗位**：嵌入式 Linux BSP / 嵌入式软件工程师 / 新能源储能嵌入式
- **投递行业**：工控、新能源储能、机器人公司（投 BSP 岗，不是 FOC 岗）
- **策略**：8 年 MCU 底子 + 自学 Linux 驱动文档输出 → 证明学习能力和潜力
- **时机**：学完中断子系统后开始投递

---

## 二、阶段划分

**时间线概览**：

- **2026.06 - 2026.09** · 突击求职 — Pinctrl收尾 → 中断子系统 → DMA → UART/SPI → I2C → PCIe → USB → **开始投简历**
- **2026.10 - 2027.06** · 站稳脚跟 — DRM/Audio/以太网 → TF-A/U-Boot
- **2027.07 - 2028.06** · 系统+模拟器 — Batocera/Rocknix移植 → C++ NES模拟器
- **2028 年起** · 硬件拓展 — 载板设计 → Qt UI（可选）

---

## 三、阶段一：突击求职（2026.06 - 2026.09）

### 核心目标
学完 I2C/PCIe/USB 后具备 Linux BSP 岗位面试通过能力。**不追求所有子系统精通，追求学过的部分能讲深讲透。**

### 6月（全职，每天 8-10 小时）

**第 1 周：Pinctrl & GPIO 收尾**

| 任务 | 产出 |
|------|------|
| 完成 02-Architecture.md 评审与修订 | 定稿 02 |
| 写 03-Kernel-Source.md（源码分析，probe 流程 + set_mux 路径）| 文档 |
| 写 04-Client-Usage.md（驱动开发者怎么用 pinctrl API + gpiod API）| 文档 |
| 写 bridge 章节：gpio-ranges、pinctrl_select_state、后门机制 | 文档 |

**第 2 周：中断子系统（IRQ）——面试高频区**

| 任务 | 产出 |
|------|------|
| 01-Usage：request_threaded_irq / devm_request_irq / IRQ 上下文（top half / bottom half）| 文档 |
| 02-Architecture：irq domain 层次化（GIC v3）、irq_chip、irq_desc、中断号映射 | 文档 |
| 对照 pinctrl-stm32.c 看 GPIO 中断的注册路径（层次化 irq domain） | 吃透代码 |

**第 3 周：DMA 子系统**

| 任务 | 产出 |
|------|------|
| DMA 01-Usage：dmaengine API、dma_slave_config、device tree 配置 | 文档 |
| DMA 02-Architecture：dma_device、dma_chan、dma_ops、virt-dma 框架 | 文档 |

**第 4 周：UART + SPI**

| 任务 | 产出 |
|------|------|
| UART 串口驱动：serial core 框架、ops 回调、与 DMA 的配合 | 文档 |
| SPI 子系统 01-Usage + 02-Architecture | 文档 |

### 7月（边找全职工作边学）

> 此时你的 Linux 驱动经验已经覆盖了 MCU 最常用外设的框架知识，基本达到 BSP 初面水平。
> 继续学习 I2C/PCIe，同时开始关注市场。

| 学习任务 | 求职动作 |
|---------|---------|
| I2C 子系统（完整文档输出）| 更新简历 |
| PCIe 子系统（配置空间、BAR、MSI-X、RC/EP 模型）| 留意岗位，不急于投递 |

### 8-9月（USB 子系统 + 开始投递）

| 学习任务 | 求职动作 |
|---------|---------|
| USB 子系统（控制器/设备/ Gadget 框架）| **正式投递**，每周 15-20 家 |
| 复习所有学过的子系统，整理面试话术 | 复盘面试问题，补短板 |
| 补齐 Linux 驱动开发的"常见操作"（如 module_param、procfs/debugfs、tracepoint、ftrace）| 把文档挂在 GitHub 上当作品 |

### 面试准备要点

面试官大概率会问的问题（你现在就能用你的知识回答的）：

```
1. 讲一下设备树里 pinctrl-0 是怎么被解析的？
   → 答：pinctrl_dt_to_map → pinctrl_map → pinctrl_setting → ops 回调
   → 你能讲出这段路径，说明你真读过源码

2. GPIO 子系统里 gpiod_get / gpiod_set_value 的完整路径？
   → 答：desc → gdev → chip → set() → BSRR 寄存器

3. 用过哪些调试手段？
   → 答：示波器看时序、debugfs 看 pinmux-pins、tracepoint、printk
```

你不需要所有子系统都懂，但谈到你学过的必须能讲透。这比"什么都听说过但什么都说不深"强得多。

---

## 四、阶段二：站稳脚跟（2026.10 - 2027.06）

### 入职后的第一个半年

| 时间 | 学习内容 | 说明 |
|------|---------|------|
| Q1 2027 | DRM + Audio 子系统 | 多媒体子系统，需要整块时间啃 |
| Q2 2027 | 以太网 + 启动固件（TF-A/OP-TEE/U-Boot）| 网络框架 + 完整启动链路 |

### 入职后的策略

- **不要停止文档输出**：工作上接触到的驱动，用自己的话写成笔记。这不仅巩固知识，也是你下一次跳槽的筹码
- **借工作积累经验**：工作中接触的实际问题（某个外设不工作、DMA 报错、中断延迟过高），都比你自学来得深刻

---

## 五、阶段三：系统移植 + NES 模拟器（2027.07 - 2028.06）

| 项目 | 说明 |
|------|------|
| **Batocera/Rocknix 移植** | 把之前的子系统知识全部串联。理解显示 pipeline、音频、输入、存储、网络如何在完整游戏 OS 下协同 |
| **C++ NES 模拟器** | 有教程，预计 2 个月。跑在开发板上，通过 DRM 输出画面、Audio 输出声音、GPIO 读取按键。打通驱动层到应用层全链路 |

---

## 六、阶段四：硬件拓展（2028 年，有余力再做）

| 项目 | 优先级 | 说明 |
|------|--------|------|
| **载板设计** | P2 | 外购 ST 官方 SOM，自己画底板（USB/HDMI/Audio/ETH/SD），KiCad 或立创EDA |
| **Qt UI** | P2 | 嵌入式 Qt，用到再查，不专门铺时间 |
| **核心板 + GBA 模拟器** | 暂缓 | 35 岁以后有条件再考虑 |

---

## 七、关于工具与硬件

| 工具 | 状态 | 建议 |
|------|------|------|
| 鼎阳 SDS804 示波器 | ✅ 已购，**不退** | 嵌入式通用工具，长期使用 |
| 烙铁 + 热风枪 | 📋 后续购入 | 平时调试和载板焊接都用的上，优先买 |
| 直流稳压电源 | 📋 后续购入 | 做电机驱动或载板调试时需要，不急 |
| 电机控制开源方案 | 📋 业余爱好 | 找工作时不用提，自己玩通了放 GitHub 作为兴趣项目 |

---

## 八、关键原则

1. **面试前不要等"全学完"** — 中断学完就开始投，在面试中学习比一个人闷头学快
2. **深度 > 广度** — 学过的子系统能讲出源码路径、能画结构体关系图，比所有子系统都"知道"有力得多
3. **文档是你最好的简历** — eMMC 和 Pinctrl 系列文档放在 GitHub，面试直接甩链接
4. **不做跟求职方向无关的事** — 电机控制、Qt、GBA 模拟器、核心板设计，这些都是"有余力再做"，6 月全职期间不要碰
5. **实话实说** — 面试时不用说"熟悉 Linux 驱动"，而是说"8 年 MCU 底子，最近半年在系统学 Linux 内核框架，有文档输出，能讲清楚 XXX（挑你学透的说）"。很多面试官吃这一套

---

## 九、项目经历（面试用）

> 以下项目经历基于 STM32MP257 ATK 开发板的真实实践。
> 你在学习每个子系统时，实际做的操作就是：读源码 → 写测试模块 → 编译部署 → 示波器看波形 → debugfs 验证 → 写文档。
> 这些汇总起来就是一段 BSP 驱动开发经历，面试时可以直接讲。

### 1. Pinctrl & GPIO 子系统验证

**STM32MP257 平台 Pinctrl & GPIO 驱动功能验证与调试**

- 基于 STM32MP257 ATK 开发板，分析 Pinctrl 子系统的核心数据结构（pinctrl_desc/pinctrl_dev/pin_desc）和 GPIO 子系统的三层架构（gpio_device/gpio_chip/gpio_desc），输出完整的技术分析文档
- 通过 debugfs 接口（`/sys/kernel/debug/pinctrl/*/pinmux-pins`）验证引脚复用配置的正确性，排查 DTS 中 pinmux 配置与实际寄存器值的一致性
- 编写内核测试模块，调用 gpiod_* API 实现 GPIO 输出控制与输入捕获，配合示波器测量 BSRR/ODR/IDR 寄存器的实际电平变化，验证 ACTIVE_LOW 标志和 open-drain 模式的硬件行为
- 深入分析 gpio-ranges 的映射机制，验证 GPIO 全局编号（0~185）与 Pinctrl pin 号的对应关系，理解后门机制中 gpio_set_direction 回调写 MODER 寄存器的完整路径
- 对比 STM32（合体架构）与 i.MX6ULL（分离架构）的硬件差异，分析不同 SoC 架构对 Pinctrl 和 GPIO 驱动设计的影响

### 2. 中断子系统验证

**STM32MP257 平台中断子系统分析与 GPIO 中断功能验证**

- 分析 Linux 中断子系统的核心框架，包括 irq_desc、irq_domain、irq_chip 等核心数据结构，以及 GICv3 中断控制器的层次化 irq domain 设计
- 研究 STM32MP257 Pinctrl 驱动的中断注册路径，分析其通过 `irq_domain_create_hierarchy()` 创建层次化 irq domain 的实现方式，理解 GPIO bank 作为中断控制器的完整链路
- 编写内核测试模块，注册 GPIO 中断处理函数，验证边沿触发和电平触发的中断响应行为，配合示波器测量中断延迟
- 对比 threaded IRQ、tasklet、workqueue 等不同 bottom half 机制的实际表现，总结中断上下文的使用场景和约束

### 3. DMA 子系统验证

**STM32MP257 平台 DMA 驱动功能验证**

- 分析 Linux DMA 引擎框架（dmaengine），研究 dma_device、dma_chan、dma_ops 等核心结构体，以及 virt-dma 辅助框架的设计思想
- 编写测试用例验证 STM32MP257 MDMA 控制器的数据传输功能，包括 memory-to-memory、peripheral-to-memory 等传输模式
- 结合示波器分析 DMA 触发信号与外设数据就绪的时序关系，验证 DMA 传输完成中断的响应路径

### 4. UART 串口驱动验证

**STM32MP257 平台 UART 驱动功能验证与调试**

- 分析 Linux serial core 框架，研究 uart_port、uart_ops 等核心结构体，理解串口驱动与 TTY 层的交互机制
- 在 ATK 开发板上验证 UART 收发功能，通过 loopback 测试和串口工具验证波特率精度、流控配置
- 配合示波器测量 TX/RX 引脚的波形，分析 UART 帧格式（起始位、数据位、校验位、停止位）的实际时序
- 调试 DTS 中 pinmux 配置与串口节点的关联，验证 pinctrl_select_state 在串口 probe 阶段的自动调用

### 5. SPI 子系统验证

**STM32MP257 平台 SPI 驱动功能验证**

- 分析 Linux SPI 核心框架，研究 spi_controller、spi_device、spi_transfer 等核心数据结构，理解 SPI 消息队列和 DMA 传输的整合方式
- 编写 SPI 设备驱动或使用 spidev 用户态接口，验证 SPI 不同时钟极性（CPOL）和相位（CPHA）模式下的数据收发正确性
- 配合示波器测量 SCK、MOSI、MISO、CS 引脚的时序波形，分析 SPI 传输速率和信号完整性
- 验证 SPI 中断模式和 DMA 模式两种传输路径的性能差异

### 6. I2C 子系统验证

**STM32MP257 平台 I2C 驱动功能验证**

- 分析 Linux I2C 核心框架，研究 i2c_adapter、i2c_algorithm、i2c_client 等核心数据结构，理解 I2C core 的总线仲裁和报文传输机制
- 外接 I2C 传感器（如温湿度传感器），编写用户态程序通过 /dev/i2c-N 接口完成寄存器读写验证
- 配合示波器测量 SCL/SDA 引脚的时序，分析起始条件、从机地址、ACK/NACK、停止条件等 I2C 协议要素的硬件波形
- 编写简单的 I2C 客户端驱动，验证 probe 和 remove 生命周期与设备树的匹配流程

### 7. PCIe 子系统分析

**STM32MP257 平台 PCIe 控制器驱动分析**

- 分析 Linux PCIe 子系统核心框架，包括配置空间访问机制、BAR 空间映射、MSI/MSI-X 中断、DMA 传输等关键概念
- 研究 STM32MP257 的 PCIe 控制器（RC 模式）驱动的 probe 流程，分析 DTS 配置与控制器初始化的对应关系
- 接上 PCIe 外设（如 NVMe SSD 或无线网卡），验证设备枚举、配置空间读取、BAR 分配和中断分配等 PCIe 枚举流程
- 分析 PCIe 拓扑结构（Root Complex → Switch → Endpoint）以及 Linux 中对应的数据结构管理方式

### 8. USB 子系统验证

**STM32MP257 平台 USB 驱动功能验证**

- 分析 Linux USB 子系统核心框架，包括 USB 主机控制器驱动（HCD）、USB 设备驱动模型、urb 数据结构与传输流程
- 验证 STM32MP257 开发板的 USB Host 功能，接入 U 盘等标准设备，分析设备枚举过程的完整链路（描述符解析、配置选择、驱动匹配）
- 编写 USB 测试用例，验证批量传输、中断传输和控制传输的完整路径，配合 USB 分析仪（或逻辑分析仪）分析总线时序
- 分析 DWC3 控制器驱动的 probe 流程和 gadget 框架的配置方式

### 9. DRM 显示子系统分析

**STM32MP257 平台 DRM 显示驱动验证**

- 分析 Linux DRM/KMS 子系统核心框架，研究 drm_device、drm_crtc、drm_encoder、drm_connector、drm_plane 等核心对象及其关系
- 验证 ATK 开发板的显示输出（RGB/LVDS/MIPI DSI 接口），通过 modetest 等工具测试不同分辨率和刷新率的显示效果
- 分析 STM32MP257 LTDC 显示控制器的 probe 流程和设备树配置，理解显示 pipeline（GPU → LTDC → 接口 → 面板）的完整链路
- 编写简单的 framebuffer 应用，验证 mmap 映射和 page-flip 的显示更新机制

### 10. Audio 子系统分析

**STM32MP257 平台 Audio 驱动验证**

- 分析 Linux ASoC 子系统核心框架，研究 soc_card、soc_dai_link、snd_soc_dai 等核心数据结构，理解 ASoC 的分层设计（Codec → DAI → DMA）
- 外接音频 Codec（如 ES8388 或 WM8960），验证 I2S 接口的音频播放与录音功能
- 分析音频驱动与 DMA 的配合方式，理解音频数据从用户态应用 → ALSA core → DMA → I2S → Codec 的完整路径

### 11. V4L2 子系统分析

**STM32MP257 平台 V4L2 驱动验证**

- 分析 Linux V4L2 子系统核心框架，研究 video_device、videobuf2、v4l2_device 等核心数据结构
- 外接摄像头模组，验证 V4L2 的 buffer 管理机制（mmap/userptr/DMABUF）和视频数据采集流程

### 12. 以太网子系统验证

**STM32MP257 平台以太网 MAC/PHY 驱动验证**

- 分析 Linux net 子系统核心框架，研究 net_device_ops、NAPI、sk_buff 等核心数据结构，理解网络数据包从硬件到协议栈的完整路径
- 验证 ATK 开发板以太网接口（YT8531 PHY）的通信功能，通过 ping/tcp/udp 测试验证网络吞吐量和稳定性
- 分析 stmmac（Synopsys DesignWare MAC）驱动的 probe 流程和 DMA 描述符管理，配合示波器测量 RGMII 接口时序

---

## 十、OTA 升级子系统（独立系列）

> 放在阶段二（站稳脚跟期），学完 TF-A/U-Boot 启动固件后开始。
> 按驱动学习的四步走规范输出完整系列文章，与 eMMC/Pinctrl 系列同等深度。

### 背景

ATK 板 BSP 已有完整的 RAUC OTA 基础设施：
- `st_stm32mp257d_atk_ota_defconfig` — OTA 配置
- `manifest.raucm` — bundle 打包清单（rootfs + fip 双镜像）
- `generate-rauc-bundle.sh` — 生成 `.raucb` 升级包
- RAUC system.conf — A/B 槽位定义，FWU metadata 后端
- `S99rauc-mark-good` — 启动标记脚本
- `uEnv.txt` — U-Boot A/B 双槽引导逻辑

### 文档系列规划（note/ota/）

| 篇号 | 标题 | 内容 |
|------|------|------|
| 00 | History | OTA 方案演化：MCU IAP → Linux A/B 分区 → RAUC → 容器化升级 |
| 01 | Usage | RAUC 命令行用法、证书生成、bundle 打包、本地升级操作 |
| 02 | Architecture | A/B 分区布局、FWU metadata、U-Boot 双槽引导、RAUC system.conf 设计 |
| 03 | TFA-Boot | TF-A 中的 FWU metadata 读写流程，槽位切换决策逻辑 |
| 04 | UBoot | U-Boot 中 A/B 引导实现，uEnv.txt 双槽脚本分析 |
| 05 | RAUC | RAUC 源码级分析：bundle 格式（verity）、slot 状态机、install 流程、hook 机制 |
| 06 | WebUI | 嵌入式 Web 服务器 + 升级管理界面实现 |
| 07 | Practice | 完整移植到 ATK 板的实操记录，掉电/坏包/回滚测试 |

### 简历写法

> **STM32MP257 平台 Linux OTA 升级方案设计与实现**
>
> - 基于 RAUC 框架构建 A/B 双分区 OTA 升级系统，配合 TF-A FWU metadata 和 U-Boot 双槽引导实现无缝切换与异常回滚
> - 设计 GPT 分区布局（fsbl/fip/u-boot-env/rootfs-a/rootfs-b），通过 genimage 生成 SD 卡镜像
> - 生成自签名证书链，配置 RAUC verity bundle 格式，实现升级包的签名验证和完整性校验
> - 编写槽位标记脚本，实现启动时自动确认槽位状态，防止损坏固件反复重启
> - 搭建嵌入式 Web 服务器，开发 OTA 升级管理界面，支持通过浏览器上传 .raucb 包并触发升级
> - 完整模拟升级失败场景（掉电、损坏包、网络中断），验证 A/B 回滚机制的有效性

---

| 时间 | 修改内容 |
|------|---------|
| 2026-05-31 | 初版创建 |
