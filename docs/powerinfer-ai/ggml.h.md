# `PowerInfer\ggml.h`

```
#pragma once

// GGML Tensor Library
// GGML张量库

// This documentation is still a work in progress.
// If you wish some specific topics to be covered, feel free to drop a comment:
// 如果您希望涵盖一些特定主题，请随时留下评论：
//   https://github.com/ggerganov/whisper.cpp/issues/40

// ## Overview
// ## 概述

// This library implements:
// 该库实现了：

//  - a set of tensor operations
//  - automatic differentiation
//  - basic optimization algorithms
//  - 一组张量操作
//  - 自动微分
//  - 基本优化算法

// The aim of this library is to provide a minimalistic approach for various machine learning tasks. This includes,
// but is not limited to, the following:
// 该库的目的是为各种机器学习任务提供一种极简主义的方法。这包括但不限于以下内容：
// 线性回归
// 支持向量机
// 神经网络

// 该库允许用户使用可用的张量操作来定义特定的函数。这个函数定义在内部通过计算图表示。函数定义中的每个张量操作对应于图中的一个节点。一旦计算图被定义，用户可以选择计算函数的值和/或相对于输入变量的梯度。可选地，可以使用其中一个可用的优化算法来优化函数。

// 例如，这里我们定义函数：f(x) = a*x^2 + b

//   {
//       struct ggml_init_params params = {
//           .mem_size   = 16*1024*1024,
//           .mem_buffer = NULL,
//       };

//       // 内存分配发生在这里
// 初始化一个 ggml 上下文，并返回上下文指针
struct ggml_context * ctx = ggml_init(params);

// 创建一个包含一个元素的一维张量 x，元素类型为 GGML_TYPE_F32
struct ggml_tensor * x = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 1);

// 设置参数 x，x 是一个输入变量
ggml_set_param(ctx, x);

// 创建包含一个元素的一维张量 a 和 b，元素类型为 GGML_TYPE_F32
struct ggml_tensor * a  = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 1);
struct ggml_tensor * b  = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 1);

// 计算 x2 = x * x，返回结果为一个新的张量 x2
struct ggml_tensor * x2 = ggml_mul(ctx, x, x);

// 计算 f = a * x2 + b，返回结果为一个新的张量 f
struct ggml_tensor * f  = ggml_add(ctx, ggml_mul(ctx, a, x2), b);

// ...
}

// 注意上面的函数定义并不涉及任何实际的计算。计算只有在用户显式请求时才执行。
// 例如，要计算 x = 2.0 时的函数值：
//
// {
//       ...
//
// 创建一个新的图形对象
struct ggml_cgraph * gf = ggml_new_graph(ctx);
// 构建图形对象的前向展开
ggml_build_forward_expand(gf, f);

// 设置输入变量和参数值
ggml_set_f32(x, 2.0f);
ggml_set_f32(a, 3.0f);
ggml_set_f32(b, 4.0f);

// 使用上下文和指定的线程数计算图形对象
ggml_graph_compute_with_ctx(ctx, &gf, n_threads);

// 打印计算结果
printf("f = %f\n", ggml_get_f32_1d(f, 0));

// 实际的计算是在 ggml_graph_compute() 函数中执行的

// ggml_new_tensor_...() 函数用于创建新的张量。它们分配在提供给 ggml_init() 函数的内存缓冲区中。
// 你必须小心，不要超出内存缓冲区的大小。因此，你必须预先知道计算所需的内存大小。
// 或者，你可以分配足够大的内存
// 在定义计算图之后，调用ggml_used_mem()函数来查看实际需要多少内存。
//
// ggml_set_param()函数将张量标记为输入变量。这对自动微分和优化算法很有用。
//
// 所描述的方法允许一次定义函数图，然后多次计算其前向或后向图。所有计算都将使用在ggml_init()函数中分配的相同内存缓冲区。这样用户就可以避免运行时的内存分配开销。
//
// 该库支持多维张量-最多4个维度。FP16和FP32数据类型是一等公民，但理论上该库可以扩展支持FP8和整数数据类型。
//
// 每个张量操作都会产生一个新的张量。最初，该库被设想为仅支持一元和二元操作的使用。大多数可用操作都属于这两类之一。随着时间的推移，变得清楚该库需要支持更复杂的操作。如何支持这些操作还不清楚，但在以下操作中演示了一些示例：
//
//   - ggml_permute()
//   - ggml_conv_1d_1s()
//   - ggml_conv_1d_2s()
//
// 对于每个张量运算符，库实现了前向和后向计算函数。前向函数计算给定输入张量值的输出张量值。后向函数计算输出张量的伴随值给定输入张量的伴随值。关于这意味着什么的详细解释，请上微积分课，或观看以下视频：
//
//   什么是自动微分？
//   https://www.youtube.com/watch?v=wG_nF1awSSY
//
//
// ## 张量数据 (struct ggml_tensor)
//
// 张量通过 ggml_tensor 结构存储在内存中。该结构提供有关张量大小、数据类型以及存储张量数据的内存缓冲区的信息。此外，它包含指向“源”张量的指针 - 即用于计算当前张量的张量。例如：
//
//   {
//       struct ggml_tensor * c = ggml_add(ctx, a, b);
//
// 断言，确保张量c的第一个元素等于a，第二个元素等于b
// 多维张量以行优先顺序存储。ggml_tensor结构包含每个维度中的元素数量（"ne"）以及字节数（"nb"，也称为步幅）。这允许存储在内存中不连续的张量，这对于诸如转置和排列的操作非常有用。所有张量操作都必须考虑步幅，并不假设张量在内存中是连续的。
// 通过"data"指针访问张量的数据。例如：
// 创建一个2维的浮点型张量a
const int nx = 2;
const int ny = 3;
struct ggml_tensor * a = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, nx, ny);

// 遍历张量a的数据
for (int y = 0; y < ny; y++) {
    for (int x = 0; x < nx; x++) {
// 在给定的数据结构a中，根据给定的x和y坐标计算出对应的内存地址，并将x + y的值存储在该地址上
// 另外，也可以使用辅助函数ggml_get_f32_1d()和ggml_set_f32_1d()来实现相同的功能
// ggml_mul_mat矩阵乘法运算符的概述
// 多线程处理的概述
// ggml.c文件的概述
// 如果 GGML_SHARED 宏已定义
#ifdef GGML_SHARED
    // 如果操作系统为 Windows，并且不是使用 MinGW 编译
    #if defined(_WIN32) && !defined(__MINGW32__)
        // 如果 GGML_BUILD 宏已定义
        #ifdef GGML_BUILD
            // 定义 GGML_API 为 __declspec(dllexport)
            #define GGML_API __declspec(dllexport)
        // 否则
        #else
            // TODO
        // 结束 GGML_BUILD 宏的判断
        #endif
    // 结束操作系统为 Windows，并且不是使用 MinGW 编译的判断
    #endif
// 结束 GGML_SHARED 宏的判断
#endif
# 如果定义了 _WIN32，则定义 GGML_API 为 __declspec(dllimport)
# 否则，如果定义了 __GNUC__，则定义 GGML_API 为 __attribute__ ((visibility ("default")))
# 否则，定义 GGML_API 为空
#ifdef _WIN32
#    define GGML_API __declspec(dllimport)
#        endif
#    else
#        define GGML_API __attribute__ ((visibility ("default")))
#    endif
#else
#    define GGML_API
#endif

// TODO: support for clang
// 如果定义了 __GNUC__，则定义 GGML_DEPRECATED(func, hint) 为 func __attribute__((deprecated(hint)))
// 否则，如果定义了 _MSC_VER，则定义 GGML_DEPRECATED(func, hint) 为 __declspec(deprecated(hint)) func
// 否则，定义 GGML_DEPRECATED(func, hint) 为 func
#ifdef __GNUC__
#    define GGML_DEPRECATED(func, hint) func __attribute__((deprecated(hint)))
#elif defined(_MSC_VER)
#    define GGML_DEPRECATED(func, hint) __declspec(deprecated(hint)) func
#else
#    define GGML_DEPRECATED(func, hint) func
#endif

// 如果未定义 __GNUC__，则定义 GGML_ATTRIBUTE_FORMAT 为空
#ifndef __GNUC__
#    define GGML_ATTRIBUTE_FORMAT(...)
#ifdef __MINGW32__
#    define GGML_ATTRIBUTE_FORMAT(...) __attribute__((format(gnu_printf, __VA_ARGS__)))
#else
#    define GGML_ATTRIBUTE_FORMAT(...) __attribute__((format(printf, __VA_ARGS__)))
#endif
```
如果定义了__MINGW32__，则使用GNU风格的printf格式，否则使用普通的printf格式。

```
#include <stdint.h>
#include <stddef.h>
#include <stdbool.h>
#ifdef __cplusplus
  #include <atomic>
  using std::atomic_int;
  using std::memory_order;
  using std::memory_order_acquire;
#else /* not __cplusplus */
  #include <stdatomic.h>
#endif /* __cplusplus */
```
包含一些标准的C库头文件，并根据是否是C++环境来选择是否包含atomic头文件，并使用相应的命名空间。

```
#define GGML_FILE_MAGIC   0x67676d6c // "ggml"
#define GGML_FILE_VERSION 1
```
定义了两个常量，GGML_FILE_MAGIC和GGML_FILE_VERSION。 GGML_FILE_MAGIC的值为0x67676d6c，GGML_FILE_VERSION的值为1。
// 定义量化格式的版本号，当量化格式发生变化时需要增加版本号
#define GGML_QNT_VERSION        2    

// 量化版本号的因子，不要更改此值
#define GGML_QNT_VERSION_FACTOR 1000 

// 最大维度
#define GGML_MAX_DIMS           4

// 最大参数数量
#define GGML_MAX_PARAMS         1024

// 最大上下文数量
#define GGML_MAX_CONTEXTS       64

// 最大源数量
#define GGML_MAX_SRC            6

// 最大名称长度
#define GGML_MAX_NAME           64

// 最大操作参数数量
#define GGML_MAX_OP_PARAMS      64

// 默认线程数
#define GGML_DEFAULT_N_THREADS  4

// 默认图形大小
#define GGML_DEFAULT_GRAPH_SIZE 2048

// 根据指针大小确定内存对齐方式
#if UINTPTR_MAX == 0xFFFFFFFF
    #define GGML_MEM_ALIGN 4
#else
    #define GGML_MEM_ALIGN 16
#endif

// 成功退出状态码
#define GGML_EXIT_SUCCESS 0

// 中止退出状态码
#define GGML_EXIT_ABORTED 1
// 定义 GGUF_MAGIC 为字符串 "GGUF"
#define GGUF_MAGIC "GGUF"
// 定义 GGUF_POWERINFER_MAGIC 为字符串 "PWRI"
#define GGUF_POWERINFER_MAGIC "PWRI"

// 定义 GGUF_VERSION 为整数 3
#define GGUF_VERSION 3

// 定义 GGUF_DEFAULT_ALIGNMENT 为整数 32
#define GGUF_DEFAULT_ALIGNMENT 32

// 定义 GGML_UNUSED 宏，用于标记未使用的变量
#define GGML_UNUSED(x) (void)(x)

// 定义 GGML_PAD 宏，用于对齐内存
#define GGML_PAD(x, n) (((x) + (n) - 1) & ~((n) - 1))

