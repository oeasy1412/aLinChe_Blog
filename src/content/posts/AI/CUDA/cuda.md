---
title: CUDA初识
published: 2025-11-11
description: CUDA初识
tags: [CUDA, 内存管理]
category: CUDA
draft: false
---
在当今计算密集型的世界里，无论是科学计算、人工智能还是大数据分析，我们对计算能力的需求都呈指数级增长。传统的CPU虽然在通用计算方面表现出色，但在处理大规模并行任务时却显得力不从心。这时，GPU（图形处理器）以其强大的并行处理能力脱颖而出，而CUDA(`Compute Unified Device Architecture`，计算统一设备架构)正是释放这股力量的钥匙。

## CUDA的意义和价值
CUDA是由NVIDIA推出的并行计算平台和编程模型。它允许开发者使用C++、Fortran等高级语言来利用GPU的计算能力，从而显著加速应用程序。

其核心价值在于：
*   **大规模并行性**：GPU拥有数千个计算核心，能够同时执行数千个线程，这使得它在处理数据并行(`Data-Paralle`)问题时具有无与伦比的优势。
*   **高内存带宽**：GPU配备了高带宽内存（如GDDR6或HBM），可以更快地读写数据，这对于数据密集型应用至关重要。
*   **简化的编程模型**：CUDA提供了**相对友好**的编程接口，让开发者可以专注于算法逻辑，而无需深入了解GPU底层硬件的复杂细节，从零开始学习晦涩的硬件指令。。
*   **丰富的生态系统**：经过多年的发展，CUDA已经拥有了成熟的工具链、库（如cuBLAS, cuDNN, Thrust）和庞大的开发者社区，极大地降低了开发门槛。

简而言之，CUDA是连接软件与GPU硬件之间的桥梁，让我们可以将计算密集型任务从CPU卸载到GPU，实现数量级的性能提升。

## CUDA核心关键字解析
要编写CUDA程序，首先需要理解几个核心的关键字，它们定义了代码在何处执行以及数据的可见范围。
*   `__global__`:
    *   **含义**: 这是一个函数类型限定符，用于声明一个**内核函数(Kernel Function)**。
    *   **执行位置**: 内核函数在GPU上执行，但由CPU代码调用。
    *   **调用方式**: 使用特殊的三重尖括号语法 `<<<...>>>` 来配置执行参数（如线程网格和线程块的维度）。
    *   **特性**: __global__函数的返回类型必须是 void。（这点很好理解，类比 lambda）
*   `__device__`:
    *   **含义**: 函数类型限定符，表示该函数只能在GPU上执行，并且只能被 `__global__` 或其他的 `__device__` 函数调用。
    *   **执行位置**: GPU。
    *   **价值**: __device__函数是实现GPU代码模块化和复用的基本工具，类似于常规C++中的普通函数。
*   `__shared__`:
    *   **含义**: 变量类型限定符，用于声明**共享内存(Shared Memory)**。
    *   **生命周期与可见性**: `__shared__` 内存由同一个线程块(Block)内的所有线程共享，且具有原子性。它的生命周期与线程块相同，当线程块执行完毕后，其占用的共享内存将被释放（和寄存器共用一块L1缓存）。
    *   **价值**: 共享内存位于GPU芯片上，访问速度远快于全局显存(Global Memory)，几乎与寄存器访问速度相当。它是实现线程块内线程高效通信和数据交换的关键，是许多高性能算法**优化的核心**。
*   `__constant__`:
    *   **含义**: 变量类型限定符，用于声明**常量内存(Constant Memory)**。
    *   **特性**: 常量内存对于GPU上的所有线程都是**只读**的。CPU可以写入数据到常量内存，但GPU内核函数不能修改它。
    *   **价值**: 常量内存有特殊的缓存机制：当一个线程束(`Warp` 32 threads)中的所有线程都访问常量内存的同一地址时，会触发一次广播，数据会同时发送给所有线程，效率极高。因此，它非常适合存储那些在内核执行期间不变的配置参数。

## CUDA线程层级：Grid > Block > Thread
理解CUDA的线程组织方式是编写高效内核的关键。CUDA以三层结构组织线程：
1.  **Grid (线程网格)**: 一次内核启动（Kernel Launch）的最高层级，由多个线程块组成。
2.  **Block (线程块)**: 一个Grid的组成单位，由多个线程组成。**同一个Block内的所有线程可以相互协作**，例如通过共享内存（`__shared__`）和同步操作（`__syncthreads()`）。
3.  **Thread (线程)**: 最基本的执行单元。

