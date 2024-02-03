# `ggml\examples\sam\main.cpp`

```cpp
// 定义宏，用于使用数学常量 M_PI
#define _USE_MATH_DEFINES // for M_PI
// 定义宏，禁用在 Windows 上荒谬的“不安全”警告
#define _CRT_SECURE_NO_DEPRECATE // Disables ridiculous "unsafe" warnigns on Windows

// 引入自定义的头文件
#include "ggml.h"
#include "ggml-alloc.h"
// 定义 STB_IMAGE_IMPLEMENTATION 宏，用于包含 stb_image 库的实现
#define STB_IMAGE_IMPLEMENTATION
#include "stb_image.h"
// 定义 STB_IMAGE_WRITE_IMPLEMENTATION 宏，用于包含 stb_image_write 库的实现
#define STB_IMAGE_WRITE_IMPLEMENTATION
#include "stb_image_write.h"

// 引入标准库头文件
#include <cassert>
#include <cmath>
#include <cstddef>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <map>
#include <string>
#include <vector>
#include <thread>
#include <cinttypes>

// 如果编译器为 MSC_VER，则禁用警告 4244 和 4267，可能会丢失数据
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// 默认的超参数结构体 sam_hparams (ViT-B SAM)
struct sam_hparams {
    // 编码器状态的维度
    int32_t n_enc_state               = 768;
    // 编码器层数
    int32_t n_enc_layer               = 12;
    // 编码器头数
    int32_t n_enc_head                = 12;
    // 编码器输出通道数
    int32_t n_enc_out_chans           = 256;
    // 图像嵌入的维度
    int32_t n_pt_embd                 = 4;
    // 解码器头数
    int32_t n_dec_heads               = 8;
    // 特征类型
    int32_t ftype                     = 1;
    // 掩码阈值
    float   mask_threshold            = 0.f;
    // IOU 阈值
    float   iou_threshold             = 0.88f;
    // 稳定性分数阈值
    float   stability_score_threshold = 0.95f;
    // 稳定性分数偏移
    float   stability_score_offset    = 1.0f;
    // 微小值
    float   eps                       = 1e-6f;
    // 解码器变换的微小值
    float   eps_decoder_transformer   = 1e-5f;

    // 计算编码器头维度
    int32_t n_enc_head_dim() const { return n_enc_state / n_enc_head; }
    // 图像大小
    int32_t n_img_size()     const { return 1024; }
    // 窗口大小
    int32_t n_window_size()  const { return 14; }
    // 补丁大小
    int32_t n_patch_size()   const { return 16; }
    // 图像嵌入
    int32_t n_img_embd()     const { return n_img_size() / n_patch_size(); }

    // 全局注意力索引
    std::vector<int32_t> global_attn_indices() const {
        switch (n_enc_state) {
            case  768: return {  2,  5,  8, 11 };
            case 1024: return {  5, 11, 17, 23 };
            case 1280: return {  7, 15, 23, 31 };
            default:
                {
                    // 打印错误信息，不支持的编码器状态
                    fprintf(stderr, "%s: unsupported n_enc_state = %d\n", __func__, n_enc_state);
                } break;
        };

        return {};
    }
    // 检查给定层是否是全局注意力层
    bool is_global_attn(int32_t layer) const {
        // 获取全局注意力层的索引
        const auto indices = global_attn_indices();

        // 遍历全局注意力层的索引
        for (const auto & idx : indices) {
            // 如果给定层是全局注意力层的索引，则返回 true
            if (layer == idx) {
                return true;
            }
        }

        // 如果给定层不是全局注意力层的索引，则返回 false
        return false;
    }
};

// 定义 sam_layer_enc 结构体，包含多个指向 ggml_tensor 结构体的指针成员
struct sam_layer_enc {
    struct ggml_tensor * norm1_w;
    struct ggml_tensor * norm1_b;

    struct ggml_tensor * rel_pos_w;
    struct ggml_tensor * rel_pos_h;

    struct ggml_tensor * qkv_w;
    struct ggml_tensor * qkv_b;

    struct ggml_tensor * proj_w;
    struct ggml_tensor * proj_b;

    struct ggml_tensor * norm2_w;
    struct ggml_tensor * norm2_b;

    struct ggml_tensor * mlp_lin1_w;
    struct ggml_tensor * mlp_lin1_b;

    struct ggml_tensor * mlp_lin2_w;
    struct ggml_tensor * mlp_lin2_b;
};

// 定义 sam_encoder_image 结构体，包含多个指向 ggml_tensor 结构体的指针成员和一个 sam_layer_enc 结构体数组成员
struct sam_encoder_image {
    struct ggml_tensor * pe;

    struct ggml_tensor * proj_w;
    struct ggml_tensor * proj_b;

    struct ggml_tensor * neck_conv_0;
    struct ggml_tensor * neck_norm_0_w;
    struct ggml_tensor * neck_norm_0_b;
    struct ggml_tensor * neck_conv_1;
    struct ggml_tensor * neck_norm_1_w;
    struct ggml_tensor * neck_norm_1_b;

    std::vector<sam_layer_enc> layers;
};

// 定义 sam_encoder_prompt 结构体，包含多个指向 ggml_tensor 结构体的指针成员和一个 ggml_tensor 结构体指针数组成员
struct sam_encoder_prompt {
    struct ggml_tensor * pe;

    struct ggml_tensor * not_a_pt_embd_w;
    std::vector<struct ggml_tensor *> pt_embd;

    struct ggml_tensor * no_mask_embd_w;
    //std::vector<struct ggml_tensor *> mask_down_w;
    //std::vector<struct ggml_tensor *> mask_down_b;
};

// 定义 sam_layer_dec_transformer_attn 结构体，包含多个指向 ggml_tensor 结构体的指针成员
struct  sam_layer_dec_transformer_attn {
    // q_proj
    struct ggml_tensor * q_w;
    struct ggml_tensor * q_b;

    // k_proj
    struct ggml_tensor * k_w;
    struct ggml_tensor * k_b;

    // v_proj
    struct ggml_tensor * v_w;
    struct ggml_tensor * v_b;

    // out_proj
    struct ggml_tensor * out_w;
    struct ggml_tensor * out_b;
};

// 定义 sam_layer_dec_transformer 结构体，包含 sam_layer_dec_transformer_attn 结构体成员和多个指向 ggml_tensor 结构体的指针成员
struct sam_layer_dec_transformer {
    sam_layer_dec_transformer_attn self_attn;

    // norm1
    struct ggml_tensor * norm1_w;
    struct ggml_tensor * norm1_b;

    sam_layer_dec_transformer_attn cross_attn_token_to_img;

    // norm2
    struct ggml_tensor * norm2_w;
    struct ggml_tensor * norm2_b;

    // mlp.lin1
    struct ggml_tensor * mlp_lin1_w;
    struct ggml_tensor * mlp_lin1_b;

    // mlp.lin2
    # 定义指向 ggml_tensor 结构体的指针变量 mlp_lin2_w，用于存储神经网络中的线性层权重
    struct ggml_tensor * mlp_lin2_w;
    # 定义指向 ggml_tensor 结构体的指针变量 mlp_lin2_b，用于存储神经网络中的线性层偏置
    struct ggml_tensor * mlp_lin2_b;
    
    # 定义指向 ggml_tensor 结构体的指针变量 norm3_w，用于存储神经网络中的归一化层权重
    struct ggml_tensor * norm3_w;
    # 定义指向 ggml_tensor 结构体的指针变量 norm3_b，用于存储神经网络中的归一化层偏置
    struct ggml_tensor * norm3_b;
    
    # 定义指向 ggml_tensor 结构体的指针变量 norm4_w，用于存储神经网络中的归一化层权重
    struct ggml_tensor * norm4_w;
    # 定义指向 ggml_tensor 结构体的指针变量 norm4_b，用于存储神经网络中的归一化层偏置
    struct ggml_tensor * norm4_b;
    
    # 定义变量 cross_attn_img_to_token，用于存储神经网络中的跨注意力机制的图像到标记的转换层
    sam_layer_dec_transformer_attn cross_attn_img_to_token;
};

// 定义结构体 sam_layer_dec_output_hypernet_mlps
struct sam_layer_dec_output_hypernet_mlps {
    // mlps_*.layers.0
    // 定义指向 ggml_tensor 结构体的指针 w_0 和 b_0
    struct ggml_tensor * w_0;
    struct ggml_tensor * b_0;

    // mlps_*.layers.1
    // 定义指向 ggml_tensor 结构体的指针 w_1 和 b_1
    struct ggml_tensor * w_1;
    struct ggml_tensor * b_1;

    // mlps_*.layers.2
    // 定义指向 ggml_tensor 结构体的指针 w_2 和 b_2
    struct ggml_tensor * w_2;
    struct ggml_tensor * b_2;
};

// 定义结构体 sam_decoder_mask
struct sam_decoder_mask {
    // 定义存储 sam_layer_dec_transformer 结构体的向量 transformer_layers
    std::vector<sam_layer_dec_transformer> transformer_layers;

    // trasnformer.final_attn_token_to_image
    // 定义 sam_layer_dec_transformer_attn 结构体 transformer_final_attn_token_to_img
    sam_layer_dec_transformer_attn transformer_final_attn_token_to_img;

    // transformer.norm_final
    // 定义指向 ggml_tensor 结构体的指针 transformer_norm_final_w 和 transformer_norm_final_b
    struct ggml_tensor * transformer_norm_final_w;
    struct ggml_tensor * transformer_norm_final_b;

    // output_upscaling.0
    // 定义指向 ggml_tensor 结构体的指针 output_upscaling_0_w 和 output_upscaling_0_b
    struct ggml_tensor * output_upscaling_0_w;
    struct ggml_tensor * output_upscaling_0_b;

    // output_upscaling.1
    // 定义指向 ggml_tensor 结构体的指针 output_upscaling_1_w 和 output_upscaling_1_b
    struct ggml_tensor * output_upscaling_1_w;
    struct ggml_tensor * output_upscaling_1_b;

    // output_upscaling.3
    // 定义指向 ggml_tensor 结构体的指针 output_upscaling_3_w 和 output_upscaling_3_b
    struct ggml_tensor * output_upscaling_3_w;
    struct ggml_tensor * output_upscaling_3_b;

    // output_hypernetworks_mlps
    // 定义存储 sam_layer_dec_output_hypernet_mlps 结构体的向量 output_hypernet_mlps
    std::vector<sam_layer_dec_output_hypernet_mlps> output_hypernet_mlps;

    // iou_prediction_head.0
    // 定义指向 ggml_tensor 结构体的指针 iou_prediction_head_0_w 和 iou_prediction_head_0_b
    struct ggml_tensor * iou_prediction_head_0_w;
    struct ggml_tensor * iou_prediction_head_0_b;

    // iou_prediction_head.1
    // 定义指向 ggml_tensor 结构体的指针 iou_prediction_head_1_w 和 iou_prediction_head_1_b
    struct ggml_tensor * iou_prediction_head_1_w;
    struct ggml_tensor * iou_prediction_head_1_b;

    // iou_prediction_head.2
    // 定义指向 ggml_tensor 结构体的指针 iou_prediction_head_2_w 和 iou_prediction_head_2_b
    struct ggml_tensor * iou_prediction_head_2_w;
    struct ggml_tensor * iou_prediction_head_2_b;

    // iou_token.weight
    // 定义指向 ggml_tensor 结构体的指针 iou_token_w
    struct ggml_tensor * iou_token_w;

    // mask_tokens.weight
    // 定义指向 ggml_tensor 结构体的指针 mask_tokens_w
    struct ggml_tensor * mask_tokens_w;
};

// 定义结构体 sam_state
struct sam_state {
    // 定义指向 ggml_tensor 结构体的指针 embd_img, low_res_masks, iou_predictions
    struct ggml_tensor * embd_img;
    struct ggml_tensor * low_res_masks;
    struct ggml_tensor * iou_predictions;

    //struct ggml_tensor * tmp_save = {};

    // 定义指向 ggml_context 结构体的指针 ctx
    struct ggml_context * ctx;

    // buffer for `ggml_graph_plan.work_data`
    // 定义存储 uint8_t 类型数据的向量 work_buffer
    std::vector<uint8_t> work_buffer;
    // buffers to evaluate the model
};
    # 创建一个存储无符号8位整数的向量，用于存储分配图像编码的缓冲区
    std::vector<uint8_t> buf_alloc_img_enc;
    
    # 创建一个存储无符号8位整数的向量，用于存储计算图像编码的缓冲区
    std::vector<uint8_t> buf_compute_img_enc;
    
    # 创建一个存储无符号8位整数的向量，用于存储分配快速计算的缓冲区
    std::vector<uint8_t> buf_alloc_fast;
    
    # 创建一个存储无符号8位整数的向量，用于存储计算快速计算的缓冲区
    std::vector<uint8_t> buf_compute_fast;
    
    # 创建一个指向 ggml_allocr 结构体的指针，并初始化为空
    struct ggml_allocr * allocr = {};
// 结构体 sam_model 包含了模型的超参数、图像编码器、提示编码器和解码器
struct sam_model {
    sam_hparams hparams; // 模型的超参数

    sam_encoder_image  enc_img; // 图像编码器
    sam_encoder_prompt enc_prompt; // 提示编码器
    sam_decoder_mask   dec; // 解码器

    //
    struct ggml_context * ctx; // GGML 上下文
    std::map<std::string, struct ggml_tensor *> tensors; // 字符串到张量的映射
};

// 结构体 sam_point 包含了两个浮点数，表示一个点的坐标
struct sam_point {
    float x; // x 坐标
    float y; // y 坐标
};

// 结构体 sam_image_u8 表示 RGB uint8 图像
struct sam_image_u8 {
    int nx; // 图像宽度
    int ny; // 图像高度

    std::vector<uint8_t> data; // 图像数据
};

// 结构体 sam_image_f32 表示 RGB float32 图像
// 内存布局：RGBRGBRGB...
struct sam_image_f32 {
    int nx; // 图像宽度
    int ny; // 图像高度

    std::vector<float> data; // 图像数据
};

// 结构体 sam_params 包含了一系列参数，包括 RNG 种子、线程数、模型路径、输入文件名、输出文件名等
struct sam_params {
    int32_t seed      = -1; // RNG 种子
    int32_t n_threads = std::min(4, (int32_t) std::thread::hardware_concurrency()); // 线程数

    std::string model     = "models/sam-vit-b/ggml-model-f16.bin"; // 模型路径
    std::string fname_inp = "img.jpg"; // 输入文件名
    std::string fname_out = "img.out"; // 输出文件名
    float   mask_threshold            = 0.f; // 阈值
    float   iou_threshold             = 0.88f; // 阈值
    float   stability_score_threshold = 0.95f; // 阈值
    float   stability_score_offset    = 1.0f; // 偏移量
    float   eps                       = 1e-6f; // 微小值
    float   eps_decoder_transformer   = 1e-5f; // 微小值
    sam_point pt = { 414.375f, 162.796875f, }; // 点的坐标
};

// 打印 float32 类型的张量
void print_t_f32(const char* title, struct ggml_tensor * t, int n = 10) {
    printf("%s\n", title); // 打印标题
    float * data = (float *)t->data; // 获取张量数据
    printf("dims: % " PRId64 " % " PRId64 " % " PRId64 " % " PRId64 " f32\n", t->ne[0], t->ne[1], t->ne[2], t->ne[3]); // 打印张量维度
    printf("First & Last %d elements:\n", n); // 打印前后 n 个元素
    for (int i = 0; i < std::min((int) (t->ne[0]*t->ne[1]), n); i++) { // 遍历前 n 个元素
        printf("%.5f ", data[i]); // 打印元素值
        if (i != 0 && i % t->ne[0] == 0) { // 换行条件
            printf("\n"); // 换行
        }
    }
    # 打印一个换行符
    printf("\n");
    # 遍历数组 data，打印数组中的元素值，最多打印 t->ne[0]*t->ne[1] 个元素或者 n 个元素，取较小值
    for (int i = 0; i < std::min((int) (t->ne[0]*t->ne[1]), n); i++) {
        printf("%.5f ", data[ggml_nelements(t) - n + i]);
        # 如果当前元素是当前行的最后一个元素，则打印一个换行符
        if ((ggml_nelements(t) - n + i) % t->ne[0] == 0) {
            printf("\n");
        }
    }
    # 打印一个换行符
    printf("\n");
    # 计算数组 data 中所有元素的和
    double sum = 0.0;
    for (int i = 0; i < ggml_nelements(t); i++) {
        sum += data[i];
    }
    # 打印数组 data 中所有元素的和
    printf("sum:  %f\n\n", sum);
// 从图中断开节点，将节点操作设置为无，将节点的源数组置空
static void ggml_disconnect_node_from_graph(ggml_tensor * t) {
    t->op = GGML_OP_NONE;
    for (int i = 0; i < GGML_MAX_SRC; i++) {
        t->src[i] = NULL;
    }
}

// 辅助函数，用于计算图的计算计划，并执行图的计算
static void ggml_graph_compute_helper(std::vector<uint8_t> & buf, ggml_cgraph * graph, int n_threads) {
    // 获取图的计划
    struct ggml_cplan plan = ggml_graph_plan(graph, n_threads);

    // 如果计划的工作大小大于0，则调整缓冲区大小，并将工作数据指针指向缓冲区
    if (plan.work_size > 0) {
        buf.resize(plan.work_size);
        plan.work_data = buf.data();
    }

    // 执行图的计算
    ggml_graph_compute(graph, &plan);
}

// 计算 sin 函数应用于张量的每个元素
static void ggml_sam_sin(struct ggml_tensor * dst , const struct ggml_tensor * src, int ith, int nth, void * userdata) {
    // 断言用户数据为空
    GGML_ASSERT(userdata == NULL);
    // 断言目标张量和源张量具有相同的形状
    GGML_ASSERT(ggml_are_same_shape(dst, src));
    // 断言目标张量是连续的
    GGML_ASSERT(ggml_is_contiguous(dst));
    // 断言源张量是连续的
    GGML_ASSERT(ggml_is_contiguous(src));

    // 获取源张量和目标张量的数据指针
    const float * src_data = ggml_get_data_f32(src);
    float * dst_data = ggml_get_data_f32(dst);

    // 计算元素的数量
    const int ne = (int)ggml_nelements(dst);
    // 计算每个线程的工作量
    const int dr = (ne + nth - 1) / nth;
    // 计算当前线程的起始索引和结束索引
    const int ie0 = dr * ith;
    const int ie1 = std::min(ie0 + dr, ne);

    // 对每个元素应用 sin 函数
    for (int i = ie0; i < ie1; ++i) {
        dst_data[i] = sinf(src_data[i]);
    }
}

// 计算 cos 函数应用于张量的每个元素
static void ggml_sam_cos(struct ggml_tensor * dst , const struct ggml_tensor * src, int ith, int nth, void * userdata) {
    // 断言用户数据为空
    GGML_ASSERT(userdata == NULL);
    // 断言目标张量和源张量具有相同的形状
    GGML_ASSERT(ggml_are_same_shape(dst, src));
    // 断言目标张量是连续的
    GGML_ASSERT(ggml_is_contiguous(dst));
    // 断言源张量是连续的
    GGML_ASSERT(ggml_is_contiguous(src));

    // 获取源张量和目标张量的数据指针
    const float * src_data = ggml_get_data_f32(src);
    float * dst_data = ggml_get_data_f32(dst);

    // 计算元素的数量
    const int ne = (int)ggml_nelements(dst);
    // 计算每个线程的工作量
    const int dr = (ne + nth - 1) / nth;
    // 计算当前线程的起始索引和结束索引
    const int ie0 = dr * ith;
    const int ie1 = std::min(ie0 + dr, ne);

    // 对每个元素应用 cos 函数
    for (int i = ie0; i < ie1; ++i) {
        dst_data[i] = cosf(src_data[i]);
    }
}

// 从文件加载图像数据到图像结构体
bool sam_image_load_from_file(const std::string & fname, sam_image_u8 & img) {
    int nx, ny, nc;
    // 使用 stb_image 库从文件中加载图像数据
    auto data = stbi_load(fname.c_str(), &nx, &ny, &nc, 3);
    # 如果数据为空，则输出错误信息并返回 false
    if (!data) {
        fprintf(stderr, "%s: failed to load '%s'\n", __func__, fname.c_str());
        return false;
    }

    # 将图像的宽度和高度赋值给对应的属性
    img.nx = nx;
    img.ny = ny;
    # 调整图像数据的大小为宽度乘以高度乘以3（RGB通道）
    img.data.resize(nx * ny * 3);
    # 将数据从指针中复制到图像数据中
    memcpy(img.data.data(), data, nx * ny * 3);

    # 释放加载的图像数据
    stbi_image_free(data);

    # 返回 true 表示加载成功
    return true;
// 从这里开始是一个C++函数的定义，用于对图像进行预处理
// 输入参数为一个8位图像结构体img，输出参数为一个32位浮点数图像结构体res
bool sam_image_preprocess(const sam_image_u8 & img, sam_image_f32 & res) {
    // 获取输入图像的宽度和高度
    const int nx = img.nx;
    const int ny = img.ny;

    // 设置输出图像的宽度和高度为1024
    const int nx2 = 1024;
    const int ny2 = 1024;

    // 将输出图像的宽度和高度赋值给res结构体
    res.nx = nx2;
    res.ny = ny2;
    // 重新分配输出图像数据的内存空间，大小为3*nx2*ny2
    res.data.resize(3*nx2*ny2);

    // 计算输入图像的最大维度与1024的比值，作为缩放比例
    const float scale = std::max(nx, ny) / 1024.0f;

    // 打印缩放比例的信息
    fprintf(stderr, "%s: scale = %f\n", __func__, scale);

    // 根据缩放比例计算输入图像经过缩放后的宽度和高度
    const int nx3 = int(nx/scale + 0.5f);
    const int ny3 = int(ny/scale + 0.5f);

    // 定义图像数据的均值和标准差
    const float m3[3] = { 123.675f, 116.280f, 103.530f };
    const float s3[3] = {  58.395f,  57.120f,  57.375f };
}
    # 遍历 y 轴像素
    for (int y = 0; y < ny3; y++) {
        # 遍历 x 轴像素
        for (int x = 0; x < nx3; x++) {
            # 遍历 RGB 通道
            for (int c = 0; c < 3; c++) {
                # 线性插值计算
                const float sx = (x + 0.5f)*scale - 0.5f;
                const float sy = (y + 0.5f)*scale - 0.5f;

                const int x0 = std::max(0, (int) std::floor(sx));
                const int y0 = std::max(0, (int) std::floor(sy));

                const int x1 = std::min(x0 + 1, nx - 1);
                const int y1 = std::min(y0 + 1, ny - 1);

                const float dx = sx - x0;
                const float dy = sy - y0;

                const int j00 = 3*(y0*nx + x0) + c;
                const int j01 = 3*(y0*nx + x1) + c;
                const int j10 = 3*(y1*nx + x0) + c;
                const int j11 = 3*(y1*nx + x1) + c;

                const float v00 = img.data[j00];
                const float v01 = img.data[j01];
                const float v10 = img.data[j10];
                const float v11 = img.data[j11];

                const float v0 = v00*(1.0f - dx) + v01*dx;
                const float v1 = v10*(1.0f - dx) + v11*dx;

                const float v = v0*(1.0f - dy) + v1*dy;

                const uint8_t v2 = std::min(std::max(std::round(v), 0.0f), 255.0f);

                const int i = 3*(y*nx3 + x) + c;

                # 将插值结果标准化后存入结果数组
                res.data[i] = (float(v2) - m3[c]) / s3[c];
            }
        }
    }

    # 返回处理结果
    return true;
// 从文件中加载模型的权重
bool sam_model_load(const sam_params & params, sam_model & model) {
    // 打印加载模型的提示信息
    fprintf(stderr, "%s: loading model from '%s' - please wait ...\n", __func__, params.model.c_str());

    // 打开模型文件
    auto fin = std::ifstream(params.model, std::ios::binary);
    // 如果文件打开失败，打印错误信息并返回false
    if (!fin) {
        fprintf(stderr, "%s: failed to open '%s'\n", __func__, params.model.c_str());
        return false;
    }

    // 验证魔数
    {
        uint32_t magic;
        fin.read((char *) &magic, sizeof(magic));
        // 如果魔数不匹配，打印错误信息并返回false
        if (magic != 0x67676d6c) {
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, params.model.c_str());
            return false;
        }
    }

    // 加载超参数
    {
        // 用用户选择的参数覆盖默认参数
        model.hparams.mask_threshold            = params.mask_threshold;
        model.hparams.iou_threshold             = params.iou_threshold;
        model.hparams.stability_score_threshold = params.stability_score_threshold;
        model.hparams.stability_score_offset    = params.stability_score_offset;
        model.hparams.eps                       = params.eps;
        model.hparams.eps_decoder_transformer   = params.eps_decoder_transformer;

        auto & hparams = model.hparams;

        // 从文件中读取编码器状态数量，并存入模型参数中
        fin.read((char *) &hparams.n_enc_state,     sizeof(hparams.n_enc_state));
        // 从文件中读取编码器层数，并存入模型参数中
        fin.read((char *) &hparams.n_enc_layer,     sizeof(hparams.n_enc_layer));
        // 从文件中读取编码器头数量，并存入模型参数中
        fin.read((char *) &hparams.n_enc_head,      sizeof(hparams.n_enc_head));
        // 从文件中读取编码器输出通道数量，并存入模型参数中
        fin.read((char *) &hparams.n_enc_out_chans, sizeof(hparams.n_enc_out_chans));
        // 从文件中读取位置编码维度，并存入模型参数中
        fin.read((char *) &hparams.n_pt_embd,       sizeof(hparams.n_pt_embd));
        // 从文件中读取特征类型，并存入模型参数中
        fin.read((char *) &hparams.ftype,           sizeof(hparams.ftype));

        // 计算特征类型的量化版本
        const int32_t qntvr = hparams.ftype / GGML_QNT_VERSION_FACTOR;

        // 打印编码器状态数量
        printf("%s: n_enc_state      = %d\n", __func__, hparams.n_enc_state);
        // 打印编码器层数
        printf("%s: n_enc_layer      = %d\n", __func__, hparams.n_enc_layer);
        // 打印编码器头数量
        printf("%s: n_enc_head       = %d\n", __func__, hparams.n_enc_head);
        // 打印编码器输出通道数量
        printf("%s: n_enc_out_chans  = %d\n", __func__, hparams.n_enc_out_chans);
        // 打印位置编码维度
        printf("%s: n_pt_embd        = %d\n", __func__, hparams.n_pt_embd);
        // 打印特征类型
        printf("%s: ftype            = %d\n", __func__, hparams.ftype);
        // 打印特征类型的量化版本
        printf("%s: qntvr            = %d\n", __func__, qntvr);

        // 计算特征类型的量化版本并存入模型参数中
        hparams.ftype %= GGML_QNT_VERSION_FACTOR;

    }

    // 对于大张量，我们有选项将数据存储为16位浮点数或量化形式
    // 以节省内存并加快计算速度
    // 将特征类型转换为GGML类型，并存入wtype中
    ggml_type wtype = ggml_ftype_to_ggml_type((ggml_ftype) (model.hparams.ftype));
    // 如果 wtype 等于 GGML_TYPE_COUNT，则输出错误信息并返回 false
    if (wtype == GGML_TYPE_COUNT) {
        fprintf(stderr, "%s: invalid model file '%s' (bad ftype value %d)\n",
                __func__, params.model.c_str(), model.hparams.ftype);
        return false;
    }

    // 获取模型的上下文
    auto & ctx = model.ctx;

    // 创建 ggml 上下文
    {
        // 定义 ggml_init_params 结构体并初始化参数
        struct ggml_init_params params = {
            /*.mem_size   =*/ ctx_size,
            /*.mem_buffer =*/ NULL,
            /*.no_alloc   =*/ false,
        };

        // 调用 ggml_init 函数创建上下文，并将结果赋值给 ctx
        ctx = ggml_init(params);
        // 如果创建失败，则输出错误信息并返回 false
        if (!ctx) {
            fprintf(stderr, "%s: ggml_init() failed\n", __func__);
            return false;
        }
    }

    // 准备权重的内存空间

    // 加载权重

    // 关闭文件流

    // 返回 true
    return true;
    // 填充密集位置编码
    struct ggml_tensor * sam_fill_dense_pe(
            // 使用模型、上下文、计算图和状态作为参数
            const sam_model   & model,
            struct ggml_context * ctx0,
            struct ggml_cgraph  * gf,
            sam_state   & state) {
    // 获取模型的超参数和编码提示
    const auto & hparams = model.hparams;
    const auto & enc     = model.enc_prompt;

    // 计算图像嵌入的数量和其倒数
    const int32_t n_img_embd = hparams.n_img_embd();
    const float n_img_embd_inv = 1.0f / n_img_embd;

    // 创建一个二维的浮点型张量
    struct ggml_tensor * xy_embed_stacked = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, 2, n_img_embd, n_img_embd);
    // 分配内存
    ggml_allocr_alloc(state.allocr, xy_embed_stacked);

    // 如果内存不是度量的
    if (!ggml_allocr_is_measure(state.allocr)) {
        // 获取张量的数据指针
        float * data = (float *) ggml_get_data(xy_embed_stacked);
        // 遍历图像嵌入
        for (int i = 0; i < n_img_embd; ++i) {
            const int row = 2*i*n_img_embd;
            const float y_val = 2 * (i + 0.5f) * n_img_embd_inv - 1;
            // 遍历图像嵌入
            for (int j = 0; j < n_img_embd; ++j) {
                const float x_val = 2 * (j + 0.5f) * n_img_embd_inv - 1;
                // 填充数据
                data[row + 2*j + 0] = x_val;
                data[row + 2*j + 1] = y_val;
            }
        }
    }

    // 计算当前张量
    struct ggml_tensor * cur = ggml_mul_mat(ctx0, ggml_cont(ctx0, ggml_transpose(ctx0, enc.pe)), xy_embed_stacked);

    // 缩放当前张量
    cur = ggml_scale(ctx0, cur, float(2.0*M_PI));

    // 连接
    // 参考：https://github.com/facebookresearch/segment-anything/blob/main/segment_anything/modeling/prompt_encoder.py#L192
    # 使用 ggml_map_custom1 函数将 ggml_sam_sin 函数应用到当前上下文和张量 cur 上，返回结果存储在 t_sin 中
    struct ggml_tensor * t_sin = ggml_map_custom1(ctx0, cur, ggml_sam_sin, GGML_N_TASKS_MAX, NULL);
    # 使用 ggml_map_custom1 函数将 ggml_sam_cos 函数应用到当前上下文和张量 cur 上，返回结果存储在 t_cos 中
    struct ggml_tensor * t_cos = ggml_map_custom1(ctx0, cur, ggml_sam_cos, GGML_N_TASKS_MAX, NULL);

    # 创建一个新的三维张量，其第一维大小为 t_sin->ne[0] + t_cos->ne[0]，其余维度与 cur 相同
    cur = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, t_sin->ne[0] + t_cos->ne[0], cur->ne[1], cur->ne[2]);

    # 使用 ggml_build_forward_expand 函数对 gf 进行前向扩展操作，将 t_sin 的数据复制到 cur 中指定的位置
    ggml_build_forward_expand(gf, ggml_cpy(ctx0, t_sin, ggml_view_3d(ctx0, cur, t_sin->ne[0], t_sin->ne[1], t_sin->ne[2], cur->nb[1], cur->nb[2], 0)));
    # 使用 ggml_build_forward_expand 函数对 gf 进行前向扩展操作，将 t_cos 的数据复制到 cur 中指定的位置
    ggml_build_forward_expand(gf, ggml_cpy(ctx0, t_cos, ggml_view_3d(ctx0, cur, t_sin->ne[0], t_sin->ne[1], t_sin->ne[2], cur->nb[1], cur->nb[2], t_sin->nb[1]));

    # 对 cur 进行维度置换，将第三维移到第一维，第一维移到第二维，第二维移到第三维，结果存储在 pe_img_dense 中
    struct ggml_tensor * pe_img_dense = ggml_cont(ctx0, ggml_permute(ctx0, cur, 2, 0, 1, 3));
    # 使用 ggml_build_forward_expand 函数对 gf 进行前向扩展操作，将 pe_img_dense 的数据传递给 gf
    ggml_build_forward_expand(gf, pe_img_dense);

    # 返回处理后的张量 pe_img_dense
    return pe_img_dense;
}

// 对二维张量进行 LayerNorm 操作
struct ggml_tensor* sam_layer_norm_2d(
                    struct ggml_context * ctx0,
                    struct ggml_tensor  * layer,
                    int                   n_channels,
                    struct ggml_tensor  * w,
                    struct ggml_tensor  * b,
                    float                 eps) {
    // LayerNorm2d
    // 沿着通道维度进行归一化
    // TODO: 更好的实现方式
    layer = ggml_permute(ctx0,
                ggml_norm(ctx0, ggml_cont(ctx0, ggml_permute(ctx0, layer, 1, 2, 0, 3)), eps),
                2, 0, 1, 3);

    layer = ggml_add(ctx0,
              ggml_mul(ctx0,
                  ggml_repeat(ctx0, ggml_reshape_3d(ctx0, w, 1, 1, n_channels), layer),
                  layer),
              ggml_repeat(ctx0, ggml_reshape_3d(ctx0, b, 1, 1, n_channels), layer));

    return layer;
}

// 对图像进行编码
struct ggml_cgraph  * sam_encode_image(
            const sam_model & model,
                  sam_state & state,
        const sam_image_f32 & img) {

    const auto & hparams = model.hparams;
    const auto & enc     = model.enc_img;

    const int32_t n_enc_state     = hparams.n_enc_state;
    const int32_t n_enc_layer     = hparams.n_enc_layer;
    const int32_t n_enc_head      = hparams.n_enc_head;
    const int32_t n_enc_head_dim  = hparams.n_enc_head_dim();
    const int32_t n_enc_out_chans = hparams.n_enc_out_chans;
    const int32_t n_img_size    = hparams.n_img_size();
    const int32_t n_window_size = hparams.n_window_size();

    // 初始化参数
    struct ggml_init_params ggml_params = {
        /*.mem_size   =*/ state.buf_compute_img_enc.size(),
        /*.mem_buffer =*/ state.buf_compute_img_enc.data(),
        /*.no_alloc   =*/ true, // 跳过分配内存，因为我们使用 ggml_alloc 来精确分配内存需求
    };

    // 初始化上下文
    struct ggml_context * ctx0   = ggml_init(ggml_params);
    // 创建计算图
    struct ggml_cgraph  * gf     = ggml_new_graph(ctx0);
    // 创建一个4维的浮点型输入张量，大小为n_img_size * n_img_size * 3 * 1
    struct ggml_tensor * inp = ggml_new_tensor_4d(ctx0, GGML_TYPE_F32, n_img_size, n_img_size, 3, 1);
    // 分配内存给输入张量
    ggml_allocr_alloc(state.allocr, inp);
    // 如果内存分配器不是度量模式
    if (!ggml_allocr_is_measure(state.allocr)) {
        // 获取输入张量的数据指针
        float * data = (float *) ggml_get_data(inp);

        // 获取图像的宽度和高度
        const int nx = img.nx;
        const int ny = img.ny;
        const int n  = nx*ny;

        // 断言图像的宽度和高度与输入张量的大小相匹配
        GGML_ASSERT(nx == n_img_size && ny == n_img_size);

        // 将图像数据转换为输入张量的数据格式
        for (int k = 0; k < 3; k++) {
            for (int y = 0; y < ny; y++) {
                for (int x = 0; x < nx; x++) {
                    data[k*n + y*nx + x] = img.data[3*(y*nx + x) + k];
                }
            }
        }
    }

    // 使用上下文和权重对输入张量进行二维卷积操作
    struct ggml_tensor * cur = ggml_conv_2d_sk_p0(ctx0, enc.proj_w, inp);
    // 将当前张量与重复的偏置项相加
    cur = ggml_add_inplace(ctx0,
            cur,
            ggml_repeat(ctx0, enc.proj_b, cur));

    // 将当前张量的维度重新排列
    cur = ggml_cont(ctx0,
            ggml_permute(ctx0, cur, 1, 2, 0, 3));

    // 将当前张量转换为F16格式（已注释掉的代码）
    //cur = ggml_cpy(ctx0,
    //        ggml_permute(ctx0, cur, 1, 2, 0, 3),
    //        ggml_new_tensor_3d(ctx0, GGML_TYPE_F16, n_enc_state, n_img_embd, n_img_embd));

    // 将当前张量与enc.pe相加
    cur = ggml_add_inplace(ctx0, cur, enc.pe);

    // 将当前张量赋值给inpL
    struct ggml_tensor * inpL = cur;

    // 将inpL的维度重新排列
    cur = ggml_cont(ctx0, ggml_permute(ctx0, inpL, 2, 0, 1, 3));

    // 使用上下文和权重对inpL进行二维卷积操作
    cur = ggml_conv_2d_sk_p0(ctx0, enc.neck_conv_0, cur);

    // 对当前张量进行二维层归一化操作
    cur = sam_layer_norm_2d(ctx0, cur, n_enc_out_chans, enc.neck_norm_0_w, enc.neck_norm_0_b, hparams.eps);

    // 使用上下文和权重对当前张量进行二维卷积操作
    cur = ggml_conv_2d_s1_ph(ctx0, enc.neck_conv_1, cur);

    // 对当前张量进行二维层归一化操作
    cur = sam_layer_norm_2d(ctx0, cur, n_enc_out_chans, enc.neck_norm_1_w, enc.neck_norm_1_b, hparams.eps);
    # 调用 ggml_cpy 函数，将 ctx0 和 state.embd_img 复制到 cur 中
    cur = ggml_cpy(ctx0, cur, state.embd_img);

    # 调用 ggml_build_forward_expand 函数，构建前向扩展，传入参数 gf 和 cur
    ggml_build_forward_expand(gf, cur);
    # 断开 state.embd_img 节点与图的连接
    ggml_disconnect_node_from_graph(state.embd_img);

    # 注释掉打印图的语句
    //ggml_graph_print(&gf);

    # 释放 ctx0 占用的内存
    ggml_free(ctx0);

    # 返回 gf
    return gf;
// 结构体定义，用于存储编码后的提示信息
struct prompt_encoder_result {
    // 稀疏嵌入的提示信息
    struct ggml_tensor * embd_prompt_sparse = {};
    // 密集嵌入的提示信息
    struct ggml_tensor * embd_prompt_dense = {};
};

// 编码提示信息
//
// - 点
// - 方框
// - 掩模
//
// TODO: 目前只为简单起见编码单个点
//
prompt_encoder_result sam_encode_prompt(
        const sam_model     & model,
        struct ggml_context * ctx0,
        struct ggml_cgraph  * gf,
                  sam_state & state,
                        int   nx,
                        int   ny,
                  sam_point   point) {

    const auto & hparams = model.hparams;
    const auto & enc = model.enc_prompt;

    // 转换点的坐标
    // 参考：https://github.com/facebookresearch/segment-anything/blob/main/segment_anything/automatic_mask_generator.py#L276
    {
        // 计算最大值
        const int nmax = std::max(nx, ny);

        // 计算缩放比例
        const float scale = hparams.n_img_size() / (float) nmax;

        // 计算新的坐标
        const int nx_new = int(nx*scale + 0.5f);
        const int ny_new = int(ny*scale + 0.5f);

        point.x = point.x*(float(nx_new)/nx) + 0.5f;
        point.y = point.y*(float(ny_new)/ny) + 0.5f;
    }

    // 创建一个二维张量
    struct ggml_tensor * inp = ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, 2, 2);

    // 分配内存
    ggml_allocr_alloc(state.allocr, inp);
    if (!ggml_allocr_is_measure(state.allocr)) {
        // 将 [0, 1] 坐标转换为 [-1, 1] 并设置输入
        float * data = (float *) inp->data;

        data[0] = 2.0f*(point.x / hparams.n_img_size()) - 1.0f;
        data[1] = 2.0f*(point.y / hparams.n_img_size()) - 1.0f;

        // 填充
        // 参考：https://github.com/facebookresearch/segment-anything/blob/main/segment_anything/modeling/prompt_encoder.py#L81-L85
        data[2] = 2.0f*(0.0f) - 1.0f;
        data[3] = 2.0f*(0.0f) - 1.0f;
    }

    // 计算当前张量
    struct ggml_tensor * cur = ggml_mul_mat(ctx0, ggml_cont(ctx0, ggml_transpose(ctx0, enc.pe)), inp);

    // 缩放
    cur = ggml_scale(ctx0, cur, float(2.0*M_PI);

    // 连接
    # 创建一个名为 t_sin 的指向 ggml_tensor 结构的指针，调用 ggml_map_custom1 函数
    struct ggml_tensor * t_sin = ggml_map_custom1(ctx0, cur, ggml_sam_sin, GGML_N_TASKS_MAX, NULL);
    # 创建一个名为 t_cos 的指向 ggml_tensor 结构的指针，调用 ggml_map_custom1 函数
    struct ggml_tensor * t_cos = ggml_map_custom1(ctx0, cur, ggml_sam_cos, GGML_N_TASKS_MAX, NULL);

    # 创建一个新的二维张量 cur，其大小为 t_sin->ne[0] + t_cos->ne[0] 行，cur->ne[1] 列
    cur = ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, t_sin->ne[0] + t_cos->ne[0], cur->ne[1]);

    # 使用 ggml_build_forward_expand 函数构建前向扩展，将 t_sin 的内容复制到 cur 的前 t_sin->ne[0] 行
    ggml_build_forward_expand(gf, ggml_cpy(ctx0, t_sin, ggml_view_2d(ctx0, cur, t_sin->ne[0], t_sin->ne[1], cur->nb[1], 0)));
    # 使用 ggml_build_forward_expand 函数构建前向扩展，将 t_cos 的内容复制到 cur 的后 t_sin->ne[0] 行
    ggml_build_forward_expand(gf, ggml_cpy(ctx0, t_cos, ggml_view_2d(ctx0, cur, t_sin->ne[0], t_sin->ne[1], cur->nb[1], t_sin->nb[1]));

    # 使用 ggml_build_forward_expand 函数构建前向扩展，将 enc.not_a_pt_embd_w 的内容复制到 cur 的最后一列
    ggml_build_forward_expand(gf, ggml_cpy(ctx0, enc.not_a_pt_embd_w, ggml_view_2d(ctx0, cur, cur->ne[0], 1, cur->nb[1], cur->nb[1])));

    # 创建一个名为 v 的指向 ggml_tensor 结构的指针，调用 ggml_view_2d 函数
    struct ggml_tensor * v = ggml_view_2d(ctx0, cur, cur->ne[0], 1, cur->nb[1], 0);
    # 使用 ggml_build_forward_expand 函数构建前向扩展，将 enc.pt_embd[1] 添加到 v 中
    ggml_build_forward_expand(gf, ggml_cpy(ctx0, ggml_add_inplace(ctx0, v, enc.pt_embd[1]), v));

    # 创建一个名为 embd_prompt_sparse 的指向 ggml_tensor 结构的指针，指向 cur
    struct ggml_tensor * embd_prompt_sparse = cur;
    # 使用 ggml_build_forward_expand 函数构建前向扩展，处理 embd_prompt_sparse
    ggml_build_forward_expand(gf, embd_prompt_sparse);

    # 创建一个名为 embd_prompt_dense 的指向 ggml_tensor 结构的指针，调用 ggml_repeat 函数
    struct ggml_tensor * embd_prompt_dense = ggml_repeat(ctx0,
            ggml_cont(ctx0,
                ggml_view_3d(ctx0, enc.no_mask_embd_w,
                    1, 1, enc.no_mask_embd_w->ne[0], enc.no_mask_embd_w->nb[0], enc.no_mask_embd_w->nb[0], 0)),
            ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, hparams.n_img_embd(), hparams.n_img_embd(), hparams.n_enc_out_chans));
    # 使用 ggml_build_forward_expand 函数构建前向扩展，处理 embd_prompt_dense
    ggml_build_forward_expand(gf, embd_prompt_dense);
    // 打印当前使用的内存大小
    //printf("used_mem = %zu\n", ggml_used_mem(ctx0));

    // 创建一个名为 res 的 prompt_encoder_result 结构体实例
    prompt_encoder_result res;
    // 将 embd_prompt_sparse 赋值给 res 结构体中的 embd_prompt_sparse 成员变量
    res.embd_prompt_sparse = embd_prompt_sparse;
    // 将 embd_prompt_dense 赋值给 res 结构体中的 embd_prompt_dense 成员变量
    res.embd_prompt_dense  = embd_prompt_dense;
    // 返回 res 结构体实例
    return res;
// 对 Transformer 注意力机制进行解码，根据给定的查询、键、值张量以及模型参数进行计算
struct ggml_tensor* sam_decode_mask_transformer_attn(
    const sam_layer_dec_transformer_attn & attn,  // 传入的 Transformer 注意力机制参数
                      struct ggml_tensor * queries,  // 查询张量
                      struct ggml_tensor * keys,     // 键张量
                      struct ggml_tensor * values,   // 值张量
                     struct ggml_context * ctx0,    // 上下文环境
                         const sam_model & model) {  // 模型参数
    const auto & hparams = model.hparams;  // 获取模型参数中的超参数
    const int n_head = hparams.n_dec_heads;  // 获取解码器头数

    struct ggml_tensor * Qcur = {};  // 初始化当前查询张量
    struct ggml_tensor * Kcur = {};  // 初始化当前键张量
    struct ggml_tensor * Vcur = {};  // 初始化当前值张量

    Qcur = ggml_mul_mat(ctx0, attn.q_w, queries);  // 计算当前查询张量
    Qcur = ggml_add_inplace(ctx0, Qcur, attn.q_b);  // 添加偏置到当前查询张量

    Kcur = ggml_mul_mat(ctx0, attn.k_w, keys);  // 计算当前键张量
    Kcur = ggml_add_inplace(ctx0, Kcur, attn.k_b);  // 添加偏置到当前键张量

    Vcur = ggml_mul_mat(ctx0, attn.v_w, values);  // 计算当前值张量
    Vcur = ggml_add_inplace(ctx0, Vcur, attn.v_b);  // 添加偏置到当前值张量

    struct ggml_tensor * Q = {};  // 初始化变换后的查询张量
    struct ggml_tensor * K = {};  // 初始化变换后的键张量
    struct ggml_tensor * V = {};  // 初始化变换后的值张量

    Q = ggml_reshape_4d(ctx0, Qcur, Qcur->ne[0]/n_head, n_head, Qcur->ne[1], Qcur->ne[2]);  // 对当前查询张量进行形状变换
    Q = ggml_cont(ctx0, ggml_permute(ctx0, Q, 0, 2, 1, 3));  // 对变换后的查询张量进行维度变换

    K = ggml_reshape_4d(ctx0, Kcur, Kcur->ne[0]/n_head, n_head, Kcur->ne[1], Kcur->ne[2]);  // 对当前键张量进行形状变换
    K = ggml_cont(ctx0, ggml_permute(ctx0, K, 0, 2, 1, 3));  // 对变换后的键张量进行维度变换

    V = ggml_reshape_4d(ctx0, Vcur, Vcur->ne[0]/n_head, n_head, Vcur->ne[1], Vcur->ne[2]);  // 对当前值张量进行形状变换
    V = ggml_cont(ctx0, ggml_permute(ctx0, V, 0, 2, 1, 3));  // 对变换后的值张量进行维度变换

    // 计算 Q * K
    struct ggml_tensor * KQ = ggml_mul_mat(ctx0, K, Q);

    // 对 KQ 进行缩放
    struct ggml_tensor * KQ_scaled = ggml_scale_inplace(ctx0, KQ, 1.0f/sqrt(float(Q->ne[0])));

    // 对 KQ 进行 Softmax 操作
    struct ggml_tensor * KQ_soft_max = ggml_soft_max_inplace(ctx0, KQ_scaled);

    // 计算 KQV
    struct ggml_tensor * KQV = ggml_mul_mat(ctx0, KQ_soft_max, ggml_cont(ctx0, ggml_transpose(ctx0, V)));

    // 对 KQV 进行合并
    struct ggml_tensor * KQV_merged = ggml_cont(ctx0, ggml_transpose(ctx0, KQV));
    KQV_merged = ggml_cont(ctx0, ggml_permute(ctx0, KQV_merged, 0, 2, 1, 3));
    # 调用 ggml_reshape_3d 函数对 KQV_merged 进行三维重塑，返回重塑后的结果
    KQV_merged = ggml_reshape_3d(ctx0, KQV_merged, KQV_merged->ne[0]*KQV_merged->ne[1], KQV_merged->ne[2], KQV_merged->ne[3]);
    # 调用 ggml_mul_mat 函数对 attn.out_w 和 KQV_merged 进行矩阵相乘，返回相乘后的结果
    KQV_merged = ggml_mul_mat(ctx0, attn.out_w, KQV_merged);
    # 调用 ggml_add_inplace 函数对 KQV_merged 和 attn.out_b 进行原地加法操作，返回加法后的结果
    KQV_merged = ggml_add_inplace(ctx0, KQV_merged, attn.out_b);
    
    # 返回最终的 KQV_merged 结果
    return KQV_merged;
# 定义一个函数，用于对输入数据进行多层感知机（MLP）和ReLU激活函数的解码操作
struct ggml_tensor * sam_decode_mask_mlp_relu_3(
    struct ggml_tensor * in,  # 输入数据
    struct ggml_tensor * w_0,  # 第一层权重
    struct ggml_tensor * b_0,  # 第一层偏置
    struct ggml_tensor * w_1,  # 第二层权重
    struct ggml_tensor * b_1,  # 第二层偏置
    struct ggml_tensor * w_2,  # 第三层权重
    struct ggml_tensor * b_2,  # 第三层偏置
    struct ggml_context * ctx0) {  # 上下文对象

    struct ggml_tensor * cur = {};  # 定义一个当前张量对象

    cur = ggml_mul_mat(ctx0, w_0, in);  # 使用上下文对象和第一层权重对输入数据进行矩阵乘法
    cur = ggml_add_inplace(ctx0, cur, b_0);  # 将第一层偏置加到当前张量中

    cur = ggml_relu_inplace(ctx0, cur);  # 对当前张量进行ReLU激活函数操作

    cur = ggml_mul_mat(ctx0, w_1, cur);  # 使用上下文对象和第二层权重对当前张量进行矩阵乘法
    cur = ggml_add_inplace(ctx0, cur, b_1);  # 将第二层偏置加到当前张量中

    cur = ggml_relu_inplace(ctx0, cur);  # 对当前张量进行ReLU激活函数操作

    cur = ggml_mul_mat(ctx0, w_2, cur);  # 使用上下文对象和第三层权重对当前张量进行矩阵乘法
    cur = ggml_add_inplace(ctx0, cur, b_2);  # 将第三层偏置加到当前张量中

    return cur;  # 返回处理后的当前张量
}

# 定义一个函数，用于对输入数据进行解码操作
bool sam_decode_mask(
    const sam_model & model,  # 输入的模型对象
    const prompt_encoder_result & prompt,  # 输入的编码结果对象
    struct ggml_tensor * pe_img,  # 输入的图像张量对象
    struct ggml_context * ctx0,  # 上下文对象
    struct ggml_cgraph  * gf,  # 计算图对象
    sam_state & state) {  # 状态对象

    const auto & hparams = model.hparams;  # 获取模型的超参数
    const auto & dec = model.dec;  # 获取模型的解码器
    const int n_img_embd = hparams.n_img_embd();  # 获取图像嵌入的数量

    struct ggml_tensor * tokens = {};  # 定义一个张量对象
    {
        // 连接输出标记
        // 参考：https://github.com/facebookresearch/segment-anything/blob/6fdee8f2727f4506cfbbe553e23b895e27956588/segment_anything/modeling/mask_decoder.py#L120
        const auto& sparse = prompt.embd_prompt_sparse;

        // 创建一个三维张量，用于存储标记
        tokens = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, dec.iou_token_w->ne[0], dec.iou_token_w->ne[1] + dec.mask_tokens_w->ne[1] + sparse->ne[1], sparse->ne[2]);

        // 定义偏移量数组
        const size_t offsets[3] = { 0, dec.iou_token_w->ne[1]*tokens->nb[1], dec.iou_token_w->ne[1]*tokens->nb[1] + dec.mask_tokens_w->ne[1]*tokens->nb[1] };
        // 将 IOU 标记权重复制到 tokens 中
        ggml_build_forward_expand(gf, ggml_cpy(ctx0, dec.iou_token_w,   ggml_view_2d(ctx0, tokens, tokens->ne[0], dec.iou_token_w->ne[1],   tokens->nb[1], offsets[0])));
        // 将 mask 标记权重复制到 tokens 中
        ggml_build_forward_expand(gf, ggml_cpy(ctx0, dec.mask_tokens_w, ggml_view_2d(ctx0, tokens, tokens->ne[0], dec.mask_tokens_w->ne[1], tokens->nb[1], offsets[1])));
        // 将稀疏提示嵌入复制到 tokens 中
        ggml_build_forward_expand(gf, ggml_cpy(ctx0, sparse,            ggml_view_2d(ctx0, tokens, tokens->ne[0], sparse->ne[1],            tokens->nb[1], offsets[2])));
        // TODO: 稀疏提示嵌入可能有多个点
    }


    // 初始化源张量和位置源张量
    struct ggml_tensor * src = {};
    struct ggml_tensor * pos_src = {};
    // 初始化源张量的维度
    int srcNE[4] = { 0, 0, 0, 0 };
    {
        // 在批处理方向上扩展每个图像数据，以便成为每个掩模
        // 参考：https://github.com/facebookresearch/segment-anything/blob/6fdee8f2727f4506cfbbe553e23b895e27956588/segment_anything/modeling/mask_decoder.py#L125
        src = ggml_new_tensor_4d(ctx0, GGML_TYPE_F32, state.embd_img->ne[0], state.embd_img->ne[1], state.embd_img->ne[2], tokens->ne[2]);
    
        src = ggml_add(ctx0,
            ggml_repeat(ctx0,
                state.embd_img,
                src),
            prompt.embd_prompt_dense);
    
        srcNE[0] = src->ne[0];
        srcNE[1] = src->ne[1];
        srcNE[2] = src->ne[2];
        srcNE[3] = src->ne[3];
    
        // 展开和排列
        // 参考：https://github.com/facebookresearch/segment-anything/blob/6fdee8f2727f4506cfbbe553e23b895e27956588/segment_anything/modeling/transformer.py#L83
        src = ggml_cont(ctx0, ggml_permute(ctx0,
            ggml_view_3d(ctx0,
                src,
                src->ne[0]*src->ne[1],
                src->ne[2],
                src->ne[3],
                src->nb[2],
                src->nb[3],
                0),
            1, 0, 2, 3));
    
        pos_src = ggml_new_tensor_4d(ctx0, GGML_TYPE_F32, pe_img->ne[0], pe_img->ne[1], pe_img->ne[2], tokens->ne[2]);
        pos_src = ggml_repeat(ctx0,
            pe_img,
            pos_src);
    
        // 展开和排列
        // 参考：https://github.com/facebookresearch/segment-anything/blob/6fdee8f2727f4506cfbbe553e23b895e27956588/segment_anything/modeling/transformer.py#L83
        pos_src = ggml_cont(ctx0, ggml_permute(ctx0,
            ggml_view_3d(ctx0,
                pos_src,
                pos_src->ne[0]*pos_src->ne[1],
                pos_src->ne[2],
                pos_src->ne[3],
                pos_src->nb[2],
                pos_src->nb[3],
                0),
            1, 0, 2, 3));
    }
    
    struct ggml_tensor * queries = tokens;
    struct ggml_tensor * keys = src;
    }
    // 创建一个指向 ggml_tensor 结构体的指针 iou_pred，指向 ggml_view_2d 函数的返回值
    struct ggml_tensor * iou_pred = ggml_view_2d(ctx0, queries, queries->ne[0], queries->ne[2], queries->nb[2], 0);
    // 定义一个整型变量 num_mask_tokens，并赋值为 4
    const int num_mask_tokens = 4; // num_multimask_outputs + 1
    // 创建一个指向 ggml_tensor 结构体的指针 mask_tokens_out，指向 ggml_view_3d 函数的返回值
    struct ggml_tensor * mask_tokens_out = ggml_view_3d(ctx0, queries, queries->ne[0], num_mask_tokens, queries->ne[2], queries->nb[1], num_mask_tokens*queries->nb[1], queries->nb[1]);

    // 放大 mask 嵌入并使用 mask tokens 预测 masks
    // 参考链接: https://github.com/facebookresearch/segment-anything/blob/6fdee8f2727f4506cfbbe553e23b895e27956588/segment_anything/modeling/mask_decoder.py#L136
    // 对 keys 进行转置操作，并将结果赋值给 keys
    keys = ggml_cont(ctx0, ggml_transpose(ctx0, keys));
    // 重新定义 keys 指向的数据，指向 ggml_view_4d 函数的返回值
    keys = ggml_view_4d(ctx0, keys, srcNE[0], srcNE[1], srcNE[2], srcNE[3], srcNE[0]*keys->nb[0], keys->nb[1], keys->nb[2], 0);
    // 创建一个指向 ggml_tensor 结构体的指针 upscaled_embedding，并赋值为空结构体
    struct ggml_tensor * upscaled_embedding = {};
    {
        // 使用 ConvTranspose2d 对 keys 进行转置卷积操作
        keys = ggml_conv_transpose_2d_p0(ctx0, dec.output_upscaling_0_w, keys, 2);
        // 分配内存空间，这里可能是一个不必要的分配
        ggml_allocr_alloc(state.allocr, keys); // TODO: This alloc shouldn't be needed
        // 对 keys 进行加法操作，使用 ggml_repeat 和 ggml_reshape_3d 函数
        keys = ggml_add_inplace(ctx0, keys, ggml_repeat(ctx0,
                                     ggml_reshape_3d(ctx0, dec.output_upscaling_0_b, 1, 1, dec.output_upscaling_0_b->ne[0]),
                                     keys));

        // 对 keys 进行 2D 归一化处理
        keys = sam_layer_norm_2d(ctx0, keys, n_img_embd, dec.output_upscaling_1_w, dec.output_upscaling_1_b, hparams.eps);

        // 使用 GELU 激活函数处理 keys
        keys = ggml_gelu_inplace(ctx0, keys);

        // 使用 ConvTranspose2d 对 keys 进行转置卷积操作
        keys = ggml_conv_transpose_2d_p0(ctx0, dec.output_upscaling_3_w, keys, 2);
        // 分配内存空间，这里可能是一个不必要的分配
        ggml_allocr_alloc(state.allocr, keys); // TODO: This alloc shouldn't be needed
        // 对 keys 进行加法操作，使用 ggml_repeat 和 ggml_reshape_3d 函数
        keys = ggml_add_inplace(ctx0, ggml_repeat(ctx0,
                                ggml_reshape_3d(ctx0, dec.output_upscaling_3_b, 1, 1, dec.output_upscaling_3_b->ne[0]),
                                keys), keys);
        // 使用 GELU 激活函数处理 keys
        keys = ggml_gelu_inplace(ctx0, keys);
        // 对 keys 进行形状重塑，得到 upscaled_embedding
        upscaled_embedding = ggml_reshape_3d(ctx0, keys, keys->ne[0]*keys->ne[1], keys->ne[2], keys->ne[3]);
        // 对 upscaled_embedding 进行转置操作，这里可能是一个不必要的操作
        upscaled_embedding = ggml_cont(ctx0, ggml_transpose(ctx0, upscaled_embedding)); // TODO: Shouldn't be needed
    }

    // 创建一个新的 3D 张量 hyper_in
    struct ggml_tensor * hyper_in = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_img_embd/2, num_mask_tokens, mask_tokens_out->ne[2]);
    # 遍历 num_mask_tokens 次
    for (int i = 0; i < num_mask_tokens; ++i) {
        # 获取当前输出层的超网络 MLP
        const auto& mlp = dec.output_hypernet_mlps[i];
        # 从 mask_tokens_out 中获取第 i 个子张量
        struct ggml_tensor * in = ggml_view_2d(ctx0, mask_tokens_out, mask_tokens_out->ne[0], mask_tokens_out->ne[2], mask_tokens_out->nb[1], i*mask_tokens_out->nb[1]);
        # 使用超网络 MLP 进行前向传播
        struct ggml_tensor * out = sam_decode_mask_mlp_relu_3(in, mlp.w_0, mlp.b_0, mlp.w_1, mlp.b_1, mlp.w_2, mlp.b_2, ctx0);
        # 构建前向传播的扩展
        ggml_build_forward_expand(gf, ggml_cpy(ctx0, out, ggml_view_2d(ctx0, hyper_in, hyper_in->ne[0], hyper_in->ne[2], hyper_in->nb[1], i*hyper_in->nb[1])));
    }

    # 计算 masks
    struct ggml_tensor * masks = ggml_mul_mat(ctx0, hyper_in, upscaled_embedding);
    # 对 masks 进行转置
    masks = ggml_cont(ctx0, ggml_transpose(ctx0, masks)); // TODO: Shouldn't be needed
    # 重塑 masks 的形状
    masks = ggml_reshape_4d(ctx0, masks, keys->ne[0], keys->ne[1], masks->ne[1], keys->ne[3]);

    # 生成 mask 质量预测
    iou_pred = sam_decode_mask_mlp_relu_3(iou_pred, dec.iou_prediction_head_0_w, dec.iou_prediction_head_0_b, dec.iou_prediction_head_1_w, dec.iou_prediction_head_1_b, dec.iou_prediction_head_2_w, dec.iou_prediction_head_2_b, ctx0);

    # 选择正确的 mask 或 masks 作为输出
    iou_pred = ggml_cpy(state.ctx, ggml_view_1d(ctx0, iou_pred, iou_pred->ne[0] - 1, iou_pred->nb[0]), state.iou_predictions);
    masks = ggml_view_4d(ctx0, masks, masks->ne[0], masks->ne[1], masks->ne[2] - 1, masks->ne[3],
                                      masks->nb[1], masks->nb[2], masks->nb[3], masks->nb[2] /* offset*/);
    masks = ggml_cpy(state.ctx, masks, state.low_res_masks);

    # 构建 masks 的前向传播扩展
    ggml_build_forward_expand(gf, masks);
    # 构建 iou_pred 的前向传播扩展
    ggml_build_forward_expand(gf, iou_pred);
    # 从图中断开低分辨率掩模节点
    ggml_disconnect_node_from_graph(state.low_res_masks);
    # 从图中断开交并比预测节点
    ggml_disconnect_node_from_graph(state.iou_predictions);
    # 返回 true
    return true;
// 写入掩模函数，根据超参数、状态、文件名写入掩模
bool sam_write_masks(const sam_hparams& hparams, int nx, int ny, const sam_state & state, const std::string & fname) {
    // 如果低分辨率掩模的第三维度为0，则返回true
    if (state.low_res_masks->ne[2] == 0) return true;
    // 如果低分辨率掩模的第三维度不等于IOU预测的第一维度，则打印错误信息并返回false
    if (state.low_res_masks->ne[2] != state.iou_predictions->ne[0]) {
        printf("Error: number of masks (%d) does not match number of iou predictions (%d)\n", (int)state.low_res_masks->ne[2], (int)state.iou_predictions->ne[0]);
        return false;
    }

    // 获取图像大小、掩模阈值、IOU阈值、稳定性分数阈值、交集阈值、并集阈值
    const int n_img_size = hparams.n_img_size();
    const float mask_threshold = hparams.mask_threshold;
    const float iou_threshold = hparams.iou_threshold;
    const float stability_score_threshold = hparams.stability_score_threshold;
    const float intersection_threshold = mask_threshold + hparams.stability_score_offset;
    const float union_threshold = mask_threshold - hparams.stability_score_offset;

    // 获取低分辨率掩模的三个维度大小
    const int ne0 = state.low_res_masks->ne[0];
    const int ne1 = state.low_res_masks->ne[1];
    const int ne2 = state.low_res_masks->ne[2];

    // 移除填充并将掩模放大到原始图像大小
    // 参考：https://github.com/facebookresearch/segment-anything/blob/efeab7296ab579d4a261e554eca80faf6b33924a/segment_anything/modeling/sam.py#L140
    const float preprocess_scale = std::max(nx, ny) / float(n_img_size);
    const int cropped_nx = int(nx / preprocess_scale + 0.5f);
    const int cropped_ny = int(ny / preprocess_scale + 0.5f);

    const float scale_x_1 = (float)ne0 / (float)n_img_size;
    const float scale_y_1 = (float)ne1 / (float)n_img_size;

    const float scale_x_2 = float(cropped_nx) / float(nx);
    const float scale_y_2 = float(cropped_ny) / float(ny);

    const auto iou_data = (float*)state.iou_predictions->data;

    // 返回true
    return true;
}

// 构建快速图函数，根据模型、状态、图像大小、点坐标构建快速图
struct ggml_cgraph  * sam_build_fast_graph(
        const sam_model     & model,
                  sam_state & state,
                        int   nx,
                        int   ny,
                  sam_point   point) {
    // 初始化 ggml_params 结构体，设置内存大小为 state.buf_compute_fast 的大小
    // 设置内存缓冲区为 state.buf_compute_fast 的数据
    // 设置不进行内存分配，因为我们使用 ggml_alloc 来精确分配内存需求
    struct ggml_init_params ggml_params = {
        /*.mem_size   =*/ state.buf_compute_fast.size(),
        /*.mem_buffer =*/ state.buf_compute_fast.data(),
        /*.no_alloc   =*/ true, // 跳过分配内存，因为我们使用 ggml_alloc 来精确分配内存需求
    };

    // 初始化 ggml_context 结构体，使用 ggml_params
    struct ggml_context * ctx0   = ggml_init(ggml_params);
    // 创建新的计算图
    struct ggml_cgraph  * gf     = ggml_new_graph(ctx0);

    // 对模型进行编码，得到编码结果
    prompt_encoder_result enc_res = sam_encode_prompt(model, ctx0, gf, state, nx, ny, point);
    // 如果编码结果中的稀疏嵌入或密集嵌入为空，则打印错误信息并返回空对象
    if (!enc_res.embd_prompt_sparse || !enc_res.embd_prompt_dense) {
        fprintf(stderr, "%s: failed to encode prompt (%f, %f)\n", __func__, point.x, point.y);
        return {};
    }

    // 获取密集位置编码的张量
    struct ggml_tensor * pe_img_dense = sam_fill_dense_pe(model, ctx0, gf, state);
    // 如果获取的密集位置编码为空，则打印错误信息并返回空对象
    if (!pe_img_dense) {
        fprintf(stderr, "%s: failed to get dense positional encoding\n", __func__);
        return {};
    }

    // 解码蒙版
    if (!sam_decode_mask(model, enc_res, pe_img_dense, ctx0, gf, state)) {
         fprintf(stderr, "%s: failed to decode mask\n", __func__);
         return {};
    }

    // 释放 ggml_context 结构体占用的内存
    ggml_free(ctx0);

    // 返回计算图
    return gf;
}
// 打印程序的使用方法
void sam_print_usage(int argc, char ** argv, const sam_params & params) {
    // 打印程序的使用方法
    fprintf(stderr, "usage: %s [options]\n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    // 打印各种选项的说明
    fprintf(stderr, "  -h, --help            show this help message and exit\n");
    fprintf(stderr, "  -s SEED, --seed SEED  RNG seed (default: -1)\n");
    fprintf(stderr, "  -t N, --threads N     number of threads to use during computation (default: %d)\n", params.n_threads);
    fprintf(stderr, "  -m FNAME, --model FNAME\n");
    fprintf(stderr, "                        model path (default: %s)\n", params.model.c_str());
    fprintf(stderr, "  -i FNAME, --inp FNAME\n");
    fprintf(stderr, "                        input file (default: %s)\n", params.fname_inp.c_str());
    fprintf(stderr, "  -o FNAME, --out FNAME\n");
    fprintf(stderr, "                        mask file name prefix (default: %s)\n", params.fname_out.c_str());
    fprintf(stderr, "SAM hyperparameters:\n");
    // 打印 SAM 的超参数说明
    fprintf(stderr, "  -mt FLOAT, --mask-threshold\n");
    fprintf(stderr, "                        mask threshold (default: %f)\n", params.mask_threshold);
    fprintf(stderr, "  -it FLOAT, --iou-threshold\n");
    fprintf(stderr, "                        iou threshold (default: %f)\n", params.iou_threshold);
    fprintf(stderr, "  -st FLOAT, --score-threshold\n");
    fprintf(stderr, "                        score threshold (default: %f)\n", params.stability_score_threshold);
    fprintf(stderr, "  -so FLOAT, --score-offset\n");
    fprintf(stderr, "                        score offset (default: %f)\n", params.stability_score_offset);
    fprintf(stderr, "  -e FLOAT, --epsilon\n");
    fprintf(stderr, "                        epsilon (default: %f)\n", params.eps);
    fprintf(stderr, "  -ed FLOAT, --epsilon-decoder-transformer\n");
    fprintf(stderr, "                        epsilon decoder transformer (default: %f)\n", params.eps_decoder_transformer);
    fprintf(stderr, "SAM prompt:\n");
}
    # 输出错误信息，提示使用 -p 或 --point-prompt 参数
    fprintf(stderr, "  -p TUPLE, --point-prompt\n");
    # 输出错误信息，提示默认的 SAM 提示点，使用 FLOAT,FLOAT 格式表示，其中 FLOAT 为浮点数
    fprintf(stderr, "                        point to be used as prompt for SAM (default: %f,%f). Must be in a format FLOAT,FLOAT \n", params.pt.x, params.pt.y);
    # 输出空行
    fprintf(stderr, "\n");
// 解析命令行参数，将解析结果存储到params中
bool sam_params_parse(int argc, char ** argv, sam_params & params) {
    // 函数体为空，需要补充解析参数的逻辑
    }

    // 解析成功，返回true
    return true;
}

// 主函数
int main(int argc, char ** argv) {
    // 记录主函数开始时间
    const int64_t t_main_start_us = ggml_time_us();

    // 初始化参数对象，设置默认模型路径
    sam_params params;
    params.model = "models/sam-vit-b/ggml-model-f16.bin";

    // 定义模型和状态对象，记录加载模型所需时间
    sam_model model;
    sam_state state;
    int64_t t_load_us = 0;

    // 解析命令行参数，如果解析失败则返回1
    if (sam_params_parse(argc, argv, params) == false) {
        return 1;
    }

    // 如果参数中未指定随机种子，则使用当前时间作为种子
    if (params.seed < 0) {
        params.seed = time(NULL);
    }
    // 打印随机种子信息
    fprintf(stderr, "%s: seed = %d\n", __func__, params.seed);

    // 加载图像文件到img0对象，加载失败则返回1
    sam_image_u8 img0;
    if (!sam_image_load_from_file(params.fname_inp, img0)) {
        fprintf(stderr, "%s: failed to load image from '%s'\n", __func__, params.fname_inp.c_str());
        return 1;
    }
    // 打印加载图像信息
    fprintf(stderr, "%s: loaded image '%s' (%d x %d)\n", __func__, params.fname_inp.c_str(), img0.nx, img0.ny);

    // 对图像进行预处理，转换为f32格式，预处理失败则返回1
    sam_image_f32 img1;
    if (!sam_image_preprocess(img0, img1)) {
        fprintf(stderr, "%s: failed to preprocess image\n", __func__);
        return 1;
    }
    // 打印预处理后图像信息
    fprintf(stderr, "%s: preprocessed image (%d x %d)\n", __func__, img1.nx, img1.ny);

    // 加载模型文件到model对象，加载失败则返回1
    {
        const int64_t t_start_us = ggml_time_us();

        if (!sam_model_load(params, model)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str());
            return 1;
        }

        t_load_us = ggml_time_us() - t_start_us;
    }
}
    {
        // 设置缓冲区大小为256MB
        static size_t buf_size = 256u*1024*1024;

        // 初始化参数结构体，包括内存大小、内存缓冲区和是否允许分配内存
        struct ggml_init_params ggml_params = {
            /*.mem_size   =*/ buf_size,  // 内存大小为buf_size
            /*.mem_buffer =*/ NULL,       // 内存缓冲区为空
            /*.no_alloc   =*/ false,      // 允许分配内存
        };

        // 使用初始化参数结构体初始化ggml上下文
        state.ctx = ggml_init(ggml_params);

        // 创建3维张量，表示嵌入图像，数据类型为GGML_TYPE_F32
        state.embd_img = ggml_new_tensor_3d(state.ctx, GGML_TYPE_F32,
                model.hparams.n_img_embd(), model.hparams.n_img_embd(), model.hparams.n_enc_out_chans);

        // 创建3维张量，表示低分辨率掩模，数据类型为GGML_TYPE_F32
        state.low_res_masks = ggml_new_tensor_3d(state.ctx, GGML_TYPE_F32,
                model.hparams.n_enc_out_chans, model.hparams.n_enc_out_chans, 3);

        // 创建1维张量，表示IOU预测，数据类型为GGML_TYPE_F32
        state.iou_predictions = ggml_new_tensor_1d(state.ctx, GGML_TYPE_F32, 3);
    }


    // 张量对齐方式为32
    static const size_t tensor_alignment = 32;
    {
        // 调整 state.buf_compute_img_enc 的大小，以容纳图像编码所需的内存空间
        state.buf_compute_img_enc.resize(ggml_tensor_overhead()*GGML_DEFAULT_GRAPH_SIZE + ggml_graph_overhead());
        // 创建一个新的内存分配器，用于测量内存需求
        state.allocr = ggml_allocr_new_measure(tensor_alignment);
        // 对图像进行编码，得到测量结果
        struct ggml_cgraph * gf_measure = sam_encode_image(model, state, img1);
        // 如果编码失败，则打印错误信息并返回 1
        if (!gf_measure) {
            fprintf(stderr, "%s: failed to encode image\n", __func__);
            return 1;
        }
    
        // 计算实际分配的内存大小，包括对齐
        size_t alloc_size = ggml_allocr_alloc_graph(state.allocr, gf_measure) + tensor_alignment;
        // 释放内存分配器
        ggml_allocr_free(state.allocr);
    
        // 根据实际内存需求重新调整 state.buf_alloc_img_enc 的大小
        state.buf_alloc_img_enc.resize(alloc_size);
        // 使用准确的内存需求重新创建内存分配器
        state.allocr = ggml_allocr_new(state.buf_alloc_img_enc.data(), state.buf_alloc_img_enc.size(), tensor_alignment);
    
        // 重置内存分配器，准备计算图像编码
        ggml_allocr_reset(state.allocr);
    
        // 对图像进行编码，得到最终结果
        struct ggml_cgraph  * gf = sam_encode_image(model, state, img1);
        // 如果编码失败，则打印错误信息并返回 1
        if (!gf) {
            fprintf(stderr, "%s: failed to encode image\n", __func__);
            return 1;
        }
    
        // 分配图像编码所需的内存空间
        ggml_allocr_alloc_graph(state.allocr, gf);
        // 使用多线程计算图像编码
        ggml_graph_compute_helper(state.work_buffer, gf, params.n_threads);
        // 打印图像编码结果
        print_t_f32("embd_img", state.embd_img);
        // 释放内存分配器
        ggml_allocr_free(state.allocr);
        // 将内存分配器置为空指针
        state.allocr = NULL;
        // 清空工作缓冲区
        state.work_buffer.clear();
    }
    {
        // 调整 state.buf_compute_fast 的大小，以容纳计算快速图形所需的内存
        state.buf_compute_fast.resize(ggml_tensor_overhead()*GGML_DEFAULT_GRAPH_SIZE + ggml_graph_overhead());
        // 创建一个新的内存分配器，用于测量内存需求
        state.allocr = ggml_allocr_new_measure(tensor_alignment);

        // TODO: 更多变化的提示
        // 输出提示信息，显示参数的 x 和 y 值
        fprintf(stderr, "prompt: (%f, %f)\n", params.pt.x, params.pt.y);

        // 测量图形的内存需求
        struct ggml_cgraph  * gf_measure = sam_build_fast_graph(model, state, img0.nx, img0.ny, params.pt);
        if (!gf_measure) {
            // 如果构建快速图形失败，则输出错误信息并返回 1
            fprintf(stderr, "%s: failed to build fast graph to measure\n", __func__);
            return 1;
        }

        // 计算分配给图形的内存大小，并加上对齐的额外内存
        size_t alloc_size = ggml_allocr_alloc_graph(state.allocr, gf_measure) + tensor_alignment;
        ggml_allocr_free(state.allocr);

        // 重新调整 state.buf_alloc_fast 的大小，以容纳准确的内存需求
        state.buf_alloc_fast.resize(alloc_size);
        // 使用准确的内存需求创建新的分配器
        state.allocr = ggml_allocr_new(state.buf_alloc_fast.data(), state.buf_alloc_fast.size(), tensor_alignment);

        // 重置分配器，准备计算上面测量到的准确内存需求的图形
        ggml_allocr_reset(state.allocr);

        // 构建快速图形
        struct ggml_cgraph  * gf = sam_build_fast_graph(model, state, img0.nx, img0.ny, params.pt);
        if (!gf) {
            // 如果构建快速图形失败，则输出错误信息并返回 1
            fprintf(stderr, "%s: failed to build fast graph\n", __func__);
            return 1;
        }

        // 分配图形所需的内存
        ggml_allocr_alloc_graph(state.allocr, gf);

        // 使用状态的工作缓冲区和参数中的线程数计算图形
        ggml_graph_compute_helper(state.work_buffer, gf, params.n_threads);

        // 释放分配器
        ggml_allocr_free(state.allocr);
        state.allocr = NULL;
    }

    // 如果无法将结果写入文件，则输出错误信息并返回 1
    if (!sam_write_masks(model.hparams, img0.nx, img0.ny, state, params.fname_out)) {
        fprintf(stderr, "%s: failed to write masks\n", __func__);
        return 1;
    }

    // 报告计时信息
    {
        // 记录主函数结束时间
        const int64_t t_main_end_us = ggml_time_us();

        // 输出加载时间
        fprintf(stderr, "\n\n");
        fprintf(stderr, "%s:     load time = %8.2f ms\n", __func__, t_load_us/1000.0f);
        // 输出总时间
        fprintf(stderr, "%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0f);
    }

    // 释放模型上下文
    ggml_free(model.ctx);

    // 返回成功状态码
    return 0;
# 闭合前面的函数定义
```