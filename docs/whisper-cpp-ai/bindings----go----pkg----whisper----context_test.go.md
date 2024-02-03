# `whisper.cpp\bindings\go\pkg\whisper\context_test.go`

```cpp
// 定义包名为 whisper_test
package whisper_test

// 导入必要的包
import (
    "os"
    "testing"

    // 导入第三方包 whisper，用于处理语音识别
    whisper "github.com/ggerganov/whisper.cpp/bindings/go/pkg/whisper"
    // 导入断言包 assert，用于测试断言
    assert "github.com/stretchr/testify/assert"
)

// 定义常量 ModelPath 和 SamplePath，分别为模型文件路径和样本文件路径
const (
    ModelPath  = "../../models/ggml-tiny.bin"
    SamplePath = "../../samples/jfk.wav"
)

// 测试函数 Test_Whisper_000
func Test_Whisper_000(t *testing.T) {
    // 创建断言对象
    assert := assert.New(t)
    // 检查模型文件是否存在，不存在则跳过测试
    if _, err := os.Stat(ModelPath); os.IsNotExist(err) {
        t.Skip("Skipping test, model not found:", ModelPath)
    }
    // 检查样本文件是否存在，不存在则跳过测试
    if _, err := os.Stat(SamplePath); os.IsNotExist(err) {
        t.Skip("Skipping test, sample not found:", SamplePath)
    }

    // 加载模型文件
    model, err := whisper.New(ModelPath)
    assert.NoError(err)
    assert.NotNil(model)
    assert.NoError(model.Close())

    // 打印支持的语言列表
    t.Log("languages=", model.Languages())
}

// 测试函数 Test_Whisper_001
func Test_Whisper_001(t *testing.T) {
    // 创建断言对象
    assert := assert.New(t)
    // 检查模型文件是否存在，不存在则跳过测试
    if _, err := os.Stat(ModelPath); os.IsNotExist(err) {
        t.Skip("Skipping test, model not found:", ModelPath)
    }
    // 检查样本文件是否存在，不存在则跳过测试
    if _, err := os.Stat(SamplePath); os.IsNotExist(err) {
        t.Skip("Skipping test, sample not found:", SamplePath)
    }

    // 加载模型文件
    model, err := whisper.New(ModelPath)
    assert.NoError(err)
    assert.NotNil(model)
    // 延迟关闭模型文件
    defer model.Close()

    // 获取用于解码的上下文
    ctx, err := model.NewContext()
    assert.NoError(err)
    assert.NotNil(ctx)
}
```