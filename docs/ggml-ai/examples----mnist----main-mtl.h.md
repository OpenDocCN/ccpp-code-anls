# `ggml\examples\mnist\main-mtl.h`

```cpp
#pragma once
// 防止头文件被重复包含

struct ggml_context;
struct ggml_cgraph;
// 声明结构体 ggml_context 和 ggml_cgraph

#ifdef __cplusplus
extern "C" {
#endif
// 如果是 C++ 环境，则按照 C 语言的方式进行编译

struct ggml_mtl_context;
// 声明结构体 ggml_mtl_context

struct ggml_mtl_context * mnist_mtl_init(
        struct ggml_context * ctx_data,
        struct ggml_context * ctx_eval,
        struct ggml_context * ctx_work,
        struct ggml_cgraph  * gf);
// 定义 mnist_mtl_init 函数，接受四个参数，返回 ggml_mtl_context 指针

void mnist_mtl_free(struct ggml_mtl_context * ctx);
// 定义 mnist_mtl_free 函数，接受 ggml_mtl_context 指针参数，无返回值

int mnist_mtl_eval(
        struct ggml_mtl_context * ctx,
        struct ggml_cgraph      * gf);
// 定义 mnist_mtl_eval 函数，接受 ggml_mtl_context 和 ggml_cgraph 指针参数，返回整型值

#ifdef __cplusplus
}
#endif
// 如果是 C++ 环境，则结束按照 C 语言的方式进行编译
```