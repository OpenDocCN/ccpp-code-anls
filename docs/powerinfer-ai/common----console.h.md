# `PowerInfer\common\console.h`

```cpp
// 控制台函数

// 防止头文件重复包含
#pragma once

// 包含字符串库
#include <string>

// 命名空间，包含控制台相关函数
namespace console {
    // 枚举类型，定义显示类型
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