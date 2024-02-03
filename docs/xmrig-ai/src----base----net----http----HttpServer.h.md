# `xmrig\src\base\net\http\HttpServer.h`

```cpp
/* XMRig
 * 版权所有 (c) 2014-2019 heapwolf    <https://github.com/heapwolf>
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证。
 * 有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_HTTPSERVER_H
#define XMRIG_HTTPSERVER_H

#include "base/kernel/interfaces/ITcpServerListener.h"  // 包含 ITcpServerListener 接口
#include "base/tools/Object.h"  // 包含 Object 类

#include <memory>  // 包含内存管理相关的头文件

namespace xmrig {

class IHttpListener;  // 声明 IHttpListener 接口

class HttpServer : public ITcpServerListener  // 定义 HttpServer 类，继承自 ITcpServerListener 接口
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(HttpServer)  // 禁用默认的拷贝和移动构造函数

    HttpServer(const std::shared_ptr<IHttpListener> &listener);  // 构造函数，接受一个 IHttpListener 接口的共享指针
    ~HttpServer() override;  // 虚析构函数

protected:
    void onConnection(uv_stream_t *stream, uint16_t port) override;  // 重写 ITcpServerListener 接口的 onConnection 函数

private:
    std::weak_ptr<IHttpListener> m_listener;  // 一个指向 IHttpListener 接口的弱引用指针
};

} // namespace xmrig

#endif // XMRIG_HTTPSERVER_H
```