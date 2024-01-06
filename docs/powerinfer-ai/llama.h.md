# `PowerInfer\llama.h`

```
// 如果 LLAMA_H 未定义，则定义 LLAMA_H
#ifndef LLAMA_H
#define LLAMA_H

// 包含 ggml.h 文件
#include "ggml.h"
// 如果 GGML_USE_CUBLAS 已定义，则包含 ggml-cuda.h 文件，并定义 LLAMA_MAX_DEVICES 为 GGML_CUDA_MAX_DEVICES
#ifdef GGML_USE_CUBLAS
#include "ggml-cuda.h"
#define LLAMA_MAX_DEVICES GGML_CUDA_MAX_DEVICES
// 否则，定义 LLAMA_MAX_DEVICES 为 1
#else
#define LLAMA_MAX_DEVICES 1
#endif // GGML_USE_CUBLAS
// 包含标准库头文件
#include <stddef.h>
#include <stdint.h>
#include <stdio.h>
#include <stdbool.h>

// 如果 LLAMA_SHARED 已定义
#ifdef LLAMA_SHARED
// 如果在 Windows 平台且非 MinGW 编译器
#    if defined(_WIN32) && !defined(__MINGW32__)
// 如果 LLAMA_BUILD 已定义，则定义 LLAMA_API 为 __declspec(dllexport)
#        ifdef LLAMA_BUILD
#            define LLAMA_API __declspec(dllexport)
// 否则
# 如果是 Windows 平台，定义 LLAMA_API 为导入 DLL 的声明
# 如果不是 Windows 平台，定义 LLAMA_API 为默认可见性
# 如果不是 Windows 平台，定义 LLAMA_API 为空

# 如果是 GCC 编译器，定义 DEPRECATED 宏为带有弃用提示的函数声明
# 如果是 MSC 编译器，定义 DEPRECATED 宏为带有弃用提示的函数声明
# 如果不是以上两种编译器，定义 DEPRECATED 宏为普通函数声明

# 定义 LLAMA_DEFAULT_SEED 为默认种子值

# 定义 LLAMA_MAX_RNG_STATE 为随机数生成器状态的最大值
// 定义 LLAMA_FILE_MAGIC_GGSN 为十六进制数 0x6767736e，即 'ggsn' 的 ASCII 码
#define LLAMA_FILE_MAGIC_GGSN 0x6767736eu // 'ggsn'

// 定义 LLAMA_SESSION_MAGIC 为 LLAMA_FILE_MAGIC_GGSN
#define LLAMA_SESSION_MAGIC   LLAMA_FILE_MAGIC_GGSN
// 定义 LLAMA_SESSION_VERSION 为 2
#define LLAMA_SESSION_VERSION 2

// 如果定义了 GGML_USE_CUBLAS 或 GGML_USE_CLBLAST 或 GGML_USE_METAL，则定义 LLAMA_SUPPORTS_GPU_OFFLOAD
#if defined(GGML_USE_CUBLAS) || defined(GGML_USE_CLBLAST) || defined(GGML_USE_METAL)
// Defined when llama.cpp is compiled with support for offloading model layers to GPU.
#define LLAMA_SUPPORTS_GPU_OFFLOAD
#endif

#ifdef __cplusplus
extern "C" {
#endif

    //
    // C interface
    //
    // TODO: show sample usage
    //
```

    // 定义名为 llama_model 的结构体
    struct llama_model;
    // 定义名为 llama_context 的结构体
    struct llama_context;

    // 定义名为 llama_pos 的类型为 int32_t
    typedef int32_t llama_pos;
    // 定义名为 llama_token 的类型为 int32_t
    typedef int32_t llama_token;
    // 定义名为 llama_seq_id 的类型为 int32_t
    typedef int32_t llama_seq_id;

    // 枚举类型，表示词汇表的类型，0 代表 SentencePiece，1 代表 Byte Pair Encoding
    enum llama_vocab_type {
        LLAMA_VOCAB_TYPE_SPM = 0, // SentencePiece
        LLAMA_VOCAB_TYPE_BPE = 1, // Byte Pair Encoding
    };

    // 枚举类型，表示 token 的类型
    enum llama_token_type {
        LLAMA_TOKEN_TYPE_UNDEFINED    = 0, // 未定义
        LLAMA_TOKEN_TYPE_NORMAL       = 1, // 普通类型
        LLAMA_TOKEN_TYPE_UNKNOWN      = 2, // 未知类型
        LLAMA_TOKEN_TYPE_CONTROL      = 3, // 控制类型
        LLAMA_TOKEN_TYPE_USER_DEFINED = 4, // 用户自定义类型
        LLAMA_TOKEN_TYPE_UNUSED       = 5, // 未使用类型
// 定义 LLAMA 令牌类型的枚举，包括字节类型
enum llama_token_type {
    LLAMA_TOKEN_TYPE_BYTE         = 6,
};

// 定义模型文件类型的枚举
enum llama_ftype {
    LLAMA_FTYPE_ALL_F32              = 0,  // 所有数据类型为 F32
    LLAMA_FTYPE_MOSTLY_F16           = 1,  // 大部分数据类型为 F16，除了 1d 张量
    LLAMA_FTYPE_MOSTLY_Q4_0          = 2,  // 大部分数据类型为 Q4_0，除了 1d 张量
    LLAMA_FTYPE_MOSTLY_Q4_1          = 3,  // 大部分数据类型为 Q4_1，除了 1d 张量
    LLAMA_FTYPE_MOSTLY_Q4_1_SOME_F16 = 4,  // tok_embeddings.weight 和 output.weight 类型为 F16
    // LLAMA_FTYPE_MOSTLY_Q4_2       = 5,  // 已移除支持
    // LLAMA_FTYPE_MOSTLY_Q4_3       = 6,  // 已移除支持
    LLAMA_FTYPE_MOSTLY_Q8_0          = 7,  // 大部分数据类型为 Q8_0，除了 1d 张量
    LLAMA_FTYPE_MOSTLY_Q5_0          = 8,  // 大部分数据类型为 Q5_0，除了 1d 张量
    LLAMA_FTYPE_MOSTLY_Q5_1          = 9,  // 大部分数据类型为 Q5_1，除了 1d 张量
    LLAMA_FTYPE_MOSTLY_Q2_K          = 10, // 大部分数据类型为 Q2_K，除了 1d 张量
    LLAMA_FTYPE_MOSTLY_Q3_K_S        = 11, // 大部分数据类型为 Q3_K_S，除了 1d 张量
    LLAMA_FTYPE_MOSTLY_Q3_K_M        = 12, // 大部分数据类型为 Q3_K_M，除了 1d 张量
    LLAMA_FTYPE_MOSTLY_Q3_K_L        = 13, // 大部分数据类型为 Q3_K_L，除了 1d 张量
    LLAMA_FTYPE_MOSTLY_Q4_K_S        = 14, // 大部分数据类型为 Q4_K_S，除了 1d 张量
};
// 定义 LLAMA_FTYPE_MOSTLY_Q4_K_M 的值为 15，表示大多数情况下是四维张量，除了一维张量
// 定义 LLAMA_FTYPE_MOSTLY_Q5_K_S 的值为 16，表示大多数情况下是五维张量，除了一维张量
// 定义 LLAMA_FTYPE_MOSTLY_Q5_K_M 的值为 17，表示大多数情况下是五维张量，除了一维张量
// 定义 LLAMA_FTYPE_MOSTLY_Q6_K 的值为 18，表示大多数情况下是六维张量，除了一维张量
// 定义 LLAMA_FTYPE_GUESSED 的值为 1024，表示在模型文件中未指定

// 定义枚举类型 llama_rope_scaling_type，表示绳索的缩放类型
// LLAMA_ROPE_SCALING_UNSPECIFIED 的值为 -1，表示未指定缩放类型
// LLAMA_ROPE_SCALING_NONE 的值为 0，表示无缩放
// LLAMA_ROPE_SCALING_LINEAR 的值为 1，表示线性缩放
// LLAMA_ROPE_SCALING_YARN 的值为 2，表示纱线缩放
// LLAMA_ROPE_SCALING_MAX_VALUE 的值为 LLAMA_ROPE_SCALING_YARN，表示最大的缩放值为纱线缩放

// 定义结构体 llama_token_data，表示 LLAMA 令牌的数据
// llama_token id 表示令牌的 ID
// float logit 表示令牌的对数几率
// float p 表示令牌的概率
    // 定义了一个名为 llama_token_data 的结构体，用于存储关于 llamanet 模型的 token 数据
    } llama_token_data;

    // 定义了一个名为 llama_token_data_array 的结构体，用于存储 llamanet 模型的 token 数据数组
    typedef struct llama_token_data_array {
        llama_token_data * data; // 指向 token 数据的指针
        size_t size; // 数组的大小
        bool sorted; // 标志数组是否已排序
    } llama_token_data_array;

    // 定义了一个名为 llama_progress_callback 的函数指针类型，用于表示进度回调函数
    typedef void (*llama_progress_callback)(float progress, void *ctx);

    // llama_decode 函数的输入数据
    // 一个 llama_batch 对象可以包含一个或多个序列的输入数据
    // 提供的数组（例如 token, embd, pos 等）必须具有 n_tokens 大小
    //
    // - token  : 输入的 token id（当 embd 为 NULL 时使用）
    // - embd   : token 的嵌入（即大小为 n_embd 的浮点向量）（当 token 为 NULL 时使用）
    // - pos    : 序列中各个 token 的位置
    // - seq_id : 各个 token 所属的序列
    // - logits : 如果为零，则不会输出相应 token 的 logits
    //
