---
title: SIMD初识
published: 2025-11-08
description: 从MMX、SSE到AVX2的性能飞跃
tags: [CUDA, SIMD, 编译原理]
category: CUDA
draft: false
---
> 在追求极致性能的计算世界里，榨干CPU的每一分潜力是开发者永恒的目标。常规的单指令单数据（SISD）处理模式早已无法满足现代应用对数据吞吐量的渴求，尤其是在图形渲染、科学计算、机器学习和多媒体处理等领域。于是，单指令多数据（SIMD）技术应运而生，让CPU能够用一条指令同时处理多个数据，实现了数据层面的并行计算，带来了革命性的性能提升。

### 1. SIMD的意义：数据并行的力量
SIMD(`Single Instruction, Multiple Data`): 现代CPU内部都设有专门的**向量处理单元**(VPU)和**宽位寄存器**。例如，一个128位的寄存器可以同时容纳4个32位的浮点数或整数，一条SIMD加法指令就能一次性完成这4对数字的相加操作，理论上将性能提升至4倍。

### 2. MMX的历史地位：先行者的探索
- 1997年，英特尔推出了其首个SIMD指令集——MMX(Multi-Media Extension)。 MMX旨在加速多媒体和通信应用，引入了64位寄存器和57条新指令，只用于整数运算。

然而，MMX的设计存在一个关键缺陷：它复用了已有的x87(最早的浮点协处理器 8087, 1979年)浮点运算单元(FPU)的寄存器。 这意味着，程序无法同时执行MMX和浮点运算，两者之间的切换需要耗时的状态管理，这极大地限制了它的应用场景。尽管MMX在商业上不算非常成功，但它作为SIMD技术的先行者，为后续更成熟的指令集铺平了道路，其历史地位不容忽视。

### 3. SSE的知识盛宴与性能剖析
为了解决MMX的弊端，英特尔在1999年随奔腾III处理器推出了**SSE**(`Streaming SIMD Extensions`)。 **SSE**的出现是SIMD发展史上的一个里程碑，它带来了根本性的改进：
*   **独立的128位寄存器**：SSE引入了8个全新的128位寄存器（XMM0-XMM7），后续版本扩展至16个（XMM0-XMM15）。这彻底解决了与FPU的冲突，让SIMD计算和浮点计算可以并行不悖。
*   **浮点数支持**：SSE原生支持单精度浮点数（32位）的并行计算，这对于图形学和科学计算至关重要。
*   **更丰富的指令集**：SSE及其后续版本（SSE2, SSE3, SSSE3, SSE4.1, SSE4.2）不断扩充指令，支持了双精度浮点数、更全面的整数类型以及各种数据处理指令。

#### 代码实战：深入理解SSE Intrinsics
要驾驭SIMD的强大力量，我们通常不直接编写汇编代码，而是使用编译器提供的**Intrinsics**函数。 这些函数看起来像C/C++函数，但它们会由编译器直接翻译成一条或几条对应的机器指令，因此几乎没有函数调用的开销。

让我们通过分析你提供的代码来逐一拆解SSE的核心操作。这段代码需要使用`-msse4.1`这样的编译器标志来启用对应的指令集。

