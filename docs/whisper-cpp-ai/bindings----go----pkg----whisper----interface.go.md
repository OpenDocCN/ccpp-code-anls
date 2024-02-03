# `whisper.cpp\bindings\go\pkg\whisper\interface.go`

```cpp
// 包 whisper 包含了语音识别模型的相关功能
package whisper

import (
    "io"    // 导入 io 包，用于实现 I/O 操作
    "time"  // 导入 time 包，用于处理时间相关操作
)

///////////////////////////////////////////////////////////////////////////////
// TYPES

// SegmentCallback 是用于实时处理片段的回调函数，在 Process 函数中调用
type SegmentCallback func(Segment)

// ProgressCallback 是用于在处理过程中报告进度的回调函数，在 Process 函数中调用
type ProgressCallback func(int)

// Model 是 whisper 模型的接口。使用 whisper.New(string) 函数创建一个新模型
type Model interface {
    io.Closer

    // 返回一个新的语音识别上下文
    NewContext() (Context, error)

    // 如果模型支持多语言，则返回 true
    IsMultilingual() bool

    // 返回所有支持的语言
    Languages() []string
}

// Context 是语音识别上下文
type Context interface {
    SetLanguage(string) error // 设置用于语音识别的语言，使用 "auto" 进行自动检测
    SetTranslate(bool)        // 设置翻译标志
    IsMultilingual() bool     // 如果模型支持多语言，则返回 true
    Language() string         // 获取语言

    SetOffset(time.Duration)        // 设置偏移量
    SetDuration(time.Duration)      // 设置持续时间
    SetThreads(uint)                // 设置要使用的线程数
    SetSpeedup(bool)                // 设置加速标志
    SetSplitOnWord(bool)            // 设置按单词拆分标志
    SetTokenThreshold(float32)      // 设置时间戳令牌概率阈值
    SetTokenSumThreshold(float32)   // 设置时间戳令牌总概率阈值
    SetMaxSegmentLength(uint)       // 设置片段的最大长度（以字符计）
    SetTokenTimestamps(bool)        // 设置令牌时间戳标志
    SetMaxTokensPerSegment(uint)    // 设置每个片段的最大令牌数（0 = 无限制）
    SetAudioCtx(uint)               // 设置音频编码器上下文
    SetInitialPrompt(prompt string) // 设置初始提示
    // 处理单声道音频数据并返回任何错误。
    // 如果定义了，新生成的片段在处理过程中传递给回调函数。
    Process([]float32, SegmentCallback, ProgressCallback) error

    // 在调用 process 后，返回直到流的末尾的片段，当返回 io.EOF 时表示已到达末尾。
    NextSegment() (Segment, error)

    IsBEG(Token) bool          // 检测是否为 "begin" 标记
    IsSOT(Token) bool          // 检测是否为 "start of transcription" 标记
    IsEOT(Token) bool          // 检测是否为 "end of transcription" 标记
    IsPREV(Token) bool         // 检测是否为 "start of prev" 标记
    IsSOLM(Token) bool         // 检测是否为 "start of lm" 标记
    IsNOT(Token) bool          // 检测是否为 "No timestamps" 标记
    IsLANG(Token, string) bool // 检测是否为与特定语言相关联的标记
    IsText(Token) bool         // 检测是否为文本标记

    // 时间信息
    PrintTimings()
    ResetTimings()

    SystemInfo() string
// Segment 是语音识别的文本结果。
type Segment struct {
    // Segment Number 表示片段编号
    Num int

    // Time beginning and end timestamps for the segment. 表示片段的开始和结束时间戳。
    Start, End time.Duration

    // The text of the segment. 表示片段的文本内容。
    Text string

    // The tokens of the segment. 表示片段的标记。
    Tokens []Token
}

// Token 是文本或特殊标记。
type Token struct {
    Id         int
    Text       string
    P          float32
    Start, End time.Duration
}
```