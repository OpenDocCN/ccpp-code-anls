# `stable-diffusion.cpp\stable-diffusion.cpp`

```
#include "ggml_extend.hpp"

#include "model.h"
#include "rng.hpp"
#include "rng_philox.hpp"
#include "stable-diffusion.h"
#include "util.h"

#include "clip.hpp"
#include "control.hpp"
#include "denoiser.hpp"
#include "esrgan.hpp"
#include "lora.hpp"
#include "tae.hpp"
#include "unet.hpp"
#include "vae.hpp"

// 定义模型版本字符串数组
const char* model_version_to_str[] = {
    "1.x",
    "2.x",
    "XL",
};

// 定义采样方法字符串数组
const char* sampling_methods_str[] = {
    "Euler A",
    "Euler",
    "Heun",
    "DPM2",
    "DPM++ (2s)",
    "DPM++ (2M)",
    "modified DPM++ (2M)",
    "LCM",
};

/*================================================== Helper Functions ================================================*/

// 计算 alphas_cumprod 数组
void calculate_alphas_cumprod(float* alphas_cumprod,
                              float linear_start = 0.00085f,
                              float linear_end   = 0.0120,
                              int timesteps      = TIMESTEPS) {
    // 计算线性起始和结束的平方根
    float ls_sqrt = sqrtf(linear_start);
    float le_sqrt = sqrtf(linear_end);
    float amount  = le_sqrt - ls_sqrt;
    float product = 1.0f;
    // 遍历时间步长，计算 alphas_cumprod 数组
    for (int i = 0; i < timesteps; i++) {
        float beta = ls_sqrt + amount * ((float)i / (timesteps - 1));
        product *= 1.0f - powf(beta, 2.0f);
        alphas_cumprod[i] = product;
    }
}

/*=============================================== StableDiffusionGGML ================================================*/

// 稳定扩散 GGML 类
class StableDiffusionGGML {
public:
    // 版本号
    SDVersion version;
    // 是否仅解码 VAE
    bool vae_decode_only         = false;
    // 是否立即释放参数
    bool free_params_immediately = false;

    // 随机数生成器
    std::shared_ptr<RNG> rng = std::make_shared<STDDefaultRNG>();
    // 线程数
    int n_threads            = -1;
    // 缩放因子
    float scale_factor       = 0.18215f;

    // 条件阶段模型
    FrozenCLIPEmbedderWithCustomWords cond_stage_model;
    // 扩散模型
    UNetModel diffusion_model;
    // 自编码器 KL 损失模型
    AutoEncoderKL first_stage_model;
    // 是否使用小型自编码器
    bool use_tiny_autoencoder = false;
    // 是否使用 VAE 平铺
    bool vae_tiling           = false;

    // 张量映射
    std::map<std::string, struct ggml_tensor*> tensors;

    // LORA 模型目录
    std::string lora_model_dir;
    // LORA 名称 => 倍增器
    // 当前 LoRa 状态的哈希表，存储 LoRa 设备的状态信息
    std::unordered_map<std::string, float> curr_lora_state;
    // 存储 LoRa 设备的模型信息
    std::map<std::string, LoraModel> loras;

    // 创建共享指针指向 Denoiser 接口的实例，使用 CompVisDenoiser 实现
    std::shared_ptr<Denoiser> denoiser = std::make_shared<CompVisDenoiser>();
    // 通用后端对象
    ggml_backend_t backend = NULL;
    // 模型数据类型
    ggml_type model_data_type = GGML_TYPE_COUNT;

    // 第一个阶段的 TinyAutoEncoder 实例
    TinyAutoEncoder tae_first_stage;
    // TinyAutoEncoder 模型路径
    std::string taesd_path;

    // 控制网络对象
    ControlNet control_net;

    // 默认构造函数
    StableDiffusionGGML() = default;

    // 带参数的构造函数
    StableDiffusionGGML(int n_threads,
                        bool vae_decode_only,
                        bool free_params_immediately,
                        std::string lora_model_dir,
                        rng_type_t rng_type)
        : n_threads(n_threads),
          vae_decode_only(vae_decode_only),
          free_params_immediately(free_params_immediately),
          lora_model_dir(lora_model_dir) {
        // 设置第一个阶段模型的解码标志
        first_stage_model.decode_only = vae_decode_only;
        tae_first_stage.decode_only = vae_decode_only;
        // 根据随机数生成器类型选择不同的随机数生成器
        if (rng_type == STD_DEFAULT_RNG) {
            rng = std::make_shared<STDDefaultRNG>();
        } else if (rng_type == CUDA_RNG) {
            rng = std::make_shared<PhiloxRNG>();
        }
    }

    // 析构函数
    ~StableDiffusionGGML() {
        // 释放后端资源
        ggml_backend_free(backend);
    }

    // 从文件加载模型数据
    bool load_from_file(const std::string& model_path,
                        const std::string& vae_path,
                        const std::string control_net_path,
                        const std::string embeddings_path,
                        const std::string& taesd_path,
                        bool vae_tiling_,
                        ggml_type wtype,
                        schedule_t schedule,
                        bool control_net_cpu) {
        // 判断是否使用 TinyAutoEncoder
        use_tiny_autoencoder = taesd_path.size() > 0;
#ifdef SD_USE_CUBLAS
        // 如果定义了 SD_USE_CUBLAS，则使用 CUDA 后端
        LOG_DEBUG("Using CUDA backend");
        // 初始化 CUDA 后端
        backend = ggml_backend_cuda_init(0);
#endif
#ifdef SD_USE_METAL
        // 如果定义了 SD_USE_METAL，则使用 Metal 后端
        LOG_DEBUG("Using Metal backend");
        // 设置 Metal 后端的日志回调函数
        ggml_metal_log_set_callback(ggml_log_callback_default, nullptr);
        // 初始化 Metal 后端
        backend = ggml_backend_metal_init();
#endif

        // 如果没有选择任何后端，则使用 CPU 后端
        if (!backend) {
            LOG_DEBUG("Using CPU backend");
            // 初始化 CPU 后端
            backend = ggml_backend_cpu_init();
        }
#ifdef SD_USE_FLASH_ATTENTION
#if defined(SD_USE_CUBLAS) || defined(SD_USE_METAL)
        // 如果同时使用了 CUDA 或 Metal 后端，则不支持 Flash Attention
        LOG_WARN("Flash Attention not supported with GPU Backend");
#else
        // 否则启用 Flash Attention
        LOG_INFO("Flash Attention enabled");
#endif
    }
    // 检查是否使用了 v 参数化来计算 sd2
    bool is_using_v_parameterization_for_sd2(ggml_context* work_ctx) {
        // 创建一个 4 维张量 x_t，数据类型为 float32，大小为 8x8x4x1
        struct ggml_tensor* x_t = ggml_new_tensor_4d(work_ctx, GGML_TYPE_F32, 8, 8, 4, 1);
        // 设置 x_t 中所有元素的值为 0.5
        ggml_set_f32(x_t, 0.5);
        // 创建一个 4 维张量 c，数据类型为 float32，大小为 1024x2x1x1
        struct ggml_tensor* c = ggml_new_tensor_4d(work_ctx, GGML_TYPE_F32, 1024, 2, 1, 1);
        // 设置 c 中所有元素的值为 0.5

        // 创建一个 1 维张量 timesteps，数据类型为 float32，大小为 1
        struct ggml_tensor* timesteps = ggml_new_tensor_1d(work_ctx, GGML_TYPE_F32, 1);                                     // [N, ]
        // 创建一个时间步长嵌入张量 t_emb，传入 timesteps 和扩散模型的通道数，大小为 N x model_channels
        struct ggml_tensor* t_emb     = new_timestep_embedding(work_ctx, NULL, timesteps, diffusion_model.model_channels);  // [N, model_channels]

        // 记录开始时间
        int64_t t0 = ggml_time_ms();
        // 设置 timesteps 中的值为 999
        ggml_set_f32(timesteps, 999);
        // 设置时间步长嵌入张量
        set_timestep_embedding(timesteps, t_emb, diffusion_model.model_channels);
        // 复制 x_t 到 out
        struct ggml_tensor* out = ggml_dup_tensor(work_ctx, x_t);
        // 分配计算缓冲区
        std::vector<struct ggml_tensor*> controls;
        diffusion_model.alloc_compute_buffer(x_t, c, controls, t_emb);
        // 计算结果
        diffusion_model.compute(out, n_threads, x_t, NULL, c, controls, 1.0f, t_emb);
        // 释放计算缓冲区
        diffusion_model.free_compute_buffer();

        // 计算结果的差值
        double result = 0.f;
        {
            float* vec_x   = (float*)x_t->data;
            float* vec_out = (float*)out->data;

            int64_t n = ggml_nelements(out);

            for (int i = 0; i < n; i++) {
                result += ((double)vec_out[i] - (double)vec_x[i]);
            }
            result /= n;
        }
        // 记录结束时间
        int64_t t1 = ggml_time_ms();
        // 输出调试信息
        LOG_DEBUG("check is_using_v_parameterization_for_sd2, taking %.2fs", (t1 - t0) * 1.0f / 1000);
        // 返回结果是否小于 -1
        return result < -1;
    }
    // 应用 LoRa 模型，根据给定的 LoRa 名称和倍增因子
    void apply_lora(const std::string& lora_name, float multiplier) {
        // 获取当前时间戳
        int64_t t0 = ggml_time_ms();
        // 拼接安全张量文件路径
        std::string st_file_path = path_join(lora_model_dir, lora_name + ".safetensors");
        // 拼接检查点文件路径
        std::string ckpt_file_path = path_join(lora_model_dir, lora_name + ".ckpt");
        std::string file_path;
        // 如果安全张量文件存在，则使用安全张量文件路径
        if (file_exists(st_file_path)) {
            file_path = st_file_path;
        } 
        // 如果检查点文件存在，则使用检查点文件路径
        else if (file_exists(ckpt_file_path)) {
            file_path = ckpt_file_path;
        } 
        // 如果两者都不存在，则记录警告信息并返回
        else {
            LOG_WARN("can not find %s or %s for lora %s", st_file_path.c_str(), ckpt_file_path.c_str(), lora_name.c_str());
            return;
        }
        // 根据文件路径创建 LoRa 模型对象
        LoraModel lora(file_path);
        // 从文件加载 LoRa 张量数据到指定后端
        if (!lora.load_from_file(backend)) {
            LOG_WARN("load lora tensors from %s failed", file_path.c_str());
            return;
        }

        // 设置 LoRa 模型的倍增因子
        lora.multiplier = multiplier;
        // 应用 LoRa 模型到指定张量和线程数
        lora.apply(tensors, n_threads);
        // 将 LoRa 模型存储到 LoRa 字典中
        loras[lora_name] = lora;
        // 释放 LoRa 模型的参数缓冲区
        lora.free_params_buffer();

        // 获取结束时间戳
        int64_t t1 = ggml_time_ms();

        // 记录信息，显示 LoRa 模型应用所花费的时间
        LOG_INFO("lora '%s' applied, taking %.2fs",
                 lora_name.c_str(),
                 (t1 - t0) * 1.0f / 1000);
    }
    // 应用 LoRA 参数到模型状态中
    void apply_loras(const std::unordered_map<std::string, float>& lora_state) {
        // 如果 LoRA 参数不为空且模型数据类型不是 GGML_TYPE_F16 和 GGML_TYPE_F32
        if (lora_state.size() > 0 && model_data_type != GGML_TYPE_F16 && model_data_type != GGML_TYPE_F32) {
            // 输出警告信息
            LOG_WARN("In quantized models when applying LoRA, the images have poor quality.");
        }
        // 创建存储 LoRA 参数差异的字典
        std::unordered_map<std::string, float> lora_state_diff;
        // 遍历传入的 LoRA 参数
        for (auto& kv : lora_state) {
            // 获取 LoRA 参数的名称和乘数
            const std::string& lora_name = kv.first;
            float multiplier             = kv.second;

            // 如果当前 LoRA 状态中存在该参数
            if (curr_lora_state.find(lora_name) != curr_lora_state.end()) {
                // 获取当前 LoRA 参数的乘数
                float curr_multiplier = curr_lora_state[lora_name];
                // 计算乘数的差异
                float multiplier_diff = multiplier - curr_multiplier;
                // 如果乘数差异不为零
                if (multiplier_diff != 0.f) {
                    // 将乘数差异存入差异字典中
                    lora_state_diff[lora_name] = multiplier_diff;
                }
            } else {
                // 如果当前 LoRA 状态中不存在该参数，直接存入差异字典中
                lora_state_diff[lora_name] = multiplier;
            }
        }

        // 遍历 LoRA 参数差异字典，应用每个参数的差异
        for (auto& kv : lora_state_diff) {
            apply_lora(kv.first, kv.second);
        }

        // 更新当前 LoRA 状态为传入的 LoRA 参数
        curr_lora_state = lora_state;
    }

    // ldm.models.diffusion.ddpm.LatentDiffusion.get_first_stage_encoding
    // 获取第一阶段的编码，根据给定的矩阵 moments
    ggml_tensor* get_first_stage_encoding(ggml_context* work_ctx, ggml_tensor* moments) {
        // 创建一个与 moments 尺寸相同的潜在变量 latent
        ggml_tensor* latent       = ggml_new_tensor_4d(work_ctx, moments->type, moments->ne[0], moments->ne[1], moments->ne[2] / 2, moments->ne[3]);
        // 复制潜在变量到 noise
        struct ggml_tensor* noise = ggml_dup_tensor(work_ctx, latent);
        // 为 noise 设置随机数
        ggml_tensor_set_f32_randn(noise, rng);
        // 从文件中加载 noise 数据
        // noise = load_tensor_from_file(work_ctx, "noise.bin");
        {
            float mean   = 0;
            float logvar = 0;
            float value  = 0;
            float std_   = 0;
            // 遍历潜在变量的维度
            for (int i = 0; i < latent->ne[3]; i++) {
                for (int j = 0; j < latent->ne[2]; j++) {
                    for (int k = 0; k < latent->ne[1]; k++) {
                        for (int l = 0; l < latent->ne[0]; l++) {
                            // 获取 moments 中的均值和对数方差
                            mean   = ggml_tensor_get_f32(moments, l, k, j, i);
                            logvar = ggml_tensor_get_f32(moments, l, k, j + (int)latent->ne[2], i);
                            // 限制对数方差的范围
                            logvar = std::max(-30.0f, std::min(logvar, 20.0f));
                            // 计算标准差
                            std_   = std::exp(0.5f * logvar);
                            // 计算最终值
                            value  = mean + std_ * ggml_tensor_get_f32(noise, l, k, j, i);
                            value  = value * scale_factor;
                            // printf("%d %d %d %d -> %f\n", i, j, k, l, value);
                            // 将计算结果设置到潜在变量中
                            ggml_tensor_set_f32(latent, value, l, k, j, i);
                        }
                    }
                }
            }
        }
        // 返回潜在变量
        return latent;
    }

    // 编码第一阶段
    ggml_tensor* encode_first_stage(ggml_context* work_ctx, ggml_tensor* x) {
        // 调用 compute_first_stage 函数，传入 x 和 false
        return compute_first_stage(work_ctx, x, false);
    }

    // 解码第一阶段
    ggml_tensor* decode_first_stage(ggml_context* work_ctx, ggml_tensor* x) {
        // 调用 compute_first_stage 函数，传入 x 和 true
        return compute_first_stage(work_ctx, x, true);
    }
};

/*================================================= SD API ==================================================*/

// 定义一个结构体 sd_ctx_t，包含一个指向 StableDiffusionGGML 对象的指针
struct sd_ctx_t {
    StableDiffusionGGML* sd = NULL;
};

// 创建一个新的 sd_ctx_t 对象，并初始化其中的 StableDiffusionGGML 对象
sd_ctx_t* new_sd_ctx(const char* model_path_c_str,
                     const char* vae_path_c_str,
                     const char* taesd_path_c_str,
                     const char* control_net_path_c_str,
                     const char* lora_model_dir_c_str,
                     const char* embed_dir_c_str,
                     bool vae_decode_only,
                     bool vae_tiling,
                     bool free_params_immediately,
                     int n_threads,
                     enum sd_type_t wtype,
                     enum rng_type_t rng_type,
                     enum schedule_t s,
                     bool keep_control_net_cpu) {
    // 分配内存以存储 sd_ctx_t 对象
    sd_ctx_t* sd_ctx = (sd_ctx_t*)malloc(sizeof(sd_ctx_t));
    // 如果内存分配失败，返回空指针
    if (sd_ctx == NULL) {
        return NULL;
    }
    // 将传入的 C 字符串转换为 C++ 字符串
    std::string model_path(model_path_c_str);
    std::string vae_path(vae_path_c_str);
    std::string taesd_path(taesd_path_c_str);
    std::string control_net_path(control_net_path_c_str);
    std::string embd_path(embed_dir_c_str);
    std::string lora_model_dir(lora_model_dir_c_str);

    // 初始化 sd_ctx_t 对象中的 StableDiffusionGGML 对象
    sd_ctx->sd = new StableDiffusionGGML(n_threads,
                                         vae_decode_only,
                                         free_params_immediately,
                                         lora_model_dir,
                                         rng_type);
    // 如果初始化失败，返回空指针
    if (sd_ctx->sd == NULL) {
        return NULL;
    }
    // 如果加载模型文件、VAE路径、控制网络路径、嵌入路径、TAESD路径、VAE平铺、权重类型、s、是否保留控制网络CPU失败，则执行以下操作
    if (!sd_ctx->sd->load_from_file(model_path,
                                    vae_path,
                                    control_net_path,
                                    embd_path,
                                    taesd_path,
                                    vae_tiling,
                                    (ggml_type)wtype,
                                    s,
                                    keep_control_net_cpu)) {
        // 删除sd_ctx->sd指向的对象
        delete sd_ctx->sd;
        // 将sd_ctx->sd指向NULL
        sd_ctx->sd = NULL;
        // 释放sd_ctx指向的内存
        free(sd_ctx);
        // 返回NULL
        return NULL;
    }
    // 返回sd_ctx
    return sd_ctx;
// 释放 sd_ctx_t 结构体指针所指向的内存，包括 sd 指针指向的内存
void free_sd_ctx(sd_ctx_t* sd_ctx) {
    // 检查 sd 指针是否为空
    if (sd_ctx->sd != NULL) {
        // 释放 sd 指针指向的内存
        delete sd_ctx->sd;
        // 将 sd 指针置为空
        sd_ctx->sd = NULL;
    }
    // 释放 sd_ctx_t 结构体指针所指向的内存
    free(sd_ctx);
}

// 将文本转换为图像
sd_image_t* txt2img(sd_ctx_t* sd_ctx,
                    const char* prompt_c_str,
                    const char* negative_prompt_c_str,
                    int clip_skip,
                    float cfg_scale,
                    int width,
                    int height,
                    enum sample_method_t sample_method,
                    int sample_steps,
                    int64_t seed,
                    int batch_count,
                    const sd_image_t* control_cond,
                    float control_strength) {
    // 记录日志信息
    LOG_DEBUG("txt2img %dx%d", width, height);
    // 检查 sd_ctx 是否为空
    if (sd_ctx == NULL) {
        return NULL;
    }
    // 将 prompt_c_str 和 negative_prompt_c_str 转换为 std::string 类型
    std::string prompt(prompt_c_str);
    std::string negative_prompt(negative_prompt_c_str);

    // 提取并移除 lora
    auto result_pair                                = extract_and_remove_lora(prompt);
    std::unordered_map<std::string, float> lora_f2m = result_pair.first;  // lora_name -> multiplier

    // 遍历 lora_f2m，记录日志信息
    for (auto& kv : lora_f2m) {
        LOG_DEBUG("lora %s:%.2f", kv.first.c_str(), kv.second);
    }

    // 更新 prompt 为提取并移除 lora 后的结果
    prompt = result_pair.second;
    LOG_DEBUG("prompt after extract and remove lora: \"%s\"", prompt.c_str());

    // 记录时间 t0
    int64_t t0 = ggml_time_ms();
    // 应用 loras 到 sd_ctx
    sd_ctx->sd->apply_loras(lora_f2m);
    // 记录时间 t1
    int64_t t1 = ggml_time_ms();
    // 记录日志信息
    LOG_INFO("apply_loras completed, taking %.2fs", (t1 - t0) * 1.0f / 1000);
    
    // 初始化 ggml_init_params 结构体
    struct ggml_init_params params;
    // 设置内存大小为 10 MB
    params.mem_size = static_cast<size_t>(10 * 1024 * 1024);  // 10 MB
    // 根据图像宽度、高度、通道数、batch 数量计算内存大小
    params.mem_size += width * height * 3 * sizeof(float);
    params.mem_size *= batch_count;
    params.mem_buffer = NULL;
    params.no_alloc   = false;
    // LOG_DEBUG("mem_size %u ", params.mem_size);

    // 初始化 ggml_context 结构体指针
    struct ggml_context* work_ctx = ggml_init(params);
}
    // 如果工作上下文为空，则初始化失败，记录错误并返回空指针
    if (!work_ctx) {
        LOG_ERROR("ggml_init() failed");
        return NULL;
    }

    // 如果种子小于0，则进行随机化处理
    if (seed < 0) {
        // 通常情况下，使用提供的命令行时，种子始终大于0。
        // 但是，为了防止在第三方以seed <0调用'stable-diffusion.cpp'作为库时可能出现问题，让我们在这里进行随机化处理。
        srand((int)time(NULL));
        seed = rand();
    }

    // 记录开始时间
    t0 = ggml_time_ms();
    // 获取学习到的条件
    auto cond_pair = sd_ctx->sd->get_learned_condition(work_ctx, prompt, clip_skip, width, height);
    ggml_tensor* c = cond_pair.first;
    ggml_tensor* c_vector = cond_pair.second;  // [adm_in_channels, ]
    struct ggml_tensor* uc = NULL;
    struct ggml_tensor* uc_vector = NULL;
    // 如果配置比例不为1.0
    if (cfg_scale != 1.0) {
        bool force_zero_embeddings = false;
        // 如果版本为VERSION_XL且negative_prompt为空
        if (sd_ctx->sd->version == VERSION_XL && negative_prompt.size() == 0) {
            force_zero_embeddings = true;
        }
        // 获取学习到的无条件
        auto uncond_pair = sd_ctx->sd->get_learned_condition(work_ctx, negative_prompt, clip_skip, width, height, force_zero_embeddings);
        uc = uncond_pair.first;
        uc_vector = uncond_pair.second;  // [adm_in_channels, ]
    }
    // 记录结束时间
    t1 = ggml_time_ms();
    LOG_INFO("get_learned_condition completed, taking %" PRId64 " ms", t1 - t0);

    // 如果立即释放参数，则释放参数缓冲区
    if (sd_ctx->sd->free_params_immediately) {
        sd_ctx->sd->cond_stage_model.free_params_buffer();
    }

    // 如果控制条件不为空，则创建图像提示
    struct ggml_tensor* image_hint = NULL;
    if (control_cond != NULL) {
        image_hint = ggml_new_tensor_4d(work_ctx, GGML_TYPE_F32, width, height, 3, 1);
        sd_image_to_tensor(control_cond->data, image_hint);
    }

    // 收集要解码的潜变量
    std::vector<struct ggml_tensor*> final_latents;
    int C = 4;
    int W = width / 8;
    int H = height / 8;
    // 记录使用的采样方法
    LOG_INFO("sampling using %s method", sampling_methods_str[sample_method]);
    // 循环生成图像，每次生成一个 batch
    for (int b = 0; b < batch_count; b++) {
        // 记录采样开始时间
        int64_t sampling_start = ggml_time_ms();
        // 计算当前种子值
        int64_t cur_seed       = seed + b;
        // 打印生成图像的信息
        LOG_INFO("generating image: %i/%i - seed %i", b + 1, batch_count, cur_seed);

        // 设置随机数生成器的种子
        sd_ctx->sd->rng->manual_seed(cur_seed);
        // 创建一个 4 维张量用于存储图像数据
        struct ggml_tensor* x_t = ggml_new_tensor_4d(work_ctx, GGML_TYPE_F32, W, H, C, 1);
        // 从随机数生成器中填充张量数据
        ggml_tensor_set_f32_randn(x_t, sd_ctx->sd->rng);

        // 获取噪声标准差的向量
        std::vector<float> sigmas = sd_ctx->sd->denoiser->schedule->get_sigmas(sample_steps);

        // 生成样本图像
        struct ggml_tensor* x_0 = sd_ctx->sd->sample(work_ctx, x_t, NULL, c, c_vector, uc, uc_vector, image_hint, cfg_scale, sample_method, sigmas, control_strength);
        // 从文件加载张量数据
        // struct ggml_tensor* x_0 = load_tensor_from_file(ctx, "samples_ddim.bin");
        // 打印张量数据
        // print_ggml_tensor(x_0);
        // 记录采样结束时间
        int64_t sampling_end = ggml_time_ms();
        // 打印采样完成信息
        LOG_INFO("sampling completed, taking %.2fs", (sampling_end - sampling_start) * 1.0f / 1000);
        // 将生成的图像张量添加到最终结果中
        final_latents.push_back(x_0);
    }

    // 如果立即释放参数，则释放参数缓冲区
    if (sd_ctx->sd->free_params_immediately) {
        sd_ctx->sd->diffusion_model.free_params_buffer();
    }
    // 记录生成图像完成时间
    int64_t t3 = ggml_time_ms();
    // 打印生成图像完成信息
    LOG_INFO("generating %" PRId64 " latent images completed, taking %.2fs", final_latents.size(), (t3 - t1) * 1.0f / 1000);

    // 打印解码潜在图像信息
    LOG_INFO("decoding %zu latents", final_latents.size());
    // 存储解码后的图像
    std::vector<struct ggml_tensor*> decoded_images;  // collect decoded images
    // 遍历所有潜在图像进行解码
    for (size_t i = 0; i < final_latents.size(); i++) {
        // 记录解码开始时间
        t1                      = ggml_time_ms();
        // 解码第一阶段的图像
        struct ggml_tensor* img = sd_ctx->sd->decode_first_stage(work_ctx, final_latents[i] /* x_0 */);
        // 打印解码后的图像
        // print_ggml_tensor(img);
        // 如果图像不为空，则添加到解码图像集合中
        if (img != NULL) {
            decoded_images.push_back(img);
        }
        // 记录解码结束时间
        int64_t t2 = ggml_time_ms();
        // 打印解码完成信息
        LOG_INFO("latent %" PRId64 " decoded, taking %.2fs", i + 1, (t2 - t1) * 1.0f / 1000);
    }

    // 记录解码第一阶段完成时间
    int64_t t4 = ggml_time_ms();
    // 打印解码第一阶段完成信息
    LOG_INFO("decode_first_stage completed, taking %.2fs", (t4 - t3) * 1.0f / 1000);
    // 如果需要立即释放参数并且不使用小型自动编码器，则释放第一阶段模型的参数缓冲区
    if (sd_ctx->sd->free_params_immediately && !sd_ctx->sd->use_tiny_autoencoder) {
        sd_ctx->sd->first_stage_model.free_params_buffer();
    }
    // 分配存储结果图像的内存空间
    sd_image_t* result_images = (sd_image_t*)calloc(batch_count, sizeof(sd_image_t));
    // 如果内存分配失败，则释放工作上下文并返回空指针
    if (result_images == NULL) {
        ggml_free(work_ctx);
        return NULL;
    }

    // 遍历解码后的图像列表，设置每个结果图像的宽度、高度、通道数和数据
    for (size_t i = 0; i < decoded_images.size(); i++) {
        result_images[i].width   = width;
        result_images[i].height  = height;
        result_images[i].channel = 3;
        result_images[i].data    = sd_tensor_to_image(decoded_images[i]);
    }
    // 释放工作上下文的内存
    ggml_free(work_ctx);
    // 记录日志，显示 txt2img 完成所花费的时间
    LOG_INFO(
        "txt2img completed in %.2fs",
        (t4 - t0) * 1.0f / 1000);

    // 返回结果图像数组
    return result_images;
// 定义一个函数，接受一些参数并返回一个指向 sd_image_t 结构体的指针
sd_image_t* img2img(sd_ctx_t* sd_ctx,
                    sd_image_t init_image,
                    const char* prompt_c_str,
                    const char* negative_prompt_c_str,
                    int clip_skip,
                    float cfg_scale,
                    int width,
                    int height,
                    sample_method_t sample_method,
                    int sample_steps,
                    float strength,
                    int64_t seed,
                    int batch_count) {
    // 如果 sd_ctx 为空指针，则返回空指针
    if (sd_ctx == NULL) {
        return NULL;
    }
    // 将 prompt_c_str 和 negative_prompt_c_str 转换为 std::string 类型
    std::string prompt(prompt_c_str);
    std::string negative_prompt(negative_prompt_c_str);

    // 记录日志信息，显示图像的宽度和高度
    LOG_INFO("img2img %dx%d", width, height);

    // 从 sd_ctx 中获取 denoiser 对象的 schedule，获取 sigmas 数组
    std::vector<float> sigmas = sd_ctx->sd->denoiser->schedule->get_sigmas(sample_steps);
    // 计算 t_enc 的值
    size_t t_enc              = static_cast<size_t>(sample_steps * strength);
    LOG_INFO("target t_enc is %zu steps", t_enc);
    // 从 sigmas 数组中获取一部分元素，赋值给 sigma_sched
    std::vector<float> sigma_sched;
    sigma_sched.assign(sigmas.begin() + sample_steps - t_enc - 1, sigmas.end());

    // 初始化 ggml_init_params 结构体，设置内存大小为 10 MB 加上图像数据所需的内存大小
    struct ggml_init_params params;
    params.mem_size = static_cast<size_t>(10 * 1024) * 1024;  // 10 MB
    params.mem_size += width * height * 3 * sizeof(float) * 2;
    params.mem_buffer = NULL;
    params.no_alloc   = false;
    // LOG_DEBUG("mem_size %u ", params.mem_size);

    // 初始化 ggml_context 结构体，如果初始化失败则返回空指针
    struct ggml_context* work_ctx = ggml_init(params);
    if (!work_ctx) {
        LOG_ERROR("ggml_init() failed");
        return NULL;
    }

    // 如果 seed 小于 0，则将其设置为当前时间的整数值
    if (seed < 0) {
        seed = (int)time(NULL);
    }

    // 手动设置随机数生成器的种子为 seed
    sd_ctx->sd->rng->manual_seed(seed);

    // 提取并移除 prompt 中的 lora，将结果存储在 lora_f2m 中
    auto result_pair                                = extract_and_remove_lora(prompt);
    std::unordered_map<std::string, float> lora_f2m = result_pair.first;  // lora_name -> multiplier
    // 遍历 lora_f2m，记录 lora 的名称和乘数
    for (auto& kv : lora_f2m) {
        LOG_DEBUG("lora %s:%.2f", kv.first.c_str(), kv.second);
    }
    // 更新 prompt 为移除 lora 后的结果
    prompt = result_pair.second;
}
    // 打印提取和移除 lora 后的提示信息
    LOG_DEBUG("prompt after extract and remove lora: \"%s\"", prompt.c_str());

    // 从文件加载 lora
    int64_t t0 = ggml_time_ms();
    // 应用 lora 到 sd
    sd_ctx->sd->apply_loras(lora_f2m);
    int64_t t1 = ggml_time_ms();
    // 打印应用 lora 完成后所花费的时间
    LOG_INFO("apply_loras completed, taking %.2fs", (t1 - t0) * 1.0f / 1000);

    // 创建一个 4D 张量用于初始化图像
    ggml_tensor* init_img = ggml_new_tensor_4d(work_ctx, GGML_TYPE_F32, width, height, 3, 1);
    // 将图像数据转换为张量
    sd_image_to_tensor(init_image.data, init_img);
    t0 = ggml_time_ms();
    ggml_tensor* init_latent = NULL;
    // 如果不使用 tiny 自动编码器
    if (!sd_ctx->sd->use_tiny_autoencoder) {
        // 对初始图像进行第一阶段编码
        ggml_tensor* moments = sd_ctx->sd->encode_first_stage(work_ctx, init_img);
        // 获取第一阶段编码结果
        init_latent = sd_ctx->sd->get_first_stage_encoding(work_ctx, moments);
    } else {
        // 使用 tiny 自动编码器对初始图像进行第一阶段编码
        init_latent = sd_ctx->sd->encode_first_stage(work_ctx, init_img);
    }
    t1 = ggml_time_ms();
    // 打印第一阶段编码完成后所花费的时间
    LOG_INFO("encode_first_stage completed, taking %.2fs", (t1 - t0) * 1.0f / 1000);

    // 获取学习到的条件
    auto cond_pair = sd_ctx->sd->get_learned_condition(work_ctx, prompt, clip_skip, width, height);
    ggml_tensor* c = cond_pair.first;
    ggml_tensor* c_vector = cond_pair.second;  // [adm_in_channels, ]
    struct ggml_tensor* uc = NULL;
    struct ggml_tensor* uc_vector = NULL;
    // 如果 cfg_scale 不等于 1.0
    if (cfg_scale != 1.0) {
        bool force_zero_embeddings = false;
        // 如果版本为 VERSION_XL 并且 negative_prompt 的大小为 0
        if (sd_ctx->sd->version == VERSION_XL && negative_prompt.size() == 0) {
            force_zero_embeddings = true;
        }
        // 获取学习到的无条件条件
        auto uncond_pair = sd_ctx->sd->get_learned_condition(work_ctx, negative_prompt, clip_skip, width, height, force_zero_embeddings);
        uc = uncond_pair.first;
        uc_vector = uncond_pair.second;  // [adm_in_channels, ]
    }
    int64_t t2 = ggml_time_ms();
    // 打印获取学习到的条件完成后所花费的时间
    LOG_INFO("get_learned_condition completed, taking %" PRId64 " ms", t2 - t1);
    // 如果需要立即释放参数
    if (sd_ctx->sd->free_params_immediately) {
        // 释放参数缓冲区
        sd_ctx->sd->cond_stage_model.free_params_buffer();
    }
    }

    // 设置随机数生成器的种子
    sd_ctx->sd->rng->manual_seed(seed);
    // 复制初始潜变量张量
    struct ggml_tensor* noise = ggml_dup_tensor(work_ctx, init_latent);
    // 为张量设置服从标准正态分布的随机数
    ggml_tensor_set_f32_randn(noise, sd_ctx->sd->rng);

    // 输出日志，指示使用的采样方法
    LOG_INFO("sampling using %s method", sampling_methods_str[sample_method]);
    // 使用指定方法进行采样
    struct ggml_tensor* x_0 = sd_ctx->sd->sample(work_ctx, init_latent, noise, c, c_vector, uc,
                                                 uc_vector, NULL, cfg_scale, sample_method, sigma_sched, 1.0f);
    // 从文件加载张量
    // struct ggml_tensor *x_0 = load_tensor_from_file(ctx, "samples_ddim.bin");
    // 打印张量信息
    // print_ggml_tensor(x_0);
    int64_t t3 = ggml_time_ms();
    // 输出日志，指示采样完成所需时间
    LOG_INFO("sampling completed, taking %.2fs", (t3 - t2) * 1.0f / 1000);
    // 如果立即释放参数
    if (sd_ctx->sd->free_params_immediately) {
        sd_ctx->sd->diffusion_model.free_params_buffer();
    }

    // 解码第一阶段
    struct ggml_tensor* img = sd_ctx->sd->decode_first_stage(work_ctx, x_0);
    // 如果立即释放参数且不使用小型自动编码器
    if (sd_ctx->sd->free_params_immediately && !sd_ctx->sd->use_tiny_autoencoder) {
        sd_ctx->sd->first_stage_model.free_params_buffer();
    }
    // 如果图像为空
    if (img == NULL) {
        ggml_free(work_ctx);
        return NULL;
    }

    // 分配结果图像内存
    sd_image_t* result_images = (sd_image_t*)calloc(1, sizeof(sd_image_t));
    // 如果分配失败
    if (result_images == NULL) {
        ggml_free(work_ctx);
        return NULL;
    }

    // 遍历结果图像数组
    for (size_t i = 0; i < 1; i++) {
        result_images[i].width   = width;
        result_images[i].height  = height;
        result_images[i].channel = 3;
        result_images[i].data    = sd_tensor_to_image(img);
    }
    ggml_free(work_ctx);

    int64_t t4 = ggml_time_ms();
    // 输出日志，指示解码第一阶段完成所需时间
    LOG_INFO("decode_first_stage completed, taking %.2fs", (t4 - t3) * 1.0f / 1000);

    // 输出日志，指示整个过程完成所需时间
    LOG_INFO("img2img completed in %.2fs", (t4 - t0) * 1.0f / 1000);

    return result_images;
# 闭合之前的代码块
```