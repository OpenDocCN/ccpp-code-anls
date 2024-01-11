# `xmrig\src\base\io\Async.cpp`

```
/*
 * XMRig
 * 版权所有（c）2015-2020 libuv 项目贡献者。
 * 版权所有（c）2020      cohcho      <https://github.com/cohcho>
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发或修改它，
 * 由自由软件基金会发布，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，但没有任何保证；甚至没有适销性或特定用途的暗示保证。有关更多详细信息，请参见GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/io/Async.h"
#include "base/kernel/interfaces/IAsyncListener.h"
#include "base/tools/Handle.h"


// 自2019.05.16，版本1.29.0（稳定版）https://github.com/xmrig/xmrig/pull/1889
#if (UV_VERSION_MAJOR >= 1) && (UV_VERSION_MINOR >= 29) && defined(__linux__)
#include <sys/eventfd.h>
#include <sys/poll.h>
#include <unistd.h>
#include <cstdlib>


namespace xmrig {


// 定义一个结构体uv_async_t，继承自uv_poll_t
struct uv_async_t: uv_poll_t
{
    // 定义一个函数指针类型uv_async_cb
    using uv_async_cb = void (*)(uv_async_t *);
    // 析构函数
    ~uv_async_t();
    // 文件描述符，默认值为-1
    int m_fd = -1;
    // 回调函数指针，默认值为nullptr
    uv_async_cb m_cb = nullptr;
};


// 使用别名定义uv_async_cb
using uv_async_cb = uv_async_t::uv_async_cb;


// uv_async_t的析构函数实现
uv_async_t::~uv_async_t()
{
    // 关闭文件描述符
    close(m_fd);
}


// 定义一个静态函数on_schedule，参数为uv_poll_t指针，两个int类型参数
static void on_schedule(uv_poll_t *handle, int, int)
{
    // 静态变量，类型为uint64_t
    static uint64_t val;
    // 将handle转换为uv_async_t指针
    auto async = reinterpret_cast<uv_async_t *>(handle);
    # 无限循环，直到条件被打破
    for (;;) {
        # 从异步文件描述符中读取数据到val变量中，读取的字节数为sizeof(val)
        int r = read(async->m_fd, &val, sizeof(val));

        # 如果读取的字节数等于val的大小，则继续下一次循环
        if (r == sizeof(val))
            continue;

        # 如果读取的字节数不等于-1，则跳出循环
        if (r != -1)
            break;

        # 如果errno为EAGAIN或者errno为EWOULDBLOCK，则跳出循环
        if (errno == EAGAIN || errno == EWOULDBLOCK)
            break;

        # 如果errno为EINTR，则继续下一次循环
        if (errno == EINTR)
            continue;

        # 异常情况，中止程序
        abort();
    }
    # 如果异步回调函数存在，则调用该回调函数并传入async参数
    if (async->m_cb) {
        (*async->m_cb)(async);
    }
} // namespace xmrig
#endif


namespace xmrig {


class AsyncPrivate
{
public:
    Async::Callback callback; // 定义回调函数
    IAsyncListener *listener    = nullptr; // 定义异步监听器，默认为空
    uv_async_t *async           = nullptr; // 定义异步对象，默认为空
};


} // namespace xmrig


xmrig::Async::Async(Callback callback) : d_ptr(new AsyncPrivate()) // 异步构造函数，初始化私有成员
{
    d_ptr->callback     = std::move(callback); // 移动回调函数
    d_ptr->async        = new uv_async_t; // 创建异步对象
    d_ptr->async->data  = this; // 设置异步对象的数据为当前对象

    uv_async_init(uv_default_loop(), d_ptr->async, [](uv_async_t *handle) { static_cast<Async *>(handle->data)->d_ptr->callback(); }); // 初始化异步对象
}


xmrig::Async::Async(IAsyncListener *listener) : d_ptr(new AsyncPrivate()) // 异步构造函数，初始化私有成员
{
    d_ptr->listener     = listener; // 设置监听器
    d_ptr->async        = new uv_async_t; // 创建异步对象
    d_ptr->async->data  = this; // 设置异步对象的数据为当前对象

    uv_async_init(uv_default_loop(), d_ptr->async, [](uv_async_t *handle) { static_cast<Async *>(handle->data)->d_ptr->listener->onAsync(); }); // 初始化异步对象
}


xmrig::Async::~Async() // 异步析构函数
{
    Handle::close(d_ptr->async); // 关闭异步对象

    delete d_ptr; // 释放内存
}


void xmrig::Async::send() // 发送异步信号
{
    uv_async_send(d_ptr->async); // 发送异步信号
}
```