---
title: 文件系统入门
published: 2025-11-13
description: 文件系统API与FAT32入门
tags: [OS, fs]
category: OS
draft: false
---

## 设备在应用程序之间共享
> why 文件系统

共享原始磁盘：字符序列不是一个磁盘的好抽象 (x)

虚拟磁盘: 
  - 提供合理的API使多个应用程序能共享数据
  - 隔离，使错误不能任意扩大
对“存储设备的虚拟化”
- 磁盘(IO设备): 可读写的字节序列
- 虚拟磁盘(文件): 可读写的动态字节序列
  - 命名管理: 名称、检索、遍历
  - 数据管理: std::vector<char> 随机读写/resize

---
## 为什么需要文件系统：实现应用程序间的安全高效共享
在现代操作系统中，多个应用程序并发运行，它们都需要持久化地存储和访问数据。一个核心问题是：这些应用程序如何安全高效地**共享底层的存储设备**（如硬盘或SSD）？而这正是文件系统要解决的关键问题。

### 1. 问题的根源：为何不能直接共享原始磁盘？
- 共享原始磁盘：字符序列不是一个磁盘的好抽象

原始的物理磁盘，对于操作系统而言，可以看作一个巨大的一维数组序列，由可读写的物理块(Block)组成。如果允许多个应用程序直接共享这个原始设备，而不加任何管理，将会导致一场灾难。
*   **缺乏基本的数据结构：** 应用程序看到的是一堆无差别的磁盘块。它们需要自行记录哪些块属于自己，哪些块是空闲的。这极易导致冲突，一个应用程序可能会意外或故意地覆盖另一个程序的数据。
*   **并发访问失控：** 如果两个应用程序同时尝试写入磁盘的同一区域，它们的操作会相互干扰覆盖。
*   **没有隔离与保护：** 任何应用程序都可以读取或修改磁盘的任意部分，包括其他程序的数据甚至操作系统的核心文件。这带来了巨大的安全风险，一个程序的错误可能导致整个系统崩溃。
*   **元数据缺失：** 诸如“文件名”、“文件大小”、“创建时间”、“访问权限”这类描述数据的数据（即元数据）无处存放。每个应用程序都需要用自己独特的方式去实现这些功能，导致数据无法通用和共享。

### 2. 文件系统：存储设备的虚拟化管理者
为了解决上述问题，操作系统引入了文件系统(File System)这一关键抽象层。文件系统可以被视为 **“对存储设备的虚拟化”**。 它在混乱的物理磁盘之上，构建了一个有序可靠高效的虚拟世界。
*   **物理磁盘(`IO设备`):** 从硬件层面看，它是一个可按**块(Block)**为单位进行读写的、**`定长`**的字节序列。
*   **文件系统提供的虚拟磁盘(`文件`):** 这是提供给应用程序的逻辑视角，它将物理磁盘抽象成一个个**文件(File)**。每个文件都是一个可读写的、**`动态`**的字节序列。（`std::vector<char>`）
这种虚拟化带来了质的飞跃，屏蔽了底层硬件的复杂性，为上层应用提供了简洁、一致的接口。

### 3. 文件系统的核心机制：
#### **隔离与保护：构建错误与恶意行为的防火墙**
这是实现安全共享的基石。文件系统提供了一套严格的访问控制机制，确保数据不被非法访问或意外破坏。
*   **访问控制：** 通过经典的权限模型（例如，Linux系统中为每个文件设定的所有者、所属组、其他人的读(r)、写(w)、执行(x)权限）和更精细的访问控制列表(ACLs)，文件系统可以精确地控制哪个用户或哪个程序能对文件做什么操作。
*   **并发控制：** 当多个应用程序需要同时访问同一个文件时，文件系统通过**文件锁(File Locking)**等机制来协调操作顺序。 这可以防止多个程序同时写入导致数据错乱，确保了数据的一致性。

