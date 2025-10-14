---
title: 设备驱动
published: 2025-10-13
description: dev
tags: [OS, dev]
category: OS
draft: false
---

## 1. 设备是一组寄存器，一个设备一个协议
设备: 一种交换信息的接口：包括：**命令**、**状态**、**数据**
从驱动程序的角度来看，所有硬件设备都可以抽象为一组寄存器。这些寄存器是CPU与硬件设备之间沟通的桥梁。驱动程序通过读写这些寄存器来控制设备的行为、获取设备的状态以及传输数据。

*   **寄存器的种类**：
    *   **控制寄存器 (Control Registers)**：用于向设备发送命令，例如启动一次数据传输、设置设备模式等。
    *   **状态寄存器 (Status Registers)**：用于获取设备的当前状态，例如数据是否准备好、是否发生错误等。
    *   **数据寄存器 (Data Registers)**：用于在CPU和设备之间传递数据。
*   **设备协议**：
    设备协议是硬件设备与CPU（通过驱动程序）进行通信的一整套规则。它定义了：
    - 寄存器映射(Register Map)：驱动在初始化时会得到设备在系统I/O地址空间或内存地址空间中的位置。所有寄存器都通过这个基地址加上一个偏移量来访问。
    - 寄存器功能(Register Function)：驱动程序必须知道每个寄存器以及其中每个比特位的确切含义。写操作(THR)、读操作(RBR)、LSR(线路状态寄存器)等等
    - 命令序列与时序 & 中断机制。
    驱动程序开发者需要依据硬件厂商提供的设备手册(Datasheet)，来理解每个寄存器的具体功能和操作流程。
    例如，要让一个I2C设备执行读写操作，驱动程序需要按照特定的顺序，向控制寄存器写入启动信号、设备地址、寄存器地址，然后才能从数据寄存器中读写数据。而串口设备​​不需要设备地址和寄存器地址​​，因为CPU直接访问其物理寄存器，​​轮询状态寄存器​​和​​读写数据寄存器​​。

    因此，驱动程序的核心任务之一，就是将这些底层的、针对特定硬件的寄存器操作，封装成操作系统内核能够理解的标准接口。

## 2. 字符设备(char dev) & 块设备(block dev)
在Linux中，设备主要分为两种类型：字符设备和块设备，以适应不同类型设备的I/O特性。

*   **字符设备 (Character Devices)**：
    *   **特性**：以字节为单位进行数据传输，不经过系统缓存。应用程序的读写请求会直接传递给驱动程序。顺序访问，适合实时性要求高、数据量小的设备。
    *   **示例**：串口、键盘、鼠标、打印机等。
    *   **接口**：驱动程序实现 `read()`、`write()` 等操作，一次可以读写任意数量的字节。

*   **块设备 (Block Devices)**：
    *   **特性**：以固定大小的数据块（通常是512字节或其倍数）为单位进行数据传输。为了提高性能，块设备的I/O操作会经过系统的块设备层(Block Layer)进行**缓存**和**调度**。可随机访问，适合存储设备、需要高效传输大量数据的设备。
    *   **示例**：硬盘、U盘、SD卡等。
    *   **接口**：驱动程序的核心是提供一个“请求函数”(`request function`)，该函数处理来自块设备层的I/O请求。

## 3. 编写一个设备驱动的核心要素

一个完整的Linux设备驱动程序通常由以下几个关键部分组成：

*   **`file_operations` 结构体**：
    这是字符设备驱动的核心。它是一个**函数指针**的集合，定义了当用户空间程序对设备文件执行 `open`、`read`、`write`、`ioctl` 等系统调用时，内核应该调用的相应驱动函数。
*   **模块初始化与退出函数 (`lx_init`, `lx_exit`)**：
    *   `module_init(lx_init)`：通过 `module_init` 宏注册的初始化函数，在驱动模块被加载到内核时调用。
        *   **资源分配**：
            - 使用 `alloc_chrdev_region()` 动态分配设备号范围
            - 使用 `class_create()` 创建设备类
        *   **设备注册**：
            - 通过 `cdev_init()` 初始化字符设备结构
            - 使用 `cdev_add()` 将设备添加到系统
            - 通过 `device_create()` 创建设备节点
        *   **硬件初始化**：
            - 执行设备特定的硬件初始化
    *   `module_exit(lx_exit)`：通过 `module_exit` 宏注册的退出函数，在模块被卸载时调用。
        *   **设备注销**：
            - 使用 `device_destroy()` 销毁设备节点
            - 使用 `cdev_del()` 移除字符设备
            - 通过 `class_unregister()` `class_destroy()` 销毁设备类
        *   **资源释放**：
            - 通过 `unregister_chrdev_region()` 释放设备号
            - 释放所有分配的内存和资源
        *   **硬件清理**：
            - 恢复硬件到安全状态
            - 释放硬件资源
