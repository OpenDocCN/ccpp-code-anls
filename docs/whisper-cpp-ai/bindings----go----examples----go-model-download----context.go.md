# `whisper.cpp\bindings\go\examples\go-model-download\context.go`

```cpp
package main

import (
    "context"
    "os"
    "os/signal"
)

// ContextForSignal returns a context object which is cancelled when a signal
// is received. It returns nil if no signal parameter is provided
func ContextForSignal(signals ...os.Signal) context.Context {
    // 如果没有提供信号参数，则返回 nil
    if len(signals) == 0 {
        return nil
    }

    // 创建一个用于接收信号的通道
    ch := make(chan os.Signal)
    // 创建一个带有取消功能的上下文对象
    ctx, cancel := context.WithCancel(context.Background())

    // 当接收到指定信号时，在通道上发送消息
    signal.Notify(ch, signals...)

    // 当接收到任何信号时，调用 cancel 函数取消上下文
    go func() {
        <-ch
        cancel()
    }()

    // 返回成功创建的上下文对象
    return ctx
}
```