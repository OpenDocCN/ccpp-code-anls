# `xmrig\src\base\net\stratum\ProxyUrl.h`

```cpp
/* XMRig
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证，适销性或特定用途的保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_PROXYURL_H
#define XMRIG_PROXYURL_H


#include "base/net/stratum/Url.h"


namespace xmrig {


class ProxyUrl : public Url
{
public:
    inline ProxyUrl() { m_port = 0; }  // 初始化代理 URL 对象的端口为 0

    ProxyUrl(const rapidjson::Value &value);  // 从 JSON 值构造代理 URL 对象

    inline bool isValid() const { return m_port > 0 && (m_scheme == UNSPECIFIED || m_scheme == SOCKS5); }  // 检查代理 URL 对象是否有效

    const String &host() const;  // 获取代理 URL 对象的主机名
    rapidjson::Value toJSON(rapidjson::Document &doc) const;  // 将代理 URL 对象转换为 JSON 值
};


} /* namespace xmrig */


#endif /* XMRIG_PROXYURL_H */
```