*   **核心操作函数 (`read`, `write`, `ioctl`)**：
    *   `read()`：从设备读取数据并将其拷贝到用户空间的功能。
    *   `write()`：将用户空间的数据写入设备的功能。
    *   `ioctl()`：`ioctl` (Input/Output Control) 是一个特殊的接口，用于处理那些无法通过简单的 `read`/`write` 实现的设备特定命令。例如，设置串口的波特率、获取摄像头支持的分辨率等。
*   **设备私有数据结构**：
    驱动程序通常会定义一个私有的结构体，用于存放该设备特有的状态信息、资源指针（如内存映射地址、中断号）、锁（用于并发控制）等。这个结构体实例通常在设备被探测到(probe)或打开(open)时创建和初始化。

## 4. 驱动程序通过`VFS`向用户空间暴露功能
驱动程序本身运行在内核空间，为了让用户空间的应用程序能够使用硬件设备，Linux提供了一套标准的机制。

*   **虚拟文件系统 (VFS - Virtual File System)**：
    VFS 是一个内核抽象层，它为用户空间程序提供了统一的文件操作接口（`open`, `read`, `write` 等），而无需关心底层文件系统或硬件设备的具体实现。

*   **设备文件节点 (Device Nodes)**：
    驱动程序在初始化时，会通过内核函数在 `/dev` 目录下创建一个特殊的设备文件。
    *   **创建**：通常使用 `class_create` + `device_create` 来创建设备节点。
    *   **主次设备号**：每个设备文件都与一个主设备号 (Major Number) 和一个次设备号 (Minor Number) 相关联。主设备号用于标识驱动程序，次设备号用于区分由同一驱动程序管理的多个同类设备。

*   **`ioctl()` 的核心作用**：
    当用户程序打开 `/dev` 下的设备文件后，就可以像操作普通文件一样对其进行读写。然而，对于复杂的设备控制，`ioctl()` 接口至关重要。
    *   **命令和参数**：用户程序通过 `ioctl` 系统调用，向驱动程序传递一个表示特定命令的整数（`cmd`）和一个可选的参数（`arg`）。
    *   **驱动实现**：驱动程序中的 `ioctl` 函数会根据传入的 `cmd`，执行相应的硬件操作，并通过 `arg` 与用户空间交换数据。这为驱动程序提供了一个灵活且可扩展的接口，用于实现任意的设备特定功能。

## 5. 设备驱动中的并发与同步
- 并发的主要来源：多任务处理、多处理器SMP、中断处理、抢占性​任务
- 原子操作 锁 信号量 中断屏蔽 WaitQ 

## 6. 设备驱动内存管理与DMA 
驱动程序在内核空间运行，必须使用内核提供的内存管理接口来安全地分配和使用内存。此外，为了实现高效的数据传输，现代设备通常使用DMA技术。

### 内存管理
*   **`kmalloc()`**：
    *   **描述**：分配的是**物理上连续**的内核内存块。
    *   **特点**：分配速度快，但可能因为物理内存碎片而导致大块内存分配失败。由于物理上连续，非常适合用于DMA操作。
    *   **标志(gfp_t flags)​​**：`GFP_KERNEL` 是最常用的标志，表示如果内存不足，调用者可以睡眠等待。`GFP_ATOMIC` 则用于中断上下文或持有自旋锁等场景，表示不能睡眠，内核要尽力满足分配请求。
*   **`vmalloc()`**：
    *   **描述**：分配的是**虚拟地址上连续**、但物理上可能不连续的大块内核内存。
    *   **特点**：可以分配非常大的内存块，因为它不受物理连续性的限制。但由于需要建立页表映射，其分配/读写性能开销比 `kmalloc()` 大，且通常不适合直接用于DMA。
*   **页分配器 `alloc_pages()`**：
    *   **描述**：以页（通常4KB）为单位分配物理连续内存。alloc_pages()返回 struct page *，__get_free_pages()返回起始虚拟地址。
    *   **特点**：可以分配比 kmalloc 更大的物理连续内存块（多页），用于DMA的缓冲区或某些特殊硬件需求。

