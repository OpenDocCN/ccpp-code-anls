# `xmrig\src\base\net\http\Fetch.h`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 它可以是许可证的第3版，也可以是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_FETCH_H
#define XMRIG_FETCH_H

#include "3rdparty/llhttp/llhttp.h"  // 引入 llhttp 库
#include "3rdparty/rapidjson/fwd.h"  // 引入 rapidjson 库的前置声明
#include "base/tools/String.h"       // 引入 String 类

#include <map>                       // 引入 map 类
#include <memory>                    // 引入 memory 类
#include <string>                    // 引入 string 类

namespace xmrig {

class IHttpListener;

class FetchRequest
{
public:
    FetchRequest() = default;  // 默认构造函数
    FetchRequest(llhttp_method method, const String &host, uint16_t port, const String &path, bool tls = false, bool quiet = false, const char *data = nullptr, size_t size = 0, const char *contentType = nullptr);  // 构造函数
    FetchRequest(llhttp_method method, const String &host, uint16_t port, const String &path, const rapidjson::Value &value, bool tls = false, bool quiet = false);  // 构造函数

    void setBody(const char *data, size_t size, const char *contentType = nullptr);  // 设置请求体
    void setBody(const rapidjson::Value &value);  // 设置请求体

    inline bool hasBody() const { return method != HTTP_GET && method != HTTP_HEAD && !body.empty(); }  // 判断是否有请求体

    bool quiet              = false;  // 是否安静模式
    bool tls                = false;  // 是否使用 TLS
    llhttp_method method    = HTTP_GET;  // 请求方法，默认为 GET
    std::map<const std::string, const std::string> headers;  // 请求头
    std::string body;  // 请求体
    String fingerprint;  // 指纹
    String host;  // 主机
    # 声明一个字符串变量 path
    String path;
    # 声明一个 16 位无符号整数变量 port，并初始化为 0
    uint16_t port           = 0;
    # 声明一个 64 位无符号整数变量 timeout，并初始化为 0
    uint64_t timeout        = 0;
// 结束 fetch 函数的声明
};

// 声明 fetch 函数，接受标签、请求、监听器、类型和 RPC ID 作为参数
void fetch(const char *tag, FetchRequest &&req, const std::weak_ptr<IHttpListener> &listener, int type = 0, uint64_t rpcId = 0);

// 结束 xmrig 命名空间
} // namespace xmrig

// 结束 XMRIG_FETCH_H 的定义
#endif // XMRIG_FETCH_H
```