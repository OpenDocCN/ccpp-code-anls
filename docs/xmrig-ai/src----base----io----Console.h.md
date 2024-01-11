# `xmrig\src\base\io\Console.h`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据GNU通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版，或者
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CONSOLE_H
#define XMRIG_CONSOLE_H


#include "base/tools/Object.h"


using uv_buf_t      = struct uv_buf_t;
using uv_handle_t   = struct uv_handle_s;
using uv_stream_t   = struct uv_stream_s;
using uv_tty_t      = struct uv_tty_s;

#ifdef XMRIG_OS_WIN
using ssize_t = intptr_t;
#else
#   include <sys/types.h>
#endif


namespace xmrig {


class IConsoleListener;


class Console
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(Console)

    Console(IConsoleListener *listener);  // 构造函数，接受一个IConsoleListener指针作为参数
    ~Console();  // 析构函数

private:
    static bool isSupported();  // 静态方法，用于检查控制台是否受支持

    static void onAllocBuffer(uv_handle_t *handle, size_t suggested_size, uv_buf_t *buf);  // 静态方法，用于分配缓冲区
    static void onRead(uv_stream_t *stream, ssize_t nread, const uv_buf_t *buf);  // 静态方法，用于读取数据

    char m_buf[1] = { 0 };  // 字符数组，用于存储数据
    IConsoleListener *m_listener;  // 指向IConsoleListener的指针
    uv_tty_t *m_tty = nullptr;  // 指向uv_tty_t的指针，初始化为nullptr
};


} /* namespace xmrig */


#endif /* XMRIG_CONSOLE_H */
```