# 05. VFS 与块层通路：从系统调用到 submit_bio

> 本文是 STM32MP257 eMMC 驱动深度分析系列的第 5 篇。
> 从用户态 `read/write` 到 `submit_bio` 的完整路径——以块设备（`/dev/mmcblk1`）为例，打通 VFS 层到块层的调用链。
>
> **前置:** [04-SourceAnalysis.md](04-SourceAnalysis.md) — MMC 初始化流程、块设备注册、blk-mq 队列创建
> **下一篇:** [06-IO-Path.md](06-IO-Path.md) — 从 `submit_bio` 到 eMMC NAND（MMC 核心层、Host 驱动、IDMA）
>
> **建议阅读时间：** 40–60 分钟

---

## 5.1 为什么需要这篇

用户敲下 `dd if=/dev/mmcblk1` 后，`read()` 是怎么从用户态经过 VFS、文件系统、块层，最终到达 `submit_bio` 的？这是本文要回答的核心问题。

1. `dd if=/dev/mmcblk1` 的 `read()` 在内核中经过哪些函数，才到达 `submit_bio`？
2. VFS 的四大核心对象（`super_block`、`inode`、`dentry`、`file`）在 eMMC 读/写场景中各扮演什么角色？
3. Buffered write 的数据在到达 `submit_bio` 之前经历了什么？

### 与 06 的分工

| 本文（05） | 06-IO-Path.md |
|-----------|--------------|
| 从系统调用到 `submit_bio` | 从 `submit_bio` 到 eMMC NAND |
| VFS + 块层 | MMC 核心层 + Host 驱动 + IDMA |
| `blkdev_read_iter`、`filemap_read`、`filemap_get_pages` | `mmc_mq_queue_rq`、CMD18/25、IDMA 描述符 |
| `__folio_mark_dirty` → 后台回写 | CMD6 FLUSH_CACHE |

---

## 5.2 VFS 架构概览

### 5.2.1 面向驱动开发者的"一切皆文件"

Linux 中"一切皆文件"的含义不是把内核对象变成文件，而是**用统一文件接口（`open/read/write/close`）访问不同类型的内核对象**。对于驱动开发者，这意味着：

```
用户态操作          背后的内核对象          VFS 路由目标
────────────────────────────────────────────────────────────
dd if=/dev/mmcblk1   块设备 (eMMC)         blkdev_open → def_blk_fops
cat /sys/class/...   kobject 属性          sysfs 文件系统
cat /proc/interrupts 内核动态数据          proc 文件系统
```

对本系列来说，块设备 `/dev/mmcblk1` 走的是 `def_blk_fops` 路径——这是本文的核心。

### 5.2.2 VFS 的分层设计

```
用户态:     fd = open("/dev/mmcblk1", O_RDWR)   read(fd, buf, 4096)
              │                                       │
              ▼                                       ▼
VFS:        do_sys_openat2()                      vfs_read()
              │                                       │
              ▼                                       ▼
具体文件系统: blkdev_open()                         blkdev_read_iter()
              │                                       │
              ▼                                       ▼
块层:       blkdev_get_by_dev()                    filemap_read() → submit_bio()
              │                                       │
              ▼                                       ▼
MMC 层:     (04 已分析: mmc_blk_probe)             (06 分析: mmc_mq_queue_rq → ...)
```

用户态看到的 `read()` 在内核中经过四次转发：系统调用入口 → VFS 通用层 → 块设备特定实现 → 块层提交。每层只做自己职责范围内的事。

---

## 5.3 VFS 四大核心对象

VFS 之所以能支持"一切皆文件"，核心在于它定义了四个层次的对象。本节用 `/dev/mmcblk1` 场景重新梳理。

### 5.3.1 super_block —— 数据通路上不参与

`struct super_block` 代表一个挂载的文件系统实例。但对于 **裸块设备访问**（`dd if=/dev/mmcblk1` 不走任何文件系统），super_block **不参与读写数据通路**。

`bd_inode->i_sb` 指向 devtmpfs 的 super_block——因为设备节点 `/dev/mmcblk1` 本身在 devtmpfs 上，但这个 super_block 仅管理节点元数据（权限、属主），对 `read/write` 调用链无影响：

```c
// include/linux/fs.h — 关键字段
struct super_block {
    struct file_system_type *s_type;    // 文件系统类型
    const struct super_operations *s_op; // 超级块操作表
    struct dentry          *s_root;     // 根目录 dentry
    struct list_head       s_inodes;    // 该 FS 所有 inode 链表
    ...
};
```

这就引出了块设备访问的一个关键认知：**裸设备绕过了文件系统层**。`read()` 从 `def_blk_fops` 直接进入块层通用函数（`blkdev_read_iter`），不需要经过 ext4/btrfs 等具体文件系统。当然后面挂载文件系统后（如 rootfs 用 ext4 挂在 eMMC 上），情况就不一样了——那时文件系统会管理数据布局，但那是另一个话题。

### 5.3.2 inode —— 两个 inode 的拼接

