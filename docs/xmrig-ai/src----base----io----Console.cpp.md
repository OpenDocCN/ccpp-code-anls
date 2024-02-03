# `xmrig\src\base\io\Console.cpp`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 许可证的版本，或（在您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/io/Console.h"
#include "base/kernel/interfaces/IConsoleListener.h"
#include "base/tools/Handle.h"


// 构造函数，初始化控制台对象
xmrig::Console::Console(IConsoleListener *listener)
    : m_listener(listener)
{
    // 如果不支持控制台，则返回
    if (!isSupported()) {
        return;
    }

    // 初始化控制台的 TTY 对象
    m_tty = new uv_tty_t;
    m_tty->data = this;
    uv_tty_init(uv_default_loop(), m_tty, 0, 1);

    // 如果 TTY 不可读，则返回
    if (!uv_is_readable(reinterpret_cast<uv_stream_t*>(m_tty))) {
        return;
    }

    // 设置 TTY 为原始模式
    uv_tty_set_mode(m_tty, UV_TTY_MODE_RAW);
    // 开始读取输入
    uv_read_start(reinterpret_cast<uv_stream_t*>(m_tty), Console::onAllocBuffer, Console::onRead);
}


// 析构函数，关闭控制台对象
xmrig::Console::~Console()
{
    // 重置 TTY 模式
    uv_tty_reset_mode();

    // 关闭 TTY
    Handle::close(m_tty);
}


// 检查控制台是否支持
bool xmrig::Console::isSupported()
{
    // 获取标准输入的句柄类型
    const uv_handle_type type = uv_guess_handle(0);
    // 如果是 TTY 或命名管道，则支持
    return type == UV_TTY || type == UV_NAMED_PIPE;
}


// 分配缓冲区
void xmrig::Console::onAllocBuffer(uv_handle_t *handle, size_t, uv_buf_t *buf)
{
    auto console = static_cast<Console*>(handle->data);
    buf->len  = 1;
    buf->base = console->m_buf;
}


// 读取输入
void xmrig::Console::onRead(uv_stream_t *stream, ssize_t nread, const uv_buf_t *buf)
{
    // 读取输入的处理代码
}
    # 如果读取的字节数小于 0，则关闭流并返回
    if (nread < 0) {
        return uv_close(reinterpret_cast<uv_handle_t*>(stream), nullptr);
    }

    # 如果读取的字节数等于 1，则调用 Console 对象的监听器的 onConsoleCommand 方法，传入缓冲区的第一个字符
    if (nread == 1) {
        static_cast<Console*>(stream->data)->m_listener->onConsoleCommand(buf->base[0]);
    }
# 闭合前面的函数定义
```