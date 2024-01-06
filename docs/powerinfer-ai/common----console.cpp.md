# `PowerInfer\common\console.cpp`

```
// 包含头文件 console.h
#include "console.h"
// 包含 vector 和 iostream 头文件
#include <vector>
#include <iostream>

// 如果是 Windows 系统
#if defined(_WIN32)
// 定义 WIN32_LEAN_AND_MEAN
#define WIN32_LEAN_AND_MEAN
// 如果未定义 NOMINMAX，则定义 NOMINMAX
#ifndef NOMINMAX
#define NOMINMAX
#endif
// 包含 windows.h 头文件
#include <windows.h>
// 包含 fcntl.h 和 io.h 头文件
#include <fcntl.h>
#include <io.h>
// 如果未定义 ENABLE_VIRTUAL_TERMINAL_PROCESSING，则定义 ENABLE_VIRTUAL_TERMINAL_PROCESSING 为 0x0004
#ifndef ENABLE_VIRTUAL_TERMINAL_PROCESSING
#define ENABLE_VIRTUAL_TERMINAL_PROCESSING 0x0004
#endif
// 如果不是 Windows 系统
#else
// 包含 limits.h、sys/ioctl.h、unistd.h 和 wchar.h 头文件
#include <climits>
#include <sys/ioctl.h>
#include <unistd.h>
#include <wchar.h>
#include <stdio.h>  // 包含标准输入输出库
#include <stdlib.h> // 包含标准库
#include <signal.h> // 包含信号处理库
#include <termios.h> // 包含终端 I/O 库
#endif // 结束条件编译指令

#define ANSI_COLOR_RED     "\x1b[31m" // 定义红色 ANSI 转义序列
#define ANSI_COLOR_GREEN   "\x1b[32m" // 定义绿色 ANSI 转义序列
#define ANSI_COLOR_YELLOW  "\x1b[33m" // 定义黄色 ANSI 转义序列
#define ANSI_COLOR_BLUE    "\x1b[34m" // 定义蓝色 ANSI 转义序列
#define ANSI_COLOR_MAGENTA "\x1b[35m" // 定义洋红色 ANSI 转义序列
#define ANSI_COLOR_CYAN    "\x1b[36m" // 定义青色 ANSI 转义序列
#define ANSI_COLOR_RESET   "\x1b[0m"  // 定义重置 ANSI 转义序列
#define ANSI_BOLD          "\x1b[1m"   // 定义加粗 ANSI 转义序列

namespace console {

    //
    // Console state
    //
```

注释部分为对代码中每个语句的解释和作用。
    // 定义静态布尔变量，用于控制是否使用高级显示和简单输入输出
    static bool      advanced_display = false;
    static bool      simple_io        = true;
    // 定义静态枚举变量，用于表示当前显示状态
    static display_t current_display  = reset;

    // 定义静态文件指针，用于输出
    static FILE*     out              = stdout;

    // 如果是在 Windows 平台下编译，定义静态指针变量 hConsole
#if defined (_WIN32)
    static void*     hConsole;
    // 否则，在非 Windows 平台下，定义静态文件指针变量 tty 和 termios 结构变量 initial_state
#else
    static FILE*     tty              = nullptr;
    static termios   initial_state;