*   **内存映射 `mmap`**：
    *   **描述**：`mmap` 是一种允许用户空间应用程序**直接访问**设备内存（如显卡的显存）或驱动程序分配的内核缓冲区的强大机制。
    *   **工作原理**：驱动程序通过实现 `file_operations` 中的 `mmap` 函数，将设备的物理内存地址或内核缓冲区映射到调用进程的虚拟地址空间。这样，应用程序就可以像访问普通内存一样读写设备，避免了在内核空间和用户空间之间进行数据拷贝的开销，极大地提高了I/O效率。

### 直接内存访问 (DMA - Direct Memory Access)
*   **概念**：DMA允许外部设备在没有CPU介入的情况下，直接与系统主内存进行数据传输。
*   **优势**：当设备需要传输大量数据时（如网卡收发数据包、磁盘读写文件），DMA可以把CPU从繁重的内存拷贝工作中解放出来，使CPU只需初始化DMA传输在传输完成时处理中断即可，从而极大地提升系统吞吐量和响应性。
*   **驱动的工作流程**：
    1.  **分配DMA缓冲区**：驱动程序使用 `dma_alloc_coherent()` API分配一块物理上连续且对设备可见的内存区域。
    2.  **获取总线地址**：内核会返回该缓冲区的CPU虚拟地址和设备总线地址。驱动程序使用虚拟地址来填充数据，而将总线地址写入设备的DMA控制器寄存器。
    3.  **编程DMA控制器**：驱动程序通过I/O操作，告诉设备要传输的数据的源地址（总线地址）、目标地址、数据长度等信息，然后启动DMA传输。
    4.  **传输完成**：传输完成后，设备会通过产生一个中断来通知CPU。驱动程序的中断处理函数随后会进行后续处理，例如通知上层协议栈数据已准备好。

## 7. Docker如何获取并使用宿主机的设备
Docker作为一个用户空间的应用程序，其容器本质上是运行在宿主机内核上的一个隔离进程。

### Docker访问设备的基本方式
1.  **`--device` 标志**：
    *   **描述**：这是最直接的方式。它允许你将宿主机上的一个设备文件（如 `/dev/sda1`、`/dev/ttyUSB0`）明确地暴露给容器。
    *   **示例**：`docker run --device=/dev/ttyUSB0 my_container`
    *   **工作原理**：Docker引擎会通过 stat() 系统调用，获取设备的元数据（设备类型、主次设备号），利用Linux内核的**cgroups**机制，在容器的设备控制组中明确授权该容器可以访问指定的主/次设备号。同时，它会在容器的挂载命名空间(Mount Namespace)的 `/dev` 目录下执行 mknod() 系统调用创建相应的`设备文件节点`，性能开销极低，实现“透传”设备。
    *   **cgroups**: type major:minor permissions(rwm)
    *   **每一次涉及该设备的系统调用**: 内核的VFS层接收到 open("/dev/mydevice", O_RDWR); 请求，解析设备信息，查看发起open的进程所属cgroup的设备控制器配置。通过后VFS就像处理任何来自宿主机的请求一样分派open请求、返回fd来执行read()/write()/ioctl()

2.  **特权模式 (`--privileged`)**：
    *   **描述**：这是一个“一刀切”的强大选项。它会移除容器与宿主机之间几乎所有的隔离（包括cgroups、AppArmor、Seccomp等），使得容器内的root用户拥有与宿主机root用户几乎相同的权限。
    *   **效果**：在这种模式下，容器可以访问宿主机上**所有**的设备，即 `/dev` 目录下的所有内容。
    *   **风险**：极不安全，给予了容器过多的权限，应仅在绝对必要且完全信任镜像来源时使用。
<!-- ### Bind Mount 详解 -->

## P.S. 
> 用户态设备驱动
- 追求​​极致性能​​且愿意承担更复杂的开发 （SPDK/DPDK）
  - **轮询模式替代中断(Polling vs. Interrupts)**: `双刃剑`(内核驱动靠网卡中断来通知数据包到达，中断带来的上下文切换开销在高流量下是巨大的。DPDK驱动采用​​主动轮询​​模式，让CPU核心一直检查网卡是否有新数据包，完全消除了中断开销，但是独占硬件资源。)
  - **零拷贝​​(Zero-Copy)**: DPDK用户态驱动可以通过 `DMA 和 mmap` ，在初始化阶段就将设备的硬件资源（如网卡的收发队列内存）直接映射到应用程序的虚拟地址空间。避免了 `copy_from_user() read()/write()` 内核态和用户态之间的数据拷贝。 e.g. UIO (Userspace I/O) VFIO (Virtual Function I/O)
  - **CPU亲和性与缓存优化(CPU Affinity & Cache Locality)**: 为了避免多核环境下的线程切换和缓存颠簸（cache bouncing），用户态驱动框架可以将特定的处理线程`绑定`到特定的CPU核心上，充分利用`缓存局部性原理`
