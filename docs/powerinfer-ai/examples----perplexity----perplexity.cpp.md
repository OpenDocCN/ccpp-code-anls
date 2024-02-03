# `PowerInfer\examples\perplexity\perplexity.cpp`

```cpp
#include "common.h"  // 引入 common.h 头文件
#include "llama.h"    // 引入 llama.h 头文件

#include <cmath>      // 引入数学函数库
#include <cstdio>     // 引入标准输入输出库
#include <cstring>    // 引入字符串处理库
#include <ctime>      // 引入时间处理库
#include <sstream>    // 引入字符串流库
#include <thread>     // 引入多线程库
#include <mutex>      // 引入互斥锁库
#include <vector>     // 引入向量库

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // 禁止特定警告
#endif

struct results_perplexity {
    std::vector<llama_token> tokens;  // 定义包含 llama_token 类型的向量 tokens
    double                   ppl_value;  // 定义 double 类型的 ppl_value
    std::vector<float>       logits;  // 定义包含 float 类型的向量 logits
    std::vector<float>       probs;  // 定义包含 float 类型的向量 probs
};

struct results_log_softmax {
    double log_softmax;  // 定义 double 类型的 log_softmax
    float  logit;  // 定义 float 类型的 logit
    float  prob;  // 定义 float 类型的 prob
};

static void write_logfile(
    const llama_context * ctx, const gpt_params & params, const llama_model * model,
    const struct results_perplexity & results
) {
    if (params.logdir.empty()) {  // 如果 logdir 为空，则返回
        return;
    }

    if (params.hellaswag) {  // 如果 params.hellaswag 为真，则输出警告信息并返回
        fprintf(stderr, "%s: warning: logging results is not implemented for HellaSwag. No files will be written.\n", __func__);
        return;
    }

    const std::string timestamp = get_sortable_timestamp();  // 获取可排序的时间戳

    const bool success = create_directory_with_parents(params.logdir);  // 创建 logdir 及其父目录
    if (!success) {  // 如果创建失败，则输出警告信息并返回
        fprintf(stderr, "%s: warning: failed to create logdir %s, cannot write logfile\n",
                __func__, params.logdir.c_str());
        return;
    }

    const std::string logfile_path = params.logdir + timestamp + ".yml";  // 构建日志文件路径
    FILE * logfile = fopen(logfile_path.c_str(), "w");  // 打开日志文件

    if (logfile == NULL) {  // 如果打开失败，则输出错误信息并返回
        fprintf(stderr, "%s: failed to open logfile %s\n", __func__, logfile_path.c_str());
        return;
    }

    fprintf(logfile, "binary: main\n");  // 向日志文件写入内容
    char model_desc[128];  // 定义长度为 128 的字符数组 model_desc
    llama_model_desc(model, model_desc, sizeof(model_desc));  // 获取模型描述信息
    dump_non_result_info_yaml(logfile, params, ctx, timestamp, results.tokens, model_desc);  // 将非结果信息转储为 YAML 格式

    fprintf(logfile, "\n");  // 向日志文件写入换行符
    fprintf(logfile, "######################\n");  // 向日志文件写入内容
    fprintf(logfile, "# Perplexity Results #\n");  // 向日志文件写入内容
    fprintf(logfile, "######################\n");  // 向日志文件写入内容
    fprintf(logfile, "\n");  // 向日志文件写入换行符
    # 将 results.logits 中的浮点数向量以 YAML 格式写入到 logfile 中
    dump_vector_float_yaml(logfile, "logits", results.logits);
    # 将 results.ppl_value 的值以指定格式写入到 logfile 中
    fprintf(logfile, "ppl_value: %f\n", results.ppl_value);
    # 将 results.probs 中的浮点数向量以 YAML 格式写入到 logfile 中
    dump_vector_float_yaml(logfile, "probs", results.probs);
    # 将上下文 ctx 的时间信息以 YAML 格式写入到 logfile 中
    llama_dump_timing_info_yaml(logfile, ctx);
    # 关闭 logfile 文件
    fclose(logfile);
// 计算给定logits的softmax值并返回概率向量
static std::vector<float> softmax(const std::vector<float>& logits) {
    // 创建一个与logits相同大小的概率向量
    std::vector<float> probs(logits.size());
    // 初始化最大logit值为logits的第一个元素
    float max_logit = logits[0];
    // 遍历logits，找到最大的logit值
    for (float v : logits) {
        max_logit = std::max(max_logit, v);
    }
    // 初始化指数和为0
    double sum_exp = 0.0;
    // 遍历logits，计算softmax值并累加指数和
    for (size_t i = 0; i < logits.size(); i++) {
        // 为了数值稳定性，从当前logit值中减去最大logit值
        const float logit = logits[i] - max_logit;
        const float exp_logit = expf(logit);
        sum_exp += exp_logit;
        probs[i] = exp_logit;
    }
    // 将概率向量中的每个值除以指数和，得到最终的softmax概率
    for (size_t i = 0; i < probs.size(); i++) {
        probs[i] /= sum_exp;
    }
    return probs;
}

// 计算logits的log_softmax值并返回结果结构体
static results_log_softmax log_softmax(int n_vocab, const float * logits, int tok) {
    // 初始化最大logit值为logits的第一个元素
    float max_logit = logits[0];
    // 遍历logits，找到最大的logit值
    for (int i = 1; i < n_vocab; ++i) {
        max_logit = std::max(max_logit, logits[i]);
    }
    // 初始化指数和为0
    double sum_exp = 0.0;
    // 遍历logits，计算log_softmax的指数和
    for (int i = 0; i < n_vocab; ++i) {
        sum_exp += expf(logits[i] - max_logit);
    }
    // 返回log_softmax结果结构体
    return {logits[tok] - max_logit - log(sum_exp), logits[tok], expf(logits[tok] - max_logit) / (float) sum_exp};
}

// 处理logits的函数
static void process_logits(
    int n_vocab, const float * logits, const int * tokens, int n_token, std::vector<std::thread> & workers,
    double & nll, double & nll2, float * logit_history, float * prob_history
) {
    // 创建互斥锁
    std::mutex mutex;
    // 初始化计数器为0
    int counter = 0;
    # 定义一个 lambda 函数 compute，该函数使用了引用传递的方式获取了多个变量
    auto compute = [&mutex, &counter, &nll, &nll2, logit_history, prob_history, n_vocab, logits, tokens, n_token] () {
        # 定义本地变量 local_nll 和 local_nll2，并初始化为 0
        double local_nll  = 0;
        double local_nll2 = 0;
        # 进入无限循环
        while (true) {
            # 使用互斥锁 lock 对 mutex 进行加锁
            std::unique_lock<std::mutex> lock(mutex);
            # 从 counter 中获取一个值并自增
            int i = counter++;
            # 如果 i 大于等于 n_token，则将 local_nll 和 local_nll2 的值累加到 nll 和 nll2 上，并跳出循环
            if (i >= n_token) {
                nll += local_nll; nll2 += local_nll2;
                break;
            }
            # 解锁互斥锁
            lock.unlock();
            # 调用 log_softmax 函数计算结果，并将结果存储到 results 中
            const results_log_softmax results = log_softmax(n_vocab, logits + i*n_vocab, tokens[i+1]);
            # 计算 v 的值
            const double v = -results.log_softmax;
            # 将 v 累加到 local_nll 和 local_nll2 上
            local_nll += v;
            local_nll2 += v*v;

            # 将 results.logit 和 results.prob 分别存储到 logit_history[i] 和 prob_history[i] 中
            logit_history[i] = results.logit;
            prob_history[i]  = results.prob;
        }
    };
    # 遍历 workers 数组，为每个元素创建一个新的线程，并执行 compute 函数
    for (auto & w : workers) {
        w = std::thread(compute);
    }
    # 调用 compute 函数
    compute();
    # 遍历 workers 数组，等待每个线程执行完毕
    for (auto & w : workers) {
        w.join();
    }
}

static results_perplexity perplexity_v2(llama_context * ctx, const gpt_params & params) {
    // 下载数据集：https://s3.amazonaws.com/research.metamind.io/wikitext/wikitext-2-raw-v1.zip?ref=salesforce-research
    // 运行 `./perplexity -m models/7B/ggml-model-q4_0.bin -f wiki.test.raw`
    // 输出：`perplexity: 13.5106 [114/114]`
    // 在评估之前，每个块都会添加 BOS 标记

    // 检查模型词汇类型是否为 SPM
    const bool is_spm = llama_vocab_type(llama_get_model(ctx)) == LLAMA_VOCAB_TYPE_SPM;
    // 如果是 SPM，则添加 BOS 标记
    const bool add_bos = is_spm;

    // 打印信息到标准错误流
    fprintf(stderr, "%s: tokenizing the input ..\n", __func__);

    // 对输入进行标记化处理
    std::vector<llama_token> tokens = ::llama_tokenize(ctx, params.prompt, add_bos);

    // 获取上下文大小
    const int n_ctx = llama_n_ctx(ctx);

    // 如果标记数量小于两倍的上下文大小，则无法评估困惑度
    if (int(tokens.size()) < 2*n_ctx) {
        fprintf(stderr, "%s: you need at least %d tokens to evaluate perplexity with a context of %d\n",__func__,2*n_ctx,
                n_ctx);
        fprintf(stderr, "%s: the data file you provided tokenizes to only %zu tokens\n",__func__,tokens.size());
        return {std::move(tokens), 0., {}, {}};
    }

    // 初始化记录 logit 和概率的历史向量
    std::vector<float> logit_history;
    std::vector<float> prob_history;

    // 调整历史向量的大小
    logit_history.resize(tokens.size());
    prob_history.resize(tokens.size());

    // 如果步长小于等于 0，则无法计算困惑度
    if (params.ppl_stride <= 0) {
        fprintf(stderr, "%s: stride is %d but must be greater than zero!\n",__func__,params.ppl_stride);
        return {tokens, -1, logit_history, prob_history};
    }

    // 计算块大小
    const int calc_chunk = n_ctx;

    // 打印信息到标准错误流
    fprintf(stderr, "%s: have %zu tokens. Calculation chunk = %d\n", __func__, tokens.size(), calc_chunk);

    // 如果标记数量小于等于计算块大小，则无法计算困惑度
    if (int(tokens.size()) <= calc_chunk) {
        fprintf(stderr, "%s: there are only %zu tokens, this is not enough for a context size of %d and stride %d\n",__func__,
                tokens.size(), n_ctx, params.ppl_stride);
        return {tokens, -1, logit_history, prob_history};
    }

    // 计算最大块数量
    const int n_chunk_max = (tokens.size() - calc_chunk + params.ppl_stride - 1)  / params.ppl_stride;
    # 计算要处理的块数，如果参数中的块数小于0，则取最大块数，否则取参数中的块数和最大块数的最小值
    const int n_chunk = params.n_chunks < 0 ? n_chunk_max : std::min(params.n_chunks, n_chunk_max);
    # 获取词汇表的大小
    const int n_vocab = llama_n_vocab(llama_get_model(ctx));
    # 获取批处理大小
    const int n_batch = params.n_batch;

    # 初始化计数器和负对数似然
    int count = 0;
    double nll = 0.0;

    # 打印计算困惑度的信息，包括块数和批处理大小
    fprintf(stderr, "%s: calculating perplexity over %d chunks, batch_size=%d\n", __func__, n_chunk, n_batch);

    # 返回结果
    }
    # 打印换行符
    printf("\n");

    # 返回结果，包括tokens, perplexity, logit_history, prob_history
    return {tokens, std::exp(nll / count), logit_history, prob_history};
}

static results_perplexity perplexity(llama_context * ctx, const gpt_params & params) {
    // 如果参数中的 PPL 步长大于 0，则调用 perplexity_v2 函数
    if (params.ppl_stride > 0) {
        return perplexity_v2(ctx, params);
    }

    // 下载：https://s3.amazonaws.com/research.metamind.io/wikitext/wikitext-2-raw-v1.zip?ref=salesforce-research
    // 运行 `./perplexity -m models/7B/ggml-model-q4_0.bin -f wiki.test.raw`
    // 输出：`perplexity: 13.5106 [114/114]`
    // 在评估之前，每个块都会添加 BOS 标记

    // 检查模型词汇类型是否为 SPM
    const bool is_spm = llama_vocab_type(llama_get_model(ctx)) == LLAMA_VOCAB_TYPE_SPM;
    // 如果是 SPM 模型，则添加 BOS 标记
    const bool add_bos = is_spm;
    // 获取上下文大小
    const int n_ctx = llama_n_ctx(ctx);

    // 记录开始时间
    auto tim1 = std::chrono::high_resolution_clock::now();
    // 打印信息：正在对输入进行标记化
    fprintf(stderr, "%s: tokenizing the input ..\n", __func__);

    // 对输入进行标记化
    std::vector<llama_token> tokens = ::llama_tokenize(ctx, params.prompt, add_bos);

    // 记录结束时间
    auto tim2 = std::chrono::high_resolution_clock::now();
    // 打印信息：标记化耗时
    fprintf(stderr, "%s: tokenization took %g ms\n",__func__,1e-3*std::chrono::duration_cast<std::chrono::microseconds>(tim2-tim1).count());

    // 如果标记数量小于两倍的上下文大小，则打印警告信息并返回空结果
    if (int(tokens.size()) < 2*n_ctx) {
        fprintf(stderr, "%s: you need at least %d tokens to evaluate perplexity with a context of %d\n",__func__,2*n_ctx,
                n_ctx);
        fprintf(stderr, "%s: the data file you provided tokenizes to only %zu tokens\n",__func__,tokens.size());
        return {std::move(tokens), 0., {}, {}};
    }

    // 初始化记录 logit 和概率的历史数组
    std::vector<float> logit_history;
    logit_history.resize(tokens.size());

    std::vector<float> prob_history;
    prob_history.resize(tokens.size());

    // 计算最大块数
    const int n_chunk_max = tokens.size() / n_ctx;

    // 确定要计算的块数
    const int n_chunk = params.n_chunks < 0 ? n_chunk_max : std::min(params.n_chunks, n_chunk_max);
    // 获取词汇表大小和批处理大小
    const int n_vocab = llama_n_vocab(llama_get_model(ctx));
    const int n_batch = params.n_batch;

    // 初始化计数器和负对数似然
    int count = 0;
    double nll = 0.0;
    double nll2 = 0.0;

    // 打印信息：正在计算困惑度
    fprintf(stderr, "%s: calculating perplexity over %d chunks, batch_size=%d\n", __func__, n_chunk, n_batch);
    // 创建一个线程向量，大小为硬件支持的线程数减一
    std::vector<std::thread> workers(std::thread::hardware_concurrency() - 1);

    // 打印换行符
    }

    // 计算平均负对数似然值
    nll2 /= count;
    nll /= count;
    // 计算困惑度
    const double ppl = exp(nll);
    // 计算方差
    nll2 -= nll * nll;
    // 如果方差大于0
    if (nll2 > 0) {
        // 计算标准差
        nll2 = sqrt(nll2/(count-1));
        // 打印最终估计的困惑度和标准差
        printf("Final estimate: PPL = %.4lf +/- %.5lf\n", ppl, nll2*ppl);
    } else {
        // 打印意外的负对数似然值标准差
        printf("Unexpected negative standard deviation of log(prob)\n");
    }

    // 返回结果，包括tokens, ppl, logit_history, prob_history
    return {tokens, ppl, logit_history, prob_history};
// 计算 hellaswag 评分（acc_norm）从提示中
//
// 从 HellaSwag 验证数据集提取的数据（MIT 许可证）https://github.com/rowanz/hellaswag/blob/master/data/hellaswag_val.jsonl
// 所有使用的数据字段都经过预处理，如 https://github.com/EleutherAI/lm-evaluation-harness/blob/df3da98c5405deafd519c2ddca52bb7c3fe36bef/lm_eval/tasks/hellaswag.py#L62-L68
//
// 所有 10042 个任务应该被提取，以保持结果的标准化，就像其他实现一样。
//
// 数据文件布局：
// ['??'] 表示 json 字段
// 每个任务 6 行：
// ['activity_label'] + "：" +['ctx']  - 查询的第一部分，上下文
// ['label'] - 最佳常识结尾的索引，也就是黄金结尾
// ['endings'][0] - 添加到查询的第一部分的结尾
// ['endings'][1]
// ['endings'][2]
// ['endings'][3]
std::vector<std::string> prompt_lines;
std::istringstream strstream(params.prompt);
std::string line;
    // 从字符串流中逐行读取内容，并将每行添加到 prompt_lines 容器中
    while (std::getline(strstream,line,'\n')) {
        prompt_lines.push_back(line);
    }

    // 如果 prompt_lines 的行数不是 6 的倍数，则输出错误信息并返回
    if( prompt_lines.size() % 6 != 0) {
        fprintf(stderr, "%s : number of lines in prompt not a multiple of 6.\n", __func__);
        return;
    }

    // 计算任务数量，每个任务包含 6 行
    size_t hs_task_count = prompt_lines.size()/6;
    fprintf(stderr, "%s : loaded %zu tasks from prompt.\n", __func__, hs_task_count);

    // 检查当前模型是否为 SPM 类型
    const bool is_spm = llama_vocab_type(llama_get_model(ctx)) == LLAMA_VOCAB_TYPE_SPM;
    fprintf(stderr, "================================= is_spm = %d\n", is_spm);

    // 根据当前模型类型决定是否添加 bos 标记
    const bool add_bos = is_spm;

    // 计算用于计算分数的任务数量，取参数中的值和实际任务数量的较小值
    if ( params.hellaswag_tasks < hs_task_count  ) {
        hs_task_count = params.hellaswag_tasks;
    }

    // 随机化任务顺序，以便分数能够快速稳定
    bool randomize_tasks = true;

    // 随机数生成器，用于随机化任务顺序
    std::mt19937 rng(1);

    // hellaswag 任务的数据结构
    struct hs_data_t {
        std::string context;
        size_t gold_ending_idx;
        std::string ending[4];
        size_t ending_logprob_count[4];
        double ending_logprob[4];
    };

    // 输出选择的任务数量和任务顺序的信息
    fprintf(stderr, "%s : selecting %zu %s tasks.\n", __func__, hs_task_count, (randomize_tasks?"randomized":"the first")  );

    // 为 hellaswag 任务创建数据存储器
    hs_data_t *hs_data = new hs_data_t[hs_task_count];
    // 遍历任务数量次数
    for (size_t i=0; i < hs_task_count; i++) {
        // 将当前索引赋值给 idx
        size_t idx = i;

        // 如果需要随机化任务顺序
        if (randomize_tasks) {
            // 创建均匀分布的随机数生成器
            std::uniform_int_distribution<size_t> dist(0, prompt_lines.size()/6-1 ) ;
            // 从分布中获取随机索引
            idx = dist(rng);
        }

        // 将选定的上下文存储到 hs_data 结构体中
        hs_data[i].context = prompt_lines[idx*6];
        // 将选定的正确结尾索引存储到 hs_data 结构体中
        hs_data[i].gold_ending_idx = std::stoi( prompt_lines[idx*6+1] );
        // 将选定的四个结尾存储到 hs_data 结构体中
        for (size_t j=0; j < 4; j++) {
            hs_data[i].ending[j] = prompt_lines[idx*6+2+j];
        }

        // 如果需要随机化任务顺序
        if (randomize_tasks) {
            // 从 prompt_lines 中删除选定的随机例子
            prompt_lines.erase( std::next(prompt_lines.begin(),idx*6)  , std::next(prompt_lines.begin(),idx*6+6) );
        }
    }

    // 打印计算 hellaswag 分数的提示信息
    fprintf(stderr, "%s : calculating hellaswag score over selected tasks.\n", __func__);
    // 打印表头
    printf("\ntask\tacc_norm\n");

    // 初始化准确率为 0
    double acc = 0.0f;
    // 获取词汇表大小
    const int n_vocab = llama_n_vocab(llama_get_model(ctx));
    // 获取上下文大小
    const int n_ctx = llama_n_ctx(ctx);

    // 初始化包含四个子向量的 ending_tokens
    std::vector<std::vector<int>> ending_tokens(4);

    // 初始化包含词汇表大小个元素的 tok_logits
    std::vector<float> tok_logits(n_vocab);
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

        // 如果金标准结束获得最大对数概率，则增加一个准确度点
        if (ending_logprob_max_idx == hs_data[task_idx].gold_ending_idx) {
            acc += 1.0;
        }

        // 打印累积准确度的平均值 x 100
        printf("%zu\t%.8lf\n",task_idx+1, acc/double(task_idx+1)*100.0);
        fflush(stdout);
    }

    delete [] hs_data;

    printf("\n");
}

int main(int argc, char ** argv) {
    gpt_params params;

    params.n_batch = 512;
    if (!gpt_params_parse(argc, argv, params)) {
        return 1;
    }

    params.logits_all = true;
    params.n_batch = std::min(params.n_batch, params.n_ctx);

    if (params.ppl_stride > 0) {
        fprintf(stderr, "将执行跨步困惑度计算 -> 调整上下文大小从 %d 到 %d\n",
                params.n_ctx, params.n_ctx + params.ppl_stride/2);
        params.n_ctx += params.ppl_stride/2;
    }

    print_build_info();

    if (params.seed == LLAMA_DEFAULT_SEED) {
        params.seed = time(NULL);
    }

    fprintf(stderr, "%s: seed  = %u\n", __func__, params.seed);

    std::mt19937 rng(params.seed);
    // 如果参数中包含随机提示，则生成随机提示
    if (params.random_prompt) {
        params.prompt = gpt_random_prompt(rng);
    }

    // 初始化LLAMA后端
    llama_backend_init(params.numa);

    // 声明LLAMA模型和上下文
    llama_model * model;
    llama_context * ctx;

    // 加载模型并应用LORA适配器（如果有的话）
    std::tie(model, ctx) = llama_init_from_gpt_params(params);
    // 如果无法加载模型，则打印错误信息并返回1
    if (model == NULL) {
        fprintf(stderr, "%s: error: unable to load model\n", __func__);
        return 1;
    }

    // 获取模型训练时的上下文标记数，并与参数中指定的上下文标记数进行比较
    const int n_ctx_train = llama_n_ctx_train(model);
    if (params.n_ctx > n_ctx_train) {
        fprintf(stderr, "%s: warning: model was trained on only %d context tokens (%d specified)\n",
                __func__, n_ctx_train, params.n_ctx);
    }

    // 打印系统信息
    {
        fprintf(stderr, "\n");
        fprintf(stderr, "%s\n", get_system_info(params).c_str());
    }

    // 如果参数中包含Hellaswag，则调用Hellaswag评分函数，否则计算困惑度
    struct results_perplexity results;
    if (params.hellaswag) {
        hellaswag_score(ctx, params);
    } else {
        results = perplexity(ctx, params);
    }

    // 打印LLAMA的时间信息
    llama_print_timings(ctx);
    // 写入日志文件
    write_logfile(ctx, params, model, results);

    // 释放LLAMA上下文和模型
    llama_free(ctx);
    llama_free_model(model);

    // 释放LLAMA后端资源
    llama_backend_free();

    // 返回0表示成功
    return 0;
# 闭合前面的函数定义
```