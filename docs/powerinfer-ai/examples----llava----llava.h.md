# `PowerInfer\examples\llava\llava.h`

```
#ifndef LLAVA_H
#define LLAVA_H

#include "ggml.h"


#ifdef LLAMA_SHARED
#    if defined(_WIN32) && !defined(__MINGW32__)
#        ifdef LLAMA_BUILD
#            define LLAVA_API __declspec(dllexport)
#        else
#            define LLAVA_API __declspec(dllimport)
#        endif
#    else
#        define LLAVA_API __attribute__ ((visibility ("default")))
#    endif
#else
#    define LLAVA_API
#endif

struct clip_ctx;

#ifdef __cplusplus
extern "C" {
#endif

struct llava_image_embed {
    float * embed;  // 指向浮点数的指针，用于存储图像嵌入数据
    int n_image_pos;  // 图像位置
};

/** sanity check for clip <-> llava embed size match */
LLAVA_API bool llava_validate_embed_size(const llama_context * ctx_llama, const clip_ctx * ctx_clip);
// 检查 clip 和 llava 嵌入大小是否匹配

/** build an image embed from image file bytes */
LLAVA_API struct llava_image_embed * llava_image_embed_make_with_bytes(struct clip_ctx * ctx_clip, int n_threads, const unsigned char * image_bytes, int image_bytes_length);
// 从图像文件字节构建图像嵌入

/** build an image embed from a path to an image filename */
LLAVA_API struct llava_image_embed * llava_image_embed_make_with_filename(struct clip_ctx * ctx_clip, int n_threads, const char * image_path);
// 从图像文件路径构建图像嵌入

LLAVA_API void llava_image_embed_free(struct llava_image_embed * embed);
/** free an embedding made with llava_image_embed_make_* */
// 释放使用 llava_image_embed_make_* 创建的嵌入

/** write the image represented by embed into the llama context with batch size n_batch, starting at context pos n_past. on completion, n_past points to the next position in the context after the image embed. */
LLAVA_API bool llava_eval_image_embed(struct llama_context * ctx_llama, const struct llava_image_embed * embed, int n_batch, int * n_past);
// 将由 embed 表示的图像写入 llama 上下文，批量大小为 n_batch，从上下文位置 n_past 开始。完成后，n_past 指向图像嵌入后上下文中的下一个位置。

#ifdef __cplusplus
}
#endif

#endif
```