- 系统安全性和稳定性: 
  - **故障隔离(Fault Isolation)**: 内核态设备驱动的 `bug` 不会直接导致 kernel panic，驱动程序作为一个普通的进程运行。即使驱动代码崩溃，也只会影响该进程自身，操作系统内核和其他应用程序依然可以稳定运行。（[VFIO](https://docs.linuxkernel.org.cn/driver-api/vfio.html)）
  - **进程隔离**: 利用现代CPU的内存管理单元(`MMU`)为用户态进程提供了独立的地址空间。
- 开发灵活性和可控制性​​: 易于开发与调试 + 高度可定制化

适用场景：
- 高性能网络处理、高性能存储、虚拟化、驱动简单且中断不频繁的定制硬件
缺点：
- **接受失去丰富的内核生态和服务**​，如内核协议栈。进而可能导致缺乏标准化接口
- **接受独占硬件**，紧密耦合
- 等

## 最后，手写一个pipe设备驱动的C语言代码示例
```c
// pipe.c
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/fs.h>
#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/kfifo.h>
#include <linux/module.h>
#include <linux/poll.h>
#include <linux/sched.h>
#include <linux/slab.h>
#include <linux/spinlock.h>
#include <linux/uaccess.h>
#include <linux/version.h>
#include <linux/wait.h>

#define MAX_DEV    2
#define MAX_EVENTS 16          // 最大事件队列长度
#define EVENT_SIZE sizeof(int) // 每个事件的大小（int）

static int dev_major = 0;
static struct class* lx_class = NULL;

static ssize_t lx_read(struct file*, char __user*, size_t, loff_t*);
static ssize_t lx_write(struct file*, const char __user*, size_t, loff_t*);
static unsigned int lx_poll(struct file*, poll_table*);

static struct file_operations fops = {
    .owner = THIS_MODULE,
    .read = lx_read,
    .write = lx_write,
    .poll = lx_poll,
};

// 设备私有数据结构
struct pipe_dev {
    struct cdev cdev;                          // 字符设备
    wait_queue_head_t read_queue;              // 读等待队列
    spinlock_t lock;                           // 保护FIFO的自旋锁
    struct kfifo fifo;                         // 事件队列
    char fifo_buffer[MAX_EVENTS * EVENT_SIZE]; // FIFO存储空间
};

static struct pipe_dev devs[MAX_DEV]; // 设备数组

static int __init lx_init(void) {
    dev_t dev;
    int i, err;
    // 动态分配设备号范围
    err = alloc_chrdev_region(&dev, 0, MAX_DEV, "pipe");
    if (err < 0) {
        printk(KERN_ERR "Failed to allocate char device region\n");
        return err;
    }

    // 获取主设备号
    dev_major = MAJOR(dev);
    // 创建设备类
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 4, 0)
    lx_class = class_create("pipe");
#else
    lx_class = class_create(THIS_MODULE, "pipe");
#endif
    if (IS_ERR(lx_class)) {
        unregister_chrdev_region(dev, MAX_DEV);
        return PTR_ERR(lx_class);
    }

    // 初始化每个设备
    for (i = 0; i < MAX_DEV; ++i) {
        struct pipe_dev* dev_ptr = &devs[i];
        // 初始化kfifo
        if (kfifo_initialized(&dev_ptr->fifo)) {
            printk(KERN_WARNING "FIFO already initialized for device %d\n", i);
        } else {
            kfifo_init(&dev_ptr->fifo, dev_ptr->fifo_buffer, sizeof(dev_ptr->fifo_buffer));
        }
        // 初始化自旋锁
        spin_lock_init(&dev_ptr->lock);
        // 初始化等待队列
        init_waitqueue_head(&dev_ptr->read_queue);
        // 初始化字符设备
        cdev_init(&dev_ptr->cdev, &fops);
        dev_ptr->cdev.owner = THIS_MODULE;
        // 添加字符设备
        err = cdev_add(&dev_ptr->cdev, MKDEV(dev_major, i), 1);
        if (err) {
            printk(KERN_ERR "Error %d adding device %d\n", err, i);
            continue;
        }
        // 创建设备节点 /dev/pipeX
        device_create(lx_class, NULL, MKDEV(dev_major, i), NULL, "pipe%d", i);
    }
    printk(KERN_INFO "pipe event notifier driver loaded with major %d\n", dev_major);
    return 0;
}

static void __exit lx_exit(void) {
    int i;
    for (i = 0; i < MAX_DEV; ++i) {
        device_destroy(lx_class, MKDEV(dev_major, i));
        cdev_del(&devs[i].cdev);
    }
    class_destroy(lx_class);
    unregister_chrdev_region(MKDEV(dev_major, 0), MAX_DEV);
    printk(KERN_INFO "pipe event notifier driver unloaded\n");
}

static unsigned int lx_poll(struct file* file, poll_table* wait) {
    struct pipe_dev* dev = container_of(file->f_inode->i_cdev, struct pipe_dev, cdev);
    unsigned int mask = 0;
    // 注册等待队列
    poll_wait(file, &dev->read_queue, wait);

    spin_lock(&dev->lock);
    if (!kfifo_is_empty(&dev->fifo)) {
        mask |= POLLIN | POLLRDNORM; // 返回可读标志
    }
    spin_unlock(&dev->lock);
    return mask;
}

static ssize_t lx_read(struct file* file, char __user* buf, size_t count, loff_t* offset) {
    struct pipe_dev* dev = container_of(file->f_inode->i_cdev, struct pipe_dev, cdev);
    int event;
    DEFINE_WAIT(wait);
    if (count < EVENT_SIZE) {
        return -EINVAL; // 缓冲区太小
    }

    // 等待直到有事件可用
    while (kfifo_is_empty(&dev->fifo)) {
        if (file->f_flags & O_NONBLOCK) {
            return -EAGAIN; // 非阻塞模式，立即返回
        }

        prepare_to_wait(&dev->read_queue, &wait, TASK_INTERRUPTIBLE);
        spin_unlock(&dev->lock); // 解锁后休眠
        // 检查是否有信号中断
        if (signal_pending(current)) {
            finish_wait(&dev->read_queue, &wait);
            return -ERESTARTSYS;
        }
        // 让出CPU直到被唤醒
        schedule();
        finish_wait(&dev->read_queue, &wait);
    }

    spin_lock(&dev->lock); // 唤醒后重新加锁
    if (kfifo_out(&dev->fifo, &event, EVENT_SIZE) != EVENT_SIZE) {
        spin_unlock(&dev->lock);
        return -EIO; // 读取失败
    }
    spin_unlock(&dev->lock);

    // 将事件复制到用户空间
    if (copy_to_user(buf, &event, EVENT_SIZE)) {
        return -EFAULT;
    }
    return EVENT_SIZE;
}

static ssize_t lx_write(struct file* file, const char __user* buf, size_t count, loff_t* offset) {
    struct pipe_dev* dev = container_of(file->f_inode->i_cdev, struct pipe_dev, cdev);
    int event;
    unsigned int pushed;
    if (count < EVENT_SIZE) {
        return -EINVAL; // 数据太小
    }
    // 从用户空间复制事件
    if (copy_from_user(&event, buf, EVENT_SIZE)) {
        return -EFAULT;
    }

    spin_lock(&dev->lock);
    // 尝试将事件推入队列
    if (kfifo_avail(&dev->fifo) < EVENT_SIZE) {
        spin_unlock(&dev->lock);
        return -ENOSPC; // 队列已满
    }
    pushed = kfifo_in(&dev->fifo, &event, EVENT_SIZE);
    spin_unlock(&dev->lock);

    if (pushed != EVENT_SIZE) {
        return -EIO; // 写入失败
    }
    // 唤醒等待的读进程
    wake_up_interruptible(&dev->read_queue);
    return EVENT_SIZE;
}

module_init(lx_init);
module_exit(lx_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("aLinChe");
MODULE_DESCRIPTION("Kernel Event Notifier Pipe Driver");
```
```makefile
# Makefile
obj-m := pipe.o
KDIR ?= /lib/modules/$(shell uname -r)/build

default:
	$(MAKE) module
module:
	$(MAKE) -C $(KDIR) M=$(PWD) modules
clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
.PHONY: module clean
# lsmod 
# insmod pipe.ko 
```