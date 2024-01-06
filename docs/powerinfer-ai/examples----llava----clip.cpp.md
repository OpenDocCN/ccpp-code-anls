# `PowerInfer\examples\llava\clip.cpp`

```
// 注意：这是从 clip.cpp 修改而来，仅用于 LLaVA，因此可能仍然存在不必要的遗留物
// 我会逐渐清理和扩展它

#include <cassert>  // 断言库
#include <cmath>  // 数学函数库
#include <cstdlib>  // 标准库
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
// 包含 stb_image 库的头文件

// 定义 CLIP_DEBUG 宏

// 定义一个格式化字符串的函数，使用可变参数列表
static std::string format(const char * fmt, ...) {
    va_list ap; // 定义可变参数列表
    va_list ap2; // 定义另一个可变参数列表
    va_start(ap, fmt); // 初始化可变参数列表
    va_copy(ap2, ap); // 复制可变参数列表
    int size = vsnprintf(NULL, 0, fmt, ap); // 计算格式化后字符串的长度
    GGML_ASSERT(size >= 0 && size < INT_MAX); // NOLINT
    std::vector<char> buf(size + 1); // 创建一个字符数组
    int size2 = vsnprintf(buf.data(), size + 1, fmt, ap2); // 将格式化后的字符串写入字符数组
    GGML_ASSERT(size2 == size); // 断言格式化后的字符串长度与之前计算的长度相等
    va_end(ap2); // 结束第二个可变参数列表
    va_end(ap); // 结束第一个可变参数列表
    return std::string(buf.data(), buf.size()); // 返回格式化后的字符串
}

//
// 定义键的常量
//

// 通用文件类型的键
#define KEY_FTYPE "general.file_type"
// 通用名称的键
#define KEY_NAME "general.name"
// 通用描述的键
#define KEY_DESCRIPTION "general.description"
// 剪辑是否具有文本编码器的键
#define KEY_HAS_TEXT_ENC "clip.has_text_encoder"
// 剪辑是否具有视觉编码器的键
#define KEY_HAS_VIS_ENC "clip.has_vision_encoder"
// 剪辑是否具有LLAVA投影仪的键
#define KEY_HAS_LLAVA_PROJ "clip.has_llava_projector"
// 剪辑是否使用GELU的键
#define KEY_USE_GELU "clip.use_gelu"
// 剪辑的嵌入长度的键
#define KEY_N_EMBD "clip.%s.embedding_length"
// 剪辑的前馈长度的键
#define KEY_N_FF "clip.%s.feed_forward_length"
// 剪辑的块数量的键
#define KEY_N_BLOCK "clip.%s.block_count"
// 剪辑的注意力头数量的键
#define KEY_N_HEAD "clip.%s.attention.head_count"
// 剪辑的注意力层归一化epsilon的键
#define KEY_LAYER_NORM_EPS "clip.%s.attention.layer_norm_epsilon"
// 剪辑的投影维度的键
#define KEY_PROJ_DIM "clip.%s.projection_dim"
// 分词器的tokens的键
#define KEY_TOKENS "tokenizer.ggml.tokens"
// 文本上下文长度的键
#define KEY_N_POSITIONS "clip.text.context_length"
// 视觉图像大小的键
#define KEY_IMAGE_SIZE "clip.vision.image_size"
// 视觉图像patch大小的键
#define KEY_PATCH_SIZE "clip.vision.patch_size"
// 定义图像均值的键名常量
#define KEY_IMAGE_MEAN "clip.vision.image_mean"
// 定义图像标准差的键名常量
#define KEY_IMAGE_STD "clip.vision.image_std"

//
// 张量名称常量
//

// 令牌嵌入权重的张量名称模板
#define TN_TOKEN_EMBD "%s.token_embd.weight"
// 位置嵌入权重的张量名称模板
#define TN_POS_EMBD "%s.position_embd.weight"
// 类别嵌入的张量名称
#define TN_CLASS_EMBD "v.class_embd"
// 补丁嵌入权重的张量名称
#define TN_PATCH_EMBD "v.patch_embd.weight"
// 注意力机制中的键、查询、值和输出张量名称模板
#define TN_ATTN_K "%s.blk.%d.attn_k.%s"
#define TN_ATTN_Q "%s.blk.%d.attn_q.%s"
#define TN_ATTN_V "%s.blk.%d.attn_v.%s"
#define TN_ATTN_OUTPUT "%s.blk.%d.attn_out.%s"
// 前馈神经网络中的下采样和上采样张量名称模板
#define TN_FFN_DOWN "%s.blk.%d.ffn_down.%s"
#define TN_FFN_UP "%s.blk.%d.ffn_up.%s"
// 第一层和第二层层归一化的张量名称模板
#define TN_LN_1 "%s.blk.%d.ln1.%s"
#define TN_LN_2 "%s.blk.%d.ln2.%s"
// 预层归一化的张量名称模板
#define TN_LN_PRE "%s.pre_ln.%s"
// 定义了三个宏，分别用于拼接字符串
#define TN_LN_POST "%s.post_ln.%s"
#define TN_TEXT_PROJ "text_projection.weight"
#define TN_VIS_PROJ "visual_projection.weight"
#define TN_LLAVA_PROJ "mm.%d.%s"

//
// 从 gguf 文件中获取数据的工具函数
//

// 根据键名获取在 gguf 上下文中的索引
static int get_key_idx(const gguf_context * ctx, const char * key) {
    // 调用 gguf_find_key 函数查找键名对应的索引
    int i = gguf_find_key(ctx, key);
    // 如果找不到对应的索引，输出错误信息并抛出异常
    if (i == -1) {
        fprintf(stderr, "key %s not found in file\n", key);
        throw std::runtime_error(format("Missing required key: %s", key));
    }
    // 返回找到的索引
    return i;
}

// 根据键名获取 gguf 上下文中的无符号 32 位整数数据
static uint32_t get_u32(const gguf_context * ctx, const std::string & key) {
// 获取指定键的索引
const int i = get_key_idx(ctx, key.c_str());

// 返回指定键对应的无符号32位整数值
return gguf_get_val_u32(ctx, i);
}

// 获取指定键对应的32位浮点数值
static float get_f32(const gguf_context * ctx, const std::string & key) {
    // 获取指定键的索引
    const int i = get_key_idx(ctx, key.c_str());

    // 返回指定键对应的32位浮点数值
    return gguf_get_val_f32(ctx, i);
}

// 获取指定名称的张量
static struct ggml_tensor * get_tensor(struct ggml_context * ctx, const std::string & name) {
    // 获取指定名称的张量
    struct ggml_tensor * cur = ggml_get_tensor(ctx, name.c_str());
    // 如果未找到张量，则抛出运行时错误
    if (!cur) {
        throw std::runtime_error(format("%s: unable to find tensor %s\n", __func__, name.c_str()));
    }

    // 返回找到的张量
    return cur;
}
// 根据文件类型返回对应的字符串表示
static std::string get_ftype(int ftype) {
    // 根据文件类型进行不同的处理
    switch (ftype) {
    case 0:
        return "f32"; // 如果文件类型为0，返回"f32"
    case 1:
        return "f16"; // 如果文件类型为1，返回"f16"
    case 2:
        return "q4_0"; // 如果文件类型为2，返回"q4_0"
    case 3:
        return "q4_1"; // 如果文件类型为3，返回"q4_1"
    case 6:
        return "q5_0"; // 如果文件类型为6，返回"q5_0"
    case 7:
        return "q5_1"; // 如果文件类型为7，返回"q5_1"
    case 8:
        return "q8_0"; // 如果文件类型为8，返回"q8_0"
    default:
        // 如果文件类型不在以上范围内，抛出运行时错误，提示文件类型未被识别
        throw std::runtime_error(format("%s: Unrecognized file type: %d\n", __func__, ftype));
    }
}
// 定义剪辑层结构体

struct clip_layer {
    // 定义注意力机制的权重和偏置
    struct ggml_tensor * k_w;
    struct ggml_tensor * k_b;
    struct ggml_tensor * q_w;
    struct ggml_tensor * q_b;
    struct ggml_tensor * v_w;
    struct ggml_tensor * v_b;

    // 定义输出层的权重和偏置
    struct ggml_tensor * o_w;
    struct ggml_tensor * o_b;

    // 定义层归一化 1 的权重和偏置
    struct ggml_tensor * ln_1_w;
    struct ggml_tensor * ln_1_b;
// 定义神经网络中的权重和偏置张量
struct ggml_tensor * ff_i_w; // 输入层到隐藏层的权重张量
struct ggml_tensor * ff_i_b; // 输入层到隐藏层的偏置张量

struct ggml_tensor * ff_o_w; // 隐藏层到输出层的权重张量
struct ggml_tensor * ff_o_b; // 隐藏层到输出层的偏置张量

// 定义神经网络中的 LayerNorm 层的权重和偏置张量
struct ggml_tensor * ln_2_w; // LayerNorm 层的权重张量
struct ggml_tensor * ln_2_b; // LayerNorm 层的偏置张量

// 定义视觉模型的结构
struct clip_vision_model {
    struct clip_vision_hparams hparams; // 视觉模型的超参数

    // 定义视觉模型中的嵌入张量
    struct ggml_tensor * class_embedding; // 类别嵌入张量
    struct ggml_tensor * patch_embeddings; // 图像块嵌入张量
    struct ggml_tensor * position_embeddings; // 位置嵌入张量
// 定义指向 ggml_tensor 结构体的指针 pre_ln_w
struct ggml_tensor * pre_ln_w;
// 定义指向 ggml_tensor 结构体的指针 pre_ln_b
struct ggml_tensor * pre_ln_b;

// 定义一个名为 layers 的 clip_layer 结构体的向量
std::vector<clip_layer> layers;

// 定义指向 ggml_tensor 结构体的指针 post_ln_w
struct ggml_tensor * post_ln_w;
// 定义指向 ggml_tensor 结构体的指针 post_ln_b

// 定义指向 ggml_tensor 结构体的指针 post_ln_b
struct ggml_tensor * post_ln_b;

// 定义指向 ggml_tensor 结构体的指针 projection
struct ggml_tensor * projection;

// 定义指向 ggml_tensor 结构体的指针 mm_0_w
// 定义指向 ggml_tensor 结构体的指针 mm_0_b
// 定义指向 ggml_tensor 结构体的指针 mm_2_w
// 定义指向 ggml_tensor 结构体的指针 mm_2_b

// 结构体的结束标记
};

// 用于替代 std::vector<uint8_t> 的结构体，不需要进行零初始化
struct clip_buffer {
    // 声明一个指向 uint8_t 类型的指针，并初始化为 NULL
    uint8_t * data = NULL;
    // 声明一个 size_t 类型的变量 size，并初始化为 0
    size_t size = 0;

    // 定义一个 resize 方法，用于重新分配内存空间
    void resize(size_t size) {
        // 释放原有的内存空间
        delete[] data;
        // 分配新的内存空间，并将指针指向新的内存地址
        data = new uint8_t[size];
        // 更新 size 变量的值
        this->size = size;
    }

    // 定义 clip_buffer 结构的析构函数，用于释放内存空间
    ~clip_buffer() { delete[] data; }
};

// 定义 clip_ctx 结构
struct clip_ctx {
    // 声明并初始化布尔类型的变量
    bool has_text_encoder = false;
    bool has_vision_encoder = false;
    bool has_llava_projector = false;
    // 声明 clip_vision_model 类型的变量
    struct clip_vision_model vision_model;
    // 声明并初始化浮点类型的数组
    float image_mean[3];
    float image_std[3];
    // 声明并初始化布尔类型的变量
    bool use_gelu = false;
```
以上是对给定代码的注释解释。
    // 定义一个名为 ftype 的 32 位整数变量，并赋值为 1
    int32_t ftype = 1;
    // 定义指向 ggml_context 结构体的指针变量 ctx
    struct ggml_context * ctx;
    // 定义指向 gguf_context 结构体的指针变量 ctx_gguf

    // 用于评估模型的内存缓冲区
    clip_buffer buf_compute;
    // 用于分配内存的内存缓冲区
    clip_buffer buf_alloc;
    // 指向 ggml_allocr 结构体的指针变量 alloc，初始值为 NULL
    ggml_allocr * alloc = NULL;
};

