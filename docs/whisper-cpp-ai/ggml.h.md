# `whisper.cpp\ggml.h`

```cpp
#pragma once

// GGML Tensor Library
// GGML 张量库

// This documentation is still a work in progress.
// If you wish some specific topics to be covered, feel free to drop a comment:
// 此文档仍在进行中。
// 如果您希望涵盖某些特定主题，请随时留下评论：
// https://github.com/ggerganov/whisper.cpp/issues/40

// ## Overview
// ## 概述

// This library implements:
//  - a set of tensor operations
//  - automatic differentiation
//  - basic optimization algorithms
// 该库实现了：
//  - 一组张量操作
//  - 自动微分
//  - 基本优化算法

// The aim of this library is to provide a minimalistic approach for various machine learning tasks. This includes,
// but is not limited to, the following:
//  - linear regression
//  - support vector machines
//  - neural networks
// 该库的目的是为各种机器学习任务提供一种极简主义的方法。这包括但不限于以下内容：
//  - 线性回归
//  - 支持向量机
//  - 神经网络

// The library allows the user to define a certain function using the available tensor operations. This function
// definition is represented internally via a computation graph. Each tensor operation in the function definition
// corresponds to a node in the graph. Having the computation graph defined, the user can choose to compute the
// function's value and/or its gradient with respect to the input variables. Optionally, the function can be optimized
// using one of the available optimization algorithms.
// 该库允许用户使用可用的张量操作定义某个函数。该函数定义通过计算图在内部表示。函数定义中的每个张量操作对应于图中的一个节点。一旦定义了计算图，用户可以选择计算函数的值和/或相对于输入变量的梯度。可选地，可以使用其中一种可用的优化算法对函数进行优化。

// For example, here we define the function: f(x) = a*x^2 + b
// 例如，在这里我们定义函数：f(x) = a*x^2 + b
{
    struct ggml_init_params params = {
        .mem_size   = 16*1024*1024,
        .mem_buffer = NULL,
    };

    // memory allocation happens here
    // 内存分配发生在这里
    struct ggml_context * ctx = ggml_init(params);

    struct ggml_tensor * x = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 1);

    ggml_set_param(ctx, x); // x is an input variable
    // 将 x 设置为输入变量

    struct ggml_tensor * a  = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 1);
    struct ggml_tensor * b  = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 1);
    struct ggml_tensor * x2 = ggml_mul(ctx, x, x);
    struct ggml_tensor * f  = ggml_add(ctx, ggml_mul(ctx, a, x2), b);

    ...
}
// Notice that the function definition above does not involve any actual computation. The computation is performed only
// when the user explicitly requests it. For example, to compute the function's value at x = 2.0:
//
//   {
//       ...
//
//       struct ggml_cgraph * gf = ggml_new_graph(ctx);
//       ggml_build_forward_expand(gf, f);
//
//       // set the input variable and parameter values
//       ggml_set_f32(x, 2.0f);
//       ggml_set_f32(a, 3.0f);
//       ggml_set_f32(b, 4.0f);
//
//       ggml_graph_compute_with_ctx(ctx, &gf, n_threads);
//
//       printf("f = %f\n", ggml_get_f32_1d(f, 0));
//
//       ...
//   }
//
// The actual computation is performed in the ggml_graph_compute() function.
//
// The ggml_new_tensor_...() functions create new tensors. They are allocated in the memory buffer provided to the
// ggml_init() function. You have to be careful not to exceed the memory buffer size. Therefore, you have to know
// in advance how much memory you need for your computation. Alternatively, you can allocate a large enough memory
// and after defining the computation graph, call the ggml_used_mem() function to find out how much memory was
// actually needed.
//
// The ggml_set_param() function marks a tensor as an input variable. This is used by the automatic
// differentiation and optimization algorithms.
//
// The described approach allows to define the function graph once and then compute its forward or backward graphs
// multiple times. All computations will use the same memory buffer allocated in the ggml_init() function. This way
// the user can avoid the memory allocation overhead at runtime.
//
// The library supports multi-dimensional tensors - up to 4 dimensions. The FP16 and FP32 data types are first class
// citizens, but in theory the library can be extended to support FP8 and integer data types.
//
// Each tensor operation produces a new tensor. Initially the library was envisioned to support only the use of unary
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
// 定义结构体指针 a，创建一个二维浮点型张量，大小为 nx * ny
// 遍历二维张量 a，将每个位置的值设置为 x + y
// 使用 ggml_get_f32_1d() 和 ggml_set_f32_1d() 等辅助函数进行操作
// 定义矩阵乘法运算符 ggml_mul_mat
// 多线程处理
// ggml.c 概述
// SIMD 优化
// 调试 ggml
// 根据 GGML_SHARED 宏定义 GGML_API，根据操作系统定义不同的导出/导入方式
// 根据 GGML_MULTIPLATFORM 宏定义 GGML_CALL，根据操作系统定义不同的调用约定
// 根据编译器类型定义 GGML_DEPRECATED，用于标记函数已废弃
// 根据编译器类型定义 GGML_ATTRIBUTE_FORMAT，用于格式化输出
// 包含标准库头文件
// 定义 GGML 文件的魔数和版本号
// 定义量化格式的版本号
// 定义 GGML_QNT_VERSION_FACTOR 常量为 1000，不要更改
#define GGML_QNT_VERSION_FACTOR 1000 // do not change this

// 定义最大维度为 4
#define GGML_MAX_DIMS           4
// 定义最大参数数量为 2048
#define GGML_MAX_PARAMS         2048
// 定义最大上下文数量为 64
#define GGML_MAX_CONTEXTS       64
// 定义最大源数量为 10
#define GGML_MAX_SRC            10
// 如果未定义 GGML_MAX_NAME，则定义为 64
#ifndef GGML_MAX_NAME
#define GGML_MAX_NAME           64
#endif
// 定义最大操作参数数量为 64
#define GGML_MAX_OP_PARAMS      64
// 默认线程数为 4
#define GGML_DEFAULT_N_THREADS  4
// 默认图大小为 2048
#define GGML_DEFAULT_GRAPH_SIZE 2048
// 根据指针大小定义内存对齐方式
#if UINTPTR_MAX == 0xFFFFFFFF
    #define GGML_MEM_ALIGN 4
#else
    #define GGML_MEM_ALIGN 16
#endif

// 定义成功退出状态为 0
#define GGML_EXIT_SUCCESS 0
// 定义中止退出状态为 1
#define GGML_EXIT_ABORTED 1

// 定义 GGUF 魔数为 "GGUF"
#define GGUF_MAGIC "GGUF"

// 定义 GGUF 版本为 3
#define GGUF_VERSION 3

// 默认对齐方式为 32
#define GGUF_DEFAULT_ALIGNMENT 32

// 定义 GGML_UNUSED 宏，用于消除未使用变量的编译警告
#define GGML_UNUSED(x) (void)(x)

// 定义 GGML_PAD 宏，用于对齐 x 到 n 的倍数
#define GGML_PAD(x, n) (((x) + (n) - 1) & ~((n) - 1))

// 定义 GGML_ASSERT 宏，用于断言条件 x 是否为真，否则输出错误信息并中止程序
#define GGML_ASSERT(x) \
    do { \
        if (!(x)) { \
            fflush(stdout); \
            fprintf(stderr, "GGML_ASSERT: %s:%d: %s\n", __FILE__, __LINE__, #x); \
            ggml_print_backtrace(); \
            abort(); \
        } \
    } while (0)

// 根据不同编译器定义 GGML_UNREACHABLE 宏，用于标记不应到达的代码
#ifndef NDEBUG
#define GGML_UNREACHABLE() GGML_ASSERT(!"statement should not be reached")
#elif defined(__GNUC__)
#define GGML_UNREACHABLE() __builtin_unreachable()
#elif defined(_MSC_VER)
#define GGML_UNREACHABLE() __assume(0)
#else
#define GGML_UNREACHABLE() ((void) 0)
#endif

// 用于将张量的元素数量和字节步长复制到本地变量中，减少代码重复和提高可读性
//
// 示例:
//
//    GGML_TENSOR_LOCALS(int64_t, ne1, src1, ne);
//    GGML_TENSOR_LOCALS(size_t,  nb1, src1, nb);
//
#define GGML_TENSOR_LOCALS_1(type, prefix, pointer, array) \
    const type prefix##0 = (pointer)->array[0]; \
    GGML_UNUSED(prefix##0);
#define GGML_TENSOR_LOCALS_2(type, prefix, pointer, array) \
    GGML_TENSOR_LOCALS_1    (type, prefix, pointer, array) \
    const type prefix##1 = (pointer)->array[1]; \
    GGML_UNUSED(prefix##1);
#define GGML_TENSOR_LOCALS_3(type, prefix, pointer, array) \
    GGML_TENSOR_LOCALS_2    (type, prefix, pointer, array) \
    // 定义一个常量类型为指针所指向的数组的第二个元素
    const type prefix##2 = (pointer)->array[2]; \
    // 使用宏定义的方式，避免编译器警告未使用的变量
    GGML_UNUSED(prefix##2);
#define GGML_TENSOR_LOCALS(type, prefix, pointer, array) \
    // 定义宏，用于声明局部变量并初始化，包括类型、前缀、指针和数组
    GGML_TENSOR_LOCALS_3  (type, prefix, pointer, array) \
    // 调用另一个宏，用于声明局部变量并初始化
    const type prefix##3 = (pointer)->array[3]; \
    // 声明并初始化一个常量
    GGML_UNUSED(prefix##3);

#define GGML_TENSOR_UNARY_OP_LOCALS \
    // 定义宏，用于声明一元操作的局部变量
    GGML_TENSOR_LOCALS(int64_t, ne0, src0, ne) \
    GGML_TENSOR_LOCALS(size_t,  nb0, src0, nb) \
    GGML_TENSOR_LOCALS(int64_t, ne,  dst,  ne) \
    GGML_TENSOR_LOCALS(size_t,  nb,  dst,  nb)

#define GGML_TENSOR_BINARY_OP_LOCALS \
    // 定义宏，用于声明二元操作的局部变量
    GGML_TENSOR_LOCALS(int64_t, ne0, src0, ne) \
    GGML_TENSOR_LOCALS(size_t,  nb0, src0, nb) \
    GGML_TENSOR_LOCALS(int64_t, ne1, src1, ne) \
    GGML_TENSOR_LOCALS(size_t,  nb1, src1, nb) \
    GGML_TENSOR_LOCALS(int64_t, ne,  dst,  ne) \
    GGML_TENSOR_LOCALS(size_t,  nb,  dst,  nb)

#ifdef  __cplusplus
extern "C" {
#endif

#if defined(__ARM_NEON) && defined(__CUDACC__)
    typedef half ggml_fp16_t;
#elif defined(__ARM_NEON) && !defined(_MSC_VER)
    typedef __fp16 ggml_fp16_t;
#else
    typedef uint16_t ggml_fp16_t;
#endif
    // 根据条件定义不同的数据类型 ggml_fp16_t

    // convert FP16 <-> FP32
    GGML_API float       ggml_fp16_to_fp32(ggml_fp16_t x);
    GGML_API ggml_fp16_t ggml_fp32_to_fp16(float x);
    // 定义函数原型，用于 FP16 和 FP32 之间的转换

    GGML_API void ggml_fp16_to_fp32_row(const ggml_fp16_t * x, float * y, int n);
    GGML_API void ggml_fp32_to_fp16_row(const float * x, ggml_fp16_t * y, int n);
    // 定义函数原型，用于行级别的 FP16 和 FP32 之间的转换

    struct ggml_object;
    struct ggml_context;
    // 声明结构体 ggml_object 和 ggml_context
    // 定义枚举类型 ggml_type，表示数据类型
    enum ggml_type {
        // 单精度浮点数
        GGML_TYPE_F32  = 0,
        // 半精度浮点数
        GGML_TYPE_F16  = 1,
        // 四位小数，整数部分占4位
        GGML_TYPE_Q4_0 = 2,
        GGML_TYPE_Q4_1 = 3,
        // 四位小数，整数部分占4位，已移除支持
        // 四位小数，整数部分占4位，已移除支持
        GGML_TYPE_Q5_0 = 6,
        GGML_TYPE_Q5_1 = 7,
        GGML_TYPE_Q8_0 = 8,
        GGML_TYPE_Q8_1 = 9,
        // k-quantizations
        GGML_TYPE_Q2_K = 10,
        GGML_TYPE_Q3_K = 11,
        GGML_TYPE_Q4_K = 12,
        GGML_TYPE_Q5_K = 13,
        GGML_TYPE_Q6_K = 14,
        GGML_TYPE_Q8_K = 15,
        GGML_TYPE_IQ2_XXS = 16,
        GGML_TYPE_IQ2_XS  = 17,
        GGML_TYPE_IQ3_XXS = 18,
        GGML_TYPE_I8,
        GGML_TYPE_I16,
        GGML_TYPE_I32,
        GGML_TYPE_COUNT,
    };

    // 精度
    enum ggml_prec {
        GGML_PREC_DEFAULT,
        GGML_PREC_F32,
    };

    // 定义枚举类型 ggml_backend_type，表示后端类型
    enum ggml_backend_type {
        // CPU 后端
        GGML_BACKEND_CPU = 0,
        // GPU 后端
        GGML_BACKEND_GPU = 10,
        // 分布式 GPU 后端
        GGML_BACKEND_GPU_SPLIT = 20,
    };

    // 模型文件类型
    // 定义枚举类型 ggml_ftype，表示不同的数据类型
    enum ggml_ftype {
        // 未知数据类型
        GGML_FTYPE_UNKNOWN     = -1,
        // 全部为 32 位浮点数
        GGML_FTYPE_ALL_F32     = 0,
        // 大部分为 16 位浮点数，除了 1 维张量
        GGML_FTYPE_MOSTLY_F16  = 1,  // except 1d tensors
        // 大部分为 Q4.0 类型，除了 1 维张量
        GGML_FTYPE_MOSTLY_Q4_0 = 2,  // except 1d tensors
        // 大部分为 Q4.1 类型，除了 1 维张量
        GGML_FTYPE_MOSTLY_Q4_1 = 3,  // except 1d tensors
        // 大部分为 Q4.1 类型，部分为 F16 类型，tok_embeddings.weight 和 output.weight 为 F16 类型
        GGML_FTYPE_MOSTLY_Q4_1_SOME_F16 = 4, // tok_embeddings.weight and output.weight are F16
        // 大部分为 Q8.0 类型，除了 1 维张量
        GGML_FTYPE_MOSTLY_Q8_0 = 7,  // except 1d tensors
        // 大部分为 Q5.0 类型，除了 1 维张量
        GGML_FTYPE_MOSTLY_Q5_0 = 8,  // except 1d tensors
        // 大部分为 Q5.1 类型，除了 1 维张量
        GGML_FTYPE_MOSTLY_Q5_1 = 9,  // except 1d tensors
        // 大部分为 Q2.K 类型，除了 1 维张量
        GGML_FTYPE_MOSTLY_Q2_K = 10, // except 1d tensors
        // 大部分为 Q3.K 类型，除了 1 维张量
        GGML_FTYPE_MOSTLY_Q3_K = 11, // except 1d tensors
        // 大部分为 Q4.K 类型，除了 1 维张量
        GGML_FTYPE_MOSTLY_Q4_K = 12, // except 1d tensors
        // 大部分为 Q5.K 类型，除了 1 维张量
        GGML_FTYPE_MOSTLY_Q5_K = 13, // except 1d tensors
        // 大部分为 Q6.K 类型，除了 1 维张量
        GGML_FTYPE_MOSTLY_Q6_K = 14, // except 1d tensors
        // 大部分为 IQ2.XXS 类型，除了 1 维张量
        GGML_FTYPE_MOSTLY_IQ2_XXS = 15, // except 1d tensors
        // 大部分为 IQ2.XS 类型，除了 1 维张量
        GGML_FTYPE_MOSTLY_IQ2_XS  = 16, // except 1d tensors
        // 大部分为 IQ3.XXS 类型，除了 1 维张量
        GGML_FTYPE_MOSTLY_IQ3_XXS = 17, // except 1d tensors
    };

    // 可用的张量操作:
    # 定义枚举类型 ggml_op，包含各种操作的枚举值
    enum ggml_op {
        # 空操作
        GGML_OP_NONE = 0,

        # 复制操作
        GGML_OP_DUP,
        # 加法操作
        GGML_OP_ADD,
        # 加法操作（只有一个操作数）
        GGML_OP_ADD1,
        # 累加操作
        GGML_OP_ACC,
        # 减法操作
        GGML_OP_SUB,
        # 乘法操作
        GGML_OP_MUL,
        # 除法操作
        GGML_OP_DIV,
        # 平方操作
        GGML_OP_SQR,
        # 平方根操作
        GGML_OP_SQRT,
        # 对数操作
        GGML_OP_LOG,
        # 求和操作
        GGML_OP_SUM,
        # 对行求和操作
        GGML_OP_SUM_ROWS,
        # 求均值操作
        GGML_OP_MEAN,
        # 求最大值索引操作
        GGML_OP_ARGMAX,
        # 重复操作
        GGML_OP_REPEAT,
        # 反向重复操作
        GGML_OP_REPEAT_BACK,
        # 连接操作
        GGML_OP_CONCAT,
        # SILU 反向操作
        GGML_OP_SILU_BACK,
        # 归一化操作
        GGML_OP_NORM, // normalize
        # 均方根归一化操作
        GGML_OP_RMS_NORM,
        # 均方根归一化反向操作
        GGML_OP_RMS_NORM_BACK,
        # 分组归一化操作
        GGML_OP_GROUP_NORM,

        # 矩阵乘法操作
        GGML_OP_MUL_MAT,
        # 矩阵乘法操作（带单位矩阵）
        GGML_OP_MUL_MAT_ID,
        # 外积操作
        GGML_OP_OUT_PROD,

        # 缩放操作
        GGML_OP_SCALE,
        # 设置操作
        GGML_OP_SET,
        # 复制操作
        GGML_OP_CPY,
        # 连续操作
        GGML_OP_CONT,
        # 重塑操作
        GGML_OP_RESHAPE,
        # 查看操作
        GGML_OP_VIEW,
        # 排列操作
        GGML_OP_PERMUTE,
        # 转置操作
        GGML_OP_TRANSPOSE,
        # 获取行操作
        GGML_OP_GET_ROWS,
        # 获取行操作（反向）
        GGML_OP_GET_ROWS_BACK,
        # 对角线操作
        GGML_OP_DIAG,
        # 对角线操作（掩码无穷大）
        GGML_OP_DIAG_MASK_INF,
        # 对角线操作（掩码零）
        GGML_OP_DIAG_MASK_ZERO,
        # Softmax 操作
        GGML_OP_SOFT_MAX,
        # Softmax 反向操作
        GGML_OP_SOFT_MAX_BACK,
        # ROPE 操作
        GGML_OP_ROPE,
        # ROPE 反向操作
        GGML_OP_ROPE_BACK,
        # ALIBI 操作
        GGML_OP_ALIBI,
        # 夹紧操作
        GGML_OP_CLAMP,
        # 一维卷积转置操作
        GGML_OP_CONV_TRANSPOSE_1D,
        # 图像转换为列操作
        GGML_OP_IM2COL,
        # 二维卷积转置操作
        GGML_OP_CONV_TRANSPOSE_2D,
        # 一维池化操作
        GGML_OP_POOL_1D,
        # 二维池化操作
        GGML_OP_POOL_2D,
        # 放大操作（最近插值）
        GGML_OP_UPSCALE, // nearest interpolate
        # 填充操作
        GGML_OP_PAD,
        # 参数排序操作
        GGML_OP_ARGSORT,
        # Leaky ReLU 操作
        GGML_OP_LEAKY_RELU,

        # FLASH 注意力操作
        GGML_OP_FLASH_ATTN,
        # FLASH 前馈操作
        GGML_OP_FLASH_FF,
        # FLASH 注意力反向操作
        GGML_OP_FLASH_ATTN_BACK,
        # 窗口部分操作
        GGML_OP_WIN_PART,
        # 窗口取消部分操作
        GGML_OP_WIN_UNPART,
        # 获取相对位置操作
        GGML_OP_GET_REL_POS,
        # 添加相对位置操作
        GGML_OP_ADD_REL_POS,

        # 一元操作
        GGML_OP_UNARY,

        # 映射一元操作
        GGML_OP_MAP_UNARY,
        # 映射二元操作
        GGML_OP_MAP_BINARY,

        # 映射自定义 1（单精度浮点数）操作
        GGML_OP_MAP_CUSTOM1_F32,
        # 映射自定义 2（单精度浮点数）操作
        GGML_OP_MAP_CUSTOM2_F32,
        # 映射自定义 3（单精度浮点数）操作
        GGML_OP_MAP_CUSTOM3_F32,

        # 映射自定义 1 操作
        GGML_OP_MAP_CUSTOM1,
        # 映射自定义 2 操作
        GGML_OP_MAP_CUSTOM2,
        # 映射自定义 3 操作
        GGML_OP_MAP_CUSTOM3,

        # 交叉熵损失操作
        GGML_OP_CROSS_ENTROPY_LOSS,
        # 交叉熵损失反向操作
        GGML_OP_CROSS_ENTROPY_LOSS_BACK,

        # 操作数量
        GGML_OP_COUNT,
    };
    // 定义枚举类型 ggml_unary_op，表示一元操作符
    enum ggml_unary_op {
        GGML_UNARY_OP_ABS,          // 绝对值操作
        GGML_UNARY_OP_SGN,          // 符号操作
        GGML_UNARY_OP_NEG,          // 取反操作
        GGML_UNARY_OP_STEP,         // 阶跃函数操作
        GGML_UNARY_OP_TANH,         // 双曲正切操作
        GGML_UNARY_OP_ELU,          // 指数线性单元操作
        GGML_UNARY_OP_RELU,         // 线性整流操作
        GGML_UNARY_OP_GELU,         // 高斯误差线性单元操作
        GGML_UNARY_OP_GELU_QUICK,   // 快速高斯误差线性单元操作
        GGML_UNARY_OP_SILU,         // 平滑整流操作
        GGML_UNARY_OP_HARDSWISH,    // 硬切线操作
        GGML_UNARY_OP_HARDSIGMOID,  // 硬切线操作

        GGML_UNARY_OP_COUNT,        // 一元操作符数量
    };

    // 定义枚举类型 ggml_object_type，表示对象类型
    enum ggml_object_type {
        GGML_OBJECT_TENSOR,         // 张量对象
        GGML_OBJECT_GRAPH,          // 图对象
        GGML_OBJECT_WORK_BUFFER     // 工作缓冲对象
    };

    // 定义枚举类型 ggml_log_level，表示日志级别
    enum ggml_log_level {
        GGML_LOG_LEVEL_ERROR = 2,   // 错误级别
        GGML_LOG_LEVEL_WARN = 3,    // 警告级别
        GGML_LOG_LEVEL_INFO = 4,    // 信息级别
        GGML_LOG_LEVEL_DEBUG = 5    // 调试级别
    };

    // 定义结构体 ggml_object，表示 ggml 对象
    struct ggml_object {
        size_t offs;                // 偏移量
        size_t size;                // 大小

        struct ggml_object * next;  // 下一个对象指针

        enum ggml_object_type type; // 对象类型

        char padding[4];            // 填充字节
    };

    // 定义常量 GGML_OBJECT_SIZE，表示 ggml_object 结构体的大小
    static const size_t GGML_OBJECT_SIZE = sizeof(struct ggml_object);

    // n-dimensional tensor
    // 定义了一个结构体 ggml_tensor，用于表示张量的相关信息
    struct ggml_tensor {
        // 张量的类型
        enum ggml_type         type;
        // 张量的后端类型
        enum ggml_backend_type backend;

        // 指向后端缓冲区的指针
        struct ggml_backend_buffer * buffer;

        // 张量每个维度的元素数量
        int64_t ne[GGML_MAX_DIMS];
        // 张量每个维度的字节步长
        size_t  nb[GGML_MAX_DIMS]; // stride in bytes:
                                   // nb[0] = ggml_type_size(type)
                                   // nb[1] = nb[0]   * (ne[0] / ggml_blck_size(type)) + padding
                                   // nb[i] = nb[i-1] * ne[i-1]

        // 计算操作
        enum ggml_op op;

        // 操作参数 - 为了对齐而分配为 int32_t 类型
        int32_t op_params[GGML_MAX_OP_PARAMS / sizeof(int32_t)];

        // 是否为参数
        bool is_param;

        // 梯度张量
        struct ggml_tensor * grad;
        // 源张量数组
        struct ggml_tensor * src[GGML_MAX_SRC];

        // 性能
        int     perf_runs;
        int64_t perf_cycles;
        int64_t perf_time_us;

        // 视图源张量
        struct ggml_tensor * view_src;
        // 视图偏移
        size_t               view_offs;

        // 数据指针
        void * data;

        // 名称
        char name[GGML_MAX_NAME];

        // 额外信息指针
        void * extra; // extra things e.g. for ggml-cuda.cu

        // 填充
        char padding[8];
    };

    // 定义了一个常量 GGML_TENSOR_SIZE，表示 ggml_tensor 结构体的大小
    static const size_t GGML_TENSOR_SIZE = sizeof(struct ggml_tensor);

    // 需要为 ggml_graph_compute() 准备的计算计划
    struct ggml_cplan {
        // 工作缓冲区的大小，由 `ggml_graph_plan()` 计算得出
        size_t    work_size;
        // 工作缓冲区的数据指针，需要在调用 `ggml_graph_compute()` 之前由调用者分配
        uint8_t * work_data;

        // 线程数
        int n_threads;

        // 当为 true 时中止 ggml_graph_compute
        bool (*abort_callback)(void * data);
        void * abort_callback_data;
    };

    // 计算图的评估顺序
    enum ggml_cgraph_eval_order {
        GGML_CGRAPH_EVAL_ORDER_LEFT_TO_RIGHT = 0,
        GGML_CGRAPH_EVAL_ORDER_RIGHT_TO_LEFT,
        GGML_CGRAPH_EVAL_ORDER_COUNT
    };

    // 哈希集合结构体
    struct ggml_hash_set {
        // 大小
        size_t size;
        // 键的张量指针数组
        struct ggml_tensor ** keys;
    };

    // 计算图
    // 定义 ggml_cgraph 结构体，用于表示计算图
    struct ggml_cgraph {
        int size; // 计算图的大小
        int n_nodes; // 节点数量
        int n_leafs; // 叶子节点数量
    
        struct ggml_tensor ** nodes; // 节点数组
        struct ggml_tensor ** grads; // 梯度数组
        struct ggml_tensor ** leafs; // 叶子节点数组
    
        struct ggml_hash_set visited_hash_table; // 访问过的节点哈希表
    
        enum ggml_cgraph_eval_order order; // 计算图的评估顺序
    
        // 性能信息
        int     perf_runs; // 性能运行次数
        int64_t perf_cycles; // 性能周期数
        int64_t perf_time_us; // 性能时间（微秒）
    };
    
    // 临时缓冲区
    struct ggml_scratch {
        size_t offs; // 偏移量
        size_t size; // 大小
        void * data; // 数据
    };
    
    // 初始化参数
    struct ggml_init_params {
        // 内存池
        size_t mem_size;   // 内存大小（字节）
        void * mem_buffer; // 如果为 NULL，则内存将在内部分配
        bool   no_alloc;   // 不为张量数据分配内存
    };
    
    // 计算类型
    
    // 注意：除非显式启用，否则不会安排 INIT 或 FINALIZE 传递。
    // 自从 https://github.com/ggerganov/llama.cpp/pull/1995 之后，此行为已更改。
    enum ggml_task_type {
        GGML_TASK_INIT = 0, // 初始化任务
        GGML_TASK_COMPUTE, // 计算任务
        GGML_TASK_FINALIZE, // 结束任务
    };
    
    struct ggml_compute_params {
        enum ggml_task_type type; // 任务类型
    
        // ith = 线程索引，nth = 线程数
        int ith, nth; // 线程索引和线程数
    
        // 所有线程的工作缓冲区
        size_t wsize; // 缓冲区大小
        void * wdata; // 缓冲区数据
    };
    
    // 杂项
    
    GGML_API void    ggml_time_init(void); // 在程序开始时调用一次
    GGML_API int64_t ggml_time_ms(void); // 获取当前时间（毫秒）
    GGML_API int64_t ggml_time_us(void); // 获取当前时间（微秒）
    GGML_API int64_t ggml_cycles(void); // 获取 CPU 周期数
    GGML_API int64_t ggml_cycles_per_ms(void); // 获取每毫秒的 CPU 周期数
    
    GGML_API void    ggml_print_backtrace(void); // 打印回溯信息
    
    GGML_API void    ggml_numa_init(void); // 在 NUMA 系统上调用一次以获得更好的性能
    GGML_API bool    ggml_is_numa(void); // 如果初始化检测到系统有 >1 个 NUMA 节点，则返回 true
    
    GGML_API void    ggml_print_object (const struct ggml_object * obj); // 打印对象信息
    GGML_API void    ggml_print_objects(const struct ggml_context * ctx); // 打印上下文中的所有对象信息
    // 返回张量中元素的数量
    GGML_API GGML_CALL int64_t ggml_nelements   (const struct ggml_tensor * tensor);
    
    // 返回张量中的行数
    GGML_API GGML_CALL int64_t ggml_nrows       (const struct ggml_tensor * tensor);
    
    // 返回张量占用的字节数
    GGML_API GGML_CALL size_t  ggml_nbytes      (const struct ggml_tensor * tensor);
    
    // 返回张量占用的字节数，按照 GGML_MEM_ALIGN 进行填充
    GGML_API           size_t  ggml_nbytes_pad  (const struct ggml_tensor * tensor);
    
    // 返回给定类型的块大小
    GGML_API GGML_CALL int    ggml_blck_size(enum ggml_type type);
    
    // 返回给定类型的每个元素在块中占用的字节数
    GGML_API GGML_CALL size_t ggml_type_size(enum ggml_type type);
    
    // 返回给定类型和行数的每个元素在行中占用的字节数
    GGML_API GGML_CALL size_t ggml_row_size (enum ggml_type type, int64_t ne);
    
    // 返回给定类型的每个元素在块中占用的字节数，以浮点数形式返回
    GGML_DEPRECATED(
    GGML_API double ggml_type_sizef(enum ggml_type type), // ggml_type_size()/ggml_blck_size() as float
    "use ggml_row_size() instead");
    
    // 返回给定类型的名称
    GGML_API GGML_CALL const char * ggml_type_name(enum ggml_type type);
    
    // 返回给定操作的名称
    GGML_API GGML_CALL const char * ggml_op_name  (enum ggml_op   op);
    
    // 返回给定操作的符号
    GGML_API           const char * ggml_op_symbol(enum ggml_op   op);
    
    // 返回给定一元操作的名称
    GGML_API           const char * ggml_unary_op_name(enum ggml_unary_op op);
    
    // 返回给定张量的操作或一元操作的名称
    GGML_API GGML_CALL const char * ggml_op_desc(const struct ggml_tensor * t);
    
    // 返回给定张量中每个元素的大小
    GGML_API GGML_CALL size_t  ggml_element_size(const struct ggml_tensor * tensor);
    
    // 检查给定类型是否是量化的
    GGML_API GGML_CALL bool    ggml_is_quantized(enum ggml_type type);
    
    // 将 ggml_ftype 转换为 ggml_type，用于模型加载
    GGML_API enum ggml_type ggml_ftype_to_ggml_type(enum ggml_ftype ftype);
    
    // 检查张量是否被转置
    GGML_API GGML_CALL bool ggml_is_transposed(const struct ggml_tensor * tensor);
    
    // 检查张量是否是连续的
    GGML_API GGML_CALL bool ggml_is_contiguous(const struct ggml_tensor * tensor);
    
    // 检查张量是否被置换
    GGML_API GGML_CALL bool ggml_is_permuted  (const struct ggml_tensor * tensor);
    
    // 检查张量是否是标量
    GGML_API           bool ggml_is_scalar    (const struct ggml_tensor * tensor);
    
    // 检查张量是否是向量
    GGML_API           bool ggml_is_vector    (const struct ggml_tensor * tensor);
    // 检查给定的张量是否是矩阵
    GGML_API bool ggml_is_matrix(const struct ggml_tensor * tensor);
    
    // 检查给定的张量是否是三维的
    GGML_API bool ggml_is_3d(const struct ggml_tensor * tensor);
    
    // 返回给定张量的维度数量，对于标量返回1
    GGML_API int ggml_n_dims(const struct ggml_tensor * tensor);
    
    // 检查两个张量是否具有相同的形状
    GGML_API bool ggml_are_same_shape(const struct ggml_tensor * t0, const struct ggml_tensor * t1);
    
    // 用于计算张量的内存开销
    GGML_API size_t ggml_tensor_overhead(void);
    
    // 初始化 GGML 上下文
    GGML_API struct ggml_context * ggml_init(struct ggml_init_params params);
    
    // 释放 GGML 上下文
    GGML_API void ggml_free(struct ggml_context * ctx);
    
    // 返回 GGML 上下文使用的内存大小
    GGML_API size_t ggml_used_mem(const struct ggml_context * ctx);
    
    // 设置 GGML 上下文的临时内存
    GGML_API size_t ggml_set_scratch(struct ggml_context * ctx, struct ggml_scratch scratch);
    
    // 获取 GGML 上下文是否允许分配内存
    GGML_API bool ggml_get_no_alloc(struct ggml_context * ctx);
    
    // 设置 GGML 上下文是否允许分配内存
    GGML_API void ggml_set_no_alloc(struct ggml_context * ctx, bool no_alloc);
    
    // 获取 GGML 上下文的内存缓冲区
    GGML_API void * ggml_get_mem_buffer(const struct ggml_context * ctx);
    
    // 获取 GGML 上下文的内存大小
    GGML_API size_t ggml_get_mem_size(const struct ggml_context * ctx);
    
    // 获取 GGML 上下文的最大张量大小
    GGML_API size_t ggml_get_max_tensor_size(const struct ggml_context * ctx);
    
    // 创建一个新的张量
    GGML_API struct ggml_tensor * ggml_new_tensor(struct ggml_context * ctx, enum ggml_type type, int n_dims, const int64_t *ne);
    
    // 创建一个新的一维张量
    GGML_API struct ggml_tensor * ggml_new_tensor_1d(struct ggml_context * ctx, enum ggml_type type, int64_t ne0);
    
    // 创建一个新的二维张量
    GGML_API struct ggml_tensor * ggml_new_tensor_2d(struct ggml_context * ctx, enum ggml_type type, int64_t ne0, int64_t ne1);
    
    // 创建一个新的三维张量
    GGML_API struct ggml_tensor * ggml_new_tensor_3d(struct ggml_context * ctx, enum ggml_type type, int64_t ne0, int64_t ne1, int64_t ne2);
    // 创建一个四维张量，包含指定类型和维度大小
    GGML_API struct ggml_tensor * ggml_new_tensor_4d(
            struct ggml_context * ctx,
            enum   ggml_type type,
            int64_t ne0,
            int64_t ne1,
            int64_t ne2,
            int64_t ne3);
    
    // 创建一个包含单个 int32_t 值的张量
    GGML_API struct ggml_tensor * ggml_new_i32(struct ggml_context * ctx, int32_t value);
    
    // 创建一个包含单个 float 值的张量
    GGML_API struct ggml_tensor * ggml_new_f32(struct ggml_context * ctx, float value);
    
    // 复制一个张量
    GGML_API struct ggml_tensor * ggml_dup_tensor (struct ggml_context * ctx, const struct ggml_tensor * src);
    
    // 创建一个张量的视图
    GGML_API struct ggml_tensor * ggml_view_tensor(struct ggml_context * ctx, struct ggml_tensor * src);
    
    // 获取上下文中的第一个张量
    GGML_API struct ggml_tensor * ggml_get_first_tensor(const struct ggml_context * ctx);
    
    // 获取上下文中下一个张量
    GGML_API struct ggml_tensor * ggml_get_next_tensor (const struct ggml_context * ctx, struct ggml_tensor * tensor);
    
    // 根据名称获取张量
    GGML_API struct ggml_tensor * ggml_get_tensor(struct ggml_context * ctx, const char * name);
    
    // 将张量的所有元素设置为零
    GGML_API struct ggml_tensor * ggml_set_zero(struct ggml_tensor * tensor);
    
    // 将张量的所有元素设置为指定的 int32_t 值
    GGML_API struct ggml_tensor * ggml_set_i32 (struct ggml_tensor * tensor, int32_t value);
    
    // 将张量的所有元素设置为指定的 float 值
    GGML_API struct ggml_tensor * ggml_set_f32 (struct ggml_tensor * tensor, float value);
    
    // 将一维索引转换为四维张量的坐标
    GGML_API void    ggml_unravel_index(const struct ggml_tensor * tensor, int64_t i, int64_t * i0, int64_t * i1, int64_t * i2, int64_t * i3);
    
    // 获取一维张量中指定索引处的 int32_t 值
    GGML_API int32_t ggml_get_i32_1d(const struct ggml_tensor * tensor, int i);
    
    // 设置一维张量中指定索引处的 int32_t 值
    GGML_API void    ggml_set_i32_1d(const struct ggml_tensor * tensor, int i, int32_t value);
    
    // 获取多维张量中指定索引处的 int32_t 值
    GGML_API int32_t ggml_get_i32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3);
    
    // 设置多维张量中指定索引处的 int32_t 值
    GGML_API void    ggml_set_i32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3, int32_t value);
    
    // 获取一维张量中指定索引处的 float 值
    GGML_API float   ggml_get_f32_1d(const struct ggml_tensor * tensor, int i);
    // 设置一维浮点数张量的值
    GGML_API void ggml_set_f32_1d(const struct ggml_tensor * tensor, int i, float value);
    
    // 获取多维浮点数张量的值
    GGML_API float ggml_get_f32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3);
    // 设置多维浮点数张量的值
    GGML_API void ggml_set_f32_nd(const struct ggml_tensor * tensor, int i0, int i1, int i2, int i3, float value);
    
    // 获取张量的数据指针
    GGML_API void * ggml_get_data(const struct ggml_tensor * tensor);
    // 获取浮点数张量的数据指针
    GGML_API float * ggml_get_data_f32(const struct ggml_tensor * tensor);
    
    // 获取张量的一元操作
    GGML_API GGML_CALL enum ggml_unary_op ggml_get_unary_op(const struct ggml_tensor * tensor);
    
    // 获取张量的名称
    GGML_API const char * ggml_get_name(const struct ggml_tensor * tensor);
    // 设置张量的名称
    GGML_API struct ggml_tensor * ggml_set_name(struct ggml_tensor * tensor, const char * name);
    // 格式化张量的名称
    GGML_ATTRIBUTE_FORMAT(2, 3)
    GGML_API struct ggml_tensor * ggml_format_name(struct ggml_tensor * tensor, const char * fmt, ...);
    
    // 在张量上进行反向传播的操作
    
    // 复制张量
    GGML_API struct ggml_tensor * ggml_dup(struct ggml_context * ctx, struct ggml_tensor * a);
    // 原地复制张量，返回视图(a)
    GGML_API struct ggml_tensor * ggml_dup_inplace(struct ggml_context * ctx, struct ggml_tensor * a);
    
    // 张量相加
    GGML_API struct ggml_tensor * ggml_add(struct ggml_context * ctx, struct ggml_tensor * a, struct ggml_tensor * b);
    // 原地张量相加
    GGML_API struct ggml_tensor * ggml_add_inplace(struct ggml_context * ctx, struct ggml_tensor * a, struct ggml_tensor * b);
    // 张量相加并转换类型
    GGML_API struct ggml_tensor * ggml_add_cast(struct ggml_context * ctx, struct ggml_tensor * a, struct ggml_tensor * b, enum ggml_type type);
    // 张量相加
    GGML_API struct ggml_tensor * ggml_add1(struct ggml_context * ctx, struct ggml_tensor * a, struct ggml_tensor * b);
    // 在张量 a 上执行原地加法操作，将结果存储在 a 中，并返回 a
    GGML_API struct ggml_tensor * ggml_add1_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    // 将张量 b 加到张量 a 上的指定视图中，视图由 nb1、nb2、nb3 和 offset 定义，返回结果张量
    GGML_API struct ggml_tensor * ggml_acc(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                nb1,
            size_t                nb2,
            size_t                nb3,
            size_t                offset);
    
    // 在张量 a 上执行原地累加操作，将张量 b 加到指定视图中，视图由 nb1、nb2、nb3 和 offset 定义，返回结果张量
    GGML_API struct ggml_tensor * ggml_acc_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                nb1,
            size_t                nb2,
            size_t                nb3,
            size_t                offset);
    
    // 计算张量 a 与张量 b 的差，返回结果张量
    GGML_API struct ggml_tensor * ggml_sub(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    // 在张量 a 上执行原地减法操作，将结果存储在 a 中，并返回 a
    GGML_API struct ggml_tensor * ggml_sub_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    // 计算张量 a 与张量 b 的乘积，返回结果张量
    GGML_API struct ggml_tensor * ggml_mul(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    // 在张量 a 上执行原地乘法操作，将结果存储在 a 中，并返回 a
    GGML_API struct ggml_tensor * ggml_mul_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    // 计算张量 a 与张量 b 的除法，返回结果张量
    GGML_API struct ggml_tensor * ggml_div(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    // 在张量 a 上执行原地除法操作，将结果存储在 a 中，并返回 a
    GGML_API struct ggml_tensor * ggml_div_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    // 计算张量 a 的平方，返回结果张量
    GGML_API struct ggml_tensor * ggml_sqr(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    // 对输入的张量进行平方操作，结果保存在原张量中
    GGML_API struct ggml_tensor * ggml_sqr_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 对输入的张量进行平方根操作
    GGML_API struct ggml_tensor * ggml_sqrt(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 对输入的张量进行平方根操作，结果保存在原张量中
    GGML_API struct ggml_tensor * ggml_sqrt_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 对输入的张量进行自然对数操作
    GGML_API struct ggml_tensor * ggml_log(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 对输入的张量进行自然对数操作，结果保存在原张量中
    GGML_API struct ggml_tensor * ggml_log_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 对输入的张量进行求和操作，返回标量
    GGML_API struct ggml_tensor * ggml_sum(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 沿着行方向对输入的张量进行求和，返回形状为[1,b,c,d]的张量
    GGML_API struct ggml_tensor * ggml_sum_rows(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 沿着行方向对输入的张量进行求均值操作
    GGML_API struct ggml_tensor * ggml_mean(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 沿着行方向对输入的张量进行求最大值索引操作
    GGML_API struct ggml_tensor * ggml_argmax(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 如果a与b的形状相同，并且a不是参数，则返回a；否则，返回一个新的张量：将a重复以适应b的形状
    GGML_API struct ggml_tensor * ggml_repeat(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    // 将a中的重复项求和，使其形状与b相同
    GGML_API struct ggml_tensor * ggml_repeat_back(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    // 在维度2上连接a和b，用于稳定扩散
    # 定义一个函数 ggml_concat，用于将两个张量连接起来
    GGML_API struct ggml_tensor * ggml_concat(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    # 定义一个函数 ggml_abs，用于计算张量的绝对值
    GGML_API struct ggml_tensor * ggml_abs(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    # 定义一个函数 ggml_abs_inplace，用于计算张量的绝对值并替换原始张量
    GGML_API struct ggml_tensor * ggml_abs_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    # 定义一个函数 ggml_sgn，用于计算张量的符号
    GGML_API struct ggml_tensor * ggml_sgn(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    # 定义一个函数 ggml_sgn_inplace，用于计算张量的符号并替换原始张量
    GGML_API struct ggml_tensor * ggml_sgn_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    # 定义一个函数 ggml_neg，用于计算张量的负值
    GGML_API struct ggml_tensor * ggml_neg(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    # 定义一个函数 ggml_neg_inplace，用于计算张量的负值并替换原始张量
    GGML_API struct ggml_tensor * ggml_neg_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    # 定义一个函数 ggml_step，用于计算张量的阶跃函数
    GGML_API struct ggml_tensor * ggml_step(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    # 定义一个函数 ggml_step_inplace，用于计算张量的阶跃函数并替换原始张量
    GGML_API struct ggml_tensor * ggml_step_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    # 定义一个函数 ggml_tanh，用于计算张量的双曲正切函数
    GGML_API struct ggml_tensor * ggml_tanh(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    # 定义一个函数 ggml_tanh_inplace，用于计算张量的双曲正切函数并替换原始张量
    GGML_API struct ggml_tensor * ggml_tanh_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    # 定义一个函数 ggml_elu，用于计算张量的ELU激活函数
    GGML_API struct ggml_tensor * ggml_elu(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    # 定义一个函数 ggml_elu_inplace，用于计算张量的ELU激活函数并替换原始张量
    GGML_API struct ggml_tensor * ggml_elu_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    # 定义一个函数 ggml_relu，用于计算张量的ReLU激活函数
    GGML_API struct ggml_tensor * ggml_relu(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    # 定义一个函数 ggml_leaky_relu，用于计算张量的Leaky ReLU激活函数，可以指定负斜率和是否替换原始张量
    GGML_API struct ggml_tensor * ggml_leaky_relu(
            struct ggml_context * ctx,
            struct ggml_tensor  * a, float negative_slope, bool inplace);
    // 定义一个函数，对输入的张量进行原地 ReLU 操作
    GGML_API struct ggml_tensor * ggml_relu_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 定义一个函数，对输入的张量进行 GELU 操作
    GGML_API struct ggml_tensor * ggml_gelu(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 定义一个函数，对输入的张量进行原地 GELU 操作
    GGML_API struct ggml_tensor * ggml_gelu_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 定义一个函数，对输入的张量进行快速 GELU 操作
    GGML_API struct ggml_tensor * ggml_gelu_quick(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 定义一个函数，对输入的张量进行原地快速 GELU 操作
    GGML_API struct ggml_tensor * ggml_gelu_quick_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 定义一个函数，对输入的张量进行 SiLU 操作
    GGML_API struct ggml_tensor * ggml_silu(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 定义一个函数，对输入的张量进行原地 SiLU 操作
    GGML_API struct ggml_tensor * ggml_silu_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 定义一个函数，对输入的张量进行 SiLU 反向传播操作
    // 参数 a 为 x，参数 b 为 dy
    GGML_API struct ggml_tensor * ggml_silu_back(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    // 定义一个函数，对输入的张量进行 HardSwish 操作
    GGML_API struct ggml_tensor * ggml_hardswish(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 定义一个函数，对输入的张量进行 HardSigmoid 操作
    GGML_API struct ggml_tensor * ggml_hardsigmoid(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 定义一个函数，对输入的张量进行沿着行方向的归一化操作
    GGML_API struct ggml_tensor * ggml_norm(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            float                 eps);
    
    // 定义一个函数，对输入的张量进行原地归一化操作
    GGML_API struct ggml_tensor * ggml_norm_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            float                 eps);
    
    // 定义一个函数，对输入的张量进行 RMS 归一化操作
    GGML_API struct ggml_tensor * ggml_rms_norm(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            float                 eps);
    // 在原地计算 RMS 归一化，返回归一化后的张量
    GGML_API struct ggml_tensor * ggml_rms_norm_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            float                 eps);

    // 沿 ne0*ne1*n_groups 维度进行分组归一化
    // 在 stable-diffusion 中使用
    // TODO: 目前 eps 被硬编码为 1e-6
    GGML_API struct ggml_tensor * ggml_group_norm(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_groups);

    // 在原地计算分组归一化，返回归一化后的张量
    GGML_API struct ggml_tensor * ggml_group_norm_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_groups);

    // 计算 a - x
    // b - dy
    GGML_API struct ggml_tensor * ggml_rms_norm_back(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            float                 eps);

    // A: k 列, n 行 => [ne03, ne02, n, k]
    // B: k 列, m 行  (即我们在内部对其进行转置) => [ne03 * x, ne02 * y, m, k]
    // 结果是 n 列, m 行 => [ne03 * x, ne02 * y, m, n]
    GGML_API struct ggml_tensor * ggml_mul_mat(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    // 更改矩阵乘法的精度
    // 设置为 GGML_PREC_F32 以获得更高的精度（对 phi-2 很有用）
    GGML_API void ggml_mul_mat_set_prec(
            struct ggml_tensor * a,
            enum ggml_prec       prec);

    // 间接矩阵乘法
    // ggml_mul_mat_id(ctx, as, ids, id, b) ~= ggml_mul_mat(as[ids[id]], b)
    GGML_API struct ggml_tensor * ggml_mul_mat_id(
            struct ggml_context * ctx,
            struct ggml_tensor  * const as[],
            int                   n_as,
            struct ggml_tensor  * ids,
            int                   id,
            struct ggml_tensor  * b);

    // A: m 列, n 行,
    // B: p 列, n 行,
    // 结果是 m 列, p 行
    // 定义一个函数，计算两个张量的外积并返回结果张量
    GGML_API struct ggml_tensor * ggml_out_prod(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    //
    // 在没有反向传播的张量上进行操作
    //
    
    // 定义一个函数，对张量进行缩放操作并返回结果张量
    GGML_API struct ggml_tensor * ggml_scale(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            float                 s);
    
    // 在原地进行缩放操作，返回视图(a)
    GGML_API struct ggml_tensor * ggml_scale_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            float                 s);
    
    // 将张量b设置为视图(a,offset,nb1,nb2,3)，返回修改后的a
    GGML_API struct ggml_tensor * ggml_set(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                nb1,
            size_t                nb2,
            size_t                nb3,
            size_t                offset);
    
    // 将张量b设置为视图(a,offset,nb1,nb2,3)，返回视图(a)
    GGML_API struct ggml_tensor * ggml_set_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                nb1,
            size_t                nb2,
            size_t                nb3,
            size_t                offset);
    
    // 将张量b设置为视图(a,offset,nb1,nb2,3)，返回修改后的a
    GGML_API struct ggml_tensor * ggml_set_1d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                offset);
    
    // 在原地将张量b设置为视图(a,offset,nb1,nb2,3)，返回视图(a)
    GGML_API struct ggml_tensor * ggml_set_1d_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                offset);
    
    // 将张量b设置为视图(a,offset,nb1,nb2,3)，返回修改后的a
    // 定义一个函数，将 b 视图设置为 a 的偏移量为 offset，长度为 nb1 的视图，返回 a 的视图
    GGML_API struct ggml_tensor * ggml_set_2d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                nb1,
            size_t                offset);

    // 将 b 视图设置为 a，返回 b 的视图
    GGML_API struct ggml_tensor * ggml_set_2d_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            size_t                nb1,
            size_t                offset);

    // 将 a 复制到 b，返回 b 的视图
    GGML_API struct ggml_tensor * ggml_cpy(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);

    // 将 a 转换为指定类型，返回转换后的视图
    GGML_API struct ggml_tensor * ggml_cast(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            enum   ggml_type      type);

    // 将 a 转换为连续存储方式，返回转换后的视图
    GGML_API struct ggml_tensor * ggml_cont(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);

    // 将 a 转换为连续存储方式，并指定新的形状，返回转换后的视图
    GGML_API struct ggml_tensor * ggml_cont_1d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0);

    // 将 a 转换为连续存储方式，并指定新的形状，返回转换后的视图
    GGML_API struct ggml_tensor * ggml_cont_2d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1);

    // 将 a 转换为连续存储方式，并指定新的形状，返回转换后的视图
    GGML_API struct ggml_tensor * ggml_cont_3d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1,
            int64_t               ne2);

    // 将 a 转换为连续存储方式，并指定新的形状，返回转换后的视图
    GGML_API struct ggml_tensor * ggml_cont_4d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1,
            int64_t               ne2,
            int64_t               ne3);
    // 返回一个视图(a)，b指定新的形状
    // TODO: 当我们开始计算梯度时，应该复制而不是视图
    GGML_API struct ggml_tensor * ggml_reshape(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    // 返回一个视图(a)
    // TODO: 当我们开始计算梯度时，应该复制而不是视图
    GGML_API struct ggml_tensor * ggml_reshape_1d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0);
    
    GGML_API struct ggml_tensor * ggml_reshape_2d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1);
    
    // 返回一个视图(a)
    // TODO: 当我们开始计算梯度时，应该复制而不是视图
    GGML_API struct ggml_tensor * ggml_reshape_3d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1,
            int64_t               ne2);
    
    GGML_API struct ggml_tensor * ggml_reshape_4d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1,
            int64_t               ne2,
            int64_t               ne3);
    
    // 偏移量（字节）
    GGML_API struct ggml_tensor * ggml_view_1d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            size_t                offset);
    
    GGML_API struct ggml_tensor * ggml_view_2d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1,
            size_t                nb1, // 行跨度（字节）
            size_t                offset);
    // 定义一个函数，用于创建一个三维视图的张量
    GGML_API struct ggml_tensor * ggml_view_3d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1,
            int64_t               ne2,
            size_t                nb1, // 行的步长（字节）
            size_t                nb2, // 切片的步长（字节）
            size_t                offset);
    
    // 定义一个函数，用于创建一个四维视图的张量
    GGML_API struct ggml_tensor * ggml_view_4d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int64_t               ne0,
            int64_t               ne1,
            int64_t               ne2,
            int64_t               ne3,
            size_t                nb1, // 行的步长（字节）
            size_t                nb2, // 切片的步长（字节）
            size_t                nb3,
            size_t                offset);
    
    // 定义一个函数，用于对张量进行轴置换
    GGML_API struct ggml_tensor * ggml_permute(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   axis0,
            int                   axis1,
            int                   axis2,
            int                   axis3);
    
    // 定义一个函数，用于对张量进行转置操作，相当于 ggml_permute(ctx, a, 1, 0, 2, 3)
    GGML_API struct ggml_tensor * ggml_transpose(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 定义一个函数，用于获取张量中的行数据，支持三维情况：a->ne[2] == b->ne[1]
    GGML_API struct ggml_tensor * ggml_get_rows(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    // 定义一个函数，用于将张量中的行数据还原回原始张量
    GGML_API struct ggml_tensor * ggml_get_rows_back(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            struct ggml_tensor  * c);
    
    // 定义一个函数，用于获取张量的对角线元素
    GGML_API struct ggml_tensor * ggml_diag(
        struct ggml_context     * ctx,
        struct ggml_tensor      * a);
    
    // 将对角线以上的元素设置为 -INF
    // 定义一个函数，用于生成对角线上方的掩码，返回一个新的张量
    GGML_API struct ggml_tensor * ggml_diag_mask_inf(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_past);
    
    // 原地操作，返回视图(a)
    GGML_API struct ggml_tensor * ggml_diag_mask_inf_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_past);
    
    // 将对角线上方的元素设置为0
    GGML_API struct ggml_tensor * ggml_diag_mask_zero(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_past);
    
    // 原地操作，返回视图(a)
    GGML_API struct ggml_tensor * ggml_diag_mask_zero_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_past);
    
    // 对张量进行 soft_max 操作
    GGML_API struct ggml_tensor * ggml_soft_max(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 原地操作，返回视图(a)
    GGML_API struct ggml_tensor * ggml_soft_max_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a);
    
    // 融合 soft_max(a*scale + mask)
    // mask 是可选的
    GGML_API struct ggml_tensor * ggml_soft_max_ext(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * mask,
            float                 scale);
    
    // soft_max 反向传播
    GGML_API struct ggml_tensor * ggml_soft_max_back(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    // 原地操作，返回视图(a)
    GGML_API struct ggml_tensor * ggml_soft_max_back_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    // 旋转位置嵌入
    // 如果 mode & 1 == 1，跳过 n_past 个元素（已弃用）
    // 如果 mode & 2 == 1，GPT-NeoX 风格
    // 如果 mode & 4 == 1，ChatGLM 风格
    //
    // 定义一个函数ggml_rope，接受上下文ctx，两个张量a和b，以及一些参数，返回一个新的张量
    GGML_API struct ggml_tensor * ggml_rope(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   n_dims,
            int                   mode,
            int                   n_ctx);

    // 在原地操作，返回a的视图
    GGML_API struct ggml_tensor * ggml_rope_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   n_dims,
            int                   mode,
            int                   n_ctx);

    // 自定义RoPE
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

    // 在原地操作，返回a的视图
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
    // 计算 YaRN RoPE 缩放的修正维度
    GGML_CALL void ggml_rope_yarn_corr_dims(
        int n_dims, int n_orig_ctx, float freq_base, float beta_fast, float beta_slow, float dims[2]);
    
    // xPos RoPE，原地操作，返回视图(a)
    GGML_API struct ggml_tensor * ggml_rope_xpos_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   n_dims,
            float                 base,
            bool                  down);
    
    // 旋转位置嵌入的反向传播，即从 dy 计算 dx
    // a - dy
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
    
    // alibi 位置嵌入
    // 原地操作，返回视图(a)
    GGML_API struct ggml_tensor * ggml_alibi(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   n_past,
            int                   n_head,
            float                 bias_max);
    
    // 夹紧
    // 原地操作，返回视图(a)
    GGML_API struct ggml_tensor * ggml_clamp(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            float                 min,
            float                 max);
    // 定义一个函数 ggml_im2col，用于将输入张量转换为矩阵形式，用于卷积操作
    GGML_API struct ggml_tensor * ggml_im2col(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                  s0,  // 水平方向步长
            int                  s1,  // 垂直方向步长
            int                  p0,  // 水平方向填充
            int                  p1,  // 垂直方向填充
            int                  d0,  // 水平方向膨胀
            int                  d1,  // 垂直方向膨胀
            bool                 is_2D);  // 是否为二维卷积
    
    // 定义一个函数 ggml_conv_depthwise_2d，用于深度可分离卷积操作
    GGML_API struct ggml_tensor * ggml_conv_depthwise_2d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                  s0,  // 水平方向步长
            int                  s1,  // 垂直方向步长
            int                  p0,  // 水平方向填充
            int                  p1,  // 垂直方向填充
            int                  d0,  // 水平方向膨胀
            int                  d1);  // 垂直方向膨胀
    
    // 定义一个函数 ggml_conv_1d，用于一维卷积操作
    GGML_API struct ggml_tensor * ggml_conv_1d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   s0,  // 步长
            int                   p0,  // 填充
            int                   d0); // 膨胀
    
    // 定义一个函数 ggml_conv_1d_ph，用于带有 half 填充的一维卷积操作
    // 是 ggml_conv_1d 的别名，参数为 a, b, s, a->ne[0]/2, d
    GGML_API struct ggml_tensor* ggml_conv_1d_ph(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   s,  // 步长
            int                   d); // 膨胀
    
    // 定义一个函数 ggml_conv_transpose_1d，用于一维转置卷积操作
    GGML_API struct ggml_tensor * ggml_conv_transpose_1d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   s0,  // 步长
            int                   p0,  // 填充
            int                   d0); // 膨胀
    // 定义一个函数 ggml_conv_2d，接受上下文 ctx、两个张量 a 和 b，以及 s0、s1、p0、p1、d0、d1 作为参数
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
    
    
    // kernel 大小为 a->ne[0] x a->ne[1]
    // 步长等于 kernel 大小
    // 填充为零
    // 示例:
    // a:     16   16    3  768
    // b:   1024 1024    3    1
    // res:   64   64  768    1
    // 在 sam 中使用
    GGML_API struct ggml_tensor * ggml_conv_2d_sk_p0(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    // kernel 大小为 a->ne[0] x a->ne[1]
    // 步长为 1
    // 填充为一半
    // 示例:
    // a:      3    3    256  256
    // b:     64   64    256    1
    // res:   64   64    256    1
    // 在 sam 中使用
    GGML_API struct ggml_tensor * ggml_conv_2d_s1_ph(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b);
    
    // 定义一个函数 ggml_conv_transpose_2d_p0，接受上下文 ctx、两个张量 a 和 b，以及 stride 作为参数
    GGML_API struct ggml_tensor * ggml_conv_transpose_2d_p0(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b,
            int                   stride);
    
    // 定义一个枚举 ggml_op_pool，包含 GGML_OP_POOL_MAX、GGML_OP_POOL_AVG、GGML_OP_POOL_COUNT 三个值
    enum ggml_op_pool {
        GGML_OP_POOL_MAX,
        GGML_OP_POOL_AVG,
        GGML_OP_POOL_COUNT,
    };
    
    // 定义一个函数 ggml_pool_1d，接受上下文 ctx、一个张量 a、操作 op、kernel 大小 k0、步长 s0、填充 p0 作为参数
    GGML_API struct ggml_tensor * ggml_pool_1d(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            enum ggml_op_pool     op,
            int                   k0, // kernel size
            int                   s0, // stride
            int                   p0); // padding
    
    // 结果的第一维将有 2*p0 的填充
    // 第二维将有 2*p1 的填充
    // 定义一个函数，用于对二维张量进行池化操作
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
    
    // 最近邻插值，用于稳定扩散
    GGML_API struct ggml_tensor * ggml_upscale(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   scale_factor);
    
    // 在每个维度上用零填充：[x, ..., x] -> [x, ..., x, 0, ..., 0]
    GGML_API struct ggml_tensor * ggml_pad(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                  p0,
            int                  p1,
            int                  p2,
            int                  p3);
    
    // 对行进行排序
    enum ggml_sort_order {
        GGML_SORT_ASC,
        GGML_SORT_DESC,
    };
    
    // 对张量进行排序操作
    GGML_API struct ggml_tensor * ggml_argsort(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            enum ggml_sort_order  order);
    
    // 每行取前 k 个元素
    GGML_API struct ggml_tensor * ggml_top_k(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   k);
    
    // Flash 注意力机制
    GGML_API struct ggml_tensor * ggml_flash_attn(
            struct ggml_context * ctx,
            struct ggml_tensor  * q,
            struct ggml_tensor  * k,
            struct ggml_tensor  * v,
            bool                  masked);
    
    // Flash 注意力机制的反向传播
    GGML_API struct ggml_tensor * ggml_flash_attn_back(
           struct ggml_context * ctx,
           struct ggml_tensor  * q,
           struct ggml_tensor  * k,
           struct ggml_tensor  * v,
           struct ggml_tensor  * d,
           bool                  masked);
    // 定义一个函数，用于执行神经网络中的前向传播操作
    GGML_API struct ggml_tensor * ggml_flash_ff(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * b0,
            struct ggml_tensor  * b1,
            struct ggml_tensor  * c0,
            struct ggml_tensor  * c1);

    // 将输入张量分割成不重叠的窗口，并在需要时进行填充
    // 例如:
    // a:   768   64   64    1
    // w:    14
    // res: 768   14   14    25
    // 在 sam 中使用
    GGML_API struct ggml_tensor * ggml_win_part(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   w);

    // ggml_win_part 的逆操作
    // 在 sam 中使用
    GGML_API struct ggml_tensor * ggml_win_unpart(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   w0,
            int                   h0,
            int                   w);

    // 对输入张量执行一元操作
    GGML_API struct ggml_tensor * ggml_unary(
            struct ggml_context * ctx,
             struct ggml_tensor * a,
             enum ggml_unary_op op);

    // 在原地对输入张量执行一元操作
    GGML_API struct ggml_tensor * ggml_unary_inplace(
        struct ggml_context * ctx,
        struct ggml_tensor  * a,
        enum ggml_unary_op op);

    // 在 sam 中使用，获取相对位置信息
    GGML_API struct ggml_tensor * ggml_get_rel_pos(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            int                   qh,
            int                   kh);

    // 在 sam 中使用，添加相对位置信息
    GGML_API struct ggml_tensor * ggml_add_rel_pos(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * pw,
            struct ggml_tensor  * ph);

    // 在原地添加相对位置信息
    GGML_API struct ggml_tensor * ggml_add_rel_pos_inplace(
            struct ggml_context * ctx,
            struct ggml_tensor  * a,
            struct ggml_tensor  * pw,
            struct ggml_tensor  * ph);

    // 自定义操作符
    typedef void (*ggml_unary_op_f32_t) (const int, float *, const float *);
    // 定义一个函数指针类型，用于表示接受两个 float 数组参数的二元操作
    typedef void (*ggml_binary_op_f32_t)(const int, float *, const float *, const float *);

    // 定义三个函数指针类型，分别表示接受一个输入张量参数的自定义操作
    typedef void (*ggml_custom1_op_f32_t)(struct ggml_tensor *, const struct ggml_tensor *);
    typedef void (*ggml_custom2_op_f32_t)(struct ggml_tensor *, const struct ggml_tensor *, const struct ggml_tensor *);
    typedef void (*ggml_custom3_op_f32_t)(struct ggml_tensor *, const struct ggml_tensor *, const struct ggml_tensor *);

    // 定义一个已弃用的函数，用于对输入张量进行一元操作，建议使用 ggml_map_custom1 替代
    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_unary_f32(
            struct ggml_context        * ctx,
            struct ggml_tensor         * a,
                   ggml_unary_op_f32_t   fun),
        "use ggml_map_custom1 instead");

    // 定义一个已弃用的函数，用于对输入张量进行原地一元操作，建议使用 ggml_map_custom1_inplace 替代
    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_unary_inplace_f32(
            struct ggml_context        * ctx,
            struct ggml_tensor         * a,
                   ggml_unary_op_f32_t   fun),
        "use ggml_map_custom1_inplace instead");

    // 定义一个已弃用的函数，用于对两个输入张量进行二元操作，建议使用 ggml_map_custom2 替代
    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_binary_f32(
            struct ggml_context         * ctx,
            struct ggml_tensor          * a,
            struct ggml_tensor          * b,
                   ggml_binary_op_f32_t   fun),
        "use ggml_map_custom2 instead");

    // 定义一个已弃用的函数，用于对两个输入张量进行原地二元操作，建议使用 ggml_map_custom2_inplace 替代
    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_binary_inplace_f32(
            struct ggml_context         * ctx,
            struct ggml_tensor          * a,
            struct ggml_tensor          * b,
                   ggml_binary_op_f32_t   fun),
        "use ggml_map_custom2_inplace instead");

    // 定义一个已弃用的函数，用于对一个输入张量进行自定义操作，建议使用 ggml_map_custom1 替代
    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom1_f32(
            struct ggml_context          * ctx,
            struct ggml_tensor           * a,
                   ggml_custom1_op_f32_t   fun),
        "use ggml_map_custom1 instead");
    // 使用 GGML_DEPRECATED 宏标记 ggml_map_custom1_inplace_f32 函数已废弃，建议使用 ggml_map_custom1_inplace 替代
    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom1_inplace_f32(
            struct ggml_context          * ctx,
            struct ggml_tensor           * a,
                   ggml_custom1_op_f32_t   fun),
        "use ggml_map_custom1_inplace instead");

    // 使用 GGML_DEPRECATED 宏标记 ggml_map_custom2_f32 函数已废弃，建议使用 ggml_map_custom2 替代
    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom2_f32(
            struct ggml_context          * ctx,
            struct ggml_tensor           * a,
            struct ggml_tensor           * b,
                   ggml_custom2_op_f32_t   fun),
        "use ggml_map_custom2 instead");

    // 使用 GGML_DEPRECATED 宏标记 ggml_map_custom2_inplace_f32 函数已废弃，建议使用 ggml_map_custom2_inplace 替代
    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom2_inplace_f32(
            struct ggml_context          * ctx,
            struct ggml_tensor           * a,
            struct ggml_tensor           * b,
                   ggml_custom2_op_f32_t   fun),
        "use ggml_map_custom2_inplace instead");

    // 使用 GGML_DEPRECATED 宏标记 ggml_map_custom3_f32 函数已废弃，建议使用 ggml_map_custom3 替代
    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom3_f32(
            struct ggml_context          * ctx,
            struct ggml_tensor           * a,
            struct ggml_tensor           * b,
            struct ggml_tensor           * c,
                   ggml_custom3_op_f32_t   fun),
        "use ggml_map_custom3 instead");

    // 使用 GGML_DEPRECATED 宏标记 ggml_map_custom3_inplace_f32 函数已废弃，建议使用 ggml_map_custom3_inplace 替代
    GGML_DEPRECATED(GGML_API struct ggml_tensor * ggml_map_custom3_inplace_f32(
            struct ggml_context          * ctx,
            struct ggml_tensor           * a,
            struct ggml_tensor           * b,
            struct ggml_tensor           * c,
                   ggml_custom3_op_f32_t   fun),
        "use ggml_map_custom3_inplace instead");

    // 定义自定义操作函数指针类型 ggml_custom1_op_t 和 ggml_custom2_op_t
    // ggml_custom1_op_t 接受一个目标张量指针 dst，一个输入张量指针 a，一个整数 ith，一个整数 nth，一个指向用户数据的指针 userdata
    typedef void (*ggml_custom1_op_t)(struct ggml_tensor * dst , const struct ggml_tensor * a, int ith, int nth, void * userdata);
    // ggml_custom2_op_t 接受一个目标张量指针 dst，两个输入张量指针 a 和 b，一个整数 ith，一个整数 nth，一个指向用户数据的指针 userdata
    typedef void (*ggml_custom2_op_t)(struct ggml_tensor * dst , const struct ggml_tensor * a, const struct ggml_tensor * b, int ith, int nth, void * userdata);
    # 定义一个指向函数的指针类型 ggml_custom3_op_t，该函数接受四个 ggml_tensor 类型的参数，两个整型参数 ith 和 nth，以及一个指向 void 类型的指针 userdata
    typedef void (*ggml_custom3_op_t)(struct ggml_tensor * dst , const struct ggml_tensor * a, const struct ggml_tensor * b, const struct ggml_tensor * c, int ith, int nth, void * userdata);

    # 定义一个宏 GGML_N_TASKS_MAX，其值为 -1
    #define GGML_N_TASKS_MAX -1

    # 定义一个函数 ggml_map_custom1，接受 ggml_context 指针类型的参数 ctx，ggml_tensor 指针类型的参数 a，ggml_custom1_op_t 类型的参数 fun，整型参数 n_tasks，以及指向 void 类型的指针 userdata
    GGML_API struct ggml_tensor * ggml_map_custom1(
            struct ggml_context   * ctx,
            struct ggml_tensor    * a,
            ggml_custom1_op_t       fun,
            int                     n_tasks,
            void                  * userdata);

    # 定义一个函数 ggml_map_custom1_inplace，接受 ggml_context 指针类型的参数 ctx，ggml_tensor 指针类型的参数 a，ggml_custom1_op_t 类型的参数 fun，整型参数 n_tasks，以及指向 void 类型的指针 userdata
    GGML_API struct ggml_tensor * ggml_map_custom1_inplace(
            struct ggml_context   * ctx,
            struct ggml_tensor    * a,
            ggml_custom1_op_t       fun,
            int                     n_tasks,
            void                  * userdata);

    # 定义一个函数 ggml_map_custom2，接受 ggml_context 指针类型的参数 ctx，ggml_tensor 指针类型的参数 a，ggml_tensor 指针类型的参数 b，ggml_custom2_op_t 类型的参数 fun，整型参数 n_tasks，以及指向 void 类型的指针 userdata
    GGML_API struct ggml_tensor * ggml_map_custom2(
            struct ggml_context   * ctx,
            struct ggml_tensor    * a,
            struct ggml_tensor    * b,
            ggml_custom2_op_t       fun,
            int                     n_tasks,
            void                  * userdata);

    # 定义一个函数 ggml_map_custom2_inplace，接受 ggml_context 指针类型的参数 ctx，ggml_tensor 指针类型的参数 a，ggml_tensor 指针类型的参数 b，ggml_custom2_op_t 类型的参数 fun，整型参数 n_tasks，以及指向 void 类型的指针 userdata
    GGML_API struct ggml_tensor * ggml_map_custom2_inplace(
            struct ggml_context   * ctx,
            struct ggml_tensor    * a,
            struct ggml_tensor    * b,
            ggml_custom2_op_t       fun,
            int                     n_tasks,
            void                  * userdata);

    # 定义一个函数 ggml_map_custom3，接受 ggml_context 指针类型的参数 ctx，ggml_tensor 指针类型的参数 a，ggml_tensor 指针类型的参数 b，ggml_tensor 指针类型的参数 c，ggml_custom3_op_t 类型的参数 fun，整型参数 n_tasks，以及指向 void 类型的指针 userdata
    GGML_API struct ggml_tensor * ggml_map_custom3(
            struct ggml_context   * ctx,
            struct ggml_tensor    * a,
            struct ggml_tensor    * b,
            struct ggml_tensor    * c,
            ggml_custom3_op_t       fun,
            int                     n_tasks,
            void                  * userdata);
    // 在给定上下文中，将三个张量映射到自定义操作中，返回结果张量
    GGML_API struct ggml_tensor * ggml_map_custom3_inplace(
            struct ggml_context   * ctx,
            struct ggml_tensor    * a,
            struct ggml_tensor    * b,
            struct ggml_tensor    * c,
            ggml_custom3_op_t       fun,
            int                     n_tasks,
            void                  * userdata);

    // 损失函数

    // 计算交叉熵损失
    GGML_API struct ggml_tensor * ggml_cross_entropy_loss(
            struct ggml_context         * ctx,
            struct ggml_tensor          * a,
            struct ggml_tensor          * b);

    // 计算交叉熵损失的反向传播
    GGML_API struct ggml_tensor * ggml_cross_entropy_loss_back(
            struct ggml_context         * ctx,
            struct ggml_tensor          * a,
            struct ggml_tensor          * b,
            struct ggml_tensor          * c);

    //
    // 自动微分
    //

    // 设置参数张量
    GGML_API void ggml_set_param(
            struct ggml_context * ctx,
            struct ggml_tensor  * tensor);


    // 构建前向传播扩展
    GGML_API void ggml_build_forward_expand (struct ggml_cgraph * cgraph, struct ggml_tensor * tensor);
    // 构建反向传播扩展
    GGML_API void ggml_build_backward_expand(struct ggml_context * ctx, struct ggml_cgraph * gf, struct ggml_cgraph * gb, bool keep);

    // 在上下文中分配图
    GGML_API struct ggml_cgraph * ggml_new_graph         (struct ggml_context * ctx); // size = GGML_DEFAULT_GRAPH_SIZE, grads = false
    // 自定义大小和梯度的新图
    GGML_API struct ggml_cgraph * ggml_new_graph_custom  (struct ggml_context * ctx, size_t size, bool grads);
    // 复制图
    GGML_API struct ggml_cgraph * ggml_graph_dup         (struct ggml_context * ctx, struct ggml_cgraph * cgraph);
    // 查看图的子图
    GGML_API struct ggml_cgraph   ggml_graph_view        (struct ggml_cgraph * cgraph, int i0, int i1);
    // 复制图
    GGML_API void                 ggml_graph_cpy         (struct ggml_cgraph * src, struct ggml_cgraph * dst);
    // 重置图，梯度清零
    GGML_API void                 ggml_graph_reset       (struct ggml_cgraph * cgraph);  // zero grads
    // 清空图形对象，释放资源
    GGML_API void ggml_graph_clear(struct ggml_cgraph * cgraph);
    
    // 返回图形对象的开销
    GGML_API size_t ggml_graph_overhead(void);
    // 返回自定义大小和梯度标志下的图形对象开销
    GGML_API size_t ggml_graph_overhead_custom(size_t size, bool grads);
    
    // 在调用 ggml_graph_compute() 之前必须调用 ggml_graph_plan()
    // 当 plan.work_size > 0 时，调用者必须为 plan.work_data 分配内存
    GGML_API struct ggml_cplan ggml_graph_plan(const struct ggml_cgraph * cgraph, int n_threads /*= GGML_DEFAULT_N_THREADS*/);
    GGML_API int ggml_graph_compute(struct ggml_cgraph * cgraph, struct ggml_cplan * cplan);
    
    // 与 ggml_graph_compute() 相同，但工作数据作为上下文的一部分分配
    // 注意：此 API 的缺点是必须确保上下文有足够的内存来存储工作数据
    GGML_API void ggml_graph_compute_with_ctx(struct ggml_context * ctx, struct ggml_cgraph * cgraph, int n_threads);
    
    // 获取图形对象中指定名称的张量
    GGML_API struct ggml_tensor * ggml_graph_get_tensor(struct ggml_cgraph * cgraph, const char * name);
    
    // 导出图形对象到文件
    GGML_API void ggml_graph_export(const struct ggml_cgraph * cgraph, const char * fname);
    // 从文件导入图形对象，并返回上下文数据和评估上下文数据
    GGML_API struct ggml_cgraph * ggml_graph_import(const char * fname, struct ggml_context ** ctx_data, struct ggml_context ** ctx_eval);
    
    // 打印图形的信息和性能信息
    GGML_API void ggml_graph_print(const struct ggml_cgraph * cgraph);
    
    // 使用 dot 格式将图形转储到文件
    GGML_API void ggml_graph_dump_dot(const struct ggml_cgraph * gb, const struct ggml_cgraph * gf, const char * filename);
    
    // 使用提供的检查点构建梯度检查点反向图 gb 用于 gf
    // gb_tmp 将包含具有重写的反向处理节点的原始反向图，但不包含第二次前向传递节点。
    // 声明一个函数，用于构建反向梯度检查点
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
        GGML_LINESEARCH_DEFAULT = 1,  // 默认线搜索方法

        GGML_LINESEARCH_BACKTRACKING_ARMIJO       = 0, // Armijo 回溯线搜索
        GGML_LINESEARCH_BACKTRACKING_WOLFE        = 1, // Wolfe 回溯线搜索
        GGML_LINESEARCH_BACKTRACKING_STRONG_WOLFE = 2, // Strong Wolfe 回溯线搜索
    };

    // 优化返回值的枚举类型
    enum ggml_opt_result {
        GGML_OPT_OK = 0,  // 优化成功
        GGML_OPT_DID_NOT_CONVERGE,  // 未收敛
        GGML_OPT_NO_CONTEXT,  // 无上下文
        GGML_OPT_INVALID_WOLFE,  // 无效的 Wolfe
        GGML_OPT_FAIL,  // 失败
        GGML_OPT_CANCEL,  // 取消

        GGML_LINESEARCH_FAIL = -128,  // 线搜索失败
        GGML_LINESEARCH_MINIMUM_STEP,  // 最小步长
        GGML_LINESEARCH_MAXIMUM_STEP,  // 最大步长
        GGML_LINESEARCH_MAXIMUM_ITERATIONS,  // 最大迭代次数
        GGML_LINESEARCH_INVALID_PARAMETERS,  // 无效参数
    };

    // 优化回调函数类型
    typedef void (*ggml_opt_callback)(void * data, int accum_step, float * sched, bool * cancel);
    // 日志回调函数类型
    typedef void (*ggml_log_callback)(enum ggml_log_level level, const char * text, void * user_data);

    // 优化参数
    //
    //   请参考 ggml.c (ggml_opt_default_params) 获取默认值
    //
    // 定义结构体 ggml_opt_params，包含了优化参数的各种设置
    struct ggml_opt_params {
        // 枚举类型，表示优化类型
        enum ggml_opt_type type;

        // 图的大小
        size_t graph_size;

        // 线程数
        int n_threads;

        // 基于 delta 的收敛测试
        //
        //   如果 past == 0 - 禁用
        //   如果 past > 0:
        //     如果 |f(x) - f(x_past)| < delta * max(1, |f(x)|)，则停止
        //
        int past;
        float delta;

        // 最大迭代次数，没有改进的情况下
        //
        //   如果 0 - 禁用
        //   如果 > 0:
        //     如果在这个迭代次数内没有成本改进，则假定收敛
        //
        int max_no_improvement;

        // 是否打印正向图
        bool print_forward_graph;
        // 是否打印反向图
        bool print_backward_graph;

        // 梯度累积次数
        int n_gradient_accumulation;

        // ADAM 参数
        struct {
            int n_iter;

            float sched; // 调度乘数（固定、衰减或预热）
            float decay; // AdamW 的权重衰减，使用 0.0f 禁用
            int   decay_min_ndim; // 应用权重衰减的最小张量维度
            float alpha; // 学习率
            float beta1;
            float beta2;
            float eps;   // 数值稳定性的 epsilon
            float eps_f; // 收敛测试的 epsilon
            float eps_g; // 收敛测试的 epsilon
            float gclip; // 梯度裁剪
        } adam;

        // LBFGS 参数
        struct {
            int m; // 近似逆 Hessian 矫正次数
            int n_iter;
            int max_linesearch;

            float eps;      // 收敛容限
            float ftol;     // 线搜索容限
            float wolfe;
            float min_step;
            float max_step;

            enum ggml_linesearch linesearch;
        } lbfgs;
    };
    // 定义优化器上下文结构体，包含了优化器的各种参数和状态信息
    struct ggml_opt_context {
        // 指向神经网络上下文结构体的指针
        struct ggml_context * ctx;
        // 优化器的参数
        struct ggml_opt_params params;

        // 迭代次数
        int iter;
        // 参数元素的数量
        int64_t nx; // number of parameter elements

        // 是否刚初始化
        bool just_initialized;

        // 优化前的损失值
        float loss_before;
        // 优化后的损失值
        float loss_after;

        // Adam 优化器的相关信息
        struct {
            // 当前梯度
            struct ggml_tensor * g;
            // 第一时刻
            struct ggml_tensor * m;
            // 第二时刻
            struct ggml_tensor * v;
            // 过去的函数值
            struct ggml_tensor * pf;
            // 最佳函数值
            float fx_best;
            // 上一个函数值
            float fx_prev;
            // 未改进次数
            int n_no_improvement;
        } adam;

        // L-BFGS 优化器的相关信息
        struct {
            // 当前参数
            struct ggml_tensor * x;
            // 上一个参数
            struct ggml_tensor * xp;
            // 当前梯度
            struct ggml_tensor * g;
            // 上一个梯度
            struct ggml_tensor * gp;
            // 搜索方向
            struct ggml_tensor * d;
            // 过去的函数值
            struct ggml_tensor * pf;
            // L-BFGS 内存 alpha
            struct ggml_tensor * lmal;
            // L-BFGS 内存 ys
            struct ggml_tensor * lmys;
            // L-BFGS 内存 s
            struct ggml_tensor * lms;
            // L-BFGS 内存 y
            struct ggml_tensor * lmy;
            // 最佳函数值
            float fx_best;
            // 步长
            float step;
            // j
            int j;
            // k
            int k;
            // end
            int end;
            // 未改进次数
            int n_no_improvement;
        } lbfgs;
    };

    // 获取默认的优化器参数
    GGML_API struct ggml_opt_params ggml_opt_default_params(enum ggml_opt_type type);

    // 优化给定张量定义的函数
    GGML_API enum ggml_opt_result ggml_opt(
            struct ggml_context * ctx,
            struct ggml_opt_params params,
            struct ggml_tensor * f);

    // 初始化优化器上下文
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
    
    // 继续优化由张量 f 定义的函数，同时传入计算图和回调函数等参数
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
    
    // - ggml_quantize_init 可以多次调用相同类型，只会在第一次调用或 ggml_quantize_free 后初始化量化表
    //   为方便起见，ggml_quantize_chunk 会自动调用此函数
    //
    // - ggml_quantize_free 会释放 ggml_quantize_init 分配的内存
    //   在程序结束时调用以避免内存泄漏
    //
    // 注意：这些函数是线程安全的
    //
    GGML_API void ggml_quantize_init(enum ggml_type type);
    GGML_API void ggml_quantize_free(void);
    
    // TODO: 这些函数可能会被移除，以支持更通用的 ggml_quantize_chunk
    GGML_API size_t ggml_quantize_q4_0(const float * src, void * dst, int n, int k, int64_t * hist);
    GGML_API size_t ggml_quantize_q4_1(const float * src, void * dst, int n, int k, int64_t * hist);
    GGML_API size_t ggml_quantize_q5_0(const float * src, void * dst, int n, int k, int64_t * hist);
    GGML_API size_t ggml_quantize_q5_1(const float * src, void * dst, int n, int k, int64_t * hist);
    GGML_API size_t ggml_quantize_q8_0(const float * src, void * dst, int n, int k, int64_t * hist);
    
    GGML_API size_t ggml_quantize_q2_K(const float * src, void * dst, int n, int k, int64_t * hist);
    // 定义函数 ggml_quantize_q3_K，用于将浮点数组 src 量化为 k 个值，并将结果存储到 dst 中，同时计算直方图 hist
    GGML_API size_t ggml_quantize_q3_K(const float * src, void * dst, int n, int k, int64_t * hist);
    // 定义函数 ggml_quantize_q4_K，用于将浮点数组 src 量化为 k 个值，并将结果存储到 dst 中，同时计算直方图 hist
    GGML_API size_t ggml_quantize_q4_K(const float * src, void * dst, int n, int k, int64_t * hist);
    // 定义函数 ggml_quantize_q5_K，用于将浮点数组 src 量化为 k 个值，并将结果存储到 dst 中，同时计算直方图 hist
    GGML_API size_t ggml_quantize_q5_K(const float * src, void * dst, int n, int k, int64_t * hist);
    // 定义函数 ggml_quantize_q6_K，用于将浮点数组 src 量化为 k 个值，并将结果存储到 dst 中，同时计算直方图 hist
    
    // 某些量化类型需要使用重要性矩阵，判断是否需要
    GGML_API bool ggml_quantize_requires_imatrix(enum ggml_type type);
    
    // 调用 ggml_quantize_init 内部函数（即可能分配内存）
    GGML_API size_t ggml_quantize_chunk(enum ggml_type type, const float * src, void * dst,
            int start, int nrows, int n_per_row, int64_t * hist, const float * imatrix);
    
    // gguf 类型枚举
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
        GGUF_TYPE_COUNT,       // 标记枚举结束
    };
    
    // gguf 上下文结构体
    struct gguf_context;
    
    // gguf 初始化参数结构体
    struct gguf_init_params {
        bool no_alloc;
    
        // 如果不为 NULL，则创建一个 ggml_context 并在其中分配张量数据
        struct ggml_context ** ctx;
    };
    
    // 初始化一个空的 gguf 上下文
    GGML_API struct gguf_context * gguf_init_empty(void);
    // 从文件初始化 gguf 上下文
    GGML_API struct gguf_context * gguf_init_from_file(const char * fname, struct gguf_init_params params);
    // 从缓冲区初始化 gguf 上下文
    //GGML_API struct gguf_context * gguf_init_from_buffer(..);
    
    // 释放 gguf 上下文
    GGML_API void gguf_free(struct gguf_context * ctx);
    
    // 获取 gguf 类型名称
    GGML_API const char * gguf_type_name(enum gguf_type type);
    
    // 获取 gguf 上下文的版本号
    GGML_API int    gguf_get_version    (const struct gguf_context * ctx);
    // 获取给定上下文中的对齐方式
    GGML_API size_t gguf_get_alignment  (const struct gguf_context * ctx);
    // 获取给定上下文中数据的偏移量
    GGML_API size_t gguf_get_data_offset(const struct gguf_context * ctx);
    // 获取给定上下文中的数据
    GGML_API void * gguf_get_data       (const struct gguf_context * ctx);
    
    // 获取给定上下文中键值对的数量
    GGML_API int          gguf_get_n_kv(const struct gguf_context * ctx);
    // 在给定上下文中查找指定键的索引
    GGML_API int          gguf_find_key(const struct gguf_context * ctx, const char * key);
    // 获取给定上下文中指定键的键名
    GGML_API const char * gguf_get_key (const struct gguf_context * ctx, int key_id);
    
    // 获取给定上下文中指定键的键值类型
    GGML_API enum gguf_type gguf_get_kv_type (const struct gguf_context * ctx, int key_id);
    // 获取给定上下文中指定键的数组类型
    GGML_API enum gguf_type gguf_get_arr_type(const struct gguf_context * ctx, int key_id);
    
    // 如果使用错误类型的键，则中止操作
    GGML_API uint8_t      gguf_get_val_u8  (const struct gguf_context * ctx, int key_id);
    GGML_API int8_t       gguf_get_val_i8  (const struct gguf_context * ctx, int key_id);
    GGML_API uint16_t     gguf_get_val_u16 (const struct gguf_context * ctx, int key_id);
    GGML_API int16_t      gguf_get_val_i16 (const struct gguf_context * ctx, int key_id);
    GGML_API uint32_t     gguf_get_val_u32 (const struct gguf_context * ctx, int key_id);
    GGML_API int32_t      gguf_get_val_i32 (const struct gguf_context * ctx, int key_id);
    GGML_API float        gguf_get_val_f32 (const struct gguf_context * ctx, int key_id);
    GGML_API uint64_t     gguf_get_val_u64 (const struct gguf_context * ctx, int key_id);
    GGML_API int64_t      gguf_get_val_i64 (const struct gguf_context * ctx, int key_id);
    GGML_API double       gguf_get_val_f64 (const struct gguf_context * ctx, int key_id);
    GGML_API bool         gguf_get_val_bool(const struct gguf_context * ctx, int key_id);
    GGML_API const char * gguf_get_val_str (const struct gguf_context * ctx, int key_id);
    GGML_API const void * gguf_get_val_data(const struct gguf_context * ctx, int key_id);
    GGML_API int          gguf_get_arr_n   (const struct gguf_context * ctx, int key_id);
    // 获取指定键值对应的数组数据
    GGML_API const void * gguf_get_arr_data(const struct gguf_context * ctx, int key_id);
    // 获取指定键值对应的字符串数据
    GGML_API const char * gguf_get_arr_str (const struct gguf_context * ctx, int key_id, int i);
    
    // 获取张量的数量
    GGML_API int            gguf_get_n_tensors    (const struct gguf_context * ctx);
    // 查找指定名称的张量
    GGML_API int            gguf_find_tensor      (const struct gguf_context * ctx, const char * name);
    // 获取指定索引处张量的偏移量
    GGML_API size_t         gguf_get_tensor_offset(const struct gguf_context * ctx, int i);
    // 获取指定索引处张量的名称
    GGML_API char *         gguf_get_tensor_name  (const struct gguf_context * ctx, int i);
    // 获取指定索引处张量的类型
    GGML_API enum ggml_type gguf_get_tensor_type  (const struct gguf_context * ctx, int i);
    
    // 覆盖现有值或添加新值（不同数据类型）
    GGML_API void gguf_set_val_u8  (struct gguf_context * ctx, const char * key, uint8_t  val);
    GGML_API void gguf_set_val_i8  (struct gguf_context * ctx, const char * key, int8_t   val);
    GGML_API void gguf_set_val_u16 (struct gguf_context * ctx, const char * key, uint16_t val);
    GGML_API void gguf_set_val_i16 (struct gguf_context * ctx, const char * key, int16_t  val);
    GGML_API void gguf_set_val_u32 (struct gguf_context * ctx, const char * key, uint32_t val);
    GGML_API void gguf_set_val_i32 (struct gguf_context * ctx, const char * key, int32_t  val);
    GGML_API void gguf_set_val_f32 (struct gguf_context * ctx, const char * key, float    val);
    GGML_API void gguf_set_val_u64 (struct gguf_context * ctx, const char * key, uint64_t val);
    GGML_API void gguf_set_val_i64 (struct gguf_context * ctx, const char * key, int64_t  val);
    GGML_API void gguf_set_val_f64 (struct gguf_context * ctx, const char * key, double   val);
    GGML_API void gguf_set_val_bool(struct gguf_context * ctx, const char * key, bool     val);
    GGML_API void gguf_set_val_str (struct gguf_context * ctx, const char * key, const char * val);
    // 设置数组数据
    GGML_API void gguf_set_arr_data(struct gguf_context * ctx, const char * key, enum gguf_type type, const void * data, int n);
    // 设置一个字符串数组到上下文中的键值对
    GGML_API void gguf_set_arr_str (struct gguf_context * ctx, const char * key, const char ** data, int n);
    
    // 从另一个上下文中设置或添加键值对
    GGML_API void gguf_set_kv(struct gguf_context * ctx, struct gguf_context * src);
    
    // 管理张量信息
    GGML_API void gguf_add_tensor(struct gguf_context * ctx, const struct ggml_tensor * tensor);
    GGML_API void gguf_set_tensor_type(struct gguf_context * ctx, const char * name, enum ggml_type type);
    GGML_API void gguf_set_tensor_data(struct gguf_context * ctx, const char * name, const void * data, size_t size);
    
    // 写入 gguf 文件有两种方式：
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
    
    // 获取元数据的字节大小（头部、键值对、张量信息）包括填充
    GGML_API size_t gguf_get_meta_size(const struct gguf_context * ctx);
    GGML_API void   gguf_get_meta_data(const struct gguf_context * ctx, void * data);
    
    //
    // 系统信息
    //
    
    GGML_API int ggml_cpu_has_avx        (void);
    GGML_API int ggml_cpu_has_avx_vnni   (void);
    GGML_API int ggml_cpu_has_avx2       (void);
    GGML_API int ggml_cpu_has_avx512     (void);
    GGML_API int ggml_cpu_has_avx512_vbmi(void);
    GGML_API int ggml_cpu_has_avx512_vnni(void);
    GGML_API int ggml_cpu_has_fma        (void);
    // 检查 CPU 是否支持 NEON 指令集
    GGML_API int ggml_cpu_has_neon       (void);
    // 检查 CPU 是否支持 ARM FMA 指令集
    GGML_API int ggml_cpu_has_arm_fma    (void);
    // 检查 CPU 是否支持 Metal 框架
    GGML_API int ggml_cpu_has_metal      (void);
    // 检查 CPU 是否支持 F16C 指令集
    GGML_API int ggml_cpu_has_f16c       (void);
    // 检查 CPU 是否支持 FP16 VA
    GGML_API int ggml_cpu_has_fp16_va    (void);
    // 检查 CPU 是否支持 WebAssembly SIMD
    GGML_API int ggml_cpu_has_wasm_simd  (void);
    // 检查 CPU 是否支持 BLAS 库
    GGML_API int ggml_cpu_has_blas       (void);
    // 检查 CPU 是否支持 cuBLAS 库
    GGML_API int ggml_cpu_has_cublas     (void);
    // 检查 CPU 是否支持 CLBlast 库
    GGML_API int ggml_cpu_has_clblast    (void);
    // 检查 CPU 是否支持 Vulkan API
    GGML_API int ggml_cpu_has_vulkan     (void);
    // 检查 CPU 是否支持 GPU BLAS 库
    GGML_API int ggml_cpu_has_gpublas    (void);
    // 检查 CPU 是否支持 SSE3 指令集
    GGML_API int ggml_cpu_has_sse3       (void);
    // 检查 CPU 是否支持 SSSE3 指令集
    GGML_API int ggml_cpu_has_ssse3      (void);
    // 检查 CPU 是否支持 SYCL 标准
    GGML_API int ggml_cpu_has_sycl       (void);
    // 检查 CPU 是否支持 VSX 指令集
    GGML_API int ggml_cpu_has_vsx        (void);

    //
    // 用于测试和基准测试的内部类型和函数
    //
#ifdef  __cplusplus
// 如果是 C++ 编译环境，则定义 GGML_RESTRICT 为空
#define GGML_RESTRICT
#else
// 如果不是 C++ 编译环境，则定义 GGML_RESTRICT 为 restrict
#define GGML_RESTRICT restrict
#endif

// 定义函数指针类型 ggml_to_float_t，用于将数据转换为 float 类型
typedef void (*ggml_to_float_t)  (const void  * GGML_RESTRICT x, float * GGML_RESTRICT y, int k);
// 定义函数指针类型 ggml_from_float_t，用于将 float 类型数据转换为其他类型
typedef void (*ggml_from_float_t)(const float * GGML_RESTRICT x, void  * GGML_RESTRICT y, int k);
// 定义函数指针类型 ggml_vec_dot_t，用于计算向量点积
typedef void (*ggml_vec_dot_t)   (const int n, float * GGML_RESTRICT s, const void * GGML_RESTRICT x, const void * GGML_RESTRICT y);

// 定义结构体 ggml_type_traits_t，存储类型相关的信息
typedef struct {
    const char      * type_name; // 类型名称
    int               blck_size; // 块大小
    size_t            type_size; // 类型大小
    bool              is_quantized; // 是否量化
    ggml_to_float_t   to_float; // 将数据转换为 float 类型的函数指针
    ggml_from_float_t from_float; // 将 float 类型数据转换为其他类型的函数指针
    ggml_from_float_t from_float_reference; // 参考的从 float 类型数据转换为其他类型的函数指针
    ggml_vec_dot_t    vec_dot; // 计算向量点积的函数指针
    enum ggml_type    vec_dot_type; // 向量点积的类型
} ggml_type_traits_t;

// 声明获取类型特征信息的函数
GGML_API ggml_type_traits_t ggml_internal_get_type_traits(enum ggml_type type);

#ifdef  __cplusplus
}
#endif
```