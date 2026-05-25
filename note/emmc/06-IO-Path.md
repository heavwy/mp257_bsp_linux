# 06. eMMC I/O 数据通路情景分析

> 本文是 STM32MP257 eMMC 驱动深度分析系列的第 6 篇。
> 从 `submit_bio` 进入块层开始，追踪 `dd read` / `dd write` / `sync` 三个真实情景在 MMC 核心层、Host 驱动、IDMA 硬件中的完整路径。
>
> **前置:** [05-VFS-to-Block.md](05-VFS-to-Block.md) — VFS 与块层通路（从系统调用到 submit_bio）
> **下一篇:** [07-Advanced.md](07-Advanced.md) — CQHCI 与 Tuning
>
> **字数：** 中文字数 8,302 + 英文单词 2,226 ≈ **10,528 字**（含代码段），**行数：1,475**
>
> **建议阅读时间：** 60–90 分钟

---

## 6.1 三个情景的纵向路线

本文分析三个用户态操作在内核存储栈中的完整路径。结构上采用"三幕剧"递进：第一幕读路径是被动触发（缺页→等 IO），第二幕写路径是主动发起（异步→后台刷盘），第三幕 sync 是强制收盘（软硬双层清算）。每幕内部拆为四个断面，从上到下穿透 VFS→块层→MMC 核心→控制器→硬件。

### 三幕速览

| 操作 | 内核路径特征 | 进程行为 | CMD |
|------|-------------|---------|-----|
| `dd if=/dev/mmcblk1 bs=4k count=1` | Page Cache 未命中 → VFS 构造 bio → blk-mq 下发 → MMC core 发 CMD18 → IDMA 从 SDMMC2 读数据到内存 → 中断 → 进程唤醒 | 进程在 `folio_wait_locked` 上睡眠，IO 完成后被唤醒 | CMD18 |
| `dd if=/dev/zero of=/mnt/testfile bs=4k count=256` | 数据写入 Page Cache → `__folio_mark_dirty` → 返回。后台 `wb_workfn` 在脏页超阈值/超时后刷出 → 构造 bio → blk-mq → MMC core 发 CMD25 → IDMA 从内存写到 SDMMC2 | 写系统调用在数据进入 Page Cache 后立即返回，刷盘由内核线程异步执行 | CMD25 |
| `sync` | 脏页全量刷出 → 文件系统 journal 提交 → `blkdev_issue_flush` 下发 `REQ_PREFLUSH` → MMC core 调用 `_mmc_flush_cache` → CMD6 写 `EXT_CSD[32]` → eMMC 内部将 SRAM Cache 刷入 NAND | 进程阻塞直到 CMD6 完成 | CMD6 |

### 单硬件队列约束

`drivers/mmc/core/queue.c:433` 设 `tag_set.nr_hw_queues = 1`。SDMMC2 只有一个 DMA 通道，同一时刻只允许一个 `mmc_request` 在总线上传输。`mmc_blk_mq_issue_rq()` 内部通过 `mmc_claim_host()` 实现互斥。blk-mq 的 per-CPU 无锁提交通道在此场景下仅缓解了软件层的锁竞争，硬件层仍是串行执行。断面 1.1 会深入分析这个"多对一"的转化细节。

### 本文与 03/04 的分工

03 和 04 已覆盖的数据结构定义（`mmc_host_ops`、`mmc_request`、`mmc_command`、`mmc_data`）和初始化流程（`mmc_init_card` CMD 序列、块设备注册），本文不再重复解释。本文从 `submit_bio` 进入块层开始追踪，逐层向下到 `mmci_irq` 中断返回为止。

---

---

## 6.2 分界线：05 止于 submit_bio，06 始于 submit_bio

[05-VFS-to-Block.md](05-VFS-to-Block.md) 完整追踪了从 `read()/write()` 系统调用到 `submit_bio()` 的 VFS + 块层路径。本文直接从 `submit_bio()` 之后开始——即第一幕从 `blk_mq_submit_bio()` 出发，经过块层、MMC 核心层、Host 驱动、IDMA，直到中断返回。

本文假设读者已经理解：
- VFS 四大对象在块设备场景中的角色（见 05 第 5.3 节）
- Page Cache 的三级读决策（`filemap_get_pages`，见 05 第 5.5.2 节）
- Buffered Write 的异步本质与 `wb_workfn` 后台回写（见 05 第 5.6.3 节）

各幕的断面编号重新从 1 开始，不再保留旧的 1.1（VFS 层内容）。

---

## 第一幕：dd read —— 被动的"拉"数据通路

```bash
echo 3 > /proc/sys/vm/drop_caches   # 清除所有缓存，让读走真实 IO
dd if=/dev/mmcblk1 of=/dev/null bs=4k count=1  # 读 eMMC 第一块
```

**行为特征：** 进程同步阻塞。由于 Page Cache 被清空，`dd` 在 `folio_wait_locked` 上睡眠，直到数据从 eMMC 取回、中断处理完成、`folio_unlock` 被调用，进程才被唤醒。

### 断面 1.1：块层 → blk-mq → MMC 核心入口

#### 函数追踪

```
submit_bio(bio)                          <-- block/blk-core.c
  └─ blk_mq_submit_bio(bio)              <-- block/blk-mq.c
       ├─ blk_mq_get_tag()               <-- 从 tags 池拿一个 tag (请求标识号)
       ├─ blk_mq_bio_to_request()        <-- bio → struct request 转换
       │    └─ blk_rq_append_bio()
       │         └─ __blk_rq_map_bio()   <-- 将 bio 中的 bvec 链到 request 中
       ├─ blk_mq_sched_bio_merge()       <-- 尝试合并到已有 request（mq-deadline 介入）
       └─ blk_mq_try_issue_directly()    <-- 直通模式：不经过 kblockd 工作队列
            └─ __blk_mq_issue_directly()
                 └─ blk_mq_request_issue_directly()
                      └─ q->mq_ops->queue_rq()
                           = mmc_mq_queue_rq()    <-- MMC 块层入口！
```

#### 关键步骤拆解

**步骤 1：拿到 tag——`blk_mq_get_tag()`**

blk-mq 的"tag"系统是它和旧块层（`blk_init_queue`）最根本的区别。旧块层 IO 路径中动态分配 `struct request`，blk-mq 在初始化时**一次性预分配所有 request**，用 tag 管理它们的分配和回收：

```c
// include/linux/blk-mq.h:735 — tag 池结构
struct blk_mq_tags {
    unsigned int nr_tags;            // 总共多少槽位 (= queue_depth)
    struct request **rqs;            // ★ 预分配的 request 指针数组
    struct sbitmap_queue bitmap_tags; // 空闲槽位位图（sbitmap: 无锁并发分配）
};
```

**tag 的本质就是数组下标**——拿到 tag = N，`rqs[N]` 就是这次 IO 用的 `struct request *`，O(1) 直接索引：

```c
// include/linux/blk-mq.h:754 — tag 转 request
static inline struct request *blk_mq_tag_to_rq(struct blk_mq_tags *tags,
                                               unsigned int tag)
{
    prefetch(tags->rqs[tag]);  // 提前加载到 cache line
    return tags->rqs[tag];
}
```

blk-mq 初始化时调用 `blk_mq_alloc_map_and_rqs()` 分配 `nr_tags` 个 `struct request` 和等量的位图条目。IO 路径上只需要从位图中找一个 0→1 的 bit，不需要做任何 `kmalloc`/`kfree`——所有内存分配都在初始化阶段完成：

```
初始化:  blk_mq_alloc_map_and_rqs(set, hctx_idx, depth)
           ├─ alloc: tags->rqs[] = kcalloc(nr_tags, sizeof(struct request *))
           ├─ alloc: tags->bitmap_tags = sbitmap_queue(nr_tags)
           └─ for i in 0..nr_tags: rqs[i] = alloc_request(set->driver_tags) ✱
                                                 ↑
                                           每个 request 预分配完成

IO 路径:  blk_mq_get_tag()
           ├─ tag = sbitmap_queue_get(&tags->bitmap_tags)  ← 位图操作，无锁
           ├─ rq = tags->rqs[tag]                           ← O(1) 拿 request
           └─ return tag

IO 完成:  blk_mq_put_tag(tags, tag)
           └─ sbitmap_queue_clear(&tags->bitmap_tags, tag)  ← 位图清 0
```

如果 64 个 tag 都用完了，第 65 个并发 IO 来了怎么办？`sbitmap_queue_get()` 返回负值——拿不到 tag。`submit_bio()` 调用者（上层文件系统或用户进程）在这里阻塞等待，直到某个 IO 完成后 `blk_mq_put_tag()` 归还 tag。这就是 **tag 系统的背压机制**——块层用 tag 数量控制并发深度，防止 MMC 层被淹死。

**那 MMC 只有一个队列，blk-mq 的"多队列"到底好在哪？**

MMC 用 blk-mq 的真正收益不是"多队列并行"（因为硬件只有一个队列），而是 **预分配 + tag 替换了动态分配**。旧块层（`blk_init_queue`）每条 IO 请求都要 `kmalloc` 一个 `struct request`，IO 完成后再 `kfree`——高并发下分配/释放开销不小。blk-mq 初始化时一次性 `kcalloc` 64 个 `struct request`，IO 路径上只需从位图拿一个 tag，0 次内存分配：

```
旧块层 (blk_init_queue):           blk-mq:
  submit_bio                          submit_bio
    ↓                                   ↓
  kmalloc(struct request)   ❌ 慢      sbitmap_get_tag()  ✅ 位图操作，几纳秒
  ... 填数据 ...                        rq = tags->rqs[tag]  ✅ 数组索引
  submit()                             submit()
  ↓                                   ↓
  完成 → kfree(request)     ❌ 慢      完成 → sbitmap_clear_tag() ✅ 位图清 0
```

