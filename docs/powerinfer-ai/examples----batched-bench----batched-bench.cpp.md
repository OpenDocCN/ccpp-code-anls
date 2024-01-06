# `PowerInfer\examples\batched-bench\batched-bench.cpp`

```
// 包含常用头文件
#include "common.h"
#include "llama.h"

// 包含算法、数学、输入输出、字符串和向量头文件
#include <algorithm>
#include <cmath>
#include <cstdio>
#include <string>
#include <vector>

// 修改输入字符串
// 解析输入字符串，将逗号分隔的数字转换为整数存入向量中
static std::vector<int> parse_list(char * p) {
    // 创建整数向量
    std::vector<int> ret;

    // 创建指针 q 指向输入字符串 p
    char * q = p;

    // 遍历输入字符串
    while (*p) {
        // 如果当前字符是逗号
        if (*p == ',') {
            // 将当前字符改为字符串结束符
            *p = '\0';
            // 将 q 到 p 之间的字符串转换为整数并存入向量中
            ret.push_back(std::atoi(q));
            // 更新指针 q 指向下一个数字的起始位置
            q = p + 1;
    }

    // 指针 p 向后移动一位
    ++p;
}

// 将解析得到的数字添加到 ret 数组中
ret.push_back(std::atoi(q));

// 返回解析得到的参数数组
return ret;
}

// 主函数
int main(int argc, char ** argv) {
    // 定义参数对象
    gpt_params params;

    // 如果命令行参数个数为 1 或者第一个参数的第一个字符为 '-'，则打印使用说明并返回 1
    if (argc == 1 || argv[1][0] == '-') {
        printf("usage: %s MODEL_PATH [N_KV_MAX] [IS_PP_SHARED] [NGL] [MMQ] <PP> <TG> <PL>\n" , argv[0]);
        printf("  <PP>, <TG> and PL are comma-separated lists of numbers without spaces\n\n");
        printf("  example: %s ggml-model-f16.gguf 2048 0 999 0 128,256,512 128,256 1,2,4,8,16,32\n\n", argv[0]);
        return 1 ;
    }
    # 设置最大键值对数为2048
    int n_kv_max     = 2048;
    # 设置是否共享内存为0
    int is_pp_shared = 0;
    # 设置GPU层数为0
    int n_gpu_layers = 0;
    # 设置mmq为0
    int mmq          = 0;

    # 初始化n_pp向量，包含一系列整数
    std::vector<int> n_pp = { 128, 256, 512, 1024, 2048, 3584, 7680, };
    # 初始化n_tg向量，包含一系列整数
    std::vector<int> n_tg = { 128, 256, };
    # 初始化n_pl向量，包含一系列整数
    std::vector<int> n_pl = { 1, 2, 4, 8, 16, 32, };
    # 如果需要更多选项，可以使用下面的注释代码替换上面的代码
    #std::vector<int> n_pl = { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 32, };

    # 如果命令行参数数量大于等于2，将第二个参数赋值给params.model
    if (argc >= 2) {
        params.model = argv[1];
    }

    # 如果命令行参数数量大于等于3，将第三个参数转换为整数并赋值给n_kv_max
    if (argc >= 3) {
        n_kv_max = std::atoi(argv[2]);
    }

    # 如果命令行参数数量大于等于4，将第四个参数转换为整数并赋值给is_pp_shared
    if (argc >= 4) {
        is_pp_shared = std::atoi(argv[3]);
    }
    }

    // 如果命令行参数数量大于等于5，将第4个参数转换为整数并赋值给n_gpu_layers
    if (argc >= 5) {
        n_gpu_layers = std::atoi(argv[4]);
    }

    // 如果命令行参数数量大于等于6，将第5个参数转换为整数并赋值给mmq
    if (argc >= 6) {
        mmq = std::atoi(argv[5]);
    }

    // 如果命令行参数数量大于等于7，调用parse_list函数解析第6个参数并赋值给n_pp
    if (argc >= 7) {
        n_pp = parse_list(argv[6]);
    }

    // 如果命令行参数数量大于等于8，调用parse_list函数解析第7个参数并赋值给n_tg
    if (argc >= 8) {
        n_tg = parse_list(argv[7]);
    }

    // 如果命令行参数数量大于等于9，调用parse_list函数解析第8个参数并赋值给n_pl
    if (argc >= 9) {
        n_pl = parse_list(argv[8]);
    }

    // 初始化LLM（LLM是什么？）

    // 初始化LLM后端，参数为NUMA节点
    llama_backend_init(params.numa);

    // 初始化模型
    // 创建LLM模型参数对象，使用默认参数
    llama_model_params model_params = llama_model_default_params();

    // 设置GPU层数
    model_params.n_gpu_layers = n_gpu_layers;

    // 从文件加载模型，参数为模型文件名和模型参数
    llama_model * model = llama_load_model_from_file(params.model.c_str(), model_params);

    // 如果模型加载失败，输出错误信息并返回1
    if (model == NULL) {
        fprintf(stderr , "%s: error: unable to load model\n" , __func__);
        return 1;
    }

    // 创建LLM上下文参数对象，使用默认参数
    llama_context_params ctx_params = llama_context_default_params();
    # 设置随机种子
    ctx_params.seed      = 1234;
    # 设置上下文的最大大小
    ctx_params.n_ctx     = n_kv_max;
    # 设置每个批次的大小
    ctx_params.n_batch   = 512;
    # 设置用于矩阵乘法的参数
    ctx_params.mul_mat_q = mmq;

    # 设置上下文的线程数
    ctx_params.n_threads       = params.n_threads;
    # 如果批次的线程数为-1，则使用上下文的线程数，否则使用指定的批次线程数
    ctx_params.n_threads_batch = params.n_threads_batch == -1 ? params.n_threads : params.n_threads_batch;

    # 使用给定的模型和上下文参数创建一个新的上下文对象
    llama_context * ctx = llama_new_context_with_model(model, ctx_params);

    # 如果创建上下文对象失败，则打印错误信息并返回1
    if (ctx == NULL) {
        fprintf(stderr , "%s: error: failed to create the llama_context\n" , __func__);
        return 1;
    }

    # 初始化一个批次对象
    llama_batch batch = llama_batch_init(n_kv_max, 0, 1);

    # 在每个批次中解码ctx_params.n_batch个标记
    auto decode_helper = [](llama_context * ctx, llama_batch & batch, int32_t n_batch) {
// 循环遍历批处理中的标记，每次增加 n_batch 个标记
for (int32_t i = 0; i < (int32_t) batch.n_tokens; i += n_batch) {
    // 计算当前批处理中实际要处理的标记数量，取 n_batch 和 (batch.n_tokens - i) 的最小值
    const int32_t n_tokens = std::min(n_batch, (int32_t) (batch.n_tokens - i));

    // 创建批处理视图，包括要处理的标记数量、标记数组的偏移量、位置数组的偏移量、序列 ID 数组的偏移量、logits 数组的偏移量
    llama_batch batch_view = {
        n_tokens,
        batch.token    + i,
        nullptr,
        batch.pos      + i,
        batch.n_seq_id + i,
        batch.seq_id   + i,
        batch.logits   + i,
        0, 0, 0, // 未使用的字段
    };

    // 调用 llama_decode 函数对批处理视图进行解码
    const int ret = llama_decode(ctx, batch_view);
    // 如果解码失败，记录日志并返回 false
    if (ret != 0) {
        LOG_TEE("failed to decode the batch, n_batch = %d, ret = %d\n", n_batch, ret);
        return false;
    }
}
    // 返回 true
    return true;
    };

    // 预热
    {
        // 循环执行16次，向批处理中添加数据
        for (int i = 0; i < 16; ++i) {
            llama_batch_add(batch, 0, i, { 0 }, false);
        }

        // 如果解码辅助函数返回 false，则记录错误信息并返回1
        if (!decode_helper(ctx, batch, ctx_params.n_batch)) {
            LOG_TEE("%s: llama_decode() failed\n", __func__);
            return 1;
        }
    }

    // 记录信息
    LOG_TEE("\n");
    LOG_TEE("%s: n_kv_max = %d, is_pp_shared = %d, n_gpu_layers = %d, mmq = %d\n", __func__, n_kv_max, is_pp_shared, n_gpu_layers, mmq);
    LOG_TEE("\n");
```
在这段代码中，我们可以看到：
- 第一个注释解释了返回 true 的作用。
- 第二个注释解释了预热的目的，以及循环向批处理中添加数据和调用解码辅助函数的过程。
- 最后两行注释解释了记录信息的作用，以及记录了一些变量的值。
# 打印表头信息
LOG_TEE("|%6s | %6s | %4s | %6s | %8s | %8s | %8s | %8s | %8s | %8s |\n", "PP",     "TG",     "B",    "N_KV",     "T_PP s",   "S_PP t/s", "T_TG s",   "S_TG t/s", "T s",      "S t/s");
# 打印表头分隔线
LOG_TEE("|%6s-|-%6s-|-%4s-|-%6s-|-%8s-|-%8s-|-%8s-|-%8s-|-%8s-|-%8s-|\n", "------", "------", "----", "------", "--------", "--------", "--------", "--------", "--------", "--------");

