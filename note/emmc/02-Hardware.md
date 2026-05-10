# 02. eMMC 硬件详解：协议、传输与控制器

> 本文是 [STM32MP257 eMMC 驱动深度分析](README.md) 系列的第 2 篇。  
> 对应框架步骤 ②：搞懂硬件本身怎么工作的，再看内核怎么封装。
>
> **前置:** [01-Usage.md](01-Usage.md) — 对 sysfs、debugfs 有基本操作经验  
> **下一篇:** 待定
>
> **字数：** 约 14,500 字  
> **建议阅读时间：** 50–70 分钟（含图表和代码）

---

## 2.1 MMC 总线协议

### 从物理层说起

MMC 总线由三组信号线组成：

```
Host (SDMMC2)                     eMMC
┌────────────────┐               ┌──────────┐
│                │──── CLK ─────→│          │
│   SDMMC        │←─── CMD ─────→│  eMMC    │
│   控制器       │←─── DAT[7:0] ─→│  芯片    │
│                │──── RST_n ────→│          │
│   (PL180)      │               │          │
└────────────────┘               └──────────┘
```

| 信号 | 方向 | 作用 |
|------|------|------|
| CLK | Host → Device | 时钟，所有信号在时钟沿采样 |
| CMD | 双向 | 命令 + 响应，同一条线分时复用 |
| DAT[7:0] | 双向 | 数据传输，可配 1/4/8 位 |
| RST_n | Host → Device | 硬件复位，低电平有效 |

**上电时序要求**：电源稳定后，eMMC 需要至少 74 个时钟周期才能开始接受第一条命令。这就是内核初始化代码中 `mmc_power_up()` 里 msleep 和发 CMD0 之间那些 delay 的原因。

### 命令在线上长什么样

一条命令在 CMD 线上传输时，是一个 **48 位的串行帧**，从起始位到结束位依次发送：

#### 帧结构

| 位域 | 47 | 46~40 | 39~8 | 7~2 | 1 | 0 |
|------|----|-------|------|-----|---|---|
| 字段 | 结束位 | CRC7 | 参数（32 bits） | 命令索引（6 bits） | 传输位 | 起始位 |
| 长度 | 1 bit | 7 bits | 32 bits | 6 bits | 1 bit | 1 bit |
| 发送顺序 | ← 最后 | | | | | 最先 → |

#### 各字段含义

| 字段 | 值 | 说明 |
|------|----|------|
| 起始位 | 固定为 0 | 总线空闲时 CMD 线是高电平，拉低表示帧开始 |
| 传输位 | 1 = 命令（Host→Device）<br>0 = 响应（Device→Host） | CMD 线是双向的，靠这个位区分方向 |
| 命令索引 | 0~63，6 bits 编码 | CMD0 ~ CMD63，每个编号对应固定功能 |
| 参数 | 32 bits，含义由命令定义 | 可能是地址、RCA、块计数等 |
| CRC7 | 覆盖 bit 1~39 | 多项式 $x^7 + x^3 + 1$，控制器硬件自动计算填充 |
| 结束位 | 固定为 1 | 帧结束标志 |

#### 实例分解：CMD9（SEND_CSD）

以 CMD9 查询 CSD 寄存器为例，假设 RCA = 1：

**第一步：Host 在 CMD 线上发命令帧**

```
                  ┌──── 48-bit 命令帧 ──────────────────────────────┐
CMD 线:           │ 起始 传输  命令索引 CMD9     参数 (RCA=1)      CRC7      结束 │
   (高电平空闲)  ─0────1────0 0 1 0 0 1────────0x00010000────[自动计算]───1───→
```

逐字段分解：

| 字段 | 值（二进制） | 说明 |
|------|-------------|------|
| 起始位 | 0 | 帧开始 |
| 传输位 | 1 | Host → Device（命令） |
| 命令索引 | 001001 | CMD9 = SEND_CSD |
| 参数 | 0000_0000_0000_0001_0000_0000_0000_0000 | RCA=1，向左偏移 16 位存入参数 |
| CRC7 | 控制器自动填充 | 多项式 $x^7 + x^3 + 1$，初值 0x00 |
| 结束位 | 1 | 帧结束 |

按字节展开（发送顺序从左到右）：

| 字节 | 值 | 位内容 |
|------|----|--------|
| Byte 0 | **0x49** | `[start=0][trans=1][cmd5=0][cmd4=0][cmd3=1][cmd2=0][cmd1=0][cmd0=1]` |
| Byte 1 | 0x00 | 参数 [31:24] |
| Byte 2 | 0x01 | 参数 [23:16] |
| Byte 3 | 0x00 | 参数 [15:8] |
| Byte 4 | 0x00 | 参数 [7:0] |
| Byte 5 | [CRC7+end] | `[crc6..crc0][end=1]` |

> **`0x49` 的构成：** CMD 线发送顺序是起始位最先 → 传输位 → 命令索引（MSB first）。所以第一字节按发送顺序排列：`start=0` → `trans=1` → `cmd5=0` → `cmd4=0` → `cmd3=1` → `cmd2=0` → `cmd1=0` → `cmd0=1`，8 位一组读出就是 `0b01001001` = **0x49**。其中 bit 6=1 对应传输位，bit 5~0=001001 就是 CMD9。

**第二步：eMMC 在 CMD 线上返回 136 位 R2 响应**

```
                  ┌──── 136-bit 响应帧 ───────────────────────────────────┐
CMD 线:      ──0────0────0 0 1 0 0 1────128-bit CSD 寄存器────CRC16(16)────1──
             起始  传输(=0)   命令索引     CSD 内容              16 位 CRC   结束
```

内核通过四次 `readl(SDMMC_RESP0~3)` 读取这 128 位 CSD，每次 32 位。

### 响应的三种常见格式

MMC 协议定义了 5 种响应类型（R1~R5），最常用的是以下三种：

**R1 响应（常规状态响应）：** 48 位

```
          ┌──── 48-bit 响应帧 ──────────────────────────────────┐
CMD 线: ─0────0────命令索引(6)────32 位状态字────CRC7────1──
         起始  传输(=0)           ↑                         结束
                                  CURRENT_STATE + 错误标志
```

R1 不带数据，但携带一个 **32 位的状态字**，包含卡的当前状态和错误标志。调试时最常看到的就是 R1——每次发 CMD13（SEND_STATUS）后返回。

状态字中的关键字段：

| 位域 | 宽度 | 含义 |
|------|------|------|
| [31:26]、[23:20]、[15:8] | 22 bits | 各种错误标志（CRC 错误、非法命令、擦除错误等）|
| **12:9** | **4 bits** | **CURRENT_STATE** — 卡当前状态，调试最关键的字段 |
| 8 | 1 bit | READY_FOR_DATA — 卡是否准备好收发数据 |

**CURRENT_STATE 编码：**

| 值 | 状态 | 含义 |
|----|------|------|
| 0 | Idle | 空闲，卡尚未初始化 |
| 1 | Ready | 已就绪 |
| 2 | Ident | 正在识别 |
| 3 | Standby | 待机 |
| 4 | **Tran** | **数据传输状态（卡正常工作时大部分时间在此）** |
| 5 | Data | 数据正在传输中 |
| 6 | Rcv | 正在接收数据 |
| 7 | Prg | 正在编程（写入）|
| 8 | Dis | 断开 |
| 9 | Btst | 位测试 |
| 10~15 | — | 保留 |

> 调试时如果怀疑 eMMC 没反应，第一步就是读 R1 看 CURRENT_STATE。`Tran(4)` 说明卡正常，`Prg(7)` 说明卡正在忙写入，`Idle(0)` 说明卡可能被复位了。

**R2 响应（CID/CSD 寄存器）：** 136 位

```
          ┌──── 136-bit 响应帧 ──────────────────────────────────┐
CMD 线: ─0────0────命令索引(6)────128-bit 寄存器值────CRC16(16)────1──
         起始  传输(=0)
```

用于 CMD2（ALL_SEND_CID）和 CMD9（SEND_CSD）。eMMC 返回完整的 128 位寄存器内容 + 16 位 CRC。内核通过 `readl(base + SDMMC_RESP0)` ~ `readl(base + SDMMC_RESP3)` 四次读取这 128 位（每次 32 位）。

**R3 响应（OCR 寄存器）：** 48 位

```
          ┌──── 48-bit 响应帧 ─────────────────────────────┐
CMD 线: ─0────0────命令索引(6)────32-bit OCR 值────CRC7────1──
```

用于 CMD1（SEND_OP_COND）。返回卡的 OCR 寄存器内容，告诉 Host 卡支持的电压范围。这是初始化流程中第一步协商的响应。

| 响应类型 | 长度 | 用在什么命令 | 关键数据 |
|---------|------|-------------|---------|
| R1 | 48 bits | CMD13、CMD6、CMD7 | 32 位状态字（含 CURRENT_STATE） |
| R2 | 136 bits | CMD2、CMD9、CMD10 | 128 位 CID/CSD 寄存器 |
| R3 | 48 bits | CMD1 | 32 位 OCR 电压信息 |

### 数据是怎么传的

#### 总线宽度与 DAT 线使用

eMMC 支持三种总线宽度模式，不同模式下 DAT 线的使用方式不同：

| 总线宽度 | 使用的 DAT 线 | 每时钟传输的位数 | ATK 板是否使用 |
|---------|-------------|---------------|--------------|
| 1-bit | DAT0 仅此一根 | 1 bit | ×（初始化阶段短暂使用）|
| 4-bit | DAT[3:0] | 4 bits | ×（SD 卡协议标准，eMMC 亦可选）|
| **8-bit** | **DAT[7:0]** | **8 bits** | **✓ ATK 板实际模式** |

**关键区别：每条 DAT 线独立传输字段**

不管配置成 1-bit 还是 8-bit，每条 DAT 线上的数据块格式是完全一样的——**每条线都有自己的起始位、CRC16 和结束位**。可以理解为 8 条独立的串行链路在同时跑，只是时钟是共享的。

```
8-bit 模式下，512 字节块在 DAT[7:0] 上的分配：

DAT0: [起始] [byte 0, byte 8, byte 16...]      [CRC16] [结束]
DAT1: [起始] [byte 1, byte 9, byte 17...]      [CRC16] [结束]
DAT2: [起始] [byte 2, byte 10, byte 18...]     [CRC16] [结束]
  ...
DAT7: [起始] [byte 7, byte 15, byte 23...]     [CRC16] [结束]
                             512 字节 ÷ 8 = 每线 64 字节
```

每条 DAT 线上的独立帧格式：

```
┌──── 每条 DAT 线上的帧格式 ────────────────────────────┐
[起始位(0)] [数据(N bits)] [CRC16(16 bits)] [结束位(1)]
  └ 1 bit    └ N = 块大小 × 8 / 总线宽度    └ 每线独立 CRC
```

以 512 字节块为例：

| 总线宽度 | 每条 DAT 线的数据量 | 数据位宽 N |
|---------|-------------------|-----------|
| 1-bit | 512 × 8 = 4096 bits | 4096 |
| 8-bit | 512 × 8 ÷ 8 = 512 bits | 512 |

**8-bit 比 1-bit 快 8 倍的根源就在于此：同样 512 字节的数据，8-bit 模式下每条线只需要传 512 bits，而 1-bit 模式需要传 4096 bits。** 多出来的 7 条线把数据从串行变成了"8 路并行"。

#### 不带数据的命令（CMD-only）

以 CMD13（SEND_STATUS）为例——只有 CMD 线上的交互，DAT 线不传输数据块：

```
CMD:  ───[CMD13]──────[R1 响应]────────────
DAT0: ─────────────────────────────────────  (高电平 = 就绪，仅传 busy 信号)
DAT[7:1]: ────────────── (不受影响) ─────────
```

#### 读操作 (Data Read)

**单块读（CMD17）：** 读一个 512 字节块

```
CMD:  ───[CMD17]──────[R1]─────────────────
DAT:  ────────────────[数据块]──────────────

8-bit 模式下 DAT[7:0] 的细节展开：
DAT0: ──[起始][byte0, byte8, byte16...][CRC16][结束]
DAT1: ──[起始][byte1, byte9, byte17...][CRC16][结束]
  ...
DAT7: ──[起始][byte7, byte15, byte23...][CRC16][结束]
```

**多块读（CMD18）：** 连续传输多个块，最后发 CMD12 停止

```
CMD:  ───[CMD18]──────[R1]───────────────────[CMD12(停止)]──[R1]──
DAT:  ────────────────[块1]──[块2]──...[块n]────────────────────────
```

#### 写操作 (Data Write)

**多块写（CMD25）：** 多块写比读多一个流程——每写一块后 eMMC 会在 DAT0 上拉低表示 busy

