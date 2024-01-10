# `PowerInfer\common\common.h`

```
// Various helper functions and utilities

#pragma once

#include "llama.h"  // 包含 llama.h 头文件

#include "sampling.h"  // 包含 sampling.h 头文件

#define LOG_NO_FILE_LINE_FUNCTION  // 定义 LOG_NO_FILE_LINE_FUNCTION 宏

#include "log.h"  // 包含 log.h 头文件

#include <cmath>  // 包含数学函数的头文件
#include <string>  // 包含字符串处理的头文件
#include <vector>  // 包含向量处理的头文件
#include <random>  // 包含随机数生成的头文件
#include <thread>  // 包含多线程处理的头文件
#include <unordered_map>  // 包含无序映射处理的头文件
#include <tuple>  // 包含元组处理的头文件

#ifdef _WIN32
#define DIRECTORY_SEPARATOR '\\'  // 如果是 Windows 系统，定义 DIRECTORY_SEPARATOR 为反斜杠
#else
#define DIRECTORY_SEPARATOR '/'  // 如果不是 Windows 系统，定义 DIRECTORY_SEPARATOR 为斜杠
#endif // _WIN32

#define die(msg)          do { fputs("error: " msg "\n", stderr);                exit(1); } while (0)  // 定义 die 函数，输出错误信息并退出程序
#define die_fmt(fmt, ...) do { fprintf(stderr, "error: " fmt "\n", __VA_ARGS__); exit(1); } while (0)  // 定义 die_fmt 函数，格式化输出错误信息并退出程序

#define print_build_info() do {                                                                     \
    fprintf(stderr, "%s: build = %d (%s)\n", __func__, LLAMA_BUILD_NUMBER, LLAMA_COMMIT);           \
    fprintf(stderr, "%s: built with %s for %s\n", __func__, LLAMA_COMPILER, LLAMA_BUILD_TARGET);    \
} while(0)  // 定义 print_build_info 函数，输出构建信息

// build info
extern int LLAMA_BUILD_NUMBER;  // 声明 LLAMA_BUILD_NUMBER 变量
extern char const *LLAMA_COMMIT;  // 声明 LLAMA_COMMIT 变量
extern char const *LLAMA_COMPILER;  // 声明 LLAMA_COMPILER 变量
extern char const *LLAMA_BUILD_TARGET;  // 声明 LLAMA_BUILD_TARGET 变量

//
// CLI argument parsing
//
int32_t get_num_physical_cores();  // 声明获取物理核心数的函数

struct gpt_params {
    uint32_t seed                           = -1;    // RNG seed  // 初始化种子为 -1
    int32_t n_threads                       = get_num_physical_cores();  // 初始化线程数为物理核心数
    int32_t n_threads_batch                 = -1;    // number of threads to use for batch processing (-1 = use n_threads)  // 初始化批处理线程数为 -1
    int32_t n_predict                       = -1;    // new tokens to predict  // 初始化预测新标记数为 -1
    int32_t n_ctx                           = 512;   // context size  // 初始化上下文大小为 512
    int32_t n_batch                         = 32;    // batch size for prompt processing (must be >=32 to use BLAS)  // 初始化提示处理的批处理大小为 32
    int32_t n_keep                          = 0;     // number of tokens to keep from initial prompt  // 初始化从初始提示中保留的标记数为 0
    int32_t n_draft                         = 16;    // number of tokens to draft during speculative decoding  // 初始化在推测解码过程中起草的标记数为 16
    int32_t n_chunks                        = -1;    // max number of chunks to process (-1 = unlimited)  // 初始化要处理的最大块数为 -1
    // 并行解码的序列数
    int32_t n_parallel                      = 1;     // number of parallel sequences to decode
    // 解码的序列数
    int32_t n_sequences                     = 1;     // number of sequences to decode
    // 接受概率
    float   p_accept                        = 0.5f;  // speculative decoding accept probability
    // 分裂概率
    float   p_split                         = 0.1f;  // speculative decoding split probability
    // 存储在 VRAM 中的层数 (-1 - 使用默认值)
    int32_t n_gpu_layers                    = -1;    // number of layers to store in VRAM (-1 - use default)
    // 用于草稿模型的 VRAM 中的层数 (-1 - 使用默认值)
    int32_t n_gpu_layers_draft              = -1;    // number of layers to store in VRAM for the draft model (-1 - use default)
    // 用于临时和小张量的 GPU
    int32_t main_gpu                        = 0;     // the GPU that is used for scratch and small tensors
    // 张量如何分布在多个 GPU 上
    float   tensor_split[LLAMA_MAX_DEVICES] = {0};   // how split tensors should be distributed across GPUs
    // 如果非零，则使用给定宽度的波束搜索
    int32_t n_beams                         = 0;     // if non-zero then use beam search of given width.
    // RoPE 基础频率
    float   rope_freq_base                  = 0.0f;  // RoPE base frequency
    // RoPE 频率缩放因子
    float   rope_freq_scale                 = 0.0f;  // RoPE frequency scaling factor
    // VRAM 预算 (GB) (-1 - 使用可用的 VRAM)
    float   vram_budget_gb                  = -1.0f; // VRAM budget in GB (-1 - use available VRAM)
    // YaRN 外推混合因子
    float   yarn_ext_factor                 = -1.0f; // YaRN extrapolation mix factor
    // YaRN 注意力因子
    float   yarn_attn_factor                = 1.0f;  // YaRN magnitude scaling factor
    // YaRN 低校正维度
    float   yarn_beta_fast                  = 32.0f; // YaRN low correction dim
    // YaRN 高校正维度
    float   yarn_beta_slow                  = 1.0f;  // YaRN high correction dim
    // YaRN 原始上下文长度
    int32_t yarn_orig_ctx                   = 0;     // YaRN original context length
    // RoPE 缩放类型
    int8_t  rope_scaling_type               = LLAMA_ROPE_SCALING_UNSPECIFIED; // TODO: better to be int32_t for alignment
                                                                              //       pinging @cebtenzzre

    // 采样参数
    struct llama_sampling_params sparams;

    // 模型路径
    std::string model             = "models/7B/ggml-model-f16.gguf"; // model path
    // 初始化 draft model 为空字符串
    std::string model_draft       = "";                              
    // 初始化 model alias 为 "unknown"
    std::string model_alias       = "unknown"; 
    // 初始化用户输入提示为空字符串
    std::string prompt            = "";
    // 初始化外部提示文件名为空字符串
    std::string prompt_file       = "";  
    // 初始化保存/加载提示评估状态的文件路径为空字符串
    std::string path_prompt_cache = "";  
    // 初始化用户输入前缀为空字符串
    std::string input_prefix      = "";  
    // 初始化用户输入后缀为空字符串
    std::string input_suffix      = "";  
    // 初始化用于提示更多用户输入的字符串向量为空
    std::vector<std::string> antiprompt; 
    // 初始化保存 YAML 日志文件的目录为空字符串
    std::string logdir            = "";  

    // TODO: 避免使用元组，使用结构体
    // 初始化 lora 适配器路径和用户定义比例的元组向量为空
    std::vector<std::tuple<std::string, float>> lora_adapter; 
    // 初始化 lora 适配器的基础模型路径为空字符串
    std::string lora_base  = "";                              

    // 初始化刷新 GPU 索引文件的布尔值为 false
    bool reset_gpu_index   = false; 
    // 初始化禁用加载 GPU 索引和拆分 ffn 的布尔值为 false
    bool disale_gpu_index  = false; 

    // 初始化用于困惑度计算的步长为 0
    int  ppl_stride        = 0;     
    // 初始化困惑度输出类型为 0
    int  ppl_output_type   = 0;     
    // 初始化 HellaSwag 计算为 false
    bool   hellaswag       = false; 
    // 初始化计算 HellaSwag 评分时使用的任务数为 400
    size_t hellaswag_tasks = 400;   

    // 如果为 true，则使用 mul_mat_q 内核而不是 cuBLAS
    bool mul_mat_q         = true;  
    // 如果为 true，则使用 f16 而不是 f32 作为内存 kv
    bool memory_f16        = true;  
    // 如果未提供提示，则不随机化提示
    bool random_prompt     = false; 
    // 是否使用颜色来区分生成和输入
    bool use_color         = false; 
    // 是否交互模式
    bool interactive       = false; 
    // 是否保存用户输入和生成的内容到提示缓存
    bool prompt_cache_all  = false; 
    // 以只读模式打开提示缓存，不更新它
    bool prompt_cache_ro   = false; 

    // 是否只获取句子嵌入
    bool embedding         = false; 
    // 是否转义 "\n", "\r", "\t", "\'", "\"", 和 "\\"
    bool escape            = false; 
    // 立即等待用户输入
    bool interactive_first = false; 
    // 反转使用 `\` 的方式
    bool multiline_input   = false; 
    // 提高与子进程和有限控制台的兼容性
    bool simple_io         = false; 
    // 动态插入新的序列以进行解码
    bool cont_batching     = false; 

    // 在用户输入前添加 BOS，即在 input_prefix 前添加 BOS
    bool input_prefix_bos  = false; 
    // 忽略生成的 EOS 标记
    bool ignore_eos        = false; 
    // 指令模式（用于 Alpaca 模型）
    bool instruct          = false; 
    // 返回批处理中所有标记的对数
    bool logits_all        = false; 
    // 使用 mmap 进行更快的加载
    bool use_mmap          = true;  
    // 使用 mlock 以保持模型在内存中
    bool use_mlock         = false; 
    // 尝试在某些 NUMA 系统上帮助优化
    bool numa              = false; 
    // 在生成前打印提示标记
    bool verbose_prompt    = false; 
    // 使用 infill 模式
    bool infill            = false; 

    // 多模态模型（参见 examples/llava）
    std::string mmproj = ""; // 多模态投影仪的路径
    std::string image = ""; // 图像文件的路径
};

// 解析带有额外参数的命令行参数
bool gpt_params_parse_ex(int argc, char ** argv, gpt_params & params);

// 解析命令行参数
bool gpt_params_parse(int argc, char ** argv, gpt_params & params);

// 打印程序的用法信息
void gpt_print_usage(int argc, char ** argv, const gpt_params & params);

// 获取系统信息
std::string get_system_info(const gpt_params & params);

// 生成随机提示
std::string gpt_random_prompt(std::mt19937 & rng);

// 处理字符串中的转义字符
void process_escapes(std::string& input);

//
// Model utils
//

// TODO: avoid tuplue, use struct
// 从 gpt_params 中初始化 llama_model 和 llama_context
std::tuple<struct llama_model *, struct llama_context *> llama_init_from_gpt_params(gpt_params & params);

// 从 gpt_params 中获取 llama_model_params
struct llama_model_params   llama_model_params_from_gpt_params  (const gpt_params & params);

// 从 gpt_params 中获取 llama_context_params
struct llama_context_params llama_context_params_from_gpt_params(const gpt_params & params);

// Batch utils

// 清空 batch
void llama_batch_clear(struct llama_batch & batch);

// 向 batch 中添加数据
void llama_batch_add(
                 struct llama_batch & batch,
                        llama_token   id,
                          llama_pos   pos,
    const std::vector<llama_seq_id> & seq_ids,
                               bool   logits);

//
// Vocab utils
//

// 将字符串分词为 token 的向量
// 类似于 Python 的 `tokenizer.encode`
std::vector<llama_token> llama_tokenize(
  const struct llama_context * ctx,
           const std::string & text,
                        bool   add_bos,
                        bool   special = false);

// 将字符串分词为 token 的向量
// 类似于 Python 的 `tokenizer.encode`
std::vector<llama_token> llama_tokenize(
    const struct llama_model * model,
           const std::string & text,
                        bool   add_bos,
                        bool   special = false);

// 将 token 转换为对应的片段
// 类似于 Python 的 `tokenizer.id_to_piece`
std::string llama_token_to_piece(
        const struct llama_context * ctx,
                       llama_token   token);

// TODO: these should be moved in llama.h C-style API under single `llama_detokenize` function
//       that takes into account the tokenizer type and decides how to handle the leading space
//
// 将一个标记向量解标记为字符串
// 应该类似于 Python 的 `tokenizer.decode`
// 从第一个非 BOS 标记中移除前导空格
std::string llama_detokenize_spm(
                         llama_context * ctx,
        const std::vector<llama_token> & tokens);

// 将一个标记向量解标记为字符串
// 应该类似于 Python 的 `tokenizer.decode`
std::string llama_detokenize_bpe(
                         llama_context * ctx,
        const std::vector<llama_token> & tokens);

//
// YAML 工具
//

// 创建带有父目录的目录
bool create_directory_with_parents(const std::string & path);
// 将浮点数向量转储为 YAML
void dump_vector_float_yaml(FILE * stream, const char * prop_name, const std::vector<float> & data);
// 将整数向量转储为 YAML
void dump_vector_int_yaml(FILE * stream, const char * prop_name, const std::vector<int> & data);
// 将多行字符串转储为 YAML
void dump_string_yaml_multiline(FILE * stream, const char * prop_name, const char * data);
// 获取可排序的时间戳
std::string get_sortable_timestamp();

// 转储非结果信息为 YAML
void dump_non_result_info_yaml(
    FILE * stream, const gpt_params & params, const llama_context * lctx,
    const std::string & timestamp, const std::vector<int> & prompt_tokens, const char * model_desc);
```