块设备访问涉及 **两个不同的 inode**，容易混淆。

**① 设备节点 inode（来自 devtmpfs）** — VFS 路径解析时找到的 inode：

```c
struct inode {
    umode_t          i_mode;      // S_IFBLK → 标识这是个块设备文件
    dev_t            i_rdev;      // 设备号（主/次设备号）
    const struct file_operations  *i_fop;  // → def_blk_fops
    struct address_space          *i_mapping; // 设备节点的 Page Cache（不用）
    struct super_block            *i_sb;      // → devtmpfs 的 super_block
    ...
};
```

当 `open("/dev/mmcblk1")` 时，VFS 通过 devtmpfs 找到这个 inode，从中读取设备号、设置 `f_op = def_blk_fops`。但这个 inode 的 `i_mapping` **不用于数据读写**。

**② bd_inode（来自 bdev_inode）** — 块设备的内部 VFS inode，提供真正的 Page Cache：

```c
// block/bdev.c:33 — bdev_inode 将 bdev 和 inode 打包在同一块内存中
struct bdev_inode {
    struct block_device bdev;      // 块设备
    struct inode        vfs_inode; // ← 这就是 bd_inode 指向的 inode
};

// include/linux/blk_types.h:40 — block_device 中的 bd_inode 指针
struct block_device {
    ...
    struct inode *bd_inode;  // 指向 bdev_inode.vfs_inode
    ...
};
```

`bd_inode` 的 `i_mapping` 是块设备读写真正的 Page Cache 挂载点。从 `bd_inode` 反推 `block_device` 的宏：

```c
// block/bdev.c:43
struct block_device *I_BDEV(struct inode *inode)
{
    return &BDEV_I(inode)->bdev;
    // → container_of(inode, struct bdev_inode, vfs_inode)->bdev
}
```

**VFS 在 open 时把两者拼接起来：**

```
  设备节点 inode (devtmpfs)             bd_inode (bdev_inode)
  ┌──────────────────────┐             ┌──────────────────────┐
  │ i_rdev = 设备号       │             │ i_mapping → Page Cache│
  │ i_fop  = def_blk_fops│             │ i_size  = 设备总扇区数│
  │ i_sb   = devtmpfs    │             └──────────────────────┘
  └─────────┬────────────┘                       │
            │                                     │
            │ blkdev_open() 读取 i_rdev 找到 bdev │
            │ 然后做了一件事：                      │
            │ filp->f_mapping = bdev->bd_inode->i_mapping
            ▼                                     ▼
   ┌─────────────────────────────────────────────────────┐
   │ struct file                                         │
   │  f_inode = 设备节点 inode  ←  VFS 自动赋值           │
   │  f_op    = def_blk_fops   ←  来自设备节点 i_fop     │
   │  f_mapping = bd_inode->i_mapping  ← blkdev_open 替换 │
   └─────────────────────────────────────────────────────┘
```

所以 `bd_inode` **不是**设备节点的 inode。`blkdev_open` 从设备节点 inode 读设备号、取 `f_op`，再从 `bd_inode` 取 `i_mapping` 给 Page Cache 用——两个 inode 各司其职。

### 5.3.3 dentry —— 路径名缓存

`dentry` 缓存路径名到 `inode` 的映射。第一次访问 `/dev/mmcblk1` 后，内核创建三级 dentry 树：

```
dentry: "/"           →  inode(根目录)
  └── dentry: "dev"   →  inode(/dev)       ← 可能跨越挂载点到 devtmpfs
        └── dentry: "mmcblk1" → 设备节点 inode
```

第二次访问时直接从 dentry 缓存中拿到设备节点 inode，零次磁盘 IO。

```c
struct dentry {
    struct dentry *d_parent;      // 父目录 dentry
    struct inode  *d_inode;       // → 设备节点 inode
    const struct qstr d_name;     // "mmcblk1"
    struct list_head d_child;     // 兄弟节点
    struct hlist_head d_children; // 子节点
    ...
};
```

### 5.3.4 file —— dd 进程的"打开会话"

每次 `open("/dev/mmcblk1")` 创建一个新的 `struct file`，记录本次打开的独立状态：

```c
struct file {
    const struct file_operations *f_op;  // → def_blk_fops
    struct inode                 *f_inode; // → 设备节点 inode（来自 dentry）
    struct path                   f_path;  // 包含 dentry + vfsmount
    loff_t                        f_pos;  // 当前读写偏移
    struct address_space          *f_mapping; // → bd_inode->i_mapping（blkdev_open 替换）
    void                          *private_data; // 块设备私有数据
    ...
};
```

对块设备来说，`f_mapping` 在 `blkdev_open()` 中被设为 `bdev->bd_inode->i_mapping`（`block/fops.c:597`）。这个 `f_mapping` 是 `filemap_read()` 和 `filemap_write()` 定位 Page Cache 的入口。

### 5.3.5 四大对象的关联关系图

