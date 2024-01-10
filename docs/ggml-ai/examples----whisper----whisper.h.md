# `ggml\examples\whisper\whisper.h`

```
#ifndef WHISPER_H
#define WHISPER_H

#include "ggml.h"

#include <stddef.h>
#include <stdint.h>
#include <stdbool.h>

#ifdef __GNUC__
#    define WHISPER_DEPRECATED(func, hint) func __attribute__((deprecated(hint)))
#elif defined(_MSC_VER)
#    define WHISPER_DEPRECATED(func, hint) __declspec(deprecated(hint)) func
#else
#    define WHISPER_DEPRECATED(func, hint) func
#endif

#ifdef WHISPER_SHARED
#    ifdef _WIN32
#        ifdef WHISPER_BUILD
#            define WHISPER_API __declspec(dllexport)
#        else
#            define WHISPER_API __declspec(dllimport)
#        endif
#    else
#        define WHISPER_API __attribute__ ((visibility ("default")))
#    endif
#else
#    define WHISPER_API
#endif

#define WHISPER_SAMPLE_RATE 16000
#define WHISPER_N_FFT       400
#define WHISPER_HOP_LENGTH  160
#define WHISPER_CHUNK_SIZE  30

#ifdef __cplusplus
extern "C" {
#endif

    //
    // C interface
    //
    // The following interface is thread-safe as long as the sample whisper_context is not used by multiple threads
    // concurrently.
    //
    // Basic usage:
    //
    //     #include "whisper.h"
    //
    //     ...
    //
    //     whisper_context_params cparams = whisper_context_default_params();
    //     // 使用默认参数初始化 whisper_context_params 结构
    //
    //     struct whisper_context * ctx = whisper_init_from_file_with_params("/path/to/ggml-base.en.bin", cparams);
    //     // 使用指定参数初始化 whisper_context 结构
    //
    //     if (whisper_full(ctx, wparams, pcmf32.data(), pcmf32.size()) != 0) {
    //         fprintf(stderr, "failed to process audio\n");
    //         return 7;
    //     }
    //     // 对音频进行全处理，如果失败则输出错误信息并返回 7
    //
    //     const int n_segments = whisper_full_n_segments(ctx);
    //     // 获取处理后的音频片段数量
    //
    //     for (int i = 0; i < n_segments; ++i) {
    //         const char * text = whisper_full_get_segment_text(ctx, i);
    //         // 获取处理后的音频片段文本
    //         printf("%s", text);
    //     }
    //
    //     whisper_free(ctx);
    //     // 释放 whisper_context 结构
    //
    //     ...
    //
    // This is a demonstration of the most straightforward usage of the library.
    // "pcmf32"包含32位浮点格式的原始音频数据。
    //
    // 该接口还允许对计算进行更精细的控制，但需要更深入地了解模型的工作原理。
    //

    // 定义whisper_context结构体
    struct whisper_context;
    // 定义whisper_state结构体
    struct whisper_state;
    // 定义whisper_full_params结构体
    struct whisper_full_params;

    // 定义whisper_pos为int32_t类型
    typedef int32_t whisper_pos;
    // 定义whisper_token为int32_t类型
    typedef int32_t whisper_token;
    // 定义whisper_seq_id为int32_t类型
    typedef int32_t whisper_seq_id;

    // 定义whisper_context_params结构体
    struct whisper_context_params {
        bool  use_gpu;  // 是否使用GPU
    };

    // 定义whisper_token_data结构体
    typedef struct whisper_token_data {
        whisper_token id;  // token id
        whisper_token tid; // 强制时间戳token id

        float p;           // token的概率
        float plog;        // token的对数概率
        float pt;          // 时间戳token的概率
        float ptsum;       // 所有时间戳token的概率之和

        // token级别的时间戳数据
        // 如果没有计算token级别的时间戳，则不要使用
        int64_t t0;        // token的开始时间
        int64_t t1;        // token的结束时间

        float vlen;        // token的语音长度
    } whisper_token_data;

    // 定义whisper_model_loader结构体
    typedef struct whisper_model_loader {
        void * context;  // 上下文

        size_t (*read)(void * ctx, void * output, size_t read_size);  // 读取函数指针
        bool    (*eof)(void * ctx);  // 判断是否到达文件末尾函数指针
        void  (*close)(void * ctx);  // 关闭函数指针
    } whisper_model_loader;

    // 语法元素类型
    // 定义枚举类型 whisper_gretype，表示语法规则类型
    enum whisper_gretype {
        // 规则定义结束
        WHISPER_GRETYPE_END            = 0,

        // 规则的替代定义开始
        WHISPER_GRETYPE_ALT            = 1,

        // 非终结符元素：引用其他规则
        WHISPER_GRETYPE_RULE_REF       = 2,

        // 终结符元素：字符（Unicode 码点）
        WHISPER_GRETYPE_CHAR           = 3,

        // 反向字符（[^a], [^a-b] [^abc]）
        WHISPER_GRETYPE_CHAR_NOT       = 4,

        // 修改前面的 WHISPER_GRETYPE_CHAR 或 LLAMA_GRETYPE_CHAR_ALT，使其成为包含范围（[a-z]）
        WHISPER_GRETYPE_CHAR_RNG_UPPER = 5,

        // 修改前面的 WHISPER_GRETYPE_CHAR 或 WHISPER_GRETYPE_CHAR_RNG_UPPER，添加一个要匹配的备选字符（[ab], [a-zA]）
        WHISPER_GRETYPE_CHAR_ALT       = 6,
    };

    // 定义结构体 whisper_grammar_element，表示语法元素
    typedef struct whisper_grammar_element {
        enum whisper_gretype type; // 元素类型
        uint32_t             value; // Unicode 码点或规则 ID
    } whisper_grammar_element;

    // 加载 ggml whisper 模型的各种函数
    // 为模型分配（几乎）所有需要的内存
    // 失败时返回 NULL
    WHISPER_API struct whisper_context * whisper_init_from_file_with_params  (const char * path_model,              struct whisper_context_params params);
    WHISPER_API struct whisper_context * whisper_init_from_buffer_with_params(void * buffer, size_t buffer_size,    struct whisper_context_params params);
    WHISPER_API struct whisper_context * whisper_init_with_params            (struct whisper_model_loader * loader, struct whisper_context_params params);

    // 这些函数与上面的函数相同，但上下文的内部状态不会自动分配
    // 调用者有责任使用 whisper_init_state() 分配状态 (#523)
    # 从文件路径和参数初始化 whisper_context 结构体，不保存状态
    WHISPER_API struct whisper_context * whisper_init_from_file_with_params_no_state  (const char * path_model,              struct whisper_context_params params);
    # 从缓冲区和参数初始化 whisper_context 结构体，不保存状态
    WHISPER_API struct whisper_context * whisper_init_from_buffer_with_params_no_state(void * buffer, size_t buffer_size,    struct whisper_context_params params);
    # 使用参数初始化 whisper_context 结构体，不保存状态
    WHISPER_API struct whisper_context * whisper_init_with_params_no_state            (struct whisper_model_loader * loader, struct whisper_context_params params);
    
    # 初始化 whisper_context 结构体，从文件路径，已废弃，建议使用 whisper_init_from_file_with_params 代替
    WHISPER_DEPRECATED(
        WHISPER_API struct whisper_context * whisper_init_from_file(const char * path_model),
        "use whisper_init_from_file_with_params instead"
    );
    # 初始化 whisper_context 结构体，从缓冲区，已废弃，建议使用 whisper_init_from_buffer_with_params 代替
    WHISPER_DEPRECATED(
        WHISPER_API struct whisper_context * whisper_init_from_buffer(void * buffer, size_t buffer_size),
        "use whisper_init_from_buffer_with_params instead"
    );
    # 初始化 whisper_context 结构体，已废弃，建议使用 whisper_init_with_params 代替
    WHISPER_DEPRECATED(
        WHISPER_API struct whisper_context * whisper_init(struct whisper_model_loader * loader),
        "use whisper_init_with_params instead"
    );
    # 初始化 whisper_context 结构体，从文件路径，不保存状态，已废弃，建议使用 whisper_init_from_file_with_params_no_state 代替
    WHISPER_DEPRECATED(
        WHISPER_API struct whisper_context * whisper_init_from_file_no_state(const char * path_model),
        "use whisper_init_from_file_with_params_no_state instead"
    );
    # 初始化 whisper_context 结构体，从缓冲区，不保存状态，已废弃，建议使用 whisper_init_from_buffer_with_params_no_state 代替
    WHISPER_DEPRECATED(
        WHISPER_API struct whisper_context * whisper_init_from_buffer_no_state(void * buffer, size_t buffer_size),
        "use whisper_init_from_buffer_with_params_no_state instead"
    );
    # 初始化 whisper_context 结构体，不保存状态，已废弃，建议使用 whisper_init_with_params_no_state 代替
    WHISPER_DEPRECATED(
        WHISPER_API struct whisper_context * whisper_init_no_state(struct whisper_model_loader * loader),
        "use whisper_init_with_params_no_state instead"
    );
    
    # 初始化 whisper_state 结构体，给定 whisper_context 结构体
    WHISPER_API struct whisper_state * whisper_init_state(struct whisper_context * ctx);
    
    # 给定上下文，启用使用 OpenVINO 进行编码推断
    # model_path: 可选的 OpenVINO 编码器 IR 模型路径。如果设置为 nullptr，
    #             路径将从传递的 ggml 模型路径生成
    // 初始化 OpenVINO 编码器，加载模型并设置设备和缓存目录
    // 如果 'path_model' 是 "/path/to/ggml-base.en.bin"，则 OpenVINO IR 模型路径将被假定为 "/path/to/ggml-base.en-encoder-openvino.xml"
    // device: 在其上运行推理的 OpenVINO 设备 ("CPU", "GPU" 等)
    // cache_dir: 可选的缓存目录，可以通过缓存编译的 'blobs' 来加快初始化时间，特别是对于 GPU
    // 如果未使用，设置为 nullptr
    // 成功返回 0。如果在构建中未启用 OpenVINO，则简单返回 1
    WHISPER_API int whisper_ctx_init_openvino_encoder(
        struct whisper_context * ctx,
                    const char * model_path,
                    const char * device,
                    const char * cache_dir);

    // 释放所有分配的内存
    WHISPER_API void whisper_free      (struct whisper_context * ctx);
    WHISPER_API void whisper_free_state(struct whisper_state * state);
    WHISPER_API void whisper_free_params(struct whisper_full_params * params);
    WHISPER_API void whisper_free_context_params(struct whisper_context_params * params);

    // 将原始 PCM 音频转换为对数梅尔频谱图
    // 结果频谱图存储在提供的 whisper 上下文的默认状态中
    // 成功返回 0
    WHISPER_API int whisper_pcm_to_mel(
            struct whisper_context * ctx,
                       const float * samples,
                               int   n_samples,
                               int   n_threads);

    // 使用状态将原始 PCM 音频转换为对数梅尔频谱图
    // 结果频谱图存储在提供的 whisper 上下文的状态中
    // 成功返回 0
    WHISPER_API int whisper_pcm_to_mel_with_state(
            struct whisper_context * ctx,
              struct whisper_state * state,
                       const float * samples,
                               int   n_samples,
                               int   n_threads);
    // 将原始 PCM 音频转换为对数梅尔频谱，但应用相位估计器以加速音频 x2。
    // 将生成的频谱存储在提供的 Whisper 上下文的默认状态中。
    // 成功时返回 0
    WHISPER_API int whisper_pcm_to_mel_phase_vocoder(
        struct whisper_context * ctx,  // 指向 Whisper 上下文结构体的指针
                   const float * samples,  // 指向原始 PCM 音频样本的指针
                           int   n_samples,  // 原始 PCM 音频样本的数量
                           int   n_threads);  // 线程数量
    
    // 将原始 PCM 音频转换为对数梅尔频谱，但应用相位估计器以加速音频 x2，并将结果存储在提供的 Whisper 上下文的默认状态中。
    // 成功时返回 0
    WHISPER_API int whisper_pcm_to_mel_phase_vocoder_with_state(
        struct whisper_context * ctx,  // 指向 Whisper 上下文结构体的指针
          struct whisper_state * state,  // 指向 Whisper 状态结构体的指针
                   const float * samples,  // 指向原始 PCM 音频样本的指针
                           int   n_samples,  // 原始 PCM 音频样本的数量
                           int   n_threads);  // 线程数量
    
    // 可用于在提供的 Whisper 上下文的默认状态中设置自定义对数梅尔频谱。
    // 如果要提供自己的对数梅尔频谱，请使用此函数，而不是 whisper_pcm_to_mel()。
    // n_mel 必须为 80
    // 成功时返回 0
    WHISPER_API int whisper_set_mel(
            struct whisper_context * ctx,  // 指向 Whisper 上下文结构体的指针
                       const float * data,  // 指向对数梅尔频谱数据的指针
                               int   n_len,  // 数据长度
                               int   n_mel);  // 对数梅尔频谱的数量
    
    // 在提供的 Whisper 上下文的默认状态中设置自定义对数梅尔频谱。
    // 如果要提供自己的对数梅尔频谱，请使用此函数，而不是 whisper_pcm_to_mel()。
    // 成功时返回 0
    WHISPER_API int whisper_set_mel_with_state(
            struct whisper_context * ctx,  // 指向 Whisper 上下文结构体的指针
              struct whisper_state * state,  // 指向 Whisper 状态结构体的指针
                       const float * data,  // 指向对数梅尔频谱数据的指针
                               int   n_len,  // 数据长度
                               int   n_mel);  // 对数梅尔频谱的数量
    
    // 在提供的 Whisper 上下文的默认状态中运行 Whisper 编码器。
    // 确保先调用 whisper_pcm_to_mel() 或 whisper_set_mel()。
    // offset 可用于指定频谱中第一帧的偏移量。
    // 成功时返回 0
    WHISPER_API int whisper_encode(
            struct whisper_context * ctx,  // 指向 Whisper 上下文结构体的指针
                               int   offset,  // 第一帧的偏移量
                               int   n_threads);  // 线程数量
    // 使用状态进行Whisper编码，返回下一个标记的logits和概率
    // 确保先调用whisper_encode()
    // tokens + n_tokens是解码器的上下文
    // n_past是从先前解码器调用中使用的标记数
    // 成功返回0
    // TODO: 添加对多个解码器的支持
    WHISPER_API int whisper_decode(
            struct whisper_context * ctx,
               const whisper_token * tokens,
                               int   n_tokens,
                               int   n_past,
                               int   n_threads);
    
    // 使用状态进行Whisper解码，返回下一个标记的logits和概率
    // tokens指针必须足够大以容纳生成的标记
    // 成功返回标记数，不超过n_max_tokens
    // 失败返回-1
    // TODO: 不确定是否正确
    WHISPER_API int whisper_tokenize(
            struct whisper_context * ctx,
                        const char * text,
                     whisper_token * tokens,
                               int   n_max_tokens);
    
    // 最大语言ID（即可用语言的数量-1）
    WHISPER_API int whisper_lang_max_id();
    
    // 返回指定语言的ID，如果未找到则返回-1
    // 例如:
    //   "de" -> 2
    //   "german" -> 2
    WHISPER_API int whisper_lang_id(const char * lang);
    // 返回指定语言id的短字符串（例如2 -> "de"），如果找不到则返回nullptr
    WHISPER_API const char * whisper_lang_str(int id);
    
    // 返回指定语言名称的短字符串（例如2 -> "german"），如果找不到则返回nullptr
    WHISPER_API const char * whisper_lang_str_full(int id);
    
    // 使用偏移量为offset_ms的mel数据尝试自动检测所说的语言
    // 确保先调用whisper_pcm_to_mel()或whisper_set_mel()
    // 返回顶部语言id，失败时返回负数
    // 如果不为null，则用所有语言的概率填充lang_probs数组
    // 数组的大小必须为whisper_lang_max_id() + 1
    // 参考：https://github.com/openai/whisper/blob/main/whisper/decoding.py#L18-L69
    WHISPER_API int whisper_lang_auto_detect(
            struct whisper_context * ctx,
                               int   offset_ms,
                               int   n_threads,
                             float * lang_probs);
    
    WHISPER_API int whisper_lang_auto_detect_with_state(
            struct whisper_context * ctx,
              struct whisper_state * state,
                               int   offset_ms,
                               int   n_threads,
                             float * lang_probs);
    
    WHISPER_API int whisper_n_len           (struct whisper_context * ctx); // mel长度
    WHISPER_API int whisper_n_len_from_state(struct whisper_state * state); // mel长度
    WHISPER_API int whisper_n_vocab         (struct whisper_context * ctx);
    WHISPER_API int whisper_n_text_ctx      (struct whisper_context * ctx);
    WHISPER_API int whisper_n_audio_ctx     (struct whisper_context * ctx);
    WHISPER_API int whisper_is_multilingual (struct whisper_context * ctx);
    
    WHISPER_API int whisper_model_n_vocab      (struct whisper_context * ctx);
    WHISPER_API int whisper_model_n_audio_ctx  (struct whisper_context * ctx);
    // 返回音频状态的数量
    WHISPER_API int whisper_model_n_audio_state(struct whisper_context * ctx);
    // 返回音频头的数量
    WHISPER_API int whisper_model_n_audio_head (struct whisper_context * ctx);
    // 返回音频层的数量
    WHISPER_API int whisper_model_n_audio_layer(struct whisper_context * ctx);
    // 返回文本上下文的数量
    WHISPER_API int whisper_model_n_text_ctx   (struct whisper_context * ctx);
    // 返回文本状态的数量
    WHISPER_API int whisper_model_n_text_state (struct whisper_context * ctx);
    // 返回文本头的数量
    WHISPER_API int whisper_model_n_text_head  (struct whisper_context * ctx);
    // 返回文本层的数量
    WHISPER_API int whisper_model_n_text_layer (struct whisper_context * ctx);
    // 返回梅尔频谱的数量
    WHISPER_API int whisper_model_n_mels       (struct whisper_context * ctx);
    // 返回特征类型
    WHISPER_API int whisper_model_ftype        (struct whisper_context * ctx);
    // 返回模型类型
    WHISPER_API int whisper_model_type         (struct whisper_context * ctx);
    
    // 获取最后一次调用whisper_decode()得到的token对数
    // 最后一个token的对数存储在最后一行
    // 行数：n_tokens
    // 列数：n_vocab
    WHISPER_API float * whisper_get_logits           (struct whisper_context * ctx);
    WHISPER_API float * whisper_get_logits_from_state(struct whisper_state * state);
    
    // Token Id -> String。使用提供的上下文中的词汇表
    WHISPER_API const char * whisper_token_to_str(struct whisper_context * ctx, whisper_token token);
    // 返回可读的模型类型
    WHISPER_API const char * whisper_model_type_readable(struct whisper_context * ctx);
    
    // 特殊token
    WHISPER_API whisper_token whisper_token_eot (struct whisper_context * ctx);
    WHISPER_API whisper_token whisper_token_sot (struct whisper_context * ctx);
    WHISPER_API whisper_token whisper_token_solm(struct whisper_context * ctx);
    WHISPER_API whisper_token whisper_token_prev(struct whisper_context * ctx);
    WHISPER_API whisper_token whisper_token_nosp(struct whisper_context * ctx);
    WHISPER_API whisper_token whisper_token_not (struct whisper_context * ctx);
    WHISPER_API whisper_token whisper_token_beg (struct whisper_context * ctx);
    // 声明一个函数，该函数接受一个指向 whisper_context 结构体的指针和一个整型参数，返回一个 whisper_token 结构体
    WHISPER_API whisper_token whisper_token_lang(struct whisper_context * ctx, int lang_id);

    // 任务令牌
    // 声明两个函数，它们都接受一个指向 whisper_context 结构体的指针，返回一个 whisper_token 结构体
    WHISPER_API whisper_token whisper_token_translate (struct whisper_context * ctx);
    WHISPER_API whisper_token whisper_token_transcribe(struct whisper_context * ctx);

    // 从默认状态获取性能信息
    // 声明两个函数，它们都接受一个指向 whisper_context 结构体的指针，无返回值
    WHISPER_API void whisper_print_timings(struct whisper_context * ctx);
    WHISPER_API void whisper_reset_timings(struct whisper_context * ctx);

    // 打印系统信息
    // 声明一个函数，无参数，返回一个指向常量字符的指针
    WHISPER_API const char * whisper_print_system_info(void);

    ////////////////////////////////////////////////////////////////////////////

    // 可用的采样策略
    // 定义一个枚举类型 whisper_sampling_strategy，包含两个取值 WHISPER_SAMPLING_GREEDY 和 WHISPER_SAMPLING_BEAM_SEARCH
    enum whisper_sampling_strategy {
        WHISPER_SAMPLING_GREEDY,      // 类似于 OpenAI 的 GreedyDecoder
        WHISPER_SAMPLING_BEAM_SEARCH, // 类似于 OpenAI 的 BeamSearchDecoder
    };

    // 文本段回调
    // 每次生成新的文本段时调用
    // 使用 whisper_full_...() 函数获取文本段
    定义一个函数指针类型 whisper_new_segment_callback，该类型的函数接受三个参数：指向 whisper_context 结构体的指针，指向 whisper_state 结构体的指针，整型参数 n_new，以及一个指向 void 的指针，无返回值
    typedef void (*whisper_new_segment_callback)(struct whisper_context * ctx, struct whisper_state * state, int n_new, void * user_data);

    // 进度回调
    // 定义一个函数指针类型 whisper_progress_callback，该类型的函数接受四个参数：指向 whisper_context 结构体的指针，指向 whisper_state 结构体的指针，整型参数 progress，以及一个指向 void 的指针，无返回值
    typedef void (*whisper_progress_callback)(struct whisper_context * ctx, struct whisper_state * state, int progress, void * user_data);

    // 编码器开始回调
    // 如果不为 NULL，在编码器开始之前调用
    // 如果返回 false，则中止计算
    定义一个函数指针类型 whisper_encoder_begin_callback，该类型的函数接受三个参数：指向 whisper_context 结构体的指针，指向 whisper_state 结构体的指针，以及一个指向 void 的指针，返回一个布尔值
    typedef bool (*whisper_encoder_begin_callback)(struct whisper_context * ctx, struct whisper_state * state, void * user_data);

    // 中止回调
    // 如果不为 NULL，在 ggml 计算之前调用
    // 如果返回 true，则中止计算
    定义一个函数指针类型 whisper_abort_callback，该类型的函数接受一个指向 void 的指针，返回一个布尔值
    typedef bool (*whisper_abort_callback)(void * user_data);

    // Logits 过滤回调
    // 可以用于修改采样前的 logits
    // 如果不为 NULL，在对 logits 应用温度后调用
    // 定义一个指向whisper_logits_filter_callback函数的指针类型
    typedef void (*whisper_logits_filter_callback)(
            struct whisper_context * ctx,
              struct whisper_state * state,
          const whisper_token_data * tokens,
                               int   n_tokens,
                             float * logits,
                              void * user_data);

    // whisper_full()函数的参数
    // 如果改变参数的顺序或添加新参数，需要确保在whisper.cpp中更新whisper_full_default_params()的默认值
    // 注意：这个结构体定义似乎是不完整的，缺少了成员变量和方法

    // 注意：这个函数分配内存，调用者有责任释放指针 - 参见whisper_free_context_params和whisper_free_params()
    WHISPER_API struct whisper_context_params * whisper_context_default_params_by_ref();
    WHISPER_API struct whisper_context_params whisper_context_default_params(void);
    WHISPER_API struct whisper_full_params * whisper_full_default_params_by_ref(enum whisper_sampling_strategy strategy);
    WHISPER_API struct whisper_full_params whisper_full_default_params(enum whisper_sampling_strategy strategy);

    // 运行整个模型：PCM -> 对数梅尔频谱图 -> 编码器 -> 解码器 -> 文本
    // 对于相同的上下文，不是线程安全的
    // 使用指定的解码策略来获取文本
    WHISPER_API int whisper_full(
                struct whisper_context * ctx,
            struct whisper_full_params   params,
                           const float * samples,
                                   int   n_samples);

    WHISPER_API int whisper_full_with_state(
                struct whisper_context * ctx,
                  struct whisper_state * state,
            struct whisper_full_params   params,
                           const float * samples,
                                   int   n_samples);

    // 将输入音频分割成块，并使用whisper_full_with_state()分别处理每个块
    // 结果存储在上下文的默认状态中
    // 在相同上下文中并行执行时，不是线程安全的。
    // 在某些情况下，这种方法似乎可以提供一些加速。
    // 但是，每个块的开头和结尾的转录准确性可能会更差。
    WHISPER_API int whisper_full_parallel(
                struct whisper_context * ctx,
            struct whisper_full_params   params,
                           const float * samples,
                                   int   n_samples,
                                   int   n_processors);

    // 生成的文本段的数量
    // 一个段可以是几个单词，一个句子，甚至是一个段落。
    WHISPER_API int whisper_full_n_segments           (struct whisper_context * ctx);
    WHISPER_API int whisper_full_n_segments_from_state(struct whisper_state * state);

    // 与上下文的默认状态关联的语言ID
    WHISPER_API int whisper_full_lang_id(struct whisper_context * ctx);

    // 与提供的状态关联的语言ID
    WHISPER_API int whisper_full_lang_id_from_state(struct whisper_state * state);

    // 获取指定段的开始和结束时间
    WHISPER_API int64_t whisper_full_get_segment_t0           (struct whisper_context * ctx, int i_segment);
    WHISPER_API int64_t whisper_full_get_segment_t0_from_state(struct whisper_state * state, int i_segment);

    WHISPER_API int64_t whisper_full_get_segment_t1           (struct whisper_context * ctx, int i_segment);
    WHISPER_API int64_t whisper_full_get_segment_t1_from_state(struct whisper_state * state, int i_segment);

    // 获取下一个段是否被预测为发言者转换
    WHISPER_API bool whisper_full_get_segment_speaker_turn_next(struct whisper_context * ctx, int i_segment);
    WHISPER_API bool whisper_full_get_segment_speaker_turn_next_from_state(struct whisper_state * state, int i_segment);

    // 获取指定段的文本
    // 获取指定段落的文本内容
    WHISPER_API const char * whisper_full_get_segment_text           (struct whisper_context * ctx, int i_segment);
    // 从状态中获取指定段落的文本内容
    WHISPER_API const char * whisper_full_get_segment_text_from_state(struct whisper_state * state, int i_segment);
    
    // 获取指定段落中标记的数量
    WHISPER_API int whisper_full_n_tokens           (struct whisper_context * ctx, int i_segment);
    // 从状态中获取指定段落中标记的数量
    WHISPER_API int whisper_full_n_tokens_from_state(struct whisper_state * state, int i_segment);
    
    // 获取指定段落中指定标记的文本内容
    WHISPER_API const char * whisper_full_get_token_text           (struct whisper_context * ctx, int i_segment, int i_token);
    // 从状态中获取指定段落中指定标记的文本内容
    WHISPER_API const char * whisper_full_get_token_text_from_state(struct whisper_context * ctx, struct whisper_state * state, int i_segment, int i_token);
    
    // 获取指定段落中指定标记的标记 ID
    WHISPER_API whisper_token whisper_full_get_token_id           (struct whisper_context * ctx, int i_segment, int i_token);
    // 从状态中获取指定段落中指定标记的标记 ID
    WHISPER_API whisper_token whisper_full_get_token_id_from_state(struct whisper_state * state, int i_segment, int i_token);
    
    // 获取指定段落中指定标记的标记数据
    // 这包括概率、时间戳等
    WHISPER_API whisper_token_data whisper_full_get_token_data           (struct whisper_context * ctx, int i_segment, int i_token);
    // 从状态中获取指定段落中指定标记的标记数据
    // 这包括概率、时间戳等
    WHISPER_API whisper_token_data whisper_full_get_token_data_from_state(struct whisper_state * state, int i_segment, int i_token);
    
    // 获取指定段落中指定标记的概率
    WHISPER_API float whisper_full_get_token_p           (struct whisper_context * ctx, int i_segment, int i_token);
    // 从状态中获取指定段落中指定标记的概率
    WHISPER_API float whisper_full_get_token_p_from_state(struct whisper_state * state, int i_segment, int i_token);
    
    // 临时辅助函数，用于暴露 ggml 接口
    # 定义一个返回整数的函数，用于测试内存拷贝性能，参数为线程数
    WHISPER_API int whisper_bench_memcpy (int n_threads);
    # 定义一个返回字符串的函数，用于测试内存拷贝性能，参数为线程数
    WHISPER_API const char * whisper_bench_memcpy_str (int n_threads);
    # 定义一个返回整数的函数，用于测试 GGML 矩阵乘法性能，参数为线程数
    WHISPER_API int whisper_bench_ggml_mul_mat (int n_threads);
    # 定义一个返回字符串的函数，用于测试 GGML 矩阵乘法性能，参数为线程数
    WHISPER_API const char * whisper_bench_ggml_mul_mat_str (int n_threads);
    
    # 控制日志输出的函数，设置日志回调函数和用户数据
    WHISPER_API void whisper_log_set(ggml_log_callback log_callback, void * user_data);
#ifdef __cplusplus
}  // 结束 C++ 的 extern "C" 块
#endif  // 结束条件编译，检查是否为 C++ 环境

#endif  // 结束条件编译，结束头文件的 ifndef 指令
```