# DMA 子系统学习笔记 — 写作规划

> 本文档是 `note/dma/` 系列笔记的总体规划。
>
> 分析对象：STM32MP257D (Cortex-A35)，Linux v6.6.78 (stm32mp-r2)
> DMA 控制器：HPDMA (st,stm32-dma3) × 3 实例 + DMAMUX + MDMA (MCE)
>
> 系列定位：**源码驱动，无需外部资料**。每一章的内容边界由源码文件划分决定。
>
> **共 5 篇**：00-History → 01-Usage → 02-Architecture → 03-SourceAnalysis → 04-Scenario

---

## 核心决策：内容边界

### 包含什么

**A. DMA 数据搬运引擎框架（dmaengine）**
- dmaengine 核心框架 (`drivers/dma/dmaengine.c` + `include/linux/dmaengine.h` + `include/linux/of_dma.h`)
- virt-dma 辅助框架 (`drivers/dma/virt-dma.h`)
- STM32 HPDMA (stm32-dma3) 驱动 —— 从 dmaengine core 到硬件寄存器的完整路径
- STM32 DMAMUX 驱动 —— 外围设备请求到 DMA 通道的路由
- STM32 MDMA 驱动 —— MCE (Master Copy Engine) 大块数据搬运
- dmatest (`drivers/dma/dmatest.c`) —— 调试验证手段

**B. DMA 内存映射 API（dma-mapping）—— 重点**
- 一致性和流式 DMA 映射：`dma_alloc_coherent` / `dma_map_single` / `dma_map_sg`
- Cache 同步原语：`dma_sync_single_for_cpu` / `dma_sync_single_for_device`
- DMA 掩码与寻址能力：`dma_set_mask` / `dma_set_coherent_mask`

dma-mapping API 和 dmaengine 分工不同、但缺一不可：
```
dma-mapping API                dmaengine
  管 buffer 映射                管数据搬运
  ┌──────────┐               ┌──────────────┐
  │ coherent │               │ dma_request_  │
  │ stream    │  mapped buf   │ slave_channel │
  │ mapping   │ ───────────→ │ prep_slave_sg │
  │ sync      │               │ issue_pending │
  └──────────┘               │ wait/complete │
                              └──────────────┘
```

**C. 外设侧 DMA 用法 —— 重点**
- SPI + DMA（slave_sg 模式）：SG 链表传输，LLI 自动切换
- 音频 + DMA（cyclic 模式）：循环 buffer，period 中断
- UART + DMA（slave_sg 模式）：RX/TX 通道分离
- 这些用法是 DMA 子系统的核心价值所在，在每个对应章节中展开

**D. 用户态 DMA（dma-buf / dma-heap）—— 重点**
- **dma-heap**：用户态通过 `/dev/dma_heap/*` 分配 DMA buffer → 拿到 dmabuf fd
- **dma-buf**：内核 buffer 共享框架，fd 在不同驱动/进程间传递
- **udmabuf**：用户态内存 → DMA buffer 的桥接
- 用户态 DMA 把整个链路打通：**用户分配→内核搬运→用户消费**
- 相关源码：`drivers/dma-buf/dma-heap.c`(326行)、`dma-buf.c`(1734行)、`udmabuf.c`(397行)、`drivers/dma-buf/heaps/`

### 不纳入系列的内容
- （无）—— 内核态 dmaengine/dma-mapping + 用户态 dma-buf/dma-heap 共同构成完整的 DMA 子系统

### 为什么把这些放在同一系列

DMAMUX、HPDMA、dma-mapping、外设 DMA 是同一个完整链路的不同层次：

```
外设 FIFO 数据就绪
  → DMAMUX 选择路由（外设请求号 → HPDMA 通道号）
  → HPDMA 从外设 FIFO 搬运到内存
  → 但搬运的前提是：内存 buffer 已经通过 dma-mapping API 准备好
  → 并且外设驱动需要在传输完成后处理数据
```

所以本系列的内容组织方式是：
| 层次 | 管什么 | 覆盖哪里 |
|------|--------|---------|
| **用户态**：dma-heap/dma-buf | 用户分配 DMA buffer + fd 传递 + mmap | 01-Usage、02-Architecture、04-Scenario |
| **内核态**：dma-mapping API | buffer 准备：申请/映射/同步 | 01-Usage、02-Architecture |
| **内核态**：dmaengine 框架 | 传输控制：描述符/提交/启动 | 01-Usage、02-Architecture |
| **硬件**：HPDMA + DMAMUX + MDMA | 硬件实现注册到 dmaengine | 02-Architecture、03-SourceAnalysis |
| **实践**：外设 DMA 用法 | SPI/音频/UART 如何调用 dmaengine | 01-Usage、04-Scenario |

完整的数据流向：
```
用户态应用
  ├── ioctl(DMA_HEAP_IOCTL_ALLOC) → dmabuf fd    ← dma-heap 分配
  ├── mmap(dmabuf fd) → 读写 buffer               ← 用户直接访问
  ├── ioctl(DEV, ... dmabuf fd) → 驱动处理        ← fd 传给内核驱动
  │     └── 驱动内部:
  │           ├── dma_buf_attach() / dma_buf_map_attachment()  ← dma-buf 导入
  │           ├── dmaengine_prep_slave_sg() → 搬运             ← dmaengine 传输
  │           └── dma_buf_unmap_attachment()                   ← 完成
  └── close(dmabuf fd)    ← 释放
```
MDMA 则是独立的 MCE 引擎，专注于大块 memory-to-memory 搬运，与 HPDMA 共用 dmaengine 框架但硬件完全不同，在架构篇对比介绍。

---

## 源码文件总索引