```
                        ksys_read(fd, ...)
                             │
                      fdget_pos(fd)
                             │
                             ▼
                     ┌───────────────┐
                     │  struct file  │ ← dd 进程打开 /dev/mmcblk1 的会话
                     │───────────────│
                     │ f_op          │──→ def_blk_fops { .read_iter, .write_iter }
                     │ f_inode       │──┐
                     │ f_mapping     │──┤
                     │ f_pos         │  │
                     └───────────────┘  │
                              │         │
                              ▼         ▼
                     ┌───────────────┐
                     │  struct inode │ ← bd_inode（块设备的 VFS 接口）
                     │───────────────│
                     │ i_mode        │ = S_IFBLK
                     │ i_rdev        │ = MKDEV(major, minor)
                     │ i_fop         │ → def_blk_fops
                     │ i_mapping     │──→ address_space（Page Cache 挂载点）
                     │ i_sb          │ → devtmpfs super_block（节点元数据）
                     └───────────────┘
                             │
                             ▼
                     ┌───────────────┐
                     │ address_space │ ← Page Cache 管理器
                     │───────────────│
                     │ host          │ → bd_inode（回指）
                     │ i_pages       │ → xarray（页号 → folio 索引）
                     │ a_ops         │ → blkdev_aops
                     │   .read_folio │ → blkdev_read_folio
                     └───────────────┘
```


---

## 5.4 OPEN 流程：打开 /dev/mmcblk1

open 是 VFS 建立从路径到 inode 再到 file 的绑定的过程。这里只简述块设备相关的关键步骤。

### 5.4.1 路径解析到 bd_inode

用户态 `open("/dev/mmcblk1", O_RDWR)` 进入内核后，`do_sys_openat2()`（`fs/open.c`）做三件事：

```c
fd = get_unused_fd_flags(flags);       // ① 分配 fd 号
struct file *f = do_filp_open(dfd, tmp, &op);  // ② 路径解析 + 创建 file
fd_install(fd, f);                      // ③ 将 file 装入 fd 表
```

第②步 `do_filp_open()` 内部调用 `path_openat()`，其核心是路径解析和 `do_open()`：

```
path_openat()
  ├─ alloc_empty_file()      ← 分配空的 struct file
  ├─ path_init()             ← 设置解析起点（当前根目录 dentry）
  ├─ link_path_walk()        ← 逐段解析 "dev"，找到 /dev 的 inode
  ├─ open_last_lookups()     ← 解析最后分量 "mmcblk1"
  │    → dentry("mmcblk1") 的 d_inode = bd_inode
  │    → bd_inode->i_mode = S_IFBLK
  │    → bd_inode->i_fop   = def_blk_fops
  └─ do_open() → vfs_open() → do_dentry_open()
```

### 5.4.2 do_dentry_open —— 绑定 f_op

`do_dentry_open()`（`fs/open.c`）是关键：它将路径解析得到的 inode 绑定到 file 上。

```c
static int do_dentry_open(struct file *f, int (*open)(struct inode *, struct file *))
{
    struct inode *inode = f->f_path.dentry->d_inode;  // → bd_inode

    f->f_inode = inode;                    // ① 绑定 inode
    f->f_mapping = inode->i_mapping;        // ② 绑定 address_space
    f->f_op = fops_get(inode->i_fop);       // ③ 关键：f_op = def_blk_fops

    if (!open)
        open = f->f_op->open;               // → blkdev_open
    if (open)
        error = open(inode, f);             // ④ 调用 blkdev_open
    ...
}
```

对块设备：第③步 `f_op = def_blk_fops`，第④步 `open = blkdev_open`。

**与字符设备的对比：**

| 步骤 | 字符设备 (chrdev) | 块设备 (blkdev) |
|------|------------------|----------------|
| `inode->i_fop` | `def_chr_fops`（`.open = chrdev_open`） | `def_blk_fops`（`.open = blkdev_open`） |
| `do_dentry_open` 第③步后 `f_op` | `def_chr_fops`（临时） | `def_blk_fops`（**最终**） |
| `f_op->open` 调用 | `chrdev_open()` → 查 `cdev_map` → **替换 `f_op` 为驱动 fops** | `blkdev_open()` → 获取 bdev → **不替换 `f_op`** |
| open 返回后 `f_op` | 驱动注册的 fops（如 `my_gpio_fops`） | `def_blk_fops`（不变） |

**字符设备多一层间接**：因为字符设备驱动种类繁多（成千上万），无法在 VFS 层静态绑定，所以通过 `chrdev_open` 在运行时按设备号查找并替换。**块设备则不然**——`def_blk_fops` 中的 `read_iter` / `write_iter` 指向块层通用函数，块设备特有的读写逻辑在 blk-mq 层（`submit_bio` 之后）处理，不需要替换 `f_op`。

### 5.4.3 blkdev_open —— 获取 block_device

