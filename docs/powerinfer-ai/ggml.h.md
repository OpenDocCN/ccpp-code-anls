# `PowerInfer\ggml.h`

```
#pragma once

// GGML Tensor Library
//
// This documentation is still a work in progress.
// If you wish some specific topics to be covered, feel free to drop a comment:
//
//   https://github.com/ggerganov/whisper.cpp/issues/40
//
// ## Overview
//
// This library implements:
//
//  - a set of tensor operations
//  - automatic differentiation
//  - basic optimization algorithms
//
// The aim of this library is to provide a minimalistic approach for various machine learning tasks. This includes,
// but is not limited to, the following:
//
//  - linear regression
//  - support vector machines
//  - neural networks
//
// The library allows the user to define a certain function using the available tensor operations. This function
// definition is represented internally via a computation graph. Each tensor operation in the function definition
// corresponds to a node in the graph. Having the computation graph defined, the user can choose to compute the
// function's value and/or its gradient with respect to the input variables. Optionally, the function can be optimized
// using one of the available optimization algorithms.
//
// For example, here we define the function: f(x) = a*x^2 + b
//
//   {
//       struct ggml_init_params params = {
//           .mem_size   = 16*1024*1024,  // 设置内存大小为16MB
//           .mem_buffer = NULL,  // 内存缓冲区为空
//       };
//
//       // memory allocation happens here
//       struct ggml_context * ctx = ggml_init(params);  // 初始化 GGML 上下文
//
//       struct ggml_tensor * x = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 1);  // 创建一个一维浮点型张量 x
//
//       ggml_set_param(ctx, x); // x is an input variable  // 将 x 设置为输入变量
//
//       struct ggml_tensor * a  = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 1);  // 创建一个一维浮点型张量 a
//       struct ggml_tensor * b  = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 1);  // 创建一个一维浮点型张量 b
//       struct ggml_tensor * x2 = ggml_mul(ctx, x, x);  // 计算 x 的平方
//       struct ggml_tensor * f  = ggml_add(ctx, ggml_mul(ctx, a, x2), b);  // 计算 a*x^2 + b
//
//       ...
//   }
//
// 注意，上面的函数定义并不涉及任何实际的计算。计算只有在用户明确请求时才执行。例如，要计算 x = 2.0 时的函数值：
//
//   {
//       ...
//
//       // 创建一个新的计算图
//       struct ggml_cgraph * gf = ggml_new_graph(ctx);
//       // 构建前向展开
//       ggml_build_forward_expand(gf, f);
//
//       // 设置输入变量和参数值
//       ggml_set_f32(x, 2.0f);
//       ggml_set_f32(a, 3.0f);
//       ggml_set_f32(b, 4.0f);
//
//       // 使用上下文计算计算图
//       ggml_graph_compute_with_ctx(ctx, &gf, n_threads);
//
//       // 打印结果
//       printf("f = %f\n", ggml_get_f32_1d(f, 0));
//
//       ...
//   }
//
// 实际的计算是在 ggml_graph_compute() 函数中执行的。
//
// ggml_new_tensor_...() 函数创建新的张量。它们分配在提供给 ggml_init() 函数的内存缓冲区中。您必须小心，不要超出内存缓冲区的大小。因此，您必须预先知道计算所需的内存量。或者，您可以分配足够大的内存，并在定义计算图后调用 ggml_used_mem() 函数，以找出实际需要的内存量。
//
// ggml_set_param() 函数将张量标记为输入变量。这是自动微分和优化算法使用的。
//
// 上述方法允许一次定义函数图，然后多次计算其前向或后向图。所有计算都将使用在 ggml_init() 函数中分配的相同内存缓冲区。这样，用户可以避免运行时的内存分配开销。
//
// 该库支持多维张量 - 最多 4 维。FP16 和 FP32 数据类型是一等公民，但理论上该库可以扩展以支持 FP8 和整数数据类型。
//
// 每个张量操作都会产生一个新的张量。最初，该库被设想为仅支持一元操作
// and binary operations. Most of the available operations fall into one of these two categories. With time, it became
// clear that the library needs to support more complex operations. The way to support these operations is not clear
// yet, but a few examples are demonstrated in the following operations:
//
//   - ggml_permute()
//   - ggml_conv_1d_1s()
//   - ggml_conv_1d_2s()
//
// For each tensor operator, the library implements a forward and backward computation function. The forward function
// computes the output tensor value given the input tensor values. The backward function computes the adjoint of the
// input tensors given the adjoint of the output tensor. For a detailed explanation of what this means, take a
// calculus class, or watch the following video:
//
//   What is Automatic Differentiation?
//   https://www.youtube.com/watch?v=wG_nF1awSSY
//
//
// ## Tensor data (struct ggml_tensor)
//
// The tensors are stored in memory via the ggml_tensor struct. The structure provides information about the size of
// the tensor, the data type, and the memory buffer where the tensor data is stored. Additionally, it contains
// pointers to the "source" tensors - i.e. the tensors that were used to compute the current tensor. For example:
//
//   {
//       struct ggml_tensor * c = ggml_add(ctx, a, b);
//
//       assert(c->src[0] == a);
//       assert(c->src[1] == b);
//   }
//
// The multi-dimensional tensors are stored in row-major order. The ggml_tensor struct contains fields for the
// number of elements in each dimension ("ne") as well as the number of bytes ("nb", a.k.a. stride). This allows
// to store tensors that are not contiguous in memory, which is useful for operations such as transposition and
// permutation. All tensor operations have to take the stride into account and not assume that the tensor is
// contiguous in memory.
//
// The data of the tensor is accessed via the "data" pointer. For example:
//
//   {
//       const int nx = 2;
// 定义常量 ny 为 3
const int ny = 3;

// 创建一个二维的 ggml_tensor 结构体指针 a，数据类型为 GGML_TYPE_F32，大小为 nx * ny
struct ggml_tensor * a = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, nx, ny);

// 遍历二维数组，给每个元素赋值为 x + y
for (int y = 0; y < ny; y++) {
    for (int x = 0; x < nx; x++) {
        *(float *) ((char *) a->data + y*a->nb[1] + x*a->nb[0]) = x + y;
    }
}

// ...
// 其他代码省略

// 定义 GGML_API 宏，根据不同平台和编译选项设置不同的属性
#ifdef GGML_SHARED
#    if defined(_WIN32) && !defined(__MINGW32__)
#        ifdef GGML_BUILD
#            define GGML_API __declspec(dllexport)
#        else
#            define GGML_API __declspec(dllimport)
#        endif
#    else
#        define GGML_API __attribute__ ((visibility ("default")))
#    endif
#else
#    define GGML_API
#endif

// 定义 GGML_DEPRECATED 宏，根据不同编译器设置不同的属性
// TODO: support for clang
#ifdef __GNUC__
#    define GGML_DEPRECATED(func, hint) func __attribute__((deprecated(hint)))
#elif defined(_MSC_VER)
#    define GGML_DEPRECATED(func, hint) __declspec(deprecated(hint)) func
#else
#    define GGML_DEPRECATED(func, hint) func
#endif

// 定义 GGML_ATTRIBUTE_FORMAT 宏，根据不同编译器设置不同的属性
#ifndef __GNUC__
#    define GGML_ATTRIBUTE_FORMAT(...)
#elif defined(__MINGW32__)
#    define GGML_ATTRIBUTE_FORMAT(...) __attribute__((format(gnu_printf, __VA_ARGS__)))
#else
#    define GGML_ATTRIBUTE_FORMAT(...) __attribute__((format(printf, __VA_ARGS__)))
#endif

// 引入标准库头文件
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

// 定义 GGML_FILE_MAGIC 和 GGML_FILE_VERSION 常量
#define GGML_FILE_MAGIC   0x67676d6c // "ggml"
#define GGML_FILE_VERSION 1
// 定义量化格式版本号，当量化格式发生变化时需要增加版本号
#define GGML_QNT_VERSION        2    
// 量化版本号因子，不要更改
#define GGML_QNT_VERSION_FACTOR 1000 

// 定义最大维度
#define GGML_MAX_DIMS           4
// 定义最大参数个数
#define GGML_MAX_PARAMS         1024
// 定义最大上下文数
#define GGML_MAX_CONTEXTS       64
// 定义最大源数
#define GGML_MAX_SRC            6
// 定义最大名称长度
#define GGML_MAX_NAME           64
// 定义最大操作参数个数
#define GGML_MAX_OP_PARAMS      64
// 默认线程数
#define GGML_DEFAULT_N_THREADS  4
// 默认图大小
#define GGML_DEFAULT_GRAPH_SIZE 2048
// 根据指针大小定义内存对齐方式
#if UINTPTR_MAX == 0xFFFFFFFF
    #define GGML_MEM_ALIGN 4
#else
    #define GGML_MEM_ALIGN 16
#endif

// 定义退出成功状态码
#define GGML_EXIT_SUCCESS 0
// 定义退出异常状态码
#define GGML_EXIT_ABORTED 1

// 定义 GGUF 文件的魔数
#define GGUF_MAGIC "GGUF"
// 定义 PowerInfer 版本的 GGUF 文件魔数
#define GGUF_POWERINFER_MAGIC "PWRI"
// 定义 GGUF 文件版本号
#define GGUF_VERSION 3
// 默认对齐方式
#define GGUF_DEFAULT_ALIGNMENT 32

// 定义一个宏，用于消除未使用变量的编译警告
#define GGML_UNUSED(x) (void)(x)

// 定义一个宏，用于计算 x 在 n 对齐下的大小
#define GGML_PAD(x, n) (((x) + (n) - 1) & ~((n) - 1))

// 定义一个宏，用于断言条件是否成立，如果不成立则输出错误信息并退出程序
#define GGML_ASSERT(x) \
    do { \
        if (!(x)) { \
            fprintf(stderr, "GGML_ASSERT: %s:%d: %s\n", __FILE__, __LINE__, #x); \
            fflush(stderr); \
            fflush(stdout); \
            ggml_print_backtrace(); \
            exit(1); \
        } \
    } while (0)

// 定义一个宏，用于在调试模式下断言条件是否成立，如果不成立则输出带有额外信息的错误信息并退出程序
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

// 如果处于调试模式下，定义一个宏，用于标记不应该被执行到的代码
#ifndef NDEBUG
#define GGML_UNREACHABLE() GGML_ASSERT(!"statement should not be reached")
// 如果是使用 GCC 编译器，使用内建函数标记不应该被执行到的代码
#elif defined(__GNUC__)
#define GGML_UNREACHABLE() __builtin_unreachable()
// 否则，使用空语句标记不应该被执行到的代码
#else
#define GGML_UNREACHABLE() ((void) 0)
#endif

// 用于将张量的元素数量和字节步长复制到本地变量中的宏
// 主要目的是减少代码重复和提高可读性
//
// 例子:
//
//    GGML_TENSOR_LOCALS(int64_t, ne1, src1, ne);
//    GGML_TENSOR_LOCALS(size_t,  nb1, src1, nb);
//
#define GGML_TENSOR_LOCALS_1(type, prefix, pointer, array) \
    # 定义一个常量类型，命名为prefix##0，其值为(pointer)->array[0]
    const type prefix##0 = (pointer)->array[0]; \
    # 使用GGML_UNUSED宏处理prefix##0，防止编译器警告
    GGML_UNUSED(prefix##0);
// 定义宏，用于声明和初始化指定类型的局部变量
#define GGML_TENSOR_LOCALS_2(type, prefix, pointer, array) \
    GGML_TENSOR_LOCALS_1    (type, prefix, pointer, array) \  // 调用上一级宏
    const type prefix##1 = (pointer)->array[1]; \  // 声明并初始化局部变量
    GGML_UNUSED(prefix##1);  // 使用宏来避免未使用变量的警告
#define GGML_TENSOR_LOCALS_3(type, prefix, pointer, array) \
    GGML_TENSOR_LOCALS_2    (type, prefix, pointer, array) \  // 调用上一级宏
    const type prefix##2 = (pointer)->array[2]; \  // 声明并初始化局部变量
    GGML_UNUSED(prefix##2);  // 使用宏来避免未使用变量的警告
#define GGML_TENSOR_LOCALS(type, prefix, pointer, array) \
    GGML_TENSOR_LOCALS_3  (type, prefix, pointer, array) \  // 调用上一级宏
    const type prefix##3 = (pointer)->array[3]; \  // 声明并初始化局部变量
    GGML_UNUSED(prefix##3);  // 使用宏来避免未使用变量的警告

#ifdef  __cplusplus  // 如果是 C++ 环境
extern "C" {  // 使用 C 语言的调用约定
#endif

#if defined(__ARM_NEON) && defined(__CUDACC__)  // 如果是 ARM NEON 和 CUDA 环境
    typedef half ggml_fp16_t;  // 定义半精度浮点类型
#elif defined(__ARM_NEON)  // 如果是 ARM NEON 环境
    typedef __fp16 ggml_fp16_t;  // 定义半精度浮点类型
#else  // 其他情况
    typedef uint16_t ggml_fp16_t;  // 定义 16 位无符号整数类型
#endif

    // 定义函数原型，用于将半精度浮点数转换为单精度浮点数
    GGML_API float       ggml_fp16_to_fp32(ggml_fp16_t x);
    // 定义函数原型，用于将单精度浮点数转换为半精度浮点数
    GGML_API ggml_fp16_t ggml_fp32_to_fp16(float x);

    // 定义函数原型，用于将半精度浮点数数组转换为单精度浮点数数组
    GGML_API void ggml_fp16_to_fp32_row(const ggml_fp16_t * x, float * y, int n);
    // 定义函数原型，用于将单精度浮点数数组转换为半精度浮点数数组
    GGML_API void ggml_fp32_to_fp16_row(const float * x, ggml_fp16_t * y, int n);

    // 声明结构体 ggml_object
    struct ggml_object;
    // 声明结构体 ggml_context
    struct ggml_context;

    // 定义枚举类型 ggml_type，表示数据类型
    enum ggml_type {
        GGML_TYPE_F32  = 0,  // 单精度浮点数
        GGML_TYPE_F16  = 1,  // 半精度浮点数
        GGML_TYPE_Q4_0 = 2,  // 四位定点数
        GGML_TYPE_Q4_1 = 3,  // 四位定点数
        // GGML_TYPE_Q4_2 = 4, support has been removed
        // GGML_TYPE_Q4_3 (5) support has been removed
        GGML_TYPE_Q5_0 = 6,  // 五位定点数
        GGML_TYPE_Q5_1 = 7,  // 五位定点数
        GGML_TYPE_Q8_0 = 8,  // 八位定点数
        GGML_TYPE_Q8_1 = 9,  // 八位定点数
        // k-quantizations
        GGML_TYPE_Q2_K = 10,  // 二位定点数
        GGML_TYPE_Q3_K = 11,  // 三位定点数
        GGML_TYPE_Q4_K = 12,  // 四位定点数
        GGML_TYPE_Q5_K = 13,  // 五位定点数
        GGML_TYPE_Q6_K = 14,  // 六位定点数
        GGML_TYPE_Q8_K = 15,  // 八位定点数
        GGML_TYPE_I8,  // 8 位整数
        GGML_TYPE_I16,  // 16 位整数
        GGML_TYPE_I32,  // 32 位整数
        GGML_TYPE_COUNT,  // 数据类型数量
    };

    // 定义枚举类型 ggml_backend_type，表示后端类型
    enum ggml_backend_type {
        GGML_BACKEND_CPU = 0,  // CPU 后端
        GGML_BACKEND_GPU = 10,  // GPU 后端
        GGML_BACKEND_GPU_SPLIT = 20,  // 分布式 GPU 后端
    };
    // 稀疏导数的枚举类型，用于表示稠密推断和稀疏推断
    enum ggml_sparse_deriv {
        GGML_DENSE_INFERENCE = 0,  // 稠密推断
        GGML_SPARSE_INFERENCE = 1,  // 稀疏推断
    };

    // 模型文件类型的枚举类型
    enum ggml_ftype {
        GGML_FTYPE_UNKNOWN     = -1,  // 未知类型
        GGML_FTYPE_ALL_F32     = 0,   // 所有数据类型为F32
        GGML_FTYPE_MOSTLY_F16  = 1,   // 大部分数据类型为F16，除了1维张量
        GGML_FTYPE_MOSTLY_Q4_0 = 2,   // 大部分数据类型为Q4.0，除了1维张量
        GGML_FTYPE_MOSTLY_Q4_1 = 3,   // 大部分数据类型为Q4.1，除了1维张量
        GGML_FTYPE_MOSTLY_Q4_1_SOME_F16 = 4, // tok_embeddings.weight 和 output.weight 数据类型为F16
        GGML_FTYPE_MOSTLY_Q8_0 = 7,   // 大部分数据类型为Q8.0，除了1维张量
        GGML_FTYPE_MOSTLY_Q5_0 = 8,   // 大部分数据类型为Q5.0，除了1维张量
        GGML_FTYPE_MOSTLY_Q5_1 = 9,   // 大部分数据类型为Q5.1，除了1维张量
        GGML_FTYPE_MOSTLY_Q2_K = 10,  // 大部分数据类型为Q2.K，除了1维张量
        GGML_FTYPE_MOSTLY_Q3_K = 11,  // 大部分数据类型为Q3.K，除了1维张量
        GGML_FTYPE_MOSTLY_Q4_K = 12,  // 大部分数据类型为Q4.K，除了1维张量
        GGML_FTYPE_MOSTLY_Q5_K = 13,  // 大部分数据类型为Q5.K，除了1维张量
        GGML_FTYPE_MOSTLY_Q6_K = 14,  // 大部分数据类型为Q6.K，除了1维张量
    };

    // 可用的张量操作：
    # 定义枚举类型 ggml_op，表示不同的操作
    enum ggml_op {
        GGML_OP_NONE = 0,  # 表示无操作
    
        GGML_OP_DUP,  # 复制操作
        GGML_OP_ADD,  # 加法操作
        GGML_OP_ADD1,  # 加1操作
        GGML_OP_ACC,  # 累加操作
        GGML_OP_SUB,  # 减法操作
        GGML_OP_MUL,  # 乘法操作
        GGML_OP_DIV,  # 除法操作
        GGML_OP_SQR,  # 平方操作
        GGML_OP_SQRT,  # 平方根操作
        GGML_OP_LOG,  # 对数操作
        GGML_OP_SUM,  # 求和操作
        GGML_OP_SUM_ROWS,  # 对行求和操作
        GGML_OP_MEAN,  # 求平均值操作
        GGML_OP_ARGMAX,  # 最大值索引操作
        GGML_OP_REPEAT,  # 重复操作
        GGML_OP_REPEAT_BACK,  # 反向重复操作
        GGML_OP_CONCAT,  # 连接操作
        GGML_OP_SILU_BACK,  # SILU 反向操作
        GGML_OP_NORM,  # 归一化操作
        GGML_OP_RMS_NORM,  # 均方根归一化操作
        GGML_OP_RMS_NORM_BACK,  # 均方根归一化反向操作
        GGML_OP_GROUP_NORM,  # 分组归一化操作
    
        GGML_OP_MUL_MAT,  # 矩阵乘法操作
        GGML_OP_AXPY,  # AXPY 操作
        GGML_OP_OUT_PROD,  # 外积操作
    
        GGML_OP_SCALE,  # 缩放操作
        GGML_OP_SET,  # 设置操作
        GGML_OP_CPY,  # 复制操作
        GGML_OP_CONT,  # 连续操作
        GGML_OP_RESHAPE,  # 重塑操作
        GGML_OP_VIEW,  # 查看操作
        GGML_OP_PERMUTE,  # 排列操作
        GGML_OP_TRANSPOSE,  # 转置操作
        GGML_OP_GET_ROWS,  # 获取行操作
        GGML_OP_GET_ROWS_BACK,  # 获取行反向操作
        GGML_OP_DIAG,  # 对角线操作
        GGML_OP_DIAG_MASK_INF,  # 对角线掩码（无穷大）操作
        GGML_OP_DIAG_MASK_ZERO,  # 对角线掩码（零）操作
        GGML_OP_SOFT_MAX,  # Softmax 操作
        GGML_OP_SOFT_MAX_BACK,  # Softmax 反向操作
        GGML_OP_ROPE,  # ROPE 操作
        GGML_OP_ROPE_BACK,  # ROPE 反向操作
        GGML_OP_ALIBI,  # ALIBI 操作
        GGML_OP_CLAMP,  # 夹紧操作
        GGML_OP_CONV_TRANSPOSE_1D,  # 1D 卷积转置操作
        GGML_OP_IM2COL,  # 图像转列操作
        GGML_OP_CONV_TRANSPOSE_2D,  # 2D 卷积转置操作
        GGML_OP_POOL_1D,  # 1D 池化操作
        GGML_OP_POOL_2D,  # 2D 池化操作
    
        GGML_OP_UPSCALE,  # 最近邻插值操作
    
        GGML_OP_FLASH_ATTN,  # FLASH 注意力操作
        GGML_OP_FLASH_FF,  # FLASH 前馈操作
        GGML_OP_FLASH_ATTN_BACK,  # FLASH 注意力反向操作
        GGML_OP_WIN_PART,  # 窗口部分操作
        GGML_OP_WIN_UNPART,  # 窗口非部分操作
        GGML_OP_GET_REL_POS,  # 获取相对位置操作
        GGML_OP_ADD_REL_POS,  # 添加相对位置操作
    
        GGML_OP_UNARY,  # 一元操作
    
        GGML_OP_MAP_UNARY,  # 映射一元操作
        GGML_OP_MAP_BINARY,  # 映射二元操作
    
        GGML_OP_MAP_CUSTOM1_F32,  # 映射自定义1（单精度浮点数）操作
        GGML_OP_MAP_CUSTOM2_F32,  # 映射自定义2（单精度浮点数）操作
        GGML_OP_MAP_CUSTOM3_F32,  # 映射自定义3（单精度浮点数）操作
    
        GGML_OP_MAP_CUSTOM1,  # 映射自定义1操作
        GGML_OP_MAP_CUSTOM2,  # 映射自定义2操作
        GGML_OP_MAP_CUSTOM3,  # 映射自定义3操作
    
        GGML_OP_CROSS_ENTROPY_LOSS,  # 交叉熵损失操作
        GGML_OP_CROSS_ENTROPY_LOSS_BACK,  # 交叉熵损失反向操作
    
        GGML_OP_COUNT,  # 操作数量
    };
    # 定义枚举类型，表示一元操作符
    enum ggml_unary_op {
        GGML_UNARY_OP_ABS,  # 绝对值
        GGML_UNARY_OP_SGN,  # 符号
        GGML_UNARY_OP_NEG,  # 取反
        GGML_UNARY_OP_STEP,  # 阶跃函数
        GGML_UNARY_OP_TANH,  # 双曲正切
        GGML_UNARY_OP_ELU,  # 指数线性单元
        GGML_UNARY_OP_RELU,  # 线性整流单元
        GGML_UNARY_OP_GELU,  # 高斯误差线性单元
        GGML_UNARY_OP_GELU_QUICK,  # 快速高斯误差线性单元
        GGML_UNARY_OP_SILU,  # 平滑整流单元
        GGML_UNARY_OP_LEAKY  # 泄漏整流单元
    };

    # 定义枚举类型，表示对象类型
    enum ggml_object_type {
        GGML_OBJECT_TENSOR,  # 张量
        GGML_OBJECT_GRAPH,  # 图
        GGML_OBJECT_WORK_BUFFER  # 工作缓冲区
    };

    # 定义枚举类型，表示日志级别
    enum ggml_log_level {
        GGML_LOG_LEVEL_ERROR = 2,  # 错误
        GGML_LOG_LEVEL_WARN = 3,  # 警告
        GGML_LOG_LEVEL_INFO = 4  # 信息
    };

    # 定义 ggml_object 结构体，表示 ggml 对象
    struct ggml_object {
        size_t offs;  # 偏移量
        size_t size;  # 大小

        struct ggml_object * next;  # 下一个对象

        enum ggml_object_type type;  # 对象类型

        char padding[4];  # 填充字节
    };

    # 定义 GGML_OBJECT_SIZE 常量，表示 ggml_object 结构体的大小
    static const size_t GGML_OBJECT_SIZE = sizeof(struct ggml_object);

    # n-dimensional tensor
    // 定义了一个结构体 ggml_tensor，用于表示张量的相关信息
    struct ggml_tensor {
        enum ggml_type         type;  // 表示张量的数据类型
        enum ggml_backend_type backend;  // 表示张量的后端类型

        struct ggml_backend_buffer * buffer;  // 指向后端缓冲区的指针

        int     n_dims;  // 表示张量的维度
        int64_t ne[GGML_MAX_DIMS]; // number of elements，表示每个维度的元素个数
        size_t  nb[GGML_MAX_DIMS]; // stride in bytes，表示每个维度的字节步长:
                                   // nb[0] = ggml_type_size(type)
                                   // nb[1] = nb[0]   * (ne[0] / ggml_blck_size(type)) + padding
                                   // nb[i] = nb[i-1] * ne[i-1]

        // compute data，表示计算数据
        enum ggml_op op;  // 表示计算操作

        // op params - allocated as int32_t for alignment，表示计算操作的参数，以 int32_t 类型分配以保证对齐
        int32_t op_params[GGML_MAX_OP_PARAMS / sizeof(int32_t)];

        bool is_param;  // 表示是否为参数

        struct ggml_tensor * grad;  // 梯度张量
        struct ggml_tensor * src[GGML_MAX_SRC];  // 源张量数组

        // performance，性能相关信息
        atomic_int is_finish;  // 原子整型，表示计算是否完成
        int     perf_runs;  // 性能运行次数
        int64_t perf_cycles;  // 性能周期数
        int64_t perf_time_us;  // 性能时间（微秒）

        struct ggml_tensor * view_src;  // 视图源张量
        size_t               view_offs;  // 视图偏移量

        void * data;  // 数据指针

        char name[GGML_MAX_NAME];  // 名称数组

        void * extra; // extra things e.g. for ggml-cuda.cu，额外的内容，例如用于 ggml-cuda.cu 的内容

        char padding[12];  // 填充数组
    };


    static const int64_t GGML_NE_WILDCARD = -1;  // 定义了一个常量 GGML_NE_WILDCARD，表示通配符

    static const size_t GGML_TENSOR_SIZE = sizeof(struct ggml_tensor);  // 定义了一个常量 GGML_TENSOR_SIZE，表示 ggml_tensor 结构体的大小

    // the compute plan that needs to be prepared for ggml_graph_compute()
    // since https://github.com/ggerganov/ggml/issues/287
    // 需要为 ggml_graph_compute() 准备的计算计划
    // 自 https://github.com/ggerganov/ggml/issues/287 以来
    struct ggml_cplan {
        size_t    work_size; // size of work buffer, calculated by `ggml_graph_plan()`，工作缓冲区的大小，由 `ggml_graph_plan()` 计算得出
        uint8_t * work_data; // work buffer, to be allocated by caller before calling to `ggml_graph_compute()`，工作缓冲区，需要在调用 `ggml_graph_compute()` 之前由调用者分配

        int n_threads;  // 线程数

        // abort ggml_graph_compute when true，当为 true 时中止 ggml_graph_compute
        bool (*abort_callback)(void * data);  // 中止回调函数
        void * abort_callback_data;  // 中止回调函数的数据
    };

    enum ggml_cgraph_eval_order {
        GGML_CGRAPH_EVAL_ORDER_LEFT_TO_RIGHT = 0,  // 表示计算图的计算顺序为从左到右
        GGML_CGRAPH_EVAL_ORDER_RIGHT_TO_LEFT,  // 表示计算图的计算顺序为从右到左
        GGML_CGRAPH_EVAL_ORDER_COUNT  // 表示计算图的计算顺序数量
    };
    // 定义哈希集合结构体，包含大小和键的数组
    struct ggml_hash_set {
        size_t size;
        struct ggml_tensor ** keys;
    };

    // 计算图结构体
    struct ggml_cgraph {
        int size;  // 大小
        int n_nodes;  // 节点数量
        int n_leafs;  // 叶子节点数量

        struct ggml_tensor ** nodes;  // 节点数组
        struct ggml_tensor ** grads;  // 梯度数组
        struct ggml_tensor ** leafs;  // 叶子节点数组

        struct ggml_hash_set visited_hash_table;  // 哈希集合

        enum ggml_cgraph_eval_order order;  // 计算图评估顺序

        // 性能
        int     perf_runs;  // 运行次数
        int64_t perf_cycles;  // 周期数
        int64_t perf_time_us;  // 时间（微秒）
    };

    // 临时缓冲区结构体
    struct ggml_scratch {
        size_t offs;  // 偏移量
        size_t size;  // 大小
        void * data;  // 数据
    };

    // 上下文结构体
    struct ggml_context {
        size_t mem_size;  // 内存大小
        void * mem_buffer;  // 内存缓冲区
        bool   mem_buffer_owned;  // 是否拥有内存缓冲区
        bool   no_alloc;  // 是否不分配内存
        bool   no_alloc_save; // 当使用临时缓冲区时保存不分配内存的状态

        int    n_objects;  // 对象数量

        struct ggml_object * objects_begin;  // 对象起始位置
        struct ggml_object * objects_end;  // 对象结束位置

        struct ggml_scratch scratch;  // 临时缓冲区
        struct ggml_scratch scratch_save;  // 保存的临时缓冲区
    };

    // 初始化参数结构体
    struct ggml_init_params {
        // 内存池
        size_t mem_size;   // 字节大小
        void * mem_buffer; // 如果为 NULL，则内存将在内部分配
        bool   no_alloc;   // 不为张量数据分配内存
    };


    // 计算类型

    // 注意：除非显式启用，否则不会安排 INIT 或 FINALIZE 传递。
    // 自从 https://github.com/ggerganov/llama.cpp/pull/1995 以来，此行为已更改。
    enum ggml_task_type {
        GGML_TASK_INIT = 0,  // 初始化
        GGML_TASK_COMPUTE,  // 计算
        GGML_TASK_FINALIZE,  // 结束
    };

    // 计算参数结构体
    struct ggml_compute_params {
        enum ggml_task_type type;  // 任务类型

        // ith = 线程索引，nth = 线程数量
        int ith, nth;  // 线程索引和线程数量

        // 所有线程的工作缓冲区
        size_t wsize;  // 缓冲区大小
        void * wdata;  // 缓冲区数据
        atomic_int *aic;  // 原子整型指针
    };

    // 其他
    // 初始化时间模块，程序开始时调用一次
    GGML_API void    ggml_time_init(void); 
    
    // 获取当前时间的毫秒数
    GGML_API int64_t ggml_time_ms(void);
    
    // 获取当前时间的微秒数
    GGML_API int64_t ggml_time_us(void);
    
    // 获取 CPU 循环计数
    GGML_API int64_t ggml_cycles(void);
    
    // 获取每毫秒的 CPU 循环数
    GGML_API int64_t ggml_cycles_per_ms(void);
    
    // 打印函数调用的回溯信息
    GGML_API void    ggml_print_backtrace(void);
    
    // 初始化 NUMA 模块，在 NUMA 系统上调用一次以获得更好的性能
    GGML_API void    ggml_numa_init(void);
    
    // 检查系统是否有多个 NUMA 节点，返回 true 表示有
    GGML_API bool    ggml_is_numa(void);
    
    // 打印单个对象的信息
    GGML_API void    ggml_print_object (const struct ggml_object * obj);
    
    // 打印上下文中所有对象的信息
    GGML_API void    ggml_print_objects(const struct ggml_context * ctx);
    
    // 获取张量中元素的数量
    GGML_API int64_t ggml_nelements   (const struct ggml_tensor * tensor);
    
    // 获取张量中的行数
    GGML_API int64_t ggml_nrows       (const struct ggml_tensor * tensor);
    
    // 获取张量占用的字节数
    GGML_API size_t  ggml_nbytes      (const struct ggml_tensor * tensor);
    
    // 获取张量占用的字节数，按 GGML_MEM_ALIGN 进行了填充
    GGML_API size_t  ggml_nbytes_pad  (const struct ggml_tensor * tensor);
    
    // 获取按指定行数分割后的张量占用的字节数
    GGML_API size_t  ggml_nbytes_split(const struct ggml_tensor * tensor, int nrows_split);
    
    // 获取指定类型的块大小
    GGML_API int     ggml_blck_size (enum ggml_type type);
    
    // 获取指定类型的每个元素占用的字节数
    GGML_API size_t  ggml_type_size (enum ggml_type type);
    
    // 获取指定类型的每个元素占用的字节数，以浮点数形式返回
    GGML_API float   ggml_type_sizef(enum ggml_type type);
    
    // 获取指定类型的名称
    GGML_API const char * ggml_type_name(enum ggml_type type);
    
    // 获取指定操作的名称
    GGML_API const char * ggml_op_name  (enum ggml_op   op);
    
    // 获取指定操作的符号
    GGML_API const char * ggml_op_symbol(enum ggml_op   op);
    
    // 获取张量中每个元素占用的字节数
    GGML_API size_t  ggml_element_size(const struct ggml_tensor * tensor);
    
    // 检查指定类型是否是量化类型
    GGML_API bool    ggml_is_quantized(enum ggml_type type);
    
    // 临时函数，直到 ggml 示例的模型加载被重构
    GGML_API enum ggml_type ggml_ftype_to_ggml_type(enum ggml_ftype ftype);
    
    // 检查张量是否被转置
    GGML_API bool ggml_is_transposed(const struct ggml_tensor * tensor);
    // 检查张量是否是连续存储的
    GGML_API bool ggml_is_contiguous(const struct ggml_tensor * tensor);
    
    // 检查张量是否被置换
    GGML_API bool ggml_is_permuted  (const struct ggml_tensor * tensor);
    
    // 检查两个张量是否具有相同的形状
    GGML_API bool ggml_are_same_shape(const struct ggml_tensor * t0, const struct ggml_tensor * t1);
    
    // 用于计算张量的内存开销
    GGML_API size_t ggml_tensor_overhead(void);
    
    // 初始化 GGML 上下文
    GGML_API struct ggml_context * ggml_init(struct ggml_init_params params);
    
    // 释放 GGML 上下文
    GGML_API void ggml_free(struct ggml_context * ctx);
    
    // 获取 GGML 上下文使用的内存大小
    GGML_API size_t ggml_used_mem(const struct ggml_context * ctx);
    
    // 设置 GGML 上下文的临时内存
    GGML_API size_t ggml_set_scratch (struct ggml_context * ctx, struct ggml_scratch scratch);
    
    // 获取 GGML 上下文是否允许分配内存
    GGML_API bool ggml_get_no_alloc(struct ggml_context * ctx);
    
    // 设置 GGML 上下文是否允许分配内存
    GGML_API void ggml_set_no_alloc(struct ggml_context * ctx, bool no_alloc);
    
    // 获取 GGML 上下文的内存缓冲区
    GGML_API void * ggml_get_mem_buffer     (const struct ggml_context * ctx);
    
    // 获取 GGML 上下文的内存大小
    GGML_API size_t ggml_get_mem_size       (const struct ggml_context * ctx);
    
    // 获取 GGML 上下文的最大张量大小
    GGML_API size_t ggml_get_max_tensor_size(const struct ggml_context * ctx);
    
    // 创建一个新的张量
    GGML_API struct ggml_tensor * ggml_new_tensor(
            struct ggml_context * ctx,
            enum   ggml_type type,
            int    n_dims,
            const int64_t *ne);
    
    // 创建一个新的一维张量
    GGML_API struct ggml_tensor * ggml_new_tensor_1d(
            struct ggml_context * ctx,
            enum   ggml_type type,
            int64_t ne0);
    
    // 创建一个新的二维张量
    GGML_API struct ggml_tensor * ggml_new_tensor_2d(
            struct ggml_context * ctx,
            enum   ggml_type type,
            int64_t ne0,
            int64_t ne1);
    
    // 创建一个新的三维张量
    GGML_API struct ggml_tensor * ggml_new_tensor_3d(
            struct ggml_context * ctx,
            enum   ggml_type type,
            int64_t ne0,
            int64_t ne1,
            int64_t ne2);
    // 创建一个四维张量，包括上下文、类型和四个维度的大小
    GGML_API struct ggml_tensor * ggml_new_tensor_4d(
            struct ggml_context * ctx,
            enum   ggml_type type,
            int64_t ne0,
            int64_t ne1,
            int64_t ne2,
            int64_t ne3);
    
    // 创建一个包含 int32_t 值的张量
    GGML_API struct ggml_tensor * ggml_new_i32(struct ggml_context * ctx, int32_t value);
    
    // 创建一个包含 float 值的张量
    GGML_API struct ggml_tensor * ggml_new_f32(struct ggml_context * ctx, float value);
    
    // 复制一个张量
    GGML_API struct ggml_tensor * ggml_dup_tensor (struct ggml_context * ctx, const struct ggml_tensor * src);
    
    // 创建一个张量的视图
    GGML_API struct ggml_tensor * ggml_view_tensor(struct ggml_context * ctx, struct ggml_tensor * src);
    
    // 获取上下文中的第一个张量
    GGML_API struct ggml_tensor * ggml_get_first_tensor(struct ggml_context * ctx);
    
    // 获取上下文中的下一个张量
    GGML_API struct ggml_tensor * ggml_get_next_tensor (struct ggml_context * ctx, struct ggml_tensor * tensor);
    
    // 根据名称获取张量
    GGML_API struct ggml_tensor * ggml_get_tensor(struct ggml_context * ctx, const char * name);
    
    // 将张量的所有元素设置为零
    GGML_API struct ggml_tensor * ggml_set_zero(struct ggml_tensor * tensor);
    
    // 将张量的所有元素设置为 int32_t 值
    GGML_API struct ggml_tensor * ggml_set_i32 (struct ggml_tensor * tensor, int32_t value);
    
    // 将张量的所有元素设置为 float 值
    GGML_API struct ggml_tensor * ggml_set_f32 (struct ggml_tensor * tensor, float value);
    
    // 将一维索引转换为四维张量的坐标
    GGML_API void    ggml_unravel_index(const struct ggml_tensor * tensor, int64_t i, int64_t * i0, int64_t * i1, int64_t * i2, int64_t * i3);
    
    // 获取一维张量中指定索引处的 int32_t 值
    GGML_API int32_t ggml_get_i32_1d(const struct ggml_tensor * tensor, int i);
    
    // 设置一维张量中指定索引处的 int32_t 值
    GGML_API void    ggml_set_i32_1d(const struct ggml_tensor * tensor, int i, int32_t value);
    
    // 获取四维张量中指定坐标处的 int32_t 值
    GGML_API int32_t ggml_get_i32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3);
    
    // 设置四维张量中指定坐标处的 int32_t 值
    GGML_API void    ggml_set_i32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3, int32_t value);
    
    // 获取一维张量中指定索引处的 float 值
    GGML_API float   ggml_get_f32_1d(const struct ggml_tensor * tensor, int i);
    
    // 设置一维张量中指定索引处的 float 值
    GGML_API void    ggml_set_f32_1d(const struct ggml_tensor * tensor, int i, float value);
    # 获取四维张量中指定位置的浮点数值
    GGML_API float ggml_get_f32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3);
    # 设置四维张量中指定位置的浮点数值
    GGML_API void ggml_set_f32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3, float value);
    
    # 获取张量的数据指针
    GGML_API void * ggml_get_data(const struct ggml_tensor * tensor);
    # 获取张量的浮点数数据指针
    GGML_API float * ggml_get_data_f32(const struct ggml_tensor * tensor);
    # 获取张量的32位整数数据指针
    GGML_API int32_t * ggml_get_data_i32(const struct ggml_tensor * tensor);
    
    # 获取张量的一元操作类型
    GGML_API enum ggml_unary_op ggml_get_unary_op(const struct ggml_tensor * tensor);
    
    # 获取张量的名称
    GGML_API const char * ggml_get_name(const struct ggml_tensor * tensor);
    # 设置张量的名称
    GGML_API struct ggml_tensor * ggml_set_name(struct ggml_tensor * tensor, const char * name);
    # 格式化张量的名称
    GGML_ATTRIBUTE_FORMAT(2, 3)
    GGML_API struct ggml_tensor * ggml_format_name(struct ggml_tensor * tensor, const char * fmt, ...);
    
    # 设置张量的后端类型
    GGML_API void ggml_set_backend(struct ggml_tensor * tensor, enum ggml_backend_type backend);
    
    # 复制张量
    GGML_API struct ggml_tensor * ggml_dup(struct ggml_context * ctx, struct ggml_tensor * a);
    # 原地复制张量，返回视图
    GGML_API struct ggml_tensor * ggml_dup_inplace(struct ggml_context * ctx, struct ggml_tensor * a);
    # 张量相加
    GGML_API struct ggml_tensor * ggml_add(struct ggml_context * ctx, struct ggml_tensor * a, struct ggml_tensor * b);
    # 带索引的张量相加
    GGML_API struct ggml_tensor *ggml_add_idx(struct ggml_context *ctx, struct ggml_tensor *a, struct ggml_tensor *b, struct ggml_tensor *idx);
    # 原地张量相加
    GGML_API struct ggml_tensor * ggml_add_inplace(struct ggml_context * ctx, struct ggml_tensor * a, struct ggml_tensor * b);
    # 定义一个函数，用于将两个张量相加并返回结果
    GGML_API struct ggml_tensor * ggml_add_cast(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            enum   ggml_type      type);
    
    # 定义一个函数，用于将两个张量相加并返回结果
    GGML_API struct ggml_tensor * ggml_add1(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    # 定义一个函数，用于将两个张量相加并将结果存储在第一个张量中
    GGML_API struct ggml_tensor * ggml_add1_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    # 定义一个函数，用于将两个张量进行累加操作并返回结果
    GGML_API struct ggml_tensor * ggml_acc(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                nb1,
            size_t                nb2,
            size_t                nb3,
            size_t                offset);
    
    # 定义一个函数，用于将两个张量进行累加操作并将结果存储在第一个张量中
    GGML_API struct ggml_tensor * ggml_acc_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                nb1,
            size_t                nb2,
            size_t                nb3,
            size_t                offset);
    
    # 定义一个函数，用于将两个张量相减并返回结果
    GGML_API struct ggml_tensor * ggml_sub(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    # 定义一个函数，用于将两个张量相减并将结果存储在第一个张量中
    GGML_API struct ggml_tensor * ggml_sub_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    # 定义一个函数，用于将两个张量相乘并返回结果
    GGML_API struct ggml_tensor * ggml_mul(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    # 定义一个函数，用于将两个张量相乘并将结果存储在第一个张量中
    GGML_API struct ggml_tensor * ggml_mul_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    # 定义一个函数，用于将两个张量相除并返回结果
    GGML_API struct ggml_tensor * ggml_div(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    # 在给定的上下文中，对两个张量进行就地除法操作，返回结果张量
    GGML_API struct ggml_tensor * ggml_div_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    # 在给定的上下文中，对张量进行平方操作，返回结果张量
    GGML_API struct ggml_tensor * ggml_sqr(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    # 在给定的上下文中，对张量进行就地平方操作，返回结果张量
    GGML_API struct ggml_tensor * ggml_sqr_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    # 在给定的上下文中，对张量进行平方根操作，返回结果张量
    GGML_API struct ggml_tensor * ggml_sqrt(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    # 在给定的上下文中，对张量进行就地平方根操作，返回结果张量
    GGML_API struct ggml_tensor * ggml_sqrt_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    # 在给定的上下文中，对张量进行对数操作，返回结果张量
    GGML_API struct ggml_tensor * ggml_log(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    # 在给定的上下文中，对张量进行就地对数操作，返回结果张量
    GGML_API struct ggml_tensor * ggml_log_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    # 在给定的上下文中，对张量进行求和操作，返回结果张量
    # 返回的结果是一个标量
    GGML_API struct ggml_tensor * ggml_sum(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    # 在给定的上下文中，对张量进行沿行求和操作，返回结果张量
    # 输入形状为 [a,b,c,d]，返回形状为 [1,b,c,d]
    GGML_API struct ggml_tensor * ggml_sum_rows(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    # 在给定的上下文中，对张量进行沿行求平均操作，返回结果张量
    GGML_API struct ggml_tensor * ggml_mean(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    # 在给定的上下文中，对张量进行沿行求最大值索引操作，返回结果张量
    GGML_API struct ggml_tensor * ggml_argmax(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    # 如果 a 的形状与 b 相同，并且 a 不是参数，则返回 a
    # 否则，返回一个新的张量：将 a 重复以适应 b 的形状
    GGML_API struct ggml_tensor * ggml_repeat(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    # 将 a 中的重复项求和成 b 的形状
    // 定义一个函数，用于在后向传播中重复张量 a 和 b
    GGML_API struct ggml_tensor * ggml_repeat_back(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    // 在维度 2 上连接张量 a 和 b，用于稳定扩散
    GGML_API struct ggml_tensor * ggml_concat(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    // 计算张量 a 的绝对值
    GGML_API struct ggml_tensor * ggml_abs(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // 计算张量 a 的绝对值，并将结果存储在 a 中
    GGML_API struct ggml_tensor * ggml_abs_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // 计算张量 a 的符号函数
    GGML_API struct ggml_tensor * ggml_sgn(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // 计算张量 a 的符号函数，并将结果存储在 a 中
    GGML_API struct ggml_tensor * ggml_sgn_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // 计算张量 a 的负值
    GGML_API struct ggml_tensor * ggml_neg(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // 计算张量 a 的负值，并将结果存储在 a 中
    GGML_API struct ggml_tensor * ggml_neg_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // 计算张量 a 的阶跃函数
    GGML_API struct ggml_tensor * ggml_step(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // 计算张量 a 的阶跃函数，并将结果存储在 a 中
    GGML_API struct ggml_tensor * ggml_step_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // 计算张量 a 的双曲正切函数
    GGML_API struct ggml_tensor * ggml_tanh(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // 计算张量 a 的双曲正切函数，并将结果存储在 a 中
    GGML_API struct ggml_tensor * ggml_tanh_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // 计算张量 a 的指数线性单元函数
    GGML_API struct ggml_tensor * ggml_elu(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // 计算张量 a 的指数线性单元函数，并将结果存储在 a 中
    GGML_API struct ggml_tensor * ggml_elu_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    // 定义一个名为 ggml_relu 的函数，接受一个 ggml_context 结构体指针和一个 ggml_tensor 结构体指针作为参数，返回一个 ggml_tensor 结构体指针
    GGML_API struct ggml_tensor * ggml_relu(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 定义一个名为 ggml_leaky 的函数，接受一个 ggml_context 结构体指针和一个 ggml_tensor 结构体指针作为参数，返回一个 ggml_tensor 结构体指针
    GGML_API struct ggml_tensor * ggml_leaky(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 定义一个名为 ggml_relu_inplace 的函数，接受一个 ggml_context 结构体指针和一个 ggml_tensor 结构体指针作为参数，返回一个 ggml_tensor 结构体指针
    GGML_API struct ggml_tensor * ggml_relu_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // TODO: double-check this computation is correct
    // 定义一个名为 ggml_gelu 的函数，接受一个 ggml_context 结构体指针和一个 ggml_tensor 结构体指针作为参数，返回一个 ggml_tensor 结构体指针
    GGML_API struct ggml_tensor * ggml_gelu(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 定义一个名为 ggml_gelu_inplace 的函数，接受一个 ggml_context 结构体指针和一个 ggml_tensor 结构体指针作为参数，返回一个 ggml_tensor 结构体指针
    GGML_API struct ggml_tensor * ggml_gelu_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 定义一个名为 ggml_gelu_quick 的函数，接受一个 ggml_context 结构体指针和一个 ggml_tensor 结构体指针作为参数，返回一个 ggml_tensor 结构体指针
    GGML_API struct ggml_tensor * ggml_gelu_quick(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 定义一个名为 ggml_gelu_quick_inplace 的函数，接受一个 ggml_context 结构体指针和一个 ggml_tensor 结构体指针作为参数，返回一个 ggml_tensor 结构体指针
    GGML_API struct ggml_tensor * ggml_gelu_quick_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 定义一个名为 ggml_silu 的函数，接受一个 ggml_context 结构体指针和一个 ggml_tensor 结构体指针作为参数，返回一个 ggml_tensor 结构体指针
    GGML_API struct ggml_tensor * ggml_silu(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 定义一个名为 ggml_silu_inplace 的函数，接受一个 ggml_context 结构体指针和一个 ggml_tensor 结构体指针作为参数，返回一个 ggml_tensor 结构体指针
    GGML_API struct ggml_tensor * ggml_silu_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 定义一个名为 ggml_silu_back 的函数，接受一个 ggml_context 结构体指针、一个 ggml_tensor 结构体指针和一个 ggml_tensor 结构体指针作为参数，返回一个 ggml_tensor 结构体指针
    // 参数 a 代表 x，参数 b 代表 dy
    GGML_API struct ggml_tensor * ggml_silu_back(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    // 定义一个名为 ggml_norm 的函数，接受一个 ggml_context 结构体指针、一个 ggml_tensor 结构体指针和一个浮点数 eps 作为参数，返回一个 ggml_tensor 结构体指针
    // 对矩阵 a 进行按行归一化
    GGML_API struct ggml_tensor * ggml_norm(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            float                 eps);
    
    // 定义一个名为 ggml_norm_inplace 的函数，接受一个 ggml_context 结构体指针、一个 ggml_tensor 结构体指针和一个浮点数 eps 作为参数，返回一个 ggml_tensor 结构体指针
    // 对矩阵 a 进行按行归一化
    GGML_API struct ggml_tensor * ggml_norm_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            float                 eps);
    
    // 定义一个名为 ggml_rms_norm 的函数，接受一个 ggml_context 结构体指针、一个 ggml_tensor 结构体指针和一个浮点数 eps 作为参数，返回一个 ggml_tensor 结构体指针
    GGML_API struct ggml_tensor * ggml_rms_norm(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            float                 eps);
    // 对输入的张量进行均方根归一化操作，返回归一化后的张量
    GGML_API struct ggml_tensor * ggml_rms_norm_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            float                 eps);

    // 沿着 ne0*ne1*n_groups 维度进行分组归一化，用于稳定扩散
    // TODO: 目前 eps 被硬编码为 1e-6
    GGML_API struct ggml_tensor * ggml_group_norm(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_groups);

    // 对输入的张量进行分组归一化操作，返回归一化后的张量
    GGML_API struct ggml_tensor * ggml_group_norm_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_groups);

    // 计算 a - x 的结果
    // b 为 dy
    GGML_API struct ggml_tensor * ggml_rms_norm_back(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            float                 eps);

    // 计算矩阵相乘，A 为 k 列 n 行 => [ne03, ne02, n, k]
    // B 为 k 列 m 行（内部进行转置） => [ne03 * x, ne02 * y, m, k]
    // 结果为 n 列 m 行 => [ne03 * x, ne02 * y, m, n]
    GGML_API struct ggml_tensor * ggml_mul_mat(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    // 计算矩阵相乘，同时考虑索引和偏置
    GGML_API struct ggml_tensor *ggml_mul_mat_idx(
            struct ggml_context *ctx,
            struct ggml_tensor *a,
            struct ggml_tensor *b,
            struct ggml_tensor *idx,
            struct ggml_tensor *d);
    // 计算特殊的矩阵相乘，同时考虑索引、偏置和参考值
    GGML_API struct ggml_tensor *ggml_mul_mat_special(
            struct ggml_context *ctx,
            struct ggml_tensor *a,
            struct ggml_tensor *b,
            struct ggml_tensor *idx,
            struct ggml_tensor *d,
            struct ggml_tensor *ref);
    // 计算 a * x + b * y + c * z + d 的结果
    GGML_API struct ggml_tensor *ggml_axpy(
            struct ggml_context *ctx,
            struct ggml_tensor *a,
            struct ggml_tensor *b,
            struct ggml_tensor *c,
            struct ggml_tensor *d);
    // 计算两个张量的外积，结果是 m 列，p 行的张量
    GGML_API struct ggml_tensor * ggml_out_prod(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    //
    // 不带反向传播的张量操作
    //

    // 乘法操作，返回新的张量
    GGML_API struct ggml_tensor * ggml_scale(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    // 原地操作，返回 a 的视图
    GGML_API struct ggml_tensor * ggml_scale_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    // 将 b 赋值给 a 的视图(view(a,offset,nb1,nb2,3))，返回修改后的 a
    GGML_API struct ggml_tensor * ggml_set(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                nb1,
            size_t                nb2,
            size_t                nb3,
            size_t                offset);

    // 将 b 赋值给 a 的视图(view(a,offset,nb1,nb2,3))，返回 a 的视图
    GGML_API struct ggml_tensor * ggml_set_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                nb1,
            size_t                nb2,
            size_t                nb3,
            size_t                offset);

    // 将 b 赋值给 a 的视图(view(a,offset,nb1,nb2,3))，返回修改后的 a
    GGML_API struct ggml_tensor * ggml_set_1d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                offset);

    // 将 b 赋值给 a 的视图(view(a,offset,nb1,nb2,3))，返回 a 的视图
    GGML_API struct ggml_tensor * ggml_set_1d_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                offset);

    // 将 b 赋值给 a 的视图(view(a,offset,nb1,nb2,3))，返回修改后的 a
    // 设置两个二维张量的值，返回第一个张量的视图
    GGML_API struct ggml_tensor * ggml_set_2d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                nb1,
            size_t                offset);
    
    // 将 b 视图设置为 a 的视图（偏移为 offset，大小为 nb1），返回 a 的视图
    GGML_API struct ggml_tensor * ggml_set_2d_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                nb1,
            size_t                offset);
    
    // 将 a 的值复制到 b，返回 b 的视图
    GGML_API struct ggml_tensor * ggml_cpy(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    // 将 a 的值复制到 b（原地操作），返回 b 的视图
    GGML_API struct ggml_tensor * ggml_cpy_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    // 使张量连续
    GGML_API struct ggml_tensor * ggml_cont(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 使张量连续（原地操作）
    GGML_API struct ggml_tensor * ggml_cont_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 使张量连续，并指定新的形状（一维）
    GGML_API struct ggml_tensor * ggml_cont_1d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0);
    
    // 使张量连续，并指定新的形状（二维）
    GGML_API struct ggml_tensor * ggml_cont_2d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1);
    
    // 使张量连续，并指定新的形状（三维）
    GGML_API struct ggml_tensor * ggml_cont_3d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1,
            int64_t               ne2);
    // 以四维数组的形式返回张量a的视图，ne0-ne3指定新的形状
    // TODO: 当我们开始计算梯度时，应该复制而不是返回视图
    GGML_API struct ggml_tensor * ggml_cont_4d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1,
            int64_t               ne2,
            int64_t               ne3);

    // 返回a的视图，b指定新的形状
    // TODO: 当我们开始计算梯度时，应该复制而不是返回视图
    GGML_API struct ggml_tensor * ggml_reshape(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    // 返回a的视图
    // TODO: 当我们开始计算梯度时，应该复制而不是返回视图
    GGML_API struct ggml_tensor * ggml_reshape_1d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0);

    // 返回a的视图
    // TODO: 当我们开始计算梯度时，应该复制而不是返回视图
    GGML_API struct ggml_tensor * ggml_reshape_2d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1);

    // 返回a的视图
    // TODO: 当我们开始计算梯度时，应该复制而不是返回视图
    GGML_API struct ggml_tensor * ggml_reshape_3d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1,
            int64_t               ne2);

    // 返回a的视图
    // TODO: 当我们开始计算梯度时，应该复制而不是返回视图
    GGML_API struct ggml_tensor * ggml_reshape_4d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1,
            int64_t               ne2,
            int64_t               ne3);

    // 以字节偏移量返回a的视图
    GGML_API struct ggml_tensor * ggml_view_1d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            size_t                offset);
    # 定义一个函数，用于创建一个二维视图的张量
    GGML_API struct ggml_tensor * ggml_view_2d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1,
            size_t                nb1, // 行字节步长
            size_t                offset);
    
    # 定义一个函数，用于创建一个三维视图的张量
    GGML_API struct ggml_tensor * ggml_view_3d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1,
            int64_t               ne2,
            size_t                nb1, // 行字节步长
            size_t                nb2, // 切片字节步长
            size_t                offset);
    
    # 定义一个函数，用于创建一个四维视图的张量
    GGML_API struct ggml_tensor * ggml_view_4d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1,
            int64_t               ne2,
            int64_t               ne3,
            size_t                nb1, // 行字节步长
            size_t                nb2, // 切片字节步长
            size_t                nb3,
            size_t                offset);
    
    # 定义一个函数，用于对张量进行轴置换
    GGML_API struct ggml_tensor * ggml_permute(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   axis0,
            int                   axis1,
            int                   axis2,
            int                   axis3);
    
    # 定
    // 从张量 a, b, c 中获取行数据，返回一个新的张量
    GGML_API struct ggml_tensor * ggml_get_rows_back(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            struct ggml_tensor  * c);

    // 创建对角线张量
    GGML_API struct ggml_tensor * ggml_diag(
        struct ggml_context     * ctx,
        struct ggml_tensor      * a);

    // 将对角线以上的元素设置为 -INF
    GGML_API struct ggml_tensor * ggml_diag_mask_inf(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_past);

    // 将对角线以上的元素设置为 -INF，原地操作，返回视图(a)
    GGML_API struct ggml_tensor * ggml_diag_mask_inf_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_past);

    // 将对角线以上的元素设置为 0
    GGML_API struct ggml_tensor * ggml_diag_mask_zero(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_past);

    // 将对角线以上的元素设置为 0，原地操作，返回视图(a)
    GGML_API struct ggml_tensor * ggml_diag_mask_zero_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_past);

    // 对张量进行 soft max 操作
    GGML_API struct ggml_tensor * ggml_soft_max(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // 对张量进行 soft max 操作，原地操作，返回视图(a)
    GGML_API struct ggml_tensor * ggml_soft_max_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // 对 soft max 操作的结果进行反向传播
    GGML_API struct ggml_tensor * ggml_soft_max_back(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    // 对 soft max 操作的结果进行反向传播，原地操作，返回视图(a)
    GGML_API struct ggml_tensor * ggml_soft_max_back_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    // 旋转位置嵌入
    // 如果 mode & 1 == 1，则跳过 n_past 个元素（已弃用）
    // 如果 mode & 2 == 1，则采用 GPT-NeoX 风格
    // 如果 mode & 4 == 1，则采用 ChatGLM 风格
    //
    // 使用 a 和 b 作为输入，返回一个新的张量，表示 RoPE（Relative Positional Encoding）
    GGML_API struct ggml_tensor * ggml_rope(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   n_dims,
            int                   mode,
            int                   n_ctx);

    // 原地操作，返回 a 的视图
    GGML_API struct ggml_tensor * ggml_rope_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   n_dims,
            int                   mode,
            int                   n_ctx);

    // 自定义 RoPE
    GGML_API struct ggml_tensor * ggml_rope_custom(
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
            float                 beta_slow);

    // 原地操作，返回 a 的视图
    # 定义一个名为 ggml_rope_custom_inplace 的函数，接受多个参数并返回一个指向 ggml_tensor 结构体的指针
    GGML_API struct ggml_tensor * ggml_rope_custom_inplace(
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
            float                 beta_slow);
    
    # 计算 YaRN RoPE 缩放的修正维度
    void ggml_rope_yarn_corr_dims(
        int n_dims, int n_orig_ctx, float freq_base, float beta_fast, float beta_slow, float dims[2]);
    
    # 定义一个名为 ggml_rope_xpos_inplace 的函数，接受多个参数并返回一个指向 ggml_tensor 结构体的指针
    # 这个函数用于计算 xPos RoPE，并且是原地操作，返回 a 的视图
    GGML_API struct ggml_tensor * ggml_rope_xpos_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   n_dims,
            float                 base,
            bool                  down);
    
    # 定义一个名为 ggml_rope_back 的函数，接受多个参数并返回一个指向 ggml_tensor 结构体的指针
    # 这个函数用于计算旋转位置嵌入的反向传播，即从 dy 计算 dx
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
    
    # 定义一个名为 alibi position embedding 的函数，用于原地操作并返回 a 的视图
    // 定义一个名为 ggml_alibi 的函数，接受一个上下文结构体指针、一个张量指针、一个整数和三个浮点数作为参数，返回一个张量指针
    GGML_API struct ggml_tensor * ggml_alibi(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_past,
            int                   n_head,
            float                 bias_max);

    // 定义一个名为 ggml_clamp 的函数，接受一个上下文结构体指针、一个张量指针和两个浮点数作为参数，返回一个张量指针
    // 在原地操作，返回一个视图(a)
    GGML_API struct ggml_tensor * ggml_clamp(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            float                 min,
            float                 max);

    // 定义一个名为 ggml_im2col 的函数，接受一个上下文结构体指针、两个张量指针和七个整数以及一个布尔值作为参数，返回一个张量指针
    GGML_API struct ggml_tensor * ggml_im2col(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                  s0,
            int                  s1,
            int                  p0,
            int                  p1,
            int                  d0,
            int                  d1,
            bool                 is_2D);

    // 定义一个名为 ggml_conv_1d 的函数，接受一个上下文结构体指针、两个张量指针和三个整数作为参数，返回一个张量指针
    GGML_API struct ggml_tensor * ggml_conv_1d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   s0,  // 步长
            int                   p0,  // 填充
            int                   d0); // 膨胀

    // 定义一个名为 ggml_conv_1d_ph 的函数，接受一个上下文结构体指针、两个张量指针和两个整数作为参数，返回一个张量指针
    // 与 ggml_conv_1d 相同，但填充 = a->ne[0]/2
    GGML_API struct ggml_tensor* ggml_conv_1d_ph(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   s,
            int                   d);

    // 定义一个名为 ggml_conv_transpose_1d 的函数，接受一个上下文结构体指针、两个张量指针和三个整数作为参数，返回一个张量指针
    GGML_API struct ggml_tensor * ggml_conv_transpose_1d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   s0,
            int                   p0,
            int                   d0);
    // 定义一个名为 ggml_conv_2d 的函数，接受指向 ggml_context 结构体的指针、两个指向 ggml_tensor 结构体的指针以及八个整型参数，并返回指向 ggml_tensor 结构体的指针
    GGML_API struct ggml_tensor * ggml_conv_2d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   s0,
            int                   s1,
            int                   p0,
            int                   p1,
            int                   d0,
            int                   d1);


    // 卷积核大小为 a->ne[0] x a->ne[1]
    // 步长等于卷积核大小
    // 填充为零
    // 示例:
    // a:     16   16    3  768
    // b:   1024 1024    3    1
    // res:   64   64  768    1
    // 用于 sam
    GGML_API struct ggml_tensor * ggml_conv_2d_sk_p0(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    // 卷积核大小为 a->ne[0] x a->ne[1]
    // 步长为 1
    // 填充为一半
    // 示例:
    // a:      3    3    256  256
    // b:     64   64    256    1
    // res:   64   64    256    1
    // 用于 sam
    GGML_API struct ggml_tensor * ggml_conv_2d_s1_ph(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    // 定义一个名为 ggml_conv_transpose_2d_p0 的函数，接受指向 ggml_context 结构体的指针、两个指向 ggml_tensor 结构体的指针以及一个整型参数，并返回指向 ggml_tensor 结构体的指针
    GGML_API struct ggml_tensor * ggml_conv_transpose_2d_p0(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   stride);

    // 定义一个名为 ggml_pool_1d 的函数，接受指向 ggml_context 结构体的指针、指向 ggml_tensor 结构体的指针、ggml_op_pool 枚举类型和三个整型参数，并返回指向 ggml_tensor 结构体的指针
    GGML_API struct ggml_tensor * ggml_pool_1d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            enum ggml_op_pool     op,
            int                   k0, // 卷积核大小
            int                   s0, // 步长
            int                   p0); // 填充

    // 结果的第一维将有 2*p0 的填充
    // 第二维将有 2*p1 的填充
    // 定义一个名为 ggml_pool_2d 的函数，用于对二维张量进行池化操作
    GGML_API struct ggml_tensor * ggml_pool_2d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            enum ggml_op_pool     op,
            int                   k0,
            int                   k1,
            int                   s0,
            int                   s1,
            float                 p0,
            float                 p1);

    // nearest interpolate
    // used in stable-diffusion
    // 定义一个名为 ggml_upscale 的函数，用于最近邻插值，用于稳定扩散
    GGML_API struct ggml_tensor * ggml_upscale(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   scale_factor);

    // 定义一个名为 ggml_flash_attn 的函数，用于执行闪电注意力机制
    GGML_API struct ggml_tensor * ggml_flash_attn(
            struct ggml_context * ctx,
            struct ggml_tensor  * q,
            struct ggml_tensor  * k,
            struct ggml_tensor  * v,
            bool                  masked);

    // 定义一个名为 ggml_flash_attn_back 的函数，用于执行闪电注意力机制的反向传播
    GGML_API struct ggml_tensor * ggml_flash_attn_back(
           struct ggml_context * ctx,
           struct ggml_tensor  * q,
           struct ggml_tensor  * k,
           struct ggml_tensor  * v,
           struct ggml_tensor  * d,
           bool                  masked);

    // 定义一个名为 ggml_flash_ff 的函数，用于执行闪电前馈操作
    GGML_API struct ggml_tensor * ggml_flash_ff(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b0,
            struct ggml_tensor  * b1,
            struct ggml_tensor  * c0,
            struct ggml_tensor  * c1);

    // partition into non-overlapping windows with padding if needed
    // example:
    // a:   768   64   64    1
    // w:    14
    // res: 768   14   14    25
    // used in sam
    // 定义一个名为 ggml_win_part 的函数，用于将张量分割为非重叠的窗口，如果需要则进行填充
    // 例如：a:   768   64   64    1，w:    14，res: 768   14   14    25，用于 sam
    GGML_API struct ggml_tensor * ggml_win_part(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   w);

    // reverse of ggml_win_part
    // used in sam
    // 定义一个名为 ggml_win_part 的函数的反向操作，用于 sam
    # 定义一个函数，用于将窗口分解为多个部分
    GGML_API struct ggml_tensor * ggml_win_unpart(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   w0,
            int                   h0,
            int                   w);

    # 定义一个函数，用于对输入的张量执行一元操作
    GGML_API struct ggml_tensor * ggml_unary(
            struct ggml_context * ctx,
             struct ggml_tensor * a,
             enum ggml_unary_op op);

    # 定义一个函数，用于对输入的张量执行一元操作，并将结果存储在原始张量中
    GGML_API struct ggml_tensor * ggml_unary_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        enum ggml_unary_op op);

    # 在 sam 中使用的函数，用于获取相对位置
    GGML_API struct ggml_tensor * ggml_get_rel_pos(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   qh,
            int                   kh);

    # 在 sam 中使用的函数，用于将相对位置添加到输入张量中
    GGML_API struct ggml_tensor * ggml_add_rel_pos(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * pw,
            struct ggml_tensor  * ph);

    # 在 sam 中使用的函数，用于将相对位置添加到输入张量中，并将结果存储在原始张量中
    GGML_API struct ggml_tensor * ggml_add_rel_pos_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * pw,
            struct ggml_tensor  * ph);

    # 自定义操作符

    # 定义一个函数指针类型，用于执行一元操作
    typedef void (*ggml_unary_op_f32_t) (const int, float *, const float *);
    # 定义一个函数指针类型，用于执行二元操作
    typedef void (*ggml_binary_op_f32_t)(const int, float *, const float *, const float *);

    # 定义一个函数指针类型，用于执行自定义操作1
    typedef void (*ggml_custom1_op_f32_t)(struct ggml_tensor *, const struct ggml_tensor *);
    # 定义一个函数指针类型，用于执行自定义操作2
    typedef void (*ggml_custom2_op_f32_t)(struct ggml_tensor *, const struct ggml_tensor *, const struct ggml_tensor *);
    # 定义一个函数指针类型，用于执行自定义操作3
    typedef void (*ggml_custom3_op_f32_t)(struct ggml_tensor *, const struct ggml_tensor *, const struct ggml_tensor *, const struct ggml_tensor *);
    # 使用 GGML_DEPRECATED 宏标记 ggml_map_unary_f32 函数，提示使用 ggml_map_custom1 替代
    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_unary_f32(
            struct ggml_context        * ctx,
            struct ggml_tensor         * a,
                   ggml_unary_op_f32_t   fun),
        "use ggml_map_custom1 instead");
    
    # 使用 GGML_DEPRECATED 宏标记 ggml_map_unary_inplace_f32 函数，提示使用 ggml_map_custom1_inplace 替代
    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_unary_inplace_f32(
            struct ggml_context        * ctx,
            struct ggml_tensor         * a,
                   ggml_unary_op_f32_t   fun),
        "use ggml_map_custom1_inplace instead");
    
    # 使用 GGML_DEPRECATED 宏标记 ggml_map_binary_f32 函数，提示使用 ggml_map_custom2 替代
    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_binary_f32(
            struct ggml_context         * ctx,
            struct ggml_tensor          * a,
            struct ggml_tensor          * b,
                   ggml_binary_op_f32_t   fun),
        "use ggml_map_custom2 instead");
    
    # 使用 GGML_DEPRECATED 宏标记 ggml_map_binary_inplace_f32 函数，提示使用 ggml_map_custom2_inplace 替代
    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_binary_inplace_f32(
            struct ggml_context         * ctx,
            struct ggml_tensor          * a,
            struct ggml_tensor          * b,
                   ggml_binary_op_f32_t   fun),
        "use ggml_map_custom2_inplace instead");
    
    # 使用 GGML_DEPRECATED 宏标记 ggml_map_custom1_f32 函数，提示使用 ggml_map_custom1 替代
    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom1_f32(
            struct ggml_context          * ctx,
            struct ggml_tensor           * a,
                   ggml_custom1_op_f32_t   fun),
        "use ggml_map_custom1 instead");
    
    # 使用 GGML_DEPRECATED 宏标记 ggml_map_custom1_inplace_f32 函数，提示使用 ggml_map_custom1_inplace 替代
    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom1_inplace_f32(
            struct ggml_context          * ctx,
            struct ggml_tensor           * a,
                   ggml_custom1_op_f32_t   fun),
        "use ggml_map_custom1_inplace instead");
    
    # 使用 GGML_DEPRECATED 宏标记 ggml_map_custom2_f32 函数，提示使用 ggml_map_custom2 替代
    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom2_f32(
            struct ggml_context          * ctx,
            struct ggml_tensor           * a,
            struct ggml_tensor           * b,
                   ggml_custom2_op_f32_t   fun),
        "use ggml_map_custom2 instead");
    # 使用 GGML_DEPRECATED 宏标记 ggml_map_custom2_inplace_f32 函数已经过时，建议使用 ggml_map_custom2_inplace 函数
    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom2_inplace_f32(
            struct ggml_context          * ctx,
            struct ggml_tensor           * a,
            struct ggml_tensor           * b,
                   ggml_custom2_op_f32_t   fun),
        "use ggml_map_custom2_inplace instead");

    # 使用 GGML_DEPRECATED 宏标记 ggml_map_custom3_f32 函数已经过时，建议使用 ggml_map_custom3 函数
    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom3_f32(
            struct ggml_context          * ctx,
            struct ggml_tensor           * a,
            struct ggml_tensor           * b,
            struct ggml_tensor           * c,
                   ggml_custom3_op_f32_t   fun),
        "use ggml_map_custom3 instead");

    # 使用 GGML_DEPRECATED 宏标记 ggml_map_custom3_inplace_f32 函数已经过时，建议使用 ggml_map_custom3_inplace 函数
    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom3_inplace_f32(
            struct ggml_context          * ctx,
            struct ggml_tensor           * a,
            struct ggml_tensor           * b,
            struct ggml_tensor           * c,
                   ggml_custom3_op_f32_t   fun),
        "use ggml_map_custom3_inplace instead");

    // custom operators v2

    # 定义 ggml_custom1_op_t 类型的函数指针
    typedef void (*ggml_custom1_op_t)(struct ggml_tensor * dst , const struct ggml_tensor * a, int ith, int nth, void * userdata);
    # 定义 ggml_custom2_op_t 类型的函数指针
    typedef void (*ggml_custom2_op_t)(struct ggml_tensor * dst , const struct ggml_tensor * a, const struct ggml_tensor * b, int ith, int nth, void * userdata);
    # 定义 ggml_custom3_op_t 类型的函数指针
    typedef void (*ggml_custom3_op_t)(struct ggml_tensor * dst , const struct ggml_tensor * a, const struct ggml_tensor * b, const struct ggml_tensor * c, int ith, int nth, void * userdata);

    # 定义 GGML_N_TASKS_MAX 常量为 -1
    #define GGML_N_TASKS_MAX -1

    # 使用 GGML_API 定义 ggml_map_custom1 函数，接受指向 ggml_custom1_op_t 函数的指针作为参数
    GGML_API struct ggml_tensor * ggml_map_custom1(
            struct ggml_context   * ctx,
            struct ggml_tensor    * a,
            ggml_custom1_op_t       fun,
            int                     n_tasks,
            void                  * userdata);
    # 定义一个函数，用于对一个张量进行自定义操作，并在原地修改
    GGML_API struct ggml_tensor * ggml_map_custom1_inplace(
            struct ggml_context   * ctx,
            struct ggml_tensor    * a,
            ggml_custom1_op_t       fun,
            int                     n_tasks,
            void                  * userdata);
    
    # 定义一个函数，用于对两个张量进行自定义操作
    GGML_API struct ggml_tensor * ggml_map_custom2(
            struct ggml_context   * ctx,
            struct ggml_tensor    * a,
            struct ggml_tensor    * b,
            ggml_custom2_op_t       fun,
            int                     n_tasks,
            void                  * userdata);
    
    # 定义一个函数，用于对两个张量进行自定义操作，并在原地修改
    GGML_API struct ggml_tensor * ggml_map_custom2_inplace(
            struct ggml_context   * ctx,
            struct ggml_tensor    * a,
            struct ggml_tensor    * b,
            ggml_custom2_op_t       fun,
            int                     n_tasks,
            void                  * userdata);
    
    # 定义一个函数，用于对三个张量进行自定义操作
    GGML_API struct ggml_tensor * ggml_map_custom3(
            struct ggml_context   * ctx,
            struct ggml_tensor    * a,
            struct ggml_tensor    * b,
            struct ggml_tensor    * c,
            ggml_custom3_op_t       fun,
            int                     n_tasks,
            void                  * userdata);
    
    # 定义一个函数，用于对三个张量进行自定义操作，并在原地修改
    GGML_API struct ggml_tensor * ggml_map_custom3_inplace(
            struct ggml_context   * ctx,
            struct ggml_tensor    * a,
            struct ggml_tensor    * b,
            struct ggml_tensor    * c,
            ggml_custom3_op_t       fun,
            int                     n_tasks,
            void                  * userdata);
    
    # 定义一个函数，用于计算交叉熵损失函数
    GGML_API struct ggml_tensor * ggml_cross_entropy_loss(
            struct ggml_context         * ctx,
            struct ggml_tensor          * a,
            struct ggml_tensor          * b);
    # 定义一个函数，计算交叉熵损失的反向传播
    GGML_API struct ggml_tensor * ggml_cross_entropy_loss_back(
            struct ggml_context         * ctx,
            struct ggml_tensor          * a,
            struct ggml_tensor          * b,
            struct ggml_tensor          * c);

    //
    // 自动微分
    //

    # 设置参数的梯度
    GGML_API void ggml_set_param(
            struct ggml_context * ctx,
            struct ggml_tensor  * tensor);


    # 构建前向扩展
    GGML_API void ggml_build_forward_expand (struct ggml_cgraph * cgraph, struct ggml_tensor * tensor);
    # 构建后向扩展
    GGML_API void ggml_build_backward_expand(struct ggml_context * ctx, struct ggml_cgraph * gf, struct ggml_cgraph * gb, bool keep);

    # 在上下文中分配图
    GGML_API struct ggml_cgraph * ggml_new_graph         (struct ggml_context * ctx); // size = GGML_DEFAULT_GRAPH_SIZE, grads = false
    # 在上下文中自定义分配图
    GGML_API struct ggml_cgraph * ggml_new_graph_custom  (struct ggml_context * ctx, size_t size, bool grads);
    # 复制图
    GGML_API struct ggml_cgraph * ggml_graph_dup         (struct ggml_context * ctx, struct ggml_cgraph * cgraph);
    # 查看图
    GGML_API struct ggml_cgraph * ggml_graph_view        (struct ggml_context * ctx, struct ggml_cgraph * cgraph, int i0, int i1);
    # 复制图
    GGML_API void                 ggml_graph_cpy         (struct ggml_cgraph * src, struct ggml_cgraph * dst);
    # 重置图
    GGML_API void                 ggml_graph_reset       (struct ggml_cgraph * cgraph);  // zero grads
    # 清空图
    GGML_API void                 ggml_graph_clear       (struct ggml_cgraph * cgraph);

    # 计算图的开销
    GGML_API size_t ggml_graph_overhead(void);
    # 自定义计算图的开销
    GGML_API size_t ggml_graph_overhead_custom(size_t size, bool grads);

    # 在计算图计算之前调用 ggml_graph_plan()
    # 当 plan.work_size > 0 时，调用者必须为 plan.work_data 分配内存
    GGML_API struct ggml_cplan ggml_graph_plan   (struct ggml_cgraph * cgraph, int n_threads /*= GGML_DEFAULT_N_THREADS*/);
    # 计算图
    GGML_API int               ggml_graph_compute(struct ggml_cgraph * cgraph, struct ggml_cplan * cplan);
    // 使用上下文分配工作数据的方式执行图计算，需要确保上下文有足够的内存来存储工作数据
    GGML_API void ggml_graph_compute_with_ctx(struct ggml_context * ctx, struct ggml_cgraph * cgraph, int n_threads);
    
    // 根据名称从计算图中获取张量
    GGML_API struct ggml_tensor * ggml_graph_get_tensor(struct ggml_cgraph * cgraph, const char * name);
    
    // 将计算图导出到文件
    GGML_API void ggml_graph_export(const struct ggml_cgraph * cgraph, const char * fname);
    
    // 从文件中导入计算图，并返回上下文数据和评估数据
    GGML_API struct ggml_cgraph * ggml_graph_import(const char * fname, struct ggml_context ** ctx_data, struct ggml_context ** ctx_eval);
    
    // 打印图的信息和性能信息
    GGML_API void ggml_graph_print(const struct ggml_cgraph * cgraph);
    
    // 使用dot格式将图转储到文件
    GGML_API void ggml_graph_dump_dot(const struct ggml_cgraph * gb, const struct ggml_cgraph * gf, const char * filename);
    
    // 使用提供的检查点构建梯度检查点后向图gb，gb_tmp将包含具有重写后向过程节点的原始后向图，但不包括第二次前向传递节点
    GGML_API void ggml_build_backward_gradient_checkpointing(
            struct ggml_context   * ctx,
            struct ggml_cgraph    * gf,
            struct ggml_cgraph    * gb,
            struct ggml_cgraph    * gb_tmp,
            struct ggml_tensor  * * checkpoints,
            int                     n_checkpoints);
    
    // 优化方法
    enum ggml_opt_type {
        GGML_OPT_ADAM,
        GGML_OPT_LBFGS,
    };
    
    // 线搜索方法
    enum ggml_linesearch {
        GGML_LINESEARCH_DEFAULT = 1,
        GGML_LINESEARCH_BACKTRACKING_ARMIJO       = 0,
        GGML_LINESEARCH_BACKTRACKING_WOLFE        = 1,
        GGML_LINESEARCH_BACKTRACKING_STRONG_WOLFE = 2,
    };
    
    // 优化返回值
    # 定义枚举类型，表示优化结果
    enum ggml_opt_result {
        GGML_OPT_OK = 0,  # 优化成功
        GGML_OPT_DID_NOT_CONVERGE,  # 优化未收敛
        GGML_OPT_NO_CONTEXT,  # 无上下文
        GGML_OPT_INVALID_WOLFE,  # 无效的 Wolfe 条件
        GGML_OPT_FAIL,  # 优化失败
        GGML_OPT_CANCEL,  # 优化被取消

        GGML_LINESEARCH_FAIL = -128,  # 线搜索失败
        GGML_LINESEARCH_MINIMUM_STEP,  # 最小步长
        GGML_LINESEARCH_MAXIMUM_STEP,  # 最大步长
        GGML_LINESEARCH_MAXIMUM_ITERATIONS,  # 最大迭代次数
        GGML_LINESEARCH_INVALID_PARAMETERS,  # 无效参数
    };

    # 定义回调函数指针类型，用于优化过程中的回调
    typedef void (*ggml_opt_callback)(void * data, int accum_step, float * sched, bool * cancel);
    typedef void (*ggml_log_callback)(enum ggml_log_level level, const char * text, void * user_data);

    # 优化参数
    #
    #   参见 ggml.c (ggml_opt_default_params) 获取默认值
    #
    # 定义结构体 ggml_opt_params，包含了一系列优化参数
    struct ggml_opt_params {
        # 枚举类型，表示优化类型
        enum ggml_opt_type type;

        # 图的大小
        size_t graph_size;

        # 线程数
        int n_threads;

        # 基于 delta 的收敛测试
        #
        #   如果 past == 0 - 禁用
        #   如果 past > 0:
        #     如果 |f(x) - f(x_past)| < delta * max(1, |f(x)|)，则停止
        #
        int past;
        float delta;

        # 最大迭代次数，没有改进时的最大迭代次数
        #
        #   如果 0 - 禁用
        #   如果 > 0:
        #     如果在这个迭代次数内没有成本改进，则假定收敛
        #
        int max_no_improvement;

        # 是否打印正向图和反向图
        bool print_forward_graph;
        bool print_backward_graph;

        # 梯度累积次数
        int n_gradient_accumulation;

        # ADAM 参数
        struct {
            int n_iter;
            float sched;  # 调度乘数（固定、衰减或预热）
            float decay;  # AdamW 的权重衰减，使用 0.0f 禁用
            int decay_min_ndim;  # 应用权重衰减的张量维度的最小数量
            float alpha;  # 学习率
            float beta1;
            float beta2;
            float eps;    # 数值稳定性的 epsilon
            float eps_f;  # 收敛测试的 epsilon
            float eps_g;  # 收敛测试的 epsilon
            float gclip;  # 梯度裁剪
        } adam;

        # LBFGS 参数
        struct {
            int m;  # 近似逆 Hessian 的修正次数
            int n_iter;
            int max_linesearch;

            float eps;      # 收敛容限
            float ftol;     # 线搜索容限
            float wolfe;
            float min_step;
            float max_step;

            # 线搜索类型
            enum ggml_linesearch linesearch;
        } lbfgs;
    };
    # 定义了一个结构体，用于存储优化器的上下文信息
    struct ggml_opt_context {
        struct ggml_context * ctx;  // 指向神经网络上下文的指针
        struct ggml_opt_params params;  // 优化器参数

        int iter;  // 迭代次数
        int64_t nx; // 参数元素的数量

        bool just_initialized;  // 标记是否刚刚初始化

        float loss_before;  // 优化前的损失值
        float loss_after;   // 优化后的损失值

        struct {
            struct ggml_tensor * g;  // 当前梯度
            struct ggml_tensor * m;  // 第一时刻
            struct ggml_tensor * v;  // 第二时刻
            struct ggml_tensor * pf; // 过去的函数值
            float fx_best;  // 最佳函数值
            float fx_prev;  // 上一个函数值
            int n_no_improvement;  // 未改善次数
        } adam;

        struct {
            struct ggml_tensor * x;    // 当前参数
            struct ggml_tensor * xp;   // 上一个参数
            struct ggml_tensor * g;    // 当前梯度
            struct ggml_tensor * gp;   // 上一个梯度
            struct ggml_tensor * d;    // 搜索方向
            struct ggml_tensor * pf;   // 过去的函数值
            struct ggml_tensor * lmal; // L-BFGS 内存 alpha
            struct ggml_tensor * lmys; // L-BFGS 内存 ys
            struct ggml_tensor * lms;  // L-BFGS 内存 s
            struct ggml_tensor * lmy;  // L-BFGS 内存 y
            float fx_best;  // 最佳函数值
            float step;  // 步长
            int j;  // 迭代次数
            int k;  // 迭代次数
            int end;  // 结束标志
            int n_no_improvement;  // 未改善次数
        } lbfgs;
    };

    # 获取默认的优化器参数
    GGML_API struct ggml_opt_params ggml_opt_default_params(enum ggml_opt_type type);

    # 优化定义在张量 f 上的函数
    GGML_API enum ggml_opt_result ggml_opt(
            struct ggml_context * ctx,
            struct ggml_opt_params params,
            struct ggml_tensor * f);

    # 初始化优化器上下文
    GGML_API void ggml_opt_init(
            struct ggml_context     * ctx,
            struct ggml_opt_context * opt,
            struct ggml_opt_params    params,
            int64_t                   nx);
    // 继续优化由张量 f 定义的函数
    GGML_API enum ggml_opt_result ggml_opt_resume(
            struct ggml_context * ctx,
            struct ggml_opt_context * opt,
            struct ggml_tensor * f);

    // 继续优化由张量 f 定义的函数，同时传入计算图 gf 和反向传播计算图 gb，以及回调函数和回调数据
    GGML_API enum ggml_opt_result ggml_opt_resume_g(
            struct ggml_context * ctx,
            struct ggml_opt_context * opt,
            struct ggml_tensor * f,
            struct ggml_cgraph * gf,
            struct ggml_cgraph * gb,
            ggml_opt_callback callback,
            void * callback_data);

    //
    // 量化
    //

    // TODO: 这些可能会被更通用的 ggml_quantize_chunk 替代
    // 将浮点数组 src 量化为 Q4.0 格式，存储到 dst 中，同时计算直方图 hist
    GGML_API size_t ggml_quantize_q4_0(const float * src, void * dst, int n, int k, int64_t * hist);
    // 将浮点数组 src 量化为 Q4.1 格式，存储到 dst 中，同时计算直方图 hist
    GGML_API size_t ggml_quantize_q4_1(const float * src, void * dst, int n, int k, int64_t * hist);
    // 将浮点数组 src 量化为 Q5.0 格式，存储到 dst 中，同时计算直方图 hist
    GGML_API size_t ggml_quantize_q5_0(const float * src, void * dst, int n, int k, int64_t * hist);
    // 将浮点数组 src 量化为 Q5.1 格式，存储到 dst 中，同时计算直方图 hist
    GGML_API size_t ggml_quantize_q5_1(const float * src, void * dst, int n, int k, int64_t * hist);
    // 将浮点数组 src 量化为 Q8.0 格式，存储到 dst 中，同时计算直方图 hist
    GGML_API size_t ggml_quantize_q8_0(const float * src, void * dst, int n, int k, int64_t * hist);

    // 将浮点数组 src 量化为 Q2_K 格式，存储到 dst 中，同时计算直方图 hist
    GGML_API size_t ggml_quantize_q2_K(const float * src, void * dst, int n, int k, int64_t * hist);
    // 将浮点数组 src 量化为 Q3_K 格式，存储到 dst 中，同时计算直方图 hist
    GGML_API size_t ggml_quantize_q3_K(const float * src, void * dst, int n, int k, int64_t * hist);
    // 将浮点数组 src 量化为 Q4_K 格式，存储到 dst 中，同时计算直方图 hist
    GGML_API size_t ggml_quantize_q4_K(const float * src, void * dst, int n, int k, int64_t * hist);
    // 将浮点数组 src 量化为 Q5_K 格式，存储到 dst 中，同时计算直方图 hist
    GGML_API size_t ggml_quantize_q5_K(const float * src, void * dst, int n, int k, int64_t * hist);
    // 将浮点数组 src 量化为 Q6_K 格式，存储到 dst 中，同时计算直方图 hist
    GGML_API size_t ggml_quantize_q6_K(const float * src, void * dst, int n, int k, int64_t * hist);

    // 将浮点数组 src 的一部分量化为指定类型的格式，存储到 dst 中，同时计算直方图 hist
    GGML_API size_t ggml_quantize_chunk(enum ggml_type type, const float * src, void * dst, int start, int n, int64_t * hist);

    //
    // gguf
    //
    # 定义枚举类型 gguf_type，包含不同数据类型的枚举值
    enum gguf_type {
        GGUF_TYPE_UINT8   = 0,  # 无符号8位整数
        GGUF_TYPE_INT8    = 1,  # 有符号8位整数
        GGUF_TYPE_UINT16  = 2,  # 无符号16位整数
        GGUF_TYPE_INT16   = 3,  # 有符号16位整数
        GGUF_TYPE_UINT32  = 4,  # 无符号32位整数
        GGUF_TYPE_INT32   = 5,  # 有符号32位整数
        GGUF_TYPE_FLOAT32 = 6,  # 32位浮点数
        GGUF_TYPE_BOOL    = 7,  # 布尔类型
        GGUF_TYPE_STRING  = 8,  # 字符串类型
        GGUF_TYPE_ARRAY   = 9,  # 数组类型
        GGUF_TYPE_UINT64  = 10, # 无符号64位整数
        GGUF_TYPE_INT64   = 11, # 有符号64位整数
        GGUF_TYPE_FLOAT64 = 12, # 64位浮点数
        GGUF_TYPE_COUNT,        // 标记枚举结束
    };
    
    # 定义结构体 gguf_context
    struct gguf_context;
    
    # 定义结构体 gguf_init_params，用于初始化参数
    struct gguf_init_params {
        bool no_alloc;  # 是否不分配内存
    
        // 如果不为 NULL，则创建一个 ggml_context 并在其中分配张量数据
        struct ggml_context ** ctx;
    };
    
    # 定义函数 gguf_init_empty，用于初始化空的 gguf_context
    GGML_API struct gguf_context * gguf_init_empty(void);
    
    # 定义函数 gguf_init_empty_sparse，用于初始化稀疏的 gguf_context
    GGML_API struct gguf_context * gguf_init_empty_sparse(void);
    
    # 定义函数 gguf_init_from_file，从文件中初始化 gguf_context，并传入初始化参数
    GGML_API struct gguf_context * gguf_init_from_file(const char * fname, struct gguf_init_params params);
    
    # 定义函数 gguf_free，用于释放 gguf_context
    GGML_API void gguf_free(struct gguf_context * ctx);
    
    # 定义函数 gguf_type_name，用于获取枚举类型对应的名称
    GGML_API const char * gguf_type_name(enum gguf_type type);
    
    # 定义函数 gguf_get_version，用于获取 gguf_context 的版本
    GGML_API int    gguf_get_version    (const struct gguf_context * ctx);
    
    # 定义函数 gguf_get_alignment，用于获取 gguf_context 的对齐方式
    GGML_API size_t gguf_get_alignment  (const struct gguf_context * ctx);
    
    # 定义函数 gguf_get_data_offset，用于获取 gguf_context 的数据偏移量
    GGML_API size_t gguf_get_data_offset(const struct gguf_context * ctx);
    
    # 定义函数 gguf_get_data，用于获取 gguf_context 的数据
    GGML_API void * gguf_get_data       (const struct gguf_context * ctx);
    
    # 定义函数 gguf_get_n_kv，用于获取 gguf_context 的键值对数量
    GGML_API int          gguf_get_n_kv(const struct gguf_context * ctx);
    
    # 定义函数 gguf_find_key，用于查找 gguf_context 中指定键的索引
    GGML_API int          gguf_find_key(const struct gguf_context * ctx, const char * key);
    
    # 定义函数 gguf_get_key，用于获取 gguf_context 中指定索引的键
    GGML_API const char * gguf_get_key (const struct gguf_context * ctx, int key_id);
    
    # 定义函数 gguf_get_kv_type，用于获取 gguf_context 中指定键值对的类型
    GGML_API enum gguf_type gguf_get_kv_type (const struct gguf_context * ctx, int key_id);
    
    # 定义函数 gguf_get_arr_type，用于获取 gguf_context 中指定键值对的数组类型
    GGML_API enum gguf_type gguf_get_arr_type(const struct gguf_context * ctx, int key_id);
    
    # 如果使用了错误的类型作为键，则会中止程序
    # 从 gguf_context 结构中获取指定键对应的无符号 8 位整数值
    GGML_API uint8_t gguf_get_val_u8 (const struct gguf_context * ctx, int key_id);
    # 从 gguf_context 结构中获取指定键对应的有符号 8 位整数值
    GGML_API int8_t gguf_get_val_i8 (const struct gguf_context * ctx, int key_id);
    # 从 gguf_context 结构中获取指定键对应的无符号 16 位整数值
    GGML_API uint16_t gguf_get_val_u16 (const struct gguf_context * ctx, int key_id);
    # 从 gguf_context 结构中获取指定键对应的有符号 16 位整数值
    GGML_API int16_t gguf_get_val_i16 (const struct gguf_context * ctx, int key_id);
    # 从 gguf_context 结构中获取指定键对应的无符号 32 位整数值
    GGML_API uint32_t gguf_get_val_u32 (const struct gguf_context * ctx, int key_id);
    # 从 gguf_context 结构中获取指定键对应的有符号 32 位整数值
    GGML_API int32_t gguf_get_val_i32 (const struct gguf_context * ctx, int key_id);
    # 从 gguf_context 结构中获取指定键对应的 32 位浮点数值
    GGML_API float gguf_get_val_f32 (const struct gguf_context * ctx, int key_id);
    # 从 gguf_context 结构中获取指定键对应的无符号 64 位整数值
    GGML_API uint64_t gguf_get_val_u64 (const struct gguf_context * ctx, int key_id);
    # 从 gguf_context 结构中获取指定键对应的有符号 64 位整数值
    GGML_API int64_t gguf_get_val_i64 (const struct gguf_context * ctx, int key_id);
    # 从 gguf_context 结构中获取指定键对应的 64 位浮点数值
    GGML_API double gguf_get_val_f64 (const struct gguf_context * ctx, int key_id);
    # 从 gguf_context 结构中获取指定键对应的布尔值
    GGML_API bool gguf_get_val_bool (const struct gguf_context * ctx, int key_id);
    # 从 gguf_context 结构中获取指定键对应的字符串值
    GGML_API const char * gguf_get_val_str (const struct gguf_context * ctx, int key_id);
    # 从 gguf_context 结构中获取指定键对应的数组的元素个数
    GGML_API int gguf_get_arr_n (const struct gguf_context * ctx, int key_id);
    # 从 gguf_context 结构中获取指定键对应的数组的数据
    GGML_API const void * gguf_get_arr_data (const struct gguf_context * ctx, int key_id);
    # 从 gguf_context 结构中获取指定键对应的数组的指定索引处的字符串值
    GGML_API const char * gguf_get_arr_str (const struct gguf_context * ctx, int key_id, int i);
    
    # 从 gguf_context 结构中获取稀疏导数的类型
    GGML_API enum ggml_sparse_deriv gguf_get_sparse_deriv (const struct gguf_context * ctx);
    # 从 gguf_context 结构中获取张量的个数
    GGML_API int gguf_get_n_tensors (const struct gguf_context * ctx);
    # 在 gguf_context 结构中查找指定名称的张量
    GGML_API int gguf_find_tensor (const struct gguf_context * ctx, const char * name);
    # 从 gguf_context 结构中获取指定索引处张量的偏移量
    GGML_API size_t gguf_get_tensor_offset (const struct gguf_context * ctx, int i);
    # 从 gguf_context 结构中获取指定索引处张量的名称
    GGML_API char * gguf_get_tensor_name (const struct gguf_context * ctx, int i);
    
    # 设置 gguf_context 结构中指定键对应的无符号 8 位整数值，如果键不存在则添加新的键值对
    GGML_API void gguf_set_val_u8 (struct gguf_context * ctx, const char * key, uint8_t val);
    // 设置 int8_t 类型数值到上下文中的指定键
    GGML_API void gguf_set_val_i8  (struct gguf_context * ctx, const char * key, int8_t   val);
    // 设置 uint16_t 类型数值到上下文中的指定键
    GGML_API void gguf_set_val_u16 (struct gguf_context * ctx, const char * key, uint16_t val);
    // 设置 int16_t 类型数值到上下文中的指定键
    GGML_API void gguf_set_val_i16 (struct gguf_context * ctx, const char * key, int16_t  val);
    // 设置 uint32_t 类型数值到上下文中的指定键
    GGML_API void gguf_set_val_u32 (struct gguf_context * ctx, const char * key, uint32_t val);
    // 设置 int32_t 类型数值到上下文中的指定键
    GGML_API void gguf_set_val_i32 (struct gguf_context * ctx, const char * key, int32_t  val);
    // 设置 float 类型数值到上下文中的指定键
    GGML_API void gguf_set_val_f32 (struct gguf_context * ctx, const char * key, float    val);
    // 设置 uint64_t 类型数值到上下文中的指定键
    GGML_API void gguf_set_val_u64 (struct gguf_context * ctx, const char * key, uint64_t val);
    // 设置 int64_t 类型数值到上下文中的指定键
    GGML_API void gguf_set_val_i64 (struct gguf_context * ctx, const char * key, int64_t  val);
    // 设置 double 类型数值到上下文中的指定键
    GGML_API void gguf_set_val_f64 (struct gguf_context * ctx, const char * key, double   val);
    // 设置 bool 类型数值到上下文中的指定键
    GGML_API void gguf_set_val_bool(struct gguf_context * ctx, const char * key, bool     val);
    // 设置字符串类型数值到上下文中的指定键
    GGML_API void gguf_set_val_str (struct gguf_context * ctx, const char * key, const char * val);
    // 设置数组数据到上下文中的指定键
    GGML_API void gguf_set_arr_data(struct gguf_context * ctx, const char * key, enum gguf_type type, const void * data, int n);
    // 设置字符串数组到上下文中的指定键
    GGML_API void gguf_set_arr_str (struct gguf_context * ctx, const char * key, const char ** data, int n);

    // 从另一个上下文中设置或添加键值对
    GGML_API void gguf_set_kv(struct gguf_context * ctx, struct gguf_context * src);

    // 管理张量信息
    GGML_API void gguf_add_tensor(struct gguf_context * ctx, const struct ggml_tensor * tensor);
    // 设置张量类型
    GGML_API void gguf_set_tensor_type(struct gguf_context * ctx, const char * name, enum ggml_type type);
    // 设置张量数据
    GGML_API void gguf_set_tensor_data(struct gguf_context * ctx, const char * name, const void * data, size_t size);

    // 可以通过两种方式进行 gguf 文件的写入：
    //
    // - 在单个步骤中将整个 gguf_context 写入二进制文件：
    //
    //   gguf_write_to_file(ctx, fname);
    //
    // 准备一个带有元数据占位符的文件，先写入张量数据，然后再写入元数据：
    //
    //   打开一个二进制文件用于写入
    //   FILE * f = fopen(fname, "wb");
    //   将文件指针移动到指定位置，用于写入元数据
    //   fseek(f, gguf_get_meta_size(ctx), SEEK_SET);
    //   写入数据到文件
    //   fwrite(f, ...);
    //   获取元数据的指针
    //   void * data = gguf_meta_get_meta_data(ctx);
    //   将文件指针移动到文件开头
    //   fseek(f, 0, SEEK_SET);
    //   写入元数据到文件
    //   fwrite(f, data, gguf_get_meta_size(ctx));
    //   释放元数据指针的内存
    //   free(data);
    //   关闭文件
    //   fclose(f);
    //
    
    // 将整个上下文写入二进制文件
    GGML_API void gguf_write_to_file(const struct gguf_context * ctx, const char * fname, bool only_meta);
    
    // 获取元数据（头部、键值对、张量信息）的字节大小，包括填充
    GGML_API size_t gguf_get_meta_size(const struct gguf_context * ctx);
    GGML_API void   gguf_get_meta_data(const struct gguf_context * ctx, void * data);
    
    //
    // 系统信息
    //
    
    GGML_API int ggml_cpu_has_avx        (void);
    GGML_API int ggml_cpu_has_avx2       (void);
    GGML_API int ggml_cpu_has_avx512     (void);
    GGML_API int ggml_cpu_has_avx512_vbmi(void);
    GGML_API int ggml_cpu_has_avx512_vnni(void);
    GGML_API int ggml_cpu_has_fma        (void);
    GGML_API int ggml_cpu_has_neon       (void);
    GGML_API int ggml_cpu_has_arm_fma    (void);
    GGML_API int ggml_cpu_has_metal      (void);
    GGML_API int ggml_cpu_has_f16c       (void);
    GGML_API int ggml_cpu_has_fp16_va    (void);
    GGML_API int ggml_cpu_has_wasm_simd  (void);
    GGML_API int ggml_cpu_has_blas       (void);
    GGML_API int ggml_cpu_has_cublas     (void);
    GGML_API int ggml_cpu_has_clblast    (void);
    GGML_API int ggml_cpu_has_gpublas    (void);
    GGML_API int ggml_cpu_has_sse3       (void);
    GGML_API int ggml_cpu_has_ssse3      (void);
    GGML_API int ggml_cpu_has_vsx        (void);
    
    //
    // 全局变量
    // 
    // TODO: 这些应该被移动到上下文中
    extern float sparse_pred_threshold;
    
    //
    // 为测试和基准测试暴露的内部类型和函数
    //
#ifdef  __cplusplus
// 如果是 C++ 环境，则定义 GGML_RESTRICT 为空
#define GGML_RESTRICT
#else
// 如果不是 C++ 环境，则定义 GGML_RESTRICT 为 restrict
#define GGML_RESTRICT restrict
#endif

// 定义函数指针类型 ggml_to_float_t，接受一个指向常量 void 类型的指针和一个指向 float 类型的指针，以及一个整数 k
typedef void (*ggml_to_float_t)  (const void  * GGML_RESTRICT x, float * GGML_RESTRICT y, int k);
// 定义函数指针类型 ggml_from_float_t，接受一个指向常量 float 类型的指针和一个指向 void 类型的指针，以及一个整数 k
typedef void (*ggml_from_float_t)(const float * GGML_RESTRICT x, void  * GGML_RESTRICT y, int k);
// 定义函数指针类型 ggml_vec_dot_t，接受一个整数 n，一个指向 float 类型的指针 s，以及两个指向常量 void 类型的指针 x 和 y
typedef void (*ggml_vec_dot_t)   (const int n, float * GGML_RESTRICT s, const void * GGML_RESTRICT x, const void * GGML_RESTRICT y);

// 定义结构体 ggml_type_traits_t，包含类型名、块大小、类型大小、是否量化、转换函数指针等信息
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

// 声明函数 ggml_internal_get_type_traits，接受一个枚举类型 ggml_type，返回 ggml_type_traits_t 结构体
GGML_API ggml_type_traits_t ggml_internal_get_type_traits(enum ggml_type type);

#ifdef  __cplusplus
}
#endif
```