通过 `<<<gridDim, blockDim, sharedMemSize, stream>>>` 语法来配置这些维度。例如 `forkernel<<<dim3(20), dim3(1024), 0, stream1>>>`：
*   `dim3(20)`: 指定了Grid的维度。这里是一个一维Grid，包含20个Block。
*   `dim3(1024)`: 指定了每个Block的维度。这里是一个一维Block，包含1024个Thread。
*   `0`: 指定了动态分配的共享内存大小（以字节为单位），这里是0。
*   `stream1`: 指定了该内核在哪个CUDA流上执行。

在内核函数内部，我们可以通过内置变量来获取当前线程的位置信息：
*   `gridDim`: Grid的维度（`gridDim.x`, `gridDim.y`, `gridDim.z`）。
*   `blockDim`: Block的维度（`blockDim.x`, `blockDim.y`, `blockDim.z`）。
*   `blockIdx`: 当前线程所在Block在Grid中的索引（`blockIdx.x`, ...）。
*   `threadIdx`: 当前线程在Block内的索引（`threadIdx.x`, ...）。

---

## 代码深度解析
### 1.Grid-Stride循环 (网格步长循环)

在 `forkernel` 函数中，我们使用了这样一种循环模式：
```cpp
__global__ void forkernel(cuda::std::span<int> arr) {
    for (int i = blockIdx.x * blockDim.x + threadIdx.x; i < arr.size(); i += gridDim.x * blockDim.x) {
        arr[i] = arr[i] * arr[i];
    }
}
```
*   **`int i = blockIdx.x * blockDim.x + threadIdx.x;`**: 这一行计算了一个**全局唯一**的线程ID。每个线程启动时，都会基于它在Grid和Block中的位置，计算出一个初始的索引 `i`，用于处理数据中的一个元素。
*   **`i < arr.size()`**: 检查索引是否越界。
*   **`i += gridDim.x * blockDim.x`**: 这是该模式的核心。`gridDim.x * blockDim.x` 是本次内核启动所创建的总线程数。当一个线程处理完第 `i` 个元素后，它会跳过总线程数的距离，去处理下一个元素，即按网格总线程数进行步进。

**为什么这种模式如此重要？**
1.  **可扩展性**: 它将线程数量与数据大小解耦。即使您启动的线程总数少于要处理的数据元素总数，该循环也能确保所有数据都被处理到。
2.  **灵活性**: 您可以自由调整Grid和Block的大小，而无需修改内核逻辑，这对于性能调优非常方便。

### 2. 内存层次与执行模型
```cpp
// 所有的局部变量等于一个4B寄存器，如果寄存器不足则会放在全局显存(Global Memory)中，即memory-bound，
// 且需要注意所有取引用操作（buf[i]）都会导致变量只能放到内存，因为寄存器无地址的概念！（即GPU不能像CPU一样用引用减小开销，大多数情况下反而增大开销）
// 如果需要访问缓存导致memwait，则SM会去执行下一个warp（32个连续线程组成的固定大小执行单元，SIMT），实现隐藏延迟（类比异步I/O）
// 所以 mem-bound 需要减少 blockDim, compute-bound 需要增加 blockDim (类比CPU是否选择超线程)
```

*   **局部变量 与 寄存器**: 在内核函数中声明的普通局部变量（如 int i，其实类似于 thread_local 因为GPU上的最小变量就是thread），编译器会优先将它们分配到**寄存器(Register)**中。寄存器是GPU上最快的存储单元，每个线程私有。但是寄存器数量有限，如果局部变量过多或者体积过大（例如一个大数组），编译器就会将它们“溢出”到**局部内存(Local Memory)**中。局部内存实际上是全局显存的一部分，访问速度很慢。
*   **取引用操作**: “取引用操作（如arr[i]）会导致变量只能放到内存”。这是因为**寄存器没有地址的概念**，无法对其进行引用或指针操作。