```c
// block/fops.c:569
static int blkdev_open(struct inode *inode, struct file *filp)
{
    struct block_device *bdev;

    bdev = blkdev_get_by_dev(inode->i_rdev, file_to_blk_mode(filp),
                             filp->private_data, NULL);
    // 用设备号查 bdget → bdev 缓存 → 初始化

    filp->f_mapping = bdev->bd_inode->i_mapping;
    // ★ 关键：将 file 的 Page Cache 入口指向块设备的 address_space

    filp->f_wb_err = filemap_sample_wb_err(filp->f_mapping);
    return 0;
}
```

`blkdev_get_by_dev()` 根据 `inode->i_rdev`（设备号）在 `bdev_map` 中找到或创建 `struct block_device`。`blkdev_open` 的最后一步将 `filp->f_mapping` 指向 `bdev->bd_inode->i_mapping`——这个赋值决定了后续 `filemap_read()` 操作的 Page Cache 属于这个块设备。

### 5.4.4 open 终点状态

```
open("/dev/mmcblk1") 返回后:

进程:          dd (PID xxx)
fd:            3（存入 task_struct->files->fdt[3]）
struct file:   f_op   = def_blk_fops
               f_inode    = bd_inode
               f_mapping  = bd_inode->i_mapping  ← Page Cache
               f_pos      = 0
               private_data = NULL

下一步:        用户态调用 read(fd, buf, 4096)
```

---

## 5.5 READ 流程：vfs_read 到 submit_bio

本节从 `dd` 的 `read()` 出发，沿函数调用链逐层分析到 `submit_bio`。这是 06 第一幕的前置知识。

### 5.5.1 全程函数追踪

```
read(fd, buf, 4096)                  [用户态]
│
├─ ksys_read(fd, buf, count)         fs/read_write.c:602
│   ├─ fdget_pos(fd)                 → struct file *
│   └─ vfs_read(file, buf, count, &pos)
│
├─ vfs_read()                        fs/read_write.c:450
│   ├─ rw_verify_area()              权限检查
│   ├─ file->f_op->read?             没有（def_blk_fops 无 .read）
│   └─ new_sync_read()               → 走 .read_iter 路径
│
├─ new_sync_read()                   fs/read_write.c:379
│   ├─ init_sync_kiocb(&kiocb, filp) 包装 IO 控制块
│   ├─ iov_iter_ubuf(&iter, buf)     包装用户 buffer
│   └─ call_read_iter(filp, &kiocb, &iter)  → f_op->read_iter
│
├─ blkdev_read_iter(&kiocb, &iter)   block/fops.c:693
│   ├─ I_BDEV(f_mapping->host)       获取 block_device
│   ├─ Direct IO? → blkdev_direct_IO (旁路)
│   └─ filemap_read(&kiocb, &iter)   ← 主角！
│
├─ filemap_read()                    mm/filemap.c:2643
│   ├─ filp->f_mapping               → bd_inode->i_mapping
│   ├─ filemap_get_pages()           查 Page Cache（详见下节）
│   ├─ copy_page_to_iter()           从 folio 拷贝到用户 buffer
│   └─ 循环直到数据读完
│
├─ filemap_get_pages()               mm/filemap.c:2561
│   ├─ filemap_get_read_batch()      查 xarray（缓存命中检查）
│   ├─ page_cache_sync_readahead()   （可选）预读
│   ├─ filemap_create_folio()        缺页：分配 + 读盘
│   └─ filemap_update_page()         更新 stale folio
│
└─ filemap_create_folio()            mm/filemap.c:2504
    ├─ filemap_alloc_folio()         从伙伴系统分配 4KB 物理页
    ├─ filemap_add_folio()           加入 xarray
    └─ filemap_read_folio()          调 a_ops->read_folio
         └─ mapping->a_ops->read_folio()
              = blkdev_read_folio()
                → iomap_read_folio()
                  → submit_bio()     ★ 进入块层！
```

### 5.5.2 逐层源码分析

#### ksys_read —— fd 到 file（`fs/read_write.c:602`）

```c
ssize_t ksys_read(unsigned int fd, char __user *buf, size_t count)
{
    struct fd f = fdget_pos(fd);         // O(1) 取 file：task->files->fdt[fd]
    loff_t pos, *ppos = file_ppos(f.file); // 取 file->f_pos

    if (ppos) {
        pos = *ppos;                     // 保存当前位置
        ppos = &pos;
    }
    ret = vfs_read(f.file, buf, count, ppos); // → VFS 通用层
    if (ret >= 0 && ppos)
        f.file->f_pos = pos;             // 更新 file->f_pos
    fdput_pos(f);
    return ret;
}
```

`fdget_pos(fd)` 做的是简单的数组索引——进程的 `task_struct->files->fdt` 是一个 `struct file **` 数组，`fd` 就是数组下标。open 时 `fd_install()` 把 `struct file *` 存入这个槽位。

#### vfs_read —— 权限检查 + 分发（`fs/read_write.c:450`）