这是 blk-mq 对所有块设备（包括 MMC）的共同好处。NVMe 额外享受了多队列并行，但 MMC 即使只有一个队列，**预分配 tag 带来的零分配 IO 路径也已经是巨大的提升**。

**MMC 的 `nr_tags` 具体是多少？** 在非 CQHCI 模式下（ATK 板 DDR52），`drivers/mmc/core/queue.c:401` 硬编码：

```c
#define MMC_QUEUE_DEPTH 64
```

所以 MMC 块层最多允许 **64 个在途 request**。对 `dd bs=4k count=1` 只有 1 个 request，实际看不到排队效果。但如果并发随机读压力大（如 `fio --rw=randread --iodepth=32`），多个进程同时提交 IO，tag 可能耗尽——`submit_bio` 阻塞在 `blk_mq_get_tag()`，形成自然的流量控制。

对比 CQHCI 模式（eMMC v5.0+，ATK 板未启用）：`queue_depth` 由 `card->ext_csd.cmdq_depth` 决定，通常是 32——因为 **CQHCI 硬件只有 32 个任务槽位**。此时 tag 不再只是软件索引，它**直接就是硬件队列的 slot 编号**：拿到 tag=5，意味着 CMD 写入 CQHCI 的 slot 5。所以 `queue_depth` 必须等于硬件队列深度，不能多也不能少。

**步骤 2：bio 转 request——`blk_mq_bio_to_request()`**

拿 `dd if=/dev/mmcblk1 bs=4k count=1` 举例。VFS 层构造的 bio 长这样：

```
struct bio (简化)
┌──────────────────────────┐
│ bi_opf       = REQ_OP_READ    ← "我要读"
│ bi_iter      = {              ← "从哪读到哪"
│   bi_sector  = 0,             ←   从 LBA 0 开始
│   bi_size    = 4096           ←   读 4096 字节
│ }
│ bi_io_vec[0] = {              ← "数据放哪"
│   bv_page    = page_3fb8...,  ←   DDR 物理页地址
│   bv_len     = 4096,          ←   这页 4KB
│   bv_offset  = 0              ←   从页头开始
│ }
└──────────────────────────┘
```

`blk_mq_bio_to_request()` 把这个 bio 的信息搬到 tag 拿到的 `struct request *rq` 里：

```c
// block/blk-mq.c — 简化
blk_mq_bio_to_request(rq, bio, nr_segs)
    // 1. 操作类型和扇区号直接抄
    rq->cmd_flags = bio->bi_opf;            // REQ_OP_READ
    rq->__sector = bio->bi_iter.bi_sector;  // LBA 0

    // 2. 把 bio 挂到 request 上
    blk_rq_append_bio(rq, bio);
    // → rq->bio = bio              ← request 持有 bio 指针
    // → rq->biotail = bio          ← 链表尾
    // → bio 中的 bvec 数组后面会被转为 scatterlist（断面 1.2 的 R3）
```

转换后的 request：

```
struct request (简化)
┌──────────────────────────────┐
│ tag         = 5                  ← 从位图拿到的 tag，槽位 5
│ __sector    = 0                  ← 读 LBA 0
│ nr_sectors  = 8                  ← 读 8 个扇区（4096 ÷ 512）
│ cmd_flags   = REQ_OP_READ        ← 读操作
│ bio         = ─────→ 指向原来的 bio（数据页地址在里面）
│ q           = ─────→ 所属的 request_queue
└──────────────────────────────┘
```

**关键点：** request 不拷贝 bio 的数据页，只是通过 `rq->bio` 指针引用它。bio 中的 `bi_io_vec[]` 数组会在后面（断面 1.2）被 `blk_rq_map_sg()` 转为 scatterlist，最终编程为 IDMA 描述符。这是"零拷贝"数据路径的第一环。

**步骤 3：IO 调度器合并——`blk_mq_sched_bio_merge()`**

`mq-deadline` 就是块层的 **IO 调度器**（也叫电梯算法）。它干的事和电梯调度类似——电梯不可能每上一层就掉头，IO 调度器也不可能让磁盘每写一个扇区就寻道一次。它负责决定 request 的**顺序**和**合并**。

`mq-deadline` 是 blk-mq 版 deadline 调度器，它的核心设计是**两队列 + 截止时间**：

```
               incoming bio
                    │
                    ▼
        ┌───────────────────────┐
        │   sorted queue        │  ← 按扇区号排序（减少寻道）
        │   ┌───┬───┬───┬───┐  │
        │   │ 0 │ 8 │16 │24 │  │
        │   └───┴───┴───┴───┘  │
        │                       │
        │   fifo queue          │  ← 按到达时间排序（防饿死）
        │   ┌───┬───┬───┬───┐  │
        │   │ 0 │ 8 │16 │24 │  │
        │   └───┴───┴───┴───┘  │
        └───────────────────────┘
                    │  dispatcher
                    ▼
               host driver
```

**sorted queue** — request 按扇区号升序排列。对 HDD 这是核心：磁头顺序扫描，减少寻道时间。对 eMMC 意义不大（固态无寻道），但也不影响正确性。

**fifo queue** — 记录每个 request 的到达时间。每个 request 有一个 deadline（读默认 500ms，写默认 5s）。如果某个 request 在 sorted queue 里等太久快要超时，调度器会"插队"把它提前发出去，防止它饿死。

**merge** — 新来的 bio 如果和队列中某个 request 扇区相邻，直接合并。比如 `sector 0-7` 正在排队，又来了 `sector 8-15` 的 bio，合并成一个 `sector 0-15` 的 request。最终 MMC 发一条 CMD18(blocks=16) 代替两条 CMD18(blocks=8)——**合并减少了一次命令开销**。

对 `dd bs=4k count=1`，只有一个 request，调度器什么都不做。对 `fio --rw=read --iodepth=32`，32 个并发 request 在调度器中排序 + 合并后再下发给 MMC。

**步骤 4：直通下发——`blk_mq_try_issue_directly()`**

```c
blk_mq_try_issue_directly(hctx, rq, ...)
    // 检查是否可以直接下发（不经过 kblockd 工作队列）
    // 条件：CPU 亲和性匹配、没有其他请求正被处理
    __blk_mq_issue_directly(hctx, rq, ...)
        blk_mq_request_issue_directly(rq)
            q->mq_ops->queue_rq(hctx, &bd)     ← 回调！
                = mmc_mq_issue_rq(hctx, bd)     ← MMC 块层入口!
```

`q->mq_ops` 是 `struct blk_mq_ops`，在 MMC 中注册为 `mmc_mq_ops`（`drivers/mmc/core/queue.c:340`）。其中的 `queue_rq` 回调是 `mmc_mq_queue_rq()`——**这是通用块层和 MMC 子系统的边界**。到此为止我们还是在内核通用块层，下一步 `mmc_mq_queue_rq` 开始进入 MMC 专属代码。

---

> **技术核弹点 R2："伪多队列"——blk-mq 在 MMC 上的妥协**

blk-mq 最初为 NVMe 设计——NVMe 控制器有多个硬件队列（每个队列独立 DMA、独立中断），多核 CPU 可以无锁并发提交到不同队列。但 MMC 只有一个硬件队列（`nr_hw_queues = 1`）：

```c
// drivers/mmc/core/queue.c:433 — 单队列约束
static int mmc_mq_init_queue(struct mmc_card *card, int slot)
{
    ...
    tag_set->nr_hw_queues = 1;    // ← 只有一个硬件队列！
    ...
}
```

这意味着什么？

| 特性 | NVMe (真多队列) | MMC (伪多队列) |
|------|---------------|----------------|
| `nr_hw_queues` | 多个（通常每个 CPU 一个） | 1 |
| 提交方式 | per-CPU 无锁，各提各的 | per-CPU 无锁，但都排到同一个队列 |
| 并发执行 | 多个队列同时 DMA | 一个请求串行执行 |
| 流量闸门 | 硬件自动仲裁 | `mmc_claim_host()` 互斥锁 |

host 的互斥获取通过 `mmc_get_card()` 间接完成：

```c
// drivers/mmc/core/queue.c — mmc_mq_queue_rq() 简化
static blk_status_t mmc_mq_queue_rq(struct blk_mq_hw_ctx *hctx,
                                     const struct blk_mq_queue_data *bd)
{
    ...
    // 第一个在途请求需要获取 host 锁
    if (get_card)
        mmc_get_card(card, &mq->ctx);    // ← 内部调 __mmc_claim_host()
    
    blk_mq_start_request(req);
    issued = mmc_blk_mq_issue_rq(mq, req);  // ← 真正的 IO 下发
    ...
}
```

`mmc_get_card()`（`drivers/mmc/core/core.c:861`）内部调用 `__mmc_claim_host()`。这不是信号量，而是一个**waitqueue 自定义锁**——等待的进程通过 `add_wait_queue` 加入 `host->wq` 等待队列后调用 `schedule()` 让出 CPU，`mmc_release_host()` 在另一个核上调用 `wake_up(&host->wq)` 唤醒它。`host->claimed` 标志位由 spinlock `host->lock` 保护：

```c
// drivers/mmc/core/core.c
void mmc_get_card(struct mmc_card *card, struct mmc_ctx *ctx)
{
    pm_runtime_get_sync(&card->dev);            // 保证控制器未挂起
    __mmc_claim_host(card->host, ctx, NULL);    // ← 真正的闸门！
}
```

为什么 `mmc_get_card` 是条件调用（`get_card` 为真时才执行）？MMC 使用 `mq->in_flight` 计数器跟踪在途请求数，只有第一个请求时才获取 host 锁，后续请求复用已有锁——它们串行在 `mmc_blk_mq_issue_rq` 内部排队。

所以 blk-mq 的 per-CPU 无锁提交通道在 MMC 场景下仅缓解了软件层的锁竞争，硬件层仍然是严格的串行执行。