// 构建图形的函数，接受 clip_ctx 和 clip_image_f32_batch 作为参数
static ggml_cgraph * clip_image_build_graph(const clip_ctx * ctx, const clip_image_f32_batch * imgs) {
    // 如果 ctx 没有视觉编码器，则打印错误信息并返回空指针
    if (!ctx->has_vision_encoder) {
        printf("This gguf file seems to have no vision encoder\n");
        return nullptr;
    }

    // 获取视觉模型和超参数
    const auto & model = ctx->vision_model;
    const auto & hparams = model.hparams;

    // 获取图像大小
    const int image_size = hparams.image_size;
    # 定义补丁大小为模型超参数中的补丁大小
    const int patch_size = hparams.patch_size;
    # 计算图像中的补丁数量
    const int num_patches = ((image_size / patch_size) * (image_size / patch_size));
    # 计算位置编码的数量，为补丁数量加1
    const int num_positions = num_patches + 1;
    # 定义隐藏层大小为模型超参数中的隐藏层大小
    const int hidden_size = hparams.hidden_size;
    # 定义头数为模型超参数中的头数
    const int n_head = hparams.n_head;
    # 计算每个头的维度大小
    const int d_head = hidden_size / n_head;
    # 定义层数为模型超参数中的层数
    const int n_layer = hparams.n_layer;
    # 定义 epsilon 为模型超参数中的 epsilon
    const float eps = hparams.eps;
    # 初始化批处理大小为图像集合的大小
    int batch_size = imgs->size;
    # 如果上下文中有 LLAVA 投影器，则断言批处理大小为1
    if(ctx->has_llava_projector) {
        GGML_ASSERT(batch_size == 1);
    }

    # 获取计算缓冲区的引用
    const auto & buf_compute = ctx->buf_compute;

    # 定义初始化参数结构体
    struct ggml_init_params params = {
        /*.mem_size =*/ buf_compute.size,  # 内存大小为计算缓冲区的大小
        /*.mem_buffer =*/ buf_compute.data,  # 内存缓冲区为计算缓冲区的数据
    /*.no_alloc =*/ false,  // 设置参数 no_alloc 为 false

    params.no_alloc = true;  // 将参数 no_alloc 设置为 true

    struct ggml_context * ctx0 = ggml_init(params);  // 初始化上下文 ctx0，并使用参数 params
    struct ggml_cgraph * gf = ggml_new_graph(ctx0);  // 创建新的计算图 gf，并使用上下文 ctx0

    struct ggml_tensor * inp_raw = ggml_new_tensor_4d(ctx0, GGML_TYPE_F32, image_size, image_size, 3, batch_size);  // 创建一个 4 维张量 inp_raw，使用上下文 ctx0，数据类型为 GGML_TYPE_F32，大小为 image_size * image_size * 3 * batch_size
    ggml_allocr_alloc(ctx->alloc, inp_raw);  // 分配 inp_raw 的内存空间

    if (!ggml_allocr_is_measure(ctx->alloc)) {  // 如果内存分配器不是测量模式
        float * data = (float *)ggml_get_data(inp_raw);  // 获取 inp_raw 的数据，并转换为 float 类型的指针

        for (size_t i = 0; i < imgs->size; i++) {  // 遍历 imgs 的大小
            const int nx = imgs->data[i].nx;  // 获取当前图片的宽度
            const int ny = imgs->data[i].ny;  // 获取当前图片的高度
            GGML_ASSERT(nx == image_size && ny == image_size);  // 断言当前图片的宽度和高度与预设的 image_size 相等

            const int n = nx * ny;  // 计算当前图片的像素总数
// 循环遍历每个批次中的数据
for (int b = 0; b < batch_size; b++) {
    // 循环遍历每个通道
    for (int k = 0; k < 3; k++) {
        // 循环遍历图像的高度
        for (int y = 0; y < ny; y++) {
            // 循环遍历图像的宽度
            for (int x = 0; x < nx; x++) {
                // 将图像数据重新排列成一维数组
                data[(b * 3 * n) + k * n + y * nx + x] = imgs->data[b].data[3 * (y * nx + x) + k];
            }
        }
    }
}
// 调整输入数据的形状
struct ggml_tensor * inp = ggml_conv_2d(ctx0, model.patch_embeddings, inp_raw, patch_size, patch_size, 0, 0, 1, 1);
inp = ggml_reshape_3d(ctx0, inp, num_patches, hidden_size, batch_size);
inp = ggml_cont(ctx0, ggml_permute(ctx0, inp, 1, 0, 2, 3));

// 创建一个新的张量用于存储嵌入向量
struct ggml_tensor * embeddings = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, hidden_size, num_positions, batch_size);
# 使用ctx->alloc分配内存给embeddings
ggml_allocr_alloc(ctx->alloc, embeddings);
# 如果ctx->alloc不是度量，则将embeddings设置为零
if (!ggml_allocr_is_measure(ctx->alloc)) {
    ggml_set_zero(embeddings);
}

# 创建一个3维的新的ggml_tensor结构体temp
struct ggml_tensor * temp = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, hidden_size, 1, batch_size);
# 使用ctx->alloc分配内存给temp
ggml_allocr_alloc(ctx->alloc, temp);

# 使用ggml_acc函数对embeddings进行操作
embeddings = ggml_acc(ctx0, embeddings, ggml_repeat(ctx0, model.class_embedding, temp), embeddings->nb[1],
                      embeddings->nb[2], embeddings->nb[3], 0);
# 使用ggml_acc函数对embeddings进行操作
embeddings =
    ggml_acc(ctx0, embeddings, inp, embeddings->nb[1], embeddings->nb[2], embeddings->nb[3], model.class_embedding->nb[1]);

# 创建一个1维的新的ggml_tensor结构体positions
struct ggml_tensor * positions = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, num_positions);
# 使用ctx->alloc分配内存给positions
ggml_allocr_alloc(ctx->alloc, positions);
# 如果ctx->alloc不是度量，则将positions的值设置为i
if (!ggml_allocr_is_measure(ctx->alloc)) {
    for (int i = 0; i < num_positions; i++) {
        ggml_set_i32_1d(positions, i, i);
    }
}
// 计算嵌入向量的加法
embeddings = ggml_add(ctx0, embeddings, ggml_repeat(ctx0, ggml_get_rows(ctx0, model.position_embeddings, positions), embeddings));

// 执行预层归一化
{
    // 对嵌入向量进行归一化处理
    embeddings = ggml_norm(ctx0, embeddings, eps);

    // 对归一化后的嵌入向量进行加权和偏置处理
    embeddings = ggml_add(ctx0, ggml_mul(ctx0, ggml_repeat(ctx0, model.pre_ln_w, embeddings), embeddings),
                          ggml_repeat(ctx0, model.pre_ln_b, embeddings));
}

// 创建一个一维张量 KQ_scale
struct ggml_tensor * KQ_scale = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
// 分配内存给 KQ_scale
ggml_allocr_alloc(ctx->alloc, KQ_scale);
// 如果不是测量模式，则设置 KQ_scale 的值为 1.0f 除以 sqrt((float)d_head)
if (!ggml_allocr_is_measure(ctx->alloc)) {
    ggml_set_f32(KQ_scale, 1.0f / sqrt((float)d_head));
}

// 循环遍历每一层
for (int il = 0; il < n_layer - 1; il++) {
        // 将指针 cur 指向 embeddings 结构体，即 cur 指向隐藏状态
        struct ggml_tensor * cur = embeddings; // embeddings = residual, cur = hidden_states

        //const size_t nb_q_w = model.layers[il].q_w->nb[0];

        // 对当前隐藏状态进行 layernorm 处理
        {
            // 对 cur 进行 layernorm 处理，eps 为参数
            cur = ggml_norm(ctx0, cur, eps);

            // 对 cur 进行加法和乘法操作，得到新的 cur
            cur = ggml_add(ctx0, ggml_mul(ctx0, ggml_repeat(ctx0, model.layers[il].ln_1_w, cur), cur),
                           ggml_repeat(ctx0, model.layers[il].ln_1_b, cur));
        }

        // 进行自注意力操作
        {

            // 创建新的结构体 Q，通过加法和矩阵乘法得到
            struct ggml_tensor * Q =
                ggml_add(ctx0, ggml_repeat(ctx0, model.layers[il].q_b, cur), ggml_mul_mat(ctx0, model.layers[il].q_w, cur));

            // 对 Q 进行缩放操作
            Q = ggml_scale_inplace(ctx0, Q, KQ_scale);
            // 对 Q 进行形状重塑，得到新的 Q
            Q = ggml_reshape_4d(ctx0, Q, d_head, n_head, num_positions, batch_size);
# 使用 ggml_cont 函数对输入的 Q 进行上下文处理
Q = ggml_cont(ctx0, ggml_permute(ctx0, Q, 0, 2, 1, 3));
# 使用 ggml_reshape_3d 函数对 Q 进行三维重塑
Q = ggml_reshape_3d(ctx0, Q, d_head, num_positions, n_head * batch_size);

# 创建 K 张量，通过 ggml_repeat 和 ggml_mul_mat 函数对 model.layers[il].k_b 和 model.layers[il].k_w 进行操作
struct ggml_tensor * K =
    ggml_add(ctx0, ggml_repeat(ctx0, model.layers[il].k_b, cur), ggml_mul_mat(ctx0, model.layers[il].k_w, cur));
# 使用 ggml_reshape_4d 函数对 K 进行四维重塑
K = ggml_reshape_4d(ctx0, K, d_head, n_head, num_positions, batch_size);
# 使用 ggml_cont 函数对 K 进行上下文处理
K = ggml_cont(ctx0, ggml_permute(ctx0, K, 0, 2, 1, 3));
# 使用 ggml_reshape_3d 函数对 K 进行三维重塑
K = ggml_reshape_3d(ctx0, K, d_head, num_positions, n_head * batch_size);

# 创建 V 张量，通过 ggml_repeat 和 ggml_mul_mat 函数对 model.layers[il].v_b 和 model.layers[il].v_w 进行操作
struct ggml_tensor * V =
    ggml_add(ctx0, ggml_repeat(ctx0, model.layers[il].v_b, cur), ggml_mul_mat(ctx0, model.layers[il].v_w, cur));
# 使用 ggml_reshape_4d 函数对 V 进行四维重塑
V = ggml_reshape_4d(ctx0, V, d_head, n_head, num_positions, batch_size);
# 使用 ggml_cont 函数对 V 进行上下文处理
V = ggml_cont(ctx0, ggml_permute(ctx0, V, 1, 2, 0, 3));
# 使用 ggml_reshape_3d 函数对 V 进行三维重塑
V = ggml_reshape_3d(ctx0, V, num_positions, d_head, n_head * batch_size);

# 创建 KQ 张量，通过 ggml_mul_mat 函数对 K 和 Q 进行矩阵相乘
struct ggml_tensor * KQ = ggml_mul_mat(ctx0, K, Q);
# 使用 ggml_soft_max_inplace 函数对 KQ 进行 softmax 操作
KQ = ggml_soft_max_inplace(ctx0, KQ);
# 创建 KQV 张量，通过 ggml_mul_mat 函数对 V 和 KQ 进行矩阵相乘
struct ggml_tensor * KQV = ggml_mul_mat(ctx0, V, KQ);
// 对 KQV 进行形状重塑，将其转换为 4 维张量
KQV = ggml_reshape_4d(ctx0, KQV, d_head, num_positions, n_head, batch_size);
// 对 KQV 进行维度置换
KQV = ggml_cont(ctx0, ggml_permute(ctx0, KQV, 0, 2, 1, 3));

// 复制 KQV 到 cur，创建一个新的 3 维张量
cur = ggml_cpy(ctx0, KQV, ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, hidden_size, num_positions, batch_size));

// 注意力输出
// 将当前值与模型层的偏置相加，然后再与模型层的权重相乘
cur = ggml_add(ctx0, ggml_repeat(ctx0, model.layers[il].o_b, cur), ggml_mul_mat(ctx0, model.layers[il].o_w, cur));

// 重新添加层输入，例如残差
cur = ggml_add(ctx0, cur, embeddings);

// 将当前值赋给 embeddings，表示残差，cur 表示隐藏状态
embeddings = cur;

// 层归一化
// 对当前值进行归一化处理
cur = ggml_norm(ctx0, cur, eps);
// 将当前值与模型层的第二层归一化权重相乘，再与当前值相加
cur = ggml_add(ctx0, ggml_mul(ctx0, ggml_repeat(ctx0, model.layers[il].ln_2_w, cur), cur),
               ggml_repeat(ctx0, model.layers[il].ln_2_b, cur));
        }

        // 使用上下文和模型参数进行矩阵乘法运算
        cur = ggml_mul_mat(ctx0, model.layers[il].ff_i_w, cur);
        // 使用上下文和模型参数进行加法运算
        cur = ggml_add(ctx0, ggml_repeat(ctx0, model.layers[il].ff_i_b, cur), cur);

        // 如果上下文使用gelu激活函数，则使用gelu激活函数对当前结果进行操作，否则使用快速gelu激活函数
        if (ctx->use_gelu) {
            cur = ggml_gelu_inplace(ctx0, cur);
        } else {
            cur = ggml_gelu_quick_inplace(ctx0, cur);
        }

        // 使用上下文和模型参数进行矩阵乘法运算
        cur = ggml_mul_mat(ctx0, model.layers[il].ff_o_w, cur);
        // 使用上下文和模型参数进行加法运算
        cur = ggml_add(ctx0, ggml_repeat(ctx0, model.layers[il].ff_o_b, cur), cur);

        // 残差连接2，将当前结果与embeddings相加
        cur = ggml_add(ctx0, embeddings, cur);

        // 更新embeddings为当前结果
        embeddings = cur;
    }