```c
ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)
{
    ret = rw_verify_area(READ, file, pos, count); // 验证可读、范围合法
    if (ret) return ret;
    if (count > MAX_RW_COUNT) count = MAX_RW_COUNT;

    if (file->f_op->read)                          // .read 优先
        ret = file->f_op->read(file, buf, count, pos);
    else if (file->f_op->read_iter)                // 否则 .read_iter
        ret = new_sync_read(file, buf, count, pos); // ← 块设备
    else
        ret = -EINVAL;
    ...
}
```

`def_blk_fops` 没有 `.read`，但有 `.read_iter = blkdev_read_iter`，所以走 `new_sync_read`。

#### new_sync_read —— 包装 kiocb（`fs/read_write.c:379`）

```c
static ssize_t new_sync_read(struct file *filp, char __user *buf, size_t len, loff_t *ppos)
{
    struct kiocb kiocb;
    struct iov_iter iter;

    init_sync_kiocb(&kiocb, filp);     // 初始化 IO 控制块（关联 filp）
    kiocb.ki_pos = (ppos ? *ppos : 0); // 设置起始偏移
    iov_iter_ubuf(&iter, ITER_DEST, buf, len); // 包装用户空间 buffer
    ret = call_read_iter(filp, &kiocb, &iter);  // → f_op->read_iter()
    if (ppos)
        *ppos = kiocb.ki_pos;                   // 更新 pos
    return ret;
}
```

这里引入的 `struct kiocb` 是内核 IO 控制块，携带文件指针、起始偏移、标志位等信息。`struct iov_iter` 是对用户 buffer 的抽象迭代器，隐藏了用户态地址的访问细节。

#### blkdev_read_iter —— 块设备读（`block/fops.c:693`）

```c
static ssize_t blkdev_read_iter(struct kiocb *iocb, struct iov_iter *to)
{
    struct block_device *bdev = I_BDEV(iocb->ki_filp->f_mapping->host);
    // 调用链: filp → f_mapping → host(bd_inode) → I_BDEV → block_device
    loff_t size = bdev_nr_bytes(bdev);
    loff_t pos = iocb->ki_pos;

    // 检查 pos 是否越界
    if (unlikely(pos + iov_iter_count(to) > size)) {
        if (pos >= size) return 0;       // 已到末尾
        size -= pos;
        shorted = iov_iter_count(to) - size;
        iov_iter_truncate(to, size);     // 截断到设备大小
    }

    count = iov_iter_count(to);
    if (!count) goto reexpand;

    if (iocb->ki_flags & IOCB_DIRECT) {  // Direct IO → 跳过 Page Cache
        ret = kiocb_write_and_wait(iocb, count);
        if (ret < 0) goto reexpand;
        file_accessed(iocb->ki_filp);
        ret = blkdev_direct_IO(iocb, to);
        ...
    }

    ret = filemap_read(iocb, to, ret);   // ← Buffered IO → Page Cache 路径
    ...
}
```

`I_BDEV` 宏（`block/bdev.c:43`）从 `bd_inode` 反推出 `struct block_device`。这是 VFS 层和块设备层的分水岭——`filemap_read` 返回后，数据从 Page Cache 拷贝到用户 buffer；如果缓存未命中，内部触发 `a_ops->read_folio` 构造 `bio`，调用 `submit_bio` 进入块层。

#### filemap_read —— Page Cache 入口（`mm/filemap.c:2643`）

```c
ssize_t filemap_read(struct kiocb *iocb, struct iov_iter *iter,
                     ssize_t already_read)
{
    struct file *filp = iocb->ki_filp;
    struct address_space *mapping = filp->f_mapping; // → bd_inode->i_mapping
    struct inode *inode = mapping->host;              // → bd_inode
    struct folio_batch fbatch;

    folio_batch_init(&fbatch);

    do {
        cond_resched();

        // ★ 核心：从 Page Cache 获取 folio
        error = filemap_get_pages(iocb, iter->count, &fbatch, false);
        if (error < 0)
            break;

        isize = i_size_read(inode);
        // ... 边界检查 ...

        // 将 folio 中的数据拷贝到用户 buffer
        for (i = 0; i < folio_batch_count(&fbatch); i++) {
            struct folio *folio = fbatch.folios[i];
            copied = copy_page_to_iter(folio, offset, bytes, iter);
            // ★ 这里就是数据到达用户态的时刻！
            ...
        }
        ...
    } while (iov_iter_count(iter) && !error);
    ...
}
```

`filemap_read` 的核心是一个 `do {} while()` 循环，每次处理一批 folio。如果 `filemap_get_pages` 返回缓存中的 folio，`copy_page_to_iter` 直接将内核 DDR 中的数据拷贝到用户 buffer——这就是**缓存命中**路径，`dd` 在 1μs 内返回。

如果 `filemap_get_pages` 内部触发了磁盘 IO（缓存未命中），进程会在 `folio_wait_locked` 上睡眠——06 第一幕会详细追踪后续路径。

#### filemap_get_pages —— 命中/未命中判断（`mm/filemap.c:2561`）