**但 blk-mq 仍带来了好处：** tag 系统的并发限流。`tag_set.queue_depth` 限制了同时在途的 request 数量，防止上层过量下发导致 MMC 层缓冲溢出。tag 耗尽时 `submit_bio` 自然阻塞，形成天然的背压（backpressure）机制。

---

#### mq-deadline 在 MMC 上的实际效果

`mq-deadline` 是 `elevator_get_default()`（`block/elevator.c:570`）在单设备队列下的默认选择。但它对 MMC 的意义不同于 HDD：

```
HDD 场景：
  排序   → 减少磁头寻道，效果显著
  合并   → 提高单次传输量

eMMC 场景（固态存储）：
  排序   → 无磁头，意义不大，但不影响正确性
  合并   → 有效！相邻扇区的 IO 合并后，MMC 层只需更少的 CMD
```

对本文的 `bs=4k count=1` 场景，只有一个 request，调度器基本不做合并。但如果并发运行多个读，mq-deadline 会将相邻的请求合并成一个多块读（CMD18），显著提高吞吐。

---

#### 断面 1.1 终点状态

```
进程:           dd (PID xxx), 调用 folio_wait_locked() 已睡眠
                → 进程状态 S (sleeping) on  folio->wait
request:        已构造, 已标记 tag=N
                cmd_flags = REQ_OP_READ | REQ_SYNC
mq-deadline:    请求已入队（如果不是直通下发）
MMC 核心:       mmc_mq_queue_rq → mmc_blk_mq_issue_rq
                host 已通过 mmc_get_card() 加锁
下一个状态:     进入 mmc_blk_issue_rw_rq() → 决定 CMD 类型
```

#### 从 request 到 CMD——即将看到的景象

```
struct request         struct bio             struct mmc_request
┌─────────────┐       ┌──────────┐            ┌─────────────┐
│ __sector    │───────│ bi_sector│            │ cmd         │─→ CMD18
│ nr_sectors  │       │ bi_io_vec│            │ data        │─→ SG list
│ bio         │──────→│  [0]: pg │            │  sg         │
│ cmd_flags   │       │  [1]: pg │            │  blocks     │
└─────────────┘       └──────────┘            └─────────────┘
```

断面 1.2 中，我们从 `mmc_blk_issue_rw_rq()` 进入，看这个 request 如何被翻译成 CMD18 多块读命令和 SG list。

### 断面 1.2：MMC 核心 → CMD 翻译 + SG List 构建

#### 函数追踪

```
mmc_mq_queue_rq()                         <-- queue.c:227 (blk-mq 回调)
  └─ mmc_blk_mq_issue_rq(mq, req)         <-- block.c:1990
       └─ mmc_blk_issue_rw_rq(mq, req)    <-- block.c:1256 (读写分发)
            └─ mmc_blk_rw_rq_prep()        <-- block.c:1652: 准备命令
                 └─ mmc_blk_data_prep()    <-- block.c:1363
                      ├─ 填充 data.blksz/block/blk_addr
                      ├─ 设置 FLAGS (MMC_DATA_READ/WRITE)
                      ├─ mmc_set_data_timeout()  <-- block.c:1466
                      └─ mmc_queue_map_sg()      <-- block.c:1469: SG 构建!
                 └─ 决定 CMD 号                   <-- block.c:1672
                      ├─ blocks > 1: CMD18 (READ_MULTIPLE_BLOCK)
                      └─ blocks = 1: CMD17 (READ_SINGLE_BLOCK)
            └─ mmc_set_info()              <-- block.c:1282
            └─ mmc_start_request()         <-- core.c:168: 发送请求!
                 └─ host->ops->request()
                      = mmci_request()     <-- 进入 host driver!
```

#### 步骤拆解

**步骤 1：`mmc_blk_mq_issue_rq()` —— 请求分发器**

`mmc_blk_mq_issue_rq()`（`drivers/mmc/core/block.c:1969`）根据 request 的类型分流：

```c
// drivers/mmc/core/block.c — 简化
static int mmc_blk_mq_issue_rq(struct mmc_queue *mq, struct request *req)
{
    struct mmc_blk_data *md = mq->blkdata;
    struct mmc_card *card = md->queue.card;
    
    switch (req_op(req)) {
    case REQ_OP_READ:
    case REQ_OP_WRITE:
        if (mq->rpmb_partition)      // RPMB 走特殊路径
            return mmc_blk_issue_rpmb_rq(mq, req);
        return mmc_blk_issue_rw_rq(mq, req);  // ← 普通读写走这里
    case REQ_OP_FLUSH:                 // sync 路径
        return mmc_blk_issue_flush(mq, req);
    case REQ_OP_SECURE_ERASE:
        ...
    }
}
```

对 `dd read` 来说 `req_op(req) = REQ_OP_READ`，进入 `mmc_blk_issue_rw_rq()`。

**步骤 2：`mmc_blk_rw_rq_prep()` —— 填充 MMC 命令**

`mmc_blk_issue_rw_rq()`（`block.c:1256`）做的工作可以概括为：**"把这个 `struct request` 转成 MMC 协议能理解的命令序列"**。核心入口是 `mmc_blk_rw_rq_prep()`，它内部再调 `mmc_blk_data_prep()` 填充数据部分。

```c
// drivers/mmc/core/block.c — mmc_blk_rw_rq_prep() 简化
static void mmc_blk_rw_rq_prep(struct mmc_queue_req *mqrq,
                                struct mmc_card *card, ...)
{
    struct mmc_blk_request *brq = &mqrq->brq;
    struct request *req = mmc_queue_req_to_req(mqrq);
    
    // ① 填充数据属性
    mmc_blk_data_prep(mq, mqrq, ...);
    
    // ② 决定 CMD 号
    if (brq->data.blocks > 1)
        readcmd = MMC_READ_MULTIPLE_BLOCK;   // CMD18
    else
        readcmd = MMC_READ_SINGLE_BLOCK;      // CMD17
    brq->cmd.opcode = readcmd;
    
    // ③ 命令参数
    brq->cmd.arg = blk_rq_pos(req);          // 起始扇区号
    brq->cmd.flags = MMC_RSP_R1 | MMC_CMD_ADTC;
    
    // ④ 需要时加 CMD23
    if (card->ext_csd.ext_csd_cmd_set & EXT_CSD_CMD_SET_NORMAL)
        brq->mrq.sbc = &brq->sbc;            // 前导块计数命令
}
```

重点看 **`mmc_blk_data_prep()`**——它把 `struct request` 里的信息填入 MMC 核心能识别的 `struct mmc_data`：

```c
// drivers/mmc/core/block.c:1363 — mmc_blk_data_prep() 核心字段
brq->data.blksz    = 512;                       ← eMMC 扇区固定 512 字节
brq->data.blocks   = blk_rq_sectors(req);       ← request 的总扇区数
brq->data.blk_addr = blk_rq_pos(req);            ← request 的起始 LBA
brq->data.sg       = mqrq->sg;                   ← scatterlist 缓冲区(预分配)
brq->data.sg_len   = mmc_queue_map_sg(mq, mqrq); ← bio_vec → SG 转换
```

这里 `mmc_queue_map_sg()` 是 **bio_vec → scatterlist** 的转换器。它调用 `blk_rq_map_sg()` 把 request 中所有 bio 的 `bi_io_vec[]` 数组合并成连续的 scatterlist：

```
bio_vec → scatterlist 转换 (mmc_queue_map_sg)

bio 1:                       bio 2:
bi_io_vec[0] = {page_A, 4K}  bi_io_vec[0] = {page_B, 4K}
bi_io_vec[1] = {page_A, 4K}  bi_io_vec[1] = {page_C, 2K}
      │                            │
      ▼                            ▼
      └──────────┬─────────────────┘
                 ▼
      blk_rq_map_sg()
          │
          ├─ 遍历所有 bio 的所有 bvec
          ├─ 检查相邻段物理地址是否连续
          │   (page_A_end == page_B_start? → 合并)
          └─ 输出 scatterlist[]:
                 sg[0] = {dma_addr=page_A, len=8K}   ← 合并了 page_A 的两段
                 sg[1] = {dma_addr=page_B, len=4K}
                 sg[2] = {dma_addr=page_C, len=2K}
```

**`struct scatterlist`** —— DMA 传输的基本单元，描述一段物理连续的内存区域：

```c
// include/linux/scatterlist.h:11
struct scatterlist {
    unsigned long  page_link;    // struct page * ＋ 低 2 bit 标记位
    unsigned int   offset;       // 页内偏移（字节）
    unsigned int   length;       // 本段长度（字节）
    dma_addr_t     dma_address;  // ★ DMA 总线地址（dma_map_sg 后填充）
};
```

**`page_link` 存了两样东西：**

```
page_link (64-bit):
┌──────────────────────────────┬──┬──┐
│  struct page *  (高 62 bit)   │C │E │
└──────────────────────────────┴──┴──┘
                                 │  │
                           bit 1 ┘  └─ bit 0
                           SG_END     SG_CHAIN

  bit 0 (SG_CHAIN): page_link 不是 page，而是指向下一个 SG 表的指针
  bit 1 (SG_END):   这个 SG 条目是最后一个
```

这样设计的原因：SG 表可能很大，需要分段分配。当 SG 条目超过单次分配上限时，用 `sg_chain()` 把多个 SG 表链接起来。对 `dd bs=4k` 来说，只有一个 SG 条目（`sg_len=1`），不需要链——`SG_END=1`、`SG_CHAIN=0`，`page_link` 就是一个普通的 `struct page *`。

**`dma_address` 是 DMA 控制器看到的总线地址：**

