# `whisper.cpp\bindings\go\whisper_test.go`

```cpp
// 定义包名为 whisper_test
package whisper_test

// 导入所需的包
import (
    "os"
    "runtime"
    "testing"
    "time"

    // 导入第三方包 whisper
    whisper "github.com/ggerganov/whisper.cpp/bindings/go"
    // 导入第三方包 wav
    wav "github.com/go-audio/wav"
    // 导入第三方包 assert
    assert "github.com/stretchr/testify/assert"
)

// 定义常量 ModelPath 和 SamplePath
const (
    ModelPath  = "models/ggml-small.en.bin"
    SamplePath = "samples/jfk.wav"
)

// 测试函数 Test_Whisper_000
func Test_Whisper_000(t *testing.T) {
    // 创建断言对象
    assert := assert.New(t)
    // 检查模型文件是否存在，如果不存在则跳过测试
    if _, err := os.Stat(ModelPath); os.IsNotExist(err) {
        t.Skip("Skipping test, model not found:", ModelPath)
    }
    // 初始化 Whisper 上下文
    ctx := whisper.Whisper_init(ModelPath)
    assert.NotNil(ctx)
    // 释放 Whisper 上下文
    ctx.Whisper_free()
}

// 测试函数 Test_Whisper_001
func Test_Whisper_001(t *testing.T) {
    // 创建断言对象
    assert := assert.New(t)
    // 检查模型文件是否存在，如果不存在则跳过测试
    if _, err := os.Stat(ModelPath); os.IsNotExist(err) {
        t.Skip("Skipping test, model not found:", ModelPath)
    }
    // 检查样本文件是否存在，如果不存在则跳过测试
    if _, err := os.Stat(SamplePath); os.IsNotExist(err) {
        t.Skip("Skipping test, sample not found:", SamplePath)
    }

    // 打开样本文件
    fh, err := os.Open(SamplePath)
    assert.NoError(err)
    defer fh.Close()

    // 读取样本文件
    d := wav.NewDecoder(fh)
    buf, err := d.FullPCMBuffer()
    assert.NoError(err)

    // 初始化 Whisper 上下文
    ctx := whisper.Whisper_init(ModelPath)
    assert.NotNil(ctx)
    defer ctx.Whisper_free()
    // 获取默认参数
    params := ctx.Whisper_full_default_params(whisper.SAMPLING_GREEDY)
    data := buf.AsFloat32Buffer().Data
    // 运行 Whisper
    err = ctx.Whisper_full(params, data, nil, nil, nil)
    assert.NoError(err)

    // 打印 tokens
    num_segments := ctx.Whisper_full_n_segments()
    assert.GreaterOrEqual(num_segments, 1)
    for i := 0; i < num_segments; i++ {
        str := ctx.Whisper_full_get_segment_text(i)
        assert.NotEmpty(str)
        t0 := time.Duration(ctx.Whisper_full_get_segment_t0(i)) * time.Millisecond
        t1 := time.Duration(ctx.Whisper_full_get_segment_t1(i)) * time.Millisecond
        t.Logf("[%6s->%-6s] %q", t0, t1, str)
    }
}

// 测试函数 Test_Whisper_002
func Test_Whisper_002(t *testing.T) {
    // 创建断言对象
    assert := assert.New(t)
    // 循环遍历从0到Whisper_lang_max_id()的所有整数
    for i := 0; i < whisper.Whisper_lang_max_id(); i++ {
        // 获取对应于i的Whisper语言字符串
        str := whisper.Whisper_lang_str(i)
        // 断言字符串不为空
        assert.NotEmpty(str)
        // 输出字符串到日志
        t.Log(str)
    }
// 测试函数 Test_Whisper_003，用于测试 Whisper 库的功能
func Test_Whisper_003(t *testing.T) {
    // 获取当前系统的 CPU 核心数
    threads := runtime.NumCPU()
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

    // 打开样本文件
    fh, err := os.Open(SamplePath)
    assert.NoError(err)
    defer fh.Close()

    // 读取样本文件内容
    d := wav.NewDecoder(fh)
    buf, err := d.FullPCMBuffer()
    assert.NoError(err)

    // 创建 Whisper 模型对象
    ctx := whisper.Whisper_init(ModelPath)
    assert.NotNil(ctx)
    defer ctx.Whisper_free()

    // 将 PCM 数据转换为 MEL 频谱
    assert.NoError(ctx.Whisper_pcm_to_mel(buf.AsFloat32Buffer().Data, threads))

    // 自动检测语言
    languages, err := ctx.Whisper_lang_auto_detect(0, threads)
    assert.NoError(err)
    // 遍历语言检测结果并输出
    for i, p := range languages {
        t.Logf("%s: %f", whisper.Whisper_lang_str(i), p)
    }
}
```