# `PowerInfer\common\console.h`

```
// 控制台函数

// 防止头文件重复包含
#pragma once

// 包含字符串库
#include <string>

// 控制台命名空间
namespace console {
    // 显示类型枚举
    enum display_t {
        reset = 0,      // 重置显示
        prompt,         // 提示
        user_input,     // 用户输入
        error           // 错误
    };

    // 初始化控制台
    void init(bool use_simple_io, bool use_advanced_display);
    // 清理控制台
    void cleanup();
    // 设置显示类型
    void set_display(display_t display);
    // 读取一行输入
    bool readline(std::string & line, bool multiline_input);
}
```