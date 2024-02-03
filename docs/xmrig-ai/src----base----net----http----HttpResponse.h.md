# `xmrig\src\base\net\http\HttpResponse.h`

```cpp
/*
 * XMRig
 * 版权所有 (c) 2014-2019 heapwolf    <https://github.com/heapwolf>
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它，
 * 其发布由自由软件基金会发布，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，但没有任何担保；甚至没有暗示的担保适用于特定目的。有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本，如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_HTTPRESPONSE_H
#define XMRIG_HTTPRESPONSE_H


#include <cstdint>
#include <map>
#include <string>


namespace xmrig {


class HttpResponse
{
public:
    // 构造函数，初始化 id 和 statusCode，默认为 200
    HttpResponse(uint64_t id, int statusCode = 200);

    // 返回状态码
    inline int statusCode() const                                           { return m_statusCode; }
    // 设置响应头
    inline void setHeader(const std::string &key, const std::string &value) { m_headers.insert({ key, value }); }
    // 设置状态码
    inline void setStatus(int code)                                         { m_statusCode = code; }

    // 检查连接是否存活
    bool isAlive() const;
    // 结束响应，发送数据
    void end(const char *data = nullptr, size_t size = 0);

private:
    const uint64_t m_id; // 响应 id
    int m_statusCode;    // 状态码
    std::map<const std::string, const std::string> m_headers; // 响应头
};


} // namespace xmrig


#endif // XMRIG_HTTPRESPONSE_H
```