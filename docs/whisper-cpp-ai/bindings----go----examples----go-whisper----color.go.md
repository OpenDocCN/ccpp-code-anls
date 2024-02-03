# `whisper.cpp\bindings\go\examples\go-whisper\color.go`

```cpp
// 主程序包
package main

// 导入 fmt 包
import "fmt"

///////////////////////////////////////////////////////////////////////////////
// 常量

// ANSI 控制码，用于重置文本颜色
const (
    Reset     = "\033[0m"
    // RGB 颜色前缀，后面跟着用冒号分隔的十进制 RGB 值
    RGBPrefix = "\033[38;5;"
    // RGB 颜色后缀
    RGBSuffix = "m"
)

///////////////////////////////////////////////////////////////////////////////
// 公共方法

// 使用 RGB 值（0-23）对文本进行着色
func Colorize(text string, v int) string {
    // 参考链接：https://en.wikipedia.org/wiki/ANSI_escape_code#8-bit
    // 灰度颜色范围在 232-255
    return RGBPrefix + fmt.Sprint(v%24+232) + RGBSuffix + text + Reset
}
```