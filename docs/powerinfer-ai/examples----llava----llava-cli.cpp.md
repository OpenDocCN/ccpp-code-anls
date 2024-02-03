# `PowerInfer\examples\llava\llava-cli.cpp`

```cpp
// 包含头文件
#include "ggml.h"
#include "common.h"
#include "clip.h"
#include "llava.h"
#include "llama.h"

// 包含第三方库的头文件
#include "base64.hpp"

// 包含标准输入输出库和标准库
#include <cstdio>
#include <cstdlib>
#include <vector>

// 定义一个静态函数，用于评估一组 tokens
static bool eval_tokens(struct llama_context * ctx_llama, std::vector<llama_token> tokens, int n_batch, int * n_past) {
    // 获取 tokens 的数量
    int N = (int) tokens.size();
    // 遍历 tokens
    for (int i = 0; i < N; i += n_batch) {
        // 计算当前批次需要评估的 tokens 数量
        int n_eval = (int) tokens.size() - i;
        if (n_eval > n_batch) {
            n_eval = n_batch;
        }
        // 调用 llama_batch_get_one 函数进行评估
        if (llama_decode(ctx_llama, llama_batch_get_one(&tokens[i], n_eval, *n_past, 0))) {
            // 如果评估失败，输出错误信息并返回 false
            fprintf(stderr, "%s : failed to eval. token %d/%d (batch size %d, n_past %d)\n", __func__, i, N, n_batch, *n_past);
            return false;
        }
        // 更新已评估的 tokens 数量
        *n_past += n_eval;
    }
    // 返回 true 表示评估成功
    return true;
}

// 定义一个静态函数，用于评估单个 token
static bool eval_id(struct llama_context * ctx_llama, int id, int * n_past) {
    // 创建一个只包含一个 token 的 vector
    std::vector<llama_token> tokens;
    tokens.push_back(id);
    // 调用 eval_tokens 函数进行评估
    return eval_tokens(ctx_llama, tokens, 1, n_past);
}

// 定义一个静态函数，用于评估字符串
static bool eval_string(struct llama_context * ctx_llama, const char* str, int n_batch, int * n_past, bool add_bos){
    // 将输入的字符串转换为 std::string 类型
    std::string              str2     = str;
    // 调用 llama_tokenize 函数对字符串进行分词，并得到 tokens
    std::vector<llama_token> embd_inp = ::llama_tokenize(ctx_llama, str2, add_bos);
    // 调用 eval_tokens 函数进行评估
    eval_tokens(ctx_llama, embd_inp, n_batch, n_past);
    // 返回 true 表示评估成功
    return true;
}

// TODO: use common/sampling.h
// 定义一个静态函数，用于从模型中采样一个 token
static llama_token sample_id(llama_context * ctx_llama, gpt_params & params) {
    // 获取采样参数
    auto & sparams = params.sparams;
    const float   temp      = sparams.temp;
    const int32_t top_k     = sparams.top_k <= 0 ? llama_n_vocab(llama_get_model(ctx_llama)) : sparams.top_k;
    const float   top_p     = sparams.top_p;
    const float   tfs_z     = sparams.tfs_z;
    const float   typical_p = sparams.typical_p;
    // const int32_t repeat_last_n   = sparams.repeat_last_n < 0 ? n_ctx : sparams.repeat_last_n;
    // const float   repeat_penalty  = sparams.repeat_penalty;
}
    // 定义常量 alpha_presence，表示存在惩罚
    // const float   alpha_presence  = sparams.presence_penalty;
    // 定义常量 alpha_frequency，表示频率惩罚
    // const float   alpha_frequency = sparams.frequency_penalty;
    // 定义常量 mirostat，表示mirostat参数
    const int     mirostat     = sparams.mirostat;
    // 定义常量 mirostat_tau，表示mirostat的tau参数
    const float   mirostat_tau = sparams.mirostat_tau;
    // 定义常量 mirostat_eta，表示mirostat的eta参数
    const float   mirostat_eta = sparams.mirostat_eta;
    // 定义常量 penalize_nl，表示是否对换行符进行惩罚
    // const bool    penalize_nl     = sparams.penalize_nl;

    // 初始化变量 id，赋值为0
    llama_token id = 0;
    {
        // 获取模型的logits
        auto logits  = llama_get_logits(ctx_llama);
        // 获取词汇表大小
        auto n_vocab = llama_n_vocab(llama_get_model(ctx_llama));

        // 应用params.logit_bias映射
        for (auto it = sparams.logit_bias.begin(); it != sparams.logit_bias.end(); it++) {
            logits[it->first] += it->second;
        }

        // 创建候选词向量
        std::vector<llama_token_data> candidates;
        candidates.reserve(n_vocab);
        for (llama_token token_id = 0; token_id < n_vocab; token_id++) {
            candidates.emplace_back(llama_token_data{token_id, logits[token_id], 0.0f});
        }

        // 候选词向量数组
        llama_token_data_array candidates_p = { candidates.data(), candidates.size(), false };

        if (temp <= 0) {
              // 贪婪抽样
            id = llama_sample_token_greedy(ctx_llama, &candidates_p);
        } else {
            if (mirostat == 1) {
                static float mirostat_mu = 2.0f * mirostat_tau;
                const  int mirostat_m    = 100;
                llama_sample_temp(ctx_llama, &candidates_p, temp);
                id = llama_sample_token_mirostat(ctx_llama, &candidates_p, mirostat_tau, mirostat_eta, mirostat_m, &mirostat_mu);
            } else if (mirostat == 2) {
                static float mirostat_mu = 2.0f * mirostat_tau;
                llama_sample_temp(ctx_llama, &candidates_p, temp);
                id = llama_sample_token_mirostat_v2(ctx_llama, &candidates_p, mirostat_tau, mirostat_eta, &mirostat_mu);
            } else {
                  // 温度抽样
                llama_sample_top_k(ctx_llama, &candidates_p, top_k, 1);
                llama_sample_tail_free(ctx_llama, &candidates_p, tfs_z, 1);
                llama_sample_typical(ctx_llama, &candidates_p, typical_p, 1);
                llama_sample_top_p(ctx_llama, &candidates_p, top_p, 1);
                llama_sample_temp(ctx_llama, &candidates_p, temp);
                id = llama_sample_token(ctx_llama, &candidates_p);
            }
        }
    }

    return id;
// 定义一个静态函数，用于从 llama_context 中获取样本，并返回样本的字符串表示
static const char * sample(struct llama_context * ctx_llama, gpt_params & params, int * n_past) {
    // 从 llama_context 中获取样本的 id
    int id = sample_id(ctx_llama, params);
    // 定义静态字符串变量 ret
    static std::string ret;
    // 如果样本的 id 是结束符，则将 ret 设置为 "</s>"
    if (id == llama_token_eos(llama_get_model(ctx_llama))) {
        ret = "</s>";
    } else {
        // 否则将 ret 设置为样本 id 对应的字符串表示
        ret = llama_token_to_piece(ctx_llama, id);
    }
    // 调用 eval_id 函数，更新 n_past
    eval_id(ctx_llama, id, n_past);
    // 返回 ret 的 C 字符串表示
    return ret.c_str();
}

// 定义两个静态常量字符串，用于表示 base64 图片标签的开始和结束
static const char* IMG_BASE64_TAG_BEGIN = "<img src=\"data:image/jpeg;base64,";
static const char* IMG_BASE64_TAG_END = "\">";

// 定义一个静态函数，用于在 prompt 中查找图片标签的起始和结束位置
static void find_image_tag_in_prompt(const std::string& prompt, size_t& begin_out, size_t& end_out) {
    // 在 prompt 中查找图片标签的开始位置
    begin_out = prompt.find(IMG_BASE64_TAG_BEGIN);
    // 在 prompt 中查找图片标签的结束位置，从开始位置开始查找
    end_out = prompt.find(IMG_BASE64_TAG_END, (begin_out == std::string::npos) ? 0UL : begin_out);
}

// 定义一个静态函数，用于检查 prompt 中是否包含图片标签
static bool prompt_contains_image(const std::string& prompt) {
    // 定义变量 begin 和 end，用于存储图片标签的起始和结束位置
    size_t begin, end;
    // 调用 find_image_tag_in_prompt 函数，在 prompt 中查找图片标签的起始和结束位置
    find_image_tag_in_prompt(prompt, begin, end);
    // 返回是否找到了图片标签
    return (begin != std::string::npos);
}

// 定义一个静态函数，用于替换 prompt 中的 base64 图片标签为 replacement
static llava_image_embed * llava_image_embed_make_with_prompt_base64(struct clip_ctx * ctx_clip, int n_threads, const std::string& prompt) {
    // 定义变量 img_base64_str_start 和 img_base64_str_end，用于存储 base64 图片标签的起始和结束位置
    size_t img_base64_str_start, img_base64_str_end;
    // 调用 find_image_tag_in_prompt 函数，在 prompt 中查找 base64 图片标签的起始和结束位置
    find_image_tag_in_prompt(prompt, img_base64_str_start, img_base64_str_end);
    // 如果没有找到 base64 图片标签，则输出错误信息并返回空指针
    if (img_base64_str_start == std::string::npos || img_base64_str_end == std::string::npos) {
        fprintf(stderr, "%s: invalid base64 image tag. must be %s<base64 byte string>%s\n", __func__, IMG_BASE64_TAG_BEGIN, IMG_BASE64_TAG_END);
        return NULL;
    }

    // 计算 base64 字符串的起始位置和长度
    auto base64_bytes_start = img_base64_str_start + strlen(IMG_BASE64_TAG_BEGIN);
    auto base64_bytes_count = img_base64_str_end - base64_bytes_start;
    // 获取 base64 字符串
    auto base64_str = prompt.substr(base64_bytes_start, base64_bytes_count );

    // 计算编码后的字节数，并创建相应大小的字节数组
    auto required_bytes = base64::required_encode_size(base64_str.size());
    auto img_bytes = std::vector<unsigned char>(required_bytes);
    # 使用base64解码函数将base64字符串解码为字节流，存储在img_bytes中
    base64::decode(base64_str.begin(), base64_str.end(), img_bytes.begin());
    
    # 使用llava_image_embed_make_with_bytes函数创建一个嵌入图像对象，传入上下文、线程数、图像字节流的数据和大小
    auto embed = llava_image_embed_make_with_bytes(ctx_clip, n_threads, img_bytes.data(), img_bytes.size());
    # 如果创建失败，打印错误信息并返回空指针
    if (!embed) {
        fprintf(stderr, "%s: could not load image from base64 string.\n", __func__);
        return NULL;
    }
    
    # 返回嵌入图像对象
    return embed;
// 从提示中移除图像，并返回处理后的字符串
static std::string remove_image_from_prompt(const std::string& prompt, const char * replacement = "") {
    // 在提示中查找图像标签的起始和结束位置
    size_t begin, end;
    find_image_tag_in_prompt(prompt, begin, end);
    // 如果未找到图像标签，则直接返回原始提示
    if (begin == std::string::npos || end == std::string::npos) {
        return prompt;
    }
    // 提取图像标签前后的内容，并替换为指定的字符串
    auto pre = prompt.substr(0, begin);
    auto post = prompt.substr(end + strlen(IMG_BASE64_TAG_END));
    return pre + replacement + post;
}

// 定义 llava_context 结构体
struct llava_context {
    struct clip_ctx * ctx_clip = NULL;
    struct llama_context * ctx_llama = NULL;
    struct llama_model * model = NULL;
};

// 显示额外信息
static void show_additional_info(int /*argc*/, char ** argv) {
    // 打印示例用法信息
    printf("\n example usage: %s -m <llava-v1.5-7b/ggml-model-q5_k.gguf> --mmproj <llava-v1.5-7b/mmproj-model-f16.gguf> --image <path/to/an/image.jpg> [--temp 0.1] [-p \"describe the image in detail.\"]\n", argv[0]);
    // 提示使用较低的温度值以获得更好的质量
    printf("  note: a lower temperature value like 0.1 is recommended for better quality.\n");
}

// 加载图像
static struct llava_image_embed * load_image(llava_context * ctx_llava, gpt_params * params) {
    // 加载和预处理图像
    llava_image_embed * embed = NULL;
    auto prompt = params->prompt;
    // 如果提示中包含图像
    if (prompt_contains_image(prompt)) {
        // 如果未指定图像路径，则使用 base64 编码的图像
        if (!params->image.empty()) {
            printf("using base64 encoded image instead of command line image path\n");
        }
        // 使用提示中的 base64 编码图像创建 llava_image_embed 对象
        embed = llava_image_embed_make_with_prompt_base64(ctx_llava->ctx_clip, params->n_threads, prompt);
        // 如果创建失败，则打印错误信息并返回 NULL
        if (!embed) {
            fprintf(stderr, "%s: can't load image from prompt\n", __func__);
            return NULL;
        }
        // 从提示中移除图像
        params->prompt = remove_image_from_prompt(prompt);
    } else {
        // 使用文件名加载图像
        embed = llava_image_embed_make_with_filename(ctx_llava->ctx_clip, params->n_threads, params->image.c_str());
        // 如果创建失败，则打印错误信息并返回 NULL
        if (!embed) {
            fprintf(stderr, "%s: is %s really an image file?\n", __func__, params->image.c_str());
            return NULL;
        }
    }
    // 返回加载的图像对象
    return embed;
}
static void process_prompt(struct llava_context * ctx_llava, struct llava_image_embed * image_embed, gpt_params * params, const std::string & prompt) {
    int n_past = 0;

    const int max_tgt_len = params->n_predict < 0 ? 256 : params->n_predict;

    // llava chat format is "<system_prompt>\nUSER:<image_embeddings>\n<textual_prompt>\nASSISTANT:"
    // 在 llava 聊天格式中，包含了系统提示、用户的图像嵌入、文本提示和助手的对话格式
    eval_string(ctx_llava->ctx_llama, "A chat between a curious human and an artificial intelligence assistant.  The assistant gives helpful, detailed, and polite answers to the human's questions.\nUSER:", params->n_batch, &n_past, true);
    // 评估字符串，将聊天格式添加到上下文中
    llava_eval_image_embed(ctx_llava->ctx_llama, image_embed, params->n_batch, &n_past);
    // 评估图像嵌入，将图像嵌入添加到上下文中
    eval_string(ctx_llava->ctx_llama, (prompt + "\nASSISTANT:").c_str(), params->n_batch, &n_past, false);
    // 评估字符串，将用户的文本提示和助手的对话格式添加到上下文中

    // generate the response
    // 生成回复
    printf("\n");

    for (int i = 0; i < max_tgt_len; i++) {
        const char * tmp = sample(ctx_llava->ctx_llama, *params, &n_past);
        // 从上下文中获取样本
        if (strcmp(tmp, "</s>") == 0) break;
        // 如果样本为结束符，则跳出循环

        printf("%s", tmp);
        fflush(stdout);
    }

    printf("\n");
}

static struct llava_context * llava_init(gpt_params * params) {
    const char * clip_path = params->mmproj.c_str();

    auto prompt = params->prompt;
    if (prompt.empty()) {
        prompt = "describe the image in detail.";
    }

    auto ctx_clip = clip_model_load(clip_path, /*verbosity=*/ 1);
    // 加载 CLIP 模型

    llama_backend_init(params->numa);
    // 初始化 LLAMA 后端

    llama_model_params model_params = llama_model_params_from_gpt_params(*params);
    // 从 GPT 参数中获取 LLAMA 模型参数

    llama_model * model = llama_load_model_from_file(params->model.c_str(), model_params);
    // 从文件中加载 LLAMA 模型
    if (model == NULL) {
        fprintf(stderr , "%s: error: unable to load model\n" , __func__);
        return NULL;
    }

    llama_context_params ctx_params = llama_context_params_from_gpt_params(*params);
    // 从 GPT 参数中获取 LLAMA 上下文参数
    ctx_params.n_ctx           = params->n_ctx < 2048 ? 2048 : params->n_ctx; // we need a longer context size to process image embeddings
    // 我们需要更长的上下文大小来处理图像嵌入
}
    // 创建一个 llama_context 结构体指针，并使用给定的模型和上下文参数进行初始化
    llama_context * ctx_llama = llama_new_context_with_model(model, ctx_params);

    // 如果 llama_context 为空，打印错误信息并返回空指针
    if (ctx_llama == NULL) {
        fprintf(stderr , "%s: error: failed to create the llama_context\n" , __func__);
        return NULL;
    }

    // 为 llava_context 结构体分配内存空间，并将其指针赋值给 ctx_llava
    auto ctx_llava = (struct llava_context *)malloc(sizeof(llava_context));

    // 将 ctx_llama 指针赋值给 ctx_llava 结构体的 ctx_llama 成员
    ctx_llava->ctx_llama = ctx_llama;
    // 将 ctx_clip 指针赋值给 ctx_llava 结构体的 ctx_clip 成员
    ctx_llava->ctx_clip = ctx_clip;
    // 将 model 指针赋值给 ctx_llava 结构体的 model 成员
    ctx_llava->model = model;
    // 返回 ctx_llava 结构体指针
    return ctx_llava;
# 释放 llava_context 结构体的内存
static void llava_free(struct llava_context * ctx_llava) {
    # 如果 ctx_clip 不为空，则释放其内存，并将指针置为 NULL
    if (ctx_llava->ctx_clip) {
        clip_free(ctx_llava->ctx_clip);
        ctx_llava->ctx_clip = NULL;
    }
    # 释放 ctx_llava->ctx_llama 指向的内存
    llama_free(ctx_llava->ctx_llama);
    # 释放 ctx_llava->model 指向的内存
    llama_free_model(ctx_llava->model);
    # 释放 llama 后端相关资源
    llama_backend_free();
}

# 主函数
int main(int argc, char ** argv) {
    # 初始化时间
    ggml_time_init();

    # 定义 gpt_params 结构体
    gpt_params params;

    # 解析命令行参数，如果解析失败则显示额外信息并返回 1
    if (!gpt_params_parse(argc, argv, params)) {
        show_additional_info(argc, argv);
        return 1;
    }
    # 如果 mmproj 为空，或者 image 为空且 prompt 不包含图片，则打印用法信息和额外信息，并返回 1
    if (params.mmproj.empty() || (params.image.empty() && !prompt_contains_image(params.prompt))) {
        gpt_print_usage(argc, argv, params);
        show_additional_info(argc, argv);
        return 1;
    }

    # 初始化 llava_context 结构体
    auto ctx_llava = llava_init(&params);
    # 如果初始化失败，则打印错误信息并返回 1
    if (ctx_llava == NULL) {
        fprintf(stderr, "%s: error: failed to init llava\n", __func__);
        return 1;
    }

    # 加载图片
    auto image_embed = load_image(ctx_llava, &params);

    # 处理提示信息
    process_prompt(ctx_llava, image_embed, &params, params.prompt);

    # 打印 llama 的时间信息
    llama_print_timings(ctx_llava->ctx_llama);

    # 释放 image_embed 的内存
    llava_image_embed_free(image_embed);
    # 释放 ctx_llava 的内存
    llava_free(ctx_llava);
    # 返回 0
    return 0;
}
```