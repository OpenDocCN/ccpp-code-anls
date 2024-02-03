# `whisper.cpp\bindings\go\examples\go-whisper\flags.go`

```cpp
package main

import (
    "flag" // 导入 flag 包，用于解析命令行参数
    "fmt" // 导入 fmt 包，用于格式化输出
    "strings" // 导入 strings 包，用于处理字符串
    "time" // 导入 time 包，用于处理时间

    // 导入 whisper 包，用于处理 whisper 数据
    whisper "github.com/ggerganov/whisper.cpp/bindings/go/pkg/whisper"
)

///////////////////////////////////////////////////////////////////////////////
// TYPES

type Flags struct {
    *flag.FlagSet // 定义 Flags 结构体，包含一个 flag.FlagSet 类型的字段
}

///////////////////////////////////////////////////////////////////////////////
// LIFECYCLE

func NewFlags(name string, args []string) (*Flags, error) {
    flags := &Flags{
        FlagSet: flag.NewFlagSet(name, flag.ContinueOnError), // 创建一个新的 FlagSet 对象
    }

    // 注册命令行参数
    registerFlags(flags)

    // 解析命令行参数
    if err := flags.Parse(args); err != nil {
        return nil, err
    }

    // 返回成功
    return flags, nil
}

///////////////////////////////////////////////////////////////////////////////
// PUBLIC METHODS

func (flags *Flags) GetModel() string {
    return flags.Lookup("model").Value.String() // 获取 model 参数的值并返回
}

func (flags *Flags) GetLanguage() string {
    return flags.Lookup("language").Value.String() // 获取 language 参数的值并返回
}

func (flags *Flags) IsTranslate() bool {
    return flags.Lookup("translate").Value.(flag.Getter).Get().(bool) // 获取 translate 参数的值并返回布尔类型
}

func (flags *Flags) GetOffset() time.Duration {
    return flags.Lookup("offset").Value.(flag.Getter).Get().(time.Duration) // 获取 offset 参数的值并返回时间类型
}

func (flags *Flags) GetDuration() time.Duration {
    return flags.Lookup("duration").Value.(flag.Getter).Get().(time.Duration) // 获取 duration 参数的值并返回时间类型
}

func (flags *Flags) GetThreads() uint {
    return flags.Lookup("threads").Value.(flag.Getter).Get().(uint) // 获取 threads 参数的值并返回无符号整数类型
}

func (flags *Flags) GetOut() string {
    return strings.ToLower(flags.Lookup("out").Value.String()) // 获取 out 参数的值并转换为小写字符串后返回
}

func (flags *Flags) IsSpeedup() bool {
    return flags.Lookup("speedup").Value.String() == "true" // 判断 speedup 参数的值是否为 "true" 并返回布尔类型
}

func (flags *Flags) IsTokens() bool {
    return flags.Lookup("tokens").Value.String() == "true" // 判断 tokens 参数的值是否为 "true" 并返回布尔类型
}

func (flags *Flags) IsColorize() bool {
    return flags.Lookup("colorize").Value.String() == "true" // 判断 colorize 参数的值是否为 "true" 并返回布尔类型
}

func (flags *Flags) GetMaxLen() uint {
    # 查找名为"max-len"的标志，并获取其值
    return flags.Lookup("max-len").Value.(flag.Getter).Get().(uint)
// 获取 Flags 结构体中 max-tokens 参数的值，并转换为 uint 类型后返回
func (flags *Flags) GetMaxTokens() uint {
    return flags.Lookup("max-tokens").Value.(flag.Getter).Get().(uint)
}

// 获取 Flags 结构体中 word-thold 参数的值，并转换为 float32 类型后返回
func (flags *Flags) GetWordThreshold() float32 {
    return float32(flags.Lookup("word-thold").Value.(flag.Getter).Get().(float64))
}

// 设置参数到 whisper.Context 中
func (flags *Flags) SetParams(context whisper.Context) error {
    // 如果语言不为空且不为"auto"，则设置语言并输出信息
    if lang := flags.GetLanguage(); lang != "" && lang != "auto" {
        fmt.Fprintf(flags.Output(), "Setting language to %q\n", lang)
        if err := context.SetLanguage(lang); err != nil {
            return err
        }
    }
    // 如果需要翻译且上下文支持多语言，则设置翻译并输出信息
    if flags.IsTranslate() && context.IsMultilingual() {
        fmt.Fprintf(flags.Output(), "Setting translate to true\n")
        context.SetTranslate(true)
    }
    // 如果偏移量不为0，则设置偏移量并输出信息
    if offset := flags.GetOffset(); offset != 0 {
        fmt.Fprintf(flags.Output(), "Setting offset to %v\n", offset)
        context.SetOffset(offset)
    }
    // 如果持续时间不为0，则设置持续时间并输出信息
    if duration := flags.GetDuration(); duration != 0 {
        fmt.Fprintf(flags.Output(), "Setting duration to %v\n", duration)
        context.SetDuration(duration)
    }
    // 如果需要加速，则设置加速并输出信息
    if flags.IsSpeedup() {
        fmt.Fprintf(flags.Output(), "Setting speedup to true\n")
        context.SetSpeedup(true)
    }
    // 如果线程数不为0，则设置线程数并输出信息
    if threads := flags.GetThreads(); threads != 0 {
        fmt.Fprintf(flags.Output(), "Setting threads to %d\n", threads)
        context.SetThreads(threads)
    }
    // 如果最大长度不为0，则设置最大分段长度并输出信息
    if max_len := flags.GetMaxLen(); max_len != 0 {
        fmt.Fprintf(flags.Output(), "Setting max_segment_length to %d\n", max_len)
        context.SetMaxSegmentLength(max_len)
    }
    // 如果最大 token 数不为0，则设置每个分段的最大 token 数并输出信息
    if max_tokens := flags.GetMaxTokens(); max_tokens != 0 {
        fmt.Fprintf(flags.Output(), "Setting max_tokens to %d\n", max_tokens)
        context.SetMaxTokensPerSegment(max_tokens)
    }
    // 如果词阈值不为0，则设置词阈值并输出信息
    if word_threshold := flags.GetWordThreshold(); word_threshold != 0 {
        fmt.Fprintf(flags.Output(), "Setting word_threshold to %f\n", word_threshold)
        context.SetTokenThreshold(word_threshold)
    }

    // 返回成功
    return nil
}
// 注册命令行参数
func registerFlags(flag *Flags) {
    // 注册 model 参数，指定模型文件路径
    flag.String("model", "", "Path to the model file")
    // 注册 language 参数，指定语言
    flag.String("language", "", "Spoken language")
    // 注册 translate 参数，是否从源语言翻译为英语
    flag.Bool("translate", false, "Translate from source language to english")
    // 注册 offset 参数，时间偏移量
    flag.Duration("offset", 0, "Time offset")
    // 注册 duration 参数，音频处理时长
    flag.Duration("duration", 0, "Duration of audio to process")
    // 注册 threads 参数，线程数
    flag.Uint("threads", 0, "Number of threads to use")
    // 注册 speedup 参数，是否启用加速
    flag.Bool("speedup", false, "Enable speedup")
    // 注册 max-len 参数，最大分段长度（字符数）
    flag.Uint("max-len", 0, "Maximum segment length in characters")
    // 注册 max-tokens 参数，每个分段的最大标记数
    flag.Uint("max-tokens", 0, "Maximum tokens per segment")
    // 注册 word-thold 参数，最大分段得分
    flag.Float64("word-thold", 0, "Maximum segment score")
    // 注册 tokens 参数，是否显示标记
    flag.Bool("tokens", false, "Display tokens")
    // 注册 colorize 参数，是否对标记进行着色
    flag.Bool("colorize", false, "Colorize tokens")
    // 注册 out 参数，输出格式（srt, none 或留空）
    flag.String("out", "", "Output format (srt, none or leave as empty string)")
}
```