| 文件 | 行数 | 覆盖篇目 | 核心函数/结构体 |
|------|------|---------|----------------|
| `include/linux/dmaengine.h` | 1639 | 01, 02, 04 | dma_device、dma_chan、dma_async_tx_descriptor、dma_slave_config |
| `include/linux/of_dma.h` | — | 01, 03 | of_dma_controller_register、of_dma_xlate_by_chan_id |
| `drivers/dma/dmaengine.c` | 1605 | 02, 03, 04 | dma_async_device_register、dma_chan_get、dma_cookie_* |
| `drivers/dma/dmaengine.h` | — | 02 | dmaengine 内部辅助函数 |
| `drivers/dma/virt-dma.h` | 227 | 02 | virt_dma_chan、virt_dma_desc、5 状态链表 |
| `drivers/dma/stm32/stm32-dma3.c` | 2428 | 01, 02, 03, 04 | stm32_dma3_chan、stm32_dma3_lli、stm32_dma3_probe |
| `drivers/dma/stm32/stm32-dmamux.c` | 402 | 01, 02, 03 | dma_router、stm32_dmamux_data、route/free 回调 |
| `drivers/dma/stm32/stm32-mdma.c` | 1830 | 02(简), 04(对) | 与 HPDMA 对比为主，不深入源码路径 |
| `drivers/dma/dmatest.c` | — | 01 | dmatest 模块参数和用法 |
| `arch/arm64/boot/dts/st/stm32mp251.dtsi` | ~3400 | 01 | hpdma/hpdma2/hpdma3 节点定义 |
| `include/linux/dma-mapping.h` | — | 01, 02 | dma_alloc_coherent、dma_map_single、dma_map_sg |
| `include/asm-generic/dma-mapping.h` | — | 02 | dma_map_ops、dma_direct_ops/dma_ioMMU_ops |
| `kernel/dma/mapping.c` | — | 01, 02, 04 | dma_alloc_coherent 实现、dma_map_page_attrs |
| `kernel/dma/direct.c` | — | 02, 04 | 直接映射（无 IOMMU）路径 |
| `include/uapi/linux/dma-heap.h` | 53 | 01 | struct dma_heap_allocation_data、DMA_HEAP_IOCTL_ALLOC |
| `drivers/dma-buf/dma-heap.c` | 326 | 01, 02, 03 | dma_heap_alloc、dma_heap_open、/dev/dma_heap/ 设备 |
| `drivers/dma-buf/dma-buf.c` | 1734 | 02, 04 | dma_buf_attach、dma_buf_mmap、dma_buf_fd、fd 生命周期 |
| `drivers/dma-buf/udmabuf.c` | 397 | 01 | UDMABUF_CREATE ioctl、memfd→dmabuf |
| `include/linux/dma-buf.h` | — | 02 | struct dma_buf、struct dma_buf_attachment、struct dma_buf_ops |
| `drivers/dma-buf/heaps/system_heap.c` | — | 01, 03 | 系统堆分配器实现 |
| `drivers/dma-buf/heaps/cma_heap.c` | — | 01, 03 | CMA 堆分配器实现 |
| `include/uapi/linux/dma-buf.h` | — | 01 | dma-buf 用户态接口（如果存在） |
| `tools/testing/selftests/dmabuf-heaps/dmabuf-heap.c` | — | 01, 04 | dma-heap 用户态测试程序 |
| `arch/arm64/boot/dts/st/stm32mp257f-dk.dts` | — | 01, 04 | 板级 reserved memory for LLI + 外设 dmas 属性示例 |

### 硬件资源总结（STM32MP257）

| 控制器 | 实例 | 通道数 | 中断线/实例 | 主要用途 |
|--------|------|--------|------------|---------|
| HPDMA (stm32-dma3) | 3 (hpdma/hpdma2/hpdma3) | 16 通道 × 3 = 48 通道 | 16 条 SPI IRQ | 外设 DMA (UART/SPI/I2C/SDMMC 等) |
| DMAMUX | 1 | 8 请求/HPDMA × 3 + 保留 | 无 | 外设请求号 ↔ HPDMA 通道映射 |
| MDMA | 1 (通过 MCE) | 多通道 | 独立 IRQ | 大块 memory-to-memory 搬运 |

---

## 第 0 篇：00-History.md — DMA 子系统演进史

**核心问题**：Linux DMA 引擎框架为什么设计成今天这样？dmaengine vs 旧式 DMA API，virt-dma 解决了什么？

### 大纲

| 章节 | 时间 | 核心变化 | 涉及源码 |
|------|------|---------|---------|
| 1.1 前传：arch-dependant DMA | v2.4 之前 | 各架构各自实现 DMA API，无统一框架 | `arch/*/kernel/dma.c` |
| 1.2 dmaengine 框架诞生 | v2.6.17 (2006) | `drivers/dma/dmaengine.c` 引入，统一 `struct dma_device`/`struct dma_chan` | `dmaengine.c`、`dmaengine.h` |
| 1.3 slave DMA 的加入 | v2.6.29 (2009) | `device_prep_slave_sg` + `dma_slave_config`，外设 DMA 成为一等公民 | `include/linux/dmaengine.h` |
| 1.4 virt-dma 辅助框架 | v3.2 (2011) | Russell King 引入 `virt_dma_chan`/`virt_dma_desc`，大量驱动复用 | `virt-dma.h` |
| 1.5 cyclic DMA 支持 | v3.6 (2012) | `device_prep_dma_cyclic` 加入，音频 DMA 得到支持 | `dmaengine.h` |
| 1.6 DMA Router 框架 | v3.19 (2015) | DMAMUX/crossbar 需求推动，`struct dma_router` 标准化 | `of_dma.c`、`dmaengine.c` |
| 1.7 dmatest 标准化 | v3.7+ | `drivers/dma/dmatest.c` 成为通用测试模块 | `dmatest.c` |
| 1.8 新一代 DMA 控制器 | 2020s | 基于 linked-list 的复杂 DMA（STM32 DMA3、TI PKTDMA 等） | 链式传输、LLI 管理 |
| **1.9 dma-buf 框架诞生** | **v3.3 (2012)** | **`drivers/dma-buf/dma-buf.c` 引入，统一 buffer 共享语义** | **dma-buf.c、dma-buf.h** |
| **1.10 dma-heap 用户态 API** | **v5.6 (2020)** | **`/dev/dma_heap/*` 用户态分配 DMA buffer，dmabuf fd 机制** | **dma-heap.c、uapi dma-heap.h** |
| **1.11 udmabuf** | **v5.13+** | **用户态 memfd → dmabuf，与 v4l2/drm 等配合** | **udmabuf.c** |

### 写作要点
- 每个阶段回答"为什么需要这个机制"
- STM32 DMA3 是"现代 DMA 控制器"的代表——LLI + DMAMUX + virt-dma
- dma-buf/dma-heap 是"从内核驱动到用户态应用"的桥梁

---

## 第 1 篇：01-Usage.md — DMA 使用方法和 DTS 配置

**核心问题**：驱动开发者在 STM32MP257 上怎么使用 DMA？有哪些 API 入口？怎么调试？

### 源码依据