##### 3.1 code
- [Intel SSE 手册](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html?wapkw=intrinsics%20guide&techs=SSE_ALL#techs=SSE_ALL) （含性能分析）
```cpp
// g++ -g -std=c++20 -msse4.1 main.cpp && ./a.out
#pragma GCC diagnostic ignored "-Wignored-attributes"
#include <iomanip>
#include <iostream>
// 下面头文件从下到上为包含关系
// #include <mmmintrin.h> // MMX
#include <xmmintrin.h> // SSE
// #include <emmintrin.h> // SSE2
// #include <pmmintrin.h> // SSE3
// #include <tmmintrin.h> // SSSE3
// #include <smmintrin.h> // SSE4.1
// #include <nmmintrin.h> // SSE4.2
// #include <immintrin.h> // AVX/AVX2/... (建议)
// #include <x86intrin.h> // auto(GCC)

using namespace std;

#define IS_SIMD_TYPE(T, SIMD_TYPE) (is_same_v<remove_cv_t<remove_reference_t<T>>, SIMD_TYPE>)
template <class T>
void print_m128_32(T v, int line = 0) {
    if constexpr (IS_SIMD_TYPE(T, __m128)) {
        const float* f = reinterpret_cast<const float*>(&v);
        cout << fixed << setprecision(6) << "[" << f[0] << ", " << f[1] << ", " << f[2] << ", " << f[3] << "]\n";
    } else if constexpr (IS_SIMD_TYPE(T, __m128i)) {
        const int* i = reinterpret_cast<const int*>(&v);
        cout << "[" << i[0] << ", " << i[1] << ", " << i[2] << ", " << i[3] << "]\n";
    } else {
        const char* file = __FILE__;
        printf("unk vec %s:%d\n", file, line);
    }
}
template <class T>
void print_m128_16(T v, int line = 0) {
    if constexpr (IS_SIMD_TYPE(T, __m128i)) {
        const short* i = reinterpret_cast<const short*>(&v);
        cout << "[" << i[0] << ", " << i[1] << ", " << i[2] << ", " << i[3] //
             << ", " << i[4] << ", " << i[5] << ", " << i[6] << ", " << i[7] << "]\n";
    } else {
        const char* file = __FILE__;
        printf("unk vec %s:%d\n", file, line);
    }
}
template <class T>
void print_m128_64(T v, int line = 0) {
    if constexpr (IS_SIMD_TYPE(T, __m128d)) {
        const double* d = reinterpret_cast<const double*>(&v);
        cout << fixed << setprecision(6) << "[" << d[0] << ", " << d[1] << "]\n";
    } else {
        const char* file = __FILE__;
        printf("unk vec %s:%d\n", file, line);
    }
}

#define PRINT32(v) print_m128_32(v, __LINE__)
#define PRINT16(v) print_m128_16(v, __LINE__)
#define PRINT64(v) print_m128_64(v, __LINE__)

// 历史遗留问题：MMX 第一个 SIMD 指令集只支持整数运算。故 SSE(Streaming SIMD Extensions)的叫epi
// mm: MultiMedia Extension
// ps: Packed Single-precision floating-point, pd: Packed Double-precision floating-point
// si: Signed Integer
// epu: Extended Packed Unsigned integer
int main() {
    {
        // 基础运算与数据类型 and 类型转换与舍入模式
        float x = 1.5, y = 2.5, z = 2.6, w = -3.5;
        __m128 m = _mm_setr_ps(x, y, z, w);
        __m128 one = _mm_set1_ps(1.0f);
        m = _mm_add_ps(m, one);
        m = _mm_sub_ps(m, one);
        m = _mm_mul_ps(m, one);
        PRINT32(m);
        __m128i mi = _mm_castps_si128(m); // 无生成指令，按位读取
        PRINT32(mi);
        _MM_SET_ROUNDING_MODE(_MM_ROUND_NEAREST); // 默认，四舍五往偶数取整六入（考虑对全.5的数据不那么影响期望）
        // _MM_SET_ROUNDING_MODE(_MM_ROUND_DOWN);
        // _MM_SET_ROUNDING_MODE(_MM_ROUND_UP);
        // _MM_SET_ROUNDING_MODE(_MM_ROUND_TOWARD_ZERO);
        mi = _mm_cvtps_epi32(m);
        PRINT32(mi);
        mi = _mm_cvttps_epi32(m); // convert truncate: _MM_ROUND_TOWARD_ZERO
        PRINT32(mi);
    }
    {
        __m128i m = _mm_setr_epi16(1, 2, 3, 4, 5, 6, 7, 8);
        __m128i two = _mm_set1_epi16(2);
        __m128i tmp;
        tmp = _mm_mulhi_epi16(m, two);
        PRINT16(tmp);
        tmp = _mm_mullo_epi16(m, two);
        PRINT16(tmp);
    }
    {
        // 广播 与 Shuffle
        __m128i mi = _mm_setr_epi32(1, 2, 3, 4);
        mi = _mm_shuffle_epi32(mi, _MM_SHUFFLE(3, 2, 1, 0)); // 要反着看（
        PRINT32(mi);
        mi = _mm_shuffle_epi32(mi, _MM_SHUFFLE(2, 2, 0, 1)); // 这个参数是立即数，只能填常量
        PRINT32(mi);
        mi = _mm_shuffle_epi32(mi, 0x55); // 广播第二个元素(索引为1)
        PRINT32(mi);

        __m128 m1 = _mm_setr_ps(1, 2, 3, 4);
        __m128 m2 = _mm_setr_ps(5, 6, 7, 8);
        __m128 m = _mm_shuffle_ps(m1, m2, _MM_SHUFFLE(3, 2, 1, 0));
        PRINT32(m);
        m = _mm_shuffle_ps(m1, m2, _MM_SHUFFLE(3, 1, 2, 0));
        PRINT32(m);
        printf("\n");
        // 向量水平求和: m1.sum()
        auto horizontal_sum = [](__m128 v) -> float {
            __m128 tmp = _mm_shuffle_ps(v, v, _MM_SHUFFLE(2, 3, 0, 1));
            __m128 sums = _mm_add_ps(v, tmp);
            tmp = _mm_shuffle_ps(sums, sums, _MM_SHUFFLE(1, 0, 3, 2));
            return _mm_cvtss_f32(_mm_add_ss(sums, tmp));
        };
        __m128 tmp = _mm_shuffle_ps(m1, m1, _MM_SHUFFLE(2, 3, 0, 1));
        PRINT32(tmp);
        tmp = _mm_add_ps(m1, tmp);
        PRINT32(tmp);
        m1 = _mm_shuffle_ps(tmp, tmp, _MM_SHUFFLE(1, 0, 3, 2));
        m1 = _mm_add_ps(m1, tmp);
        PRINT32(m1);
        float sum = _mm_cvtss_f32(m1);
        printf("m1 sum = %f\n", sum);
        printf("m2 sum = %f\n\n", horizontal_sum(m2));
    }
    {
        // Load and Store
        // 高效构造 f[]
        int _space = 0;
        float f[4] = {1, 2, 3, 4};
        alignas(32) __m128 m = _mm_loadu_ps(f); // _mm_load_ps(), _mm_load_ss(), _mm_load1_ps()
        PRINT32(m);
        float a[4];
        _mm_storeu_ps(a, m);
        for (float f : a) {
            printf(" %f ", f);
        }
    }
}
```

##### 3.2 `-msse4.1`函数方法性能开销探讨
这是一个开发者普遍关心的问题：使用 SSE intrinsics 和相关的编译器标志会带来多大的开销？

1.  **编译器标志 (`-msse4.1`) 的作用**：这个标志本身**不产生运行时开销**。它的作用是告知编译器，你的目标CPU支持SSE4.1指令集。这使得编译器在代码生成阶段可以自由使用这些高效的指令。 如果不开启此标志，编译器要么无法识别intrinsic函数，要么可能会用一系列低效的标量指令来模拟它，导致性能大幅下降。
2.  **Intrinsic函数的开销**：如前所述，Intrinsic函数并非真正的函数调用。它们是给编译器的“内联提示”，会被直接替换成相应的汇编指令。 因此，**不存在传统意义上的函数调用开销**（如压栈、跳转等）。其成本就是底层汇编指令本身的成本，包括指令的**延迟(Latency)**和**吞吐量(Throughput)**（e.g. _mm_add_ps() Latency=4, Throughput=0.5CPI）。
    *   **延迟**：指一条指令从开始执行到结果可用的时间。
    *   **吞吐量(`CPI`)**：指CPU每个时钟周期可以开始执行多少条同类指令。
    *   在`horizontal_sum`的例子中，数据之间存在依赖关系（后一次加法依赖前一次的结果），此时性能瓶颈主要受延迟影响。而在没有数据依赖的大规模循环中，吞吐量则更为关键。
3.  **潜在的性能陷阱**：
    *   **内存对齐**：像`_mm_load_ps`这样的指令要求内存地址是16字节对齐的。如果地址不对齐，程序会崩溃。代码中使用的`_mm_loadu_ps`（`u`代表unaligned）可以在不对齐的内存上工作，但在旧架构的CPU上可能会有性能损失。现代CPU对非对齐加载的惩罚已经很小，但为了极致性能，数据对齐仍是一个好习惯。但从工程稳定性来说，可以使用_mm_loadu_ps方法保证程序稳定性（
    *   **SSE/AVX混合惩罚**：这是一个非常关键且隐蔽的性能杀手。当代码中同时包含256位的AVX指令和传统的（非VEX编码的）SSE指令时，CPU在两者之间切换会产生巨大的性能开销（可能高达几十个时钟周期）。 这是因为CPU需要保存或恢复YMM寄存器的高128位。 解决方法是使用`-mavx`编译整个项目，让编译器为SSE指令也使用VEX编码，或者在AVX代码块之后手动调用`_mm256_zeroupper()`来清空高位，避免切换惩罚。

### 4. AVX2：性能的再次飞跃
如果说SSE是SIMD的成熟之作，那么AVX/AVX2(`Advanced Vector Extensions 2`)就是它的威力加强版，于2011年由Intel推出，并在Haswell架构中引入AVX2。

AVX2带来了两大核心提升：
1.  **256位寄存器**：AVX2将SIMD寄存器宽度从128位(XMM)翻倍到256位(YMM)。 这意味着，一条指令可以同时处理8个单精度浮点数或4个双精度浮点数，理论吞吐量直接翻倍。
2.  **更强的指令集**：
    *   **三操作数指令**：SSE指令通常是`a = a + b`的形式，会破坏一个源操作数。AVX引入了非破坏性的`c = a + b`三操作数形式，减少了不必要的数据拷贝，使代码更高效、更易读。
    *   **FMA (Fused Multiply-Add)**：融合乘加指令（如`_mm256_fmadd_ps`）可以在一个步骤内完成`a * b + c`的操作。这不仅提升了计算密度，还通过单次舍入提高了精度，是科学计算和机器学习中的大杀器。
    *   **完善的整数指令**：AVX2极大地增强了对256位整数向量的支持，弥补了AVX1的不足。
    *   **Gather指令**：允许从非连续的内存位置高效地加载数据到向量寄存器中，对于处理稀疏数据非常有用。

从SSE迁移到AVX2，对于数据密集型应用来说，性能提升通常是显著的。在理想情况下，计算密集且内存带宽充足的应用可以获得接近2倍的性能提升。


### 其他示例代码：
```cpp
// g++ -g -std=c++20 -mavx2 -mfma saxpy.cpp -lbenchmark -lgtest -lopenblas -lpthread && ./a.out
// --- saxpy ---
// __restrict 承诺指针的无内存重叠，无需每次访问都从内存中重新读取
[[gnu::optimize("O2")]] void saxpy1(int n, float a, const float* __restrict x, float* __restrict y) {
    for (int i = 0; i < n; ++i) {
        y[i] = a * x[i] + y[i];
    }
}
// __m128 优化
[[gnu::optimize("O2")]] void saxpy2(int n, float a, const float* __restrict x, float* __restrict y) {
    const __m128 A = _mm_set1_ps(a); // 广播
    int i = 0;
    for (; i < (n & ~3); i += 4) {
        __m128 xi = _mm_loadu_ps(&x[i]);
        __m128 yi = _mm_loadu_ps(&y[i]);
        _mm_storeu_ps(&y[i], _mm_add_ps(_mm_mul_ps(A, xi), yi));
    }
    for (; i < n; ++i) {
        y[i] = a * x[i] + y[i];
    }
}
// __m128 建议要 <=8个(CPU可用矢量寄存器的数量) x86 SSE架构提供了8个128位的XMM寄存器，对“溢出”的变量会被存储到(栈)内存上
[[gnu::optimize("O2")]] void saxpy3(int n, float a, const float* __restrict x, float* __restrict y) {
    const __m128 A = _mm_set1_ps(a); // 广播
    int i = 0;
    for (; i < (n & ~7); i += 8) {
        __m128 xi1 = _mm_loadu_ps(&x[i]);
        __m128 xi2 = _mm_loadu_ps(&x[i + 4]);
        __m128 yi1 = _mm_loadu_ps(&y[i]);
        __m128 yi2 = _mm_loadu_ps(&y[i + 4]);
        _mm_storeu_ps(&y[i], _mm_add_ps(_mm_mul_ps(A, xi1), yi1));
        _mm_storeu_ps(&y[i + 4], _mm_add_ps(_mm_mul_ps(A, xi2), yi2));
    }
    for (; i < n; ++i) {
        y[i] = a * x[i] + y[i];
    }
}

// --- sum ---
float sum(int n, const float* x) {
    __m128 ret1 = _mm_setzero_ps();
    __m128 ret2 = _mm_setzero_ps();
    int i = 0;
    for (; i < (n & ~8); i += 8) {
        ret1 = _mm_add_ps(ret1, _mm_loadu_ps(x + i));
        ret2 = _mm_add_ps(ret2, _mm_loadu_ps(x + i + 4));
    }
    ret1 = _mm_add_ps(ret1, ret2);
    ret1 = _mm_add_ps(ret1, _mm_shuffle_ps(ret1, ret1, _MM_SHUFFLE(2, 3, 0, 1)));
    ret1 = _mm_add_ps(ret1, _mm_shuffle_ps(ret1, ret1, _MM_SHUFFLE(1, 0, 3, 2)));
    float ret = _mm_cvtss_f32(ret1);
    for (; i < n; ++i) {
        ret += x[i];
    }
    return ret;
}

// --- 数组中大于p的个数 ---
int countp(int n, const float* x, float y) {
    const __m128 pred = _mm_set1_ps(y);
    int ret = 0;
    int i = 0;
    for (; i < (n & ~8); i += 8) {
        __m128 xi1 = _mm_loadu_ps(x + i);
        __m128 mask1 = _mm_cmpgt_ps(xi1, pred);
        __m128 xi2 = _mm_loadu_ps(x + i + 4);
        __m128 mask2 = _mm_cmpgt_ps(xi2, pred);
        ret += _mm_popcnt_u32(_mm_movemask_ps(mask1)) + _mm_popcnt_u32(_mm_movemask_ps(mask2));
    }
    for (; i < n; ++i) {
        __m128 xi = _mm_load_ss(x + i);
        __m128 mask = _mm_cmpgt_ss(xi, pred);
        int m = _mm_extract_ps(mask, 0);
        ret += !!m;
    }
    return ret;
}
```
