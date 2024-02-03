# `whisper.cpp\bindings\go\examples\go-model-download\main.go`

```cpp
// 导入所需的包
import (
    "context"
    "flag"
    "fmt"
    "io"
    "net/http"
    "net/url"
    "os"
    "path/filepath"
    "syscall"
    "time"
)

///////////////////////////////////////////////////////////////////////////////
// 常量

const (
    srcUrl  = "https://huggingface.co/ggerganov/whisper.cpp/resolve/main" // 模型的位置
    srcExt  = ".bin"                                                      // 文件名扩展名
    bufSize = 1024 * 64                                                   // 用于下载模型的缓冲区大小
)

var (
    // 如果没有指定模型作为参数，将下载的模型
    modelNames = []string{"ggml-tiny.en", "ggml-tiny", "ggml-base.en", "ggml-base", "ggml-small.en", "ggml-small", "ggml-medium.en", "ggml-medium", "ggml-large-v1", "ggml-large-v2", "ggml-large-v3"}
)

var (
    // 输出文件夹。如果未设置，则使用当前工作目录。
    flagOut = flag.String("out", "", "输出文件夹")

    // HTTP 超时参数 - 如果下载模型超过此时间将超时
    flagTimeout = flag.Duration("timeout", 30*time.Minute, "HTTP 超时")

    // 安静模式参数 - 如果设置，将不打印进度
    flagQuiet = flag.Bool("quiet", false, "安静模式")
)

///////////////////////////////////////////////////////////////////////////////
// 主函数

func main() {
    // 设置 flag.Usage 函数
    flag.Usage = func() {
        name := filepath.Base(flag.CommandLine.Name())
        fmt.Fprintf(flag.CommandLine.Output(), "Usage: %s [options] <model>\n\n", name)
        flag.PrintDefaults()
    }
    // 解析命令行参数
    flag.Parse()

    // 获取输出路径
    out, err := GetOut()
    if err != nil {
        fmt.Fprintln(os.Stderr, "Error:", err)
        os.Exit(-1)
    }

    // 创建在 SIGINT 或 SIGQUIT 信号时退出的上下文
    ctx := ContextForSignal(os.Interrupt, syscall.SIGQUIT)

    // 进度文件句柄
    progress := os.Stdout
    // 如果 flagQuiet 标志被设置，则将进度输出重定向到 os.DevNull
    if *flagQuiet {
        progress, err = os.Open(os.DevNull)
        // 如果打开文件出错，则输出错误信息并退出程序
        if err != nil {
            fmt.Fprintln(os.Stderr, "Error:", err)
            os.Exit(-1)
        }
        // 延迟关闭进度文件
        defer progress.Close()
    }

    // 下载模型 - 在错误或中断时退出循环
    for _, model := range GetModels() {
        // 获取模型对应的 URL
        url, err := URLForModel(model)
        // 如果获取 URL 出错，则输出错误信息并继续下一个模型
        if err != nil {
            fmt.Fprintln(os.Stderr, "Error:", err)
            continue
        } else if path, err := Download(ctx, progress, url, out); err == nil || err == io.EOF {
            // 如果下载成功或遇到 EOF，则继续下一个模型
            continue
        } else if err == context.Canceled {
            // 如果下载被取消，则删除已下载的文件，输出中断信息，并跳出循环
            os.Remove(path)
            fmt.Fprintln(progress, "\nInterrupted")
            break
        } else if err == context.DeadlineExceeded {
            // 如果下载超时，则删除已下载的文件，输出超时信息，并继续下一个模型
            os.Remove(path)
            fmt.Fprintln(progress, "Timeout downloading model")
            continue
        } else {
            // 如果下载出现其他错误，则删除已下载的文件，输出错误信息，并跳出循环
            os.Remove(path)
            fmt.Fprintln(os.Stderr, "Error:", err)
            break
        }
    }
// 结束当前代码块
}

///////////////////////////////////////////////////////////////////////////////
// 公共方法

// GetOut 返回输出目录的路径
func GetOut() (string, error) {
    // 如果输出目录为空，则返回当前工作目录
    if *flagOut == "" {
        return os.Getwd()
    }
    // 检查输出目录是否存在
    if info, err := os.Stat(*flagOut); err != nil {
        return "", err
    } else if !info.IsDir() {
        return "", fmt.Errorf("not a directory: %s", info.Name())
    } else {
        return *flagOut, nil
    }
}

// GetModels 返回要下载的模型列表
func GetModels() []string {
    // 如果没有指定模型，则返回默认模型列表
    if flag.NArg() == 0 {
        return modelNames
    } else {
        return flag.Args()
    }
}

// URLForModel 返回给定模型在 huggingface.co 上的 URL
func URLForModel(model string) (string, error) {
    // 如果模型的扩展名不是 srcExt，则添加 srcExt
    if filepath.Ext(model) != srcExt {
        model += srcExt
    }
    // 解析 srcUrl 为 URL 对象
    url, err := url.Parse(srcUrl)
    if err != nil {
        return "", err
    } else {
        // 拼接模型路径到 URL
        url.Path = filepath.Join(url.Path, model)
    }
    return url.String(), nil
}

// Download 从给定 URL 下载模型到给定输出目录
func Download(ctx context.Context, p io.Writer, model, out string) (string, error) {
    // 创建 HTTP 客户端
    client := http.Client{
        Timeout: *flagTimeout,
    }

    // 初始化下载请求
    req, err := http.NewRequest("GET", model, nil)
    if err != nil {
        return "", err
    }
    resp, err := client.Do(req)
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()
    if resp.StatusCode != http.StatusOK {
        return "", fmt.Errorf("%s: %s", model, resp.Status)
    }

    // 如果输出文件存在且与模型大小相同，则跳过下载
    path := filepath.Join(out, filepath.Base(model))
    if info, err := os.Stat(path); err == nil && info.Size() == resp.ContentLength {
        fmt.Fprintln(p, "Skipping", model, "as it already exists")
        return "", nil
    }

    // 创建文件
    w, err := os.Create(path)
    // 如果有错误发生，则返回错误信息
    if err != nil {
        return "", err
    }
    // 延迟关闭写入器
    defer w.Close()

    // 输出下载信息
    fmt.Fprintln(p, "Downloading", model, "to", out)

    // 逐步下载模型数据
    data := make([]byte, bufSize)
    count, pct := int64(0), int64(0)
    ticker := time.NewTicker(5 * time.Second)
    for {
        select {
        case <-ctx.Done():
            // 如果取消下载，则返回错误
            return path, ctx.Err()
        case <-ticker.C:
            // 更新下载进度报告
            pct = DownloadReport(p, pct, count, resp.ContentLength)
        default:
            // 读取响应体
            n, err := resp.Body.Read(data)
            if err != nil {
                // 更新下载进度报告并返回错误
                DownloadReport(p, pct, count, resp.ContentLength)
                return path, err
            } else if m, err := w.Write(data[:n]); err != nil {
                // 如果写入出错，则返回错误
                return path, err
            } else {
                count += int64(m)
            }
        }
    }
// DownloadReport函数定期报告下载进度，当百分比变化时
func DownloadReport(w io.Writer, pct, count, total int64) int64 {
    // 计算当前百分比
    pct_ := count * 100 / total
    // 如果当前百分比大于之前的百分比，则输出下载进度信息
    if pct_ > pct {
        fmt.Fprintf(w, "  ...%d MB written (%d%%)\n", count/1e6, pct_)
    }
    // 返回当前百分比
    return pct_
}
```