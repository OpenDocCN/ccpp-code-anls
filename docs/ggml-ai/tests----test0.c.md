# `ggml\tests\test0.c`

```cpp
#include "ggml/ggml.h"

#include <stdio.h>
#include <stdlib.h>

int main(int argc, const char ** argv) {
    // 初始化参数结构体，设置内存大小为128MB，内存缓冲区为空，允许分配内存
    struct ggml_init_params params = {
        .mem_size   = 128*1024*1024,
        .mem_buffer = NULL,
        .no_alloc   = false,
    };

    // 初始化 GGML 上下文
    struct ggml_context * ctx0 = ggml_init(params);

    // 创建一个一维的浮点型张量，大小为10
    struct ggml_tensor * t1 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 10);
    // 创建一个二维的16位整型张量，大小为10x20
    struct ggml_tensor * t2 = ggml_new_tensor_2d(ctx0, GGML_TYPE_I16, 10, 20);
    // 创建一个三维的32位整型张量，大小为10x20x30
    struct ggml_tensor * t3 = ggml_new_tensor_3d(ctx0, GGML_TYPE_I32, 10, 20, 30);

    // 断言张量 t1 的维度为1
    GGML_ASSERT(ggml_n_dims(t1) == 1);
    // 断言张量 t1 的第一个维度大小为10
    GGML_ASSERT(t1->ne[0]  == 10);
    // 断言张量 t1 的内存大小为10个浮点数大小
    GGML_ASSERT(t1->nb[1]  == 10*sizeof(float));

    // 断言张量 t2 的维度为2
    GGML_ASSERT(ggml_n_dims(t2) == 2);
    // 断言张量 t2 的第一个维度大小为10
    GGML_ASSERT(t2->ne[0]  == 10);
    // 断言张量 t2 的第二个维度大小为20
    GGML_ASSERT(t2->ne[1]  == 20);
    // 断言张量 t2 的内存大小为10个16位整数大小和10x20个16位整数大小
    GGML_ASSERT(t2->nb[1]  == 10*sizeof(int16_t));
    GGML_ASSERT(t2->nb[2]  == 10*20*sizeof(int16_t));

    // 断言张量 t3 的维度为3
    GGML_ASSERT(ggml_n_dims(t3) == 3);
    // 断言张量 t3 的第一个维度大小为10
    GGML_ASSERT(t3->ne[0]  == 10);
    // 断言张量 t3 的第二个维度大小为20
    GGML_ASSERT(t3->ne[1]  == 20);
    // 断言张量 t3 的第三个维度大小为30
    GGML_ASSERT(t3->ne[2]  == 30);
    // 断言张量 t3 的内存大小为10个32位整数大小、10x20个32位整数大小和10x20x30个32位整数大小
    GGML_ASSERT(t3->nb[1]  == 10*sizeof(int32_t));
    GGML_ASSERT(t3->nb[2]  == 10*20*sizeof(int32_t));
    GGML_ASSERT(t3->nb[3]  == 10*20*30*sizeof(int32_t));

    // 打印 GGML 上下文中的对象信息
    ggml_print_objects(ctx0);

    // 释放 GGML 上下文
    ggml_free(ctx0);

    return 0;
}
```