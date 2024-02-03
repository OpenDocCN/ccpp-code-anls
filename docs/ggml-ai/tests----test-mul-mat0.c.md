# `ggml\tests\test-mul-mat0.c`

```cpp
#define _CRT_SECURE_NO_DEPRECATE // Disables ridiculous "unsafe" warnigns on Windows
#include "ggml/ggml.h"

#include <math.h>
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>
#include <inttypes.h>

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

#define MAX_NARGS 2

// 生成一个 0 到 1 之间的随机浮点数
float frand(void) {
    return (float)rand()/(float)RAND_MAX;
}

// 生成一个 0 到 n-1 之间的随机整数
int irand(int n) {
    return rand()%n;
}

// 生成随机维度
void get_random_dims(int64_t * dims, int ndims) {
    // 初始化维度数组
    dims[0] = dims[1] = dims[2] = dims[3] = 1;

    // 生成随机维度
    for (int i = 0; i < ndims; i++) {
        dims[i] = 1 + irand(4);
    }
}

// 生成一个随机张量
struct ggml_tensor * get_random_tensor(
        struct ggml_context * ctx0,
        int ndims,
        int64_t ne[],
        float fmin,
        float fmax) {
    // 创建一个新的张量对象
    struct ggml_tensor * result = ggml_new_tensor(ctx0, GGML_TYPE_F32, ndims, ne);
    # 根据维度数量进行不同的处理
    switch (ndims) {
        # 当维度为1时
        case 1:
            # 遍历第一维度，生成随机数并存入结果数组
            for (int i0 = 0; i0 < ne[0]; i0++) {
                ((float *)result->data)[i0] = frand()*(fmax - fmin) + fmin;
            }
            break;
        # 当维度为2时
        case 2:
            # 遍历第二维度和第一维度，生成随机数并存入结果数组
            for (int i1 = 0; i1 < ne[1]; i1++) {
                for (int i0 = 0; i0 < ne[0]; i0++) {
                    ((float *)result->data)[i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
                }
            }
            break;
        # 当维度为3时
        case 3:
            # 遍历第三维度、第二维度和第一维度，生成随机数并存入结果数组
            for (int i2 = 0; i2 < ne[2]; i2++) {
                for (int i1 = 0; i1 < ne[1]; i1++) {
                    for (int i0 = 0; i0 < ne[0]; i0++) {
                        ((float *)result->data)[i2*ne[1]*ne[0] + i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
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
                            ((float *)result->data)[i3*ne[2]*ne[1]*ne[0] + i2*ne[1]*ne[0] + i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
                        }
                    }
                }
            }
            break;
        # 默认情况下
        default:
            # 断言为假，即维度数量不符合预期
            assert(false);
    };
    # 返回结果数组
    return result;
# 获取张量中指定索引位置的元素值
float get_element(const struct ggml_tensor * t, int idx) {
    return ((float *)t->data)[idx];
}

# 设置张量中指定索引位置的元素值
void set_element(struct ggml_tensor * t, int idx, float value) {
    ((float *)t->data)[idx] = value;
}

# 检查梯度
bool check_gradient(
        const char * op_name,  # 操作名称
        struct ggml_context * ctx0,  # 上下文
        struct ggml_tensor * x[],  # 输入张量数组
        struct ggml_tensor * f,  # 输出张量
        int ndims,  # 维度
        int nargs,  # 参数数量
        float eps,  # 误差范围
        float max_error_abs,  # 最大绝对误差
        float max_error_rel) {  # 最大相对误差
    const int n_threads = 1;  # 线程数

    # 创建新的计算图
    struct ggml_cgraph * gf = ggml_new_graph_custom(ctx0, GGML_DEFAULT_GRAPH_SIZE, true);
    # 构建前向传播计算图
    ggml_build_forward_expand(gf, f);
    # 复制前向传播计算图
    struct ggml_cgraph * gb = ggml_graph_dup(ctx0, gf);
    # 构建反向传播计算图
    ggml_build_backward_expand(ctx0, gf, gb, false);

    # 使用上下文计算前向传播计算图
    ggml_graph_compute_with_ctx(ctx0, gf, n_threads);
    # 重置计算图
    ggml_graph_reset(gf);
    # 设置输出张量的梯度为1.0
    ggml_set_f32(f->grad, 1.0f);
    # 使用上下文计算反向传播计算图
    ggml_graph_compute_with_ctx(ctx0, gb, n_threads);

    # 将前向传播计算图导出为.dot文件
    ggml_graph_dump_dot(gf, NULL, "test-grad0-forward.dot");
    # 将反向传播计算图导出为.dot文件
    ggml_graph_dump_dot(gb, gf, "test-grad0-backward.dot");
    # 遍历输入参数列表
    for (int i = 0; i < nargs; ++i) {
        # 获取当前参数的元素个数
        const int64_t nelements = ggml_nelements(x[i]);
        # 遍历当前参数的每个元素
        for (int64_t k = 0; k < nelements; ++k) {
            # 使用有限差分法计算梯度
            const float x0 = get_element(x[i], k);

            # 设置当前元素的值为 x0 + eps
            set_element(x[i], k, x0 + eps);
            # 使用上下文 ctx0 和计算图 gf 进行计算
            ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

            # 获取计算结果 f0
            const float f0 = ggml_get_f32_1d(f, 0);

            # 设置当前元素的值为 x0 - eps
            set_element(x[i], k, x0 - eps);
            # 使用上下文 ctx0 和计算图 gf 进行计算
            ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

            # 获取计算结果 f1
            const float f1 = ggml_get_f32_1d(f, 0);

            # 计算梯度 g0
            const float g0 = (f0 - f1)/(2.0f*eps);

            # 恢复当前元素的原始值
            set_element(x[i], k, x0);

            # 使用反向计算图计算梯度
            ggml_graph_reset  (gf);
            ggml_set_f32      (f->grad, 1.0f);
            ggml_graph_compute_with_ctx(ctx0, gb, n_threads);

            # 获取计算得到的梯度 g1
            const float g1 = get_element(x[i]->grad, k);

            # 计算绝对误差和相对误差
            const float error_abs = fabsf(g0 - g1);
            const float error_rel = g0 != 0 ? fabsf(g0 - g1)/fabs(g0) : 0;

            # 如果误差超过阈值，则输出错误信息并断言
            if (error_abs > max_error_abs || error_rel > max_error_rel) {
                printf("%s: ndims=%d, i=%d, k=%" PRId64 ", g0=%f, g1=%f, error_abs=%f, error_rel=%f\n", op_name, ndims, i, k, g0, g1, error_abs, error_rel);
                assert(false);
            }
        }
    }

    # 返回 true
    return true;
    # 从四维张量中获取指定位置的浮点数值
    float mat_get(const struct ggml_tensor * t, int i0, int i1, int i2, int i3) {
        # 获取张量在各维度上的大小
        const size_t nb0 = t->nb[0];
        const size_t nb1 = t->nb[1];
        const size_t nb2 = t->nb[2];
        const size_t nb3 = t->nb[3];

        # 计算指定位置的浮点数值的内存地址，并返回其值
        return
            *((float*) ((char*)t->data + i0*nb0 + i1*nb1 + i2*nb2 + i3*nb3));
    }

    # 检查矩阵相乘是否合法
    bool check_mat_mul(
            const struct ggml_tensor * y,
            const struct ggml_tensor * x0,
            const struct ggml_tensor * x1) {
        # 获取输入张量 x0 在各维度上的大小
        const int64_t n00 = x0->ne[0];
        const int64_t n10 = x0->ne[1];
        const int64_t n20 = x0->ne[2];
        const int64_t n30 = x0->ne[3];

        # 获取输入张量 x1 在各维度上的大小
        const int64_t n01 = x1->ne[0];
        const int64_t n11 = x1->ne[1];
        const int64_t n21 = x1->ne[2];
        const int64_t n31 = x1->ne[3];

        # 获取输出张量 y 在各维度上的大小
        const int64_t n02 = y->ne[0];
        const int64_t n12 = y->ne[1];
        const int64_t n22 = y->ne[2];
        const int64_t n32 = y->ne[3];

        # 打印输入张量 x0 在各维度上的大小
        printf("x0: [%" PRId64 ", %" PRId64 ", %" PRId64 ", %" PRId64 "]\n", n00, n10, n20, n30);
        # 遍历并打印输入张量 x0 的数据
        for (int j = 0; j < n10; ++j) {
            for (int i = 0; i < n00; ++i) {
                printf("%6.3f ", mat_get(x0, i, j, 0, 0));
            }
            printf("\n");
        }
        printf("\n");

        # 打印输入张量 x1 在各维度上的大小
        printf("x1: [%" PRId64 ", %" PRId64 ", %" PRId64 ", %" PRId64 "]\n", n01, n11, n21, n31);
        # 遍历并打印输入张量 x1 的数据
        for (int j = 0; j < n11; ++j) {
            for (int i = 0; i < n01; ++i) {
                printf("%6.3f ", mat_get(x1, i, j, 0, 0));
            }
            printf("\n");
        }
        printf("\n");

        # 打印输出张量 y 在各维度上的大小
        printf("y: [%" PRId64 ", %" PRId64 ", %" PRId64 ", %" PRId64 "]\n", n02, n12, n22, n32);
        # 遍历并打印输出张量 y 的数据
        for (int j = 0; j < n12; ++j) {
            for (int i = 0; i < n02; ++i) {
                printf("%6.3f ", mat_get(y, i, j, 0, 0));
            }
            printf("\n");
        }
    # 遍历四层嵌套的循环，分别对应四维矩阵的索引
    for (int i3 = 0; i3 < n32; ++i3) {
        for (int i2 = 0; i2 < n22; ++i2) {
            for (int i1 = 0; i1 < n12; ++i1) {
                for (int i0 = 0; i0 < n02; ++i0) {
                    # 初始化求和变量
                    float sum = 0.0f;
                    # 遍历第一个矩阵的第一维，计算乘积并累加到sum中
                    for (int k = 0; k < n00; ++k) {
                        sum += mat_get(x0, k, i0, i2, i3) * mat_get(x1, k, i1, i2, i3);
                    }
                    # 判断计算得到的sum与目标矩阵y中的值是否相差超过1e-5
                    if (fabsf(sum - mat_get(y, i0, i1, i2, i3)) > 1e-5) {
                        # 如果相差超过1e-5，输出错误信息并终止程序
                        printf("error: i0=%d, i1=%d, i2=%d, i3=%d, sum=%f, y=%f\n",
                                i0, i1, i2, i3, sum, mat_get(y, i0, i1, i2, i3));
                        assert(false);
                        return false;
                    }
                }
            }
        }
    }
    # 如果所有计算结果与目标矩阵y中的值都相差不超过1e-5，返回true
    return true;
}

int main(int argc, const char ** argv) {
    // 初始化参数结构体，设置内存大小为128MB，内存缓冲区为空，允许分配内存
    struct ggml_init_params params = {
        .mem_size   = 128*1024*1024,
        .mem_buffer = NULL,
        .no_alloc   = false,
    };

    // 定义一个包含4个int64_t类型元素的数组
    int64_t ne[4];

    // 设置循环次数的初始值为500
    // 检查环境变量中是否有GGML_NLOOP，如果有则将其转换为整数并赋值给niter
    const char *env = getenv("GGML_NLOOP");
    if (env != NULL) {
        niter = atoi(env);
    }
    // 如果命令行参数大于1，则将第一个参数转换为整数并赋值给niter
    if (argc > 1) {
        niter = atoi(argv[1]);
    }

    // 设置线程数为1

    }

    // 返回0表示程序正常结束
    return 0;
}
```