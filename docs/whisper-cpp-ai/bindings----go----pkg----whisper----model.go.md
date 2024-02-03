# `whisper.cpp\bindings\go\pkg\whisper\model.go`

```cpp
// 导入必要的包
package whisper

import (
    "fmt"
    "os"
    "runtime"

    // 导入绑定的包
    whisper "github.com/ggerganov/whisper.cpp/bindings/go"
)

///////////////////////////////////////////////////////////////////////////////
// TYPES

// 定义结构体 model
type model struct {
    path string
    ctx  *whisper.Context
}

// 确保 model 符合接口 Model
var _ Model = (*model)(nil)

///////////////////////////////////////////////////////////////////////////////
// LIFECYCLE

// 创建新的 model 实例
func New(path string) (Model, error) {
    model := new(model)
    // 检查指定路径是否存在
    if _, err := os.Stat(path); err != nil {
        return nil, err
    } else if ctx := whisper.Whisper_init(path); ctx == nil {
        return nil, ErrUnableToLoadModel
    } else {
        model.ctx = ctx
        model.path = path
    }

    // 返回成功
    return model, nil
}

// 关闭 model 实例
func (model *model) Close() error {
    if model.ctx != nil {
        model.ctx.Whisper_free()
    }

    // 释放资源
    model.ctx = nil

    // 返回成功
    return nil
}

///////////////////////////////////////////////////////////////////////////////
// STRINGIFY

// 将 model 结构体转换为字符串
func (model *model) String() string {
    str := "<whisper.model"
    if model.ctx != nil {
        str += fmt.Sprintf(" model=%q", model.path)
    }
    return str + ">"
}

///////////////////////////////////////////////////////////////////////////////
// PUBLIC METHODS

// 返回 true 如果 model 支持多语言（支持语言和翻译选项）
func (model *model) IsMultilingual() bool {
    return model.ctx.Whisper_is_multilingual() != 0
}

// 返回所有识别的语言。初始设置为自动检测
func (model *model) Languages() []string {
    result := make([]string, 0, whisper.Whisper_lang_max_id())
    for i := 0; i < whisper.Whisper_lang_max_id(); i++ {
        str := whisper.Whisper_lang_str(i)
        if model.ctx.Whisper_lang_id(str) >= 0 {
            result = append(result, str)
        }
    }
    return result
}
// 创建新的上下文
func (model *model) NewContext() (Context, error) {
    // 如果模型的上下文为空，则返回错误
    if model.ctx == nil {
        return nil, ErrInternalAppError
    }

    // 创建新的参数对象，使用默认参数
    params := model.ctx.Whisper_full_default_params(whisper.SAMPLING_GREEDY)
    // 设置参数不进行翻译
    params.SetTranslate(false)
    // 设置参数不打印特殊信息
    params.SetPrintSpecial(false)
    // 设置参数不打印进度信息
    params.SetPrintProgress(false)
    // 设置参数不实时打印信息
    params.SetPrintRealtime(false)
    // 设置参数不打印时间戳
    params.SetPrintTimestamps(false)
    // 设置参数的线程数为当前 CPU 核心数
    params.SetThreads(runtime.NumCPU())
    // 设置参数不包含上下文信息
    params.SetNoContext(true)

    // 返回新的上下文对象
    return newContext(model, params)
}
```