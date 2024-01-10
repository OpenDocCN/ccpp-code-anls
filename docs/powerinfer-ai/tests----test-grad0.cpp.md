# `PowerInfer\tests\test-grad0.cpp`

```
#define _CRT_SECURE_NO_DEPRECATE // Disables ridiculous "unsafe" warnigns on Windows
#include "ggml.h"

#include <cmath>
#include <cstdio>
#include <cstdlib>
#include <cassert>

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

#if defined(__GNUC__)
#pragma GCC diagnostic ignored "-Wdouble-promotion"
#endif

#define MAX_NARGS 3

#undef MIN
#undef MAX
#define MIN(a, b) ((a) < (b) ? (a) : (b)) // 定义取最小值的宏
#define MAX(a, b) ((a) > (b) ? (a) : (b)) // 定义取最大值的宏

#define GGML_SILU_FP16

//
// logging
//

#if (GGML_DEBUG >= 1)
#define GGML_PRINT_DEBUG(...) printf(__VA_ARGS__) // 如果调试级别大于等于1，打印调试信息
#else
#define GGML_PRINT_DEBUG(...)
#endif

#if (GGML_DEBUG >= 5)
#define GGML_PRINT_DEBUG_5(...) printf(__VA_ARGS__) // 如果调试级别大于等于5，打印调试信息
#else
#define GGML_PRINT_DEBUG_5(...)
#endif

#if (GGML_DEBUG >= 10)
#define GGML_PRINT_DEBUG_10(...) printf(__VA_ARGS__) // 如果调试级别大于等于10，打印调试信息
#else
#define GGML_PRINT_DEBUG_10(...)
#endif

#define GGML_PRINT(...) printf(__VA_ARGS__) // 打印信息

static float frand(void) {
    return (float)rand()/(float)RAND_MAX; // 生成一个随机浮点数
}

static int irand(int n) {
    if (n == 0) return 0;
    return rand()%n; // 生成一个小于n的随机整数
}

static void get_random_dims(int64_t * dims, int ndims) {
    dims[0] = dims[1] = dims[2] = dims[3] = 1; // 初始化维度数组

    for (int i = 0; i < ndims; i++) {
        dims[i] = 1 + irand(4); // 生成随机维度
    }
}

static struct ggml_tensor * get_random_tensor_f32(
        struct ggml_context * ctx0,
        int ndims,
        int64_t ne[],
        float fmin,
        float fmax) {
    struct ggml_tensor * result = ggml_new_tensor(ctx0, GGML_TYPE_F32, ndims, ne); // 生成一个新的浮点型张量
    # 根据维度的不同，使用不同的循环方式对结果数组进行赋值
    switch (ndims) {
        # 当维度为1时，使用单层循环对结果数组进行赋值
        case 1:
            for (int i0 = 0; i0 < ne[0]; i0++) {
                ((float *)result->data)[i0] = frand()*(fmax - fmin) + fmin;
            }
            break;
        # 当维度为2时，使用嵌套循环对结果数组进行赋值
        case 2:
            for (int i1 = 0; i1 < ne[1]; i1++) {
                for (int i0 = 0; i0 < ne[0]; i0++) {
                    ((float *)result->data)[i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
                }
            }
            break;
        # 当维度为3时，使用三层嵌套循环对结果数组进行赋值
        case 3:
            for (int i2 = 0; i2 < ne[2]; i2++) {
                for (int i1 = 0; i1 < ne[1]; i1++) {
                    for (int i0 = 0; i0 < ne[0]; i0++) {
                        ((float *)result->data)[i2*ne[1]*ne[0] + i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
                    }
                }
            }
            break;
        # 当维度为4时，使用四层嵌套循环对结果数组进行赋值
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
        # 默认情况下，维度不应超过4，否则断言失败
        default:
            assert(false);
    }

    # 返回赋值后的结果数组
    return result;
# 返回一个随机生成的浮点数张量，数据类型为半精度浮点数
static struct ggml_tensor * get_random_tensor_f16(
        # 获取上下文和张量的维度信息
        struct ggml_context * ctx0,
        int ndims,
        int64_t ne[],
        float fmin,
        float fmax) {
    # 创建一个新的半精度浮点数张量
    struct ggml_tensor * result = ggml_new_tensor(ctx0, GGML_TYPE_F16, ndims, ne);

    # 根据张量的维度进行不同情况的处理
    switch (ndims) {
        # 当维度为1时
        case 1:
            # 遍历张量的第一维度，生成随机的半精度浮点数并赋值给张量
            for (int i0 = 0; i0 < ne[0]; i0++) {
                ((ggml_fp16_t *)result->data)[i0] = ggml_fp32_to_fp16(frand()*(fmax - fmin) + fmin);
            }
            break;
        # 当维度为2时
        case 2:
            # 遍历张量的第二维度和第一维度，生成随机的半精度浮点数并赋值给张量
            for (int i1 = 0; i1 < ne[1]; i1++) {
                for (int i0 = 0; i0 < ne[0]; i0++) {
                    ((ggml_fp16_t *)result->data)[i1*ne[0] + i0] = ggml_fp32_to_fp16(frand()*(fmax - fmin) + fmin);
                }
            }
            break;
        # 当维度为3时
        case 3:
            # 遍历张量的第三维度、第二维度和第一维度，生成随机的半精度浮点数并赋值给张量
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
            # 遍历张量的第四维度、第三维度、第二维度和第一维度，生成随机的半精度浮点数并赋值给张量
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
        # 默认情况下
        default:
            # 断言，如果出现默认情况则为假
            assert(false);
    }

    # 返回生成的张量
    return result;
}

# 返回一个随机生成的整型张量，数据类型为32位整数
static struct ggml_tensor * get_random_tensor_i32(
        # 获取上下文和张量的维度信息
        struct ggml_context * ctx0,
        int ndims,
        int64_t ne[],
        int32_t imin,
        int32_t imax) {
    # 创建一个新的32位整数张量
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

    // 静态变量，用于存储线程数，默认为-1
    static int n_threads = -1;
    // 如果线程数小于0，则设置为默认线程数
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

    // 使用指定上下文和线程数计算前向计算图
    ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

    // 重置前向计算图
    ggml_graph_reset  (gf);
    // 设置输出张量的梯度为1.0
    ggml_set_f32      (f->grad, 1.0f);

    // 使用指定上下文和线程数计算后向计算图
    ggml_graph_compute_with_ctx(ctx0, gb, n_threads);

    // 输出前向和后向计算图的结构到.dot文件，用于调试
    // ggml_graph_dump_dot(gf, NULL, "test-grad0-forward.dot");
    // ggml_graph_dump_dot(gb, gf,  "test-grad0-backward.dot");
}
    for (int i = 0; i < nargs; ++i) {
        // 获取输入参数 x[i] 的元素个数
        const int nelements = ggml_nelements(x[i]);
        // 遍历输入参数 x[i] 的每个元素
        for (int k = 0; k < nelements; ++k) {
            // 使用有限差分法计算梯度
            const float x0 = ggml_get_f32_1d(x[i], k);
            const float xm = x0 - eps;
            const float xp = x0 + eps;
            ggml_set_f32_1d(x[i], k, xp);

            ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

            // 获取计算结果 f 的第一个元素
            const double f0 = ggml_get_f32_1d(f, 0);

            ggml_set_f32_1d(x[i], k, xm);

            ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

            // 获取计算结果 f 的第一个元素
            const double f1 = ggml_get_f32_1d(f, 0);
            // 计算梯度
            const double g0 = (f0 - f1)/(2.0*(double) eps);

            ggml_set_f32_1d(x[i], k, x0);

            // 使用反向图计算梯度
            ggml_graph_reset  (gf);
            ggml_set_f32      (f->grad, 1.0f);

            ggml_graph_compute_with_ctx(ctx0, gb, n_threads);

            // 获取 x[i]->grad 的第 k 个元素，即梯度
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
// 检查矩阵相乘的函数，接受三个 ggml_tensor 结构体指针作为参数
static bool check_mat_mul(
        const struct ggml_tensor * y,
        const struct ggml_tensor * x0,
        const struct ggml_tensor * x1) {
    // 将输出结果的数据指针转换为 float 类型指针
    float * dst  = (float *) y->data;
    // 将输入矩阵 x0 的数据指针转换为 float 类型指针
    float * src0 = (float *) x0->data;
    // 将输入矩阵 x1 的数据指针转换为 float 类型指针
    float * src1 = (float *) x1->data;

    // 获取输入矩阵 x0 的列数
    const int nc = x0->ne[1];
    // 获取输入矩阵 x1 的行数
    const int nr = x1->ne[1];
    // 获取输入矩阵 x0 的行数
    const int nk = x0->ne[0];

    // 打印调试信息，输出输入矩阵 x0 和 x1 的维度信息
    GGML_PRINT_DEBUG("check_mat_mul: nc=%d, nr=%d, nk=%d\n", nc, nr, nk);

    // 打印调试信息，输出输入矩阵 x0 的数据
    GGML_PRINT_DEBUG("x0:\n");
    for (int j = 0; j < x0->ne[1]; ++j) {
        for (int i = 0; i < x0->ne[0]; ++i) {
            GGML_PRINT_DEBUG("%6.3f ", src0[j*nk + i]);
        }
        GGML_PRINT_DEBUG("\n");
    }
    GGML_PRINT_DEBUG("\n");

    // 打印调试信息，输出输入矩阵 x1 的数据
    GGML_PRINT_DEBUG("x1:\n");
    for (int j = 0; j < x1->ne[1]; ++j) {
        for (int i = 0; i < x1->ne[0]; ++i) {
            GGML_PRINT_DEBUG("%6.3f ", src1[j*nk + i]);
        }
        GGML_PRINT_DEBUG("\n");
    }
    GGML_PRINT_DEBUG("\n");

    // 打印调试信息，输出输出矩阵 y 的维度信息
    GGML_PRINT_DEBUG("y: n_dims = %d, (%lld, %lld)\n", y->n_dims, y->ne[0], y->ne[1]);
    for (int j = 0; j < y->ne[1]; ++j) {
        for (int i = 0; i < y->ne[0]; ++i) {
            GGML_PRINT_DEBUG("%6.3f ", dst[j*nr + i]);
        }
        GGML_PRINT_DEBUG("\n");
    }

    // 遍历计算矩阵相乘的结果，并与输出矩阵 y 进行比较
    for (int i = 0; i < nr; ++i) {
        for (int j = 0; j < nc; ++j) {
            float sum = 0.0f;

            for (int k = 0; k < nk; ++k) {
                sum += src0[j*nk + k]*src1[i*nk + k];
            }

            // 如果计算结果与输出矩阵 y 的对应元素不相等，则输出错误信息并终止程序
            if (fabsf(dst[i*nc + j] - sum) > 1e-5f) {
                fprintf(stderr, "check_mat_mul: dst[%d] = %f, sum = %f\n", i*nc + j, dst[i*nc + j], sum);
                assert(false);
                return false;
            }
        }
    }

    // 矩阵相乘结果正确，返回 true
    return true;
}

// 定义全排列的数量
#define NUM_PERMUTATIONS (4*3*2*1)

// 主函数
int main(int argc, const char ** argv) {
    // 初始化参数结构体
    struct ggml_init_params params = {
        /* .mem_size   = */ 256*1024*1024,  // 内存大小
        /* .mem_buffer = */ NULL,           // 内存缓冲区
        /* .no_alloc   = */ false,          // 是否允许分配内存
    };

    // 定义一个长度为 4 的整型数组
    int64_t ne[4];
    # 创建一个包含所有排列的数组，每个排列包含4个元素
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