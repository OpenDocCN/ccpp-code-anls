# `PowerInfer\examples\batched-bench\batched-bench.cpp`

```cpp
#include "common.h"
#include "llama.h"

#include <algorithm>
#include <cmath>
#include <cstdio>
#include <string>
#include <vector>

// mutates the input string
// 解析以逗号分隔的数字字符串，将其转换为整数数组
static std::vector<int> parse_list(char * p) {
    std::vector<int> ret;

    char * q = p;

    // 遍历字符串
    while (*p) {
        // 如果当前字符是逗号，则将其替换为字符串结束符，并将前面的数字字符串转换为整数存入数组中
        if (*p == ',') {
            *p = '\0';
            ret.push_back(std::atoi(q));
            q = p + 1;
        }

        ++p;
    }

    // 将最后一个数字字符串转换为整数存入数组中
    ret.push_back(std::atoi(q));

    return ret;
}

int main(int argc, char ** argv) {
    gpt_params params;

    // 如果没有参数或第一个参数以'-'开头，则打印用法说明并返回1
    if (argc == 1 || argv[1][0] == '-') {
        printf("usage: %s MODEL_PATH [N_KV_MAX] [IS_PP_SHARED] [NGL] [MMQ] <PP> <TG> <PL>\n" , argv[0]);
        printf("  <PP>, <TG> and PL are comma-separated lists of numbers without spaces\n\n");
        printf("  example: %s ggml-model-f16.gguf 2048 0 999 0 128,256,512 128,256 1,2,4,8,16,32\n\n", argv[0]);
        return 1 ;
    }

    int n_kv_max     = 2048;
    int is_pp_shared = 0;
    int n_gpu_layers = 0;
    int mmq          = 0;

    // 初始化默认的参数列表
    std::vector<int> n_pp = { 128, 256, 512, 1024, 2048, 3584, 7680, };
    std::vector<int> n_tg = { 128, 256, };
    std::vector<int> n_pl = { 1, 2, 4, 8, 16, 32, };
    //std::vector<int> n_pl = { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 32, };

    // 根据参数个数设置参数值
    if (argc >= 2) {
        params.model = argv[1];
    }

    if (argc >= 3) {
        n_kv_max = std::atoi(argv[2]);
    }

    if (argc >= 4) {
        is_pp_shared = std::atoi(argv[3]);
    }

    if (argc >= 5) {
        n_gpu_layers = std::atoi(argv[4]);
    }

    if (argc >= 6) {
        mmq = std::atoi(argv[5]);
    }

    if (argc >= 7) {
        n_pp = parse_list(argv[6]);
    }

    if (argc >= 8) {
        n_tg = parse_list(argv[7]);
    }

    if (argc >= 9) {
        n_pl = parse_list(argv[8]);
    }

    // 初始化LLM
    llama_backend_init(params.numa);

    // 初始化模型参数
    llama_model_params model_params = llama_model_default_params();
    # 设置模型参数中的 GPU 层数
    model_params.n_gpu_layers = n_gpu_layers;

    # 从文件中加载 LLAMA 模型
    llama_model * model = llama_load_model_from_file(params.model.c_str(), model_params);

    # 如果加载模型失败，则输出错误信息并返回 1
    if (model == NULL) {
        fprintf(stderr , "%s: error: unable to load model\n" , __func__);
        return 1;
    }

    # 设置 LLAMA 上下文参数为默认值
    llama_context_params ctx_params = llama_context_default_params();

    # 设置上下文参数的种子、最大键值数、批处理大小、乘法矩阵 Q
    ctx_params.seed      = 1234;
    ctx_params.n_ctx     = n_kv_max;
    ctx_params.n_batch   = 512;
    ctx_params.mul_mat_q = mmq;

    # 设置上下文参数的线程数和批处理线程数
    ctx_params.n_threads       = params.n_threads;
    ctx_params.n_threads_batch = params.n_threads_batch == -1 ? params.n_threads : params.n_threads_batch;

    # 创建 LLAMA 上下文对象
    llama_context * ctx = llama_new_context_with_model(model, ctx_params);

    # 如果创建上下文对象失败，则输出错误信息并返回 1
    if (ctx == NULL) {
        fprintf(stderr , "%s: error: failed to create the llama_context\n" , __func__);
        return 1;
    }

    # 初始化 LLAMA 批处理对象
    llama_batch batch = llama_batch_init(n_kv_max, 0, 1);

    # 在 ctx_params.n_batch 个标记的批次中进行解码
    auto decode_helper = [](llama_context * ctx, llama_batch & batch, int32_t n_batch) {
        for (int32_t i = 0; i < (int32_t) batch.n_tokens; i += n_batch) {
            const int32_t n_tokens = std::min(n_batch, (int32_t) (batch.n_tokens - i));

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

            # 调用 LLAMA 解码函数进行解码
            const int ret = llama_decode(ctx, batch_view);
            if (ret != 0) {
                LOG_TEE("failed to decode the batch, n_batch = %d, ret = %d\n", n_batch, ret);
                return false;
            }
        }

        return true;
    };

    # 预热
    {
        // 使用循环向批处理中添加数据
        for (int i = 0; i < 16; ++i) {
            llama_batch_add(batch, 0, i, { 0 }, false);
        }
    
        // 如果解码辅助函数返回失败，则记录错误信息并返回1
        if (!decode_helper(ctx, batch, ctx_params.n_batch)) {
            LOG_TEE("%s: llama_decode() failed\n", __func__);
            return 1;
        }
    }
    
    // 打印空行
    LOG_TEE("\n");
    // 打印函数信息和参数值
    LOG_TEE("%s: n_kv_max = %d, is_pp_shared = %d, n_gpu_layers = %d, mmq = %d\n", __func__, n_kv_max, is_pp_shared, n_gpu_layers, mmq);
    // 打印空行
    LOG_TEE("\n");
    
    // 打印表头
    LOG_TEE("|%6s | %6s | %4s | %6s | %8s | %8s | %8s | %8s | %8s | %8s |\n", "PP",     "TG",     "B",    "N_KV",     "T_PP s",   "S_PP t/s", "T_TG s",   "S_TG t/s", "T s",      "S t/s");
    // 打印表格分隔线
    LOG_TEE("|%6s-|-%6s-|-%4s-|-%6s-|-%8s-|-%8s-|-%8s-|-%8s-|-%8s-|-%8s-|\n", "------", "------", "----", "------", "--------", "--------", "--------", "--------", "--------", "--------");
    
    }
    
    // 打印 llama 的时间信息
    llama_print_timings(ctx);
    
    // 释放批处理内存
    llama_batch_free(batch);
    
    // 释放 llama 上下文内存
    llama_free(ctx);
    // 释放 llama 模型内存
    llama_free_model(model);
    
    // 释放 llama 后端内存
    llama_backend_free();
    
    // 打印空行
    fprintf(stderr, "\n\n");
    
    // 返回0表示成功
    return 0;
    }
# 闭合前面的函数定义
```