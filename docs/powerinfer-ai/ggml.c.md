# `PowerInfer\ggml.c`

```
// 禁用 Windows 上荒谬的“不安全”警告
#define _CRT_SECURE_NO_DEPRECATE 
// 在 MSVC 上使用 M_PI
#define _USE_MATH_DEFINES 

#include "ggml-impl.h" // 引入 ggml-impl.h 头文件
#include "ggml-quants.h" // 引入 ggml-quants.h 头文件

#if defined(_MSC_VER) || defined(__MINGW32__)
#include <malloc.h> // 在 MSC/MINGW 上使用 malloc.h
#elif !defined(__FreeBSD__) && !defined(__NetBSD__) && !defined(__OpenBSD__)
#include <alloca.h> // 在其他系统上使用 alloca.h
#endif

#include <assert.h> // 引入断言头文件
#include <errno.h> // 引入错误处理头文件
#include <time.h> // 引入时间处理头文件
#include <math.h> // 引入数学处理头文件
#include <stdlib.h> // 引入标准库头文件
#include <string.h> // 引入字符串处理头文件
#include <stdint.h> // 引入整数类型头文件
#include <inttypes.h> // 引入整数格式化头文件
#include <stdio.h> // 引入标准输入输出头文件
#include <float.h> // 引入浮点数处理头文件
#include <limits.h> // 引入整数限制头文件
#include <stdarg.h> // 引入可变参数头文件
#include <signal.h> // 引入信号处理头文件

// #define _GNU_SOURCE
// #include <sched.h>

#ifdef GGML_USE_METAL
#include <unistd.h> // 在使用 Metal 时引入头文件
#endif

#if defined(_MSC_VER)
// 禁用“可能丢失数据”的警告，避免大量强制类型转换
// 我们应该小心使用类型转换
#pragma warning(disable: 4244 4267)

// 禁用 POSIX 废弃警告
// 这些函数永远不会消失
#pragma warning(disable: 4996)
#endif

#if defined(_WIN32)

#include <windows.h> // 在 Windows 上引入 Windows 头文件

typedef volatile LONG atomic_int; // 定义原子整型
typedef atomic_int atomic_bool; // 定义原子布尔型

static void atomic_store(atomic_int * ptr, LONG val) { // 原子存储操作
    InterlockedExchange(ptr, val);
}
static LONG atomic_load(atomic_int * ptr) { // 原子加载操作
    return InterlockedCompareExchange(ptr, 0, 0);
}
static LONG atomic_fetch_add(atomic_int * ptr, LONG inc) { // 原子加操作
    return InterlockedExchangeAdd(ptr, inc);
}
static LONG atomic_fetch_sub(atomic_int * ptr, LONG dec) { // 原子减操作
    return atomic_fetch_add(ptr, -(dec));
}

typedef HANDLE pthread_t; // 定义线程句柄类型

typedef DWORD thread_ret_t; // 定义线程返回类型
static int pthread_create(pthread_t * out, void * unused, thread_ret_t(*func)(void *), void * arg) { // 创建线程
    (void) unused;
    HANDLE handle = CreateThread(NULL, 0, (LPTHREAD_START_ROUTINE) func, arg, 0, NULL);
    if (handle == NULL)
    {
        return EAGAIN;
    }

    *out = handle;
    return 0;
}

static int pthread_join(pthread_t thread, void * unused) { // 等待线程结束
    (void) unused;
    # 等待线程的状态变为 signaled，返回等待结果
    int ret = (int) WaitForSingleObject(thread, INFINITE);
    # 关闭线程句柄
    CloseHandle(thread);
    # 返回等待结果
    return ret;
}

static int sched_yield (void) {
    // 在当前线程放弃执行，让出 CPU 时间片给其他线程执行
    Sleep (0);
    // 返回 0，表示成功
    return 0;
}
#else
#include <pthread.h>
#include <stdatomic.h>

typedef void * thread_ret_t;

#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>

#endif

#ifdef GGML_USE_CPU_HBM
#include <hbwmalloc.h>
#endif

#if defined(__APPLE__)
#include <TargetConditionals.h>
#endif

#if (defined(__linux__) || defined(__APPLE__) || defined(__FreeBSD__) || defined(__NetBSD__) || defined(__OpenBSD__)) && \
    (!defined(TARGET_OS_TV) && !defined(TARGET_OS_WATCH))

#include <sys/wait.h>

void ggml_print_backtrace(void) {
    /*
    #include <execinfo.h>
    #include <dlfcn.h>

    void * trace[100];

    int nptrs = backtrace(trace, sizeof(trace)/sizeof(trace[0]));

    backtrace_symbols_fd(trace, nptrs, STDERR_FILENO);
    */

    // backtrack_symbols does not show line numbers, use gdb instead
    // 使用 gdb 打印当前线程的调用栈信息
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
            NULL);
    } else {
        waitpid(pid, NULL, 0);
    }
}
#else
void ggml_print_backtrace(void) {
    // 平台不支持，无法打印调用栈信息
    // platform not supported
}
#endif

// #define GGML_PERF
#define GGML_DEBUG 0
#define GGML_GELU_FP16
#define GGML_GELU_QUICK_FP16
#define GGML_SILU_FP16
// #define GGML_CROSS_ENTROPY_EXP_FP16
// #define GGML_FLASH_ATTN_EXP_FP16

#define GGML_SOFT_MAX_UNROLL 4
#define GGML_VEC_DOT_UNROLL  2
#define GGML_VEC_MAD_UNROLL  32

//
// logging
//

#if (GGML_DEBUG >= 1)
#define GGML_PRINT_DEBUG(...) printf(__VA_ARGS__)
#else
#define GGML_PRINT_DEBUG(...)
#endif

#if (GGML_DEBUG >= 5)
#define GGML_PRINT_DEBUG_5(...) printf(__VA_ARGS__)
#else
#define GGML_PRINT_DEBUG_5(...)
#endif

#if (GGML_DEBUG >= 10)
#define GGML_PRINT_DEBUG_10(...) printf(__VA_ARGS__)
#else
// 定义一个宏，用于在调试级别为10时打印信息
#define GGML_PRINT_DEBUG_10(...)
#endif

// 定义一个宏，用于打印信息
#define GGML_PRINT(...) printf(__VA_ARGS__)

//
// 日志块结束
//

#ifdef GGML_USE_ACCELERATE
// 取消注释以使用 vDSP 进行软最大值计算
// 注意：不确定它是否实际上更快
//#define GGML_SOFT_MAX_ACCELERATE
#endif

#if defined(_MSC_VER) || defined(__MINGW32__)
// 定义一个宏，用于在 Windows 平台上分配对齐内存
#define GGML_ALIGNED_MALLOC(size) _aligned_malloc(size, GGML_MEM_ALIGN)
// 定义一个宏，用于在 Windows 平台上释放对齐内存
#define GGML_ALIGNED_FREE(ptr)    _aligned_free(ptr)
#else
// 定义一个静态内联函数，用于分配对齐内存
inline static void * ggml_aligned_malloc(size_t size) {
    if (size == 0) {
        GGML_PRINT("WARNING: Behavior may be unexpected when allocating 0 bytes for ggml_aligned_malloc!\n");
        return NULL;
    }
    void * aligned_memory = NULL;
#ifdef GGML_USE_CPU_HBM
    int result = hbw_posix_memalign(&aligned_memory, 16, size);
#elif GGML_USE_METAL
    int result = posix_memalign(&aligned_memory, sysconf(_SC_PAGESIZE), size);
#else
    int result = posix_memalign(&aligned_memory, GGML_MEM_ALIGN, size);
#endif
    if (result != 0) {
        // 处理分配失败
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
        return NULL;
    }
    return aligned_memory;
}
// 定义一个宏，用于在非 Windows 平台上分配对齐内存
#define GGML_ALIGNED_MALLOC(size) ggml_aligned_malloc(size)
#ifdef GGML_USE_CPU_HBM
// 定义一个宏，用于在使用 CPU HBM 时释放对齐内存
#define GGML_ALIGNED_FREE(ptr)    if(NULL != ptr) hbw_free(ptr)
#else
// 定义一个宏，用于在非 CPU HBM 平台上释放对齐内存
#define GGML_ALIGNED_FREE(ptr)    free(ptr)
#endif
#endif

// 定义一个宏，表示未使用的变量
#define UNUSED GGML_UNUSED
// 定义一个宏，用于交换两个变量的值
#define SWAP(x, y, T) do { T SWAP = x; x = y; y = SWAP; } while (0)

//
// 张量访问宏
//

// 定义一个宏，用于在一元操作中本地化张量
#define GGML_TENSOR_UNARY_OP_LOCALS \
    GGML_TENSOR_LOCALS(int64_t, ne0, src0, ne) \
    GGML_TENSOR_LOCALS(size_t,  nb0, src0, nb) \
    # 定义宏GGML_TENSOR_LOCALS，用于声明和初始化指定类型的变量
    # 使用int64_t类型声明并初始化ne和dst变量
    GGML_TENSOR_LOCALS(int64_t, ne,  dst,  ne) \
    # 使用size_t类型声明并初始化nb和dst变量
    GGML_TENSOR_LOCALS(size_t,  nb,  dst,  nb)
// 定义宏，用于声明局部变量
#define GGML_TENSOR_BINARY_OP_LOCALS \
    GGML_TENSOR_LOCALS(int64_t, ne0, src0, ne) \
    GGML_TENSOR_LOCALS(size_t,  nb0, src0, nb) \
    GGML_TENSOR_LOCALS(int64_t, ne1, src1, ne) \
    GGML_TENSOR_LOCALS(size_t,  nb1, src1, nb) \
    GGML_TENSOR_LOCALS(int64_t, ne,  dst,  ne) \
    GGML_TENSOR_LOCALS(size_t,  nb,  dst,  nb)

// 根据不同的宏定义，包含不同的头文件
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
#endif

// 用于累加和的浮点类型
typedef double ggml_float;

// 取消 MIN 和 MAX 宏定义，重新定义
#undef MIN
#undef MAX
#define MIN(a, b) ((a) < (b) ? (a) : (b)) // 求最小值
#define MAX(a, b) ((a) > (b) ? (a) : (b)) // 求最大值

// 全局数据

// 预先计算的 f16 gelu 表 (128 KB)
static ggml_fp16_t ggml_table_gelu_f16[1 << 16];

// 预先计算的 f16 快速 gelu 表 (128 KB)
static ggml_fp16_t ggml_table_gelu_quick_f16[1 << 16];

// 预先计算的 f16 silu 表 (128 KB)
static ggml_fp16_t ggml_table_silu_f16[1 << 16];

// 预先计算的 f16 exp 表 (128 KB)
static ggml_fp16_t ggml_table_exp_f16[1 << 16];

// 预先计算的 f16 到 f32 表 (256 KB) (ggml-impl.h)
float ggml_table_f32_f16[1 << 16];

// 注意：不要在 ggml.c 中使用这些
// 这些是通过 ggml.h API 使用的
// 将 f16 转换为 f32
float ggml_fp16_to_fp32(ggml_fp16_t x) {
    return (float) GGML_FP16_TO_FP32(x);
}

// 将 f32 转换为 f16
ggml_fp16_t ggml_fp32_to_fp16(float x) {
    return GGML_FP32_TO_FP16(x);
}

// 将 f16 数组转换为 f32 数组
void ggml_fp16_to_fp32_row(const ggml_fp16_t * x, float * y, int n) {
    for (int i = 0; i < n; i++) {
        y[i] = GGML_FP16_TO_FP32(x[i]);
    }
}

// 将 f32 数组转换为 f16 数组
void ggml_fp32_to_fp16_row(const float * x, ggml_fp16_t * y, int n) {
    int i = 0;
#if defined(__F16C__)
    # 使用 AVX 指令集对 x 数组中的数据进行批量处理，每次处理 8 个元素
    for (; i + 7 < n; i += 8) {
        # 从 x 数组中加载 8 个单精度浮点数到 AVX 寄存器 x_vec
        __m256 x_vec = _mm256_loadu_ps(x + i);
        # 将 x_vec 中的单精度浮点数转换成整数，并使用最近整数舍入模式
        __m128i y_vec = _mm256_cvtps_ph(x_vec, _MM_FROUND_TO_NEAREST_INT);
        # 将 y_vec 中的整数存储到 y 数组中
        _mm_storeu_si128((__m128i *)(y + i), y_vec);
    }
    # 使用 SSE 指令集对 x 数组中剩余的数据进行批量处理，每次处理 4 个元素
    for(; i + 3 < n; i += 4) {
        # 从 x 数组中加载 4 个单精度浮点数到 SSE 寄存器 x_vec
        __m128 x_vec = _mm_loadu_ps(x + i);
        # 将 x_vec 中的单精度浮点数转换成整数，并使用最近整数舍入模式
        __m128i y_vec = _mm_cvtps_ph(x_vec, _MM_FROUND_TO_NEAREST_INT);
        # 将 y_vec 中的整数存储到 y 数组中
        _mm_storel_epi64((__m128i *)(y + i), y_vec);
    }
#endif
    // 从 i 开始遍历到 n，将 x[i] 的值转换为 FP16 存入 y[i]
    for (; i < n; i++) {
        y[i] = GGML_FP32_TO_FP16(x[i]);
    }
}

//
// timing
//

#if defined(_MSC_VER) || defined(__MINGW32__)
static int64_t timer_freq, timer_start;
void ggml_time_init(void) {
    LARGE_INTEGER t;
    QueryPerformanceFrequency(&t);
    timer_freq = t.QuadPart;

    // 下面的乘法可能会导致溢出，因此减去程序启动时间以减少发生溢出的可能性
    QueryPerformanceCounter(&t);
    timer_start = t.QuadPart;
}
int64_t ggml_time_ms(void) {
    LARGE_INTEGER t;
    QueryPerformanceCounter(&t);
    return ((t.QuadPart-timer_start) * 1000) / timer_freq;
}
int64_t ggml_time_us(void) {
    LARGE_INTEGER t;
    QueryPerformanceCounter(&t);
    return ((t.QuadPart-timer_start) * 1000000) / timer_freq;
}
#else
void ggml_time_init(void) {}
int64_t ggml_time_ms(void) {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return (int64_t)ts.tv_sec*1000 + (int64_t)ts.tv_nsec/1000000;
}

int64_t ggml_time_us(void) {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return (int64_t)ts.tv_sec*1000000 + (int64_t)ts.tv_nsec/1000;
}
#endif

// 返回当前时钟周期数
int64_t ggml_cycles(void) {
    return clock();
}

// 返回每毫秒的时钟周期数
int64_t ggml_cycles_per_ms(void) {
    return CLOCKS_PER_SEC/1000;
}

#ifdef GGML_PERF
#define ggml_perf_time_ms()       ggml_time_ms()
#define ggml_perf_time_us()       ggml_time_us()
#define ggml_perf_cycles()        0
#define ggml_perf_cycles_per_ms() 0
#else
#define ggml_perf_time_ms()       0
#define ggml_perf_time_us()       0
#define ggml_perf_cycles()        0
#define ggml_perf_cycles_per_ms() 0
#endif

//
// cache line
//

#if defined(__cpp_lib_hardware_interference_size)
#define CACHE_LINE_SIZE hardware_destructive_interference_size
#else
#if defined(__POWER9_VECTOR__)
#define CACHE_LINE_SIZE 128
#else
#define CACHE_LINE_SIZE 64
#endif
#endif
// 将缓存行大小除以 float 类型的大小，得到缓存行可以容纳的 float 数量
static const size_t CACHE_LINE_SIZE_F32 = CACHE_LINE_SIZE/sizeof(float);

// 计算两个 float 数组的点积，结果保存在 s 中
static void ggml_vec_dot_f32(const int n, float * restrict s, const float * restrict x, const float * restrict y);
// 计算两个 ggml_fp16_t 类型数组的点积，结果保存在 s 中
static void ggml_vec_dot_f16(const int n, float * restrict s, ggml_fp16_t * restrict x, ggml_fp16_t * restrict y);

// 定义不同类型的数据的特性
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
        .type_name                = "f32",
        .blck_size                = 1,
        .type_size                = sizeof(float),
        .is_quantized             = false,
        .vec_dot                  = (ggml_vec_dot_t) ggml_vec_dot_f32,
        .vec_dot_type             = GGML_TYPE_F32,
    },
    [GGML_TYPE_F16] = {
        .type_name                = "f16",
        .blck_size                = 1,
        .type_size                = sizeof(ggml_fp16_t),
        .is_quantized             = false,
        .to_float                 = (ggml_to_float_t) ggml_fp16_to_fp32_row,
        .from_float               = (ggml_from_float_t) ggml_fp32_to_fp16_row,
        .from_float_reference     = (ggml_from_float_t) ggml_fp32_to_fp16_row,
        .vec_dot                  = (ggml_vec_dot_t) ggml_vec_dot_f16,
        .vec_dot_type             = GGML_TYPE_F16,
    },
    [GGML_TYPE_Q4_0] = {
        // 定义类型名称为 "q4_0"
        .type_name                = "q4_0",
        // 定义块大小为 QK4_0
        .blck_size                = QK4_0,
        // 定义类型大小为 block_q4_0 的大小
        .type_size                = sizeof(block_q4_0),
        // 设置为量化状态为 true
        .is_quantized             = true,
        // 设置将浮点数转换为 q4_0 类型的函数
        .to_float                 = (ggml_to_float_t) dequantize_row_q4_0,
        // 设置将 q4_0 类型转换为浮点数的函数
        .from_float               = quantize_row_q4_0,
        // 设置将浮点数转换为 q4_0 类型的参考函数
        .from_float_reference     = (ggml_from_float_t) quantize_row_q4_0_reference,
        // 设置 q4_0 类型与 q8_0 类型的向量点乘函数
        .vec_dot                  = ggml_vec_dot_q4_0_q8_0,
        // 设置向量点乘的类型为 GGML_TYPE_Q8_0
        .vec_dot_type             = GGML_TYPE_Q8_0,
    },
    [GGML_TYPE_Q4_1] = {
        // 定义类型名称为 "q4_1"
        .type_name                = "q4_1",
        // 定义块大小为 QK4_1
        .blck_size                = QK4_1,
        // 定义类型大小为 block_q4_1 的大小
        .type_size                = sizeof(block_q4_1),
        // 设置为量化状态为 true
        .is_quantized             = true,
        // 设置将浮点数转换为 q4_1 类型的函数
        .to_float                 = (ggml_to_float_t) dequantize_row_q4_1,
        // 设置将 q4_1 类型转换为浮点数的函数
        .from_float               = quantize_row_q4_1,
        // 设置将浮点数转换为 q4_1 类型的参考函数
        .from_float_reference     = (ggml_from_float_t) quantize_row_q4_1_reference,
        // 设置 q4_1 类型与 q8_1 类型的向量点乘函数
        .vec_dot                  = ggml_vec_dot_q4_1_q8_1,
        // 设置向量点乘的类型为 GGML_TYPE_Q8_1
        .vec_dot_type             = GGML_TYPE_Q8_1,
    },
    [4] = { // GGML_TYPE_Q4_2
        // 定义类型名称为 "DEPRECATED"
        .type_name                = "DEPRECATED",
        // 定义块大小为 0
        .blck_size                = 0,
        // 定义类型大小为 0
        .type_size                = 0,
        // 设置为量化状态为 false
        .is_quantized             = false,
        // 设置将浮点数转换为 NULL
        .to_float                 = NULL,
        // 设置将 NULL 转换为浮点数
        .from_float               = NULL,
        // 设置将 NULL 转换为浮点数的参考函数
        .from_float_reference     = NULL,
        // 设置向量点乘为 NULL
        .vec_dot                  = NULL,
        // 设置向量点乘的类型为 GGML_TYPE_COUNT
        .vec_dot_type             = GGML_TYPE_COUNT,
    },
    [5] = { // GGML_TYPE_Q4_3
        // 定义类型名称为 "DEPRECATED"
        .type_name                = "DEPRECATED",
        // 定义块大小为 0
        .blck_size                = 0,
        // 定义类型大小为 0
        .type_size                = 0,
        // 设置为量化状态为 false
        .is_quantized             = false,
        // 设置将浮点数转换为 NULL
        .to_float                 = NULL,
        // 设置将 NULL 转换为浮点数
        .from_float               = NULL,
        // 设置将 NULL 转换为浮点数的参考函数
        .from_float_reference     = NULL,
        // 设置向量点乘为 NULL
        .vec_dot                  = NULL,
        // 设置向量点乘的类型为 GGML_TYPE_COUNT
        .vec_dot_type             = GGML_TYPE_COUNT,
    },
    # 定义 GGML_TYPE_Q5_0 类型的属性
    [GGML_TYPE_Q5_0] = {
        # 类型名称为 "q5_0"
        .type_name                = "q5_0",
        # 块大小为 QK5_0
        .blck_size                = QK5_0,
        # 类型大小为 block_q5_0 的大小
        .type_size                = sizeof(block_q5_0),
        # 是否量化为真
        .is_quantized             = true,
        # 转换为浮点数的函数为 dequantize_row_q5_0
        .to_float                 = (ggml_to_float_t) dequantize_row_q5_0,
        # 从浮点数转换的函数为 quantize_row_q5_0
        .from_float               = quantize_row_q5_0,
        # 从浮点数转换的参考函数为 quantize_row_q5_0_reference
        .from_float_reference     = (ggml_from_float_t) quantize_row_q5_0_reference,
        # 向量点积函数为 ggml_vec_dot_q5_0_q8_0
        .vec_dot                  = ggml_vec_dot_q5_0_q8_0,
        # 向量点积类型为 GGML_TYPE_Q8_0
        .vec_dot_type             = GGML_TYPE_Q8_0,
    },
    # 定义 GGML_TYPE_Q5_1 类型的属性
    [GGML_TYPE_Q5_1] = {
        # 类型名称为 "q5_1"
        .type_name                = "q5_1",
        # 块大小为 QK5_1
        .blck_size                = QK5_1,
        # 类型大小为 block_q5_1 的大小
        .type_size                = sizeof(block_q5_1),
        # 是否量化为真
        .is_quantized             = true,
        # 转换为浮点数的函数为 dequantize_row_q5_1
        .to_float                 = (ggml_to_float_t) dequantize_row_q5_1,
        # 从浮点数转换的函数为 quantize_row_q5_1
        .from_float               = quantize_row_q5_1,
        # 从浮点数转换的参考函数为 quantize_row_q5_1_reference
        .from_float_reference     = (ggml_from_float_t) quantize_row_q5_1_reference,
        # 向量点积函数为 ggml_vec_dot_q5_1_q8_1
        .vec_dot                  = ggml_vec_dot_q5_1_q8_1,
        # 向量点积类型为 GGML_TYPE_Q8_1
        .vec_dot_type             = GGML_TYPE_Q8_1,
    },
    # 定义 GGML_TYPE_Q8_0 类型的属性
    [GGML_TYPE_Q8_0] = {
        # 类型名称为 "q8_0"
        .type_name                = "q8_0",
        # 块大小为 QK8_0
        .blck_size                = QK8_0,
        # 类型大小为 block_q8_0 的大小
        .type_size                = sizeof(block_q8_0),
        # 是否量化为真
        .is_quantized             = true,
        # 转换为浮点数的函数为 dequantize_row_q8_0
        .to_float                 = (ggml_to_float_t) dequantize_row_q8_0,
        # 从浮点数转换的函数为 quantize_row_q8_0
        .from_float               = quantize_row_q8_0,
        # 从浮点数转换的参考函数为 quantize_row_q8_0_reference
        .from_float_reference     = (ggml_from_float_t) quantize_row_q8_0_reference,
        # 向量点积函数为 ggml_vec_dot_q8_0_q8_0
        .vec_dot                  = ggml_vec_dot_q8_0_q8_0,
        # 向量点积类型为 GGML_TYPE_Q8_0
        .vec_dot_type             = GGML_TYPE_Q8_0,
    },
    # 定义 GGML_TYPE_Q8_1 类型的属性
    [GGML_TYPE_Q8_1] = {
        # 类型名称为 "q8_1"
        .type_name                = "q8_1",
        # 块大小为 QK8_1
        .blck_size                = QK8_1,
        # 类型大小为 block_q8_1 结构体的大小
        .type_size                = sizeof(block_q8_1),
        # 是否进行量化
        .is_quantized             = true,
        # 从浮点数转换为 q8_1 类型的量化函数
        .from_float               = quantize_row_q8_1,
        # 从浮点数转换为 q8_1 类型的量化函数的参考实现
        .from_float_reference     = (ggml_from_float_t) quantize_row_q8_1_reference,
        # 向量点积类型为 GGML_TYPE_Q8_1
        .vec_dot_type             = GGML_TYPE_Q8_1,
    },
    # 定义 GGML_TYPE_Q2_K 类型的属性
    [GGML_TYPE_Q2_K] = {
        # 类型名称为 "q2_K"
        .type_name                = "q2_K",
        # 块大小为 QK_K
        .blck_size                = QK_K,
        # 类型大小为 block_q2_K 结构体的大小
        .type_size                = sizeof(block_q2_K),
        # 是否进行量化
        .is_quantized             = true,
        # 从浮点数转换为 q2_K 类型的量化函数
        .to_float                 = (ggml_to_float_t) dequantize_row_q2_K,
        # 从浮点数转换为 q2_K 类型的量化函数
        .from_float               = quantize_row_q2_K,
        # 从浮点数转换为 q2_K 类型的量化函数的参考实现
        .from_float_reference     = (ggml_from_float_t) quantize_row_q2_K_reference,
        # 向量点积函数为 ggml_vec_dot_q2_K_q8_K
        .vec_dot                  = ggml_vec_dot_q2_K_q8_K,
        # 向量点积类型为 GGML_TYPE_Q8_K
        .vec_dot_type             = GGML_TYPE_Q8_K,
    },
    # 定义 GGML_TYPE_Q3_K 类型的属性
    [GGML_TYPE_Q3_K] = {
        # 类型名称为 "q3_K"
        .type_name                = "q3_K",
        # 块大小为 QK_K
        .blck_size                = QK_K,
        # 类型大小为 block_q3_K 结构体的大小
        .type_size                = sizeof(block_q3_K),
        # 是否进行量化
        .is_quantized             = true,
        # 从浮点数转换为 q3_K 类型的量化函数
        .to_float                 = (ggml_to_float_t) dequantize_row_q3_K,
        # 从浮点数转换为 q3_K 类型的量化函数
        .from_float               = quantize_row_q3_K,
        # 从浮点数转换为 q3_K 类型的量化函数的参考实现
        .from_float_reference     = (ggml_from_float_t) quantize_row_q3_K_reference,
        # 向量点积函数为 ggml_vec_dot_q3_K_q8_K
        .vec_dot                  = ggml_vec_dot_q3_K_q8_K,
        # 向量点积类型为 GGML_TYPE_Q8_K
        .vec_dot_type             = GGML_TYPE_Q8_K,
    },
    # 定义 GGML_TYPE_Q4_K 类型的属性
    [GGML_TYPE_Q4_K] = {
        # 类型名称为 "q4_K"
        .type_name                = "q4_K",
        # 块大小为 QK_K
        .blck_size                = QK_K,
        # 类型大小为 block_q4_K 的大小
        .type_size                = sizeof(block_q4_K),
        # 是否量化为真
        .is_quantized             = true,
        # 转换为浮点数的函数为 dequantize_row_q4_K
        .to_float                 = (ggml_to_float_t) dequantize_row_q4_K,
        # 从浮点数转换的函数为 quantize_row_q4_K
        .from_float               = quantize_row_q4_K,
        # 从浮点数转换的参考函数为 quantize_row_q4_K_reference
        .from_float_reference     = (ggml_from_float_t) quantize_row_q4_K_reference,
        # 向量点积函数为 ggml_vec_dot_q4_K_q8_K
        .vec_dot                  = ggml_vec_dot_q4_K_q8_K,
        # 向量点积类型为 GGML_TYPE_Q8_K
        .vec_dot_type             = GGML_TYPE_Q8_K,
    },
    # 定义 GGML_TYPE_Q5_K 类型的属性
    [GGML_TYPE_Q5_K] = {
        # 类型名称为 "q5_K"
        .type_name                = "q5_K",
        # 块大小为 QK_K
        .blck_size                = QK_K,
        # 类型大小为 block_q5_K 的大小
        .type_size                = sizeof(block_q5_K),
        # 是否量化为真
        .is_quantized             = true,
        # 转换为浮点数的函数为 dequantize_row_q5_K
        .to_float                 = (ggml_to_float_t) dequantize_row_q5_K,
        # 从浮点数转换的函数为 quantize_row_q5_K
        .from_float               = quantize_row_q5_K,
        # 从浮点数转换的参考函数为 quantize_row_q5_K_reference
        .from_float_reference     = (ggml_from_float_t) quantize_row_q5_K_reference,
        # 向量点积函数为 ggml_vec_dot_q5_K_q8_K
        .vec_dot                  = ggml_vec_dot_q5_K_q8_K,
        # 向量点积类型为 GGML_TYPE_Q8_K
        .vec_dot_type             = GGML_TYPE_Q8_K,
    },
    # 定义 GGML_TYPE_Q6_K 类型的属性
    [GGML_TYPE_Q6_K] = {
        # 类型名称为 "q6_K"
        .type_name                = "q6_K",
        # 块大小为 QK_K
        .blck_size                = QK_K,
        # 类型大小为 block_q6_K 的大小
        .type_size                = sizeof(block_q6_K),
        # 是否量化为真
        .is_quantized             = true,
        # 转换为浮点数的函数为 dequantize_row_q6_K
        .to_float                 = (ggml_to_float_t) dequantize_row_q6_K,
        # 从浮点数转换的函数为 quantize_row_q6_K
        .from_float               = quantize_row_q6_K,
        # 从浮点数转换的参��函数为 quantize_row_q6_K_reference
        .from_float_reference     = (ggml_from_float_t) quantize_row_q6_K_reference,
        # 向量点积函数为 ggml_vec_dot_q6_K_q8_K
        .vec_dot                  = ggml_vec_dot_q6_K_q8_K,
        # 向量点积类型为 GGML_TYPE_Q8_K
        .vec_dot_type             = GGML_TYPE_Q8_K,
    },
    # 定义 GGML_TYPE_Q8_K 类型的属性
    [GGML_TYPE_Q8_K] = {
        # 类型名称为 "q8_K"
        .type_name                = "q8_K",
        # 块大小为 QK_K
        .blck_size                = QK_K,
        # 类型大小为 block_q8_K 的大小
        .type_size                = sizeof(block_q8_K),
        # 是否量化为真
        .is_quantized             = true,
        # 从浮点数转换的函数为 quantize_row_q8_K
        .from_float               = quantize_row_q8_K,
    }
// 内部测试使用，根据给定的类型返回类型特征
ggml_type_traits_t ggml_internal_get_type_traits(enum ggml_type type) {
    // 断言给定的类型小于类型数量
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

// 定义一个静态内联函数，对四个单精度浮点数进行相加
inline static float vaddvq_f32(float32x4_t v) {
    return vgetq_lane_f32(v, 0) + vgetq_lane_f32(v, 1) + vgetq_lane_f32(v, 2) + vgetq_lane_f32(v, 3);
}

#endif
#endif

// 我们定义一组通用的 C 宏，根据当前架构特定的内部指令集来映射
// 然后我们在下面的基本计算操作中只使用这些宏
// 添加对新架构的支持需要定义相应的 SIMD 宏
//
// GGML_F32_STEP / GGML_F16_STEP
//   单步处理的元素数量
//
// GGML_F32_EPR / GGML_F16_EPR
//   单个寄存器中可以容纳的元素数量
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
    # 循环遍历从 0 到 offset-1 的索引值
    for (int i = 0; i < offset; ++i) {         \
        # 将 x[i] 和 x[offset+i] 对应位置的元素相加，并将结果存储到 x[i] 中
        x[i] = vaddq_f32(x[i], x[offset+i]);   \
    }                                          \
    # 对 x[0] 中的元素进行一次浮点数四元素向量的规约操作，将结果存储到 res 中
    res = GGML_F32x4_REDUCE_ONE(x[0]);         \
// 定义宏，将 GGML_F32_VEC 重命名为 GGML_F32x4
#define GGML_F32_VEC        GGML_F32x4
// 定义宏，将 GGML_F32_VEC_ZERO 重命名为 GGML_F32x4_ZERO
#define GGML_F32_VEC_ZERO   GGML_F32x4_ZERO
// 定义宏，将 GGML_F32_VEC_SET1 重命名为 GGML_F32x4_SET1
#define GGML_F32_VEC_SET1   GGML_F32x4_SET1
// 定义宏，将 GGML_F32_VEC_LOAD 重命名为 GGML_F32x4_LOAD
#define GGML_F32_VEC_LOAD   GGML_F32x4_LOAD
// 定义宏，将 GGML_F32_VEC_STORE 重命名为 GGML_F32x4_STORE
#define GGML_F32_VEC_STORE  GGML_F32x4_STORE
// 定义宏，将 GGML_F32_VEC_FMA 重命名为 GGML_F32x4_FMA
#define GGML_F32_VEC_FMA    GGML_F32x4_FMA
// 定义宏，将 GGML_F32_VEC_ADD 重命名为 GGML_F32x4_ADD
#define GGML_F32_VEC_ADD    GGML_F32x4_ADD
// 定义宏，将 GGML_F32_VEC_MUL 重命名为 GGML_F32x4_MUL
#define GGML_F32_VEC_MUL    GGML_F32x4_MUL
// 定义宏，将 GGML_F32_VEC_REDUCE 重命名为 GGML_F32x4_REDUCE
#define GGML_F32_VEC_REDUCE GGML_F32x4_REDUCE

// 如果支持 ARM FP16 向量运算
#if defined(__ARM_FEATURE_FP16_VECTOR_ARITHMETIC)
    // 定义宏，将 GGML_F16_STEP 设置为 32
    #define GGML_F16_STEP 32
    // 定义宏，将 GGML_F16_EPR 设置为 8
    #define GGML_F16_EPR  8

    // 定义宏，将 GGML_F16x8 重命名为 float16x8_t
    #define GGML_F16x8              float16x8_t
    // 定义宏，将 GGML_F16x8_ZERO 设置为 vdupq_n_f16(0.0f)
    #define GGML_F16x8_ZERO         vdupq_n_f16(0.0f)
    // 定义宏，将 GGML_F16x8_SET1(x) 设置为 vdupq_n_f16(x)
    #define GGML_F16x8_SET1(x)      vdupq_n_f16(x)
    // 定义宏，将 GGML_F16x8_LOAD 设置为 vld1q_f16
    #define GGML_F16x8_LOAD         vld1q_f16
    // 定义宏，将 GGML_F16x8_STORE 设置为 vst1q_f16
    #define GGML_F16x8_STORE        vst1q_f16
    // 定义宏，将 GGML_F16x8_FMA(a, b, c) 设置为 vfmaq_f16(a, b, c)
    #define GGML_F16x8_FMA(a, b, c) vfmaq_f16(a, b, c)
    // 定义宏，将 GGML_F16x8_ADD 设置为 vaddq_f16
    #define GGML_F16x8_ADD          vaddq_f16
    // 定义宏，将 GGML_F16x8_MUL 设置为 vmulq_f16
    #define GGML_F16x8_MUL          vmulq_f16
    // 定义宏，将 GGML_F16x8_REDUCE(res, x) 设置为空
    #define GGML_F16x8_REDUCE(res, x)                             \
    # 定义一个 do-while 循环，用于执行下面的一系列操作
    do {                                                          \
        # 计算偏移量，将 GGML_F16_ARR 右移一位
        int offset = GGML_F16_ARR >> 1;                           \
        # 对数组 x 进行循环操作，将前半部分与后半部分的元素相加
        for (int i = 0; i < offset; ++i) {                        \
            x[i] = vaddq_f16(x[i], x[offset+i]);                  \
        }                                                         \
        # 将偏移量右移一位
        offset >>= 1;                                             \
        # 对数组 x 进行循环操作，将前半部分与后半部分的元素相加
        for (int i = 0; i < offset; ++i) {                        \
            x[i] = vaddq_f16(x[i], x[offset+i]);                  \
        }                                                         \
        # 将偏移量右移一位
        offset >>= 1;                                             \
        # 对数组 x 进行循环操作，将前半部分与后半部分的元素相加
        for (int i = 0; i < offset; ++i) {                        \
            x[i] = vaddq_f16(x[i], x[offset+i]);                  \
        }                                                         \
        # 定义一个 float32x4_t 类型的变量 t0，将 x[0] 的低位转换为 float32 类型
        const float32x4_t t0 = vcvt_f32_f16(vget_low_f16 (x[0])); \
        # 定义一个 float32x4_t 类型的变量 t1，将 x[0] 的高位转换为 float32 类型
        const float32x4_t t1 = vcvt_f32_f16(vget_high_f16(x[0])); \
        # 将 t0 和 t1 相加，并将结果转换为 ggml_float 类型，赋值给 res
        res = (ggml_float) vaddvq_f32(vaddq_f32(t0, t1));         \
    } while (0)

    # 定义一系列宏，用于对 GGML_F16_VEC 类型的变量进行操作
    #define GGML_F16_VEC                GGML_F16x8
    #define GGML_F16_VEC_ZERO           GGML_F16x8_ZERO
    #define GGML_F16_VEC_SET1           GGML_F16x8_SET1
    #define GGML_F16_VEC_LOAD(p, i)     GGML_F16x8_LOAD(p)
    #define GGML_F16_VEC_STORE(p, r, i) GGML_F16x8_STORE(p, r[i])
    #define GGML_F16_VEC_FMA            GGML_F16x8_FMA
    #define GGML_F16_VEC_ADD            GGML_F16x8_ADD
    #define GGML_F16_VEC_MUL            GGML_F16x8_MUL
    #define GGML_F16_VEC_REDUCE         GGML_F16x8_REDUCE
// 如果不支持FP16向量运算，则使用FP32，并利用vcvt_函数来进行FP16的转换
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
    int offset = GGML_F32_ARR >> 1;                               \
    # 对数组 x 进行循环操作，将前半部分和后半部分的元素相加
    for (int i = 0; i < offset; ++i) {                            \
        x[i] = _mm256_add_ps(x[i], x[offset+i]);                  \
    }                                                             \
    # 偏移量右移一位
    offset >>= 1;                                                 \
    # 对数组 x 进行循环操作，将前半部分和后半部分的元素相加
    for (int i = 0; i < offset; ++i) {                            \
        x[i] = _mm256_add_ps(x[i], x[offset+i]);                  \
    }                                                             \
    # 偏移量右移一位
    offset >>= 1;                                                 \
    # 对数组 x 进行循环操作，将前半部分和后半部分的元素相加
    for (int i = 0; i < offset; ++i) {                            \
        x[i] = _mm256_add_ps(x[i], x[offset+i]);                  \
    }                                                             \
    # 计算 t0，将 x[0] 的低128位和高128位相加
    const __m128 t0 = _mm_add_ps(_mm256_castps256_ps128(x[0]),    \
                                 _mm256_extractf128_ps(x[0], 1)); \
    # 计算 t1，将 t0 的两个元素相加
    const __m128 t1 = _mm_hadd_ps(t0, t0);                        \
    # 计算 res，将 t1 的两个元素相加并转换为单精度浮点数
    res = _mm_cvtss_f32(_mm_hadd_ps(t1, t1));                     \
} while (0)
// 结束 do-while 循环，这种写法通常用于确保循环体至少执行一次

// 定义宏，将 GGML_F32_VEC 替换为 GGML_F32x8
#define GGML_F32_VEC        GGML_F32x8
// 定义宏，将 GGML_F32_VEC_ZERO 替换为 GGML_F32x8_ZERO
#define GGML_F32_VEC_ZERO   GGML_F32x8_ZERO
// 定义宏，将 GGML_F32_VEC_SET1 替换为 GGML_F32x8_SET1
#define GGML_F32_VEC_SET1   GGML_F32x8_SET1
// 定义宏，将 GGML_F32_VEC_LOAD 替换为 GGML_F32x8_LOAD
#define GGML_F32_VEC_LOAD   GGML_F32x8_LOAD
// 定义宏，将 GGML_F32_VEC_STORE 替换为 GGML_F32x8_STORE
#define GGML_F32_VEC_STORE  GGML_F32x8_STORE
// 定义宏，将 GGML_F32_VEC_FMA 替换为 GGML_F32x8_FMA
#define GGML_F32_VEC_FMA    GGML_F32x8_FMA
// 定义宏，将 GGML_F32_VEC_ADD 替换为 GGML_F32x8_ADD
#define GGML_F32_VEC_ADD    GGML_F32x8_ADD
// 定义宏，将 GGML_F32_VEC_MUL 替换为 GGML_F32x8_MUL
#define GGML_F32_VEC_MUL    GGML_F32x8_MUL
// 定义宏，将 GGML_F32_VEC_REDUCE 替换为 GGML_F32x8_REDUCE
#define GGML_F32_VEC_REDUCE GGML_F32x8_REDUCE

// 定义宏，将 GGML_F16_STEP 替换为 32
#define GGML_F16_STEP 32
// 定义宏，将 GGML_F16_EPR 替换为 8
#define GGML_F16_EPR  8

// F16 arithmetic is not supported by AVX, so we use F32 instead
// F16 算术不受 AVX 支持，因此我们使用 F32 代替

// 定义 GGML_F32Cx8 类型为 __m256
#define GGML_F32Cx8             __m256
// 定义 GGML_F32Cx8_ZERO 为 _mm256_setzero_ps()
#define GGML_F32Cx8_ZERO        _mm256_setzero_ps()
// 定义 GGML_F32Cx8_SET1(x) 为 _mm256_set1_ps(x)
#define GGML_F32Cx8_SET1(x)     _mm256_set1_ps(x)

#if defined(__F16C__)
// the  _mm256_cvt intrinsics require F16C
// _mm256_cvt 内在函数需要 F16C 支持
// 定义 GGML_F32Cx8_LOAD(x) 为 _mm256_cvtph_ps(_mm_loadu_si128((__m128i *)(x)))
// 定义 GGML_F32Cx8_STORE(x, y) 为 _mm_storeu_si128((__m128i *)(x), _mm256_cvtps_ph(y, 0))
#else
// 如果不支持 F16C
// 定义内联函数 __avx_f32cx8_load，将 ggml_fp16_t 类型的指针转换为 __m256 类型
static inline __m256 __avx_f32cx8_load(ggml_fp16_t *x) {
    float tmp[8];

    for (int i = 0; i < 8; i++) {
        tmp[i] = GGML_FP16_TO_FP32(x[i]);
    }

    return _mm256_loadu_ps(tmp);
}
// 定义内联函数 __avx_f32cx8_store，将 __m256 类型转换为 ggml_fp16_t 类型的指针
static inline void __avx_f32cx8_store(ggml_fp16_t *x, __m256 y) {
    float arr[8];

    _mm256_storeu_ps(arr, y);

    for (int i = 0; i < 8; i++)
        x[i] = GGML_FP32_TO_FP16(arr[i]);
}
// 定义 GGML_F32Cx8_LOAD(x) 为 __avx_f32cx8_load(x)
// 定义 GGML_F32Cx8_STORE(x, y) 为 __avx_f32cx8_store(x, y)
#endif

// 定义 GGML_F32Cx8_FMA 为 GGML_F32x8_FMA
#define GGML_F32Cx8_FMA         GGML_F32x8_FMA
// 定义 GGML_F32Cx8_ADD 为 _mm256_add_ps
#define GGML_F32Cx8_ADD         _mm256_add_ps
// 定义 GGML_F32Cx8_MUL 为 _mm256_mul_ps
#define GGML_F32Cx8_MUL         _mm256_mul_ps
// 定义 GGML_F32Cx8_REDUCE 为 GGML_F32x8_REDUCE

// 定义 GGML_F16_VEC 为 GGML_F32Cx8
#define GGML_F16_VEC                GGML_F32Cx8
// 定义 GGML_F16_VEC_ZERO 为 GGML_F32Cx8_ZERO
#define GGML_F16_VEC_ZERO           GGML_F32Cx8_ZERO
// 定义 GGML_F16_VEC_SET1 为 GGML_F32Cx8_SET1
#define GGML_F16_VEC_SET1           GGML_F32Cx8_SET1
// 定义 GGML_F16_VEC_LOAD(p, i) 为 GGML_F32Cx8_LOAD(p)
// 定义 GGML_F16_VEC_STORE(p, r, i) 为 GGML_F32Cx8_STORE(p, r[i])
// 定义 GGML_F16_VEC_FMA 为 GGML_F32Cx8_FMA
// 定义 F16 向量加法为 F32x8 加法
#define GGML_F16_VEC_ADD            GGML_F32Cx8_ADD
// 定义 F16 向量乘法为 F32x8 乘法
#define GGML_F16_VEC_MUL            GGML_F32Cx8_MUL
// 定义 F16 向量归约为 F32x8 归约
#define GGML_F16_VEC_REDUCE         GGML_F32Cx8_REDUCE

#elif defined(__POWER9_VECTOR__)

// 定义 GGML_SIMD
#define GGML_SIMD

// F32 POWER9

// 定义 F32 步长为 32
#define GGML_F32_STEP 32
// 定义 F32 每寄存器元素数为 4
#define GGML_F32_EPR  4

// 定义 F32x4 为 4 个 float 的向量
#define GGML_F32x4              vector float
// 定义 F32x4 零向量
#define GGML_F32x4_ZERO         0.0f
// 定义 F32x4 设置所有元素为相同值
#define GGML_F32x4_SET1         vec_splats
// 定义 F32x4 从内存加载数据
#define GGML_F32x4_LOAD(p)      vec_xl(0, p)
// 定义 F32x4 存储数据到内存
#define GGML_F32x4_STORE(p, r)  vec_xst(r, 0, p)
// 定义 F32x4 浮点数 FMA（fused multiply-add）操作
#define GGML_F32x4_FMA(a, b, c) vec_madd(b, c, a)
// 定义 F32x4 向量加法
#define GGML_F32x4_ADD          vec_add
// 定义 F32x4 向量乘法
#define GGML_F32x4_MUL          vec_mul
// 定义 F32x4 向量归约
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

// 定义 F32 向量为 F32x4
#define GGML_F32_VEC        GGML_F32x4
// 定义 F32 零向量为 F32x4 零向量
#define GGML_F32_VEC_ZERO   GGML_F32x4_ZERO
// 定义 F32 设置所有元素为相同值为 F32x4 设置所有元素为相同值
#define GGML_F32_VEC_SET1   GGML_F32x4_SET1
// 定义 F32 从内存加载数据为 F32x4 从内存加载数据
#define GGML_F32_VEC_LOAD   GGML_F32x4_LOAD
// 定义 F32 存储数据到内存为 F32x4 存储数据到内存
#define GGML_F32_VEC_STORE  GGML_F32x4_STORE
// 定义 F32 浮点数 FMA（fused multiply-add）操作为 F32x4 浮点数 FMA（fused multiply-add）操作
#define GGML_F32_VEC_FMA    GGML_F32x4_FMA
// 定义 F32 向量加法为 F32x4 向量加法
#define GGML_F32_VEC_ADD    GGML_F32x4_ADD
// 定义 F32 向量乘法为 F32x4 向量乘法
#define GGML_F32_VEC_MUL    GGML_F32x4_MUL
// 定义 F32 向量归约为 F32x4 向量归约
#define GGML_F32_VEC_REDUCE GGML_F32x4_REDUCE

// F16 POWER9
// 定义 F16 步长为 F32 步长
#define GGML_F16_STEP       GGML_F32_STEP
// 将 GGML_F16_EPR 定义为 GGML_F32_EPR
#define GGML_F16_EPR        GGML_F32_EPR
// 将 GGML_F16_VEC 定义为 GGML_F32x4
#define GGML_F16_VEC        GGML_F32x4
// 将 GGML_F16_VEC_ZERO 定义为 GGML_F32x4_ZERO
#define GGML_F16_VEC_ZERO   GGML_F32x4_ZERO
// 将 GGML_F16_VEC_SET1 定义为 GGML_F32x4_SET1
#define GGML_F16_VEC_SET1   GGML_F32x4_SET1
// 将 GGML_F16_VEC_FMA 定义为 GGML_F32x4_FMA
#define GGML_F16_VEC_FMA    GGML_F32x4_FMA
// 将 GGML_F16_VEC_REDUCE 定义为 GGML_F32x4_REDUCE
#define GGML_F16_VEC_REDUCE GGML_F32x4_REDUCE
// 在加载地址未对齐的情况下，使用 vec_xl 而不是 vec_ld
// 定义 GGML_F16_VEC_LOAD 宏
#define GGML_F16_VEC_LOAD(p, i) (i & 0x1) ?                   \
  vec_extract_fp32_from_shorth(vec_xl(0, p - GGML_F16_EPR)) : \
  vec_extract_fp32_from_shortl(vec_xl(0, p))
// 定义 GGML_ENDIAN_BYTE 宏
#define GGML_ENDIAN_BYTE(i) ((unsigned char *)&(uint16_t){1})[i]
// 定义 GGML_F16_VEC_STORE 宏
#define GGML_F16_VEC_STORE(p, r, i)                             \
  if (i & 0x1)                                                  \
    vec_xst(vec_pack_to_short_fp32(r[i - GGML_ENDIAN_BYTE(1)],  \
                                   r[i - GGML_ENDIAN_BYTE(0)]), \
            0, p - GGML_F16_EPR)

#elif defined(__wasm_simd128__)

// 定义 GGML_SIMD 宏
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
// 定义 GGML_F32x4_REDUCE 宏
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
    # 遍历从 0 到 offset 的范围
    for (int i = 0; i < offset; ++i) {             \
        # 将 x[i] 和 x[offset+i] 对应位置的元素相加，并将结果存入 x[i]
        x[i] = wasm_f32x4_add(x[i], x[offset+i]);  \
    }                                              \
    # 将 x[0] 中的四个元素提取出来，并相加得到最终结果
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

// 定义内联函数 __wasm_f16x4_load，将 ggml_fp16_t 类型的指针转换为 v128_t 类型
inline static v128_t __wasm_f16x4_load(const ggml_fp16_t * p) {
    float tmp[4];

    // 将 ggml_fp16_t 类型的数据转换为 float 类型
    tmp[0] = GGML_FP16_TO_FP32(p[0]);
    tmp[1] = GGML_FP16_TO_FP32(p[1]);
    tmp[2] = GGML_FP16_TO_FP32(p[2]);
    tmp[3] = GGML_FP16_TO_FP32(p[3]);

    // 将转换后的数据加载到 v128_t 类型中并返回
    return wasm_v128_load(tmp);
}

// 定义内联函数 __wasm_f16x4_store，将 v128_t 类型转换为 ggml_fp16_t 类型的指针
inline static void __wasm_f16x4_store(ggml_fp16_t * p, v128_t x) {
    float tmp[4];

    // 将 v128_t 类型的数据转换为 float 类型
    wasm_v128_store(tmp, x);

    // 将转换后的数据存储到 ggml_fp16_t 类型的指针中
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
// 定义 GGML_F16x4_REDUCE(res, x) 为 对 x 进行归约操作
#define GGML_F16x4_REDUCE(res, x)                  \
{                                                  \
    int offset = GGML_F16_ARR >> 1;                \
    for (int i = 0; i < offset; ++i) {             \
        x[i] = wasm_f32x4_add(x[i], x[offset+i]);  \
    }                                              \
    offset >>= 1;                                  \
    for (int i = 0; i < offset; ++i) {             \
        x[i] = wasm_f32x4_add(x[i], x[offset+i]);  \
    }                                              \
    offset >>= 1;                                  \
    # 遍历从 0 到 offset 的范围
    for (int i = 0; i < offset; ++i) {             \
        # 将 x[i] 和 x[offset+i] 对应位置的元素相加，并将结果存入 x[i]
        x[i] = wasm_f32x4_add(x[i], x[offset+i]);  \
    }                                              \
    # 将 x[0] 中的四个元素提取出来，并相加得到最终结果
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
#define GGML_F16_VEC_REDUCE         GGML_F16x4_REDUCE

// 如果定义了 __SSE3__，则执行以下代码
#elif defined(__SSE3__)

// 定义 GGML_SIMD
#define GGML_SIMD

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
// 如果定义了 __FMA__，则定义 GGML_F32x4_FMA 为 _mm_fmadd_ps(b, c, a)，否则为 _mm_add_ps(_mm_mul_ps(b, c), a)
#if defined(__FMA__)
    #define GGML_F32x4_FMA(a, b, c) _mm_fmadd_ps(b, c, a)
#else
    #define GGML_F32x4_FMA(a, b, c) _mm_add_ps(_mm_mul_ps(b, c), a)
#endif
// 定义 GGML_F32x4_ADD 为 _mm_add_ps
#define GGML_F32x4_ADD     _mm_add_ps
// 定义 GGML_F32x4_MUL 为 _mm_mul_ps
#define GGML_F32x4_MUL     _mm_mul_ps
// 定义 GGML_F32x4_REDUCE(res, x) 为 SIMD 归约操作
#define GGML_F32x4_REDUCE(res, x)                                 \
{                                                                 \
    int offset = GGML_F32_ARR >> 1;                               \
    for (int i = 0; i < offset; ++i) {                            \
        x[i] = _mm_add_ps(x[i], x[offset+i]);                     \
    }                                                             \
    offset >>= 1;                                                 \
    for (int i = 0; i < offset; ++i) {                            \
        x[i] = _mm_add_ps(x[i], x[offset+i]);                     \
    }                                                             \
    offset >>= 1;                                                 \
    for (int i = 0; i < offset; ++i) {                            \
        x[i] = _mm_add_ps(x[i], x[offset+i]);                     \
    }                                                             \
    # 计算参数 x[0] 中每个元素的水平加法
    const __m128 t0 = _mm_hadd_ps(x[0], x[0]);                    \
    # 对 t0 中的元素进行水平加法，并将结果转换为标量
    res = _mm_cvtss_f32(_mm_hadd_ps(t0, t0));                     \
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

// 定义 __sse_f16x4_load 函数，将 ggml_fp16_t 类型的指针转换为 __m128 类型
static inline __m128 __sse_f16x4_load(ggml_fp16_t *x) {
    // 创建临时数组 tmp，存储转换后的数据
    float tmp[4];
    // 将 ggml_fp16_t 类型的数据转换为 float 类型，并存入 tmp 数组
    tmp[0] = GGML_FP16_TO_FP32(x[0]);
    tmp[1] = GGML_FP16_TO_FP32(x[1]);
    tmp[2] = GGML_FP16_TO_FP32(x[2]);
    tmp[3] = GGML_FP16_TO_FP32(x[3]);
    // 返回转换后的数据作为 __m128 类型
    return _mm_loadu_ps(tmp);
}

// 定义 __sse_f16x4_store 函数，将 __m128 类型的数据存储到 ggml_fp16_t 类型的指针中
static inline void __sse_f16x4_store(ggml_fp16_t *x, __m128 y) {
    // 创建临时数组 arr，存储转换后的数据
    float arr[4];
    // 将 __m128 类型的数据转换为 float 类型，并存入 arr 数组
    _mm_storeu_ps(arr, y);
    // 将转换后的数据存入 ggml_fp16_t 类型的指针中
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
// 定义每个步骤使用的寄存器数量
#ifdef GGML_SIMD
#define GGML_F32_ARR (GGML_F32_STEP/GGML_F32_EPR)
#define GGML_F16_ARR (GGML_F16_STEP/GGML_F16_EPR)
#endif

//
// 基本操作
//

// 设置 int8_t 类型数组的值
inline static void ggml_vec_set_i8(const int n, int8_t * x, const int8_t v) { for (int i = 0; i < n; ++i) x[i] = v; }

// 设置 int16_t 类型数组的值
inline static void ggml_vec_set_i16(const int n, int16_t * x, const int16_t v) { for (int i = 0; i < n; ++i) x[i] = v; }

// 设置 int32_t 类型数组的值
inline static void ggml_vec_set_i32(const int n, int32_t * x, const int32_t v) { for (int i = 0; i < n; ++i) x[i] = v; }

// 设置 ggml_fp16_t 类型数组的值
inline static void ggml_vec_set_f16(const int n, ggml_fp16_t * x, const int32_t v) { for (int i = 0; i < n; ++i) x[i] = v; }

// 将两个 float 类型数组对应位置的值相加，结果存入第三个数组
inline static void ggml_vec_add_f32 (const int n, float * z, const float * x, const float * y) { for (int i = 0; i < n; ++i) z[i]  = x[i] + y[i]; }

// 将一个 float 类型数组和一个常量值相加，结果存入另一个数组
inline static void ggml_vec_add1_f32(const int n, float * z, const float * x, const float   v) { for (int i = 0; i < n; ++i) z[i]  = x[i] + v;    }

// 将一个 float 类型数组和另一个数组对应位置的值相加，结果存入第一个数组
inline static void ggml_vec_acc_f32 (const int n, float * y, const float * x)                  { for (int i = 0; i < n; ++i) y[i] += x[i];        }

// 将一个 float 类型数组和一个常量值相加，结果存入第一个数组
inline static void ggml_vec_acc1_f32(const int n, float * y, const float   v)                  { for (int i = 0; i < n; ++i) y[i] += v;           }

// 将两个 float 类型数组对应位置的值相减，结果存入第三个数组
inline static void ggml_vec_sub_f32 (const int n, float * z, const float * x, const float * y) { for (int i = 0; i < n; ++i) z[i]  = x[i] - y[i]; }

// 设置 float 类型数组的值为常量
inline static void ggml_vec_set_f32 (const int n, float * x, const float   v)                  { for (int i = 0; i < n; ++i) x[i]  = v;           }

// 复制一个 float 类型数组的值到另一个数组
inline static void ggml_vec_cpy_f32 (const int n, float * y, const float * x)                  { for (int i = 0; i < n; ++i) y[i]  = x[i];        }

// 将一个 float 类型数组的值取反，结果存入另一个数组
inline static void ggml_vec_neg_f32 (const int n, float * y, const float * x)                  { for (int i = 0; i < n; ++i) y[i]  = -x[i];       }
// 用于计算两个 float 数组的元素对应位置相乘，结果存入第三个数组
inline static void ggml_vec_mul_f32 (const int n, float * z, const float * x, const float * y) { for (int i = 0; i < n; ++i) z[i]  = x[i]*y[i];   }
// 用于计算两个 float 数组的元素对应位置相除，结果存入第三个数组
inline static void ggml_vec_div_f32 (const int n, float * z, const float * x, const float * y) { for (int i = 0; i < n; ++i) z[i]  = x[i]/y[i];   }

// 用于计算两个 float 数组的点积
static void ggml_vec_dot_f32(const int n, float * restrict s, const float * restrict x, const float * restrict y) {
#ifdef GGML_SIMD
    float sumf = 0.0f;
    const int np = (n & ~(GGML_F32_STEP - 1));

    GGML_F32_VEC sum[GGML_F32_ARR] = { GGML_F32_VEC_ZERO };

    GGML_F32_VEC ax[GGML_F32_ARR];
    GGML_F32_VEC ay[GGML_F32_ARR];

    for (int i = 0; i < np; i += GGML_F32_STEP) {
        for (int j = 0; j < GGML_F32_ARR; j++) {
            ax[j] = GGML_F32_VEC_LOAD(x + i + j*GGML_F32_EPR);
            ay[j] = GGML_F32_VEC_LOAD(y + i + j*GGML_F32_EPR);

            sum[j] = GGML_F32_VEC_FMA(sum[j], ax[j], ay[j]);
        }
    }

    // 将 sum0..sum3 求和得到 sum0
    GGML_F32_VEC_REDUCE(sumf, sum);

    // 处理剩余的元素
    for (int i = np; i < n; ++i) {
        sumf += x[i]*y[i];
    }
#else
    // 标量计算
    ggml_float sumf = 0.0;
    for (int i = 0; i < n; ++i) {
        sumf += (ggml_float)(x[i]*y[i]);
    }
#endif

    *s = sumf;
}

// 用于计算两个 ggml_fp16_t 数组的点积
static void ggml_vec_dot_f16(const int n, float * restrict s, ggml_fp16_t * restrict x, ggml_fp16_t * restrict y) {
    ggml_float sumf = 0.0;

#if defined(GGML_SIMD)
    const int np = (n & ~(GGML_F16_STEP - 1));

    GGML_F16_VEC sum[GGML_F16_ARR] = { GGML_F16_VEC_ZERO };

    GGML_F16_VEC ax[GGML_F16_ARR];
    GGML_F16_VEC ay[GGML_F16_ARR];

    for (int i = 0; i < np; i += GGML_F16_STEP) {
        for (int j = 0; j < GGML_F16_ARR; j++) {
            ax[j] = GGML_F16_VEC_LOAD(x + i + j*GGML_F16_EPR, j);
            ay[j] = GGML_F16_VEC_LOAD(y + i + j*GGML_F16_EPR, j);

            sum[j] = GGML_F16_VEC_FMA(sum[j], ax[j], ay[j]);
        }
    }

    // 将 sum0..sum3 求和得到 sum0
    GGML_F16_VEC_REDUCE(sumf, sum);

    // 处理剩余的元素
    # 从 np 开始遍历到 n-1
    for (int i = np; i < n; ++i) {
        # 将 x[i] 和 y[i] 转换为 32 位浮点数，相乘后累加到 sumf 中
        sumf += (ggml_float)(GGML_FP16_TO_FP32(x[i])*GGML_FP16_TO_FP32(y[i]));
    }
// 如果不满足条件，执行以下代码块
#else
    // 对每个元素进行循环
    for (int i = 0; i < n; ++i) {
        // 计算 x[i] 和 y[i] 的乘积，并将结果加到 sumf 中
        sumf += (ggml_float)(GGML_FP16_TO_FP32(x[i])*GGML_FP16_TO_FP32(y[i]));
    }
#endif

    // 将 sumf 的值赋给指针 s 所指向的位置
    *s = sumf;
}

// 一次计算 GGML_VEC_DOT_UNROLL 个点积
// xs - x 的行跨度（以字节为单位）
inline static void ggml_vec_dot_f16_unroll(const int n, const int xs, float * restrict s, void * restrict xv, ggml_fp16_t * restrict y) {
    // 初始化 GGML_VEC_DOT_UNROLL 个元素的 sumf 数组，全部赋值为 0.0
    ggml_float sumf[GGML_VEC_DOT_UNROLL] = { 0.0 };

    // 初始化 GGML_VEC_DOT_UNROLL 个指向 ggml_fp16_t 类型的指针数组 x
    ggml_fp16_t * restrict x[GGML_VEC_DOT_UNROLL];

    // 对 GGML_VEC_DOT_UNROLL 个元素进行循环
    for (int i = 0; i < GGML_VEC_DOT_UNROLL; ++i) {
        // 将 xv 的地址加上 i*xs，得到 x[i] 的地址
        x[i] = (ggml_fp16_t *) ((char *) xv + i*xs);
    }

    // 如果定义了 GGML_SIMD
#if defined(GGML_SIMD)
    // 计算 np，即 n 与 GGML_F16_STEP - 1 的按位取反与运算的结果
    const int np = (n & ~(GGML_F16_STEP - 1));

    // 初始化 GGML_VEC_DOT_UNROLL * GGML_F16_ARR 个元素的 sum 数组，全部赋值为 GGML_F16_VEC_ZERO
    GGML_F16_VEC sum[GGML_VEC_DOT_UNROLL][GGML_F16_ARR] = { { GGML_F16_VEC_ZERO } };

    // 初始化 GGML_F16_ARR 个 GGML_F16_VEC 类型的数组 ax 和 ay
    GGML_F16_VEC ax[GGML_F16_ARR];
    GGML_F16_VEC ay[GGML_F16_ARR];

    // 对 np 个元素进行循环
    for (int i = 0; i < np; i += GGML_F16_STEP) {
        // 对 GGML_F16_ARR 个元素进行循环
        for (int j = 0; j < GGML_F16_ARR; j++) {
            // 从 y 中加载 GGML_F16_EPR 个元素到 ay[j] 中
            ay[j] = GGML_F16_VEC_LOAD(y + i + j*GGML_F16_EPR, j);

            // 对 GGML_VEC_DOT_UNROLL 个元素进行循环
            for (int k = 0; k < GGML_VEC_DOT_UNROLL; ++k) {
                // 从 x[k] 中加载 GGML_F16_EPR 个元素到 ax[j] 中
                ax[j] = GGML_F16_VEC_LOAD(x[k] + i + j*GGML_F16_EPR, j);

                // 计算 sum[k][j] = sum[k][j] + ax[j] * ay[j]
                sum[k][j] = GGML_F16_VEC_FMA(sum[k][j], ax[j], ay[j]);
            }
        }
    }

    // 将 sum0..sum3 合并到 sum0 中
    for (int k = 0; k < GGML_VEC_DOT_UNROLL; ++k) {
        GGML_F16_VEC_REDUCE(sumf[k], sum[k]);
    }

    // 处理剩余的元素
    for (int i = np; i < n; ++i) {
        // 对 GGML_VEC_DOT_UNROLL 个元素进行循环
        for (int j = 0; j < GGML_VEC_DOT_UNROLL; ++j) {
            // 计算 x[j][i] 和 y[i] 的乘积，并将结果加到 sumf[j] 中
            sumf[j] += (ggml_float)(GGML_FP16_TO_FP32(x[j][i])*GGML_FP16_TO_FP32(y[i]));
        }
    }
#else
    // 如果未定义 GGML_SIMD
    // 对 n 个元素进行循环
    for (int i = 0; i < n; ++i) {
        // 对 GGML_VEC_DOT_UNROLL 个元素进行循环
        for (int j = 0; j < GGML_VEC_DOT_UNROLL; ++j) {
            // 计算 x[j][i] 和 y[i] 的乘积，并将结果加到 sumf[j] 中
            sumf[j] += (ggml_float)(GGML_FP16_TO_FP32(x[j][i])*GGML_FP16_TO_FP32(y[i]));
        }
    }
#endif

    // 对 GGML_VEC_DOT_UNROLL 个元素进行循环
    for (int i = 0; i < GGML_VEC_DOT_UNROLL; ++i) {
        // 将 sumf[i] 的值赋给指针 s[i] 所指向的位置
        s[i] = sumf[i];
    }
}

// 计算 GGML_VEC_DOT_UNROLL 个 GGML_FP16_TO_FP32(x[i])*GGML_FP16_TO_FP32(y[i]) + v
inline static void ggml_vec_mad_f32(const int n, float * restrict y, const float * restrict x, const float v) {
    // 如果定义了 GGML_SIMD
#if defined(GGML_SIMD)
    // 计算 np，np 是 n 和 GGML_F32_STEP - 1 按位取反后的结果
    const int np = (n & ~(GGML_F32_STEP - 1));

    // 使用 v 创建一个 GGML_F32_VEC 类型的向量 vx
    GGML_F32_VEC vx = GGML_F32_VEC_SET1(v);

    // 创建 GGML_F32_ARR 个 GGML_F32_VEC 类型的向量数组 ax 和 ay
    GGML_F32_VEC ax[GGML_F32_ARR];
    GGML_F32_VEC ay[GGML_F32_ARR];

    // 循环遍历 np，每次增加 GGML_F32_STEP
    for (int i = 0; i < np; i += GGML_F32_STEP) {
        // 内层循环遍历 GGML_F32_ARR
        for (int j = 0; j < GGML_F32_ARR; j++) {
            // 从 x 数组中加载数据到 ax[j] 向量
            ax[j] = GGML_F32_VEC_LOAD(x + i + j*GGML_F32_EPR);
            // 从 y 数组中加载数据到 ay[j] 向量
            ay[j] = GGML_F32_VEC_LOAD(y + i + j*GGML_F32_EPR);
            // 计算 ay[j] = ay[j] + ax[j] * vx
            ay[j] = GGML_F32_VEC_FMA(ay[j], ax[j], vx);
            // 将 ay[j] 向量中的数据存储回 y 数组中
            GGML_F32_VEC_STORE(y + i + j*GGML_F32_EPR, ay[j]);
        }
    }

    // 处理剩余的数据
    for (int i = np; i < n; ++i) {
        // 计算 y[i] = y[i] + x[i] * v
        y[i] += x[i]*v;
    }
#else
    // 如果不满足条件，使用标量方式进行计算
    for (int i = 0; i < n; ++i) {
        y[i] += x[i]*v;
    }
#endif
}

// xs and vs are byte strides of x and v
// 对于给定的 x 和 v，xs 和 vs 分别是它们的字节步长
inline static void ggml_vec_mad_f32_unroll(const int n, const int xs, const int vs, float * restrict y, const float * restrict xv, const float * restrict vv) {

    const float * restrict x[GGML_VEC_MAD_UNROLL];
    const float * restrict v[GGML_VEC_MAD_UNROLL];

    for (int i = 0; i < GGML_VEC_MAD_UNROLL; ++i) {
        x[i] = (const float *) ((const char *) xv + i*xs);
        v[i] = (const float *) ((const char *) vv + i*vs);
    }

#if defined(GGML_SIMD)
    // 计算 n 的最大倍数
    const int np = (n & ~(GGML_F32_STEP - 1));

    GGML_F32_VEC vx[GGML_VEC_MAD_UNROLL];

    for (int k = 0; k < GGML_VEC_MAD_UNROLL; ++k) {
        vx[k] = GGML_F32_VEC_SET1(v[k][0]);
    }

    GGML_F32_VEC ax[GGML_VEC_MAD_UNROLL][GGML_F32_ARR];
    GGML_F32_VEC ay[GGML_F32_ARR];

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

    // 处理剩余的部分
    for (int k = 0; k < GGML_VEC_MAD_UNROLL; ++k) {
        for (int i = np; i < n; ++i) {
            y[i] += x[k][i]*v[k][0];
        }
    }
#else
    // 如果不满足条件，使用标量方式进行计算
    for (int k = 0; k < GGML_VEC_MAD_UNROLL; ++k) {
        for (int i = 0; i < n; ++i) {
            y[i] += x[k][i]*v[k][0];
        }
    }
#endif
}

// 对给定的 y 数组中的每个元素乘以常数 v
inline static void ggml_vec_scale_f32(const int n, float * y, const float   v) {
#if defined(GGML_USE_ACCELERATE)
    // 使用加速库进行乘法操作
    vDSP_vsmul(y, 1, &v, y, 1, n);
#elif defined(GGML_SIMD)
    # 计算 n 与 GGML_F32_STEP - 1 的按位与，得到 np
    const int np = (n & ~(GGML_F32_STEP - 1));

    # 使用 v 创建一个 GGML_F32_VEC 向量
    GGML_F32_VEC vx = GGML_F32_VEC_SET1(v);

    # 创建一个长度为 GGML_F32_ARR 的 GGML_F32_VEC 数组
    GGML_F32_VEC ay[GGML_F32_ARR];

    # 遍历 np 范围内的数据，每次步长为 GGML_F32_STEP
    for (int i = 0; i < np; i += GGML_F32_STEP) {
        # 遍历 GGML_F32_ARR 范围内的数据
        for (int j = 0; j < GGML_F32_ARR; j++) {
            # 从数组 y 中加载数据到 ay[j] 中
            ay[j] = GGML_F32_VEC_LOAD(y + i + j*GGML_F32_EPR);
            # 将 ay[j] 中的数据与 vx 中的数据逐个相乘
            ay[j] = GGML_F32_VEC_MUL(ay[j], vx);
            # 将 ay[j] 中的数据存储回数组 y 中
            GGML_F32_VEC_STORE(y + i + j*GGML_F32_EPR, ay[j]);
        }
    }

    # 处理剩余的数据
    for (int i = np; i < n; ++i) {
        # 将 y[i] 乘以 v
        y[i] *= v;
    }
// 如果条件不成立，则执行以下代码块
#else
    // 对于每个元素，将其乘以标量 v
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
// 计算向量的ELU函数
inline static void ggml_vec_elu_f32  (const int n, float * y, const float * x) { for (int i = 0; i < n; ++i) y[i] = (x[i] > 0.f) ? x[i] : expf(x[i])-1; }
// 计算向量的ReLU函数
inline static void ggml_vec_relu_f32 (const int n, float * y, const float * x) { for (int i = 0; i < n; ++i) y[i] = (x[i] > 0.f) ? x[i] : 0.f; }
// 计算向量的Leaky ReLU函数
inline static void ggml_vec_leaky_f32 (const int n, float * y, const float * x) { for (int i = 0; i < n; ++i) y[i] = (x[i] > 0.f) ? x[i] : 0.1f*x[i]; }

// 定义常量 GELU_COEF_A 的值
static const float GELU_COEF_A     = 0.044715f;
// 定义常量 GELU_QUICK_COEF 的值
static const float GELU_QUICK_COEF = -1.702f;
// 定义常量 SQRT_2_OVER_PI 的值
static const float SQRT_2_OVER_PI  = 0.79788456080286535587989211986876f;

// 计算GELU函数
inline static float ggml_gelu_f32(float x) {
    return 0.5f*x*(1.0f + tanhf(SQRT_2_OVER_PI*x*(1.0f + GELU_COEF_A*x*x)));
}
// 对输入的半精度浮点数数组进行 GELU 激活函数计算，结果存储在输出数组中
inline static void ggml_vec_gelu_f16(const int n, ggml_fp16_t * y, const ggml_fp16_t * x) {
    // 将输入数组转换为无符号 16 位整数数组
    const uint16_t * i16 = (const uint16_t *) x;
    // 遍历输入数组，根据索引从预先计算好的 GELU 表中获取值，存储到输出数组中
    for (int i = 0; i < n; ++i) {
        y[i] = ggml_table_gelu_f16[i16[i]];
    }
}

#ifdef GGML_GELU_FP16
// 对输入的单精度浮点数数组进行 GELU 激活函数计算，结果存储在输出数组中
inline static void ggml_vec_gelu_f32(const int n, float * y, const float * x) {
    uint16_t t;
    // 遍历输入数组，将每个单精度浮点数转换为半精度浮点数，然后根据索引从预先计算好的 GELU 表中获取值，存储到输出数组中
    for (int i = 0; i < n; ++i) {
        ggml_fp16_t fp16 = GGML_FP32_TO_FP16(x[i]);
        memcpy(&t, &fp16, sizeof(uint16_t));
        y[i] = GGML_FP16_TO_FP32(ggml_table_gelu_f16[t]);
    }
#else
// 对输入的单精度浮点数数组进行 GELU 激活函数计算，结果存储在输出数组中
inline static void ggml_vec_gelu_f32(const int n, float * y, const float * x) {
    // 遍历输入数组，对每个单精度浮点数进行 GELU 激活函数计算，结果存储到输出数组中
    for (int i = 0; i < n; ++i) {
        y[i] = ggml_gelu_f32(x[i]);
    }
#endif

// 对输入的单精度浮点数进行快速 GELU 激活函数计算
inline static float ggml_gelu_quick_f32(float x) {
    return x*(1.0f/(1.0f+expf(GELU_QUICK_COEF*x)));
}

#ifdef GGML_GELU_QUICK_FP16
// 对输入的单精度浮点数数组进行快速 GELU 激活函数计算，结果存储在输出数组中
inline static void ggml_vec_gelu_quick_f32(const int n, float * y, const float * x) {
    uint16_t t;
    // 遍历输入数组，将每个单精度浮点数转换为半精度浮点数，然后根据索引从预先计算好的快速 GELU 表中获取值，存储到输出数组中
    for (int i = 0; i < n; ++i) {
        ggml_fp16_t fp16 = GGML_FP32_TO_FP16(x[i]);
        memcpy(&t, &fp16, sizeof(uint16_t));
        y[i] = GGML_FP16_TO_FP32(ggml_table_gelu_quick_f16[t]);
    }
#else
// 对输入的单精度浮点数数组进行快速 GELU 激活函数计算，结果存储在输出数组中
inline static void ggml_vec_gelu_quick_f32(const int n, float * y, const float * x) {
    // 遍历输入数组，对每个单精度浮点数进行快速 GELU 激活函数计算，结果存储到输出数组中
    for (int i = 0; i < n; ++i) {
        y[i] = ggml_gelu_quick_f32(x[i]);
    }
#endif

// Sigmoid Linear Unit (SiLU) 函数
inline static float ggml_silu_f32(float x) {
    return x/(1.0f + expf(-x));
}
// 计算输入数组 x 的前 n 个元素的 Sigmoid-Weighted Linear Unit (SiLU) 函数值，存储到数组 y 中
inline static void ggml_vec_silu_f32(const int n, float * y, const float * x) {
    uint16_t t;
    // 遍历数组 x 的前 n 个元素
    for (int i = 0; i < n; ++i) {
        // 将 x[i] 转换为半精度浮点数
        ggml_fp16_t fp16 = GGML_FP32_TO_FP16(x[i]);
        // 将半精度浮点数转换为 16 位整数
        memcpy(&t, &fp16, sizeof(uint16_t));
        // 使用预先计算好的 SiLU 函数值表，计算 y[i] 的值
        y[i] = GGML_FP16_TO_FP32(ggml_table_silu_f16[t]);
    }
}
#else
// 如果没有启用半精度浮点数支持，则直接计算 SiLU 函数值
inline static void ggml_vec_silu_f32(const int n, float * y, const float * x) {
    for (int i = 0; i < n; ++i) {
        y[i] = ggml_silu_f32(x[i]);
    }
}
#endif

// 计算 SiLU 函数的反向传播值
inline static float ggml_silu_backward_f32(float x, float dy) {
    // 计算 SiLU 函数值
    const float s = 1.0f/(1.0f + expf(-x));
    // 计算 SiLU 函数的反向传播值
    return dy*s*(1.0f + x*(1.0f - s));
}

#ifdef GGML_SILU_FP16
// 如果启用了半精度浮点数支持，则计算 SiLU 函数的反向传播值
inline static void ggml_vec_silu_backward_f32(const int n, float * dx, const float * x, const float * dy) {
    for (int i = 0; i < n; ++i) {
        // 使用 x[i] 的 f16 等价值来计算前向 SiLU 函数值的导数
        ggml_fp16_t fp16 = GGML_FP32_TO_FP16(x[i]);
        float usedx = GGML_FP16_TO_FP32(fp16);
        dx[i] = ggml_silu_backward_f32(usedx, dy[i]);
    }
}
#else
// 如果没有启用半精度浮点数支持，则直接计算 SiLU 函数的反向传播值
inline static void ggml_vec_silu_backward_f32(const int n, float * dx, const float * x, const float * dy) {
    for (int i = 0; i < n; ++i) {
        dx[i] = ggml_silu_backward_f32(x[i], dy[i]);
    }
}
#endif

// 计算输入数组 x 的前 n 个元素的和，存储到 s 中
inline static void ggml_vec_sum_f32(const int n, float * s, const float * x) {
#ifndef GGML_USE_ACCELERATE
    ggml_float sum = 0.0;
    // 遍历数组 x 的前 n 个元素，计算它们的和
    for (int i = 0; i < n; ++i) {
        sum += (ggml_float)x[i];
    }
    // 将计算得到的和存储到 s 中
    *s = sum;
#else
    // 使用加速库计算数组 x 的前 n 个元素的和
    vDSP_sve(x, 1, s, n);
#endif
}

// 计算输入数组 x 的前 n 个元素的和，存储到 s 中（使用 ggml_float 类型）
inline static void ggml_vec_sum_f32_ggf(const int n, ggml_float * s, const float * x) {
    ggml_float sum = 0.0;
    // 遍历数组 x 的前 n 个元素，计算它们的和
    for (int i = 0; i < n; ++i) {
        sum += (ggml_float)x[i];
    }
    // 将计算得到的和存储到 s 中
    *s = sum;
}

// 计算输入数组 x 的前 n 个元素的和，存储到 s 中（使用半精度浮点数类型）
inline static void ggml_vec_sum_f16_ggf(const int n, float * s, const ggml_fp16_t * x) {
    float sum = 0.0f;
    // 遍历数组 x 的前 n 个元素，将其转换为单精度浮点数后计算它们的和
    for (int i = 0; i < n; ++i) {
        sum += GGML_FP16_TO_FP32(x[i]);
    }
    // 将计算得到的和存储到 s 中
    *s = sum;
}
# 计算整型数组的和
inline static void ggml_vec_sum_i32_ggf(const int n, int64_t * s, const int32_t * x) {
    int64_t sum = 0;  # 初始化和为0
    for (int i = 0; i < n; ++i) {  # 遍历整型数组
        sum += (int64_t)x[i];  # 将数组元素累加到和中
    }
    *s = sum;  # 将和赋值给输出参数
}

# 计算浮点型数组的最大值
inline static void ggml_vec_max_f32(const int n, float * s, const float * x) {
#ifndef GGML_USE_ACCELERATE
    float max = -INFINITY;  # 初始化最大值为负无穷
    for (int i = 0; i < n; ++i) {  # 遍历浮点型数组
        max = MAX(max, x[i]);  # 更新最大值
    }
    *s = max;  # 将最大值赋值给输出参数
#else
    vDSP_maxv(x, 1, s, n);  # 使用加速库计算最大值
#endif
}

# 计算浮点型数组的逆范数
inline static void ggml_vec_norm_inv_f32(const int n, float * s, const float * x) {
    ggml_vec_norm_f32(n, s, x);  # 调用计算范数的函数
    *s = 1.f/(*s);  # 计算逆范数并赋值给输出参数
}

# 计算浮点型数组的最大值索引
inline static void ggml_vec_argmax_f32(const int n, int * s, const float * x) {
    float max = -INFINITY;  # 初始化最大值为负无穷
    int idx = 0;  # 初始化最大值索引为0
    for (int i = 0; i < n; ++i) {  # 遍历浮点型数组
        max = MAX(max, x[i]);  # 更新最大值
        if (max == x[i]) { idx = i; }  # 更新最大值索引
    }
    *s = idx;  # 将最大值索引赋值给输出参数
}

# 定义操作名称数组
static const char * GGML_OP_NAME[GGML_OP_COUNT] = {
    "NONE",  # 空操作

    "DUP",  # 复制
    "ADD",  # 加法
    "ADD1",  # 加1
    "ACC",  # 累加
    "SUB",  # 减法
    "MUL",  # 乘法
    "DIV",  # 除法
    "SQR",  # 平方
    "SQRT",  # 平方根
    "LOG",  # 对数
    "SUM",  # 求和
    "SUM_ROWS",  # 按行求和
    "MEAN",  # 平均值
    "ARGMAX",  # 最大值索引
    "REPEAT",  # 重复
    "REPEAT_BACK",  # 反向重复
    "CONCAT",  # 连接
    "SILU_BACK",  # SILU反向
    "NORM",  # 范数
    "RMS_NORM",  # 均方根范数
    "RMS_NORM_BACK",  # 均方根范数反向
    "GROUP_NORM",  # 分组范数

    "MUL_MAT",  # 矩阵乘法
    "AXPY",  # AXPY操作
    "OUT_PROD",  # 外积

    "SCALE",  # 缩放
    "SET",  # 设置
    "CPY",  # 复制
    "CONT",  # 连续
    "RESHAPE",  # 重塑
    "VIEW",  # 视图
    "PERMUTE",  # 排列
    "TRANSPOSE",  # 转置
    "GET_ROWS",  # 获取行
    "GET_ROWS_BACK",  # 获取行反向
    "DIAG",  # 对角线
    "DIAG_MASK_INF",  # 对角线掩码（无穷）
    "DIAG_MASK_ZERO",  # 对角线掩码（零）
    "SOFT_MAX",  # Softmax
    "SOFT_MAX_BACK",  # Softmax反向
    "ROPE",  # 绳索
    "ROPE_BACK",  # 绳索反向
    "ALIBI",  # ALIBI
    "CLAMP",  # 夹紧
    "CONV_TRANSPOSE_1D",  # 转置卷积1D
    "IM2COL",  # 图像到列
    "CONV_TRANSPOSE_2D",  # 转置卷积2D
    "POOL_1D",  # 池化1D
    "POOL_2D",  # 池化2D
    "UPSCALE",  # 放大

    "FLASH_ATTN",  # 闪光注意力
    "FLASH_FF",  # 闪光前馈
    "FLASH_ATTN_BACK",  # 闪光注意力反向
    "WIN_PART",  # 窗口部分
    "WIN_UNPART",  # 窗口非部分
    "GET_REL_POS",  # 获取相对位置
    "ADD_REL_POS",  # 添加相对位置

    "UNARY",  # 一元

    "MAP_UNARY",  # 映射一元
    "MAP_BINARY",  # 映射二元

    "MAP_CUSTOM1_F32",  # 映射自定义1（浮点型）
    "MAP_CUSTOM2_F32",  # 映射自定义2（浮点型）
    "MAP_CUSTOM3_F32",  # 映射自定义3（浮点型）

    "MAP_CUSTOM1",  # 映射自定义1
    "MAP_CUSTOM2",  # 映射自定义2
    "MAP_CUSTOM3",  # 映射自定义3
    # 定义交叉熵损失函数的名称
    "CROSS_ENTROPY_LOSS",
    # 定义交叉熵损失函数的反向传播函数的名称
    "CROSS_ENTROPY_LOSS_BACK",
// 定义了 GGML_OP_SYMBOL 数组，包含了所有操作的符号表示
static const char * GGML_OP_SYMBOL[GGML_OP_COUNT] = {
    "none",  // 空操作

    "x",  // x
    "x+y",  // x+y
    "x+y",  // x+y
    "view(x,nb,offset)+=y->x",  // 视图操作
    "x-y",  // x-y
    "x*y",  // x*y
    "x/y",  // x/y
    "x^2",  // x的平方
    "√x",  // x的平方根
    "log(x)",  // x的自然对数
    "Σx",  // x的求和
    "Σx_k",  // x的求和
    "Σx/n",  // x的求和
    "argmax(x)",  // x的最大值索引
    "repeat(x)",  // 重复操作
    "repeat_back(x)",  // 重复操作
    "concat(x, y)",  // 连接操作
    "silu_back(x)",  // SILU反向操作
    "norm(x)",  // 范数操作
    "rms_norm(x)",  // 均方根范数操作
    "rms_norm_back(x)",  // 均方根范数反向操作
    "group_norm(x)",  // 分组范数操作
    // ... 其他操作

    "cross_entropy_loss(x,y)",  // 交叉熵损失操作
    "cross_entropy_loss_back(x,y)",  // 交叉熵损失反向操作
};

// 静态断言，确保 GGML_OP_POOL_COUNT 的值为 2
static_assert(GGML_OP_POOL_COUNT == 2, "GGML_OP_POOL_COUNT != 2");

// 静态断言，确保 ggml_object 结构体的大小是 GGML_MEM_ALIGN 的倍数
static_assert(sizeof(struct ggml_object)%GGML_MEM_ALIGN == 0, "ggml_object size must be a multiple of GGML_MEM_ALIGN");
// 静态断言，确保 ggml_tensor 结构体的大小是 GGML_MEM_ALIGN 的倍数
static_assert(sizeof(struct ggml_tensor)%GGML_MEM_ALIGN == 0, "ggml_tensor size must be a multiple of GGML_MEM_ALIGN");

// 警告信息，提醒配置错误可能导致的问题
// * 最好情况下，会导致崩溃或输出无意义的信息
// * 最坏情况下，会导致微小但难以察觉的差异
//
// 当操作需要 INIT 或 FINALIZE 时，必须启用其中一个分支
// 注意编译选项（例如，GGML_USE_xxx）
static bool GGML_OP_HAS_INIT    [GGML_OP_COUNT] = { 0 };
static bool GGML_OP_HAS_FINALIZE[GGML_OP_COUNT] = { 0 };

static void ggml_setup_op_has_task_pass(void) {
    {   // INIT
        bool * p = GGML_OP_HAS_INIT;

        p[GGML_OP_ACC                    ] = true;
        p[GGML_OP_MUL_MAT                ] = true;
        p[GGML_OP_AXPY                   ] = true;
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

struct ggml_context_container {
    bool used;

    struct ggml_context context;
};

//
// NUMA support
//

#define GGML_NUMA_MAX_NODES 8
#define GGML_NUMA_MAX_CPUS 512

struct ggml_numa_node {
    uint32_t cpus[GGML_NUMA_MAX_CPUS]; // 该节点上的硬件线程
    uint32_t n_cpus;
};

struct ggml_numa_nodes {
    struct ggml_numa_node nodes[GGML_NUMA_MAX_NODES];
    uint32_t n_nodes;
    uint32_t total_cpus; // 系统上的硬件线程
};

//
// ggml state
//

struct ggml_state {
    struct ggml_context_container contexts[GGML_MAX_CONTEXTS];
    struct ggml_numa_nodes numa;
};

// 全局状态
static struct ggml_state g_state;
static atomic_int g_state_barrier = 0;

// 通过自旋锁实现的屏障
inline static void ggml_critical_section_start(void) {
    int processing = atomic_fetch_add(&g_state_barrier, 1);
    # 当处理数量大于0时，进入循环
    while (processing > 0) {
        # 等待其他线程完成
        atomic_fetch_sub(&g_state_barrier, 1);
        # 调度器让出CPU，TODO: 重新考虑这一步
        sched_yield(); 
        # 更新处理数量，等待其他线程完成
        processing = atomic_fetch_add(&g_state_barrier, 1);
    }
// 结束临界区操作，通过原子操作减少状态屏障的值
inline static void ggml_critical_section_end(void) {
    atomic_fetch_sub(&g_state_barrier, 1);
}

// 初始化 NUMA
void ggml_numa_init(void) {
    // 如果 NUMA 节点数大于 0，则打印错误信息并返回
    if (g_state.numa.n_nodes > 0) {
        fprintf(stderr, "ggml_numa_init: NUMA already initialized\n");
        return;
    }

    // 如果是 Linux 系统
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
    while (g_state.numa.total_cpus < GGML_NUMA_MAX_CPUS) {
        rv = snprintf(path, sizeof(path), "/sys/devices/system/cpu/cpu%u", g_state.numa.total_cpus);
        GGML_ASSERT(rv > 0 && (unsigned)rv < sizeof(path));
        if (stat(path, &st) != 0) { break; }
        ++g_state.numa.total_cpus;
    }

    // 打印找到的 NUMA 节点数和 CPU 数
    GGML_PRINT_DEBUG("found %u numa nodes, %u CPUs\n", g_state.numa.n_nodes, g_state.numa.total_cpus);

    // 如果找到的 NUMA 节点数小于 1 或 CPU 数小于 1，则将 NUMA 节点数重置为 0 并返回
    if (g_state.numa.n_nodes < 1 || g_state.numa.total_cpus < 1) {
        g_state.numa.n_nodes = 0;
        return;
    }

    // 遍历 NUMA 节点
    for (uint32_t n = 0; n < g_state.numa.n_nodes; ++n) {
        struct ggml_numa_node * node = &g_state.numa.nodes[n];
        GGML_PRINT_DEBUG("CPUs on node %u:", n);
        node->n_cpus = 0;
        // 遍历 CPU
        for (uint32_t c = 0; c < g_state.numa.total_cpus; ++c) {
            rv = snprintf(path, sizeof(path), "/sys/devices/system/node/node%u/cpu%u", n, c);
            GGML_ASSERT(rv > 0 && (unsigned)rv < sizeof(path));
            if (stat(path, &st) == 0) {
                node->cpus[node->n_cpus++] = c;
                GGML_PRINT_DEBUG(" %u", c);
            }
        }
        GGML_PRINT_DEBUG("\n");
    }
}
    # 检查系统是否支持 NUMA 架构
    if (ggml_is_numa()) {
        # 以只读方式打开 /proc/sys/kernel/numa_balancing 文件
        FILE *fptr = fopen("/proc/sys/kernel/numa_balancing", "r");
        # 如果文件指针不为空
        if (fptr != NULL) {
            # 创建一个大小为42的字符数组
            char buf[42];
            # 从文件中读取一行数据到 buf 中，并检查是否与 "0\n" 不相等
            if (fgets(buf, sizeof(buf), fptr) && strncmp(buf, "0\n", sizeof(buf)) != 0) {
                # 如果不相等，则打印警告信息
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

bool ggml_is_numa(void) {
    // 检查系统是否支持 NUMA 架构，返回节点数是否大于1
    return g_state.numa.n_nodes > 1;
}

////////////////////////////////////////////////////////////////////////////////

void ggml_print_object(const struct ggml_object * obj) {
    // 打印 ggml_object 结构体的信息，包括类型、偏移量、大小和下一个对象的地址
    GGML_PRINT(" - ggml_object: type = %d, offset = %zu, size = %zu, next = %p\n",
            obj->type, obj->offs, obj->size, (const void *) obj->next);
}

void ggml_print_objects(const struct ggml_context * ctx) {
    // 打印上下文中的对象信息
    struct ggml_object * obj = ctx->objects_begin;

    GGML_PRINT("%s: objects in context %p:\n", __func__, (const void *) ctx);

    // 遍历对象链表，逐个打印对象信息
    while (obj != NULL) {
        ggml_print_object(obj);
        obj = obj->next;
    }

    GGML_PRINT("%s: --- end ---\n", __func__);
}

int64_t ggml_nelements(const struct ggml_tensor * tensor) {
    // 计算张量中元素的总数
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    return tensor->ne[0]*tensor->ne[1]*tensor->ne[2]*tensor->ne[3];
}

int64_t ggml_nrows(const struct ggml_tensor * tensor) {
    // 计算张量的行数
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    return tensor->ne[1]*tensor->ne[2]*tensor->ne[3];
}

size_t ggml_nbytes(const struct ggml_tensor * tensor) {
    // 计算张量占用的字节数
    size_t nbytes;
    size_t blck_size = ggml_blck_size(tensor->type);
    if (blck_size == 1) {
        nbytes = ggml_type_size(tensor->type);
        for (int i = 0; i < GGML_MAX_DIMS; ++i) {
            nbytes += (tensor->ne[i] - 1)*tensor->nb[i];
        }
    }
    else {
        nbytes = tensor->ne[0]*tensor->nb[0]/blck_size;
        for (int i = 1; i < GGML_MAX_DIMS; ++i) {
            nbytes += (tensor->ne[i] - 1)*tensor->nb[i];
        }
    }

    return nbytes;
}

size_t ggml_nbytes_pad(const struct ggml_tensor * tensor) {
    // 计算张量占用的字节数，考虑内存对齐
    return GGML_PAD(ggml_nbytes(tensor), GGML_MEM_ALIGN);
}

size_t ggml_nbytes_split(const struct ggml_tensor * tensor, int nrows_split) {
    // 计算分割后的张量占用的字节数
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");
    # 返回计算得到的值，表示分割后的行数乘以张量元素数量再乘以元素类型的大小，再除以元素类型的块大小
// 返回指定类型的块大小
int ggml_blck_size(enum ggml_type type) {
    return type_traits[type].blck_size;
}

// 返回指定类型的大小
size_t ggml_type_size(enum ggml_type type) {
    return type_traits[type].type_size;
}

// 返回指定类型的大小（浮点数形式）
float ggml_type_sizef(enum ggml_type type) {
    return ((float)(type_traits[type].type_size))/type_traits[type].blck_size;
}

// 返回指定类型的名称
const char * ggml_type_name(enum ggml_type type) {
    return type_traits[type].type_name;
}

// 返回指定类型是否是量化的
bool ggml_is_quantized(enum ggml_type type) {
    return type_traits[type].is_quantized;
}

// 返回指定操作的名称
const char * ggml_op_name(enum ggml_op op) {
    return GGML_OP_NAME[op];
}

// 返回指定操作的符号
const char * ggml_op_symbol(enum ggml_op op) {
    return GGML_OP_SYMBOL[op];
}

// 返回指定张量的元素大小
size_t ggml_element_size(const struct ggml_tensor * tensor) {
    return ggml_type_size(tensor->type);
}

// 判断指定张量是否是标量
static inline bool ggml_is_scalar(const struct ggml_tensor * tensor) {
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    return tensor->ne[0] == 1 && tensor->ne[1] == 1 && tensor->ne[2] == 1 && tensor->ne[3] == 1;
}

// 判断指定张量是否是向量
static inline bool ggml_is_vector(const struct ggml_tensor * tensor) {
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    return tensor->ne[1] == 1 && tensor->ne[2] == 1 && tensor->ne[3] == 1;
}

// 判断指定张量是否是矩阵
static inline bool ggml_is_matrix(const struct ggml_tensor * tensor) {
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    return tensor->ne[2] == 1 && tensor->ne[3] == 1;
}

// 判断两个张量是否可以相乘
static inline bool ggml_can_mul_mat(const struct ggml_tensor * t0, const struct ggml_tensor * t1) {
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    return (t0->ne[0]           == t1->ne[0])  &&
           (t1->ne[2]%t0->ne[2] == 0)          && // 验证 t0 是否可广播
           (t1->ne[3]%t0->ne[3] == 0);
}

// 判断两个张量是否可以进行外积
static inline bool ggml_can_out_prod(const struct ggml_tensor * t0, const struct ggml_tensor * t1) {
    # 静态断言，确保 GGML_MAX_DIMS 的值为 4，否则输出错误信息
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    # 返回三个条件的逻辑与结果，用于验证 t0 是否可以进行广播
    return (t0->ne[1] == t1->ne[1])   &&  # 验证第一维度是否相等
           (t1->ne[2]%t0->ne[2] == 0) &&  # 验证第二维度是否可以整除
           (t1->ne[3]%t0->ne[3] == 0);    # 验证第三维度是否可以整除
# 将 ggml_ftype 转换为 ggml_type 的枚举类型
enum ggml_type ggml_ftype_to_ggml_type(enum ggml_ftype ftype) {
    # 初始化 ggml_type 变量 wtype
    enum ggml_type wtype = GGML_TYPE_COUNT;

    # 根据不同的 ggml_ftype 值进行不同的转换
    switch (ftype) {
        case GGML_FTYPE_ALL_F32:              wtype = GGML_TYPE_F32;   break;
        case GGML_FTYPE_MOSTLY_F16:           wtype = GGML_TYPE_F16;   break;
        case GGML_FTYPE_MOSTLY_Q4_0:          wtype = GGML_TYPE_Q4_0;  break;
        case GGML_FTYPE_MOSTLY_Q4_1:          wtype = GGML_TYPE_Q4_1;  break;
        case GGML_FTYPE_MOSTLY_Q5_0:          wtype = GGML_TYPE_Q5_0;  break;
        case GGML_FTYPE_MOSTLY_Q5_1:          wtype = GGML_TYPE_Q5_1;  break;
        case GGML_FTYPE_MOSTLY_Q8_0:          wtype = GGML_TYPE_Q8_0;  break;
        case GGML_FTYPE_MOSTLY_Q2_K:          wtype = GGML_TYPE_Q2_K;  break;
        case GGML_FTYPE_MOSTLY_Q3_K:          wtype = GGML_TYPE_Q3_K;  break;
        case GGML_FTYPE_MOSTLY_Q4_K:          wtype = GGML_TYPE_Q4_K;  break;
        case GGML_FTYPE_MOSTLY_Q5_K:          wtype = GGML_TYPE_Q5_K;  break;
        case GGML_FTYPE_MOSTLY_Q6_K:          wtype = GGML_TYPE_Q6_K;  break;
        case GGML_FTYPE_UNKNOWN:              wtype = GGML_TYPE_COUNT; break;
        case GGML_FTYPE_MOSTLY_Q4_1_SOME_F16: wtype = GGML_TYPE_COUNT; break;
    }

    # 断言 wtype 不等于 GGML_TYPE_COUNT
    GGML_ASSERT(wtype != GGML_TYPE_COUNT);

    # 返回转换后的 ggml_type
    return wtype;
}

# 返回 ggml_tensor 的开销大小
size_t ggml_tensor_overhead(void) {
    return GGML_OBJECT_SIZE + GGML_TENSOR_SIZE;
}

# 判断 ggml_tensor 是否转置
bool ggml_is_transposed(const struct ggml_tensor * tensor) {
    return tensor->nb[0] > tensor->nb[1];
}

# 判断 ggml_tensor 是否连续
bool ggml_is_contiguous(const struct ggml_tensor * tensor) {
    # 静态断言，确保 GGML_MAX_DIMS 等于 4
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    # 判断 ggml_tensor 是否在每个维度上都是连续的
    return
        tensor->nb[0] == ggml_type_size(tensor->type) &&
        tensor->nb[1] == (tensor->nb[0]*tensor->ne[0])/ggml_blck_size(tensor->type) &&
        tensor->nb[2] == tensor->nb[1]*tensor->ne[1] &&
        tensor->nb[3] == tensor->nb[2]*tensor->ne[2];
}

# 判断 ggml_tensor 在除了第一个维度以外的维度上是否连续
static inline bool ggml_is_contiguous_except_dim_1(const struct ggml_tensor * tensor) {
    # 确保 GGML_MAX_DIMS 的值为 4，如果不是则触发静态断言错误信息
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");
    
    # 返回一个布尔值，判断是否满足以下条件：
    # 第一个维度的元素数量等于数据类型的大小
    # 第三个维度的元素数量等于第二个维度的元素数量乘以第二个维度的扩展数量
    # 第四个维度的元素数量等于第三个维度的元素数量乘以第三个维度的扩展数量
    return
        tensor->nb[0] == ggml_type_size(tensor->type) &&
        tensor->nb[2] == tensor->nb[1]*tensor->ne[1] &&
        tensor->nb[3] == tensor->nb[2]*tensor->ne[2];
// 检查给定的张量是否被置换
bool ggml_is_permuted(const struct ggml_tensor * tensor) {
    // 静态断言，确保 GGML_MAX_DIMS 的值为 4，否则更新此函数
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    // 返回张量的维度是否被置换
    return tensor->nb[0] > tensor->nb[1] || tensor->nb[1] > tensor->nb[2] || tensor->nb[2] > tensor->nb[3];
}

// 检查给定的张量是否是一维填充的
static inline bool ggml_is_padded_1d(const struct ggml_tensor * tensor) {
    // 静态断言，确保 GGML_MAX_DIMS 的值为 4，否则更新此函数
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    // 返回张量是否是一维填充的
    return
        tensor->nb[0] == ggml_type_size(tensor->type) &&
        tensor->nb[2] == tensor->nb[1]*tensor->ne[1] &&
        tensor->nb[3] == tensor->nb[2]*tensor->ne[2];
}

// 检查两个张量是否具有相同的形状
bool ggml_are_same_shape(const struct ggml_tensor * t0, const struct ggml_tensor * t1) {
    // 静态断言，确保 GGML_MAX_DIMS 的值为 4，否则更新此函数
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    // 返回两个张量是否具有相同的形状
    return
        (t0->ne[0] == t1->ne[0] ) &&
        (t0->ne[1] == t1->ne[1] ) &&
        (t0->ne[2] == t1->ne[2] ) &&
        (t0->ne[3] == t1->ne[3] );
}

// 检查 t1 是否可以表示为 t0 的重复
static inline bool ggml_can_repeat(const struct ggml_tensor * t0, const struct ggml_tensor * t1) {
    // 静态断言，确保 GGML_MAX_DIMS 的值为 4，否则更新此函数
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    // 返回 t1 是否可以表示为 t0 的重复
    return
        (t1->ne[0]%t0->ne[0] == 0) &&
        (t1->ne[1]%t0->ne[1] == 0) &&
        (t1->ne[2]%t0->ne[2] == 0) &&
        (t1->ne[3]%t0->ne[3] == 0);
}

// 检查 t0 和 t1 是否具有相同的行形状，并且 t1 是否可以表示为 t0 的重复
static inline bool ggml_can_repeat_rows(const struct ggml_tensor * t0, const struct ggml_tensor * t1) {
    // 静态断言，确保 GGML_MAX_DIMS 的值为 4，否则更新此函数
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    // 返回 t0 和 t1 是否具有相同的行形状，并且 t1 是否可以表示为 t0 的重复
    return (t0->ne[0] == t1->ne[0]) && ggml_can_repeat(t0, t1);
}

// 返回大于等于 n 的最小的 32 的倍数
static inline int ggml_up32(int n) {
    return (n + 31) & ~31;
}

// 返回大于等于 n 的最小的 m 的倍数
static inline int ggml_up(int n, int m) {
    // 断言 m 是 2 的幂
    GGML_ASSERT((m & (m - 1)) == 0);
    return (n + m - 1) & ~(m - 1);
}

// 断言指针对齐到 GGML_MEM_ALIGN
#define ggml_assert_aligned(ptr) \
    # 使用 GGML_ASSERT 宏来检查指针 ptr 是否按照 GGML_MEM_ALIGN 对齐
    GGML_ASSERT(((uintptr_t) (ptr))%GGML_MEM_ALIGN == 0)
// 初始化 GGML 上下文，根据给定参数
struct ggml_context * ggml_init(struct ggml_init_params params) {
    // 使这个函数具有线程安全性
    ggml_critical_section_start();

    // 静态变量，标记是否是第一次调用该函数
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
                // 将 16 位无符号整数转换为 ggml_fp16_t 类型
                memcpy(&ii, &ui, sizeof(ii));
                // 计算并存储对应的浮点数值
                const float f = ggml_table_f32_f16[i] = GGML_COMPUTE_FP16_TO_FP32(ii);
                // 计算 GELU 值并存储
                ggml_table_gelu_f16[i] = GGML_FP32_TO_FP16(ggml_gelu_f32(f));
                // 计算 Quick GELU 值并存储
                ggml_table_gelu_quick_f16[i] = GGML_FP32_TO_FP16(ggml_gelu_quick_f32(f));
                // 计算 SILU 值并存储
                ggml_table_silu_f16[i] = GGML_FP32_TO_FP16(ggml_silu_f32(f));
                // 计算 EXP 值并存储
                ggml_table_exp_f16[i]  = GGML_FP32_TO_FP16(expf(f));
            }

            // 记录结束时间
            const uint64_t t_end = ggml_time_us(); UNUSED(t_end);

            // 打印初始化时间
            GGML_PRINT_DEBUG("%s: GELU, Quick GELU, SILU and EXP tables initialized in %f ms\n", __func__, (t_end - t_start)/1000.0f);
        }

        // 初始化 g_state
        {
            // 记录开始时间
            const uint64_t t_start = ggml_time_us(); UNUSED(t_start);

            // 初始化 g_state 结构
            g_state = (struct ggml_state) {
                /*.contexts =*/ { { 0 } },
                /*.numa =*/ {
                    .n_nodes = 0,
                    .total_cpus = 0,
                },
            };

            // 将 g_state 中的 contexts 数组标记为未使用
            for (int i = 0; i < GGML_MAX_CONTEXTS; ++i) {
                g_state.contexts[i].used = false;
            }

            // 记录结束时间
            const uint64_t t_end = ggml_time_us(); UNUSED(t_end);

            // 打印初始化时间
            GGML_PRINT_DEBUG("%s: g_state initialized in %f ms\n", __func__, (t_end - t_start)/1000.0f);
        }

        // 如果定义了 GGML_USE_CUBLAS，则初始化 cublas
#if defined(GGML_USE_CUBLAS)
        ggml_init_cublas();
#elif defined(GGML_USE_CLBLAST)
        ggml_cl_init();
#endif

        ggml_setup_op_has_task_pass();

        is_first_call = false;
    }

    // 在 g_state 中找到一个未使用的上下文
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

void ggml_free(struct ggml_context * ctx) {
    // 使该函数线程安全
    ggml_critical_section_start();

    bool found = false;
    # 遍历上下文数组，查找与给定上下文相匹配的上下文
    for (int i = 0; i < GGML_MAX_CONTEXTS; i++) {
        # 如果找到与给定上下文相匹配的上下文
        if (&g_state.contexts[i].context == ctx) {
            # 将该上下文标记为未使用
            g_state.contexts[i].used = false;

            # 打印调试信息，显示已释放上下文的内存使用情况
            GGML_PRINT_DEBUG("%s: context %d has been freed. memory used = %zu\n",
                    __func__, i, ggml_used_mem(ctx));

            # 如果上下文的内存缓冲区是自己拥有的
            if (ctx->mem_buffer_owned) {
                # 释放内存缓冲区
                GGML_ALIGNED_FREE(ctx->mem_buffer);
            }

            # 标记为找到相匹配的上下文
            found = true;
            # 退出循环
            break;
        }
    }

    # 如果未找到相匹配的上下文
    if (!found) {
        # 打印调试信息，显示未找到相匹配的上下文
        GGML_PRINT_DEBUG("%s: context not found\n", __func__);
    }

    # 结束临界区
    ggml_critical_section_end();
// 返回已使用的内存大小
size_t ggml_used_mem(const struct ggml_context * ctx) {
    return ctx->objects_end == NULL ? 0 : ctx->objects_end->offs + ctx->objects_end->size;
}

// 设置临时缓冲区，并返回之前的缓冲区偏移量
size_t ggml_set_scratch(struct ggml_context * ctx, struct ggml_scratch scratch) {
    const size_t result = ctx->scratch.data ? ctx->scratch.offs : 0;

    ctx->scratch = scratch;

    return result;
}

// 获取是否禁止分配内存
bool ggml_get_no_alloc(struct ggml_context * ctx) {
    return ctx->no_alloc;
}

// 设置是否禁止分配内存
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

    struct ggml_object * obj = ctx->objects_begin;

    while (obj != NULL) {
        if (obj->type == GGML_OBJECT_TENSOR) {
            struct ggml_tensor * tensor = (struct ggml_tensor *) ((char *) ctx->mem_buffer + obj->offs);

            const size_t size = ggml_nbytes(tensor);

            if (max_size < size) {
                max_size = size;
            }
        }

        obj = obj->next;
    }

    return max_size;
}

// 保存临时缓冲区
static void ggml_scratch_save(struct ggml_context * ctx) {
    // 保存禁止分配内存的状态
    ctx->no_alloc_save = ctx->no_alloc;
    ctx->no_alloc      = false;

    // 保存当前的临时缓冲区
    ctx->scratch_save = ctx->scratch;
    ctx->scratch.data = NULL;
}

// 加载临时缓冲区
static void ggml_scratch_load(struct ggml_context * ctx) {
    // 恢复禁止分配内存的状态
    ctx->no_alloc = ctx->no_alloc_save;

    // 恢复之前保存的临时缓冲区
    ctx->scratch = ctx->scratch_save;
}
// 创建一个新的 GGML 对象，并将其插入到上下文的内存池末尾
static struct ggml_object * ggml_new_object(struct ggml_context * ctx, enum ggml_object_type type, size_t size) {
    // 始终将对象插入到上下文内存池的末尾
    struct ggml_object * obj_cur = ctx->objects_end;

    const size_t cur_offs = obj_cur == NULL ? 0 : obj_cur->offs;
    const size_t cur_size = obj_cur == NULL ? 0 : obj_cur->size;
    const size_t cur_end  = cur_offs + cur_size;

    // 对齐到 GGML_MEM_ALIGN
    size_t size_needed = GGML_PAD(size, GGML_MEM_ALIGN);

    char * const mem_buffer = ctx->mem_buffer;
    struct ggml_object * const obj_new = (struct ggml_object *)(mem_buffer + cur_end);

    if (cur_end + size_needed + GGML_OBJECT_SIZE > ctx->mem_size) {
        GGML_PRINT("%s: not enough space in the context's memory pool (needed %zu, available %zu)\n",
                __func__, cur_end + size_needed, ctx->mem_size);
        assert(false);
        return NULL;
    }

    // 初始化新对象
    *obj_new = (struct ggml_object) {
        .offs = cur_end + GGML_OBJECT_SIZE,
        .size = size_needed,
        .next = NULL,
        .type = type,
    };

    // 确保内存对齐
    ggml_assert_aligned(mem_buffer + obj_new->offs);

    if (obj_cur != NULL) {
        obj_cur->next = obj_new;
    } else {
        // 这是上下文中的第一个对象
        ctx->objects_begin = obj_new;
    }

    ctx->objects_end = obj_new;

    //printf("%s: inserted new object at %zu, size = %zu\n", __func__, cur_end, obj_new->size);

    return obj_new;
}

// 创建一个新的 GGML 张量
static struct ggml_tensor * ggml_new_tensor_impl(
        struct ggml_context * ctx,
        enum   ggml_type      type,
        int                   n_dims,
        const int64_t       * ne,
        struct ggml_tensor  * view_src,
        size_t                view_offs) {

    assert(n_dims >= 1 && n_dims <= GGML_MAX_DIMS);

    // 找到基本张量和绝对偏移量
    // 如果视图源不为空且视图源的视图源不为空
    if (view_src != NULL && view_src->view_src != NULL) {
        // 视图偏移量增加视图源的视图偏移量
        view_offs += view_src->view_offs;
        // 视图源指向其视图源
        view_src   = view_src->view_src;
    }

    // 计算数据大小
    size_t data_size = ggml_type_size(type)*(ne[0]/ggml_blck_size(type));
    for (int i = 1; i < n_dims; i++) {
        data_size *= ne[i];
    }

    // 断言视图源为空或数据大小加上视图偏移量不大于视图源的字节数
    GGML_ASSERT(view_src == NULL || data_size + view_offs <= ggml_nbytes(view_src));

    // 初始化数据指针为视图源的数据，如果数据不为空，则指针偏移视图偏移量
    void * data = view_src != NULL ? view_src->data : NULL;
    if (data != NULL) {
        data = (char *) data + view_offs;
    }

    // 初始化对象分配大小为0
    size_t obj_alloc_size = 0;

    // 如果视图源为空且上下文的不分配标志为假
    if (view_src == NULL && !ctx->no_alloc) {
        // 如果上下文的临时缓冲区数据不为空
        if (ctx->scratch.data != NULL) {
            // 在临时缓冲区中分配张量数据
            if (ctx->scratch.offs + data_size > ctx->scratch.size) {
                // 如果临时内存池中空间不足，打印错误信息并断言
                GGML_PRINT("%s: not enough space in the scratch memory pool (needed %zu, available %zu)\n",
                        __func__, ctx->scratch.offs + data_size, ctx->scratch.size);
                assert(false);
                return NULL;
            }

            // 数据指针指向临时缓冲区的数据，并更新临时缓冲区的偏移量
            data = (char * const) ctx->scratch.data + ctx->scratch.offs;

            ctx->scratch.offs += data_size;
        } else {
            // 在上下文的内存池中分配张量数据
            obj_alloc_size = data_size;
        }
    }

    // 在上下文中创建新的对象，类型为GGML_OBJECT_TENSOR，大小为GGML_TENSOR_SIZE + obj_alloc_size
    struct ggml_object * const obj_new = ggml_new_object(ctx, GGML_OBJECT_TENSOR, GGML_TENSOR_SIZE + obj_alloc_size);

    // TODO: 对于可恢复的错误，我们需要在这里释放从临时缓冲区分配的数据

    // 结果指针指向上下文内存缓冲区中新对象的偏移位置处
    struct ggml_tensor * const result = (struct ggml_tensor *)((char *)ctx->mem_buffer + obj_new->offs);
    *result = (struct ggml_tensor) {
        /*.type         =*/ type,  // 设置结构体中的 type 字段为给定的 type 值
        /*.backend      =*/ GGML_BACKEND_CPU,  // 设置结构体中的 backend 字段为 GGML_BACKEND_CPU
        /*.buffer       =*/ NULL,  // 设置结构体中的 buffer 字段为 NULL
        /*.n_dims       =*/ n_dims,  // 设置结构体中的 n_dims 字段为给定的 n_dims 值
        /*.ne           =*/ { 1, 1, 1, 1 },  // 设置结构体中的 ne 数组为 {1, 1, 1, 1}
        /*.nb           =*/ { 0, 0, 0, 0 },  // 设置结构体中的 nb 数组为 {0, 0, 0, 0}
        /*.op           =*/ GGML_OP_NONE,  // 设置结构体中的 op 字段为 GGML_OP_NONE
        /*.op_params    =*/ { 0 },  // 设置结构体中的 op_params 数组为 {0}
        /*.is_param     =*/ false,  // 设置结构体中的 is_param 字段为 false
        /*.grad         =*/ NULL,  // 设置结构体中的 grad 字段为 NULL
        /*.src          =*/ { NULL },  // 设置结构体中的 src 数组为 {NULL}
        /*.is_finish    =*/ ATOMIC_VAR_INIT(0),  // 设置结构体中的 is_finish 字段为 ATOMIC_VAR_INIT(0)
        /*.perf_runs    =*/ 0,  // 设置结构体中的 perf_runs 字段为 0
        /*.perf_cycles  =*/ 0,  // 设置结构体中的 perf_cycles 字段为 0
        /*.perf_time_us =*/ 0,  // 设置结构体中的 perf_time_us 字段为 0
        /*.view_src     =*/ view_src,  // 设置结构体中的 view_src 字段为给定的 view_src 值
        /*.view_offs    =*/ view_offs,  // 设置结构体中的 view_offs 字段为给定的 view_offs 值
        /*.data         =*/ obj_alloc_size > 0 ? (void *)(result + 1) : data,  // 根据条件设置结构体中的 data 字段
        /*.name         =*/ { 0 },  // 设置结构体中的 name 数组为 {0}
        /*.extra        =*/ NULL,  // 设置结构体中的 extra 字段为 NULL
        /*.padding      =*/ { 0 },  // 设置结构体中的 padding 数组为 {0}
    };

    // TODO: this should not be needed as long as we don't rely on aligned SIMD loads
    //ggml_assert_aligned(result->data);  // 断言 result->data 是对齐的，如果不依赖于对齐的 SIMD 加载，则不需要这个断言

    for (int i = 0; i < n_dims; i++) {
        result->ne[i] = ne[i];  // 遍历设置结构体中的 ne 数组
    }

    result->nb[0] = ggml_type_size(type);  // 设置结构体中的 nb 数组的第一个元素
    result->nb[1] = result->nb[0]*(result->ne[0]/ggml_blck_size(type));  // 设置结构体中的 nb 数组的第二个元素
    for (int i = 2; i < GGML_MAX_DIMS; i++) {
        result->nb[i] = result->nb[i - 1]*result->ne[i - 1];  // 遍历设置结构体中的 nb 数组
    }

    ctx->n_objects++;  // 上下文中对象数量加一

    return result;  // 返回 result 结构体
# 创建一个新的张量，根据给定的上下文、类型、维度数和维度数组
struct ggml_tensor * ggml_new_tensor(
        struct ggml_context * ctx,
        enum   ggml_type      type,
        int                   n_dims,
        const int64_t       * ne) {
    # 调用内部实现函数创建新的张量
    return ggml_new_tensor_impl(ctx, type, n_dims, ne, NULL, 0);
}

# 创建一个一维张量
struct ggml_tensor * ggml_new_tensor_1d(
        struct ggml_context * ctx,
        enum   ggml_type      type,
        int64_t ne0) {
    # 调用 ggml_new_tensor 函数创建一维张量
    return ggml_new_tensor(ctx, type, 1, &ne0);
}

# 创建一个二维张量
struct ggml_tensor * ggml_new_tensor_2d(
        struct ggml_context * ctx,
        enum   ggml_type      type,
        int64_t ne0,
        int64_t ne1) {
    # 创建包含两个维度的数组
    const int64_t ne[2] = { ne0, ne1 };
    # 调用 ggml_new_tensor 函数创建二维张量
    return ggml_new_tensor(ctx, type, 2, ne);
}

# 创建一个三维张量
struct ggml_tensor * ggml_new_tensor_3d(
        struct ggml_context * ctx,
        enum   ggml_type      type,
        int64_t ne0,
        int64_t ne1,
        int64_t ne2) {
    # 创建包含三个维度的数组
    const int64_t ne[3] = { ne0, ne1, ne2 };
    # 调用 ggml_new_tensor 函数创建三维张量
    return ggml_new_tensor(ctx, type, 3, ne);
}

# 创建一个四维张量
struct ggml_tensor * ggml_new_tensor_4d(
        struct ggml_context * ctx,
        enum   ggml_type type,
        int64_t ne0,
        int64_t ne1,
        int64_t ne2,
        int64_t ne3) {
    # 创建包含四个维度的数组
    const int64_t ne[4] = { ne0, ne1, ne2, ne3 };
    # 调用 ggml_new_tensor 函数创建四维张量
    return ggml_new_tensor(ctx, type, 4, ne);
}

# 创建一个包含 int32_t 类型值的张量
struct ggml_tensor * ggml_new_i32(struct ggml_context * ctx, int32_t value) {
    # 保存上下文状态
    ggml_scratch_save(ctx);
    # 创建一个一维张量
    struct ggml_tensor * result = ggml_new_tensor_1d(ctx, GGML_TYPE_I32, 1);
    # 恢复之前保存的上下文状态
    ggml_scratch_load(ctx);
    # 设置张量的值为给定的 int32_t 类型值
    ggml_set_i32(result, value);
    # 返回创建的张量
    return result;
}

# 创建一个包含 float 类型值的张量
struct ggml_tensor * ggml_new_f32(struct ggml_context * ctx, float value) {
    # 保存上下文状态
    ggml_scratch_save(ctx);
    # 创建一个一维张量
    struct ggml_tensor * result = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 1);
    # 恢复之前保存的上下文状态
    ggml_scratch_load(ctx);
    # 设置张量的值为给定的 float 类型值
    ggml_set_f32(result, value);
    # 返回创建的张量
    return result;
}

# 复制给定张量的内容，创建一个新的张量
struct ggml_tensor * ggml_dup_tensor(struct ggml_context * ctx, const struct ggml_tensor * src) {
    # 调用 ggml_new_tensor 函数创建一个与给定张量内容相同的新张量
    return ggml_new_tensor(ctx, src->type, src->n_dims, src->ne);
}
# 设置张量的操作参数，包括参数指针和参数大小
static void ggml_set_op_params(struct ggml_tensor * tensor, const void * params, size_t params_size) {
    GGML_ASSERT(tensor != NULL); // 禁止 -Warray-bounds 警告
    assert(params_size <= GGML_MAX_OP_PARAMS);  # 确保参数大小不超过最大操作参数大小
    memcpy(tensor->op_params, params, params_size);  # 将参数内容复制到张量的操作参数中
}

# 获取张量操作参数中的第i个32位整数
static int32_t ggml_get_op_params_i32(const struct ggml_tensor * tensor, uint32_t i) {
    assert(i < GGML_MAX_OP_PARAMS / sizeof(int32_t));  # 确保索引i不超过操作参数的大小
    return ((const int32_t *)(tensor->op_params))[i];  # 返回操作参数中第i个32位整数
}

# 设置张量操作参数中的第i个32位整数为给定的值
static void ggml_set_op_params_i32(struct ggml_tensor * tensor, uint32_t i, int32_t value) {
    assert(i < GGML_MAX_OP_PARAMS / sizeof(int32_t));  # 确保索引i不超过操作参数的大小
    ((int32_t *)(tensor->op_params))[i] = value;  # 将操作参数中第i个32位整数设置为给定的值
}

# 将张量的数据内容全部设置为0
struct ggml_tensor * ggml_set_zero(struct ggml_tensor * tensor) {
    memset(tensor->data, 0, ggml_nbytes(tensor));  # 将张量的数据内容全部设置为0
    return tensor;  # 返回修改后的张量
}

# 将张量的数据内容全部设置为给定的32位整数值
struct ggml_tensor * ggml_set_i32 (struct ggml_tensor * tensor, int32_t value) {
    const int n     = ggml_nrows(tensor);  # 获取张量的行数
    const int nc    = tensor->ne[0];  # 获取张量的第一个维度大小
    const size_t n1 = tensor->nb[1];  # 获取张量的第二维度大小

    char * const data = tensor->data;  # 获取张量的数据指针
    # 根据张量的类型进行不同的操作
    switch (tensor->type) {
        # 如果张量类型为 int8
        case GGML_TYPE_I8:
            {
                # 确保张量的第一个维度大小为 int8_t 的大小
                assert(tensor->nb[0] == sizeof(int8_t));
                # 遍历张量数据，对每个元素设置为指定值
                for (int i = 0; i < n; i++) {
                    ggml_vec_set_i8(nc, (int8_t *)(data + i*n1), value);
                }
            } break;
        # 如果张量类型为 int16
        case GGML_TYPE_I16:
            {
                # 确保张量的第一个维度大小为 int16_t 的大小
                assert(tensor->nb[0] == sizeof(int16_t));
                # 遍历张量数据，对每个元素设置为指定值
                for (int i = 0; i < n; i++) {
                    ggml_vec_set_i16(nc, (int16_t *)(data + i*n1), value);
                }
            } break;
        # 如果张量类型为 int32
        case GGML_TYPE_I32:
            {
                # 确保张量的第一个维度大小为 int32_t 的大小
                assert(tensor->nb[0] == sizeof(int32_t));
                # 遍历张量数据，对每个元素设置为指定值
                for (int i = 0; i < n; i++) {
                    ggml_vec_set_i32(nc, (int32_t *)(data + i*n1), value);
                }
            } break;
        # 如果张量类型为 f16
        case GGML_TYPE_F16:
            {
                # 确保张量的第一个维度大小为 ggml_fp16_t 的大小
                assert(tensor->nb[0] == sizeof(ggml_fp16_t));
                # 遍历张量数据，对每个元素设置为指定值的 f16 类型
                for (int i = 0; i < n; i++) {
                    ggml_vec_set_f16(nc, (ggml_fp16_t *)(data + i*n1), GGML_FP32_TO_FP16(value));
                }
            } break;
        # 如果张量类型为 f32
        case GGML_TYPE_F32:
            {
                # 确保张量的第一个维度大小为 float 的大小
                assert(tensor->nb[0] == sizeof(float));
                # 遍历张量数据，对每个元素设置为指定值
                for (int i = 0; i < n; i++) {
                    ggml_vec_set_f32(nc, (float *)(data + i*n1), value);
                }
            } break;
        # 默认情况下
        default:
            {
                # 断言失败
                GGML_ASSERT(false);
            } break;
    }
    # 返回处理后的张量
    return tensor;
# 设置浮点数值到张量中
struct ggml_tensor * ggml_set_f32(struct ggml_tensor * tensor, float value) {
    # 获取张量的行数
    const int n     = ggml_nrows(tensor);
    # 获取张量的第一个维度大小
    const int nc    = tensor->ne[0];
    # 获取张量的第二个维度大小
    const size_t n1 = tensor->nb[1];

    # 获取张量数据的指针
    char * const data = tensor->data;

    # 根据张量的类型进行不同的处理
    switch (tensor->type) {
        # 如果张量类型为8位整数
        case GGML_TYPE_I8:
            {
                # 确保张量的第一个维度大小为8位整数的大小
                assert(tensor->nb[0] == sizeof(int8_t));
                # 遍历张量的行数，设置每行数据为指定的浮点数值
                for (int i = 0; i < n; i++) {
                    ggml_vec_set_i8(nc, (int8_t *)(data + i*n1), value);
                }
            } break;
        # 如果张量类型为16位整数
        case GGML_TYPE_I16:
            {
                # 确保张量的第一个维度大小为16位整数的大小
                assert(tensor->nb[0] == sizeof(int16_t));
                # 遍历张量的行数，设置每行数据为指定的浮点数值
                for (int i = 0; i < n; i++) {
                    ggml_vec_set_i16(nc, (int16_t *)(data + i*n1), value);
                }
            } break;
        # 如果张量类型为32位整数
        case GGML_TYPE_I32:
            {
                # 确保张量的第一个维度大小为32位整数的大小
                assert(tensor->nb[0] == sizeof(int32_t));
                # 遍历张量的行数，设置每行数据为指定的浮点数值
                for (int i = 0; i < n; i++) {
                    ggml_vec_set_i32(nc, (int32_t *)(data + i*n1), value);
                }
            } break;
        # 如果张量类型为16位浮点数
        case GGML_TYPE_F16:
            {
                # 确保张量的第一个维度大小为16位浮点数的大小
                assert(tensor->nb[0] == sizeof(ggml_fp16_t));
                # 遍历张量的行数，设置每行数据为指定的浮点数值的16位浮点数表示
                for (int i = 0; i < n; i++) {
                    ggml_vec_set_f16(nc, (ggml_fp16_t *)(data + i*n1), GGML_FP32_TO_FP16(value));
                }
            } break;
        # 如果张量类型为32位浮点数
        case GGML_TYPE_F32:
            {
                # 确保张量的第一个维度大小为32位浮点数的大小
                assert(tensor->nb[0] == sizeof(float));
                # 遍历张量的行数，设置每行数据为指定的浮点数值
                for (int i = 0; i < n; i++) {
                    ggml_vec_set_f32(nc, (float *)(data + i*n1), value);
                }
            } break;
        # 如果张量类型为其他类型，抛出断言错误
        default:
            {
                GGML_ASSERT(false);
            } break;
    }

    # 返回设置后的张量
    return tensor;
}

# 根据索引值解析张量的多维索引
void ggml_unravel_index(const struct ggml_tensor * tensor, int64_t i, int64_t * i0, int64_t * i1, int64_t * i2, int64_t * i3) {
    # 获取张量第三个维度的大小
    const int64_t ne2 = tensor->ne[2];
    # 获取张量第二个维度的大小
    const int64_t ne1 = tensor->ne[1];
    # 获取张量第一个维度的大小
    const int64_t ne0 = tensor->ne[0];

    # 根据索引值计算第四个维度的索引
    const int64_t i3_ = (i/(ne2*ne1*ne0));
    # 计算在四维数组中的索引位置
    const int64_t i2_ = (i - i3_*ne2*ne1*ne0)/(ne1*ne0);
    const int64_t i1_ = (i - i3_*ne2*ne1*ne0 - i2_*ne1*ne0)/ne0;
    const int64_t i0_ = (i - i3_*ne2*ne1*ne0 - i2_*ne1*ne0 - i1_*ne0);

    # 如果传入的指针不为空，则将计算得到的索引值赋给相应的指针所指向的变量
    if (i0) {
        * i0 = i0_;
    }
    if (i1) {
        * i1 = i1_;
    }
    if (i2) {
        * i2 = i2_;
    }
    if (i3) {
        * i3 = i3_;
    }
}
// 获取一维整型数组中指定索引位置的元素值
int32_t ggml_get_i32_1d(const struct ggml_tensor * tensor, int i) {
    // 如果数组不是连续存储的，则需要通过索引计算出对应的多维索引，再获取对应元素值
    if (!ggml_is_contiguous(tensor)) {
        int64_t id[4] = { 0, 0, 0, 0 };
        ggml_unravel_index(tensor, i, &id[0], &id[1], &id[2], &id[3]);
        return ggml_get_i32_nd(tensor, id[0], id[1], id[2], id[3]);
    }
    // 如果数组是连续存储的，则根据数组类型直接获取对应元素值
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
    // 如果数组不是连续存储的，则需要通过索引计算出对应的多维索引，再设置对应元素值
    if (!ggml_is_contiguous(tensor)) {
        int64_t id[4] = { 0, 0, 0, 0 };
        ggml_unravel_index(tensor, i, &id[0], &id[1], &id[2], &id[3]);
        ggml_set_i32_nd(tensor, id[0], id[1], id[2], id[3], value);
        return;
    }
    # 根据张量的类型进行不同的操作
    switch (tensor->type) {
        # 如果张量类型为 int8
        case GGML_TYPE_I8:
            {
                # 断言张量的第一个维度大小为 int8_t 的大小
                GGML_ASSERT(tensor->nb[0] == sizeof(int8_t));
                # 将值赋给 int8_t 类型的数据
                ((int8_t *)(tensor->data))[i] = value;
            } break;
        # 如果张量类型为 int16
        case GGML_TYPE_I16:
            {
                # 断言张量的第一个维度大小为 int16_t 的大小
                GGML_ASSERT(tensor->nb[0] == sizeof(int16_t));
                # 将值赋给 int16_t 类型的数据
                ((int16_t *)(tensor->data))[i] = value;
            } break;
        # 如果张量类型为 int32
        case GGML_TYPE_I32:
            {
                # 断言张量的第一个维度大小为 int32_t 的大小
                GGML_ASSERT(tensor->nb[0] == sizeof(int32_t));
                # 将值赋给 int32_t 类型的数据
                ((int32_t *)(tensor->data))[i] = value;
            } break;
        # 如果张量类型为 float16
        case GGML_TYPE_F16:
            {
                # 断言张量的第一个维度大小为 ggml_fp16_t 的大小
                GGML_ASSERT(tensor->nb[0] == sizeof(ggml_fp16_t));
                # 将值转换为 float16 类型的数据并赋值
                ((ggml_fp16_t *)(tensor->data))[i] = GGML_FP32_TO_FP16(value);
            } break;
        # 如果张量类型为 float32
        case GGML_TYPE_F32:
            {
                # 断言张量的第一个维度大小为 float 的大小
                GGML_ASSERT(tensor->nb[0] == sizeof(float));
                # 将值赋给 float 类型的数据
                ((float *)(tensor->data))[i] = value;
            } break;
        # 默认情况
        default:
            {
                # 断言为假，抛出异常
                GGML_ASSERT(false);
            } break;
    }
}
// 结束函数定义

int32_t ggml_get_i32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3) {
    // 计算数据在张量中的偏移量
    void * data   = (char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1] + i2*tensor->nb[2] + i3*tensor->nb[3];
    // 根据张量类型进行不同类型的数据读取
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

void ggml_set_i32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3, int32_t value) {
    // 计算数据在张量中的偏移量
    void * data   = (char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1] + i2*tensor->nb[2] + i3*tensor->nb[3];
    // 根据张量类型进行不同类型的数据设置
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

float ggml_get_f32_1d(const struct ggml_tensor * tensor, int i) {
    // 如果张量不是连续的，则根据索引计算多维索引再进行数据读取
    if (!ggml_is_contiguous(tensor)) {
        int64_t id[4] = { 0, 0, 0, 0 };
        ggml_unravel_index(tensor, i, &id[0], &id[1], &id[2], &id[3]);
        return ggml_get_f32_nd(tensor, id[0], id[1], id[2], id[3]);
    }
    # 根据张量的类型选择不同的数据类型进行处理
    switch (tensor->type) {
        # 如果张量类型为 int8，进行以下操作
        case GGML_TYPE_I8:
            {
                # 断言张量的第一个维度大小为 int8_t 类型的大小
                GGML_ASSERT(tensor->nb[0] == sizeof(int8_t));
                # 返回 int8_t 类型数据的第 i 个元素
                return ((int8_t *)(tensor->data))[i];
            }
        # 如果张量类型为 int16，进行以下操作
        case GGML_TYPE_I16:
            {
                # 断言张量的第一个维度大小为 int16_t 类型的大小
                GGML_ASSERT(tensor->nb[0] == sizeof(int16_t));
                # 返回 int16_t 类型数据的第 i 个元素
                return ((int16_t *)(tensor->data))[i];
            }
        # 如果张量类型为 int32，进行以下操作
        case GGML_TYPE_I32:
            {
                # 断言张量的第一个维度大小为 int32_t 类型的大小
                GGML_ASSERT(tensor->nb[0] == sizeof(int32_t));
                # 返回 int32_t 类型数据的第 i 个元素
                return ((int32_t *)(tensor->data))[i];
            }
        # 如果张量类型为 F16，进行以下操作
        case GGML_TYPE_F16:
            {
                # 断言张量的第一个维度大小为 ggml_fp16_t 类型的大小
                GGML_ASSERT(tensor->nb[0] == sizeof(ggml_fp16_t));
                # 返回将 ggml_fp16_t 类型数据转换为 float 类型的第 i 个元素
                return GGML_FP16_TO_FP32(((ggml_fp16_t *)(tensor->data))[i]);
            }
        # 如果张量类型为 F32，进行以下操作
        case GGML_TYPE_F32:
            {
                # 断言张量的第一个维度大小为 float 类型的大小
                GGML_ASSERT(tensor->nb[0] == sizeof(float));
                # 返回 float 类型数据的第 i 个元素
                return ((float *)(tensor->data))[i];
            }
        # 如果张量类型为其他类型，进行以下操作
        default:
            {
                # 断言条件为假，即张量类型不在预期范围内
                GGML_ASSERT(false);
            }
    }

    # 如果没有匹配到任何类型，返回默认值 0.0f
    return 0.0f;
}
// 设置一维浮点数张量的值
void ggml_set_f32_1d(const struct ggml_tensor * tensor, int i, float value) {
    // 如果张量不是连续的，则根据索引值获取多维张量的索引
    if (!ggml_is_contiguous(tensor)) {
        int64_t id[4] = { 0, 0, 0, 0 };
        ggml_unravel_index(tensor, i, &id[0], &id[1], &id[2], &id[3]);
        ggml_set_f32_nd(tensor, id[0], id[1], id[2], id[3], value);
        return;
    }
    // 根据张量的类型设置对应位置的值
    switch (tensor->type) {
        case GGML_TYPE_I8:
            {
                GGML_ASSERT(tensor->nb[0] == sizeof(int8_t));
                ((int8_t *)(tensor->data))[i] = value;
            } break;
        case GGML_TYPE_I16:
            {
                GGML_ASSERT(tensor->nb[0] == sizeof(int16_t));
                ((int16_t *)(tensor->data))[i] = value;
            } break;
        case GGML_TYPE_I32:
            {
                GGML_ASSERT(tensor->nb[0] == sizeof(int32_t));
                ((int32_t *)(tensor->data))[i] = value;
            } break;
        case GGML_TYPE_F16:
            {
                GGML_ASSERT(tensor->nb[0] == sizeof(ggml_fp16_t));
                ((ggml_fp16_t *)(tensor->data))[i] = GGML_FP32_TO_FP16(value);
            } break;
        case GGML_TYPE_F32:
            {
                GGML_ASSERT(tensor->nb[0] == sizeof(float));
                ((float *)(tensor->data))[i] = value;
            } break;
        default:
            {
                GGML_ASSERT(false);
            } break;
    }
}

// 获取多维浮点数张量的值
float ggml_get_f32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3) {
    // 计算多维张量中指定位置的数据地址
    void * data   = (char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1] + i2*tensor->nb[2] + i3*tensor->nb[3];
    # 根据张量的类型返回对应的数据值
    switch (tensor->type) {
        # 如果张量类型为8位整数，返回对应的数据值
        case GGML_TYPE_I8:
            return ((int8_t *) data)[0];
        # 如果张量类型为16位整数，返回对应的数据值
        case GGML_TYPE_I16:
            return ((int16_t *) data)[0];
        # 如果张量类型为32位整数，返回对应的数据值
        case GGML_TYPE_I32:
            return ((int32_t *) data)[0];
        # 如果张量类型为16位浮点数，将16位浮点数转换为32位浮点数后返回对应的数据值
        case GGML_TYPE_F16:
            return GGML_FP16_TO_FP32(((ggml_fp16_t *) data)[0]);
        # 如果张量类型为32位浮点数，返回对应的数据值
        case GGML_TYPE_F32:
            return ((float *) data)[0];
        # 如果张量类型为其他类型，抛出断言错误
        default:
            GGML_ASSERT(false);
    }

    # 默认情况下返回浮点数0.0
    return 0.0f;
// 设置指定位置的四维张量的值为指定的浮点数
void ggml_set_f32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3, float value) {
    // 计算数据指针的偏移量
    void * data   = (char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1] + i2*tensor->nb[2] + i3*tensor->nb[3];
    // 根据张量的类型进行不同的赋值操作
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

// 获取张量的数据指针
void * ggml_get_data(const struct ggml_tensor * tensor) {
    return tensor->data;
}

// 获取张量的数据指针，并确保类型为 float
float * ggml_get_data_f32(const struct ggml_tensor * tensor) {
    assert(tensor->type == GGML_TYPE_F32);
    return (float *)(tensor->data);
}

// 获取张量的数据指针，并确保类型为 int32_t
int32_t * ggml_get_data_i32(const struct ggml_tensor * tensor) {
    assert(tensor->type == GGML_TYPE_I32);
    return (int32_t *)(tensor->data);
}

// 获取一元操作的操作类型
enum ggml_unary_op ggml_get_unary_op(const struct ggml_tensor * tensor) {
    GGML_ASSERT(tensor->op == GGML_OP_UNARY);
    return (enum ggml_unary_op) ggml_get_op_params_i32(tensor, 0);
}

// 获取张量的名称
const char * ggml_get_name(const struct ggml_tensor * tensor) {
    if (tensor == NULL) return NULL;
    return tensor->name;
}

// 设置张量的名称
struct ggml_tensor * ggml_set_name(struct ggml_tensor * tensor, const char * name) {
    if (tensor == NULL) return NULL;
    // 将名称拷贝到张量的名称字段中
    strncpy(tensor->name, name, sizeof(tensor->name));
    tensor->name[sizeof(tensor->name) - 1] = '\0';
    return tensor;
}

// 格式化张量的名称
struct ggml_tensor * ggml_format_name(struct ggml_tensor * tensor, const char * fmt, ...) {
    va_list args;
    # 以可变参数列表开始处理参数
    va_start(args, fmt);
    # 将格式化的字符串写入到tensor->name中，避免缓冲区溢出
    vsnprintf(tensor->name, sizeof(tensor->name), fmt, args);
    # 结束可变参数列表的处理
    va_end(args);
    # 返回tensor对象
    return tensor;
// 返回一个新的张量，作为给定张量的视图
struct ggml_tensor * ggml_view_tensor(
        struct ggml_context * ctx,
        struct ggml_tensor  * src) {
    // 创建一个新的张量，作为给定张量的视图
    struct ggml_tensor * result = ggml_new_tensor_impl(ctx, src->type, src->n_dims, src->ne, src, 0);
    // 为新张量设置格式化名称
    ggml_format_name(result, "%s (view)", src->name);

    // 复制给定张量的维度信息到新张量
    for (int i = 0; i < GGML_MAX_DIMS; i++) {
        result->nb[i] = src->nb[i];
    }
    // 设置新张量的操作类型为视图
    result->op = GGML_OP_VIEW;

    // 返回新张量
    return result;
}

// 获取上下文中的第一个张量
struct ggml_tensor * ggml_get_first_tensor(struct ggml_context * ctx) {
    // 获取上下文中的第一个对象
    struct ggml_object * obj = ctx->objects_begin;

    // 获取上下文中的内存缓冲区
    char * const mem_buffer = ctx->mem_buffer;

    // 遍历对象链表，找到第一个张量并返回
    while (obj != NULL) {
        if (obj->type == GGML_OBJECT_TENSOR) {
            return (struct ggml_tensor *)(mem_buffer + obj->offs);
        }

        obj = obj->next;
    }

    // 如果没有找到张量，则返回空指针
    return NULL;
}

// 获取上下文中给定张量的下一个张量
struct ggml_tensor * ggml_get_next_tensor(struct ggml_context * ctx, struct ggml_tensor * tensor) {
    // 获取给定张量的对象指针
    struct ggml_object * obj = (struct ggml_object *) ((char *)tensor - GGML_OBJECT_SIZE);
    // 获取下一个对象
    obj = obj->next;

    // 获取上下文中的内存缓冲区
    char * const mem_buffer = ctx->mem_buffer;

    // 遍历对象链表，找到给定张量的下一个张量并返回
    while (obj != NULL) {
        if (obj->type == GGML_OBJECT_TENSOR) {
            return (struct ggml_tensor *)(mem_buffer + obj->offs);
        }

        obj = obj->next;
    }

    // 如果没有找到下一个张量，则返回空指针
    return NULL;
}

// 根据名称获取上下文中的张量
struct ggml_tensor * ggml_get_tensor(struct ggml_context * ctx, const char * name) {
    // 获取上下文中的第一个对象
    struct ggml_object * obj = ctx->objects_begin;

    // 获取上下文中的内存缓冲区
    char * const mem_buffer = ctx->mem_buffer;

    // 遍历对象链表，根据名称找到对应的张量并返回
    while (obj != NULL) {
        if (obj->type == GGML_OBJECT_TENSOR) {
            struct ggml_tensor * cur = (struct ggml_tensor *)(mem_buffer + obj->offs);
            if (strcmp(cur->name, name) == 0) {
                return cur;
            }
        }

        obj = obj->next;
    }

    // 如果没有找到对应名称的张量，则返回空指针
    return NULL;
}

////////////////////////////////////////////////////////////////////////////////

// ggml_dup

// 复制给定张量的实现函数
static struct ggml_tensor * ggml_dup_impl(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        bool inplace) {
    # 定义一个布尔变量 is_node，并初始化为 false
    bool is_node = false;

    # 如果不是原地操作且 a 的梯度存在，则将 is_node 设置为 true
    if (!inplace && (a->grad)) {
        is_node = true;
    }

    # 根据 inplace 的取值选择创建一个新的视图张量或者复制一个新的张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    # 设置结果张量的操作类型为复制
    result->op   = GGML_OP_DUP;
    # 如果 is_node 为 true，则为结果张量创建一个新的梯度张量，否则设置为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    # 设置结果张量的源张量为 a
    result->src[0] = a;

    # 返回结果张量
    return result;
}

// 复制张量a的内容并返回一个新的张量
struct ggml_tensor * ggml_dup(
        struct ggml_context * ctx,
        struct ggml_tensor * a) {
    return ggml_dup_impl(ctx, a, false);
}

// 在原地复制张量a的内容并返回一个新的张量
struct ggml_tensor * ggml_dup_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor * a) {
    return ggml_dup_impl(ctx, a, true);
}

// ggml_add

// 实现张量a和张量b的加法操作
static struct ggml_tensor * ggml_add_impl(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b,
        bool inplace) {
    // TODO: 支持更宽松的约束
    //       GGML_ASSERT(ggml_can_repeat(b, a));
    if (a == NULL)
        return b;
    if (b == NULL)
        return a;
    GGML_ASSERT(ggml_can_repeat_rows(b, a));

    bool is_node = false;

    if (!inplace && (a->grad || b->grad)) {
        // TODO: 支持广播的反向传播
        GGML_ASSERT(ggml_are_same_shape(a, b));
        is_node = true;
    }

    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    result->op   = GGML_OP_ADD;
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;
    result->src[1] = b;
    result->src[2] = NULL;

    return result;
}

// 对张量a和张量b执行加法操作
struct ggml_tensor * ggml_add(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b) {
    return ggml_add_impl(ctx, a, b, false);
}

// 在原地对张量a和张量b执行加法操作
struct ggml_tensor * ggml_add_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b) {
    return ggml_add_impl(ctx, a, b, true);
}

// ggml_add_cast

// 实现对张量a和张量b执行加法操作并进行类型转换
static struct ggml_tensor * ggml_add_cast_impl(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b,
        enum   ggml_type     type) {
    // TODO: 支持更宽松的约束
    //       GGML_ASSERT(ggml_can_repeat(b, a));
    GGML_ASSERT(ggml_can_repeat_rows(b, a));
    # 断言输入类型为量化或者 f16，否则抛出异常
    GGML_ASSERT(ggml_is_quantized(a->type) || a->type == GGML_TYPE_F16); // currently only supported for quantized input and f16

    # 初始化一个布尔变量 is_node，用于标记是否需要进行反向传播
    bool is_node = false;

    # 如果 a 或者 b 其中一个有梯度信息
    if (a->grad || b->grad) {
        # TODO: 支持广播的反向传播
        GGML_ASSERT(ggml_are_same_shape(a, b));  # 断言 a 和 b 的形状相同，否则抛出异常
        is_node = true;  # 设置 is_node 为 true，表示需要进行反向传播
    }

    # 创建一个新的张量 result，用于存储计算结果
    struct ggml_tensor * result = ggml_new_tensor(ctx, type, a->n_dims, a->ne);

    # 设置 result 的操作类型为加法
    result->op   = GGML_OP_ADD;
    # 如果需要进行反向传播，则为 result 分配一个梯度张量，否则为 NULL
    result->grad = is_node ? ggml_new_tensor(ctx, GGML_TYPE_F32, a->n_dims, a->ne) : NULL;
    # 设置 result 的源张量为 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    # 返回计算结果张量 result
    return result;
// 定义一个静态函数，用于实现两个张量相加并添加索引的操作
static struct ggml_tensor * ggml_add_idx_impl(
        struct ggml_context * ctx,  // 上下文对象指针
        struct ggml_tensor * a,      // 第一个张量指针
        struct ggml_tensor * b,      // 第二个张量指针
        struct ggml_tensor * idx,    // 索引张量指针
        bool inplace) {              // 是否原地操作的布尔值
    // TODO: 支持更宽松的约束
    //       GGML_ASSERT(ggml_can_repeat(b, a));
    // GGML_ASSERT(ggml_can_repeat_rows(b, a));
    // 打印调试信息
    // printf("in add_idx\n");
    if (a == NULL)  // 如果第一个张量为空，则返回第二个张量
        return b;
    if (b == NULL)  // 如果第二个张量为空，则返回第一个张量
        return a;

    bool is_node = false;  // 初始化节点标志为假

    if (!inplace && (a->grad || b->grad)) {  // 如果不是原地操作且至少一个张量需要梯度
        // TODO: 支持广播的反向传播
        GGML_ASSERT(ggml_are_same_shape(a, b));  // 断言两个张量的形状相同
        is_node = true;  // 设置节点标志为真
    }

    // 创建结果张量，如果是原地操作则创建视图张量，否则创建副本张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    result->op   = GGML_OP_ADD;  // 设置结果张量的操作为加法
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;  // 如果是节点，则设置梯度张量为结果张量的副本
    result->src[0] = a;  // 设置结果张量的源张量1为第一个张量
    result->src[1] = b;  // 设置结果张量的源张量2为第二个张量
    result->src[2] = idx;  // 设置结果张量的源张量3为索引张量

    return result;  // 返回结果张量
}
// 为全局聚合添加操作
struct ggml_tensor * ggml_add_idx(
        struct ggml_context * ctx,  // 上下文对象指针
        struct ggml_tensor * a,      // 第一个张量指针
        struct ggml_tensor * b,      // 第二个张量指针
        struct ggml_tensor * idx) {  // 索引张量指针
    return ggml_add_idx_impl(ctx, a, b, idx, false);  // 调用实现函数，返回结果张量
}

// 为类型转换添加操作
struct ggml_tensor * ggml_add_cast(
        struct ggml_context * ctx,  // 上下文对象指针
        struct ggml_tensor * a,      // 第一个张量指针
        struct ggml_tensor * b,      // 第二个张量指针
        enum   ggml_type     type) {  // 类型枚举
    return ggml_add_cast_impl(ctx, a, b, type);  // 调用实现函数，返回结果张量
}

// ggml_add1

// 定义一个静态函数，用于实现一个张量与标量相加的操作
static struct ggml_tensor * ggml_add1_impl(
        struct ggml_context * ctx,  // 上下文对象指针
        struct ggml_tensor * a,      // 第一个张量指针
        struct ggml_tensor * b,      // 第二个张量指针
        bool inplace) {              // 是否原地操作的布尔值
    GGML_ASSERT(ggml_is_scalar(b));  // 断言第二个张量是标量
    GGML_ASSERT(ggml_is_padded_1d(a));  // 断言第一个张量是填充的一维张量

    bool is_node = false;  // 初始化节点标志为假

    if (a->grad || b->grad) {  // 如果第一个张量或第二个张量需要梯度
        is_node = true;  // 设置节点标志为真
    }

    // 创建结果张量，如果是原地操作则创建视图张量，否则创建副本张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    result->op   = GGML_OP_ADD1;  // 设置结果张量的操作为加法
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;  // 如果是节点，则设置梯度张量为结果张量的副本
    result->src[0] = a;  // 设置结果张量的源张量1为第一个张量
    # 将 result 对象的 src 列表中的第二个元素赋值为 b
    result->src[1] = b;
    # 返回 result 对象
    return result;
}

// 定义一个函数，实现两个张量的加法操作，返回结果张量
struct ggml_tensor * ggml_add1(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b) {
    return ggml_add1_impl(ctx, a, b, false); // 调用内部函数实现加法操作
}

// 定义一个函数，实现两个张量的加法操作，并将结果存储在第一个张量中，返回结果张量
struct ggml_tensor * ggml_add1_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b) {
    return ggml_add1_impl(ctx, a, b, true); // 调用内部函数实现原地加法操作
}

// ggml_acc

// 内部函数，实现张量的累加操作
static struct ggml_tensor * ggml_acc_impl(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b,
        size_t               nb1,
        size_t               nb2,
        size_t               nb3,
        size_t               offset,
        bool inplace) {
    GGML_ASSERT(ggml_nelements(b) <= ggml_nelements(a)); // 断言，确保 b 的元素个数不超过 a 的元素个数
    GGML_ASSERT(ggml_is_contiguous(a)); // 断言，确保 a 是连续存储的
    GGML_ASSERT(a->type == GGML_TYPE_F32); // 断言，确保 a 的数据类型为 F32
    GGML_ASSERT(b->type == GGML_TYPE_F32); // 断言，确保 b 的数据类型为 F32

    bool is_node = false;

    if (!inplace && (a->grad || b->grad)) {
        is_node = true;
    }

    // 根据是否原地操作选择复制或者视图操作
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    int32_t params[] = { nb1, nb2, nb3, offset, inplace ? 1 : 0 };
    ggml_set_op_params(result, params, sizeof(params)); // 设置操作参数

    result->op   = GGML_OP_ACC; // 设置操作类型为累加
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL; // 如果是节点，则复制张量，否则为 NULL
    result->src[0] = a; // 设置源张量
    result->src[1] = b; // 设置源张量

    return result; // 返回结果张量
}

// 定义一个函数，实现两个张量的累加操作，返回结果张量
struct ggml_tensor * ggml_acc(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b,
        size_t               nb1,
        size_t               nb2,
        size_t               nb3,
        size_t               offset) {
    return ggml_acc_impl(ctx, a, b, nb1, nb2, nb3, offset, false); // 调用内部函数实现累加操作
}

// 定义一个函数，实现两个张量的累加操作，并将结果存储在第一个张量中，返回结果张量
struct ggml_tensor * ggml_acc_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b,
        size_t               nb1,
        size_t               nb2,
        size_t               nb3,
        size_t               offset) {
    # 调用 ggml_acc_impl 函数，传入参数 ctx, a, b, nb1, nb2, nb3, offset, true，并返回结果
    return ggml_acc_impl(ctx, a, b, nb1, nb2, nb3, offset, true);
// ggml_sub

// 实现两个张量相减的函数
static struct ggml_tensor * ggml_sub_impl(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b,
        bool inplace) {
    // 断言两个张量具有相同的形状
    GGML_ASSERT(ggml_are_same_shape(a, b));

    // 初始化一个布尔变量，用于标记是否为节点
    bool is_node = false;

    // 如果不是原地操作，并且其中一个张量具有梯度，则标记为节点
    if (!inplace && (a->grad || b->grad)) {
        is_node = true;
    }

    // 根据是否原地操作选择复制张量或者创建视图张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置结果张量的操作类型为减法
    result->op   = GGML_OP_SUB;
    // 如果是节点，则为结果张量分配一个梯度张量
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    return result;
}

// 对外接口，实现两个张量相减的函数
struct ggml_tensor * ggml_sub(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b) {
    return ggml_sub_impl(ctx, a, b, false);
}

// 对外接口，实现原地操作的两个张量相减的函数
struct ggml_tensor * ggml_sub_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b) {
    return ggml_sub_impl(ctx, a, b, true);
}

// ggml_mul

// 实现两个张量相乘的函数
static struct ggml_tensor * ggml_mul_impl(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b,
        bool inplace) {
    // TODO: support less-strict constraint
    //       GGML_ASSERT(ggml_can_repeat(b, a));
    // 断言张量 b 可以重复行以匹配张量 a
    GGML_ASSERT(ggml_can_repeat_rows(b, a));

    // 初始化一个布尔变量，用于标记是否为节点
    bool is_node = false;

    // 如果不是原地操作，并且其中一个张量具有梯度
    if (!inplace && (a->grad || b->grad)) {
        // TODO: support backward pass for broadcasting
        // 断言两个张量具有相同的形状
        GGML_ASSERT(ggml_are_same_shape(a, b));
        is_node = true;
    }

    // 如果是原地操作，则断言不是节点
    if (inplace) {
        GGML_ASSERT(!is_node);
    }

    // 根据是否原地操作选择复制张量或者创建视图张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置结果张量的操作类型为乘法
    result->op   = GGML_OP_MUL;
    // 如果是节点，则为结果张量分配一个梯度张量
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    return result;
}

// 对外接口，实现两个张量相乘的函数
struct ggml_tensor * ggml_mul(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    return ggml_mul_impl(ctx, a, b, false);
}
// 对两个张量进行就地乘法操作，返回结果张量
struct ggml_tensor * ggml_mul_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    return ggml_mul_impl(ctx, a, b, true);
}

// ggml_div

// 实现张量的除法操作
static struct ggml_tensor * ggml_div_impl(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b,
        bool inplace) {
    // 断言两个张量具有相同的形状
    GGML_ASSERT(ggml_are_same_shape(a, b));

    bool is_node = false;

    // 如果不是就地操作且其中一个张量具有梯度，则设置 is_node 为 true
    if (!inplace && (a->grad || b->grad)) {
        is_node = true;
    }

    // 如果是就地操作，则断言 is_node 为 false
    if (inplace) {
        GGML_ASSERT(!is_node);
    }

    // 根据是否就地操作，创建一个新的结果张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置结果张量的操作类型为除法
    result->op   = GGML_OP_DIV;
    // 如果 is_node 为 true，则为结果张量创建一个梯度张量，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;
    result->src[1] = b;

    return result;
}

// 对两个张量进行除法操作，返回结果张量
struct ggml_tensor * ggml_div(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    return ggml_div_impl(ctx, a, b, false);
}

// 对两个张量进行就地除法操作，返回结果张量
struct ggml_tensor * ggml_div_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    return ggml_div_impl(ctx, a, b, true);
}

// ggml_sqr

// 实现张量的平方操作
static struct ggml_tensor * ggml_sqr_impl(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        bool inplace) {
    bool is_node = false;

    // 如果不是就地操作且张量具有梯度，则设置 is_node 为 true
    if (!inplace && (a->grad)) {
        is_node = true;
    }

    // 根据是否就地操作，创建一个新的结果张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置结果张量的操作类型为平方
    result->op   = GGML_OP_SQR;
    // 如果 is_node 为 true，则为结果张量创建一个梯度张量，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;

    return result;
}

// 对张量进行平方操作，返回结果张量
struct ggml_tensor * ggml_sqr(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_sqr_impl(ctx, a, false);
}

// 对张量进行就地平方操作，返回结果张量
struct ggml_tensor * ggml_sqr_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_sqr_impl(ctx, a, true);
}

// ggml_sqrt
// 计算平方根的实现函数
static struct ggml_tensor * ggml_sqrt_impl(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        bool inplace) {
    // 初始化节点标志为假
    bool is_node = false;

    // 如果不是原地计算且存在梯度，则将节点标志设置为真
    if (!inplace && (a->grad)) {
        is_node = true;
    }

    // 根据原地计算标志选择创建新的视图张量或者复制张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置结果张量的操作为平方根
    result->op   = GGML_OP_SQRT;
    // 如果是节点，则复制结果张量作为梯度，否则梯度为空
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为输入张量
    result->src[0] = a;

    // 返回结果张量
    return result;
}

// 计算平方根的接口函数
struct ggml_tensor * ggml_sqrt(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    // 调用平方根的实现函数，不进行原地计算
    return ggml_sqrt_impl(ctx, a, false);
}

// 原地计算平方根的接口函数
struct ggml_tensor * ggml_sqrt_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    // 调用平方根的实现函数，进行原地计算
    return ggml_sqrt_impl(ctx, a, true);
}

// 计算对数的实现函数
static struct ggml_tensor * ggml_log_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        bool inplace) {
    // 初始化节点标志为假
    bool is_node = false;

    // 如果不是原地计算且存在梯度，则将节点标志设置为真
    if (!inplace && (a->grad)) {
        is_node = true;
    }

    // 根据原地计算标志选择创建新的视图张量或者复制张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置结果张量的操作为对数
    result->op   = GGML_OP_LOG;
    // 如果是节点，则复制结果张量作为梯度，否则梯度为空
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为输入张量
    result->src[0] = a;

    // 返回结果张量
    return result;
}

// 计算对数的接口函数
struct ggml_tensor * ggml_log(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    // 调用对数的实现函数，不进行原地计算
    return ggml_log_impl(ctx, a, false);
}

// 原地计算对数的接口函数
struct ggml_tensor * ggml_log_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    // 调用对数的实现函数，进行原地计算
    return ggml_log_impl(ctx, a, true);
}

// 计算张量的和
struct ggml_tensor * ggml_sum(
        struct ggml_context * ctx,
        struct ggml_tensor * a) {
    // 初始化节点标志为假
    bool is_node = false;

    // 如果存在梯度，则将节点标志设置为真
    if (a->grad) {
        is_node = true;
    }

    // 创建一个新的一维张量作为结果
    struct ggml_tensor * result = ggml_new_tensor_1d(ctx, a->type, 1);

    // 设置结果张量的操作为求和
    result->op   = GGML_OP_SUM;
    // 如果是节点，则复制结果张量作为梯度，否则梯度为空
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为输入张量
    result->src[0] = a;

    // 返回结果张量
    return result;
}

// 计算行的和
// 对输入的张量进行行求和操作，返回结果张量
struct ggml_tensor * ggml_sum_rows(
        struct ggml_context * ctx,
        struct ggml_tensor * a) {
    // 初始化节点标志为假
    bool is_node = false;

    // 如果输入张量具有梯度，则将节点标志设置为真
    if (a->grad) {
        is_node = true;
    }

    // 初始化一个长度为4的数组ne，全部元素为1
    int64_t ne[4] = {1,1,1,1};
    // 遍历输入张量的维度，更新ne数组对应位置的值
    for (int i=1; i<a->n_dims; ++i) {
        ne[i] = a->ne[i];
    }

    // 创建一个新的张量result，与输入张量a具有相同的类型和维度
    struct ggml_tensor * result = ggml_new_tensor(ctx, a->type, a->n_dims, ne);

    // 设置result的操作类型为求和行
    result->op   = GGML_OP_SUM_ROWS;
    // 如果节点标志为真，则将result的梯度设置为result的副本，否则设置为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置result的源张量为输入张量a
    result->src[0] = a;

    // 返回结果张量
    return result;
}

// ggml_mean

// ... (以下代码同上，对ggml_mean, ggml_argmax, ggml_repeat函数进行相同的注释)
    # 如果是节点，则复制张量并赋值给 grad，否则赋值为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    # 将 a 赋值给 result 的第一个源
    result->src[0] = a;
    # 返回 result
    return result;
// ggml_repeat_back

// 重复反向传播函数，用于计算梯度
struct ggml_tensor * ggml_repeat_back(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b) {
    // 确保可以进行重复操作
    GGML_ASSERT(ggml_can_repeat(b, a));

    // 初始化节点标志为假
    bool is_node = false;

    // 如果 a 有梯度，则将节点标志设为真
    if (a->grad) {
        is_node = true;
    }

    // 如果 a 和 b 的形状相同且 a 不是节点，则返回 a
    if (ggml_are_same_shape(a, b) && !is_node) {
        return a;
    }

    // 创建一个新的张量用于存储结果
    struct ggml_tensor * result = ggml_new_tensor(ctx, a->type, b->n_dims, b->ne);

    // 设置结果张量的操作类型为重复反向传播
    result->op   = GGML_OP_REPEAT_BACK;
    // 如果是节点，则复制结果张量作为梯度，否则梯度为空
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为 a
    result->src[0] = a;

    // 返回结果张量
    return result;
}

// ggml_concat

// 连接函数，用于连接两个张量
struct ggml_tensor * ggml_concat(
    struct ggml_context* ctx,
    struct ggml_tensor* a,
    struct ggml_tensor* b) {
    // 确保 a 和 b 的形状适合连接
    GGML_ASSERT(a->ne[0] == b->ne[0] && a->ne[1] == b->ne[1] && a->ne[3] == b->ne[3]);

    // 初始化节点标志为假
    bool is_node = false;

    // 如果 a 或 b 有梯度，则将节点标志设为真
    if (a->grad || b->grad) {
        is_node = true;
    }

    // 创建一个新的张量用于存储结果
    struct ggml_tensor * result = ggml_new_tensor_4d(ctx, a->type, a->ne[0], a->ne[1], a->ne[2] + b->ne[2], a->ne[3]);

    // 设置结果张量的操作类型为连接
    result->op = GGML_OP_CONCAT;
    // 如果是节点，则复制结果张量作为梯度，否则梯度为空
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    // 返回结果张量
    return result;
}

// ggml_abs

// 绝对值函数，用于计算张量的绝对值
struct ggml_tensor * ggml_abs(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    // 调用 ggml_unary 函数，传入绝对值操作类型
    return ggml_unary(ctx, a, GGML_UNARY_OP_ABS);
}

// 原地绝对值函数，用于计算张量的绝对值并覆盖原始张量
struct ggml_tensor * ggml_abs_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    // 调用 ggml_unary_inplace 函数，传入绝对值操作类型
    return ggml_unary_inplace(ctx, a, GGML_UNARY_OP_ABS);
}

// ggml_sgn

// 符号函数，用于计算张量的符号
struct ggml_tensor * ggml_sgn(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    // 调用 ggml_unary 函数，传入符号操作类型
    return ggml_unary(ctx, a, GGML_UNARY_OP_SGN);
}

// 原地符号函数，用于计算张量的符号并覆盖原始张量
struct ggml_tensor * ggml_sgn_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    // 调用 ggml_unary_inplace 函数，传入符号操作类型
    return ggml_unary_inplace(ctx, a, GGML_UNARY_OP_SGN);
}

// ggml_neg

// 取负函数，用于计算张量的相反数
struct ggml_tensor * ggml_neg(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    # 调用 ggml_unary 函数，传入上下文 ctx、操作数 a，以及一元操作符 NEG
    return ggml_unary(ctx, a, GGML_UNARY_OP_NEG);
// ggml_neg_inplace函数，对输入的张量进行就地负数操作，返回结果张量
struct ggml_tensor * ggml_neg_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_unary_inplace(ctx, a, GGML_UNARY_OP_NEG);
}

// ggml_step函数，对输入的张量进行阶跃函数操作，返回结果张量
struct ggml_tensor * ggml_step(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_unary(ctx, a, GGML_UNARY_OP_STEP);
}

// ggml_step_inplace函数，对输入的张量进行就地阶跃函数操作，返回结果张量
struct ggml_tensor * ggml_step_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_unary_inplace(ctx, a, GGML_UNARY_OP_STEP);
}

// ggml_tanh函数，对输入的张量进行双曲正切函数操作，返回结果张量
struct ggml_tensor * ggml_tanh(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_unary(ctx, a, GGML_UNARY_OP_TANH);
}

// ggml_tanh_inplace函数，对输入的张量进行就地双曲正切函数操作，返回结果张量
struct ggml_tensor * ggml_tanh_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_unary_inplace(ctx, a, GGML_UNARY_OP_TANH);
}

// ggml_elu函数，对输入的张量进行ELU函数操作，返回结果张量
struct ggml_tensor * ggml_elu(
    struct ggml_context * ctx,
    struct ggml_tensor  * a) {
    return ggml_unary(ctx, a, GGML_UNARY_OP_ELU);
}

// ggml_elu_inplace函数，对输入的张量进行就地ELU函数操作，返回结果张量
struct ggml_tensor * ggml_elu_inplace(
    struct ggml_context * ctx,
    struct ggml_tensor  * a) {
    return ggml_unary_inplace(ctx, a, GGML_UNARY_OP_ELU);
}

// ggml_relu函数，对输入的张量进行ReLU函数操作，返回结果张量
struct ggml_tensor * ggml_relu(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_unary(ctx, a, GGML_UNARY_OP_RELU);
}

// ggml_relu_inplace函数，对输入的张量进行就地ReLU函数操作，返回结果张量
struct ggml_tensor * ggml_relu_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_unary_inplace(ctx, a, GGML_UNARY_OP_RELU);
}

// ggml_leaky函数，对输入的张量进行Leaky ReLU函数操作，返回结果张量
struct ggml_tensor * ggml_leaky(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_unary(ctx, a, GGML_UNARY_OP_LEAKY);
}

// ggml_gelu函数，对输入的张量进行GELU函数操作，返回结果张量
struct ggml_tensor * ggml_gelu(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_unary(ctx, a, GGML_UNARY_OP_GELU);
}

// ggml_gelu_inplace函数，对输入的张量进行就地GELU函数操作，返回结果张量
struct ggml_tensor * ggml_gelu_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    # 调用 ggml_unary_inplace 函数，对输入的 a 执行 GGML_UNARY_OP_GELU 操作，并将结果返回
    return ggml_unary_inplace(ctx, a, GGML_UNARY_OP_GELU);
// ggml_gelu_quick

// 使用快速 GELU 函数对输入张量进行操作
struct ggml_tensor * ggml_gelu_quick(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_unary(ctx, a, GGML_UNARY_OP_GELU_QUICK);
}

// 使用快速 GELU 函数对输入张量进行原地操作
struct ggml_tensor * ggml_gelu_quick_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_unary_inplace(ctx, a, GGML_UNARY_OP_GELU_QUICK);
}

// ggml_silu

// 使用 SiLU 函数对输入张量进行操作
struct ggml_tensor * ggml_silu(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_unary(ctx, a, GGML_UNARY_OP_SILU);
}

// 使用 SiLU 函数对输入张量进行原地操作
struct ggml_tensor * ggml_silu_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_unary_inplace(ctx, a, GGML_UNARY_OP_SILU);
}

// ggml_silu_back

// 使用 SiLU 函数的反向传播
struct ggml_tensor * ggml_silu_back(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    bool is_node = false;

    if (a->grad || b->grad) {
        // 如果输入张量或输出张量需要梯度计算，则实现反向传播
        // TODO: implement backward
        is_node = true;
    }

    // 复制输入张量，准备进行反向传播
    struct ggml_tensor * result = ggml_dup_tensor(ctx, a);

    result->op   = GGML_OP_SILU_BACK;
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;
    result->src[1] = b;

    return result;
}

// ggml_norm

// 实现对输入张量进行范数计算
static struct ggml_tensor * ggml_norm_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        float eps,
        bool inplace) {
    bool is_node = false;

    if (!inplace && (a->grad)) {
        // 如果不是原地操作且需要梯度计算，则抛出异常
        GGML_ASSERT(false); // TODO: implement backward
        is_node = true;
    }

    // 根据是否原地操作选择复制输入张量或创建视图张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    ggml_set_op_params(result, &eps, sizeof(eps));

    result->op   = GGML_OP_NORM;
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;

    return result;
}

// 对输入张量进行范数计算
struct ggml_tensor * ggml_norm(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        float eps) {
    # 调用 ggml_norm_impl 函数，传入参数 ctx, a, eps, false，并返回结果
    return ggml_norm_impl(ctx, a, eps, false);
}

// ggml_norm_inplace函数，对输入的张量进行归一化操作
struct ggml_tensor * ggml_norm_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        float eps) {
    // 调用ggml_norm_impl函数进行归一化操作，并指定inplace参数为true
    return ggml_norm_impl(ctx, a, eps, true);
}

// ggml_rms_norm_impl函数，实现RMS归一化操作
static struct ggml_tensor * ggml_rms_norm_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        float eps,
        bool inplace) {
    // 初始化is_node变量为false
    bool is_node = false;

    // 如果不是inplace操作且输入张量有梯度，则将is_node设置为true
    if (!inplace && (a->grad)) {
        is_node = true;
    }

    // 根据inplace参数选择复制输入张量或者创建视图张量，并赋值给result
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置result的操作参数为eps
    ggml_set_op_params(result, &eps, sizeof(eps));

    // 设置result的操作类型为GGML_OP_RMS_NORM
    result->op   = GGML_OP_RMS_NORM;
    // 如果is_node为true，则result的梯度为result的复制张量，否则为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置result的源张量为a
    result->src[0] = a;

    // 返回result
    return result;
}

// ggml_rms_norm函数，对输入的张量进行RMS归一化操作
struct ggml_tensor * ggml_rms_norm(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        float  eps) {
    // 调用ggml_rms_norm_impl函数进行RMS归一化操作，并指定inplace参数为false
    return ggml_rms_norm_impl(ctx, a, eps, false);
}

// ggml_rms_norm_inplace函数，对输入的张量进行RMS归一化操作（inplace版本）
struct ggml_tensor * ggml_rms_norm_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        float eps) {
    // 调用ggml_rms_norm_impl函数进行RMS归一化操作，并指定inplace参数为true
    return ggml_rms_norm_impl(ctx, a, eps, true);
}

// ggml_rms_norm_back函数，对输入的张量进行RMS归一化的反向传播操作
struct ggml_tensor * ggml_rms_norm_back(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        float  eps) {
    // 初始化is_node变量为false
    bool is_node = false;

    // 如果输入张量a有梯度，则将is_node设置为true
    if (a->grad) {
        // TODO: implement backward
        is_node = true;
    }

    // 复制输入张量a，并赋值给result
    struct ggml_tensor * result = ggml_dup_tensor(ctx, a);

    // 设置result的操作参数为eps
    ggml_set_op_params(result, &eps, sizeof(eps));

    // 设置result的操作类型为GGML_OP_RMS_NORM_BACK
    result->op   = GGML_OP_RMS_NORM_BACK;
    // 如果is_node为true，则result的梯度为result的复制张量，否则为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置result的源张量为a和b
    result->src[0] = a;
    result->src[1] = b;

    // 返回result
    return result;
}

// ggml_group_norm_impl函数，实现分组归一化操作
static struct ggml_tensor * ggml_group_norm_impl(
    struct ggml_context * ctx,
    struct ggml_tensor * a,
    int n_groups,
    bool inplace) {
    // 初始化is_node变量为false
    bool is_node = false;
    # 如果不是原地操作且输入张量有梯度
    if (!inplace && (a->grad)) {
        # 断言，提示需要实现反向传播
        GGML_ASSERT(false); // TODO: implement backward
        # 设置节点标志为真
        is_node = true;
    }

    # 根据是否原地操作选择创建视图张量或复制张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    # 设置结果张量的操作类型为组归一化
    result->op = GGML_OP_GROUP_NORM;
    # 设置操作参数为分组数
    result->op_params[0] = n_groups;
    # 如果是节点，设置梯度为结果张量的复制，否则为空
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    # 设置结果张量的输入源为输入张量
    result->src[0] = a;
    # 设置第二个输入源为空，待实现
    result->src[1] = NULL; // TODO: maybe store epsilon here?

    # 返回结果张量
    return result;
// 定义一个函数，实现组归一化操作，返回处理后的张量
struct ggml_tensor * ggml_group_norm(
    struct ggml_context * ctx,
    struct ggml_tensor * a,
    int n_groups) {
    return ggml_group_norm_impl(ctx, a, n_groups, false);
}

// 定义一个函数，实现原地组归一化操作，返回处理后的张量
struct ggml_tensor * ggml_group_norm_inplace(
    struct ggml_context * ctx,
    struct ggml_tensor * a,
    int n_groups) {
    return ggml_group_norm_impl(ctx, a, n_groups, true);
}

// 定义一个函数，实现矩阵相乘操作，返回处理后的张量
struct ggml_tensor * ggml_mul_mat(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    // 断言输入张量可以进行矩阵相乘操作
    GGML_ASSERT(ggml_can_mul_mat(a, b));
    // 断言输入张量不是转置的
    GGML_ASSERT(!ggml_is_transposed(a));

    bool is_node = false;

    // 如果输入张量有梯度信息，则设置is_node为true
    if (a->grad || b->grad) {
        is_node = true;
    }

    // 创建一个新的张量用于存储矩阵相乘的结果
    const int64_t ne[4] = { a->ne[1], b->ne[1], b->ne[2], b->ne[3] };
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F32, MAX(a->n_dims, b->n_dims), ne);

    // 设置结果张量的操作类型为矩阵相乘
    result->op   = GGML_OP_MUL_MAT;
    // 如果is_node为true，则为结果张量设置梯度信息
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量
    result->src[0] = a;
    result->src[1] = b;
    result->src[2] = NULL;
    result->src[3] = NULL;

    return result;
}

// 定义一个函数，实现特殊矩阵相乘操作，返回处理后的张量
struct ggml_tensor * ggml_mul_mat_special(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        struct ggml_tensor  * c,
        struct ggml_tensor  * d,
        struct ggml_tensor  * ref) {
    // 如果输入张量a或b为空，则返回空指针
    if (a == NULL || b == NULL)
        return NULL;

    bool is_node = false;

    // 如果输入张量a或b有梯度信息，则设置is_node为true
    if (a->grad || b->grad) {
        is_node = true;
    }

    // 创建一个新的张量用于存储矩阵相乘的结果
    const int64_t ne[4] = { ref->ne[1], b->ne[1], b->ne[2], b->ne[3] };
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F32, MAX(a->n_dims, b->n_dims), ne);

    // 设置结果张量的操作类型为矩阵相乘
    result->op   = GGML_OP_MUL_MAT;
    // 如果is_node为true，则为结果张量设置梯度信息
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量
    result->src[0] = a;
    result->src[1] = b;
    result->src[2] = c;
    result->src[3] = d;

    return result;
}
// 计算两个张量的矩阵乘法，返回结果张量
struct ggml_tensor * ggml_mul_mat_idx(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        struct ggml_tensor  * c,
        struct ggml_tensor  * d) {
    // 如果输入张量 a 或 b 为空，则返回空指针
    if (a == NULL || b == NULL)
        return NULL;
    // 断言输入张量 a 不是转置的
    GGML_ASSERT(!ggml_is_transposed(a));

    // 初始化布尔变量 is_node 为 false
    bool is_node = false;

    // 如果输入张量 a 或 b 具有梯度，则将 is_node 设置为 true
    if (a->grad || b->grad) {
        is_node = true;
    }

    // 创建一个包含四个元素的整型数组 ne，分别为 a 和 b 的维度信息
    const int64_t ne[4] = { a->ne[1], b->ne[1], b->ne[2], b->ne[3] };
    // 创建一个新的张量 result，用于存储矩阵乘法的结果
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F32, MAX(a->n_dims, b->n_dims), ne);

    // 设置结果张量的操作类型为矩阵乘法
    result->op   = GGML_OP_MUL_MAT;
    // 如果 is_node 为 true，则将结果张量的梯度设置为自身的副本，否则设置为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量，分别为 a、b、c、d
    result->src[0] = a;
    result->src[1] = b;
    result->src[2] = c;
    result->src[3] = d;

    // 返回结果张量
    return result;
}

// ggml_axpy

// 计算张量的线性组合，返回结果张量
struct ggml_tensor * ggml_axpy(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        struct ggml_tensor  * c,
        struct ggml_tensor  * d) {
    // 如果输入张量 a 或 b 为空，则返回空指针
    if (a == NULL || b == NULL)
        return NULL;
    // 断言输入张量 a 不是转置的
    GGML_ASSERT(!ggml_is_transposed(a));
    // 初始化布尔变量 is_node 为 false
    bool is_node = false;

    // 如果输入张量 a 或 b 具有梯度，则将 is_node 设置为 true
    if (a->grad || b->grad) {
        is_node = true;
    }

    // 创建一个包含四个元素的整型数组 ne，分别为 a 和 b 的维度信息
    const int64_t ne[4] = { a->ne[0], b->ne[1], b->ne[2], b->ne[3] };
    // 创建一个新的张量 result，用于存储线性组合的结果
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F32, MAX(a->n_dims, b->n_dims), ne);

    // 设置结果张量的操作类型为线性组合
    result->op   = GGML_OP_AXPY;
    // 如果 is_node 为 true，则将结果张量的梯度设置为自身的副本，否则设置为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量，分别为 a、b、c、d
    result->src[0] = a;
    result->src[1] = b;
    result->src[2] = c;
    result->src[3] = d;

    // 返回结果张量
    return result;
}

// ggml_out_prod

// 计算两个张量的外积，返回结果张量
struct ggml_tensor * ggml_out_prod(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    // 断言输入张量 a 和 b 可以进行外积运算
    GGML_ASSERT(ggml_can_out_prod(a, b));
    // 断言输入张量 a 不是转置的
    GGML_ASSERT(!ggml_is_transposed(a));

    // 初始化布尔变量 is_node 为 false
    bool is_node = false;

    // 如果输入张量 a 或 b 具有梯度，则将 is_node 设置为 true
    if (a->grad || b->grad) {
        is_node = true;
    }

    // a 可以广播到 b 的 ne[2] 和 ne[3]，使用 b->ne[2] 和 b->ne[3]
    // 定义一个包含4个元素的常量整型数组ne，分别为a->ne[0], b->ne[0], b->ne[2], b->ne[3]
    const int64_t ne[4] = { a->ne[0], b->ne[0], b->ne[2], b->ne[3] };
    // 创建一个新的张量result，数据类型为GGML_TYPE_F32，维度为a和b的维度中较大的那个，形状为ne
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F32, MAX(a->n_dims, b->n_dims), ne);

    // 设置result的操作类型为GGML_OP_OUT_PROD
    result->op   = GGML_OP_OUT_PROD;
    // 如果是节点，则result的梯度为result的副本，否则为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置result的源张量为a和b
    result->src[0] = a;
    result->src[1] = b;

    // 返回result张量
    return result;
// ggml_scale

// 实现按比例缩放张量的函数
static struct ggml_tensor * ggml_scale_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        bool inplace) {
    // 断言张量 b 是标量
    GGML_ASSERT(ggml_is_scalar(b));
    // 断言张量 a 是一维张量
    GGML_ASSERT(ggml_is_padded_1d(a));

    // 初始化变量 is_node
    bool is_node = false;

    // 如果张量 a 或 b 具有梯度，则将 is_node 设置为 true
    if (a->grad || b->grad) {
        is_node = true;
    }

    // 根据 inplace 参数选择是创建 a 的视图还是复制 a
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置结果张量的操作类型为 GGML_OP_SCALE
    result->op   = GGML_OP_SCALE;
    // 如果 is_node 为 true，则为结果张量设置梯度张量，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    // 返回结果张量
    return result;
}

// 对外接口函数，实现按比例缩放张量
struct ggml_tensor * ggml_scale(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b) {
    return ggml_scale_impl(ctx, a, b, false);
}

// 对外接口函数，实现按比例缩放张量（原地操作）
struct ggml_tensor * ggml_scale_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b) {
    return ggml_scale_impl(ctx, a, b, true);
}

// ggml_set

// 实现设置张量值的函数
static struct ggml_tensor * ggml_set_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        size_t                nb1,
        size_t                nb2,
        size_t                nb3,
        size_t                offset,
        bool inplace) {
    // 断言张量 a 的元素数量大于等于张量 b 的元素数量
    GGML_ASSERT(ggml_nelements(a) >= ggml_nelements(b));

    // 初始化变量 is_node
    bool is_node = false;

    // 如果张量 a 或 b 具有梯度，则将 is_node 设置为 true
    if (a->grad || b->grad) {
        is_node = true;
    }

    // 创建目标张量的视图或复制目标张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置操作参数
    int32_t params[] = { nb1, nb2, nb3, offset, inplace ? 1 : 0 };
    ggml_set_op_params(result, params, sizeof(params));

    // 设置结果张量的操作类型为 GGML_OP_SET
    result->op   = GGML_OP_SET;
    // 如果 is_node 为 true，则为结果张量设置梯度张量，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    // 返回结果张量
    return result;
}
# 设置两个张量的值，并返回结果张量
struct ggml_tensor * ggml_set(
        struct ggml_context * ctx,
        struct ggml_tensor *  a,
        struct ggml_tensor *  b,
        size_t                nb1,
        size_t                nb2,
        size_t                nb3,
        size_t                offset) {
    return ggml_set_impl(ctx, a, b, nb1, nb2, nb3, offset, false);
}

# 在原地设置两个张量的值，并返回结果张量
struct ggml_tensor * ggml_set_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor *  a,
        struct ggml_tensor *  b,
        size_t                nb1,
        size_t                nb2,
        size_t                nb3,
        size_t                offset) {
    return ggml_set_impl(ctx, a, b, nb1, nb2, nb3, offset, true);
}

# 设置两个一维张量的值，并返回结果张量
struct ggml_tensor * ggml_set_1d(
        struct ggml_context * ctx,
        struct ggml_tensor *  a,
        struct ggml_tensor *  b,
        size_t                offset) {
    return ggml_set_impl(ctx, a, b, a->nb[1], a->nb[2], a->nb[3], offset, false);
}

# 在原地设置两个一维张量的值，并返回结果张量
struct ggml_tensor * ggml_set_1d_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor *  a,
        struct ggml_tensor *  b,
        size_t                offset) {
    return ggml_set_impl(ctx, a, b, a->nb[1], a->nb[2], a->nb[3], offset, true);
}

# 设置两个二维张量的值，并返回结果张量
struct ggml_tensor * ggml_set_2d(
        struct ggml_context * ctx,
        struct ggml_tensor *  a,
        struct ggml_tensor *  b,
        size_t                nb1,
        size_t                offset) {
    return ggml_set_impl(ctx, a, b, nb1, a->nb[2], a->nb[3], offset, false);
}

# 在原地设置两个二维张量的值，并返回结果张量
struct ggml_tensor * ggml_set_2d_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor *  a,
        struct ggml_tensor *  b,
        size_t                nb1,
        size_t                offset) {
    return ggml_set_impl(ctx, a, b, nb1, a->nb[2], a->nb[3], offset, false);
}

# 复制张量的实现函数
static struct ggml_tensor * ggml_cpy_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        bool inplace) {
    # 断言两个张量的元素个数相等
    GGML_ASSERT(ggml_nelements(a) == ggml_nelements(b));

    # 初始化一个布尔变量，用于标记是否为节点
    bool is_node = false;

    # 如果不是原地操作，并且 a 或 b 其中一个具有梯度信息，则标记为节点
    if (!inplace && (a->grad || b->grad)) {
        is_node = true;
    }

    # 创建目标张量的视图
    struct ggml_tensor * result = ggml_view_tensor(ctx, b);
    # 如果张量 b 的名称长度大于 0，则使用格式化的名称
    if (strlen(b->name) > 0) {
        ggml_format_name(result, "%s (copy of %s)", b->name, a->name);
    } else {
        ggml_format_name(result, "%s (copy)", a->name);
    }

    # 设置目标张量的操作类型为复制
    result->op   = GGML_OP_CPY;
    # 如果是节点，则复制目标张量的梯度信息
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    # 设置目标张量的源张量为 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    # 返回结果张量
    return result;
// 复制张量 b 的数据到张量 a，返回结果
struct ggml_tensor * ggml_cpy(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b) {
    return ggml_cpy_impl(ctx, a, b, false);
}

// 在原地复制张量 b 的数据到张量 a，返回结果
struct ggml_tensor * ggml_cpy_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b) {
    return ggml_cpy_impl(ctx, a, b, true);
}

// ggml_cont

// 实现创建连续张量的函数
static struct ggml_tensor * ggml_cont_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        bool inplace) {
    bool is_node = false;

    // 如果不是原地操作且张量 a 有梯度，则设置 is_node 为 true
    if (!inplace && a->grad) {
        is_node = true;
    }

    // 根据是否原地操作选择创建视图张量或复制张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);
    // 格式化结果张量的名称
    ggml_format_name(result, "%s (cont)", a->name);

    // 设置结果张量的操作类型为 GGML_OP_CONT
    result->op   = GGML_OP_CONT;
    // 如果是节点，则为结果张量设置梯度张量，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为张量 a
    result->src[0] = a;

    return result;
}

// 创建连续张量的函数
struct ggml_tensor * ggml_cont(
        struct ggml_context * ctx,
        struct ggml_tensor * a) {
    return ggml_cont_impl(ctx, a, false);
}

// 在原地创建连续张量的函数
struct ggml_tensor * ggml_cont_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor * a) {
    return ggml_cont_impl(ctx, a, true);
}

// 创建一维连续张量的函数
GGML_API struct ggml_tensor * ggml_cont_1d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0) {
    return ggml_cont_4d(ctx, a, ne0, 1, 1, 1);
}

// 创建二维连续张量的函数
GGML_API struct ggml_tensor * ggml_cont_2d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0,
        int64_t               ne1) {
    return ggml_cont_4d(ctx, a, ne0, ne1, 1, 1);
}

// 创建三维连续张量的函数
GGML_API struct ggml_tensor * ggml_cont_3d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0,
        int64_t               ne1,
        int64_t               ne2) {
    return ggml_cont_4d(ctx, a, ne0, ne1, ne2, 1);
}
// 4维连续张量操作，返回一个新的4维张量
struct ggml_tensor * ggml_cont_4d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0,
        int64_t               ne1,
        int64_t               ne2,
        int64_t               ne3) {
    // 断言输入张量的元素数量等于给定的4维形状的乘积
    GGML_ASSERT(ggml_nelements(a) == (ne0*ne1*ne2*ne3));

    // 初始化布尔变量is_node为false
    bool is_node = false;

    // 创建一个新的4维张量result，形状为给定的ne0, ne1, ne2, ne3
    struct ggml_tensor * result = ggml_new_tensor_4d(ctx, a->type, ne0, ne1, ne2, ne3);
    // 为result设置名称，格式为原张量名称 + " (cont)"
    ggml_format_name(result, "%s (cont)", a->name);

    // 设置result的操作类型为GGML_OP_CONT
    result->op   = GGML_OP_CONT;
    // 如果is_node为true，则为result设置梯度为result的副本，否则为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置result的源张量为a
    result->src[0] = a;

    // 返回result张量
    return result;
}

// ggml_reshape

// 重塑张量操作，返回一个新的张量
struct ggml_tensor * ggml_reshape(
        struct ggml_context * ctx,
        struct ggml_tensor * a,
        struct ggml_tensor * b) {
    // 断言张量a是连续的
    GGML_ASSERT(ggml_is_contiguous(a));
    // 断言张量a和b的元素数量相等
    GGML_ASSERT(ggml_nelements(a) == ggml_nelements(b));

    // 初始化布尔变量is_node为false
    bool is_node = false;

    // 如果张量a有梯度，则将is_node设置为true
    if (a->grad) {
        is_node = true;
    }

    // 如果张量b有梯度，则抛出异常
    if (b->grad) {
        // gradient propagation is not supported
        //GGML_ASSERT(false);
    }

    // 创建一个新的张量result，类型和形状与b相同，源张量为a
    struct ggml_tensor * result = ggml_new_tensor_impl(ctx, a->type, b->n_dims, b->ne, a, 0);
    // 为result设置名称，格式为原张量名称 + " (reshaped)"
    ggml_format_name(result, "%s (reshaped)", a->name);

    // 设置result的操作类型为GGML_OP_RESHAPE
    result->op   = GGML_OP_RESHAPE;
    // 如果is_node为true，则为result设置梯度为result的副本，否则为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置result的源张量为a
    result->src[0] = a;

    // 返回result张量
    return result;
}

// 重塑1维张量操作，返回一个新的张量
struct ggml_tensor * ggml_reshape_1d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0) {
    // 断言张量a是连续的
    GGML_ASSERT(ggml_is_contiguous(a));
    // 断言张量a的元素数量等于给定的ne0
    GGML_ASSERT(ggml_nelements(a) == ne0);

    // 初始化布尔变量is_node为false
    bool is_node = false;

    // 如果张量a有梯度，则将is_node设置为true
    if (a->grad) {
        is_node = true;
    }

    // 创建一个新的张量result，类型为a的类型，形状为给定的ne0，源张量为a
    const int64_t ne[1] = { ne0 };
    struct ggml_tensor * result = ggml_new_tensor_impl(ctx, a->type, 1, ne, a, 0);
    // 为result设置名称，格式为原张量名称 + " (reshaped)"
    ggml_format_name(result, "%s (reshaped)", a->name);

    // 设置result的操作类型为GGML_OP_RESHAPE
    # 如果是节点，将结果的梯度设置为结果的副本，否则设置为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    # 将结果的第一个源张量设置为a
    result->src[0] = a;
    # 返回结果
    return result;
}

// 重塑二维张量
struct ggml_tensor * ggml_reshape_2d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0,
        int64_t               ne1) {
    // 断言张量是连续的
    GGML_ASSERT(ggml_is_contiguous(a));
    // 断言张量元素数量等于 ne0*ne1
    GGML_ASSERT(ggml_nelements(a) == ne0*ne1);

    bool is_node = false;

    // 如果张量有梯度，则设置 is_node 为 true
    if (a->grad) {
        is_node = true;
    }

    // 创建一个包含 ne0 和 ne1 的整数数组
    const int64_t ne[2] = { ne0, ne1 };
    // 创建一个新的张量，重塑原始张量 a
    struct ggml_tensor * result = ggml_new_tensor_impl(ctx, a->type, 2, ne, a, 0);
    // 格式化新张量的名称
    ggml_format_name(result, "%s (reshaped)", a->name);

    // 设置新张量的操作类型为重塑
    result->op   = GGML_OP_RESHAPE;
    // 如果 is_node 为 true，则复制新张量的梯度
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置新张量的源张量为原始张量 a
    result->src[0] = a;

    // 返回新张量
    return result;
}

// 重塑三维张量
struct ggml_tensor * ggml_reshape_3d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0,
        int64_t               ne1,
        int64_t               ne2) {
    // 断言张量是连续的
    GGML_ASSERT(ggml_is_contiguous(a));
    // 断言张量元素数量等于 ne0*ne1*ne2
    GGML_ASSERT(ggml_nelements(a) == ne0*ne1*ne2);

    bool is_node = false;

    // 如果张量有梯度，则设置 is_node 为 true
    if (a->grad) {
        is_node = true;
    }

    // 创建一个包含 ne0、ne1 和 ne2 的整数数组
    const int64_t ne[3] = { ne0, ne1, ne2 };
    // 创建一个新的张量，重塑原始张量 a
    struct ggml_tensor * result = ggml_new_tensor_impl(ctx, a->type, 3, ne, a, 0);
    // 格式化新张量的名称
    ggml_format_name(result, "%s (reshaped)", a->name);

    // 设置新张量的操作类型为重塑
    result->op   = GGML_OP_RESHAPE;
    // 如果 is_node 为 true，则复制新张量的梯度
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置新张量的源张量为原始张量 a
    result->src[0] = a;

    // 返回新张量
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
    // 断言张量是连续的
    GGML_ASSERT(ggml_is_contiguous(a));
    // 断言张量元素数量等于 ne0*ne1*ne2*ne3
    GGML_ASSERT(ggml_nelements(a) == ne0*ne1*ne2*ne3);

    bool is_node = false;

    // 如果张量有梯度，则设置 is_node 为 true
    if (a->grad) {
        is_node = true;
    }

    // 创建一个包含 ne0、ne1、ne2 和 ne3 的整数数组
    const int64_t ne[4] = { ne0, ne1, ne2, ne3 };
    // 创建一个新的张量，重塑原始张量 a
    struct ggml_tensor * result = ggml_new_tensor_impl(ctx, a->type, 4, ne, a, 0);
    # 格式化结果的名称，添加"(reshaped)"后缀
    ggml_format_name(result, "%s (reshaped)", a->name);

    # 设置结果的操作类型为reshape
    result->op   = GGML_OP_RESHAPE;
    
    # 如果是节点，则为结果分配一个梯度张量，否则为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    
    # 设置结果的源张量为a
    result->src[0] = a;
    
    # 返回结果
    return result;
// 定义一个静态函数 ggml_view_impl，接受上下文、张量、维度、维度大小和偏移量作为参数
static struct ggml_tensor * ggml_view_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   n_dims,
        const int64_t       * ne,
        size_t                offset) {

    // 初始化一个布尔变量 is_node，并赋值为 false
    bool is_node = false;

    // 如果张量 a 的梯度存在，则将 is_node 赋值为 true
    if (a->grad) {
        is_node = true;
    }

    // 调用 ggml_new_tensor_impl 函数创建一个新的张量 result，传入上下文、张量类型、维度、维度大小、张量 a 和偏移量作为参数
    struct ggml_tensor * result = ggml_new_tensor_impl(ctx, a->type, n_dims, ne, a, offset);
    // 调用 ggml_format_name 函数给 result 设置名称，传入原始张量的名称和 "(view)" 作为参数
    ggml_format_name(result, "%s (view)", a->name);

    // 调用 ggml_set_op_params 函数设置 result 的操作参数，传入偏移量和偏移量的大小作为参数
    ggml_set_op_params(result, &offset, sizeof(offset));

    // 设置 result 的操作类型为 GGML_OP_VIEW
    result->op   = GGML_OP_VIEW;
    // 如果 is_node 为 true，则将 result 的梯度设置为 result 的副本，否则设置为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置 result 的源张量为 a
    result->src[0] = a;

    // 返回 result
    return result;
}

// 定义一个函数 ggml_view_1d，接受上下文、张量、维度大小和偏移量作为参数
struct ggml_tensor * ggml_view_1d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0,
        size_t                offset) {

    // 调用 ggml_view_impl 函数创建一个新的张量 result，传入上下文、张量 a、1 作为维度、ne0 和偏移量作为参数
    struct ggml_tensor * result = ggml_view_impl(ctx, a, 1, &ne0, offset);

    // 返回 result
    return result;
}

// 定义一个函数 ggml_view_2d，接受上下文、张量、两个维度大小和两个偏移量作为参数
struct ggml_tensor * ggml_view_2d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0,
        int64_t               ne1,
        size_t                nb1,
        size_t                offset) {

    // 创建一个长度为 2 的整型数组 ne，存储 ne0 和 ne1
    const int64_t ne[2] = { ne0, ne1 };

    // 调用 ggml_view_impl 函数创建一个新的张量 result，传入上下文、张量 a、2 作为维度、ne 和偏移量作为参数
    struct ggml_tensor * result = ggml_view_impl(ctx, a, 2, ne, offset);

    // 设置 result 的第二个维度大小为 nb1
    result->nb[1] = nb1;
    // 设置 result 的第三个维度大小为 result 的第二个维度大小乘以 ne1
    result->nb[2] = result->nb[1]*ne1;
    // 设置 result 的第四个维度大小为 result 的第三个维度大小
    result->nb[3] = result->nb[2];

    // 返回 result
    return result;
}

// 定义一个函数 ggml_view_3d，接受上下文、张量、三个维度大小和三个偏移量作为参数
struct ggml_tensor * ggml_view_3d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0,
        int64_t               ne1,
        int64_t               ne2,
        size_t                nb1,
        size_t                nb2,
        size_t                offset) {

    // 创建一个长度为 3 的整型数组 ne，存储 ne0、ne1 和 ne2
    const int64_t ne[3] = { ne0, ne1, ne2 };

    // 调用 ggml_view_impl 函数创建一个新的张量 result，传入上下文、张量 a、3 作为维度、ne 和偏移量作为参数
    struct ggml_tensor * result = ggml_view_impl(ctx, a, 3, ne, offset);

    // 设置 result 的第二个维度大小为 nb1
    result->nb[1] = nb1;
    // 设置 result 的第三个维度大小为 nb2
    result->nb[2] = nb2;
    // 设置 result 的第四个维度大小为 result 的第三个维度大小乘以 ne2
    result->nb[3] = result->nb[2]*ne2;

    // 返回 result
    return result;
}
    # 返回result变量的值
    return result;
// 定义一个名为 ggml_view_4d 的函数，用于创建一个新的 4 维视图张量
struct ggml_tensor * ggml_view_4d(
        struct ggml_context * ctx,  // 上下文对象指针
        struct ggml_tensor  * a,    // 输入张量指针
        int64_t               ne0,   // 第一维的大小
        int64_t               ne1,   // 第二维的大小
        int64_t               ne2,   // 第三维的大小
        int64_t               ne3,   // 第四维的大小
        size_t                nb1,   // 第一维的步长
        size_t                nb2,   // 第二维的步长
        size_t                nb3,   // 第三维的步长
        size_t                offset) {  // 偏移量

    const int64_t ne[4] = { ne0, ne1, ne2, ne3 };  // 创建包含四个维度大小的数组

    // 调用 ggml_view_impl 函数创建新的视图张量
    struct ggml_tensor * result = ggml_view_impl(ctx, a, 4, ne, offset);

    // 设置新张量的步长
    result->nb[1] = nb1;
    result->nb[2] = nb2;
    result->nb[3] = nb3;

    // 返回新的视图张量
    return result;
}

// 定义一个名为 ggml_permute 的函数，用于对张量进行轴置换
struct ggml_tensor * ggml_permute(
        struct ggml_context * ctx,  // 上下文对象指针
        struct ggml_tensor  * a,    // 输入张量指针
        int                   axis0,  // 要置换的第一个轴
        int                   axis1,  // 要置换的第二个轴
        int                   axis2,  // 要置换的第三个轴
        int                   axis3) {  // 要置换的第四个轴
    GGML_ASSERT(axis0 >= 0 && axis0 < GGML_MAX_DIMS);  // 断言轴的合法性
    GGML_ASSERT(axis1 >= 0 && axis1 < GGML_MAX_DIMS);
    GGML_ASSERT(axis2 >= 0 && axis2 < GGML_MAX_DIMS);
    GGML_ASSERT(axis3 >= 0 && axis3 < GGML_MAX_DIMS);

    // 断言轴的不重复性
    GGML_ASSERT(axis0 != axis1);
    GGML_ASSERT(axis0 != axis2);
    GGML_ASSERT(axis0 != axis3);
    GGML_ASSERT(axis1 != axis2);
    GGML_ASSERT(axis1 != axis3);
    GGML_ASSERT(axis2 != axis3);

    bool is_node = false;  // 初始化布尔变量 is_node 为 false

    if (a->grad) {  // 如果输入张量有梯度信息
        is_node = true;  // 将 is_node 设置为 true
    }

    // 创建一个新的张量 result，作为输入张量的视图
    struct ggml_tensor * result = ggml_view_tensor(ctx, a);
    // 格式化新张量的名称
    ggml_format_name(result, "%s (permuted)", a->name);

    int ne[GGML_MAX_DIMS];  // 创建一个包含最大维度大小的数组
    int nb[GGML_MAX_DIMS];  // 创建一个包含最大维度步长的数组

    // 根据轴的顺序重新排列维度大小和步长
    ne[axis0] = a->ne[0];
    ne[axis1] = a->ne[1];
    ne[axis2] = a->ne[2];
    ne[axis3] = a->ne[3];

    nb[axis0] = a->nb[0];
    nb[axis1] = a->nb[1];
    nb[axis2] = a->nb[2];
    nb[axis3] = a->nb[3];

    // 更新新张量的维度大小和步长
    result->ne[0] = ne[0];
    result->ne[1] = ne[1];
    result->ne[2] = ne[2];
    result->ne[3] = ne[3];

    result->nb[0] = nb[0];
    result->nb[1] = nb[1];
    result->nb[2] = nb[2];
    result->nb[3] = nb[3];
    # 设置结果操作为排列操作
    result->op   = GGML_OP_PERMUTE;
    # 如果是节点，则为结果梯度分配一个新的张量，否则为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    # 设置结果的第一个源张量为a
    result->src[0] = a;

    # 创建一个包含轴参数的整型数组
    int32_t params[] = { axis0, axis1, axis2, axis3 };
    # 设置结果操作的参数为params数组，并指定数组的大小
    ggml_set_op_params(result, params, sizeof(params));

    # 返回结果
    return result;
}

// ggml_transpose

// 对输入的张量进行转置操作
struct ggml_tensor * ggml_transpose(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    // 初始化节点标志为假
    bool is_node = false;

    // 如果输入张量具有梯度，则将节点标志设置为真
    if (a->grad) {
        is_node = true;
    }

    // 创建一个新的张量作为结果，并将其视图设置为输入张量
    struct ggml_tensor * result = ggml_view_tensor(ctx, a);
    // 格式化结果张量的名称，添加“ (transposed)”后缀
    ggml_format_name(result, "%s (transposed)", a->name);

    // 交换结果张量的维度大小
    result->ne[0] = a->ne[1];
    result->ne[1] = a->ne[0];

    // 交换结果张量的维度边界
    result->nb[0] = a->nb[1];
    result->nb[1] = a->nb[0];

    // 设置结果张量的操作类型为转置
    result->op   = GGML_OP_TRANSPOSE;
    // 如果节点标志为真，则将结果张量的梯度设置为结果张量的副本，否则设置为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为输入张量
    result->src[0] = a;

    // 返回结果张量
    return result;
}

// ggml_get_rows

// 获取矩阵的指定行
struct ggml_tensor * ggml_get_rows(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    // 断言输入张量a为矩阵，输入张量b为向量，且b的类型为GGML_TYPE_I32
    GGML_ASSERT(ggml_is_matrix(a) && ggml_is_vector(b) && b->type == GGML_TYPE_I32);

    // 初始化节点标志为假
    bool is_node = false;

    // 如果输入张量a或b具有梯度，则将节点标志设置为真
    if (a->grad || b->grad) {
        is_node = true;
    }

    // 创建一个新的张量作为结果，其类型为GGML_TYPE_F32，大小为a的行数乘以b的元素个数
    struct ggml_tensor * result = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, a->ne[0], b->ne[0]);

    // 设置结果张量的操作类型为获取行
    result->op   = GGML_OP_GET_ROWS;
    // 如果节点标志为真，则将结果张量的梯度设置为结果张量的副本，否则设置为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为输入张量a和b
    result->src[0] = a;
    result->src[1] = b;

    // 返回结果张量
    return result;
}

// ggml_get_rows_back

// 获取矩阵的指定行的反向传播
struct ggml_tensor * ggml_get_rows_back(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        struct ggml_tensor  * c) {
    // 断言输入张量a为矩阵，输入张量b为向量，且b的类型为GGML_TYPE_I32
    GGML_ASSERT(ggml_is_matrix(a) && ggml_is_vector(b) && b->type == GGML_TYPE_I32);
    // 断言输入张量c为矩阵，且其行数与输入张量a的行数相同
    GGML_ASSERT(ggml_is_matrix(c) && (a->ne[0] == c->ne[0]));

    // 初始化节点标志为假
    bool is_node = false;

    // 如果输入张量a或b具有梯度，则将节点标志设置为真
    if (a->grad || b->grad) {
        is_node = true;
    }

    // 创建一个新的张量作为结果，其类型为GGML_TYPE_F32，大小为c的行数和列数
    struct ggml_tensor * result = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, c->ne[0], c->ne[1]);
    # 设置结果操作为获取行数的反向操作
    result->op   = GGML_OP_GET_ROWS_BACK;
    # 如果是节点，则将结果的梯度设置为结果的副本，否则设置为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    # 设置结果的第一个源操作数为a
    result->src[0] = a;
    # 设置结果的第二个源操作数为b
    result->src[1] = b;

    # 返回结果
    return result;
// ggml_diag

// 创建对角线张量
struct ggml_tensor * ggml_diag(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    // 断言张量 a 的第二维度大小为 1
    GGML_ASSERT(a->ne[1] == 1);
    // 初始化 is_node 为 false
    bool is_node = false;

    // 如果张量 a 有梯度信息，则将 is_node 设置为 true
    if (a->grad) {
        is_node = true;
    }

    // 创建一个新的张量 result，其维度为 a 的第一维度大小，类型与 a 相同
    const int64_t ne[4] = { a->ne[0], a->ne[0], a->ne[2], a->ne[3] };
    struct ggml_tensor * result = ggml_new_tensor(ctx, a->type, MAX(a->n_dims, 2), ne);

    // 设置 result 的操作类型为 GGML_OP_DIAG
    result->op   = GGML_OP_DIAG;
    // 如果 is_node 为 true，则将 result 的梯度设置为 result 的副本，否则设置为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置 result 的源张量为 a
    result->src[0] = a;

    // 返回创建的对角线张量 result
    return result;
}

// ggml_diag_mask_inf

// 创建对角线掩码张量
static struct ggml_tensor * ggml_diag_mask_inf_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   n_past,
        bool                  inplace) {
    // 初始化 is_node 为 false
    bool is_node = false;

    // 如果张量 a 有梯度信息，则将 is_node 设置为 true
    if (a->grad) {
        is_node = true;
    }

    // 创建一个新的张量 result，如果 inplace 为 true，则为 a 的视图，否则为 a 的副本
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置 result 的操作类型为 GGML_OP_DIAG_MASK_INF
    result->op   = GGML_OP_DIAG_MASK_INF;
    // 如果 is_node 为 true，则将 result 的梯度设置为 result 的副本，否则设置为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置 result 的源张量为 a
    result->src[0] = a;

    // 返回创建的对角线掩码张量 result
    return result;
}

// 创建对角线掩码张量
struct ggml_tensor * ggml_diag_mask_inf(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   n_past) {
    // 调用 ggml_diag_mask_inf_impl 函数，inplace 参数为 false
    return ggml_diag_mask_inf_impl(ctx, a, n_past, false);
}

// 创建对角线掩码张量（原地操作）
struct ggml_tensor * ggml_diag_mask_inf_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   n_past) {
    // 调用 ggml_diag_mask_inf_impl 函数，inplace 参数为 true
    return ggml_diag_mask_inf_impl(ctx, a, n_past, true);
}

// ggml_diag_mask_zero

// 创建对角线零掩码张量
static struct ggml_tensor * ggml_diag_mask_zero_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   n_past,
        bool                  inplace) {
    // 初始化 is_node 为 false
    bool is_node = false;

    // 如果张量 a 有梯度信息，则将 is_node 设置为 true
    if (a->grad) {
        is_node = true;
    }
    # 创建一个指向 ggml_tensor 结构体的指针 result，如果 inplace 为真则指向 a，否则指向 a 的副本
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);
    
    # 创建一个整型数组 params，包含 n_past 的值
    int32_t params[] = { n_past };
    # 设置 result 的操作参数为 params 数组，数组长度为 params 的长度
    ggml_set_op_params(result, params, sizeof(params));
    
    # 设置 result 的操作为 GGML_OP_DIAG_MASK_ZERO
    result->op   = GGML_OP_DIAG_MASK_ZERO;
    # 如果 is_node 为真，则为 result 的梯度 grad 分配内存，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    # 设置 result 的源数据为 a
    result->src[0] = a;
    
    # 返回 result 结构体指针
    return result;
// 返回一个新的张量，该张量是输入张量的对角线上的元素为零的版本
struct ggml_tensor * ggml_diag_mask_zero(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   n_past) {
    return ggml_diag_mask_zero_impl(ctx, a, n_past, false);
}

// 返回一个就地操作后的输入张量，该张量是输入张量的对角线上的元素为零的版本
struct ggml_tensor * ggml_diag_mask_zero_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   n_past) {
    return ggml_diag_mask_zero_impl(ctx, a, n_past, true);
}

// ggml_soft_max

// 实现 softmax 操作的内部函数
static struct ggml_tensor * ggml_soft_max_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        bool                  inplace) {
    bool is_node = false;

    // 如果输入张量具有梯度，则设置 is_node 为 true
    if (a->grad) {
        is_node = true;
    }

    // 创建一个新的张量，如果 inplace 为 true，则为输入张量的视图，否则为输入张量的副本
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置结果张量的操作类型为 GGML_OP_SOFT_MAX
    result->op   = GGML_OP_SOFT_MAX;
    // 如果 is_node 为 true，则为结果张量分配一个副本作为梯度，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为输入张量
    result->src[0] = a;

    return result;
}

// 返回一个新的张量，该张量是输入张量的 softmax 操作的结果
struct ggml_tensor * ggml_soft_max(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_soft_max_impl(ctx, a, false);
}

// 返回一个就地操作后的输入张量，该张量是输入张量的 softmax 操作的结果
struct ggml_tensor * ggml_soft_max_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a) {
    return ggml_soft_max_impl(ctx, a, true);
}

// ggml_soft_max_back

// 实现 softmax 反向传播操作的内部函数
static struct ggml_tensor * ggml_soft_max_back_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        bool                  inplace) {
    bool is_node = false;

    // 如果输入张量 a 或 b 具有梯度，则设置 is_node 为 true
    if (a->grad || b->grad) {
        is_node = true; // TODO : implement backward pass
    }

    // 创建一个新的张量，如果 inplace 为 true，则为输入张量 a 的视图，否则为输入张量 a 的副本
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置结果张量的操作类型为 GGML_OP_SOFT_MAX_BACK
    result->op   = GGML_OP_SOFT_MAX_BACK;
    // 如果 is_node 为 true，则为结果张量分配一个副本作为梯度，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为输入张量 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    return result;
}
// 对输入的两个张量进行 soft max 反向传播计算，返回结果张量
struct ggml_tensor * ggml_soft_max_back(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    return ggml_soft_max_back_impl(ctx, a, b, false);
}

// 对输入的两个张量进行 inplace 模式的 soft max 反向传播计算，返回结果张量
struct ggml_tensor * ggml_soft_max_back_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    return ggml_soft_max_back_impl(ctx, a, b, true);
}

// ggml_rope

// 实现 ggml_rope 操作的函数，根据输入参数进行相关计算
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
    GGML_ASSERT(ggml_is_vector(b));  // 断言张量 b 是一个向量
    GGML_ASSERT(b->type == GGML_TYPE_I32);  // 断言张量 b 的数据类型为 GGML_TYPE_I32
    GGML_ASSERT(a->ne[2] == b->ne[0]);  // 断言张量 a 的第三维度与张量 b 的第一维度相等

    bool is_node = false;  // 初始化布尔变量 is_node 为 false

    if (a->grad) {  // 如果张量 a 有梯度信息
        is_node = true;  // 将 is_node 设置为 true
    }

    // 根据 inplace 参数选择是创建新的张量还是对原张量进行操作，并赋值给 result
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    int32_t params[13] = { /*n_past*/ 0, n_dims, mode, n_ctx, n_orig_ctx };  // 初始化长度为 13 的整型数组 params
    memcpy(params +  5, &freq_base,    sizeof(float));  // 将 float 类型的参数复制到 params 数组中
    memcpy(params +  6, &freq_scale,   sizeof(float));
    memcpy(params +  7, &ext_factor,   sizeof(float));
    memcpy(params +  8, &attn_factor,  sizeof(float));
    memcpy(params +  9, &beta_fast,    sizeof(float));
    memcpy(params + 10, &beta_slow,    sizeof(float));
    memcpy(params + 11, &xpos_base,    sizeof(float));
    memcpy(params + 12, &xpos_down,    sizeof(bool));
    ggml_set_op_params(result, params, sizeof(params));  // 设置 result 的操作参数为 params 数组

    result->op   = GGML_OP_ROPE;  // 设置 result 的操作类型为 GGML_OP_ROPE
    # 如果是节点，则复制张量并将其赋给梯度，否则将梯度设为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    # 将a赋给result的第一个源
    result->src[0] = a;
    # 将b赋给result的第二个源
    result->src[1] = b;
    # 返回result
    return result;
# 定义一个函数 ggml_rope，接受上下文、两个张量、维度数量、模式和上下文数量作为参数，返回一个张量指针
struct ggml_tensor * ggml_rope(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                   n_dims,
        int                   mode,
        int                   n_ctx) {
    # 调用 ggml_rope_impl 函数，传入参数并返回结果
    return ggml_rope_impl(
        ctx, a, b, n_dims, mode, n_ctx, 0, 10000.0f, 1.0f, 0.0f, 1.0f, 0.0f, 0.0f, 0.0f, false, false
    );
}

# 定义一个函数 ggml_rope_inplace，接受上下文、两个张量、维度数量、模式和上下文数量作为参数，返回一个张量指针
struct ggml_tensor * ggml_rope_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                   n_dims,
        int                   mode,
        int                   n_ctx) {
    # 调用 ggml_rope_impl 函数，传入参数并返回结果
    return ggml_rope_impl(
        ctx, a, b, n_dims, mode, n_ctx, 0, 10000.0f, 1.0f, 0.0f, 1.0f, 0.0f, 0.0f, 0.0f, false, true
    );
}

# 定义一个函数 ggml_rope_custom，接受上下文、两个张量、维度数量、模式、上下文数量、原始上下文数量、频率基数、频率比例、扩展因子、注意力因子、快速 beta、慢速 beta 作为参数，返回一个张量指针
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
    # 调用 ggml_rope_impl 函数，传入参数并返回结果
    return ggml_rope_impl(
        ctx, a, b, n_dims, mode, n_ctx, n_orig_ctx, freq_base, freq_scale,
        ext_factor, attn_factor, beta_fast, beta_slow, 0.0f, false, false
    );
}
// 使用 ggml_rope_impl 函数实现自定义操作，返回处理后的张量
struct ggml_tensor * ggml_rope_custom_inplace(
        struct ggml_context * ctx,  // 上下文对象指针
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
    // 调用 ggml_rope_impl 函数，返回处理后的张量
    return ggml_rope_impl(
        ctx, a, b, n_dims, mode, n_ctx, n_orig_ctx, freq_base, freq_scale,
        ext_factor, attn_factor, beta_fast, beta_slow, 0.0f, false, true
    );
}

// 使用 ggml_rope_impl 函数实现 xpos 操作，返回处理后的张量
struct ggml_tensor * ggml_rope_xpos_inplace(
        struct ggml_context * ctx,  // 上下文对象指针
        struct ggml_tensor  * a,    // 输入张量 a
        struct ggml_tensor  * b,    // 输入张量 b
        int                   n_dims,  // 张量维度
        float                 base,     // 基数
        bool                  down) {   // 是否向下
    // 调用 ggml_rope_impl 函数，返回处理后的张量
    return ggml_rope_impl(ctx, a, b, n_dims, 0, 0, 0, 10000.0f, 1.0f, 0.0f, 1.0f, 0.0f, 0.0f, base, down, true);
}

// ggml_rope_back

// 使用 ggml_rope_impl 函数实现反向操作，返回处理后的张量
struct ggml_tensor * ggml_rope_back(
        struct ggml_context * ctx,  // 上下文对象指针
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
    // 断言 a 的第三维度等于 b 的第一维度
    GGML_ASSERT(a->ne[2] == b->ne[0]);

    // 断言 mode 的第三位是 0，如果不是则输出错误信息
    GGML_ASSERT((mode & 4) == 0 && "ggml_rope_back() for ChatGLM not implemented yet");

    // 初始化 is_node 为 false
    bool is_node = false;
}
    // 如果输入张量 a 有梯度信息，则将 is_node 设为 false，表示需要实现反向传播
    if (a->grad) {
        is_node = false; // TODO: implement backward
    }

    // 复制输入张量 a，得到一个新的张量 result
    struct ggml_tensor * result = ggml_dup_tensor(ctx, a);

    // 定义包含13个元素的参数数组，并初始化前5个元素
    int32_t params[13] = { /*n_past*/ 0, n_dims, mode, n_ctx, n_orig_ctx };
    // 将后续8个参数值拷贝到参数数组中
    memcpy(params +  5, &freq_base,    sizeof(float));
    memcpy(params +  6, &freq_scale,   sizeof(float));
    memcpy(params +  7, &ext_factor,   sizeof(float));
    memcpy(params +  8, &attn_factor,  sizeof(float));
    memcpy(params +  9, &beta_fast,    sizeof(float));
    memcpy(params + 10, &beta_slow,    sizeof(float));
    memcpy(params + 11, &xpos_base,    sizeof(float));
    memcpy(params + 12, &xpos_down,    sizeof(bool));
    // 将参数数组传递给结果张量的操作参数
    ggml_set_op_params(result, params, sizeof(params));

    // 设置结果张量的操作类型为 GGML_OP_ROPE_BACK
    result->op   = GGML_OP_ROPE_BACK;
    // 如果 is_node 为 true，则将结果张量的梯度设置为输入张量 a 的复制，否则设置为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的源张量为输入张量 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    // 返回结果张量
    return result;
}

// ggml_alibi

// 定义一个函数 ggml_alibi，接受一个上下文结构体指针，一个张量指针 a，一个整数 n_past，一个整数 n_head，一个浮点数 bias_max，返回一个指向 ggml_tensor 结构体的指针
struct ggml_tensor * ggml_alibi(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   n_past,
        int                   n_head,
        float                 bias_max) {
    GGML_ASSERT(n_past >= 0); // 断言 n_past 大于等于 0
    bool is_node = false; // 初始化布尔变量 is_node 为 false

    if (a->grad) { // 如果张量 a 的 grad 成员不为空
        GGML_ASSERT(false); // 断言为假，提示需要实现反向传播
        is_node = true; // 将 is_node 设置为 true
    }

    // TODO: when implement backward, fix this:
    //struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);
    struct ggml_tensor * result = ggml_view_tensor(ctx, a); // 创建一个指向 ggml_tensor 结构体的指针 result，指向 a 的视图

    int32_t op_params[3] = { n_past, n_head }; // 创建一个包含 n_past 和 n_head 的整型数组 op_params
    memcpy(op_params + 2, &bias_max, sizeof(float)); // 将 bias_max 的值复制到 op_params 数组的第三个位置
    ggml_set_op_params(result, op_params, sizeof(op_params)); // 设置 result 的操作参数为 op_params

    result->op   = GGML_OP_ALIBI; // 设置 result 的操作类型为 GGML_OP_ALIBI
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL; // 如果 is_node 为 true，则将 result 的 grad 成员设置为 result 的副本，否则设置为 NULL
    result->src[0] = a; // 设置 result 的源张量为 a

    return result; // 返回 result
}

// ggml_clamp

// 定义一个函数 ggml_clamp，接受一个上下文结构体指针，一个张量指针 a，一个最小值 min，一个最大值 max，返回一个指向 ggml_tensor 结构体的指针
struct ggml_tensor * ggml_clamp(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        float                 min,
        float                 max) {
    bool is_node = false; // 初始化布尔变量 is_node 为 false

    if (a->grad) { // 如果张量 a 的 grad 成员不为空
        GGML_ASSERT(false); // 断言为假，提示需要实现反向传播
        is_node = true; // 将 is_node 设置为 true
    }

    // TODO: when implement backward, fix this:
    struct ggml_tensor * result = ggml_view_tensor(ctx, a); // 创建一个指向 ggml_tensor 结构体的指针 result，指向 a 的视图

    float params[] = { min, max }; // 创建一个包含 min 和 max 的浮点数数组 params
    ggml_set_op_params(result, params, sizeof(params)); // 设置 result 的操作参数为 params

    result->op   = GGML_OP_CLAMP; // 设置 result 的操作类型为 GGML_OP_CLAMP
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL; // 如果 is_node 为 true，则将 result 的 grad 成员设置为 result 的副本，否则设置为 NULL
    result->src[0] = a; // 设置 result 的源张量为 a

    return result; // 返回 result
}

// ggml_conv_1d

// 定义一个静态函数 ggml_calc_conv_output_size，接受输入大小 ins，卷积核大小 ks，步长 s，填充 p，膨胀 d，返回卷积输出大小
static int64_t ggml_calc_conv_output_size(int64_t ins, int64_t ks, int s, int p, int d) {
    return (ins + 2 * p - d * (ks - 1) - 1) / s + 1; // 返回卷积输出大小的计算结果
}

// 定义一个函数 ggml_conv_1d，接受一个上下文结构体指针，一个输入张量指针 a，一个卷积核张量指针 b，一个步长 s0，一个填充 p0，一个膨胀 d0，返回一个指向 ggml_tensor 结构体的指针
GGML_API struct ggml_tensor * ggml_conv_1d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                   s0,
        int                   p0,
        int                   d0) {
    // 使用 ggml_im2col 函数将输入张量 a 转换为 im2col 张量，参数包括输入张量 a、卷积核张量 b、步长 s0、填充 p0、膨胀系数 d0，返回的张量形状为 [N, OL, IC * K]
    struct ggml_tensor * im2col = ggml_im2col(ctx, a, b, s0, 0, p0, 0, d0, 0, false);

    // 使用 ggml_mul_mat 函数进行矩阵乘法运算，参数包括上一步得到的 im2col 张量和经过 reshape 后的输入张量 a，返回的结果张量形状为 [N, OC, OL]
    struct ggml_tensor * result =
        ggml_mul_mat(ctx,
                ggml_reshape_2d(ctx, im2col, im2col->ne[0], (im2col->ne[2] * im2col->ne[1])), // [N, OL, IC * K] => [N*OL, IC * K]
                ggml_reshape_2d(ctx, a, (a->ne[0] * a->ne[1]), a->ne[2]));                    // [OC，IC, K] => [OC, IC * K]

    // 使用 ggml_reshape_3d 函数将上一步得到的结果张量 result 进行形状重塑，返回的结果张量形状为 [N, OC, OL]
    result = ggml_reshape_3d(ctx, result, im2col->ne[1], a->ne[2], im2col->ne[2]);

    // 返回最终结果张量
    return result;
}

// ggml_conv_1d_ph

// 对输入的一维张量进行一维卷积操作
struct ggml_tensor* ggml_conv_1d_ph(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                   s,
        int                   d) {
    // 调用 ggml_conv_1d 函数进行一维卷积操作
    return ggml_conv_1d(ctx, a, b, s, a->ne[0] / 2, d);
}

// ggml_conv_transpose_1d

// 计算一维卷积转置操作的输出大小
static int64_t ggml_calc_conv_transpose_1d_output_size(int64_t ins, int64_t ks, int s, int p, int d) {
    return (ins - 1) * s - 2 * p + d * (ks - 1) + 1;
}

// 对输入的一维张量进行一维卷积转置操作
GGML_API struct ggml_tensor * ggml_conv_transpose_1d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                   s0,
        int                   p0,
        int                   d0) {
    GGML_ASSERT(ggml_is_matrix(b)); // 断言输入张量 b 是矩阵
    GGML_ASSERT(a->ne[2] == b->ne[1]); // 断言输入张量 a 的第三维与输入张量 b 的第二维相等
    GGML_ASSERT(a->ne[3] == 1); // 断言输入张量 a 的第四维为1

    GGML_ASSERT(p0 == 0); // 断言输入的 p0 为0
    GGML_ASSERT(d0 == 1); // 断言输入的 d0 为1

    bool is_node = false;

    if (a->grad || b->grad) {
        GGML_ASSERT(false); // 断言梯度不存在，待实现反向传播
        is_node = true;
    }

    // 计算输出张量的维度
    const int64_t ne[4] = {
        ggml_calc_conv_transpose_1d_output_size(b->ne[0], a->ne[0], s0, 0 /*p0*/, 1 /*d0*/),
        a->ne[1], b->ne[2], 1,
    };
    // 创建新的输出张量
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F32, 4, ne);

    int32_t params[] = { s0, p0, d0 };
    ggml_set_op_params(result, params, sizeof(params));

    result->op = GGML_OP_CONV_TRANSPOSE_1D;
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;
    result->src[1] = b;

    return result;
}

// ggml_conv_2d

// im2col: [N, IC, IH, IW] => [N, OH, OW, IC*KH*KW]
// a: [OC，IC, KH, KW]
// b: [N, IC, IH, IW]
// result: [N, OH, OW, IC*KH*KW]
// 对输入的二维张量进行 im2col 操作
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
    # 根据输入的维度信息判断是否为二维数据
    bool                 is_2D) {

    # 如果是二维数据，检查两个输入的第三维是否相等
    if(is_2D) {
        GGML_ASSERT(a->ne[2] == b->ne[2]);
    } else {
        # 如果不是二维数据，检查两个输入的第二维是否相等
        GGML_ASSERT(a->ne[1] == b->ne[1]);
    }
    # 初始化节点标志为假
    bool is_node = false;

    # 如果输入中有梯度信息，则抛出断言错误，并将节点标志设置为真
    if (a->grad || b->grad) {
        GGML_ASSERT(false); // TODO: implement backward
        is_node = true;
    }

    # 计算输出的高度和宽度
    const int64_t OH = is_2D ? ggml_calc_conv_output_size(b->ne[1], a->ne[1], s1, p1, d1) : 0;
    const int64_t OW =         ggml_calc_conv_output_size(b->ne[0], a->ne[0], s0, p0, d0);

    # 根据是否为二维数据，计算输出的维度信息
    const int64_t ne[4] = {
        is_2D ? (a->ne[2] * a->ne[1] * a->ne[0]) : a->ne[1] * a->ne[0],
        OW,
        is_2D ? OH : b->ne[2],
        is_2D ?      b->ne[3] : 1,
    };

    # 创建一个新的张量对象，并设置其维度信息和操作参数
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F16, 4, ne);
    int32_t params[] = { s0, s1, p0, p1, d0, d1, (is_2D ? 1 : 0) };
    ggml_set_op_params(result, params, sizeof(params));

    # 设置张量对象的操作类型和梯度信息
    result->op = GGML_OP_IM2COL;
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;
    result->src[1] = b;

    # 返回创建的张量对象
    return result;
// 定义一个函数，实现二维卷积操作
// a: [OC，IC, KH, KW]，表示输入数据的形状
// b: [N, IC, IH, IW]，表示卷积核的形状
// result: [N, OC, OH, OW]，表示输出数据的形状
struct ggml_tensor * ggml_conv_2d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                  s0,  // 步长
        int                  s1,  // 步长
        int                  p0,  // 填充
        int                  p1,  // 填充
        int                  d0,  // 膨胀
        int                  d1) {  // 膨胀
    // 将输入数据转换为im2col格式
    struct ggml_tensor * im2col = ggml_im2col(ctx, a, b, s0, s1, p0, p1, d0, d1, true); // [N, OH, OW, IC * KH * KW]

    // 执行矩阵乘法操作
    struct ggml_tensor * result =
        ggml_mul_mat(ctx,
                ggml_reshape_2d(ctx, im2col, im2col->ne[0],  im2col->ne[3] * im2col->ne[2] * im2col->ne[1]), // [N, OH, OW, IC * KH * KW] => [N*OH*OW, IC * KH * KW]
                ggml_reshape_2d(ctx, a, (a->ne[0] * a->ne[1] * a->ne[2]),  a->ne[3]));                       // [OC，IC, KH, KW] => [OC, IC * KH * KW]

    // 将结果转换为指定形状
    result = ggml_reshape_4d(ctx, result, im2col->ne[1], im2col->ne[2], a->ne[3], im2col->ne[3]); // [N, OC, OH, OW]

    // 返回结果
    return result;
}

// 定义一个函数，实现带有默认填充的二维卷积操作
struct ggml_tensor * ggml_conv_2d_sk_p0(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    // 调用ggml_conv_2d函数，设置默认填充参数
    return ggml_conv_2d(ctx, a, b, a->ne[0], a->ne[1], 0, 0, 1, 1);
}

// 定义一个函数，实现带有默认步长和自动填充的二维卷积操作
struct ggml_tensor * ggml_conv_2d_s1_ph(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b) {
    // 调用ggml_conv_2d函数，设置默认步长和自动填充参数
    return ggml_conv_2d(ctx, a, b, 1, 1, a->ne[0] / 2, a->ne[1] / 2, 1, 1);
}

// 定义一个函数，实现带有默认填充的二维转置卷积操作
struct ggml_tensor * ggml_conv_transpose_2d_p0(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                   stride) {
    // 检查输入数据和卷积核的维度是否匹配
    GGML_ASSERT(a->ne[3] == b->ne[2]);

    // 计算转置卷积输出的大小
    bool is_node = false;
    int64_t output_size = ggml_calc_conv_transpose_output_size(a->ne[1], b->ne[2], stride, 0);

    // 返回结果
    return result;
}
    // 如果 a 或 b 的梯度存在，则断言为假，提示需要实现反向传播
    if (a->grad || b->grad) {
        GGML_ASSERT(false); // TODO: implement backward
        is_node = true;
    }

    // 计算转置卷积输出的大小，并存储在数组 ne 中
    const int64_t ne[4] = {
        ggml_calc_conv_transpose_output_size(b->ne[0], a->ne[0], stride, 0 /*p0*/),
        ggml_calc_conv_transpose_output_size(b->ne[1], a->ne[1], stride, 0 /*p1*/),
        a->ne[2], b->ne[3],
    };

    // 创建一个新的张量 result，类型为 GGML_TYPE_F32，维度为 4，大小为 ne
    struct ggml_tensor* result = ggml_new_tensor(ctx, GGML_TYPE_F32, 4, ne);

    // 设置张量 result 的操作参数为 stride
    ggml_set_op_params_i32(result, 0, stride);

    // 设置张量 result 的操作类型为 GGML_OP_CONV_TRANSPOSE_2D
    result->op = GGML_OP_CONV_TRANSPOSE_2D;
    // 如果 is_node 为真，则将 result 的梯度设置为 result 的副本，否则设置为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置张量 result 的源张量为 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    // 返回创建的张量 result
    return result;
// ggml_pool_*

// 计算池化层输出大小的函数
static int64_t ggml_calc_pool_output_size(int64_t ins, int ks, int s, float p) {
    return (ins + 2 * p - ks) / s + 1;
}

// ggml_pool_1d

// 对输入的一维张量进行池化操作
struct ggml_tensor * ggml_pool_1d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        enum ggml_op_pool     op,
        int                   k0,
        int                   s0,
        int                   p0) {

    bool is_node = false;

    // 如果输入张量有梯度，则抛出断言错误
    if (a->grad) {
        GGML_ASSERT(false); // TODO: implement backward
        is_node = true;
    }

    // 计算池化后的输出大小
    const int64_t ne[3] = {
        ggml_calc_pool_output_size(a->ne[0], k0, s0, p0),
        a->ne[1],
    };
    // 创建新的张量用于存储池化后的结果
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F32, 2, ne);

    // 设置池化操作的参数
    int32_t params[] = { op, k0, s0, p0 };
    ggml_set_op_params(result, params, sizeof(params));

    result->op = GGML_OP_POOL_1D;
    // 如果是节点，则复制张量作为梯度
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;

    return result;
}

// ggml_pool_2d

// 对输入的二维张量进行池化操作
struct ggml_tensor * ggml_pool_2d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        enum ggml_op_pool     op,
        int                   k0,
        int                   k1,
        int                   s0,
        int                   s1,
        float                 p0,
        float                 p1) {

    bool is_node = false;

    // 如果输入张量有梯度，则抛出断言错误
    if (a->grad) {
        GGML_ASSERT(false); // TODO: implement backward
        is_node = true;
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

    result->op = GGML_OP_POOL_2D;
    // 如果是节点，则复制张量作为梯度
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;

    return result;
}

// ggml_upscale
// 实现对输入张量进行上采样操作
static struct ggml_tensor * ggml_upscale_impl(
    struct ggml_context * ctx,
    struct ggml_tensor * a,
    int scale_factor) {
    bool is_node = false; // 初始化一个布尔变量 is_node，用于表示是否为节点

    if (a->grad) { // 如果输入张量存在梯度
        GGML_ASSERT(false); // 断言，提示需要实现反向传播
        is_node = true; // 将 is_node 设置为 true
    }

    // 创建一个新的张量 result，其维度为输入张量的维度乘以缩放因子
    struct ggml_tensor * result = ggml_new_tensor_4d(ctx, a->type,
            a->ne[0] * scale_factor,
            a->ne[1] * scale_factor,
            a->ne[2], a->ne[3]);

    result->op = GGML_OP_UPSCALE; // 设置 result 的操作类型为上采样
    result->op_params[0] = scale_factor; // 设置 result 的操作参数为缩放因子
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL; // 如果 is_node 为 true，则将 result 的梯度设置为 result 的副本，否则设置为 NULL
    result->src[0] = a; // 设置 result 的源张量为输入张量 a
    result->src[1] = NULL; // 设置 result 的第二个源张量为 NULL

    return result; // 返回结果张量
}

// 对外接口，实现对输入张量进行上采样操作
struct ggml_tensor * ggml_upscale(
    struct ggml_context * ctx,
    struct ggml_tensor * a,
    int scale_factor) {
    return ggml_upscale_impl(ctx, a, scale_factor); // 调用内部实现函数 ggml_upscale_impl
}

// 实现对输入张量进行闪存注意力操作
struct ggml_tensor * ggml_flash_attn(
        struct ggml_context * ctx,
        struct ggml_tensor  * q,
        struct ggml_tensor  * k,
        struct ggml_tensor  * v,
        bool                  masked) {
    GGML_ASSERT(ggml_can_mul_mat(k, q)); // 断言，提示是否可以对 k 和 q 进行矩阵乘法
    // TODO: check if vT can be multiplied by (k*qT) // 待实现：检查是否可以对 v 转置后与 (k*q) 进行矩阵乘法

    bool is_node = false; // 初始化一个布尔变量 is_node，用于表示是否为节点

    if (q->grad || k->grad || v->grad) { // 如果输入张量 q、k、v 中任意一个存在梯度
        is_node = true; // 将 is_node 设置为 true
    }

    // 创建一个新的张量 result，其类型为 GGML_TYPE_F32，维度和形状与输入张量 q 相同
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F32, q->n_dims, q->ne);

    int32_t t = masked ? 1 : 0; // 根据 masked 的值设置 t 的值为 1 或 0
    ggml_set_op_params(result, &t, sizeof(t)); // 设置 result 的操作参数为 t

    result->op   = GGML_OP_FLASH_ATTN; // 设置 result 的操作类型为闪存注意力
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL; // 如果 is_node 为 true，则将 result 的梯度设置为 result 的副本，否则设置为 NULL
    result->src[0] = q; // 设置 result 的第一个源张量为输入张量 q
    result->src[1] = k; // 设置 result 的第二个源张量为输入张量 k
    result->src[2] = v; // 设置 result 的第三个源张量为输入张量 v

    return result; // 返回结果张量
}

// 实现对输入张量进行闪存前馈操作
struct ggml_tensor * ggml_flash_ff(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b0,
        struct ggml_tensor  * b1,
        struct ggml_tensor  * c0,
        struct ggml_tensor  * c1) {
    GGML_ASSERT(ggml_can_mul_mat(b0, a)); // 断言，提示是否可以对 b0 和 a 进行矩阵乘法
    // TODO: more checks  // 添加更多的检查

    // 检查是否存在梯度，如果存在则设置 is_node 为 true
    bool is_node = false;

    if (a->grad || b0->grad || b1->grad || c0->grad || c1->grad) {
        is_node = true;
    }

    // 创建一个新的张量 result，与张量 a 具有相同的维度和元素个数
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F32, a->n_dims, a->ne);

    // 设置 result 的操作类型为 GGML_OP_FLASH_FF
    result->op   = GGML_OP_FLASH_FF;
    // 如果 is_node 为 true，则为 result 创建一个梯度张量，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置 result 的源张量
    result->src[0] = a;
    result->src[1] = b0;
    result->src[2] = b1;
    result->src[3] = c0;
    result->src[4] = c1;

    // 返回创建的结果张量
    return result;
    // 定义一个名为 ggml_flash_attn_back 的函数，接受多个参数并返回一个指向 ggml_tensor 结构体的指针
    struct ggml_tensor * ggml_flash_attn_back(
        struct ggml_context * ctx,
        struct ggml_tensor  * q,
        struct ggml_tensor  * k,
        struct ggml_tensor  * v,
        struct ggml_tensor  * d,
        bool                  masked) {
    // 断言 k 和 q 可以进行矩阵乘法运算
    GGML_ASSERT(ggml_can_mul_mat(k, q));
    // TODO: 检查是否可以将 vT 与 (k*qT) 相乘

    // 定义变量并初始化
    const int64_t     D = q->ne[0];
    const int64_t     N = q->ne[1];
    const int64_t     M = k->ne[1];
    const int64_t   ne2 = q->ne[2];
    const int64_t   ne3 = q->ne[3];
    const int64_t kvne2 = k->ne[2];

    // 断言确保各个张量的形状符合要求
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

    // 断言确保 ne2 能被 kvne2 整除
    GGML_ASSERT(ne2 % kvne2 == 0);

    // 初始化一个布尔变量 is_node
    bool is_node = false;

    // 如果 q、k 或 v 具有梯度，则将 is_node 设置为 false
    if (q->grad || k->grad || v->grad) {
        // 在反向传播中使用此操作时，这些梯度会被设置
        // 我们不希望创建我们的结果的（大）梯度，因此 is_node 设置为 false
        is_node = false;
    }

    // 将 q、k 和 v 的梯度存储为连续张量，并连接到结果中
    // 注意：v 和 gradv 实际上是转置的，即 v->ne[0] != D
    const int64_t elem_q = ggml_nelements(q);
    const int64_t elem_k = ggml_nelements(k);
    const int64_t elem_v = ggml_nelements(v);

    // 定义结果类型为 GGML_TYPE_F32
    enum ggml_type result_type = GGML_TYPE_F32;
    GGML_ASSERT(ggml_blck_size(result_type) == 1);
    const size_t tsize = ggml_type_size(result_type);

    // 定义偏移量 offs_q 和 offs_k
    const size_t offs_q = 0;
    const size_t offs_k = offs_q + GGML_PAD(elem_q * tsize, GGML_MEM_ALIGN);
    # 计算偏移量 offs_v，为元素 k 分配内存并按照内存对齐方式进行偏移
    const size_t offs_v = offs_k + GGML_PAD(elem_k * tsize, GGML_MEM_ALIGN);
    # 计算结束位置 end，为元素 v 分配内存并按照内存对齐方式进行偏移
    const size_t end    = offs_v + GGML_PAD(elem_v * tsize, GGML_MEM_ALIGN);

    # 计算元素个数 nelements，向上取整到 tsize 的倍数
    const size_t nelements = (end + tsize - 1)/tsize;

    # 创建一个一维的浮点型张量 result
    struct ggml_tensor * result = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, nelements);

    # 根据 masked 的值设置 masked_i 的值为 1 或 0
    int32_t masked_i = masked ? 1 : 0;
    # 设置 result 的操作参数为 masked_i
    ggml_set_op_params(result, &masked_i, sizeof(masked_i));

    # 设置 result 的操作为 GGML_OP_FLASH_ATTN_BACK
    result->op   = GGML_OP_FLASH_ATTN_BACK;
    # 如果是节点，则为 result 的梯度分配内存，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    # 设置 result 的源数据为 q, k, v, d
    result->src[0] = q;
    result->src[1] = k;
    result->src[2] = v;
    result->src[3] = d;

    # 返回创建的张量 result
    return result;
// ggml_win_part

// 定义一个函数，用于对输入的张量进行窗口划分操作
struct ggml_tensor * ggml_win_part(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   w) {
    // 断言输入张量的第三维度大小为1
    GGML_ASSERT(a->ne[3] == 1);
    // 断言输入张量的数据类型为 GGML_TYPE_F32
    GGML_ASSERT(a->type  == GGML_TYPE_F32);

    // 初始化一个布尔变量 is_node，并赋值为 false
    bool is_node = false;

    // 如果输入张量存在梯度
    if (a->grad) {
        // 断言为假，提示需要实现反向传播
        GGML_ASSERT(false); // TODO: implement backward
        // 将 is_node 设置为 true
        is_node = true;
    }

    // 计算填充值
    const int px = (w - a->ne[1]%w)%w;
    const int py = (w - a->ne[2]%w)%w;

    // 计算新的宽度和高度
    const int npx = (px + a->ne[1])/w;
    const int npy = (py + a->ne[2])/w;
    const int np  = npx*npy;

    // 创建一个包含新形状的张量 result
    const int64_t ne[4] = { a->ne[0], w, w, np, };
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F32, 4, ne);

    // 设置操作参数
    int32_t params[] = { npx, npy, w };
    ggml_set_op_params(result, params, sizeof(params));

    // 设置张量的操作类型为 GGML_OP_WIN_PART
    result->op   = GGML_OP_WIN_PART;
    // 如果 is_node 为 true，则将 result 的梯度设置为 result 的副本，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置 result 的源张量为输入张量 a
    result->src[0] = a;

    // 返回 result
    return result;
}

// ggml_win_unpart

// 定义一个函数，用于对输入的张量进行窗口合并操作
struct ggml_tensor * ggml_win_unpart(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   w0,
        int                   h0,
        int                   w) {
    // 断言输入张量的数据类型为 GGML_TYPE_F32
    GGML_ASSERT(a->type == GGML_TYPE_F32);

    // 初始化一个布尔变量 is_node，并赋值为 false
    bool is_node = false;

    // 如果输入张量存在梯度
    if (a->grad) {
        // 断言为假，提示需要实现反向传播
        GGML_ASSERT(false); // TODO: implement backward
        // 将 is_node 设置为 true
        is_node = true;
    }

    // 创建一个包含新形状的张量 result
    const int64_t ne[4] = { a->ne[0], w0, h0, 1, };
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F32, 3, ne);

    // 设置操作参数
    int32_t params[] = { w };
    ggml_set_op_params(result, params, sizeof(params));

    // 设置张量的操作类型为 GGML_OP_WIN_UNPART
    result->op   = GGML_OP_WIN_UNPART;
    // 如果 is_node 为 true，则将 result 的梯度设置为 result 的副本，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置 result 的源张量为输入张量 a
    result->src[0] = a;

    // 返回 result
    return result;
}

// ggml_get_rel_pos

// 定义一个函数，用于获取相对位置信息
struct ggml_tensor * ggml_get_rel_pos(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   qh,
        int                   kh) {
    // 断言 qh 等于 kh
    GGML_ASSERT(qh == kh);
    // 断言 2*MAX(qh, kh) - 1 等于输入张量的第二维度大小
    GGML_ASSERT(2*MAX(qh, kh) - 1 == a->ne[1]);
    # 定义一个布尔变量 is_node，并初始化为 false
    bool is_node = false;

    # 如果 a 的梯度存在
    if (a->grad) {
        # 断言，提示需要实现反向传播
        GGML_ASSERT(false); // TODO: implement backward
        # 将 is_node 设置为 true
        is_node = true;
    }

    # 定义一个包含四个元素的整型数组 ne，并初始化为 a->ne[0]、kh、qh、1
    const int64_t ne[4] = { a->ne[0], kh, qh, 1, };
    # 创建一个新的张量 result，类型为 GGML_TYPE_F16，维度为 3，形状为 ne
    struct ggml_tensor * result = ggml_new_tensor(ctx, GGML_TYPE_F16, 3, ne);

    # 设置 result 的操作类型为 GGML_OP_GET_REL_POS
    result->op   = GGML_OP_GET_REL_POS;
    # 如果 is_node 为 true，则将 result 的梯度设置为 result 的副本，否则设置为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    # 设置 result 的源张量为 a，第二个源张量为 NULL
    result->src[0] = a;
    result->src[1] = NULL;

    # 返回 result
    return result;
// ggml_add_rel_pos

// 实现相对位置加法操作
static struct ggml_tensor * ggml_add_rel_pos_impl(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * pw,
        struct ggml_tensor  * ph,
        bool                  inplace) {
    // 断言pw和ph具有相同的形状
    GGML_ASSERT(ggml_are_same_shape(pw, ph));
    // 断言a是连续的
    GGML_ASSERT(ggml_is_contiguous(a));
    // 断言pw是连续的
    GGML_ASSERT(ggml_is_contiguous(pw));
    // 断言ph是连续的
    GGML_ASSERT(ggml_is_contiguous(ph));
    // 断言ph的类型为GGML_TYPE_F32
    GGML_ASSERT(ph->type == GGML_TYPE_F32);
    // 断言pw的类型为GGML_TYPE_F32
    GGML_ASSERT(pw->type == GGML_TYPE_F32);
    // 断言pw的第四个维度的大小等于a的第三个维度的大小
    GGML_ASSERT(pw->ne[3] == a->ne[2]);
    // 断言pw的第一维和第二维的乘积等于a的第一维的大小
    GGML_ASSERT(pw->ne[0]*pw->ne[0] == a->ne[0]);
    // 断言pw的第二维和第三维的乘积等于a的第二维的大小
    GGML_ASSERT(pw->ne[1]*pw->ne[2] == a->ne[1]);

    // 初始化is_node为false
    bool is_node = false;

    // 如果不是原地操作且a、pw、ph中有任意一个具有梯度，则将is_node设置为true
    if (!inplace && (a->grad || pw->grad || ph->grad)) {
        is_node = true;
    }

    // 如果是原地操作，则创建a的视图，否则创建a的副本
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);
    // 设置操作参数为0或1，取决于是否是原地操作
    ggml_set_op_params_i32(result, 0, inplace ? 1 : 0);

    // 设置结果的操作类型为GGML_OP_ADD_REL_POS
    result->op   = GGML_OP_ADD_REL_POS;
    // 如果is_node为true，则创建result的梯度，否则为NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果的源张量为a、pw、ph
    result->src[0] = a;
    result->src[1] = pw;
    result->src[2] = ph;

    // 返回结果
    return result;
}

// 创建相对位置加法操作的接口函数
struct ggml_tensor * ggml_add_rel_pos(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * pw,
        struct ggml_tensor  * ph) {
    return ggml_add_rel_pos_impl(ctx, a, pw, ph, false);
}

// 创建原地相对位置加法操作的接口函数
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
    // 初始化is_node为false
    bool is_node = false;

    // 如果不是原地操作且a具有梯度，则将is_node设置为true
    if (!inplace && (a->grad)) {
        is_node = true;
    }

    // 如果是原地操作，则创建a的视图，否则创建a的副本
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);
    # 设置 result 结构体中的第一个参数为整型的 op
    ggml_set_op_params_i32(result, 0, (int32_t) op);

    # 设置 result 结构体中的操作类型为一元操作
    result->op   = GGML_OP_UNARY;
    # 如果是节点，则设置 result 结构体中的梯度为 result 的副本，否则设置为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    # 设置 result 结构体中的第一个源操作数为 a
    result->src[0] = a;

    # 返回 result 结构体
    return result;
}

# 定义一个函数，对输入的张量进行一元操作，并返回结果张量
struct ggml_tensor * ggml_unary(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        enum ggml_unary_op op) {
    return ggml_unary_impl(ctx, a, op, false);
}

# 定义一个函数，对输入的张量进行原地一元操作，并返回结果张量
struct ggml_tensor * ggml_unary_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        enum ggml_unary_op op) {
    return ggml_unary_impl(ctx, a, op, true);
}

# 定义一个静态函数，对输入的张量进行一元操作，并返回结果张量
static struct ggml_tensor * ggml_map_unary_impl_f32(
        struct ggml_context        * ctx,
        struct ggml_tensor         * a,
        const  ggml_unary_op_f32_t fun,
        bool   inplace) {
    bool is_node = false;

    # 如果不是原地操作且输入张量有梯度，则设置 is_node 为 true
    if (!inplace && a->grad) {
        is_node = true;
    }

    # 根据是否原地操作，复制或者创建输入张量的视图
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    # 设置结果张量的操作参数
    ggml_set_op_params(result, (const void *) &fun, sizeof(fun));

    # 设置结果张量的操作类型
    result->op = GGML_OP_MAP_UNARY;
    # 如果是节点，则复制结果张量作为梯度，否则梯度为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;

    return result;
}

# 定义一个函数，对输入的张量进行一元操作，并返回结果张量
struct ggml_tensor * ggml_map_unary_f32(
        struct ggml_context        * ctx,
        struct ggml_tensor         * a,
        const  ggml_unary_op_f32_t fun) {
    return ggml_map_unary_impl_f32(ctx, a, fun, false);
}

# 定义一个函数，对输入的张量进行原地一元操作，并返回结果张量
struct ggml_tensor * ggml_map_unary_inplace_f32(
        struct ggml_context        * ctx,
        struct ggml_tensor         * a,
        const  ggml_unary_op_f32_t fun) {
    return ggml_map_unary_impl_f32(ctx, a, fun, true);
}

# 定义一个静态函数，对输入的两个张量进行二元操作，并返回结果张量
static struct ggml_tensor * ggml_map_binary_impl_f32(
        struct ggml_context         * ctx,
        struct ggml_tensor          * a,
        struct ggml_tensor          * b,
        const  ggml_binary_op_f32_t fun,
        bool   inplace) {
    # 断言两个输入张量的形状相同
    GGML_ASSERT(ggml_are_same_shape(a, b));

    bool is_node = false;

    # 如果不是原地操作且输入张量有梯度，则设置 is_node 为 true
    if (!inplace && (a->grad || b->grad)) {
        is_node = true;
    }

    # 根据是否原地操作，复制或者创建输入张量的视图
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);
    # 设置操作参数，将函数指针和其大小传递给 result 结构体
    ggml_set_op_params(result, (const void *) &fun, sizeof(fun));

    # 设置 result 结构体的操作类型为 GGML_OP_MAP_BINARY
    result->op = GGML_OP_MAP_BINARY;
    # 如果是节点，则将 result 结构体的梯度设置为 result 自身的副本，否则设置为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    # 设置 result 结构体的第一个源操作数为 a
    result->src[0] = a;
    # 设置 result 结构体的第二个源操作数为 b
    result->src[1] = b;

    # 返回 result 结构体
    return result;
// 定义函数 ggml_map_binary_f32，接受两个输入张量和一个二元操作函数指针，返回一个新的张量
struct ggml_tensor * ggml_map_binary_f32(
        struct ggml_context         * ctx,
        struct ggml_tensor          * a,
        struct ggml_tensor          * b,
        const  ggml_binary_op_f32_t fun) {
    // 调用 ggml_map_binary_impl_f32 函数，传入参数 a, b, fun 和 false，返回结果
    return ggml_map_binary_impl_f32(ctx, a, b, fun, false);
}

// 定义函数 ggml_map_binary_inplace_f32，接受两个输入张量和一个二元操作函数指针，返回一个新的张量
struct ggml_tensor * ggml_map_binary_inplace_f32(
        struct ggml_context         * ctx,
        struct ggml_tensor          * a,
        struct ggml_tensor          * b,
        const  ggml_binary_op_f32_t fun) {
    // 调用 ggml_map_binary_impl_f32 函数，传入参数 a, b, fun 和 true，返回结果
    return ggml_map_binary_impl_f32(ctx, a, b, fun, true);
}

// 定义静态函数 ggml_map_custom1_impl_f32，接受一个输入张量和一个自定义操作函数指针，返回一个新的张量
static struct ggml_tensor * ggml_map_custom1_impl_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        const  ggml_custom1_op_f32_t   fun,
        bool   inplace) {
    // 初始化变量 is_node 为 false
    bool is_node = false;

    // 如果不是原地操作且输入张量有梯度，则将 is_node 设置为 true
    if (!inplace && a->grad) {
        is_node = true;
    }

    // 根据 inplace 的值选择复制输入张量或者创建其视图，并将结果保存在 result 中
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置结果张量的操作参数为自定义操作函数指针
    ggml_set_op_params(result, (const void *) &fun, sizeof(fun));

    // 设置结果张量的操作类型为 GGML_OP_MAP_CUSTOM1_F32
    result->op = GGML_OP_MAP_CUSTOM1_F32;
    // 如果 is_node 为 true，则为结果张量创建一个梯度张量，否则为其设置为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置结果张量的第一个源张量为输入张量 a
    result->src[0] = a;

    // 返回结果张量
    return result;
}

// 定义函数 ggml_map_custom1_f32，接受一个输入张量和一个自定义操作函数指针，返回一个新的张量
struct ggml_tensor * ggml_map_custom1_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        const  ggml_custom1_op_f32_t   fun) {
    // 调用 ggml_map_custom1_impl_f32 函数，传入参数 a, fun 和 false，返回结果
    return ggml_map_custom1_impl_f32(ctx, a, fun, false);
}

// 定义函数 ggml_map_custom1_inplace_f32，接受一个输入张量和一个自定义操作函数指针，返回一个新的张量
struct ggml_tensor * ggml_map_custom1_inplace_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        const  ggml_custom1_op_f32_t   fun) {
    // 调用 ggml_map_custom1_impl_f32 函数，传入参数 a, fun 和 true，返回结果
    return ggml_map_custom1_impl_f32(ctx, a, fun, true);
}

// 定义静态函数 ggml_map_custom2_impl_f32，接受两个输入张量和一个自定义操作函数指针，返回一个新的张量
static struct ggml_tensor * ggml_map_custom2_impl_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
        const  ggml_custom2_op_f32_t   fun,
        bool   inplace) {
    // 初始化变量 is_node 为 false
    bool is_node = false;
    # 如果不是原地操作且其中一个操作数有梯度，则将 is_node 设为 true
    if (!inplace && (a->grad || b->grad)) {
        is_node = true;
    }

    # 如果是原地操作，则创建一个指向 a 的视图张量，否则创建 a 的副本张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    # 设置操作参数为 fun，并指定参数大小
    ggml_set_op_params(result, (const void *) &fun, sizeof(fun));

    # 将操作设置为 GGML_OP_MAP_CUSTOM2_F32
    result->op = GGML_OP_MAP_CUSTOM2_F32;
    # 如果 is_node 为 true，则为结果张量创建一个梯度张量，否则设为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    # 设置结果张量的源操作数为 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    # 返回结果张量
    return result;
// 定义一个函数，将两个张量和自定义的二元操作函数映射为一个新的张量
struct ggml_tensor * ggml_map_custom2_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
        const  ggml_custom2_op_f32_t   fun) {
    return ggml_map_custom2_impl_f32(ctx, a, b, fun, false);
}

// 定义一个函数，将两个张量和自定义的二元操作函数映射为一个新的张量（原地操作）
struct ggml_tensor * ggml_map_custom2_inplace_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
        const  ggml_custom2_op_f32_t   fun) {
    return ggml_map_custom2_impl_f32(ctx, a, b, fun, true);
}

// 静态函数，实现将三个张量和自定义的三元操作函数映射为一个新的张量
static struct ggml_tensor * ggml_map_custom3_impl_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
        struct ggml_tensor           * c,
        const  ggml_custom3_op_f32_t   fun,
        bool   inplace) {
    bool is_node = false;

    // 如果不是原地操作，并且任意一个张量具有梯度，则设置 is_node 为 true
    if (!inplace && (a->grad || b->grad || c->grad)) {
        is_node = true;
    }

    // 如果是原地操作，则创建一个 a 的视图张量，否则创建 a 的副本张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置操作参数
    ggml_set_op_params(result, (const void *) &fun, sizeof(fun));

    // 设置操作类型为 GGML_OP_MAP_CUSTOM3_F32
    result->op = GGML_OP_MAP_CUSTOM3_F32;
    // 如果是节点，则创建 result 的梯度张量，否则设置为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;
    result->src[1] = b;
    result->src[2] = c;

    return result;
}

// 定义一个函数，将三个张量和自定义的三元操作函数映射为一个新的张量
struct ggml_tensor * ggml_map_custom3_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
        struct ggml_tensor           * c,
        const  ggml_custom3_op_f32_t   fun) {
    return ggml_map_custom3_impl_f32(ctx, a, b, c, fun, false);
}

// 定义一个函数，将三个张量和自定义的三元操作函数映射为一个新的张量（原地操作）
struct ggml_tensor * ggml_map_custom3_inplace_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
        struct ggml_tensor           * c,
        const  ggml_custom3_op_f32_t   fun) {
    # 调用 ggml_map_custom3_impl_f32 函数，传入参数 ctx, a, b, c, fun, true，并返回结果
    return ggml_map_custom3_impl_f32(ctx, a, b, c, fun, true);
}

// 定义结构体 ggml_map_custom1_op_params，包含自定义操作函数、任务数量和用户数据
struct ggml_map_custom1_op_params {
    ggml_custom1_op_t fun;
    int n_tasks;
    void * userdata;
};

// 定义静态函数 ggml_map_custom1_impl，实现自定义操作
static struct ggml_tensor * ggml_map_custom1_impl(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        const  ggml_custom1_op_t       fun,
        int                            n_tasks,
        void                         * userdata,
        bool                           inplace) {
    GGML_ASSERT(n_tasks == GGML_N_TASKS_MAX || n_tasks > 0);

    bool is_node = false;

    // 如果不是原地操作且输入张量有梯度，则设置 is_node 为 true
    if (!inplace && a->grad) {
        is_node = true;
    }

    // 根据 inplace 参数选择创建新张量或者视图张量
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    // 设置自定义操作的参数
    struct ggml_map_custom1_op_params params = {
        /*.fun      =*/ fun,
        /*.n_tasks  =*/ n_tasks,
        /*.userdata =*/ userdata
    };
    ggml_set_op_params(result, (const void *) &params, sizeof(params));

    // 设置结果张量的操作类型和梯度
    result->op = GGML_OP_MAP_CUSTOM1;
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    result->src[0] = a;

    return result;
}

// 定义函数 ggml_map_custom1，调用 ggml_map_custom1_impl 实现自定义操作
struct ggml_tensor * ggml_map_custom1(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        const  ggml_custom1_op_t       fun,
        int                            n_tasks,
        void                         * userdata) {
    return ggml_map_custom1_impl(ctx, a, fun, n_tasks, userdata, false);
}

// 定义函数 ggml_map_custom1_inplace，调用 ggml_map_custom1_impl 实现原地自定义操作
struct ggml_tensor * ggml_map_custom1_inplace(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        const  ggml_custom1_op_t       fun,
        int                            n_tasks,
        void                         * userdata) {
    return ggml_map_custom1_impl(ctx, a, fun, n_tasks, userdata, true);
}

// 定义结构体 ggml_map_custom2_op_params，包含自定义操作函数、任务数量和用户数据
struct ggml_map_custom2_op_params {
    ggml_custom2_op_t fun;
    int n_tasks;
    void * userdata;
};
# 定义一个静态函数，实现自定义的二元操作
static struct ggml_tensor * ggml_map_custom2_impl(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
        const  ggml_custom2_op_t       fun,
        int                            n_tasks,
        void                         * userdata,
        bool                           inplace) {
    # 断言任务数为最大任务数或者大于0
    GGML_ASSERT(n_tasks == GGML_N_TASKS_MAX || n_tasks > 0);

    # 初始化一个布尔变量 is_node
    bool is_node = false;

    # 如果不是原地操作且 a 或 b 其中一个具有梯度
    if (!inplace && (a->grad || b->grad)) {
        # 将 is_node 设置为 true
        is_node = true;
    }

    # 根据 inplace 的值选择创建新的结果张量或者在原张量上进行操作
    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);

    # 初始化自定义二元操作的参数
    struct ggml_map_custom2_op_params params = {
        /*.fun      =*/ fun,
        /*.n_tasks  =*/ n_tasks,
        /*.userdata =*/ userdata
    };
    # 将参数设置到结果张量上
    ggml_set_op_params(result, (const void *) &params, sizeof(params));

    # 设置结果张量的操作类型为自定义二元操作
    result->op = GGML_OP_MAP_CUSTOM2;
    # 如果 is_node 为 true，则为结果张量设置梯度张量，否则为 null
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    # 设置结果张量的源张量为 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    # 返回结果张量
    return result;
}

# 定义一个函数，调用 ggml_map_custom2_impl 函数，实现自定义的二元操作
struct ggml_tensor * ggml_map_custom2(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
        const  ggml_custom2_op_t       fun,
        int                            n_tasks,
        void                         * userdata) {
    return ggml_map_custom2_impl(ctx, a, b, fun, n_tasks, userdata, false);
}

# 定义一个函数，调用 ggml_map_custom2_impl 函数，实现原地的自定义的二元操作
struct ggml_tensor * ggml_map_custom2_inplace(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
        const  ggml_custom2_op_t       fun,
        int                            n_tasks,
        void                         * userdata) {
    return ggml_map_custom2_impl(ctx, a, b, fun, n_tasks, userdata, true);
}

# 定义自定义的三元操作的参数结构体
struct ggml_map_custom3_op_params {
    ggml_custom3_op_t fun;
    int n_tasks;
    void * userdata;
}
# 定义一个名为 ggml_map_custom3_impl 的函数，接受多个参数并返回一个指向 ggml_tensor 结构体的指针
static struct ggml_tensor * ggml_map_custom3_impl(
        struct ggml_context          * ctx,  # 参数：指向 ggml_context 结构体的指针，表示上下文
        struct ggml_tensor           * a,    # 参数：指向 ggml_tensor 结构体的指针，表示第一个张量
        struct ggml_tensor           * b,    # 参数：指向 ggml_tensor 结构体的指针，表示第二个张量
        struct ggml_tensor           * c,    # 参数：指向 ggml_tensor 结构体的指针，表示第三个张量
        const  ggml_custom3_op_t       fun,  # 参数：指向 ggml_custom3_op_t 类型的函数指针，表示自定义操作
        int                            n_tasks,  # 参数：整数类型，表示任务数量
        void                         * userdata,  # 参数：指向 void 类型的指针，表示用户数据
        bool                           inplace) {  # 参数：布尔类型，表示是否原地操作
    GGML_ASSERT(n_tasks == GGML_N_TASKS_MAX || n_tasks > 0);  # 断言：任务数量必须等于最大任务数量或大于0

    bool is_node = false;  # 声明并初始化布尔变量 is_node

    if (!inplace && (a->grad || b->grad || c->grad)) {  # 如果不是原地操作且任意一个张量具有梯度
        is_node = true;  # 将 is_node 设置为 true
    }

    struct ggml_tensor * result = inplace ? ggml_view_tensor(ctx, a) : ggml_dup_tensor(ctx, a);  # 根据 inplace 的值选择创建新张量或者创建视图张量

    struct ggml_map_custom3_op_params params = {  # 声明并初始化 ggml_map_custom3_op_params 结构体
        /*.fun      =*/ fun,  # 设置结构体成员 fun 的值为传入的 fun 参数
        /*.n_tasks  =*/ n_tasks,  # 设置结构体成员 n_tasks 的值为传入的 n_tasks 参数
        /*.userdata =*/ userdata  # 设置结构体成员 userdata 的值为传入的 userdata 参数
    };
    ggml_set_op_params(result, (const void *) &params, sizeof(params));  # 调用 ggml_set_op_params 函数设置操作参数

    result->op = GGML_OP_MAP_CUSTOM3;  # 设置 result 指向的 ggml_tensor 结构体的 op 成员为 GGML_OP_MAP_CUSTOM3
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;  # 根据 is_node 的值选择创建新张量或者设置为 NULL
    result->src[0] = a;  # 设置 result 指向的 ggml_tensor 结构体的 src 数组的第一个元素为 a
    result->src[1] = b;  # 设置 result 指向的 ggml_tensor 结构体的 src 数组的第二个元素为 b
    result->src[2] = c;  # 设置 result 指向的 ggml_tensor 结构体的 src 数组的第三个元素为 c

    return result;  # 返回 result 指针
}

# 定义一个名为 ggml_map_custom3 的函数，接受多个参数并返回一个指向 ggml_tensor 结构体的指针
struct ggml_tensor * ggml_map_custom3(
        struct ggml_context          * ctx,  # 参数：指向 ggml_context 结构体的指针，表示上下文
        struct ggml_tensor           * a,    # 参数：指向 ggml_tensor 结构体的指针，表示第一个张量
        struct ggml_tensor           * b,    # 参数：指向 ggml_tensor 结构体的指针，表示第二个张量
        struct ggml_tensor           * c,    # 参数：指向 ggml_tensor 结构体的指针，表示第三个张量
        const  ggml_custom3_op_t       fun,  # 参数：指向 ggml_custom3_op_t 类型的函数指针，表示自定义操作
        int                            n_tasks,  # 参数：整数类型，表示任务数量
        void                         * userdata) {  # 参数：指向 void 类型的指针，表示用户数据
    return ggml_map_custom3_impl(ctx, a, b, c, fun, n_tasks, userdata, false);  # 调用 ggml_map_custom3_impl 函数，传入参数并返回结果
}

# 定义一个名为 ggml_map_custom3_inplace 的函数，接受多个参数并返回一个指向 ggml_tensor 结构体的指针
struct ggml_tensor * ggml_map_custom3_inplace(
        struct ggml_context          * ctx,  # 参数：指向 ggml_context 结构体的指针，表示上下文
        struct ggml_tensor           * a,    # 参数：指向 ggml_tensor 结构体的指针，表示第一个张量
        struct ggml_tensor           * b,    # 参数：指向 ggml_tensor 结构体的指针，表示第二个张量
        struct ggml_tensor           * c,    # 参数：指向 ggml_tensor 结构体的指针，表示第三个张量
        const  ggml_custom3_op_t       fun,  # 参数：指向 ggml_custom3_op_t 类型的函数指针，表示自定义操作
        int                            n_tasks,  # 参数：整数类型，表示任务数量
        void                         * userdata) {  # 参数：指向 void 类型的指针，表示用户数据
    return ggml_map_custom3_impl(ctx, a, b, c, fun, n_tasks, userdata, true);  # 调用 ggml_map_custom3_impl 函数，传入参数并返回结果
}
// 计算交叉熵损失
struct ggml_tensor * ggml_cross_entropy_loss(
        struct ggml_context         * ctx,
        struct ggml_tensor          * a,
        struct ggml_tensor          * b) {
    // 断言输入张量 a 和 b 的形状相同
    GGML_ASSERT(ggml_are_same_shape(a, b));
    // 初始化 is_node 为 false
    bool is_node = false;

    // 如果输入张量 a 或 b 具有梯度，则将 is_node 设置为 true
    if (a->grad || b->grad) {
        is_node = true;
    }

    // 创建一个新的一维张量 result，用于存储结果
    struct ggml_tensor * result = ggml_new_tensor_1d(ctx, a->type, 1);

    // 设置 result 的操作类型为交叉熵损失
    result->op   = GGML_OP_CROSS_ENTROPY_LOSS;
    // 如果 is_node 为 true，则为 result 分配一个梯度张量，否则为 NULL
    result->grad = is_node ? ggml_dup_tensor(ctx, result) : NULL;
    // 设置 result 的源张量为 a 和 b
    result->src[0] = a;
    result->src[1] = b;

    // 返回结果张量
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
    // 断言输入张量 c 是标量
    GGML_ASSERT(ggml_is_scalar(c));

    // 复制输入张量 a，作为结果张量
    struct ggml_tensor * result = ggml_dup_tensor(ctx, a);

    // 设置结果张量的操作类型为交叉熵损失的反向传播
    result->op   = GGML_OP_CROSS_ENTROPY_LOSS_BACK;
    // 将结果张量的梯度设置为 NULL
    result->grad = NULL;
    // 设置结果张量的源张量为 a、b 和 c
    result->src[0] = a;
    result->src[1] = b;
    result->src[2] = c;

    // 返回结果张量
    return result;
}

////////////////////////////////////////////////////////////////////////////////

// 设置张量为参数
void ggml_set_param(
        struct ggml_context * ctx,
        struct ggml_tensor * tensor) {
    // 将张量标记为参数
    tensor->is_param = true;

    // 断言张量的梯度为 NULL，然后为其分配一个梯度张量
    GGML_ASSERT(tensor->grad == NULL);
    tensor->grad = ggml_dup_tensor(ctx, tensor);
    // 为梯度张量设置名称
    ggml_format_name(tensor->grad, "%s (grad)", tensor->name);
}

// 执行前向计算，处理连续内存块相同的情况
static void ggml_compute_forward_dup_same_cont(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言目标张量和源张量的元素数量相同
    GGML_ASSERT(ggml_nelements(dst) == ggml_nelements(src0));
    // 断言目标张量和源张量都是连续内存块，并且数据类型相同
    GGML_ASSERT(ggml_is_contiguous(dst) && ggml_is_contiguous(src0));
    GGML_ASSERT(src0->type == dst->type);

    // 如果任务类型为初始化或结束，直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }
}
    // 获取源数据的第一个维度大小
    const size_t nb00 = src0->nb[0];
    // 获取目标数据的第一个维度大小
    const size_t nb0 = dst->nb[0];

    // 获取线程索引
    const int ith = params->ith; // thread index
    // 获取线程总数
    const int nth = params->nth; // number of threads

    // 并行化处理每个元素
    const int ne = ggml_nelements(dst);
    // 计算每个线程处理的元素数量
    const int dr = (ne + nth - 1) / nth;
    // 计算当前线程处理的元素范围
    const int ie0 = dr * ith;
    const int ie1 = MIN(ie0 + dr, ne);

    // 如果当前线程有处理的元素
    if (ie0 < ie1) {
        // 将源数据的部分内容复制到目标数据中
        memcpy(
            ((char *)  dst->data + ie0*nb0),
            ((char *) src0->data + ie0*nb00),
            (ie1 - ie0) * ggml_type_size(src0->type));
    }
// 计算前向传播的重复操作，将源张量的数据复制到目标张量中
static void ggml_compute_forward_dup_f16(
        const struct ggml_compute_params * params,  // 传入的计算参数
        const struct ggml_tensor * src0,  // 源张量
        struct ggml_tensor * dst) {  // 目标张量
    GGML_ASSERT(ggml_nelements(dst) == ggml_nelements(src0));  // 断言目标张量和源张量的元素数量相等

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  // 如果参数类型为初始化或结束，则直接返回
        return;
    }

    GGML_TENSOR_UNARY_OP_LOCALS  // 定义张量一元操作的本地变量

    const int ith = params->ith; // 线程索引
    const int nth = params->nth; // 线程数量

    if (ggml_is_contiguous(src0) && ggml_is_contiguous(dst) && src0->type == dst->type) {  // 如果源张量和目标张量都是连续存储，并且类型相同
        ggml_compute_forward_dup_same_cont(params, src0, dst);  // 调用相同连续存储的数据复制函数
        return;
    }

    // 按行并行化
    const int nr = ne01;  // 行数
    // 每个线程的行数
    const int dr = (nr + nth - 1) / nth;
    // 该线程的行范围
    const int ir0 = dr * ith;
    const int ir1 = MIN(ir0 + dr, nr);

    if (src0->type == dst->type &&
        ne00 == ne0 &&
        nb00 == ggml_type_size(src0->type) && nb0 == ggml_type_size(dst->type)) {
        // 按行复制
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

    // TODO: add more special-case implementations for tensor shapes/strides that can benefit from memcpy
    // TODO: 为可以从memcpy中受益的张量形状/步幅添加更多特殊情况的实现

    }

    // 目标张量计数器
    int64_t i10 = 0;
    int64_t i11 = 0;
    int64_t i12 = 0;
    int64_t i13 = 0;

    } else {
        GGML_ASSERT(false); // TODO: implement
    }
}

static void ggml_compute_forward_dup_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    # 确保目标张量和源张量的元素数量相等
    GGML_ASSERT(ggml_nelements(dst) == ggml_nelements(src0));

    # 如果任务类型是初始化或者结束，直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    # 定义一些本地变量
    GGML_TENSOR_UNARY_OP_LOCALS

    # 获取线程索引和线程数量
    const int ith = params->ith; // 线程索引
    const int nth = params->nth; // 线程数量

    # 如果源张量和目标张量都是连续的，并且类型相同，直接调用函数进行前向计算并返回
    if (ggml_is_contiguous(src0) && ggml_is_contiguous(dst) && src0->type == dst->type) {
        ggml_compute_forward_dup_same_cont(params, src0, dst);
        return;
    }

    # 并行化处理每一行
    const int nr = ne01;  # 行数
    const int dr = (nr + nth - 1) / nth;  # 每个线程处理的行数
    const int ir0 = dr * ith;  # 当前线程处理的起始行
    const int ir1 = MIN(ir0 + dr, nr);  # 当前线程处理的结束行

    # 如果源张量和目标张量的类型相同，并且一些其他条件满足，使用循环进行数据拷贝
    if (src0->type == dst->type &&
        ne00 == ne0 &&
        nb00 == ggml_type_size(src0->type) && nb0 == ggml_type_size(dst->type)) {
        # 按行进行数据拷贝
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

    # 如果不满足上述条件，暂时不处理，抛出断言错误
    } else {
        GGML_ASSERT(false); // TODO: implement
    }
// 计算前向传播的重复操作，处理非连续内存情况
static void ggml_compute_forward_dup(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 检查源张量和目标张量是否都是连续内存，并且类型相同
    if (ggml_is_contiguous(src0) && ggml_is_contiguous(dst) && src0->type == dst->type) {
        // 如果满足条件，调用相同内存布局的重复操作函数
        ggml_compute_forward_dup_same_cont(params, src0, dst);
        return;
    }
    // 如果不满足条件，根据源张量的类型进行不同的处理
    switch (src0->type) {
        // 如果源张量类型为 GGML_TYPE_F16
        case GGML_TYPE_F16:
            {
                // 调用处理 GGML_TYPE_F16 类型的重复操作函数
                ggml_compute_forward_dup_f16(params, src0, dst);
            } break;
        // 如果源张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用处理 GGML_TYPE_F32 类型的重复操作函数
                ggml_compute_forward_dup_f32(params, src0, dst);
            } break;
        // 如果源张量类型为其他类型
        default:
            {
                // 抛出断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_add

// 处理 GGML_TYPE_F32 类型的前向传播加法操作
static void ggml_compute_forward_add_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 断言：src1 可以重复行，src0 和 dst 的形状相同
    GGML_ASSERT(ggml_can_repeat_rows(src1, src0) && ggml_are_same_shape(src0, dst));

    // 如果任务类型为初始化或者结束，直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取参数中的线程索引和线程总数
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取 src0 的行数
    const int nr  = ggml_nrows(src0);

    // 定义一些局部变量

    // 断言：nb0 和 nb00 的大小为 float 类型
    GGML_ASSERT( nb0 == sizeof(float));
    GGML_ASSERT(nb00 == sizeof(float));

    // 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);
    // 获取 dst 的第三个源张量
    struct ggml_tensor *src2 = dst->src[2];
    float *ft;
    // 如果 src2 存在
    if (src2 != NULL) 
        // 获取 src2 的数据
        ft = src2->data;
    // 如果 nb10 等于 float 的大小
    if (nb10 == sizeof(float)) {
        // 遍历 ir0 到 ir1 之间的值
        for (int ir = ir0; ir < ir1; ++ir) {
            // 在 i1、i2、i3 上，src1 可以被广播到 src0 和 dst
            const int64_t i03 = ir/(ne02*ne01);
            const int64_t i02 = (ir - i03*ne02*ne01)/ne01;
            const int64_t i01 = (ir - i03*ne02*ne01 - i02*ne01);

            const int64_t i13 = i03 % ne13;
            const int64_t i12 = i02 % ne12;
            const int64_t i11 = i01 % ne11;

            // 计算目标指针的位置
            float * dst_ptr  = (float *) ((char *) dst->data  + i03*nb3  + i02*nb2  + i01*nb1 );
            // 计算 src0 指针的位置
            float * src0_ptr = (float *) ((char *) src0->data + i03*nb03 + i02*nb02 + i01*nb01);
            // 计算 src1 指针的位置
            float * src1_ptr = (float *) ((char *) src1->data + i13*nb13 + i12*nb12 + i11*nb11);
#ifdef GGML_USE_ACCELERATE
            // 如果定义了 GGML_USE_ACCELERATE，则使用 vDSP_vadd 函数进行向量加法运算
            vDSP_vadd(src0_ptr, 1, src1_ptr, 1, dst_ptr, 1, ne00);
#else
            // 如果未定义 GGML_USE_ACCELERATE，则使用 ggml_vec_add_f32 函数进行向量加法运算
            // 如果 src2 为空，则直接调用 ggml_vec_add_f32 函数
            if (src2 == NULL)
                ggml_vec_add_f32(ne00, dst_ptr, src0_ptr, src1_ptr);
            else
            {
                // 如果 src2 不为空，则根据条件选择不同的处理方式
                int num = src2->ne[0];
                // 如果 num 大于 1000，则使用条件判断进行向量加法运算
                if (num > 1000) {
                    for (int i = 0; i < ne00; i++)
                    {
                        dst_ptr[i] = ft[i] >= 0.0f ? src0_ptr[i] + src1_ptr[i] : 0;
                    }
                }
                else {
                    // 如果 num 不大于 1000，则根据条件选择不同的处理方式
                    for (int i = 0; i < num; i++)
                    {
                        int id = i << 7;
                        // 根据条件判断进行向量加法运算
                        if (ft[i] < -7.0f){
                            for (int j = 0; j < 128; j++)
                                dst_ptr[id + j] = 0;
                            continue;
                        }
                        else
                        {
                            for (int j = 0; j < 128; j++)
                                dst_ptr[id+j] = src0_ptr[id+j] + src1_ptr[id+j];
                        }
                    }
                }
            }
#endif
        }
    } else {
        // 如果 src1 不是连续的
        for (int ir = ir0; ir < ir1; ++ir) {
            // 在 i1、i2、i3 上，src1 可以被广播到 src0 和 dst
            const int64_t i03 = ir/(ne02*ne01);
            const int64_t i02 = (ir - i03*ne02*ne01)/ne01;
            const int64_t i01 = (ir - i03*ne02*ne01 - i02*ne01);

            const int64_t i13 = i03 % ne13;
            const int64_t i12 = i02 % ne12;
            const int64_t i11 = i01 % ne11;

            float * dst_ptr  = (float *) ((char *) dst->data  + i03*nb3  + i02*nb2  + i01*nb1 );
            float * src0_ptr = (float *) ((char *) src0->data + i03*nb03 + i02*nb02 + i01*nb01);

            for (int i0 = 0; i0 < ne0; i0++) {
                float * src1_ptr = (float *) ((char *) src1->data + i13*nb13 + i12*nb12 + i11*nb11 + i0*nb10);

                dst_ptr[i0] = src0_ptr[i0] + *src1_ptr;
            }
        }
    }
# 计算两个张量的加法，结果存储在目标张量中
static void ggml_compute_forward_add_f16_f32(
        const struct ggml_compute_params * params,  # 传入计算参数的指针
        const struct ggml_tensor * src0,  # 第一个输入张量
        const struct ggml_tensor * src1,  # 第二个输入张量
        struct ggml_tensor * dst) {  # 输出结果张量
    GGML_ASSERT(ggml_are_same_shape(src0, src1) && ggml_are_same_shape(src0, dst));  # 断言输入张量和输出张量的形状相同

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  # 如果任务类型是初始化或者结束，则直接返回
        return;
    }

    const int ith = params->ith;  # 当前线程的索引
    const int nth = params->nth;  # 线程总数

    const int nr  = ggml_nrows(src0);  # 获取输入张量的行数

    GGML_TENSOR_BINARY_OP_LOCALS  # 定义张量二元操作的本地变量

    GGML_ASSERT(src0->type == GGML_TYPE_F16);  # 断言第一个输入张量的数据类型为F16
    GGML_ASSERT(src1->type == GGML_TYPE_F32);  # 断言第二个输入张量的数据类型为F32

    if (dst->type == GGML_TYPE_F32) {  # 如果输出张量的数据类型为F32
        GGML_ASSERT( nb0 == sizeof(float));  # 断言nb0的大小为float类型的大小
    }
    else {  # 如果输出张量的数据类型不是F32
        GGML_ASSERT(dst->type  == GGML_TYPE_F16);  # 断言输出张量的数据类型为F16
        GGML_ASSERT( nb0 == sizeof(ggml_fp16_t));  # 断言nb0的大小为ggml_fp16_t类型的大小
    }

    GGML_ASSERT(nb00 == sizeof(ggml_fp16_t));  # 断言nb00的大小为ggml_fp16_t类型的大小

    // rows per thread  # 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // row range for this thread  # 当前线程处理的行范围
    const int ir0 = dr*ith;  # 当前线程处理的起始行索引
    const int ir1 = MIN(ir0 + dr, nr);  # 当前线程处理的结束行索引，取ir0+dr和nr中的较小值
    # 如果 nb10 等于 float 的大小
    if (nb10 == sizeof(float)) {
        # 如果目标数据类型是 GGML_TYPE_F16
        if (dst->type == GGML_TYPE_F16) {
            # 遍历指定范围内的索引
            for (int ir = ir0; ir < ir1; ++ir) {
                # 计算三维索引
                const int i3 = ir/(ne2*ne1);
                const int i2 = (ir - i3*ne2*ne1)/ne1;
                const int i1 = (ir - i3*ne2*ne1 - i2*ne1);

                # 计算目标、源数据的指针位置
                ggml_fp16_t * dst_ptr  = (ggml_fp16_t *) ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1);
                ggml_fp16_t * src0_ptr = (ggml_fp16_t *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01);
                float *       src1_ptr = (float *)       ((char *) src1->data + i3*nb13 + i2*nb12 + i1*nb11);

                # 对每个元素进行操作
                for (int i = 0; i < ne0; i++) {
                    dst_ptr[i] = GGML_FP32_TO_FP16(GGML_FP16_TO_FP32(src0_ptr[i]) + src1_ptr[i]);
                }
            }
        } else {
            # 遍历指定范围内的索引
            for (int ir = ir0; ir < ir1; ++ir) {
                # 计算三维索引
                const int i3 = ir/(ne2*ne1);
                const int i2 = (ir - i3*ne2*ne1)/ne1;
                const int i1 = (ir - i3*ne2*ne1 - i2*ne1);

                # 计算目标、源数据的指针位置
                float *       dst_ptr  = (float *)       ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1);
                ggml_fp16_t * src0_ptr = (ggml_fp16_t *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01);
                float *       src1_ptr = (float *)       ((char *) src1->data + i3*nb13 + i2*nb12 + i1*nb11);

                # 对每个元素进行操作
                for (int i = 0; i < ne0; i++) {
                    dst_ptr[i] = GGML_FP16_TO_FP32(src0_ptr[i]) + src1_ptr[i];
                }
            }
        }
    }
    else {
        # 如果 nb10 不等于 float 的大小，抛出异常
        GGML_ASSERT(false);
    }
// 计算两个输入张量的加法，结果存储在目标张量中
static void ggml_compute_forward_add_f16_f16(
        const struct ggml_compute_params * params,  // 传入的计算参数
        const struct ggml_tensor * src0,  // 第一个输入张量
        const struct ggml_tensor * src1,  // 第二个输入张量
        struct ggml_tensor * dst) {  // 存储结果的目标张量
    GGML_ASSERT(ggml_are_same_shape(src0, src1) && ggml_are_same_shape(src0, dst));  // 断言输入张量和目标张量的形状相同

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  // 如果参数的类型是初始化或者结束，则直接返回
        return;
    }

    const int ith = params->ith;  // 获取当前线程的索引
    const int nth = params->nth;  // 获取线程总数

    const int nr  = ggml_nrows(src0);  // 获取输入张量的行数

    GGML_TENSOR_BINARY_OP_LOCALS  // 定义张量二元操作的本地变量

    GGML_ASSERT(src0->type == GGML_TYPE_F16);  // 断言第一个输入张量的数据类型为F16
    GGML_ASSERT(src1->type == GGML_TYPE_F16);  // 断言第二个输入张量的数据类型为F16
    GGML_ASSERT(dst->type  == GGML_TYPE_F16);  // 断言目标张量的数据类型为F16

    GGML_ASSERT( nb0 == sizeof(ggml_fp16_t));  // 断言nb0的大小等于ggml_fp16_t的大小
    GGML_ASSERT(nb00 == sizeof(ggml_fp16_t));  // 断言nb00的大小等于ggml_fp16_t的大小

    // 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    if (nb10 == sizeof(ggml_fp16_t)) {  // 如果nb10的大小等于ggml_fp16_t的大小
        for (int ir = ir0; ir < ir1; ++ir) {  // 遍历当前线程处理的行
            // src0, src1 and dst are same shape => same indices
            const int i3 = ir/(ne2*ne1);  // 计算第三维的索引
            const int i2 = (ir - i3*ne2*ne1)/ne1;  // 计算第二维的索引
            const int i1 = (ir - i3*ne2*ne1 - i2*ne1);  // 计算第一维的索引

            ggml_fp16_t * dst_ptr  = (ggml_fp16_t *) ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1);  // 计算目标张量的指针位置
            ggml_fp16_t * src0_ptr = (ggml_fp16_t *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01);  // 计算第一个输入张量的指针位置
            ggml_fp16_t * src1_ptr = (ggml_fp16_t *) ((char *) src1->data + i3*nb13 + i2*nb12 + i1*nb11);  // 计算第二个输入张量的指针位置

            for (int i = 0; i < ne0; i++) {  // 遍历当前位置的元素
                dst_ptr[i] = GGML_FP32_TO_FP16(GGML_FP16_TO_FP32(src0_ptr[i]) + GGML_FP16_TO_FP32(src1_ptr[i]));  // 计算加法结果并存储在目标张量中
            }
        }
    }
    else {
        // src1 is not contiguous
        GGML_ASSERT(false);  // 断言src1不是连续的
    }
}
# 计算两个输入张量的加法，并将结果存储到目标张量中
static void ggml_compute_forward_add_q_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    # 确保输入张量和目标张量具有相同的形状
    GGML_ASSERT(ggml_are_same_shape(src0, src1) && ggml_are_same_shape(src0, dst));

    # 如果任务类型是初始化或者结束，直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    # 获取输入张量的行数
    const int nr  = ggml_nrows(src0);

    # 定义一些局部变量
    GGML_TENSOR_BINARY_OP_LOCALS

    # 获取当前线程的索引和总线程数
    const int ith = params->ith;
    const int nth = params->nth;

    # 获取输入张量和目标张量的数据类型
    const enum ggml_type type = src0->type;
    const enum ggml_type dtype = dst->type;
    ggml_to_float_t const dequantize_row_q = type_traits[type].to_float;
    ggml_from_float_t const quantize_row_q = type_traits[dtype].from_float;

    # 确保不支持置换的输入张量
    GGML_ASSERT(nb00 == ggml_type_size(type));
    GGML_ASSERT(nb10 == sizeof(float));

    # 确保目标张量不能被转置或置换
    GGML_ASSERT(nb0 <= nb1);
    GGML_ASSERT(nb1 <= nb2);
    GGML_ASSERT(nb2 <= nb3);

    # 确保输入张量是量化的，而且第二个输入张量是单精度浮点型
    GGML_ASSERT(ggml_is_quantized(src0->type));
    GGML_ASSERT(src1->type == GGML_TYPE_F32);

    # 计算每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    # 计算当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    # 将参数中的 wdata 转换为 float 指针，并根据当前线程的索引偏移
    float * wdata = (float *) params->wdata + (ne00 + CACHE_LINE_SIZE_F32) * ith;
}
    for (int ir = ir0; ir < ir1; ++ir) {
        // 遍历 ir0 到 ir1 之间的索引
        // src0 indices
        // 计算 src0 的三维索引
        const int i03 = ir/(ne02*ne01);
        const int i02 = (ir - i03*ne02*ne01)/ne01;
        const int i01 = (ir - i03*ne02*ne01 - i02*ne01);

        // src1 and dst are same shape as src0 => same indices
        // 计算 src1 和 dst 的三维索引，与 src0 相同
        const int i13 = i03;
        const int i12 = i02;
        const int i11 = i01;

        const int i3 = i03;
        const int i2 = i02;
        const int i1 = i01;

        // 计算 src0、src1 和 dst 对应行的内存地址
        void  * src0_row = (void *) ((char *) src0->data + (i01*nb01 + i02*nb02 + i03*nb03));
        float * src1_row = (float *)((char *) src1->data + (i11*nb11 + i12*nb12 + i13*nb13));
        void  * dst_row  = (void *) ((char *)  dst->data + ( i1*nb1  +  i2*nb2  +  i3*nb3));

        // 断言 ne00 能被 32 整除
        assert(ne00 % 32 == 0);

        // 将 src0 行解量化到临时缓冲区
        dequantize_row_q(src0_row, wdata, ne00);
        // 将 src1 加到临时缓冲区
        ggml_vec_acc_f32(ne00, wdata, src1_row);
        // 将行量化到 dst
        if (quantize_row_q != NULL) {
            quantize_row_q(wdata, dst_row, ne00);
        } else {
            // 如果没有量化函数，则直接复制数据到 dst
            memcpy(dst_row, wdata, ne0*nb0);
        }
    }
// 计算两个张量的加法操作
static void ggml_compute_forward_add(
        const struct ggml_compute_params * params,  // 接收计算参数的结构体指针
        const struct ggml_tensor * src0,  // 接收第一个张量的结构体指针
        const struct ggml_tensor * src1,  // 接收第二个张量的结构体指针
        struct ggml_tensor * dst) {  // 接收结果张量的结构体指针
    switch (src0->type) {  // 根据第一个张量的类型进行判断
        case GGML_TYPE_F32:  // 如果第一个张量的类型是 F32
            {
                ggml_compute_forward_add_f32(params, src0, src1, dst);  // 调用 F32 类型的加法计算函数
            } break;
        case GGML_TYPE_F16:  // 如果第一个张量的类型是 F16
            {
                if (src1->type == GGML_TYPE_F16) {  // 如果第二个张量的类型也是 F16
                    ggml_compute_forward_add_f16_f16(params, src0, src1, dst);  // 调用 F16 类型的加法计算函数
                }
                else if (src1->type == GGML_TYPE_F32) {  // 如果第二个张量的类型是 F32
                    ggml_compute_forward_add_f16_f32(params, src0, src1, dst);  // 调用 F16 和 F32 类型的加法计算函数
                }
                else {
                    GGML_ASSERT(false);  // 抛出断言错误
                }
            } break;
        case GGML_TYPE_Q4_0:  // 如果第一个张量的类型是 Q4_0
        case GGML_TYPE_Q4_1:  // 如果第一个张量的类型是 Q4_1
        case GGML_TYPE_Q5_0:  // 如果第一个张量的类型是 Q5_0
        case GGML_TYPE_Q5_1:  // 如果第一个张量的类型是 Q5_1
        case GGML_TYPE_Q8_0:  // 如果第一个张量的类型是 Q8_0
        case GGML_TYPE_Q2_K:  // 如果第一个张量的类型是 Q2_K
        case GGML_TYPE_Q3_K:  // 如果第一个张量的类型是 Q3_K
        case GGML_TYPE_Q4_K:  // 如果第一个张量的类型是 Q4_K
        case GGML_TYPE_Q5_K:  // 如果第一个张量的类型是 Q5_K
        case GGML_TYPE_Q6_K:  // 如果第一个张量的类型是 Q6_K
            {
                ggml_compute_forward_add_q_f32(params, src0, src1, dst);  // 调用 Q 类型和 F32 类型的加法计算函数
            } break;
        default:  // 如果第一个张量的类型不在以上列举的类型中
            {
                GGML_ASSERT(false);  // 抛出断言错误
            } break;
    }
}

// ggml_compute_forward_add1

static void ggml_compute_forward_add1_f32(
        const struct ggml_compute_params * params,  // 接收计算参数的结构体指针
        const struct ggml_tensor * src0,  // 接收第一个张量的结构体指针
        const struct ggml_tensor * src1,  // 接收第二个张量的结构体指针
        struct ggml_tensor * dst) {  // 接收结果张量的结构体指针
    GGML_ASSERT(ggml_are_same_shape(src0, dst));  // 断言第一个张量和结果张量的形状相同
    GGML_ASSERT(ggml_is_scalar(src1));  // 断言第二个张量是标量

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  // 如果计算参数的类型是初始化或者结束
        return;  // 直接返回
    }

    const int ith = params->ith;  // 获取计算参数中的 ith 值
    const int nth = params->nth;  // 获取计算参数中的 nth 值

    const int nr  = ggml_nrows(src0);  // 获取第一个张量的行数

    GGML_TENSOR_UNARY_OP_LOCALS  // 定义张量一元操作的本地变量

    GGML_ASSERT( nb0 == sizeof(float));  // 断言 nb0 的大小等于 float 类型的大小
    GGML_ASSERT(nb00 == sizeof(float));  // 断言 nb00 的大小等于 float 类型的大小

    // rows per thread
    // 计算每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 为当前线程确定行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 遍历行范围内的每一行
    for (int ir = ir0; ir < ir1; ++ir) {
        // 计算三维数组中的索引
        const int i3 = ir/(ne2*ne1);
        const int i2 = (ir - i3*ne2*ne1)/ne1;
        const int i1 = (ir - i3*ne2*ne1 - i2*ne1);
#ifdef GGML_USE_ACCELERATE
        // 如果定义了 GGML_USE_ACCELERATE，则使用加速库
        UNUSED(ggml_vec_add1_f32);

        // 使用 vDSP_vadd 函数进行向量加法运算
        vDSP_vadd(
                (float *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01), 1,  // 第一个向量的起始地址和步长
                (float *) ((char *) src1->data), 0,  // 第二个向量的起始地址和步长
                (float *) ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1 ), 1,  // 结果向量的起始地址和步长
                ne0);  // 向量的长度
#else
        // 如果未定义 GGML_USE_ACCELERATE，则使用自定义的向量加法函数
        ggml_vec_add1_f32(ne0,
                (float *) ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1 ),  // 结果向量的起始地址
                (float *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01),  // 第一个向量的起始地址
               *(float *) src1->data);  // 第二个向量的值
#endif
    }

static void ggml_compute_forward_add1_f16_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    GGML_ASSERT(ggml_are_same_shape(src0, dst));  // 断言 src0 和 dst 的形状相同
    GGML_ASSERT(ggml_is_scalar(src1));  // 断言 src1 是标量

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;  // 如果任务类型是初始化或结束，则直接返回
    }

    // 获取标量值
    const float v = *(float *) src1->data;

    const int ith = params->ith;  // 获取当前线程的索引
    const int nth = params->nth;  // 获取线程总数

    const int nr  = ggml_nrows(src0);  // 获取 src0 的行数

    GGML_TENSOR_UNARY_OP_LOCALS  // 定义一些局部变量

    GGML_ASSERT(src0->type == GGML_TYPE_F16);  // 断言 src0 的数据类型为 GGML_TYPE_F16
    GGML_ASSERT(src1->type == GGML_TYPE_F32);  // 断言 src1 的数据类型为 GGML_TYPE_F32
    GGML_ASSERT(dst->type  == GGML_TYPE_F16);  // 断言 dst 的数据类型为 GGML_TYPE_F16

    GGML_ASSERT( nb0 == sizeof(ggml_fp16_t));  // 断言 nb0 的大小为 ggml_fp16_t 的大小
    GGML_ASSERT(nb00 == sizeof(ggml_fp16_t));  // 断言 nb00 的大小为 ggml_fp16_t 的大小

    // 计算每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 计算当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);
    // 遍历 ir0 到 ir1 之间的整数
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
            // 将 src0_ptr[i] 转换为 float32，加上 v 后再转换为 float16，赋值给 dst_ptr[i]
            dst_ptr[i] = GGML_FP32_TO_FP16(GGML_FP16_TO_FP32(src0_ptr[i]) + v);
        }
    }
}
// 定义一个静态函数，用于计算两个 f16 类型的张量相加的前向传播
static void ggml_compute_forward_add1_f16_f16(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 断言 src0 和 dst 张量具有相同的形状
    GGML_ASSERT(ggml_are_same_shape(src0, dst));
    // 断言 src1 张量是标量
    GGML_ASSERT(ggml_is_scalar(src1));

    // 如果任务类型是初始化或者结束，直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取标量值
    const float v = GGML_FP16_TO_FP32(*(ggml_fp16_t *) src1->data);

    // 获取参数中的索引和数量
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取 src0 张量的行数
    const int nr  = ggml_nrows(src0);

    // 定义一些局部变量
    GGML_TENSOR_UNARY_OP_LOCALS

    // 断言 src0、src1 和 dst 张量的类型都是 f16
    GGML_ASSERT(src0->type == GGML_TYPE_F16);
    GGML_ASSERT(src1->type == GGML_TYPE_F16);
    GGML_ASSERT(dst->type  == GGML_TYPE_F16);

    // 断言 nb0 和 nb00 的大小
    GGML_ASSERT( nb0 == sizeof(ggml_fp16_t));
    GGML_ASSERT(nb00 == sizeof(ggml_fp16_t));

    // 计算每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 计算当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 遍历行范围内的每一行
    for (int ir = ir0; ir < ir1; ++ir) {
        // 计算索引
        const int i3 = ir/(ne2*ne1);
        const int i2 = (ir - i3*ne2*ne1)/ne1;
        const int i1 = (ir - i3*ne2*ne1 - i2*ne1);

        // 获取 dst 和 src0 的指针
        ggml_fp16_t * dst_ptr  = (ggml_fp16_t *) ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1 );
        ggml_fp16_t * src0_ptr = (ggml_fp16_t *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01);
        // 对每个元素进行相加操作
        for (int i = 0; i < ne0; i++) {
            dst_ptr[i] = GGML_FP32_TO_FP16(GGML_FP16_TO_FP32(src0_ptr[i]) + v);
        }
    }
}

// 定义一个静态函数，用于计算一个 f32 类型的张量和一个标量的相加操作
static void ggml_compute_forward_add1_q_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 断言 src0 和 dst 张量具有相同的形状
    GGML_ASSERT(ggml_are_same_shape(src0, dst));
    // 断言 src1 张量是标量
    GGML_ASSERT(ggml_is_scalar(src1));
    // 如果参数类型为初始化或者结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 定义一个浮点数变量v，用于存储src1数据的值
    const float v = *(float *) src1->data;

    // 获取参数中的ith和nth值
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取src0的行数
    const int nr  = ggml_nrows(src0);

    // 定义一些本地变量
    GGML_TENSOR_UNARY_OP_LOCALS

    // 获取src0的数据类型
    const enum ggml_type type = src0->type;
    // 获取src0数据类型对应的转换函数
    ggml_to_float_t const dequantize_row_q = type_traits[type].to_float;
    ggml_from_float_t const quantize_row_q = type_traits[type].from_float;

    // 检查nb00是否等于src0数据类型的大小
    GGML_ASSERT(nb00 == ggml_type_size(type));

    // 检查dst是否可以被转置或者排列
    GGML_ASSERT(nb0 <= nb1);
    GGML_ASSERT(nb1 <= nb2);
    GGML_ASSERT(nb2 <= nb3);

    // 检查src0和dst是否为量化类型
    GGML_ASSERT(ggml_is_quantized(src0->type));
    GGML_ASSERT(dst->type == src0->type);
    GGML_ASSERT(src1->type == GGML_TYPE_F32);

    // 计算每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 计算当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 定义一个float类型的指针wdata，指向params->wdata的内存位置
    float * wdata = (float *) params->wdata + (ne0 + CACHE_LINE_SIZE_F32) * ith;

    // 遍历每一行数据
    for (int ir = ir0; ir < ir1; ++ir) {
        // 计算当前行数据在src0和dst中的索引
        const int i3 = ir/(ne2*ne1);
        const int i2 = (ir - i3*ne2*ne1)/ne1;
        const int i1 = (ir - i3*ne2*ne1 - i2*ne1);

        // 获取src0和dst当前行的内存地址
        void  * src0_row = (void *) ((char *) src0->data + (i1*nb01 + i2*nb02 + i3*nb03));
        void  * dst_row  = (void *) ((char *)  dst->data + (i1*nb1  + i2*nb2  + i3*nb0 ));

        // 检查ne0是否为32的倍数
        assert(ne0 % 32 == 0);

        // 将src0行数据从量化转换为浮点数存储到临时缓冲区wdata中
        dequantize_row_q(src0_row, wdata, ne0);
        // 将src1的值加到wdata中
        ggml_vec_acc1_f32(ne0, wdata, v);
        // 将wdata中的数据从浮点数转换为量化数据存储到dst中
        quantize_row_q(wdata, dst_row, ne0);
    }
// 计算前向加法操作，根据输入参数和张量类型进行计算
static void ggml_compute_forward_add1(
        const struct ggml_compute_params * params,  // 输入参数结构体指针
        const struct ggml_tensor * src0,  // 第一个输入张量指针
        const struct ggml_tensor * src1,  // 第二个输入张量指针
        struct ggml_tensor * dst) {  // 输出张量指针
    // 根据第一个输入张量的类型进行不同的操作
    switch (src0->type) {
        case GGML_TYPE_F32:  // 如果类型为 GGML_TYPE_F32
            {
                ggml_compute_forward_add1_f32(params, src0, src1, dst);  // 调用 GGML_TYPE_F32 类型的前向加法操作函数
            } break;
        case GGML_TYPE_F16:  // 如果类型为 GGML_TYPE_F16
            {
                // 根据第二个输入张量的类型进行不同的操作
                if (src1->type == GGML_TYPE_F16) {
                    ggml_compute_forward_add1_f16_f16(params, src0, src1, dst);  // 调用 GGML_TYPE_F16 类型和 GGML_TYPE_F16 类型的前向加法操作函数
                }
                else if (src1->type == GGML_TYPE_F32) {
                    ggml_compute_forward_add1_f16_f32(params, src0, src1, dst);  // 调用 GGML_TYPE_F16 类型和 GGML_TYPE_F32 类型的前向加法操作函数
                }
                else {
                    GGML_ASSERT(false);  // 断言错误
                }
            } break;
        // 其他类型的操作
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
            {
                ggml_compute_forward_add1_q_f32(params, src0, src1, dst);  // 调用其他类型的前向加法操作函数
            } break;
        default:
            {
                GGML_ASSERT(false);  // 断言错误
            } break;
    }
}

// ggml_compute_forward_acc

// 计算前向累加操作，针对 GGML_TYPE_F32 类型的输入张量进行计算
static void ggml_compute_forward_acc_f32(
        const struct ggml_compute_params * params,  // 输入参数结构体指针
        const struct ggml_tensor * src0,  // 第一个输入张量指针
        const struct ggml_tensor * src1,  // 第二个输入张量指针
        struct ggml_tensor * dst) {  // 输出张量指针
    GGML_ASSERT(ggml_are_same_shape(src0, dst));  // 断言，确保输入张量和输出张量形状相同
    GGML_ASSERT(ggml_is_contiguous(dst) && ggml_is_contiguous(src0));  // 断言，确保输入张量和输出张量是连续的

    // 在累加过程中使用这些步长和数据偏移量来查看 src0 和 dst
    // nb0 隐式地是 element_size，因为 src0 和 dst 是连续的
    size_t nb1     = ((int32_t *) dst->op_params)[0];  // 获取 dst 的操作参数中的第一个值
    size_t nb2     = ((int32_t *) dst->op_params)[1];  // 获取 dst 的操作参数中的第二个值
    # 从目标操作数的参数中获取第三个整数值，作为 nb3
    size_t nb3     = ((int32_t *) dst->op_params)[2];
    # 从目标操作数的参数中获取第四个整数值，作为 offset
    size_t offset  = ((int32_t *) dst->op_params)[3];
    # 从目标操作数的参数中获取第五个整数值，转换为布尔值，作为 inplace
    bool   inplace = (bool) ((int32_t *) dst->op_params)[4];

    # 如果不是原地操作且参数类型为 GGML_TASK_INIT
    if (!inplace && (params->type == GGML_TASK_INIT)) {
        # 在初始化阶段执行 memcpy 需要在线程间同步以避免竞争条件
        # => 在初始化阶段执行
        memcpy(
            ((char *)  dst->data),
            ((char *) src0->data),
            ggml_nbytes(dst));
    }

    # 如果参数类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    # 获取当前线程的索引和线程总数
    const int ith = params->ith;
    const int nth = params->nth;

    # 获取 src1 的行数和列数
    const int nr = ggml_nrows(src1);
    const int nc = src1->ne[0];

    # 定义宏 GGML_TENSOR_LOCALS，获取 src1 的 ne 和 nb
    GGML_TENSOR_LOCALS(int64_t, ne1, src1, ne)
    GGML_TENSOR_LOCALS(size_t,  nb1, src1, nb)

    # 获取 src0 的元素大小
    const size_t nb0 = ggml_element_size(src0);

    # 定义一些临时变量
    const size_t nb00 = nb0;
    const size_t nb01 = nb1;
    const size_t nb02 = nb2;
    const size_t nb03 = nb3;

    # 断言，确保偏移量加上各维度的索引乘以对应的大小不超过目标数据的字节数
    GGML_ASSERT(offset + (ne10 == 0 ? 0 : ne10-1)*nb0  + (ne11 == 0 ? 0 : ne11-1)*nb1  + (ne12 == 0 ? 0 : ne12-1)*nb2  + (ne13 == 0 ? 0 : ne13-1)*nb3  < ggml_nbytes(dst));
    # 断言，确保偏移量加上各维度的索引乘以对应的大小不超过源数据的字节数
    GGML_ASSERT(offset + (ne10 == 0 ? 0 : ne10-1)*nb00 + (ne11 == 0 ? 0 : ne11-1)*nb01 + (ne12 == 0 ? 0 : ne12-1)*nb02 + (ne13 == 0 ? 0 : ne13-1)*nb03 < ggml_nbytes(src0));

    # 断言，确保 nb10 的大小为 float 的大小
    GGML_ASSERT(nb10 == sizeof(float));

    # 计算每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    # 计算当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    # 遍历当前线程处理的行
    for (int ir = ir0; ir < ir1; ++ir) {
        # 根据 src1 的形状和偏移量，以及当前行的索引计算出在 src0 和 dst 中的相应索引
        # => 相同的索引
        const int i3 = ir/(ne12*ne11);
        const int i2 = (ir - i3*ne12*ne11)/ne11;
        const int i1 = (ir - i3*ne12*ne11 - i2*ne11);
#ifdef GGML_USE_ACCELERATE
        // 如果定义了 GGML_USE_ACCELERATE，则使用 vDSP_vadd 函数进行向量加法运算
        vDSP_vadd(
                (float *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01 + offset), 1,
                (float *) ((char *) src1->data + i3*nb13 + i2*nb12 + i1*nb11), 1,
                (float *) ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1  + offset), 1, nc);
#else
        // 如果未定义 GGML_USE_ACCELERATE，则使用 ggml_vec_add_f32 函数进行向量加法运算
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
    // 根据 src0 的类型进行不同的计算
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                // 如果 src0 的类型是 GGML_TYPE_F32，则调用 ggml_compute_forward_acc_f32 函数进行计算
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
        default:
            {
                // 如果 src0 的类型不在上述情况中，则断言失败
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
    // 断言，确保 params->ith 等于 0
    assert(params->ith == 0);
    // 断言，确保 src0、src1 和 dst 的形状相同
    assert(ggml_are_same_shape(src0, src1) && ggml_are_same_shape(src0, dst));

    // 如果 params->type 是 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取 src0 的行数
    const int nr  = ggml_nrows(src0);

    // 定义 GGML_TENSOR_BINARY_OP_LOCALS

    // 断言，确保 nb0 的大小等于 sizeof(float)
    GGML_ASSERT( nb0 == sizeof(float));
    # 确保 nb00 等于 float 类型的大小
    GGML_ASSERT(nb00 == sizeof(float));

    # 如果 nb10 等于 float 类型的大小
    if (nb10 == sizeof(float)) {
        # 遍历 nr 次数
        for (int ir = 0; ir < nr; ++ir) {
            # 计算三维数组中的索引 i3, i2, i1
            # i3 表示 ir 在 ne2*ne1 中的位置
            const int i3 = ir/(ne2*ne1);
            # i2 表示 ir 在 ne1 中的位置
            const int i2 = (ir - i3*ne2*ne1)/ne1;
            # i1 表示 ir 在 ne1 中的位置
            const int i1 = (ir - i3*ne2*ne1 - i2*ne1);
#ifdef GGML_USE_ACCELERATE
            // 如果定义了 GGML_USE_ACCELERATE，则使用 vDSP_vsub 函数进行向量减法操作
            vDSP_vsub(
                    (float *) ((char *) src1->data + i3*nb13 + i2*nb12 + i1*nb11), 1,
                    (float *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01), 1,
                    (float *) ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1 ), 1,
                    ne0);
#else
            // 如果未定义 GGML_USE_ACCELERATE，则使用 ggml_vec_sub_f32 函数进行向量减法操作
            ggml_vec_sub_f32(ne0,
                    (float *) ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1 ),
                    (float *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01),
                    (float *) ((char *) src1->data + i3*nb13 + i2*nb12 + i1*nb11));
#endif
                // }
            // }
        }
    } else {
        // 如果 src1 不是连续的
        for (int ir = 0; ir < nr; ++ir) {
            // src0、src1 和 dst 具有相同的形状 => 相同的索引
            const int i3 = ir/(ne2*ne1);
            const int i2 = (ir - i3*ne2*ne1)/ne1;
            const int i1 = (ir - i3*ne2*ne1 - i2*ne1);

            float * dst_ptr  = (float *) ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1 );
            float * src0_ptr = (float *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01);
            for (int i0 = 0; i0 < ne0; i0++) {
                float * src1_ptr = (float *) ((char *) src1->data + i3*nb13 + i2*nb12 + i1*nb11 + i0*nb10);

                dst_ptr[i0] = src0_ptr[i0] - *src1_ptr;
            }
        }
    }
}

static void ggml_compute_forward_sub(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                // 如果 src0 的类型是 GGML_TYPE_F32，则调用 ggml_compute_forward_sub_f32 函数
                ggml_compute_forward_sub_f32(params, src0, src1, dst);
            } break;
        default:
            {
                // 如果 src0 的类型不是 GGML_TYPE_F32，则断言失败
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
    GGML_ASSERT(ggml_can_repeat_rows(src1, src0) && ggml_are_same_shape(src0, dst));

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }
    const int ith = params->ith;
    const int nth = params->nth;

#ifdef GGML_USE_CLBLAST
    if (src1->backend == GGML_BACKEND_GPU) {
        if (ith == 0) {
            ggml_cl_mul(src0, src1, dst);
        }
        return;
    }
#endif

    const int64_t nr = ggml_nrows(src0);

    GGML_TENSOR_BINARY_OP_LOCALS

    GGML_ASSERT( nb0 == sizeof(float));
    GGML_ASSERT(nb00 == sizeof(float));
    GGML_ASSERT(ne00 == ne10);

    if (nb10 == sizeof(float)) {
        for (int64_t ir = ith; ir < nr; ir += nth) {
            // src0 and dst are same shape => same indices
            const int64_t i03 = ir/(ne02*ne01);
            const int64_t i02 = (ir - i03*ne02*ne01)/ne01;
            const int64_t i01 = (ir - i03*ne02*ne01 - i02*ne01);

            const int64_t i13 = i03 % ne13;
            const int64_t i12 = i02 % ne12;
            const int64_t i11 = i01 % ne11;

            float * dst_ptr  = (float *) ((char *) dst->data  + i03*nb3  + i02*nb2  + i01*nb1 );
            float * src0_ptr = (float *) ((char *) src0->data + i03*nb03 + i02*nb02 + i01*nb01);
            float * src1_ptr = (float *) ((char *) src1->data + i13*nb13 + i12*nb12 + i11*nb11);

#ifdef GGML_USE_ACCELERATE
            UNUSED(ggml_vec_mul_f32);

            vDSP_vmul( src0_ptr, 1, src1_ptr, 1, dst_ptr,  1, ne00);
#else
            ggml_vec_mul_f32(ne00, dst_ptr, src0_ptr, src1_ptr);
#endif
                // }
            // }
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

            for (int64_t i0 = 0; i0 < ne00; i0++) {
                float * src1_ptr = (float *) ((char *) src1->data + i13*nb13 + i12*nb12 + i11*nb11 + i0*nb10);

                dst_ptr[i0] = src0_ptr[i0] * (*src1_ptr);
            }
        }
    }
}
# 计算两个张量的乘法
static void ggml_compute_forward_mul(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    # 断言src1的类型为GGML_TYPE_F32
    GGML_ASSERT(src1->type == GGML_TYPE_F32 && "only f32 src1 supported for now");

    # 根据src0的类型进行不同的处理
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                # 调用ggml_compute_forward_mul_f32函数进行计算
                ggml_compute_forward_mul_f32(params, src0, src1, dst);
            } break;
        default:
            {
                # 断言失败
                GGML_ASSERT(false);
            } break;
    }
}

# ggml_compute_forward_div

# 计算两个张量的除法（针对f32类型）
static void ggml_compute_forward_div_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    # 断言params的ith属性为0
    assert(params->ith == 0);
    # 断言src0、src1和dst具有相同的形状
    assert(ggml_are_same_shape(src0, src1) && ggml_are_same_shape(src0, dst));

    # 如果params的type为GGML_TASK_INIT或GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    # 获取src0的行数
    const int nr  = ggml_nrows(src0);

    # 定义一些局部变量

    # 断言nb0的大小为float类型
    GGML_ASSERT( nb0 == sizeof(float));
    # 断言nb00的大小为float类型
    GGML_ASSERT(nb00 == sizeof(float));

    # 如果nb10的大小为float类型
    if (nb10 == sizeof(float)) {
        # 遍历行数
        for (int ir = 0; ir < nr; ++ir) {
            # 计算索引
            const int i3 = ir/(ne2*ne1);
            const int i2 = (ir - i3*ne2*ne1)/ne1;
            const int i1 = (ir - i3*ne2*ne1 - i2*ne1);

            # 如果使用加速库
            # 调用vDSP_vdiv函数进行向量除法计算
            vDSP_vdiv(
                    (float *) ((char *) src1->data + i3*nb13 + i2*nb12 + i1*nb11), 1,
                    (float *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01), 1,
                    (float *) ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1 ), 1,
                    ne0);
#else
            ggml_vec_div_f32(ne0,
                    (float *) ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1 ),
                    (float *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01),
                    (float *) ((char *) src1->data + i3*nb13 + i2*nb12 + i1*nb11));
#endif
                // }
            // }
        }
    } else {
        // 如果 src1 不是连续的
        for (int ir = 0; ir < nr; ++ir) {
            // src0, src1 和 dst 具有相同的形状 => 相同的索引
            const int i3 = ir/(ne2*ne1);
            const int i2 = (ir - i3*ne2*ne1)/ne1;
            const int i1 = (ir - i3*ne2*ne1 - i2*ne1);

            float * dst_ptr  = (float *) ((char *) dst->data  + i3*nb3  + i2*nb2  + i1*nb1 );
            float * src0_ptr = (float *) ((char *) src0->data + i3*nb03 + i2*nb02 + i1*nb01);
            for (int i0 = 0; i0 < ne0; i0++) {
                float * src1_ptr = (float *) ((char *) src1->data + i3*nb13 + i2*nb12 + i1*nb11 + i0*nb10);

                dst_ptr[i0] = src0_ptr[i0] / (*src1_ptr);
            }
        }
    }
}

static void ggml_compute_forward_div(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                ggml_compute_forward_div_f32(params, src0, src1, dst);
            } break;
        default:
            {
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_sqr

static void ggml_compute_forward_sqr_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    assert(params->ith == 0);
    assert(ggml_are_same_shape(src0, dst));

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    const int n     = ggml_nrows(src0);
    const int nc    = src0->ne[0];
    # 检查目标数据的第一个维度的字节数是否为浮点数的大小
    assert( dst->nb[0] == sizeof(float));
    # 检查源数据的第一个维度的字节数是否为浮点数的大小
    assert(src0->nb[0] == sizeof(float));

    # 遍历 n 次
    for (int i = 0; i < n; i++) {
        # 调用 ggml_vec_sqr_f32 函数，对两个浮点数数组进行平方操作
        ggml_vec_sqr_f32(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1])));
    }
// 计算前向平方操作
static void ggml_compute_forward_sqr(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的操作
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用对应的前向平方操作函数
                ggml_compute_forward_sqr_f32(params, src0, dst);
            } break;
        // 如果输入张量类型不在已知类型中
        default:
            {
                // 抛出断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_sqrt

// 计算前向平方根操作（针对 F32 类型的张量）
static void ggml_compute_forward_sqrt_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数的正确性
    assert(params->ith == 0);
    assert(ggml_are_same_shape(src0, dst));

    // 如果参数类型为初始化或者结束任务，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取张量的行数和列数
    const int n  = ggml_nrows(src0);
    const int nc = src0->ne[0];

    // 断言目标张量和源张量的数据类型为 float
    assert( dst->nb[0] == sizeof(float));
    assert(src0->nb[0] == sizeof(float));

    // 遍历张量数据进行平方根操作
    for (int i = 0; i < n; i++) {
        ggml_vec_sqrt_f32(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1])));
    }
}

// 计算前向平方根操作
static void ggml_compute_forward_sqrt(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的操作
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用对应的前向平方根操作函数
                ggml_compute_forward_sqrt_f32(params, src0, dst);
            } break;
        // 如果输入张量类型不在已知类型中
        default:
            {
                // 抛出断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_log

// 计算前向对数操作（针对 F32 类型的张量）
static void ggml_compute_forward_log_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数的正确性
    GGML_ASSERT(params->ith == 0);
    GGML_ASSERT(ggml_are_same_shape(src0, dst));

    // 如果参数类型为初始化或者结束任务，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }
    // 获取 src0 的行数
    const int n  = ggml_nrows(src0);
    // 获取 src0 的列数
    const int nc = src0->ne[0];

    // 断言 dst 的第一个维度的大小等于 float 类型的大小
    GGML_ASSERT( dst->nb[0] == sizeof(float));
    // 断言 src0 的第一个维度的大小等于 float 类型的大小
    GGML_ASSERT(src0->nb[0] == sizeof(float));

    // 遍历 src0 的行
    for (int i = 0; i < n; i++) {
        // 对 src0 和 dst 的数据进行 float 类型的对数运算
        ggml_vec_log_f32(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1])));
    }
// 计算前向对数操作，根据输入参数和源张量计算结果存储到目标张量中
static void ggml_compute_forward_log(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据源张量的数据类型进行不同的操作
    switch (src0->type) {
        // 如果数据类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用 ggml_compute_forward_log_f32 函数进行计算
                ggml_compute_forward_log_f32(params, src0, dst);
            } break;
        // 如果数据类型为其他类型
        default:
            {
                // 断言失败
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_sum

// 计算前向求和操作，针对数据类型为 float 的情况
static void ggml_compute_forward_sum_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数的 ith 属性为 0
    assert(params->ith == 0);
    // 断言目标张量为标量
    assert(ggml_is_scalar(dst));

    // 如果参数的类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 断言源张量为标量
    assert(ggml_is_scalar(dst));
    // 断言源张量的第一个维度大小为 float 的大小
    assert(src0->nb[0] == sizeof(float));

    // 定义局部变量 ne0 和 nb0
    GGML_TENSOR_LOCALS(int64_t, ne0, src0, ne)
    GGML_TENSOR_LOCALS(size_t,  nb0, src0, nb)

    // 定义 float 类型的 sum 和 row_sum 变量
    ggml_float sum     = 0;
    ggml_float row_sum = 0;

    // 循环计算求和
    for (int64_t i03 = 0; i03 < ne03; i03++) {
        for (int64_t i02 = 0; i02 < ne02; i02++) {
            for (int64_t i01 = 0; i01 < ne01; i01++) {
                // 调用 ggml_vec_sum_f32_ggf 函数计算行求和
                ggml_vec_sum_f32_ggf(ne00,
                        &row_sum,
                        (float *) ((char *) src0->data + i01*nb01 + i02*nb02 + i03*nb03));
                sum += row_sum;
            }
        }
    }
    // 将求和结果存储到目标张量的数据中
    ((float *) dst->data)[0] = sum;
}

// 计算前向求和操作，针对数据类型为 ggml_fp16_t 的情况
static void ggml_compute_forward_sum_f16(
    const struct ggml_compute_params * params,
    const struct ggml_tensor * src0,
          struct ggml_tensor * dst) {
    // 断言参数的 ith 属性为 0
    assert(params->ith == 0);
    // 断言目标张量为标量
    assert(ggml_is_scalar(dst));

    // 如果参数的类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 断言源张量的第一个维度大小为 ggml_fp16_t 的大小
    assert(src0->nb[0] == sizeof(ggml_fp16_t));

    // 定义 float 类型的 sum 和 row_sum 变量
    float sum = 0;
    float row_sum = 0;
    # 遍历第一维度
    for (int64_t i03 = 0; i03 < ne03; i03++) {
        # 遍历第二维度
        for (int64_t i02 = 0; i02 < ne02; i02++) {
            # 遍历第三维度
            for (int64_t i01 = 0; i01 < ne01; i01++) {
                # 调用 ggml_vec_sum_f16_ggf 函数计算部分和并存储到 row_sum 中
                ggml_vec_sum_f16_ggf(ne00,
                    &row_sum,
                    (ggml_fp16_t *) ((char *) src0->data + i01 * nb01 + i02 * nb02 + i03 * nb03));
                # 将部分和累加到总和中
                sum += row_sum;
            }
        }
    }
    # 将总和转换为 ggml_fp16_t 类型并存储到目标数据的第一个元素中
    ((ggml_fp16_t *) dst->data)[0] = GGML_FP32_TO_FP16(sum);
// 计算 int32 类型的张量的前向和
static void ggml_compute_forward_sum_i32(
    const struct ggml_compute_params * params,  // 接收计算参数的指针
    const struct ggml_tensor * src0,  // 接收输入张量的指针
          struct ggml_tensor * dst) {  // 接收输出张量的指针
    assert(params->ith == 0);  // 断言参数中的 ith 属性为 0
    assert(ggml_is_scalar(dst));  // 断言输出张量为标量

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  // 如果参数中的类型为初始化或者结束任务，则返回
        return;
    }

    assert(src0->nb[0] == sizeof(int32_t));  // 断言输入张量的第一个维度大小为 int32_t 的大小

    GGML_TENSOR_LOCALS(int64_t, ne0, src0, ne)  // 定义输入张量的局部变量 ne0
    GGML_TENSOR_LOCALS(size_t,  nb0, src0, nb)  // 定义输入张量的局部变量 nb0

    int64_t sum = 0;  // 初始化 sum 为 0
    int64_t row_sum = 0;  // 初始化 row_sum 为 0

    for (int64_t i03 = 0; i03 < ne03; i03++) {  // 循环遍历第三维度
        for (int64_t i02 = 0; i02 < ne02; i02++) {  // 循环遍历第二维度
            for (int64_t i01 = 0; i01 < ne01; i01++) {  // 循环遍历第一维度
                ggml_vec_sum_i32_ggf(ne00,  // 调用 ggml_vec_sum_i32_ggf 函数
                    &row_sum,  // 传入 row_sum 的地址
                    (int32_t *) ((char *) src0->data + i01 * nb01 + i02 * nb02 + i03 * nb03));  // 传入指定位置的 int32_t 数据
                sum += row_sum;  // 将 row_sum 加到 sum 上
            }
        }
    }
    ((int32_t *) dst->data)[0] = sum;  // 将 sum 的值存入输出张量的数据中
}

// 根据输入张量的类型选择对应的前向和计算函数
static void ggml_compute_forward_sum(
        const struct ggml_compute_params * params,  // 接收计算参数的指针
        const struct ggml_tensor * src0,  // 接收输入张量的指针
        struct ggml_tensor * dst) {  // 接收输出张量的指针
    switch (src0->type) {  // 根据输入张量的类型进行选择
        case GGML_TYPE_F32:  // 如果类型为 GGML_TYPE_F32
            {
                ggml_compute_forward_sum_f32(params, src0, dst);  // 调用对应的计算函数
            } break;
        case GGML_TYPE_F16:  // 如果类型为 GGML_TYPE_F16
            {
                ggml_compute_forward_sum_f16(params, src0, dst);  // 调用对应的计算函数
            } break;
        case GGML_TYPE_I32:  // 如果类型为 GGML_TYPE_I32
            {
                ggml_compute_forward_sum_i32(params, src0, dst);  // 调用对应的计算函数
            } break;
        default:  // 如果类型为其他
            {
                GGML_ASSERT(false);  // 断言为假
            } break;
    }
}

// 计算 float32 类型的张量的行和
static void ggml_compute_forward_sum_rows_f32(
        const struct ggml_compute_params * params,  // 接收计算参数的指针
        const struct ggml_tensor * src0,  // 接收输入张量的指针
        struct ggml_tensor * dst) {  // 接收输出张量的指针
    GGML_ASSERT(params->ith == 0);  // 断言参数中的 ith 属性为 0

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  // 如果参数中的类型为初始化或者结束任务，则返回
        return;
    }
    # 检查src0的第一个维度的大小是否为float的大小
    GGML_ASSERT(src0->nb[0] == sizeof(float));
    # 检查dst的第一个维度的大小是否为float的大小
    GGML_ASSERT(dst->nb[0] == sizeof(float));

    # 定义一些本地变量，用于后续的操作
    GGML_TENSOR_UNARY_OP_LOCALS

    # 检查ne0是否等于1
    GGML_ASSERT(ne0 == 1);
    # 检查ne1是否等于ne01
    GGML_ASSERT(ne1 == ne01);
    # 检查ne2是否等于ne02
    GGML_ASSERT(ne2 == ne02);
    # 检查ne3是否等于ne03
    GGML_ASSERT(ne3 == ne03);

    # 循环遍历多维数组
    for (int64_t i3 = 0; i3 < ne03; i3++) {
        for (int64_t i2 = 0; i2 < ne02; i2++) {
            for (int64_t i1 = 0; i1 < ne01; i1++) {
                # 计算src_row和dst_row的指针位置
                float * src_row = (float *) ((char *) src0->data + i1*nb01 + i2*nb02 + i3*nb03);
                float * dst_row = (float *) ((char *) dst->data  + i1*nb1  + i2*nb2  + i3*nb3);
                # 初始化行和为0
                float row_sum = 0;
                # 计算src_row的前ne00个元素的和
                ggml_vec_sum_f32(ne00, &row_sum, src_row);
                # 将行和赋值给dst_row的第一个元素
                dst_row[0] = row_sum;
            }
        }
    }
// 计算前向求和行的函数
static void ggml_compute_forward_sum_rows(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的处理
    switch (src0->type) {
        // 如果是单精度浮点型
        case GGML_TYPE_F32:
            {
                // 调用单精度浮点型的前向求和行函数
                ggml_compute_forward_sum_rows_f32(params, src0, dst);
            } break;
        // 如果是其他类型
        default:
            {
                // 抛出断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// 计算前向均值的单精度浮点型函数
static void ggml_compute_forward_mean_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数的索引为0
    assert(params->ith == 0);

    // 如果参数类型为初始化或者结束任务，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 断言输入张量的第一个维度的大小为单精度浮点型的大小
    assert(src0->nb[0] == sizeof(float));

    // 定义一些本地变量
    GGML_TENSOR_UNARY_OP_LOCALS

    // 断言一些维度的大小
    assert(ne0 == 1);
    assert(ne1 == ne01);
    assert(ne2 == ne02);
    assert(ne3 == ne03);

    // 使用UNUSED宏来避免未使用的变量警告
    UNUSED(ne0);
    UNUSED(ne1);
    UNUSED(ne2);
    UNUSED(ne3);

    // 遍历张量的维度进行计算
    for (int64_t i03 = 0; i03 < ne03; i03++) {
        for (int64_t i02 = 0; i02 < ne02; i02++) {
            for (int64_t i01 = 0; i01 < ne01; i01++) {
                // 调用单精度浮点型的向量求和函数
                ggml_vec_sum_f32(ne00,
                        (float *) ((char *)  dst->data + i01*nb1  + i02*nb2  + i03*nb3),
                        (float *) ((char *) src0->data + i01*nb01 + i02*nb02 + i03*nb03));

                // 将求和结果除以维度大小，得到均值
                *(float *) ((char *) dst->data + i01*nb1 + i02*nb2 + i03*nb3) /= (float) ne00;
            }
        }
    }
}

// 计算前向均值的函数
static void ggml_compute_forward_mean(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的处理
    switch (src0->type) {
        // 如果是单精度浮点型
        case GGML_TYPE_F32:
            {
                // 调用单精度浮点型的前向均值函数
                ggml_compute_forward_mean_f32(params, src0, dst);
            } break;
        // 如果是其他类型
        default:
            {
                // 抛出断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// 计算前向最大值的函数
# 计算前向最大值的索引，将结果存储在目标张量中
static void ggml_compute_forward_argmax_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    # 确保参数中的 ith 值为 0
    assert(params->ith == 0);

    # 如果任务类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    # 确保输入张量 src0 的第一个维度的元素大小为 float 类型的大小
    assert(src0->nb[0] == sizeof(float));
    # 确保目标张量 dst 的第一个维度的元素大小为 float 类型的大小
    assert(dst->nb[0] == sizeof(float));

    # 获取输入张量 src0 的第一个和第二个维度的大小
    const int64_t ne00 = src0->ne[0];
    const int64_t ne01 = src0->ne[1];

    # 获取输入张量 src0 的第二个维度的字节数
    const size_t nb01 = src0->nb[1];
    # 获取目标张量 dst 的第一个维度的字节数
    const size_t nb0 = dst->nb[0];

    # 遍历输入张量 src0 的第二个维度
    for (int64_t i1 = 0; i1 < ne01; i1++) {
        # 获取指向当前元素的指针，并将其转换为 float 类型指针
        float * src = (float *) ((char *) src0->data + i1*nb01);
        # 获取指向目标张量当前元素的指针，并将其转换为 int32_t 类型指针
        int32_t * dst_ = (int32_t *) ((char *)  dst->data + i1*nb0);
        # 初始化变量 v 为 0，调用 ggml_vec_argmax_f32 函数计算最大值的索引，并将结果存储在 v 中
        int v = 0;
        ggml_vec_argmax_f32(ne00, &v, src);
        # 将计算得到的最大值的索引存储在目标张量中
        dst_[0] = v;
    }
}

# 根据输入张量的类型调用相应的前向最大值计算函数
static void ggml_compute_forward_argmax(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                ggml_compute_forward_argmax_f32(params, src0, dst);
            } break;
        default:
            {
                GGML_ASSERT(false);
            } break;
    }
}

# ggml_compute_forward_repeat

# 计算前向重复操作，将结果存储在目标张量中
static void ggml_compute_forward_repeat_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    # 确保参数中的 ith 值为 0
    GGML_ASSERT(params->ith == 0);
    # 确保输入张量 src0 和目标张量 dst 可以进行重复操作
    GGML_ASSERT(ggml_can_repeat(src0, dst));

    # 如果任务类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    # 定义局部变量
    GGML_TENSOR_UNARY_OP_LOCALS

    # 由于在 ggml_can_repeat 中已经保证是整数，计算每个维度的重复次数
    const int nr0 = (int)(ne0/ne00);
    const int nr1 = (int)(ne1/ne01);
    const int nr2 = (int)(ne2/ne02);
    const int nr3 = (int)(ne3/ne03);

    # TODO: 支持转置/置换张量
    # 确保目标张量 dst 的第一个维度的元素大小为 float 类型的大小
    GGML_ASSERT(nb0  == sizeof(float));
}
    // 确保 nb00 等于 float 的大小
    GGML_ASSERT(nb00 == sizeof(float));

    // TODO: 这可能不是最优的解决方案？
    for                         (int i3 = 0; i3 < nr3;  i3++) {
        for                     (int k3 = 0; k3 < ne03; k3++) {
            for                 (int i2 = 0; i2 < nr2;  i2++) {
                for             (int k2 = 0; k2 < ne02; k2++) {
                    for         (int i1 = 0; i1 < nr1;  i1++) {
                        for     (int k1 = 0; k1 < ne01; k1++) {
                            for (int i0 = 0; i0 < nr0;  i0++) {
                                // 复制长度为 ne00 的 float 数组
                                ggml_vec_cpy_f32(ne00,
                                        // 目标地址
                                        (float *) ((char *)  dst->data + (i3*ne03 + k3)*nb3  + (i2*ne02 + k2)*nb2  + (i1*ne01 + k1)*nb1  + (i0*ne00)*nb0),
                                        // 源地址
                                        (float *) ((char *) src0->data + (          k3)*nb03 + (          k2)*nb02 + (          k1)*nb01));
                            }
                        }
                    }
                }
            }
        }
    }
# 计算前向重复操作，将源张量中的数据复制到目标张量中
static void ggml_compute_forward_repeat_f16(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    # 断言参数中的 ith 值为 0
    GGML_ASSERT(params->ith == 0);
    # 断言源张量可以重复到目标张量
    GGML_ASSERT(ggml_can_repeat(src0, dst));

    # 如果任务类型为初始化或者结束，直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    # 定义局部变量
    GGML_TENSOR_UNARY_OP_LOCALS;

    # 由于在 ggml_can_repeat 中已经保证是整数，所以进行强制类型转换
    const int nr0 = (int)(ne0/ne00);
    const int nr1 = (int)(ne1/ne01);
    const int nr2 = (int)(ne2/ne02);
    const int nr3 = (int)(ne3/ne03);

    # 断言 nb0 和 nb00 的大小为 ggml_fp16_t 类型的大小
    GGML_ASSERT(nb0  == sizeof(ggml_fp16_t));
    GGML_ASSERT(nb00 == sizeof(ggml_fp16_t));

    # 循环遍历多维张量，进行数据复制操作
    for (int i3 = 0; i3 < nr3;  i3++) {
        for (int k3 = 0; k3 < ne03; k3++) {
            for (int i2 = 0; i2 < nr2;  i2++) {
                for (int k2 = 0; k2 < ne02; k2++) {
                    for (int i1 = 0; i1 < nr1;  i1++) {
                        for (int k1 = 0; k1 < ne01; k1++) {
                            for (int i0 = 0; i0 < nr0;  i0++) {
                                # 计算目标张量和源张量的偏移位置，进行数据复制
                                ggml_fp16_t * y = (ggml_fp16_t *) ((char *)  dst->data + (i3*ne03 + k3)*nb3  + (i2*ne02 + k2)*nb2  + (i1*ne01 + k1)*nb1  + (i0*ne00)*nb0);
                                ggml_fp16_t * x = (ggml_fp16_t *) ((char *) src0->data + (          k3)*nb03 + (          k2)*nb02 + (          k1)*nb01);
                                # 逐个元素复制数据
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
static void ggml_compute_forward_repeat(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    switch (src0->type) {
        case GGML_TYPE_F16:
            {
                ggml_compute_forward_repeat_f16(params, src0, dst);
            } break;
        case GGML_TYPE_F32:
            {
                ggml_compute_forward_repeat_f32(params, src0, dst);
            } break;
        default:
            {
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_repeat_back

static void ggml_compute_forward_repeat_back_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    GGML_ASSERT(params->ith == 0);
    GGML_ASSERT(ggml_can_repeat(dst, src0));

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    GGML_TENSOR_UNARY_OP_LOCALS

    // 根据 ne00/ne0 计算重复次数并转换为整数
    const int nr0 = (int)(ne00/ne0);
    const int nr1 = (int)(ne01/ne1);
    const int nr2 = (int)(ne02/ne2);
    const int nr3 = (int)(ne03/ne3);

    // TODO: 支持转置/置换张量
    GGML_ASSERT(nb0  == sizeof(float));
    GGML_ASSERT(nb00 == sizeof(float));

    // 如果目标张量是连续的，则将其数据全部设置为0
    if (ggml_is_contiguous(dst)) {
        ggml_vec_set_f32(ne0*ne1*ne2*ne3, dst->data, 0);
    } else {
        // 否则，逐个元素将数据设置为0
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
    for                         (int i3 = 0; i3 < nr3; i3++) {
        // 循环遍历第一维度
        for                     (int k3 = 0; k3 < ne3; k3++) {
            // 循环遍历第一维度的元素
            for                 (int i2 = 0; i2 < nr2; i2++) {
                // 循环遍历第二维度
                for             (int k2 = 0; k2 < ne2; k2++) {
                    // 循环遍历第二维度的元素
                    for         (int i1 = 0; i1 < nr1; i1++) {
                        // 循环遍历第三维度
                        for     (int k1 = 0; k1 < ne1; k1++) {
                            // 循环遍历第三维度的元素
                            for (int i0 = 0; i0 < nr0; i0++) {
                                // 循环遍历第四维度
                                ggml_vec_acc_f32(ne0,
                                        // 调用 ggml_vec_acc_f32 函数，传入参数
                                        (float *) ((char *)  dst->data + (         k3)*nb3  + (         k2)*nb2  + (         k1)*nb1),
                                        // 调用 ggml_vec_acc_f32 函数，传入参数
                                        (float *) ((char *) src0->data + (i3*ne3 + k3)*nb03 + (i2*ne2 + k2)*nb02 + (i1*ne1 + k1)*nb01 + (i0*ne0)*nb00));
                            }
                        }
                    }
                }
            }
        }
    }
// 计算前向和反向传播的函数，根据输入参数和张量类型进行计算
static void ggml_compute_forward_repeat_back(
        const struct ggml_compute_params * params,  // 输入参数结构体指针
        const struct ggml_tensor * src0,  // 输入张量指针
        struct ggml_tensor * dst) {  // 输出张量指针
    // 根据输入张量的类型进行不同的计算
    switch (src0->type) {
        case GGML_TYPE_F32:  // 如果输入张量类型为 GGML_TYPE_F32
            {
                ggml_compute_forward_repeat_back_f32(params, src0, dst);  // 调用相应的计算函数
            } break;
        default:  // 如果输入张量类型不在已知范围内
            {
                GGML_ASSERT(false);  // 抛出断言错误
            } break;
    }
}

// ggml_compute_forward_concat

// 计算前向拼接的函数，根据输入参数和张量类型进行计算
static void ggml_compute_forward_concat_f32(
    const struct ggml_compute_params * params,  // 输入参数结构体指针
    const struct ggml_tensor * src0,  // 输入张量指针
    const struct ggml_tensor * src1,  // 输入张量指针
    struct ggml_tensor * dst) {  // 输出张量指针
    // 如果参数类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }
    // 断言输入张量的第一个维度大小为 sizeof(float)
    GGML_ASSERT(src0->nb[0] == sizeof(float));

    const int ith = params->ith;  // 获取参数中的 ith 值

    GGML_TENSOR_BINARY_OP_LOCALS  // 定义张量二元操作的本地变量

    // TODO: support for transposed / permuted tensors
    GGML_ASSERT(nb0  == sizeof(float));  // 断言张量的大小为 sizeof(float)
    GGML_ASSERT(nb00 == sizeof(float));  // 断言张量的大小为 sizeof(float)
    GGML_ASSERT(nb10 == sizeof(float));  // 断言张量的大小为 sizeof(float)
}
    // 遍历第三维度的数据
    for (int i3 = 0; i3 < ne3; i3++) {
        // 遍历第二维度的数据
        for (int i2 = ith; i2 < ne2; i2++) {
            // 如果 i2 小于 ne02，则处理 src0 数据
            if (i2 < ne02) { // src0
                // 遍历第一维度的数据
                for (int i1 = 0; i1 < ne1; i1++) {
                    // 遍历第零维度的数据
                    for (int i0 = 0; i0 < ne0; i0++) {
                        // 计算 src0 数据的地址
                        const float * x = (float *)((char *) src0->data + i0 * nb00 + i1 * nb01 + i2 * nb02 + i3 * nb03);

                        // 计算 dst 数据的地址
                        float * y = (float *)((char *)dst->data + i0 * nb0 + i1 * nb1 + i2 * nb2 + i3 * nb3);
                        // 将 src0 数据复制到 dst 数据
                        *y = *x;
                    }
                }
            } // src1
            // 如果 i2 大于等于 ne02，则处理 src1 数据
            else {
                // 遍历第一维度的数据
                for (int i1 = 0; i1 < ne1; i1++) {
                    // 遍历第零维度的数据
                    for (int i0 = 0; i0 < ne0; i0++) {
                        // 计算 src1 数据的地址
                        const float * x = (float *)((char *) src1->data + i0 * nb10 + i1 * nb11 + (i2 - ne02) * nb12 + i3 * nb13);

                        // 计算 dst 数据的地址
                        float * y = (float *)((char *)dst->data + i0 * nb0 + i1 * nb1 + i2 * nb2 + i3 * nb3);
                        // 将 src1 数据复制到 dst 数据
                        *y = *x;
                    }
                }
            }
        }
    }
// 计算拼接操作的前向传播，将两个输入张量拼接成一个输出张量
static void ggml_compute_forward_concat(
    const struct ggml_compute_params* params,
    const struct ggml_tensor* src0,
    const struct ggml_tensor* src1,
    struct ggml_tensor* dst) {
    // 根据第一个输入张量的类型进行不同的处理
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用针对 GGML_TYPE_F32 类型的拼接操作的前向传播函数
                ggml_compute_forward_concat_f32(params, src0, src1, dst);
            } break;
        // 如果输入张量类型不是 GGML_TYPE_F32
        default:
            {
                // 抛出断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_abs

// 针对 GGML_TYPE_F32 类型的绝对值操作的前向传播函数
static void ggml_compute_forward_abs_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数中的 ith 值为 0
    assert(params->ith == 0);
    // 断言输入张量和输出张量具有相同的形状
    assert(ggml_are_same_shape(src0, dst));

    // 如果任务类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取输入张量的行数和通道数
    const int n  = ggml_nrows(src0);
    const int nc = src0->ne[0];

    // 断言输出张量和输入张量的字节大小为 sizeof(float)
    assert(dst->nb[0]  == sizeof(float));
    assert(src0->nb[0] == sizeof(float));

    // 遍历每一行，对输入张量的每个元素取绝对值，存入输出张量
    for (int i = 0; i < n; i++) {
        ggml_vec_abs_f32(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1])));
    }
}

// 针对 GGML_TYPE_F32 类型的绝对值操作的前向传播函数
static void ggml_compute_forward_abs(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的处理
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用针对 GGML_TYPE_F32 类型的绝对值操作的前向传播函数
                ggml_compute_forward_abs_f32(params, src0, dst);
            } break;
        // 如果输入张量类型不是 GGML_TYPE_F32
        default:
            {
                // 抛出断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_sgn

// 针对 GGML_TYPE_F32 类型的符号函数操作的前向传播函数
static void ggml_compute_forward_sgn_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数中的 ith 值为 0
    assert(params->ith == 0);
    // 断言输入张量和输出张量具有相同的形状
    assert(ggml_are_same_shape(src0, dst));

    // 如果任务类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }
    # 获取 src0 的行数
    const int n  = ggml_nrows(src0);
    # 获取 src0 的列数
    const int nc = src0->ne[0];

    # 确保 dst 的每个元素占用的字节数为 float 类型的大小
    assert(dst->nb[0]  == sizeof(float));
    # 确保 src0 的每个元素占用的字节数为 float 类型的大小
    assert(src0->nb[0] == sizeof(float));

    # 遍历每一行，对每一行的数据进行处理
    for (int i = 0; i < n; i++) {
        # 对每一行的数据进行符号函数处理，结果存入 dst 中
        ggml_vec_sgn_f32(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1])));
    }
// 计算正向传播的符号函数
static void ggml_compute_forward_sgn(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的处理
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用 ggml_compute_forward_sgn_f32 函数处理
                ggml_compute_forward_sgn_f32(params, src0, dst);
            } break;
        // 如果输入张量类型不是 GGML_TYPE_F32
        default:
            {
                // 断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// 计算正向传播的负数函数

// 处理 GGML_TYPE_F32 类型的张量
static void ggml_compute_forward_neg_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数的 ith 属性为 0
    assert(params->ith == 0);
    // 断言输入张量和输出张量具有相同的形状
    assert(ggml_are_same_shape(src0, dst));

    // 如果任务类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取输入张量的行数
    const int n  = ggml_nrows(src0);
    // 获取输入张量的通道数
    const int nc = src0->ne[0];

    // 断言输出张量的第一个维度的字节数为 float 类型的大小
    assert(dst->nb[0]  == sizeof(float));
    // 断言输入张量的第一个维度的字节数为 float 类型的大小
    assert(src0->nb[0] == sizeof(float));

    // 遍历每一行，对输入张量的数据取负并存入输出张量
    for (int i = 0; i < n; i++) {
        ggml_vec_neg_f32(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1])));
    }
}

// 计算正向传播的负数函数
static void ggml_compute_forward_neg(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的处理
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用 ggml_compute_forward_neg_f32 函数处理
                ggml_compute_forward_neg_f32(params, src0, dst);
            } break;
        // 如果输入张量类型不是 GGML_TYPE_F32
        default:
            {
                // 断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// 计算正向传播的步进函数

// 处理 GGML_TYPE_F32 类型的张量
static void ggml_compute_forward_step_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数的 ith 属性为 0
    assert(params->ith == 0);
    // 断言输入张量和输出张量具有相同的形状
    assert(ggml_are_same_shape(src0, dst));

    // 如果任务类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取输入张量的行数
    const int n  = ggml_nrows(src0);
    # 定义常量nc，其值为src0的第一个维度的大小
    const int nc = src0->ne[0];

    # 断言，确保dst的第一个维度的字节数等于float类型的字节数
    assert(dst->nb[0]  == sizeof(float));
    # 断言，确保src0的第一个维度的字节数等于float类型的字节数
    assert(src0->nb[0] == sizeof(float));

    # 遍历n次循环
    for (int i = 0; i < n; i++) {
        # 调用ggml_vec_step_f32函数，传入参数nc，以及根据偏移计算出的dst和src0的地址
        ggml_vec_step_f32(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1])));
    }
// 计算前向传播步骤的函数，根据输入参数和源张量计算目标张量
static void ggml_compute_forward_step(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据源张量的数据类型进行不同的处理
    switch (src0->type) {
        // 如果数据类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用处理 GGML_TYPE_F32 类型数据的函数
                ggml_compute_forward_step_f32(params, src0, dst);
            } break;
        // 如果数据类型为其他类型
        default:
            {
                // 断言失败
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_tanh

// 处理 GGML_TYPE_F32 类型数据的函数
static void ggml_compute_forward_tanh_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数的特定条件
    assert(params->ith == 0);
    assert(ggml_are_same_shape(src0, dst));

    // 如果参数类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取张量的行数和列数
    const int n  = ggml_nrows(src0);
    const int nc = src0->ne[0];

    // 断言目标张量和源张量的数据类型为 float
    assert(dst->nb[0]  == sizeof(float));
    assert(src0->nb[0] == sizeof(float));

    // 遍历张量的行数，对每一行的数据进行 tanh 操作
    for (int i = 0; i < n; i++) {
        ggml_vec_tanh_f32(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1])));
    }
}

// 处理 GGML_TYPE_F32 类型数据的函数
static void ggml_compute_forward_tanh(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据源张量的数据类型进行不同的处理
    switch (src0->type) {
        // 如果数据类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用处理 GGML_TYPE_F32 类型数据的函数
                ggml_compute_forward_tanh_f32(params, src0, dst);
            } break;
        // 如果数据类型为其他类型
        default:
            {
                // 断言失败
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_elu

// 处理 GGML_TYPE_F32 类型数据的函数
static void ggml_compute_forward_elu_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数的特定条件
    assert(params->ith == 0);
    assert(ggml_are_same_shape(src0, dst));

    // 如果参数类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取张量的行数
    const int n  = ggml_nrows(src0);
    # 定义常量nc为src0的第一个维度的大小
    const int nc = src0->ne[0];

    # 断言dst的第一个维度的大小等于float类型的大小
    assert(dst->nb[0]  == sizeof(float));
    # 断言src0的第一个维度的大小等于float类型的大小
    assert(src0->nb[0] == sizeof(float));

    # 遍历n次
    for (int i = 0; i < n; i++) {
        # 调用ggml_vec_elu_f32函数，传入参数为nc、dst数据指针偏移i倍dst的第二个维度大小、src0数据指针偏移i倍src0的第二个维度大小
        ggml_vec_elu_f32(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1])));
    }
// 计算 ELU 激活函数的前向传播
static void ggml_compute_forward_elu(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的处理
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用处理 GGML_TYPE_F32 类型的函数
                ggml_compute_forward_elu_f32(params, src0, dst);
            } break;
        // 如果输入张量类型不是 GGML_TYPE_F32
        default:
            {
                // 抛出断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_relu

// 计算 ReLU 激活函数的前向传播（针对 F32 类型的张量）
static void ggml_compute_forward_relu_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数的特定条件
    assert(params->ith == 0);
    assert(ggml_are_same_shape(src0, dst));

    // 如果参数类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取张量的行数和第一维度的大小
    const int n  = ggml_nrows(src0);
    const int nc = src0->ne[0];

    // 断言目标张量和源张量的第一维度字节数为 float 类型的大小
    assert(dst->nb[0]  == sizeof(float));
    assert(src0->nb[0] == sizeof(float));

    // 遍历张量并对每个元素应用 ReLU 激活函数
    for (int i = 0; i < n; i++) {
        ggml_vec_relu_f32(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1])));
    }
}

// 计算 ReLU 激活函数的前向传播
static void ggml_compute_forward_relu(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的处理
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用处理 GGML_TYPE_F32 类型的函数
                ggml_compute_forward_relu_f32(params, src0, dst);
            } break;
        // 如果输入张量类型不是 GGML_TYPE_F32
        default:
            {
                // 抛出断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_gelu

// 计算 GELU 激活函数的前向传播（针对 F32 类型的张量）
static void ggml_compute_forward_gelu_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言源张量和目标张量在除了第一维度之外都是连续的
    GGML_ASSERT(ggml_is_contiguous_except_dim_1(src0));
    GGML_ASSERT(ggml_is_contiguous_except_dim_1(dst));
    GGML_ASSERT(ggml_are_same_shape(src0, dst));
    // 如果任务类型是初始化或者结束，直接返回，不执行下面的代码
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取当前线程的索引和总线程数
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取输入数据的列数和行数
    const int nc = src0->ne[0];
    const int nr = ggml_nrows(src0);

    // 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 循环处理当前线程负责的行范围内的数据
    for (int i1 = ir0; i1 < ir1; i1++) {
        // 对输入数据进行 GELU 激活函数处理，并将结果存入目标数据中
        ggml_vec_gelu_f32(nc,
                (float *) ((char *) dst->data  + i1*( dst->nb[1])),
                (float *) ((char *) src0->data + i1*(src0->nb[1])));
#ifndef NDEBUG
        // 如果处于调试模式，则进行以下操作
        for (int k = 0; k < nc; k++) {
            // 从目标数据中获取指定位置的浮点数
            const float x = ((float *) ((char *) dst->data + i1*( dst->nb[1])))[k];
            // 使用 UNUSED 宏来避免编译器警告
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
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                // 调用 ggml_compute_forward_gelu_f32 函数进行计算
                ggml_compute_forward_gelu_f32(params, src0, dst);
            } break;
        default:
            {
                // 如果类型不是 GGML_TYPE_F32，则断言失败
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_gelu_quick

static void ggml_compute_forward_gelu_quick_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言 src0 和 dst 是除了第一维度以外都是连续的
    GGML_ASSERT(ggml_is_contiguous_except_dim_1(src0));
    GGML_ASSERT(ggml_is_contiguous_except_dim_1(dst));
    // 断言 src0 和 dst 的形状相同
    GGML_ASSERT(ggml_are_same_shape(src0, dst));

    // 如果任务类型是 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取线程的索引和总数
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取 src0 的第一维度大小和行数
    const int nc = src0->ne[0];
    const int nr = ggml_nrows(src0);

    // 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    for (int i1 = ir0; i1 < ir1; i1++) {
        // 调用 ggml_vec_gelu_quick_f32 函数进行计算
        ggml_vec_gelu_quick_f32(nc,
                (float *) ((char *) dst->data  + i1*( dst->nb[1])),
                (float *) ((char *) src0->data + i1*(src0->nb[1])));

#ifndef NDEBUG
        // 如果处于调试模式，则进行以下操作
        for (int k = 0; k < nc; k++) {
            // 从目标数据中获取指定位置的浮点数
            const float x = ((float *) ((char *) dst->data + i1*( dst->nb[1])))[k];
            // 使用 UNUSED 宏来避免编译器警告
            UNUSED(x);
            // 断言 x 不是 NaN
            assert(!isnan(x));
            // 断言 x 不是无穷大
            assert(!isinf(x));
        }
#endif
    }
}
# 计算快速 GELU 激活函数的前向传播
static void ggml_compute_forward_gelu_quick(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    # 根据输入张量的类型进行不同的处理
    switch (src0->type) {
        # 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                # 调用处理 GGML_TYPE_F32 类型的函数
                ggml_compute_forward_gelu_quick_f32(params, src0, dst);
            } break;
        # 如果输入张量类型不在已知类型中
        default:
            {
                # 抛出断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_silu

# 计算快速 SiLU 激活函数的前向传播（针对 F32 类型）
static void ggml_compute_forward_silu_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    # 断言输入张量 src0 是除了第一个维度之外是连续的
    GGML_ASSERT(ggml_is_contiguous_except_dim_1(src0));
    # 断言输出张量 dst 是除了第一个维度之外是连续的
    GGML_ASSERT(ggml_is_contiguous_except_dim_1(dst));
    # 断言输入张量 src0 和输出张量 dst 的形状相同
    GGML_ASSERT(ggml_are_same_shape(src0, dst));

    # 如果参数类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    # 获取当前线程的索引和总线程数
    const int ith = params->ith;
    const int nth = params->nth;

    # 获取输入张量的第一个维度大小和行数
    const int nc = src0->ne[0];
    const int nr = ggml_nrows(src0);

    # 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    # 当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    # 遍历处理当前线程负责的行
    for (int i1 = ir0; i1 < ir1; i1++) {
        # 调用 F32 类型的 SiLU 函数处理当前行
        ggml_vec_silu_f32(nc,
                (float *) ((char *) dst->data  + i1*( dst->nb[1])),
                (float *) ((char *) src0->data + i1*(src0->nb[1])));

        # 在调试模式下，检查处理后的数据是否包含 NaN 或无穷大的值
#ifndef NDEBUG
        for (int k = 0; k < nc; k++) {
            const float x = ((float *) ((char *) dst->data + i1*(dst->nb[1])))[k];
            UNUSED(x);
            assert(!isnan(x));
            assert(!isinf(x));
        }
#endif
    }
}

# 计算 SiLU 激活函数的前向传播
static void ggml_compute_forward_silu(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    # 根据 src0 的类型进行不同的操作
    switch (src0->type) {
        # 如果 src0 的类型是 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                # 调用 ggml_compute_forward_silu_f32 函数进行计算，并将结果存储到 dst 中
                ggml_compute_forward_silu_f32(params, src0, dst);
            } break;
        # 如果 src0 的类型不是 GGML_TYPE_F32
        default:
            {
                # 断言，表示出现了不应该出现的情况
                GGML_ASSERT(false);
            } break;
    }
// ggml_compute_forward_leaky

// 计算前向泄漏线性整流单元的操作，针对单精度浮点数
static void ggml_compute_forward_leaky_f32(
        const struct ggml_compute_params * params,  // 输入参数结构体指针
        const struct ggml_tensor * src0,  // 输入张量指针
        struct ggml_tensor * dst) {  // 输出张量指针
    assert(params->ith == 0);  // 断言，确保 ith 等于 0
    assert(ggml_are_same_shape(src0, dst));  // 断言，确保输入和输出张量形状相同

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  // 如果任务类型是初始化或者结束
        return;  // 直接返回
    }

    const int n  = ggml_nrows(src0);  // 获取输入张量的行数
    const int nc = src0->ne[0];  // 获取输入张量的第一个维度大小

    assert(dst->nb[0]  == sizeof(float));  // 断言，确保输出张量的第一个维度大小等于单精度浮点数的大小
    assert(src0->nb[0] == sizeof(float));  // 断言，确保输入张量的第一个维度大小等于单精度浮点数的大小

    for (int i = 0; i < n; i++) {  // 遍历每一行
        ggml_vec_leaky_f32(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),  // 计算输出张量中的偏移位置
                (float *) ((char *) src0->data + i*(src0->nb[1])));  // 计算输入张量中的偏移位置
    }
}

static void ggml_compute_forward_leaky(
        const struct ggml_compute_params * params,  // 输入参数结构体指针
        const struct ggml_tensor * src0,  // 输入张量指针
        struct ggml_tensor * dst) {  // 输出张量指针
    switch (src0->type) {  // 根据输入张量的类型进行切换
        case GGML_TYPE_F32:  // 如果是单精度浮点数
            {
                ggml_compute_forward_leaky_f32(params, src0, dst);  // 调用单精度浮点数的计算函数
            } break;
        default:  // 如果是其他类型
            {
                GGML_ASSERT(false);  // 断言，抛出错误
            } break;
    }
}

// ggml_compute_forward_silu_back

// 计算前向自适应整流线性单元的反向传播操作，针对单精度浮点数
static void ggml_compute_forward_silu_back_f32(
        const struct ggml_compute_params * params,  // 输入参数结构体指针
        const struct ggml_tensor * src0,  // 输入张量指针
        const struct ggml_tensor * grad,  // 梯度张量指针
        struct ggml_tensor * dst) {  // 输出张量指针
    GGML_ASSERT(ggml_is_contiguous_except_dim_1(grad));  // 断言，确保梯度张量在除了第一维度以外是连续的
    GGML_ASSERT(ggml_is_contiguous_except_dim_1(src0));  // 断言，确保输入张量在除了第一维度以外是连续的
    GGML_ASSERT(ggml_is_contiguous_except_dim_1(dst));  // 断言，确保输出张量在除了第一维度以外是连续的
    GGML_ASSERT(ggml_are_same_shape(src0, dst));  // 断言，确保输入和输出张量形状相同
    GGML_ASSERT(ggml_are_same_shape(src0, grad));  // 断言，确保输入和梯度张量形状相同

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  // 如果任务类型是初始化或者结束
        return;  // 直接返回
    }

    const int ith = params->ith;  // 获取当前线程的索引
    const int nth = params->nth;  // 获取线程总数

    const int nc = src0->ne[0];  // 获取输入张量的第一个维度大小
    const int nr = ggml_nrows(src0);  // 获取输入张量的行数

    // 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;  // 计算每个线程处理的行数
    // 为当前线程确定行范围
    const int ir0 = dr*ith;  // 计算当前线程处理的起始行
    const int ir1 = MIN(ir0 + dr, nr);  // 计算当前线程处理的结束行，取ir0 + dr和nr中的较小值

    for (int i1 = ir0; i1 < ir1; i1++) {
        // 对每一行进行反向的 GGML SILU 激活函数的计算
        ggml_vec_silu_backward_f32(nc,
                (float *) ((char *) dst->data  + i1*( dst->nb[1])),  // 访问目标数据中第i1行的数据
                (float *) ((char *) src0->data + i1*(src0->nb[1])),  // 访问源数据中第i1行的数据
                (float *) ((char *) grad->data + i1*(grad->nb[1])));  // 访问梯度数据中第i1行的数据
#ifndef NDEBUG
        // 如果处于调试模式下
        for (int k = 0; k < nc; k++) {
            // 遍历每个通道
            const float x = ((float *) ((char *) dst->data + i1*( dst->nb[1])))[k];
            // 从目标张量中获取数据
            UNUSED(x);
            // 标记 x 为未使用，防止编译器警告
            assert(!isnan(x));
            // 断言 x 不是 NaN
            assert(!isinf(x));
            // 断言 x 不是无穷大
        }
#endif
    }
}

static void ggml_compute_forward_silu_back(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * grad,
        struct ggml_tensor * dst) {
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                ggml_compute_forward_silu_back_f32(params, src0, grad, dst);
                // 如果输入张量类型为 GGML_TYPE_F32，则调用 ggml_compute_forward_silu_back_f32 函数
            } break;
        default:
            {
                GGML_ASSERT(false);
                // 如果输入张量类型不是 GGML_TYPE_F32，则触发断言错误
            } break;
    }
}

// ggml_compute_forward_norm

static void ggml_compute_forward_norm_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    GGML_ASSERT(ggml_are_same_shape(src0, dst));
    // 断言输入张量和输出张量具有相同的形状

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
        // 如果任务类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    }

    GGML_ASSERT(src0->nb[0] == sizeof(float));
    // 断言输入张量的第一个维度大小为 float 的大小

    const int ith = params->ith;
    const int nth = params->nth;

    GGML_TENSOR_UNARY_OP_LOCALS
    // 定义一些张量操作的本地变量

    float eps;
    memcpy(&eps, dst->op_params, sizeof(float));
    // 从目标张量的操作参数中复制 eps 的值

    // TODO: optimize
    // 待优化的部分
    # 遍历第三维度的索引
    for (int64_t i03 = 0; i03 < ne03; i03++) {
        # 遍历第二维度的索引
        for (int64_t i02 = 0; i02 < ne02; i02++) {
            # 遍历第一维度的索引，使用线程数作为步长
            for (int64_t i01 = ith; i01 < ne01; i01 += nth) {
                # 计算源数据的地址
                const float * x = (float *) ((char *) src0->data + i01*nb01 + i02*nb02 + i03*nb03);

                # 初始化求和变量
                ggml_float sum = 0.0;
                # 遍历第零维度的索引，计算求和
                for (int64_t i00 = 0; i00 < ne00; i00++) {
                    sum += (ggml_float)x[i00];
                }

                # 计算均值
                float mean = sum/ne00;

                # 计算目标数据的地址
                float * y = (float *) ((char *) dst->data + i01*nb1 + i02*nb2 + i03*nb3);

                # 初始化求和变量
                ggml_float sum2 = 0.0;
                # 遍历第零维度的索引，计算方差和标准化后的数据
                for (int64_t i00 = 0; i00 < ne00; i00++) {
                    # 计算偏差
                    float v = x[i00] - mean;
                    # 将偏差赋值给目标数据
                    y[i00] = v;
                    # 计算方差
                    sum2 += (ggml_float)(v*v);
                }

                # 计算方差
                float variance = sum2/ne00;
                # 计算标准化的缩放因子
                const float scale = 1.0f/sqrtf(variance + eps);

                # 对目标数据进行标准化
                ggml_vec_scale_f32(ne00, y, scale);
            }
        }
    }
# 计算前向归一化操作
static void ggml_compute_forward_norm(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    # 根据输入张量的类型进行不同的处理
    switch (src0->type) {
        # 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                # 调用 ggml_compute_forward_norm_f32 函数进行 F32 类型的前向归一化计算
                ggml_compute_forward_norm_f32(params, src0, dst);
            } break;
        # 如果输入张量类型为其他类型
        default:
            {
                # 断言错误，终止程序运行
                GGML_ASSERT(false);
            } break;
    }
}

# ggml_compute_forward_group_rms_norm

# 计算 F32 类型的前向 RMS 归一化操作
static void ggml_compute_forward_rms_norm_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    # 断言输入张量 src0 和目标张量 dst 的形状相同
    GGML_ASSERT(ggml_are_same_shape(src0, dst));

    # 如果参数类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    # 断言输入张量 src0 的第一个维度大小为 sizeof(float)
    GGML_ASSERT(src0->nb[0] == sizeof(float));

    # 获取参数中的 ith 和 nth
    const int ith = params->ith;
    const int nth = params->nth;

    # 定义局部变量
    GGML_TENSOR_UNARY_OP_LOCALS

    # 定义 eps 变量，并从目标张量的操作参数中复制出 eps 的值
    float eps;
    memcpy(&eps, dst->op_params, sizeof(float));

    # 循环计算前向 RMS 归一化操作
    for (int64_t i03 = 0; i03 < ne03; i03++) {
        for (int64_t i02 = 0; i02 < ne02; i02++) {
            for (int64_t i01 = ith; i01 < ne01; i01 += nth) {
                # 获取输入张量中的数据指针
                const float * x = (float *) ((char *) src0->data + i01*nb01 + i02*nb02 + i03*nb03);

                # 计算输入数据的平方和
                ggml_float sum = 0.0;
                for (int64_t i00 = 0; i00 < ne00; i00++) {
                    sum += (ggml_float)(x[i00] * x[i00]);
                }

                # 计算平均值
                const float mean = sum/ne00;

                # 获取目标张量中的数据指针
                float * y = (float *) ((char *) dst->data + i01*nb1 + i02*nb2 + i03*nb3);

                # 复制输入数据到目标数据
                memcpy(y, x, ne00 * sizeof(float));

                # 计算缩放因子
                const float scale = 1.0f/sqrtf(mean + eps);

                # 对目标数据进行缩放操作
                ggml_vec_scale_f32(ne00, y, scale);
            }
        }
    }
}
// 计算前向 RMS 归一化
static void ggml_compute_forward_rms_norm(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的处理
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用 ggml_compute_forward_rms_norm_f32 函数进行处理
                ggml_compute_forward_rms_norm_f32(params, src0, dst);
            } break;
        // 如果输入张量类型不在已知范围内
        default:
            {
                // 断言，抛出错误
                GGML_ASSERT(false);
            } break;
    }
}

// 计算反向 RMS 归一化（针对 F32 类型）
static void ggml_compute_forward_rms_norm_back_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 断言，检查输入张量的形状是否一致
    GGML_ASSERT(ggml_are_same_shape(src0, dst) && ggml_are_same_shape(src0, src1));

    // 如果任务类型为初始化或者结束，直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 断言，检查输入张量的数据类型是否为 float
    GGML_ASSERT(src0->nb[0] == sizeof(float));

    // 获取参数中的 ith 和 nth
    const int ith = params->ith;
    const int nth = params->nth;

    // 定义局部变量
    GGML_TENSOR_BINARY_OP_LOCALS

    // 定义变量 eps，并从 dst->op_params 中复制 sizeof(float) 大小的数据到 eps
    float eps;
    memcpy(&eps, dst->op_params, sizeof(float));

    // TODO: 优化
}

// 计算反向 RMS 归一化
static void ggml_compute_forward_rms_norm_back(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的处理
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用 ggml_compute_forward_rms_norm_back_f32 函数进行处理
                ggml_compute_forward_rms_norm_back_f32(params, src0, src1, dst);
            } break;
        // 如果输入张量类型不在已知范围内
        default:
            {
                // 断言，抛出错误
                GGML_ASSERT(false);
            } break;
    }
}

// 计算前向组归一化（针对 F32 类型）
static void ggml_compute_forward_group_norm_f32(
    const struct ggml_compute_params * params,
    const struct ggml_tensor * src0,
    struct ggml_tensor * dst) {
    // 断言，检查输入张量的形状是否一致
    GGML_ASSERT(ggml_are_same_shape(src0, dst));

    // 如果任务类型为初始化或者结束，直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 断言，检查输入张量的数据类型是否为 float
    GGML_ASSERT(src0->nb[0] == sizeof(float));

    // 获取参数中的 ith
    const int ith = params->ith;
    // 定义一个常量，表示参数中的第n个值
    const int nth = params->nth;

    // 定义一些本地变量，用于存储张量的一元操作
    GGML_TENSOR_UNARY_OP_LOCALS

    // 定义一个常量，表示一个很小的浮点数，用于计算机精度
    const float eps = 1e-6f; // TODO: make this a parameter

    // TODO: 优化代码

    // 计算源张量的第三维度的通道数
    int n_channels = src0->ne[2];
    // 获取目标张量的操作参数中的第一个值，表示分组数
    int n_groups = dst->op_params[0];
    // 计算每个分组中的通道数，向上取整
    int n_channels_per_group = (n_channels + n_groups - 1) / n_groups;
    # 循环遍历每个组，从第ith个开始，每次增加nth个
    for (int i = ith; i < n_groups; i+=nth) {
        # 计算每个组的起始通道和结束通道
        int start = i * n_channels_per_group;
        int end = start + n_channels_per_group;
        # 如果结束通道超过总通道数，则将结束通道设置为总通道数
        if (end > n_channels) {
            end = n_channels;
        }
        # 计算每个组的通道步长
        int step = end - start;

        # 遍历每个组内的数据
        for (int64_t i03 = 0; i03 < ne03; i03++) {
            # 初始化求和变量
            ggml_float sum = 0.0;
            # 遍历每个通道
            for (int64_t i02 = start; i02 < end; i02++) {
                # 遍历每个高度
                for (int64_t i01 = 0; i01 < ne01; i01++) {
                    # 计算源数据和目标数据的指针位置
                    const float * x = (float *)((char *) src0->data + i01 * nb01 + i02 * nb02 + i03 * nb03);

                    # 遍历每个宽度
                    for (int64_t i00 = 0; i00 < ne00; i00++) {
                        # 求和
                        sum += (ggml_float)x[i00];
                    }
                }
            }
            # 计算均值
            float mean = sum / (ne00 * ne01 * step);
            # 初始化求平方和的变量
            ggml_float sum2 = 0.0;

            # 遍历每个组内的数据
            for (int64_t i02 = start; i02 < end; i02++) {
                # 遍历每个高度
                for (int64_t i01 = 0; i01 < ne01; i01++) {
                    # 计算源数据和目标数据的指针位置
                    const float * x = (float *)((char *) src0->data + i01 * nb01 + i02 * nb02 + i03 * nb03);
                    float * y = (float *)((char *) dst->data + i01 * nb1 + i02 * nb2 + i03 * nb3);

                    # 遍历每个宽度
                    for (int64_t i00 = 0; i00 < ne00; i00++) {
                        # 计算偏差并赋值给目标数据
                        float v = x[i00] - mean;
                        y[i00] = v;
                        # 求平方和
                        sum2 += (ggml_float)(v * v);
                    }
                }
            }
            # 计算方差
            float variance = sum2 / (ne00 * ne01 * step);
            # 计算缩放比例
            const float scale = 1.0f / sqrtf(variance + eps);

            # 遍历每个组内的数据
            for (int64_t i02 = start; i02 < end; i02++) {
                # 遍历每个高度
                for (int64_t i01 = 0; i01 < ne01; i01++) {
                    # 计算目标数据的指针位置并进行缩放
                    float * y = (float *)((char *) dst->data + i01 * nb1 + i02 * nb2 + i03 * nb3);
                    ggml_vec_scale_f32(ne00, y, scale);
                }
            }
        }
    }
// 计算前向分组归一化的函数
static void ggml_compute_forward_group_norm(
    const struct ggml_compute_params * params,  // 参数结构体指针
    const struct ggml_tensor * src0,  // 输入张量指针
    struct ggml_tensor * dst) {  // 输出张量指针
    switch (src0->type) {  // 根据输入张量的类型进行判断
        case GGML_TYPE_F32:  // 如果类型为 GGML_TYPE_F32
            {
                ggml_compute_forward_group_norm_f32(params, src0, dst);  // 调用相应类型的前向分组归一化函数
            } break;
        default:  // 如果类型不匹配
            {
                GGML_ASSERT(false);  // 抛出断言错误
            } break;
    }
}

// ggml_compute_forward_mul_mat

#if defined(GGML_USE_ACCELERATE) || defined(GGML_USE_OPENBLAS)
// 用于确定是否使用 BLAS 更好的辅助函数
// 对于大矩阵，BLAS 更快
static bool ggml_compute_forward_mul_mat_use_blas(
        const struct ggml_tensor * src0,  // 输入张量指针
        const struct ggml_tensor * src1,  // 输入张量指针
              struct ggml_tensor * dst) {  // 输出张量指针
    //const int64_t ne00 = src0->ne[0];
    //const int64_t ne01 = src0->ne[1];

    const int64_t ne10 = src1->ne[0];  // 获取输入张量 src1 的维度

    const int64_t ne0 = dst->ne[0];  // 获取输出张量 dst 的维度
    const int64_t ne1 = dst->ne[1];  // 获取输出张量 dst 的维度

    // TODO: find the optimal values for these
    if (ggml_is_contiguous(src0) &&  // 判断输入张量 src0 是否是连续的
        ggml_is_contiguous(src1) &&  // 判断输入张量 src1 是否是连续的
        src0->type == GGML_TYPE_F32 &&  // 判断输入张量 src0 的类型是否为 GGML_TYPE_F32
        src1->type == GGML_TYPE_F32 &&  // 判断输入张量 src1 的类型是否为 GGML_TYPE_F32
        (ne0 >= 32 && ne1 >= 32 && ne10 >= 32)) {  // 判断输出张量 dst 和输入张量 src1 的维度是否大于等于 32
        /*printf("BLAS: %d %d %d %d %d\n", ne0, ne1, ne10, ne00, ne01);*/
        return true;  // 返回 true，表示使用 BLAS 更好
    }

    return false;  // 返回 false，表示不使用 BLAS 更好
}
#endif

static void ggml_compute_forward_mul_mat(
        const struct ggml_compute_params * params,  // 参数结构体指针
        const struct ggml_tensor * src0,  // 输入张量指针
        const struct ggml_tensor * src1,  // 输入张量指针
              struct ggml_tensor * dst) {  // 输出张量指针
    int64_t t0 = ggml_perf_time_us();  // 获取当前时间
    UNUSED(t0);  // 标记 t0 为未使用

    GGML_TENSOR_BINARY_OP_LOCALS  // 定义张量二元操作的本地变量

    const int ith = params->ith;  // 获取参数结构体中的 ith 值
    const int nth = params->nth;  // 获取参数结构体中的 nth 值

    const enum ggml_type type = src0->type;  // 获取输入张量 src0 的类型

    const bool src1_cont = ggml_is_contiguous(src1);  // 判断输入张量 src1 是否是连续的

    ggml_vec_dot_t    const vec_dot               = type_traits[type].vec_dot;  // 获取类型对应的向量点乘函数
    # 定义常量 vec_dot_type，其值为 type_traits[type].vec_dot_type
    enum ggml_type    const vec_dot_type          = type_traits[type].vec_dot_type;
    # 定义常量 from_float_to_vec_dot，其值为 type_traits[vec_dot_type].from_float
    ggml_from_float_t const from_float_to_vec_dot = type_traits[vec_dot_type].from_float;

    # 断言 ne0 等于 ne01
    GGML_ASSERT(ne0 == ne01);
    # 断言 ne1 等于 ne11
    GGML_ASSERT(ne1 == ne11);
    # 断言 ne2 等于 ne12
    GGML_ASSERT(ne2 == ne12);
    # 断言 ne3 等于 ne13
    GGML_ASSERT(ne3 == ne13);

    # 断言 nb00 等于 type 类型的大小
    GGML_ASSERT(nb00 == ggml_type_size(type));
    # 断言 nb10 等于 src1 的类型的大小
    GGML_ASSERT(nb10 == ggml_type_size(src1->type));

    # 断言 nb0 的大小为 sizeof(float)
    GGML_ASSERT(nb0 == sizeof(float));
    # 断言 nb0 小于等于 nb1
    GGML_ASSERT(nb0 <= nb1);
    # 断言 nb1 小于等于 nb2
    GGML_ASSERT(nb1 <= nb2);
    # 断言 nb2 小于等于 nb3
    GGML_ASSERT(nb2 <= nb3);

    # 计算广播因子 r2，其值为 ne12/ne02
    const int64_t r2 = ne12/ne02;
    # 计算广播因子 r3，其值为 ne13/ne03
    const int64_t r3 = ne13/ne03;

    # 断言 nb01 大于等于 nb00 - 表示 src0 未被转置
    #   通过 src0 的行来计算
# 如果定义了 GGML_USE_CLBLAST，则执行以下代码块
if (ggml_cl_can_mul_mat(src0, src1, dst)) {
    # 如果参数中的 ith 为 0 并且 type 为 GGML_TASK_COMPUTE，则执行以下代码块
    if (params->ith == 0 && params->type == GGML_TASK_COMPUTE) {
        # 调用 ggml_cl_mul_mat 函数，进行矩阵相乘操作
        ggml_cl_mul_mat(src0, src1, dst, params->wdata, params->wsize);
    }
    # 返回
    return;
}
# 如果定义了 GGML_USE_ACCELERATE 或 GGML_USE_OPENBLAS，则执行以下代码块
    // 如果使用 BLAS 计算前向乘法矩阵成功，则执行以下代码块
    if (ggml_compute_forward_mul_mat_use_blas(src0, src1, dst)) {
        // 如果参数中的 ith 不等于 0，则直接返回
        if (params->ith != 0) {
            return;
        }
        // 如果参数中的 type 为 GGML_TASK_INIT，则直接返回
        if (params->type == GGML_TASK_INIT) {
            return;
        }
        // 如果参数中的 type 为 GGML_TASK_FINALIZE，则直接返回
        if (params->type == GGML_TASK_FINALIZE) {
            return;
        }
        // 遍历循环，i13 从 0 到 ne13，i12 从 0 到 ne12
        for (int64_t i13 = 0; i13 < ne13; i13++) {
            for (int64_t i12 = 0; i12 < ne12; i12++) {
                // 将 src0 在第二维和第三维度上广播到 src1
                const int64_t i03 = i13/r3;
                const int64_t i02 = i12/r2;

                const void  * x = (char *)            src0->data + i02*nb02 + i03*nb03;
                const float * y = (float *) ((char *) src1->data + i12*nb12 + i13*nb13);

                float * d = (float *) ((char *) dst->data + i12*nb2 + i13*nb3);

                // 如果类型不是 GGML_TYPE_F32，则进行类型转换
                if (type != GGML_TYPE_F32) {
                    float * const wdata    = params->wdata;
                    ggml_to_float_t const to_float = type_traits[type].to_float;

                    size_t id = 0;
                    for (int64_t i01 = 0; i01 < ne01; ++i01) {
                        to_float((const char *) x + i01*nb01, wdata + id, ne00);
                        id += ne00;
                    }

                    assert(id*sizeof(float) <= params->wsize);
                    x = wdata;
                }

                // 使用 BLAS 库中的 cblas_sgemm 函数进行矩阵乘法运算
                cblas_sgemm(CblasRowMajor, CblasNoTrans, CblasTrans,
                        ne11, ne01, ne10,
                        1.0f,    y, ne10,
                                 x, ne00,
                        0.0f,    d, ne01);
            }
        }

        // 打印 CBLAS 运行时间和矩阵维度信息
        //printf("CBLAS = %f ms, %d x %d x %d x %d\n", (ggml_perf_time_us() - t0)/1000.0, ne0, ne1, ne2, ne3);

        // 返回
        return;
    }
#endif

    // 如果任务类型是初始化
    if (params->type == GGML_TASK_INIT) {
        // 如果 src1 的类型不是 vec_dot_type
        if (src1->type != vec_dot_type) {
            // 获取参数中的 wdata
            char * wdata = params->wdata;
            // 计算每行数据的大小
            const size_t row_size = ne10*ggml_type_size(vec_dot_type)/ggml_blck_size(vec_dot_type);

            // 循环遍历数据
            for (int64_t i13 = 0; i13 < ne13; ++i13) {
                for (int64_t i12 = 0; i12 < ne12; ++i12) {
                    for (int64_t i11 = 0; i11 < ne11; ++i11) {
                        // 将 float 类型转换为 vec_dot 类型
                        from_float_to_vec_dot((float *)((char *) src1->data + i13*nb13 + i12*nb12 + i11*nb11), (void *) wdata, ne10);
                        wdata += row_size;
                    }
                }
            }
        }

        // 返回
        return;
    }

    // 如果任务类型是结束
    if (params->type == GGML_TASK_FINALIZE) {
        // 返回
        return;
    }

    // 获取 wdata
    const void * wdata    = (src1->type == vec_dot_type) ? src1->data : params->wdata;
    // 计算每行数据的大小
    const size_t row_size = ne10*ggml_type_size(vec_dot_type)/ggml_blck_size(vec_dot_type);

    // 获取 src0 和 src1 的行数
    const int64_t nr0 = ne01;           // src0 rows
    const int64_t nr1 = ne11*ne12*ne13; // src1 rows

    // 并行化处理，根据行数多少选择内循环或外循环
    const int64_t nth0 = nr0 > nr1 ? nth : 1; // parallelize by src0 rows
    const int64_t nth1 = nr0 > nr1 ? 1 : nth; // parallelize by src1 rows

    // 计算当前线程在 nth0 和 nth1 中的位置
    const int64_t ith0 = ith % nth0;
    const int64_t ith1 = ith / nth0;

    // 计算每个线程处理的行数
    const int64_t dr0 = (nr0 + nth0 - 1)/nth0;
    const int64_t dr1 = (nr1 + nth1 - 1)/nth1;

    // 计算当前线程处理的行数范围
    const int64_t ir010 = dr0*ith0;
    const int64_t ir011 = MIN(ir010 + dr0, nr0);

    const int64_t ir110 = dr1*ith1;
    const int64_t ir111 = MIN(ir110 + dr1, nr1);

    // 如果当前线程没有工作，则让出 CPU
    if (ir010 >= ir011 || ir110 >= ir111) {
        sched_yield();
        return;
    }

    // 断言 ne12 能被 ne02 整除，ne13 能被 ne03 整除
    assert(ne12 % ne02 == 0);
    assert(ne13 % ne03 == 0);
    // 尝试使用块瓦片化
    const int64_t blck_0 = 16;  // 定义块大小的常量，第一维度为16
    const int64_t blck_1 = 16;  // 定义块大小的常量，第二维度为16
    
    // 尝试减少伪共享（似乎没有任何区别）
    float tmp[16];  // 创建一个长度为16的浮点数数组，用于临时存储数据
// ggml_compute_forward_out_prod

static void ggml_compute_forward_out_prod_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
              struct ggml_tensor * dst) {
    // int64_t t0 = ggml_perf_time_us();
    // UNUSED(t0);

    GGML_TENSOR_BINARY_OP_LOCALS

    const int ith = params->ith;  // 获取当前线程的索引
    const int nth = params->nth;  // 获取线程总数

    GGML_ASSERT(ne02 == ne12);  // 断言确保张量维度匹配
    GGML_ASSERT(ne03 == ne13);  // 断言确保张量维度匹配
    GGML_ASSERT(ne2  == ne12);  // 断言确保张量维度匹配
    GGML_ASSERT(ne3  == ne13);  // 断言确保张量维度匹配

    // we don't support permuted src0 or src1
    GGML_ASSERT(nb00 == sizeof(float));  // 断言确保数据类型为 float

    // dst cannot be transposed or permuted
    GGML_ASSERT(nb0 == sizeof(float));  // 断言确保数据类型为 float
    // GGML_ASSERT(nb0 <= nb1);
    // GGML_ASSERT(nb1 <= nb2);
    // GGML_ASSERT(nb2 <= nb3);

    GGML_ASSERT(ne0 == ne00);  // 断言确保张量维度匹配
    GGML_ASSERT(ne1 == ne10);  // 断言确保张量维度匹配
    GGML_ASSERT(ne2 == ne02);  // 断言确保张量维度匹配
    GGML_ASSERT(ne3 == ne03);  // 断言确保张量维度匹配

    // nb01 >= nb00 - src0 is not transposed
    //   compute by src0 rows

    // TODO: #if defined(GGML_USE_CUBLAS) ggml_cuda_out_prod
    // TODO: #if defined(GGML_USE_ACCELERATE) || defined(GGML_USE_OPENBLAS) || defined(GGML_USE_CLBLAST)

    if (params->type == GGML_TASK_INIT) {
        ggml_vec_set_f32(ne0*ne1*ne2*ne3, dst->data, 0);  // 初始化 dst 数据为 0
        return;
    }

    if (params->type == GGML_TASK_FINALIZE) {
        return;  // 结束函数
    }

    // dst[:,:,:,:] = 0
    // for i2,i3:
    //   for i1:
    //     for i01:
    //       for i0:
    //         dst[i0,i1,i2,i3] += src0[i0,i01,i2,i3] * src1[i1,i01,i2,i3]

    // parallelize by last three dimensions

    // total rows in dst
    const int64_t nr = ne1*ne2*ne3;  // 计算 dst 的总行数

    // rows per thread
    const int64_t dr = (nr + nth - 1)/nth;  // 每个线程处理的行数

    // row range for this thread
    const int64_t ir0 = dr*ith;  // 当前线程处理的起始行索引
    const int64_t ir1 = MIN(ir0 + dr, nr);  // 当前线程处理的结束行索引

    // block-tiling attempt
    const int64_t blck_0 = MAX(GGML_VEC_MAD_UNROLL, 32);  // 计算块大小
    const int64_t blck_1 = 16;  // 计算块大小
    for (int64_t bir = ir0; bir < ir1; bir += blck_1) {
        // 循环遍历ir0到ir1之间的值，每次增加blck_1
        const int64_t bir1 = MIN(bir + blck_1, ir1);
        // 计算bir + blck_1和ir1的最小值
        for (int64_t bi01 = 0; bi01 < ne01; bi01 += blck_0) {
            // 循环遍历0到ne01之间的值，每次增加blck_0
            const int64_t bne01 = MIN(bi01 + blck_0, ne01);
            // 计算bi01 + blck_0和ne01的最小值
            for (int64_t ir = bir; ir < bir1; ++ir) {
                // 循环遍历bir到bir1之间的值
                // dst indices
                // 计算i3, i2, i1的值
                const int64_t i3 = ir/(ne2*ne1);
                const int64_t i2 = (ir - i3*ne2*ne1)/ne1;
                const int64_t i1 = (ir - i3*ne2*ne1 - i2*ne1);

                const int64_t i02 = i2;
                const int64_t i03 = i3;

                //const int64_t i10 = i1;
                const int64_t i12 = i2;
                const int64_t i13 = i3;
                // 计算i12, i13的值
#if GGML_VEC_MAD_UNROLL > 2
                // 如果 GGML_VEC_MAD_UNROLL 大于 2，则执行以下代码块
                const int64_t bne01_unroll = bne01 - (bne01 % GGML_VEC_MAD_UNROLL);
                // 计算 bne01_unroll，使其为 GGML_VEC_MAD_UNROLL 的整数倍
                for (int64_t i01 = bi01; i01 < bne01_unroll; i01 += GGML_VEC_MAD_UNROLL) {
                    // 循环遍历 i01，每次增加 GGML_VEC_MAD_UNROLL
                    const int64_t i11 = i01;

                    float * s0 = (float *) ((char *) src0->data + (          i01*nb01 + i02*nb02 + i03*nb03));
                    float * s1 = (float *) ((char *) src1->data + (i1*nb10 + i11*nb11 + i12*nb12 + i13*nb13));
                    float * d  = (float *) ((char *)  dst->data + (          i1*nb1 + i2*nb2 + i3*nb3));

                    ggml_vec_mad_f32_unroll(ne0, nb01, nb11, d, s0, s1);
                    // 调用 ggml_vec_mad_f32_unroll 函数
                }
                for (int64_t i01 = bne01_unroll; i01 < bne01; ++i01) {
                    // 循环遍历 i01，直到 bne01
                    const int64_t i11 = i01;

                    float * s0 = (float *) ((char *) src0->data + (          i01*nb01 + i02*nb02 + i03*nb03));
                    float * s1 = (float *) ((char *) src1->data + (i1*nb10 + i11*nb11 + i12*nb12 + i13*nb13));
                    float * d  = (float *) ((char *)  dst->data + (          i1*nb1 + i2*nb2 + i3*nb3));

                    ggml_vec_mad_f32(ne0, d, s0, *s1);
                    // 调用 ggml_vec_mad_f32 函数
                }
#else
                // 如果 GGML_VEC_MAD_UNROLL 不大于 2，则执行以下代码块
                for (int64_t i01 = bi01; i01 < bne01; ++i01) {
                    // 循环遍历 i01，直到 bne01
                    const int64_t i11 = i01;

                    float * s0 = (float *) ((char *) src0->data + (          i01*nb01 + i02*nb02 + i03*nb03));
                    float * s1 = (float *) ((char *) src1->data + (i1*nb10 + i11*nb11 + i12*nb12 + i13*nb13));
                    float * d  = (float *) ((char *)  dst->data + (          i1*nb1 + i2*nb2 + i3*nb3));

                    ggml_vec_mad_f32(ne0, d, s0, *s1);
                    // 调用 ggml_vec_mad_f32 函数
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

    // 打印任务编号、总任务数、执行时间、累积值
    // printf("XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX task %d/%d: %d us, acc = %d\n", ith, nth, (int) (t1 - t0), (int) acc);
    //}
static void ggml_compute_forward_out_prod_q_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
              struct ggml_tensor * dst) {
    // 定义一个静态函数，计算两个输入张量的张量积，并将结果存储在目标张量中

    // 定义一些本地变量
    GGML_TENSOR_BINARY_OP_LOCALS;

    // 获取参数中的线程索引和线程总数
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取输入张量的数据类型
    const enum ggml_type type = src0->type;
    ggml_to_float_t const dequantize_row_q = type_traits[type].to_float;

    // 断言一些维度的相等性
    GGML_ASSERT(ne02 == ne12);
    GGML_ASSERT(ne03 == ne13);
    GGML_ASSERT(ne2  == ne12);
    GGML_ASSERT(ne3  == ne13);

    // 断言不支持对输入张量的第一个维度进行置换
    GGML_ASSERT(nb00 == ggml_type_size(type));

    // 断言目标张量的第一个维度不能被转置或置换
    GGML_ASSERT(nb0 == sizeof(float));

    // 断言一些维度的大小关系
    // GGML_ASSERT(nb0 <= nb1);
    // GGML_ASSERT(nb1 <= nb2);
    // GGML_ASSERT(nb2 <= nb3);

    GGML_ASSERT(ne0 == ne00);
    GGML_ASSERT(ne1 == ne10);
    GGML_ASSERT(ne2 == ne02);
    GGML_ASSERT(ne3 == ne03);

    // nb01 >= nb00 - src0 is not transposed
    //   compute by src0 rows

    // TODO: #if defined(GGML_USE_CUBLAS) ggml_cuda_out_prod
    // TODO: #if defined(GGML_USE_ACCELERATE) || defined(GGML_USE_OPENBLAS) || defined(GGML_USE_CLBLAST)

    // 如果任务类型是初始化，则将目标张量的数据全部设置为0，并返回
    if (params->type == GGML_TASK_INIT) {
        ggml_vec_set_f32(ne0*ne1*ne2*ne3, dst->data, 0);
        return;
    }

    // 如果任务类型是最终化，则直接返回
    if (params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 并行化最后三个维度

    // 目标张量中的总行数
    const int64_t nr = ne1*ne2*ne3;

    // 每个线程处理的行数
    const int64_t dr = (nr + nth - 1)/nth;

    // 当前线程处理的行范围
    const int64_t ir0 = dr*ith;
    const int64_t ir1 = MIN(ir0 + dr, nr);

    // 计算目标张量的值
    // dst[:,:,:,:] = 0
    // for i2,i3:
    //   for i1:
    //     for i01:
    //       for i0:
    //         dst[i0,i1,i2,i3] += src0[i0,i01,i2,i3] * src1[i1,i01,i2,i3]
}
    // 指针 wdata 指向 params->wdata 数组中的特定位置
    float * wdata = (float *) params->wdata + (ne0 + CACHE_LINE_SIZE_F32) * ith;

    // 遍历 ir0 到 ir1 之间的整数
    for (int64_t ir = ir0; ir < ir1; ++ir) {
        // 计算 dst 索引的三个维度
        const int64_t i3 = ir/(ne2*ne1);
        const int64_t i2 = (ir - i3*ne2*ne1)/ne1;
        const int64_t i1 = (ir - i3*ne2*ne1 - i2*ne1);

        const int64_t i02 = i2;
        const int64_t i03 = i3;

        //const int64_t i10 = i1;
        const int64_t i12 = i2;
        const int64_t i13 = i3;

        // 遍历 ne01 个整数
        for (int64_t i01 = 0; i01 < ne01; ++i01) {
            const int64_t i11 = i01;

            // 计算指向 src0、src1 和 dst 数组特定位置的指针
            float * s0 = (float *) ((char *) src0->data + (          i01*nb01 + i02*nb02 + i03*nb03));
            float * s1 = (float *) ((char *) src1->data + (i1*nb10 + i11*nb11 + i12*nb12 + i13*nb13));
            float * d  = (float *) ((char *)  dst->data + (          i1*nb1 + i2*nb2 + i3*nb3));

            // 对 s0 进行反量化
            dequantize_row_q(s0, wdata, ne0);
            // 使用 s1 和 wdata 进行向量乘加操作
            ggml_vec_mad_f32(ne0, d, wdata, *s1);
        }
    }

    // 下面是一段被注释掉的代码，暂时不需要解释
// 计算两个张量的乘积，并将结果存储到目标张量中
static void ggml_compute_forward_out_prod(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 根据第一个源张量的类型进行不同的处理
    switch (src0->type) {
        // 如果是以下类型之一，调用对应的函数进行计算
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
            {
                ggml_compute_forward_out_prod_q_f32(params, src0, src1, dst);
            } break;
        // 如果是 F16 类型，暂时断言失败，待实现
        case GGML_TYPE_F16:
            {
                GGML_ASSERT(false); // todo
                // ggml_compute_forward_out_prod_f16_f32(params, src0, src1, dst);
            } break;
        // 如果是 F32 类型，调用对应的函数进行计算
        case GGML_TYPE_F32:
            {
                ggml_compute_forward_out_prod_f32(params, src0, src1, dst);
            } break;
        // 其他类型断言失败
        default:
            {
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_scale

// 计算两个张量的缩放，并将结果存储到目标张量中
static void ggml_compute_forward_scale_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 断言源张量和目标张量是连续的
    GGML_ASSERT(ggml_is_contiguous(src0));
    GGML_ASSERT(ggml_is_contiguous(dst));
    // 断言源张量和目标张量具有相同的形状
    GGML_ASSERT(ggml_are_same_shape(src0, dst));
    // 断言源张量 src1 是标量
    GGML_ASSERT(ggml_is_scalar(src1));

    // 如果任务类型是初始化或者结束，直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 缩放因子
    const float v = *(float *) src1->data;

    const int ith = params->ith;
    const int nth = params->nth;

    const int nc = src0->ne[0];
    const int nr = ggml_nrows(src0);

    // 每个线程的行数
    const int dr = (nr + nth - 1)/nth;

    // 本线程的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    const size_t nb01 = src0->nb[1];
    // 获取 dst 结构体中 nb 数组的第二个元素的大小
    const size_t nb1 = dst->nb[1];

    // 遍历 ir0 到 ir1 之间的整数
    for (int i1 = ir0; i1 < ir1; i1++) {
        // 如果 dst 的数据指针不等于 src0 的数据指针
        if (dst->data != src0->data) {
            // 将 src0 数据指针中 i1*nb01 处开始的 nc 个 float 数据拷贝到 dst 数据指针中 i1*nb1 处
            memcpy((char *)dst->data + i1*nb1, (char *)src0->data + i1*nb01, nc * sizeof(float));
        }
        // 对 dst 数据指针中 i1*nb1 处开始的 nc 个 float 数据进行缩放，缩放因子为 v
        ggml_vec_scale_f32(nc, (float *) ((char *) dst->data + i1*nb1), v);
    }
// 计算前向缩放操作，根据输入参数和张量进行计算，将结果存储到目标张量中
static void ggml_compute_forward_scale(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 根据 src0 的数据类型进行不同的操作
    switch (src0->type) {
        // 如果数据类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用 ggml_compute_forward_scale_f32 函数进行计算
                ggml_compute_forward_scale_f32(params, src0, src1, dst);
            } break;
        // 如果数据类型不是 GGML_TYPE_F32
        default:
            {
                // 断言，表示出现了不应该出现的情况
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_set

// 根据输入参数和张量进行前向设置操作，将结果存储到目标张量中
static void ggml_compute_forward_set_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 断言，确保 src0 和 dst 的形状相同
    GGML_ASSERT(ggml_are_same_shape(src0, dst));
    // 断言，确保 src0 和 dst 是连续的
    GGML_ASSERT(ggml_is_contiguous(dst) && ggml_is_contiguous(src0));

    // 在设置期间使用这些步幅和数据偏移来查看 src0 和 dst
    // nb0 隐式地是 element_size，因为 src0 和 dst 是连续的
    size_t nb1     = ((int32_t *) dst->op_params)[0];
    size_t nb2     = ((int32_t *) dst->op_params)[1];
    size_t nb3     = ((int32_t *) dst->op_params)[2];
    size_t offset  = ((int32_t *) dst->op_params)[3];
    bool   inplace = (bool) ((int32_t *) dst->op_params)[4];

    // 如果不是原地操作且类型为 GGML_TASK_INIT
    if (!inplace && (params->type == GGML_TASK_INIT)) {
        // 需要在不同线程之间同步 memcpy，以避免竞争条件
        // => 在初始化阶段执行
        memcpy(
            ((char *)  dst->data),
            ((char *) src0->data),
            ggml_nbytes(dst));
    }

    // 如果类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取当前线程编号和线程总数
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取 src1 的行数和列数
    const int nr = ggml_nrows(src1);
    const int nc = src1->ne[0];

    // 定义 ne1 和 nb1 的本地变量
    GGML_TENSOR_LOCALS(int64_t, ne1, src1, ne)
    GGML_TENSOR_LOCALS(size_t,  nb1, src1, nb)

    // 在设置期间查看 src0 和 dst 时的 nb0
    const size_t nb0 = ggml_element_size(src0);
}
    // 计算 im0，如果 ne10 为 0 则为 0，否则为 ne10-1
    const int im0 = (ne10 == 0 ? 0 : ne10-1);
    // 计算 im1，如果 ne11 为 0 则为 0，否则为 ne11-1
    const int im1 = (ne11 == 0 ? 0 : ne11-1);
    // 计算 im2，如果 ne12 为 0 则为 0，否则为 ne12-1
    const int im2 = (ne12 == 0 ? 0 : ne12-1);
    // 计算 im3，如果 ne13 为 0 则为 0，否则为 ne13-1
    const int im3 = (ne13 == 0 ? 0 : ne13-1);

    // 断言，确保偏移量加上各维度索引乘以对应的步长不超过目标内存的字节数
    GGML_ASSERT(offset + im0*nb0  + im1*nb1  + im2*nb2  + im3*nb3  <= ggml_nbytes(dst));

    // 断言，确保 nb10 的大小为 float 类型的大小
    GGML_ASSERT(nb10 == sizeof(float));

    // 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 本线程处理的行范围
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
// 计算前向传播的设置，根据不同的输入类型进行不同的处理
static void ggml_compute_forward_set(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {

    // 根据输入张量的类型进行不同的处理
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用相应的处理函数
                ggml_compute_forward_set_f32(params, src0, src1, dst);
            } break;
        // 如果输入张量类型为其他类型
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
        default:
            {
                // 抛出断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_cpy

// 复制输入张量到输出张量
static void ggml_compute_forward_cpy(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    ggml_compute_forward_dup(params, src0, dst);
}

// ggml_compute_forward_cont

// 复制输入张量到输出张量
static void ggml_compute_forward_cont(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    ggml_compute_forward_dup(params, src0, dst);
}

// ggml_compute_forward_reshape

// 对输入张量进行重塑操作
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

// 对输入张量进行视图操作
static void ggml_compute_forward_view(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0) {
    // 空操作
    UNUSED(params);
    UNUSED(src0);
}

// ggml_compute_forward_permute

// 对输入张量进行排列操作
static void ggml_compute_forward_permute(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0) {
    // 空操作
    UNUSED(params);
}
    # 忽略未使用的变量 src0
    UNUSED(src0);
// ggml_compute_forward_transpose

// 计算前向传播的转置操作，参数为计算参数和源张量
static void ggml_compute_forward_transpose(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0) {
    // 空操作
    UNUSED(params);
    UNUSED(src0);
}

// ggml_compute_forward_get_rows

// 计算前向传播的获取行操作，参数为计算参数、两个源张量和目标张量
static void ggml_compute_forward_get_rows_q(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
              struct ggml_tensor * dst) {
    // 断言参数的ith属性为0
    assert(params->ith == 0);

    // 如果参数的类型为初始化或者结束，则返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取源张量src0的第一个维度大小和源张量src1的元素个数
    const int nc = src0->ne[0];
    const int nr = ggml_nelements(src1);
    // 获取源张量src0的数据类型
    const enum ggml_type type = src0->type;
    // 获取对应数据类型的转换函数
    ggml_to_float_t const dequantize_row_q = type_traits[type].to_float;

    // 断言目标张量的第一个维度大小为nc
    assert( dst->ne[0] == nc);
    // 断言目标张量的第二个维度大小为nr
    assert( dst->ne[1] == nr);
    // 断言源张量src0的第一个维度字节数等于对应数据类型的大小
    assert(src0->nb[0] == ggml_type_size(type));

    // 遍历nr次
    for (int i = 0; i < nr; ++i) {
        // 获取src1中第i个元素作为行索引
        const int r = ((int32_t *) src1->data)[i];

        // 调用转换函数，将src0中第r行的数据转换为float类型，存入dst中的第i行
        dequantize_row_q(
                (const void *) ((char *) src0->data + r*src0->nb[1]),
                     (float *) ((char *)  dst->data + i*dst->nb[1]), nc);
    }
}

// 计算前向传播的获取行操作，参数为计算参数、两个源张量和目标张量
static void ggml_compute_forward_get_rows_f16(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
              struct ggml_tensor * dst) {
    // 断言参数的ith属性为0
    assert(params->ith == 0);

    // 如果参数的类型为初始化或者结束，则返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取源张量src0的第一个维度大小和源张量src1的元素个数
    const int nc = src0->ne[0];
    const int nr = ggml_nelements(src1);

    // 断言目标张量的第一个维度大小为nc
    assert( dst->ne[0] == nc);
    // 断言目标张量的第二个维度大小为nr
    assert( dst->ne[1] == nr);
    // 断言源张量src0的第一个维度字节数等于ggml_fp16_t类型的大小
    assert(src0->nb[0] == sizeof(ggml_fp16_t));
    # 遍历行数范围内的索引
    for (int i = 0; i < nr; ++i) {
        # 从第一个输入数据中获取第 i 个元素的数值
        const int r = ((int32_t *) src1->data)[i];

        # 遍历列数范围内的索引
        for (int j = 0; j < nc; ++j) {
            # 从第二个输入数据中获取第 r 行 j 列的元素的数值
            ggml_fp16_t v = ((ggml_fp16_t *) ((char *) src0->data + r*src0->nb[1]))[j];
            # 将获取的数值转换为 32 位浮点数，并存储到目标数据中的第 i 行 j 列
            ((float *) ((char *)  dst->data + i*dst->nb[1]))[j] = GGML_FP16_TO_FP32(v);
        }
    }
// 计算前向传播的函数，获取指定行的数据，数据类型为单精度浮点数
static void ggml_compute_forward_get_rows_f32(
        const struct ggml_compute_params * params,  // 传入的计算参数结构体指针
        const struct ggml_tensor * src0,  // 源张量1
        const struct ggml_tensor * src1,  // 源张量2
              struct ggml_tensor * dst) {  // 目标张量

    assert(params->ith == 0);  // 断言，确保参数中的 ith 值为 0

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  // 如果参数中的类型为初始化或者结束任务，则返回
        return;
    }

    const int nc = src0->ne[0];  // 获取源张量1的第一个维度大小
    const int nr = ggml_nelements(src1);  // 获取源张量2的元素个数

    assert( dst->ne[0] == nc);  // 断言，确保目标张量的第一个维度大小与源张量1相同
    assert( dst->ne[1] == nr);  // 断言，确保目标张量的第二个维度大小与源张量2相同
    assert(src0->nb[0] == sizeof(float));  // 断言，确保源张量1的数据类型为单精度浮点数

    for (int i = 0; i < nr; ++i) {  // 遍历源张量2的元素个数
        const int r = ((int32_t *) src1->data)[i];  // 获取源张量2中第 i 个元素的值，并转换为 int 类型

        ggml_vec_cpy_f32(nc,  // 调用函数，将源张量1中的指定行数据复制到目标张量中
                (float *) ((char *)  dst->data + i*dst->nb[1]),  // 目标张量中的起始位置
                (float *) ((char *) src0->data + r*src0->nb[1]));  // 源张量1中的指定行数据的起始位置
    }
}

static void ggml_compute_forward_get_rows(  // 计算前向传播的函数，获取指定行的数据
        const struct ggml_compute_params * params,  // 传入的计算参数结构体指针
        const struct ggml_tensor * src0,  // 源张量1
        const struct ggml_tensor * src1,  // 源张量2
        struct ggml_tensor * dst) {  // 目标张量

    switch (src0->type) {  // 根据源张量1的数据类型进行切换
        case GGML_TYPE_Q4_0:  // 如果数据类型为 Q4_0
        case GGML_TYPE_Q4_1:  // 如果数据类型为 Q4_1
        case GGML_TYPE_Q5_0:  // 如果数据类型为 Q5_0
        case GGML_TYPE_Q5_1:  // 如果数据类型为 Q5_1
        case GGML_TYPE_Q8_0:  // 如果数据类型为 Q8_0
        case GGML_TYPE_Q8_1:  // 如果数据类型为 Q8_1
        case GGML_TYPE_Q2_K:  // 如果数据类型为 Q2_K
        case GGML_TYPE_Q3_K:  // 如果数据类型为 Q3_K
        case GGML_TYPE_Q4_K:  // 如果数据类型为 Q4_K
        case GGML_TYPE_Q5_K:  // 如果数据类型为 Q5_K
        case GGML_TYPE_Q6_K:  // 如果数据类型为 Q6_K
            {
                ggml_compute_forward_get_rows_q(params, src0, src1, dst);  // 调用相应的函数处理
            } break;
        case GGML_TYPE_F16:  // 如果数据类型为 F16
            {
                ggml_compute_forward_get_rows_f16(params, src0, src1, dst);  // 调用相应的函数处理
            } break;
        case GGML_TYPE_F32:  // 如果数据类型为 F32
            {
                ggml_compute_forward_get_rows_f32(params, src0, src1, dst);  // 调用相应的函数处理
            } break;
        default:  // 如果数据类型为其他类型
            {
                GGML_ASSERT(false);  // 断言，抛出错误
            } break;
    }

    //static bool first = true;
    //printf("ne0 = %d, ne1 = %d, ne2 = %d\n", dst->ne[0], dst->ne[1], dst->ne[2]);
    //if (first) {
    //    first = false;
    // 如果条件不成立，执行以下代码块
    // 遍历 dst 结构体中的数据
    // 第一层循环，遍历 dst 结构体中的第二维数据
    // 第二层循环，遍历 dst 结构体中的第一维数据，每次遍历16个元素
    // 第三层循环，遍历16个元素，并打印出每个元素的值
    // 换行
    // 换行
    // 打印空行
    // 退出程序
// ggml_compute_forward_get_rows_back 函数的静态实现，用于计算前向传播时获取行数据并返回到指定位置
static void ggml_compute_forward_get_rows_back_f32_f16(
        const struct ggml_compute_params * params,  // 传入的计算参数结构体指针
        const struct ggml_tensor * src0,  // 源张量1的指针
        const struct ggml_tensor * src1,  // 源张量2的指针
              struct ggml_tensor * dst) {  // 目标张量的指针
    GGML_ASSERT(params->ith == 0);  // 断言，判断参数结构体中的 ith 是否为 0
    GGML_ASSERT(ggml_is_contiguous(dst));  // 断言，判断目标张量是否是连续的

    // ggml_compute_forward_dup_same_cont(params, opt0, dst);  // 调用其他函数，暂时注释掉

    if (params->type == GGML_TASK_INIT) {  // 如果参数结构体中的 type 为 GGML_TASK_INIT
        memset(dst->data, 0, ggml_nbytes(dst));  // 将目标张量的数据全部置为 0
    }

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  // 如果参数结构体中的 type 为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE
        return;  // 直接返回，不执行后续代码
    }

    const int nc = src0->ne[0];  // 获取源张量1的第一个维度大小
    const int nr = ggml_nelements(src1);  // 获取源张量2的元素个数

    GGML_ASSERT( dst->ne[0] == nc);  // 断言，判断目标张量的第一个维度大小是否等于 nc
    GGML_ASSERT(src0->nb[0] == sizeof(ggml_fp16_t));  // 断言，判断源张量1的数据类型大小是否等于 ggml_fp16_t 的大小

    for (int i = 0; i < nr; ++i) {  // 遍历 nr 次
        const int r = ((int32_t *) src1->data)[i];  // 获取源张量2中第 i 个元素的值，转换为 int 类型

        for (int j = 0; j < nc; ++j) {  // 遍历 nc 次
            ggml_fp16_t v = ((ggml_fp16_t *) ((char *) src0->data + i*src0->nb[1]))[j];  // 获取源张量1中指定位置的值，转换为 ggml_fp16_t 类型
            ((float *) ((char *) dst->data + r*dst->nb[1]))[j] += GGML_FP16_TO_FP32(v);  // 将计算结果加到目标张量的指定位置
        }
    }
}

// ggml_compute_forward_get_rows_back 函数的静态实现，用于计算前向传播时获取行数据并返回到指定位置
static void ggml_compute_forward_get_rows_back_f32(
        const struct ggml_compute_params * params,  // 传入的计算参数结构体指针
        const struct ggml_tensor * src0,  // 源张量1的指针
        const struct ggml_tensor * src1,  // 源张量2的指针
              struct ggml_tensor * dst) {  // 目标张量的指针
    GGML_ASSERT(params->ith == 0);  // 断言，判断参数结构体中的 ith 是否为 0
    GGML_ASSERT(ggml_is_contiguous(dst));  // 断言，判断目标张量是否是连续的

    // ggml_compute_forward_dup_same_cont(params, opt0, dst);  // 调用其他函数，暂时注释掉

    if (params->type == GGML_TASK_INIT) {  // 如果参数结构体中的 type 为 GGML_TASK_INIT
        memset(dst->data, 0, ggml_nbytes(dst));  // 将目标张量的数据全部置为 0
    }

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  // 如果参数结构体中的 type 为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE
        return;  // 直接返回，不执行后续代码
    }

    const int nc = src0->ne[0];  // 获取源张量1的第一个维度大小
    const int nr = ggml_nelements(src1);  // 获取源张量2的元素个数

    GGML_ASSERT( dst->ne[0] == nc);  // 断言，判断目标张量的第一个维度大小是否等于 nc
    GGML_ASSERT(src0->nb[0] == sizeof(float));  // 断言，判断源张量1的数据类型大小是否等于 float 的大小
    # 遍历循环，从 0 到 nr-1
    for (int i = 0; i < nr; ++i) {
        # 从 src1 数据中读取第 i 个元素，转换为 int 类型
        const int r = ((int32_t *) src1->data)[i];

        # 调用 ggml_vec_add_f32 函数，对 nc 中的数据进行操作
        ggml_vec_add_f32(nc,
                # 将 dst 数据中的指定位置转换为 float 指针
                (float *) ((char *)  dst->data + r*dst->nb[1]),
                # 将 dst 数据中的指定位置转换为 float 指针
                (float *) ((char *)  dst->data + r*dst->nb[1]),
                # 将 src0 数据中的指定位置转换为 float 指针
                (float *) ((char *) src0->data + i*src0->nb[1]));
    }
// 根据给定参数计算前向传播的行数，返回结果到目标张量
static void ggml_compute_forward_get_rows_back(
        const struct ggml_compute_params * params,  // 前向传播计算参数
        const struct ggml_tensor * src0,  // 输入张量1
        const struct ggml_tensor * src1,  // 输入张量2
        struct ggml_tensor * dst) {  // 输出张量
    switch (src0->type) {  // 根据输入张量1的类型进行判断
        case GGML_TYPE_F16:  // 如果类型为 F16
            {
                ggml_compute_forward_get_rows_back_f32_f16(params, src0, src1, dst);  // 调用相应的计算函数
            } break;
        case GGML_TYPE_F32:  // 如果类型为 F32
            {
                ggml_compute_forward_get_rows_back_f32(params, src0, src1, dst);  // 调用相应的计算函数
            } break;
        default:  // 其他情况
            {
                GGML_ASSERT(false);  // 断言错误
            } break;
    }

    // 下面是一段注释掉的代码，暂时不执行
}

// ggml_compute_forward_diag

// 根据给定参数计算前向传播的对角线，返回结果到目标张量
static void ggml_compute_forward_diag_f32(
        const struct ggml_compute_params * params,  // 前向传播计算参数
        const struct ggml_tensor * src0,  // 输入张量
        struct ggml_tensor * dst) {  // 输出张量
    GGML_ASSERT(params->ith == 0);  // 断言参数的 ith 属性为 0

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  // 如果参数的类型为初始化或者结束
        return;  // 直接返回
    }

    // TODO: handle transposed/permuted matrices  // 处理转置/置换矩阵的情况

    GGML_TENSOR_UNARY_OP_LOCALS  // 定义一些张量操作的本地变量

    GGML_ASSERT(ne00 == ne0);  // 断言 ne00 等于 ne0
    GGML_ASSERT(ne00 == ne1);  // 断言 ne00 等于 ne1
    GGML_ASSERT(ne01 == 1);  // 断言 ne01 等于 1
    GGML_ASSERT(ne02 == ne2);  // 断言 ne02 等于 ne2
    GGML_ASSERT(ne03 == ne3);  // 断言 ne03 等于 ne3

    GGML_ASSERT(nb00 == sizeof(float));  // 断言 nb00 等于 float 的大小
    GGML_ASSERT(nb0  == sizeof(float));  // 断言 nb0 等于 float 的大小
}
    # 遍历第三维度
    for (int i3 = 0; i3 < ne3; i3++) {
        # 遍历第二维度
        for (int i2 = 0; i2 < ne2; i2++) {
            # 遍历第一维度
            for (int i1 = 0; i1 < ne1; i1++) {
                # 计算目标数据指针位置
                float * d = (float *)((char *)  dst->data + i3*nb3  + i2*nb2 + i1*nb1);
                # 计算源数据指针位置
                float * s = (float *)((char *) src0->data + i3*nb03 + i2*nb02);
                # 将目标数据中i1之前的元素置为0
                for (int i0 = 0; i0 < i1; i0++) {
                    d[i0] = 0;
                }
                # 将目标数据中i1位置的元素赋值为源数据中i1位置的元素
                d[i1] = s[i1];
                # 将目标数据中i1之后的元素置为0
                for (int i0 = i1+1; i0 < ne0; i0++) {
                    d[i0] = 0;
                }
            }
        }
    }
// 计算前向对角线操作，根据输入参数和源张量计算结果存储到目标张量中
static void ggml_compute_forward_diag(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据源张量的数据类型进行不同的操作
    switch (src0->type) {
        // 如果数据类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用对应的 F32 类型的前向对角线计算函数
                ggml_compute_forward_diag_f32(params, src0, dst);
            } break;
        // 如果数据类型为其他类型
        default:
            {
                // 断言错误，暂不支持的数据类型
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_diag_mask_inf

// 计算 F32 类型的前向对角线操作，根据输入参数和源张量计算结果存储到目标张量中，同时使用给定的值进行掩码操作
static void ggml_compute_forward_diag_mask_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst,
        const float value) {

    // 获取参数中的 ith 和 nth 值
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取目标张量中的过去数据量和是否为原地操作的标志
    const int  n_past  = ((int32_t *) dst->op_params)[0];
    const bool inplace = src0->data == dst->data;

    // 断言过去数据量大于等于 0
    GGML_ASSERT(n_past >= 0);

    // 如果不是原地操作且任务类型为 GGML_TASK_INIT
    if (!inplace && (params->type == GGML_TASK_INIT)) {
        // 需要在 INIT 阶段同步执行 memcpy，避免竞争条件
        GGML_ASSERT(ggml_nelements(dst) == ggml_nelements(src0));
        GGML_ASSERT(ggml_is_contiguous(dst) && ggml_is_contiguous(src0));
        // 执行内存拷贝操作
        memcpy(
            ((char *)  dst->data),
            ((char *) src0->data),
            ggml_nbytes(dst));
    }

    // 如果任务类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
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

    // 遍历计算结果张量的深度
    for (int k = 0; k < nz; k++) {
        // 遍历计算结果张量的列
        for (int j = ith; j < nr; j += nth) {
            // 遍历计算结果张量的行
            for (int i = n_past; i < nc; i++) {
                // 如果行索引大于过去数据量加上列索引
                if (i > n_past + j) {
                    // 将给定值写入到计算结果张量的对应位置
                    *(float *)((char *) dst->data + k*dst->nb[2] + j*dst->nb[1] + i*dst->nb[0]) = value;
                }
            }
        }
    }
}
    # 代码块结束
// 计算前向对角线掩码为负无穷的操作
static void ggml_compute_forward_diag_mask_inf(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的数据类型进行不同的操作
    switch (src0->type) {
        // 如果数据类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用具体的计算函数，传入参数和负无穷
                ggml_compute_forward_diag_mask_f32(params, src0, dst, -INFINITY);
            } break;
        // 如果数据类型不是 GGML_TYPE_F32
        default:
            {
                // 抛出断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// 计算前向对角线掩码为零的操作
static void ggml_compute_forward_diag_mask_zero(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的数据类型进行不同的操作
    switch (src0->type) {
        // 如果数据类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用具体的计算函数，传入参数和零
                ggml_compute_forward_diag_mask_f32(params, src0, dst, 0);
            } break;
        // 如果数据类型不是 GGML_TYPE_F32
        default:
            {
                // 抛出断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// 计算前向软最大值
static void ggml_compute_forward_soft_max_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言输入张量和输出张量是连续的
    GGML_ASSERT(ggml_is_contiguous(src0));
    GGML_ASSERT(ggml_is_contiguous(dst));
    // 断言输入张量和输出张量的形状相同
    GGML_ASSERT(ggml_are_same_shape(src0, dst));

    // 如果参数类型为初始化或者结束，直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // TODO: 处理转置/置换矩阵

    // 获取线程的索引和总数
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取输入张量的列数和行数
    const int nc = src0->ne[0];
    const int nr = ggml_nrows(src0);

    // 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 遍历处理每一行
    for (int i1 = ir0; i1 < ir1; i1++) {
        // 获取当前行的指针
        float *sp = (float *)((char *) src0->data + i1*src0->nb[1]);
        float *dp = (float *)((char *)  dst->data +  i1*dst->nb[1]);
#ifndef NDEBUG
        // 如果处于调试模式，检查数组中的每个元素是否为 NaN
        for (int i = 0; i < nc; ++i) {
            //printf("p[%d] = %f\n", i, p[i]);
            // 使用断言检查数组中的每个元素是否为 NaN
            assert(!isnan(sp[i]));
        }
#endif

        // 初始化最大值为负无穷大
        float max = -INFINITY;
        // 找到数组中的最大值
        ggml_vec_max_f32(nc, &max, sp);

        // 初始化和值为 0
        ggml_float sum = 0.0;

        // 定义一个 16 位无符号整数
        uint16_t scvt;
        // 遍历数组中的每个元素
        for (int i = 0; i < nc; i++) {
            // 如果数组中的元素为负无穷大，则将对应的目标数组元素设为 0
            if (sp[i] == -INFINITY) {
                dp[i] = 0.0f;
            } else {
                // 将数组中的元素减去最大值，然后计算指数值
                ggml_fp16_t s = GGML_FP32_TO_FP16(sp[i] - max);
                // 将 16 位浮点数转换为 16 位无符号整数
                memcpy(&scvt, &s, sizeof(scvt));
                // 从预先计算的指数表中获取指数值
                const float val = GGML_FP16_TO_FP32(ggml_table_exp_f16[scvt]);
                // 将指数值加到和值中
                sum += (ggml_float)val;
                // 将指数值存储到目标数组中
                dp[i] = val;
            }
        }

        // 使用断言检查和值是否大于 0
        assert(sum > 0.0);

        // 计算和值的倒数
        sum = 1.0/sum;
        // 将目标数组中的每个元素乘以和值的倒数
        ggml_vec_scale_f32(nc, dp, sum);

#ifndef NDEBUG
        // 如果处于调试模式，检查数组中的每个元素是否为 NaN 或无穷大
        for (int i = 0; i < nc; ++i) {
            // 使用断言检查数组中的每个元素是否为 NaN
            assert(!isnan(dp[i]));
            // 使用断言检查数组中的每个元素是否为无穷大
            assert(!isinf(dp[i]));
        }
#endif
    }
}

// 计算前向 Softmax 函数
static void ggml_compute_forward_soft_max(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的处理
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                // 调用处理单精度浮点数的前向 Softmax 函数
                ggml_compute_forward_soft_max_f32(params, src0, dst);
            } break;
        default:
            {
                // 如果类型不匹配，则抛出断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// 计算后向 Softmax 函数的单精度浮点数版本
static void ggml_compute_forward_soft_max_back_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst) {
    // 使用断言检查输入张量的连续性
    GGML_ASSERT(ggml_is_contiguous(src0));
    GGML_ASSERT(ggml_is_contiguous(src1));
    GGML_ASSERT(ggml_is_contiguous(dst));
    // 使用断言检查输入张量和输出张量的形状是否相同
    GGML_ASSERT(ggml_are_same_shape(src0, dst));
    GGML_ASSERT(ggml_are_same_shape(src1, dst));
    // 如果任务类型是初始化或者结束，直接返回，不做任何处理
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // TODO: 处理转置/置换矩阵的情况

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
        // 获取源矩阵、目标矩阵和结果矩阵的指针
        float *dy = (float *)((char *) src0->data + i1*src0->nb[1]);
        float *y  = (float *)((char *) src1->data + i1*src1->nb[1]);
        float *dx = (float *)((char *) dst->data  + i1*dst->nb[1]);
#ifndef NDEBUG
        // 如果处于调试模式，进行以下断言检查
        for (int i = 0; i < nc; ++i) {
            // 检查 dy[i] 和 y[i] 是否为 NaN
            assert(!isnan(dy[i]));
            assert(!isnan(y[i]));
        }
#endif
        // 计算 J 矩阵的各个元素
        // Jii = yi - yi*yi
        // Jij = -yi*yj
        // J = diag(y)-y.T*y
        // 计算 dx = J * dy
        // 计算 dxk = sum_i(Jki * dyi)
        // 展开 dxk 的计算过程
        // dxk = sum_i(-yk*yi * dyi) - (-yk*yk)*dyk + (yk - yk*yk)*dyk
        // dxk = sum_i(-yk*yi * dyi) + yk*yk*dyk + yk*dyk - yk*yk*dyk
        // dxk = sum_i(-yk*yi * dyi) + yk*dyk
        // dxk = -yk * sum_i(yi * dyi) + yk*dyk
        // dxk = -yk * dot(y, dy) + yk*dyk
        // dxk = yk * (- dot(y, dy) + dyk)
        // dxk = yk * (dyk - dot(y, dy))
        //
        // 后序计算顺序:
        // 计算点积 dot_y_dy := dot(y, dy)
        // 将 dy 复制给 dx
        // dx = dx - dot_y_dy
        // dx = dx * y

        // 线性运行时间，不需要额外的内存
        float dot_y_dy = 0;
        // 计算 y 和 dy 的点积
        ggml_vec_dot_f32 (nc, &dot_y_dy, y, dy);
        // 将 dy 复制给 dx
        ggml_vec_cpy_f32 (nc, dx, dy);
        // dx = dx - dot_y_dy
        ggml_vec_acc1_f32(nc, dx, -dot_y_dy);
        // dx = dx * y
        ggml_vec_mul_f32 (nc, dx, dx, y);

#ifndef NDEBUG
        // 如果处于调试模式，进行以下断言检查
        for (int i = 0; i < nc; ++i) {
            // 检查 dx[i] 是否为 NaN 和 无穷大
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
                // 如果类型不是 f32，抛出断言错误
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
    // 断言参数中的 ith 值为 0
    assert(params->ith == 0);
    // 如果任务类型是初始化或者结束，直接返回，不执行后续操作
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 从目标参数中获取n_head的值
    const int n_head = ((int32_t *) dst->op_params)[1];
    float max_bias;
    // 从目标参数中获取max_bias的值
    memcpy(&max_bias, (int32_t *) dst->op_params + 2, sizeof(float));

    // 获取src0的ne数组中的三个值，分别表示all_seq_len、seq_len_without_past和n_head
    const int64_t ne0 = src0->ne[0];
    const int64_t ne1 = src0->ne[1];
    const int64_t ne2 = src0->ne[2];

    // 获取src0的行数n
    const int64_t n  = ggml_nrows(src0);
    // 计算ne2_ne3，表示n/ne1
    const int64_t ne2_ne3 = n/ne1;

    // 获取src0的nb数组中的三个值
    const size_t nb0 = src0->nb[0];
    const size_t nb1 = src0->nb[1];
    const size_t nb2 = src0->nb[2];

    // 断言nb0的大小为float的大小
    GGML_ASSERT(nb0 == sizeof(float);
    // 断言n_head的值等于ne2的值
    GGML_ASSERT(n_head == ne2);

    // 计算n_head的对数的最大整数值
    const int n_heads_log2_floor = 1 << (int) floor(log2(n_head));

    // 计算m0和m1的值
    const float m0 = powf(2.0f, -(max_bias) / n_heads_log2_floor);
    const float m1 = powf(2.0f, -(max_bias / 2.0f) / n_heads_log2_floor);

    // 循环遍历ne0、ne1和ne2_ne3
    for (int64_t i = 0; i < ne0; i++) {
        for (int64_t j = 0; j < ne1; j++) {
            for (int64_t k = 0; k < ne2_ne3; k++) {
                // 获取src和pdst的指针
                float * const src = (float *)((char *) src0->data + i*nb0 + j*nb1 + k*nb2);
                float *      pdst = (float *)((char *)  dst->data + i*nb0 + j*nb1 + k*nb2);

                // 根据k的值计算m_k的值
                float m_k;
                if (k < n_heads_log2_floor) {
                    m_k = powf(m0, k + 1);
                } else {
                    m_k = powf(m1, 2 * (k - n_heads_log2_floor) + 1);
                }

                // 计算pdst[0]的值
                pdst[0] = i * m_k + src[0];
            }
        }
    }
}
# 定义一个静态函数，用于计算前向传播的alibi_f16
static void ggml_compute_forward_alibi_f16(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    # 断言，确保参数中的ith为0
    assert(params->ith == 0);

    # 如果任务类型为初始化或者结束，直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    # 从dst的操作参数中获取n_head
    const int n_head = ((int32_t *) dst->op_params)[1];
    float max_bias;
    # 从dst的操作参数中获取max_bias
    memcpy(&max_bias, (int32_t *) dst->op_params + 2, sizeof(float));

    # 获取src0的ne数组中的值
    const int ne0 = src0->ne[0]; # all_seq_len = n_past + ne1
    const int ne1 = src0->ne[1]; # seq_len_without_past
    const int ne2 = src0->ne[2]; # n_head -> this is k
    #const int ne3 = src0->ne[3]; # 1 -> bsz

    # 获取src0的行数n
    const int n  = ggml_nrows(src0);
    # 计算ne2_ne3
    const int ne2_ne3 = n/ne1; # ne2*ne3

    # 获取src0的nb数组中的值
    const int nb0 = src0->nb[0];
    const int nb1 = src0->nb[1];
    const int nb2 = src0->nb[2];
    #const int nb3 = src0->nb[3];

    # 断言，确保nb0的大小为ggml_fp16_t的大小
    GGML_ASSERT(nb0 == sizeof(ggml_fp16_t));
    # 断言，确保n_head等于ne2
    GGML_ASSERT(n_head == ne2);

    # 计算n_heads_log2_floor
    const int n_heads_log2_floor = 1 << (int) floor(log2(n_head));

    # 计算m0和m1
    const float m0 = powf(2.0f, -(max_bias) / n_heads_log2_floor);
    const float m1 = powf(2.0f, -(max_bias / 2.0f) / n_heads_log2_floor);
    # 遍历 ne0 次
    for (int i = 0; i < ne0; i++) {
        # 遍历 ne1 次
        for (int j = 0; j < ne1; j++) {
            # 遍历 ne2_ne3 次
            for (int k = 0; k < ne2_ne3; k++) {
                # 计算源数据的地址
                ggml_fp16_t * const src  = (ggml_fp16_t *)((char *) src0->data + i*nb0 + j*nb1 + k*nb2);
                # 计算目标数据的地址
                float *      pdst  =       (float *)((char *)  dst->data + i*nb0 + j*nb1 + k*nb2);

                # TODO: k*nb2 or k*nb3
                # 待办事项：k*nb2 还是 k*nb3

                # 定义变量 m_k
                float m_k;

                # 根据 k 的值计算 m_k
                if (k < n_heads_log2_floor) {
                    m_k = powf(m0, k + 1);
                } else {
                    m_k = powf(m1, 2 * (k - n_heads_log2_floor) + 1);
                }

                # 返回 F32 类型的值
                pdst[0] = i * m_k + GGML_FP16_TO_FP32(src[0]);
            }
        }
    }
// 计算前向传播的阿里巴巴函数，根据输入参数和源张量计算结果存储到目标张量中
static void ggml_compute_forward_alibi(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据源张量的数据类型进行不同的处理
    switch (src0->type) {
        // 如果数据类型为 GGML_TYPE_F16
        case GGML_TYPE_F16:
            {
                // 调用特定的 F16 类型的前向传播函数
                ggml_compute_forward_alibi_f16(params, src0, dst);
            } break;
        // 如果数据类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用特定的 F32 类型的前向传播函数
                ggml_compute_forward_alibi_f32(params, src0, dst);
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
        case GGML_TYPE_Q8_K:
        case GGML_TYPE_I8:
        case GGML_TYPE_I16:
        case GGML_TYPE_I32:
        case GGML_TYPE_COUNT:
            {
                // 抛出断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_clamp

// 计算 F32 类型的前向传播的阿里巴巴函数
static void ggml_compute_forward_clamp_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 断言参数的索引为 0
    assert(params->ith == 0);

    // 如果任务类型为初始化或者结束，直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 定义最小值和最大值
    float min;
    float max;
    // 从目标张量的操作参数中复制最小值和最大值
    memcpy(&min, (float *) dst->op_params + 0, sizeof(float));
    memcpy(&max, (float *) dst->op_params + 1, sizeof(float));

    // 定义一些常量和变量
    const int ith = params->ith;
    const int nth = params->nth;

    const int n  = ggml_nrows(src0);
    const int nc = src0->ne[0];

    const size_t nb00 = src0->nb[0];
    const size_t nb01 = src0->nb[1];

    const size_t nb0 = dst->nb[0];
    const size_t nb1 = dst->nb[1];

    // 断言目标张量的第一个维度大小为 float 类型的大小
    GGML_ASSERT( nb0 == sizeof(float));
    // 断言源张量的第一个维度大小为 float 类型的大小
    GGML_ASSERT(nb00 == sizeof(float));
    # 遍历从ith开始，每隔nth取一个值，直到n结束
    for (int j = ith; j < n; j += nth) {
        # 计算目标数据指针的位置
        float * dst_ptr  = (float *) ((char *)  dst->data + j*nb1);
        # 计算源数据0指针的位置
        float * src0_ptr = (float *) ((char *) src0->data + j*nb01);

        # 遍历处理每个通道的数据
        for (int i = 0; i < nc; i++) {
            # 将源数据0的值限制在最小值和最大值之间，然后赋值给目标数据
            dst_ptr[i] = MAX(MIN(src0_ptr[i], max), min);
        }
    }
// 根据输入参数计算前向传播的夹紧操作
static void ggml_compute_forward_clamp(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的操作
    switch (src0->type) {
        // 如果是 F32 类型
        case GGML_TYPE_F32:
            {
                // 调用对应的 F32 类型的夹紧操作函数
                ggml_compute_forward_clamp_f32(params, src0, dst);
            } break;
        // 如果是其他类型
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
        case GGML_TYPE_Q8_K:
        case GGML_TYPE_I8:
        case GGML_TYPE_I16:
        case GGML_TYPE_I32:
        case GGML_TYPE_COUNT:
            {
                // 抛出断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// 计算绳索的斜率
static float rope_yarn_ramp(const float low, const float high, const int i0) {
    // 计算斜率
    const float y = (i0 / 2 - low) / MAX(0.001f, high - low);
    // 返回斜率的值
    return 1 - MIN(1, MAX(0, y));
}

// 基于 LlamaYaRNScaledRotaryEmbedding.py 的 YaRN 算法
// 来自 https://github.com/jquesnelle/yarn
// MIT 许可证。版权所有 (c) 2023 Jeffrey Quesnelle 和 Bowen Peng。
// 计算绳索
static void rope_yarn(
    float theta_extrap, float freq_scale, float corr_dims[2], int64_t i0, float ext_factor, float mscale,
    float * cos_theta, float * sin_theta
) {
    // 获取校正后的旋转缩放
    float theta_interp = freq_scale * theta_extrap;
    float theta = theta_interp;
    // 如果外推因子不为0
    if (ext_factor != 0.0f) {
        // 计算斜坡混合
        float ramp_mix = rope_yarn_ramp(corr_dims[0], corr_dims[1], i0) * ext_factor;
        // 根据斜坡混合调整角度
        theta = theta_interp * (1 - ramp_mix) + theta_extrap * ramp_mix;

        // 获取校正后的插值尺度
        mscale *= 1.0f + 0.1f * logf(1.0f / freq_scale);
    }
    // 计算余弦和正弦值
    *cos_theta = cosf(theta) * mscale;
    *sin_theta = sinf(theta) * mscale;
}
// 计算维度修正值，根据给定的维度、原始上下文数、旋转角度和基数
static float ggml_rope_yarn_corr_dim(int n_dims, int n_orig_ctx, float n_rot, float base) {
    return n_dims * logf(n_orig_ctx / (n_rot * 2 * (float)M_PI)) / (2 * logf(base));
}

// 计算起始和结束的修正维度
void ggml_rope_yarn_corr_dims(
    int n_dims, int n_orig_ctx, float freq_base, float beta_fast, float beta_slow, float dims[2]
) {
    // start and end correction dims
    dims[0] = MAX(0,         floorf(ggml_rope_yarn_corr_dim(n_dims, n_orig_ctx, beta_fast, freq_base)));
    dims[1] = MIN(n_dims - 1, ceilf(ggml_rope_yarn_corr_dim(n_dims, n_orig_ctx, beta_slow, freq_base)));
}

// 计算前向 RoPE f32
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

    // these two only relevant for xPos RoPE:
    float xpos_base;
    bool  xpos_down;

    //const int n_past     = ((int32_t *) dst->op_params)[0];
    const int n_dims     = ((int32_t *) dst->op_params)[1];
    const int mode       = ((int32_t *) dst->op_params)[2];
    const int n_ctx      = ((int32_t *) dst->op_params)[3];
    const int n_orig_ctx = ((int32_t *) dst->op_params)[4];

    // 从 op_params 数组中复制参数值
    memcpy(&freq_base,   (int32_t *) dst->op_params +  5, sizeof(float));
    memcpy(&freq_scale,  (int32_t *) dst->op_params +  6, sizeof(float));
    memcpy(&ext_factor,  (int32_t *) dst->op_params +  7, sizeof(float));
    memcpy(&attn_factor, (int32_t *) dst->op_params +  8, sizeof(float));
    memcpy(&beta_fast,   (int32_t *) dst->op_params +  9, sizeof(float));
    memcpy(&beta_slow,   (int32_t *) dst->op_params + 10, sizeof(float));
}
    // 从目标内存中的第11个参数开始，复制一个float大小的数据到xpos_base
    memcpy(&xpos_base,   (int32_t *) dst->op_params + 11, sizeof(float));
    // 从目标内存中的第12个参数开始，复制一个bool大小的数据到xpos_down
    memcpy(&xpos_down,   (int32_t *) dst->op_params + 12, sizeof(bool));

    // 定义了一个宏，用于声明一些本地变量
    GGML_TENSOR_UNARY_OP_LOCALS

    // 断言nb00的值等于float的大小
    GGML_ASSERT(nb00 == sizeof(float));

    // 从参数中获取ith和nth的值
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取dst的行数
    const int nr = ggml_nrows(dst);

    // 断言n_dims小于等于ne0
    GGML_ASSERT(n_dims <= ne0);
    // 断言n_dims能被2整除
    GGML_ASSERT(n_dims % 2 == 0);

    // 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 用于确定使用哪个线程的行索引
    int ir = 0;

    // 计算theta_scale的值
    const float theta_scale = powf(freq_base, -2.0f/n_dims);
    // 计算inv_ndims的值
    const float inv_ndims = -1.f/n_dims;
    // 定义一个长度为2的float数组corr_dims，并计算其值
    float corr_dims[2];
    ggml_rope_yarn_corr_dims(n_dims, n_orig_ctx, freq_base, beta_fast, beta_slow, corr_dims);

    // 判断是否是neox模式
    const bool is_neox = mode & 2;
    // 判断是否是glm模式
    const bool is_glm  = mode & 4;

    // 如果是正向过程，则sin_sign为1.0，否则为-1.0
    const float sin_sign = forward ? 1.0f : -1.0f;

    // 将src1的数据转换为int32_t类型的指针
    const int32_t * pos = (const int32_t *) src1->data;
}
# 定义一个静态函数，用于计算前向绳索传播
static void ggml_compute_forward_rope_f16(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst,
        const bool forward) {
    # 如果任务类型是初始化或者结束，直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    float freq_base, freq_scale, ext_factor, attn_factor, beta_fast, beta_slow;

    # 从目标张量的操作参数中获取相关参数
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

    # 确保目标张量的元素字节大小为 ggml_fp16_t 类型的大小
    GGML_ASSERT(nb0 == sizeof(ggml_fp16_t));

    # 获取当前线程的索引和总线程数
    const int ith = params->ith;
    const int nth = params->nth;

    # 获取目标张量的行数
    const int nr = ggml_nrows(dst);

    # 确保 n_dims 小于等于 ne0
    GGML_ASSERT(n_dims <= ne0);
    # 确保 n_dims 是偶数
    GGML_ASSERT(n_dims % 2 == 0);

    # 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    # 当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    # 用于确定使用哪个线程的行索引
    int ir = 0;

    # 计算频率比例和维度的倒数
    const float theta_scale = powf(freq_base, -2.0f/n_dims);
    const float inv_ndims = -1.f/n_dims;
    float corr_dims[2];
    # 使用给定参数调用 ggml_rope_yarn_corr_dims 函数
    ggml_rope_yarn_corr_dims(n_dims, n_orig_ctx, freq_base, beta_fast, beta_slow, corr_dims);

    # 检查 mode 是否包含 2，判断是否为 neox 模式
    const bool is_neox = mode & 2;
    # 检查 mode 是否包含 4，判断是否为 glm 模式
    const bool is_glm  = mode & 4;

    # 如果是 forward 模式，sin_sign 为 1.0，否则为 -1.0
    # 这里解释了正向和反向过程中 sin 的符号变化
    const float sin_sign = forward ? 1.0f : -1.0f;

    # 将 src1->data 强制转换为 int32_t 指针类型，并赋值给 pos
    const int32_t * pos = (const int32_t *) src1->data;
// 计算前向传播的绳索操作，根据输入参数和张量类型进行计算
static void ggml_compute_forward_rope(
        const struct ggml_compute_params * params,  // 输入参数结构体指针
        const struct ggml_tensor * src0,  // 输入张量指针
        const struct ggml_tensor * src1,  // 输入张量指针
        struct ggml_tensor * dst) {  // 输出张量指针
    // 根据输入张量 src0 的类型进行不同的操作
    switch (src0->type) {
        case GGML_TYPE_F16:  // 如果输入张量类型为 GGML_TYPE_F16
            {
                ggml_compute_forward_rope_f16(params, src0, src1, dst, true);  // 调用对应类型的前向传播绳索操作函数
            } break;
        case GGML_TYPE_F32:  // 如果输入张量类型为 GGML_TYPE_F32
            {
                ggml_compute_forward_rope_f32(params, src0, src1, dst, true);  // 调用对应类型的前向传播绳索操作函数
            } break;
        default:  // 如果输入张量类型不是上述两种类型
            {
                GGML_ASSERT(false);  // 抛出断言错误
            } break;
    }
}

// ggml_compute_forward_rope_back

// 计算反向传播的绳索操作，根据输入参数和张量类型进行计算
static void ggml_compute_forward_rope_back(
        const struct ggml_compute_params * params,  // 输入参数结构体指针
        const struct ggml_tensor * src0,  // 输入张量指针
        const struct ggml_tensor * src1,  // 输入张量指针
        struct ggml_tensor * dst) {  // 输出张量指针
    // 根据输入张量 src0 的类型进行不同的操作
    switch (src0->type) {
        case GGML_TYPE_F16:  // 如果输入张量类型为 GGML_TYPE_F16
            {
                ggml_compute_forward_rope_f16(params, src0, src1, dst, false);  // 调用对应类型的前向传播绳索操作函数
            } break;
        case GGML_TYPE_F32:  // 如果输入张量类型为 GGML_TYPE_F32
            {
                ggml_compute_forward_rope_f32(params, src0, src1, dst, false);  // 调用对应类型的前向传播绳索操作函数
            } break;
        default:  // 如果输入张量类型不是上述两种类型
            {
                GGML_ASSERT(false);  // 抛出断言错误
            } break;
    }
}

// ggml_compute_forward_conv_transpose_1d

// 计算一维卷积转置的前向传播操作，根据输入参数和张量类型进行计算
static void ggml_compute_forward_conv_transpose_1d_f16_f32(
        const struct ggml_compute_params * params,  // 输入参数结构体指针
        const struct ggml_tensor * src0,  // 输入张量指针
        const struct ggml_tensor * src1,  // 输入张量指针
              struct ggml_tensor * dst) {  // 输出张量指针
    GGML_ASSERT(src0->type == GGML_TYPE_F16);  // 断言输入张量 src0 的类型为 GGML_TYPE_F16
    GGML_ASSERT(src1->type == GGML_TYPE_F32);  // 断言输入张量 src1 的类型为 GGML_TYPE_F32
    GGML_ASSERT( dst->type == GGML_TYPE_F32);  // 断言输出张量 dst 的类型为 GGML_TYPE_F32

    int64_t t0 = ggml_perf_time_us();  // 获取当前时间的微秒数
    UNUSED(t0);  // 标记 t0 为未使用

    GGML_TENSOR_BINARY_OP_LOCALS  // 定义张量二元操作的本地变量

    const int ith = params->ith;  // 获取参数结构体中的 ith 值
    const int nth = params->nth;  // 获取参数结构体中的 nth 值

    const int nk = ne00*ne01*ne02;  // 计算 nk 值

    GGML_ASSERT(nb00 == sizeof(ggml_fp16_t));  // 断言 nb00 的值等于 ggml_fp16_t 的大小
    GGML_ASSERT(nb10 == sizeof(float));  // 断言 nb10 的值等于 float 的大小
    // 如果任务类型为初始化
    if (params->type == GGML_TASK_INIT) {
        // 将 params->wdata 内存块清零
        memset(params->wdata, 0, params->wsize);

        // 对内存块中的数据进行重新排列，从 (K x Cout x Cin) 到 (Cin x K x Cout)
        {
            // 获取 wdata 指针，并偏移 0 个 ggml_fp16_t 单位
            ggml_fp16_t * const wdata = (ggml_fp16_t *) params->wdata + 0;

            // 遍历 ne02
            for (int64_t i02 = 0; i02 < ne02; i02++) {
                // 遍历 ne01
                for (int64_t i01 = 0; i01 < ne01; i01++) {
                    // 获取 src 指针
                    const ggml_fp16_t * const src = (ggml_fp16_t *)((char *) src0->data + i02*nb02 + i01*nb01);
                    // 获取 dst_data 指针
                    ggml_fp16_t * dst_data = wdata + i01*ne00*ne02;
                    // 遍历 ne00
                    for (int64_t i00 = 0; i00 < ne00; i00++) {
                        // 重新排列数据
                        dst_data[i00*ne02 + i02] = src[i00];
                    }
                }
            }
        }

        // 对源数据进行重新排列，从 (L x Cin) 到 (Cin x L)
        {
            // 获取 wdata 指针，并偏移 nk 个 ggml_fp16_t 单位
            ggml_fp16_t * const wdata = (ggml_fp16_t *) params->wdata + nk;
            // 获取 dst_data 指针
            ggml_fp16_t * dst_data = wdata;

            // 遍历 ne11
            for (int64_t i11 = 0; i11 < ne11; i11++) {
                // 获取 src 指针
                const float * const src = (float *)((char *) src1->data + i11*nb11);
                // 遍历 ne10
                for (int64_t i10 = 0; i10 < ne10; i10++) {
                    // 将 float 类型数据转换为 ggml_fp16_t 类型，并重新排列数据
                    dst_data[i10*ne11 + i11] = GGML_FP32_TO_FP16(src[i10]);
                }
            }
        }

        // 将目标数据内存块清零，因为将要对其进行累加
        memset(dst->data, 0, ggml_nbytes(dst));

        // 返回
        return;
    }

    // 如果任务类型为结束
    if (params->type == GGML_TASK_FINALIZE) {
        // 返回
        return;
    }

    // 获取 dst->op_params 中的第一个 int32_t 类型数据
    const int32_t s0 = ((const int32_t*)(dst->op_params))[0];

    // dst 中的总行数
    const int nr = ne1;

    // 每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 本线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 获取 wdata 指针，并偏移 0 个 ggml_fp16_t 单位
    ggml_fp16_t * const wdata     = (ggml_fp16_t *) params->wdata + 0;
    // 获取 wdata_src 指针
    ggml_fp16_t * const wdata_src = wdata + nk;
    # 遍历 ir0 到 ir1 之间的整数
    for (int i1 = ir0; i1 < ir1; i1++) {
        # 计算 dst_data 的地址
        float * dst_data = (float *)((char *) dst->data + i1*nb1);
        # 计算 wdata_kernel 的地址
        ggml_fp16_t * wdata_kernel = wdata + i1*ne02*ne00;
        # 遍历 ne10 之间的整数
        for (int i10 = 0; i10 < ne10; i10++) {
            # 计算 i1n
            const int i1n = i10*ne11;
            # 遍历 ne00 之间的整数
            for (int i00 = 0; i00 < ne00; i00++) {
                # 初始化 v 为 0
                float v = 0;
                # 计算内积并将结果存入 v
                ggml_vec_dot_f16(ne02, &v,
                        (ggml_fp16_t *)    wdata_src + i1n,
                        (ggml_fp16_t *) wdata_kernel + i00*ne02);
                # 将 v 加到 dst_data[i10*s0 + i00] 上
                dst_data[i10*s0 + i00] += v;
            }
        }
    }
static void ggml_compute_forward_conv_transpose_1d_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
              struct ggml_tensor * dst) {
    // 断言输入张量的数据类型为 GGML_TYPE_F32
    GGML_ASSERT(src0->type == GGML_TYPE_F32);
    GGML_ASSERT(src1->type == GGML_TYPE_F32);
    GGML_ASSERT( dst->type == GGML_TYPE_F32);

    // 获取当前时间，但未使用
    int64_t t0 = ggml_perf_time_us();
    UNUSED(t0);

    // 定义二元操作的本地变量
    GGML_TENSOR_BINARY_OP_LOCALS

    // 获取参数中的 ith 和 nth
    const int ith = params->ith;
    const int nth = params->nth;

    // 计算 nk 的值
    const int nk = ne00*ne01*ne02;

    // 断言 nb00 和 nb10 的大小为 float 类型
    GGML_ASSERT(nb00 == sizeof(float));
    GGML_ASSERT(nb10 == sizeof(float);

    // 如果任务类型为 GGML_TASK_INIT
    if (params->type == GGML_TASK_INIT) {
        // 将参数中的 wdata 初始化为 0
        memset(params->wdata, 0, params->wsize);

        // 准备核数据（src0），从（K x Cout x Cin）转换为（Cin x K x Cout）
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

        // 需要将目标张量 dst 初始化为 0，因为我们将累加到其中
        memset(dst->data, 0, ggml_nbytes(dst));

        return;
    }
}
    // 如果任务类型为 GGML_TASK_FINALIZE，则直接返回，不执行后续操作
    if (params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 从目标数据结构的操作参数中获取第一个整数值
    const int32_t s0 = ((const int32_t*)(dst->op_params))[0];

    // 计算目标数据结构中的总行数
    const int nr = ne1;

    // 计算每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 计算当前线程处理的行数范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 获取参数结构中的 wdata 指针，并偏移 nk 个元素的位置
    float * const wdata     = (float *) params->wdata + 0;
    float * const wdata_src = wdata + nk;

    // 遍历当前线程处理的行范围
    for (int i1 = ir0; i1 < ir1; i1++) {
        // 获取目标数据结构中第 i1 行的数据指针
        float * dst_data = (float *)((char *) dst->data + i1*nb1);
        // 获取 wdata 中第 i1 行对应的权重数据指针
        float * wdata_kernel = wdata + i1*ne02*ne00;
        // 遍历权重数据的维度
        for (int i10 = 0; i10 < ne10; i10++) {
            const int i1n = i10*ne11;
            for (int i00 = 0; i00 < ne00; i00++) {
                // 初始化累加变量 v
                float v = 0;
                // 调用 ggml_vec_dot_f32 函数计算两个向量的点积，并累加到 v 中
                ggml_vec_dot_f32(ne02, &v,
                        wdata_src + i1n,
                        wdata_kernel + i00*ne02);
                // 将累加结果加到目标数据结构的对应位置上
                dst_data[i10*s0 + i00] += v;
            }
        }
    }
// 计算一维卷积转置操作
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
                // 调用 ggml_compute_forward_conv_transpose_1d_f16_f32 函数进行计算
                ggml_compute_forward_conv_transpose_1d_f16_f32(params, src0, src1, dst);
            } break;
        // 如果数据类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用 ggml_compute_forward_conv_transpose_1d_f32 函数进行计算
                ggml_compute_forward_conv_transpose_1d_f32(params, src0, src1, dst);
            } break;
        // 如果数据类型不是上述两种情况
        default:
            {
                // 抛出断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// src0: kernel [OC, IC, KH, KW]
// src1: image [N, IC, IH, IW]
// dst:  result [N, OH, OW, IC*KH*KW]
// 计算 im2col 操作
static void ggml_compute_forward_im2col_f16(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
              struct ggml_tensor * dst) {
    // 断言输入张量的数据类型
    GGML_ASSERT(src0->type == GGML_TYPE_F16);
    GGML_ASSERT(src1->type == GGML_TYPE_F32);
    GGML_ASSERT( dst->type == GGML_TYPE_F16);

    // 获取当前时间
    int64_t t0 = ggml_perf_time_us();
    UNUSED(t0);

    // 定义本地变量
    GGML_TENSOR_BINARY_OP_LOCALS;

    // 从 dst 的操作参数中获取值
    const int32_t s0 = ((const int32_t *)(dst->op_params))[0];
    const int32_t s1 = ((const int32_t *)(dst->op_params))[1];
    const int32_t p0 = ((const int32_t *)(dst->op_params))[2];
    const int32_t p1 = ((const int32_t *)(dst->op_params))[3];
    const int32_t d0 = ((const int32_t *)(dst->op_params))[4];
    const int32_t d1 = ((const int32_t *)(dst->op_params))[5];
    const bool is_2D = ((const int32_t *)(dst->op_params))[6] == 1;

    // 获取参数值
    const int ith = params->ith;
    const int nth = params->nth;

    // 根据是否为二维数据，定义不同的维度
    const int64_t N  = is_2D ? ne13 : ne12;
    const int64_t IC = is_2D ? ne12 : ne11;
    const int64_t IH = is_2D ? ne11 : 1;
    const int64_t IW = ne10;

    const int64_t KH = is_2D ? ne01 : 1;
    const int64_t KW = ne00;

    const int64_t OH = is_2D ? ne2 : 1;
    // 定义常量 OW，其值为 ne1
    const int64_t OW = ne1;

    // 根据条件 is_2D 判断 ofs0 和 ofs1 的值
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
        // 将目标数据转换为 ggml_fp16_t 类型的指针
        ggml_fp16_t * const wdata = (ggml_fp16_t *) dst->data;

        // 遍历 N
        for (int64_t in = 0; in < N; in++) {
            // 遍历 OH
            for (int64_t ioh = 0; ioh < OH; ioh++) { // 1
                // 遍历 OW
                for (int64_t iow = 0; iow < OW; iow++) {
                    // 遍历 IC
                    for (int64_t iic = ith; iic < IC; iic += nth) {

                        // 微内核
                        ggml_fp16_t * dst_data = wdata + (in*OH*OW + ioh*OW + iow)*(IC*KH*KW); // [IC, KH, KW]
                        const float * const src_data = (float *)((char *) src1->data + in*ofs0 + iic*ofs1); // [IH, IW]

                        // 遍历 KH
                        for (int64_t ikh = 0; ikh < KH; ikh++) {  // 1
                            // 遍历 KW
                            for (int64_t ikw = 0; ikw < KW; ikw++) {
                                const int64_t iiw = iow*s0 + ikw*d0 - p0;
                                const int64_t iih = ioh*s1 + ikh*d1 - p1;

                                // 如果满足条件，则将对应位置的值设为 0，否则将其转换为 ggml_fp16_t 类型并赋值
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
// 计算前向卷积操作，将输入数据转换为im2col格式
static void ggml_compute_forward_im2col(
        const struct ggml_compute_params * params,  // 输入参数结构体指针
        const struct ggml_tensor * src0,  // 输入张量指针
        const struct ggml_tensor * src1,  // 输入张量指针
              struct ggml_tensor * dst) {  // 输出张量指针
    switch (src0->type) {  // 根据输入张量的数据类型进行判断
        case GGML_TYPE_F16:  // 如果数据类型为F16
            {
                ggml_compute_forward_im2col_f16(params, src0, src1, dst);  // 调用F16类型的im2col计算函数
            } break;
        case GGML_TYPE_F32:  // 如果数据类型为F32
            {
                GGML_ASSERT(false);  // 抛出断言错误
            } break;
        default:  // 其他情况
            {
                GGML_ASSERT(false);  // 抛出断言错误
            } break;
    }
}

// ggml_compute_forward_conv_transpose_2d

// 计算前向2D卷积转置操作
static void ggml_compute_forward_conv_transpose_2d(
        const struct ggml_compute_params * params,  // 输入参数结构体指针
        const struct ggml_tensor * src0,  // 输入张量指针
        const struct ggml_tensor * src1,  // 输入张量指针
              struct ggml_tensor * dst) {  // 输出张量指针
    GGML_ASSERT(src0->type == GGML_TYPE_F16);  // 断言输入张量src0的数据类型为F16
    GGML_ASSERT(src1->type == GGML_TYPE_F32);  // 断言输入张量src1的数据类型为F32
    GGML_ASSERT( dst->type == GGML_TYPE_F32);  // 断言输出张量dst的数据类型为F32

    int64_t t0 = ggml_perf_time_us();  // 获取当前时间的微秒数
    UNUSED(t0);  // 标记t0变量为未使用

    GGML_TENSOR_BINARY_OP_LOCALS  // 定义张量二元操作的本地变量

    const int ith = params->ith;  // 获取参数结构体中的ith值
    const int nth = params->nth;  // 获取参数结构体中的nth值

    const int nk = ne00*ne01*ne02*ne03;  // 计算nk值

    GGML_ASSERT(nb00 == sizeof(ggml_fp16_t));  // 断言nb00的值等于ggml_fp16_t的大小
    GGML_ASSERT(nb10 == sizeof(float));  // 断言nb10的值等于float的大小
}
    // 如果任务类型为初始化
    if (params->type == GGML_TASK_INIT) {
        // 将参数中的 wdata 内存块清零
        memset(params->wdata, 0, params->wsize);

        // 对内存块中的数据进行重新排列，将 src0 的数据从 (Kw x Kh x Cout x Cin) 排列为 (Cin x Kw x Kh x Cout)
        {
            // 将 wdata 强制类型转换为 ggml_fp16_t 指针，并偏移 0 个元素
            ggml_fp16_t * const wdata = (ggml_fp16_t *) params->wdata + 0;

            // 循环遍历 ne03
            for (int64_t i03 = 0; i03 < ne03; i03++) {
                // 循环遍历 ne02
                for (int64_t i02 = 0; i02 < ne02; i02++) {
                    // 计算 src 的地址
                    const ggml_fp16_t * const src = (ggml_fp16_t *)((char *) src0->data + i03*nb03 + i02*nb02);
                    // 计算 dst_data 的地址
                    ggml_fp16_t * dst_data = wdata + i02*ne01*ne00*ne03;
                    // 循环遍历 ne01
                    for (int64_t i01 = 0; i01 < ne01; i01++) {
                        // 循环遍历 ne00
                        for (int64_t i00 = 0; i00 < ne00; i00++) {
                            // 对 dst_data 进行重新排列
                            dst_data[i01*ne00*ne03 + i00*ne03 + i03] = src[i01 * ne00 + i00];
                        }
                    }
                }
            }
        }

        // 对内存块中的数据进行重新排列，将 src1 的数据从 (Sw x Sh x Cin) 排列为 (Cin x Sw x Sh)
        {
            // 将 wdata 强制类型转换为 ggml_fp16_t 指针，并偏移 nk 个元素
            ggml_fp16_t * const wdata = (ggml_fp16_t *) params->wdata + nk;
            // 循环遍历 ne12
            for (int i12 = 0; i12 < ne12; i12++) {
                // 循环遍历 ne11
                for (int i11 = 0; i11 < ne11; i11++) {
                    // 计算 src 的地址
                    const float * const src = (float *)((char *) src1->data + i12*nb12 + i11*nb11);
                    // 计算 dst_data 的地址
                    ggml_fp16_t * dst_data = wdata + i11*ne10*ne12;
                    // 循环遪历 ne10
                    for (int i10 = 0; i10 < ne10; i10++) {
                        // 对 dst_data 进行重新排列，将 float 类型转换为 ggml_fp16_t 类型
                        dst_data[i10*ne12 + i12] = GGML_FP32_TO_FP16(src[i10]);
                    }
                }
            }
        }

        // 将目标数据块清零
        memset(dst->data, 0, ggml_nbytes(dst));

        // 返回
        return;
    }

    // 如果任务类型为结束
    if (params->type == GGML_TASK_FINALIZE) {
        // 返回
        return;
    }

    // 获取 dst 参数中索引为 0 的 int32_t 类型的值
    const int32_t stride = ggml_get_op_params_i32(dst, 0);

    // 计算总共的 patch 数量
    const int np = ne2;

    // 计算每个线程处理的 patch 数量
    const int dp = (np + nth - 1)/nth;

    // 计算当前线程处理的 patch 范围
    const int ip0 = dp*ith;
    const int ip1 = MIN(ip0 + dp, np);
    # 定义指向params->wdata的指针，并偏移0个单位
    ggml_fp16_t * const wdata = (ggml_fp16_t *) params->wdata + 0;
    # 定义指向wdata的指针，并偏移nk个单位
    ggml_fp16_t * const wdata_src = wdata + nk;

    # 遍历ip0到ip1之间的值，表示Cout
    for (int i2 = ip0; i2 < ip1; i2++) { // Cout
        # 计算dst_data的地址，表示输出数据
        float * dst_data = (float *)((char *) dst->data + i2*nb2);
        # 计算wdata_kernel的地址，表示权重数据
        ggml_fp16_t * wdata_kernel = wdata + i2*ne01*ne00*ne03;
        # 遍历ne11个值
        for (int i11 = 0; i11 < ne11; i11++) {
            # 遍历ne10个值
            for (int i10 = 0; i10 < ne10; i10++) {
                # 计算i1n的值
                const int i1n = i11*ne10*ne12 + i10*ne12;
                # 遍历ne01个值
                for (int i01 = 0; i01 < ne01; i01++) {
                    # 遍历ne00个值
                    for (int i00 = 0; i00 < ne00; i00++) {
                        # 初始化v为0
                        float v = 0;
                        # 计算内积并存储到v中
                        ggml_vec_dot_f16(ne03, &v,
                                wdata_src + i1n,
                                wdata_kernel + i01*ne00*ne03 + i00*ne03);
                        # 将v加到dst_data对应位置上
                        dst_data[(i11*stride + i01)*ne0 + i10*stride + i00] += v;
                    }
                }
            }
        }
    }
// ggml_compute_forward_pool_1d_sk_p0

// 计算一维池化操作的前向传播，根据给定参数和输入张量，将结果存储到目标张量中
static void ggml_compute_forward_pool_1d_sk_p0(
        const struct ggml_compute_params * params,  // 输入参数结构体指针
        const enum ggml_op_pool op,  // 池化操作类型枚举
        const struct ggml_tensor * src,  // 输入张量指针
        const int k,  // 池化核大小
        struct ggml_tensor * dst) {  // 输出张量指针
    assert(src->type == GGML_TYPE_F32);  // 断言输入张量数据类型为单精度浮点数
    assert(params->ith == 0);  // 断言参数中的 ith 值为 0

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  // 如果参数类型为初始化或结束，则直接返回
        return;
    }

    const char * cdata = (const char *)src->data;  // 将输入张量数据转换为字符指针
    const char * const data_end = cdata + ggml_nbytes(src);  // 计算输入张量数据结束位置的指针
    float * drow = (float *)dst->data;  // 将输出张量数据转换为单精度浮点数指针

    const int64_t rs = dst->ne[0];  // 获取输出张量的第一个维度大小

    while (cdata < data_end) {  // 循环遍历输入张量数据
        const float * const srow = (const float *)cdata;  // 将当前位置的输入数据转换为单精度浮点数指针

        int j = 0;  // 初始化 j 值为 0

        for (int64_t i = 0; i < rs; ++i) {  // 循环遍历输出张量的第一个维度
            switch (op) {  // 根据池化操作类型进行不同的处理
                case GGML_OP_POOL_AVG:   drow[i] = 0;        break;  // 平均池化操作，将输出值初始化为 0
                case GGML_OP_POOL_MAX:   drow[i] = -FLT_MAX; break;  // 最大池化操作，将输出值初始化为负无穷大
                case GGML_OP_POOL_COUNT: GGML_ASSERT(false); break;  // 计数池化操作，抛出断言错误
            }
            for (int ki = 0; ki < k; ++ki) {  // 循环遍历池化核大小
                switch (op) {  // 根据池化操作类型进行不同的处理
                    case GGML_OP_POOL_AVG:                          drow[i] += srow[j]; break;  // 平均池化操作，累加输入值到输出值
                    case GGML_OP_POOL_MAX:   if (srow[j] > drow[i]) drow[i]  = srow[j]; break;  // 最大池化操作，更新输出值为最大值
                    case GGML_OP_POOL_COUNT:                        GGML_ASSERT(false); break;  // 计数池化操作，抛出断言错误
                }
                ++j;  // j 值加一
            }
            switch (op) {  // 根据池化操作类型进行不同的处理
                case GGML_OP_POOL_AVG:         drow[i] /= k; break;  // 平均池化操作，将输出值除以池化核大小
                case GGML_OP_POOL_MAX:                       break;  // 最大池化操作，无需额外处理
                case GGML_OP_POOL_COUNT: GGML_ASSERT(false); break;  // 计数池化操作，抛出断言错误
            }
        }

        cdata += src->nb[1];  // 更新输入数据指针位置
        drow  += rs;  // 更新输出数据指针位置
    }
}

// ggml_compute_forward_pool_1d

// 计算一维池化操作的前向传播，根据给定参数和输入张量，将结果存储到目标张量中
static void ggml_compute_forward_pool_1d(
        const struct ggml_compute_params * params,  // 输入参数结构体指针
        const struct ggml_tensor * src0,  // 输入张量指针
              struct ggml_tensor * dst) {  // 输出张量指针
    // 将目标指针转换为 int32_t 类型的指针，并赋值给 opts
    const int32_t * opts = (const int32_t *)dst->op_params;
    // 从 opts 数组中取出第一个元素赋值给 op
    enum ggml_op_pool op = opts[0];
    // 从 opts 数组中取出第二个元素赋值给 k0
    const int k0 = opts[1];
    // 从 opts 数组中取出第三个元素赋值给 s0
    const int s0 = opts[2];
    // 从 opts 数组中取出第四个元素赋值给 p0
    const int p0 = opts[3];
    // 断言 p0 等于 0，如果不等于则抛出异常，表示不支持填充
    GGML_ASSERT(p0 == 0); // padding not supported
    // 断言 k0 等于 s0，如果不等于则抛出异常，表示只支持 s = k
    GGML_ASSERT(k0 == s0); // only s = k supported

    // 调用 ggml_compute_forward_pool_1d_sk_p0 函数进行前向池化计算
    ggml_compute_forward_pool_1d_sk_p0(params, op, src0, k0, dst);
// 定义静态函数 ggml_compute_forward_pool_2d，用于计算 2D 池化操作的前向传播
static void ggml_compute_forward_pool_2d(
        const struct ggml_compute_params * params,  // 接收计算参数结构体指针
        const struct ggml_tensor * src,  // 接收输入张量指针
        struct ggml_tensor * dst) {  // 接收输出张量指针
    assert(src->type == GGML_TYPE_F32);  // 断言输入张量数据类型为 GGML_TYPE_F32
    assert(params->ith == 0);  // 断言计算参数中的 ith 属性为 0

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  // 如果计算参数中的 type 属性为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则返回
        return;
    }

    const int32_t * opts = (const int32_t *)dst->op_params;  // 从输出张量的操作参数中获取 int32_t 类型的参数数组
    enum ggml_op_pool op = opts[0];  // 获取操作类型
    const int k0 = opts[1];  // 获取参数 k0
    const int k1 = opts[2];  // 获取参数 k1
    const int s0 = opts[3];  // 获取参数 s0
    const int s1 = opts[4];  // 获取参数 s1
    const int p0 = opts[5];  // 获取参数 p0
    const int p1 = opts[6];  // 获取参数 p1
    const char * cdata = (const char*)src->data;  // 将输入张量的数据转换为 char* 类型
    const char * const data_end = cdata + ggml_nbytes(src);  // 计算输入张量数据的结束位置

    const int64_t px = dst->ne[0];  // 获取输出张量在第一个维度上的大小
    const int64_t py = dst->ne[1];  // 获取输出张量在第二个维度上的大小
    const int64_t pa = px * py;  // 计算输出张量的面积

    float * dplane = (float *)dst->data;  // 将输出张量的数据转换为 float* 类型

    const int ka = k0 * k1;  // 计算卷积核的大小
    const int offset0 = -p0;  // 计算偏移量 offset0
    const int offset1 = -p1;  // 计算偏移量 offset1
    # 当前数据位置小于数据结束位置时执行循环
    while (cdata < data_end) {
        # 遍历输出平面的每一行
        for (int oy = 0; oy < py; ++oy) {
            # 计算当前行的起始地址
            float * const drow = dplane + oy * px;
            # 遍历当前行的每一个像素
            for (int ox = 0; ox < px; ++ox) {
                # 计算当前像素的地址
                float * const out =  drow + ox;
                # 根据操作类型进行不同的初始化
                switch (op) {
                    case GGML_OP_POOL_AVG:     *out = 0;        break;
                    case GGML_OP_POOL_MAX:     *out = -FLT_MAX; break;
                    case GGML_OP_POOL_COUNT: GGML_ASSERT(false); break;
                }
                # 计算输入数据的索引
                const int ix = offset0 + ox * s0;
                const int iy = offset1 + oy * s1;
                # 遍历池化核的每一行
                for (int ky = 0; ky < k1; ++ky) {
                    # 如果计算出的输入数据索引超出范围，则跳过
                    if (iy + ky < 0 || iy + ky >= src->ne[1]) continue;
                    # 计算当前输入数据行的地址
                    const float * const srow = (const float *)(cdata + src->nb[1] * (iy + ky));
                    # 遍历池化核的每一个像素
                    for (int kx = 0; kx < k0; ++kx) {
                        # 计算当前输入数据的索引
                        int j = ix + kx;
                        # 如果计算出的输入数据索引超出范围，则跳过
                        if (j < 0 || j >= src->ne[0]) continue;
                        # 根据操作类型进行不同的处理
                        switch (op) {
                            case GGML_OP_POOL_AVG:                     *out += srow[j]; break;
                            case GGML_OP_POOL_MAX: if (srow[j] > *out) *out  = srow[j]; break;
                            case GGML_OP_POOL_COUNT:                GGML_ASSERT(false); break;
                        }
                    }
                }
                # 根据操作类型进行不同的处理
                switch (op) {
                    case GGML_OP_POOL_AVG:           *out /= ka; break;
                    case GGML_OP_POOL_MAX:                       break;
                    case GGML_OP_POOL_COUNT: GGML_ASSERT(false); break;
                }
            }
        }
        # 更新当前数据位置
        cdata  += src->nb[2];
        # 更新输出平面的地址
        dplane += pa;
    }
// ggml_compute_forward_upscale

// 计算前向上采样的函数，将输入张量按照指定的比例进行上采样
static void ggml_compute_forward_upscale_f32(
    const struct ggml_compute_params * params,  // 输入参数结构体指针
    const struct ggml_tensor * src0,  // 输入张量指针
    struct ggml_tensor * dst) {  // 输出张量指针

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  // 如果参数类型为初始化或者结束，则直接返回
        return;
    }

    GGML_ASSERT(src0->nb[0] == sizeof(float));  // 断言输入张量的第一个维度大小为float类型的大小

    const int ith = params->ith;  // 获取参数结构体中的ith值

    GGML_TENSOR_UNARY_OP_LOCALS  // 定义张量一元操作的本地变量

    const int scale_factor = dst->op_params[0];  // 获取输出张量的操作参数中的比例因子

    // TODO: optimize  // 待优化的部分

    for (int i03 = 0; i03 < ne03; i03++) {  // 遍历第三维度
        for (int i02 = ith; i02 < ne02; i02++) {  // 遍历第二维度
            for (int m = 0; m < dst->ne[1]; m++) {  // 遍历输出张量的第一维度
                int i01 = m / scale_factor;  // 计算上采样后的第二维度索引
                for (int n = 0; n < dst->ne[0]; n++) {  // 遍历输出张量的第零维度
                    int i00 = n / scale_factor;  // 计算上采样后的第一维度索引

                    const float * x = (float *)((char *) src0->data + i00 * nb00 +i01 * nb01 + i02 * nb02 + i03 * nb03);  // 获取输入张量中的数据指针

                    float * y = (float *)((char *) dst->data + n * dst->nb[0] + m * dst->nb[1] + i02 * dst->nb[2] + i03 * dst->nb[3]);  // 获取输出张量中的数据指针

                    *y = *x;  // 将输入数据复制到输出数据中
                }
            }
        }
    }
}

static void ggml_compute_forward_upscale(
    const struct ggml_compute_params * params,  // 输入参数结构体指针
    const struct ggml_tensor * src0,  // 输入张量指针
    struct ggml_tensor * dst) {  // 输出张量指针
    switch (src0->type) {  // 根据输入张量的类型进行切换
        case GGML_TYPE_F32:  // 如果类型为float
            {
                ggml_compute_forward_upscale_f32(params, src0, dst);  // 调用上采样函数
            } break;
        default:  // 默认情况
            {
                GGML_ASSERT(false);  // 断言为假
            } break;
    }
}

// ggml_compute_forward_flash_attn

// 计算前向闪光注意力的函数，根据输入的查询、键、值张量计算注意力
static void ggml_compute_forward_flash_attn_f32(
        const struct ggml_compute_params * params,  // 输入参数结构体指针
        const struct ggml_tensor * q,  // 查询张量指针
        const struct ggml_tensor * k,  // 键张量指针
        const struct ggml_tensor * v,  // 值张量指针
        const bool masked,  // 是否进行掩码
        struct ggml_tensor * dst) {  // 输出张量指针
    int64_t t0 = ggml_perf_time_us();  // 获取当前时间
    UNUSED(t0);  // 不使用t0

    GGML_TENSOR_LOCALS(int64_t, neq, q,   ne)  // 定义查询张量的本地变量
    GGML_TENSOR_LOCALS(size_t,  nbq, q,   nb)  // 定义查询张量的本地变量
    // 定义并初始化 nek 变量，表示 neq0 的值
    GGML_TENSOR_LOCALS(int64_t, nek, k,   ne)
    // 定义并初始化 nbk 变量，表示 neq1 的值
    GGML_TENSOR_LOCALS(size_t,  nbk, k,   nb)
    // 定义并初始化 nev 变量，表示 neq1 的值
    GGML_TENSOR_LOCALS(int64_t, nev, v,   ne)
    // 定义并初始化 nbv 变量，表示 neq1 的值
    GGML_TENSOR_LOCALS(size_t,  nbv, v,   nb)
    // 定义并初始化 ne 变量，表示 dst 的值
    GGML_TENSOR_LOCALS(int64_t, ne,  dst, ne)
    // 定义并初始化 nb 变量，表示 dst 的值
    GGML_TENSOR_LOCALS(size_t,  nb,  dst, nb)

    // 获取参数中的 ith 值
    const int ith = params->ith;
    // 获取参数中的 nth 值
    const int nth = params->nth;

    // 获取 neq0 的值，赋给 D
    const int64_t D = neq0;
    // 获取 neq1 的值，赋给 N
    const int64_t N = neq1;
    // 计算 nek1 - N 的值，赋给 P
    const int64_t P = nek1 - N;
    // 计算 P + N 的值，赋给 M
    const int64_t M = P + N;

    // 计算 M 的上界，赋给 Mup
    const int Mup = ggml_up(M, GGML_SOFT_MAX_UNROLL);

    // 断言 ne0 等于 D
    GGML_ASSERT(ne0 == D);
    // 断言 ne1 等于 N
    GGML_ASSERT(ne1 == N);
    // 断言 P 大于等于 0
    GGML_ASSERT(P >= 0);

    // 断言 nbq0 的值等于 sizeof(float)
    GGML_ASSERT(nbq0 == sizeof(float));
    // 断言 nbk0 的值等于 sizeof(float)
    GGML_ASSERT(nbk0 == sizeof(float));
    // 断言 nbv0 的值等于 sizeof(float)
    GGML_ASSERT(nbv0 == sizeof(float));

    // 断言 neq0 的值等于 D
    GGML_ASSERT(neq0 == D);
    // 断言 nek0 的值等于 D
    GGML_ASSERT(nek0 == D);
    // 断言 nev1 的值等于 D
    GGML_ASSERT(nev1 == D);

    // 断言 neq1 的值等于 N
    GGML_ASSERT(neq1 == N);
    // 断言 nek1 的值等于 N + P
    GGML_ASSERT(nek1 == N + P);
    // 断言 nev1 的值等于 D
    GGML_ASSERT(nev1 == D);

    // 断言 nb0 的值等于 sizeof(float)
    GGML_ASSERT(nb0 == sizeof(float);
    // 断言 nb0 小于等于 nb1
    GGML_ASSERT(nb0 <= nb1);
    // 断言 nb1 小于等于 nb2
    GGML_ASSERT(nb1 <= nb2);
    // 断言 nb2 小于等于 nb3
    GGML_ASSERT(nb2 <= nb3);

    // 如果参数中的 type 值为 GGML_TASK_INIT，则返回
    if (params->type == GGML_TASK_INIT) {
        return;
    }

    // 如果参数中的 type 值为 GGML_TASK_FINALIZE，则返回
    if (params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 并行化，使用 ggml_vec_dot_f32 按 q 行
    // 计算 q 中的总行数
    const int nr = neq1*neq2*neq3;
    // 每个线程的行数
    const int dr = (nr + nth - 1)/nth;
    // 该线程的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 计算缩放比例
    const float scale = 1.0f/sqrtf(D);

    //printf("P=%d N=%d D=%d ir0=%d ir1=%d scale = %f\n", P, N, D, ir0, ir1, scale);
    // 遍历 ir0 到 ir1 之间的整数
    for (int ir = ir0; ir < ir1; ++ir) {
        // 计算 q 索引
        const int iq3 = ir/(neq2*neq1);
        const int iq2 = (ir - iq3*neq2*neq1)/neq1;
        const int iq1 = (ir - iq3*neq2*neq1 - iq2*neq1);

        // 获取参数中的 S 数组
        float * S = (float *) params->wdata + ith*(Mup + CACHE_LINE_SIZE_F32);

        // 将 S 数组中 M 到 Mup 之间的元素设置为负无穷
        for (int i = M; i < Mup; ++i) {
            S[i] = -INFINITY;
        }

        // 计算 masked_begin 的值
        const int64_t masked_begin = masked ? (P + iq1 + 1) : M;
        // 遍历 0 到 masked_begin 之间的整数
        for (int64_t ic = 0; ic < masked_begin; ++ic) {
            // 计算 k 索引
            const int ik3 = iq3;
            const int ik2 = iq2 % nek2;
            const int ik1 = ic;

            // 计算 S 索引
            const int i1 = ik1;

            // 调用 ggml_vec_dot_f32 函数进行向量点积运算
            ggml_vec_dot_f32(neq0,
                    S + i1,
                    (float *) ((char *) k->data + (ik1*nbk1 + ik2*nbk2 + ik3*nbk3)),
                    (float *) ((char *) q->data + (iq1*nbq1 + iq2*nbq2 + iq3*nbq3)));
        }

        // 调用 ggml_vec_scale_f32 函数对 S 数组进行缩放
        ggml_vec_scale_f32(masked_begin, S, scale);

        // 将 S 数组中 masked_begin 到 M 之间的元素设置为负无穷
        for (int64_t i = masked_begin; i < M; i++) {
            S[i] = -INFINITY;
        }

        // softmax 操作
        // 从 max 和循环中排除已知的 -INF S[..] 值
        // 不要忘记将它们的 SW 值设置为零
        {
            // 初始化 max 为负无穷
            float max = -INFINITY;
            // 调用 ggml_vec_max_f32 函数找到 S 数组中的最大值
            ggml_vec_max_f32(masked_begin, &max, S);

            // 初始化 sum 为 0.0
            ggml_float sum = 0.0;
#ifdef GGML_SOFT_MAX_ACCELERATE
                // 如果定义了 GGML_SOFT_MAX_ACCELERATE，则取相反数
                max = -max;
                // 将 S 中的每个元素加上 max
                vDSP_vsadd(S, 1, &max, S, 1, Mup);
                // 对 S 中的每个元素进行指数运算
                vvexpf(S, S, &Mup);
                // 计算 Mup 中所有元素的和
                ggml_vec_sum_f32(Mup, &sum, S);
#else
                // 定义一个 uint16_t 数组 scvt，长度为 GGML_SOFT_MAX_UNROLL
                uint16_t   scvt[GGML_SOFT_MAX_UNROLL]; UNUSED(scvt);
                // 定义一个长度为 GGML_SOFT_MAX_UNROLL 的浮点数数组 sump，并初始化为 0.0
                ggml_float sump[GGML_SOFT_MAX_UNROLL] = { 0.0 };

                // 循环，每次增加 GGML_SOFT_MAX_UNROLL
                for (int i = 0; i < Mup; i += GGML_SOFT_MAX_UNROLL) {
                    // 如果 i 大于等于 masked_begin，则跳出循环
                    if (i >= masked_begin) {
                        break;
                    }
                    // 定义一个指向 S[i] 的指针 SS
                    float * SS = S + i;

                    // 循环，每次增加 1
                    for (int j = 0; j < GGML_SOFT_MAX_UNROLL; ++j) {
                        // 如果 i+j 大于等于 masked_begin，则跳出循环
                        if (i + j >= masked_begin) {
                            break;
                        } 
                        // 如果 SS[j] 等于负无穷，则将其赋值为 0.0
                        else if (SS[j] == -INFINITY) {
                            SS[j] = 0.0f;
                        } 
                        // 否则
                        else {
                            // 如果未定义 GGML_FLASH_ATTN_EXP_FP16，则计算 expf(SS[j] - max)
#ifndef GGML_FLASH_ATTN_EXP_FP16
                            const float val = expf(SS[j] - max);
                            // 将 val 加到 sump[j] 上
                            sump[j] += (ggml_float)val;
                            // 将 SS[j] 赋值为 val
                            SS[j] = val;
#else
                            // 否则，将 SS[j] - max 转换为 GGML_FP16 类型，并存储到 scvt[j] 中
                            ggml_fp16_t s = GGML_FP32_TO_FP16(SS[j] - max);
                            memcpy(&scvt[j], &s, sizeof(uint16_t));
                            // 从 ggml_table_exp_f16 中取出 scvt[j] 对应的值，并转换为 GGML_FP32 类型，存储到 val 中
                            const float val = GGML_FP16_TO_FP32(ggml_table_exp_f16[scvt[j]]);
                            // 将 val 加到 sump[j] 上
                            sump[j] += (ggml_float)val;
                            // 将 SS[j] 赋值为 val
                            SS[j] = val;
#endif
                        }
                    }
                }

                // 循环，每次增加 1，将 sump 中的每个元素加到 sum 上
                for (int i = 0; i < GGML_SOFT_MAX_UNROLL; i++) {
                    sum += sump[i];
                }
#endif
            }

            // 断言 sum 大于 0.0
            assert(sum > 0.0);

            // 计算 sum 的倒数，并赋值给 sum
            sum = 1.0/sum;
            // 将 S 中的前 masked_begin 个元素乘以 sum
            ggml_vec_scale_f32(masked_begin, S, sum);

#ifndef NDEBUG
            // 循环，每次增加 1，断言 S[i] 不是 NaN 且不是无穷大
            for (int i = 0; i < masked_begin; ++i) {
                assert(!isnan(S[i]));
                assert(!isinf(S[i]));
            }
#endif
        }

        for (int64_t ic = 0; ic < nev1; ++ic) {
            // dst indices
            // 定义目标张量的索引
            const int i1 = iq1;
            const int i2 = iq2;
            const int i3 = iq3;

            // v indices
            // 定义v张量的索引
            const int iv2 = iq2 % nev2;
            const int iv3 = iq3;

            // 调用ggml_vec_dot_f32函数，计算两个向量的点积
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

    // 获取参数中的ith和nth值
    const int ith = params->ith;
    const int nth = params->nth;

    // 定义并初始化一系列常量
    const int64_t D = neq0;
    const int64_t N = neq1;
    const int64_t P = nek1 - N;
    const int64_t M = P + N;

    const int Mup = ggml_up(M, GGML_SOFT_MAX_UNROLL);

    // 断言，确保一些条件成立
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

    // 断言，确保dst张量不能被转置或置换
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

    // 使用 ggml_vec_dot_f32 并行化按 q 行进行计算

    // q 中的总行数
    const int nr = neq1*neq2*neq3;

    // 每个线程的行数
    const int dr = (nr + nth - 1)/nth;

    // 该线程的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 缩放因子
    const float scale = 1.0f/sqrtf(D);

    //printf("P=%d N=%d D=%d ir0=%d ir1=%d scale = %f\n", P, N, D, ir0, ir1, scale);
#ifdef GGML_SOFT_MAX_ACCELERATE
                // 如果定义了 GGML_SOFT_MAX_ACCELERATE，则取相反数
                max = -max;
                // 将 max 加到 S 中的每个元素上
                vDSP_vsadd(S, 1, &max, S, 1, Mup);
                // 对 S 中的每个元素进行指数运算
                vvexpf(S, S, &Mup);
                // 计算 Mup 中所有元素的和
                ggml_vec_sum_f32(Mup, &sum, S);
#else
                // 定义一个长度为 GGML_SOFT_MAX_UNROLL 的无符号 16 位整型数组
                uint16_t   scvt[GGML_SOFT_MAX_UNROLL];
                // 定义一个长度为 GGML_SOFT_MAX_UNROLL 的浮点型数组，并初始化为 0.0
                ggml_float sump[GGML_SOFT_MAX_UNROLL] = { 0.0 };

                // 循环，每次增加 GGML_SOFT_MAX_UNROLL
                for (int i = 0; i < Mup; i += GGML_SOFT_MAX_UNROLL) {
                    // 定义一个指向 S[i] 的指针
                    float * SS = S + i;

                    // 循环，每次增加 1
                    for (int j = 0; j < GGML_SOFT_MAX_UNROLL; ++j) {
                        // 如果 SS[j] 等于负无穷大，则将其赋值为 0.0
                        if (SS[j] == -INFINITY) {
                            SS[j] = 0.0f;
                        } else {
                            // 将 SS[j] 减去 max，转换成 GGML_FP16 类型，并拷贝到 scvt[j] 中
                            ggml_fp16_t s = GGML_FP32_TO_FP16(SS[j] - max);
                            memcpy(&scvt[j], &s, sizeof(uint16_t));
                            // 从 ggml_table_exp_f16 中取出对应的值，并将其转换成浮点型，加到 sump[j] 上
                            const float val = GGML_FP16_TO_FP32(ggml_table_exp_f16[scvt[j]]);
                            sump[j] += (ggml_float)val;
                            // 将 SS[j] 赋值为 val
                            SS[j] = val;
                        }
                    }
                }

                // 循环，每次增加 1，将 sump 中的值加到 sum 上
                for (int i = 0; i < GGML_SOFT_MAX_UNROLL; i++) {
                    sum += sump[i];
                }
#endif
            }

            // 断言 sum 大于 0.0
            assert(sum > 0.0);

            // 计算 sum 的倒数，并将结果赋值给 sum
            sum = 1.0/sum;
            // 将 S 中的每个元素乘以 sum
            ggml_vec_scale_f32(M, S, sum);

#ifndef NDEBUG
            // 调试模式下，循环，每次增加 1，断言 S[i] 不是 NaN 和无穷大
            for (int i = 0; i < M; ++i) {
                assert(!isnan(S[i]));
                assert(!isinf(S[i]));
            }
#endif
        }

        ggml_fp16_t * S16 = (ggml_fp16_t *) ((float *) params->wdata + ith*(2*Mup + CACHE_LINE_SIZE_F32) + Mup);

        for (int64_t i = 0; i < M; i++) {
            S16[i] = GGML_FP32_TO_FP16(S[i]);
        }

        // todo: exclude known zero S[..] values from dot (reducing nev0 and increasing begin of v and S16).
        if (GGML_VEC_DOT_UNROLL == 1 || (nev1 % GGML_VEC_DOT_UNROLL != 0)) {
            for (int64_t ic = 0; ic < nev1; ++ic) {
                // dst indices
                const int i1 = iq1;
                const int i2 = iq2;
                const int i3 = iq3;

                // v indices
                const int iv2 = iq2 % nev2;
                const int iv3 = iq3;

                ggml_vec_dot_f16(nev0,
                        (float *)       ((char *) dst->data + (ic*nb0 + i1*nb1  + i2*nb2   + i3*nb3)),
                        (ggml_fp16_t *) ((char *) v->data   + (         ic*nbv1 + iv2*nbv2 + iv3*nbv3)),
                        S16);
            }
        } else {
            for (int64_t ic = 0; ic < nev1; ic += GGML_VEC_DOT_UNROLL) {
                // dst indices
                const int i1 = iq1;
                const int i2 = iq2;
                const int i3 = iq3;

                // v indices
                const int iv2 = iq2 % nev2;
                const int iv3 = iq3;

                ggml_vec_dot_f16_unroll(nev0, nbv1,
                        (float *) ((char *) dst->data + (ic*nb0 + i1*nb1  + i2*nb2   + i3*nb3)),
                        ((char *)             v->data + (         ic*nbv1 + iv2*nbv2 + iv3*nbv3)),
                        S16);
            }
        }
    }
}

static void ggml_compute_forward_flash_attn(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * q,
        const struct ggml_tensor * k,
        const struct ggml_tensor * v,
        const bool masked,
        struct ggml_tensor * dst) {
    # 根据输入的类型进行不同的操作
    switch (q->type) {
        # 如果类型为 GGML_TYPE_F16，则调用 ggml_compute_forward_flash_attn_f16 函数
        case GGML_TYPE_F16:
            {
                ggml_compute_forward_flash_attn_f16(params, q, k, v, masked, dst);
            } break;
        # 如果类型为 GGML_TYPE_F32，则调用 ggml_compute_forward_flash_attn_f32 函数
        case GGML_TYPE_F32:
            {
                ggml_compute_forward_flash_attn_f32(params, q, k, v, masked, dst);
            } break;
        # 如果类型不是上述两种类型，则触发断言错误
        default:
            {
                GGML_ASSERT(false);
            } break;
    }
// 计算前向传播的闪存前馈
static void ggml_compute_forward_flash_ff_f16(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * a,  // F16
        const struct ggml_tensor * b0, // F16 fc_w
        const struct ggml_tensor * b1, // F32 fc_b
        const struct ggml_tensor * c0, // F16 proj_w
        const struct ggml_tensor * c1, // F32 proj_b
        struct ggml_tensor * dst) {
    int64_t t0 = ggml_perf_time_us();  // 记录当前时间
    UNUSED(t0);  // 不使用 t0 变量

    GGML_TENSOR_LOCALS(int64_t, nea,  a,   ne)  // 定义 nea 变量
    GGML_TENSOR_LOCALS(size_t,  nba,  a,   nb)  // 定义 nba 变量
    GGML_TENSOR_LOCALS(int64_t, neb0, b0,  ne)  // 定义 neb0 变量
    GGML_TENSOR_LOCALS(size_t,  nbb0, b0,  nb)  // 定义 nbb0 变量
    GGML_TENSOR_LOCALS(int64_t, neb1, b1,  ne)  // 定义 neb1 变量
    GGML_TENSOR_LOCALS(size_t,  nbb1, b1,  nb)  // 定义 nbb1 变量
    GGML_TENSOR_LOCALS(int64_t, nec0, c0,  ne)  // 定义 nec0 变量
    GGML_TENSOR_LOCALS(size_t,  nbc0, c0,  nb)  // 定义 nbc0 变量
    GGML_TENSOR_LOCALS(int64_t, nec1, c1,  ne)  // 定义 nec1 变量
    GGML_TENSOR_LOCALS(size_t,  nbc1, c1,  nb)  // 定义 nbc1 变量
    GGML_TENSOR_LOCALS(int64_t, ne,   dst, ne)  // 定义 ne 变量
    GGML_TENSOR_LOCALS(size_t,  nb,   dst, nb)  // 定义 nb 变量

    const int ith = params->ith;  // 获取 params 结构体中的 ith 变量
    const int nth = params->nth;  // 获取 params 结构体中的 nth 变量

    const int64_t D = nea0;  // 获取 nea0 变量并赋值给 D
    //const int64_t N = nea1;  // 注释掉的代码
    const int64_t M = neb01;  // 获取 neb01 变量并赋值给 M

    GGML_ASSERT(ne0 == nea0);  // 断言 ne0 等于 nea0
    GGML_ASSERT(ne1 == nea1);  // 断言 ne1 等于 nea1
    GGML_ASSERT(ne2 == nea2);  // 断言 ne2 等于 nea2

    GGML_ASSERT(nba0  == sizeof(ggml_fp16_t));  // 断言 nba0 等于 ggml_fp16_t 的大小
    GGML_ASSERT(nbb00 == sizeof(ggml_fp16_t));  // 断言 nbb00 等于 ggml_fp16_t 的大小
    GGML_ASSERT(nbb10 == sizeof(float));  // 断言 nbb10 等于 float 的大小
    GGML_ASSERT(nbc00 == sizeof(ggml_fp16_t));  // 断言 nbc00 等于 ggml_fp16_t 的大小
    GGML_ASSERT(nbc10 == sizeof(float));  // 断言 nbc10 等于 float 的大小

    GGML_ASSERT(neb00 == D);  // 断言 neb00 等于 D
    GGML_ASSERT(neb01 == M);  // 断言 neb01 等于 M
    GGML_ASSERT(neb10 == M);  // 断言 neb10 等于 M
    GGML_ASSERT(neb11 == 1);  // 断言 neb11 等于 1

    GGML_ASSERT(nec00 == M);  // 断言 nec00 等于 M
    GGML_ASSERT(nec01 == D);  // 断言 nec01 等于 D
    GGML_ASSERT(nec10 == D);  // 断言 nec10 等于 D
    GGML_ASSERT(nec11 == 1);  // 断言 nec11 等于 1

    // dst 不能被转置或排列
    GGML_ASSERT(nb0 == sizeof(float));  // 断言 nb0 等于 float 的大小
    GGML_ASSERT(nb0 <= nb1);  // 断言 nb0 小于等于 nb1
    GGML_ASSERT(nb1 <= nb2);  // 断言 nb1 小于等于 nb2
    GGML_ASSERT(nb2 <= nb3);  // 断言 nb2 小于等于 nb3

    if (params->type == GGML_TASK_INIT) {  // 如果 params 结构体中的 type 等于 GGML_TASK_INIT
        return;  // 直接返回
    }
}
    // 如果任务类型为 GGML_TASK_FINALIZE，则直接返回，不进行并行化处理
    if (params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 使用 ggml_vec_dot_f32 函数按行并行处理

    // 计算矩阵 a 的总行数
    const int nr = nea1*nea2*nea3;

    // 计算每个线程处理的行数
    const int dr = (nr + nth - 1)/nth;

    // 计算当前线程处理的行范围
    const int ir0 = dr*ith;
    const int ir1 = MIN(ir0 + dr, nr);
    for (int ir = ir0; ir < ir1; ++ir) {
        // 遍历 ir0 到 ir1 之间的整数
        // a indices
        // 计算 a 的索引
        const int ia3 = ir/(nea2*nea1);
        const int ia2 = (ir - ia3*nea2*nea1)/nea1;
        const int ia1 = (ir - ia3*nea2*nea1 - ia2*nea1);

        float * S = (float *) params->wdata + ith*(2*M + CACHE_LINE_SIZE_F32);

        for (int64_t ic = 0; ic < neb01; ++ic) {
            // 遍历 0 到 neb01 之间的整数
            // b0 indices
            // 计算 b0 的索引
            const int ib03 = ia3;
            const int ib02 = ia2;
            const int ib01 = ic;

            // S indices
            // 计算 S 的索引
            const int i1 = ib01;

            ggml_vec_dot_f16(nea0,
                    S + i1,
                    (ggml_fp16_t *) ((char *) b0->data + (ib01*nbb01 + ib02*nbb02 + ib03*nbb03)),
                    (ggml_fp16_t *) ((char *)  a->data + ( ia1*nba1  +  ia2*nba2  +  ia3*nba3)));
        }

        ggml_vec_add_f32(neb01, S, S, (float *) b1->data);
        //ggml_vec_gelu_f32(neb01, S, S);

        ggml_fp16_t * S16 = (ggml_fp16_t *) ((float *) params->wdata + ith*(2*M + CACHE_LINE_SIZE_F32) + M);

        for (int64_t i = 0; i < M; i++) {
            // 遍历 0 到 M 之间的整数
            S16[i] = GGML_FP32_TO_FP16(S[i]);
        }

        ggml_vec_gelu_f16(neb01, S16, S16);

        {
            // dst indices
            // 计算 dst 的索引
            const int i1 = ia1;
            const int i2 = ia2;
            const int i3 = ia3;

            for (int64_t ic = 0; ic < nec01; ++ic) {
                // 遍历 0 到 nec01 之间的整数
                ggml_vec_dot_f16(neb01,
                        (float *)       ((char *) dst->data + (ic*nb0 + i1*nb1   + i2*nb2   + i3*nb3)),
                        (ggml_fp16_t *) ((char *) c0->data  + (         ic*nbc01 + i2*nbc02 + i3*nbc03)),
                        S16);
            }

            ggml_vec_add_f32(nec01,
                    (float *) ((char *) dst->data + (i1*nb1 + i2*nb2 + i3*nb3)),
                    (float *) ((char *) dst->data + (i1*nb1 + i2*nb2 + i3*nb3)),
                    (float *) c1->data);
        }
    }
// 计算前向闪存注意力反向传播的函数，根据输入参数和张量计算结果并存储到目标张量中
static void ggml_compute_forward_flash_ff(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * a,
        const struct ggml_tensor * b0,
        const struct ggml_tensor * b1,
        const struct ggml_tensor * c0,
        const struct ggml_tensor * c1,
        struct ggml_tensor * dst) {
    // 根据 b0 张量的类型进行不同的处理
    switch (b0->type) {
        // 如果 b0 的类型为 GGML_TYPE_F16
        case GGML_TYPE_F16:
            {
                // 调用 ggml_compute_forward_flash_ff_f16 函数进行计算
                ggml_compute_forward_flash_ff_f16(params, a, b0, b1, c0, c1, dst);
            } break;
        // 如果 b0 的类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 抛出断言错误，表示该部分功能尚未实现
                GGML_ASSERT(false); // TODO
            } break;
        // 其他情况
        default:
            {
                // 抛出断言错误，表示不支持的类型
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_flash_attn_back

// 计算前向闪存注意力反向传播的函数，针对 F32 类型的张量进行计算
static void ggml_compute_forward_flash_attn_back_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * q,
        const struct ggml_tensor * k,
        const struct ggml_tensor * v,
        const struct ggml_tensor * d,
        const bool masked,
              struct ggml_tensor * dst) {
    // 获取当前时间并标记为未使用
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

    // 计算一系列变量的值
    const int64_t D = neq0;
    const int64_t N = neq1;
    const int64_t P = nek1 - N;
    const int64_t M = P + N;

    const int Mup  = ggml_up(M, GGML_SOFT_MAX_UNROLL);
    const int mxDM = MAX(D, Mup);

    // 抛出断言错误，表示不支持的情况
    GGML_ASSERT(P >= 0);

    // 抛出断言错误，表示不支持的情况
    GGML_ASSERT(nbq0 == sizeof(float));
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
    // 确保 nb0 小于等于 nb1
    GGML_ASSERT(nb0 <= nb1);
    // 确保 nb1 小于等于 nb2
    GGML_ASSERT(nb1 <= nb2);
    // 确保 nb2 小于等于 nb3
    GGML_ASSERT(nb2 <= nb3);

    // 如果任务类型为初始化
    if (params->type == GGML_TASK_INIT) {
        // 如果是第一个线程
        if (ith == 0) {
            // 将 dst->data 的内容全部置为 0
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

    // 计算一些大小
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
    printf("P=%d N=%d D=%d ir0=%d ir1=%d scale = %f\n", P, N, D, ir0, ir1, scale);

    // 计算 k2（和 v2）在 q2 中重复出现的次数
    int nrep = neq2/nek2;
#ifdef GGML_SOFT_MAX_ACCELERATE
                        // 如果定义了 GGML_SOFT_MAX_ACCELERATE，则取最大值的相反数
                        max = -max;
                        // 将 SM 中的每个元素加上 max
                        vDSP_vsadd(SM, 1, &max, SM, 1, Mup);
                        // 对 SM 中的每个元素进行指数运算
                        vvexpf(SM, SM, &Mup);
                        // 计算 Mup 中所有元素的和
                        ggml_vec_sum_f32(Mup, &sum, SM);
#else
                        // 定义一个 uint16_t 数组 scvt，但未使用
                        uint16_t   scvt[GGML_SOFT_MAX_UNROLL]; UNUSED(scvt);
                        // 定义一个长度为 GGML_SOFT_MAX_UNROLL 的浮点数数组 sump，并初始化为 0.0
                        ggml_float sump[GGML_SOFT_MAX_UNROLL] = { 0.0 };

                        // 循环遍历 Mup，每次增加 GGML_SOFT_MAX_UNROLL
                        for (int i = 0; i < Mup; i += GGML_SOFT_MAX_UNROLL) {
                            // 如果 i 大于等于 masked_begin，则跳出循环
                            if (i >= masked_begin) {
                                break;
                            }
                            // 定义指向 S[i] 和 SM[i] 的指针
                            float * SR =  S + i;
                            float * SW = SM + i;

                            // 循环遍历 GGML_SOFT_MAX_UNROLL 次
                            for (int j = 0; j < GGML_SOFT_MAX_UNROLL; ++j) {
                                // 如果 i+j 大于等于 masked_begin，则跳出循环
                                if (i + j >= masked_begin) {
                                    break;
                                } 
                                // 如果 SR[j] 等于负无穷，则将 SW[j] 置为 0.0
                                else if (SR[j] == -INFINITY) {
                                    SW[j] = 0.0f;
                                } 
                                // 否则根据条件编译选择不同的指数运算方式，并更新 sump[j] 和 SW[j]
                                else {
#ifndef GGML_FLASH_ATTN_EXP_FP16
                                    const float val = expf(SR[j] - max);
#else
                                    ggml_fp16_t s = GGML_FP32_TO_FP16(SR[j] - max);
                                    memcpy(&scvt[j], &s, sizeof(uint16_t));
                                    const float val = GGML_FP16_TO_FP32(ggml_table_exp_f16[scvt[j]]);
#endif
                                    sump[j] += (ggml_float)val;
                                    SW[j] = val;
                                }
                            }
                        }

                        // 循环遍历 GGML_SOFT_MAX_UNROLL 次，将 sump 中的值加到 sum 上
                        for (int i = 0; i < GGML_SOFT_MAX_UNROLL; i++) {
                            sum += sump[i];
                        }
    }
}
# 计算前向闪存注意力反向传播
def ggml_compute_forward_flash_attn_back(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * q,
        const struct ggml_tensor * k,
        const struct ggml_tensor * v,
        const struct ggml_tensor * d,
        const bool masked,
        struct ggml_tensor * dst) {
    # 根据输入张量 q 的类型进行不同的处理
    switch (q->type) {
        # 如果类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                # 调用 ggml_compute_forward_flash_attn_back_f32 函数处理
                ggml_compute_forward_flash_attn_back_f32(params, q, k, v, d, masked, dst);
            } break;
        # 如果类型不是 GGML_TYPE_F32
        default:
            {
                # 抛出断言错误
                GGML_ASSERT(false);
            } break;
    }
}

# ggml_compute_forward_win_part

# 计算前向窗口部分的 f32 类型处理
static void ggml_compute_forward_win_part_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    # 如果任务类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    # 定义局部变量 ne0 和 ne，分别表示 src0 和 dst 的 ne 属性
    GGML_TENSOR_LOCALS(int64_t, ne0, src0, ne)
    GGML_TENSOR_LOCALS(int64_t, ne,  dst,  ne)

    # 从 dst 的操作参数中获取 nep0、nep1 和 w
    const int32_t nep0 = ((const int32_t *)(dst->op_params))[0];
    const int32_t nep1 = ((const int32_t *)(dst->op_params))[1];
    const int32_t w    = ((const int32_t *)(dst->op_params))[2];

    # 断言 ne0 等于 ne0，ne3 等于 nep0*nep1
    assert(ne00 == ne0);
    assert(ne3  == nep0*nep1);

    # TODO: 优化 / 多线程处理
}
    # 遍历第一个维度的索引，范围是从 0 到 nep1-1
    for (int py = 0; py < nep1; ++py) {
        # 遍历第二个维度的索引，范围是从 0 到 nep0-1
        for (int px = 0; px < nep0; ++px) {
            # 计算三维索引 i3
            const int64_t i3 = py*nep0 + px;
            # 遍历第三个维度的索引，范围是从 0 到 ne2-1
            for (int64_t i2 = 0; i2 < ne2; ++i2) {
                # 遍历第四个维度的索引，范围是从 0 到 ne1-1
                for (int64_t i1 = 0; i1 < ne1; ++i1) {
                    # 遍历第五个维度的索引，范围是从 0 到 ne0-1
                    for (int64_t i0 = 0; i0 < ne0; ++i0) {
                        # 计算二维索引 i02
                        const int64_t i02 = py*w + i2;
                        # 计算一维索引 i01
                        const int64_t i01 = px*w + i1;
                        # 计算零维索引 i00
                        const int64_t i00 = i0;

                        # 计算一维索引 i
                        const int64_t i = i3*ne2*ne1*ne0 + i2*ne1*ne0    + i1*ne0   + i0;
                        # 计算一维索引 j
                        const int64_t j =                  i02*ne01*ne00 + i01*ne00 + i00;

                        # 检查索引是否超出范围，如果超出则将目标数组中的值设为 0.0f
                        if (py*w + i2 >= ne02 || px*w + i1 >= ne01) {
                            ((float *) dst->data)[i] = 0.0f;
                        } else {
                            # 否则将源数组中的值赋给目标数组
                            ((float *) dst->data)[i] = ((float *) src0->data)[j];
                        }
                    }
                }
            }
        }
    }
// 计算前向窗口非分块操作
static void ggml_compute_forward_win_part(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 根据输入张量的类型进行不同的操作
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用 ggml_compute_forward_win_part_f32 函数进行计算
                ggml_compute_forward_win_part_f32(params, src0, dst);
            } break;
        // 如果输入张量类型不是 GGML_TYPE_F32
        default:
            {
                // 断言错误
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_win_unpart

// 计算前向窗口非分块操作（针对 F32 类型的张量）
static void ggml_compute_forward_win_unpart_f32(
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

    // 从 dst 的操作参数中获取 w 的值
    const int32_t w = ((const int32_t *)(dst->op_params))[0];

    // 计算 padding
    const int px = (w - ne1%w)%w;
    //const int py = (w - ne2%w)%w;

    const int npx = (px + ne1)/w;
    //const int npy = (py + ne2)/w;

    // 断言 ne0 等于 ne00
    assert(ne0 == ne00);

    // 待优化 / 多线程
    // 遍历张量元素进行计算
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

                // 将 src0 的数据复制到 dst 中
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
// 计算前向一元操作，根据参数和输入张量计算输出张量
static void ggml_compute_forward_unary(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    // 获取一元操作类型
    const enum ggml_unary_op op = ggml_get_unary_op(dst);

    // 根据不同的一元操作类型进行不同的计算
    switch (op) {
        case GGML_UNARY_OP_ABS:
            {
                ggml_compute_forward_abs(params, src0, dst);
            } break;
        case GGML_UNARY_OP_SGN:
            {
                ggml_compute_forward_sgn(params, src0, dst);
            } break;
        case GGML_UNARY_OP_NEG:
            {
                ggml_compute_forward_neg(params, src0, dst);
            } break;
        case GGML_UNARY_OP_STEP:
            {
                ggml_compute_forward_step(params, src0, dst);
            } break;
        case GGML_UNARY_OP_TANH:
            {
                ggml_compute_forward_tanh(params, src0, dst);
            } break;
        case GGML_UNARY_OP_ELU:
            {
                ggml_compute_forward_elu(params, src0, dst);
            } break;
        case GGML_UNARY_OP_RELU:
            {
                ggml_compute_forward_relu(params, src0, dst);
            } break;
        case GGML_UNARY_OP_GELU:
            {
                ggml_compute_forward_gelu(params, src0, dst);
            } break;
        case GGML_UNARY_OP_GELU_QUICK:
            {
                ggml_compute_forward_gelu_quick(params, src0, dst);
            } break;
        case GGML_UNARY_OP_SILU:
            {
                ggml_compute_forward_silu(params, src0, dst);
            } break;
        case GGML_UNARY_OP_LEAKY:
            {
                ggml_compute_forward_leaky(params, src0, dst);
            } break;
        default:
            {
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_get_rel_pos
# 计算相对位置并将结果存储在目标张量中（使用 16 位浮点数）
static void ggml_compute_forward_get_rel_pos_f16(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    # 如果参数类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 引用：https://github.com/facebookresearch/segment-anything/blob/main/segment_anything/modeling/image_encoder.py#L292-L322

    GGML_TENSOR_UNARY_OP_LOCALS

    const int64_t w = ne1;

    ggml_fp16_t * src0_data = (ggml_fp16_t *) src0->data;
    ggml_fp16_t * dst_data  = (ggml_fp16_t *) dst->data;

    for (int64_t i2 = 0; i2 < ne2; ++i2) {
        for (int64_t i1 = 0; i1 < ne1; ++i1) {
            const int64_t pos = (w - i1 - 1) + i2;
            for (int64_t i0 = 0; i0 < ne0; ++i0) {
                dst_data[i2*ne1*ne0 + i1*ne0 + i0] = src0_data[pos*ne00 + i0];
            }
        }
    }
}

static void ggml_compute_forward_get_rel_pos(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst) {
    # 根据源张量的类型进行不同的处理
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

static void ggml_compute_forward_add_rel_pos_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        const struct ggml_tensor * src2,
        struct ggml_tensor * dst) {
    # 检查是否为原地操作
    const bool inplace = (bool) ((int32_t *) dst->op_params)[0];
    # 如果不是原地操作且参数类型为初始化，则将源张量的数据复制到目标张量中并返回
    if (!inplace && params->type == GGML_TASK_INIT) {
        memcpy((char *) dst->data, (char *) src0->data, ggml_nbytes(dst));
        return;
    }
    # 如果参数类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    int64_t t0 = ggml_perf_time_us();
    UNUSED(t0);
}
    // 将 src1 的数据类型转换为 float 指针类型
    float * src1_data = (float *) src1->data;
    // 将 src2 的数据类型转换为 float 指针类型
    float * src2_data = (float *) src2->data;
    // 将 dst 的数据类型转换为 float 指针类型
    float * dst_data  = (float *) dst->data;

    // 获取 src1 的各维度大小
    const int64_t ne10 = src1->ne[0];
    const int64_t ne11 = src1->ne[1];
    const int64_t ne12 = src1->ne[2];
    const int64_t ne13 = src1->ne[3];

    // 获取 params 结构体中的 ith 和 nth 值
    const int ith = params->ith;
    const int nth = params->nth;

    // 计算 dst 中的总补丁数
    const int np = ne13;

    // 计算每个线程处理的补丁数
    const int dp = (np + nth - 1)/nth;

    // 计算当前线程处理的补丁范围
    const int ip0 = dp*ith;
    const int ip1 = MIN(ip0 + dp, np);

    // 循环遍历每个补丁
    for (int64_t i13 = ip0; i13 < ip1; ++i13) {
        for (int64_t i12 = 0; i12 < ne12; ++i12) {
            for (int64_t i11 = 0; i11 < ne11; ++i11) {
                // 计算当前位置在一维数组中的索引
                const int64_t jp1 = i13*ne12*ne11*ne10 + i12*ne11*ne10 + i11*ne10;
                for (int64_t i10 = 0; i10 < ne10; ++i10) {
                    // 计算当前位置在一维数组中的索引
                    const int64_t jp0  = jp1 + i10;
                    // 获取 src1 和 src2 的值
                    const float src1_e = src1_data[jp0];
                    const float src2_e = src2_data[jp0];

                    // 计算当前位置在一维数组中的水平和垂直偏移
                    const int64_t jdh = jp0 * ne10;
                    const int64_t jdw = jdh - (ne10 - 1) * i10;

                    // 循环遍历每个像素，并更新 dst 的值
                    for (int64_t j = 0; j < ne10; ++j) {
                        dst_data[jdh + j     ] += src2_e;
                        dst_data[jdw + j*ne10] += src1_e;
                    }
                }
            }
        }
    }
// 计算前向传播时，根据输入张量 src0 的类型进行不同的操作
static void ggml_compute_forward_add_rel_pos(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        const struct ggml_tensor * src2,
        struct ggml_tensor * dst) {
    // 根据输入张量 src0 的类型进行不同的操作
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用 ggml_compute_forward_add_rel_pos_f32 函数进行相关操作
                ggml_compute_forward_add_rel_pos_f32(params, src0, src1, src2, dst);
            } break;
        // 如果输入张量类型不为 GGML_TYPE_F32
        default:
            {
                // 断言失败
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_map_unary

// 对输入张量 src0 进行一元操作
static void ggml_compute_forward_map_unary_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst,
        const ggml_unary_op_f32_t fun) {
    // 断言输入张量 src0 和目标张量 dst 的形状相同
    GGML_ASSERT(ggml_are_same_shape(src0, dst));

    // 如果任务类型为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取输入张量 src0 的行数
    const int n  = ggml_nrows(src0);
    // 获取输入张量 src0 的通道数
    const int nc = src0->ne[0];

    // 断言目标张量 dst 的第一个维度的字节数为 float 类型的字节数
    assert( dst->nb[0] == sizeof(float));
    // 断言输入张量 src0 的第一个维度的字节数为 float 类型的字节数
    assert(src0->nb[0] == sizeof(float));

    // 遍历输入张量 src0 的行
    for (int i = 0; i < n; i++) {
        // 调用 fun 函数对输入张量 src0 的数据进行一元操作，结果存入目标张量 dst
        fun(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1])));
    }
}

// 对输入张量 src0 进行一元操作
static void ggml_compute_forward_map_unary(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        struct ggml_tensor * dst,
        const ggml_unary_op_f32_t fun) {
    // 根据输入张量 src0 的类型进行不同的操作
    switch (src0->type) {
        // 如果输入张量类型为 GGML_TYPE_F32
        case GGML_TYPE_F32:
            {
                // 调用 ggml_compute_forward_map_unary_f32 函数进行相关操作
                ggml_compute_forward_map_unary_f32(params, src0, dst, fun);
            } break;
        // 如果输入张量类型不为 GGML_TYPE_F32
        default:
            {
                // 断言失败
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_map_binary
// 计算二进制操作的前向映射，输入参数包括计算参数、两个输入张量、一个输出张量和一个二进制操作函数
static void ggml_compute_forward_map_binary_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst,
        const ggml_binary_op_f32_t fun) {
    // 断言参数中的ith为0
    assert(params->ith == 0);
    // 断言输入张量src0、src1和输出张量dst具有相同的形状
    assert(ggml_are_same_shape(src0, src1) && ggml_are_same_shape(src0, dst));

    // 如果计算参数的类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 获取输入张量src0的行数
    const int n  = ggml_nrows(src0);
    // 获取输入张量src0的通道数
    const int nc = src0->ne[0];

    // 断言输出张量dst的第一个维度的字节数为float类型的字节数
    assert( dst->nb[0] == sizeof(float));
    // 断言输入张量src0的第一个维度的字节数为float类型的字节数
    assert(src0->nb[0] == sizeof(float));
    // 断言输入张量src1的第一个维度的字节数为float类型的字节数
    assert(src1->nb[0] == sizeof(float));

    // 遍历输入张量src0的行数，对每一行进行二进制操作
    for (int i = 0; i < n; i++) {
        fun(nc,
                (float *) ((char *) dst->data  + i*( dst->nb[1])),
                (float *) ((char *) src0->data + i*(src0->nb[1])),
                (float *) ((char *) src1->data + i*(src1->nb[1])));
    }
}

// 根据输入张量的类型选择相应的前向映射二进制操作函数
static void ggml_compute_forward_map_binary(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        struct ggml_tensor * dst,
        const ggml_binary_op_f32_t fun) {
    switch (src0->type) {
        // 如果输入张量的类型为GGML_TYPE_F32，则调用ggml_compute_forward_map_binary_f32函数
        case GGML_TYPE_F32:
            {
                ggml_compute_forward_map_binary_f32(params, src0, src1, dst, fun);
            } break;
        // 如果输入张量的类型不是GGML_TYPE_F32，则断言为假
        default:
            {
                GGML_ASSERT(false);
            } break;
    }
}

// ggml_compute_forward_map_custom1

// 计算自定义操作1的前向映射，输入参数包括计算参数、一个输入张量、一个输出张量和一个自定义操作1函数
static void ggml_compute_forward_map_custom1_f32(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * a,
        struct ggml_tensor * dst,
        const ggml_custom1_op_f32_t fun) {
    // 断言参数中的ith为0
    assert(params->ith == 0);

    // 如果计算参数的类型为初始化或结束，则直接返回
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 调用自定义操作1函数对输出张量dst进行操作
    fun(dst, a);
}

// ggml_compute_forward_map_custom2
// 定义一个函数，计算自定义操作的前向映射，输入参数为浮点数
static void ggml_compute_forward_map_custom2_f32(
        const struct ggml_compute_params * params,  // 输入参数：计算参数结构体指针
        const struct ggml_tensor * a,  // 输入参数：第一个张量指针
        const struct ggml_tensor * b,  // 输入参数：第二个张量指针
        struct ggml_tensor * dst,  // 输出参数：目标张量指针
        const ggml_custom2_op_f32_t fun) {  // 输入参数：自定义操作函数指针
    assert(params->ith == 0);  // 断言：计算参数结构体中的 ith 成员是否为 0

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  // 如果计算参数结构体中的 type 成员为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE
        return;  // 直接返回
    }

    fun(dst, a, b);  // 调用自定义操作函数，传入目标张量指针、第一个张量指针、第二个张量指针
}

// ggml_compute_forward_map_custom3

// 定义一个函数，计算自定义操作的前向映射，输入参数为浮点数
static void ggml_compute_forward_map_custom3_f32(
        const struct ggml_compute_params * params,  // 输入参数：计算参数结构体指针
        const struct ggml_tensor * a,  // 输入参数：第一个张量指针
        const struct ggml_tensor * b,  // 输入参数：第二个张量指针
        const struct ggml_tensor * c,  // 输入参数：第三个张量指针
        struct ggml_tensor * dst,  // 输出参数：目标张量指针
        const ggml_custom3_op_f32_t fun) {  // 输入参数：自定义操作函数指针
    assert(params->ith == 0);  // 断言：计算参数结构体中的 ith 成员是否为 0

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  // 如果计算参数结构体中的 type 成员为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE
        return;  // 直接返回
    }

    fun(dst, a, b, c);  // 调用自定义操作函数，传入目标张量指针、第一个张量指针、第二个张量指针、第三个张量指针
}

// ggml_compute_forward_map_custom1

// 定义一个函数，计算自定义操作的前向映射
static void ggml_compute_forward_map_custom1(
        const struct ggml_compute_params * params,  // 输入参数：计算参数结构体指针
        const struct ggml_tensor * a,  // 输入参数：第一个张量指针
              struct ggml_tensor * dst) {  // 输出参数：目标张量指针
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  // 如果计算参数结构体中的 type 成员为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE
        return;  // 直接返回
    }

    struct ggml_map_custom1_op_params * p = (struct ggml_map_custom1_op_params *) dst->op_params;  // 定义指向目标张量 op_params 成员的指针

    p->fun(dst, a, params->ith, params->nth, p->userdata);  // 调用自定义操作函数，传入目标张量指针、第一个张量指针、ith、nth、userdata
}

// ggml_compute_forward_map_custom2

// 定义一个函数，计算自定义操作的前向映射
static void ggml_compute_forward_map_custom2(
        const struct ggml_compute_params * params,  // 输入参数：计算参数结构体指针
        const struct ggml_tensor * a,  // 输入参数：第一个张量指针
        const struct ggml_tensor * b,  // 输入参数：第二个张量指针
              struct ggml_tensor * dst) {  // 输出参数：目标张量指针
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  // 如果计算参数结构体中的 type 成员为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE
        return;  // 直接返回
    }

    struct ggml_map_custom2_op_params * p = (struct ggml_map_custom2_op_params *) dst->op_params;  // 定义指向目标张量 op_params 成员的指针

    p->fun(dst, a, b, params->ith, params->nth, p->userdata);  // 调用自定义操作函数，传入目标张量指针、第一个张量指针、第二个张量指针、ith、nth、userdata
}

// ggml_compute_forward_map_custom3
// 计算自定义操作的前向映射
static void ggml_compute_forward_map_custom3(
        const struct ggml_compute_params * params,  // 参数结构体指针
        const struct ggml_tensor * a,  // 输入张量a
        const struct ggml_tensor * b,  // 输入张量b
        const struct ggml_tensor * c,  // 输入张量c
              struct ggml_tensor * dst) {  // 输出张量dst
    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  // 如果任务类型是初始化或者结束，则返回
        return;
    }

    struct ggml_map_custom3_op_params * p = (struct ggml_map_custom3_op_params *) dst->op_params;  // 获取输出张量的自定义操作参数

    p->fun(dst, a, b, c, params->ith, params->nth, p->userdata);  // 调用自定义操作函数
}

// ggml_compute_forward_cross_entropy_loss

static void ggml_compute_forward_cross_entropy_loss_f32(
        const struct ggml_compute_params * params,  // 参数结构体指针
        const struct ggml_tensor * src0,  // 输入张量src0
        const struct ggml_tensor * src1,  // 输入张量src1
        struct ggml_tensor * dst) {  // 输出张量dst
    GGML_ASSERT(ggml_is_contiguous(src0));  // 断言输入张量src0是连续的
    GGML_ASSERT(ggml_is_contiguous(src1));  // 断言输入张量src1是连续的
    GGML_ASSERT(ggml_is_scalar(dst));  // 断言输出张量dst是标量
    GGML_ASSERT(ggml_are_same_shape(src0, src1));  // 断言输入张量src0和src1具有相同的形状

    const int ith = params->ith;  // 获取当前线程的索引
    const int nth = params->nth;  // 获取线程总数

    float * sums = (float *) params->wdata;  // 获取参数结构体中的浮点数数组

    // TODO: 处理转置/置换矩阵
    const int nc = src0->ne[0];  // 获取输入张量src0的第一个维度大小
    const int nr = ggml_nrows(src0);  // 获取输入张量src0的行数

    GGML_ASSERT(params->wsize >= sizeof(float) * (nth + nth * nc));  // 断言参数结构体中的数组大小满足要求

    if (params->type == GGML_TASK_INIT) {  // 如果任务类型是初始化
        if (ith == 0) {  // 如果当前线程索引为0
            memset(sums, 0, sizeof(float) * (nth + nth * nc));  // 将数组sums清零
        }
        return;
    }

    if (params->type == GGML_TASK_FINALIZE) {  // 如果任务类型是结束
        if (ith == 0) {  // 如果当前线程索引为0
            float * dp = (float *) dst->data;  // 获取输出张量dst的数据指针
            ggml_vec_sum_f32(nth, dp, sums);  // 对输出张量dst的数据进行求和
            dp[0] *= -1.0f / (float) nr;  // 对求和结果进行乘法和除法操作
        }
        return;
    }

    const double eps = 1e-9;  // 定义一个小的浮点数值

    // 每个线程的行数
    const int dr = (nr + nth - 1)/nth;  // 计算每个线程处理的行数

    // 当前线程的行范围
    const int ir0 = dr*ith;  // 计算当前线程处理的起始行索引
    const int ir1 = MIN(ir0 + dr, nr);  // 计算当前线程处理的结束行索引，取最小值
    # 遍历 ir0 到 ir1 之间的整数，i1 作为循环变量
    for (int i1 = ir0; i1 < ir1; i1++) {
        # 计算 src0 中第 i1 行数据的起始地址，并将其转换为 float 指针
        float * s0 = (float *)((char *) src0->data + i1*src0->nb[1]);
        # 计算 src1 中第 i1 行数据的起始地址，并将其转换为 float 指针
        float * s1 = (float *)((char *) src1->data + i1*src1->nb[1]);
        # 计算 params->wdata 中第 nth + ith*nc 个元素的地址，并将其转换为 float 指针
        float * st = ((float *) params->wdata) + nth + ith*nc;
#ifndef NDEBUG
        // 如果处于调试模式，进行以下操作
        for (int i = 0; i < nc; ++i) {
            // 检查并打印数组中的值
            //printf("p[%d] = %f\n", i, p[i]);
            // 断言数组中的值不是 NaN
            assert(!isnan(s0[i]));
            // 断言数组中的值不是 NaN
            assert(!isnan(s1[i]));
        }
#endif
        // soft_max
        // 计算 soft_max
        ggml_float sum = 0.0;
        {
            // 初始化最大值为负无穷
            float max = -INFINITY;
            // 获取数组 s0 中的最大值
            ggml_vec_max_f32(nc, &max, s0);

            uint16_t scvt; UNUSED(scvt);
            // 遍历数组
            for (int i = 0; i < nc; i++) {
                // 如果数组中的值为负无穷
                if (s0[i] == -INFINITY) {
                    // 将 st 数组中的值设为 0.0
                    st[i] = 0.0f;
                } else {
#ifndef GGML_CROSS_ENTROPY_EXP_FP16
                    // 计算 s0[i] 减去最大值的结果
                    const float s = s0[i] - max;
                    // 计算 e 的 s 次方
                    const float val = expf(s);
#else
                    // 将 s0[i] 减去最大值的结果转换为 fp16 类型
                    ggml_fp16_t s = GGML_FP32_TO_FP16(s0[i] - max);
                    // 将 fp16 类型的值转换为 uint16_t 类型
                    memcpy(&scvt, &s, sizeof(scvt));
                    // 从 ggml_table_exp_f16 中获取对应值并转换为 fp32 类型
                    const float val = GGML_FP16_TO_FP32(ggml_table_exp_f16[scvt]);
#endif
                    // 将 val 加到 sum 中
                    sum += (ggml_float)val;
                    // 将 val 赋值给 st[i]
                    st[i] = val;
                }
            }

            // 断言 sum 大于 0.0
            assert(sum > 0.0);
            // sum = 1.0/sum;
        }
        // 避免由于 rescaling 导致的 log(0)，将 sum 重新缩放到 [eps..1] 区间
        sum = (1.0 - eps) / sum;
        // 将数组 st 中的值缩放为 sum
        ggml_vec_scale_f32(nc, st, sum);
        // 将数组 st 中的值加上 eps
        ggml_vec_add1_f32(nc, st, st, eps);
        // 对数组 st 中的值取对数
        ggml_vec_log_f32(nc, st, st);
        // 将数组 st 中的值与数组 s1 中的值相乘
        ggml_vec_mul_f32(nc, st, st, s1);

        // 计算数组 st 中的值的总和
        float st_sum = 0;
        ggml_vec_sum_f32(nc, &st_sum, st);
        // 将 st_sum 加到 sums[ith] 中
        sums[ith] += st_sum;

#ifndef NDEBUG
        // 如果处于调试模式，进行以下操作
        for (int i = 0; i < nc; ++i) {
            // 断言数组中的值不是 NaN
            assert(!isnan(st[i]));
            // 断言数组中的值不是无穷大
            assert(!isinf(st[i]));
        }
#endif
    }
}

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
                # 调用 ggml_compute_forward_cross_entropy_loss_f32 函数进行计算
                ggml_compute_forward_cross_entropy_loss_f32(params, src0, src1, dst);
            } break;
        # 如果 src0 的类型不是 GGML_TYPE_F32
        default:
            {
                # 断言，如果不是 GGML_TYPE_F32 类型则抛出错误
                GGML_ASSERT(false);
            } break;
    }
// 定义静态函数 ggml_compute_forward_cross_entropy_loss_back_f32，计算交叉熵损失的反向传播
static void ggml_compute_forward_cross_entropy_loss_back_f32(
        const struct ggml_compute_params * params,  // 参数结构体指针
        const struct ggml_tensor * src0,  // 输入张量指针
        const struct ggml_tensor * src1,  // 输入张量指针
        const struct ggml_tensor * opt0,  // 输入张量指针
        struct ggml_tensor * dst) {  // 输出张量指针
    GGML_ASSERT(ggml_is_contiguous(dst));  // 断言输出张量是连续的
    GGML_ASSERT(ggml_is_contiguous(src0));  // 断言输入张量是连续的
    GGML_ASSERT(ggml_is_contiguous(src1));  // 断言输入张量是连续的
    GGML_ASSERT(ggml_is_contiguous(opt0));  // 断言输入张量是连续的
    GGML_ASSERT(ggml_are_same_shape(src0, src1) && ggml_are_same_shape(src0, dst));  // 断言输入张量和输出张量具有相同的形状

    const int64_t ith = params->ith;  // 获取参数结构体中的 ith 值
    const int64_t nth = params->nth;  // 获取参数结构体中的 nth 值

    if (params->type == GGML_TASK_INIT || params->type == GGML_TASK_FINALIZE) {  // 如果参数结构体中的 type 值为 GGML_TASK_INIT 或 GGML_TASK_FINALIZE
        return;  // 直接返回
    }

    const double eps = 1e-9;  // 定义 eps 值为 1e-9

    // TODO: 处理转置/置换矩阵
    const int64_t nc = src0->ne[0];  // 获取输入张量的第一个维度大小
    const int64_t nr = ggml_nrows(src0);  // 获取输入张量的行数

    // 每个线程的行数
    const int64_t dr = (nr + nth - 1)/nth;  // 计算每个线程处理的行数

    // 该线程的行范围
    const int64_t ir0 = dr*ith;  // 计算该线程处理的起始行
    const int64_t ir1 = MIN(ir0 + dr, nr);  // 计算该线程处理的结束行，取 ir0 + dr 和 nr 中的较小值

    float * d   = (float *) opt0->data;  // 获取 opt0 张量的数据指针

    for (int64_t i1 = ir0; i1 < ir1; i1++) {  // 遍历该线程处理的行范围
        float * ds0 = (float *)((char *) dst->data  + i1*dst->nb[1]);  // 计算输出张量中当前行的起始地址
        float * s0  = (float *)((char *) src0->data + i1*src0->nb[1]);  // 计算输入张量 src0 中当前行的起始地址
        float * s1  = (float *)((char *) src1->data + i1*src1->nb[1]);  // 计算输入张量 src1 中当前行的起始地址

#ifndef NDEBUG
        for (int i = 0; i < nc; ++i) {  // 遍历当前行的每个元素
            //printf("p[%d] = %f\n", i, p[i]);
            assert(!isnan(s0[i]));  // 断言输入张量 src0 中当前元素不是 NaN
            assert(!isnan(s1[i]));  // 断言输入张量 src1 中当前元素不是 NaN
        }
#endif

        // soft_max
        ggml_float sum = 0.0;  // 定义 sum 值为 0.0
        {
            float max = -INFINITY;  // 定义 max 值为负无穷
            ggml_vec_max_f32(nc, &max, s0);  // 计算输入张量 src0 中每行的最大值

            uint16_t scvt; UNUSED(scvt);  // 定义未使用的 uint16_t 类型变量 scvt
            for (int i = 0; i < nc; i++) {  // 遍历当前行的每个元素
                if (s0[i] == -INFINITY) {  // 如果当前元素为负无穷
                    ds0[i] = 0.0f;  // 输出张量中对应位置的值设为 0.0
                } else {
#ifndef GGML_CROSS_ENTROPY_EXP_FP16
                    // 如果不是使用 FP16 数据类型，则执行以下代码
                    const float s = s0[i] - max;
                    // 计算 e^(s) 的值
                    const float val = expf(s);
#else
                    // 如果使用 FP16 数据类型，则执行以下代码
                    ggml_fp16_t s = GGML_FP32_TO_FP16(s0[i] - max);
                    // 将 FP16 数据类型转换为字节流
                    memcpy(&scvt, &s, sizeof(scvt));
                    // 从预先计算好的表中获取 e^(s) 的值
                    const float val = GGML_FP16_TO_FP32(ggml_table_exp_f16[scvt]);
#endif
                    // 累加 e^(s) 的值
                    sum += (ggml_float)val;
                    // 将 e^(s) 的值赋给 ds0[i]
                    ds0[i] = val;
                }
            }

            // 断言 sum 的值大于 0.0
            assert(sum > 0.0);
            // 计算 (1.0 - eps)/sum 的值，并赋给 sum
            sum = (1.0 - eps)/sum;
        }

        // 计算 grad(src0) = (softmax(src0) - src1) * grad(cross_entropy_loss(src0, src1)) / nr
        ggml_vec_scale_f32(nc, ds0, sum);
        ggml_vec_add1_f32(nc, ds0, ds0, eps);
        ggml_vec_sub_f32(nc, ds0, ds0, s1);
        ggml_vec_scale_f32(nc, ds0, d[0] / (float) nr);

#ifndef NDEBUG
        // 如果处于调试模式，则执行以下代码
        for (int i = 0; i < nc; ++i) {
            // 断言 ds0[i] 不是 NaN
            assert(!isnan(ds0[i]));
            // 断言 ds0[i] 不是无穷大
            assert(!isinf(ds0[i]));
        }
#endif
    }
}

// 根据不同的数据类型，调用不同的计算函数
static void ggml_compute_forward_cross_entropy_loss_back(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
        const struct ggml_tensor * opt0,
        struct ggml_tensor * dst) {
    switch (src0->type) {
        case GGML_TYPE_F32:
            {
                ggml_compute_forward_cross_entropy_loss_back_f32(params, src0, src1, opt0, dst);
            } break;
        default:
            {
                // 如果数据类型不是 F32，则断言失败
                GGML_ASSERT(false);
            } break;
    }
}

// 计算稀疏矩阵相乘的前向传播
static void ggml_compute_forward_mul_mat_sparse_head(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
              struct ggml_tensor * dst) {
    // 获取当前时间
    int64_t t0 = ggml_perf_time_us();
    UNUSED(t0);

    GGML_TENSOR_BINARY_OP_LOCALS;

    // 获取参数中的 ith 和 nth
    const int ith = params->ith;
    const int nth = params->nth;

    // 获取 src0 的数据类型
    const enum ggml_type type = src0->type;
    // 检查 src1 是否是连续的
    const bool src1_cont = ggml_is_contiguous(src1);

    // 获取类型特性中的向量点积函数和向量点积类型
    ggml_vec_dot_t    const vec_dot               = type_traits[type].vec_dot;
    enum ggml_type    const vec_dot_type          = type_traits[type].vec_dot_type;
    ggml_from_float_t const from_float_to_vec_dot = type_traits[vec_dot_type].from_float;

    // 断言 ne0 等于 ne01，ne1 等于 ne11，ne2 等于 ne12，ne3 等于 ne13
    GGML_ASSERT(ne0 == ne01);
    GGML_ASSERT(ne1 == ne11);
    GGML_ASSERT(ne2 == ne12);
    GGML_ASSERT(ne3 == ne13);

    // 断言不支持 src0 或 src1 的置换
    GGML_ASSERT(nb00 == ggml_type_size(type));
    GGML_ASSERT(nb10 == sizeof(float));

    // 断言 dst 不能被转置或置换
    GGML_ASSERT(nb0 == sizeof(float));
    GGML_ASSERT(nb0 <= nb1);
    GGML_ASSERT(nb1 <= nb2);
    GGML_ASSERT(nb2 <= nb3);

    // 计算广播因子
    const int64_t r2 = ne12/ne02;
    const int64_t r3 = ne13/ne03;

    // nb01 >= nb00 - src0 未被转置，按 src0 行计算

    // 如果任务类型为 GGML_TASK_INIT
    if (params->type == GGML_TASK_INIT) {
        // 如果 src1 的类型不是 vec_dot_type
        if (src1->type != vec_dot_type) {
            // 初始化 wdata 和 row_size
            char * wdata = params->wdata;
            const size_t row_size = ne10*ggml_type_size(vec_dot_type)/ggml_blck_size(vec_dot_type);

            // 遍历计算 from_float_to_vec_dot
            for (int64_t i13 = 0; i13 < ne13; ++i13) {
                for (int64_t i12 = 0; i12 < ne12; ++i12) {
                    for (int64_t i11 = 0; i11 < ne11; ++i11) {
                        from_float_to_vec_dot((float *)((char *) src1->data + i13*nb13 + i12*nb12 + i11*nb11), (void *) wdata, ne10);
                        wdata += row_size;
                    }
                }
            }
        }
        // 将 dst 设置为零，并将 params->aic 原子存储为 0
        ggml_set_zero(dst);
        atomic_store(params->aic, 0);

        return;
    }

    // 如果任务类型为 GGML_TASK_FINALIZE
    if (params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 初始化 wdata 和 row_size
    const void * wdata    = (src1->type == vec_dot_type) ? src1->data : params->wdata;
    const size_t row_size = ne10*ggml_type_size(vec_dot_type)/ggml_blck_size(vec_dot_type);

    // 初始化 nr0 和 nr1
    const int64_t nr0 = ne01;           // src0 行数
    const int64_t nr1 = ne11*ne12*ne13; // src1 行数
    //printf("nr0 = %lld, nr1 = %lld\n", nr0, nr1);

    // 根据哪个循环更大来分配线程工作，内循环或外循环

    const int64_t nth0 = nr0 > nr1 ? nth : 1; // 通过 src0 行并行化
    const int64_t nth1 = nr0 > nr1 ? 1 : nth; // 通过 src1 行并行化

    const int64_t ith0 = ith % nth0;
    const int64_t ith1 = ith / nth0;

    const int64_t dr0 = (nr0 + 8*nth0 - 1)/(8*nth0);
    const int64_t dr1 = (nr1 + nth1 - 1)/nth1;

    int64_t ir010 = dr0*ith0;
    // const int64_t ir011 = MIN(ir010 + dr0, nr0);
    // const int64_t ir011 = ir010 + dr0;

    const int64_t ir110 = dr1*ith1;
    const int64_t ir111 = MIN(ir110 + dr1, nr1);

    //printf("ir010 = %6lld, ir011 = %6lld, ir110 = %6lld, ir111 = %6lld\n", ir010, ir011, ir110, ir111);

    // 没有工作的线程直接让出 CPU（不确定是否有帮助）
    // if (ir010 >= ir011 || ir110 >= ir111) {
    //     sched_yield();
    //     return;
    // }

    assert(ne12 % ne02 == 0);
    assert(ne13 % ne03 == 0);

    // 尝试块状分割
    // const int64_t blck_0 = 16;
    const int64_t blck_1 = 16;

    // 尝试减少伪共享（似乎没有什么区别）
    // float tmp[16];
    float *ffdata = (float *)dst->src[2]->data;
    // int *gid = (int *)dst->src[3]->data;
    } 
    # 计算稀疏矩阵的乘法
    static void ggml_compute_forward_mul_mat_sparse(
        # 获取当前时间
        const struct ggml_compute_params * params,
        # 源张量 src0
        const struct ggml_tensor * src0,
        # 源张量 src1
        const struct ggml_tensor * src1,
        # 目标张量 dst
              struct ggml_tensor * dst) {
    # 获取当前时间并标记为未使用
    int64_t t0 = ggml_perf_time_us();
    UNUSED(t0);

    # 定义一些局部变量
    GGML_TENSOR_BINARY_OP_LOCALS;

    # 获取参数中的 ith 和 nth
    const int ith = params->ith;
    const int nth = params->nth;

    # 获取 src0 的数据类型
    const enum ggml_type type = src0->type;

    # 检查 src1 是否是连续的
    const bool src1_cont = ggml_is_contiguous(src1);

    # 获取类型相关的函数指针
    ggml_vec_dot_t    const vec_dot               = type_traits[type].vec_dot;
    enum ggml_type    const vec_dot_type          = type_traits[type].vec_dot_type;
    ggml_from_float_t const from_float_to_vec_dot = type_traits[vec_dot_type].from_float;

    # 获取稀疏矩阵的阈值
    const float threshold = sparse_pred_threshold;

    # 断言一些条件
    GGML_ASSERT(ne0 == ne01);
    GGML_ASSERT(ne1 == ne11);
    GGML_ASSERT(ne2 == ne12);
    GGML_ASSERT(ne3 == ne13);

    # 断言不支持 src0 或 src1 的置换
    GGML_ASSERT(nb00 == ggml_type_size(type));
    GGML_ASSERT(nb10 == sizeof(float));

    # 断言 dst 不能被转置或置换
    GGML_ASSERT(nb0 == sizeof(float));
    GGML_ASSERT(nb0 <= nb1);
    GGML_ASSERT(nb1 <= nb2);
    GGML_ASSERT(nb2 <= nb3);

    # 计算广播因子
    const int64_t r2 = ne12/ne02;
    const int64_t r3 = ne13/ne03;

    # nb01 >= nb00 - src0 不是转置的
    #   通过 src0 的行进行计算

    # 如果支持使用 CLBlast 库
    if (ggml_cl_can_mul_mat(src0, src1, dst)) {
        # TODO: 处理当 src0 可以在第二、第三维度上广播到 src1 的情况
        #       参考: https://github.com/ggerganov/ggml/pull/224
        GGML_ASSERT(ne02 == ne12);
        GGML_ASSERT(ne03 == ne13);

        # 如果是计算任务的第一个线程，则调用 CLBlast 库进行矩阵乘法计算
        if (params->ith == 0 && params->type == GGML_TASK_COMPUTE) {
            ggml_cl_mul_mat(src0, src1, dst, params->wdata, params->wsize);
        }
        return;
    }
#endif

# 如果使用了 Accelerate 或 OpenBLAS 库
    # 如果使用 BLAS 计算前向乘法矩阵成功，则执行以下操作
    if (ggml_compute_forward_mul_mat_use_blas(src0, src1, dst)):

        # 如果参数中的 ith 不等于 0，则直接返回
        if (params->ith != 0):
            return;

        # 如果参数中的 type 为 GGML_TASK_INIT，则直接返回
        if (params->type == GGML_TASK_INIT):
            return;

        # 如果参数中的 type 为 GGML_TASK_FINALIZE，则直接返回
        if (params->type == GGML_TASK_FINALIZE):
            return;

        # 遍历循环，i13 从 0 到 ne13，i12 从 0 到 ne12
        for (int64_t i13 = 0; i13 < ne13; i13++):
            for (int64_t i12 = 0; i12 < ne12; i12++):
                # 将 src0 在第二维和第三维度上广播到 src1
                const int64_t i03 = i13/r3;
                const int64_t i02 = i12/r2;

                const void  * x = (char *)            src0->data + i02*nb02 + i03*nb03;
                const float * y = (float *) ((char *) src1->data + i12*nb12 + i13*nb13);

                float * d = (float *) ((char *) dst->data + i12*nb2 + i13*nb3);

                # 如果类型不是 GGML_TYPE_F32，则执行以下操作
                if (type != GGML_TYPE_F32):
                    float * const wdata    = params->wdata;
                    ggml_to_float_t const to_float = type_traits[type].to_float;

                    size_t id = 0;
                    # 遍历循环，i01 从 0 到 ne01
                    for (int64_t i01 = 0; i01 < ne01; ++i01):
                        to_float((const char *) x + i01*nb01, wdata + id, ne00);
                        id += ne00;

                    # 断言确保 id*sizeof(float) 小于等于 params->wsize
                    assert(id*sizeof(float) <= params->wsize);
                    x = wdata;

                # 使用 BLAS 库中的 cblas_sgemm 函数进行矩阵乘法运算
                cblas_sgemm(CblasRowMajor, CblasNoTrans, CblasTrans,
                        ne11, ne01, ne10,
                        1.0f,    y, ne10,
                                 x, ne00,
                        0.0f,    d, ne01);

        # 打印 CBLAS 运行时间和矩阵维度信息
        #printf("CBLAS = %f ms, %d x %d x %d x %d\n", (ggml_perf_time_us() - t0)/1000.0, ne0, ne1, ne2, ne3);

        # 返回
        return;
#endif

    // 如果任务类型是初始化
    if (params->type == GGML_TASK_INIT) {
        // 如果 src1 的类型不是 vec_dot_type
        if (src1->type != vec_dot_type) {
            // 获取参数中的 wdata
            char * wdata = params->wdata;
            // 计算每行数据的大小
            const size_t row_size = ne10*ggml_type_size(vec_dot_type)/ggml_blck_size(vec_dot_type);

            // 循环遍历三维数组
            for (int64_t i13 = 0; i13 < ne13; ++i13) {
                for (int64_t i12 = 0; i12 < ne12; ++i12) {
                    for (int64_t i11 = 0; i11 < ne11; ++i11) {
                        // 将 float 类型转换为 vec_dot 类型
                        from_float_to_vec_dot((float *)((char *) src1->data + i13*nb13 + i12*nb12 + i11*nb11), (void *) wdata, ne10);
                        wdata += row_size;
                    }
                }
            }
        }
        // 将 params->aic 设置为 0
        atomic_store(params->aic, 0);

        // 返回
        return;
    }

    // 如果任务类型是结束
    if (params->type == GGML_TASK_FINALIZE) {
        // 返回
        return;
    }

    // 获取 wdata
    const void * wdata    = (src1->type == vec_dot_type) ? src1->data : params->wdata;
    // 计算每行数据的大小
    const size_t row_size = ne10*ggml_type_size(vec_dot_type)/ggml_blck_size(vec_dot_type);

    // 计算 src0 和 src1 的行数
    const int64_t nr0 = ne01;           // src0 rows
    const int64_t nr1 = ne11*ne12*ne13; // src1 rows

    // 根据行数的大小分配线程工作
    const int64_t nth0 = nr0 > nr1 ? nth : 1; // parallelize by src0 rows
    const int64_t nth1 = nr0 > nr1 ? 1 : nth; // parallelize by src1 rows

    const int64_t ith0 = ith % nth0;
    const int64_t ith1 = ith / nth0;

    // 计算每个线程的行数
    const int64_t dr0 = (nr0 + 8*nth0 - 1)/(8*nth0);
    const int64_t dr1 = (nr1 + nth1 - 1)/nth1;

    // 计算每个线程的起始和结束行数
    int64_t ir010 = dr0*ith0;
    int64_t ir011 = MIN(ir010 + dr0, nr0);

    const int64_t ir110 = dr1*ith1;
    const int64_t ir111 = MIN(ir110 + dr1, nr1);

    // 线程没有工作时简单地让出 (不确定是否有帮助)
    // 确保 ne12 能被 ne02 整除
    assert(ne12 % ne02 == 0);
    // 确保 ne13 能被 ne03 整除
    assert(ne13 % ne03 == 0);

    // 尝试使用块状分割
    // const int64_t blck_0 = 16;
    // 定义块大小为 16
    const int64_t blck_1 = 16;
    // int total = 0;

    // 尝试减少伪共享（似乎没有影响）
    // float tmp[16];
    // 将 dst->src[2]->data 转换为 float 指针
    float *ffdata = (float *)dst->src[2]->data;
    // 将 dst->src[3]->data 转换为 int 指针
    int *gid = (int *)dst->src[3]->data;
    // 将 dst->src[2]->data 转换为 float 指针
    float *predictor_data = (float *)dst->src[2]->data;
    // 计算预测器数据行大小
    const size_t predictor_row_size = dst->src[2]->ne[0]*ggml_type_size(GGML_TYPE_F32)/ggml_blck_size(GGML_TYPE_F32);

    }
    // printf("total %d\n", total);

    // int predictor_cpu = 0;
    // int predictor = 0;
    // for (int i = 0; i < 9216 *4 ; i++) {
    //     if (ffdata[i] > 0.5f && gid[i] == 0)
    //         predictor_cpu += 1;
    //     if (ffdata[i] > 0.5f)
    //         predictor += 1;
    // }
    // if (ith == 0)
    //     printf("predictor %d predictor_cpu %d\n", predictor, predictor_cpu);
// 定义函数 ggml_axpy_normal_f16，实现向量运算 vz = alpha * vx + vy
static void ggml_axpy_normal_f16(const int n, const ggml_fp16_t * vx, const ggml_fp16_t * restrict vy, void* restrict vz, ggml_fp16_t alpha) {
    float *res = (float *)vz;  // 将结果指针强制类型转换为 float 指针
    for (int i = 0; i < n; i++) {  // 遍历向量
            res[i] = res[i] + (GGML_FP16_TO_FP32(vx[i])*GGML_FP16_TO_FP32(alpha));  // 执行向量运算
    }
    (void) vy;  // 防止未使用的参数警告
}

// 定义函数 ggml_axpy_avx_f16，实现 AVX 指令集下的向量运算
static void ggml_axpy_avx_f16(const int n, const ggml_fp16_t * restrict vx, const ggml_fp16_t * vy, void* vz, ggml_fp16_t alpha) {
#if defined(__AVX2__)  // 判断是否支持 AVX2 指令集
    float *result = (float *)vz;  // 将结果指针强制类型转换为 float 指针
    float alpha_f32 = GGML_FP16_TO_FP32(alpha);  // 将 alpha 转换为 float 类型
    __m256 scale = _mm256_set1_ps(alpha_f32);  // 创建一个包含 alpha_f32 的 AVX 向量
    for (int i = 0; i < n; i += 8) {  // 每次处理 8 个元素
        __m128i vx_low = _mm_loadu_si128((__m128i const*)(&vx[i]));  // 加载 vx 的低位 128 位数据
        __m256 vx_f32 = _mm256_cvtph_ps(vx_low);  // 将 vx 转换为 float 类型的 AVX 向量
        __m256 vy_f32 = _mm256_loadu_ps((float const*)(result+ i));  // 加载 vy 的数据
        __m256 res = _mm256_fmadd_ps(vx_f32, scale, vy_f32);  // 执行向量加法和乘法操作
        _mm256_storeu_ps((float*)(&result[i]), res);  // 存储结果
    }
#else
    float *res = (float *)vz;  // 将结果指针强制类型转换为 float 指针
    float alpha_convert = GGML_FP16_TO_FP32(alpha);  // 将 alpha 转换为 float 类型
    for (int i = 0; i < n; i++) {  // 遍历向量
        res[i] = res[i] + (GGML_FP16_TO_FP32(vx[i])*alpha_convert);  // 执行向量运算
    }
#endif
    (void)vy;  // 防止未使用的参数警告
}

// 定义原子锁
atomic_flag g_axpy_dense_lock = ATOMIC_FLAG_INIT;

// 定义函数 ggml_compute_forward_mul_mat_axpy_dense，实现稠密矩阵乘法和向量加法
static void ggml_compute_forward_mul_mat_axpy_dense(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
              struct ggml_tensor * dst) {
    int64_t t0 = ggml_perf_time_us();  // 获取当前时间
    UNUSED(t0);  // 防止未使用的变量警告

    GGML_TENSOR_BINARY_OP_LOCALS;  // 定义局部变量

    // const int ith = params->ith;  // 获取参数中的 ith 值
    const int nth = params->nth;  // 获取参数中的 nth 值

    const enum ggml_type type = src0->type;  // 获取 src0 的数据类型

    // const bool src1_cont = ggml_is_contiguous(src1);  // 判断 src1 是否是连续的

    // ggml_vec_dot_t    const vec_dot               = type_traits[type].vec_dot;  // 获取数据类型对应的向量点乘函数
    enum ggml_type    const vec_dot_type          = type_traits[type].vec_dot_type;  // 获取数据类型对应的向量点乘函数的数据类型
}
    // 从类型特征数组中获取从浮点数到向量点乘类型的转换函数
    ggml_from_float_t const from_float_to_vec_dot = type_traits[vec_dot_type].from_float;

    // 断言ne0等于ne01，ne1等于ne11，ne2等于ne12，ne3等于ne13
    // GGML_ASSERT(ne0 == ne01);
    // GGML_ASSERT(ne1 == ne11);
    // GGML_ASSERT(ne2 == ne12);
    // GGML_ASSERT(ne3 == ne13);

    // 不支持src0或src1的置换
    GGML_ASSERT(nb00 == ggml_type_size(type));
    GGML_ASSERT(nb10 == sizeof(float));

    // 目标不能被转置或置换
    GGML_ASSERT(nb0 == sizeof(float));
    GGML_ASSERT(nb0 <= nb1);
    GGML_ASSERT(nb1 <= nb2);
    GGML_ASSERT(nb2 <= nb3);

    // 广播因子
    // const int64_t r2 = ne12/ne02;
    // const int64_t r3 = ne13/ne03;

    // nb01 >= nb00 - src0未被转置
    //   通过src0行计算

    if (params->type == GGML_TASK_INIT) {
        // 将dst置零
        ggml_set_zero(dst);
        // 如果src1的类型不是向量点乘类型
        if (src1->type != vec_dot_type) {
            char * wdata = params->wdata;
            const size_t row_size = ne10*ggml_type_size(vec_dot_type)/ggml_blck_size(vec_dot_type);

            for (int64_t i13 = 0; i13 < ne13; ++i13) {
                for (int64_t i12 = 0; i12 < ne12; ++i12) {
                    for (int64_t i11 = 0; i11 < ne11; ++i11) {
                        from_float_to_vec_dot((float *)((char *) src1->data + i13*nb13 + i12*nb12 + i11*nb11), (void *) wdata, ne10);
                        wdata += row_size;
                    }
                }
            }
        }
        // 原子地将params的aic设置为0
        atomic_store(params->aic, 0);

        return;
    }

    if (params->type == GGML_TASK_FINALIZE) {
        return;
    }

    // 如果src1的类型是向量点乘类型，则wdata等于src1的数据，否则等于params的wdata
    ggml_fp16_t* wdata    = (src1->type == vec_dot_type) ? src1->data : params->wdata;
    // const size_t row_size = ne10*ggml_type_size(vec_dot_type)/ggml_blck_size(vec_dot_type);

    // 获取dst的第二个源张量
    struct ggml_tensor *src2 = dst->src[2];
    
    // 通过src0行并行化
    const int64_t dr = (src2->ne[0] + 8*nth - 1)/(8*nth);
    const int nr = ggml_nrows(src0);

    // const int64_t ir10 = dr*ith;
    // const int64_t ir11 = MIN(ir10 + dr, src2->ne[0]);

    // src1行
    // 计算向量的长度
    const int64_t nr1 = ne11*ne12*ne13;
    // 获取源数据的指针
    float *idx = src2->data;
    // 获取目标数据的指针
    int *gid = (int *)(dst->src[3]->data);
    // 打印信息
    printf("down %d up %d ne00 %d\n", ir10, ir11, ne00);

    // 创建一个长度为 ne00*4 的浮点数数组
    float vec[ne00*4];
    // 获取数组的指针
    void *vy = vec;
    // 将数组的内容全部初始化为 0
    memset(vy, 0, ne00*4);
    // 获取源数据的行指针
    char* src0_row = (char *) src0->data;
    // 进入循环
    while(true) {
        // 获取原子计数的值
        const int ir0 = atomic_fetch_add(params->aic, dr);
        // 遍历数据
        for (int64_t ir1 = ir0; ir1 < ir0+dr; ir1++) {
            // 如果超出范围则跳出循环
            if (ir1 >= nr) break;
            // 调用函数进行计算
            ggml_axpy_avx_f16(ne00, (ggml_fp16_t *)(src0_row+nb01*ir1), (ggml_fp16_t *)vy, vy, wdata[ir1]);
        }
        // 如果超出范围则跳出循环
        if (ir0 + dr >= nr)
            break;
    }
    
    // 获取锁
    while (atomic_flag_test_and_set(&g_axpy_dense_lock)) {
        // 如果锁已经被占用，则等待
    }
    
    // 获取目标数据的浮点数指针
    float *res = (float *)(dst->data);
    // 获取临时数据的浮点数指针
    float *tmp = (float *)vy;
    // 初始化变量 i
    int i;

    // 计算剩余的元素个数
    int remainder = ne00 % 8;
#if defined(__AVX2__)
    // 如果定义了 AVX2 指令集，则使用 AVX 指令进行向量化计算
    for (i = 0; i < ne00 - remainder; i += 8) {
        // 加载 res 中的 8 个浮点数到 AVX 寄存器
        __m256 res_vec = _mm256_loadu_ps(res + i);
        // 加载 tmp 中的 8 个浮点数到 AVX 寄存器
        __m256 tmp_vec = _mm256_loadu_ps(tmp + i);
        // 执行 AVX 指令进行 8 个浮点数的加法运算
        __m256 result = _mm256_add_ps(res_vec, tmp_vec);
        // 将结果存储到 res 中
        _mm256_storeu_ps(res + i, result);
    }

    // 处理剩余的元素
    for (i = ne00 - remainder; i < ne00; i++) {
        res[i] += tmp[i];
    }
#else
    // 如果未定义 AVX2 指令集，则使用普通的循环进行计算
    for (i = 0; i < dst->ne[0]; i++) {
        res[i] += tmp[i];
    }
#endif
    // 清除原子标志，表示计算完成
    atomic_flag_clear(&g_axpy_dense_lock);
}

// 初始化原子标志和原子整型变量
atomic_flag g_axpy_lock = ATOMIC_FLAG_INIT;
atomic_int g_axpy_control = 0;
static void ggml_compute_forward_mul_mat_axpy(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
              struct ggml_tensor * dst) {
    int64_t t0 = ggml_perf_time_us();
    UNUSED(t0);

    GGML_TENSOR_BINARY_OP_LOCALS;

    const int ith = params->ith;
    const int nth = params->nth;

    const enum ggml_type type = src0->type;

    // const bool src1_cont = ggml_is_contiguous(src1);

    // 获取向量点积函数和类型转换函数
    // ggml_vec_dot_t    const vec_dot               = type_traits[type].vec_dot;
    enum ggml_type    const vec_dot_type          = type_traits[type].vec_dot_type;
    ggml_from_float_t const from_float_to_vec_dot = type_traits[vec_dot_type].from_float;

    const float threshold = sparse_pred_threshold;

    // 断言确保张量的维度匹配
    // GGML_ASSERT(ne0 == ne01);
    // GGML_ASSERT(ne1 == ne11);
    // GGML_ASSERT(ne2 == ne12);
    // GGML_ASSERT(ne3 == ne13);

    // 断言确保不支持 src0 或 src1 的置换
    GGML_ASSERT(nb00 == ggml_type_size(type));
    GGML_ASSERT(nb10 == sizeof(float));

    // 断言确保 dst 不能被转置或置换
    GGML_ASSERT(nb0 == sizeof(float));
    GGML_ASSERT(nb0 <= nb1);
    GGML_ASSERT(nb1 <= nb2);
    GGML_ASSERT(nb2 <= nb3);

    // 广播因子
    // const int64_t r2 = ne12/ne02;
    // const int64_t r3 = ne13/ne03;
}
    // 如果任务类型是初始化
    if (params->type == GGML_TASK_INIT) {
        // 将目标数据清零
        ggml_set_zero(dst);
        // 如果src1的类型不是vec_dot_type
        if (src1->type != vec_dot_type) {
            // 获取参数中的wdata
            char * wdata = params->wdata;
            // 计算每行的大小
            const size_t row_size = ne10*ggml_type_size(vec_dot_type)/ggml_blck_size(vec_dot_type);

            // 遍历ne13、ne12、ne11
            for (int64_t i13 = 0; i13 < ne13; ++i13) {
                for (int64_t i12 = 0; i12 < ne12; ++i12) {
                    for (int64_t i11 = 0; i11 < ne11; ++i11) {
                        // 将src1中的数据转换成vec_dot类型，并存储到wdata中
                        from_float_to_vec_dot((float *)((char *) src1->data + i13*nb13 + i12*nb12 + i11*nb11), (void *) wdata, ne10);
                        // 更新wdata的位置
                        wdata += row_size;
                    }
                }
            }
        }
        // 将params中的aic设置为0
        atomic_store(params->aic, 0);
        // 将g_axpy_control设置为0
        atomic_store(&g_axpy_control, 0);

        // 返回
        return;
    }

    // 如果任务类型是最终化
    if (params->type == GGML_TASK_FINALIZE) {
        // 返回
        return;
    }

    // 如果src1的类型是vec_dot_type，则将wdata设置为src1的数据，否则设置为params中的wdata
    ggml_fp16_t* wdata    = (src1->type == vec_dot_type) ? src1->data : params->wdata;
    // 计算每行的大小
    const size_t row_size = ne10*ggml_type_size(vec_dot_type)/ggml_blck_size(vec_dot_type);

    // 获取dst的第二个源张量
    struct ggml_tensor *src2 = dst->src[2];
    
    // 并行化，按照src0的行进行分割
    const int64_t dr = (ne01 + nth - 1)/(nth);
    const int nr = ggml_nrows(src0);

    const int64_t ir10 = dr*ith;

    // src1的行数
    const int64_t nr1 = ne11*ne12*ne13;
    // 获取src2的数据
    float *idx = src2->data;
    int idx_row_size = src2->nb[1];
    // 获取dst的第三个源张量的数据
    int *gid = (int *)(dst->src[3]->data);

    // 创建一个大小为ne00*4的向量
    float vec[ne00*4];
    void *vy = vec;
    // 获取src0的数据
    char* src0_row = (char *) src0->data;
    ggml_fp16_t * src1_ptr = NULL;
    # 遍历列索引，从0到nr1-1
    for (int col_idx = 0; col_idx < nr1; col_idx++) {
        # 计算src1_ptr的指针位置
        src1_ptr = (ggml_fp16_t *)((char *)wdata + col_idx * row_size);
        # 计算idx的指针位置
        idx = (float *)((char *)src2->data + col_idx * idx_row_size);
        # 将vy数组清零
        memset(vy, 0, ne00*4);
        # 可能编写一个特殊的axpy函数用于批处理1
        # while(true) {
            # 使用原子操作获取aic的值并增加dr
            # const int ir0 = atomic_fetch_add(params->aic, dr);
            # 遍历ir1，从ir10到ir10+dr-1
            for (int64_t ir1 = ir10; ir1 < ir10+dr; ir1++) {
                # 如果ir1超过了nr，则跳出循环
                if (ir1 >= nr) {
                    break;
                }
                # 如果src1_ptr[ir1]等于0，则跳过本次循环
                if (src1_ptr[ir1]==0)
                    continue;
                # 如果gid[ir1]等于1，则跳过本次循环
                if (gid[ir1] == 1) {
                    continue;
                }
                # 如果idx[ir1]小于threshold，则跳过本次循环
                if (idx[ir1] < threshold)
                    continue;
                # 调用ggml_axpy_avx_f16函数进行向量运算
                ggml_axpy_avx_f16(ne00, (ggml_fp16_t *)(src0_row+nb01*ir1), (ggml_fp16_t *)vy, vy, src1_ptr[ir1]);
            }

            # 获取锁，如果锁已经被占用，则等待
            while (atomic_flag_test_and_set(&g_axpy_lock))
            {
                # 如果锁已经被占用，则等待
            }
        
        # 计算res的指针位置
        float *res = (float *)((char *)(dst->data) + col_idx * nb1);
        # 计算tmp的指针位置
        float *tmp = (float *)vy;
        # 定义变量i
        int i;
    
        # 计算剩余的元素个数，取余数
        int remainder = ne00 % 8;
#if defined(__AVX2__)
        // 如果定义了 AVX2 指令集，则使用 AVX 指令进行向量化计算
        for (i = 0; i < ne00 - remainder; i += 8) {
            // 加载 res 中的8个浮点数到 AVX 寄存器
            __m256 res_vec = _mm256_loadu_ps(res + i);
            // 加载 tmp 中的8个浮点数到 AVX 寄存器
            __m256 tmp_vec = _mm256_loadu_ps(tmp + i);
            // 执行 AVX 指令集中的加法运算
            __m256 result = _mm256_add_ps(res_vec, tmp_vec);
            // 将结果存储到 res 中
            _mm256_storeu_ps(res + i, result);
        }

        // 处理剩余的元素
        for (i = ne00 - remainder; i < ne00; i++) {
            res[i] += tmp[i];
        }
#else
        // 如果未定义 AVX2 指令集，则使用普通的循环进行计算
        for (i = 0; i < ne00; i++) {
            res[i] += tmp[i];
        }
#endif
        // 清除原子标志位 g_axpy_lock
        atomic_flag_clear(&g_axpy_lock);
    }

}
static void ggml_compute_forward_mul_mat_axpy_q4_0(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
              struct ggml_tensor * dst) {
    int64_t t0 = ggml_perf_time_us();
    UNUSED(t0);

    GGML_TENSOR_BINARY_OP_LOCALS;

    const int ith = params->ith;
    const int nth = params->nth;

    const enum ggml_type type = src0->type;

    // const bool src1_cont = ggml_is_contiguous(src1);

    // ggml_vec_dot_t    const vec_dot               = type_traits[type].vec_dot;
    enum ggml_type    const vec_dot_type          = type_traits[type].vec_dot_type;
    ggml_from_float_t const from_float_to_vec_dot = type_traits[vec_dot_type].from_float;

    const float threshold = sparse_pred_threshold;

    // GGML_ASSERT(ne0 == ne01);
    // GGML_ASSERT(ne1 == ne11);
    // GGML_ASSERT(ne2 == ne12);
    // GGML_ASSERT(ne3 == ne13);

    // we don't support permuted src0 or src1
    GGML_ASSERT(nb00 == ggml_type_size(type));
    GGML_ASSERT(nb10 == sizeof(float));

    // dst cannot be transposed or permuted
    GGML_ASSERT(nb0 == sizeof(float));
    GGML_ASSERT(nb0 <= nb1);
    GGML_ASSERT(nb1 <= nb2);
    GGML_ASSERT(nb2 <= nb3);

    // broadcast factors
    // const int64_t r2 = ne12/ne02;
    // const int64_t r3 = ne13/ne03;
    // 如果任务类型是初始化
    if (params->type == GGML_TASK_INIT) {
        // 将目标数据清零
        ggml_set_zero(dst);
        // 如果src1的类型不是vec_dot_type
        if (src1->type != vec_dot_type) {
            // 获取参数中的权重数据
            char * wdata = params->wdata;
            // 计算每行的大小
            const size_t row_size = ne10*ggml_type_size(vec_dot_type)/ggml_blck_size(vec_dot_type);

            // 遍历src1的维度
            for (int64_t i13 = 0; i13 < ne13; ++i13) {
                for (int64_t i12 = 0; i12 < ne12; ++i12) {
                    for (int64_t i11 = 0; i11 < ne11; ++i11) {
                        // 将src1的数据转换成vec_dot类型
                        from_float_to_vec_dot((float *)((char *) src1->data + i13*nb13 + i12*nb12 + i11*nb11), (void *) wdata, ne10);
                        // 更新权重数据指针
                        wdata += row_size;
                    }
                }
            }
        }
        // 将原子计数器设置为0
        atomic_store(params->aic, 0);

        // 返回
        return;
    }

    // 如果任务类型是最终化
    if (params->type == GGML_TASK_FINALIZE) {
        // 返回
        return;
    }

    // 如果src1的类型是vec_dot_type，则wdata为src1的数据，否则为参数中的权重数据
    ggml_fp16_t* wdata    = (src1->type == vec_dot_type) ? src1->data : params->wdata;
    // 计算每行的大小
    const size_t row_size = ne10*ggml_type_size(vec_dot_type)/ggml_blck_size(vec_dot_type);

    // 获取dst的第二个源张量
    struct ggml_tensor *src2 = dst->src[2];
    
    // 并行化，按照src0的行数
    const int64_t dr = (src2->ne[0] + nth - 1)/(nth);
    const int nr = ggml_nrows(src0);

    const int64_t ir10 = dr*ith;

    // src1的行数
    const int64_t nr1 = ne11*ne12*ne13;
    float *idx = src2->data;
    int idx_row_size = src2->nb[1];
    int *gid = (int *)(dst->src[3]->data);
    
    // 创建一个大小为ne00*4的向量
    float vec[ne00*4];
    void *vy = vec;
    // 将src0的数据转换成字符型指针
    char* src0_row = (char *) src0->data;
    // 遍历列索引，从0到nr1-1
    for (int col_idx = 0; col_idx < nr1; col_idx++) {
        // 将wdata强制转换为block_q8_0指针，并赋值给nerual
        const block_q8_0 *restrict nerual = (block_q8_0 *)((char *)wdata + col_idx * row_size);
        // 将src2->data强制转换为float指针，并赋值给idx
        idx = (float *)((char *)src2->data + col_idx * idx_row_size);
        // 将vy数组中的元素全部置为0
        memset(vy, 0, ne00 * 4);

        // 遍历ir1，从ir10到ir10+dr-1
        for (int64_t ir1 = ir10; ir1 < ir10 + dr; ir1++)
        {
            // 如果ir1大于等于nr，则跳出循环
            if (ir1 >= nr)
                break;
            // 如果gid[ir1]等于1，则继续下一次循环
            if (gid[ir1] == 1)
                continue;
            // 如果idx[ir1]小于threshold，则继续下一次循环
            if (idx[ir1] < threshold)
                continue;
            // 计算bid和qsid
            int bid = ir1 / QK8_0;
            int qsid = ir1 % QK8_0;
            // 获取nerual[bid].qs[qsid]的值，并赋给b
            int b = (int)nerual[bid].qs[qsid];
            // 如果b等于0，则继续下一次循环
            if (b == 0)
                continue;
            // 获取nerual[bid].d的值，并赋给d
            ggml_fp16_t d = nerual[bid].d;
            // 调用ggml_axpy_q4_0_q8_0函数进行计算
            ggml_axpy_q4_0_q8_0(ne00, src0_row + nb01 * ir1, vy, vy, b, d);
        }

        // 获取锁，如果锁已经被占用，则等待
        while (atomic_flag_test_and_set(&g_axpy_lock))
        {
            // 如果锁已经被占用，则等待
        }

        // 将dst->data强制转换为float指针，并赋值给res
        float *res = (float *)((char *)(dst->data) + col_idx * nb1);
        // 将vy强制转换为float指针，并赋值给tmp
        float *tmp = (float *)vy;
        int i;

        // 计算剩余的元素个数
        int remainder = ne00 % 8;
#if defined(__AVX2__)
        // 如果定义了 AVX2 指令集，则使用 AVX 指令进行向量化计算
        for (i = 0; i < ne00 - remainder; i += 8)
        {
            // 加载 res 中的 8 个浮点数到 AVX 寄存器
            __m256 res_vec = _mm256_loadu_ps(res + i);
            // 加载 tmp 中的 8 个浮点数到 AVX 寄存器
            __m256 tmp_vec = _mm256_loadu_ps(tmp + i);
            // 执行 AVX 指令集中的加法运算
            __m256 result = _mm256_add_ps(res_vec, tmp_vec);
            // 将结果存储到 res 中
            _mm256_storeu_ps(res + i, result);
        }

        // 处理剩余的元素
        for (i = ne00 - remainder; i < ne00; i++)
        {
            res[i] += tmp[i];
        }
#else
        // 如果未定义 AVX2 指令集，则使用普通的循环进行计算
        for (i = 0; i < ne00; i++) {
            res[i] += tmp[i];
        }
#endif
        // 清除全局的原子标志位 g_axpy_lock
        atomic_flag_clear(&g_axpy_lock);
    }

}
// 初始化全局的原子标志位 g_axpy_head_lock
atomic_flag g_axpy_head_lock = ATOMIC_FLAG_INIT;
// 定义一个静态函数 ggml_compute_forward_mul_mat_axpy_head
static void ggml_compute_forward_mul_mat_axpy_head(
        const struct ggml_compute_params * params,
        const struct ggml_tensor * src0,
        const struct ggml_tensor * src1,
              struct ggml_tensor * dst) {
    int64_t t0 = ggml_perf_time_us();
    UNUSED(t0);

    GGML_TENSOR_BINARY_OP_LOCALS;

    // const int ith = params->ith;
    // const int nth = params->nth;

    const enum ggml_type type = src0->type;

    // const bool src1_cont = ggml_is_contiguous(src1);

    // ggml_vec_dot_t    const vec_dot               = type_traits[type].vec_dot;
    enum ggml_type    const vec_dot_type          = type_traits[type].vec_dot_type;
    ggml_from_float_t const from_float_to_vec_dot = type_traits[vec_dot_type].from_float;

    // GGML_ASSERT(ne0 == ne01);
    // GGML_ASSERT(ne1 == ne11);
    // GGML_ASSERT(ne2 == ne12);
    // GGML_ASSERT(ne3 == ne13);

    // we don't support permuted src0 or src1
    GGML_ASSERT(nb00 == ggml_type_size(type));
    GGML_ASSERT(nb10 == sizeof(float));

    // dst cannot be transposed or permuted
    GGML_ASSERT(nb0 == sizeof(float));
    GGML_ASSERT(nb0 <= nb1);
    GGML_ASSERT(nb1 <= nb2);
    GGML_ASSERT(nb2 <= nb3);

    // broadcast factors
    // const int64_t r2 = ne12/ne02;
    // 计算 r3 的值，r3 是 ne13 除以 ne03 的结果

    // 如果任务类型是初始化
    if (params->type == GGML_TASK_INIT) {
        // 将目标数据全部置零
        ggml_set_zero(dst);
        // 如果 src1 的类型不是 vec_dot_type
        if (src1->type != vec_dot_type) {
            // 获取参数中的 wdata 和每行的大小
            char * wdata = params->wdata;
            const size_t row_size = ne10*ggml_type_size(vec_dot_type)/ggml_blck_size(vec_dot_type);

            // 遍历 src1 的维度
            for (int64_t i13 = 0; i13 < ne13; ++i13) {
                for (int64_t i12 = 0; i12 < ne12; ++i12) {
                    for (int64_t i11 = 0; i11 < ne11; ++i11) {
                        // 将 src1 的数据转换成 vec_dot_type 类型，并存储到 wdata 中
                        from_float_to_vec_dot((float *)((char *) src1->data + i13*nb13 + i12*nb12 + i11*nb11), (void *) wdata, ne10);
                        wdata += row_size;
                    }
                }
            }
        }
        // 将原子计数器 params->aic 置零
        atomic_store(params->aic, 0);

        // 返回
        return;
    }

    // 如果任务类型是最终化
    if (params->type == GGML_TASK_FINALIZE) {
        // 返回
        return;
    }

    // 根据 src1 的类型选择 wdata
    const ggml_fp16_t* wdata    = (src1->type == vec_dot_type) ? src1->data : params->wdata;

    // 获取 dst 的第二个源数据
    struct ggml_tensor *src2 = dst->src[2];
    // 计算分块大小
    int chunk = ne00 / 32;
    
    // 根据 src0 的行并行化
    const int64_t dr = (src2->ne[0] + chunk - 1)/chunk;
    const int nr = ggml_nrows(src0);

    // 初始化向量 vec，并将其全部置零
    float vec[ne00*4];
    void *vy = vec;
    memset(vy, 0, ne00*4);
    // 获取 src0 的行数据
    char* src0_row = (char *) src0->data;
    // 进入无限循环，条件为始终为真
    while (true) {
        // 原子操作，将 params->aic 的值加上 dr，并返回加法前的值
        const int ir0 = atomic_fetch_add(params->aic, dr);
        // 下面两行代码被注释掉，不执行
        // int id = ir0 >> 7;
        // if (idx[id] < -15.0f)
        //     continue;
        // 循环，从 ir0 开始，每次加一，直到 ir0+dr，对每个值进行处理
        for (int64_t ir1 = ir0; ir1 < ir0+dr; ir1++) {
            // 如果 ir1 大于等于 nr，则跳出循环
            if (ir1 >= nr) break;
            // 调用 ggml_axpy_avx_f16 函数进行一些操作
            ggml_axpy_avx_f16(ne00, (ggml_fp16_t *)(src0_row+nb01*ir1), (ggml_fp16_t *)vy, vy, wdata[ir1]);
        }
        // 如果 ir0+dr 大于等于 nr，则跳出循环
        if (ir0 + dr >= nr)
            break;
    }
    
    // 进入循环，直到获取到锁
    while (atomic_flag_test_and_set(&g_axpy_head_lock)) {
        // 如果锁已经被占用，则等待
    }
    // 将 dst->data 强制转换为 float 指针，赋值给 res
    float *res = (float *)(dst->data);
    // 将 vy 强制转换为 float 指针，赋值给 tmp
    float *tmp = (float *)vy;
    // 声明整型变量 i
    int i;
 
    // 计算 ne00 除以 8 的余数，赋值给 remainder
    int remainder = ne00 % 8;
#if defined(__AVX2__)
    // 如果定义了 AVX2 指令集，则使用 AVX 指令进行向量化计算
    for (i = 0; i < ne00 - remainder; i += 8) {
        // 加载 res 中的 8 个浮点数到 AVX 寄存器
        __m256 res_vec = _mm256_loadu_ps(res + i);
        // 加载 tmp 中的 8 个浮点数到 AVX 寄存器
        __m256 tmp_vec = _mm256_loadu_ps(tmp + i);
        // 执行 AVX 指令集中的加法运算
        __m256 result = _mm256_add_ps(res_vec, tmp_vec);
        // 将结果存储到 res 中
        _mm256_storeu_ps(res + i, result);
    }

    // 处理剩余的元素
    for (i = ne00 - remainder; i < ne00; i++) {
        res[i] += tmp[i];
    }
#else
    // 如果未定义 AVX2 指令集，则使用普通的循环进行计算
    for (i = 0; i < ne00; i++) {
        res[i] += tmp[i];
    }
#endif
    // 清除全局变量 g_axpy_head_lock 的原子标志
    atomic_flag_clear(&g_axpy_head_lock);
}

/////////////////////////////////

static void ggml_compute_forward(struct ggml_compute_params * params, struct ggml_tensor * tensor) {
    // 断言参数 params 不为空
    GGML_ASSERT(params);

    // 如果张量的操作为 GGML_OP_NONE，则直接返回
    if (tensor->op == GGML_OP_NONE) {
        return;
    }

#ifdef GGML_USE_CUBLAS
    // 如果定义了 GGML_USE_CUBLAS，则调用 ggml_cuda_compute_forward 函数进行计算
    bool skip_cpu = ggml_cuda_compute_forward(params, tensor);
    // 如果 skip_cpu 为 true，则直接返回
    if (skip_cpu) {
        return;
    }
    // 断言张量的源张量为 NULL 或者其后端为 GGML_BACKEND_CPU
    GGML_ASSERT(tensor->src[0] == NULL || tensor->src[0]->backend == GGML_BACKEND_CPU);
    GGML_ASSERT(tensor->src[1] == NULL || tensor->src[1]->backend == GGML_BACKEND_CPU);
#endif // GGML_USE_CUBLAS
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
    # 如果 l 小于 n_primes，则 sz 等于 primes[l]，否则 sz 等于 min_sz 与 1 的最小值
    size_t sz = l < n_primes ? primes[l] : min_sz | 1;
    # 返回 sz 的值
    return sz;
}

// 计算指针的哈希值
static size_t ggml_hash(const void * p) {
    return (size_t)p;
}

// 在哈希表中查找指定键的位置
size_t ggml_hash_find(const struct ggml_hash_set hash_set, struct ggml_tensor * key) {
    size_t h = ggml_hash(key) % hash_set.size;

    // 线性探测
    size_t i = h;
    while (hash_set.keys[i] != NULL && hash_set.keys[i] != key) {
        i = (i + 1) % hash_set.size;
        if (i == h) {
            // 遍历了所有哈希表条目 -> 未找到
            return GGML_HASHTABLE_FULL;
        }
    }
    return i;
}

// 检查哈希表中是否包含指定键
bool ggml_hash_contains(struct ggml_hash_set hash_set, struct ggml_tensor * key) {
    size_t i = ggml_hash_find(hash_set, key);
    return i != GGML_HASHTABLE_FULL && hash_set.keys[i] == key;
}

// 向哈希表中插入指定键
size_t ggml_hash_insert(struct ggml_hash_set hash_set, struct ggml_tensor * key) {
    size_t i = ggml_hash_find(hash_set, key);

    GGML_ASSERT(i != GGML_HASHTABLE_FULL);

    if (hash_set.keys[i] == key) {
        return GGML_HASHTABLE_ALREADY_EXISTS;
    }

    // 插入
    GGML_ASSERT(hash_set.keys[i] == NULL);
    hash_set.keys[i] = key;
    return i;
}

// 查找或插入指定键
size_t ggml_hash_find_or_insert(struct ggml_hash_set hash_set, struct ggml_tensor * key) {
    size_t i = ggml_hash_find(hash_set, key);

    GGML_ASSERT(i != GGML_HASHTABLE_FULL);

    hash_set.keys[i] = key;
    return i;
}

// 创建新的哈希集合
static struct ggml_hash_set ggml_hash_set_new(size_t size) {
    size = ggml_hash_size(size);
    struct ggml_hash_set result;
    result.size = size;
    result.keys = malloc(sizeof(struct ggml_tensor *) * size);
    memset(result.keys, 0, sizeof(struct ggml_tensor *) * size);
    return result;
}

// 释放哈希集合
static void ggml_hash_set_free(struct ggml_hash_set hash_set) {
    free(hash_set.keys);
}

// 哈希映射结构
struct hash_map {
    struct ggml_hash_set set;
    struct ggml_tensor ** vals;
};

// 创建新的哈希映射
static struct hash_map * ggml_new_hash_map(size_t size) {
    struct hash_map * result = malloc(sizeof(struct hash_map));
    result->set = ggml_hash_set_new(size);
    # 为 result->vals 分配内存，大小为 result->set.size 个 ggml_tensor 指针的大小
    result->vals = malloc(sizeof(struct ggml_tensor *) * result->set.size);
    # 将 result->vals 的内存空间初始化为 0，大小为 result->set.size 个 ggml_tensor 指针的大小
    memset(result->vals, 0, sizeof(struct ggml_tensor *) * result->set.size);
    # 返回 result 结果
    return result;
// 释放哈希映射结构体及其内存
static void ggml_hash_map_free(struct hash_map * map) {
    // 释放哈希集合
    ggml_hash_set_free(map->set);
    // 释放值数组内存
    free(map->vals);
    // 释放哈希映射结构体内存
    free(map);
}

// 梯度检查点

static struct ggml_tensor * ggml_recompute_graph_node(
        struct ggml_context * ctx,
        struct ggml_cgraph  * graph,
        struct hash_map     * replacements,
        struct ggml_tensor  * node) {

    // 如果节点为空，则返回空
    if (node == NULL) {
        return NULL;
    }

    // 如果节点是参数，则返回节点本身
    if (node->is_param) {
        return node;
    }

    // 如果节点不在已访问的哈希表中，则返回节点本身
    if (!ggml_hash_contains(graph->visited_hash_table, node)) {
        return node;
    }

    // 计算节点的子节点数量
    int count_children = 0;
    for (int k = 0; k < GGML_MAX_SRC; ++k) {
        if (node->src[k]) {
            ++count_children;
        }
    }

    // 如果节点没有子节点，则返回节点本身
    if (count_children == 0) {
        return node;
    }

    // 在替换哈希表中查找节点
    size_t i = ggml_hash_find(replacements->set, node);
    GGML_ASSERT(i != GGML_HASHTABLE_FULL); // 断言哈希表不是满的
    if (replacements->set.keys[i] == node) {
        return replacements->vals[i];
    }

    // 创建节点的克隆
    struct ggml_tensor * clone = ggml_new_tensor(ctx, node->type, node->n_dims, node->ne);

    // 将克隆节点插入替换哈希表中
    GGML_ASSERT(replacements->set.keys[i] == NULL); // 断言不会覆盖
    replacements->set.keys[i] = node;
    replacements->vals[i] = clone;

    // 复制节点的属性和子节点
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
    # 如果节点的视图源不为空
    if (node->view_src != NULL) {
        # 如果克隆节点的数据为空
        clone->data = (node->view_src->data == NULL)
                        ? NULL // 视图源尚未分配
                        : (char *) node->view_src->data // 视图源已经分配
                                 + node->view_offs;
        # 将克隆节点的视图源设置为节点的视图源
        clone->view_src  = node->view_src;
        # 将克隆节点的视图偏移设置为节点的视图偏移
        clone->view_offs = node->view_offs;
    }

    # 确保节点的操作参数占用的内存大小符合预期
    GGML_ASSERT(sizeof(node->op_params) == sizeof(int32_t) * (GGML_MAX_OP_PARAMS / sizeof(int32_t)));
    # 确保节点的名称占用的内存大小符合预期
    GGML_ASSERT(sizeof(node->name)      == GGML_MAX_NAME);
    # 将节点的操作参数复制到克隆节点的操作参数
    memcpy(clone->op_params, node->op_params, sizeof(node->op_params));
    # 格式化克隆节点的名称，添加 "(clone)" 后缀
    ggml_format_name(clone, "%s (clone)", ggml_get_name(node));

    # 返回克隆节点
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

    // 如果检查点数量小于等于0，直接复制临时图到反向图并返回
    if (n_checkpoints <= 0) {
        ggml_graph_cpy(gb_tmp, gb);
        return;
    }

    // 创建替换哈希表
    struct hash_map * replacements = ggml_new_hash_map(gf->n_nodes + gf->n_leafs + n_checkpoints);

    // 在替换哈希表中插入检查点
    for (int i = 0; i < n_checkpoints; ++i) {
        size_t k = ggml_hash_find(replacements->set, checkpoints[i]);
        GGML_ASSERT(k != GGML_HASHTABLE_FULL); // 断言哈希表不是满的
        GGML_ASSERT(replacements->set.keys[k] == NULL); // 断言不会覆盖已有键
        replacements->set.keys[k] = checkpoints[i];
        replacements->vals[k]     = checkpoints[i];
    }

    // 复制前向图到反向图
    ggml_graph_cpy(gf, gb);
    // 重写 gb_tmp->nodes[gf->n_nodes:gb_tmp->n_nodes]，用检查点重新计算替换 gf->nodes[0:gf->n_nodes] 的引用
    for (int i = gf->n_nodes; i<gb_tmp->n_nodes; ++i) {
        struct ggml_tensor * node = gb_tmp->nodes[i];
        for (int k = 0; k < GGML_MAX_SRC; ++k) {
            // 插入重新计算的新张量，重用已经做好的替换
            // 记住替换：使用对应的 gf 节点映射新张量
            // 对输入张量进行递归，除非输入张量是替换（如检查点）时终止
            node->src[k] = ggml_recompute_graph_node(ctx, gf, replacements, node->src[k]);
        }
        // 插入带有已做替换的重写后向节点到结果反向图 gb
        ggml_build_forward_expand(gb, node);
    }

    // 释放替换哈希表
    ggml_hash_map_free(replacements);
}
// 定义一个函数，用于根据输入的 a 是否在 zero_table 中来决定返回 b 或者调用 ggml_add_impl 函数
static struct ggml_tensor * ggml_add_or_set(struct ggml_context * ctx, struct ggml_tensor * a, struct ggml_tensor * b, struct ggml_hash_set zero_table) {
    // 如果 a 在 zero_table 中，则返回 b
    if (ggml_hash_contains(zero_table, a)) {
        return b;
    } else {
        // 否则调用 ggml_add_impl 函数
        return ggml_add_impl(ctx, a, b, false);
    }
}

// 定义一个函数，用于根据输入的 a 是否在 zero_table 中来决定返回经过处理的 a 或者调用 ggml_acc_impl 函数
static struct ggml_tensor * ggml_acc_or_set(struct ggml_context * ctx, struct ggml_tensor * a, struct ggml_tensor * b, size_t nb1, size_t nb2, size_t nb3, size_t offset, struct ggml_hash_set zero_table) {
    // 如果 a 在 zero_table 中
    if (ggml_hash_contains(zero_table, a)) {
        // 创建一个值为 0 的新 tensor a_zero，并返回调用 ggml_acc_impl 函数
        struct ggml_tensor * a_zero = ggml_scale(ctx, a, ggml_new_f32(ctx, 0));
        return ggml_acc_impl(ctx, a_zero, b, nb1, nb2, nb3, offset, false);
    } else {
        // 否则直接调用 ggml_acc_impl 函数
        return ggml_acc_impl(ctx, a, b, nb1, nb2, nb3, offset, false);
    }
}

// 定义一个函数，用于根据输入的 a 是否在 zero_table 中来决定返回重复 b 或者调用 ggml_add1_impl 函数
static struct ggml_tensor * ggml_add1_or_set(struct ggml_context * ctx, struct ggml_tensor * a, struct ggml_tensor * b, struct ggml_hash_set zero_table) {
    // 如果 a 在 zero_table 中，则返回重复 b 的结果
    if (ggml_hash_contains(zero_table, a)) {
        return ggml_repeat(ctx, b, a);
    } else {
        // 否则调用 ggml_add1_impl 函数
        return ggml_add1_impl(ctx, a, b, false);
    }
}

// 定义一个函数，用于根据输入的 a 是否在 zero_table 中来决定返回 b 的相反数 或者调用 ggml_sub_impl 函数
static struct ggml_tensor * ggml_sub_or_set(struct ggml_context * ctx, struct ggml_tensor * a, struct ggml_tensor * b, struct ggml_hash_set zero_table) {
    // 如果 a 在 zero_table 中，则返回 b 的相反数
    if (ggml_hash_contains(zero_table, a)) {
        return ggml_neg(ctx, b);
    } else {
        // 否则调用 ggml_sub_impl 函数
        return ggml_sub_impl(ctx, a, b, false);
    }
}

// 定义一个函数，用于计算反向传播
static void ggml_compute_backward(struct ggml_context * ctx, struct ggml_tensor * tensor, struct ggml_hash_set zero_table) {
    // 获取 tensor 的两个源 tensor
    struct ggml_tensor * src0 = tensor->src[0];
    struct ggml_tensor * src1 = tensor->src[1];

    // 遍历 tensor 的源 tensor
    for (int i = 0; i < GGML_MAX_SRC; ++i) {
        // 如果源 tensor 存在并且具有梯度
        if (tensor->src[i] && tensor->src[i]->grad) {
            // 断言源 tensor 和其梯度具有相同的形状
            GGML_ASSERT(ggml_are_same_shape(tensor->src[i], tensor->src[i]->grad));
        }
    }
}
# 访问父节点，构建计算图
static void ggml_visit_parents(struct ggml_cgraph * cgraph, struct ggml_tensor * node) {
    # 如果节点的梯度为空
    if (node->grad == NULL) {
        # 这通常发生在我们从反向传播中的常量生成中间节点时
        # 它也可能在前向传播期间发生，如果用户使用常量进行计算
        if (node->op != GGML_OP_NONE) {
            # 打印警告信息，节点没有梯度，但是有操作
            #GGML_PRINT_DEBUG("%s: warning: node %p has no grad, but op %d\n", __func__, (void *) node, node->op);
        }
    }

    # 检查节点是否已经被访问过
    if (ggml_hash_insert(cgraph->visited_hash_table, node) == GGML_HASHTABLE_ALREADY_EXISTS) {
        return;
    }

    # 遍历节点的源节点
    for (int i = 0; i < GGML_MAX_SRC; ++i) {
        const int k =
            (cgraph->order == GGML_CGRAPH_EVAL_ORDER_LEFT_TO_RIGHT) ? i :
            (cgraph->order == GGML_CGRAPH_EVAL_ORDER_RIGHT_TO_LEFT) ? (GGML_MAX_SRC-1-i) :
            /* unknown order, just fall back to using i*/ i;
        if (node->src[k]) {
            ggml_visit_parents(cgraph, node->src[k]);
        }
    }

    # 如果节点的操作为无，且梯度为空
    if (node->op == GGML_OP_NONE && node->grad == NULL) {
        # 达到叶节点，不属于梯度图（例如常量）
        GGML_ASSERT(cgraph->n_leafs < cgraph->size);

        # 如果节点名称长度为0，则格式化节点名称
        if (strlen(node->name) == 0) {
            ggml_format_name(node, "leaf_%d", cgraph->n_leafs);
        }

        cgraph->leafs[cgraph->n_leafs] = node;
        cgraph->n_leafs++;
        # 原子操作，将节点的is_finish标志设置为1
        atomic_store(&(node->is_finish), 1); 
    } else {
        # 断言，计算图中节点数小于计算图大小
        GGML_ASSERT(cgraph->n_nodes < cgraph->size);

        # 如果节点名称长度为0，则格式化节点名称
        if (strlen(node->name) == 0) {
            ggml_format_name(node, "node_%d", cgraph->n_nodes);
        }

        cgraph->nodes[cgraph->n_nodes] = node;
        # 如果存在梯度，则将节点的梯度存入grads数组中
        if (cgraph->grads) {
            cgraph->grads[cgraph->n_nodes] = node->grad;
        }
        cgraph->n_nodes++;
    }
}

# 构建前向传播实现
static void ggml_build_forward_impl(struct ggml_cgraph * cgraph, struct ggml_tensor * tensor, bool expand) {
    // 如果不需要扩展，则清空图形对象
    if (!expand) {
        // TODO: this branch isn't accessible anymore, maybe move this to ggml_build_forward_expand
        ggml_graph_clear(cgraph);
    }

    // 获取当前图形对象的节点数量
    const int n0 = cgraph->n_nodes;
    // 未使用的变量，用于标记节点数量

    // 访问父节点
    ggml_visit_parents(cgraph, tensor);

    // 计算新添加的节点数量
    const int n_new = cgraph->n_nodes - n0;
    // 打印调试信息，显示访问了多少个新节点
    GGML_PRINT_DEBUG("%s: visited %d new nodes\n", __func__, n_new);

    // 如果有新节点被访问
    if (n_new > 0) {
        // 最后添加的节点应该始终是起始点
        GGML_ASSERT(cgraph->nodes[cgraph->n_nodes - 1] == tensor);
    }
// 构建前向传播扩展函数，接受计算图和张量作为参数
void ggml_build_forward_expand(struct ggml_cgraph * cgraph, struct ggml_tensor * tensor) {
    // 调用内部的前向传播实现函数
    ggml_build_forward_impl(cgraph, tensor, true);
}

// 构建反向传播扩展函数，接受上下文、梯度图、反向梯度图和是否保留梯度作为参数
void ggml_build_backward_expand(struct ggml_context * ctx, struct ggml_cgraph * gf, struct ggml_cgraph * gb, bool keep) {
    // 断言梯度图的节点数大于0
    GGML_ASSERT(gf->n_nodes > 0);

    // 如果需要保留梯度图，需要将梯度节点从原始图中分离出来
    if (keep) {
        for (int i = 0; i < gf->n_nodes; i++) {
            struct ggml_tensor * node = gf->nodes[i];

            if (node->grad) {
                // 复制梯度节点并将其赋值给原始图的梯度数组
                node->grad = ggml_dup_tensor(ctx, node);
                gf->grads[i] = node->grad;
            }
        }
    }

    // 记录初始梯度值为零的梯度节点
    struct ggml_hash_set zero_table = ggml_hash_set_new(gf->size);
    for (int i = 0; i < gf->n_nodes; i++) {
        if (gf->grads[i]) {
            ggml_hash_insert(zero_table, gf->grads[i]);
        }
    }

    // 从后向前遍历节点，计算梯度
    for (int i = gf->n_nodes - 1; i >= 0; i--) {
        struct ggml_tensor * node = gf->nodes[i];

        // 对于具有梯度的节点，使用反向传播计算梯度
        if (node->grad) {
            ggml_compute_backward(ctx, node, zero_table);
        }
    }

    // 遍历节点，如果是参数节点，则调用前向传播扩展函数
    for (int i = 0; i < gf->n_nodes; i++) {
        struct ggml_tensor * node = gf->nodes[i];

        if (node->is_param) {
            GGML_PRINT_DEBUG("%s: found root node %p\n", __func__, (void *) node);
            ggml_build_forward_expand(gb, node->grad);
        }
    }

    // 释放零值表
    ggml_hash_set_free(zero_table);
}

// 计算图占用内存的大小
static size_t ggml_graph_nbytes(size_t size, bool grads) {
    size_t nbytes = sizeof(struct ggml_cgraph);
    nbytes += size * sizeof(struct ggml_tensor *) * 2; // 叶子节点 + 普通节点
    if (grads) {
        nbytes += size * sizeof(struct ggml_tensor *); // 梯度节点
    }
    nbytes += ggml_hash_size(size * 2) * sizeof(struct ggml_tensor *); // 哈希集合
    # 返回变量nbytes的值
    return nbytes;
# 计算图对象的额外开销，包括对象大小和内存对齐
size_t ggml_graph_overhead_custom(size_t size, bool grads) {
    return GGML_OBJECT_SIZE + GGML_PAD(ggml_graph_nbytes(size, grads), GGML_MEM_ALIGN);
}

# 返回默认大小的图对象的额外开销
size_t ggml_graph_overhead(void) {
    return ggml_graph_overhead_custom(GGML_DEFAULT_GRAPH_SIZE, false);
}

# 创建自定义大小的图对象
struct ggml_cgraph * ggml_new_graph_custom(struct ggml_context * ctx, size_t size, bool grads) {
    # 计算对象大小
    const size_t obj_size = ggml_graph_nbytes(size, grads);
    # 创建新的对象
    struct ggml_object * obj = ggml_new_object(ctx, GGML_OBJECT_GRAPH, obj_size);
    # 计算图对象的指针
    struct ggml_cgraph * cgraph = (struct ggml_cgraph *) ((char *) ctx->mem_buffer + obj->offs);

    # 计算数据起始位置
    struct ggml_tensor ** data_start = (struct ggml_tensor **) (cgraph + 1);

    # 计算哈希表大小
    size_t hash_size = ggml_hash_size(size * 2);
    # 计算节点指针位置
    struct ggml_tensor ** nodes_ptr = data_start;
    # 计算叶子节点指针位置
    struct ggml_tensor ** leafs_ptr = nodes_ptr + size;
    # 计算哈希键指针位置
    struct ggml_tensor ** hash_keys_ptr = leafs_ptr + size;
    # 如果有梯度，则计算梯度指针位置
    struct ggml_tensor ** grads_ptr = grads ? hash_keys_ptr + hash_size : NULL;

    # 检查是否分配了正确数量的内存
    assert(obj_size == (size_t) (
        (grads ? (char *)(grads_ptr + size) : (char *)(hash_keys_ptr + hash_size)) - (char *)cgraph));

    # 将哈希键指针位置的内存清零
    memset(hash_keys_ptr, 0, hash_size * sizeof(struct ggml_tensor *));

    # 初始化图对象的属性
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

    # 返回创建的图对象
    return cgraph;
}

# 创建默认大小的图对象
struct ggml_cgraph * ggml_new_graph(struct ggml_context * ctx) {
    return ggml_new_graph_custom(ctx, GGML_DEFAULT_GRAPH_SIZE, false);
}

# 创建图对象的视图
struct ggml_cgraph * ggml_graph_view(struct ggml_context * ctx, struct ggml_cgraph * cgraph0, int i0, int i1) {
    // 定义结构体 ggml_cgraph 的大小
    const size_t obj_size = sizeof(struct ggml_cgraph);
    // 在上下文 ctx 中创建一个类型为 GGML_OBJECT_GRAPH，大小为 obj_size 的新对象
    struct ggml_object * obj = ggml_new_object(ctx, GGML_OBJECT_GRAPH, obj_size);
    // 将内存缓冲区中的偏移量为 obj->offs 的位置解释为 ggml_cgraph 结构体
    struct ggml_cgraph * cgraph = (struct ggml_cgraph *) ((char *) ctx->mem_buffer + obj->offs);

    // 初始化 cgraph 结构体
    *cgraph = (struct ggml_cgraph) {
        /*.size         =*/ 0,  // 结构体大小为 0
        /*.n_nodes      =*/ i1 - i0,  // 节点数为 i1 - i0
        /*.n_leafs      =*/ 0,  // 叶子节点数为 0
        /*.nodes        =*/ cgraph0->nodes + i0,  // 节点指针指向 cgraph0->nodes + i0
        /*.grads        =*/ cgraph0->grads ? cgraph0->grads + i0 : NULL,  // 梯度指针指向 cgraph0->grads + i0，如果 cgraph0->grads 为真，否则为 NULL
        /*.leafs        =*/ NULL,  // 叶子节点指针为 NULL
        /*.hash_table   =*/ { 0, NULL },  // 哈希表初始化为 { 0, NULL }
        /*.order        =*/ cgraph0->order,  // 顺序与 cgraph0 相同
        /*.perf_runs    =*/ 0,  // 性能运行次数为 0
        /*.perf_cycles  =*/ 0,  // 性能周期数为 0
        /*.perf_time_us =*/ 0,  // 性能时间为 0 微秒
    };

    // 返回初始化后的 cgraph 结构体指针
    return cgraph;
// 复制一个图形结构体，将源结构体的内容复制到目标结构体中
void ggml_graph_cpy(struct ggml_cgraph * src, struct ggml_cgraph * dst) {
    // 确保目标结构体的大小大于等于源结构体的叶子节点数
    GGML_ASSERT(dst->size >= src->n_leafs);
    // 确保目标结构体的大小大于等于源结构体的节点数
    GGML_ASSERT(dst->size >= src->n_nodes);
    // 确保目标结构体的访问哈希表大小大于等于源结构体的访问哈希表大小
    GGML_ASSERT(dst->visited_hash_table.size >= src->visited_hash_table.size);

    // 将目标结构体的叶子节点数设置为源结构体的叶子节点数
    dst->n_leafs = src->n_leafs;
    // 将目标结构体的节点数设置为源结构体的节点数
    dst->n_nodes = src->n_nodes;
    // 将目标结构体的顺序设置为源结构体的顺序
    dst->order   = src->order;

    // 复制源结构体的叶子节点到目标结构体
    for (int i = 0; i < src->n_leafs; ++i) {
        dst->leafs[i] = src->leafs[i];
    }

    // 复制源结构体的节点到目标结构体
    for (int i = 0; i < src->n_nodes; ++i) {
        dst->nodes[i] = src->nodes[i];
    }

    // 如果源结构体的梯度存在
    if (src->grads) {
        // 确保目标结构体的梯度不为空
        GGML_ASSERT(dst->grads != NULL);
        // 复制源结构体的梯度到目标结构体
        for (int i = 0; i < src->n_nodes; ++i) {
            dst->grads[i] = src->grads[i];
        }
    }

    // 遍历源结构体的访问哈希表，将键插入到目标结构体的访问哈希表中
    for (size_t i = 0; i < src->visited_hash_table.size; ++i) {
        if (src->visited_hash_table.keys[i]) {
            ggml_hash_insert(dst->visited_hash_table, src->visited_hash_table.keys[i]);
        }
    }
}

// 复制一个图形结构体，并返回复制后的结构体指针
struct ggml_cgraph * ggml_graph_dup(struct ggml_context * ctx, struct ggml_cgraph * cgraph) {
    // 创建一个新的图形结构体，使用给定的上下文和大小，以及是否存在梯度
    struct ggml_cgraph * result = ggml_new_graph_custom(ctx, cgraph->size, cgraph->grads != NULL);
    // 复制源图形结构体的内容到新创建的图形结构体中
    ggml_graph_cpy(cgraph, result);
    // 返回新创建的图形结构体指针
    return result;
}

// 重置图形结构体的梯度为零
void ggml_graph_reset(struct ggml_cgraph * cgraph) {
    // 确保图形结构体的梯度不为空
    GGML_ASSERT(cgraph->grads != NULL);

    // 遍历图形结构体的节点，将每个节点的梯度设置为零
    for (int i = 0; i < cgraph->n_nodes; i++) {
        struct ggml_tensor * grad = cgraph->grads[i];

        if (grad) {
            ggml_set_zero(grad);
        }
    }
}

// 清空图形结构体的叶子节点数、节点数和访问哈希表的键
void ggml_graph_clear(struct ggml_cgraph * cgraph) {
    // 将图形结构体的叶子节点数设置为零
    cgraph->n_leafs = 0;
    // 将图形结构体的节点数设置为零
    cgraph->n_nodes = 0;
    // 将图形结构体的访问哈希表的键全部设置为零
    memset(cgraph->visited_hash_table.keys, 0, cgraph->visited_hash_table.size * sizeof(struct ggml_tensor *));
}

//
// 线程数据
//
// 通过忙循环进行同步
// 尝试使用自旋锁，但不确定如何正确使用 - 尝试过的方法比忙循环慢
//

#ifdef __APPLE__

//#include <os/lock.h>
//
//typedef os_unfair_lock ggml_lock_t;
//
//#define ggml_lock_init(x)    UNUSED(x)
// 定义宏 ggml_lock_destroy，将参数 x 替换为 UNUSED(x)
#define ggml_lock_destroy(x) UNUSED(x)
// 定义宏 ggml_lock_lock，将参数 x 替换为 os_unfair_lock_lock
#define ggml_lock_lock       os_unfair_lock_lock
// 定义宏 ggml_lock_unlock，将参数 x 替换为 os_unfair_lock_unlock
#define ggml_lock_unlock     os_unfair_lock_unlock

// 定义宏 GGML_LOCK_INITIALIZER，将其替换为 OS_UNFAIR_LOCK_INIT
#define GGML_LOCK_INITIALIZER OS_UNFAIR_LOCK_INIT

// 定义类型别名 ggml_lock_t 为 int
typedef int ggml_lock_t;

// 定义宏 ggml_lock_init，将参数 x 替换为 UNUSED(x)
#define ggml_lock_init(x)    UNUSED(x)
// 定义宏 ggml_lock_destroy，将参数 x 替换为 UNUSED(x)
#define ggml_lock_destroy(x) UNUSED(x)
// 定义宏 ggml_lock_lock，将参数 x 替换为 UNUSED(x)
#define ggml_lock_lock(x)    UNUSED(x)
// 定义宏 ggml_lock_unlock，将参数 x 替换为 UNUSED(x)
#define ggml_lock_unlock(x)  UNUSED(x)

// 定义宏 GGML_LOCK_INITIALIZER 为 0
#define GGML_LOCK_INITIALIZER 0

// 定义类型别名 ggml_thread_t 为 pthread_t
typedef pthread_t ggml_thread_t;

// 定义宏 ggml_thread_create，将其替换为 pthread_create
#define ggml_thread_create pthread_create
// 定义宏 ggml_thread_join，将其替换为 pthread_join
#define ggml_thread_join   pthread_join

#else

//typedef pthread_spinlock_t ggml_lock_t;

// 定义类型别名 ggml_lock_t 为 int
typedef int ggml_lock_t;

// 定义宏 ggml_lock_init，将参数 x 替换为 UNUSED(x)
#define ggml_lock_init(x)    UNUSED(x)
// 定义宏 ggml_lock_destroy，将参数 x 替换为 UNUSED(x)
#define ggml_lock_destroy(x) UNUSED(x)
// 如果定义了 __x86_64__ 或者 (_MSC_VER 和 _M_AMD64)，则定义宏 ggml_lock_lock 为 _mm_pause()
#define ggml_lock_lock(x)    _mm_pause()
// 否则，定义宏 ggml_lock_lock，将参数 x 替换为 UNUSED(x)
#define ggml_lock_lock(x)    UNUSED(x)
// 定义宏 ggml_lock_unlock，将参数 x 替换为 UNUSED(x)
#define ggml_lock_unlock(x)  UNUSED(x)

// 定义宏 GGML_LOCK_INITIALIZER 为 0
#define GGML_LOCK_INITIALIZER 0

// 定义类型别名 ggml_thread_t 为 pthread_t
typedef pthread_t ggml_thread_t;

// 定义宏 ggml_thread_create，将其替换为 pthread_create
#define ggml_thread_create pthread_create
// 定义宏 ggml_thread_join，将其替换为 pthread_join
#define ggml_thread_join   pthread_join

#endif

// 如果定义了 __linux__ 并且未定义 __BIONIC__
#if defined(__linux__) && !defined(__BIONIC__)
// 定义静态函数 set_numa_thread_affinity，设置线程亲和性
static void set_numa_thread_affinity(int thread_n, int n_threads) {
    // 如果不是 NUMA 架构，直接返回
    if (!ggml_is_numa()) {
        return;
    }

    // 将线程运行在特定的 NUMA 节点上
    // 计算线程应该运行的节点编号
    const int node_num = thread_n / ((n_threads + g_state.numa.n_nodes - 1) / g_state.numa.n_nodes);
    // 获取对应节点的信息
    struct ggml_numa_node * node = &g_state.numa.nodes[node_num];
    // 计算 CPU 集合的大小
    size_t setsize = CPU_ALLOC_SIZE(g_state.numa.total_cpus);

    // 创建 CPU 集合
    cpu_set_t * cpus = CPU_ALLOC(g_state.numa.total_cpus);
    // 清空 CPU 集合
    CPU_ZERO_S(setsize, cpus);
    // 将节点上的 CPU 添加到集合中
    for (size_t i = 0; i < node->n_cpus; ++i) {
        CPU_SET_S(node->cpus[i], setsize, cpus);
    }
    # 设置当前线程的 CPU 亲和性，将其绑定到指定的 CPU 上
    int rv = pthread_setaffinity_np(pthread_self(), setsize, cpus);
    # 如果设置失败，输出警告信息
    if (rv) {
            fprintf(stderr, "warning: pthread_setaffinity_np() failed: %s\n",
                    strerror(rv));
    }

    # 释放 CPU 集合的内存
    CPU_FREE(cpus);
}
// 清除 NUMA 线程亲和性设置
static void clear_numa_thread_affinity(void) {
    // 如果不是 NUMA 架构，直接返回
    if (!ggml_is_numa()) {
        return;
    }

    // 计算 CPU 集合的大小
    size_t setsize = CPU_ALLOC_SIZE(g_state.numa.total_cpus);

    // 分配 CPU 集合内存并清零
    cpu_set_t * cpus = CPU_ALLOC(g_state.numa.total_cpus);
    CPU_ZERO_S(setsize, cpus);
    // 设置每个 CPU 的位
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

    // 释放 CPU 集合内存
    CPU_FREE(cpus);
}
#else
// TODO: Windows etc.
// (the linux implementation may also work on BSD, someone should test)
// 设置 NUMA 线程亲和性（在 Windows 等系统上的实现）
static void set_numa_thread_affinity(int thread_n, int n_threads) { UNUSED(thread_n); UNUSED(n_threads);  }
// 清除 NUMA 线程亲和性（在 Windows 等系统上的实现）
static void clear_numa_thread_affinity(void) {}
#endif

// 共享的计算状态结构体
struct ggml_compute_state_shared {
    const struct ggml_cgraph * cgraph;  // 指向计算图的指针
    const struct ggml_cplan  * cplan;   // 指向计算计划的指针

    int64_t perf_node_start_cycles;  // 性能统计起始周期数
    int64_t perf_node_start_time_us;  // 性能统计起始时间（微秒）

    const int n_threads;  // 线程数
    atomic_int  aic;  // 原子整型变量

    // 同步原语
    atomic_int n_active;  // 活跃线程数
    atomic_int node_n;    // 活跃图节点

    bool (*abort_callback)(void * data);  // 当为 true 时中止 ggml_graph_compute
    void * abort_callback_data;  // 中止回调数据
};

// 计算状态结构体
struct ggml_compute_state {
    ggml_thread_t thrd;  // 线程
    int ith;  // 线程索引
    struct ggml_compute_state_shared * shared;  // 指向共享计算状态的指针
};

// 计算性能统计节点（CPU）
static void ggml_graph_compute_perf_stats_node(struct ggml_tensor * node, const struct ggml_compute_state_shared * st) {
    // 计算当前周期数和时间
    int64_t cycles_cur  = ggml_perf_cycles()  - st->perf_node_start_cycles;
    int64_t time_us_cur = ggml_perf_time_us() - st->perf_node_start_time_us;

    // 更新节点的性能统计信息
    node->perf_runs++;
    node->perf_cycles  += cycles_cur;
    node->perf_time_us += time_us_cur;
}
// 计算性能统计节点（GPU）
static void ggml_graph_compute_perf_stats_node_gpu(struct ggml_tensor * node, const struct ggml_compute_state_shared * st) {
    # 计算当前时刻的 CPU 循环数与起始时刻的 CPU 循环数之差
    int64_t cycles_cur  = ggml_perf_cycles()  - st->perf_node_start_cycles;
    # 计算当前时刻的时间与起始时刻的时间之差（单位：微秒）
    int64_t time_us_cur = ggml_perf_time_us() - st->perf_node_start_time_us;

    # 性能统计：运行次数加2
    node->perf_runs+=2;
    # 性能统计：累加 CPU 循环数差
    node->perf_cycles  += cycles_cur;
    # 性能统计：累加时间差（单位：微秒）
    node->perf_time_us += time_us_cur;
    // 获取节点需要执行的任务数量
static int ggml_get_n_tasks(struct ggml_tensor * node, int n_threads) {
    int n_tasks = 0;

#if defined(GGML_USE_CUBLAS)
                // 如果可以使用 CUDA 进行矩阵乘法运算，则任务数量为1
                if (ggml_cuda_can_mul_mat(node->src[0], node->src[1], node)) {
                    n_tasks = 1; // TODO: this actually is doing nothing
                                 //       the threads are still spinning
                }
#elif defined(GGML_USE_CLBLAST)
                // 如果可以使用 CLBLAST 进行矩阵乘法运算，则任务数量为1
                if (ggml_cl_can_mul_mat(node->src[0], node->src[1], node)) {
                    n_tasks = 1; // TODO: this actually is doing nothing
                                 //       the threads are still spinning
                }
#endif
#if defined(GGML_USE_ACCELERATE) || defined(GGML_USE_OPENBLAS)
                // 如果可以使用 ACCELERATE 或 OPENBLAS 进行矩阵乘法运算，则任务数量为1
                if (ggml_compute_forward_mul_mat_use_blas(node->src[0], node->src[1], node)) {
                    n_tasks = 1; // TODO: this actually is doing nothing
                                 //       the threads are still spinning
                }
    }

    // 断言任务数量大于0
    assert(n_tasks > 0);

    // 返回任务数量
    return n_tasks;
}

// 计算图的计算线程
static thread_ret_t ggml_graph_compute_thread(void * data) {
    // 获取计算状态
    struct ggml_compute_state * state = (struct ggml_compute_state *) data;

    // 获取计算图和计划
    const struct ggml_cgraph * cgraph = state->shared->cgraph;
    const struct ggml_cplan  * cplan  = state->shared->cplan;

    // 获取线程数量
    const int   n_threads   = state->shared->n_threads;

    // 设置 NUMA 线程亲和性
    set_numa_thread_affinity(state->ith, n_threads);

    // 初始化节点编号
    int node_n = -1;

#if defined(GGML_USE_ACCELERATE) || defined(GGML_USE_OPENBLAS)
                // 调度线程
                sched_yield();
#endif

                // 从共享状态中原子加载节点数量
                node_n = atomic_load(&state->shared->node_n);
                // 如果节点数量不等于上一次的数量，则跳出循环
                if (node_n != last) break;
            };
        }

        // 检查是否应该停止
        if (node_n >= cgraph->n_nodes) break;

        /* COMPUTE */
        // 获取当前节点
        struct ggml_tensor * node = cgraph->nodes[node_n];
        // 获取当前节点的任务数量
        const int n_tasks = ggml_get_n_tasks(node, n_threads);

        // 设置计算参数
        struct ggml_compute_params params = {
            /*.type  =*/ GGML_TASK_COMPUTE,
            /*.ith   =*/ state->ith,
            /*.nth   =*/ n_tasks,
            /*.wsize =*/ cplan->work_size,
            /*.wdata =*/ cplan->work_data,
            /*.aic   =*/ &state->shared->aic,
        };

        // 如果当前线程编号小于任务数量，则进行前向计算
        if (state->ith < n_tasks) {
            ggml_compute_forward(&params, node);
        }
    }

    return GGML_EXIT_SUCCESS;

}


static thread_ret_t ggml_graph_compute_thread_hybrid(void * data) {
    // 获取计算状态
    struct ggml_compute_state * state = (struct ggml_compute_state *) data;

    // 获取计算图和计划
    const struct ggml_cgraph * cgraph = state->shared->cgraph;
    const struct ggml_cplan  * cplan  = state->shared->cplan;

    // 获取线程数量
    const int   n_threads   = state->shared->n_threads;

    // 设置NUMA线程亲和性
    set_numa_thread_affinity(state->ith, n_threads);

    // 使用调度器让出CPU时间片
    sched_yield();
#endif

                // 从共享状态中原子加载节点数量
                node_n = atomic_load(&state->shared->node_n);
                // 如果节点数量不等于上一次的数量，则跳出循环
                if (node_n != last) break;
            };
        }

        // 检查是否应该停止
        if (node_n >= cgraph->n_nodes) break;

        /* COMPUTE */
        // 获取当前节点
        struct ggml_tensor * node = cgraph->nodes[node_n];
        // 获取当前节点的任务数量
        const int n_tasks = ggml_get_n_tasks(node, n_threads);

        // 设置计算参数
        struct ggml_compute_params params = {
            /*.type  =*/ GGML_TASK_COMPUTE,
            /*.ith   =*/ state->ith-1,
            /*.nth   =*/ n_tasks-1,
            /*.wsize =*/ cplan->work_size,
            /*.wdata =*/ cplan->work_data,
            /*.aic   =*/ &state->shared->aic,
        };

        // 如果当前线程的索引小于任务数量，则进行前向计算
        if (state->ith < n_tasks) {
            ggml_compute_forward(&params, node);
        }
    }

    // 返回成功退出状态
    return GGML_EXIT_SUCCESS;
}

// 为图计划分配线程
struct ggml_cplan ggml_graph_plan(struct ggml_cgraph * cgraph, int n_threads) {
    // 如果线程数量小于等于0，则使用默认线程数量
    if (n_threads <= 0) {
        n_threads = GGML_DEFAULT_N_THREADS;
    }

    // 初始化工作大小
    size_t work_size = 0;

    // 初始化图计划
    struct ggml_cplan cplan;
    memset(&cplan, 0, sizeof(struct ggml_cplan));

    // 为不同操作进行线程调度 + 估算工作缓冲区大小
    // 遍历计算图中的节点
    for (int i = 0; i < cgraph->n_nodes; i++) {
        // 初始化任务数为1
        int n_tasks = 1;

        // 获取当前节点
        struct ggml_tensor * node = cgraph->nodes[i];

        // 初始化当前节点数据的偏移量
        size_t cur = 0;

        // 根据节点操作类型进行不同的处理
        switch (node->op) {
            // 如果是复制或者复制并增加操作
            case GGML_OP_CPY:
            case GGML_OP_DUP:
                {
                    // 任务数为线程数
                    n_tasks = n_threads;

                    // 如果节点类型是量化的，则计算当前节点数据的偏移量
                    if (ggml_is_quantized(node->type)) {
                        cur = ggml_type_size(GGML_TYPE_F32) * node->ne[0] * n_tasks;
                    }
                } break;
            // 如果是加法或者加法并增加1操作
            case GGML_OP_ADD:
            case GGML_OP_ADD1:
                {
                    // 任务数为线程数
                    n_tasks = n_threads;

                    // 如果源节点类型是量化的，则计算当前节点数据的偏移量
                    if (ggml_is_quantized(node->src[0]->type)) {
                        cur = ggml_type_size(GGML_TYPE_F32) * node->src[0]->ne[0] * n_tasks;
                    }
                } break;
            // 如果是累加操作
            case GGML_OP_ACC:
                {
                    // 任务数为线程数
                    n_tasks = n_threads;

                    // 如果源节点类型是量化的，则计算当前节点数据的偏移量
                    if (ggml_is_quantized(node->src[0]->type)) {
                        cur = ggml_type_size(GGML_TYPE_F32) * node->src[1]->ne[0] * n_tasks;
                    }
                } break;
            // 如果是矩阵乘法操作
            case GGML_OP_MUL_MAT:
                {
                    // 获取源节点类型对应的向量点乘类型
                    const enum ggml_type vec_dot_type = type_traits[node->src[0]->type].vec_dot_type;
#if defined(GGML_USE_CLBLAST)
                    // 如果定义了 GGML_USE_CLBLAST，并且可以使用 CLBLAST 进行矩阵乘法计算
                    if (ggml_cl_can_mul_mat(node->src[0], node->src[1], node)) {
                        // 获取使用 CLBLAST 进行矩阵乘法计算所需的工作大小
                        cur = ggml_cl_mul_mat_get_wsize(node->src[0], node->src[1], node);
                    } else
#endif
#if defined(GGML_USE_ACCELERATE) || defined(GGML_USE_OPENBLAS)
                    // 如果定义了 GGML_USE_ACCELERATE 或 GGML_USE_OPENBLAS，并且可以使用 BLAS 进行矩阵乘法计算
                    if (ggml_compute_forward_mul_mat_use_blas(node->src[0], node->src[1], node)) {
                        // 如果输入矩阵的数据类型不是 GGML_TYPE_F32
                        if (node->src[0]->type != GGML_TYPE_F32) {
                            // 在这里我们需要为来自 src0 的单个 2D 矩阵分配内存
                            cur = ggml_type_size(GGML_TYPE_F32)*(node->src[0]->ne[0]*node->src[0]->ne[1]);
                        }
                    } else
    }

    // 如果工作大小大于 0
    if (work_size > 0) {
        // 将工作大小增加到 CACHE_LINE_SIZE*(n_threads - 1)
        work_size += CACHE_LINE_SIZE*(n_threads - 1);
    }

    // 设置 cplan 结构体的 n_threads 字段
    cplan.n_threads = n_threads;
    // 设置 cplan 结构体的 work_size 字段
    cplan.work_size = work_size;
    // 设置 cplan 结构体的 work_data 字段为 NULL
    cplan.work_data = NULL;

    // 返回 cplan 结构体
    return cplan;
}

// 计算图的计算函数
int ggml_graph_compute(struct ggml_cgraph * cgraph, struct ggml_cplan * cplan) {
    {
        // 断言 cplan 不为空
        GGML_ASSERT(cplan);
        // 断言 cplan 的 n_threads 大于 0
        GGML_ASSERT(cplan->n_threads > 0);

        // 如果工作大小大于 0
        if (cplan->work_size > 0) {
            // 断言 cplan 的 work_data 不为空
            GGML_ASSERT(cplan->work_data);
        }
    }

    // 获取 n_threads 的值
    const int n_threads = cplan->n_threads;
#ifdef GGML_USE_CUBLAS
    // 定义 ggml_compute_state_shared 结构体并初始化字段数值
    struct ggml_compute_state_shared state_shared = {
        /*.cgraph                  =*/ cgraph,
        /*.cgraph_plan             =*/ cplan,
        /*.perf_node_start_cycles  =*/ 0,
        /*.perf_node_start_time_us =*/ 0,
        /*.n_threads               =*/ n_threads-1,
        /*.aic                     =*/ 0,
        /*.n_active                =*/ n_threads-1,
        /*.node_n                  =*/ -1,
        /*.abort_callback          =*/ NULL,
        /*.abort_callback_data     =*/ NULL,
    };
#else
    # 定义一个结构体变量 state_shared，包含以下字段
    struct ggml_compute_state_shared state_shared = {
        # cgraph 字段，指向 cgraph 变量
        /*.cgraph                  =*/ cgraph,
        # cgraph_plan 字段，指向 cplan 变量
        /*.cgraph_plan             =*/ cplan,
        # perf_node_start_cycles 字段，初始化为 0
        /*.perf_node_start_cycles  =*/ 0,
        # perf_node_start_time_us 字段，初始化为 0
        /*.perf_node_start_time_us =*/ 0,
        # n_threads 字段，指定线程数
        /*.n_threads               =*/ n_threads,
        # aic 字段，初始化为 0
        /*.aic                     =*/ 0,
        # n_active 字段，指定活跃线程数
        /*.n_active                =*/ n_threads,
        # node_n 字段，初始化为 -1
        /*.node_n                  =*/ -1,
        # abort_callback 字段，初始化为 NULL
        /*.abort_callback          =*/ NULL,
        # abort_callback_data 字段，初始化为 NULL
        /*.abort_callback_data     =*/ NULL,
    };
#endif
    // 为每个线程分配内存空间
    struct ggml_compute_state * workers = alloca(sizeof(struct ggml_compute_state)*n_threads);

    // 创建线程池
    if (n_threads > 1) {
        // 为每个线程初始化状态
        for (int j = 1; j < n_threads; ++j) {
            workers[j] = (struct ggml_compute_state) {
                .thrd   = 0,
                .ith = j,
                .shared = &state_shared,
            };
#ifdef GGML_USE_CUBLAS
            // 使用不同的线程函数创建线程
            const int rc = ggml_thread_create(&workers[j].thrd, NULL, ggml_graph_compute_thread_hybrid, &workers[j]);
#else
            const int rc = ggml_thread_create(&workers[j].thrd, NULL, ggml_graph_compute_thread, &workers[j]);
#endif
            // 确保线程创建成功
            GGML_ASSERT(rc == 0);
            UNUSED(rc);
        }
    }

    // 初始化主线程状态
    workers[0].ith = 0;
    workers[0].shared = &state_shared;

    // 记录性能起始时间
    const int64_t perf_start_cycles  = ggml_perf_cycles();
    const int64_t perf_start_time_us = ggml_perf_time_us();

    // 运行工作线程
#ifdef GGML_USE_CUBLAS
    int compute_status = (size_t) ggml_graph_compute_thread_hybrid(&workers[0]);
#else
    int compute_status = (size_t) ggml_graph_compute_thread(&workers[0]);
#endif

    // 清除主线程的亲和性设置
    clear_numa_thread_affinity();

    // 等待或终止线程池中的线程
    if (n_threads > 1) {
        for (int j = 1; j < n_threads; j++) {
            const int rc = ggml_thread_join(workers[j].thrd, NULL);
            // 确保线程成功结束
            GGML_ASSERT(rc == 0);
        }
    }

    // 性能统计（图形）
    {
        // 计算当前性能周期数
        int64_t perf_cycles_cur  = ggml_perf_cycles()  - perf_start_cycles;
        // 计算当前性能时间（微秒）
        int64_t perf_time_us_cur = ggml_perf_time_us() - perf_start_time_us;

        // 性能运行次数加一
        cgraph->perf_runs++;
        // 性能周期数累加
        cgraph->perf_cycles  += perf_cycles_cur;
        // 性能时间累加
        cgraph->perf_time_us += perf_time_us_cur;

        // 打印调试信息，包括函数名、性能运行次数、CPU时间和墙钟时间
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

// 使用上下文和计算图计算结果
void ggml_graph_compute_with_ctx(struct ggml_context * ctx, struct ggml_cgraph * cgraph, int n_threads) {
    // 根据计算图和线程数创建计划
    struct ggml_cplan cplan = ggml_graph_plan(cgraph, n_threads);

    // 在上下文中创建新的对象
    struct ggml_object * obj = ggml_new_object(ctx, GGML_OBJECT_WORK_BUFFER, cplan.work_size);

    // 设置计划的工作数据
    cplan.work_data = (uint8_t *)ctx->mem_buffer + obj->offs;

    // 执行计算图的计算
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

    // 遍历普通节点
    for (int i = 0; i < cgraph->n_nodes; i++) {
        struct ggml_tensor * node = cgraph->nodes[i];

        // 如果找到指定名称的普通节点，则返回该节点
        if (strcmp(node->name, name) == 0) {
            return node;
        }
    }

    // 如果未找到指定名称的节点，则返回空指针
    return NULL;
}

// 导出叶子节点的信息到文件
static void ggml_graph_export_leaf(const struct ggml_tensor * tensor, FILE * fout) {
    const int64_t * ne = tensor->ne;
    const size_t  * nb = tensor->nb;

    // 将叶子节点的信息格式化输出到文件
    fprintf(fout, "%-6s %-12s %8d %" PRId64 " %" PRId64 " %" PRId64 " %" PRId64 " %16zu %16zu %16zu %16zu %16p %32s\n",
            ggml_type_name(tensor->type),
            ggml_op_name  (tensor->op),
            tensor->n_dims,
            ne[0], ne[1], ne[2], ne[3],
            nb[0], nb[1], nb[2], nb[3],
            tensor->data,
            tensor->name);
}

// 导出普通节点的信息到文件
static void ggml_graph_export_node(const struct ggml_tensor * tensor, const char * arg, FILE * fout) {
    const int64_t * ne = tensor->ne;
    const size_t  * nb = tensor->nb;

    // 将普通节点的信息格式化输出到文件
    fprintf(fout, "%-6s %-6s %-12s %8d %" PRId64 " %" PRId64 " %" PRId64 " %" PRId64 " %16zu %16zu %16zu %16zu %16p %32s\n",
            arg,
            ggml_type_name(tensor->type),
            ggml_op_name  (tensor->op),
            tensor->n_dims,
            ne[0], ne[1], ne[2], ne[3],
            nb[0], nb[1], nb[2], nb[3],
            tensor->data,
            tensor->name);
}
void ggml_graph_export(const struct ggml_cgraph * cgraph, const char * fname) {
    uint64_t size_eval = 0;  // 初始化中间结果的大小为0

    // 计算中间结果的大小
    // TODO: does not take into account scratch buffers !!!!
    for (int i = 0; i < cgraph->n_nodes; ++i) {  // 遍历节点
        size_eval += ggml_nbytes_pad(cgraph->nodes[i]);  // 计算节点的大小并累加到中间结果的大小
    }

    // 打印
    {
        FILE * fout = stdout;  // 初始化输出文件为标准输出

        fprintf(fout, "\n");  // 输出空行
        fprintf(fout, "%-16s %8x\n", "magic",        GGML_FILE_MAGIC);  // 输出魔数
        fprintf(fout, "%-16s %8d\n", "version",      GGML_FILE_VERSION);  // 输出版本号
        fprintf(fout, "%-16s %8d\n", "leafs",        cgraph->n_leafs);  // 输出叶子节点数
        fprintf(fout, "%-16s %8d\n", "nodes",        cgraph->n_nodes);  // 输出节点数
        fprintf(fout, "%-16s %" PRIu64 "\n", "eval", size_eval);  // 输出中间结果的大小

        // 头部
        fprintf(fout, "\n");
        fprintf(fout, "%-6s %-12s %8s %8s %8s %8s %8s %16s %16s %16s %16s %16s %16s\n",
                "TYPE", "OP", "NDIMS", "NE0", "NE1", "NE2", "NE3", "NB0", "NB1", "NB2", "NB3", "DATA", "NAME");

        for (int i = 0; i < cgraph->n_leafs; ++i) {  // 遍历叶子节点
            ggml_graph_export_leaf(cgraph->leafs[i], fout);  // 导出叶子节点

            GGML_ASSERT(cgraph->leafs[i]->op   == GGML_OP_NONE);  // 断言叶子节点的操作为NONE
            GGML_ASSERT(cgraph->leafs[i]->src[0] == NULL);  // 断言叶子节点的第一个源节点为空
            GGML_ASSERT(cgraph->leafs[i]->src[1] == NULL);  // 断言叶子节点的第二个源节点为空
        }

        // 头部
        fprintf(fout, "\n");
        fprintf(fout, "%-6s %-6s %-12s %8s %8s %8s %8s %8s %16s %16s %16s %16s %8s %16s %16s\n",
                "ARG", "TYPE", "OP", "NDIMS", "NE0", "NE1", "NE2", "NE3", "NB0", "NB1", "NB2", "NB3", "NTASKS", "DATA", "NAME");

        for (int i = 0; i < cgraph->n_nodes; ++i) {  // 遍历节点
            ggml_graph_export_node(cgraph->nodes[i], "DST", fout);  // 导出节点

            for (int j = 0; j < GGML_MAX_SRC; ++j) {  // 遍历节点的源节点
                if (cgraph->nodes[i]->src[j]) {  // 如果源节点存在
                    ggml_graph_export_node(cgraph->nodes[i]->src[j], "SRC", fout);  // 导出源节点
                }
            }

            fprintf(fout, "\n");  // 输出空行
        }

        fprintf(fout, "\n");  // 输出空行
    }
}
    // 结束当前的代码块
    }

    // 写入二进制数据
    }
}

// 从文件导入图形数据
struct ggml_cgraph * ggml_graph_import(const char * fname, struct ggml_context ** ctx_data, struct ggml_context ** ctx_eval) {
    // 断言数据上下文和评估上下文为空
    assert(*ctx_data == NULL);
    assert(*ctx_eval == NULL);

    // 初始化结果为 NULL
    struct ggml_cgraph * result = NULL;

    // 初始化数据张量为 NULL
    struct ggml_tensor * data = NULL;

    // 读取文件内容到数据中
    {
        // 打开文件
        FILE * fin = fopen(fname, "rb");
        // 如果文件打开失败，输出错误信息并返回结果为 NULL
        if (!fin) {
            fprintf(stderr, "%s: failed to open %s\n", __func__, fname);
            return result;
        }

        // 获取文件大小
        size_t fsize = 0;

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

            // 初始化数据上下文
            *ctx_data = ggml_init(params);

            // 如果数据上下文初始化失败，输出错误信息，关闭文件并返回结果为 NULL
            if (!*ctx_data) {
                fprintf(stderr, "%s: failed to create ggml context\n", __func__);
                fclose(fin);
                return result;
            }
        }

        // 创建一维张量
        data = ggml_new_tensor_1d(*ctx_data, GGML_TYPE_I8, fsize);

        // 从文件中读取数据到张量中
        {
            const size_t ret = fread(data->data, sizeof(char), fsize, fin);
            // 如果读取的数据大小不等于文件大小，输出错误信息，关闭文件并返回结果为 NULL
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

// 打印图形数据
void ggml_graph_print(const struct ggml_cgraph * cgraph) {
    // 初始化每个操作的总性能时间为 0
    int64_t perf_total_per_op_us[GGML_OP_COUNT] = {0};

    // 打印图形信息
    GGML_PRINT("=== GRAPH ===\n");

    GGML_PRINT("n_nodes = %d\n", cgraph->n_nodes);
    # 遍历计算图中的节点
    for (int i = 0; i < cgraph->n_nodes; i++) {
        # 获取当前节点
        struct ggml_tensor * node = cgraph->nodes[i];

        # 更新每个操作的总执行时间
        perf_total_per_op_us[node->op] += MAX(1, node->perf_time_us);

        # 打印节点的性能信息
        GGML_PRINT(" - %3d: [ %5" PRId64 ", %5" PRId64 ", %5" PRId64 "] %16s %s (%3d) cpu = %7.3f / %7.3f ms, wall = %7.3f / %7.3f ms\n",
                i,
                node->ne[0], node->ne[1], node->ne[2],
                ggml_op_name(node->op), node->is_param ? "x" : node->grad ? "g" : " ", node->perf_runs,
                (double) node->perf_cycles  / (double) ggml_cycles_per_ms(),
                (double) node->perf_cycles  / (double) ggml_cycles_per_ms() / (double) node->perf_runs,
                (double) node->perf_time_us / 1000.0,
                (double) node->perf_time_us / 1000.0 / node->perf_runs);
    }

    # 打印叶子节点的信息
    GGML_PRINT("n_leafs = %d\n", cgraph->n_leafs);
    for (int i = 0; i < cgraph->n_leafs; i++) {
        # 获取当前叶子节点
        struct ggml_tensor * node = cgraph->leafs[i];

        # 打印叶子节点的信息
        GGML_PRINT(" - %3d: [ %5" PRId64 ", %5" PRId64 "] %8s %16s\n",
                i,
                node->ne[0], node->ne[1],
                ggml_op_name(node->op),
                ggml_get_name(node));
    }

    # 打印每个操作的总执行时间
    for (int i = 0; i < GGML_OP_COUNT; i++) {
        if (perf_total_per_op_us[i] == 0) {
            continue;
        }

        GGML_PRINT("perf_total_per_op_us[%16s] = %7.3f ms\n", ggml_op_name(i), (double) perf_total_per_op_us[i] / 1000.0);
    }

    # 打印分隔线
    GGML_PRINT("========================================\n");
// 检查节点是否是图中的一部分
static bool ggml_graph_find(const struct ggml_cgraph * cgraph, const struct ggml_tensor * node) {
    // 如果图为空，则返回 true
    if (cgraph == NULL) {
        return true;
    }

    // 遍历图中的节点，如果找到与给定节点相同的节点，则返回 true
    for (int i = 0; i < cgraph->n_nodes; i++) {
        if (cgraph->nodes[i] == node) {
            return true;
        }
    }

    // 如果未找到相同的节点，则返回 false
    return false;
}

// 获取给定节点的父节点
static struct ggml_tensor * ggml_graph_get_parent(const struct ggml_cgraph * cgraph, const struct ggml_tensor * node) {
    // 遍历图中的节点，如果找到具有给定节点作为梯度的节点，则返回该节点
    for (int i = 0; i < cgraph->n_nodes; i++) {
        struct ggml_tensor * parent = cgraph->nodes[i];

        if (parent->grad == node) {
            return parent;
        }
    }

    // 如果未找到具有给定节点作为梯度的节点，则返回 NULL
    return NULL;
}

// 将节点和父节点的关系以及标签信息输出到 DOT 文件中
static void ggml_graph_dump_dot_node_edge(FILE * fp, const struct ggml_cgraph * gb, struct ggml_tensor * node, struct ggml_tensor * parent, const char * label)  {
    // 获取节点的父节点
    struct ggml_tensor * gparent = ggml_graph_get_parent(gb, node);
    struct ggml_tensor * gparent0 = ggml_graph_get_parent(gb, parent);
    // 将节点和父节点的关系以及标签信息输出到 DOT 文件中
    fprintf(fp, "  \"%p\":%s -> \"%p\":%s [ arrowhead = %s; style = %s; label = \"%s\"; ]\n",
            gparent0 ? (void *) gparent0 : (void *) parent,
            gparent0 ? "g" : "x",
            gparent ? (void *) gparent : (void *) node,
            gparent ? "g" : "x",
            gparent ? "empty" : "vee",
            gparent ? "dashed" : "solid",
            label);
}

// 将叶子节点和父节点的关系以及标签信息输出到 DOT 文件中
static void ggml_graph_dump_dot_leaf_edge(FILE * fp, struct ggml_tensor * node, struct ggml_tensor * parent, const char * label)  {
    // 将叶子节点和父节点的关系以及标签信息输出到 DOT 文件中
    fprintf(fp, "  \"%p\":%s -> \"%p\":%s [ label = \"%s\"; ]\n",
            (void *) parent, "x",
            (void *) node, "x",
            label);
}

// 将计算图的结构以及关系输出到 DOT 文件中
void ggml_graph_dump_dot(const struct ggml_cgraph * gb, const struct ggml_cgraph * gf, const char * filename) {
    char color[16];

    // 打开文件以供写入
    FILE * fp = fopen(filename, "w");
    GGML_ASSERT(fp);

    // 输出 DOT 文件的头部信息
    fprintf(fp, "digraph G {\n");
    fprintf(fp, "  newrank = true;\n");
    fprintf(fp, "  rankdir = LR;\n");
    // 遍历节点数组
    for (int i = 0; i < gb->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = gb->nodes[i];

        // 如果当前节点有父节点，则跳过
        if (ggml_graph_get_parent(gb, node) != NULL) {
            continue;
        }

        // 如果当前节点是参数节点，则设置颜色为黄色
        if (node->is_param) {
            snprintf(color, sizeof(color), "yellow");
        } 
        // 如果当前节点有梯度
        else if (node->grad) {
            // 如果在图中找到当前节点，则设置颜色为绿色
            if (ggml_graph_find(gf, node)) {
                snprintf(color, sizeof(color), "green");
            } 
            // 否则设置颜色为浅蓝色
            else {
                snprintf(color, sizeof(color), "lightblue");
            }
        } 
        // 如果当前节点既不是参数节点也没有梯度，则设置颜色为白色
        else {
            snprintf(color, sizeof(color), "white");
        }

        // 输出节点信息到文件
        fprintf(fp, "  \"%p\" [ "
                    "style = filled; fillcolor = %s; shape = record; "
                    "label=\"",
                (void *) node, color);

        // 如果节点名称长度大于0，则输出节点名称和类型
        if (strlen(node->name) > 0) {
            fprintf(fp, "%s (%s)|", node->name, ggml_type_name(node->type));
        } 
        // 否则只输出节点类型
        else {
            fprintf(fp, "(%s)|", ggml_type_name(node->type));
        }

        // 如果节点维度为2，则输出节点索引和维度信息
        if (node->n_dims == 2) {
            fprintf(fp, "%d [%" PRId64 ", %" PRId64 "] | <x>%s", i, node->ne[0], node->ne[1], ggml_op_symbol(node->op));
        } 
        // 否则输出节点索引和维度信息
        else {
            fprintf(fp, "%d [%" PRId64 ", %" PRId64 ", %" PRId64 "] | <x>%s", i, node->ne[0], node->ne[1], node->ne[2], ggml_op_symbol(node->op));
        }

        // 如果节点有梯度，则输出梯度信息
        if (node->grad) {
            fprintf(fp, " | <g>%s\"; ]\n", ggml_op_symbol(node->grad->op));
        } 
        // 否则结束节点信息输出
        else {
            fprintf(fp, "\"; ]\n");
        }
    }
    # 遍历叶子节点数组
    for (int i = 0; i < gb->n_leafs; i++) {
        # 获取当前叶子节点
        struct ggml_tensor * node = gb->leafs[i];

        # 格式化颜色字符串为 "pink"
        snprintf(color, sizeof(color), "pink");

        # 将节点信息输出到文件中
        fprintf(fp, "  \"%p\" [ "
                    "style = filled; fillcolor = %s; shape = record; "
                    "label=\"<x>",
                (void *) node, color);

        # 如果节点名称长度大于0，则输出节点名称和类型；否则只输出节点类型
        if (strlen(node->name) > 0) {
            fprintf(fp, "%s (%s)|", node->name, ggml_type_name(node->type));
        } else {
            fprintf(fp, "(%s)|", ggml_type_name(node->type));
        }

        # 输出节点的常量值和索引
        fprintf(fp, "CONST %d [%" PRId64 ", %" PRId64 "]", i, node->ne[0], node->ne[1]);
        
        # 如果节点元素个数小于5，则输出节点的元素值
        if (ggml_nelements(node) < 5) {
            fprintf(fp, " | (");
            for (int j = 0; j < ggml_nelements(node); j++) {
                # 根据节点类型输出不同类型的元素值
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
        # 输出节点信息结束
        fprintf(fp, "\"; ]\n");
    }

    # 遍历节点数组
    for (int i = 0; i < gb->n_nodes; i++) {
        # 获取当前节点
        struct ggml_tensor * node = gb->nodes[i];

        # 遍历节点的源
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            # 如果节点的源存在，则输出节点和源之间的边
            if (node->src[j]) {
                char label[16];
                snprintf(label, sizeof(label), "src %d", j);
                ggml_graph_dump_dot_node_edge(fp, gb, node, node->src[j], label);
            }
        }
    }
    # 遍历叶子节点数组
    for (int i = 0; i < gb->n_leafs; i++) {
        # 获取当前叶子节点
        struct ggml_tensor * node = gb->leafs[i];

        # 遍历当前节点的源节点数组
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            # 如果当前源节点存在
            if (node->src[j]) {
                # 创建标签字符串
                char label[16];
                snprintf(label, sizeof(label), "src %d", j);
                # 将当前节点和源节点以及标签信息写入到 DOT 文件中
                ggml_graph_dump_dot_leaf_edge(fp, node, node->src[j], label);
            }
        }
    }

    # 写入 DOT 文件结束标识
    fprintf(fp, "}\n");

    # 关闭文件指针
    fclose(fp);

    # 打印命令行操作提示信息
    GGML_PRINT("%s: dot -Tpng %s -o %s.png && open %s.png\n", __func__, filename, filename, filename);
// 设置参数的优化器函数，将参数数组中的值设置到对应的张量中
static void ggml_opt_set_params(int np, struct ggml_tensor * const ps[], const float * x) {
    int i = 0;  // 初始化索引 i 为 0
    for (int p = 0; p < np; ++p) {  // 遍历参数数组
        const int64_t ne = ggml_nelements(ps[p]) ;  // 获取张量中元素的数量
        // TODO: add function to set tensor from array
        for (int64_t j = 0; j < ne; ++j) {  // 遍历张量中的元素
            ggml_set_f32_1d(ps[p], j, x[i++]);  // 将参数数组中的值设置到对应的张量中
        }
    }
}

// 获取参数的优化器函数，将张量中的值获取到参数数组中
static void ggml_opt_get_params(int np, struct ggml_tensor * const ps[], float * x) {
    int i = 0;  // 初始化索引 i 为 0
    for (int p = 0; p < np; ++p) {  // 遍历参数数组
        const int64_t ne = ggml_nelements(ps[p]) ;  // 获取张量中元素的数量
        // TODO: add function to get all elements at once
        for (int64_t j = 0; j < ne; ++j) {  // 遍历张量中的元素
            x[i++] = ggml_get_f32_1d(ps[p], j);  // 将张量中的值获取到参数数组中
        }
    }
}

// 获取梯度的优化器函数，将张量中的梯度值获取到梯度数组中
static void ggml_opt_get_grad(int np, struct ggml_tensor * const ps[], float * g) {
    int64_t i = 0;  // 初始化索引 i 为 0
    for (int p = 0; p < np; ++p) {  // 遍历参数数组
        const int64_t ne = ggml_nelements(ps[p]) ;  // 获取张量中元素的数量
        // TODO: add function to get all elements at once
        for (int64_t j = 0; j < ne; ++j) {  // 遍历张量中的元素
            g[i++] = ggml_get_f32_1d(ps[p]->grad, j);  // 将张量中的梯度值获取到梯度数组中
        }
    }
}

// 累积梯度的优化器函数，将张量中的梯度值乘以比例后累积到梯度数组中
static void ggml_opt_acc_grad(int np, struct ggml_tensor * const ps[], float * g, float scale) {
    int64_t i = 0;  // 初始化索引 i 为 0
    for (int p = 0; p < np; ++p) {  // 遍历参数数组
        const int64_t ne = ggml_nelements(ps[p]) ;  // 获取张量中元素的数量
        // TODO: add function to get all elements at once
        for (int64_t j = 0; j < ne; ++j) {  // 遍历张量中的元素
            g[i++] += ggml_get_f32_1d(ps[p]->grad, j) * scale;  // 将张量中的梯度值乘以比例后累积到梯度数组中
        }
    }
}

//
// ADAM
//
//   ref: https://arxiv.org/pdf/1412.6980.pdf
//

// ADAM 优化器函数，根据给定的参数和梯度进行优化
static enum ggml_opt_result ggml_opt_adam(
        struct ggml_context * ctx,
        struct ggml_opt_context * opt,
        struct ggml_opt_params params,
        struct ggml_tensor * f,
        struct ggml_cgraph * gf,
        struct ggml_cgraph * gb,
        ggml_opt_callback callback,
        void * callback_data) {
    GGML_ASSERT(ggml_is_scalar(f));  // 断言张量 f 是标量
    // 定义一个指针数组，用于存储要优化的参数
    struct ggml_tensor * ps[GGML_MAX_PARAMS];

    // 初始化参数数量为0，初始化参数总数为0
    int np = 0;
    int64_t nx = 0;

    // 遍历计算图中的节点
    for (int i = 0; i < gf->n_nodes; ++i) {
        // 如果节点是参数节点
        if (gf->nodes[i]->is_param) {
            // 输出找到的参数信息
            GGML_PRINT_DEBUG("found param %d: grad->op = %d\n", np, gf->nodes[i]->grad->op);

            // 断言参数数量小于最大参数数量
            GGML_ASSERT(np < GGML_MAX_PARAMS);

            // 将参数节点存储到参数数组中，并增加参数总数
            ps[np++] = gf->nodes[i];
            nx += ggml_nelements(gf->nodes[i]);
        }
    }

    // 如果参数类型、参数总数或参数过去值发生变化
    if ((opt->params.type != params.type) || (opt->nx != nx) || (opt->params.past != params.past)) {
        // 保存当前迭代次数
        int iter = opt->iter;
        // 初始化优化器
        ggml_opt_init(opt->ctx, opt, params, nx);
        // 恢复迭代次数
        opt->iter = iter;
    }

    // 定义常量
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

    // 获取梯度、一阶矩和二阶矩的指针
    float * g  = opt->adam.g->data;  // gradients
    float * m  = opt->adam.m->data;  // first moment
    float * v  = opt->adam.v->data;  // second moment

    // 如果过去值大于0，则获取过去函数值的指针，否则为NULL
    float * pf = params.past > 0 ? opt->adam.pf->data : NULL; // past function values

    // 计划计算图的工作缓冲区和对象
    struct ggml_cplan cplan = ggml_graph_plan(gb, params.n_threads);
    struct ggml_object * obj = ggml_new_object(ctx, GGML_OBJECT_WORK_BUFFER, cplan.work_size);
    cplan.work_data = (uint8_t *)ctx->mem_buffer + obj->offs;

    // 取消标志初始化为false
    bool cancel = false;

    // 计算函数值初始化为0
    float fx = 0;
    // 将梯度值初始化为0
    ggml_set_zero(opt->adam.g);
    // 循环累积步骤，从0到n_accum-1
    for (int accum_step = 0; accum_step < n_accum; ++accum_step) {
        // 如果存在回调函数
        if (callback) {
            // 调用回调函数，传入回调数据、累积步骤、调度指针和取消指针
            callback(callback_data, accum_step, &sched, &cancel);
            // 如果取消标志为真，则返回取消状态
            if (cancel) {
                return GGML_OPT_CANCEL;
            }
        }
        // 重置图形
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

    // 设置adam.fx_prev为fx
    opt->adam.fx_prev = fx;
    // 设置adam.fx_best为adam.fx_prev
    opt->adam.fx_best = opt->adam.fx_prev;
    // 如果pf不为空
    if (pf) {
        // 将opt->iter对params.past取模，将opt->adam.fx_prev存入pf数组
        pf[opt->iter % params.past] = opt->adam.fx_prev;
    }

    // 设置loss_before和loss_after为adam.fx_prev
    opt->loss_before = opt->adam.fx_prev;
    opt->loss_after  = opt->adam.fx_prev;

    // 如果刚初始化
    if (opt->just_initialized) {
        // 将n_no_improvement设置为0
        opt->adam.n_no_improvement = 0;
        // 将just_initialized设置为false
        opt->just_initialized = false;
    }

    // 设置fx_best指针指向adam.fx_best
    float * fx_best = &opt->adam.fx_best;
    // 设置fx_prev指针指向adam.fx_prev
    float * fx_prev = &opt->adam.fx_prev;
    // 设置n_no_improvement指针指向adam.n_no_improvement
    int * n_no_improvement = &opt->adam.n_no_improvement;

    // 将iter0设置为opt->iter
    int iter0 = opt->iter;

    // 运行优化器

    // 返回未收敛状态
    return GGML_OPT_DID_NOT_CONVERGE;
// 结构体，用于存储 L-BFGS 迭代数据
struct ggml_lbfgs_iteration_data {
    float alpha;  // 步长
    float ys;     // y 和 s 的内积
    float * s;    // 存储最近 m 步的 s 向量
    float * y;    // 存储最近 m 步的 y 向量
};

// 使用回溯线搜索进行优化
static enum ggml_opt_result linesearch_backtracking(
        const struct ggml_opt_params * params,  // 优化参数
        int nx,                                 // 变量 x 的维度
        float * x,                              // 变量 x
        float * fx,                             // 目标函数值
        float * g,                              // 梯度
        float * d,                              // 搜索方向
        float * step,                           // 步长
        const float * xp,                       // 先前的变量 x
        struct ggml_tensor * f,                 // 目标函数
        struct ggml_cgraph * gb,                // 反向传播计算图
        struct ggml_cplan  * cplan,             // 计算计划
        const int np,                           // 参数数量
        struct ggml_tensor * ps[],              // 参数数组
        bool * cancel,                          // 取消标志
        ggml_opt_callback callback,             // 优化回调函数
        void * callback_data) {                 // 回调函数数据
    int count = 0;

    float width  = 0.0f;  // 步长范围
    float dg     = 0.0f;  // 梯度和搜索方向的内积
    float finit  = 0.0f;  // 初始目标函数值
    float dginit = 0.0f;  // 初始梯度和搜索方向的内积
    float dgtest = 0.0f;  // 用于线搜索的测试值

    const float dec = 0.5f;  // 减小步长的因子
    const float inc = 2.1f;  // 增加步长的因子

    const int n_accum = MAX(1, params->n_gradient_accumulation);  // 梯度累积的最大次数
    const float accum_norm = 1.0f / (float) n_accum;  // 梯度累积的归一化系数

    if (*step <= 0.f) {
        return GGML_LINESEARCH_INVALID_PARAMETERS;  // 步长无效
    }

    // 计算搜索方向上的初始梯度
    ggml_vec_dot_f32(nx, &dginit, g, d);

    // 确保搜索方向是下降方向
    if (0 < dginit) {
        return GGML_LINESEARCH_FAIL;  // 线搜索失败
    }

    // 初始化局部变量
    finit = *fx;      // 记录初始目标函数值
    dgtest = params->lbfgs.ftol*dginit;  // 计算用于线搜索的测试值
}

// L-BFGS 优化算法
static enum ggml_opt_result ggml_opt_lbfgs(
        struct ggml_context * ctx,              // 上下文
        struct ggml_opt_context * opt,          // 优化上下文
        struct ggml_opt_params params,          // 优化参数
        struct ggml_tensor * f,                 // 目标函数
        struct ggml_cgraph * gf,                // 前向传播计算图
        struct ggml_cgraph * gb,                // 反向传播计算图
        ggml_opt_callback callback,             // 优化回调函数
        void * callback_data) {                 // 回调函数数据
    // 如果使用的是 Backtracking Line Search，并且 Wolfe 参数小于等于 ftol 或大于等于 1，则返回无效的 Wolfe 参数
    if (params.lbfgs.linesearch == GGML_LINESEARCH_BACKTRACKING_WOLFE ||
        params.lbfgs.linesearch == GGML_LINESEARCH_BACKTRACKING_STRONG_WOLFE) {
        if (params.lbfgs.wolfe <= params.lbfgs.ftol || 1.f <= params.lbfgs.wolfe) {
            return GGML_OPT_INVALID_WOLFE;
        }
    }

    // 获取 L-BFGS 算法的参数 m
    const int m = params.lbfgs.m;

    // 创建一个存储要优化的参数的数组
    struct ggml_tensor * ps[GGML_MAX_PARAMS];

    // 初始化参数计数器和参数总数
    int np = 0;
    int nx = 0;

    // 遍历计算图的节点，找到需要优化的参数
    for (int i = 0; i < gf->n_nodes; ++i) {
        if (gf->nodes[i]->is_param) {
            GGML_PRINT_DEBUG("found param %d: grad->op = %d\n", np, gf->nodes[i]->grad->op);

            // 断言参数计数器小于最大参数数量
            GGML_ASSERT(np < GGML_MAX_PARAMS);

            // 将找到的参数存储到数组中，并更新参数总数
            ps[np++] = gf->nodes[i];
            nx += ggml_nelements(gf->nodes[i]);
        }
    }

    // 如果优化器的类型、参数总数、过去的参数等不同于当前参数，则重新初始化优化器
    if ((opt->params.type != params.type) || (opt->nx != nx) || (opt->params.past != params.past) || (opt->params.lbfgs.m != params.lbfgs.m)) {
        int iter = opt->iter;
        ggml_opt_init(ctx, opt, params, nx);
        opt->iter = iter;
    }

    // 创建计划以执行计算图
    struct ggml_cplan cplan = ggml_graph_plan(gb, params.n_threads);
    // 创建一个新的对象用于存储工作缓冲区
    struct ggml_object * obj = ggml_new_object(ctx, GGML_OBJECT_WORK_BUFFER, cplan.work_size);
    // 将工作数据指针指向内存缓冲区的偏移位置
    cplan.work_data = (uint8_t *)ctx->mem_buffer + obj->offs;

    // 获取当前参数、上一次参数、当前梯度、上一次梯度、搜索方向的指针
    float * x  = opt->lbfgs.x->data;  // current parameters
    float * xp = opt->lbfgs.xp->data; // previous parameters
    float * g  = opt->lbfgs.g->data;  // current gradient
    float * gp = opt->lbfgs.gp->data; // previous gradient
    float * d  = opt->lbfgs.d->data;  // search direction

    // 如果过去的函数值大于 0，则获取过去的函数值的指针
    float * pf = params.past > 0 ? opt->lbfgs.pf->data : NULL; // past function values

    // 计算梯度累积的数量和累积的归一化值
    const int n_accum = MAX(1, params.n_gradient_accumulation);
    const float accum_norm = 1.0f / (float) n_accum;

    // 初始化成本函数值、参数范数和梯度范数
    float fx    = 0.0f; // cost function value
    float xnorm = 0.0f; // ||x||
    float gnorm = 0.0f; // ||g||

    // 从计算图节点中获取参数的初始值
    ggml_opt_get_params(np, ps, x);

    // L-BFGS 内存
    // 定义指向 lbfgs 结构体中 lmal 数据的指针
    float * lm_alpha = opt->lbfgs.lmal->data;
    // 定义指向 lbfgs 结构体中 lmys 数据的指针
    float * lm_ys    = opt->lbfgs.lmys->data;
    // 定义指向 lbfgs 结构体中 lms 数据的指针
    float * lm_s     = opt->lbfgs.lms->data;
    // 定义指向 lbfgs 结构体中 lmy 数据的指针
    float * lm_y     = opt->lbfgs.lmy->data;

    // 定义一个布尔变量 cancel，并初始化为 false
    bool cancel = false;

    // 评估函数值及其梯度
    {
        // 设置参数
        ggml_opt_set_params(np, ps, x);

        // 初始化函数值为 0，梯度为 0
        fx = 0;
        memset(g, 0, sizeof(float)*nx);
        // 循环累积步数
        for (int accum_step = 0; accum_step < n_accum; ++accum_step) {
            if (callback) {
                // LBFG-S 不支持学习率，忽略学习进度
                float sched = 0;
                // 调用回调函数
                callback(callback_data, accum_step, &sched, &cancel);
                // 如果 cancel 为 true，则返回 GGML_OPT_CANCEL
                if (cancel) {
                    return GGML_OPT_CANCEL;
                }
            }
            // 重置图
            // ggml_graph_reset  (gf);
            // 设置 f 的梯度为 1.0
            ggml_set_f32      (f->grad, 1.0f);
            // 计算图
            ggml_graph_compute(gb, &cplan);
            // 累积梯度
            ggml_opt_acc_grad(np, ps, g, accum_norm);
            // 累积函数值
            fx += ggml_get_f32_1d(f, 0);
        }
        // 乘以累积步长
        fx *= accum_norm;

        // 记录优化前后的损失值
        opt->loss_before = fx;
        opt->loss_after  = fx;
    }

    // 搜索方向为负梯度
    ggml_vec_neg_f32(nx, d, g);

    // 计算 ||x|| 和 ||g||
    ggml_vec_norm_f32(nx, &xnorm, x);
    ggml_vec_norm_f32(nx, &gnorm, g);

    // 如果 ||x|| 小于 1.0，则将其设置为 1.0
    if (xnorm < 1.0f) {
        xnorm = 1.0f;
    }

    // 如果梯度范数除以参数 lbfgs.eps 小于等于 1，则返回 GGML_OPT_OK
    if (gnorm/xnorm <= params.lbfgs.eps) {
        return GGML_OPT_OK;
    }

    // 如果刚初始化
    if (opt->just_initialized) {
        // 如果 pf 不为空，则将其第一个元素设置为 fx
        if (pf) {
            pf[0] = fx;
        }
        // 记录最佳函数值为 fx
        opt->lbfgs.fx_best = fx;

        // 初始化步长
        ggml_vec_norm_inv_f32(nx, &opt->lbfgs.step, d);
        // 初始化 j 和 k
        opt->lbfgs.j                = 0;
        opt->lbfgs.k                = 1;
        opt->lbfgs.end              = 0;
        opt->lbfgs.n_no_improvement = 0;
        opt->just_initialized       = false;
    }

    // 定义指向 lbfgs 结构体中 fx_best 数据的指针
    float * fx_best        = &opt->lbfgs.fx_best;
    // 定义指向 lbfgs 结构体中 step 数据的指针
    float * step           = &opt->lbfgs.step;
    // 定义指向 lbfgs 结构体中 j 数据的指针
    int * j                = &opt->lbfgs.j;
    # 定义指针变量 k，指向 opt->lbfgs.k 的地址
    int * k                = &opt->lbfgs.k;
    # 定义指针变量 end，指向 opt->lbfgs.end 的地址
    int * end              = &opt->lbfgs.end;
    # 定义指针变量 n_no_improvement，指向 opt->lbfgs.n_no_improvement 的地址
    int * n_no_improvement = &opt->lbfgs.n_no_improvement;

    # 初始化 ls 为 0
    int ls     = 0;
    # 初始化 bound 为 0
    int bound  = 0;

    # 初始化 ys 为 0.0
    float ys   = 0.0f;
    # 初始化 yy 为 0.0
    float yy   = 0.0f;
    # 初始化 beta 为 0.0
    float beta = 0.0f;

    # 初始化 it 为 0
    int it = 0;

    # 标记函数结束，表示不应该执行到这里
    }

    # 表示不应该执行到这里，如果执行到这里，会触发错误
    GGML_UNREACHABLE();
# 定义一个函数，返回默认的优化参数结构体，根据不同的优化类型
struct ggml_opt_params ggml_opt_default_params(enum ggml_opt_type type) {
    # 声明一个结果结构体
    struct ggml_opt_params result;

    # 返回结果结构体
    return result;
}

# 初始化优化器
GGML_API void ggml_opt_init(
        struct ggml_context * ctx,
        struct ggml_opt_context * opt,
        struct ggml_opt_params params,
        int64_t nx) {
    # 将上下文和参数赋值给优化器上下文
    opt->ctx = ctx;
    opt->params = params;
    opt->iter = 0;
    opt->nx = nx;
    opt->just_initialized = true;
    # 如果上下文为空
    if (opt->ctx == NULL) {
        # 初始化上下文参数
        struct ggml_init_params ctx_opt_params;
        # 如果参数类型是 ADAM
        if (opt->params.type == GGML_OPT_ADAM) {
            # 计算内存大小
            ctx_opt_params.mem_size = GGML_MEM_ALIGN*3 + ggml_tensor_overhead()*3 + ggml_type_size(GGML_TYPE_F32)*nx*3;
            # 如果过去的步数大于0
            if (opt->params.past > 0) {
                # 增加内存大小
                ctx_opt_params.mem_size += GGML_MEM_ALIGN + ggml_tensor_overhead() + ggml_type_size(GGML_TYPE_F32)*opt->params.past;
            }
        } else if (opt->params.type == GGML_OPT_LBFGS) {
            # 计算内存大小
            ctx_opt_params.mem_size = GGML_MEM_ALIGN*9 + ggml_tensor_overhead()*9 + ggml_type_size(GGML_TYPE_F32)*(nx*5 + opt->params.lbfgs.m*2 + nx*opt->params.lbfgs.m*2);
            # 如果过去的步数大于0
            if (opt->params.past > 0) {
                # 增加内存大小
                ctx_opt_params.mem_size += GGML_MEM_ALIGN + ggml_tensor_overhead() + ggml_type_size(GGML_TYPE_F32)*opt->params.past;
            }
        }
        # 初始化上下文
        ctx_opt_params.mem_buffer = NULL;
        ctx_opt_params.no_alloc   = false;

        # 初始化上下文
        opt->ctx = ggml_init(ctx_opt_params);
    }
}

# 优化函数
enum ggml_opt_result ggml_opt(
        struct ggml_context * ctx,
        struct ggml_opt_params params,
        struct ggml_tensor * f) {
    # 是否释放上下文
    bool free_ctx = false;
    # 如果上下文为空
    if (ctx == NULL) {
        # 初始化上下文参数
        struct ggml_init_params params_ctx = {
            .mem_size   = 16*1024*1024,
            .mem_buffer = NULL,
            .no_alloc   = false,
        };

        # 初始化上下文
        ctx = ggml_init(params_ctx);
        # 如果上下文为空
        if (ctx == NULL) {
            # 返回无上下文错误
            return GGML_OPT_NO_CONTEXT;
        }

        # ��放上下文
        free_ctx = true;
    }

    # 声明结果枚举
    enum ggml_opt_result result = GGML_OPT_OK;
    # 分配内存空间以存储 ggml_opt_context 结构体，并将其地址赋给 opt 指针
    struct ggml_opt_context * opt = (struct ggml_opt_context *) alloca(sizeof(struct ggml_opt_context));
    
    # 初始化 opt 指针指向的 ggml_opt_context 结构体，使用给定的参数和标志
    ggml_opt_init(ctx, opt, params, 0);
    
    # 在给定的上下文和 opt 结构体的情况下，恢复执行 f 函数，并将结果赋给 result
    result = ggml_opt_resume(ctx, opt, f);
    
    # 如果 free_ctx 为真，则释放上下文 ctx 占用的内存空间
    if (free_ctx) {
        ggml_free(ctx);
    }
    
    # 返回执行结果
    return result;
// 恢复优化过程，构建前向和后向计算图
enum ggml_opt_result ggml_opt_resume(
        struct ggml_context * ctx,
        struct ggml_opt_context * opt,
        struct ggml_tensor * f) {

    // 构建前向计算图
    struct ggml_cgraph * gf = ggml_new_graph_custom(ctx, opt->params.graph_size, true);
    ggml_build_forward_expand(gf, f);

    // 复制前向计算图，构建后向计算图
    struct ggml_cgraph * gb = ggml_graph_dup(ctx, gf);
    ggml_build_backward_expand(ctx, gf, gb, true);

    // 调用 ggml_opt_resume_g 函数
    return ggml_opt_resume_g(ctx, opt, f, gf, gb, NULL, NULL);
}

// 恢复优化过程，构建前向和后向计算图
enum ggml_opt_result ggml_opt_resume_g(
        struct ggml_context * ctx,
        struct ggml_opt_context * opt,
        struct ggml_tensor * f,
        struct ggml_cgraph * gf,
        struct ggml_cgraph * gb,
        ggml_opt_callback callback,
        void * callback_data) {

    // 构建前向和后向计算图
    enum ggml_opt_result result = GGML_OPT_OK;

    // 根据不同的优化类型调用不同的优化函数
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

    // 如果需要打印前向计算图，则打印并输出为 dot 文件
    if (opt->params.print_forward_graph) {
        ggml_graph_print   (gf);
        ggml_graph_dump_dot(gf, NULL, "opt-forward.dot");
    }

    // 如果需要打印后向计算图，则打印并输出为 dot 文件
    if (opt->params.print_backward_graph) {
        ggml_graph_print   (gb);
        ggml_graph_dump_dot(gb, gf, "opt-backward.dot");
    }

    return result;
}

////////////////////////////////////////////////////////////////////////////////

// 对输入的浮点数进行量化处理
size_t ggml_quantize_q4_0(const float * src, void * dst, int n, int k, int64_t * hist) {
    // 断言，确保 k 是 QK4_0 的整数倍
    assert(k % QK4_0 == 0);
    const int nb = k / QK4_0;
    # 对输入数据进行分块处理，每次处理 k 个元素
    for (int b = 0; b < n; b += k) {
        # 将目标地址转换为 block_q4_0 类型的指针
        block_q4_0 * restrict y = (block_q4_0 *) dst + b/QK4_0;

        # 对输入数据进行量化处理，存储到目标地址中
        quantize_row_q4_0_reference(src + b, y, k);

        # 遍历每个块中的元素
        for (int i = 0; i < nb; i++) {
            # 遍历每个块中的每个元素的低位和高位
            for (int j = 0; j < QK4_0; j += 2) {
                # 获取当前元素的低位和高位
                const uint8_t vi0 = y[i].qs[j/2] & 0x0F;
                const uint8_t vi1 = y[i].qs[j/2] >> 4;

                # 更新直方图中对应低位和高位的计数
                hist[vi0]++;
                hist[vi1]++;
            }
        }
    }

    # 返回处理的数据量
    return (n/QK4_0*sizeof(block_q4_0));
}
# 定义一个函数，将浮点数数组量化为 Q4_1 格式的数据，并统计直方图
size_t ggml_quantize_q4_1(const float * src, void * dst, int n, int k, int64_t * hist) {
    # 确保 k 是 QK4_1 的倍数
    assert(k % QK4_1 == 0);
    # 计算每个块的数量
    const int nb = k / QK4_1;

    # 遍历每个块
    for (int b = 0; b < n; b += k) {
        # 将目标地址转换为 Q4_1 格式的块
        block_q4_1 * restrict y = (block_q4_1 *) dst + b/QK4_1;

        # 将源数据量化为 Q4_1 格式
        quantize_row_q4_1_reference(src + b, y, k);

        # 遍历每个块的每个元素
        for (int i = 0; i < nb; i++) {
            for (int j = 0; j < QK4_1; j += 2) {
                # 提取每个元素的低四位和高四位
                const uint8_t vi0 = y[i].qs[j/2] & 0x0F;
                const uint8_t vi1 = y[i].qs[j/2] >> 4;

                # 更新直方图
                hist[vi0]++;
                hist[vi1]++;
            }
        }
    }

    # 返回压缩后的数据大小
    return (n/QK4_1*sizeof(block_q4_1));
}

# 定义一个函数，将浮点数数组量化为 Q5_0 格式的数据，并统计直方图
size_t ggml_quantize_q5_0(const float * src, void * dst, int n, int k, int64_t * hist) {
    # 确保 k 是 QK5_0 的倍数
    assert(k % QK5_0 == 0);
    # 计算每个块的数量
    const int nb = k / QK5_0;

    # 遍历每个块
    for (int b = 0; b < n; b += k) {
        # 将目标地址转换为 Q5_0 格式的块
        block_q5_0 * restrict y = (block_q5_0 *)dst + b/QK5_0;

        # 将源数据量化为 Q5_0 格式
        quantize_row_q5_0_reference(src + b, y, k);

        # 遍历每个块的每个元素
        for (int i = 0; i < nb; i++) {
            uint32_t qh;
            memcpy(&qh, &y[i].qh, sizeof(qh));

            for (int j = 0; j < QK5_0; j += 2) {
                # 提取每个元素的高位信息
                const uint8_t vh0 = ((qh & (1u << (j + 0 ))) >> (j + 0 )) << 4;
                const uint8_t vh1 = ((qh & (1u << (j + 16))) >> (j + 12));

                # 将高位信息与低位信息组合成 16 个 bin，并更新直方图
                const uint8_t vi0 = ((y[i].qs[j/2] & 0x0F) | vh0) / 2;
                const uint8_t vi1 = ((y[i].qs[j/2] >>   4) | vh1) / 2;

                hist[vi0]++;
                hist[vi1]++;
            }
        }
    }

    # 返回压缩后的数据大小
    return (n/QK5_0*sizeof(block_q5_0));
}

# 定义一个函数，将浮点数数组量化为 Q5_1 格式的数据，并统计直方图
size_t ggml_quantize_q5_1(const float * src, void * dst, int n, int k, int64_t * hist) {
    # 确保 k 是 QK5_1 的倍数
    assert(k % QK5_1 == 0);
    # 计算每个块的数量
    const int nb = k / QK5_1;
    // 对输入数据进行分块处理，每次处理 k 个元素
    for (int b = 0; b < n; b += k) {
        // 将目标地址转换为 block_q5_1 类型指针
        block_q5_1 * restrict y = (block_q5_1 *)dst + b/QK5_1;

        // 对输入数据进行量化处理
        quantize_row_q5_1_reference(src + b, y, k);

        // 遍历每个块中的元素
        for (int i = 0; i < nb; i++) {
            // 定义一个 32 位无符号整数，用于存储 y[i].qh 的值
            uint32_t qh;
            // 从 y[i].qh 复制 sizeof(qh) 大小的数据到 qh 中
            memcpy(&qh, &y[i].qh, sizeof(qh));

            // 遍历每个块中的元素
            for (int j = 0; j < QK5_1; j += 2) {
                // 从 qh 中提取出两个位，构成一个 8 位无符号整数 vh0 和 vh1
                const uint8_t vh0 = ((qh & (1u << (j + 0 ))) >> (j + 0 )) << 4;
                const uint8_t vh1 = ((qh & (1u << (j + 16))) >> (j + 12));

                // 将 y[i].qs[j/2] 和 vh0 合并成一个 8 位无符号整数 vi0
                const uint8_t vi0 = ((y[i].qs[j/2] & 0x0F) | vh0) / 2;
                // 将 y[i].qs[j/2] 右移 4 位和 vh1 合并成一个 8 位无符号整数 vi1
                const uint8_t vi1 = ((y[i].qs[j/2] >>   4) | vh1) / 2;

                // 更新直方图中 vi0 和 vi1 对应的计数
                hist[vi0]++;
                hist[vi1]++;
            }
        }
    }

    // 返回处理的数据量
    return (n/QK5_1*sizeof(block_q5_1));
// 以 QK8_0 为单位检查 k 是否为 QK8_0 的倍数
size_t ggml_quantize_q8_0(const float * src, void * dst, int n, int k, int64_t * hist) {
    assert(k % QK8_0 == 0);
    // 计算 QK8_0 的数量
    const int nb = k / QK8_0;

    // 对每个块进行量化
    for (int b = 0; b < n; b += k) {
        // 将 dst 转换为 block_q8_0 类型，并定位到正确的位置
        block_q8_0 * restrict y = (block_q8_0 *)dst + b/QK8_0;

        // 对每一行进行量化
        quantize_row_q8_0_reference(src + b, y, k);

        // 统计直方图
        for (int i = 0; i < nb; i++) {
            for (int j = 0; j < QK8_0; ++j) {
                const int8_t vi = y[i].qs[j];

                hist[vi/16 + 8]++;
            }
        }
    }

    // 返回结果的大小
    return (n/QK8_0*sizeof(block_q8_0));
}

// 量化数据块
size_t ggml_quantize_chunk(enum ggml_type type, const float * src, void * dst, int start, int n, int64_t * hist) {
    size_t result = 0;
    // 返回结果
    return result;
}

////////////////////////////////////////////////////////////////////////////////

// GGUF 结构体
struct gguf_str {
    uint64_t n;  // GGUFv2
    char * data;
};

// GGUF 类型的大小
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
    [GGUF_TYPE_ARRAY]   = 0, // 未定义
};
// 断言 GGUF_TYPE_COUNT 是否等于 13
static_assert(GGUF_TYPE_COUNT == 13, "GGUF_TYPE_COUNT != 13");

// GGUF 类型的名称
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
    # 将 GGUF_TYPE_FLOAT64 映射到字符串 "f64"
// 静态断言，确保 GGUF_TYPE_COUNT 的值为 13
static_assert(GGUF_TYPE_COUNT == 13, "GGUF_TYPE_COUNT != 13");

// 定义一个联合体 gguf_value，包含不同类型的值
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

    struct {
        enum gguf_type type;

        uint64_t n;  // GGUFv2
        void * data;
    } arr;
};

// 定义一个键值对结构 gguf_kv
struct gguf_kv {
    struct gguf_str key;  // 键

    enum  gguf_type  type;  // 值的类型
    union gguf_value value;  // 值
};

// 定义一个头部结构 gguf_header
struct gguf_header {
    char magic[4];  // 魔数
    uint32_t version;  // 版本号
    uint64_t n_tensors; // GGUFv2，张量数量
    uint64_t n_kv;      // GGUFv2，键值对数量
};

// 定义一个张量信息结构 gguf_tensor_info
struct gguf_tensor_info {
    struct gguf_str name;  // 张量名称

    uint32_t n_dims;  // 维度数量
    uint64_t ne[GGML_MAX_DIMS];  // 每个维度的元素数量

    enum ggml_type type;  // 数据类型

    uint64_t offset; // 数据在 `data` 中的偏移量，必须是 `ALIGNMENT` 的倍数

    // 用于写入 API
    const void * data;  // 数据
    size_t size;  // 数据大小
};

// 定义一个上下文结构 gguf_context
struct gguf_context {
    struct gguf_header header;  // 头部信息
    enum ggml_sparse_deriv sparse_deriv;  // 稀疏导数

    struct gguf_kv          * kv;  // 键值对数组
    struct gguf_tensor_info * infos;  // 张量信息数组

    size_t alignment;  // 对齐大小
    size_t offset;    // `data` 相对于文件开头的偏移量
    size_t size;      // `data` 的大小，以字节为单位

    //uint8_t * padding;
    void * data;  // 数据
};

// 从文件中读取指定大小的数据到目标地址，并更新偏移量
static bool gguf_fread_el(FILE * file, void * dst, size_t size, size_t * offset) {
    const size_t n = fread(dst, 1, size, file);  // 从文件中读取数据
    *offset += n;  // 更新偏移量
    return n == size;  // 返回读取是否成功
}

// 从文件中读取字符串数据到结构 gguf_str，并更新偏移量
static bool gguf_fread_str(FILE * file, struct gguf_str * p, size_t * offset) {
    p->n    = 0;  // 初始化字符串长度为 0
    p->data = NULL;  // 初始化字符串数据为空

    bool ok = true;  // 初始化读取状态为 true

    ok = ok && gguf_fread_el(file, &p->n,    sizeof(p->n), offset);  // 从文件中读取字符串长度，并更新偏移量
    p->data = calloc(p->n + 1, 1);  // 为字符串数据分配内存
    ok = ok && gguf_fread_el(file,  p->data, p->n,         offset);  // 从文件中读取字符串数据，并更新偏移量

    return ok;  // 返回读取状态
}

// 初始化一个空的 gguf_context 结构
struct gguf_context * gguf_init_empty(void) {
    struct gguf_context * ctx = GGML_ALIGNED_MALLOC(sizeof(struct gguf_context));  // 分配内存空间
    // 将 GGUF_MAGIC 的内容复制到 ctx->header.magic 中
    memcpy(ctx->header.magic, GGUF_MAGIC, sizeof(ctx->header.magic));
    // 设置 ctx->header.version 为 GGUF_VERSION
    ctx->header.version   = GGUF_VERSION;
    // 设置 ctx->header.n_tensors 为 0
    ctx->header.n_tensors = 0;
    // 设置 ctx->header.n_kv 为 0
    ctx->header.n_kv      = 0;

    // 将 ctx->kv 设置为 NULL
    ctx->kv    = NULL;
    // 将 ctx->infos 设置为 NULL
    ctx->infos = NULL;

    // 设置 ctx->alignment 为 GGUF_DEFAULT_ALIGNMENT
    ctx->alignment = GGUF_DEFAULT_ALIGNMENT;
    // 设置 ctx->offset 为 0
    ctx->offset    = 0;
    // 设置 ctx->size 为 0
    ctx->size      = 0;

    // 将 ctx->data 设置为 NULL
    ctx->data = NULL;

    // 返回 ctx
    return ctx;
// 初始化一个空的稀疏上下文结构体
struct gguf_context * gguf_init_empty_sparse(void) {
    // 调用 gguf_init_empty() 函数初始化一个空的上下文结构体
    struct gguf_context * ctx = gguf_init_empty();
    // 将指定长度的数据从 ctx->header.magic 复制到 GGUF_POWERINFER_MAGIC
    memcpy(ctx->header.magic, GGUF_POWERINFER_MAGIC, sizeof(ctx->header.magic));
    // 返回初始化后的上下文结构体
    return ctx;
}

// 从文件初始化上下文结构体
struct gguf_context * gguf_init_from_file(const char * fname, struct gguf_init_params params) {
    // 以只读方式打开文件
    FILE * file = fopen(fname, "rb");
    // 如果文件打开失败，返回空指针
    if (!file) {
        return NULL;
    }

    // 从文件开始的偏移量
    size_t offset = 0;

    char magic[4];
    enum ggml_sparse_deriv sparse_deriv;

    // 在分配内存之前检查魔术字符
    {
        // 从文件中读取指定长度的数据到 magic 数组中，并更新偏移量
        gguf_fread_el(file, &magic, sizeof(magic), &offset);

        // 检查魔术字符是否与 GGUF_MAGIC 相同，确定稀疏派生类型
        if (strncmp(magic, GGUF_MAGIC, sizeof(magic)) == 0) {
            sparse_deriv = GGML_DENSE_INFERENCE;
        } else if (strncmp(magic, GGUF_POWERINFER_MAGIC, sizeof(magic)) == 0) {
            sparse_deriv = GGML_SPARSE_INFERENCE;
        } else {
            // 打印错误信息并关闭文件，返回空指针
            fprintf(stderr, "%s: invalid magic characters %s.\n", __func__, magic);
            fclose(file);
            return NULL;
        }
    }

    bool ok = true;

    // 分配对齐的内存空间来存储上下文结构体
    struct gguf_context * ctx = GGML_ALIGNED_MALLOC(sizeof(struct gguf_context));
    // 设置上下文结构体的稀疏派生类型
    ctx->sparse_deriv = sparse_deriv;

    // 读取头部信息
    {
        // 将 magic 字符串的前 4 个字符复制到 ctx->header.magic 中
        strncpy(ctx->header.magic, magic, 4);

        // 初始化 ctx->kv、ctx->infos、ctx->data 为 NULL
        ctx->kv    = NULL;
        ctx->infos = NULL;
        ctx->data  = NULL;

        // 依次读取文件中的版本号、张量数量、键值对数量，并将结果存入 ctx->header 中
        ok = ok && gguf_fread_el(file, &ctx->header.version,   sizeof(ctx->header.version),   &offset);
        ok = ok && gguf_fread_el(file, &ctx->header.n_tensors, sizeof(ctx->header.n_tensors), &offset);
        ok = ok && gguf_fread_el(file, &ctx->header.n_kv,      sizeof(ctx->header.n_kv),      &offset);

        // 如果版本号为 1，则输出错误信息，关闭文件并释放内存后返回 NULL
        if (ctx->header.version == 1) {
            fprintf(stderr, "%s: GGUFv1 is no longer supported. please use a more up-to-date version\n", __func__);
            fclose(file);
            gguf_free(ctx);
            return NULL;
        }

        // 如果读取失败，则输出错误信息，关闭文件并释放内存后返回 NULL
        if (!ok) {
            fprintf(stderr, "%s: failed to read header\n", __func__);
            fclose(file);
            gguf_free(ctx);
            return NULL;
        }
    }

    // 读取键值对

    // 读取张量信息
    {
        // 为 ctx->infos 分配内存，大小为 ctx->header.n_tensors 乘以 struct gguf_tensor_info 的大小
        ctx->infos = malloc(ctx->header.n_tensors * sizeof(struct gguf_tensor_info));

        // 遍历 ctx->infos 数组
        for (uint32_t i = 0; i < ctx->header.n_tensors; ++i) {
            // 获取当前索引处的 tensor 信息
            struct gguf_tensor_info * info = &ctx->infos[i];

            // 初始化 tensor 的每个维度为 1
            for (int j = 0; j < GGML_MAX_DIMS; ++j) {
                info->ne[j] = 1;
            }

            // 读取 tensor 的名称
            ok = ok && gguf_fread_str(file, &info->name, &offset);
            // 读取 tensor 的维度数量
            ok = ok && gguf_fread_el (file, &info->n_dims, sizeof(info->n_dims), &offset);
            // 读取每个维度的大小
            for (uint32_t j = 0; j < info->n_dims; ++j) {
                ok = ok && gguf_fread_el(file, &info->ne[j], sizeof(info->ne[j]), &offset);
            }
            // 读取 tensor 的数据类型
            ok = ok && gguf_fread_el (file, &info->type, sizeof(info->type), &offset);
            // 读取 tensor 的偏移量
            ok = ok && gguf_fread_el (file, &info->offset, sizeof(info->offset), &offset);

            // 如果读取失败，则输出错误信息，释放内存，返回 NULL
            if (!ok) {
                fprintf(stderr, "%s: failed to read tensor info\n", __func__);
                fclose(file);
                gguf_free(ctx);
                return NULL;
            }
        }
    }

    // 设置默认的对齐方式
    ctx->alignment = GGUF_DEFAULT_ALIGNMENT;

    // 查找并设置特定键的对齐方式
    int alignment_idx = gguf_find_key(ctx, "general.alignment");
    if (alignment_idx != -1) {
        ctx->alignment = gguf_get_val_u32(ctx, alignment_idx);
    }

    // 要求数据部分对齐，所以考虑任何填充
    {
        // 计算偏移量的填充量
        const size_t offset_pad = offset % ctx->alignment;

        // 如果填充量不为 0，则调整偏移量并移动文件指针
        if (offset_pad != 0) {
            offset += ctx->alignment - offset_pad;
            fseek(file, offset, SEEK_SET);
        }
    }

    // 存储当前文件偏移量 - 这是数据部分的起始位置
    ctx->offset = offset;

    // 计算数据部分的总大小，考虑对齐
    {
        // 初始化上下文中的大小为0
        ctx->size = 0;
        // 遍历上下文中头部中张量的数量
        for (uint32_t i = 0; i < ctx->header.n_tensors; ++i) {
            // 获取当前张量的信息
            struct gguf_tensor_info * info = &ctx->infos[i];

            // 计算当前张量的元素数量
            const int64_t ne =
                (int64_t) info->ne[0] *
                (int64_t) info->ne[1] *
                (int64_t) info->ne[2] *
                (int64_t) info->ne[3];

            // 检查当前张量的元素数量是否是块大小的倍数，如果不是则输出错误信息并返回空指针
            if (ne % ggml_blck_size(info->type) != 0) {
                fprintf(stderr, "%s: tensor '%s' number of elements (%" PRId64 ") is not a multiple of block size (%d)\n",
                        __func__, info->name.data, ne, ggml_blck_size(info->type));
                fclose(file);
                gguf_free(ctx);
                return NULL;
            }

            // 计算当前张量数据的大小，并将其加入到上下文的大小中
            const size_t size_cur = (ne*ggml_type_size(info->type))/ggml_blck_size(info->type);
            ctx->size += GGML_PAD(size_cur, ctx->alignment);
        }
    }

    // 只有在请求时加载张量数据

    // 关闭文件
    fclose(file);

    // 返回上下文
    return ctx;
}
// 释放 gguf_context 结构体所占用的内存空间
void gguf_free(struct gguf_context * ctx) {
    // 如果传入的指针为空，则直接返回，不进行任何操作
    if (ctx == NULL) {
        return;
    }

    // 如果 ctx->kv 不为空
    if (ctx->kv) {
        // 释放字符串内存 - 不太好..
        for (uint32_t i = 0; i < ctx->header.n_kv; ++i) {
            // 获取指向当前 kv 结构体的指针
            struct gguf_kv * kv = &ctx->kv[i];

            // 如果 key.data 不为空，则释放 key.data 所指向的内存
            if (kv->key.data) {
                free(kv->key.data);
            }

            // 如果类型为 GGUF_TYPE_STRING
            if (kv->type == GGUF_TYPE_STRING) {
                // 如果 value.str.data 不为空，则释放 value.str.data 所指向的内存
                if (kv->value.str.data) {
                    free(kv->value.str.data);
                }
            }

            // 如果类型为 GGUF_TYPE_ARRAY
            if (kv->type == GGUF_TYPE_ARRAY) {
                // 如果 value.arr.data 不为空
                if (kv->value.arr.data) {
                    // 如果 value.arr.type 为 GGUF_TYPE_STRING
                    if (kv->value.arr.type == GGUF_TYPE_STRING) {
                        // 遍历数组中的每个元素
                        for (uint32_t j = 0; j < kv->value.arr.n; ++j) {
                            // 获取指向当前字符串结构体的指针
                            struct gguf_str * str = &((struct gguf_str *) kv->value.arr.data)[j];
                            // 如果 data 不为空，则释放 data 所指向的内存
                            if (str->data) {
                                free(str->data);
                            }
                        }
                    }
                    // 释放数组数据所占用的内存
                    free(kv->value.arr.data);
                }
            }
        }
        // 释放 kv 数组所占用的内存
        free(ctx->kv);
    }

    // 如果 ctx->infos 不为空
    if (ctx->infos) {
        // 遍历数组中的每个元素
        for (uint32_t i = 0; i < ctx->header.n_tensors; ++i) {
            // 获取指向当前 tensor_info 结构体的指针
            struct gguf_tensor_info * info = &ctx->infos[i];

            // 如果 name.data 不为空，则释放 name.data 所指向的内存
            if (info->name.data) {
                free(info->name.data);
            }
        }
        // 释放 infos 数组所占用的内存
        free(ctx->infos);
    }

    // 释放 ctx 所占用的内存
    GGML_ALIGNED_FREE(ctx);
}

// 返回指定类型的名称
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
# 在给定的 gguf_context 中查找指定的 key，如果找到返回其索引，否则返回 -1
int gguf_find_key(const struct gguf_context * ctx, const char * key) {
    # 初始化 keyfound 为 -1，表示未找到
    int keyfound = -1;

    # 获取 gguf_context 中键值对的数量
    const int n_kv = gguf_get_n_kv(ctx);

    # 遍历键值对数组
    for (int i = 0; i < n_kv; ++i) {
        # 如果找到了指定的 key，更新 keyfound 的值为当前索引并跳出循环
        if (strcmp(key, gguf_get_key(ctx, i)) == 0) {
            keyfound = i;
            break;
        }
    }

    # 返回找到的 key 的索引或者 -1
    return keyfound;
}

# 获取指定 key 对应的键名
const char * gguf_get_key(const struct gguf_context * ctx, int key_id) {
    return ctx->kv[key_id].key.data;
}

# 获取指定 key 对应的值的类型
enum gguf_type gguf_get_kv_type(const struct gguf_context * ctx, int key_id) {
    return ctx->kv[key_id].type;
}

# 获取指定 key 对应的值的数组类型
enum gguf_type gguf_get_arr_type(const struct gguf_context * ctx, int key_id) {
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_ARRAY);
    return ctx->kv[key_id].value.arr.type;
}

# 获取指定 key 对应的值的数组数据
const void * gguf_get_arr_data(const struct gguf_context * ctx, int key_id) {
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_ARRAY);
    return ctx->kv[key_id].value.arr.data;
}

# 获取指定 key 对应的值的数组中指定索引的字符串数据
const char * gguf_get_arr_str(const struct gguf_context * ctx, int key_id, int i) {
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_ARRAY);
    struct gguf_kv * kv = &ctx->kv[key_id];
    struct gguf_str * str = &((struct gguf_str *) kv->value.arr.data)[i];
    return str->data;
}

# 获取指定 key 对应的值的数组的长度
int gguf_get_arr_n(const struct gguf_context * ctx, int key_id) {
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_ARRAY);
    return ctx->kv[key_id].value.arr.n;
}

# 获取指定 key 对应的值的无符号 8 位整数数据
uint8_t gguf_get_val_u8(const struct gguf_context * ctx, int key_id) {
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_UINT8);
    return ctx->kv[key_id].value.uint8;
}

# 获取指定 key 对应的值的有符号 8 位整数数据
int8_t gguf_get_val_i8(const struct gguf_context * ctx, int key_id) {
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_INT8);
    return ctx->kv[key_id].value.int8;
}

# 获取指定 key 对应的值的无符号 16 位整数数据
uint16_t gguf_get_val_u16(const struct gguf_context * ctx, int key_id) {
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_UINT16);
    return ctx->kv[key_id].value.uint16;
}

# 获取指定 key 对应的值的有符号 16 位整数数据
int16_t gguf_get_val_i16(const struct gguf_context * ctx, int key_id) {
    # 检查上下文中键对应的值的类型是否为 int16
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_INT16);
    # 返回上下文中键对应的 int16 值
    return ctx->kv[key_id].value.int16;
# 获取无符号32位整数类型的值
uint32_t gguf_get_val_u32(const struct gguf_context * ctx, int key_id) {
    # 断言键值对的类型为无符号32位整数类型
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_UINT32);
    # 返回键值对的无符号32位整数值
    return ctx->kv[key_id].value.uint32;
}

# 获取有符号32位整数类型的值
int32_t gguf_get_val_i32(const struct gguf_context * ctx, int key_id) {
    # 断言键值对的类型为有符号32位整数类型
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_INT32);
    # 返回键值对的有符号32位整数值
    return ctx->kv[key_id].value.int32;
}

# 获取32位浮点数类型的值
float gguf_get_val_f32(const struct gguf_context * ctx, int key_id) {
    # 断言键值对的类型为32位浮点数类型
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_FLOAT32);
    # 返回键值对的32位浮点数值
    return ctx->kv[key_id].value.float32;
}

# 获取无符号64位整数类型的值
uint64_t gguf_get_val_u64(const struct gguf_context * ctx, int key_id) {
    # 断言键值对的类型为无符号64位整数类型
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_UINT64);
    # 返回键值对的无符号64位整数值
    return ctx->kv[key_id].value.uint64;
}

# 获取有符号64位整数类型的值
int64_t gguf_get_val_i64(const struct gguf_context * ctx, int key_id) {
    # 断言键值对的类型为有符号64位整数类型
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_INT64);
    # 返回键值对的有符号64位整数值
    return ctx->kv[key_id].value.int64;
}

# 获取64位浮点数类型的值
double gguf_get_val_f64(const struct gguf_context * ctx, int key_id) {
    # 断言键值对的类型为64位浮点数类型
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_FLOAT64);
    # 返回键值对的64位浮点数值
    return ctx->kv[key_id].value.float64;
}

# 获取布尔类型的值
bool gguf_get_val_bool(const struct gguf_context * ctx, int key_id) {
    # 断言键值对的类型为布尔类型
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_BOOL);
    # 返回键值对的布尔值
    return ctx->kv[key_id].value.bool_;
}

# 获取字符串类型的值
const char * gguf_get_val_str(const struct gguf_context * ctx, int key_id) {
    # 断言键值对的类型为字符串类型
    GGML_ASSERT(ctx->kv[key_id].type == GGUF_TYPE_STRING);
    # 返回键值对的字符串值
    return ctx->kv[key_id].value.str.data;
}

# 获取张量数量
int gguf_get_n_tensors(const struct gguf_context * ctx) {
    # 返回上下文中的张量数量
    return ctx->header.n_tensors;
}

# 获取稀疏导数类型
enum ggml_sparse_deriv gguf_get_sparse_deriv(const struct gguf_context * ctx) {
    # 返回上下文中的稀疏导数类型
    return ctx->sparse_deriv;
}

# 查找张量
int gguf_find_tensor(const struct gguf_context * ctx, const char * name) {
    # 如果找不到张量，则返回-1
    int tensorfound = -1;

    # 获取张量数量
    const int n_tensors = gguf_get_n_tensors(ctx);

    # 遍历张量，查找指定名称的张量
    for (int i = 0; i < n_tensors; ++i) {
        if (strcmp(name, gguf_get_tensor_name(ctx, i)) == 0) {
            tensorfound = i;
            break;
        }
    }

    # 返回找到的张量索引
    return tensorfound;
}
// 获取指定索引处张量的偏移量
size_t gguf_get_tensor_offset(const struct gguf_context * ctx, int i) {
    return ctx->infos[i].offset;
}

// 获取指定索引处张量的名称
char * gguf_get_tensor_name(const struct gguf_context * ctx, int i) {
    return ctx->infos[i].name.data;
}

// 获取键的索引，如果不存在则添加新键并返回其索引
static int gguf_get_or_add_key(struct gguf_context * ctx, const char * key) {
    // 查找键的索引
    const int idx = gguf_find_key(ctx, key);
    // 如果键已存在，则返回其索引
    if (idx >= 0) {
        return idx;
    }

    // 获取键值对的数量
    const int n_kv = gguf_get_n_kv(ctx);

    // 重新分配内存以容纳新的键值对
    ctx->kv = realloc(ctx->kv, (n_kv + 1) * sizeof(struct gguf_kv));
    // 设置新键的名称和长度
    ctx->kv[n_kv].key.n    = strlen(key);
    ctx->kv[n_kv].key.data = strdup(key);
    ctx->header.n_kv++;

    return n_kv;
}

// 设置无符号8位整数类型的键值对
void gguf_set_val_u8(struct gguf_context * ctx, const char * key, uint8_t val) {
    const int idx = gguf_get_or_add_key(ctx, key);

    ctx->kv[idx].type        = GGUF_TYPE_UINT8;
    ctx->kv[idx].value.uint8 = val;
}

// 设置有符号8位整数类型的键值对
void gguf_set_val_i8(struct gguf_context * ctx, const char * key, int8_t val) {
    const int idx = gguf_get_or_add_key(ctx, key);

    ctx->kv[idx].type       = GGUF_TYPE_INT8;
    ctx->kv[idx].value.int8 = val;
}

// 设置无符号16位整数类型的键值对
void gguf_set_val_u16(struct gguf_context * ctx, const char * key, uint16_t val) {
    const int idx = gguf_get_or_add_key(ctx, key);

    ctx->kv[idx].type         = GGUF_TYPE_UINT16;
    ctx->kv[idx].value.uint16 = val;
}

// 设置有符号16位整数类型的键值对
void gguf_set_val_i16(struct gguf_context * ctx, const char * key, int16_t val) {
    const int idx = gguf_get_or_add_key(ctx, key);

    ctx->kv[idx].type        = GGUF_TYPE_INT16;
    ctx->kv[idx].value.int16 = val;
}

// 设置无符号32位整数类型的键值对
void gguf_set_val_u32(struct gguf_context * ctx, const char * key, uint32_t val) {
    const int idx = gguf_get_or_add_key(ctx, key);

    ctx->kv[idx].type         = GGUF_TYPE_UINT32;
    ctx->kv[idx].value.uint32 = val;
}

// 设置有符号32位整数类型的键值对
void gguf_set_val_i32(struct gguf_context * ctx, const char * key, int32_t val) {
    const int idx = gguf_get_or_add_key(ctx, key);

    ctx->kv[idx].type        = GGUF_TYPE_INT32;
    ctx->kv[idx].value.int32 = val;
}
# 设置浮点数数值到上下文中指定键的数值
void gguf_set_val_f32(struct gguf_context * ctx, const char * key, float val) {
    # 获取或添加键的索引
    const int idx = gguf_get_or_add_key(ctx, key);

    # 设置键值对应的类型为浮点数
    ctx->kv[idx].type          = GGUF_TYPE_FLOAT32;
    # 设置键值对应的浮点数数值
    ctx->kv[idx].value.float32 = val;
}

# 设置无符号64位整数数值到上下文中指定键的数值
void gguf_set_val_u64(struct gguf_context * ctx, const char * key, uint64_t val) {
    # 获取或添加键的索引
    const int idx = gguf_get_or_add_key(ctx, key);

    # 设置键值对应的类型为无符号64位整数
    ctx->kv[idx].type         = GGUF_TYPE_UINT64;
    # 设置键值对应的无符号64位整数数值
    ctx->kv[idx].value.uint64 = val;
}

# 设置有符号64位整数数值到上下文中指定键的数值
void gguf_set_val_i64(struct gguf_context * ctx, const char * key, int64_t val) {
    # 获取或添加键的索引
    const int idx = gguf_get_or_add_key(ctx, key);

    # 设置键值对应的类型为有符号64位整数
    ctx->kv[idx].type        = GGUF_TYPE_INT64;
    # 设置键值对应的有符号64位整数数值
    ctx->kv[idx].value.int64 = val;
}

# 设置双精度浮点数数值到上下文中指定键的数值
void gguf_set_val_f64(struct gguf_context * ctx, const char * key, double val) {
    # 获取或添加键的索引
    const int idx = gguf_get_or_add_key(ctx, key);

    # 设置键值对应的类型为双精度浮点数
    ctx->kv[idx].type          = GGUF_TYPE_FLOAT64;
    # 设置键值对应的双精度浮点数数值
    ctx->kv[idx].value.float64 = val;
}

# 设置布尔值到上下文中指定键的数值
void gguf_set_val_bool(struct gguf_context * ctx, const char * key, bool val) {
    # 获取或添加键的索引
    const int idx = gguf_get_or_add_key(ctx, key);

    # 设置键值对应的类型为布尔值
    ctx->kv[idx].type        = GGUF_TYPE_BOOL;
    # 设置键值对应的布尔值
    ctx->kv[idx].value.bool_ = val;
}

# 设置字符串到上下文中指定键的数值
void gguf_set_val_str(struct gguf_context * ctx, const char * key, const char * val) {
    # 获取或添加键的索引
    const int idx = gguf_get_or_add_key(ctx, key);

    # 设置键值对应的类型为字符串
    ctx->kv[idx].type           = GGUF_TYPE_STRING;
    # 设置键值对应的字符串长度
    ctx->kv[idx].value.str.n    = strlen(val);
    # 复制字符串数据到键值对应的数据中
    ctx->kv[idx].value.str.data = strdup(val);
}

# 设置数组数据到上下文中指定键的数值
void gguf_set_arr_data(struct gguf_context * ctx, const char * key, enum gguf_type type, const void * data, int n) {
    # 获取或添加键的索引
    const int idx = gguf_get_or_add_key(ctx, key);

    # 设置键值对应的类型为数组
    ctx->kv[idx].type           = GGUF_TYPE_ARRAY;
    # 设置数组元素的类型
    ctx->kv[idx].value.arr.type = type;
    # 设置数组元素的个数
    ctx->kv[idx].value.arr.n    = n;
    # 分配内存并复制数组数据到键值对应的数据中
    ctx->kv[idx].value.arr.data = malloc(n*GGUF_TYPE_SIZE[type]);
    memcpy(ctx->kv[idx].value.arr.data, data, n*GGUF_TYPE_SIZE[type]);
}

# 设置字符串数组到上下文中指定键的数值
void gguf_set_arr_str(struct gguf_context * ctx, const char * key, const char ** data, int n) {
    # 获取或添加键的索引
    const int idx = gguf_get_or_add_key(ctx, key);
    # 设置键值对数组中的类型为数组
    ctx->kv[idx].type           = GGUF_TYPE_ARRAY;
    # 设置键值对数组中的值的类型为字符串
    ctx->kv[idx].value.arr.type = GGUF_TYPE_STRING;
    # 设置键值对数组中的值的数量为n
    ctx->kv[idx].value.arr.n    = n;
    # 为键值对数组中的值分配内存空间，大小为n个结构体 gguf_str 的大小
    ctx->kv[idx].value.arr.data = malloc(n*sizeof(struct gguf_str));
    # 遍历数据数组，为每个字符串分配内存并赋值
    for (int i = 0; i < n; i++) {
        # 获取当前索引对应的 gguf_str 结构体指针
        struct gguf_str * str = &((struct gguf_str *)ctx->kv[idx].value.arr.data)[i];
        # 设置当前字符串的长度为 data[i] 的长度
        str->n    = strlen(data[i]);
        # 复制 data[i] 的内容到新分配的内存空间中
        str->data = strdup(data[i]);
    }
// 设置或添加另一个上下文中的键值对
void gguf_set_kv(struct gguf_context * ctx, struct gguf_context * src) {
    // 留空，未实现具体功能
}

// 向上下文中添加张量
void gguf_add_tensor(
             struct gguf_context * ctx,
        const struct ggml_tensor * tensor) {
    // 获取当前张量的索引
    const int idx = ctx->header.n_tensors;
    // 重新分配内存以容纳新的张量信息
    ctx->infos = realloc(ctx->infos, (idx + 1)*sizeof(struct gguf_tensor_info));

    // 设置张量名称的长度和数据
    ctx->infos[idx].name.n    = strlen(tensor->name);
    ctx->infos[idx].name.data = strdup(tensor->name);

    // 初始化张量的每个维度的大小为1
    for (int i = 0; i < GGML_MAX_DIMS; ++i) {
        ctx->infos[idx].ne[i] = 1;
    }

    // 设置张量的维度数量和每个维度的大小
    ctx->infos[idx].n_dims = tensor->n_dims;
    for (int i = 0; i < tensor->n_dims; i++) {
        ctx->infos[idx].ne[i] = tensor->ne[i];
    }

    // 设置张量的类型、偏移量、数据和大小
    ctx->infos[idx].type   = tensor->type;
    ctx->infos[idx].offset = 0;
    ctx->infos[idx].data   = tensor->data;
    ctx->infos[idx].size   = ggml_nbytes(tensor);

    // 如果上下文中已有张量，则更新新张量的偏移量
    if (ctx->header.n_tensors > 0) {
        ctx->infos[idx].offset = ctx->infos[idx - 1].offset + GGML_PAD(ctx->infos[idx - 1].size, ctx->alignment);
    }

    // 增加上下文中张量的数量
    ctx->header.n_tensors++;
}

// 设置张量的类型
void gguf_set_tensor_type(struct gguf_context * ctx, const char * name, enum ggml_type type) {
    // 查找张量的索引
    const int idx = gguf_find_tensor(ctx, name);
    // 如果未找到张量，则断言失败
    if (idx < 0) {
        GGML_ASSERT(false && "tensor not found");
    }

    // 设置张量的类型
    ctx->infos[idx].type = type;
}

// 设置张量的数据
void gguf_set_tensor_data(struct gguf_context * ctx, const char * name, const void * data, size_t size) {
    // 查找张量的索引
    const int idx = gguf_find_tensor(ctx, name);
    // 如果未找到张量，则断言失败
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

// 静态函数，用于向文件中写入字符串
//static void gguf_fwrite_str(FILE * file, const struct gguf_str * val) {
//    fwrite(&val->n,   sizeof(val->n),    1, file);
// 定义一个结构体 gguf_buf，包含数据指针、大小和偏移量
struct gguf_buf {
    void * data;
    size_t size;
    size_t offset;
};

// 初始化 gguf_buf 结构体
static struct gguf_buf gguf_buf_init(size_t size) {
    // 根据传入的大小分配内存，初始化结构体
    struct gguf_buf buf = {
        /*buf.data   =*/ size == 0 ? NULL : malloc(size),
        /*buf.size   =*/ size,
        /*buf.offset =*/ 0,
    };

    return buf;
}

// 释放 gguf_buf 结构体中的数据指针所指向的内存
static void gguf_buf_free(struct gguf_buf buf) {
    if (buf.data) {
        free(buf.data);
    }
}

// 根据需要的大小扩展 gguf_buf 结构体中的数据指针所指向的内存
static void gguf_buf_grow(struct gguf_buf * buf, size_t size) {
    if (buf->offset + size > buf->size) {
        buf->size = 1.5*(buf->offset + size);
        if (buf->data) {
            buf->data = realloc(buf->data, buf->size);
        }
    }
}

// 将 gguf_str 结构体中的数据写入到 gguf_buf 结构体中
static void gguf_bwrite_str(struct gguf_buf * buf, const struct gguf_str * val) {
    gguf_buf_grow(buf, sizeof(val->n) + val->n);

    if (buf->data) {
        memcpy((char *) buf->data + buf->offset, &val->n, sizeof(val->n));
    }
    buf->offset += sizeof(val->n);

    if (buf->data) {
        memcpy((char *) buf->data + buf->offset, val->data, val->n);
    }
    buf->offset += val->n;
}

// 将任意类型的数据写入到 gguf_buf 结构体中
static void gguf_bwrite_el(struct gguf_buf * buf, const void * val, size_t el_size) {
    gguf_buf_grow(buf, el_size);

    if (buf->data) {
        memcpy((char *) buf->data + buf->offset, val, el_size);
    }
    buf->offset += el_size;
}

// 将上下文中的数据写入到 gguf_buf 结构体中
static void gguf_write_to_buf(const struct gguf_context * ctx, struct gguf_buf * buf, bool only_meta) {
    // 写入头部信息
    gguf_bwrite_el(buf, &ctx->header.magic,     sizeof(ctx->header.magic));
    gguf_bwrite_el(buf, &ctx->header.version,   sizeof(ctx->header.version));
    gguf_bwrite_el(buf, &ctx->header.n_tensors, sizeof(ctx->header.n_tensors));
    gguf_bwrite_el(buf, &ctx->header.n_kv,      sizeof(ctx->header.n_kv));

    // 写入键值对
    }

    // 写入张量信息

**注意：** 由于代码片段中的部分代码被省略了，因此最后两行的注释是对省略部分的猜测。
    // 遍历上下文中的张量数量
    for (uint32_t i = 0; i < ctx->header.n_tensors; ++i) {
        // 获取当前张量的信息
        struct gguf_tensor_info * info = &ctx->infos[i];

        // 将张量名称写入缓冲区
        gguf_bwrite_str(buf, &info->name);
        // 将张量维度数量写入缓冲区
        gguf_bwrite_el (buf, &info->n_dims, sizeof(info->n_dims));
        // 遍历张量的每个维度，将维度大小写入缓冲区
        for (uint32_t j = 0; j < info->n_dims; ++j) {
            gguf_bwrite_el(buf, &info->ne[j], sizeof(info->ne[j]));
        }
        // 将张量类型写入缓冲区
        gguf_bwrite_el(buf, &info->type,   sizeof(info->type));
        // 将张量偏移量写入缓冲区
        gguf_bwrite_el(buf, &info->offset, sizeof(info->offset));
    }

    // 确保数据段对齐，考虑任何填充
    {
        // 记录当前偏移量
        const size_t offset     = buf->offset;
        // 计算对齐后的偏移量
        const size_t offset_pad = GGML_PAD(offset, ctx->alignment);

        // 如果对齐后的偏移量不等于当前偏移量
        if (offset_pad != offset) {
            // 填充缓冲区，使得对齐后的偏移量等于当前偏移量
            uint8_t pad = 0;
            for (size_t i = 0; i < offset_pad - offset; ++i) {
                gguf_bwrite_el(buf, &pad, sizeof(pad));
            }
        }
    }

    // 如果只需要元数据，则直接返回
    if (only_meta) {
        return;
    }

    // 初始化偏移量为0
    size_t offset = 0;

    // 写入张量数据
    for (uint32_t i = 0; i < ctx->header.n_tensors; ++i) {
        // 获取当前张量的信息
        struct gguf_tensor_info * info = &ctx->infos[i];

        // 记录当前张量数据大小
        const size_t size     = info->size;
        // 计算对齐后的张量数据大小
        const size_t size_pad = GGML_PAD(size, ctx->alignment);

        // 将张量数据写入缓冲区
        gguf_bwrite_el(buf, info->data, size);

        // 如果对齐后的张量数据大小不等于当前张量数据大小
        if (size_pad != size) {
            // 填充缓冲区，使得对齐后的张量数据大小等于当前张量数据大小
            uint8_t pad = 0;
            for (size_t j = 0; j < size_pad - size; ++j) {
                gguf_bwrite_el(buf, &pad, sizeof(pad));
            }
        }

        // 断言当前偏移量等于张量的偏移量
        GGML_ASSERT(offset == info->offset);

        // 更新偏移量
        offset += size_pad;
    }
// 将数据写入文件
void gguf_write_to_file(const struct gguf_context * ctx, const char * fname, bool only_meta) {
    // 打开文件准备写入
    FILE * file = fopen(fname, "wb");
    // 如果文件打开失败，则输出错误信息
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
    // 不分配内存，只计算大小
    struct gguf_buf buf = gguf_buf_init(0);

    // 将数据写入缓冲区
    gguf_write_to_buf(ctx, &buf, true);

    // 返回缓冲区的大小
    return buf.offset;
}

// 获取元数据的数据
void gguf_get_meta_data(const struct gguf_context * ctx, void * data) {
    // 初始化缓冲区
    struct gguf_buf buf = gguf_buf_init(16*1024);

    // 将数据写入缓冲区
    gguf_write_to_buf(ctx, &buf, true);

    // 将缓冲区的数据复制到指定的数据中
    memcpy(data, buf.data, buf.offset);

    // 释放缓冲区
    gguf_buf_free(buf);
}

// 检查 CPU 是否支持 AVX 指令集
int ggml_cpu_has_avx(void) {
#if defined(__AVX__)
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

// 检查 CPU 是否支持 AVX512VBMI 指令集
int ggml_cpu_has_avx512_vbmi(void) {
#if defined(__AVX512VBMI__)
    return 1;
#else
    return 0;
#endif
}

// 检查 CPU 是否支持 AVX512VNNI 指令集
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
    return 1;
#else
    return 0;
#endif
}

// 检查 CPU 是否支持 FP16 VA 指令集
int ggml_cpu_has_fp16_va(void) {
#if defined(__ARM_FEATURE_FP16_VECTOR_ARITHMETIC)
    // 如果定义了__ARM_FEATURE_FP16_VECTOR_ARITHMETIC，则返回1
    return 1;
#else
    // 否则返回0
    return 0;
#endif
}

int ggml_cpu_has_wasm_simd(void) {
#if defined(__wasm_simd128__)
    // 如果定义了__wasm_simd128__，则返回1
    return 1;
#else
    // 否则返回0
    return 0;
#endif
}

int ggml_cpu_has_blas(void) {
#if defined(GGML_USE_ACCELERATE) || defined(GGML_USE_OPENBLAS) || defined(GGML_USE_CUBLAS) || defined(GGML_USE_CLBLAST)
    // 如果定义了任何一个宏（GGML_USE_ACCELERATE、GGML_USE_OPENBLAS、GGML_USE_CUBLAS、GGML_USE_CLBLAST），则返回1
    return 1;
#else
    // 否则返回0
    return 0;
#endif
}

int ggml_cpu_has_cublas(void) {
#if defined(GGML_USE_CUBLAS)
    // 如果定义了GGML_USE_CUBLAS，则返回1
    return 1;
#else
    // 否则返回0
    return 0;
#endif
}

int ggml_cpu_has_clblast(void) {
#if defined(GGML_USE_CLBLAST)
    // 如果定义了GGML_USE_CLBLAST，则返回1
    return 1;
#else
    // 否则返回0
    return 0;
#endif
}

int ggml_cpu_has_gpublas(void) {
    // 返回GGML_USE_CUBLAS或GGML_USE_CLBLAST的结果
    return ggml_cpu_has_cublas() || ggml_cpu_has_clblast();
}

int ggml_cpu_has_sse3(void) {
#if defined(__SSE3__)
    // 如果定义了__SSE3__，则返回1
    return 1;
#else
    // 否则返回0
    return 0;
#endif
}

int ggml_cpu_has_ssse3(void) {
#if defined(__SSSE3__)
    // 如果定义了__SSSE3__，则返回1
    return 1;
#else
    // 否则返回0
    return 0;
#endif
}

int ggml_cpu_has_vsx(void) {
#if defined(__POWER9_VECTOR__)
    // 如果定义了__POWER9_VECTOR__，则返回1
    return 1;
#else
    // 否则返回0
    return 0;
#endif
}

void ggml_set_backend(struct ggml_tensor * tensor, enum ggml_backend_type backend) {
    if (backend == GGML_BACKEND_CPU) {
        // 如果backend为GGML_BACKEND_CPU，则设置tensor的backend为GGML_BACKEND_CPU并返回
        tensor->backend = GGML_BACKEND_CPU;
        return;
    }
    if (backend == GGML_BACKEND_GPU || backend == GGML_BACKEND_GPU_SPLIT) {
        #if defined(GGML_USE_CUBLAS)
            // 如果定义了GGML_USE_CUBLAS，则设置tensor的backend为传入的backend值并返回
            tensor->backend = backend;
            return;
        #endif
    }
    // 如果不满足上述条件，则抛出错误信息
    GGML_ASSERT(false && "invalid backend");
}

////////////////////////////////////////////////////////////////////////////////
```