```c
// mmci_stm32_sdmmc.c — IDMA 准备数据
sdmmc_idma_prep_data()
    n_elem = dma_map_sg(dev, data->sg, data->sg_len,
                        mmc_get_dma_dir(data));
    // ↑ 遍历所有 SG 条目，填充 dma_address
    //   可能合并（物理连续时返回更少的条目）
    //   返回映射后的 SG 条目数

// 之后 IDMA 直接读 sg->dma_address 做传输
sg_dma_address(sg)  →  0x9440_0000  (DDR 总线地址)
sg_dma_len(sg)      →  4096         (4KB)
```

CPU 和 DMA 控制器看到的地址不同（MMU 虚拟地址 vs 总线地址），`dma_map_sg()` 就是这两者之间的翻译器。

**一条 SG 的完整生命周期：**

```
① sg_init_table(sg, n)           ← 初始化 SG 表，置 SG_END
② sg_set_page(sg, page, len, 0)  ← 绑定物理页: page_link=page, length=len
     │                            ← 此时 dma_address 还是空的
③ dma_map_sg(dev, sg, n, dir)    ← 填充 dma_address（CPU VA → 总线地址）
     │                            ← 返回映射后的条目数（可能 < n）
④ IDMA 读 sg_dma_address(sg)     ← DMA 控制器用总线地址做传输
     写 IDMA 描述符
⑤ dma_unmap_sg(dev, sg, n, dir)  ← IO 完成，清理
```

和 bio_vec 的区别：

| | bio_vec | scatterlist |
|---|---|---|
| 所属层 | VFS/块层 | 块层/DMA API |
| 寻址方式 | `(struct page*, offset, len)` | `(dma_addr_t, len)` |
| 物理连续合并 | 不合并，一个 folio 对应一个 bvec | **合并相邻物理页**，减少 DMA 段数 |
| 最终用途 | 给内核用（`copy_to_user`） | **给 DMA 控制器用**（`dma_map_sg` 后传输） |

对 `dd bs=4k count=1` 来说，一个 bio 只有一个 bvec（4KB 单页），`blk_rq_map_sg` 输出一条 SG：

```
bio_vec: [ {page=0x3fb8..., len=4096, offset=0} ]
    ↓ blk_rq_map_sg()
sg[]:    [ {page_link=0x3fb8..., offset=0, length=4096, dma_address=0} ]
    ↓ dma_map_sg()  把 struct page* 翻译成总线地址
sg[]:    [ {page_link=0x3fb8..., offset=0, length=4096, dma_address=0x94400000} ]
    ↑                    ↑                              ↑
    |                    DMA 控制器从这里读数据          这是 IDMABASE0R 要写的值
    sg_len = 1

```c
// mmc_blk_data_prep() 续：方向和超时
if (rq_data_dir(req) == READ)
    brq->data.flags = MMC_DATA_READ;
else
    brq->data.flags = MMC_DATA_WRITE;

mmc_set_data_timeout(&brq->data, card);          // 计算超时
```

拿 `dd if=/dev/mmcblk1 bs=4k count=1` 举例，request 和 `mmc_data` 的对应关系：

```
struct request (blk-mq):                struct mmc_data (MMC 核心):
┌──────────────────────────┐             ┌──────────────────────────┐
│ __sector = 0             │─────blk_addr→│ blk_addr = 0            │
│ nr_sectors = 8           │─────blocks → │ blocks = 8              │
│ bio → bio_vec[0]         │─────sg    → │ blksz = 512             │
│   .page = DDR_page_X     │             │ sg[0] = { DDR_page_X,   │
│   .len = 4096            │             │           4096 }         │
│   .offset = 0            │             │ sg_len = 1              │
│ cmd_flags = REQ_OP_READ  │─────flags → │ flags = MMC_DATA_READ   │
└──────────────────────────┘             │ timeout = 300ms         │
                                         └──────────────────────────┘
```

**三个关键转换：**

| request 字段 | mmc_data 字段 | 说明 |
|---|---|---|
| `nr_sectors = 8` | `blocks = 8` | 块层以 512B 扇区计数，MMC 协议也以 512B 块计数。`bs=4k = 8 × 512B` |
| `bio->bi_io_vec[]` | `sg[]` | bio 的数据段 vec 被 `mmc_queue_map_sg()` 转为 scatterlist，供 IDMA 使用 |
| `rq_data_dir(req)` | `MMC_DATA_READ/WRITE` | 读/写方向决定 DMA 方向和响应类型（R1 vs R1B） |

> **CMD23 的作用：** 当 card 支持 CMD23（通过 `ext_csd.ext_csd_cmd_set & EXT_CSD_CMD_SET_NORMAL` 检测），MMC 核心会在 CMD18 之前追加一个 CMD23（SET_BLOCK_COUNT），告诉 eMMC 芯片"接下来要读 N 个块"。不使用 CMD23 时，eMMC 靠 DAT0 线上的 STOP_TRANSMISSION 信号（CMD12）结束多块传输。CMD23 的好处是 eMMC 可以提前知道传输总量，优化内部调度。

**步骤 3：`mmc_set_data_timeout()` —— 超时计算**

根据卡的类型和传输速度，eMMC 核心计算一个合理的超时值：

```c
// drivers/mmc/core/mmc_ops.c — 简化逻辑
mmc_set_data_timeout(data, card)
    // 对 eMMC DDR52 模式 166MHz 8-bit:
    // timeout_clk = card->csd.taac_ns × blocks
    // 实际超时通常设为数百毫秒
```

这个超时值最终写入 host driver 的 `MCI_DATATIMER` 寄存器。如果 DMA 传输在超时前没有完成，控制器产生超时中断。

**步骤 4：`mmc_start_request()` —— 启动传输**

`mmc_start_request()`（`drivers/mmc/core/core.c:168`）做最终检查后，调用 `host->ops->request()` 回调。对于 STM32MP2 就是 `mmci_request()`。

---

> **技术核弹点 R3：SG List 的构建——数据从 bio_vec 到 IDMA 的蜕变**

数据路径中最优雅的设计之一是 Scatter-Gather 列表的逐层转换。

**起点：bio 中的 bvec**

`struct bio` 内部包含一个 `bi_io_vec[]` 数组：

```c
struct bio_vec {    // 每个条目描述一个物理内存段
    struct page *bv_page;   // 物理页框
    unsigned int bv_len;    // 段长度
    unsigned int bv_offset; // 页内偏移
};
```

`bio_add_folio()` 将 folio 的物理页挂载为 bio 的一个 bvec 段。

**第一次转换：blk_rq_map_sg()**

`mmc_queue_map_sg()`（`block.c:1469` → `queue.c:519`）调用 `blk_rq_map_sg()`，将 request 中所有 bio 的 bvec 合并为**连续的 scatterlist**：

```c
// drivers/mmc/core/queue.c — queue.c:519
unsigned int mmc_queue_map_sg(struct mmc_queue *mq, struct mmc_queue_req *mqrq)
{
    struct request *req = mmc_queue_req_to_req(mqrq);
    return blk_rq_map_sg(mq->queue, req, mqrq->sg);
}
```

`blk_rq_map_sg()` 做的重要工作是**合并物理连续的段**。如果多个 bvec 指向的物理页在总线地址上是连续的，它们被合并成一个 SG 条目（`sg_merge` 逻辑）。对 `bs=4k count=1` 的场景，一个 bio 只有一个 bvec，所以结果是单条 SG。

```
bio_vec[0]: {page=P1, len=4096, off=0}  →  sg[0]: {addr=b us_addr(P1), len=4096}
```

**第二次转换：sg 到 IDMA 描述符**

在 host driver 中（断面 1.3），SG 条目被编程为 IDMA 的 LLI 描述符。但 R3 中先理解到 SG 为止——SG 是 MMC 核心层和 host driver 之间的接口协议。

**数据转换链全景：**

```
块层:         struct request
               └── rq->bio → bio->bi_io_vec[] = {page, len, offset}
                             ↓ blk_rq_map_sg()
MMC 核心:      struct scatterlist[]
               └── sg_table.sgl = {dma_addr, dma_len}
                             ↓ dma_map_sg() → sg_dma_address()
Host driver:   IDMA LLI 描述符
               └── sdmmc_lli_desc[] = {src, dst, next, ctrl}
                             ↓ MCI_IDMABASE0
硬件:          SDMMC2 IDMA 控制器
               └── 自动遍历 LLI 链，完成 DMA 传输
```

---

#### 断面 1.2 终点状态

```
struct mmc_request:      已填充完成
  ├─ cmd.opcode = 18 (CMD18 READ_MULTIPLE_BLOCK)
  ├─ cmd.arg = 扇区号 (例如 sector 0)
  ├─ data.blk_addr = 扇区号
  ├─ data.blocks = 8 (bs=4k / 512)
  ├─ data.sg = 已通过 mmc_queue_map_sg 填充
  └─ data.timeout_ns / timeout_clks = 已计算

当前所处位置: 即将通过 host->ops->request() 进入 host driver
下一个状态:   mmci_request() → IDMA 描述符构建 → 寄存器写入 → 硬件启动
```

### 断面 1.3：Host Driver → IDMA → 中断唤醒

如果说前两个断面是我们从软件层逐渐下降，断面 1.3 就是**触及硬件的最后一级**。在这一层，struct mmc_request 被翻译成寄存器写操作，数据通过 IDMA 在总线上流动，最后由一个中断通知"读完了"。

#### 函数追踪

```
mmci_request(host, mrq)                     <-- mmci.c:1878
  ├─ mmci_start_data(host, mrq->data)        <-- mmci.c:1235: 配数据层
  │    ├─ writel(timeout, MMCIDATATIMER)     <-- 设超时
  │    ├─ writel(size, MMCIDATALENGTH)       <-- 设传输大小 (4KB)
  │    └─ mmci_dma_start(host, datactrl)     <-- mmci.c:575
  │         └─ host->ops->dma_start()
  │              = sdmmc_idma_start()        <-- stm32_sdmmc.c:243
  │                ├─ for_each_sg() 遍历 SG 填充 idmalar/idmabase/idmasize
  │                ├─ dma_wmb()              <-- 内存屏障，保证描述符已写入 DDR
  │                ├─ writel(LLI, MMCI_STM32_IDMABAR)       ← 底座地址
  │                ├─ writel(desc[0], MMCI_STM32_IDMALAR)   ← 首描述符 link
  │                ├─ writel(desc[0], MMCI_STM32_IDMABASE0R)← 首描述符 data addr
  │                ├─ writel(desc[0], MMCI_STM32_IDMABSIZER)← 首描述符 size
  │                └─ writel(EN|LLIEN, MMCI_STM32_IDMACTRLR)← 启动 IDMA!
  └─ mmci_start_command(host, mrq->cmd, 0)   <-- mmci.c:1320: 发命令!
       ├─ writel(cmd->arg, MMCIARGUMENT)     <-- 写参数
       └─ writel(cmd|flags, MMCICOMMAND)     <-- 写命令，启动传输!
