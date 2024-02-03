# `whisper.cpp\examples\addon.node\addon.cpp`

```cpp
// 包含 NAPI 库的头文件
#include "napi.h"
// 包含自定义的 common.h 头文件
#include "common.h"

// 包含 whisper.h 头文件
#include "whisper.h"

// 包含标准库的头文件
#include <string>
#include <thread>
#include <vector>
#include <cmath>
#include <cstdint>

// 定义 whisper_params 结构体，包含多个参数
struct whisper_params {
    // 线程数，默认为最小值为 4 和硬件并发数的较小值
    int32_t n_threads    = std::min(4, (int32_t) std::thread::hardware_concurrency());
    // 处理器数，默认为 1
    int32_t n_processors = 1;
    // 时间偏移（毫秒），默认为 0
    int32_t offset_t_ms  = 0;
    // 数量偏移，默认为 0
    int32_t offset_n     = 0;
    // 持续时间（毫秒），默认为 0
    int32_t duration_ms  = 0;
    // 最大上下文数，默认为 -1
    int32_t max_context  = -1;
    // 最大长度，默认为 0
    int32_t max_len      = 0;
    // 最佳结果次数，默认为 5
    int32_t best_of      = 5;
    // 波束大小，默认为 -1
    int32_t beam_size    = -1;

    // 单词阈值，默认为 0.01
    float word_thold    = 0.01f;
    // 熵阈值，默认为 2.4
    float entropy_thold = 2.4f;
    // 对数概率阈值，默认为 -1.0
    float logprob_thold = -1.0f;

    // 是否加速，默认为 false
    bool speed_up       = false;
    // 是否翻译，默认为 false
    bool translate      = false;
    // 是否分离，默认为 false
    bool diarize        = false;
    // 是否输出文本，默认为 false
    bool output_txt     = false;
    // 是否输出 VTT，默认为 false
    bool output_vtt     = false;
    // 是否输出 SRT，默认为 false
    bool output_srt     = false;
    // 是否输出 WTS，默认为 false
    bool output_wts     = false;
    // 是否输出 CSV，默认为 false
    bool output_csv     = false;
    // 是否打印特殊信息，默认为 false
    bool print_special  = false;
    // 是否打印颜色，默认为 false
    bool print_colors   = false;
    // 是否打印进度，默认为 false
    bool print_progress = false;
    // 是否不包含时间戳，默认为 false
    bool no_timestamps  = false;
    // 是否使用 GPU，默认为 true
    bool use_gpu        = true;

    // 语言，默认为 "en"
    std::string language = "en";
    // 提示信息
    std::string prompt;
    // 模型，默认为 "../../ggml-large.bin"
    std::string model    = "../../ggml-large.bin";

    // 输入文件名列表
    std::vector<std::string> fname_inp = {};
    // 输出文件名列表
    std::vector<std::string> fname_out = {};
};

// 定义 whisper_print_user_data 结构体，包含参数和 PCM 数据
struct whisper_print_user_data {
    const whisper_params * params;
    const std::vector<std::vector<float>> * pcmf32s;
};

// 将时间戳转换为字符串格式，可选择是否使用逗号
//  500 -> 00:05.000
// 6000 -> 01:00.000
std::string to_timestamp(int64_t t, bool comma = false) {
    // 将时间戳转换为毫秒
    int64_t msec = t * 10;
    // 计算小时
    int64_t hr = msec / (1000 * 60 * 60);
    msec = msec - hr * (1000 * 60 * 60);
    // 计算分钟
    int64_t min = msec / (1000 * 60);
    msec = msec - min * (1000 * 60);
    // 计算秒
    int64_t sec = msec / 1000;
    msec = msec - sec * 1000;

    // 格式化输出时间戳字符串
    char buf[32];
    snprintf(buf, sizeof(buf), "%02d:%02d:%02d%s%03d", (int) hr, (int) min, (int) sec, comma ? "," : ".", (int) msec);

    return std::string(buf);
}

// 将时间戳转换为采样数
int timestamp_to_sample(int64_t t, int n_samples) {
    # 返回 n_samples - 1 和 (t*WHISPER_SAMPLE_RATE)/100 中的较小值，且不小于 0
    return std::max(0, std::min((int) n_samples - 1, (int) ((t*WHISPER_SAMPLE_RATE)/100)));
// 结束函数 whisper_print_segment_callback
}

void whisper_print_segment_callback(struct whisper_context * ctx, struct whisper_state * state, int n_new, void * user_data) {
    // 从用户数据中获取参数和 PCM 数据
    const auto & params  = *((whisper_print_user_data *) user_data)->params;
    const auto & pcmf32s = *((whisper_print_user_data *) user_data)->pcmf32s;

    // 获取完整段数
    const int n_segments = whisper_full_n_segments(ctx);

    // 初始化说话者字符串
    std::string speaker = "";

    // 初始化时间戳 t0 和 t1
    int64_t t0;
    int64_t t1;

    // 打印最后 n_new 个段
    const int s0 = n_segments - n_new;

    // 如果 s0 为 0，则打印换行符
    if (s0 == 0) {
        printf("\n");
    }
    // 遍历从 s0 开始到 n_segments 结束的所有段落
    for (int i = s0; i < n_segments; i++) {
        // 如果不禁用时间戳或者进行说话人分离
        if (!params.no_timestamps || params.diarize) {
            // 获取第 i 段的起始时间和结束时间
            t0 = whisper_full_get_segment_t0(ctx, i);
            t1 = whisper_full_get_segment_t1(ctx, i);
        }

        // 如果不禁用时间戳
        if (!params.no_timestamps) {
            // 打印起始时间和结束时间的时间戳
            printf("[%s --> %s]  ", to_timestamp(t0).c_str(), to_timestamp(t1).c_str());
        }

        // 如果进行说话人分离并且 pcmf32s 的大小为 2
        if (params.diarize && pcmf32s.size() == 2) {
            // 获取音频样本数
            const int64_t n_samples = pcmf32s[0].size();

            // 将时间戳转换为样本索引
            const int64_t is0 = timestamp_to_sample(t0, n_samples);
            const int64_t is1 = timestamp_to_sample(t1, n_samples);

            double energy0 = 0.0f;
            double energy1 = 0.0f;

            // 计算两个说话人在该段落内的能量
            for (int64_t j = is0; j < is1; j++) {
                energy0 += fabs(pcmf32s[0][j]);
                energy1 += fabs(pcmf32s[1][j]);
            }

            // 根据能量比较确定说话人
            if (energy0 > 1.1*energy1) {
                speaker = "(speaker 0)";
            } else if (energy1 > 1.1*energy0) {
                speaker = "(speaker 1)";
            } else {
                speaker = "(speaker ?)";
            }

            // 打印调试信息
            //printf("is0 = %lld, is1 = %lld, energy0 = %f, energy1 = %f, %s\n", is0, is1, energy0, energy1, speaker.c_str());
        }

        // 打印说话人和文本内容
        const char * text = whisper_full_get_segment_text(ctx, i);
        printf("%s%s", speaker.c_str(), text);

        // 如果有时间戳或者说话人信息，则每个段落换行打印
        if (!params.no_timestamps || params.diarize) {
            printf("\n");
        }

        // 刷新标准输出
        fflush(stdout);
    }
}

// 运行函数，接受参数和结果向量，返回运行状态
int run(whisper_params &params, std::vector<std::vector<std::string>> &result) {
    // 如果输入文件名为空，输出错误信息并返回状态码2
    if (params.fname_inp.empty()) {
        fprintf(stderr, "error: no input files specified\n");
        return 2;
    }

    // 如果语言不是"auto"且语言ID为-1，输出错误信息并退出程序
    if (params.language != "auto" && whisper_lang_id(params.language.c_str()) == -1) {
        fprintf(stderr, "error: unknown language '%s'\n", params.language.c_str());
        exit(0);
    }

    // 初始化 whisper

    // 设置 whisper 上下文参数
    struct whisper_context_params cparams;
    cparams.use_gpu = params.use_gpu;
    // 从文件初始化 whisper 上下文
    struct whisper_context * ctx = whisper_init_from_file_with_params(params.model.c_str(), cparams);

    // 如果上下文为空，输出错误信息并返回状态码3
    if (ctx == nullptr) {
        fprintf(stderr, "error: failed to initialize whisper context\n");
        return 3;
    }

    }

    // 获取 whisper 上下文的段数
    const int n_segments = whisper_full_n_segments(ctx);
    // 调整结果向量大小以容纳段数
    result.resize(n_segments);
    // 遍历每个段
    for (int i = 0; i < n_segments; ++i) {
        // 获取段的文本
        const char * text = whisper_full_get_segment_text(ctx, i);
        // 获取段的起始时间戳
        const int64_t t0 = whisper_full_get_segment_t0(ctx, i);
        // 获取段的结束时间戳
        const int64_t t1 = whisper_full_get_segment_t1(ctx, i);

        // 将起始时间戳和结束时间戳转换为时间字符串，存入结果向量
        result[i].emplace_back(to_timestamp(t0, true));
        result[i].emplace_back(to_timestamp(t1, true));
        result[i].emplace_back(text);
    }

    // 打印 whisper 的时间信息
    whisper_print_timings(ctx);
    // 释放 whisper 上下文
    whisper_free(ctx);

    return 0;
}

// Worker 类，继承自 Napi 的异步工作者类
class Worker : public Napi::AsyncWorker {
 public:
  // 构造函数，接受回调函数和参数
  Worker(Napi::Function& callback, whisper_params params)
      : Napi::AsyncWorker(callback), params(params) {}

  // 执行函数，调用 run 函数
  void Execute() override {
    run(params, result);
  }

  // 完成函数，处理结果向量并返回给 JavaScript
  void OnOK() override {
    Napi::HandleScope scope(Env());
    // 创建结果对象
    Napi::Object res = Napi::Array::New(Env(), result.size());
    // 遍历结果向量
    for (uint64_t i = 0; i < result.size(); ++i) {
      // 创建临时对象
      Napi::Object tmp = Napi::Array::New(Env(), 3);
      // 遍历每个段的结果
      for (uint64_t j = 0; j < 3; ++j) {
        // 将结果存入临时对象
        tmp[j] = Napi::String::New(Env(), result[i][j]);
      }
      // 将临时对象存入结果对象
      res[i] = tmp;
    }
    // 使用回调函数调用Call方法，传入空环境和res参数
    Callback().Call({Env().Null(), res});
  }

 private:
  // 定义whisper_params类型的params变量
  whisper_params params;
  // 定义存储vector<vector<string>>类型数据的result变量
  std::vector<std::vector<std::string>> result;
};

// 定义 whisper 函数，接受回调信息对象 info
Napi::Value whisper(const Napi::CallbackInfo& info) {
  // 获取当前环境
  Napi::Env env = info.Env();
  // 检查参数长度是否为 0 或第一个参数是否为对象，若不是则抛出类型错误异常
  if (info.Length() <= 0 || !info[0].IsObject()) {
    Napi::TypeError::New(env, "object expected").ThrowAsJavaScriptException();
  }
  // 定义 whisper_params 结构体
  whisper_params params;

  // 获取传入的对象参数
  Napi::Object whisper_params = info[0].As<Napi::Object>();
  // 获取语言参数并转换为字符串
  std::string language = whisper_params.Get("language").As<Napi::String>();
  // 获取模型参数并转换为字符串
  std::string model = whisper_params.Get("model").As<Napi::String>();
  // 获取输入文件名参数并转换为字符串
  std::string input = whisper_params.Get("fname_inp").As<Napi::String>();
  // 获取是否使用 GPU 参数并转换为布尔值
  bool use_gpu = whisper_params.Get("use_gpu").As<Napi::Boolean>();

  // 将参数赋值给 params 结构体
  params.language = language;
  params.model = model;
  params.fname_inp.emplace_back(input);
  params.use_gpu = use_gpu;

  // 获取回调函数
  Napi::Function callback = info[1].As<Napi::Function>();
  // 创建 Worker 对象并传入回调函数和参数结构体，将其加入队列
  Worker* worker = new Worker(callback, params);
  worker->Queue();
  // 返回 Undefined
  return env.Undefined();
}

// 初始化函数，将 whisper 函数导出
Napi::Object Init(Napi::Env env, Napi::Object exports) {
  exports.Set(
      Napi::String::New(env, "whisper"),
      Napi::Function::New(env, whisper)
  );
  return exports;
}

// 导出 whisper 模块
NODE_API_MODULE(whisper, Init);
```