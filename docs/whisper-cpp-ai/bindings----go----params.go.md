# `whisper.cpp\bindings\go\params.go`

```cpp
// 导入 whisper 包
package whisper

// 导入 fmt 包
import (
    "fmt"
)

///////////////////////////////////////////////////////////////////////////////
// CGO

// 导入 C 语言头文件 whisper.h
/*
#include <whisper.h>
*/
import "C"

///////////////////////////////////////////////////////////////////////////////
// PUBLIC METHODS

// 设置是否进行翻译
func (p *Params) SetTranslate(v bool) {
    p.translate = toBool(v)
}

// 设置是否在单词间分割
func (p *Params) SetSplitOnWord(v bool) {
    p.split_on_word = toBool(v)
}

// 设置是否不使用上下文
func (p *Params) SetNoContext(v bool) {
    p.no_context = toBool(v)
}

// 设置是否只处理单个段落
func (p *Params) SetSingleSegment(v bool) {
    p.single_segment = toBool(v)
}

// 设置是否打印特殊字符
func (p *Params) SetPrintSpecial(v bool) {
    p.print_special = toBool(v)
}

// 设置是否打印处理进度
func (p *Params) SetPrintProgress(v bool) {
    p.print_progress = toBool(v)
}

// 设置是否实时打印结果
func (p *Params) SetPrintRealtime(v bool) {
    p.print_realtime = toBool(v)
}

// 设置是否打印时间戳
func (p *Params) SetPrintTimestamps(v bool) {
    p.print_timestamps = toBool(v)
}

// 设置是否加速处理
func (p *Params) SetSpeedup(v bool) {
    p.speed_up = toBool(v)
}

// 设置语言 ID
func (p *Params) SetLanguage(lang int) error {
    // 如果语言 ID 为 -1，则设置语言为 nil
    if lang == -1 {
        p.language = nil
        return nil
    }
    // 获取语言字符串
    str := C.whisper_lang_str(C.int(lang))
    // 如果语言字符串为空，则返回错误
    if str == nil {
        return ErrInvalidLanguage
    } else {
        p.language = str
    }
    return nil
}

// 获取语言 ID
func (p *Params) Language() int {
    // 如果语言为 nil，则返回 -1
    if p.language == nil {
        return -1
    }
    return int(C.whisper_lang_id(p.language))
}

// 返回可用线程数
func (p *Params) Threads() int {
    return int(p.n_threads)
}

// 设置要使用的线程数
func (p *Params) SetThreads(threads int) {
    p.n_threads = C.int(threads)
}

// 设置开始偏移量（毫秒）
func (p *Params) SetOffset(offset_ms int) {
    p.offset_ms = C.int(offset_ms)
}

// 设置要处理的音频持续时间（毫秒）
func (p *Params) SetDuration(duration_ms int) {
    p.duration_ms = C.int(duration_ms)
}

// 设置时间戳标记概率阈值（~0.01）
func (p *Params) SetTokenThreshold(t float32) {
    p.thold_pt = C.float(t)
}
// 设置时间戳令牌总概率阈值（~0.01）
func (p *Params) SetTokenSumThreshold(t float32) {
    // 将传入的浮点数转换为 C 语言的 float 类型，并设置为参数对象的 thold_ptsum 属性
    p.thold_ptsum = C.float(t)
}

// 设置最大段落长度（以字符为单位）
func (p *Params) SetMaxSegmentLength(n int) {
    // 将传入的整数转换为 C 语言的 int 类型，并设置为参数对象的 max_len 属性
    p.max_len = C.int(n)
}

func (p *Params) SetTokenTimestamps(b bool) {
    // 将布尔值转换为 C 语言的 bool 类型，并设置为参数对象的 token_timestamps 属性
    p.token_timestamps = toBool(b)
}

// 设置每个段落的最大令牌数（0 表示无限制）
func (p *Params) SetMaxTokensPerSegment(n int) {
    // 将传入的整数转换为 C 语言的 int 类型，并设置为参数对象的 max_tokens 属性
    p.max_tokens = C.int(n)
}

// 设置音频编码器上下文
func (p *Params) SetAudioCtx(n int) {
    // 将传入的整数转换为 C 语言的 int 类型，并设置为参数对象的 audio_ctx 属性
    p.audio_ctx = C.int(n)
}

// 设置初始提示
func (p *Params) SetInitialPrompt(prompt string) {
    // 将传入的字符串转换为 C 语言的字符串，并设置为参数对象的 initial_prompt 属性
    p.initial_prompt = C.CString(prompt)
}

///////////////////////////////////////////////////////////////////////////////
// 私有方法

func toBool(v bool) C.bool {
    // 将布尔值转换为 C 语言的 bool 类型
    if v {
        return C.bool(true)
    }
    return C.bool(false)
}

///////////////////////////////////////////////////////////////////////////////
// 字符串化

func (p *Params) String() string {
    // 初始化字符串
    str := "<whisper.params"
    // 格式化输出参数对象的属性
    str += fmt.Sprintf(" strategy=%v", p.strategy)
    str += fmt.Sprintf(" n_threads=%d", p.n_threads)
    if p.language != nil {
        str += fmt.Sprintf(" language=%s", C.GoString(p.language))
    }
    str += fmt.Sprintf(" n_max_text_ctx=%d", p.n_max_text_ctx)
    str += fmt.Sprintf(" offset_ms=%d", p.offset_ms)
    str += fmt.Sprintf(" duration_ms=%d", p.duration_ms)
    str += fmt.Sprintf(" audio_ctx=%d", p.audio_ctx)
    str += fmt.Sprintf(" initial_prompt=%s", C.GoString(p.initial_prompt))
    // 根据属性值是否为真，添加相应的标记到字符串中
    if p.translate {
        str += " translate"
    }
    if p.no_context {
        str += " no_context"
    }
    if p.single_segment {
        str += " single_segment"
    }
    if p.print_special {
        str += " print_special"
    }
    if p.print_progress {
        str += " print_progress"
    }
    if p.print_realtime {
        str += " print_realtime"
    }
    if p.print_timestamps {
        str += " print_timestamps"
}
    # 如果 p 中的 token_timestamps 属性为真，则将字符串 " token_timestamps" 添加到 str 中
    if p.token_timestamps {
        str += " token_timestamps"
    }
    # 如果 p 中的 speed_up 属性为真，则将字符串 " speed_up" 添加到 str 中
    if p.speed_up {
        str += " speed_up"
    }

    # 返回拼接后的字符串，末尾添加字符 ">"
    return str + ">"
# 闭合之前的代码块
```