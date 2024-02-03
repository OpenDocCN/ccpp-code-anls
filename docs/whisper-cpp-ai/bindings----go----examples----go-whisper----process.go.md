# `whisper.cpp\bindings\go\examples\go-whisper\process.go`

```cpp
package main


import (
    "fmt"
    "io"
    "os"
    "time"

    // Package imports
    whisper "github.com/ggerganov/whisper.cpp/bindings/go/pkg/whisper"
    wav "github.com/go-audio/wav"
)


func Process(model whisper.Model, path string, flags *Flags) error {


    var data []float32


    // Create processing context
    context, err := model.NewContext()
    if err != nil {
        return err
    }


    // Set the parameters
    if err := flags.SetParams(context); err != nil {
        return err
    }


    fmt.Printf("\n%s\n", context.SystemInfo())


    // Open the file
    fmt.Fprintf(flags.Output(), "Loading %q\n", path)
    fh, err := os.Open(path)
    if err != nil {
        return err
    }
    defer fh.Close()


    // Decode the WAV file - load the full buffer
    dec := wav.NewDecoder(fh)
    if buf, err := dec.FullPCMBuffer(); err != nil {
        return err
    } else if dec.SampleRate != whisper.SampleRate {
        return fmt.Errorf("unsupported sample rate: %d", dec.SampleRate)
    } else if dec.NumChans != 1 {
        return fmt.Errorf("unsupported number of channels: %d", dec.NumChans)
    } else {
        data = buf.AsFloat32Buffer().Data
    }


    // Segment callback when -tokens is specified
    var cb whisper.SegmentCallback
    if flags.IsTokens() {
        cb = func(segment whisper.Segment) {
            fmt.Fprintf(flags.Output(), "%02d [%6s->%6s] ", segment.Num, segment.Start.Truncate(time.Millisecond), segment.End.Truncate(time.Millisecond))
            for _, token := range segment.Tokens {
                if flags.IsColorize() && context.IsText(token) {
                    fmt.Fprint(flags.Output(), Colorize(token.Text, int(token.P*24.0)), " ")
                } else {
                    fmt.Fprint(flags.Output(), token.Text, " ")
                }
            }
            fmt.Fprintln(flags.Output(), "")
            fmt.Fprintln(flags.Output(), "")
        }
    }


    // Process the data
    // 使用 fmt.Fprintf 格式化输出信息到标准输出，显示正在处理的文件路径
    fmt.Fprintf(flags.Output(), "  ...processing %q\n", path)
    // 重置计时器
    context.ResetTimings()
    // 调用 context.Process 处理数据，传入数据、回调函数和空指针，如果出错则返回错误
    if err := context.Process(data, cb, nil); err != nil {
        return err
    }

    // 打印处理时间
    context.PrintTimings()

    // 根据输出格式进行不同的处理
    switch {
    case flags.GetOut() == "srt":
        // 如果输出格式为 srt，则调用 OutputSRT 输出到标准输出，并返回结果
        return OutputSRT(os.Stdout, context)
    case flags.GetOut() == "none":
        // 如果输出格式为 none，则返回空
        return nil
    default:
        // 其他情况下，调用 Output 输出到标准输出，根据是否需要着色
        return Output(os.Stdout, context, flags.IsColorize())
    }
// 输出文本作为 SRT 文件
func OutputSRT(w io.Writer, context whisper.Context) error {
    n := 1
    for {
        // 获取下一个片段
        segment, err := context.NextSegment()
        // 如果到达文件末尾，返回 nil
        if err == io.EOF {
            return nil
        } else if err != nil {
            return err
        }
        // 输出片段编号
        fmt.Fprintln(w, n)
        // 输出片段的时间戳
        fmt.Fprintln(w, srtTimestamp(segment.Start), " --> ", srtTimestamp(segment.End))
        // 输出片段文本
        fmt.Fprintln(w, segment.Text)
        // 输出空行
        fmt.Fprintln(w, "")
        n++
    }
}

// 输出文本到终端
func Output(w io.Writer, context whisper.Context, colorize bool) error {
    for {
        // 获取下一个片段
        segment, err := context.NextSegment()
        // 如果到达文件末尾，返回 nil
        if err == io.EOF {
            return nil
        } else if err != nil {
            return err
        }
        // 输出片段的时间戳
        fmt.Fprintf(w, "[%6s->%6s]", segment.Start.Truncate(time.Millisecond), segment.End.Truncate(time.Millisecond))
        // 如果需要着色
        if colorize {
            // 遍历片段的标记
            for _, token := range segment.Tokens {
                // 如果不是文本标记，跳过
                if !context.IsText(token) {
                    continue
                }
                // 输出着色后的文本
                fmt.Fprint(w, " ", Colorize(token.Text, int(token.P*24.0)))
            }
            // 输出换行
            fmt.Fprint(w, "\n")
        } else {
            // 输出未着色的文本
            fmt.Fprintln(w, " ", segment.Text)
        }
    }
}

// 返回 SRT 时间戳
func srtTimestamp(t time.Duration) string {
    return fmt.Sprintf("%02d:%02d:%02d,%03d", t/time.Hour, (t%time.Hour)/time.Minute, (t%time.Minute)/time.Second, (t%time.Second)/time.Millisecond)
}
```