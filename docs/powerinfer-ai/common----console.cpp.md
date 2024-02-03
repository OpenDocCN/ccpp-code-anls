# `PowerInfer\common\console.cpp`

```cpp
#include "console.h"
#include <vector>
#include <iostream>

#if defined(_WIN32)
#define WIN32_LEAN_AND_MEAN
#ifndef NOMINMAX
#define NOMINMAX
#endif
#include <windows.h>
#include <fcntl.h>
#include <io.h>
#ifndef ENABLE_VIRTUAL_TERMINAL_PROCESSING
#define ENABLE_VIRTUAL_TERMINAL_PROCESSING 0x0004
#endif
#else
#include <climits>
#include <sys/ioctl.h>
#include <unistd.h>
#include <wchar.h>
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <termios.h>
#endif

#define ANSI_COLOR_RED     "\x1b[31m"
#define ANSI_COLOR_GREEN   "\x1b[32m"
#define ANSI_COLOR_YELLOW  "\x1b[33m"
#define ANSI_COLOR_BLUE    "\x1b[34m"
#define ANSI_COLOR_MAGENTA "\x1b[35m"
#define ANSI_COLOR_CYAN    "\x1b[36m"
#define ANSI_COLOR_RESET   "\x1b[0m"
#define ANSI_BOLD          "\x1b[1m"

namespace console {

    //
    // Console state
    //

    static bool      advanced_display = false;  // 是否使用高级显示功能，默认为false
    static bool      simple_io        = true;   // 是否使用简单输入输出，默认为true
    static display_t current_display  = reset;  // 当前显示状态，默认为reset

    static FILE*     out              = stdout;  // 输出流，默认为标准输出流

#if defined (_WIN32)
    static void*     hConsole;  // Windows 控制台句柄
#else
    static FILE*     tty              = nullptr;  // 终端文件流，默认为空指针
    static termios   initial_state;  // 终端初始状态
#endif

    //
    // Init and cleanup
    //

    void init(bool use_simple_io, bool use_advanced_display) {
        advanced_display = use_advanced_display;  // 设置是否使用高级显示功能
        simple_io = use_simple_io;  // 设置是否使用简单输入输出
#if defined(_WIN32)
        // Windows-specific console initialization
        // 定义了_WIN32的情况下，进行Windows特定的控制台初始化

        DWORD dwMode = 0;
        // 定义一个DWORD类型的变量dwMode，并初始化为0
        hConsole = GetStdHandle(STD_OUTPUT_HANDLE);
        // 获取标准输出的句柄
        if (hConsole == INVALID_HANDLE_VALUE || !GetConsoleMode(hConsole, &dwMode)) {
            // 如果获取标准输出句柄失败，或者无法获取控制台模式
            hConsole = GetStdHandle(STD_ERROR_HANDLE);
            // 获取标准错误输出的句柄
            if (hConsole != INVALID_HANDLE_VALUE && (!GetConsoleMode(hConsole, &dwMode))) {
                // 如果获取标准错误输出句柄成功，但无法获取控制台模式
                hConsole = nullptr;
                // 将hConsole置为空指针
                simple_io = true;
                // 将simple_io标记为true
            }
        }
        if (hConsole) {
            // 如果hConsole不为空
            if (advanced_display && !(dwMode & ENABLE_VIRTUAL_TERMINAL_PROCESSING) &&
                !SetConsoleMode(hConsole, dwMode | ENABLE_VIRTUAL_TERMINAL_PROCESSING)) {
                // 检查条件组合以减少嵌套
                // 如果advanced_display为true，并且dwMode中不包含ENABLE_VIRTUAL_TERMINAL_PROCESSING，并且无法设置控制台模式
                advanced_display = false;
                // 将advanced_display标记为false
            }
            // 设置控制台输出代码页为UTF8
            SetConsoleOutputCP(CP_UTF8);
        }
        HANDLE hConIn = GetStdHandle(STD_INPUT_HANDLE);
        // 获取标准输入的句柄
        if (hConIn != INVALID_HANDLE_VALUE && GetConsoleMode(hConIn, &dwMode)) {
            // 如果获取标准输入句柄成功，并且能够获取控制台模式
            // 设置控制台输入代码页为UTF16
            _setmode(_fileno(stdin), _O_WTEXT);

            // 设置ICANON (ENABLE_LINE_INPUT)和ECHO (ENABLE_ECHO_INPUT)
            if (simple_io) {
                // 如果simple_io为true
                dwMode |= ENABLE_LINE_INPUT | ENABLE_ECHO_INPUT;
                // 启用行输入和回显输入
            } else {
                dwMode &= ~(ENABLE_LINE_INPUT | ENABLE_ECHO_INPUT);
                // 禁用行输入和回显输入
            }
            if (!SetConsoleMode(hConIn, dwMode)) {
                // 如果无法设置控制台模式
                simple_io = true;
                // 将simple_io标记为true
            }
        }
    // 如果不是简单的输入输出模式，则进行 POSIX 特定的控制台初始化
    if (!simple_io) {
        // 保存初始的终端设置
        struct termios new_termios;
        tcgetattr(STDIN_FILENO, &initial_state);
        new_termios = initial_state;
        // 关闭规范模式和回显
        new_termios.c_lflag &= ~(ICANON | ECHO);
        new_termios.c_cc[VMIN] = 1;
        new_termios.c_cc[VTIME] = 0;
        // 设置新的终端设置
        tcsetattr(STDIN_FILENO, TCSANOW, &new_termios);

        // 打开控制台设备文件，用于输出
        tty = fopen("/dev/tty", "w+");
        if (tty != nullptr) {
            out = tty;
        }
    }

    // 设置本地化环境
    setlocale(LC_ALL, "");
#endif
}

void cleanup() {
    // 重置控制台显示
    set_display(reset);

    // 在 POSIX 系统上恢复设置
#if !defined(_WIN32)
    if (!simple_io) {
        if (tty != nullptr) {
            out = stdout;
            fclose(tty);
            tty = nullptr;
        }
        // 恢复初始的终端设置
        tcsetattr(STDIN_FILENO, TCSANOW, &initial_state);
    }
#endif
}

//
// 显示和输入输出
//

// 跟踪当前显示，并且只在显示改变时发出 ANSI 代码
void set_display(display_t display) {
    if (advanced_display && current_display != display) {
        fflush(stdout);
        switch(display) {
            case reset:
                fprintf(out, ANSI_COLOR_RESET);
                break;
            case prompt:
                fprintf(out, ANSI_COLOR_YELLOW);
                break;
            case user_input:
                fprintf(out, ANSI_BOLD ANSI_COLOR_GREEN);
                break;
            case error:
                fprintf(out, ANSI_BOLD ANSI_COLOR_RED);
        }
        current_display = display;
        fflush(out);
    }
}

static char32_t getchar32() {
#if defined(_WIN32)
        // 获取标准输入句柄
        HANDLE hConsole = GetStdHandle(STD_INPUT_HANDLE);
        // 初始化高代理项
        wchar_t high_surrogate = 0;

        // 进入循环，等待输入
        while (true) {
            // 读取控制台输入记录
            INPUT_RECORD record;
            DWORD count;
            // 如果无法读取或者读取数量为0，则返回WEOF
            if (!ReadConsoleInputW(hConsole, &record, 1, &count) || count == 0) {
                return WEOF;
            }

            // 如果事件类型为按键事件且按键按下
            if (record.EventType == KEY_EVENT && record.Event.KeyEvent.bKeyDown) {
                // 获取Unicode字符
                wchar_t wc = record.Event.KeyEvent.uChar.UnicodeChar;
                // 如果字符为0，则继续循环
                if (wc == 0) {
                    continue;
                }

                // 检查是否为高代理项
                if ((wc >= 0xD800) && (wc <= 0xDBFF)) { // Check if wc is a high surrogate
                    high_surrogate = wc;
                    continue;
                }
                // 检查是否为低代理项
                if ((wc >= 0xDC00) && (wc <= 0xDFFF)) { // Check if wc is a low surrogate
                    // 检查是否存在高代理项
                    if (high_surrogate != 0) { // Check if we have a high surrogate
                        // 计算并返回代理项对应的码点
                        return ((high_surrogate - 0xD800) << 10) + (wc - 0xDC00) + 0x10000;
                    }
                }

                // 重置高代理项
                high_surrogate = 0; // Reset the high surrogate
                // 返回Unicode字符
                return static_cast<char32_t>(wc);
            }
        }
#else
        // 读取宽字符
        wchar_t wc = getwchar();
        // 如果读取到的宽字符为WEOF，则返回WEOF
        if (static_cast<wint_t>(wc) == WEOF) {
            return WEOF;
        }

        // 如果wchar_t的最大值为0xFFFF
#if WCHAR_MAX == 0xFFFF
        // 检查是否为高代理项
        if ((wc >= 0xD800) && (wc <= 0xDBFF)) { // Check if wc is a high surrogate
            // 读取下一个宽字符
            wchar_t low_surrogate = getwchar();
            // 检查下一个宽字符是否为低代理项
            if ((low_surrogate >= 0xDC00) && (low_surrogate <= 0xDFFF)) { // Check if the next wchar is a low surrogate
                // 计算并返回代理项对应的码点
                return (static_cast<char32_t>(wc & 0x03FF) << 10) + (low_surrogate & 0x03FF) + 0x10000;
            }
        }
        // 检查是否为无效的代理项对
        if ((wc >= 0xD800) && (wc <= 0xDFFF)) { // Invalid surrogate pair
            // 返回替换字符U+FFFD
            return 0xFFFD; // Return the replacement character U+FFFD
        }
#endif

        // 返回Unicode字符
        return static_cast<char32_t>(wc);
#endif
    }

    // 弹出光标
    static void pop_cursor() {
#if defined(_WIN32)
        # 如果是在 Windows 平台
        if (hConsole != NULL):
            # 获取控制台屏幕缓冲区信息
            CONSOLE_SCREEN_BUFFER_INFO bufferInfo;
            GetConsoleScreenBufferInfo(hConsole, &bufferInfo);

            # 获取新的光标位置
            COORD newCursorPosition = bufferInfo.dwCursorPosition;
            if (newCursorPosition.X == 0):
                newCursorPosition.X = bufferInfo.dwSize.X - 1;
                newCursorPosition.Y -= 1;
            else:
                newCursorPosition.X -= 1;

            # 设置新的光标位置
            SetConsoleCursorPosition(hConsole, newCursorPosition);
            return;
#endif
        # 在 Windows 平台之外，执行退格操作
        putc('\b', out);
    }

    # 估算字符宽度
    static int estimateWidth(char32_t codepoint):
#if defined(_WIN32)
        (void)codepoint;
        return 1;
#else
        return wcwidth(codepoint);
#endif

    # 输出字符
    static int put_codepoint(const char* utf8_codepoint, size_t length, int expectedWidth):
#if defined(_WIN32)
        # 如果是在 Windows 平台
        CONSOLE_SCREEN_BUFFER_INFO bufferInfo;
        if (!GetConsoleScreenBufferInfo(hConsole, &bufferInfo)):
            # 使用默认值
            return expectedWidth;
        COORD initialPosition = bufferInfo.dwCursorPosition;
        DWORD nNumberOfChars = length;
        WriteConsole(hConsole, utf8_codepoint, nNumberOfChars, &nNumberOfChars, NULL);

        CONSOLE_SCREEN_BUFFER_INFO newBufferInfo;
        GetConsoleScreenBufferInfo(hConsole, &newBufferInfo);

        # 如果在最后一列，计算真实位置
        if (utf8_codepoint[0] != 0x09 && initialPosition.X == newBufferInfo.dwSize.X - 1):
            DWORD nNumberOfChars;
            WriteConsole(hConsole, &" \b", 2, &nNumberOfChars, NULL);
            GetConsoleScreenBufferInfo(hConsole, &newBufferInfo);

        int width = newBufferInfo.dwCursorPosition.X - initialPosition.X;
        if (width < 0):
            width += newBufferInfo.dwSize.X;
        return width;
    // 如果没有期望的宽度或者没有终端，则可以信任期望的宽度
    if (expectedWidth >= 0 || tty == nullptr) {
        // 将 UTF-8 编码的字符写入输出流
        fwrite(utf8_codepoint, length, 1, out);
        // 返回期望的宽度
        return expectedWidth;
    }

    // 发送查询光标位置的控制序列
    fputs("\033[6n", tty); 
    int x1;
    int y1;
    int x2;
    int y2;
    int results = 0;
    // 从终端读取光标位置
    results = fscanf(tty, "\033[%d;%dR", &y1, &x1);

    // 将 UTF-8 编码的字符写入终端
    fwrite(utf8_codepoint, length, 1, tty);

    // 再次发送查询光标位置的控制序列
    fputs("\033[6n", tty); 
    // 从终端读取光标位置
    results += fscanf(tty, "\033[%d;%dR", &y2, &x2);

    // 如果无法成功获取光标位置，则返回期望的宽度
    if (results != 4) {
        return expectedWidth;
    }

    // 计算字符宽度
    int width = x2 - x1;
    if (width < 0) {
        // 考虑文本换行后的宽度
        struct winsize w;
        ioctl(STDOUT_FILENO, TIOCGWINSZ, &w);
        width += w.ws_col;
    }
    // 返回字符宽度
    return width;
#endif
    }

    // 替换最后一个字符
    static void replace_last(char ch) {
#if defined(_WIN32)
        // 恢复光标位置
        pop_cursor();
        // 输出字符
        put_codepoint(&ch, 1, 1);
#else
        // 在终端上输出退格符和字符
        fprintf(out, "\b%c", ch);
#endif
    }
    // 将 Unicode 字符转换为 UTF-8 编码并追加到输出字符串中
    static void append_utf8(char32_t ch, std::string & out) {
        if (ch <= 0x7F) {
            out.push_back(static_cast<unsigned char>(ch));
        } else if (ch <= 0x7FF) {
            out.push_back(static_cast<unsigned char>(0xC0 | ((ch >> 6) & 0x1F)));
            out.push_back(static_cast<unsigned char>(0x80 | (ch & 0x3F)));
        } else if (ch <= 0xFFFF) {
            out.push_back(static_cast<unsigned char>(0xE0 | ((ch >> 12) & 0x0F)));
            out.push_back(static_cast<unsigned char>(0x80 | ((ch >> 6) & 0x3F)));
            out.push_back(static_cast<unsigned char>(0x80 | (ch & 0x3F)));
        } else if (ch <= 0x10FFFF) {
            out.push_back(static_cast<unsigned char>(0xF0 | ((ch >> 18) & 0x07)));
            out.push_back(static_cast<unsigned char>(0x80 | ((ch >> 12) & 0x3F)));
            out.push_back(static_cast<unsigned char>(0x80 | ((ch >> 6) & 0x3F)));
            out.push_back(static_cast<unsigned char>(0x80 | (ch & 0x3F)));
        } else {
            // 无效的 Unicode 代码点
        }
    }

    // 从字符串中移除最后一个 UTF-8 字符的辅助函数
    static void pop_back_utf8_char(std::string & line) {
        if (line.empty()) {
            return;
        }

        size_t pos = line.length() - 1;

        // 找到最后一个 UTF-8 字符的起始位置（向前检查最多 4 个字节）
        for (size_t i = 0; i < 3 && pos > 0; ++i, --pos) {
            if ((line[pos] & 0xC0) != 0x80) {
                break; // 找到字符的起始位置
            }
        }
        line.erase(pos);
    }

    // 从输入流中读取一行简单的字符串
    static bool readline_simple(std::string & line, bool multiline_input) {
#if defined(_WIN32)
        // 如果是 Windows 系统
        std::wstring wline;
        // 从标准输入流中读取一行字符串，存储在 wline 中
        if (!std::getline(std::wcin, wline)) {
            // 输入流出现问题或者接收到了 EOF
            line.clear();
            // 生成控制台中断事件，发送 CTRL+C 信号
            GenerateConsoleCtrlEvent(CTRL_C_EVENT, 0);
            return false;
        }

        // 将宽字符转换为多字节字符，获取所需的缓冲区大小
        int size_needed = WideCharToMultiByte(CP_UTF8, 0, &wline[0], (int)wline.size(), NULL, 0, NULL, NULL);
        line.resize(size_needed);
        // 将宽字符转换为多字节字符，存储在 line 中
        WideCharToMultiByte(CP_UTF8, 0, &wline[0], (int)wline.size(), &line[0], size_needed, NULL, NULL);
#else
        // 如果不是 Windows 系统
        if (!std::getline(std::cin, line)) {
            // 输入流出现问题或者接收到了 EOF
            line.clear();
            return false;
        }
#endif
        // 如果输入的行不为空
        if (!line.empty()) {
            // 获取行的最后一个字符
            char last = line.back();
            // 如果最后一个字符是 '/'，则返回 false
            if (last == '/') { // Always return control on '/' symbol
                line.pop_back();
                return false;
            }
            // 如果最后一个字符是 '\\'，则改变默认操作
            if (last == '\\') { // '\\' changes the default action
                line.pop_back();
                // 切换多行输入状态
                multiline_input = !multiline_input;
            }
        }
        // 在行末尾添加换行符
        line += '\n';

        // 默认情况下，如果设置了多行输入状态，则继续输入
        return multiline_input;
    }

    // 读取一行输入
    bool readline(std::string & line, bool multiline_input) {
        // 设置用户输入的显示
        set_display(user_input);

        // 如果是简单输入模式
        if (simple_io) {
            return readline_simple(line, multiline_input);
        }
        // 如果是高级输入模式
        return readline_advanced(line, multiline_input);
    }
}
```