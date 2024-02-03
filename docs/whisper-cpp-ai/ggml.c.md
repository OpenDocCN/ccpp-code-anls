# `whisper.cpp\ggml.c`

```cpp
// 禁用 Windows 上荒谬的“不安全”警告
#define _CRT_SECURE_NO_DEPRECATE 
// 在 MSVC 上使用 M_PI
#define _USE_MATH_DEFINES 

// 引入内部实现头文件
#include "ggml-impl.h"
// 引入量化计算头文件
#include "ggml-quants.h"

// 根据不同编译器和操作系统引入不同的头文件
#if defined(_MSC_VER) || defined(__MINGW32__)
#include <malloc.h> // 在 MSC/MINGW 上使用 malloc.h
#elif !defined(__FreeBSD__) && !defined(__NetBSD__) && !defined(__OpenBSD__)
#include <alloca.h>
#endif

#include <assert.h>
#include <errno.h>
#include <time.h>
#include <math.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include <inttypes.h>
#include <stdio.h>
#include <float.h>
#include <limits.h>
#include <stdarg.h>
#include <signal.h>

#ifdef GGML_USE_METAL
#include <unistd.h>
#endif

// 根据不同编译器设置不同的警告和禁用
#if defined(_MSC_VER)
// 禁用“可能丢失数据”警告，避免大量强制类型转换
// 我们应该小心处理 :)
#pragma warning(disable: 4244 4267)

// 禁用 POSIX 废弃警告
// 这些函数永远不会消失
#pragma warning(disable: 4996)
#endif

// 根据不同操作系统设置不同的头文件和类型定义
#if defined(_WIN32)

#include <windows.h>

typedef volatile LONG atomic_int;
typedef atomic_int atomic_bool;

// 原子操作：存储
static void atomic_store(atomic_int * ptr, LONG val) {
    InterlockedExchange(ptr, val);
}
// 原子操作：加载
static LONG atomic_load(atomic_int * ptr) {
    return InterlockedCompareExchange(ptr, 0, 0);
}
// 原子操作：加法
static LONG atomic_fetch_add(atomic_int * ptr, LONG inc) {
    return InterlockedExchangeAdd(ptr, inc);
}
// 原子操作：减法
static LONG atomic_fetch_sub(atomic_int * ptr, LONG dec) {
    return atomic_fetch_add(ptr, -(dec));
}

typedef HANDLE pthread_t;

typedef DWORD thread_ret_t;
// 创建线程
static int pthread_create(pthread_t * out, void * unused, thread_ret_t(*func)(void *), void * arg) {
    (void) unused;
    HANDLE handle = CreateThread(NULL, 0, (LPTHREAD_START_ROUTINE) func, arg, 0, NULL);
    if (handle == NULL)
    {
        return EAGAIN;
    }

    *out = handle;
    return 0;
}

// 等待线程结束
static int pthread_join(pthread_t thread, void * unused) {
    (void) unused;
    int ret = (int) WaitForSingleObject(thread, INFINITE);
    CloseHandle(thread);
    # 返回变量 ret 的值
    return ret;
// 定义静态函数 sched_yield，用于让出 CPU 时间片
static int sched_yield (void) {
    // 调用 Windows API Sleep 函数，参数为 0 表示让出 CPU 时间片
    Sleep (0);
    // 返回 0 表示成功
    return 0;
}
#else
// 如果不是 Windows 平台，则包含以下头文件
#include <pthread.h>
#include <stdatomic.h>

// 定义线程返回类型为 void 指针
typedef void * thread_ret_t;

#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>

#endif

#ifdef GGML_USE_CPU_HBM
// 如果定义了 GGML_USE_CPU_HBM，则包含 hbwmalloc 头文件
#include <hbwmalloc.h>
#endif

#if defined(__APPLE__)
// 如果是苹果系统，则包含 TargetConditionals 头文件
#include <TargetConditionals.h>
#endif

// 如果是类 Unix 系统，并且不是 TV 或 Watch 系统
#if (defined(__linux__) || defined(__APPLE__) || defined(__FreeBSD__) || defined(__NetBSD__) || defined(__OpenBSD__)) && \
    (!defined(TARGET_OS_TV) && !defined(TARGET_OS_WATCH))

#include <sys/wait.h>

// 定义函数 ggml_print_backtrace，用于打印函数调用栈
void ggml_print_backtrace(void) {
    // 使用 gdb 打印函数调用栈
    char attach[32];
    snprintf(attach, sizeof(attach), "attach %d", getpid());
    int pid = fork();
    if (pid == 0) {
        execlp("gdb", "gdb", "--batch",
            "-ex", "set style enabled on",
            "-ex", attach,
            "-ex", "bt -frame-info source-and-location",
            "-ex", "detach",
            "-ex", "quit",
            (char *) NULL);
    } else {
        waitpid(pid, NULL, 0);
    }
}
#else
// 如果不是支持的平台，则定义函数 ggml_print_backtrace 为空函数
void ggml_print_backtrace(void) {
    // platform not supported
}
#endif

/*#define GGML_PERF*/
// 定义 GGML_DEBUG 为 0
#define GGML_DEBUG 0
// 定义 GGML_GELU_FP16
#define GGML_GELU_FP16
// 定义 GGML_GELU_QUICK_FP16
#define GGML_GELU_QUICK_FP16
// 定义 GGML_SILU_FP16
#define GGML_SILU_FP16
// 定义 GGML_SOFT_MAX_UNROLL 为 4
#define GGML_SOFT_MAX_UNROLL 4
// 定义 GGML_VEC_DOT_UNROLL 为 2
#define GGML_VEC_DOT_UNROLL  2
// 定义 GGML_VEC_MAD_UNROLL 为 32
#define GGML_VEC_MAD_UNROLL  32

//
// logging
//

// 如果 GGML_DEBUG 大于等于 1，则定义 GGML_PRINT_DEBUG 为 printf
#if (GGML_DEBUG >= 1)
#define GGML_PRINT_DEBUG(...) printf(__VA_ARGS__)
#else
#define GGML_PRINT_DEBUG(...)
#endif

// 如果 GGML_DEBUG 大于等于 5，则定义 GGML_PRINT_DEBUG_5 为 printf
#if (GGML_DEBUG >= 5)
#define GGML_PRINT_DEBUG_5(...) printf(__VA_ARGS__)
#else
#define GGML_PRINT_DEBUG_5(...)
#endif

// 如果 GGML_DEBUG 大于等于 10，则定义 GGML_PRINT_DEBUG_10 为 printf
#if (GGML_DEBUG >= 10)
#define GGML_PRINT_DEBUG_10(...) printf(__VA_ARGS__)
#else
// 定义宏 GGML_PRINT_DEBUG_10，用于打印调试信息，但未指定具体内容
#endif

// 定义宏 GGML_PRINT，用于打印信息，参数为可变参数列表
#define GGML_PRINT(...) printf(__VA_ARGS__)

//
// 日志块结束
//

#ifdef GGML_USE_ACCELERATE
// 如果定义了 GGML_USE_ACCELERATE，则使用 vDSP 进行软最大值计算
// 注意：不确定是否实际更快
//#define GGML_SOFT_MAX_ACCELERATE
#endif

#if defined(_MSC_VER) || defined(__MINGW32__)
// 如果编译器为 MSC 或者 MinGW32，则定义 GGML_ALIGNED_MALLOC 和 GGML_ALIGNED_FREE 为对应的函数
#define GGML_ALIGNED_MALLOC(size) _aligned_malloc(size, GGML_MEM_ALIGN)
#define GGML_ALIGNED_FREE(ptr)    _aligned_free(ptr)
#else
// 如果不是 MSC 或者 MinGW32 编译器，则定义 ggml_aligned_malloc 函数
inline static void * ggml_aligned_malloc(size_t size) {
    // 如果分配大小为 0，则打印警告信息并返回 NULL
    if (size == 0) {
        GGML_PRINT("WARNING: Behavior may be unexpected when allocating 0 bytes for ggml_aligned_malloc!\n");
        return NULL;
    }
    void * aligned_memory = NULL;
#ifdef GGML_USE_CPU_HBM
    // 如果定义了 GGML_USE_CPU_HBM，则使用 hbw_posix_memalign 进行内存对齐分配
    int result = hbw_posix_memalign(&aligned_memory, 16, size);
#elif GGML_USE_METAL
    // 如果定义了 GGML_USE_METAL，则使用 posix_memalign 进行内存对齐分配
    int result = posix_memalign(&aligned_memory, sysconf(_SC_PAGESIZE), size);
#else
    // 否则使用 posix_memalign 进行内存对齐分配
    int result = posix_memalign(&aligned_memory, GGML_MEM_ALIGN, size);
#endif
    // 如果分配失败，则打印错误信息并返回 NULL
    if (result != 0) {
        // 处理分配失败情况
        const char *error_desc = "unknown allocation error";
        switch (result) {
            case EINVAL:
                error_desc = "invalid alignment value";
                break;
            case ENOMEM:
                error_desc = "insufficient memory";
                break;
        }
        GGML_PRINT("%s: %s (attempted to allocate %6.2f MB)\n", __func__, error_desc, size/(1024.0*1024.0));
        GGML_ASSERT(false);
        return NULL;
    }
    return aligned_memory;
}
// 定义 GGML_ALIGNED_MALLOC 为 ggml_aligned_malloc 函数
#define GGML_ALIGNED_MALLOC(size) ggml_aligned_malloc(size)
#ifdef GGML_USE_CPU_HBM
// 如果定义了 GGML_USE_CPU_HBM，则定义 GGML_ALIGNED_FREE 为 hbw_free
#define GGML_ALIGNED_FREE(ptr)    if(NULL != ptr) hbw_free(ptr)
#else
// 否则定义 GGML_ALIGNED_FREE 为 free
#define GGML_ALIGNED_FREE(ptr)    free(ptr)
#endif
#endif

// 定义 ggml_malloc 函数，用于分配内存
inline static void * ggml_malloc(size_t size) {
    // 如果分配大小为 0，则打印警告信息并返回 NULL
    if (size == 0) {
        GGML_PRINT("WARNING: Behavior may be unexpected when allocating 0 bytes for ggml_malloc!\n");
        return NULL;
    }
    // 调用系统的 malloc 函数进行内存分配
    void * result = malloc(size);
    # 如果结果为空指针
    if (result == NULL) {
        # 打印错误信息，显示函数名和分配的内存大小（以MB为单位）
        GGML_PRINT("%s: failed to allocate %6.2f MB\n", __func__, size/(1024.0*1024.0));
        # 断言，终止程序执行
        GGML_ASSERT(false);
    }
    # 返回结果指针
    return result;
// 定义 ggml_calloc 函数，用于分配内存并初始化为零
inline static void * ggml_calloc(size_t num, size_t size) {
    // 如果 num 或 size 为 0，则打印警告信息并返回 NULL
    if (num == 0 || size == 0) {
        GGML_PRINT("WARNING: Behavior may be unexpected when allocating 0 bytes for ggml_calloc!\n");
        return NULL;
    }
    // 调用 calloc 函数分配内存
    void * result = calloc(num, size);
    // 如果分配失败，则打印错误信息并终止程序
    if (result == NULL) {
        GGML_PRINT("%s: failed to allocate %6.2f MB\n", __func__, size/(1024.0*1024.0));
        GGML_ASSERT(false);
    }
    // 返回分配的内存指针
    return result;
}

// 定义 GGML_MALLOC 宏，用于调用 ggml_malloc 函数
#define GGML_MALLOC(size)      ggml_malloc(size)

// 定义 GGML_CALLOC 宏，用于调用 ggml_calloc 函数
#define GGML_CALLOC(num, size) ggml_calloc(num, size)

// 定义 GGML_FREE 宏，用于释放内存
#define GGML_FREE(ptr) free(ptr)

// 定义 UNUSED 宏，用于标记未使用的变量
#define UNUSED GGML_UNUSED

// 定义 SWAP 宏，用于交换两个变量的值
#define SWAP(x, y, T) do { T SWAP = x; x = y; y = SWAP; } while (0)

// 根据不同的宏定义，包含不同的头文件和库
#if defined(GGML_USE_ACCELERATE)
#include <Accelerate/Accelerate.h>
#if defined(GGML_USE_CLBLAST) // 允许在 Accelerate 函数中使用 CLBlast
#include "ggml-opencl.h"
#endif
#elif defined(GGML_USE_OPENBLAS)
#if defined(GGML_BLAS_USE_MKL)
#include <mkl.h>
#else
#include <cblas.h>
#endif
#elif defined(GGML_USE_CUBLAS)
#include "ggml-cuda.h"
#elif defined(GGML_USE_CLBLAST)
#include "ggml-opencl.h"
#elif defined(GGML_USE_VULKAN)
#include "ggml-vulkan.h"
#elif defined(GGML_USE_SYCL)
#include "ggml-sycl.h"
#endif

// 定义浮点类型 ggml_float 用于累加求和
typedef double ggml_float;

// 重新定义 MIN 和 MAX 宏，用于返回两个值中的最小值和最大值
#undef MIN
#undef MAX
#define MIN(a, b) ((a) < (b) ? (a) : (b))
#define MAX(a, b) ((a) > (b) ? (a) : (b)

// 全局数据

// 预先计算的 gelu 表格，用于 f16 类型 (128 KB)
static ggml_fp16_t ggml_table_gelu_f16[1 << 16];

// 预先计算的快速 gelu 表格，用于 f16 类型 (128 KB)
static ggml_fp16_t ggml_table_gelu_quick_f16[1 << 16];

// 预先计算的 silu 表格，用于 f16 类型 (128 KB)
static ggml_fp16_t ggml_table_silu_f16[1 << 16];

// 预先计算的 exp 表格，用于 f16 类型 (128 KB)
static ggml_fp16_t ggml_table_exp_f16[1 << 16];

// 预先计算的 f32 表格，用于 f16 类型 (256 KB) (ggml-impl.h)
float ggml_table_f32_f16[1 << 16];

// 注意：不要在 ggml.c 文件中使用这些变量
// 这些变量应通过 ggml.h API 使用
// 将 ggml_fp16_t 类型的数据转换为 float 类型数据
float ggml_fp16_to_fp32(ggml_fp16_t x) {
    return (float) GGML_FP16_TO_FP32(x);
}

// 将 float 类型数据转换为 ggml_fp16_t 类型的数据
ggml_fp16_t ggml_fp32_to_fp16(float x) {
    return GGML_FP32_TO_FP16(x);
}

// 将 ggml_fp16_t 类型数组 x 转换为 float 类型数组 y，数组长度为 n
void ggml_fp16_to_fp32_row(const ggml_fp16_t * x, float * y, int n) {
    for (int i = 0; i < n; i++) {
        y[i] = GGML_FP16_TO_FP32(x[i]);
    }
}

// 将 float 类型数组 x 转换为 ggml_fp16_t 类型数组 y，数组长度为 n
void ggml_fp32_to_fp16_row(const float * x, ggml_fp16_t * y, int n) {
    int i = 0;
#if defined(__F16C__)
    for (; i + 7 < n; i += 8) {
        __m256 x_vec = _mm256_loadu_ps(x + i);
        __m128i y_vec = _mm256_cvtps_ph(x_vec, _MM_FROUND_TO_NEAREST_INT);
        _mm_storeu_si128((__m128i *)(y + i), y_vec);
    }
    for(; i + 3 < n; i += 4) {
        __m128 x_vec = _mm_loadu_ps(x + i);
        __m128i y_vec = _mm_cvtps_ph(x_vec, _MM_FROUND_TO_NEAREST_INT);
        _mm_storel_epi64((__m128i *)(y + i), y_vec);
    }
#endif
    for (; i < n; i++) {
        y[i] = GGML_FP32_TO_FP16(x[i]);
    }
}

//
// timing
//

#if defined(_MSC_VER) || defined(__MINGW32__)
static int64_t timer_freq, timer_start;
// 初始化计时器
void ggml_time_init(void) {
    LARGE_INTEGER t;
    QueryPerformanceFrequency(&t);
    timer_freq = t.QuadPart;

    // 下面的乘法运算可能会导致溢出，为了减少这种可能性，减去程序启动时间
    QueryPerformanceCounter(&t);
    timer_start = t.QuadPart;
}
// 返回毫秒级时间
int64_t ggml_time_ms(void) {
    LARGE_INTEGER t;
    QueryPerformanceCounter(&t);
    return ((t.QuadPart-timer_start) * 1000) / timer_freq;
}
// 返回微秒级时间
int64_t ggml_time_us(void) {
    LARGE_INTEGER t;
    QueryPerformanceCounter(&t);
    return ((t.QuadPart-timer_start) * 1000000) / timer_freq;
}
#else
// 初始化计时器
void ggml_time_init(void) {}
// 返回毫秒级时间
int64_t ggml_time_ms(void) {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return (int64_t)ts.tv_sec*1000 + (int64_t)ts.tv_nsec/1000000;
}

// 返回微秒级时间
int64_t ggml_time_us(void) {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    # 将秒数转换为微秒，并加上纳秒数转换为微秒后的值，返回总的微秒数
    return (int64_t)ts.tv_sec*1000000 + (int64_t)ts.tv_nsec/1000;
// 结束条件判断，如果定义了 #endif，则结束当前代码块
#endif

// 返回当前时钟周期数
int64_t ggml_cycles(void) {
    return clock();
}

// 返回每毫秒的时钟周期数
int64_t ggml_cycles_per_ms(void) {
    return CLOCKS_PER_SEC/1000;
}

// 根据是否定义了 GGML_PERF 宏，选择性定义不同的性能测试函数
#ifdef GGML_PERF
#define ggml_perf_time_ms()       ggml_time_ms()
#define ggml_perf_time_us()       ggml_time_us()
#define ggml_perf_cycles()        ggml_cycles()
#define ggml_perf_cycles_per_ms() ggml_cycles_per_ms()
#else
#define ggml_perf_time_ms()       0
#define ggml_perf_time_us()       0
#define ggml_perf_cycles()        0
#define ggml_perf_cycles_per_ms() 0
#endif

//
// cache line
//

// 根据硬件支持的缓存行大小定义 CACHE_LINE_SIZE
#if defined(__cpp_lib_hardware_interference_size)
#define CACHE_LINE_SIZE hardware_destructive_interference_size
#else
#if defined(__POWER9_VECTOR__)
#define CACHE_LINE_SIZE 128
#else
#define CACHE_LINE_SIZE 64
#endif
#endif

// 计算 float 类型数据在缓存行中的个数
static const size_t CACHE_LINE_SIZE_F32 = CACHE_LINE_SIZE/sizeof(float);

// 声明两个函数，用于计算 float 和 ggml_fp16_t 类型数据的点积
static void ggml_vec_dot_f32(const int n, float * restrict s, const float * restrict x, const float * restrict y);
static void ggml_vec_dot_f16(const int n, float * restrict s, ggml_fp16_t * restrict x, ggml_fp16_t * restrict y);

// 定义不同数据类型的特性，包括类型名称、块大小、类型大小和是否量化
static const ggml_type_traits_t type_traits[GGML_TYPE_COUNT] = {
    [GGML_TYPE_I8] = {
        .type_name                = "i8",
        .blck_size                = 1,
        .type_size                = sizeof(int8_t),
        .is_quantized             = false,
    },
    [GGML_TYPE_I16] = {
        .type_name                = "i16",
        .blck_size                = 1,
        .type_size                = sizeof(int16_t),
        .is_quantized             = false,
    },
    [GGML_TYPE_I32] = {
        .type_name                = "i32",
        .blck_size                = 1,
        .type_size                = sizeof(int32_t),
        .is_quantized             = false,
    },
    [GGML_TYPE_F32] = {
        # 设置类型名称为"f32"
        .type_name                = "f32",
        # 设置块大小为1
        .blck_size                = 1,
        # 设置类型大小为float的大小
        .type_size                = sizeof(float),
        # 设置是否量化为false
        .is_quantized             = false,
        # 设置向量点乘函数为ggml_vec_dot_f32
        .vec_dot                  = (ggml_vec_dot_t) ggml_vec_dot_f32,
        # 设置向量点乘类型为GGML_TYPE_F32
        .vec_dot_type             = GGML_TYPE_F32,
    },
    [GGML_TYPE_F16] = {
        # 设置类型名称为"f16"
        .type_name                = "f16",
        # 设置块大小为1
        .blck_size                = 1,
        # 设置类型大小为ggml_fp16_t的大小
        .type_size                = sizeof(ggml_fp16_t),
        # 设置是否量化为false
        .is_quantized             = false,
        # 设置转换为浮点数函数为ggml_fp16_to_fp32_row
        .to_float                 = (ggml_to_float_t) ggml_fp16_to_fp32_row,
        # 设置从浮点数转换函数为ggml_fp32_to_fp16_row
        .from_float               = (ggml_from_float_t) ggml_fp32_to_fp16_row,
        # 设置从浮点数转换函数为ggml_fp32_to_fp16_row
        .from_float_reference     = (ggml_from_float_t) ggml_fp32_to_fp16_row,
        # 设置向量点乘函数为ggml_vec_dot_f16
        .vec_dot                  = (ggml_vec_dot_t) ggml_vec_dot_f16,
        # 设置向量点乘类型为GGML_TYPE_F16
        .vec_dot_type             = GGML_TYPE_F16,
    },
    [GGML_TYPE_Q4_0] = {
        # 设置类型名称为"q4_0"
        .type_name                = "q4_0",
        # 设置块大小为QK4_0
        .blck_size                = QK4_0,
        # 设置类型大小为block_q4_0的大小
        .type_size                = sizeof(block_q4_0),
        # 设置是否量化为true
        .is_quantized             = true,
        # 设置转换为浮点数函数为dequantize_row_q4_0
        .to_float                 = (ggml_to_float_t) dequantize_row_q4_0,
        # 设置从浮点数转换函数为quantize_row_q4_0
        .from_float               = quantize_row_q4_0,
        # 设置从浮点数转换函数为quantize_row_q4_0_reference
        .from_float_reference     = (ggml_from_float_t) quantize_row_q4_0_reference,
        # 设置向量点乘函数为ggml_vec_dot_q4_0_q8_0
        .vec_dot                  = ggml_vec_dot_q4_0_q8_0,
        # 设置向量点乘类型为GGML_TYPE_Q8_0
        .vec_dot_type             = GGML_TYPE_Q8_0,
    },
    [GGML_TYPE_Q4_1] = {
        # 设置类型名称为"q4_1"
        .type_name                = "q4_1",
        # 设置块大小为QK4_1
        .blck_size                = QK4_1,
        # 设置类型大小为block_q4_1的大小
        .type_size                = sizeof(block_q4_1),
        # 设置是否量化为true
        .is_quantized             = true,
        # 设置转换为浮点数函数为dequantize_row_q4_1
        .to_float                 = (ggml_to_float_t) dequantize_row_q4_1,
        # 设置从浮点数转换函数为quantize_row_q4_1
        .from_float               = quantize_row_q4_1,
        # 设置从浮点数转换函数为quantize_row_q4_1_reference
        .from_float_reference     = (ggml_from_float_t) quantize_row_q4_1_reference,
        # 设置向量点乘函数为ggml_vec_dot_q4_1_q8_1
        .vec_dot                  = ggml_vec_dot_q4_1_q8_1,
        # 设置向量点乘类型为GGML_TYPE_Q8_1
        .vec_dot_type             = GGML_TYPE_Q8_1,
    },
    [4] = { // GGML_TYPE_Q4_2
        // 类型名称为"DEPRECATED"
        .type_name                = "DEPRECATED",
        // 块大小为0
        .blck_size                = 0,
        // 类型大小为0
        .type_size                = 0,
        // 未量化
        .is_quantized             = false,
        // 转换为浮点数的函数为空
        .to_float                 = NULL,
        // 从浮点数转换的函数为空
        .from_float               = NULL,
        // 从浮点数转换的参考函数为空
        .from_float_reference     = NULL,
        // 向量点积函数为空
        .vec_dot                  = NULL,
        // 向量点积类型为GGML_TYPE_COUNT
        .vec_dot_type             = GGML_TYPE_COUNT,
    },
    [5] = { // GGML_TYPE_Q4_3
        // 类型名称为"DEPRECATED"
        .type_name                = "DEPRECATED",
        // 块大小为0
        .blck_size                = 0,
        // 类型大小为0
        .type_size                = 0,
        // 未量化
        .is_quantized             = false,
        // 转换为浮点数的函数为空
        .to_float                 = NULL,
        // 从浮点数转换的函数为空
        .from_float               = NULL,
        // 从浮点数转换的参考函数为空
        .from_float_reference     = NULL,
        // 向量点积函数为空
        .vec_dot                  = NULL,
        // 向量点积类型为GGML_TYPE_COUNT
        .vec_dot_type             = GGML_TYPE_COUNT,
    },
    [GGML_TYPE_Q5_0] = {
        // 类型名称为"q5_0"
        .type_name                = "q5_0",
        // 块大小为QK5_0
        .blck_size                = QK5_0,
        // 类型大小为block_q5_0的大小
        .type_size                = sizeof(block_q5_0),
        // 已量化
        .is_quantized             = true,
        // 转换为浮点数的函数为dequantize_row_q5_0
        .to_float                 = (ggml_to_float_t) dequantize_row_q5_0,
        // 从浮点数转换的函数为quantize_row_q5_0
        .from_float               = quantize_row_q5_0,
        // 从浮点数转换的参考函数为quantize_row_q5_0_reference
        .from_float_reference     = (ggml_from_float_t) quantize_row_q5_0_reference,
        // 向量点积函数为ggml_vec_dot_q5_0_q8_0
        .vec_dot                  = ggml_vec_dot_q5_0_q8_0,
        // 向量点积类型为GGML_TYPE_Q8_0
        .vec_dot_type             = GGML_TYPE_Q8_0,
    },
    [GGML_TYPE_Q5_1] = {
        // 类型名称为"q5_1"
        .type_name                = "q5_1",
        // 块大小为QK5_1
        .blck_size                = QK5_1,
        // 类型大小为block_q5_1的大小
        .type_size                = sizeof(block_q5_1),
        // 已量化
        .is_quantized             = true,
        // 转换为浮点数的函数为dequantize_row_q5_1
        .to_float                 = (ggml_to_float_t) dequantize_row_q5_1,
        // 从浮点数转换的函数为quantize_row_q5_1
        .from_float               = quantize_row_q5_1,
        // 从浮点数转换的参考函数为quantize_row_q5_1_reference
        .from_float_reference     = (ggml_from_float_t) quantize_row_q5_1_reference,
        // 向量点积函数为ggml_vec_dot_q5_1_q8_1
        .vec_dot                  = ggml_vec_dot_q5_1_q8_1,
        // 向量点积类型为GGML_TYPE_Q8_1
        .vec_dot_type             = GGML_TYPE_Q8_1,
    },
    [GGML_TYPE_Q8_0] = {
        # 定义类型名称为 "q8_0"
        .type_name                = "q8_0",
        # 定义块大小为 QK8_0
        .blck_size                = QK8_0,
        # 定义类型大小为 block_q8_0 结构体的大小
        .type_size                = sizeof(block_q8_0),
        # 表示该类型是经过量化的
        .is_quantized             = true,
        # 将浮点数转换为 q8_0 类型的函数
        .to_float                 = (ggml_to_float_t) dequantize_row_q8_0,
        # 将 q8_0 类型转换为浮点数的函数
        .from_float               = quantize_row_q8_0,
        # 参考用的将浮点数转换为 q8_0 类型的函数
        .from_float_reference     = (ggml_from_float_t) quantize_row_q8_0_reference,
        # 计算 q8_0 类型向量点积的函数
        .vec_dot                  = ggml_vec_dot_q8_0_q8_0,
        # 向量点积的类型为 GGML_TYPE_Q8_0
        .vec_dot_type             = GGML_TYPE_Q8_0,
    },
    [GGML_TYPE_Q8_1] = {
        # 定义类型名称为 "q8_1"
        .type_name                = "q8_1",
        # 定义块大小为 QK8_1
        .blck_size                = QK8_1,
        # 定义类型大小为 block_q8_1 结构体的大小
        .type_size                = sizeof(block_q8_1),
        # 表示该类型是经过量化的
        .is_quantized             = true,
        # 将浮点数转换为 q8_1 类型的函数
        .from_float               = quantize_row_q8_1,
        # 参考用的将浮点数转换为 q8_1 类型的函数
        .from_float_reference     = (ggml_from_float_t) quantize_row_q8_1_reference,
        # 向量点积的类型为 GGML_TYPE_Q8_1
        .vec_dot_type             = GGML_TYPE_Q8_1,
    },
    [GGML_TYPE_Q2_K] = {
        # 定义类型名称为 "q2_K"
        .type_name                = "q2_K",
        # 定义块大小为 QK_K
        .blck_size                = QK_K,
        # 定义类型大小为 block_q2_K 结构体的大小
        .type_size                = sizeof(block_q2_K),
        # 表示该类型是经过量化的
        .is_quantized             = true,
        # 将浮点数转换为 q2_K 类型的函数
        .to_float                 = (ggml_to_float_t) dequantize_row_q2_K,
        # 将 q2_K 类型转换为浮点数的函数
        .from_float               = quantize_row_q2_K,
        # 参考用的将浮点数转换为 q2_K 类型的函数
        .from_float_reference     = (ggml_from_float_t) quantize_row_q2_K_reference,
        # 计算 q2_K 类型向量点积的函数
        .vec_dot                  = ggml_vec_dot_q2_K_q8_K,
        # 向量点积的类型为 GGML_TYPE_Q8_K
        .vec_dot_type             = GGML_TYPE_Q8_K,
    },
    [GGML_TYPE_Q3_K] = {
        # 定义类型名称为 "q3_K"
        .type_name                = "q3_K",
        # 定义块大小为 QK_K
        .blck_size                = QK_K,
        # 定义类型大小为 block_q3_K 的大小
        .type_size                = sizeof(block_q3_K),
        # 设置为量化状态为 true
        .is_quantized             = true,
        # 设置将浮点数转换为 q3_K 类型的函数
        .to_float                 = (ggml_to_float_t) dequantize_row_q3_K,
        # 设置将 q3_K 类型转换为浮点数的函数
        .from_float               = quantize_row_q3_K,
        # 设置将浮点数转换为 q3_K 类型的参考函数
        .from_float_reference     = (ggml_from_float_t) quantize_row_q3_K_reference,
        # 设置 q3_K 类型与 q8_K 类型的向量点乘函数
        .vec_dot                  = ggml_vec_dot_q3_K_q8_K,
        # 设置向量点乘的类型为 GGML_TYPE_Q8_K
        .vec_dot_type             = GGML_TYPE_Q8_K,
    },
    [GGML_TYPE_Q4_K] = {
        # 定义类型名称为 "q4_K"
        .type_name                = "q4_K",
        # 定义块大小为 QK_K
        .blck_size                = QK_K,
        # 定义类型大小为 block_q4_K 的大小
        .type_size                = sizeof(block_q4_K),
        # 设置为量化状态为 true
        .is_quantized             = true,
        # 设置将浮点数转换为 q4_K 类型的函数
        .to_float                 = (ggml_to_float_t) dequantize_row_q4_K,
        # 设置将 q4_K 类型转换为浮点数的函数
        .from_float               = quantize_row_q4_K,
        # 设置将浮点数转换为 q4_K 类型的参考函数
        .from_float_reference     = (ggml_from_float_t) quantize_row_q4_K_reference,
        # 设置 q4_K 类型与 q8_K 类型的向量点乘函数
        .vec_dot                  = ggml_vec_dot_q4_K_q8_K,
        # 设置向量点乘的类型为 GGML_TYPE_Q8_K
        .vec_dot_type             = GGML_TYPE_Q8_K,
    },
    [GGML_TYPE_Q5_K] = {
        # 定义类型名称为 "q5_K"
        .type_name                = "q5_K",
        # 定义块大小为 QK_K
        .blck_size                = QK_K,
        # 定义类型大小为 block_q5_K 的大小
        .type_size                = sizeof(block_q5_K),
        # 设置为量化状态为 true
        .is_quantized             = true,
        # 设置将浮点数转换为 q5_K 类型的函数
        .to_float                 = (ggml_to_float_t) dequantize_row_q5_K,
        # 设置将 q5_K 类型转换为浮点数的函数
        .from_float               = quantize_row_q5_K,
        # 设置将浮点数转换为 q5_K 类型的参考函数
        .from_float_reference     = (ggml_from_float_t) quantize_row_q5_K_reference,
        # 设置 q5_K 类型与 q8_K 类型的向量点乘函数
        .vec_dot                  = ggml_vec_dot_q5_K_q8_K,
        # 设置向量点乘的类型为 GGML_TYPE_Q8_K
        .vec_dot_type             = GGML_TYPE_Q8_K,
    },
    [GGML_TYPE_Q6_K] = {
        // 定义类型名称为 "q6_K"
        .type_name                = "q6_K",
        // 定义块大小为 QK_K
        .blck_size                = QK_K,
        // 定义类型大小为 block_q6_K 的大小
        .type_size                = sizeof(block_q6_K),
        // 设置为量化类型
        .is_quantized             = true,
        // 将数据转换为浮点数的函数为 dequantize_row_q6_K
        .to_float                 = (ggml_to_float_t) dequantize_row_q6_K,
        // 将数据从浮点数转换的函数为 quantize_row_q6_K
        .from_float               = quantize_row_q6_K,
        // 浮点数转换的参考函数为 quantize_row_q6_K_reference
        .from_float_reference     = (ggml_from_float_t) quantize_row_q6_K_reference,
        // 向量点积函数为 ggml_vec_dot_q6_K_q8_K
        .vec_dot                  = ggml_vec_dot_q6_K_q8_K,
        // 向量点积的类型为 GGML_TYPE_Q8_K
        .vec_dot_type             = GGML_TYPE_Q8_K,
    },
    [GGML_TYPE_IQ2_XXS] = {
        // 定义类型名称为 "iq2_xxs"
        .type_name                = "iq2_xxs",
        // 定义块大小为 QK_K
        .blck_size                = QK_K,
        // 定义类型大小为 block_iq2_xxs 的大小
        .type_size                = sizeof(block_iq2_xxs),
        // 设置为量化类型
        .is_quantized             = true,
        // 将数据转换为浮点数的函数为 dequantize_row_iq2_xxs
        .to_float                 = (ggml_to_float_t) dequantize_row_iq2_xxs,
        // 无浮点数转换函数
        .from_float               = NULL,
        // 无浮点数转换参考函数
        .from_float_reference     = NULL,
        // 向量点积函数为 ggml_vec_dot_iq2_xxs_q8_K
        .vec_dot                  = ggml_vec_dot_iq2_xxs_q8_K,
        // 向量点积的类型为 GGML_TYPE_Q8_K
        .vec_dot_type             = GGML_TYPE_Q8_K,
    },
    [GGML_TYPE_IQ2_XS] = {
        // 定义类型名称为 "iq2_xs"
        .type_name                = "iq2_xs",
        // 定义块大小为 QK_K
        .blck_size                = QK_K,
        // 定义类型大小为 block_iq2_xs 的大小
        .type_size                = sizeof(block_iq2_xs),
        // 设置为量化类型
        .is_quantized             = true,
        // 将数据转换为浮点数的函数为 dequantize_row_iq2_xs
        .to_float                 = (ggml_to_float_t) dequantize_row_iq2_xs,
        // 无浮点数转换函数
        .from_float               = NULL,
        // 无浮点数转换参考函数
        .from_float_reference     = NULL,
        // 向量点积函数为 ggml_vec_dot_iq2_xs_q8_K
        .vec_dot                  = ggml_vec_dot_iq2_xs_q8_K,
        // 向量点积的类型为 GGML_TYPE_Q8_K
        .vec_dot_type             = GGML_TYPE_Q8_K,
    },
    [GGML_TYPE_IQ3_XXS] = {
        # 定义类型为 iq3_xxs 的配置信息
        .type_name                = "iq3_xxs",
        # 块大小为 QK_K
        .blck_size                = QK_K,
        # 类型大小为 block_iq3_xxs 结构体的大小
        .type_size                = sizeof(block_iq3_xxs),
        # 是否量化为 true
        .is_quantized             = true,
        # 转换为浮点数的函数为 dequantize_row_iq3_xxs
        .to_float                 = (ggml_to_float_t) dequantize_row_iq3_xxs,
        # 从浮点数转换的函数为 quantize_row_iq3_xxs
        .from_float               = quantize_row_iq3_xxs,
        # 从浮点数转换的参考函数为 quantize_row_iq3_xxs_reference
        .from_float_reference     = (ggml_from_float_t)quantize_row_iq3_xxs_reference,
        # 向量点乘函数为 ggml_vec_dot_iq3_xxs_q8_K
        .vec_dot                  = ggml_vec_dot_iq3_xxs_q8_K,
        # 向量点乘的类型为 GGML_TYPE_Q8_K
        .vec_dot_type             = GGML_TYPE_Q8_K,
    },
    [GGML_TYPE_Q8_K] = {
        # 定义类型为 q8_K 的配置信息
        .type_name                = "q8_K",
        # 块大小为 QK_K
        .blck_size                = QK_K,
        # 类型大小为 block_q8_K 结构体的大小
        .type_size                = sizeof(block_q8_K),
        # 是否量化为 true
        .is_quantized             = true,
        # 从浮点数转换的函数为 quantize_row_q8_K
        .from_float               = quantize_row_q8_K,
    }
// 内部测试使用，根据给定的枚举类型返回对应的类型特征
ggml_type_traits_t ggml_internal_get_type_traits(enum ggml_type type) {
    // 断言枚举类型小于类型数量
    GGML_ASSERT(type < GGML_TYPE_COUNT);
    // 返回对应类型的类型特征
    return type_traits[type];
}

//
// SIMD 映射
//

#if defined(__ARM_NEON)
#if !defined(__aarch64__)

// 64位兼容性

// 定义一个函数，对一个 float32x4_t 类型的向量进行逐元素相加并返回结果
inline static float vaddvq_f32(float32x4_t v) {
    return vgetq_lane_f32(v, 0) + vgetq_lane_f32(v, 1) + vgetq_lane_f32(v, 2) + vgetq_lane_f32(v, 3);
}

#endif
#endif

// 我们定义一组通用的 C 宏，根据当前架构将其映射到特定的内联函数
// 然后我们在下面的基本计算操作中只使用这些宏
// 添加对新架构的支持需要定义相应的 SIMD 宏
//
// GGML_F32_STEP / GGML_F16_STEP
//   每次处理的元素数量
//
// GGML_F32_EPR / GGML_F16_EPR
//   每个寄存器中可以容纳的元素数量
//

#if defined(__ARM_NEON) && defined(__ARM_FEATURE_FMA)

#define GGML_SIMD

// F32 NEON

#define GGML_F32_STEP 16
#define GGML_F32_EPR  4

#define GGML_F32x4              float32x4_t
#define GGML_F32x4_ZERO         vdupq_n_f32(0.0f)
#define GGML_F32x4_SET1(x)      vdupq_n_f32(x)
#define GGML_F32x4_LOAD         vld1q_f32
#define GGML_F32x4_STORE        vst1q_f32
#define GGML_F32x4_FMA(a, b, c) vfmaq_f32(a, b, c)
#define GGML_F32x4_ADD          vaddq_f32
#define GGML_F32x4_MUL          vmulq_f32
#define GGML_F32x4_REDUCE_ONE(x) vaddvq_f32(x)
#define GGML_F32x4_REDUCE(res, x)              \
{                                              \
    int offset = GGML_F32_ARR >> 1;            \
    for (int i = 0; i < offset; ++i) {         \
        x[i] = vaddq_f32(x[i], x[offset+i]);   \
    }                                          \
    offset >>= 1;                              \
    for (int i = 0; i < offset; ++i) {         \
        x[i] = vaddq_f32(x[i], x[offset+i]);   \
    }                                          \
    offset >>= 1;                              \
    # 遍历从0到offset的整数序列
    for (int i = 0; i < offset; ++i) {         \
        # 将x[i]和x[offset+i]对应位置的元素相加，并将结果存储在x[i]中
        x[i] = vaddq_f32(x[i], x[offset+i]);   \
    }                                          \
    # 对x[0]进行一次特定的浮点数运算，将结果存储在res中
    res = GGML_F32x4_REDUCE_ONE(x[0]);         \
// 定义宏，将 GGML_F32_VEC 替换为 GGML_F32x4
#define GGML_F32_VEC        GGML_F32x4
// 定义宏，将 GGML_F32_VEC_ZERO 替换为 GGML_F32x4_ZERO
#define GGML_F32_VEC_ZERO   GGML_F32x4_ZERO
// 定义宏，将 GGML_F32_VEC_SET1 替换为 GGML_F32x4_SET1
#define GGML_F32_VEC_SET1   GGML_F32x4_SET1
// 定义宏，将 GGML_F32_VEC_LOAD 替换为 GGML_F32x4_LOAD
#define GGML_F32_VEC_LOAD   GGML_F32x4_LOAD
// 定义宏，将 GGML_F32_VEC_STORE 替换为 GGML_F32x4_STORE
#define GGML_F32_VEC_STORE  GGML_F32x4_STORE
// 定义宏，将 GGML_F32_VEC_FMA 替换为 GGML_F32x4_FMA
#define GGML_F32_VEC_FMA    GGML_F32x4_FMA
// 定义宏，将 GGML_F32_VEC_ADD 替换为 GGML_F32x4_ADD
#define GGML_F32_VEC_ADD    GGML_F32x4_ADD
// 定义宏，将 GGML_F32_VEC_MUL 替换为 GGML_F32x4_MUL
#define GGML_F32_VEC_MUL    GGML_F32x4_MUL
// 定义宏，将 GGML_F32_VEC_REDUCE 替换为 GGML_F32x4_REDUCE

// 如果支持 ARM FP16 向量运算
#if defined(__ARM_FEATURE_FP16_VECTOR_ARITHMETIC)
    // 定义 GGML_F16_STEP 为 32
    #define GGML_F16_STEP 32
    // 定义 GGML_F16_EPR 为 8
    #define GGML_F16_EPR  8

    // 定义 GGML_F16x8 为 float16x8_t 类型
    #define GGML_F16x8              float16x8_t
    // 定义 GGML_F16x8_ZERO 为将 0.0f 复制为 float16x8_t 类型
    #define GGML_F16x8_ZERO         vdupq_n_f16(0.0f)
    // 定义 GGML_F16x8_SET1(x) 为将 x 复制为 float16x8_t 类型
    #define GGML_F16x8_SET1(x)      vdupq_n_f16(x)
    // 定义 GGML_F16x8_LOAD 为加载 float16x8_t 类型
    #define GGML_F16x8_LOAD         vld1q_f16
    // 定义 GGML_F16x8_STORE 为存储 float16x8_t 类型
    #define GGML_F16x8_STORE        vst1q_f16
    // 定义 GGML_F16x8_FMA(a, b, c) 为 float16x8_t 类型的 FMA 运算
    #define GGML_F16x8_FMA(a, b, c) vfmaq_f16(a, b, c)
    // 定义 GGML_F16x8_ADD 为 float16x8_t 类型的加法运算
    #define GGML_F16x8_ADD          vaddq_f16
    // 定义 GGML_F16x8_MUL 为 float16x8_t 类型的乘法运算
    #define GGML_F16x8_MUL          vmulq_f16
    // 定义 GGML_F16x8_REDUCE(res, x) 为空
    #define GGML_F16x8_REDUCE(res, x)                             \
    # 定义一个 do-while 循环，用于执行一系列操作
    do {                                                          \
        # 计算偏移量，将 GGML_F16_ARR 右移一位
        int offset = GGML_F16_ARR >> 1;                           \
        # 对数组 x 进行操作，将前半部分与后半部分相加
        for (int i = 0; i < offset; ++i) {                        \
            x[i] = vaddq_f16(x[i], x[offset+i]);                  \
        }                                                         \
        # 更新偏移量
        offset >>= 1;                                             \
        # 对数组 x 进行操作，将前半部分与后半部分相加
        for (int i = 0; i < offset; ++i) {                        \
            x[i] = vaddq_f16(x[i], x[offset+i]);                  \
        }                                                         \
        # 更新偏移量
        offset >>= 1;                                             \
        # 对数组 x 进行操作，将前半部分与后半部分相加
        for (int i = 0; i < offset; ++i) {                        \
            x[i] = vaddq_f16(x[i], x[offset+i]);                  \
        }                                                         \
        # 将 x[0] 的低位和高位转换为 float32 类型
        const float32x4_t t0 = vcvt_f32_f16(vget_low_f16 (x[0])); \
        const float32x4_t t1 = vcvt_f32_f16(vget_high_f16(x[0])); \
        # 将 t0 和 t1 相加，并转换为 ggml_float 类型
        res = (ggml_float) vaddvq_f32(vaddq_f32(t0, t1));         \
    } while (0)

    # 定义一系列宏，用于操作 GGML_F16x8 类型的向量
    #define GGML_F16_VEC                GGML_F16x8
    #define GGML_F16_VEC_ZERO           GGML_F16x8_ZERO
    #define GGML_F16_VEC_SET1           GGML_F16x8_SET1
    #define GGML_F16_VEC_LOAD(p, i)     GGML_F16x8_LOAD(p)
    #define GGML_F16_VEC_STORE(p, r, i) GGML_F16x8_STORE(p, r[i])
    #define GGML_F16_VEC_FMA            GGML_F16x8_FMA
    #define GGML_F16_VEC_ADD            GGML_F16x8_ADD
    #define GGML_F16_VEC_MUL            GGML_F16x8_MUL
    #define GGML_F16_VEC_REDUCE         GGML_F16x8_REDUCE
// 如果不支持 FP16 向量运算，则使用 FP32，并利用 vcvt_ 函数来进行 FP16 与 FP32 之间的转换

#define GGML_F16_STEP 16
#define GGML_F16_EPR  4

#define GGML_F32Cx4              float32x4_t
#define GGML_F32Cx4_ZERO         vdupq_n_f32(0.0f)
#define GGML_F32Cx4_SET1(x)      vdupq_n_f32(x)
#define GGML_F32Cx4_LOAD(x)      vcvt_f32_f16(vld1_f16(x))
#define GGML_F32Cx4_STORE(x, y)  vst1_f16(x, vcvt_f16_f32(y))
#define GGML_F32Cx4_FMA(a, b, c) vfmaq_f32(a, b, c)
#define GGML_F32Cx4_ADD          vaddq_f32
#define GGML_F32Cx4_MUL          vmulq_f32
#define GGML_F32Cx4_REDUCE       GGML_F32x4_REDUCE

#define GGML_F16_VEC                GGML_F32Cx4
#define GGML_F16_VEC_ZERO           GGML_F32Cx4_ZERO
#define GGML_F16_VEC_SET1           GGML_F32Cx4_SET1
#define GGML_F16_VEC_LOAD(p, i)     GGML_F32Cx4_LOAD(p)
#define GGML_F16_VEC_STORE(p, r, i) GGML_F32Cx4_STORE(p, r[i])
#define GGML_F16_VEC_FMA            GGML_F32Cx4_FMA
#define GGML_F16_VEC_ADD            GGML_F32Cx4_ADD
#define GGML_F16_VEC_MUL            GGML_F32Cx4_MUL
#define GGML_F16_VEC_REDUCE         GGML_F32Cx4_REDUCE
#endif

#elif defined(__AVX__)

#define GGML_SIMD

// F32 AVX

#define GGML_F32_STEP 32
#define GGML_F32_EPR  8

#define GGML_F32x8         __m256
#define GGML_F32x8_ZERO    _mm256_setzero_ps()
#define GGML_F32x8_SET1(x) _mm256_set1_ps(x)
#define GGML_F32x8_LOAD    _mm256_loadu_ps
#define GGML_F32x8_STORE   _mm256_storeu_ps
#if defined(__FMA__)
    #define GGML_F32x8_FMA(a, b, c) _mm256_fmadd_ps(b, c, a)
#else
    #define GGML_F32x8_FMA(a, b, c) _mm256_add_ps(_mm256_mul_ps(b, c), a)
#endif
#define GGML_F32x8_ADD     _mm256_add_ps
#define GGML_F32x8_MUL     _mm256_mul_ps
#define GGML_F32x8_REDUCE(res, x)                                 \
do {                                                              \
    # 计算偏移量，将 GGML_F32_ARR 右移一位
    int offset = GGML_F32_ARR >> 1;                               
    # 对前一半数据进行累加操作
    for (int i = 0; i < offset; ++i) {                            
        x[i] = _mm256_add_ps(x[i], x[offset+i]);                  
    }                                                             
    # 更新偏移量为原偏移量的一半
    offset >>= 1;                                                 
    # 对前一半数据进行累加操作
    for (int i = 0; i < offset; ++i) {                            
        x[i] = _mm256_add_ps(x[i], x[offset+i]);                  
    }                                                             
    # 更新偏移量为原偏移量的一半
    offset >>= 1;                                                 
    # 对前一半数据进行累加操作
    for (int i = 0; i < offset; ++i) {                            
        x[i] = _mm256_add_ps(x[i], x[offset+i]);                  
    }                                                             
    # 计算 t0，将 x[0] 的低128位和高128位相加
    const __m128 t0 = _mm_add_ps(_mm256_castps256_ps128(x[0]),    
                                 _mm256_extractf128_ps(x[0], 1)); 
    # 对 t0 进行水平加法操作
    const __m128 t1 = _mm_hadd_ps(t0, t0);                        
    # 对 t1 进行水平加法操作，并将结果转换为 float 类型
    res = _mm_cvtss_f32(_mm_hadd_ps(t1, t1));                     
// 使用 do-while 循环，目的是为了确保代码块至少执行一次
} while (0)
// TODO: 这样写是否最优？

// 定义宏，将 GGML_F32x8 重命名为 GGML_F32_VEC
#define GGML_F32_VEC        GGML_F32x8
// 定义宏，将 GGML_F32x8_ZERO 重命名为 GGML_F32_VEC_ZERO
#define GGML_F32_VEC_ZERO   GGML_F32x8_ZERO
// 定义宏，将 GGML_F32x8_SET1 重命名为 GGML_F32_VEC_SET1
#define GGML_F32_VEC_SET1   GGML_F32x8_SET1
// 定义宏，将 GGML_F32x8_LOAD 重命名为 GGML_F32_VEC_LOAD
#define GGML_F32_VEC_LOAD   GGML_F32x8_LOAD
// 定义宏，将 GGML_F32x8_STORE 重命名为 GGML_F32_VEC_STORE
#define GGML_F32_VEC_STORE  GGML_F32x8_STORE
// 定义宏，将 GGML_F32x8_FMA 重命名为 GGML_F32_VEC_FMA
#define GGML_F32_VEC_FMA    GGML_F32x8_FMA
// 定义宏，将 GGML_F32x8_ADD 重命名为 GGML_F32_VEC_ADD
#define GGML_F32_VEC_ADD    GGML_F32x8_ADD
// 定义宏，将 GGML_F32x8_MUL 重命名为 GGML_F32_VEC_MUL
#define GGML_F32_VEC_MUL    GGML_F32x8_MUL
// 定义宏，将 GGML_F32x8_REDUCE 重命名为 GGML_F32_VEC_REDUCE
#define GGML_F32_VEC_REDUCE GGML_F32x8_REDUCE

// 定义宏，将 GGML_F16_STEP 设置为 32
#define GGML_F16_STEP 32
// 定义宏，将 GGML_F16_EPR 设置为 8
#define GGML_F16_EPR  8

// F16 算术不受 AVX 支持，因此我们使用 F32 代替

// 定义宏，将 GGML_F32Cx8 重命名为 __m256
#define GGML_F32Cx8             __m256
// 定义宏，将 GGML_F32Cx8_ZERO 重命名为 _mm256_setzero_ps()
#define GGML_F32Cx8_ZERO        _mm256_setzero_ps()
// 定义宏，将 GGML_F32Cx8_SET1(x) 重命名为 _mm256_set1_ps(x)
#define GGML_F32Cx8_SET1(x)     _mm256_set1_ps(x)

#if defined(__F16C__)
// 使用 _mm256_cvt 的内联函数需要 F16C
// 定义宏，将 GGML_F32Cx8_LOAD(x) 重命名为 _mm256_cvtph_ps(_mm_loadu_si128((__m128i *)(x)))
#define GGML_F32Cx8_LOAD(x)     _mm256_cvtph_ps(_mm_loadu_si128((__m128i *)(x))
// 定义宏，将 GGML_F32Cx8_STORE(x, y) 重命名为 _mm_storeu_si128((__m128i *)(x), _mm256_cvtps_ph(y, 0))
#define GGML_F32Cx8_STORE(x, y) _mm_storeu_si128((__m128i *)(x), _mm256_cvtps_ph(y, 0))
#else
// 如果不支持 F16C，则定义内联函数 __avx_f32cx8_load 和 __avx_f32cx8_store
static inline __m256 __avx_f32cx8_load(ggml_fp16_t *x) {
    float tmp[8];

    for (int i = 0; i < 8; i++) {
        tmp[i] = GGML_FP16_TO_FP32(x[i]);
    }

    return _mm256_loadu_ps(tmp);
}
static inline void __avx_f32cx8_store(ggml_fp16_t *x, __m256 y) {
    float arr[8];

    _mm256_storeu_ps(arr, y);

    for (int i = 0; i < 8; i++)
        x[i] = GGML_FP32_TO_FP16(arr[i]);
}
// 定义宏，将 GGML_F32Cx8_LOAD(x) 重命名为 __avx_f32cx8_load(x)
#define GGML_F32Cx8_LOAD(x)     __avx_f32cx8_load(x)
// 定义宏，将 GGML_F32Cx8_STORE(x, y) 重命名为 __avx_f32cx8_store(x, y)
#define GGML_F32Cx8_STORE(x, y) __avx_f32cx8_store(x, y)
#endif

// 定义宏，将 GGML_F32Cx8_FMA 重命名为 GGML_F32x8_FMA
#define GGML_F32Cx8_FMA         GGML_F32x8_FMA
// 定义宏，将 GGML_F32Cx8_ADD 重命名为 _mm256_add_ps
#define GGML_F32Cx8_ADD         _mm256_add_ps
// 定义宏，将 GGML_F32Cx8_MUL 重命名为 _mm256_mul_ps
#define GGML_F32Cx8_MUL         _mm256_mul_ps
// 定义宏，将 GGML_F32Cx8_REDUCE 重命名为 GGML_F32x8_REDUCE
#define GGML_F32Cx8_REDUCE      GGML_F32x8_REDUCE

// 定义宏，将 GGML_F16_VEC 重命名为 GGML_F32Cx8
#define GGML_F16_VEC                GGML_F32Cx8
// 定义宏，将 GGML_F16_VEC_ZERO 重命名为 GGML_F32Cx8_ZERO
#define GGML_F16_VEC_ZERO           GGML_F32Cx8_ZERO
// 定义宏，将 GGML_F16_VEC_SET1 重命名为 GGML_F32Cx8_SET1
#define GGML_F16_VEC_SET1           GGML_F32Cx8_SET1
// 定义宏，将 GGML_F16_VEC_LOAD(p, i) 重命名为 GGML_F32Cx8_LOAD(p)
#define GGML_F16_VEC_LOAD(p, i)     GGML_F32Cx8_LOAD(p)
// 定义宏，将 GGML_F16_VEC_STORE(p, r, i) 重命名为 GGML_F32Cx8_STORE(p, r[i])
#define GGML_F16_VEC_STORE(p, r, i) GGML_F32Cx8_STORE(p, r[i])
// 定义宏，将 GGML_F16_VEC_FMA 重命名为 GGML_F32Cx8_FMA
#define GGML_F16_VEC_FMA            GGML_F32Cx8_FMA
// 定义 GGML_F16_VEC_ADD 为 GGML_F32Cx8_ADD
#define GGML_F16_VEC_ADD            GGML_F32Cx8_ADD
// 定义 GGML_F16_VEC_MUL 为 GGML_F32Cx8_MUL
#define GGML_F16_VEC_MUL            GGML_F32Cx8_MUL
// 定义 GGML_F16_VEC_REDUCE 为 GGML_F32Cx8_REDUCE
#define GGML_F16_VEC_REDUCE         GGML_F32Cx8_REDUCE

#elif defined(__POWER9_VECTOR__)

// 定义 GGML_SIMD
#define GGML_SIMD

// F32 POWER9

// 定义 GGML_F32_STEP 为 32
#define GGML_F32_STEP 32
// 定义 GGML_F32_EPR 为 4
#define GGML_F32_EPR  4

// 定义 GGML_F32x4 为 vector float
#define GGML_F32x4              vector float
// 定义 GGML_F32x4_ZERO 为 0.0f
#define GGML_F32x4_ZERO         0.0f
// 定义 GGML_F32x4_SET1 为 vec_splats
#define GGML_F32x4_SET1         vec_splats
// 定义 GGML_F32x4_LOAD(p) 为 vec_xl(0, p)
#define GGML_F32x4_LOAD(p)      vec_xl(0, p)
// 定义 GGML_F32x4_STORE(p, r) 为 vec_xst(r, 0, p)
#define GGML_F32x4_STORE(p, r)  vec_xst(r, 0, p)
// 定义 GGML_F32x4_FMA(a, b, c) 为 vec_madd(b, c, a)
#define GGML_F32x4_FMA(a, b, c) vec_madd(b, c, a)
// 定义 GGML_F32x4_ADD 为 vec_add
#define GGML_F32x4_ADD          vec_add
// 定义 GGML_F32x4_MUL 为 vec_mul
#define GGML_F32x4_MUL          vec_mul
// 定义 GGML_F32x4_REDUCE(res, x) 为对 x 进行多次累加操作，结果存储在 res 中
#define GGML_F32x4_REDUCE(res, x)              \
{                                              \
    int offset = GGML_F32_ARR >> 1;            \
    for (int i = 0; i < offset; ++i) {         \
        x[i] = vec_add(x[i], x[offset+i]);     \
    }                                          \
    offset >>= 1;                              \
    for (int i = 0; i < offset; ++i) {         \
        x[i] = vec_add(x[i], x[offset+i]);     \
    }                                          \
    offset >>= 1;                              \
    for (int i = 0; i < offset; ++i) {         \
        x[i] = vec_add(x[i], x[offset+i]);     \
    }                                          \
    res = vec_extract(x[0], 0) +               \
          vec_extract(x[0], 1) +               \
          vec_extract(x[0], 2) +               \
          vec_extract(x[0], 3);                \
}

// 定义 GGML_F32_VEC 为 GGML_F32x4
#define GGML_F32_VEC        GGML_F32x4
// 定义 GGML_F32_VEC_ZERO 为 GGML_F32x4_ZERO
#define GGML_F32_VEC_ZERO   GGML_F32x4_ZERO
// 定义 GGML_F32_VEC_SET1 为 GGML_F32x4_SET1
#define GGML_F32_VEC_SET1   GGML_F32x4_SET1
// 定义 GGML_F32_VEC_LOAD 为 GGML_F32x4_LOAD
#define GGML_F32_VEC_LOAD   GGML_F32x4_LOAD
// 定义 GGML_F32_VEC_STORE 为 GGML_F32x4_STORE
#define GGML_F32_VEC_STORE  GGML_F32x4_STORE
// 定义 GGML_F32_VEC_FMA 为 GGML_F32x4_FMA
#define GGML_F32_VEC_FMA    GGML_F32x4_FMA
// 定义 GGML_F32_VEC_ADD 为 GGML_F32x4_ADD
#define GGML_F32_VEC_ADD    GGML_F32x4_ADD
// 定义 GGML_F32_VEC_MUL 为 GGML_F32x4_MUL
#define GGML_F32_VEC_MUL    GGML_F32x4_MUL
// 定义 GGML_F32_VEC_REDUCE 为 GGML_F32x4_REDUCE
#define GGML_F32_VEC_REDUCE GGML_F32x4_REDUCE

// F16 POWER9
// 定义 GGML_F16_STEP 为 GGML_F32_STEP
#define GGML_F16_STEP       GGML_F32_STEP
// 定义 GGML_F16_EPR 为 GGML_F32_EPR
#define GGML_F16_EPR        GGML_F32_EPR
// 定义 GGML_F16_VEC 为 GGML_F32x4
#define GGML_F16_VEC        GGML_F32x4
// 定义 GGML_F16_VEC_ZERO 为 GGML_F32x4_ZERO
#define GGML_F16_VEC_ZERO   GGML_F32x4_ZERO
// 定义 GGML_F16_VEC_SET1 为 GGML_F32x4_SET1
#define GGML_F16_VEC_SET1   GGML_F32x4_SET1
// 定义 GGML_F16_VEC_FMA 为 GGML_F32x4_FMA
#define GGML_F16_VEC_FMA    GGML_F32x4_FMA
// 定义 GGML_F16_VEC_REDUCE 为 GGML_F32x4_REDUCE
#define GGML_F16_VEC_REDUCE GGML_F32x4_REDUCE
// 在加载地址未对齐的情况下使用 vec_xl，而不是 vec_ld
// 定义 GGML_F16_VEC_LOAD 宏，根据 i 的值选择不同的加载方式
#define GGML_F16_VEC_LOAD(p, i) (i & 0x1) ?                   \
  vec_extract_fp32_from_shorth(vec_xl(0, p - GGML_F16_EPR)) : \
  vec_extract_fp32_from_shortl(vec_xl(0, p))
// 定义 GGML_ENDIAN_BYTE 宏，获取字节序
#define GGML_ENDIAN_BYTE(i) ((unsigned char *)&(uint16_t){1})[i]
// 定义 GGML_F16_VEC_STORE 宏，根据 i 的值选择不同的存储方式
#define GGML_F16_VEC_STORE(p, r, i)                             \
  if (i & 0x1)                                                  \
    vec_xst(vec_pack_to_short_fp32(r[i - GGML_ENDIAN_BYTE(1)],  \
                                   r[i - GGML_ENDIAN_BYTE(0)]), \
            0, p - GGML_F16_EPR)

#elif defined(__wasm_simd128__)

// 定义 GGML_SIMD
#define GGML_SIMD

// F32 WASM

// 定义 GGML_F32_STEP 为 16
#define GGML_F32_STEP 16
// 定义 GGML_F32_EPR 为 4
#define GGML_F32_EPR  4

// 定义 GGML_F32x4 为 v128_t
#define GGML_F32x4              v128_t
// 定义 GGML_F32x4_ZERO 为 wasm_f32x4_splat(0.0f)
#define GGML_F32x4_ZERO         wasm_f32x4_splat(0.0f)
// 定义 GGML_F32x4_SET1(x) 为 wasm_f32x4_splat(x)
#define GGML_F32x4_SET1(x)      wasm_f32x4_splat(x)
// 定义 GGML_F32x4_LOAD 为 wasm_v128_load
#define GGML_F32x4_LOAD         wasm_v128_load
// 定义 GGML_F32x4_STORE 为 wasm_v128_store
#define GGML_F32x4_STORE        wasm_v128_store
// 定义 GGML_F32x4_FMA(a, b, c) 为 wasm_f32x4_add(wasm_f32x4_mul(b, c), a)
#define GGML_F32x4_FMA(a, b, c) wasm_f32x4_add(wasm_f32x4_mul(b, c), a)
// 定义 GGML_F32x4_ADD 为 wasm_f32x4_add
#define GGML_F32x4_ADD          wasm_f32x4_add
// 定义 GGML_F32x4_MUL 为 wasm_f32x4_mul
#define GGML_F32x4_MUL          wasm_f32x4_mul
// 定义 GGML_F32x4_REDUCE(res, x) 宏，对 x 进行归约操作
#define GGML_F32x4_REDUCE(res, x)                  \
{                                                  \
    int offset = GGML_F32_ARR >> 1;                \
    for (int i = 0; i < offset; ++i) {             \
        x[i] = wasm_f32x4_add(x[i], x[offset+i]);  \
    }                                              \
    offset >>= 1;                                  \
    for (int i = 0; i < offset; ++i) {             \
        x[i] = wasm_f32x4_add(x[i], x[offset+i]);  \
    }                                              \
    offset >>= 1;                                  \
    # 遍历数组 x，从索引 0 到 offset-1
    for (int i = 0; i < offset; ++i) {             \
        # 将 x[i] 和 x[offset+i] 对应位置的元素相加，并将结果存入 x[i]
        x[i] = wasm_f32x4_add(x[i], x[offset+i]);  \
    }                                              \
    # 计算 x[0] 中每个元素的值并相加，得到最终结果
    res = wasm_f32x4_extract_lane(x[0], 0) +       \
          wasm_f32x4_extract_lane(x[0], 1) +       \
          wasm_f32x4_extract_lane(x[0], 2) +       \
          wasm_f32x4_extract_lane(x[0], 3);        \
// 定义 GGML_F32_VEC 为 GGML_F32x4
#define GGML_F32_VEC        GGML_F32x4
// 定义 GGML_F32_VEC_ZERO 为 GGML_F32x4_ZERO
#define GGML_F32_VEC_ZERO   GGML_F32x4_ZERO
// 定义 GGML_F32_VEC_SET1 为 GGML_F32x4_SET1
#define GGML_F32_VEC_SET1   GGML_F32x4_SET1
// 定义 GGML_F32_VEC_LOAD 为 GGML_F32x4_LOAD
#define GGML_F32_VEC_LOAD   GGML_F32x4_LOAD
// 定义 GGML_F32_VEC_STORE 为 GGML_F32x4_STORE
#define GGML_F32_VEC_STORE  GGML_F32x4_STORE
// 定义 GGML_F32_VEC_FMA 为 GGML_F32x4_FMA
#define GGML_F32_VEC_FMA    GGML_F32x4_FMA
// 定义 GGML_F32_VEC_ADD 为 GGML_F32x4_ADD
#define GGML_F32_VEC_ADD    GGML_F32x4_ADD
// 定义 GGML_F32_VEC_MUL 为 GGML_F32x4_MUL
#define GGML_F32_VEC_MUL    GGML_F32x4_MUL
// 定义 GGML_F32_VEC_REDUCE 为 GGML_F32x4_REDUCE
#define GGML_F32_VEC_REDUCE GGML_F32x4_REDUCE

// 定义 GGML_F16_STEP 为 16
#define GGML_F16_STEP 16
// 定义 GGML_F16_EPR 为 4
#define GGML_F16_EPR  4

// 定义内联函数 __wasm_f16x4_load，加载 ggml_fp16_t 类型指针数据并转换为 v128_t 类型
inline static v128_t __wasm_f16x4_load(const ggml_fp16_t * p) {
    // 创建临时数组 tmp 存储转换后的数据
    float tmp[4];

    // 将 ggml_fp16_t 类型数据转换为 float 类型存储在 tmp 数组中
    tmp[0] = GGML_FP16_TO_FP32(p[0]);
    tmp[1] = GGML_FP16_TO_FP32(p[1]);
    tmp[2] = GGML_FP16_TO_FP32(p[2]);
    tmp[3] = GGML_FP16_TO_FP32(p[3]);

    // 返回加载后的 v128_t 类型数据
    return wasm_v128_load(tmp);
}

// 定义内联函数 __wasm_f16x4_store，将 v128_t 类型数据存储到 ggml_fp16_t 类型指针中
inline static void __wasm_f16x4_store(ggml_fp16_t * p, v128_t x) {
    // 创建临时数组 tmp 存储转换后的数据
    float tmp[4];

    // 将 v128_t 类型数据存储到 tmp 数组中
    wasm_v128_store(tmp, x);

    // 将 tmp 数组中的数据转换为 ggml_fp16_t 类型存储到指针 p 中
    p[0] = GGML_FP32_TO_FP16(tmp[0]);
    p[1] = GGML_FP32_TO_FP16(tmp[1]);
    p[2] = GGML_FP32_TO_FP16(tmp[2]);
    p[3] = GGML_FP32_TO_FP16(tmp[3]);
}

// 定义 GGML_F16x4 为 v128_t
#define GGML_F16x4             v128_t
// 定义 GGML_F16x4_ZERO 为 wasm_f32x4_splat(0.0f)
#define GGML_F16x4_ZERO        wasm_f32x4_splat(0.0f)
// 定义 GGML_F16x4_SET1(x) 为 wasm_f32x4_splat(x)
#define GGML_F16x4_SET1(x)     wasm_f32x4_splat(x)
// 定义 GGML_F16x4_LOAD(x) 为 __wasm_f16x4_load(x)
#define GGML_F16x4_LOAD(x)     __wasm_f16x4_load(x)
// 定义 GGML_F16x4_STORE(x, y) 为 __wasm_f16x4_store(x, y)
#define GGML_F16x4_STORE(x, y) __wasm_f16x4_store(x, y)
// 定义 GGML_F16x4_FMA 为 GGML_F32x4_FMA
#define GGML_F16x4_FMA         GGML_F32x4_FMA
// 定义 GGML_F16x4_ADD 为 wasm_f32x4_add
#define GGML_F16x4_ADD         wasm_f32x4_add
// 定义 GGML_F16x4_MUL 为 wasm_f32x4_mul
#define GGML_F16x4_MUL         wasm_f32x4_mul
// 定义 GGML_F16x4_REDUCE(res, x) 为 对 x 进行规约操作
#define GGML_F16x4_REDUCE(res, x)                  \
{                                                  \
    // 初始化偏移量为 GGML_F16_ARR 的一半
    int offset = GGML_F16_ARR >> 1;                \
    // 对 x 进行规约操作
    for (int i = 0; i < offset; ++i) {             \
        x[i] = wasm_f32x4_add(x[i], x[offset+i]);  \
    }                                              \
    offset >>= 1;                                  \
    for (int i = 0; i < offset; ++i) {             \
        x[i] = wasm_f32x4_add(x[i], x[offset+i]);  \
    }                                              \
    offset >>= 1;                                  \
    # 遍历数组 x，从索引 0 到 offset-1
    for (int i = 0; i < offset; ++i) {             \
        # 将 x[i] 和 x[offset+i] 对应位置的元素相加，并将结果存入 x[i]
        x[i] = wasm_f32x4_add(x[i], x[offset+i]);  \
    }                                              \
    # 计算 x[0] 中每个元素的值并相加，得到最终结果
    res = wasm_f32x4_extract_lane(x[0], 0) +       \
          wasm_f32x4_extract_lane(x[0], 1) +       \
          wasm_f32x4_extract_lane(x[0], 2) +       \
          wasm_f32x4_extract_lane(x[0], 3);        \
// 定义 GGML_F16_VEC 为 GGML_F16x4
#define GGML_F16_VEC                GGML_F16x4
// 定义 GGML_F16_VEC_ZERO 为 GGML_F16x4_ZERO
#define GGML_F16_VEC_ZERO           GGML_F16x4_ZERO
// 定义 GGML_F16_VEC_SET1 为 GGML_F16x4_SET1
#define GGML_F16_VEC_SET1           GGML_F16x4_SET1
// 定义 GGML_F16_VEC_LOAD(p, i) 为 GGML_F16x4_LOAD(p)
#define GGML_F16_VEC_LOAD(p, i)     GGML_F16x4_LOAD(p)
// 定义 GGML_F16_VEC_STORE(p, r, i) 为 GGML_F16x4_STORE(p, r[i])
#define GGML_F16_VEC_STORE(p, r, i) GGML_F16x4_STORE(p, r[i])
// 定义 GGML_F16_VEC_FMA 为 GGML_F16x4_FMA
#define GGML_F16_VEC_FMA            GGML_F16x4_FMA
// 定义 GGML_F16_VEC_ADD 为 GGML_F16x4_ADD
#define GGML_F16_VEC_ADD            GGML_F16x4_ADD
// 定义 GGML_F16_VEC_MUL 为 GGML_F16x4_MUL
#define GGML_F16_VEC_MUL            GGML_F16x4_MUL
// 定义 GGML_F16_VEC_REDUCE 为 GGML_F16x4_REDUCE

#elif defined(__SSE3__)

// 定义 GGML_SIMD

// F32 SSE

// 定义 GGML_F32_STEP 为 32
#define GGML_F32_STEP 32
// 定义 GGML_F32_EPR 为 4
#define GGML_F32_EPR  4

// 定义 GGML_F32x4 为 __m128
#define GGML_F32x4         __m128
// 定义 GGML_F32x4_ZERO 为 _mm_setzero_ps()
#define GGML_F32x4_ZERO    _mm_setzero_ps()
// 定义 GGML_F32x4_SET1(x) 为 _mm_set1_ps(x)
#define GGML_F32x4_SET1(x) _mm_set1_ps(x)
// 定义 GGML_F32x4_LOAD 为 _mm_loadu_ps
#define GGML_F32x4_LOAD    _mm_loadu_ps
// 定义 GGML_F32x4_STORE 为 _mm_storeu_ps
#define GGML_F32x4_STORE   _mm_storeu_ps
#if defined(__FMA__)
    // 如果定义了 __FMA__，则定义 GGML_F32x4_FMA(a, b, c) 为 _mm_fmadd_ps(b, c, a)
    #define GGML_F32x4_FMA(a, b, c) _mm_fmadd_ps(b, c, a)
#else
    // 如果未定义 __FMA__，则定义 GGML_F32x4_FMA(a, b, c) 为 _mm_add_ps(_mm_mul_ps(b, c), a)
    #define GGML_F32x4_FMA(a, b, c) _mm_add_ps(_mm_mul_ps(b, c), a)
#endif
// 定义 GGML_F32x4_ADD 为 _mm_add_ps
#define GGML_F32x4_ADD     _mm_add_ps
// 定义 GGML_F32x4_MUL 为 _mm_mul_ps
#define GGML_F32x4_MUL     _mm_mul_ps
// 定义 GGML_F32x4_REDUCE(res, x) 的实现
#define GGML_F32x4_REDUCE(res, x)                                 \
{                                                                 \
    // 初始化偏移量为 GGML_F32_ARR 的一半
    int offset = GGML_F32_ARR >> 1;
    // 第一轮循环，将前半部分与后半部分相加
    for (int i = 0; i < offset; ++i) {                            
        x[i] = _mm_add_ps(x[i], x[offset+i]);                     
    }                                                             
    // 偏移量右移一位
    offset >>= 1;                                                 
    // 第二轮循环，将前半部分与后半部分相加
    for (int i = 0; i < offset; ++i) {                            
        x[i] = _mm_add_ps(x[i], x[offset+i]);                     
    }                                                             
    // 偏移量右移一位
    offset >>= 1;                                                 
    // 第三轮循环，将前半部分与后半部分相加
    for (int i = 0; i < offset; ++i) {                            
        x[i] = _mm_add_ps(x[i], x[offset+i]);                     
    }                                                             \  # 结束大括号，表示代码块结束
    const __m128 t0 = _mm_hadd_ps(x[0], x[0]);                    \  # 使用 SSE 指令对 x[0] 中的数据进行水平加法操作，结果存储在 t0 中
    res = _mm_cvtss_f32(_mm_hadd_ps(t0, t0));                     \  # 使用 SSE 指令对 t0 中的数据进行水平加法操作，将结果转换为 float 类型，存储在 res 中
// 结束宏定义区块
}

// TODO: 这个是否是最优解？

// 定义 GGML_F32_VEC 为 GGML_F32x4
#define GGML_F32_VEC        GGML_F32x4
// 定义 GGML_F32_VEC_ZERO 为 GGML_F32x4_ZERO
#define GGML_F32_VEC_ZERO   GGML_F32x4_ZERO
// 定义 GGML_F32_VEC_SET1 为 GGML_F32x4_SET1
#define GGML_F32_VEC_SET1   GGML_F32x4_SET1
// 定义 GGML_F32_VEC_LOAD 为 GGML_F32x4_LOAD
#define GGML_F32_VEC_LOAD   GGML_F32x4_LOAD
// 定义 GGML_F32_VEC_STORE 为 GGML_F32x4_STORE
#define GGML_F32_VEC_STORE  GGML_F32x4_STORE
// 定义 GGML_F32_VEC_FMA 为 GGML_F32x4_FMA
#define GGML_F32_VEC_FMA    GGML_F32x4_FMA
// 定义 GGML_F32_VEC_ADD 为 GGML_F32x4_ADD
#define GGML_F32_VEC_ADD    GGML_F32x4_ADD
// 定义 GGML_F32_VEC_MUL 为 GGML_F32x4_MUL
#define GGML_F32_VEC_MUL    GGML_F32x4_MUL
// 定义 GGML_F32_VEC_REDUCE 为 GGML_F32x4_REDUCE
#define GGML_F32_VEC_REDUCE GGML_F32x4_REDUCE

// 定义 GGML_F16_STEP 为 32
#define GGML_F16_STEP 32
// 定义 GGML_F16_EPR 为 4
#define GGML_F16_EPR  4

// 定义内联函数 __sse_f16x4_load，加载 ggml_fp16_t 类型的数据到 __m128 类型
static inline __m128 __sse_f16x4_load(ggml_fp16_t *x) {
    // 创建一个临时数组 tmp，存储转换后的数据
    float tmp[4];

    // 将 ggml_fp16_t 类型数据转换为 float 类型，并存储到 tmp 数组中
    tmp[0] = GGML_FP16_TO_FP32(x[0]);
    tmp[1] = GGML_FP16_TO_FP32(x[1]);
    tmp[2] = GGML_FP16_TO_FP32(x[2]);
    tmp[3] = GGML_FP16_TO_FP32(x[3]);

    // 返回加载后的 __m128 类型数据
    return _mm_loadu_ps(tmp);
}

// 定义内联函数 __sse_f16x4_store，将 __m128 类型数据存储到 ggml_fp16_t 类型数组中
static inline void __sse_f16x4_store(ggml_fp16_t *x, __m128 y) {
    // 创建一个临时数组 arr，存储转换后的数据
    float arr[4];

    // 将 __m128 类型数据存储到 arr 数组中
    _mm_storeu_ps(arr, y);

    // 将 arr 数组中的数据转换为 ggml_fp16_t 类型，并存储到 ggml_fp16_t 类型数组 x 中
    x[0] = GGML_FP32_TO_FP16(arr[0]);
    x[1] = GGML_FP32_TO_FP16(arr[1]);
    x[2] = GGML_FP32_TO_FP16(arr[2]);
    x[3] = GGML_FP32_TO_FP16(arr[3]);
}

// 定义 GGML_F32Cx4 为 __m128
#define GGML_F32Cx4             __m128
// 定义 GGML_F32Cx4_ZERO 为 _mm_setzero_ps()
#define GGML_F32Cx4_ZERO        _mm_setzero_ps()
// 定义 GGML_F32Cx4_SET1(x) 为 _mm_set1_ps(x)
#define GGML_F32Cx4_SET1(x)     _mm_set1_ps(x)
// 定义 GGML_F32Cx4_LOAD(x) 为 __sse_f16x4_load(x)
#define GGML_F32Cx4_LOAD(x)     __sse_f16x4_load(x)
// 定义 GGML_F32Cx4_STORE(x, y) 为 __sse_f16x4_store(x, y)
#define GGML_F32Cx4_STORE(x, y) __sse_f16x4_store(x, y)
// 定义 GGML_F32Cx4_FMA 为 GGML_F32x4_FMA
#define GGML_F32Cx4_FMA         GGML_F32x4_FMA
// 定义 GGML_F32Cx4_ADD 为 _mm_add_ps
#define GGML_F32Cx4_ADD         _mm_add_ps
// 定义 GGML_F32Cx4_MUL 为 _mm_mul_ps
#define GGML_F32Cx4_MUL         _mm_mul_ps
// 定义 GGML_F32Cx4_REDUCE 为 GGML_F32x4_REDUCE
#define GGML_F32Cx4_REDUCE      GGML_F32x4_REDUCE

// 定义 GGML_F16_VEC 为 GGML_F32Cx4
#define GGML_F16_VEC                 GGML_F32Cx4
// 定义 GGML_F16_VEC_ZERO 为 GGML_F32Cx4_ZERO
#define GGML_F16_VEC_ZERO            GGML_F32Cx4_ZERO
// 定义 GGML_F16_VEC_SET1 为 GGML_F32Cx4_SET1
#define GGML_F16_VEC_SET1            GGML_F32Cx4_SET1
// 定义 GGML_F16_VEC_LOAD(p, i) 为 GGML_F32Cx4_LOAD(p)
#define GGML_F16_VEC_LOAD(p, i)      GGML_F32Cx4_LOAD(p)
// 定义 GGML_F16_VEC_STORE(p, r, i) 为 GGML_F32Cx4_STORE(p, r[i])
#define GGML_F16_VEC_STORE(p, r, i)  GGML_F32Cx4_STORE(p, r[i])
// 定义 GGML_F16_VEC_FMA 为 GGML_F32Cx4_FMA
#define GGML_F16_VEC_FMA             GGML_F32Cx4_FMA
// 定义 GGML_F16_VEC_ADD 为 GGML_F32Cx4_ADD
#define GGML_F16_VEC_ADD             GGML_F32Cx4_ADD
// 定义 GGML_F16_VEC_MUL 为 GGML_F32Cx4_MUL
#define GGML_F16_VEC_MUL             GGML_F32Cx4_MUL
// 定义 GGML_F16_VEC_REDUCE 为 GGML_F32Cx4_REDUCE
#define GGML_F16_VEC_REDUCE          GGML_F32Cx4_REDUCE

// 结束宏定义区块
#endif

// GGML_F32_ARR / GGML_F16_ARR
// 定义了在每一步中要使用的寄存器数量
#ifdef GGML_SIMD
#define GGML_F32_ARR (GGML_F32_STEP/GGML_F32_EPR)
#define GGML_F16_ARR (GGML_F16_STEP/GGML_F16_EPR)
#endif

//
// 基本操作
//

// 设置整型数组中所有元素为指定值
inline static void ggml_vec_set_i8(const int n, int8_t * x, const int8_t v) { for (int i = 0; i < n; ++i) x[i] = v; }

// 设置短整型数组中所有元素为指定值
inline static void ggml_vec_set_i16(const int n, int16_t * x, const int16_t v) { for (int i = 0; i < n; ++i) x[i] = v; }

// 设置整型数组中所有元素为指定值
inline static void ggml_vec_set_i32(const int n, int32_t * x, const int32_t v) { for (int i = 0; i < n; ++i) x[i] = v; }

// 设置半精度浮点数数组中所有元素为指定值
inline static void ggml_vec_set_f16(const int n, ggml_fp16_t * x, const int32_t v) { for (int i = 0; i < n; ++i) x[i] = v; }

// 将两个单精度浮点数数组对应位置的元素相加，结果存入第三个数组
inline static void ggml_vec_add_f32 (const int n, float * z, const float * x, const float * y) { for (int i = 0; i < n; ++i) z[i]  = x[i] + y[i]; }

// 将单精度浮点数数组中的每个元素与一个常数相加，结果存入另一个数组
inline static void ggml_vec_add1_f32(const int n, float * z, const float * x, const float   v) { for (int i = 0; i < n; ++i) z[i]  = x[i] + v;    }

// 将两个单精度浮点数数组对应位置的元素相加，结果累加到第一个数组
inline static void ggml_vec_acc_f32 (const int n, float * y, const float * x)                  { for (int i = 0; i < n; ++i) y[i] += x[i];        }

// 将单精度浮点数数组中的每个元素与一个常数相加，结果累加到数组中
inline static void ggml_vec_acc1_f32(const int n, float * y, const float   v)                  { for (int i = 0; i < n; ++i) y[i] += v;           }

// 将两个单精度浮点数数组对应位置的元素相减，结果存入第三个数组
inline static void ggml_vec_sub_f32 (const int n, float * z, const float * x, const float * y) { for (int i = 0; i < n; ++i) z[i]  = x[i] - y[i]; }

// 将单精度浮点数数组中所有元素设置为指定值
inline static void ggml_vec_set_f32 (const int n, float * x, const float   v)                  { for (int i = 0; i < n; ++i) x[i]  = v;           }

// 复制一个单精度浮点数数组到另一个数组
inline static void ggml_vec_cpy_f32 (const int n, float * y, const float * x)                  { for (int i = 0; i < n; ++i) y[i]  = x[i];        }

// 将单精度浮点数数组中所有元素取负值
inline static void ggml_vec_neg_f32 (const int n, float * y, const float * x)                  { for (int i = 0; i < n; ++i) y[i]  = -x[i];       }
// 计算两个 float 数组的元素对应位置相乘，结果存储在第三个数组中
inline static void ggml_vec_mul_f32 (const int n, float * z, const float * x, const float * y) { for (int i = 0; i < n; ++i) z[i]  = x[i]*y[i];   }
// 计算两个 float 数组的元素对应位置相除，结果存储在第三个数组中
inline static void ggml_vec_div_f32 (const int n, float * z, const float * x, const float * y) { for (int i = 0; i < n; ++i) z[i]  = x[i]/y[i];   }

// 计算两个 float 数组的点积
static void ggml_vec_dot_f32(const int n, float * restrict s, const float * restrict x, const float * restrict y) {
#ifdef GGML_SIMD
    // 初始化变量
    float sumf = 0.0f;
    const int np = (n & ~(GGML_F32_STEP - 1));

    GGML_F32_VEC sum[GGML_F32_ARR] = { GGML_F32_VEC_ZERO };

    GGML_F32_VEC ax[GGML_F32_ARR];
    GGML_F32_VEC ay[GGML_F32_ARR];

    // 使用 SIMD 计算点积
    for (int i = 0; i < np; i += GGML_F32_STEP) {
        for (int j = 0; j < GGML_F32_ARR; j++) {
            ax[j] = GGML_F32_VEC_LOAD(x + i + j*GGML_F32_EPR);
            ay[j] = GGML_F32_VEC_LOAD(y + i + j*GGML_F32_EPR);

            sum[j] = GGML_F32_VEC_FMA(sum[j], ax[j], ay[j]);
        }
    }

    // 将 sum0..sum3 累加到 sum0
    GGML_F32_VEC_REDUCE(sumf, sum);

    // 处理剩余部分
    for (int i = np; i < n; ++i) {
        sumf += x[i]*y[i];
    }
#else
    // 如果不支持 SIMD，则使用标量计算
    ggml_float sumf = 0.0;
    for (int i = 0; i < n; ++i) {
        sumf += (ggml_float)(x[i]*y[i]);
    }
#endif

    // 将结果存储在指定地址
    *s = sumf;
}

// 计算两个 ggml_fp16_t 数组的点积
static void ggml_vec_dot_f16(const int n, float * restrict s, ggml_fp16_t * restrict x, ggml_fp16_t * restrict y) {
    ggml_float sumf = 0.0;

#if defined(GGML_SIMD)
    const int np = (n & ~(GGML_F16_STEP - 1));

    GGML_F16_VEC sum[GGML_F16_ARR] = { GGML_F16_VEC_ZERO };

    GGML_F16_VEC ax[GGML_F16_ARR];
    GGML_F16_VEC ay[GGML_F16_ARR];

    // 使用 SIMD 计算点积
    for (int i = 0; i < np; i += GGML_F16_STEP) {
        for (int j = 0; j < GGML_F16_ARR; j++) {
            ax[j] = GGML_F16_VEC_LOAD(x + i + j*GGML_F16_EPR, j);
            ay[j] = GGML_F16_VEC_LOAD(y + i + j*GGML_F16_EPR, j);

            sum[j] = GGML_F16_VEC_FMA(sum[j], ax[j], ay[j]);
        }
    }

    // 将 sum0..sum3 累加到 sum0
    GGML_F16_VEC_REDUCE(sumf, sum);

    // 处理剩余部分
    # 从索引 np 开始遍历到索引 n-1
    for (int i = np; i < n; ++i) {
        # 计算 x[i] 和 y[i] 的乘积，并将结果转换为单精度浮点数后累加到 sumf 中
        sumf += (ggml_float)(GGML_FP16_TO_FP32(x[i])*GGML_FP16_TO_FP32(y[i]));
    }
// 如果不支持 SIMD 指令集，则使用普通循环计算向量点积
#else
    for (int i = 0; i < n; ++i) {
        // 计算两个半精度浮点数的乘积，并将结果累加到 sumf 中
        sumf += (ggml_float)(GGML_FP16_TO_FP32(x[i])*GGML_FP16_TO_FP32(y[i]));
    }
#endif

    // 将计算得到的点积结果存储到输出数组 s 中
    *s = sumf;
}

// 一次计算 GGML_VEC_DOT_UNROLL 个点积
// xs - x 行的字节步长
inline static void ggml_vec_dot_f16_unroll(const int n, const int xs, float * restrict s, void * restrict xv, ggml_fp16_t * restrict y) {
    // 初始化 GGML_VEC_DOT_UNROLL 个和为 0.0
    ggml_float sumf[GGML_VEC_DOT_UNROLL] = { 0.0 };

    // GGML_VEC_DOT_UNROLL 个 x 指针数组
    ggml_fp16_t * restrict x[GGML_VEC_DOT_UNROLL];

    // 初始化 GGML_VEC_DOT_UNROLL 个 x 指针
    for (int i = 0; i < GGML_VEC_DOT_UNROLL; ++i) {
        x[i] = (ggml_fp16_t *) ((char *) xv + i*xs);
    }

#if defined(GGML_SIMD)
    // 计算 n 的 GGML_F16_STEP 的倍数
    const int np = (n & ~(GGML_F16_STEP - 1));

    // 初始化 GGML_VEC_DOT_UNROLL 个和向量数组
    GGML_F16_VEC sum[GGML_VEC_DOT_UNROLL][GGML_F16_ARR] = { { GGML_F16_VEC_ZERO } };

    GGML_F16_VEC ax[GGML_F16_ARR];
    GGML_F16_VEC ay[GGML_F16_ARR];

    // 使用 SIMD 指令集计算点积
    for (int i = 0; i < np; i += GGML_F16_STEP) {
        for (int j = 0; j < GGML_F16_ARR; j++) {
            // 加载 y 到 ay 向量
            ay[j] = GGML_F16_VEC_LOAD(y + i + j*GGML_F16_EPR, j);

            for (int k = 0; k < GGML_VEC_DOT_UNROLL; ++k) {
                // 加载 x 到 ax 向量
                ax[j] = GGML_F16_VEC_LOAD(x[k] + i + j*GGML_F16_EPR, j);

                // 使用 FMA 指令计算点积
                sum[k][j] = GGML_F16_VEC_FMA(sum[k][j], ax[j], ay[j]);
            }
        }
    }

    // 将 sum0..sum3 向量累加到 sum0 向量
    for (int k = 0; k < GGML_VEC_DOT_UNROLL; ++k) {
        GGML_F16_VEC_REDUCE(sumf[k], sum[k]);
    }

    // 处理剩余的数据
    for (int i = np; i < n; ++i) {
        for (int j = 0; j < GGML_VEC_DOT_UNROLL; ++j) {
            // 计算剩余数据的点积
            sumf[j] += (ggml_float)(GGML_FP16_TO_FP32(x[j][i])*GGML_FP16_TO_FP32(y[i]));
        }
    }
#else
    // 如果不支持 SIMD 指令集，则使用普通循环计算向量点积
    for (int i = 0; i < n; ++i) {
        for (int j = 0; j < GGML_VEC_DOT_UNROLL; ++j) {
            // 计算两个半精度浮点数的乘积，并将结果累加到 sumf 中
            sumf[j] += (ggml_float)(GGML_FP16_TO_FP32(x[j][i])*GGML_FP16_TO_FP32(y[i]));
        }
    }
#endif

    // 将计算得到的点积结果存储到输出数组 s 中
    for (int i = 0; i < GGML_VEC_DOT_UNROLL; ++i) {
        s[i] = sumf[i];
    }
}

// 使用 FMA 指令计算浮点数乘加
inline static void ggml_vec_mad_f32(const int n, float * restrict y, const float * restrict x, const float v) {
#if defined(GGML_SIMD)
    // 计算 np，np 是 n 与 GGML_F32_STEP - 1 的按位与结果
    const int np = (n & ~(GGML_F32_STEP - 1));

    // 将标量 v 转换为 GGML_F32_VEC 类型
    GGML_F32_VEC vx = GGML_F32_VEC_SET1(v);

    // 创建 GGML_F32_ARR 个 GGML_F32_VEC 类型的数组 ax 和 ay
    GGML_F32_VEC ax[GGML_F32_ARR];
    GGML_F32_VEC ay[GGML_F32_ARR];

    // 循环处理每个 GGML_F32_STEP 大小的数据块
    for (int i = 0; i < np; i += GGML_F32_STEP) {
        // 循环处理 GGML_F32_ARR 个数据块
        for (int j = 0; j < GGML_F32_ARR; j++) {
            // 从地址 x + i + j*GGML_F32_EPR 处加载 GGML_F32_VEC 类型数据到 ax[j]
            ax[j] = GGML_F32_VEC_LOAD(x + i + j*GGML_F32_EPR);
            // 从地址 y + i + j*GGML_F32_EPR 处加载 GGML_F32_VEC 类型数据到 ay[j]
            ay[j] = GGML_F32_VEC_LOAD(y + i + j*GGML_F32_EPR);
            // 计算 ay[j] = ay[j] + ax[j]*vx
            ay[j] = GGML_F32_VEC_FMA(ay[j], ax[j], vx);

            // 将 ay[j] 存储到地址 y + i + j*GGML_F32_EPR 处
            GGML_F32_VEC_STORE(y + i + j*GGML_F32_EPR, ay[j]);
        }
    }

    // 处理剩余的数据
    for (int i = np; i < n; ++i) {
        // 计算 y[i] = y[i] + x[i]*v
        y[i] += x[i]*v;
    }
// 如果不满足条件，使用标量方式计算向量乘法
#else
    // 对每个元素进行标量乘法
    for (int i = 0; i < n; ++i) {
        y[i] += x[i]*v;
    }
#endif
}

// xs 和 vs 是 x 和 v 的字节步长
// 对长度为 n 的向量进行乘加操作，使用浮点数数组 y、xv、vv
inline static void ggml_vec_mad_f32_unroll(const int n, const int xs, const int vs, float * restrict y, const float * restrict xv, const float * restrict vv) {

    const float * restrict x[GGML_VEC_MAD_UNROLL];
    const float * restrict v[GGML_VEC_MAD_UNROLL];

    // 初始化 x 和 v 数组
    for (int i = 0; i < GGML_VEC_MAD_UNROLL; ++i) {
        x[i] = (const float *) ((const char *) xv + i*xs);
        v[i] = (const float *) ((const char *) vv + i*vs);
    }

#if defined(GGML_SIMD)
    // 计算 np，使其为 GGML_F32_STEP 的倍数
    const int np = (n & ~(GGML_F32_STEP - 1));

    GGML_F32_VEC vx[GGML_VEC_MAD_UNROLL];

    // 初始化 vx 数组
    for (int k = 0; k < GGML_VEC_MAD_UNROLL; ++k) {
        vx[k] = GGML_F32_VEC_SET1(v[k][0]);
    }

    GGML_F32_VEC ax[GGML_VEC_MAD_UNROLL][GGML_F32_ARR];
    GGML_F32_VEC ay[GGML_F32_ARR];

    // 使用 SIMD 计算乘加操作
    for (int i = 0; i < np; i += GGML_F32_STEP) {
        for (int j = 0; j < GGML_F32_ARR; j++) {
            ay[j] = GGML_F32_VEC_LOAD(y + i + j*GGML_F32_EPR);

            for (int k = 0; k < GGML_VEC_MAD_UNROLL; ++k) {
                ax[k][j] = GGML_F32_VEC_LOAD(x[k] + i + j*GGML_F32_EPR);
                ay[j] = GGML_F32_VEC_FMA(ay[j], ax[k][j], vx[k]);
            }

            GGML_F32_VEC_STORE(y + i + j*GGML_F32_EPR, ay[j]);
        }
    }

    // 处理剩余的元素
    for (int k = 0; k < GGML_VEC_MAD_UNROLL; ++k) {
        for (int i = np; i < n; ++i) {
            y[i] += x[k][i]*v[k][0];
        }
    }
#else
    // 如果不支持 SIMD，使用标量方式计算向量乘法
    for (int k = 0; k < GGML_VEC_MAD_UNROLL; ++k) {
        for (int i = 0; i < n; ++i) {
            y[i] += x[k][i]*v[k][0];
        }
    }
#endif
}

// 对长度为 n 的向量进行标量乘法
inline static void ggml_vec_scale_f32(const int n, float * y, const float   v) {
#if defined(GGML_USE_ACCELERATE)
    // 使用加速库进行标量乘法
    vDSP_vsmul(y, 1, &v, y, 1, n);
#elif defined(GGML_SIMD)
    // 计算向下取整到 GGML_F32_STEP 的倍数
    const int np = (n & ~(GGML_F32_STEP - 1));

    // 将标量 v 转换为 GGML_F32_VEC 类型
    GGML_F32_VEC vx = GGML_F32_VEC_SET1(v);

    // 创建 GGML_F32_ARR 个 GGML_F32_VEC 类型的数组
    GGML_F32_VEC ay[GGML_F32_ARR];

    // 遍历数组中的元素，每次处理 GGML_F32_STEP 个元素
    for (int i = 0; i < np; i += GGML_F32_STEP) {
        // 遍历 GGML_F32_ARR 个元素
        for (int j = 0; j < GGML_F32_ARR; j++) {
            // 从内存中加载 y 数组中的数据到 ay 数组中
            ay[j] = GGML_F32_VEC_LOAD(y + i + j*GGML_F32_EPR);
            // 将 ay 数组中的数据与 vx 相乘
            ay[j] = GGML_F32_VEC_MUL(ay[j], vx);
            // 将计算结果存储回 y 数组中
            GGML_F32_VEC_STORE(y + i + j*GGML_F32_EPR, ay[j]);
        }
    }

    // 处理剩余的元素
    for (int i = np; i < n; ++i) {
        // 将 y 数组中剩余的元素与标量 v 相乘
        y[i] *= v;
    }
// 对于标量情况，将向量中的每个元素乘以标量
#else
    // scalar
    for (int i = 0; i < n; ++i) {
        y[i] *= v;
    }
#endif
}

// 计算向量的二范数
inline static void ggml_vec_norm_f32 (const int n, float * s, const float * x) { ggml_vec_dot_f32(n, s, x, x); *s = sqrtf(*s);   }

// 计算向量的平方
inline static void ggml_vec_sqr_f32  (const int n, float * y, const float * x) { for (int i = 0; i < n; ++i) y[i] = x[i]*x[i];   }

// 计算向量的平方根
inline static void ggml_vec_sqrt_f32 (const int n, float * y, const float * x) { for (int i = 0; i < n; ++i) y[i] = sqrtf(x[i]); }

// 计算向量的自然对数
inline static void ggml_vec_log_f32  (const int n, float * y, const float * x) { for (int i = 0; i < n; ++i) y[i] = logf(x[i]);   }

// 计算向量的绝对值
inline static void ggml_vec_abs_f32  (const int n, float * y, const float * x) { for (int i = 0; i < n; ++i) y[i] = fabsf(x[i]); }

// 计算向量的符号函数
inline static void ggml_vec_sgn_f32  (const int n, float * y, const float * x) { for (int i = 0; i < n; ++i) y[i] = (x[i] > 0.f) ? 1.f : ((x[i] < 0.f) ? -1.f : 0.f); }

// 计算向量的阶跃函数
inline static void ggml_vec_step_f32 (const int n, float * y, const float * x) { for (int i = 0; i < n; ++i) y[i] = (x[i] > 0.f) ? 1.f : 0.f; }

// 计算向量的双曲正切函数
inline static void ggml_vec_tanh_f32 (const int n, float * y, const float * x) { for (int i = 0; i < n; ++i) y[i] = tanhf(x[i]);  }

// 计算向量的指数线性单元函数
inline static void ggml_vec_elu_f32  (const int n, float * y, const float * x) { for (int i = 0; i < n; ++i) y[i] = (x[i] > 0.f) ? x[i] : expf(x[i])-1; }

// 计算向量的修正线性单元函数
inline static void ggml_vec_relu_f32 (const int n, float * y, const float * x) { for (int i = 0; i < n; ++i) y[i] = (x[i] > 0.f) ? x[i] : 0.f; }

// 计算向量的泄漏修正线性单元函数
inline static void ggml_vec_leaky_relu_f32 (const int n, float * y, const float * x, const float ns) { for (int i = 0; i < n; ++i) y[i] = ((x[i] > 0.f) ? x[i] : 0.f) + ns * ((x[i] < 0.0f) ? x[i] : 0.f); }

// 计算向量的硬切线函数
// TODO: 优化性能
inline static void ggml_vec_hardswish_f32 (const int n, float * y, const float * x) { for (int i = 0; i < n; ++i) y[i] = x[i] * fminf(1.0f, fmaxf(0.0f, (x[i] + 3.0f) / 6.0f)); }
// 在给定的浮点数数组 x 上应用硬 sigmoid 函数，将结果存储在数组 y 中
inline static void ggml_vec_hardsigmoid_f32 (const int n, float * y, const float * x) { for (int i = 0; i < n; ++i) y[i] = fminf(1.0f, fmaxf(0.0f, (x[i] + 3.0f) / 6.0f)); }

// 定义 GELU 函数所需的常量
static const float GELU_COEF_A     = 0.044715f;
static const float GELU_QUICK_COEF = -1.702f;
static const float SQRT_2_OVER_PI  = 0.79788456080286535587989211986876f;

// 计算 GELU 函数在单个浮点数 x 上的值
inline static float ggml_gelu_f32(float x) {
    return 0.5f*x*(1.0f + tanhf(SQRT_2_OVER_PI*x*(1.0f + GELU_COEF_A*x*x)));
}

// 在给定的半精度浮点数数组 x 上应用 GELU 函数，将结果存储在数组 y 中
inline static void ggml_vec_gelu_f16(const int n, ggml_fp16_t * y, const ggml_fp16_t * x) {
    const uint16_t * i16 = (const uint16_t *) x;
    for (int i = 0; i < n; ++i) {
        y[i] = ggml_table_gelu_f16[i16[i]];
    }
}

#ifdef GGML_GELU_FP16
// 在给定的单精度浮点数数组 x 上应用 GELU 函数，将结果存储在数组 y 中
inline static void ggml_vec_gelu_f32(const int n, float * y, const float * x) {
    uint16_t t;
    for (int i = 0; i < n; ++i) {
        ggml_fp16_t fp16 = GGML_FP32_TO_FP16(x[i]);
        memcpy(&t, &fp16, sizeof(uint16_t));
        y[i] = GGML_FP16_TO_FP32(ggml_table_gelu_f16[t]);
    }
}
#else
// 在给定的单精度浮点数数组 x 上应用 GELU 函数，将结果存储在数组 y 中
inline static void ggml_vec_gelu_f32(const int n, float * y, const float * x) {
    for (int i = 0; i < n; ++i) {
        y[i] = ggml_gelu_f32(x[i]);
    }
}
#endif

// 计算快速 GELU 函数在单个浮点数 x 上的值
inline static float ggml_gelu_quick_f32(float x) {
    return x*(1.0f/(1.0f+expf(GELU_QUICK_COEF*x)));
}

// 在给定的单精度浮点数数组 x 上应用快速 GELU 函数，将结果存储在数组 y 中
#ifdef GGML_GELU_QUICK_FP16
inline static void ggml_vec_gelu_quick_f32(const int n, float * y, const float * x) {
    uint16_t t;
    for (int i = 0; i < n; ++i) {
        ggml_fp16_t fp16 = GGML_FP32_TO_FP16(x[i]);
        memcpy(&t, &fp16, sizeof(uint16_t));
        y[i] = GGML_FP16_TO_FP32(ggml_table_gelu_quick_f16[t]);
    }
}
#else
// 在给定的单精度浮点数数组 x 上应用快速 GELU 函数，将结果存储在数组 y 中
inline static void ggml_vec_gelu_quick_f32(const int n, float * y, const float * x) {
    # 遍历整数 i 从 0 到 n-1
    for (int i = 0; i < n; ++i) {
        # 对输入数组 x 中的每个元素进行 ggml_gelu_quick_f32 函数的计算，并将结果存储在输出数组 y 中对应的位置
        y[i] = ggml_gelu_quick_f32(x[i]);
    }
// 结束条件判断，检查是否定义了宏
#endif

// Sigmoid Linear Unit (SiLU) 函数，计算输入 x 的 SiLU 函数值
inline static float ggml_silu_f32(float x) {
    return x/(1.0f + expf(-x));
}

#ifdef GGML_SILU_FP16
// 如果定义了 GGML_SILU_FP16 宏，则使用 f32 到 f16 的转换计算 SiLU 函数
inline static void ggml_vec_silu_f32(const int n, float * y, const float * x) {
    uint16_t t;
    for (int i = 0; i < n; ++i) {
        ggml_fp16_t fp16 = GGML_FP32_TO_FP16(x[i]);
        memcpy(&t, &fp16, sizeof(uint16_t));
        y[i] = GGML_FP16_TO_FP32(ggml_table_silu_f16[t]);
    }
}
#else
// 如果未定义 GGML_SILU_FP16 宏，则直接计算 SiLU 函数
inline static void ggml_vec_silu_f32(const int n, float * y, const float * x) {
    for (int i = 0; i < n; ++i) {
        y[i] = ggml_silu_f32(x[i]);
    }
}
#endif

// 计算 SiLU 函数的反向传播，返回输入 x 和导数 dy 对应位置的梯度
inline static float ggml_silu_backward_f32(float x, float dy) {
    const float s = 1.0f/(1.0f + expf(-x));
    return dy*s*(1.0f + x*(1.0f - s));
}

#ifdef GGML_SILU_FP16
// 如果定义了 GGML_SILU_FP16 宏，则使用 f32 到 f16 的转换计算 SiLU 函数的反向传播
inline static void ggml_vec_silu_backward_f32(const int n, float * dx, const float * x, const float * dy) {
    for (int i = 0; i < n; ++i) {
        ggml_fp16_t fp16 = GGML_FP32_TO_FP16(x[i]);
        float usedx = GGML_FP16_TO_FP32(fp16);
        dx[i] = ggml_silu_backward_f32(usedx, dy[i]);
    }
}
#else
// 如果未定义 GGML_SILU_FP16 宏，则直接计算 SiLU 函数的反向传播
inline static void ggml_vec_silu_backward_f32(const int n, float * dx, const float * x, const float * dy) {
    for (int i = 0; i < n; ++i) {
        dx[i] = ggml_silu_backward_f32(x[i], dy[i]);
    }
}
#endif

// 计算输入数组 x 的和，结果保存在 s 中
inline static void ggml_vec_sum_f32(const int n, float * s, const float * x) {
#ifndef GGML_USE_ACCELERATE
    // 如果未定义 GGML_USE_ACCELERATE 宏，则使用循环计算和
    ggml_float sum = 0.0;
    for (int i = 0; i < n; ++i) {
        sum += (ggml_float)x[i];
    }
    *s = sum;
#else
    // 如果定义了 GGML_USE_ACCELERATE 宏，则使用加速库计算和
    vDSP_sve(x, 1, s, n);
#endif
}
// 计算浮点数组的和，存储在指定的 ggml_float 类型变量中
inline static void ggml_vec_sum_f32_ggf(const int n, ggml_float * s, const float * x) {
    // 初始化和为 0
    ggml_float sum = 0.0;
    // 遍历数组，累加每个元素到和中
    for (int i = 0; i < n; ++i) {
        sum += (ggml_float)x[i];
    }
    // 将计算得到的和存储在指定变量中
    *s = sum;
}

// 计算半精度浮点数组的和，存储在指定的 float 类型变量中
inline static void ggml_vec_sum_f16_ggf(const int n, float * s, const ggml_fp16_t * x) {
    // 初始化和为 0
    float sum = 0.0f;
    // 遍历数组，将每个元素转换为单精度浮点数后累加到和中
    for (int i = 0; i < n; ++i) {
        sum += GGML_FP16_TO_FP32(x[i]);
    }
    // 将计算得到的和存储在指定变量中
    *s = sum;
}

// 计算浮点数组中的最大值，存储在指定的 float 类型变量中
inline static void ggml_vec_max_f32(const int n, float * s, const float * x) {
#ifndef GGML_USE_ACCELERATE
    // 初始化最大值为负无穷
    float max = -INFINITY;
    // 遍历数组，更新最大值
    for (int i = 0; i < n; ++i) {
        max = MAX(max, x[i]);
    }
    // 将计算得到的最大值存储在指定变量中
    *s = max;
#else
    // 使用加速库计算数组中的最大值
    vDSP_maxv(x, 1, s, n);
#endif
}

// 计算浮点数组的 L2 范数的倒数，存储在指定的 float 类型变量中
inline static void ggml_vec_norm_inv_f32(const int n, float * s, const float * x) {
    // 计算数组的 L2 范数
    ggml_vec_norm_f32(n, s, x);
    // 计算 L2 范数的倒数
    *s = 1.f/(*s);
}

// 计算浮点数组中的最大值的索引，存储在指定的 int 类型变量中
inline static void ggml_vec_argmax_f32(const int n, int * s, const float * x) {
    // 初始化最大值为负无穷，索引为 0
    float max = -INFINITY;
    int idx = 0;
    // 遍历数组，更新最大值和对应的索引
    for (int i = 0; i < n; ++i) {
        max = MAX(max, x[i]);
        if (max == x[i]) { idx = i; }
    }
    // 将计算得到的最大值的索引存储在指定变量中
    *s = idx;
}

//
// data types
//

// 定义 GGML_OP_NAME 数组，存储操作名称
static const char * GGML_OP_NAME[GGML_OP_COUNT] = {
    "NONE",

    // 其他操作名称
    "DUP",
    "ADD",
    "ADD1",
    "ACC",
    // 省略部分操作名称...
};
    "WIN_PART",  # 定义常量，表示窗口分区
    "WIN_UNPART",  # 定义常量，表示窗口未分区
    "GET_REL_POS",  # 定义常量，表示获取相对位置
    "ADD_REL_POS",  # 定义常量，表示添加相对位置

    "UNARY",  # 定义常量，表示一元操作

    "MAP_UNARY",  # 定义常量，表示一元映射操作
    "MAP_BINARY",  # 定义常量，表示二元映射操作

    "MAP_CUSTOM1_F32",  # 定义常量，表示自定义1的32位浮点数映射操作
    "MAP_CUSTOM2_F32",  # 定义常量，表示自定义2的32位浮点数映射操作
    "MAP_CUSTOM3_F32",  # 定义常量，表示自定义3的32位浮点数映射操作

    "MAP_CUSTOM1",  # 定义常量，表示自定义1的映射操作
    "MAP_CUSTOM2",  # 定义常量，表示自定义2的映射操作
    "MAP_CUSTOM3",  # 定义常量，表示自定义3的映射操作

    "CROSS_ENTROPY_LOSS",  # 定义常量，表示交叉熵损失
    "CROSS_ENTROPY_LOSS_BACK",  # 定义常量，表示交叉熵损失的反向传播
// 静态断言，检查 GGML_OP_COUNT 是否等于 72，如果不等于则触发断言错误
static_assert(GGML_OP_COUNT == 72, "GGML_OP_COUNT != 72");

// 定义包含 GGML_OP_COUNT 个字符串指针的数组 GGML_OP_SYMBOL，表示不同操作的符号
static const char * GGML_OP_SYMBOL[GGML_OP_COUNT] = {
    "none",

    "x",
    "x+y",
    "x+y",
    "view(x,nb,offset)+=y->x",
    "x-y",
    "x*y",
    "x/y",
    "x^2",
    "√x",
    "log(x)",
    "Σx",
    "Σx_k",
    "Σx/n",
    "argmax(x)",
    "repeat(x)",
    "repeat_back(x)",
    "concat(x, y)",
    "silu_back(x)",
    "norm(x)",
    "rms_norm(x)",
    "rms_norm_back(x)",
    "group_norm(x)",

    "X*Y",
    "X[i]*Y",
    "X*Y",

    "x*v",
    "y->view(x)",
    "x->y",
    "cont(x)",
    "reshape(x)",
    "view(x)",
    "permute(x)",
    "transpose(x)",
    "get_rows(x)",
    "get_rows_back(x)",
    "diag(x)",
    "diag_mask_inf(x)",
    "diag_mask_zero(x)",
    "soft_max(x)",
    "soft_max_back(x)",
    "rope(x)",
    "rope_back(x)",
    "alibi(x)",
    "clamp(x)",
    "conv_transpose_1d(x)",
    "im2col(x)",
    "conv_transpose_2d(x)",
    "pool_1d(x)",
    "pool_2d(x)",
    "upscale(x)",
    "pad(x)",
    "argsort(x)",
    "leaky_relu(x)",

    "flash_attn(x)",
    "flash_ff(x)",
    "flash_attn_back(x)",
    "win_part(x)",
    "win_unpart(x)",
    "get_rel_pos(x)",
    "add_rel_pos(x)",

    "unary(x)",

    "f(x)",
    "f(x,y)",

    "custom_f32(x)",
    "custom_f32(x,y)",
    "custom_f32(x,y,z)",

    "custom(x)",
    "custom(x,y)",
    "custom(x,y,z)",

    "cross_entropy_loss(x,y)",
    "cross_entropy_loss_back(x,y)",
};

// 静态断言，检查 GGML_OP_COUNT 是否等于 72，如果不等于则触发断言错误
static_assert(GGML_OP_COUNT == 72, "GGML_OP_COUNT != 72");

// 静态断言，检查 GGML_OP_POOL_COUNT 是否等于 2，如果不等于则触发断言错误
static_assert(GGML_OP_POOL_COUNT == 2, "GGML_OP_POOL_COUNT != 2");

// 定义包含 GGML_UNARY_OP_COUNT 个字符串指针的数组 GGML_UNARY_OP_NAME，表示不同一元操作的名称
static const char * GGML_UNARY_OP_NAME[GGML_UNARY_OP_COUNT] = {
    "ABS",
    "SGN",
    "NEG",
    "STEP",
    "TANH",
    "ELU",
    "RELU",
    "GELU",
    "GELU_QUICK",
    "SILU",
    "HARDSWISH",
    "HARDSIGMOID",
};

// 静态断言，检查 GGML_UNARY_OP_COUNT 是否等于 12，如果不等于则触发断言错误
static_assert(GGML_UNARY_OP_COUNT == 12, "GGML_UNARY_OP_COUNT != 12");

// 静态断言，检查结构体 ggml_object 的大小是否是 GGML_MEM_ALIGN 的倍数，如果不是则触发断言错误
static_assert(sizeof(struct ggml_object)%GGML_MEM_ALIGN == 0, "ggml_object size must be a multiple of GGML_MEM_ALIGN");
// 确保 ggml_tensor 结构体的大小是 GGML_MEM_ALIGN 的倍数
static_assert(sizeof(struct ggml_tensor)%GGML_MEM_ALIGN == 0, "ggml_tensor size must be a multiple of GGML_MEM_ALIGN");

// 警告：
// 配置错误可能导致难以理解的问题：
// * 最好情况下会导致崩溃或输出无意义的内容。
// * 最坏情况下会导致输出略有不同但难以察觉的内容。
//
// 当操作的任何分支需要某个 pass 时，操作必须启用 INIT 或 FINALIZE。
// 注意编译选项（例如，GGML_USE_xxx）。
static bool GGML_OP_HAS_INIT    [GGML_OP_COUNT] = { 0 };
static bool GGML_OP_HAS_FINALIZE[GGML_OP_COUNT] = { 0 };

static void ggml_setup_op_has_task_pass(void) {
    {   // INIT
        bool * p = GGML_OP_HAS_INIT;

        p[GGML_OP_ACC                    ] = true;
        p[GGML_OP_MUL_MAT                ] = true;
        p[GGML_OP_MUL_MAT_ID             ] = true;
        p[GGML_OP_OUT_PROD               ] = true;
        p[GGML_OP_SET                    ] = true;
        p[GGML_OP_GET_ROWS_BACK          ] = true;
        p[GGML_OP_DIAG_MASK_INF          ] = true;
        p[GGML_OP_DIAG_MASK_ZERO         ] = true;
        p[GGML_OP_CONV_TRANSPOSE_1D      ] = true;
        p[GGML_OP_CONV_TRANSPOSE_2D      ] = true;
        p[GGML_OP_FLASH_ATTN_BACK        ] = true;
        p[GGML_OP_CROSS_ENTROPY_LOSS     ] = true;
        p[GGML_OP_ADD_REL_POS            ] = true;
    }

    {   // FINALIZE
        bool * p = GGML_OP_HAS_FINALIZE;

        p[GGML_OP_CROSS_ENTROPY_LOSS     ] = true;
    }
}

//
// ggml context
//

// ggml 上下文结构体
struct ggml_context {
    size_t mem_size;
    void * mem_buffer;
    bool   mem_buffer_owned;
    bool   no_alloc;
    bool   no_alloc_save; // 当使用临时缓冲区时，用于保存 no_alloc 状态

    int    n_objects;

    struct ggml_object * objects_begin;
    struct ggml_object * objects_end;

    struct ggml_scratch scratch;
    struct ggml_scratch scratch_save;
};

// ggml 上下文容器结构体
struct ggml_context_container {
    bool used;

    struct ggml_context context;
};

//
// NUMA support
//
// 定义最大 NUMA 节点数和最大 CPU 数
#define GGML_NUMA_MAX_NODES 8
#define GGML_NUMA_MAX_CPUS 512

// 定义 NUMA 节点结构体，包含该节点上的硬件线程和线程数
struct ggml_numa_node {
    uint32_t cpus[GGML_NUMA_MAX_CPUS]; // hardware threads on this node
    uint32_t n_cpus;
};

// 定义 NUMA 节点集合结构体，包含所有节点和总线程数
struct ggml_numa_nodes {
    struct ggml_numa_node nodes[GGML_NUMA_MAX_NODES];
    uint32_t n_nodes;
    uint32_t total_cpus; // hardware threads on system
};

//
// ggml state
//

// 定义 ggml 状态结构体，包含上下文容器和 NUMA 节点集合
struct ggml_state {
    struct ggml_context_container contexts[GGML_MAX_CONTEXTS];
    struct ggml_numa_nodes numa;
};

// 全局状态变量
static struct ggml_state g_state;
static atomic_int g_state_barrier = 0;

// 使用自旋锁实现的屏障
inline static void ggml_critical_section_start(void) {
    int processing = atomic_fetch_add(&g_state_barrier, 1);

    while (processing > 0) {
        // 等待其他线程完成
        atomic_fetch_sub(&g_state_barrier, 1);
        sched_yield(); // TODO: reconsider this
        processing = atomic_fetch_add(&g_state_barrier, 1);
    }
}

// 结束临界区
inline static void ggml_critical_section_end(void) {
    atomic_fetch_sub(&g_state_barrier, 1);
}

// 初始化 NUMA
void ggml_numa_init(void) {
    if (g_state.numa.n_nodes > 0) {
        fprintf(stderr, "ggml_numa_init: NUMA already initialized\n");

        return;
    }

#ifdef __linux__
    struct stat st;
    char path[256];
    int rv;

    // 枚举节点
    while (g_state.numa.n_nodes < GGML_NUMA_MAX_NODES) {
        rv = snprintf(path, sizeof(path), "/sys/devices/system/node/node%u", g_state.numa.n_nodes);
        GGML_ASSERT(rv > 0 && (unsigned)rv < sizeof(path));
        if (stat(path, &st) != 0) { break; }
        ++g_state.numa.n_nodes;
    }

    // 枚举 CPU
    # 当 NUMA 总 CPU 数小于最大 CPU 数时循环
    while (g_state.numa.total_cpus < GGML_NUMA_MAX_CPUS) {
        # 格式化路径字符串，指定 CPU 编号
        rv = snprintf(path, sizeof(path), "/sys/devices/system/cpu/cpu%u", g_state.numa.total_cpus);
        # 断言路径字符串长度大于 0 且小于 path 数组长度
        GGML_ASSERT(rv > 0 && (unsigned)rv < sizeof(path));
        # 检查路径是否存在
        if (stat(path, &st) != 0) { break; }
        # 增加 NUMA 总 CPU 数
        ++g_state.numa.total_cpus;
    }

    # 打印 NUMA 节点数和 CPU 数
    GGML_PRINT_DEBUG("found %u numa nodes, %u CPUs\n", g_state.numa.n_nodes, g_state.numa.total_cpus);

    # 如果 NUMA 节点数小于 1 或 CPU 数小于 1，则重置节点数并返回
    if (g_state.numa.n_nodes < 1 || g_state.numa.total_cpus < 1) {
        g_state.numa.n_nodes = 0;
        return;
    }

    # 遍历 NUMA 节点
    for (uint32_t n = 0; n < g_state.numa.n_nodes; ++n) {
        # 获取当前节点
        struct ggml_numa_node * node = &g_state.numa.nodes[n];
        # 打印当前节点的 CPU
        GGML_PRINT_DEBUG("CPUs on node %u:", n);
        # 初始化当前节点的 CPU 数为 0
        node->n_cpus = 0;
        # 遍历当前节点的 CPU
        for (uint32_t c = 0; c < g_state.numa.total_cpus; ++c) {
            # 格式化路径字符串，指定节点和 CPU 编号
            rv = snprintf(path, sizeof(path), "/sys/devices/system/node/node%u/cpu%u", n, c);
            # 断言路径字符串长度大于 0 且小于 path 数组长度
            GGML_ASSERT(rv > 0 && (unsigned)rv < sizeof(path));
            # 检查路径是否存在
            if (stat(path, &st) == 0) {
                # 将 CPU 编号添加到当前节点的 CPU 数组中
                node->cpus[node->n_cpus++] = c;
                # 打印当前 CPU 编号
                GGML_PRINT_DEBUG(" %u", c);
            }
        }
        # 打印换行符
        GGML_PRINT_DEBUG("\n");
    }

    # 如果是 NUMA 架构
    if (ggml_is_numa()) {
        # 打开 /proc/sys/kernel/numa_balancing 文件
        FILE *fptr = fopen("/proc/sys/kernel/numa_balancing", "r");
        # 如果文件打开成功
        if (fptr != NULL) {
            # 读取文件内容到缓冲区
            char buf[42];
            if (fgets(buf, sizeof(buf), fptr) && strncmp(buf, "0\n", sizeof(buf)) != 0) {
                # 如果 numa_balancing 启用，打印警告信息
                GGML_PRINT("WARNING: /proc/sys/kernel/numa_balancing is enabled, this has been observed to impair performance\n");
            }
            # 关闭文件
            fclose(fptr);
        }
    }
#else
    // 如果条件不成立，则执行以下代码
#endif
}

// 检查系统是否支持 NUMA 架构
bool ggml_is_numa(void) {
    // 返回系统中 NUMA 节点数量是否大于 1
    return g_state.numa.n_nodes > 1;
}

////////////////////////////////////////////////////////////////////////////////

// 打印 ggml_object 结构体的信息
void ggml_print_object(const struct ggml_object * obj) {
    // 打印 ggml_object 结构体的类型、偏移量、大小和下一个对象的地址
    GGML_PRINT(" - ggml_object: type = %d, offset = %zu, size = %zu, next = %p\n",
            obj->type, obj->offs, obj->size, (const void *) obj->next);
}

// 打印 ggml_context 中的所有对象信息
void ggml_print_objects(const struct ggml_context * ctx) {
    // 获取 ggml_context 中的第一个对象
    struct ggml_object * obj = ctx->objects_begin;

    // 打印函数名称和 ggml_context 的地址
    GGML_PRINT("%s: objects in context %p:\n", __func__, (const void *) ctx);

    // 遍历所有对象并打印信息
    while (obj != NULL) {
        ggml_print_object(obj);
        obj = obj->next;
    }

    // 打印结束标记
    GGML_PRINT("%s: --- end ---\n", __func__);
}

// 计算 ggml_tensor 中元素的总数
GGML_CALL int64_t ggml_nelements(const struct ggml_tensor * tensor) {
    // 确保 GGML_MAX_DIMS 等于 4
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    // 返回元素数量
    return tensor->ne[0]*tensor->ne[1]*tensor->ne[2]*tensor->ne[3];
}

// 计算 ggml_tensor 中行数
GGML_CALL int64_t ggml_nrows(const struct ggml_tensor * tensor) {
    // 确保 GGML_MAX_DIMS 等于 4
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    // 返回行数
    return tensor->ne[1]*tensor->ne[2]*tensor->ne[3];
}

// 计算 ggml_tensor 占用的字节数
GGML_CALL size_t ggml_nbytes(const struct ggml_tensor * tensor) {
    size_t nbytes;
    size_t blck_size = ggml_blck_size(tensor->type);
    if (blck_size == 1) {
        // 计算字节数
        nbytes = ggml_type_size(tensor->type);
        for (int i = 0; i < GGML_MAX_DIMS; ++i) {
            nbytes += (tensor->ne[i] - 1)*tensor->nb[i];
        }
    }
    else {
        // 计算字节数
        nbytes = tensor->ne[0]*tensor->nb[0]/blck_size;
        for (int i = 1; i < GGML_MAX_DIMS; ++i) {
            nbytes += (tensor->ne[i] - 1)*tensor->nb[i];
        }
    }

    return nbytes;
}

// 计算 ggml_tensor 占用的字节数，包括内存对齐
size_t ggml_nbytes_pad(const struct ggml_tensor * tensor) {
    return GGML_PAD(ggml_nbytes(tensor), GGML_MEM_ALIGN);
}

// 获取 ggml_type 对应的块大小
GGML_CALL int ggml_blck_size(enum ggml_type type) {
    return type_traits[type].blck_size;
}

// 获取 ggml_type 对应的类型大小
GGML_CALL size_t ggml_type_size(enum ggml_type type) {
    # 返回给定类型的类型大小
    return type_traits[type].type_size;
// 计算给定类型和元素数量的行大小
GGML_CALL size_t ggml_row_size(enum ggml_type type, int64_t ne) {
    // 断言元素数量是块大小的整数倍
    assert(ne % ggml_blck_size(type) == 0);
    // 返回行大小
    return ggml_type_size(type)*ne/ggml_blck_size(type);
}

// 返回给定类型的大小
double ggml_type_sizef(enum ggml_type type) {
    // 返回类型大小除以块大小的浮点数值
    return ((double)(type_traits[type].type_size))/type_traits[type].blck_size;
}

// 返回给定类型的名称
GGML_CALL const char * ggml_type_name(enum ggml_type type) {
    // 返回类型名称
    return type_traits[type].type_name;
}

// 检查给定类型是否是量化的
GGML_CALL bool ggml_is_quantized(enum ggml_type type) {
    // 返回类型是否量化的布尔值
    return type_traits[type].is_quantized;
}

// 返回给定操作的名称
GGML_CALL const char * ggml_op_name(enum ggml_op op) {
    // 返回操作名称
    return GGML_OP_NAME[op];
}

// 返回给定操作的符号
const char * ggml_op_symbol(enum ggml_op op) {
    // 返回操作符号
    return GGML_OP_SYMBOL[op];
}

// 返回给定一元操作的名称
const char * ggml_unary_op_name(enum ggml_unary_op op) {
    // 返回一元操作名称
    return GGML_UNARY_OP_NAME[op];
}

// 返回给定张量的操作描述
GGML_CALL const char * ggml_op_desc(const struct ggml_tensor * t) {
    // 如果操作是一元操作，则返回一元操作的名称
    if (t->op == GGML_OP_UNARY) {
        enum ggml_unary_op uop = ggml_get_unary_op(t);
        return ggml_unary_op_name(uop);
    }
    // 否则返回操作的名称
    else {
        return ggml_op_name(t->op);
    }
}

// 返回给定张量的元素大小
GGML_CALL size_t ggml_element_size(const struct ggml_tensor * tensor) {
    // 返回张量元素的大小
    return ggml_type_size(tensor->type);
}

// 检查给定张量是否是标量
bool ggml_is_scalar(const struct ggml_tensor * tensor) {
    // 静态断言，确保 GGML_MAX_DIMS 为 4
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    // 检查张量是否是标量
    return tensor->ne[0] == 1 && tensor->ne[1] == 1 && tensor->ne[2] == 1 && tensor->ne[3] == 1;
}

// 检查给定张量是否是向量
bool ggml_is_vector(const struct ggml_tensor * tensor) {
    // 静态断言，确保 GGML_MAX_DIMS 为 4
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    // 检查张量是否是向量
    return tensor->ne[1] == 1 && tensor->ne[2] == 1 && tensor->ne[3] == 1;
}

// 检查给定张量是否是矩阵
bool ggml_is_matrix(const struct ggml_tensor * tensor) {
    // 静态断言，确保 GGML_MAX_DIMS 为 4
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    // 检查张量是否是矩阵
    return tensor->ne[2] == 1 && tensor->ne[3] == 1;
}

// 检查给定张量是否是三维的
bool ggml_is_3d(const struct ggml_tensor * tensor) {
    // 检查张量是否是三维的
    return tensor->ne[3] == 1;
}

// 返回给定张量的维度数量
int ggml_n_dims(const struct ggml_tensor * tensor) {
    # 从最大维度开始向最小维度遍历
    for (int i = GGML_MAX_DIMS - 1; i >= 1; --i) {
        # 如果当前维度的元素数量大于1
        if (tensor->ne[i] > 1) {
            # 返回当前维度加1
            return i + 1;
        }
    }
    # 如果所有维度的元素数量都为1，则返回1
    return 1;
}

// 检查两个张量是否可以进行矩阵乘法
static inline bool ggml_can_mul_mat(const struct ggml_tensor * t0, const struct ggml_tensor * t1) {
    // 确保 GGML_MAX_DIMS 的值为 4，否则更新此函数
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    return (t0->ne[0]           == t1->ne[0])  && // 检查第一个维度是否相等
           (t1->ne[2]%t0->ne[2] == 0)          && // 验证 t0 是否可广播
           (t1->ne[3]%t0->ne[3] == 0);
}

// 检查两个张量是否可以进行外积
static inline bool ggml_can_out_prod(const struct ggml_tensor * t0, const struct ggml_tensor * t1) {
    // 确保 GGML_MAX_DIMS 的值为 4，否则更新此函数
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    return (t0->ne[1] == t1->ne[1])   && // 检查第二个维度是否相等
           (t1->ne[2]%t0->ne[2] == 0) && // 验证 t0 是否可广播
           (t1->ne[3]%t0->ne[3] == 0);
}

// 将 ggml_ftype 转换为 ggml_type
enum ggml_type ggml_ftype_to_ggml_type(enum ggml_ftype ftype) {
    enum ggml_type wtype = GGML_TYPE_COUNT;
    # 根据不同的文件类型（ftype）选择对应的数据类型（wtype）
    switch (ftype) {
        case GGML_FTYPE_ALL_F32:              wtype = GGML_TYPE_F32;   break;  # 如果文件类型为GGML_FTYPE_ALL_F32，则数据类型为GGML_TYPE_F32
        case GGML_FTYPE_MOSTLY_F16:           wtype = GGML_TYPE_F16;   break;  # 如果文件类型为GGML_FTYPE_MOSTLY_F16，则数据类型为GGML_TYPE_F16
        case GGML_FTYPE_MOSTLY_Q4_0:          wtype = GGML_TYPE_Q4_0;  break;  # 如果文件类型为GGML_FTYPE_MOSTLY_Q4_0，则数据类型为GGML_TYPE_Q4_0
        case GGML_FTYPE_MOSTLY_Q4_1:          wtype = GGML_TYPE_Q4_1;  break;  # 如果文件类型为GGML_FTYPE_MOSTLY_Q4_1，则数据类型为GGML_TYPE_Q4_1
        case GGML_FTYPE_MOSTLY_Q5_0:          wtype = GGML_TYPE_Q5_0;  break;  # 如果文件类型为GGML_FTYPE_MOSTLY_Q5_0，则数据类型为GGML_TYPE_Q5_0
        case GGML_FTYPE_MOSTLY_Q5_1:          wtype = GGML_TYPE_Q5_1;  break;  # 如果文件类型为GGML_FTYPE_MOSTLY_Q5_1，则数据类型为GGML_TYPE_Q5_1
        case GGML_FTYPE_MOSTLY_Q8_0:          wtype = GGML_TYPE_Q8_0;  break;  # 如果文件类型为GGML_FTYPE_MOSTLY_Q8_0，则数据类型为GGML_TYPE_Q8_0
        case GGML_FTYPE_MOSTLY_Q2_K:          wtype = GGML_TYPE_Q2_K;  break;  # 如果文件类型为GGML_FTYPE_MOSTLY_Q2_K，则数据类型为GGML_TYPE_Q2_K
        case GGML_FTYPE_MOSTLY_Q3_K:          wtype = GGML_TYPE_Q3_K;  break;  # 如果文件类型为GGML_FTYPE_MOSTLY_Q3_K，则数据类型为GGML_TYPE_Q3_K
        case GGML_FTYPE_MOSTLY_Q4_K:          wtype = GGML_TYPE_Q4_K;  break;  # 如果文件类型为GGML_FTYPE_MOSTLY_Q4_K，则数据类型为GGML_TYPE_Q4_K
        case GGML_FTYPE_MOSTLY_Q5_K:          wtype = GGML_TYPE_Q5_K;  break;  # 如果文件类型为GGML_FTYPE_MOSTLY_Q5_K，则数据类型为GGML_TYPE_Q5_K
        case GGML_FTYPE_MOSTLY_Q6_K:          wtype = GGML_TYPE_Q6_K;  break;  # 如果文件类型为GGML_FTYPE_MOSTLY_Q6_K，则数据类型为GGML_TYPE_Q6_K
        case GGML_FTYPE_MOSTLY_IQ2_XXS:       wtype = GGML_TYPE_IQ2_XXS;  break;  # 如果文件类型为GGML_FTYPE_MOSTLY_IQ2_XXS，则数据类型为GGML_TYPE_IQ2_XXS
        case GGML_FTYPE_MOSTLY_IQ2_XS:        wtype = GGML_TYPE_IQ2_XS;   break;  # 如果文件类型为GGML_FTYPE_MOSTLY_IQ2_XS，则数据类型为GGML_TYPE_IQ2_XS
        case GGML_FTYPE_MOSTLY_IQ3_XXS:       wtype = GGML_TYPE_IQ3_XXS;  break;  # 如果文件类型为GGML_FTYPE_MOSTLY_IQ3_XXS，则数据类型为GGML_TYPE_IQ3_XXS
        case GGML_FTYPE_UNKNOWN:              wtype = GGML_TYPE_COUNT; break;  # 如果文件类型为GGML_FTYPE_UNKNOWN，则数据类型为GGML_TYPE_COUNT
        case GGML_FTYPE_MOSTLY_Q4_1_SOME_F16: wtype = GGML_TYPE_COUNT; break;  # 如果文件类型为GGML_FTYPE_MOSTLY_Q4_1_SOME_F16，则数据类型为GGML_TYPE_COUNT
    }

    # 断言数据类型不为GGML_TYPE_COUNT
    GGML_ASSERT(wtype != GGML_TYPE_COUNT);

    # 返回选择的数据类型
    return wtype;
// 返回 GGML_OBJECT_SIZE 和 GGML_TENSOR_SIZE 的总和，表示 tensor 结构体的内存开销
size_t ggml_tensor_overhead(void) {
    return GGML_OBJECT_SIZE + GGML_TENSOR_SIZE;
}

// 检查给定的 tensor 是否是转置的，通过比较第一个维度和第二个维度的大小关系来判断
GGML_CALL bool ggml_is_transposed(const struct ggml_tensor * tensor) {
    return tensor->nb[0] > tensor->nb[1];
}

// 检查给定的 tensor 是否是连续的，需要满足一定的条件，包括维度大小和元素数量的关系
GGML_CALL bool ggml_is_contiguous(const struct ggml_tensor * tensor) {
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    return
        tensor->nb[0] == ggml_type_size(tensor->type) &&
        tensor->nb[1] == (tensor->nb[0]*tensor->ne[0])/ggml_blck_size(tensor->type) &&
        tensor->nb[2] == tensor->nb[1]*tensor->ne[1] &&
        tensor->nb[3] == tensor->nb[2]*tensor->ne[2];
}

// 检查给定的 tensor 是否在除第一个维度外是连续的
static inline bool ggml_is_contiguous_except_dim_1(const struct ggml_tensor * tensor) {
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    return
        tensor->nb[0] == ggml_type_size(tensor->type) &&
        tensor->nb[2] == tensor->nb[1]*tensor->ne[1] &&
        tensor->nb[3] == tensor->nb[2]*tensor->ne[2];
}

// 检查给定的 tensor 是否是置换的，通过比较相邻维度的大小关系来判断
GGML_CALL bool ggml_is_permuted(const struct ggml_tensor * tensor) {
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    return tensor->nb[0] > tensor->nb[1] || tensor->nb[1] > tensor->nb[2] || tensor->nb[2] > tensor->nb[3];
}

// 检查给定的 tensor 是否是一维的，并且在除第一个维度外是连续的
static inline bool ggml_is_padded_1d(const struct ggml_tensor * tensor) {
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    return
        tensor->nb[0] == ggml_type_size(tensor->type) &&
        tensor->nb[2] == tensor->nb[1]*tensor->ne[1] &&
        tensor->nb[3] == tensor->nb[2]*tensor->ne[2];
}

// 检查两个 tensor 是否具有相同的形状，通过比较各个维度的元素数量来判断
bool ggml_are_same_shape(const struct ggml_tensor * t0, const struct ggml_tensor * t1) {
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    return
        (t0->ne[0] == t1->ne[0] ) &&
        (t0->ne[1] == t1->ne[1] ) &&
        (t0->ne[2] == t1->ne[2] ) &&
        (t0->ne[3] == t1->ne[3] );
}

// 检查 t1 是否可以表示为 t0 的重复
// 检查两个张量是否可以重复，即每个维度的尺寸是否可以整除
static inline bool ggml_can_repeat(const struct ggml_tensor * t0, const struct ggml_tensor * t1) {
    // 静态断言，确保 GGML_MAX_DIMS 的值为 4
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    return
        (t1->ne[0]%t0->ne[0] == 0) &&  // 检查第一个维度是否可以整除
        (t1->ne[1]%t0->ne[1] == 0) &&  // 检查第二个维度是否可以整除
        (t1->ne[2]%t0->ne[2] == 0) &&  // 检查第三个维度是否可以整除
        (t1->ne[3]%t0->ne[3] == 0);    // 检查第四个维度是否可以整除
}

// 检查两个张量的行是否可以重复
static inline bool ggml_can_repeat_rows(const struct ggml_tensor * t0, const struct ggml_tensor * t1) {
    // 静态断言，确保 GGML_MAX_DIMS 的值为 4
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    return (t0->ne[0] == t1->ne[0]) && ggml_can_repeat(t0, t1);
}

// 将整数 n 向上舍入到最接近的 32 的倍数
static inline int ggml_up32(int n) {
    return (n + 31) & ~31;
}

// 将整数 n 向上舍入到最接近的 m 的倍数
static inline int ggml_up(int n, int m) {
    // 断言 m 是 2 的幂
    GGML_ASSERT((m & (m - 1)) == 0);
    return (n + m - 1) & ~(m - 1);
}

// 断言指针对齐到 GGML_MEM_ALIGN
#define ggml_assert_aligned(ptr) \
    GGML_ASSERT(((uintptr_t) (ptr))%GGML_MEM_ALIGN == 0)

////////////////////////////////////////////////////////////////////////////////

// 初始化 GGML 上下文
struct ggml_context * ggml_init(struct ggml_init_params params) {
    // 使此函数线程安全
    ggml_critical_section_start();

    static bool is_first_call = true;
    // 如果是第一次调用
    if (is_first_call) {
        // 初始化时间系统（在 Windows 上是必需的）
        ggml_time_init();

        // 初始化 GELU、Quick GELU、SILU 和 EXP F32 表
        {
            // 记录开始时间
            const uint64_t t_start = ggml_time_us(); UNUSED(t_start);

            // 循环遍历 0 到 2^16 之间的整数
            ggml_fp16_t ii;
            for (int i = 0; i < (1 << 16); ++i) {
                // 将整数转换为 16 位无符号整数
                uint16_t ui = i;
                // 将 16 位无符号整数拷贝到 ggml_fp16_t 类型中
                memcpy(&ii, &ui, sizeof(ii));
                // 计算并存储对应的浮点数值到表中
                const float f = ggml_table_f32_f16[i] = GGML_COMPUTE_FP16_TO_FP32(ii);
                // 计算 GELU 函数值并存储到表中
                ggml_table_gelu_f16[i] = GGML_FP32_TO_FP16(ggml_gelu_f32(f));
                // 计算 Quick GELU 函数值并存储到表中
                ggml_table_gelu_quick_f16[i] = GGML_FP32_TO_FP16(ggml_gelu_quick_f32(f));
                // 计算 SILU 函数值并存储到表中
                ggml_table_silu_f16[i] = GGML_FP32_TO_FP16(ggml_silu_f32(f));
                // 计算 EXP 函数值并存储到表中
                ggml_table_exp_f16[i]  = GGML_FP32_TO_FP16(expf(f));
            }

            // 记录结束时间
            const uint64_t t_end = ggml_time_us(); UNUSED(t_end);

            // 打印初始化时间和表的信息
            GGML_PRINT_DEBUG("%s: GELU, Quick GELU, SILU and EXP tables initialized in %f ms\n", __func__, (t_end - t_start)/1000.0f);
        }

        // 初始化 g_state
        {
            // 记录开始时间
            const uint64_t t_start = ggml_time_us(); UNUSED(t_start);

            // 初始化 g_state 结构体
            g_state = (struct ggml_state) {
                /*.contexts =*/ { { 0 } }, // 初始化 contexts 字段
                /*.numa =*/ {
                    .n_nodes = 0, // 初始化 n_nodes 字段
                    .total_cpus = 0, // 初始化 total_cpus 字段
                },
            };

            // 循环遍历所有的 contexts，将 used 字段初始化为 false
            for (int i = 0; i < GGML_MAX_CONTEXTS; ++i) {
                g_state.contexts[i].used = false;
            }

            // 记录结束时间
            const uint64_t t_end = ggml_time_us(); UNUSED(t_end);

            // 打印初始化时间和 g_state 的信息
            GGML_PRINT_DEBUG("%s: g_state initialized in %f ms\n", __func__, (t_end - t_start)/1000.0f);
        }
    }
// 如果定义了 GGML_USE_CUBLAS，则初始化 CUBLAS
        ggml_init_cublas();
// 如果定义了 GGML_USE_CLBLAST，则初始化 CLBLAST
        ggml_cl_init();
// 如果定义了 GGML_USE_VULKAN，则初始化 VULKAN
        ggml_vk_init();
// 如果定义了 GGML_USE_SYCL，则初始化 SYCL
        ggml_init_sycl();

        // 设置操作是否有任务传递
        ggml_setup_op_has_task_pass();

        // 标记不是第一次调用
        is_first_call = false;
    }

    // 在 g_state 中找到未使用的上下文
    struct ggml_context * ctx = NULL;

    for (int i = 0; i < GGML_MAX_CONTEXTS; i++) {
        if (!g_state.contexts[i].used) {
            g_state.contexts[i].used = true;
            ctx = &g_state.contexts[i].context;

            GGML_PRINT_DEBUG("%s: found unused context %d\n", __func__, i);
            break;
        }
    }

    if (ctx == NULL) {
        GGML_PRINT_DEBUG("%s: no unused context found\n", __func__);

        ggml_critical_section_end();

        return NULL;
    }

    // 允许使用 0 大小调用 ggml_init
    if (params.mem_size == 0) {
        params.mem_size = GGML_MEM_ALIGN;
    }

    const size_t mem_size = params.mem_buffer ? params.mem_size : GGML_PAD(params.mem_size, GGML_MEM_ALIGN);

    *ctx = (struct ggml_context) {
        /*.mem_size           =*/ mem_size,
        /*.mem_buffer         =*/ params.mem_buffer ? params.mem_buffer : GGML_ALIGNED_MALLOC(mem_size),
        /*.mem_buffer_owned   =*/ params.mem_buffer ? false : true,
        /*.no_alloc           =*/ params.no_alloc,
        /*.no_alloc_save      =*/ params.no_alloc,
        /*.n_objects          =*/ 0,
        /*.objects_begin      =*/ NULL,
        /*.objects_end        =*/ NULL,
        /*.scratch            =*/ { 0, 0, NULL, },
        /*.scratch_save       =*/ { 0, 0, NULL, },
    };

    GGML_ASSERT(ctx->mem_buffer != NULL);

    ggml_assert_aligned(ctx->mem_buffer);

    GGML_PRINT_DEBUG("%s: context initialized\n", __func__);

    ggml_critical_section_end();

    return ctx;
}

// 释放 ggml_context 结构
void ggml_free(struct ggml_context * ctx) {
    if (ctx == NULL) {
        return;
    }

    // 使此函数线程安全
    # 进入临界区，开始对关键部分进行保护
    ggml_critical_section_start();

    # 初始化一个布尔变量 found，用于标记是否找到了指定的上下文
    bool found = false;

    # 遍历所有上下文，查找与给定上下文相同的上下文
    for (int i = 0; i < GGML_MAX_CONTEXTS; i++) {
        # 如果找到了与给定上下文相同的上下文
        if (&g_state.contexts[i].context == ctx) {
            # 将找到的上下文标记为未使用
            g_state.contexts[i].used = false;

            # 打印调试信息，指示已释放上下文并显示内存使用情况
            GGML_PRINT_DEBUG("%s: context %d has been freed. memory used = %zu\n",
                    __func__, i, ggml_used_mem(ctx));

            # 如果上下文拥有内存缓冲区，则释放该内存
            if (ctx->mem_buffer_owned) {
                GGML_ALIGNED_FREE(ctx->mem_buffer);
            }

            # 标记为找到了指定上下文，并跳出循环
            found = true;
            break;
        }
    }

    # 如果未找到指定上下文，则打印调试信息
    if (!found) {
        GGML_PRINT_DEBUG("%s: context not found\n", __func__);
    }

    # 退出临界区，结束对关键部分的保护
    ggml_critical_section_end();
// 返回已使用的内存大小，如果对象结束指针为空，则返回0，否则返回对象结束指针的偏移量加上对象结束指针的大小
size_t ggml_used_mem(const struct ggml_context * ctx) {
    return ctx->objects_end == NULL ? 0 : ctx->objects_end->offs + ctx->objects_end->size;
}

// 设置临时缓冲区，返回之前的临时缓冲区的偏移量，如果之前的临时缓冲区为空则返回0
size_t ggml_set_scratch(struct ggml_context * ctx, struct ggml_scratch scratch) {
    const size_t result = ctx->scratch.data ? ctx->scratch.offs : 0;

    // 设置新的临时缓冲区
    ctx->scratch = scratch;

    return result;
}

// 获取是否禁止分配内存的标志
bool ggml_get_no_alloc(struct ggml_context * ctx) {
    return ctx->no_alloc;
}

// 设置是否禁止分配内存的标志
void ggml_set_no_alloc(struct ggml_context * ctx, bool no_alloc) {
    ctx->no_alloc = no_alloc;
}

// 获取内存缓冲区
void * ggml_get_mem_buffer(const struct ggml_context * ctx) {
    return ctx->mem_buffer;
}

// 获取内存大小
size_t ggml_get_mem_size(const struct ggml_context * ctx) {
    return ctx->mem_size;
}

// 获取最大张量大小
size_t ggml_get_max_tensor_size(const struct ggml_context * ctx) {
    size_t max_size = 0;

    // 遍历所有张量，找到最大的张量大小
    for (struct ggml_tensor * tensor = ggml_get_first_tensor(ctx); tensor != NULL; tensor = ggml_get_next_tensor(ctx, tensor)) {
        max_size = MAX(max_size, ggml_nbytes(tensor));
    }

    return max_size;
}

// 保存临时缓冲区
static void ggml_scratch_save(struct ggml_context * ctx) {
    // 保存禁止分配内存的标志和临时缓冲区
    ctx->no_alloc_save = ctx->no_alloc;
    ctx->no_alloc      = false;

    ctx->scratch_save = ctx->scratch;
    ctx->scratch.data = NULL;
}

// 加载临时缓冲区
static void ggml_scratch_load(struct ggml_context * ctx) {
    // 恢复禁止分配内存的标志和临时缓冲区
    ctx->no_alloc = ctx->no_alloc_save;

    ctx->scratch = ctx->scratch_save;
}

////////////////////////////////////////////////////////////////////////////////

// 创建新对象，始终将对象插入到上下文内存池的末尾
static struct ggml_object * ggml_new_object(struct ggml_context * ctx, enum ggml_object_type type, size_t size) {
    // always insert objects at the end of the context's memory pool
    // 获取当前对象指针，指向上一个对象的末尾
    struct ggml_object * obj_cur = ctx->objects_end;

    // 计算当前对象的偏移量
    const size_t cur_offs = obj_cur == NULL ? 0 : obj_cur->offs;
    // 计算当前对象的大小
    const size_t cur_size = obj_cur == NULL ? 0 : obj_cur->size;
    // 计算当前对象的结束位置
    const size_t cur_end  = cur_offs + cur_size;

    // 将需要的内存大小对齐到 GGML_MEM_ALIGN
    size_t size_needed = GGML_PAD(size, GGML_MEM_ALIGN);

    // 获取上下文中的内存缓冲区
    char * const mem_buffer = ctx->mem_buffer;
    // 计算新对象的指针位置
    struct ggml_object * const obj_new = (struct ggml_object *)(mem_buffer + cur_end);

    // 检查内存池是否有足够的空间存放新对象
    if (cur_end + size_needed + GGML_OBJECT_SIZE > ctx->mem_size) {
        // 打印错误信息并断言
        GGML_PRINT("%s: not enough space in the context's memory pool (needed %zu, available %zu)\n",
                __func__, cur_end + size_needed, ctx->mem_size);
        assert(false);
        return NULL;
    }

    // 初始化新对象的属性
    *obj_new = (struct ggml_object) {
        .offs = cur_end + GGML_OBJECT_SIZE,
        .size = size_needed,
        .next = NULL,
        .type = type,
    };

    // 确保新对象的偏移量对齐
    ggml_assert_aligned(mem_buffer + obj_new->offs);

    // 将新对象链接到对象链表中
    if (obj_cur != NULL) {
        obj_cur->next = obj_new;
    } else {
        // 如果是上下文中的第一个对象
        ctx->objects_begin = obj_new;
    }

    // 更新上下文中的对象指针
    ctx->objects_end = obj_new;

    // 返回新对象指针
    return obj_new;
}



// 创建新的张量对象
static struct ggml_tensor * ggml_new_tensor_impl(
        struct ggml_context * ctx,
        enum   ggml_type      type,
        int                   n_dims,
        const int64_t       * ne,
        struct ggml_tensor  * view_src,
        size_t                view_offs) {

    // 确保张量维度在有效范围内
    assert(n_dims >= 1 && n_dims <= GGML_MAX_DIMS);

    // 查找基本张量和绝对偏移量
    if (view_src != NULL && view_src->view_src != NULL) {
        view_offs += view_src->view_offs;
        view_src   = view_src->view_src;
    }

    // 计算数据大小
    size_t data_size = ggml_row_size(type, ne[0]);
    for (int i = 1; i < n_dims; i++) {
        data_size *= ne[i];
    }

    // 断言视图源为空或数据大小加偏移量不超过视图源的字节数
    GGML_ASSERT(view_src == NULL || data_size + view_offs <= ggml_nbytes(view_src));

    // 初始化数据指针
    void * data = view_src != NULL ? view_src->data : NULL;
    if (data != NULL) {
        data = (char *) data + view_offs;
    }

    size_t obj_alloc_size = 0;

    // 如果视图源为空且不禁止分配内存
    if (view_src == NULL && !ctx->no_alloc) {
        if (ctx->scratch.data != NULL) {
            // 在临时缓冲区中分配张量数据
            if (ctx->scratch.offs + data_size > ctx->scratch.size) {
                GGML_PRINT("%s: not enough space in the scratch memory pool (needed %zu, available %zu)\n",
                        __func__, ctx->scratch.offs + data_size, ctx->scratch.size);
                assert(false);
                return NULL;
            }

            data = (char * const) ctx->scratch.data + ctx->scratch.offs;

            ctx->scratch.offs += data_size;
        } else {
            // 在上下文的内存池中分配张量数据
            obj_alloc_size = data_size;
        }
    }

    // 创建新的对象
    struct ggml_object * const obj_new = ggml_new_object(ctx, GGML_OBJECT_TENSOR, GGML_TENSOR_SIZE + obj_alloc_size);

    // TODO: 对于可恢复的错误，我们需要在这里释放从临时缓冲区分配的数据

    // 返回新的张量对象
    struct ggml_tensor * const result = (struct ggml_tensor *)((char *)ctx->mem_buffer + obj_new->offs);
    // 初始化一个 ggml_tensor 结构体，并设置各个字段的初始值
    *result = (struct ggml_tensor) {
        /*.type         =*/ type, // 设置类型
        /*.backend      =*/ GGML_BACKEND_CPU, // 设置后端为 CPU
        /*.buffer       =*/ NULL, // 设置缓冲区为空
        /*.ne           =*/ { 1, 1, 1, 1 }, // 设置维度大小为 1
        /*.nb           =*/ { 0, 0, 0, 0 }, // 设置维度偏移为 0
        /*.op           =*/ GGML_OP_NONE, // 设置操作为无
        /*.op_params    =*/ { 0 }, // 设置操作参数为 0
        /*.is_param     =*/ false, // 设置是否为参数为 false
        /*.grad         =*/ NULL, // 设置梯度为空
        /*.src          =*/ { NULL }, // 设置源为空
        /*.perf_runs    =*/ 0, // 设置性能运行次数为 0
        /*.perf_cycles  =*/ 0, // 设置性能周期为 0
        /*.perf_time_us =*/ 0, // 设置性能时间为 0
        /*.view_src     =*/ view_src, // 设置视图源
        /*.view_offs    =*/ view_offs, // 设置视图偏移
        /*.data         =*/ obj_alloc_size > 0 ? (void *)(result + 1) : data, // 根据条件设置数据
        /*.name         =*/ { 0 }, // 设置名称为 0
        /*.extra        =*/ NULL, // 设置额外信息为空
        /*.padding      =*/ { 0 }, // 设置填充为 0
    };

    // TODO: this should not be needed as long as we don't rely on aligned SIMD loads
    //ggml_assert_aligned(result->data);

    // 遍历维度，设置每个维度的大小
    for (int i = 0; i < n_dims; i++) {
        result->ne[i] = ne[i];
    }

    // 计算每个维度的字节数
    result->nb[0] = ggml_type_size(type);
    result->nb[1] = result->nb[0]*(result->ne[0]/ggml_blck_size(type));
    for (int i = 2; i < GGML_MAX_DIMS; i++) {
        result->nb[i] = result->nb[i - 1]*result->ne[i - 1];
    }

    // 增加上下文中对象的数量
    ctx->n_objects++;

    // 返回初始化后的 ggml_tensor 结构体
    return result;
// 创建一个新的张量结构体，根据给定的上下文、类型、维度数和维度数组
struct ggml_tensor * ggml_new_tensor(
        struct ggml_context * ctx,
        enum   ggml_type      type,
        int                   n_dims,
        const int64_t       * ne) {
    // 调用内部实现函数创建新的张量结构体
    return ggml_new_tensor_impl(ctx, type, n_dims, ne, NULL, 0);
}

// 创建一个新的一维张量结构体，根据给定的上下文、类型和第一个维度大小
struct ggml_tensor * ggml_new_tensor_1d(
        struct ggml_context * ctx,
        enum   ggml_type      type,
        int64_t ne0) {
    // 调用 ggml_new_tensor 函数创建新的一维张量结构体
    return ggml_new_tensor(ctx, type, 1, &ne0);
}

// 创建一个新的二维张量结构体，根据给定的上下文、类型和两个维度大小
struct ggml_tensor * ggml_new_tensor_2d(
        struct ggml_context * ctx,
        enum   ggml_type      type,
        int64_t ne0,
        int64_t ne1) {
    // 创建包含两个维度大小的数组
    const int64_t ne[2] = { ne0, ne1 };
    // 调用 ggml_new_tensor 函数创建新的二维张量结构体
    return ggml_new_tensor(ctx, type, 2, ne);
}

// 创建一个新的三维张量结构体，根据给定的上下文、类型和三个维度大小
struct ggml_tensor * ggml_new_tensor_3d(
        struct ggml_context * ctx,
        enum   ggml_type      type,
        int64_t ne0,
        int64_t ne1,
        int64_t ne2) {
    // 创建包含三个维度大小的数组
    const int64_t ne[3] = { ne0, ne1, ne2 };
    // 调用 ggml_new_tensor 函数创建新的三维张量结构体
    return ggml_new_tensor(ctx, type, 3, ne);
}

// 创建一个新的四维张量结构体，根据给定的上下文、类型和四个维度大小
struct ggml_tensor * ggml_new_tensor_4d(
        struct ggml_context * ctx,
        enum   ggml_type type,
        int64_t ne0,
        int64_t ne1,
        int64_t ne2,
        int64_t ne3) {
    // 创建包含四个维度大小的数组
    const int64_t ne[4] = { ne0, ne1, ne2, ne3 };
    // 调用 ggml_new_tensor 函数创建新的四维张量结构体
    return ggml_new_tensor(ctx, type, 4, ne);
}

// 创建一个新的包含整型数据的张量结构体，根据给定的上下文和整型值
struct ggml_tensor * ggml_new_i32(struct ggml_context * ctx, int32_t value) {
    // 保存上下文的状态
    ggml_scratch_save(ctx);
    // 创建一个新的一维整型张量结构体
    struct ggml_tensor * result = ggml_new_tensor_1d(ctx, GGML_TYPE_I32, 1);
    // 恢复上下文的状态
    ggml_scratch_load(ctx);
    // 设置张量结构体的整型值
    ggml_set_i32(result, value);
    // 返回结果张量结构体
    return result;
}

// 创建一个新的包含浮点型数据的张量结构体，根据给定的上下文和浮点值
struct ggml_tensor * ggml_new_f32(struct ggml_context * ctx, float value) {
    // 保存上下文的状态
    ggml_scratch_save(ctx);
    // 创建一个新的一维浮点型张量结构体
    struct ggml_tensor * result = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 1);
    // 恢复上下文的状态
    ggml_scratch_load(ctx);
    // 设置张量结构体的浮点值
    ggml_set_f32(result, value);
    // 返回结果张量结构体
    return result;
}

// 复制给定张量结构体的数据，创建一个新的张量结构体
struct ggml_tensor * ggml_dup_tensor(struct ggml_context * ctx, const struct ggml_tensor * src) {
    // 调用 ggml_new_tensor 函数创建新的张量结构体，复制给定张量结构体的数据
    return ggml_new_tensor(ctx, src->type, GGML_MAX_DIMS, src->ne);
}
// 设置张量的操作参数，包括参数指针和参数大小
static void ggml_set_op_params(struct ggml_tensor * tensor, const void * params, size_t params_size) {
    GGML_ASSERT(tensor != NULL); // 消除 -Warray-bounds 警告
    assert(params_size <= GGML_MAX_OP_PARAMS);
    // 将参数拷贝到张量的操作参数中
    memcpy(tensor->op_params, params, params_size);
}

// 获取张量操作参数中的第 i 个 int32_t 值
static int32_t ggml_get_op_params_i32(const struct ggml_tensor * tensor, uint32_t i) {
    assert(i < GGML_MAX_OP_PARAMS / sizeof(int32_t));
    return ((const int32_t *)(tensor->op_params))[i];
}

// 设置张量操作参数中的第 i 个 int32_t 值
static void ggml_set_op_params_i32(struct ggml_tensor * tensor, uint32_t i, int32_t value) {
    assert(i < GGML_MAX_OP_PARAMS / sizeof(int32_t));
    ((int32_t *)(tensor->op_params))[i] = value;
}

// 将张量数据中的所有元素设置为零
struct ggml_tensor * ggml_set_zero(struct ggml_tensor * tensor) {
    memset(tensor->data, 0, ggml_nbytes(tensor));
    return tensor;
}

// 将张量数据中的所有元素设置为指定的 int32_t 值
struct ggml_tensor * ggml_set_i32 (struct ggml_tensor * tensor, int32_t value) {
    const int n     = ggml_nrows(tensor);
    const int nc    = tensor->ne[0];
    const size_t n1 = tensor->nb[1];

    char * const data = tensor->data;
    # 根据张量的数据类型进行不同的操作
    switch (tensor->type) {
        # 当数据类型为 int8 时
        case GGML_TYPE_I8:
            {
                # 确保数据类型大小符合预期
                assert(tensor->nb[0] == sizeof(int8_t));
                # 遍历数据并设置 int8 值
                for (int i = 0; i < n; i++) {
                    ggml_vec_set_i8(nc, (int8_t *)(data + i*n1), value);
                }
            } break;
        # 当数据类型为 int16 时
        case GGML_TYPE_I16:
            {
                # 确保数据类型大小符合预期
                assert(tensor->nb[0] == sizeof(int16_t));
                # 遍历数据并设置 int16 值
                for (int i = 0; i < n; i++) {
                    ggml_vec_set_i16(nc, (int16_t *)(data + i*n1), value);
                }
            } break;
        # 当数据类型为 int32 时
        case GGML_TYPE_I32:
            {
                # 确保数据类型大小符合预期
                assert(tensor->nb[0] == sizeof(int32_t));
                # 遍历数据并设置 int32 值
                for (int i = 0; i < n; i++) {
                    ggml_vec_set_i32(nc, (int32_t *)(data + i*n1), value);
                }
            } break;
        # 当数据类型为 f16 时
        case GGML_TYPE_F16:
            {
                # 确保数据类型大小符合预期
                assert(tensor->nb[0] == sizeof(ggml_fp16_t));
                # 遍历数据并设置 f16 值
                for (int i = 0; i < n; i++) {
                    ggml_vec_set_f16(nc, (ggml_fp16_t *)(data + i*n1), GGML_FP32_TO_FP16(value));
                }
            } break;
        # 当数据类型为 f32 时
        case GGML_TYPE_F32:
            {
                # 确保数据类型大小符合预期
                assert(tensor->nb[0] == sizeof(float));
                # 遍历数据并设置 float 值
                for (int i = 0; i < n; i++) {
                    ggml_vec_set_f32(nc, (float *)(data + i*n1), value);
                }
            } break;
        # 默认情况下
        default:
            {
                # 抛出断言错误
                GGML_ASSERT(false);
            } break;
    }

    # 返回处理后的张量
    return tensor;
    // 设置浮点数值到张量中
    struct ggml_tensor * ggml_set_f32(struct ggml_tensor * tensor, float value) {
        // 获取张量的行数
        const int n     = ggml_nrows(tensor);
        // 获取张量的第一个维度大小
        const int nc    = tensor->ne[0];
        // 获取张量的第二个维度大小
        const size_t n1 = tensor->nb[1];

        // 获取张量数据的指针
        char * const data = tensor->data;

        // 根据张量的类型进行不同的处理
        switch (tensor->type) {
            // 如果张量类型为 int8
            case GGML_TYPE_I8:
                {
                    // 断言张量的第一个维度大小为 int8_t 类型的大小
                    assert(tensor->nb[0] == sizeof(int8_t));
                    // 遍历张量的行数，设置 int8_t 类型的值
                    for (int i = 0; i < n; i++) {
                        ggml_vec_set_i8(nc, (int8_t *)(data + i*n1), value);
                    }
                } break;
            // 如果张量类型为 int16
            case GGML_TYPE_I16:
                {
                    // 断言张量的第一个维度大小为 int16_t 类型的大小
                    assert(tensor->nb[0] == sizeof(int16_t));
                    // 遍历张量的行数，设置 int16_t 类型的值
                    for (int i = 0; i < n; i++) {
                        ggml_vec_set_i16(nc, (int16_t *)(data + i*n1), value);
                    }
                } break;
            // 如果张量类型为 int32
            case GGML_TYPE_I32:
                {
                    // 断言张量的第一个维度大小为 int32_t 类型的大小
                    assert(tensor->nb[0] == sizeof(int32_t));
                    // 遍历张量的行数，设置 int32_t 类型的值
                    for (int i = 0; i < n; i++) {
                        ggml_vec_set_i32(nc, (int32_t *)(data + i*n1), value);
                    }
                } break;
            // 如果张量类型为 f16
            case GGML_TYPE_F16:
                {
                    // 断言张量的第一个维度大小为 ggml_fp16_t 类型的大小
                    assert(tensor->nb[0] == sizeof(ggml_fp16_t));
                    // 遍历张量的行数，设置 ggml_fp16_t 类型的值
                    for (int i = 0; i < n; i++) {
                        ggml_vec_set_f16(nc, (ggml_fp16_t *)(data + i*n1), GGML_FP32_TO_FP16(value));
                    }
                } break;
            // 如果张量类型为 f32
            case GGML_TYPE_F32:
                {
                    // 断言张量的第一个维度大小为 float 类型的大小
                    assert(tensor->nb[0] == sizeof(float));
                    // 遍历张量的行数，设置 float 类型的值
                    for (int i = 0; i < n; i++) {
                        ggml_vec_set_f32(nc, (float *)(data + i*n1), value);
                    }
                } break;
            // 默认情况
            default:
                {
                    // 断言失败
                    GGML_ASSERT(false);
                } break;
        }

        // 返回张量
        return tensor;
    }

    // 根据索引值解开张量的索引
    void ggml_unravel_index(const struct ggml_tensor * tensor, int64_t i, int64_t * i0, int64_t * i1, int64_t * i2, int64_t * i3) {
        // 获取张量的第三个维度大小
        const int64_t ne2 = tensor->ne[2];
        // 获取张量的第二个维度大小
        const int64_t ne1 = tensor->ne[1];
        // 获取张量的第一个维度大小
        const int64_t ne0 = tensor->ne[0];

        // 计算第四个维度的索引
        const int64_t i3_ = (i/(ne2*ne1*ne0));
    // 计算在四维数组中的索引 i 对应的三维坐标 (i3, i2, i1, i0)
    const int64_t i2_ = (i - i3_*ne2*ne1*ne0)/(ne1*ne0);
    const int64_t i1_ = (i - i3_*ne2*ne1*ne0 - i2_*ne1*ne0)/ne0;
    const int64_t i0_ = (i - i3_*ne2*ne1*ne0 - i2_*ne1*ne0 - i1_*ne0);

    // 如果传入的指针 i0 不为空，则将计算得到的 i0_ 赋值给 i0
    if (i0) {
        *i0 = i0_;
    }
    // 如果传入的指针 i1 不为空，则将计算得到的 i1_ 赋值给 i1
    if (i1) {
        *i1 = i1_;
    }
    // 如果传入的指针 i2 不为空，则将计算得到的 i2_ 赋值给 i2
    if (i2) {
        *i2 = i2_;
    }
    // 如果传入的指针 i3 不为空，则将计算得到的 i3_ 赋值给 i3
    if (i3) {
        *i3 = i3_;
    }
}



// 获取一维整型数组中指定索引位置的元素值
int32_t ggml_get_i32_1d(const struct ggml_tensor * tensor, int i) {
    // 如果张量不是连续的，则需要根据索引值获取多维张量的索引，并调用相应的函数获取元素值
    if (!ggml_is_contiguous(tensor)) {
        int64_t id[4] = { 0, 0, 0, 0 };
        ggml_unravel_index(tensor, i, &id[0], &id[1], &id[2], &id[3]);
        return ggml_get_i32_nd(tensor, id[0], id[1], id[2], id[3]);
    }
    // 根据张量的数据类型不同，获取对应索引位置的元素值
    switch (tensor->type) {
        case GGML_TYPE_I8:
            {
                GGML_ASSERT(tensor->nb[0] == sizeof(int8_t));
                return ((int8_t *)(tensor->data))[i];
            }
        case GGML_TYPE_I16:
            {
                GGML_ASSERT(tensor->nb[0] == sizeof(int16_t));
                return ((int16_t *)(tensor->data))[i];
            }
        case GGML_TYPE_I32:
            {
                GGML_ASSERT(tensor->nb[0] == sizeof(int32_t));
                return ((int32_t *)(tensor->data))[i];
            }
        case GGML_TYPE_F16:
            {
                GGML_ASSERT(tensor->nb[0] == sizeof(ggml_fp16_t));
                return GGML_FP16_TO_FP32(((ggml_fp16_t *)(tensor->data))[i]);
            }
        case GGML_TYPE_F32:
            {
                GGML_ASSERT(tensor->nb[0] == sizeof(float));
                return ((float *)(tensor->data))[i];
            }
        default:
            {
                GGML_ASSERT(false);
            }
    }

    return 0.0f;
}

// 设置一维整型数组中指定索引位置的元素值
void ggml_set_i32_1d(const struct ggml_tensor * tensor, int i, int32_t value) {
    // 如果张量不是连续的，则需要根据索引值获取多维张量的索引，并调用相应的函数设置元素值
    if (!ggml_is_contiguous(tensor)) {
        int64_t id[4] = { 0, 0, 0, 0 };
        ggml_unravel_index(tensor, i, &id[0], &id[1], &id[2], &id[3]);
        ggml_set_i32_nd(tensor, id[0], id[1], id[2], id[3], value);
        return;
    }
    # 根据张量的数据类型进行不同的操作
    switch (tensor->type) {
        # 如果数据类型为 int8
        case GGML_TYPE_I8:
            {
                # 确保张量的数据大小为 int8_t 类型的大小
                GGML_ASSERT(tensor->nb[0] == sizeof(int8_t));
                # 将值赋给 int8_t 类型的数据数组中的第 i 个元素
                ((int8_t *)(tensor->data))[i] = value;
            } break;
        # 如果数据类型为 int16
        case GGML_TYPE_I16:
            {
                # 确保张量的数据大小为 int16_t 类型的大小
                GGML_ASSERT(tensor->nb[0] == sizeof(int16_t));
                # 将值赋给 int16_t 类型的数据数组中的第 i 个元素
                ((int16_t *)(tensor->data))[i] = value;
            } break;
        # 如果数据类型为 int32
        case GGML_TYPE_I32:
            {
                # 确保张量的数据大小为 int32_t 类型的大小
                GGML_ASSERT(tensor->nb[0] == sizeof(int32_t));
                # 将值赋给 int32_t 类型的数据数组中的第 i 个元素
                ((int32_t *)(tensor->data))[i] = value;
            } break;
        # 如果数据类型为 f16
        case GGML_TYPE_F16:
            {
                # 确保张量的数据大小为 ggml_fp16_t 类型的大小
                GGML_ASSERT(tensor->nb[0] == sizeof(ggml_fp16_t));
                # 将值转换为 ggml_fp16_t 类型后赋给数据数组中的第 i 个元素
                ((ggml_fp16_t *)(tensor->data))[i] = GGML_FP32_TO_FP16(value);
            } break;
        # 如果数据类型为 f32
        case GGML_TYPE_F32:
            {
                # 确保张量的数据大小为 float 类型的大小
                GGML_ASSERT(tensor->nb[0] == sizeof(float));
                # 将值赋给 float 类型的数据数组中的第 i 个元素
                ((float *)(tensor->data))[i] = value;
            } break;
        # 默认情况下
        default:
            {
                # 断言失败
                GGML_ASSERT(false);
            } break;
    }
}

// 获取四维张量中指定位置的 int32_t 类型数据
int32_t ggml_get_i32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3) {
    // 计算数据在张量中的偏移量
    void * data   = (char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1] + i2*tensor->nb[2] + i3*tensor->nb[3];
    // 根据张量类型进行数据类型转换并返回数据
    switch (tensor->type) {
        case GGML_TYPE_I8:
            return ((int8_t *) data)[0];
        case GGML_TYPE_I16:
            return ((int16_t *) data)[0];
        case GGML_TYPE_I32:
            return ((int32_t *) data)[0];
        case GGML_TYPE_F16:
            return GGML_FP16_TO_FP32(((ggml_fp16_t *) data)[0]);
        case GGML_TYPE_F32:
            return ((float *) data)[0];
        default:
            GGML_ASSERT(false);
    }

    return 0.0f;
}

// 设置四维张量中指定位置的 int32_t 类型数据
void ggml_set_i32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3, int32_t value) {
    // 计算数据在张量中的偏移量
    void * data   = (char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1] + i2*tensor->nb[2] + i3*tensor->nb[3];
    // 根据张量类型进行数据类型转换并设置数据
    switch (tensor->type) {
        case GGML_TYPE_I8:
            {
                ((int8_t *)(data))[0] = value;
            } break;
        case GGML_TYPE_I16:
            {
                ((int16_t *)(data))[0] = value;
            } break;
        case GGML_TYPE_I32:
            {
                ((int32_t *)(data))[0] = value;
            } break;
        case GGML_TYPE_F16:
            {
                ((ggml_fp16_t *)(data))[0] = GGML_FP32_TO_FP16(value);
            } break;
        case GGML_TYPE_F32:
            {
                ((float *)(data))[0] = value;
            } break;
        default:
            {
                GGML_ASSERT(false);
            } break;
    }
}

// 获取一维张量中指定位置的 float 类型数据
float ggml_get_f32_1d(const struct ggml_tensor * tensor, int i) {
    // 如果张量不是连续的，则根据索引计算多维索引并获取数据
    if (!ggml_is_contiguous(tensor)) {
        int64_t id[4] = { 0, 0, 0, 0 };
        ggml_unravel_index(tensor, i, &id[0], &id[1], &id[2], &id[3]);
        return ggml_get_f32_nd(tensor, id[0], id[1], id[2], id[3]);
    }
    # 根据张量的数据类型进行不同的处理
    switch (tensor->type) {
        # 如果数据类型为 int8
        case GGML_TYPE_I8:
            {
                # 断言张量的第一个维度大小为 int8_t 类型的大小
                GGML_ASSERT(tensor->nb[0] == sizeof(int8_t));
                # 返回第 i 个元素的 int8_t 类型数据
                return ((int8_t *)(tensor->data))[i];
            }
        # 如果数据类型为 int16
        case GGML_TYPE_I16:
            {
                # 断言张量的第一个维度大小为 int16_t 类型的大小
                GGML_ASSERT(tensor->nb[0] == sizeof(int16_t));
                # 返回第 i 个元素的 int16_t 类型数据
                return ((int16_t *)(tensor->data))[i];
            }
        # 如果数据类型为 int32
        case GGML_TYPE_I32:
            {
                # 断言张量的第一个维度大小为 int32_t 类型的大小
                GGML_ASSERT(tensor->nb[0] == sizeof(int32_t));
                # 返回第 i 个元素的 int32_t 类型数据
                return ((int32_t *)(tensor->data))[i];
            }
        # 如果数据类型为 f16
        case GGML_TYPE_F16:
            {
                # 断言张量的第一个维度大小为 ggml_fp16_t 类型的大小
                GGML_ASSERT(tensor->nb[0] == sizeof(ggml_fp16_t));
                # 返回第 i 个元素的 ggml_fp16_t 类型数据转换为 float 类型数据
                return GGML_FP16_TO_FP32(((ggml_fp16_t *)(tensor->data))[i]);
            }
        # 如果数据类型为 f32
        case GGML_TYPE_F32:
            {
                # 断言张量的第一个维度大小为 float 类型的大小
                GGML_ASSERT(tensor->nb[0] == sizeof(float));
                # 返回第 i 个元素的 float 类型数据
                return ((float *)(tensor->data))[i];
            }
        # 默认情况
        default:
            {
                # 断言条件为假
                GGML_ASSERT(false);
            }
    }

    # 如果没有匹配到任何数据类型，返回默认值 0.0f
    return 0.0f;
}



void ggml_set_f32_1d(const struct ggml_tensor * tensor, int i, float value) {
    // 如果张量不是连续的，则需要根据索引解开并设置值
    if (!ggml_is_contiguous(tensor)) {
        // 创建一个包含四个元素的索引数组，并解开索引
        int64_t id[4] = { 0, 0, 0, 0 };
        ggml_unravel_index(tensor, i, &id[0], &id[1], &id[2], &id[3]);
        // 调用多维设置函数设置值
        ggml_set_f32_nd(tensor, id[0], id[1], id[2], id[3], value);
        return;
    }
    // 根据张量类型选择不同的处理方式
    switch (tensor->type) {
        case GGML_TYPE_I8:
            {
                // 断言张量的字节数等于 int8_t 类型的字节数
                GGML_ASSERT(tensor->nb[0] == sizeof(int8_t));
                // 将值设置到 int8_t 类型的数据中
                ((int8_t *)(tensor->data))[i] = value;
            } break;
        case GGML_TYPE_I16:
            {
                // 断言张量的字节数等于 int16_t 类型的字节数
                GGML_ASSERT(tensor->nb[0] == sizeof(int16_t));
                // 将值设置到 int16_t 类型的数据中
                ((int16_t *)(tensor->data))[i] = value;
            } break;
        case GGML_TYPE_I32:
            {
                // 断言张量的字节数等于 int32_t 类型的字节数
                GGML_ASSERT(tensor->nb[0] == sizeof(int32_t));
                // 将值设置到 int32_t 类型的数据中
                ((int32_t *)(tensor->data))[i] = value;
            } break;
        case GGML_TYPE_F16:
            {
                // 断言张量的字节数等于 ggml_fp16_t 类型的字节数
                GGML_ASSERT(tensor->nb[0] == sizeof(ggml_fp16_t));
                // 将值设置到 ggml_fp16_t 类型的数据中，将 float 转换为 ggml_fp16_t
                ((ggml_fp16_t *)(tensor->data))[i] = GGML_FP32_TO_FP16(value);
            } break;
        case GGML_TYPE_F32:
            {
                // 断言张量的字节数等于 float 类型的字节数
                GGML_ASSERT(tensor->nb[0] == sizeof(float));
                // 将值设置到 float 类型的数据中
                ((float *)(tensor->data))[i] = value;
            } break;
        default:
            {
                // 断言失败
                GGML_ASSERT(false);
            } break;
    }
}

float ggml_get_f32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3) {
    // 计算多维索引对应的数据地址
    void * data   = (char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1] + i2*tensor->nb[2] + i3*tensor->nb[3];
    # 根据张量的类型进行不同的处理
    switch (tensor->type) {
        # 如果类型为8位整数，返回第一个元素的值
        case GGML_TYPE_I8:
            return ((int8_t *) data)[0];
        # 如果类型为16位整数，返回第一个元素的值
        case GGML_TYPE_I16:
            return ((int16_t *) data)[0];
        # 如果类型为32位整数，返回第一个元素的值
        case GGML_TYPE_I32:
            return ((int32_t *) data)[0];
        # 如果类型为16位浮点数，将16位浮点数转换为32位浮点数后返回第一个元素的值
        case GGML_TYPE_F16:
            return GGML_FP16_TO_FP32(((ggml_fp16_t *) data)[0]);
        # 如果类型为32位浮点数，返回第一个元素的值
        case GGML_TYPE_F32:
            return ((float *) data)[0];
        # 默认情况下，断言为假
        default:
            GGML_ASSERT(false);
    }

    # 如果没有匹配到任何类型，返回0.0f
    return 0.0f;
}

// 设置四维张量中指定位置的值为单精度浮点数
void ggml_set_f32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3, float value) {
    // 计算数据在内存中的位置
    void * data   = (char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1] + i2*tensor->nb[2] + i3*tensor->nb[3];
    // 根据张量的数据类型进行赋值操作
    switch (tensor->type) {
        case GGML_TYPE_I8:
            {
                ((int8_t *)(data))[0] = value;
            } break;
        case GGML_TYPE_I16:
            {
                ((int16_t *)(data))[0] = value;
            } break;
        case GGML_TYPE_I32:
            {
                ((int32_t *)(data))[0] = value;
            } break;
        case GGML_TYPE_F16:
            {
                ((ggml_fp16_t *)(data))[0] = GGML_FP32_TO_FP16(value);
            } break;
        case GGML_TYPE_F32:
            {
                ((float *)(data))[0] = value;
            } break;
        default:
            {
                // 断言，如果出现未知的数据类型，则终止程序
                GGML_ASSERT(false);
            } break;
    }
}

// 获取张量的数据指针
void * ggml_get_data(const struct ggml_tensor * tensor) {
    return tensor->data;
}

// 获取张量数据的单精度浮点数指针
float * ggml_get_data_f32(const struct ggml_tensor * tensor) {
    // 断言，确保张量的数据类型为单精度浮点数
    assert(tensor->type == GGML_TYPE_F32);
    return (float *)(tensor->data);
}

// 获取一元操作的操作符
GGML_CALL enum ggml_unary_op ggml_get_unary_op(const struct ggml_tensor * tensor) {
    // 断言，确保张量的操作类型为一元操作
    GGML_ASSERT(tensor->op == GGML_OP_UNARY);
    return (enum ggml_unary_op) ggml_get_op_params_i32(tensor, 0);
}

// 获取张量的名称
const char * ggml_get_name(const struct ggml_tensor * tensor) {
    return tensor->name;
}

// 设置张量的名称
struct ggml_tensor * ggml_set_name(struct ggml_tensor * tensor, const char * name) {
    // 将名称拷贝到张量的名称字段中
    strncpy(tensor->name, name, sizeof(tensor->name));
    tensor->name[sizeof(tensor->name) - 1] = '\0';
    return tensor;
}

// 格式化张量的名称
struct ggml_tensor * ggml_format_name(struct ggml_tensor * tensor, const char * fmt, ...) {
    va_list args;
    va_start(args, fmt);
    // 使用可变参数列表格式化名称
    vsnprintf(tensor->name, sizeof(tensor->name), fmt, args);
    va_end(args);
    return tensor;
}
// 创建一个新的张量，用于查看原始张量的内容
struct ggml_tensor * ggml_view_tensor(
        struct ggml_context * ctx,
        struct ggml_tensor  * src) {
    // 使用 ggml_new_tensor_impl 函数创建一个新的张量，与原始张量类型相同，维度为 GGML_MAX_DIMS，元素数量为 src->ne，数据指向原始张量 src，偏移为 0
    struct ggml_tensor * result = ggml_new_tensor_impl(ctx, src->type, GGML_MAX_DIMS, src->ne, src, 0);
    // 为新张量设置名称，格式为 "%s (view)"，其中 %s 为原始张量的名称
    ggml_format_name(result, "%s (view)", src->name);

    // 复制原始张量的维度信息到新张量
    for (int i = 0; i < GGML_MAX_DIMS; i++) {
        result->nb[i] = src->nb[i];
    }

    // 返回新的张量
    return result;
}

// 获取上下文中的第一个张量对象
struct ggml_tensor * ggml_get_first_tensor(const struct ggml_context * ctx) {
    // 从上下文的 objects_begin 开始遍历对象
    struct ggml_object * obj = ctx->objects_begin;

    // 获取上下文的内存缓冲区
    char * const mem_buffer = ctx->mem_buffer;

    // 遍历对象链表，找到第一个类型为 GGML_OBJECT_TENSOR 的对象，返回对应的张量指针
    while (obj != NULL) {
        if (obj->type == GGML_OBJECT_TENSOR) {
            return (struct ggml_tensor *)(mem_buffer + obj->offs);
        }

        obj = obj->next;
    }

    // 如果没有找到符合条件的张量对象，返回 NULL
    return NULL;
}

// 获取上下文中下一个张量对象
struct ggml_tensor * ggml_get_next_tensor(const struct ggml_context * ctx, struct ggml_tensor * tensor) {
    // 根据当前张量指针计算出对应的对象指针
    struct ggml_object * obj = (struct ggml_object *) ((char *)tensor - GGML_OBJECT_SIZE);
    // 获取下一个对象
    obj = obj->next;

    // 获取上下文的内存缓冲区
    char * const mem_buffer = ctx->mem_buffer;

    // 遍历对象链表，找到下一个类型为 GGML_OBJECT_TENSOR 的对象，返回对应的张量指针
    while (obj != NULL) {
        if (obj->type == GGML_OBJECT_TENSOR) {
            return (struct ggml_tensor *)(mem_buffer + obj->offs);
        }

        obj = obj->next;
    }

    // 如果没有找到符合条件的张量对象，返回 NULL
    return NULL;
}

// 根据名称获取上下文中的张量对象
struct ggml_tensor * ggml_get_tensor(struct ggml_context * ctx, const char * name) {
    // 从上下文的 objects_begin 开始遍历对象
    struct ggml_object * obj = ctx->objects_begin;

    // 获取上下文的内存缓冲区
    char * const mem_buffer = ctx->mem_buffer;

    // 遍历对象链表，找到类型为 GGML_OBJECT_TENSOR 且名称与给定名称相同的对象，返回对应的张量指针
    while (obj != NULL) {
        if (obj->type == GGML_OBJECT_TENSOR) {
            struct ggml_tensor * cur = (struct ggml_tensor *)(mem_buffer + obj->offs);
            if (strcmp(cur->name, name) == 0) {
                return cur;
            }
        }

        obj = obj->next;
    }

    // 如果没有找到符合条件的张量对象，返回 NULL
    return NULL;
}

////////////////////////////////////////////////////////////////////////////////

// ggml_dup

// 复制张量的实现函数，根据 inplace 参数决定是否原地复制
static struct ggml_tensor * ggml_dup_impl(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        bool inplace) {
    // 初始化一个标志位，表示是否为节点
    bool is_node = false;
    // 如果不是原地操作且输入张量有梯度，则将 is_node 设置为 true
    if (!inplace && (a->grad)) {
        is_node = true;
    }

    // 根据是否原地操作选择创建视图张量或复制张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置结果张量的操作类型为复制
    result->op   = GGML_OP_DUP;
    // 如果是节点，则为结果张量创建梯度张量，否则梯度为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为输入张量 a
    result->src[0] = a;

    // 返回结果张量
    return result;
// 复制一个张量，不进行原地操作
struct ggml_tensor * ggml_dup(
        struct ggml_context * ctx,
        struct ggml_tensor * a) {
    return ggml_dup_impl(ctx, a, false);
}

// 复制一个张量，进行原地操作
struct ggml_tensor * ggml_dup_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor * a) {
    return ggml_dup_impl(ctx, a, true);
}

// 实现张量相加的函数
static struct ggml_tensor * ggml_add_impl(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b,
        bool inplace) {
    // 断言张量 b 可以重复张量 a
    GGML_ASSERT(ggml_can_repeat(b, a));

    bool is_node = false;

    // 如果不是原地操作且张量 a 或 b 具有梯度
    if (!inplace && (a->grad || b->grad)) {
        // TODO: 支持广播的反向传播
        GGML_ASSERT(ggml_are_same_shape(a, b));
        is_node = true;
    }

    // 创建结果张量，根据是否原地操作选择复制或视图
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置结果张量的操作类型为相加
    result->op   = GGML_OP_ADD;
    // 如果是节点，复制结果张量作为梯度，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;
    result->src[1] = b;

    return result;
}

// 张量相加的外部接口函数
struct ggml_tensor * ggml_add(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b) {
    return ggml_add_impl(ctx, a, b, false);
}

// 原地张量相加的外部接口函数
struct ggml_tensor * ggml_add_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b) {
    return ggml_add_impl(ctx, a, b, true);
}

// 实现张量相加并进行类型转换的函数
static struct ggml_tensor * ggml_add_cast_impl(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b,
        enum   ggml_type     type) {
    // TODO: 支持更宽松的约束
    //       GGML_ASSERT(ggml_can_repeat(b, a));
    // 断言张量 b 可以重复张量 a 的行
    GGML_ASSERT(ggml_can_repeat_rows(b, a));
    // 断言张量 a 的类型为量化或者 f16，目前仅支持这两种类型
    GGML_ASSERT(ggml_is_quantized(a->type) || a->type == GGML_TYPE_F16);

    bool is_node = false;
    // 如果 a 或 b 中有一个需要梯度计算，则需要支持广播的反向传播
    if (a->grad || b->grad) {
        // 断言 a 和 b 的形状相同
        GGML_ASSERT(ggml_are_same_shape(a, b));
        // 设置节点标志为 true
        is_node = true;
    }

    // 创建一个新的张量 result，包含指定类型和维度，维度与张量 a 的元素个数相同
    struct ggml_tensor * result = ggml_new_tensor(ctx, type, GGML_MAX_DIMS, a->ne);

    // 设置 result 的操作为加法
    result->op   = GGML_OP_ADD;
    // 如果 is_node 为 true，则为 result 创建一个新的梯度张量，否则为 NULL
    result->grad = is_node ? ggml_new_tensor(ctx, GGML_TYPE_F32, GGML_MAX_DIMS, a->ne) : NULL;
    // 设置 result 的源张量为 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    // 返回 result
    return result;
}

// 添加一个类型转换操作
struct ggml_tensor * ggml_add_cast(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b,
        enum   ggml_type     type) {
    // 调用内部实现函数进行类型转换操作
    return ggml_add_cast_impl(ctx, a, b, type);
}

// ggml_add1

// 内部实现函数，用于执行加法操作
static struct ggml_tensor * ggml_add1_impl(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b,
        bool inplace) {
    // 断言 b 是标量
    GGML_ASSERT(ggml_is_scalar(b));
    // 断言 a 是一维张量
    GGML_ASSERT(ggml_is_padded_1d(a));

    bool is_node = false;

    // 如果 a 或 b 具有梯度信息，则设置 is_node 为 true
    if (a->grad || b->grad) {
        is_node = true;
    }

    // 创建一个新的结果张量，根据 inplace 参数决定是创建视图还是复制张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置结果张量的操作类型为加法
    result->op   = GGML_OP_ADD1;
    // 如果 is_node 为 true，则为结果张量创建一个梯度张量，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;
    result->src[1] = b;

    return result;
}

// 执行加法操作
struct ggml_tensor * ggml_add1(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b) {
    return ggml_add1_impl(ctx, a, b, false);
}

// 在原地执行加法操作
struct ggml_tensor * ggml_add1_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b) {
    return ggml_add1_impl(ctx, a, b, true);
}

// ggml_acc

// 内部实现函数，用于执行累加操作
static struct ggml_tensor * ggml_acc_impl(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b,
        size_t               nb1,
        size_t               nb2,
        size_t               nb3,
        size_t               offset,
        bool inplace) {
    // 断言 b 的元素数量小于等于 a 的元素数量
    GGML_ASSERT(ggml_nelements(b) <= ggml_nelements(a));
    // 断言 a 是连续的张量
    GGML_ASSERT(ggml_is_contiguous(a));
    // 断言 a 和 b 的类型为 GGML_TYPE_F32
    GGML_ASSERT(a->type == GGML_TYPE_F32);
    GGML_ASSERT(b->type == GGML_TYPE_F32);

    bool is_node = false;

    // 如果不是原地操作且 a 或 b 具有梯度信息，则设置 is_node 为 true
    if (!inplace && (a->grad || b->grad)) {
        is_node = true;
    }

    // 创建一个新的结果张量，根据 inplace 参数决定是创建视图还是复制张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 定义参数数组
    int32_t params[] = { nb1, nb2, nb3, offset, inplace ? 1 : 0 };
    # 设置操作的参数
    ggml_set_op_params(result, params, sizeof(params));

    # 设置操作类型为 GGML_OP_ACC
    result->op   = GGML_OP_ACC;
    # 如果是节点，则复制结果张量作为梯度，否则梯度为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    # 设置操作的第一个输入源为 a
    result->src[0] = a;
    # 设置操作的第二个输入源为 b
    result->src[1] = b;

    # 返回操作结果
    return result;
// 结束 ggml_sub_impl 函数定义

static struct ggml_tensor * ggml_sub_impl(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b,
        bool inplace) {
    // 断言两个张量的形状相同
    GGML_ASSERT(ggml_are_same_shape(a, b));

    // 初始化是否为节点的标志为 false
    bool is_node = false;

    // 如果不是原地操作且至少一个张量具有梯度，则将 is_node 标志设置为 true
    if (!inplace && (a->grad || b->grad)) {
        is_node = true;
    }

    // 根据是否原地操作选择复制张量或者视图张量作为结果
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置结果张量的操作为减法
    result->op   = GGML_OP_SUB;
    // 如果是节点，则为结果张量分配一个梯度张量，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    // 返回结果张量
    return result;
}

// 定义 ggml_sub 函数，实现张量的减法操作

struct ggml_tensor * ggml_sub(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b) {
    // 调用 ggml_sub_impl 函数执行减法操作，不是原地操作
    return ggml_sub_impl(ctx, a, b, false);
}

// 定义 ggml_sub_inplace 函数，实现张量的原地减法操作

struct ggml_tensor * ggml_sub_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b) {
    // 调用 ggml_sub_impl 函数执行减法操作，是原地操作
    return ggml_sub_impl(ctx, a, b, true);
}

// ggml_mul

// 定义 ggml_mul_impl 函数，实现张量的乘法操作

static struct ggml_tensor * ggml_mul_impl(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b,
        bool inplace) {
    // 断言张量 b 可以重复与张量 a 的形状
    GGML_ASSERT(ggml_can_repeat(b, a));

    // 初始化是否为节点的标志为 false
    bool is_node = false;
    // 如果不是原地操作且其中一个操作数有梯度，则需要支持广播的反向传播
    if (!inplace && (a->grad || b->grad)) {
        // TODO: support backward pass for broadcasting
        // 断言两个操作数的形状相同
        GGML_ASSERT(ggml_are_same_shape(a, b));
        // 设置标志位为 true
        is_node = true;
    }

    // 如果是原地操作
    if (inplace) {
        // 断言不是节点操作
        GGML_ASSERT(!is_node);
    }

    // 根据是否原地操作选择复制或者视图操作
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置结果张量的操作为乘法
    result->op   = GGML_OP_MUL;
    // 如果是节点操作，则复制结果张量作为梯度，否则梯度为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源操作数
    result->src[0] = a;
    result->src[1] = b;

    // 返回结果张量
    return result;
// 结构体 ggml_tensor 的乘法操作，返回乘法结果
struct ggml_tensor * ggml_mul(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    return ggml_mul_impl(ctx, a, b, false);
}

// 结构体 ggml_tensor 的原地乘法操作，返回乘法结果
struct ggml_tensor * ggml_mul_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    return ggml_mul_impl(ctx, a, b, true);
}

// ggml_div

// 内部实现函数，用于结构体 ggml_tensor 的除法操作
static struct ggml_tensor * ggml_div_impl(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b,
        bool inplace) {
    // 断言 b 可以重复使用 a
    GGML_ASSERT(ggml_can_repeat(b, a));

    bool is_node = false;

    // 如果不是原地操作且 a 或 b 具有梯度信息，则设置 is_node 为 true
    if (!inplace && (a->grad || b->grad)) {
        is_node = true;
    }

    // 如果是原地操作，则断言不是节点
    if (inplace) {
        GGML_ASSERT(!is_node);
    }

    // 创建结果结构体，根据是否原地操作选择复制或视图
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置结果结构体的操作类型为除法
    result->op   = GGML_OP_DIV;
    // 如果是节点，则复制结果结构体作为梯度信息，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;
    result->src[1] = b;

    return result;
}

// 结构体 ggml_tensor 的除法操作，返回除法结果
struct ggml_tensor * ggml_div(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    return ggml_div_impl(ctx, a, b, false);
}

// 结构体 ggml_tensor 的原地除法操作，返回除法结果
struct ggml_tensor * ggml_div_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    return ggml_div_impl(ctx, a, b, true);
}

// ggml_sqr

// 内部实现函数，用于结构体 ggml_tensor 的平方操作
static struct ggml_tensor * ggml_sqr_impl(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        bool inplace) {
    bool is_node = false;

    // 如果不是原地操作且 a 具有梯度信息，则设置 is_node 为 true
    if (!inplace && (a->grad)) {
        is_node = true;
    }

    // 创建结果结构体，根据是否原地操作选择复制或视图
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置结果结构体的操作类型为平方
    result->op   = GGML_OP_SQR;
    // 如果是节点，则复制结果结构体作为梯度信息，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;

    return result;
}

// 结构体 ggml_tensor 的平方操作，返回平方结果
struct ggml_tensor * ggml_sqr(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    # 调用 ggml_sqr_impl 函数，传入参数 ctx, a, false，并返回结果
    return ggml_sqr_impl(ctx, a, false);
}

// 对输入张量进行平方操作，返回结果张量
struct ggml_tensor * ggml_sqr_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_sqr_impl(ctx, a, true);
}

// 实现平方根操作
static struct ggml_tensor * ggml_sqrt_impl(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        bool inplace) {
    bool is_node = false;

    // 如果不是原地操作且输入张量具有梯度，则设置为节点
    if (!inplace && (a->grad)) {
        is_node = true;
    }

    // 创建结果张量，根据是否原地操作选择不同的方式
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置结果张量的操作类型和梯度
    result->op   = GGML_OP_SQRT;
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;

    return result;
}

// 对输入张量进行平方根操作，返回结果张量
struct ggml_tensor * ggml_sqrt(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_sqrt_impl(ctx, a, false);
}

// 对输入张量进行原地平方根操作，返回结果张量
struct ggml_tensor * ggml_sqrt_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_sqrt_impl(ctx, a, true);
}

// 实现对数操作
static struct ggml_tensor * ggml_log_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        bool inplace) {
    bool is_node = false;

    // 如果不是原地操作且输入张量具有梯度，则设置为节点
    if (!inplace && (a->grad)) {
        is_node = true;
    }

    // 创建结果张量，根据是否原地操作选择不同的方式
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置结果张量的操作类型和梯度
    result->op   = GGML_OP_LOG;
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;

    return result;
}

// 对输入张量进行对数操作，返回结果张量
struct ggml_tensor * ggml_log(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_log_impl(ctx, a, false);
}

// 对输入张量进行原地对数操作，返回结果张量
struct ggml_tensor * ggml_log_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_log_impl(ctx, a, true);
}

// 计算输入张量的元素和，返回结果张量
struct ggml_tensor * ggml_sum(
        struct ggml_context * ctx,
        struct ggml_tensor * a) {
    bool is_node = false;

    // 如果输入张量具有梯度，则设置为节点
    if (a->grad) {
        is_node = true;
    }

    // 创建结果张量，为一维张量，元素个数为1
    struct ggml_tensor * result = ggml_new_tensor_1d(ctx, a->type, 1);
    // 设置结果节点的操作为求和
    result->op   = GGML_OP_SUM;
    // 如果是节点，则将结果节点的梯度设置为与结果节点相同的张量，否则设置为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果节点的第一个源节点为a
    result->src[0] = a;

    // 返回结果节点
    return result;
// ggml_sum_rows

// 对输入张量的行进行求和操作，返回结果张量
struct ggml_tensor * ggml_sum_rows(
        struct ggml_context * ctx,
        struct ggml_tensor * a) {
    // 初始化节点标志为假
    bool is_node = false;

    // 如果输入张量具有梯度信息，则将节点标志设置为真
    if (a->grad) {
        is_node = true;
    }

    // 初始化一个大小为 GGML_MAX_DIMS 的数组 ne，所有元素初始化为 1
    int64_t ne[GGML_MAX_DIMS] = { 1 };
    // 复制输入张量的维度信息到 ne 数组
    for (int i = 1; i < GGML_MAX_DIMS; ++i) {
        ne[i] = a->ne[i];
    }

    // 创建一个新的结果张量，维度信息为 ne 数组中的值
    struct ggml_tensor * result = ggml_new_tensor(ctx, a->type, GGML_MAX_DIMS, ne);

    // 设置结果张量的操作类型为 GGML_OP_SUM_ROWS
    result->op   = GGML_OP_SUM_ROWS;
    // 如果是节点，则复制结果张量作为梯度信息，否则梯度信息为空
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为输入张量
    result->src[0] = a;

    // 返回结果张量
    return result;
}

// ggml_mean

// 对输入张量进行均值计算，返回结果张量
struct ggml_tensor * ggml_mean(
        struct ggml_context * ctx,
        struct ggml_tensor * a) {
    // 初始化节点标志为假
    bool is_node = false;

    // 如果输入张量具有梯度信息，则抛出异常，设置节点标志为真
    if (a->grad) {
        GGML_ASSERT(false); // TODO: implement
        is_node = true;
    }

    // 初始化一个大小为 4 的数组 ne，第一个元素为 1，其余元素为输入张量的维度信息
    int64_t ne[4] = { 1, a->ne[1], a->ne[2], a->ne[3] };
    // 创建一个新的结果张量，类型为 GGML_TYPE_F32，维度信息为 ne 数组中的值
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F32, 4, ne);

    // 设置结果张量的操作类型为 GGML_OP_MEAN
    result->op   = GGML_OP_MEAN;
    // 如果是节点，则复制结果张量作为梯度信息，否则梯度信息为空
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为输入张量
    result->src[0] = a;

    // 返回结果张量
    return result;
}

// ggml_argmax

// 对输入张量进行 argmax 操作，返回结果张量
struct ggml_tensor * ggml_argmax(
        struct ggml_context * ctx,
        struct ggml_tensor * a) {
    // 断言输入张量为矩阵
    GGML_ASSERT(ggml_is_matrix(a));
    // 初始化节点标志为假
    bool is_node = false;

    // 如果输入张量具有梯度信息，则抛出异常，设置节点标志为真
    if (a->grad) {
        GGML_ASSERT(false);
        is_node = true;
    }

    // 创建一个新的结果张量，类型为 GGML_TYPE_I32，大小为输入张量的第二维度大小
    struct ggml_tensor * result = ggml_new_tensor_1d(ctx, GGML_TYPE_I32, a->ne[1]);

    // 设置结果张量的操作类型为 GGML_OP_ARGMAX
    result->op   = GGML_OP_ARGMAX;
    // 如果是节点，则复制结果张量作为梯度信息，否则梯度信息为空
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为输入张量
    result->src[0] = a;

    // 返回结果张量
    return result;
}

// ggml_repeat

// 对输入张量进行重复操作，返回结果张量
struct ggml_tensor * ggml_repeat(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b) {
    // 断言输入张量 a 和 b 可以进行重复操作
    GGML_ASSERT(ggml_can_repeat(a, b));

    // 初始化节点标志为假
    bool is_node = false;

    // 如果输入张量 a 具有梯度信息，则设置节点标志为真
    if (a->grad) {
        is_node = true;
    }

    // 创建一个新的结果张量，类型与输入张量 a 相同，维度信息为输入张量 b 的维度信息
    struct ggml_tensor * result = ggml_new_tensor(ctx, a->type, GGML_MAX_DIMS, b->ne);

    // 设置结果张量的操作类型为 GGML_OP_REPEAT
    result->op   = GGML_OP_REPEAT;
    // 如果是节点，则复制结果张量，否则为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 将第一个源张量赋值给结果的第一个源张量
    result->src[0] = a;

    // 返回结果
    return result;
// ggml_repeat_back

// 重复反向传播函数，根据输入张量 a 和 b 返回一个新的张量
struct ggml_tensor * ggml_repeat_back(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b) {
    // 确保可以重复张量 b 到张量 a
    GGML_ASSERT(ggml_can_repeat(b, a));

    // 初始化一个布尔值用于判断是否为节点
    bool is_node = false;

    // 如果输入张量 a 有梯度，则将 is_node 设置为 true
    if (a->grad) {
        is_node = true;
    }

    // 如果输入张量 a 和 b 形状相同且不是节点，则直接返回输入张量 a
    if (ggml_are_same_shape(a, b) && !is_node) {
        return a;
    }

    // 创建一个新的张量 result，形状为 a 的类型，最大维度为 b 的维度
    struct ggml_tensor * result = ggml_new_tensor(ctx, a->type, GGML_MAX_DIMS, b->ne);

    // 设置结果张量的操作类型为重复反向传播
    result->op   = GGML_OP_REPEAT_BACK;
    // 如果是节点，则将结果张量的梯度设置为 result 的副本，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为输入张量 a
    result->src[0] = a;

    // 返回结果张量
    return result;
}

// ggml_concat

// 连接函数，根据输入张量 a 和 b 返回一个新的张量
struct ggml_tensor * ggml_concat(
    struct ggml_context* ctx,
    struct ggml_tensor* a,
    struct ggml_tensor* b) {
    // 确保输入张量 a 和 b 的第一、第二和第四维度相同
    GGML_ASSERT(a->ne[0] == b->ne[0] && a->ne[1] == b->ne[1] && a->ne[3] == b->ne[3]);

    // 初始化一个布尔值用于判断是否为节点
    bool is_node = false;

    // 如果输入张量 a 或 b 有梯度，则将 is_node 设置为 true
    if (a->grad || b->grad) {
        is_node = true;
    }

    // 创建一个新的张量 result，形状为 a 的类型，第一、第二和第三维度为 a 的对应维度之和，第四维度为 a 的第四维度
    struct ggml_tensor * result = ggml_new_tensor_4d(ctx, a->type, a->ne[0], a->ne[1], a->ne[2] + b->ne[2], a->ne[3]);

    // 设置结果张量的操作类型为连接
    result->op = GGML_OP_CONCAT;
    // 如果是节点，则将结果张量的梯度设置为 result 的副本，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为输入张量 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    // 返回结果张量
    return result;
}

// ggml_abs

// 绝对值函数，对输入张量 a 进行绝对值操作
struct ggml_tensor * ggml_abs(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    // 调用 ggml_unary 函数，传入绝对值操作符
    return ggml_unary(ctx, a, GGML_UNARY_OP_ABS);
}

// ggml_abs_inplace

// 原地绝对值函数，对输入张量 a 进行原地绝对值操作
struct ggml_tensor * ggml_abs_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    // 调用 ggml_unary_inplace 函数，传入绝对值操作符
    return ggml_unary_inplace(ctx, a, GGML_UNARY_OP_ABS);
}

// ggml_sgn

// 符号函数，对输入张量 a 进行符号操作
struct ggml_tensor * ggml_sgn(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    // 调用 ggml_unary 函数，传入符号操作符
    return ggml_unary(ctx, a, GGML_UNARY_OP_SGN);
}

// ggml_sgn_inplace

// 原地符号函数，对输入张量 a 进行原地符号操作
struct ggml_tensor * ggml_sgn_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    // 调用 ggml_unary_inplace 函数，传入符号操作符
    return ggml_unary_inplace(ctx, a, GGML_UNARY_OP_SGN);

}

// ggml_neg

// 取负函数，对输入张量 a 进行取负操作
struct ggml_tensor * ggml_neg(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    # 返回对给定参数 a 执行负数运算的结果
    return ggml_unary(ctx, a, GGML_UNARY_OP_NEG);
// ggml_neg_inplace函数，对输入的张量进行取负操作，返回结果张量
struct ggml_tensor * ggml_neg_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    // 调用ggml_unary_inplace函数，传入上下文和操作类型GGML_UNARY_OP_NEG
    return ggml_unary_inplace(ctx, a, GGML_UNARY_OP_NEG);
}

// ggml_step函数，对输入的张量进行阶跃函数操作，返回结果张量
struct ggml_tensor * ggml_step(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    // 调用ggml_unary函数，传入上下文和操作类型GGML_UNARY_OP_STEP
    return ggml_unary(ctx, a, GGML_UNARY_OP_STEP);
}

// ggml_step_inplace函数，对输入的张量进行阶跃函数操作，结果覆盖原张量
struct ggml_tensor * ggml_step_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    // 调用ggml_unary_inplace函数，传入上下文和操作类型GGML_UNARY_OP_STEP
    return ggml_unary_inplace(ctx, a, GGML_UNARY_OP_STEP);
}

// ggml_tanh函数，对输入的张量进行双曲正切函数操作，返回结果张量
struct ggml_tensor * ggml_tanh(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    // 调用ggml_unary函数，传入上下文和操作类型GGML_UNARY_OP_TANH
    return ggml_unary(ctx, a, GGML_UNARY_OP_TANH);
}

// ggml_tanh_inplace函数，对输入的张量进行双曲正切函数操作，结果覆盖原张量
struct ggml_tensor * ggml_tanh_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    // 调用ggml_unary_inplace函数，传入上下文和操作类型GGML_UNARY_OP_TANH
    return ggml_unary_inplace(ctx, a, GGML_UNARY_OP_TANH);
}

// ggml_elu函数，对输入的张量进行ELU函数操作，返回结果张量
struct ggml_tensor * ggml_elu(
    struct ggml_context * ctx,
    struct ggml_tensor  * a) {
    // 调用ggml_unary函数，传入上下文和操作类型GGML_UNARY_OP_ELU
    return ggml_unary(ctx, a, GGML_UNARY_OP_ELU);
}

// ggml_elu_inplace函数，对输入的张量进行ELU函数操作，结果覆盖原张量
struct ggml_tensor * ggml_elu_inplace(
    struct ggml_context * ctx,
    struct ggml_tensor  * a) {
    // 调用ggml_unary_inplace函数，传入上下文和操作类型GGML_UNARY_OP_ELU
    return ggml_unary_inplace(ctx, a, GGML_UNARY_OP_ELU);
}

// ggml_relu函数，对输入的张量进行ReLU函数操作，返回结果张量
struct ggml_tensor * ggml_relu(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    // 调用ggml_unary函数，传入上下文和操作类型GGML_UNARY_OP_RELU
    return ggml_unary(ctx, a, GGML_UNARY_OP_RELU);
}

// ggml_relu_inplace函数，对输入的张量进行ReLU函数操作，结果覆盖原张量
struct ggml_tensor * ggml_relu_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    // 调用ggml_unary_inplace函数，传入上下文和操作类型GGML_UNARY_OP_RELU
    return ggml_unary_inplace(ctx, a, GGML_UNARY_OP_RELU);
}

// ggml_leaky_relu函数，对输入的张量进行Leaky ReLU函数操作，返回结果张量
struct ggml_tensor * ggml_leaky_relu(
        struct ggml_context * ctx,
        struct ggml_tensor  * a, float negative_slope, bool inplace) {
    // 初始化is_node为false
    bool is_node = false;

    // 如果不是原地操作且输入张量有梯度信息，则将is_node设为true
    if (!inplace && (a->grad)) {
        is_node = true;
    }

    // 根据是否原地操作选择复制或者视图操作，得到结果张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);
    // 设置Leaky ReLU操作的参数为负斜率
    ggml_set_op_params(result, &negative_slope, sizeof(negative_slope));

    // 设置结果张量的操作类型为GGML_OP_LEAKY_RELU
    result->op   = GGML_OP_LEAKY_RELU;
}
    // 如果是节点，则复制结果张量，否则为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 将第一个源张量赋值给结果的第一个源张量
    result->src[0] = a;

    // 返回结果
    return result;
// ggml_gelu

// 对输入张量应用 GELU 激活函数，返回结果张量
struct ggml_tensor * ggml_gelu(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_unary(ctx, a, GGML_UNARY_OP_GELU);
}

// 在原地对输入张量应用 GELU 激活函数，返回结果张量
struct ggml_tensor * ggml_gelu_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_unary_inplace(ctx, a, GGML_UNARY_OP_GELU);
}

// ggml_gelu_quick

// 对输入张量应用快速 GELU 激活函数，返回结果张量
struct ggml_tensor * ggml_gelu_quick(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_unary(ctx, a, GGML_UNARY_OP_GELU_QUICK);
}

// 在原地对输入张量应用快速 GELU 激活函数，返回结果张量
struct ggml_tensor * ggml_gelu_quick_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_unary_inplace(ctx, a, GGML_UNARY_OP_GELU_QUICK);
}

// ggml_silu

// 对输入张量应用 SiLU 激活函数，返回结果张量
struct ggml_tensor * ggml_silu(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_unary(ctx, a, GGML_UNARY_OP_SILU);
}

// 在原地对输入张量应用 SiLU 激活函数，返回结果张量
struct ggml_tensor * ggml_silu_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_unary_inplace(ctx, a, GGML_UNARY_OP_SILU);
}

// ggml_silu_back

// 计算 SiLU 激活函数的反向传播
struct ggml_tensor * ggml_silu_back(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    bool is_node = false;

    if (a->grad || b->grad) {
        // TODO: implement backward
        is_node = true;
    }

    // 复制输入张量 a，作为结果张量
    struct ggml_tensor * result = ggml_dup_tensor(ctx, a);

    // 设置结果张量的操作为 GGML_OP_SILU_BACK
    result->op   = GGML_OP_SILU_BACK;
    // 如果需要计算梯度，复制结果张量作为梯度张量
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;
    result->src[1] = b;

    return result;
}

// ggml hardswish

// 对输入张量应用 HardSwish 激活函数，返回结果张量
struct ggml_tensor * ggml_hardswish(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_unary(ctx, a, GGML_UNARY_OP_HARDSWISH);
}

// ggml hardsigmoid

// 对输入张量应用 HardSigmoid 激活函数，返回结果张量
struct ggml_tensor * ggml_hardsigmoid(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_unary(ctx, a, GGML_UNARY_OP_HARDSIGMOID);
}

// ggml_norm
// 实现对输入张量进行归一化操作，返回归一化后的张量
static struct ggml_tensor * ggml_norm_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        float eps,
        bool inplace) {
    // 初始化节点标志为假
    bool is_node = false;

    // 如果不是原地操作且输入张量有梯度
    if (!inplace && (a->grad)) {
        // 断言，提示需要实现反向传播
        GGML_ASSERT(false); // TODO: implement backward
        // 设置节点标志为真
        is_node = true;
    }

    // 根据是否原地操作选择复制或者视图操作
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置操作参数
    ggml_set_op_params(result, &eps, sizeof(eps));

    // 设置操作类型为归一化
    result->op   = GGML_OP_NORM;
    // 如果是节点，复制结果张量作为梯度张量，否则为空
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置源张量为输入张量
    result->src[0] = a;

    // 返回结果张量
    return result;
}

// 对输入张量进行归一化操作，返回结果张量
struct ggml_tensor * ggml_norm(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        float eps) {
    return ggml_norm_impl(ctx, a, eps, false);
}

// 对输入张量进行原地归一化操作，返回结果张量
struct ggml_tensor * ggml_norm_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        float eps) {
    return ggml_norm_impl(ctx, a, eps, true);
}

// ggml_rms_norm

// 实现对输入张量进行 RMS 归一化操作，返回归一化后的张量
static struct ggml_tensor * ggml_rms_norm_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        float eps,
        bool inplace) {
    // 初始化节点标志为假
    bool is_node = false;

    // 如果不是原地操作且输入张量有梯度
    if (!inplace && (a->grad)) {
        // 设置节点标志为真
        is_node = true;
    }

    // 根据是否原地操作选择复制或者视图操作
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置操作参数
    ggml_set_op_params(result, &eps, sizeof(eps));

    // 设置操作类型为 RMS 归一化
    result->op   = GGML_OP_RMS_NORM;
    // 如果是节点，复制结果张量作为梯度张量，否则为空
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置源张量为输入张量
    result->src[0] = a;

    // 返回结果张量
    return result;
}

// 对输入张量进行 RMS 归一化操作，返回结果张量
struct ggml_tensor * ggml_rms_norm(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        float  eps) {
    return ggml_rms_norm_impl(ctx, a, eps, false);
}

// 对输入张量进行原地 RMS 归一化操作，返回结果张量
struct ggml_tensor * ggml_rms_norm_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        float eps) {
    return ggml_rms_norm_impl(ctx, a, eps, true);
}

// ggml_rms_norm_back
// 计算 RMS Norm 反向传播
struct ggml_tensor * ggml_rms_norm_back(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        float  eps) {
    // 初始化节点标志为假
    bool is_node = false;

    // 如果输入张量 a 有梯度信息
    if (a->grad) {
        // TODO: 实现反向传播
        // 设置节点标志为真
        is_node = true;
    }

    // 复制输入张量 a 作为结果张量
    struct ggml_tensor * result = ggml_dup_tensor(ctx, a);

    // 设置操作参数为 eps
    ggml_set_op_params(result, &eps, sizeof(eps));

    // 设置结果张量的操作类型为 GGML_OP_RMS_NORM_BACK
    result->op   = GGML_OP_RMS_NORM_BACK;
    // 如果是节点，复制结果张量作为梯度张量；否则梯度张量为空
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    // 返回结果张量
    return result;
}

// ggml_group_norm

// 实现 Group Norm 操作
static struct ggml_tensor * ggml_group_norm_impl(
    struct ggml_context * ctx,
    struct ggml_tensor * a,
    int n_groups,
    bool inplace) {

    // 初始化节点标志为假
    bool is_node = false;
    // 如果不是原地操作且输入张量 a 有梯度信息
    if (!inplace && (a->grad)) {
        // 断言失败，提示实现反向传播
        GGML_ASSERT(false); // TODO: 实现反向传播
        // 设置节点标志为真
        is_node = true;
    }

    // 如果是原地操作，复制视图张量；否则复制输入张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置操作参数为 n_groups
    result->op_params[0] = n_groups;

    // 设置结果张量的操作类型为 GGML_OP_GROUP_NORM
    result->op = GGML_OP_GROUP_NORM;
    // 如果是节点，复制结果张量作为梯度张量；否则梯度张量为空
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为 a
    result->src[0] = a;

    // 返回结果张量
    return result;
}

// 对外接口，调用 Group Norm 操作
struct ggml_tensor * ggml_group_norm(
    struct ggml_context * ctx,
    struct ggml_tensor * a,
    int n_groups) {
    return ggml_group_norm_impl(ctx, a, n_groups, false);
}

// 对外接口，调用原地 Group Norm 操作
struct ggml_tensor * ggml_group_norm_inplace(
    struct ggml_context * ctx,
    struct ggml_tensor * a,
    int n_groups) {
    return ggml_group_norm_impl(ctx, a, n_groups, true);
}

// ggml_mul_mat

// 矩阵相乘操作
struct ggml_tensor * ggml_mul_mat(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    // 断言可以进行矩阵相乘
    GGML_ASSERT(ggml_can_mul_mat(a, b));
    // 断言输入张量 a 不是转置的
    GGML_ASSERT(!ggml_is_transposed(a));

    // 初始化节点标志为假
    bool is_node = false;

    // 如果输入张量 a 或 b 有梯度信息
    if (a->grad || b->grad) {
        // 设置节点标志为真
        is_node = true;
    }

    // 定义存储结果张量维度的数组
    const int64_t ne[4] = { a->ne[1], b->ne[1], b->ne[2], b->ne[3];
    // 创建一个新的浮点型张量，包含4个元素，ne为元素个数
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F32, 4, ne);

    // 设置张量的操作类型为矩阵相乘
    result->op   = GGML_OP_MUL_MAT;
    // 如果是节点，则复制结果张量作为梯度，否则梯度为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为a和b
    result->src[0] = a;
    result->src[1] = b;

    // 返回结果张量
    return result;
}

// 设置矩阵乘法操作的精度
void ggml_mul_mat_set_prec(
        struct ggml_tensor * a,
        enum ggml_prec       prec) {
    // 将枚举类型转换为int32_t类型
    const int32_t prec_i32 = (int32_t) prec;

    // 设置操作参数为prec_i32
    ggml_set_op_params_i32(a, 0, prec_i32);
}

// 矩阵乘法操作
struct ggml_tensor * ggml_mul_mat_id(
        struct ggml_context * ctx,
        struct ggml_tensor  * const as[],
        int                   n_as,
        struct ggml_tensor  * ids,
        int                   id,
        struct ggml_tensor  * b) {

    // 断言ids的类型为GGML_TYPE_I32
    GGML_ASSERT(ids->type == GGML_TYPE_I32);
    // 断言ids的ne[2]和ne[3]为1
    GGML_ASSERT(ids->ne[2] == 1 && ids->ne[3] == 1);
    // 断言ids的ne[1]等于b的ne[1]
    GGML_ASSERT(ids->ne[1] == b->ne[1]);
    // 断言ids的ne[2]和ne[3]等于b的ne[2]和ne[3]
    GGML_ASSERT(ids->ne[2] == b->ne[2] && ids->ne[3] == b->ne[3]);
    // 断言n_as大于0且小于等于GGML_MAX_SRC - 2
    GGML_ASSERT(n_as > 0 && n_as <= GGML_MAX_SRC - 2);
    // 断言id大于等于0且小于ids的ne[0]
    GGML_ASSERT(id >= 0 && id < ids->ne[0]);

    bool is_node = false;

    // 如果as[0]或b有梯度信息，则is_node为true
    if (as[0]->grad || b->grad) {
        is_node = true;
    }

    // 定义ne数组，存储as[0]和b的维度信息
    const int64_t ne[4] = { as[0]->ne[1], b->ne[1], b->ne[2], b->ne[3] };
    // 创建新的tensor对象result
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F32, 4, ne);

    // 设置操作参数为id和n_as
    ggml_set_op_params_i32(result, 0, id);
    ggml_set_op_params_i32(result, 1, n_as);

    // 设置result的操作为GGML_OP_MUL_MAT_ID
    result->op   = GGML_OP_MUL_MAT_ID;
    // 如果is_node为true，则result的grad为result的副本，否则为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = ids;
    result->src[1] = b;

    // 遍历as数组，设置result的源tensor
    for (int i = 0; i < n_as; i++) {
        struct ggml_tensor * a = as[i];
        // 断言as[0]和a的形状相同
        GGML_ASSERT(ggml_are_same_shape(as[0], a));
        // 断言a和b可以进行矩阵乘法
        GGML_ASSERT(ggml_can_mul_mat(a, b));
        // 断言a不是转置的
        GGML_ASSERT(!ggml_is_transposed(a));
        result->src[i + 2] = a;
    }

    return result;
}

// 外积操作
struct ggml_tensor * ggml_out_prod(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    // 断言a和b可以进行外积操作
    GGML_ASSERT(ggml_can_out_prod(a, b));
    // 断言a不是转置的
    GGML_ASSERT(!ggml_is_transposed(a));

    bool is_node = false;

    // 如果a或b有梯度信息，则is_node为true
    if (a->grad || b->grad) {
        is_node = true;
    }

    // a可以广播到b的ne[2]和ne[3]，使用b的ne[2]和ne[3]
    // 定义一个包含4个元素的常量整型数组ne，分别为a->ne[0], b->ne[0], b->ne[2], b->ne[3]
    const int64_t ne[4] = { a->ne[0], b->ne[0], b->ne[2], b->ne[3] };
    // 使用ggml_new_tensor函数创建一个新的结构体指针result，类型为GGML_TYPE_F32，维度为4，大小为ne数组中的元素
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F32, 4, ne);

    // 设置result结构体指针的op字段为GGML_OP_OUT_PROD
    result->op   = GGML_OP_OUT_PROD;
    // 如果is_node为真，则将result结构体指针的grad字段设置为result的副本，否则设置为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置result结构体指针的src[0]字段为a
    result->src[0] = a;
    // 设置result结构体指针的src[1]字段为b

    // 返回result结构体指针
    return result;
// ggml_scale

// 实现对张量进行缩放操作
static struct ggml_tensor * ggml_scale_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        float                 s,
        bool inplace) {
    // 断言输入张量为一维填充张量
    GGML_ASSERT(ggml_is_padded_1d(a));

    bool is_node = false;

    // 如果输入张量具有梯度信息，则设置为节点
    if (a->grad) {
        is_node = true;
    }

    // 根据是否原地操作选择创建视图或复制张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置操作参数为缩放因子
    ggml_set_op_params(result, &s, sizeof(s));

    // 设置操作类型为缩放
    result->op   = GGML_OP_SCALE;
    // 如果是节点，则复制结果张量作为梯度张量
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;

    return result;
}

// 对外接口，对张量进行缩放操作
struct ggml_tensor * ggml_scale(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        float                s) {
    return ggml_scale_impl(ctx, a, s, false);
}

// 对外接口，对张量进行原地缩放操作
struct ggml_tensor * ggml_scale_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        float                s) {
    return ggml_scale_impl(ctx, a, s, true);
}

// ggml_set

// 实现对张量进行设置操作
static struct ggml_tensor * ggml_set_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        size_t                nb1,
        size_t                nb2,
        size_t                nb3,
        size_t                offset,
        bool inplace) {
    // 断言目标张量元素数量大于等于源张量元素数量
    GGML_ASSERT(ggml_nelements(a) >= ggml_nelements(b));

    bool is_node = false;

    // 如果目标张量或源张量具有梯度信息，则设置为节点
    if (a->grad || b->grad) {
        is_node = true;
    }

    // 创建目标张量的视图或复制
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    int32_t params[] = { nb1, nb2, nb3, offset, inplace ? 1 : 0 };
    // 设置操作参数
    ggml_set_op_params(result, params, sizeof(params));

    // 设置操作类型为设置
    result->op   = GGML_OP_SET;
    // 如果是节点，则复制结果张量作为梯度张量
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;
    result->src[1] = b;

    return result;
}
// 设置两个张量的值，返回结果张量
struct ggml_tensor * ggml_set(
        struct ggml_context * ctx,
        struct ggml_tensor *  a,
        struct ggml_tensor *  b,
        size_t                nb1,
        size_t                nb2,
        size_t                nb3,
        size_t                offset) {
    // 调用内部实现函数 ggml_set_impl，传入参数和标志位 false
    return ggml_set_impl(ctx, a, b, nb1, nb2, nb3, offset, false);
}

// 在原地设置两个张量的值，返回结果张量
struct ggml_tensor * ggml_set_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor *  a,
        struct ggml_tensor *  b,
        size_t                nb1,
        size_t                nb2,
        size_t                nb3,
        size_t                offset) {
    // 调用内部实现函数 ggml_set_impl，传入参数和标志位 true
    return ggml_set_impl(ctx, a, b, nb1, nb2, nb3, offset, true);
}

// 设置一维张量的值，返回结果张量
struct ggml_tensor * ggml_set_1d(
        struct ggml_context * ctx,
        struct ggml_tensor *  a,
        struct ggml_tensor *  b,
        size_t                offset) {
    // 调用内部实现函数 ggml_set_impl，传入参数和标志位 false
    return ggml_set_impl(ctx, a, b, a->nb[1], a->nb[2], a->nb[3], offset, false);
}

// 在原地设置一维张量的值，返回结果张量
struct ggml_tensor * ggml_set_1d_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor *  a,
        struct ggml_tensor *  b,
        size_t                offset) {
    // 调用内部实现函数 ggml_set_impl，传入参数和标志位 true
    return ggml_set_impl(ctx, a, b, a->nb[1], a->nb[2], a->nb[3], offset, true);
}

// 设置二维张量的值，返回结果张量
struct ggml_tensor * ggml_set_2d(
        struct ggml_context * ctx,
        struct ggml_tensor *  a,
        struct ggml_tensor *  b,
        size_t                nb1,
        size_t                offset) {
    // 调用内部实现函数 ggml_set_impl，传入参数和标志位 false
    return ggml_set_impl(ctx, a, b, nb1, a->nb[2], a->nb[3], offset, false);
}

// 在原地设置二维张量的值，返回结果张量
struct ggml_tensor * ggml_set_2d_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor *  a,
        struct ggml_tensor *  b,
        size_t                nb1,
        size_t                offset) {
    // 调用内部实现函数 ggml_set_impl，传入参数和标志位 true
    return ggml_set_impl(ctx, a, b, nb1, a->nb[2], a->nb[3], offset, true);
}

// 复制张量的实现函数
static struct ggml_tensor * ggml_cpy_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    // 断言两个张量的元素个数相等
    GGML_ASSERT(ggml_nelements(a) == ggml_nelements(b));

    // 初始化一个布尔变量 is_node，用于标记是否为节点
    bool is_node = false;

    // 如果 a 或 b 中有梯度信息
    if (a->grad || b->grad) {
        // 设置 is_node 为 true
        is_node = true;
    }

    // 创建目标张量的视图
    struct ggml_tensor * result = ggml_view_tensor(ctx, b);
    
    // 如果 b 的名称长度大于 0
    if (strlen(b->name) > 0) {
        // 格式化目标张量的名称为 "b 的名称 (copy of a 的名称)"
        ggml_format_name(result, "%s (copy of %s)", b->name, a->name);
    } else {
        // 格式化目标张量的名称为 "a 的名称 (copy)"
        ggml_format_name(result, "%s (copy)", a->name);
    }

    // 设置目标张量的操作为复制
    result->op   = GGML_OP_CPY;
    // 如果是节点，则复制目标张量的梯度信息
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置目标张量的源张量为 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    // 返回目标张量
    return result;
// 复制一个张量，返回复制后的张量指针
struct ggml_tensor * ggml_cpy(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b) {
    调用 ggml_cpy_impl 函数实现张量的复制操作，并返回结果
    return ggml_cpy_impl(ctx, a, b);
}

// 将一个张量转换为指定类型，返回转换后的张量指针
struct ggml_tensor * ggml_cast(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        enum   ggml_type      type) {
    初始化一个布尔变量 is_node，并赋值为 false
    bool is_node = false;

    根据给定类型和原张量的 ne 属性创建一个新的张量 result
    struct ggml_tensor * result = ggml_new_tensor(ctx, type, GGML_MAX_DIMS, a->ne);
    为新张量设置名称为原张量名称后加上 "(copy)"
    ggml_format_name(result, "%s (copy)", a->name);

    设置新张量的操作类型为 GGML_OP_CPY
    result->op   = GGML_OP_CPY;
    根据 is_node 的值决定是否为新张量设置梯度张量
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    设置新张量的源张量为原张量 a，以及自身为源张量
    result->src[0] = a;
    result->src[1] = result;

    返回新张量 result
    return result;
}

// 实现连续操作的内部函数
static struct ggml_tensor * ggml_cont_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    初始化一个布尔变量 is_node，并赋值为 false
    bool is_node = false;

    如果原张量 a 的梯度不为空，则将 is_node 设置为 true
    if (a->grad) {
        is_node = true;
    }

    复制原张量 a，得到新的张量 result
    struct ggml_tensor * result = ggml_dup_tensor(ctx, a);
    为新张量设置名称为原张量名称后加上 "(cont)"
    ggml_format_name(result, "%s (cont)", a->name);

    设置新张量的操作类型为 GGML_OP_CONT
    result->op   = GGML_OP_CONT;
    根据 is_node 的值决定是否为新张量设置梯度张量
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    设置新张量的源张量为原张量 a
    result->src[0] = a;

    返回新张量 result
    return result;
}

// 连续操作的外部接口函数，返回连续操作后的张量指针
struct ggml_tensor * ggml_cont(
        struct ggml_context * ctx,
        struct ggml_tensor * a) {
    调用 ggml_cont_impl 函数实现连续操作，并返回结果
    return ggml_cont_impl(ctx, a);
}

// 进行连续操作并指定新形状的一维张量
GGML_API struct ggml_tensor * ggml_cont_1d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0) {
    调用 ggml_cont_4d 函数实现连续操作并指定新形状的一维张量，并返回结果
    return ggml_cont_4d(ctx, a, ne0, 1, 1, 1);
}

// 进行连续操作并指定新形状的二维张量
GGML_API struct ggml_tensor * ggml_cont_2d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0,
        int64_t               ne1) {
    调用 ggml_cont_4d 函数实现连续操作并指定新形状的二维张量，并返回结果
    return ggml_cont_4d(ctx, a, ne0, ne1, 1, 1);
}

// 进行连续操作并指定新形状的三维张量
GGML_API struct ggml_tensor * ggml_cont_3d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0,
        int64_t               ne1,
        int64_t               ne2) {
    调用 ggml_cont_4d 函数实现连续操作并指定新形状的三维张量，并返回结果
    return ggml_cont_4d(ctx, a, ne0, ne1, ne2, 1);
}
// 为给定的四维张量创建一个连续的新张量
struct ggml_tensor * ggml_cont_4d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0,
        int64_t               ne1,
        int64_t               ne2,
        int64_t               ne3) {
    // 断言输入张量的元素数量等于 ne0*ne1*ne2*ne3
    GGML_ASSERT(ggml_nelements(a) == (ne0*ne1*ne2*ne3));

    // 初始化一个布尔值为 false
    bool is_node = false;

    // 创建一个新的四维张量，与输入张量 a 具有相同的数据类型和指定的维度
    struct ggml_tensor * result = ggml_new_tensor_4d(ctx, a->type, ne0, ne1, ne2, ne3);
    // 为新张量设置名称
    ggml_format_name(result, "%s (cont)", a->name);

    // 设置新张量的操作类型为 GGML_OP_CONT
    result->op   = GGML_OP_CONT;
    // 根据 is_node 的值决定是否为新张量设置梯度
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 将输入张量 a 设置为新张量的源张量
    result->src[0] = a;

    // 返回新创建的张量
    return result;
}

// ggml_reshape

// 为给定的张量执行重塑操作
struct ggml_tensor * ggml_reshape(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b) {
    // 断言输入张量 a 是连续的
    GGML_ASSERT(ggml_is_contiguous(a));
    // 断言输入张量 a 和 b 的元素数量相等
    GGML_ASSERT(ggml_nelements(a) == ggml_nelements(b));

    // 初始化一个布尔值为 false
    bool is_node = false;

    // 如果输入张量 a 具有梯度，则将 is_node 设置为 true
    if (a->grad) {
        is_node = true;
    }

    // 如果输入张量 b 具有梯度，则抛出异常
    if (b->grad) {
        // gradient propagation is not supported
        //GGML_ASSERT(false);
    }

    // 创建一个新的张量，根据输入张量 a 和 b 的形状确定维度
    struct ggml_tensor * result = ggml_new_tensor_impl(ctx, a->type, GGML_MAX_DIMS, b->ne, a, 0);
    // 为新张量设置名称
    ggml_format_name(result, "%s (reshaped)", a->name);

    // 设置新张量的操作类型为 GGML_OP_RESHAPE
    result->op   = GGML_OP_RESHAPE;
    // 根据 is_node 的值决定是否为新张量设置梯度
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 将输入张量 a 设置为新张量的源张量
    result->src[0] = a;

    // 返回新创建的张量
    return result;
}

// 为给定的一维张量执行重塑操作
struct ggml_tensor * ggml_reshape_1d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0) {
    // 断言输入张量 a 是连续的
    GGML_ASSERT(ggml_is_contiguous(a));
    // 断言输入张量 a 的元素数量等于 ne0
    GGML_ASSERT(ggml_nelements(a) == ne0);

    // 初始化一个布尔值为 false
    bool is_node = false;

    // 如果输入张量 a 具有梯度，则将 is_node 设置为 true
    if (a->grad) {
        is_node = true;
    }

    // 创建一个包含 ne0 个元素的新张量
    const int64_t ne[1] = { ne0 };
    struct ggml_tensor * result = ggml_new_tensor_impl(ctx, a->type, 1, ne, a, 0);
    // 为新张量设置名称
    ggml_format_name(result, "%s (reshaped)", a->name);

    // 设置新张量的操作类型为 GGML_OP_RESHAPE
    // 如果是节点，则复制结果张量，否则为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 将第一个源张量赋值给结果的第一个源张量
    result->src[0] = a;

    // 返回结果
    return result;
}

// 重塑二维张量
struct ggml_tensor * ggml_reshape_2d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0,
        int64_t               ne1) {
    // 断言输入张量是连续的
    GGML_ASSERT(ggml_is_contiguous(a));
    // 断言输入张量元素数量符合要求
    GGML_ASSERT(ggml_nelements(a) == ne0*ne1);

    bool is_node = false;

    // 如果输入张量有梯度信息，则设置为节点
    if (a->grad) {
        is_node = true;
    }

    // 创建新的张量，重塑为二维
    const int64_t ne[2] = { ne0, ne1 };
    struct ggml_tensor * result = ggml_new_tensor_impl(ctx, a->type, 2, ne, a, 0);
    ggml_format_name(result, "%s (reshaped)", a->name);

    result->op   = GGML_OP_RESHAPE;
    // 如果是节点，则复制梯度信息
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;

    return result;
}

// 重塑三维张量
struct ggml_tensor * ggml_reshape_3d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0,
        int64_t               ne1,
        int64_t               ne2) {
    // 断言输入张量是连续的
    GGML_ASSERT(ggml_is_contiguous(a));
    // 断言输入张量元素数量符合要求
    GGML_ASSERT(ggml_nelements(a) == ne0*ne1*ne2);

    bool is_node = false;

    // 如果输入张量有梯度信息，则设置为节点
    if (a->grad) {
        is_node = true;
    }

    // 创建新的张量，重塑为三维
    const int64_t ne[3] = { ne0, ne1, ne2 };
    struct ggml_tensor * result = ggml_new_tensor_impl(ctx, a->type, 3, ne, a, 0);
    ggml_format_name(result, "%s (reshaped)", a->name);

    result->op   = GGML_OP_RESHAPE;
    // 如果是节点，则复制梯度信息
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;

    return result;
}

// 重塑四维张量
struct ggml_tensor * ggml_reshape_4d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0,
        int64_t               ne1,
        int64_t               ne2,
        int64_t               ne3) {
    // 断言输入张量是连续的
    GGML_ASSERT(ggml_is_contiguous(a));
    // 断言输入张量元素数量符合要求
    GGML_ASSERT(ggml_nelements(a) == ne0*ne1*ne2*ne3);

    bool is_node = false;

    // 如果输入张量有梯度信息，则设置为节点
    if (a->grad) {
        is_node = true;
    }

    // 创建新的张量，重塑为四维
    const int64_t ne[4] = { ne0, ne1, ne2, ne3 };
    struct ggml_tensor * result = ggml_new_tensor_impl(ctx, a->type, 4, ne, a, 0);
    # 格式化结果的名称，添加"(reshaped)"后缀
    ggml_format_name(result, "%s (reshaped)", a->name);

    # 设置结果的操作类型为重塑
    result->op   = GGML_OP_RESHAPE;
    
    # 如果是节点，则为结果设置梯度，否则梯度为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    
    # 设置结果的源节点为a
    result->src[0] = a;

    # 返回结果
    return result;
// 定义一个静态函数，用于创建一个新的视图张量
static struct ggml_tensor * ggml_view_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   n_dims,
        const int64_t       * ne,
        size_t                offset) {

    // 初始化一个布尔值，用于表示是否存在梯度
    bool is_node = false;

    // 如果输入张量存在梯度，则将 is_node 设置为 true
    if (a->grad) {
        is_node = true;
    }

    // 创建一个新的张量 result，作为视图张量
    struct ggml_tensor * result = ggml_new_tensor_impl(ctx, a->type, n_dims, ne, a, offset);
    // 格式化结果张量的名称
    ggml_format_name(result, "%s (view)");
    // 设置操作参数
    ggml_set_op_params(result, &offset, sizeof(offset));

    // 设置结果张量的操作类型为 VIEW
    result->op   = GGML_OP_VIEW;
    // 如果存在梯度，则复制结果张量作为梯度张量，否则设置为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为输入张量 a
    result->src[0] = a;

    // 返回创建的视图张量
    return result;
}

// 定义一个函数，用于创建一个一维视图张量
struct ggml_tensor * ggml_view_1d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0,
        size_t                offset) {

    // 创建一个一维视图张量，调用 ggml_view_impl 函数
    struct ggml_tensor * result = ggml_view_impl(ctx, a, 1, &ne0, offset);

    // 返回创建的一维视图张量
    return result;
}

// 定义一个函数，用于创建一个二维视图张量
struct ggml_tensor * ggml_view_2d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0,
        int64_t               ne1,
        size_t                nb1,
        size_t                offset) {

    // 定义一个包含两个维度大小的数组 ne
    const int64_t ne[2] = { ne0, ne1 };

    // 创建一个二维视图张量，调用 ggml_view_impl 函数
    struct ggml_tensor * result = ggml_view_impl(ctx, a, 2, ne, offset);

    // 设置结果张量的第二个维度大小为 nb1
    result->nb[1] = nb1;
    // 计算结果张量的第三个维度大小
    result->nb[2] = result->nb[1]*ne1;
    // 设置结果张量的第四个维度大小
    result->nb[3] = result->nb[2];

    // 返回创建的二维视图张量
    return result;
}

// 定义一个函数，用于创建一个三维视图张量
struct ggml_tensor * ggml_view_3d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0,
        int64_t               ne1,
        int64_t               ne2,
        size_t                nb1,
        size_t                nb2,
        size_t                offset) {

    // 定义一个包含三个维度大小的数组 ne
    const int64_t ne[3] = { ne0, ne1, ne2 };

    // 创建一个三维视图张量，调用 ggml_view_impl 函数
    struct ggml_tensor * result = ggml_view_impl(ctx, a, 3, ne, offset);

    // 设置结果张量的第二个维度大小为 nb1
    result->nb[1] = nb1;
    // 设置结果张量的第三个维度大小为 nb2
    result->nb[2] = nb2;
    // 计算结果张量的第四个维度大小
    result->nb[3] = result->nb[2]*ne2;

    // 返回创建的三维视图张量
    return result;
}
    # 返回result变量的值
    return result;
// 定义一个名为 ggml_view_4d 的函数，用于创建一个四维视图张量
struct ggml_tensor * ggml_view_4d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0,
        int64_t               ne1,
        int64_t               ne2,
        int64_t               ne3,
        size_t                nb1,
        size_t                nb2,
        size_t                nb3,
        size_t                offset) {

    // 将四个维度的大小存储在数组 ne 中
    const int64_t ne[4] = { ne0, ne1, ne2, ne3 };

    // 调用 ggml_view_impl 函数创建一个视图张量
    struct ggml_tensor * result = ggml_view_impl(ctx, a, 4, ne, offset);

    // 设置视图张量的第二、第三、第四维度的大小
    result->nb[1] = nb1;
    result->nb[2] = nb2;
    result->nb[3] = nb3;

    // 返回创建的视图张量
    return result;
}

// 定义一个名为 ggml_permute 的函数，用于对张量进行轴置换
struct ggml_tensor * ggml_permute(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   axis0,
        int                   axis1,
        int                   axis2,
        int                   axis3) {
    // 对轴的索引进行范围检查
    GGML_ASSERT(axis0 >= 0 && axis0 < GGML_MAX_DIMS);
    GGML_ASSERT(axis1 >= 0 && axis1 < GGML_MAX_DIMS);
    GGML_ASSERT(axis2 >= 0 && axis2 < GGML_MAX_DIMS);
    GGML_ASSERT(axis3 >= 0 && axis3 < GGML_MAX_DIMS);

    // 确保轴之间没有重复
    GGML_ASSERT(axis0 != axis1);
    GGML_ASSERT(axis0 != axis2);
    GGML_ASSERT(axis0 != axis3);
    GGML_ASSERT(axis1 != axis2);
    GGML_ASSERT(axis1 != axis3);
    GGML_ASSERT(axis2 != axis3);

    // 初始化一个布尔变量 is_node，并根据输入张量是否有梯度来设置其值
    bool is_node = false;

    if (a->grad) {
        is_node = true;
    }

    // 创建一个新的张量 result，作为输入张量的视图
    struct ggml_tensor * result = ggml_view_tensor(ctx, a);
    // 格式化新张量的名称
    ggml_format_name(result, "%s (permuted)", a->name);

    // 初始化 ne 和 nb 数组，用于存储轴置换后的维度大小和步长
    int ne[GGML_MAX_DIMS];
    int nb[GGML_MAX_DIMS];

    // 根据轴置换的顺序重新排列维度大小和步长
    ne[axis0] = a->ne[0];
    ne[axis1] = a->ne[1];
    ne[axis2] = a->ne[2];
    ne[axis3] = a->ne[3];

    nb[axis0] = a->nb[0];
    nb[axis1] = a->nb[1];
    nb[axis2] = a->nb[2];
    nb[axis3] = a->nb[3];

    // 更新新张量的维度大小
    result->ne[0] = ne[0];
    result->ne[1] = ne[1];
    result->ne[2] = ne[2];
    result->ne[3] = ne[3];

    // 更新新张量的步长
    result->nb[0] = nb[0];
    result->nb[1] = nb[1];
    result->nb[2] = nb[2];
    result->nb[3] = nb[3];
    // 设置结果节点的操作类型为 GGML_OP_PERMUTE
    result->op   = GGML_OP_PERMUTE;
    // 如果是节点，则为结果节点的梯度分配内存，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果节点的第一个源节点为 a
    result->src[0] = a;

    // 创建包含轴参数的整型数组
    int32_t params[] = { axis0, axis1, axis2, axis3 };
    // 设置结果节点的操作参数为 params 数组，数组长度为 sizeof(params)
    ggml_set_op_params(result, params, sizeof(params));

    // 返回结果节点
    return result;
// ggml_transpose

// 对输入的张量进行转置操作
struct ggml_tensor * ggml_transpose(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    // 初始化节点标志为假
    bool is_node = false;

    // 如果输入张量具有梯度信息，则将节点标志设置为真
    if (a->grad) {
        is_node = true;
    }

    // 创建一个新的张量作为结果，并将其视图设置为输入张量
    struct ggml_tensor * result = ggml_view_tensor(ctx, a);
    // 格式化结果张量的名称为输入张量名称加上"(transposed)"
    ggml_format_name(result, "%s (transposed)", a->name);

    // 调整结果张量的维度信息，实现转置操作
    result->ne[0] = a->ne[1];
    result->ne[1] = a->ne[0];

    result->nb[0] = a->nb[1];
    result->nb[1] = a->nb[0];

    // 设置结果张量的操作类型为转置
    result->op   = GGML_OP_TRANSPOSE;
    // 如果是节点，则复制结果张量作为梯度信息，否则设置为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为输入张量
    result->src[0] = a;

    // 返回结果张量
    return result;
}

// ggml_get_rows

// 从输入张量中获取指定行的数据
struct ggml_tensor * ggml_get_rows(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    // 断言输入张量a的第三维度与输入张量b的第二维度相等
    GGML_ASSERT(a->ne[2] == b->ne[1]);
    // 断言输入张量b的第四维度为1且类型为GGML_TYPE_I32
    GGML_ASSERT(b->ne[3] == 1);
    GGML_ASSERT(b->type == GGML_TYPE_I32);

    // 初始化节点标志为假
    bool is_node = false;

    // 如果输入张量a或b具有梯度信息，则将节点标志设置为真
    if (a->grad || b->grad) {
        is_node = true;
    }

    // TODO: 实现非F32返回类型
    // 根据输入张量a的类型确定结果张量的类型
    enum ggml_type type = GGML_TYPE_F32;
    if (a->type == GGML_TYPE_I32) {
        type = a->type;
    }
    // 创建一个新的四维张量作为结果
    struct ggml_tensor * result = ggml_new_tensor_4d(ctx, type, a->ne[0], b->ne[0], b->ne[1], b->ne[2]);

    // 设置结果张量的操作类型为获取行
    result->op   = GGML_OP_GET_ROWS;
    // 如果是节点，则复制结果张量作为梯度信息，否则设置为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为输入张量a和b
    result->src[0] = a;
    result->src[1] = b;

    // 返回结果张量
    return result;
}

// ggml_get_rows_back

// 从输入张量a和b中获取指定行的数据，并将结果存储在张量c中
struct ggml_tensor * ggml_get_rows_back(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        struct ggml_tensor  * c) {
    // 断言输入张量a为矩阵，输入张量b为向量，且类型为GGML_TYPE_I32
    GGML_ASSERT(ggml_is_matrix(a) && ggml_is_vector(b) && b->type == GGML_TYPE_I32);
    // 断言输入张量c为矩阵，且与输入张量a的第一维度相等
    GGML_ASSERT(ggml_is_matrix(c) && (a->ne[0] == c->ne[0]));

    // 初始化节点标志为假
    bool is_node = false;

    // 如果输入张量a或b具有梯度信息，则将节点标志设置为真
    if (a->grad || b->grad) {
        is_node = true;
    }

    // TODO: 实现非F32返回类型
    // 创建一个新的二维张量作为结果
    //struct ggml_tensor * result = ggml_new_tensor_2d(ctx, a->type, a->ne[0], b->ne[0]);
    // 创建一个新的二维浮点型张量，大小为 c->ne[0] 行 c->ne[1] 列
    struct ggml_tensor * result = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, c->ne[0], c->ne[1]);

    // 设置结果张量的操作类型为获取行反向传播
    result->op   = GGML_OP_GET_ROWS_BACK;
    
    // 如果是节点，则将结果张量的梯度设置为结果张量的副本，否则设置为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    
    // 设置结果张量的第一个源张量为 a
    result->src[0] = a;
    
    // 设置结果张量的第二个源张量为 b
    result->src[1] = b;

    // 返回结果张量
    return result;
// ggml_diag

// 创建一个对角线张量，将输入张量的第二个维度扩展为与第一个维度相同的大小
struct ggml_tensor * ggml_diag(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    // 断言输入张量的第二个维度大小为1
    GGML_ASSERT(a->ne[1] == 1);
    bool is_node = false;

    // 如果输入张量具有梯度信息，则设置is_node为true
    if (a->grad) {
        is_node = true;
    }

    // 创建一个新的张量，其维度为输入张量的第一个维度大小，第二个维度大小与第一个相同，其余维度与输入张量相同
    const int64_t ne[4] = { a->ne[0], a->ne[0], a->ne[2], a->ne[3] };
    struct ggml_tensor * result = ggml_new_tensor(ctx, a->type, 4, ne);

    // 设置结果张量的操作类型为GGML_OP_DIAG
    result->op   = GGML_OP_DIAG;
    // 如果is_node为true，则将结果张量的梯度设置为结果张量的副本，否则设置为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为输入张量
    result->src[0] = a;

    return result;
}

// ggml_diag_mask_inf

// 创建一个对角线掩码张量，将输入张量的对角线元素替换为正无穷
static struct ggml_tensor * ggml_diag_mask_inf_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   n_past,
        bool                  inplace) {
    bool is_node = false;

    // 如果输入张量具有梯度信息，则设置is_node为true
    if (a->grad) {
        is_node = true;
    }

    // 根据inplace参数决定是否创建一个新的张量或者使用输入张量的视图
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置操作参数为n_past
    int32_t params[] = { n_past };
    ggml_set_op_params(result, params, sizeof(params));

    // 设置结果张量的操作类型为GGML_OP_DIAG_MASK_INF
    result->op   = GGML_OP_DIAG_MASK_INF;
    // 如果is_node为true，则将结果张量的梯度设置为结果张量的副本，否则设置为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为输入张量
    result->src[0] = a;

    return result;
}

// 创建一个对角线掩码张量，将输入张量的对角线元素替换为正无穷
struct ggml_tensor * ggml_diag_mask_inf(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   n_past) {
    return ggml_diag_mask_inf_impl(ctx, a, n_past, false);
}

// 创建一个对角线掩码张量，将输入张量的对角线元素替换为正无穷（原地操作）
struct ggml_tensor * ggml_diag_mask_inf_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   n_past) {
    return ggml_diag_mask_inf_impl(ctx, a, n_past, true);
}

// ggml_diag_mask_zero

// 创建一个对角线掩码张量，将输入张量的对角线元素替换为0
static struct ggml_tensor * ggml_diag_mask_zero_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   n_past,
        bool                  inplace) {
    bool is_node = false;

    // 如果输入张量具有梯度信息，则设置is_node为true
    if (a->grad) {
        is_node = true;
    }

    // 根据inplace参数决定是否创建一个新的张量或者使用输入张量的视图
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);
    # 定义一个包含 n_past 的整型数组 params
    int32_t params[] = { n_past };
    # 设置操作的参数为 params，params 的大小为 sizeof(params)
    ggml_set_op_params(result, params, sizeof(params));

    # 设置操作类型为 GGML_OP_DIAG_MASK_ZERO
    result->op   = GGML_OP_DIAG_MASK_ZERO;
    # 如果是节点，则为梯度赋值为 result，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    # 设置源节点为 a
    result->src[0] = a;

    # 返回 result
    return result;
// 返回一个对角线上的元素为零的张量，通过调用 ggml_diag_mask_zero_impl 函数实现
struct ggml_tensor * ggml_diag_mask_zero(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   n_past) {
    return ggml_diag_mask_zero_impl(ctx, a, n_past, false);
}

// 返回一个对角线上的元素为零的张量，通过调用 ggml_diag_mask_zero_impl 函数实现，支持原地操作
struct ggml_tensor * ggml_diag_mask_zero_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   n_past) {
    return ggml_diag_mask_zero_impl(ctx, a, n_past, true);
}

// 实现 Softmax 操作的内部函数
static struct ggml_tensor * ggml_soft_max_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * mask,
        float                 scale,
        bool                  inplace) {
    GGML_ASSERT(ggml_is_contiguous(a)); // 断言张量 a 是连续的
    if (mask) {
        GGML_ASSERT(ggml_is_contiguous(mask)); // 断言张量 mask 是连续的
        GGML_ASSERT(mask->ne[2] == 1); // 断言张量 mask 的第三维大小为 1
        GGML_ASSERT(mask->ne[3] == 1); // 断言张量 mask 的第四维大小为 1
        GGML_ASSERT(ggml_can_repeat_rows(mask, a)); // 断言 mask 可以重复行与 a 的行数相匹配
    }

    bool is_node = false;

    if (a->grad) {
        is_node = true;
    }

    // 创建结果张量，根据 inplace 参数决定是创建视图还是复制张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    float params[] = { scale };
    ggml_set_op_params(result, params, sizeof(params)); // 设置操作参数

    result->op   = GGML_OP_SOFT_MAX; // 设置操作类型为 Softmax
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL; // 如果是节点，则复制结果张量作为梯度张量，否则为 NULL
    result->src[0] = a; // 设置源张量为 a
    result->src[1] = mask; // 设置掩码张量为 mask

    return result; // 返回结果张量
}

// 对输入张量进行 Softmax 操作，调用 ggml_soft_max_impl 函数实现
struct ggml_tensor * ggml_soft_max(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_soft_max_impl(ctx, a, NULL, 1.0f, false);
}

// 对输入张量进行 Softmax 操作，支持原地操作，调用 ggml_soft_max_impl 函数实现
struct ggml_tensor * ggml_soft_max_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_soft_max_impl(ctx, a, NULL, 1.0f, true);
}

// 对输入张量进行 Softmax 操作，支持扩展参数 mask 和 scale，调用 ggml_soft_max_impl 函数实现
struct ggml_tensor * ggml_soft_max_ext(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * mask,
        float                 scale) {
    return ggml_soft_max_impl(ctx, a, mask, scale, false);
}

// ggml_soft_max_back
// 实现 Softmax 反向传播的函数
static struct ggml_tensor * ggml_soft_max_back_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        bool                  inplace) {
    // 初始化节点标志为假
    bool is_node = false;

    // 如果输入张量 a 或 b 具有梯度信息
    if (a->grad || b->grad) {
        // 设置节点标志为真，表示需要实现反向传播
        is_node = true; // TODO : implement backward pass
    }

    // 根据 inplace 参数选择复制张量或者视图张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置结果张量的操作类型为 GGML_OP_SOFT_MAX_BACK
    result->op   = GGML_OP_SOFT_MAX_BACK;
    // 如果是节点，则复制结果张量作为梯度张量，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为输入张量 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    // 返回结果张量
    return result;
}

// 调用 Softmax 反向传播函数，不进行原地操作
struct ggml_tensor * ggml_soft_max_back(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    return ggml_soft_max_back_impl(ctx, a, b, false);
}

// 调用 Softmax 反向传播函数，进行原地操作
struct ggml_tensor * ggml_soft_max_back_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    return ggml_soft_max_back_impl(ctx, a, b, true);
}

// ggml_rope

// 实现 Rope 函数
static struct ggml_tensor * ggml_rope_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                   n_dims,
        int                   mode,
        int                   n_ctx,
        int                   n_orig_ctx,
        float                 freq_base,
        float                 freq_scale,
        float                 ext_factor,
        float                 attn_factor,
        float                 beta_fast,
        float                 beta_slow,
        float                 xpos_base,
        bool                  xpos_down,
        bool                  inplace) {
    // 断言张量 b 是向量
    GGML_ASSERT(ggml_is_vector(b));
    // 断言张量 b 的类型为 GGML_TYPE_I32
    GGML_ASSERT(b->type == GGML_TYPE_I32);
    // 断言张量 a 的第三维度与张量 b 的第一维度相等
    GGML_ASSERT(a->ne[2] == b->ne[0]);

    // 初始化节点标志为假
    bool is_node = false;

    // 如果输入张量 a 具有梯度信息
    if (a->grad) {
        // 设置节点标志为真
        is_node = true;
    }

    // 根据 inplace 参数选择复制张量或者视图张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);
    // 定义一个包含13个元素的整型数组params，初始化前5个元素，分别为n_past、n_dims、mode、n_ctx、n_orig_ctx
    int32_t params[13] = { /*n_past*/ 0, n_dims, mode, n_ctx, n_orig_ctx };
    // 将freq_base的值拷贝到params数组的第5个元素位置
    memcpy(params +  5, &freq_base,    sizeof(float));
    // 将freq_scale的值拷贝到params数组的第6个元素位置
    memcpy(params +  6, &freq_scale,   sizeof(float));
    // 将ext_factor的值拷贝到params数组的第7个元素位置
    memcpy(params +  7, &ext_factor,   sizeof(float));
    // 将attn_factor的值拷贝到params数组的第8个元素位置
    memcpy(params +  8, &attn_factor,  sizeof(float));
    // 将beta_fast的值拷贝到params数组的第9个元素位置
    memcpy(params +  9, &beta_fast,    sizeof(float));
    // 将beta_slow的值拷贝到params数组的第10个元素位置
    memcpy(params + 10, &beta_slow,    sizeof(float));
    // 将xpos_base的值拷贝到params数组的第11个元素位置
    memcpy(params + 11, &xpos_base,    sizeof(float));
    // 将xpos_down的值拷贝到params数组的第12个元素位置
    memcpy(params + 12, &xpos_down,    sizeof(bool));
    // 将params数组传递给ggml_set_op_params函数，设置操作参数
    ggml_set_op_params(result, params, sizeof(params));

    // 设置result的操作类型为GGML_OP_ROPE
    result->op   = GGML_OP_ROPE;
    // 如果is_node为真，则将result的梯度设置为ctx的副本，否则设置为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置result的第一个源操作数为a
    result->src[0] = a;
    // 设置result的第二个源操作数为b
    result->src[1] = b;

    // 返回result
    return result;
}

# 使用 ggml_rope_impl 函数计算两个张量的 ROPE 操作结果
struct ggml_tensor * ggml_rope(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                   n_dims,
        int                   mode,
        int                   n_ctx) {
    # 调用 ggml_rope_impl 函数，返回 ROPE 操作结果
    return ggml_rope_impl(
        ctx, a, b, n_dims, mode, n_ctx, 0, 10000.0f, 1.0f, 0.0f, 1.0f, 0.0f, 0.0f, 0.0f, false, false
    );
}

# 使用 ggml_rope_impl 函数计算两个张量的 ROPE 操作结果（原地操作）
struct ggml_tensor * ggml_rope_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                   n_dims,
        int                   mode,
        int                   n_ctx) {
    # 调用 ggml_rope_impl 函数，返回 ROPE 操作结果（原地操作）
    return ggml_rope_impl(
        ctx, a, b, n_dims, mode, n_ctx, 0, 10000.0f, 1.0f, 0.0f, 1.0f, 0.0f, 0.0f, 0.0f, false, true
    );
}

# 使用 ggml_rope_impl 函数计算两个张量的自定义 ROPE 操作结果
struct ggml_tensor * ggml_rope_custom(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                   n_dims,
        int                   mode,
        int                   n_ctx,
        int                   n_orig_ctx,
        float                 freq_base,
        float                 freq_scale,
        float                 ext_factor,
        float                 attn_factor,
        float                 beta_fast,
        float                 beta_slow) {
    # 调用 ggml_rope_impl 函数，返回自定义 ROPE 操作结果
    return ggml_rope_impl(
        ctx, a, b, n_dims, mode, n_ctx, n_orig_ctx, freq_base, freq_scale,
        ext_factor, attn_factor, beta_fast, beta_slow, 0.0f, false, false
    );
}
// 在原地执行自定义的绳索操作，返回处理后的张量
struct ggml_tensor * ggml_rope_custom_inplace(
        struct ggml_context * ctx,  // 上下文对象
        struct ggml_tensor  * a,    // 输入张量 a
        struct ggml_tensor  * b,    // 输入张量 b
        int                   n_dims,  // 张量维度
        int                   mode,     // 模式
        int                   n_ctx,    // 上下文数量
        int                   n_orig_ctx,  // 原始上下文数量
        float                 freq_base,   // 频率基数
        float                 freq_scale,  // 频率比例
        float                 ext_factor,  // 扩展因子
        float                 attn_factor,  // 注意力因子
        float                 beta_fast,    // 快速 beta
        float                 beta_slow) {  // 慢速 beta
    // 调用 ggml_rope_impl 函数执行绳索操作
    return ggml_rope_impl(
        ctx, a, b, n_dims, mode, n_ctx, n_orig_ctx, freq_base, freq_scale,
        ext_factor, attn_factor, beta_fast, beta_slow, 0.0f, false, true
    );
}

// 在原地执行 xpos 操作，返回处理后的张量
struct ggml_tensor * ggml_rope_xpos_inplace(
        struct ggml_context * ctx,  // 上下文对象
        struct ggml_tensor  * a,    // 输入张量 a
        struct ggml_tensor  * b,    // 输入张量 b
        int                   n_dims,  // 张量维度
        float                 base,     // 基数
        bool                  down) {   // 是否向下
    // 调用 ggml_rope_impl 函数执行 xpos 操作
    return ggml_rope_impl(ctx, a, b, n_dims, 0, 0, 0, 10000.0f, 1.0f, 0.0f, 1.0f, 0.0f, 0.0f, base, down, true);
}

// ggml_rope_back

// 执行绳索反向操作，返回处理后的张量
struct ggml_tensor * ggml_rope_back(
        struct ggml_context * ctx,  // 上下文对象
        struct ggml_tensor  * a,    // 输入张量 a
        struct ggml_tensor  * b,    // 输入张量 b
        int                   n_dims,  // 张量维度
        int                   mode,     // 模式
        int                   n_ctx,    // 上下文数量
        int                   n_orig_ctx,  // 原始上下文数量
        float                 freq_base,   // 频率基数
        float                 freq_scale,  // 频率比例
        float                 ext_factor,  // 扩展因子
        float                 attn_factor,  // 注意力因子
        float                 beta_fast,    // 快速 beta
        float                 beta_slow,    // 慢速 beta
        float                 xpos_base,    // xpos 基数
        bool                  xpos_down) {  // xpos 是否向下
    // 断言 b 是向量
    GGML_ASSERT(ggml_is_vector(b));
    // 断言 b 的类型是 GGML_TYPE_I32
    GGML_ASSERT(b->type == GGML_TYPE_I32);
    // 断言 a 的第三个维度等于 b 的第一个维度
    GGML_ASSERT(a->ne[2] == b->ne[0]);

    // 断言 mode 的第 3 位是 0，如果不是则输出错误信息
    GGML_ASSERT((mode & 4) == 0 && "ggml_rope_back() for ChatGLM not implemented yet");

    // 初始化 is_node 为 false
    bool is_node = false;
}
    // 如果输入节点 a 有梯度信息，则将 is_node 设为 false，表示需要实现反向传播
    if (a->grad) {
        is_node = false; // TODO: implement backward
    }

    // 复制输入节点 a，得到一个新的 tensor 结构 result
    struct ggml_tensor * result = ggml_dup_tensor(ctx, a);

    // 定义包含参数的数组 params，共 13 个元素
    int32_t params[13] = { /*n_past*/ 0, n_dims, mode, n_ctx, n_orig_ctx };
    // 将特定参数值拷贝到 params 数组中对应位置
    memcpy(params +  5, &freq_base,    sizeof(float));
    memcpy(params +  6, &freq_scale,   sizeof(float));
    memcpy(params +  7, &ext_factor,   sizeof(float));
    memcpy(params +  8, &attn_factor,  sizeof(float));
    memcpy(params +  9, &beta_fast,    sizeof(float));
    memcpy(params + 10, &beta_slow,    sizeof(float));
    memcpy(params + 11, &xpos_base,    sizeof(float));
    memcpy(params + 12, &xpos_down,    sizeof(bool));
    // 将参数数组 params 设置到 result 的操作参数中
    ggml_set_op_params(result, params, sizeof(params));

    // 设置 result 的操作类型为 GGML_OP_ROPE_BACK
    result->op   = GGML_OP_ROPE_BACK;
    // 如果 is_node 为 true，则将 result 的梯度设置为复制的 result，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置 result 的源节点为 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    // 返回 result 结构
    return result;
// ggml_alibi

// 实现 ALIBI 操作，返回一个新的张量
struct ggml_tensor * ggml_alibi(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   n_past,
        int                   n_head,
        float                 bias_max) {
    // 断言 n_past 大于等于 0
    GGML_ASSERT(n_past >= 0);
    bool is_node = false;

    // 如果输入张量有梯度信息
    if (a->grad) {
        // 断言为假，表示暂未实现反向传播
        GGML_ASSERT(false); // TODO: implement backward
        is_node = true;
    }

    // 创建一个新的张量，与输入张量共享数据
    struct ggml_tensor * result = ggml_view_tensor(ctx, a);

    // 设置操作参数，包括 n_past、n_head 和 bias_max
    int32_t op_params[3] = { n_past, n_head };
    memcpy(op_params + 2, &bias_max, sizeof(float));
    ggml_set_op_params(result, op_params, sizeof(op_params));

    // 设置操作类型为 ALIBI
    result->op   = GGML_OP_ALIBI;
    // 如果是节点，为结果张量创建一个梯度张量
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;

    return result;
}

// ggml_clamp

// 实现 CLAMP 操作，返回一个新的张量
struct ggml_tensor * ggml_clamp(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        float                 min,
        float                 max) {
    bool is_node = false;

    // 如果输入张量有梯度信息
    if (a->grad) {
        // 断言为假，表示暂未实现反向传播
        GGML_ASSERT(false); // TODO: implement backward
        is_node = true;
    }

    // 创建一个新的张量，与输入张量共享数据
    struct ggml_tensor * result = ggml_view_tensor(ctx, a);

    // 设置操作参数，包括最小值和最大值
    float params[] = { min, max };
    ggml_set_op_params(result, params, sizeof(params));

    // 设置操作类型为 CLAMP
    result->op   = GGML_OP_CLAMP;
    // 如果是节点，为结果张量创建一个梯度张量
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;

    return result;
}

// ggml_conv_1d

// 计算卷积输出大小的函数
static int64_t ggml_calc_conv_output_size(int64_t ins, int64_t ks, int s, int p, int d) {
    return (ins + 2 * p - d * (ks - 1) - 1) / s + 1;
}

// 实现 1D 卷积操作，返回一个新的张量
GGML_API struct ggml_tensor * ggml_conv_1d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                   s0,
        int                   p0,
        int                   d0) {
    // 使用 ggml_im2col 函数将输入张量 a 转换为 im2col 张量，返回结果为 [N, OL, IC * K]
    struct ggml_tensor * im2col = ggml_im2col(ctx, a, b, s0, 0, p0, 0, d0, 0, false);

    // 使用 ggml_mul_mat 函数进行矩阵乘法运算，将两个二维张量相乘
    // 第一个参数为 im2col 张量的二维重塑结果，形状为 [N*OL, IC * K]
    // 第二个参数为输入张量 a 的二维重塑结果，形状为 [OC，IC, K]
    struct ggml_tensor * result =
        ggml_mul_mat(ctx,
                ggml_reshape_2d(ctx, im2col, im2col->ne[0], (im2col->ne[2] * im2col->ne[1])), // [N, OL, IC * K] => [N*OL, IC * K]
                ggml_reshape_2d(ctx, a, (a->ne[0] * a->ne[1]), a->ne[2]));                    // [OC，IC, K] => [OC, IC * K]

    // 使用 ggml_reshape_3d 函数将结果张量 result 重塑为三维张量，形状为 [N, OC, OL]
    result = ggml_reshape_3d(ctx, result, im2col->ne[1], a->ne[2], im2col->ne[2]);

    // 返回重塑后的结果张量
    return result;
// 结构体 ggml_conv_1d_ph，用于进行一维卷积操作
struct ggml_tensor* ggml_conv_1d_ph(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                   s,
        int                   d) {
    // 调用 ggml_conv_1d 函数进行一维卷积操作，返回结果
    return ggml_conv_1d(ctx, a, b, s, a->ne[0] / 2, d);
}

// 静态函数 ggml_calc_conv_transpose_1d_output_size，计算一维卷积转置操作的输出大小
static int64_t ggml_calc_conv_transpose_1d_output_size(int64_t ins, int64_t ks, int s, int p, int d) {
    // 计算一维卷积转置操作的输出大小并返回
    return (ins - 1) * s - 2 * p + d * (ks - 1) + 1;
}

// 函数 ggml_conv_transpose_1d，用于进行一维卷积转置操作
GGML_API struct ggml_tensor * ggml_conv_transpose_1d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                   s0,
        int                   p0,
        int                   d0) {
    // 断言 b 是矩阵类型
    GGML_ASSERT(ggml_is_matrix(b));
    // 断言 a 的第三维度与 b 的第二维度相等
    GGML_ASSERT(a->ne[2] == b->ne[1]);
    // 断言 a 的第四维度为 1
    GGML_ASSERT(a->ne[3] == 1);

    // 断言 p0 为 0
    GGML_ASSERT(p0 == 0);
    // 断言 d0 为 1
    GGML_ASSERT(d0 == 1);

    // 初始化 is_node 为 false
    bool is_node = false;

    // 如果 a 或 b 具有梯度信息
    if (a->grad || b->grad) {
        // 断言为假，提示需要实现反向传播
        GGML_ASSERT(false); // TODO: implement backward
        // 将 is_node 设置为 true
        is_node = true;
    }

    // 计算新的张量维度信息 ne
    const int64_t ne[4] = {
        ggml_calc_conv_transpose_1d_output_size(b->ne[0], a->ne[0], s0, 0 /*p0*/, 1 /*d0*/),
        a->ne[1], b->ne[2], 1,
    };
    // 创建新的张量 result
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F32, 4, ne);

    // 设置操作参数为 s0, p0, d0
    int32_t params[] = { s0, p0, d0 };
    ggml_set_op_params(result, params, sizeof(params));

    // 设置操作类型为 GGML_OP_CONV_TRANSPOSE_1D
    result->op = GGML_OP_CONV_TRANSPOSE_1D;
    // 如果 is_node 为 true，则为 result 创建梯度张量
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置 result 的源张量为 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    // 返回结果张量 result
    return result;
}

// 函数 ggml_conv_depthwise_2d，用于进行深度可分离二维卷积操作
struct ggml_tensor * ggml_conv_depthwise_2d(
    struct ggml_context * ctx,
    struct ggml_tensor * a,
    struct ggml_tensor * b,
    int                  s0,
    int                  s1,
    int                  p0,
    int                  p1,
    int                  d0,
    int                  d1) {
    // 通过 ggml_reshape_4d 函数对输入张量 a 进行重塑
    struct ggml_tensor * new_a = ggml_reshape_4d(ctx, a, a->ne[0], a->ne[1], 1, a->ne[2] * a->ne[3]);
}
    // 使用ggml_im2col函数将输入张量new_a转换为im2col张量，参数包括上下文ctx、输入张量new_a、重塑后的张量b、步长s0、s1、填充p0、p1、膨胀d0、d1以及布尔值true
    struct ggml_tensor * im2col = ggml_im2col(ctx, new_a,
                                        ggml_reshape_4d(ctx, b, b->ne[0], b->ne[1], 1, b->ne[2] * b->ne[3]),
                                        s0, s1, p0, p1, d0, d1, true); // [N * IC, OH, OW, KH * KW]

    // 使用ggml_mul_mat函数进行矩阵乘法运算，参数包括上下文ctx、重塑后的张量new_a、重塑后的im2col张量，得到结果张量result
    struct ggml_tensor * result =
        ggml_mul_mat(ctx,
                ggml_reshape_4d(ctx, new_a, (new_a->ne[0] * new_a->ne[1]), new_a->ne[2],  new_a->ne[3], 1),                       // [OC，1, KH, KW] => [1, OC, 1, KH * KW]
                ggml_reshape_4d(ctx, im2col, im2col->ne[0], im2col->ne[2] * im2col->ne[1], b->ne[2], b->ne[3]); // [N * IC, OH, OW, KH * KW] => [N, IC, OH * OW, KH * KW]

    // 重塑result张量的维度，得到最终结果张量，维度为[N, OC, OH, OW]
    result = ggml_reshape_4d(ctx, result, im2col->ne[1], im2col->ne[2], b->ne[2], b->ne[3]); // [N, OC, OH, OW]

    // 返回最终结果张量
    return result;
// ggml_conv_2d

// im2col: [N, IC, IH, IW] => [N, OH, OW, IC*KH*KW]
// a: [OC，IC, KH, KW]
// b: [N, IC, IH, IW]
// result: [N, OH, OW, IC*KH*KW]
struct ggml_tensor * ggml_im2col(
    struct ggml_context * ctx,
    struct ggml_tensor  * a,
    struct ggml_tensor  * b,
    int                  s0,
    int                  s1,
    int                  p0,
    int                  p1,
    int                  d0,
    int                  d1,
    bool                 is_2D) {

    // 如果是二维卷积，检查输入张量 a 和 b 的第三维是否相等
    if(is_2D) {
        GGML_ASSERT(a->ne[2] == b->ne[2]);
    } else {
        GGML_ASSERT(a->ne[1] == b->ne[1]);
    }
    bool is_node = false;

    // 如果输入张量 a 或 b 需要梯度计算
    if (a->grad || b->grad) {
        GGML_ASSERT(false); // TODO: implement backward
        is_node = true;
    }

    // 计算输出张量的高度 OH 和宽度 OW
    const int64_t OH = is_2D ? ggml_calc_conv_output_size(b->ne[1], a->ne[1], s1, p1, d1) : 0;
    const int64_t OW =         ggml_calc_conv_output_size(b->ne[0], a->ne[0], s0, p0, d0);

    // 定义输出张量的维度 ne
    const int64_t ne[4] = {
        is_2D ? (a->ne[2] * a->ne[1] * a->ne[0]) : a->ne[1] * a->ne[0],
        OW,
        is_2D ? OH : b->ne[2],
        is_2D ?      b->ne[3] : 1,
    };

    // 创建新的输出张量 result，并设置操作参数
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F16, 4, ne);
    int32_t params[] = { s0, s1, p0, p1, d0, d1, (is_2D ? 1 : 0) };
    ggml_set_op_params(result, params, sizeof(params));

    result->op = GGML_OP_IM2COL;
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;
    result->src[1] = b;

    return result;
}

// a: [OC，IC, KH, KW]
// b: [N, IC, IH, IW]
// result: [N, OC, OH, OW]
struct ggml_tensor * ggml_conv_2d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                  s0,
        int                  s1,
        int                  p0,
        int                  p1,
        int                  d0,
        int                  d1) {
    // 使用 ggml_im2col 函数将输入张量 a 转换为 im2col 张量，参数包括输入张量 a、卷积核张量 b、步长 s0、s1、填充 p0、p1、膨胀系数 d0、d1，返回的张量形状为 [N, OH, OW, IC * KH * KW]
    struct ggml_tensor * im2col = ggml_im2col(ctx, a, b, s0, s1, p0, p1, d0, d1, true);

    // 使用 ggml_mul_mat 函数进行矩阵乘法运算，参数包括上一步得到的 im2col 张量和经过 reshape 后的输入张量 a，返回的结果张量形状为 [N, OC, OH, OW]
    struct ggml_tensor * result =
        ggml_mul_mat(ctx,
                ggml_reshape_2d(ctx, im2col, im2col->ne[0],  im2col->ne[3] * im2col->ne[2] * im2col->ne[1]), // 将 im2col 张量 reshape 成 [N*OH*OW, IC * KH * KW]
                ggml_reshape_2d(ctx, a, (a->ne[0] * a->ne[1] * a->ne[2]),  a->ne[3]));                       // 将输入张量 a reshape 成 [OC, IC * KH * KW]

    // 将上一步得到的结果张量 result 进行 reshape，形状变为 [N, OC, OH, OW]
    result = ggml_reshape_4d(ctx, result, im2col->ne[1], im2col->ne[2], a->ne[3], im2col->ne[3]);

    // 返回最终结果张量
    return result;
// 结构体 ggml_conv_2d_sk_p0，实现卷积操作，返回卷积结果
struct ggml_tensor * ggml_conv_2d_sk_p0(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    // 调用 ggml_conv_2d 函数进行卷积计算，传入参数为输入张量 a、卷积核张量 b，以及输入张量的宽度和高度，步长为 1
    return ggml_conv_2d(ctx, a, b, a->ne[0], a->ne[1], 0, 0, 1, 1);
}

// 结构体 ggml_conv_2d_s1_ph，实现卷积操作，返回卷积结果
struct ggml_tensor * ggml_conv_2d_s1_ph(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    // 调用 ggml_conv_2d 函数进行卷积计算，传入参数为输入张量 a、卷积核张量 b，以及卷积核在水平和垂直方向的填充值，步长为 1
    return ggml_conv_2d(ctx, a, b, 1, 1, a->ne[0] / 2, a->ne[1] / 2, 1, 1);
}

// 静态函数 ggml_calc_conv_transpose_output_size，计算转置卷积输出大小
static int64_t ggml_calc_conv_transpose_output_size(int64_t ins, int64_t ks, int s, int p) {
    // 根据输入大小、卷积核大小、步长和填充值计算转置卷积的输出大小
    return (ins - 1) * s - 2 * p + ks;
}

// 结构体 ggml_conv_transpose_2d_p0，实现转置卷积操作，返回转置卷积结果
struct ggml_tensor * ggml_conv_transpose_2d_p0(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                   stride) {
    // 断言输入张量 a 和卷积核张量 b 的深度相同
    GGML_ASSERT(a->ne[3] == b->ne[2]);

    bool is_node = false;

    // 如果输入张量 a 或卷积核张量 b 需要计算梯度
    if (a->grad || b->grad) {
        GGML_ASSERT(false); // TODO: implement backward
        is_node = true;
    }

    // 计算转置卷积操作后的张量大小
    const int64_t ne[4] = {
        ggml_calc_conv_transpose_output_size(b->ne[0], a->ne[0], stride, 0 /*p0*/),
        ggml_calc_conv_transpose_output_size(b->ne[1], a->ne[1], stride, 0 /*p1*/),
        a->ne[2], b->ne[3],
    };

    // 创建新的张量用于存储转置卷积结果
    struct ggml_tensor* result = ggml_new_tensor(ctx, GGML_TYPE_F32, 4, ne);

    // 设置张量的操作参数为步长
    ggml_set_op_params_i32(result, 0, stride);

    // 设置张量的操作类型为转置卷积
    result->op = GGML_OP_CONV_TRANSPOSE_2D;
    // 如果需要计算梯度，复制张量作为梯度张量
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;
    result->src[1] = b;

    return result;
}

// 静态函数 ggml_calc_pool_output_size，计算池化操作输出大小
static int64_t ggml_calc_pool_output_size(int64_t ins, int ks, int s, float p) {
    // 根据输入大小、核大小、步长和填充值计算池化操作的输出大小
    return (ins + 2 * p - ks) / s + 1;
}

// 结构体 ggml_pool_1d，实现一维池化操作，返回池化结果
struct ggml_tensor * ggml_pool_1d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        enum ggml_op_pool     op,
        int                   k0,
        int                   s0,
        int                   p0) {

    bool is_node = false;
    // 如果输入节点的梯度存在，则抛出断言错误，提示需要实现反向传播
    if (a->grad) {
        GGML_ASSERT(false); // TODO: implement backward
        is_node = true;
    }

    // 计算池化操作输出的大小，并存储在数组 ne 中
    const int64_t ne[2] = {
        ggml_calc_pool_output_size(a->ne[0], k0, s0, p0),
        a->ne[1],
    };
    // 创建一个新的浮点型张量 result，包含两个维度，大小由 ne 数组指定
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F32, 2, ne);

    // 设置操作参数为 op、k0、s0、p0
    int32_t params[] = { op, k0, s0, p0 };
    ggml_set_op_params(result, params, sizeof(params));

    // 设置结果张量的操作类型为 1D 池化
    result->op = GGML_OP_POOL_1D;
    // 如果 is_node 为真，则为结果张量设置梯度为自身的副本，否则设置为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源节点为输入节点 a
    result->src[0] = a;

    // 返回结果张量
    return result;
// 结束 ggml_pool_2d 函数定义

// ggml_pool_2d 函数用于对输入的张量进行 2D 池化操作
struct ggml_tensor * ggml_pool_2d(
        struct ggml_context * ctx, // 上下文对象
        struct ggml_tensor  * a,   // 输入张量
        enum ggml_op_pool     op,  // 池化操作类型
        int                   k0,  // 池化核大小的第一个维度
        int                   k1,  // 池化核大小的第二维度
        int                   s0,  // 步幅的第一个维度
        int                   s1,  // 步幅的第二维度
        float                 p0,  // 填充的第一个维度
        float                 p1) { // 填充的第二维度

    bool is_node = false; // 初始化是否为节点的标志为 false

    if (a->grad) { // 如果输入张量存在梯度
        GGML_ASSERT(false); // 断言，提示需要实现反向传播
        is_node = true; // 将是否为节点的标志设置为 true
    }

    // 计算池化后的输出大小
    const int64_t ne[3] = {
        ggml_calc_pool_output_size(a->ne[0], k0, s0, p0),
        ggml_calc_pool_output_size(a->ne[1], k1, s1, p1),
        a->ne[2],
    };
    // 创建新的张量用于存储池化后的结果
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F32, 3, ne);

    // 设置池化操作的参数
    int32_t params[] = { op, k0, k1, s0, s1, p0, p1 };
    ggml_set_op_params(result, params, sizeof(params));

    result->op = GGML_OP_POOL_2D; // 设置操作类型为 2D 池化
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL; // 如果是节点，则复制张量，否则为 NULL
    result->src[0] = a; // 设置源张量为输入张量

    return result; // 返回池化后的结果张量
}

// ggml_upscale_impl 函数用于对输入的张量进行上采样操作
static struct ggml_tensor * ggml_upscale_impl(
    struct ggml_context * ctx, // 上下文对象
    struct ggml_tensor * a,    // 输入张量
    int scale_factor) {        // 上采样因子

    bool is_node = false; // 初始化是否为节点的标志为 false

    if (a->grad) { // 如果输入张量存在梯度
        GGML_ASSERT(false); // 断言，提示需要实现反向传播
        is_node = true; // 将是否为节点的标志设置为 true
    }

    // 创建新的张量用于存储上采样后的结果
    struct ggml_tensor * result = ggml_new_tensor_4d(ctx, a->type,
            a->ne[0] * scale_factor,
            a->ne[1] * scale_factor,
            a->ne[2], a->ne[3]);

    result->op = GGML_OP_UPSCALE; // 设置操作类型为上采样
    result->op_params[0] = scale_factor; // 设置上采样因子
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL; // 如果是节点，则复制张量，否则为 NULL
    result->src[0] = a; // 设置源张量为输入张量

    return result; // 返回上采样后的结果张量
}

// ggml_pad 函数用于对输入的张量进行填充操作
struct ggml_tensor * ggml_pad(
    struct ggml_context * ctx, // 上下文对象
    struct ggml_tensor  * a,   // 输入张量
    int p0, int p1, int p2, int p3) { // 填充的四个维度

    bool is_node = false; // 初始化是否为节点的标志为 false

    if (a->grad) { // 如果输入张量存在梯度
        GGML_ASSERT(false); // 断言，提示需要实现反向传播
        is_node = true; // 将是否为节点的标志设置为 true
    }
    // 创建一个新的四维张量，根据给定的参数和输入张量的属性
    struct ggml_tensor * result = ggml_new_tensor_4d(ctx, a->type,
            a->ne[0] + p0,
            a->ne[1] + p1,
            a->ne[2] + p2,
            a->ne[3] + p3);

    // 设置结果张量的操作类型为填充
    result->op = GGML_OP_PAD;
    // 如果是节点，则复制结果张量作为梯度，否则梯度为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为输入张量a
    result->src[0] = a;

    // 返回创建的结果张量
    return result;
// 结构体 ggml_tensor 指针类型的 upscale 函数，用于对输入的张量进行上采样操作
struct ggml_tensor * ggml_upscale(
    struct ggml_context * ctx,
    struct ggml_tensor * a,
    int scale_factor) {
    // 调用 ggml_upscale_impl 函数实现上采样操作
    return ggml_upscale_impl(ctx, a, scale_factor);
}

// ggml_argsort

// 结构体 ggml_tensor 指针类型的 argsort 函数，用于对输入的张量进行排序操作
struct ggml_tensor * ggml_argsort(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        enum ggml_sort_order  order) {
    // 初始化布尔变量 is_node 为 false
    bool is_node = false;

    // 创建一个新的张量 result，用于存储排序后的结果
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_I32, GGML_MAX_DIMS, a->ne);

    // 设置张量 result 的操作参数为排序顺序 order 的整数值
    ggml_set_op_params_i32(result, 0, (int32_t) order);

    // 设置张量 result 的操作为 ARGSORT，梯度为 is_node 为真时的 result 张量，源张量为 a
    result->op   = GGML_OP_ARGSORT;
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;

    // 返回排序后的结果张量
    return result;
}

// ggml_top_k

// 结构体 ggml_tensor 指针类型的 top_k 函数，用于获取输入张量中的前 k 个最大值
struct ggml_tensor * ggml_top_k(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   k) {
    // 断言输入张量 a 的第一个维度大小大于等于 k
    GGML_ASSERT(a->ne[0] >= k);

    // 调用 ggml_argsort 函数对输入张量 a 进行排序，得到结果张量 result
    struct ggml_tensor * result = ggml_argsort(ctx, a, GGML_SORT_DESC);

    // 对结果张量 result 进行维度变换，保留前 k 个最大值
    result = ggml_view_4d(ctx, result,
                k, result->ne[1], result->ne[2], result->ne[3],
                   result->nb[1], result->nb[2], result->nb[3],
                0);

    // 返回处理后的结果张量
    return result;
}

// ggml_flash_attn

// 结构体 ggml_tensor 指针类型的 flash_attn 函数，用于执行注意力机制操作
struct ggml_tensor * ggml_flash_attn(
        struct ggml_context * ctx,
        struct ggml_tensor  * q,
        struct ggml_tensor  * k,
        struct ggml_tensor  * v,
        bool                  masked) {
    // 断言矩阵 k 和 q 可以相乘
    GGML_ASSERT(ggml_can_mul_mat(k, q));
    // TODO: check if vT can be multiplied by (k*qT)

    // 初始化布尔变量 is_node 为 false
    bool is_node = false;

    // 如果输入张量 q、k、v 中任意一个具有梯度信息，则将 is_node 设置为真
    if (q->grad || k->grad || v->grad) {
        is_node = true;
    }

    // 创建一个新的张量 result，用于存储注意力机制的结果
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F32, GGML_MAX_DIMS, q->ne);

    // 根据是否进行 mask 操作，设置参数 t 的值
    int32_t t = masked ? 1 : 0;
    ggml_set_op_params(result, &t, sizeof(t));

    // 设置张量 result 的操作为 FLASH_ATTN，梯度为 is_node 为真时的 result 张量，源张量为 q、k、v
    result->op   = GGML_OP_FLASH_ATTN;
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = q;
    result->src[1] = k;
    result->src[2] = v;

    // 返回注意力机制的结果张量
    return result;
}
// ggml_flash_ff

// 实现前向传播的函数，计算两个张量的乘积
struct ggml_tensor * ggml_flash_ff(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b0,
        struct ggml_tensor  * b1,
        struct ggml_tensor  * c0,
        struct ggml_tensor  * c1) {
    // 检查是否可以对矩阵进行乘法运算
    GGML_ASSERT(ggml_can_mul_mat(b0, a));
    // TODO: more checks

    bool is_node = false;

    // 检查是否有梯度信息，如果有则设置为节点
    if (a->grad || b0->grad || b1->grad || c0->grad || c1->grad) {
        is_node = true;
    }

    // 创建一个新的张量用于存储结果
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F32, GGML_MAX_DIMS, a->ne);

    result->op   = GGML_OP_FLASH_FF;
    // 如果是节点，则复制结果张量
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;
    result->src[1] = b0;
    result->src[2] = b1;
    result->src[3] = c0;
    result->src[4] = c1;

    return result;
}

// ggml_flash_attn_back

// 实现注意力机制的反向传播函数
struct ggml_tensor * ggml_flash_attn_back(
        struct ggml_context * ctx,
        struct ggml_tensor  * q,
        struct ggml_tensor  * k,
        struct ggml_tensor  * v,
        struct ggml_tensor  * d,
        bool                  masked) {
    // 检查是否可以对矩阵进行乘法运算
    GGML_ASSERT(ggml_can_mul_mat(k, q));
    // TODO: check if vT can be multiplied by (k*qT)

    // 设置各个张量的维度信息
    const int64_t     D = q->ne[0];
    const int64_t     N = q->ne[1];
    const int64_t     M = k->ne[1];
    const int64_t   ne2 = q->ne[2];
    const int64_t   ne3 = q->ne[3];
    const int64_t kvne2 = k->ne[2];

    // 进行维度信息的检查
    GGML_ASSERT(k->ne[0] == D);
    GGML_ASSERT(v->ne[0] == M);
    GGML_ASSERT(v->ne[1] == D);
    GGML_ASSERT(d->ne[0] == D);
    GGML_ASSERT(d->ne[1] == N);
    GGML_ASSERT(k->ne[2] == kvne2);
    GGML_ASSERT(k->ne[3] == ne3);
    GGML_ASSERT(v->ne[2] == kvne2);
    GGML_ASSERT(v->ne[3] == ne3);
    GGML_ASSERT(d->ne[2] == ne2);
    GGML_ASSERT(d->ne[3] == ne3);

    // 检查 ne2 是否可以被 kvne2 整除
    GGML_ASSERT(ne2 % kvne2 == 0);

    bool is_node = false;
    // 如果 q、k、v 中任意一个有梯度，则将 is_node 设为 false，避免创建结果的梯度
    if (q->grad || k->grad || v->grad) {
        is_node = false;
    }

    // 将 q、k、v 的梯度作为连续张量存储在结果中，注意 v 和 gradv 实际上是转置的，即 v->ne[0] != D
    const int64_t elem_q = ggml_nelements(q);
    const int64_t elem_k = ggml_nelements(k);
    const int64_t elem_v = ggml_nelements(v);

    // 设置结果类型为 GGML_TYPE_F32，并计算每个元素的大小
    enum ggml_type result_type = GGML_TYPE_F32;
    GGML_ASSERT(ggml_blck_size(result_type) == 1);
    const size_t tsize = ggml_type_size(result_type);

    // 计算 q、k、v 在结果中的偏移量
    const size_t offs_q = 0;
    const size_t offs_k = offs_q + GGML_PAD(elem_q * tsize, GGML_MEM_ALIGN);
    const size_t offs_v = offs_k + GGML_PAD(elem_k * tsize, GGML_MEM_ALIGN);
    const size_t end    = offs_v + GGML_PAD(elem_v * tsize, GGML_MEM_ALIGN);

    // 计算结果张量的元素数量
    const size_t nelements = (end + tsize - 1)/tsize;

    // 创建一个新的一维张量作为结果
    struct ggml_tensor * result = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, nelements);

    // 设置操作参数为 masked_i，如果 masked 为真则为 1，否则为 0
    int32_t masked_i = masked ? 1 : 0;
    ggml_set_op_params(result, &masked_i, sizeof(masked_i));

    // 设置结果张量的操作类型为 GGML_OP_FLASH_ATTN_BACK，梯度根据 is_node 决定是否复制
    result->op   = GGML_OP_FLASH_ATTN_BACK;
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = q;
    result->src[1] = k;
    result->src[2] = v;
    result->src[3] = d;

    // 返回结果张量
    return result;
// 定义 ggml_win_part 函数，用于对输入的张量进行窗口划分操作
struct ggml_tensor * ggml_win_part(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   w) {
    // 断言输入张量的第四维度大小为1
    GGML_ASSERT(a->ne[3] == 1);
    // 断言输入张量的数据类型为 GGML_TYPE_F32

    GGML_ASSERT(a->type  == GGML_TYPE_F32);

    // 初始化一个布尔值 is_node 为 false
    bool is_node = false;

    // 如果输入张量存在梯度
    if (a->grad) {
        // 断言为假，提示需要实现反向传播
        GGML_ASSERT(false); // TODO: implement backward
        // 将 is_node 设置为 true
        is_node = true;
    }

    // 计算填充值 px 和 py
    const int px = (w - a->ne[1]%w)%w;
    const int py = (w - a->ne[2]%w)%w;

    // 计算 npx 和 npy
    const int npx = (px + a->ne[1])/w;
    const int npy = (py + a->ne[2])/w;
    const int np  = npx*npy;

    // 创建一个新的张量 result，维度为 4
    const int64_t ne[4] = { a->ne[0], w, w, np, };
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F32, 4, ne);

    // 设置操作参数
    int32_t params[] = { npx, npy, w };
    ggml_set_op_params(result, params, sizeof(params));

    // 设置操作类型为 GGML_OP_WIN_PART
    result->op   = GGML_OP_WIN_PART;
    // 如果 is_node 为 true，则复制 result 作为梯度
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置 result 的源张量为 a
    result->src[0] = a;

    // 返回 result
    return result;
}

// 定义 ggml_win_unpart 函数，用于对输入的张量进行窗口合并操作
struct ggml_tensor * ggml_win_unpart(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   w0,
        int                   h0,
        int                   w) {
    // 断言输入张量的数据类型为 GGML_TYPE_F32
    GGML_ASSERT(a->type == GGML_TYPE_F32);

    // 初始化一个布尔值 is_node 为 false
    bool is_node = false;

    // 如果输入张量存在梯度
    if (a->grad) {
        // 断言为假，提示需要实现反向传播
        GGML_ASSERT(false); // TODO: implement backward
        // 将 is_node 设置为 true
        is_node = true;
    }

    // 创建一个新的张量 result，维度为 3
    const int64_t ne[4] = { a->ne[0], w0, h0, 1, };
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F32, 3, ne);

    // 设置操作参数
    int32_t params[] = { w };
    ggml_set_op_params(result, params, sizeof(params));

    // 设置操作类型为 GGML_OP_WIN_UNPART
    result->op   = GGML_OP_WIN_UNPART;
    // 如果 is_node 为 true，则复制 result 作为梯度
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置 result 的源张量为 a
    result->src[0] = a;

    // 返回 result
    return result;
}

// 定义 ggml_get_rel_pos 函数，用于获取相对位置信息
struct ggml_tensor * ggml_get_rel_pos(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   qh,
        int                   kh) {
    // 断言 qh 和 kh 相等
    GGML_ASSERT(qh == kh);
    // 断言 2*MAX(qh, kh) - 1 等于输入张量的第二维度大小
    // 初始化一个布尔变量 is_node，用于标记是否为节点
    bool is_node = false;

    // 如果输入张量 a 的梯度不为空
    if (a->grad) {
        // 断言，提示需要实现反向传播
        GGML_ASSERT(false); // TODO: implement backward
        // 将 is_node 设置为 true
        is_node = true;
    }

    // 定义一个包含四个元素的整型数组 ne，分别为 a->ne[0]、kh、qh、1
    const int64_t ne[4] = { a->ne[0], kh, qh, 1, };
    // 创建一个新的张量 result，数据类型为 GGML_TYPE_F16，维度为 3，形状为 ne
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F16, 3, ne);

    // 设置 result 的操作类型为 GGML_OP_GET_REL_POS
    result->op   = GGML_OP_GET_REL_POS;
    // 如果 is_node 为 true，则将 result 的梯度设置为 result 的副本，否则设置为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置 result 的源张量为 a
    result->src[0] = a;

    // 返回创建的结果张量 result
    return result;
}

// ggml_add_rel_pos

// 实现相对位置加法操作
static struct ggml_tensor * ggml_add_rel_pos_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * pw,
        struct ggml_tensor  * ph,
        bool                  inplace) {
    // 断言两个张量的形状相同
    GGML_ASSERT(ggml_are_same_shape(pw, ph));
    // 断言张量 a 是连续的
    GGML_ASSERT(ggml_is_contiguous(a));
    // 断言张量 pw 是连续的
    GGML_ASSERT(ggml_is_contiguous(pw));
    // 断言张量 ph 是连续的
    GGML_ASSERT(ggml_is_contiguous(ph));
    // 断言张量 ph 的数据类型为 GGML_TYPE_F32
    GGML_ASSERT(ph->type == GGML_TYPE_F32);
    // 断言张量 pw 的数据类型为 GGML_TYPE_F32
    GGML_ASSERT(pw->type == GGML_TYPE_F32);
    // 断言 pw 的第四维度大小等于 a 的第三维度大小
    GGML_ASSERT(pw->ne[3] == a->ne[2]);
    // 断言 pw 的第一维度乘以第一维度等于 a 的第一维度
    GGML_ASSERT(pw->ne[0]*pw->ne[0] == a->ne[0]);
    // 断言 pw 的第二维度乘以第三维度等于 a 的第二维度
    GGML_ASSERT(pw->ne[1]*pw->ne[2] == a->ne[1]);

    bool is_node = false;

    // 如果不是原地操作且任一张量具有梯度，则设置 is_node 为 true
    if (!inplace && (a->grad || pw->grad || ph->grad)) {
        is_node = true;
    }

    // 根据 inplace 参数选择创建新张量或者视图张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);
    // 设置操作参数为整数 0 或 1，取决于是否原地操作
    ggml_set_op_params_i32(result, 0, inplace ? 1 : 0);

    // 设置结果张量的操作类型为 GGML_OP_ADD_REL_POS
    result->op   = GGML_OP_ADD_REL_POS;
    // 如果是节点，则为结果张量创建梯度张量，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量
    result->src[0] = a;
    result->src[1] = pw;
    result->src[2] = ph;

    return result;
}

// 对外接口，执行相对位置加法操作
struct ggml_tensor * ggml_add_rel_pos(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * pw,
        struct ggml_tensor  * ph) {
    return ggml_add_rel_pos_impl(ctx, a, pw, ph, false);
}

// 对外接口，执行原地相对位置加法操作
struct ggml_tensor * ggml_add_rel_pos_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * pw,
        struct ggml_tensor  * ph) {
    return ggml_add_rel_pos_impl(ctx, a, pw, ph, true);
}

// gmml_unary

// 实现一元操作
static struct ggml_tensor * ggml_unary_impl(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        enum ggml_unary_op op,
        bool inplace) {
    bool is_node = false;

    // 如果不是原地操作且张量具有梯度，则设置 is_node 为 true
    if (!inplace && (a->grad)) {
        is_node = true;
    }

    // 根据 inplace 参数选择创建新张量或者视图张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);
    # 设置 result 结构体中的第一个参数为整型 op
    ggml_set_op_params_i32(result, 0, (int32_t) op);

    # 设置 result 结构体中的操作类型为一元操作
    result->op   = GGML_OP_UNARY;
    # 如果是节点，则为 result 结构体中的梯度 grad 赋值为 result 的副本，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    # 设置 result 结构体中的第一个源操作数为 a
    result->src[0] = a;

    # 返回 result 结构体
    return result;
// 定义一个函数，对输入的张量进行一元操作，并返回结果张量
struct ggml_tensor * ggml_unary(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        enum ggml_unary_op op) {
    return ggml_unary_impl(ctx, a, op, false);
}

// 定义一个函数，对输入的张量进行原地一元操作，并返回结果张量
struct ggml_tensor * ggml_unary_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        enum ggml_unary_op op) {
    return ggml_unary_impl(ctx, a, op, true);
}

// 定义一个静态函数，实现对浮点型张量进行一元操作的映射
static struct ggml_tensor * ggml_map_unary_impl_f32(
        struct ggml_context        * ctx,
        struct ggml_tensor         * a,
        const  ggml_unary_op_f32_t fun,
        bool   inplace) {
    bool is_node = false;

    // 如果不是原地操作且输入张量具有梯度，则设置为节点
    if (!inplace && a->grad) {
        is_node = true;
    }

    // 根据是否原地操作选择复制或者视图操作
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置操作参数
    ggml_set_op_params(result, (const void *) &fun, sizeof(fun));

    // 设置结果张量的操作类型为 GGML_OP_MAP_UNARY
    result->op = GGML_OP_MAP_UNARY;
    // 如果是节点，则复制结果张量作为梯度，否则梯度为空
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;

    return result;
}

// 定义一个函数，对浮点型张量进行一元操作的映射
struct ggml_tensor * ggml_map_unary_f32(
        struct ggml_context        * ctx,
        struct ggml_tensor         * a,
        const  ggml_unary_op_f32_t fun) {
    return ggml_map_unary_impl_f32(ctx, a, fun, false);
}

// 定义一个函数，对浮点型张量进行原地一元操作的映射
struct ggml_tensor * ggml_map_unary_inplace_f32(
        struct ggml_context        * ctx,
        struct ggml_tensor         * a,
        const  ggml_unary_op_f32_t fun) {
    return ggml_map_unary_impl_f32(ctx, a, fun, true);
}

// 定义一个静态函数，实现对浮点型张量进行二元操作的映射
static struct ggml_tensor * ggml_map_binary_impl_f32(
        struct ggml_context         * ctx,
        struct ggml_tensor          * a,
        struct ggml_tensor          * b,
        const  ggml_binary_op_f32_t fun,
        bool   inplace) {
    // 断言两个张量具有相同的形状
    GGML_ASSERT(ggml_are_same_shape(a, b));

    bool is_node = false;

    // 如果不是原地操作且输入张量具有梯度，则设置为节点
    if (!inplace && (a->grad || b->grad)) {
        is_node = true;
    }

    // 根据是否原地操作选择复制或者视图操作
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);
    # 设置操作的参数，将函数指针 fun 转换为常量指针，并指定大小
    ggml_set_op_params(result, (const void *) &fun, sizeof(fun));

    # 设置操作类型为二元映射
    result->op = GGML_OP_MAP_BINARY;
    # 如果是节点，则复制结果的梯度张量，否则设置为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    # 设置操作的第一个输入源为 a
    result->src[0] = a;
    # 设置操作的第二个输入源为 b
    result->src[1] = b;

    # 返回操作结果
    return result;
// 定义一个函数，将两个浮点数张量进行二元操作，并返回结果张量
struct ggml_tensor * ggml_map_binary_f32(
        struct ggml_context         * ctx,
        struct ggml_tensor          * a,
        struct ggml_tensor          * b,
        const  ggml_binary_op_f32_t fun) {
    // 调用内部实现函数，传入参数和标志位false
    return ggml_map_binary_impl_f32(ctx, a, b, fun, false);
}

// 定义一个函数，将两个浮点数张量进行原地二元操作，并返回结果张量
struct ggml_tensor * ggml_map_binary_inplace_f32(
        struct ggml_context         * ctx,
        struct ggml_tensor          * a,
        struct ggml_tensor          * b,
        const  ggml_binary_op_f32_t fun) {
    // 调用内部实现函数，传入参数和标志位true
    return ggml_map_binary_impl_f32(ctx, a, b, fun, true);
}

// 定义一个静态函数，实现对一个浮点数张量进行自定义操作1
static struct ggml_tensor * ggml_map_custom1_impl_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        const  ggml_custom1_op_f32_t   fun,
        bool   inplace) {
    // 初始化一个布尔值is_node为false
    bool is_node = false;

    // 如果不是原地操作且张量a有梯度
    if (!inplace && a->grad) {
        // 将is_node设置为true
        is_node = true;
    }

    // 根据inplace标志位选择复制张量或者视图张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置结果张量的操作参数为fun
    ggml_set_op_params(result, (const void *) &fun, sizeof(fun));

    // 设置结果张量的操作类型为GGML_OP_MAP_CUSTOM1_F32
    result->op = GGML_OP_MAP_CUSTOM1_F32;
    // 如果is_node为true，则为结果张量设置梯度为自身的复制，否则为null
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为a
    result->src[0] = a;

    // 返回结果张量
    return result;
}

// 定义一个函数，将一个浮点数张量进行自定义操作1，并返回结果张量
struct ggml_tensor * ggml_map_custom1_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        const  ggml_custom1_op_f32_t   fun) {
    // 调用内部实现函数，传入参数和标志位false
    return ggml_map_custom1_impl_f32(ctx, a, fun, false);
}

// 定义一个函数，将一个浮点数张量进行原地自定义操作1，并返回结果张量
struct ggml_tensor * ggml_map_custom1_inplace_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        const  ggml_custom1_op_f32_t   fun) {
    // 调用内部实现函数，传入参数和标志位true
    return ggml_map_custom1_impl_f32(ctx, a, fun, true);
}

// 定义一个静态函数，实现对两个浮点数张量进行自定义操作2
static struct ggml_tensor * ggml_map_custom2_impl_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
        const  ggml_custom2_op_f32_t   fun,
        bool   inplace) {
    // 初始化一个布尔值is_node为false
    bool is_node = false;
    // 如果不是原地操作且输入张量 a 或 b 具有梯度信息，则将 is_node 设置为 true
    if (!inplace && (a->grad || b->grad)) {
        is_node = true;
    }

    // 根据是否原地操作选择创建视图张量或复制张量，并将结果保存在 result 中
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置操作参数为自定义函数 fun，并指定参数大小
    ggml_set_op_params(result, (const void *) &fun, sizeof(fun));

    // 设置结果张量的操作类型为 GGML_OP_MAP_CUSTOM2_F32
    result->op = GGML_OP_MAP_CUSTOM2_F32;
    
    // 如果 is_node 为 true，则为结果张量的梯度信息创建一个新的张量，否则设置为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    
    // 设置结果张量的源张量为输入张量 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    // 返回结果张量
    return result;
// 定义一个函数，将两个输入张量按照自定义的操作函数进行映射，返回结果张量
struct ggml_tensor * ggml_map_custom2_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
        const  ggml_custom2_op_f32_t   fun) {
    // 调用内部实现函数，传入参数和标志位false
    return ggml_map_custom2_impl_f32(ctx, a, b, fun, false);
}

// 定义一个函数，将两个输入张量按照自定义的操作函数进行映射，结果直接覆盖到第一个输入张量
struct ggml_tensor * ggml_map_custom2_inplace_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
        const  ggml_custom2_op_f32_t   fun) {
    // 调用内部实现函数，传入参数和标志位true
    return ggml_map_custom2_impl_f32(ctx, a, b, fun, true);
}

// 内部实现函数，将三个输入张量按照自定义的操作函数进行映射
static struct ggml_tensor * ggml_map_custom3_impl_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
        struct ggml_tensor           * c,
        const  ggml_custom3_op_f32_t   fun,
        bool   inplace) {
    // 初始化一个标志位is_node为false
    bool is_node = false;

    // 如果不是inplace操作且任意一个输入张量有梯度信息，则将is_node标志位设为true
    if (!inplace && (a->grad || b->grad || c->grad)) {
        is_node = true;
    }

    // 根据inplace标志位选择复制还是视图操作，得到结果张量result
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置结果张量的操作参数为自定义操作函数fun
    ggml_set_op_params(result, (const void *) &fun, sizeof(fun));

    // 设置结果张量的操作类型为GGML_OP_MAP_CUSTOM3_F32
    result->op = GGML_OP_MAP_CUSTOM3_F32;
    // 如果is_node为true，则为结果张量设置梯度信息，否则为null
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为a、b、c
    result->src[0] = a;
    result->src[1] = b;
    result->src[2] = c;

    // 返回结果张量
    return result;
}

// 定义一个函数，将三个输入张量按照自定义的操作函数进行映射，返回结果张量
struct ggml_tensor * ggml_map_custom3_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
        struct ggml_tensor           * c,
        const  ggml_custom3_op_f32_t   fun) {
    // 调用内部实现函数，传入参数和标志位false
    return ggml_map_custom3_impl_f32(ctx, a, b, c, fun, false);
}

// 定义一个函数，将三个输入张量按照自定义的操作函数进行映射，结果直接覆盖到第一个输入张量
struct ggml_tensor * ggml_map_custom3_inplace_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
        struct ggml_tensor           * c,
        const  ggml_custom3_op_f32_t   fun) {
    # 调用 ggml_map_custom3_impl_f32 函数，传入参数 ctx, a, b, c, fun, true，并返回结果
    return ggml_map_custom3_impl_f32(ctx, a, b, c, fun, true);
// 结构体定义，用于存储自定义操作函数、任务数量和用户数据
struct ggml_map_custom1_op_params {
    ggml_custom1_op_t fun; // 自定义操作函数
    int n_tasks; // 任务数量
    void * userdata; // 用户数据
};

// 实现自定义操作函数的映射，返回一个新的张量
static struct ggml_tensor * ggml_map_custom1_impl(
        struct ggml_context          * ctx, // 上下文
        struct ggml_tensor           * a, // 输入张量
        const  ggml_custom1_op_t       fun, // 自定义操作函数
        int                            n_tasks, // 任务数量
        void                         * userdata, // 用户数据
        bool                           inplace) { // 是否原地操作

    GGML_ASSERT(n_tasks == GGML_N_TASKS_MAX || n_tasks > 0); // 断言任务数量合法

    bool is_node = false; // 初始化是否为节点标志为假

    if (!inplace && a->grad) { // 如果不是原地操作且输入张量有梯度
        is_node = true; // 设置为节点
    }

    // 根据是否原地操作选择复制或者视图输入张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置自定义操作函数的参数
    struct ggml_map_custom1_op_params params = {
        /*.fun      =*/ fun, // 自定义操作函数
        /*.n_tasks  =*/ n_tasks, // 任务数量
        /*.userdata =*/ userdata // 用户数据
    };
    ggml_set_op_params(result, (const void *) &params, sizeof(params)); // 设置操作参数

    result->op = GGML_OP_MAP_CUSTOM1; // 设置操作类型为自定义操作1
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL; // 如果是节点，复制梯度张量，否则为空
    result->src[0] = a; // 设置源张量为输入张量

    return result; // 返回结果张量
}

// 对外接口，调用自定义操作函数的映射
struct ggml_tensor * ggml_map_custom1(
        struct ggml_context          * ctx, // 上下文
        struct ggml_tensor           * a, // 输入张量
        const  ggml_custom1_op_t       fun, // 自定义操作函数
        int                            n_tasks, // 任务数量
        void                         * userdata) { // 用户数据
    return ggml_map_custom1_impl(ctx, a, fun, n_tasks, userdata, false); // 调用实现函数，不原地操作
}

// 对外接口，调用自定义操作函数的映射（原地操作）
struct ggml_tensor * ggml_map_custom1_inplace(
        struct ggml_context          * ctx, // 上下文
        struct ggml_tensor           * a, // 输入张量
        const  ggml_custom1_op_t       fun, // 自定义操作函数
        int                            n_tasks, // 任务数量
        void                         * userdata) { // 用户数据
    return ggml_map_custom1_impl(ctx, a, fun, n_tasks, userdata, true); // 调用实现函数，原地操作
}

// 结构体定义，用于存储自定义操作函数、任务数量和用户数据
struct ggml_map_custom2_op_params {
    ggml_custom2_op_t fun; // 自定义操作函数
    int n_tasks; // 任务数量
    void * userdata; // 用户数据
};
// 实现自定义的二元操作映射函数，返回一个新的张量
static struct ggml_tensor * ggml_map_custom2_impl(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
        const  ggml_custom2_op_t       fun,
        int                            n_tasks,
        void                         * userdata,
        bool                           inplace) {
    // 断言任务数为最大任务数或者大于0
    GGML_ASSERT(n_tasks == GGML_N_TASKS_MAX || n_tasks > 0);

    // 初始化节点标志为假
    bool is_node = false;

    // 如果不是原地操作且张量 a 或 b 具有梯度
    if (!inplace && (a->grad || b->grad)) {
        // 设置节点标志为真
        is_node = true;
    }

    // 根据原地操作标志选择复制或者视图张量 a
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置自定义二元操作参数
    struct ggml_map_custom2_op_params params = {
        /*.fun      =*/ fun,
        /*.n_tasks  =*/ n_tasks,
        /*.userdata =*/ userdata
    };
    ggml_set_op_params(result, (const void *) &params, sizeof(params));

    // 设置操作类型为自定义二元操作
    result->op = GGML_OP_MAP_CUSTOM2;
    // 如果是节点，则复制结果张量作为梯度，否则梯度为空
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置源张量为 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    // 返回结果张量
    return result;
}

// 调用 ggml_map_custom2_impl 函数，不进行原地操作
struct ggml_tensor * ggml_map_custom2(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
        const  ggml_custom2_op_t       fun,
        int                            n_tasks,
        void                         * userdata) {
    return ggml_map_custom2_impl(ctx, a, b, fun, n_tasks, userdata, false);
}

// 调用 ggml_map_custom2_impl 函数，进行原地操作
struct ggml_tensor * ggml_map_custom2_inplace(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
        const  ggml_custom2_op_t       fun,
        int                            n_tasks,
        void                         * userdata) {
    return ggml_map_custom2_impl(ctx, a, b, fun, n_tasks, userdata, true);
}

// 自定义三元操作映射函数的参数结构体
struct ggml_map_custom3_op_params {
    ggml_custom3_op_t fun;
    int n_tasks;
    void * userdata;
};
// 实现自定义三元操作的映射，返回结果张量
static struct ggml_tensor * ggml_map_custom3_impl(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
        struct ggml_tensor           * c,
        const  ggml_custom3_op_t       fun,
        int                            n_tasks,
        void                         * userdata,
        bool                           inplace) {
    // 断言任务数为最大任务数或者大于0
    GGML_ASSERT(n_tasks == GGML_N_TASKS_MAX || n_tasks > 0);

    // 初始化节点标志为假
    bool is_node = false;

    // 如果不是原地操作且任意一个输入张量有梯度，则设置节点标志为真
    if (!inplace && (a->grad || b->grad || c->grad)) {
        is_node = true;
    }

    // 根据原地操作标志选择复制或者视图操作
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置自定义三元操作参数
    struct ggml_map_custom3_op_params params = {
        /*.fun      =*/ fun,
        /*.n_tasks  =*/ n_tasks,
        /*.userdata =*/ userdata
    };
    ggml_set_op_params(result, (const void *) &params, sizeof(params));

    // 设置结果张量的操作类型为自定义三元操作
    result->op = GGML_OP_MAP_CUSTOM3;
    // 如果是节点，则复制结果张量作为梯度张量，否则梯度张量为空
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的输入张量
    result->src[0] = a;
    result->src[1] = b;
    result->src[2] = c;

    // 返回结果张量
    return result;
}

// 对外接口，执行自定义三元操作的映射
struct ggml_tensor * ggml_map_custom3(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
        struct ggml_tensor           * c,
        const  ggml_custom3_op_t       fun,
        int                            n_tasks,
        void                         * userdata) {
    return ggml_map_custom3_impl(ctx, a, b, c, fun, n_tasks, userdata, false);
}

// 对外接口，执行自定义三元操作的原地映射
struct ggml_tensor * ggml_map_custom3_inplace(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
        struct ggml_tensor           * c,
        const  ggml_custom3_op_t       fun,
        int                            n_tasks,
        void                         * userdata) {
    return ggml_map_custom3_impl(ctx, a, b, c, fun, n_tasks, userdata, true);
}
// 计算交叉熵损失
struct ggml_tensor * ggml_cross_entropy_loss(
        struct ggml_context         * ctx,
        struct ggml_tensor          * a,
        struct ggml_tensor          * b) {
    // 断言两个张量形状相同
    GGML_ASSERT(ggml_are_same_shape(a, b));
    // 初始化节点标志为 false
    bool is_node = false;

    // 如果输入张量 a 或 b 具有梯度信息，则将节点标志设置为 true
    if (a->grad || b->grad) {
        is_node = true;
    }

    // 创建一个新的一维张量用于存储结果
    struct ggml_tensor * result = ggml_new_tensor_1d(ctx, a->type, 1);

    // 设置结果张量的操作类型为交叉熵损失
    result->op   = GGML_OP_CROSS_ENTROPY_LOSS;
    // 如果节点标志为 true，则为结果张量分配一个梯度张量，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为输入张量 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    return result;
}

// 计算交叉熵损失的反向传播
struct ggml_tensor * ggml_cross_entropy_loss_back(
        struct ggml_context         * ctx,
        struct ggml_tensor          * a,
        struct ggml_tensor          * b,
        struct ggml_tensor          * c) {
    // 断言输入张量 a 和 b 的形状相同
    GGML_ASSERT(ggml_are_same_shape(a, b));
    // 断言张量 c 是标量
    GGML_ASSERT(ggml_is_scalar(c));

    // 复制输入张量 a 作为结果张量
    struct ggml_tensor * result = ggml_dup_tensor(ctx, a);

    // 设置结果张量的操作类型为交叉熵损失的反向传播
    result->op   = GGML_OP_CROSS_ENTROPY_LOSS_BACK;
    // 将结果张量的梯度设置为 NULL
    result->grad = NULL;
    // 设置结果张量的源张量为输入张量 a、b 和 c
    result->src[0] = a;
    result->src[1] = b;
    result->src[2] = c;

    return result;
}

////////////////////////////////////////////////////////////////////////////////

// 设置参数张量
void ggml_set_param(
        struct ggml_context * ctx,
        struct ggml_tensor * tensor) {
    // 将张量标记为参数张量
    tensor->is_param = true;

    // 断言张量的梯度为 NULL，然后复制张量作为梯度张量
    GGML_ASSERT(tensor->grad == NULL);
    tensor->grad = ggml_dup_tensor(ctx, tensor);
    // 格式化梯度张量的名称
    ggml_format_name(tensor->grad, "%s (grad)", tensor->name);
}

// 计算前向传播的重复
static void ggml_compute_forward_dup_same_cont(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言目标张量和源张量的元素数量相同
    GGML_ASSERT(ggml_nelements(dst) == ggml_nelements(src0));
    // 断言目标张量和源张量是连续的，并且类型相同
    GGML_ASSERT(ggml_is_contiguous(dst) && ggml_is_contiguous(src0));
    GGML_ASSERT(src0->type == dst->type);

    // 如果任务类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }
}
    // 获取源数据 src0 的第一个维度大小
    const size_t nb00 = src0->nb[0];
    // 获取目标数据 dst 的第一个维度大小
    const size_t nb0 = dst->nb[0];

    // 获取线程索引
    const int ith = params->ith; // thread index
    // 获取线程总数
    const int nth = params->nth; // number of threads

    // 并行化处理每个元素
    const int ne = ggml_nelements(dst);
    // 计算每个线程需要处理的元素数量
    const int dr = (ne + nth - 1) / nth;
    // 计算当前线程处理的元素范围
    const int ie0 = dr * ith;
    const int ie1 = MIN(ie0 + dr, ne);

    // 如果当前线程有元素需要处理
    if (ie0 < ie1) {
        // 将源数据 src0 的部分数据复制到目标数据 dst 中
        memcpy(
            ((char *)  dst->data + ie0*nb0),
            ((char *) src0->data + ie0*nb00),
            (ie1 - ie0) * ggml_type_size(src0->type));
    }
// 计算前向传播的函数，复制操作，处理半精度浮点数数据
static void ggml_compute_forward_dup_f16(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言目标张量和源张量的元素数量相等
    GGML_ASSERT(ggml_nelements(dst) == ggml_nelements(src0));

    // 如果参数类型为初始化或结束任务，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 定义一些本地变量
    GGML_TENSOR_UNARY_OP_LOCALS

    // 获取线程索引和线程数量
    const int ith = params->ith; // 线程索引
    const int nth = params->nth; // 线程数量

    // 如果源张量和目标张量都是连续的，并且类型相同，则调用相同连续复制函数并返回
    if (ggml_is_contiguous(src0) && ggml_is_contiguous(dst) && src0->type == dst->type) {
        ggml_compute_forward_dup_same_cont(params, src0, dst);
        return;
    }

    // 并行化处理每一行数据
    const int nr = ne01; // 行数
    const int dr = (nr + nth - 1) / nth; // 每个线程处理的行数
    const int ir0 = dr * ith; // 该线程处理的起始行
    const int ir1 = MIN(ir0 + dr, nr); // 该线程处理的结束行

    // 如果源张量和目标张量类型相同，并且满足一些条件，则通过行复制数据
    if (src0->type == dst->type &&
        ne00 == ne0 &&
        nb00 == ggml_type_size(src0->type) && nb0 == ggml_type_size(dst->type)) {
        // 计算每行数据的大小
        const size_t rs = ne00*nb00;
        // 循环复制数据
        for (int64_t i03 = 0; i03 < ne03; i03++) {
            for (int64_t i02 = 0; i02 < ne02; i02++) {
                for (int64_t i01 = ir0; i01 < ir1; i01++) {
                    memcpy(
                        ((char *)  dst->data + i01*nb1  + i02*nb2  + i03*nb3),
                        ((char *) src0->data + i01*nb01 + i02*nb02 + i03*nb03),
                        rs);
                }
            }
        }
        return;
    }

    // TODO: 为可以从 memcpy 中受益的张量形状/步长添加更多特殊情况的实现

    // 初始化目标张量的计数器
    int64_t i10 = 0;
    int64_t i11 = 0;
    int64_t i12 = 0;
    int64_t i13 = 0;

    // 否则，抛出断言错误，提示需要实现
    } else {
        GGML_ASSERT(false); // TODO: implement
    }
}

// 计算前向传播的函数，复制操作，处理单精度浮点数数据
static void ggml_compute_forward_dup_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 检查目标张量和源张量的元素数量是否相等
    GGML_ASSERT(ggml_nelements(dst) == ggml_nelements(src0));

    // 如果任务类型是初始化或者结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 定义一些本地变量
    GGML_TENSOR_UNARY_OP_LOCALS

    // 获取线程索引和线程数量
    const int ith = params->ith; // 线程索引
    const int nth = params->nth; // 线程数量

    // 如果源张量和目标张量都是连续存储的，并且数据类型相同，则直接复制数据
    if (ggml_is_contiguous(src0) && ggml_is_contiguous(dst) && src0->type == dst->type) {
        ggml_compute_forward_dup_same_cont(params, src0, dst);
        return;
    }

    // 按行并行处理
    const int nr = ne01; // 行数
    // 每个线程处理的行数
    const int dr = (nr + nth - 1) / nth;
    // 当前线程处理的行范围
    const int ir0 = dr * ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 如果源张量和目标张量的数据类型相同，并且其他条件满足，则按行复制数据
    if (src0->type == dst->type &&
        ne00 == ne0 &&
        nb00 == ggml_type_size(src0->type) && nb0 == ggml_type_size(dst->type)) {
        // 按行复制数据
        const size_t rs = ne00*nb00;
        for (int64_t i03 = 0; i03 < ne03; i03++) {
            for (int64_t i02 = 0; i02 < ne02; i02++) {
                for (int64_t i01 = ir0; i01 < ir1; i01++) {
                    memcpy(
                        ((char *)  dst->data + i01*nb1  + i02*nb2  + i03*nb3),
                        ((char *) src0->data + i01*nb01 + i02*nb02 + i03*nb03),
                        rs);
                }
            }
        }
        return;
    }

    // 初始化目标张量的计数器
    int64_t i10 = 0;
    int64_t i11 = 0;
    int64_t i12 = 0;
    int64_t i13 = 0;

    // 如果不满足前面的条件，则输出错误信息
    } else {
        GGML_ASSERT(false); // TODO: implement
    }
// 一个简化版本的 ggml_compute_forward_dup 函数，不进行浮点类型的转换，直接使用 memcpy 进行复制
static void ggml_compute_forward_dup_bytes(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言目标张量和源张量的元素数量相同
    GGML_ASSERT(ggml_nelements(dst) == ggml_nelements(src0));
    // 断言源张量和目标张量的数据类型相同
    GGML_ASSERT(src0->type == dst->type);

    // 如果任务类型是初始化或者结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 如果源张量和目标张量是连续的，则调用 ggml_compute_forward_dup_same_cont 函数进行复制，并返回
    if (ggml_is_contiguous(src0) && ggml_is_contiguous(dst)) {
        ggml_compute_forward_dup_same_cont(params, src0, dst);
        return;
    }

    // 定义一些本地变量
    GGML_TENSOR_UNARY_OP_LOCALS;

    // 计算数据类型的大小
    const size_t type_size = ggml_type_size(src0->type);
    // 线程索引
    const int ith = params->ith;
    // 线程数量
    const int nth = params->nth;

    // 按行并行化
    const int nr = ne01;
    // 每个线程的行数
    const int dr = (nr + nth - 1) / nth;
    // 该线程的行范围
    const int ir0 = dr * ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 如果源张量和目标张量的数据类型相同，并且其他条件满足，则按行复制数据
    if (src0->type == dst->type &&
        ne00 == ne0 &&
        nb00 == type_size && nb0 == type_size) {
        // 按行复制数据
        const size_t rs = ne00 * type_size;
        for (int64_t i03 = 0; i03 < ne03; i03++) {
            for (int64_t i02 = 0; i02 < ne02; i02++) {
                for (int64_t i01 = ir0; i01 < ir1; i01++) {
                    memcpy(
                        ((char *)  dst->data + i01*nb1  + i02*nb2  + i03*nb3),
                        ((char *) src0->data + i01*nb01 + i02*nb02 + i03*nb03),
                        rs);
                }
            }
        }
        return;
    }
}
    // 检查目标数组是否是连续的
    if (ggml_is_contiguous(dst)) {
        // 初始化索引值和目标数组指针
        size_t id = 0;
        char * dst_ptr = (char *) dst->data;
        const size_t rs = ne00 * type_size;

        // 如果源数组在第一个维度上是连续的，按行复制
        if (nb00 == type_size) {
            for (int64_t i03 = 0; i03 < ne03; i03++) {
                for (int64_t i02 = 0; i02 < ne02; i02++) {
                    id += rs * ir0;
                    for (int64_t i01 = ir0; i01 < ir1; i01++) {
                        const char * src0_ptr = (char *) src0->data + i01*nb01 + i02*nb02 + i03*nb03;
                        memcpy(dst_ptr + id, src0_ptr, rs);
                        id += rs;
                    }
                    id += rs * (ne01 - ir1);
                }
            }
        } else {
            // 如果源数组不是在第一个维度上连续，进行复制
            //printf("%s: this is not optimal - fix me\n", __func__);

            for (int64_t i03 = 0; i03 < ne03; i03++) {
                for (int64_t i02 = 0; i02 < ne02; i02++) {
                    id += rs * ir0;
                    for (int64_t i01 = ir0; i01 < ir1; i01++) {
                        for (int64_t i00 = 0; i00 < ne00; i00++) {
                            const char * src0_ptr = (char *) src0->data + i00*nb00 + i01*nb01 + i02*nb02 + i03*nb03;
                            memcpy(dst_ptr + id, src0_ptr, type_size);

                            id += type_size;
                        }
                    }
                    id += rs * (ne01 - ir1);
                }
            }
        }

        // 返回
        return;
    }

    // 目标数组计数器初始化
    int64_t i10 = 0;
    int64_t i11 = 0;
    int64_t i12 = 0;
    int64_t i13 = 0;
    // 循环遍历第三维度
    for (int64_t i03 = 0; i03 < ne03; i03++) {
        // 循环遍历第二维度
        for (int64_t i02 = 0; i02 < ne02; i02++) {
            // 更新 i10
            i10 += ne00 * ir0;
            // 处理 i10 超过 ne0 的情况
            while (i10 >= ne0) {
                i10 -= ne0;
                // 更新 i11，i12，i13
                if (++i11 == ne1) {
                    i11 = 0;
                    if (++i12 == ne2) {
                        i12 = 0;
                        if (++i13 == ne3) {
                            i13 = 0;
                        }
                    }
                }
            }
            // 循环遍历第一维度
            for (int64_t i01 = ir0; i01 < ir1; i01++) {
                // 循环遍历第零维度
                for (int64_t i00 = 0; i00 < ne00; i00++) {
                    // 计算源指针和目标指针位置
                    const char * src0_ptr = ((char *) src0->data + i00*nb00 + i01*nb01 + i02*nb02 + i03*nb03);
                    char * dst_ptr  = ((char *)  dst->data + i10*nb0  + i11*nb1  + i12*nb2  + i13*nb3);

                    // 复制数据
                    memcpy(dst_ptr, src0_ptr, type_size);

                    // 更新 i10，i11，i12，i13
                    if (++i10 == ne0) {
                        i10 = 0;
                        if (++i11 == ne1) {
                            i11 = 0;
                            if (++i12 == ne2) {
                                i12 = 0;
                                if (++i13 == ne3) {
                                    i13 = 0;
                                }
                            }
                        }
                    }
                }
            }
            // 更新 i10，i11，i12，i13
            i10 += ne00 * (ne01 - ir1);
            // 处理 i10 超过 ne0 的情况
            while (i10 >= ne0) {
                i10 -= ne0;
                // 更新 i11，i12，i13
                if (++i11 == ne1) {
                    i11 = 0;
                    if (++i12 == ne2) {
                        i12 = 0;
                        if (++i13 == ne3) {
                            i13 = 0;
                        }
                    }
                }
            }
        }
    }
}

// 计算前向传播的重复操作
static void ggml_compute_forward_dup(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 如果源张量和目标张量的类型相同，则直接调用重复操作函数
    if (src0->type == dst->type) {
        ggml_compute_forward_dup_bytes(params, src0, dst);
        return;
    }

    // 根据源张量的类型进行不同的操作
    switch (src0->type) {
        case GGML_TYPE_F16:
            {
                ggml_compute_forward_dup_f16(params, src0, dst);
            } break;
        case GGML_TYPE_F32:
            {
                ggml_compute_forward_dup_f32(params, src0, dst);
            } break;
        default:
            {
                // 如果类型不匹配，则断言失败
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_add

// 计算前向传播的加法操作（针对 F32 类型）
static void ggml_compute_forward_add_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 断言源张量 src1 可以重复到与 src0 相同的形状，并且 src0 与 dst 的形状相同
    GGML_ASSERT(ggml_can_repeat(src1, src0) && ggml_are_same_shape(src0, dst));

    // 如果任务类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取当前线程的索引和总线程数
    const int ith = params->ith;
    const int nth = params->nth;

    // 如果使用 CLBLAST 并且源张量 src1 在 GPU 上，则执行相应操作
#ifdef GGML_USE_CLBLAST
    if (src1->backend == GGML_BACKEND_GPU) {
        // TODO: OpenCL kernel support full broadcast
        GGML_ASSERT(ggml_can_repeat_rows(src1, src0));
        if (ith == 0) {
            ggml_cl_add(src0, src1, dst);
        }
        return;
    }
#endif

    // 获取源张量 src0 的行数
    const int nr  = ggml_nrows(src0);

    // 定义一些本地变量

    // 断言每个元素的字节数为 float 类型大小
    GGML_ASSERT( nb0 == sizeof(float));
    GGML_ASSERT(nb00 == sizeof(float));

    // 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);
    # 如果 nb10 等于 float 的大小
    if (nb10 == sizeof(float)) {
        # 遍历 ir0 到 ir1 之间的整数
        for (int ir = ir0; ir < ir1; ++ir) {
            # 计算 i03, i02, i01 分别为 ir 在 ne02*ne01, ne01, 1 的商和余数
            const int64_t i03 = ir/(ne02*ne01);
            const int64_t i02 = (ir - i03*ne02*ne01)/ne01;
            const int64_t i01 = (ir - i03*ne02*ne01 - i02*ne01);

            # 计算 i13, i12, i11 分别为 i03, i02, i01 在 ne13, ne12, ne11 的余数
            const int64_t i13 = i03 % ne13;
            const int64_t i12 = i02 % ne12;
            const int64_t i11 = i01 % ne11;
            # 计算 nr0 为 ne00 除以 ne10 的商
            const int64_t nr0 = ne00 / ne10;

            # 计算 dst_ptr, src0_ptr, src1_ptr 分别为 dst, src0, src1 数据的指针
            float * dst_ptr  = (float *) ((char *) dst->data  + i03*nb3  + i02*nb2  + i01*nb1 );
            float * src0_ptr = (float *) ((char *) src0->data + i03*nb03 + i02*nb02 + i01*nb01);
            float * src1_ptr = (float *) ((char *) src1->data + i13*nb13 + i12*nb12 + i11*nb11);

            # 遍历 nr0 次
            for (int64_t r = 0; r < nr0; ++r) {
#ifdef GGML_USE_ACCELERATE
                // 如果定义了 GGML_USE_ACCELERATE，则使用 vDSP_vadd 函数进行加法运算
                vDSP_vadd(src0_ptr + r*ne10, 1, src1_ptr, 1, dst_ptr + r*ne10, 1, ne10);
#else
                // 如果未定义 GGML_USE_ACCELERATE，则使用 ggml_vec_add_f32 函数进行加法运算
                ggml_vec_add_f32(ne10, dst_ptr + r*ne10, src0_ptr + r*ne10, src1_ptr);
#endif
            }
        }
    } else {
        // 如果 src1 不是连续的
        for (int ir = ir0; ir < ir1; ++ir) {
            // 在 i1、i2、i3 上 src1 可以广播到 src0 和 dst
            const int64_t i03 = ir/(ne02*ne01);
            const int64_t i02 = (ir - i03*ne02*ne01)/ne01;
            const int64_t i01 = (ir - i03*ne02*ne01 - i02*ne01);

            const int64_t i13 = i03 % ne13;
            const int64_t i12 = i02 % ne12;
            const int64_t i11 = i01 % ne11;

            float * dst_ptr  = (float *) ((char *) dst->data  + i03*nb3  + i02*nb2  + i01*nb1 );
            float * src0_ptr = (float *) ((char *) src0->data + i03*nb03 + i02*nb02 + i01*nb01);

            for (int64_t i0 = 0; i0 < ne0; ++i0) {
                const int64_t i10 = i0 % ne10;
                float * src1_ptr = (float *) ((char *) src1->data + i13*nb13 + i12*nb12 + i11*nb11 + i10*nb10);

                // 将 src0_ptr[i0] 和 *src1_ptr 相加，结果存储在 dst_ptr[i0] 中
                dst_ptr[i0] = src0_ptr[i0] + *src1_ptr;
            }
        }
    }
}

static void ggml_compute_forward_add_f16_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 断言 src0、src1 和 dst 具有相同的形状
    GGML_ASSERT(ggml_are_same_shape(src0, src1) && ggml_are_same_shape(src0, dst));

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    const int ith = params->ith;
    const int nth = params->nth;

    const int nr  = ggml_nrows(src0);

    GGML_TENSOR_BINARY_OP_LOCALS

    // 断言 src0 的类型为 GGML_TYPE_F16
    GGML_ASSERT(src0->type == GGML_TYPE_F16);
    // 断言 src1 的类型为 GGML_TYPE_F32
    GGML_ASSERT(src1->type == GGML_TYPE_F32);

    if (dst->type == GGML_TYPE_F32) {
        // 断言 nb0 的大小为 sizeof(float)
        GGML_ASSERT( nb0 == sizeof(float));
    }
    else {
        // 如果目标数据类型为 GGML_TYPE_F16，则断言为真
        GGML_ASSERT(dst->type  == GGML_TYPE_F16);
        // 断言 nb0 的大小为 ggml_fp16_t 的大小
        GGML_ASSERT( nb0 == sizeof(ggml_fp16_t));
    }

    // 断言 nb00 的大小为 ggml_fp16_t 的大小
    GGML_ASSERT(nb00 == sizeof(ggml_fp16_t));

    // 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 该线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 如果 nb10 的大小为 float
    if (nb10 == sizeof(float)) {
        // 如果目标数据类型为 GGML_TYPE_F16
        if (dst->type == GGML_TYPE_F16) {
            for (int ir = ir0; ir < ir1; ++ir) {
                // src0、src1 和 dst 具有相同的形状 => 相同的索引
                const int i3 = ir/(ne2*ne1);
                const int i2 = (ir - i3*ne2*ne1)/ne1;
                const int i1 = (ir - i3*ne2*ne1 - i2*ne1);

                ggml_fp16_t * dst_ptr  = (ggml_fp16_t *) ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1);
                ggml_fp16_t * src0_ptr = (ggml_fp16_t *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01);
                float *       src1_ptr = (float *)       ((char *) src1->data + i3*nb13 + i2*nb12 + i1*nb11);

                for (int i = 0; i < ne0; i++) {
                    dst_ptr[i] = GGML_FP32_TO_FP16(GGML_FP16_TO_FP32(src0_ptr[i]) + src1_ptr[i]);
                }
            }
        } else {
            for (int ir = ir0; ir < ir1; ++ir) {
                // src0、src1 和 dst 具有相同的形状 => 相同的索引
                const int i3 = ir/(ne2*ne1);
                const int i2 = (ir - i3*ne2*ne1)/ne1;
                const int i1 = (ir - i3*ne2*ne1 - i2*ne1);

                float *       dst_ptr  = (float *)       ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1);
                ggml_fp16_t * src0_ptr = (ggml_fp16_t *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01);
                float *       src1_ptr = (float *)       ((char *) src1->data + i3*nb13 + i2*nb12 + i1*nb11);

                for (int i = 0; i < ne0; i++) {
                    dst_ptr[i] = GGML_FP16_TO_FP32(src0_ptr[i]) + src1_ptr[i];
                }
            }
        }
    }
    else {
        // 如果 src1 不是连续的，则触发断言错误
        GGML_ASSERT(false);
    }
// 计算两个 F16 类型张量的加法结果，存储到目标张量中
static void ggml_compute_forward_add_f16_f16(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 断言源张量和目标张量具有相同的形状
    GGML_ASSERT(ggml_are_same_shape(src0, src1) && ggml_are_same_shape(src0, dst));

    // 如果任务类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取当前线程和总线程数
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取源张量的行数
    const int nr  = ggml_nrows(src0);

    // 定义二元操作的本地变量
    GGML_TENSOR_BINARY_OP_LOCALS

    // 断言源张量和目标张量的数据类型为 F16
    GGML_ASSERT(src0->type == GGML_TYPE_F16);
    GGML_ASSERT(src1->type == GGML_TYPE_F16);
    GGML_ASSERT(dst->type  == GGML_TYPE_F16);

    // 断言数据块大小为 F16 类型大小
    GGML_ASSERT( nb0 == sizeof(ggml_fp16_t));
    GGML_ASSERT(nb00 == sizeof(ggml_fp16_t));

    // 计算每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 计算当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 如果数据块大小为 F16 类型大小
    if (nb10 == sizeof(ggml_fp16_t)) {
        for (int ir = ir0; ir < ir1; ++ir) {
            // 计算当前索引在三维张量中的位置
            const int i3 = ir/(ne2*ne1);
            const int i2 = (ir - i3*ne2*ne1)/ne1;
            const int i1 = (ir - i3*ne2*ne1 - i2*ne1);

            // 获取目标张量、源张量0、源张量1 的指针位置
            ggml_fp16_t * dst_ptr  = (ggml_fp16_t *) ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1);
            ggml_fp16_t * src0_ptr = (ggml_fp16_t *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01);
            ggml_fp16_t * src1_ptr = (ggml_fp16_t *) ((char *) src1->data + i3*nb13 + i2*nb12 + i1*nb11);

            // 对每个元素进行加法操作
            for (int i = 0; i < ne0; i++) {
                dst_ptr[i] = GGML_FP32_TO_FP16(GGML_FP16_TO_FP32(src0_ptr[i]) + GGML_FP16_TO_FP32(src1_ptr[i]));
            }
        }
    }
    else {
        // 如果数据块大小不为 F16 类型大小，则抛出断言错误
        GGML_ASSERT(false);
    }
}
// 计算两个输入张量的加法操作，并将结果存储到目标张量中
static void ggml_compute_forward_add_q_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 断言输入张量的形状相同
    GGML_ASSERT(ggml_are_same_shape(src0, src1) && ggml_are_same_shape(src0, dst));

    // 如果任务类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取输入张量的行数
    const int nr  = ggml_nrows(src0);

    // 定义局部变量
    GGML_TENSOR_BINARY_OP_LOCALS

    // 获取当前线程的索引和总线程数
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取输入张量和输出张量的数据类型
    const enum ggml_type type = src0->type;
    const enum ggml_type dtype = dst->type;
    ggml_to_float_t const dequantize_row_q = type_traits[type].to_float;
    ggml_from_float_t const quantize_row_q = type_traits[dtype].from_float;

    // 断言不支持输入张量 src0 或 src1 的置换
    GGML_ASSERT(nb00 == ggml_type_size(type));
    GGML_ASSERT(nb10 == sizeof(float));

    // 断言输出张量 dst 不能被转置或置换
    GGML_ASSERT(nb0 <= nb1);
    GGML_ASSERT(nb1 <= nb2);
    GGML_ASSERT(nb2 <= nb3);

    // 断言输入张量 src0 是量化的，输入张量 src1 是单精度浮点型
    GGML_ASSERT(ggml_is_quantized(src0->type));
    GGML_ASSERT(src1->type == GGML_TYPE_F32);

    // 计算每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 计算当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 计算当前线程的工作数据指针
    float * wdata = (float *) params->wdata + (ne00 + CACHE_LINE_SIZE_F32) * ith;
}
    // 遍历 ir0 到 ir1 之间的整数
    for (int ir = ir0; ir < ir1; ++ir) {
        // 计算 src0 的索引
        const int i03 = ir/(ne02*ne01);
        const int i02 = (ir - i03*ne02*ne01)/ne01;
        const int i01 = (ir - i03*ne02*ne01 - i02*ne01);

        // src1 和 dst 与 src0 具有相同的形状 => 相同的索引
        const int i13 = i03;
        const int i12 = i02;
        const int i11 = i01;

        const int i3 = i03;
        const int i2 = i02;
        const int i1 = i01;

        // 获取 src0、src1 和 dst 对应行的指针
        void  * src0_row = (void *) ((char *) src0->data + (i01*nb01 + i02*nb02 + i03*nb03));
        float * src1_row = (float *)((char *) src1->data + (i11*nb11 + i12*nb12 + i13*nb13));
        void  * dst_row  = (void *) ((char *)  dst->data + ( i1*nb1  +  i2*nb2  +  i3*nb3));

        // 断言 ne00 能被 32 整除
        assert(ne00 % 32 == 0);

        // 从 src0 解量化行到临时缓冲区
        dequantize_row_q(src0_row, wdata, ne00);
        // 将 src1 添加到 wdata
        ggml_vec_acc_f32(ne00, wdata, src1_row);
        // 将行量化到 dst
        if (quantize_row_q != NULL) {
            quantize_row_q(wdata, dst_row, ne00);
        } else {
            // 如果没有量化函数，则直接复制 wdata 到 dst_row
            memcpy(dst_row, wdata, ne0*nb0);
        }
    }
// 定义一个静态函数，用于计算两个张量的加法操作并将结果存储在目标张量中
static void ggml_compute_forward_add(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 根据第一个源张量的数据类型进行不同的处理
    switch (src0->type) {
        // 如果第一个源张量的数据类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 如果第二个源张量的数据类型也为 GGML_TYPE_F32，则调用相应的函数进行计算
                if (src1->type == GGML_TYPE_F32) {
                    ggml_compute_forward_add_f32(params, src0, src1, dst);
                }
                // 如果第二个源张量的数据类型不为 GGML_TYPE_F32，则抛出断言错误
                else {
                    GGML_ASSERT(false);
                }
            } break;
        // 如果第一个源张量的数据类型为 GGML_TYPE_F16
        case GGML_TYPE_F16:
            {
                // 根据第二个源张量的数据类型进行不同的处理
                if (src1->type == GGML_TYPE_F16) {
                    ggml_compute_forward_add_f16_f16(params, src0, src1, dst);
                }
                else if (src1->type == GGML_TYPE_F32) {
                    ggml_compute_forward_add_f16_f32(params, src0, src1, dst);
                }
                else {
                    GGML_ASSERT(false);
                }
            } break;
        // 如果第一个源张量的数据类型为其他类型
        case GGML_TYPE_Q4_0:
        case GGML_TYPE_Q4_1:
        case GGML_TYPE_Q5_0:
        case GGML_TYPE_Q5_1:
        case GGML_TYPE_Q8_0:
        case GGML_TYPE_Q2_K:
        case GGML_TYPE_Q3_K:
        case GGML_TYPE_Q4_K:
        case GGML_TYPE_Q5_K:
        case GGML_TYPE_Q6_K:
        case GGML_TYPE_IQ2_XXS:
        case GGML_TYPE_IQ2_XS:
        case GGML_TYPE_IQ3_XXS:
            {
                // 调用相应的函数进行计算
                ggml_compute_forward_add_q_f32(params, src0, src1, dst);
            } break;
        // 如果第一个源张量的数据类型为其他未知类型，则抛出断言错误
        default:
            {
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_add1

// 定义一个静态函数，用于计算两个浮点型张量的加法操作并将结果存储在目标张量中
static void ggml_compute_forward_add1_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 断言源张量和目标张量具有相同的形状
    GGML_ASSERT(ggml_are_same_shape(src0, dst));
    // 断言第二个源张量为标量
    GGML_ASSERT(ggml_is_scalar(src1));

    // 如果参数类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }
}
    // 获取参数中的ith和nth值
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取src0的行数
    const int nr  = ggml_nrows(src0);

    // 定义一些本地变量
    GGML_TENSOR_UNARY_OP_LOCALS

    // 断言nb0和nb00的大小为float
    GGML_ASSERT( nb0 == sizeof(float));
    GGML_ASSERT(nb00 == sizeof(float));

    // 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 遍历行范围内的每一行
    for (int ir = ir0; ir < ir1; ++ir) {
        // src0和dst具有相同的形状=>相同的索引
        const int i3 = ir/(ne2*ne1);
        const int i2 = (ir - i3*ne2*ne1)/ne1;
        const int i1 = (ir - i3*ne2*ne1 - i2*ne1);
#ifdef GGML_USE_ACCELERATE
        // 如果使用加速库，则不需要使用自定义的向量加法函数
        UNUSED(ggml_vec_add1_f32);

        // 使用 Accelerate 框架中的 vDSP_vadd 函数进行向量加法操作
        vDSP_vadd(
                (float *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01), 1,
                (float *) ((char *) src1->data), 0,
                (float *) ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1 ), 1,
                ne0);
#else
        // 如果不使用加速库，则调用自定义的向量加法函数 ggml_vec_add1_f32 进行向量加法操作
        ggml_vec_add1_f32(ne0,
                (float *) ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1 ),
                (float *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01),
               *(float *) src1->data);
#endif
    }
}

// 计算前向加法操作，将 src0 和 src1 相加得到 dst
static void ggml_compute_forward_add1_f16_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 断言 src0 和 dst 的形状相同
    GGML_ASSERT(ggml_are_same_shape(src0, dst));
    // 断言 src1 是标量
    GGML_ASSERT(ggml_is_scalar(src1));

    // 如果任务类型是初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取标量值
    const float v = *(float *) src1->data;

    // 获取参数中的 ith 和 nth
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取 src0 的行数
    const int nr  = ggml_nrows(src0);

    GGML_TENSOR_UNARY_OP_LOCALS

    // 断言 src0 和 src1 的类型分别为 GGML_TYPE_F16 和 GGML_TYPE_F32，dst 的类型为 GGML_TYPE_F16
    GGML_ASSERT(src0->type == GGML_TYPE_F16);
    GGML_ASSERT(src1->type == GGML_TYPE_F32);
    GGML_ASSERT(dst->type  == GGML_TYPE_F16);

    // 断言 nb0 和 nb00 的大小分别为 ggml_fp16_t 的大小
    GGML_ASSERT( nb0 == sizeof(ggml_fp16_t));
    GGML_ASSERT(nb00 == sizeof(ggml_fp16_t));

    // 计算每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 计算当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);
    // 遍历 ir0 到 ir1 之间的索引
    for (int ir = ir0; ir < ir1; ++ir) {
        // src0 和 dst 具有相同的形状 => 相同的索引
        // 计算三维索引 i3, i2, i1
        const int i3 = ir/(ne2*ne1);
        const int i2 = (ir - i3*ne2*ne1)/ne1;
        const int i1 = (ir - i3*ne2*ne1 - i2*ne1);

        // 计算 dst_ptr 和 src0_ptr 的指针位置
        ggml_fp16_t * dst_ptr  = (ggml_fp16_t *) ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1 );
        ggml_fp16_t * src0_ptr = (ggml_fp16_t *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01);
        
        // 遍历 ne0 个元素
        for (int i = 0; i < ne0; i++) {
            // 将 src0_ptr 中的值转换为 float32，加上 v 后再转换为 float16，赋值给 dst_ptr
            dst_ptr[i] = GGML_FP32_TO_FP16(GGML_FP16_TO_FP32(src0_ptr[i]) + v);
        }
    }
// 计算两个 float16 类型张量的加法，结果存储在目标张量中
static void ggml_compute_forward_add1_f16_f16(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 断言源张量 src0 和目标张量 dst 具有相同的形状
    GGML_ASSERT(ggml_are_same_shape(src0, dst));
    // 断言源张量 src1 是标量
    GGML_ASSERT(ggml_is_scalar(src1));

    // 如果任务类型是初始化或者结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取标量值
    const float v = GGML_FP16_TO_FP32(*(ggml_fp16_t *) src1->data);

    // 获取参数中的索引信息
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取源张量 src0 的行数
    const int nr  = ggml_nrows(src0);

    // 定义一些局部变量
    GGML_TENSOR_UNARY_OP_LOCALS

    // 断言源张量 src0、src1 和目标张量 dst 的类型都是 float16
    GGML_ASSERT(src0->type == GGML_TYPE_F16);
    GGML_ASSERT(src1->type == GGML_TYPE_F16);
    GGML_ASSERT(dst->type  == GGML_TYPE_F16);

    // 断言 nb0 和 nb00 的大小为 float16 类型的大小
    GGML_ASSERT( nb0 == sizeof(ggml_fp16_t));
    GGML_ASSERT(nb00 == sizeof(ggml_fp16_t));

    // 计算每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 计算当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 遍历处理每一行
    for (int ir = ir0; ir < ir1; ++ir) {
        // 计算当前元素在三维张量中的索引
        const int i3 = ir/(ne2*ne1);
        const int i2 = (ir - i3*ne2*ne1)/ne1;
        const int i1 = (ir - i3*ne2*ne1 - i2*ne1);

        // 获取目标张量和源张量的指针
        ggml_fp16_t * dst_ptr  = (ggml_fp16_t *) ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1 );
        ggml_fp16_t * src0_ptr = (ggml_fp16_t *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01);
        // 对每个元素进行加法操作
        for (int i = 0; i < ne0; i++) {
            dst_ptr[i] = GGML_FP32_TO_FP16(GGML_FP16_TO_FP32(src0_ptr[i]) + v);
        }
    }
}

// 计算一个 float32 类型张量和一个标量的加法
static void ggml_compute_forward_add1_q_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 断言源张量 src0 和目标张量 dst 具有相同的形状
    GGML_ASSERT(ggml_are_same_shape(src0, dst));
    // 断言源张量 src1 是标量
    GGML_ASSERT(ggml_is_scalar(src1));
    // 如果任务类型是初始化或者结束，则直接返回，不执行后续操作
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取要添加的标量值
    const float v = *(float *) src1->data;

    // 获取当前线程的索引和总线程数
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取 src0 的行数
    const int nr  = ggml_nrows(src0);

    // 定义一些本地变量
    GGML_TENSOR_UNARY_OP_LOCALS

    // 获取 src0 的数据类型
    const enum ggml_type type = src0->type;
    // 获取将数据从量化到浮点数的函数
    ggml_to_float_t const dequantize_row_q = type_traits[type].to_float;
    // 获取将数据从浮点数到量化的函数
    ggml_from_float_t const quantize_row_q = type_traits[type].from_float;

    // 检查是否支持 src0 的排列
    GGML_ASSERT(nb00 == ggml_type_size(type));

    // 检查目标数据不能被转置或排列
    GGML_ASSERT(nb0 <= nb1);
    GGML_ASSERT(nb1 <= nb2);
    GGML_ASSERT(nb2 <= nb3);

    // 检查 src0 和 dst 的数据类型是否为量化类型，src1 的数据类型是否为 GGML_TYPE_F32
    GGML_ASSERT(ggml_is_quantized(src0->type));
    GGML_ASSERT(dst->type == src0->type);
    GGML_ASSERT(src1->type == GGML_TYPE_F32);

    // 计算每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 计算当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 计算临时缓冲区的起始位置
    float * wdata = (float *) params->wdata + (ne0 + CACHE_LINE_SIZE_F32) * ith;

    // 遍历当前线程处理的行
    for (int ir = ir0; ir < ir1; ++ir) {
        // 计算当前行在三维数组中的索引
        const int i3 = ir/(ne2*ne1);
        const int i2 = (ir - i3*ne2*ne1)/ne1;
        const int i1 = (ir - i3*ne2*ne1 - i2*ne1);

        // 获取当前行在 src0 和 dst 中的地址
        void  * src0_row = (void *) ((char *) src0->data + (i1*nb01 + i2*nb02 + i3*nb03));
        void  * dst_row  = (void *) ((char *)  dst->data + (i1*nb1  + i2*nb2  + i3*nb0 ));

        // 确保 ne0 是 32 的倍数
        assert(ne0 % 32 == 0);

        // 将 src0 行数据从量化转换为浮点数并存储到临时缓冲区
        dequantize_row_q(src0_row, wdata, ne0);
        // 将标量值 v 添加到临时缓冲区中
        ggml_vec_acc1_f32(ne0, wdata, v);
        // 将临时缓冲区中的数据从浮点数转换为量化数据并存储到 dst 中
        quantize_row_q(wdata, dst_row, ne0);
    }
// 计算前向加法操作，根据输入张量的类型选择不同的实现
static void ggml_compute_forward_add1(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 根据 src0 的数据类型进行不同的处理
    switch (src0->type) {
        // 如果数据类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用 GGML_TYPE_F32 类型的前向加法函数
                ggml_compute_forward_add1_f32(params, src0, src1, dst);
            } break;
        // 如果数据类型为 GGML_TYPE_F16
        case GGML_TYPE_F16:
            {
                // 根据 src1 的数据类型选择不同的前向加法函数
                if (src1->type == GGML_TYPE_F16) {
                    ggml_compute_forward_add1_f16_f16(params, src0, src1, dst);
                }
                else if (src1->type == GGML_TYPE_F32) {
                    ggml_compute_forward_add1_f16_f32(params, src0, src1, dst);
                }
                else {
                    GGML_ASSERT(false);
                }
            } break;
        // 如果数据类型为其他类型
        case GGML_TYPE_Q4_0:
        case GGML_TYPE_Q4_1:
        case GGML_TYPE_Q5_0:
        case GGML_TYPE_Q5_1:
        case GGML_TYPE_Q8_0:
        case GGML_TYPE_Q8_1:
        case GGML_TYPE_Q2_K:
        case GGML_TYPE_Q3_K:
        case GGML_TYPE_Q4_K:
        case GGML_TYPE_Q5_K:
        case GGML_TYPE_Q6_K:
        case GGML_TYPE_IQ2_XXS:
        case GGML_TYPE_IQ2_XS:
        case GGML_TYPE_IQ3_XXS:
            {
                // 调用 GGML_TYPE_Q 类型的前向加法函数
                ggml_compute_forward_add1_q_f32(params, src0, src1, dst);
            } break;
        // 默认情况
        default:
            {
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_acc

// 计算前向累加操作，数据类型为 GGML_TYPE_F32
static void ggml_compute_forward_acc_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 断言 src0 和 dst 的形状相同
    GGML_ASSERT(ggml_are_same_shape(src0, dst));
    // 断言 dst 和 src0 是连续的
    GGML_ASSERT(ggml_is_contiguous(dst) && ggml_is_contiguous(src0));

    // 在累加期间使用这些步长和数据偏移字节查看 src0 和 dst
    // nb0 隐式地是 element_size，因为 src0 和 dst 是连续的
    // 从目标操作的参数中获取四个整数值和一个布尔值
    size_t nb1     = ((int32_t *) dst->op_params)[0];
    size_t nb2     = ((int32_t *) dst->op_params)[1];
    size_t nb3     = ((int32_t *) dst->op_params)[2];
    size_t offset  = ((int32_t *) dst->op_params)[3];
    bool   inplace = (bool) ((int32_t *) dst->op_params)[4];

    // 如果不是原地操作且任务类型为初始化，则执行下面的操作
    if (!inplace && (params->type == GGML_TASK_INIT)) {
        // 如果不是第一个线程，则直接返回
        if (params->ith != 0) {
            return;
        }
        // 在初始化阶段需要对 memcpy 进行同步以避免竞争条件
        memcpy(
            ((char *)  dst->data),
            ((char *) src0->data),
            ggml_nbytes(dst));
    }

    // 如果任务类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取线程号和线程总数
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取源张量的行数和列数
    const int nr = ggml_nrows(src1);
    const int nc = src1->ne[0];

    // 定义宏用于获取源张量的元素大小和张量大小
    GGML_TENSOR_LOCALS(int64_t, ne1, src1, ne)
    GGML_TENSOR_LOCALS(size_t,  nb1, src1, nb)

    // 获取 src0 和 dst 在累加期间的视图
    const size_t nb0 = ggml_element_size(src0);

    const size_t nb00 = nb0;
    const size_t nb01 = nb1;
    const size_t nb02 = nb2;
    const size_t nb03 = nb3;

    // 断言确保偏移量加上各维度的偏移量乘以大小不超过目标张量的总大小
    GGML_ASSERT(offset + (ne10 == 0 ? 0 : ne10-1)*nb0  + (ne11 == 0 ? 0 : ne11-1)*nb1  + (ne12 == 0 ? 0 : ne12-1)*nb2  + (ne13 == 0 ? 0 : ne13-1)*nb3  < ggml_nbytes(dst));
    GGML_ASSERT(offset + (ne10 == 0 ? 0 : ne10-1)*nb00 + (ne11 == 0 ? 0 : ne11-1)*nb01 + (ne12 == 0 ? 0 : ne12-1)*nb02 + (ne13 == 0 ? 0 : ne13-1)*nb03 < ggml_nbytes(src0));

    // 断言确保 nb10 的大小为 float 类型的大小
    GGML_ASSERT(nb10 == sizeof(float));

    // 计算每个线程的行数
    const int dr = (nr + nth - 1)/nth;

    // 计算当前线程的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);
    // 遍历 ir0 到 ir1 之间的整数
    for (int ir = ir0; ir < ir1; ++ir) {
        // 将 src0 和 dst 视为具有 src1 的形状和偏移量的视图
        // => 相同的索引
        // 计算在三维数组中的索引 i3
        const int i3 = ir/(ne12*ne11);
        // 计算在三维数组中的索引 i2
        const int i2 = (ir - i3*ne12*ne11)/ne11;
        // 计算在三维数组中的索引 i1
        const int i1 = (ir - i3*ne12*ne11 - i2*ne11);
#ifdef GGML_USE_ACCELERATE
        // 使用 Accelerate 框架中的 vDSP_vadd 函数进行向量加法运算
        vDSP_vadd(
                (float *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01 + offset), 1,
                (float *) ((char *) src1->data + i3*nb13 + i2*nb12 + i1*nb11), 1,
                (float *) ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1  + offset), 1, nc);
#else
        // 使用自定义的 ggml_vec_add_f32 函数进行向量加法运算
        ggml_vec_add_f32(nc,
                (float *) ((char *)  dst->data + i3*nb3  + i2*nb2  + i1*nb1  + offset),
                (float *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01 + offset),
                (float *) ((char *) src1->data + i3*nb13 + i2*nb12 + i1*nb11));
#endif
    }
}

static void ggml_compute_forward_acc(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {

    // 根据 src0 的数据类型进行不同的计算
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                // 调用 ggml_compute_forward_acc_f32 函数进行 F32 类型的计算
                ggml_compute_forward_acc_f32(params, src0, src1, dst);
            } break;
        case GGML_TYPE_F16:
        case GGML_TYPE_Q4_0:
        case GGML_TYPE_Q4_1:
        case GGML_TYPE_Q5_0:
        case GGML_TYPE_Q5_1:
        case GGML_TYPE_Q8_0:
        case GGML_TYPE_Q8_1:
        case GGML_TYPE_Q2_K:
        case GGML_TYPE_Q3_K:
        case GGML_TYPE_Q4_K:
        case GGML_TYPE_Q5_K:
        case GGML_TYPE_Q6_K:
        case GGML_TYPE_IQ2_XXS:
        case GGML_TYPE_IQ2_XS:
        case GGML_TYPE_IQ3_XXS:
        default:
            {
                // 抛出断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_sub

static void ggml_compute_forward_sub_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 断言 ith 等于 0
    assert(params->ith == 0);
    // 断言 src0、src1、dst 的形状相同
    assert(ggml_are_same_shape(src0, src1) && ggml_are_same_shape(src0, dst));

    // 如果任务类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }
    // 获取 src0 张量的行数
    const int nr  = ggml_nrows(src0);

    // 定义二元操作的本地变量
    GGML_TENSOR_BINARY_OP_LOCALS

    // 断言 nb0 的大小为 float 类型
    GGML_ASSERT( nb0 == sizeof(float));
    // 断言 nb00 的大小为 float 类型
    GGML_ASSERT(nb00 == sizeof(float));

    // 如果 nb10 的大小为 float 类型
    if (nb10 == sizeof(float)) {
        // 遍历每一行
        for (int ir = 0; ir < nr; ++ir) {
            // src0, src1 和 dst 具有相同的形状 => 相同的索引
            // 计算三维张量的索引
            const int i3 = ir/(ne2*ne1);
            const int i2 = (ir - i3*ne2*ne1)/ne1;
            const int i1 = (ir - i3*ne2*ne1 - i2*ne1);
#ifdef GGML_USE_ACCELERATE
            // 使用 Accelerate 框架进行向量减法运算
            vDSP_vsub(
                    (float *) ((char *) src1->data + i3*nb13 + i2*nb12 + i1*nb11), 1,
                    (float *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01), 1,
                    (float *) ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1 ), 1,
                    ne0);
#else
            // 使用自定义函数进行向量减法运算
            ggml_vec_sub_f32(ne0,
                    (float *) ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1 ),
                    (float *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01),
                    (float *) ((char *) src1->data + i3*nb13 + i2*nb12 + i1*nb11));
#endif
                // }
            // }
        }
    } else {
        // 当 src1 不是连续的时候
        for (int ir = 0; ir < nr; ++ir) {
            // src0, src1 和 dst 具有相同的形状 => 相同的索引
            const int i3 = ir/(ne2*ne1);
            const int i2 = (ir - i3*ne2*ne1)/ne1;
            const int i1 = (ir - i3*ne2*ne1 - i2*ne1);

            // 获取指向目标、源0、源1的指针
            float * dst_ptr  = (float *) ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1 );
            float * src0_ptr = (float *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01);
            for (int i0 = 0; i0 < ne0; i0++) {
                float * src1_ptr = (float *) ((char *) src1->data + i3*nb13 + i2*nb12 + i1*nb11 + i0*nb10);

                // 计算 dst_ptr[i0] = src0_ptr[i0] - *src1_ptr
                dst_ptr[i0] = src0_ptr[i0] - *src1_ptr;
            }
        }
    }
}

// 根据不同的数据类型调用相应的前向子函数
static void ggml_compute_forward_sub(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                ggml_compute_forward_sub_f32(params, src0, src1, dst);
            } break;
        default:
            {
                // 断言，如果不是 GGML_TYPE_F32 类型，则为假
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_mul
static void ggml_compute_forward_mul_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 断言 src1 可以重复 src0，且 src0 和 dst 具有相同的形状
    GGML_ASSERT(ggml_can_repeat(src1, src0) && ggml_are_same_shape(src0, dst));

    // 如果参数类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }
    // 获取参数中的 ith 和 nth
    const int ith = params->ith;
    const int nth = params->nth;

#if defined(GGML_USE_CLBLAST)
    // 如果使用 GGML_USE_CLBLAST 并且 src1 的后端为 GGML_BACKEND_GPU
    if (src1->backend == GGML_BACKEND_GPU) {
        // TODO: OpenCL kernel support full broadcast
        // 断言 src1 可以重复 src0 的行
        GGML_ASSERT(ggml_can_repeat_rows(src1, src0));
        // 如果 ith 为 0，则调用 ggml_cl_mul 函数
        if (ith == 0) {
            ggml_cl_mul(src0, src1, dst);
        }
        return;
    }
#endif

    // 获取 src0 的行数
    const int64_t nr = ggml_nrows(src0);

    // 定义 GGML_TENSOR_BINARY_OP_LOCALS

    // 断言 nb0 的大小为 float 类型的大小
    GGML_ASSERT( nb0 == sizeof(float));
    // 断言 nb00 的大小为 float 类型的大小

    if (nb10 == sizeof(float)) {
        // 遍历 src0 的行
        for (int64_t ir = ith; ir < nr; ir += nth) {
            // 计算索引
            const int64_t i03 = ir/(ne02*ne01);
            const int64_t i02 = (ir - i03*ne02*ne01)/ne01;
            const int64_t i01 = (ir - i03*ne02*ne01 - i02*ne01);

            const int64_t i13 = i03 % ne13;
            const int64_t i12 = i02 % ne12;
            const int64_t i11 = i01 % ne11;
            const int64_t nr0 = ne00 / ne10;

            // 获取指针
            float * dst_ptr  = (float *) ((char *) dst->data  + i03*nb3  + i02*nb2  + i01*nb1 );
            float * src0_ptr = (float *) ((char *) src0->data + i03*nb03 + i02*nb02 + i01*nb01);
            float * src1_ptr = (float *) ((char *) src1->data + i13*nb13 + i12*nb12 + i11*nb11);

            // 遍历 nr0
            for (int64_t r = 0 ; r < nr0; ++r) {
#ifdef GGML_USE_ACCELERATE
                UNUSED(ggml_vec_mul_f32);

                // 使用 vDSP_vmul 函数进行向量相乘
                vDSP_vmul(src0_ptr + r*ne10, 1, src1_ptr, 1, dst_ptr + r*ne10, 1, ne10);
#else
                // 使用 ggml_vec_mul_f32 函数进行向量相乘
                ggml_vec_mul_f32(ne10, dst_ptr + r*ne10, src0_ptr + r*ne10, src1_ptr);
#else
            }
        }
    } else {
        // 如果 src1 不是连续的
        for (int64_t ir = ith; ir < nr; ir += nth) {
            // src0 和 dst 具有相同的形状 => 相同的索引
            // src1 在 i1、i2、i3 上可以广播到 src0 和 dst
            const int64_t i03 = ir/(ne02*ne01);
            const int64_t i02 = (ir - i03*ne02*ne01)/ne01;
            const int64_t i01 = (ir - i03*ne02*ne01 - i02*ne01);

            const int64_t i13 = i03 % ne13;
            const int64_t i12 = i02 % ne12;
            const int64_t i11 = i01 % ne11;

            float * dst_ptr  = (float *) ((char *) dst->data  + i03*nb3  + i02*nb2  + i01*nb1 );
            float * src0_ptr = (float *) ((char *) src0->data + i03*nb03 + i02*nb02 + i01*nb01);

            for (int64_t i0 = 0; i0 < ne00; ++i0) {
                const int64_t i10 = i0 % ne10;
                float * src1_ptr = (float *) ((char *) src1->data + i13*nb13 + i12*nb12 + i11*nb11 + i10*nb10);

                dst_ptr[i0] = src0_ptr[i0] * (*src1_ptr);
            }
        }
    }
}

static void ggml_compute_forward_mul(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    GGML_ASSERT(src1->type == GGML_TYPE_F32 && "only f32 src1 supported for now");

    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                ggml_compute_forward_mul_f32(params, src0, src1, dst);
            } break;
        default:
            {
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_div

static void ggml_compute_forward_div_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    GGML_ASSERT(ggml_can_repeat(src1, src0) && ggml_are_same_shape(src0, dst));
    // 如果任务类型是初始化或者结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取当前线程的索引和总线程数
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取输入张量 src0 的行数
    const int64_t nr = ggml_nrows(src0);

    // 定义一些本地变量
    GGML_TENSOR_BINARY_OP_LOCALS

    // 断言输入张量的数据类型为 float
    GGML_ASSERT( nb0 == sizeof(float));
    GGML_ASSERT(nb00 == sizeof(float));

    // 如果输入张量 src1 的数据类型为 float
    if (nb10 == sizeof(float)) {
        // 遍历输入张量 src0 的行
        for (int64_t ir = ith; ir < nr; ir += nth) {
            // 计算当前行在三维张量中的索引
            const int64_t i03 = ir/(ne02*ne01);
            const int64_t i02 = (ir - i03*ne02*ne01)/ne01;
            const int64_t i01 = (ir - i03*ne02*ne01 - i02*ne01);

            // 计算当前行在四维张量中的索引
            const int64_t i13 = i03 % ne13;
            const int64_t i12 = i02 % ne12;
            const int64_t i11 = i01 % ne11;
            const int64_t nr0 = ne00 / ne10;

            // 计算目标张量、src0 张量和 src1 张量的指针位置
            float * dst_ptr  = (float *) ((char *) dst->data  + i03*nb3  + i02*nb2  + i01*nb1 );
            float * src0_ptr = (float *) ((char *) src0->data + i03*nb03 + i02*nb02 + i01*nb01);
            float * src1_ptr = (float *) ((char *) src1->data + i13*nb13 + i12*nb12 + i11*nb11);

            // 遍历当前行的元素
            for (int64_t r = 0; r < nr0; ++r) {
// 如果定义了 GGML_USE_ACCELERATE 宏
#ifdef GGML_USE_ACCELERATE
                // 未使用 ggml_vec_div_f32 函数
                UNUSED(ggml_vec_div_f32);

                // 使用 vDSP_vdiv 函数进行向量除法操作
                vDSP_vdiv(src1_ptr, 1, src0_ptr + r*ne10, 1, dst_ptr + r*ne10, 1, ne10);
#else
                // 使用 ggml_vec_div_f32 函数进行向量除法操作
                ggml_vec_div_f32(ne10, dst_ptr + r*ne10, src0_ptr + r*ne10, src1_ptr);
#endif
            }
        }
    } else {
        // 如果 src1 不是连续的
        for (int64_t ir = ith; ir < nr; ir += nth) {
            // src0 和 dst 具有相同的形状 => 相同的索引
            // src1 在 i1、i2、i3 上可广播到 src0 和 dst
            const int64_t i03 = ir/(ne02*ne01);
            const int64_t i02 = (ir - i03*ne02*ne01)/ne01;
            const int64_t i01 = (ir - i03*ne02*ne01 - i02*ne01);

            const int64_t i13 = i03 % ne13;
            const int64_t i12 = i02 % ne12;
            const int64_t i11 = i01 % ne11;

            // 计算指针位置
            float * dst_ptr  = (float *) ((char *) dst->data  + i03*nb3  + i02*nb2  + i01*nb1 );
            float * src0_ptr = (float *) ((char *) src0->data + i03*nb03 + i02*nb02 + i01*nb01);

            for (int64_t i0 = 0; i0 < ne00; ++i0) {
                const int64_t i10 = i0 % ne10;
                float * src1_ptr = (float *) ((char *) src1->data + i13*nb13 + i12*nb12 + i11*nb11 + i10*nb10);

                // 执行除法操作
                dst_ptr[i0] = src0_ptr[i0] / (*src1_ptr);
            }
        }
    }
}

// 计算前向除法操作
static void ggml_compute_forward_div(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                // 调用 ggml_compute_forward_div_f32 函数进行前向除法操作
                ggml_compute_forward_div_f32(params, src0, src1, dst);
            } break;
        default:
            {
                // 断言，如果不是 GGML_TYPE_F32 类型则报错
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_sqr

// 计算前向平方操作，针对 float 类型数据
static void ggml_compute_forward_sqr_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    # 确保参数中的ith为0
    assert(params->ith == 0);
    # 确保src0和dst具有相同的形状
    assert(ggml_are_same_shape(src0, dst));

    # 如果任务类型为初始化或者结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    # 获取src0的行数n和列数nc
    const int n     = ggml_nrows(src0);
    const int nc    = src0->ne[0];

    # 确保dst和src0的数据类型为float
    assert( dst->nb[0] == sizeof(float));
    assert(src0->nb[0] == sizeof(float));

    # 遍历n行数据，对每行数据进行平方操作
    for (int i = 0; i < n; i++) {
        ggml_vec_sqr_f32(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1])));
    }
// 计算前向平方操作，根据输入参数和张量类型调用相应的函数
static void ggml_compute_forward_sqr(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据源张量的类型进行不同的操作
    switch (src0->type) {
        // 如果源张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用针对 GGML_TYPE_F32 类型的前向平方操作函数
                ggml_compute_forward_sqr_f32(params, src0, dst);
            } break;
        // 如果源张量类型不是 GGML_TYPE_F32
        default:
            {
                // 断言失败
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_sqrt

// 计算前向平方根操作，针对 GGML_TYPE_F32 类型的张量
static void ggml_compute_forward_sqrt_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数的 ith 值为 0
    assert(params->ith == 0);
    // 断言源张量和目标张量具有相同的形状
    assert(ggml_are_same_shape(src0, dst));

    // 如果参数类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取源张量的行数和通道数
    const int n  = ggml_nrows(src0);
    const int nc = src0->ne[0];

    // 断言目标张量和源张量的数据类型为 float
    assert( dst->nb[0] == sizeof(float));
    assert(src0->nb[0] == sizeof(float));

    // 遍历每一行，对每个元素进行平方根操作
    for (int i = 0; i < n; i++) {
        ggml_vec_sqrt_f32(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1]));
    }
}

// 计算前向平方根操作，根据输入参数和张量类型调用相应的函数
static void ggml_compute_forward_sqrt(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据源张量的类型进行不同的操作
    switch (src0->type) {
        // 如果源张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用针对 GGML_TYPE_F32 类型的前向平方根操作函数
                ggml_compute_forward_sqrt_f32(params, src0, dst);
            } break;
        // 如果源张量类型不是 GGML_TYPE_F32
        default:
            {
                // 断言失败
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_log

// 计算前向对数操作，针对 GGML_TYPE_F32 类型的张量
static void ggml_compute_forward_log_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数的 ith 值为 0
    GGML_ASSERT(params->ith == 0);
    // 断言源张量和目标张量具有相同的形状
    GGML_ASSERT(ggml_are_same_shape(src0, dst));

    // 如果参数类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }
    // 获取源数据矩阵的行数
    const int n  = ggml_nrows(src0);
    // 获取源数据矩阵的列数
    const int nc = src0->ne[0];

    // 断言目标数据矩阵的每个元素大小为 float 类型
    GGML_ASSERT( dst->nb[0] == sizeof(float));
    // 断言源数据矩阵的每个元素大小为 float 类型
    GGML_ASSERT(src0->nb[0] == sizeof(float));

    // 遍历源数据矩阵的每一行
    for (int i = 0; i < n; i++) {
        // 对每一行进行对数运算，结果存储在目标数据矩阵中
        ggml_vec_log_f32(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1])));
    }
// 计算前向传播的对数操作，根据输入参数和张量类型选择对应的计算函数
static void ggml_compute_forward_log(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的操作
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                // 调用对应的 float 类型计算函数
                ggml_compute_forward_log_f32(params, src0, dst);
            } break;
        default:
            {
                // 断言错误，不支持的张量类型
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_sum

// 计算 float 类型的前向传播求和操作
static void ggml_compute_forward_sum_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数 ith 为 0
    assert(params->ith == 0);
    // 断言目标张量为标量
    assert(ggml_is_scalar(dst));

    // 如果参数类型为初始化或结束任务，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 断言目标张量为标量
    assert(ggml_is_scalar(dst));
    // 断言输入张量的第一个维度大小为 float 类型大小
    assert(src0->nb[0] == sizeof(float));

    // 定义局部变量 ne0 和 nb0
    GGML_TENSOR_LOCALS(int64_t, ne0, src0, ne)
    GGML_TENSOR_LOCALS(size_t,  nb0, src0, nb)

    // 初始化 sum 和 row_sum
    ggml_float sum     = 0;
    ggml_float row_sum = 0;

    // 循环计算张量数据的和
    for (int64_t i03 = 0; i03 < ne03; i03++) {
        for (int64_t i02 = 0; i02 < ne02; i02++) {
            for (int64_t i01 = 0; i01 < ne01; i01++) {
                // 调用向量求和函数，更新 row_sum
                ggml_vec_sum_f32_ggf(ne00,
                        &row_sum,
                        (float *) ((char *) src0->data + i01*nb01 + i02*nb02 + i03*nb03));
                // 累加 row_sum 到 sum
                sum += row_sum;
            }
        }
    }
    // 将计算结果存入目标张量的数据中
    ((float *) dst->data)[0] = sum;
}

// 计算 ggml_fp16_t 类型的前向传播求和操作
static void ggml_compute_forward_sum_f16(
    const struct ggml_compute_params * params,
    const struct ggml_tensor * src0,
          struct ggml_tensor * dst) {
    // 断言参数 ith 为 0
    assert(params->ith == 0);
    // 断言目标张量为标量
    assert(ggml_is_scalar(dst));

    // 如果参数类型为初始化或结束任务，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 断言输入张量的第一个维度大小为 ggml_fp16_t 类型大小
    assert(src0->nb[0] == sizeof(ggml_fp16_t));

    // 定义局部变量 ne0 和 nb0
    GGML_TENSOR_LOCALS(int64_t, ne0, src0, ne)
    GGML_TENSOR_LOCALS(size_t,  nb0, src0, nb)

    // 初始化 sum 和 row_sum
    float sum = 0;
    float row_sum = 0;
    // 循环遍历三维数组，计算每个元素的和
    for (int64_t i03 = 0; i03 < ne03; i03++) {
        for (int64_t i02 = 0; i02 < ne02; i02++) {
            for (int64_t i01 = 0; i01 < ne01; i01++) {
                // 调用函数计算指定位置的元素和，并将结果存储在row_sum中
                ggml_vec_sum_f16_ggf(ne00,
                    &row_sum,
                    (ggml_fp16_t *) ((char *) src0->data + i01 * nb01 + i02 * nb02 + i03 * nb03));
                // 将每行的和累加到总和sum中
                sum += row_sum;
            }
        }
    }
    // 将总和sum转换为fp16类型，并存储在目标数组的第一个元素中
    ((ggml_fp16_t *) dst->data)[0] = GGML_FP32_TO_FP16(sum);
// 计算前向求和操作，根据输入张量类型调用相应的函数
static void ggml_compute_forward_sum(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量类型进行不同的操作
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用针对 GGML_TYPE_F32 类型的前向求和函数
                ggml_compute_forward_sum_f32(params, src0, dst);
            } break;
        // 如果输入张量类型为 GGML_TYPE_F16
        case GGML_TYPE_F16:
            {
                // 调用针对 GGML_TYPE_F16 类型的前向求和函数
                ggml_compute_forward_sum_f16(params, src0, dst);
            } break;
        // 默认情况下
        default:
            {
                // 断言失败
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_sum_rows

// 针对 GGML_TYPE_F32 类型的前向求和行操作
static void ggml_compute_forward_sum_rows_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言 ith 等于 0
    GGML_ASSERT(params->ith == 0);

    // 如果任务类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 断言输入张量 src0 的第一个维度大小为 sizeof(float)
    GGML_ASSERT(src0->nb[0] == sizeof(float));
    // 断言输出张量 dst 的第一个维度大小为 sizeof(float)
    GGML_ASSERT(dst->nb[0] == sizeof(float));

    // 定义一些本地变量
    GGML_TENSOR_UNARY_OP_LOCALS

    // 断言 ne0 等于 1
    GGML_ASSERT(ne0 == 1);
    // 断言 ne1 等于 ne01
    GGML_ASSERT(ne1 == ne01);
    // 断言 ne2 等于 ne02
    GGML_ASSERT(ne2 == ne02);
    // 断言 ne3 等于 ne03
    GGML_ASSERT(ne3 == ne03);

    // 遍历张量的维度进行求和操作
    for (int64_t i3 = 0; i3 < ne03; i3++) {
        for (int64_t i2 = 0; i2 < ne02; i2++) {
            for (int64_t i1 = 0; i1 < ne01; i1++) {
                // 计算当前行的起始位置
                float * src_row = (float *) ((char *) src0->data + i1*nb01 + i2*nb02 + i3*nb03);
                float * dst_row = (float *) ((char *) dst->data  + i1*nb1  + i2*nb2  + i3*nb3);
                float row_sum = 0;
                // 对当前行进行求和操作
                ggml_vec_sum_f32(ne00, &row_sum, src_row);
                // 将求和结果存入目标张量的对应位置
                dst_row[0] = row_sum;
            }
        }
    }
}

// 针对 GGML_TYPE_F32 类型的前向求和行操作
static void ggml_compute_forward_sum_rows(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    # 根据 src0 的类型进行不同的操作
    switch (src0->type) {
        # 如果 src0 的类型是 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                # 调用 ggml_compute_forward_sum_rows_f32 函数，计算并存储结果到 dst
                ggml_compute_forward_sum_rows_f32(params, src0, dst);
            } break;
        # 如果 src0 的类型不是 GGML_TYPE_F32
        default:
            {
                # 断言，表示出现了不应该出现的情况
                GGML_ASSERT(false);
            } break;
    }
// 计算前向均值操作的函数，针对单精度浮点数类型的数据
static void ggml_compute_forward_mean_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数中的 ith 值为 0
    assert(params->ith == 0);

    // 如果任务类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 断言输入数据 src0 的第一个维度的大小为单精度浮点数的大小
    assert(src0->nb[0] == sizeof(float));

    // 定义一些本地变量
    GGML_TENSOR_UNARY_OP_LOCALS

    // 断言 ne0 为 1
    assert(ne0 == 1);
    // 断言 ne1 等于 ne01
    assert(ne1 == ne01);
    // 断言 ne2 等于 ne02
    assert(ne2 == ne02);
    // 断言 ne3 等于 ne03
    assert(ne3 == ne03);

    // 使用 UNUSED 宏标记未使用的变量
    UNUSED(ne0);
    UNUSED(ne1);
    UNUSED(ne2);
    UNUSED(ne3);

    // 循环遍历数据，计算均值并更新目标数据
    for (int64_t i03 = 0; i03 < ne03; i03++) {
        for (int64_t i02 = 0; i02 < ne02; i02++) {
            for (int64_t i01 = 0; i01 < ne01; i01++) {
                // 计算 src0 和 dst 对应位置的数据之和
                ggml_vec_sum_f32(ne00,
                        (float *) ((char *)  dst->data + i01*nb1  + i02*nb2  + i03*nb3),
                        (float *) ((char *) src0->data + i01*nb01 + i02*nb02 + i03*nb03));

                // 将 dst 对应位置的数据除以 ne00，得到均值
                *(float *) ((char *) dst->data + i01*nb1 + i02*nb2 + i03*nb3) /= (float) ne00;
            }
        }
    }
}

// 根据输入数据类型选择对应的前向均值计算函数
static void ggml_compute_forward_mean(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                ggml_compute_forward_mean_f32(params, src0, dst);
            } break;
        default:
            {
                // 断言选择的数据类型不支持
                GGML_ASSERT(false);
            } break;
    }
}

// 计算前向 argmax 操作的函数，针对单精度浮点数类型的数据
static void ggml_compute_forward_argmax_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数中的 ith 值为 0
    assert(params->ith == 0);

    // 如果任务类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 断言输入数据 src0 和目标数据 dst 的第一个维度的大小为单精度浮点数的大小
    assert(src0->nb[0] == sizeof(float));
    assert(dst->nb[0] == sizeof(float));

    // 获取输入数据的第一个维度大小
    const int64_t ne00 = src0->ne[0];
    // 获取源数据的第二维大小
    const int64_t ne01 = src0->ne[1];

    // 获取源数据的第二维字节数
    const size_t nb01 = src0->nb[1];
    
    // 获取目标数据的第一维字节数
    const size_t nb0 = dst->nb[0];

    // 遍历源数据的第二维
    for (int64_t i1 = 0; i1 < ne01; i1++) {
        // 获取当前源数据的指针
        float * src = (float *) ((char *) src0->data + i1*nb01);
        
        // 获取当前目标数据的指针
        int32_t * dst_ = (int32_t *) ((char *)  dst->data + i1*nb0);
        
        // 初始化变量 v 为 0
        int v = 0;
        
        // 在当前源数据中找到最大值的索引，并将其存储在 v 中
        ggml_vec_argmax_f32(ne00, &v, src);
        
        // 将 v 的值存储到目标数据中
        dst_[0] = v;
    }
// 结束函数 ggml_compute_forward_argmax
static void ggml_compute_forward_argmax(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据源张量的类型进行不同的操作
    switch (src0->type) {
        // 如果源张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用 ggml_compute_forward_argmax_f32 函数处理
                ggml_compute_forward_argmax_f32(params, src0, dst);
            } break;
        // 如果源张量类型不在已知范围内
        default:
            {
                // 断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_repeat

// 处理 GGML_TYPE_F32 类型的张量的重复计算
static void ggml_compute_forward_repeat_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言 ith 值为 0
    GGML_ASSERT(params->ith == 0);
    // 断言 src0 和 dst 可以重复
    GGML_ASSERT(ggml_can_repeat(src0, dst));

    // 如果任务类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 定义本地变量

    // 由于 ggml_can_repeat 中的检查，nr0 保证为整数
    const int nr0 = (int)(ne0/ne00);
    const int nr1 = (int)(ne1/ne01);
    const int nr2 = (int)(ne2/ne02);
    const int nr3 = (int)(ne3/ne03);

    // TODO: 支持转置/置换张量
    GGML_ASSERT(nb0  == sizeof(float));
    GGML_ASSERT(nb00 == sizeof(float));

    // TODO: 可能这不是最佳选择？
}
    # 循环遍历第一维度的数据
    for (int i3 = 0; i3 < nr3;  i3++) {
        # 循环遍历第一维度的扩展数据
        for (int k3 = 0; k3 < ne03; k3++) {
            # 循环遍历第二维度的数据
            for (int i2 = 0; i2 < nr2;  i2++) {
                # 循环遍历第二维度的扩展数据
                for (int k2 = 0; k2 < ne02; k2++) {
                    # 循环遍历第三维度的数据
                    for (int i1 = 0; i1 < nr1;  i1++) {
                        # 循环遍历第三维度的扩展数据
                        for (int k1 = 0; k1 < ne01; k1++) {
                            # 循环遍历第四维度的数据
                            for (int i0 = 0; i0 < nr0;  i0++) {
                                # 调用函数将源数据复制到目标数据
                                ggml_vec_cpy_f32(ne00,
                                        (float *) ((char *)  dst->data + (i3*ne03 + k3)*nb3  + (i2*ne02 + k2)*nb2  + (i1*ne01 + k1)*nb1  + (i0*ne00)*nb0),
                                        (float *) ((char *) src0->data + (          k3)*nb03 + (          k2)*nb02 + (          k1)*nb01));
                            }
                        }
                    }
                }
            }
        }
    }
// 计算前向传播的重复操作，将源张量中的数据复制到目标张量中
static void ggml_compute_forward_repeat_f16(
        const struct ggml_compute_params * params,  // 接收计算参数的指针
        const struct ggml_tensor * src0,  // 源张量的指针
        struct ggml_tensor * dst) {  // 目标张量的指针
    // 断言参数中的 ith 值为 0
    GGML_ASSERT(params->ith == 0);
    // 断言源张量和目标张量可以重复
    GGML_ASSERT(ggml_can_repeat(src0, dst));

    // 如果参数中的任务类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    GGML_TENSOR_UNARY_OP_LOCALS

    // 根据源张量和目标张量的维度计算重复次数
    const int nr0 = (int)(ne0/ne00);
    const int nr1 = (int)(ne1/ne01);
    const int nr2 = (int)(ne2/ne02);
    const int nr3 = (int)(ne3/ne03);

    // 断言每个元素的大小为 ggml_fp16_t 类型
    GGML_ASSERT(nb0  == sizeof(ggml_fp16_t));
    GGML_ASSERT(nb00 == sizeof(ggml_fp16_t));

    // 循环遍历源张量中的数据，并复制到目标张量中
    for (int i3 = 0; i3 < nr3;  i3++) {
        for (int k3 = 0; k3 < ne03; k3++) {
            for (int i2 = 0; i2 < nr2;  i2++) {
                for (int k2 = 0; k2 < ne02; k2++) {
                    for (int i1 = 0; i1 < nr1;  i1++) {
                        for (int k1 = 0; k1 < ne01; k1++) {
                            for (int i0 = 0; i0 < nr0;  i0++) {
                                // 计算目标张量和源张量中数据的地址，并进行数据复制
                                ggml_fp16_t * y = (ggml_fp16_t *) ((char *)  dst->data + (i3*ne03 + k3)*nb3  + (i2*ne02 + k2)*nb2  + (i1*ne01 + k1)*nb1  + (i0*ne00)*nb0);
                                ggml_fp16_t * x = (ggml_fp16_t *) ((char *) src0->data + (          k3)*nb03 + (          k2)*nb02 + (          k1)*nb01);
                                // ggml_vec_cpy_f16(ne00, y, x)
                                for (int i = 0; i < ne00; ++i) {
                                    y[i]  = x[i];
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
// 计算前向重复操作，根据输入参数和张量类型选择相应的函数进行计算
static void ggml_compute_forward_repeat(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的操作
    switch (src0->type) {
        case GGML_TYPE_F16:
        case GGML_TYPE_I16:
            {
                // 调用针对 F16 和 I16 类型的前向重复计算函数
                ggml_compute_forward_repeat_f16(params, src0, dst);
            } break;
        case GGML_TYPE_F32:
        case GGML_TYPE_I32:
            {
                // 调用针对 F32 和 I32 类型的前向重复计算函数
                ggml_compute_forward_repeat_f32(params, src0, dst);
            } break;
        default:
            {
                // 默认情况下断言失败
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_repeat_back

// 针对 F32 类型的前向重复计算函数
static void ggml_compute_forward_repeat_back_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言当前操作是第一个操作
    GGML_ASSERT(params->ith == 0);
    // 断言目标张量可以重复
    GGML_ASSERT(ggml_can_repeat(dst, src0));

    // 如果当前操作是初始化或结束操作，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    GGML_TENSOR_UNARY_OP_LOCALS

    // 由于在 ggml_can_repeat 中进行了检查，这里保证是整数
    const int nr0 = (int)(ne00/ne0);
    const int nr1 = (int)(ne01/ne1);
    const int nr2 = (int)(ne02/ne2);
    const int nr3 = (int)(ne03/ne3);

    // TODO: 支持转置/置换张量
    GGML_ASSERT(nb0  == sizeof(float));
    GGML_ASSERT(nb00 == sizeof(float));

    // 如果目标张量是连续的，则将其所有元素设置为0
    if (ggml_is_contiguous(dst)) {
        ggml_vec_set_f32(ne0*ne1*ne2*ne3, dst->data, 0);
    } else {
        // 否则，按照非连续的方式设置目标张量的元素为0
        for         (int k3 = 0; k3 < ne3; k3++) {
            for     (int k2 = 0; k2 < ne2; k2++) {
                for (int k1 = 0; k1 < ne1; k1++) {
                    ggml_vec_set_f32(ne0,
                        (float *) ((char *) dst->data + k1*nb1 + k2*nb2 + k3*nb3),
                        0);
                }
            }
        }
    }

    // TODO: 可能这不是最优解？
}
    // 循环遍历第一维度
    for (int i3 = 0; i3 < nr3; i3++) {
        // 循环遍历第一维度的元素
        for (int k3 = 0; k3 < ne3; k3++) {
            // 循环遍历第二维度
            for (int i2 = 0; i2 < nr2; i2++) {
                // 循环遍历第二维度的元素
                for (int k2 = 0; k2 < ne2; k2++) {
                    // 循环遍历第三维度
                    for (int i1 = 0; i1 < nr1; i1++) {
                        // 循环遍历第三维度的元素
                        for (int k1 = 0; k1 < ne1; k1++) {
                            // 循环遍历第四维度
                            for (int i0 = 0; i0 < nr0; i0++) {
                                // 调用 ggml_vec_acc_f32 函数，传入参数
                                ggml_vec_acc_f32(ne0,
                                        // 计算目标数据指针位置
                                        (float *) ((char *)  dst->data + (         k3)*nb3  + (         k2)*nb2  + (         k1)*nb1),
                                        // 计算源数据指针位置
                                        (float *) ((char *) src0->data + (i3*ne3 + k3)*nb03 + (i2*ne2 + k2)*nb02 + (i1*ne1 + k1)*nb01 + (i0*ne0)*nb00));
                            }
                        }
                    }
                }
            }
        }
    }
// 结束函数 ggml_compute_forward_repeat_back
static void ggml_compute_forward_repeat_back(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的操作
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用相应的函数处理 GGML_TYPE_F32 类型的张量
                ggml_compute_forward_repeat_back_f32(params, src0, dst);
            } break;
        // 如果输入张量类型不是 GGML_TYPE_F32
        default:
            {
                // 断言错误，终止程序
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_concat

// 处理 GGML_TYPE_F32 类型的张量拼接操作
static void ggml_compute_forward_concat_f32(
    const struct ggml_compute_params * params,
    const struct ggml_tensor * src0,
    const struct ggml_tensor * src1,
    struct ggml_tensor * dst) {

    // 如果参数类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 断言 src0 的第一个维度大小为 sizeof(float)
    GGML_ASSERT(src0->nb[0] == sizeof(float));

    // 获取参数中的 ith 和 nth
    const int ith = params->ith;
    const int nth = params->nth;

    // 定义 GGML_TENSOR_BINARY_OP_LOCALS

    // TODO: 支持转置/置换张量
    // 断言 nb0、nb00、nb10 的大小为 sizeof(float)
    GGML_ASSERT(nb0  == sizeof(float));
    GGML_ASSERT(nb00 == sizeof(float));
    GGML_ASSERT(nb10 == sizeof(float));
}
    // 遍历第三维度数据
    for (int i3 = 0; i3 < ne3; i3++) {
        // 遍历第二维度数据
        for (int i2 = ith; i2 < ne2; i2 += nth) {
            // 检查是否在 src0 范围内
            if (i2 < ne02) { // src0
                // 遍历第一维度数据
                for (int i1 = 0; i1 < ne1; i1++) {
                    // 遍历零维度数据
                    for (int i0 = 0; i0 < ne0; i0++) {
                        // 计算 src0 中的偏移量，获取指向数据的指针 x
                        const float * x = (float *)((char *) src0->data + i0 * nb00 + i1 * nb01 + i2 * nb02 + i3 * nb03);

                        // 计算目标数据中的偏移量，获取指向数据的指针 y
                        float * y = (float *)((char *)dst->data + i0 * nb0 + i1 * nb1 + i2 * nb2 + i3 * nb3);
                        // 将 x 指向的数据赋值给 y 指向的数据
                        *y = *x;
                    }
                }
            } // src1
            else {
                // 遍历第一维度数据
                for (int i1 = 0; i1 < ne1; i1++) {
                    // 遍历零维度数据
                    for (int i0 = 0; i0 < ne0; i0++) {
                        // 计算 src1 中的偏移量，获取指向数据的指针 x
                        const float * x = (float *)((char *) src1->data + i0 * nb10 + i1 * nb11 + (i2 - ne02) * nb12 + i3 * nb13);

                        // 计算目标数据中的偏移量，获取指向数据的指针 y
                        float * y = (float *)((char *)dst->data + i0 * nb0 + i1 * nb1 + i2 * nb2 + i3 * nb3);
                        // 将 x 指向的数据赋值给 y 指向的数据
                        *y = *x;
                    }
                }
            }
        }
    }
// 计算拼接操作的前向传播，根据输入张量的类型调用相应的函数
static void ggml_compute_forward_concat(
    const struct ggml_compute_params* params,
    const struct ggml_tensor* src0,
    const struct ggml_tensor* src1,
    struct ggml_tensor* dst) {
    // 根据第一个输入张量的类型进行分支处理
    switch (src0->type) {
        // 如果类型为 GGML_TYPE_F32 或 GGML_TYPE_I32
        case GGML_TYPE_F32:
        case GGML_TYPE_I32:
            {
                // 调用相应的函数进行拼接操作
                ggml_compute_forward_concat_f32(params, src0, src1, dst);
            } break;
        // 如果类型不在上述范围内
        default:
            {
                // 断言失败
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_abs

// 计算绝对值操作的前向传播，针对浮点数类型的输入张量
static void ggml_compute_forward_abs_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数 ith 为 0
    assert(params->ith == 0);
    // 断言输入张量 src0 和输出张量 dst 具有相同的形状
    assert(ggml_are_same_shape(src0, dst));

    // 如果任务类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取输入张量的行数和列数
    const int n  = ggml_nrows(src0);
    const int nc = src0->ne[0];

    // 断言输出张量和输入张量的数据类型为 float
    assert(dst->nb[0]  == sizeof(float));
    assert(src0->nb[0] == sizeof(float));

    // 遍历每一行，计算绝对值操作
    for (int i = 0; i < n; i++) {
        ggml_vec_abs_f32(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1]));
    }
}

// 计算绝对值操作的前向传播，根据输入张量的类型调用相应的函数
static void ggml_compute_forward_abs(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行分支处理
    switch (src0->type) {
        // 如果类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用相应的函数进行绝对值操作
                ggml_compute_forward_abs_f32(params, src0, dst);
            } break;
        // 如果类型不为 GGML_TYPE_F32
        default:
            {
                // 断言失败
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_sgn

// 计算符号函数操作的前向传播，针对浮点数类型的输入张量
static void ggml_compute_forward_sgn_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数 ith 为 0
    assert(params->ith == 0);
    // 断言输入张量 src0 和输出张量 dst 具有相同的形状
    assert(ggml_are_same_shape(src0, dst));
    // 如果任务类型为初始化或者结束，则直接返回，不执行后续操作
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取源数据的行数和列数
    const int n  = ggml_nrows(src0);
    const int nc = src0->ne[0];

    // 确保目标数据和源数据的第一个维度的字节大小为 float 类型的大小
    assert(dst->nb[0]  == sizeof(float));
    assert(src0->nb[0] == sizeof(float));

    // 遍历每一行数据，对每一行数据进行处理
    for (int i = 0; i < n; i++) {
        // 对每一行数据进行 float 类型的符号函数处理
        ggml_vec_sgn_f32(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1])));
    }
// 计算正向传播的符号函数
static void ggml_compute_forward_sgn(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的数据类型进行不同的处理
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                // 调用处理浮点数类型的正向传播符号函数
                ggml_compute_forward_sgn_f32(params, src0, dst);
            } break;
        default:
            {
                // 断言错误，不支持的数据类型
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_neg

// 处理浮点数类型的负向传播函数
static void ggml_compute_forward_neg_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数 ith 的值为 0
    assert(params->ith == 0);
    // 断言输入张量 src0 和输出张量 dst 的形状相同
    assert(ggml_are_same_shape(src0, dst));

    // 如果任务类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取输入张量的行数和通道数
    const int n  = ggml_nrows(src0);
    const int nc = src0->ne[0];

    // 断言输出张量和输入张量的字节大小为 float 类型
    assert(dst->nb[0]  == sizeof(float));
    assert(src0->nb[0] == sizeof(float));

    // 遍历每一行数据，对每个元素取负值
    for (int i = 0; i < n; i++) {
        ggml_vec_neg_f32(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1]));
    }
}

// 处理负向传播函数
static void ggml_compute_forward_neg(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的数据类型进行不同的处理
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                // 调用处理浮点数类型的负向传播函数
                ggml_compute_forward_neg_f32(params, src0, dst);
            } break;
        default:
            {
                // 断言错误，不支持的数据类型
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_step

// 处理浮点数类型的步进正向传播函数
static void ggml_compute_forward_step_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数 ith 的值为 0
    assert(params->ith == 0);
    // 断言输入张量 src0 和输出张量 dst 的形状相同

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    const int n  = ggml_nrows(src0);
    # 定义常量nc为src0的第一个维度大小
    const int nc = src0->ne[0];

    # 断言dst的第一个维度大小为float类型的大小
    assert(dst->nb[0]  == sizeof(float));
    # 断言src0的第一个维度大小为float类型的大小
    assert(src0->nb[0] == sizeof(float));

    # 遍历n次
    for (int i = 0; i < n; i++) {
        # 调用ggml_vec_step_f32函数，传入参数为nc，dst数据指针偏移i倍dst的第二个维度大小，src0数据指针偏移i倍src0的第二个维度大小
        ggml_vec_step_f32(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1])));
    }
// 计算前向传播步骤的函数，根据输入参数和输入张量计算结果并存储在目标张量中
static void ggml_compute_forward_step(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的处理
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用处理 GGML_TYPE_F32 类型的函数
                ggml_compute_forward_step_f32(params, src0, dst);
            } break;
        // 如果输入张量类型不在已知范围内
        default:
            {
                // 断言失败
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_tanh

// 处理 GGML_TYPE_F32 类型的 tanh 函数
static void ggml_compute_forward_tanh_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数 ith 为 0
    assert(params->ith == 0);
    // 断言输入张量 src0 和目标张量 dst 具有相同的形状
    assert(ggml_are_same_shape(src0, dst));

    // 如果参数类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取输入张量的行数和列数
    const int n  = ggml_nrows(src0);
    const int nc = src0->ne[0];

    // 断言目标张量和输入张量的数据类型为 float
    assert(dst->nb[0]  == sizeof(float));
    assert(src0->nb[0] == sizeof(float));

    // 遍历每一行，对每一行的数据进行 tanh 操作
    for (int i = 0; i < n; i++) {
        ggml_vec_tanh_f32(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1]));
    }
}

// 处理 tanh 函数的前向传播
static void ggml_compute_forward_tanh(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的处理
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用处理 GGML_TYPE_F32 类型的 tanh 函数
                ggml_compute_forward_tanh_f32(params, src0, dst);
            } break;
        // 如果输入张量类型不在已知范围内
        default:
            {
                // 断言失败
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_elu

// 处理 GGML_TYPE_F32 类型的 elu 函数
static void ggml_compute_forward_elu_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数 ith 为 0
    assert(params->ith == 0);
    // 断言输入张量 src0 和目标张量 dst 具有相同的形状
    assert(ggml_are_same_shape(src0, dst));

    // 如果参数类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取输入张量的行数
    const int n  = ggml_nrows(src0);
    # 定义常量 nc 为 src0 的第一个维度大小
    const int nc = src0->ne[0];

    # 断言目标数据的第一个维度大小为 float 类型的大小
    assert(dst->nb[0]  == sizeof(float));
    # 断言源数据的第一个维度大小为 float 类型的大小
    assert(src0->nb[0] == sizeof(float));

    # 遍历 n 次
    for (int i = 0; i < n; i++) {
        # 对 nc 个元素进行 ELU 激活函数操作，将结果存储到目标数据和源数据对应位置
        ggml_vec_elu_f32(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1]));
    }
// 结束 ggml_compute_forward_elu 函数定义

static void ggml_compute_forward_elu(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的操作
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用 ggml_compute_forward_elu_f32 函数处理
                ggml_compute_forward_elu_f32(params, src0, dst);
            } break;
        // 如果输入张量类型不是 GGML_TYPE_F32
        default:
            {
                // 断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_relu

// 处理 GGML_TYPE_F32 类型的张量的 ReLU 操作
static void ggml_compute_forward_relu_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数 ith 为 0
    assert(params->ith == 0);
    // 断言输入张量和输出张量形状相同
    assert(ggml_are_same_shape(src0, dst));

    // 如果任务类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取输入张量的行数
    const int n  = ggml_nrows(src0);
    // 获取输入张量的第一个维度大小
    const int nc = src0->ne[0];

    // 断言输出张量的第一个维度大小为 float 类型大小
    assert(dst->nb[0]  == sizeof(float));
    // 断言输入张量的第一个维度大小为 float 类型大小
    assert(src0->nb[0] == sizeof(float));

    // 遍历每一行，对每一行进行 ReLU 操作
    for (int i = 0; i < n; i++) {
        ggml_vec_relu_f32(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1]));
    }
}

// 处理 GGML_TYPE_F32 类型的张量的 ReLU 操作
static void ggml_compute_forward_relu(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的操作
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用 ggml_compute_forward_relu_f32 函数处理
                ggml_compute_forward_relu_f32(params, src0, dst);
            } break;
        // 如果输入张量类型不是 GGML_TYPE_F32
        default:
            {
                // 断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_gelu

// 处理 GGML_TYPE_F32 类型的张量的 GELU 操作
static void ggml_compute_forward_gelu_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言输入张量在除了第一个维度以外是连续的
    GGML_ASSERT(ggml_is_contiguous_except_dim_1(src0));
    // 断言输出张量在除了第一个维度以外是连续的
    GGML_ASSERT(ggml_is_contiguous_except_dim_1(dst));
    // 断言输入张量和输出张量形状相同
    GGML_ASSERT(ggml_are_same_shape(src0, dst));
    // 如果任务类型为初始化或者结束，则直接返回，不执行后续操作
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取当前线程的索引和总线程数
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取输入数据的列数和行数
    const int nc = src0->ne[0];
    const int nr = ggml_nrows(src0);

    // 计算每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 计算当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 循环处理当前线程负责的行范围内的数据
    for (int i1 = ir0; i1 < ir1; i1++) {
        // 对输入数据进行 GELU 激活函数的计算，并将结果存储到目标数据中
        ggml_vec_gelu_f32(nc,
                (float *) ((char *) dst->data  + i1*( dst->nb[1])),
                (float *) ((char *) src0->data + i1*(src0->nb[1])));
#ifndef NDEBUG
        // 如果处于调试模式下
        for (int k = 0; k < nc; k++) {
            // 获取目标数据中第 i1 行第 k 列的值
            const float x = ((float *) ((char *) dst->data + i1*( dst->nb[1])))[k];
            // 未使用 x，用于消除编译器警告
            UNUSED(x);
            // 断言 x 不是 NaN
            assert(!isnan(x));
            // 断言 x 不是无穷大
            assert(!isinf(x));
        }
#endif
    }
}

static void ggml_compute_forward_gelu(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的操作
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                // 调用针对 F32 类型的 gelu 前向计算函数
                ggml_compute_forward_gelu_f32(params, src0, dst);
            } break;
        default:
            {
                // 断言不会执行到这里
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_gelu_quick

static void ggml_compute_forward_gelu_quick_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言输入张量 src0 在除了第一维度以外是连续的
    GGML_ASSERT(ggml_is_contiguous_except_dim_1(src0));
    // 断言输出张量 dst 在除了第一维度以外是连续的
    GGML_ASSERT(ggml_is_contiguous_except_dim_1(dst));
    // 断言输入张量 src0 和输出张量 dst 的形状相同
    GGML_ASSERT(ggml_are_same_shape(src0, dst));

    // 如果任务类型是初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取当前线程的索引和线程总数
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取输入张量的第一维度大小和行数
    const int nc = src0->ne[0];
    const int nr = ggml_nrows(src0);

    // 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 遍历当前线程处理的行
    for (int i1 = ir0; i1 < ir1; i1++) {
        // 调用快速 gelu 函数处理当前行
        ggml_vec_gelu_quick_f32(nc,
                (float *) ((char *) dst->data  + i1*( dst->nb[1])),
                (float *) ((char *) src0->data + i1*(src0->nb[1])));

#ifndef NDEBUG
        // 如果处于调试模式下
        for (int k = 0; k < nc; k++) {
            // 获取目标数据中第 i1 行第 k 列的值
            const float x = ((float *) ((char *) dst->data + i1*( dst->nb[1])))[k];
            // 未使用 x，用于消除编译器警告
            UNUSED(x);
            // 断言 x 不是 NaN
            assert(!isnan(x));
            // 断言 x 不是无穷大
            assert(!isinf(x));
        }
#endif
    }
}
// 计算快速 GELU 激活函数的前向传播
static void ggml_compute_forward_gelu_quick(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的数据类型进行不同的处理
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                // 调用针对 F32 类型的快速 GELU 前向传播函数
                ggml_compute_forward_gelu_quick_f32(params, src0, dst);
            } break;
        default:
            {
                // 断言，如果不是 F32 类型则抛出异常
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_silu

// 计算单个 F32 类型张量的 SiLU 激活函数的前向传播
static void ggml_compute_forward_silu_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言，检查输入张量是否在除了第一个维度以外是连续的
    GGML_ASSERT(ggml_is_contiguous_except_dim_1(src0));
    // 断言，检查输出张量是否在除了第一个维度以外是连续的
    GGML_ASSERT(ggml_is_contiguous_except_dim_1(dst));
    // 断言，检查输入和输出张量的形状是否相同
    GGML_ASSERT(ggml_are_same_shape(src0, dst));

    // 如果任务类型是初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取线程索引和总线程数
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取输入张量的第一个维度大小和行数
    const int nc = src0->ne[0];
    const int nr = ggml_nrows(src0);

    // 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 遍历处理每一行
    for (int i1 = ir0; i1 < ir1; i1++) {
        // 调用 F32 类型的 SiLU 函数处理每一行
        ggml_vec_silu_f32(nc,
                (float *) ((char *) dst->data  + i1*( dst->nb[1])),
                (float *) ((char *) src0->data + i1*(src0->nb[1])));

#ifndef NDEBUG
        // 调试模式下，检查输出张量中的值是否为非法值
        for (int k = 0; k < nc; k++) {
            const float x = ((float *) ((char *) dst->data + i1*(dst->nb[1])))[k];
            UNUSED(x);
            assert(!isnan(x));
            assert(!isinf(x));
        }
#endif
    }
}

// 计算 SiLU 激活函数的前向传播
static void ggml_compute_forward_silu(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    # 根据 src0 的类型进行不同的操作
    switch (src0->type) {
        # 如果 src0 的类型是 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                # 调用 ggml_compute_forward_silu_f32 函数处理参数 params, src0, dst
                ggml_compute_forward_silu_f32(params, src0, dst);
            } break;
        # 如果 src0 的类型不是 GGML_TYPE_F32
        default:
            {
                # 断言，表示出现了不应该出现的情况
                GGML_ASSERT(false);
            } break;
    }
// ggml_compute_forward_leaky_relu

// 计算 leaky relu 激活函数的前向传播，对于 float32 类型的数据
static void ggml_compute_forward_leaky_relu_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 确保参数中的 ith 为 0
    assert(params->ith == 0);
    // 确保 src0 和 dst 张量具有相同的形状
    assert(ggml_are_same_shape(src0, dst));

    // 如果任务类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取 src0 张量的行数和通道数
    const int n  = ggml_nrows(src0);
    const int nc = src0->ne[0];

    // 从 dst 张量的操作参数中复制负斜率值
    float negative_slope;
    memcpy(&negative_slope, dst->op_params, sizeof(float));

    // 确保 dst 和 src0 张量的数据类型为 float32
    assert(dst->nb[0]  == sizeof(float));
    assert(src0->nb[0] == sizeof(float));

    // 遍历每一行数据，对每个元素应用 leaky relu 激活函数
    for (int i = 0; i < n; i++) {
        ggml_vec_leaky_relu_f32(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1])), negative_slope);
    }
}

// 根据输入张量的数据类型选择对应的前向传播函数
static void ggml_compute_forward_leaky_relu(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                ggml_compute_forward_leaky_relu_f32(params, src0, dst);
            } break;
        default:
            {
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_silu_back

// 计算 silu 激活函数的反向传播，对于 float32 类型的数据
static void ggml_compute_forward_silu_back_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * grad,
        struct ggml_tensor * dst) {
    // 确保 grad、src0 和 dst 张量在除第一个维度外是连续的
    GGML_ASSERT(ggml_is_contiguous_except_dim_1(grad));
    GGML_ASSERT(ggml_is_contiguous_except_dim_1(src0));
    GGML_ASSERT(ggml_is_contiguous_except_dim_1(dst));
    // 确保 src0、dst 和 grad 张量具有相同的形状
    GGML_ASSERT(ggml_are_same_shape(src0, dst));
    GGML_ASSERT(ggml_are_same_shape(src0, grad));

    // 如果任务类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取当前线程的索引和总线程数
    const int ith = params->ith;
    const int nth = params->nth;
    // 获取源数据的列数
    const int nc = src0->ne[0];
    // 获取源数据的行数
    const int nr = ggml_nrows(src0);

    // 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 循环遍历处理当前线程负责的行范围
    for (int i1 = ir0; i1 < ir1; i1++) {
        // 对每一行进行反向的 SILU 激活函数计算
        ggml_vec_silu_backward_f32(nc,
                (float *) ((char *) dst->data  + i1*( dst->nb[1])),
                (float *) ((char *) src0->data + i1*(src0->nb[1])),
                (float *) ((char *) grad->data + i1*(grad->nb[1])));
#ifndef NDEBUG
        // 如果处于调试模式
        for (int k = 0; k < nc; k++) {
            // 获取目标数据中第 i1 行第 k 列的值
            const float x = ((float *) ((char *) dst->data + i1*( dst->nb[1])))[k];
            // 未使用 x，用于消除编译器警告
            UNUSED(x);
            // 断言 x 不是 NaN
            assert(!isnan(x));
            // 断言 x 不是无穷大
            assert(!isinf(x));
        }
#endif
    }
}

static void ggml_compute_forward_silu_back(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * grad,
        struct ggml_tensor * dst) {
    // 根据源张量的类型进行不同的操作
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                // 调用针对 float 类型的 ggml_compute_forward_silu_back_f32 函数
                ggml_compute_forward_silu_back_f32(params, src0, grad, dst);
            } break;
        default:
            {
                // 断言不会执行到这里
                GGML_ASSERT(false);
            } break;
    }
}

static void ggml_compute_forward_hardswish_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言当前是第 0 个任务
    assert(params->ith == 0);
    // 断言源张量和目标张量形状相同
    assert(ggml_are_same_shape(src0, dst));

    // 如果是初始化或结束任务，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取行数和列数
    const int n  = ggml_nrows(src0);
    const int nc = src0->ne[0];

    // 断言目标张量和源张量的数据类型都是 float
    assert(dst->nb[0]  == sizeof(float));
    assert(src0->nb[0] == sizeof(float));

    // 遍历每一行，对每一行进行 hardswish 操作
    for (int i = 0; i < n; i++) {
        ggml_vec_hardswish_f32(nc,
                // 获取目标数据中第 i 行的起始地址
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                // 获取源数据中第 i 行的起始地址
                (float *) ((char *) src0->data + i*(src0->nb[1])));
    }
}

static void ggml_compute_forward_hardswish(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据源张量的类型进行不同的操作
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                // 调用针对 float 类型的 ggml_compute_forward_hardswish_f32 函数
                ggml_compute_forward_hardswish_f32(params, src0, dst);
            } break;
        default:
            {
                // 断言不会执行到这里
                GGML_ASSERT(false);
            } break;
    }
}
// 计算前向硬Sigmoid激活函数的浮点数版本
static void ggml_compute_forward_hardsigmoid_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数中的ith为0
    assert(params->ith == 0);
    // 断言src0和dst具有相同的形状
    assert(ggml_are_same_shape(src0, dst));

    // 如果任务类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取src0的行数n和第一个维度大小nc
    const int n  = ggml_nrows(src0);
    const int nc = src0->ne[0];

    // 断言dst和src0的第一个维度大小为float类型的大小
    assert(dst->nb[0]  == sizeof(float));
    assert(src0->nb[0] == sizeof(float));

    // 遍历n次，对每个元素应用硬Sigmoid激活函数
    for (int i = 0; i < n; i++) {
        ggml_vec_hardsigmoid_f32(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1]));
    }
}

// 计算前向硬Sigmoid激活函数
static void ggml_compute_forward_hardsigmoid(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据src0的类型选择对应的计算函数
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                ggml_compute_forward_hardsigmoid_f32(params, src0, dst);
            } break;
        default:
            {
                GGML_ASSERT(false);
            } break;
    }
}

// 计算前向归一化的浮点数版本
static void ggml_compute_forward_norm_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言src0和dst具有相同的形状
    GGML_ASSERT(ggml_are_same_shape(src0, dst));

    // 如果任务类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 断言src0的第一个维度大小为float类型的大小
    GGML_ASSERT(src0->nb[0] == sizeof(float));

    // 获取ith和nth参数
    const int ith = params->ith;
    const int nth = params->nth;

    // 本地变量定义
    GGML_TENSOR_UNARY_OP_LOCALS

    // 从dst的操作参数中复制eps值
    float eps;
    memcpy(&eps, dst->op_params, sizeof(float));

    // 断言eps大于0
    GGML_ASSERT(eps > 0.0f);

    // 待优化部分
    // TODO: optimize
}
    // 遍历三维数组的三个维度，i03表示最外层循环，i02表示中间循环，i01表示最内层循环
    for (int64_t i03 = 0; i03 < ne03; i03++) {
        for (int64_t i02 = 0; i02 < ne02; i02++) {
            for (int64_t i01 = ith; i01 < ne01; i01 += nth) {
                // 计算当前位置在源数据中的偏移量，获取指向当前位置的指针
                const float * x = (float *) ((char *) src0->data + i01*nb01 + i02*nb02 + i03*nb03);

                // 初始化求和变量sum为0
                ggml_float sum = 0.0;
                // 遍历第一维度，计算当前位置的和
                for (int64_t i00 = 0; i00 < ne00; i00++) {
                    sum += (ggml_float)x[i00];
                }

                // 计算均值
                float mean = sum/ne00;

                // 计算当前位置在目标数据中的偏移量，获取指向当前位置的指针
                float * y = (float *) ((char *) dst->data + i01*nb1 + i02*nb2 + i03*nb3);

                // 初始化求和变量sum2为0
                ggml_float sum2 = 0.0;
                // 遍历第一维度，计算当前位置的方差
                for (int64_t i00 = 0; i00 < ne00; i00++) {
                    // 计算当前位置的偏差值
                    float v = x[i00] - mean;
                    // 将偏差值存入目标数据中
                    y[i00] = v;
                    // 计算方差的累加值
                    sum2 += (ggml_float)(v*v);
                }

                // 计算方差
                float variance = sum2/ne00;
                // 计算缩放比例，用于标准化数据
                const float scale = 1.0f/sqrtf(variance + eps);

                // 对目标数据进行缩放操作
                ggml_vec_scale_f32(ne00, y, scale);
            }
        }
    }
// 计算前向归一化操作，根据输入参数和张量进行计算，将结果存储到目标张量中
static void ggml_compute_forward_norm(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的操作
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                // 调用针对 float 类型的前向归一化计算函数
                ggml_compute_forward_norm_f32(params, src0, dst);
            } break;
        default:
            {
                // 断言，如果不是 float 类型则抛出异常
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_group_rms_norm

// 针对 float 类型的前向 RMS 归一化计算函数
static void ggml_compute_forward_rms_norm_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言，确保输入张量和目标张量具有相同的形状
    GGML_ASSERT(ggml_are_same_shape(src0, dst));

    // 如果参数类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 断言，确保输入张量的第一个维度大小为 float 类型的大小
    GGML_ASSERT(src0->nb[0] == sizeof(float));

    // 获取参数中的 ith 和 nth 值
    const int ith = params->ith;
    const int nth = params->nth;

    // 定义局部变量
    GGML_TENSOR_UNARY_OP_LOCALS

    // 定义 eps 变量，并从目标张量的操作参数中复制出来
    float eps;
    memcpy(&eps, dst->op_params, sizeof(float));

    // 断言，确保 eps 大于 0
    GGML_ASSERT(eps > 0.0f);

    // 循环遍历张量的维度进行计算
    for (int64_t i03 = 0; i03 < ne03; i03++) {
        for (int64_t i02 = 0; i02 < ne02; i02++) {
            for (int64_t i01 = ith; i01 < ne01; i01 += nth) {
                // 获取当前位置的数据指针
                const float * x = (float *) ((char *) src0->data + i01*nb01 + i02*nb02 + i03*nb03);

                // 计算平方和
                ggml_float sum = 0.0;
                for (int64_t i00 = 0; i00 < ne00; i00++) {
                    sum += (ggml_float)(x[i00] * x[i00]);
                }

                // 计算均值
                const float mean = sum/ne00;

                // 获取目标位置的数据指针
                float * y = (float *) ((char *) dst->data + i01*nb1 + i02*nb2 + i03*nb3);

                // 将 x 复制到 y
                memcpy(y, x, ne00 * sizeof(float));

                // 计算缩放比例
                const float scale = 1.0f/sqrtf(mean + eps);

                // 对 y 进行缩放操作
                ggml_vec_scale_f32(ne00, y, scale);
            }
        }
    }
}
// 计算前向 RMS 归一化操作，根据输入参数和张量 src0 计算结果存储到张量 dst 中
static void ggml_compute_forward_rms_norm(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据 src0 的数据类型进行不同的处理
    switch (src0->type) {
        // 如果数据类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用 ggml_compute_forward_rms_norm_f32 函数进行计算
                ggml_compute_forward_rms_norm_f32(params, src0, dst);
            } break;
        // 如果数据类型不是 GGML_TYPE_F32
        default:
            {
                // 断言失败
                GGML_ASSERT(false);
            } break;
    }
}

// 计算前向 RMS 归一化反向操作，根据输入参数和张量 src0、src1 计算结果存储到张量 dst 中
static void ggml_compute_forward_rms_norm_back_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 断言 src0、dst、src1 的形状相同
    GGML_ASSERT(ggml_are_same_shape(src0, dst) && ggml_are_same_shape(src0, src1));

    // 如果参数类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 断言 src0 的第一个维度大小为 sizeof(float)
    GGML_ASSERT(src0->nb[0] == sizeof(float));

    // 获取 params 中的 ith 和 nth
    const int ith = params->ith;
    const int nth = params->nth;

    // 定义 GGML_TENSOR_BINARY_OP_LOCALS

    // 定义 eps 为 float 类型，从 dst->op_params 中复制 sizeof(float) 大小的数据到 eps
    float eps;
    memcpy(&eps, dst->op_params, sizeof(float));

    // TODO: optimize
}

// 计算前向 RMS 归一化反向操作，根据输入参数和张量 src0、src1 计算结果存储到张量 dst 中
static void ggml_compute_forward_rms_norm_back(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 根据 src0 的数据类型进行不同的处理
    switch (src0->type) {
        // 如果数据类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用 ggml_compute_forward_rms_norm_back_f32 函数进行计算
                ggml_compute_forward_rms_norm_back_f32(params, src0, src1, dst);
            } break;
        // 如果数据类型不是 GGML_TYPE_F32
        default:
            {
                // 断言失败
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_group_norm

// 计算前向分组归一化操作，根据输入参数和张量 src0 计算结果存储到张量 dst 中
static void ggml_compute_forward_group_norm_f32(
    const struct ggml_compute_params * params,
    const struct ggml_tensor * src0,
    struct ggml_tensor * dst) {
    // 断言 src0 和 dst 的形状相同
    GGML_ASSERT(ggml_are_same_shape(src0, dst));

    // 如果参数类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 断言 src0 的第一个维度大小为 sizeof(float)
    GGML_ASSERT(src0->nb[0] == sizeof(float));

    // 获取 params 中的 ith
    const int ith = params->ith;
}
    // 定义一个常量 nth，其值为 params 结构体中的 nth 成员
    const int nth = params->nth;

    // 定义 GGML_TENSOR_UNARY_OP_LOCALS 宏

    // 定义一个常量 eps，其值为 1e-6f，表示一个很小的数，用于计算数值稳定性
    const float eps = 1e-6f; // TODO: make this a parameter

    // TODO: optimize，待优化

    // 获取 src0 张量的第三维度大小，即通道数
    int n_channels = src0->ne[2];
    // 获取目标张量的操作参数中的第一个参数，表示分组数
    int n_groups = dst->op_params[0];
    // 计算每个分组中的通道数，向上取整
    int n_channels_per_group = (n_channels + n_groups - 1) / n_groups;
    // 循环遍历每个组，每次处理一个组的数据
    for (int i = ith; i < n_groups; i+=nth) {
        // 计算当前组的起始通道和结束通道
        int start = i * n_channels_per_group;
        int end = start + n_channels_per_group;
        // 如果结束通道超过总通道数，则将结束通道设置为总通道数
        if (end > n_channels) {
            end = n_channels;
        }
        // 计算步长
        int step = end - start;

        // 遍历每个 i03 维度
        for (int64_t i03 = 0; i03 < ne03; i03++) {
            // 初始化 sum 为 0
            ggml_float sum = 0.0;
            // 遍历每个 i02 维度
            for (int64_t i02 = start; i02 < end; i02++) {
                // 遍历每个 i01 维度
                for (int64_t i01 = 0; i01 < ne01; i01++) {
                    // 计算当前位置的指针 x
                    const float * x = (float *)((char *) src0->data + i01 * nb01 + i02 * nb02 + i03 * nb03);

                    // 遍历每个 i00 维度
                    for (int64_t i00 = 0; i00 < ne00; i00++) {
                        // 累加当前值到 sum
                        sum += (ggml_float)x[i00];
                    }
                }
            }
            // 计算均值
            float mean = sum / (ne00 * ne01 * step);
            // 初始化 sum2 为 0
            ggml_float sum2 = 0.0;

            // 遍历每个 i02 维度
            for (int64_t i02 = start; i02 < end; i02++) {
                // 遍历每个 i01 维度
                for (int64_t i01 = 0; i01 < ne01; i01++) {
                    // 计算当前位置的指针 x
                    const float * x = (float *)((char *) src0->data + i01 * nb01 + i02 * nb02 + i03 * nb03);
                    // 计算当前位置的指针 y
                    float * y = (float *)((char *) dst->data + i01 * nb1 + i02 * nb2 + i03 * nb3);

                    // 遍历每个 i00 维度
                    for (int64_t i00 = 0; i00 < ne00; i00++) {
                        // 计算 v 值
                        float v = x[i00] - mean;
                        // 将 v 值赋给 y
                        y[i00] = v;
                        // 累加 v*v 到 sum2
                        sum2 += (ggml_float)(v * v);
                    }
                }
            }
            // 计算方差
            float variance = sum2 / (ne00 * ne01 * step);
            // 计算缩放比例
            const float scale = 1.0f / sqrtf(variance + eps);

            // 遍历每个 i02 维度
            for (int64_t i02 = start; i02 < end; i02++) {
                // 遍历每个 i01 维度
                for (int64_t i01 = 0; i01 < ne01; i01++) {
                    // 计算当前位置的指针 y
                    float * y = (float *)((char *) dst->data + i01 * nb1 + i02 * nb2 + i03 * nb3);
                    // 对 y 进行缩放
                    ggml_vec_scale_f32(ne00, y, scale);
                }
            }
        }
    }
// 计算前向组归一化操作
static void ggml_compute_forward_group_norm(
    const struct ggml_compute_params * params,  // 输入参数结构体指针
    const struct ggml_tensor * src0,  // 输入张量指针
    struct ggml_tensor * dst) {  // 输出张量指针
    // 根据输入张量类型进行不同的操作
    switch (src0->type) {
        case GGML_TYPE_F32:  // 如果输入张量类型为 GGML_TYPE_F32
            {
                ggml_compute_forward_group_norm_f32(params, src0, dst);  // 调用相应的计算函数
            } break;
        default:  // 默认情况
            {
                GGML_ASSERT(false);  // 断言错误
            } break;
    }
}

// 判断是否使用 BLAS 进行矩阵乘法计算
#if defined(GGML_USE_ACCELERATE) || defined(GGML_USE_OPENBLAS)
// 辅助函数，确定是使用 BLAS 还是不使用 BLAS
// 对于大矩阵，BLAS 更快
static bool ggml_compute_forward_mul_mat_use_blas(struct ggml_tensor * dst) {
    const struct ggml_tensor * src0 = dst->src[0];  // 获取输入张量1
    const struct ggml_tensor * src1 = dst->src[1];  // 获取输入张量2

    const int64_t ne10 = src1->ne[0];  // 获取输入张量2的第一个维度大小

    const int64_t ne0 = dst->ne[0];  // 获取输出张量的第一个维度大小
    const int64_t ne1 = dst->ne[1];  // 获取输出张量的第二个维度大小

    // 注意：对于 GGML_OP_MUL_MAT_ID，我们不希望进入 BLAS 分支，因为它会对每个批次元素的所有专家进行去量化（转换为浮点数）
    //       处理会变得非常缓慢
    // TODO: 找到这些的最佳值
    if (dst->op != GGML_OP_MUL_MAT_ID &&
        ggml_is_contiguous(src0) &&
        ggml_is_contiguous(src1) &&
      //src0->type == GGML_TYPE_F32 &&
        src1->type == GGML_TYPE_F32 &&
        (ne0 >= 32 && ne1 >= 32 && ne10 >= 32)) {

        /*printf("BLAS: %d %d %d %d %d\n", ne0, ne1, ne10, ne00, ne01);*/
        return true;  // 返回 true，表示使用 BLAS
    }

    return false;  // 返回 false，表示不使用 BLAS
}
#endif

static void ggml_compute_forward_mul_mat(
        const struct ggml_compute_params * params,  // 输入参数结构体指针
        const struct ggml_tensor * src0,  // 输入张量1指针
        const struct ggml_tensor * src1,  // 输入张量2指针
              struct ggml_tensor * dst) {  // 输出张量指针
    int64_t t0 = ggml_perf_time_us();  // 记录当前时间
    UNUSED(t0);  // 不使用 t0

    GGML_TENSOR_BINARY_OP_LOCALS  // 定义张量二元操作的本地变量

    const int ith = params->ith;  // 获取参数中的 ith 值
    // 定义一个常量，表示参数中的第几个元素
    const int nth = params->nth;

    // 获取第一个输入参数的数据类型
    const enum ggml_type type = src0->type;

    // 检查第二个输入参数是否是连续的
    const bool src1_cont = ggml_is_contiguous(src1);

    // 获取与输入参数数据类型相关的向量点积函数
    ggml_vec_dot_t    const vec_dot               = type_traits[type].vec_dot;
    // 获取向量点积函数的数据类型
    enum ggml_type    const vec_dot_type          = type_traits[type].vec_dot_type;
    // 获取将浮点数转换为向量点积函数数据类型的函数
    ggml_from_float_t const from_float_to_vec_dot = type_traits[vec_dot_type].from_float;

    // 断言输入参数的维度是否相等
    GGML_ASSERT(ne0 == ne01);
    GGML_ASSERT(ne1 == ne11);
    GGML_ASSERT(ne2 == ne12);
    GGML_ASSERT(ne3 == ne13);

    // 断言不支持对输入参数进行置换
    GGML_ASSERT(nb00 == ggml_type_size(type));
    GGML_ASSERT(nb10 == ggml_type_size(src1->type));

    // 断言目标参数不能被转置或置换
    GGML_ASSERT(nb0 == sizeof(float));
    GGML_ASSERT(nb0 <= nb1);
    GGML_ASSERT(nb1 <= nb2);
    GGML_ASSERT(nb2 <= nb3);

    // 计算广播因子
    const int64_t r2 = ne12/ne02;
    const int64_t r3 = ne13/ne03;

    // nb01 >= nb00 - src0 is not transposed
    // 根据 src0 的行数计算
#if defined(GGML_USE_CLBLAST)
    // 如果定义了 GGML_USE_CLBLAST
    if (ggml_cl_can_mul_mat(src0, src1, dst)) {
        // 如果可以使用 CLBLAST 进行矩阵乘法操作
        if (params->ith == 0 && params->type == GGML_TASK_COMPUTE) {
            // 如果是第一个线程并且是计算任务
            ggml_cl_mul_mat(src0, src1, dst, params->wdata, params->wsize);
            // 调用 CLBLAST 进行矩阵乘法操作
        }
        return;
    }
#endif

#if defined(GGML_USE_ACCELERATE) || defined(GGML_USE_OPENBLAS)
    // 如果定义了 GGML_USE_ACCELERATE 或 GGML_USE_OPENBLAS
    }
#endif

    if (params->type == GGML_TASK_INIT) {
        // 如果是初始化任务
        if (ith != 0) {
            // 如果不是第一个线程
            return;
        }
        if (src1->type != vec_dot_type) {
            // 如果 src1 的类型不是 vec_dot_type
            char * wdata = params->wdata;
            // 获取参数中的 wdata
            const size_t row_size = ggml_row_size(vec_dot_type, ne10);
            // 计算行大小

            assert(params->wsize >= ne11*ne12*ne13*row_size);
            // 断言 wsize 大于等于 ne11*ne12*ne13*row_size
            GGML_ASSERT(src1->type == GGML_TYPE_F32);
            // 断言 src1 的类型是 GGML_TYPE_F32

            for (int64_t i13 = 0; i13 < ne13; ++i13) {
                for (int64_t i12 = 0; i12 < ne12; ++i12) {
                    for (int64_t i11 = 0; i11 < ne11; ++i11) {
                        // 遍历 ne11*ne12*ne13 维度
                        from_float_to_vec_dot((float *)((char *) src1->data + i13*nb13 + i12*nb12 + i11*nb11), (void *) wdata, ne10);
                        // 将浮点数转换为 vec_dot
                        wdata += row_size;
                    }
                }
            }
        }

        return;
    }

    if (params->type == GGML_TASK_FINALIZE) {
        // 如果是最终化任务
        return;
    }

    const void * wdata    = (src1->type == vec_dot_type) ? src1->data : params->wdata;
    // 根据 src1 的类型选择 wdata
    const size_t row_size = ggml_row_size(vec_dot_type, ne10);
    // 计算行大小

    const int64_t nr0 = ne01;          // src0 rows
    const int64_t nr1 = ne1*ne12*ne13; // src1 rows

    //printf("nr0 = %lld, nr1 = %lld\n", nr0, nr1);

    // distribute the thread work across the inner or outer loop based on which one is larger

    const int64_t nth0 = nr0 > nr1 ? nth : 1; // parallelize by src0 rows
    const int64_t nth1 = nr0 > nr1 ? 1 : nth; // parallelize by src1 rows

    const int64_t ith0 = ith % nth0;
    const int64_t ith1 = ith / nth0;

    const int64_t dr0 = (nr0 + nth0 - 1)/nth0;
    const int64_t dr1 = (nr1 + nth1 - 1)/nth1;

    const int64_t ir010 = dr0*ith0;
    // 计算 ir010
    // 计算第一个维度的起始和结束索引
    const int64_t ir011 = MIN(ir010 + dr0, nr0);

    // 计算第二个维度的起始和结束索引
    const int64_t ir110 = dr1*ith1;
    const int64_t ir111 = MIN(ir110 + dr1, nr1);

    // 如果没有工作要做的线程，则让出 CPU 时间片（不确定是否有帮助）
    if (ir010 >= ir011 || ir110 >= ir111) {
        sched_yield();
        return;
    }

    // 断言确保 ne12 能被 ne02 整除，ne13 能被 ne03 整除
    assert(ne12 % ne02 == 0);
    assert(ne13 % ne03 == 0);

    // 尝试使用块状分割
    const int64_t blck_0 = 16;
    const int64_t blck_1 = 16;

    // 尝试减少伪共享（似乎没有影响）
    float tmp[16];
// ggml_compute_forward_mul_mat_id 函数用于计算矩阵乘法
static void ggml_compute_forward_mul_mat_id(
        const struct ggml_compute_params * params, // 传入的计算参数
        const struct ggml_tensor * ids, // 输入张量 ids
        const struct ggml_tensor * src1, // 输入张量 src1
              struct ggml_tensor * dst) { // 输出张量 dst

    const struct ggml_tensor * src0 = dst->src[2]; // 获取 dst 的第三个源张量，仅用于 GGML_TENSOR_BINARY_OP_LOCALS

    GGML_TENSOR_BINARY_OP_LOCALS // 定义 GGML_TENSOR_BINARY_OP_LOCALS

    const int ith = params->ith; // 获取 params 中的 ith
    const int nth = params->nth; // 获取 params 中的 nth

    const enum ggml_type type = src0->type; // 获取 src0 的数据类型

    const bool src1_cont = ggml_is_contiguous(src1); // 检查 src1 是否是连续的

    ggml_vec_dot_t    const vec_dot               = type_traits[type].vec_dot; // 获取类型为 type 的 vec_dot 函数
    enum ggml_type    const vec_dot_type          = type_traits[type].vec_dot_type; // 获取类型为 type 的 vec_dot_type
    ggml_from_float_t const from_float_to_vec_dot = type_traits[vec_dot_type].from_float; // 获取从浮点数到 vec_dot_type 的转换函数

    GGML_ASSERT(ne0 == ne01); // 断言 ne0 等于 ne01
    GGML_ASSERT(ne1 == ne11); // 断言 ne1 等于 ne11
    GGML_ASSERT(ne2 == ne12); // 断言 ne2 等于 ne12
    GGML_ASSERT(ne3 == ne13); // 断言 ne3 等于 ne13

    // 不支持 src0 或 src1 的置换
    GGML_ASSERT(nb00 == ggml_type_size(type); // 断言 nb00 等于类型 type 的大小
    GGML_ASSERT(nb10 == ggml_type_size(src1->type); // 断言 nb10 等于 src1 类型的大小

    // dst 不能被转置或置换
    GGML_ASSERT(nb0 == sizeof(float)); // 断言 nb0 等于 float 的大小
    GGML_ASSERT(nb0 <= nb1); // 断言 nb0 小于等于 nb1
    GGML_ASSERT(nb1 <= nb2); // 断言 nb1 小于等于 nb2
    GGML_ASSERT(nb2 <= nb3); // 断言 nb2 小于等于 nb3

    // 广播因子
    const int64_t r2 = ne12/ne02; // 计算 r2
    const int64_t r3 = ne13/ne03; // 计算 r3

    // 行组
    const int id   = ggml_get_op_params_i32(dst, 0); // 获取 dst 中的第一个操作参数
    const int n_as = ggml_get_op_params_i32(dst, 1); // 获取 dst 中的第二个操作参数

    char * wdata_src1_end = (src1->type == vec_dot_type) ?
            (char *) params->wdata :
            (char *) params->wdata + GGML_PAD(ggml_row_size(vec_dot_type, ggml_nelements(src1)), sizeof(int64_t)); // 计算 wdata_src1_end

    int64_t * matrix_row_counts = (int64_t *) (wdata_src1_end); // 定义 matrix_row_counts，存储在 wdata_src1_end 后面的数据，大小为 n_as
    int64_t * matrix_rows       = matrix_row_counts + n_as;     // 定义 matrix_rows，存储在 matrix_row_counts 后面的数据，大小为 n_as*ne11
}
    // 定义宏，用于访问矩阵的行数据
    #define MMID_MATRIX_ROW(row_id, i1) matrix_rows[(row_id)*ne11 + (i1)]

   // 如果任务类型为初始化
   if (params->type == GGML_TASK_INIT) {
        // 如果不是第一个线程，直接返回
        if (ith != 0) {
            return;
        }
        // 获取参数中的 wdata
        char * wdata = params->wdata;
        // 如果 src1 的类型不是 vec_dot_type
        if (src1->type != vec_dot_type) {
            // 计算每行的大小
            const size_t row_size = ggml_row_size(vec_dot_type, ne10);

            // 确保 wdata 的大小足够存储数据
            assert(params->wsize >= ne11*ne12*ne13*row_size);
            // 确保 src1 的类型为 GGML_TYPE_F32
            assert(src1->type == GGML_TYPE_F32);

            // 遍历 ne13、ne12、ne11，将数据从 src1 转换为 vec_dot 存储到 wdata 中
            for (int64_t i13 = 0; i13 < ne13; ++i13) {
                for (int64_t i12 = 0; i12 < ne12; ++i12) {
                    for (int64_t i11 = 0; i11 < ne11; ++i11) {
                        from_float_to_vec_dot((float *)((char *) src1->data + i13*nb13 + i12*nb12 + i11*nb11), (void *) wdata, ne10);
                        wdata += row_size;
                    }
                }
            }
        }

        // 初始化 matrix_row_counts
        GGML_ASSERT(wdata == wdata_src1_end);
        memset(matrix_row_counts, 0, n_as*sizeof(int64_t));

        // 根据 src0 矩阵对行进行分组
        for (int64_t i01 = 0; i01 < ids->ne[1]; i01++) {
            const int32_t row_id = *(const int32_t *) ((const char *) ids->data + i01*ids->nb[1] + id*ids->nb[0]);

            GGML_ASSERT(row_id >= 0 && row_id < n_as);
            MMID_MATRIX_ROW(row_id, matrix_row_counts[row_id]) = i01;
            matrix_row_counts[row_id] += 1;
        }

        return;
    }

    // 如果任务类型为结束
    if (params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 依次计算每个矩阵乘法
    }

    // 取消定义宏
    #undef MMID_MATRIX_ROW
// 函数开始
}

// 函数 ggml_compute_forward_out_prod_f32，计算两个张量的乘积并存储到目标张量中
static void ggml_compute_forward_out_prod_f32(
        const struct ggml_compute_params * params,  // 参数结构体指针
        const struct ggml_tensor * src0,  // 第一个输入张量指针
        const struct ggml_tensor * src1,  // 第二个输入张量指针
              struct ggml_tensor * dst) {  // 输出张量指针
    // int64_t t0 = ggml_perf_time_us();
    // UNUSED(t0);

    GGML_TENSOR_BINARY_OP_LOCALS

    const int ith = params->ith;  // 获取参数中的 ith 值
    const int nth = params->nth;  // 获取参数中的 nth 值

    GGML_ASSERT(ne0  == ne00);  // 断言 ne0 等于 ne00
    GGML_ASSERT(ne1  == ne10);  // 断言 ne1 等于 ne10
    GGML_ASSERT(ne2  == ne02);  // 断言 ne2 等于 ne02
    GGML_ASSERT(ne02 == ne12);  // 断言 ne02 等于 ne12
    GGML_ASSERT(ne3  == ne13);  // 断言 ne3 等于 ne13
    GGML_ASSERT(ne03 == ne13);  // 断言 ne03 等于 ne13

    // we don't support permuted src0 or src1
    GGML_ASSERT(nb00 == sizeof(float));  // 断言 nb00 等于 float 类型的大小

    // dst cannot be transposed or permuted
    GGML_ASSERT(nb0 == sizeof(float));  // 断言 nb0 等于 float 类型的大小
    // GGML_ASSERT(nb0 <= nb1);
    // GGML_ASSERT(nb1 <= nb2);
    // GGML_ASSERT(nb2 <= nb3);

    // nb01 >= nb00 - src0 is not transposed
    //   compute by src0 rows

    // TODO: #if defined(GGML_USE_CUBLAS) ggml_cuda_out_prod
    // TODO: #if defined(GGML_USE_CLBLAST)

#if defined(GGML_USE_ACCELERATE) || defined(GGML_USE_OPENBLAS)
    bool use_blas = ggml_is_matrix(src0) &&
        ggml_is_matrix(src1) &&
        ggml_is_contiguous(src0) &&
        (ggml_is_contiguous(src1) || ggml_is_transposed(src1));
#endif

    // 根据参数类型执行不同操作
    if (params->type == GGML_TASK_INIT) {
#if defined(GGML_USE_ACCELERATE) || defined(GGML_USE_OPENBLAS) // gemm beta will zero dst
        if (use_blas) {
            return;  // 如果使用 BLAS，则直接返回
        }
#endif
        if (ith != 0) {
            return;  // 如果 ith 不为 0，则直接返回
        }
        ggml_vec_set_f32(ne0*ne1*ne2*ne3, dst->data, 0);  // 将目标张量数据初始化为 0
        return;
    }

    if (params->type == GGML_TASK_FINALIZE) {
        return;  // 如果参数类型为 GGML_TASK_FINALIZE，则直接返回
    }

#if defined(GGML_USE_ACCELERATE) || defined(GGML_USE_OPENBLAS)
    // 如果使用 BLAS 库
    if (use_blas) {
        // 如果当前线程不是第一个线程，则不执行任何工作，直接返回
        if (params->ith != 0) {
            return;
        }
        // 调用 ggml_compute_forward_out_prod 函数的参数说明（以主要维度和次要维度表示）
        // src0: (k,n)
        // src1: (k,m)
        // dst:  (m,n)
        //
        // 调用 sgemm 函数的参数说明（参见 https://github.com/Reference-LAPACK/lapack/blob/master/BLAS/SRC/sgemm.f）
        // 同样以（主要维度，次要维度）表示
        // a: (m,k): 因此 src1 被转置
        // b: (k,n): 因此 src0
        // c: (m,n)
        //
        // 然而，如果 ggml_is_transposed(src1) 为真，则
        // src1->data 已经包含一个转置版本，因此 sgemm 不应再次转置它。

        // 获取 src0 的维度信息
        int n = src0->ne[0];
        int k = src0->ne[1];
        // 获取 src1 的维度信息
        int m = src1->ne[0];

        int transposeA, lda;

        // 如果 src1 没有被转置
        if (!ggml_is_transposed(src1)) {
            transposeA = CblasTrans;
            lda = m;
        } else {
            transposeA = CblasNoTrans;
            lda = k;
        }

        // 将 src1、src0、dst 的数据转换为 float 指针
        float * a = (float *) ((char *) src1->data);
        float * b = (float *) ((char *) src0->data);
        float * c = (float *) ((char *) dst->data);

        // 调用 cblas_sgemm 函数进行矩阵乘法运算
        cblas_sgemm(CblasRowMajor, transposeA, CblasNoTrans, m, n, k, 1.0, a, lda, b, n, 0.0, c, n);

        return;
    }
#endif

    // 将目标数组初始化为0
    // 遍历最后三个维度
    // 并行化最后三个维度

    // 目标数组的总行数
    const int64_t nr = ne1*ne2*ne3;

    // 每个线程处理的行数
    const int64_t dr = (nr + nth - 1)/nth;

    // 当前线程处理的行范围
    const int64_t ir0 = dr*ith;
    const int64_t ir1 = MIN(ir0 + dr, nr);

    // 尝试块状分割
    const int64_t blck_0 = MAX(GGML_VEC_MAD_UNROLL, 32);
    const int64_t blck_1 = 16;

    for (int64_t bir = ir0; bir < ir1; bir += blck_1) {
        const int64_t bir1 = MIN(bir + blck_1, ir1);
        for (int64_t bi01 = 0; bi01 < ne01; bi01 += blck_0) {
            const int64_t bne01 = MIN(bi01 + blck_0, ne01);
            for (int64_t ir = bir; ir < bir1; ++ir) {
                // 目标数组的索引
                const int64_t i3 = ir/(ne2*ne1);
                const int64_t i2 = (ir - i3*ne2*ne1)/ne1;
                const int64_t i1 = (ir - i3*ne2*ne1 - i2*ne1);

                const int64_t i02 = i2;
                const int64_t i03 = i3;

                //const int64_t i10 = i1;
                const int64_t i12 = i2;
                const int64_t i13 = i3;
#if GGML_VEC_MAD_UNROLL > 2
                // 计算循环次数的上限，使得循环次数是 GGML_VEC_MAD_UNROLL 的整数倍
                const int64_t bne01_unroll = bne01 - (bne01 % GGML_VEC_MAD_UNROLL);
                // 循环遍历，每次增加 GGML_VEC_MAD_UNROLL
                for (int64_t i01 = bi01; i01 < bne01_unroll; i01 += GGML_VEC_MAD_UNROLL) {
                    const int64_t i11 = i01;

                    // 计算源数据和目标数据的指针位置
                    float * s0 = (float *) ((char *) src0->data + (          i01*nb01 + i02*nb02 + i03*nb03));
                    float * s1 = (float *) ((char *) src1->data + (i1*nb10 + i11*nb11 + i12*nb12 + i13*nb13));
                    float * d  = (float *) ((char *)  dst->data + (          i1*nb1 + i2*nb2 + i3*nb3));

                    // 调用函数进行向量乘加操作
                    ggml_vec_mad_f32_unroll(ne0, nb01, nb11, d, s0, s1);
                }
                // 处理剩余的循环次数
                for (int64_t i01 = bne01_unroll; i01 < bne01; ++i01) {
                    const int64_t i11 = i01;

                    // 计算源数据和目标数据的指针位置
                    float * s0 = (float *) ((char *) src0->data + (          i01*nb01 + i02*nb02 + i03*nb03));
                    float * s1 = (float *) ((char *) src1->data + (i1*nb10 + i11*nb11 + i12*nb12 + i13*nb13));
                    float * d  = (float *) ((char *)  dst->data + (          i1*nb1 + i2*nb2 + i3*nb3));

                    // 调用函数进行向量乘加操作
                    ggml_vec_mad_f32(ne0, d, s0, *s1);
                }
#else
                // 处理 GGML_VEC_MAD_UNROLL 不大于 2 的情况
                for (int64_t i01 = bi01; i01 < bne01; ++i01) {
                    const int64_t i11 = i01;

                    // 计算源数据和目标数据的指针位置
                    float * s0 = (float *) ((char *) src0->data + (          i01*nb01 + i02*nb02 + i03*nb03));
                    float * s1 = (float *) ((char *) src1->data + (i1*nb10 + i11*nb11 + i12*nb12 + i13*nb13));
                    float * d  = (float *) ((char *)  dst->data + (          i1*nb1 + i2*nb2 + i3*nb3));

                    // 调用函数进行向量乘加操作
                    ggml_vec_mad_f32(ne0, d, s0, *s1);
                }
#endif
            }
        }
    }

    //int64_t t1 = ggml_perf_time_us();
    //static int64_t acc = 0;
    //acc += t1 - t0;
    //if (t1 - t0 > 10) {
    //    printf("\n");
    //    printf("ne00 = %5d, ne01 = %5d, ne02 = %5d, ne03 = %5d\n", ne00, ne01, ne02, ne03);
    // 打印 nb00, nb01, nb02, nb03 的值
    // printf("nb00 = %5d, nb01 = %5d, nb02 = %5d, nb03 = %5d\n", nb00, nb01, nb02, nb03);
    // 打印 ne10, ne11, ne12, ne13 的值
    // printf("ne10 = %5d, ne11 = %5d, ne12 = %5d, ne13 = %5d\n", ne10, ne11, ne12, ne13);
    // 打印 nb10, nb11, nb12, nb13 的值
    // printf("nb10 = %5d, nb11 = %5d, nb12 = %5d, nb13 = %5d\n", nb10, nb11, nb12, nb13);

    // 打印任务的执行时间和准确性
    // printf("XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX task %d/%d: %d us, acc = %d\n", ith, nth, (int) (t1 - t0), (int) acc);
    //}
// 定义一个静态函数，计算两个张量的乘积并存储到目标张量中
static void ggml_compute_forward_out_prod_q_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
              struct ggml_tensor * dst) {
    // 定义一些局部变量
    GGML_TENSOR_BINARY_OP_LOCALS;

    // 获取参数中的ith和nth值
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取输入张量src0的数据类型
    const enum ggml_type type = src0->type;
    // 获取将src0的数据类型转换为float类型的函数
    ggml_to_float_t const dequantize_row_q = type_traits[type].to_float;

    // 断言一些维度的相等性
    GGML_ASSERT(ne02 == ne12);
    GGML_ASSERT(ne03 == ne13);
    GGML_ASSERT(ne2  == ne12);
    GGML_ASSERT(ne3  == ne13);

    // 断言src0的dim0不能被置换
    GGML_ASSERT(nb00 == ggml_type_size(type));

    // 断言目标张量dst的dim0不能被置换
    GGML_ASSERT(nb0 == sizeof(float));

    // 断言一些维度的大小关系
    // GGML_ASSERT(nb0 <= nb1);
    // GGML_ASSERT(nb1 <= nb2);
    // GGML_ASSERT(nb2 <= nb3);

    GGML_ASSERT(ne0 == ne00);
    GGML_ASSERT(ne1 == ne10);
    GGML_ASSERT(ne2 == ne02);
    GGML_ASSERT(ne3 == ne03);

    // nb01 >= nb00 - src0未被置换
    //   通过src0的行计算

    // TODO: #if defined(GGML_USE_CUBLAS) ggml_cuda_out_prod
    // TODO: #if defined(GGML_USE_ACCELERATE) || defined(GGML_USE_OPENBLAS) || defined(GGML_USE_CLBLAST)

    // 如果任务类型为GGML_TASK_INIT
    if (params->type == GGML_TASK_INIT) {
        // 如果ith不为0，则返回
        if (ith != 0) {
            return;
        }
        // 将目标张量dst的数据全部置为0
        ggml_vec_set_f32(ne0*ne1*ne2*ne3, dst->data, 0);
        return;
    }

    // 如果任务类型为GGML_TASK_FINALIZE
    if (params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 按照最后三个维度并行化

    // 目标张量dst中的总行数
    const int64_t nr = ne1*ne2*ne3;

    // 每个线程处理的行数
    const int64_t dr = (nr + nth - 1)/nth;

    // 该线程处理的行范围
    const int64_t ir0 = dr*ith;
    const int64_t ir1 = MIN(ir0 + dr, nr);

    // 将dst的所有元素置为0
    // 对于i2,i3:
    //   对于i1:
    //     对于i01:
    //       对于i0:
    //         dst[i0,i1,i2,i3] += src0[i0,i01,i2,i3] * src1[i1,i01,i2,i3]
    // 根据参数 params 和 ith 计算出当前线程需要处理的数据起始位置的指针
    float * wdata = (float *) params->wdata + (ne0 + CACHE_LINE_SIZE_F32) * ith;

    // 遍历 dst 矩阵的行索引范围 [ir0, ir1)
    for (int64_t ir = ir0; ir < ir1; ++ir) {
        // 计算 dst 矩阵的三维索引 i3, i2, i1
        const int64_t i3 = ir/(ne2*ne1);
        const int64_t i2 = (ir - i3*ne2*ne1)/ne1;
        const int64_t i1 = (ir - i3*ne2*ne1 - i2*ne1);

        const int64_t i02 = i2;
        const int64_t i03 = i3;

        //const int64_t i10 = i1;
        const int64_t i12 = i2;
        const int64_t i13 = i3;

        // 遍历 ne01 维度，计算 src0, src1, dst 的指针位置
        for (int64_t i01 = 0; i01 < ne01; ++i01) {
            const int64_t i11 = i01;

            // 计算 src0, src1, dst 的指针位置
            float * s0 = (float *) ((char *) src0->data + (          i01*nb01 + i02*nb02 + i03*nb03));
            float * s1 = (float *) ((char *) src1->data + (i1*nb10 + i11*nb11 + i12*nb12 + i13*nb13));
            float * d  = (float *) ((char *)  dst->data + (          i1*nb1 + i2*nb2 + i3*nb3));

            // 对 s0 进行反量化操作
            dequantize_row_q(s0, wdata, ne0);
            // 计算矩阵乘法并加到 d 上
            ggml_vec_mad_f32(ne0, d, wdata, *s1);
        }
    }

    // 下面是一段注释掉的代码，用于性能测试和输出调试信息
// 计算两个张量的乘积，并将结果存储在目标张量中
static void ggml_compute_forward_out_prod(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 根据第一个源张量的类型进行不同的处理
    switch (src0->type) {
        // 对于不同类型的量化张量，调用相应的函数计算乘积
        case GGML_TYPE_Q4_0:
        case GGML_TYPE_Q4_1:
        case GGML_TYPE_Q5_0:
        case GGML_TYPE_Q5_1:
        case GGML_TYPE_Q8_0:
        case GGML_TYPE_Q2_K:
        case GGML_TYPE_Q3_K:
        case GGML_TYPE_Q4_K:
        case GGML_TYPE_Q5_K:
        case GGML_TYPE_Q6_K:
        case GGML_TYPE_IQ2_XXS:
        case GGML_TYPE_IQ2_XS:
        case GGML_TYPE_IQ3_XXS:
            {
                // 调用相应的函数计算乘积
                ggml_compute_forward_out_prod_q_f32(params, src0, src1, dst);
            } break;
        // 对于 F16 类型的张量，暂未实现，输出断言
        case GGML_TYPE_F16:
            {
                GGML_ASSERT(false); // todo
                // ggml_compute_forward_out_prod_f16_f32(params, src0, src1, dst);
            } break;
        // 对于 F32 类型的张量，调用相应的函数计算乘积
        case GGML_TYPE_F32:
            {
                ggml_compute_forward_out_prod_f32(params, src0, src1, dst);
            } break;
        // 默认情况下输出断言
        default:
            {
                GGML_ASSERT(false);
            } break;
    }
}

// 计算张量的缩放

static void ggml_compute_forward_scale_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 检查源张量和目标张量是否是连续的
    GGML_ASSERT(ggml_is_contiguous(src0));
    GGML_ASSERT(ggml_is_contiguous(dst));
    GGML_ASSERT(ggml_are_same_shape(src0, dst));

    // 如果任务类型是初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 缩放因子
    float v;
    memcpy(&v, dst->op_params, sizeof(float));

    const int ith = params->ith;
    const int nth = params->nth;

    const int nc = src0->ne[0];
    const int nr = ggml_nrows(src0);

    // 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 本线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);
}
    // 获取 src0 的第二维大小
    const size_t nb01 = src0->nb[1];

    // 获取 dst 的第二维大小
    const size_t nb1 = dst->nb[1];

    // 遍历 ir0 到 ir1 之间的索引
    for (int i1 = ir0; i1 < ir1; i1++) {
        // 如果 dst 和 src0 的数据指针不相同
        if (dst->data != src0->data) {
            // src0 和 dst 具有相同的形状 => 相同的索引
            // 从 src0 的数据指针复制数据到 dst 的数据指针
            memcpy((char *)dst->data + i1*nb1, (char *)src0->data + i1*nb01, nc * sizeof(float));
        }
        // 对 dst 的数据指针进行缩放操作
        ggml_vec_scale_f32(nc, (float *) ((char *) dst->data + i1*nb1), v);
    }
// 计算前向缩放操作，根据输入参数和张量进行操作，将结果存储到目标张量中
static void ggml_compute_forward_scale(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的操作
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用针对 F32 类型的前向缩放操作函数
                ggml_compute_forward_scale_f32(params, src0, dst);
            } break;
        // 如果输入张量类型不在已知范围内
        default:
            {
                // 断言失败，抛出异常
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_set

// 针对 F32 类型的前向设置操作
static void ggml_compute_forward_set_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 断言输入张量 src0 和目标张量 dst 具有相同的形状
    GGML_ASSERT(ggml_are_same_shape(src0, dst));
    // 断言目标张量 dst 和输入张量 src0 是连续的
    GGML_ASSERT(ggml_is_contiguous(dst) && ggml_is_contiguous(src0));

    // 在设置期间使用这些步长和数据偏移量来查看 src0 和 dst
    // nb0 隐式地是 element_size，因为 src0 和 dst 是连续的
    size_t nb1     = ((int32_t *) dst->op_params)[0];
    size_t nb2     = ((int32_t *) dst->op_params)[1];
    size_t nb3     = ((int32_t *) dst->op_params)[2];
    size_t offset  = ((int32_t *) dst->op_params)[3];
    bool   inplace = (bool) ((int32_t *) dst->op_params)[4];

    // 如果不是原地操作且任务类型为 GGML_TASK_INIT
    if (!inplace && (params->type == GGML_TASK_INIT)) {
        // 如果不是第一个任务，则直接返回
        if (params->ith != 0) {
            return;
        }
        // 在 INIT 阶段需要同步执行 memcpy，避免竞争条件
        memcpy(
            ((char *)  dst->data),
            ((char *) src0->data),
            ggml_nbytes(dst));
    }

    // 如果任务类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取当前任务的索引和总任务数
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取 src1 的行数和列数
    const int nr = ggml_nrows(src1);
    const int nc = src1->ne[0];

    // 定义局部变量 ne1 和 nb1，用于获取 src1 的元素数量和步长
    GGML_TENSOR_LOCALS(int64_t, ne1, src1, ne)
    GGML_TENSOR_LOCALS(size_t,  nb1, src1, nb)

    // 在设置期间查看 src0 和 dst
    const size_t nb0 = ggml_element_size(src0);
}
    // 计算上一个索引值，如果 ne10 为 0，则为 0，否则为 ne10-1
    const int im0 = (ne10 == 0 ? 0 : ne10-1);
    // 计算上一个索引值，如果 ne11 为 0，则为 0，否则为 ne11-1
    const int im1 = (ne11 == 0 ? 0 : ne11-1);
    // 计算上一个索引值，如果 ne12 为 0，则为 0，否则为 ne12-1
    const int im2 = (ne12 == 0 ? 0 : ne12-1);
    // 计算上一个索引值，如果 ne13 为 0，则为 0，否则为 ne13-1
    const int im3 = (ne13 == 0 ? 0 : ne13-1);

    // 断言，确保偏移量加上四个索引值乘以对应的步长不超过目标数据的字节数
    GGML_ASSERT(offset + im0*nb0  + im1*nb1  + im2*nb2  + im3*nb3  <= ggml_nbytes(dst));

    // 断言，确保 nb10 的大小为 float 类型的大小
    GGML_ASSERT(nb10 == sizeof(float));

    // 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 该线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    for (int ir = ir0; ir < ir1; ++ir) {
        // src0 和 dst 被视为具有 src1 和偏移量的形状
        // => 相同的索引
        const int i3 = ir/(ne12*ne11);
        const int i2 = (ir - i3*ne12*ne11)/ne11;
        const int i1 = (ir - i3*ne12*ne11 - i2*ne11);

        // 复制 src1 中的数据到 dst 中，数据类型为 float
        ggml_vec_cpy_f32(nc,
                (float *) ((char *)  dst->data + i3*nb3  + i2*nb2  + i1*nb1  + offset),
                (float *) ((char *) src1->data + i3*nb13 + i2*nb12 + i1*nb11));
    }
// 计算前向传播的设置，根据输入张量的类型调用相应的函数
static void ggml_compute_forward_set(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {

    // 根据输入张量 src0 的类型进行不同的处理
    switch (src0->type) {
        // 如果类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用相应的处理函数
                ggml_compute_forward_set_f32(params, src0, src1, dst);
            } break;
        // 如果类型为其他类型
        case GGML_TYPE_F16:
        case GGML_TYPE_Q4_0:
        case GGML_TYPE_Q4_1:
        case GGML_TYPE_Q5_0:
        case GGML_TYPE_Q5_1:
        case GGML_TYPE_Q8_0:
        case GGML_TYPE_Q8_1:
        case GGML_TYPE_Q2_K:
        case GGML_TYPE_Q3_K:
        case GGML_TYPE_Q4_K:
        case GGML_TYPE_Q5_K:
        case GGML_TYPE_Q6_K:
        case GGML_TYPE_IQ2_XXS:
        case GGML_TYPE_IQ2_XS:
        case GGML_TYPE_IQ3_XXS:
        default:
            {
                // 断言，抛出错误
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_cpy

// 复制前向传播的计算结果
static void ggml_compute_forward_cpy(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 调用复制函数
    ggml_compute_forward_dup(params, src0, dst);
}

// ggml_compute_forward_cont

// 继续前向传播的计算
static void ggml_compute_forward_cont(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 调用复制函数
    ggml_compute_forward_dup(params, src0, dst);
}

// ggml_compute_forward_reshape

// 重塑前向传播的计算结果
static void ggml_compute_forward_reshape(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 空操作
    UNUSED(params);
    UNUSED(src0);
    UNUSED(dst);
}

// ggml_compute_forward_view

// 查看前向传播的计算结果
static void ggml_compute_forward_view(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0) {
    // 空操作
    UNUSED(params);
    UNUSED(src0);
}

// ggml_compute_forward_permute
// 计算前向传播的置换操作，但是没有实际操作
static void ggml_compute_forward_permute(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0) {
    // 空操作，不执行任何操作
    UNUSED(params);
    UNUSED(src0);
}

// ggml_compute_forward_transpose

// 计算前向传播的转置操作，但是没有实际操作
static void ggml_compute_forward_transpose(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0) {
    // 空操作，不执行任何操作
    UNUSED(params);
    UNUSED(src0);
}

// ggml_compute_forward_get_rows

// 计算前向传播的获取行操作
static void ggml_compute_forward_get_rows_q(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
              struct ggml_tensor * dst) {
    // 断言确保参数中的ith为0
    assert(params->ith == 0);

    // 如果任务类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    GGML_TENSOR_BINARY_OP_LOCALS

    // 获取src1的列数和src0的行数
    const int64_t nc = ne00;
    const int64_t nr = ggml_nelements(src1); GGML_UNUSED(nr);

    const enum ggml_type type = src0->type;
    ggml_to_float_t const dequantize_row_q = type_traits[type].to_float;

    // 断言确保各个维度的大小匹配
    assert(ne0  == nc);
    assert(ne02 == ne11);
    assert(nb00 == ggml_type_size(type));
    assert(ggml_nrows(dst) == nr);

    // TODO: 多线程处理
    // 遍历src1的维度，根据src1的值获取src0对应位置的值，进行解量化操作，将结果存入dst中
    for (int64_t i12 = 0; i12 < ne12; ++i12) {
        for (int64_t i11 = 0; i11 < ne11; ++i11) {
            for (int64_t i10 = 0; i10 < ne10; ++i10) {
                const int64_t i01 = *(int32_t *) ((char *) src1->data + i10*nb10 + i11*nb11 + i12*nb12);

                dequantize_row_q(
                        (const void *) ((char *) src0->data + i01*nb01 + i11*nb02 + i12*nb03),
                             (float *) ((char *)  dst->data + i10*nb1  + i11*nb2  + i12*nb3), nc);
            }
        }
    }
}

// 计算前向传播的获取行操作，针对f16类型的数据
static void ggml_compute_forward_get_rows_f16(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
              struct ggml_tensor * dst) {
    // 断言确保参数中的ith为0
    assert(params->ith == 0);
    // 如果任务类型是初始化或者结束，则直接返回，不执行下面的操作
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 定义本地变量
    GGML_TENSOR_BINARY_OP_LOCALS

    // 获取矩阵的列数
    const int64_t nc = ne00;
    // 获取矩阵的行数，并标记为未使用
    const int64_t nr = ggml_nelements(src1); GGML_UNUSED(nr);

    // 断言确保矩阵维度匹配
    assert(ne0  == nc);
    assert(ne02 == ne11);
    assert(nb00 == sizeof(ggml_fp16_t));
    assert(ggml_nrows(dst) == nr);

    // 循环遍历矩阵，进行计算
    // TODO: 多线程处理
    for (int64_t i12 = 0; i12 < ne12; ++i12) {
        for (int64_t i11 = 0; i11 < ne11; ++i11) {
            for (int64_t i10 = 0; i10 < ne10; ++i10) {
                // 计算索引值
                const int64_t i01 = *(int32_t *) ((char *) src1->data + i10*nb10 + i11*nb11 + i12*nb12);

                // 将 fp16 转换为 fp32，并存储到目标矩阵中
                ggml_fp16_to_fp32_row(
                        (const void *) ((char *) src0->data + i01*nb01 + i11*nb02 + i12*nb03),
                             (float *) ((char *)  dst->data + i10*nb1  + i11*nb2  + i12*nb3), nc);
            }
        }
    }
// 计算前向传播，获取行数据，使用单精度浮点数
static void ggml_compute_forward_get_rows_f32(
        // 参数结构体指针，包含计算参数
        const struct ggml_compute_params * params,
        // 源张量指针，用于获取数据
        const struct ggml_tensor * src0,
        // 源张量指针，用于获取数据
        const struct ggml_tensor * src1,
        // 目标张量指针，用于存储结果数据
              struct ggml_tensor * dst) {
    // 断言，确保参数中的 ith 值为 0
    assert(params->ith == 0);

    // 如果任务类型为初始化或结束，直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 定义本地变量
    GGML_TENSOR_BINARY_OP_LOCALS

    // 计算列数和行数
    const int64_t nc = ne00;
    const int64_t nr = ggml_nelements(src1); GGML_UNUSED(nr);

    // 断言，确保 ne0 等于列数，ne02 等于 ne11，nb00 等于单精度浮点数大小，目标张量行数等于行数
    assert(ne0  == nc);
    assert(ne02 == ne11);
    assert(nb00 == sizeof(float));
    assert(ggml_nrows(dst) == nr);

    // 循环计算每个元素
    // TODO: 多线程
    for (int64_t i12 = 0; i12 < ne12; ++i12) {
        for (int64_t i11 = 0; i11 < ne11; ++i11) {
            for (int64_t i10 = 0; i10 < ne10; ++i10) {
                // 计算索引值
                const int64_t i01 = *(int32_t *) ((char *) src1->data + i10*nb10 + i11*nb11 + i12*nb12);

                // 复制单精度浮点数数据
                ggml_vec_cpy_f32(nc,
                        (float *) ((char *)  dst->data + i10*nb1  + i11*nb2  + i12*nb3),
                        (float *) ((char *) src0->data + i01*nb01 + i11*nb02 + i12*nb03));
            }
        }
    }
}

// 计算前向传播，获取行数据
static void ggml_compute_forward_get_rows(
        // 参数结构体指针，包含计算参数
        const struct ggml_compute_params * params,
        // 源张量指针，用于获取数据
        const struct ggml_tensor * src0,
        // 源张量指针，用于获取数据
        const struct ggml_tensor * src1,
        // 目标张量指针，用于存储结果数据
        struct ggml_tensor * dst) {
    // 根据 src0 的类型进行不同的操作
    switch (src0->type) {
        // 如果 src0 的类型是以下之一，则调用 ggml_compute_forward_get_rows_q 函数
        case GGML_TYPE_Q4_0:
        case GGML_TYPE_Q4_1:
        case GGML_TYPE_Q5_0:
        case GGML_TYPE_Q5_1:
        case GGML_TYPE_Q8_0:
        case GGML_TYPE_Q8_1:
        case GGML_TYPE_Q2_K:
        case GGML_TYPE_Q3_K:
        case GGML_TYPE_Q4_K:
        case GGML_TYPE_Q5_K:
        case GGML_TYPE_Q6_K:
        case GGML_TYPE_IQ2_XXS:
        case GGML_TYPE_IQ2_XS:
        case GGML_TYPE_IQ3_XXS:
            {
                ggml_compute_forward_get_rows_q(params, src0, src1, dst);
            } break;
        // 如果 src0 的类型是 GGML_TYPE_F16，则调用 ggml_compute_forward_get_rows_f16 函数
        case GGML_TYPE_F16:
            {
                ggml_compute_forward_get_rows_f16(params, src0, src1, dst);
            } break;
        // 如果 src0 的类型是 GGML_TYPE_F32 或 GGML_TYPE_I32，则调用 ggml_compute_forward_get_rows_f32 函数
        case GGML_TYPE_F32:
        case GGML_TYPE_I32:
            {
                ggml_compute_forward_get_rows_f32(params, src0, src1, dst);
            } break;
        // 如果 src0 的类型不在以上列举的类型中，则断言失败
        default:
            {
                GGML_ASSERT(false);
            } break;
    }

    // 下面是一段注释掉的代码，用于打印输出 dst 结构体中的数据
    //static bool first = true;
    //printf("ne0 = %d, ne1 = %d, ne2 = %d\n", dst->ne[0], dst->ne[1], dst->ne[2]);
    //if (first) {
    //    first = false;
    //} else {
    //    for (int k = 0; k < dst->ne[1]; ++k) {
    //        for (int j = 0; j < dst->ne[0]/16; ++j) {
    //            for (int i = 0; i < 16; ++i) {
    //                printf("%8.4f ", ((float *) dst->data)[k*dst->ne[0] + j*16 + i]);
    //            }
    //            printf("\n");
    //        }
    //        printf("\n");
    //    }
    //    printf("\n");
    //    exit(0);
    //}
// ggml_compute_forward_get_rows_back 函数的实现，用于计算前向传播并更新目标张量数据
static void ggml_compute_forward_get_rows_back_f32_f16(
        const struct ggml_compute_params * params,  // 参数结构体指针，包含计算参数信息
        const struct ggml_tensor * src0,  // 源张量1指针，用于计算
        const struct ggml_tensor * src1,  // 源张量2指针，用于计算
              struct ggml_tensor * dst) {  // 目标张量指针，用于存储计算结果
    GGML_ASSERT(params->ith == 0);  // 断言，确保参数中的 ith 值为 0
    GGML_ASSERT(ggml_is_contiguous(dst));  // 断言，确保目标张量是连续的

    // ggml_compute_forward_dup_same_cont(params, opt0, dst);  // 注释掉的函数调用

    if (params->type == GGML_TASK_INIT) {  // 如果任务类型为初始化
        if (params->ith != 0) {  // 如果 ith 不为 0
            return;  // 返回
        }
        memset(dst->data, 0, ggml_nbytes(dst));  // 将目标张量数据清零
    }

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  // 如果任务类型为初始化或结束
        return;  // 返回
    }

    const int nc = src0->ne[0];  // 获取源张量1的第一个维度大小
    const int nr = ggml_nelements(src1);  // 获取源张量2的元素个数

    GGML_ASSERT( dst->ne[0] == nc);  // 断言，确保目标张量的第一个维度大小与源张量1相同
    GGML_ASSERT(src0->nb[0] == sizeof(ggml_fp16_t));  // 断言，确保源张量1的数据类型为 ggml_fp16_t

    for (int i = 0; i < nr; ++i) {  // 遍历源张量2的元素
        const int r = ((int32_t *) src1->data)[i];  // 获取源张量2中第 i 个元素的值

        for (int j = 0; j < nc; ++j) {  // 遍历源张量1的第一个维度
            ggml_fp16_t v = ((ggml_fp16_t *) ((char *) src0->data + i*src0->nb[1]))[j];  // 获取源张量1中指定位置的值
            ((float *) ((char *) dst->data + r*dst->nb[1]))[j] += GGML_FP16_TO_FP32(v);  // 更新目标张量中指定位置的值
        }
    }
}

// ggml_compute_forward_get_rows_back_f32 函数的实现，用于计算前向传播并更新目标张量数据
static void ggml_compute_forward_get_rows_back_f32(
        const struct ggml_compute_params * params,  // 参数结构体指针，包含计算参数信息
        const struct ggml_tensor * src0,  // 源张量1指针，用于计算
        const struct ggml_tensor * src1,  // 源张量2指针，用于计算
              struct ggml_tensor * dst) {  // 目标张量指针，用于存储计算结果
    GGML_ASSERT(params->ith == 0);  // 断言，确保参数中的 ith 值为 0
    GGML_ASSERT(ggml_is_contiguous(dst));  // 断言，确保目标张量是连续的

    // ggml_compute_forward_dup_same_cont(params, opt0, dst);  // 注释掉的函数调用

    if (params->type == GGML_TASK_INIT) {  // 如果任务类型为初始化
        if (params->ith != 0) {  // 如果 ith 不为 0
            return;  // 返回
        }
        memset(dst->data, 0, ggml_nbytes(dst));  // 将目标张量数据清零
    }

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  // 如果任务类型为初始化或结束
        return;  // 返回
    }

    const int nc = src0->ne[0];  // 获取源张量1的第一个维度大小
    const int nr = ggml_nelements(src1);  // 获取源张量2的元素个数

    GGML_ASSERT( dst->ne[0] == nc);  // 断言，确保目标张量的第一个维度大小与源张量1相同
    GGML_ASSERT(src0->nb[0] == sizeof(float));  // 断言，确保源张量1的数据类型为 float
    # 遍历循环，i 从 0 到 nr-1
    for (int i = 0; i < nr; ++i) {
        # 从 src1 数据中获取第 i 个元素的值，转换为整型
        const int r = ((int32_t *) src1->data)[i];

        # 在 dst 数据中的偏移位置 r*dst->nb[1] 处执行浮点数相加操作
        ggml_vec_add_f32(nc,
                (float *) ((char *)  dst->data + r*dst->nb[1]),
                # 将结果存储在 dst 数据中的偏移位置 r*dst->nb[1] 处
                (float *) ((char *)  dst->data + r*dst->nb[1]),
                # 将 src0 数据中的偏移位置 i*src0->nb[1] 处的数据作为第三个参数传入
                (float *) ((char *) src0->data + i*src0->nb[1]));
    }
}

// 根据不同的数据类型进行前向计算，获取行数据
static void ggml_compute_forward_get_rows_back(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 根据源张量的数据类型进行不同的操作
    switch (src0->type) {
        // 如果数据类型为 GGML_TYPE_F16
        case GGML_TYPE_F16:
            {
                // 调用相应的函数进行前向计算
                ggml_compute_forward_get_rows_back_f32_f16(params, src0, src1, dst);
            } break;
        // 如果数据类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用相应的函数进行前向计算
                ggml_compute_forward_get_rows_back_f32(params, src0, src1, dst);
            } break;
        // 默认情况
        default:
            {
                // 断言，如果不是以上两种数据类型，则为假
                GGML_ASSERT(false);
            } break;
    }

    // 下面的代码段被注释掉，可能是用于调试或测试
    //static bool first = true;
    //printf("ne0 = %d, ne1 = %d, ne2 = %d\n", dst->ne[0], dst->ne[1], dst->ne[2]);
    //if (first) {
    //    first = false;
    //} else {
    //    for (int k = 0; k < dst->ne[1]; ++k) {
    //        for (int j = 0; j < dst->ne[0]/16; ++j) {
    //            for (int i = 0; i < 16; ++i) {
    //                printf("%8.4f ", ((float *) dst->data)[k*dst->ne[0] + j*16 + i]);
    //            }
    //            printf("\n");
    //        }
    //        printf("\n");
    //    }
    //    printf("\n");
    //    exit(0);
    //}
}

// ggml_compute_forward_diag

// 根据数据类型进行前向对角线计算
static void ggml_compute_forward_diag_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言，如果 ith 不为 0，则为假
    GGML_ASSERT(params->ith == 0);

    // 如果任务类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // TODO: 处理转置/置换矩阵

    GGML_TENSOR_UNARY_OP_LOCALS

    // 断言，检查张量的维度和大小
    GGML_ASSERT(ne00 == ne0);
    GGML_ASSERT(ne00 == ne1);
    GGML_ASSERT(ne01 == 1);
    GGML_ASSERT(ne02 == ne2);
    GGML_ASSERT(ne03 == ne3);

    GGML_ASSERT(nb00 == sizeof(float));
    GGML_ASSERT(nb0  == sizeof(float));
}
    // 遍历第三维度
    for (int i3 = 0; i3 < ne3; i3++) {
        // 遍历第二维度
        for (int i2 = 0; i2 < ne2; i2++) {
            // 遍历第一维度
            for (int i1 = 0; i1 < ne1; i1++) {
                // 计算目标数据指针位置
                float * d = (float *)((char *)  dst->data + i3*nb3  + i2*nb2 + i1*nb1);
                // 计算源数据指针位置
                float * s = (float *)((char *) src0->data + i3*nb03 + i2*nb02);
                // 将目标数据中 i1 之前的元素置为 0
                for (int i0 = 0; i0 < i1; i0++) {
                    d[i0] = 0;
                }
                // 将目标数据中 i1 处的元素赋值为源数据中对应位置的值
                d[i1] = s[i1];
                // 将目标数据中 i1 之后的元素置为 0
                for (int i0 = i1+1; i0 < ne0; i0++) {
                    d[i0] = 0;
                }
            }
        }
    }
}

// 计算前向对角线操作
static void ggml_compute_forward_diag(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据源张量的数据类型进行不同的操作
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                // 调用针对 F32 类型的前向对角线计算函数
                ggml_compute_forward_diag_f32(params, src0, dst);
            } break;
        default:
            {
                // 断言错误，不支持的数据类型
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_diag_mask_inf

// 计算前向对角线掩码操作（针对 F32 类型）
static void ggml_compute_forward_diag_mask_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst,
        const float value) {

    // 获取参数中的 ith 和 nth
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取目标张量中的过去数据量
    const int  n_past  = ((int32_t *) dst->op_params)[0];
    // 检查是否为原地操作
    const bool inplace = src0->data == dst->data;

    // 断言过去数据量大于等于 0
    GGML_ASSERT(n_past >= 0);

    // 如果不是原地操作且为初始化任务类型
    if (!inplace && (params->type == GGML_TASK_INIT)) {
        if (ith != 0) {
            return;
        }
        // 在初始化阶段需要同步执行 memcpy 以避免竞争条件
        GGML_ASSERT(ggml_nelements(dst) == ggml_nelements(src0));
        GGML_ASSERT(ggml_is_contiguous(dst) && ggml_is_contiguous(src0));
        // 执行数据拷贝操作
        memcpy(
            ((char *)  dst->data),
            ((char *) src0->data),
            ggml_nbytes(dst));
    }

    // 如果任务类型为初始化或结束，直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // TODO: 处理转置/置换矩阵

    // 获取源张量的行数、列数和深度
    const int n  = ggml_nrows(src0);
    const int nc = src0->ne[0];
    const int nr = src0->ne[1];
    const int nz = n/nr;

    // 断言目标张量和源张量的数据类型为 float
    GGML_ASSERT( dst->nb[0] == sizeof(float));
    GGML_ASSERT(src0->nb[0] == sizeof(float));
}
    // 遍历第一维度 nz
    for (int k = 0; k < nz; k++) {
        // 遍历第二维度 nr，每次增加 nth 步长
        for (int j = ith; j < nr; j += nth) {
            // 遍历第三维度 nc，从 n_past 开始
            for (int i = n_past; i < nc; i++) {
                // 如果 i 大于 n_past + j，则执行以下操作
                if (i > n_past + j) {
                    // 计算 dst 中的偏移量，将 value 赋值给该位置
                    *(float *)((char *) dst->data + k*dst->nb[2] + j*dst->nb[1] + i*dst->nb[0]) = value;
                }
            }
        }
    }
// 计算前向对角线掩码为负无穷的操作
static void ggml_compute_forward_diag_mask_inf(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据源张量的数据类型进行不同的操作
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                // 调用具体的计算函数，传入参数为负无穷
                ggml_compute_forward_diag_mask_f32(params, src0, dst, -INFINITY);
            } break;
        default:
            {
                // 断言错误，不支持的数据类型
                GGML_ASSERT(false);
            } break;
    }
}

// 计算前向对角线掩码为零的操作
static void ggml_compute_forward_diag_mask_zero(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据源张量的数据类型进行不同的操作
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                // 调用具体的计算函数，传入参数为零
                ggml_compute_forward_diag_mask_f32(params, src0, dst, 0);
            } break;
        default:
            {
                // 断言错误，不支持的数据类型
                GGML_ASSERT(false);
            } break;
    }
}

// 计算前向 Softmax 操作
static void ggml_compute_forward_soft_max_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
              struct ggml_tensor * dst) {
    // 断言目标张量是连续的
    assert(ggml_is_contiguous(dst));
    // 断言源张量和目标张量具有相同的形状
    assert(ggml_are_same_shape(src0, dst));

    // 如果任务类型为初始化或结束，直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 初始化缩放因子为1.0
    float scale = 1.0f;
    // 从目标张量的操作参数中复制缩放因子
    memcpy(&scale, (float *) dst->op_params + 0, sizeof(float));

    // TODO: 处理转置/置换矩阵

    // 获取线程的索引和总线程数
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取 src1 的第二维大小
    const int64_t ne11 = src1 ? src1->ne[1] : 1;

    // 获取 src0 的列数和行数
    const int nc = src0->ne[0];
    const int nr = ggml_nrows(src0);

    // 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 本线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 计算参数中的浮点数数组指针，加上缓存行大小
    float * wp = (float *) params->wdata + (nc + CACHE_LINE_SIZE_F32) * ith;
}
    // 遍历 ir0 到 ir1 之间的整数
    for (int i1 = ir0; i1 < ir1; i1++) {
        // 获取源数据中第 i1 行的指针
        float * sp = (float *)((char *) src0->data + i1*src0->nb[1]);
        // 获取目标数据中第 i1 行的指针
        float * dp = (float *)((char *)  dst->data +  i1*dst->nb[1]);

        // 如果存在 src1，则获取 src1 中第 (i1%ne11) 行的指针，否则为 NULL
        float * mp = src1 ? (float *)((char *) src1->data + (i1%ne11)*src1->nb[1]) : NULL;

        // 复制源数据中第 i1 行的数据到 wp 中
        ggml_vec_cpy_f32  (nc, wp, sp);
        // 将 wp 中的数据按照 scale 进行缩放
        ggml_vec_scale_f32(nc, wp, scale);
        // 如果 mp 存在，则将 mp 中的数据累加到 wp 中
        if (mp) {
            ggml_vec_acc_f32(nc, wp, mp);
        }
    }
#ifndef NDEBUG
        // 如果处于调试模式，检查权重数组中是否存在 NaN 值
        for (int i = 0; i < nc; ++i) {
            //printf("p[%d] = %f\n", i, p[i]);
            // 断言权重数组中不包含 NaN 值
            assert(!isnan(wp[i]));
        }
#endif

        // 初始化最大值为负无穷
        float max = -INFINITY;
        // 找到权重数组中的最大值
        ggml_vec_max_f32(nc, &max, wp);

        // 初始化和值为 0
        ggml_float sum = 0.0;

        // 初始化用于存储转换后的半精度浮点数的变量
        uint16_t scvt;
        // 遍历权重数组
        for (int i = 0; i < nc; i++) {
            // 如果权重为负无穷，则输出为 0
            if (wp[i] == -INFINITY) {
                dp[i] = 0.0f;
            } else {
                // 将权重值转换为半精度浮点数，然后再转换回单精度浮点数
                ggml_fp16_t s = GGML_FP32_TO_FP16(wp[i] - max);
                memcpy(&scvt, &s, sizeof(scvt));
                const float val = GGML_FP16_TO_FP32(ggml_table_exp_f16[scvt]);
                sum += (ggml_float)val;
                dp[i] = val;
            }
        }

        // 断言和值大于 0
        assert(sum > 0.0);

        // 计算归一化因子
        sum = 1.0/sum;
        // 对输出进行缩放
        ggml_vec_scale_f32(nc, dp, sum);

#ifndef NDEBUG
        // 如果处于调试模式，检查输出数组中是否存在 NaN 和无穷值
        for (int i = 0; i < nc; ++i) {
            // 断言输出数组中不包含 NaN 值
            assert(!isnan(dp[i]));
            // 断言输出数组中不包含无穷值
            assert(!isinf(dp[i]));
        }
#endif
    }
}

// 根据输入张量类型选择对应的前向 Softmax 计算函数
static void ggml_compute_forward_soft_max(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
              struct ggml_tensor * dst) {
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                // 调用单精度浮点数类型的前向 Softmax 计算函数
                ggml_compute_forward_soft_max_f32(params, src0, src1, dst);
            } break;
        default:
            {
                // 如果类型不匹配，断言失败
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_soft_max_back

// 单精度浮点数类型的反向 Softmax 计算函数
static void ggml_compute_forward_soft_max_back_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 断言输入张量是连续的
    GGML_ASSERT(ggml_is_contiguous(src0));
    GGML_ASSERT(ggml_is_contiguous(src1));
    GGML_ASSERT(ggml_is_contiguous(dst));
    // 断言输入张量和输出张量具有相同的形状
    GGML_ASSERT(ggml_are_same_shape(src0, dst));
    GGML_ASSERT(ggml_are_same_shape(src1, dst));
    // 如果任务类型是初始化或者结束，则直接返回，不做处理
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // TODO: 处理转置/置换矩阵

    // 获取当前线程的索引和总线程数
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取源矩阵的列数和行数
    const int nc = src0->ne[0];
    const int nr = ggml_nrows(src0);

    // 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 遍历当前线程处理的行范围
    for (int i1 = ir0; i1 < ir1; i1++) {
        // 获取源矩阵、目标矩阵和结果矩阵当前行的指针
        float *dy = (float *)((char *) src0->data + i1*src0->nb[1]);
        float *y  = (float *)((char *) src1->data + i1*src1->nb[1]);
        float *dx = (float *)((char *) dst->data  + i1*dst->nb[1]);
#ifndef NDEBUG
        // 如果处于调试模式下
        for (int i = 0; i < nc; ++i) {
            // 遍历每个元素，确保不是 NaN
            //printf("p[%d] = %f\n", i, p[i]);
            assert(!isnan(dy[i]));
            assert(!isnan(y[i]));
        }
#endif
        // 计算 J 矩阵的各个元素
        // Jii = yi - yi*yi
        // Jij = -yi*yj
        // J = diag(y)-y.T*y
        // dx = J * dy
        // dxk = sum_i(Jki * dyi)
        // dxk = sum_i(-yk*yi * dyi) - (-yk*yk)*dyk + (yk - yk*yk)*dyk
        // dxk = sum_i(-yk*yi * dyi) + yk*yk*dyk + yk*dyk - yk*yk*dyk
        // dxk = sum_i(-yk*yi * dyi) + yk*dyk
        // dxk = -yk * sum_i(yi * dyi) + yk*dyk
        // dxk = -yk * dot(y, dy) + yk*dyk
        // dxk = yk * (- dot(y, dy) + dyk)
        // dxk = yk * (dyk - dot(y, dy))
        //
        // 后序遍历:
        // dot_y_dy := dot(y, dy)
        // dx := dy
        // dx := dx - dot_y_dy
        // dx := dx * y

        // 线性运行时间，不需要额外内存
        // 计算 y 和 dy 的点积
        float dot_y_dy = 0;
        ggml_vec_dot_f32 (nc, &dot_y_dy, y, dy);
        // 将 dy 复制到 dx
        ggml_vec_cpy_f32 (nc, dx, dy);
        // dx 减去点积
        ggml_vec_acc1_f32(nc, dx, -dot_y_dy);
        // dx 乘以 y
        ggml_vec_mul_f32 (nc, dx, dx, y);

#ifndef NDEBUG
        // 如果处于调试模式下
        for (int i = 0; i < nc; ++i) {
            // 确保 dx 中没有 NaN 或无穷大的值
            assert(!isnan(dx[i]));
            assert(!isinf(dx[i]));
        }
#endif
    }
}

// 计算 softmax 反向传播
static void ggml_compute_forward_soft_max_back(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                // 调用 f32 类型的 softmax 反向传播函数
                ggml_compute_forward_soft_max_back_f32(params, src0, src1, dst);
            } break;
        default:
            {
                // 抛出断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_alibi

// 计算 alibi 前向传播
static void ggml_compute_forward_alibi_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 确保 ith 为 0
    assert(params->ith == 0);
    // 如果任务类型是初始化或者结束，则直接返回，不执行后续操作
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 从目标操作参数中获取 n_head 的值
    const int n_head = ((int32_t *) dst->op_params)[1];
    float max_bias;
    // 从目标操作参数中获取 max_bias 的值
    memcpy(&max_bias, (int32_t *) dst->op_params + 2, sizeof(float));

    // 获取 src0 的 ne0, ne1, ne2 的值
    const int64_t ne0 = src0->ne[0]; // all_seq_len = n_past + ne1
    const int64_t ne1 = src0->ne[1]; // seq_len_without_past
    const int64_t ne2 = src0->ne[2]; // n_head -> this is k

    // 获取 src0 的行数 n
    const int64_t n  = ggml_nrows(src0);
    // 计算 ne2_ne3 的值
    const int64_t ne2_ne3 = n/ne1; // ne2*ne3

    // 获取 src0 的 nb0, nb1, nb2 的值
    const size_t nb0 = src0->nb[0];
    const size_t nb1 = src0->nb[1];
    const size_t nb2 = src0->nb[2];

    // 断言 nb0 的大小为 float 类型，n_head 的值等于 ne2
    GGML_ASSERT(nb0 == sizeof(float));
    GGML_ASSERT(n_head == ne2);

    // 计算 n_heads_log2_floor 的值
    const int n_heads_log2_floor = 1 << (int) floor(log2(n_head));

    // 计算 m0 和 m1 的值
    const float m0 = powf(2.0f, -(max_bias) / n_heads_log2_floor);
    const float m1 = powf(2.0f, -(max_bias / 2.0f) / n_heads_log2_floor);

    // 循环遍历 ne0, ne1, ne2_ne3，对数据进行处理
    for (int64_t i = 0; i < ne0; i++) {
        for (int64_t j = 0; j < ne1; j++) {
            for (int64_t k = 0; k < ne2_ne3; k++) {
                // 获取 src 和 pdst 的指针
                float * const src = (float *)((char *) src0->data + i*nb0 + j*nb1 + k*nb2);
                float *      pdst = (float *)((char *)  dst->data + i*nb0 + j*nb1 + k*nb2);

                // TODO: k*nb2 or k*nb3

                float m_k;

                // 根据 k 的值计算 m_k 的值
                if (k < n_heads_log2_floor) {
                    m_k = powf(m0, k + 1);
                } else {
                    m_k = powf(m1, 2 * (k - n_heads_log2_floor) + 1);
                }

                // 更新 pdst 的值
                pdst[0] = i * m_k + src[0];
            }
        }
    }
// 计算前向传播的 ALiBi 操作，根据给定参数、源张量和目标张量
static void ggml_compute_forward_alibi_f16(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数中的 ith 值为 0
    assert(params->ith == 0);

    // 如果参数类型为初始化或结束任务，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 从目标张量的操作参数中获取 n_head 值
    const int n_head = ((int32_t *) dst->op_params)[1];
    float max_bias;
    // 从目标张量的操作参数中获取 max_bias 值
    memcpy(&max_bias, (int32_t *) dst->op_params + 2, sizeof(float));

    // 获取源张量的 ne0、ne1、ne2 的值
    const int ne0 = src0->ne[0]; // all_seq_len = n_past + ne1
    const int ne1 = src0->ne[1]; // seq_len_without_past
    const int ne2 = src0->ne[2]; // n_head -> this is k

    // 获取源张量的 n 值
    const int n  = ggml_nrows(src0);
    const int ne2_ne3 = n/ne1; // ne2*ne3

    // 获取源张量的 nb0、nb1、nb2 的值
    const int nb0 = src0->nb[0];
    const int nb1 = src0->nb[1];
    const int nb2 = src0->nb[2];

    // 断言 nb0 的值为 ggml_fp16_t 的大小
    GGML_ASSERT(nb0 == sizeof(ggml_fp16_t);
    // 断言 n_head 的值等于 ne2
    GGML_ASSERT(n_head == ne2);

    // 计算 n_heads_log2_floor 值
    const int n_heads_log2_floor = 1 << (int) floor(log2(n_head));

    // 计算 m0 和 m1 的值
    const float m0 = powf(2.0f, -(max_bias) / n_heads_log2_floor);
    const float m1 = powf(2.0f, -(max_bias / 2.0f) / n_heads_log2_floor);
}
    // 循环遍历三维数组的每个元素
    for (int i = 0; i < ne0; i++) {
        for (int j = 0; j < ne1; j++) {
            for (int k = 0; k < ne2_ne3; k++) {
                // 计算源数据和目标数据的指针位置
                ggml_fp16_t * const src  = (ggml_fp16_t *)((char *) src0->data + i*nb0 + j*nb1 + k*nb2);
                float *      pdst  =       (float *)((char *)  dst->data + i*nb0 + j*nb1 + k*nb2);

                // TODO: k*nb2 or k*nb3
                // 待完成：确定是使用 k*nb2 还是 k*nb3

                float m_k;

                // 根据 k 的值计算 m_k
                if (k < n_heads_log2_floor) {
                    m_k = powf(m0, k + 1);
                } else {
                    m_k = powf(m1, 2 * (k - n_heads_log2_floor) + 1);
                }

                // 将计算结果赋值给目标数据数组
                // 返回 F32 类型数据
                pdst[0] = i * m_k + GGML_FP16_TO_FP32(src[0]);
            }
        }
    }
}

// 计算前向传播的阿里巴巴函数
static void ggml_compute_forward_alibi(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的处理
    switch (src0->type) {
        // 如果是 F16 类型
        case GGML_TYPE_F16:
            {
                // 调用 F16 类型的前向传播函数
                ggml_compute_forward_alibi_f16(params, src0, dst);
            } break;
        // 如果是 F32 类型
        case GGML_TYPE_F32:
            {
                // 调用 F32 类型的前向传播函数
                ggml_compute_forward_alibi_f32(params, src0, dst);
            } break;
        // 如果是其他类型
        case GGML_TYPE_Q4_0:
        case GGML_TYPE_Q4_1:
        case GGML_TYPE_Q5_0:
        case GGML_TYPE_Q5_1:
        case GGML_TYPE_Q8_0:
        case GGML_TYPE_Q8_1:
        case GGML_TYPE_Q2_K:
        case GGML_TYPE_Q3_K:
        case GGML_TYPE_Q4_K:
        case GGML_TYPE_Q5_K:
        case GGML_TYPE_Q6_K:
        case GGML_TYPE_IQ2_XXS:
        case GGML_TYPE_IQ2_XS:
        case GGML_TYPE_IQ3_XXS:
        case GGML_TYPE_Q8_K:
        case GGML_TYPE_I8:
        case GGML_TYPE_I16:
        case GGML_TYPE_I32:
        case GGML_TYPE_COUNT:
            {
                // 断言，抛出异常
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_clamp

// 计算前向传播的夹紧函数，针对 F32 类型
static void ggml_compute_forward_clamp_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言，确保参数 ith 为 0
    assert(params->ith == 0);

    // 如果任务类型为初始化或结束，直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    float min;
    float max;
    // 从目标张量的操作参数中复制最小值和最大值
    memcpy(&min, (float *) dst->op_params + 0, sizeof(float));
    memcpy(&max, (float *) dst->op_params + 1, sizeof(float));

    const int ith = params->ith;
    const int nth = params->nth;

    const int n  = ggml_nrows(src0);
    const int nc = src0->ne[0];

    const size_t nb00 = src0->nb[0];
    const size_t nb01 = src0->nb[1];

    const size_t nb0 = dst->nb[0];
    const size_t nb1 = dst->nb[1];

    // 断言，确保目标张量的第一个维度大小为 float 类型大小
    GGML_ASSERT( nb0 == sizeof(float));
    // 断言，确保源张量的第一个维度大小为 float 类型大小
    GGML_ASSERT(nb00 == sizeof(float));
    # 遍历数据，从第ith个元素开始，每次增加nth个元素
    for (int j = ith; j < n; j += nth) {
        # 计算目标数据指针的位置
        float * dst_ptr  = (float *) ((char *)  dst->data + j*nb1);
        # 计算源数据0指针的位置
        float * src0_ptr = (float *) ((char *) src0->data + j*nb01);

        # 遍历每个通道
        for (int i = 0; i < nc; i++) {
            # 将源数据0的值限制在[min, max]范围内，然后赋值给目标数据
            dst_ptr[i] = MAX(MIN(src0_ptr[i], max), min);
        }
    }
// 计算前向传播的夹紧操作，根据输入张量的类型选择不同的处理方式
static void ggml_compute_forward_clamp(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的数据类型进行不同的处理
    switch (src0->type) {
        // 如果数据类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用处理 GGML_TYPE_F32 类型数据的函数
                ggml_compute_forward_clamp_f32(params, src0, dst);
            } break;
        // 如果数据类型为其他类型
        case GGML_TYPE_F16:
        case GGML_TYPE_Q4_0:
        case GGML_TYPE_Q4_1:
        case GGML_TYPE_Q5_0:
        case GGML_TYPE_Q5_1:
        case GGML_TYPE_Q8_0:
        case GGML_TYPE_Q8_1:
        case GGML_TYPE_Q2_K:
        case GGML_TYPE_Q3_K:
        case GGML_TYPE_Q4_K:
        case GGML_TYPE_Q5_K:
        case GGML_TYPE_Q6_K:
        case GGML_TYPE_IQ2_XXS:
        case GGML_TYPE_IQ2_XS:
        case GGML_TYPE_IQ3_XXS:
        case GGML_TYPE_Q8_K:
        case GGML_TYPE_I8:
        case GGML_TYPE_I16:
        case GGML_TYPE_I32:
        case GGML_TYPE_COUNT:
            {
                // 断言，暂不支持的数据类型
                GGML_ASSERT(false);
            } break;
    }
}

// 计算绳索的斜坡值
static float rope_yarn_ramp(const float low, const float high, const int i0) {
    // 计算斜坡值
    const float y = (i0 / 2 - low) / MAX(0.001f, high - low);
    // 返回斜坡值
    return 1 - MIN(1, MAX(0, y));
}

// 基于 LlamaYaRNScaledRotaryEmbedding.py 中的 YaRN 算法
// MIT 许可证。版权所有 (c) 2023 Jeffrey Quesnelle 和 Bowen Peng。
// 计算绳索
static void rope_yarn(
    float theta_extrap, float freq_scale, float corr_dims[2], int64_t i0, float ext_factor, float mscale,
    float * cos_theta, float * sin_theta
) {
    // 获取旋转缩放校正后的 n 维旋转
    float theta_interp = freq_scale * theta_extrap;
    float theta = theta_interp;
}
    // 如果扩展因子不为0
    if (ext_factor != 0.0f) {
        // 计算绳子纱线斜坡混合值
        float ramp_mix = rope_yarn_ramp(corr_dims[0], corr_dims[1], i0) * ext_factor;
        // 根据斜坡混合值插值计算角度
        theta = theta_interp * (1 - ramp_mix) + theta_extrap * ramp_mix;

        // 获取经过插值校正的n维幅度缩放
        mscale *= 1.0f + 0.1f * logf(1.0f / freq_scale);
    }
    // 计算余弦角度并乘以幅度缩放
    *cos_theta = cosf(theta) * mscale;
    // 计算正弦角度并乘以幅度缩放
    *sin_theta = sinf(theta) * mscale;
// 结束当前函数
}

// 计算绳索维度的相关性
// 根据公式 `n_rot = 2pi * x * base^((2 * max_pos_emb) / n_dims)` 求解 x，得到 `corr_dim(n_rot) = n_dims * log(max_pos_emb / (n_rot * 2pi)) / (2 * log(base))`
static float ggml_rope_yarn_corr_dim(int n_dims, int n_orig_ctx, float n_rot, float base) {
    return n_dims * logf(n_orig_ctx / (n_rot * 2 * (float)M_PI)) / (2 * logf(base));
}

// 初始化绳索缓存
static void ggml_rope_cache_init(
     float theta_base, float freq_scale, float corr_dims[2], int64_t ne0, float ext_factor, float mscale,
     float * cache, float sin_sign, float theta_scale
) {
    float theta = theta_base;
    for (int64_t i0 = 0; i0 < ne0; i0 += 2) {
        rope_yarn(
            theta, freq_scale, corr_dims, i0, ext_factor, mscale, &cache[i0 + 0], &cache[i0 + 1]
        );
        cache[i0 + 1] *= sin_sign;

        theta *= theta_scale;
    }
}

// 计算绳索维度的相关性
GGML_CALL void ggml_rope_yarn_corr_dims(
    int n_dims, int n_orig_ctx, float freq_base, float beta_fast, float beta_slow, float dims[2]
) {
    // 开始和结束的修正维度
    dims[0] = MAX(0,         floorf(ggml_rope_yarn_corr_dim(n_dims, n_orig_ctx, beta_fast, freq_base)));
    dims[1] = MIN(n_dims - 1, ceilf(ggml_rope_yarn_corr_dim(n_dims, n_orig_ctx, beta_slow, freq_base)));
}

// 计算前向传播的绳索浮点数
static void ggml_compute_forward_rope_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst,
        const bool forward) {
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    float freq_base, freq_scale, ext_factor, attn_factor, beta_fast, beta_slow;

    // 仅对 xPos RoPE 有关：
    float xpos_base;
    bool  xpos_down;

    //const int n_past     = ((int32_t *) dst->op_params)[0];
    const int n_dims     = ((int32_t *) dst->op_params)[1];
    const int mode       = ((int32_t *) dst->op_params)[2];
    const int n_ctx      = ((int32_t *) dst->op_params)[3];
}
    // 从目标操作参数中获取原始上下文数
    const int n_orig_ctx = ((int32_t *) dst->op_params)[4];

    // 从目标操作参数中获取频率基数
    memcpy(&freq_base,   (int32_t *) dst->op_params +  5, sizeof(float));
    // 从目标操作参数中获取频率比例
    memcpy(&freq_scale,  (int32_t *) dst->op_params +  6, sizeof(float));
    // 从目标操作参数中获取扩展因子
    memcpy(&ext_factor,  (int32_t *) dst->op_params +  7, sizeof(float));
    // 从目标操作参数中获取注意力因子
    memcpy(&attn_factor, (int32_t *) dst->op_params +  8, sizeof(float));
    // 从目标操作参数中获取快速 beta 值
    memcpy(&beta_fast,   (int32_t *) dst->op_params +  9, sizeof(float));
    // 从目标操作参数中获取慢速 beta 值
    memcpy(&beta_slow,   (int32_t *) dst->op_params + 10, sizeof(float));
    // 从目标操作参数中获取 x 位置基数
    memcpy(&xpos_base,   (int32_t *) dst->op_params + 11, sizeof(float));
    // 从目标操作参数中获取 x 位置下降标志
    memcpy(&xpos_down,   (int32_t *) dst->op_params + 12, sizeof(bool));

    // 定义 GGML_TENSOR_UNARY_OP_LOCALS

    // 打印 ne0, ne1, ne2, ne3 的值
    // 打印 n_past 和 ne2 的值
    // 断言 nb00 的值为 sizeof(float)

    // 获取 params 中的 ith 和 nth 值
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取 dst 的行数
    const int nr = ggml_nrows(dst);

    // 断言 n_dims 小于等于 ne0
    // 断言 n_dims 为偶数

    // 计算每个线程的行数
    const int dr = (nr + nth - 1)/nth;

    // 计算当前线程的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 初始化行索引为 0

    // 计算 theta_scale
    // 计算 inv_ndims
    // 计算 corr_dims
    ggml_rope_yarn_corr_dims(n_dims, n_orig_ctx, freq_base, beta_fast, beta_slow, corr_dims);

    // 检查是否为 neox 模式
    // 检查是否为 glm 模式

    // 如果是反向过程，则 sin_sign 为 1.0，否则为 -1.0

    // 获取 src1 数据的指针
    const int32_t * pos = (const int32_t *) src1->data;
// 计算前向传播的绳索操作，根据给定的参数、源张量和目标张量进行计算
static void ggml_compute_forward_rope_f16(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst,
        const bool forward) {
    // 如果参数类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    float freq_base, freq_scale, ext_factor, attn_factor, beta_fast, beta_slow;

    // 从目标张量的操作参数中获取相关参数值
    const int n_dims     = ((int32_t *) dst->op_params)[1];
    const int mode       = ((int32_t *) dst->op_params)[2];
    const int n_ctx      = ((int32_t *) dst->op_params)[3];
    const int n_orig_ctx = ((int32_t *) dst->op_params)[4];
    memcpy(&freq_base,   (int32_t *) dst->op_params +  5, sizeof(float));
    memcpy(&freq_scale,  (int32_t *) dst->op_params +  6, sizeof(float));
    memcpy(&ext_factor,  (int32_t *) dst->op_params +  7, sizeof(float));
    memcpy(&attn_factor, (int32_t *) dst->op_params +  8, sizeof(float));
    memcpy(&beta_fast,   (int32_t *) dst->op_params +  9, sizeof(float));
    memcpy(&beta_slow,   (int32_t *) dst->op_params + 10, sizeof(float));

    GGML_TENSOR_UNARY_OP_LOCALS

    // 确保目标张量的数据类型大小为 ggml_fp16_t
    GGML_ASSERT(nb0 == sizeof(ggml_fp16_t));

    const int ith = params->ith;
    const int nth = params->nth;

    const int nr = ggml_nrows(dst);

    GGML_ASSERT(n_dims <= ne0);
    GGML_ASSERT(n_dims % 2 == 0);

    // 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 用于确定使用哪个线程的行索引
    int ir = 0;

    // 计算频率缩放因子和维度相关参数
    const float theta_scale = powf(freq_base, -2.0f/n_dims);
    const float inv_ndims = -1.f/n_dims;
    float corr_dims[2];
    // 调用函数 ggml_rope_yarn_corr_dims，传入参数 n_dims, n_orig_ctx, freq_base, beta_fast, beta_slow, corr_dims
    ggml_rope_yarn_corr_dims(n_dims, n_orig_ctx, freq_base, beta_fast, beta_slow, corr_dims);

    // 检查 mode 中是否包含 2 的位，将结果存储在 is_neox 中
    const bool is_neox = mode & 2;
    // 检查 mode 中是否包含 4 的位，将结果存储在 is_glm 中
    const bool is_glm  = mode & 4;

    // 如果是 forward 模式，sin_sign 为 1.0，否则为 -1.0
    // 这里描述了反向过程使用 cos 和 sin 的逆旋转，sin 的符号会被改变
    const float sin_sign = forward ? 1.0f : -1.0f;

    // 将 src1->data 强制转换为 const int32_t 指针类型，赋值给 pos
    const int32_t * pos = (const int32_t *) src1->data;

    // 代码块结束
// 计算前向传播的 ROPE 操作，根据输入张量类型选择对应的操作函数
static void ggml_compute_forward_rope(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 根据输入张量 src0 的类型进行分支选择
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F16
        case GGML_TYPE_F16:
            {
                // 调用对应的 F16 类型的 ROPE 操作函数
                ggml_compute_forward_rope_f16(params, src0, src1, dst, true);
            } break;
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用对应的 F32 类型的 ROPE 操作函数
                ggml_compute_forward_rope_f32(params, src0, src1, dst, true);
            } break;
        // 默认情况下
        default:
            {
                // 断言错误，应该不会执行到这里
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_rope_back

// 计算反向传播的 ROPE 操作
static void ggml_compute_forward_rope_back(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 根据输入张量 src0 的类型进行分支选择
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F16
        case GGML_TYPE_F16:
            {
                // 调用对应的 F16 类型的 ROPE 操作函数
                ggml_compute_forward_rope_f16(params, src0, src1, dst, false);
            } break;
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用对应的 F32 类型的 ROPE 操作函数
                ggml_compute_forward_rope_f32(params, src0, src1, dst, false);
            } break;
        // 默认情况下
        default:
            {
                // 断言错误，应该不会执行到这里
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_conv_transpose_1d

// 计算一维卷积转置的前向传播操作，根据输入张量类型选择对应的操作函数
static void ggml_compute_forward_conv_transpose_1d_f16_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
              struct ggml_tensor * dst) {
    // 断言输入张量类型为 GGML_TYPE_F16
    GGML_ASSERT(src0->type == GGML_TYPE_F16);
    // 断言输入张量类型为 GGML_TYPE_F32
    GGML_ASSERT(src1->type == GGML_TYPE_F32);
    // 断言目标张量类型为 GGML_TYPE_F32
    GGML_ASSERT( dst->type == GGML_TYPE_F32);

    // 获取当前时间
    int64_t t0 = ggml_perf_time_us();
    // 未使用 t0，避免编译器警告
    UNUSED(t0);

    // 定义本地变量
    GGML_TENSOR_BINARY_OP_LOCALS

    // 获取 params 中的 ith 和 nth
    const int ith = params->ith;
    const int nth = params->nth;

    // 计算 nk
    const int nk = ne00*ne01*ne02;

    // 断言 nb00 的大小为 ggml_fp16_t 的大小
    GGML_ASSERT(nb00 == sizeof(ggml_fp16_t));
    // 断言 nb10 的大小为 float 的大小
    GGML_ASSERT(nb10 == sizeof(float));
}
    // 如果任务类型为初始化
    if (params->type == GGML_TASK_INIT) {
        // 如果不是第一个线程，则直接返回
        if (ith != 0) {
            return;
        }
        // 将 params->wdata 内存块清零
        memset(params->wdata, 0, params->wsize);

        // 对内核数据（src0）进行排列，从（K x Cout x Cin）到（Cin x K x Cout）
        {
            // 获取 wdata 指针，并偏移0个单位
            ggml_fp16_t * const wdata = (ggml_fp16_t *) params->wdata + 0;

            // 遍历 ne02
            for (int64_t i02 = 0; i02 < ne02; i02++) {
                // 遍历 ne01
                for (int64_t i01 = 0; i01 < ne01; i01++) {
                    // 计算 src 指针位置
                    const ggml_fp16_t * const src = (ggml_fp16_t *)((char *) src0->data + i02*nb02 + i01*nb01);
                    // 计算 dst_data 指针位置
                    ggml_fp16_t * dst_data = wdata + i01*ne00*ne02;
                    // 遍历 ne00
                    for (int64_t i00 = 0; i00 < ne00; i00++) {
                        // 将 src 数据复制到 dst_data
                        dst_data[i00*ne02 + i02] = src[i00];
                    }
                }
            }
        }

        // 对源数据（src1）进行排列，从（L x Cin）到（Cin x L）
        {
            // 获取 wdata 指针，并偏移 nk 个单位
            ggml_fp16_t * const wdata = (ggml_fp16_t *) params->wdata + nk;
            ggml_fp16_t * dst_data = wdata;

            // 遍历 ne11
            for (int64_t i11 = 0; i11 < ne11; i11++) {
                // 计算 src 指针位置
                const float * const src = (float *)((char *) src1->data + i11*nb11);
                // 遍历 ne10
                for (int64_t i10 = 0; i10 < ne10; i10++) {
                    // 将 src 数据转换为 fp16 格式后复制到 dst_data
                    dst_data[i10*ne11 + i11] = GGML_FP32_TO_FP16(src[i10]);
                }
            }
        }

        // 需要将 dst 清零，因为我们要对其进行累加
        memset(dst->data, 0, ggml_nbytes(dst));

        return;
    }

    // 如果任务类型为结束
    if (params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取 dst->op_params 中的第一个 int32_t 值
    const int32_t s0 = ((const int32_t*)(dst->op_params))[0];

    // dst 中的总行数
    const int nr = ne1;

    // 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 该线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 获取 wdata 指针，并偏移0个单位
    ggml_fp16_t * const wdata = (ggml_fp16_t *) params->wdata + 0;
    // 获取 wdata_src 指针，并偏移 nk 个单位
    ggml_fp16_t * const wdata_src = wdata + nk;
    // 遍历 ir0 到 ir1 之间的索引
    for (int i1 = ir0; i1 < ir1; i1++) {
        // 计算目标数据的指针位置
        float * dst_data = (float *)((char *) dst->data + i1*nb1);
        // 计算权重数据的指针位置
        ggml_fp16_t * wdata_kernel = wdata + i1*ne02*ne00;
        // 遍历 ne10 个元素
        for (int i10 = 0; i10 < ne10; i10++) {
            // 计算 i1n 的值
            const int i1n = i10*ne11;
            // 遍历 ne00 个元素
            for (int i00 = 0; i00 < ne00; i00++) {
                // 初始化 v 为 0
                float v = 0;
                // 计算两个向量的点积并将结果存储在 v 中
                ggml_vec_dot_f16(ne02, &v,
                        (ggml_fp16_t *)    wdata_src + i1n,
                        (ggml_fp16_t *) wdata_kernel + i00*ne02);
                // 将计算结果加到目标数据中的指定位置
                dst_data[i10*s0 + i00] += v;
            }
        }
    }
// 定义一个静态函数，用于计算一维转置卷积的前向传播，参数包括输入张量 src0、src1 和输出张量 dst
static void ggml_compute_forward_conv_transpose_1d_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
              struct ggml_tensor * dst) {
    // 断言输入张量 src0 的数据类型为 GGML_TYPE_F32
    GGML_ASSERT(src0->type == GGML_TYPE_F32);
    // 断言输入张量 src1 的数据类型为 GGML_TYPE_F32
    GGML_ASSERT(src1->type == GGML_TYPE_F32);
    // 断言输出张量 dst 的数据类型为 GGML_TYPE_F32
    GGML_ASSERT( dst->type == GGML_TYPE_F32);

    // 记录当前时间
    int64_t t0 = ggml_perf_time_us();
    UNUSED(t0);

    // 定义本地变量
    GGML_TENSOR_BINARY_OP_LOCALS

    // 获取参数中的 ith 和 nth
    const int ith = params->ith;
    const int nth = params->nth;

    // 计算 nk
    const int nk = ne00*ne01*ne02;

    // 断言 nb00 和 nb10 的大小为 sizeof(float)
    GGML_ASSERT(nb00 == sizeof(float));
    GGML_ASSERT(nb10 == sizeof(float));

    // 如果任务类型为 GGML_TASK_INIT
    if (params->type == GGML_TASK_INIT) {
        // 如果 ith 不为 0，则直接返回
        if (ith != 0) {
            return;
        }
        // 将 params->wdata 的内容全部置为 0
        memset(params->wdata, 0, params->wsize);

        // 准备卷积核数据（src0）从 (K x Cout x Cin) 转置为 (Cin x K x Cout)
        {
            float * const wdata = (float *) params->wdata + 0;

            for (int64_t i02 = 0; i02 < ne02; i02++) {
                for (int64_t i01 = 0; i01 < ne01; i01++) {
                    const float * const src = (float *)((char *) src0->data + i02*nb02 + i01*nb01);
                    float * dst_data = wdata + i01*ne00*ne02;
                    for (int64_t i00 = 0; i00 < ne00; i00++) {
                        dst_data[i00*ne02 + i02] = src[i00];
                    }
                }
            }
        }

        // 准备源数据（src1）
        {
            float * const wdata = (float *) params->wdata + nk;
            float * dst_data = wdata;

            for (int64_t i11 = 0; i11 < ne11; i11++) {
                const float * const src = (float *)((char *) src1->data + i11*nb11);
                for (int64_t i10 = 0; i10 < ne10; i10++) {
                    dst_data[i10*ne11 + i11] = src[i10];
                }
            }
        }

        // 需要将输出张量 dst 置为 0，因为要对其进行累加
        memset(dst->data, 0, ggml_nbytes(dst));

        return;
    }
}
    // 如果参数类型为 GGML_TASK_FINALIZE，则直接返回，不执行后续操作
    if (params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 从目标操作参数中获取第一个整数值
    const int32_t s0 = ((const int32_t*)(dst->op_params))[0];

    // 计算目标数据的总行数
    const int nr = ne1;

    // 计算每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 计算当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 获取参数中的 wdata 指针，并偏移 nk 个位置得到 wdata_src 指针
    float * const wdata     = (float *) params->wdata + 0;
    float * const wdata_src = wdata + nk;

    // 遍历处理当前线程负责的行范围
    for (int i1 = ir0; i1 < ir1; i1++) {
        // 获取目标数据中当前行的指针
        float * dst_data = (float *)((char *) dst->data + i1*nb1);
        // 获取当前行对应的 wdata_kernel 指针
        float * wdata_kernel = wdata + i1*ne02*ne00;
        // 遍历 ne10
        for (int i10 = 0; i10 < ne10; i10++) {
            // 计算 i1n
            const int i1n = i10*ne11;
            // 遍历 ne00
            for (int i00 = 0; i00 < ne00; i00++) {
                // 初始化 v 为 0
                float v = 0;
                // 调用 ggml_vec_dot_f32 函数计算点积并更新 v
                ggml_vec_dot_f32(ne02, &v,
                        wdata_src + i1n,
                        wdata_kernel + i00*ne02);
                // 更新目标数据中的值
                dst_data[i10*s0 + i00] += v;
            }
        }
    }
// 定义一个静态函数，用于计算一维卷积转置操作
static void ggml_compute_forward_conv_transpose_1d(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
              struct ggml_tensor * dst) {
    // 根据输入张量 src0 的数据类型进行不同的操作
    switch (src0->type) {
        // 如果数据类型为 GGML_TYPE_F16
        case GGML_TYPE_F16:
            {
                // 调用相应的函数进行 F16 到 F32 的转置操作
                ggml_compute_forward_conv_transpose_1d_f16_f32(params, src0, src1, dst);
            } break;
        // 如果数据类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用相应的函数进行 F32 的转置操作
                ggml_compute_forward_conv_transpose_1d_f32(params, src0, src1, dst);
            } break;
        // 默认情况下
        default:
            {
                // 断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// src0: kernel [OC, IC, KH, KW]
// src1: image [N, IC, IH, IW]
// dst:  result [N, OH, OW, IC*KH*KW]
// 定义一个静态函数，用于将输入张量 src0 和 src1 转换为 dst 张量
static void ggml_compute_forward_im2col_f16(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
              struct ggml_tensor * dst) {
    // 断言输入张量的数据类型
    GGML_ASSERT(src0->type == GGML_TYPE_F16);
    GGML_ASSERT(src1->type == GGML_TYPE_F32);
    GGML_ASSERT( dst->type == GGML_TYPE_F16);

    // 定义变量 t0 并记录当前时间
    int64_t t0 = ggml_perf_time_us();
    UNUSED(t0);

    // 定义本地变量
    GGML_TENSOR_BINARY_OP_LOCALS;

    // 从 dst 的操作参数中获取相关参数
    const int32_t s0 = ((const int32_t *)(dst->op_params))[0];
    const int32_t s1 = ((const int32_t *)(dst->op_params))[1];
    const int32_t p0 = ((const int32_t *)(dst->op_params))[2];
    const int32_t p1 = ((const int32_t *)(dst->op_params))[3];
    const int32_t d0 = ((const int32_t *)(dst->op_params))[4];
    const int32_t d1 = ((const int32_t *)(dst->op_params))[5];
    const bool is_2D = ((const int32_t *)(dst->op_params))[6] == 1;

    // 获取 params 中的 ith 和 nth 参数
    const int ith = params->ith;
    const int nth = params->nth;

    // 根据是否为二维数据，定义不同的维度参数
    const int64_t N  = is_2D ? ne13 : ne12;
    const int64_t IC = is_2D ? ne12 : ne11;
    const int64_t IH = is_2D ? ne11 : 1;
    const int64_t IW = ne10;

    const int64_t KH = is_2D ? ne01 : 1;
    const int64_t KW = ne00;

    const int64_t OH = is_2D ? ne2 : 1;
    // 定义 OW 常量，值为 ne1
    const int64_t OW = ne1;

    // 根据是否为二维，确定 ofs0 和 ofs1 的值
    int ofs0 = is_2D ? nb13 : nb12;
    int ofs1 = is_2D ? nb12 : nb11;

    // 断言 nb00 的值等于 ggml_fp16_t 的大小
    GGML_ASSERT(nb00 == sizeof(ggml_fp16_t));
    // 断言 nb10 的值等于 float 的大小
    GGML_ASSERT(nb10 == sizeof(float));

    // 如果参数类型为 GGML_TASK_INIT，则直接返回
    if (params->type == GGML_TASK_INIT) {
        return;
    }

    // 如果参数类型为 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // im2col: [N, IC, IH, IW] => [N, OH, OW, IC*KH*KW]
    {
        // 将目标数据转换为 ggml_fp16_t 类型指针
        ggml_fp16_t * const wdata = (ggml_fp16_t *) dst->data;

        // 遍历 N
        for (int64_t in = 0; in < N; in++) {
            // 遍历 OH
            for (int64_t ioh = 0; ioh < OH; ioh++) {
                // 遍历 OW
                for (int64_t iow = 0; iow < OW; iow++) {
                    // 遍历 IC
                    for (int64_t iic = ith; iic < IC; iic += nth) {

                        // 微内核
                        ggml_fp16_t * dst_data = wdata + (in*OH*OW + ioh*OW + iow)*(IC*KH*KW); // [IC, KH, KW]
                        const float * const src_data = (float *)((char *) src1->data + in*ofs0 + iic*ofs1); // [IH, IW]

                        // 遍历 KH
                        for (int64_t ikh = 0; ikh < KH; ikh++) {
                            // 遍历 KW
                            for (int64_t ikw = 0; ikw < KW; ikw++) {
                                // 计算 iiw 和 iih
                                const int64_t iiw = iow*s0 + ikw*d0 - p0;
                                const int64_t iih = ioh*s1 + ikh*d1 - p1;

                                // 如果超出边界，则将值设为 0，否则转换为 ggml_fp16_t 类型后存入目标数据
                                if (iih < 0 || iih >= IH || iiw < 0 || iiw >= IW) {
                                    dst_data[iic*(KH*KW) + ikh*KW + ikw] = 0;
                                } else {
                                    dst_data[iic*(KH*KW) + ikh*KW + ikw] = GGML_FP32_TO_FP16(src_data[iih*IW + iiw]);
                                }
                            }
                        }
                    }
                }
            }
        }
    }
// 根据输入的两个源张量和参数计算前向卷积转置操作，结果存储在目标张量中
static void ggml_compute_forward_conv_transpose_2d(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
              struct ggml_tensor * dst) {
    // 断言源张量 src0 的数据类型为 GGML_TYPE_F16
    GGML_ASSERT(src0->type == GGML_TYPE_F16);
    // 断言源张量 src1 的数据类型为 GGML_TYPE_F32
    GGML_ASSERT(src1->type == GGML_TYPE_F32);
    // 断言目标张量 dst 的数据类型为 GGML_TYPE_F32
    GGML_ASSERT( dst->type == GGML_TYPE_F32);

    // 记录当前时间
    int64_t t0 = ggml_perf_time_us();
    // 未使用 t0，避免编译器警告
    UNUSED(t0);

    // 定义局部变量
    GGML_TENSOR_BINARY_OP_LOCALS

    // 获取参数中的 ith 和 nth 值
    const int ith = params->ith;
    const int nth = params->nth;

    // 计算 nk 值
    const int nk = ne00*ne01*ne02*ne03;

    // 断言 nb00 的大小为 ggml_fp16_t 类型的大小
    GGML_ASSERT(nb00 == sizeof(ggml_fp16_t);
    // 断言 nb10 的大小为 float 类型的大小
    GGML_ASSERT(nb10 == sizeof(float));
}
    // 如果任务类型为初始化
    if (params->type == GGML_TASK_INIT) {
        // 如果不是第一个线程，则直接返回
        if (ith != 0) {
            return;
        }
        // 将 params->wdata 内存块清零
        memset(params->wdata, 0, params->wsize);

        // 重新排列内核数据（src0）从（Kw x Kh x Cout x Cin）到（Cin x Kw x Kh x Cout）
        {
            // 获取 params->wdata 的指针，并转换为 ggml_fp16_t 类型
            ggml_fp16_t * const wdata = (ggml_fp16_t *) params->wdata + 0;

            for (int64_t i03 = 0; i03 < ne03; i03++) {
                for (int64_t i02 = 0; i02 < ne02; i02++) {
                    // 计算源数据的指针位置
                    const ggml_fp16_t * const src = (ggml_fp16_t *)((char *) src0->data + i03*nb03 + i02*nb02);
                    // 计算目标数据的指针位置
                    ggml_fp16_t * dst_data = wdata + i02*ne01*ne00*ne03;
                    for (int64_t i01 = 0; i01 < ne01; i01++) {
                        for (int64_t i00 = 0; i00 < ne00; i00++) {
                            // 重新排列数据
                            dst_data[i01*ne00*ne03 + i00*ne03 + i03] = src[i01 * ne00 + i00];
                        }
                    }
                }
            }
        }

        // 重新排列源数据（src1）从（Sw x Sh x Cin）到（Cin x Sw x Sh）
        {
            // 获取 params->wdata 的指针，并转换为 ggml_fp16_t 类型
            ggml_fp16_t * const wdata = (ggml_fp16_t *) params->wdata + nk;
            for (int i12 = 0; i12 < ne12; i12++) {
                for (int i11 = 0; i11 < ne11; i11++) {
                    // 计算源数据的指针位置
                    const float * const src = (float *)((char *) src1->data + i12*nb12 + i11*nb11);
                    // 计算目标数据的指针位置
                    ggml_fp16_t * dst_data = wdata + i11*ne10*ne12;
                    for (int i10 = 0; i10 < ne10; i10++) {
                        // 将 float 类型数据转换为 ggml_fp16_t 类型，并重新排列数据
                        dst_data[i10*ne12 + i12] = GGML_FP32_TO_FP16(src[i10]);
                    }
                }
            }
        }

        // 将目标数据内存块清零
        memset(dst->data, 0, ggml_nbytes(dst));

        return;
    }

    // 如果任务类型为结束，直接返回
    if (params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取 dst 的第一个参数作为步长
    const int32_t stride = ggml_get_op_params_i32(dst, 0);

    // 计算 dst 中总共的补丁数
    const int np = ne2;

    // 计算每个线程处理的补丁数
    const int dp = (np + nth - 1)/nth;

    // 计算当前线程处理的补丁范围
    const int ip0 = dp*ith;
    // 计算 ip1 的值，取 ip0 + dp 和 np 中的较小值
    const int ip1 = MIN(ip0 + dp, np);

    // 定义指向参数 wdata 的指针，并偏移 0 个元素
    ggml_fp16_t * const wdata = (ggml_fp16_t *) params->wdata + 0;
    // 定义指向 wdata 的指针，并偏移 nk 个元素
    ggml_fp16_t * const wdata_src = wdata + nk;

    // 遍历 Cout 的范围 [ip0, ip1)
    for (int i2 = ip0; i2 < ip1; i2++) { // Cout
        // 计算 dst_data 的地址，偏移 i2*nb2 个字节
        float * dst_data = (float *)((char *) dst->data + i2*nb2);
        // 计算 wdata_kernel 的地址，偏移 i2*ne01*ne00*ne03 个元素
        ggml_fp16_t * wdata_kernel = wdata + i2*ne01*ne00*ne03;
        // 遍历 ne11
        for (int i11 = 0; i11 < ne11; i11++) {
            // 遍历 ne10
            for (int i10 = 0; i10 < ne10; i10++) {
                // 计算 i1n 的值
                const int i1n = i11*ne10*ne12 + i10*ne12;
                // 遍历 ne01
                for (int i01 = 0; i01 < ne01; i01++) {
                    // 遍历 ne00
                    for (int i00 = 0; i00 < ne00; i00++) {
                        // 初始化 v 为 0
                        float v = 0;
                        // 计算点积并将结果存储在 v 中
                        ggml_vec_dot_f16(ne03, &v,
                                wdata_src + i1n,
                                wdata_kernel + i01*ne00*ne03 + i00*ne03);
                        // 将 v 加到 dst_data 的相应位置上
                        dst_data[(i11*stride + i01)*ne0 + i10*stride + i00] += v;
                    }
                }
            }
        }
    }
// 定义一个静态函数，用于计算一维池化操作的前向传播，参数包括计算参数、操作类型、源张量、k值和目标张量
static void ggml_compute_forward_pool_1d_sk_p0(
        const struct ggml_compute_params * params,
        const enum ggml_op_pool op,
        const struct ggml_tensor * src,
        const int k,
        struct ggml_tensor * dst) {
    // 断言源张量的数据类型为 GGML_TYPE_F32
    assert(src->type == GGML_TYPE_F32);
    // 断言计算参数的 ith 值为 0
    assert(params->ith == 0);

    // 如果计算参数的类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 将源张量的数据转换为字符型指针
    const char * cdata = (const char *)src->data;
    // 计算数据结束位置
    const char * const data_end = cdata + ggml_nbytes(src);
    // 将目标张量的数据转换为浮点型指针
    float * drow = (float *)dst->data;

    // 获取目标张量的第一个维度大小
    const int64_t rs = dst->ne[0];

    // 循环遍历源张量数据
    while (cdata < data_end) {
        // 将当前位置的数据转换为浮点型指针
        const float * const srow = (const float *)cdata;

        // 初始化变量 j 为 0
        int j = 0;

        // 循环遍历目标张量的第一个维度
        for (int64_t i = 0; i < rs; ++i) {
            // 根据操作类型进行不同的处理
            switch (op) {
                case GGML_OP_POOL_AVG:   drow[i] = 0;        break;
                case GGML_OP_POOL_MAX:   drow[i] = -FLT_MAX; break;
                case GGML_OP_POOL_COUNT: GGML_ASSERT(false); break;
            }
            // 循环遍历 k 值
            for (int ki = 0; ki < k; ++ki) {
                // 根据操作类型进行不同的处理
                switch (op) {
                    case GGML_OP_POOL_AVG:                          drow[i] += srow[j]; break;
                    case GGML_OP_POOL_MAX:   if (srow[j] > drow[i]) drow[i]  = srow[j]; break;
                    case GGML_OP_POOL_COUNT:                        GGML_ASSERT(false); break;
                }
                ++j;
            }
            // 根据操作类型进行不同的处理
            switch (op) {
                case GGML_OP_POOL_AVG:         drow[i] /= k; break;
                case GGML_OP_POOL_MAX:                       break;
                case GGML_OP_POOL_COUNT: GGML_ASSERT(false); break;
            }
        }

        // 更新源张量数据位置和目标张量数据位置
        cdata += src->nb[1];
        drow  += rs;
    }
}

// 定义一个静态函数，用于计算一维池化操作的前向传播，参数包括计算参数、源张量和目标张量
static void ggml_compute_forward_pool_1d(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
              struct ggml_tensor * dst) {
    // 将目标数据结构中的操作参数转换为整型指针数组
    const int32_t * opts = (const int32_t *)dst->op_params;
    // 从操作参数数组中获取操作类型
    enum ggml_op_pool op = opts[0];
    // 从操作参数数组中获取卷积核大小
    const int k0 = opts[1];
    // 从操作参数数组中获取步长大小
    const int s0 = opts[2];
    // 从操作参数数组中获取填充大小
    const int p0 = opts[3];
    // 断言填充大小为0，不支持填充
    GGML_ASSERT(p0 == 0); // padding not supported
    // 断言卷积核大小等于步长大小，只支持步长等于卷积核大小
    GGML_ASSERT(k0 == s0); // only s = k supported

    // 调用函数计算一维池化操作的前向传播，传入参数、操作类型、输入数据、卷积核大小和目标数据结构
    ggml_compute_forward_pool_1d_sk_p0(params, op, src0, k0, dst);
// 定义一个静态函数，用于计算2D池化操作的前向传播
static void ggml_compute_forward_pool_2d(
        const struct ggml_compute_params * params,  // 接收计算参数的结构体指针
        const struct ggml_tensor * src,  // 接收输入张量的结构体指针
        struct ggml_tensor * dst) {  // 接收输出张量的结构体指针
    assert(src->type == GGML_TYPE_F32);  // 断言输入张量的数据类型为float32
    assert(params->ith == 0);  // 断言参数中的ith字段为0

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  // 如果参数中的任务类型为初始化或结束，则直接返回
        return;
    }

    const int32_t * opts = (const int32_t *)dst->op_params;  // 从输出张量的操作参数中获取int32类型的参数数组
    enum ggml_op_pool op = opts[0];  // 获取池化操作类型
    const int k0 = opts[1];  // 获取池化核的大小
    const int k1 = opts[2];  // 获取池化核的大小
    const int s0 = opts[3];  // 获取步长
    const int s1 = opts[4];  // 获取步长
    const int p0 = opts[5];  // 获取填充大小
    const int p1 = opts[6];  // 获取填充大小
    const char * cdata = (const char*)src->data;  // 将输入张量的数据转换为char类型指针
    const char * const data_end = cdata + ggml_nbytes(src);  // 计算数据结束位置

    const int64_t px = dst->ne[0];  // 获取输出张量的维度大小
    const int64_t py = dst->ne[1];  // 获取输出张量的维度大小
    const int64_t pa = px * py;  // 计算输出张量的面积

    float * dplane = (float *)dst->data;  // 将输出张量的数据转换为float类型指针

    const int ka = k0 * k1;  // 计算池化核的面积
    const int offset0 = -p0;  // 计算填充偏移量
    const int offset1 = -p1;  // 计算填充偏移量
    // 循环直到当前数据指针达到数据结束位置
    while (cdata < data_end) {
        // 遍历输出平面的每一行
        for (int oy = 0; oy < py; ++oy) {
            // 计算当前行在输出平面中的起始位置
            float * const drow = dplane + oy * px;
            // 遍历当前行的每一个元素
            for (int ox = 0; ox < px; ++ox) {
                // 计算当前元素在输出平面中的位置
                float * const out =  drow + ox;
                // 根据操作类型进行不同的处理
                switch (op) {
                    // 平均池化操作，初始化为0
                    case GGML_OP_POOL_AVG:     *out = 0;        break;
                    // 最大池化操作，初始化为负无穷
                    case GGML_OP_POOL_MAX:     *out = -FLT_MAX; break;
                    // 计数池化操作，抛出断言错误
                    case GGML_OP_POOL_COUNT: GGML_ASSERT(false); break;
                }

                // 计算当前元素在输入数据中的位置
                const int ix = offset0 + ox * s0;
                const int iy = offset1 + oy * s1;

                // 遍历卷积核的每一行
                for (int ky = 0; ky < k1; ++ky) {
                    // 如果当前行超出输入数据的范围，则跳过
                    if (iy + ky < 0 || iy + ky >= src->ne[1]) continue;
                    // 获取当前行的数据指针
                    const float * const srow = (const float *)(cdata + src->nb[1] * (iy + ky));
                    // 遍历卷积核的每一列
                    for (int kx = 0; kx < k0; ++kx) {
                        // 计算当前列在输入数据中的位置
                        int j = ix + kx;
                        // 如果当前列超出输入数据的范围，则跳过
                        if (j < 0 || j >= src->ne[0]) continue;
                        // 根据操作类型进行不同的处理
                        switch (op) {
                            // 平均池化操作，累加当前元素的值
                            case GGML_OP_POOL_AVG:                     *out += srow[j]; break;
                            // 最大池化操作，更新当前元素的值为最大值
                            case GGML_OP_POOL_MAX: if (srow[j] > *out) *out  = srow[j]; break;
                            // 计数池化操作，抛出断言错误
                            case GGML_OP_POOL_COUNT:                GGML_ASSERT(false); break;
                        }
                    }
                }
                // 根据操作类型进行不同的处理
                switch (op) {
                    // 平均池化操作，除以卷积核大小
                    case GGML_OP_POOL_AVG:           *out /= ka; break;
                    // 最大池化操作，不需要额外处理
                    case GGML_OP_POOL_MAX:                       break;
                    // 计数池化操作，抛出断言错误
                    case GGML_OP_POOL_COUNT: GGML_ASSERT(false); break;
                }
            }
        }

        // 更新当前数据指针到下一个数据块
        cdata  += src->nb[2];
        // 更新输出平面指针到下一行
        dplane += pa;
    }
// ggml_compute_forward_upscale 函数，用于对输入张量进行上采样操作
static void ggml_compute_forward_upscale_f32(
    const struct ggml_compute_params * params,  // 参数结构体指针，包含计算参数
    const struct ggml_tensor * src0,  // 输入张量指针，源数据
    struct ggml_tensor * dst) {  // 输出张量指针，目标数据

    // 如果参数类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 断言输入张量的第一个维度大小为 float 类型的大小
    GGML_ASSERT(src0->nb[0] == sizeof(float));

    // 获取当前线程和总线程数
    const int ith = params->ith;
    const int nth = params->nth;

    // 定义局部变量
    GGML_TENSOR_UNARY_OP_LOCALS

    // 获取上采样因子
    const int scale_factor = dst->op_params[0];

    // 循环遍历张量数据进行上采样操作
    for (int64_t i3 = 0; i3 < ne3; i3++) {
        const int64_t i03 = i3;
        for (int64_t i2 = ith; i2 < ne2; i2 += nth) {
            const int64_t i02 = i2;
            for (int64_t i1 = 0; i1 < ne1; i1++) {
                const int64_t i01 = i1 / scale_factor;
                for (int64_t i0 = 0; i0 < ne0; i0++) {
                    const int64_t i00 = i0 / scale_factor;

                    // 获取源数据和目标数据指针
                    const float * x = (float *)((char *) src0->data + i00*nb00 + i01*nb01 + i02*nb02 + i03*nb03);
                    float * y = (float *)((char *)  dst->data +  i0*nb0  +  i1*nb1  +  i2*nb2  +  i3*nb3);

                    // 进行数据拷贝
                    *y = *x;
                }
            }
        }
    }
}

// ggml_compute_forward_upscale 函数，根据输入张量类型调用相应的上采样函数
static void ggml_compute_forward_upscale(
    const struct ggml_compute_params * params,  // 参数结构体指针，包含计算参数
    const struct ggml_tensor * src0,  // 输入张量指针，源数据
    struct ggml_tensor * dst) {  // 输出张量指针，目标数据
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                ggml_compute_forward_upscale_f32(params, src0, dst);  // 调用 float 类型的上采样函数
            } break;
        default:
            {
                GGML_ASSERT(false);  // 断言错误，不支持的数据类型
            } break;
    }
}

// ggml_compute_forward_pad 函数，用于对输入张量进行填充操作
static void ggml_compute_forward_pad_f32(
    const struct ggml_compute_params * params,  // 参数结构体指针，包含计算参数
    const struct ggml_tensor * src0,  // 输入张量指针，源数据
    struct ggml_tensor * dst) {  // 输出张量指针，目标数据

    // 如果参数类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 断言输入张量的第一个维度大小为 float 类型的大小
    GGML_ASSERT(src0->nb[0] == sizeof(float));
    // 确保目标张量的第一个维度的大小为 float 类型的大小
    GGML_ASSERT( dst->nb[0] == sizeof(float));

    // 获取参数中的 ith 和 nth 值
    const int ith = params->ith;
    const int nth = params->nth;

    // 定义一些局部变量
    GGML_TENSOR_UNARY_OP_LOCALS

    // 将目标张量的数据指针转换为 float 类型指针
    float * dst_ptr = (float *) dst->data;

    // 待优化的部分

    // 循环遍历张量的各个维度，按照一定规则填充目标张量
    for (int64_t i2 = 0; i2 < ne2; ++i2) {
        for (int64_t i1 = ith; i1 < ne1; i1 += nth) {
            for (int64_t i0 = 0; i0 < ne0; ++i0) {
                for (int64_t i3 = 0; i3 < ne3; ++i3) {
                    // 计算目标张量中的索引
                    const int64_t dst_idx = i3*(ne0*ne1*ne2) + i2*(ne0*ne1) + i1*ne0 + i0;

                    // 计算源张量中的数据指针
                    const float * src_ptr = (const float *)((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01 + i0*nb00);

                    // 如果索引在各个维度的范围内，则将源数据复制到目标张量中，否则填充为 0
                    if (i0 < ne00 && i1 < ne01 && i2 < ne02 && i3 < ne03) {
                        dst_ptr[dst_idx] = *src_ptr;
                    } else {
                        dst_ptr[dst_idx] = 0;
                    }
                }
            }
        }
    }
// 计算前向排序操作，根据输入参数和张量进行排序
static void ggml_compute_forward_pad(
    const struct ggml_compute_params * params,
    const struct ggml_tensor * src0,
    struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的操作
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用针对 GGML_TYPE_F32 类型的前向填充操作
                ggml_compute_forward_pad_f32(params, src0, dst);
            } break;
        // 如果输入张量类型不在已知范围内
        default:
            {
                // 断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_argsort

// 针对 GGML_TYPE_F32 类型的前向排序操作
static void ggml_compute_forward_argsort_f32(
    const struct ggml_compute_params * params,
    const struct ggml_tensor * src0,
    struct ggml_tensor * dst) {
    // 如果参数类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 定义本地变量
    GGML_TENSOR_UNARY_OP_LOCALS

    // 断言 nb0 的大小为 float 类型的大小
    GGML_ASSERT(nb0 == sizeof(float));

    // 获取参数中的 ith 和 nth 值
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取输入张量 src0 的行数
    const int64_t nr = ggml_nrows(src0);

    // 获取排序顺序
    enum ggml_sort_order order = (enum ggml_sort_order) ggml_get_op_params_i32(dst, 0);

    // 遍历行数，根据 ith 和 nth 进行排序操作
    for (int64_t i = ith; i < nr; i += nth) {
        // 获取目标数据和源数据的指针
        int32_t * dst_data = (int32_t *)((char *) dst->data + i*nb1);
        const float * src_data = (float *)((char *) src0->data + i*nb01);

        // 初始化目标数据
        for (int64_t j = 0; j < ne0; j++) {
            dst_data[j] = j;
        }

        // 使用冒泡排序进行排序
        for (int64_t j = 0; j < ne0; j++) {
            for (int64_t k = j + 1; k < ne0; k++) {
                if ((order == GGML_SORT_ASC && src_data[dst_data[j]] > src_data[dst_data[k]]) ||
                    (order == GGML_SORT_DESC && src_data[dst_data[j]] < src_data[dst_data[k]])) {
                    int32_t tmp = dst_data[j];
                    dst_data[j] = dst_data[k];
                    dst_data[k] = tmp;
                }
            }
        }
    }
}

// 计算前向排序操作，根据输入参数和张量进行排序
static void ggml_compute_forward_argsort(
    const struct ggml_compute_params * params,
    const struct ggml_tensor * src0,
    struct ggml_tensor * dst) {
    # 根据 src0 的类型进行不同的操作
    switch (src0->type) {
        # 如果 src0 的类型是 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                # 调用 ggml_compute_forward_argsort_f32 函数进行计算
                ggml_compute_forward_argsort_f32(params, src0, dst);
            } break;
        # 如果 src0 的类型不是 GGML_TYPE_F32
        default:
            {
                # 断言，表示出现了不应该出现的情况
                GGML_ASSERT(false);
            } break;
    }
// 定义一个静态函数，用于计算前向闪存注意力机制，输入参数包括查询张量q、键张量k、值张量v、是否进行掩码操作、目标张量dst
static void ggml_compute_forward_flash_attn_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * q,
        const struct ggml_tensor * k,
        const struct ggml_tensor * v,
        const bool masked,
        struct ggml_tensor * dst) {
    // 记录函数开始时间
    int64_t t0 = ggml_perf_time_us();
    // 使用 UNUSED 宏避免编译器警告
    UNUSED(t0);

    // 定义查询张量的本地变量
    GGML_TENSOR_LOCALS(int64_t, neq, q,   ne)
    // 定义查询张量的本地变量
    GGML_TENSOR_LOCALS(size_t,  nbq, q,   nb)
    // 定义键张量的本地变量
    GGML_TENSOR_LOCALS(int64_t, nek, k,   ne)
    // 定义键张量的本地变量
    GGML_TENSOR_LOCALS(size_t,  nbk, k,   nb)
    // 定义值张量的本地变量
    GGML_TENSOR_LOCALS(int64_t, nev, v,   ne)
    // 定义值张量的本地变量
    GGML_TENSOR_LOCALS(size_t,  nbv, v,   nb)
    // 定义目标张量的本地变量
    GGML_TENSOR_LOCALS(int64_t, ne,  dst, ne)
    // 定义目标张量的本地变量
    GGML_TENSOR_LOCALS(size_t,  nb,  dst, nb)

    // 获取参数中的 ith 和 nth 值
    const int ith = params->ith;
    const int nth = params->nth;

    // 计算维度信息
    const int64_t D = neq0;
    const int64_t N = neq1;
    const int64_t P = nek1 - N;
    const int64_t M = P + N;

    // 计算 M 的上取整值
    const int Mup = ggml_up(M, GGML_SOFT_MAX_UNROLL);

    // 断言维度信息
    GGML_ASSERT(ne0 == D);
    GGML_ASSERT(ne1 == N);
    GGML_ASSERT(P >= 0);

    // 断言数据类型为 float
    GGML_ASSERT(nbq0 == sizeof(float));
    GGML_ASSERT(nbk0 == sizeof(float));
    GGML_ASSERT(nbv0 == sizeof(float));

    // 断言维度信息一致
    GGML_ASSERT(neq0 == D);
    GGML_ASSERT(nek0 == D);
    GGML_ASSERT(nev1 == D);

    GGML_ASSERT(neq1 == N);
    GGML_ASSERT(nek1 == N + P);
    GGML_ASSERT(nev1 == D);

    // 目标张量不能进行转置或置换
    GGML_ASSERT(nb0 == sizeof(float));
    GGML_ASSERT(nb0 <= nb1);
    GGML_ASSERT(nb1 <= nb2);
    GGML_ASSERT(nb2 <= nb3);

    // 如果任务类型为初始化，则直接返回
    if (params->type == GGML_TASK_INIT) {
        return;
    }

    // 如果任务类型为结束，则直接返回
    if (params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 通过查询张量的行数并行化计算，使用 ggml_vec_dot_f32 函数
    // 查询张量的总行数
    const int nr = neq1*neq2*neq3;

    // 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 缩放因子
    const float scale = 1.0f/sqrtf(D);
    // 打印参数 P、N、D、ir0、ir1、scale 的数值
    //printf("P=%d N=%d D=%d ir0=%d ir1=%d scale = %f\n", P, N, D, ir0, ir1, scale);

    // 遍历 ir0 到 ir1 之间的值
    for (int ir = ir0; ir < ir1; ++ir) {
        // 计算 q 索引
        const int iq3 = ir/(neq2*neq1);
        const int iq2 = (ir - iq3*neq2*neq1)/neq1;
        const int iq1 = (ir - iq3*neq2*neq1 - iq2*neq1);

        // 获取 S 数组的指针
        float * S = (float *) params->wdata + ith*(Mup + CACHE_LINE_SIZE_F32);

        // 将 S 数组中 M 到 Mup 之间的值设为负无穷
        for (int i = M; i < Mup; ++i) {
            S[i] = -INFINITY;
        }

        // 计算 masked_begin 的值，根据 masked 是否为真
        const int64_t masked_begin = masked ? (P + iq1 + 1) : M;
        // 遍历 ic 从 0 到 masked_begin 之间的值
        for (int64_t ic = 0; ic < masked_begin; ++ic) {
            // 计算 k 索引
            const int ik3 = iq3;
            const int ik2 = iq2 % nek2;
            const int ik1 = ic;

            // 计算 S 索引
            const int i1 = ik1;

            // 调用 ggml_vec_dot_f32 函数计算向量点积
            ggml_vec_dot_f32(neq0,
                    S + i1,
                    (float *) ((char *) k->data + (ik1*nbk1 + ik2*nbk2 + ik3*nbk3)),
                    (float *) ((char *) q->data + (iq1*nbq1 + iq2*nbq2 + iq3*nbq3)));
        }

        // 对 masked_begin 之前的 S 数组进行缩放
        ggml_vec_scale_f32(masked_begin, S, scale);

        // 将 masked_begin 之后的 S 数组值设为负无穷
        for (int64_t i = masked_begin; i < M; i++) {
            S[i] = -INFINITY;
        }

        // softmax 操作
        // 从 max 和循环中排除已知的 -INF S[..] 值
        // 不要忘记将它们的 SW 值设为零
        {
            // 初始化 max 为负无穷
            float max = -INFINITY;
            // 计算 S 数组中 masked_begin 之前的最大值
            ggml_vec_max_f32(masked_begin, &max, S);

            // 初始化 sum 为 0.0
            ggml_float sum = 0.0;
#ifdef GGML_SOFT_MAX_ACCELERATE
                // 如果定义了 GGML_SOFT_MAX_ACCELERATE，则对最大值取负数
                max = -max;
                // 将最大值加到 S 中的每个元素上
                vDSP_vsadd(S, 1, &max, S, 1, Mup);
                // 对 S 中的每个元素进行指数运算
                vvexpf(S, S, &Mup);
                // 计算 S 中所有元素的和
                ggml_vec_sum_f32(Mup, &sum, S);
#else
                // 定义一个 uint16_t 数组 scvt，用于存储转换后的数据
                uint16_t   scvt[GGML_SOFT_MAX_UNROLL]; UNUSED(scvt);
                // 定义一个浮点数数组 sump，初始化为 0.0
                ggml_float sump[GGML_SOFT_MAX_UNROLL] = { 0.0 };

                // 循环处理 S 中的元素
                for (int i = 0; i < Mup; i += GGML_SOFT_MAX_UNROLL) {
                    // 如果超出 masked_begin，则跳出循环
                    if (i >= masked_begin) {
                        break;
                    }
                    // 获取当前位置的指针
                    float * SS = S + i;

                    // 循环处理 GGML_SOFT_MAX_UNROLL 个元素
                    for (int j = 0; j < GGML_SOFT_MAX_UNROLL; ++j) {
                        // 如果超出 masked_begin，则跳出循环
                        if (i + j >= masked_begin) {
                            break;
                        } else if (SS[j] == -INFINITY) {
                            // 如果当前元素为负无穷，则将其设为 0.0
                            SS[j] = 0.0f;
                        } else {
#ifndef GGML_FLASH_ATTN_EXP_FP16
                            // 如果未定义 GGML_FLASH_ATTN_EXP_FP16，则计算指数值
                            const float val = expf(SS[j] - max);
#else
                            // 如果定义了 GGML_FLASH_ATTN_EXP_FP16，则进行 FP16 转换
                            ggml_fp16_t s = GGML_FP32_TO_FP16(SS[j] - max);
                            // 将转换后的数据存入 scvt 数组
                            memcpy(&scvt[j], &s, sizeof(uint16_t));
                            // 从表中获取 FP16 转换后的值
                            const float val = GGML_FP16_TO_FP32(ggml_table_exp_f16[scvt[j]]);
#endif
                            // 将计算得到的值加到 sump 数组中
                            sump[j] += (ggml_float)val;
                            // 将计算得到的值存入 SS 中
                            SS[j] = val;
                        }
                    }
                }

                // 将 sump 数组中的值加到 sum 中
                for (int i = 0; i < GGML_SOFT_MAX_UNROLL; i++) {
                    sum += sump[i];
                }
#endif
            }

            // 断言 sum 大于 0.0
            assert(sum > 0.0);

            // 计算 sum 的倒数
            sum = 1.0/sum;
            // 对 S 中的元素进行缩放
            ggml_vec_scale_f32(masked_begin, S, sum);

#ifndef NDEBUG
            // 调试模式下，检查 S 中的元素是否为 NaN 或无穷大
            for (int i = 0; i < masked_begin; ++i) {
                assert(!isnan(S[i]));
                assert(!isinf(S[i]));
            }
#endif
        }

        for (int64_t ic = 0; ic < nev1; ++ic) {
            // dst indices
            // 计算目标张量的索引
            const int i1 = iq1;
            const int i2 = iq2;
            const int i3 = iq3;

            // v indices
            // 计算 v 张量的索引
            const int iv2 = iq2 % nev2;
            const int iv3 = iq3;

            // 调用 ggml_vec_dot_f32 函数计算点积
            ggml_vec_dot_f32(masked_begin,
                    (float *) ((char *) dst->data + (ic*nb0 + i1*nb1  + i2*nb2   + i3*nb3)),
                    (float *) ((char *) v->data   + (         ic*nbv1 + iv2*nbv2 + iv3*nbv3)),
                    S);
        }
    }
}

static void ggml_compute_forward_flash_attn_f16(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * q,
        const struct ggml_tensor * k,
        const struct ggml_tensor * v,
        const bool masked,
        struct ggml_tensor * dst) {
    int64_t t0 = ggml_perf_time_us();
    UNUSED(t0);

    // 定义并初始化一系列局部变量
    GGML_TENSOR_LOCALS(int64_t, neq, q,   ne)
    GGML_TENSOR_LOCALS(size_t,  nbq, q,   nb)
    GGML_TENSOR_LOCALS(int64_t, nek, k,   ne)
    GGML_TENSOR_LOCALS(size_t,  nbk, k,   nb)
    GGML_TENSOR_LOCALS(int64_t, nev, v,   ne)
    GGML_TENSOR_LOCALS(size_t,  nbv, v,   nb)
    GGML_TENSOR_LOCALS(int64_t, ne,  dst, ne)
    GGML_TENSOR_LOCALS(size_t,  nb,  dst, nb)

    // 获取参数中的 ith 和 nth 值
    const int ith = params->ith;
    const int nth = params->nth;

    // 计算 D、N、P、M 和 Mup 的值
    const int64_t D = neq0;
    const int64_t N = neq1;
    const int64_t P = nek1 - N;
    const int64_t M = P + N;

    const int Mup = ggml_up(M, GGML_SOFT_MAX_UNROLL);

    // 断言一系列条件
    GGML_ASSERT(ne0 == D);
    GGML_ASSERT(ne1 == N);
    GGML_ASSERT(P >= 0);

    GGML_ASSERT(nbq0 == sizeof(ggml_fp16_t));
    GGML_ASSERT(nbk0 == sizeof(ggml_fp16_t));
    GGML_ASSERT(nbv0 == sizeof(ggml_fp16_t));

    GGML_ASSERT(neq0 == D);
    GGML_ASSERT(nek0 == D);
    GGML_ASSERT(nev1 == D);

    GGML_ASSERT(neq1 == N);
    GGML_ASSERT(nek1 == N + P);
    GGML_ASSERT(nev1 == D);

    // 断言 dst 不能被转置或排列
    GGML_ASSERT(nb0 == sizeof(float));
    // 确保 nb0 小于等于 nb1
    GGML_ASSERT(nb0 <= nb1);
    // 确保 nb1 小于等于 nb2
    GGML_ASSERT(nb1 <= nb2);
    // 确保 nb2 小于等于 nb3
    GGML_ASSERT(nb2 <= nb3);

    // 如果参数类型为 GGML_TASK_INIT，则直接返回
    if (params->type == GGML_TASK_INIT) {
        return;
    }

    // 如果参数类型为 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 使用 ggml_vec_dot_f32 在 q 行上并行化

    // q 中的总行数
    const int nr = neq1 * neq2 * neq3;

    // 每个线程的行数
    const int dr = (nr + nth - 1) / nth;

    // 该线程的行范围
    const int ir0 = dr * ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 缩放因子
    const float scale = 1.0f / sqrtf(D);

    //printf("P=%d N=%d D=%d ir0=%d ir1=%d scale = %f\n", P, N, D, ir0, ir1, scale);
#ifdef GGML_SOFT_MAX_ACCELERATE
                // 如果定义了 GGML_SOFT_MAX_ACCELERATE，则执行以下代码块
                max = -max;
                // 取最大值的相反数
                vDSP_vsadd(S, 1, &max, S, 1, Mup);
                // 将数组 S 中的每个元素加上 max
                vvexpf(S, S, &Mup);
                // 对数组 S 中的每个元素进行指数运算
                ggml_vec_sum_f32(Mup, &sum, S);
                // 计算数组 Mup 的和，结果存储在 sum 中
#else
                // 如果未定义 GGML_SOFT_MAX_ACCELERATE，则执行以下代码块
                uint16_t   scvt[GGML_SOFT_MAX_UNROLL];
                // 定义 uint16_t 类型数组 scvt
                ggml_float sump[GGML_SOFT_MAX_UNROLL] = { 0.0 };
                // 定义 ggml_float 类型数组 sump，并初始化为 0.0

                for (int i = 0; i < Mup; i += GGML_SOFT_MAX_UNROLL) {
                    // 循环遍历 Mup，每次增加 GGML_SOFT_MAX_UNROLL
                    float * SS = S + i;
                    // 定义指针 SS 指向数组 S 的第 i 个元素

                    for (int j = 0; j < GGML_SOFT_MAX_UNROLL; ++j) {
                        // 循环遍历 GGML_SOFT_MAX_UNROLL 次
                        if (SS[j] == -INFINITY) {
                            // 如果 SS[j] 的值为 -INFINITY
                            SS[j] = 0.0f;
                            // 将 SS[j] 的值设为 0.0f
                        } else {
                            ggml_fp16_t s = GGML_FP32_TO_FP16(SS[j] - max);
                            // 将 SS[j] 减去 max 转换为 ggml_fp16_t 类型
                            memcpy(&scvt[j], &s, sizeof(uint16_t));
                            // 将 s 的值拷贝到 scvt[j] 中
                            const float val = GGML_FP16_TO_FP32(ggml_table_exp_f16[scvt[j]]);
                            // 通过 ggml_table_exp_f16 查找对应值，并转换为 float 类型
                            sump[j] += (ggml_float)val;
                            // 将 val 转换为 ggml_float 类型并加到 sump[j] 中
                            SS[j] = val;
                            // 将 SS[j] 的值设为 val
                        }
                    }
                }

                for (int i = 0; i < GGML_SOFT_MAX_UNROLL; i++) {
                    // 循环遍历 GGML_SOFT_MAX_UNROLL 次
                    sum += sump[i];
                    // 将 sump[i] 的值加到 sum 中
                }
#endif
            }

            assert(sum > 0.0);
            // 断言 sum 大于 0.0

            sum = 1.0/sum;
            // 计算 sum 的倒数

            ggml_vec_scale_f32(M, S, sum);
            // 对数组 S 中的每个元素乘以 sum

#ifndef NDEBUG
            // 如果未定义 NDEBUG，则执行以下代码块
            for (int i = 0; i < M; ++i) {
                // 循环遍历 M 次
                assert(!isnan(S[i]));
                // 断言 S[i] 不是 NaN
                assert(!isinf(S[i]));
                // 断言 S[i] 不是无穷大
            }
#endif
        }

        // 将指针 S16 指向 params->wdata 中的特定位置
        ggml_fp16_t * S16 = (ggml_fp16_t *) ((float *) params->wdata + ith*(2*Mup + CACHE_LINE_SIZE_F32) + Mup);

        // 将浮点数数组 S 转换为半精度浮点数数组 S16
        for (int64_t i = 0; i < M; i++) {
            S16[i] = GGML_FP32_TO_FP16(S[i]);
        }

        // 如果 GGML_VEC_DOT_UNROLL 为 1 或 nev1 不能被 GGML_VEC_DOT_UNROLL 整除，则执行以下代码块
        if (GGML_VEC_DOT_UNROLL == 1 || (nev1 % GGML_VEC_DOT_UNROLL != 0)) {
            for (int64_t ic = 0; ic < nev1; ++ic) {
                // 计算目标数据的索引
                const int i1 = iq1;
                const int i2 = iq2;
                const int i3 = iq3;

                // 计算 v 数据的索引
                const int iv2 = iq2 % nev2;
                const int iv3 = iq3;

                // 调用 ggml_vec_dot_f16 函数计算点积
                ggml_vec_dot_f16(nev0,
                        (float *)       ((char *) dst->data + (ic*nb0 + i1*nb1  + i2*nb2   + i3*nb3)),
                        (ggml_fp16_t *) ((char *) v->data   + (         ic*nbv1 + iv2*nbv2 + iv3*nbv3)),
                        S16);
            }
        } else {
            // 否则执行以下代码块
            for (int64_t ic = 0; ic < nev1; ic += GGML_VEC_DOT_UNROLL) {
                // 计算目标数据的索引
                const int i1 = iq1;
                const int i2 = iq2;
                const int i3 = iq3;

                // 计算 v 数据的索引
                const int iv2 = iq2 % nev2;
                const int iv3 = iq3;

                // 调用 ggml_vec_dot_f16_unroll 函数计算点积
                ggml_vec_dot_f16_unroll(nev0, nbv1,
                        (float *) ((char *) dst->data + (ic*nb0 + i1*nb1  + i2*nb2   + i3*nb3)),
                        ((char *)             v->data + (         ic*nbv1 + iv2*nbv2 + iv3*nbv3)),
                        S16);
            }
        }
    }
}

// 计算前向传播的闪电注意力机制
static void ggml_compute_forward_flash_attn(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * q,
        const struct ggml_tensor * k,
        const struct ggml_tensor * v,
        const bool masked,
        struct ggml_tensor * dst) {
    # 根据查询参数的类型进行不同的操作
    switch (q->type) {
        # 如果查询参数的类型是 GGML_TYPE_F16
        case GGML_TYPE_F16:
            {
                # 调用针对 F16 类型的前向闪存注意力计算函数
                ggml_compute_forward_flash_attn_f16(params, q, k, v, masked, dst);
            } break;
        # 如果查询参数的类型是 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                # 调用针对 F32 类型的前向闪存注意力计算函数
                ggml_compute_forward_flash_attn_f32(params, q, k, v, masked, dst);
            } break;
        # 如果查询参数的类型不是 F16 或 F32
        default:
            {
                # 断言，抛出错误
                GGML_ASSERT(false);
            } break;
    }
// 计算前向传播的 FlashFF 算法，处理 F16 数据类型
static void ggml_compute_forward_flash_ff_f16(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * a,  // F16 输入张量 a
        const struct ggml_tensor * b0, // F16 全连接层权重张量 b0
        const struct ggml_tensor * b1, // F32 全连接层偏置张量 b1
        const struct ggml_tensor * c0, // F16 投影层权重张量 c0
        const struct ggml_tensor * c1, // F32 投影层偏置张量 c1
        struct ggml_tensor * dst) {    // 输出张量 dst
    int64_t t0 = ggml_perf_time_us(); // 记录当前时间
    UNUSED(t0); // 防止编译器警告

    // 定义宏，简化获取张量维度大小的操作
    GGML_TENSOR_LOCALS(int64_t, nea,  a,   ne)
    GGML_TENSOR_LOCALS(size_t,  nba,  a,   nb)
    GGML_TENSOR_LOCALS(int64_t, neb0, b0,  ne)
    GGML_TENSOR_LOCALS(size_t,  nbb0, b0,  nb)
    GGML_TENSOR_LOCALS(int64_t, neb1, b1,  ne)
    GGML_TENSOR_LOCALS(size_t,  nbb1, b1,  nb)
    GGML_TENSOR_LOCALS(int64_t, nec0, c0,  ne)
    GGML_TENSOR_LOCALS(size_t,  nbc0, c0,  nb)
    GGML_TENSOR_LOCALS(int64_t, nec1, c1,  ne)
    GGML_TENSOR_LOCALS(size_t,  nbc1, c1,  nb)
    GGML_TENSOR_LOCALS(int64_t, ne,   dst, ne)
    GGML_TENSOR_LOCALS(size_t,  nb,   dst, nb)

    // 获取参数中的 ith 和 nth
    const int ith = params->ith;
    const int nth = params->nth;

    const int64_t D = nea0; // 获取输入张量 a 的第一个维度大小
    //const int64_t N = nea1;
    const int64_t M = neb01; // 获取全连接层权重张量 b0 的第二个维度大小

    // 断言各张量的维度大小是否匹配
    GGML_ASSERT(ne0 == nea0);
    GGML_ASSERT(ne1 == nea1);
    GGML_ASSERT(ne2 == nea2);

    GGML_ASSERT(nba0  == sizeof(ggml_fp16_t));
    GGML_ASSERT(nbb00 == sizeof(ggml_fp16_t));
    GGML_ASSERT(nbb10 == sizeof(float));
    GGML_ASSERT(nbc00 == sizeof(ggml_fp16_t));
    GGML_ASSERT(nbc10 == sizeof(float));

    GGML_ASSERT(neb00 == D);
    GGML_ASSERT(neb01 == M);
    GGML_ASSERT(neb10 == M);
    GGML_ASSERT(neb11 == 1);

    GGML_ASSERT(nec00 == M);
    GGML_ASSERT(nec01 == D);
    GGML_ASSERT(nec10 == D);
    GGML_ASSERT(nec11 == 1);

    // 断言输出张量 dst 不能进行转置或排列
    GGML_ASSERT(nb0 == sizeof(float));
    GGML_ASSERT(nb0 <= nb1);
    GGML_ASSERT(nb1 <= nb2);
    GGML_ASSERT(nb2 <= nb3);

    // 如果参数中的任务类型为 GGML_TASK_INIT，则直接返回
    if (params->type == GGML_TASK_INIT) {
        return;
    }
}
    // 如果任务类型为 GGML_TASK_FINALIZE，则直接返回，不执行后续操作
    if (params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 使用 ggml_vec_dot_f32 函数并行计算每一行

    // 计算矩阵 a 的总行数
    const int nr = nea1 * nea2 * nea3;

    // 计算每个线程处理的行数
    const int dr = (nr + nth - 1) / nth;

    // 计算当前线程处理的行范围
    const int ir0 = dr * ith;
    const int ir1 = MIN(ir0 + dr, nr);
    // 遍历 a 的索引范围
    for (int ir = ir0; ir < ir1; ++ir) {
        // a 的三维索引计算
        const int ia3 = ir/(nea2*nea1);
        const int ia2 = (ir - ia3*nea2*nea1)/nea1;
        const int ia1 = (ir - ia3*nea2*nea1 - ia2*nea1);

        // 获取 S 指针
        float * S = (float *) params->wdata + ith*(2*M + CACHE_LINE_SIZE_F32);

        // 遍历 b0 的索引范围
        for (int64_t ic = 0; ic < neb01; ++ic) {
            // b0 的三维索引计算
            const int ib03 = ia3;
            const int ib02 = ia2;
            const int ib01 = ic;

            // S 的索引计算
            const int i1 = ib01;

            // 调用 ggml_vec_dot_f16 函数
            ggml_vec_dot_f16(nea0,
                    S + i1,
                    (ggml_fp16_t *) ((char *) b0->data + (ib01*nbb01 + ib02*nbb02 + ib03*nbb03)),
                    (ggml_fp16_t *) ((char *)  a->data + ( ia1*nba1  +  ia2*nba2  +  ia3*nba3));
        }

        // 调用 ggml_vec_add_f32 函数
        ggml_vec_add_f32(neb01, S, S, (float *) b1->data);
        //ggml_vec_gelu_f32(neb01, S, S);

        // 获取 S16 指针
        ggml_fp16_t * S16 = (ggml_fp16_t *) ((float *) params->wdata + ith*(2*M + CACHE_LINE_SIZE_F32) + M);

        // 遍历 M
        for (int64_t i = 0; i < M; i++) {
            // 将 S 中的数据转换为 ggml_fp16_t 类型
            S16[i] = GGML_FP32_TO_FP16(S[i]);
        }

        // 调用 ggml_vec_gelu_f16 函数
        ggml_vec_gelu_f16(neb01, S16, S16);

        {
            // dst 的索引计算
            const int i1 = ia1;
            const int i2 = ia2;
            const int i3 = ia3;

            // 遍历 c 的索引范围
            for (int64_t ic = 0; ic < nec01; ++ic) {
                // 调用 ggml_vec_dot_f16 函数
                ggml_vec_dot_f16(neb01,
                        (float *)       ((char *) dst->data + (ic*nb0 + i1*nb1   + i2*nb2   + i3*nb3)),
                        (ggml_fp16_t *) ((char *) c0->data  + (         ic*nbc01 + i2*nbc02 + i3*nbc03)),
                        S16);
            }

            // 调用 ggml_vec_add_f32 函数
            ggml_vec_add_f32(nec01,
                    (float *) ((char *) dst->data + (i1*nb1 + i2*nb2 + i3*nb3)),
                    (float *) ((char *) dst->data + (i1*nb1 + i2*nb2 + i3*nb3)),
                    (float *) c1->data);
        }
    }
// 计算前向传播的 Flash FF 操作，根据输入参数和张量进行计算，将结果存储在目标张量中
static void ggml_compute_forward_flash_ff(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * a,
        const struct ggml_tensor * b0,
        const struct ggml_tensor * b1,
        const struct ggml_tensor * c0,
        const struct ggml_tensor * c1,
        struct ggml_tensor * dst) {
    // 根据 b0 张量的类型进行不同的操作
    switch (b0->type) {
        // 如果 b0 张量的类型为 GGML_TYPE_F16
        case GGML_TYPE_F16:
            {
                // 调用相应的 f16 版本的前向传播 Flash FF 函数
                ggml_compute_forward_flash_ff_f16(params, a, b0, b1, c0, c1, dst);
            } break;
        // 如果 b0 张量的类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 输出错误信息，表示该部分功能尚未实现
                GGML_ASSERT(false); // TODO
            } break;
        // 其他情况
        default:
            {
                // 输出错误信息，表示不支持的类型
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_flash_attn_back

// 计算前向传播的 Flash Attention Back 操作，根据输入参数和张量进行计算，将结果存储在目标张量中
static void ggml_compute_forward_flash_attn_back_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * q,
        const struct ggml_tensor * k,
        const struct ggml_tensor * v,
        const struct ggml_tensor * d,
        const bool masked,
              struct ggml_tensor * dst) {
    // 记录当前时间
    int64_t t0 = ggml_perf_time_us();
    UNUSED(t0);

    // 定义并初始化一系列本地变量
    GGML_TENSOR_LOCALS(int64_t, neq, q,   ne)
    GGML_TENSOR_LOCALS(size_t,  nbq, q,   nb)
    GGML_TENSOR_LOCALS(int64_t, nek, k,   ne)
    GGML_TENSOR_LOCALS(size_t,  nbk, k,   nb)
    GGML_TENSOR_LOCALS(int64_t, nev, v,   ne)
    GGML_TENSOR_LOCALS(size_t,  nbv, v,   nb)
    GGML_TENSOR_LOCALS(int64_t, ned, d,   ne)
    GGML_TENSOR_LOCALS(size_t,  nbd, d,   nb)
    GGML_TENSOR_LOCALS(int64_t, ne,  dst, ne)
    GGML_TENSOR_LOCALS(size_t,  nb,  dst, nb)

    // 获取参数中的 ith 和 nth 值
    const int ith = params->ith;
    const int nth = params->nth;

    // 计算一系列维度相关的值
    const int64_t D = neq0;
    const int64_t N = neq1;
    const int64_t P = nek1 - N;
    const int64_t M = P + N;

    // 计算 Mup 和 mxDM 的值
    const int Mup  = ggml_up(M, GGML_SOFT_MAX_UNROLL);
    const int mxDM = MAX(D, Mup);

    // 输出断言信息，确保一些条件成立
    // GGML_ASSERT(ne0 == D);
    // GGML_ASSERT(ne1 == N);
    GGML_ASSERT(P >= 0);

    // 输出断言信息，确保 nbq0 的值为 sizeof(float)
    GGML_ASSERT(nbq0 == sizeof(float));
}
    // 确保 nbk0 等于 float 的大小
    GGML_ASSERT(nbk0 == sizeof(float));
    // 确保 nbv0 等于 float 的大小
    GGML_ASSERT(nbv0 == sizeof(float));

    // 确保 neq0 等于 D
    GGML_ASSERT(neq0 == D);
    // 确保 nek0 等于 D
    GGML_ASSERT(nek0 == D);
    // 确保 nev1 等于 D
    GGML_ASSERT(nev1 == D);
    // 确保 ned0 等于 D
    GGML_ASSERT(ned0 == D);

    // 确保 neq1 等于 N
    GGML_ASSERT(neq1 == N);
    // 确保 nek1 等于 N + P
    GGML_ASSERT(nek1 == N + P);
    // 确保 nev1 等于 D
    GGML_ASSERT(nev1 == D);
    // 确保 ned1 等于 N
    GGML_ASSERT(ned1 == N);

    // dst 不能被转置或置换
    GGML_ASSERT(nb0 == sizeof(float));
    GGML_ASSERT(nb0 <= nb1);
    GGML_ASSERT(nb1 <= nb2);
    GGML_ASSERT(nb2 <= nb3);

    // 如果任务类型为初始化
    if (params->type == GGML_TASK_INIT) {
        // 如果是第一个线程
        if (ith == 0) {
            // 将 dst->data 初始化为 0
            memset(dst->data, 0, nb0*ne0*ne1*ne2*ne3);
        }
        // 返回
        return;
    }

    // 如果任务类型为结束
    if (params->type == GGML_TASK_FINALIZE) {
        // 返回
        return;
    }

    // 计算 q 和 k 的元素个数
    const int64_t elem_q = ggml_nelements(q);
    const int64_t elem_k = ggml_nelements(k);

    // 获取结果类型
    enum ggml_type result_type = dst->type;
    // 确保结果类型的块大小为 1
    GGML_ASSERT(ggml_blck_size(result_type) == 1);
    // 获取结果类型的大小
    const size_t tsize = ggml_type_size(result_type);

    // 计算偏移量
    const size_t offs_q = 0;
    const size_t offs_k = offs_q + GGML_PAD(elem_q * tsize, GGML_MEM_ALIGN);
    const size_t offs_v = offs_k + GGML_PAD(elem_k * tsize, GGML_MEM_ALIGN);

    // 获取梯度的指针
    void * grad_q = (char *) dst->data;
    void * grad_k = (char *) dst->data + offs_k;
    void * grad_v = (char *) dst->data + offs_v;

    // 计算各维度的大小
    const size_t nbgq1 = nb0*neq0;
    const size_t nbgq2 = nb0*neq0*neq1;
    const size_t nbgq3 = nb0*neq0*neq1*neq2;

    const size_t nbgk1 = nb0*nek0;
    const size_t nbgk2 = nb0*nek0*nek1;
    const size_t nbgk3 = nb0*nek0*nek1*neq2;

    const size_t nbgv1 = nb0*nev0;
    const size_t nbgv2 = nb0*nev0*nev1;
    const size_t nbgv3 = nb0*nev0*nev1*neq2;

    // 通过 k 行并行化使用 ggml_vec_dot_f32

    // k 中的总行数
    const int nr = nek2*nek3;

    // 每个线程的行数
    const int dr = (nr + nth - 1)/nth;

    // 该线程的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 缩放因子
    const float scale = 1.0f/sqrtf(D);
    // 打印变量 P、N、D、ir0、ir1、scale 的值
    //printf("P=%d N=%d D=%d ir0=%d ir1=%d scale = %f\n", P, N, D, ir0, ir1, scale);

    // 计算 k2（和 v2）在 q2 中重复出现的次数
    int nrep = neq2/nek2;
#ifdef GGML_SOFT_MAX_ACCELERATE
                        // 如果定义了 GGML_SOFT_MAX_ACCELERATE，则执行以下代码块
                        max = -max;
                        // 取最大值的相反数
                        vDSP_vsadd(SM, 1, &max, SM, 1, Mup);
                        // 使用 vDSP 函数将 SM 中的每个元素加上 max
                        vvexpf(SM, SM, &Mup);
                        // 使用 Accelerate 框架的 vvexpf 函数对 SM 中的每个元素进行指数运算
                        ggml_vec_sum_f32(Mup, &sum, SM);
                        // 调用自定义函数 ggml_vec_sum_f32 对 Mup 中的元素求和，结果存储在 sum 中
#else
                        // 如果未定义 GGML_SOFT_MAX_ACCELERATE，则执行以下代码块
                        uint16_t   scvt[GGML_SOFT_MAX_UNROLL]; UNUSED(scvt);
                        // 定义 uint16_t 类型数组 scvt，长度为 GGML_SOFT_MAX_UNROLL，但未使用
                        ggml_float sump[GGML_SOFT_MAX_UNROLL] = { 0.0 };
                        // 定义 ggml_float 类型数组 sump，长度为 GGML_SOFT_MAX_UNROLL，初始化为 0.0

                        for (int i = 0; i < Mup; i += GGML_SOFT_MAX_UNROLL) {
                            // 循环遍历 Mup，每次增加 GGML_SOFT_MAX_UNROLL
                            if (i >= masked_begin) {
                                // 如果 i 大于等于 masked_begin，则跳出循环
                                break;
                            }
                            float * SR =  S + i;
                            // 定义指向 S + i 的 float 指针 SR
                            float * SW = SM + i;
                            // 定义指向 SM + i 的 float 指针 SW

                            for (int j = 0; j < GGML_SOFT_MAX_UNROLL; ++j) {
                                // 循环遍历 GGML_SOFT_MAX_UNROLL 次
                                if (i + j >= masked_begin) {
                                    // 如果 i + j 大于等于 masked_begin，则跳出内层循环
                                    break;
                                } else if (SR[j] == -INFINITY) {
                                    // 如果 SR[j] 等于负无穷，则将 SW[j] 置为 0.0f
                                    SW[j] = 0.0f;
                                } else {
#ifndef GGML_FLASH_ATTN_EXP_FP16
                                    // 如果未定义 GGML_FLASH_ATTN_EXP_FP16，则执行以下代码块
                                    const float val = expf(SR[j] - max);
                                    // 计算 exp(SR[j] - max) 并赋值给 val
#else
                                    // 如果定义了 GGML_FLASH_ATTN_EXP_FP16，则执行以下代码块
                                    ggml_fp16_t s = GGML_FP32_TO_FP16(SR[j] - max);
                                    // 将 SR[j] - max 转换为 ggml_fp16_t 类型并赋值给 s
                                    memcpy(&scvt[j], &s, sizeof(uint16_t));
                                    // 将 s 的内容复制到 scvt[j] 中
                                    const float val = GGML_FP16_TO_FP32(ggml_table_exp_f16[scvt[j]]);
                                    // 从 ggml_table_exp_f16 中取出 scvt[j] 对应的值，并转换为 float 赋值给 val
#endif
                                    sump[j] += (ggml_float)val;
                                    // 将 val 转换为 ggml_float 类型并加到 sump[j] 中
                                    SW[j] = val;
                                    // 将 val 赋值给 SW[j]
                                }
                            }
                        }

                        for (int i = 0; i < GGML_SOFT_MAX_UNROLL; i++) {
                            // 循环遍历 GGML_SOFT_MAX_UNROLL 次
                            sum += sump[i];
                            // 将 sump[i] 加到 sum 中
                        }
    }
}
// 计算前向闪存注意力反向传播，根据参数和输入张量计算输出张量
static void ggml_compute_forward_flash_attn_back(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * q,
        const struct ggml_tensor * k,
        const struct ggml_tensor * v,
        const struct ggml_tensor * d,
        const bool masked,
        struct ggml_tensor * dst) {
    // 根据输入张量的数据类型进行不同的处理
    switch (q->type) {
        // 如果数据类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用计算前向闪存注意力反向传播的函数，传入参数和输入张量，计算结果存储在输出张量中
                ggml_compute_forward_flash_attn_back_f32(params, q, k, v, d, masked, dst);
            } break;
        // 如果数据类型不是 GGML_TYPE_F32
        default:
            {
                // 断言失败，抛出异常
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_win_part

// 计算前向窗口部分，根据参数和输入张量计算输出张量
static void ggml_compute_forward_win_part_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 如果参数类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 定义局部变量 ne0 和 ne，分别表示 src0 和 dst 的元素数量
    GGML_TENSOR_LOCALS(int64_t, ne0, src0, ne)
    GGML_TENSOR_LOCALS(int64_t, ne,  dst,  ne)

    // 从输出张量的操作参数中获取 nep0、nep1 和 w 的值
    const int32_t nep0 = ((const int32_t *)(dst->op_params))[0];
    const int32_t nep1 = ((const int32_t *)(dst->op_params))[1];
    const int32_t w    = ((const int32_t *)(dst->op_params))[2];

    // 断言 ne00 等于 ne0，ne3 等于 nep0*nep1
    assert(ne00 == ne0);
    assert(ne3  == nep0*nep1);

    // TODO: optimize / multi-thread
}
    // 遍历第一个维度上的索引值
    for (int py = 0; py < nep1; ++py) {
        // 遍历第二个维度上的索引值
        for (int px = 0; px < nep0; ++px) {
            // 计算三维索引值
            const int64_t i3 = py*nep0 + px;
            // 遍历第三个维度上的索引值
            for (int64_t i2 = 0; i2 < ne2; ++i2) {
                // 遍历第四个维度上的索引值
                for (int64_t i1 = 0; i1 < ne1; ++i1) {
                    // 遍历第五个维度上的索引值
                    for (int64_t i0 = 0; i0 < ne0; ++i0) {
                        // 计算三维索引值
                        const int64_t i02 = py*w + i2;
                        // 计算二维索引值
                        const int64_t i01 = px*w + i1;
                        // 计算一维索引值
                        const int64_t i00 = i0;

                        // 计算源数据数组中的索引值
                        const int64_t i = i3*ne2*ne1*ne0 + i2*ne1*ne0 + i1*ne0 + i0;
                        // 计算目标数据数组中的索引值
                        const int64_t j = i02*ne01*ne00 + i01*ne00 + i00;

                        // 检查索引是否越界，如果越界则将目标数据数组中的值设为0.0
                        if (py*w + i2 >= ne02 || px*w + i1 >= ne01) {
                            ((float *) dst->data)[i] = 0.0f;
                        } else {
                            // 将源数据数组中的值复制到目标数据数组中
                            ((float *) dst->data)[i] = ((float *) src0->data)[j];
                        }
                    }
                }
            }
        }
    }
// 计算前向窗口非分块操作，根据输入参数和数据源计算结果存储到目标数据结构中
static void ggml_compute_forward_win_part(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据数据源的类型进行不同的操作
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                // 调用相应的计算前向窗口非分块操作函数
                ggml_compute_forward_win_part_f32(params, src0, dst);
            } break;
        default:
            {
                // 断言，如果不是指定的类型则报错
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_win_unpart

// 计算前向窗口非分块操作，针对浮点数类型的数据源
static void ggml_compute_forward_win_unpart_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 如果参数类型为初始化或者结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 定义局部变量 ne0 和 ne，分别表示 src0 和 dst 的维度
    GGML_TENSOR_LOCALS(int64_t, ne0, src0, ne)
    GGML_TENSOR_LOCALS(int64_t, ne,  dst,  ne)

    // 从目标数据结构的操作参数中获取一个整数值
    const int32_t w = ((const int32_t *)(dst->op_params))[0];

    // 计算填充值 px
    const int px = (w - ne1%w)%w;
    //const int py = (w - ne2%w)%w;

    // 计算 npx
    const int npx = (px + ne1)/w;
    //const int npy = (py + ne2)/w;

    // 断言，确保 ne0 等于 ne00
    assert(ne0 == ne00);

    // 循环遍历数据源的维度
    // TODO: optimize / multi-thread
    for (int64_t i2 = 0; i2 < ne2; ++i2) {
        for (int64_t i1 = 0; i1 < ne1; ++i1) {
            for (int64_t i0 = 0; i0 < ne0; ++i0) {
                const int ip2 = i2/w;
                const int ip1 = i1/w;

                const int64_t i02 = i2%w;
                const int64_t i01 = i1%w;
                const int64_t i00 = i0;

                const int64_t i = (ip2*npx + ip1)*ne02*ne01*ne00 + i02*ne01*ne00 + i01*ne00 + i00;
                const int64_t j =                                  i2*ne1*ne0    + i1*ne0   + i0;

                // 将源数据结构中的数据复制到目标数据结构中
                ((float *) dst->data)[j] = ((float *) src0->data)[i];
            }
        }
    }
}

// 计算前向窗口非分块操作
static void ggml_compute_forward_win_unpart(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    # 根据 src0 的类型进行不同的操作
    switch (src0->type) {
        # 如果 src0 的类型是 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                # 调用 ggml_compute_forward_win_unpart_f32 函数进行计算
                ggml_compute_forward_win_unpart_f32(params, src0, dst);
            } break;
        # 如果 src0 的类型不是 GGML_TYPE_F32
        default:
            {
                # 断言，表示出现了不应该出现的情况
                GGML_ASSERT(false);
            } break;
    }
// 计算一元操作的前向传播，根据操作类型调用相应的函数
static void ggml_compute_forward_unary(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 获取一元操作的类型
    const enum ggml_unary_op op = ggml_get_unary_op(dst);

    // 根据不同的一元操作类型进行不同的处理
    switch (op) {
        // 绝对值操作
        case GGML_UNARY_OP_ABS:
            {
                ggml_compute_forward_abs(params, src0, dst);
            } break;
        // 符号操作
        case GGML_UNARY_OP_SGN:
            {
                ggml_compute_forward_sgn(params, src0, dst);
            } break;
        // 取反操作
        case GGML_UNARY_OP_NEG:
            {
                ggml_compute_forward_neg(params, src0, dst);
            } break;
        // 阶跃函数操作
        case GGML_UNARY_OP_STEP:
            {
                ggml_compute_forward_step(params, src0, dst);
            } break;
        // 双曲正切操作
        case GGML_UNARY_OP_TANH:
            {
                ggml_compute_forward_tanh(params, src0, dst);
            } break;
        // ELU 激活函数操作
        case GGML_UNARY_OP_ELU:
            {
                ggml_compute_forward_elu(params, src0, dst);
            } break;
        // ReLU 激活函数操作
        case GGML_UNARY_OP_RELU:
            {
                ggml_compute_forward_relu(params, src0, dst);
            } break;
        // GELU 激活函数操作
        case GGML_UNARY_OP_GELU:
            {
                ggml_compute_forward_gelu(params, src0, dst);
            } break;
        // 快速 GELU 激活函数操作
        case GGML_UNARY_OP_GELU_QUICK:
            {
                ggml_compute_forward_gelu_quick(params, src0, dst);
            } break;
        // SiLU 激活函数操作
        case GGML_UNARY_OP_SILU:
            {
                ggml_compute_forward_silu(params, src0, dst);
            } break;
        // HardSwish 激活函数操作
        case GGML_UNARY_OP_HARDSWISH:
            {
                ggml_compute_forward_hardswish(params, src0, dst);
            } break;
        // HardSigmoid 激活函数操作
        case GGML_UNARY_OP_HARDSIGMOID:
            {
                ggml_compute_forward_hardsigmoid(params, src0, dst);
            } break;
        // 默认情况，断言操作为假
        default:
            {
                GGML_ASSERT(false);
            } break;
    }
}

// 获取相对位置的前向传播
// 计算相对位置并将结果存储在目标张量中，针对半精度浮点数类型的张量
static void ggml_compute_forward_get_rel_pos_f16(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 如果参数类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 引用：https://github.com/facebookresearch/segment-anything/blob/main/segment_anything/modeling/image_encoder.py#L292-L322

    GGML_TENSOR_UNARY_OP_LOCALS

    // 获取 ne1 的值
    const int64_t w = ne1;

    // 将源张量的数据转换为半精度浮点数类型指针
    ggml_fp16_t * src0_data = (ggml_fp16_t *) src0->data;
    // 将目标张量的数据转换为半精度浮点数类型指针
    ggml_fp16_t * dst_data  = (ggml_fp16_t *) dst->data;

    // 遍历 ne2
    for (int64_t i2 = 0; i2 < ne2; ++i2) {
        // 遍历 ne1
        for (int64_t i1 = 0; i1 < ne1; ++i1) {
            // 计算位置
            const int64_t pos = (w - i1 - 1) + i2;
            // 遍历 ne0
            for (int64_t i0 = 0; i0 < ne0; ++i0) {
                // 将源张量数据复制到目标张量中
                dst_data[i2*ne1*ne0 + i1*ne0 + i0] = src0_data[pos*ne00 + i0];
            }
        }
    }
}

// 根据源张量的类型调用相应的函数
static void ggml_compute_forward_get_rel_pos(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    switch (src0->type) {
        case GGML_TYPE_F16:
            {
                ggml_compute_forward_get_rel_pos_f16(params, src0, dst);
            } break;
        default:
            {
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_add_rel_pos

// 计算相对位置并将结果添加到目标张量中，针对单精度浮点数类型的张量
static void ggml_compute_forward_add_rel_pos_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        const struct ggml_tensor * src2,
        struct ggml_tensor * dst) {

    // 检查是否为原地操作
    const bool inplace = (bool) ((int32_t *) dst->op_params)[0];
    // 如果不是原地操作且参数类型为初始化，则进行数据复制
    if (!inplace && params->type == GGML_TASK_INIT) {
        if (params->ith != 0) {
            return;
        }
        // 将源张量的数据复制到目标张量中
        memcpy((char *) dst->data, (char *) src0->data, ggml_nbytes(dst));
        return;
    }
    // 如果参数类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    // 结束当前函数的定义
    }

    // 记录当前时间戳
    int64_t t0 = ggml_perf_time_us();
    UNUSED(t0);

    // 引用：https://github.com/facebookresearch/segment-anything/blob/main/segment_anything/modeling/image_encoder.py#L357-L359

    // 将输入数据转换为 float 类型指针
    float * src1_data = (float *) src1->data;
    float * src2_data = (float *) src2->data;
    float * dst_data  = (float *) dst->data;

    // 获取输入数据的维度信息
    const int64_t ne10 = src1->ne[0];
    const int64_t ne11 = src1->ne[1];
    const int64_t ne12 = src1->ne[2];
    const int64_t ne13 = src1->ne[3];

    // 获取线程参数
    const int ith = params->ith;
    const int nth = params->nth;

    // 计算目标数据中的总补丁数
    const int np = ne13;

    // 每个线程处理的补丁数
    const int dp = (np + nth - 1)/nth;

    // 当前线程处理的补丁范围
    const int ip0 = dp*ith;
    const int ip1 = MIN(ip0 + dp, np);

    // 循环遍历处理每个补丁
    for (int64_t i13 = ip0; i13 < ip1; ++i13) {
        for (int64_t i12 = 0; i12 < ne12; ++i12) {
            for (int64_t i11 = 0; i11 < ne11; ++i11) {
                const int64_t jp1 = i13*ne12*ne11*ne10 + i12*ne11*ne10 + i11*ne10;
                for (int64_t i10 = 0; i10 < ne10; ++i10) {
                    const int64_t jp0  = jp1 + i10;
                    const float src1_e = src1_data[jp0];
                    const float src2_e = src2_data[jp0];

                    const int64_t jdh = jp0 * ne10;
                    const int64_t jdw = jdh - (ne10 - 1) * i10;

                    // 更新目标数据中的值
                    for (int64_t j = 0; j < ne10; ++j) {
                        dst_data[jdh + j     ] += src2_e;
                        dst_data[jdw + j*ne10] += src1_e;
                    }
                }
            }
        }
    }
// 计算前向操作，将两个输入张量相加，并将结果存储在目标张量中
static void ggml_compute_forward_add_rel_pos(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        const struct ggml_tensor * src2,
        struct ggml_tensor * dst) {
    // 根据第一个输入张量的类型进行不同的操作
    switch (src0->type) {
        // 如果类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用相应的函数进行 F32 类型的操作
                ggml_compute_forward_add_rel_pos_f32(params, src0, src1, src2, dst);
            } break;
        // 如果类型不是 GGML_TYPE_F32
        default:
            {
                // 断言失败
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_map_unary

// 对 F32 类型的输入张量进行一元操作
static void ggml_compute_forward_map_unary_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst,
        const ggml_unary_op_f32_t fun) {
    // 断言输入张量和输出张量的形状相同
    GGML_ASSERT(ggml_are_same_shape(src0, dst));

    // 如果任务类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取输入张量的行数和通道数
    const int n  = ggml_nrows(src0);
    const int nc = src0->ne[0];

    // 断言输出张量和输入张量的数据类型为 float
    assert( dst->nb[0] == sizeof(float));
    assert(src0->nb[0] == sizeof(float));

    // 遍历每一行数据，对每一行进行一元操作
    for (int i = 0; i < n; i++) {
        fun(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1]));
    }
}

// 对输入张量进行一元操作
static void ggml_compute_forward_map_unary(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst,
        const ggml_unary_op_f32_t fun) {
    // 根据输入张量的类型进行不同的操作
    switch (src0->type) {
        // 如果类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用相应的函数进行 F32 类型的操作
                ggml_compute_forward_map_unary_f32(params, src0, dst, fun);
            } break;
        // 如果类型不是 GGML_TYPE_F32
        default:
            {
                // 断言失败
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_map_binary
// 计算两个浮点数张量的二元操作结果，并存储到目标张量中
static void ggml_compute_forward_map_binary_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst,
        const ggml_binary_op_f32_t fun) {
    // 断言参数中的 ith 值为 0
    assert(params->ith == 0);
    // 断言 src0、src1 和 dst 张量具有相同的形状
    assert(ggml_are_same_shape(src0, src1) && ggml_are_same_shape(src0, dst));

    // 如果任务类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取 src0 张量的行数
    const int n  = ggml_nrows(src0);
    // 获取 src0 张量的列数
    const int nc = src0->ne[0];

    // 断言目标张量的第一个维度大小为 float 类型的大小
    assert( dst->nb[0] == sizeof(float));
    // 断言 src0 张量的第一个维度大小为 float 类型的大小
    assert(src0->nb[0] == sizeof(float));
    // 断言 src1 张量的第一个维度大小为 float 类型的大小
    assert(src1->nb[0] == sizeof(float));

    // 遍历每一行，对每一行的数据进行二元操作
    for (int i = 0; i < n; i++) {
        fun(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1])),
                (float *) ((char *) src1->data + i*(src1->nb[1])));
    }
}

// 根据不同的张量类型选择对应的二元操作函数进行计算
static void ggml_compute_forward_map_binary(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst,
        const ggml_binary_op_f32_t fun) {
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                ggml_compute_forward_map_binary_f32(params, src0, src1, dst, fun);
            } break;
        default:
            {
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_map_custom1

// 自定义操作函数，对浮点数张量进行操作
static void ggml_compute_forward_map_custom1_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * a,
        struct ggml_tensor * dst,
        const ggml_custom1_op_f32_t fun) {
    // 断言参数中的 ith 值为 0
    assert(params->ith == 0);

    // 如果任务类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 调用自定义操作函数，对目标张量进行操作
    fun(dst, a);
}

// ggml_compute_forward_map_custom2
// 计算具有自定义操作函数的前向映射，输入为两个张量，输出为一个张量
static void ggml_compute_forward_map_custom2_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * a,
        const struct ggml_tensor * b,
        struct ggml_tensor * dst,
        const ggml_custom2_op_f32_t fun) {
    // 断言参数中的 ith 值为 0
    assert(params->ith == 0);

    // 如果任务类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 调用自定义操作函数，将结果存储在目标张量中
    fun(dst, a, b);
}

// ggml_compute_forward_map_custom3

// 计算具有自定义操作函数的前向映射，输入为三个张量，输出为一个张量
static void ggml_compute_forward_map_custom3_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * a,
        const struct ggml_tensor * b,
        const struct ggml_tensor * c,
        struct ggml_tensor * dst,
        const ggml_custom3_op_f32_t fun) {
    // 断言参数中的 ith 值为 0
    assert(params->ith == 0);

    // 如果任务类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 调用自定义操作函数，将结果存储在目标张量中
    fun(dst, a, b, c);
}

// ggml_compute_forward_map_custom1

// 计算具有自定义操作函数的前向映射，输入为一个张量，输出为一个张量
static void ggml_compute_forward_map_custom1(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * a,
              struct ggml_tensor * dst) {
    // 如果任务类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取目标张量的自定义操作参数
    struct ggml_map_custom1_op_params * p = (struct ggml_map_custom1_op_params *) dst->op_params;

    // 调用自定义操作函数，将结果存储在目标张量中
    p->fun(dst, a, params->ith, params->nth, p->userdata);
}

// ggml_compute_forward_map_custom2

// 计算具有自定义操作函数的前向映射，输入为两个张量，输出为一个张量
static void ggml_compute_forward_map_custom2(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * a,
        const struct ggml_tensor * b,
              struct ggml_tensor * dst) {
    // 如果任务类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取目标张量的自定义操作参数
    struct ggml_map_custom2_op_params * p = (struct ggml_map_custom2_op_params *) dst->op_params;

    // 调用自定义操作函数，将结果存储在目标张量中
    p->fun(dst, a, b, params->ith, params->nth, p->userdata);
}

// ggml_compute_forward_map_custom3
// 计算自定义操作的前向映射函数，根据参数和输入张量计算输出张量
static void ggml_compute_forward_map_custom3(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * a,
        const struct ggml_tensor * b,
        const struct ggml_tensor * c,
              struct ggml_tensor * dst) {
    // 如果任务类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取输出张量的自定义操作参数
    struct ggml_map_custom3_op_params * p = (struct ggml_map_custom3_op_params *) dst->op_params;

    // 调用自定义操作函数，计算输出张量的值
    p->fun(dst, a, b, c, params->ith, params->nth, p->userdata);
}

// ggml_compute_forward_cross_entropy_loss

// 计算交叉熵损失的前向传播函数，根据参数和输入张量计算输出张量
static void ggml_compute_forward_cross_entropy_loss_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 断言输入张量 src0 是连续的
    GGML_ASSERT(ggml_is_contiguous(src0));
    // 断言输入张量 src1 是连续的
    GGML_ASSERT(ggml_is_contiguous(src1));
    // 断言输出张量 dst 是标量
    GGML_ASSERT(ggml_is_scalar(dst));
    // 断言输入张量 src0 和 src1 的形状相同
    GGML_ASSERT(ggml_are_same_shape(src0, src1));

    // 获取参数中的 ith 和 nth
    const int ith = params->ith;
    const int nth = params->nth;

    // 将参数中的 wdata 转换为 float 类型的指针
    float * sums = (float *) params->wdata;

    // TODO: 处理转置/置换矩阵
    // 获取输入张量 src0 的列数和行数
    const int nc = src0->ne[0];
    const int nr = ggml_nrows(src0);

    // 断言参数中的 wsize 大于等于所需的内存大小
    GGML_ASSERT(params->wsize >= sizeof(float) * (nth + nth * nc));

    // 如果任务类型为初始化
    if (params->type == GGML_TASK_INIT) {
        // 如果是第一个线程
        if (ith == 0) {
            // 将 sums 数组清零
            memset(sums, 0, sizeof(float) * (nth + nth * nc));
        }
        return;
    }

    // 如果任务类型为结束
    if (params->type == GGML_TASK_FINALIZE) {
        // 如果是第一个线程
        if (ith == 0) {
            // 将 dst 数据转换为 float 类型的指针
            float * dp = (float *) dst->data;
            // 对 dp 进行求和操作
            ggml_vec_sum_f32(nth, dp, sums);
            // 将 dp 的第一个元素乘以 -1.0f / nr
            dp[0] *= -1.0f / (float) nr;
        }
        return;
    }

    // 定义一个很小的值 eps
    const double eps = 1e-9;

    // 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);
}
    # 遍历循环，从 ir0 开始，直到 ir1 结束
    for (int i1 = ir0; i1 < ir1; i1++) {
        # 计算第一个源数据的地址，根据数据类型和偏移量计算得到
        float * s0 = (float *)((char *) src0->data + i1*src0->nb[1]);
        # 计算第二个源数据的地址，根据数据类型和偏移量计算得到
        float * s1 = (float *)((char *) src1->data + i1*src1->nb[1]);
        # 计算目标数据的地址，根据参数中的数据宽度、线程数和线程索引计算得到
        float * st = ((float *) params->wdata) + nth + ith*nc;
#ifndef NDEBUG
        // 如果处于调试模式，检查输入数据是否包含 NaN 值
        for (int i = 0; i < nc; ++i) {
            //printf("p[%d] = %f\n", i, p[i]);
            assert(!isnan(s0[i]));
            assert(!isnan(s1[i]));
        }
#endif
        // soft_max
        // 初始化变量 sum 为 0.0
        ggml_float sum = 0.0;
        {
            // 初始化变量 max 为负无穷
            float max = -INFINITY;
            // 获取 s0 中的最大值
            ggml_vec_max_f32(nc, &max, s0);

            uint16_t scvt; UNUSED(scvt);
            // 遍历 s0 中的元素
            for (int i = 0; i < nc; i++) {
                // 如果 s0[i] 为负无穷，则将 st[i] 设为 0.0
                if (s0[i] == -INFINITY) {
                    st[i] = 0.0f;
                } else {
#ifndef GGML_CROSS_ENTROPY_EXP_FP16
                    // 计算 s0[i] - max，并计算 exp(s0[i] - max)
                    const float s = s0[i] - max;
                    const float val = expf(s);
#else
                    // 将 s0[i] - max 转换为 fp16 格式，并计算 exp(ggml_table_exp_f16[scvt])
                    ggml_fp16_t s = GGML_FP32_TO_FP16(s0[i] - max);
                    memcpy(&scvt, &s, sizeof(scvt));
                    const float val = GGML_FP16_TO_FP32(ggml_table_exp_f16[scvt]);
#endif
                    // 更新 sum，并将计算结果存入 st[i]
                    sum += (ggml_float)val;
                    st[i] = val;
                }
            }

            // 断言 sum 大于 0.0
            assert(sum > 0.0);
            // sum = 1.0/sum;
        }
        // 避免 log(0)，通过重新缩放将值从 [0..1] 调整到 [eps..1]
        sum = (1.0 - eps) / sum;
        // 对 st 中的元素进行缩放
        ggml_vec_scale_f32(nc, st, sum);
        // 对 st 中的元素进行加法操作
        ggml_vec_add1_f32(nc, st, st, eps);
        // 对 st 中的元素进行对数运算
        ggml_vec_log_f32(nc, st, st);
        // 对 st 和 s1 中的元素进行乘法操作
        ggml_vec_mul_f32(nc, st, st, s1);

        // 计算 st 中元素的总和
        float st_sum = 0;
        ggml_vec_sum_f32(nc, &st_sum, st);
        // 将结果存入 sums[ith]
        sums[ith] += st_sum;

#ifndef NDEBUG
        // 如果处于调试模式，检查 st 中的元素是否包含 NaN 或无穷值
        for (int i = 0; i < nc; ++i) {
            assert(!isnan(st[i]));
            assert(!isinf(st[i]));
        }
#endif
    }

}

// 计算前向交叉熵损失
static void ggml_compute_forward_cross_entropy_loss(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    # 根据 src0 的类型进行不同的操作
    switch (src0->type) {
        # 如果 src0 的类型是 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                # 调用 ggml_compute_forward_cross_entropy_loss_f32 函数处理参数 params, src0, src1, dst
                ggml_compute_forward_cross_entropy_loss_f32(params, src0, src1, dst);
            } break;
        # 如果 src0 的类型不是 GGML_TYPE_F32
        default:
            {
                # 断言，表示出现了不应该出现的情况
                GGML_ASSERT(false);
            } break;
    }
// 定义一个静态函数，用于计算前向交叉熵损失的反向传播，更新梯度
static void ggml_compute_forward_cross_entropy_loss_back_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        const struct ggml_tensor * opt0,
        struct ggml_tensor * dst) {
    // 断言目标张量是连续的
    GGML_ASSERT(ggml_is_contiguous(dst));
    // 断言输入张量 src0 是连续的
    GGML_ASSERT(ggml_is_contiguous(src0));
    // 断言输入张量 src1 是连续的
    GGML_ASSERT(ggml_is_contiguous(src1));
    // 断言输入张量 opt0 是连续的
    GGML_ASSERT(ggml_is_contiguous(opt0));
    // 断言输入张量 src0、src1 和目标张量 dst 的形状相同
    GGML_ASSERT(ggml_are_same_shape(src0, src1) && ggml_are_same_shape(src0, dst));

    // 获取当前线程的索引和总线程数
    const int64_t ith = params->ith;
    const int64_t nth = params->nth;

    // 如果任务类型是初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 设置一个很小的值 eps
    const double eps = 1e-9;

    // TODO: 处理转置/置换矩阵

    // 获取输入张量 src0 的列数和行数
    const int64_t nc = src0->ne[0];
    const int64_t nr = ggml_nrows(src0);

    // 每个线程处理的行数
    const int64_t dr = (nr + nth - 1)/nth;

    // 当前线程处理的行范围
    const int64_t ir0 = dr*ith;
    const int64_t ir1 = MIN(ir0 + dr, nr);

    // 获取梯度数据指针
    float * d   = (float *) opt0->data;

    // 遍历每一行
    for (int64_t i1 = ir0; i1 < ir1; i1++) {
        // 获取目标张量 dst 的数据指针
        float * ds0 = (float *)((char *) dst->data  + i1*dst->nb[1]);
        // 获取输入张量 src0 的数据指针
        float * s0  = (float *)((char *) src0->data + i1*src0->nb[1]);
        // 获取输入张量 src1 的数据指针
        float * s1  = (float *)((char *) src1->data + i1*src1->nb[1]);

#ifndef NDEBUG
        // 在调试模式下，检查输入张量 s0 和 s1 中是否有 NaN 值
        for (int i = 0; i < nc; ++i) {
            assert(!isnan(s0[i]));
            assert(!isnan(s1[i]));
        }
#endif

        // soft_max
        ggml_float sum = 0.0;
        {
            // 初始化最大值为负无穷
            float max = -INFINITY;
            // 获取输入张量 s0 中的最大值
            ggml_vec_max_f32(nc, &max, s0);

            uint16_t scvt; UNUSED(scvt);
            // 遍历每个元素
            for (int i = 0; i < nc; i++) {
                // 如果 s0[i] 是负无穷，则将 ds0[i] 设为 0.0
                if (s0[i] == -INFINITY) {
                    ds0[i] = 0.0f;
                } else {
#ifndef GGML_CROSS_ENTROPY_EXP_FP16
                    // 如果不使用 FP16 进行计算
                    const float s = s0[i] - max;
                    // 计算指数函数值
                    const float val = expf(s);
#else
                    // 如果使用 FP16 进行计算
                    ggml_fp16_t s = GGML_FP32_TO_FP16(s0[i] - max);
                    // 将 FP16 类型转换为字节流
                    memcpy(&scvt, &s, sizeof(scvt));
                    // 通过查表获取指数函数值
                    const float val = GGML_FP16_TO_FP32(ggml_table_exp_f16[scvt]);
#endif
                    // 累加指数函数值
                    sum += (ggml_float)val;
                    // 将指数函数值赋给 ds0[i]
                    ds0[i] = val;
                }
            }

            // 断言 sum 大于 0.0
            assert(sum > 0.0);
            // 计算归一化因子
            sum = (1.0 - eps)/sum;
        }

        // 计算梯度 grad(src0) = (softmax(src0) - src1) * grad(cross_entropy_loss(src0, src1)) / nr
        ggml_vec_scale_f32(nc, ds0, sum);
        ggml_vec_add1_f32(nc, ds0, ds0, eps);
        ggml_vec_sub_f32(nc, ds0, ds0, s1);
        ggml_vec_scale_f32(nc, ds0, d[0] / (float) nr);

#ifndef NDEBUG
        // 调试模式下的断言
        for (int i = 0; i < nc; ++i) {
            assert(!isnan(ds0[i]));
            assert(!isinf(ds0[i]));
        }
#endif
    }
}

// 计算交叉熵损失函数的反向传播
static void ggml_compute_forward_cross_entropy_loss_back(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        const struct ggml_tensor * opt0,
        struct ggml_tensor * dst) {
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                // 调用 F32 类型的反向传播函数
                ggml_compute_forward_cross_entropy_loss_back_f32(params, src0, src1, opt0, dst);
            } break;
        default:
            {
                // 断言错误
                GGML_ASSERT(false);
            } break;
    }
}

/////////////////////////////////

// 计算前向传播
static void ggml_compute_forward(struct ggml_compute_params * params, struct ggml_tensor * tensor) {
    GGML_ASSERT(params);

    // 如果操作为 NONE，则直接返回
    if (tensor->op == GGML_OP_NONE) {
        return;
    }

#ifdef GGML_USE_CUBLAS
    // 使用 CUDA 计算前向传播
    bool skip_cpu = ggml_cuda_compute_forward(params, tensor);
    // 如果跳过 CPU 计算，则直接返回
    if (skip_cpu) {
        return;
    }
    // 断言 tensor 的第一个源张量为空或者后端为 CPU
    GGML_ASSERT(tensor->src[0] == NULL || tensor->src[0]->backend == GGML_BACKEND_CPU);
    # 断言第二个源张量为空或者其后端为 CPU
    GGML_ASSERT(tensor->src[1] == NULL || tensor->src[1]->backend == GGML_BACKEND_CPU);
#elif defined(GGML_USE_VULKAN)
    // 如果使用 Vulkan，则调用 ggml_vk_compute_forward 函数计算前向传播
    const bool skip_cpu = ggml_vk_compute_forward(params, tensor);
#ifdef GGML_VULKAN_CHECK_RESULTS
    // 如果设置了 GGML_VULKAN_CHECK_RESULTS 宏，并且 skip_cpu 为真，则调用 ggml_vk_check_results_1 函数
    if (skip_cpu) {
        ggml_vk_check_results_1(params, tensor);
    }
#endif
    // 如果 skip_cpu 为真，则直接返回，不再执行后续代码
    if (skip_cpu) {
        return;
    }
    // 断言 tensor 的第一个源张量为空或者其后端为 GGML_BACKEND_CPU
    GGML_ASSERT(tensor->src[0] == NULL || tensor->src[0]->backend == GGML_BACKEND_CPU);
    // 断言 tensor 的第二个源张量为空或者其后端为 GGML_BACKEND_CPU
    GGML_ASSERT(tensor->src[1] == NULL || tensor->src[1]->backend == GGML_BACKEND_CPU);
#endif // GGML_USE_CUBLAS

#ifdef GGML_USE_SYCL
    // 如果使用 SYCL，则调用 ggml_sycl_compute_forward 函数计算前向传播
    bool skip_cpu = ggml_sycl_compute_forward(params, tensor);
    // 如果 skip_cpu 为真，则直接返回，不再执行后续代码
    if (skip_cpu) {
        return;
    }
#endif // GGML_USE_SYCL
    }
}

////////////////////////////////////////////////////////////////////////////////

static size_t ggml_hash_size(size_t min_sz) {
    // 下一个大于等于 min_sz 的素数
    static const size_t primes[] = {
        2, 3, 5, 11, 17, 37, 67, 131, 257, 521, 1031,
        2053, 4099, 8209, 16411, 32771, 65537, 131101,
        262147, 524309, 1048583, 2097169, 4194319, 8388617,
        16777259, 33554467, 67108879, 134217757, 268435459,
        536870923, 1073741827, 2147483659
    };
    static const size_t n_primes = sizeof(primes)/sizeof(primes[0]);

    // 找到大于等于 min_sz 的最小素数
    size_t l = 0;
    size_t r = n_primes;
    while (l < r) {
        size_t m = (l + r)/2;
        if (primes[m] < min_sz) {
            l = m + 1;
        } else {
            r = m;
        }
    }
    size_t sz = l < n_primes ? primes[l] : min_sz | 1;
    return sz;
}

static size_t ggml_hash(const void * p) {
    // 将指针转换为大小整数
    return (size_t)p;
}

size_t ggml_hash_find(const struct ggml_hash_set hash_set, struct ggml_tensor * key) {
    // 计算哈希值
    size_t h = ggml_hash(key) % hash_set.size;

    // 线性探测
    size_t i = h;
    while (hash_set.keys[i] != NULL && hash_set.keys[i] != key) {
        i = (i + 1) % hash_set.size;
        if (i == h) {
            // 遍历所有哈希表条目，未找到
            return GGML_HASHTABLE_FULL;
        }
    }
    # 返回变量 i 的值
    return i;
// 检查哈希集合中是否包含指定键值对应的索引
bool ggml_hash_contains(struct ggml_hash_set hash_set, struct ggml_tensor * key) {
    // 查找键值对应的索引
    size_t i = ggml_hash_find(hash_set, key);
    // 如果索引不等于哈希表已满且哈希集合中的键值等于指定键值，则返回 true
    return i != GGML_HASHTABLE_FULL && hash_set.keys[i] == key;
}

// 向哈希集合中插入指定键值对应的索引
size_t ggml_hash_insert(struct ggml_hash_set hash_set, struct ggml_tensor * key) {
    // 查找键值对应的索引
    size_t i = ggml_hash_find(hash_set, key);

    // 断言索引不等于哈希表已满
    GGML_ASSERT(i != GGML_HASHTABLE_FULL);

    // 如果哈希集合中的键值等于指定键值，则返回已存在的标志
    if (hash_set.keys[i] == key) {
        return GGML_HASHTABLE_ALREADY_EXISTS;
    }

    // 插入键值对应的索引
    GGML_ASSERT(hash_set.keys[i] == NULL);
    hash_set.keys[i] = key;
    return i;
}

// 查找或插入指定键值对应的索引
size_t ggml_hash_find_or_insert(struct ggml_hash_set hash_set, struct ggml_tensor * key) {
    // 查找键值对应的索引
    size_t i = ggml_hash_find(hash_set, key);

    // 断言索引不等于哈希表已满
    GGML_ASSERT(i != GGML_HASHTABLE_FULL);

    // 将指定键值插入到哈希集合中
    hash_set.keys[i] = key;
    return i;
}

// 创建新的哈希集合
struct ggml_hash_set ggml_hash_set_new(size_t size) {
    // 调整哈希集合的大小
    size = ggml_hash_size(size);
    struct ggml_hash_set result;
    result.size = size;
    // 分配内存并初始化哈希集合的键数组
    result.keys = GGML_MALLOC(sizeof(struct ggml_tensor *) * size);
    memset(result.keys, 0, sizeof(struct ggml_tensor *) * size);
    return result;
}

// 释放哈希集合的内存
static void ggml_hash_set_free(struct ggml_hash_set hash_set) {
    GGML_FREE(hash_set.keys);
}

// 哈希映射结构体
struct hash_map {
    struct ggml_hash_set set;
    struct ggml_tensor ** vals;
};

// 创建新的哈希映射
static struct hash_map * ggml_new_hash_map(size_t size) {
    // 分配内存并初始化哈希映射结构体
    struct hash_map * result = GGML_MALLOC(sizeof(struct hash_map));
    result->set = ggml_hash_set_new(size);
    // 分配内存并初始化哈希映射的值数组
    result->vals = GGML_MALLOC(sizeof(struct ggml_tensor *) * result->set.size);
    memset(result->vals, 0, sizeof(struct ggml_tensor *) * result->set.size);
    return result;
}

// 释放哈希映射的内存
static void ggml_hash_map_free(struct hash_map * map) {
    // 释放哈希集合的内存
    ggml_hash_set_free(map->set);
    // 释放哈希映射的值数组内存
    GGML_FREE(map->vals);
    // 释放哈希映射结构体内存
    GGML_FREE(map);
}

// 梯度检查点
static struct ggml_tensor * ggml_recompute_graph_node(
        struct ggml_context * ctx,
        struct ggml_cgraph  * graph,
        struct hash_map     * replacements,
        struct ggml_tensor  * node) {
    // 如果节点为空，则返回空指针
    if (node == NULL) {
        return NULL;
    }

    // 如果节点是参数节点，则直接返回该节点
    if (node->is_param) {
        return node;
    }

    // 如果节点不在已访问的哈希表中，则返回该节点
    if (!ggml_hash_contains(graph->visited_hash_table, node)) {
        return node;
    }

    // 统计节点的子节点数量
    int count_children = 0;
    for (int k = 0; k < GGML_MAX_SRC; ++k) {
        if (node->src[k]) {
            ++count_children;
        }
    }

    // 如果节点没有子节点，则返回该节点
    if (count_children == 0) {
        return node;
    }

    // 在替换集合中查找节点的索引
    size_t i = ggml_hash_find(replacements->set, node);
    GGML_ASSERT(i != GGML_HASHTABLE_FULL); // 断言替换集合不是满的

    // 如果替换集合中已经存在该节点的映射，则返回映射的克隆节点
    if (replacements->set.keys[i] == node) {
        return replacements->vals[i];
    }

    // 创建节点的克隆
    struct ggml_tensor * clone = ggml_new_tensor(ctx, node->type, GGML_MAX_DIMS, node->ne);

    // 将克隆节点插入替换集合
    GGML_ASSERT(replacements->set.keys[i] == NULL); // 断言不会覆盖已有映射
    replacements->set.keys[i] = node;
    replacements->vals[i] = clone;

    // 复制节点的属性到克隆节点
    clone->op       = node->op;
    clone->grad     = node->grad;
    clone->is_param = node->is_param;
    clone->extra    = node->extra;
    for (int k = 0; k < GGML_MAX_DIMS; ++k) {
        clone->nb[k] = node->nb[k];
    }
    for (int k = 0; k < GGML_MAX_SRC; ++k) {
        clone->src[k] = ggml_recompute_graph_node(ctx, graph, replacements, node->src[k]);
    }
    if (node->view_src != NULL) {
        clone->data = (node->view_src->data == NULL)
                        ? NULL // 如果view_src尚未分配，则数据为空
                        : (char *) node->view_src->data // 如果view_src已经分配，则指向数据偏移
                                 + node->view_offs;
        clone->view_src  = node->view_src;
        clone->view_offs = node->view_offs;
    }

    // 断言节点的操作参数和名称的大小
    GGML_ASSERT(sizeof(node->op_params) == sizeof(int32_t) * (GGML_MAX_OP_PARAMS / sizeof(int32_t)));
    GGML_ASSERT(sizeof(node->name)      == GGML_MAX_NAME);
    // 复制节点的操作参数，并格式化克隆节点的名称
    memcpy(clone->op_params, node->op_params, sizeof(node->op_params));
    ggml_format_name(clone, "%s (clone)", ggml_get_name(node));

    // 返回克隆节点
    return clone;
// 构建反向梯度检查点
void ggml_build_backward_gradient_checkpointing(
        struct ggml_context   * ctx,
        struct ggml_cgraph    * gf,
        struct ggml_cgraph    * gb,
        struct ggml_cgraph    * gb_tmp,
        struct ggml_tensor  * * checkpoints,
        int                     n_checkpoints) {
    // 复制前向图到临时图
    ggml_graph_cpy(gf, gb_tmp);
    // 构建反向扩展
    ggml_build_backward_expand(ctx, gf, gb_tmp, true);

    // 如果检查点数量小于等于0，则直接复制临时图到反向图并返回
    if (n_checkpoints <= 0) {
        ggml_graph_cpy(gb_tmp, gb);
        return;
    }

    // 创建哈希映射用于替换
    struct hash_map * replacements = ggml_new_hash_map(gf->n_nodes + gf->n_leafs + n_checkpoints);

    // 在替换中插入检查点
    for (int i = 0; i < n_checkpoints; ++i) {
        size_t k = ggml_hash_find(replacements->set, checkpoints[i]);
        GGML_ASSERT(k != GGML_HASHTABLE_FULL); // 断言不是满的
        GGML_ASSERT(replacements->set.keys[k] == NULL); // 断言不会覆盖
        replacements->set.keys[k] = checkpoints[i];
        replacements->vals[k]     = checkpoints[i];
    }

    // 复制前向图到反向图
    ggml_graph_cpy(gf, gb);
    // 重写 gb_tmp->nodes[gf->n_nodes:gb_tmp->n_nodes],
    // 通过从检查点重新计算替换对 gb_tmp->nodes[0:gf->n_nodes]（等于 gf->nodes[0:gf->n_nodes]）的引用
    for (int i = gf->n_nodes; i<gb_tmp->n_nodes; ++i) {
        struct ggml_tensor * node = gb_tmp->nodes[i];
        for (int k = 0; k < GGML_MAX_SRC; ++k) {
            // 插入重新计算 src 的新张量，重用已经做好的替换，
            // 记住替换：记住具有从对应的 gf 节点的映射的新张量
            // 递归处理输入张量，
            // 除非（即终止时）输入张量是替换（如检查点）
            node->src[k] = ggml_recompute_graph_node(ctx, gf, replacements, node->src[k]);
        }
        // 将带有已进行替换的节点插入到结果反向图 gb 中
        ggml_build_forward_expand(gb, node);
    }

    // 释放哈希映射
    ggml_hash_map_free(replacements);
}
// 根据输入 a 是否在 zero_table 中来决定返回 b 或者调用 ggml_add_impl 函数
static struct ggml_tensor * ggml_add_or_set(struct ggml_context * ctx, struct ggml_tensor * a, struct ggml_tensor * b, struct ggml_hash_set zero_table) {
    if (ggml_hash_contains(zero_table, a)) {
        return b;
    } else {
        return ggml_add_impl(ctx, a, b, false);
    }
}

// 根据输入 a 是否在 zero_table 中来决定返回经过处理的 b 或者调用 ggml_acc_impl 函数
static struct ggml_tensor * ggml_acc_or_set(struct ggml_context * ctx, struct ggml_tensor * a, struct ggml_tensor * b, size_t nb1, size_t nb2, size_t nb3, size_t offset, struct ggml_hash_set zero_table) {
    if (ggml_hash_contains(zero_table, a)) {
        struct ggml_tensor * a_zero = ggml_scale(ctx, a, 0.0f);
        return ggml_acc_impl(ctx, a_zero, b, nb1, nb2, nb3, offset, false);
    } else {
        return ggml_acc_impl(ctx, a, b, nb1, nb2, nb3, offset, false);
    }
}

// 根据输入 a 是否在 zero_table 中来决定返回 b 的重复值或者调用 ggml_add1_impl 函数
static struct ggml_tensor * ggml_add1_or_set(struct ggml_context * ctx, struct ggml_tensor * a, struct ggml_tensor * b, struct ggml_hash_set zero_table) {
    if (ggml_hash_contains(zero_table, a)) {
        return ggml_repeat(ctx, b, a);
    } else {
        return ggml_add1_impl(ctx, a, b, false);
    }
}

// 根据输入 a 是否在 zero_table 中来决定返回 b 的负值或者调用 ggml_sub_impl 函数
static struct ggml_tensor * ggml_sub_or_set(struct ggml_context * ctx, struct ggml_tensor * a, struct ggml_tensor * b, struct ggml_hash_set zero_table) {
    if (ggml_hash_contains(zero_table, a)) {
        return ggml_neg(ctx, b);
    } else {
        return ggml_sub_impl(ctx, a, b, false);
    }
}

// 计算反向传播，根据输入 tensor 的 src[0] 和 src[1] 来进行处理
static void ggml_compute_backward(struct ggml_context * ctx, struct ggml_tensor * tensor, struct ggml_hash_set zero_table) {
    struct ggml_tensor * src0 = tensor->src[0];
    struct ggml_tensor * src1 = tensor->src[1];

    // 遍历 tensor 的 src 数组，检查是否存在梯度，并且梯度与源张量的形状相同
    for (int i = 0; i < GGML_MAX_SRC; ++i) {
        if (tensor->src[i] && tensor->src[i]->grad) {
            GGML_ASSERT(ggml_are_same_shape(tensor->src[i], tensor->src[i]->grad));
        }
    }
}
// 访问节点的父节点，构建计算图
static void ggml_visit_parents(struct ggml_cgraph * cgraph, struct ggml_tensor * node) {
    // 如果节点的梯度为空
    if (node->grad == NULL) {
        // 这通常发生在我们从反向传播中的常量生成中间节点时
        // 也可能在前向传播期间发生，如果用户使用常量进行计算
        if (node->op != GGML_OP_NONE) {
            //GGML_PRINT_DEBUG("%s: warning: node %p has no grad, but op %d\n", __func__, (void *) node, node->op);
        }
    }

    // 检查是否已经访问过
    if (ggml_hash_insert(cgraph->visited_hash_table, node) == GGML_HASHTABLE_ALREADY_EXISTS) {
        return;
    }

    for (int i = 0; i < GGML_MAX_SRC; ++i) {
        const int k =
            (cgraph->order == GGML_CGRAPH_EVAL_ORDER_LEFT_TO_RIGHT) ? i :
            (cgraph->order == GGML_CGRAPH_EVAL_ORDER_RIGHT_TO_LEFT) ? (GGML_MAX_SRC-1-i) :
            /* unknown order, just fall back to using i*/ i;
        if (node->src[k]) {
            ggml_visit_parents(cgraph, node->src[k]);
        }
    }

    if (node->op == GGML_OP_NONE && node->grad == NULL) {
        // 到达叶节点，不是梯度图的一部分（例如常量）
        GGML_ASSERT(cgraph->n_leafs < cgraph->size);

        if (strlen(node->name) == 0) {
            ggml_format_name(node, "leaf_%d", cgraph->n_leafs);
        }

        cgraph->leafs[cgraph->n_leafs] = node;
        cgraph->n_leafs++;
    } else {
        GGML_ASSERT(cgraph->n_nodes < cgraph->size);

        if (strlen(node->name) == 0) {
            ggml_format_name(node, "node_%d", cgraph->n_nodes);
        }

        cgraph->nodes[cgraph->n_nodes] = node;
        if (cgraph->grads) {
            cgraph->grads[cgraph->n_nodes] = node->grad;
        }
        cgraph->n_nodes++;
    }
}

static void ggml_build_forward_impl(struct ggml_cgraph * cgraph, struct ggml_tensor * tensor, bool expand) {
    // 如果不需要扩展图结构
    if (!expand) {
        // TODO: 这个分支不再可访问，也许应该将其移到 ggml_build_forward_expand
        // 清空计算图中的节点
        ggml_graph_clear(cgraph);
    }

    // 获取当前计算图中节点的数量
    const int n0 = cgraph->n_nodes;
    // 未使用的变量，用于消除编译器警告
    UNUSED(n0);

    // 访问父节点，更新计算图结构
    ggml_visit_parents(cgraph, tensor);

    // 计算新添加的节点数量
    const int n_new = cgraph->n_nodes - n0;
    // 打印调试信息，显示访问了多少个新节点
    GGML_PRINT_DEBUG("%s: visited %d new nodes\n", __func__, n_new);

    // 如果有新节点被访问
    if (n_new > 0) {
        // 最后添加的节点应该始终是起始点
        // 断言最后一个节点是指定的张量
        GGML_ASSERT(cgraph->nodes[cgraph->n_nodes - 1] == tensor);
    }
}

// 构建前向传播的扩展
void ggml_build_forward_expand(struct ggml_cgraph * cgraph, struct ggml_tensor * tensor) {
    // 调用内部实现函数进行前向传播
    ggml_build_forward_impl(cgraph, tensor, true);
}

// 构建反向传播的扩展
void ggml_build_backward_expand(struct ggml_context * ctx, struct ggml_cgraph * gf, struct ggml_cgraph * gb, bool keep) {
    // 断言梯度图中节点数量大于0
    GGML_ASSERT(gf->n_nodes > 0);

    // 如果需要保留梯度图，必须将梯度节点从原始图中分离
    if (keep) {
        for (int i = 0; i < gf->n_nodes; i++) {
            struct ggml_tensor * node = gf->nodes[i];

            if (node->grad) {
                // 复制梯度节点
                node->grad = ggml_dup_tensor(ctx, node);
                gf->grads[i] = node->grad;
            }
        }
    }

    // 记录初始梯度，其值为零
    struct ggml_hash_set zero_table = ggml_hash_set_new(gf->size);
    for (int i = 0; i < gf->n_nodes; i++) {
        if (gf->grads[i]) {
            ggml_hash_insert(zero_table, gf->grads[i]);
        }
    }

    // 从后向前遍历节点，计算反向传播
    for (int i = gf->n_nodes - 1; i >= 0; i--) {
        struct ggml_tensor * node = gf->nodes[i];

        // inplace 操作不会被 ggml_compute_backward 创建，使用分配器自动创建 inplace 操作
        if (node->grad) {
            ggml_compute_backward(ctx, node, zero_table);
        }
    }

    // 遍历节点，如果是参数节点，则进行前向传播扩展
    for (int i = 0; i < gf->n_nodes; i++) {
        struct ggml_tensor * node = gf->nodes[i];

        if (node->is_param) {
            GGML_PRINT_DEBUG("%s: found root node %p\n", __func__, (void *) node);
            ggml_build_forward_expand(gb, node->grad);
        }
    }

    // 释放 zero_table
    ggml_hash_set_free(zero_table);
}

// 计算图的字节数
static size_t ggml_graph_nbytes(size_t size, bool grads) {
    size_t nbytes = sizeof(struct ggml_cgraph);
    nbytes += size * sizeof(struct ggml_tensor *) * 2; // 叶子节点 + 普通节点
    if (grads) {
        nbytes += size * sizeof(struct ggml_tensor *); // 梯度节点
    }
    nbytes += ggml_hash_size(size * 2) * sizeof(struct ggml_tensor *); // 哈希集合
    # 返回变量nbytes的值
    return nbytes;
// 计算自定义图形的额外开销，包括对象大小和内存对齐
size_t ggml_graph_overhead_custom(size_t size, bool grads) {
    return GGML_OBJECT_SIZE + GGML_PAD(ggml_graph_nbytes(size, grads), GGML_MEM_ALIGN);
}

// 返回默认图形的额外开销
size_t ggml_graph_overhead(void) {
    return ggml_graph_overhead_custom(GGML_DEFAULT_GRAPH_SIZE, false);
}

// 创建新的自定义图形对象
struct ggml_cgraph * ggml_new_graph_custom(struct ggml_context * ctx, size_t size, bool grads) {
    // 计算对象大小
    const size_t obj_size = ggml_graph_nbytes(size, grads);
    // 创建新对象
    struct ggml_object * obj = ggml_new_object(ctx, GGML_OBJECT_GRAPH, obj_size);
    // 获取图形对象的指针
    struct ggml_cgraph * cgraph = (struct ggml_cgraph *) ((char *) ctx->mem_buffer + obj->offs);

    // 初始化数据起始指针
    struct ggml_tensor ** data_start = (struct ggml_tensor **) (cgraph + 1);

    // 计算哈希表大小
    size_t hash_size = ggml_hash_size(size * 2);
    // 初始化节点、叶子、哈希键和梯度指针
    struct ggml_tensor ** nodes_ptr = data_start;
    struct ggml_tensor ** leafs_ptr = nodes_ptr + size;
    struct ggml_tensor ** hash_keys_ptr = leafs_ptr + size;
    struct ggml_tensor ** grads_ptr = grads ? hash_keys_ptr + hash_size : NULL;

    // 检查是否分配了正确数量的内存
    assert(obj_size == (size_t) (
        (grads ? (char *)(grads_ptr + size) : (char *)(hash_keys_ptr + hash_size)) - (char *)cgraph));

    // 将哈希键指针清零
    memset(hash_keys_ptr, 0, hash_size * sizeof(struct ggml_tensor *));

    // 初始化图形对象的属性
    *cgraph = (struct ggml_cgraph) {
        /*.size         =*/ size,
        /*.n_nodes      =*/ 0,
        /*.n_leafs      =*/ 0,
        /*.nodes        =*/ nodes_ptr,
        /*.grads        =*/ grads_ptr,
        /*.leafs        =*/ leafs_ptr,
        /*.hash_table   =*/ { hash_size, hash_keys_ptr },
        /*.order        =*/ GGML_CGRAPH_EVAL_ORDER_LEFT_TO_RIGHT,
        /*.perf_runs    =*/ 0,
        /*.perf_cycles  =*/ 0,
        /*.perf_time_us =*/ 0,
    };

    // 返回图形对象指针
    return cgraph;
}

// 创建新的默认图形对象
struct ggml_cgraph * ggml_new_graph(struct ggml_context * ctx) {
    return ggml_new_graph_custom(ctx, GGML_DEFAULT_GRAPH_SIZE, false);
}

// 返回图形对象的视图
struct ggml_cgraph ggml_graph_view(struct ggml_cgraph * cgraph0, int i0, int i1) {
    // 定义一个结构体变量 cgraph，表示计算图
    struct ggml_cgraph cgraph = {
        // 计算图的大小为 0
        /*.size         =*/ 0,
        // 计算图的节点数量为 i1 - i0
        /*.n_nodes      =*/ i1 - i0,
        // 计算图的叶子节点数量为 0
        /*.n_leafs      =*/ 0,
        // 计算图的节点数组指针指向 cgraph0 结构体中的 nodes 数组的第 i0 个元素
        /*.nodes        =*/ cgraph0->nodes + i0,
        // 计算图的梯度数组指针指向 cgraph0 结构体中的 grads 数组的第 i0 个元素，如果 grads 为 NULL，则指向 NULL
        /*.grads        =*/ cgraph0->grads ? cgraph0->grads + i0 : NULL,
        // 计算图的叶子节点数组指针为 NULL
        /*.leafs        =*/ NULL,
        // 计算图的哈希表初始化为 { 0, NULL }
        /*.hash_table   =*/ { 0, NULL },
        // 计算图的顺序与 cgraph0 结构体的顺序相同
        /*.order        =*/ cgraph0->order,
        // 计算图的性能运行次数为 0
        /*.perf_runs    =*/ 0,
        // 计算图的性能周期数为 0
        /*.perf_cycles  =*/ 0,
        // 计算图的性能时间为 0 微秒
        /*.perf_time_us =*/ 0,
    };

    // 返回构建好的计算图结构体
    return cgraph;
}

// 复制源图到目标图
void ggml_graph_cpy(struct ggml_cgraph * src, struct ggml_cgraph * dst) {
    // 断言目标图的大小大于等于源图的叶子节点数
    GGML_ASSERT(dst->size >= src->n_leafs);
    // 断言目标图的大小大于等于源图的节点数
    GGML_ASSERT(dst->size >= src->n_nodes);
    // 断言目标图的访问哈希表大小大于等于源图的访问哈希表大小
    GGML_ASSERT(dst->visited_hash_table.size >= src->visited_hash_table.size);

    // 复制源图的叶子节点数到目标图
    dst->n_leafs = src->n_leafs;
    // 复制源图的节点数到目标图
    dst->n_nodes = src->n_nodes;
    // 复制源图的顺序到目标图
    dst->order   = src->order;

    // 复制源图的叶子节点到目标图
    for (int i = 0; i < src->n_leafs; ++i) {
        dst->leafs[i] = src->leafs[i];
    }

    // 复制源图的节点到目标图
    for (int i = 0; i < src->n_nodes; ++i) {
        dst->nodes[i] = src->nodes[i];
    }

    // 如果源图的梯度存在
    if (src->grads) {
        // 断言目标图的梯度不为空
        GGML_ASSERT(dst->grads != NULL);
        // 复制源图的梯度到目标图
        for (int i = 0; i < src->n_nodes; ++i) {
            dst->grads[i] = src->grads[i];
        }
    }

    // 复制源图的访问哈希表到目标图
    for (size_t i = 0; i < src->visited_hash_table.size; ++i) {
        // 如果源图的访问哈希表键存在
        if (src->visited_hash_table.keys[i]) {
            // 插入源图的访问哈希表键到目标图的访问哈希表
            ggml_hash_insert(dst->visited_hash_table, src->visited_hash_table.keys[i]);
        }
    }
}

// 复制图结构
struct ggml_cgraph * ggml_graph_dup(struct ggml_context * ctx, struct ggml_cgraph * cgraph) {
    // 创建一个新的图结构
    struct ggml_cgraph * result = ggml_new_graph_custom(ctx, cgraph->size, cgraph->grads != NULL);
    // 复制源图到新图
    ggml_graph_cpy(cgraph, result);
    // 返回新图
    return result;
}

// 重置图结构的梯度
void ggml_graph_reset(struct ggml_cgraph * cgraph) {
    // 断言图结构的梯度不为空
    GGML_ASSERT(cgraph->grads != NULL);

    // 遍历图结构的节点
    for (int i = 0; i < cgraph->n_nodes; i++) {
        // 获取节点的梯度
        struct ggml_tensor * grad = cgraph->grads[i];

        // 如果梯度存在
        if (grad) {
            // 将梯度置零
            ggml_set_zero(grad);
        }
    }
}

// 清空图结构
void ggml_graph_clear(struct ggml_cgraph * cgraph) {
    // 将叶子节点数和节点数置零
    cgraph->n_leafs = 0;
    cgraph->n_nodes = 0;
    // 将访问哈希表的键置零
    memset(cgraph->visited_hash_table.keys, 0, cgraph->visited_hash_table.size * sizeof(struct ggml_tensor *));
}

//
// 线程数据
//
// 同步通过忙循环完成
// 尝试使用自旋锁，但不确定如何正确使用 - 尝试过的方法比忙循环慢
//

#ifdef __APPLE__

//#include <os/lock.h>
//
//typedef os_unfair_lock ggml_lock_t;
//
//#define ggml_lock_init(x)    UNUSED(x)
// 定义宏 ggml_lock_destroy(x)，将其替换为 UNUSED(x)
// 定义宏 ggml_lock_lock，将其替换为 os_unfair_lock_lock
// 定义宏 ggml_lock_unlock，将其替换为 os_unfair_lock_unlock
//
// 定义宏 GGML_LOCK_INITIALIZER，初始化为 OS_UNFAIR_LOCK_INIT
typedef int ggml_lock_t;

// 定义宏 ggml_lock_init(x)，将其替换为 UNUSED(x)
// 定义宏 ggml_lock_destroy(x)，将其替换为 UNUSED(x)
// 定义宏 ggml_lock_lock(x)，将其替换为 UNUSED(x)
// 定义宏 ggml_lock_unlock(x)，将其替换为 UNUSED(x)
// 定义宏 GGML_LOCK_INITIALIZER，初始化为 0
typedef pthread_t ggml_thread_t;

// 定义宏 ggml_thread_create，将其替换为 pthread_create
// 定义宏 ggml_thread_join，将其替换为 pthread_join
#else

// 定义宏 ggml_lock_init(x)，将其替换为 UNUSED(x)
// 定义宏 ggml_lock_destroy(x)，将其替换为 UNUSED(x)
// 根据不同平台选择合适的指令来实现自旋锁
// 如果是 x86_64 平台或者 MSC 编译器且是 AMD64 架构，则使用 _mm_pause() 指令
// 否则将 ggml_lock_lock(x) 替换为 UNUSED(x)
// 定义宏 ggml_lock_unlock(x)，将其替换为 UNUSED(x)
// 定义宏 GGML_LOCK_INITIALIZER，初始化为 0
typedef pthread_t ggml_thread_t;

// 定义宏 ggml_thread_create，将其替换为 pthread_create
// 定义宏 ggml_thread_join，将其替换为 pthread_join
#endif

// 如果是 Linux 平台且不是 BIONIC 实现的 libc，则定义函数 set_numa_thread_affinity
static void set_numa_thread_affinity(int thread_n, int n_threads) {
    // 如果不支持 NUMA，则直接返回
    if (!ggml_is_numa()) {
        return;
    }

    // 计算线程应该运行在哪个 NUMA 节点上
    const int node_num = thread_n / ((n_threads + g_state.numa.n_nodes - 1) / g_state.numa.n_nodes);
    // 获取对应的 NUMA 节点
    struct ggml_numa_node * node = &g_state.numa.nodes[node_num];
    // 计算 CPU 集合的大小
    size_t setsize = CPU_ALLOC_SIZE(g_state.numa.total_cpus);

    // 分配 CPU 集合内存并初始化为 0
    cpu_set_t * cpus = CPU_ALLOC(g_state.numa.total_cpus);
    CPU_ZERO_S(setsize, cpus);
    // 将 NUMA 节点的 CPU 添加到 CPU 集合中
    for (size_t i = 0; i < node->n_cpus; ++i) {
        CPU_SET_S(node->cpus[i], setsize, cpus);
    }
    // 设置当前线程的 CPU 亲和性，将其绑定到指定的 CPU 核心上
    int rv = pthread_setaffinity_np(pthread_self(), setsize, cpus);
    // 如果设置失败，输出警告信息
    if (rv) {
            fprintf(stderr, "warning: pthread_setaffinity_np() failed: %s\n",
                    strerror(rv));
    }

    // 释放 CPU 集合的内存
    CPU_FREE(cpus);
// 清除 NUMA 线程亲和性设置
static void clear_numa_thread_affinity(void) {
    // 如果不是 NUMA 架构，直接返回
    if (!ggml_is_numa()) {
        return;
    }

    // 计算 CPU 集合的大小
    size_t setsize = CPU_ALLOC_SIZE(g_state.numa.total_cpus);

    // 分配 CPU 集合内存空间
    cpu_set_t * cpus = CPU_ALLOC(g_state.numa.total_cpus);
    // 将 CPU 集合清空
    CPU_ZERO_S(setsize, cpus);
    // 遍历所有 CPU，将其加入 CPU 集合
    for (unsigned i = 0; i < g_state.numa.total_cpus; ++i) {
        CPU_SET_S(i, setsize, cpus);
    }

    // 设置当前线程的亲和性
    int rv = pthread_setaffinity_np(pthread_self(), setsize, cpus);
    // 如果设置失败，输出警告信息
    if (rv) {
        fprintf(stderr, "warning: pthread_setaffinity_np() failed: %s\n",
            strerror(rv));
    }

    // 释放 CPU 集合内存空间
    CPU_FREE(cpus);
}
#else
// TODO: Windows etc.
// (the linux implementation may also work on BSD, someone should test)
// 设置 NUMA 线程亲和性，Windows 等系统的实现
static void set_numa_thread_affinity(int thread_n, int n_threads) { UNUSED(thread_n); UNUSED(n_threads);  }
// 清除 NUMA 线程亲和性，Windows 等系统的实现
static void clear_numa_thread_affinity(void) {}
#endif

// 共享的计算状态结构体
struct ggml_compute_state_shared {
    const struct ggml_cgraph * cgraph; // 指向计算图的指针
    const struct ggml_cplan  * cplan;  // 指向计算计划的指针

    int64_t perf_node_start_cycles;    // 节点性能统计开始时的 CPU 周期数
    int64_t perf_node_start_time_us;   // 节点性能统计开始时的时间戳（微秒）

    const int n_threads;               // 线程数

    // 同步原语
    atomic_int n_active;  // 活跃线程数
    atomic_int node_n;    // 活跃图节点
    atomic_int node_task; // 活跃图节点任务阶段

    bool (*abort_callback)(void * data); // 当为真时中止 ggml_graph_compute
    void * abort_callback_data;
};

// 计算状态结构体
struct ggml_compute_state {
    ggml_thread_t thrd; // 线程
    int ith;            // 线程索引
    struct ggml_compute_state_shared * shared; // 共享的计算状态
};

// 计算节点性能统计
static void ggml_graph_compute_perf_stats_node(struct ggml_tensor * node, const struct ggml_compute_state_shared * st) {
    // 计算当前 CPU 周期数和时间戳与节点性能统计开始时的差值
    int64_t cycles_cur  = ggml_perf_cycles()  - st->perf_node_start_cycles;
    int64_t time_us_cur = ggml_perf_time_us() - st->perf_node_start_time_us;

    // 更新节点性能统计信息
    node->perf_runs++;
    node->perf_cycles  += cycles_cur;
    node->perf_time_us += time_us_cur;
}

// 获取任务数
static int ggml_get_n_tasks(struct ggml_tensor * node, int n_threads) {
    int n_tasks = 0;

    // 断言任务数大于 0
    assert(n_tasks > 0);

    return n_tasks;
}
// 同步节点计算线程，等待其他线程完成
static void ggml_graph_compute_thread_sync_node(int * node_n, struct ggml_compute_state * state, const bool do_yield) {
    // 保存上一个节点数
    const int last_node_n = * node_n;

    // 循环等待其他线程完成
    while (true) {
        // 如果需要让出 CPU 时间片，则调用 sched_yield()
        if (do_yield) {
            sched_yield();
        }

        // 获取共享状态中的节点数，并与上一个节点数比较
        * node_n = atomic_load(&state->shared->node_n);
        if (* node_n != last_node_n) break;
    }
}

// 同步任务计算线程，等待其他线程完成
static void ggml_graph_compute_thread_sync_task(int * task_phase, struct ggml_compute_state * state, const bool do_yield) {
    // 保存上一个任务阶段
    const int last_task_phase = * task_phase;

    // 循环等待其他线程完成
    while (true) {
        // 如果需要让出 CPU 时间片，则调用 sched_yield()
        if (do_yield) {
            sched_yield();
        }

        // 获取共享状态中的任务阶段，并与上一个任务阶段比较
        * task_phase = atomic_load(&state->shared->node_task);
        if (* task_phase != last_task_phase) break;
    }
}

// 图计算线程函数
static thread_ret_t ggml_graph_compute_thread(void * data) {
    // 将传入的数据转换为 ggml_compute_state 结构体指针
    struct ggml_compute_state * state = (struct ggml_compute_state *) data;

    // 获取共享状态中的图和计划
    const struct ggml_cgraph * cgraph = state->shared->cgraph;
    const struct ggml_cplan  * cplan  = state->shared->cplan;

    // 获取共享状态中的线程数
    const int   n_threads   = state->shared->n_threads;

    // 设置 NUMA 线程亲和性
    set_numa_thread_affinity(state->ith, n_threads);

    // 初始化节点数和任务阶段
    int node_n     = -1;
    int task_phase = GGML_TASK_FINALIZE;

    // 返回成功退出
    return GGML_EXIT_SUCCESS;
}

// 创建图计划
struct ggml_cplan ggml_graph_plan(const struct ggml_cgraph * cgraph, int n_threads) {
    // 如果线程数小于等于 0，则使用默认线程数
    if (n_threads <= 0) {
        n_threads = GGML_DEFAULT_N_THREADS;
    }

    // 初始化工作大小为 0
    size_t work_size = 0;

    // 初始化图计划结构体
    struct ggml_cplan cplan;
    memset(&cplan, 0, sizeof(struct ggml_cplan));

    // 线程调度不同操作 + 估算工作缓冲区大小
}
    // 遍历计算图中的每个节点
    for (int i = 0; i < cgraph->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = cgraph->nodes[i];

        // 获取当前节点的任务数量
        const int n_tasks = ggml_get_n_tasks(node, n_threads);

        // 初始化当前节点的数据大小为0
        size_t cur = 0;

        // 根据节点的操作类型进行不同的处理
        switch (node->op) {
            // 如果是复制或者复制并增加操作
            case GGML_OP_CPY:
            case GGML_OP_DUP:
                {
                    // 如果节点的类型是量化的
                    if (ggml_is_quantized(node->type)) {
                        // 计算当前节点数据大小
                        cur = ggml_type_size(GGML_TYPE_F32) * node->ne[0] * n_tasks;
                    }
                } break;
            // 如果是加法或者加法并增加操作
            case GGML_OP_ADD:
            case GGML_OP_ADD1:
                {
                    // 如果第一个源节点的类型是量化的
                    if (ggml_is_quantized(node->src[0]->type)) {
                        // 计算当前节点数据大小
                        cur = ggml_type_size(GGML_TYPE_F32) * node->src[0]->ne[0] * n_tasks;
                    }
                } break;
            // 如果是累加操作
            case GGML_OP_ACC:
                {
                    // 如果第一个源节点的类型是量化的
                    if (ggml_is_quantized(node->src[0]->type)) {
                        // 计算当前节点数据大小
                        cur = ggml_type_size(GGML_TYPE_F32) * node->src[1]->ne[0] * n_tasks;
                    }
                } break;
            // 如果是矩阵乘法操作
            case GGML_OP_MUL_MAT:
                {
                    // 获取第一个源节点的类型对应的向量点乘类型
                    const enum ggml_type vec_dot_type = type_traits[node->src[0]->type].vec_dot_type;
#if defined(GGML_USE_CLBLAST)
                    // 如果定义了 GGML_USE_CLBLAST，并且可以使用 CLBLAST 进行矩阵乘法计算
                    if (ggml_cl_can_mul_mat(node->src[0], node->src[1], node)) {
                        // 获取 CLBLAST 执行矩阵乘法所需的工作大小
                        cur = ggml_cl_mul_mat_get_wsize(node->src[0], node->src[1], node);
                    } else
#endif
#if defined(GGML_USE_ACCELERATE) || defined(GGML_USE_OPENBLAS)
                    // 如果定义了 GGML_USE_ACCELERATE 或 GGML_USE_OPENBLAS，并且可以使用 BLAS 进行矩阵乘法计算
                    if (ggml_compute_forward_mul_mat_use_blas(node)) {
                        // 如果节点的输入数据类型不是 GGML_TYPE_F32
                        if (node->src[0]->type != GGML_TYPE_F32) {
                            // 在这里我们需要为从 src0 完全去量化的矩阵分配内存
                            // 考虑到 src0 可能会广播到 src1[2,3] 中
                            cur = ggml_type_size(GGML_TYPE_F32)
                                * node->src[0]->ne[0]*node->src[0]->ne[1]
                                * node->src[1]->ne[2]*node->src[1]->ne[3];
                        }
                    } else
    }

    // 如果工作大小大于 0
    if (work_size > 0) {
        // 调整工作大小，以适应缓存行大小和线程数
        work_size += CACHE_LINE_SIZE*(n_threads - 1);
    }

    // 设置计划的线程数和工作大小
    cplan.n_threads = n_threads;
    cplan.work_size = work_size;
    cplan.work_data = NULL;

    // 返回计划
    return cplan;
}

// 计算图形的函数
int ggml_graph_compute(struct ggml_cgraph * cgraph, struct ggml_cplan * cplan) {
    {
        // 断言计划不为空且线程数大于 0
        GGML_ASSERT(cplan);
        GGML_ASSERT(cplan->n_threads > 0);

        // 如果工作大小大于 0，则断言工作数据不为空
        if (cplan->work_size > 0) {
            GGML_ASSERT(cplan->work_data);
        }
    }

#ifdef GGML_USE_VULKAN
    // 如果定义了 GGML_USE_VULKAN
    for (int i = 0; i < cgraph->n_nodes; i++) {
        // 预先分配图形节点的 Vulkan 缓冲区
        ggml_vk_preallocate_buffers_graph(cgraph->nodes[i]);
    }
    // 预先分配 Vulkan 缓冲区
    ggml_vk_preallocate_buffers();

    for (int i = 0; i < cgraph->n_nodes; i++) {
        // 构建图形节点的 Vulkan 图形
        ggml_vk_build_graph(cgraph->nodes[i], i == cgraph->n_nodes - 1);
    }
#endif

    // 获取线程数
    const int n_threads = cplan->n_threads;
    // 初始化共享状态结构体，包括一些参数和回调函数
    struct ggml_compute_state_shared state_shared = {
        /*.cgraph                  =*/ cgraph,
        /*.cgraph_plan             =*/ cplan,
        /*.perf_node_start_cycles  =*/ 0,
        /*.perf_node_start_time_us =*/ 0,
        /*.n_threads               =*/ n_threads,
        /*.n_active                =*/ n_threads,
        /*.node_n                  =*/ -1,
        /*.node_task               =*/ GGML_TASK_FINALIZE,
        /*.abort_callback          =*/ NULL,
        /*.abort_callback_data     =*/ NULL,
    };
    // 分配存储空间给工作线程状态结构体数组
    struct ggml_compute_state * workers = alloca(sizeof(struct ggml_compute_state)*n_threads);

    // 创建线程池
    if (n_threads > 1) {
        // 为每个线程初始化状态结构体并创建线程
        for (int j = 1; j < n_threads; ++j) {
            workers[j] = (struct ggml_compute_state) {
                .thrd   = 0,
                .ith = j,
                .shared = &state_shared,
            };

            // 创建线程并执行计算函数
            const int rc = ggml_thread_create(&workers[j].thrd, NULL, ggml_graph_compute_thread, &workers[j]);
            GGML_ASSERT(rc == 0);
            UNUSED(rc);
        }
    }

    // 设置第一个线程的参数
    workers[0].ith = 0;
    workers[0].shared = &state_shared;

    // 记录性能起始时间
    const int64_t perf_start_cycles  = ggml_perf_cycles();
    const int64_t perf_start_time_us = ggml_perf_time_us();

    // 执行计算函数
    int compute_status = (size_t) ggml_graph_compute_thread(&workers[0]);

    // 清除主线程的亲和性设置
    clear_numa_thread_affinity();

    // 加入或终止线程池中的线程
    if (n_threads > 1) {
        for (int j = 1; j < n_threads; j++) {
            const int rc = ggml_thread_join(workers[j].thrd, NULL);
            GGML_ASSERT(rc == 0);
        }
    }
#ifdef GGML_USE_VULKAN
    // 如果定义了 GGML_USE_VULKAN，则清理 Vulkan 图形资源
    ggml_vk_graph_cleanup();
#endif

    // 性能统计信息（图形）
    {
        // 计算当前性能周期数和时间
        int64_t perf_cycles_cur  = ggml_perf_cycles()  - perf_start_cycles;
        int64_t perf_time_us_cur = ggml_perf_time_us() - perf_start_time_us;

        // 更新性能统计信息
        cgraph->perf_runs++;
        cgraph->perf_cycles  += perf_cycles_cur;
        cgraph->perf_time_us += perf_time_us_cur;

        // 打印性能统计信息
        GGML_PRINT_DEBUG("%s: perf (%d) - cpu = %.3f / %.3f ms, wall = %.3f / %.3f ms\n",
                __func__, cgraph->perf_runs,
                (double) perf_cycles_cur      / (double) ggml_cycles_per_ms(),
                (double) cgraph->perf_cycles  / (double) ggml_cycles_per_ms() / (double) cgraph->perf_runs,
                (double) perf_time_us_cur     / 1000.0,
                (double) cgraph->perf_time_us / 1000.0 / cgraph->perf_runs);
    }

    // 返回计算状态
    return compute_status;
}

// 使用上下文和计算图进行计算
void ggml_graph_compute_with_ctx(struct ggml_context * ctx, struct ggml_cgraph * cgraph, int n_threads) {
    // 计划计算图
    struct ggml_cplan cplan = ggml_graph_plan(cgraph, n_threads);

    // 创建工作缓冲区对象
    struct ggml_object * obj = ggml_new_object(ctx, GGML_OBJECT_WORK_BUFFER, cplan.work_size);

    // 设置工作数据
    cplan.work_data = (uint8_t *)ctx->mem_buffer + obj->offs;

    // 执行计算图计算
    ggml_graph_compute(cgraph, &cplan);
}

// 获取计算图中指定名称的张量
struct ggml_tensor * ggml_graph_get_tensor(struct ggml_cgraph * cgraph, const char * name) {
    // 遍历叶子节点
    for (int i = 0; i < cgraph->n_leafs; i++) {
        struct ggml_tensor * leaf = cgraph->leafs[i];

        // 如果找到指定名称的叶子节点，则返回该节点
        if (strcmp(leaf->name, name) == 0) {
            return leaf;
        }
    }

    // 遍历中间节点
    for (int i = 0; i < cgraph->n_nodes; i++) {
        struct ggml_tensor * node = cgraph->nodes[i];

        // 如果找到指定名称的中间节点，则返回该节点
        if (strcmp(node->name, name) == 0) {
            return node;
        }
    }

    // 如果未找到对应名称的节点，则返回空指针
    return NULL;
}

// 导出叶子节点的张量数据到文件
static void ggml_graph_export_leaf(const struct ggml_tensor * tensor, FILE * fout) {
    const int64_t * ne = tensor->ne;
    const size_t  * nb = tensor->nb;
    # 将格式化后的数据写入文件流中，包括类型、操作、维度、元素数量、边界数量、数据指针和名称
    fprintf(fout, "%-6s %-12s %8d %" PRId64 " %" PRId64 " %" PRId64 " %" PRId64 " %16zu %16zu %16zu %16zu %16p %32s\n",
            ggml_type_name(tensor->type),  # 获取张量类型的名称
            ggml_op_name  (tensor->op),    # 获取张量操作的名称
            ggml_n_dims(tensor),            # 获取张量的维度
            ne[0], ne[1], ne[2], ne[3],     # 获取张量每个维度的元素数量
            nb[0], nb[1], nb[2], nb[3],     # 获取张量每个维度的边界数量
            tensor->data,                   # 获取张量的数据指针
            tensor->name);                  # 获取张量的名称
// 导出节点信息到文件
static void ggml_graph_export_node(const struct ggml_tensor * tensor, const char * arg, FILE * fout) {
    // 获取节点的元素数量和维度大小
    const int64_t * ne = tensor->ne;
    const size_t  * nb = tensor->nb;

    // 将节点信息格式化输出到文件
    fprintf(fout, "%-6s %-6s %-12s %8d %" PRId64 " %" PRId64 " %" PRId64 " %" PRId64 " %16zu %16zu %16zu %16zu %16p %32s\n",
            arg,
            ggml_type_name(tensor->type),
            ggml_op_name  (tensor->op),
            ggml_n_dims(tensor),
            ne[0], ne[1], ne[2], ne[3],
            nb[0], nb[1], nb[2], nb[3],
            tensor->data,
            tensor->name);
}

// 导出计算图信息到文件
void ggml_graph_export(const struct ggml_cgraph * cgraph, const char * fname) {
    uint64_t size_eval = 0;

    // 计算中间结果的总大小
    // TODO: does not take into account scratch buffers !!!!
    for (int i = 0; i < cgraph->n_nodes; ++i) {
        size_eval += ggml_nbytes_pad(cgraph->nodes[i]);
    }

    // 打印
}
    {
        // 将输出流设置为标准输出
        FILE * fout = stdout;
    
        // 输出空行
        fprintf(fout, "\n");
        // 输出文件头信息：magic值
        fprintf(fout, "%-16s %8x\n", "magic",        GGML_FILE_MAGIC);
        // 输出文件头信息：version值
        fprintf(fout, "%-16s %8d\n", "version",      GGML_FILE_VERSION);
        // 输出文件头信息：叶子节点数量
        fprintf(fout, "%-16s %8d\n", "leafs",        cgraph->n_leafs);
        // 输出文件头信息：节点数量
        fprintf(fout, "%-16s %8d\n", "nodes",        cgraph->n_nodes);
        // 输出文件头信息：评估大小
        fprintf(fout, "%-16s %" PRIu64 "\n", "eval", size_eval);
    
        // 输出表头信息
        fprintf(fout, "\n");
        // 输出表头信息：各列的标题
        fprintf(fout, "%-6s %-12s %8s %8s %8s %8s %8s %16s %16s %16s %16s %16s %16s\n",
                "TYPE", "OP", "NDIMS", "NE0", "NE1", "NE2", "NE3", "NB0", "NB1", "NB2", "NB3", "DATA", "NAME");
    
        // 遍历叶子节点，导出叶子节点信息
        for (int i = 0; i < cgraph->n_leafs; ++i) {
            ggml_graph_export_leaf(cgraph->leafs[i], fout);
    
            // 断言叶子节点的操作为GGML_OP_NONE
            GGML_ASSERT(cgraph->leafs[i]->op   == GGML_OP_NONE);
            // 断言叶子节点的源节点为NULL
            GGML_ASSERT(cgraph->leafs[i]->src[0] == NULL);
            GGML_ASSERT(cgraph->leafs[i]->src[1] == NULL);
        }
    
        // 输出表头信息
        fprintf(fout, "\n");
        // 输出表头信息：各列的标题
        fprintf(fout, "%-6s %-6s %-12s %8s %8s %8s %8s %8s %16s %16s %16s %16s %8s %16s %16s\n",
                "ARG", "TYPE", "OP", "NDIMS", "NE0", "NE1", "NE2", "NE3", "NB0", "NB1", "NB2", "NB3", "NTASKS", "DATA", "NAME");
    
        // 遍历节点，导出节点信息
        for (int i = 0; i < cgraph->n_nodes; ++i) {
            ggml_graph_export_node(cgraph->nodes[i], "DST", fout);
    
            // 遍历节点的源节点，导出源节点信息
            for (int j = 0; j < GGML_MAX_SRC; ++j) {
                if (cgraph->nodes[i]->src[j]) {
                    ggml_graph_export_node(cgraph->nodes[i]->src[j], "SRC", fout);
                }
            }
    
            // 输出空行
            fprintf(fout, "\n");
        }
    
        // 输出空行
        fprintf(fout, "\n");
    }
// 导入 GGML 图形，从文件中读取数据并创建上下文
struct ggml_cgraph * ggml_graph_import(const char * fname, struct ggml_context ** ctx_data, struct ggml_context ** ctx_eval) {
    // 断言数据上下文和评估上下文为空
    assert(*ctx_data == NULL);
    assert(*ctx_eval == NULL);

    // 初始化结果为 NULL
    struct ggml_cgraph * result = NULL;

    // 初始化数据为 NULL
    struct ggml_tensor * data = NULL;

    // 读取文件内容到数据
    {
        // 打开文件
        FILE * fin = fopen(fname, "rb");
        // 如果文件打开失败，输出错误信息并返回结果
        if (!fin) {
            fprintf(stderr, "%s: failed to open %s\n", __func__, fname);
            return result;
        }

        size_t fsize = 0;

        // 获取文件大小
        fseek(fin, 0, SEEK_END);
        fsize = ftell(fin);
        fseek(fin, 0, SEEK_SET);

        // 创建数据上下文
        {
            // 计算数据上下文的开销
            const size_t overhead = 1*ggml_tensor_overhead();

            // 初始化参数
            struct ggml_init_params params = {
                .mem_size   = fsize + overhead,
                .mem_buffer = NULL,
                .no_alloc   = false,
            };

            // 创建数据上下文
            *ctx_data = ggml_init(params);

            // 如果创建数据上下文失败，输出错误信息并返回结果
            if (!*ctx_data) {
                fprintf(stderr, "%s: failed to create ggml context\n", __func__);
                fclose(fin);
                return result;
            }
        }

        // 创建一维张量
        data = ggml_new_tensor_1d(*ctx_data, GGML_TYPE_I8, fsize);

        {
            // 读取文件数据到张量
            const size_t ret = fread(data->data, sizeof(char), fsize, fin);
            // 如果读取失败，输出错误信息并返回结果
            if (ret != fsize) {
                fprintf(stderr, "%s: failed to read %s\n", __func__, fname);
                fclose(fin);
                return result;
            }
        }

        // 关闭文件
        fclose(fin);
    }

    // 填充结果
    }

    // 返回结果
    return result;
}

// 打印 GGML 图形
void ggml_graph_print(const struct ggml_cgraph * cgraph) {
    // 初始化每个操作的总性能时间为 0
    int64_t perf_total_per_op_us[GGML_OP_COUNT] = {0};

    // 打印图形信息
    GGML_PRINT("=== GRAPH ===\n");

    GGML_PRINT("n_nodes = %d\n", cgraph->n_nodes);
}
    // 遍历计算图中的每个节点
    for (int i = 0; i < cgraph->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = cgraph->nodes[i];

        // 更新每个操作的总执行时间
        perf_total_per_op_us[node->op] += MAX(1, node->perf_time_us);

        // 打印节点信息
        GGML_PRINT(" - %3d: [ %5" PRId64 ", %5" PRId64 ", %5" PRId64 "] %16s %s (%3d) cpu = %7.3f / %7.3f ms, wall = %7.3f / %7.3f ms\n",
                i,
                node->ne[0], node->ne[1], node->ne[2],
                ggml_op_name(node->op), node->is_param ? "x" : node->grad ? "g" : " ", node->perf_runs,
                (double) node->perf_cycles  / (double) ggml_cycles_per_ms(),
                (double) node->perf_cycles  / (double) ggml_cycles_per_ms() / (double) node->perf_runs,
                (double) node->perf_time_us / 1000.0,
                (double) node->perf_time_us / 1000.0 / node->perf_runs);
    }

    // 打印叶子节点信息
    GGML_PRINT("n_leafs = %d\n", cgraph->n_leafs);
    for (int i = 0; i < cgraph->n_leafs; i++) {
        // 获取当前叶子节点
        struct ggml_tensor * node = cgraph->leafs[i];

        // 打印叶子节点信息
        GGML_PRINT(" - %3d: [ %5" PRId64 ", %5" PRId64 "] %8s %16s\n",
                i,
                node->ne[0], node->ne[1],
                ggml_op_name(node->op),
                ggml_get_name(node));
    }

    // 遍历每种操作类型
    for (int i = 0; i < GGML_OP_COUNT; i++) {
        // 如果该操作类型的总执行时间为0，则跳过
        if (perf_total_per_op_us[i] == 0) {
            continue;
        }

        // 打印每种操作类型的总执行时间
        GGML_PRINT("perf_total_per_op_us[%16s] = %7.3f ms\n", ggml_op_name(i), (double) perf_total_per_op_us[i] / 1000.0);
    }

    // 打印分隔线
    GGML_PRINT("========================================\n");
// 检查节点是否属于图中
static bool ggml_graph_find(const struct ggml_cgraph * cgraph, const struct ggml_tensor * node) {
    // 如果图为空，则返回 true
    if (cgraph == NULL) {
        return true;
    }

    // 遍历图中的节点，查找是否存在目标节点
    for (int i = 0; i < cgraph->n_nodes; i++) {
        if (cgraph->nodes[i] == node) {
            return true;
        }
    }

    // 如果未找到目标节点，则返回 false
    return false;
}

// 获取节点的父节点
static struct ggml_tensor * ggml_graph_get_parent(const struct ggml_cgraph * cgraph, const struct ggml_tensor * node) {
    // 遍历图中的节点，查找目标节点的父节点
    for (int i = 0; i < cgraph->n_nodes; i++) {
        struct ggml_tensor * parent = cgraph->nodes[i];

        if (parent->grad == node) {
            return parent;
        }
    }

    // 如果未找到父节点，则返回 NULL
    return NULL;
}

// 输出节点之间的边到 DOT 文件
static void ggml_graph_dump_dot_node_edge(FILE * fp, const struct ggml_cgraph * gb, struct ggml_tensor * node, struct ggml_tensor * parent, const char * label)  {
    // 获取节点的父节点
    struct ggml_tensor * gparent = ggml_graph_get_parent(gb, node);
    struct ggml_tensor * gparent0 = ggml_graph_get_parent(gb, parent);
    // 输出节点之间的边信息到文件
    fprintf(fp, "  \"%p\":%s -> \"%p\":%s [ arrowhead = %s; style = %s; label = \"%s\"; ]\n",
            gparent0 ? (void *) gparent0 : (void *) parent,
            gparent0 ? "g" : "x",
            gparent ? (void *) gparent : (void *) node,
            gparent ? "g" : "x",
            gparent ? "empty" : "vee",
            gparent ? "dashed" : "solid",
            label);
}

// 输出叶子节点之间的边到 DOT 文件
static void ggml_graph_dump_dot_leaf_edge(FILE * fp, struct ggml_tensor * node, struct ggml_tensor * parent, const char * label)  {
    // 输出叶子节点之间的边信息到文件
    fprintf(fp, "  \"%p\":%s -> \"%p\":%s [ label = \"%s\"; ]\n",
            (void *) parent, "x",
            (void *) node, "x",
            label);
}

// 输出图的结构到 DOT 文件
void ggml_graph_dump_dot(const struct ggml_cgraph * gb, const struct ggml_cgraph * gf, const char * filename) {
    char color[16];

    // 打开文件准备写入
    FILE * fp = fopen(filename, "w");
    GGML_ASSERT(fp);

    // 输出 DOT 文件的头部信息
    fprintf(fp, "digraph G {\n");
    fprintf(fp, "  newrank = true;\n");
    fprintf(fp, "  rankdir = LR;\n");
    // 遍历图中的每个节点
    for (int i = 0; i < gb->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = gb->nodes[i];

        // 如果当前节点有父节点，则跳过
        if (ggml_graph_get_parent(gb, node) != NULL) {
            continue;
        }

        // 根据节点属性设置节点颜色
        if (node->is_param) {
            snprintf(color, sizeof(color), "yellow");
        } else if (node->grad) {
            // 如果节点有梯度并且在前向图中找到了该节点，则设置为绿色，否则设置为浅蓝色
            if (ggml_graph_find(gf, node)) {
                snprintf(color, sizeof(color), "green");
            } else {
                snprintf(color, sizeof(color), "lightblue");
            }
        } else {
            // 其他情况设置为白色
            snprintf(color, sizeof(color), "white");
        }

        // 输出节点信息到文件
        fprintf(fp, "  \"%p\" [ "
                    "style = filled; fillcolor = %s; shape = record; "
                    "label=\"",
                (void *) node, color);

        // 输出节点名称和类型
        if (strlen(node->name) > 0) {
            fprintf(fp, "%s (%s)|", node->name, ggml_type_name(node->type));
        } else {
            fprintf(fp, "(%s)|", ggml_type_name(node->type));
        }

        // 输出节点形状和操作符信息
        if (ggml_is_matrix(node)) {
            fprintf(fp, "%d [%" PRId64 ", %" PRId64 "] | <x>%s", i, node->ne[0], node->ne[1], ggml_op_symbol(node->op));
        } else {
            fprintf(fp, "%d [%" PRId64 ", %" PRId64 ", %" PRId64 "] | <x>%s", i, node->ne[0], node->ne[1], node->ne[2], ggml_op_symbol(node->op));
        }

        // 如果节点有梯度，则输出梯度操作符信息
        if (node->grad) {
            fprintf(fp, " | <g>%s\"; ]\n", ggml_op_symbol(node->grad->op));
        } else {
            fprintf(fp, "\"; ]\n");
        }
    }
    // 遍历叶子节点数组
    for (int i = 0; i < gb->n_leafs; i++) {
        // 获取当前叶子节点
        struct ggml_tensor * node = gb->leafs[i];

        // 设置节点颜色为 "pink"
        snprintf(color, sizeof(color), "pink");

        // 输出节点信息到文件
        fprintf(fp, "  \"%p\" [ "
                    "style = filled; fillcolor = %s; shape = record; "
                    "label=\"<x>",
                (void *) node, color);

        // 如果节点名称长度大于0，则输出节点名称和类型
        if (strlen(node->name) > 0) {
            fprintf(fp, "%s (%s)|", node->name, ggml_type_name(node->type));
        } else {
            fprintf(fp, "(%s)|", ggml_type_name(node->type));
        }

        // 输出节点常量信息
        fprintf(fp, "CONST %d [%" PRId64 ", %" PRId64 "]", i, node->ne[0], node->ne[1]);
        
        // 如果节点元素个数小于5，则输出节点元素值
        if (ggml_nelements(node) < 5) {
            fprintf(fp, " | (");
            for (int j = 0; j < ggml_nelements(node); j++) {
                // 根据节点类型输出不同类型的元素值
                if (node->type == GGML_TYPE_I8 || node->type == GGML_TYPE_I16 || node->type == GGML_TYPE_I32) {
                    fprintf(fp, "%d", ggml_get_i32_1d(node, j));
                }
                else if (node->type == GGML_TYPE_F32 || node->type == GGML_TYPE_F16) {
                    fprintf(fp, "%.1e", (double)ggml_get_f32_1d(node, j));
                }
                else {
                    fprintf(fp, "#");
                }
                if (j < ggml_nelements(node) - 1) {
                    fprintf(fp, ", ");
                }
            }
            fprintf(fp, ")");
        }
        fprintf(fp, "\"; ]\n");
    }

    // 遍历节点数组
    for (int i = 0; i < gb->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = gb->nodes[i];

        // 遍历节点的源节点
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            // 如果源节点存在，则输出节点之间的边
            if (node->src[j]) {
                char label[16];
                snprintf(label, sizeof(label), "src %d", j);
                ggml_graph_dump_dot_node_edge(fp, gb, node, node->src[j], label);
            }
        }
    }
    // 遍历叶子节点数组
    for (int i = 0; i < gb->n_leafs; i++) {
        // 获取当前叶子节点
        struct ggml_tensor * node = gb->leafs[i];

        // 遍历当前节点的源节点数组
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            // 如果源节点存在
            if (node->src[j]) {
                // 创建标签字符串
                char label[16];
                snprintf(label, sizeof(label), "src %d", j);
                // 将当前节点与源节点之间的关系以 DOT 格式输出到文件中
                ggml_graph_dump_dot_leaf_edge(fp, node, node->src[j], label);
            }
        }
    }

    // 输出 DOT 文件结束符
    fprintf(fp, "}\n");

    // 关闭文件指针
    fclose(fp);

    // 打印命令行操作，生成 PNG 图像并打开
    GGML_PRINT("%s: dot -Tpng %s -o %s.png && open %s.png\n", __func__, filename, filename, filename);
// 设置优化器参数，将参数数组中的值设置到对应的张量中
static void ggml_opt_set_params(int np, struct ggml_tensor * const ps[], const float * x) {
    // 初始化索引 i 为 0
    int i = 0;
    // 遍历参数数组中的张量
    for (int p = 0; p < np; ++p) {
        // 获取张量中元素的数量
        const int64_t ne = ggml_nelements(ps[p]) ;
        // 遍历张量中的每个元素，将参数数组中的值设置到张量中
        for (int64_t j = 0; j < ne; ++j) {
            ggml_set_f32_1d(ps[p], j, x[i++]);
        }
    }
}

// 获取张量中的参数值，存储到参数数组中
static void ggml_opt_get_params(int np, struct ggml_tensor * const ps[], float * x) {
    // 初始化索引 i 为 0
    int i = 0;
    // 遍历参数数组中的张量
    for (int p = 0; p < np; ++p) {
        // 获取张量中元素的数量
        const int64_t ne = ggml_nelements(ps[p]) ;
        // 遍历张量中的每个元素，将参数值存储到参数数组中
        for (int64_t j = 0; j < ne; ++j) {
            x[i++] = ggml_get_f32_1d(ps[p], j);
        }
    }
}

// 获取张量中的梯度值，存储到梯度数组中
static void ggml_opt_get_grad(int np, struct ggml_tensor * const ps[], float * g) {
    // 初始化索引 i 为 0
    int64_t i = 0;
    // 遍历参数数组中的张量
    for (int p = 0; p < np; ++p) {
        // 获取张量中元素的数量
        const int64_t ne = ggml_nelements(ps[p]) ;
        // 遍历张量中的每个元素，将梯度值存储到梯度数组中
        for (int64_t j = 0; j < ne; ++j) {
            g[i++] = ggml_get_f32_1d(ps[p]->grad, j);
        }
    }
}

// 累积梯度值，将梯度值乘以比例因子加到梯度数组中
static void ggml_opt_acc_grad(int np, struct ggml_tensor * const ps[], float * g, float scale) {
    // 初始化索引 i 为 0
    int64_t i = 0;
    // 遍历参数数组中的张量
    for (int p = 0; p < np; ++p) {
        // 获取张量中元素的数量
        const int64_t ne = ggml_nelements(ps[p]) ;
        // 遍历张量中的每个元素，将梯度值乘以比例因子加到梯度数组中
        for (int64_t j = 0; j < ne; ++j) {
            g[i++] += ggml_get_f32_1d(ps[p]->grad, j) * scale;
        }
    }
}

//
// 使用 AdamW 优化算法 - 参考: https://arxiv.org/pdf/1711.05101v3.pdf
//
// (原始 Adam 优化算法 - 参考: https://arxiv.org/pdf/1412.6980.pdf)
//

// 使用 Adam 优化算法进行优化
static enum ggml_opt_result ggml_opt_adam(
        struct ggml_context * ctx,
        struct ggml_opt_context * opt,
        struct ggml_opt_params params,
        struct ggml_tensor * f,
        struct ggml_cgraph * gf,
        struct ggml_cgraph * gb,
        ggml_opt_callback callback,
        void * callback_data) {
    // 确保输入的张量是标量
    GGML_ASSERT(ggml_is_scalar(f));

    // 这些变量将存储我们要优化的参数
    struct ggml_tensor * ps[GGML_MAX_PARAMS];

    int np = 0; // 参数数量
    int64_t nx = 0; // 参数总元素数量
    for (int i = 0; i < gf->n_nodes; ++i) {
        // 如果节点是参数节点
        if (gf->nodes[i]->is_param) {
            GGML_PRINT_DEBUG("found param %d: grad->op = %d\n", np, gf->nodes[i]->grad->op);

            GGML_ASSERT(np < GGML_MAX_PARAMS);

            // 将参数节点存储到数组中
            ps[np++] = gf->nodes[i];
            // 计算参数节点的元素数量
            nx += ggml_nelements(gf->nodes[i]);
        }
    }

    // 如果优化器的参数类型、参数数量或历史记录与当前参数不匹配，则重新初始化优化器
    if ((opt->params.type != params.type) || (opt->nx != nx) || (opt->params.past != params.past)) {
        int iter = opt->iter;
        ggml_opt_init(opt->ctx, opt, params, nx);
        opt->iter = iter;
    }

    // 常量
    float sched = params.adam.sched;
    const float alpha = params.adam.alpha;
    const float decay = params.adam.decay * alpha;
    const float beta1 = params.adam.beta1;
    const float beta2 = params.adam.beta2;
    const float eps   = params.adam.eps;
    const float gclip = params.adam.gclip;
    const int decay_min_ndim = params.adam.decay_min_ndim;
    const int n_accum = MAX(1, params.n_gradient_accumulation);
    const float accum_norm = 1.0f / (float) n_accum;

    float * g  = opt->adam.g->data;  // 梯度
    float * m  = opt->adam.m->data;  // 第一时刻估计
    float * v  = opt->adam.v->data;  // 第二时刻估计

    float * pf = params.past > 0 ? opt->adam.pf->data : NULL; // 过去的函数值

    // 创建计算计划和工作缓冲区对象
    struct ggml_cplan cplan = ggml_graph_plan(gb, params.n_threads);
    struct ggml_object * obj = ggml_new_object(ctx, GGML_OBJECT_WORK_BUFFER, cplan.work_size);
    cplan.work_data = (uint8_t *)ctx->mem_buffer + obj->offs;

    bool cancel = false; // 取消标志

    // 计算函数值
    float fx = 0;
    ggml_set_zero(opt->adam.g);
    // 遍历累积步数，从0到n_accum-1
    for (int accum_step = 0; accum_step < n_accum; ++accum_step) {
        // 如果存在回调函数
        if (callback) {
            // 调用回调函数，传入回调数据、当前步数、调度器指针和取消标志指针
            callback(callback_data, accum_step, &sched, &cancel);
            // 如果取消标志为真，则返回取消状态
            if (cancel) {
                return GGML_OPT_CANCEL;
            }
        }
        // 重置图形状态
        // ggml_graph_reset  (gf);
        // 设置梯度为1.0
        ggml_set_f32      (f->grad, 1.0f);
        // 计算图形
        ggml_graph_compute(gb, &cplan);
        // 累积梯度
        ggml_opt_acc_grad(np, ps, g, accum_norm);
        // 累积损失值
        fx += ggml_get_f32_1d(f, 0);
    }
    // 将累积损失值乘以累积归一化系数
    fx *= accum_norm;

    // 更新优化器中的先前损失值和最佳损失值
    opt->adam.fx_prev = fx;
    opt->adam.fx_best = opt->adam.fx_prev;
    // 如果存在pf数组，则更新其中的值
    if (pf) {
        pf[opt->iter % params.past] = opt->adam.fx_prev;
    }

    // 更新优化器中的损失值
    opt->loss_before = opt->adam.fx_prev;
    opt->loss_after  = opt->adam.fx_prev;

    // 如果刚初始化过
    if (opt->just_initialized) {
        // 重置未改善次数
        opt->adam.n_no_improvement = 0;
        opt->just_initialized = false;
    }

    // 获取最佳损失值、先前损失值和未改善次数的指针
    float * fx_best = &opt->adam.fx_best;
    float * fx_prev = &opt->adam.fx_prev;
    int * n_no_improvement = &opt->adam.n_no_improvement;

    // 获取迭代次数
    int iter0 = opt->iter;

    // 运行优化器

    // 返回未收敛状态
    return GGML_OPT_DID_NOT_CONVERGE;
// 结构体，用于存储 L-BFGS 算法的迭代数据
struct ggml_lbfgs_iteration_data {
    float alpha;
    float ys;
    float * s;
    float * y;
};

// 使用回溯线搜索进行优化
static enum ggml_opt_result linesearch_backtracking(
        const struct ggml_opt_params * params,
        int nx,
        float * x,
        float * fx,
        float * g,
        float * d,
        float * step,
        const float * xp,
        struct ggml_tensor * f,
        struct ggml_cgraph * gb,
        struct ggml_cplan  * cplan,
        const int np,
        struct ggml_tensor * ps[],
        bool * cancel,
        ggml_opt_callback callback,
        void * callback_data) {
    int count = 0;

    float width  = 0.0f;
    float dg     = 0.0f;
    float finit  = 0.0f;
    float dginit = 0.0f;
    float dgtest = 0.0f;

    const float dec = 0.5f;
    const float inc = 2.1f;

    const int n_accum = MAX(1, params->n_gradient_accumulation);
    const float accum_norm = 1.0f / (float) n_accum;

    // 如果步长小于等于0，则返回无效参数
    if (*step <= 0.f) {
        return GGML_LINESEARCH_INVALID_PARAMETERS;
    }

    // 计算搜索方向上的初始梯度
    ggml_vec_dot_f32(nx, &dginit, g, d);

    // 确保 d 指向一个下降方向
    if (0 < dginit) {
        return GGML_LINESEARCH_FAIL;
    }

    // 初始化局部变量
    finit = *fx;
    dgtest = params->lbfgs.ftol*dginit;

    // 不可达代码，表示此处不应该执行到
    GGML_UNREACHABLE();
}

// L-BFGS 优化算法的入口函数
static enum ggml_opt_result ggml_opt_lbfgs(
        struct ggml_context * ctx,
        struct ggml_opt_context * opt,
        struct ggml_opt_params params,
        struct ggml_tensor * f,
        struct ggml_cgraph * gf,
        struct ggml_cgraph * gb,
        ggml_opt_callback callback,
        void * callback_data) {
    // 如果线搜索类型为 GGML_LINESEARCH_BACKTRACKING_WOLFE 或 GGML_LINESEARCH_BACKTRACKING_STRONG_WOLFE，并且 wolfe 参数不在有效范围内，则返回无效的 wolfe 错误
    if (params.lbfgs.linesearch == GGML_LINESEARCH_BACKTRACKING_WOLFE ||
        params.lbfgs.linesearch == GGML_LINESEARCH_BACKTRACKING_STRONG_WOLFE) {
        if (params.lbfgs.wolfe <= params.lbfgs.ftol || 1.f <= params.lbfgs.wolfe) {
            return GGML_OPT_INVALID_WOLFE;
        }
    }

    // 获取 LBFGS 优化器的参数 m
    const int m = params.lbfgs.m;

    // 创建一个数组用于存储要优化的参数
    struct ggml_tensor * ps[GGML_MAX_PARAMS];

    int np = 0; // 参数数量
    int nx = 0; // 参数维度
    // 遍历图中的节点，找到需要优化的参数
    for (int i = 0; i < gf->n_nodes; ++i) {
        if (gf->nodes[i]->is_param) {
            GGML_PRINT_DEBUG("found param %d: grad->op = %d\n", np, gf->nodes[i]->grad->op);

            GGML_ASSERT(np < GGML_MAX_PARAMS);

            ps[np++] = gf->nodes[i];
            nx += ggml_nelements(gf->nodes[i]);
        }
    }

    // 如果优化器参数发生变化，则重新初始化优化器
    if ((opt->params.type != params.type) || (opt->nx != nx) || (opt->params.past != params.past) || (opt->params.lbfgs.m != params.lbfgs.m)) {
        int iter = opt->iter;
        ggml_opt_init(ctx, opt, params, nx);
        opt->iter = iter;
    }

    // 创建一个计划用于图的执行，并创建一个工作缓冲区对象
    struct ggml_cplan cplan = ggml_graph_plan(gb, params.n_threads);
    struct ggml_object * obj = ggml_new_object(ctx, GGML_OBJECT_WORK_BUFFER, cplan.work_size);
    cplan.work_data = (uint8_t *)ctx->mem_buffer + obj->offs;

    // 获取当前参数、上一次参数、当前梯度、上一次梯度、搜索方向等数组的指针
    float * x  = opt->lbfgs.x->data;  // 当前参数
    float * xp = opt->lbfgs.xp->data; // 上一次参数
    float * g  = opt->lbfgs.g->data;  // 当前梯度
    float * gp = opt->lbfgs.gp->data; // 上一次梯度
    float * d  = opt->lbfgs.d->data;  // 搜索方向

    // 如果参数 past 大于 0，则获取过去函数值的指针，否则为 NULL
    float * pf = params.past > 0 ? opt->lbfgs.pf->data : NULL; // 过去函数值

    // 计算梯度累积的数量和归一化系数
    const int n_accum = MAX(1, params.n_gradient_accumulation);
    const float accum_norm = 1.0f / (float) n_accum;

    float fx    = 0.0f; // 代价函数值
    float xnorm = 0.0f; // ||x||
    float gnorm = 0.0f; // ||g||

    // 从图节点中初始化参数 x
    ggml_opt_get_params(np, ps, x);

    // L-BFGS 内存
    // 获取指向 lbfgs 结构体中 alpha 数组的指针
    float * lm_alpha = opt->lbfgs.lmal->data;
    // 获取指向 lbfgs 结构体中 ys 数组的指针
    float * lm_ys    = opt->lbfgs.lmys->data;
    // 获取指向 lbfgs 结构体中 s 数组的指针
    float * lm_s     = opt->lbfgs.lms->data;
    // 获取指向 lbfgs 结构体中 y 数组的指针
    float * lm_y     = opt->lbfgs.lmy->data;

    // 初始化一个布尔变量 cancel，用于标记是否取消优化
    bool cancel = false;

    // 评估函数值及其梯度
    {
        // 设置参数
        ggml_opt_set_params(np, ps, x);

        // 初始化函数值为 0，梯度为 0
        fx = 0;
        memset(g, 0, sizeof(float)*nx);
        // 循环执行累积步骤
        for (int accum_step = 0; accum_step < n_accum; ++accum_step) {
            if (callback) {
                // LBFG-S 不支持学习率，忽略学习进度
                float sched = 0;
                callback(callback_data, accum_step, &sched, &cancel);
                if (cancel) {
                    return GGML_OPT_CANCEL;
                }
            }
            // 重置图
            // ggml_graph_reset  (gf);
            // 设置梯度为 1.0
            ggml_set_f32      (f->grad, 1.0f);
            // 计算图
            ggml_graph_compute(gb, &cplan);
            // 累积梯度
            ggml_opt_acc_grad(np, ps, g, accum_norm);
            // 累积函数值
            fx += ggml_get_f32_1d(f, 0);
        }
        // 乘以累积因子
        fx *= accum_norm;

        // 记录优化前后的损失值
        opt->loss_before = fx;
        opt->loss_after  = fx;
    }

    // 搜索方向为负梯度
    ggml_vec_neg_f32(nx, d, g);

    // 计算向量 x 和 g 的范数
    ggml_vec_norm_f32(nx, &xnorm, x);
    ggml_vec_norm_f32(nx, &gnorm, g);

    // 如果 x 的范数小于 1.0，则将 x 的范数设为 1.0
    if (xnorm < 1.0f) {
        xnorm = 1.0f;
    }

    // 如果梯度范数除以 x 范数小于等于 lbfgs 参数 eps，则认为已经优化完成
    if (gnorm/xnorm <= params.lbfgs.eps) {
        return GGML_OPT_OK;
    }

    // 如果刚初始化完成
    if (opt->just_initialized) {
        if (pf) {
            pf[0] = fx;
        }
        opt->lbfgs.fx_best = fx;

        // 初始化步长
        ggml_vec_norm_inv_f32(nx, &opt->lbfgs.step, d);
        opt->lbfgs.j                = 0;
        opt->lbfgs.k                = 1;
        opt->lbfgs.end              = 0;
        opt->lbfgs.n_no_improvement = 0;
        opt->just_initialized       = false;
    }

    // 获取指向 lbfgs 结构体中 fx_best 的指针
    float * fx_best        = &opt->lbfgs.fx_best;
    // 获取指向 lbfgs 结构体中 step 的指针
    float * step           = &opt->lbfgs.step;
    // 获取指向 lbfgs 结构体中 j 的指针
    int * j                = &opt->lbfgs.j;
    // 定义指针 k，指向 opt 结构体中 lbfgs 结构体的 k 成员
    int * k = &opt->lbfgs.k;
    // 定义指针 end，指向 opt 结构体中 lbfgs 结构体的 end 成员
    int * end = &opt->lbfgs.end;
    // 定义指针 n_no_improvement，指向 opt 结构体中 lbfgs 结构体的 n_no_improvement 成员

    int * n_no_improvement = &opt->lbfgs.n_no_improvement;

    // 初始化 ls 变量为 0
    int ls = 0;
    // 初始化 bound 变量为 0
    int bound = 0;

    // 初始化 ys 变量为 0.0
    float ys = 0.0f;
    // 初始化 yy 变量为 0.0
    float yy = 0.0f;
    // 初始化 beta 变量为 0.0
    float beta = 0.0f;

    // 初始化 it 变量为 0
    int it = 0;

    // 结束当前函数
    }

    // 指示代码不应该执行到这里，因为前面的代码块已经结束
    GGML_UNREACHABLE();
}

// 根据优化类型返回默认的优化参数
struct ggml_opt_params ggml_opt_default_params(enum ggml_opt_type type) {
    struct ggml_opt_params result;

    // 返回默认的优化参数
    return result;
}

// 初始化优化器
GGML_API void ggml_opt_init(
        struct ggml_context * ctx,
        struct ggml_opt_context * opt,
        struct ggml_opt_params params,
        int64_t nx) {
    opt->ctx = ctx;
    opt->params = params;
    opt->iter = 0;
    opt->nx = nx;
    opt->just_initialized = true;
    // 如果上下文为空
    if (opt->ctx == NULL) {
        struct ggml_init_params ctx_opt_params;
        // 如果优化类型为 ADAM
        if (opt->params.type == GGML_OPT_ADAM) {
            ctx_opt_params.mem_size = GGML_MEM_ALIGN*3 + ggml_tensor_overhead()*3 + ggml_type_size(GGML_TYPE_F32)*nx*3;
            // 如果过去的迭代次数大于0
            if (opt->params.past > 0) {
                ctx_opt_params.mem_size += GGML_MEM_ALIGN + ggml_tensor_overhead() + ggml_type_size(GGML_TYPE_F32)*opt->params.past;
            }
        } else if (opt->params.type == GGML_OPT_LBFGS) {
            ctx_opt_params.mem_size = GGML_MEM_ALIGN*9 + ggml_tensor_overhead()*9 + ggml_type_size(GGML_TYPE_F32)*(nx*5 + opt->params.lbfgs.m*2 + nx*opt->params.lbfgs.m*2);
            // 如果过去的迭代次数大于0
            if (opt->params.past > 0) {
                ctx_opt_params.mem_size += GGML_MEM_ALIGN + ggml_tensor_overhead() + ggml_type_size(GGML_TYPE_F32)*opt->params.past;
            }
        }
        ctx_opt_params.mem_buffer = NULL;
        ctx_opt_params.no_alloc   = false;

        // 初始化上下文
        opt->ctx = ggml_init(ctx_opt_params);
    }
    // 结束函数
}

// 进行优化
enum ggml_opt_result ggml_opt(
        struct ggml_context * ctx,
        struct ggml_opt_params params,
        struct ggml_tensor * f) {
    bool free_ctx = false;
    // 如果上下文为空
    if (ctx == NULL) {
        struct ggml_init_params params_ctx = {
            .mem_size   = 16*1024*1024,
            .mem_buffer = NULL,
            .no_alloc   = false,
        };

        // 初始化上下文
        ctx = ggml_init(params_ctx);
        // 如果上下文为空
        if (ctx == NULL) {
            return GGML_OPT_NO_CONTEXT;
        }

        free_ctx = true;
    }

    // 初始化结果为 OK
    enum ggml_opt_result result = GGML_OPT_OK;
    // 使用alloca函数在栈上分配内存，大小为ggml_opt_context结构体的大小，返回指向该内存的指针
    struct ggml_opt_context * opt = (struct ggml_opt_context *) alloca(sizeof(struct ggml_opt_context));

    // 初始化opt结构体，传入ctx上下文、opt指针、params参数和标志位0
    ggml_opt_init(ctx, opt, params, 0);
    
    // 调用ggml_opt_resume函数，传入ctx上下文、opt指针和f参数，返回结果
    result = ggml_opt_resume(ctx, opt, f);

    // 如果free_ctx为真，则释放ctx上下文
    if (free_ctx) {
        ggml_free(ctx);
    }

    // 返回结果
    return result;
}

// 恢复优化过程，构建前向和后向计算图
enum ggml_opt_result ggml_opt_resume(
        struct ggml_context * ctx,
        struct ggml_opt_context * opt,
        struct ggml_tensor * f) {

    // 创建前向计算图
    struct ggml_cgraph * gf = ggml_new_graph_custom(ctx, opt->params.graph_size, true);
    ggml_build_forward_expand(gf, f);

    // 复制前向计算图，创建后向计算图
    struct ggml_cgraph * gb = ggml_graph_dup(ctx, gf);
    ggml_build_backward_expand(ctx, gf, gb, true);

    // 调用 ggml_opt_resume_g 函数
    return ggml_opt_resume_g(ctx, opt, f, gf, gb, NULL, NULL);
}

// 继续优化过程，根据不同的优化类型选择相应的优化函数
enum ggml_opt_result ggml_opt_resume_g(
        struct ggml_context * ctx,
        struct ggml_opt_context * opt,
        struct ggml_tensor * f,
        struct ggml_cgraph * gf,
        struct ggml_cgraph * gb,
        ggml_opt_callback callback,
        void * callback_data) {

    // 创建前向和后向计算图
    enum ggml_opt_result result = GGML_OPT_OK;

    switch (opt->params.type) {
        case GGML_OPT_ADAM:
            {
                result = ggml_opt_adam(ctx, opt, opt->params, f, gf, gb, callback, callback_data);
            } break;
        case GGML_OPT_LBFGS:
            {
                result = ggml_opt_lbfgs(ctx, opt, opt->params, f, gf, gb, callback, callback_data);
            } break;
    }

    // 打印前向计算图
    if (opt->params.print_forward_graph) {
        ggml_graph_print   (gf);
        ggml_graph_dump_dot(gf, NULL, "opt-forward.dot");
    }

    // 打印后向计算图
    if (opt->params.print_backward_graph) {
        ggml_graph_print   (gb);
        ggml_graph_dump_dot(gb, gf, "opt-backward.dot");
    }

    return result;
}

////////////////////////////////////////////////////////////////////////////////

// 初始化量化类型
void ggml_quantize_init(enum ggml_type type) {
    ggml_critical_section_start();

    switch (type) {
        case GGML_TYPE_IQ2_XXS: iq2xs_init_impl(256); break;
        case GGML_TYPE_IQ2_XS:  iq2xs_init_impl(512); break;
        case GGML_TYPE_IQ3_XXS: iq3xs_init_impl(256); break;
        default: // 无操作
            break;
    }

    ggml_critical_section_end();
}
void ggml_quantize_free(void) {
    // 进入临界区
    ggml_critical_section_start();

    // 释放 256 维度的量化内存
    iq2xs_free_impl(256);
    // 释放 512 维度的量化内存
    iq2xs_free_impl(512);

    // 离开临界区
    ggml_critical_section_end();
}

size_t ggml_quantize_q4_0(const float * src, void * dst, int n, int k, int64_t * hist) {
    // 确保 k 是 QK4_0 的倍数
    assert(k % QK4_0 == 0);
    const int nb = k / QK4_0;

    // 对每个块进行量化
    for (int b = 0; b < n; b += k) {
        // 将目标指针转换为 block_q4_0 类型
        block_q4_0 * restrict y = (block_q4_0 *) dst + b/QK4_0;

        // 对每行进行 QK4_0 量化
        quantize_row_q4_0_reference(src + b, y, k);

        // 更新直方图
        for (int i = 0; i < nb; i++) {
            for (int j = 0; j < QK4_0; j += 2) {
                const uint8_t vi0 = y[i].qs[j/2] & 0x0F;
                const uint8_t vi1 = y[i].qs[j/2] >> 4;

                hist[vi0]++;
                hist[vi1]++;
            }
        }
    }

    return (n/QK4_0*sizeof(block_q4_0));
}

size_t ggml_quantize_q4_1(const float * src, void * dst, int n, int k, int64_t * hist) {
    // 确保 k 是 QK4_1 的倍数
    assert(k % QK4_1 == 0);
    const int nb = k / QK4_1;

    // 对每个块进行量化
    for (int b = 0; b < n; b += k) {
        // 将目标指针转换为 block_q4_1 类型
        block_q4_1 * restrict y = (block_q4_1 *) dst + b/QK4_1;

        // 对每行进行 QK4_1 量化
        quantize_row_q4_1_reference(src + b, y, k);

        // 更新直方图
        for (int i = 0; i < nb; i++) {
            for (int j = 0; j < QK4_1; j += 2) {
                const uint8_t vi0 = y[i].qs[j/2] & 0x0F;
                const uint8_t vi1 = y[i].qs[j/2] >> 4;

                hist[vi0]++;
                hist[vi1]++;
            }
        }
    }

    return (n/QK4_1*sizeof(block_q4_1));
}

size_t ggml_quantize_q5_0(const float * src, void * dst, int n, int k, int64_t * hist) {
    // 确保 k 是 QK5_0 的倍数
    assert(k % QK5_0 == 0);
    const int nb = k / QK5_0;
    // 对输入数据进行分块处理，每次处理 k 个元素
    for (int b = 0; b < n; b += k) {
        // 将目标地址转换为 block_q5_0 类型指针
        block_q5_0 * restrict y = (block_q5_0 *)dst + b/QK5_0;

        // 对输入数据进行量化处理，存储到 y 中
        quantize_row_q5_0_reference(src + b, y, k);

        // 遍历每个 block 中的元素
        for (int i = 0; i < nb; i++) {
            uint32_t qh;
            // 从 y[i].qh 中复制数据到 qh
            memcpy(&qh, &y[i].qh, sizeof(qh));

            // 遍历每个 block 中的元素
            for (int j = 0; j < QK5_0; j += 2) {
                // 计算 vh0 和 vh1
                const uint8_t vh0 = ((qh & (1u << (j/2 + 0 ))) >> (j/2 + 0 )) << 4;
                const uint8_t vh1 = ((qh & (1u << (j/2 + 16))) >> (j/2 + 12));

                // 将数据转换为 16 个 bin
                const uint8_t vi0 = ((y[i].qs[j/2] & 0x0F) | vh0) / 2;
                const uint8_t vi1 = ((y[i].qs[j/2] >>   4) | vh1) / 2;

                // 更新直方图中 vi0 和 vi1 的计数
                hist[vi0]++;
                hist[vi1]++;
            }
        }
    }

    // 返回处理的数据量
    return (n/QK5_0*sizeof(block_q5_0));
}

size_t ggml_quantize_q5_1(const float * src, void * dst, int n, int k, int64_t * hist) {
    // 确保 k 能被 QK5_1 整除
    assert(k % QK5_1 == 0);
    // 计算每个块的数量
    const int nb = k / QK5_1;

    // 遍历每个块
    for (int b = 0; b < n; b += k) {
        // 将目标指针转换为 block_q5_1 类型
        block_q5_1 * restrict y = (block_q5_1 *)dst + b/QK5_1;

        // 对当前块进行量化
        quantize_row_q5_1_reference(src + b, y, k);

        // 遍历每个块中的元素
        for (int i = 0; i < nb; i++) {
            uint32_t qh;
            // 从 y[i].qh 复制数据到 qh
            memcpy(&qh, &y[i].qh, sizeof(qh));

            // 遍历每个块中的元素
            for (int j = 0; j < QK5_1; j += 2) {
                // 计算 vh0 和 vh1
                const uint8_t vh0 = ((qh & (1u << (j/2 + 0 ))) >> (j/2 + 0 )) << 4;
                const uint8_t vh1 = ((qh & (1u << (j/2 + 16))) >> (j/2 + 12));

                // 将数据转换为 16 个 bin
                const uint8_t vi0 = ((y[i].qs[j/2] & 0x0F) | vh0) / 2;
                const uint8_t vi1 = ((y[i].qs[j/2] >>   4) | vh1) / 2;

                // 更新直方图
                hist[vi0]++;
                hist[vi1]++;
            }
        }
    }

    // 返回结果大小
    return (n/QK5_1*sizeof(block_q5_1));
}

size_t ggml_quantize_q8_0(const float * src, void * dst, int n, int k, int64_t * hist) {
    // 确保 k 能被 QK8_0 整除
    assert(k % QK8_0 == 0);
    // 计算每个块的数量
    const int nb = k / QK8_0;

    // 遍历每个块
    for (int b = 0; b < n; b += k) {
        // 将目标指针转换为 block_q8_0 类型
        block_q8_0 * restrict y = (block_q8_0 *)dst + b/QK8_0;

        // 对当前块进行量化
        quantize_row_q8_0_reference(src + b, y, k);

        // 遍历每个块中的元素
        for (int i = 0; i < nb; i++) {
            // 遍历每个块中的元素
            for (int j = 0; j < QK8_0; ++j) {
                // 获取 vi
                const int8_t vi = y[i].qs[j];

                // 更新直方图
                hist[vi/16 + 8]++;
            }
        }
    }

    // 返回结果大小
    return (n/QK8_0*sizeof(block_q8_0));
}

bool ggml_quantize_requires_imatrix(enum ggml_type type) {
    // 判断是否需要 imatrix
    return
        type == GGML_TYPE_IQ2_XXS ||
        type == GGML_TYPE_IQ2_XS;
}

size_t ggml_quantize_chunk(enum ggml_type type, const float * src, void * dst, int start,
        int nrows, int n_per_row, int64_t * hist, const float * imatrix) {
    ggml_quantize_init(type); // 如果已经初始化，则不执行任何操作
    size_t result = 0;
    int n = nrows * n_per_row;
    }
    return result;
}
// 定义一个结构体 gguf_str，包含一个 uint64_t 类型的 n 和一个 char 指针类型的 data
struct gguf_str {
    uint64_t n;  // GGUFv2
    char * data;
};

// 定义一个静态常量数组 GGUF_TYPE_SIZE，包含 GGUF_TYPE_COUNT 个元素
static const size_t GGUF_TYPE_SIZE[GGUF_TYPE_COUNT] = {
    [GGUF_TYPE_UINT8]   = sizeof(uint8_t),
    [GGUF_TYPE_INT8]    = sizeof(int8_t),
    [GGUF_TYPE_UINT16]  = sizeof(uint16_t),
    [GGUF_TYPE_INT16]   = sizeof(int16_t),
    [GGUF_TYPE_UINT32]  = sizeof(uint32_t),
    [GGUF_TYPE_INT32]   = sizeof(int32_t),
    [GGUF_TYPE_FLOAT32] = sizeof(float),
    [GGUF_TYPE_BOOL]    = sizeof(bool),
    [GGUF_TYPE_STRING]  = sizeof(struct gguf_str),
    [GGUF_TYPE_UINT64]  = sizeof(uint64_t),
    [GGUF_TYPE_INT64]   = sizeof(int64_t),
    [GGUF_TYPE_FLOAT64] = sizeof(double),
    [GGUF_TYPE_ARRAY]   = 0, // undefined
};
// 断言 GGUF_TYPE_COUNT 等于 13，如果不等于则输出错误信息
static_assert(GGUF_TYPE_COUNT == 13, "GGUF_TYPE_COUNT != 13");

// 定义一个静态常量数组 GGUF_TYPE_NAME，包含 GGUF_TYPE_COUNT 个元素
static const char * GGUF_TYPE_NAME[GGUF_TYPE_COUNT] = {
    [GGUF_TYPE_UINT8]   = "u8",
    [GGUF_TYPE_INT8]    = "i8",
    [GGUF_TYPE_UINT16]  = "u16",
    [GGUF_TYPE_INT16]   = "i16",
    [GGUF_TYPE_UINT32]  = "u32",
    [GGUF_TYPE_INT32]   = "i32",
    [GGUF_TYPE_FLOAT32] = "f32",
    [GGUF_TYPE_BOOL]    = "bool",
    [GGUF_TYPE_STRING]  = "str",
    [GGUF_TYPE_ARRAY]   = "arr",
    [GGUF_TYPE_UINT64]  = "u64",
    [GGUF_TYPE_INT64]   = "i64",
    [GGUF_TYPE_FLOAT64] = "f64",
};
// 断言 GGUF_TYPE_COUNT 等于 13，如果不等于则输出错误信息
static_assert(GGUF_TYPE_COUNT == 13, "GGUF_TYPE_COUNT != 13");

// 定义一个联合体 gguf_value，包含不同类型的数据成员
union gguf_value {
    uint8_t  uint8;
    int8_t   int8;
    uint16_t uint16;
    int16_t  int16;
    uint32_t uint32;
    int32_t  int32;
    float    float32;
    uint64_t uint64;
    int64_t  int64;
    double   float64;
    bool     bool_;

    struct gguf_str str;

    // 匿名结构体，包含 gguf_type 类型的 type，uint64_t 类型的 n 和 void 指针类型的 data
    struct {
        enum gguf_type type;

        uint64_t n;  // GGUFv2
        void * data;
    } arr;
};

// 定义一个结构体 gguf_kv，包含一个 gguf_str 类型的 key，gguf_type 类型的 type 和 gguf_value 类型的 value
struct gguf_kv {
    struct gguf_str key;

    enum  gguf_type  type;
    union gguf_value value;
};

// 定义一个结构体 gguf_header，包含一个长度为 4 的 char 数组 magic，uint32_t 类型的 version，uint64_t 类型的 n_tensors 和 n_kv
struct gguf_header {
    char magic[4];

    uint32_t version;
    uint64_t n_tensors; // GGUFv2
    uint64_t n_kv;      // GGUFv2
// 结构体定义，包含名称、维度、元素数量、数据类型、偏移量、数据和大小等信息
struct gguf_tensor_info {
    struct gguf_str name; // 张量名称

    uint32_t n_dims; // 维度数量
    uint64_t ne[GGML_MAX_DIMS]; // 每个维度的元素数量

    enum ggml_type type; // 数据类型

    uint64_t offset; // 数据在`data`中的偏移量，必须是`ALIGNMENT`的倍数

    // 用于写入 API
    const void * data; // 数据指针
    size_t size; // 数据大小
};

// 上下文结构体，包含头部信息、键值对、张量信息、对齐方式、数据偏移量、数据大小和数据等信息
struct gguf_context {
    struct gguf_header header; // 头部信息

    struct gguf_kv          * kv; // 键值对
    struct gguf_tensor_info * infos; // 张量信息

    size_t alignment; // 对齐方式
    size_t offset;    // 数据在文件开始处的偏移量
    size_t size;      // 数据的大小（字节数）

    //uint8_t * padding;
    void * data; // 数据指针
};

// 根据数据类型返回数据大小
static size_t gguf_type_size(enum gguf_type type) {
    GGML_ASSERT(0 <= type && type < GGUF_TYPE_COUNT);
    return GGUF_TYPE_SIZE[type];
}

// 对张量信息进行检查，确保维度、数据类型和元素数量的合法性
static void gguf_tensor_info_sanitize(struct gguf_tensor_info * info) {
    GGML_ASSERT(info->n_dims <= GGML_MAX_DIMS);
    GGML_ASSERT(0 <= info->type && info->type < GGML_TYPE_COUNT);

    for (uint32_t i = 0; i < info->n_dims; ++i) {
        GGML_ASSERT(info->ne[i] > 0);
    }

    // 防止总元素数量溢出
    GGML_ASSERT(INT64_MAX/info->ne[1] > info->ne[0]);
    GGML_ASSERT(INT64_MAX/info->ne[2] > info->ne[0]*info->ne[1]);
    GGML_ASSERT(INT64_MAX/info->ne[3] > info->ne[0]*info->ne[1]*info->ne[2]);
}

// 从文件中读取指定大小的数据到目标地址，并更新偏移量
static bool gguf_fread_el(FILE * file, void * dst, size_t size, size_t * offset) {
    const size_t n = fread(dst, 1, size, file);
    *offset += n;
    return n == size;
}

// 从文件中读取字符串数据到结构体中，并更新偏移量
static bool gguf_fread_str(FILE * file, struct gguf_str * p, size_t * offset) {
    p->n    = 0;
    p->data = NULL;

    bool ok = true;

    ok = ok && gguf_fread_el(file, &p->n, sizeof(p->n), offset);

    // 如果字符串长度无效，则提前退出，防止整数溢出
    if (p->n == SIZE_MAX) {
        fprintf(stderr, "%s: invalid string length (%" PRIu64 ")\n", __func__, p->n);
        return false;
    }

    p->data = GGML_CALLOC(p->n + 1, 1);

    ok = ok && gguf_fread_el(file,  p->data, p->n, offset);

    return ok;
}
// 初始化一个空的 gguf_context 结构体指针
struct gguf_context * gguf_init_empty(void) {
    // 分配内存空间给 gguf_context 结构体指针
    struct gguf_context * ctx = GGML_ALIGNED_MALLOC(sizeof(struct gguf_context));

    // 将 GGUF_MAGIC 的内容复制到 ctx->header.magic 中
    memcpy(ctx->header.magic, GGUF_MAGIC, sizeof(ctx->header.magic));
    // 设置 ctx->header.version 为 GGUF_VERSION
    ctx->header.version   = GGUF_VERSION;
    // 设置 ctx->header.n_tensors 为 0
    ctx->header.n_tensors = 0;
    // 设置 ctx->header.n_kv 为 0
    ctx->header.n_kv      = 0;

    // 初始化 ctx->kv 和 ctx->infos 为 NULL
    ctx->kv    = NULL;
    ctx->infos = NULL;

    // 设置 ctx->alignment 为 GGUF_DEFAULT_ALIGNMENT
    ctx->alignment = GGUF_DEFAULT_ALIGNMENT;
    // 设置 ctx->offset 为 0
    ctx->offset    = 0;
    // 设置 ctx->size 为 0
    ctx->size      = 0;

    // 初始化 ctx->data 为 NULL
    ctx->data = NULL;

    // 返回初始化后的 gguf_context 结构体指针
    return ctx;
}

// 从文件中初始化 gguf_context 结构体指针
struct gguf_context * gguf_init_from_file(const char * fname, struct gguf_init_params params) {
    // 打开文件
    FILE * file = fopen(fname, "rb");
    // 如果文件打开失败，返回 NULL
    if (!file) {
        return NULL;
    }

    // 从文件开始的偏移量
    size_t offset = 0;

    // 用于存储魔数的数组
    char magic[4];

    // 在分配内存之前检查魔数
    {
        // 从文件中读取魔数
        gguf_fread_el(file, &magic, sizeof(magic), &offset);

        // 检查魔数是否正确
        for (uint32_t i = 0; i < sizeof(magic); i++) {
            if (magic[i] != GGUF_MAGIC[i]) {
                // 如果魔数不正确，输出错误信息，关闭文件，返回 NULL
                fprintf(stderr, "%s: invalid magic characters '%c%c%c%c'\n", __func__, magic[0], magic[1], magic[2], magic[3]);
                fclose(file);
                return NULL;
            }
        }
    }

    // 标记是否初始化成功
    bool ok = true;

    // 分配内存给 gguf_context 结构体指针
    struct gguf_context * ctx = GGML_ALIGNED_MALLOC(sizeof(struct gguf_context));

    // 读取头部信息
    {
        // 将 magic 字符串的前 4 个字符复制到 ctx->header.magic 中
        strncpy(ctx->header.magic, magic, 4);

        // 初始化 ctx->kv、ctx->infos、ctx->data 为 NULL
        ctx->kv    = NULL;
        ctx->infos = NULL;
        ctx->data  = NULL;

        // 从文件中读取 ctx->header.version、ctx->header.n_tensors、ctx->header.n_kv 的值，并更新 offset
        ok = ok && gguf_fread_el(file, &ctx->header.version,   sizeof(ctx->header.version),   &offset);
        ok = ok && gguf_fread_el(file, &ctx->header.n_tensors, sizeof(ctx->header.n_tensors), &offset);
        ok = ok && gguf_fread_el(file, &ctx->header.n_kv,      sizeof(ctx->header.n_kv),      &offset);

        // 如果版本号为 1，则输出错误信息并返回 NULL
        if (ctx->header.version == 1) {
            fprintf(stderr, "%s: GGUFv1 is no longer supported. please use a more up-to-date version\n", __func__);
            fclose(file);
            gguf_free(ctx);
            return NULL;
        }

        // 检查以防止整数/缓冲区溢出

        // 检查 ctx->header.n_tensors 是否小于 (SIZE_MAX/2)/sizeof(struct gguf_tensor_info)
        ok = ok && (ctx->header.n_tensors < (SIZE_MAX/2)/sizeof(struct gguf_tensor_info));
        // 检查 ctx->header.n_tensors 是否小于 (SIZE_MAX/2)/ggml_tensor_overhead()
        ok = ok && (ctx->header.n_tensors < (SIZE_MAX/2)/ggml_tensor_overhead());
        // 检查 ctx->header.n_kv 是否小于 (SIZE_MAX/2)/sizeof(struct gguf_kv)
        ok = ok && (ctx->header.n_kv      < (SIZE_MAX/2)/sizeof(struct gguf_kv);

        // 如果检查失败，则输出错误信息并返回 NULL
        if (!ok) {
            fprintf(stderr, "%s: failed to read header\n", __func__);
            fclose(file);
            gguf_free(ctx);
            return NULL;
        }
    }

    // 读取 kv 对
    }

    // 读取张量信息
    {
        // 为 ctx->infos 分配内存，大小为 ctx->header.n_tensors 乘以每个 tensor 信息结构体的大小
        ctx->infos = GGML_MALLOC(ctx->header.n_tensors * sizeof(struct gguf_tensor_info));
    
        // 遍历每个 tensor 的信息
        for (uint64_t i = 0; i < ctx->header.n_tensors; ++i) {
            // 获取当前 tensor 的信息结构体指针
            struct gguf_tensor_info * info = &ctx->infos[i];
    
            // 初始化 tensor 的每个维度为 1
            for (int j = 0; j < GGML_MAX_DIMS; ++j) {
                info->ne[j] = 1;
            }
    
            // 读取 tensor 的名称
            ok = ok && gguf_fread_str(file, &info->name, &offset);
            // 读取 tensor 的维度
            ok = ok && gguf_fread_el(file, &info->n_dims, sizeof(info->n_dims), &offset);
    
            // 检查 tensor 的维度是否小于等于最大维度 GGML_MAX_DIMS
            ok = ok && (info->n_dims <= GGML_MAX_DIMS);
    
            // 读取 tensor 的每个维度大小
            for (uint32_t j = 0; j < info->n_dims; ++j) {
                ok = ok && gguf_fread_el(file, &info->ne[j], sizeof(info->ne[j]), &offset);
            }
    
            // 读取 tensor 的数据类型
            ok = ok && gguf_fread_el(file, &info->type, sizeof(info->type), &offset);
            // 读取 tensor 的偏移量
            ok = ok && gguf_fread_el(file, &info->offset, sizeof(info->offset), &offset);
    
            // 对 tensor 的信息进行清理和验证
            gguf_tensor_info_sanitize(info);
    
            // 如果读取失败，则输出错误信息，关闭文件，释放内存并返回空指针
            if (!ok) {
                fprintf(stderr, "%s: failed to read tensor info\n", __func__);
                fclose(file);
                gguf_free(ctx);
                return NULL;
            }
        }
    }
    
    // 设置 ctx 的对齐方式为默认对齐值 GGUF_DEFAULT_ALIGNMENT
    ctx->alignment = GGUF_DEFAULT_ALIGNMENT;
    
    // 查找是否存在 "general.alignment" 键，如果存在则设置 ctx 的对齐方式为对应的值
    int alignment_idx = gguf_find_key(ctx, "general.alignment");
    if (alignment_idx != -1) {
        ctx->alignment = gguf_get_val_u32(ctx, alignment_idx);
    }
    
    // 计算偏移量的填充量，以确保数据段对齐
    {
        const size_t offset_pad = offset % ctx->alignment;
    
        // 如果填充量不为 0，则调整偏移量并移动文件指针
        if (offset_pad != 0) {
            offset += ctx->alignment - offset_pad;
            fseek(file, offset, SEEK_SET);
        }
    }
    
    // 存储当前文件偏移量，即数据段开始位置
    ctx->offset = offset;
    
    // 计算数据段的总大小，考虑对齐
    }
    {
        // 初始化上下文中的大小为0
        ctx->size = 0;
        // 遍历上下文中的张量信息
        for (uint64_t i = 0; i < ctx->header.n_tensors; ++i) {
            // 获取当前张量信息
            struct gguf_tensor_info * info = &ctx->infos[i];

            // 计算当前张量的元素个数
            const int64_t ne =
                (int64_t) info->ne[0] *
                (int64_t) info->ne[1] *
                (int64_t) info->ne[2] *
                (int64_t) info->ne[3];

            // 检查当前张量的元素个数是否是块大小的倍数
            if (ne % ggml_blck_size(info->type) != 0) {
                // 打印错误信息并返回空值
                fprintf(stderr, "%s: tensor '%s' of type %d (%s) number of elements (%" PRId64 ") is not a multiple of block size (%d)\n",
                        __func__, info->name.data, (int)info->type, ggml_type_name(info->type), ne, ggml_blck_size(info->type));
                fclose(file);
                gguf_free(ctx);
                return NULL;
            }

            // 计算当前张量数据的大小
            const size_t size_cur = ggml_row_size(info->type, ne);

            // 更新上下文中的大小，考虑对齐
            ctx->size += GGML_PAD(size_cur, ctx->alignment);
        }
    }

    // 仅在请求时加载张量数据

    // 关闭文件
    fclose(file);

    // 返回上下文
    return ctx;
}
// 释放 gguf_context 结构体及其内存
void gguf_free(struct gguf_context * ctx) {
    // 如果 ctx 为空指针，则直接返回
    if (ctx == NULL) {
        return;
    }

    // 如果 ctx 中的 kv 不为空
    if (ctx->kv) {
        // 释放字符串内存 - 不太好..
        // 遍历 kv 数组
        for (uint64_t i = 0; i < ctx->header.n_kv; ++i) {
            // 获取当前 kv 结构体
            struct gguf_kv * kv = &ctx->kv[i];

            // 如果 key.data 不为空，则释放其内存
            if (kv->key.data) {
                GGML_FREE(kv->key.data);
            }

            // 如果类型为 GGUF_TYPE_STRING
            if (kv->type == GGUF_TYPE_STRING) {
                // 如果 value.str.data 不为空，则释放其内存
                if (kv->value.str.data) {
                    GGML_FREE(kv->value.str.data);
                }
            }

            // 如果类型为 GGUF_TYPE_ARRAY
            if (kv->type == GGUF_TYPE_ARRAY) {
                // 如果 value.arr.data 不为空
                if (kv->value.arr.data) {
                    // 如果数组元素类型为 GGUF_TYPE_STRING
                    if (kv->value.arr.type == GGUF_TYPE_STRING) {
                        // 遍历数组元素
                        for (uint64_t j = 0; j < kv->value.arr.n; ++j) {
                            // 获取当前字符串结构体
                            struct gguf_str * str = &((struct gguf_str *) kv->value.arr.data)[j];
                            // 如果字符串数据不为空，则释放其内存
                            if (str->data) {
                                GGML_FREE(str->data);
                            }
                        }
                    }
                    // 释放数组数据内存
                    GGML_FREE(kv->value.arr.data);
                }
            }
        }

        // 释放 kv 数组内存
        GGML_FREE(ctx->kv);
    }

    // 如果 ctx 中的 infos 不为空
    if (ctx->infos) {
        // 遍历 infos 数组
        for (uint64_t i = 0; i < ctx->header.n_tensors; ++i) {
            // 获取当前 tensor_info 结构体
            struct gguf_tensor_info * info = &ctx->infos[i];

            // 如果 name.data 不为空，则释放其内存
            if (info->name.data) {
                GGML_FREE(info->name.data);
            }
        }

        // 释放 infos 数组内存
        GGML_FREE(ctx->infos);
    }

    // 释放 ctx 结构体内存
    GGML_ALIGNED_FREE(ctx);
}

// 返回指定 gguf_type 的名称
const char * gguf_type_name(enum gguf_type type) {
    return GGUF_TYPE_NAME[type];
}

// 获取 gguf_context 结构体中的版本号
int gguf_get_version(const struct gguf_context * ctx) {
    return ctx->header.version;
}

// 获取 gguf_context 结构体中的对齐方式
size_t gguf_get_alignment(const struct gguf_context * ctx) {
    return ctx->alignment;
}

// 获取 gguf_context 结构体中的数据偏移量
size_t gguf_get_data_offset(const struct gguf_context * ctx) {
    return ctx->offset;
}

// 获取 gguf_context 结构体中的数据指针
void * gguf_get_data(const struct gguf_context * ctx) {
    return ctx->data;
}

// 获取 gguf_context 结构体中的 kv 数量
int gguf_get_n_kv(const struct gguf_context * ctx) {
    return ctx->header.n_kv;
}
}

// 在给定的 gguf_context 中查找指定 key 的索引，如果找不到则返回 -1
int gguf_find_key(const struct gguf_context * ctx, const char * key) {
    // 初始化 keyfound 为 -1，表示未找到
    int keyfound = -1;

    // 获取 gguf_context 中键值对的数量
    const int n_kv = gguf_get_n_kv(ctx);

    // 遍历键值对数组，查找指定 key
    for (int i = 0; i < n_kv; ++i) {
        // 如果找到指定 key，则更新 keyfound 的值为当前索引 i，并跳出循环
        if (strcmp(key, gguf_get_key(ctx, i)) == 0) {
            keyfound = i;
            break;
        }
    }

    // 返回找到的 key 的索引
    return keyfound;
}

// 获取指定 key_id 对应的 key
const char * gguf_get_key(const struct gguf_context * ctx, int key_id) {
    // 断言 key_id 的合法性
    GGML_ASSERT(key_id >= 0 && key_id < gguf_get_n_kv(ctx));
    // 返回指定 key_id 对应的 key
    return ctx->kv[key_id].key.data;
}

// 获取指定 key_id 对应的键值对类型
enum gguf_type gguf_get_kv_type(const struct gguf_context * ctx, int key_id) {
    // 断言 key_id 的合法性
    GGML_ASSERT(key_id >= 0 && key_id < gguf_get_n_kv(ctx));
    // 返回指定 key_id 对应的键值对类型
    return ctx->kv[key_id].type;
}

// 获取指定 key_id 对应的数组类型
enum gguf_type gguf_get_arr_type(const struct gguf_context * ctx, int key_id) {
    // 断言 key_id 的合法性
    GGML_ASSERT(key_id >= 0 && key_id < gguf_get_n_kv(ctx));
    // 断言指定 key_id 对应的类型为数组类型
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_ARRAY);
    // 返回指定 key_id 对应的数组类型
    return ctx->kv[key_id].value.arr.type;
}

// 获取指定 key_id 对应的数组数据
const void * gguf_get_arr_data(const struct gguf_context * ctx, int key_id) {
    // 断言 key_id 的合法性
    GGML_ASSERT(key_id >= 0 && key_id < gguf_get_n_kv(ctx));
    // 断言指定 key_id 对应的类型为数组类型
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_ARRAY);
    // 返回指定 key_id 对应的数组数据
    return ctx->kv[key_id].value.arr.data;
}

// 获取指定 key_id 对应的数组中第 i 个字符串数据
const char * gguf_get_arr_str(const struct gguf_context * ctx, int key_id, int i) {
    // 断言 key_id 的合法性
    GGML_ASSERT(key_id >= 0 && key_id < gguf_get_n_kv(ctx));
    // 断言指定 key_id 对应的类型为数组类型
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_ARRAY);
    // 获取指定 key_id 对应的键值对
    struct gguf_kv * kv = &ctx->kv[key_id];
    // 获取数组中第 i 个字符串数据
    struct gguf_str * str = &((struct gguf_str *) kv->value.arr.data)[i];
    return str->data;
}

// 获取指定 key_id 对应的数组的元素数量
int gguf_get_arr_n(const struct gguf_context * ctx, int key_id) {
    // 断言 key_id 的合法性
    GGML_ASSERT(key_id >= 0 && key_id < gguf_get_n_kv(ctx));
    // 断言指定 key_id 对应的类型为数组类型
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_ARRAY);
    // 返回指定 key_id 对应的数组的元素数量
    return ctx->kv[key_id].value.arr.n;
}

// 获取指定 key_id 对应的 uint8_t 类型的值
uint8_t gguf_get_val_u8(const struct gguf_context * ctx, int key_id) {
    // 断言 key_id 的合法性
    GGML_ASSERT(key_id >= 0 && key_id < gguf_get_n_kv(ctx));
    // 断言指定 key_id 对应的类型为 uint8_t 类型
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_UINT8);
    // 返回指定 key_id 对应的 uint8_t 类型的值
    return ctx->kv[key_id].value.uint8;
}
// 获取 int8_t 类型的值
int8_t gguf_get_val_i8(const struct gguf_context * ctx, int key_id) {
    // 断言 key_id 大于等于 0 且小于键值对数量
    GGML_ASSERT(key_id >= 0 && key_id < gguf_get_n_kv(ctx));
    // 断言键值对的类型为 GGUF_TYPE_INT8
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_INT8);
    // 返回键值对中 int8_t 类型的值
    return ctx->kv[key_id].value.int8;
}

// 获取 uint16_t 类型的值
uint16_t gguf_get_val_u16(const struct gguf_context * ctx, int key_id) {
    // 断言 key_id 大于等于 0 且小于键值对数量
    GGML_ASSERT(key_id >= 0 && key_id < gguf_get_n_kv(ctx));
    // 断言键值对的类型为 GGUF_TYPE_UINT16
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_UINT16);
    // 返回键值对中 uint16_t 类型的值
    return ctx->kv[key_id].value.uint16;
}

// 获取 int16_t 类型的值
int16_t gguf_get_val_i16(const struct gguf_context * ctx, int key_id) {
    // 断言 key_id 大于等于 0 且小于键值对数量
    GGML_ASSERT(key_id >= 0 && key_id < gguf_get_n_kv(ctx));
    // 断言键值对的类型为 GGUF_TYPE_INT16
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_INT16);
    // 返回键值对中 int16_t 类型的值
    return ctx->kv[key_id].value.int16;
}

// 获取 uint32_t 类型的值
uint32_t gguf_get_val_u32(const struct gguf_context * ctx, int key_id) {
    // 断言 key_id 大于等于 0 且小于键值对数量
    GGML_ASSERT(key_id >= 0 && key_id < gguf_get_n_kv(ctx));
    // 断言键值对的类型为 GGUF_TYPE_UINT32
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_UINT32);
    // 返回键值对中 uint32_t 类型的值
    return ctx->kv[key_id].value.uint32;
}

// 获取 int32_t 类型的值
int32_t gguf_get_val_i32(const struct gguf_context * ctx, int key_id) {
    // 断言 key_id 大于等于 0 且小于键值对数量
    GGML_ASSERT(key_id >= 0 && key_id < gguf_get_n_kv(ctx));
    // 断言键值对的类型为 GGUF_TYPE_INT32
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_INT32);
    // 返回键值对中 int32_t 类型的值
    return ctx->kv[key_id].value.int32;
}

// 获取 float 类型的值
float gguf_get_val_f32(const struct gguf_context * ctx, int key_id) {
    // 断言 key_id 大于等于 0 且小于键值对数量
    GGML_ASSERT(key_id >= 0 && key_id < gguf_get_n_kv(ctx));
    // 断言键值对的类型为 GGUF_TYPE_FLOAT32
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_FLOAT32);
    // 返回键值对中 float 类型的值
    return ctx->kv[key_id].value.float32;
}

// 获取 uint64_t 类型的值
uint64_t gguf_get_val_u64(const struct gguf_context * ctx, int key_id) {
    // 断言 key_id 大于等于 0 且小于键值对数量
    GGML_ASSERT(key_id >= 0 && key_id < gguf_get_n_kv(ctx));
    // 断言键值对的类型为 GGUF_TYPE_UINT64
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_UINT64);
    // 返回键值对中 uint64_t 类型的值
    return ctx->kv[key_id].value.uint64;
}

// 获取 int64_t 类型的值
int64_t gguf_get_val_i64(const struct gguf_context * ctx, int key_id) {
    // 断言 key_id 大于等于 0 且小于键值对数量
    GGML_ASSERT(key_id >= 0 && key_id < gguf_get_n_kv(ctx));
    // 断言键值对的类型为 GGUF_TYPE_INT64
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_INT64);
    // 返回键值对中 int64_t 类型的值
    return ctx->kv[key_id].value.int64;
}

// 获取 double 类型的值
double gguf_get_val_f64(const struct gguf_context * ctx, int key_id) {
    # 断言 key_id 大于等于 0 并且小于键值对的数量
    GGML_ASSERT(key_id >= 0 && key_id < gguf_get_n_kv(ctx));
    # 断言键值对的类型为 GGUF_TYPE_FLOAT64
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_FLOAT64);
    # 返回键值对中浮点数类型的值
    return ctx->kv[key_id].value.float64;
// 返回指定键值对的布尔值
bool gguf_get_val_bool(const struct gguf_context * ctx, int key_id) {
    // 断言键值对的索引在有效范围内
    GGML_ASSERT(key_id >= 0 && key_id < gguf_get_n_kv(ctx));
    // 断言键值对的类型为布尔型
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_BOOL);
    // 返回键值对的布尔值
    return ctx->kv[key_id].value.bool_;
}

// 返回指定键值对的字符串值
const char * gguf_get_val_str(const struct gguf_context * ctx, int key_id) {
    // 断言键值对的索引在有效范围内
    GGML_ASSERT(key_id >= 0 && key_id < gguf_get_n_kv(ctx));
    // 断言键值对的类型为字符串型
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_STRING);
    // 返回键值对的字符串值
    return ctx->kv[key_id].value.str.data;
}

// 返回指定键值对的数据值
const void * gguf_get_val_data(const struct gguf_context * ctx, int key_id) {
    // 断言键值对的索引在有效范围内
    GGML_ASSERT(key_id >= 0 && key_id < gguf_get_n_kv(ctx));
    // 断言键值对的类型不是数组型或字符串型
    GGML_ASSERT(ctx->kv[key_id].type != GGUF_TYPE_ARRAY);
    GGML_ASSERT(ctx->kv[key_id].type != GGUF_TYPE_STRING);
    // 返回键值对的数据值
    return &ctx->kv[key_id].value;
}

// 返回张量的数量
int gguf_get_n_tensors(const struct gguf_context * ctx) {
    return ctx->header.n_tensors;
}

// 查找指定名称的张量，返回其索引
int gguf_find_tensor(const struct gguf_context * ctx, const char * name) {
    // 如果未找到张量，则返回-1
    int tensorfound = -1;

    const int n_tensors = gguf_get_n_tensors(ctx);

    for (int i = 0; i < n_tensors; ++i) {
        if (strcmp(name, gguf_get_tensor_name(ctx, i)) == 0) {
            tensorfound = i;
            break;
        }
    }

    return tensorfound;
}

// 返回指定索引的张量的偏移量
size_t gguf_get_tensor_offset(const struct gguf_context * ctx, int i) {
    return ctx->infos[i].offset;
}

// 返回指定索引的张量的名称
char * gguf_get_tensor_name(const struct gguf_context * ctx, int i) {
    return ctx->infos[i].name.data;
}

// 返回指定索引的张量的类型
enum ggml_type gguf_get_tensor_type(const struct gguf_context * ctx, int i) {
    return ctx->infos[i].type;
}

// 获取或添加指定键的索引
static int gguf_get_or_add_key(struct gguf_context * ctx, const char * key) {
    // 查找键的索引
    const int idx = gguf_find_key(ctx, key);
    // 如果找到键，则返回其索引
    if (idx >= 0) {
        return idx;
    }

    const int n_kv = gguf_get_n_kv(ctx);

    // 重新分配内存以添加新的键值对
    ctx->kv = realloc(ctx->kv, (n_kv + 1) * sizeof(struct gguf_kv));
    ctx->kv[n_kv].key.n    = strlen(key);
    ctx->kv[n_kv].key.data = strdup(key);
    ctx->header.n_kv++;
}
    # 返回变量 n_kv 的值
    return n_kv;
// 设置一个无符号8位整数类型的键值对到上下文中
void gguf_set_val_u8(struct gguf_context * ctx, const char * key, uint8_t val) {
    // 获取或添加键的索引
    const int idx = gguf_get_or_add_key(ctx, key);

    // 设置键值对的类型为无符号8位整数
    ctx->kv[idx].type        = GGUF_TYPE_UINT8;
    // 设置键值对的值为传入的无符号8位整数值
    ctx->kv[idx].value.uint8 = val;
}

// 设置一个有符号8位整数类型的键值对到上下文中
void gguf_set_val_i8(struct gguf_context * ctx, const char * key, int8_t val) {
    // 获取或添加键的索引
    const int idx = gguf_get_or_add_key(ctx, key);

    // 设置键值对的类型为有符号8位整数
    ctx->kv[idx].type       = GGUF_TYPE_INT8;
    // 设置键值对的值为传入的有符号8位整数值
    ctx->kv[idx].value.int8 = val;
}

// 设置一个无符号16位整数类型的键值对到上下文中
void gguf_set_val_u16(struct gguf_context * ctx, const char * key, uint16_t val) {
    // 获取或添加键的索引
    const int idx = gguf_get_or_add_key(ctx, key);

    // 设置键值对的类型为无符号16位整数
    ctx->kv[idx].type         = GGUF_TYPE_UINT16;
    // 设置键值对的值为传入的无符号16位整数值
    ctx->kv[idx].value.uint16 = val;
}

// 设置一个有符号16位整数类型的键值对到上下文中
void gguf_set_val_i16(struct gguf_context * ctx, const char * key, int16_t val) {
    // 获取或添加键的索引
    const int idx = gguf_get_or_add_key(ctx, key);

    // 设置键值对的类型为有符号16位整数
    ctx->kv[idx].type        = GGUF_TYPE_INT16;
    // 设置键值对的值为传入的有符号16位整数值
    ctx->kv[idx].value.int16 = val;
}

// 设置一个无符号32位整数类型的键值对到上下文中
void gguf_set_val_u32(struct gguf_context * ctx, const char * key, uint32_t val) {
    // 获取或添加键的索引
    const int idx = gguf_get_or_add_key(ctx, key);

    // 设置键值对的类型为无符号32位整数
    ctx->kv[idx].type         = GGUF_TYPE_UINT32;
    // 设置键值对的值为传入的无符号32位整数值
    ctx->kv[idx].value.uint32 = val;
}

// 设置一个有符号32位整数类型的键值对到上下文中
void gguf_set_val_i32(struct gguf_context * ctx, const char * key, int32_t val) {
    // 获取或添加键的索引
    const int idx = gguf_get_or_add_key(ctx, key);

    // 设置键值对的类型为有符号32位整数
    ctx->kv[idx].type        = GGUF_TYPE_INT32;
    // 设置键值对的值为传入的有符号32位整数值
    ctx->kv[idx].value.int32 = val;
}

// 设置一个32位浮点数类型的键值对到上下文中
void gguf_set_val_f32(struct gguf_context * ctx, const char * key, float val) {
    // 获取或添加键的索引
    const int idx = gguf_get_or_add_key(ctx, key);

    // 设置键值对的类型为32位浮点数
    ctx->kv[idx].type          = GGUF_TYPE_FLOAT32;
    // 设置键值对的值为传入的32位浮点数值
    ctx->kv[idx].value.float32 = val;
}

// 设置一个无符号64位整数类型的键值对到上下文中
void gguf_set_val_u64(struct gguf_context * ctx, const char * key, uint64_t val) {
    // 获取或添加键的索引
    const int idx = gguf_get_or_add_key(ctx, key);

    // 设置键值对的类型为无符号64位整数
    ctx->kv[idx].type         = GGUF_TYPE_UINT64;
    // 设置键值对的值为传入的无符号64位整数值
    ctx->kv[idx].value.uint64 = val;
}

// 设置一个有符号64位整数类型的键值对到上下文中
void gguf_set_val_i64(struct gguf_context * ctx, const char * key, int64_t val) {
    // 获取或添加键的索引
    const int idx = gguf_get_or_add_key(ctx, key);

    // 设置键值对的类型为有符号64位整数
    ctx->kv[idx].type        = GGUF_TYPE_INT64;
    // 设置键值对的值为传入的有符号64位整数值
    ctx->kv[idx].value.int64 = val;
}
// 设置浮点数类型的键值对
void gguf_set_val_f64(struct gguf_context * ctx, const char * key, double val) {
    // 获取或添加键的索引
    const int idx = gguf_get_or_add_key(ctx, key);

    // 设置键值对的类型为浮点数
    ctx->kv[idx].type          = GGUF_TYPE_FLOAT64;
    // 设置键值对的值为传入的浮点数
    ctx->kv[idx].value.float64 = val;
}

// 设置布尔类型的键值对
void gguf_set_val_bool(struct gguf_context * ctx, const char * key, bool val) {
    // 获取或添加键的索引
    const int idx = gguf_get_or_add_key(ctx, key);

    // 设置键值对的类型为布尔值
    ctx->kv[idx].type        = GGUF_TYPE_BOOL;
    // 设置键值对的值为传入的布尔值
    ctx->kv[idx].value.bool_ = val;
}

// 设置字符串类型的键值对
void gguf_set_val_str(struct gguf_context * ctx, const char * key, const char * val) {
    // 获取或添加键的索引
    const int idx = gguf_get_or_add_key(ctx, key);

    // 设置键值对的类型为字符串
    ctx->kv[idx].type           = GGUF_TYPE_STRING;
    // 设置键值对的字符串长度为传入字符串的长度
    ctx->kv[idx].value.str.n    = strlen(val);
    // 复制传入字符串的内容到键值对的数据中
    ctx->kv[idx].value.str.data = strdup(val);
}

// 设置数组类型的键值对
void gguf_set_arr_data(struct gguf_context * ctx, const char * key, enum gguf_type type, const void * data, int n) {
    // 获取或添加键的索引
    const int idx = gguf_get_or_add_key(ctx, key);

    // 设置键值对的类型为数组
    ctx->kv[idx].type           = GGUF_TYPE_ARRAY;
    // 设置数组的类型
    ctx->kv[idx].value.arr.type = type;
    // 设置数组的长度
    ctx->kv[idx].value.arr.n    = n;
    // 分配内存并复制传入数据到键值对的数据中
    ctx->kv[idx].value.arr.data = GGML_MALLOC(n*gguf_type_size(type));
    memcpy(ctx->kv[idx].value.arr.data, data, n*gguf_type_size(type));
}

// 设置字符串数组类型的键值对
void gguf_set_arr_str(struct gguf_context * ctx, const char * key, const char ** data, int n) {
    // 获取或添加键的索引
    const int idx = gguf_get_or_add_key(ctx, key);

    // 设置键值对的类型为数组
    ctx->kv[idx].type           = GGUF_TYPE_ARRAY;
    // 设置数组的类型为字符串
    ctx->kv[idx].value.arr.type = GGUF_TYPE_STRING;
    // 设置数组的长度
    ctx->kv[idx].value.arr.n    = n;
    // 分配内存并复制传入字符串数组到键值对的数据中
    ctx->kv[idx].value.arr.data = GGML_MALLOC(n*sizeof(struct gguf_str));
    // 遍历传入字符串数组，设置每个字符串的长度和内容
    for (int i = 0; i < n; i++) {
        struct gguf_str * str = &((struct gguf_str *)ctx->kv[idx].value.arr.data)[i];
        str->n    = strlen(data[i]);
        str->data = strdup(data[i]);
    }
}

// 从另一个上下文中设置或添加键值对
void gguf_set_kv(struct gguf_context * ctx, struct gguf_context * src) {
    // 略
}

// 添加张量到上下文中
void gguf_add_tensor(struct gguf_context * ctx, const struct ggml_tensor * tensor) {
    // 略
}
    // 获取当前张量的索引
    const int idx = ctx->header.n_tensors;
    // 重新分配内存以存储张量信息
    ctx->infos = realloc(ctx->infos, (idx + 1)*sizeof(struct gguf_tensor_info));

    // 设置张量名称的长度和数据
    ctx->infos[idx].name.n    = strlen(tensor->name);
    ctx->infos[idx].name.data = strdup(tensor->name);

    // 初始化张量的每个维度为1
    for (int i = 0; i < GGML_MAX_DIMS; ++i) {
        ctx->infos[idx].ne[i] = 1;
    }

    // 获取张量的维度数量并设置每个维度的大小
    ctx->infos[idx].n_dims = ggml_n_dims(tensor);
    for (uint32_t i = 0; i < ctx->infos[idx].n_dims; i++) {
        ctx->infos[idx].ne[i] = tensor->ne[i];
    }

    // 设置张量的类型、偏移量、数据和大小
    ctx->infos[idx].type   = tensor->type;
    ctx->infos[idx].offset = 0;
    ctx->infos[idx].data   = tensor->data;
    ctx->infos[idx].size   = ggml_nbytes(tensor);

    // 如果存在多个张量，则计算当前张量的偏移量
    if (ctx->header.n_tensors > 0) {
        ctx->infos[idx].offset = ctx->infos[idx - 1].offset + GGML_PAD(ctx->infos[idx - 1].size, ctx->alignment);
    }

    // 增加张量数量
    ctx->header.n_tensors++;
// 设置指定张量的类型
void gguf_set_tensor_type(struct gguf_context * ctx, const char * name, enum ggml_type type) {
    // 查找张量在上下文中的索引
    const int idx = gguf_find_tensor(ctx, name);
    // 如果索引小于0，表示未找到张量，触发断言错误
    if (idx < 0) {
        GGML_ASSERT(false && "tensor not found");
    }

    // 设置张量的类型
    ctx->infos[idx].type = type;
}

// 设置指定张量的数据
void gguf_set_tensor_data(struct gguf_context * ctx, const char * name, const void * data, size_t size) {
    // 查找张量在上下文中的索引
    const int idx = gguf_find_tensor(ctx, name);
    // 如果索引小于0，表示未找到张量，触发断言错误
    if (idx < 0) {
        GGML_ASSERT(false && "tensor not found");
    }

    // 设置张量的数据和大小
    ctx->infos[idx].data = data;
    ctx->infos[idx].size = size;

    // 更新偏移量
    for (uint32_t i = idx + 1; i < ctx->header.n_tensors; ++i) {
        ctx->infos[i].offset = ctx->infos[i - 1].offset + GGML_PAD(ctx->infos[i - 1].size, ctx->alignment);
    }
}

// 初始化缓冲区结构体
struct gguf_buf {
    void * data;
    size_t size;
    size_t offset;
};

// 初始化缓冲区结构体
static struct gguf_buf gguf_buf_init(size_t size) {
    struct gguf_buf buf = {
        /*buf.data   =*/ size == 0 ? NULL : GGML_MALLOC(size),
        /*buf.size   =*/ size,
        /*buf.offset =*/ 0,
    };

    return buf;
}

// 释放缓冲区内存
static void gguf_buf_free(struct gguf_buf buf) {
    if (buf.data) {
        GGML_FREE(buf.data);
    }
}

// 扩展缓冲区大小
static void gguf_buf_grow(struct gguf_buf * buf, size_t size) {
    if (buf->offset + size > buf->size) {
        buf->size = 1.5*(buf->offset + size);
        if (buf->data) {
            buf->data = realloc(buf->data, buf->size);
        }
    }
}

// 写入字符串到缓冲区
static void gguf_bwrite_str(struct gguf_buf * buf, const struct gguf_str * val) {
    gguf_buf_grow(buf, sizeof(val->n) + val->n);

    if (buf->data) {
        memcpy((char *) buf->data + buf->offset, &val->n, sizeof(val->n));
    }
}
    # 将偏移量增加一个整数的大小
    buf->offset += sizeof(val->n);

    # 如果缓冲区中有数据
    if (buf->data) {
        # 将值的数据复制到缓冲区中的偏移位置
        memcpy((char *) buf->data + buf->offset, val->data, val->n);
    }
    # 将偏移量增加值的大小
    buf->offset += val->n;
// 写入单个元素到缓冲区中，确保缓冲区足够大
static void gguf_bwrite_el(struct gguf_buf * buf, const void * val, size_t el_size) {
    // 扩展缓冲区大小以容纳新元素
    gguf_buf_grow(buf, el_size);

    // 如果缓冲区存在
    if (buf->data) {
        // 将值复制到缓冲区中
        memcpy((char *) buf->data + buf->offset, val, el_size);
    }
    // 更新偏移量
    buf->offset += el_size;
}

// 将上下文中的信息写入缓冲区中，可选择是否只写入元数据
static void gguf_write_to_buf(const struct gguf_context * ctx, struct gguf_buf * buf, bool only_meta) {
    // 写入头部信息
    gguf_bwrite_el(buf, &ctx->header.magic,     sizeof(ctx->header.magic));
    gguf_bwrite_el(buf, &ctx->header.version,   sizeof(ctx->header.version));
    gguf_bwrite_el(buf, &ctx->header.n_tensors, sizeof(ctx->header.n_tensors));
    gguf_bwrite_el(buf, &ctx->header.n_kv,      sizeof(ctx->header.n_kv));

    // 写入键值对

    // 写入张量信息
    for (uint32_t i = 0; i < ctx->header.n_tensors; ++i) {
        struct gguf_tensor_info * info = &ctx->infos[i];

        // 写入字符串到缓冲区
        gguf_bwrite_str(buf, &info->name);
        gguf_bwrite_el (buf, &info->n_dims, sizeof(info->n_dims));
        for (uint32_t j = 0; j < info->n_dims; ++j) {
            gguf_bwrite_el(buf, &info->ne[j], sizeof(info->ne[j]));
        }
        gguf_bwrite_el(buf, &info->type,   sizeof(info->type));
        gguf_bwrite_el(buf, &info->offset, sizeof(info->offset));
    }

    // 我们要求数据部分对齐，因此考虑任何填充
    {
        const size_t offset     = buf->offset;
        const size_t offset_pad = GGML_PAD(offset, ctx->alignment);

        // 如果需要填充
        if (offset_pad != offset) {
            uint8_t pad = 0;
            // 填充缓冲区
            for (size_t i = 0; i < offset_pad - offset; ++i) {
                gguf_bwrite_el(buf, &pad, sizeof(pad));
            }
        }
    }

    // 如果只需要元数据，则返回
    if (only_meta) {
        return;
    }

    size_t offset = 0;

    // 写入张量数据
}
    // 遍历上下文中的张量数量
    for (uint32_t i = 0; i < ctx->header.n_tensors; ++i) {
        // 获取当前张量的信息
        struct gguf_tensor_info * info = &ctx->infos[i];

        // 获取当前张量的大小
        const size_t size     = info->size;
        // 计算当前张量的对齐后大小
        const size_t size_pad = GGML_PAD(size, ctx->alignment);

        // 将当前张量的数据写入缓冲区
        gguf_bwrite_el(buf, info->data, size);

        // 如果对齐后大小不等于原大小
        if (size_pad != size) {
            // 补齐数据
            uint8_t pad = 0;
            for (size_t j = 0; j < size_pad - size; ++j) {
                gguf_bwrite_el(buf, &pad, sizeof(pad));
            }
        }

        // 断言偏移量与当前张量的偏移量相等
        GGML_ASSERT(offset == info->offset);

        // 更新偏移量
        offset += size_pad;
    }
// 写入数据到文件
void gguf_write_to_file(const struct gguf_context * ctx, const char * fname, bool only_meta) {
    // 打开文件准备写入
    FILE * file = fopen(fname, "wb");
    // 如果文件打开失败，则断言并输出错误信息
    if (!file) {
        GGML_ASSERT(false && "failed to open file for writing");
    }

    // 初始化缓冲区
    struct gguf_buf buf = gguf_buf_init(16*1024);

    // 将数据写入缓冲区
    gguf_write_to_buf(ctx, &buf, only_meta);

    // 将缓冲区的数据写入文件
    fwrite(buf.data, 1, buf.offset, file);

    // 释放缓冲区
    gguf_buf_free(buf);

    // 关闭文件
    fclose(file);
}

// 获取元数据的大小
size_t gguf_get_meta_size(const struct gguf_context * ctx) {
    // 无需分配内存 - 只计算大小
    struct gguf_buf buf = gguf_buf_init(0);

    // 将数据写入缓冲区
    gguf_write_to_buf(ctx, &buf, true);

    // 返回缓冲区的偏移量作为元数据的大小
    return buf.offset;
}

// 获取元数据的数据
void gguf_get_meta_data(const struct gguf_context * ctx, void * data) {
    // 初始化缓冲区
    struct gguf_buf buf = gguf_buf_init(16*1024);

    // 将数据写入缓冲区
    gguf_write_to_buf(ctx, &buf, true);

    // 将缓冲区的数据复制到给定的数据指针中
    memcpy(data, buf.data, buf.offset);

    // 释放缓冲区
    gguf_buf_free(buf);
}

////////////////////////////////////////////////////////////////////////////////

// 检查 CPU 是否支持 AVX 指令集
int ggml_cpu_has_avx(void) {
#if defined(__AVX__)
    return 1;
#else
    return 0;
#endif
}

// 检查 CPU 是否支持 AVX VNNI 指令集
int ggml_cpu_has_avx_vnni(void) {
#if defined(__AVXVNNI__)
    return 1;
#else
    return 0;
#endif
}

// 检查 CPU 是否支持 AVX2 指令集
int ggml_cpu_has_avx2(void) {
#if defined(__AVX2__)
    return 1;
#else
    return 0;
#endif
}

// 检查 CPU 是否支持 AVX512 指令集
int ggml_cpu_has_avx512(void) {
#if defined(__AVX512F__)
    return 1;
#else
    return 0;
#endif
}

// 检查 CPU 是否支持 AVX512 VBMI 指令集
int ggml_cpu_has_avx512_vbmi(void) {
#if defined(__AVX512VBMI__)
    return 1;
#else
    return 0;
#endif
}

// 检查 CPU 是否支持 AVX512 VNNI 指令集
int ggml_cpu_has_avx512_vnni(void) {
#if defined(__AVX512VNNI__)
    return 1;
#else
    return 0;
#endif
}

// 检查 CPU 是否支持 FMA 指令集
int ggml_cpu_has_fma(void) {
#if defined(__FMA__)
    return 1;
#else
    return 0;
#endif
}

// 检查 CPU 是否支持 NEON 指令集
int ggml_cpu_has_neon(void) {
#if defined(__ARM_NEON)
    return 1;
#else
    return 0;
#endif
}

// 检查 CPU 是否支持 ARM FMA 指令集
int ggml_cpu_has_arm_fma(void) {
#if defined(__ARM_FEATURE_FMA)
    return 1;
#else
    return 0;
#endif
}

// 检查 CPU 是否支持 Metal 框架
int ggml_cpu_has_metal(void) {
#if defined(GGML_USE_METAL)
    return 1;
#else
    return 0;
#endif
}

// 检查 CPU 是否支持 F16C 指令集
int ggml_cpu_has_f16c(void) {
#if defined(__F16C__)
    // 如果支持 F16C 指令集，则返回 1
    return 1;
#else
    // 否则返回 0
    return 0;
#endif
}

// 检查是否支持 ARM FP16 向量运算
int ggml_cpu_has_fp16_va(void) {
#if defined(__ARM_FEATURE_FP16_VECTOR_ARITHMETIC)
    // 如果支持 ARM FP16 向量运算，则返回 1
    return 1;
#else
    // 否则返回 0
    return 0;
#endif
}

// 检查是否支持 WASM SIMD
int ggml_cpu_has_wasm_simd(void) {
#if defined(__wasm_simd128__)
    // 如果支持 WASM SIMD，则返回 1
    return 1;
#else
    // 否则返回 0
    return 0;
#endif
}

// 检查是否支持 BLAS 库
int ggml_cpu_has_blas(void) {
#if defined(GGML_USE_ACCELERATE) || defined(GGML_USE_OPENBLAS) || defined(GGML_USE_CUBLAS) || defined(GGML_USE_VULKAN) || defined(GGML_USE_CLBLAST) || defined(GGML_USE_SYCL)
    // 如果支持 BLAS 库中的任意一种，则返回 1
    return 1;
#else
    // 否则返回 0
    return 0;
#endif
}

// 检查是否支持 CUBLAS
int ggml_cpu_has_cublas(void) {
#if defined(GGML_USE_CUBLAS)
    // 如果支持 CUBLAS，则返回 1
    return 1;
#else
    // 否则返回 0
    return 0;
#endif
}

// 检查是否支持 CLBLAST
int ggml_cpu_has_clblast(void) {
#if defined(GGML_USE_CLBLAST)
    // 如果支持 CLBLAST，则返回 1
    return 1;
#else
    // 否则返回 0
    return 0;
#endif
}

// 检查是否支持 VULKAN
int ggml_cpu_has_vulkan(void) {
#if defined(GGML_USE_VULKAN)
    // 如果支持 VULKAN，则返回 1
    return 1;
#else
    // 否则返回 0
    return 0;
#endif
}

// 检查是否支持 SYCL
int ggml_cpu_has_sycl(void) {
#if defined(GGML_USE_SYCL)
    // 如果支持 SYCL，则返回 1
    return 1;
#else
    // 否则返回 0
    return 0;
#endif
}

// 检查是否支持 GPU BLAS
int ggml_cpu_has_gpublas(void) {
    // 如果支持 CUBLAS、CLBLAST、VULKAN 或 SYCL 中的任意一种，则返回 1
    return ggml_cpu_has_cublas() || ggml_cpu_has_clblast() || ggml_cpu_has_vulkan() || ggml_cpu_has_sycl();
}

// 检查是否支持 SSE3 指令集
int ggml_cpu_has_sse3(void) {
#if defined(__SSE3__)
    // 如果支持 SSE3 指令集，则返回 1
    return 1;
#else
    // 否则返回 0
    return 0;
#endif
}

// 检查是否支持 SSSE3 指令集
int ggml_cpu_has_ssse3(void) {
#if defined(__SSSE3__)
    // 如果支持 SSSE3 指令集，则返回 1
    return 1;
#else
    // 否则返回 0
    return 0;
#endif
}

// 检查是否支持 VSX 指令集
int ggml_cpu_has_vsx(void) {
#if defined(__POWER9_VECTOR__)
    // 如果支持 VSX 指令集，则返回 1
    return 1;
#else
    // 否则返回 0
    return 0;
#endif
}

////////////////////////////////////////////////////////////////////////////////
```