# `PowerInfer\examples\llava\llava.cpp`

```
// 包含所需的头文件
#include "clip.h"
#include "common.h"
#include "llama.h"
#include "llava.h"

// 包含标准输入输出和动态内存分配的头文件
#include <cstdio>
#include <cstdlib>
#include <vector>

// 包含 base64 编码的头文件
#include "base64.hpp"

// 使用 static 修饰的函数，只在当前文件内可见
static bool encode_image_with_clip(clip_ctx * ctx_clip, int n_threads, const clip_image_u8 * img, float * image_embd, int * n_img_pos) {
    // 创建一个浮点数类型的图像对象
    clip_image_f32 * img_res = make_clip_image_f32();
    // 对图像进行预处理，将其转换为浮点数类型的图像
    if (!clip_image_preprocess(ctx_clip, img, img_res, /*pad2square =*/ true)) {
        // 如果预处理失败，输出错误信息并释放内存
        fprintf(stderr, "%s: unable to preprocess image\n", __func__);
        clip_image_f32_free(img_res);
        return false;
    }

    // 获取图像的位置数量
    *n_img_pos = clip_n_patches(ctx_clip);
    // 记录图像编码开始时间
    const int64_t t_img_enc_start_us = ggml_time_us();
    // 对图像进行编码
    bool encoded = clip_image_encode(ctx_clip, n_threads, img_res, image_embd);
    // 释放图像资源
    clip_image_f32_free(img_res);
    // 如果编码失败，输出错误信息并返回false
    if (!encoded) {
        fprintf(stderr, "Unable to encode image\n");
        return false;
    }
    // 记录图像编码结束时间
    const int64_t t_img_enc_end_us = ggml_time_us();
    // 计算图像编码时间
    float t_img_enc_ms = (t_img_enc_end_us - t_img_enc_start_us) / 1000.0;
    // 输出图像编码时间信息
    printf("\n%s: image encoded in %8.2f ms by CLIP (%8.2f ms per image patch)\n", __func__, t_img_enc_ms, t_img_enc_ms / *n_img_pos);
    // 返回true表示验证通过
    return true;
}

// 验证嵌入大小是否正确
bool llava_validate_embed_size(const llama_context * ctx_llama, const clip_ctx * ctx_clip) {
    // 确保使用了正确的mmproj，即进行苹果与苹果的比较
// 获取LLaMA模型的嵌入维度
int n_llama_embd = llama_n_embd(llama_get_model(ctx_llama));
// 获取CLIP模型的图像嵌入维度
auto n_image_embd = clip_n_mmproj_embd(ctx_clip);
// 如果图像嵌入维度与LLaMA模型的嵌入维度不相等，则输出错误信息并返回false
if (n_image_embd != n_llama_embd) {
    printf("%s: embedding dim of the multimodal projector (%d) is not equal to that of LLaMA (%d). Make sure that you use the correct mmproj file.\n", __func__, n_image_embd, n_llama_embd);
    return false;
}
// 返回true
return true;
}

// 使用CLIP模型对图像进行嵌入
static bool llava_image_embed_make_with_clip_img(clip_ctx * ctx_clip, int n_threads, const clip_image_u8 * img, float ** image_embd_out, int * n_img_pos_out) {
    // 分配内存用于存储图像嵌入
    float * image_embd = (float *)malloc(clip_embd_nbytes(ctx_clip));
    // 如果内存分配失败，则输出错误信息并返回false
    if (!image_embd) {
        fprintf(stderr, "Unable to allocate memory for image embeddings\n");
        free(image_embd);
        return false;
    }

    int n_img_pos;
    // 使用CLIP模型对图像进行编码
    if (!encode_image_with_clip(ctx_clip, n_threads, img, image_embd, &n_img_pos)) {
        fprintf(stderr, "%s: cannot encode image, aborting\n", __func__);
```

        // 释放图像嵌入的内存空间
        free(image_embd);
        // 返回 false
        return false;
    }
    // 将图像嵌入和图像位置的指针赋值给输出参数
    *image_embd_out = image_embd;
    *n_img_pos_out = n_img_pos;

    // 返回 true
    return true;
}