#### **命名管理：让数据井然有序，易于检索**
文件系统为存储在磁盘上的数据提供了人类可读的命名空间，解决了“找得到”和“管得好”的问题。
*   **命名与检索：** 文件系统允许我们为文件和目录赋予有意义的名称，并通过路径(Path)来定位它们。例如 /home/user/document.txt。
*   **层次化结构（遍历）：** 现代文件系统大多采用树状的目录结构来组织文件。 这种层次化的命名空间非常符合人类的组织习惯，使得我们可以方便地对文件进行分类、组织和遍历。
*   **元数据(Metadata)管理：** 文件系统不仅存储文件本身的数据，还存储着关于文件的“数据”——元数据(Metadata)。 这包括文件名、大小、权限、所有者、创建/修改时间戳，以及指向真正数据块的指针等。在Linux等系统中，这些信息被集中存放在一个叫做**索引节点(`inode`)** 的数据结构中。

#### **数据管理：提供动态灵活的字节容器**
*   **屏蔽物理细节：** 应用程序操作文件时，看到的是一个连续的字节序列。但实际上，文件的数据块可能是不连续地（碎片化地）存储在磁盘的各个位置。文件系统负责维护这种从“逻辑连续”到“物理分散”的映射关系，对应用程序完全透明。
*   **随机读写：** 文件系统提供了强大的API，允许应用程序在文件的任意位置（通过偏移量 `offset`）进行读写，而不仅仅是从头到尾顺序读写。
*   **动态调整大小(Resize)：** 文件的大小不是固定的。当应用程序向文件写入更多数据时，文件系统会自动从磁盘的“空闲空间池”中分配新的数据块给该文件；当文件数据被删除或截断时，文件系统会回收这些数据块，使其可以被其他文件重用。


## mount 
```c
int mount(const char *source, const char *target, const char *filesystemtype, unsigned long mountflags, const void *data);
```
```sh
sudo strace mount ./disk.img ./mnt
# ioctl(3, LOOP_CTL_GET_FREE)
# ioctl(4, LOOP_SET_FD, 3)
```

### 硬链接
理解硬链接和软链接（也叫符号链接）的区别，关键在于理解Linux文件系统如何通过inode来管理文件。下面这个表格能让你快速把握它们的核心差异。

| 特性 | 硬链接 (Hard Link) | 软链接 (Symbolic Link) |
| :--- | :--- | :--- |
| **本质** | 是原始文件的**另一个别名**，直接指向文件数据的inode  | 是一个**独立的特殊文件**，内容为原始文件的路径字符串  |
| **inode号码** | 与原始文件**相同**  | 与原始文件**不同**，拥有自己的inode  |
| **跨文件系统** | **不支持**。必须在同一文件系统内  | **支持**。可以链接到不同分区或设备上的文件  |
| **链接目录** | **通常不允许**（超级用户可能有例外）  | **支持**  |
| **删除原始文件** | **不受影响**。文件数据仍可通过硬链接访问  | **链接失效**。变成“悬空链接”，指向无效路径  |
| **文件大小** | 与原始文件**相同**（因为它们共享数据块）  | 通常很小，是**存储的路径名的字符长度**  |
| **关系比喻** | 一个人的多个**绰号**，指向同一个实体  | 指向某个文件的**快捷方式**  |
-   **使用硬链接(ln)的场景**：
    -   **防止误删重要文件**：为关键文件创建硬链接，即使“原始”文件名被删除，数据仍通过硬链接安全存在。
    -   **文件备份与同步**：在一些备份场景中，创建硬链接可以节省空间，并且保证备份文件与源文件同步更新。但请注意，硬链接无法备份目录本身（除非对目录内每个文件分别创建硬链接）。
-   **使用软链接(ln -s)的场景**：
    -   **创建快捷方式**：这是最常用的场景，比如在桌面为深藏目录中的程序创建一个软链接，方便访问。
    -   **链接到目录**：这是软链接的独特优势，你可以为整个目录创建软链接。
    -   **跨文件系统链接**：当需要链接的文件位于另一个硬盘或分区时，必须使用软链接。

link unlink 
getdents 

readdir globbing 


## FAT (File Allocation Table)
文件分配表（File Allocation Table）是一种自 20 世纪 70 年代末期便已存在的文件系统，最初是为软盘设计的。 随着磁盘容量的不断增长，FAT 也经历了 FAT12、FAT16 到 FAT32 的演进。 FAT32 中的“32”指的是文件分配表中使用 32 位来寻址，这使得它能够支持更大的分区和文件。