```
CMD:  ───[CMD25]──────[R1]──────────────────────────
DAT:  ──────[块1]──(DAT0 busy)──[块2]──(DAT0 busy)──

DAT0 busy 期间的细节：
DAT0: ──[块1]──────[DAT0被拉低(busy)]────────[块2]────
DAT1: ──[块1数据]─────────(空闲)─────────────[块2数据]──
  ...   （busy 期间只有 DAT0 被拉低，其他 DAT 线可忽略）
```

> **DAT0 busy 的调试意义：** 多块写后，eMMC 需要时间把数据从 FIFO 写入内部 Flash 阵列，此期间 DAT0 拉低。如果内核没有 `MMC_CAP_WAIT_WHILE_BUSY` 能力，就不会等 busy 结束就发下一条命令，导致数据丢失。ATK 板的 `caps=0x40401347` 中 bit 9 置位，说明驱动已配置此能力。

> **每个块每条 DAT 线独立 CRC16 的意义：** 
> 如果 8-bit 模式下出现 `data_crc_err`，错误可能只发生在某一条 DAT 线上。通过调试寄存器（如果硬件支持）可以定位到具体是哪条线出了问题，这比 1-bit 模式"要么对要么错"的二分诊断精确得多。

### 总线模式切换

初始化阶段 CMD 线工作在 **开漏（Open Drain）模式**——多个设备可以同时拉低 CMD 线来响应。一旦选中卡后，切换到 **推挽（Push-Pull）模式**——速度更快。

```
初始化阶段: CMD = Open Drain (多个设备竞争总线)
选中后:     CMD = Push-Pull (单设备通信)
```

这个切换由 `SDMMC_POWER` 寄存器控制。内核代码中能看到 `mmc->ios.bus_mode = MMC_BUSMODE_OPENDRAIN` 和 `MMC_BUSMODE_PUSHPULL` 的切换。

## 2.2 eMMC 命令集

MMC 协议定义了 CMD0~CMD63 共 64 条基本命令。eMMC 实现了其中的一个子集，加上少数应用命令（ACMD）。本节按**功能分组**，覆盖驱动初始化和数据传输中实际遇见的命令，侧重三条：**为什么需要这条命令**、**参数怎么用**、**与 SD 卡的区别**。

### 2.2.1 卡识别与复位命令

#### CMD0 — GO_IDLE_STATE（强制复位）

**作用：** 强制卡进入 Idle 状态，无论卡当前处于什么状态。这是总线上对卡最直接的复位方式。

**参数：** 固定为 0x00000000。不能带额外信息，因为 Idle 的卡还没分配地址。

**响应：** 无响应。卡收到 CMD0 后无条件复位，不需要回复。

**为什么这么设计：**

卡可能处于不可预知的状态——上电未初始化、上次掉电时卡在 Prg（编程）状态、或者被 CMD52/CMD53（SDIO）进入了奇怪模式。需要一个"核按钮"让所有卡回到统一的已知起点。不要求响应也是刻意的：某些状态下卡可能根本发不出响应，这时候 CMD0 是唯一能"叫醒"它的方式。

**与 SD 卡的区别：**

SD 卡的 CMD0 多了一个功能——参数为 0x00000000 时进入 SD 模式，参数为 0xFFFFFFF0 时提示"我要切到 SPI 模式"。eMMC 没有 SPI 模式，不需要这个功能。

#### CMD1 — SEND_OP_COND（电压协商 + 初始化查询）

CMD1 是 eMMC 初始化阶段最核心的命令，负责两件事：**电压协商**（Host 和卡确认彼此支持的电压范围）和 **初始化状态查询**（卡的内置上电流程是否跑完）。

##### OCR 寄存器详解

CMD1 的参数和响应都叫 OCR（Operating Conditions Register），本质上 Host 和卡通过同一个 32 位寄存器相互通信：

```
参数/响应 = OCR 寄存器（32 位）
  [31]      [30]      [29:24]    [23:20]    [19:16]    [15:8]     [7:0]
  BUSY      HCS      保留       保留       保留       保留       电压窗口
  ↑         ↑                                                 ↑
初始化完成?  高容量?                                           支持的电压范围
```

**电压窗口（[7:0]）**——每个 bit 对应一个电压档位：

| Bit | 电压范围 | eMMC 是否常用 |
|-----|---------|--------------|
| 0 | 2.7~2.8V | 少见 |
| 1 | 2.8~2.9V | 少见 |
| 2 | 2.9~3.0V | 少见 |
| 3 | 3.0~3.1V | 少见 |
| 4 | 3.1~3.2V | 少见 |
| 5 | 3.2~3.3V | ✓ 常见（3.3V 标准）|
| 6 | 3.3~3.4V | ✓ 常见 |
| 7 | 3.4~3.6V | 少见 |

ATK 板供电 3.3V，卡返回的 OCR 中 bit 5~6 置位，表示支持 3.2~3.4V 范围。卡可以同时置位多个 bit 以说明支持宽电压。

**Bit 30 — HCS（High Capacity Support）：**
- 0：卡使用**字节寻址**，最大 2GB
- 1：卡使用**扇区寻址**（LBA），每扇区固定 512 字节，支持 >2GB 容量

ATK 板的 Kingston 58A43A 是 14.6 GiB 的 eMMC，HCS 必须置 1 才能完整寻址 14.6 GiB。

**Bit 31 — BUSY（初始化完成标志）：**
- 0：卡内部初始化尚未完成，Host 继续发 CMD1 轮询
- 1：卡已就绪，Host 可以发下一条命令

内核中用 `include/linux/mmc/mmc.h:206` 定义为 `#define MMC_CARD_BUSY 0x80000000`。

##### 两阶段协商流程

CMD1 不是一发一收就结束的，而是**两阶段轮询协议**：

```
阶段一：探测（probe）
  Host → CMD1(参数=0x00000000)           ← 参数=0，问"卡你支持什么电压"
  Host ← R3(OCR=xxxxxxxx)                 ← 卡返回 OCR，此时 busy=0 仍在初始化

阶段二：确认 + 轮询 busy
  Host → CMD1(参数=card_ocr | BIT(30))   ← 用卡报告的电压窗口 + HCS 声明
  Host ← R3(OCR=xxxxxxxx，busy=0)         ← "还没好，继续等"（每 4ms 轮询一次）
  Host → CMD1(...)  ← 重复几百毫秒到 1 秒
  Host ← R3(OCR=xxxxxxxx，busy=1)         ← "准备好了，继续下一步"
```

**为什么分成两阶段？**

Host 不知道卡支持什么电压时，没法构造参数。第一阶段（参数=0）让卡主动暴露 OCR，Host 获得电压信息后再用匹配的电压正式协商。第二个阶段如果电压不匹配（如 Host 只支持 3.3V 但卡只支持 1.8V），卡会反复发 busy=0 直到超时，内核 `mmc_select_voltage()` 检测到后返回 -EINVAL。

##### 内核实现分析

代码位于 `drivers/mmc/core/mmc_ops.c:223`：

```c
int mmc_send_op_cond(struct mmc_host *host, u32 ocr, u32 *rocr)
{
    cmd.opcode = MMC_SEND_OP_COND;     // = CMD1
    cmd.arg = ocr;                      // 阶段一=0，阶段二=card_ocr | HCS
    cmd.flags = MMC_RSP_R3;             // 响应类型 R3

    return __mmc_poll_for_busy(host, 4000, 1000, callback, &cb_data);
    //                              ↑4ms   ↑1秒
}
```

关键参数：
- **轮询间隔：** 4000 μs = 4ms。太快增加总线压力，太慢增加初始化时间
- **超时时间：** 1000 ms = 1 秒。大多数 eMMC 在 100~300ms 内完成初始化
- **回调逻辑：** 检测响应 bit 31。如果 busy=0 且参数 ocr=0（阶段一），自动将下次参数设为 `resp[0] | BIT(30)`

注意 `mmc_send_op_cond()` 在内核中有两处不同的调用，**用途不同**：

**第一处（在 `mmc_attach_mmc()` 中）：**

```c
// 文件 drivers/mmc/core/mmc.c:2304
err = mmc_send_op_cond(host, 0, &ocr);
```

这是 MMC 核心层的**卡类型探测**。核心层不知道总线上接的是 eMMC、SD 还是 SDIO 卡，所以逐个试：先发 CMD1(0)。如果回来有效 OCR，说明是 eMMC 卡，走 `mmc_init_card()`。如果超时无响应，换 CMD5（SDIO）或 ACMD41（SD）。

**第二处（在 `mmc_init_card()` 中）：**

```c
// 文件 drivers/mmc/core/mmc.c:1624
mmc_go_idle(host);                                    // CMD0 复位 → 卡回到 Idle
err = mmc_send_op_cond(host, ocr | BIT(30), &rocr);  // 再协商 + HCS
mmc_send_cid(host, cid);                              // CMD2 读 CID
```

`mmc_init_card()` 执行前先发了 CMD0 复位，**CMD0 复位后卡的所有之前协商好的参数都丢了**，所以必须重新协商 OCR。这次参数中带上阶段一拿到的电压值 + HCS=1。

两次调用的完整流程：

```c
/* 第一次 CMD1：探测卡类型 */
mmc_attach_mmc():
    mmc_send_op_cond(host, 0, &ocr);         // CMD1(0) → 如果有响应，说明是 eMMC
    rocr = mmc_select_voltage(host, ocr);     // 检查电压交集
    mmc_init_card(host, rocr, NULL);          // 进入 eMMC 初始化路径
      /* 第二次 CMD1：CMD0 复位后重新协商 */
      mmc_go_idle(host);                     // CMD0 → 卡回到 Idle
      mmc_send_op_cond(host, ocr|BIT(30), &rocr);  // CMD1 → 重新协商 + HCS
      mmc_send_cid(host, cid);               // CMD2 → 读 CID
```

> **为什么 CMD0 复位后要重新协商？** 因为 CMD0 让卡回到上电初始状态，之前协商的电压窗口和 HCS 位都被清掉了。如果不重新协商直接发 CMD2，卡在逻辑上还处于"我没被配置过电压"的状态。

##### 卡在忙什么（为什么需要几百毫秒）

上电到 busy=1 之间，eMMC 内部在做：

```
上电 → 内部 LDO 稳压 → Flash 阵列升压 → FTL 表加载到 SRAM → BIST(自检) → busy=1
```

FTL（Flash Translation Layer）映射表加载是大头——地址映射表存放在 Flash 保留区域，上电时必须搬到 SRAM 中。容量越大，FTL 表越大，初始化越慢。

##### 与 SD 卡的关键区别

| 特性 | eMMC | SD 卡 |
|------|------|-------|
| **命令号** | **CMD1** | **ACMD41**（CMD55 + CMD41）|
| 总线宽度协商 | CMD6 + Ext_CSD | ACMD6 |
| CCC 信息 | CMD1 不提供 | ACMD41 响应带 CCC（Card Command Class）|
| 1.8V 切换 | CMD6 + POWER_CLASS | CMD11 + S18R 位 |

SD 卡从一开始就和 MMC 分叉发展，初始化阶段用 ACMD41 代替 CMD1 是明确的分界线。两者初始化走完全不同的代码路径：

```
eMMC: mmc_attach_mmc() → mmc_init_card()     → CMD1 → CMD6
 SD:  mmc_attach_sd()  → mmc_sd_init_card()  → ACMD41 → ACMD6
```

内核协议栈根据命令响应自动判断卡类型——CMD1 返回有效 OCR 就走 eMMC 路径，CMD1 无响应就试 ACMD41 走 SD 路径。

#### CMD2 — ALL_SEND_CID（仲裁 + 读 CID）

**作用：** 在总线上所有处于 **Ready** 状态的卡中，通过仲裁选出一张，读取其 CID，让这张卡进入后续的初始化步骤（分配 RCA→选中→配置）。**CMD2 的核心不是"读数据"，而是"选人"——每执行一次 CMD2，就有一张卡被选中进入下一步。**

参数：无。CMD2 是广播命令，所有处于 Ready 状态的卡都会参与仲裁。

响应（R2）：128 位 CID + 16 位 CRC。

##### CID 寄存器格式

CID（Card Identification Register）是 eMMC 的"身份证"——128 位固化在芯片中的唯一标识：

```
CID（128 位）
[127:120]  [119:104]  [103:56]      [55:48]   [47:16]   [15:8]    [7:1]  [0]
  MID        OID        PNM          PRV       PSN       MDT       CRC7   保留
  8 bits    16 bits    48 bits      8 bits    32 bits   12 bits   7 bits  1 bit
```

| 字段 | 全称 | 长度 | 内容 |
|------|------|------|------|
| MID | Manufacturer ID | 8 bits | JEDEC 分配的厂家编码，如 Kingston=0x70, Samsung=0x15 |
| OID | OEM/Application ID | 16 bits | 厂家自定义的 OEM 标识（ASCII 字符）|
| PNM | Product Name | 48 bits | 6 个 ASCII 字符的型号名（如 "58A43A"）|
| PRV | Product Revision | 8 bits | 高 4 位主版本，低 4 位次版本（如 0x10 = rev 1.0）|
| PSN | Product Serial Number | 32 bits | 厂家分配的序列号，唯一标识一颗芯片 |
| MDT | Manufacturing Date | 12 bits | 生产日期（年 = 2000 + [11:8]，月 = [7:4]，日不用）|
| CRC7 | CID Checksum | 7 bits | CID 的 CRC7 校验值 |

