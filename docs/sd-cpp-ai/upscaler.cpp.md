# `stable-diffusion.cpp\upscaler.cpp`

```
// 包含头文件 esrgan.hpp
// 包含头文件 ggml_extend.hpp
// 包含头文件 model.h
// 包含头文件 stable-diffusion.h
#include "esrgan.hpp"
#include "ggml_extend.hpp"
#include "model.h"
#include "stable-diffusion.h"

// 定义结构体 UpscalerGGML
struct UpscalerGGML {
    ggml_backend_t backend    = NULL;  // 通用后端
    ggml_type model_data_type = GGML_TYPE_F16; // 模型数据类型为 GGML_TYPE_F16
    ESRGAN esrgan_upscaler; // ESRGAN 实例
    std::string esrgan_path; // ESRGAN 模型路径
    int n_threads; // 线程数

    // 构造函数，初始化线程数
    UpscalerGGML(int n_threads)
        : n_threads(n_threads) {
    }

    // 从文件加载模型
    bool load_from_file(const std::string& esrgan_path) {
#ifdef SD_USE_CUBLAS
        LOG_DEBUG("Using CUDA backend"); // 使用 CUDA 后端
        backend = ggml_backend_cuda_init(0); // 初始化 CUDA 后端
#endif
#ifdef SD_USE_METAL
        LOG_DEBUG("Using Metal backend"); // 使用 Metal 后端
        ggml_metal_log_set_callback(ggml_log_callback_default, nullptr); // 设置 Metal 日志回调
        backend = ggml_backend_metal_init(); // 初始化 Metal 后端
#endif

        // 如果没有指定后端，则使用 CPU 后端
        if (!backend) {
            LOG_DEBUG("Using CPU backend"); // 使用 CPU 后端
            backend = ggml_backend_cpu_init(); // 初始化 CPU 后端
        }
        LOG_INFO("Upscaler weight type: %s", ggml_type_name(model_data_type)); // 打印模型数据类型
        // 从文件加载 ESRGAN 模型
        if (!esrgan_upscaler.load_from_file(esrgan_path, backend)) {
            return false;
        }
        return true;
    }
};

// 定义结构体 upscaler_ctx_t
struct upscaler_ctx_t {
    UpscalerGGML* upscaler = NULL; // UpscalerGGML 实例指针
};

// 创建新的 upscaler 上下文
upscaler_ctx_t* new_upscaler_ctx(const char* esrgan_path_c_str,
                                 int n_threads,
                                 enum sd_type_t wtype) {
    upscaler_ctx_t* upscaler_ctx = (upscaler_ctx_t*)malloc(sizeof(upscaler_ctx_t)); // 分配内存
    if (upscaler_ctx == NULL) {
        return NULL;
    }
    std::string esrgan_path(esrgan_path_c_str); // 将 C 字符串转换为 C++ 字符串

    upscaler_ctx->upscaler = new UpscalerGGML(n_threads); // 创建 UpscalerGGML 实例
    if (upscaler_ctx->upscaler == NULL) {
        return NULL;
    }

    // 从文件加载模型到 upscaler 上下文
    if (!upscaler_ctx->upscaler->load_from_file(esrgan_path)) {
        delete upscaler_ctx->upscaler;
        upscaler_ctx->upscaler = NULL;
        free(upscaler_ctx);
        return NULL;
    }
    return upscaler_ctx;
}

// 图像放大函数
sd_image_t upscale(upscaler_ctx_t* upscaler_ctx, sd_image_t input_image, uint32_t upscale_factor) {
    # 调用upscaler_ctx指针所指向的upscaler对象的upscale方法，传入input_image和upscale_factor作为参数，并返回结果
    return upscaler_ctx->upscaler->upscale(input_image, upscale_factor);
}
释放上采样器上下文的内存空间
void free_upscaler_ctx(upscaler_ctx_t* upscaler_ctx) {
    // 检查上采样器是否已经被创建
    if (upscaler_ctx->upscaler != NULL) {
        // 删除上采样器对象
        delete upscaler_ctx->upscaler;
        // 将上采样器指针置为空
        upscaler_ctx->upscaler = NULL;
    }
    // 释放上采样器上下文的内存空间
    free(upscaler_ctx);
}
```