*   **线程束(`Warp`) 与 流多处理器(`SM`)**: GPU的执行模型是 **`SIMT`(Single-Instruction, Multiple-Thread，单指令多线程)**。`Warp`是`SM`(`Streaming Multiprocessor`)上的最基本执行单位，会以32个线程为一组来调度，这一组被称为一个**线程束(`Warp`)**。一个Warp中的32个线程在同一时刻执行 **`完全相同的指令`**，这与CPU的SIMD（单指令多数据）在思想上高度一致。
    *   例如，一个 dim3(1024) 的Block，在硬件层面会被看作是 1024/32=32 个Warp。如果一个Block的线程数不是32的倍数，比如100，它将被划分为 ceil(100/32)=4 个Warp，其中最后一个Warp会有 28 个线程是“非活动”的，但它们依然会占用调度资源！（可以把Warp类比为一辆只能坐32个工人的车，SM按车来派活）
*   **线程束分化(Warp Divergence) 与 `Mask`**: 如果Warp内发生分支（如 if-else），且不同线程进入不同分支，就会产生线程束分化。此时，硬件会依次执行每一个分支路径，并通过`Mask`**禁用不在当前路径上的线程**。这会导致部分计算资源被闲置，应尽量避免（但其实并没有CPU流水线那么担心分支问题，因为GPU没有像CPU那样复杂的乱序执行和分支预测流水线，其代价是**可预期的串行化执行**）
*   **内存等待(Memory Wait) 与 延迟隐藏(Latency Hiding)**: 当一个Warp执行访存指令（如 `arr[i] = ...`）时，需要从显存中读取或写入数据。这个过程有很高的延迟。为了隐藏这些延迟，SM会立刻切换到另一个**准备就绪(Ready)**的Warp去执行计算指令，而不是原地等待。只要有足够多的活动Warp**可供调度**，SM的计算单元就能始终保持忙碌，从而掩盖了访存延迟。这就是GPU实现高吞吐量的关键。

*   **性能调优启发式规则**: 资源占用率 & 延迟隐藏
    *   **`Memory-Bound`**:
        *   **瓶颈**：等待数据从全局显存中读写。
        *   **策略**：**最大化SM上的活动Warp总数**（即提高占用率Occupancy），以最大限度地**隐藏内存延迟**。
        *  适当地减少 blockDim(Block中的线程数) 是实现这一目标的有效手段之一，因为每一个Block需要的寄存器数减少，SM就可能能容纳更多的Block，提高占用率就可能提高了 Warp 数量来隐藏内存延迟。
    *   **`Compute-Bound`**:
        *   **瓶颈**：ALU（算术逻辑单元）的处理速度。
        *   **策略**：为编译器提供最大的**优化空间**，并**减少调度开销**。
        *   在硬件限制内增加 blockDim 是实现这一目标的有效手段之一，一个较大的Block意味着它包含更多的Warp，这使得SM可以在一个Block内部进行大量的Warp切换和指令级并行优化，而无需频繁地加载和卸载Block的上下文。而且还减少了寄存器“溢出”到全局显存的可能性。
    *   P.S. 与CPU超线程的类比：CPU核心开启超线程，是为了在一个物理核心上模拟出两个逻辑核心。当一个线程因为缓存未命中而停顿时，物理核心可以**立即**去执行另一个线程的指令。这与GPU SM通过切换Warp来**隐藏内存延迟**的思想是完全一致的，都是为了榨干计算单元的每一个时钟周期。

*   **统一内存 (Unified Memory)**: 
    *   GPU的MMU也以页(Page)为单位来管理内存。一个页是虚拟地址到物理地址映射的最小单元。不同的是GPU一般采用2MB大页，便于访问大块连续内存，显著降低地址翻译的开销。
    *   GPTE.Dirty

**统一内存的“按需页面迁移 (On-Demand Page Migration)”工作流：**
1.  **初始状态**：`std::vector`在主机端创建，数据物理上在主机RAM中。
2.  **GPU访问(缺页中断)**：
    *   GPU内核首次尝试访问某个地址。
    *   GPU MMU发现这个虚拟地址对应的页面不在VRAM中，触发**缺页中断(Page Fault)**。
    *   `CUDA驱动` 捕获中断，在VRAM中分配一个物理页，将数据从主机RAM拷贝到VRAM。
    *   驱动更新页表，将虚拟地址映射到新的VRAM物理地址。
    *   中断返回，GPU内核从刚才中断的指令处继续执行，此时访问成功。