```

#### 步骤拆解

**步骤 1：`mmci_request()` —— host driver 入口**

`mmci_request()`（`mmci.c:1878`）是 `mmc_host_ops.request` 的实现。它接收来自 MMC 核心层的 `struct mmc_request *mrq`，先准备数据阶段，再发命令：

```c
// mmci.c:1897 — 简化
static void mmci_request(struct mmc_host *mmc, struct mmc_request *mrq)
{
    struct mmci_host *host = mmc_priv(mmc);
    
    host->mrq = mrq;
    
    if (mrq->data)                          // 有数据?
        mmci_start_data(host, mrq->data);   // → 准备 IDMA + 数据寄存器
    
    mmci_start_command(host, mrq->cmd, 0);  // → 写命令寄存器, 启动!
}
```

**为什么数据先于命令？** SDMMC2 控制器的设计是数据层和命令层分离——必须在 CMD 发出去之前把 DMA 描述符配好，因为 eMMC 在收到 CMD18 后立刻开始通过 DAT 线回传数据，如果 IDMA 还没准备好，RX FIFO 会溢出。

**步骤 2：`mmci_start_data()` —— 数据寄存器写入**

`mmci_start_data()`（`mmci.c:1235`）做三件事：

```c
// mmci.c:1235 — 简化
static void mmci_start_data(struct mmci_host *host, struct mmc_data *data)
{
    // 1. 超时寄存器
    timeout = data->timeout_clks + ...;
    writel(timeout, host->base + MMCIDATATIMER);
    
    // 2. 传输大小寄存器
    writel(data->blksz * data->blocks, MMCIDATALENGTH);
    
    // 3. 尝试启动 DMA
    if (!mmci_dma_start(host, datactrl))
        return;    // DMA 启动成功, 走 DMA 路径
    
    // 如果 mmci_dma_start 返回非零 → DMA 启动失败
    // → 回退 PIO 模式 (CPU 逐字节搬运)
    mmci_init_sg(host, data);
}
```

**步骤 3：`sdmmc_idma_start()` —— 构建并启动 IDMA**

这是整条 IO 路径中最贴近硬件的函数（`mmci_stm32_sdmmc.c:243`）。从 SG 构建 LLI 描述符并写入 IDMA 寄存器。

**a) 简单模式（单 SG 段）：**

```c
if (!host->variant->dma_lli || data->sg_len == 1 || ...) {
    // 只有一个 SG 段，不需要链表
    dma_addr = sg_dma_address(data->sg);
    writel_relaxed(dma_addr, MMCI_STM32_IDMABASE0R);     // 数据地址
    writel_relaxed(IDMAEN, MMCI_STM32_IDMACTRLR);         // 启动 IDMA
    return 0;
}
```

对于 `bs=4k count=1` 的 4KB 单页读，正好走这个简单路径——数据在一个连续物理页内，一个 SG 段就够了。

**b) 链表模式（多个 SG 段）：**

```c
// 多个物理段 → 需要 LLI 链表
for_each_sg(data->sg, sg, data->sg_len, i) {
    desc[i].idmalar  = (i + 1) * sizeof(struct sdmmc_lli_desc);
    desc[i].idmalar |= ULA | ULS | ABR;       // 链接属性
    desc[i].idmabase = sg_dma_address(sg);     // DDR 总线地址
    desc[i].idmasize = sg_dma_len(sg);         // 传输字节数
}
desc[data->sg_len - 1].idmalar &= ~ULA;        // 最后一个 → ULA=0 终止

dma_wmb();                                       // 确保描述符写入 DDR

// 写入 IDMA 寄存器组
writel_relaxed(idma->sg_dma,      MMCI_STM32_IDMABAR);   // 描述符基地址
writel_relaxed(desc[0].idmalar,   MMCI_STM32_IDMALAR);   // 首描述符 link
writel_relaxed(desc[0].idmabase,  MMCI_STM32_IDMABASE0R);// 首描述符 data addr
writel_relaxed(desc[0].idmasize,  MMCI_STM32_IDMABSIZER);// 首描述符 size
writel_relaxed(IDMAEN | IDMALLIEN, MMCI_STM32_IDMACTRLR);// 启动 IDMA!
```

**IDMA 硬件自动遍历 LLI 的原理：**

```
IDMA 启动后的硬件行为（无需 CPU 干预）:
  ① 控制器从 IDMABAR 加载描述符基地址
  ② 自动获取 desc[0]: idmabase=DDR_addr_0, idmasize=4096
  ③ 启动 DMA: SDMMC2 RX FIFO → DDR_addr_0
  ④ 传输完成, 检查 desc[0].idmalar 的 ULA 位
  ⑤ ULA=1 (有后继) → 从 idmalar 计算的偏移加载 desc[1]
  ⑥ 重复直到 desc[N-1].idmalar.ULA=0
  ⑦ 置位中断位: MMCI_STM32_IDMAEND
```

---

> **技术核弹点 R4：DMA 一致性——真实的 cache 维护在哪里？**

框架中提到了 `dma_sync_sg_for_device()`，但**实际 STM32MP2 的 mmci driver 中并没有显式调用这个函数**。cache 一致性是通过 `dma_map_sg()` 隐式保证的：

```c
// mmci_stm32_sdmmc.c:137 — sdmmc_idma_prep_data()
static int _sdmmc_idma_prep_data(struct mmci_host *host, struct mmc_data *data)
{
    n_elem = dma_map_sg(mmc_dev(host->mmc),
                        data->sg,
                        data->sg_len,
                        mmc_get_dma_dir(data));   // ← 关键：方向参数
    ...
}
```

`dma_map_sg()` 内部根据方向参数做不同的 cache 操作：

| 方向 | dma_map_sg 内部行为 |
|------|-------------------|
| `DMA_FROM_DEVICE`（读） | **invalidate** CPU Cache 中对应地址的 cacheline，防止读到 stale 数据 |
| `DMA_TO_DEVICE`（写） | **clean + invalidate**，将脏 cacheline 写回 DDR，让 IDMA 看到最新数据 |
| `DMA_BIDIRECTIONAL` | clean + invalidate 都做 |

**这个设计是一般性原理，不是 STM32MP2 特有的。** 无论哪个平台，只要 CPU 有 Cache，DMA 传输就需要 cache 一致性维护。差异只在于：
- **无硬件一致性**（STM32MP2）：`dma_map_sg` 在每次 DMA 时做软件 cache 维护
- **有硬件一致性**（如 IO-coherent 总线）：`dma_map_sg` 可能是个空操作（但 API 层保证了兼容性）

> **调试启示：** 如果在 eMMC 读测试中看到"数据有时候是错的，有时候是对的"——先怀疑 DMA cache 一致性问题。常见原因：bounce buffer 路径的 `sg_copy_from_buffer` 漏了 `dma_wmb()`，或者 `dma_unmap_sg` 的时机不对。

---

**步骤 4：`mmci_start_command()` —— 触发传输**

```c
// mmci.c:1320
writel(cmd->arg, base + MMCIARGUMENT);    // 先写参数
// cmd->opcode = 18, flags 包含响应类型等
c = cmd->opcode | MCI_CPSM_ENABLE;
writel(c, base + MMCICOMMAND);             // 写命令 → CMD 线上立刻发送!
```

`MMCICOMMAND` 寄存器写入是真正的"发令枪"。示波器上看，这一瞬间 CMD 线从高阻变成有toggle。对 CMD18，eMMC 响应后随即在 DAT 线上回传数据，IDMA 开始搬运到 DDR。

**步骤 5：`mmci_irq()` —— 中断处理**

当 IDMA 完成所有数据传输后，中断触发：

```c
// mmci.c — mmci_irq() 简化
static irqreturn_t mmci_irq(int irq, void *dev_id)
{
    status = readl(host->base + MCI_STATUS);
    status &= readl(host->base + MCI_MASK0);   // 只处理已使能的位
    
    // 数据完成 → 处理 data_end
    if (status & MCI_DATAEND)
        mmci_data_end(host, ...);
    
    // IDMA 传输完成
    if (status & MCI_IDMAEND)
        ...;  // 通常在 sdmmc_idma_start 中注册回调
    
    // 命令响应到达
    if (status & MCI_CMDRESPEND || status & MCI_CMDSENT)
        mmci_cmd_irq(host, ...);
}
```

中断发生后，最终通过 `mmc_request_done()` 逐级回调到 `folio_unlock()`：

```
mmci_irq()
  └→ mmci_data_end()
      └→ mmc_request_done(host->mmc, host->mrq)  <-- core.c:223
          └→ mmc_blk_mq_complete()                 <-- 通知 blk-mq
              └→ blk_mq_complete_request(req)       <-- request 完成
                  └→ blk_update_request(req)
                      └→ bio_endio(bio)              <-- bio 完成
                          └→ folio_unlock(folio)     <-- 解锁 folio!
                              └→ wake_up(&folio->wait) ← 唤醒 dd 进程
