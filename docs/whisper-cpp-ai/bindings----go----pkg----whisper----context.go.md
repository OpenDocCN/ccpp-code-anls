# `whisper.cpp\bindings\go\pkg\whisper\context.go`

```cpp
// 导入必要的包
package whisper

import (
    "fmt"
    "io"
    "runtime"
    "strings"
    "time"

    // 导入绑定的库
    whisper "github.com/ggerganov/whisper.cpp/bindings/go"
)

///////////////////////////////////////////////////////////////////////////////
// TYPES

// 定义上下文结构体
type context struct {
    n      int
    model  *model
    params whisper.Params
}

// 确保 context 结构体符合接口 Context
var _ Context = (*context)(nil)

///////////////////////////////////////////////////////////////////////////////
// LIFECYCLE

// 创建新的上下文对象
func newContext(model *model, params whisper.Params) (Context, error) {
    context := new(context)
    context.model = model
    context.params = params

    // 返回成功
    return context, nil
}

///////////////////////////////////////////////////////////////////////////////
// PUBLIC METHODS

// 设置语音识别的语言
func (context *context) SetLanguage(lang string) error {
    // 检查上下文的模型是否为空
    if context.model.ctx == nil {
        return ErrInternalAppError
    }
    // 检查模型是否支持多语言
    if !context.model.IsMultilingual() {
        return ErrModelNotMultilingual
    }

    // 根据语言设置语言参数
    if lang == "auto" {
        context.params.SetLanguage(-1)
    } else if id := context.model.ctx.Whisper_lang_id(lang); id < 0 {
        return ErrUnsupportedLanguage
    } else if err := context.params.SetLanguage(id); err != nil {
        return err
    }
    // 返回成功
    return nil
}

// 检查是否支持多语言
func (context *context) IsMultilingual() bool {
    return context.model.IsMultilingual()
}

// 获取语言
func (context *context) Language() string {
    id := context.params.Language()
    if id == -1 {
        return "auto"
    }
    return whisper.Whisper_lang_str(context.params.Language())
}

// 设置翻译标志
func (context *context) SetTranslate(v bool) {
    context.params.SetTranslate(v)
}

// 设置加速标志
func (context *context) SetSpeedup(v bool) {
    context.params.SetSpeedup(v)
}

// 设置按单词拆分标志
func (context *context) SetSplitOnWord(v bool) {
    context.params.SetSplitOnWord(v)
}
// 设置要使用的线程数
func (context *context) SetThreads(v uint) {
    context.params.SetThreads(int(v))
}

// 设置时间偏移量
func (context *context) SetOffset(v time.Duration) {
    context.params.SetOffset(int(v.Milliseconds()))
}

// 设置要处理的音频持续时间
func (context *context) SetDuration(v time.Duration) {
    context.params.SetDuration(int(v.Milliseconds()))
}

// 设置时间戳令牌概率阈值（~0.01）
func (context *context) SetTokenThreshold(t float32) {
    context.params.SetTokenThreshold(t)
}

// 设置时间戳令牌总概率阈值（~0.01）
func (context *context) SetTokenSumThreshold(t float32) {
    context.params.SetTokenSumThreshold(t)
}

// 设置每个段的最大长度（以字符为单位）
func (context *context) SetMaxSegmentLength(n uint) {
    context.params.SetMaxSegmentLength(int(n))
}

// 设置时间戳令牌标志
func (context *context) SetTokenTimestamps(b bool) {
    context.params.SetTokenTimestamps(b)
}

// 设置每个段的最大令牌数（0 = 无限制）
func (context *context) SetMaxTokensPerSegment(n uint) {
    context.params.SetMaxTokensPerSegment(int(n))
}

// 设置音频编码器上下文
func (context *context) SetAudioCtx(n uint) {
    context.params.SetAudioCtx(int(n))
}

// 设置初始提示
func (context *context) SetInitialPrompt(prompt string) {
    context.params.SetInitialPrompt(prompt)
}

// 重置模式计时。应在处理之前调用
func (context *context) ResetTimings() {
    context.model.ctx.Whisper_reset_timings()
}

// 将模型计时打印到标准输出
func (context *context) PrintTimings() {
    context.model.ctx.Whisper_print_timings()
}

// 返回系统信息
func (context *context) SystemInfo() string {
    return fmt.Sprintf("system_info: n_threads = %d / %d | %s\n",
        context.params.Threads(),
        runtime.NumCPU(),
        whisper.Whisper_print_system_info(),
    )
}
// 使用偏移量 offset_ms 的 mel 数据尝试自动检测所说的语言
// 确保先调用 whisper_pcm_to_mel() 或 whisper_set_mel()
// 返回所有语言的概率
func (context *context) WhisperLangAutoDetect(offset_ms int, n_threads int) ([]float32, error) {
    // 调用底层的 Whisper_lang_auto_detect 方法获取语言概率
    langProbs, err := context.model.ctx.Whisper_lang_auto_detect(offset_ms, n_threads)
    if err != nil {
        return nil, err
    }
    return langProbs, nil
}

// 处理新的样本数据并返回任何错误
func (context *context) Process(
    data []float32,
    callNewSegment SegmentCallback,
    callProgress ProgressCallback,
) error {
    if context.model.ctx == nil {
        return ErrInternalAppError
    }
    // 如果回调函数已定义，则强制启用单段模式
    if callNewSegment != nil {
        context.params.SetSingleSegment(true)
    }

    // 目前我们不进行并行处理
    processors := 0
    if processors > 1 {
        // 如果处理器数量大于1，则调用 Whisper_full_parallel 方法
        if err := context.model.ctx.Whisper_full_parallel(context.params, data, processors, nil, func(new int) {
            if callNewSegment != nil {
                num_segments := context.model.ctx.Whisper_full_n_segments()
                s0 := num_segments - new
                for i := s0; i < num_segments; i++ {
                    callNewSegment(toSegment(context.model.ctx, i))
                }
            }
        }); err != nil {
            return err
        }
    } else if err := context.model.ctx.Whisper_full(context.params, data, nil, func(new int) {
        if callNewSegment != nil {
            num_segments := context.model.ctx.Whisper_full_n_segments()
            s0 := num_segments - new
            for i := s0; i < num_segments; i++ {
                callNewSegment(toSegment(context.model.ctx, i))
            }
        }
    }, func(progress int) {
        if callProgress != nil {
            callProgress(progress)
        }
    }); err != nil {
        return err
    }

    // 返回成功
}
    # 返回空值
    return nil
// 返回下一个 token 段
func (context *context) NextSegment() (Segment, error) {
    // 检查上下文中的模型是否为空
    if context.model.ctx == nil {
        return Segment{}, ErrInternalAppError
    }
    // 检查当前 token 段是否超过了模型中的最大段数
    if context.n >= context.model.ctx.Whisper_full_n_segments() {
        return Segment{}, io.EOF
    }

    // 生成结果
    result := toSegment(context.model.ctx, context.n)

    // 增加游标
    context.n++

    // 返回成功
    return result, nil
}

// 检查是否为文本 token
func (context *context) IsText(t Token) bool {
    switch {
    case context.IsBEG(t):
        return false
    case context.IsSOT(t):
        return false
    case whisper.Token(t.Id) >= context.model.ctx.Whisper_token_eot():
        return false
    case context.IsPREV(t):
        return false
    case context.IsSOLM(t):
        return false
    case context.IsNOT(t):
        return false
    default:
        return true
    }
}

// 检查是否为 "begin" token
func (context *context) IsBEG(t Token) bool {
    return whisper.Token(t.Id) == context.model.ctx.Whisper_token_beg()
}

// 检查是否为 "start of transcription" token
func (context *context) IsSOT(t Token) bool {
    return whisper.Token(t.Id) == context.model.ctx.Whisper_token_sot()
}

// 检查是否为 "end of transcription" token
func (context *context) IsEOT(t Token) bool {
    return whisper.Token(t.Id) == context.model.ctx.Whisper_token_eot()
}

// 检查是否为 "start of prev" token
func (context *context) IsPREV(t Token) bool {
    return whisper.Token(t.Id) == context.model.ctx.Whisper_token_prev()
}

// 检查是否为 "start of lm" token
func (context *context) IsSOLM(t Token) bool {
    return whisper.Token(t.Id) == context.model.ctx.Whisper_token_solm()
}

// 检查是否为 "No timestamps" token
func (context *context) IsNOT(t Token) bool {
    return whisper.Token(t.Id) == context.model.ctx.Whisper_token_not()
}

// 检查是否为与特定语言相关的 token
func (context *context) IsLANG(t Token, lang string) bool {
    # 使用 := 运算符同时进行变量赋值和判断，获取指定语言的 ID
    if id := context.model.ctx.Whisper_lang_id(lang); id >= 0 {
        # 如果 ID 大于等于 0，则返回指定语言的令牌是否等于指定语言的令牌
        return whisper.Token(t.Id) == context.model.ctx.Whisper_token_lang(id)
    } else {
        # 如果 ID 小于 0，则返回 false
        return false
    }
// 将整数 n 转换为 Segment 结构体
func toSegment(ctx *whisper.Context, n int) Segment {
    // 返回一个 Segment 结构体，包含 Num、Text、Start、End 和 Tokens 字段
    return Segment{
        Num:    n,
        Text:   strings.TrimSpace(ctx.Whisper_full_get_segment_text(n)), // 获取段落文本并去除首尾空格
        Start:  time.Duration(ctx.Whisper_full_get_segment_t0(n)) * time.Millisecond * 10, // 获取段落起始时间并转换为纳秒
        End:    time.Duration(ctx.Whisper_full_get_segment_t1(n)) * time.Millisecond * 10, // 获取段落结束时间并转换为纳秒
        Tokens: toTokens(ctx, n), // 调用 toTokens 函数获取段落的 Token 列表
    }
}

// 将整数 n 转换为 Token 列表
func toTokens(ctx *whisper.Context, n int) []Token {
    // 创建一个 Token 结构体切片，长度为段落 n 的 Token 数量
    result := make([]Token, ctx.Whisper_full_n_tokens(n))
    // 遍历 Token 结构体切片
    for i := 0; i < len(result); i++ {
        // 获取第 i 个 Token 的数据
        data := ctx.Whisper_full_get_token_data(n, i)

        // 将 Token 结构体的字段填充
        result[i] = Token{
            Id:    int(ctx.Whisper_full_get_token_id(n, i)), // 获取 Token 的 ID
            Text:  ctx.Whisper_full_get_token_text(n, i), // 获取 Token 的文本
            P:     ctx.Whisper_full_get_token_p(n, i), // 获取 Token 的概率
            Start: time.Duration(data.T0()) * time.Millisecond * 10, // 获取 Token 的起始时间并转换为纳秒
            End:   time.Duration(data.T1()) * time.Millisecond * 10, // 获取 Token 的结束时间并转换为纳秒
        }
    }
    // 返回填充好的 Token 结构体切片
    return result
}
```