```c
static int filemap_get_pages(struct kiocb *iocb, size_t count,
                             struct folio_batch *fbatch, bool need_uptodate)
{
    struct address_space *mapping = filp->f_mapping;
    pgoff_t index = iocb->ki_pos >> PAGE_SHIFT;  // 字节偏移 → 页号

    // ① 第一次查：xarray 中批量取 folio
    filemap_get_read_batch(mapping, index, last_index - 1, fbatch);
    if (!folio_batch_count(fbatch)) {
        // ② 未命中 → 尝试 readahead（推测性预读）
        page_cache_sync_readahead(mapping, ra, filp, index, last_index - index);
        filemap_get_read_batch(mapping, index, last_index - 1, fbatch);
    }
    if (!folio_batch_count(fbatch)) {
        // ③ readahead 后仍为空 → 真缺页，分配新 folio 并读盘
        err = filemap_create_folio(filp, mapping,
                                   iocb->ki_pos >> PAGE_SHIFT, fbatch);
        ...
    }

    // ④ folio 存在但不是 uptodate → 更新数据
    folio = fbatch->folios[...];
    if (!folio_test_uptodate(folio)) {
        err = filemap_update_page(iocb, mapping, count, folio, need_uptodate);
        ...
    }
    return 0;
}
```

三次尝试对应三种性能路径：

| 步骤 | 条件 | 发生什么 | 延迟量级 |
|------|------|---------|---------|
| ① `filemap_get_read_batch` | xarray 查到 folio | **缓存命中**，直接拷贝 | ~100ns |
| ② `page_cache_sync_readahead` | 第一次未命中 | 推测性预读，减小后续缺页概率 | ~100μs（IO） |
| ③ `filemap_create_folio` | readahead 后仍为空 | **真正缺页**：分配页 → 调 `read_folio` → `submit_bio` | ~1ms+（eMMC IO） |

#### filemap_create_folio —— 缺页处理（`mm/filemap.c:2504`）

```c
static int filemap_create_folio(struct file *file,
        struct address_space *mapping, pgoff_t index,
        struct folio_batch *fbatch)
{
    folio = filemap_alloc_folio(mapping_gfp_mask(mapping), 0);
    // ① 从伙伴系统分配一个 4KB 物理页

    filemap_invalidate_lock_shared(mapping);
    error = filemap_add_folio(mapping, folio, index,
                              mapping_gfp_constraint(mapping, GFP_KERNEL));
    // ② 将新 folio 加入 xarray（mapping->i_pages）
    // 如果返回 -EEXIST → folio 已被其他进程加入，标记重试

    error = filemap_read_folio(file, mapping->a_ops->read_folio, folio);
    // ③ ★ 调 read_folio 从设备读数据！
    //    对块设备 = blkdev_read_folio()
    //    → iomap_read_folio() → submit_bio()
    ...
}
```

第③步 `filemap_read_folio` → `mapping->a_ops->read_folio` 对于块设备就是 `blkdev_read_folio`（`block/fops.c`）。它构造 `struct bio`，调用 `submit_bio`——**这就是 VFS 层和块层的分界线**。`submit_bio` 之后，流程进入 blk-mq 框架，由 MMC 块层驱动接管。

### 5.5.3 读路径全过程数据流

```
vfs_read()
  │  file = fdget(fd), f_op = def_blk_fops
  ▼
new_sync_read()
  │  包装 kiocb + iov_iter
  ▼  
blkdev_read_iter()
  │  I_BDEV(f_mapping->host) 获取 bdev
  │  Direct IO? → blkdev_direct_IO
  ▼
filemap_read()
  │  mapping = filp->f_mapping (bd_inode->i_mapping)
  ▼
filemap_get_pages()
  │  index = ki_pos >> PAGE_SHIFT
  ├───① filemap_get_read_batch(mapping->i_pages, index)  → 命中? 拷贝到用户态
  ├───② page_cache_sync_readahead()                        → 预读
  └───③ filemap_create_folio()                              → 缺页!
        │
        ▼
        filemap_read_folio(mapping->a_ops->read_folio)
          = blkdev_read_folio()
            → iomap_read_folio()
              → submit_bio(bio)  ★ 进入块层 → 06 接管
```

---

## 5.6 WRITE 流程：vfs_write 到 submit_bio

写路径分为两种模式：

| 模式 | 数据路径 | write() 返回时机 | 真正 IO 执行者 |
|------|---------|----------------|--------------|
| **Buffered Write**（默认） | 用户 → Page Cache → 后台刷盘 | 数据进入 Page Cache 后立即返回 | `wb_workfn` 内核线程 |
| **Direct Write**（`O_DIRECT`） | 用户 → `submit_bio` → 硬件 | IO 完成后返回 | 进程自己 |

块设备的 Buffered Write 是理解 IO 路径的关键——`dd if=/dev/zero of=/mnt/testfile bs=4k count=256` 几乎瞬间返回的原因不是 eMMC 写得快，而是数据还没到 eMMC。

### 5.6.1 vfs_write 到 blkdev_write_iter