**为什么需要 CID？** 三条理由：
- **唯一识别每张卡**：MID + PNM + PSN 三元组可以唯一确定世界上的每一张 eMMC。在多卡系统或有热插拔的场景中至关重要。
- **仲裁基础**：CMD2 利用 CID 的独特性做逐位仲裁（见多卡系统说明），不需要额外的硬件引脚。
- **系统确权**：某些系统启动时读 CID 锁卡，如果不是指定的制造商/序列号就拒绝启动。

##### 在 ATK 板上读 CID

```bash
# 原始 CID（16 进制字符串）
$ cat /sys/block/mmcblk1/device/cid
70114a4138343441af886901000a1500

# 解析后的各个字段
$ cat /sys/block/mmcblk1/device/manfid     # MID → 0x70 = Kingston
0x000070
$ cat /sys/block/mmcblk1/device/name        # PNM
58A43A
$ cat /sys/block/mmcblk1/device/serial      # PSN
0xaf886901
$ cat /sys/block/mmcblk1/device/date        # MDT → 04/2025
04/2025
$ cat /sys/block/mmcblk1/device/fwrev       # PRV → 0x10 = rev 1.0
0x10
```

> sysfs 的路径是 `/sys/block/mmcblk1/device/*`，这些信息在 01-Usage.md 中已详细介绍过。这里是从协议层面解释数据的来源——CID 不是内核编出来的，是 CMD2 从卡里读出来的原始寄存器值。

##### 单卡系统（ATK 板）的初始化序列

```
CMD0  → Idle                    复位卡
CMD1  → Ready                   电压协商，等 busy=1
CMD2  → Ident                   读 CID ◀──
CMD3  → Standby                 分配 RCA
CMD7  → Tran                    选中卡，准备传数据
CMD6  → 配置完成               切总线宽度/时序
```

CMD1 返回 busy=1 之后 CMD2 紧随发出，中间没有任何等待。对于 ATK 板这颗焊接的 eMMC，CMD2 只执行一次。

##### 多卡系统：每次 CMD2 循环 = 初始化一张卡

先理解 CMD1 在多卡下的行为——CMD1 也是广播，所有卡一起做电压协商，**等所有卡都 busy=1 后，同时进入 Ready**。但从 Ready 开始，每张卡各自走完 CMD2→CMD3 的循环：

```
CMD0                  → 所有卡复位到 Idle
CMD1(ocr|HCS) × N     → 电压协商，等所有卡 busy=1 → 同时进入 Ready
                         ╔═════════════════════════════════════╗
循环 ① ══→  CMD2   → 卡 A 仲裁胜出 → 返回 CID → 卡 A→Ident  ║
         CMD3(1) → 卡 A 领 RCA=1  → 卡 A→Standby             ║
                         ╠═════════════════════════════════════╣
循环 ② ══→  CMD2   → 只剩卡 B 响应 → 返回 CID → 卡 B→Ident  ║
         CMD3(2) → 卡 B 领 RCA=2  → 卡 B→Standby             ║
                         ╚═════════════════════════════════════╝
每个循环 = 一张卡的完整识别过程。
CMD1 之后重复 CMD2→CMD3 的组合，每轮识别一张卡。
```

> **多卡同时响应 CMD1 不会冲突吗？** 不会。CMD1（以及后面的 CMD2、CMD3）都工作在 Open Drain 模式——卡只拉低 CMD 线（输出 0），不主动拉高（输出 1 靠释放让上拉电阻完成）。所以多卡同时响应时，每 bit 的结果是所有卡的**线与**：只要有一张卡在该位输出 0（拉低），这一位就是 0。这让 CMD1 能天然完成两件事：
> 
> ① **电压窗口取交集**：卡 A 支持 3.2~3.3V（bit 5=1, bit 6=0），卡 B 支持 3.3~3.4V（bit 5=0, bit 6=1）。线与后 bit 5=0, bit 6=0 —— 交集为空，说明没有共同电压，协商失败。只有所有卡同时支持某个电压时（bit 5 全是 1），结果才是 1。
> 
> ② **busy 位是所有卡的就绪信号的"线与"**：只要还有一张卡在初始化（busy=0），它就把这一位拉低，Host 看到的就是 0。只有所有卡的 busy 都变成 1（都不再拉低），Host 才能读到 1，才会进行下一步。
> 
> CMD2 则不同——CID 是每张卡唯一的，线与会把 CID 混成乱码，所以不能走"公共信号取交集"的思路，必须改用仲裁来选出一张独占 CMD 线。两者硬件都是 Open Drain，但信号的解读方式不同。

**为什么是循环而不是一次性识别所有卡？** 因为 CMD 线在同一时刻只能传输一张卡的 CID。仲裁机制虽然让所有卡都"尝试"输出，但每轮只有一张卡能成功传完。要识别下一张，就得再发一次 CMD2。**每轮循环之间，卡不需要重新初始化**——没胜出的卡一直在 Ready 状态等着，发 CMD2 它就响应。

> Linux 内核中一个 mmc_host 只管理一张卡，所以 `mmc_init_card()` 里的 CMD2 只会被执行一次。多卡循环是 MMC 协议本身的能力，Linux 驱动层没有实现这个场景。

#### CMD3 — SET_RELATIVE_ADDR（分配 RCA）

**作用：** 给卡分配一个 16 位的相对地址（RCA），后续所有命令都用这个地址来指代这张卡，不再用 CID。

**参数：**

```
参数 [31:16]    [15:0]
    RCA       保留(0)
```

**响应：** R1。

**为什么设计 RCA 而不是一直用 CID：**

CID 有 128 位，每次命令都带 CID 会浪费大量的 CMD 线带宽。RCA 只需要 16 位，效率高得多。另外，RCA 是 Host 分配的，Host 可以按自己的策略管理地址空间。

**与 SD 卡的关键区别：**

| 特性 | eMMC | SD 卡 |
|------|------|-------|
| RCA 谁分配 | **Host 分配** | **卡自己分配** |
| RCA 出现在 | 命令参数中 | R1 响应中 |
| 原因 | eMMC 通常内置焊接（单卡），Host 直接给 | SD 卡可插拔，卡自行选取 |

SD 卡的 CMD3 在参数位置填 0x00000000，响应（R1）中才会返回卡自选的 RCA。eMMC 的 CMD3 参数直接填 Host 分配的 RCA。ATK 板只有一个 eMMC，内核通常分配 RCA=1。

#### CMD9 — SEND_CSD（读 CSD 寄存器）

**作用：** 读取卡的 CSD（Card Specific Data）寄存器——它描述了卡的基本能力。CSD 是 128 位宽的只读寄存器，通过 R2 响应返回。

**参数：** RCA（高 16 位），低 16 位保留。

**响应：** R2（136 bits = 128 位 CSD + 8 位 CRC+结束）。

**什么时候发：** CMD3 分配 RCA 之后、CMD7 选中卡之前。内核初始化序列：

```
CMD2(读CID) → CMD3(赋RCA) → CMD9(读CSD) → CMD7(选中卡)
                              ↑
                          拿到卡的能力，决定怎么操作
```

**CSD 寄存器结构（128 bits，仅列关键字段）：**

| 位域 | 字段 | 作用 |
|------|------|------|
| [127:126] | CSD_STRUCTURE | CSD 版本（v1.0=0, v2.0=1, v4.0+=2）|
| [103:96] | TRAN_SPEED | 最大时钟频率（编码值，见下方）|
| [95:84] | CCC | 支持的命令类（bitmask，12 bits）|
| [83:80] | READ_BL_LEN | 最大块长度（2^n，典型值 512 = n=9）|
| [79] | READ_BL_PARTIAL | 是否支持非整块读 |
| [73:62] + [49:47] | C_SIZE + C_SIZE_MULT | 卡容量（仅用于 ≤2GB 卡）|
| [45:39] | SECTOR_SIZE | 擦除扇区大小 |
| [25:22] | WRITE_BL_LEN | 最大写块长度 |

**TRAN_SPEED 编码（0x32 = 50MHz，0x2A = 25MHz，0x12 = 10MHz）：**

```
[103:96] 的高 3 位是倍率（0=1x, 1=10x, 2=100x），低 5 位是基值
  0x32 = 0b011 00010 → 倍率=10, 基值=50 → 50MHz ← ATK 板典型值
```

**高容量卡（>2GB）的容量怎么读：** CSD 中的 C_SIZE + C_SIZE_MULT 只能表达最大 2GB。对于 ATK 板上 14.6GiB 的 Kingston 卡，容量信息来自 **Ext_CSD[212:215]（SEC_CNT）**，通过 CMD8 读取，不是 CMD9。CSD 在高容量卡上的主要作用是告诉 Host 块大小、命令类支持、擦除粒度等"能力信息"。

**内核中对应的结构体：**

```c
// include/linux/mmc/card.h
struct mmc_card {
    struct mmc_csd csd;  // CMD9 的返回值存在这里

    u32 raw_csd[4];      // CSD 原始 128 位（4 × 32）
};
```

内核在 `mmc_read_csd()` 中解析 CSD 各字段，填充 `card->csd`，之后如 `mmc_set_bus_speed()` 中通过 `card->csd.max_dtr` 获取最大时钟限制。

### 2.2.2 数据传输命令

数据传输命令涵盖了从**发送请求 → 卡进入 Data 状态 → 数据在 DAT 线上传输 → 卡回到 Tran 状态**的完整过程。要理解这一节，先记住一个状态变迁：

```
CMD7(rca) 之后 → 卡在 Tran 状态（就绪）
   ↓
CMD17/18/24/25 → 卡从 Tran → Data/Receive/Program 状态
   ↓
数据传输完成   → 卡回到 Tran 状态
   ↓
CMD13          → 确认卡已回到 Tran，且无错误标志
```

每次数据命令执行前后都用 CMD13 确认状态——这是内核数据路径的标准做法。

#### CMD16 — SET_BLOCKLEN（设置块长度）

**作用：** 设置后续数据传输的块长度（字节数），最大 512。

参数：块长度值。响应：R1。

**为什么要有这条命令：** 早期的 MMC 支持 1~512 字节可变块长度。但到了高容量 eMMC（HCS=1），**CMD16 被完全忽略**——块长度固定为 512 字节。内核初始化时仍会发一次 CMD16(512)，纯粹是兼容性考量。

#### CMD17 — READ_SINGLE_BLOCK（单块读）

**作用：** 从指定地址读一个 512 字节块。

参数：数据地址。扇区寻址（HCS=1）时参数是扇区号，不是字节地址。

状态变迁：

```
CMD17 前 → 卡在 Tran
CMD17 发 → 卡进入 Data 状态
数据传完 → 卡回到 Tran
```

完整时序：

```
CMD:  ───[CMD17(addr)]──────[R1]───────────────[CMD13]──[R1]──
                              ↑ 卡→Data         确认卡回到 Tran
DAT:  ───────────────────────[数据块]────────────────────────
                             512 字节 + 2 字节 CRC16
```

**为什么传完后要用 CMD13 确认？** 因为 DAT 线上的 CRC16 只能检测数据是否传错，不能检测卡是否从 Data 状态回到了 Tran。如果卡卡在 Data 状态，发下一条命令会失败。内核在每次数据命令后发 CMD13 检查 `CURRENT_STATE=4(Tran)` 和 `READY_FOR_DATA=1`，这是数据路径的标准健壮性做法。

> 如果 CMD13 查到卡长时间卡在 Data 状态（例如超时后还在 Data），内核会走错误恢复：先试 CMD12 停止，再试 CMD0 复位，最后报告 I/O 错误。

#### CMD18 — READ_MULTIPLE_BLOCK（多块读）

**作用：** 从指定地址连续读取多个 512 字节块。

参数：起始地址。响应：R1 + 连续的 DAT 块序列。

状态变迁和单块读类似，但多了一个"要不要停止"的决策：

```
CMD18 → 卡进入 Data → 传块1 → 传块2 → ... → 停止决策
                                                 ↓
                                    ┌──── [有 CMD23] ──→ 自动停止，卡→Tran
                                    └──── [无 CMD23] ──→ CMD12 发 → 卡→Tran
```

**为什么需要 CMD12（STOP_TRANSMISSION）：**

多块读没有内置的"最后一块"标记。卡不知道自己该传多少块——它只是一块接一块地传，直到 Host 明确说"够了"。CMD12 就是那个"够了"的信号。

**无 CMD23 时每个 CMD12 产生一次 RTT：**

