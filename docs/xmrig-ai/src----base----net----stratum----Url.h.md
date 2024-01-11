# `xmrig\src\base\net\stratum\Url.h`

```
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 许可证的版本，或（在您的选择）任何以后的版本。
 *
 * 本程序是希望它有用
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_URL_H
#define XMRIG_URL_H

#include "base/tools/String.h"

namespace xmrig {

class Url
{
public:
    enum Scheme {
        UNSPECIFIED,
        STRATUM,
        DAEMON,
        SOCKS5
    };

    // 默认构造函数
    Url() = default;
    // 从 URL 字符串构造函数
    Url(const char *url);
    // 从主机、端口、是否使用 TLS 和协议类型构造函数
    Url(const char *host, uint16_t port, bool tls = false, Scheme scheme = UNSPECIFIED);

    // 返回是否使用 TLS
    inline bool isTLS() const                           { return m_tls; }
    // 返回 URL 是否有效
    inline bool isValid() const                         { return !m_host.isNull() && m_port > 0; }
    // 返回主机名
    inline const String &host() const                   { return m_host; }
    // 返回完整 URL
    inline const String &url() const                    { return m_url; }
    // 返回协议类型
    inline Scheme scheme() const                        { return m_scheme; }
    // 返回端口号
    inline uint16_t port() const                        { return m_port; }

    // 不等于运算符重载
    inline bool operator!=(const Url &other) const      { return !isEqual(other); }
    // 等于运算符重载
    inline bool operator==(const Url &other) const      { return isEqual(other); }

    // 判断是否与另一个 URL 相等
    bool isEqual(const Url &other) const;

protected:
    // 解析 URL 字符串
    bool parse(const char *url);
    // 解析 IPv6 地址
    bool parseIPv6(const char *addr);

    // 是否使用 TLS
    bool m_tls      = false;
    # 定义一个枚举类型的变量m_scheme，初始值为UNSPECIFIED
    Scheme m_scheme = UNSPECIFIED;
    # 定义一个字符串类型的变量m_host
    String m_host;
    # 定义一个字符串类型的变量m_url
    String m_url;
    # 定义一个16位无符号整数类型的变量m_port，并初始化为3333
    uint16_t m_port = 3333;
}; 
// 结束了 xmrig 命名空间的定义

} /* namespace xmrig */
// 声明 xmrig 命名空间的结束

#endif /* XMRIG_URL_H */
// 结束了 XMRIG_URL_H 文件的条件编译
```