// 定义 GGML_ASSERT 宏，用于断言条件，如果条件不满足则输出错误信息并退出程序
#define GGML_ASSERT(x) \
    do { \
        if (!(x)) { \
            fprintf(stderr, "GGML_ASSERT: %s:%d: %s\n", __FILE__, __LINE__, #x); \
            fflush(stderr); \
            fflush(stdout); \
            ggml_print_backtrace(); \
            exit(1); \
// 定义一个带有断言信息的宏，如果条件不满足，则打印错误信息，输出调用栈，然后退出程序
#define GGML_ASSERT_DBG(x, s, ...) \
    do { \
        if (!(x)) { \
            fprintf(stderr, "GGML_ASSERT: %s:%d: " s "\n", __FILE__, __LINE__, ##__VA_ARGS__); \
            fflush(stderr); \
            fflush(stdout); \
            ggml_print_backtrace(); \
            exit(1); \
        } \
    } while (0)

// 如果处于调试模式，则定义一个不可达的宏，否则根据编译器类型定义不可达的宏
#ifndef NDEBUG
#define GGML_UNREACHABLE() GGML_ASSERT(!"statement should not be reached")
#elif defined(__GNUC__)
#define GGML_UNREACHABLE() __builtin_unreachable()
#else
#define GGML_UNREACHABLE() ((void) 0)
// 用于将张量的元素数量和字节步长复制到本地变量中。
// 主要目的是减少代码重复，提高可读性。
//
// 示例：
//
//    GGML_TENSOR_LOCALS(int64_t, ne1, src1, ne);
//    GGML_TENSOR_LOCALS(size_t,  nb1, src1, nb);
//
// 宏定义，用于定义一个局部变量，表示张量的元素数量和字节步长
#define GGML_TENSOR_LOCALS_1(type, prefix, pointer, array) \
    const type prefix##0 = (pointer)->array[0]; \  // 定义一个局部变量，表示张量的第一个元素数量或字节步长
    GGML_UNUSED(prefix##0);  // 使用宏定义，避免编译器警告
#define GGML_TENSOR_LOCALS_2(type, prefix, pointer, array) \
    GGML_TENSOR_LOCALS_1    (type, prefix, pointer, array) \  // 调用上一个宏定义
    const type prefix##1 = (pointer)->array[1]; \  // 定义一个局部变量，表示张量的第二个元素数量或字节步长
    GGML_UNUSED(prefix##1);  // 使用宏定义，避免编译器警告
#define GGML_TENSOR_LOCALS_3(type, prefix, pointer, array) \
    GGML_TENSOR_LOCALS_2    (type, prefix, pointer, array) \  // 调用上一个宏定义
    const type prefix##2 = (pointer)->array[2]; \  // 定义一个局部变量，表示张量的第三个元素数量或字节步长
    // 定义宏，用于标记未使用的变量
    GGML_UNUSED(prefix##2);

    // 定义宏，用于声明局部变量，并且包含对应的未使用标记
#define GGML_TENSOR_LOCALS(type, prefix, pointer, array) \
    // 调用另一个宏，定义局部变量
    GGML_TENSOR_LOCALS_3  (type, prefix, pointer, array) \
    // 声明并初始化局部变量
    const type prefix##3 = (pointer)->array[3]; \
    // 使用未使用标记
    GGML_UNUSED(prefix##3);

#ifdef  __cplusplus
extern "C" {
#endif

#if defined(__ARM_NEON) && defined(__CUDACC__)
    // 根据条件定义不同的数据类型
    typedef half ggml_fp16_t;
#elif defined(__ARM_NEON)
    // 根据条件定义不同的数据类型
    typedef __fp16 ggml_fp16_t;
#else
    // 根据条件定义不同的数据类型
    typedef uint16_t ggml_fp16_t;
#endif

    // 声明函数，用于 FP16 和 FP32 之间的转换
    GGML_API float       ggml_fp16_to_fp32(ggml_fp16_t x);
# 定义一个函数，将单精度浮点数转换为半精度浮点数
GGML_API ggml_fp16_t ggml_fp32_to_fp16(float x);

# 定义一个函数，将一行半精度浮点数数组转换为单精度浮点数数组
GGML_API void ggml_fp16_to_fp32_row(const ggml_fp16_t * x, float * y, int n);

# 定义一个函数，将一行单精度浮点数数组转换为半精度浮点数数组
GGML_API void ggml_fp32_to_fp16_row(const float * x, ggml_fp16_t * y, int n);

# 定义一个结构体 ggml_object
struct ggml_object;

# 定义一个结构体 ggml_context
struct ggml_context;

# 定义一个枚举类型 ggml_type，包括不同的数据类型
enum ggml_type {
    GGML_TYPE_F32  = 0,  # 单精度浮点数
    GGML_TYPE_F16  = 1,  # 半精度浮点数
    GGML_TYPE_Q4_0 = 2,  # 四位定点数，小数点位置为0
    GGML_TYPE_Q4_1 = 3,  # 四位定点数，小数点位置为1
    GGML_TYPE_Q5_0 = 6,  # 五位定点数，小数点位置为0
    GGML_TYPE_Q5_1 = 7,  # 五位定点数，小数点位置为1
    GGML_TYPE_Q8_0 = 8,  # 八位定点数，小数点位置为0
    GGML_TYPE_Q8_1 = 9,  # 八位定点数，小数点位置为1
    # k-quantizations
# 定义枚举类型，表示不同的数据类型
GGML_TYPE_Q2_K = 10,  # 表示 Q2_K 数据类型
GGML_TYPE_Q3_K = 11,  # 表示 Q3_K 数据类型
GGML_TYPE_Q4_K = 12,  # 表示 Q4_K 数据类型
GGML_TYPE_Q5_K = 13,  # 表示 Q5_K 数据类型
GGML_TYPE_Q6_K = 14,  # 表示 Q6_K 数据类型
GGML_TYPE_Q8_K = 15,  # 表示 Q8_K 数据类型
GGML_TYPE_I8,         # 表示 I8 数据类型
GGML_TYPE_I16,        # 表示 I16 数据类型
GGML_TYPE_I32,        # 表示 I32 数据类型
GGML_TYPE_COUNT,      # 表示数据类型的数量

# 定义枚举类型，表示不同的后端类型
GGML_BACKEND_CPU = 0,        # 表示 CPU 后端
GGML_BACKEND_GPU = 10,       # 表示 GPU 后端
GGML_BACKEND_GPU_SPLIT = 20, # 表示分割的 GPU 后端

# 定义枚举类型，表示不同的稀疏导数类型
GGML_DENSE_INFERENCE = 0,    # 表示密集推断
// 稀疏推断类型
enum ggml_sparse_inference {
    GGML_SPARSE_INFERENCE = 1,
};

// 模型文件类型
enum ggml_ftype {
    GGML_FTYPE_UNKNOWN     = -1,
    GGML_FTYPE_ALL_F32     = 0,
    GGML_FTYPE_MOSTLY_F16  = 1,  // 除了1维张量
    GGML_FTYPE_MOSTLY_Q4_0 = 2,  // 除了1维张量
    GGML_FTYPE_MOSTLY_Q4_1 = 3,  // 除了1维张量
    GGML_FTYPE_MOSTLY_Q4_1_SOME_F16 = 4, // tok_embeddings.weight 和 output.weight 是 F16
    GGML_FTYPE_MOSTLY_Q8_0 = 7,  // 除了1维张量
    GGML_FTYPE_MOSTLY_Q5_0 = 8,  // 除了1维张量
    GGML_FTYPE_MOSTLY_Q5_1 = 9,  // 除了1维张量
    GGML_FTYPE_MOSTLY_Q2_K = 10, // 除了1维张量
    GGML_FTYPE_MOSTLY_Q3_K = 11, // 除了1维张量
    GGML_FTYPE_MOSTLY_Q4_K = 12, // 除了1维张量
    GGML_FTYPE_MOSTLY_Q5_K = 13, // 除了1维张量
    GGML_FTYPE_MOSTLY_Q6_K = 14, // 除了1维张量
};
// 定义了可用的张量操作枚举类型
enum ggml_op {
    GGML_OP_NONE = 0, // 无操作

    GGML_OP_DUP, // 复制
    GGML_OP_ADD, // 加法
    GGML_OP_ADD1, // 加1
    GGML_OP_ACC, // 累加
    GGML_OP_SUB, // 减法
    GGML_OP_MUL, // 乘法
    GGML_OP_DIV, // 除法
    GGML_OP_SQR, // 平方
    GGML_OP_SQRT, // 平方根
    GGML_OP_LOG, // 对数
    GGML_OP_SUM, // 求和
    GGML_OP_SUM_ROWS, // 对行求和
    GGML_OP_MEAN, // 求平均值
    GGML_OP_ARGMAX, // 最大值索引
    GGML_OP_REPEAT, // 重复
        GGML_OP_REPEAT_BACK, // 重复操作
        GGML_OP_CONCAT, // 连接操作
        GGML_OP_SILU_BACK, // SILU 反向操作
        GGML_OP_NORM, // 归一化操作
        GGML_OP_RMS_NORM, // RMS 归一化操作
        GGML_OP_RMS_NORM_BACK, // RMS 归一化反向操作
        GGML_OP_GROUP_NORM, // 分组归一化操作

        GGML_OP_MUL_MAT, // 矩阵相乘操作
        GGML_OP_AXPY, // AXPY 操作
        GGML_OP_OUT_PROD, // 外积操作

        GGML_OP_SCALE, // 缩放操作
        GGML_OP_SET, // 设置操作
        GGML_OP_CPY, // 复制操作
        GGML_OP_CONT, // 连续操作
        GGML_OP_RESHAPE, // 重塑操作
        GGML_OP_VIEW, // 视图操作
        GGML_OP_PERMUTE, // 排列操作
        GGML_OP_TRANSPOSE, // 转置操作
        GGML_OP_GET_ROWS, // 获取行数据操作
        GGML_OP_GET_ROWS_BACK, // 获取行数据操作的反向操作
        GGML_OP_DIAG, // 对角线操作
        GGML_OP_DIAG_MASK_INF, // 对角线掩码（无穷大）操作
        GGML_OP_DIAG_MASK_ZERO, // 对角线掩码（零）操作
        GGML_OP_SOFT_MAX, // 软最大值操作
        GGML_OP_SOFT_MAX_BACK, // 软最大值操作的反向操作
        GGML_OP_ROPE, // 绳索操作
        GGML_OP_ROPE_BACK, // 绳索操作的反向操作
        GGML_OP_ALIBI, // 辅助操作
        GGML_OP_CLAMP, // 夹紧操作
        GGML_OP_CONV_TRANSPOSE_1D, // 1D卷积转置操作
        GGML_OP_IM2COL, // 图像转列操作
        GGML_OP_CONV_TRANSPOSE_2D, // 2D卷积转置操作
        GGML_OP_POOL_1D, // 1D池化操作
        GGML_OP_POOL_2D, // 2D池化操作

        GGML_OP_UPSCALE, // 最近邻插值操作

        GGML_OP_FLASH_ATTN, // 闪光注意力操作
# 定义不同的操作类型，如闪烁、窗口分割、获取相对位置等
GGML_OP_FLASH_FF,  # 闪烁
GGML_OP_FLASH_ATTN_BACK,  # 闪烁并返回
GGML_OP_WIN_PART,  # 窗口分割
GGML_OP_WIN_UNPART,  # 窗口取消分割
GGML_OP_GET_REL_POS,  # 获取相对位置
GGML_OP_ADD_REL_POS,  # 添加相对位置

GGML_OP_UNARY,  # 一元操作

GGML_OP_MAP_UNARY,  # 映射一元操作
GGML_OP_MAP_BINARY,  # 映射二元操作

GGML_OP_MAP_CUSTOM1_F32,  # 映射自定义类型1（32位浮点数）
GGML_OP_MAP_CUSTOM2_F32,  # 映射自定义类型2（32位浮点数）
GGML_OP_MAP_CUSTOM3_F32,  # 映射自定义类型3（32位浮点数）

GGML_OP_MAP_CUSTOM1,  # 映射自定义类型1
GGML_OP_MAP_CUSTOM2,  # 映射自定义类型2
GGML_OP_MAP_CUSTOM3,  # 映射自定义类型3
    // 定义神经网络中的交叉熵损失操作和其反向传播操作
    GGML_OP_CROSS_ENTROPY_LOSS,
    GGML_OP_CROSS_ENTROPY_LOSS_BACK,

    // 定义神经网络中的计数操作
    GGML_OP_COUNT,
};

// 定义神经网络中的一元操作，如绝对值、符号函数、取反等
enum ggml_unary_op {
    GGML_UNARY_OP_ABS,
    GGML_UNARY_OP_SGN,
    GGML_UNARY_OP_NEG,
    GGML_UNARY_OP_STEP,
    GGML_UNARY_OP_TANH,
    GGML_UNARY_OP_ELU,
    GGML_UNARY_OP_RELU,
    GGML_UNARY_OP_GELU,
    GGML_UNARY_OP_GELU_QUICK,
    GGML_UNARY_OP_SILU,
    GGML_UNARY_OP_LEAKY
};
    // 定义枚举类型，表示 ggml_object 的类型
    enum ggml_object_type {
        GGML_OBJECT_TENSOR, // 表示张量类型
        GGML_OBJECT_GRAPH, // 表示图类型
        GGML_OBJECT_WORK_BUFFER // 表示工作缓冲区类型
    };

    // 定义枚举类型，表示日志级别
    enum ggml_log_level {
        GGML_LOG_LEVEL_ERROR = 2, // 表示错误级别
        GGML_LOG_LEVEL_WARN = 3, // 表示警告级别
        GGML_LOG_LEVEL_INFO = 4 // 表示信息级别
    };

    // ggml_object 结构体，表示通用对象
    struct ggml_object {
        size_t offs; // 偏移量
        size_t size; // 大小

        struct ggml_object * next; // 指向下一个 ggml_object 结构体的指针

        enum ggml_object_type type; // 表示对象类型的枚举值
    // 4字节填充，用于对齐结构体
    char padding[4];
    };

    // 定义结构体 ggml_object 的大小
    static const size_t GGML_OBJECT_SIZE = sizeof(struct ggml_object);

    // n维张量
    struct ggml_tensor {
        // 数据类型
        enum ggml_type         type;
        // 后端类型
        enum ggml_backend_type backend;

        // 后端缓冲区
        struct ggml_backend_buffer * buffer;

        // 维度数
        int     n_dims;
        // 每个维度的元素个数
        int64_t ne[GGML_MAX_DIMS]; // number of elements
        // 每个维度的步长（以字节为单位）
        size_t  nb[GGML_MAX_DIMS]; // stride in bytes:
                                   // nb[0] = ggml_type_size(type)
                                   // nb[1] = nb[0]   * (ne[0] / ggml_blck_size(type)) + padding
                                   // nb[i] = nb[i-1] * ne[i-1]
// 计算数据
enum ggml_op op; // 定义枚举类型变量 op，用于表示操作类型

// op 参数 - 为了对齐而分配为 int32_t 类型
int32_t op_params[GGML_MAX_OP_PARAMS / sizeof(int32_t)]; // 定义整型数组 op_params，用于存储操作参数

bool is_param; // 布尔变量 is_param，用于表示是否有参数

struct ggml_tensor * grad; // 定义指向 ggml_tensor 结构体的指针 grad，用于表示梯度
struct ggml_tensor * src[GGML_MAX_SRC]; // 定义指向 ggml_tensor 结构体的指针数组 src，用于表示源数据

// 性能
atomic_int is_finish; // 原子整型变量 is_finish，用于表示是否完成
int perf_runs; // 整型变量 perf_runs，用于表示性能运行次数
int64_t perf_cycles; // 长整型变量 perf_cycles，用于表示性能周期数
int64_t perf_time_us; // 长整型变量 perf_time_us，用于表示性能时间（微秒）

struct ggml_tensor * view_src; // 定义指向 ggml_tensor 结构体的指针 view_src，用于表示视图源数据
size_t view_offs; // size_t 类型变量 view_offs，用于表示视图偏移量
// 用于存储数据的指针
void * data;

// 用于存储名称的字符数组
char name[GGML_MAX_NAME];

// 用于存储额外的数据，例如 ggml-cuda.cu 中的内容
void * extra;

// 用于填充的字符数组，长度为12
char padding[12];
};

// 表示通配符的常量
static const int64_t GGML_NE_WILDCARD = -1;

// 表示 ggml_tensor 结构体的大小
static const size_t GGML_TENSOR_SIZE = sizeof(struct ggml_tensor);

// 需要为 ggml_graph_compute() 准备的计算计划
// 自 https://github.com/ggerganov/ggml/issues/287 以来
struct ggml_cplan {
    // 工作缓冲区的大小，由 `ggml_graph_plan()` 计算得出
    size_t work_size;

    // 工作缓冲区，需要在调用 `ggml_graph_compute()` 之前由调用者分配
    uint8_t * work_data;
        // 定义整型变量 n_threads
        int n_threads;

        // 定义一个函数指针，用于判断是否中止 ggml_graph_compute
        bool (*abort_callback)(void * data);
        // 用于传递给中止回调函数的数据
        void * abort_callback_data;
    };

    // 定义枚举类型 ggml_cgraph_eval_order，表示计算图的评估顺序
    enum ggml_cgraph_eval_order {
        GGML_CGRAPH_EVAL_ORDER_LEFT_TO_RIGHT = 0, // 从左到右的评估顺序
        GGML_CGRAPH_EVAL_ORDER_RIGHT_TO_LEFT, // 从右到左的评估顺序
        GGML_CGRAPH_EVAL_ORDER_COUNT // 评估顺序的数量
    };

    // 定义哈希集合结构体 ggml_hash_set
    struct ggml_hash_set {
        // 哈希集合的大小
        size_t size;
        // 存储键的 ggml_tensor 指针数组
        struct ggml_tensor ** keys;
    };

    // 计算图结构体 ggml_cgraph
    struct ggml_cgraph {
        // 定义变量 size，用于存储节点的大小
        int size;
        // 定义变量 n_nodes，用于存储节点的数量
        int n_nodes;
        // 定义变量 n_leafs，用于存储叶子节点的数量

        // 定义指向 ggml_tensor 结构体指针的数组 nodes，用于存储节点数据
        struct ggml_tensor ** nodes;
        // 定义指向 ggml_tensor 结构体指针的数组 grads，用于存储梯度数据
        struct ggml_tensor ** grads;
        // 定义指向 ggml_tensor 结构体指针的数组 leafs，用于存储叶子节点数据

        // 定义哈希表 visited_hash_table，用于存储已访问的节点
        struct ggml_hash_set visited_hash_table;

        // 定义枚举类型变量 order，用于存储计算图的评估顺序

        // 性能相关变量
        // 定义变量 perf_runs，用于存储性能运行次数
        int     perf_runs;
        // 定义变量 perf_cycles，用于存储性能周期数
        int64_t perf_cycles;
        // 定义变量 perf_time_us，用于存储性能时间（微秒）

    };

    // 临时缓冲区
    // 定义结构体 ggml_scratch，用于存储临时缓冲区数据
    struct ggml_scratch {
// 定义一个结构体 ggml_object，包含偏移量、大小和数据
    size_t offs;
    size_t size;
    void * data;
};

// 定义一个结构体 ggml_context，包含内存大小、内存缓冲区、内存缓冲区所有权标志、禁止分配内存标志、保存禁止分配内存状态的标志、对象数量、对象起始指针、对象结束指针、临时缓冲区、保存的临时缓冲区
struct ggml_context {
    size_t mem_size;
    void * mem_buffer;
    bool   mem_buffer_owned;
    bool   no_alloc;
    bool   no_alloc_save; // 用于保存在使用临时缓冲区时的禁止分配内存状态
    int    n_objects;
    struct ggml_object * objects_begin;
    struct ggml_object * objects_end;
    struct ggml_scratch scratch;
    struct ggml_scratch scratch_save;
};
// 定义结构体 ggml_init_params，用于初始化参数
struct ggml_init_params {
    // 内存池大小，以字节为单位
    size_t mem_size;
    // 内存缓冲区指针，如果为 NULL，则内部将分配内存
    void * mem_buffer;
    // 是否不分配张量数据的内存
    bool   no_alloc;
};

// 计算类型

// 注意：除非显式启用，否则不会安排 INIT 或 FINALIZE 传递。此行为自 https://github.com/ggerganov/llama.cpp/pull/1995 以来已更改。
enum ggml_task_type {
    GGML_TASK_INIT = 0,
    GGML_TASK_COMPUTE,
    GGML_TASK_FINALIZE,
};

// 定义结构体 ggml_compute_params，用于计算参数
struct ggml_compute_params {
// 定义枚举类型 ggml_task_type
enum ggml_task_type type;

// 定义整型变量 ith 表示线程索引，nth 表示线程数量
int ith, nth;

// 为所有线程准备的工作缓冲区
size_t wsize;
void * wdata;
atomic_int *aic;

// 初始化时间函数，程序开始时调用一次
GGML_API void    ggml_time_init(void);
// 返回当前时间的毫秒数
GGML_API int64_t ggml_time_ms(void);
// 返回当前时间的微秒数
GGML_API int64_t ggml_time_us(void);
// 返回 CPU 循环次数
GGML_API int64_t ggml_cycles(void);
// 返回每毫秒的 CPU 循环次数
GGML_API int64_t ggml_cycles_per_ms(void);

// 打印函数调用的回溯信息
GGML_API void    ggml_print_backtrace(void);
// 初始化 NUMA 系统，提高性能
GGML_API void ggml_numa_init(void);

// 检测系统是否有多个 NUMA 节点
GGML_API bool ggml_is_numa(void);

// 打印单个 ggml_object 对象
GGML_API void ggml_print_object(const struct ggml_object *obj);

// 打印 ggml_context 中的所有对象
GGML_API void ggml_print_objects(const struct ggml_context *ctx);

// 返回 tensor 中元素的数量
GGML_API int64_t ggml_nelements(const struct ggml_tensor *tensor);

// 返回 tensor 中的行数
GGML_API int64_t ggml_nrows(const struct ggml_tensor *tensor);

// 返回 tensor 占用的字节数
GGML_API size_t ggml_nbytes(const struct ggml_tensor *tensor);

// 返回 tensor 占用的字节数，按 GGML_MEM_ALIGN 进行填充
GGML_API size_t ggml_nbytes_pad(const struct ggml_tensor *tensor);

// 返回按指定行数分割后的 tensor 占用的字节数
GGML_API size_t ggml_nbytes_split(const struct ggml_tensor *tensor, int nrows_split);

// 返回指定类型的块大小
GGML_API int ggml_blck_size(enum ggml_type type);

// 返回指定类型的每个元素的字节数
GGML_API size_t ggml_type_size(enum ggml_type type);

// 返回指定类型的每个元素的字节数，以浮点数形式返回
GGML_API float ggml_type_sizef(enum ggml_type type);

// 返回指定类型的名称
GGML_API const char *ggml_type_name(enum ggml_type type);
// 返回操作符对应的名称
GGML_API const char * ggml_op_name  (enum ggml_op   op);

// 返回操作符对应的符号
GGML_API const char * ggml_op_symbol(enum ggml_op   op);

// 返回张量元素的大小
GGML_API size_t  ggml_element_size(const struct ggml_tensor * tensor);

// 判断类型是否是量化的
GGML_API bool    ggml_is_quantized(enum ggml_type type);

// 将 ggml_ftype 转换为 ggml_type，暂时用于模型加载
GGML_API enum ggml_type ggml_ftype_to_ggml_type(enum ggml_ftype ftype);

// 判断张量是否被转置
GGML_API bool ggml_is_transposed(const struct ggml_tensor * tensor);
// 判断张量是否是连续的
GGML_API bool ggml_is_contiguous(const struct ggml_tensor * tensor);
// 判断张量是否被置换
GGML_API bool ggml_is_permuted  (const struct ggml_tensor * tensor);

// 判断两个张量是否具有相同的形状
GGML_API bool ggml_are_same_shape(const struct ggml_tensor * t0, const struct ggml_tensor * t1);

// 用于计算张量的内存开销
GGML_API size_t ggml_tensor_overhead(void);

// 主函数
# 初始化 GGML 上下文，返回上下文结构体指针
GGML_API struct ggml_context * ggml_init(struct ggml_init_params params);

# 释放 GGML 上下文，接受上下文结构体指针作为参数
GGML_API void ggml_free(struct ggml_context * ctx);

# 返回 GGML 上下文使用的内存大小，接受上下文结构体指针作为参数
GGML_API size_t ggml_used_mem(const struct ggml_context * ctx);

# 设置 GGML 上下文的临时内存，接受上下文结构体指针和临时内存结构体作为参数
GGML_API size_t ggml_set_scratch(struct ggml_context * ctx, struct ggml_scratch scratch);

# 获取 GGML 上下文是否允许分配内存，接受上下文结构体指针作为参数
GGML_API bool ggml_get_no_alloc(struct ggml_context * ctx);

# 设置 GGML 上下文是否允许分配内存，接受上下文结构体指针和布尔值作为参数
GGML_API void ggml_set_no_alloc(struct ggml_context * ctx, bool no_alloc);

# 获取 GGML 上下文的内存缓冲区，接受上下文结构体指针作为参数
GGML_API void * ggml_get_mem_buffer(const struct ggml_context * ctx);

# 获取 GGML 上下文的内存大小，接受上下文结构体指针作为参数
GGML_API size_t ggml_get_mem_size(const struct ggml_context * ctx);

# 获取 GGML 上下文的最大张量大小，接受上下文结构体指针作为参数
GGML_API size_t ggml_get_max_tensor_size(const struct ggml_context * ctx);

# 创建新的张量，接受上下文结构体指针、张量类型、维度数量和维度数组作为参数
GGML_API struct ggml_tensor * ggml_new_tensor(
        struct ggml_context * ctx,
        enum ggml_type type,
        int n_dims,
        const int64_t *ne);
# 创建一个一维张量，返回一个指向该张量的指针
GGML_API struct ggml_tensor * ggml_new_tensor_1d(
        struct ggml_context * ctx,  # 上下文环境指针
        enum   ggml_type type,       # 张量的数据类型
        int64_t ne0);                # 张量的大小

# 创建一个二维张量，返回一个指向该张量的指针
GGML_API struct ggml_tensor * ggml_new_tensor_2d(
        struct ggml_context * ctx,  # 上下文环境指针
        enum   ggml_type type,       # 张量的数据类型
        int64_t ne0,                 # 第一维的大小
        int64_t ne1);                # 第二维的大小

# 创建一个三维张量，返回一个指向该张量的指针
GGML_API struct ggml_tensor * ggml_new_tensor_3d(
        struct ggml_context * ctx,  # 上下文环境指针
        enum   ggml_type type,       # 张量的数据类型
        int64_t ne0,                 # 第一维的大小
        int64_t ne1,                 # 第二维的大小
        int64_t ne2);                # 第三维的大小

# 创建一个四维张量，返回一个指向该张量的指针
GGML_API struct ggml_tensor * ggml_new_tensor_4d(
        struct ggml_context * ctx,  # 上下文环境指针
// 定义一个枚举类型为 ggml_type 的枚举变量 type 和四个 int64_t 类型的变量 ne0, ne1, ne2, ne3
enum ggml_type type,
int64_t ne0,
int64_t ne1,
int64_t ne2,
int64_t ne3);

// 创建一个新的 ggml_tensor 结构体指针，其值为 int32_t 类型的 value
GGML_API struct ggml_tensor * ggml_new_i32(struct ggml_context * ctx, int32_t value);

// 创建一个新的 ggml_tensor 结构体指针，其值为 float 类型的 value
GGML_API struct ggml_tensor * ggml_new_f32(struct ggml_context * ctx, float value);

// 复制一个 ggml_tensor 结构体，返回一个新的 ggml_tensor 结构体指针
GGML_API struct ggml_tensor * ggml_dup_tensor (struct ggml_context * ctx, const struct ggml_tensor * src);

// 创建一个指向现有 ggml_tensor 结构体的视图，返回一个新的 ggml_tensor 结构体指针
GGML_API struct ggml_tensor * ggml_view_tensor(struct ggml_context * ctx, struct ggml_tensor * src);

// 获取给定上下文中的第一个 ggml_tensor 结构体指针
GGML_API struct ggml_tensor * ggml_get_first_tensor(struct ggml_context * ctx);

// 获取给定上下文中的下一个 ggml_tensor 结构体指针
GGML_API struct ggml_tensor * ggml_get_next_tensor (struct ggml_context * ctx, struct ggml_tensor * tensor);

// 根据给定的名称获取给定上下文中的 ggml_tensor 结构体指针
GGML_API struct ggml_tensor * ggml_get_tensor(struct ggml_context * ctx, const char * name);

// 将给定 ggml_tensor 结构体的值设置为零，返回修改后的 ggml_tensor 结构体指针
GGML_API struct ggml_tensor * ggml_set_zero(struct ggml_tensor * tensor);

// 将给定 ggml_tensor 结构体的值设置为 int32_t 类型的 value，返回修改后的 ggml_tensor 结构体指针
GGML_API struct ggml_tensor * ggml_set_i32 (struct ggml_tensor * tensor, int32_t value);

// 将给定 ggml_tensor 结构体的值设置为 float 类型的 value，返回修改后的 ggml_tensor 结构体指针
GGML_API struct ggml_tensor * ggml_set_f32 (struct ggml_tensor * tensor, float value);
// 将一个平面索引转换为坐标
GGML_API void ggml_unravel_index(const struct ggml_tensor * tensor, int64_t i, int64_t * i0, int64_t * i1, int64_t * i2, int64_t * i3);

// 获取一维整数数组中的元素
GGML_API int32_t ggml_get_i32_1d(const struct ggml_tensor * tensor, int i);
// 设置一维整数数组中的元素
GGML_API void ggml_set_i32_1d(const struct ggml_tensor * tensor, int i, int32_t value);

// 获取多维整数数组中的元素
GGML_API int32_t ggml_get_i32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3);
// 设置多维整数数组中的元素
GGML_API void ggml_set_i32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3, int32_t value);

// 获取一维浮点数数组中的元素
GGML_API float ggml_get_f32_1d(const struct ggml_tensor * tensor, int i);
// 设置一维浮点数数组中的元素
GGML_API void ggml_set_f32_1d(const struct ggml_tensor * tensor, int i, float value);

// 获取多维浮点数数组中的元素
GGML_API float ggml_get_f32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3);
// 设置多维浮点数数组中的元素
GGML_API void ggml_set_f32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3, float value);

// 获取数据指针
GGML_API void * ggml_get_data(const struct ggml_tensor * tensor);
// 获取浮点数数据指针
GGML_API float * ggml_get_data_f32(const struct ggml_tensor * tensor);
// 获取整数数据指针
GGML_API int32_t * ggml_get_data_i32(const struct ggml_tensor * tensor);
# 获取给定张量的一元操作
GGML_API enum ggml_unary_op ggml_get_unary_op(const struct ggml_tensor * tensor);

# 获取给定张量的名称
GGML_API const char *         ggml_get_name   (const struct ggml_tensor * tensor);
# 设置给定张量的名称
GGML_API struct ggml_tensor * ggml_set_name   (      struct ggml_tensor * tensor, const char * name);
# 格式化给定张量的名称
GGML_ATTRIBUTE_FORMAT(2, 3)
GGML_API struct ggml_tensor * ggml_format_name(      struct ggml_tensor * tensor, const char * fmt, ...);

# 设置给定张量的后端
GGML_API void ggml_set_backend(struct ggml_tensor * tensor, enum ggml_backend_type backend);


# 张量的反向传播操作
GGML_API struct ggml_tensor * ggml_dup(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

# 在原地进行操作，返回视图(a)
GGML_API struct ggml_tensor * ggml_dup_inplace(
# 定义一个函数，用于将两个张量相加并返回结果
GGML_API struct ggml_tensor * ggml_add(
        struct ggml_context * ctx,  # 上下文对象指针
        struct ggml_tensor  * a,     # 第一个张量指针
        struct ggml_tensor  * b);    # 第二个张量指针

# 定义一个函数，用于将两个张量相加并返回结果，同时指定索引
GGML_API struct ggml_tensor *ggml_add_idx(
        struct ggml_context *ctx,    # 上下文对象指针
        struct ggml_tensor *a,       # 第一个张量指针
        struct ggml_tensor *b,       # 第二个张量指针
        struct ggml_tensor *idx);    # 索引张量指针

# 定义一个函数，用于将两个张量相加并将结果存储在第一个张量中
GGML_API struct ggml_tensor * ggml_add_inplace(
        struct ggml_context * ctx,  # 上下文对象指针
        struct ggml_tensor  * a,     # 第一个张量指针
        struct ggml_tensor  * b);    # 第二个张量指针

# 定义一个函数，用于将两个张量相加并返回结果，同时进行类型转换
GGML_API struct ggml_tensor * ggml_add_cast(
# 定义一个函数，用于执行某种操作，接受一个上下文对象、两个张量对象和一个枚举类型作为参数
GGML_API struct ggml_tensor * ggml_add(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            enum   ggml_type      type);

# 定义一个函数，用于执行某种操作，接受一个上下文对象和两个张量对象作为参数
GGML_API struct ggml_tensor * ggml_add1(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

# 定义一个函数，用于执行某种操作，接受一个上下文对象和两个张量对象作为参数
GGML_API struct ggml_tensor * ggml_add1_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

# 定义一个函数，用于执行某种操作，接受一个上下文对象、两个张量对象和一个整数作为参数
GGML_API struct ggml_tensor * ggml_acc(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                nb1,
# 定义一个函数，用于在给定上下文中对两个张量进行加法操作，并返回结果张量
# 参数包括上下文、两个张量、以及操作数的数量和偏移量
GGML_API struct ggml_tensor * ggml_acc_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        size_t                nb1,
        size_t                nb2,
        size_t                nb3,
        size_t                offset);

# 定义一个函数，用于在给定上下文中对两个张量进行原地加法操作，并返回结果张量
# 参数包括上下文、两个张量、以及操作数的数量和偏移量
GGML_API struct ggml_tensor * ggml_acc_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        size_t                nb1,
        size_t                nb2,
        size_t                nb3,
        size_t                offset);

# 定义一个函数，用于在给定上下文中对两个张量进行减法操作，并返回结果张量
# 参数包括上下文、两个张量
GGML_API struct ggml_tensor * ggml_sub(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b);

# 定义一个函数，用于在给定上下文中对两个张量进行原地减法操作，并返回结果张量
# 参数包括上下文、两个张量
GGML_API struct ggml_tensor * ggml_sub_inplace(
        struct ggml_context * ctx,
# 定义一个函数，用于计算两个张量的乘积
GGML_API struct ggml_tensor * ggml_mul(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b);

# 定义一个函数，用于计算两个张量的乘积，并返回新的张量
GGML_API struct ggml_tensor * ggml_mul_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b);

# 定义一个函数，用于计算两个张量的除法，并返回新的张量
GGML_API struct ggml_tensor * ggml_div(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b);

# 定义一个函数，用于计算两个张量的除法，并将结果存储在第一个张量中
GGML_API struct ggml_tensor * ggml_div_inplace(
        struct ggml_context * ctx,
# 定义一个函数，计算两个张量的乘积
struct ggml_tensor * ggml_mul(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b);

# 定义一个函数，计算张量的平方
struct ggml_tensor * ggml_sqr(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

# 定义一个函数，计算张量的平方并覆盖原始张量
struct ggml_tensor * ggml_sqr_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

# 定义一个函数，计算张量的平方根
struct ggml_tensor * ggml_sqrt(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

# 定义一个函数，计算张量的平方根并覆盖原始张量
struct ggml_tensor * ggml_sqrt_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

# 定义一个函数，计算张量的自然对数
// 使用上下文和张量作为输入，计算对数并将结果存储在输入张量中
GGML_API struct ggml_tensor * ggml_log_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

// 返回标量
GGML_API struct ggml_tensor * ggml_sum(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

// 沿着行求和，对于输入形状为 [a,b,c,d] 的张量，返回形状为 [1,b,c,d]
GGML_API struct ggml_tensor * ggml_sum_rows(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

// 沿着行求平均值
GGML_API struct ggml_tensor * ggml_mean(
        struct ggml_context * ctx,
// 对输入的张量进行 argmax 操作，沿着行的方向
GGML_API struct ggml_tensor * ggml_argmax(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

// 如果 a 和 b 的形状相同，并且 a 不是参数，则返回 a
// 否则，返回一个新的张量：将 a 重复以适应 b 的形状
GGML_API struct ggml_tensor * ggml_repeat(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b);

// 将 a 中的重复项求和，使其形状与 b 相同
GGML_API struct ggml_tensor * ggml_repeat_back(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b);
// 在维度2上连接a和b
// 在稳定扩散中使用
GGML_API struct ggml_tensor * ggml_concat(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b);

// 返回a的绝对值
GGML_API struct ggml_tensor * ggml_abs(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

// 就地计算a的绝对值
GGML_API struct ggml_tensor * ggml_abs_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

// 返回a的符号函数
GGML_API struct ggml_tensor * ggml_sgn(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

// 就地计算a的符号函数
GGML_API struct ggml_tensor * ggml_sgn_inplace(
# 定义一个函数，对输入的张量进行取负操作，返回一个新的张量
GGML_API struct ggml_tensor * ggml_neg(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

# 定义一个函数，对输入的张量进行取负操作，将结果存储在原始张量中，并返回原始张量
GGML_API struct ggml_tensor * ggml_neg_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

# 定义一个函数，对输入的张量进行步进操作，返回一个新的张量
GGML_API struct ggml_tensor * ggml_step(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

# 定义一个函数，对输入的张量进行步进操作，将结果存储在原始张量中，并返回原始张量
GGML_API struct ggml_tensor * ggml_step_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

# 定义一个函数，对输入的张量进行双曲正切操作，返回一个新的张量
GGML_API struct ggml_tensor * ggml_tanh(
# 定义一个函数，接受一个上下文和一个张量作为参数，返回一个张量
struct ggml_tensor * ggml_tanh(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

# 定义一个函数，接受一个上下文和一个张量作为参数，返回一个张量，原地操作
struct ggml_tensor * ggml_tanh_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

# 定义一个函数，接受一个上下文和一个张量作为参数，返回一个张量
struct ggml_tensor * ggml_elu(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

# 定义一个函数，接受一个上下文和一个张量作为参数，返回一个张量，原地操作
struct ggml_tensor * ggml_elu_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

# 定义一个函数，接受一个上下文和一个张量作为参数，返回一个张量
struct ggml_tensor * ggml_relu(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

# 定义一个函数，接受一个上下文和一个张量作为参数，返回一个张量，使用leaky ReLU激活函数
struct ggml_tensor * ggml_leaky(
// 定义一个函数，接受一个上下文和一个张量作为参数，返回一个张量
GGML_API struct ggml_tensor * ggml_relu(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

// 定义一个函数，接受一个上下文和一个张量作为参数，返回一个张量，原地修改输入张量
GGML_API struct ggml_tensor * ggml_relu_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

// 定义一个函数，接受一个上下文和一个张量作为参数，返回一个张量，计算 GELU 函数
// TODO: 双重检查这个计算是否正确
GGML_API struct ggml_tensor * ggml_gelu(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

// 定义一个函数，接受一个上下文和一个张量作为参数，返回一个张量，计算 GELU 函数并原地修改输入张量
GGML_API struct ggml_tensor * ggml_gelu_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

// 定义一个函数，接受一个上下文和一个张量作为参数，返回一个张量，快速计算 GELU 函数
GGML_API struct ggml_tensor * ggml_gelu_quick(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);
// 定义一个函数，实现快速gelu激活函数的操作，对输入的张量进行操作
GGML_API struct ggml_tensor * ggml_gelu_quick_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

// 定义一个函数，实现silu激活函数的操作，对输入的张量进行操作
GGML_API struct ggml_tensor * ggml_silu(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

// 定义一个函数，实现inplace版本的silu激活函数的操作，对输入的张量进行操作
GGML_API struct ggml_tensor * ggml_silu_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

// 定义一个函数，实现silu激活函数的反向传播操作，对输入的张量进行操作
// 参数a表示输入张量，参数b表示梯度
GGML_API struct ggml_tensor * ggml_silu_back(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b);

// 沿着行方向进行归一化操作
# 定义一个名为 ggml_norm 的函数，接受一个上下文结构体指针和一个张量指针作为参数，以及一个浮点数 eps，返回一个张量指针
GGML_API struct ggml_tensor * ggml_norm(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        float                 eps);

# 定义一个名为 ggml_norm_inplace 的函数，接受一个上下文结构体指针和一个张量指针作为参数，以及一个浮点数 eps，返回一个张量指针
GGML_API struct ggml_tensor * ggml_norm_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        float                 eps);

# 定义一个名为 ggml_rms_norm 的函数，接受一个上下文结构体指针和一个张量指针作为参数，以及一个浮点数 eps，返回一个张量指针
GGML_API struct ggml_tensor * ggml_rms_norm(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        float                 eps);

# 定义一个名为 ggml_rms_norm_inplace 的函数，接受一个上下文结构体指针和一个张量指针作为参数，以及一个浮点数 eps，返回一个张量指针
GGML_API struct ggml_tensor * ggml_rms_norm_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        float                 eps);
// 在ne0*ne1*n_groups维度上进行分组归一化
// 在稳定扩散中使用
// TODO: 目前eps硬编码为1e-6
GGML_API struct ggml_tensor * ggml_group_norm(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   n_groups);

GGML_API struct ggml_tensor * ggml_group_norm_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   n_groups);

// a - x
// b - dy
GGML_API struct ggml_tensor * ggml_rms_norm_back(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        float                 eps);
// 两个矩阵相乘，其中矩阵A有k列，n行，矩阵B有k列，m行（内部进行了转置），结果是n列，m行的矩阵
GGML_API struct ggml_tensor * ggml_mul_mat(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b);

// 两个矩阵相乘，其中矩阵A有k列，n行，矩阵B有k列，m行（内部进行了转置），结果是n列，m行的矩阵
GGML_API struct ggml_tensor *ggml_mul_mat_idx(
        struct ggml_context *ctx,
        struct ggml_tensor *a,
        struct ggml_tensor *b,
        struct ggml_tensor *idx,
        struct ggml_tensor *d);

// 两个矩阵相乘，其中矩阵A有k列，n行，矩阵B有k列，m行（内部进行了转置），结果是n列，m行的矩阵
GGML_API struct ggml_tensor *ggml_mul_mat_special(
        struct ggml_context *ctx,
        struct ggml_tensor *a,
        struct ggml_tensor *b,
        struct ggml_tensor *idx,
        struct ggml_tensor *d,
// 传入参数为上下文、张量a、张量b、张量c、张量d，返回一个张量
GGML_API struct ggml_tensor *ggml_axpy(
        struct ggml_context *ctx,
        struct ggml_tensor *a,
        struct ggml_tensor *b,
        struct ggml_tensor *c,
        struct ggml_tensor *d);

// 张量A: m列，n行，张量B: p列，n行，返回结果为m列，p行的张量
GGML_API struct ggml_tensor *ggml_out_prod(
        struct ggml_context *ctx,
        struct ggml_tensor *a,
        struct ggml_tensor *b);

// 在没有反向传播的情况下对张量进行操作
// 使用 GGML_API 声明一个函数，该函数接受一个上下文指针和两个张量指针作为参数，并返回一个张量指针
struct ggml_tensor * ggml_scale(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b);

// 在原地操作，返回视图(a)
struct ggml_tensor * ggml_scale_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b);

// 将b设置为view(a,offset,nb1,nb2,3)，返回修改后的a
struct ggml_tensor * ggml_set(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        size_t                nb1,
        size_t                nb2,
        size_t                nb3,
        size_t                offset);
// 使用 ggml_set_inplace 函数将 b 的部分数据复制到 a 中，并返回 a
GGML_API struct ggml_tensor * ggml_set_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        size_t                nb1,
        size_t                nb2,
        size_t                nb3,
        size_t                offset);

// 使用 ggml_set_1d 函数将 b 的数据复制到 a 的指定位置，并返回 a
GGML_API struct ggml_tensor * ggml_set_1d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        size_t                offset);

// 使用 ggml_set_1d_inplace 函数将 b 的数据复制到 a 的指定位置，并返回 a
GGML_API struct ggml_tensor * ggml_set_1d_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
// 将张量 b 视图化为张量 a 的子集，返回修改后的张量 a
GGML_API struct ggml_tensor * ggml_set_2d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        size_t                nb1,
        size_t                offset);

// 将张量 b 视图化为张量 a 的子集，返回视图化后的张量 a
GGML_API struct ggml_tensor * ggml_set_2d_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        size_t                nb1,
        size_t                offset);

// 将张量 a 视图化为张量 b，返回视图化后的张量 b
// 复制张量 a 到张量 b，返回张量 b 的视图
GGML_API struct ggml_tensor * ggml_cpy(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b);

// 将张量 a 复制到张量 b，原地操作，返回张量 b 的视图
GGML_API struct ggml_tensor * ggml_cpy_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b);

// 使张量 a 连续存储
GGML_API struct ggml_tensor * ggml_cont(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

// 使张量 a 连续存储，原地操作
GGML_API struct ggml_tensor * ggml_cont_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);
// 将张量变为连续存储，并指定新的形状
GGML_API struct ggml_tensor * ggml_cont_1d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0);

GGML_API struct ggml_tensor * ggml_cont_2d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0,
        int64_t               ne1);

GGML_API struct ggml_tensor * ggml_cont_3d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0,
        int64_t               ne1,
        int64_t               ne2);
// 定义一个函数，接受一个上下文指针、一个张量指针和四个整数参数，返回一个张量指针
GGML_API struct ggml_tensor * ggml_cont_4d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0,
        int64_t               ne1,
        int64_t               ne2,
        int64_t               ne3);

// 返回一个视图(a)，b指定新的形状
// TODO: 当我们开始计算梯度时，不要返回视图，而是返回一个副本
GGML_API struct ggml_tensor * ggml_reshape(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b);

// 返回一个视图(a)
// TODO: 当我们开始计算梯度时，不要返回视图，而是返回一个副本
GGML_API struct ggml_tensor * ggml_reshape_1d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
    // 重塑输入张量为一维张量
    GGML_API struct ggml_tensor * ggml_reshape_1d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0);

    // 重塑输入张量为二维张量
    GGML_API struct ggml_tensor * ggml_reshape_2d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1);

    // 返回输入张量的视图
    // TODO: 当我们开始计算梯度时，不要返回视图，而是返回副本
    GGML_API struct ggml_tensor * ggml_reshape_3d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1,
            int64_t               ne2);

    // 重塑输入张量为四维张量
    GGML_API struct ggml_tensor * ggml_reshape_4d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
// 定义一个函数，用于创建一个一维视图的张量
// 参数包括上下文，原始张量，以及张量的维度和偏移量
GGML_API struct ggml_tensor * ggml_view_1d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0,
        size_t                offset);

// 定义一个函数，用于创建一个二维视图的张量
// 参数包括上下文，原始张量，以及张量的维度、行步长和偏移量
GGML_API struct ggml_tensor * ggml_view_2d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int64_t               ne0,
        int64_t               ne1,
        size_t                nb1, // 行步长（以字节为单位）
        size_t                offset);
// 定义一个名为 ggml_view_3d 的函数，返回一个指向 ggml_tensor 结构体的指针
// 该函数接受一个指向 ggml_context 结构体的指针，一个指向 ggml_tensor 结构体的指针，以及四个整型参数 ne0, ne1, ne2, offset
// 还有三个 size_t 类型的参数 nb1, nb2, offset，分别表示字节中的行步长、切片步长和偏移量

// 定义一个名为 ggml_view_4d 的函数，返回一个指向 ggml_tensor 结构体的指针
// 该函数接受一个指向 ggml_context 结构体的指针，一个指向 ggml_tensor 结构体的指针，以及五个整型参数 ne0, ne1, ne2, ne3
// 还有三个 size_t 类型的参数 nb1, nb2, nb3，分别表示字节中的行步长、切片步长和第四维度的步长
// 定义一个函数，用于对张量进行维度重排
GGML_API struct ggml_tensor * ggml_permute(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   axis0,
        int                   axis1,
        int                   axis2,
        int                   axis3);

// 定义一个函数，用于对张量进行转置操作，即维度重排为(1, 0, 2, 3)
// 实际调用了 ggml_permute 函数
GGML_API struct ggml_tensor * ggml_transpose(
        struct ggml_context * ctx,
        struct ggml_tensor  * a);

// 定义一个函数，用于从张量 a 中获取指定的行，结果存储在张量 b 中
GGML_API struct ggml_tensor * ggml_get_rows(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b);
// 从给定的张量中获取指定行的数据，并返回结果张量
GGML_API struct ggml_tensor * ggml_get_rows_back(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        struct ggml_tensor  * c);

// 返回一个对角线元素为给定张量的对角线元素的张量
GGML_API struct ggml_tensor * ggml_diag(
    struct ggml_context     * ctx,
    struct ggml_tensor      * a);

// 将对角线上方的元素设置为负无穷大
GGML_API struct ggml_tensor * ggml_diag_mask_inf(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   n_past);

// 就地操作，返回视图(a)
GGML_API struct ggml_tensor * ggml_diag_mask_inf_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
    // 声明一个函数，将对角线以上的元素设为0
    GGML_API struct ggml_tensor * ggml_diag_mask_zero(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_past);

    // 声明一个函数，对输入的张量进行对角线以上的元素设为0的操作，返回操作后的张量
    GGML_API struct ggml_tensor * ggml_diag_mask_zero_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_past);

    // 声明一个函数，对输入的张量进行 softmax 操作，返回操作后的张量
    GGML_API struct ggml_tensor * ggml_soft_max(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // 声明一个函数，对输入的张量进行 softmax 操作，返回操作后的张量的视图
    GGML_API struct ggml_tensor * ggml_soft_max_inplace(
// 定义一个函数，计算softmax函数的反向传播
GGML_API struct ggml_tensor * ggml_soft_max_back(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b);

// 定义一个函数，in-place方式计算softmax函数的反向传播，并返回a的视图
GGML_API struct ggml_tensor * ggml_soft_max_back_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b);

// 旋转位置嵌入
// 如果mode & 1 == 1，跳过n_past个元素（已弃用）
// 如果mode & 2 == 1，GPT-NeoX风格
// 如果mode & 4 == 1，ChatGLM风格
//
// b是一个大小为a->ne[2]的int32向量，它包含位置信息
// 定义一个名为 ggml_rope 的函数，接受一个上下文结构体指针，两个张量指针，以及三个整数参数，返回一个张量指针
GGML_API struct ggml_tensor * ggml_rope(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                   n_dims,
        int                   mode,
        int                   n_ctx);

// 定义一个名为 ggml_rope_inplace 的函数，接受一个上下文结构体指针，两个张量指针，以及三个整数参数，返回一个张量指针，表示在原地操作，返回视图(a)
GGML_API struct ggml_tensor * ggml_rope_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                   n_dims,
        int                   mode,
        int                   n_ctx);

// 定义一个名为 ggml_rope_custom 的函数，接受一个上下文结构体指针，返回一个张量指针，表示自定义的 RoPE
GGML_API struct ggml_tensor * ggml_rope_custom(
        struct ggml_context * ctx,
// 定义一个函数ggml_rope_custom_inplace，接受多个参数
// 参数a：指向ggml_tensor结构体的指针
// 参数b：指向ggml_tensor结构体的指针
// 参数n_dims：整数，表示张量的维度
// 参数mode：整数，表示模式
// 参数n_ctx：整数，表示上下文的数量
// 参数n_orig_ctx：整数，表示原始上下文的数量
// 参数freq_base：浮点数，表示基础频率
// 参数freq_scale：浮点数，表示频率缩放
// 参数ext_factor：浮点数，表示扩展因子
// 参数attn_factor：浮点数，表示注意力因子
// 参数beta_fast：浮点数，表示快速beta值
// 参数beta_slow：浮点数，表示慢速beta值
// 返回一个指向ggml_tensor结构体的指针，指向a的视图
GGML_API struct ggml_tensor * ggml_rope_custom_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                   n_dims,
        int                   mode,
// 定义一个函数，接受多个参数，包括整数、浮点数等，用于计算 YaRN RoPE 缩放的修正维度
void ggml_rope_yarn_corr_dims(
    int n_dims, int n_orig_ctx, float freq_base, float beta_fast, float beta_slow, float dims[2]);

// 定义一个函数，接受多个参数，包括上下文、张量等，用于在原地计算 xPos RoPE，并返回视图(a)
GGML_API struct ggml_tensor * ggml_rope_xpos_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                   n_dims,
        float                 base,
        // ...
```
这段代码是C语言的函数声明部分，包括了两个函数的声明。第一个函数用于计算YaRN RoPE缩放的修正维度，接受多个整数和浮点数作为参数。第二个函数用于在原地计算xPos RoPE，并返回视图(a)，同样接受多个参数，包括上下文和张量等。
// 定义一个函数，用于计算旋转位置嵌入的反向传播，即根据dy计算dx
// ctx - 上下文环境
// a - dy张量
// b - 另一个张量
// n_dims - 张量维度
// mode - 模式
// n_ctx - 上下文数量
// n_orig_ctx - 原始上下文数量
// freq_base - 频率基数
// freq_scale - 频率比例
// ext_factor - 扩展因子
// attn_factor - 注意力因子
// beta_fast - 快速beta值
// beta_slow - 慢速beta值
// xpos_base - 位置基数
// xpos_down - 是否向下旋转位置
GGML_API struct ggml_tensor * ggml_rope_back(
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
        bool                  xpos_down);
    // alibi position embedding
    // 在原地操作，返回视图(a)
    GGML_API struct ggml_tensor * ggml_alibi(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_past,
            int                   n_head,
            float                 bias_max);

    // clamp
    // 在原地操作，返回视图(a)
    GGML_API struct ggml_tensor * ggml_clamp(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            float                 min,
            float                 max);

    // 将图像转换为列
    GGML_API struct ggml_tensor * ggml_im2col(
            struct ggml_context * ctx,
```
// 定义一个函数 ggml_conv_1d，用于进行一维卷积操作
// 参数包括输入张量 a，卷积核张量 b，步长 s0，填充 p0，膨胀系数 d0，以及是否为二维卷积的标志 is_2D
GGML_API struct ggml_tensor * ggml_conv_1d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                   s0,  // 步长
        int                   p0,  // 填充
        int                   d0); // 膨胀

// conv_1d 函数的一个变种，使用填充为输入张量长度的一半
// 是 ggml_conv_1d(a, b, s, a->ne[0]/2, d) 的别名
# 定义一个名为 ggml_conv_1d_ph 的函数，接受上下文、两个张量、步长和填充值作为参数，返回一个张量指针
GGML_API struct ggml_tensor* ggml_conv_1d_ph(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                   s,
        int                   d);

# 定义一个名为 ggml_conv_transpose_1d 的函数，接受上下文、两个张量、步长、填充值和输出扩展因子作为参数，返回一个张量指针
GGML_API struct ggml_tensor * ggml_conv_transpose_1d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                   s0,
        int                   p0,
        int                   d0);

# 定义一个名为 ggml_conv_2d 的函数，接受上下文、两个张量、步长和填充值作为参数，返回一个张量指针
GGML_API struct ggml_tensor * ggml_conv_2d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                   s0,
// 定义一个函数，参数包括输入张量a、卷积核张量b，返回卷积后的张量
GGML_API struct ggml_tensor * ggml_conv_2d_sk_p0(
        struct ggml_context * ctx, // 上下文结构体指针
        struct ggml_tensor  * a,   // 输入张量指针
        struct ggml_tensor  * b);  // 卷积核张量指针
// 定义一个函数，用于实现 2D 卷积操作，stride 为 1，padding 为 half
// 该函数接受三个参数：上下文结构体指针，输入张量 a，输入张量 b
// 返回一个指向 ggml_tensor 结构体的指针
// 该函数在 sam 模块中被使用
GGML_API struct ggml_tensor * ggml_conv_2d_s1_ph(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b);

// 定义一个函数，用于实现 2D 转置卷积操作，padding 为 0
// 该函数接受四个参数：上下文结构体指针，输入张量 a，输入张量 b，步长 stride
// 返回一个指向 ggml_tensor 结构体的指针
GGML_API struct ggml_tensor * ggml_conv_transpose_2d_p0(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b,
        int                   stride);

// 定义一个枚举类型 ggml_op_pool
    // 定义了一个枚举类型，表示池化操作的最大值、平均值和计数
    GGML_OP_POOL_MAX,
    GGML_OP_POOL_AVG,
    GGML_OP_POOL_COUNT,
};

// 定义了一个函数，用于对输入的一维张量进行池化操作
GGML_API struct ggml_tensor * ggml_pool_1d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        enum ggml_op_pool     op,
        int                   k0, // 表示核大小
        int                   s0, // 表示步长
        int                   p0); // 表示填充

// 对于结果张量的第一维度，将会有2*p0的填充
// 对于结果张量的第二维度，将会有2*p1的填充
GGML_API struct ggml_tensor * ggml_pool_2d(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        enum ggml_op_pool     op,
        int                   k0,
    // 定义函数，参数为 k1, s0, s1, p0, p1，返回类型为 int
    int k1,
    int s0,
    int s1,
    float p0,
    float p1);

    // 最近邻插值，用于稳定扩散
    GGML_API struct ggml_tensor * ggml_upscale(
            struct ggml_context * ctx,  // 上下文对象指针
            struct ggml_tensor  * a,    // 输入张量指针
            int                   scale_factor);  // 缩放因子

    GGML_API struct ggml_tensor * ggml_flash_attn(
            struct ggml_context * ctx,  // 上下文对象指针
            struct ggml_tensor  * q,    // 查询张量指针
            struct ggml_tensor  * k,    // 键张量指针
            struct ggml_tensor  * v,    // 值张量指针
            bool                  masked);  // 是否使用掩码
# 定义一个名为ggml_flash_attn_back的函数，接受ggml_context结构体指针ctx，ggml_tensor结构体指针q、k、v、d，以及一个布尔类型的masked参数，并返回一个ggml_tensor结构体指针
GGML_API struct ggml_tensor * ggml_flash_attn_back(
       struct ggml_context * ctx,
       struct ggml_tensor  * q,
       struct ggml_tensor  * k,
       struct ggml_tensor  * v,
       struct ggml_tensor  * d,
       bool                  masked);

# 定义一个名为ggml_flash_ff的函数，接受ggml_context结构体指针ctx，ggml_tensor结构体指针a、b0、b1、c0、c1， 并返回一个ggml_tensor结构体指针
GGML_API struct ggml_tensor * ggml_flash_ff(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        struct ggml_tensor  * b0,
        struct ggml_tensor  * b1,
        struct ggml_tensor  * c0,
        struct ggml_tensor  * c1);

# 将输入数据分割成不重叠的窗口，如果需要则进行填充
# 例如：
# a:   768   64   64    1
# w:    14
// 定义一个名为 ggml_win_part 的函数，用于在给定上下文中对张量进行窗口切割操作，返回切割后的张量
// 该函数在 sam 中被调用
GGML_API struct ggml_tensor * ggml_win_part(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   w);

// 定义一个名为 ggml_win_unpart 的函数，用于在给定上下文中对张量进行窗口合并操作，返回合并后的张量
// 该函数在 sam 中被调用
GGML_API struct ggml_tensor * ggml_win_unpart(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        int                   w0,
        int                   h0,
        int                   w);

// 定义一个名为 ggml_unary 的函数，用于在给定上下文中对张量进行一元操作，返回操作后的张量
GGML_API struct ggml_tensor * ggml_unary(
        struct ggml_context * ctx,
         struct ggml_tensor * a,
         enum ggml_unary_op op);
// 定义了一个函数，对输入的张量进行一元操作，并在原地修改
GGML_API struct ggml_tensor * ggml_unary_inplace(
    struct ggml_context * ctx,
    struct ggml_tensor  * a,
    enum ggml_unary_op op);

// 在sam中使用的函数，用于获取相对位置
GGML_API struct ggml_tensor * ggml_get_rel_pos(
    struct ggml_context * ctx,
    struct ggml_tensor  * a,
    int                   qh,
    int                   kh);

// 在sam中使用的函数，用于添加相对位置
GGML_API struct ggml_tensor * ggml_add_rel_pos(
    struct ggml_context * ctx,
    struct ggml_tensor  * a,
    struct ggml_tensor  * pw,
    struct ggml_tensor  * ph);
// 定义一个函数，用于在上下文中添加相对位置信息，返回一个指向 ggml_tensor 结构的指针
struct ggml_tensor * ggml_add_rel_pos_inplace(
        struct ggml_context * ctx,  // 上下文指针
        struct ggml_tensor  * a,    // 输入张量指针
        struct ggml_tensor  * pw,   // 输入张量指针
        struct ggml_tensor  * ph);  // 输入张量指针

// 自定义操作符

// 定义一个函数指针类型，用于表示对单个浮点数进行操作
typedef void (*ggml_unary_op_f32_t) (const int, float *, const float *);

// 定义一个函数指针类型，用于表示对两个浮点数进行操作
typedef void (*ggml_binary_op_f32_t)(const int, float *, const float *, const float *);

// 定义一个函数指针类型，用于表示自定义操作1
typedef void (*ggml_custom1_op_f32_t)(struct ggml_tensor *, const struct ggml_tensor *);

// 定义一个函数指针类型，用于表示自定义操作2
typedef void (*ggml_custom2_op_f32_t)(struct ggml_tensor *, const struct ggml_tensor *, const struct ggml_tensor *);

// 定义一个函数指针类型，用于表示自定义操作3
typedef void (*ggml_custom3_op_f32_t)(struct ggml_tensor *, const struct ggml_tensor *, const struct ggml_tensor *, const struct ggml_tensor *);

// 弃用的函数，用于对输入张量进行一元浮点数操作
GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_unary_f32(
        struct ggml_context        * ctx,  // 上下文指针
        struct ggml_tensor         * a,    // 输入张量指针
               ggml_unary_op_f32_t   fun), // 一元操作函数指针
# 使用 ggml_map_custom1 代替
GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_unary_f32(
        struct ggml_context        * ctx,
        struct ggml_tensor         * a,
               ggml_unary_op_f32_t   fun),
    "use ggml_map_custom1 instead");

# 使用 ggml_map_custom1_inplace 代替
GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_unary_inplace_f32(
        struct ggml_context        * ctx,
        struct ggml_tensor         * a,
               ggml_unary_op_f32_t   fun),
    "use ggml_map_custom1_inplace instead");

# 使用 ggml_map_custom2 代替
GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_binary_f32(
        struct ggml_context         * ctx,
        struct ggml_tensor          * a,
        struct ggml_tensor          * b,
               ggml_binary_op_f32_t   fun),
    "use ggml_map_custom2 instead");

# 使用 ggml_map_custom2 代替
GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_binary_inplace_f32(
        struct ggml_context         * ctx,
        struct ggml_tensor          * a,
        struct ggml_tensor          * b,
               ggml_binary_op_f32_t   fun),
# 使用 ggml_map_custom2_inplace 替代
GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom2_inplace_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
               ggml_custom2_op_f32_t   fun),
    "use ggml_map_custom2_inplace instead");

# 使用 ggml_map_custom1 替代
GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom1_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
               ggml_custom1_op_f32_t   fun),
    "use ggml_map_custom1 instead");

# 使用 ggml_map_custom1_inplace 替代
GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom1_inplace_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
               ggml_custom1_op_f32_t   fun),
    "use ggml_map_custom1_inplace instead");