```
CMD18──块1──块2──...──块n──[CMD12──等 R1 响应]──完成
                               ↑ 额外一个 RTT（Round Trip Time）
```

> **RTT（Round Trip Time）** 指从 Host 发出一条命令到收到响应所花的往返时间。在 CLK=50MHz 时，一次 CMD12 的 RTT 大约包括：命令发出（11 个时钟）+ 卡处理（若干时钟）+ 响应返回（11 个时钟），约 0.5~1μs。单次 RTT 很小，但频繁发生时（如每读 4KB 就停一次），累积开销可观。

CMD23 正是为了解决这个问题——事先告诉卡块数，卡传完自动停，免去 CMD12 的 RTT。

#### CMD23 — SET_BLOCK_COUNT（预设块数）

**作用：** 预设定接下来多块传输的块数，配合 CMD18/CMD25 使用。

参数格式：

```
参数 [31]       [30]        [29:16]      [15:0]
    MSB(0)   可靠写位     保留          块计数(1~65535)
```

**有 CMD23 与无 CMD23 的时序对比：**

```
无 CMD23:  CMD18 → 块1 → 块2 → ... → 块n → [CMD12 → 等 R1] → Tran
                                              ↑ 每次多一个 RTT

有 CMD23:  CMD23(n) → CMD18 → 块1 → 块2 → ... → 块n → 自动→Tran
                                                      ↑ 零 RTT
```

**为什么 RTT 在小块频繁传输时特别明显？**

假设一次读 4KB（8 块），CLK=50MHz：

```
无 CMD23（每 8 块发一次 CMD12）:
  数据时间: 8 × 512 × 8 / 50MHz ≈ 655μs
  CMD12 RTT: ≈ 1μs
  RTT 占比: 0.15% ← 几乎可以忽略

有 CMD23（每读 8 块预设）:
  数据时间: 655μs
  CMD12 RTT: 0
  节省: 1μs / 655μs = 0.15% ← 几乎一样
```

但如果读操作反复启动、每次只读少量（如文件系统频繁读 inode）：

```
无 CMD23（每次读 1 块发一次 CMD12）:
  数据时间: 1 × 512 × 8 / 50MHz ≈ 82μs
  CMD12 RTT: ≈ 1μs
  RTT 占比: 1.2% ← 仍是小头
```

所以 CMD23 的真正价值不只在 RTT 节省，更在于**消除 CMD12 失败的风险**——CMD12 可能因卡忙而超时，错误恢复路径复杂。CMD23 从根本上避免了这个问题。

**可靠写（Reliable Write）：**

参数 bit 30 置位时启用——保证指定的多个块要么全部写入成功，要么全部失败（原子操作）。这对文件系统的元数据（FAT 表、ext4 journal）至关重要——写一半断电时，元数据半新半旧，文件系统就挂了。

**与 SD 卡的区别：**

SD 也有 CMD23，但 SD 的 CMD23 是 ACMD23（需要 CMD55 前缀），且主要用于预擦除而非可靠写。

#### CMD24 — WRITE_SINGLE_BLOCK（单块写）

**作用：** 从指定地址写入一个数据块。

流程：

```
CMD:  ───[CMD24(addr)]──[R1]──────────────────────[CMD13]──[R1]──
DAT:  ───────────────────[数据块]──(DAT0 busy)─────────────
                          512B+CRC16   ↑ 编程期间拉低
```

状态变迁：

```
CMD24 发 → 卡从 Tran → Rcv（接收数据）
数据收完 → 卡从 Rcv → Prg（编程中）
DAT0 解除 busy → 卡从 Prg → Tran
```

> **写后 CMD13 的作用：** 不光确认卡回到 Tran，还要查状态字中的 WP_VIOLATION 位（写保护违规）和 ERROR 位。如果 busy 时间异常长（> 卡指定的 max_busy_timeout），说明可能遇到了坏块，卡在内部做坏块管理。

写操作最耗时的不是 DAT 线上的数据传输，而是内部 Flash 编程。ATK 板 Kingston 58A43A 实测写约 66 MB/s，远低于读的 83 MB/s，差的就是 Flash 编程的固定开销。

#### CMD25 — WRITE_MULTIPLE_BLOCK（多块写）

**作用：** 连续写入多个数据块。

流程：

```
CMD:  ───[CMD25(addr)]──[R1]────────────────────────────
DAT:  ───────────────────[块1]──(busy)──[块2]──(busy)──...
                          ↑每块都要等编程完成
```

状态变迁比读复杂：

```
CMD25 发      → 卡从 Tran → Rcv
块1 收完       → 卡从 Rcv → Prg（块1 编程）
块1 编程完     → 卡从 Prg → Rcv（继续接收块2）
块2 收完       → 卡从 Rcv → Prg（块2 编程）
  ... 循环直到所有块写完成
             → 卡回到 Tran
```

**多块写 vs 多块读的性能差异根源：**

```
多块读: 传块1[82μs] → 传块2[82μs] → ... → 无等待
多块写: 传块1[82μs] → 编程[~几百μs] → 传块2[82μs] → 编程[...]
                      ↑ 每块都要等 Flash 写完，这是瓶颈
```

这也是 DDR52 8-bit 模式写性能无法达到理论带宽极限的根本原因——卡内部 Flash 编程速度是瓶颈，不是总线速度。

### 2.2.3 控制与状态命令

这一组的命令不传数据，但控制着**卡的状态迁移**和**模式配置**，是初始化和错误处理路径的核心。

#### CMD7 — SELECT/DESELECT_CARD（选中/取消卡）

**作用：** 选中一张已分配 RCA 的卡，使其进入 **Tran** 状态。RCA=0 时取消选中。**这是进入数据传输状态的最后一道门**——卡必须在 Tran 状态才能收发数据。

参数：RCA 在 [31:16]。响应：R1。

状态变迁：

```
Standby ─── CMD7(rca) ───→ Tran（选中，就绪）
Tran    ─── CMD7(rca=0) ──→ Standby（取消选中，省电）
```

**为什么选卡是单独的命令：** 在多卡系统中，CMD7 控制哪张卡当前占用数据总线。未被选中的卡在 Standby 状态保持低功耗。ATK 板只有一个 eMMC，CMD7 更像一个"启用"信号——卡必须先被选中，然后 CMD17/18/24/25 才能工作。

> **调试经验：** 如果 CMD13 查到卡在 Standby(3) 但数据命令一直失败，说明忘记发 CMD7 选卡了。内核初始化路径中 CMD7 紧跟在 CMD3 之后。

#### CMD12 — STOP_TRANSMISSION（停止传输）

**作用：** 强制停止正在进行的多块读写。

参数：无。响应：R1。

**什么时候必须用：**
- 没有 CMD23 预设块数就发起了 CMD18/CMD25，传够了需要停
- 数据命令出错需要紧急终止
- 读数据提前结束（文件系统只读了部分内容）

**风险点：** 如果卡正在 Data/Prg 状态，CMD12 可能因卡忙而超时。内核的错误恢复路径（`mmc_stop_request()`）有重试逻辑。重试多次失败后，最终的恢复手段是 CMD0 复位整张卡。

#### CMD13 — SEND_STATUS（查询状态）

**作用：** 读取卡的 32 位状态字，这是**调试最常用的命令**。内核在数据路径的每一个关键节点后都会发 CMD13 确认状态。

参数：RCA 在 [31:16]。响应：R1（48 位，含 32 位状态字）。

**为什么 CMD13 无处不在？**

CMD13 是 eMMC 调试者最重要的信息源。它不传数据，带宽消耗极小（48 位命令帧 + 48 位响应帧，在 50MHz 下不到 2μs），但能精确回答以下问题：

| 问题 | CMD13 怎么回答 |
|------|---------------|
| 卡现在在哪个状态？ | CURRENT_STATE 字段（4 bits） |
| 卡可以传数据了吗？ | READY_FOR_DATA 位 |
| 上一条命令卡接受了吗？ | ILLEGAL_COMMAND 位（bit 22） |
| 数据线上有 CRC 错误吗？ | COM_CRC_ERROR 位（bit 23） |
| 擦除/写入出错了吗？ | WP_VIOLATION、ERROR 等位 |

**CMD13 在内核数据路径中的典型用法：**

```
写一块数据:
  CMD24(addr) → [DAT 传数据] → DAT0 busy → [CMD13 反复发] → 等到 Tran + Ready
                                                                 ↑
                                                     轮询频率可达每 1~2μs 一次

读多块:
  CMD18      → [传块1]...[传块n] → [CMD13 确认回到 Tran]
                                          ↑一次确认即可
```

**总线测试模式：** 调试时可以手动发 CMD13 观察卡状态。五个典型结果见 2.1 节的 CURRENT_STATE 表。

**与 SD 卡的区别：** SD 的 CMD13 响应中多一点 SCC（SD Card Configuration）相关的状态位，但核心的 CURRENT_STATE+READY_FOR_DATA 与 eMMC 相同。

#### CMD6 — SWITCH（模式切换）

##### 用途：CMD6 是用来做什么的

CMD6 是 eMMC **唯一的配置命令**——它直接写入 Ext_CSD 寄存器，修改卡的工作模式。没有 CMD6，卡永远跑在出厂默认的 1-bit legacy 模式。

**它能改变什么（按 ATK 板使用频率排序）：**

| 配置项 | Ext_CSD 偏移 | 作用 | ATK 板是否用到 |
|--------|-------------|------|---------------|
| 总线宽度与 DDR 模式 | BUS_WIDTH[183] | 1/4/8-bit, SDR/DDR | **是** → 最终 8-bit DDR52 |
| 时序模式 | HS_TIMING[185] | Legacy/HS/HS200/HS400 | **是** → HS (mmc_select_hs) |
| 电源等级 | POWER_CLASS[187] | 电流上限等级 | 否（卡默认足够）|
| 缓存控制 | CACHE_CTRL[33] | 开/关内部缓存 | 内核配置但 ATK 板未使用 |
| 分区切换 | PART_CONFIG[179] | user/boot1/boot2/RPMB | 内核初始化时设置 |

**CMD6 和 CMD17/18/24/25 的本质区别：**

```
CMD17/18/24/25 → 读/写用户数据（卡的用户数据区）
CMD6          → 改寄存器值（卡的工作模式）

类比 MCU 开发：
  CMD17/24  = 读写 Flash 数据
  CMD6      = 写 MCU 外设寄存器（改 GPIO 模式、切时钟源）
```

##### 用法：CMD6 怎么用

CMD6 的参数直接编码了"要改哪个寄存器、改成什么值"：

```
参数 [31:26]   [25:24]     [23:16]    [15:8]     [7:3]   [2:0]
    保留      访问模式    寄存器索引   写入值     保留   命令集(0)
```

**一条具体的 CMD6：把 BUS_WIDTH[183] 改成 6（8-bit DDR）**

```
参数 = (3 << 24) | (183 << 16) | (6 << 8) | 0
        ↑           ↑            ↑         ↑
     访问模式3   寄存器索引     写入值    命令集0
     (Write Byte) (BUS_WIDTH)  (DDR52 8-bit)
```

**访问模式（四种）：**

| 值 | 宏 | 行为 | 用途 |
|----|----|------|------|
| 0 | MMC_SWITCH_MODE_CMD_SET | 改命令集 | eMMC 只有命令集 0，基本不用 |
| 1 | MMC_SWITCH_MODE_SET_BITS | 按位或 | 只置位不清理，操作独立 bit |
| 2 | MMC_SWITCH_MODE_CLEAR_BITS | 按位与非 | 只清理不置位 |
| **3** | **MMC_SWITCH_MODE_WRITE_BYTE** | **直接覆盖** | **实际最常用——用新值替换整个字节** |

内核源码中 CMD6 的封装（`drivers/mmc/core/mmc_ops.c:613`）：

```c
cmd.opcode = MMC_SWITCH;
cmd.arg = (MMC_SWITCH_MODE_WRITE_BYTE << 24) |  // 模式 3
          (EXT_CSD_BUS_WIDTH << 16) |             // 索引 183
          (ext_csd_bits << 8) |                   // 值 6
          EXT_CSD_CMD_SET_NORMAL;                 // 命令集 0
```

**响应：** R1b。因为写寄存器需要时间（DAT0 拉低表示 busy），超时时间由 `EXT_CSD_GENERIC_CMD6_TIME[248]` 给出，典型值 150ms。

##### 实际场景：ATK 板 DDR52 的完整切换流程

从卡上电到 DDR52 8-bit 模式就绪，内核共发 **3 次 CMD6**，顺序固定：

