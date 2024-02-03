# `stable-diffusion.cpp\stable-diffusion.h`

```
#ifndef __STABLE_DIFFUSION_H__
#define __STABLE_DIFFUSION_H__

#if defined(_WIN32) || defined(__CYGWIN__)
#ifndef SD_BUILD_SHARED_LIB
#define SD_API
#else
#ifdef SD_BUILD_DLL
#define SD_API __declspec(dllexport)
#else
#define SD_API __declspec(dllimport)
#endif
#endif
#else
#if __GNUC__ >= 4
#define SD_API __attribute__((visibility("default")))
#else
#define SD_API
#endif
#endif

#ifdef __cplusplus
extern "C" {
#endif

#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>
#include <string.h>

enum rng_type_t {
    STD_DEFAULT_RNG,
    CUDA_RNG
};

enum sample_method_t {
    EULER_A,
    EULER,
    HEUN,
    DPM2,
    DPMPP2S_A,
    DPMPP2M,
    DPMPP2Mv2,
    LCM,
    N_SAMPLE_METHODS
};

enum schedule_t {
    DEFAULT,
    DISCRETE,
    KARRAS,
    N_SCHEDULES
};

// same as enum ggml_type
enum sd_type_t {
    SD_TYPE_F32  = 0,
    SD_TYPE_F16  = 1,
    SD_TYPE_Q4_0 = 2,
    SD_TYPE_Q4_1 = 3,
    // SD_TYPE_Q4_2 = 4, support has been removed
    // SD_TYPE_Q4_3 (5) support has been removed
    SD_TYPE_Q5_0 = 6,
    SD_TYPE_Q5_1 = 7,
    SD_TYPE_Q8_0 = 8,
    SD_TYPE_Q8_1 = 9,
    // k-quantizations
    SD_TYPE_Q2_K    = 10,
    SD_TYPE_Q3_K    = 11,
    SD_TYPE_Q4_K    = 12,
    SD_TYPE_Q5_K    = 13,
    SD_TYPE_Q6_K    = 14,
    SD_TYPE_Q8_K    = 15,
    SD_TYPE_IQ2_XXS = 16,
    SD_TYPE_I8,
    SD_TYPE_I16,
    SD_TYPE_I32,
    SD_TYPE_COUNT,
};

// 返回指定 sd_type_t 类型的名称
SD_API const char* sd_type_name(enum sd_type_t type);

enum sd_log_level_t {
    SD_LOG_DEBUG,
    SD_LOG_INFO,
    SD_LOG_WARN,
    SD_LOG_ERROR
};

// 定义日志回调函数类型
typedef void (*sd_log_cb_t)(enum sd_log_level_t level, const char* text, void* data);

// 设置日志回调函数
SD_API void sd_set_log_callback(sd_log_cb_t sd_log_cb, void* data);
// 获取物理核心数量
SD_API int32_t get_num_physical_cores();
// 获取系统信息
SD_API const char* sd_get_system_info();

// 定义图像结构体
typedef struct {
    uint32_t width;
    uint32_t height;
    uint32_t channel;
    uint8_t* data;
} sd_image_t;

// 定义上下文结构体
typedef struct sd_ctx_t sd_ctx_t;
# 创建一个新的 SD 上下文对象，用于处理各种操作
# 参数包括模型路径、VAE 路径、TAESD 路径、控制网络路径、LoRa 模型目录、嵌入目录、解码标志、平铺标志、立即释放参数标志、线程数、权重类型、随机数生成器类型、调度类型、保留控制网络在 CPU 上的标志
SD_API sd_ctx_t* new_sd_ctx(const char* model_path,
                            const char* vae_path,
                            const char* taesd_path,
                            const char* control_net_path_c_str,
                            const char* lora_model_dir,
                            const char* embed_dir_c_str,
                            bool vae_decode_only,
                            bool vae_tiling,
                            bool free_params_immediately,
                            int n_threads,
                            enum sd_type_t wtype,
                            enum rng_type_t rng_type,
                            enum schedule_t s,
                            bool keep_control_net_cpu);

# 释放 SD 上下文对象所占用的内存
SD_API void free_sd_ctx(sd_ctx_t* sd_ctx);

# 将文本转换为图像
# 参数包括 SD 上下文对象、提示文本、负面提示文本、剪辑跳过数、配置比例、宽度、高度、采样方法、采样步数、种子、批次计数、控制条件图像、控制强度
SD_API sd_image_t* txt2img(sd_ctx_t* sd_ctx,
                           const char* prompt,
                           const char* negative_prompt,
                           int clip_skip,
                           float cfg_scale,
                           int width,
                           int height,
                           enum sample_method_t sample_method,
                           int sample_steps,
                           int64_t seed,
                           int batch_count,
                           const sd_image_t* control_cond,
                           float control_strength);

# 将图像转换为图像
# 参数包括 SD 上下文对象、初始图像、提示文本、负面提示文本、剪辑跳过数、配置比例、宽度、高度、采样方法、采样步数、强度、种子、批次计数
SD_API sd_image_t* img2img(sd_ctx_t* sd_ctx,
                           sd_image_t init_image,
                           const char* prompt,
                           const char* negative_prompt,
                           int clip_skip,
                           float cfg_scale,
                           int width,
                           int height,
                           enum sample_method_t sample_method,
                           int sample_steps,
                           float strength,
                           int64_t seed,
                           int batch_count);
// 定义结构体 upscaler_ctx_t
typedef struct upscaler_ctx_t upscaler_ctx_t;

// 创建一个新的 upscaler 上下文，传入 ESRGAN 路径、线程数和图像类型
SD_API upscaler_ctx_t* new_upscaler_ctx(const char* esrgan_path,
                                        int n_threads,
                                        enum sd_type_t wtype);

// 释放 upscaler 上下文
SD_API void free_upscaler_ctx(upscaler_ctx_t* upscaler_ctx);

// 对输入图像进行放大，传入 upscaler 上下文、输入图像和放大因子
SD_API sd_image_t upscale(upscaler_ctx_t* upscaler_ctx, sd_image_t input_image, uint32_t upscale_factor);

// 将输入路径的图像转换为指定类型的图像，传入输入路径、VAE 路径、输出路径和输出类型
SD_API bool convert(const char* input_path, const char* vae_path, const char* output_path, sd_type_t output_type);

// 结束 C++ 的声明
#ifdef __cplusplus
}
#endif

// 结束头文件的声明
#endif  // __STABLE_DIFFUSION_H__
```