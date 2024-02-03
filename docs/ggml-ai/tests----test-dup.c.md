# `ggml\tests\test-dup.c`

```cpp
#include "ggml/ggml.h"

#include <stdio.h>
#include <stdlib.h>

void arange(struct ggml_tensor* tensor) {
    // 确保张量是连续的
    GGML_ASSERT(ggml_is_contiguous(tensor));
    // 遍历张量的每个元素，将索引赋值给元素
    for (int i = 0; i < ggml_nelements(tensor); ++i) {
        ggml_set_i32_1d(tensor, i, i);
    }
}

void dup_to(struct ggml_tensor* src, struct ggml_tensor* dst) {
    // 确保目标张量是视图
    GGML_ASSERT(dst->op == GGML_OP_VIEW);
    // 确保源张量和目标张量的元素个数相同
    GGML_ASSERT(ggml_nelements(src) == ggml_nelements(dst));
    // 将目标张量的操作类型设置为复制
    dst->op = GGML_OP_DUP;
    // 设置目标张量的源张量为给定的源张量
    dst->src[0] = src;
}

bool can_dup(enum ggml_type src_type, enum ggml_type dst_type) {
    // 如果源类型和目标类型相同，则可以复制
    if (src_type == dst_type) return true;
    // 如果源类型是 GGML_TYPE_F32 并且目标类型可以从浮点数转换，则可以复制
    if (src_type == GGML_TYPE_F32 && ggml_internal_get_type_traits(dst_type).from_float) return true;
    // 如果目标类型是 GGML_TYPE_F32 并且源类型可以转换为浮点数，则可以复制
    if (dst_type == GGML_TYPE_F32 && ggml_internal_get_type_traits(src_type).to_float) return true;

    return false;
}

int main(int argc, const char ** argv) {
    // 初始化参数结构体
    struct ggml_init_params params = {
        .mem_size   = 128*1024*1024,
        .mem_buffer = NULL,
        .no_alloc   = false,
    };

    // 定义一个包含四种类型的数组
    enum ggml_type type[4] = {GGML_TYPE_I16, GGML_TYPE_I32, GGML_TYPE_F16, GGML_TYPE_F32};
    }

    return 0;
}
```