| 功能 | 源码文件 | 核心函数/接口 |
|------|---------|--------------|
| **dma-mapping API** | | |
| 一致性映射 | `kernel/dma/mapping.c` | `dma_alloc_coherent()` / `dma_free_coherent()` |
| 流式映射 | `kernel/dma/mapping.c` | `dma_map_single()` / `dma_unmap_single()` |
| SG 映射 | `kernel/dma/mapping.c` | `dma_map_sg()` / `dma_unmap_sg()` |
| Cache 同步 | `kernel/dma/mapping.c` | `dma_sync_single_for_cpu/device()` |
| DMA 掩码 | `kernel/dma/mapping.c` | `dma_set_mask()` / `dma_set_coherent_mask()` |
| **dmaengine API** | | |
| 分配 DMA 通道 | `dmaengine.c` | `dma_request_chan()` / `dma_request_slave_channel()` |
| 配置 slave 参数 | `dmaengine.h` | `dmaengine_slave_config()`、`struct dma_slave_config` |
| 准备传输描述符 | 驱动实现 | `dmaengine_prep_slave_sg()`、`dmaengine_prep_dma_cyclic()` |
| 提交 + 启动 | `dmaengine.c` | `dmaengine_submit()` → `dma_async_issue_pending()` |
| 等待完成 | `dmaengine.h` | `dma_wait_for_async_tx()` / 异步回调 |
| 释放通道 | `dmaengine.c` | `dma_release_channel()` |
| 内存搬运 | 驱动实现 | `dmaengine_prep_dma_memcpy()` |
| 调试 | `dmatest.c` | 内核模块参数、`/sys/kernel/debug/dmaengine/` |
| DTS 配置 | `stm32mp251.dtsi` | hpdma 节点、`dmas`/`dma-names` 属性 |
| **外设 DMA 用法的参考驱动** | | |
| SPI + DMA | `drivers/spi/spi-stm32.c` | stm32_spi_transfer_one_dma |
| UART + DMA | `drivers/tty/serial/stm32-usart.c` | stm32_usart_dma_xxx |
| I2C + DMA | `drivers/i2c/busses/i2c-stm32f7.c` | stm32_i2c_swap_setup_dma |
| Audio cyclic | `sound/soc/stm/stm32_*.c` | snd_dmaengine_pcm_trigger |

### 大纲

| 章节 | 内容 | 关键点 |
|------|------|--------|
| **1.0 DMA 的两面：dma-mapping 与 dmaengine** | | |
| 1.0.1 分工关系图 | buffer 准备（dma-mapping） vs 搬运（dmaengine） | 为什么需要两套 API |
| 1.0.2 典型 DMA 使用的三段式 | ①映射 buffer → ②编排传输 → ③启动等待 | 完整代码骨架 |
| **1.1 场景引导** | 三种典型 DMA 使用场景 | 外设数据搬运、memcpy 加速、cyclic(音频) |
| **1.2 DTS 配置** | | |
| 1.2.1 DMA 控制器节点 | hpdma/hpdma2/hpdma3 的 3 cell 含义 | `<&hpdma REQ_ID CHANNEL_CFG LLI_CFG>` |
| 1.2.2 外设 dmas 属性 | USART/I2C/SPI/SDMMC 的 dmas + dma-names | 请求号来源（DMAMUX 映射表） |
| 1.2.3 Reserved Memory for LLI | 板级 dts 中 `memory-region` 配置 | LLI 池需要连续的物理内存 |
| 1.2.4 3 cell 字段详解 | `<0x62 0x00003121>` 逐位解析 | 触发选择、优先级、FIFO、数据宽度等 |
| **1.3 DMA 内存分布与分配策略** | | |
| 1.3.1 DMA 地址空间总览 | 物理内存布局 & DMA 可用范围 | STM32MP257 内存映射、DMA 能访问的地址范围 |
| 1.3.2 ZONE_DMA / ZONE_DMA32 | `include/linux/mmzone.h` 中 zone 划分 | 为什么有这些 zone、`dma_set_mask` 如何决定 zone |
| 1.3.3 CMA (Contiguous Memory Allocator) | 大块物理连续内存分配 | `cma=xxx` 内核参数、DTS `reserved-memory` + `reusable` |
| 1.3.4 dma_pool / dma_pool_alloc | `include/linux/dmapool.h` | 小尺寸、固定大小的 DMA buffer 池 |
| 1.3.5 reserved-memory 专用区域 | DTS 中 `no-map` / `pool` 区域 | 给特定 DMA 外设预留的物理内存 |
| 1.3.6 分配路径总结 | 驱动代码中如何选择 | `dma_alloc_coherent` vs `dma_pool_alloc` vs `dma_map_sg` 选择树 |
| **1.4 dma-mapping API（buffer 准备）** | | |
| 1.4.1 一致性映射 | `dma_alloc_coherent()` | 适合 DMA+CPU 都频繁访问的场景 |
| 1.4.2 流式映射 | `dma_map_single()` / `dma_map_sg()` | 单次传输场景，需要 cache 同步 |
| 1.4.3 Cache 同步 API | `dma_sync_single_for_cpu/device()` / `dma_sync_sg_*()` | 何时调用、cache line 对齐要求、刷回/失效 |
| 1.4.4 什么时候用哪个 | 对比表格 + 代码选择逻辑 | 频率、大小、延迟要求 |
| 1.4.5 DMA 掩码 | `dma_set_mask_and_coherent()` | 告知硬件能访问的地址范围 |
| 1.4.6 实际例子：SPI DMA | coherent vs streaming 在 SPI 中的实际选择 | 驱动源码中的真实用法 |
| **1.5 dmaengine API（传输控制）** | | |
| 1.4.1 通道生命周期 | `dma_request_chan` → config → prep → submit → issue | 状态机：requested → configured → submitted → issued → active → completed |
| 1.4.2 slave_sg 传输 | `dmaengine_prep_slave_sg()` 完整参数 | scatterlist 的构造、方向和 flag |
| 1.4.3 cyclic 传输 | `dmaengine_prep_dma_cyclic()` 参数 | period_len 与 buf_len 的关系、音频应用 |
| 1.4.4 memcpy 传输 | `dmaengine_prep_dma_memcpy()` | 纯内存搬运、对齐要求 |
| 1.4.5 异步完成机制 | 回调函数 + cookie 轮询 | `dma_async_tx_callback`、`dma_wait_for_async_tx()` |
| 1.4.6 同步原语 | `dmaengine_terminate_sync()` / `dmaengine_synchronize()` | 终止后的安全回收 |
| **1.5 外设 DMA 用法实战（重点）** | | |
| 1.5.1 SPI + slave_sg | SPI 传输的完整 DMA 路径 | `spi_stm32_transfer_one_dma` 源码逐行 |
| 1.5.1 SPI + slave_sg | SPI 传输的完整 DMA 路径 | `spi_stm32_transfer_one_dma` 源码逐行 |
| 1.5.2 UART + slave_sg | UART RX/TX 双通道 DMA | `stm32_usart_dma_xxx` 源码逐行 |
| 1.5.3 Audio + cyclic | 音频播放的 cyclic DMA 用法 | period 回调、乒乓 buffer |
| 1.5.4 I2C + DMA | I2C 大块数据传输的 DMA 路径 | `stm32_i2c_swap_setup_dma` |
| **1.6 用户态 DMA API（重点）** | | |
| 1.6.1 dma-heap 分配 | `open("/dev/dma_heap/system") → ioctl(DMA_HEAP_IOCTL_ALLOC)` | 分配 DMA buffer → 拿 dmabuf fd |
| 1.6.2 dma-heap 类型 | system_heap vs cma_heap | 物理连续 vs 非连续、性能差异 |
| 1.6.3 dmabuf fd 生命周期 | fd → mmap → 传递给内核驱动 → close | 引用计数、共享语义 |
| 1.6.4 udmabuf | `ioctl(UDMABUF_CREATE)` | 用户态内存 → dma-buf |
| 1.6.5 dma-buf mmap | `mmap(dmabuf_fd)` | 用户态直接访问 DMA buffer |
| 1.6.6 完整用户态 DMA 示例 | open heap → alloc → mmap → ioctl(驱动) | 用户态代码骨架 |
| **1.7 调试手段** | | |
| 1.7.1 dmatest 模块 | insmod dmatest.ko 参数配置 | 通道选择、传输大小、迭代次数 |
| 1.7.2 debugfs 接口 | `/sys/kernel/debug/dmaengine/` → `/dma_buf/` | 查看通道状态、dmabuf 引用 |
| 1.7.3 tracepoint | `fbtrace/dmaengine:dma.*` + `dmabuf:*` events | 传输生命周期追踪 |
| 1.7.4 寄存器级调试 | devmem 查看 HPDMA 寄存器 | CCR/CSR/CBR1 等状态寄存器解读 |
| 1.7.5 dmabuf 调试 | `/sys/kernel/debug/dma_buf/` | 查看当前所有 dmabuf 分配、引用计数 |

