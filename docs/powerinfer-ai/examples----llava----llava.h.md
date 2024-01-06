# `PowerInfer\examples\llava\llava.h`

```
#ifndef LLAVA_H
// 如果 LLAVA_H 未定义，则定义 LLAVA_H

#define LLAVA_API
// 定义 LLAVA_API 为空

#include "ggml.h"
// 包含 ggml.h 文件

#ifdef LLAMA_SHARED
// 如果 LLAMA_SHARED 已定义

#    if defined(_WIN32) && !defined(__MINGW32__)
// 如果操作系统为 Windows，并且不是使用 MinGW 编译器

#        ifdef LLAMA_BUILD
// 如果 LLAMA_BUILD 已定义

#            define LLAVA_API __declspec(dllexport)
// 定义 LLAVA_API 为 __declspec(dllexport)

#        else
// 否则

#            define LLAVA_API __declspec(dllimport)
// 定义 LLAVA_API 为 __declspec(dllimport)

#        endif
#    else
// 如果不是以上情况

#        define LLAVA_API __attribute__ ((visibility ("default")))
// 定义 LLAVA_API 为 __attribute__ ((visibility ("default")))

#    endif
#else
// 如果 LLAMA_SHARED 未定义

#    define LLAVA_API
// 定义 LLAVA_API 为空

#endif
// 定义一个名为 clip_ctx 的结构体

#ifdef __cplusplus
extern "C" {
#endif

// 定义一个名为 llava_image_embed 的结构体，包含指向浮点数的指针和整数类型的 n_image_pos

/** 对比 clip 和 llava 的嵌入大小是否匹配的检查 */
LLAVA_API bool llava_validate_embed_size(const llama_context * ctx_llama, const clip_ctx * ctx_clip);

/** 从图像文件字节构建图像嵌入 */
LLAVA_API struct llava_image_embed * llava_image_embed_make_with_bytes(struct clip_ctx * ctx_clip, int n_threads, const unsigned char * image_bytes, int image_bytes_length);
/** 从图像文件路径构建图像嵌入 */
LLAVA_API struct llava_image_embed * llava_image_embed_make_with_filename(struct clip_ctx * ctx_clip, int n_threads, const char * image_path);
LLAVA_API void llava_image_embed_free(struct llava_image_embed * embed);
/** 释放使用 llava_image_embed_make_* 创建的嵌入 */
# 如果是 C++ 环境，结束 extern "C" 块
#ifdef __cplusplus
}
#endif
# 结束头文件的条件编译
#endif
```