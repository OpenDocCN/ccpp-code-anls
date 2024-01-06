# `PowerInfer\examples\llava\clip.h`

```
#ifndef CLIP_H
#define CLIP_H
// 如果 CLIP_H 未定义，则定义 CLIP_H

#include <stddef.h>
#include <stdint.h>
// 包含标准库头文件 <stddef.h> 和 <stdint.h>

#ifdef LLAMA_SHARED
// 如果 LLAMA_SHARED 已定义
#    if defined(_WIN32) && !defined(__MINGW32__)
// 如果操作系统为 Windows，并且不是使用 MinGW 编译器
#        ifdef LLAMA_BUILD
// 如果 LLAMA_BUILD 已定义
#            define CLIP_API __declspec(dllexport)
// 定义 CLIP_API 为导出符号
#        else
// 否则
#            define CLIP_API __declspec(dllimport)
// 定义 CLIP_API 为导入符号
#        endif
#    else
// 否则
#        define CLIP_API __attribute__ ((visibility ("default")))
// 定义 CLIP_API 为默认可见性属性
#    endif
#else
// 如果 LLAMA_SHARED 未定义
#    define CLIP_API
// 定义 CLIP_API 为空
#endif
// 定义 clip_ctx 结构体

#ifdef __cplusplus
extern "C" {
#endif

// 定义 clip_vision_hparams 结构体，包含模型的超参数

/** 加载 mmproj 模型 */
CLIP_API struct clip_ctx * clip_model_load(const char * fname, const int verbosity);
/** 释放 mmproj 模型 */
// 释放 clip_ctx 结构体所占用的内存
CLIP_API void clip_free(struct clip_ctx * ctx);

// 返回嵌入的字节数
size_t clip_embd_nbytes(const struct clip_ctx * ctx);

// 返回补丁的数量
int clip_n_patches(const struct clip_ctx * ctx);

// 返回多模态投影的嵌入数量
int clip_n_mmproj_embd(const struct clip_ctx * ctx);

// 表示 RGB uint8 图像的结构体
struct clip_image_u8 {
    int nx; // 图像的宽度
    int ny; // 图像的高度
    uint8_t * data = NULL; // 图像数据的指针
    size_t size; // 图像数据的大小
};

// 表示 RGB float32 图像的结构体（NHWC格式）
// 内存布局：RGBRGBRGB...
struct clip_image_f32 {
    int nx; // 图像的宽度
    int ny; // 图像的高度
    float * data = NULL; // 图像数据的指针
};
// 定义了一个结构体，用于表示一个无符号8位整数类型的图像
struct clip_image_u8 {
    unsigned char * data; // 图像数据
    size_t width; // 图像宽度
    size_t height; // 图像高度
    size_t channels; // 图像通道数
};

// 定义了一个结构体，用于表示一批无符号8位整数类型的图像
struct clip_image_u8_batch {
    struct clip_image_u8 * data; // 图像数据数组
    size_t size; // 图像数量
};

// 定义了一个结构体，用于表示一个32位浮点数类型的图像
struct clip_image_f32 {
    float * data; // 图像数据
    size_t width; // 图像宽度
    size_t height; // 图像高度
    size_t channels; // 图像通道数
};

// 定义了一个结构体，用于表示一批32位浮点数类型的图像
struct clip_image_f32_batch {
    struct clip_image_f32 * data; // 图像数据数组
    size_t size; // 图像数量
};

// 申明了一个函数，用于创建一个无符号8位整数类型的图像
struct clip_image_u8 * make_clip_image_u8();

// 申明了一个函数，用于创建一个32位浮点数类型的图像
struct clip_image_f32 * make_clip_image_f32();

// 申明了一个函数，用于释放无符号8位整数类型的图像内存
CLIP_API void clip_image_u8_free(clip_image_u8 * img);

// 申明了一个函数，用于释放32位浮点数类型的图像内存
CLIP_API void clip_image_f32_free(clip_image_f32 * img);

// 申明了一个函数，用于从文件加载图像数据到无符号8位整数类型的图像
CLIP_API bool clip_image_load_from_file(const char * fname, struct clip_image_u8 * img);

// 申明了一个函数，用于从字节流加载图像数据到无符号8位整数类型的图像
// 参数 bytes: 字节流数据
// 参数 bytes_length: 字节流长度
// 参数 img: 无符号8位整数类型的图像
CLIP_API bool clip_image_load_from_bytes(const unsigned char * bytes, size_t bytes_length, struct clip_image_u8 * img);
// 对输入的图像进行预处理，将其转换为浮点数格式，如果需要则进行填充成正方形
bool clip_image_preprocess(const struct clip_ctx * ctx, const struct clip_image_u8 * img, struct clip_image_f32 * res, const bool pad2square);

// 对输入的图像进行编码，使用指定数量的线程进行处理，将结果存储在给定的向量中
bool clip_image_encode(const struct clip_ctx * ctx, const int n_threads, struct clip_image_f32 * img, float * vec);

// 对输入的图像批量进行编码，使用指定数量的线程进行处理，将结果存储在给定的向量中
bool clip_image_batch_encode(const struct clip_ctx * ctx, const int n_threads, const struct clip_image_f32_batch * imgs, float * vec);

// 对模型进行量化，将输入的模型文件进行量化处理，并将结果保存到指定的输出文件中
bool clip_model_quantize(const char * fname_inp, const char * fname_out, const int itype);

#ifdef __cplusplus
}
#endif

#endif // CLIP_H
```