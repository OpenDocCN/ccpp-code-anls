# `PowerInfer\examples\llava\llava.cpp`

```
// 使用 clip.h 头文件
#include "clip.h"
// 使用 common.h 头文件
#include "common.h"
// 使用 llama.h 头文件
#include "llama.h"
// 使用 llava.h 头文件
#include "llava.h"

// 使用标准输入输出库
#include <cstdio>
// 使用标准库
#include <cstdlib>
// 使用向量库
#include <vector>

// 使用 base64.hpp 头文件
#include "base64.hpp"

// 定义静态函数，使用 clip 上下文、线程数、图像、图像嵌入和图像位置作为参数进行图像编码
static bool encode_image_with_clip(clip_ctx * ctx_clip, int n_threads, const clip_image_u8 * img, float * image_embd, int * n_img_pos) {
    // 创建一个 clip_image_f32 结构体指针，用于存储图像的浮点数表示
    clip_image_f32 * img_res = make_clip_image_f32();
    // 如果无法预处理图像，则打印错误信息并释放内存后返回 false
    if (!clip_image_preprocess(ctx_clip, img, img_res, /*pad2square =*/ true)) {
        fprintf(stderr, "%s: unable to preprocess image\n", __func__);
        clip_image_f32_free(img_res);
        return false;
    }

    // 获取图像块的数量
    *n_img_pos = clip_n_patches(ctx_clip);

    // 获取图像编码开始时间
    const int64_t t_img_enc_start_us = ggml_time_us();
    // 使用 clip 对图像进行编码
    bool encoded = clip_image_encode(ctx_clip, n_threads, img_res, image_embd);
    // 释放图像资源
    clip_image_f32_free(img_res);
    // 如果无法编码图像，则打印错误信息后返回 false
    if (!encoded) {
        fprintf(stderr, "Unable to encode image");
        return false;
    }

    // 获取图像编码结束时间
    const int64_t t_img_enc_end_us = ggml_time_us();
    // 计算图像编码时间
    float t_img_enc_ms = (t_img_enc_end_us - t_img_enc_start_us) / 1000.0;

    // 打印图像编码时间信息
    printf("\n%s: image encoded in %8.2f ms by CLIP (%8.2f ms per image patch)\n", __func__, t_img_enc_ms, t_img_enc_ms / *n_img_pos);

    return true;
}

// 验证图像嵌入的大小是否与 LLaMA 的一致
bool llava_validate_embed_size(const llama_context * ctx_llama, const clip_ctx * ctx_clip) {
    // 确保使用了正确的 mmproj 文件，比较两者的嵌入维度是否相等
    int n_llama_embd = llama_n_embd(llama_get_model(ctx_llama));
    auto n_image_embd = clip_n_mmproj_embd(ctx_clip);
    // 如果图像嵌入维度与 LLaMA 的不相等，则打印错误信息后返回 false
    if (n_image_embd != n_llama_embd) {
        printf("%s: embedding dim of the multimodal projector (%d) is not equal to that of LLaMA (%d). Make sure that you use the correct mmproj file.\n", __func__, n_image_embd, n_llama_embd);
        return false;
    }
    return true;
}

// 使用 clip 上下文、线程数、图像、图像嵌入输出和图像位置输出作为参数进行图像编码
static bool llava_image_embed_make_with_clip_img(clip_ctx * ctx_clip, int n_threads, const clip_image_u8 * img, float ** image_embd_out, int * n_img_pos_out) {
    // 分配内存以存储图像嵌入
    float * image_embd = (float *)malloc(clip_embd_nbytes(ctx_clip));
    # 如果 image_embd 为空，则打印错误信息并释放内存，返回 false
    if (!image_embd) {
        fprintf(stderr, "Unable to allocate memory for image embeddings\n");
        free(image_embd);
        return false;
    }

    # 定义变量 n_img_pos
    int n_img_pos;
    # 如果无法使用 ctx_clip 和 n_threads 对图像进行编码，则打印错误信息，释放内存，返回 false
    if (!encode_image_with_clip(ctx_clip, n_threads, img, image_embd, &n_img_pos)) {
        fprintf(stderr, "%s: cannot encode image, aborting\n", __func__);
        free(image_embd);
        return false;
    }
    # 将 image_embd 和 n_img_pos 赋值给输出参数
    *image_embd_out = image_embd;
    *n_img_pos_out = n_img_pos;

    # 返回 true
    return true;
    // 评估嵌入图像的函数，使用 llama_context 上下文和 llava_image_embed 结构体
    bool llava_eval_image_embed(llama_context * ctx_llama, const struct llava_image_embed * image_embed, int n_batch, int * n_past) {
        // 获取 llama_model 对象的嵌入维度
        int n_embd  = llama_n_embd(llama_get_model(ctx_llama));

        // 遍历图像嵌入的位置
        for (int i = 0; i < image_embed->n_image_pos; i += n_batch) {
            // 计算当前批次的评估数量
            int n_eval = image_embed->n_image_pos - i;
            if (n_eval > n_batch) {
                n_eval = n_batch;
            }
            // 创建 llama_batch 对象
            llama_batch batch = {int32_t(n_eval), nullptr, (image_embed->embed+i*n_embd), nullptr, nullptr, nullptr, nullptr, *n_past, 1, 0, };
            // 调用 llama_decode 函数进行评估
            if (llama_decode(ctx_llama, batch)) {
                fprintf(stderr, "%s : failed to eval\n", __func__);
                return false;
            }
            // 更新已评估的数量
            *n_past += n_eval;
        }
        return true;
    }

    // 使用字节数组创建 llava_image_embed 结构体
    LLAVA_API struct llava_image_embed * llava_image_embed_make_with_bytes(struct clip_ctx * ctx_clip, int n_threads, const unsigned char * image_bytes, int image_bytes_length) {
        // 从字节数组创建 clip_image_u8 对象
        clip_image_u8 * img = make_clip_image_u8();
        if (!clip_image_load_from_bytes(image_bytes, image_bytes_length, img)) {
            clip_image_u8_free(img);
            fprintf(stderr, "%s: can't load image from bytes, is it a valid image?", __func__);
            return NULL;
        }

        // 初始化图像嵌入结果和位置
        float* image_embed = NULL;
        int n_image_pos = 0;
        // 调用 llava_image_embed_make_with_clip_img 函数进行图像嵌入
        bool image_embed_result = llava_image_embed_make_with_clip_img(ctx_clip, n_threads, img, &image_embed, &n_image_pos);
        if (!image_embed_result) {
            clip_image_u8_free(img);
            fprintf(stderr, "%s: coulnd't embed the image\n", __func__);
            return NULL;
        }

        // 释放图像对象
        clip_image_u8_free(img);
        // 分配内存并返回 llava_image_embed 结构体
        auto result = (llava_image_embed*)malloc(sizeof(llava_image_embed));
        result->embed = image_embed;
        result->n_image_pos = n_image_pos;
        return result;
    }

    // 从文件加载字节数组
    static bool load_file_to_bytes(const char* path, unsigned char** bytesOut, long *sizeOut) {
        // 打开文件
        auto file = fopen(path, "rb");
        if (file == NULL) {
            fprintf(stderr, "%s: can't read file %s\n", __func__, path);
            return false;
        }

        // 定位到文件末尾以获取文件大小
        fseek(file, 0, SEEK_END);
    // 获取文件大小
    auto fileSize = ftell(file);
    // 将文件指针移动到文件开头
    fseek(file, 0, SEEK_SET);

    // 分配内存来存储文件数据
    auto buffer = (unsigned char *)malloc(fileSize);
    // 检查内存分配是否成功
    if (buffer == NULL) {
        // 打印错误信息并返回 false
        fprintf(stderr, "%s: failed to alloc %ld bytes for file %s\n", __func__, fileSize, path);
        perror("Memory allocation error");
        fclose(file);
        return false;
    }
    // 将文件数据读取到缓冲区中
    fread(buffer, 1, fileSize, file);
    // 关闭文件
    fclose(file);

    // 将缓冲区和文件大小传递给输出参数
    *bytesOut = buffer;
    *sizeOut = fileSize;
    // 返回 true
    return true;
# 用文件名创建包含图像数据的结构体
LLAVA_API struct llava_image_embed * llava_image_embed_make_with_filename(struct clip_ctx * ctx_clip, int n_threads, const char * image_path) {
    # 定义图像数据的字节流和长度
    unsigned char* image_bytes;
    long image_bytes_length;
    # 调用函数加载文件内容到字节流中
    auto loaded = load_file_to_bytes(image_path, &image_bytes, &image_bytes_length);
    # 如果加载失败，输出错误信息并返回空指针
    if (!loaded) {
        fprintf(stderr, "%s: failed to load %s\n", __func__, image_path);
        return NULL;
    }
    # 调用函数使用图像字节流创建图像嵌入结构体
    auto embed = llava_image_embed_make_with_bytes(ctx_clip, n_threads, image_bytes, image_bytes_length);
    # 释放图像字节流的内存
    free(image_bytes);
    # 返回图像嵌入结构体
    return embed;
}

# 释放图像嵌入结构体的内存
LLAVA_API void llava_image_embed_free(struct llava_image_embed * embed) {
    # 释放图像嵌入结构体中的嵌入数据内存
    free(embed->embed);
    # 释放图像嵌入结构体的内存
    free(embed);
}
```