写路径的前半段（从系统调用到块设备层）和读路径对称。

#### 函数追踪

```
write(fd, buf, count)                [用户态]
│
├─ ksys_write(fd, buf, count)        fs/read_write.c
│   └─ vfs_write(file, buf, count, &pos)
│
├─ vfs_write()                       fs/read_write.c
│   ├─ rw_verify_area(WRITE, ...)    权限检查
│   └─ new_sync_write()              → .write_iter 路径
│
├─ new_sync_write()                  fs/read_write.c
│   └─ call_write_iter() → f_op->write_iter
│
├─ blkdev_write_iter(&kiocb, &iter)  block/fops.c:644
│   ├─ I_BDEV(file->f_mapping->host) 获取 block_device
│   ├─ bdev_read_only()? → -EPERM
│   ├─ Direct IO? → blkdev_direct_write(iocb, from)
│   └─ Buffered → blkdev_buffered_write(iocb, from)  ← Buffered 路径
│
└─ blkdev_buffered_write()           block/fops.c:632
    └─ iomap_file_buffered_write(iocb, from, &blkdev_iomap_ops)
         → iomap_write_iter()
           → iomap_write_begin() + copy_from_iter() + __folio_mark_dirty()
```

前三个函数（`ksys_write`、`vfs_write`、`new_sync_write`）与读路径对称，不再重复。关键差异在 `blkdev_write_iter` 之后。

#### blkdev_write_iter —— 写路径分发（`block/fops.c:644`）

```c
static ssize_t blkdev_write_iter(struct kiocb *iocb, struct iov_iter *from)
{
    struct file *file = iocb->ki_filp;
    struct block_device *bdev = I_BDEV(file->f_mapping->host);
    struct inode *bd_inode = bdev->bd_inode;

    if (bdev_read_only(bdev))          // 只读检查
        return -EPERM;
    if (IS_SWAPFILE(bd_inode) && ...)  // swap 文件保护
        return -ETXTBSY;

    ret = file_update_time(file);      // 更新文件时间戳

    if (iocb->ki_flags & IOCB_DIRECT) {
        ret = blkdev_direct_write(iocb, from);  // Direct IO: 跳过 Page Cache
        if (ret >= 0 && iov_iter_count(from))
            ret = direct_write_fallback(iocb, from, ret,
                    blkdev_buffered_write(iocb, from));  // 部分回退
    } else {
        ret = blkdev_buffered_write(iocb, from);  // ← Buffered IO
    }

    if (ret > 0)
        ret = generic_write_sync(iocb, ret);  // 需要 fsync? 处理 REQ_FUA
    return ret;
}
```

`blkdev_buffered_write`（`block/fops.c:632`）只有一行：

```c
static ssize_t blkdev_buffered_write(struct kiocb *iocb, struct iov_iter *from)
{
    return iomap_file_buffered_write(iocb, from, &blkdev_iomap_ops);
}
```

`iomap_file_buffered_write` 是内核文件系统写路径的标准入口，内部调用 `iomap_write_iter`。

### 5.6.2 iomap_write_iter —— 数据进入 Page Cache

```c
// mm/iomap.c — 简化
static size_t iomap_write_iter(struct iov_iter *iter, struct address_space *mapping, ...)
{
    do {
        // ① 获取/分配 folio
        status = iomap_write_begin(mapping, pos, bytes, &folio);
        // → filemap_get_folio() 查 xarray
        // → 没找到则分配新 folio

        // ② 从用户态拷贝数据到内核 folio
        copied = copy_from_iter_atomic(folio_address(folio) + offset,
                                       bytes, iter);
        // ★ 数据到了这里！Page Cache 已包含用户写的内容

        // ③ 标记 folio 脏 + end io
        status = iomap_write_end(mapping, pos, bytes, copied, folio, ...);
        // → iomap_set_range_uptodate()   ← 标记"数据有效"
        // → __folio_mark_dirty()         ← 标记"待写回"！

    } while (iter->count > 0);
}
```

`copy_from_iter_atomic` 之后，用户数据已经拷贝到内核的 Page Cache 中。`__folio_mark_dirty` 将 folio 的 `PG_dirty` 位置 1，并将其加入 inode 的脏页链表。

**关键认知：** `copy_from_iter_atomic` 执行完毕、write() 返回时，数据还在 DDR 的 Page Cache 里，没有到 eMMC。真正的刷盘由后台 `wb_workfn` 线程在脏页超阈值或超时后执行。

### 5.6.3 Buffered Write 的"异步"本质

