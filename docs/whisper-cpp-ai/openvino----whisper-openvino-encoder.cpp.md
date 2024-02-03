# `whisper.cpp\openvino\whisper-openvino-encoder.cpp`

```cpp
#include "openvino/whisper-openvino-encoder.h"
#include "ggml.h"
#include <openvino/openvino.hpp>
#include <iostream>

// 定义一个结构体，用于存储 OpenVINO 上下文信息
struct whisper_openvino_context {
    ov::InferRequest inferRequest; // OpenVINO 推理请求对象
};

// 初始化 OpenVINO 上下文
struct whisper_openvino_context * whisper_openvino_init(const char* path_model,
    const char* device,
    const char* cache_dir)
{
    // 检查模型路径和设备是否为空
    if (!path_model || !device) {
        fprintf(stderr, "%s: path_model and/or device is null\n", __func__);
        return nullptr;
    }

    // 打印初始化信息
    fprintf(stderr, "%s: path_model = %s, device = %s, cache_dir = %s\n",
        __func__, path_model, device, cache_dir ? cache_dir : "(not set)");

    // 创建 OpenVINO 上下文对象
    whisper_openvino_context *context = new whisper_openvino_context;
    try {
        ov::Core core;

        // 如果存在缓存目录，设置设备特定 'blobs' 缓存以加速编译模型的调用
        if (cache_dir) {
            core.set_property(ov::cache_dir(cache_dir));
        }

        // 从磁盘读取 OpenVINO 编码器 IR (.xml/.bin) 文件，生成 ov::Model 对象
        std::shared_ptr<ov::Model> model = core.read_model(path_model);

        // 根据设备编译模型，生成编译后的模型对象
        auto compiledModel = core.compile_model(model, device);

        // 从编译后的模型对象创建推理请求对象，用于后续触发推理执行
        context->inferRequest = compiledModel.create_infer_request();
    }
    catch (const std::exception& error) {
        // 捕获异常并打印错误信息
        std::cout << "in openvino encoder compile routine: exception: " << error.what() << std::endl;
        delete context;
        context = nullptr;
    }

    return context;
}

// 释放 OpenVINO 上下文资源
void whisper_openvino_free(struct whisper_openvino_context * ctx) {
    if( ctx ) {
        delete ctx;
    }
}

// OpenVINO 编码函数
int whisper_openvino_encode(
    whisper_openvino_context* ctx,
    ggml_tensor* mel,
    ggml_tensor* out) {
    // 检查传入的指针是否为空，如果有任何一个为空则输出错误信息并返回0
    if (!ctx || !mel || !out) {
        fprintf(stderr, "%s: Error! ctx / mel / out is null\n", __func__);
        return 0;
    }

    // 检查传入的 mel ggml_tensor 是否为二维，如果不是则输出错误信息并返回0
    if (ggml_n_dims(mel) != 2) {
        fprintf(stderr, "%s: Error! mel ggml_tensor expected to have n_dims=2, but it has n_dims=%d\n",
            __func__, ggml_n_dims(mel));
        return 0;
    }

    // 检查传入的 out ggml_tensor 是否为二维，如果不是则输出错误信息并返回0
    if (ggml_n_dims(out) != 2) {
        fprintf(stderr, "%s: Error! out ggml_tensor expected to have n_dims=2, but it has n_dims=%d\n",
            __func__, ggml_n_dims(out));
        return 0;
    }

    try {

        // 将传入的 mel ggml_tensor 包装为 OpenVINO Tensor 对象，并设置为推断请求的输入张量
        {
            // 注意，我们以与 ne / nb 数组中列出的顺序相反的方式填充形状和步幅维度
            ov::Shape input_shape = { 1, (unsigned long long)mel->ne[1], (unsigned long long)mel->ne[0] };
            ov::Strides input_strides = { mel->nb[2], mel->nb[1], mel->nb[0] };
            ov::Tensor input_tensor(ov::element::f32, input_shape, mel->data, input_strides);
            ctx->inferRequest.set_input_tensor(input_tensor);
        }

        // 将传入的 out ggml_tensor 包装为 OpenVINO Tensor 对象，并设置为推断请求的输出张量
        {
            // 注意，我们以与 ne / nb 数组中列出的顺序相反的方式填充形状和步幅维度
            ov::Shape output_shape = { 1, (unsigned long long)out->ne[1], (unsigned long long)out->ne[0] };
            ov::Strides output_strides = { out->nb[2], out->nb[1], out->nb[0] };
            ov::Tensor out_tensor(ov::element::f32, output_shape, out->data, output_strides);
            ctx->inferRequest.set_output_tensor(out_tensor);
        }

        // 运行推断
        ctx->inferRequest.infer();
    }
    catch (const std::exception& error) {
        // 捕获异常并输出异常信息
        std::cout << "in openvino encode inference execution routine: exception: " << error.what() << std::endl;
        return 0;
    }
    # 返回整数值1
    return 1;
# 闭合之前的代码块
```