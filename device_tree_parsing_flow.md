# 设备树解析流程详细解释

## 1. 概述

设备树（Device Tree）是一种描述硬件配置的数据结构，用于在操作系统启动时向内核传递硬件信息。设备树解析流程是指内核从引导加载程序（如U-Boot）接收设备树二进制（DTB）后，将其解析为内核可用的设备节点和属性的过程。

## 2. 主要阶段

### 2.1 引导加载程序阶段

- 引导加载程序（如U-Boot）将设备树二进制（DTB）加载到内存中。
- 引导加载程序可能根据实际硬件修改设备树（例如填充内存大小、MAC地址等）。
- 引导加载程序将DTB的物理地址传递给内核（通常通过寄存器或特定约定）。

### 2.2 内核早期启动阶段

- 内核入口点（如`start_kernel`）调用`setup_arch`，其中包含设备树解析的初始化。
- 在ARM架构中，`setup_machine_fdt`函数被调用，它：
  - 验证DTB的魔数（`OF_DT_HEADER`）。
  - 检查DTB的总大小是否在合理范围内。
  - 将DTB映射到内核虚拟地址空间（如果尚未映射）。
  - 调用`early_init_dt_scan`函数。

### 2.3 早期扫描（early_init_dt_scan）

- `early_init_dt_scan`遍历设备树的根节点和子节点，执行以下操作：
  - 解析`chosen`节点，获取内核启动参数（`bootargs`）、`initrd`起始地址和大小等。
  - 解析`memory`节点，获取物理内存布局。
  - 解析`aliases`节点，建立设备别名映射。
  - 解析`cpus`节点，获取CPU信息。

### 2.4 设备树展开（unflatten）

- 内核将扁平设备树（FDT）转换为展开的设备树（由`struct device_node`和`struct property`组成的树形结构）。
- 函数`unflatten_device_tree`完成此工作：
  - 分配内存用于存储设备节点和属性。
  - 递归遍历FDT，为每个节点创建`struct device_node`。
  - 为每个属性创建`struct property`，并复制属性值。
  - 建立父子关系、兄弟关系等指针。

### 2.5 设备驱动匹配

- 内核遍历展开的设备树，为每个节点寻找匹配的驱动程序。
- 匹配过程基于`compatible`属性、`device_type`属性等。
- 当找到匹配的驱动时，调用驱动的`probe`函数，并将设备节点信息传递给驱动。

### 2.6 设备资源获取

- 驱动程序通过设备树API（如`of_property_read_u32`、`of_iomap`、`of_irq_get`等）从设备节点获取资源：
  - 寄存器地址（`reg`属性）。
  - 中断号（`interrupts`属性）。
  - GPIO、时钟、DMA等。
- 这些API内部会查找节点的属性并返回相应的值。

## 3. 关键数据结构

- `struct device_node`：表示一个设备节点，包含名称、类型、父节点、子节点、属性链表等。
- `struct property`：表示一个属性，包含名称、长度、值指针。
- `struct of_device_id`：用于驱动匹配的兼容性表。

## 4. 示例流程（以ARM Linux为例）

1. 引导加载程序加载DTB到内存地址`0x80000000`。
2. 内核启动，`start_kernel`调用`setup_arch`。
3. `setup_arch`调用`setup_machine_fdt(dtb_phys)`。
4. `setup_machine_fdt`验证DTB，调用`early_init_dt_scan`。
5. `early_init_dt_scan`解析`chosen`节点，获取`bootargs`。
6. 内核调用`unflatten_device_tree`，将FDT转换为展开树。
7. 内核遍历展开树，为每个节点调用`of_platform_device_create`或`of_platform_populate`。
8. 驱动匹配成功后，`probe`函数执行，通过`of_*` API获取硬件资源。

## 5. 总结

设备树解析流程是Linux内核启动过程中至关重要的一环，它将硬件描述从引导加载程序传递到内核，并最终驱动硬件设备。理解这一流程有助于调试硬件相关问题以及编写设备树源文件（DTS）。