```
Step 1: CMD6 写 HS_TIMING[185] = 1
  → 函数: mmc_select_hs() in mmc.c:1076
  → 效果: 卡从 Legacy 模式进入 HS 模式
  → 同时 Host 侧时序切到 MMC_TIMING_MMC_HS
  → 此时 CLK 还很低（初始化时钟），后续提升

Step 2: Host 提升时钟频率
  → 函数: mmc_set_bus_speed() in mmc.c:985
  → 写 SDMMC_CLKCR 分频器，CLK 输出 ~52MHz
  → ATK 板实际 ~50MHz（PLL 分频粒度限制）

Step 3: CMD6 写 BUS_WIDTH[183] = 2（8-bit SDR）
  → 函数: mmc_select_bus_width() in mmc.c:1005
  → 从 1-bit 切换到 8-bit SDR 模式
  → ATK 板 MMC_CAP_8_BIT_DATA 置位，直接试 8-bit

Step 4: CMD6 写 BUS_WIDTH[183] = 6（8-bit DDR）
  → 函数: mmc_select_hs_ddr() in mmc.c:1094
  → 编码 EXT_CSD_DDR_BUS_WIDTH_8 = 6
  → 参数 timing = MMC_TIMING_MMC_DDR52：CMD6 完成后 Host 切到 DDR 模式
```

**为什么 Step 3 和 Step 4 都是写同一个寄存器？**

因为 DDR 模式不是 SDR 的"超集"。SDR 和 DDR 用不同的采样策略（单沿 vs 双沿），卡内部需要分别配置。内核的策略是：先切到最宽 SDR，再从 SDR 切到 DDR。这保证了卡和 Host 之间的时序同步。

**一句话总结 DDR52 初始化：** 一条 CMD6 改 HS_TIMING，一条 CMD6 改 BUS_WIDTH(SDR)，一条 CMD6 改 BUS_WIDTH(DDR)，中间穿插时钟升频。

##### 关于 VCCQ 电压的说明

DDR52 可以在 3.3V/1.8V/1.2V 三种 IO 电压下工作，具体用哪个由板级硬件设计决定，**不是通过 CMD6 切换的**。

ATK 板 eMMC 的 VCCQ（IO 电源）由硬件固定为 **1.8V**，上电即确定，内核不需要做任何电压切换。内核中 `mmc_select_hs_ddr()` 的电压切换代码（`mmc_set_signal_voltage()`）是为那些 VCCQ 可调的板子准备的——如果卡支持更低电压（如 1.2V DDR），内核会尝试调低 VCCQ 以降低功耗。对 ATK 板，这段代码不会实际改变电压。

> **一个容易混淆的点：** SD 卡在 UHS 模式下必须通过 CMD11 主动切换电压（3.3V→1.8V），这是 SD 协议的要求。eMMC 完全不同——电压由硬件固定，不是协议协商的一部分。如果你看到"eMMC 切电压"的说法，那是在说板级 PMIC 调节器，不是 eMMC 协议行为。

**与 SD 卡 CMD6 的区别：**

| 特性 | eMMC | SD 卡 |
|------|------|-------|
| 切换对象 | 直接写 Ext_CSD 寄存器（字节索引） | 写 SWITCH_FUNC 状态（函数组 + 6 个 group）|
| 总线宽度 | CMD6(索引=183, 值=6) | 专用 ACMD6（CMD55 + CMD6，arg 为总线宽）|
| 时序切换 | CMD6(索引=185, 值=1/2/3) | CMD6(函数组=1, 值=1) 切高速 |
| 寄存器机制 | 暴露寄存器地址，像 MCU 一样写 | 函数组（Function Group），4 个 bits |
| 初始化流程 | CMD1 协商电压，CMD6 配置模式 | CMD55+ACMD41 协商同时选模式 |
| 回退行为 | 掉电后 Ext_CSD 恢复默认 | 掉电后所有配置丢失 |

**本质差异：** eMMC 的 CMD6 是对 MCU 开发者最友好的设计——"写偏移地址 = 改对应寄存器"。SD 卡的 CMD6 基于"函数组"抽象，每个组有 4 个功能编号和 4 个保留编号，可读性远不如 eMMC 的直接寄存器映射。

#### CMD8 — SEND_EXT_CSD（读 Ext_CSD）

CMD6 的配套读命令，用于**读取完整的 512 字节 Ext_CSD 寄存器文件**。CMD6 写之前要先读、写之后也要读回来验证。

**命令类型：** adtc（带数据传输的寻址命令）——和 CMD17/CMD18 一样，从 DAT 线上返回数据块。

**参数：** 始终为 0x00000000。

**响应：** R1 + 512 字节数据块。

**数据块格式（固定 512 字节，无块长度变化）：**

```
Ext_CSD[0]  ……  Ext_CSD[511]
  最低地址         最高地址
```

每个寄存器（如 BUS_WIDTH[183]）都是这个数组中的一个字节。

**CMD8 在内核初始化中的位置：**

```
CMD0  →  CMD1  →  CMD2  →  CMD3  →  CMD9  →  CMD7  →  CMD8  →  CMD6(配置)
复位     协商     读CID    赋RCA   读CSD   选中卡   读Ext_CSD  写配置
                                                        ↑
                                                  先知道卡支持什么
                                                  才能决定怎么配
```

内核函数 `mmc_read_ext_csd()` 在 `drivers/mmc/core/mmc.c` 中实现，CMD8 返回的 512 字节被解析后填充到 `card->ext_csd` 结构体：

```c
// drivers/mmc/core/mmc.c
struct mmc_card {
    ...
    struct mmc_ext_csd ext_csd;  // ← CMD8 的返回值存在这里
};

struct mmc_ext_csd {
    u8          bus_width;       // EXT_CSD[183]，当前总线宽度配置
    u8          hs_timing;       // EXT_CSD[185]，当前时序模式
    u8          power_class;     // EXT_CSD[187]，当前电源等级
    unsigned int card_type;      // EXT_CSD[196..197]，卡支持哪些时序
    unsigned int sec_count;      // EXT_CSD[212..215]，卡容量（扇区数）
    unsigned int generic_cmd6_time; // EXT_CSD[248]，CMD6 写入超时 ms
    ...
};
```

**调试用法：** 内核没有直接暴露 CMD8 给用户态，但 `mmc_read_ext_csd()` 解析后的部分字段可通过 sysfs 查看：

```bash
# 查看卡当前时序模式和总线宽度（来自 debugfs ios）
$ cat /sys/kernel/debug/mmc1/ios | grep -E "timing|bus width"
timing spec:	14 (mmc DDR52)
bus width:	3 (8 bits)

# 查看卡容量（块设备大小，单位 512 字节扇区）
$ cat /sys/block/mmcblk1/size
62533296

# 查看 Ext_CSD 版本号
$ cat /sys/block/mmcblk1/device/rev
0x07

# 查看 CID 生产日期（不是 Ext_CSD，但可从 sysfs 直接读）
$ cat /sys/block/mmcblk1/device/date
04/2025

# 查看寿命估算（来自 Ext_CSD[267..269]）
$ cat /sys/block/mmcblk1/device/pre_eol_info
0x00
$ cat /sys/block/mmcblk1/device/life_time
0x00 0x00
```

> 注意：`EXT_CSD_CARD_TYPE[196]` 信息虽然内核会在 `mmc_read_ext_csd()` 中解析并存入 `card->ext_csd.card_type`，但**没有对应的 sysfs 属性**。内核仅将其用于内部的 `mmc_select_hs_ddr()` 等函数中做能力判断。

> **CMD6 vs CMD8 是一对"写/读"操作：** CMD6 写一个寄存器（修改卡的行为），CMD8 读所有寄存器（验证当前状态）。这在调试中非常有用——改完了读回来确认，和开发 MCU 外设的思路完全一致。

**与 SD 卡的 CMD8 的区别：**

这是一个典型的**同名不同命**命令——eMMC 和 SD 卡的 CMD8 功能完全不同：

| 特性 | eMMC CMD8 (SEND_EXT_CSD) | SD 卡 CMD8 (SEND_IF_COND) |
|------|--------------------------|---------------------------|
| 目的 | 读 512 字节 Ext_CSD 寄存器 | 检查电压是否匹配 + SD 协议版本检测 |
| 数据 | 512 字节（固定） | 8 字节（固定） |
| 响应 | R1 + DATA | R1 + DATA（8 字节） |
| 参数 | 恒为 0 | 电压检查模式（bit[11:8] + 检查码） |
| 调用时机 | 初始化后期（CMD7 之后） | 初始化早期（CMD0 之后 → ACMD41 之前） |
| eMMC 是否有等价功能 | — | SD 没有 Ext_CSD 概念 |

SD 卡的 CMD8 作用是：Host 发一个已知电压参数 + 检查码，卡用同样的检查码回复来确认"我们电压匹配"。而 eMMC 在 CMD1 中已经完成了电压协商，CMD8 是纯粹的寄存器读取命令。

### 2.2.4 擦除命令（CMD32/CMD33 + CMD38）

eMMC 的擦除分两步：先指定范围，再执行擦除。

| 命令 | 名称 | 参数 | 作用 |
|------|------|------|------|
| CMD32 | ERASE_WR_BLK_START | 起始地址 | 设置擦除范围开头 |
| CMD33 | ERASE_WR_BLK_END | 结束地址 | 设置擦除范围结尾 |
| CMD38 | ERASE | 无 | 执行擦除 |

**为什么不是一条命令搞定？**

因为擦除范围可能涉及多个命令之间的间隙（卡可能在做别的事），分两步让 Host 可以先设定范围、确认没问题后、再执行擦除。类似于"打开文件 → 确认文件名 → 删除文件"的动作拆分。

### 2.2.5 应用命令（CMD55 + ACMD）

#### CMD55 — APP_CMD（应用命令前缀）

**作用：** 告诉卡"接下来的一条命令不是标准命令，是一条应用特定命令"。

**参数：** RCA。

**响应：** R1（响应中 APP_CMD 位会置位，表示卡已接受此前缀）。

**为什么要设计 ACMD 机制：**

CMD0~CMD63 只有 64 个编码，标准协议就占了大部分。如果要扩展命令号（如 CMD64），就得改协议。MMC 的解决方案是：用 CMD55 作为"前缀"，后面的命令被解释为 ACMD（如 ACMD6、ACMD41、ACMD51），相当于把命令空间扩展了一倍。

**eMMC 实际用到的 ACMD：**

| 组合 | 用途 | eMMC 使用频率 |
|------|------|---------------|
| CMD55 + CMD6 | 改总线宽度（ACMD6） | **几乎不用** — eMMC 用 CMD6 + Ext_CSD 替代 |
| CMD55 + CMD41 | 电压协商（ACMD41） | **不用** — eMMC 用 CMD1 |
| CMD55 + CMD51 | 读 SCR 寄存器（ACMD51）| 初始化时使用，读卡的能力寄存器 |
| CMD55 + CMD13 | 读 SD Status（ACMD13）| **不用** — eMMC 用 CMD13 即可 |

**eMMC 和 SD 卡在用 ACMD 上的核心差异：**

```
SD 卡初始化： CMD0 → CMD55+ACMD41 (电压协商) → CMD2 → CMD3 → ...
                 ↑ 必须走 ACMD，不能用 CMD1

eMMC 初始化： CMD0 → CMD1 (电压协商) → CMD2 → CMD3 → ...
                 ↑ 直接用 CMD1，不走 ACMD
```

SD 卡完全依赖 ACMD 进行初始化，eMMC 只把 ACMD 作为扩展机制。这是两者协议血缘相近但实现分叉的关键区别。

### 2.2.6 调试速查表

| 场景 | 需要什么 | 发什么命令 | 期望响应 | 如果不对 |
|------|---------|-----------|---------|---------|
| 卡完全没反应 | 强制复位 | CMD0 | 无（无条件复位）| 检查 CLK 引脚有无信号 |
| 查支持的电压 | OCR 协商 | CMD1(0x00) | R3 + busy=0 说明在初始化 | 检查电源电压是否在卡范围内 |
| 查是否初始化完 | 轮询 busy | CMD1(ocr) | R3 + busy=1 | 超时退出，可能是卡坏了 |
| 读 CID | 卡身份标识 | CMD2 | R2（128 位 CID）| 如果无响应检查是否执行了 CMD1 |
| 分配 RCA | 地址 | CMD3(rca) | R1（状态进入 Standby）| 确认 CID 已拿到 |
| 读 CSD | 卡能力信息 | CMD9(rca) | R2（128 位 CSD）| TRAN_SPEED 决定最高时钟 |
| 选中卡 | 操作目标 | CMD7(rca) | R1（状态进入 Tran）| 确认卡在 Standby 状态 |
| 查当前状态 | 状态字 | CMD13(rca) | R1（CURRENT_STATE + 错误位）| 无响应 → RCA 不对或 CMD 线故障 |
| 读 Ext_CSD | 完整寄存器 | CMD8(0) | R1 + 512 字节数据 | 检查数据 CRC |
| 模式切换 | 配置寄存器 | CMD6 | R1b + busy | 检查 SWITCH_ERROR 位 |
| 预设块数 | 多块/可靠写 | CMD23(n) | R1 | 必须紧跟 CMD18/25 |
| 读数据 | 一个块 | CMD17(addr) | R1 + 数据块 | data_crc_err → 检查 DAT 线信号质量 |
| 读大量数据 | 多个块 | CMD18(addr) | R1 + 数据块序列 | CMD12 停止传输 |
| 写数据 | 写 + busy | CMD24(addr) | R1 + busy | busy 太长 → 可能是坏块 |
| 写大量数据 | 多个块 | CMD25(addr) | R1 + 数据块序列 + busy | 每块写入后卡进入 Prg 状态 |
| 停止传输 | 中止 | CMD12 | R1 | 多次重试后可能需复位卡 |
| 擦除 | 删数据 | CMD32/33 + CMD38 | R1 + busy | 擦除后的数据读出来应该是全 0 或全 1 |