```

**步骤 6：进程唤醒**

`folio_unlock()` 是 IO 完成和进程唤醒的衔接点：

```c
// folio_unlock() 内部:
// 1. clear_bit(PG_locked, &folio->flags)  — 清除锁标志
// 2. smp_mb__after_atomic()               — 内存屏障
// 3. wake_up_page(folio, PG_locked)        — 唤醒等待队列中的进程
```

之前 `filemap_read()` 中因为 folio 被锁而调用 `folio_wait_locked()` 睡眠的 `dd` 进程，此刻被唤醒：

```c
// filemap_read() 中 — 简化
filemap_read(file, iter, ...)
    // ... 前面 submit_bio 已经提交 ...
    folio_wait_locked(folio);           // ← 之前在这里睡了
    // ← 现在被唤醒了！
    
    if (folio_test_uptodate(folio)) {
        copy_page_to_iter(folio, offset, bytes, iter);  // → 用户空间
    }
```

`copy_page_to_iter()` 将内核 Page Cache 中的 4KB 数据拷贝到 `dd` 进程的用户空间缓冲区。`dd` 拿到数据，write() 到 `/dev/null`，**一次完整的 IO 完成**。

---

> **技术核弹点 R5：中断上下文中的"完成回调链"**

从 `mmci_irq()` 到 `folio_unlock()`，是一个跨越多个子系统的函数回调链：

```
中断上下文 (mmci_irq)
  │  mmc_request_done()    → 通知 MMC 核心：这个 request 完成了
  ▼
mmc_blk_mq_complete()      → 通知 blk-mq：生命周期结束
  │
  ▼
blk_mq_complete_request()  → 触发 completion 回调
  │
  ▼
blk_update_request()       → 遍历所有 bio，逐个标记完成
  │
  ▼
bio_endio()                → 通知具体 bio
  │
  ▼
folio_unlock()             → 解锁 folio，唤醒 dd 进程
  │
  ▼
dd 进程被唤醒 → copy_to_user → read() 返回
```

这个链的设计模式是**完成回调链**——中断处理函数不直接唤醒进程，而是一层层回调，让每层执行自己的完成逻辑（清理状态、统计、释放）。**中断上下文只做最快的通知操作。**

---

#### 断面 1.3 终点状态

```
进程:       dd (PID xxx), 已从 folio_wait_locked 唤醒 → R (running)
数据:       Page Cache 中 folio 包含 sector 0 的 4KB 数据
            → copy_page_to_iter 到用户缓冲区
硬件:       IDMA 已完成，MCI_STATUS 中 DATAEND 置位
中断:       已完成，返回 IRQ_HANDLED
```

---

> **第一幕总结：读路径全景**

```
dd if=/dev/mmcblk1 bs=4k count=1
  │
  ├─ ① VFS:     filemap_read → filemap_get_pages → 缺页 → blkdev_read_folio → submit_bio
  │              进程在 folio_wait_locked 上睡眠
  ├─ ② blk-mq:  submit_bio → blk_mq_submit_bio → mmc_mq_queue_rq → mmc_blk_mq_issue_rq
  │              tag 分配, mq-deadline 合并, mmc_get_card 获 host 锁
  ├─ ③ MMC Core: mmc_blk_rw_rq_prep → CMD18 + SG → mmc_start_request
  │              bio_vec → SG 转换
  ├─ ④ Host:    mmci_request → mmci_start_data → sdmmc_idma_start
  │              IDMA LLI → 寄存器 → mmci_start_command → 写 MMCICOMMAND
  │    ↓
  ├─ ⚡ 硬件:   CMD18 → eMMC 响应 → DAT 线数据 → IDMA → DDR
  │    ↓
  └─ ⑤ 中断:   mmci_irq → mmc_request_done → ... → folio_unlock → 进程唤醒
```

从用户按下回车到数据返回，经过 4 个子系统、6 个函数调用层级、约 20 个关键函数。但对用户态的 `dd` 来说只是一次 `read()` 系统调用。**这就是内核分层设计的力量——每层只关心自己的抽象，又通过接口协议紧密协作。**

---

## 第二幕：dd write（Buffered IO）—— 异步的"推"数据通路

```bash
dd if=/dev/zero of=/dev/mmcblk1 bs=4k count=256
# 敲完回车时，数据还在 Page Cache 里, 没有到达 eMMC!
```

**行为特征：** 进程快速返回。Buffered Write 的数据路径已在 05 第 5.6 节完整分析。本文从 `submit_bio` 之后的写路径开始——即后台 `wb_workfn` 刷脏页到 blk-mq，再由 MMC 层执行 CMD25 多块写。

**写路径和读路径的核心差异：**

| 维度 | 读路径 (第一幕) | 写路径 (第二幕) |
|------|---------------|---------------|
| 真正的 IO 执行者 | `dd` 进程本身（陷入内核） | `wb_workfn` 内核线程 (后台) |
| CMD | CMD18 READ_MULTIPLE_BLOCK | CMD25 WRITE_MULTIPLE_BLOCK |
| 响应类型 | R1 (无 BUSY) | R1B (带 BUSY，eMMC 写 NAND 期间拉低 DAT0) |
| DMA 方向 | eMMC → DDR | DDR → eMMC |
| Cache 一致性 | DMA 前 invalidate Cache | DMA 前 clean Cache |

### 断面 2.1：后台回写（wb_workfn）—— 写路径的起始点

写缓存在 05 第 5.6.2 节已分析（`iomap_write_iter` → `__folio_mark_dirty`）。真正的 eMMC 写操作——后台刷盘——从这里才开始。

#### 触发条件

wb_workfn 在以下四个条件之一满足时执行：

| 触发条件 | 参数 | 场景示例 |
|---------|------|---------|
| 脏页比例超阈值 | `dirty_background_ratio=10%` | 大量连续写，DDR 中脏页 > 总内存的 10% |
| 脏页超时 | `dirty_expire_centisecs=3000` (30s) | 少量写但隔了 30 秒，内核认为太久没刷了 |
| 显式 sync/fsync | 用户调用 `sync` | 用户敲 sync |
| 内存压力 | `kswapd` 回收页 | 系统内存不足，需要把脏页刷出以释放 Page Cache |

对 `dd bs=4k count=256` 写入 1MB 数据：如果系统内存充足（>1GB），脏页比例不足以触发阈值（1MB / 总内存 < 10%），所以通常是**脏页超时**触发回写。

#### 函数追踪

```c
// 内核线程入口
wb_workfn(wb)                              // fs/fs-writeback.c
  └─ wb_do_writeback(wb)
       └─ wb_writeback(wb, &work)
            └─ __writeback_inodes_wb(wb, work)
                 └─ writeback_sb_inodes(sb, wb)
                      └─ __writeback_single_inode(inode, wb, work)
                           └─ do_writepages(mapping, wbc)
                                └─ mapping->a_ops->writepages()
                                     = ext4_writepages()       <-- 文件系统写回
                                       └─ ext4_io_submit()
                                            └─ submit_bio(wbc) <-- 提交到块层！
                                                ← 从这里开始和读路径汇合！
```

**关键认知：** `submit_bio()` 之后，Buffered write 的刷盘路径和 read 路径在 bio 层面之后是完全相同的代码路径！差异只在：

| 层面 | 读路径 | 写路径 |
|------|-------|-------|
| CMD | CMD18 | CMD25 |
| 数据方向 | eMMC → DDR | DDR → eMMC |
| 响应类型 | R1 | R1B |
| DMA 方向 | DMA_FROM_DEVICE | DMA_TO_DEVICE |
| DMA 一致性 | invalidate Cache | clean Cache |

所以断面 2.2 我们只聚焦写路径独有的差异点，相同部分（blk-mq → mmc core → IDMA）不再展开。

---

#### 断面 2.1 终点状态

```
执行者:       wb_workfn (内核线程, PID 可变)
脏页:        256 个 folio 正被逐批处理 → 构造 bio → submit_bio
bio:         已提交到块层，和读路径同路径
下一个状态:   blk_mq_submit_bio → ... → mmc_blk_mq_issue_rq (CMD25)
```

### 断面 2.2：CMD25 多块写入 + R1B 处理

写路径在 MMC 核心层和 host driver 层和读路径汇合，但有两个关键差异：**CMD 不同**和**响应类型不同**。

#### CMD25 多块写

`mmc_blk_rw_rq_prep()` 中的 CMD 选择逻辑（同读路径的 `block.c:1672`）：

```c
if (brq->data.blocks > 1) {
    readcmd = MMC_READ_MULTIPLE_BLOCK;     // CMD18
    writecmd = MMC_WRITE_MULTIPLE_BLOCK;   // CMD25
} else {
    readcmd = MMC_READ_SINGLE_BLOCK;       // CMD17
    writecmd = MMC_WRITE_BLOCK;            // CMD24
}

brq->cmd.opcode = (rq_data_dir(req) == READ) ? readcmd : writecmd;
```

对写路径，`rq_data_dir(req) == WRITE` → `brq->cmd.opcode = MMC_WRITE_MULTIPLE_BLOCK`（CMD25，多块写）或 `MMC_WRITE_BLOCK`（CMD24，单块写）。

**CMD25 和 CMD18 的区别仅在于数据方向和 STOP 命令：**

```
CMD18 读:  主机发 CMD18 → eMMC 开始在 DAT 线上输出数据
           主机读 DAT 线 → IDMA 写入 DDR
           所有块传输完 → 主机发 CMD12 (STOP_TRANSMISSION)

CMD25 写:  主机发 CMD25 + 数据 → eMMC 接收并写入内部缓存
           IDMA 从 DDR 读取数据 → 发送到 DAT 线
           所有块传输完 → 主机关闭数据输出（自动）