---

## 第 2 篇：02-Architecture.md — DMA 核心框架与数据结构

**核心问题**：dmaengine 框架的骨架是什么？dma_device、dma_chan、virt-dma、dma_router 如何配合？

```
                    ┌──────────────────┐
                    │    dma_device     │
                    │  (DMA 控制器)     │
                    │  .channels 链表   │
                    │  .cap_mask        │
                    │  .device_* 回调   │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ dma_chan  │  │ dma_chan  │  │ dma_chan  │
        │ (通道 0)  │  │ (通道 1)  │  │ (通道 15) │
        │ .cookie   │  │ .cookie   │  │ .cookie   │
        │ .dev      │  │ .dev      │  │ .dev      │
        └────┬─────┘  └──────────┘  └──────────┘
             │
    ┌────────┴───────── virt-dma 封装（stm32-dma3 使用）
    ▼
┌─────────────────────────────────────────────┐
│          virt_dma_chan                       │
│  ├── chan (dma_chan)                         │
│  ├── task (tasklet, 完成回调调度)              │
│  ├── lock (spinlock)                         │
│  ├── desc_allocated  链表  ← prep 放入        │
│  ├── desc_submitted  链表  ← submit 放入      │
│  ├── desc_issued     链表  ← issue_pending 放入│
│  ├── desc_completed  链表  ← ISR 放入          │
│  └── desc_terminated 链表  ← terminate 放入    │
└─────────────────────────────────────────────┘

DMA 路由器:
┌──────────┐     DMAMUX (dma_router)     ┌──────────┐
│ 外设请求号 ├──→  stm32_dmamux_data  ───→│ HPDMA 通道│
│ (0~254)   │     .route / .free        │ (0~15)   │
└──────────┘                             └──────────┘
```

### 大纲