## 2.3 MMC 状态机

MMC 协议定义了 **10 个状态**，卡在任何时刻都处于其中一个。状态的迁移由**命令触发**——每发一条合法命令，卡的状态就可能改变。理解状态机就是理解"发这个命令时卡应该在哪个状态，发完后卡会去哪里"。

本节不罗列状态定义，而是从**卡初始化和数据读写**两条实际流程来展示状态如何变迁。

### 2.3.1 初始化流程中的状态变迁

从卡上电到准备就绪，经过以下状态链：

```
Idle → Ready → Ident → Standby → Tran
```

整个初始化过程涉及两次关键的时钟频率变化：400kHz 识别频率 → 52MHz 高速频率。下面逐步骤标出时钟和总线模式。

```
Step 0: 上电或硬件复位
  状态: 卡进入 Idle
  时钟: ~400kHz（识别频率，host->f_init）
  总线: Open Drain（多卡兼容模式）
  说明: CMD 线处于开漏模式，所有卡可以同时拉低 CMD 线
        不会产生电气冲突。时钟限制在 400kHz 以下是因为
        Open Drain 模式下信号上升沿受 RC 电路限制，无法跑高频
        为何 400kHz: JEDEC 规范规定识别阶段的时钟必须在
        0~400kHz 范围，内核在 mmc_power_up() 中将时钟设为
        host->f_init（mmc/core/core.c:1354）

Step 1: CMD1（SEND_OP_COND）— 电压协商 + 初始化
  状态: Idle → Ready（当 OCR.BUSY=1 时）
  时钟: ~400kHz（保持识别频率）
  命令参数: 先发 0 探测，再发 card_ocr|HCS 确认
  响应: R3（OCR）
  总线: 保持 Open Drain（多卡同时响应，取电压交集）
  失败: 如果电压不匹配，卡永远不发 BUSY=1，Host 超时退出
  调试: 卡如果一直不回 BUSY=1，说明电压不在卡支持的范围内

Step 2: CMD2（ALL_SEND_CID）— 仲裁读 CID
  状态: Ready → Ident（赢得仲裁的卡）
  时钟: ~400kHz（保持识别频率）
  命令参数: 无（广播）
  响应: R2（128 位 CID）
  总线: 保持 Open Drain（仲裁利用开漏特性做逐位竞争）
  多卡: 每张卡依次进入 Ident，然后 CMD3 分配 RCA，再回到 Step 2
        直到所有卡都通过仲裁

Step 3: CMD3（SET_RELATIVE_ADDR）— 分配 RCA
  状态: Ident → Standby
  时钟: ~400kHz（识别频率的最后一次使用）
  命令参数: RCA（Host 分配，ATK 板为 1）
  响应: R1
  总线: Open Drain → **Push-Pull**（内核 mmc.c:1683 切换）
  说明: 这是 Open Drain 模式的最后一次使用。CMD3 完成后，
        总线切换到推挽模式，后续所有命令走高速单端信号
  为什么此时切换: Ident 之后只有一张卡被分配 RCA，
        不再需要多卡仲裁，推挽模式信号质量更好速度更快
  为什么此时不提频: 虽然总线已切 Push-Pull，但卡还未选中，
        内核在选中卡并读完能力寄存器前，还不知道卡的
        最高工作频率，所以仍保持 400kHz

Step 4: CMD9（SEND_CSD）— 读 CSD
  状态: Standby（发命令前后状态不变）
  时钟: ~400kHz（Push-Pull 模式但频率未变）
  命令参数: RCA
  响应: R2（CSD 寄存器，128 位）
  说明: CSD 是只读寄存器，读操作不改变状态。
        内核从 CSD 解析出 card->csd.max_dtr（最大时钟频率）

Step 5: CMD7（SELECT_CARD）— 选中卡
  状态: Standby → Tran
  时钟: ~400kHz
  命令参数: RCA
  响应: R1（CURRENT_STATE 变为 Tran）
  说明: Tran 是"传输就绪状态"，所有数据读写命令必须在此状态执行
  验证: 发 CMD13 确认 CURRENT_STATE = Tran

Step 6: CMD8（SEND_EXT_CSD）— 读 Ext_CSD
  状态: Tran（不改变状态）
  时钟: ~400kHz（仍在识别频率，Ext_CSD 512 字节在 400kHz
        下传输需约 10ms，是初始化中最耗时的一步）
  命令参数: 0x00000000
  响应: R1 + 512 字节数据
  说明: 此时卡已就绪，读 Ext_CSD 获取卡能力。
        内核从 Ext_CSD 解析出 card->ext_csd.hs_max_dtr（52MHz）
        和 card->mmc_avail_type（支持的传输模式）

Step 7a: CMD6（SWITCH）— 写 HS_TIMING=1
  状态: Tran（不改变状态）
  时钟: ~400kHz（提频前的准备，先告诉卡"我要切高速了"）
  命令: __mmc_switch(card, CMD_SET_NORMAL, HS_TIMING, 1,
         MMC_TIMING_MMC_HS) （mmc.c:1080）
  响应: R1b
  说明: 将 Ext_CSD[185] 设为 1，卡内部切换到高速模式。
        内核在 __mmc_switch 中通过最后一个参数 timing
        同时更新 host->ios.timing = MMC_TIMING_MMC_HS

Step 7b: 内核提频（mmc_set_bus_speed）
  时钟: ~400kHz → ~52MHz（mmc.c:997, mmc_set_clock）
  说明: CMD6 返回后，内核调用 mmc_set_bus_speed(card)，
        根据 card->ext_csd.hs_max_dtr 将时钟从 400kHz
        直接提升到 52MHz。
  为什么先切 HS_TIMING 再提频: 如果先提频再发 CMD6，
        卡还在 backward compatible 时序模式，跟不上
        52MHz 时钟会导致 CRC 错误。必须先告诉卡"我要提频了"
        （CMD6 HS_TIMING=1），再提频。

Step 7c: CMD6（SWITCH）— 配置总线宽度和 DDR
  时钟: ~52MHz（现在工作在高速模式）
  子步骤:
    ├─ mmc_select_bus_width(): CMD6 写 BUS_WIDTH=2
    │  （8-bit SDR，mmc.c:1830）
    └─ mmc_select_hs_ddr(): CMD6 写 BUS_WIDTH=6
       （8-bit DDR52，mmc.c:1110-1112）
  响应: R1b（每次写入后 DAT0 busy）
  说明: 总线宽度配置必须在提频之后进行，因为 Ext_CSD
        读写需要完整的时钟才能及时完成
```

**初始化的时钟频率变化全景：**

```
时钟
  ^
  │  400kHz（识别频率）                52MHz（高速模式）
  │  ───────────────                    ─────────
52Mhz┤                                    ← Step 7b 提频
  │                                    ← mmc_set_bus_speed()
  │                                    ← card->ext_csd.hs_max_dtr
  │
400k┤←──── CMD0~CMD8 ────────────────→
  │   Step 0~6                          
  │                                    
  │
  └────────────────────────────────────────────────────► 时间
     卡识别阶段                          数据传输阶段
     Open Drain → Push-Pull             DDR52 8-bit
```

**为什么初始阶段必须用 400kHz：**

- **Open Drain 模式限制**：开漏输出的上拉电阻导致信号上升沿很慢，频率一高信号就变形
- **卡还没初始化完**：此时卡的内部稳压器刚启动，PLL 还没锁相，给不了高速时钟
- **兼容多卡仲裁**：多卡在 Open Drain 模式下逐位竞争 CID，需要所有卡在同一时钟边沿采样

**提频条件：** 内核只有在完成以下条件后才会提频：
1. 卡已在 Tran 状态（CMD7 选中）
2. 已读完 Ext_CSD，确认 card->ext_csd.hs_max_dtr
3. CMD6(HS_TIMING=1) 已成功执行
4. 卡回复 OPERATION_COMPLETE（R1b busy 释放）

### 2.3.2 数据读写中的状态变迁

卡进入 Tran 后，所有数据操作都按"Tran → 传输态 → Tran"的三步模式执行。

**单块读（CMD17）流程：**

```
Tran 状态          Data 状态         Tran 状态
  │                  │                  │
  ├─ CMD17(addr) ──→ │                  │    ← 命令触发状态迁移
  │   返回 R1        │  数据在 DAT 线    │
  │                  │  上传回           │
  │                  ├─ 传输完成 ───────→│    ← 自动回到 Tran
  │                  │                  │
  │◄── CMD13 确认 ───┤                  │    ← 验证 CURRENT_STATE=Tran
```

具体过程：
1. 卡在 Tran 状态，Host 发 CMD17(addr)
2. 卡收到命令后立即从 Tran 切换到 Data 状态（此时 CMD 线上返回 R1）
3. 卡从指定地址读取数据，从 DAT 线上发送
4. 数据发送完成后，卡自动从 Data 回到 Tran
5. Host 发 CMD13 确认 CURRENT_STATE=Tran（实际内核在 mmc.c 的数据路径中不是每一步都发 CMD13，只有关键节点验证）

**多块读（CMD18）流程：**

```
Tran ──CMD18──→ Data ──(传块1)(传块2)...(传块n)──→ Tran（自动）
                │                                    ↑
                └──── CMD12(手动停止) ────────────────┘
```

两种退出 Data 的方式：
- **自动退出**：CMD23 预设了块数且发完所有块 → 自动回到 Tran
- **手动退出**：没预设块数或想提前停止 → 发 CMD12（在 Data 状态发 CMD12 有效）

**单块写（CMD24）流程：**

```
Tran 状态          Rcv 状态          Prg 状态          Tran 状态
  │                  │                  │                │
  ├─ CMD24(addr)───→ │                  │                │
  │   返回 R1        │  等待数据         │                │
  │   发送数据块 ────→│  接收完成         │                │
  │                  ├─ 进入 Prg ──────→ │                │
  │                  │                  │ 内部擦写 Flash  │
  │                  │                  │ DAT0 拉低 busy  │
  │                  │                  ├─ 写完 ────────→ │
  │                  │                  │                │
  │◄── CMD13 确认 ───┘                  └── 状态确认 ────→│
```

关键区别：
- 读操作走 **Tran → Data → Tran**（Data 状态下 DAT 线传数据）
- 写操作走 **Tran → Rcv → Prg → Tran**（Rcv 收数据，Prg 擦写 Flash）
- Prg 状态期间 **DAT0 拉低表示 busy**，这是 CMD13 之外的硬件 busy 信号
- 如果卡编程时间长，DAT0 会一直低，Host 可以通过 `MMC_CAP_WAIT_WHILE_BUSY` 硬件等待

**多块写（CMD25）流程：**

```
Tran ──CMD25──→ Rcv ──(块1)──→ Prg ──→ Rcv ──(块2)──→ Prg ──→ ... ──→ Tran
                 │                │
              DAT0 busy       DAT0 busy
```

每块写完后卡都会进入 Prg 状态擦写 Flash，完成后如果还有下一块就回到 Rcv 继续收，收完最后一块回到 Tran。

**CMD23（预设块数）+ CMD18/25 的优化：**

不预设块数时：卡不知道要发多少块，会一直停留在 Data/Rcv 直到收到 CMD12 才停止。

```
无 CMD23:   CMD18 → Data → ... → CMD12 → Tran
有 CMD23:   CMD23(n) → CMD18 → Data → (自动，发完 n 块) → Tran
                                           ↑ 不需要 CMD12，省一次 RTT
```

CMD23 的主要作用就是**消除尾部 CMD12 的 RTT（~1μs）**。对高频小数据块读写有明显提升。

### 2.3.3 状态迁移速查表

