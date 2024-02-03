# `whisper.cpp\bindings\java\src\main\java\io\github\ggerganov\whispercpp\WhisperCppJnaLibrary.java`

```cpp
// 导入必要的库
package io.github.ggerganov.whispercpp;

import com.sun.jna.Library;
import com.sun.jna.Native;
import com.sun.jna.Pointer;
import io.github.ggerganov.whispercpp.model.WhisperModelLoader;
import io.github.ggerganov.whispercpp.model.WhisperTokenData;
import io.github.ggerganov.whispercpp.params.WhisperContextParams;
import io.github.ggerganov.whispercpp.params.WhisperFullParams;

// 定义接口 WhisperCppJnaLibrary 继承自 Library
public interface WhisperCppJnaLibrary extends Library {
    // 加载名为 "whisper" 的库，并创建 WhisperCppJnaLibrary 实例
    WhisperCppJnaLibrary instance = Native.load("whisper", WhisperCppJnaLibrary.class);

    // 打印系统信息
    String whisper_print_system_info();

    /**
     * DEPRECATED. Allocate (almost) all memory needed for the model by loading from a file.
     *
     * @param path_model Path to the model file
     * @return Whisper context on success, null on failure
     */
    // 从文件中加载模型，返回 Whisper 上下文
    Pointer whisper_init_from_file(String path_model);
    
    /**
     * Provides default params which can be used with `whisper_init_from_file_with_params()` etc.
     * Because this function allocates memory for the params, the caller must call either:
     * - call `whisper_free_context_params()`
     * - `Native.free(Pointer.nativeValue(pointer));`
     */
    // 提供可用于 `whisper_init_from_file_with_params()` 等函数的默认参数
    // 调用者必须释放内存
    Pointer whisper_context_default_params_by_ref();

    // 释放上下文参数内存
    void whisper_free_context_params(Pointer params);

    /**
     * Allocate (almost) all memory needed for the model by loading from a file.
     *
     * @param path_model Path to the model file
     * @param params     Pointer to whisper_context_params
     * @return Whisper context on success, null on failure
     */
    // 从文件中加载模型，并使用给定参数
    Pointer whisper_init_from_file_with_params(String path_model, WhisperContextParams params);

    /**
     * Allocate (almost) all memory needed for the model by loading from a buffer.
     *
     * @param buffer       Model buffer
     * @param buffer_size  Size of the model buffer
     * @return Whisper context on success, null on failure
     */
    // 从缓冲区中加载模型，返回 Whisper 上下文
    Pointer whisper_init_from_buffer(Pointer buffer, int buffer_size);
}
    /**
     * 使用模型加载器分配几乎所有需要的内存来初始化 Whisper 上下文。
     *
     * @param loader 模型加载器
     * @return 成功时返回 Whisper 上下文，失败时返回 null
     */
    Pointer whisper_init(WhisperModelLoader loader);
    
    /**
     * 通过从文件加载模型来分配几乎所有需要的内存，但不分配状态。
     *
     * @param path_model 模型文件的路径
     * @return 成功时返回 Whisper 上下文，失败时返回 null
     */
    Pointer whisper_init_from_file_no_state(String path_model);
    
    /**
     * 通过从缓冲区加载模型来分配几乎所有需要的内存，但不分配状态。
     *
     * @param buffer       模型缓冲区
     * @param buffer_size  模型缓冲区的大小
     * @return 成功时返回 Whisper 上下文，失败时返回 null
     */
    Pointer whisper_init_from_buffer_no_state(Pointer buffer, int buffer_size);
// 初始化 Whisper 上下文，不包含状态信息
/**
 * 使用模型加载器分配几乎所有模型所需的内存，但不分配状态。
 *
 * @param loader 模型加载器
 * @return 成功时返回 Whisper 上下文，失败时返回 null
 */
Pointer whisper_init_no_state(WhisperModelLoader loader);

// 初始化 Whisper 状态的内存
/**
 * 为 Whisper 状态分配内存。
 *
 * @param ctx Whisper 上下文
 * @return 成功时返回 Whisper 状态，失败时返回 null
 */
Pointer whisper_init_state(Pointer ctx);

// 释放与 Whisper 上下文关联的所有分配的内存
/**
 * 释放与 Whisper 上下文关联的所有分配的内存。
 *
 * @param ctx Whisper 上下文
 */
void whisper_free(Pointer ctx);

// 释放与 Whisper 状态关联的所有分配的内存
/**
 * 释放与 Whisper 状态关联的所有分配的内存。
 *
 * @param state Whisper 状态
 */
void whisper_free_state(Pointer state);

// 将原始 PCM 音频转换为对数梅尔频谱图
/**
 * 将原始 PCM 音频转换为对数梅尔频谱图。
 * 结果频谱图存储在所提供的 Whisper 上下文的默认状态内。
 *
 * @param ctx - 指向 WhisperContext 的指针
 * @return 成功时返回 0
 */
int whisper_pcm_to_mel(Pointer ctx, final float[] samples, int n_samples, int n_threads);

/**
 * @param ctx 指向 WhisperContext 的指针
 * @param state 指向 WhisperState 的指针
 * @param n_samples
 * @param n_threads
 * @return 成功时返回 0
 */
int whisper_pcm_to_mel_with_state(Pointer ctx, Pointer state, final float[] samples, int n_samples, int n_threads);

// 可用于在所提供的 Whisper 上下文的默认状态内设置自定义对数梅尔频谱图
/**
 * 可用于在所提供的 Whisper 上下文的默认状态内设置自定义对数梅尔频谱图。
 * 如果要提供自己的对数梅尔频谱图，请使用此方法，而不是 whisper_pcm_to_mel()。
 * n_mel 必须为 80
 * @return 成功时返回 0
 */
int whisper_set_mel(Pointer ctx, final float[] data, int n_len, int n_mel);
    /**
     * 使用状态指针和数据数组执行 Whisper 编码器，生成梅尔频谱
     * @param ctx Whisper 上下文指针
     * @param state 状态指针
     * @param data 浮点数数组，存储梅尔频谱数据
     * @param n_len 数据长度
     * @param n_mel 梅尔频谱长度
     */
    int whisper_set_mel_with_state(Pointer ctx, Pointer state, final float[] data, int n_len, int n_mel);

    /**
     * 在提供的 Whisper 上下文中的默认状态上运行 Whisper 编码器，对存储在其中的对数梅尔频谱进行编码
     * 确保先调用 whisper_pcm_to_mel() 或 whisper_set_mel()
     * 偏移量可用于指定频谱中第一帧的偏移量
     * @return 成功返回 0
     */
    int whisper_encode(Pointer ctx, int offset, int n_threads);

    /**
     * 在提供的 Whisper 上下文中的状态上运行 Whisper 编码器，对存储在其中的对数梅尔频谱进行编码
     * @param ctx Whisper 上下文指针
     * @param state 状态指针
     * @param offset 偏移量
     * @param n_threads 线程数
     */
    int whisper_encode_with_state(Pointer ctx, Pointer state, int offset, int n_threads);

    /**
     * 运行 Whisper 解码器，获取下一个标记的对数和概率
     * 确保先调用 whisper_encode()
     * tokens + n_tokens 是解码器的上下文
     * n_past 是要使用的来自先前解码器调用的标记数
     * 返回成功返回 0
     * TODO: 添加对多个解码器的支持
     */
    int whisper_decode(Pointer ctx, Pointer tokens, int n_tokens, int n_past, int n_threads);

    /**
     * 在提供的 Whisper 上下文中的状态上运行 Whisper 解码器，获取下一个标记的对数和概率
     * @param ctx Whisper 上下文指针
     * @param state 状态指针
     * @param tokens 指向整数标记的指针
     * @param n_tokens 标记数
     * @param n_past 使用的先前标记数
     * @param n_threads 线程数
     * @return
     */
    int whisper_decode_with_state(Pointer ctx, Pointer state, Pointer tokens, int n_tokens, int n_past, int n_threads);

    /**
     * 将提供的文本转换为标记
     * tokens 指针必须足够大以容纳生成的标记
     * 成功时返回标记数，不超过 n_max_tokens
     * 失败时返回 -1
     * TODO: 不确定是否正确
     */
    int whisper_tokenize(Pointer ctx, String text, Pointer tokens, int n_max_tokens);

    /** 最大语言 id（可用语言数量 - 1） */
    int whisper_lang_max_id();

    /**
     * 返回指定语言的 id，如果未找到则返回 -1
     * 示例：
     *   "de" -> 2
     *   "german" -> 2
     */
    // 定义一个函数，根据语言ID返回对应的短字符串
    int whisper_lang_id(String lang);

    /** 
     * 返回指定语言ID的短字符串（例如，2 -> "de"），如果未找到则返回nullptr
     */
    String whisper_lang_str(int id);

    /**
     * 使用偏移时间为offset_ms的mel数据尝试自动检测所说的语言。
     * 确保先调用whisper_pcm_to_mel()或whisper_set_mel()
     * 返回顶部语言ID，失败时返回负数
     * 如果不为null，则用所有语言的概率填充lang_probs数组
     * 数组的大小必须为whisper_lang_max_id() + 1
     *
     * 参考：https://github.com/openai/whisper/blob/main/whisper/decoding.py#L18-L69
     */
    int whisper_lang_auto_detect(Pointer ctx, int offset_ms, int n_threads, float[] lang_probs);

    // 使用状态指针state从偏移时间为offset_ms处自动检测语言
    int whisper_lang_auto_detect_with_state(Pointer ctx, Pointer state, int offset_ms, int n_threads, float[] lang_probs);

    // 获取mel长度
    int whisper_n_len(Pointer ctx); 

    // 从状态获取mel长度
    int whisper_n_len_from_state(Pointer state); 

    // 获取词汇表大小
    int whisper_n_vocab(Pointer ctx);

    // 获取文本上下文大小
    int whisper_n_text_ctx(Pointer ctx);

    // 获取音频上下文大小
    int whisper_n_audio_ctx(Pointer ctx);

    // 检查是否支持多语言
    int whisper_is_multilingual(Pointer ctx);

    // 获取模型词汇表大小
    int whisper_model_n_vocab(Pointer ctx);

    // 获取模型音频上下文大小
    int whisper_model_n_audio_ctx(Pointer ctx);

    // 获取模型音频状态大小
    int whisper_model_n_audio_state(Pointer ctx);

    // 获取模型音频头大小
    int whisper_model_n_audio_head(Pointer ctx);

    // 获取模型音频层大小
    int whisper_model_n_audio_layer(Pointer ctx);

    // 获取模型文本上下文大小
    int whisper_model_n_text_ctx(Pointer ctx);

    // 获取模型文本状态大小
    int whisper_model_n_text_state(Pointer ctx);

    // 获取模型文本头大小
    int whisper_model_n_text_head(Pointer ctx);

    // 获取模型文本层大小
    int whisper_model_n_text_layer(Pointer ctx);

    // 获取模型mel数量
    int whisper_model_n_mels(Pointer ctx);

    // 获取模型特征类型
    int whisper_model_ftype(Pointer ctx);

    // 获取模型类型
    int whisper_model_type(Pointer ctx);

    /**
     * 上一次调用whisper_decode()获取的token logits。
     * 最后一个token的logits存储在最后一行
     * 行数：n_tokens
     * 列数：n_vocab
     */
    // 获取模型输出的logits数组
    float[] whisper_get_logits(Pointer ctx);
    // 从状态中获取模型输出的logits数组
    float[] whisper_get_logits_from_state(Pointer state);
    
    // 将token id转换为字符串，使用提供的上下文中的词汇表
    String whisper_token_to_str(Pointer ctx, int token);
    // 获取模型类型的可读字符串
    String whisper_model_type_readable(Pointer ctx);
    
    // 特殊token
    int whisper_token_eot(Pointer ctx);
    int whisper_token_sot(Pointer ctx);
    int whisper_token_prev(Pointer ctx);
    int whisper_token_solm(Pointer ctx);
    int whisper_token_not(Pointer ctx);
    int whisper_token_beg(Pointer ctx);
    int whisper_token_lang(Pointer ctx, int lang_id);
    
    // 任务相关token
    int whisper_token_translate(Pointer ctx);
    int whisper_token_transcribe(Pointer ctx);
    
    // 从默认状态获取性能信息
    void whisper_print_timings(Pointer ctx);
    void whisper_reset_timings(Pointer ctx);
    
    // 提供默认参数，可用于`whisper_full()`等函数
    // 由于此函数为参数分配内存，调用者必须调用以下之一：
    // - 调用`whisper_free_params()`
    // - `Native.free(Pointer.nativeValue(pointer));`
    // 参数strategy为WhisperSamplingStrategy.value
    Pointer whisper_full_default_params_by_ref(int strategy);
    
    // 释放参数内存
    void whisper_free_params(Pointer params);
    
    /**
     * 运行整个模型：PCM -> log mel spectrogram -> encoder -> decoder -> text
     * 对于同一上下文，不是线程安全的
     * 使用指定的解码策略获取文本
     */
    int whisper_full(Pointer ctx, WhisperFullParams params, final float[] samples, int n_samples);
    // 使用给定的上下文、状态和参数，对输入的音频进行处理，返回处理后的结果
    int whisper_full_with_state(Pointer ctx, Pointer state, WhisperFullParams params, final float[] samples, int n_samples);

    // 将输入音频分成多个块，并分别使用whisper_full_with_state()处理每个块
    // 结果存储在上下文的默认状态中
    // 如果在同一上下文中并行执行，不是线程安全的
    // 在某些情况下，这种方法可能会提供一些加速
    // 但是，每个块的开头和结尾的转录准确性可能会更差
    int whisper_full_parallel(Pointer ctx, WhisperFullParams params, final float[] samples, int n_samples, int n_processors);

    /**
     * 生成的文本段数
     * 一个段落可以是几个单词、一个句子，甚至是一个段落
     * @param ctx WhisperContext的指针
     */
    int whisper_full_n_segments (Pointer ctx);

    /**
     * @param state WhisperState的指针
     */
    int whisper_full_n_segments_from_state(Pointer state);

    /**
     * 与上下文的默认状态关联的语言ID
     * @param ctx WhisperContext的指针
     */
    int whisper_full_lang_id(Pointer ctx);

    /** 与提供的状态关联的语言ID */
    int whisper_full_lang_id_from_state(Pointer state);

    /**
     * 将原始PCM音频转换为对数梅尔频谱，但应用相位估计器以加速音频x2
     * 结果的频谱图存储在提供的whisper上下文的默认状态中
     * @return 成功返回0
     */
    int whisper_pcm_to_mel_phase_vocoder(Pointer ctx, final float[] samples, int n_samples, int n_threads);

    int whisper_pcm_to_mel_phase_vocoder_with_state(Pointer ctx, Pointer state, final float[] samples, int n_samples, int n_threads);

    /** 获取指定段的开始时间 */
    long whisper_full_get_segment_t0(Pointer ctx, int i_segment);

    /** 从状态中获取指定段的开始时间 */
    /** 获取指定段落的起始时间 */
    long whisper_full_get_segment_t0_from_state(Pointer state, int i_segment);

    /** 获取指定段落的结束时间 */
    long whisper_full_get_segment_t1(Pointer ctx, int i_segment);

    /** 从状态中获取指定段落的结束时间 */
    long whisper_full_get_segment_t1_from_state(Pointer state, int i_segment);

    /** 获取指定段落的文本内容 */
    String whisper_full_get_segment_text(Pointer ctx, int i_segment);

    /** 从状态中获取指定段落的文本内容 */
    String whisper_full_get_segment_text_from_state(Pointer state, int i_segment);

    /** 获取指定段落中的标记数量 */
    int whisper_full_n_tokens(Pointer ctx, int i_segment);

    /** 从状态中获取指定段落中的标记数量 */
    int whisper_full_n_tokens_from_state(Pointer state, int i_segment);

    /** 获取指定段落中指定标记的文本内容 */
    String whisper_full_get_token_text(Pointer ctx, int i_segment, int i_token);

    /** 从状态中获取指定段落中指定标记的文本内容 */
    String whisper_full_get_token_text_from_state(Pointer ctx, Pointer state, int i_segment, int i_token);

    /** 获取指定段落中指定标记的标记ID */
    int whisper_full_get_token_id(Pointer ctx, int i_segment, int i_token);

    /** 从状态中获取指定段落中指定标记的标记ID */
    int whisper_full_get_token_id_from_state(Pointer state, int i_segment, int i_token);

    /** 获取指定段落中指定标记的标记数据 */
    WhisperTokenData whisper_full_get_token_data(Pointer ctx, int i_segment, int i_token);

    /** 从状态中获取指定段落中指定标记的标记数据 */
    WhisperTokenData whisper_full_get_token_data_from_state(Pointer state, int i_segment, int i_token);
    /** 获取指定段落中指定标记的概率。 */
    float whisper_full_get_token_p(Pointer ctx, int i_segment, int i_token);

    /** 从状态中获取指定段落中指定标记的概率。 */
    float whisper_full_get_token_p_from_state(Pointer state, int i_segment, int i_token);

    /**
     * 用于 memcpy 的基准测试函数。
     *
     * @param nThreads 用于基准测试的线程数。
     * @return 基准测试的结果。
     */
    int whisper_bench_memcpy(int nThreads);

    /**
     * 用于将 memcpy 作为字符串的基准测试函数。
     *
     * @param nThreads 用于基准测试的线程数。
     * @return 基准测试的结果作为字符串。
     */
    String whisper_bench_memcpy_str(int nThreads);

    /**
     * 用于 ggml_mul_mat 的基准测试函数。
     *
     * @param nThreads 用于基准测试的线程数。
     * @return 基准测试的结果。
     */
    int whisper_bench_ggml_mul_mat(int nThreads);

    /**
     * 用于将 ggml_mul_mat 作为字符串的基准测试函数。
     *
     * @param nThreads 用于基准测试的线程数。
     * @return 基准测试的结果作为字符串。
     */
    String whisper_bench_ggml_mul_mat_str(int nThreads);
# 闭合之前的代码块
```