// 定义了一个名为 llama_batch 的结构体，用于存储一批数据
typedef struct llama_batch {
    int32_t n_tokens;  // 用于存储数据中的 token 数量

    llama_token  *  token;  // 指向存储 token 的指针
    float        *  embd;   // 指向存储嵌入向量的指针
    llama_pos    *  pos;    // 指向存储位置信息的指针
    int32_t      *  n_seq_id;  // 指向存储序列 ID 数量的指针
    llama_seq_id ** seq_id;  // 指向存储序列 ID 的指针的指针
    int8_t       *  logits;  // 指向存储逻辑值的指针

    // 注意：用于平滑 API 过渡的辅助字段 - 可以在将来被弃用
    //       为了未来的代码兼容性，请使用上面的字段，忽略下面的内容
    //
    // pos[i] = all_pos_0 + i*all_pos_1
    //
    llama_pos    all_pos_0;  // 如果 pos == NULL，则使用该字段
    llama_pos    all_pos_1;  // 如果 pos == NULL，则使用该字段
    llama_seq_id all_seq_id; // 如果 seq_id == NULL，则使用该字段
} llama_batch;
    // 定义了一个结构体 llama_model_params，用于存储模型参数
    struct llama_model_params {
        // 存储在 VRAM 中的层数
        int32_t n_gpu_layers;
        // 用于临时和小张量的 GPU
        int32_t main_gpu;
        // VRAM 预算（以 GB 为单位），-1 表示所有可用的 VRAM（针对单个 GPU）
        float vram_budget_gb;
        // 用于将层分割到多个 GPU 上的数组指针（大小为 LLAMA_MAX_DEVICES）

        // 用于传递进度值（介于 0 和 1 之间），传递 NULL 表示禁用
        llama_progress_callback progress_callback;
        // 传递给进度回调的上下文指针
        void * progress_callback_user_data;

        // 将布尔值放在一起，以避免在按值复制时出现错位
        bool vocab_only; // 仅加载词汇表，不加载权重
        bool use_mmap;   // 如果可能，使用 mmap
        bool use_mlock;  // 强制系统将模型保留在 RAM 中
        bool reset_gpu_index; // 强制重置 GPU 索引
        bool disable_gpu_index; // 绕过 GPU 索引和 FFN 分割
    };

    // 定义了一个结构体 llama_context_params
    struct llama_context_params {
        uint32_t seed;              // 随机数生成器的种子，-1表示随机
        uint32_t n_ctx;             // 文本上下文，0表示从模型中获取
        uint32_t n_batch;           // 提示处理的最大批处理大小
        uint32_t n_threads;         // 用于生成的线程数
        uint32_t n_threads_batch;   // 用于批处理的线程数
        int8_t   rope_scaling_type; // RoPE缩放类型，来自`enum llama_rope_scaling_type`

        // 参考：https://github.com/ggerganov/llama.cpp/pull/2054
        float    rope_freq_base;   // RoPE基础频率，0表示从模型中获取
        float    rope_freq_scale;  // RoPE频率缩放因子，0表示从模型中获取
        float    yarn_ext_factor;  // YaRN外推混合因子，NaN表示从模型中获取
        float    yarn_attn_factor; // YaRN幅度缩放因子
        float    yarn_beta_fast;   // YaRN低校正维度
        float    yarn_beta_slow;   // YaRN高校正维度
        uint32_t yarn_orig_ctx;    // YaRN原始上下文大小

        // 将布尔值放在一起，以避免在按值复制时出现错误对齐。
        bool mul_mat_q;  // 如果为true，则使用实验性的mul_mat_q内核（已弃用 - 总是为true）
        bool f16_kv;     // 对KV缓存使用fp16，否则使用fp32
        bool logits_all; // llama_eval()调用计算所有logits，而不仅仅是最后一个
    // 嵌入模式标志，仅用于嵌入模式
    bool embedding;  

    // 模型量化参数
    typedef struct llama_model_quantize_params {
        // 用于量化的线程数，如果<=0，则使用std::thread::hardware_concurrency()
        int nthread;                 
        // 量化到这种llama_ftype
        enum llama_ftype ftype;      
        // 允许量化非f32/f16张量
        bool allow_requantize;       
        // 量化输出权重张量
        bool quantize_output_tensor; 
        // 仅复制张量 - ftype，allow_requantize和quantize_output_tensor将被忽略
        bool only_copy;              
        // 纯量化，禁用k-quant混合，并将所有张量量化为相同类型
        bool pure;                   
    } llama_model_quantize_params;

    // 语法类型
    struct llama_grammar;

    // 语法元素类型
    enum llama_gretype {
        // 规则定义结束
        LLAMA_GRETYPE_END            = 0,
// 定义规则的替代定义的开始
LLAMA_GRETYPE_ALT = 1,

// 非终结符元素：对规则的引用
LLAMA_GRETYPE_RULE_REF = 2,

// 终结符元素：字符（码点）
LLAMA_GRETYPE_CHAR = 3,

// 反向字符（[^a], [^a-b] [^abc]）
LLAMA_GRETYPE_CHAR_NOT = 4,

// 修改前面的LLAMA_GRETYPE_CHAR或LLAMA_GRETYPE_CHAR_ALT，使其成为包含范围（[a-z]）
LLAMA_GRETYPE_CHAR_RNG_UPPER = 5,

// 修改前面的LLAMA_GRETYPE_CHAR或LLAMA_GRETYPE_CHAR_RNG_UPPER，添加一个替代字符进行匹配（[ab], [a-zA]）
LLAMA_GRETYPE_CHAR_ALT = 6,
    };

    // 定义 llmama_grammar_element 结构体，包含类型和值
    typedef struct llama_grammar_element {
        enum llama_gretype type; // 元素类型
        uint32_t           value; // Unicode 码点或规则 ID
    } llama_grammar_element;

    // 性能计时信息
    struct llama_timings {
        double t_start_ms; // 开始时间
        double t_end_ms; // 结束时间
        double t_load_ms; // 加载时间
        double t_sample_ms; // 采样时间
        double t_p_eval_ms; // 预测评估时间
        double t_eval_ms; // 评估时间

        int32_t n_sample; // 采样次数
        int32_t n_p_eval; // 预测评估次数
        int32_t n_eval; // 评估次数
    };
// 获取默认参数的辅助函数
LLAMA_API struct llama_model_params llama_model_default_params(void);
LLAMA_API struct llama_context_params llama_context_default_params(void);
LLAMA_API struct llama_model_quantize_params llama_model_quantize_default_params(void);

// 初始化llama + ggml后端
// 如果numa为true，则使用NUMA优化
// 在程序开始时调用一次
LLAMA_API void llama_backend_init(bool numa);

// 在程序结束时调用一次 - 目前仅用于MPI
LLAMA_API void llama_backend_free(void);

// 从文件加载模型
LLAMA_API struct llama_model * llama_load_model_from_file(
                         const char * path_model,
        struct llama_model_params     params);

// 释放模型内存
LLAMA_API void llama_free_model(struct llama_model * model);
// 创建一个新的 LLAMA 上下文，使用给定的模型和参数
LLAMA_API struct llama_context * llama_new_context_with_model(
                 struct llama_model * model,
        struct llama_context_params   params);

// 释放所有分配的内存
LLAMA_API void llama_free(struct llama_context * ctx);

// 返回当前时间的微秒数
LLAMA_API int64_t llama_time_us(void);

// 返回 LLAMA 支持的最大设备数
LLAMA_API int  llama_max_devices    (void);
// 返回是否支持 LLAMA 内存映射
LLAMA_API bool llama_mmap_supported (void);
// 返回是否支持 LLAMA 内存锁定
LLAMA_API bool llama_mlock_supported(void);

// 返回 LLAMA 上下文所使用的模型
LLAMA_API const struct llama_model * llama_get_model(const struct llama_context * ctx);

// 返回 LLAMA 上下文的数量
LLAMA_API int llama_n_ctx      (const struct llama_context * ctx);

// 返回给定模型的词汇表类型
LLAMA_API enum llama_vocab_type llama_vocab_type(const struct llama_model * model);
// 返回给定模型是否使用稀疏推断
LLAMA_API bool llama_use_sparse_inference(const struct llama_model * model);
// 获取模型的词汇量
LLAMA_API int llama_n_vocab    (const struct llama_model * model);

// 获取模型的上下文训练数量
LLAMA_API int llama_n_ctx_train(const struct llama_model * model);

// 获取模型的嵌入数量
LLAMA_API int llama_n_embd     (const struct llama_model * model);

// 获取模型的 RoPE 频率缩放因子
LLAMA_API float llama_rope_freq_scale_train(const struct llama_model * model);

// 获取描述模型类型的字符串
LLAMA_API int llama_model_desc(const struct llama_model * model, char * buf, size_t buf_size);

// 返回模型中所有张量的总大小（以字节为单位）
LLAMA_API uint64_t llama_model_size(const struct llama_model * model);

// 返回模型中的参数总数
LLAMA_API uint64_t llama_model_n_params(const struct llama_model * model);

// 获取 llama 模型张量
LLAMA_API struct ggml_tensor * llama_get_model_tensor(struct llama_model * model, const char * name);

// 成功时返回 0
// 定义了一个名为 llama_model_quantize 的函数，用于对模型进行量化处理
// fname_inp 是输入文件的路径
// fname_out 是输出文件的路径
// params 是量化参数
LLAMA_API int llama_model_quantize(
        const char * fname_inp,
        const char * fname_out,
        const llama_model_quantize_params * params);

// 对加载的模型应用 LoRA 适配器
// path_base_model 是用作基础模型的高质量模型的路径，用于修改适配器修改的层。如果为NULL，则使用当前加载的模型。
// 在应用新的适配器之前，需要重新加载模型，否则适配器将被应用在之前的适配器之上
// 成功时返回0
LLAMA_API DEPRECATED(int llama_apply_lora_from_file(
        struct llama_context * ctx,
                  const char * path_lora,
                       float   scale,
                  const char * path_base_model,
                         int   n_threads),
        "use llama_model_apply_lora_from_file instead");

// 对模型应用 LoRA 适配器
// ctx 是上下文
// path_lora 是 LoRA 文件的路径
// scale 是缩放比例
// path_base_model 是基础模型的路径
// n_threads 是线程数
LLAMA_API int llama_model_apply_lora_from_file(
// 声明一个函数，接受一个llama_model结构体指针，一个指向char类型的路径的指针，一个浮点数scale，一个指向char类型的基本模型路径的指针，一个整数n_threads
LLAMA_API int llama_model_apply_gpu_idx_from_file(
                  struct llama_model * model,
                          const char * path_mlp,
                                bool   use_mmap);

// 声明一个函数，接受一个llama_model结构体指针，一个指向char类型的路径的指针，一个布尔值use_mmap
LLAMA_API int llama_model_apply_gpu_idx_from_file(
                  struct llama_model * model,
                          const char * path_mlp,
                                bool   use_mmap);

// 声明一个函数，接受一个llama_model结构体指针，返回一个size_t类型的值
LLAMA_API size_t llama_model_offload_ffn_split(struct llama_model * model);

// 返回KV缓存中的令牌数
LLAMA_API DEPRECATED(int llama_get_kv_cache_token_count(const struct llama_context * ctx),
            "avoid using this, it will be removed in the future, instead - count the tokens in user code");
    // 清空 KV 缓存
    LLAMA_API void llama_kv_cache_clear(
            struct llama_context * ctx);

    // 移除所有属于指定序列并且位置在 [p0, p1) 的标记
    // seq_id < 0 : 匹配任意序列
    // p0 < 0     : [0,  p1]
    // p1 < 0     : [p0, 无穷大)
    LLAMA_API void llama_kv_cache_seq_rm(
            struct llama_context * ctx,
                    llama_seq_id   seq_id,
                       llama_pos   p0,
                       llama_pos   p1);

    // 将所有属于指定序列的标记复制到另一个序列
    // 注意，这不会分配额外的 KV 缓存内存 - 它只是将标记分配给新序列
    // p0 < 0 : [0,  p1]
    // p1 < 0 : [p0, 无穷大)
    LLAMA_API void llama_kv_cache_seq_cp(
// 声明一个函数 llama_kv_cache_seq_remove，接受一个 llama_context 结构体指针，两个 llama_seq_id 类型的参数，以及两个 llama_pos 类型的参数
// 该函数用于移除所有不属于指定序列的令牌
LLAMA_API void llama_kv_cache_seq_remove(
        struct llama_context * ctx,
                llama_seq_id   seq_id_src,
                llama_seq_id   seq_id_dst,
                   llama_pos   p0,
                   llama_pos   p1);

// 声明一个函数 llama_kv_cache_seq_keep，接受一个 llama_context 结构体指针和一个 llama_seq_id 类型的参数
// 该函数用于保留属于指定序列的所有令牌，移除其他令牌
LLAMA_API void llama_kv_cache_seq_keep(
        struct llama_context * ctx,
                llama_seq_id   seq_id);

// 声明一个函数 llama_kv_cache_seq_shift，接受一个 llama_context 结构体指针，一个 llama_seq_id 类型的参数，以及两个 llama_pos 类型的参数
// 该函数用于将属于指定序列且位置在 [p0, p1) 范围内的所有令牌的相对位置 "delta" 添加到令牌上
// 如果 KV 缓存是 RoPEd，那么 KV 数据也会相应更新
// 当 p0 < 0 时，表示范围为 [0, p1]
// 当 p1 < 0 时，表示范围为 [p0, inf)
LLAMA_API void llama_kv_cache_seq_shift(
        struct llama_context * ctx,
                llama_seq_id   seq_id,
                   llama_pos   p0,
                   llama_pos   p1,
    // 定义了一个函数 llama_get_state_size，用于返回状态（rng、logits、embedding和kv_cache）的最大大小，通常在压缩令牌后会更小
    LLAMA_API size_t llama_get_state_size(const struct llama_context * ctx);

    // 定义了一个函数 llama_copy_state_data，用于将状态数据复制到指定的目标地址，目标地址需要分配足够的内存，返回复制的字节数
    LLAMA_API size_t llama_copy_state_data(
            struct llama_context * ctx,
                         uint8_t * dst);

    // 定义了一个函数 llama_set_state_data，用于从指定地址设置状态，返回读取的字节数
    LLAMA_API size_t llama_set_state_data(
    // 定义一个函数，用于加载指定路径的会话文件，将会话文件中的令牌存储到 tokens_out 中
    LLAMA_API bool llama_load_session_file(
            struct llama_context * ctx,  // 指向 llama_context 结构体的指针
                      const char * path_session,  // 会话文件的路径
                     llama_token * tokens_out,  // 存储令牌的数组
                          size_t   n_token_capacity,  // 令牌容量
                          size_t * n_token_count_out);  // 存储实际令牌数量的指针

    // 定义一个函数，用于保存指定路径的会话文件，将 tokens 中的令牌存储到会话文件中
    LLAMA_API bool llama_save_session_file(
            struct llama_context * ctx,  // 指向 llama_context 结构体的指针
                      const char * path_session,  // 会话文件的路径
               const llama_token * tokens,  // 要保存的令牌数组
                          size_t   n_token_count);  // 令牌数量

    //
    // 解码
    //
// 运行羊驼推断以获取下一个标记的对数和概率。
// tokens + n_tokens 是要处理的新标记的提供批处理
// n_past 是要使用的先前评估调用的标记数
// 成功时返回0
// 已弃用：请改用llama_decode()代替
LLAMA_API DEPRECATED(int llama_eval(
        struct llama_context * ctx,
                 llama_token * tokens,
                     int32_t   n_tokens,
                         int   n_past),
        "use llama_decode() instead");

// 与llama_eval相同，但直接使用浮点矩阵输入。
// 已弃用：请改用llama_decode()代替
LLAMA_API DEPRECATED(int llama_eval_embd(
        struct llama_context * ctx,
                       float * embd,
                     int32_t   n_tokens,
                         int   n_past),
    // 使用 llama_decode() 替代此函数
    "use llama_decode() instead");

    // 返回从 pos_0 开始的单个令牌序列的批处理
    //
    // 注意：这是一个辅助函数，用于便于过渡到新的批处理 API - 避免使用它
    //
    LLAMA_API struct llama_batch llama_batch_get_one(
                  llama_token * tokens,
                      int32_t   n_tokens,
                    llama_pos   pos_0,
                 llama_seq_id   seq_id);

    // 在堆上分配一个最大可容纳 n_tokens 的令牌批处理
    // 每个令牌最多可以分配给 n_seq_max 个序列 id
    // 批处理必须使用 llama_batch_free() 释放
    // 如果 embd != 0，则 llama_batch.embd 将分配大小为 n_tokens * embd * sizeof(float)
    // 否则，llama_batch.token 将分配大小以存储 n_tokens 个 llama_token
    // 其余的 llama_batch 成员将分配大小为 n_tokens
    // 所有成员都将保持未初始化状态
    LLAMA_API struct llama_batch llama_batch_init(
// 定义了一个函数，用于初始化一个批量的 tokens，n_tokens 表示 tokens 的数量，embd 表示嵌入的维度，n_seq_max 表示最大序列长度
LLAMA_API struct llama_batch llama_batch_init(
            int32_t n_tokens,
            int32_t embd,
            int32_t n_seq_max);

// 释放使用 llama_batch_init() 分配的一批 tokens
LLAMA_API void llama_batch_free(struct llama_batch batch);

// 正数的返回值并不意味着致命错误，而是一个警告。
//   0 - 成功
//   1 - 无法为批量找到 KV 槽位（尝试减少批量大小或增加上下文）
// < 0 - 错误
LLAMA_API int llama_decode(
            struct llama_context * ctx,
              struct llama_batch   batch);

// 设置用于解码的线程数
// n_threads 是用于生成（单个 token）的线程数
// n_threads_batch 是用于提示和批处理（多个 token）的线程数
LLAMA_API void llama_set_n_threads(struct llama_context * ctx, uint32_t n_threads, uint32_t n_threads_batch);
// 从最后一次调用llama_eval()中获取的token logits
// 最后一个token的logits存储在最后一行
// 当llama_batch.logits[i] == 0时，对应的logits是未定义的
// 行数：llama_batch提供的n_tokens
// 列数：n_vocab
LLAMA_API float * llama_get_logits(struct llama_context * ctx);

// 第i个token的logits。等同于：
// llama_get_logits(ctx) + i*n_vocab
LLAMA_API float * llama_get_logits_ith(struct llama_context * ctx, int32_t i);

// 获取输入的嵌入
// 形状：[n_embd]（一维）
LLAMA_API float * llama_get_embeddings(struct llama_context * ctx);

//
// 词汇表
//

LLAMA_API const char * llama_token_get_text(const struct llama_model * model, llama_token token);
// 获取指定模型和令牌的得分
LLAMA_API float llama_token_get_score(const struct llama_model * model, llama_token token);

// 获取指定模型和令牌的类型
LLAMA_API enum llama_token_type llama_token_get_type(const struct llama_model * model, llama_token token);

// 特殊令牌
LLAMA_API llama_token llama_token_bos(const struct llama_model * model); // 句子开头
LLAMA_API llama_token llama_token_eos(const struct llama_model * model); // 句子结尾
LLAMA_API llama_token llama_token_nl (const struct llama_model * model); // 换行

// codellama infill 令牌
LLAMA_API llama_token llama_token_prefix(const struct llama_model * model); // 开始填充前缀
LLAMA_API llama_token llama_token_middle(const struct llama_model * model); // 开始填充中间部分
LLAMA_API llama_token llama_token_suffix(const struct llama_model * model); // 开始填充后缀
LLAMA_API llama_token llama_token_eot   (const struct llama_model * model); // 结束填充中间部分

//
// 令牌化
//
    /// @details Convert the provided text into tokens.
    /// @param tokens The tokens pointer must be large enough to hold the resulting tokens.
    /// @return Returns the number of tokens on success, no more than n_max_tokens
    /// @return Returns a negative number on failure - the number of tokens that would have been returned
    /// @param special Allow tokenizing special and/or control tokens which otherwise are not exposed and treated as plaintext.
    ///                Does not insert a leading space.
    LLAMA_API int llama_tokenize(
        const struct llama_model * model,  // 指向 llama_model 结构的指针
                      const char * text,  // 要转换为标记的文本
                             int   text_len,  // 文本的长度
                     llama_token * tokens,  // 用于存储结果标记的指针，必须足够大
                             int   n_max_tokens,  // 最大标记数
                            bool   add_bos,  // 是否添加 BOS（Beginning of Sentence）标记
                            bool   special);  // 是否允许标记化特殊和/或控制标记，否则将被视为纯文本。不插入前导空格。

    // Token Id -> Piece.
    // Uses the vocabulary in the provided context.
    // Does not write null terminator to the buffer.
    // User code is responsible to remove the leading whitespace of the first non-BOS token when decoding multiple tokens.
    LLAMA_API int llama_token_to_piece(
// 声明一个函数，接受一个指向 llama_model 结构体的指针，一个 llama_token 类型的参数，一个指向字符数组的指针，一个整型参数
void llama_model_generate(const struct llama_model * model,
                           llama_token   token,
                                  char * buf,
                                  int    length);

//
// 语法
//

// 初始化一个 llama_grammar 结构体，接受一个指向 llama_grammar_element 指针数组的指针，一个 size_t 类型的参数，一个表示起始规则索引的 size_t 类型参数
LLAMA_API struct llama_grammar * llama_grammar_init(
            const llama_grammar_element ** rules,
                                 size_t    n_rules,
                                 size_t    start_rule_index);

// 释放一个 llama_grammar 结构体的内存
LLAMA_API void llama_grammar_free(struct llama_grammar * grammar);

// 复制一个 llama_grammar 结构体，接受一个指向 llama_grammar 结构体的指针
LLAMA_API struct llama_grammar * llama_grammar_copy(const struct llama_grammar * grammar);

//
// 采样函数
    // 设置当前的随机数种子
    LLAMA_API void llama_set_rng_seed(struct llama_context * ctx, uint32_t seed);

    /// @details 在 CTRL 学术论文 https://arxiv.org/abs/1909.05858 中描述的重复惩罚，带有负 logit 修复。
    /// @details 在 OpenAI API https://platform.openai.com/docs/api-reference/parameter-details 中描述的频率和存在惩罚。
    LLAMA_API void llama_sample_repetition_penalties(
            struct llama_context * ctx,
            llama_token_data_array * candidates,
            const llama_token * last_tokens,
            size_t   penalty_last_n,
            float   penalty_repeat,
            float   penalty_freq,
            float   penalty_present);

    /// @details 根据学术论文 "Stay on topic with Classifier-Free Guidance" https://arxiv.org/abs/2306.17806 中描述的方法，对 logits 应用无分类器的指导
    /// @param candidates 包含候选 token 的 `llama_token_data` 向量，logits 必须直接从原始生成上下文中提取，而不进行排序。
    /// @params guidance_ctx 与模型相同的另一个上下文。除了开头的负提示之外，它应该包含从主上下文中复制的所有生成和用户输入的 token。
    /// @params scale 指导强度。1.0f 表示没有指导。更高的值表示更强的指导。
    // 释放指导上下文中的样本分类器
    LLAMA_API void llama_sample_classifier_free_guidance(
              struct llama_context * ctx,
            llama_token_data_array * candidates,
              struct llama_context * guidance_ctx,
                             float   scale);

    /// @details 按照logits降序对候选token进行排序，并基于logits计算概率
    LLAMA_API void llama_sample_softmax(
            struct llama_context * ctx,
          llama_token_data_array * candidates);

    /// @details 在学术论文"The Curious Case of Neural Text Degeneration"中描述的Top-K采样 https://arxiv.org/abs/1904.09751
    LLAMA_API void llama_sample_top_k(
            struct llama_context * ctx,
          llama_token_data_array * candidates,
                             int   k,
                          size_t   min_keep);

    /// @details 在学术论文"The Curious Case of Neural Text Degeneration"中描述的Nucleus采样 https://arxiv.org/abs/1904.09751
    LLAMA_API void llama_sample_top_p(
// llama_sample_min_p函数的作用是根据最小P采样，具体描述可以参考链接https://github.com/ggerganov/llama.cpp/pull/3841
LLAMA_API void llama_sample_min_p(
        struct llama_context * ctx,
        llama_token_data_array * candidates,
        float   p,
        size_t   min_keep);

// llama_sample_tail_free函数的作用是进行尾部免费采样，具体描述可以参考链接https://www.trentonbricken.com/Tail-Free-Sampling/
LLAMA_API void llama_sample_tail_free(
        struct llama_context * ctx,
        llama_token_data_array * candidates,
        float   z,
        size_t   min_keep);

// 该函数是局部典型采样的实现，具体描述可以参考论文https://arxiv.org/abs/2202.00666
LLAMA_API void llama_sample_tail_free(
        struct llama_context * ctx,
        llama_token_data_array * candidates,
        float   z,
        size_t   min_keep);
// 定义名为 llama_sample_typical 的函数，接受 llama_context 结构体指针、llama_token_data_array 指针、float 类型的 p 和 size_t 类型的 min_keep 参数
LLAMA_API void llama_sample_typical(
        struct llama_context * ctx,
        llama_token_data_array * candidates,
        float   p,
        size_t   min_keep);

// 定义名为 llama_sample_temp 的函数，接受 llama_context 结构体指针、llama_token_data_array 指针和 float 类型的 temp 参数
LLAMA_API void llama_sample_temp(
        struct llama_context * ctx,
        llama_token_data_array * candidates,
        float   temp);

// 定义名为 llama_sample_temperature 的函数，接受 llama_context 结构体指针、llama_token_data_array 指针和 float 类型的 temp 参数，同时标记为已废弃，并提供替代函数 llama_sample_temp
LLAMA_API DEPRECATED(void llama_sample_temperature(
            struct llama_context * ctx,
            llama_token_data_array * candidates,
            float   temp),
        "use llama_sample_temp instead");

// 定义名为 llama_sample_grammar 的函数，接受 llama_context 结构体指针
/// @details Apply constraints from grammar
LLAMA_API void llama_sample_grammar(
        struct llama_context * ctx,
// 声明一个函数 llama_sample_token_mirostat，该函数使用 Mirostat 1.0 算法，接受一个 llama_context 结构体指针，一个 llama_token_data_array 结构体指针，以及一些浮点数参数
/// @details 详细描述了使用 Mirostat 1.0 算法的函数，可以在 https://arxiv.org/abs/2007.14966 找到相关论文。该算法使用 tokens 而不是 words。
/// @param candidates 包含候选 tokens、它们的概率 (p) 以及在生成文本中当前位置的 log-odds (logit) 的 `llama_token_data` 向量
LLAMA_API llama_token llama_sample_token_mirostat(
        struct llama_context * ctx,
      llama_token_data_array * candidates,
                       float   tau,
                       float   eta,
                         int   m,
                       float * mu);

// 声明一个函数 llama_sample_token_mirostat_v2，该函数使用 Mirostat 2.0 算法，接受一个 llama_context 结构体指针，一个 llama_token_data_array 结构体指针，以及一些浮点数参数
/// @details 详细描述了使用 Mirostat 2.0 算法的函数，可以在 https://arxiv.org/abs/2007.14966 找到相关论文。该算法使用 tokens 而不是 words。
/// @param candidates 包含候选 tokens、它们的概率 (p) 以及在生成文本中当前位置的 log-odds (logit) 的 `llama_token_data` 向量
LLAMA_API llama_token llama_sample_token_mirostat_v2(
        struct llama_context * ctx,
      llama_token_data_array * candidates,
                       float   tau,
                       float   eta,
// 声明一个函数，用于从候选项中选择具有最高概率的标记
// 不计算标记概率。应该使用 llama_sample_softmax() 函数代替
LLAMA_API llama_token llama_sample_token_greedy(
        struct llama_context * ctx,
        llama_token_data_array * candidates);

// 声明一个函数，根据候选项的概率随机选择一个标记
LLAMA_API llama_token llama_sample_token(
        struct llama_context * ctx,
        llama_token_data_array * candidates);

// 声明一个函数，将选定的标记接受到语法中
LLAMA_API void llama_grammar_accept_token(
        struct llama_context * ctx,
        struct llama_grammar * grammar,
        llama_token   token);
    // Beam search
    // 

    // 定义了一个结构体 llama_beam_view，用于表示每个 beam 的信息
    struct llama_beam_view {
        const llama_token * tokens;  // 指向 tokens 的指针
        size_t n_tokens;             // tokens 的数量
        float  p;                    // 累积 beam 概率（相对于所有 beam 进行重新归一化）
        bool   eob;                  // 当一个 beam 到达结束时，回调函数应该将此设置为 true。
    };

    // 传递给 beam_search_callback 函数的结构体
    // 每当 0 < common_prefix_length 时，应该从任何 beam（例如 beams[0]）中复制这些数量的 tokens，
    // 因为它们将从所有后续回调中的所有 beam 中被移除（shifted）。
    // 这些指针仅在同步回调期间有效，因此不应该保存。
    struct llama_beams_state {
        struct llama_beam_view * beam_views;  // 指向 beam_views 的指针
        size_t n_beams;                       // beam_views[] 中的元素数量
        size_t common_prefix_length;          // 当前所有 beam 共享的前缀 tokens 的最大长度
    };
        bool   last_call;             // True iff this is the last callback invocation. 布尔值，表示这是否是最后一次回调调用。

    };

    // Type of pointer to the beam_search_callback function.
    // void* callback_data is any custom data passed to llama_beam_search, that is subsequently
    // passed back to beam_search_callback. This avoids having to use global variables in the callback.
    typedef void (*llama_beam_search_callback_fn_t)(void * callback_data, struct llama_beams_state);
    // 指向beam_search_callback函数的指针类型。
    // void* callback_data是传递给llama_beam_search的任何自定义数据，随后传递回beam_search_callback。这避免了在回调中使用全局变量。

    /// @details Deterministically returns entire sentence constructed by a beam search.
    /// @param ctx Pointer to the llama_context. 指向llama_context的指针。
    /// @param callback Invoked for each iteration of the beam_search loop, passing in beams_state. 在beam_search循环的每次迭代中调用，传入beams_state。
    /// @param callback_data A pointer that is simply passed back to callback. 一个简单地传递回调的指针。
    /// @param n_beams Number of beams to use. 要使用的beam数量。
    /// @param n_past Number of tokens already evaluated. 已经评估的标记数量。
    /// @param n_predict Maximum number of tokens to predict. EOS may occur earlier. 要预测的最大标记数量。EOS可能会更早出现。
    LLAMA_API void llama_beam_search(
                   struct llama_context * ctx,
        llama_beam_search_callback_fn_t   callback,
                                   void * callback_data,
                                 size_t   n_beams,
// 定义一个函数 llama_init，用于初始化 LLAMA 库
LLAMA_API void llama_init(void);

// 定义一个函数 llama_destroy，用于销毁 LLAMA 库
LLAMA_API void llama_destroy(void);

// 定义一个函数 llama_train，用于训练 LLAMA 模型
LLAMA_API void llama_train(struct llama_context * ctx,
                                    const float * data,
                                    int   n_samples,
                                    int   n_features,
                                    int   n_labels,
                                    int   n_past,
                                    int   n_predict);

// 定义一个函数 llama_get_timings，用于获取性能信息
LLAMA_API struct llama_timings llama_get_timings(struct llama_context * ctx);

// 定义一个函数 llama_print_timings，用于打印性能信息
LLAMA_API void llama_print_timings(struct llama_context * ctx);

// 定义一个函数 llama_reset_timings，用于重置性能信息
LLAMA_API void llama_reset_timings(struct llama_context * ctx);

// 定义一个函数 llama_print_system_info，用于打印系统信息
LLAMA_API const char * llama_print_system_info(void);

// 定义一个函数 llama_log_set，用于设置日志回调函数
// 如果未调用此函数或提供 NULL，则所有内容将输出到 stderr
LLAMA_API void llama_log_set(ggml_log_callback log_callback, void * user_data);

// 定义一个函数 llama_dump_timing_info_yaml，用于将定时信息以 YAML 格式输出到文件流
LLAMA_API void llama_dump_timing_info_yaml(FILE * stream, const struct llama_context * ctx);

#ifdef __cplusplus
}
// 结束条件编译指令，用于条件编译，表示如果之前定义的宏已经定义，则编译以下代码
#endif

// 内部 API，由 llama.cpp 实现并且只能被 tests/benchmarks 使用
#ifdef LLAMA_API_INTERNAL

// 包含 vector 和 string 头文件
#include <vector>
#include <string>

// 定义了一个结构体 ggml_tensor
struct ggml_tensor;

// 声明了一个函数 llama_internal_get_tensor_map，返回一个存储了字符串和 ggml_tensor 指针的 vector 的引用
const std::vector<std::pair<std::string, struct ggml_tensor *>> & llama_internal_get_tensor_map(
    struct llama_context * ctx
);

#endif // LLAMA_API_INTERNAL

#endif // LLAMA_H
```