#endif

    //
    // 初始化和清理
    //

    // 初始化函数，接受两个布尔参数，用于设置是否使用简单输入输出和高级显示
    void init(bool use_simple_io, bool use_advanced_display) {
        // 将参数值分别赋给对应的静态变量
        advanced_display = use_advanced_display;
        // 根据平台定义是否使用简单的输入输出
        simple_io = use_simple_io;
#if defined(_WIN32)
        // Windows特定的控制台初始化
        DWORD dwMode = 0;
        hConsole = GetStdHandle(STD_OUTPUT_HANDLE);
        // 检查标准输出句柄是否有效，获取控制台模式
        if (hConsole == INVALID_HANDLE_VALUE || !GetConsoleMode(hConsole, &dwMode)) {
            // 如果标准输出句柄无效，尝试获取标准错误句柄并获取控制台模式
            hConsole = GetStdHandle(STD_ERROR_HANDLE);
            if (hConsole != INVALID_HANDLE_VALUE && (!GetConsoleMode(hConsole, &dwMode))) {
                // 如果标准错误句柄有效但无法获取控制台模式，则将控制台句柄置为空，并使用简单的输入输出
                hConsole = nullptr;
                simple_io = true;
            }
        }
        if (hConsole) {
            // 组合条件以减少嵌套
            if (advanced_display && !(dwMode & ENABLE_VIRTUAL_TERMINAL_PROCESSING) &&
                !SetConsoleMode(hConsole, dwMode | ENABLE_VIRTUAL_TERMINAL_PROCESSING)) {
                // 如果高级显示开启但控制台不支持虚拟终端处理，或者设置控制台模式失败，则关闭高级显示
                advanced_display = false;
            }
            // 设置控制台输出代码页为UTF8
            SetConsoleOutputCP(CP_UTF8);
        }
        // 获取标准输入句柄
        HANDLE hConIn = GetStdHandle(STD_INPUT_HANDLE);
        // 如果获取成功并且能够获取控制台模式
        if (hConIn != INVALID_HANDLE_VALUE && GetConsoleMode(hConIn, &dwMode)) {
            // 设置控制台输入代码页为UTF16
            _setmode(_fileno(stdin), _O_WTEXT);

            // 设置ICANON（ENABLE_LINE_INPUT）和ECHO（ENABLE_ECHO_INPUT）
            if (simple_io) {
                dwMode |= ENABLE_LINE_INPUT | ENABLE_ECHO_INPUT;
            } else {
                dwMode &= ~(ENABLE_LINE_INPUT | ENABLE_ECHO_INPUT);
            }
            // 如果无法设置控制台模式，则将simple_io设置为true
            if (!SetConsoleMode(hConIn, dwMode)) {
                simple_io = true;
            }
        }
        // 否则，执行POSIX特定的控制台初始化
        else {
            if (!simple_io) {
                struct termios new_termios;
// 获取标准输入的终端属性，并保存到initial_state中
tcgetattr(STDIN_FILENO, &initial_state);
// 将initial_state的值赋给new_termios
new_termios = initial_state;
// 关闭ICANON和ECHO标志位
new_termios.c_lflag &= ~(ICANON | ECHO);
// 设置最小输入字符数为1
new_termios.c_cc[VMIN] = 1;
// 设置读取输入的超时时间为0
new_termios.c_cc[VTIME] = 0;
// 设置标准输入的终端属性为new_termios
tcsetattr(STDIN_FILENO, TCSANOW, &new_termios);

// 打开/dev/tty文件，以可读写方式
tty = fopen("/dev/tty", "w+");
// 如果打开成功，则将out指向tty
if (tty != nullptr) {
    out = tty;
}

// 设置本地化环境为默认
setlocale(LC_ALL, "");
// 重置控制台显示
set_display(reset);
#if !defined(_WIN32)
        // 如果不是在 Windows 系统上，恢复 POSIX 系统的设置
        if (!simple_io) {
            // 如果不是简单的输入输出，恢复标准输出和终端设置
            if (tty != nullptr) {
                out = stdout;
                fclose(tty);
                tty = nullptr;
            }
            tcsetattr(STDIN_FILENO, TCSANOW, &initial_state);
        }
#endif
    }

    //
    // Display and IO
    //

    // 跟踪当前显示，并且只在显示改变时发出 ANSI 代码
    void set_display(display_t display) {
# 如果高级显示开启并且当前显示模式不等于要切换的显示模式
if (advanced_display && current_display != display) {
    # 清空标准输出缓冲区
    fflush(stdout);
    # 根据显示模式进行切换
    switch(display) {
        # 如果是重置显示模式
        case reset:
            # 输出 ANSI 控制码来重置显示样式
            fprintf(out, ANSI_COLOR_RESET);
            break;
        # 如果是提示显示模式
        case prompt:
            # 输出 ANSI 控制码来设置显示为黄色
            fprintf(out, ANSI_COLOR_YELLOW);
            break;
        # 如果是用户输入显示模式
        case user_input:
            # 输出 ANSI 控制码来设置显示为加粗的绿色
            fprintf(out, ANSI_BOLD ANSI_COLOR_GREEN);
            break;
        # 如果是错误显示模式
        case error:
            # 输出 ANSI 控制码来设置显示为加粗的红色
            fprintf(out, ANSI_BOLD ANSI_COLOR_RED);
    }
    # 更新当前显示模式
    current_display = display;
    # 清空输出缓冲区
    fflush(out);
}
    // 定义一个返回32位字符的函数
    static char32_t getchar32() {
        // 如果是 Windows 系统
#if defined(_WIN32)
        // 获取标准输入句柄
        HANDLE hConsole = GetStdHandle(STD_INPUT_HANDLE);
        // 初始化高代理项
        wchar_t high_surrogate = 0;

        // 循环读取输入
        while (true) {
            // 定义输入记录和计数
            INPUT_RECORD record;
            DWORD count;
            // 读取控制台输入
            if (!ReadConsoleInputW(hConsole, &record, 1, &count) || count == 0) {
                return WEOF;
            }

            // 如果是键盘事件并且按键按下
            if (record.EventType == KEY_EVENT && record.Event.KeyEvent.bKeyDown) {
                // 获取 Unicode 字符
                wchar_t wc = record.Event.KeyEvent.uChar.UnicodeChar;
                // 如果是控制字符，继续循环
                if (wc == 0) {
                    continue;
                }

                // 如果是高代理项
                if ((wc >= 0xD800) && (wc <= 0xDBFF)) { // Check if wc is a high surrogate
                    // 存储高代理项
                    high_surrogate = wc;
                continue; // 跳过当前循环，继续下一次循环
                }
                if ((wc >= 0xDC00) && (wc <= 0xDFFF)) { // 检查 wc 是否为低代理项
                    if (high_surrogate != 0) { // 检查是否有高代理项
                        return ((high_surrogate - 0xD800) << 10) + (wc - 0xDC00) + 0x10000; // 计算并返回补充字符
                    }
                }

                high_surrogate = 0; // 重置高代理项
                return static_cast<char32_t>(wc); // 返回 Unicode 字符
            }
        }
#else
        wchar_t wc = getwchar(); // 从标准输入获取宽字符
        if (static_cast<wint_t>(wc) == WEOF) { // 检查是否到达文件结尾
            return WEOF; // 返回文件结尾标识
        }

#if WCHAR_MAX == 0xFFFF
        if ((wc >= 0xD800) && (wc <= 0xDBFF)) { // 检查 wc 是否为高代理项
            // 读取下一个宽字符
            wchar_t low_surrogate = getwchar();
            // 检查下一个宽字符是否为低代理项
            if ((low_surrogate >= 0xDC00) && (low_surrogate <= 0xDFFF)) { 
                // 如果是低代理项，则计算代理项对的 Unicode 码点
                return (static_cast<char32_t>(wc & 0x03FF) << 10) + (low_surrogate & 0x03FF) + 0x10000;
            }
        }
        // 如果宽字符为无效的代理项
        if ((wc >= 0xD800) && (wc <= 0xDFFF)) { 
            // 返回替换字符 U+FFFD
            return 0xFFFD; 
        }
#endif

        // 返回宽字符的 Unicode 码点
        return static_cast<char32_t>(wc);
#endif
    }

    // 弹出光标
    static void pop_cursor() {
#if defined(_WIN32)
        // 如果控制台句柄不为空
        if (hConsole != NULL) {
            // 获取控制台屏幕缓冲区信息
            CONSOLE_SCREEN_BUFFER_INFO bufferInfo;
            GetConsoleScreenBufferInfo(hConsole, &bufferInfo);
# 定义一个新的光标位置变量，并将其初始化为当前光标位置
COORD newCursorPosition = bufferInfo.dwCursorPosition;
# 如果新的光标位置的 X 坐标为 0，则将 X 坐标设置为屏幕宽度减一，Y 坐标减一；否则，将 X 坐标减一
if (newCursorPosition.X == 0) {
    newCursorPosition.X = bufferInfo.dwSize.X - 1;
    newCursorPosition.Y -= 1;
} else {
    newCursorPosition.X -= 1;
}

# 设置控制台光标位置为新的光标位置
SetConsoleCursorPosition(hConsole, newCursorPosition);
# 返回
return;
# 如果是在 Windows 平台下，返回字符宽度为 1；否则，返回字符宽度为 0
static int estimateWidth(char32_t codepoint) {
#if defined(_WIN32)
    (void)codepoint;
    return 1;
#else
        // 返回给定代码点的宽度
        return wcwidth(codepoint);
#endif
    }

    // 将 UTF-8 编码的代码点写入控制台，并返回实际宽度
    static int put_codepoint(const char* utf8_codepoint, size_t length, int expectedWidth) {
#if defined(_WIN32)
        // 获取控制台屏幕缓冲区信息
        CONSOLE_SCREEN_BUFFER_INFO bufferInfo;
        // 如果获取失败，返回预期宽度
        if (!GetConsoleScreenBufferInfo(hConsole, &bufferInfo)) {
            // go with the default
            return expectedWidth;
        }
        // 获取初始光标位置
        COORD initialPosition = bufferInfo.dwCursorPosition;
        DWORD nNumberOfChars = length;
        // 写入 UTF-8 编码的代码点到控制台
        WriteConsole(hConsole, utf8_codepoint, nNumberOfChars, &nNumberOfChars, NULL);

        // 获取新的控制台屏幕缓冲区信息
        CONSOLE_SCREEN_BUFFER_INFO newBufferInfo;
        GetConsoleScreenBufferInfo(hConsole, &newBufferInfo);

        // 如果在最后一列，计算实际位置
        if (utf8_codepoint[0] != 0x09 && initialPosition.X == newBufferInfo.dwSize.X - 1) {
            // 声明一个变量用于存储写入字符的数量
            DWORD nNumberOfChars;
            // 在控制台上写入一个空格，相当于清除最后一个字符
            WriteConsole(hConsole, &" \b", 2, &nNumberOfChars, NULL);
            // 获取控制台屏幕缓冲区的信息
            GetConsoleScreenBufferInfo(hConsole, &newBufferInfo);
        }

        // 计算控制台窗口的宽度
        int width = newBufferInfo.dwCursorPosition.X - initialPosition.X;
        // 如果宽度为负数，则加上控制台窗口的总宽度
        if (width < 0) {
            width += newBufferInfo.dwSize.X;
        }
        // 返回计算得到的宽度
        return width;
#else
        // 如果有预期的宽度值或者没有控制台，则直接写入字符并返回预期宽度
        if (expectedWidth >= 0 || tty == nullptr) {
            fwrite(utf8_codepoint, length, 1, out);
            return expectedWidth;
        }
        // 向控制台发送查询光标位置的指令
        fputs("\033[6n", tty); // Query cursor position
        // 声明变量用于存储光标位置的坐标
        int x1;
        int y1;
        // 声明变量 x2 和 y2
        int x2;
        int y2;
        // 声明并初始化变量 results 为 0
        int results = 0;
        // 从 tty 中读取数据，将结果存入 results 中
        results = fscanf(tty, "\033[%d;%dR", &y1, &x1);

        // 将 utf8_codepoint 中的内容写入 tty
        fwrite(utf8_codepoint, length, 1, tty);

        // 向 tty 发送查询光标位置的指令
        fputs("\033[6n", tty); // Query cursor position
        // 从 tty 中读取数据，将结果存入 results 中
        results += fscanf(tty, "\033[%d;%dR", &y2, &x2);

        // 如果结果不符合预期，返回 expectedWidth
        if (results != 4) {
            return expectedWidth;
        }

        // 计算宽度
        int width = x2 - x1;
        // 如果宽度小于 0，考虑文本换行后重新计算宽度
        if (width < 0) {
            // 获取终端窗口大小
            struct winsize w;
            ioctl(STDOUT_FILENO, TIOCGWINSZ, &w);
            // 考虑文本换行后重新计算宽度
            width += w.ws_col;
    }
    return width;
#endif
}

// 替换最后一个字符
static void replace_last(char ch) {
#if defined(_WIN32)
    // 弹出光标
    pop_cursor();
    // 输出字符
    put_codepoint(&ch, 1, 1);
#else
    // 在标准输出中移动光标到前一个位置，然后输出字符
    fprintf(out, "\b%c", ch);
#endif
}

// 追加 UTF-8 字符到字符串
static void append_utf8(char32_t ch, std::string & out) {
    if (ch <= 0x7F) {
        // 如果字符编码在 0x7F 以内，直接追加到字符串中
        out.push_back(static_cast<unsigned char>(ch));
    } else if (ch <= 0x7FF) {
        // 如果字符编码在 0x7FF 以内，按照 UTF-8 编码规则追加到字符串中
        out.push_back(static_cast<unsigned char>(0xC0 | ((ch >> 6) & 0x1F)));
        out.push_back(static_cast<unsigned char>(0x80 | (ch & 0x3F)));
        } else if (ch <= 0xFFFF) {
            // 如果 Unicode 编码小于等于 0xFFFF，则使用 3 个字节表示
            out.push_back(static_cast<unsigned char>(0xE0 | ((ch >> 12) & 0x0F)));
            out.push_back(static_cast<unsigned char>(0x80 | ((ch >> 6) & 0x3F)));
            out.push_back(static_cast<unsigned char>(0x80 | (ch & 0x3F)));
        } else if (ch <= 0x10FFFF) {
            // 如果 Unicode 编码小于等于 0x10FFFF，则使用 4 个字节表示
            out.push_back(static_cast<unsigned char>(0xF0 | ((ch >> 18) & 0x07)));
            out.push_back(static_cast<unsigned char>(0x80 | ((ch >> 12) & 0x3F)));
            out.push_back(static_cast<unsigned char>(0x80 | ((ch >> 6) & 0x3F)));
            out.push_back(static_cast<unsigned char>(0x80 | (ch & 0x3F)));
        } else {
            // 无效的 Unicode 编码点
            // 如果 Unicode 编码大于 0x10FFFF，则为无效编码点
        }
    }

    // 从字符串中移除最后一个 UTF-8 字符的辅助函数
    static void pop_back_utf8_char(std::string & line) {
        // 如果字符串为空，则直接返回
        if (line.empty()) {
            return;
        }
        // 获取字符串的长度
        size_t pos = line.length() - 1;

        // 寻找最后一个UTF-8字符的起始位置（向前检查最多4个字节）
        for (size_t i = 0; i < 3 && pos > 0; ++i, --pos) {
            if ((line[pos] & 0xC0) != 0x80) {
                break; // 找到字符的起始位置
            }
        }
        // 删除最后一个字符及其后面的内容
        line.erase(pos);
    }

    // 高级读取一行函数
    static bool readline_advanced(std::string & line, bool multiline_input) {
        // 如果输出不是标准输出，刷新标准输出
        if (out != stdout) {
            fflush(stdout);
        }

        // 清空字符串
        line.clear();
        // 存储字符宽度的向量
        std::vector<int> widths;
        // 是否为特殊字符
        bool is_special_char = false;
        // 是否到达流的末尾
        bool end_of_stream = false;
// 声明一个 char32_t 类型的变量 input_char
char32_t input_char;
// 进入无限循环
while (true) {
    // 刷新输出缓冲区，确保所有输出在等待输入之前都已显示
    fflush(out);
    // 从标准输入获取一个字符
    input_char = getchar32();

    // 如果输入是回车或换行符，跳出循环
    if (input_char == '\r' || input_char == '\n') {
        break;
    }

    // 如果输入是文件结束符或 Ctrl+D，设置流结束标志并跳出循环
    if (input_char == (char32_t) WEOF || input_char == 0x04 /* Ctrl+D*/) {
        end_of_stream = true;
        break;
    }

    // 如果输入是特殊字符
    if (is_special_char) {
        // 设置显示用户输入
        set_display(user_input);
        // 替换最后一个字符
        replace_last(line.back());
        // 重置特殊字符标志
        is_special_char = false;
    }
}
            if (input_char == '\033') { // 如果输入字符是转义字符
                char32_t code = getchar32(); // 获取下一个字符
                if (code == '[' || code == 0x1B) { // 如果是 '[' 或者 0x1B
                    // 丢弃剩余的转义序列
                    while ((code = getchar32()) != (char32_t) WEOF) { // 循环读取下一个字符，直到遇到文件结束符
                        if ((code >= 'A' && code <= 'Z') || (code >= 'a' && code <= 'z') || code == '~') { // 如果是大写字母、小写字母或者波浪号
                            break; // 跳出循环
                        }
                    }
                }
            } else if (input_char == 0x08 || input_char == 0x7F) { // 如果输入字符是退格键或者删除键
                if (!widths.empty()) { // 如果宽度不为空
                    int count;
                    do {
                        count = widths.back(); // 获取最后一个宽度
                        widths.pop_back(); // 移除最后一个宽度
                        // 将光标移回，打印空格，再次将光标移回
                        for (int i = 0; i < count; i++) { // 循环打印空格
                            replace_last(' '); // 替换最后一个字符为' '
                pop_cursor();
            } // 弹出光标位置
            pop_back_utf8_char(line); // 弹出最后一个 UTF-8 字符
        } while (count == 0 && !widths.empty()); // 当计数为 0 且宽度不为空时循环

    } else {
        int offset = line.length(); // 获取行长度
        append_utf8(input_char, line); // 在行末添加 UTF-8 字符
        int width = put_codepoint(line.c_str() + offset, line.length() - offset, estimateWidth(input_char)); // 计算字符宽度
        if (width < 0) { // 如果宽度小于 0
            width = 0; // 将宽度设为 0
        }
        widths.push_back(width); // 将宽度添加到宽度列表中
    }

    if (!line.empty() && (line.back() == '\\' || line.back() == '/')) { // 如果行不为空且最后一个字符是反斜杠或斜杠
        set_display(prompt); // 设置显示提示
        replace_last(line.back()); // 替换最后一个字符
        is_special_char = true; // 设置特殊字符标志为真
    }
        }

        // 检查是否有更多的多行输入
        bool has_more = multiline_input;
        // 如果是特殊字符
        if (is_special_char) {
            // 替换最后一个字符为' '
            replace_last(' ');
            // 弹出光标
            pop_cursor();

            // 获取最后一个字符
            char last = line.back();
            // 移除最后一个字符
            line.pop_back();
            // 如果最后一个字符是'\\'
            if (last == '\\') {
                // 在行末添加换行符
                line += '\n';
                // 将换行符写入输出
                fputc('\n', out);
                // 更新是否有更多多行输入的标志
                has_more = !has_more;
            } else {
                // 如果行长度为1且最后一个字符是空格
                if (line.length() == 1 && line.back() == ' ') {
                    // 清空行
                    line.clear();
                    // 弹出光标
                    pop_cursor();
                }
                // 更新是否有更多多行输入的标志
                has_more = false;
        }
        // 如果输入流出现问题或者接收到了EOF，则清空行内容
        line.clear();
        // 生成控制台中断事件，发送CTRL+C信号
        GenerateConsoleCtrlEvent(CTRL_C_EVENT, 0);
        // 返回false
        return false;
    }

    // 计算将宽字符转换为UTF-8编码所需的字节数
    int size_needed = WideCharToMultiByte(CP_UTF8, 0, &wline[0], (int)wline.size(), NULL, 0, NULL, NULL);
    // 调整字符串大小以容纳转换后的字节数
    line.resize(size_needed);
    // 将宽字符转换为UTF-8编码
    WideCharToMultiByte(CP_UTF8, 0, &wline[0], (int)wline.size(), &line[0], size_needed, NULL, NULL);
#else
    // 如果不是Windows系统
    if (!std::getline(std::cin, line)) {
        // 输入流出现问题或者接收到EOF
        line.clear();
        // 返回false
        return false;
    }
#endif
    // 如果输入的行不为空
    if (!line.empty()) {
        // 获取行的最后一个字符
        char last = line.back();
        // 如果最后一个字符是'/'，则移除并返回false
        if (last == '/') { 
            line.pop_back();
            return false;
        }
        if (last == '\\') { // '\\' changes the default action
            // 如果上一个字符是'\'，则改变默认操作
            line.pop_back();
            // 删除最后一个字符
            multiline_input = !multiline_input;
            // 切换多行输入状态
        }
    }
    line += '\n';
    // 在行末添加换行符

    // 默认情况下，如果设置了多行输入，则继续输入
    return multiline_input;
    // 返回多行输入状态
}

bool readline(std::string & line, bool multiline_input) {
    set_display(user_input);
    // 设置显示用户输入

    if (simple_io) {
        return readline_simple(line, multiline_input);
        // 如果是简单输入，调用简单输入函数
    }
    return readline_advanced(line, multiline_input);
    // 否则调用高级输入函数
}
这是一个代码块的结束标记，表示前面的函数或者循环等代码块的结束。
```