3.  **GPU写入**：
    *   GPU内核对该页面进行写操作。
    *   硬件自动将该页面的GPTE中的**Dirty位置为1**。
4.  **CPU再次访问**：
    *   当主机代码需要访问这块内存时，`CUDA驱动`会检查页表。
    *   它发现这个页面现在位于VRAM中，并且Dirty位为1。
    *   驱动明白，最新的数据在VRAM里。于是它会把这个页面从VRAM**拷贝回**主机RAM，然后CPU才能安全地访问。

这个机制与现代操作系统的虚拟内存管理几乎如出一辙，只是它协调的是CPU和GPU这两个异构处理器之间的内存一致性。

**优化**: 

### 3. CUDA Stream 的本质是什么？
**本质上，一个CUDA Stream就是一个先进先出(FIFO)的异步任务队列**。

1.  **发布任务(Asynchronous Call)**：CPU作为生成者，将一个个任务（如 `cudaMemcpyAsync`、内核启动 `kernel<<<...>>>`）放到任务队列上。这些API调用是**异步**的，意味着CPU把任务放入队列后，就**立即返回**，继续执行自己的代码，而不会等待任务完成。
2.  **GPU执行任务**：GPU作为消费者，按顺序取下任务并执行。只要有可用GPU硬件资源，GPU就可以**同时从不同的任务队列上取任务来执行**。
3.  **顺序保证**：在同一个Stream上，任务的**开始执行顺序**严格按照入队顺序进行。

**核心价值**：实现**计算与数据传输的重叠(`Overlap`)**。
这是CUDA性能优化的一个关键技巧，可以：
*   在`stream1`上启动一个数据拷贝任务（CPU -> GPU）。
*   同时，在`stream2`上启动一个计算密集型的内核（它处理的是上一次拷贝来的数据）。
*   同时，在`stream3`上启动一个数据回传任务（GPU -> CPU）。
只要硬件资源允许（比如GPU有独立的Copy Engine和Compute Engine），这三个操作就可以真正地并行执行，从而掩盖数据传输的开销，最大化GPU的利用率。

### 4. warp级原语
在CUDA中，同一个**线程块 (Block)** 内的线程间通信，标准方式是使用 `__shared__` 内存：
1.  线程A将数据写入 `__shared__` 内存中的某个位置。
2.  调用 `__syncthreads()` 确保块内所有线程都完成了写入操作。
3.  线程B从 `__shared__` 内存中读取线程A写入的数据。

这个过程有两个开销：
*   **内存延迟**：访问 `__shared__` 内存虽然比全局显存快得多，但仍比访问**寄存器 (Register)** 慢。
*   **同步开销**：`__syncthreads()` 是一个重量级路障，它会暂停整个Block的执行，直到所有线程都到达该点。

然而，在一个Warp（32个线程）内部，由于所有线程在物理上是同步执行的（SIMT），我们其实**不需要**重量级的 `__syncthreads()` 来进行同步。Warp级原语(`Intrinsics`)正是利用了这一点，允许Warp内的线程**直接通过寄存器网络交换数据**，完全绕过了共享内存。这带来了极致的速度。

#### **`__shfl_sync`**
`__shfl_sync` (Shuffle Sync) 是最常用的Warp级原语之一。它的目的是：让一个Warp内的线程可以读取**同一个Warp内其他任意线程**的寄存器中的值。
```cpp
T __shfl_sync(unsigned mask, T var, int srcLane, int width = warpSize); // = __syncwarp(); + __shfl(...);
```
*   `unsigned mask`: 这是理解 `_sync` 后缀的关键。它是一个32位的掩码，明确指定了**哪些线程必须参与这次shuffle操作**。只有在`mask`中对应位为1的线程，才会参与数据交换。这也是一个**同步保证**：函数会等待`mask`中指定的所有线程都执行到这个`__shfl_sync`调用点（即`__syncwarp()`），然后才进行数据交换。
*   `T var`: 当前线程想要“广播”或“发送”出去的变量。这个变量必须存在于寄存器中。
*   `int srcLane`: **源线程**在Warp内的ID（也叫Lane ID，范围是0-31）。当前线程执行这个函数后，会得到来自`srcLane`号线程的`var`变量的值。（类比`_MM_SHUFFLE()`）

