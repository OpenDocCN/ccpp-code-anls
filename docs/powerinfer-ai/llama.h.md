# `PowerInfer\llama.h`

```
#ifndef LLAMA_H
#define LLAMA_H

#include "ggml.h"  // 包含 ggml.h 头文件
#ifdef GGML_USE_CUBLAS
#include "ggml-cuda.h"  // 如果使用 CUDA，则包含 ggml-cuda.h 头文件
#define LLAMA_MAX_DEVICES GGML_CUDA_MAX_DEVICES  // 如果使用 CUDA，则定义 LLAMA_MAX_DEVICES 为 GGML_CUDA_MAX_DEVICES
#else
#define LLAMA_MAX_DEVICES 1  // 如果不使用 CUDA，则定义 LLAMA_MAX_DEVICES 为 1
#endif // GGML_USE_CUBLAS
#include <stddef.h>  // 包含标准库头文件
#include <stdint.h>  // 包含标准整数类型头文件
#include <stdio.h>  // 包含标准输入输出头文件
#include <stdbool.h>  // 包含布尔类型头文件

#ifdef LLAMA_SHARED
#    if defined(_WIN32) && !defined(__MINGW32__)
#        ifdef LLAMA_BUILD
#            define LLAMA_API __declspec(dllexport)  // 如果在 Windows 下编译并且是构建 LLAMA，则定义 LLAMA_API 为 __declspec(dllexport)
#        else
#            define LLAMA_API __declspec(dllimport)  // 如果在 Windows 下编译但不是构建 LLAMA，则定义 LLAMA_API 为 __declspec(dllimport)
#        endif
#    else
#        define LLAMA_API __attribute__ ((visibility ("default")))  // 在其他平台下，定义 LLAMA_API 为默认可见性
#    endif
#else
#    define LLAMA_API  // 如果不是共享库，则不定义 LLAMA_API
#endif

#ifdef __GNUC__
#    define DEPRECATED(func, hint) func __attribute__((deprecated(hint)))  // 如果是 GCC 编译器，则定义 DEPRECATED 宏
#elif defined(_MSC_VER)
#    define DEPRECATED(func, hint) __declspec(deprecated(hint)) func  // 如果是 MSVC 编译器，则定义 DEPRECATED 宏
#else
#    define DEPRECATED(func, hint) func  // 如果是其他编译器，则定义 DEPRECATED 宏
#endif

#define LLAMA_DEFAULT_SEED 0xFFFFFFFF  // 定义 LLAMA_DEFAULT_SEED 为 0xFFFFFFFF

#define LLAMA_MAX_RNG_STATE (64*1024)  // 定义 LLAMA_MAX_RNG_STATE 为 64*1024

#define LLAMA_FILE_MAGIC_GGSN 0x6767736nu // 'ggsn'  // 定义 LLAMA_FILE_MAGIC_GGSN 为十六进制数 0x6767736nu

#define LLAMA_SESSION_MAGIC   LLAMA_FILE_MAGIC_GGSN  // 定义 LLAMA_SESSION_MAGIC 为 LLAMA_FILE_MAGIC_GGSN
#define LLAMA_SESSION_VERSION 2  // 定义 LLAMA_SESSION_VERSION 为 2

#if defined(GGML_USE_CUBLAS) || defined(GGML_USE_CLBLAST) || defined(GGML_USE_METAL)
// Defined when llama.cpp is compiled with support for offloading model layers to GPU.
#define LLAMA_SUPPORTS_GPU_OFFLOAD  // 如果定义了 GGML 使用 CUDA 或 CLBLAST 或 METAL，则定义 LLAMA_SUPPORTS_GPU_OFFLOAD
#endif

#ifdef __cplusplus
extern "C" {
#endif

    //
    // C interface
    //
    // TODO: show sample usage
    //

    struct llama_model;  // 声明结构体 llama_model
    struct llama_context;  // 声明结构体 llama_context

    typedef int32_t llama_pos;  // 定义 llama_pos 为 int32_t 类型
    typedef int32_t llama_token;  // 定义 llama_token 为 int32_t 类型
    typedef int32_t llama_seq_id;  // 定义 llama_seq_id 为 int32_t 类型

    enum llama_vocab_type {
        LLAMA_VOCAB_TYPE_SPM = 0, // SentencePiece  // 定义枚举类型 llama_vocab_type，取值为 0 时表示 SentencePiece
        LLAMA_VOCAB_TYPE_BPE = 1, // Byte Pair Encoding  // 取值为 1 时表示 Byte Pair Encoding
    };
    // 定义枚举类型 llama_token_type，表示不同的 token 类型
    enum llama_token_type {
        // 未定义类型
        LLAMA_TOKEN_TYPE_UNDEFINED    = 0,
        // 普通类型
        LLAMA_TOKEN_TYPE_NORMAL       = 1,
        // 未知类型
        LLAMA_TOKEN_TYPE_UNKNOWN      = 2,
        // 控制类型
        LLAMA_TOKEN_TYPE_CONTROL      = 3,
        // 用户自定义类型
        LLAMA_TOKEN_TYPE_USER_DEFINED = 4,
        // 未使用类型
        LLAMA_TOKEN_TYPE_UNUSED       = 5,
        // 字节类型
        LLAMA_TOKEN_TYPE_BYTE         = 6,
    };

    // 定义枚举类型 llama_ftype，表示不同的模型文件类型
    enum llama_ftype {
        // 所有类型为 F32
        LLAMA_FTYPE_ALL_F32              = 0,
        // 大部分类型为 F16，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_F16           = 1,  // except 1d tensors
        // 大部分类型为 Q4.0，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q4_0          = 2,  // except 1d tensors
        // 大部分类型为 Q4.1，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q4_1          = 3,  // except 1d tensors
        // 大部分类型为 Q4.1，部分为 F16（tok_embeddings.weight 和 output.weight）
        LLAMA_FTYPE_MOSTLY_Q4_1_SOME_F16 = 4,  // tok_embeddings.weight and output.weight are F16
        // 大部分类型为 Q8.0，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q8_0          = 7,  // except 1d tensors
        // 大部分类型为 Q5.0，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q5_0          = 8,  // except 1d tensors
        // 大部分类型为 Q5.1，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q5_1          = 9,  // except 1d tensors
        // 大部分类型为 Q2_K，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q2_K          = 10, // except 1d tensors
        // 大部分类型为 Q3_K_S，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q3_K_S        = 11, // except 1d tensors
        // 大部分类型为 Q3_K_M，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q3_K_M        = 12, // except 1d tensors
        // 大部分类型为 Q3_K_L，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q3_K_L        = 13, // except 1d tensors
        // 大部分类型为 Q4_K_S，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q4_K_S        = 14, // except 1d tensors
        // 大部分类型为 Q4_K_M，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q4_K_M        = 15, // except 1d tensors
        // 大部分类型为 Q5_K_S，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q5_K_S        = 16, // except 1d tensors
        // 大部分类型为 Q5_K_M，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q5_K_M        = 17, // except 1d tensors
        // 大部分类型为 Q6_K，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q6_K          = 18, // except 1d tensors

        // 猜测的类型，模型文件中未指定
        LLAMA_FTYPE_GUESSED = 1024, // not specified in the model file
    };
    # 定义枚举类型 llama_rope_scaling_type，表示绳索缩放类型
    enum llama_rope_scaling_type {
        LLAMA_ROPE_SCALING_UNSPECIFIED = -1,  # 未指定的缩放类型
        LLAMA_ROPE_SCALING_NONE        = 0,   # 无缩放
        LLAMA_ROPE_SCALING_LINEAR      = 1,   # 线性缩放
        LLAMA_ROPE_SCALING_YARN        = 2,   # 纱线缩放
        LLAMA_ROPE_SCALING_MAX_VALUE   = LLAMA_ROPE_SCALING_YARN,  # 最大值为纱线缩放
    };

    # 定义结构体 llama_token_data，表示羊驼令牌数据
    typedef struct llama_token_data {
        llama_token id;  // 令牌 ID
        float logit;     // 令牌的对数几率
        float p;         // 令牌的概率
    } llama_token_data;

    # 定义结构体 llama_token_data_array，表示羊驼令牌数据数组
    typedef struct llama_token_data_array {
        llama_token_data * data;  // 令牌数据指针
        size_t size;              // 大小
        bool sorted;              // 是否已排序
    } llama_token_data_array;

    # 定义函数指针类型 llama_progress_callback，表示羊驼进度回调函数
    typedef void (*llama_progress_callback)(float progress, void *ctx);

    # 定义结构体 llama_batch，表示 llama_decode 的输入数据
    # 一个 llama_batch 对象可以包含有关一个或多个序列的输入
    # 提供的数组（例如 token、embd、pos 等）必须具有 n_tokens 大小
    #
    # - token  : 输入的令牌 ID（当 embd 为 NULL 时使用）
    # - embd   : 令牌嵌入（即大小为 n_embd 的浮点向量）（当 token 为 NULL 时使用）
    # - pos    : 序列中各令牌的位置
    # - seq_id : 各令牌所属的序列
    # - logits : 如果为零，则不会输出各令牌的对数几率
    #
    typedef struct llama_batch {
        int32_t n_tokens;  # 令牌数量

        llama_token  *  token;    # 令牌 ID 数组
        float        *  embd;     # 令牌嵌入数组
        llama_pos    *  pos;      # 位置数组
        int32_t      *  n_seq_id; # 序列 ID 数组
        llama_seq_id ** seq_id;   # 序列 ID 指针数组
        int8_t       *  logits;   # 对数几率数组

        # 用于平滑 API 过渡的辅助字段 - 可在将来被弃用
        # 为了未来的代码兼容性，请使用上述字段，并忽略以下所有内容
        #
        # pos[i] = all_pos_0 + i*all_pos_1
        #
        llama_pos    all_pos_0;  # 如果 pos 为 NULL，则使用的位置
        llama_pos    all_pos_1;  # 如果 pos 为 NULL，则使用的位置
        llama_seq_id all_seq_id; # 如果 seq_id 为 NULL，则使用的序列 ID
    } llama_batch;
    // 定义了一个结构体 llama_model_params，用于存储模型参数
    struct llama_model_params {
        int32_t n_gpu_layers; // 存储在 VRAM 中的层数
        int32_t main_gpu;     // 用于临时和小张量的 GPU
        float vram_budget_gb; // VRAM 预算，单位为 GB，-1 表示所有可用的 VRAM（针对单个 GPU）
        const float * tensor_split; // 如何在多个 GPU 上分割层（大小为 LLAMA_MAX_DEVICES）

        // 用于传递进度值的回调函数，传递 NULL 表示禁用
        llama_progress_callback progress_callback;
        // 传递给进度回调函数的上下文指针
        void * progress_callback_user_data;

        // 将布尔值放在一起，以避免在按值复制时出现错位
        bool vocab_only; // 仅加载词汇表，不加载权重
        bool use_mmap;   // 如果可能，使用 mmap
        bool use_mlock;  // 强制系统将模型保留在 RAM 中
        bool reset_gpu_index; // 强制重置 GPU 索引
        bool disable_gpu_index; // 绕过 GPU 索引和 FFN 分割
    };
    // 定义了 llama 上下文参数的结构体
    struct llama_context_params {
        uint32_t seed;              // 随机数种子，-1 表示随机
        uint32_t n_ctx;             // 文本上下文，0 表示从模型中获取
        uint32_t n_batch;           // 提示处理的最大批处理大小
        uint32_t n_threads;         // 用于生成的线程数
        uint32_t n_threads_batch;   // 用于批处理的线程数
        int8_t   rope_scaling_type; // RoPE 缩放类型，来自枚举 llama_rope_scaling_type

        // 参考：https://github.com/ggerganov/llama.cpp/pull/2054
        float    rope_freq_base;   // RoPE 基础频率，0 表示从模型中获取
        float    rope_freq_scale;  // RoPE 频率缩放因子，0 表示从模型中获取
        float    yarn_ext_factor;  // YaRN 外推混合因子，NaN 表示从模型中获取
        float    yarn_attn_factor; // YaRN 幅度缩放因子
        float    yarn_beta_fast;   // YaRN 低校正维度
        float    yarn_beta_slow;   // YaRN 高校正维度
        uint32_t yarn_orig_ctx;    // YaRN 原始上下文大小

        // 保持布尔值在一起，以避免在按值复制时出现错位。
        bool mul_mat_q;  // 如果为 true，则使用实验性的 mul_mat_q 内核（已弃用 - 始终为 true）
        bool f16_kv;     // 对 KV 缓存使用 fp16，否则使用 fp32
        bool logits_all; // llama_eval() 调用计算所有 logits，而不仅仅是最后一个
        bool embedding;  // 仅嵌入模式
    };

    // 模型量化参数
    // 定义了用于量化模型的参数结构体
    typedef struct llama_model_quantize_params {
        int nthread;                 // 用于量化的线程数，如果<=0，则使用std::thread::hardware_concurrency()
        enum llama_ftype ftype;      // 量化到指定的llama_ftype类型
        bool allow_requantize;       // 允许对非f32/f16张量进行重新量化
        bool quantize_output_tensor; // 量化输出权重张量
        bool only_copy;              // 仅复制张量 - ftype、allow_requantize和quantize_output_tensor将被忽略
        bool pure;                   // 禁用k-quant混合，将所有张量量化为相同类型
    } llama_model_quantize_params;

    // 语法类型
    struct llama_grammar;

    // 语法元素类型
    enum llama_gretype {
        // 规则定义结束
        LLAMA_GRETYPE_END            = 0,

        // 规则的备选定义开始
        LLAMA_GRETYPE_ALT            = 1,

        // 非终结符元素：引用规则
        LLAMA_GRETYPE_RULE_REF       = 2,

        // 终结符元素：字符（Unicode码点）
        LLAMA_GRETYPE_CHAR           = 3,

        // 反向字符（[^a]，[^a-b] [^abc]）
        LLAMA_GRETYPE_CHAR_NOT       = 4,

        // 修改前面的LLAMA_GRETYPE_CHAR或LLAMA_GRETYPE_CHAR_ALT，使其成为包含范围（[a-z]）
        LLAMA_GRETYPE_CHAR_RNG_UPPER = 5,

        // 修改前面的LLAMA_GRETYPE_CHAR或LLAMA_GRETYPE_CHAR_RNG_UPPER，添加一个要匹配的备选字符（[ab]，[a-zA]）
        LLAMA_GRETYPE_CHAR_ALT       = 6,
    };

    typedef struct llama_grammar_element {
        enum llama_gretype type;
        uint32_t           value; // Unicode码点或规则ID
    } llama_grammar_element;

    // 性能定时信息
    // 定义了一个名为 llama_timings 的结构体，包含了各种时间和计数的成员变量
    struct llama_timings {
        double t_start_ms;
        double t_end_ms;
        double t_load_ms;
        double t_sample_ms;
        double t_p_eval_ms;
        double t_eval_ms;
    
        int32_t n_sample;
        int32_t n_p_eval;
        int32_t n_eval;
    };
    
    // 获取默认的模型参数
    LLAMA_API struct llama_model_params llama_model_default_params(void);
    
    // 获取默认的上下文参数
    LLAMA_API struct llama_context_params llama_context_default_params(void);
    
    // 获取默认的模型量化参数
    LLAMA_API struct llama_model_quantize_params llama_model_quantize_default_params(void);
    
    // 初始化 llama + ggml 后端，如果 numa 为 true，则使用 NUMA 优化
    // 在程序开始时调用一次
    LLAMA_API void llama_backend_init(bool numa);
    
    // 在程序结束时调用一次，目前仅用于 MPI
    LLAMA_API void llama_backend_free(void);
    
    // 从文件加载模型
    LLAMA_API struct llama_model * llama_load_model_from_file(
                             const char * path_model,
            struct llama_model_params     params);
    
    // 释放模型内存
    LLAMA_API void llama_free_model(struct llama_model * model);
    
    // 使用模型创建新的上下文
    LLAMA_API struct llama_context * llama_new_context_with_model(
                     struct llama_model * model,
            struct llama_context_params   params);
    
    // 释放所有分配的内存
    LLAMA_API void llama_free(struct llama_context * ctx);
    
    // 获取当前时间的微秒数
    LLAMA_API int64_t llama_time_us(void);
    
    // 获取最大设备数
    LLAMA_API int  llama_max_devices    (void);
    
    // 检查是否支持内存映射
    LLAMA_API bool llama_mmap_supported (void);
    
    // 检查是否支持内存锁定
    LLAMA_API bool llama_mlock_supported(void);
    
    // 获取上下文对应的模型
    LLAMA_API const struct llama_model * llama_get_model(const struct llama_context * ctx);
    
    // 获取上下文的数量
    LLAMA_API int llama_n_ctx      (const struct llama_context * ctx);
    
    // 获取模型的词汇表类型
    LLAMA_API enum llama_vocab_type llama_vocab_type(const struct llama_model * model);
    
    // 检查模型是否使用稀疏推断
    LLAMA_API bool llama_use_sparse_inference(const struct llama_model * model);
    
    // 获取模型的词汇表数量
    LLAMA_API int llama_n_vocab    (const struct llama_model * model);
    // 训练上下文中的模型
    LLAMA_API int llama_n_ctx_train(const struct llama_model * model);

    // 获取模型的嵌入层数量
    LLAMA_API int llama_n_embd     (const struct llama_model * model);

    // 获取模型的 RoPE 频率缩放因子
    LLAMA_API float llama_rope_freq_scale_train(const struct llama_model * model);

    // 获取描述模型类型的字符串
    LLAMA_API int llama_model_desc(const struct llama_model * model, char * buf, size_t buf_size);

    // 返回模型中所有张量的总大小（以字节为单位）
    LLAMA_API uint64_t llama_model_size(const struct llama_model * model);

    // 返回模型中参数的总数
    LLAMA_API uint64_t llama_model_n_params(const struct llama_model * model);

    // 获取 llama 模型张量
    LLAMA_API struct ggml_tensor * llama_get_model_tensor(struct llama_model * model, const char * name);

    // 对模型进行量化
    // fname_inp: 输入文件名
    // fname_out: 输出文件名
    // params: 量化参数
    // 返回成功时返回 0
    LLAMA_API int llama_model_quantize(
            const char * fname_inp,
            const char * fname_out,
            const llama_model_quantize_params * params);

    // 对加载的模型应用 LoRA 适配器
    // path_base_model: 用作适配器修改层基础的高质量模型的路径。可以为 NULL，表示使用当前加载的模型。
    // 模型在应用新适配器之前需要重新加载，否则适配器将应用在之前的适配器之上
    // 返回成功时返回 0
    LLAMA_API DEPRECATED(int llama_apply_lora_from_file(
            struct llama_context * ctx,
                      const char * path_lora,
                           float   scale,
                      const char * path_base_model,
                             int   n_threads),
            "use llama_model_apply_lora_from_file instead");
    // 从文件中应用 LoRa 模型，返回操作结果
    LLAMA_API int llama_model_apply_lora_from_file(
            const struct llama_model * model,
                      const char * path_lora,
                           float   scale,
                      const char * path_base_model,
                             int   n_threads);

    // 从文件中应用 GPU 索引模型，返回操作结果
    LLAMA_API int llama_model_apply_gpu_idx_from_file(
                  struct llama_model * model,
                          const char * path_mlp,
                                bool   use_mmap);

    // 将前馈神经网络模型拆分并卸载到内存中
    LLAMA_API size_t llama_model_offload_ffn_split(struct llama_model * model);

    //
    // KV cache
    //

    // 返回 KV 缓存中的令牌数
    LLAMA_API DEPRECATED(int llama_get_kv_cache_token_count(const struct llama_context * ctx),
            "avoid using this, it will be removed in the future, instead - count the tokens in user code");

    // 清空 KV 缓存
    LLAMA_API void llama_kv_cache_clear(
            struct llama_context * ctx);

    // 移除属于指定序列并且位置在 [p0, p1) 范围内的所有令牌
    // seq_id < 0 : 匹配任何序列
    // p0 < 0     : [0,  p1]
    // p1 < 0     : [p0, inf)
    LLAMA_API void llama_kv_cache_seq_rm(
            struct llama_context * ctx,
                    llama_seq_id   seq_id,
                       llama_pos   p0,
                       llama_pos   p1);

    // 将属于指定序列的所有令牌复制到另一个序列中
    // 注意：这不会分配额外的 KV 缓存内存 - 它只是将令牌分配给新序列
    // p0 < 0 : [0,  p1]
    // p1 < 0 : [p0, inf)
    LLAMA_API void llama_kv_cache_seq_cp(
            struct llama_context * ctx,
                    llama_seq_id   seq_id_src,
                    llama_seq_id   seq_id_dst,
                       llama_pos   p0,
                       llama_pos   p1);

    // 移除不属于指定序列的所有令牌
    // 保持指定序列的键值缓存的顺序
    LLAMA_API void llama_kv_cache_seq_keep(
            struct llama_context * ctx,
                    llama_seq_id   seq_id);

    // 将相对位置“delta”添加到所有属于指定序列并在[p0, p1)范围内的标记
    // 如果KV缓存被RoPEd，则相应地更新KV数据
    // p0 < 0 : [0,  p1]
    // p1 < 0 : [p0, inf)
    LLAMA_API void llama_kv_cache_seq_shift(
            struct llama_context * ctx,
                    llama_seq_id   seq_id,
                       llama_pos   p0,
                       llama_pos   p1,
                       llama_pos   delta);

    //
    // 状态 / 会话
    //

    // 返回状态（rng、logits、embedding和kv_cache）的最大字节大小 - 在压缩标记后通常会更小
    LLAMA_API size_t llama_get_state_size(const struct llama_context * ctx);

    // 将状态复制到指定的目标地址。
    // 目标需要分配足够的内存。
    // 返回复制的字节数
    LLAMA_API size_t llama_copy_state_data(
            struct llama_context * ctx,
                         uint8_t * dst);

    // 从指定地址设置状态
    // 返回读取的字节数
    LLAMA_API size_t llama_set_state_data(
            struct llama_context * ctx,
                         uint8_t * src);

    // 保存/加载会话文件
    LLAMA_API bool llama_load_session_file(
            struct llama_context * ctx,
                      const char * path_session,
                     llama_token * tokens_out,
                          size_t   n_token_capacity,
                          size_t * n_token_count_out);

    LLAMA_API bool llama_save_session_file(
            struct llama_context * ctx,
                      const char * path_session,
               const llama_token * tokens,
                          size_t   n_token_count);

    //
    // 解码
    //
    // 运行羊驼推断以获取下一个标记的对数和概率。
    // tokens + n_tokens 是要处理的新标记提供的批处理
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
            "use llama_decode() instead");
    
    // 返回从pos_0开始的单个标记序列的批处理
    //
    // 注意：这是一个辅助函数，以便过渡到新的批处理API - 避免使用它
    //
    LLAMA_API struct llama_batch llama_batch_get_one(
                  llama_token * tokens,
                      int32_t   n_tokens,
                    llama_pos   pos_0,
                 llama_seq_id   seq_id);
    
    // 在堆上分配一个可以容纳最多n_tokens的标记批处理
    // 每个标记最多可以分配n_seq_max个序列id
    // 必须使用llama_batch_free()释放批处理
    // 如果embd != 0，则llama_batch.embd将分配n_tokens * embd * sizeof(float)的大小
    // 否则，llama_batch.token将被分配以存储n_tokens llama_token
    // llama_batch的其余成员将分配大小为n_tokens
    // 所有成员都保持未初始化状态
    LLAMA_API struct llama_batch llama_batch_init(
            int32_t n_tokens,
            int32_t embd,
            int32_t n_seq_max);
    // 释放使用 llama_batch_init() 分配的一批令牌
    LLAMA_API void llama_batch_free(struct llama_batch batch);
    
    // 正返回值并不意味着致命错误，而是警告。
    //   0 - 成功
    //   1 - 无法为批处理找到 KV 槽（尝试减少批处理大小或增加上下文）
    // < 0 - 错误
    LLAMA_API int llama_decode(
            struct llama_context * ctx,
            struct llama_batch   batch);
    
    // 设置用于解码的线程数
    // n_threads 是用于生成（单个令牌）的线程数
    // n_threads_batch 是用于提示和批处理处理（多个令牌）的线程数
    LLAMA_API void llama_set_n_threads(struct llama_context * ctx, uint32_t n_threads, uint32_t n_threads_batch);
    
    // 从上一次 llama_eval() 调用中获取的令牌 logits
    // 最后一个令牌的 logits 存储在最后一行中
    // 对于其中 llama_batch.logits[i] == 0 的 logits 是未定义的
    // 行数：使用 llama_batch 提供的 n_tokens
    // 列数：n_vocab
    LLAMA_API float * llama_get_logits(struct llama_context * ctx);
    
    // 第 i 个令牌的 logits。等同于：
    // llama_get_logits(ctx) + i*n_vocab
    LLAMA_API float * llama_get_logits_ith(struct llama_context * ctx, int32_t i);
    
    // 获取输入的嵌入
    // 形状：[n_embd]（一维）
    LLAMA_API float * llama_get_embeddings(struct llama_context * ctx);
    
    //
    // 词汇表
    //
    
    LLAMA_API const char * llama_token_get_text(const struct llama_model * model, llama_token token);
    
    LLAMA_API float llama_token_get_score(const struct llama_model * model, llama_token token);
    
    LLAMA_API enum llama_token_type llama_token_get_type(const struct llama_model * model, llama_token token);
    
    // 特殊令牌
    LLAMA_API llama_token llama_token_bos(const struct llama_model * model); // 句子开头
    // 定义函数 llama_token_eos，返回一个表示句子结束的 token
    LLAMA_API llama_token llama_token_eos(const struct llama_model * model); // end-of-sentence
    // 定义函数 llama_token_nl，返回一个表示下一行的 token
    LLAMA_API llama_token llama_token_nl (const struct llama_model * model); // next-line
    
    // 定义函数 llama_token_prefix，返回一个表示填充前缀开始的 token
    LLAMA_API llama_token llama_token_prefix(const struct llama_model * model); // Beginning of infill prefix
    // 定义函数 llama_token_middle，返回一个表示填充中间部分开始的 token
    LLAMA_API llama_token llama_token_middle(const struct llama_model * model); // Beginning of infill middle
    // 定义函数 llama_token_suffix，返回一个表示填充后缀开始的 token
    LLAMA_API llama_token llama_token_suffix(const struct llama_model * model); // Beginning of infill suffix
    // 定义函数 llama_token_eot，返回一个表示填充中间结束的 token
    LLAMA_API llama_token llama_token_eot   (const struct llama_model * model); // End of infill middle
    
    //
    // Tokenization
    //
    
    /// @details 将提供的文本转换为 token。
    /// @param tokens tokens 指针必须足够大，以容纳生成的 tokens。
    /// @return 成功时返回 token 的数量，不超过 n_max_tokens
    /// @return 失败时返回负数 - 应返回的 token 数量
    /// @param special 允许对特殊和/或控制 token 进行标记化，否则将被视为纯文本。不插入前导空格。
    LLAMA_API int llama_tokenize(
        const struct llama_model * model,
                      const char * text,
                             int   text_len,
                     llama_token * tokens,
                             int   n_max_tokens,
                            bool   add_bos,
                            bool   special);
    
    // Token Id -> Piece.
    // 使用提供的上下文中的词汇表。
    // 不向缓冲区写入空终止符。
    // 用户代码负责在解码多个 token 时去除第一个非 BOS token 的前导空格。
    # 将令牌转换为片段，使用给定的模型和缓冲区长度
    LLAMA_API int llama_token_to_piece(
              const struct llama_model * model,
                           llama_token   token,
                                  char * buf,
                                  int    length);

    //
    // Grammar
    //

    # 初始化 LLAMA 语法对象，包括规则、规则数量和起始规则索引
    LLAMA_API struct llama_grammar * llama_grammar_init(
            const llama_grammar_element ** rules,
                                 size_t    n_rules,
                                 size_t    start_rule_index);

    # 释放 LLAMA 语法对象占用的内存
    LLAMA_API void llama_grammar_free(struct llama_grammar * grammar);

    # 复制 LLAMA 语法对象
    LLAMA_API struct llama_grammar * llama_grammar_copy(const struct llama_grammar * grammar);

    //
    // Sampling functions
    //

    # 设置当前的随机数生成器种子
    LLAMA_API void llama_set_rng_seed(struct llama_context * ctx, uint32_t seed);

    # 对候选令牌进行重复惩罚，包括最后 N 个令牌、重复惩罚、频率惩罚和存在惩罚
    LLAMA_API void llama_sample_repetition_penalties(
            struct llama_context * ctx,
          llama_token_data_array * candidates,
               const llama_token * last_tokens,
                          size_t   penalty_last_n,
                           float   penalty_repeat,
                           float   penalty_freq,
                           float   penalty_present);

    # 将分类器无关的指导应用于对数，如学术论文 "Stay on topic with Classifier-Free Guidance" 中所述
    # 参数包括候选令牌数据和对数，对数必须直接从原始生成上下文中提取，而不是排序
    /// @param candidates A vector of `llama_token_data` containing the candidate tokens, the logits must be directly extracted from the original generation context without being sorted.
    // 释放给定上下文中的指定候选标记，根据另一个上下文提供的指导信息进行采样
    LLAMA_API void llama_sample_classifier_free_guidance(
              struct llama_context * ctx,
            llama_token_data_array * candidates,
              struct llama_context * guidance_ctx,
                             float   scale);
    
    // 根据标记的logits值按降序对候选标记进行排序，并基于logits计算概率
    LLAMA_API void llama_sample_softmax(
            struct llama_context * ctx,
          llama_token_data_array * candidates);
    
    // 在学术论文"The Curious Case of Neural Text Degeneration"中描述的Top-K采样
    LLAMA_API void llama_sample_top_k(
            struct llama_context * ctx,
          llama_token_data_array * candidates,
                             int   k,
                          size_t   min_keep);
    
    // 在学术论文"The Curious Case of Neural Text Degeneration"中描述的Nucleus采样
    LLAMA_API void llama_sample_top_p(
            struct llama_context * ctx,
          llama_token_data_array * candidates,
                           float   p,
                          size_t   min_keep);
    
    // 如https://github.com/ggerganov/llama.cpp/pull/3841中描述的最小P采样
    LLAMA_API void llama_sample_min_p(
            struct llama_context * ctx,
          llama_token_data_array * candidates,
                           float   p,
                          size_t   min_keep);
    
    // 在https://www.trentonbricken.com/Tail-Free-Sampling/中描述的Tail Free采样
    // 释放尾部样本，释放内存
    LLAMA_API void llama_sample_tail_free(
            struct llama_context * ctx,
          llama_token_data_array * candidates,
                           float   z,
                          size_t   min_keep);
    
    /// @details 在论文 https://arxiv.org/abs/2202.00666 中描述的局部典型抽样实现。
    LLAMA_API void llama_sample_typical(
            struct llama_context * ctx,
          llama_token_data_array * candidates,
                           float   p,
                          size_t   min_keep);
    
    // 温度抽样
    LLAMA_API void llama_sample_temp(
            struct llama_context * ctx,
          llama_token_data_array * candidates,
                           float   temp);
    
    // 温度抽样，已弃用
    LLAMA_API DEPRECATED(void llama_sample_temperature(
                struct llama_context * ctx,
              llama_token_data_array * candidates,
                               float   temp),
            "use llama_sample_temp instead");
    
    /// @details 应用语法约束
    LLAMA_API void llama_sample_grammar(
            struct llama_context * ctx,
          llama_token_data_array * candidates,
      const struct llama_grammar * grammar);
    
    /// @details 在论文 https://arxiv.org/abs/2007.14966 中描述的 Mirostat 1.0 算法。使用标记而不是单词。
    /// @param candidates 包含候选标记、它们在生成文本中当前位置的概率 (p) 和对数几率 (logit) 的 `llama_token_data` 向量。
    /// @param tau  你想要为生成的文本实现的目标交叉熵 (或惊喜) 值。较高的值对应于更令人惊讶或不太可预测的文本，而较低的值对应于较不令人惊讶或更可预测的文本。
    /// @param eta The learning rate used to update `mu` based on the error between the target and observed surprisal of the sampled word. A larger learning rate will cause `mu` to be updated more quickly, while a smaller learning rate will result in slower updates.
    /// 参数 eta：学习率，用于根据采样单词的目标和观察到的意外性之间的误差来更新 `mu`。较大的学习率会导致 `mu` 更新更快，而较小的学习率会导致更新较慢。

    /// @param m The number of tokens considered in the estimation of `s_hat`. This is an arbitrary value that is used to calculate `s_hat`, which in turn helps to calculate the value of `k`. In the paper, they use `m = 100`, but you can experiment with different values to see how it affects the performance of the algorithm.
    /// 参数 m：在估计 `s_hat` 时考虑的标记数量。这是一个任意值，用于计算 `s_hat`，进而帮助计算 `k` 的值。在论文中，他们使用 `m = 100`，但您可以尝试不同的值，看看它如何影响算法的性能。

    /// @param mu Maximum cross-entropy. This value is initialized to be twice the target cross-entropy (`2 * tau`) and is updated in the algorithm based on the error between the target and observed surprisal.
    /// 参数 mu：最大交叉熵。该值初始化为目标交叉熵的两倍（`2 * tau`），并根据目标和观察到的意外性之间的误差在算法中进行更新。

    LLAMA_API llama_token llama_sample_token_mirostat(
            struct llama_context * ctx,
          llama_token_data_array * candidates,
                           float   tau,
                           float   eta,
                             int   m,
                           float * mu);

    /// @details Mirostat 2.0 algorithm described in the paper https://arxiv.org/abs/2007.14966. Uses tokens instead of words.
    /// 详细信息：在论文 https://arxiv.org/abs/2007.14966 中描述的 Mirostat 2.0 算法。使用标记而不是单词。

    /// @param candidates A vector of `llama_token_data` containing the candidate tokens, their probabilities (p), and log-odds (logit) for the current position in the generated text.
    /// 参数 candidates：包含候选标记、它们的概率（p）和当前生成文本位置的对数几率（log-odds）的 `llama_token_data` 向量。

    /// @param tau  The target cross-entropy (or surprise) value you want to achieve for the generated text. A higher value corresponds to more surprising or less predictable text, while a lower value corresponds to less surprising or more predictable text.
    /// 参数 tau：生成文本所需达到的目标交叉熵（或意外性）值。较高的值对应于更令人惊讶或不太可预测的文本，而较低的值对应于不太令人惊讶或更可预测的文本。

    /// @param eta The learning rate used to update `mu` based on the error between the target and observed surprisal of the sampled word. A larger learning rate will cause `mu` to be updated more quickly, while a smaller learning rate will result in slower updates.
    /// 参数 eta：学习率，用于根据采样单词的目标和观察到的意外性之间的误差来更新 `mu`。较大的学习率会导致 `mu` 更新更快，而较小的学习率会导致更新较慢。
    /// @param mu Maximum cross-entropy. This value is initialized to be twice the target cross-entropy (`2 * tau`) and is updated in the algorithm based on the error between the target and observed surprisal.
    LLAMA_API llama_token llama_sample_token_mirostat_v2(
            struct llama_context * ctx,
          llama_token_data_array * candidates,
                           float   tau,
                           float   eta,
                           float * mu);

    /// @details Selects the token with the highest probability.
    ///          Does not compute the token probabilities. Use llama_sample_softmax() instead.
    LLAMA_API llama_token llama_sample_token_greedy(
            struct llama_context * ctx,
          llama_token_data_array * candidates);

    /// @details Randomly selects a token from the candidates based on their probabilities.
    LLAMA_API llama_token llama_sample_token(
            struct llama_context * ctx,
          llama_token_data_array * candidates);

    /// @details Accepts the sampled token into the grammar
    LLAMA_API void llama_grammar_accept_token(
            struct llama_context * ctx,
            struct llama_grammar * grammar,
                     llama_token   token);

    //
    // Beam search
    //

    struct llama_beam_view {
        const llama_token * tokens;  // Array of tokens
        size_t n_tokens;             // Number of tokens in the array
        float  p;                    // Cumulative beam probability (renormalized relative to all beams)
        bool   eob;                  // Callback should set this to true when a beam is at end-of-beam.
    };

    // Passed to beam_search_callback function.
    // Whenever 0 < common_prefix_length, this number of tokens should be copied from any of the beams
    // (e.g. beams[0]) as they will be removed (shifted) from all beams in all subsequent callbacks.
    // These pointers are valid only during the synchronous callback, so should not be saved.
    // 定义了一个结构体，用于存储 LLAMA 系统中的光束搜索状态
    struct llama_beams_state {
        struct llama_beam_view * beam_views;  // 指向光束视图的指针

        size_t n_beams;               // beam_views[] 中元素的数量
        size_t common_prefix_length;  // 当前所有光束共享的前缀标记的最大长度
        bool   last_call;             // 如果这是最后一次回调调用，则为 true
    };

    // 指向 beam_search_callback 函数的指针类型
    // void* callback_data 是传递给 llama_beam_search 的任何自定义数据，随后传递给 beam_search_callback。这避免了在回调中使用全局变量。
    typedef void (*llama_beam_search_callback_fn_t)(void * callback_data, struct llama_beams_state);

    /// @details 确定性地返回由光束搜索构建的整个句子。
    /// @param ctx 指向 llama_context 的指针
    /// @param callback 每次 beam_search 循环迭代时调用，传入 beams_state
    /// @param callback_data 简单地传递给回调的指针
    /// @param n_beams 要使用的光束数量
    /// @param n_past 已评估的标记数量
    /// @param n_predict 要预测的最大标记数量。EOS 可能会更早出现。
    LLAMA_API void llama_beam_search(
                   struct llama_context * ctx,
        llama_beam_search_callback_fn_t   callback,
                                   void * callback_data,
                                 size_t   n_beams,
                                    int   n_past,
                                    int   n_predict);

    // 性能信息
    LLAMA_API struct llama_timings llama_get_timings(struct llama_context * ctx);

    LLAMA_API void llama_print_timings(struct llama_context * ctx);
    LLAMA_API void llama_reset_timings(struct llama_context * ctx);

    // 打印系统信息
    LLAMA_API const char * llama_print_system_info(void);

    // 设置所有未来日志事件的回调。
    // 设置日志回调函数和用户数据，如果不调用此函数或者提供 NULL，则所有内容都输出到 stderr
    LLAMA_API void llama_log_set(ggml_log_callback log_callback, void * user_data);
    
    // 将上下文中的时间信息以 YAML 格式输出到指定的文件流中
    LLAMA_API void llama_dump_timing_info_yaml(FILE * stream, const struct llama_context * ctx);
#ifdef __cplusplus
}
#endif

// 仅供 llama.cpp 实现并且被 tests/benchmarks 使用的内部 API
#ifdef LLAMA_API_INTERNAL

#include <vector>  // 包含 vector 头文件
#include <string>  // 包含 string 头文件

struct ggml_tensor;  // 声明 ggml_tensor 结构体

// 返回指向 (字符串, ggml_tensor 指针) 对的 vector 的引用
const std::vector<std::pair<std::string, struct ggml_tensor *>> & llama_internal_get_tensor_map(
    struct llama_context * ctx
);

#endif // LLAMA_API_INTERNAL

#endif // LLAMA_H
```