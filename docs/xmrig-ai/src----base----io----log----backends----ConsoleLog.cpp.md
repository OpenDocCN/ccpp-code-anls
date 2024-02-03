# `xmrig\src\base\io\log\backends\ConsoleLog.cpp`

```cpp
/* XMRig
 * 版权所有（c）2019      Spudz76     <https://github.com/Spudz76>
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发或修改它
 *   由自由软件基金会发布，无论是许可证的第3版还是
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/io/log/backends/ConsoleLog.h"
#include "base/io/log/Log.h"
#include "base/kernel/config/Title.h"
#include "base/tools/Handle.h"


#include <cstdio>


xmrig::ConsoleLog::ConsoleLog(const Title &title)
{
    // 如果不支持控制台日志，则设置颜色为false并返回
    if (!isSupported()) {
        Log::setColors(false);
        return;
    }

    // 初始化控制台日志
    m_tty = new uv_tty_t;

    // 如果初始化失败，则设置颜色为false并返回
    if (uv_tty_init(uv_default_loop(), m_tty, 1, 0) < 0) {
        Log::setColors(false);
        return;
    }

    // 设置控制台模式为UV_TTY_MODE_NORMAL
    uv_tty_set_mode(m_tty, UV_TTY_MODE_NORMAL);

#   ifdef XMRIG_OS_WIN
    // 如果是Windows系统，则设置流为m_tty
    m_stream = reinterpret_cast<uv_stream_t*>(m_tty);

    // 获取标准输入句柄
    HANDLE handle = GetStdHandle(STD_INPUT_HANDLE);
    // 如果句柄有效
    if (handle != INVALID_HANDLE_VALUE) { // NOLINT(cppcoreguidelines-pro-type-cstyle-cast, performance-no-int-to-ptr)
        // 获取控制台模式
        DWORD mode = 0;
        if (GetConsoleMode(handle, &mode)) {
           // 关闭快速编辑模式，设置扩展标志
           mode &= ~ENABLE_QUICK_EDIT_MODE;
           SetConsoleMode(handle, mode | ENABLE_EXTENDED_FLAGS);
        }
    }

    // 如果启用了标题，则设置控制台标题
    if (title.isEnabled()) {
        SetConsoleTitleA(title.value());
    }
#   endif
}


xmrig::ConsoleLog::~ConsoleLog()
{
    // 关闭控制台日志
    Handle::close(m_tty);
}
void xmrig::ConsoleLog::print(uint64_t, int, const char *line, size_t, size_t size, bool colors)
{
    // 如果没有终端或者颜色设置不匹配，则直接返回
    if (!m_tty || Log::isColors() != colors) {
        return;
    }

#   ifdef XMRIG_OS_WIN
    // 初始化一个 uv_buf_t 对象，用于写入数据
    uv_buf_t buf = uv_buf_init(const_cast<char *>(line), static_cast<unsigned int>(size));

    // 如果不可写，则将数据输出到标准输出并刷新
    if (!isWritable()) {
        fputs(line, stdout);
        fflush(stdout);
    }
    // 否则尝试写入数据到流
    else {
        uv_try_write(m_stream, &buf, 1);
    }
#   else
    // 将数据输出到标准输出并刷新
    fputs(line, stdout);
    fflush(stdout);
#   endif
}


bool xmrig::ConsoleLog::isSupported()
{
    // 获取标准输出的句柄类型
    const uv_handle_type type = uv_guess_handle(1);
    // 判断句柄类型是否为终端或命名管道
    return type == UV_TTY || type == UV_NAMED_PIPE;
}


#ifdef XMRIG_OS_WIN
bool xmrig::ConsoleLog::isWritable() const
{
    // 如果流为空或者不可写，则返回 false
    if (!m_stream || uv_is_writable(m_stream) != 1) {
        return false;
    }

    // 判断是否支持写入操作
    return isSupported();
}
#endif
```