# 使用 ggml_map_custom2 替代
GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom2_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
               ggml_custom2_op_f32_t   fun),
    "use ggml_map_custom2 instead");
# 使用 GGML_DEPRECATED 宏标记 ggml_map_custom2_inplace_f32 函数，提示用户使用 ggml_map_custom2_inplace 函数代替
GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom2_inplace_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
               ggml_custom2_op_f32_t   fun),
    "use ggml_map_custom2_inplace instead");

# 使用 GGML_DEPRECATED 宏标记 ggml_map_custom3_f32 函数，提示用户使用 ggml_map_custom3 函数代替
GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom3_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
        struct ggml_tensor           * c,
               ggml_custom3_op_f32_t   fun),
    "use ggml_map_custom3 instead");

# 使用 GGML_DEPRECATED 宏标记 ggml_map_custom3_inplace_f32 函数，提示用户使用 ggml_map_custom3_inplace 函数代替
GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom3_inplace_f32(
        struct ggml_context          * ctx,
        struct ggml_tensor           * a,
        struct ggml_tensor           * b,
    // 定义了三种不同的自定义操作函数指针类型
    typedef void (*ggml_custom1_op_t)(struct ggml_tensor * dst , const struct ggml_tensor * a, int ith, int nth, void * userdata);
    typedef void (*ggml_custom2_op_t)(struct ggml_tensor * dst , const struct ggml_tensor * a, const struct ggml_tensor * b, int ith, int nth, void * userdata);
    typedef void (*ggml_custom3_op_t)(struct ggml_tensor * dst , const struct ggml_tensor * a, const struct ggml_tensor * b, const struct ggml_tensor * c, int ith, int nth, void * userdata);

    // 定义了最大任务数的宏
    #define GGML_N_TASKS_MAX -1

    // 定义了使用自定义操作函数指针的函数 ggml_map_custom1
    GGML_API struct ggml_tensor * ggml_map_custom1(
            struct ggml_context   * ctx,
            struct ggml_tensor    * a,
            ggml_custom1_op_t       fun,
            int                     n_tasks,
            void                  * userdata);

    // 定义了使用自定义操作函数指针的函数 ggml_map_custom1_inplace
    GGML_API struct ggml_tensor * ggml_map_custom1_inplace(
# 定义一个函数，用于将自定义操作应用于一个张量，并返回结果张量
# 参数包括上下文环境ctx，输入张量a，自定义操作fun，任务数量n_tasks，以及用户数据指针
GGML_API struct ggml_tensor * ggml_map_custom1(
            struct ggml_context   * ctx,
            struct ggml_tensor    * a,
            ggml_custom1_op_t       fun,
            int                     n_tasks,
            void                  * userdata);

# 定义一个函数，用于将自定义操作应用于两个张量，并返回结果张量
# 参数包括上下文环境ctx，输入张量a和b，自定义操作fun，任务数量n_tasks，以及用户数据指针
GGML_API struct ggml_tensor * ggml_map_custom2(
            struct ggml_context   * ctx,
            struct ggml_tensor    * a,
            struct ggml_tensor    * b,
            ggml_custom2_op_t       fun,
            int                     n_tasks,
            void                  * userdata);

# 定义一个函数，用于将自定义操作应用于两个张量，并将结果存储在第一个张量中
# 参数包括上下文环境ctx，输入张量a和b，自定义操作fun，任务数量n_tasks，以及用户数据指针
GGML_API struct ggml_tensor * ggml_map_custom2_inplace(
            struct ggml_context   * ctx,
            struct ggml_tensor    * a,
            struct ggml_tensor    * b,
            ggml_custom2_op_t       fun,
            int                     n_tasks,
# 定义一个函数，将三个张量映射到一个自定义操作上，并返回结果张量
GGML_API struct ggml_tensor * ggml_map_custom3(
        struct ggml_context   * ctx,  # 上下文对象
        struct ggml_tensor    * a,    # 第一个输入张量
        struct ggml_tensor    * b,    # 第二个输入张量
        struct ggml_tensor    * c,    # 第三个输入张量
        ggml_custom3_op_t       fun,  # 自定义操作函数指针
        int                     n_tasks,  # 任务数量
        void                  * userdata);  # 用户数据指针
# 定义一个函数，将三个张量映射到一个自定义操作上，并将结果存储在给定的输出张量中
GGML_API struct ggml_tensor * ggml_map_custom3_inplace(
        struct ggml_context   * ctx,  # 上下文对象
        struct ggml_tensor    * a,    # 第一个输入张量
        struct ggml_tensor    * b,    # 第二个输入张量
        struct ggml_tensor    * c,    # 第三个输入张量
        ggml_custom3_op_t       fun,  # 自定义操作函数指针
        int                     n_tasks,  # 任务数量
        void                  * userdata);  # 用户数据指针
    // 定义交叉熵损失函数，计算两个张量之间的交叉熵损失
    GGML_API struct ggml_tensor * ggml_cross_entropy_loss(
            struct ggml_context         * ctx,
            struct ggml_tensor          * a,
            struct ggml_tensor          * b);

    // 定义交叉熵损失函数的反向传播，计算损失函数对输入张量的梯度
    GGML_API struct ggml_tensor * ggml_cross_entropy_loss_back(
            struct ggml_context         * ctx,
            struct ggml_tensor          * a,
            struct ggml_tensor          * b,
            struct ggml_tensor          * c);

    //
    // 自动微分
    //

    // 设置参数张量用于自动微分
    GGML_API void ggml_set_param(
            struct ggml_context * ctx,
            struct ggml_tensor  * tensor);
// 前向传播构建
GGML_API void ggml_build_forward_expand (struct ggml_cgraph * cgraph, struct ggml_tensor * tensor);
// 反向传播构建
GGML_API void ggml_build_backward_expand(struct ggml_context * ctx, struct ggml_cgraph * gf, struct ggml_cgraph * gb, bool keep);

// 在上下文中分配图
GGML_API struct ggml_cgraph * ggml_new_graph         (struct ggml_context * ctx); // 大小 = GGML_DEFAULT_GRAPH_SIZE, grads = false
GGML_API struct ggml_cgraph * ggml_new_graph_custom  (struct ggml_context * ctx, size_t size, bool grads);
GGML_API struct ggml_cgraph * ggml_graph_dup         (struct ggml_context * ctx, struct ggml_cgraph * cgraph);
GGML_API struct ggml_cgraph * ggml_graph_view        (struct ggml_context * ctx, struct ggml_cgraph * cgraph, int i0, int i1);
GGML_API void                 ggml_graph_cpy         (struct ggml_cgraph * src, struct ggml_cgraph * dst);
GGML_API void                 ggml_graph_reset       (struct ggml_cgraph * cgraph);  // 清零梯度
GGML_API void                 ggml_graph_clear       (struct ggml_cgraph * cgraph);

// 计算图的额外开销
GGML_API size_t ggml_graph_overhead(void);
GGML_API size_t ggml_graph_overhead_custom(size_t size, bool grads);

// 在 ggml_graph_compute() 之前必须调用 ggml_graph_plan()
// 当 plan.work_size > 0 时，调用者必须为 plan.work_data 分配内存
GGML_API struct ggml_cplan ggml_graph_plan   (struct ggml_cgraph * cgraph, int n_threads /*= GGML_DEFAULT_N_THREADS*/);
// 计算给定计算图的结果
GGML_API int ggml_graph_compute(struct ggml_cgraph * cgraph, struct ggml_cplan * cplan);

// 与ggml_graph_compute()相同，但工作数据作为上下文的一部分分配
// 注意：此API的缺点是您必须确保上下文具有足够的内存来存储工作数据
GGML_API void ggml_graph_compute_with_ctx(struct ggml_context * ctx, struct ggml_cgraph * cgraph, int n_threads);

// 获取给定计算图中指定名称的张量
GGML_API struct ggml_tensor * ggml_graph_get_tensor(struct ggml_cgraph * cgraph, const char * name);

// 将计算图导出到指定文件中
GGML_API void ggml_graph_export(const struct ggml_cgraph * cgraph, const char * fname);

// 从文件中导入计算图，并返回上下文数据和评估上下文数据
GGML_API struct ggml_cgraph * ggml_graph_import(const char * fname, struct ggml_context ** ctx_data, struct ggml_context ** ctx_eval);

// 打印计算图的信息和性能信息
GGML_API void ggml_graph_print(const struct ggml_cgraph * cgraph);

// 使用dot格式将计算图转储到文件中
GGML_API void ggml_graph_dump_dot(const struct ggml_cgraph * gb, const struct ggml_cgraph * gf, const char * filename);

// 使用提供的检查点为gf构建梯度检查点后向图gb
// gb_tmp将包含具有重写的后向处理节点的原始后向图，但不包含第二次前向传递节点。
    // 定义一个名为 ggml_build_backward_gradient_checkpointing 的函数，用于构建反向梯度检查点
    // 参数包括上下文 ctx，前向计算图 gf，反向计算图 gb，临时反向计算图 gb_tmp，检查点数组 checkpoints，检查点数量 n_checkpoints
    GGML_API void ggml_build_backward_gradient_checkpointing(
            struct ggml_context   * ctx,
            struct ggml_cgraph    * gf,
            struct ggml_cgraph    * gb,
            struct ggml_cgraph    * gb_tmp,
            struct ggml_tensor  * * checkpoints,
            int                     n_checkpoints);
    //
    // optimization
    //

    // 优化方法的枚举类型
    enum ggml_opt_type {
        GGML_OPT_ADAM,  // Adam 优化方法
        GGML_OPT_LBFGS, // L-BFGS 优化方法
    };

    // 线搜索方法的枚举类型
    enum ggml_linesearch {
        GGML_LINESEARCH_DEFAULT = 1, // 默认线搜索方法
// 定义线搜索算法类型，分别为Armijo、Wolfe和Strong Wolfe
GGML_LINESEARCH_BACKTRACKING_ARMIJO       = 0,
GGML_LINESEARCH_BACKTRACKING_WOLFE        = 1,
GGML_LINESEARCH_BACKTRACKING_STRONG_WOLFE = 2,

// 优化返回结果枚举类型，包括收敛、未收敛、无上下文、无效Wolfe条件、失败和取消
enum ggml_opt_result {
    GGML_OPT_OK = 0,
    GGML_OPT_DID_NOT_CONVERGE,
    GGML_OPT_NO_CONTEXT,
    GGML_OPT_INVALID_WOLFE,
    GGML_OPT_FAIL,
    GGML_OPT_CANCEL,

    // 线搜索失败的返回结果
    GGML_LINESEARCH_FAIL = -128,
    GGML_LINESEARCH_MINIMUM_STEP,
    GGML_LINESEARCH_MAXIMUM_STEP,
    GGML_LINESEARCH_MAXIMUM_ITERATIONS,
    GGML_LINESEARCH_INVALID_PARAMETERS,
    };

    // 定义一个函数指针类型 ggml_opt_callback，用于接收优化过程中的回调函数
    typedef void (*ggml_opt_callback)(void * data, int accum_step, float * sched, bool * cancel);
    // 定义一个函数指针类型 ggml_log_callback，用于接收日志输出的回调函数
    typedef void (*ggml_log_callback)(enum ggml_log_level level, const char * text, void * user_data);

    // 优化参数
    //
    //   参考 ggml.c (ggml_opt_default_params) 获取默认值
    //
    // 定义一个结构体 ggml_opt_params，用于存储优化参数
    struct ggml_opt_params {
        // 优化类型
        enum ggml_opt_type type;

        // 图的大小
        size_t graph_size;

        // 线程数
        int n_threads;

        // 基于增量的收敛测试
        //
        //   如果 past == 0 - 禁用
        //   如果 past > 0 - 启用
        // 停止条件：如果 |f(x) - f(x_past)| < delta * max(1, |f(x)|) 则停止迭代
        //
        int past; // 上一次迭代的值
        float delta; // 允许的误差范围

        // 最大不改善的迭代次数
        //
        //   如果为0 - 禁用
        //   如果 > 0:
        //     如果在这个迭代次数内没有成本改善，则假定收敛
        //
        int max_no_improvement; // 最大不改善的迭代次数

        bool print_forward_graph; // 是否打印前向图
        bool print_backward_graph; // 是否打印反向图

        int n_gradient_accumulation; // 梯度累积次数

        // ADAM 参数
        struct {
            int n_iter; // 迭代次数

            float sched; // 调度乘数（固定、衰减或预热）
            float decay; // AdamW 的权重衰减，使用 0.0f 禁用
            int   decay_min_ndim; // 应用权重衰减的张量的最小维度
            float alpha; // 学习率
            float beta1;
            float beta2;
            float eps;   // 数值稳定性的 epsilon
            float eps_f; // 收敛测试的 epsilon
            float eps_g; // 收敛测试的 epsilon
            float gclip; // 梯度裁剪
        } adam; // Adam 优化器的参数

        // LBFGS 参数
        struct {
            int m; // 近似逆 Hessian 矩阵的修正次数
            int n_iter; // 迭代次数
            int max_linesearch; // 最大线搜索次数
            float eps;      // 收敛容限
            float ftol;     // 线搜索容限
            float wolfe;    // Wolfe条件参数
            float min_step; // 最小步长
            float max_step; // 最大步长

            enum ggml_linesearch linesearch; // 线搜索类型枚举
        } lbfgs; // LBFGS优化器参数结构体

    };

    struct ggml_opt_context {
        struct ggml_context * ctx; // 上下文指针
        struct ggml_opt_params params; // 优化参数

        int iter; // 迭代次数
        int64_t nx; // 参数元素个数

        bool just_initialized; // 是否刚初始化

        float loss_before; // 优化前的损失值
        // 定义浮点型变量 loss_after，用于存储损失值

        // 定义结构体 adam，包含当前梯度、第一时刻、第二时刻、过去函数值、最佳函数值、上一次函数值、无改善次数
        struct {
            struct ggml_tensor * g;  // 当前梯度
            struct ggml_tensor * m;  // 第一时刻
            struct ggml_tensor * v;  // 第二时刻
            struct ggml_tensor * pf; // 过去函数值
            float fx_best;            // 最佳函数值
            float fx_prev;            // 上一次函数值
            int n_no_improvement;     // 无改善次数
        } adam;

        // 定义结构体，包含当前参数、上一次参数、当前梯度、上一次梯度、搜索方向、过去函数值、L-BFGS 内存 alpha
        struct {
            struct ggml_tensor * x;    // 当前参数
            struct ggml_tensor * xp;   // 上一次参数
            struct ggml_tensor * g;    // 当前梯度
            struct ggml_tensor * gp;   // 上一次梯度
            struct ggml_tensor * d;    // 搜索方向
            struct ggml_tensor * pf;   // 过去函数值
            struct ggml_tensor * lmal; // L-BFGS 内存 alpha
// 定义结构体 lbfgs，用于存储 L-BFGS 算法的相关参数和状态
struct lbfgs {
    struct ggml_tensor * lmys; // 存储 L-BFGS 算法的记忆 ys
    struct ggml_tensor * lms;  // 存储 L-BFGS 算法的记忆 s
    struct ggml_tensor * lmy;  // 存储 L-BFGS 算法的记忆 y
    float fx_best;             // 存储最佳的函数值
    float step;                // 存储步长
    int j;                     // 存储迭代次数
    int k;                     // 存储迭代次数
    int end;                   // 存储结束标志
    int n_no_improvement;      // 存储无改善次数
};

// 默认的优化参数，根据给定的优化类型返回默认的参数
GGML_API struct ggml_opt_params ggml_opt_default_params(enum ggml_opt_type type);

// 优化由张量 f 定义的函数
GGML_API enum ggml_opt_result ggml_opt(
        struct ggml_context * ctx,    // 上下文环境
        struct ggml_opt_params params, // 优化参数
        struct ggml_tensor * f);       // 待优化的张量
// 初始化优化器上下文
GGML_API void ggml_opt_init(
        struct ggml_context     * ctx,  // 上下文对象
        struct ggml_opt_context * opt,  // 优化器上下文对象
        struct ggml_opt_params    params,  // 优化参数
        int64_t                   nx);  // 参数数量

// 继续优化由张量 f 定义的函数
GGML_API enum ggml_opt_result ggml_opt_resume(
        struct ggml_context * ctx,  // 上下文对象
        struct ggml_opt_context * opt,  // 优化器上下文对象
        struct ggml_tensor * f);  // 定义函数的张量

// 继续优化由张量 f 定义的函数
GGML_API enum ggml_opt_result ggml_opt_resume_g(
        struct ggml_context * ctx,  // 上下文对象
        struct ggml_opt_context * opt,  // 优化器上下文对象
        struct ggml_tensor * f,  // 定义函数的张量
        struct ggml_cgraph * gf,  // 函数的梯度
        struct ggml_cgraph * gb,  // 函数的反向梯度
// 定义一个回调函数指针，用于处理量化后的数据
ggml_opt_callback callback,
// 回调函数的参数
void * callback_data);

//
// 量化
//

// TODO: 这些可能会被移除，而使用更通用的ggml_quantize_chunk
// 对输入数据进行Q4.0量化
GGML_API size_t ggml_quantize_q4_0(const float * src, void * dst, int n, int k, int64_t * hist);
// 对输入数据进行Q4.1量化
GGML_API size_t ggml_quantize_q4_1(const float * src, void * dst, int n, int k, int64_t * hist);
// 对输入数据进行Q5.0量化
GGML_API size_t ggml_quantize_q5_0(const float * src, void * dst, int n, int k, int64_t * hist);
// 对输入数据进行Q5.1量化
GGML_API size_t ggml_quantize_q5_1(const float * src, void * dst, int n, int k, int64_t * hist);
// 对输入数据进行Q8.0量化
GGML_API size_t ggml_quantize_q8_0(const float * src, void * dst, int n, int k, int64_t * hist);

// 对输入数据进行Q2_K量化
GGML_API size_t ggml_quantize_q2_K(const float * src, void * dst, int n, int k, int64_t * hist);
// 对输入数据进行Q3_K量化
GGML_API size_t ggml_quantize_q3_K(const float * src, void * dst, int n, int k, int64_t * hist);
// 对输入数据进行Q4_K量化
GGML_API size_t ggml_quantize_q4_K(const float * src, void * dst, int n, int k, int64_t * hist);
// 对输入数据进行Q5_K量化
GGML_API size_t ggml_quantize_q5_K(const float * src, void * dst, int n, int k, int64_t * hist);
// 对输入数据进行Q6_K量化
GGML_API size_t ggml_quantize_q6_K(const float * src, void * dst, int n, int k, int64_t * hist);
// 定义一个函数，用于对输入的数据进行量化处理，返回处理后的数据大小
GGML_API size_t ggml_quantize_chunk(enum ggml_type type, const float * src, void * dst, int start, int n, int64_t * hist);

//
// gguf
//

// 定义一个枚举类型，包含不同的数据类型
enum gguf_type {
    GGUF_TYPE_UINT8   = 0,
    GGUF_TYPE_INT8    = 1,
    GGUF_TYPE_UINT16  = 2,
    GGUF_TYPE_INT16   = 3,
    GGUF_TYPE_UINT32  = 4,
    GGUF_TYPE_INT32   = 5,
    GGUF_TYPE_FLOAT32 = 6,
    GGUF_TYPE_BOOL    = 7,
    GGUF_TYPE_STRING  = 8,
    GGUF_TYPE_ARRAY   = 9,
    GGUF_TYPE_UINT64  = 10,
    GGUF_TYPE_INT64   = 11,
    GGUF_TYPE_FLOAT64 = 12,
```
    // 定义枚举类型 GGUF_TYPE，用于标记枚举的结束
    GGUF_TYPE_COUNT,

    // 定义结构体 gguf_context，用于存储上下文信息
    struct gguf_context;

    // 定义结构体 gguf_init_params，用于初始化参数
    struct gguf_init_params {
        bool no_alloc;  // 是否禁止分配内存

        // 如果不为 NULL，则创建一个 ggml_context 并在其中分配张量数据
        struct ggml_context ** ctx;
    };

    // 初始化空的 gguf_context 对象
    GGML_API struct gguf_context * gguf_init_empty(void);

    // 初始化空的稀疏 gguf_context 对象
    GGML_API struct gguf_context * gguf_init_empty_sparse(void);

    // 从文件中初始化 gguf_context 对象，传入文件名和初始化参数
    GGML_API struct gguf_context * gguf_init_from_file(const char * fname, struct gguf_init_params params);

    // 释放 gguf_context 对象
    GGML_API void gguf_free(struct gguf_context * ctx);

    // 返回 gguf_type 枚举类型的名称
    GGML_API const char * gguf_type_name(enum gguf_type type);
// 获取 GGUF 上下文中的版本号
GGML_API int gguf_get_version(const struct gguf_context * ctx);

// 获取 GGUF 上下文中的对齐方式
GGML_API size_t gguf_get_alignment(const struct gguf_context * ctx);

// 获取 GGUF 上下文中数据的偏移量
GGML_API size_t gguf_get_data_offset(const struct gguf_context * ctx);

// 获取 GGUF 上下文中的数据
GGML_API void * gguf_get_data(const struct gguf_context * ctx);

// 获取 GGUF 上下文中键值对的数量
GGML_API int gguf_get_n_kv(const struct gguf_context * ctx);

// 在 GGUF 上下文中查找指定键的索引
GGML_API int gguf_find_key(const struct gguf_context * ctx, const char * key);

// 获取 GGUF 上下文中指定键的键名
GGML_API const char * gguf_get_key(const struct gguf_context * ctx, int key_id);

// 获取 GGUF 上下文中指定键的值的类型
GGML_API enum gguf_type gguf_get_kv_type(const struct gguf_context * ctx, int key_id);

// 获取 GGUF 上下文中指定键的数组类型
GGML_API enum gguf_type gguf_get_arr_type(const struct gguf_context * ctx, int key_id);

// 如果使用错误的类型获取键的值，则中止程序
GGML_API uint8_t gguf_get_val_u8(const struct gguf_context * ctx, int key_id);
GGML_API int8_t gguf_get_val_i8(const struct gguf_context * ctx, int key_id);
GGML_API uint16_t gguf_get_val_u16(const struct gguf_context * ctx, int key_id);
GGML_API int16_t gguf_get_val_i16(const struct gguf_context * ctx, int key_id);
GGML_API uint32_t gguf_get_val_u32(const struct gguf_context * ctx, int key_id);
GGML_API int32_t gguf_get_val_i32(const struct gguf_context * ctx, int key_id);
// 从给定的 gguf_context 中获取指定 key_id 对应的 float 值
GGML_API float gguf_get_val_f32(const struct gguf_context * ctx, int key_id);

// 从给定的 gguf_context 中获取指定 key_id 对应的 uint64_t 值
GGML_API uint64_t gguf_get_val_u64(const struct gguf_context * ctx, int key_id);

// 从给定的 gguf_context 中获取指定 key_id 对应的 int64_t 值
GGML_API int64_t gguf_get_val_i64(const struct gguf_context * ctx, int key_id);

// 从给定的 gguf_context 中获取指定 key_id 对应的 double 值
GGML_API double gguf_get_val_f64(const struct gguf_context * ctx, int key_id);

// 从给定的 gguf_context 中获取指定 key_id 对应的 bool 值
GGML_API bool gguf_get_val_bool(const struct gguf_context * ctx, int key_id);

// 从给定的 gguf_context 中获取指定 key_id 对应的字符串值
GGML_API const char * gguf_get_val_str(const struct gguf_context * ctx, int key_id);

// 从给定的 gguf_context 中获取指定 key_id 对应的数组长度
GGML_API int gguf_get_arr_n(const struct gguf_context * ctx, int key_id);

// 从给定的 gguf_context 中获取指定 key_id 对应的数组数据
GGML_API const void * gguf_get_arr_data(const struct gguf_context * ctx, int key_id);

// 从给定的 gguf_context 中获取指定 key_id 对应的数组中索引为 i 的字符串值
GGML_API const char * gguf_get_arr_str(const struct gguf_context * ctx, int key_id, int i);

// 从给定的 gguf_context 中获取稀疏导数类型
GGML_API enum ggml_sparse_deriv gguf_get_sparse_deriv(const struct gguf_context * ctx);

// 从给定的 gguf_context 中获取张量的数量
GGML_API int gguf_get_n_tensors(const struct gguf_context * ctx);

// 从给定的 gguf_context 中查找指定名称的张量
GGML_API int gguf_find_tensor(const struct gguf_context * ctx, const char * name);

// 从给定的 gguf_context 中获取指定索引 i 对应的张量偏移量
GGML_API size_t gguf_get_tensor_offset(const struct gguf_context * ctx, int i);

// 从给定的 gguf_context 中获取指定索引 i 对应的张量名称
GGML_API char * gguf_get_tensor_name(const struct gguf_context * ctx, int i);

// 设置给定的 gguf_context 中指定 key 对应的 uint8_t 值
GGML_API void gguf_set_val_u8(struct gguf_context * ctx, const char * key, uint8_t val);

// 设置给定的 gguf_context 中指定 key 对应的 int8_t 值
GGML_API void gguf_set_val_i8(struct gguf_context * ctx, const char * key, int8_t val);

// 设置给定的 gguf_context 中指定 key 对应的 uint16_t 值
GGML_API void gguf_set_val_u16(struct gguf_context * ctx, const char * key, uint16_t val);
// 设置一个int16_t类型的值到指定的key
GGML_API void gguf_set_val_i16 (struct gguf_context * ctx, const char * key, int16_t  val);
// 设置一个uint32_t类型的值到指定的key
GGML_API void gguf_set_val_u32 (struct gguf_context * ctx, const char * key, uint32_t val);
// 设置一个int32_t类型的值到指定的key
GGML_API void gguf_set_val_i32 (struct gguf_context * ctx, const char * key, int32_t  val);
// 设置一个float类型的值到指定的key
GGML_API void gguf_set_val_f32 (struct gguf_context * ctx, const char * key, float    val);
// 设置一个uint64_t类型的值到指定的key
GGML_API void gguf_set_val_u64 (struct gguf_context * ctx, const char * key, uint64_t val);
// 设置一个int64_t类型的值到指定的key
GGML_API void gguf_set_val_i64 (struct gguf_context * ctx, const char * key, int64_t  val);
// 设置一个double类型的值到指定的key
GGML_API void gguf_set_val_f64 (struct gguf_context * ctx, const char * key, double   val);
// 设置一个bool类型的值到指定的key
GGML_API void gguf_set_val_bool(struct gguf_context * ctx, const char * key, bool     val);
// 设置一个字符串类型的值到指定的key
GGML_API void gguf_set_val_str (struct gguf_context * ctx, const char * key, const char * val);
// 设置一个数组数据到指定的key
GGML_API void gguf_set_arr_data(struct gguf_context * ctx, const char * key, enum gguf_type type, const void * data, int n);
// 设置一个字符串数组到指定的key
GGML_API void gguf_set_arr_str (struct gguf_context * ctx, const char * key, const char ** data, int n);

// 从另一个上下文中设置或添加键值对
GGML_API void gguf_set_kv(struct gguf_context * ctx, struct gguf_context * src);

// 管理张量信息
GGML_API void gguf_add_tensor(struct gguf_context * ctx, const struct ggml_tensor * tensor);
// 设置张量的类型
GGML_API void gguf_set_tensor_type(struct gguf_context * ctx, const char * name, enum ggml_type type);
// 设置张量的数据
GGML_API void gguf_set_tensor_data(struct gguf_context * ctx, const char * name, const void * data, size_t size);
// 写入 gguf 文件有两种方法：
//
// - 在单个步骤中将整个 gguf_context 写入二进制文件：
//
//   gguf_write_to_file(ctx, fname);
//
// - 首先准备一个带有元数据占位符的文件，写入张量数据，然后写入元数据：
//
//   FILE * f = fopen(fname, "wb");
//   fseek(f, gguf_get_meta_size(ctx), SEEK_SET);
//   fwrite(f, ...);
//   void * data = gguf_meta_get_meta_data(ctx);
//   fseek(f, 0, SEEK_SET);
//   fwrite(f, data, gguf_get_meta_size(ctx));
//   free(data);
//   fclose(f);
//

// 将整个上下文写入二进制文件
GGML_API void gguf_write_to_file(const struct gguf_context * ctx, const char * fname, bool only_meta);
// 获取元数据（头部、键值对、张量信息）的字节大小，包括填充
GGML_API size_t gguf_get_meta_size(const struct gguf_context * ctx);

// 获取元数据的内容
GGML_API void   gguf_get_meta_data(const struct gguf_context * ctx, void * data);

//
// 系统信息
//

// 检查 CPU 是否支持 AVX 指令集
GGML_API int ggml_cpu_has_avx(void);

// 检查 CPU 是否支持 AVX2 指令集
GGML_API int ggml_cpu_has_avx2(void);

// 检查 CPU 是否支持 AVX512 指令集
GGML_API int ggml_cpu_has_avx512(void);

// 检查 CPU 是否支持 AVX512 VBMI 指令集
GGML_API int ggml_cpu_has_avx512_vbmi(void);

// 检查 CPU 是否支持 AVX512 VNNI 指令集
GGML_API int ggml_cpu_has_avx512_vnni(void);

// 检查 CPU 是否支持 FMA 指令集
GGML_API int ggml_cpu_has_fma(void);

// 检查 CPU 是否支持 NEON 指令集
GGML_API int ggml_cpu_has_neon(void);

// 检查 CPU 是否支持 ARM FMA 指令集
GGML_API int ggml_cpu_has_arm_fma(void);

// 检查 CPU 是否支持 Metal 指令集
GGML_API int ggml_cpu_has_metal(void);

// 检查 CPU 是否支持 F16C 指令集
GGML_API int ggml_cpu_has_f16c(void);

// 检查 CPU 是否支持 FP16 VA 指令集
GGML_API int ggml_cpu_has_fp16_va(void);
    // 检查 CPU 是否支持 WebAssembly SIMD
    GGML_API int ggml_cpu_has_wasm_simd  (void);
    // 检查 CPU 是否支持 BLAS
    GGML_API int ggml_cpu_has_blas       (void);
    // 检查 CPU 是否支持 cuBLAS
    GGML_API int ggml_cpu_has_cublas     (void);
    // 检查 CPU 是否支持 CLBlast
    GGML_API int ggml_cpu_has_clblast    (void);
    // 检查 CPU 是否支持 GPUBlas
    GGML_API int ggml_cpu_has_gpublas    (void);
    // 检查 CPU 是否支持 SSE3
    GGML_API int ggml_cpu_has_sse3       (void);
    // 检查 CPU 是否支持 SSSE3
    GGML_API int ggml_cpu_has_ssse3      (void);
    // 检查 CPU 是否支持 VSX
    GGML_API int ggml_cpu_has_vsx        (void);

    //
    // 全局变量
    // 
    // TODO: 这些应该移动到上下文中
    // 外部声明稀疏预测阈值
    extern float sparse_pred_threshold;

    //
    // 用于测试和基准测试的内部类型和函数
    //

#ifdef  __cplusplus
// 定义 GGML_RESTRICT 宏，如果不是标准的 C++ 则定义为空，否则定义为 restrict
#define GGML_RESTRICT
// 如果不是标准的 C++ 则将 GGML_RESTRICT 定义为 restrict
#else
#define GGML_RESTRICT restrict
#endif
// 定义 ggml_to_float_t 类型，表示将输入转换为浮点数
typedef void (*ggml_to_float_t)  (const void  * GGML_RESTRICT x, float * GGML_RESTRICT y, int k);
// 定义 ggml_from_float_t 类型，表示将浮点数转换为输出
typedef void (*ggml_from_float_t)(const float * GGML_RESTRICT x, void  * GGML_RESTRICT y, int k);
// 定义 ggml_vec_dot_t 类型，表示向量点积操作
typedef void (*ggml_vec_dot_t)   (const int n, float * GGML_RESTRICT s, const void * GGML_RESTRICT x, const void * GGML_RESTRICT y);

// 定义 ggml_type_traits_t 结构体，包含类型名称、块大小、类型大小、是否量化、转换为浮点数的函数指针、从浮点数转换的函数指针、参考的从浮点数转换的函数指针、向量点积的函数指针、向量点积的类型
typedef struct {
    const char      * type_name;
    int               blck_size;
    size_t            type_size;
    bool              is_quantized;
    ggml_to_float_t   to_float;
    ggml_from_float_t from_float;
    ggml_from_float_t from_float_reference;
    ggml_vec_dot_t    vec_dot;
    enum ggml_type    vec_dot_type;
} ggml_type_traits_t;
# 定义一个名为 ggml_internal_get_type_traits 的函数，该函数接受一个 ggml_type 枚举类型的参数，并返回一个 ggml_type_traits_t 类型的值
GGML_API ggml_type_traits_t ggml_internal_get_type_traits(enum ggml_type type);

# 如果是 C++ 环境，则结束 extern "C" 块
#ifdef  __cplusplus
}
#endif
```