// 将嵌入向量重塑为二维张量
embeddings = ggml_reshape_2d(ctx0, embeddings, embeddings->ne[0], embeddings->ne[1]);

// 创建一个包含 num_patches 个元素的一维张量
struct ggml_tensor * patches = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, num_patches);
ggml_allocr_alloc(ctx->alloc, patches);
if (!ggml_allocr_is_measure(ctx->alloc)) {
    // 如果分配器不是度量器，则为 patches 张量赋值
    for (int i = 0; i < num_patches; ++i) {
        ggml_set_i32_1d(patches, i, i+1);
    }
}

// 从嵌入向量中获取指定行的数据
embeddings = ggml_get_rows(ctx0, embeddings, patches);

// 进行矩阵乘法运算
embeddings = ggml_mul_mat(ctx0, model.mm_0_w, embeddings);

// 将偏置矩阵重复与嵌入向量相加
embeddings = ggml_add(ctx0, ggml_repeat(ctx0, model.mm_0_b, embeddings), embeddings);

// 对嵌入向量进行 GELU 激活函数处理
embeddings = ggml_gelu(ctx0, embeddings);
    // 使用 ggml_mul_mat 函数对 embeddings 进行矩阵乘法运算
    embeddings = ggml_mul_mat(ctx0, model.mm_2_w, embeddings);
    // 使用 ggml_repeat 函数对 model.mm_2_b 进行重复操作，然后与 embeddings 相加
    embeddings = ggml_add(ctx0, ggml_repeat(ctx0, model.mm_2_b, embeddings), embeddings);
    }

    // 构建前向传播图
    ggml_build_forward_expand(gf, embeddings);

    // 释放上下文
    ggml_free(ctx0);

    // 返回前向传播图
    return gf;
}

