# `PowerInfer\examples\perplexity\perplexity.cpp`

```
// 包含自定义的头文件 common.h 和 llama.h
#include "common.h"
#include "llama.h"

// 包含标准库的头文件
#include <cmath>
#include <cstdio>
#include <cstring>
#include <ctime>
#include <sstream>
#include <thread>
#include <mutex>
#include <vector>

// 如果编译器是 MSC_VER，则禁用警告 4244 和 4267，这两个警告是可能的数据丢失
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// 定义结构体 results_perplexity，包含 tokens、ppl_value 和 logits 三个成员变量
struct results_perplexity {
    std::vector<llama_token> tokens; // 用于存储 llama_token 类型的向量
    double                   ppl_value; // 用于存储 double 类型的概率值
    std::vector<float>       logits; // 用于存储 float 类型的向量
// 定义一个结构体，包含一个存储概率的浮点数向量
struct results_perplexity {
    std::vector<float> probs;
};

// 定义一个结构体，包含对数softmax、logit和概率
struct results_log_softmax {
    double log_softmax;
    float  logit;
    float  prob;
};

// 定义一个静态函数，用于写入日志文件
static void write_logfile(
    const llama_context * ctx, const gpt_params & params, const llama_model * model,
    const struct results_perplexity & results
) {
    // 如果日志目录为空，则直接返回
    if (params.logdir.empty()) {
        return;
    }

    // 如果参数中包含HellaSwag，则打印警告信息并返回
    if (params.hellaswag) {
        fprintf(stderr, "%s: warning: logging results is not implemented for HellaSwag. No files will be written.\n", __func__);
        return;
    }
```
    }

    // 获取可排序的时间戳
    const std::string timestamp = get_sortable_timestamp();

    // 创建具有父目录的目录
    const bool success = create_directory_with_parents(params.logdir);
    if (!success) {
        // 如果创建目录失败，则打印警告信息
        fprintf(stderr, "%s: warning: failed to create logdir %s, cannot write logfile\n",
                __func__, params.logdir.c_str());
        return;
    }

    // 构建日志文件路径
    const std::string logfile_path = params.logdir + timestamp + ".yml";
    // 打开日志文件
    FILE * logfile = fopen(logfile_path.c_str(), "w");

    // 如果无法打开日志文件，则打印错误信息
    if (logfile == NULL) {
        fprintf(stderr, "%s: failed to open logfile %s\n", __func__, logfile_path.c_str());
        return;
    }

    // 向日志文件写入信息
    fprintf(logfile, "binary: main\n");
    // 声明一个长度为128的字符数组，用于存储模型描述
    char model_desc[128];
    // 调用llama_model_desc函数，将模型描述存储到model_desc数组中
    llama_model_desc(model, model_desc, sizeof(model_desc));
    // 将非结果信息以YAML格式写入日志文件
    dump_non_result_info_yaml(logfile, params, ctx, timestamp, results.tokens, model_desc);

    // 在日志文件中打印分隔符和Perplexity Results标题
    fprintf(logfile, "\n");
    fprintf(logfile, "######################\n");
    fprintf(logfile, "# Perplexity Results #\n");
    fprintf(logfile, "######################\n");
    fprintf(logfile, "\n");

    // 将logits以YAML格式写入日志文件
    dump_vector_float_yaml(logfile, "logits", results.logits);
    // 在日志文件中打印ppl_value的值
    fprintf(logfile, "ppl_value: %f\n", results.ppl_value);
    // 将probs以YAML格式写入日志文件
    dump_vector_float_yaml(logfile, "probs", results.probs);

    // 将上下文的时间信息以YAML格式写入日志文件
    llama_dump_timing_info_yaml(logfile, ctx);
    // 关闭日志文件
    fclose(logfile);
}

// 定义一个softmax函数，接受一个float类型的向量logits作为参数，返回一个概率向量probs
static std::vector<float> softmax(const std::vector<float>& logits) {
    // 创建一个与logits相同长度的probs向量
    std::vector<float> probs(logits.size());
    # 初始化最大logit值为logits数组的第一个元素
    float max_logit = logits[0];
    # 遍历logits数组，找到最大的logit值
    for (float v : logits) {
        max_logit = std::max(max_logit, v);
    }
    # 初始化指数求和变量为0
    double sum_exp = 0.0;
    # 遍历logits数组，进行log_softmax操作
    for (size_t i = 0; i < logits.size(); i++) {
        # 为了数值稳定性，将当前logit值减去最大的logit值
        const float logit = logits[i] - max_logit;
        # 计算logit值的指数
        const float exp_logit = expf(logit);
        # 将指数值加入到指数求和变量中
        sum_exp += exp_logit;
        # 将指数值存入probs数组中
        probs[i] = exp_logit;
    }
    # 将probs数组中的每个值除以指数求和变量，得到最终的概率值
    for (size_t i = 0; i < probs.size(); i++) {
        probs[i] /= sum_exp;
    }
    # 返回概率数组
    return probs;
}

