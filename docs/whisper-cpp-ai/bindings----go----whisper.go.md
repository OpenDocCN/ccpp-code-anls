# `whisper.cpp\bindings\go\whisper.go`

```cpp
package whisper

import (
    "errors"
    "unsafe"
)

///////////////////////////////////////////////////////////////////////////////
// CGO

/*
#cgo LDFLAGS: -lwhisper -lm -lstdc++
#cgo darwin LDFLAGS: -framework Accelerate
#include <whisper.h>
#include <stdlib.h>

extern void callNewSegment(void* user_data, int new);
extern void callProgress(void* user_data, int progress);
extern bool callEncoderBegin(void* user_data);

// Text segment callback
// 每次生成新的文本段时调用
// 使用 whisper_full_...() 函数获取文本段
static void whisper_new_segment_cb(struct whisper_context* ctx, struct whisper_state* state, int n_new, void* user_data) {
    if(user_data != NULL && ctx != NULL) {
        callNewSegment(user_data, n_new);
    }
}

// Progress callback
// 每次生成新的文本段时调用
// 使用 whisper_full_...() 函数获取文本段
static void whisper_progress_cb(struct whisper_context* ctx, struct whisper_state* state, int progress, void* user_data) {
    if(user_data != NULL && ctx != NULL) {
        callProgress(user_data, progress);
    }
}

// Encoder begin callback
// 如果不为 NULL，在编码器开始之前调用
// 如果返回 false，则计算中止
static bool whisper_encoder_begin_cb(struct whisper_context* ctx, struct whisper_state* state, void* user_data) {
    if(user_data != NULL && ctx != NULL) {
        return callEncoderBegin(user_data);
    }
    return false;
}

// 获取默认参数并设置回调函数
static struct whisper_full_params whisper_full_default_params_cb(struct whisper_context* ctx, enum whisper_sampling_strategy strategy) {
    struct whisper_full_params params = whisper_full_default_params(strategy);
    params.new_segment_callback = whisper_new_segment_cb;
    params.new_segment_callback_user_data = (void*)(ctx);
    params.encoder_begin_callback = whisper_encoder_begin_cb;
    params.encoder_begin_callback_user_data = (void*)(ctx);
    # 设置进度回调函数为 whisper_progress_cb
    params.progress_callback = whisper_progress_cb;
    # 设置进度回调函数的用户数据为 ctx
    params.progress_callback_user_data = (void*)(ctx);
    # 返回设置好的参数对象
    return params;
// 导入 C 包
import "C"

///////////////////////////////////////////////////////////////////////////////
// TYPES

// 定义类型别名
type (
    Context          C.struct_whisper_context
    Token            C.whisper_token
    TokenData        C.struct_whisper_token_data
    SamplingStrategy C.enum_whisper_sampling_strategy
    Params           C.struct_whisper_full_params
)

///////////////////////////////////////////////////////////////////////////////
// GLOBALS

// 定义常量
const (
    SAMPLING_GREEDY      SamplingStrategy = C.WHISPER_SAMPLING_GREEDY
    SAMPLING_BEAM_SEARCH SamplingStrategy = C.WHISPER_SAMPLING_BEAM_SEARCH
)

const (
    SampleRate = C.WHISPER_SAMPLE_RATE                 // 期望的采样率，每秒采样数
    SampleBits = uint16(unsafe.Sizeof(C.float(0))) * 8 // 采样位数
    NumFFT     = C.WHISPER_N_FFT
    HopLength  = C.WHISPER_HOP_LENGTH
    ChunkSize  = C.WHISPER_CHUNK_SIZE
)

// 定义全局变量
var (
    ErrTokenizerFailed  = errors.New("whisper_tokenize failed")
    ErrAutoDetectFailed = errors.New("whisper_lang_auto_detect failed")
    ErrConversionFailed = errors.New("whisper_convert failed")
    ErrInvalidLanguage  = errors.New("invalid language")
)

///////////////////////////////////////////////////////////////////////////////
// PUBLIC METHODS

// 初始化模型并加载给定文件的模型
// 失败时返回 NULL
func Whisper_init(path string) *Context {
    // 将 Go 字符串转换为 C 字符串
    cPath := C.CString(path)
    defer C.free(unsafe.Pointer(cPath))
    // 调用 C 函数初始化模型
    if ctx := C.whisper_init_from_file_with_params(cPath, C.whisper_context_default_params()); ctx != nil {
        return (*Context)(ctx)
    } else {
        return nil
    }
}

// 释放模型分配的所有内存
func (ctx *Context) Whisper_free() {
    C.whisper_free((*C.struct_whisper_context)(ctx))
}

// 将原始 PCM 音频转换为对数梅尔频谱图
// 结果存储在提供的 whisper 上下文中
// 将 PCM 数据转换为 Mel 频谱数据，使用指定的线程数
func (ctx *Context) Whisper_pcm_to_mel(data []float32, threads int) error {
    // 调用 C 函数 whisper_pcm_to_mel 将 PCM 数据转换为 Mel 频谱数据
    if C.whisper_pcm_to_mel((*C.struct_whisper_context)(ctx), (*C.float)(&data[0]), C.int(len(data)), C.int(threads)) == 0 {
        return nil
    } else {
        return ErrConversionFailed
    }
}

// 设置自定义的 Mel 频谱数据到提供的 whisper 上下文中
// 如果要提供自定义的 Mel 频谱数据，请使用此函数，n_mel 必须为 80
func (ctx *Context) Whisper_set_mel(data []float32, n_mel int) error {
    // 调用 C 函数 whisper_set_mel 将自定义的 Mel 频谱数据设置到 whisper 上下文中
    if C.whisper_set_mel((*C.struct_whisper_context)(ctx), (*C.float)(&data[0]), C.int(len(data)), C.int(n_mel)) == 0 {
        return nil
    } else {
        return ErrConversionFailed
    }
}

// 在提供的 whisper 上下文中运行 Whisper 编码器，对存储的 Mel 频谱数据进行编码
// 确保先调用 whisper_pcm_to_mel() 或 whisper_set_mel()
// offset 可以用于指定频谱中第一帧的偏移量
func (ctx *Context) Whisper_encode(offset, threads int) error {
    // 调用 C 函数 whisper_encode 在提供的 whisper 上下文中运行 Whisper 编码器
    if C.whisper_encode((*C.struct_whisper_context)(ctx), C.int(offset), C.int(threads)) == 0 {
        return nil
    } else {
        return ErrConversionFailed
    }
}

// 运行 Whisper 解码器以获取下一个标记的对数和概率
// 确保先调用 whisper_encode()
// tokens + n_tokens 是解码器的提供上下文
// n_past 是要使用的来自先前解码器调用的标记数
func (ctx *Context) Whisper_decode(tokens []Token, past, threads int) error {
    // 调用 C 函数 whisper_decode 在提供的 whisper 上下文中运行 Whisper 解码器
    if C.whisper_decode((*C.struct_whisper_context)(ctx), (*C.whisper_token)(&tokens[0]), C.int(len(tokens)), C.int(past), C.int(threads)) == 0 {
        return nil
    } else {
        return ErrConversionFailed
    }
}

// 将提供的文本转换为标记。tokens 指针必须足够大以容纳生成的标记
// 成功时返回标记数
// 使用 Whisper 模型对文本进行分词，返回分词结果和可能的错误
func (ctx *Context) Whisper_tokenize(text string, tokens []Token) (int, error) {
    // 将 Go 字符串转换为 C 字符串
    cText := C.CString(text)
    // 在函数返回时释放 C 字符串的内存
    defer C.free(unsafe.Pointer(cText))
    // 调用 C 函数 whisper_tokenize 处理文本分词
    if n := C.whisper_tokenize((*C.struct_whisper_context)(ctx), cText, (*C.whisper_token)(&tokens[0]), C.int(len(tokens))); n >= 0 {
        return int(n), nil
    } else {
        return 0, ErrTokenizerFailed
    }
}

// 返回指定语言的 ID，如果未找到则返回 -1
func (ctx *Context) Whisper_lang_id(lang string) int {
    return int(C.whisper_lang_id(C.CString(lang)))
}

// 返回最大的语言 ID（即可用语言数量 - 1）
func Whisper_lang_max_id() int {
    return int(C.whisper_lang_max_id())
}

// 返回指定语言 ID 的短字符串（例如 2 -> "de"），如果未找到则返回空字符串
func Whisper_lang_str(id int) string {
    return C.GoString(C.whisper_lang_str(C.int(id)))
}

// 使用偏移时间 offset_ms 的 mel 数据尝试自动检测所说语言
// 确保先调用 whisper_pcm_to_mel() 或 whisper_set_mel()
// 返回所有语言的概率
func (ctx *Context) Whisper_lang_auto_detect(offset_ms, n_threads int) ([]float32, error) {
    // 创建概率数组
    probs := make([]float32, Whisper_lang_max_id()+1)
    // 调用 C 函数 whisper_lang_auto_detect 进行语言自动检测
    if n := int(C.whisper_lang_auto_detect((*C.struct_whisper_context)(ctx), C.int(offset_ms), C.int(n_threads), (*C.float)(&probs[0]))); n < 0 {
        return nil, ErrAutoDetectFailed
    } else {
        return probs, nil
    }
}

// 返回 Whisper 模型中 n 的长度
func (ctx *Context) Whisper_n_len() int {
    return int(C.whisper_n_len((*C.struct_whisper_context)(ctx)))
}

// 返回 Whisper 模型中词汇表的大小
func (ctx *Context) Whisper_n_vocab() int {
    return int(C.whisper_n_vocab((*C.struct_whisper_context)(ctx)))
}

// 返回 Whisper 模型中文本上下文的数量
func (ctx *Context) Whisper_n_text_ctx() int {
    return int(C.whisper_n_text_ctx((*C.struct_whisper_context)(ctx)))
}

// 返回 Whisper 模型中音频上下文的数量
func (ctx *Context) Whisper_n_audio_ctx() {
    // 未提供具体实现，需要根据具体情况补充
}
    # 调用 C 库中的 whisper_n_audio_ctx 函数，并将 ctx 转换为 C.struct_whisper_context 类型后作为参数传入，返回一个整数结果
    return int(C.whisper_n_audio_ctx((*C.struct_whisper_context)(ctx)))
// 检查当前上下文是否支持多语言
func (ctx *Context) Whisper_is_multilingual() int {
    return int(C.whisper_is_multilingual((*C.struct_whisper_context)(ctx)))
}

// 将 Token Id 转换为对应的字符串，使用提供上下文中的词汇表
func (ctx *Context) Whisper_token_to_str(token Token) string {
    return C.GoString(C.whisper_token_to_str((*C.struct_whisper_context)(ctx), C.whisper_token(token)))
}

// 返回结束标记的 Token
func (ctx *Context) Whisper_token_eot() Token {
    return Token(C.whisper_token_eot((*C.struct_whisper_context)(ctx)))
}

// 返回起始标记的 Token
func (ctx *Context) Whisper_token_sot() Token {
    return Token(C.whisper_token_sot((*C.struct_whisper_context)(ctx)))
}

// 返回前一个标记的 Token
func (ctx *Context) Whisper_token_prev() Token {
    return Token(C.whisper_token_prev((*C.struct_whisper_context)(ctx)))
}

// 返回语言开始标记的 Token
func (ctx *Context) Whisper_token_solm() Token {
    return Token(C.whisper_token_solm((*C.struct_whisper_context)(ctx)))
}

// 返回否定标记的 Token
func (ctx *Context) Whisper_token_not() Token {
    return Token(C.whisper_token_not((*C.struct_whisper_context)(ctx)))
}

// 返回开始标记的 Token
func (ctx *Context) Whisper_token_beg() Token {
    return Token(C.whisper_token_beg((*C.struct_whisper_context)(ctx)))
}

// 返回指定语言标记的 Token
func (ctx *Context) Whisper_token_lang(lang_id int) Token {
    return Token(C.whisper_token_lang((*C.struct_whisper_context)(ctx), C.int(lang_id)))
}

// 返回翻译任务标记的 Token
func (ctx *Context) Whisper_token_translate() Token {
    return Token(C.whisper_token_translate((*C.struct_whisper_context)(ctx)))
}

// 返回转录任务标记的 Token
func (ctx *Context) Whisper_token_transcribe() Token {
    return Token(C.whisper_token_transcribe((*C.struct_whisper_context)(ctx)))
}
// 打印时间信息
func (ctx *Context) Whisper_print_timings() {
    // 调用 C 函数 whisper_print_timings，传入 Context 结构体指针作为参数
    C.whisper_print_timings((*C.struct_whisper_context)(ctx))
}

// 重置时间信息
func (ctx *Context) Whisper_reset_timings() {
    // 调用 C 函数 whisper_reset_timings，传入 Context 结构体指针作为参数
    C.whisper_reset_timings((*C.struct_whisper_context)(ctx))
}

// 打印系统信息
func Whisper_print_system_info() string {
    // 调用 C 函数 whisper_print_system_info，并将返回的 C 字符串转换为 Go 字符串
    return C.GoString(C.whisper_print_system_info())
}

// 返回指定策略的默认参数
func (ctx *Context) Whisper_full_default_params(strategy SamplingStrategy) Params {
    // 调用 C 回调函数 whisper_full_default_params_cb，传入 Context 结构体指针和 SamplingStrategy 枚举值作为参数
    // 将返回的参数转换为 Params 结构体
    return Params(C.whisper_full_default_params_cb((*C.struct_whisper_context)(ctx), C.enum_whisper_sampling_strategy(strategy)))
}

// 运行整个模型：PCM -> 对数梅尔频谱图 -> 编码器 -> 解码器 -> 文本
// 使用指定的解码策略获取文本
func (ctx *Context) Whisper_full(
    params Params,
    samples []float32,
    encoderBeginCallback func() bool,
    newSegmentCallback func(int),
    progressCallback func(int),
) error {
    // 注册编码器开始回调函数、新段回调函数和进度回调函数
    registerEncoderBeginCallback(ctx, encoderBeginCallback)
    registerNewSegmentCallback(ctx, newSegmentCallback)
    registerProgressCallback(ctx, progressCallback)
    // 在函数返回时取消注册回调函数
    defer registerEncoderBeginCallback(ctx, nil)
    defer registerNewSegmentCallback(ctx, nil)
    defer registerProgressCallback(ctx, nil)
    // 调用 C 函数 whisper_full，传入 Context 结构体指针、Params 结构体、样本数据、样本长度作为参数
    // 返回值为 0 表示成功，否则返回 ErrConversionFailed 错误
    if C.whisper_full((*C.struct_whisper_context)(ctx), (C.struct_whisper_full_params)(params), (*C.float)(&samples[0]), C.int(len(samples))) == 0 {
        return nil
    } else {
        return ErrConversionFailed
    }
}

// 将输入音频分成块，并分别使用 whisper_full() 处理每个块
// 在某些情况下，这种方法可能会提供一些加速
// 但在每个块的开头和结尾，转录准确性可能会较差
func (ctx *Context) Whisper_full_parallel(params Params, samples []float32, processors int, encoderBeginCallback func() bool, newSegmentCallback func(int)) error {
    # 注册编码器开始回调函数
    registerEncoderBeginCallback(ctx, encoderBeginCallback)
    # 注册新段回调函数
    registerNewSegmentCallback(ctx, newSegmentCallback)
    # 在函数返回时取消注册编码器开始回调函数
    defer registerEncoderBeginCallback(ctx, nil)
    # 在函数返回时取消注册新段回调函数
    defer registerNewSegmentCallback(ctx, nil)
    
    # 调用 C 语言函数 whisper_full_parallel 进行全并行处理
    if C.whisper_full_parallel((*C.struct_whisper_context)(ctx), (C.struct_whisper_full_params)(params), (*C.float)(&samples[0]), C.int(len(samples)), C.int(processors)) == 0:
        # 如果处理成功，返回空
        return nil
    else:
        # 如果处理失败，返回错误信息 ErrConversionFailed
        return ErrConversionFailed
// 返回自动检测到的语言的ID，如果未找到则返回-1
// 在 https://github.com/ggerganov/whisper.cpp/commit/a1c1583cc7cd8b75222857afc936f0638c5683d6 中添加到 whisper.cpp
//
// 示例:
//
//    "de" -> 2
//    "german" -> 2
func (ctx *Context) Whisper_full_lang_id() int {
    return int(C.whisper_full_lang_id((*C.struct_whisper_context)(ctx)))
}

// 生成的文本段数。
// 一个段落可以是几个单词，一个句子，甚至是一个段落。
func (ctx *Context) Whisper_full_n_segments() int {
    return int(C.whisper_full_n_segments((*C.struct_whisper_context)(ctx)))
}

// 获取指定段落的开始和结束时间。
func (ctx *Context) Whisper_full_get_segment_t0(segment int) int64 {
    return int64(C.whisper_full_get_segment_t0((*C.struct_whisper_context)(ctx), C.int(segment)))
}

// 获取指定段落的开始和结束时间。
func (ctx *Context) Whisper_full_get_segment_t1(segment int) int64 {
    return int64(C.whisper_full_get_segment_t1((*C.struct_whisper_context)(ctx), C.int(segment)))
}

// 获取指定段落的文本。
func (ctx *Context) Whisper_full_get_segment_text(segment int) string {
    return C.GoString(C.whisper_full_get_segment_text((*C.struct_whisper_context)(ctx), C.int(segment)))
}

// 获取指定段落中的标记数。
func (ctx *Context) Whisper_full_n_tokens(segment int) int {
    return int(C.whisper_full_n_tokens((*C.struct_whisper_context)(ctx), C.int(segment)))
}

// 获取指定段落中指定标记索引的标记文本。
func (ctx *Context) Whisper_full_get_token_text(segment int, token int) string {
    return C.GoString(C.whisper_full_get_token_text((*C.struct_whisper_context)(ctx), C.int(segment), C.int(token)))
}

// 获取指定段落中指定标记索引的标记。
func (ctx *Context) Whisper_full_get_token_id(segment int, token int) Token {
    # 调用 C 语言函数 whisper_full_get_token_id，传入参数 ctx（转换为 C 结构体指针）、segment、token，并返回结果
    return Token(C.whisper_full_get_token_id((*C.struct_whisper_context)(ctx), C.int(segment), C.int(token)))
// 获取指定段落中指定标记的标记数据，包括概率、时间戳等信息
func (ctx *Context) Whisper_full_get_token_data(segment int, token int) TokenData {
    return TokenData(C.whisper_full_get_token_data((*C.struct_whisper_context)(ctx), C.int(segment), C.int(token)))
}

// 获取指定段落中指定标记的概率
func (ctx *Context) Whisper_full_get_token_p(segment int, token int) float32 {
    return float32(C.whisper_full_get_token_p((*C.struct_whisper_context)(ctx), C.int(segment), C.int(token)))
}

///////////////////////////////////////////////////////////////////////////////
// 回调函数

// 定义新段落回调函数
var (
    cbNewSegment   = make(map[unsafe.Pointer]func(int))
    cbProgress     = make(map[unsafe.Pointer]func(int))
    cbEncoderBegin = make(map[unsafe.Pointer]func() bool)
)

// 注册新段落回调函数
func registerNewSegmentCallback(ctx *Context, fn func(int)) {
    if fn == nil {
        删除新段落回调函数
        delete(cbNewSegment, unsafe.Pointer(ctx))
    } else {
        注册新段落回调函数
        cbNewSegment[unsafe.Pointer(ctx)] = fn
    }
}

// 注册进度回调函数
func registerProgressCallback(ctx *Context, fn func(int)) {
    if fn == nil {
        删除进度回调函数
        delete(cbProgress, unsafe.Pointer(ctx))
    } else {
        注册进度回调函数
        cbProgress[unsafe.Pointer(ctx)] = fn
    }
}

// 注册编码器开始回调函数
func registerEncoderBeginCallback(ctx *Context, fn func() bool) {
    if fn == nil {
        删除编码器开始回调函数
        delete(cbEncoderBegin, unsafe.Pointer(ctx))
    } else {
        注册编码器开始回调函数
        cbEncoderBegin[unsafe.Pointer(ctx)] = fn
    }
}

// 导出新段落回调函数
//export callNewSegment
func callNewSegment(user_data unsafe.Pointer, new C.int) {
    if fn, ok := cbNewSegment[user_data]; ok {
        调用新段落回调函数
        fn(int(new))
    }
}

// 导出进度回调函数
//export callProgress
func callProgress(user_data unsafe.Pointer, progress C.int) {
    if fn, ok := cbProgress[user_data]; ok {
        调用进度回调函数
        fn(int(progress))
    }
}

// 导出编码器开始回调函数
//export callEncoderBegin
func callEncoderBegin(user_data unsafe.Pointer) C.bool {
    // 检查是否存在与用户数据对应的回调函数
    if fn, ok := cbEncoderBegin[user_data]; ok {
        // 如果存在回调函数，则调用该函数
        if fn() {
            // 如果回调函数返回 true，则返回 C 语言中的 true
            return C.bool(true)
        } else {
            // 如果回调函数返回 false，则返回 C 语言中的 false
            return C.bool(false)
        }
    }
    // 如果不存在回调函数，则返回 true
    return true
# 定义 TokenData 结构体的方法 T0，返回 t0 字段的值
func (t TokenData) T0() int64 {
    return int64(t.t0)
}

# 定义 TokenData 结构体的方法 T1，返回 t1 字段的值
func (t TokenData) T1() int64 {
    return int64(t.t1)
}

# 定义 TokenData 结构体的方法 Id，返回 id 字段的值，并将其转换为 Token 类型
func (t TokenData) Id() Token {
    return Token(t.id)
}
```