| 当前状态 | 触发命令 | 下一个状态 | 实际场景 |
|---------|---------|----------|---------|
| Idle | CMD1（BUSY=1）| Ready | CMD1 协商成功 |
| Idle | CMD0 | Idle | 复位后不动 |
| Ready | CMD2（仲裁胜出）| Ident | 赢得 CID 仲裁 |
| Ident | CMD3 | Standby | 收到 RCA |
| Standby | CMD7(rca) | Tran | 选中卡 |
| Standby | CMD7(0) | Standby | 不选中任何卡（deselect）|
| Tran | CMD7(0) | Standby | 取消选中 |
| Tran | CMD7(other_rca) | Standby | 切到另一张卡 |
| Tran | CMD17/18 | Data | 开始读数据 |
| Tran | CMD24/25 | Rcv | 开始写数据 |
| Tran | CMD6/8/9/13/16 | Tran | 不改变状态（读寄存器/配置）|
| Data | 传输完成 | Tran | 数据读完了 |
| Data | CMD12 | Tran | 停止多块读 |
| Rcv | 数据收完 | Prg | 开始内部擦写 |
| Prg | 编程完成 | Tran | Flash 擦写结束 |
| Prg | — | Tran | busy 释放后自动切回 |
| 任意状态 | CMD0 | Idle | 软复位 |
| 任意状态 | 上电/硬件复位 | Idle | 硬复位 |

### 2.3.4 实践意义

**为什么理解状态机对调试重要：**

- **卡没响应 → 检查 CURRENT_STATE**：发 CMD13 看状态字。如果卡在 Prg（编程中），它根本不会响应 CMD17/18，不是卡坏了，是还在写。
  ```
  # Prg 状态下发 CMD18 会怎么样？
  卡不返回数据，CMD 线上 R1 的 ILLEGAL_COMMAND 位置位。
  因为 Prg 状态下不允许数据读命令。
  ```
- **写性能问题 → 看 Prg 状态的 busy 时间**：CMD24 后 DAT0 拉低的时间就是卡擦写 Flash 的时间。如果持续几毫秒以上，考虑用 CMD23 的可靠写优化。
- **初始化卡在某个状态 → 定位到具体 Step**：如果 CMD13 读回的 CURRENT_STATE 一直是 Ready 而不是 Ident，说明 CMD2 仲裁有问题，检查 CMD 线电气特性。
- **CMD12 停止后检查状态**：CMD12 发完后必须确认卡回到 Tran，否则后续命令会失败。内核 `mmc_stop_request()` 中发完 CMD12 后会发 CMD13 确认。

## 2.4 传输模式演进：PIO → SDMA → ADMA

SD/MMC 控制器传输数据的方式经历了三代演进。每一代解决的核心问题都是同一个：**如何更快地把数据在 DDR 和 eMMC 之间搬完，同时让 CPU 少干活**。STM32MP2 SDMMC 控制器的 IDMA（Internal DMA）实现了 SDMA 和 ADMA 两种模式，分别对应两代演进。

### 2.4.1 PIO（Programmed I/O）— CPU 逐字搬运

**工作原理：**

CPU 逐字地从 SDMMC FIFO 读取或写入数据。每传输一个字（4 字节），控制器触发一次中断，CPU 在中断处理程序中搬下一个字。

```
CPU (DDR)          SDMMC FIFO (16 words)      DAT[7:0]
  │                      │                      │
  │ 读 FIFO（每次 1 word）│                      │
  │←──── 中断驱动 ───────│←──── 从卡接收数据 ────│
  │                      │                      │
  │ 写 FIFO（每次 1 word）│                      │
  │──── 中断驱动 ───────→│──── 发送给卡 ────────→│
```

**关键参数（STM32MP2 SDMMC）：**

| 参数 | 值 |
|------|-----|
| FIFO 深度 | 16 字 × 4 字节 = 64 字节 |
| FIFO 半满中断阈值 | FIFOHALF = 8 字 = 32 字节 |
| 每次中断处理的数据量 | 最多 8 字（32 字节）|

**中断频率计算（DDR52 52MHz 8-bit）：**

```
DDR52 8-bit 理论带宽 = 52 MHz × 8 bit × 2(DDR 双边沿) = 83 MB/s
每 32 字节触发一次中断 → 83 MB/s ÷ 32 B = 2.6 百万次中断/秒
```

每秒钟 260 万次中断，每次中断涉及上下文保存、寄存器访问、中断返回——CPU 几乎全部时间花在搬数据上。

**内核中的体现（mmci.c）：**

```c
static struct variant_data variant_stm32_sdmmcv2 = {
    .fifosize       = 16 * 4,    /* 16 words × 4 bytes = 64 bytes */
    .fifohalfsize   = 8 * 4,     /* 半满阈值 8 字 = 32 字节 */
    .irq_pio_mask   = MCI_IRQ_PIO_STM32_MASK,
};
```

DMA 不可用或数据量很小时，驱动自动回退到 PIO（mmci.c:1042）：

```c
/* If less than or equal to the fifo size, don't bother with DMA */
if (data->blksz * data->blocks <= variant->fifosize)
    return 0;  // 不走 DMA，直接用 PIO
```

> **PIO 仍然存在的理由：** 小于 64 字节的小数据（如 ioctl、读 Ext_CSD 等），设置 DMA 描述符的开销比直接 PIO 还大。

### 2.4.2 SDMA（Single DMA）— 单地址 DMA

**解决的问题：** PIO 模式下 CPU 被中断淹没了。让 DMA 控制器代替 CPU 在 DDR 和 SDMMC FIFO 之间搬数据，CPU 只需设置一次传输参数。

**SDHCI 标准中的 SDMA：**

SD Host Controller 标准规范了 SDMA 模式——控制器内部只有一个 DMA 地址寄存器和一个块计数组件。CPU 设置好 DDR 地址和传输大小，SDMA 硬件自动完成全部数据传输，完成后触发一次完成中断。

**STM32MP2 IDMA 单段模式（等效 SDMA）：**

STM32MP2 的 IDMA 在非链表模式下的行为等同于 SDMA——设置 DDR 地址 + 传输大小，硬件自动搬完。

```
IDMACTRLR = IDMAEN（bit0=1）
IDMABASE0R = DDR 起始地址
IDMABSIZER = 传输字节数

CPU (DDR)            IDMA 引擎           SDMMC FIFO        DAT[7:0]
  │                    │                    │                │
  │ 写 IDMABASE0R      │                    │                │
  │ 写 IDMABSIZER      │                    │                │
  │ 写 IDMACTRLR=EN    │                    │                │
  │───────────────────→│                    │                │
  │                    │──── 读 DDR ───────→│←── DAT 线数据 ─│
  │                    │──── 读 DDR ───────→│←── DAT 线数据 ─│
  │                    │  ...(CPU 休眠)...  │                │
  │◄── DATAEND 中断 ───│                    │                │
```

对应代码（mmci_stm32_sdmmc.c:272-275）：

```c
writel_relaxed(dma_addr, host->base + MMCI_STM32_IDMABASE0R);
writel_relaxed(MMCI_STM32_IDMAEN, host->base + MMCI_STM32_IDMACTRLR);
```

**SDMA 的限制：**

单次传输只能处理一段连续内存。如果数据在 DDR 中不连续（文件系统典型场景——scatter-gather list），需要：
- 要么先拷贝到连续 buffer（bounce buffer）
- 要么多次设置 IDMABASE0R，每次处理一个分片，每次都需要中断 CPU

驱动在 sdmmc_idma_validate_data() 中处理了这个限制——当 scatterlist 不满足对齐约束时，使用 bounce buffer：

```c
// mmci_stm32_sdmmc.c:100-107
for_each_sg(data->sg, sg, data->sg_len - 1, i) {
    if (!IS_ALIGNED(sg->offset, sizeof(u32)) ||
        !IS_ALIGNED(sg->length, host->variant->stm32_idmabsize_align)) {
        goto use_bounce_buffer;  // 不满足对齐 → 用 bounce buffer
    }
}
```

### 2.4.3 ADMA（Advanced DMA）— 描述符表 DMA

**为什么需要 ADMA：** SDMA 只能处理连续内存，用 bounce buffer 多了拷贝开销。ADMA 使用 DDR 中的描述符表，可以在一次传输中处理多个不连续的内存块。

**ADMA2 标准描述符（SDHCI 规范）：**

ADMA2 使用 64-bit 描述符，每个描述符包含数据地址和数据长度，通过链表串联多个不连续 buffer。ADMA2 支持以下控制标志：

| 标志 | 含义 |
|------|------|
| Valid | 描述符有效 |
| End | 链表结束 |
| Int | 此描述符完成后触发中断 |
| Act(0) | 无操作 |
| Act(1) | 传输数据 |

**STM32MP2 IDMA 链表模式（等效 ADMA2）：**

当 `dma_lli = true` 时（STM32MP25 variant），IDMA 支持链表模式，本质就是 ADMA2 的概念——DDR 中的描述符表 + 硬件链表跳转。

描述符结构（mmci_stm32_sdmmc.c:52）：

```c
struct sdmmc_lli_desc {
    u32 idmalar;    // 下一个描述符的字节偏移 + 控制标志
    u32 idmabase;   // 数据 buffer 的 DDR 地址
    u32 idmasize;   // 数据 buffer 的大小
};
```

每个描述符 12 字节，控制标志编码在 `idmalar` 的低 3 位：

| 标志 | 位 | 含义 |
|------|----|------|
| ULA (Update Link Address) | bit0 | 使能链表跳转（= ADMA 的 Valid） |
| ULS (Update Link Size) | bit1 | 更新传输大小 |
| ABR (Abort) | bit2 | 传输完成后中止 |

**链表传输的工作流程：**

```
DDR 中的描述符表：

        desc[0]                    desc[1]                    desc[2]（最后）
  ┌────────────────┐        ┌────────────────┐        ┌────────────────┐
  │ idmalar        │        │ idmalar        │        │ idmalar        │
  │  ├ next offset │   ──→  │  ├ next offset │   ──→  │  ├ ULA=0      │ ← 链表结束
  │  ├ ULA=1       │        │  ├ ULA=1       │        │  ├ ULS=1       │
  │  ├ ULS=1       │        │  ├ ULS=1       │        │  └ ABR=1       │
  │  └ ABR=1       │        │  └ ABR=1       │        │                │
  ├────────────────┤        ├────────────────┤        ├────────────────┤
  │ idmabase=addrA │        │ idmabase=addrB │        │ idmabase=addrC │
  ├────────────────┤        ├────────────────┤        ├────────────────┤
  │ idmasize=4KB   │        │ idmasize=8KB   │        │ idmasize=2KB   │
  └────────────────┘        └────────────────┘        └────────────────┘
         ▲                          ▲                          ▲
         │ DDR 物理地址              │ DDR 物理地址              │ DDR 物理地址
         ▼                          ▼                          ▼
  ┌──────────────┐          ┌──────────────┐          ┌──────────────┐
  │  数据块 A    │          │  数据块 B    │          │  数据块 C    │
  │  (4KB)       │          │  (8KB)       │          │  (2KB)       │
  └──────────────┘          └──────────────┘          └──────────────┘
```

初始化过程（mmci_stm32_sdmmc.c:279-296）：

```c
// Step 1: CPU 在 DDR 中构建描述符表
for_each_sg(data->sg, sg, data->sg_len, i) {
    desc[i].idmalar = (i + 1) * sizeof(struct sdmmc_lli_desc);
    desc[i].idmalar |= MMCI_STM32_ULA | MMCI_STM32_ULS | MMCI_STM32_ABR;
    desc[i].idmabase = sg_dma_address(sg);
    desc[i].idmasize = sg_dma_len(sg);
}
desc[data->sg_len - 1].idmalar &= ~MMCI_STM32_ULA;  // 最后一项: 链表结束

// Step 2: 通知 IDMA 描述符表基址
writel_relaxed(idma->sg_dma, host->base + MMCI_STM32_IDMABAR);

// Step 3: 写入首个描述符
writel_relaxed(desc[0].idmalar,  host->base + MMCI_STM32_IDMALAR);
writel_relaxed(desc[0].idmabase, host->base + MMCI_STM32_IDMABASE0R);
writel_relaxed(desc[0].idmasize, host->base + MMCI_STM32_IDMABSIZER);

// Step 4: 使能 IDMA（使能 + 链表模式）
writel_relaxed(MMCI_STM32_IDMAEN | MMCI_STM32_IDMALLIEN,
               host->base + MMCI_STM32_IDMACTRLR);
```

**传输过程（以读为例）：**

```
  DDR                       IDMA 引擎                      SDMMC FIFO
  │                            │                              │
  │ 写入 IDMABAR               │                              │
  │ 写入 IDMACTRLR(EN+LLI)    │                              │
  │───────────────────────────→│                              │
  │                            │← 读 IDMABAR → 找到 desc table
  │                            │                              │
  │                            │ 读 desc[0].idmabase         │
  │ desc[0].idmabase=addrA ───→│                              │
  │                            │ IDMA 从 SDMMC FIFO 收数据   │
  │ ◄───────────────────────── │ ← ← ← ← ← ← ← ← ← ← ← ←  │
  │   addrA: 数据块 A          │                              │
  │                            │                              │
  │ desc[0] 完成               │                              │
  │ idmalar.ULA=1 → 跳转      │ 读 desc[1].idmabase         │
  │ desc[1].idmabase=addrB ───→│                              │
  │                            │ IDMA 继续收数据              │
  │ ◄───────────────────────── │ ← ← ← ← ← ← ← ← ← ← ← ←  │
  │   addrB: 数据块 B          │                              │
  │                            │                              │
  │ ...                        │                              │
  │                            │                              │
  │ desc[n-1].ULA=0            │ 链表结束 → DATAEND 中断     │
  │◄─── DATAEND 中断 ──────────│                              │
```