// 读取并创建包含张量及其数据的 ggml_context 结构
struct clip_ctx * clip_model_load(const char * fname, const int verbosity = 1) {

    // 初始化 meta 为 NULL
    struct ggml_context * meta = NULL;

    // 初始化 gguf_init_params 结构体
    struct gguf_init_params params = {
        /*.no_alloc = */ true,  // 设置 no_alloc 为 true
        /*.ctx      = */ &meta,  // 设置 ctx 指向 meta
```

    };

    // 从文件中初始化 gguf_context 结构体
    struct gguf_context * ctx = gguf_init_from_file(fname, params);
    // 如果初始化失败，抛出运行时错误
    if (!ctx) {
        throw std::runtime_error(format("%s: failed to load CLIP model from %s. Does this file exist?\n", __func__, fname));
    }

    // 如果输出详细信息的级别大于等于1
    if (verbosity >= 1) {
        // 获取张量数量、键值对数量、文件类型、描述和名称等信息
        const int n_tensors = gguf_get_n_tensors(ctx);
        const int n_kv = gguf_get_n_kv(ctx);
        const int ftype = get_u32(ctx, KEY_FTYPE);
        const std::string ftype_str = get_ftype(ftype);
        const int idx_desc = get_key_idx(ctx, KEY_DESCRIPTION);
        const std::string description = gguf_get_val_str(ctx, idx_desc);
        const int idx_name = gguf_find_key(ctx, KEY_NAME);
        // 如果存在名称，打印模型名称
        if (idx_name != -1) { // make name optional temporarily as some of the uploaded models missing it due to a bug
            const std::string name = gguf_get_val_str(ctx, idx_name);
            printf("%s: model name:   %s\n", __func__, name.c_str());
        }
        // 打印模型描述
        printf("%s: description:  %s\n", __func__, description.c_str());
// 打印函数名和 GGUF 版本号
printf("%s: GGUF version: %d\n", __func__, gguf_get_version(ctx));
// 打印函数名和对齐方式
printf("%s: alignment:    %zu\n", __func__, gguf_get_alignment(ctx));
// 打印函数名和张量数量
printf("%s: n_tensors:    %d\n", __func__, n_tensors);
// 打印函数名和键值对数量
printf("%s: n_kv:         %d\n", __func__, n_kv);
// 打印函数名和文件类型
printf("%s: ftype:        %s\n", __func__, ftype_str.c_str());
// 打印空行
printf("\n");
}

// 如果详细程度大于等于 3
if (verbosity >= 3) {
    // 获取键值对数量
    const int n_kv = gguf_get_n_kv(ctx);

    // 遍历键值对
    for (int i = 0; i < n_kv; ++i) {
        // 获取键值对的键
        const char * key = gguf_get_key(ctx, i);

        // 打印函数名、键值对索引和键
        printf("%s: kv[%d]: key = %s\n", __func__, i, key);
    }
    // 打印空行
    printf("\n");
}
// 定义变量 ctx_size，并初始化为 0
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

        // 根据张量名称从 meta 中获取张量的结构
        struct ggml_tensor * cur = ggml_get_tensor(meta, name);
        // 计算当前张量占用的内存大小
        ctx_size += sizeof(struct ggml_tensor) + GGML_OBJECT_SIZE;
        // 获取张量的实际大小
        size_t tensor_size = ggml_nbytes(cur);
        // 获取张量的填充后的大小
        size_t padded_size = ggml_nbytes_pad(cur);
        // 将填充后的大小加入到上下文的大小中
        ctx_size += padded_size;
        // 如果日志级别大于等于 3，则打印张量的信息
        if (verbosity >= 3) {
            printf("%s: tensor[%d]: n_dims = %d, name = %s, tensor_size=%zu, padded_size=%zu, offset=%zu\n", __func__, i,
                   cur->n_dims, cur->name, tensor_size, padded_size, offset);
        }
    }
}
    // 创建一个新的剪贴板上下文对象
    clip_ctx * new_clip = new clip_ctx;

    // 获取模型大小和功能
    {
        // 获取是否有文本编码器的索引，并将结果存入新剪贴板对象的属性中
        int idx = get_key_idx(ctx, KEY_HAS_TEXT_ENC);
        new_clip->has_text_encoder = gguf_get_val_bool(ctx, idx);

        // 获取是否有视觉编码器的索引，并将结果存入新剪贴板对象的属性中
        idx = get_key_idx(ctx, KEY_HAS_VIS_ENC);
        new_clip->has_vision_encoder = gguf_get_val_bool(ctx, idx);

        // 查找是否有 LLAVA 投影仪的索引，如果有则将结果存入新剪贴板对象的属性中
        idx = gguf_find_key(ctx, KEY_HAS_LLAVA_PROJ);
        if (idx != -1) {
            new_clip->has_llava_projector = gguf_get_val_bool(ctx, idx);
        }

        // 断言新剪贴板对象有 LLAVA 投影仪，用于语义搜索的图像和/或文本编码，请参见 monatis/clip.cpp
        GGML_ASSERT(new_clip->has_llava_projector);
        // 断言新剪贴板对象有视觉编码器
        GGML_ASSERT(new_clip->has_vision_encoder);
        // 断言新剪贴板对象没有文本编码器
        GGML_ASSERT(!new_clip->has_text_encoder);
    }
        // 获取 KEY_USE_GELU 的索引
        idx = get_key_idx(ctx, KEY_USE_GELU);
        // 根据索引获取布尔值并赋给 new_clip->use_gelu
        new_clip->use_gelu = gguf_get_val_bool(ctx, idx);

        // 如果verbosity大于等于1，则输出以下信息
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
        // 初始化参数结构体
        struct ggml_init_params params = {
            /*.mem_size =*/ ctx_size,
            /*.mem_buffer =*/ NULL,
            /*.no_alloc =*/ false,
        };
        // 初始化新的剪辑上下文
        new_clip->ctx = ggml_init(params);
        // 如果初始化失败，打印错误信息并释放资源，返回空指针
        if (!new_clip->ctx) {
            fprintf(stderr, "%s: ggml_init() failed\n", __func__);
            clip_free(new_clip);
            return nullptr;
        }

        // 打开模型文件，以二进制方式读取
        auto fin = std::ifstream(fname, std::ios::binary);
        // 如果文件打开失败，打印错误信息，释放资源，返回空指针
        if (!fin) {
            printf("cannot open model file for loading tensors\n");
            clip_free(new_clip);
            return nullptr;
        }

        // 获取上下文中张量的数量
        const int n_tensors = gguf_get_n_tensors(ctx);
        // 遍历张量
        for (int i = 0; i < n_tensors; ++i) {
            // 获取张量的名称
            const char * name = gguf_get_tensor_name(ctx, i);
            // 获取元数据中的张量
            struct ggml_tensor * t = ggml_get_tensor(meta, name);
            // 复制张量到新的剪辑上下文中
            struct ggml_tensor * cur = ggml_dup_tensor(new_clip->ctx, t);
            // 设置张量的名称
            ggml_set_name(cur, name);
// 计算数据偏移量，用于定位特定张量的数据位置
const size_t offset = gguf_get_data_offset(ctx) + gguf_get_tensor_offset(ctx, i);
// 从文件开头偏移指定位置，定位到张量数据的起始位置
fin.seekg(offset, std::ios::beg);
// 如果定位失败，输出错误信息并释放内存，返回空指针
if (!fin) {
    printf("%s: failed to seek for tensor %s\n", __func__, name);
    clip_free(new_clip);
    return nullptr;
}
// 从文件中读取指定长度的数据到当前张量的数据缓冲区
fin.read(reinterpret_cast<char *>(cur->data), ggml_nbytes(t));

// 关闭文件流
fin.close();

// 如果新剪辑包含视觉编码器
if (new_clip->has_vision_encoder) {
    // 加载视觉模型
    auto & vision_model = new_clip->vision_model;
    auto & hparams = vision_model.hparams;
}
        # 设置隐藏层大小为从上下文中获取的名为"vision"的键对应的无符号32位整数值
        hparams.hidden_size = get_u32(ctx, format(KEY_N_EMBD, "vision"));
        # 设置头数为从上下文中获取的名为"vision"的键对应的无符号32位整数值
        hparams.n_head = get_u32(ctx, format(KEY_N_HEAD, "vision"));
        # 设置中间层大小为从上下文中获取的名为"vision"的键对应的无符号32位整数值
        hparams.n_intermediate = get_u32(ctx, format(KEY_N_FF, "vision"));
        # 设置层数为从上下文中获取的名为"vision"的键对应的无符号32位整数值
        hparams.n_layer = get_u32(ctx, format(KEY_N_BLOCK, "vision"));
        # 设置图像大小为从上下文中获取的名为"image_size"的键对应的无符号32位整数值
        hparams.image_size = get_u32(ctx, KEY_IMAGE_SIZE);
        # 设置补丁大小为从上下文中获取的名为"patch_size"的键对应的无符号32位整数值
        hparams.patch_size = get_u32(ctx, KEY_PATCH_SIZE);
        # 设置投影维度为从上下文中获取的名为"vision"的键对应的无符号32位整数值
        hparams.projection_dim = get_u32(ctx, format(KEY_PROJ_DIM, "vision"));
        # 设置eps为从上下文中获取的名为"vision"的键对应的单精度浮点数值
        hparams.eps = get_f32(ctx, format(KEY_LAYER_NORM_EPS, "vision"));

        # 获取图像均值和标准差的键索引
        int idx_mean = get_key_idx(ctx, KEY_IMAGE_MEAN);
        int idx_std = get_key_idx(ctx, KEY_IMAGE_STD);
        # 循环3次，分别将图像均值和标准差从上下文中获取并存储到新的剪辑对象中
        for (int i = 0; i < 3; ++i) {
            new_clip->image_mean[i] = *((const float *)gguf_get_arr_data(ctx, idx_mean));
            new_clip->image_std[i] = *((const float *)gguf_get_arr_data(ctx, idx_std));
        }

        # 如果详细程度大于等于2，打印以下信息
        if (verbosity >= 2) {
            printf("\n%s: vision model hparams\n", __func__);
            printf("image_size         %d\n", hparams.image_size);
            printf("patch_size         %d\n", hparams.patch_size);
// 打印隐藏层大小
printf("v_hidden_size      %d\n", hparams.hidden_size);
// 打印中间层数量
printf("v_n_intermediate   %d\n", hparams.n_intermediate);
// 打印投影维度
printf("v_projection_dim   %d\n", hparams.projection_dim);
// 打印头部数量
printf("v_n_head           %d\n", hparams.n_head);
// 打印层数量
printf("v_n_layer          %d\n", hparams.n_layer);

// 设置视觉模型的补丁嵌入
vision_model.patch_embeddings = get_tensor(new_clip->ctx, TN_PATCH_EMBD);
// 设置视觉模型的类别嵌入
vision_model.class_embedding = get_tensor(new_clip->ctx, TN_CLASS_EMBD);
// 设置视觉模型的位置嵌入
vision_model.position_embeddings = get_tensor(new_clip->ctx, format(TN_POS_EMBD, "v"));
// 设置视觉模型的预层归一化权重
vision_model.pre_ln_w = get_tensor(new_clip->ctx, format(TN_LN_PRE, "v", "weight"));
// 设置视觉模型的预层归一化偏置
vision_model.pre_ln_b = get_tensor(new_clip->ctx, format(TN_LN_PRE, "v", "bias"));
// 设置视觉模型的 mm_0 权重
vision_model.mm_0_w = get_tensor(new_clip->ctx, format(TN_LLAVA_PROJ, 0, "weight"));
// 设置视觉模型的 mm_0 偏置
vision_model.mm_0_b = get_tensor(new_clip->ctx, format(TN_LLAVA_PROJ, 0, "bias"));
// 设置视觉模型的 mm_2 权重
vision_model.mm_2_w = get_tensor(new_clip->ctx, format(TN_LLAVA_PROJ, 2, "weight"));
// 设置视觉模型的 mm_2 偏置
vision_model.mm_2_b = get_tensor(new_clip->ctx, format(TN_LLAVA_PROJ, 2, "bias"));

// 调整视觉模型的层数
vision_model.layers.resize(hparams.n_layer);
for (int il = 0; il < hparams.n_layer; ++il) {
    // 获取当前层的引用
    auto & layer = vision_model.layers[il];
# 为 layer 对象的各个属性赋值，使用 get_tensor 函数获取对应的张量数据
layer.k_w = get_tensor(new_clip->ctx, format(TN_ATTN_K, "v", il, "weight"));
layer.q_w = get_tensor(new_clip->ctx, format(TN_ATTN_Q, "v", il, "weight"));
layer.v_w = get_tensor(new_clip->ctx, format(TN_ATTN_V, "v", il, "weight"));
layer.o_w = get_tensor(new_clip->ctx, format(TN_ATTN_OUTPUT, "v", il, "weight"));
layer.ln_1_w = get_tensor(new_clip->ctx, format(TN_LN_1, "v", il, "weight"));
layer.ln_2_w = get_tensor(new_clip->ctx, format(TN_LN_2, "v", il, "weight"));
layer.ff_i_w = get_tensor(new_clip->ctx, format(TN_FFN_DOWN, "v", il, "weight"));
layer.ff_o_w = get_tensor(new_clip->ctx, format(TN_FFN_UP, "v", il, "weight"));
# 为 layer 对象的各个属性赋值，使用 get_tensor 函数获取对应的张量数据
layer.k_b = get_tensor(new_clip->ctx, format(TN_ATTN_K, "v", il, "bias"));
layer.q_b = get_tensor(new_clip->ctx, format(TN_ATTN_Q, "v", il, "bias"));
layer.v_b = get_tensor(new_clip->ctx, format(TN_ATTN_V, "v", il, "bias"));
layer.o_b = get_tensor(new_clip->ctx, format(TN_ATTN_OUTPUT, "v", il, "bias"));
layer.ln_1_b = get_tensor(new_clip->ctx, format(TN_LN_1, "v", il, "bias"));
layer.ln_2_b = get_tensor(new_clip->ctx, format(TN_LN_2, "v", il, "bias"));
layer.ff_i_b = get_tensor(new_clip->ctx, format(TN_FFN_DOWN, "v", il, "bias"));
layer.ff_o_b = get_tensor(new_clip->ctx, format(TN_FFN_UP, "v", il, "bias"));
# 释放 meta 对象占用的内存
ggml_free(meta);
    // 将新的上下文赋值给新的剪辑对象
    new_clip->ctx_gguf = ctx;

    // 测量内存需求并分配
    {
        // 定义张量对齐要求
        static const size_t tensor_alignment = 32;
        // 调整计算缓冲区大小
        new_clip->buf_compute.resize(ggml_tensor_overhead()*GGML_DEFAULT_GRAPH_SIZE + ggml_graph_overhead());
        // 创建内存分配器对象
        new_clip->alloc = ggml_allocr_new_measure(tensor_alignment);
        // 创建图像批次对象
        clip_image_f32_batch batch;
        batch.size = 1;
        // 构建图形对象
        ggml_cgraph * gf = clip_image_build_graph(new_clip, &batch);
        // 计算分配大小
        size_t alloc_size = ggml_allocr_alloc_graph(new_clip->alloc, gf) + tensor_alignment;
        // 释放内存分配器对象
        ggml_allocr_free(new_clip->alloc);
        // 调整分配缓冲区大小
        new_clip->buf_alloc.resize(alloc_size);
        // 创建新的内存分配器对象
        new_clip->alloc = ggml_allocr_new(new_clip->buf_alloc.data, new_clip->buf_alloc.size, tensor_alignment);

        // 打印总分配内存大小
        printf("%s: total allocated memory: %.2f MB\n", __func__, (new_clip->buf_compute.size + alloc_size)/1024.0/1024.0);
    }

    // 返回新的剪辑对象
    return new_clip;
// 创建一个 clip_image_u8 类型的指针，并返回
clip_image_u8 * make_clip_image_u8() {
    auto img = new clip_image_u8(); // 创建一个 clip_image_u8 类型的对象
    return img; // 返回该对象的指针
}
// 创建一个 clip_image_f32 类型的指针，并返回
clip_image_f32 * make_clip_image_f32() { return new clip_image_f32(); }

// 释放 clip_image_u8 对象占用的内存
void clip_image_u8_free(clip_image_u8 * img) { 
    if (img->data) { delete[] img->data; } // 如果对象的数据存在，则释放内存
    delete img; // 释放对象本身的内存
}
// 释放 clip_image_f32 对象占用的内存
void clip_image_f32_free(clip_image_f32 * img) { 
    if (img->data) { delete[] img->data; } // 如果对象的数据存在，则释放内存
    delete img; // 释放对象本身的内存
}

// 从给定的数据构建一个 clip_image_u8 对象
static void build_clip_img_from_data(const stbi_uc * data, int nx, int ny, clip_image_u8 * img) {
    img->nx = nx; // 设置对象的宽度
    img->ny = ny; // 设置对象的高度
    img->size = nx * ny * 3; // 计算对象数据的大小
    img->data = new uint8_t[img->size](); // 分配内存并初始化为0
    memcpy(img->data, data, img->size); // 将给定数据复制到对象的数据中
}

// 从文件加载数据到 clip_image_u8 对象
bool clip_image_load_from_file(const char * fname, clip_image_u8 * img) {
    # 定义变量nx, ny, nc，用于存储图像的宽度、高度和通道数
    int nx, ny, nc;
    # 使用stbi_load函数从文件中加载图像数据，并将图像的宽度、高度和通道数存储到nx, ny, nc中
    auto data = stbi_load(fname, &nx, &ny, &nc, 3);
    # 如果加载失败，则输出错误信息并返回false
    if (!data) {
        fprintf(stderr, "%s: failed to load image '%s'\n", __func__, fname);
        return false;
    }
    # 根据加载的图像数据构建图像剪裁对象
    build_clip_img_from_data(data, nx, ny, img);
    # 释放图像数据占用的内存
    stbi_image_free(data);
    # 返回true表示加载成功
    return true;
}

# 从内存中加载图像数据
bool clip_image_load_from_bytes(const unsigned char * bytes, size_t bytes_length, struct clip_image_u8 * img) {
    int nx, ny, nc;
    # 使用stbi_load_from_memory函数从内存中加载图像数据，并将图像的宽度、高度和通道数存储到nx, ny, nc中
    auto data = stbi_load_from_memory(bytes, bytes_length, &nx, &ny, &nc, 3);
    # 如果加载失败，则输出错误信息并返回false
    if (!data) {
        fprintf(stderr, "%s: failed to decode image bytes\n", __func__);
        return false;
    }
    # 根据加载的图像数据构建图像剪裁对象
    build_clip_img_from_data(data, nx, ny, img);
    # 释放图像数据占用的内存
    stbi_image_free(data);
    // 返回 true
    return true;
}

// 归一化：x = (x - mean) / std
// TODO: 实现双三次插值，而不是线性插值。
bool clip_image_preprocess(const clip_ctx * ctx, const clip_image_u8 * img, clip_image_f32 * res, const bool pad2square) {
    // 如果上下文中没有视觉编码器，则打印错误信息并返回 false
    if (!ctx->has_vision_encoder) {
        printf("This gguf file seems to have no vision encoder\n");
        return false;
    }

    // 以下逻辑是将较短的一边填充到较长的一边，使用背景颜色：rgb(122, 116, 104)
    // 参考 https://github.com/haotian-liu/LLaVA/blob/e854a2bf85118c504f6f16bf5c3c7c92f8fa8c6b/llava/conversation.py#L113-L156

    // 创建临时图像数据对象，用于暂时保存输入图像数据
    clip_image_u8 * temp = make_clip_image_u8(); 
    // 如果需要填充成正方形，并且图像的宽高不相等
    if (pad2square && img->nx != img->ny) {
        // 计算较长的一边
        int longer_side = std::max(img->nx, img->ny);
        // 设置临时图像对象的宽高
        temp->nx = longer_side;
        temp->ny = longer_side;
        // 计算临时图像对象的大小
        temp->size = 3 * longer_side * longer_side;
        // 为temp->data分配内存，大小为temp->size，并初始化为0
        temp->data = new uint8_t[temp->size]();
        // 定义RGB格式的背景颜色数组
        uint8_t bc[3] = {122, 116, 104}; // bakground color in RGB from LLaVA

        // 使用背景颜色填充temp->data
        for (size_t i = 0; i < temp->size; i++) {
            temp->data[i] = bc[i % 3];
        }

        // 从输入图像复制数据到temp->data
        for (int y = 0; y < img->ny; y++) {
            for (int x = 0; x < img->nx; x++) {
                // 计算输入图像和temp图像中的索引
                const int i = 3 * (y * img->nx + x);
                const int j = 3 * (y * temp->nx + x);
                // 复制RGB数据
                temp->data[j] = img->data[i];
                temp->data[j+1] = img->data[i+1];
                temp->data[j+2] = img->data[i+2];
            }
        }
    } else {
        // 如果条件不满足，设置temp->nx为img->nx
        temp->nx   = img->nx;
    // 将临时对象的ny属性设置为img对象的ny属性
    temp->ny   = img->ny;
    // 将临时对象的size属性设置为img对象的size属性
    temp->size = img->size;
    // 为临时对象的data属性分配内存空间，并初始化为0
    temp->data = new uint8_t[temp->size]();
    // 将img对象的data属性复制到临时对象的data属性中
    memcpy(&temp->data[0], &img->data[0], temp->size); // copy

    // 获取临时对象的nx属性
    const int nx = temp->nx;
    // 获取临时对象的ny属性
    const int ny = temp->ny;

    // 获取ctx对象中vision_model属性的image_size属性，并赋值给nx2和ny2
    const int nx2 = ctx->vision_model.hparams.image_size;
    const int ny2 = ctx->vision_model.hparams.image_size;

    // 将res对象的nx属性设置为nx2
    res->nx = nx2;
    // 将res对象的ny属性设置为ny2
    res->ny = ny2;
    // 将res对象的size属性设置为3 * nx2 * ny2
    res->size = 3 * nx2 * ny2;
    // 为res对象的data属性分配内存空间，并初始化为0
    res->data = new float[res->size]();

    // 计算缩放比例，取nx和ny中的最大值除以ctx对象中vision_model属性的image_size属性
    const float scale = std::max(nx, ny) / (float)ctx->vision_model.hparams.image_size;

    // 将nx除以缩放比例再加上0.5后取整，赋值给nx3
    const int nx3 = int(nx / scale + 0.5f);
    // 计算缩放后的图像高度
    const int ny3 = int(ny / scale + 0.5f);

    // 获取图像的均值
    const auto & m3 = ctx->image_mean; // {0.48145466f, 0.4578275f, 0.40821073f};
    // 获取图像的标准差
    const auto & s3 = ctx->image_std;  // {0.26862954f, 0.26130258f, 0.27577711f};

    // 遍历缩放后的图像像素
    for (int y = 0; y < ny3; y++) {
        for (int x = 0; x < nx3; x++) {
            for (int c = 0; c < 3; c++) {
                // 线性插值
                const float sx = (x + 0.5f) * scale - 0.5f;
                const float sy = (y + 0.5f) * scale - 0.5f;

                // 计算插值点的坐标
                const int x0 = std::max(0, (int)std::floor(sx));
                const int y0 = std::max(0, (int)std::floor(sy));

                // 计算插值点的坐标
                const int x1 = std::min(x0 + 1, nx - 1);
                const int y1 = std::min(y0 + 1, ny - 1);

                // 计算插值点的偏移量
                const float dx = sx - x0;
                const float dy = sy - y0;
// 计算在图像数据中的索引位置
const int j00 = 3 * (y0 * nx + x0) + c; // 计算左上角像素在一维数组中的索引位置
const int j01 = 3 * (y0 * nx + x1) + c; // 计算右上角像素在一维数组中的索引位置
const int j10 = 3 * (y1 * nx + x0) + c; // 计算左下角像素在一维数组中的索引位置
const int j11 = 3 * (y1 * nx + x1) + c; // 计算右下角像素在一维数组中的索引位置

// 获取插值需要的像素值
const float v00 = temp->data[j00]; // 获取左上角像素的值
const float v01 = temp->data[j01]; // 获取右上角像素的值
const float v10 = temp->data[j10]; // 获取左下角像素的值
const float v11 = temp->data[j11]; // 获取右下角像素的值

// 进行双线性插值计算
const float v0 = v00 * (1.0f - dx) + v01 * dx; // 在 x 方向上进行线性插值
const float v1 = v10 * (1.0f - dx) + v11 * dx; // 在 x 方向上进行线性插值
const float v = v0 * (1.0f - dy) + v1 * dy; // 在 y 方向上进行线性插值

// 将插值结果限制在 0 到 255 之间
const uint8_t v2 = std::min(std::max(std::round(v), 0.0f), 255.0f); // 对插值结果进行范围限制

// 计算最终的索引位置
const int i = 3 * (y * nx3 + x) + c; // 计算最终像素在一维数组中的索引位置
// 将计算结果存储到数据结构中
res->data[i] = ((float(v2) / 255.0f) - m3[c]) / s3[c];
// 释放临时变量
}
}
}
clip_image_u8_free(temp);
// 返回操作成功
return true;
}

// 释放内存
void clip_free(clip_ctx * ctx) {
    // 释放内存
    ggml_free(ctx->ctx);
    gguf_free(ctx->ctx_gguf);
    // 释放对象
    delete ctx;
}

// 对图像进行编码
bool clip_image_encode(const clip_ctx * ctx, const int n_threads, clip_image_f32 * img, float * vec) {
    // 如果没有视觉编码器，则打印错误信息并返回失败
    if (!ctx->has_vision_encoder) {
        printf("This gguf file seems to have no vision encoder\n");
        return false;
    }
// 创建一个包含浮点图像数据的批处理对象
clip_image_f32_batch imgs{};
// 设置批处理对象的大小为1
imgs.size = 1;
// 将图像数据赋值给批处理对象的数据属性
imgs.data = img;
// 调用clip_image_batch_encode函数对图像进行编码并返回结果
return clip_image_batch_encode(ctx, n_threads, &imgs, vec);
}

// 对图像批处理对象进行编码
bool clip_image_batch_encode(const clip_ctx * ctx, const int n_threads, const clip_image_f32_batch * imgs, float * vec) {

    // 如果上下文中没有视觉编码器，则打印错误信息并返回false
    if (!ctx->has_vision_encoder) {
        printf("This gguf file seems to have no vision encoder\n");
        return false;
    }

    // 获取批处理对象的大小
    int batch_size = imgs->size;
    // 如果上下文中有llava投影仪，则断言批处理大小为1，否则打印TODO信息
    if(ctx->has_llava_projector) {
        GGML_ASSERT(batch_size == 1); // TODO: support multiple images
    }

    // 重置分配的缓冲区，以清除先前调用的内存
    // 重置分配器状态
    ggml_allocr_reset(ctx->alloc);

    // 构建推理图
    ggml_cgraph * gf = clip_image_build_graph(ctx, imgs);
    // 分配图所需的内存
    ggml_allocr_alloc_graph(ctx->alloc, gf);

    // 计划图的执行
    struct ggml_cplan plan = ggml_graph_plan(gf, n_threads);
    // 如果计划需要工作空间，则分配工作空间
    if (plan.work_size > 0) {
        plan.work_data = (uint8_t *)malloc(plan.work_size);
    }

    // 执行图的计算
    ggml_graph_compute(gf, &plan);

    // 最后一个节点是嵌入张量
    struct ggml_tensor * embeddings = gf->nodes[gf->n_nodes - 1];

    // 将嵌入数据复制到用户传递的位置
    memcpy(vec, ggml_get_data_f32(embeddings), ggml_nbytes(embeddings));

    // 如果计划需要工作空间，则进行清理
    if (plan.work_size > 0) {
    // 释放 plan.work_data 所指向的内存空间
    free(plan.work_data);
    // 返回 true
    return true;
}

// 对模型进行量化处理
bool clip_model_quantize(const char * fname_inp, const char * fname_out, const int itype) {
    // 初始化量化类型为 GGML_TYPE_Q4_1
    ggml_type type = GGML_TYPE_Q4_1;

    // 根据 itype 的值选择不同的量化类型
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
    // 根据不同的情况设置类型
    case 7:
        type = GGML_TYPE_Q5_1;
        break;
    case 8:
        type = GGML_TYPE_Q8_0;
        break;
    default:
        // 打印错误信息并返回false
        fprintf(stderr, "%s: invalid quantization type %d\n", __func__, itype);
        return false;
    };

    // 加载输入文件的剪辑模型
    auto ctx_clip = clip_model_load(fname_inp, 2);
    // 获取剪辑模型的gguf上下文
    const auto & ctx_src = ctx_clip->ctx_gguf;
    // 获取剪辑模型的上下文
    const auto & ctx_data = ctx_clip->ctx;

    // 初始化一个空的gguf上下文
    auto ctx_out = gguf_init_empty();
    // 设置键值对到ctx_out
    gguf_set_kv(ctx_out, ctx_src);
    // 设置ctx_out的"general.quantization_version"值为GGML_QNT_VERSION
    gguf_set_val_u32(ctx_out, "general.quantization_version", GGML_QNT_VERSION);
    // 设置ctx_out的"general.file_type"值为itype
    gguf_set_val_u32(ctx_out, "general.file_type", itype);
    // 打开一个以二进制方式写入的文件流
    auto fout = std::ofstream(fname_out, std::ios::binary);

    // 获取源上下文中张量的数量
    const int n_tensors = gguf_get_n_tensors(ctx_src);

    // 遍历源上下文中的张量，将其添加到输出上下文中
    for (int i = 0; i < n_tensors; ++i) {
        const char * name = gguf_get_tensor_name(ctx_src, i);
        struct ggml_tensor * cur = ggml_get_tensor(ctx_data, name);
        gguf_add_tensor(ctx_out, cur);
    }

    // 获取输出上下文的元数据大小，并在文件流中写入相应数量的零
    const size_t meta_size = gguf_get_meta_size(ctx_out);
    for (size_t i = 0; i < meta_size; ++i) {
        fout.put(0);
    }

    // 要进行量化的张量名称的正则表达式
    const std::vector<std::string> k_names = {
        ".*weight",
    };
    // 创建一个大小为512的uint8_t类型的向量，用于存储读取的数据
    std::vector<uint8_t> read_data(512);
    // 创建一个大小为512的uint8_t类型的向量，用于存储处理中的数据
    std::vector<uint8_t> work(512);
    // 创建一个大小为512的float类型的向量，用于存储卷积缓冲区
    std::vector<float> conv_buf(512);
    // 创建一个大小为16的int64_t类型的向量，用于存储所有直方图的数据，初始值为0
    std::vector<int64_t> hist_all(1 << 4, 0);
    // 初始化原始数据总大小和新数据总大小为0
    size_t total_size_org = 0;
    size_t total_size_new = 0;

    // 遍历n_tensors次，进行以下操作
    for (int i = 0; i < n_tensors; ++i) {
        // 获取第i个张量的名称
        const std::string name = gguf_get_tensor_name(ctx_src, i);
        // 获取当前张量的指针
        struct ggml_tensor * cur = ggml_get_tensor(ctx_data, name.c_str());

        // 定义新类型、新数据和新大小
        enum ggml_type new_type;
        void * new_data;
        size_t new_size;

        // 初始化量化标志为false
        bool quantize = false;
        // 遍历k_names向量，检查张量名称是否匹配正则表达式
        for (const auto & s : k_names) {
            if (std::regex_match(name, std::regex(s))) {
                // 如果匹配，则将量化标志设置为true，并跳出循环
                quantize = true;
                break;
        }
    }

    // 只对二维张量进行量化
    quantize &= (cur->n_dims == 2);

    // 如果需要量化
    if (quantize) {
        // 保存原始数据类型
        new_type = type;
        // 获取张量元素个数
        const size_t n_elms = ggml_nelements(cur);
        // 用于存储转换后的数据的指针
        float * f32_data;

        // 根据当前张量的数据类型进行处理
        switch (cur->type) {
        // 如果是 32 位浮点型
        case GGML_TYPE_F32:
            // 直接将数据转换为 float 类型
            f32_data = (float *)cur->data;
            break;
        // 如果是 16 位浮点型
        case GGML_TYPE_F16:
            // 如果转换缓冲区大小不足，进行扩容
            if (conv_buf.size() < n_elms) {
                conv_buf.resize(n_elms);
            }
            // 遍历每个元素，进行数据转换
            for (size_t j = 0; j < n_elms; ++j) {
                // 将 ggml_fp16_t 类型的数据转换为 float 类型的数据，并存储到 conv_buf 中
                conv_buf[j] = ggml_fp16_to_fp32(((ggml_fp16_t *)cur->data)[j]);
                // 将 conv_buf 中的数据转换为 float 类型的指针
                f32_data = (float *)conv_buf.data();
                break;
            default:
                // 如果输入文件不是 f32 或 f16 类型，则打印错误信息并返回 false
                printf("Please use an input file in f32 or f16\n");
                return false;
            }

            // 如果 work 的大小小于 n_elms * 4，则将其大小调整为 n_elms * 4
            if (work.size() < n_elms * 4) {
                work.resize(n_elms * 4);
            }
            // 将 work 中的数据转换为 new_data 指针
            new_data = work.data();

            // 创建一个大小为 1 << 4 的 int64_t 类型的 vector，初始化为 0
            std::vector<int64_t> hist_cur(1 << 4, 0);

            // 根据 new_type 的不同进行不同的处理
            switch (new_type) {
                case GGML_TYPE_Q4_0: {
                    // 如果 new_type 为 GGML_TYPE_Q4_0，则调用 ggml_quantize_q4_0 函数进行处理
                    new_size = ggml_quantize_q4_0(f32_data, new_data, n_elms, cur->ne[0], hist_cur.data());
                } break;
# 根据不同的量化类型，调用相应的函数进行数据量化处理，并更新数据大小
case GGML_TYPE_Q4_1: {
    new_size = ggml_quantize_q4_1(f32_data, new_data, n_elms, cur->ne[0], hist_cur.data());
} break;
case GGML_TYPE_Q5_0: {
    new_size = ggml_quantize_q5_0(f32_data, new_data, n_elms, cur->ne[0], hist_cur.data());
} break;
case GGML_TYPE_Q5_1: {
    new_size = ggml_quantize_q5_1(f32_data, new_data, n_elms, cur->ne[0], hist_cur.data());
} break;
case GGML_TYPE_Q8_0: {
    new_size = ggml_quantize_q8_0(f32_data, new_data, n_elms, cur->ne[0], hist_cur.data());
} break;
# 如果量化类型不在以上范围内，则输出错误信息并返回 false
default: {
    fprintf(stderr, "%s: unsupported quantization type %d\n", __func__, new_type);
    return false;
}

# 将当前处理的直方图数据累加到总的直方图数据中
for (size_t j = 0; j < hist_cur.size(); ++j) {
    hist_all[j] += hist_cur[j];
}
        } else {
            // 如果当前节点不是叶子节点，则将新类型、新数据和新大小设置为当前节点的类型、数据和大小
            new_type = cur->type;
            new_data = cur->data;
            new_size = ggml_nbytes(cur);
        }
        // 获取当前节点的原始大小
        const size_t orig_size = ggml_nbytes(cur);
        // 更新原始大小的总和
        total_size_org += orig_size;
        // 更新新大小的总和
        total_size_new += new_size;
        // 在输出上下文中设置张量类型
        gguf_set_tensor_type(ctx_out, name.c_str(), new_type);
        // 在输出上下文中设置张量数据
        gguf_set_tensor_data(ctx_out, name.c_str(), new_data, new_size);
        // 将新数据写入输出文件
        fout.write((const char *)new_data, new_size);
        // 计算需要填充的字节数
        size_t pad = GGML_PAD(new_size, gguf_get_alignment(ctx_out)) - new_size;
        // 在输出文件中填充字节
        for (size_t j = 0; j < pad; ++j) {
            fout.put(0);
        }

        // 打印节点的信息，包括名称、维度、量化信息和大小的变化
        printf("%s: n_dims = %d | quantize=%d | size = %f MB -> %f MB\n", name.c_str(), cur->n_dims, quantize,
               orig_size / 1024.0 / 1024.0, new_size / 1024.0 / 1024.0);
    }
    // 将文件指针移动到文件开头，并写入更新后的元数据
    fout.seekp(0, std::ios::beg);
    std::vector<uint8_t> meta(meta_size);
    gguf_get_meta_data(ctx_out, meta.data());
    fout.write((const char *)meta.data(), meta_size);

    // 关闭输出文件
    fout.close();

    // 释放内存
    clip_free(ctx_clip);
    gguf_free(ctx_out);

    // 打印原始大小和量化后的大小
    printf("%s: original size  = %8.2f MB\n", __func__, total_size_org / 1024.0 / 1024.0);
    printf("%s: quantized size  = %8.2f MB\n", __func__, total_size_new / 1024.0 / 1024.0);

    // 计算所有直方图值的总和
    int64_t sum_all = 0;
    for (size_t i = 0; i < hist_all.size(); ++i) {
        sum_all += hist_all[i];
    }
// 打印函数名和直方图数据
printf("%s: hist: ", __func__);
for (size_t i = 0; i < hist_all.size(); ++i) {
    // 打印每个直方图数据除以总和的比例
    printf("%5.3f ", hist_all[i] / (float)sum_all);
}
// 换行
printf("\n");
}

// 返回视觉模型的嵌入维度
return true;
}

// 返回嵌入维度的第一个元素
int clip_n_mmproj_embd(const struct clip_ctx * ctx) {
    return ctx->vision_model.mm_2_b->ne[0];
}

// 返回图像分块的数量
int clip_n_patches(const struct clip_ctx * ctx) {
    // 获取视觉模型的超参数
    auto & params = ctx->vision_model.hparams;

    // 返回图像分块的数量
    return (params.image_size / params.patch_size) * (params.image_size / params.patch_size);
}
# 计算嵌入的字节数，根据给定的上下文结构体
size_t clip_embd_nbytes(const struct clip_ctx * ctx) {
    # 返回嵌入的总字节数，由补丁数量、多模态投影数量和浮点数大小计算得出
    return clip_n_patches(ctx) * clip_n_mmproj_embd(ctx) * sizeof(float);
}
```