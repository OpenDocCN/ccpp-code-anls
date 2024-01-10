# `PowerInfer\examples\llava\clip.cpp`

```
// NOTE: This is modified from clip.cpp only for LLaVA,
// so there might be still unnecessary artifacts hanging around
// I'll gradually clean and extend it

#include <cassert>  // 断言库
#include <cmath>  // 数学函数库
#include <cstdlib>  // C 标准库
#include <cstring>  // 字符串操作库
#include <fstream>  // 文件流库
#include <iostream>  // 输入输出流库
#include <map>  // 映射库
#include <regex>  // 正则表达式库
#include <stdexcept>  // 异常库
#include <vector>  // 向量库

#include "clip.h"  // 引入 clip.h 头文件
#include "ggml.h"  // 引入 ggml.h 头文件
#include "ggml-alloc.h"  // 引入 ggml-alloc.h 头文件

#define STB_IMAGE_IMPLEMENTATION  // 定义 STB_IMAGE_IMPLEMENTATION

#include "stb_image.h"  // 引入 stb_image.h 头文件

#define CLIP_DEBUG  // 定义 CLIP_DEBUG

static std::string format(const char * fmt, ...) {  // 定义格式化字符串函数
    va_list ap;  // 定义可变参数列表
    va_list ap2;  // 定义可变参数列表
    va_start(ap, fmt);  // 初始化可变参数列表
    va_copy(ap2, ap);  // 复制可变参数列表
    int size = vsnprintf(NULL, 0, fmt, ap);  // 计算格式化后字符串的长度
    GGML_ASSERT(size >= 0 && size < INT_MAX); // NOLINT
    std::vector<char> buf(size + 1);  // 创建字符数组
    int size2 = vsnprintf(buf.data(), size + 1, fmt, ap2);  // 格式化字符串
    GGML_ASSERT(size2 == size);  // 断言格式化后字符串长度正确
    va_end(ap2);  // 结束可变参数列表
    va_end(ap);  // 结束可变参数列表
    return std::string(buf.data(), buf.size());  // 返回格式化后的字符串
}

//
// key constants
//

#define KEY_FTYPE "general.file_type"  // 定义文件类型键
#define KEY_NAME "general.name"  // 定义名称键
#define KEY_DESCRIPTION "general.description"  // 定义描述键
#define KEY_HAS_TEXT_ENC "clip.has_text_encoder"  // 定义是否有文本编码器键
#define KEY_HAS_VIS_ENC "clip.has_vision_encoder"  // 定义是否有视觉编码器键
#define KEY_HAS_LLAVA_PROJ "clip.has_llava_projector"  // 定义是否有 LLaVA 投影仪键
#define KEY_USE_GELU "clip.use_gelu"  // 定义是否使用 GELU 键
#define KEY_N_EMBD "clip.%s.embedding_length"  // 定义嵌入长度键
#define KEY_N_FF "clip.%s.feed_forward_length"  // 定义前馈长度键
#define KEY_N_BLOCK "clip.%s.block_count"  // 定义块数量键
#define KEY_N_HEAD "clip.%s.attention.head_count"  // 定义注意力头数量键
#define KEY_LAYER_NORM_EPS "clip.%s.attention.layer_norm_epsilon"  // 定义层归一化 epsilon 键
#define KEY_PROJ_DIM "clip.%s.projection_dim"  // 定义投影维度键
#define KEY_TOKENS "tokenizer.ggml.tokens"  // 定义标记键
#define KEY_N_POSITIONS "clip.text.context_length"  // 定义文本上下文长度键
#define KEY_IMAGE_SIZE "clip.vision.image_size"  // 定义图像大小键
#define KEY_PATCH_SIZE "clip.vision.patch_size"  // 定义补丁大小键
#define KEY_IMAGE_MEAN "clip.vision.image_mean"  // 定义图像均值键
#define KEY_IMAGE_STD "clip.vision.image_std"  // 定义图像标准差键

//
// tensor name constants
//

#define TN_TOKEN_EMBD "%s.token_embd.weight"  // 定义标记嵌入张量名称
#define TN_POS_EMBD "%s.position_embd.weight"  // 定义位置嵌入张量名称
#define TN_CLASS_EMBD "v.class_embd"  // 定义类别嵌入张量名称
// 定义常量字符串，表示不同的文件名
#define TN_PATCH_EMBD "v.patch_embd.weight"
#define TN_ATTN_K "%s.blk.%d.attn_k.%s"
#define TN_ATTN_Q "%s.blk.%d.attn_q.%s"
#define TN_ATTN_V "%s.blk.%d.attn_v.%s"
#define TN_ATTN_OUTPUT "%s.blk.%d.attn_out.%s"
#define TN_FFN_DOWN "%s.blk.%d.ffn_down.%s"
#define TN_FFN_UP "%s.blk.%d.ffn_up.%s"
#define TN_LN_1 "%s.blk.%d.ln1.%s"
#define TN_LN_2 "%s.blk.%d.ln2.%s"
#define TN_LN_PRE "%s.pre_ln.%s"
#define TN_LN_POST "%s.post_ln.%s"
#define TN_TEXT_PROJ "text_projection.weight"
#define TN_VIS_PROJ "visual_projection.weight"
#define TN_LLAVA_PROJ "mm.%d.%s"

//
// 从 gguf 文件中获取数据的工具函数
//

// 获取指定键在 gguf 上下文中的索引
static int get_key_idx(const gguf_context * ctx, const char * key) {
    int i = gguf_find_key(ctx, key);
    if (i == -1) {
        // 如果找不到指定的键，则输出错误信息并抛出异常
        fprintf(stderr, "key %s not found in file\n", key);
        throw std::runtime_error(format("Missing required key: %s", key));
    }

    return i;
}

// 获取指定键对应的无符号 32 位整数值
static uint32_t get_u32(const gguf_context * ctx, const std::string & key) {
    const int i = get_key_idx(ctx, key.c_str());

    return gguf_get_val_u32(ctx, i);
}

// 获取指定键对应的单精度浮点数值
static float get_f32(const gguf_context * ctx, const std::string & key) {
    const int i = get_key_idx(ctx, key.c_str());

    return gguf_get_val_f32(ctx, i);
}

// 获取指定名称的张量
static struct ggml_tensor * get_tensor(struct ggml_context * ctx, const std::string & name) {
    struct ggml_tensor * cur = ggml_get_tensor(ctx, name.c_str());
    if (!cur) {
        // 如果找不到指定名称的张量，则抛出异常
        throw std::runtime_error(format("%s: unable to find tensor %s\n", __func__, name.c_str()));
    }

    return cur;
}

// 获取文件类型的字符串表示
static std::string get_ftype(int ftype) {
    switch (ftype) {
    case 0:
        return "f32";
    case 1:
        return "f16";
    case 2:
        return "q4_0";
    case 3:
        return "q4_1";
    case 6:
        return "q5_0";
    case 7:
        return "q5_1";
    case 8:
        return "q8_0";
    default:
        // 如果文件类型不在已知范围内，则抛出异常
        throw std::runtime_error(format("%s: Unrecognized file type: %d\n", __func__, ftype));
    }
}

//
// 剪辑层
//

struct clip_layer {
    // 定义变量 k_w，表示注意力机制中的权重
    struct ggml_tensor * k_w;
    // 定义变量 k_b，表示注意力机制中的偏置
    struct ggml_tensor * k_b;
    // 定义变量 q_w，表示注意力机制中的权重
    struct ggml_tensor * q_w;
    // 定义变量 q_b，表示注意力机制中的偏置
    struct ggml_tensor * q_b;
    // 定义变量 v_w，表示注意力机制中的权重
    struct ggml_tensor * v_w;
    // 定义变量 v_b，表示注意力机制中的偏置
    struct ggml_tensor * v_b;
    
    // 定义变量 o_w，表示输出层中的权重
    struct ggml_tensor * o_w;
    // 定义变量 o_b，表示输出层中的偏置
    struct ggml_tensor * o_b;
    
    // 定义变量 ln_1_w，表示第一个 LayerNorm 层中的权重
    struct ggml_tensor * ln_1_w;
    // 定义变量 ln_1_b，表示第一个 LayerNorm 层中的偏置
    struct ggml_tensor * ln_1_b;
    
    // 定义变量 ff_i_w，表示前馈神经网络中输入层的权重
    struct ggml_tensor * ff_i_w;
    // 定义变量 ff_i_b，表示前馈神经网络中输入层的偏置
    struct ggml_tensor * ff_i_b;
    
    // 定义变量 ff_o_w，表示前馈神经网络中输出层的权重
    struct ggml_tensor * ff_o_w;
    // 定义变量 ff_o_b，表示前馈神经网络中输出层的偏置
    struct ggml_tensor * ff_o_b;
    
    // 定义变量 ln_2_w，表示第二个 LayerNorm 层中的权重
    struct ggml_tensor * ln_2_w;
    // 定义变量 ln_2_b，表示第二个 LayerNorm 层中的偏置
    struct ggml_tensor * ln_2_b;
};

// 定义一个结构体，包含视觉模型的超参数和各种张量
struct clip_vision_model {
    struct clip_vision_hparams hparams;

    // 嵌入
    struct ggml_tensor * class_embedding;
    struct ggml_tensor * patch_embeddings;
    struct ggml_tensor * position_embeddings;

    struct ggml_tensor * pre_ln_w;
    struct ggml_tensor * pre_ln_b;

    std::vector<clip_layer> layers;

    struct ggml_tensor * post_ln_w;
    struct ggml_tensor * post_ln_b;

    struct ggml_tensor * projection;

    // LLaVA 投影
    struct ggml_tensor * mm_0_w;
    struct ggml_tensor * mm_0_b;
    struct ggml_tensor * mm_2_w;
    struct ggml_tensor * mm_2_b;
};

// 替代 std::vector<uint8_t> 的结构体，不需要零初始化
struct clip_buffer {
    uint8_t * data = NULL;
    size_t size = 0;

    // 重新调整缓冲区大小
    void resize(size_t size) {
        delete[] data;
        data = new uint8_t[size];
        this->size = size;
    }

    // 析构函数，释放内存
    ~clip_buffer() { delete[] data; }
};

// 定义一个结构体，包含上下文信息和模型评估所需的内存缓冲区
struct clip_ctx {
    bool has_text_encoder = false;
    bool has_vision_encoder = false;
    bool has_llava_projector = false;
    struct clip_vision_model vision_model;
    float image_mean[3];
    float image_std[3];
    bool use_gelu = false;
    int32_t ftype = 1;
    struct ggml_context * ctx;
    struct gguf_context * ctx_gguf;

    // 用于评估模型的内存缓冲区
    clip_buffer buf_compute;
    clip_buffer buf_alloc;
    ggml_allocr * alloc = NULL;
};

// 构建图形的函数，根据上下文和图像批次构建图形
static ggml_cgraph * clip_image_build_graph(const clip_ctx * ctx, const clip_image_f32_batch * imgs) {
    // 如果没有视觉编码器，则打印错误信息并返回空指针
    if (!ctx->has_vision_encoder) {
        printf("This gguf file seems to have no vision encoder\n");
        return nullptr;
    }

    // 获取模型和超参数
    const auto & model = ctx->vision_model;
    const auto & hparams = model.hparams;

    // 计算图像大小、补丁大小、补丁数量、位置数量和隐藏层大小
    const int image_size = hparams.image_size;
    const int patch_size = hparams.patch_size;
    const int num_patches = ((image_size / patch_size) * (image_size / patch_size));
    const int num_positions = num_patches + 1;
    const int hidden_size = hparams.hidden_size;
    // 定义变量n_head，表示注意力头的数量
    const int n_head = hparams.n_head;
    // 定义变量d_head，表示每个注意力头的隐藏层大小
    const int d_head = hidden_size / n_head;
    // 定义变量n_layer，表示模型的层数
    const int n_layer = hparams.n_layer;
    // 定义变量eps，表示一个小的常数值
    const float eps = hparams.eps;
    // 获取输入数据的批量大小
    int batch_size = imgs->size;
    // 如果上下文中有llava_projector，则断言批量大小为1
    if(ctx->has_llava_projector) {
        GGML_ASSERT(batch_size == 1);
    }

    // 获取上下文中的buf_compute
    const auto & buf_compute = ctx->buf_compute;

    // 初始化参数结构体params
    struct ggml_init_params params = {
        /*.mem_size =*/ buf_compute.size,
        /*.mem_buffer =*/ buf_compute.data,
        /*.no_alloc =*/ false,
    };

    // 设置参数结构体params的no_alloc为true
    params.no_alloc = true;

    // 初始化上下文ctx0
    struct ggml_context * ctx0 = ggml_init(params);
    // 创建新的计算图gf
    struct ggml_cgraph * gf = ggml_new_graph(ctx0);

    // 创建原始输入张量inp_raw
    struct ggml_tensor * inp_raw = ggml_new_tensor_4d(ctx0, GGML_TYPE_F32, image_size, image_size, 3, batch_size);
    // 分配inp_raw的内存
    ggml_allocr_alloc(ctx->alloc, inp_raw);

    // 如果不是测量模式
    if (!ggml_allocr_is_measure(ctx->alloc)) {
        // 获取inp_raw的数据指针
        float * data = (float *)ggml_get_data(inp_raw);

        // 遍历输入图片
        for (size_t i = 0; i < imgs->size; i++) {
            // 获取图片的宽和高
            const int nx = imgs->data[i].nx;
            const int ny = imgs->data[i].ny;
            // 断言图片的宽和高与image_size相等
            GGML_ASSERT(nx == image_size && ny == image_size);

            // 计算图片像素点的数量
            const int n = nx * ny;

            // 遍历批量中的每张图片
            for (int b = 0; b < batch_size; b++) {
                // 遍历RGB三个通道
                for (int k = 0; k < 3; k++) {
                    // 遍历图片的每个像素点
                    for (int y = 0; y < ny; y++) {
                        for (int x = 0; x < nx; x++) {
                            // 将图片数据存入inp_raw中
                            data[(b * 3 * n) + k * n + y * nx + x] = imgs->data[b].data[3 * (y * nx + x) + k];
                        }
                    }
                }
            }
        }
    }

    // 对inp_raw进行二维卷积操作，得到inp
    struct ggml_tensor * inp = ggml_conv_2d(ctx0, model.patch_embeddings, inp_raw, patch_size, patch_size, 0, 0, 1, 1);
    // 对inp进行三维重塑操作，得到inp
    inp = ggml_reshape_3d(ctx0, inp, num_patches, hidden_size, batch_size);
    // 对inp进行维度置换操作，得到inp
    inp = ggml_cont(ctx0, ggml_permute(ctx0, inp, 1, 0, 2, 3));

    // 连接class_embeddings和patch_embeddings
    # 创建一个三维张量 embeddings，包含隐藏大小、位置数量和批处理大小
    struct ggml_tensor * embeddings = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, hidden_size, num_positions, batch_size);
    # 分配内存给 embeddings
    ggml_allocr_alloc(ctx->alloc, embeddings);
    # 如果内存未被测量，则将 embeddings 设置为零
    if (!ggml_allocr_is_measure(ctx->alloc)) {
        ggml_set_zero(embeddings);
    }

    # 创建一个三维张量 temp，包含隐藏大小、1 和批处理大小
    struct ggml_tensor * temp = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, hidden_size, 1, batch_size);
    # 分配内存给 temp
    ggml_allocr_alloc(ctx->alloc, temp);

    # 使用 ggml_repeat 函数将 model.class_embedding 重复 temp 次数，并将结果与 embeddings 相加
    embeddings = ggml_acc(ctx0, embeddings, ggml_repeat(ctx0, model.class_embedding, temp), embeddings->nb[1],
                          embeddings->nb[2], embeddings->nb[3], 0);
    # 使用 ggml_acc 函数将 inp 与 embeddings 相加
    embeddings =
        ggml_acc(ctx0, embeddings, inp, embeddings->nb[1], embeddings->nb[2], embeddings->nb[3], model.class_embedding->nb[1]);

    # 创建一个一维张量 positions，包含位置数量
    struct ggml_tensor * positions = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, num_positions);
    # 分配内存给 positions
    ggml_allocr_alloc(ctx->alloc, positions);
    # 如果内存未被测量，则将 positions 设置为从 0 到 num_positions-1 的整数
    if (!ggml_allocr_is_measure(ctx->alloc)) {
        for (int i = 0; i < num_positions; i++) {
            ggml_set_i32_1d(positions, i, i);
        }
    }

    # 使用 ggml_get_rows 函数获取 model.position_embeddings 中与 positions 对应的行，并将结果与 embeddings 相加
    embeddings =
        ggml_add(ctx0, embeddings, ggml_repeat(ctx0, ggml_get_rows(ctx0, model.position_embeddings, positions), embeddings));

    # 对 embeddings 进行预层归一化
    {
        embeddings = ggml_norm(ctx0, embeddings, eps);
        # 使用 ggml_repeat 函数将 model.pre_ln_w 重复 embeddings 次数，并将结果与 embeddings 相乘，再与 ggml_repeat 函数将 model.pre_ln_b 重复 embeddings 次数的结果相加
        embeddings = ggml_add(ctx0, ggml_mul(ctx0, ggml_repeat(ctx0, model.pre_ln_w, embeddings), embeddings),
                              ggml_repeat(ctx0, model.pre_ln_b, embeddings));
    }

    # 创建一个一维张量 KQ_scale，包含一个元素
    struct ggml_tensor * KQ_scale = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
    # 分配内存给 KQ_scale
    ggml_allocr_alloc(ctx->alloc, KQ_scale);
    # 如果内存未被测量，则将 KQ_scale 设置为 1.0 除以 d_head 的平方根
    if (!ggml_allocr_is_measure(ctx->alloc)) {
        ggml_set_f32(KQ_scale, 1.0f / sqrt((float)d_head));
    }

    # 循环遍历层
    }

    # llava projector
    # 重新调整嵌入矩阵的维度
    embeddings = ggml_reshape_2d(ctx0, embeddings, embeddings->ne[0], embeddings->ne[1]);

    # 创建一个包含 num_patches 个元素的一维张量
    struct ggml_tensor * patches = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, num_patches);
    # 分配内存给 patches
    ggml_allocr_alloc(ctx->alloc, patches);
    # 如果内存分配成功
    if (!ggml_allocr_is_measure(ctx->alloc)) {
        # 对 patches 中的每个元素赋值
        for (int i = 0; i < num_patches; ++i) {
            ggml_set_i32_1d(patches, i, i+1);
        }
    }

    # 从嵌入矩阵中获取指定行的数据
    embeddings = ggml_get_rows(ctx0, embeddings, patches);

    # mm projection 0
    # 对嵌入矩阵进行矩阵乘法操作
    embeddings = ggml_mul_mat(ctx0, model.mm_0_w, embeddings);
    # 对嵌入矩阵进行加法操作
    embeddings = ggml_add(ctx0, ggml_repeat(ctx0, model.mm_0_b, embeddings), embeddings);

    # 对嵌入矩阵进行 GELU 激活函数操作
    embeddings = ggml_gelu(ctx0, embeddings);

    # 对嵌入矩阵进行矩阵乘法操作
    embeddings = ggml_mul_mat(ctx0, model.mm_2_w, embeddings);
    # 对嵌入矩阵进行加法操作
    embeddings = ggml_add(ctx0, ggml_repeat(ctx0, model.mm_2_b, embeddings), embeddings);
    # 构建前向传播图
    ggml_build_forward_expand(gf, embeddings);

    # 释放上下文
    ggml_free(ctx0);

    # 返回前向传播图
    return gf;
// 读取并创建包含张量及其数据的 ggml_context
struct clip_ctx * clip_model_load(const char * fname, const int verbosity = 1) {

    // 初始化 meta 为 NULL
    struct ggml_context * meta = NULL;

    // 初始化 gguf_init_params 结构体
    struct gguf_init_params params = {
        /*.no_alloc = */ true,  // 设置 no_alloc 为 true
        /*.ctx      = */ &meta,  // 设置 ctx 指向 meta
    };

    // 从文件中初始化 gguf_context
    struct gguf_context * ctx = gguf_init_from_file(fname, params);
    // 如果初始化失败，抛出运行时错误
    if (!ctx) {
        throw std::runtime_error(format("%s: failed to load CLIP model from %s. Does this file exist?\n", __func__, fname));
    }

    // 如果 verbosity 大于等于 1
    if (verbosity >= 1) {
        // 获取张量数量
        const int n_tensors = gguf_get_n_tensors(ctx);
        // 获取键值对数量
        const int n_kv = gguf_get_n_kv(ctx);
        // 获取文件类型
        const int ftype = get_u32(ctx, KEY_FTYPE);
        // 获取文件类型的字符串表示
        const std::string ftype_str = get_ftype(ftype);
        // 获取描述键的索引
        const int idx_desc = get_key_idx(ctx, KEY_DESCRIPTION);
        // 获取描述信息
        const std::string description = gguf_get_val_str(ctx, idx_desc);
        // 获取名称键的索引
        const int idx_name = gguf_find_key(ctx, KEY_NAME);
        // 如果索引不为 -1
        if (idx_name != -1) { // 暂时将名称设置为可选，因为一些上传的模型由于错误而缺少它
            // 获取名称信息
            const std::string name = gguf_get_val_str(ctx, idx_name);
            // 打印模型名称
            printf("%s: model name:   %s\n", __func__, name.c_str());
        }
        // 打印描述信息
        printf("%s: description:  %s\n", __func__, description.c_str());
        // 打印 GGUF 版本
        printf("%s: GGUF version: %d\n", __func__, gguf_get_version(ctx));
        // 打印对齐方式
        printf("%s: alignment:    %zu\n", __func__, gguf_get_alignment(ctx));
        // 打印张量数量
        printf("%s: n_tensors:    %d\n", __func__, n_tensors);
        // 打印键值对数量
        printf("%s: n_kv:         %d\n", __func__, n_kv);
        // 打印文件类型
        printf("%s: ftype:        %s\n", __func__, ftype_str.c_str());
        // 打印换行
        printf("\n");
    }

    // 如果 verbosity 大于等于 3
    if (verbosity >= 3) {
        // 获取键值对数量
        const int n_kv = gguf_get_n_kv(ctx);

        // 遍历键值对
        for (int i = 0; i < n_kv; ++i) {
            // 获取键
            const char * key = gguf_get_key(ctx, i);

            // 打印键值对信息
            printf("%s: kv[%d]: key = %s\n", __func__, i, key);
        }
        // 打印换行
        printf("\n");
    }

    // 数据
    size_t ctx_size = 0;
    {
        // 获取上下文中张量的数量
        const int n_tensors = gguf_get_n_tensors(ctx);

        // 遍历每个张量
        for (int i = 0; i < n_tensors; ++i) {
            // 获取张量的名称
            const char * name = gguf_get_tensor_name(ctx, i);
            // 获取张量的偏移量
            const size_t offset = gguf_get_tensor_offset(ctx, i);

            // 通过名称获取元数据中的张量
            struct ggml_tensor * cur = ggml_get_tensor(meta, name);
            // 计算上下文大小，包括张量结构和对象大小
            ctx_size += sizeof(struct ggml_tensor) + GGML_OBJECT_SIZE;
            // 获取张量的大小
            size_t tensor_size = ggml_nbytes(cur);
            // 获取填充后的张量大小
            size_t padded_size = ggml_nbytes_pad(cur);
            // 更新上下文大小
            ctx_size += padded_size;
            // 如果详细程度大于等于3，打印张量信息
            if (verbosity >= 3) {
                printf("%s: tensor[%d]: n_dims = %d, name = %s, tensor_size=%zu, padded_size=%zu, offset=%zu\n", __func__, i,
                       cur->n_dims, cur->name, tensor_size, padded_size, offset);
            }
        }
    }

    // 创建新的剪辑上下文对象
    clip_ctx * new_clip = new clip_ctx;

    // 模型大小和功能
    {
        // 获取键值为 KEY_HAS_TEXT_ENC 的索引
        int idx = get_key_idx(ctx, KEY_HAS_TEXT_ENC);
        // 将获取到的布尔值赋给新剪贴板的 has_text_encoder 属性
        new_clip->has_text_encoder = gguf_get_val_bool(ctx, idx);

        // 获取键值为 KEY_HAS_VIS_ENC 的索引
        idx = get_key_idx(ctx, KEY_HAS_VIS_ENC);
        // 将获取到的布尔值赋给新剪贴板的 has_vision_encoder 属性
        new_clip->has_vision_encoder = gguf_get_val_bool(ctx, idx);

        // 查找键值为 KEY_HAS_LLAVA_PROJ 的索引
        idx = gguf_find_key(ctx, KEY_HAS_LLAVA_PROJ);
        // 如果找到了键值为 KEY_HAS_LLAVA_PROJ 的索引，则将获取到的布尔值赋给新剪贴板的 has_llava_projector 属性
        if (idx != -1) {
            new_clip->has_llava_projector = gguf_get_val_bool(ctx, idx);
        }

        // 断言新剪贴板的 has_llava_projector 属性为真
        GGML_ASSERT(new_clip->has_llava_projector); // see monatis/clip.cpp for image and/or text encoding for semantic search
        // 断言新剪贴板的 has_vision_encoder 属性为真
        GGML_ASSERT(new_clip->has_vision_encoder);
        // 断言新剪贴板的 has_text_encoder 属性为假
        GGML_ASSERT(!new_clip->has_text_encoder);

        // 获取键值为 KEY_USE_GELU 的索引
        idx = get_key_idx(ctx, KEY_USE_GELU);
        // 将获取到的布尔值赋给新剪贴板的 use_gelu 属性
        new_clip->use_gelu = gguf_get_val_bool(ctx, idx);

        // 如果详细程度大于等于1，则打印以下信息
        if (verbosity >= 1) {
            printf("%s: text_encoder:   %d\n", __func__, new_clip->has_text_encoder);
            printf("%s: vision_encoder: %d\n", __func__, new_clip->has_vision_encoder);
            printf("%s: llava_projector:  %d\n", __func__, new_clip->has_llava_projector);
            printf("%s: model size:     %.2f MB\n", __func__, (ctx_size / 1024.0 / 1024.0));
            printf("%s: metadata size:  %.2f MB\n", __func__, ggml_get_mem_size(meta) / 1024.0 / 1024.0);
        }
    }

    // 加载张量
    {
        // 初始化参数结构体，设置内存大小为ctx_size，内存缓冲区为NULL，允许分配内存
        struct ggml_init_params params = {
            /*.mem_size =*/ ctx_size,
            /*.mem_buffer =*/ NULL,
            /*.no_alloc =*/ false,
        };

        // 使用初始化参数初始化上下文
        new_clip->ctx = ggml_init(params);
        // 如果初始化失败，输出错误信息，释放资源，返回空指针
        if (!new_clip->ctx) {
            fprintf(stderr, "%s: ggml_init() failed\n", __func__);
            clip_free(new_clip);
            return nullptr;
        }

        // 打开二进制文件流
        auto fin = std::ifstream(fname, std::ios::binary);
        // 如果无法打开文件，输出错误信息，释放资源，返回空指针
        if (!fin) {
            printf("cannot open model file for loading tensors\n");
            clip_free(new_clip);
            return nullptr;
        }

        // 获取上下文中张量的数量
        const int n_tensors = gguf_get_n_tensors(ctx);
        // 遍历每个张量
        for (int i = 0; i < n_tensors; ++i) {
            // 获取张量的名称
            const char * name = gguf_get_tensor_name(ctx, i);
            // 获取元数据中的张量
            struct ggml_tensor * t = ggml_get_tensor(meta, name);
            // 复制张量到新的上下文中
            struct ggml_tensor * cur = ggml_dup_tensor(new_clip->ctx, t);
            // 设置张量的名称
            ggml_set_name(cur, name);

            // 计算张量数据在文件中的偏移量
            const size_t offset = gguf_get_data_offset(ctx) + gguf_get_tensor_offset(ctx, i);
            // 定位到文件中的偏移位置
            fin.seekg(offset, std::ios::beg);
            // 如果定位失败，输出错误信息，释放资源，返回空指针
            if (!fin) {
                printf("%s: failed to seek for tensor %s\n", __func__, name);
                clip_free(new_clip);
                return nullptr;
            }

            // 从文件中读取张量数据到当前张量的数据缓冲区
            fin.read(reinterpret_cast<char *>(cur->data), ggml_nbytes(t));
        }

        // 关闭文件流
        fin.close();
    }

    // vision model
    }

    // 释放元数据资源
    ggml_free(meta);

    // 将上下文中的gguf赋值给新的上下文
    new_clip->ctx_gguf = ctx;
// 测量内存需求并分配
{
    // 定义张量对齐要求
    static const size_t tensor_alignment = 32;
    // 调整新剪辑的计算缓冲区大小
    new_clip->buf_compute.resize(ggml_tensor_overhead()*GGML_DEFAULT_GRAPH_SIZE + ggml_graph_overhead());
    // 为新剪辑分配内存
    new_clip->alloc = ggml_allocr_new_measure(tensor_alignment);
    // 创建图像批次
    clip_image_f32_batch batch;
    batch.size = 1;
    // 构建图形
    ggml_cgraph * gf = clip_image_build_graph(new_clip, &batch);
    // 计算分配大小
    size_t alloc_size = ggml_allocr_alloc_graph(new_clip->alloc, gf) + tensor_alignment;
    // 释放内存
    ggml_allocr_free(new_clip->alloc);
    // 调整新剪辑的分配缓冲区大小
    new_clip->buf_alloc.resize(alloc_size);
    // 为新剪辑分配内存
    new_clip->alloc = ggml_allocr_new(new_clip->buf_alloc.data, new_clip->buf_alloc.size, tensor_alignment);

    // 打印总分配的内存大小
    printf("%s: total allocated memory: %.2f MB\n", __func__, (new_clip->buf_compute.size + alloc_size)/1024.0/1024.0);
}

// 返回新剪辑
return new_clip;
}

// 创建无符号8位图像剪辑
clip_image_u8 * make_clip_image_u8() {
    // 创建新的无符号8位图像剪辑
    auto img = new clip_image_u8();
    // 返回图像剪辑
    return img;
}
// 创建32位浮点图像剪辑
clip_image_f32 * make_clip_image_f32() { return new clip_image_f32(); }

// 释放无符号8位图像剪辑
void clip_image_u8_free(clip_image_u8 * img) { if (img->data) { delete[] img->data; } delete img; }
// 释放32位浮点图像剪辑
void clip_image_f32_free(clip_image_f32 * img) { if (img->data) { delete[] img->data; } delete img; }

// 从数据构建图像剪辑
static void build_clip_img_from_data(const stbi_uc * data, int nx, int ny, clip_image_u8 * img) {
    // 设置图像剪辑的尺寸和数据
    img->nx = nx;
    img->ny = ny;
    img->size = nx * ny * 3;
    img->data = new uint8_t[img->size]();
    // 复制数据到图像剪辑
    memcpy(img->data, data, img->size);
}

// 从文件加载图像剪辑
bool clip_image_load_from_file(const char * fname, clip_image_u8 * img) {
    int nx, ny, nc;
    // 加载图像数据
    auto data = stbi_load(fname, &nx, &ny, &nc, 3);
    // 如果加载失败，则打印错误信息并返回false
    if (!data) {
        fprintf(stderr, "%s: failed to load image '%s'\n", __func__, fname);
        return false;
    }
    // 从数据构建图像剪辑
    build_clip_img_from_data(data, nx, ny, img);
    // 释放图像数据
    stbi_image_free(data);
    // 返回true
    return true;
}

// 从字节加载图像剪辑
bool clip_image_load_from_bytes(const unsigned char * bytes, size_t bytes_length, struct clip_image_u8 * img) {
    int nx, ny, nc;
    // ...
}
    // 从内存中加载图像数据，并获取图像的宽度、高度和通道数
    auto data = stbi_load_from_memory(bytes, bytes_length, &nx, &ny, &nc, 3);
    // 如果加载失败，则输出错误信息并返回 false
    if (!data) {
        fprintf(stderr, "%s: failed to decode image bytes\n", __func__);
        return false;
    }
    // 根据图像数据构建剪裁后的图像
    build_clip_img_from_data(data, nx, ny, img);
    // 释放图像数据占用的内存
    stbi_image_free(data);
    // 返回 true，表示图像处理成功
    return true;
// normalize: x = (x - mean) / std
// TODO: implement bicubic interpolation instead of linear.
// 对图像进行预处理，将像素值标准化为 (x - mean) / std，如果需要，实现双三次插值而不是线性插值
bool clip_image_preprocess(const clip_ctx * ctx, const clip_image_u8 * img, clip_image_f32 * res, const bool pad2square) {
    // 如果上下文中没有视觉编码器，则打印错误信息并返回 false
    if (!ctx->has_vision_encoder) {
        printf("This gguf file seems to have no vision encoder\n");
        return false;
    }

    // 下面的逻辑是将较短的一边填充到较长的一边，使用背景颜色：rgb(122, 116, 104)
    // 参考 https://github.com/haotian-liu/LLaVA/blob/e854a2bf85118c504f6f16bf5c3c7c92f8fa8c6b/llava/conversation.py#L113-L156

    // 创建临时图像对象，用于暂时保存输入图像数据
    clip_image_u8 * temp = make_clip_image_u8(); 
    if (pad2square && img->nx != img->ny) {
        // 如果需要填充成正方形且图像的宽高不相等
        int longer_side = std::max(img->nx, img->ny);
        temp->nx = longer_side;
        temp->ny = longer_side;
        temp->size = 3 * longer_side * longer_side;
        temp->data = new uint8_t[temp->size]();
        uint8_t bc[3] = {122, 116, 104}; // LLaVA 中的背景颜色，RGB 格式

        // 使用背景颜色填充
        for (size_t i = 0; i < temp->size; i++) {
            temp->data[i] = bc[i % 3];
        }

        // 从输入图像复制数据
        for (int y = 0; y < img->ny; y++) {
            for (int x = 0; x < img->nx; x++) {
                const int i = 3 * (y * img->nx + x);
                const int j = 3 * (y * temp->nx + x);
                temp->data[j] = img->data[i];
                temp->data[j+1] = img->data[i+1];
                temp->data[j+2] = img->data[i+2];
            }
        }
    } else {
        // 如果不需要填充成正方形或者图像的宽高相等，则直接复制输入图像数据
        temp->nx   = img->nx;
        temp->ny   = img->ny;
        temp->size = img->size;
        temp->data = new uint8_t[temp->size]();
        memcpy(&temp->data[0], &img->data[0], temp->size); // 复制
    }

    // 获取临时图像的宽高
    const int nx = temp->nx;
    const int ny = temp->ny;

    // 获取视觉模型的图像大小
    const int nx2 = ctx->vision_model.hparams.image_size;
    const int ny2 = ctx->vision_model.hparams.image_size;
    // 设置结果 res 的宽度为 nx2
    res->nx = nx2;
    // 设置结果 res 的高度为 ny2
    res->ny = ny2;
    // 设置结果 res 的大小为 3 * nx2 * ny2
    res->size = 3 * nx2 * ny2;
    // 为结果 res 的数据分配内存空间，并初始化为 0
    res->data = new float[res->size]();

    // 计算缩放比例，将原始图像的宽度和高度缩放到模型要求的大小
    const float scale = std::max(nx, ny) / (float)ctx->vision_model.hparams.image_size;

    // 计算缩放后的宽度和高度
    const int nx3 = int(nx / scale + 0.5f);
    const int ny3 = int(ny / scale + 0.5f);

    // 获取图像的均值和标准差
    const auto & m3 = ctx->image_mean; // {0.48145466f, 0.4578275f, 0.40821073f};
    const auto & s3 = ctx->image_std;  // {0.26862954f, 0.26130258f, 0.27577711f};

    // 对缩放后的图像进行插值处理
    for (int y = 0; y < ny3; y++) {
        for (int x = 0; x < nx3; x++) {
            for (int c = 0; c < 3; c++) {
                // 计算插值点的坐标
                const float sx = (x + 0.5f) * scale - 0.5f;
                const float sy = (y + 0.5f) * scale - 0.5f;

                // 计算插值点周围的四个像素坐标
                const int x0 = std::max(0, (int)std::floor(sx));
                const int y0 = std::max(0, (int)std::floor(sy));
                const int x1 = std::min(x0 + 1, nx - 1);
                const int y1 = std::min(y0 + 1, ny - 1);

                // 计算插值点在四个像素之间的偏移量
                const float dx = sx - x0;
                const float dy = sy - y0;

                // 计算插值点周围四个像素的索引
                const int j00 = 3 * (y0 * nx + x0) + c;
                const int j01 = 3 * (y0 * nx + x1) + c;
                const int j10 = 3 * (y1 * nx + x0) + c;
                const int j11 = 3 * (y1 * nx + x1) + c;

                // 获取插值点周围四个像素的值
                const float v00 = temp->data[j00];
                const float v01 = temp->data[j01];
                const float v10 = temp->data[j10];
                const float v11 = temp->data[j11];

                // 进行双线性插值
                const float v0 = v00 * (1.0f - dx) + v01 * dx;
                const float v1 = v10 * (1.0f - dx) + v11 * dx;

                const float v = v0 * (1.0f - dy) + v1 * dy;

                // 将插值结果限制在 0 到 255 之间
                const uint8_t v2 = std::min(std::max(std::round(v), 0.0f), 255.0f);

                // 计算结果 res 中像素的索引
                const int i = 3 * (y * nx3 + x) + c;

                // 对插值结果进行归一化处理，并存入结果 res 中
                res->data[i] = ((float(v2) / 255.0f) - m3[c]) / s3[c];
            }
        }
    }
    // 释放临时变量 temp 的内存空间
    clip_image_u8_free(temp);

    // 返回处理结果
    return true;
// 释放 clip_ctx 结构体指针所指向的内存
void clip_free(clip_ctx * ctx) {
    // 释放 ctx->ctx 指向的内存
    ggml_free(ctx->ctx);
    // 释放 ctx->ctx_gguf 指向的内存
    gguf_free(ctx->ctx_gguf);
    // 释放 ctx 指向的内存
    delete ctx;
}

// 对图像进行编码并返回结果
bool clip_image_encode(const clip_ctx * ctx, const int n_threads, clip_image_f32 * img, float * vec) {
    // 如果 ctx 没有视觉编码器，则打印错误信息并返回 false
    if (!ctx->has_vision_encoder) {
        printf("This gguf file seems to have no vision encoder\n");
        return false;
    }

    // 创建 clip_image_f32_batch 结构体并设置大小和数据
    clip_image_f32_batch imgs{};
    imgs.size = 1;
    imgs.data = img;
    // 调用 clip_image_batch_encode 函数并返回结果
    return clip_image_batch_encode(ctx, n_threads, &imgs, vec);
}

// 对图像批量进行编码并返回结果
bool clip_image_batch_encode(const clip_ctx * ctx, const int n_threads, const clip_image_f32_batch * imgs, float * vec) {
    // 如果 ctx 没有视觉编码器，则打印错误信息并返回 false
    if (!ctx->has_vision_encoder) {
        printf("This gguf file seems to have no vision encoder\n");
        return false;
    }

    // 获取批量大小
    int batch_size = imgs->size;
    // 如果 ctx 有 llava 投影仪，则断言批量大小为 1
    if(ctx->has_llava_projector) {
        GGML_ASSERT(batch_size == 1); // TODO: support multiple images
    }

    // 重置分配缓冲区以清除先前调用的内存
    ggml_allocr_reset(ctx->alloc);

    // 构建推理图
    ggml_cgraph * gf = clip_image_build_graph(ctx, imgs);
    ggml_allocr_alloc_graph(ctx->alloc, gf);

    // 计划推理图的执行
    struct ggml_cplan plan = ggml_graph_plan(gf, n_threads);
    if (plan.work_size > 0) {
        plan.work_data = (uint8_t *)malloc(plan.work_size);
    }

    // 执行推理图
    ggml_graph_compute(gf, &plan);

    // 最后一个节点是嵌入张量
    struct ggml_tensor * embeddings = gf->nodes[gf->n_nodes - 1];

    // 将嵌入张量复制到用户传递的位置
    memcpy(vec, ggml_get_data_f32(embeddings), ggml_nbytes(embeddings));

    // 如果工作大小大于 0，则释放工作数据
    if (plan.work_size > 0) {
        free(plan.work_data);
    }

    return true;
}

// 对模型进行量化
bool clip_model_quantize(const char * fname_inp, const char * fname_out, const int itype) {
    // 初始化类型为 GGML_TYPE_Q4_1
    ggml_type type = GGML_TYPE_Q4_1;

    // 根据输入类型设置不同的量化类型
    switch (itype) {
    case 2:
        type = GGML_TYPE_Q4_0;
        break;
    case 3:
        type = GGML_TYPE_Q4_1;
        break;
    case 6:
        type = GGML_TYPE_Q5_0;
        break;
    // 其他情况保持默认值 GGML_TYPE_Q4_1
    }
    // 根据不同的情况设置类型变量
    case 7:
        type = GGML_TYPE_Q5_1;
        break;
    case 8:
        type = GGML_TYPE_Q8_0;
        break;
    // 默认情况下输出错误信息并返回 false
    default:
        fprintf(stderr, "%s: invalid quantization type %d\n", __func__, itype);
        return false;
    };

    // 加载输入文件的剪辑模型
    auto ctx_clip = clip_model_load(fname_inp, 2);
    // 获取剪辑模型的 GGUF 上下文
    const auto & ctx_src = ctx_clip->ctx_gguf;
    // 获取剪辑模型的上下文
    const auto & ctx_data = ctx_clip->ctx;

    // 初始化一个空的 GGUF 上下文
    auto ctx_out = gguf_init_empty();
    // 设置 GGUF 上下文的键值对
    gguf_set_kv(ctx_out, ctx_src);
    // 设置 GGUF 上下文的 u32 值
    gguf_set_val_u32(ctx_out, "general.quantization_version", GGML_QNT_VERSION);
    gguf_set_val_u32(ctx_out, "general.file_type", itype);

    // 以二进制写模式打开输出文件
    auto fout = std::ofstream(fname_out, std::ios::binary);

    // 获取 GGUF 上下文中张量的数量
    const int n_tensors = gguf_get_n_tensors(ctx_src);

    // 遍历所有张量
    for (int i = 0; i < n_tensors; ++i) {
        // 获取当前张量的名称
        const char * name = gguf_get_tensor_name(ctx_src, i);
        // 获取当前张量的指针
        struct ggml_tensor * cur = ggml_get_tensor(ctx_data, name);
        // 将当前张量添加到 GGUF 上下文中
        gguf_add_tensor(ctx_out, cur);
    }

    // 获取元数据的大小
    const size_t meta_size = gguf_get_meta_size(ctx_out);
    // 将文件指针移动到文件开头，并写入更新后的元数据
    fout.seekp(0, std::ios::beg);
    std::vector<uint8_t> meta(meta_size);
    gguf_get_meta_data(ctx_out, meta.data());
    fout.write((const char *)meta.data(), meta_size);

    // 关闭输出文件
    fout.close();

    // 释放剪辑模型的上下文
    clip_free(ctx_clip);
    // 释放 GGUF 上下文
    gguf_free(ctx_out);
    {
        // 打印原始大小，转换为 MB
        printf("%s: original size  = %8.2f MB\n", __func__, total_size_org / 1024.0 / 1024.0);
        // 打印量化后的大小，转换为 MB
        printf("%s: quantized size  = %8.2f MB\n", __func__, total_size_new / 1024.0 / 1024.0);

        // 计算所有元素的总和
        int64_t sum_all = 0;
        for (size_t i = 0; i < hist_all.size(); ++i) {
            sum_all += hist_all[i];
        }

        // 打印直方图
        printf("%s: hist: ", __func__);
        for (size_t i = 0; i < hist_all.size(); ++i) {
            printf("%5.3f ", hist_all[i] / (float)sum_all);
        }
        printf("\n");
    }

    // 返回 true
    return true;
# 返回视觉模型中的嵌入维度的第一个元素
int clip_n_mmproj_embd(const struct clip_ctx * ctx) {
    return ctx->vision_model.mm_2_b->ne[0];
}

# 返回图像分块的数量
int clip_n_patches(const struct clip_ctx * ctx) {
    # 获取视觉模型的超参数
    auto & params = ctx->vision_model.hparams;
    # 计算图像分块的数量
    return (params.image_size / params.patch_size) * (params.image_size / params.patch_size);
}

# 返回嵌入的字节数
size_t clip_embd_nbytes(const struct clip_ctx * ctx) {
    # 计算嵌入的字节数
    return clip_n_patches(ctx) * clip_n_mmproj_embd(ctx) * sizeof(float);
}
```