static results_log_softmax log_softmax(int n_vocab, const float * logits, int tok) {
    # 初始化最大logit值为logits数组的第一个元素
    float max_logit = logits[0];
    // 遍历logits数组，找出最大值
    for (int i = 1; i < n_vocab; ++i) {
        max_logit = std::max(max_logit, logits[i]);
    }
    // 初始化sum_exp变量
    double sum_exp = 0.0;
    // 遍历logits数组，计算expf(logits[i] - max_logit)的和
    for (int i = 0; i < n_vocab; ++i) {
        sum_exp += expf(logits[i] - max_logit);
    }
    // 返回包含三个值的数组
    return {logits[tok] - max_logit - log(sum_exp), logits[tok], expf(logits[tok] - max_logit) / (float) sum_exp};
}

// 处理logits的函数
static void process_logits(
    int n_vocab, const float * logits, const int * tokens, int n_token, std::vector<std::thread> & workers,
    double & nll, double & nll2, float * logit_history, float * prob_history
) {
    // 创建互斥锁
    std::mutex mutex;
    // 初始化计数器
    int counter = 0;
    // 定义lambda表达式compute，用于并行计算
    auto compute = [&mutex, &counter, &nll, &nll2, logit_history, prob_history, n_vocab, logits, tokens, n_token] () {
        // 初始化本地变量
        double local_nll  = 0;
        double local_nll2 = 0;
        // 循环处理logits数组
        while (true) {
// 创建互斥锁，确保多线程访问的安全性
std::unique_lock<std::mutex> lock(mutex);
// 获取当前计数器的值，并将其赋给 i，然后递增计数器
int i = counter++;
// 如果 i 大于等于 n_token，则累加 local_nll 到 nll，累加 local_nll2 到 nll2，然后跳出循环
if (i >= n_token) {
    nll += local_nll; nll2 += local_nll2;
    break;
}
// 释放互斥锁
lock.unlock();
// 调用 log_softmax 函数计算结果
const results_log_softmax results = log_softmax(n_vocab, logits + i*n_vocab, tokens[i+1]);
// 计算结果的负对数似然值
const double v = -results.log_softmax;
// 累加到 local_nll
local_nll += v;
// 累加 v 的平方到 local_nll2
local_nll2 += v*v;

// 将结果记录到 logit_history 和 prob_history 数组中
logit_history[i] = results.logit;
prob_history[i]  = results.prob;
// 创建并启动线程执行 compute 函数
};
for (auto & w : workers) {
    w = std::thread(compute);
}
// 执行 compute 函数
compute();
    // 遍历 workers 容器中的每个元素，调用其 join() 方法
    for (auto & w : workers) {
        w.join();
    }
}

static results_perplexity perplexity_v2(llama_context * ctx, const gpt_params & params) {
    // 下载：https://s3.amazonaws.com/research.metamind.io/wikitext/wikitext-2-raw-v1.zip?ref=salesforce-research
    // 运行 `./perplexity -m models/7B/ggml-model-q4_0.bin -f wiki.test.raw`
    // 输出：`perplexity: 13.5106 [114/114]`
    // 在评估之前，每个块都会添加 BOS 标记

    // 检查模型词汇类型是否为 SPM
    const bool is_spm = llama_vocab_type(llama_get_model(ctx)) == LLAMA_VOCAB_TYPE_SPM;
    // 如果模型词汇类型为 SPM，则添加 BOS 标记
    const bool add_bos = is_spm;

    // 打印信息到标准错误输出
    fprintf(stderr, "%s: tokenizing the input ..\n", __func__);

    // 使用 llama_tokenize 函数对输入进行标记化，结果存储在 tokens 容器中
    std::vector<llama_token> tokens = ::llama_tokenize(ctx, params.prompt, add_bos);

    // 获取上下文的大小
    const int n_ctx = llama_n_ctx(ctx);
    # 检查 tokens 的数量是否足够用于计算困惑度
    if (int(tokens.size()) < 2*n_ctx) {
        # 如果 tokens 数量不够，输出错误信息并返回空结果
        fprintf(stderr, "%s: you need at least %d tokens to evaluate perplexity with a context of %d\n",__func__,2*n_ctx,
                n_ctx);
        fprintf(stderr, "%s: the data file you provided tokenizes to only %zu tokens\n",__func__,tokens.size());
        return {std::move(tokens), 0., {}, {}};
    }

    # 初始化 logit_history 和 prob_history 数组
    std::vector<float> logit_history;
    std::vector<float> prob_history;

    # 调整 logit_history 和 prob_history 数组的大小为 tokens 数组的大小
    logit_history.resize(tokens.size());
    prob_history.resize(tokens.size());

    # 检查参数中的 ppl_stride 是否合法
    if (params.ppl_stride <= 0) {
        # 如果 ppl_stride 不合法，输出错误信息并返回空结果
        fprintf(stderr, "%s: stride is %d but must be greater than zero!\n",__func__,params.ppl_stride);
        return {tokens, -1, logit_history, prob_history};
    }

    # 初始化计算块的大小为 n_ctx
    const int calc_chunk = n_ctx;
    # 打印函数名、tokens的数量和计算块大小
    fprintf(stderr, "%s: have %zu tokens. Calculation chunk = %d\n", __func__, tokens.size(), calc_chunk);

    # 如果tokens的数量小于等于计算块大小
    if (int(tokens.size()) <= calc_chunk) {
        # 打印警告信息并返回结果
        fprintf(stderr, "%s: there are only %zu tokens, this is not enough for a context size of %d and stride %d\n",__func__,
                tokens.size(), n_ctx, params.ppl_stride);
        return {tokens, -1, logit_history, prob_history};
    }

    # 计算最大的块数量
    const int n_chunk_max = (tokens.size() - calc_chunk + params.ppl_stride - 1)  / params.ppl_stride;

    # 计算实际的块数量，如果参数中指定了块数量则取参数中的值，否则取最大块数量和参数中指定的块数量的较小值
    const int n_chunk = params.n_chunks < 0 ? n_chunk_max : std::min(params.n_chunks, n_chunk_max);
    # 获取词汇表的大小
    const int n_vocab = llama_n_vocab(llama_get_model(ctx));
    # 获取批处理的大小
    const int n_batch = params.n_batch;

    # 初始化计数和负对数似然
    int count = 0;
    double nll = 0.0;

    # 打印计算困惑度的信息
    fprintf(stderr, "%s: calculating perplexity over %d chunks, batch_size=%d\n", __func__, n_chunk, n_batch);

    # 循环计算每个块的困惑度
    for (int i = 0; i < n_chunk; ++i) {
        // 计算每个批次的起始位置
        const int start =     i * params.ppl_stride;
        // 计算每个批次的结束位置
        const int end   = start + calc_chunk;

        // 计算需要的批次数
        const int num_batches = (calc_chunk + n_batch - 1) / n_batch;
        // 输出日志，显示正在评估的范围和使用的批次数
        //fprintf(stderr, "%s: evaluating %d...%d using %d batches\n", __func__, start, end, num_batches);

        // 创建一个存储浮点数的向量
        std::vector<float> logits;

        // 获取当前时间
        const auto t_start = std::chrono::high_resolution_clock::now();

        // 清空 KV 缓存
        llama_kv_cache_clear(ctx);

        // 循环处理每个批次
        for (int j = 0; j < num_batches; ++j) {
            // 计算当前批次的起始位置和大小
            const int batch_start = start + j * n_batch;
            const int batch_size  = std::min(end - batch_start, n_batch);

            // 输出日志，显示当前批次的信息
            //fprintf(stderr, "    Batch %d: starts at %d, size is %d, n_past is %d\n",j,batch_start,batch_size,j * n_batch);
            // 如果 llama_decode 函数返回失败，则输出日志
            if (llama_decode(ctx, llama_batch_get_one(tokens.data() + batch_start, batch_size, j * n_batch, 0))) {
                //fprintf(stderr, "%s : failed to eval\n", __func__);
            // 返回一个包含tokens, -1, logit_history, prob_history的字典
            return {tokens, -1, logit_history, prob_history};
        }

        // 保存原始token，并在eval之后恢复它
        const auto token_org = tokens[batch_start];

        // 对于每个chunk的第一个batch，添加BOS token
        if (add_bos && j == 0) {
            tokens[batch_start] = llama_token_bos(llama_get_model(ctx));
        }

        // 获取当前batch的logits，并将其添加到logits中
        const auto batch_logits = llama_get_logits(ctx);
        logits.insert(logits.end(), batch_logits, batch_logits + batch_size * n_vocab);

        // 如果是chunk的第一个batch，恢复原始token
        if (j == 0) {
            tokens[batch_start] = token_org;
        }
    }

    // 记录结束时间
    const auto t_end = std::chrono::high_resolution_clock::now();
        // 如果是第一次循环
        if (i == 0) {
            // 计算总共花费的时间
            const float t_total = std::chrono::duration<float>(t_end - t_start).count();
            // 输出每次循环花费的时间和预计剩余时间
            fprintf(stderr, "%s: %.2f seconds per pass - ETA ", __func__, t_total);
            // 计算总共剩余的秒数
            int total_seconds = (int)(t_total * n_chunk);
            // 如果总共剩余的秒数超过1小时，输出剩余小时数并更新剩余秒数
            if (total_seconds >= 60*60) {
                fprintf(stderr, "%d hours ", total_seconds / (60*60));
                total_seconds = total_seconds % (60*60);
            }
            // 输出剩余分钟数
            fprintf(stderr, "%.2f minutes\n", total_seconds / 60.0);
        }

        // 循环遍历每个上下文
        for (int j = n_ctx - params.ppl_stride - 1; j < n_ctx - 1; ++j) {

            // 计算下一个标记的概率，给定前面的标记
            const std::vector<float> tok_logits(
                logits.begin() + (j + 0) * n_vocab,
                logits.begin() + (j + 1) * n_vocab);
        }
        // 计算softmax概率分布中指定位置的概率值
        const float prob = softmax(tok_logits)[tokens[start + j + 1]];
        // 记录当前位置的logit值
        logit_history[start + j + 1] = tok_logits[tokens[start + j + 1]];
        // 记录当前位置的概率值
        prob_history[start + j + 1]  = prob;

        // 计算负对数似然，并累加到nll中
        nll += -std::log(prob);
        // 计数器加一
        ++count;
    }
    // 计算困惑度，即e的平均负对数似然
    if (params.ppl_output_type == 0) {
        // 输出困惑度值
        printf("[%d]%.4lf,", i + 1, std::exp(nll / count));
    } else {
        // 输出困惑度值和对应的位置
        printf("%8d  %.4lf\n", i*params.ppl_stride, std::exp(nll / count));
    }
    // 刷新标准输出流
    fflush(stdout);
}
// 输出换行符
printf("\n");

// 返回包含tokens、困惑度值、logit历史和概率历史的元组
return {tokens, std::exp(nll / count), logit_history, prob_history};
// 计算困惑度的函数，接受一个 llama_context 指针和一个 gpt_params 结构体作为参数
static results_perplexity perplexity(llama_context * ctx, const gpt_params & params) {
    // 如果参数中的 ppl_stride 大于 0，则调用 perplexity_v2 函数
    if (params.ppl_stride > 0) {
        return perplexity_v2(ctx, params);
    }

    // 下载链接：https://s3.amazonaws.com/research.metamind.io/wikitext/wikitext-2-raw-v1.zip?ref=salesforce-research
    // 运行 `./perplexity -m models/7B/ggml-model-q4_0.bin -f wiki.test.raw`
    // 输出：`perplexity: 13.5106 [114/114]`
    // 在评估之前，每个块都会添加 BOS 标记

    // 检查模型的词汇类型是否为 SPM
    const bool is_spm = llama_vocab_type(llama_get_model(ctx)) == LLAMA_VOCAB_TYPE_SPM;
    // 如果是 SPM 模型，则需要添加 BOS 标记
    const bool add_bos = is_spm;
    // 获取上下文的大小
    const int n_ctx = llama_n_ctx(ctx);

    // 记录开始时间
    auto tim1 = std::chrono::high_resolution_clock::now();
    // 打印信息到标准错误输出，提示正在对输入进行标记化处理
    fprintf(stderr, "%s: tokenizing the input ..\n", __func__);

    // 对输入进行标记化处理，返回标记化后的结果
    std::vector<llama_token> tokens = ::llama_tokenize(ctx, params.prompt, add_bos);

    // 记录结束时间
    auto tim2 = std::chrono::high_resolution_clock::now();
    // 输出tokenization所花费的时间
    fprintf(stderr, "%s: tokenization took %g ms\n",__func__,1e-3*std::chrono::duration_cast<std::chrono::microseconds>(tim2-tim1).count());

    // 检查tokens的数量是否足够，如果不够则输出错误信息并返回空结果
    if (int(tokens.size()) < 2*n_ctx) {
        fprintf(stderr, "%s: you need at least %d tokens to evaluate perplexity with a context of %d\n",__func__,2*n_ctx,
                n_ctx);
        fprintf(stderr, "%s: the data file you provided tokenizes to only %zu tokens\n",__func__,tokens.size());
        return {std::move(tokens), 0., {}, {}};
    }

    // 初始化logit_history和prob_history向量，并设置其大小为tokens的大小
    std::vector<float> logit_history;
    logit_history.resize(tokens.size());

    std::vector<float> prob_history;
    prob_history.resize(tokens.size());

    // 计算n_chunk_max，即tokens数量除以n_ctx
    const int n_chunk_max = tokens.size() / n_ctx;

    // 计算n_chunk，取params.n_chunks和n_chunk_max中较小的值
    const int n_chunk = params.n_chunks < 0 ? n_chunk_max : std::min(params.n_chunks, n_chunk_max);
    
    // 获取词汇表大小和批处理大小
    const int n_vocab = llama_n_vocab(llama_get_model(ctx));
    const int n_batch = params.n_batch;
    // 初始化计数器
    int count = 0;
    // 初始化两个浮点数变量
    double nll = 0.0;
    double nll2 = 0.0;

    // 打印计算信息到标准错误流
    fprintf(stderr, "%s: calculating perplexity over %d chunks, batch_size=%d\n", __func__, n_chunk, n_batch);

    // 创建线程向量，数量为可用的硬件线程数减一
    std::vector<std::thread> workers(std::thread::hardware_concurrency() - 1);

    // 循环处理每个数据块
    for (int i = 0; i < n_chunk; ++i) {
        // 计算当前数据块的起始和结束位置
        const int start =     i * n_ctx;
        const int end   = start + n_ctx;

        // 计算当前数据块需要的批次数
        const int num_batches = (n_ctx + n_batch - 1) / n_batch;

        // 创建存储logits的向量
        std::vector<float> logits;

        // 记录开始时间
        const auto t_start = std::chrono::high_resolution_clock::now();

        // 清空KV缓存
// 清除 llama_kv_cache 中的缓存
llama_kv_cache_clear(ctx);

// 遍历每个批次
for (int j = 0; j < num_batches; ++j) {
    // 计算当前批次的起始位置和大小
    const int batch_start = start + j * n_batch;
    const int batch_size  = std::min(end - batch_start, n_batch);

    // 保存原始的 token，并在评估后恢复它
    const auto token_org = tokens[batch_start];

    // 如果需要添加 BOS token，并且是每个 chunk 的第一个批次
    if (add_bos && j == 0) {
        // 在每个 chunk 的第一个批次添加 BOS token
        tokens[batch_start] = llama_token_bos(llama_get_model(ctx));
    }

    // 调用 llama_decode 进行解码
    if (llama_decode(ctx, llama_batch_get_one(tokens.data() + batch_start, batch_size, j * n_batch, 0))) {
        // 如果解码失败，打印错误信息并返回结果
        fprintf(stderr, "%s : failed to eval\n", __func__);
        return {tokens, -1, logit_history, prob_history};
    }

    // 恢复原始的 token，以防它被设置为 BOS
}
// 将token_org赋值给tokens[batch_start]
tokens[batch_start] = token_org;

// 获取当前批次的logits，并将其添加到logits向量末尾
const auto * batch_logits = llama_get_logits(ctx);
logits.insert(logits.end(), batch_logits, batch_logits + batch_size * n_vocab);
}

// 计算时间间隔
const auto t_end = std::chrono::high_resolution_clock::now();

// 如果是第一次循环
if (i == 0) {
    // 计算总时间
    const float t_total = std::chrono::duration<float>(t_end - t_start).count();
    // 打印函数名和每次循环的平均时间
    fprintf(stderr, "%s: %.2f seconds per pass - ETA ", __func__, t_total);
    // 计算总共需要的时间，并打印预计剩余时间
    int total_seconds = (int)(t_total * n_chunk);
    if (total_seconds >= 60*60) {
        fprintf(stderr, "%d hours ", total_seconds / (60*60));
        total_seconds = total_seconds % (60*60);
    }
    fprintf(stderr, "%.2f minutes\n", total_seconds / 60.0);
}

// 获取上下文窗口（params.n_ctx）中所有token的logits
// 从上面的llama_eval中获取。现在，根据https://huggingface.co/docs/transformers/perplexity，
// 计算窗口的后半部分的困惑度（这样模型始终有一些上下文来预测标记）。
//
// 我们依赖于前向传播中的注意力只关注先前的标记，因此返回的每个标记的logits是模型在那一点上的准确表示。
//
// 例如，我们有一个512的上下文窗口，我们将计算最后256个标记的困惑度。然后，我们将输入分割成上下文窗口大小的块来处理整个提示。
const int first = n_ctx/2;
process_logits(n_vocab, logits.data() + first*n_vocab, tokens.data() + start + first, n_ctx - 1 - first,
               workers, nll, nll2, logit_history.data() + start + first, prob_history.data() + start + first);
count += n_ctx - first - 1;

// 困惑度是e^(平均负对数似然)
if (params.ppl_output_type == 0) {
    printf("[%d]%.4lf,", i + 1, std::exp(nll / count));
} else {
    // 计算平均值
    double av = nll/count;
    // 计算方差
    double av2 = nll2/count - av*av;
    // 如果方差大于0，计算标准差
    if (av2 > 0) av2 = sqrt(av2/(count-1));
    // 打印结果
    printf("%8d  %.4lf  %4lf  %4lf\n", i*n_ctx, std::exp(nll / count), av, av2);
    // 刷新标准输出缓冲区
    fflush(stdout);
    // 计算平均值和方差的平方
    nll2 /= count;
    nll /= count;
    // 计算困惑度
    const double ppl = exp(nll);
    nll2 -= nll * nll;
    // 如果方差的平方大于0，计算标准差
    if (nll2 > 0) {
        nll2 = sqrt(nll2/(count-1));
        // 打印最终估计结果
        printf("Final estimate: PPL = %.4lf +/- %.5lf\n", ppl, nll2*ppl);
    } else {
        // 打印异常情况
        printf("Unexpected negative standard deviation of log(prob)\n");
    }
// 返回包含 tokens, ppl, logit_history, prob_history 的字典
    return {tokens, ppl, logit_history, prob_history};
}

// 对 tokens 进行评估并返回结果
static std::vector<float> hellaswag_evaluate_tokens(
    llama_context * ctx, std::vector<int> & tokens, int n_past, int n_batch, int n_vocab
) {
    // 创建结果向量并预留空间
    std::vector<float> result;
    result.reserve(tokens.size() * n_vocab);
    // 计算需要分块的数量
    size_t n_chunk = (tokens.size() + n_batch - 1)/n_batch;
    // 遍历每个分块
    for (size_t i_chunk = 0; i_chunk < n_chunk; ++i_chunk) {
        // 计算当前分块的 tokens 数量
        size_t n_tokens = tokens.size() - i_chunk * n_batch;
        n_tokens = std::min(n_tokens, size_t(n_batch));
        // 调用 llama_decode 对 tokens 进行评估
        if (llama_decode(ctx, llama_batch_get_one(tokens.data() + i_chunk * n_batch, n_tokens, n_past, 0))) {
            // 如果评估失败，打印错误信息并返回空向量
            fprintf(stderr, "%s : failed to eval\n", __func__);
            return {};
        }
        // 获取评估结果 logits，并插入到结果向量中
        const auto logits = llama_get_logits(ctx);
        result.insert(result.end(), logits, logits + n_tokens * n_vocab);
        n_past += n_tokens;
    }
    return result;
}

static void hellaswag_score(llama_context * ctx, const gpt_params & params) {
    // 计算来自提示的 hellaswag 分数（acc_norm）
    //
    // 从 HellaSwag 验证数据集提取的数据（MIT 许可证）https://github.com/rowanz/hellaswag/blob/master/data/hellaswag_val.jsonl
    // 所有使用的数据字段都经过预处理，如 https://github.com/EleutherAI/lm-evaluation-harness/blob/df3da98c5405deafd519c2ddca52bb7c3fe36bef/lm_eval/tasks/hellaswag.py#L62-L68
    //
    // 所有 10042 个任务都应被提取，以保持结果与其他实现的标准化。
    //
    // 数据文件布局：
    // ['??'] 表示 json 字段
    // 每个任务 6 行：
    // ['activity_label'] + ": " +['ctx']  - 查询的第一部分，上下文
    // ['label'] - 最佳常识结尾的索引，也就是黄金结尾
    // ['endings'][0] - 添加到查询的第一部分的结尾
    // ['endings'][1]
    // 创建一个存储字符串的向量
    std::vector<std::string> prompt_lines;
    // 从参数中的prompt字符串创建一个字符串流
    std::istringstream strstream(params.prompt);
    // 用于存储每行字符串的变量
    std::string line;

    // 逐行读取字符串流中的内容，并存储到prompt_lines向量中
    while (std::getline(strstream,line,'\n')) {
        prompt_lines.push_back(line);
    }

    // 检查prompt_lines中的行数是否是6的倍数，如果不是则打印错误信息并返回
    if( prompt_lines.size() % 6 != 0) {
        fprintf(stderr, "%s : number of lines in prompt not a multiple of 6.\n", __func__);
        return;
    }

    // 计算任务数量
    size_t hs_task_count = prompt_lines.size()/6;
    // 打印加载任务数量的信息
    fprintf(stderr, "%s : loaded %zu tasks from prompt.\n", __func__, hs_task_count);

    // 检查当前模型是否为SPM类型
    const bool is_spm = llama_vocab_type(llama_get_model(ctx)) == LLAMA_VOCAB_TYPE_SPM;
    // 打印 is_spm 变量的值
    fprintf(stderr, "================================= is_spm = %d\n", is_spm);

    // 根据 is_spm 变量的值来确定是否需要添加 bos
    const bool add_bos = is_spm;

    // 计算分数时要使用的任务数量
    if ( params.hellaswag_tasks < hs_task_count  ) {
        hs_task_count = params.hellaswag_tasks;
    }

    // 任务应该被随机化，以便分数能够快速稳定下来
    bool randomize_tasks = true;

    // 如果计算的任务数量足够多，随机种子不应该影响最终结果，因此暂时保持硬编码
    std::mt19937 rng(1);

    // hellaswag 任务的数据持有者
    struct hs_data_t {
        std::string context;
        size_t gold_ending_idx;
// 定义了一个包含4个字符串的数组，用于存储结尾信息
std::string ending[4];
// 定义了一个包含4个整数的数组，用于存储结尾对数计数
size_t ending_logprob_count[4];
// 定义了一个包含4个双精度浮点数的数组，用于存储结尾对数概率
double ending_logprob[4];
};

// 打印选择的任务数量和方式
fprintf(stderr, "%s : selecting %zu %s tasks.\n", __func__, hs_task_count, (randomize_tasks?"randomized":"the first")  );

// 选择并读取来自提示行的数据
hs_data_t *hs_data = new hs_data_t[hs_task_count];
for (size_t i=0; i < hs_task_count; i++) {
    size_t idx = i;

    // 如果需要随机选择任务，则从提示行中选择一个随机示例
    if (randomize_tasks) {
        std::uniform_int_distribution<size_t> dist(0, prompt_lines.size()/6-1 ) ;
        idx = dist(rng);
    }

    // 将提示行中的上下文存储到hs_data结构体中
    hs_data[i].context = prompt_lines[idx*6];
    // 将提示行中的正确结尾索引转换为整数并存储到hs_data结构体中
    hs_data[i].gold_ending_idx = std::stoi( prompt_lines[idx*6+1] );
        // 遍历数组，将指定位置的数据复制到结构体数组中
        for (size_t j=0; j < 4; j++) {
            hs_data[i].ending[j] = prompt_lines[idx*6+2+j];
        }

        // 如果需要随机化任务顺序，从prompt_lines中删除选定的随机示例
        if (randomize_tasks) {
            prompt_lines.erase( std::next(prompt_lines.begin(),idx*6)  , std::next(prompt_lines.begin(),idx*6+6) );
        }
    }

    // 输出计算hellaswag分数的提示信息
    fprintf(stderr, "%s : calculating hellaswag score over selected tasks.\n", __func__);
    // 输出表头
    printf("\ntask\tacc_norm\n");

    // 初始化准确率为0
    double acc = 0.0f;
    // 获取词汇表大小和上下文大小
    const int n_vocab = llama_n_vocab(llama_get_model(ctx));
    const int n_ctx = llama_n_ctx(ctx);

    // 初始化包含4个子数组的二维数组
    std::vector<std::vector<int>> ending_tokens(4);

    // 初始化包含n_vocab个元素的浮点数数组
    std::vector<float> tok_logits(n_vocab);
// 遍历任务索引，从0到任务数量-1
for (size_t task_idx = 0; task_idx < hs_task_count; task_idx++) {
    // 对上下文进行分词，以计算标记数量
    std::vector<int> context_embd = ::llama_tokenize(ctx, hs_data[task_idx].context, add_bos);
    // 获取上下文的标记数量
    size_t context_size = context_embd.size();

    // 遍历4次，对每个结尾进行处理
    for (int i = 0; i < 4; ++i) {
        // 对上下文和每个结尾进行分词
        ending_tokens[i] = ::llama_tokenize(ctx, hs_data[task_idx].context + " " + hs_data[task_idx].ending[i], add_bos);
        // 对比上下文和结尾的标记，查找不同之处
        for (int k = 0; k < int(context_size); ++k) {
            if (ending_tokens[i][k] != context_embd[k]) {
                // 输出错误信息，指出任务中第i个结尾与上下文在位置k处不同
                fprintf(stderr, "Oops: ending %d of task %d differs from context at position %d\n",i,int(task_idx),k);
                // 中断当前循环
                break;
            }
        }
    }

    // 处理第一个结尾
    // 在这种情况下，在评估时包括上下文
    //auto query_embd = ::llama_tokenize(ctx, hs_data[task_idx].context + hs_data[task_idx].ending[0], add_bos);
    // 将第一个结尾的标记赋值给查询标记
    auto query_embd = ending_tokens[0];
}
        // 获取查询嵌入的大小
        auto query_size = query_embd.size();

        // 如果查询的大小超过了上下文窗口的大小，则停止执行并打印错误信息
        if (query_size > (size_t)n_ctx) {
            fprintf(stderr, "%s : number of tokens in query %zu > n_ctxl\n", __func__, query_size);
            return;
        }

        // 通过至少评估32个标记来加快小规模评估的速度
        if (query_size < 32) {
            // 调整查询嵌入的大小为32
            query_embd.resize(32);
        }

        // 清空KV缓存
        llama_kv_cache_clear(ctx);

        // 评估标记并获取logits
        auto logits = hellaswag_evaluate_tokens(ctx, query_embd, 0, params.n_batch, n_vocab);
        // 如果logits为空，则打印错误信息并返回
        if (logits.empty()) {
            fprintf(stderr, "%s : failed to eval\n", __func__);
            return;
        }

        // 将logits中的数据复制到tok_logits中，用于计算token的logits
        std::memcpy(tok_logits.data(), logits.data() + (context_size-1)*n_vocab, n_vocab*sizeof(float));
        // 计算第一个token的概率分布
        const auto first_probs = softmax(tok_logits);

        // 将第一个token的log概率存储到数据结构中
        hs_data[task_idx].ending_logprob_count[0] = 1;
        hs_data[task_idx].ending_logprob[0] = std::log(first_probs[query_embd[context_size]]);

        // 计算结束token的log概率
        for (size_t j = context_size; j < query_size - 1; j++) {
            // 将logits中的数据复制到tok_logits中，用于计算token的logits
            std::memcpy(tok_logits.data(), logits.data() + j*n_vocab, n_vocab*sizeof(float));
            // 计算token的概率分布
            const float prob = softmax(tok_logits)[query_embd[j + 1]];

            // 将token的log概率累加到数据结构中
            hs_data[task_idx].ending_logprob[0] += std::log(prob);
            hs_data[task_idx].ending_logprob_count[0]++;
        }

        // 计算token的平均log概率用于acc_norm
        // 将任务索引对应的结束对数的对数概率除以对数概率计数
        hs_data[task_idx].ending_logprob[0] /= hs_data[task_idx].ending_logprob_count[0];

        // 处理剩余的结束对数
        // 对于这些，我们使用具有 n_past = context_size 的裸结束对数
        //
        for (size_t ending_idx = 1; ending_idx < 4; ending_idx++) {

            // 对查询进行标记化
            query_embd.resize(ending_tokens[ending_idx].size() - context_size);
            std::memcpy(query_embd.data(), ending_tokens[ending_idx].data() + context_size, query_embd.size()*sizeof(int));
            query_size = query_embd.size();

            // 如果查询无法适应上下文窗口，则停止
            if (context_size + query_size > (size_t)n_ctx) {
                fprintf(stderr, "%s : number of tokens in query %zu > n_ctxl\n", __func__, query_size);
                return;
            }

            // 通过至少评估 32 个标记来加快小规模评估的速度
            // 不，调整大小为 32 实际上略慢一些（至少在 CUDA 上）
            // 如果查询大小小于32，则调整查询嵌入的大小为32
            //if (query_size < 32) {
            //    query_embd.resize(32);
            //}

            // 评估查询
            // 使用上下文、查询嵌入、上下文大小、批量大小和词汇量来计算logits
            logits = hellaswag_evaluate_tokens(ctx, query_embd, context_size, params.n_batch, n_vocab);
            // 如果logits为空，则打印错误信息并返回
            if (logits.empty()) {
                fprintf(stderr, "%s : failed to eval\n", __func__);
                return;
            }

            // 设置结束的logprob计数为1
            hs_data[task_idx].ending_logprob_count[ending_idx] = 1;
            // 计算第一个概率的log值，并存储在ending_logprob中
            hs_data[task_idx].ending_logprob[ending_idx] = std::log(first_probs[query_embd[0]]);

            // 计算结束的logprobs
            for (size_t j = 0; j < query_size - 1; j++) {
                // 将logits中的数据复制到tok_logits中
                std::memcpy(tok_logits.data(), logits.data() + j*n_vocab, n_vocab*sizeof(float));
                // 计算softmax后的概率值，并存储在prob中
                const float prob = softmax(tok_logits)[query_embd[j + 1]];
// 将概率的对数加到对应任务和结束索引的结束日志概率中
hs_data[task_idx].ending_logprob[ending_idx] += std::log(prob);
// 增加对应任务和结束索引的结束日志概率计数
hs_data[task_idx].ending_logprob_count[ending_idx]++;
}

// 计算 acc_norm 的平均标记对数概率
hs_data[task_idx].ending_logprob[ending_idx] /= hs_data[task_idx].ending_logprob_count[ending_idx];


//            printf("task %lu, ending %lu, whole_len %lu, context_len %lu, ending_logprob_count %lu, ending_logprob %.4f\n",
//                task_idx,ending_idx,whole_size,context_size, hs_data[task_idx].ending_logprob_count[ending_idx], hs_data[task_idx].ending_logprob[ending_idx] );
}

// 找到具有最大对数概率的结束
size_t ending_logprob_max_idx = 0;
double ending_logprob_max_val = hs_data[task_idx].ending_logprob[0];
for (size_t j = 1; j < 4; j++) {
    if (hs_data[task_idx].ending_logprob[j] > ending_logprob_max_val) {
        ending_logprob_max_idx = j;
        ending_logprob_max_val =  hs_data[task_idx].ending_logprob[j];
    }
        }

//        printf("max logprob ending idx %lu, gold ending idx %lu\n", ending_logprob_max_idx, hs_data[task_idx].gold_ending_idx);

        // 如果最大对数概率的结束索引与黄金结束索引相同，则增加一个准确度点
        if (ending_logprob_max_idx == hs_data[task_idx].gold_ending_idx) {
            acc += 1.0;
        }

        // 打印累积准确度的平均值乘以 100
        printf("%zu\t%.8lf\n",task_idx+1, acc/double(task_idx+1)*100.0);
        fflush(stdout);
    }

    delete [] hs_data;

    printf("\n");
}

int main(int argc, char ** argv) {
```

在这个示例中，我添加了对每个语句的注释，解释了它们的作用。
    # 声明一个名为params的gpt_params结构体变量
    gpt_params params;

    # 设置params结构体变量中的n_batch字段为512
    params.n_batch = 512;

    # 调用gpt_params_parse函数解析命令行参数，如果解析失败则返回1
    if (!gpt_params_parse(argc, argv, params)) {
        return 1;
    }

    # 设置params结构体变量中的logits_all字段为true
    params.logits_all = true;

    # 将params结构体变量中的n_batch字段和n_ctx字段中较小的值赋给n_batch字段
    params.n_batch = std::min(params.n_batch, params.n_ctx);

    # 如果params结构体变量中的ppl_stride字段大于0，则调整n_ctx字段的值
    if (params.ppl_stride > 0) {
        fprintf(stderr, "Will perform strided perplexity calculation -> adjusting context size from %d to %d\n",
                params.n_ctx, params.n_ctx + params.ppl_stride/2);
        params.n_ctx += params.ppl_stride/2;
    }

    # 打印构建信息
    print_build_info();

    # 如果params结构体变量中的seed字段为LLAMA_DEFAULT_SEED，则将其设置为当前时间
    if (params.seed == LLAMA_DEFAULT_SEED) {
        params.seed = time(NULL);
    }

    // 打印种子值
    fprintf(stderr, "%s: seed  = %u\n", __func__, params.seed);

    // 使用种子值初始化随机数生成器
    std::mt19937 rng(params.seed);
    // 如果需要随机提示，则生成随机提示
    if (params.random_prompt) {
        params.prompt = gpt_random_prompt(rng);
    }

    // 初始化 LLAMA 后端
    llama_backend_init(params.numa);

    llama_model * model;
    llama_context * ctx;

    // 加载模型并应用 lora 适配器（如果有的话）
    std::tie(model, ctx) = llama_init_from_gpt_params(params);
    // 如果无法加载模型，则打印错误信息并返回 1
    if (model == NULL) {
        fprintf(stderr, "%s: error: unable to load model\n", __func__);
        return 1;
    }
// 获取模型的训练上下文长度
const int n_ctx_train = llama_n_ctx_train(model);
// 如果指定的上下文长度大于模型的训练上下文长度，输出警告信息
if (params.n_ctx > n_ctx_train) {
    fprintf(stderr, "%s: warning: model was trained on only %d context tokens (%d specified)\n",
            __func__, n_ctx_train, params.n_ctx);
}

// 打印系统信息
{
    fprintf(stderr, "\n");
    fprintf(stderr, "%s\n", get_system_info(params).c_str());
}

// 定义结果变量
struct results_perplexity results;
// 如果参数中指定了 hellaswag 标志，则调用 hellaswag_score 函数
if (params.hellaswag) {
    hellaswag_score(ctx, params);
} 
// 否则调用 perplexity 函数，并将结果赋给 results
else {
    results = perplexity(ctx, params);
}
    # 打印程序执行时间
    llama_print_timings(ctx);
    # 写入日志文件
    write_logfile(ctx, params, model, results);

    # 释放上下文资源
    llama_free(ctx);
    # 释放模型资源
    llama_free_model(model);

    # 释放后端资源
    llama_backend_free();

    # 返回成功状态码
    return 0;
}
```