#### **`__activemask()`**
`__activemask()` 是一个内建函数，它返回一个32位的`unsigned int`，代表当前Warp中**所有活动线程**的掩码。如果Warp中的第`i`号线程是活动的（即没有因为分支而被Mask禁用），那么返回值的第`i`位就是1，否则是0。
将`__activemask()`作为`mask`参数传递给`__shfl_sync`，是最常见和安全的做法。它告诉shuffle指令：“请在当前所有真正参与计算的线程之间进行数据交换”。

#### **数组反转代码示例（硬编码版本）**
```cpp
__global__ void forkernel(cuda::std::span<int> arr) {
    const int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i > arr.size() / 2) {
        return;
    }
    const int warpCnt = threadIdx.x / warpSize;
    const int warpIdx = threadIdx.x % warpSize;
    const int j = arr.size() - blockIdx.x * blockDim.x - warpCnt * warpSize - warpSize + warpIdx;
    int tmp = arr[j];
    arr[j] = __shfl_sync(__activemask(), arr[i], warpSize - 1 - warpIdx, warpSize);
    arr[i] = __shfl_sync(__activemask(), tmp, warpSize - 1 - warpIdx, warpSize);
}
```

### 5. 共享内存 Bank Conflict**
为了实现高带宽访问，`__shared__`内存并不是一整块连续的存储器。在物理上，它被划分为**32个等宽的存储体**(与Warp线程数一致)，称为**Bank**。这些Bank是**低位交叉编址**的。对于32位宽的数据，连续的32个`int`值会分别存入32个不同的Bank中。（类比DRAM低位交叉编址）

当一个Warp（32个线程）发起一次访存请求时，硬件会检查这32个线程要访问的地址。
*   **无冲突并行访问**：32个线程访问的地址，恰好落在了**32个不同的Bank**上，那么所有请求可在一个时钟周期内并行处理。
*   **广播访问**：32个线程访问同一个Bank中的同一个地址，数据被广播给所有请求线程，只需一次传输。
*   **存储体冲突**：如果有多个线程访问的地址，落在了**同一个Bank**上，就发生了`Bank Conflict`，硬件别无选择，只能**串行处理**对这个Bank的访问。
    *   **n-way Bank Conflict**：如果有n个线程访问同一个Bank，就需要n个周期。

最常用的技巧是**内存填充 (Padding)**。
我们将共享内存数组的宽度从32改为33：
```cpp
__shared__ int tile[32][33];
```
现在，`tile[row][col]`的字地址是 `row * 33 + col`。
它对应的Bank是 `(row * 33 + col) % 32` = `(row * (32+1) + col) % 32` = `(row + col) % 32`。

当Warp 0 (线程0-31)按列访问 `tile[t][some_col]` 时：
*   线程0 访问的Bank是 `(0 + some_col) % 32`
*   线程1 访问的Bank是 `(1 + some_col) % 32`
*   线程2 访问的Bank是 `(2 + some_col) % 32`
*   ...

现在，32个线程访问的Bank ID变成了 `some_col`, `some_col+1`, `some_col+2`, ...，它们各不相同。**Bank Conflict就此消除**。我们只是付出了一点点额外的共享内存空间，就换来了巨大的性能提升。

### 参考代码：
```cpp
// nvcc -g -std=c++20 --extended-lambda main.cu && ./a.out
#include "cudapp.cuh"

#include <cstdio>
#include <cstdlib>
#include <cuda/std/span>
#include <cuda_runtime.h>
#include <nvfunctional>
#include <vector>

using namespace cudapp;

template <class Index, class Func>
__global__ void paralelFor(Index n, Func func) {
    for (Index i = blockIdx.x * blockDim.x + threadIdx.x; i < n; i += gridDim.x * blockDim.x) {
        func(i); // GPTE.Dirty
    }
}

int main() {
    // 400MB, 1亿+int, 在GPU上初始化，并使用统一内存
    std::vector<int, CudaAllocator<int, CudaManagedArena>> arr(100 << 20);
    int n = arr.size();
    printf("arr.size() = %d\n", n);
    CudaStream stream1 = CudaStream::Builder().withNonBlocking().build();
    paralelFor<<<dim3(10), dim3(1024), 0, stream1>>>(arr.size(), [arr = arr.data()] __device__(int i) { arr[i] = i; });
    CudaEvent st = stream1.recordEvent();
    paralelFor<<<dim3(10), dim3(1024), 0, stream1>>>(
        arr.size() / 2, [arr = cuda::std::span<int>(arr.data(), arr.size())] __device__(int i) {
            cuda::std::swap(arr[i], arr[arr.size() - i - 1]); // 内存密集型，可以适当提高blockDim
        });
    CudaEvent ed = stream1.recordEvent();
    CHECK_CUDA(cudaGetLastError()); // 检查内核启动是否成功
    // while (!ed.poll()) {
    //     printf("waiting...\n");
    // }
    stream1.join();
    printf("time bigkernel: %fms\n", ed - st);
    paralelFor<<<dim3(10), dim3(1024), 0, stream1>>>(
        arr.size(), [arr = cuda::std::span<int>(arr.data(), arr.size())] __device__(int i) {
            if (arr[i] != arr.size() - i - 1) [[unlikely]]
                printf("%d ", arr[i]);
        });
    // int a;
    // scanf("%d", &a);
    // watch -n 1 nvidia-smi
}
```