**IDMA ADMA 模式的关键优势：**

1. **CPU 零干预**：设置好描述符后，IDMA 自动遍历链表搬完所有数据，CPU 可以休眠
2. **硬件链表跳转**：IDMA 引擎硬件读取 idmalar 中的 next 偏移，自动跳转到下一个描述符
3. **scatter-gather 原生支持**：文件系统的 buffer 天然不连续，一次描述符表处理所有分片
4. **瓶颈不在描述符遍历**：描述符表在 DDR 中，IDMA 引擎以 DMA 速度读取，远快于 eMMC 总线速度

### 2.4.4 三代模式对比

| 特性 | PIO | SDMA（IDMA 单段）| ADMA（IDMA 链表）|
|------|-----|-----------------|-----------------|
| **数据搬运者** | CPU | IDMA 引擎 | IDMA 引擎 |
| **每次传输 CPU 干预** | 每字一次中断 | 整个传输一次中断 | 整个传输一次中断 |
| **中断频率（83MB/s）** | 260 万次/s | ~500 次/s | ~500 次/s |
| **DDR 地址设置方式** | CPU 直接读 FIFO | 写 IDMABASE0R | 描述符表 + IDMABAR |
| **scatter-gather** | 不支持 | 不支持（需 bounce buffer）| **原生支持** |
| **适合场景** | ≤ 64 字节小数据 | 连续大块数据 | 不连续多 buffer 数据 |
| **设置开销** | 无 | 低（3 次 register write）| 中（构建描述符表）|
| **内核代码路径** | mmci.c PIO 中断 | mmci_stm32_sdmmc.c sdmmc_idma_start 单段分支 | mmci_stm32_sdmmc.c sdmmc_idma_start 链表分支 |

当数据量 ≤ 64 字节时 → PIO（cpu 直接搬，无设置开销最划算）
当有 scatterlist 且满足对齐约束 → ADMA 链表模式（一次描述符表处理所有分片）
当有 scatterlist 但不满足对齐约束 → SDMA 单段模式 + bounce buffer（先拷贝到连续 buffer）

## 2.5 STM32MP2 SDMMC 控制器特性

SDMMC2 控制器是 ARM PrimeCell PL180 的 ST 定制版本，非标准 SDHCI IP。基址 0x48220000，挂载在 AXIM 总线。

### 支持的传输模式与速率

| 模式 | 最大时钟 | ATK eMMC 是否使用 | 驱动支持情况 |
|------|---------|-----------------|------------|
| Backward Compatibility (Legacy) | ~26 MHz | × 初始化阶段短暂使用 | mmci.c + variant ops |
| High Speed (HS-SDR) | 52 MHz | × 过渡状态 | mmc_select_hs() |
| **HS-DDR (DDR52)** | **52 MHz DDR** | **✓ 最终模式** | mmc_select_hs_ddr() |
| HS200 (SDR) | 200 MHz | × eMMC 不支持 | 需 tuning（MP25 Delay Block）|
| HS400 (DDR) | 200 MHz DDR | × eMMC 不支持 | 需 tuning + HS200 fallback |

ATK 板实测工作在 DDR52 8-bit 模式，52 MHz 双沿采样，读 ~83 MB/s、写 ~66 MB/s。

### IDMA 引擎（核心特性）

控制器内置 IDMA，支持两种模式（详见 2.4）：

- **SDMA 模式**：设 IDMABASE0R + IDMABSIZER 一次传输一段连续内存
- **ADMA 模式**（dma_lli）：DDR 中的描述符表 + 硬件链表跳转，原生 scatter-gather
- **Bounce buffer 兜底**：当 scatterlist 不满足对齐约束时，驱动自动回退到 bounce buffer + 单段 IDMA

IDMA 描述符 table 与 DDR 的直接交互是 STM32MP25 新增的特性（dma_lli = true，定义在 mmci_stm32_sdmmc.c variant_data），MP15 仅支持单段模式。

### 时钟与总线配置

控制器时钟来自 CK_BUS（通常 260 MHz），通过 CLKCR 的分频器产生 SDMMC_CK：

```
SDMMC_CK = mclk / (2 × CLKDIV)    // 非 DDR 模式
SDMMC_CK = mclk / (2 × CLKDIV)    // DDR 模式（双沿采样，不支持 bypass）
```

时钟控制寄存器（CLKCR@0x004）的关键配置位：

| 特性 | 位 | 说明 |
|------|----|------|
| 时钟分频 | bit9:0 CLKDIV | 0 = bypass（直通，仅非 DDR 模式）|
| 1-bit/4-bit/8-bit | bit14/bit15 | 总线宽度选择 |
| DDR 模式 | bit18 | 使能双沿采样 |
| 硬件流控 | bit17 HWFCEN | DAT 线忙时自动延长时钟周期 |
| Bus Speed | bit19 | SDR104/HS200 使能 |
| 时钟源选择 | bit21:20 | CK/SELCK/SELFBCK（HS200 时选内部反馈时钟）|

### 数据 FIFO

FIFO 深度 64 字节（16 字 × 4 字节），是 PIO 模式和中断频率的关键参数。FIFO 阈值寄存器（FIFOTHRR@0x044）允许配置 HS200/HS400 模式下 FIFO 中断的触发水位。

PIO 模式下 FIFO 半满（32 字节）触发中断，CPU 每次搬 8 个字。DMA 模式下 FIFO 直接与 IDMA 引擎交互，CPU 不参与。

### 命令路径（CPSM）

CPSM 管理命令的发送和响应接收：

- 写 MMCIARGUMENT 填入参数 → 写 MMCICOMMAND 触发发送
- 响应类型可配置：无响应 / 短响应(48-bit) / 长响应(136-bit)
- 硬件自动计算 CRC7 并校验响应 CRC
- 响应完成后触发 CMDRESPEND 中断

短响应（R1/R1b/R3）从 RESPONSE0 读取 32 位状态字，长响应（R2）从 RESPONSE0~3 读取 4 次（128 位 CID/CSD）。

### 数据路径（DPSM）

DPSM 管理与 eMMC 之间的数据块传输：

- 块模式：单块或多块传输，支持自动停止（CMD23 预设块数）
- 方向控制：读（Device→Host）或写（Host→Device）
- 数据超时：由 DATATIMER 寄存器设定，单位是时钟周期
- IDMA 使能后，数据在 DDR ↔ IDMA ↔ FIFO ↔ DAT 线之间自动流转

### 中断管理

所有事件汇集到 STATUS 寄存器（0x034），通过 MASK0（0x03C）使能/屏蔽。关键中断源：

| 中断 | 触发条件 | 调试用途 |
|------|---------|---------|
| CMDRESPEND | 命令响应已接收 | 确认命令已送达 |
| DATAEND | 数据传输完成 | 确认数据已传完 |
| CMDTIMEOUT | 命令无响应 | 卡没回应（检查连线、电压）|
| DATATIMEOUT | 数据无响应 | 数据线问题 |
| BUSYD0 | DAT0 正在 busy | R1b 响应后检查卡状态 |
| BUSYD0END | DAT0 busy 结束 | R1b busy 释放 |
| TXUNDERRUN | FIFO 空但还需发送 | DMA 未及时填充 FIFO |
| RXOVERRUN | FIFO 满但还需接收 | CPU/DMA 未及时读取 FIFO |

### 延时块与信号调优（Delay Block）

HS200/HS400 模式下，因为时钟频率高（200 MHz），CMD/DAT 信号的采样窗口可能偏移。STM32MP2 SDMMC 内置了 Delay Block（DLYB）：

- **MP25（STM32MP2x）**：使用 SYSCFG_DLYBSD 延迟块，支持 32 级 TAP 选择，通过 tuning 过程（sdmmc_dlyb_phase_tuning）找到最优采样相位
- **MP15（STM32MP1x）**：内建 DLYB，11 级延迟 + unit 选择

tuning 过程（sdmmc_execute_tuning）遍历所有相位，发送 CMD21（SEND_TUNING_BLOCK），选择误码率最低的相位作为最终配置。ATK 板 eMMC 工作于 DDR52 模式（52 MHz），不需要 tuning。

### 与标准 SDHCI 控制器的差异

这是调试时最容易混淆的地方：

| 方面 | 标准 SDHCI | STM32MP2 SDMMC |
|------|-----------|---------------|
| **IP 核** | SD Host Controller Standard | ARM PrimeCell PL180 定制版 |
| **DMA** | ADMA2 标准描述符 | IDMA 自定义描述符 |
| **寄存器布局** | 按 SDHCI 规范 0x00~0xFC | 非标准偏移 |
| **CMD 触发** | 分步写 Argument + Command | 写 Argument → 写 Command(ENABLE) |
| **时钟分频** | SDCLK Frequency Select | CLKDIV + bypass + DDR 模式 |
| **电压切换** | SDHCI 标准信号 | VSWITCH + VSWITCHEN 自定义控制 |
| **硬件流控** | 标准支持 | HWFCEN 自定义控制 |
| **总线宽度** | 1/4/8-bit 标准 | WIDEBUS_4/WIDEBUS_8 自定义位 |
| **驱动文件** | drivers/mmc/host/sdhci-*.c | drivers/mmc/host/mmci.c + mmci_stm32_sdmmc.c |

## 2.6 本章总结

### 五层知识速查

| 层面 | 核心要点 |
|------|---------|
| **2.1 总线协议** | CMD 线 48-bit 帧（CRC7），DAT 线每线独立帧（CRC16），R1/R2/R3 三种响应 |
| **2.2 命令集** | 初始化 CMD0→1→2→3→7→8 顺序严格；CMD6 写 Ext_CSD 是唯一配置手段；CMD23 省 CMD12 RTT |
| **2.3 状态机** | 10 个状态，初始化走 Idle→Ready→Ident→Standby→Tran，数据操作走 Tran↔Data/Rcv/Prg↔Tran |
| **2.4 传输模式** | PIO(≤64B) → SDMA(连续) → ADMA(链表+scatter-gather)，IDMA 描述符含 ULA/ULS/ABR 控制标志 |
| **2.5 控制器** | ARM PL180 定制版，非 SDHCI，IDMA 内置，DDR52 52MHz 8-bit 实测读 83MB/s 写 66MB/s |

### 初始化全流程（时钟 + 状态 + 命令）

```
Step    命令              时钟         卡状态          总线模式
 0      上电              400kHz       Idle            Open Drain
 1      CMD1(ocr)         400kHz       Idle→Ready      Open Drain
 2      CMD2              400kHz       Ready→Ident     Open Drain
 3      CMD3(rca)         400kHz       Ident→Standby   Open Drain→Push-Pull
 4      CMD9(rca)         400kHz       Standby          Push-Pull
 5      CMD7(rca)         400kHz       Standby→Tran     Push-Pull
 6      CMD8(0)           400kHz       Tran             Push-Pull
 7a     CMD6(HS_TIMING=1) 400kHz       Tran             Push-Poll  ← 先切时序
 7b     mmc_set_bus_speed 400kHz→52MHz Tran             Push-Pull  ← 再提频
 7c     CMD6(BUS_WIDTH=6) 52MHz        Tran             Push-Pull  ← 后配位宽
```

### 读/写路径对比

```
读操作：       CMD17/18 → [DAT 传数据] → CMD13(确认 Tran)     ← 无 busy
写操作：       CMD24/25 → [DAT 传数据] → [DAT0 busy] → CMD13  ← 每块等编程
瓶颈：         读受总线带宽限制，写受卡内部 Flash 编程速度限制
实测 ATK：     读 ~83 MB/s，写 ~66 MB/s（差值 = Flash 编程开销）
```

### 后续方向

本文硬件知识直接对应内核源码中的以下模块：

| 硬件概念 | 内核源码对应 | 后续文档 |
|---------|------------|---------|
| CMD/响应/状态机 | `core/mmc_ops.c`, `core/mmc.c` | 03-Architecture.md |
| 数据传输/IDMA | `host/mmci_stm32_sdmmc.c` | 04-Probe-Analysis.md |
| 时钟/位宽/时序配置 | `host/mmci.c` → `mmci_set_ios()` | 04-Probe-Analysis.md |
| 块设备/队列 | `core/block.c`, `core/queue.c` | 05-DataPath-Analysis.md |
| 寄存器级操作 | `host/mmci_stm32_sdmmc.c` + RM | 06-Hardware-Registers.md |
