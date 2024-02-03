# `stable-diffusion.cpp\examples\cli\main.cpp`

```
#include <stdio.h> // 包含标准输入输出头文件
#include <string.h> // 包含字符串处理头文件
#include <time.h> // 包含时间处理头文件
#include <iostream> // 包含输入输出流头文件
#include <random> // 包含随机数生成头文件
#include <string> // 包含字符串处理头文件
#include <vector> // 包含向量容器头文件

#include "preprocessing.hpp" // 包含预处理头文件
#include "stable-diffusion.h" // 包含稳定扩散头文件

#define STB_IMAGE_IMPLEMENTATION // 定义图像处理宏
#include "stb_image.h" // 包含图像读取头文件

#define STB_IMAGE_WRITE_IMPLEMENTATION // 定义图像写入宏
#define STB_IMAGE_WRITE_STATIC
#include "stb_image_write.h" // 包含图像写入头文件

const char* rng_type_to_str[] = { // 定义随机数生成器类型字符串数组
    "std_default",
    "cuda",
};

// Names of the sampler method, same order as enum sample_method in stable-diffusion.h
const char* sample_method_str[] = { // 定义采样方法字符串数组，与 stable-diffusion.h 中的枚举 sample_method 顺序相同
    "euler_a",
    "euler",
    "heun",
    "dpm2",
    "dpm++2s_a",
    "dpm++2m",
    "dpm++2mv2",
    "lcm",
};

// Names of the sigma schedule overrides, same order as sample_schedule in stable-diffusion.h
const char* schedule_str[] = { // 定义 sigma 调度覆盖的字符串数组，与 stable-diffusion.h 中的 sample_schedule 顺序相同
    "default",
    "discrete",
    "karras",
};

const char* modes_str[] = { // 定义模式字符串数组
    "txt2img",
    "img2img",
    "convert",
};

enum SDMode { // 定义 SDMode 枚举类型
    TXT2IMG,
    IMG2IMG,
    CONVERT,
    MODE_COUNT
};

struct SDParams { // 定义 SDParams 结构体
    int n_threads = -1; // 初始化线程数为 -1
    SDMode mode   = TXT2IMG; // 初始化模式为 TXT2IMG

    std::string model_path; // 模型路径字符串
    std::string vae_path; // VAE 路径字符串
    std::string taesd_path; // TAESD 路径字符串
    std::string esrgan_path; // ESRGAN 路径字符串
    std::string controlnet_path; // 控制网络路径字符串
    std::string embeddings_path; // 嵌入路径字符串
    sd_type_t wtype = SD_TYPE_COUNT; // 初始化 wtype 为 SD_TYPE_COUNT
    std::string lora_model_dir; // Lora 模型目录字符串
    std::string output_path = "output.png"; // 输出路径字符串，默认为 output.png
    std::string input_path; // 输入路径字符串
    std::string control_image_path; // 控制图像路径字符串

    std::string prompt; // 提示字符串
    std::string negative_prompt; // 负面提示字符串
    float cfg_scale = 7.0f; // 配置比例为 7.0
    int clip_skip   = -1;  // <= 0 represents unspecified // 剪辑跳过值，小于等于 0 表示未指定
    int width       = 512; // 宽度为 512
    int height      = 512; // 高度为 512
    int batch_count = 1; // 批次计数为 1

    sample_method_t sample_method = EULER_A; // 采样方法为 EULER_A
    schedule_t schedule           = DEFAULT; // 调度为 DEFAULT
    int sample_steps              = 20; // 采样步数为 20
    float strength                = 0.75f; // 强度为 0.75
    float control_strength        = 0.9f; // 控制强度为 0.9
    rng_type_t rng_type           = CUDA_RNG; // 随机数生成器类型为 CUDA_RNG
    int64_t seed                  = 42; // 种子为 42
    bool verbose                  = false; // 是否详细输出为 false
    # 定义一个布尔变量 vae_tiling，并初始化为 false，用于标识是否使用 VAE 平铺
    bool vae_tiling               = false;
    # 定义一个布尔变量 control_net_cpu，并初始化为 false，用于标识是否控制网络在 CPU 上运行
    bool control_net_cpu          = false;
    # 定义一个布尔变量 canny_preprocess，并初始化为 false，用于标识是否进行 Canny 预处理
    bool canny_preprocess         = false;
// 打印参数信息
void print_params(SDParams params) {
    // 打印线程数
    printf("Option: \n");
    printf("    n_threads:         %d\n", params.n_threads);
    // 打印模式
    printf("    mode:              %s\n", modes_str[params.mode]);
    // 打印模型路径
    printf("    model_path:        %s\n", params.model_path.c_str());
    // 打印权重类型
    printf("    wtype:             %s\n", params.wtype < SD_TYPE_COUNT ? sd_type_name(params.wtype) : "unspecified");
    // 打印 VAE 路径
    printf("    vae_path:          %s\n", params.vae_path.c_str());
    // 打印 TAESD 路径
    printf("    taesd_path:        %s\n", params.taesd_path.c_str());
    // 打印 ESRGAN 路径
    printf("    esrgan_path:       %s\n", params.esrgan_path.c_str());
    // 打印控制网络路径
    printf("    controlnet_path:   %s\n", params.controlnet_path.c_str());
    // 打印嵌入路径
    printf("    embeddings_path:   %s\n", params.embeddings_path.c_str());
    // 打印输出路径
    printf("    output_path:       %s\n", params.output_path.c_str());
    // 打印初始图像路径
    printf("    init_img:          %s\n", params.input_path.c_str());
    // 打印控制图像路径
    printf("    control_image:     %s\n", params.control_image_path.c_str());
    // 打印控制网络是否在 CPU 上运行
    printf("    controlnet cpu:    %s\n", params.control_net_cpu ? "true" : "false");
    // 打印控制强度
    printf("    strength(control): %.2f\n", params.control_strength);
    // 打印提示
    printf("    prompt:            %s\n", params.prompt.c_str());
    // 打印负面提示
    printf("    negative_prompt:   %s\n", params.negative_prompt.c_str());
    // 打印配置比例
    printf("    cfg_scale:         %.2f\n", params.cfg_scale);
    // 打印跳过帧数
    printf("    clip_skip:         %d\n", params.clip_skip);
    // 打印宽度
    printf("    width:             %d\n", params.width);
    // 打印高度
    printf("    height:            %d\n", params.height);
    // 打印采样方法
    printf("    sample_method:     %s\n", sample_method_str[params.sample_method]);
    // 打印调度
    printf("    schedule:          %s\n", schedule_str[params.schedule]);
    // 打印采样步数
    printf("    sample_steps:      %d\n", params.sample_steps);
    // 打印图像强度
    printf("    strength(img2img): %.2f\n", params.strength);
    // 打印随机数生成器类型
    printf("    rng:               %s\n", rng_type_to_str[params.rng_type]);
    // 打印种子
    printf("    seed:              %ld\n", params.seed);
    // 打印批次数
    printf("    batch_count:       %d\n", params.batch_count);
}
    # 打印参数 vae_tiling 的取值，如果为 true 则打印字符串 "true"，否则打印字符串 "false"
    printf("    vae_tiling:        %s\n", params.vae_tiling ? "true" : "false");
// 打印程序的使用方法，包括参数说明
void print_usage(int argc, const char* argv[]) {
    // 打印程序的使用方法，显示可执行文件名
    printf("usage: %s [arguments]\n", argv[0]);
    // 打印空行
    printf("\n");
    // 打印参数说明
    printf("arguments:\n");
    // 打印各个参数的说明
    printf("  -h, --help                         show this help message and exit\n");
    printf("  -M, --mode [MODEL]                 run mode (txt2img or img2img or convert, default: txt2img)\n");
    printf("  -t, --threads N                    number of threads to use during computation (default: -1).\n");
    printf("                                     If threads <= 0, then threads will be set to the number of CPU physical cores\n");
    printf("  -m, --model [MODEL]                path to model\n");
    printf("  --vae [VAE]                        path to vae\n");
    printf("  --taesd [TAESD_PATH]               path to taesd. Using Tiny AutoEncoder for fast decoding (low quality)\n");
    printf("  --control-net [CONTROL_PATH]       path to control net model\n");
    printf("  --embd-dir [EMBEDDING_PATH]        path to embeddings.\n");
    printf("  --upscale-model [ESRGAN_PATH]      path to esrgan model. Upscale images after generate, just RealESRGAN_x4plus_anime_6B supported by now.\n");
    printf("  --type [TYPE]                      weight type (f32, f16, q4_0, q4_1, q5_0, q5_1, q8_0)\n");
    printf("                                     If not specified, the default is the type of the weight file.\n");
    printf("  --lora-model-dir [DIR]             lora model directory\n");
    printf("  -i, --init-img [IMAGE]             path to the input image, required by img2img\n");
    printf("  --control-image [IMAGE]            path to image condition, control net\n");
    printf("  -o, --output OUTPUT                path to write result image to (default: ./output.png)\n");
    printf("  -p, --prompt [PROMPT]              the prompt to render\n");
    printf("  -n, --negative-prompt PROMPT       the negative prompt (default: \"\")\n");
}
    # 打印指定参数的描述信息
    printf("  --cfg-scale SCALE                  unconditional guidance scale: (default: 7.0)\n");
    printf("  --strength STRENGTH                strength for noising/unnoising (default: 0.75)\n");
    printf("  --control-strength STRENGTH        strength to apply Control Net (default: 0.9)\n");
    printf("                                     1.0 corresponds to full destruction of information in init image\n");
    printf("  -H, --height H                     image height, in pixel space (default: 512)\n");
    printf("  -W, --width W                      image width, in pixel space (default: 512)\n");
    printf("  --sampling-method {euler, euler_a, heun, dpm2, dpm++2s_a, dpm++2m, dpm++2mv2, lcm}\n");
    printf("                                     sampling method (default: \"euler_a\")\n");
    printf("  --steps  STEPS                     number of sample steps (default: 20)\n");
    printf("  --rng {std_default, cuda}          RNG (default: cuda)\n");
    printf("  -s SEED, --seed SEED               RNG seed (default: 42, use random seed for < 0)\n");
    printf("  -b, --batch-count COUNT            number of images to generate.\n");
    printf("  --schedule {discrete, karras}      Denoiser sigma schedule (default: discrete)\n");
    printf("  --clip-skip N                      ignore last layers of CLIP network; 1 ignores none, 2 ignores one layer (default: -1)\n");
    printf("                                     <= 0 represents unspecified, will be 1 for SD1.x, 2 for SD2.x\n");
    printf("  --vae-tiling                       process vae in tiles to reduce memory usage\n");
    printf("  --control-net-cpu                  keep controlnet in cpu (for low vram)\n");
    printf("  --canny                            apply canny preprocessor (edge detection)\n");
    printf("  -v, --verbose                      print extra info\n");
}

void parse_args(int argc, const char** argv, SDParams& params) {
    bool invalid_arg = false;
    std::string arg;
    }

    // 如果存在无效参数，则输出错误信息并打印用法，然后退出程序
    if (invalid_arg) {
        fprintf(stderr, "error: invalid parameter for argument: %s\n", arg.c_str());
        print_usage(argc, argv);
        exit(1);
    }

    // 如果线程数小于等于0，则设置为物理核心数
    if (params.n_threads <= 0) {
        params.n_threads = get_num_physical_cores();
    }

    // 如果模式不是转换模式且提示长度为0，则输出错误信息并打印用法，然后退出程序
    if (params.mode != CONVERT && params.prompt.length() == 0) {
        fprintf(stderr, "error: the following arguments are required: prompt\n");
        print_usage(argc, argv);
        exit(1);
    }

    // 如果模型路径长度为0，则输出错误信息并打印用法，然后退出程序
    if (params.model_path.length() == 0) {
        fprintf(stderr, "error: the following arguments are required: model_path\n");
        print_usage(argc, argv);
        exit(1);
    }

    // 如果模式为 IMG2IMG 且输入路径长度为0，则输出错误信息并打印用法，然后退出程序
    if (params.mode == IMG2IMG && params.input_path.length() == 0) {
        fprintf(stderr, "error: when using the img2img mode, the following arguments are required: init-img\n");
        print_usage(argc, argv);
        exit(1);
    }

    // 如果输出路径长度为0，则输出错误信息并打印用法，然后退出程序
    if (params.output_path.length() == 0) {
        fprintf(stderr, "error: the following arguments are required: output_path\n");
        print_usage(argc, argv);
        exit(1);
    }

    // 如果宽度小于等于0或不是64的倍数，则输出错误信息并退出程序
    if (params.width <= 0 || params.width % 64 != 0) {
        fprintf(stderr, "error: the width must be a multiple of 64\n");
        exit(1);
    }

    // 如果高度小于等于0或不是64的倍数，则输出错误信息并退出程序
    if (params.height <= 0 || params.height % 64 != 0) {
        fprintf(stderr, "error: the height must be a multiple of 64\n");
        exit(1);
    }

    // 如果采样步数小于等于0，则输出错误信息并退出程序
    if (params.sample_steps <= 0) {
        fprintf(stderr, "error: the sample_steps must be greater than 0\n");
        exit(1);
    }

    // 如果强度小于0或大于1，则输出错误信息并退出程序
    if (params.strength < 0.f || params.strength > 1.f) {
        fprintf(stderr, "error: can only work with strength in [0.0, 1.0]\n");
        exit(1);
    }

    // 如果种子小于0，则设置随机种子为当前时间的整数秒数，然后将种子设置为随机数
    if (params.seed < 0) {
        srand((int)time(NULL));
        params.seed = rand();
    }
    # 如果参数中的模式为CONVERT
    if (params.mode == CONVERT) {
        # 如果输出路径为"output.png"
        if (params.output_path == "output.png") {
            # 将输出路径修改为"output.gguf"
            params.output_path = "output.gguf";
        }
    }
}

// 获取图像参数的字符串表示
std::string get_image_params(SDParams params, int64_t seed) {
    // 初始化参数字符串为提示信息
    std::string parameter_string = params.prompt + "\n";
    // 如果存在负面提示，则添加到参数字符串中
    if (params.negative_prompt.size() != 0) {
        parameter_string += "Negative prompt: " + params.negative_prompt + "\n";
    }
    // 添加步数、CFG 比例、种子、尺寸、模型、随机数生成器和采样器信息到参数字符串中
    parameter_string += "Steps: " + std::to_string(params.sample_steps) + ", ";
    parameter_string += "CFG scale: " + std::to_string(params.cfg_scale) + ", ";
    parameter_string += "Seed: " + std::to_string(seed) + ", ";
    parameter_string += "Size: " + std::to_string(params.width) + "x" + std::to_string(params.height) + ", ";
    parameter_string += "Model: " + sd_basename(params.model_path) + ", ";
    parameter_string += "RNG: " + std::string(rng_type_to_str[params.rng_type]) + ", ";
    parameter_string += "Sampler: " + std::string(sample_method_str[params.sample_method]);
    // 如果调度方式为 KARRAS，则添加 karras 到参数字符串中
    if (params.schedule == KARRAS) {
        parameter_string += " karras";
    }
    parameter_string += ", ";
    parameter_string += "Version: stable-diffusion.cpp";
    return parameter_string;
}

// 日志回调函数
void sd_log_cb(enum sd_log_level_t level, const char* log, void* data) {
    // 将数据转换为 SDParams 指针
    SDParams* params = (SDParams*)data;
    // 如果不是详细模式且日志级别低于 DEBUG，则返回
    if (!params->verbose && level <= SD_LOG_DEBUG) {
        return;
    }
    // 如果日志级别低于 INFO，则输出到标准输出
    if (level <= SD_LOG_INFO) {
        fputs(log, stdout);
        fflush(stdout);
    } else {
        // 否则输出到标准错误
        fputs(log, stderr);
        fflush(stderr);
    }
}

// 主函数
int main(int argc, const char* argv[]) {
    // 初始化参数对象
    SDParams params;
    // 解析命令行参数
    parse_args(argc, argv, params);

    // 设置日志回调函数
    sd_set_log_callback(sd_log_cb, (void*)&params);

    // 如果是详细模式，则打印参数信息和系统信息
    if (params.verbose) {
        print_params(params);
        printf("%s", sd_get_system_info());
    }
    // 如果参数中的模式为CONVERT，则执行以下代码块
    if (params.mode == CONVERT) {
        // 调用convert函数，将模型路径、VAE路径、输出路径和权重类型传入，并将返回值存储在success变量中
        bool success = convert(params.model_path.c_str(), params.vae_path.c_str(), params.output_path.c_str(), params.wtype);
        // 如果转换失败，则输出错误信息并返回1
        if (!success) {
            fprintf(stderr,
                    "convert '%s'/'%s' to '%s' failed\n",
                    params.model_path.c_str(),
                    params.vae_path.c_str(),
                    params.output_path.c_str());
            return 1;
        } else {
            // 如果转换成功，则输出成功信息并返回0
            printf("convert '%s'/'%s' to '%s' success\n",
                   params.model_path.c_str(),
                   params.vae_path.c_str(),
                   params.output_path.c_str());
            return 0;
        }
    }

    // 初始化变量
    bool vae_decode_only        = true;
    uint8_t* input_image_buffer = NULL;
    // 如果参数中的模式为IMG2IMG，则执行以下代码块
    if (params.mode == IMG2IMG) {
        // 将vae_decode_only设置为false
        vae_decode_only = false;

        int c              = 0;
        // 从输入路径加载图像数据到input_image_buffer中，并获取图像的宽度和高度
        input_image_buffer = stbi_load(params.input_path.c_str(), &params.width, &params.height, &c, 3);
        // 如果加载图像失败，则输出错误信息并返回1
        if (input_image_buffer == NULL) {
            fprintf(stderr, "load image from '%s' failed\n", params.input_path.c_str());
            return 1;
        }
        // 如果图像通道数不为3，则输出错误信息并释放内存后返回1
        if (c != 3) {
            fprintf(stderr, "input image must be a 3 channels RGB image, but got %d channels\n", c);
            free(input_image_buffer);
            return 1;
        }
        // 如果图像宽度不合法，则输出错误信息并释放内存后返回1
        if (params.width <= 0 || params.width % 64 != 0) {
            fprintf(stderr, "error: the width of image must be a multiple of 64\n");
            free(input_image_buffer);
            return 1;
        }
        // 如果图像高度不合法，则输出错误信息并释放内存后返回1
        if (params.height <= 0 || params.height % 64 != 0) {
            fprintf(stderr, "error: the height of image must be a multiple of 64\n");
            free(input_image_buffer);
            return 1;
        }
    }
    # 创建一个 sd_ctx_t 结构体指针，并使用参数中的路径初始化该结构体
    sd_ctx_t* sd_ctx = new_sd_ctx(params.model_path.c_str(),
                                  params.vae_path.c_str(),
                                  params.taesd_path.c_str(),
                                  params.controlnet_path.c_str(),
                                  params.lora_model_dir.c_str(),
                                  params.embeddings_path.c_str(),
                                  vae_decode_only,
                                  params.vae_tiling,
                                  true,
                                  params.n_threads,
                                  params.wtype,
                                  params.rng_type,
                                  params.schedule,
                                  params.control_net_cpu);
    
    # 检查 sd_ctx 是否为 NULL，如果为 NULL 则输出错误信息并返回 1
    if (sd_ctx == NULL) {
        printf("new_sd_ctx_t failed\n");
        return 1;
    }
    
    # 创建一个 sd_image_t 结构体指针 results
    sd_image_t* results;
    // 如果参数中的模式为TXT2IMG
    if (params.mode == TXT2IMG) {
        // 控制图像指针初始化为空
        sd_image_t* control_image = NULL;
        // 如果控制网络路径和控制图像路径都不为空
        if (params.controlnet_path.size() > 0 && params.control_image_path.size() > 0) {
            // 初始化变量c为0
            int c = 0;
            // 从控制图像路径加载图像数据到输入图像缓冲区
            input_image_buffer = stbi_load(params.control_image_path.c_str(), &params.width, &params.height, &c, 3);
            // 如果加载失败
            if (input_image_buffer == NULL) {
                // 输出错误信息
                fprintf(stderr, "load image from '%s' failed\n", params.control_image_path.c_str());
                // 返回错误码1
                return 1;
            }
            // 创建控制图像对象
            control_image = new sd_image_t{(uint32_t)params.width,
                                           (uint32_t)params.height,
                                           3,
                                           input_image_buffer};
            // 如果需要进行canny预处理
            if (params.canny_preprocess) {
                // 输出日志信息
                LOG_INFO("Applying canny preprocessor");
                // 对控制图像数据进行canny预处理
                control_image->data = preprocess_canny(control_image->data, control_image->width, control_image->height);
            }
        }
        // 调用txt2img函数生成图像
        results = txt2img(sd_ctx,
                          params.prompt.c_str(),
                          params.negative_prompt.c_str(),
                          params.clip_skip,
                          params.cfg_scale,
                          params.width,
                          params.height,
                          params.sample_method,
                          params.sample_steps,
                          params.seed,
                          params.batch_count,
                          control_image,
                          params.control_strength);
    }
    } else {
        // 如果不是第一次迭代，则创建输入图像结构体
        sd_image_t input_image = {(uint32_t)params.width,
                                  (uint32_t)params.height,
                                  3,
                                  input_image_buffer};

        // 调用img2img函数进行图像处理
        results = img2img(sd_ctx,
                          input_image,
                          params.prompt.c_str(),
                          params.negative_prompt.c_str(),
                          params.clip_skip,
                          params.cfg_scale,
                          params.width,
                          params.height,
                          params.sample_method,
                          params.sample_steps,
                          params.strength,
                          params.seed,
                          params.batch_count);
    }

    // 如果处理结果为空，则打印错误信息并释放资源
    if (results == NULL) {
        printf("generate failed\n");
        free_sd_ctx(sd_ctx);
        return 1;
    }

    int upscale_factor = 4;  // 用于RealESRGAN_x4plus_anime_6B.pth的放大因子，未使用
    // 如果ESRGAN路径不为空，则进行图像放大处理
    if (params.esrgan_path.size() > 0) {
        // 创建图像放大器上下文
        upscaler_ctx_t* upscaler_ctx = new_upscaler_ctx(params.esrgan_path.c_str(),
                                                        params.n_threads,
                                                        params.wtype);

        // 如果创建失败，则打印错误信息
        if (upscaler_ctx == NULL) {
            printf("new_upscaler_ctx failed\n");
        } else {
            // 遍历处理结果，进行图像放大
            for (int i = 0; i < params.batch_count; i++) {
                // 如果处理结果数据为空，则跳过
                if (results[i].data == NULL) {
                    continue;
                }
                // 调用upscale函数进行图像放大
                sd_image_t upscaled_image = upscale(upscaler_ctx, results[i], upscale_factor);
                // 如果放大失败，则打印错误信息并继续下一个处理结果
                if (upscaled_image.data == NULL) {
                    printf("upscale failed\n");
                    continue;
                }
                // 释放原始处理结果数据，并将放大后的图像数据替换原始数据
                free(results[i].data);
                results[i] = upscaled_image;
            }
        }
    }

    // 获取输出路径中最后一个"."的位置
    size_t last            = params.output_path.find_last_of(".");
    // 根据输出路径获取虚拟名称，如果路径中包含文件扩展名，则去掉扩展名
    std::string dummy_name = last != std::string::npos ? params.output_path.substr(0, last) : params.output_path;
    // 遍历结果数组中的每个元素
    for (int i = 0; i < params.batch_count; i++) {
        // 如果结果数据为空，则跳过当前循环
        if (results[i].data == NULL) {
            continue;
        }
        // 根据当前索引生成最终图片路径
        std::string final_image_path = i > 0 ? dummy_name + "_" + std::to_string(i + 1) + ".png" : dummy_name + ".png";
        // 将结果数据写入 PNG 文件
        stbi_write_png(final_image_path.c_str(), results[i].width, results[i].height, results[i].channel,
                       results[i].data, 0, get_image_params(params, params.seed + i).c_str());
        // 打印保存结果图片的路径
        printf("save result image to '%s'\n", final_image_path.c_str());
        // 释放结果数据内存
        free(results[i].data);
        results[i].data = NULL;
    }
    // 释放结果数组内存
    free(results);
    // 释放 SD 上下文内存
    free_sd_ctx(sd_ctx);

    // 返回成功状态码
    return 0;
# 闭合之前的代码块
```