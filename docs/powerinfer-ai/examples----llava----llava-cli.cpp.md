# `PowerInfer\examples\llava\llava-cli.cpp`

```
// 包含所需的头文件
#include "ggml.h"
#include "common.h"
#include "clip.h"
#include "llava.h"
#include "llama.h"

// 包含 base64 库的头文件
#include "base64.hpp"

// 包含标准输入输出库和动态内存分配库的头文件
#include <cstdio>
#include <cstdlib>
#include <vector>

// 定义一个静态函数，用于评估 tokens
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
        // 调用 llama_batch_get_one 函数进行解码
        if (llama_decode(ctx_llama, llama_batch_get_one(&tokens[i], n_eval, *n_past, 0))) {
// 打印错误信息，指示在评估过程中出现了问题
fprintf(stderr, "%s : failed to eval. token %d/%d (batch size %d, n_past %d)\n", __func__, i, N, n_batch, *n_past);
// 返回 false，表示评估失败
return false;
// 更新 n_past 的值
*n_past += n_eval;
// 返回 true，表示评估成功
return true;
}

// 评估指定 id 对应的 token
static bool eval_id(struct llama_context * ctx_llama, int id, int * n_past) {
    // 创建一个包含指定 id 的 token 数组
    std::vector<llama_token> tokens;
    tokens.push_back(id);
    // 调用 eval_tokens 函数进行评估
    return eval_tokens(ctx_llama, tokens, 1, n_past);
}

// 评估指定字符串对应的 token
static bool eval_string(struct llama_context * ctx_llama, const char* str, int n_batch, int * n_past, bool add_bos){
    // 将输入的字符串转换为 std::string 类型
    std::string              str2     = str;
    // 调用 llama_tokenize 函数将字符串转换为 token 数组
    std::vector<llama_token> embd_inp = ::llama_tokenize(ctx_llama, str2, add_bos);
    // 调用 eval_tokens 函数进行评估
    eval_tokens(ctx_llama, embd_inp, n_batch, n_past);
    // 返回 true，表示评估成功
    return true;
}
// TODO: 使用 common/sampling.h 中的函数
static llama_token sample_id(llama_context * ctx_llama, gpt_params & params) {
    auto & sparams = params.sparams;

    // 从用户输入中，根据参数进行下一个 token 的采样
    const float   temp      = sparams.temp;  // 温度参数
    const int32_t top_k     = sparams.top_k <= 0 ? llama_n_vocab(llama_get_model(ctx_llama)) : sparams.top_k;  // top-k 采样的参数
    const float   top_p     = sparams.top_p;  // top-p 采样的参数
    const float   tfs_z     = sparams.tfs_z;  // tfs_z 参数
    const float   typical_p = sparams.typical_p;  // 典型概率参数
    // const int32_t repeat_last_n   = sparams.repeat_last_n < 0 ? n_ctx : sparams.repeat_last_n;  // 重复上一个 token 的参数
    // const float   repeat_penalty  = sparams.repeat_penalty;  // 重复惩罚参数
    // const float   alpha_presence  = sparams.presence_penalty;  // 存在惩罚参数
    // const float   alpha_frequency = sparams.frequency_penalty;  // 频率惩罚参数
    const int     mirostat     = sparams.mirostat;  // mirostat 参数
    const float   mirostat_tau = sparams.mirostat_tau;  // mirostat_tau 参数
    const float   mirostat_eta = sparams.mirostat_eta;  // mirostat_eta 参数
    // const bool    penalize_nl     = sparams.penalize_nl;  // penalize_nl 参数
}
    // 声明一个名为id的llama_token类型的变量，并初始化为0
    llama_token id = 0;
    
    // 获取llama上下文中的logits
    auto logits  = llama_get_logits(ctx_llama);
    // 获取llama模型中的词汇量大小
    auto n_vocab = llama_n_vocab(llama_get_model(ctx_llama));

    // 对params.logit_bias映射应用
    for (auto it = sparams.logit_bias.begin(); it != sparams.logit_bias.end(); it++) {
        logits[it->first] += it->second;
    }

    // 声明一个名为candidates的llama_token_data类型的向量，并预留n_vocab个空间
    std::vector<llama_token_data> candidates;
    candidates.reserve(n_vocab);
    
    // 遍历n_vocab，将每个token_id和对应的logits[token_id]、0.0f组成的llama_token_data对象加入candidates向量中
    for (llama_token token_id = 0; token_id < n_vocab; token_id++) {
        candidates.emplace_back(llama_token_data{token_id, logits[token_id], 0.0f});
    }

    // 声明一个名为candidates_p的llama_token_data_array结构体，包含candidates的数据指针、大小和是否拥有数据的标志
    llama_token_data_array candidates_p = { candidates.data(), candidates.size(), false };

    // 如果temp小于等于0
    if (temp <= 0) {
        // 贪婪抽样
        id = llama_sample_token_greedy(ctx_llama, &candidates_p);
        // 使用贪婪算法从候选集中选择一个 token

    } else {
        if (mirostat == 1) {
            static float mirostat_mu = 2.0f * mirostat_tau;
            const  int mirostat_m    = 100;
            // 使用 Mirostat 算法从候选集中选择一个 token
            llama_sample_temp(ctx_llama, &candidates_p, temp);
            id = llama_sample_token_mirostat(ctx_llama, &candidates_p, mirostat_tau, mirostat_eta, mirostat_m, &mirostat_mu);
        } else if (mirostat == 2) {
            static float mirostat_mu = 2.0f * mirostat_tau;
            // 使用改进版 Mirostat 算法从候选集中选择一个 token
            llama_sample_temp(ctx_llama, &candidates_p, temp);
            id = llama_sample_token_mirostat_v2(ctx_llama, &candidates_p, mirostat_tau, mirostat_eta, &mirostat_mu);
        } else {
            // 温度抽样
            llama_sample_top_k(ctx_llama, &candidates_p, top_k, 1);
            // 尾部自由抽样
            llama_sample_tail_free(ctx_llama, &candidates_p, tfs_z, 1);
            // 典型抽样
            llama_sample_typical(ctx_llama, &candidates_p, typical_p, 1);
            // Top-p 抽样
            llama_sample_top_p(ctx_llama, &candidates_p, top_p, 1);
            // 温度抽样
            llama_sample_temp(ctx_llama, &candidates_p, temp);
            // 从候选集中选择一个 token
            id = llama_sample_token(ctx_llama, &candidates_p);
        }
    }
// 结束函数定义
}

// 返回采样的标识符
return id;
}

// 返回样本
static const char * sample(struct llama_context * ctx_llama, gpt_params & params, int * n_past) {
    // 通过调用sample_id函数获取id
    int id = sample_id(ctx_llama, params);
    // 静态字符串ret
    static std::string ret;
    // 如果id等于模型的结束标识符
    if (id == llama_token_eos(llama_get_model(ctx_llama))) {
        // 设置ret为结束标签
        ret = "</s>";
    } else {
        // 否则，将ret设置为id对应的片段
        ret = llama_token_to_piece(ctx_llama, id);
    }
    // 评估id
    eval_id(ctx_llama, id, n_past);
    // 返回ret的C风格字符串
    return ret.c_str();
}

// 图像base64标签的开始和结束
static const char* IMG_BASE64_TAG_BEGIN = "<img src=\"data:image/jpeg;base64,";
static const char* IMG_BASE64_TAG_END = "\">";
// 在提示中查找图片标签的起始和结束位置
static void find_image_tag_in_prompt(const std::string& prompt, size_t& begin_out, size_t& end_out) {
    begin_out = prompt.find(IMG_BASE64_TAG_BEGIN); // 查找图片标签的起始位置
    end_out = prompt.find(IMG_BASE64_TAG_END, (begin_out == std::string::npos) ? 0UL : begin_out); // 查找图片标签的结束位置
}

// 检查提示中是否包含图片
static bool prompt_contains_image(const std::string& prompt) {
    size_t begin, end;
    find_image_tag_in_prompt(prompt, begin, end); // 调用函数查找图片标签的起始和结束位置
    return (begin != std::string::npos); // 返回是否找到图片标签
}

// 用`replacement`替换提示中的base64图片标签
static llava_image_embed * llava_image_embed_make_with_prompt_base64(struct clip_ctx * ctx_clip, int n_threads, const std::string& prompt) {
    size_t img_base64_str_start, img_base64_str_end;
    find_image_tag_in_prompt(prompt, img_base64_str_start, img_base64_str_end); // 调用函数查找图片标签的起始和结束位置
    if (img_base64_str_start == std::string::npos || img_base64_str_end == std::string::npos) {
        fprintf(stderr, "%s: invalid base64 image tag. must be %s<base64 byte string>%s\n", __func__, IMG_BASE64_TAG_BEGIN, IMG_BASE64_TAG_END); // 输出错误信息
        return NULL; // 返回空指针
    }
}
    // 计算 base64 字符串的起始位置
    auto base64_bytes_start = img_base64_str_start + strlen(IMG_BASE64_TAG_BEGIN);
    // 计算 base64 字符串的长度
    auto base64_bytes_count = img_base64_str_end - base64_bytes_start;
    // 提取 base64 字符串
    auto base64_str = prompt.substr(base64_bytes_start, base64_bytes_count );

    // 计算解码后的字节数
    auto required_bytes = base64::required_encode_size(base64_str.size());
    // 创建存储解码后数据的容器
    auto img_bytes = std::vector<unsigned char>(required_bytes);
    // 解码 base64 字符串
    base64::decode(base64_str.begin(), base64_str.end(), img_bytes.begin());

    // 使用解码后的数据创建图像对象
    auto embed = llava_image_embed_make_with_bytes(ctx_clip, n_threads, img_bytes.data(), img_bytes.size());
    // 如果创建失败，打印错误信息并返回空指针
    if (!embed) {
        fprintf(stderr, "%s: could not load image from base64 string.\n", __func__);
        return NULL;
    }

    // 返回创建的图像对象
    return embed;
}

// 从提示中移除图像
static std::string remove_image_from_prompt(const std::string& prompt, const char * replacement = "") {
    // 定义起始和结束位置
    size_t begin, end;
    # 在提示中查找图像标签的起始和结束位置
    find_image_tag_in_prompt(prompt, begin, end);
    # 如果起始或结束位置未找到，则返回原始提示
    if (begin == std::string::npos || end == std::string::npos) {
        return prompt;
    }
    # 截取起始位置之前的部分作为前缀
    auto pre = prompt.substr(0, begin);
    # 截取结束位置之后的部分加上替换的内容作为后缀
    auto post = prompt.substr(end + strlen(IMG_BASE64_TAG_END));
    return pre + replacement + post;
}

# 定义 llava_context 结构体
struct llava_context {
    # 初始化指向 clip_ctx 结构体的指针
    struct clip_ctx * ctx_clip = NULL;
    # 初始化指向 llama_context 结构体的指针
    struct llama_context * ctx_llama = NULL;
    # 初始化指向 llama_model 结构体的指针
    struct llama_model * model = NULL;
};

# 显示额外信息的静态函数
static void show_additional_info(int /*argc*/, char ** argv) {
    # 打印建议使用较低的温度值以获得更好的质量
    printf("  note: a lower temperature value like 0.1 is recommended for better quality.\n");
}

# 加载图像的函数
static struct llava_image_embed * load_image(llava_context * ctx_llava, gpt_params * params) {
// 加载和预处理图像
llava_image_embed * embed = NULL;  // 创建指向图像嵌入对象的指针，并初始化为 NULL
auto prompt = params->prompt;  // 从参数中获取提示信息
if (prompt_contains_image(prompt)) {  // 检查提示信息中是否包含图像
    if (!params->image.empty()) {  // 如果参数中的图像路径不为空
        printf("using base64 encoded image instead of command line image path\n");  // 打印提示信息
    }
    embed = llava_image_embed_make_with_prompt_base64(ctx_llava->ctx_clip, params->n_threads, prompt);  // 使用提示信息中的 base64 编码图像创建图像嵌入对象
    if (!embed) {  // 如果图像嵌入对象为空
        fprintf(stderr, "%s: can't load image from prompt\n", __func__);  // 打印错误信息
        return NULL;  // 返回空指针
    }
    params->prompt = remove_image_from_prompt(prompt);  // 从提示信息中移除图像
} else {  // 如果提示信息中不包含图像
    embed = llava_image_embed_make_with_filename(ctx_llava->ctx_clip, params->n_threads, params->image.c_str());  // 使用参数中的图像路径创建图像嵌入对象
    if (!embed) {  // 如果图像嵌入对象为空
        fprintf(stderr, "%s: is %s really an image file?\n", __func__, params->image.c_str());  // 打印错误信息
        return NULL;  // 返回空指针
    }
}
    }

    return embed;
}

static void process_prompt(struct llava_context * ctx_llava, struct llava_image_embed * image_embed, gpt_params * params, const std::string & prompt) {
    int n_past = 0;

    // 设置最大目标长度为256，如果参数中没有指定预测长度，则默认为256
    const int max_tgt_len = params->n_predict < 0 ? 256 : params->n_predict;

    // llava 聊天格式为 "<system_prompt>\nUSER:<image_embeddings>\n<textual_prompt>\nASSISTANT:"
    // 使用图像嵌入进行 llava 评估
    llava_eval_image_embed(ctx_llava->ctx_llama, image_embed, params->n_batch, &n_past);
    // 使用文本提示进行评估
    eval_string(ctx_llava->ctx_llama, (prompt + "\nASSISTANT:").c_str(), params->n_batch, &n_past, false);

    // 生成响应
    printf("\n");

    for (int i = 0; i < max_tgt_len; i++) {
        // 从模型中采样生成文本
        const char * tmp = sample(ctx_llava->ctx_llama, *params, &n_past);
        // 如果临时字符串等于"</s>"，则跳出循环
        if (strcmp(tmp, "</s>") == 0) break;

        // 打印临时字符串
        printf("%s", tmp);
        // 刷新标准输出流
        fflush(stdout);
    }

    // 打印换行符
    printf("\n");
}


// 初始化 llava 上下文
static struct llava_context * llava_init(gpt_params * params) {
    // 获取剪贴板路径
    const char * clip_path = params->mmproj.c_str();

    // 获取提示信息，如果为空则使用默认提示
    auto prompt = params->prompt;
    if (prompt.empty()) {
        prompt = "describe the image in detail.";
    }

    // 加载剪贴板模型
    auto ctx_clip = clip_model_load(clip_path, /*verbosity=*/ 1);
// 初始化 LLAMA 后端
llama_backend_init(params->numa);

// 从 GPT 参数中创建 LLAMA 模型参数
llama_model_params model_params = llama_model_params_from_gpt_params(*params);

// 从文件加载 LLAMA 模型
llama_model * model = llama_load_model_from_file(params->model.c_str(), model_params);
if (model == NULL) {
    fprintf(stderr , "%s: error: unable to load model\n" , __func__);
    return NULL;
}

// 从 GPT 参数中创建 LLAMA 上下文参数
llama_context_params ctx_params = llama_context_params_from_gpt_params(*params);
ctx_params.n_ctx = params->n_ctx < 2048 ? 2048 : params->n_ctx; // 如果需要处理图像嵌入，我们需要更长的上下文大小

// 创建带有模型的 LLAMA 上下文
llama_context * ctx_llama = llama_new_context_with_model(model, ctx_params);

if (ctx_llama == NULL) {
    fprintf(stderr , "%s: error: failed to create the llama_context\n" , __func__);
    return NULL;
}
    // 分配内存以存储 llava_context 结构体
    auto ctx_llava = (struct llava_context *)malloc(sizeof(llava_context));

    // 将 ctx_llama 指针赋值给 ctx_llava 结构体中的 ctx_llama 成员
    ctx_llava->ctx_llama = ctx_llama;
    // 将 ctx_clip 指针赋值给 ctx_llava 结构体中的 ctx_clip 成员
    ctx_llava->ctx_clip = ctx_clip;
    // 将 model 指针赋值给 ctx_llava 结构体中的 model 成员
    ctx_llava->model = model;
    // 返回指向 ctx_llava 结构体的指针
    return ctx_llava;
}

// 释放 llava_context 结构体占用的内存
static void llava_free(struct llava_context * ctx_llava) {
    // 如果 ctx_clip 指针不为空，则释放其占用的内存并将指针置为 NULL
    if (ctx_llava->ctx_clip) {
        clip_free(ctx_llava->ctx_clip);
        ctx_llava->ctx_clip = NULL;
    }

    // 释放 ctx_llama 指针指向的内存
    llama_free(ctx_llava->ctx_llama);
    // 释放 model 指针指向的内存
    llama_free_model(ctx_llava->model);
    // 释放 llama 后端资源
    llama_backend_free();
}

// 主函数
int main(int argc, char ** argv) {
    # 初始化 ggml 时间
    ggml_time_init();

    # 声明 gpt_params 结构体变量
    gpt_params params;

    # 解析命令行参数，如果解析失败则显示额外信息并返回 1
    if (!gpt_params_parse(argc, argv, params)) {
        show_additional_info(argc, argv);
        return 1;
    }

    # 检查参数中 mmproj 是否为空，以及 image 是否为空并且 prompt 不包含图片，则显示用法信息和额外信息并返回 1
    if (params.mmproj.empty() || (params.image.empty() && !prompt_contains_image(params.prompt))) {
        gpt_print_usage(argc, argv, params);
        show_additional_info(argc, argv);
        return 1;
    }

    # 初始化 llava 上下文，如果初始化失败则打印错误信息并返回 1
    auto ctx_llava = llava_init(&params);
    if (ctx_llava == NULL) {
        fprintf(stderr, "%s: error: failed to init llava\n", __func__);
        return 1;
    }
    // 加载图像并嵌入到上下文中
    auto image_embed = load_image(ctx_llava, &params);

    // 处理提示信息
    process_prompt(ctx_llava, image_embed, &params, params.prompt);

    // 打印时间信息
    llama_print_timings(ctx_llava->ctx_llama);

    // 释放图像嵌入的内存
    llava_image_embed_free(image_embed);
    // 释放上下文的内存
    llava_free(ctx_llava);
    // 返回成功状态码
    return 0;
}
```