# 循环遍历不同的参数组合
for (        int i_pp = 0; i_pp < (int) n_pp.size(); ++i_pp) {
    for (    int i_tg = 0; i_tg < (int) n_tg.size(); ++i_tg) {
        for (int i_pl = 0; i_pl < (int) n_pl.size(); ++i_pl) {
            # 获取当前参数组合的值
            const int pp = n_pp[i_pp];
            const int tg = n_tg[i_tg];
            const int pl = n_pl[i_pl];

            # 根据不同的条件计算 n_ctx_req 的值
            const int n_ctx_req = is_pp_shared ? pp + pl*tg : pl*(pp + tg);

            # 如果 n_ctx_req 大于 n_kv_max，则跳过当前循环
            if (n_ctx_req > n_kv_max) {
                continue;
            }

            # 清空 batch
            llama_batch_clear(batch);

            # 根据不同的条件计算 n_tokens 的值
            const int n_tokens = is_pp_shared ? pp : pl*pp;
// 遍历 n_tokens 次，向 batch 中添加数据
for (int i = 0; i < n_tokens; ++i) {
    llama_batch_add(batch, 0, i, { 0 }, false);
}
// 设置 batch 中最后一个 token 的 logits 为 true
batch.logits[batch.n_tokens - 1] = true;

// 记录当前时间
const auto t_pp_start = ggml_time_us();

// 清空 kv 缓存
llama_kv_cache_clear(ctx);

// 调用 decode_helper 函数进行解码，如果失败则记录日志并返回 1
if (!decode_helper(ctx, batch, ctx_params.n_batch)) {
    LOG_TEE("%s: llama_decode() failed\n", __func__);
    return 1;
}

// 如果是 pp_shared，则将数据从 ctx 的第 0 个位置复制到后续位置
if (is_pp_shared) {
    for (int32_t i = 1; i < pl; ++i) {
        llama_kv_cache_seq_cp(ctx, 0, i, 0, pp);
    }
}
// 记录当前时间点，用于计算时间间隔
const auto t_pp_end = ggml_time_us();

// 记录当前时间点，用于计算时间间隔
const auto t_tg_start = ggml_time_us();

// 循环执行 tg 次
for (int i = 0; i < tg; ++i) {
    // 清空批处理对象
    llama_batch_clear(batch);

    // 循环执行 pl 次
    for (int j = 0; j < pl; ++j) {
        // 向批处理对象中添加数据
        llama_batch_add(batch, 0, pp + i, { j }, true);
    }

    // 调用 decode_helper 函数进行解码
    if (!decode_helper(ctx, batch, ctx_params.n_batch)) {
        // 如果解码失败，记录日志并返回错误码
        LOG_TEE("%s: llama_decode() failed\n", __func__);
        return 1;
    }
}

// 记录当前时间点，用于计算时间间隔
const auto t_tg_end = ggml_time_us();

// 记录 n_ctx_req 的值
const int32_t n_kv = n_ctx_req;
// 计算 t_pp，即 pp 阶段的时间
const float t_pp = (t_pp_end - t_pp_start) / 1000000.0f;
// 计算 t_tg，即 tg 阶段的时间
const float t_tg = (t_tg_end - t_tg_start) / 1000000.0f;
// 计算总时间 t
const float t    = t_pp + t_tg;

// 计算 pp 阶段的速度
const float speed_pp = is_pp_shared ? pp / t_pp : pl*pp / t_pp;
// 计算 tg 阶段的速度
const float speed_tg = pl*tg / t_tg;
// 计算总速度
const float speed    = n_kv / t;

// 打印日志，显示 pp, tg, pl, n_kv, t_pp, speed_pp, t_tg, speed_tg, t, speed 的值
LOG_TEE("|%6d | %6d | %4d | %6d | %8.3f | %8.2f | %8.3f | %8.2f | %8.3f | %8.2f |\n", pp, tg, pl, n_kv, t_pp, speed_pp, t_tg, speed_tg, t, speed);

// 打印时间统计信息
llama_print_timings(ctx);

// 释放批处理资源
llama_batch_free(batch);

// 释放上下文资源
llama_free(ctx);
// 释放模型资源
llama_free_model(model);
# 释放后端资源
llama_backend_free();
# 打印换行符到标准错误流
fprintf(stderr, "\n\n");
# 返回 0，表示程序正常结束
return 0;
```