| 章节 | 源码文件/结构体 | 内容 |
|------|----------------|------|
| **2.x DMA 内存区域与地址空间** | | |
| **2.1 物理内存布局与 DMA zone** | `include/linux/mmzone.h` ZONE_DMA/ZONE_DMA32/ZONE_NORMAL | 为什么有 ZONE_DMA（ISA 24位）→ ZONE_DMA32（32位外设）→ 现代 SoC 上的情况 |
| **2.2 dma_set_mask 与 zone 选择** | `kernel/dma/mapping.c` `dma_direct_supported` | 驱动设 32bit mask → ZONE_DMA32；设 24bit mask → ZONE_DMA；设不对可能分配失败 |
| **2.3 CMA 架构** | `kernel/dma/contiguous.c` + `mm/cma.c` | CMA 区域初始化、page migration 腾出连续内存、reusable 属性 |
| **2.4 dma_pool 架构** | `drivers/dma/dma-pool.c` | 小 buffer 的 SLAB-like 分配器、减少碎片 |
| **2.5 reserved-memory 与 DMA** | DTS `reserved-memory` + `no-map` | 给特定硬件预留的专用内存区域（如 HPDMA LLI 池） |
| **2.x Cache 管理与一致性（展开重点）** | | |
| **2.6 ARM64 cache 层次** | L1/L2 cache line 大小（64B）、write-back 策略 | cache 命中和缺失路径、DMA 为什么必须关心 cache |
| **2.7 非一致性总线模型** | 大部分嵌入式 SoC 的 AXI 总线 | DMA 直接读写物理内存 vs CPU 通过 cache 读写 → 数据不一致 |
| **2.8 coherent 映射的实现** | `kernel/dma/direct.c` `__dma_direct_alloc_pages` | ARM64 上通过页表属性（PTE_MAIR_NORMAL_NC）把页面设为 Non-cacheable，或使用 CMA 区 |
| **2.9 streaming 映射的 cache 维护** | `kernel/dma/direct.c` `dma_direct_map_page` | `dma_map_single` 内部调用架构相关的 `arch_sync_dma_for_device` → ARM64 上发出 `dc cvac` 指令刷 cache |
| **2.10 ARM64 cache 维护指令** | `arch/arm64/mm/cache.S` | `dc civac`（清理+失效）、`dc cvac`（清理）、`dc cvau`（清理到 PoU）、`ic ivau`（指令 cache） |
| **2.11 dma_sync_* 的完整路径** | `dma_sync_single_for_cpu` → `arch_sync_dma_for_cpu` → `dc civac` | 从驱动 API 到具体 cache 指令的完整调用链 |
| **2.12 DMA 地址空间模型** | CPU 虚拟地址 vs 物理地址 vs 总线/DMA 地址 | IOMMU/SMMU 映射、直接映射（STM32MP257 无 SMMU） |
| **2.13 dma_map_ops 虚函数表** | `include/asm-generic/dma-mapping.h` | direct ops vs iommu ops、STM32MP257 上的路径 |
| **2.14 各映射方式的选择决策树** | 结合外设场景 | 何时 coherent、何时 streaming、何时 dma_pool、何时 CMA |
| **2.x dmaengine 核心架构（原有内容）** | | |
| **2.7 三层架构总览** | 消费者 → Core → 控制器驱动 | dmaengine 的分层设计思想 |
| **2.8 struct dma_device** | `include/linux/dmaengine.h` L854 | cap_mask 能力位图、channels 链表、ops 回调表（~20 个回调） |
| **2.9 struct dma_chan** | `include/linux/dmaengine.h` L328 | cookie 计数、percpu 本地状态、slave 配置指针 |
| **2.10 struct dma_async_tx_descriptor** | `include/linux/dmaengine.h` | 传输描述符：cookie/callback/status/phys 地址 |
| **2.11 dmaengine 同步模型** | cookie 机制 | submit 分配 cookie → issue_pending → ISR 回写 completed_cookie |
| **2.12 dma_cap_mask_t 能力位图** | `include/linux/dmaengine.h` L227 | DMA_MEMCPY/DMA_SLAVE/DMA_CYCLIC 等 17 种能力 |
| **2.13 virt-dma 框架** | `drivers/dma/virt-dma.h` | 5 状态链表 + tasklet 回调调度 |
| **2.14 dma_slave_config** | `include/linux/dmaengine.h` | src_addr/dst_addr/addr_width/maxburst 等配置 |
| **2.15 DMA Router 框架** | `stm32-dmamux.c` | dma_router、route/free 回调、请求号到通道号的映射 |
| **2.16 dma_chan 分配机制** | `dmaengine.c` | dma_request_chan → filter 回调匹配 |
| **2.17 of_dma 解析** | `of_dma.h` | of_dma_router_register、xlate 回调、dmas/dma-names 解析 |
| **2.18 STM32 HPDMA3 数据结构** | `stm32-dma3.c` | stm32_dma3_chan（通道上下文）、stm32_dma3_lli（链表节点）、stm32_dma3_ot（触发配置） |
| **2.19 DMAMUX 数据结构** | `stm32-dmamux.c` L31-52 | stm32_dmamux 映射表、dma_inuse 位图、CCR 寄存器备份 |
| **2.20 MDMA 架构概要** | `stm32-mdma.c` | 与 HPDMA 的定位差异（内存搬运 vs 外设 DMA），MCE 引擎特点 |
| **2.x dma-buf 架构（用户态 DMA 基础）** | | |
| **2.21 dma-buf 设计思想** | buffer 共享 vs buffer 拷贝 | 为什么需要 dma-buf：一次 DMA 数据多个消费者（显示+编码） |
| **2.22 struct dma_buf** | `include/linux/dma-buf.h` | fd 管理、attachments 链表、ops(dynamic) |
| **2.23 struct dma_buf_attachment** | `include/linux/dma-buf.h` | 导入者、sg_table、dma_buf_map_attachment |
| **2.24 dma-buf 生命周期** | alloc → export(fd) → mmap → attach → map → unmap → detach → close(fd) | 引用计数：fd 计数 + kernel count |
| **2.25 dma-heap 架构** | `drivers/dma-buf/dma-heap.c` | struct dma_heap(ops: alloc)、/dev/dma_heap/* 设备文件、ioctl 分发 |
| **2.26 system_heap vs cma_heap** | `heaps/system_heap.c` vs `heaps/cma_heap.c` | 分配策略、物理连续性、延迟差异 |

---

## 第 3 篇：03-SourceAnalysis.md — DMA 初始化与传输路径源码分析

**核心问题**：系统启动后 DMA 子系统怎么初始化的？一次完整的 DMA 传输从 API 调用到硬件寄存器操作是怎么走的？

### 调用链全景

```
初始化路径:
start_kernel
  └── (各种子系统初始化，DMA 在驱动 probe 阶段)
        └── 设备驱动 bind 阶段
              ├── stm32_dma3_probe()            ← platform_driver
              │     ├── 硬件参数探测 (HWCFGR 寄存器)
              │     ├── dma_async_device_register()   ← 注册到 dmaengine core
              │     └── of_dma_controller_register()  ← DTS dma 解析注册
              ├── stm32_dmamux_probe()
              │     ├── of_dma_router_register()      ← 注册 dma_router
              │     └── 等待 DMA 控制器 ready
              └── stm32_mdma_probe()        ← 可选，用于内存搬运
                    └── dma_async_device_register()

外设 DMA 传输路径 (以 SPI 读为例):
spi_device 驱动 (consumer)
  ├── dma_request_slave_channel(dev, "rx")   ← DTS dmas 属性解析
  │     └── of_dma_request_slave_channel()
  │           └── of_dma_match_channel()     ← dma-names 匹配
  │                 └── of_dma_xlate() 回调   ← DMAMUX 路由 + HPDMA 通道分配
  │                       └── stm32_dmamux_route()     ← 配置 CCR 寄存器
  │                             └── dma_request_channel()  ← HPDMA 分配
  ├── dmaengine_slave_config(&rx_config)     ← 配置 src/dst 地址、宽度、burst
  │     └── stm32_dma3_config()
  │           └── 更新 stm32_dma3_chan 中的配置参数（CTR1/CTR2 待写）
  ├── dmaengine_prep_slave_sg(sgl, len, DMA_DEV_TO_MEM)  ← 准备 LLI 链表
  │     └── stm32_dma3_prep_slave_sg()
  │           ├── stm32_dma3_chan_desc_alloc()  ← 从 DMA pool 分配 LLI
  │           ├── 构建每段传输的 Linked List Item (LLI)
  │           │     ├── CTR1: 数据宽度、burst size、FIFO 阈值
  │           │     ├── CTR2: 传输模式、交换模式、SINC/DINC
  │           │     ├── CLLR: 下一段 LLI 地址（链表指针）
  │           │     ├── CSAR: 源地址（外设 FIFO 地址）
  │           │     └── CDAR: 目标地址（内存 scatterlist 地址）
  │           └── virt-dma: 放入 desc_allocated 链表
  ├── dmaengine_submit(tx_desc)              ← 分配 cookie
  │     └── vchan_tx_submit()
  │           └── virt-dma: desc_allocated → desc_submitted
  └── dma_async_issue_pending(chan)          ← 启动传输
        └── stm32_dma3_issue_pending()
              ├── stm32_dma3_start()         ← 写硬件寄存器
              │     ├── CSAR = 外设 FIFO 地址
              │     ├── CDAR = 第一个 SG 段地址
              │     ├── CLLR = 第一条 LLI 地址（或 NULL 单段传输）
              │     ├── CTR1/CTR2 = 传输参数
              │     └── CCR.EN = 1           ← 启动 DMA
              └── virt-dma: desc_submitted → desc_issued

传输完成:
  └── DMA 硬件传输完成
        ├── CxSR.TCF = 1 (传输完成标志)
        ├── 中断触发 → stm32_dma3_irq_handler()
        │     ├── stm32_dma3_chan_irq()  ← 读取 CSR 判断事件类型
        │     │     ├── TCF: 传输完成
        │     │     ├── HTF: 半传输
        │     │     ├── DTEF: 数据传输错误
        │     │     └── SUSPF: 暂停完成
        │     ├── vchan_cookie_complete()  ← virt-dma: desc_issued → desc_completed
        │     └── tasklet_schedule(&vc->task)  ← 调度回调 tasklet
        └── vchan tasklet 执行
              └── stm32_dma3_complete_callback()
                    └── 调用 consumer 注册的回调函数
```

### 大纲

| 阶段 | 源码函数 | 文件 | 关键作用 |
|------|---------|------|---------|
| **3.1 DMA 控制器初始化** | | | |
| 3.1.1 | `stm32_dma3_probe()` | stm32-dma3.c | platform_driver probe，11 步顺序 |
| 3.1.2 | 硬件配置探测 → HWCFGR | stm32-dma3.c | 读取通道数、FIFO 大小、请求 ID 范围、AXI 主端口数 |
| 3.1.3 | `stm32_dma3_channel_probe()` | stm32-dma3.c | per-channel 初始化：virt_dma_chan 初始化、IRQ 申请 |
| 3.1.4 | LLI 内存池初始化 | stm32-dma3.c | 保留内存（dma_pool 或 reserved memory）用于 linked-list |
| 3.1.5 | `dma_async_device_register()` | dmaengine.c | 注册到 dmaengine core，初始化 dma_device 字段 |
| 3.1.6 | `of_dma_controller_register()` | of_dma.c | DTS 解析入口注册 |
| 3.1.7 | DMAMUX probe | stm32-dmamux.c | `of_dma_router_register()` + 通道等待 |
| **3.2 通道分配路径** | | | |
| 3.2.1 | `dma_request_slave_channel()` | dmaengine.c | 从 DTS dmas 属性解析请求 |
| 3.2.2 | `of_dma_request_slave_channel()` | of_dma.c | xlate 回调链：DMAMUX → HPDMA |
| 3.2.3 | `stm32_dmamux_route()` | stm32-dmamux.c | 写 CCR 寄存器，建立外设请求到 HPDMA 通道的映射 |
| **3.3 传输生命周期** | | | |
| 3.3.1 | `stm32_dma3_config()` | stm32-dma3.c | slave_config 参数校验与存储 |
| 3.3.2 | `stm32_dma3_prep_slave_sg()` | stm32-dma3.c | LLI 链表构建（核心） |
| 3.3.3 | `stm32_dma3_chan_desc_alloc()` | stm32-dma3.c | 描述符+LLI 分配 |
| 3.3.4 | `stm32_dma3_start()` | stm32-dma3.c | 寄存器写入+启动传输 |
| 3.3.5 | ISR 处理 → `stm32_dma3_irq_handler()` | stm32-dma3.c | 中断分发 |
| 3.3.6 | virt-dma 完成回调路径 | virt-dma.h | vchan_cookie_complete → tasklet → 用户回调 |
| **3.4 dmatest 路径（可选）** | | | |
| 3.4.1 | dmatest 线程循环 | dmatest.c | memcpy/slave 测试的完整路径 |

---

## 第 4 篇：04-Scenario.md — DMA 运行时情景分析

**核心问题**：SPI 从 MMC 卡读数据时 DMA 怎么加速的？音频播放时 cyclic DMA 如何工作？

### 场景一：SPI 读 MMC（slave_sg DMA）

```
用户态读 MMC 卡
  ├── VFS → block layer → MMC 驱动
  ├── MMC 驱动将请求转化为 SPI 传输（SPI 模式下）
  │     └── spi_stm32 驱动初始化 DMA 通道:
  │           ├── dma_request_slave_channel(dev, "rx")   ← DTS dmas 属性
  │           ├── dma_request_slave_channel(dev, "tx")
  │           └── dmaengine_slave_config(&rx_conf)       ← 配置 SPI RX FIFO 地址
  │
  ├── 传输开始: 
  │     ├── dmaengine_prep_slave_sg(rx_sgl, nents, DMA_DEV_TO_MEM)
  │     │     └── stm32_dma3_prep_slave_sg()
  │     │           ├── stm32_dma3_chan_desc_alloc()     ← 从 dma_pool 分配 LLI
  │     │           ├── 构建 LLI 链表:
  │     │           │     LLI[0].CSAR = SPI RX FIFO 地址
  │     │           │     LLI[0].CDAR = buf_A 物理地址
  │     │           │     LLI[0].CLLR = &LLI[1]
  │     │           │     LLI[1].CSAR = SPI RX FIFO (同地址)
  │     │           │     LLI[1].CDAR = buf_B 物理地址
  │     │           │     LLI[1].CLLR = NULL (最后一项)
  │     │           └── virt-dma: desc_allocated 链表
  │     ├── dmaengine_submit(tx_desc)
  │     └── dma_async_issue_pending(chan)
  │           └── stm32_dma3_start()
  │                 ├── 写 CDAR = LLI[0].CDAR (第一个 SG 段)
  │                 ├── 写 CLLR = LLI[0] 地址 (硬件自动遍历链表)
  │                 ├── 写 CTR1 = 数据宽度 + burst + FIFO 阈值
  │                 └── CCR.EN = 1   ← 硬件开始搬运
  │
  ├── 硬件自动执行（无 CPU 干预）:
  │     ├── HPDMA 从 SPI RX FIFO 读 → AXI 总线写 buf_A
  │     ├── 读完 LLI[0] → 硬件自动加载 CLLR 取 LLI[1]
  │     ├── HPDMA 从 SPI RX FIFO 读 → AXI 总线写 buf_B
  │     └── 读完 LLI[1] (CLLR=NULL) → 传输完成
  │
  ├── 中断路径:
  │     ├── CSR.TCF = 1
  │     ├── 中断 → stm32_dma3_irq_handler()
  │     │     ├── 清 TCF 标志
  │     │     └── vchan_cookie_complete()
  │     │           ├── desc_issued → desc_completed
  │     │           └── tasklet_schedule()
  │     ├── tasklet → stm32_dma3_complete_callback()
  │     └── spi_stm32 回调 → mmc 读取完成
  │
  └── Buffer 的 DMA 映射 (dma-mapping API 侧):
        ├── 如果 rx_sgl 是 streaming 映射 →
        │     dma_map_sg(dev, sgl, nents, DMA_FROM_DEVICE)
        │      → 硬件自动 cache 一致性维护（或 CPU 手动 dma_sync）
        └── 如果用 coherent 映射 →
              dma_alloc_coherent(dev, size, &dma_handle, GFP_KERNEL)
               → 分配 uncached buffer，CPU 和 DMA 看到一致数据
```

### 场景二：音频 playback（cyclic DMA）

```
用户态播放音频
  ├── ALSA 应用写 PCM 数据到 ALSA buffer
  │     └── ALSA core 分配 cyclic DMA buffer (dma_alloc_coherent)
  │
  ├── 音频驱动初始化 DMA:
  │     ├── snd_dmaengine_pcm_open()
  │     │     └── dma_request_chan(dev, "rx")  ← DTS dmas 属性
  │     └── dmaengine_slave_config(&i2s_config)
  │           ├── src_addr = I2S TX FIFO (内存→设备方向)
  │           └── src_addr_width = DMA_SLAVE_BUSWIDTH_4_BYTES
  │
  ├── trigger(PLAYBACK):
  │     └── dmaengine_prep_dma_cyclic(chan, buf_addr, buf_len, period_len, DMA_MEM_TO_DEV)
  │           └── stm32_dma3_prep_dma_cyclic()
  │                 ├── 分配 LLI: 将循环 buffer 拆成多个 period 描述符
  │                 │     LLI[0]: period 0 → I2S TX FIFO
  │                 │     LLI[1]: period 1 → I2S TX FIFO
  │                 │     ...
  │                 │     LLI[N-1]: period N-1 → I2S TX FIFO
  │                 │     LLI[N-1].CLLR = &LLI[0]  ← 形成循环链表！
  │                 └── virt-dma: desc_allocated
  │
  ├── 硬件循环搬运（无需 CPU 介入）:
  │     ├── HPDMA 读 memory → 写 I2S TX FIFO
  │     ├── 完成 period 0 → 硬件自动跳 LLI[1]
  │     ├── 完成 period 1 → 硬件自动跳 LLI[2]
  │     ├── ...
  │     ├── 完成 period N-1 → CLLR 指向 LLI[0] → 循环
  │     └── 一直循环到 trigger(STOP)
  │
  ├── period 中断 (LLI 完成中断):
  │     ├── 每完成一个 period → 对应 LLI 的 CIE 中断
  │     ├── 中断 → stm32_dma3_irq_handler()
  │     │     └── vchan_cyclic_callback()
  │     │           └── tasklet_schedule()
  │     └── tasklet → ALSA period_elapsed()
  │           └── ALSA 更新下一个 period 的数据 (乒乓 buffer)
  │
  └── dma-mapping:
        └── dma_alloc_coherent() 分配音频 buffer
              → ARM64 上通过页表属性修改为 non-cacheable
              → DMA 和 CPU 看到一致数据，不需要 dma_sync
```

### 场景三：UART TX/RX 双通道 DMA

```
UART 串口收发大量数据
  ├── 驱动初始化:
  │     ├── dma_request_slave_channel(dev, "rx")
  │     ├── dma_request_slave_channel(dev, "tx")
  │     └── dmaengine_slave_config(&rx_config, &tx_config)
  │           ├── rx: src_addr = USART_RDR 寄存器
  │           └── tx: dst_addr = USART_TDR 寄存器
  │
  ├── RX 路径（外设→内存）:
  │     ├── 循环用 dmaengine_prep_slave_sg(chan, rx_sgl, 1, DMA_DEV_TO_MEM)
  │     │     每次准备 1 个 SG 段 → 接收完成后再提交下一个
  │     ├── 传输完成 (TCF) → 驱动读取数据
  │     └── CPU 每次中断处理的是"一整包数据"而不是"每个字节"
  │
  ├── TX 路径（内存→外设）:
  │     ├── write() 系统调用 → UART 驱动 → dmaengine_prep_slave_sg(tx_chan, buf, DMA_MEM_TO_DEV)
  │     ├── dma_async_issue_pending(tx_chan)
  │     └── 传输完成 → 通知 TTY 层
  │
  └── 对比 PIO 模式:
        ├── PIO: 每个字节都触发中断，CPU 在 ISR 里搬一个 byte
        └── DMA: 一包数据一次中断，CPU 只处理开头和结尾
```

### 大纲

| 阶段 | 源码路径 | 文件 |
|------|---------|------|
| **4.1 场景设定** | DTS + 硬件连接 | stm32mp257d-atk.dts |
| **4.2 SPI + DMA（slave_sg）** | | |
| 4.2.1 SPI 驱动初始化 DMA | spi_stm32_probe → dma_request_slave_channel | spi-stm32.c |
| 4.2.2 Buffer 映射 (dma-mapping) | dma_map_sg() streaming 映射 | spi/stm32-dma3 |
| 4.2.3 LLI 链表构建 | stm32_dma3_prep_slave_sg() LLI 逐项填充 | stm32-dma3.c |
| 4.2.4 硬件自动执行 | CSR.TCF 等待，LLI 自动遍历 | stm32-dma3.c |
| 4.2.5 完成回调 | ISR → tasklet → SPI 回调 | stm32-dma3.c + spi-stm32.c |
| 4.2.6 PIO fallback | dma 分配失败/出错时的降级路径 | spi-stm32.c |
| **4.3 音频 cyclic DMA** | | |
| 4.3.1 Audio 驱动 DMA 通道分配 | ASoC → snd_dmaengine_pcm_open → dma_request_chan | ALSA core |
| 4.3.2 cyclic 配置与 LLI 循环链表 | dmaengine_prep_dma_cyclic 参数实际值 | stm32-dma3.c |
| 4.3.3 period 中断处理 | LLI 完成中断 → vchan_cyclic_callback → ALSA period_elapsed | stm32-dma3.c + ALSA |
| **4.4 UART 双通道 DMA** | | |
| 4.4.1 RX/TX 通道分离 | dma_request_slave_channel × 2 | stm32-usart.c |
| 4.4.2 RX 单段 SG 循环 | prep_slave_sg(1 SG) → complete → resubmit | stm32-usart.c |
| 4.4.3 TX 批量发送 | prep_slave_sg → issue_pending | stm32-usart.c |
| **4.5 PIO vs DMA 性能对比** | 实测数据 | |
| 4.5.1 CPU 占用率对比 | 相同数据传输量，perf 测量 | dmatest + ftrace |
| 4.5.2 延迟对比 | 小数据量下 DMA vs PIO 延迟 | 建立开销分析 |
| **4.6 DMAMUX 路由场景** | | |
| 4.6.1 竞争场景 | 两个外设同时请求同一个 HPDMA 通道 | stm32-dmamux.c |
| 4.6.2 分配与释放 | DMAMUX CCR 寄存器操作 | stm32-dmamux.c |
| **4.7 用户态 DMA 场景（新增）** | | |
| 4.7.1 dma-heap 分配场景 | 摄像头应用：open heap → alloc dmabuf → mmap → 传给 V4L2 驱动 | dma-heap.c + v4l2 |
| 4.7.2 dmabuf 跨设备共享 | V4L2 采集 → dmabuf fd → 传给 DRM 显示（zero-copy） | dma-buf.c + v4l2 + drm |
| 4.7.3 udmabuf 场景 | 用户态分配内存 → udmabuf → 传给 DMA 外设 | udmabuf.c |
| 4.7.4 用户态 DMA 完整链路 | 从用户分配→内核搬运→用户 mmap 消费 | 全链路代码 |
| **4.8 错误处理** | | |
| 4.8.1 传输错误中断 | CSR.DTEF → callback 错误状态 | stm32-dma3.c |
| 4.8.2 dmaengine 超时处理 | dma_wait_for_async_tx 超时路径 | dmaengine.c |
| 4.8.3 通道释放安全 | terminate_sync → 等待未完成传输 | dmaengine.c |

---

## 章节内容分配矩阵

```
                                 00-History  01-Usage  02-Arch   03-Source  04-Scenario
                                   演进史     API/调试   数据结构   初始化     运行时路径

【DMA 内存区域与分配】
ZONE_DMA/ZONE_DMA32 划分              ✅                     ✅
dma_set_mask → zone 选择                                  ✅
CMA 架构 (migration + 分配)                               ✅
dma_pool 小对象分配器                                      ✅
reserved-memory DMA 区域              ✅                   ✅

【Cache 管理】
ARM64 Cache 层次 (L1/L2, 64B line)                        ✅
非一致性总线模型                                           ✅
coherent 映射：页表 Non-cacheable                          ✅
streaming 映射：dc cvac/civac 指令                         ✅
dma_sync_single_* 完整调用链              ✅               ✅

【dma-mapping API 使用】
dma-mapping API 演化史               ✅
coherent 映射：alloc_coherent                  ✅
streaming 映射：map_single/map_sg                ✅
cache 同步 API                                     ✅
DMA 掩码 set_mask                               ✅
dma_address_space 模型 (direct/iommu)                       ✅
dma_map_ops 虚函数表                                          ✅

【dmaengine 框架】
dmaengine 框架演进史             ✅
virt-dma 诞生                    ✅
dma_router 演进                  ✅

dma_request_chan                              ✅
dmaengine_slave_config                        ✅
prep_slave_sg/cyclic/memcpy                   ✅
dmaengine_submit/issue_pending                ✅
dmatest 用法                                  ✅
debugfs DMA 状态                              ✅

DTS hpdma 3 cell 格式                         ✅
外设 dmas 属性写法                            ✅
LLI reserved memory                           ✅

三层架构                                               ✅
dma_device/chan/descriptor                              ✅
cookie 同步模型                                         ✅
dma_cap_mask_t                                          ✅
virt-dma 5 链表 + tasklet                               ✅
dma_slave_config                                        ✅
DMA Router 框架                                         ✅
of_dma 解析机制                                         ✅
stm32_dma3_chan/lli/ot                                  ✅
DMAMUX 数据结构                                         ✅

stm32_dma3_probe                                                 ✅
HWCFGR 硬件探测                                                   ✅
dma_async_device_register                                        ✅
of_dma_controller_register                                       ✅
通道分配：DMAMUX route → HPDMA alloc                               ✅
LLI 链表构建                                                       ✅
描述符生命周期 prep→submit→issue                                     ✅
寄存器写入启动                                                     ✅
ISR → vchan_cookie_complete                                        ✅
tasklet 回调调度                                                   ✅

【外设 DMA 实战】
SPI + slave_sg 完整路径                                                            ✅
音频 cyclic DMA 路径                                                                ✅
UART 双通道 DMA                                                                     ✅
PIO vs DMA 性能实测                                                                ✅
DMAMUX 竞争与路由                                                                   ✅
PIO fallback 降级                                                                   ✅
错误处理：DTEF/超时/安全回收                                                            ✅
dmatest 测试实例                                                                     ✅

【用户态 DMA（dma-buf/dma-heap）】
dma-buf 框架演化史          ✅
dma-heap 诞生              ✅

dma-heap 分配 ioctl                                ✅
system_heap vs cma_heap                            ✅
udmabuf 创建                                       ✅
dmabuf fd mmap                                     ✅
dmabuf 调试 /sys/kernel/debug                      ✅

struct dma_buf 架构                                           ✅
struct dma_buf_attachment                                      ✅
dma-buf 生命周期 (fd→mmap→attach→map→close)                    ✅
dma-heap 数据结构 (ops: alloc)                                 ✅
system_heap/cma_heap 分配策略                                  ✅

V4L2+dmabuf 跨设备共享                                                       ✅
udmabuf 数据流                                                                  ✅
用户态 DMA 完整链路（分配→搬运→消费）                                             ✅
```

---

## 写作原则

1. **源码唯一**：每个结论必须对应 `.c`/`.h` 文件的函数或行号，参考资料仅作理解参考不直接引用
2. **不堆代码**：代码块不超过全文 30%，关键路径用流程图/表格辅助
3. **场景驱动**：04 篇用 SPI flash 读 + 音频播放两个实际场景串联，其他篇的示例贴近 STM32MP257 实际硬件
4. **分层写作**：先写 01/02 并行，然后 03 需要完整的源码阅读，最后 04 综合串联

---

*规划完成日期：2026-07-15*
*预计开始写作：01-Usage.md 第一版*
