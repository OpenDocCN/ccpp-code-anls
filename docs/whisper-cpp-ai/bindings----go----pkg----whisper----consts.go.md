# `whisper.cpp\bindings\go\pkg\whisper\consts.go`

```cpp
// 导入 whisper 包
import (
    "errors"

    // 导入 whisper.cpp/bindings/go 包，提供对应的绑定
    whisper "github.com/ggerganov/whisper.cpp/bindings/go"
)

///////////////////////////////////////////////////////////////////////////////
// ERRORS

// 定义错误变量
var (
    // 无法加载模型的错误
    ErrUnableToLoadModel = errors.New("unable to load model")
    // 内部应用程序错误
    ErrInternalAppError = errors.New("internal application error")
    // 处理失败的错误
    ErrProcessingFailed = errors.New("processing failed")
    // 不支持的语言错误
    ErrUnsupportedLanguage = errors.New("unsupported language")
    // 模型不是多语言的错误
    ErrModelNotMultilingual = errors.New("model is not multilingual")
)

///////////////////////////////////////////////////////////////////////////////
// CONSTANTS

// SampleRate 是音频数据的采样率
const SampleRate = whisper.SampleRate

// SampleBits 是每个样本的字节数
const SampleBits = whisper.SampleBits
```