// 评估图像嵌入
bool llava_eval_image_embed(llama_context * ctx_llama, const struct llava_image_embed * image_embed, int n_batch, int * n_past) {
    // 获取模型的嵌入维度
    int n_embd  = llama_n_embd(llama_get_model(ctx_llama));

    // 遍历图像位置
    for (int i = 0; i < image_embed->n_image_pos; i += n_batch) {
        // 计算当前批次的评估数量
        int n_eval = image_embed->n_image_pos - i;
        if (n_eval > n_batch) {
            n_eval = n_batch;
        }
        // 创建批次对象
        llama_batch batch = {int32_t(n_eval), nullptr, (image_embed->embed+i*n_embd), nullptr, nullptr, nullptr, nullptr, *n_past, 1, 0, };
        // 解码批次对象
        if (llama_decode(ctx_llama, batch)) {
            // 打印错误信息
            fprintf(stderr, "%s : failed to eval\n", __func__);
// 返回 false，表示操作失败
return false;
// 更新已处理的字节数
*n_past += n_eval;
// 返回 true，表示操作成功
return true;
}

// 根据给定的字节流创建图像嵌入对象
LLAVA_API struct llava_image_embed * llava_image_embed_make_with_bytes(struct clip_ctx * ctx_clip, int n_threads, const unsigned char * image_bytes, int image_bytes_length) {
    // 创建一个无符号8位整型的图像对象
    clip_image_u8 * img = make_clip_image_u8();
    // 如果无法从字节流中加载图像，则释放图像对象并返回空指针
    if (!clip_image_load_from_bytes(image_bytes, image_bytes_length, img)) {
        clip_image_u8_free(img);
        fprintf(stderr, "%s: can't load image from bytes, is it a valid image?", __func__);
        return NULL;
    }

    // 初始化图像嵌入结果和图像位置
    float* image_embed = NULL;
    int n_image_pos = 0;
    // 使用给定的图像对象创建图像嵌入对象，并获取结果和位置
    bool image_embed_result = llava_image_embed_make_with_clip_img(ctx_clip, n_threads, img, &image_embed, &n_image_pos);
    // 如果图像嵌入操作失败，则释放图像对象并返回空指针
    if (!image_embed_result) {
        clip_image_u8_free(img);
// 打印错误信息，指示无法嵌入图像
fprintf(stderr, "%s: coulnd't embed the image\n", __func__);
// 返回空指针，表示无法嵌入图像
return NULL;
}

// 释放图像内存
clip_image_u8_free(img);
// 分配内存以存储嵌入的图像和位置信息
auto result = (llava_image_embed*)malloc(sizeof(llava_image_embed));
// 存储嵌入的图像数据
result->embed = image_embed;
// 存储图像位置信息
result->n_image_pos = n_image_pos;
// 返回包含嵌入图像和位置信息的结构体指针
return result;
}

// 从文件中加载字节数据
static bool load_file_to_bytes(const char* path, unsigned char** bytesOut, long *sizeOut) {
// 打开文件
auto file = fopen(path, "rb");
// 如果无法打开文件，打印错误信息并返回false
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
    if (buffer == NULL) {
        // 如果内存分配失败，打印错误信息并返回 false
        fprintf(stderr, "%s: failed to alloc %ld bytes for file %s\n", __func__, fileSize, path);
        perror("Memory allocation error");
        fclose(file);
        return false;
    }
    // 从文件中读取数据到缓冲区
    fread(buffer, 1, fileSize, file);
    // 关闭文件
    fclose(file);

    // 将缓冲区指针和文件大小传递给调用者
    *bytesOut = buffer;
    *sizeOut = fileSize;
    return true;
}

// 使用文件名创建包含图像数据的结构
LLAVA_API struct llava_image_embed * llava_image_embed_make_with_filename(struct clip_ctx * ctx_clip, int n_threads, const char * image_path) {
    unsigned char* image_bytes;
    long image_bytes_length;
// 调用函数加载图像文件到字节流中，并将结果保存在loaded变量中
auto loaded = load_file_to_bytes(image_path, &image_bytes, &image_bytes_length);
// 如果加载失败，打印错误信息并返回空指针
if (!loaded) {
    fprintf(stderr, "%s: failed to load %s\n", __func__, image_path);
    return NULL;
}
// 调用函数创建一个包含图像字节流的嵌入对象
auto embed = llava_image_embed_make_with_bytes(ctx_clip, n_threads, image_bytes, image_bytes_length);
// 释放图像字节流的内存
free(image_bytes);
// 返回嵌入对象
return embed;
}

// 释放图像嵌入对象的内存
LLAVA_API void llava_image_embed_free(struct llava_image_embed * embed) {
    // 释放嵌入对象中的嵌入数据内存
    free(embed->embed);
    // 释放嵌入对象的内存
    free(embed);
}
```