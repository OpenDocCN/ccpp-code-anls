# `ggml\tests\test-rel-pos.c`

```cpp
#include "ggml/ggml.h"  // 引入 ggml 库的头文件

#include <string.h>  // 引入字符串处理函数的头文件
#include <stdio.h>  // 引入标准输入输出函数的头文件
#include <stdlib.h>  // 引入标准库函数的头文件

struct ggml_context* make_ctx(void) {  // 定义一个返回 ggml_context 指针的函数 make_ctx
    struct ggml_init_params params = {  // 定义 ggml_init_params 结构体变量 params
        .mem_size = 2 * 1024 * 1024,  // 初始化 params 结构体的 mem_size 成员
    };

    return ggml_init(params);  // 调用 ggml_init 函数并返回结果
}

void check_tensor(struct ggml_tensor * t, float * expected_t_d, int ne0, int ne1, int ne2) {  // 定义一个检查张量的函数 check_tensor
    GGML_ASSERT(t->type == GGML_TYPE_F32);  // 断言张量的类型为 GGML_TYPE_F32
    GGML_ASSERT(t->ne[0] == ne0);  // 断言张量的第一个维度大小为 ne0
    GGML_ASSERT(t->ne[1] == ne1);  // 断言张量的第二个维度大小为 ne1
    GGML_ASSERT(t->ne[2] == ne2);  // 断言张量的第三个维度大小为 ne2
    for (int i2 = 0; i2 < ne2; ++i2) {  // 循环遍历张量的第三个维度
        for (int i1 = 0; i1 < ne1; ++i1) {  // 循环遍历张量的第二个维度
            for (int i0 = 0; i0 < ne0; ++i0) {  // 循环遍历张量的第一个维度
                float expected = *(expected_t_d + i2 * ne1 * ne0 + i1 * ne0 + i0);  // 计算期望值
                float actual = ggml_get_data_f32(t)[i2 * ne1 * ne0 + i1 * ne0 + i0];  // 获取实际值
                GGML_ASSERT(expected == actual);  // 断言期望值等于实际值
            }
        }
    }
}

int main(int argc, const char** argv) {  // 定义主函数
    ggml_fp16_t buf_f16[1024];  // 声明一个长度为 1024 的 ggml_fp16_t 数组
    for (int i = 0; i < 1024; ++i) {  // 循环遍历数组
        buf_f16[i] = ggml_fp32_to_fp16((float)i);  // 将 float 类型转换为 ggml_fp16_t 类型并赋值给数组元素
    }

    float expected_out[4][9] = {  // 声明一个二维数组 expected_out
        { 8.0, 9.0, 10.0, 9.0, 10.0, 11.0, 10.0, 11.0, 12.0 },  // 初始化第一行
        { 2.0, 3.0, 4.0, 3.0, 4.0, 5.0, 4.0, 5.0, 6.0 },  // 初始化第二行
        { 14.0, 15.0, 16.0, 15.0, 16.0, 17.0, 16.0, 17.0, 18.0 },  // 初始化第三行
        { 8.0, 9.0, 10.0, 9.0, 10.0, 11.0, 10.0, 11.0, 12.0 },  // 初始化第四行
    };
    # 创建一个 ggml_context 结构体指针，并初始化
    struct ggml_context * ctx = make_ctx();

    # 创建一个 3x3 的 GGML_TYPE_F16 类型的二维张量 t
    struct ggml_tensor * t = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, 3, 3);
    # 将 buf_f16 的数据拷贝到 t 的数据中
    ggml_fp16_t* t_d = (ggml_fp16_t*)t->data;
    memcpy(t_d, buf_f16, ggml_nbytes(t));

    # 创建另一个 3x3 的 GGML_TYPE_F16 类型的二维张量 t_2
    struct ggml_tensor * t_2 = ggml_new_tensor_2d(ctx, GGML_TYPE_F16, 3, 3);
    # 将 buf_f16 + 1 的数据拷贝到 t_2 的数据中
    ggml_fp16_t* t_d_2 = (ggml_fp16_t*)t_2->data;
    memcpy(t_d_2, buf_f16 + 1, ggml_nbytes(t_2));

    # 获取 t 的相对位置为 (2, 2) 的张量 rw
    struct ggml_tensor * rw = ggml_get_rel_pos(ctx, t, 2, 2);
    # 获取 t_2 的相对位置为 (2, 2) 的张量 rh
    struct ggml_tensor * rh = ggml_get_rel_pos(ctx, t_2, 2, 2);

    # 创建一个 3x2x2 的 GGML_TYPE_F32 类型的三维张量 rw_f32，并将 rw 的数据拷贝到 rw_f32
    struct ggml_tensor * rw_f32 = ggml_cpy(ctx, rw, ggml_new_tensor_3d(ctx, GGML_TYPE_F32, 3, 2, 2));
    # 创建一个 3x2x2 的 GGML_TYPE_F32 类型的三维张量 rh_f32，并将 rh 的数据拷贝到 rh_f32
    struct ggml_tensor * rh_f32 = ggml_cpy(ctx, rh, ggml_new_tensor_3d(ctx, GGML_TYPE_F32, 3, 2, 2));

    # 创建一个 9x4 的 GGML_TYPE_F32 类型的二维张量 in，并初始化数据为 1.0
    struct ggml_tensor * in = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, 9, 4);
    # 创建一个 9x4 的 GGML_TYPE_F32 类型的二维张量 out_inplace，并初始化数据为 1.0
    struct ggml_tensor * out_inplace = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, 9, 4);
    # 获取 in 和 out_inplace 的数据指针，并初始化数据为 1.0
    float * in_d          = (float*)in->data;
    float * out_inplace_d = (float*)out_inplace->data;
    for (int i = 0; i < ggml_nelements(in); ++i) {
        in_d[i]          = 1.f;
        out_inplace_d[i] = 1.f;
    }

    # 创建一个与 in、rw_f32、rh_f32 相加后的结果张量 out
    struct ggml_tensor * out = ggml_add_rel_pos(ctx, in, rw_f32, rh_f32);
    # 创建一个新的计算图 gf，并将 out 作为输入构建前向传播
    struct ggml_cgraph * gf = ggml_new_graph(ctx);
    ggml_build_forward_expand(gf, out);
    # 使用 ctx 计算 gf 的前向传播
    ggml_graph_compute_with_ctx(ctx, gf, 1);

    # 将 out_inplace 与 rw_f32、rh_f32 相加，并将结果存回 out_inplace
    out_inplace = ggml_add_rel_pos_inplace(ctx, out_inplace, rw_f32, rh_f32);
    # 创建一个新的计算图 gf_2，并将 out_inplace 作为输入构建前向传播
    struct ggml_cgraph * gf_2 = ggml_new_graph(ctx);
    ggml_build_forward_expand(gf_2, out_inplace);
    # 使用 ctx 计算 gf_2 的前向传播
    ggml_graph_compute_with_ctx(ctx, gf_2, 1);

    # 检查张量 out 的数据是否与期望的数据一致
    check_tensor(out, (float*)expected_out, 9, 4, 1);
    # 检查张量 out_inplace 的数据是否与期望的数据一致
    check_tensor(out_inplace, (float*)expected_out, 9, 4, 1);
    
    # 返回 0，表示程序执行成功
    return 0;
# 闭合前面的函数定义
```