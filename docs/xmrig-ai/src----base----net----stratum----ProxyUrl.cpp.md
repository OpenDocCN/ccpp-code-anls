# `xmrig\src\base\net\stratum\ProxyUrl.cpp`

```
/* XMRig
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（按您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有适销性或特定用途的暗示保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include "base/net/stratum/ProxyUrl.h"
#include "3rdparty/rapidjson/document.h"


namespace xmrig {

static const String kLocalhost = "127.0.0.1";

} // namespace xmrig


xmrig::ProxyUrl::ProxyUrl(const rapidjson::Value &value)
{
    m_port = 0;

    if (value.IsString()) {
        parse(value.GetString());
    }
    else if (value.IsUint()) {
        m_port = value.GetUint();
    }
}


const xmrig::String &xmrig::ProxyUrl::host() const
{
    return m_host.isNull() && isValid() ? kLocalhost : m_host;
}


rapidjson::Value xmrig::ProxyUrl::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;
    if (!isValid()) {
        return Value(kNullType);
    }

    if (!m_host.isNull()) {
        return m_url.toJSON(doc);
    }

    return Value(m_port);
}
```