### else 
```sh
# GTX 1060 5GB显卡
cuda-samples 1_Utilities deviceQuery deviceQuery
deviceQuery Starting...

 CUDA Device Query (Runtime API) version (CUDART static linking)

Detected 1 CUDA Capable device(s)

Device 0: "NVIDIA GeForce GTX 1060 5GB"
  CUDA Driver Version / Runtime Version          12.2 / 12.9
  CUDA Capability Major/Minor version number:    6.1  # 计算能力6.1
  Total amount of global memory:                 5053 MBytes (5298716672 bytes)
  (010) Multiprocessors, (128) CUDA Cores/MP:    1280 CUDA Cores  # 10SM × 128核心/SM
  GPU Max Clock rate:                            1709 MHz (1.71 GHz)
  Memory Clock rate:                             4004 Mhz
  Memory Bus Width:                              160-bit
  L2 Cache Size:                                 1310720 bytes  # 1.25M L2
  Maximum Texture Dimension Size (x,y,z)         1D=(131072), 2D=(131072, 65536), 3D=(16384, 16384, 16384)
  Maximum Layered 1D Texture Size, (num) layers  1D=(32768), 2048 layers
  Maximum Layered 2D Texture Size, (num) layers  2D=(32768, 32768), 2048 layers
  Total amount of constant memory:               65536 bytes
  Total amount of shared memory per block:       49152 bytes
  Total shared memory per multiprocessor:        98304 bytes
  Total number of registers available per block: 65536  # 每个Block可用寄存器 65536个
  Warp size:                                     32     # 32 threads
  Maximum number of threads per multiprocessor:  2048   # 每 SM 最大线程  2048 (64个Warp)
  Maximum number of threads per block:           1024   # 每Block最大线程 1024 (32个Warp)
  Max dimension size of a thread block (x,y,z): (1024, 1024, 64)
  Max dimension size of a grid size    (x,y,z): (2147483647, 65535, 65535)
  Maximum memory pitch:                          2147483647 bytes
  Texture alignment:                             512 bytes
  Concurrent copy and kernel execution:          Yes with 2 copy engine(s) # 2个拷贝引擎并行工作
  Run time limit on kernels:                     No
  Integrated GPU sharing Host Memory:            No
  Support host page-locked memory mapping:       Yes
  Alignment requirement for Surfaces:            Yes
  Device has ECC support:                        Disabled
  Device supports Unified Addressing (UVA):      Yes  # 统一虚拟地址
  Device supports Managed Memory:                Yes  # 自动CPU-GPU数据传输
  Device supports Compute Preemption:            Yes  # 计算抢占
  Supports Cooperative Kernel Launch:            Yes  # 多GPU协同计算
  Supports MultiDevice Co-op Kernel Launch:      Yes
  Device PCI Domain ID / Bus ID / location ID:   0 / 35 / 0
  Compute Mode:
     < Default (multiple host threads can use ::cudaSetDevice() with device simultaneously) >

deviceQuery, CUDA Driver = CUDART, CUDA Driver Version = 12.2, CUDA Runtime Version = 12.9, NumDevs = 1
Result = PASS  # CUDA驱动(12.2)与运行时(12.9)版本兼容，所有硬件功能正常启用
```