尽管现代操作系统（如 Windows NT 及其后续版本）大多采用 NTFS 文件系统，但 FAT32 凭借其卓越的兼容性，至今仍在移动存储介质（如 U 盘、SD 卡）和需要跨平台数据交换的场景中扮演着不可或缺的角色。 它几乎被所有主流操作系统支持，包括 Windows、macOS 和 Linux。

**RTFM**：[Microsoft Extensible File Allocation Table File System Specification, FAT: General Overview of On-Disk Format](https://jyywiki.cn/pages/OS/manuals/MSFAT-spec.pdf)

---
### 卷结构 (Volume Structure)：磁盘上的三大法定区域
1.  **保留区域 (`Reserved Region`)**
2.  **FAT 区域 (`FAT Region`)**
3.  **数据区域 (`File and Directory Data Region`)**
4.  （可选）**未分配空间 (Unallocated Space)**

#### 1. 保留区域：文件系统的“启动参数”
该区域位于卷的起始位置，其首个扇区被称为**卷引导记录(`VBR`, Volume Boot Record)**。 `VBR` 对于操作系统而言至关重要，因为它内部包含了一个名为**BIOS 参数块(`BPB`, BIOS Parameter Block)** 的数据结构，而 `BPB` 定义了该卷的所有物理和逻辑参数。

```cpp
#pragma pack(push, 1)
struct Header {
    // ... (跳转指令和OEM名称)
    u16 BPB_BytsPerSec; // 每个扇区的字节数 (必须是 512, 1024, 2048 或 4096)
    u8  BPB_SecPerClus; // 每个簇的扇区数 (必须是2的幂)
    u16 BPB_RsvdSecCnt; // 保留区域的总扇区数
    u8  BPB_NumFATs;    // FAT 的数量 (强烈推荐为 2)
    // ...
    u32 BPB_TotSec32;   // 卷中的总扇区数
    u32 BPB_FATSz32;    // 单个 FAT 表占用的扇区数
    u32 BPB_RootClus;   // 根目录的起始簇号 (通常是 2)
    // ...
    u16 Signature_word; // 结尾签名，必须是 0xAA55
};
#pragma pack(pop)
```
操作系统启动时，首先读取 VBR，并根据 `BPB_RsvdSecCnt` 确定 FAT 区域的起始位置，根据 `BPB_NumFATs` 和 `BPB_FATSz32` 确定数据区域的起始位置。这些参数是后续所有地址计算的基础。

#### 2. FAT 区域：簇链的索引地图
紧随保留区域之后的是一个或多个文件分配表 (FAT)。微软规范将 FAT 定义为一个**簇索引数组**，其核心功能是记录数据区中每个簇的状态。
*   **结构**：在 FAT32 中，这是一个由 32 位条目(u32)组成的数组。数组的下标（索引）与数据区的簇号一一对应。
*   **28位寻址**：尽管条目是 32 位的，但 FAT32 仅使用**低 28 位**来存储簇号。高 4 位被保留，且在读写时必须被操作系统屏蔽。
*   **特殊值**：
    *   `0x00000000`: 标志着一个**空闲簇(Free Cluster)**。
    *   `0x0FFFFFF7`: 标志着一个**坏簇(Bad Cluster)**，应被操作系统禁用。
    *   `0x0FFFFFF8` - `0x0FFFFFFF`: 任何等于或大于此范围的值都表示**链表结束标记 (`EOC`, End-of-Chain)**。`0x0FFFFFFF` 是最常用的 EOC 标记。
    *   其他值：指向文件中下一个簇的簇号。

#### 3. 数据区域：文件内容的最终归宿
这里是存储所有文件和目录实际数据的广阔空间。该区域被划分为大小一致的**簇(`Cluster`)**，簇是 FAT32 文件系统进行空间分配的最小单位。一个文件的数据可能散布在数据区内多个不连续的簇中，而将它们在逻辑上串联起来的“线”，就是 FAT 中记录的簇链。

### 核心机制：操作系统如何追踪文件簇链
1.  **定位目录条目 (Directory Entry)**：操作系统首先在父目录的数据中搜索目标文件。目录本身也是一个文件，其内容是由 32 字节的 `DirectoryEntry` 结构体组成的序列。
    ```cpp
    struct DirectoryEntry {
        u8  DIR_Name[11];  // 8.3 格式文件名
        u8  DIR_Attr;      // 文件属性
        // ...
        u16 DIR_FstClusHI; // 起始簇号高 16 位
        u16 DIR_FstClusLO; // 起始簇号低 16 位
        u32 DIR_FileSize;  // 文件大小（字节）
    };
    ```
2.  **获取起始簇号**：找到条目后，操作系统将 `DIR_FstClusHI` 和 `DIR_FstClusLO` 组合成一个 32 位整数，这便是文件数据的**第一个簇号**。
    `u32 start_cluster = (entry->DIR_FstClusHI << 16) | entry->DIR_FstClusLO;`
3.  **计算物理地址**：操作系统使用 BPB 中的参数计算出该簇在磁盘上的精确字节偏移。
    *   `FirstDataSector = BPB_RsvdSecCnt + (BPB_NumFATs * BPB_FATSz32);`
    *   `ClusterOffset = (cluster_num - 2) * BPB_SecPerClus * BPB_BytsPerSec;`
    *   `PhysicalAddress = FirstDataSector * BPB_BytsPerSec + ClusterOffset;`
4.  **读取数据**：操作系统从计算出的物理地址读取 `BPB_SecPerClus * BPB_BytsPerSec` 字节的数据，这是文件的第一块内容。
5.  **查询 FAT 表以追踪链条**：为了找到文件的下一部分，操作系统必须查询 FAT。
    *   **计算 FAT 条目地址**：`FATEntryOffset = current_cluster_num * 4;`
    *   **读取条目值**：操作系统从 FAT 区域的起始地址加上这个偏移，读取一个 32 位的值。
    *   **屏蔽高 4 位**：将读取到的值与 `0x0FFFFFFF` 进行按位与操作，以获取真实的下一个簇号。
    ```cpp
    u32 next_cluster(u32 cluster_num) const {
        // 定位 FAT 表
        const u32* fat_table = ...;
        // 索引并屏蔽
        return fat_table[cluster_num] & 0x0FFFFFFF;
    }
    ```
6.  **循环与终止**：操作系统将上一步获得的新簇号作为 `current_cluster_num`，重复步骤 3、4、5，直到从 FAT 中读取到的值是 EOC 标记。此时，文件读取过程完成。

### 长文件名 (LFN) 支持：一个巧妙的兼容层
微软规范详细描述了 FAT32 如何在不破坏与旧系统兼容性的前提下支持长文件名。
*   当一个文件拥有长文件名时，文件系统会为其创建一组或多组特殊的**长文件名目录条目 (LFN Entry)**。
*   这些 LFN 条目紧挨着该文件的真实 8.3 格式目录条目之前。
*   LFN 条目具有 `READ_ONLY | HIDDEN | SYSTEM | VOLUME_ID` 的特殊属性组合，不识别 LFN 的旧系统会直接忽略它们。
*   文件名（Unicode 编码）被分割并存储在这些 LFN 条目中，操作系统需要从后向前依次读取它们，才能重构出完整的文件名。

### P.S. 规范即代码，代码即规范 (RTFM & check)
```sh
dd if=/dev/zero of=fat32.img bs=1M count=100 # 100MB
mkfs.fat -F 32 -n "MY_FAT32" -i 0066CCFF -v fat32.img
fdisk -l fat32.img
fsck.fat -v fat32.img

# sudo mount fat32.img mnt
# sudo cp -r ... mnt
# ./a.out fat32.img &| vim -
```

```cpp
#include <cassert>
#include <cstdint>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <filesystem>
#include <fstream>
#include <iostream>
#include <memory>
#include <string>
#include <system_error>
#include <vector>

#ifdef __linux__
#include <fcntl.h>
#include <sys/mman.h>
#include <unistd.h>
#endif

using u8 = uint8_t;
using u16 = uint16_t;
using u32 = uint32_t;

namespace fat32 {

// 常量定义
constexpr u32 CLUSTER_INVALID = 0xFFFFFF7;
constexpr u16 SIGNATURE_WORD = 0xAA55;

// 使用强类型枚举代替宏定义
enum class Attribute : u8 {
    READ_ONLY = 0x01,
    HIDDEN = 0x02,
    SYSTEM = 0x04,
    VOLUME_ID = 0x08,
    DIRECTORY = 0x10,
    ARCHIVE = 0x20
};

// 使用POD结构体，确保内存布局
#pragma pack(push, 1)
struct Header {
    u8 BS_jmpBoot[3];
    u8 BS_OEMName[8];
    u16 BPB_BytsPerSec;
    u8 BPB_SecPerClus;
    u16 BPB_RsvdSecCnt;
    u8 BPB_NumFATs;
    u16 BPB_RootEntCnt;
    u16 BPB_TotSec16;
    u8 BPB_Media;
    u16 BPB_FATSz16;
    u16 BPB_SecPerTrk;
    u16 BPB_NumHeads;
    u32 BPB_HiddSec;
    u32 BPB_TotSec32;
    u32 BPB_FATSz32;
    u16 BPB_ExtFlags;
    u16 BPB_FSVer;
    u32 BPB_RootClus;
    u16 BPB_FSInfo;
    u16 BPB_BkBootSec;
    u8 BPB_Reserved[12];
    u8 BS_DrvNum;
    u8 BS_Reserved1;
    u8 BS_BootSig;
    u32 BS_VolID;
    u8 BS_VolLab[11];
    u8 BS_FilSysType[8];
    u8 __padding_1[420];
    u16 Signature_word;
};

struct DirectoryEntry {
    u8 DIR_Name[11];
    u8 DIR_Attr;
    u8 DIR_NTRes;
    u8 DIR_CrtTimeTenth;
    u16 DIR_CrtTime;
    u16 DIR_CrtDate;
    u16 DIR_LastAccDate;
    u16 DIR_FstClusHI;
    u16 DIR_WrtTime;
    u16 DIR_WrtDate;
    u16 DIR_FstClusLO;
    u32 DIR_FileSize;
};
#pragma pack(pop)

static_assert(sizeof(Header) == 512, "Header size mismatch");
static_assert(sizeof(DirectoryEntry) == 32, "DirectoryEntry size mismatch");

// 异常类用于错误处理
class Fat32Exception : public std::exception {
  private:
    std::string m_msg;

  public:
    explicit Fat32Exception(const std::string& msg) : m_msg(msg) {}
    const char* what() const noexcept override { return m_msg.c_str(); }
};

// RAII类管理内存映射
class MemoryMappedFile {
  private:
    void* m_data = nullptr;
    size_t m_size = 0;

  public:
    MemoryMappedFile(const std::string& filename) {
#ifdef __linux__
        int fd = open(filename.c_str(), O_RDONLY);
        if (fd < 0) {
            throw Fat32Exception("Failed to open file: " + filename);
        }
        // lseek获取文件大小
        m_size = lseek(fd, 0, SEEK_END);
        if (m_size == static_cast<off_t>(-1)) {
            close(fd);
            throw Fat32Exception("Failed to get file size");
        }
        // mmap内存映射
        m_data = mmap(nullptr, m_size, PROT_READ, MAP_PRIVATE, fd, 0);
        close(fd);
        if (m_data == MAP_FAILED) {
            throw Fat32Exception("Memory mapping failed");
        }
#else
        throw Fat32Exception("Memory mapping only supported on Linux");
#endif
    }

    ~MemoryMappedFile() {
        if (m_data) {
#ifdef __linux__
            munmap(m_data, m_size);
#endif
        }
    }

    // 禁用拷贝
    MemoryMappedFile(const MemoryMappedFile&) = delete;
    MemoryMappedFile& operator=(const MemoryMappedFile&) = delete;
    // 允许移动
    MemoryMappedFile(MemoryMappedFile&& other) noexcept : m_data(other.m_data), m_size(other.m_size) {
        other.m_data = nullptr;
        other.m_size = 0;
    }
    MemoryMappedFile& operator=(MemoryMappedFile&& other) noexcept {
        if (this != &other) {
            if (m_data) {
#ifdef __linux__
                munmap(m_data, m_size);
#endif
            }
            m_data = other.m_data;
            m_size = other.m_size;
            other.m_data = nullptr;
            other.m_size = 0;
        }
        return *this;
    }

    const void* data() const { return m_data; }
    size_t size() const { return m_size; }

    template <typename T>
    const T* as() const {
        return reinterpret_cast<const T*>(m_data);
    }
};

// 文件块范围信息结构体
struct ClusterRange {
    u32 start_cluster;
    u32 end_cluster;
    u32 cluster_count{};

    ClusterRange(u32 start, u32 end) : start_cluster(start), end_cluster(end) {
        if (start <= end) {
            cluster_count = end - start + 1;
        }
    }

    std::string to_string() const {
        if (cluster_count == 0) {
            return "empty";
        }
        if (cluster_count == 1) {
            return "(" + std::to_string(start_cluster) + ")";
        }
        return "(" + std::to_string(start_cluster) + "-" + std::to_string(end_cluster) + ") # " +
               std::to_string(cluster_count) + " clusters";
    }
};

// 主FAT32解析器类
class Fat32Parser {
  private:
    MemoryMappedFile m_mapped_file;
    const Header* m_header = nullptr;

  public:
    explicit Fat32Parser(const std::string& filename) : m_mapped_file(filename) {
        m_header = m_mapped_file.as<Header>();
        if (!m_header) {
            throw Fat32Exception("Invalid header pointer");
        }
        validate_header();
        print_header_info(filename);
    }

    void traverse_filesystem() {
        std::cout << "File System Structure:\n";
        std::cout << "=====================\n";
        ClusterRange root_range = get_cluster_range(m_header->BPB_RootClus);
        std::cout << "[根目录] <DIR> 簇块范围: " << root_range.to_string() << "\n";
        dfs(m_header->BPB_RootClus, 1, true);
    }

  private:
    void validate_header() const {
        if (m_header->Signature_word != SIGNATURE_WORD) {
            throw Fat32Exception("Invalid MBR signature");
        }
        if (m_header->BPB_TotSec32 * m_header->BPB_BytsPerSec != m_mapped_file.size()) {
            throw Fat32Exception("File size mismatch");
        }
    }

    void print_header_info(const std::string& filename) const {
        std::cout << filename << ": DOS/MBR boot sector, ";
        std::cout << "OEM-ID \"";
        // 安全打印OEM名称（可能不是null终止的）
        for (int i = 0; i < 8 && m_header->BS_OEMName[i] != 0; ++i) {
            std::cout << static_cast<char>(m_header->BS_OEMName[i]);
        }
        std::cout << "\", ";
        std::cout << "sectors/cluster " << static_cast<int>(m_header->BPB_SecPerClus) << ", ";
        std::cout << "sectors " << m_header->BPB_TotSec32 << ", ";
        std::cout << "sectors/FAT " << m_header->BPB_FATSz32 << ", ";
        std::cout << "serial number 0x" << std::hex << m_header->BS_VolID << std::dec << "\n";
    }

    u32 next_cluster(u32 cluster_num) const {
        const u32* fat_table = reinterpret_cast<const u32*>(reinterpret_cast<const u8*>(m_header) +
                                                            m_header->BPB_RsvdSecCnt * m_header->BPB_BytsPerSec);
        if (cluster_num >= m_header->BPB_FATSz32 * 128) { // 近似检查
            throw Fat32Exception("Cluster number out of range");
        }
        return fat_table[cluster_num] & 0x0FFFFFFF; // 清除高4位
    }

    const void* cluster_to_sector(u32 cluster_num) const {
        u32 data_sector = m_header->BPB_RsvdSecCnt + m_header->BPB_NumFATs * m_header->BPB_FATSz32;
        data_sector += (cluster_num - 2) * m_header->BPB_SecPerClus;
        return reinterpret_cast<const u8*>(m_header) + data_sector * m_header->BPB_BytsPerSec;
    }

    std::string get_filename(const DirectoryEntry* entry) const {
        std::string filename;
        filename.reserve(12); // 8.3格式
        // 解析8.3文件名格式
        for (int i = 0; i < 8; ++i) {
            if (entry->DIR_Name[i] != ' ') {
                filename += static_cast<char>(entry->DIR_Name[i]);
            }
        }
        // 添加扩展名（如果有）
        if (entry->DIR_Name[8] != ' ') {
            filename += '.';
            for (int i = 8; i < 11; ++i) {
                if (entry->DIR_Name[i] != ' ') {
                    filename += static_cast<char>(entry->DIR_Name[i]);
                }
            }
        }
        return filename;
    }

    // 获取簇块范围
    ClusterRange get_cluster_range(u32 start_cluster) const {
        if (start_cluster == 0 || start_cluster >= CLUSTER_INVALID) {
            return ClusterRange(0, 0);
        }
        u32 cur = start_cluster;
        u32 first_cluster = start_cluster;
        u32 last_cluster = start_cluster;
        u32 cluster_count = 1;
        // 遍历簇链，找到连续或非连续的簇块范围
        while (true) {
            u32 next = next_cluster(cur);
            // 检查是否到达簇链末尾或无效簇
            if (next >= CLUSTER_INVALID || next == 0) {
                break;
            }
            // 如果是连续的簇，更新结束簇号
            if (next == last_cluster + 1) {
                last_cluster = next;
            } else {
                // 不连续，但我们继续统计总数
                last_cluster = next;
            }
            ++cluster_count;
            cur = next;
            // 安全限制，避免无限循环
            if (cluster_count > 100000) {
                break;
            }
        }
        return ClusterRange(first_cluster, last_cluster);
    }

    void dfs(u32 cluster_id, int depth, bool is_directory) const {
        for (; cluster_id < CLUSTER_INVALID; cluster_id = next_cluster(cluster_id)) {
            if (is_directory) {
                int entries_per_cluster = m_header->BPB_BytsPerSec * m_header->BPB_SecPerClus / sizeof(DirectoryEntry);
                for (int i = 0; i < entries_per_cluster; ++i) {
                    const DirectoryEntry* entry =
                        reinterpret_cast<const DirectoryEntry*>(cluster_to_sector(cluster_id)) + i;
                    // 跳过空条目、删除的条目和隐藏条目
                    if (entry->DIR_Name[0] == 0x00 || entry->DIR_Name[0] == 0xE5 ||
                        (entry->DIR_Attr & static_cast<u8>(Attribute::HIDDEN))) {
                        continue;
                    }

                    std::string filename = get_filename(entry);
                    // 获取文件/目录的簇块范围
                    u32 data_cluster = entry->DIR_FstClusLO | (entry->DIR_FstClusHI << 16);
                    ClusterRange range = get_cluster_range(data_cluster);
                    // 缩进显示目录结构
                    std::cout << std::string(depth * 4, ' ');
                    std::cout << "[" << filename << "]";
                    if (entry->DIR_Attr & static_cast<u8>(Attribute::DIRECTORY)) {
                        std::cout << " <DIR>: " << range.to_string() << "\n";
                        // 跳过"."和".."目录的递归
                        if (filename != "." && filename != "..") {
                            dfs(data_cluster, depth + 1, true);
                        }
                    } else {
                        std::cout << " " << entry->DIR_FileSize << " bytes: " << range.to_string() << "\n";
                    }
                }
            } else {
                // 非目录文件的处理
                ClusterRange range = get_cluster_range(cluster_id);
                std::cout << "## " << cluster_id << ": " << range.to_string() << "\n";
            }
        }
    }
};

} // namespace fat32

int main(int argc, char* argv[]) {
    if (argc < 2) {
        std::cerr << "Usage: " << argv[0] << " fs-image\n";
        return 1;
    }

    try {
        fat32::Fat32Parser parser(argv[1]);
        parser.traverse_filesystem();
    } catch (const fat32::Fat32Exception& e) {
        std::cerr << "Error: " << e.what() << "\n";
        return 1;
    } catch (const std::exception& e) {
        std::cerr << "Unexpected error: " << e.what() << "\n";
        return 1;
    }

    return 0;
}
```