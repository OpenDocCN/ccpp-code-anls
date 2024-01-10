# `ggml\tests\test-conv-transpose.c`

```
#include "ggml/ggml.h"  // 引入 ggml 库的头文件

#include <string.h>  // 引入字符串处理函数的头文件
#include <stdio.h>   // 引入标准输入输出函数的头文件
#include <stdlib.h>  // 引入标准库函数的头文件

struct ggml_context* make_ctx(void) {  // 定义一个返回 ggml_context 指针的函数
    struct ggml_init_params params = {  // 定义 ggml_init_params 结构体变量 params
        .mem_size = 2 * 1024 * 1024,  // 初始化 params 结构体的 mem_size 成员
    };

    return ggml_init(params);  // 调用 ggml_init 函数并返回结果
}

void printf_tensor(struct ggml_tensor * t) {  // 定义一个打印 ggml_tensor 结构体的函数
    if (t->type == GGML_TYPE_F32) {  // 如果 ggml_tensor 的类型是 GGML_TYPE_F32
        const float * t_d = ggml_get_data_f32(t);  // 获取 ggml_tensor 的数据并赋值给 t_d
        for (int i = 0; i < t->ne[2]; ++i) {  // 遍历第三维
            for (int j = 0; j < t->ne[1]; ++j) {  // 遍历第二维
                for (int k = 0; k < t->ne[0]; ++k) {  // 遍历第一维
                    printf("%.1f ", t_d[i * t->ne[1] * t->ne[0] + j * t->ne[0] + k]);  // 打印数据
                }
                printf("\n");  // 换行
            }
            printf("---\n");  // 打印分隔线
        }
    }
    else if (t->type == GGML_TYPE_F16) {  // 如果 ggml_tensor 的类型是 GGML_TYPE_F16
        const ggml_fp16_t * t_d = ggml_get_data(t);  // 获取 ggml_tensor 的数据并赋值给 t_d
        for (int i = 0; i < t->ne[2]; ++i) {  // 遍历第三维
            for (int j = 0; j < t->ne[1]; ++j) {  // 遍历第二维
                for (int k = 0; k < t->ne[0]; ++k) {  // 遍历第一维
                    printf("%.1f ", ggml_fp16_to_fp32(t_d[i * t->ne[1] * t->ne[0] + j * t->ne[0] + k]));  // 打印数据
                }
                printf("\n");  // 换行
            }
            printf("---\n");  // 打印分隔线
        }
    }
    else {  // 如果 ggml_tensor 的类型不是 GGML_TYPE_F32 也不是 GGML_TYPE_F16
        printf("unknown type\n");  // 打印未知类型
    }
}

void check_tensor(struct ggml_tensor * t, float * expected_t_d, int ne0, int ne1, int ne2) {  // 定义一个检查 ggml_tensor 结构体的函数
    GGML_ASSERT(t->type == GGML_TYPE_F32);  // 断言 ggml_tensor 的类型是 GGML_TYPE_F32
    GGML_ASSERT(t->ne[0] == ne0);  // 断言 ggml_tensor 的第一维大小符合预期
    GGML_ASSERT(t->ne[1] == ne1);  // 断言 ggml_tensor 的第二维大小符合预期
    GGML_ASSERT(t->ne[2] == ne2);  // 断言 ggml_tensor 的第三维大小符合预期
    for (int i2 = 0; i2 < ne2; ++i2) {  // 遍历第三维
        for (int i1 = 0; i1 < ne1; ++i1) {  // 遍历第二维
            for (int i0 = 0; i0 < ne0; ++i0) {  // 遍历第一维
                float expected = *(expected_t_d + i2 * ne1 * ne0 + i1 * ne0 + i0);  // 获取预期值
                float actual = ggml_get_data_f32(t)[i2 * ne1 * ne0 + i1 * ne0 + i0];  // 获取实际值
                if (expected != actual) {  // 如果预期值不等于实际值
                    printf("expected %.1f, got %.1f\n", expected, actual);  // 打印预期值和实际值
                }
                GGML_ASSERT(expected == actual);  // 断言预期值等于实际值
            }
        }
    }
}
void test_conv_transpose_1d(void) {
    // 创建一个包含 1024 个元素的单精度浮点数数组
    float buf_f32[1024];
    // 循环填充数组，每个元素的值为其索引值
    for (int i = 0; i < 1024; ++i) {
        buf_f32[i] = (float)i;
    }

    // 创建一个包含 1024 个元素的半精度浮点数数组
    ggml_fp16_t buf_f16[1024];
    // 循环填充数组，每个元素的值为将对应的单精度浮点数转换为半精度浮点数后的值
    for (int i = 0; i < 1024; ++i) {
        buf_f16[i] = ggml_fp32_to_fp16((float)i);
    }

    // 创建期望输出的三个 3x4 的二维数组
    float expected_out_1[3][4] = {
        {18.0, 45.0, 59.0, 37.0},
        {24.0, 61.0, 83.0, 51.0},
        {30.0, 77.0, 107.0, 65.0},
    };
    float expected_out_2[3][6] = {
        {18.0, 21.0, 24.0, 29.0, 30.0, 37.0},
        {24.0, 27.0, 34.0, 39.0, 44.0, 51.0},
        {30.0, 33.0, 44.0, 49.0, 58.0, 65.0},
    };
    float expected_out_3[3][8] = {
        {18.0, 21.0, 0.0, 24.0, 29.0, 0.0, 30.0, 37.0},
        {24.0, 27.0, 0.0, 34.0, 39.0, 0.0, 44.0, 51.0},
        {30.0, 33.0, 0.0, 44.0, 49.0, 0.0, 58.0, 65.0},
    };

    // 执行一维卷积转置操作，包括步长为 1、2 和 3
    # 创建一个 ggml 上下文对象
    struct ggml_context * ctx = make_ctx();

    # 创建一个 3x2 的浮点型二维张量对象 t
    struct ggml_tensor * t = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, 3, 2); // l x cin
    # 将 buf_f32 中的数据拷贝到张量 t 中
    memcpy(t->data, buf_f32, ggml_nbytes(t));

    # 创建一个 2x3x2 的半精度浮点型三维张量对象 k
    struct ggml_tensor * k = ggml_new_tensor_3d(ctx, GGML_TYPE_F16, 2, 3, 2); // k x cout x cin
    # 将 buf_f16 中的数据拷贝到张量 k 中
    memcpy(k->data, buf_f16, ggml_nbytes(k));

    # 使用卷积转置操作计算得到 out_1 张量
    struct ggml_tensor * out_1 = ggml_conv_transpose_1d(ctx, k, t, 1 /* s0 */, 0 /* p0 */, 1 /* d0 */);
    # 使用卷积转置操作计算得到 out_2 张量
    struct ggml_tensor * out_2 = ggml_conv_transpose_1d(ctx, k, t, 2 /* s0 */, 0 /* p0 */, 1 /* d0 */);
    # 使用卷积转置操作计算得到 out_3 张量
    struct ggml_tensor * out_3 = ggml_conv_transpose_1d(ctx, k, t, 3 /* s0 */, 0 /* p0 */, 1 /* d0 */);

    # 创建一个新的计算图对象 gf_1
    struct ggml_cgraph * gf_1 = ggml_new_graph(ctx);
    # 创建一个新的计算图对象 gf_2
    struct ggml_cgraph * gf_2 = ggml_new_graph(ctx);
    # 创建一个新的计算图对象 gf_3
    struct ggml_cgraph * gf_3 = ggml_new_graph(ctx);

    # 构建计算图 gf_1，用于计算 out_1
    ggml_build_forward_expand(gf_1, out_1);
    # 构建计算图 gf_2，用于计算 out_2
    ggml_build_forward_expand(gf_2, out_2);
    # 构建计算图 gf_3，用于计算 out_3
    ggml_build_forward_expand(gf_3, out_3);

    # 使用上下文对象 ctx 计算计算图 gf_1
    ggml_graph_compute_with_ctx(ctx, gf_1, 1);
    # 使用上下文对象 ctx 计算计算图 gf_2
    ggml_graph_compute_with_ctx(ctx, gf_2, 1);
    # 使用上下文对象 ctx 计算计算图 gf_3
    ggml_graph_compute_with_ctx(ctx, gf_3, 1);

    # 检查 out_1 张量的值是否与预期值 (float*)expected_out_1 相符
    check_tensor(out_1, (float*)expected_out_1, 4, 3, 1);
    # 检查 out_2 张量的值是否与预期值 (float*)expected_out_2 相符
    check_tensor(out_2, (float*)expected_out_2, 6, 3, 1);
    # 检查 out_3 张量的值是否与预期值 (float*)expected_out_3 相符
    check_tensor(out_3, (float*)expected_out_3, 8, 3, 1);
// 定义测试函数 test_conv_transpose_2d
void test_conv_transpose_2d(void) {

    // 定义并初始化一个包含 1024 个元素的 float 类型数组 buf_f32
    float buf_f32[1024];
    for (int i = 0; i < 1024; ++i) {
        buf_f32[i] = (float)i;
    }

    // 定义并初始化一个包含 1024 个元素的 ggml_fp16_t 类型数组 buf_f16
    ggml_fp16_t buf_f16[1024];
    for (int i = 0; i < 1024; ++i) {
        // 将 float 类型数据转换为 ggml_fp16_t 类型，并存入数组 buf_f16
        buf_f16[i] = ggml_fp32_to_fp16((float)i);
    }

    // 定义一个包含 3 个 3x4 的 float 数组 expected_out_1，并初始化其值
    float expected_out_1[3][3][4] = {
        {
            {72.0, 162.0, 188.0, 106.0},
            {192.0, 430.0, 490.0, 274.0},
            {132.0, 292.0, 326.0, 180.0},
        },
        {
            {96.0, 218.0, 260.0, 146.0},
            {264.0, 590.0, 682.0, 378.0},
            {180.0, 396.0, 446.0, 244.0},
        },
        {
            {120.0, 274.0, 332.0, 186.0},
            {336.0, 750.0, 874.0, 482.0},
            {228.0, 500.0, 566.0, 308.0},
        },
    };

    // 定义一个包含 3 个 4x6 的 float 数组 expected_out_2，并初始化其值
    float expected_out_2[3][4][6] = {
        {
            {72.0, 78.0, 84.0, 92.0, 96.0, 106.0},
            {84.0, 90.0, 100.0, 108.0, 116.0, 126.0},
            {108.0, 120.0, 120.0, 134.0, 132.0, 148.0},
            {132.0, 144.0, 148.0, 162.0, 164.0, 180.0},
        },
        {
            {96.0, 102.0, 116.0, 124.0, 136.0, 146.0},
            {108.0, 114.0, 132.0, 140.0, 156.0, 166.0},
            {156.0, 168.0, 176.0, 190.0, 196.0, 212.0},
            {180.0, 192.0, 204.0, 218.0, 228.0, 244.0},
        },
        {
            {120.0, 126.0, 148.0, 156.0, 176.0, 186.0},
            {132.0, 138.0, 164.0, 172.0, 196.0, 206.0},
            {204.0, 216.0, 232.0, 246.0, 260.0, 276.0},
            {228.0, 240.0, 260.0, 274.0, 292.0, 308.0},
        },
    };
}
    // 定义一个三维浮点数组，包含3个5x8的二维数组
    float expected_out_3[3][5][8] = {
        // 第一个二维数组
        {
            // 第一行
            {72.0, 78.0, 0.0, 84.0, 92.0, 0.0, 96.0, 106.0},
            // 第二行
            {84.0, 90.0, 0.0, 100.0, 108.0, 0.0, 116.0, 126.0},
            // 第三行
            {0.0, 0.0, 0.0, 0.0, 0.0, 0.0},
            // 第四行
            {108.0, 120.0, 0.0, 120.0, 134.0, 0.0, 132.0, 148.0},
            // 第五行
            {132.0, 144.0, 0.0, 148.0, 162.0, 0.0, 164.0, 180.0},
        },
        // 第二个二维数组
        {
            // ... 以此类推
        },
        // 第三个二维数组
        {
            // ... 以此类推
        },
    };

    // 执行卷积转置操作，2维数组的步长分别为1、2和3
    # 创建一个 ggml 上下文对象
    struct ggml_context * ctx = make_ctx();

    # 创建一个 4 维的浮点型张量对象 t，大小为 3 x 2 x 2 x 1
    struct ggml_tensor * t = ggml_new_tensor_4d(ctx, GGML_TYPE_F32, 3, 2, 2, 1); // w x h x cin
    # 将 buf_f32 中的数据拷贝到张量 t 中
    memcpy(t->data, buf_f32, ggml_nbytes(t));

    # 创建一个 4 维的半精度浮点型张量对象 k，大小为 2 x 2 x 3 x 2
    struct ggml_tensor * k = ggml_new_tensor_4d(ctx, GGML_TYPE_F16, 2, 2, 3, 2); // w x h cin x cout
    # 将 buf_f16 中的数据拷贝到张量 k 中
    memcpy(k->data, buf_f16, ggml_nbytes(k));

    # 使用 ggml_conv_transpose_2d_p0 函数对张量 k 和 t 进行转置卷积操作，步长为 1
    struct ggml_tensor * out_1 = ggml_conv_transpose_2d_p0(ctx, k, t, 1);
    # 使用 ggml_conv_transpose_2d_p0 函数对张量 k 和 t 进行转置卷积操作，步长为 2
    struct ggml_tensor * out_2 = ggml_conv_transpose_2d_p0(ctx, k, t, 2);
    # 使用 ggml_conv_transpose_2d_p0 函数对张量 k 和 t 进行转置卷积操作，步长为 3
    struct ggml_tensor * out_3 = ggml_conv_transpose_2d_p0(ctx, k, t, 3);

    # 创建一个新的计算图对象 gf_1
    struct ggml_cgraph * gf_1 = ggml_new_graph(ctx);
    # 创建一个新的计算图对象 gf_2
    struct ggml_cgraph * gf_2 = ggml_new_graph(ctx);
    # 创建一个新的计算图对象 gf_3
    struct ggml_cgraph * gf_3 = ggml_new_graph(ctx);

    # 构建计算图 gf_1，用于计算 out_1
    ggml_build_forward_expand(gf_1, out_1);
    # 构建计算图 gf_2，用于计算 out_2
    ggml_build_forward_expand(gf_2, out_2);
    # 构建计算图 gf_3，用于计算 out_3
    ggml_build_forward_expand(gf_3, out_3);

    # 使用上下文对象 ctx 计算计算图 gf_1
    ggml_graph_compute_with_ctx(ctx, gf_1, 1);
    # 使用上下文对象 ctx 计算计算图 gf_2
    ggml_graph_compute_with_ctx(ctx, gf_2, 1);
    # 使用上下文对象 ctx 计算计算图 gf_3
    ggml_graph_compute_with_ctx(ctx, gf_3, 1);

    # 打印张量 t 的内容
    // printf("in\n");
    // printf_tensor(t);
    # 打印张量 k 的内容
    // printf("\n\nkernel\n");
    // printf_tensor(k);
    # 打印张量 out 的内容
    // printf("\n\nout\n");
    // printf_tensor(out);
    # 打印张量 out_2 的内容
    // printf("\n\nout_2\n");
    // printf_tensor(out_2);
    # 打印张量 out_3 的内容
    // printf("\n\nout_3\n");
    // printf_tensor(out_3);

    # 检查 out_1 的内容是否与 expected_out_1 中的数据一致
    check_tensor(out_1, (float*)expected_out_1, 4, 3, 3);
    # 检查 out_2 的内容是否与 expected_out_2 中的数据一致
    check_tensor(out_2, (float*)expected_out_2, 6, 4, 3);
    # 检查 out_3 的内容是否与 expected_out_3 中的数据一致
    check_tensor(out_3, (float*)expected_out_3, 8, 5, 3);
# 主函数入口，接受参数个数和参数列表
int main(int argc, const char * argv[]) {
    # 调用测试函数，测试一维转置卷积
    test_conv_transpose_1d();
    # 调用测试函数，测试二维转置卷积
    test_conv_transpose_2d();
    # 返回执行成功状态
    return 0;
}
```