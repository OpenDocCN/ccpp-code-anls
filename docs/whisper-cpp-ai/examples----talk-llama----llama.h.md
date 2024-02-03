# `whisper.cpp\examples\talk-llama\llama.h`

```cpp
#ifndef LLAMA_H
#define LLAMA_H



#include "ggml.h"
#include "ggml-backend.h"
#ifdef GGML_USE_CUBLAS
#include "ggml-cuda.h"
#define LLAMA_MAX_DEVICES GGML_CUDA_MAX_DEVICES
#elif defined(GGML_USE_SYCL)
#include "ggml-sycl.h"
#define LLAMA_MAX_DEVICES GGML_SYCL_MAX_DEVICES
#else
#define LLAMA_MAX_DEVICES 1
#endif // GGML_USE_CUBLAS



#include <stddef.h>
#include <stdint.h>
#include <stdio.h>
#include <stdbool.h>



#ifdef LLAMA_SHARED
#    if defined(_WIN32) && !defined(__MINGW32__)
#        ifdef LLAMA_BUILD
#            define LLAMA_API __declspec(dllexport)
#        else
#            define LLAMA_API __declspec(dllimport)
#        endif
#    else
#        define LLAMA_API __attribute__ ((visibility ("default")))
#    endif
#else
#    define LLAMA_API
#endif



#ifdef __GNUC__
#    define DEPRECATED(func, hint) func __attribute__((deprecated(hint)))
#elif defined(_MSC_VER)
#    define DEPRECATED(func, hint) __declspec(deprecated(hint)) func
#else
#    define DEPRECATED(func, hint) func
#endif



#define LLAMA_DEFAULT_SEED 0xFFFFFFFF



#define LLAMA_MAX_RNG_STATE (64*1024)



#define LLAMA_FILE_MAGIC_GGLA 0x67676c61u // 'ggla'
#define LLAMA_FILE_MAGIC_GGSN 0x6767736eu // 'ggsn'



#define LLAMA_SESSION_MAGIC   LLAMA_FILE_MAGIC_GGSN
#define LLAMA_SESSION_VERSION 4



#if defined(GGML_USE_CUBLAS) || defined(GGML_USE_CLBLAST) || defined(GGML_USE_METAL) || defined(GGML_USE_VULKAN) || defined(GGML_USE_SYCL)
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



    struct llama_model;
    struct llama_context;

    typedef int32_t llama_pos;
    typedef int32_t llama_token;
    typedef int32_t llama_seq_id;

    enum llama_vocab_type {
        LLAMA_VOCAB_TYPE_SPM = 0, // SentencePiece
        LLAMA_VOCAB_TYPE_BPE = 1, // Byte Pair Encoding
    };



#endif
    // 定义枚举类型 llama_token_type，表示不同的 token 类型
    enum llama_token_type {
        // 未定义的 token 类型
        LLAMA_TOKEN_TYPE_UNDEFINED    = 0,
        // 普通的 token 类型
        LLAMA_TOKEN_TYPE_NORMAL       = 1,
        // 未知的 token 类型
        LLAMA_TOKEN_TYPE_UNKNOWN      = 2,
        // 控制类的 token 类型
        LLAMA_TOKEN_TYPE_CONTROL      = 3,
        // 用户自定义的 token 类型
        LLAMA_TOKEN_TYPE_USER_DEFINED = 4,
        // 未使用的 token 类型
        LLAMA_TOKEN_TYPE_UNUSED       = 5,
        // 字节类型的 token 类型
        LLAMA_TOKEN_TYPE_BYTE         = 6,
    };

    // 模型文件类型
    // 定义枚举类型 llama_ftype，表示不同的数据类型
    enum llama_ftype {
        // 所有数据类型为 F32
        LLAMA_FTYPE_ALL_F32              = 0,
        // 大部分数据类型为 F16，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_F16           = 1,  // except 1d tensors
        // 大部分数据类型为 Q4_0，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q4_0          = 2,  // except 1d tensors
        // 大部分数据类型为 Q4_1，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q4_1          = 3,  // except 1d tensors
        // 大部分数据类型为 Q4_1，除了 1d 张量，tok_embeddings.weight 和 output.weight 为 F16
        LLAMA_FTYPE_MOSTLY_Q4_1_SOME_F16 = 4,  // tok_embeddings.weight and output.weight are F16
        // 大部分数据类型为 Q8_0，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q8_0          = 7,  // except 1d tensors
        // 大部分数据类型为 Q5_0，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q5_0          = 8,  // except 1d tensors
        // 大部分数据类型为 Q5_1，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q5_1          = 9,  // except 1d tensors
        // 大部分数据类型为 Q2_K，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q2_K          = 10, // except 1d tensors
        // 大部分数据类型为 Q3_K_S，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q3_K_S        = 11, // except 1d tensors
        // 大部分数据类型为 Q3_K_M，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q3_K_M        = 12, // except 1d tensors
        // 大部分数据类型为 Q3_K_L，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q3_K_L        = 13, // except 1d tensors
        // 大部分数据类型为 Q4_K_S，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q4_K_S        = 14, // except 1d tensors
        // 大部分数据类型为 Q4_K_M，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q4_K_M        = 15, // except 1d tensors
        // 大部分数据类型为 Q5_K_S，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q5_K_S        = 16, // except 1d tensors
        // 大部分数据类型为 Q5_K_M，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q5_K_M        = 17, // except 1d tensors
        // 大部分数据类型为 Q6_K，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q6_K          = 18, // except 1d tensors
        // 大部分数据类型为 IQ2_XXS，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_IQ2_XXS       = 19, // except 1d tensors
        // 大部分数据类型为 IQ2_XS，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_IQ2_XS        = 20, // except 1d tensors
        // 大部分数据类型为 Q2_K_S，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q2_K_S        = 21, // except 1d tensors
        // 大部分数据类型为 Q3_K_XS，除了 1d 张量
        LLAMA_FTYPE_MOSTLY_Q3_K_XS       = 22, // except 1d tensors

        // 猜测的数据类型，未在模型文件中指定
        LLAMA_FTYPE_GUESSED = 1024, // not specified in the model file
    };

    // 定义枚举类型 llama_rope_scaling_type，表示不同的缩放类型
    enum llama_rope_scaling_type {
        // 未指定缩放类型
        LLAMA_ROPE_SCALING_UNSPECIFIED = -1,
        // 无缩放
        LLAMA_ROPE_SCALING_NONE        = 0,
        // 线性缩放
        LLAMA_ROPE_SCALING_LINEAR      = 1,
        // 麻绳缩放
        LLAMA_ROPE_SCALING_YARN        = 2,
        // 最大值为麻绳缩放
        LLAMA_ROPE_SCALING_MAX_VALUE   = LLAMA_ROPE_SCALING_YARN,
    };
    // 定义枚举类型 llama_split_mode，表示 LLAMA 的分割模式
    enum llama_split_mode {
        LLAMA_SPLIT_NONE    = 0, // 单个 GPU
        LLAMA_SPLIT_LAYER   = 1, // 将层和键值对跨 GPU 分割
        LLAMA_SPLIT_ROW     = 2, // 将行跨 GPU 分割
    };

    // 定义结构体 llama_token_data，包含 token 数据的结构
    typedef struct llama_token_data {
        llama_token id; // token 的 id
        float logit;    // token 的对数几率
        float p;        // token 的概率
    } llama_token_data;

    // 定义结构体 llama_token_data_array，包含 token 数据数组的结构
    typedef struct llama_token_data_array {
        llama_token_data * data; // token 数据指针
        size_t size;             // 数组大小
        bool sorted;             // 是否已排序
    } llama_token_data_array;

    // 定义函数指针类型 llama_progress_callback，用于表示进度回调函数
    typedef bool (*llama_progress_callback)(float progress, void *ctx);

    // llama_decode 的输入数据结构 llama_batch
    // 一个 llama_batch 对象可以包含一个或多个序列的输入数据
    // 提供的数组（例如 token, embd, pos 等）必须具有 n_tokens 大小
    //
    // - token  : 输入的 token id（当 embd 为 NULL 时使用）
    // - embd   : token embeddings（大小为 n_embd 的浮点向量）（当 token 为 NULL 时使用）
    // - pos    : 序列中各个 token 的位置
    // - seq_id : 各个 token 所属的序列
    // - logits : 如果为零，则不会输出相应 token 的 logits
    //
    typedef struct llama_batch {
        int32_t n_tokens;       // token 数量

        llama_token  *  token;  // token 数组
        float        *  embd;   // embeddings 数组
        llama_pos    *  pos;    // 位置数组
        int32_t      *  n_seq_id; // 序列 id 数组
        llama_seq_id ** seq_id;   // 序列 id 指针数组
        int8_t       *  logits;   // logits 数组

        // 注意：用于平滑 API 迁移的辅助字段 - 可能在将来被弃用
        //       为了未来代码兼容性，请使用上述字段，并忽略下面的内容
        //
        // pos[i] = all_pos_0 + i*all_pos_1
        //
        llama_pos    all_pos_0;  // 如果 pos == NULL，则使用
        llama_pos    all_pos_1;  // 如果 pos == NULL，则使用
        llama_seq_id all_seq_id; // 如果 seq_id == NULL，则使用
    } llama_batch;
    // 定义枚举类型 llama_model_kv_override_type，用于表示 llama_model_kv_override 结构体中的 tag 类型
    enum llama_model_kv_override_type {
        LLAMA_KV_OVERRIDE_INT, // 整型
        LLAMA_KV_OVERRIDE_FLOAT, // 浮点型
        LLAMA_KV_OVERRIDE_BOOL, // 布尔型
    };

    // 定义结构体 llama_model_kv_override，用于存储模型键值对的覆盖信息
    struct llama_model_kv_override {
        char key[128]; // 键名，最大长度为 128
        enum llama_model_kv_override_type tag; // 键值类型
        union {
            int64_t int_value; // 整型值
            double float_value; // 浮点型值
            bool bool_value; // 布尔型值
        };
    };

    // 定义结构体 llama_model_params，用于存储模型参数信息
    struct llama_model_params {
        int32_t n_gpu_layers; // 存储在 VRAM 中的层数
        enum llama_split_mode split_mode; // 模型在多个 GPU 上的分割模式

        // main_gpu 的解释取决于 split_mode：
        // LLAMA_SPLIT_NONE: 用于整个模型的 GPU
        // LLAMA_SPLIT_ROW: 用于小张量和中间结果的 GPU
        // LLAMA_SPLIT_LAYER: 忽略
        int32_t main_gpu;

        // 每个 GPU 上卸载模型（层或行）的比例，大小为 LLAMA_MAX_DEVICES
        const float * tensor_split;

        // 传入一个介于 0.0 和 1.0 之间的进度值的回调函数。传入 NULL 禁用此功能。
        // 如果提供的 progress_callback 返回 true，则模型加载继续。
        // 如果返回 false，则立即中止模型加载。
        llama_progress_callback progress_callback;

        // 传递给进度回调的上下文指针
        void * progress_callback_user_data;

        // 覆盖模型元数据的键值对
        const struct llama_model_kv_override * kv_overrides;

        // 将布尔值放在一起，以避免在按值复制时出现对齐错误。
        bool vocab_only; // 仅加载词汇表，不加载权重
        bool use_mmap;   // 如果可能，使用 mmap
        bool use_mlock;  // 强制系统将模型保留在 RAM 中
    };
    // 定义了 llama_context_params 结构体，用于存储 LLAMA 模型的上下文参数
    struct llama_context_params {
        uint32_t seed;              // 随机数生成器的种子，-1 表示随机
        uint32_t n_ctx;             // 文本上下文，0 表示从模型中获取
        uint32_t n_batch;           // 提示处理的最大批处理大小
        uint32_t n_threads;         // 用于生成的线程数
        uint32_t n_threads_batch;   // 用于批处理的线程数
        int8_t   rope_scaling_type; // RoPE 缩放类型，来自枚举 llama_rope_scaling_type

        // 参考链接: https://github.com/ggerganov/llama.cpp/pull/2054
        float    rope_freq_base;   // RoPE 基础频率，0 表示从模型中获取
        float    rope_freq_scale;  // RoPE 频率缩放因子，0 表示从模型中获取
        float    yarn_ext_factor;  // YaRN 外推混合因子，负数表示从模型中获取
        float    yarn_attn_factor; // YaRN 幅度缩放因子
        float    yarn_beta_fast;   // YaRN 低校正维度
        float    yarn_beta_slow;   // YaRN 高校正维度
        uint32_t yarn_orig_ctx;    // YaRN 原始上下文大小

        ggml_backend_sched_eval_callback cb_eval;
        void * cb_eval_user_data;

        enum ggml_type type_k; // K 缓存的数据类型
        enum ggml_type type_v; // V 缓存的数据类型

        // 将布尔值放在一起，以避免在按值复制时出现错位。
        bool mul_mat_q;   // 如果为 true，则使用实验性的 mul_mat_q 内核（已弃用 - 始终为 true）
        bool logits_all;  // llama_eval() 调用计算所有对数概率，而不仅仅是最后一个（已弃用 - 设置 llama_batch.logits 代替）
        bool embedding;   // 仅嵌入模式
        bool offload_kqv; // 是否将 KQV 操作（包括 KV 缓存）卸载到 GPU
    };

    // 模型量化参数
    // 定义了一个结构体 llama_model_quantize_params，用于存储量化模型的参数
    typedef struct llama_model_quantize_params {
        int32_t nthread;             // 用于量化的线程数，如果小于等于0，则使用 std::thread::hardware_concurrency()
        enum llama_ftype ftype;      // 量化到这个 llama_ftype 类型
        bool allow_requantize;       // 允许量化非 f32/f16 张量
        bool quantize_output_tensor; // 量化输出权重
        bool only_copy;              // 仅复制张量 - 忽略 ftype、allow_requantize 和 quantize_output_tensor
        bool pure;                   // 禁用 k-quant 混合，将所有张量量化为相同类型
        void * imatrix;              // 指向重要性矩阵数据的指针
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

        // 终结符元素：字符（Unicode 码点）
        LLAMA_GRETYPE_CHAR           = 3,

        // 反向字符（[^a]，[^a-b] [^abc]）
        LLAMA_GRETYPE_CHAR_NOT       = 4,

        // 修改前面的 LLAMA_GRETYPE_CHAR 或 LLAMA_GRETYPE_CHAR_ALT 为包含范围（[a-z]）
        LLAMA_GRETYPE_CHAR_RNG_UPPER = 5,

        // 修改前面的 LLAMA_GRETYPE_CHAR 或 LLAMA_GRETYPE_CHAR_RNG_UPPER 添加一个匹配的备选字符（[ab]，[a-zA]）
        LLAMA_GRETYPE_CHAR_ALT       = 6,
    };

    typedef struct llama_grammar_element {
        enum llama_gretype type;
        uint32_t           value; // Unicode 码点或规则 ID
    } llama_grammar_element;

    // 性能计时信息
    // 定义了一个结构体 llama_timings，包含了一系列时间相关的双精度浮点数和整型变量
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
    
    // 获取默认的模型参数的辅助函数声明
    LLAMA_API struct llama_model_params llama_model_default_params(void);
    // 获取默认的上下文参数的辅助函数声明
    LLAMA_API struct llama_context_params llama_context_default_params(void);
    // 获取默认的模型量化参数的辅助函数声明
    LLAMA_API struct llama_model_quantize_params llama_model_quantize_default_params(void);
    
    // 初始化 llama + ggml 后端，如果 numa 为 true，则使用 NUMA 优化
    // 在程序开始时调用一次
    LLAMA_API void llama_backend_init(bool numa);
    
    // 在程序结束时调用一次，目前仅用于 MPI
    LLAMA_API void llama_backend_free(void);
    
    // 从文件加载模型，返回 llama_model 指针
    LLAMA_API struct llama_model * llama_load_model_from_file(
                             const char * path_model,
            struct llama_model_params     params);
    
    // 释放模型内存
    LLAMA_API void llama_free_model(struct llama_model * model);
    
    // 创建带有模型的新上下文，返回 llama_context 指针
    LLAMA_API struct llama_context * llama_new_context_with_model(
                     struct llama_model * model,
            struct llama_context_params   params);
    
    // 释放所有分配的内存
    LLAMA_API void llama_free(struct llama_context * ctx);
    
    // 返回当前时间的微秒数
    LLAMA_API int64_t llama_time_us(void);
    
    // 返回最大设备数量
    LLAMA_API int32_t  llama_max_devices(void);
    // 返回是否支持内存映射
    LLAMA_API bool llama_mmap_supported (void);
    // 返回是否支持内存锁定
    LLAMA_API bool llama_mlock_supported(void);
    
    // 返回上下文中的模型指针
    LLAMA_API const struct llama_model * llama_get_model(const struct llama_context * ctx);
    
    // 返回上下文中的上下文数量
    LLAMA_API uint32_t llama_n_ctx      (const struct llama_context * ctx);
    // 返回上下文中的批次数量
    LLAMA_API uint32_t llama_n_batch    (const struct llama_context * ctx);
    
    // 返回模型的词汇表类型
    LLAMA_API enum llama_vocab_type llama_vocab_type(const struct llama_model * model);
    
    // 返回模型的词汇表大小
    LLAMA_API int32_t llama_n_vocab    (const struct llama_model * model);
    // 定义一个函数，用于训练模型的上下文
    LLAMA_API int32_t llama_n_ctx_train(const struct llama_model * model);
    
    // 定义一个函数，用于获取模型的嵌入
    LLAMA_API int32_t llama_n_embd(const struct llama_model * model);
    
    // 获取模型的 RoPE 频率缩放因子
    LLAMA_API float llama_rope_freq_scale_train(const struct llama_model * model);
    
    // 用于访问模型的 GGUF 元数据标量值的函数
    // - 函数在成功时返回字符串的长度，失败时返回 -1
    // - 输出字符串始终以空字符结尾，并在失败时清空
    // - 这些函数不支持 GGUF 数组值
    
    // 通过键名获取元数据值作为字符串
    LLAMA_API int32_t llama_model_meta_val_str(const struct llama_model * model, const char * key, char * buf, size_t buf_size);
    
    // 获取元数据键/值对的数量
    LLAMA_API int32_t llama_model_meta_count(const struct llama_model * model);
    
    // 通过索引获取元数据键名
    LLAMA_API int32_t llama_model_meta_key_by_index(const struct llama_model * model, int32_t i, char * buf, size_t buf_size);
    
    // 通过索引获取元数据值作为字符串
    LLAMA_API int32_t llama_model_meta_val_str_by_index(const struct llama_model * model, int32_t i, char * buf, size_t buf_size);
    
    // 获取描述模型类型的字符串
    LLAMA_API int32_t llama_model_desc(const struct llama_model * model, char * buf, size_t buf_size);
    
    // 返回模型中所有张量的总字节大小
    LLAMA_API uint64_t llama_model_size(const struct llama_model * model);
    
    // 返回模型中参数的总数
    LLAMA_API uint64_t llama_model_n_params(const struct llama_model * model);
    
    // 获取一个 llama 模型张量
    LLAMA_API struct ggml_tensor * llama_get_model_tensor(struct llama_model * model, const char * name);
    
    // 成功时返回 0
    // 定义一个函数 llama_model_quantize，用于对模型进行量化
    // fname_inp 是输入模型文件名
    // fname_out 是输出模型文件名
    // params 是 llama_model_quantize_params 结构体指针
    LLAMA_API uint32_t llama_model_quantize(
            const char * fname_inp,
            const char * fname_out,
            const llama_model_quantize_params * params);

    // 应用 LoRA 适配器到加载的模型
    // path_base_model 是用作基础模型的高质量模型的路径
    // 可以为 NULL，表示使用当前加载的模型作为基础模型
    // 在应用新适配器之前，需要重新加载模型，否则适配器将叠加在之前的适配器之上
    // 成功返回 0
    LLAMA_API DEPRECATED(int32_t llama_apply_lora_from_file(
            struct llama_context * ctx,
                      const char * path_lora,
                           float   scale,
                      const char * path_base_model,
                         int32_t   n_threads),
            "use llama_model_apply_lora_from_file instead");

    // 使用文件中的 LoRA 适配器应用到模型
    // model 是 llama_model 结构体指针
    // path_lora 是 LoRA 适配器文件的路径
    // scale 是缩放比例
    // path_base_model 是基础模型的路径
    // n_threads 是线程数
    LLAMA_API int32_t llama_model_apply_lora_from_file(
            const struct llama_model * model,
                      const char * path_lora,
                           float   scale,
                      const char * path_base_model,
                         int32_t   n_threads);

    //
    // KV cache
    //

    // KV 缓存视图中与单个单元格相关联的信息
    struct llama_kv_cache_view_cell {
        // 此单元格的位置，考虑 KV 缓存的偏移量
        // 如果单元格未填充，则可能为负值
        llama_pos pos;
    };

    // KV 缓存的可更新视图
    // 定义一个结构体，用于表示 KV 缓存视图
    struct llama_kv_cache_view {
        // KV 缓存单元的数量，与上下文大小相同
        int32_t n_cells;

        // 每个单元中可以存在的最大序列数。如果单元中的序列数超过此值，不会报错，但在视图中将不可见
        int32_t n_max_seq;

        // 缓存中的令牌数量。例如，如果有两个填充的单元，第一个单元中有1个序列ID，第二个单元中有2个序列ID，则总共有3个令牌
        int32_t token_count;

        // 填充的缓存单元数量
        int32_t used_cells;

        // 缓存中最大连续空槽的数量
        int32_t max_contiguous;

        // 最大连续空槽范围的起始索引。当缓存已满时可能为负值
        int32_t max_contiguous_idx;

        // 每个单元的信息
        struct llama_kv_cache_view_cell * cells;

        // 每个单元的序列。每个单元将有 n_max_seq 个项目
        llama_seq_id * cells_sequences;
    };

    // 创建一个空的 KV 缓存视图（仅用于调试目的）
    LLAMA_API struct llama_kv_cache_view llama_kv_cache_view_init(const struct llama_context * ctx, int32_t n_max_seq);

    // 释放 KV 缓存视图（仅用于调试目的）
    LLAMA_API void llama_kv_cache_view_free(struct llama_kv_cache_view * view);

    // 使用当前 KV 缓存的状态更新 KV 缓存视图结构（仅用于调试目的）
    LLAMA_API void llama_kv_cache_view_update(const struct llama_context * ctx, struct llama_kv_cache_view * view);

    // 返回 KV 缓存中的令牌数量（较慢，仅用于调试）
    // 如果一个 KV 单元分配了多个序列，它将被多次计算
    LLAMA_API int32_t llama_get_kv_cache_token_count(const struct llama_context * ctx);
    // 返回已使用的 KV 单元格数量（即至少有一个序列分配给它们的单元格数）
    LLAMA_API int32_t llama_get_kv_cache_used_cells(const struct llama_context * ctx);
    
    // 清空 KV 缓存
    LLAMA_API void llama_kv_cache_clear(
            struct llama_context * ctx);
    
    // 移除所有属于指定序列并且位置在 [p0, p1) 范围内的令牌
    // seq_id < 0 : 匹配任何序列
    // p0 < 0     : [0,  p1]
    // p1 < 0     : [p0, 无穷)
    LLAMA_API void llama_kv_cache_seq_rm(
            struct llama_context * ctx,
                    llama_seq_id   seq_id,
                       llama_pos   p0,
                       llama_pos   p1);
    
    // 将所有属于指定序列的令牌复制到另一个序列
    // 注意，这不会分配额外的 KV 缓存内存 - 它只是将令牌分配给新序列
    // p0 < 0 : [0,  p1]
    // p1 < 0 : [p0, 无穷)
    LLAMA_API void llama_kv_cache_seq_cp(
            struct llama_context * ctx,
                    llama_seq_id   seq_id_src,
                    llama_seq_id   seq_id_dst,
                       llama_pos   p0,
                       llama_pos   p1);
    
    // 移除所有不属于指定序列的令牌
    LLAMA_API void llama_kv_cache_seq_keep(
            struct llama_context * ctx,
                    llama_seq_id   seq_id);
    
    // 将相对位置 "delta" 添加到所有属于指定序列并且位置在 [p0, p1) 范围内的令牌
    // 如果 KV 缓存是 RoPEd，那么 KV 数据会相应更新
    // p0 < 0 : [0,  p1]
    // p1 < 0 : [p0, 无穷)
    LLAMA_API void llama_kv_cache_seq_shift(
            struct llama_context * ctx,
                    llama_seq_id   seq_id,
                       llama_pos   p0,
                       llama_pos   p1,
                       llama_pos   delta);
    
    // 通过大于 1 的因子 `d` 对位置进行整数除法
    // 如果 KV 缓存被 RoPEd，则相应更新 KV 数据
    // p0 < 0 : [0,  p1]
    // p1 < 0 : [p0, inf)
    LLAMA_API void llama_kv_cache_seq_div(
            struct llama_context * ctx,
                    llama_seq_id   seq_id,
                       llama_pos   p0,
                       llama_pos   p1,
                             int   d);

    //
    // State / sessions
    //

    // 返回状态（rng、logits、embedding 和 kv_cache）的最大字节大小 - 在压缩令牌后通常会更小
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
    // Decoding
    //

    // 运行 LLAMA 推断以获取下一个令牌的 logits 和概率。
    // tokens + n_tokens 是要处理的新令牌批次
    // n_past 是要使用的先前评估调用的令牌数
    // 成功返回 0
    // 使用 llama_decode() 替代 llama_eval()，此函数已废弃
    LLAMA_API DEPRECATED(int llama_eval(
            struct llama_context * ctx,
                     llama_token * tokens,
                         int32_t   n_tokens,
                         int32_t   n_past),
            "use llama_decode() instead");

    // 与 llama_eval 相同，但直接使用浮点矩阵输入
    // 使用 llama_decode() 替代 llama_eval_embd()，此函数已废弃
    LLAMA_API DEPRECATED(int llama_eval_embd(
            struct llama_context * ctx,
                           float * embd,
                         int32_t   n_tokens,
                         int32_t   n_past),
            "use llama_decode() instead");

    // 返回从 pos_0 开始的单个 token 序列的批次
    //
    // 注意：这是一个辅助函数，用于帮助过渡到新的批处理 API - 避免使用它
    //
    LLAMA_API struct llama_batch llama_batch_get_one(
                  llama_token * tokens,
                      int32_t   n_tokens,
                    llama_pos   pos_0,
                 llama_seq_id   seq_id);

    // 在堆上分配一个可以容纳最多 n_tokens 个 token 的批次
    // 每个 token 最多可以分配 n_seq_max 个序列 id
    // 必须使用 llama_batch_free() 释放批次
    // 如果 embd != 0，则 llama_batch.embd 将分配大小为 n_tokens * embd * sizeof(float) 的空间
    // 否则，将分配 llama_token 的 n_tokens 大小的空间
    // 其余的 llama_batch 成员将分配 n_tokens 大小的空间
    // 所有成员都将保持未初始化状态
    LLAMA_API struct llama_batch llama_batch_init(
            int32_t n_tokens,
            int32_t embd,
            int32_t n_seq_max);

    // 释放使用 llama_batch_init() 分配的 token 批次
    LLAMA_API void llama_batch_free(struct llama_batch batch);

    // 正返回值并不意味着致命错误，而是一个警告。
    //   0 - 成功
    //   1 - 批处理中找不到 KV 槽位（尝试减少批处理大小或增加上下文）
    // < 0 - 错误
    LLAMA_API int32_t llama_decode(
            struct llama_context * ctx,
              struct llama_batch   batch);

    // 设置用于解码的线程数
    // n_threads 是用于生成（单个标记）的线程数
    // n_threads_batch 是用于提示和批处理（多个标记）的线程数
    LLAMA_API void llama_set_n_threads(struct llama_context * ctx, uint32_t n_threads, uint32_t n_threads_batch);

    // 上一次调用 llama_eval() 获取的标记对数
    // 上一个标记的对数存储在最后一行
    // 对于 llama_batch.logits[i] == 0 的对数是未定义的
    // 行数：使用 llama_batch 提供的标记数
    // 列数：n_vocab
    LLAMA_API float * llama_get_logits(struct llama_context * ctx);

    // 第 i 个标记的对数。等同于：
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

    // 特殊标记
    LLAMA_API llama_token llama_token_bos(const struct llama_model * model); // 句子开头
    LLAMA_API llama_token llama_token_eos(const struct llama_model * model); // 句子结尾
    LLAMA_API llama_token llama_token_nl (const struct llama_model * model); // 下一行

    // 如果未知则返回 -1，为真返回 1，为假返回 0。
    // 添加 BOS（Beginning of Sentence）标记到模型中，返回整型结果
    LLAMA_API int32_t llama_add_bos_token(const struct llama_model * model);
    
    // 如果未知返回-1，为真返回1，为假返回0
    // 向模型中添加 EOS（End of Sentence）标记
    LLAMA_API int32_t llama_add_eos_token(const struct llama_model * model);
    
    // 返回填充前缀的标记
    LLAMA_API llama_token llama_token_prefix(const struct llama_model * model); // Beginning of infill prefix
    // 返回填充中间部分的标记
    LLAMA_API llama_token llama_token_middle(const struct llama_model * model); // Beginning of infill middle
    // 返回填充后缀的标记
    LLAMA_API llama_token llama_token_suffix(const struct llama_model * model); // Beginning of infill suffix
    // 返回填充中间结束的标记
    LLAMA_API llama_token llama_token_eot   (const struct llama_model * model); // End of infill middle
    
    //
    // Tokenization
    //
    
    /// @details 将提供的文本转换为标记
    /// @param tokens 指针必须足够大以容纳结果标记
    /// @return 成功时返回标记数量，不超过 n_max_tokens
    /// @return 失败时返回负数 - 将返回的标记数量
    /// @param special 允许标记化特殊和/或控制标记，否则将被视为纯文本
    ///                不插入前导空格
    LLAMA_API int32_t llama_tokenize(
        const struct llama_model * model,
                      const char * text,
                         int32_t   text_len,
                     llama_token * tokens,
                         int32_t   n_max_tokens,
                            bool   add_bos,
                            bool   special);
    
    // 标记 ID -> 片段
    // 使用提供的上下文中的词汇表
    // 不向缓冲区写入空终止符
    // 用户代码负责在解码多个标记时去除第一个非 BOS 标记的前导空格
    // 将 LLAMA 令牌转换为片段，返回片段的长度
    LLAMA_API int32_t llama_token_to_piece(
              const struct llama_model * model,
                           llama_token   token,
                                  char * buf,
                               int32_t   length);
    
    //
    // 语法
    //
    
    // 初始化 LLAMA 语法对象
    LLAMA_API struct llama_grammar * llama_grammar_init(
            const llama_grammar_element ** rules,
                                 size_t    n_rules,
                                 size_t    start_rule_index);
    
    // 释放 LLAMA 语法对象占用的内存
    LLAMA_API void llama_grammar_free(struct llama_grammar * grammar);
    
    // 复制 LLAMA 语法对象
    LLAMA_API struct llama_grammar * llama_grammar_copy(const struct llama_grammar * grammar);
    
    //
    // 采样函数
    //
    
    // 设置当前的随机数生成器种子
    LLAMA_API void llama_set_rng_seed(struct llama_context * ctx, uint32_t seed);
    
    /// @details 在 CTRL 学术论文 https://arxiv.org/abs/1909.05858 中描述的重复惩罚，带有负对数修正。
    /// @details 在 OpenAI API https://platform.openai.com/docs/api-reference/parameter-details 中描述的频率和存在惩罚。
    LLAMA_API void llama_sample_repetition_penalties(
            struct llama_context * ctx,
          llama_token_data_array * candidates,
               const llama_token * last_tokens,
                          size_t   penalty_last_n,
                           float   penalty_repeat,
                           float   penalty_freq,
                           float   penalty_present);
    
    /// @details 将分类器无关的指导应用于对数，如学术论文“Stay on topic with Classifier-Free Guidance” https://arxiv.org/abs/2306.17806 中描述
    /// @param logits 从原始生成上下文中提取的对数
    /// @param logits_guidance 从相同模型的不同上下文中提取的对数。除了开头的负提示外，它应该将所有生成和用户输入令牌从主上下文复制过来。
    /// @param scale Guidance strength. 1.0f means no guidance. Higher values mean stronger guidance.
    LLAMA_API void llama_sample_apply_guidance(
              struct llama_context * ctx,
                             float * logits,
                             float * logits_guidance,
                             float   scale);

    LLAMA_API DEPRECATED(void llama_sample_classifier_free_guidance(
              struct llama_context * ctx,
            llama_token_data_array * candidates,
              struct llama_context * guidance_ctx,
                             float   scale),
              "use llama_sample_apply_guidance() instead");

    /// @details Sorts candidate tokens by their logits in descending order and calculate probabilities based on logits.
    LLAMA_API void llama_sample_softmax(
            struct llama_context * ctx,
          llama_token_data_array * candidates);

    /// @details Top-K sampling described in academic paper "The Curious Case of Neural Text Degeneration" https://arxiv.org/abs/1904.09751
    LLAMA_API void llama_sample_top_k(
            struct llama_context * ctx,
          llama_token_data_array * candidates,
                         int32_t   k,
                          size_t   min_keep);

    /// @details Nucleus sampling described in academic paper "The Curious Case of Neural Text Degeneration" https://arxiv.org/abs/1904.09751
    LLAMA_API void llama_sample_top_p(
            struct llama_context * ctx,
          llama_token_data_array * candidates,
                           float   p,
                          size_t   min_keep);

    /// @details Minimum P sampling as described in https://github.com/ggerganov/llama.cpp/pull/3841
    LLAMA_API void llama_sample_min_p(
            struct llama_context * ctx,
          llama_token_data_array * candidates,
                           float   p,
                          size_t   min_keep);
    /// @details 使用尾部免费抽样描述在 https://www.trentonbricken.com/Tail-Free-Sampling/ 中。
    LLAMA_API void llama_sample_tail_free(
            struct llama_context * ctx,
          llama_token_data_array * candidates,
                           float   z,
                          size_t   min_keep);
    
    /// @details 本地典型抽样实现描述在论文 https://arxiv.org/abs/2202.00666 中。
    LLAMA_API void llama_sample_typical(
            struct llama_context * ctx,
          llama_token_data_array * candidates,
                           float   p,
                          size_t   min_keep);
    
    /// @details 动态温度实现描述在论文 https://arxiv.org/abs/2309.02772 中。
    LLAMA_API void llama_sample_entropy(
            struct llama_context * ctx,
          llama_token_data_array * candidates_p,
                           float   min_temp,
                           float   max_temp,
                           float   exponent_val);
    
    LLAMA_API void llama_sample_temp(
            struct llama_context * ctx,
          llama_token_data_array * candidates,
                           float   temp);
    
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
    
    /// @details Mirostat 1.0 算法描述在论文 https://arxiv.org/abs/2007.14966 中。使用 tokens 而不是 words。
    /// @param candidates 包含候选 tokens、它们的概率 (p) 和对当前生成文本位置的 log-odds (logit) 的 `llama_token_data` 向量。
    /// @param tau  The target cross-entropy (or surprise) value you want to achieve for the generated text. A higher value corresponds to more surprising or less predictable text, while a lower value corresponds to less surprising or more predictable text.
    /// @param eta The learning rate used to update `mu` based on the error between the target and observed surprisal of the sampled word. A larger learning rate will cause `mu` to be updated more quickly, while a smaller learning rate will result in slower updates.
    /// @param m The number of tokens considered in the estimation of `s_hat`. This is an arbitrary value that is used to calculate `s_hat`, which in turn helps to calculate the value of `k`. In the paper, they use `m = 100`, but you can experiment with different values to see how it affects the performance of the algorithm.
    /// @param mu Maximum cross-entropy. This value is initialized to be twice the target cross-entropy (`2 * tau`) and is updated in the algorithm based on the error between the target and observed surprisal.
    LLAMA_API llama_token llama_sample_token_mirostat(
            struct llama_context * ctx,
          llama_token_data_array * candidates,
                           float   tau,
                           float   eta,
                         int32_t   m,
                           float * mu);

    /// @details Mirostat 2.0 algorithm described in the paper https://arxiv.org/abs/2007.14966. Uses tokens instead of words.
    /// @param candidates A vector of `llama_token_data` containing the candidate tokens, their probabilities (p), and log-odds (logit) for the current position in the generated text.
    /// @param tau  The target cross-entropy (or surprise) value you want to achieve for the generated text. A higher value corresponds to more surprising or less predictable text, while a lower value corresponds to less surprising or more predictable text.
    /// @param eta 学习率，用于根据采样词的目标和观察到的惊讶度之间的误差来更新 `mu`。学习率越大，`mu` 更新越快，学习率越小，更新越慢。
    /// @param mu 最大交叉熵。此值初始化为目标交叉熵的两倍（`2 * tau`），并根据算法中目标和观察到的惊讶度之间的误差进行更新。
    LLAMA_API llama_token llama_sample_token_mirostat_v2(
            struct llama_context * ctx,
          llama_token_data_array * candidates,
                           float   tau,
                           float   eta,
                           float * mu);

    /// @details 选择具有最高概率的令牌。
    ///          不计算令牌概率。请改用 llama_sample_softmax()。
    LLAMA_API llama_token llama_sample_token_greedy(
            struct llama_context * ctx,
          llama_token_data_array * candidates);

    /// @details 根据候选项的概率随机选择一个令牌。
    LLAMA_API llama_token llama_sample_token(
            struct llama_context * ctx,
          llama_token_data_array * candidates);

    /// @details 将采样的令牌接受到语法中
    LLAMA_API void llama_grammar_accept_token(
            struct llama_context * ctx,
            struct llama_grammar * grammar,
                     llama_token   token);

    //
    // Beam search
    //

    struct llama_beam_view {
        const llama_token * tokens;

        size_t n_tokens;
        float  p;        // 累积波束概率（相对于所有波束重新归一化）
        bool   eob;      // 当波束到达结束时，回调应将此设置为 true。
    };

    // 传递给 beam_search_callback 函数。
    // 每当 0 < common_prefix_length 时，应从任何波束中复制此数量的令牌
    // 定义结构体 llama_beams_state，用于保存 beam_search_callback 函数的状态信息
    // beam_views: 指向 llama_beam_view 结构体的指针
    // n_beams: beam_views[] 中元素的数量
    // common_prefix_length: 所有 beams 共享的前缀 token 的最大长度
    // last_call: 如果这是最后一次回调调用，则为 true
    struct llama_beams_state {
        struct llama_beam_view * beam_views;

        size_t n_beams;               // beam_views[] 中元素的数量
        size_t common_prefix_length;  // 所有 beams 共享的前缀 token 的最大长度
        bool   last_call;             // 如果这是最后一次回调调用，则为 true
    };

    // 定义函数指针类型 llama_beam_search_callback_fn_t，用于指向 beam_search_callback 函数
    // void* callback_data 是传递给 llama_beam_search 的自定义数据，随后传递给 beam_search_callback
    // 这避免了在回调中使用全局变量
    typedef void (*llama_beam_search_callback_fn_t)(void * callback_data, struct llama_beams_state);

    /// @details 通过确定性地返回由 beam search 构建的整个句子
    /// @param ctx 指向 llama_context 的指针
    /// @param callback 每次 beam_search 循环迭代时调用，传入 beams_state
    /// @param callback_data 一个简单传递给 callback 的指针
    /// @param n_beams 要使用的 beams 数量
    /// @param n_past 已评估的 token 数量
    /// @param n_predict 最大要预测的 token 数量。EOS 可能会更早出现
    LLAMA_API void llama_beam_search(
                   struct llama_context * ctx,
        llama_beam_search_callback_fn_t   callback,
                                   void * callback_data,
                                 size_t   n_beams,
                                int32_t   n_past,
                                int32_t   n_predict);

    // 获取性能信息
    LLAMA_API struct llama_timings llama_get_timings(struct llama_context * ctx);

    // 打印性能信息
    LLAMA_API void llama_print_timings(struct llama_context * ctx);
    // 重置 LLAMA 库中与时间相关的统计信息
    LLAMA_API void llama_reset_timings(struct llama_context * ctx);
    
    // 打印系统信息
    LLAMA_API const char * llama_print_system_info(void);
    
    // 设置所有未来日志事件的回调函数
    // 如果未调用此函数或提供 NULL，则所有内容将输出到 stderr
    LLAMA_API void llama_log_set(ggml_log_callback log_callback, void * user_data);
    
    // 将 LLAMA 上下文中的时间信息以 YAML 格式输出到文件流
    LLAMA_API void llama_dump_timing_info_yaml(FILE * stream, const struct llama_context * ctx);
#ifdef __cplusplus
}
#endif

// 如果是 C++ 环境，结束 extern "C" 块
#ifdef LLAMA_API_INTERNAL

#include <vector>
#include <string>

// 定义结构体 ggml_tensor
struct ggml_tensor;

// 声明 llama_internal_get_tensor_map 函数，用于 llama.cpp 实现，仅供测试和基准测试使用
const std::vector<std::pair<std::string, struct ggml_tensor *>> & llama_internal_get_tensor_map(
    struct llama_context * ctx
);

#endif // LLAMA_API_INTERNAL

#endif // LLAMA_H
```