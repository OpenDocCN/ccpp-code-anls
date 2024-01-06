# `PowerInfer\common\common.h`

```
// 包含 llama.h 头文件
#include "llama.h"

// 包含 sampling.h 头文件
#include "sampling.h"

// 定义 LOG_NO_FILE_LINE_FUNCTION 宏
#define LOG_NO_FILE_LINE_FUNCTION
// 包含 log.h 头文件
#include "log.h"

// 包含数学函数头文件
#include <cmath>
// 包含字符串处理头文件
#include <string>
// 包含向量容器头文件
#include <vector>
// 包含随机数生成头文件
#include <random>
// 包含线程操作头文件
#include <thread>
// 包含无序映射容器头文件
#include <unordered_map>
// 包含元组头文件
#include <tuple>

// 如果是 Windows 系统，则执行以下代码
#ifdef _WIN32
// 定义 DIRECTORY_SEPARATOR 为反斜杠或者斜杠，根据操作系统不同进行选择
#ifdef _WIN32
#define DIRECTORY_SEPARATOR '\\'
#else
#define DIRECTORY_SEPARATOR '/'
#endif // _WIN32

// 定义 die 函数，输出错误信息并退出程序
#define die(msg)          do { fputs("error: " msg "\n", stderr);                exit(1); } while (0)
// 定义 die_fmt 函数，输出格式化的错误信息并退出程序
#define die_fmt(fmt, ...) do { fprintf(stderr, "error: " fmt "\n", __VA_ARGS__); exit(1); } while (0)

// 定义 print_build_info 函数，输出构建信息
#define print_build_info() do {                                                                     \
    fprintf(stderr, "%s: build = %d (%s)\n", __func__, LLAMA_BUILD_NUMBER, LLAMA_COMMIT);           \
    fprintf(stderr, "%s: built with %s for %s\n", __func__, LLAMA_COMPILER, LLAMA_BUILD_TARGET);    \
} while(0)

// 声明构建信息的外部变量
extern int LLAMA_BUILD_NUMBER;
extern char const *LLAMA_COMMIT;
extern char const *LLAMA_COMPILER;
extern char const *LLAMA_BUILD_TARGET;

//
// CLI argument parsing
// 命令行参数解析

// 获取物理核心数
int32_t get_num_physical_cores();

// 定义结构体 gpt_params，包含各种参数设置
struct gpt_params {
    uint32_t seed                           = -1;    // 随机数种子

    int32_t n_threads                       = get_num_physical_cores();  // 线程数
    int32_t n_threads_batch                 = -1;    // 用于批处理的线程数 (-1 = 使用 n_threads)
    int32_t n_predict                       = -1;    // 预测的新标记数
    int32_t n_ctx                           = 512;   // 上下文大小
    int32_t n_batch                         = 32;    // 用于提示处理的批处理大小 (必须 >=32 以使用 BLAS)
    int32_t n_keep                          = 0;     // 从初始提示中保留的标记数
    int32_t n_draft                         = 16;    // 在推测解码期间起草的标记数
    int32_t n_chunks                        = -1;    // 要处理的最大块数 (-1 = 无限制)
    int32_t n_parallel                      = 1;     // 要解码的并行序列数
    int32_t n_sequences                     = 1;     // 要解码的序列数
    float   p_accept                        = 0.5f;  // 接受推测解码的概率
    float   p_split                         = 0.1f;  // 分割推测解码的概率
    int32_t n_gpu_layers                    = -1;    // 在 VRAM 中存储的层数 (-1 - 使用默认值)
    int32_t n_gpu_layers_draft              = -1;    // 存储在VRAM中的草稿模型层数（-1 - 使用默认值）
    int32_t main_gpu                        = 0;     // 用于临时和小张量的GPU
    float   tensor_split[LLAMA_MAX_DEVICES] = {0};   // 张量在多个GPU上的分布比例
    int32_t n_beams                         = 0;     // 如果非零，则使用给定宽度的波束搜索。
    float   rope_freq_base                  = 0.0f;  // RoPE基础频率
    float   rope_freq_scale                 = 0.0f;  // RoPE频率缩放因子
    float   vram_budget_gb                  = -1.0f; // VRAM预算（GB）（-1 - 使用可用的VRAM）
    float   yarn_ext_factor                 = -1.0f; // YaRN外推混合因子
    float   yarn_attn_factor                = 1.0f;  // YaRN幅度缩放因子
    float   yarn_beta_fast                  = 32.0f; // YaRN低校正维度
    float   yarn_beta_slow                  = 1.0f;  // YaRN高校正维度
    int32_t yarn_orig_ctx                   = 0;     // YaRN原始上下文长度
    int8_t  rope_scaling_type               = LLAMA_ROPE_SCALING_UNSPECIFIED; // TODO: 最好是int32_t以便对齐
                                                                              //       询问 @cebtenzzre

    // // 采样参数
    struct llama_sampling_params sparams;

    std::string model             = "models/7B/ggml-model-f16.gguf"; // 模型路径
    std::string model_draft       = "";                              // 用于推测解码的草稿模型
    std::string model_alias       = "unknown"; // 模型别名
    std::string prompt            = "";        // 提示信息
    std::string prompt_file       = "";        // 存储外部提示文件名
    std::string path_prompt_cache = "";        // 保存/加载提示评估状态的文件路径
    std::string input_prefix      = "";        // 用户输入的前缀字符串
    std::string input_suffix      = "";        // 用户输入的后缀字符串
    std::vector<std::string> antiprompt;      // 触发更多用户输入的字符串
    std::string logdir            = "";        // 保存 YAML 日志文件的目录

    // TODO: 避免使用元组，使用结构体
    std::vector<std::tuple<std::string, float>> lora_adapter; // 具有用户定义比例的 lora 适配器路径
    std::string lora_base  = "";                              // lora 适配器的基础模型路径

    bool reset_gpu_index   = false; // 刷新 GPU 索引文件
    bool disale_gpu_index  = false; // 禁用加载 GPU 索引和分割 ffn

    int  ppl_stride        = 0;     // 困惑度计算的步长。如果保留为 0，则将使用预先存在的方法。
    int  ppl_output_type   = 0;     // = 0 -> 困惑度输出与通常一样，= 1 -> 困惑度输出为 num_tokens，每行一个
                                    //                                       （这对于绘图更方便）
    // 是否计算 HellaSwag 分数
    bool   hellaswag       = false; // compute HellaSwag score over random tasks from datafile supplied in prompt
    // 用于计算 HellaSwag 分数的任务数量
    size_t hellaswag_tasks = 400;   // number of tasks to use when computing the HellaSwag score

    // 是否使用 mul_mat_q 内核而不是 cuBLAS
    bool mul_mat_q         = true;  // if true, use mul_mat_q kernels instead of cuBLAS
    // 是否使用 f16 代替 f32 作为内存 kv
    bool memory_f16        = true;  // use f16 instead of f32 for memory kv
    // 如果没有提供提示，则不随机化提示
    bool random_prompt     = false; // do not randomize prompt if none provided
    // 是否使用颜色区分生成和输入
    bool use_color         = false; // use color to distinguish generations and inputs
    // 是否交互模式
    bool interactive       = false; // interactive mode
    // 是否将用户输入和生成保存到提示缓存
    bool prompt_cache_all  = false; // save user input and generations to prompt cache
    // 以只读方式打开提示缓存，不更新它
    bool prompt_cache_ro   = false; // open the prompt cache read-only and do not update it

    // 是否仅获取句子嵌入
    bool embedding         = false; // get only sentence embedding
    // 转义 "\n", "\r", "\t", "\'", "\"", 和 "\\"
    bool escape            = false; // escape "\n", "\r", "\t", "\'", "\"", and "\\"
    // 立即等待用户输入
    bool interactive_first = false; // wait for user input immediately
    // 反转使用 `\` 的方式
    bool multiline_input   = false; // reverse the usage of `\`
    // 提高与子进程和有限控制台的兼容性
    bool simple_io         = false; // improves compatibility with subprocesses and limited consoles
    // 实时插入新序列以进行解码
    bool cont_batching     = false; // insert new sequences for decoding on-the-fly

    // 在用户输入前添加 BOS，即 input_prefix
    bool input_prefix_bos  = false; // prefix BOS to user inputs, preceding input_prefix
    // 忽略生成的 EOS 标记
    bool ignore_eos        = false; // ignore generated EOS tokens
// 定义指令模式，用于 Alpaca 模型
bool instruct = false; 
// 返回批处理中所有标记的对数
bool logits_all = false; 
// 使用 mmap 进行更快的加载
bool use_mmap = true;  
// 使用 mlock 保持模型在内存中
bool use_mlock = false; 
// 尝试在一些 NUMA 系统上帮助优化
bool numa = false; 
// 在生成之前打印提示标记
bool verbose_prompt = false; 
// 使用 infill 模式
bool infill = false; 

// 多模态模型（参见 examples/llava）
std::string mmproj = ""; // 多模态投影仪的路径
std::string image = ""; // 图像文件的路径
};

// 解析参数的扩展版本
bool gpt_params_parse_ex(int argc, char ** argv, gpt_params & params);

// 解析参数
bool gpt_params_parse(int argc, char ** argv, gpt_params & params);

// 打印用法
void gpt_print_usage(int argc, char ** argv, const gpt_params & params);

// 获取系统信息
std::string get_system_info(const gpt_params & params);
// 声明一个函数，该函数接受一个 mt19937 类型的引用参数，并返回一个字符串
std::string gpt_random_prompt(std::mt19937 & rng);

// 声明一个函数，该函数接受一个字符串引用参数，并处理其中的转义字符
void process_escapes(std::string& input);

//
// Model utils
//

// TODO: 避免使用元组，使用结构体
// 根据 gpt_params 参数初始化 llama_model 和 llama_context，返回一个包含这两个指针的元组
std::tuple<struct llama_model *, struct llama_context *> llama_init_from_gpt_params(gpt_params & params);

// 根据 gpt_params 参数创建 llama_model_params 结构体
struct llama_model_params   llama_model_params_from_gpt_params  (const gpt_params & params);

// 根据 gpt_params 参数创建 llama_context_params 结构体
struct llama_context_params llama_context_params_from_gpt_params(const gpt_params & params);

// Batch utils

// 清空 llama_batch 结构体
void llama_batch_clear(struct llama_batch & batch);

// 向 llama_batch 结构体中添加数据
void llama_batch_add(
// 定义一个函数，接受一个 llama_batch 结构体的引用，一个 llama_token 类型的 id，一个 llama_pos 类型的 pos，一个 llama_seq_id 类型的向量 seq_ids，一个布尔类型的 logits，并且没有返回值

// 词汇工具
// 将一个字符串分词为一个词汇的向量
// 应该类似于 Python 的 `tokenizer.encode` 函数
// 接受一个 llama_context 结构体的指针 ctx，一个字符串的引用 text，一个布尔类型的 add_bos，一个可选的布尔类型的 special，默认值为 false
std::vector<llama_token> llama_tokenize(
  const struct llama_context * ctx,
           const std::string & text,
                        bool   add_bos,
                        bool   special = false);

// 将一个字符串分词为一个词汇的向量
// 接受一个 llama_model 结构体的指针 model，一个字符串的引用 text，一个布尔类型的 add_bos，一个可选的布尔类型的 special，默认值为 false
std::vector<llama_token> llama_tokenize(
    const struct llama_model * model,
// tokenizes a piece of text into tokens, with options to add beginning of sentence marker and handle special cases
// 参数text为要分词的文本，add_bos表示是否添加句子开头标记，special表示是否处理特殊情况
std::vector<llama_token> llama_tokenize(
        const std::string & text,
                        bool   add_bos,
                        bool   special = false);

// 将一个token转换为一个piece
// 应该类似于Python的`tokenizer.id_to_piece`函数
std::string llama_token_to_piece(
        const struct llama_context * ctx,
                       llama_token   token);

// TODO: 这些应该移动到llama.h的C风格API中，放在一个名为`llama_detokenize`的函数下
//       该函数考虑分词器类型并决定如何处理前导空格
//
// 将一个token向量解码为一个字符串
// 应该类似于Python的`tokenizer.decode`函数
// 从第一个非BOS token中移除前导空格
std::string llama_detokenize_spm(
                         llama_context * ctx,
        const std::vector<llama_token> & tokens);
// 将一个标记向量解码为字符串
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

// 将非结果信息转储为 YAML
void dump_non_result_info_yaml(
    FILE * stream, const gpt_params & params, const llama_context * lctx,
    const std::string & timestamp, const std::vector<int> & prompt_tokens, const char * model_desc);
```