```
write(fd, buf, 4096)
  │
  ├─ blkdev_buffered_write()
  │   └─ iomap_file_buffered_write()
  │       └─ iomap_write_iter()
  │           ├─ iomap_write_begin()      分配/获取 folio
  │           ├─ copy_from_iter_atomic()  用户数据 → 内核 folio
  │           └─ __folio_mark_dirty()     标记脏页
  │
  └─ write() 返回 ← 此时数据在 Page Cache，不在 eMMC！
                    │
                    ▼
                    （时间流逝...）
                    │
                    ▼
                wb_workfn()               [内核线程，后台执行]
                    │
                    ├─ 触发条件:
                    │   A. dirty_background_ratio > 10%（脏页太多）
                    │   B. dirty_expire_interval > 30s（脏页超时）
                    │   C. 显式 sync/fsync
                    │   D. 内存压力（kswapd 回收）
                    │
                    ├─ writeback_sb_inodes()
                    │   └─ ext4_writepages() / blkdev_writepages()
                    │       └─ submit_bio(wbc)   ★ 到这里才真正提交到块层！
                    │
                    └─ 之后进入 06 的 CMD25 多块写路径
```

对 `dd if=/dev/zero of=/mnt/testfile bs=4k count=256` 写入 1MB 的场景：如果系统内存充足（>1GB），1MB 脏页远未达到 `dirty_background_ratio`（默认 10%），所以通常由脏页超时（30s）触发后台回写。

### 5.6.4 Direct Write 路径（对比）

```c
// 用户态:
dd if=/dev/zero of=/mnt/testfile bs=4k count=256 oflag=direct
```

`O_DIRECT` 切断了 Page Cache 路径：

```
vfs_write()
  └─ blkdev_write_iter()
       ├─ IOCB_DIRECT 标记
       └─ blkdev_direct_write()
            └─ blkdev_direct_IO()
                 └─ mapping->a_ops->direct_IO()
                      = blkdev_direct_IO()
                        → __blkdev_direct_IO()
                          → submit_bio()     ★ 直接提交！
```

**差异总结：**

| 维度 | Buffered Write | Direct Write |
|------|---------------|-------------|
| 数据路径 | 用户 → Page Cache → wb_workfn → submit_bio | 用户 → submit_bio |
| write() 返回 | 数据进 Page Cache 立即返回 | IO 完成后返回 |
| 进程行为 | 几乎不阻塞 | 在 `dio->submit.wait` 上阻塞 |
| 块对齐 | 无要求 | 必须 512B 对齐 |
| 适用场景 | 常规文件 IO | 数据库、定制 IO 引擎 |

### 5.6.5 写路径总结

```
Buffered Write:
  用户 → vfs_write → blkdev_write_iter → blkdev_buffered_write
    → iomap_write_iter → copy_from_iter → __folio_mark_dirty
    → write() 返回
    → (后台 wb_workfn) → writeback → submit_bio(bio)  ★ 06 接管

Direct Write:
  用户 → vfs_write → blkdev_write_iter → blkdev_direct_write
    → submit_bio(bio)                                    ★ 06 接管
```

---

## 5.7 与 06 的分工边界

```
本文(05)                            06-IO-Path.md
══════════════════════════          ══════════════════════════
用户态: dd read/write               submit_bio 之后
  │                                    │
vfs_read / vfs_write                   │
  │                                    │
blkdev_read_iter / blkdev_write_iter   │
  │                                    │
filemap_read / iomap_write_iter        │
  │                                    │
submit_bio(bio) ─────── 分界线 ───→  mmc_mq_queue_rq()
                                      │
                                    mmc_blk_mq_issue_rq()
                                      │
                                    CMD18/CMD25 + IDMA
                                      │
                                    中断 → 进程唤醒
```

- **05** 关心 VFS 层如何将用户 IO 转化为 `bio`，然后调用 `submit_bio`
- **06** 从 `submit_bio` 开始，追踪 blk-mq 如何将 `bio` 派发给 MMC 块层驱动，MMC core 如何翻译为 CMD18/CMD25，SDMMC2 IDMA 如何传输数据，中断如何完成 IO

---

## 5.8 关键代码文件索引

| 文件 | 作用 | 章节 |
|------|------|------|
| `fs/read_write.c` | `ksys_read`、`vfs_read`、`new_sync_read` | 5.5 |
| `block/fops.c` | `def_blk_fops`、`blkdev_read_iter`、`blkdev_write_iter`、`blkdev_open` | 5.4, 5.5, 5.6 |
| `block/bdev.c` | `I_BDEV`、`bdev_inode` 结构体 | 5.3 |
| `mm/filemap.c` | `filemap_read`、`filemap_get_pages`、`filemap_create_folio` | 5.5 |
| `mm/iomap.c` | `iomap_file_buffered_write`、`iomap_write_iter` | 5.6 |
| `include/linux/fs.h` | `struct file`、`struct inode`、`struct address_space` | 5.3 |
| `include/linux/dcache.h` | `struct dentry` | 5.3 |
| `include/linux/blkdev.h` | `struct block_device`、`I_BDEV` 声明 | 5.3 |

---

> **下一篇：** [06-IO-Path.md](06-IO-Path.md) — eMMC I/O 数据通路情景分析。从 `submit_bio` 进入块层开始，追踪 `dd read` / `dd write` / `sync` 三个真实情景在 MMC 核心层、Host 驱动、IDMA 硬件中的完整路径。
