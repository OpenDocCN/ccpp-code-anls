# `PowerInfer\examples\llava\clip.h`

```
#ifndef CLIP_H
#define CLIP_H

#include <stddef.h>
#include <stdint.h>

#ifdef LLAMA_SHARED
#    if defined(_WIN32) && !defined(__MINGW32__)
#        ifdef LLAMA_BUILD
#            define CLIP_API __declspec(dllexport)
#        else
#            define CLIP_API __declspec(dllimport)
#        endif
#    else
#        define CLIP_API __attribute__ ((visibility ("default")))
#    endif
#else
#    define CLIP_API
#endif

struct clip_ctx;

#ifdef __cplusplus
extern "C" {
#endif

struct clip_vision_hparams {
    int32_t image_size;  // 图像大小
    int32_t patch_size;  // 补丁大小
    int32_t hidden_size;  // 隐藏层大小
    int32_t n_intermediate;  // 中间层数量
    int32_t projection_dim;  // 投影维度
    int32_t n_head;  // 头数量
    int32_t n_layer;  // 层数量
    float eps;  // epsilon
};

/** load mmproj model */
CLIP_API struct clip_ctx * clip_model_load(const char * fname, const int verbosity);  // 加载 mmproj 模型
/** free mmproj model */
CLIP_API void clip_free(struct clip_ctx * ctx);  // 释放 mmproj 模型

size_t clip_embd_nbytes(const struct clip_ctx * ctx);  // 获取嵌入字节数
int clip_n_patches(const struct clip_ctx * ctx);  // 获取补丁数量
int clip_n_mmproj_embd(const struct clip_ctx * ctx);  // 获取 mmproj 嵌入数量

// RGB uint8 image
struct clip_image_u8 {
    int nx;  // x轴像素数量
    int ny;  // y轴像素数量
    uint8_t * data = NULL;  // 数据指针，默认为空
    size_t size;  // 大小
};

// RGB float32 image (NHWC)
// Memory layout: RGBRGBRGB...
struct clip_image_f32 {
    int nx;  // x轴像素数量
    int ny;  // y轴像素数量
    float * data = NULL;  // 数据指针，默认为空
    size_t size;  // 大小
};

struct clip_image_u8_batch {
    struct clip_image_u8 * data;  // 数据指针
    size_t size;  // 大小
};

struct clip_image_f32_batch {
    struct clip_image_f32 * data;  // 数据指针
    size_t size;  // 大小
};

struct clip_image_u8 * make_clip_image_u8();  // 创建 clip_image_u8 对象
struct clip_image_f32 * make_clip_image_f32();  // 创建 clip_image_f32 对象
CLIP_API void clip_image_u8_free(clip_image_u8 * img);  // 释放 clip_image_u8 对象
CLIP_API void clip_image_f32_free(clip_image_f32 * img);  // 释放 clip_image_f32 对象
CLIP_API bool clip_image_load_from_file(const char * fname, struct clip_image_u8 * img);  // 从文件加载图像数据到 clip_image_u8 对象
/** interpret bytes as an image file with length bytes_length, and use the result to populate img */
CLIP_API bool clip_image_load_from_bytes(const unsigned char * bytes, size_t bytes_length, struct clip_image_u8 * img);  // 从字节数据加载图像数据到 clip_image_u8 对象
// 对输入图像进行预处理，将其转换为浮点数格式，可选择是否填充成正方形
bool clip_image_preprocess(const struct clip_ctx * ctx, const struct clip_image_u8 * img, struct clip_image_f32 * res, const bool pad2square);

// 对输入图像进行编码，使用指定数量的线程，将图像转换为一维向量
bool clip_image_encode(const struct clip_ctx * ctx, const int n_threads, struct clip_image_f32 * img, float * vec);

// 对输入图像批量进行编码，使用指定数量的线程，将图像批量转换为一维向量
bool clip_image_batch_encode(const struct clip_ctx * ctx, const int n_threads, const struct clip_image_f32_batch * imgs, float * vec);

// 对模型进行量化，将输入模型文件名和输出模型文件名以及量化类型作为参数
bool clip_model_quantize(const char * fname_inp, const char * fname_out, const int itype);

#ifdef __cplusplus
}
#endif

#endif // CLIP_H
```