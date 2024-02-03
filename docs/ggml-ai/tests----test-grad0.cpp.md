# `ggml\tests\test-grad0.cpp`

```cpp
// 禁用 Windows 上荒谬的“不安全”警告
#define _CRT_SECURE_NO_DEPRECATE 
#include "ggml.h"

#include <cmath> // 包含数学函数库
#include <cstdio> // 包含输入输出函数库
#include <cstdlib> // 包含通用工具函数库
#include <cassert> // 包含断言函数库

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // 可能丢失数据的警告
#endif

#if defined(__GNUC__)
#pragma GCC diagnostic ignored "-Wdouble-promotion"
#endif

#define MAX_NARGS 3 // 定义最大参数个数

#undef MIN // 取消 MIN 宏定义
#undef MAX // 取消 MAX 宏定义
#define MIN(a, b) ((a) < (b) ? (a) : (b)) // 定义取最小值的宏
#define MAX(a, b) ((a) > (b) ? (a) : (b)) // 定义取最大值的宏

#define GGML_SILU_FP16 // 定义 GGML_SILU_FP16

//
// logging
//

#if (GGML_DEBUG >= 1)
#define GGML_PRINT_DEBUG(...) printf(__VA_ARGS__) // 如果调试级别大于等于1，定义调试打印宏
#else
#define GGML_PRINT_DEBUG(...) // 否则为空
#endif

#if (GGML_DEBUG >= 5)
#define GGML_PRINT_DEBUG_5(...) printf(__VA_ARGS__) // 如果调试级别大于等于5，定义调试打印宏
#else
#define GGML_PRINT_DEBUG_5(...) // 否则为空
#endif

#if (GGML_DEBUG >= 10)
#define GGML_PRINT_DEBUG_10(...) printf(__VA_ARGS__) // 如果调试级别大于等于10，定义调试打印宏
#else
#define GGML_PRINT_DEBUG_10(...) // 否则为空
#endif

#define GGML_PRINT(...) printf(__VA_ARGS__) // 定义打印宏

static float frand(void) {
    return (float)rand()/(float)RAND_MAX; // 返回 0 到 1 之间的随机浮点数
}

static int irand(int n) {
    if (n == 0) return 0; // 如果 n 为 0，返回 0
    return rand()%n; // 返回 0 到 n-1 之间的随机整数
}

static void get_random_dims(int64_t * dims, int ndims) {
    dims[0] = dims[1] = dims[2] = dims[3] = 1; // 初始化维度数组

    for (int i = 0; i < ndims; i++) { // 遍历维度数组
        dims[i] = 1 + irand(4); // 生成 1 到 4 之间的随机数，赋值给维度数组
    }
}

static struct ggml_tensor * get_random_tensor_f32(
        struct ggml_context * ctx0,
        int ndims,
        int64_t ne[],
        float fmin,
        float fmax) {
    struct ggml_tensor * result = ggml_new_tensor(ctx0, GGML_TYPE_F32, ndims, ne); // 创建一个新的浮点型张量
    # 根据维度的不同，使用不同的循环方式为结果数组赋值
    switch (ndims) {
        # 当维度为1时，使用单层循环为结果数组赋值
        case 1:
            for (int i0 = 0; i0 < ne[0]; i0++) {
                ((float *)result->data)[i0] = frand()*(fmax - fmin) + fmin;
            }
            break;
        # 当维度为2时，使用嵌套循环为结果数组赋值
        case 2:
            for (int i1 = 0; i1 < ne[1]; i1++) {
                for (int i0 = 0; i0 < ne[0]; i0++) {
                    ((float *)result->data)[i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
                }
            }
            break;
        # 当维度为3时，使用三层嵌套循环为结果数组赋值
        case 3:
            for (int i2 = 0; i2 < ne[2]; i2++) {
                for (int i1 = 0; i1 < ne[1]; i1++) {
                    for (int i0 = 0; i0 < ne[0]; i0++) {
                        ((float *)result->data)[i2*ne[1]*ne[0] + i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
                    }
                }
            }
            break;
        # 当维度为4时，使用四层嵌套循环为结果数组赋值
        case 4:
            for (int i3 = 0; i3 < ne[3]; i3++) {
                for (int i2 = 0; i2 < ne[2]; i2++) {
                    for (int i1 = 0; i1 < ne[1]; i1++) {
                        for (int i0 = 0; i0 < ne[0]; i0++) {
                            ((float *)result->data)[i3*ne[2]*ne[1]*ne[0] + i2*ne[1]*ne[0] + i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
                        }
                    }
                }
            }
            break;
        # 默认情况下，如果维度不在1到4之间，触发断言错误
        default:
            assert(false);
    }

    # 返回结果数组
    return result;
# 返回一个随机生成的浮点数张量，数据类型为半精度浮点数
static struct ggml_tensor * get_random_tensor_f16(
        struct ggml_context * ctx0,
        int ndims,
        int64_t ne[],
        float fmin,
        float fmax) {
    # 创建一个新的张量对象，数据类型为半精度浮点数
    struct ggml_tensor * result = ggml_new_tensor(ctx0, GGML_TYPE_F16, ndims, ne);

    # 根据张量的维度进行不同情况的处理
    switch (ndims) {
        # 当维度为1时
        case 1:
            # 遍历第一维度，生成随机的半精度浮点数并赋值给张量的数据
            for (int i0 = 0; i0 < ne[0]; i0++) {
                ((ggml_fp16_t *)result->data)[i0] = ggml_fp32_to_fp16(frand()*(fmax - fmin) + fmin);
            }
            break;
        # 当维度为2时
        case 2:
            # 遍历第二维度和第一维度，生成随机的半精度浮点数并赋值给张量的数据
            for (int i1 = 0; i1 < ne[1]; i1++) {
                for (int i0 = 0; i0 < ne[0]; i0++) {
                    ((ggml_fp16_t *)result->data)[i1*ne[0] + i0] = ggml_fp32_to_fp16(frand()*(fmax - fmin) + fmin);
                }
            }
            break;
        # 当维度为3时
        case 3:
            # 遍历第三维度、第二维度和第一维度，生成随机的半精度浮点数并赋值给张量的数据
            for (int i2 = 0; i2 < ne[2]; i2++) {
                for (int i1 = 0; i1 < ne[1]; i1++) {
                    for (int i0 = 0; i0 < ne[0]; i0++) {
                        ((ggml_fp16_t *)result->data)[i2*ne[1]*ne[0] + i1*ne[0] + i0] = ggml_fp32_to_fp16(frand()*(fmax - fmin) + fmin);
                    }
                }
            }
            break;
        # 当维度为4时
        case 4:
            # 遍历第四维度、第三维度、第二维度和第一维度，生成随机的半精度浮点数并赋值给张量的数据
            for (int i3 = 0; i3 < ne[3]; i3++) {
                for (int i2 = 0; i2 < ne[2]; i2++) {
                    for (int i1 = 0; i1 < ne[1]; i1++) {
                        for (int i0 = 0; i0 < ne[0]; i0++) {
                            ((ggml_fp16_t *)result->data)[i3*ne[2]*ne[1]*ne[0] + i2*ne[1]*ne[0] + i1*ne[0] + i0] = ggml_fp32_to_fp16(frand()*(fmax - fmin) + fmin);
                        }
                    }
                }
            }
            break;
        # 默认情况下，断言为假
        default:
            assert(false);
    }

    # 返回生成的张量
    return result;
}

# 返回一个随机生成的整型张量，数据类型为32位整数
static struct ggml_tensor * get_random_tensor_i32(
        struct ggml_context * ctx0,
        int ndims,
        int64_t ne[],
        int32_t imin,
        int32_t imax) {
    # 创建一个新的张量对象，数据类型为32位整数
    struct ggml_tensor * result = ggml_new_tensor(ctx0, GGML_TYPE_I32, ndims, ne);
    # 根据维度数量进行不同的处理
    switch (ndims) {
        # 当维度为1时
        case 1:
            # 遍历第一维度，生成随机数并存入结果数组
            for (int i0 = 0; i0 < ne[0]; i0++) {
                ((int32_t *)result->data)[i0] = irand(imax - imin) + imin;
            }
            break;
        # 当维度为2时
        case 2:
            # 遍历第二维度和第一维度，生成随机数并存入结果数组
            for (int i1 = 0; i1 < ne[1]; i1++) {
                for (int i0 = 0; i0 < ne[0]; i0++) {
                    ((int32_t *)result->data)[i1*ne[0] + i0] = irand(imax - imin) + imin;
                }
            }
            break;
        # 当维度为3时
        case 3:
            # 遍历第三维度、第二维度和第一维度，生成随机数并存入结果数组
            for (int i2 = 0; i2 < ne[2]; i2++) {
                for (int i1 = 0; i1 < ne[1]; i1++) {
                    for (int i0 = 0; i0 < ne[0]; i0++) {
                        ((int32_t *)result->data)[i2*ne[1]*ne[0] + i1*ne[0] + i0] = irand(imax - imin) + imin;
                    }
                }
            }
            break;
        # 当维度为4时
        case 4:
            # 遍历第四维度、第三维度、第二维度和第一维度，生成随机数并存入结果数组
            for (int i3 = 0; i3 < ne[3]; i3++) {
                for (int i2 = 0; i2 < ne[2]; i2++) {
                    for (int i1 = 0; i1 < ne[1]; i1++) {
                        for (int i0 = 0; i0 < ne[0]; i0++) {
                            ((int32_t *)result->data)[i3*ne[2]*ne[1]*ne[0] + i2*ne[1]*ne[0] + i1*ne[0] + i0] = irand(imax - imin) + imin;
                        }
                    }
                }
            }
            break;
        # 默认情况下
        default:
            # 断言为假，即维度数量不符合预期
            assert(false);
    }

    # 返回结果数组
    return result;
    // 检查梯度的函数，用于验证反向传播的正确性
    static bool check_gradient(
        // 操作的名称
        const char * op_name,
        // 上下文指针
        struct ggml_context * ctx0,
        // 输入张量数组
        struct ggml_tensor * x[],
        // 输出张量
        struct ggml_tensor * f,
        // 张量维度
        int ndims,
        // 参数个数
        int nargs,
        // 计算梯度时的步长
        float eps,
        // 最大绝对误差
        float max_error_abs,
        // 最大相对误差
        float max_error_rel) {

    // 静态变量，用于记录线程数，默认为-1
    static int n_threads = -1;
    // 如果线程数小于0，设置为默认线程数
    if (n_threads < 0) {
        n_threads = GGML_DEFAULT_N_THREADS;

        // 从环境变量中获取线程数
        const char *env = getenv("GGML_N_THREADS");
        if (env) {
            n_threads = atoi(env);
        }

        // 打印线程数
        printf("GGML_N_THREADS = %d\n", n_threads);
    }

    // 创建前向计算图
    struct ggml_cgraph * gf = ggml_new_graph_custom(ctx0, GGML_DEFAULT_GRAPH_SIZE, true);
    // 创建后向计算图
    struct ggml_cgraph * gb = ggml_new_graph_custom(ctx0, GGML_DEFAULT_GRAPH_SIZE, true);
    // 构建前向计算图
    ggml_build_forward_expand(gf, f);
    // 复制前向计算图到后向计算图
    ggml_graph_cpy(gf, gb);
    // 构建后向计算图
    ggml_build_backward_expand(ctx0, gf, gb, false);

    // 使用指定上下文计算前向计算图
    ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

    // 重置前向计算图
    ggml_graph_reset  (gf);
    // 设置输出张量的梯度为1.0
    ggml_set_f32      (f->grad, 1.0f);

    // 使用指定上下文计算后向计算图
    ggml_graph_compute_with_ctx(ctx0, gb, n_threads);

    // 输出前向计算图和后向计算图的结构
    // ggml_graph_dump_dot(gf, NULL, "test-grad0-forward.dot");
    // ggml_graph_dump_dot(gb, gf,  "test-grad0-backward.dot");
}
    // 遍历输入参数列表
    for (int i = 0; i < nargs; ++i) {
        // 获取当前参数的元素个数
        const int nelements = ggml_nelements(x[i]);
        // 遍历当前参数的每个元素
        for (int k = 0; k < nelements; ++k) {
            // 使用有限差分计算梯度
            const float x0 = ggml_get_f32_1d(x[i], k);
            const float xm = x0 - eps;
            const float xp = x0 + eps;
            ggml_set_f32_1d(x[i], k, xp);

            ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

            const double f0 = ggml_get_f32_1d(f, 0);

            ggml_set_f32_1d(x[i], k, xm);

            ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

            const double f1 = ggml_get_f32_1d(f, 0);
            const double g0 = (f0 - f1)/(2.0*(double) eps);

            ggml_set_f32_1d(x[i], k, x0);

            // 使用反向图计算梯度
            ggml_graph_reset  (gf);
            ggml_set_f32      (f->grad, 1.0f);

            ggml_graph_compute_with_ctx(ctx0, gb, n_threads);

            const double g1 = ggml_get_f32_1d(x[i]->grad, k);

            // 计算绝对误差和相对误差
            const double error_abs = fabs(g0 - g1);
            const double error_rel = g0 != 0 ? fabs(g0 - g1)/fabs(g0) : 0;

            // 如果误差超过阈值，输出错误信息并返回 false
            if (error_abs > max_error_abs || error_rel > max_error_rel) {
                printf("%s: ndims=%d, i=%d, k=%d, x0=%f, xm=%f, xp=%f, f0=%f, f1=%f, g0=%f, g1=%f, eps=%f, error_abs=%f, error_rel=%f\n",
                            op_name, ndims, i, k, x0, xm, xp, f0, f1, g0, g1, eps, error_abs, error_rel);
                //assert(false);
                return false;
            }
        }
    }

    return true;
// 静态函数，用于检查矩阵相乘的结果是否正确
static bool check_mat_mul(
        const struct ggml_tensor * y,  // 结果矩阵
        const struct ggml_tensor * x0, // 矩阵 x0
        const struct ggml_tensor * x1) { // 矩阵 x1
    float * dst  = (float *) y->data;  // 结果矩阵数据
    float * src0 = (float *) x0->data; // 矩阵 x0 数据
    float * src1 = (float *) x1->data; // 矩阵 x1 数据

    const int nc = x0->ne[1]; // x0 列数
    const int nr = x1->ne[1]; // x1 行数
    const int nk = x0->ne[0]; // x0 行数

    GGML_PRINT_DEBUG("check_mat_mul: nc=%d, nr=%d, nk=%d\n", nc, nr, nk); // 打印调试信息

    GGML_PRINT_DEBUG("x0:\n"); // 打印调试信息
    for (int j = 0; j < x0->ne[1]; ++j) { // 循环遍历 x0 列
        for (int i = 0; i < x0->ne[0]; ++i) { // 循环遍历 x0 行
            GGML_PRINT_DEBUG("%6.3f ", src0[j*nk + i]); // 打印调试信息
        }
        GGML_PRINT_DEBUG("\n"); // 打印调试信息
    }
    GGML_PRINT_DEBUG("\n"); // 打印调试信息

    GGML_PRINT_DEBUG("x1:\n"); // 打印调试信息
    for (int j = 0; j < x1->ne[1]; ++j) { // 循环遍历 x1 列
        for (int i = 0; i < x1->ne[0]; ++i) { // 循环遍历 x1 行
            GGML_PRINT_DEBUG("%6.3f ", src1[j*nk + i]); // 打印调试信息
        }
        GGML_PRINT_DEBUG("\n"); // 打印调试信息
    }
    GGML_PRINT_DEBUG("\n"); // 打印调试信息

    GGML_PRINT_DEBUG("y: n_dims = %d, (%lld, %lld)\n", y->n_dims, y->ne[0], y->ne[1]); // 打印调试信息
    for (int j = 0; j < y->ne[1]; ++j) { // 循环遍历结果矩阵的列
        for (int i = 0; i < y->ne[0]; ++i) { // 循环遍历结果矩阵的行
            GGML_PRINT_DEBUG("%6.3f ", dst[j*nr + i]); // 打印调试信息
        }
        GGML_PRINT_DEBUG("\n"); // 打印调试信息
    }

    for (int i = 0; i < nr; ++i) { // 循环遍历 x1 行
        for (int j = 0; j < nc; ++j) { // 循环遍历 x0 列
            float sum = 0.0f;

            for (int k = 0; k < nk; ++k) { // 循环遍历 x0 行
                sum += src0[j*nk + k]*src1[i*nk + k]; // 计算矩阵相乘结果
            }

            if (fabsf(dst[i*nc + j] - sum) > 1e-5f) { // 判断矩阵相乘结果是否正确
                fprintf(stderr, "check_mat_mul: dst[%d] = %f, sum = %f\n", i*nc + j, dst[i*nc + j], sum); // 打印错误信息
                assert(false); // 断言，如果结果不正确则终止程序
                return false; // 返回 false
            }
        }
    }

    return true; // 返回 true
}

// 定义宏，表示排列组合的数量
#define NUM_PERMUTATIONS (4*3*2*1)

// 主函数
int main(int argc, const char ** argv) {
    // 初始化参数结构体
    struct ggml_init_params params = {
        /* .mem_size   = */ 256*1024*1024, // 内存大小
        /* .mem_buffer = */ NULL, // 内存缓冲区
        /* .no_alloc   = */ false, // 是否允许分配内存
    };

    int64_t ne[4]; // 用于存储 4 个整数值
    # 创建一个包含所有排列的数组，每个排列有4个元素
    int all_permutations[4 * NUM_PERMUTATIONS];
    {
        # 初始化计数器
        int count = 0;
        # 循环遍历4个轴的排列
        for (int ax0=0; ax0<4; ++ax0) {
            for (int ax1=0; ax1<4; ++ax1) {
                # 如果轴1和轴0相同，则跳过
                if (ax1 == ax0) continue;
                for (int ax2=0; ax2<4; ++ax2) {
                    # 如果轴2和轴0相同，则跳过
                    if (ax2 == ax0) continue;
                    # 如果轴2和轴1相同，则跳过
                    if (ax2 == ax1) continue;
                    for (int ax3=0; ax3<4; ++ax3) {
                        # 如果轴3和轴0相同，则跳过
                        if (ax3 == ax0) continue;
                        # 如果轴3和轴1相同，则跳过
                        if (ax3 == ax1) continue;
                        # 如果轴3和轴2相同，则跳过
                        if (ax3 == ax2) continue;
                        # 断言计数器小于排列数
                        assert(count < NUM_PERMUTATIONS);
                        # 将排列添加到数组中
                        all_permutations[count*4+0] = ax0;
                        all_permutations[count*4+1] = ax1;
                        all_permutations[count*4+2] = ax2;
                        all_permutations[count*4+3] = ax3;
                        # 更新计数器
                        ++count;
                    }
                }
            }
        }
    }

    # 初始化随机数种子
    unsigned seed_iter = 1;

    # 设置迭代次数，默认为4
    int niter = 4;
    # 从环境变量中获取迭代次数
    const char *env = getenv("GGML_NLOOP");
    if (env != NULL) {
        niter = atoi(env);
    }
    # 如果命令行参数中指定了迭代次数，则使用命令行参数中的值
    if (argc > 1) {
        niter = atoi(argv[1]);
    }
#ifdef GGML_SILU_FP16
                // 如果定义了 GGML_SILU_FP16，则由于 GGML_SILU_FP16，有限差分法会略有错误 -> 增加误差边界。
                check_gradient("silu", ctx0, x, f, ndims, nargs, 1e-3f, 0.5, INFINITY);
#else
                // 如果未定义 GGML_SILU_FP16，则正常进行梯度检查
                check_gradient("silu", ctx0, x, f, ndims, nargs, 1e-3f, 1e-3f, INFINITY);
    }

    // 返回 0，表示正常结束
    return 0;
}
```