```

#### R1B 响应的特殊性

写入操作（尤其是最后一块写入后）的响应中包含 **BUSY 信号**：

```
CMD25 写操作的时序:
  主机:  ┌── CMD25 ──┬─── DATA (DDR→eMMC) ───┬──┐
                                       DAT0:    │▄▄▄▄▄▄▄│  ← BUSY!
                                               ↑
                          eMMC 正在将数据从 SRAM Cache 写入 NAND
                          此时 DAT0 被拉低，指示主机不要发新命令
                          BUSY 时间: 典型 10-100ms，最长可达 500ms
```

R1B 响应和普通 R1 的区别在于 BUSY 信号的处理：

```c
// 读: flags = MMC_RSP_R1 (无 BUSY)
brq->stop.flags = MMC_RSP_SPI_R1 | MMC_RSP_R1 | MMC_CMD_AC;

// 写: flags = MMC_RSP_R1B (带 BUSY)
brq->stop.flags = MMC_RSP_SPI_R1B | MMC_RSP_R1B | MMC_CMD_AC;
```

> **技术核弹点 W2：STM32MP2 的硬件忙检测 — busy_detect**

如果主机在 `mmc_wait_for_req()` 中用 `while(readl(status) & BUSY)` 死等 DAT0 被拉高，两个问题：
1. CPU 100% 空转烧功耗
2. 单核系统上其他请求拿不到 `host->claimed` 锁

STM32MP2 通过 `variant_data.busy_detect = true` 使能硬件忙检测：

```
主机发完 CMD25 + DATA 后:
  ① sdmmc_set_busy_detect() 配置 DLYB 单元的忙检测逻辑
  ② CPU 可以调度/睡眠 (其他进程可以用 CPU)
  ③ eMMC 内部写完 NAND → 拉高 DAT0
  ④ SDMMC2 控制器检测到 DAT0 升沿 → 产生 MCI_BUSYD0END 中断
  ⑤ mmci_irq() 中处理 BUSY 中断 → mmc_request_done()
  ⑥ wb_workfn 被唤醒 → 继续处理下一批脏页
```

**没有 busy_detect 支持的主机**：MMC 核心层用 `mmc_poll_for_busy()` 轮询 DAT0，行为由 `cmd.busy_timeout` 控制。超时未完成走错误恢复路径。

#### DMA 方向的改变

写路径的 IDMA 设置和读路径的区别在 `sdmmc_idma_start()` 的数据方向参数：

```c
// mmc_get_dma_dir(data) 根据 data->flags 返回方向
// MMC_DATA_READ  → DMA_FROM_DEVICE   — invalidate Cache
// MMC_DATA_WRITE → DMA_TO_DEVICE      — clean Cache

// 在 sdmmc_idma_prep_data() 中:
n_elem = dma_map_sg(mmc_dev(host->mmc), data->sg, data->sg_len,
                    mmc_get_dma_dir(data));   // ← 方向决定 cache 操作
```

写路径的 `dma_map_sg(DMA_TO_DEVICE)` 在 cache 维护上做的事情是 **clean**（将脏 cacheline 写回 DDR），而不是 invalidate：

```
读 (DMA_FROM_DEVICE):  invalidate cacheline  → 防止 CPU 读 stale 数据
写 (DMA_TO_DEVICE):     clean cacheline       → 将 CPU 最新数据刷到 DDR，让 IDMA 能看到
```

---

#### 断面 2.2 终点状态

```
写路径状态:  CMD25 + DATA 传输完成
            R1B 响应已处理 (BUSY 结束 or 硬件忙检测中断已收到)
DMA:        dma_unmap_sg(DMA_TO_DEVICE) 已调用 (clean+unmap)
进程:       wb_workfn 继续处理下一批脏页，或睡眠等待新脏页
eMMC:       数据已从 eMMC SRAM Cache 写入 NAND (如果是最后一块)
            (或者数据还在 eMMC 的 Cache 中 — 掉电会丢)
```

> **第二幕小结**
>
> 写路径和读路径共享 blk-mq → MMC core → Host → IDMA 的路径，但差异点正好反映了"读写不对称性"：
>
> - **CMD 不同**：CMD25 写 vs CMD18 读
> - **响应不同**：R1B (带 BUSY) vs R1 (无 BUSY)
> - **DMA 方向反**：CPU→eMMC vs eMMC→CPU
> - **进程模型不同**：后台线程异步刷 vs 进程同步等
> - **DMA 一致性操作不同**：clean vs invalidate

---

---

## 第三幕：sync —— 终极清算与硬件落盘

```bash
sync
# 此时数据才真正到达 eMMC NAND 颗粒
# (确切地说: sync 保证数据被推到 eMMC 芯片, 
#  但如果 eMMC Cache 开启, 数据可能还在 eMMC 内部的 SRAM 里)
```

**行为特征：** 进程阻塞等待，直到全部脏页刷出 + 文件系统 journal 提交 + 块设备 FLUSH 完成。

`sync` 的前两步（刷脏页、提交 journal）属于 VFS 和文件系统层，已在 05 的 `generic_write_sync` 节分析。本文从第三步开始——`blkdev_issue_flush()` 下发 `REQ_PREFLUSH` 之后的 MMC 层路径。

---

### 断面 3.1：REQ_PREFLUSH 在 MMC 层的翻译

`blkdev_issue_flush()` 构造一个**没有数据的 bio**（只有 `REQ_OP_FLUSH` 标志位），提交到块层：

```c
// block/blk-flash.c — 简化
blkdev_issue_flush(bdev)
    bio = bio_alloc(bdev, 0, REQ_OP_FLUSH, GFP_KERNEL);
    // ↑ 注意: nr_vecs = 0, 没有数据页!
    submit_bio(bio);
    // 进程在这里等待 bio 完成
```

这个空的 bio 经过 blk-mq 分发后，到达 `mmc_blk_mq_issue_rq()`：

```c
// block.c — mmc_blk_mq_issue_rq() 简化
switch (req_op(req)) {
case REQ_OP_READ:
case REQ_OP_WRITE:
    return mmc_blk_issue_rw_rq(mq, req);
case REQ_OP_FLUSH:                     // ← sync 的 bio 走这里
    return mmc_blk_issue_flush(mq, req);
}
```

`mmc_blk_issue_flush()` 中调用 `mmc_set_flush_flag()` 标记此请求需要 flush，然后调用 `mmc_blk_issue_rw_rq()`——这意味着**FLUSH 请求在 MMC 层被当作一次特殊的"读写请求"处理**。

#### 函数追踪

```
mmc_blk_issue_flush(mq, req)              <-- block.c:637
  └─ _mmc_flush_cache(card)               <-- mmc.c:2087
       └─ mmc_switch(card, EXT_CSD_CMD_SET_NORMAL,
                      EXT_CSD_FLUSH_CACHE,     ← 32 (byte)
                      1,                       ← 1 = FLUSH
                      CACHE_FLUSH_TIMEOUT_MS)  ← 超时 (30s, mmc.c:33)
            └─ mmc_poll_for_busy()         <-- 等 DAT0 释放
```

调用 `_mmc_flush_cache()` 之前会做两个检查：

```c
// mmc.c:2087 — 简化
static int _mmc_flush_cache(struct mmc_host *host)
{
    struct mmc_card *card = host->card;
    int err = 0;
    
    // 检查 1: 卡是否支持 Cache 功能
    if (!(card->ext_csd.cache_size > 0))
        return err;
    
    // 检查 2: Cache 是否已启用
    if (!(card->ext_csd.cache_ctrl & 1))
        return err;
    
    return mmc_switch(card, EXT_CSD_CMD_SET_NORMAL,
                      EXT_CSD_FLUSH_CACHE,   // byte 32
                      1,                      // value = FLUSH
                      CACHE_FLUSH_TIMEOUT_MS); // 1000ms 超时
}
```

如果 eMMC 不支持 Cache（`cache_size == 0`）或 Cache 未使能（`cache_ctrl & 1 == 0`），`_mmc_flush_cache()` 直接返回——**是的，如果 eMMC Cache 没开，`sync` 的第三步（FLUSH）什么都不做即成功返回**。

---

---

### 断面 3.2：CMD6 写 EXT_CSD[32] —— eMMC 硬件缓存的"清算"

当 `_mmc_flush_cache()` 真的调用 `mmc_switch()` 时，它发出一条 **CMD6（SWITCH）** 命令，写 EXT_CSD 寄存器的 byte 32。

#### CMD6 协议格式

```
CMD6 写 EXT_CSD 的格式:
  [31:26] 0           — 保留
  [25:24] 0x3         — CMD_SET (NORMAL)
  [23:16] 0x20 (32)   — INDEX (EXT_CSD_FLUSH_CACHE 的字节号)
  [15:8]  0x01        — VALUE (1 = FLUSH)
  [7:3]   0           — 保留
  [2:0]   0x7         — CMD_SET 掩码 (访问 NORMAL 命令集)
```

#### eMMC 内部发生了什么

```
sync 进程                                          eMMC 芯片
  │                                                    │
  ├─ CMD6 ──────────────────────────────────────────→   │
  │   SWITCH(byte=32, value=1)                        │
  │                                                    │
  │                                                    ├─ 检查 Cache 使能
  │                                                    ├─ 开始将 SRAM Cache 刷入 NAND
  │                                                    │
  │ ← DAT0 拉低 (BUSY) ────────────────────────────   │
  │   (进程在 mmc_poll_for_busy 中等待)                  │
  │                                                    │
  │                                                    ├─ 逐个刷 Cache 页到 NAND
  │                                                    ├─ 典型耗时 10-100ms
  │                                                    │   (Cache 脏页多时可达 500ms+)
  │                                                    │
  │ ← DAT0 拉高 ────────────────────────────────────   │
  │   (BUSY 结束)                                      │
  │                                                    │  ✓ 数据已进 NAND
  │                                                    │
  ├─ CMD6 完成 ────────────────────────────────────→   │
  └─ sync 返回用户态                                    │
