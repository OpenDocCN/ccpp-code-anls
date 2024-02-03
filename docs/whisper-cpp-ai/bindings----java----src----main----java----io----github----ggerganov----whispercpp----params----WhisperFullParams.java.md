# `whisper.cpp\bindings\java\src\main\java\io\github\ggerganov\whispercpp\params\WhisperFullParams.java`

```cpp
package io.github.ggerganov.whispercpp.params;

import com.sun.jna.*;
import io.github.ggerganov.whispercpp.callbacks.WhisperEncoderBeginCallback;
import io.github.ggerganov.whispercpp.callbacks.WhisperLogitsFilterCallback;
import io.github.ggerganov.whispercpp.callbacks.WhisperNewSegmentCallback;
import io.github.ggerganov.whispercpp.callbacks.WhisperProgressCallback;

import java.util.Arrays;
import java.util.List;

/**
 * Parameters for the whisper_full() function.
 * If you change the order or add new parameters, make sure to update the default values in whisper.cpp:
 * whisper_full_default_params()
 */
public class WhisperFullParams extends Structure {

    public WhisperFullParams(Pointer p) {
        super(p);
//        super(p, ALIGN_MSVC);
//        super(p, ALIGN_GNUC);
    }

    /** Sampling strategy for whisper_full() function. */
    public int strategy;

    /** Number of threads. (default = 4) */
    public int n_threads;

    /** Maximum tokens to use from past text as a prompt for the decoder. (default = 16384) */
    public int n_max_text_ctx;

    /** Start offset in milliseconds. (default = 0) */
    public int offset_ms;

    /** Audio duration to process in milliseconds. (default = 0) */
    public int duration_ms;

    /** Translate flag. (default = false) */
    public CBool translate;

    /** The compliment of translateMode() */
    public void transcribeMode() {
        translate = CBool.FALSE;
    }

    /** The compliment of transcribeMode() */
    public void translateMode() {
        translate = CBool.TRUE;
    }

    /** Flag to indicate whether to use past transcription (if any) as an initial prompt for the decoder. (default = true) */
    public CBool no_context;

    /** Flag to indicate whether to use past transcription (if any) as an initial prompt for the decoder. (default = true) */
    public void enableContext(boolean enable) {
        no_context = enable ? CBool.FALSE : CBool.TRUE;
    }
}
    /** 是否生成时间戳？ */
    public CBool no_timestamps;

    /** 强制输出单个段落（对流式传输很有用）。 (默认值 = false) */
    public CBool single_segment;

    /** 强制输出单个段落（对流式传输很有用）。 (默认值 = false) */
    public void singleSegment(boolean single) {
        single_segment = single ? CBool.TRUE : CBool.FALSE;
    }

    /** 打印特殊标记（例如，&lt;SOT>，&lt;EOT>，&lt;BEG>等）。 (默认值 = false) */
    public CBool print_special;

    /** 打印特殊标记（例如，&lt;SOT>，&lt;EOT>，&lt;BEG>等）。 (默认值 = false) */
    public void printSpecial(boolean enable) {
        print_special = enable ? CBool.TRUE : CBool.FALSE;
    }

    /** 打印进度信息的标志。 (默认值 = true) */
    public CBool print_progress;

    /** 打印进度信息的标志。 (默认值 = true) */
    public void printProgress(boolean enable) {
        print_progress = enable ? CBool.TRUE : CBool.FALSE;
    }

    /** 打印来自whisper.cpp的结果（避免使用，改用回调函数）。 (默认值 = true) */
    public CBool print_realtime;

    /** 打印来自whisper.cpp的结果（避免使用，改用回调函数）。 (默认值 = true) */
    public void printRealtime(boolean enable) {
        print_realtime = enable ? CBool.TRUE : CBool.FALSE;
    }

    /** 打印实时打印时每个文本段的时间戳。 (默认值 = true) */
    public CBool print_timestamps;

    /** 打印实时打印时每个文本段的时间戳。 (默认值 = true) */
    public void printTimestamps(boolean enable) {
        print_timestamps = enable ? CBool.TRUE : CBool.FALSE;
    }

    /** [实验性] 启用标记级时间戳的标志。 (默认值 = false) */
    public CBool token_timestamps;

    /** [实验性] 启用标记级时间戳的标志。 (默认值 = false) */
    // 设置是否启用令牌时间戳
    public void tokenTimestamps(boolean enable) {
        token_timestamps = enable ? CBool.TRUE : CBool.FALSE;
    }

    /** [实验性功能] 令牌时间戳概率阈值（~0.01）。 (默认值 = 0.01) */
    public float thold_pt;

    /** [实验性功能] 令牌时间戳总概率阈值（~0.01）。 */
    public float thold_ptsum;

    /** 最大段长度（以字符为单位）。 (默认值 = 0) */
    public int max_len;

    /** 标志，用于在单词上拆分而不是在令牌上拆分（与 max_len 一起使用时）。 (默认值 = false) */
    public CBool split_on_word;

    /** 标志，用于在单词上拆分而不是在令牌上拆分（与 max_len 一起使用时）。 (默认值 = false) */
    public void splitOnWord(boolean enable) {
        split_on_word = enable ? CBool.TRUE : CBool.FALSE;
    }

    /** 每个段的最大令牌数（0，表示无限制，默认值 = 无限制） */
    public int max_tokens;

    /** 使用相位声码器将音频加速 2 倍的标志。 (默认值 = false) */
    public CBool speed_up;

    /** 使用相位声码器将音频加速 2 倍的标志。 (默认值 = false) */
    public void speedUp(boolean enable) {
        speed_up = enable ? CBool.TRUE : CBool.FALSE;
    }

    /** 覆盖音频上下文大小（0 = 使用默认值）。 */
    public int audio_ctx;

    /** 启用 tinydiarize（默认值 = false） */
    public CBool tdrz_enable;

    /** 启用 tinydiarize（默认值 = false） */
    public void tdrzEnable(boolean enable) {
        tdrz_enable = enable ? CBool.TRUE : CBool.FALSE;
    }

    /** 作为初始提示提供给耳语解码器的令牌。
     * 这些令牌将添加到先前调用中存在的任何现有文本上下文之前。 */
    public String initial_prompt;

    /** 提示令牌。 (int*) */
    public Pointer prompt_tokens;

    public void setPromptTokens(int[] tokens) {
        Memory mem = new Memory(tokens.length * 4L);
        mem.write(0, tokens, 0, tokens.length);
        prompt_tokens = mem;
    }

    /** 提示令牌的数量。 */
    public int prompt_n_tokens;
    /** Language for auto-detection.
     * For auto-detection, set to `null`, `""`, or "auto". */
    // 用于自动检测语言的语言设置，设置为`null`、`""`或"auto"以进行自动检测
    public String language;

    /** Flag to indicate whether to detect language automatically. */
    // 用于指示是否自动检测语言的标志
    public CBool detect_language;

    /** Flag to indicate whether to detect language automatically. */
    // 用于指示是否自动检测语言的标志
    public void detectLanguage(boolean enable) {
        // 根据传入的参数值设置是否自动检测语言
        detect_language = enable ? CBool.TRUE : CBool.FALSE;
    }

    // Common decoding parameters.

    /** Flag to suppress blank tokens. */
    // 用于抑制空白标记的标志
    public CBool suppress_blank;

    public void suppressBlanks(boolean enable) {
        // 根据传入的参数值设置是否抑制空白标记
        suppress_blank = enable ? CBool.TRUE : CBool.FALSE;
    }

    /** Flag to suppress non-speech tokens. */
    // 用于抑制非语音标记的标志
    public CBool suppress_non_speech_tokens;

    /** Flag to suppress non-speech tokens. */
    // 用于抑制非语音标记的标志
    public void suppressNonSpeechTokens(boolean enable) {
        // 根据传入的参数值设置是否抑制非语音标记
        suppress_non_speech_tokens = enable ? CBool.TRUE : CBool.FALSE;
    }

    /** Initial decoding temperature. */
    // 初始解码温度
    public float temperature;

    /** Maximum initial timestamp. */
    // 最大初始时间戳
    public float max_initial_ts;

    /** Length penalty. */
    // 长度惩罚
    public float length_penalty;

    // Fallback parameters.

    /** Temperature increment. */
    // 温度增量
    public float temperature_inc;

    /** Entropy threshold (similar to OpenAI's "compression_ratio_threshold"). */
    // 熵阈值（类似于OpenAI的“压缩比率阈值”）
    public float entropy_thold;

    /** Log probability threshold. */
    // 对数概率阈值
    public float logprob_thold;

    /** No speech threshold. */
    // 无语音阈值
    public float no_speech_thold;

    /** Greedy decoding parameters. */
    // 贪婪解码参数
    public GreedyParams greedy;

    /**
     * Beam search decoding parameters.
     */
    // Beam搜索解码参数
    public BeamSearchParams beam_search;

    public void setBestOf(int bestOf) {
        if (greedy == null) {
            greedy = new GreedyParams();
        }
        // 设置最佳结果数
        greedy.best_of = bestOf;
    }

    public void setBeamSize(int beamSize) {
        if (beam_search == null) {
            beam_search = new BeamSearchParams();
        }
        // 设置Beam大小
        beam_search.beam_size = beamSize;
    }
    /**
     * 设置束搜索大小和耐心度
     * Set the beam search size and patience
     */
    public void setBeamSizeAndPatience(int beamSize, float patience) {
        // 如果束搜索对象为空，则创建一个新的束搜索参数对象
        if (beam_search == null) {
            beam_search = new BeamSearchParams();
        }
        // 设置束搜索大小和耐心度
        beam_search.beam_size = beamSize;
        beam_search.patience = patience;
    }

    /**
     * 每次生成新文本片段时的回调
     * Callback for every newly generated text segment
     */
    public Pointer new_segment_callback;

    /**
     * new_segment_callback 的用户数据
     * User data for the new_segment_callback
     */
    public Pointer new_segment_callback_user_data;

    /**
     * 每次进度更新时的回调
     * Callback on each progress update
     */
    public Pointer progress_callback;

    /**
     * progress_callback 的用户数据
     * User data for the progress_callback
     */
    public Pointer progress_callback_user_data;

    /**
     * 每次编码器开始前的回调
     * Callback each time before the encoder starts
     */
    public Pointer encoder_begin_callback;

    /**
     * encoder_begin_callback 的用户数据
     * User data for the encoder_begin_callback
     */
    public Pointer encoder_begin_callback_user_data;

    /**
     * 每个解码器用于过滤获取的对数的回调
     * Callback by each decoder to filter obtained logits
     */
    public Pointer logits_filter_callback;

    /**
     * logits_filter_callback 的用户数据
     * User data for the logits_filter_callback
     */
    public Pointer logits_filter_callback_user_data;

    /**
     * 设置新文本片段回调
     * Set the new segment callback
     */
    public void setNewSegmentCallback(WhisperNewSegmentCallback callback) {
        // 获取回调函数的函数指针并设置给 new_segment_callback
        new_segment_callback = CallbackReference.getFunctionPointer(callback);
    }

    /**
     * 设置进度回调
     * Set the progress callback
     */
    public void setProgressCallback(WhisperProgressCallback callback) {
        // 获取回调函数的函数指针并设置给 progress_callback
        progress_callback = CallbackReference.getFunctionPointer(callback);
    }

    /**
     * 设置编码器开始回调
     * Set the encoder begin callback
     */
    public void setEncoderBeginCallbackeginCallbackCallback(WhisperEncoderBeginCallback callback) {
        // 获取回调函数的函数指针并设置给 encoder_begin_callback
        encoder_begin_callback = CallbackReference.getFunctionPointer(callback);
    }

    /**
     * 设置对数过滤回调
     * Set the logits filter callback
     */
    public void setLogitsFilterCallback(WhisperLogitsFilterCallback callback) {
        // 获取回调函数的函数指针并设置给 logits_filter_callback
        logits_filter_callback = CallbackReference.getFunctionPointer(callback);
    }

    /** 语法相关 */
    // 声明一个指向语法规则的指针
    public Pointer grammar_rules;
    // 声明语法规则的数量
    public long n_grammar_rules;
    // 声明起始规则的索引
    public long i_start_rule;
    // 声明语法惩罚值
    public float grammar_penalty;

    // 重写 getFieldOrder 方法，指定字段的顺序
    @Override
    protected List<String> getFieldOrder() {
        // 返回字段的顺序列表
        return Arrays.asList("strategy", "n_threads", "n_max_text_ctx", "offset_ms", "duration_ms", "translate",
                "no_context", "single_segment", "no_timestamps",
                "print_special", "print_progress", "print_realtime", "print_timestamps",  "token_timestamps",
                "thold_pt", "thold_ptsum", "max_len", "split_on_word", "max_tokens", "speed_up", "audio_ctx",
                "tdrz_enable", "initial_prompt", "prompt_tokens", "prompt_n_tokens", "language", "detect_language",
                "suppress_blank", "suppress_non_speech_tokens", "temperature", "max_initial_ts", "length_penalty",
                "temperature_inc", "entropy_thold", "logprob_thold", "no_speech_thold", "greedy", "beam_search",
                "new_segment_callback", "new_segment_callback_user_data",
                "progress_callback", "progress_callback_user_data",
                "encoder_begin_callback", "encoder_begin_callback_user_data",
                "logits_filter_callback", "logits_filter_callback_user_data",
                "grammar_rules", "n_grammar_rules", "i_start_rule", "grammar_penalty");
    }
# 闭合之前的代码块
```