```

**CMD6 FLUSH_CACHE 和普通 CMD6 的区别：** FLUSH_CACHE 不改变任何 EXT_CSD 寄存器的值。它只是一个"触发"命令——告诉 eMMC 执行缓存刷出操作。寄存器 byte 32（CACHE_CTRL）的值在 FLUSH 完成后不变（bit 0 仍然为 1 表示 Cache 使能）。**FLUSH 不是禁用 Cache，而是让 eMMC 把 Cache 中的脏数据倒出来。**

---

> **技术核弹点 S2：软件和硬件两层缓存的对比**

```
                  Page Cache                          eMMC Cache
  位置:           DDR (系统内存)                      eMMC 芯片内部 SRAM
  大小:           动态 (默认 ~10% DDR)                固定 (通常 8-64MB)
  管理:           内核 wb_workfn                       eMMC 内部固件
  刷出命令:        `sync` 系统调用                      CMD6 FLUSH_CACHE
  掉电安全:        数据在 DDR 中, 掉电即丢               数据在 eMMC 芯片内, 刷出前掉电也丢
  对用户可见:      `/proc/meminfo` 中的 Dirty 字段       不可见 (eMMC 固件管理)
```

---

---

### 断面 3.3：FUA（Force Unit Access）—— 单次 IO 级别的落盘保证

FUA 是比 `sync` 更细粒度的落盘机制。它不是全局清算，而是单个 IO 的"写穿透"。

#### FUA 和 sync 的对比

```
场景:                   sync                           FUA
                ──────────────────────        ──────────────────────
粒度:           全局                         单次 IO
触发方式:       用户显式调用                  文件系统在构造 bio 时标记
实现机制:       REQ_OP_FLUSH + CMD6           REQ_FUA 标志
落盘保证:       所有页面全量清算              只有这一个 bio 的数据保证落盘
性能影响:       大 (全量刷)                   小 (只刷单次)
典型用户:       手动操作                       SQLite WAL 模式、ext4 journal
```

#### FUA 在 MMC 中的实现

文件系统在构造 bio 时检查是否需要在最后追加 FLUSH：

```c
// ext4 写回时 — 如果 inode 标记了 FUA:
bio->bi_opf |= REQ_FUA;

// MMC 块层收到 REQ_FUA 后的处理:
// block.c — mmc_set_flush_flag()
if (req->cmd_flags & REQ_FUA)
    mqrq->brq.data.flags |= MMC_DATA_FUA;
```

但 MMC 对 FUA 的处理不是像 NVMe 那样硬件支持，而是**软件模拟**的——MMC 块驱动在完成写请求后，追加一次 FLUSH：

```c
// block.c — mmc_blk_issue_rw_rq() 中处理 MMC_DATA_FUA
// 在 CMD25 完成后，如果带有 FUA 标记:
//   调用 _mmc_flush_cache() → CMD6 FLUSH_CACHE
//   等 BUSY 结束 → 返回 blk-mq 完成
```

这就是 SQLite WAL 模式的安全保证：每条 WAL 日志写入时带 FUA 标记，写入完成后 eMMC Cache 被刷出，日志到达 NAND。即使下一秒断电，WAL 日志可用。

---

> **第三幕小结**
>
> `sync` 不是终点，CMD6 FLUSH_CACHE 才是。理解"三幕"的关键在于认识到数据在每个层级都有缓存：
>
> 1. **Page Cache**（DDR）—— 最快，但掉电丢
> 2. **eMMC SRAM Cache**（芯片内）—— 次快，掉电也丢
> 3. **NAND Array**（芯片内）—— 最慢，掉电不丢
>
> `sync` 清第 1 层，`CMD6 FLUSH_CACHE` 清第 2 层。**没有 FLUSH 的 sync 是不完整的 sync。**

---

## 全篇总结：三幕数据流总图

```
操作:          dd read                    dd write                     sync
          ──────────────────      ──────────────────      ─────────────────────
用户态:      dd if=/dev/mmcblk1        dd of=/mnt/test          sync
               │                          │                       │
VFS + Cache    page cache miss            page cache hit          writeback
               │   (缺页)                  │   (标记脏页)           脏页全量刷出
               │   alloc folio             │   快速返回              │
               ▼                           ▼                       ▼
块层:        submit_bio                 submit_bio              submit_bio(FLUSH)
               │                          │                       │
blk-mq        mmc_mq_queue_rq            mmc_mq_queue_rq         mmc_mq_queue_rq
               │   tag分配                 │   tag分配              │  REQ_PREFLUSH
               │   mmc_get_card 锁         │   mmc_get_card 锁      │
               ▼                           ▼                       ▼
MMC Core:    CMD18 多块读               CMD25 多块写            CMD6 FLUSH_CACHE
               │   SG 构建                │   R1B 响应             │  EXT_CSD[32]=1
               │   CMD23 块数              │   busy_detect          │  DAT0 BUSY
               ▼                           ▼                       ▼
Host:        mmci_request                mmci_request            mmci_request
               │   IDMA setup(读方向)      │   IDMA setup(写方向)    │  (无数据, 仅命令)
               │   dma_map_sg(FROM_DEV)    │   dma_map_sg(TO_DEV)   │
               ▼                           ▼                       ▼
硬件:        SDMMC2 RX → DDR            DDR → SDMMC2 TX         CMD6 FLUSH
               │                          │                       │
中断:        DATAEND → folio_unlock      DATAEND → BUSY end      BUSYEND → CMD6 ok
               │                          │                       │
完成:        copy_to_user → dd 拿到数据  wb_workfn 继续/睡眠    sync 返回用户态
```

### 全篇核心结论

1. **读路径是"同步拉"**：进程缺页→等待 IO→数据到达→返回，进程在整个过程中睡眠在 `folio_wait_locked`。

2. **写路径是"异步推"**：数据先到 Page Cache，write() 立即返回，后台 `wb_workfn` 线程慢慢刷盘。写操作真正的 IO 由后台线程完成。

3. **`sync` 是"软硬双层清算"**：软件层清 Page Cache，硬件层通过 CMD6 FLUSH_CACHE 清 eMMC Cache。仅有 `sync` 不保证数据进 NAND，需要 FLUSH 完成才安全。

4. **SG → IDMA 的映射**是数据路径中最优雅的设计——块层的分散物理页通过 LLI 链串联，硬件无需 CPU 干预自动遍历。STM32MP2 的 IDMA 支持 128 条 LLI。

5. **blk-mq 在 MMC 上是"伪多队列"**：per-CPU 无锁提交 + 单硬件串行执行，`mmc_get_card/host->claimed` 是真正的流量闸门。但 tag 系统的背压（backpressure）机制仍带来了并发安全。

6. **DMA 一致性是不可跳过的数据安全屏障**：`dma_map_sg(DMA_FROM_DEVICE)` 隐式 invalidate CPU Cache 防止读到 stale 数据；`dma_map_sg(DMA_TO_DEVICE)` 隐式 clean Cache 让 IDMA 看到最新数据。如果 cache 维护遗漏，会出现"数据时对时错"的随机性 bug。

7. **掉电安全的完整链**：`sync` 不保证数据落 NAND（eMMC Cache 开启时），需要 `CMD6 FLUSH_CACHE` 确认才算真正落盘。FUA 提供单次 IO 级别的精准落盘，SQLite WAL 模式依赖此保证。

### 本文涉及的关键代码文件一览

| 文件 | 作用 | 章节 |
|------|------|------|
| `fs/read_write.c` | 系统调用入口 (read/write) | 1.1, 2.1 |
| `mm/filemap.c` | Page Cache 管理、缺页处理 | 1.1, 2.1 |
| `block/fops.c` | 块设备 address_space_ops | 1.1 |
| `block/blk-core.c` | submit_bio 入口 | 1.2 |
| `block/blk-mq.c` | blk-mq 框架 (tag、直通提交) | 1.2 |
| `block/elevator.c` | IO 调度器选择和合并 | 1.2 |
| `drivers/mmc/core/queue.c` | MMC 块层队列管理、mmc_mq_queue_rq | 1.2 |
| `drivers/mmc/core/block.c` | MMC 块设备请求处理、CMD 翻译 | 1.3, 2.3, 3.2 |
| `drivers/mmc/core/core.c` | MMC 核心层请求分发、mmc_start_request | 1.3 |
| `drivers/mmc/core/mmc.c` | eMMC 卡功能 (_mmc_flush_cache) | 3.2 |
| `drivers/mmc/core/mmc_ops.c` | CMD6、CMD 封装 | 3.3 |
| `drivers/mmc/host/mmci.c` | host driver 主体 (mmci_request、中断) | 1.4 |
| `drivers/mmc/host/mmci_stm32_sdmmc.c` | STM32MP2 IDMA 实现 | 1.4, 2.3 |
| `fs/fs-writeback.c` | 后台脏页回写 (wb_workfn) | 2.2 |
| `mm/iomap.c` | 文件系统写迭代器 (iomap_write_iter) | 2.1 |
| `fs/sync.c` | sync 系统调用 | 3.1 |
| `block/blk-flush.c` | FLUSH/FUA 支持 | 3.2 |

---

> **下一篇：** [07-Advanced.md](07-Advanced.md) — CQHCI 与 Tuning（硬件命令队列和高速模式信号完整性）。CQHCI 是 eMMC 5.1 的硬件命令队列引擎，通过门铃模型让 eMMC 自主调度多笔请求。Tuning 是 HS200/HS400 高速模式必需的调相机制，通过 CMD21 扫描采样窗口。如果你用的是 DDR52 以下模式，Tuning 用不上；但理解 CQHCI 可以帮你从架构层面理解 MMC 子系统的极限。
