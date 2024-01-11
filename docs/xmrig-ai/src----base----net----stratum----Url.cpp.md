# `xmrig\src\base\net\stratum\Url.cpp`

```
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有对适销性或特定用途的隐含保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include "base/net/stratum/Url.h"


#include <cassert>
#include <cstring>
#include <cstdlib>
#include <cstdio>


#ifdef _MSC_VER
#   define strncasecmp _strnicmp
#endif


namespace xmrig {

static const char kStratumTcp[]            = "stratum+tcp://";
static const char kStratumSsl[]            = "stratum+ssl://";
static const char kSOCKS5[]                = "socks5://";

#ifdef XMRIG_FEATURE_HTTP
static const char kDaemonHttp[]            = "daemon+http://";
static const char kDaemonHttps[]           = "daemon+https://";
#endif

} // namespace xmrig


// 根据 URL 解析并初始化 Url 对象
xmrig::Url::Url(const char *url)
{
    parse(url);
}


// 根据主机、端口、TLS、协议方案初始化 Url 对象
xmrig::Url::Url(const char *host, uint16_t port, bool tls, Scheme scheme) :
    m_tls(tls),
    m_scheme(scheme),
    m_host(host),
    m_port(port)
{
    const size_t size = m_host.size() + 8;
    assert(size > 8);

    char *url = new char[size]();
    snprintf(url, size - 1, "%s:%d", m_host.data(), m_port);

    m_url = url;
}


// 判断两个 Url 对象是否相等
bool xmrig::Url::isEqual(const Url &other) const
{
    return (m_tls == other.m_tls && m_scheme == other.m_scheme && m_host == other.m_host && m_url == other.m_url && m_port == other.m_port);
}
bool xmrig::Url::parse(const char *url)
{
    // 如果 URL 为空指针，则返回 false
    if (url == nullptr) {
        return false;
    }

    // 在 URL 中查找 "://" 的位置
    const char *p    = strstr(url, "://");
    // 将 base 指针指向 URL
    const char *base = url;

    // 如果找到了 "://"
    if (p) {
        // 如果 URL 以 "stratum+tcp" 开头
        if (strncasecmp(url, kStratumTcp, sizeof(kStratumTcp) - 1) == 0) {
            m_scheme = STRATUM;
            m_tls    = false;
        }
        // 如果 URL 以 "stratum+ssl" 开头
        else if (strncasecmp(url, kStratumSsl, sizeof(kStratumSsl) - 1) == 0) {
            m_scheme = STRATUM;
            m_tls    = true;
        }
        // 如果 URL 以 "socks5" 开头
        else if (strncasecmp(url, kSOCKS5, sizeof(kSOCKS5) - 1) == 0) {
            m_scheme = SOCKS5;
            m_tls    = false;
        }
#       ifdef XMRIG_FEATURE_HTTP
        // 如果 URL 以 "https" 开头
        else if (strncasecmp(url, kDaemonHttps, sizeof(kDaemonHttps) - 1) == 0) {
            m_scheme = DAEMON;
            m_tls    = true;
        }
        // 如果 URL 以 "http" 开头
        else if (strncasecmp(url, kDaemonHttp, sizeof(kDaemonHttp) - 1) == 0) {
            m_scheme = DAEMON;
            m_tls    = false;
        }
#       endif
        // 如果 URL 不符合以上规则，则返回 false
        else {
            return false;
        }

        // 将 base 指针指向 "://" 后面的内容
        base = p + 3;
    }

    // 如果 base 为空或者以 "/" 开头，则返回 false
    if (!strlen(base) || *base == '/') {
        return false;
    }

    // 将 m_url 设置为传入的 URL
    m_url = url;
    // 如果 base 的第一个字符是 "["，则调用 parseIPv6 函数解析 IPv6 地址
    if (base[0] == '[') {
        return parseIPv6(base);
    }

    // 在 base 中查找 ":" 的位置
    const char *port = strchr(base, ':');
    // 如果没有找到 ":"，则将 m_host 设置为 base，并返回 true
    if (!port) {
        m_host = base;
        return true;
    }

    // 计算 port 的长度，并动态分配内存存储 host
    const auto size = static_cast<size_t>(port++ - base + 1);
    char *host      = new char[size]();
    memcpy(host, base, size - 1);

    // 将 m_host 设置为 host，将 m_port 设置为端口号
    m_host = host;
    m_port = static_cast<uint16_t>(strtol(port, nullptr, 10));

    return true;
}


bool xmrig::Url::parseIPv6(const char *addr)
{
    // 在 addr 中查找 "]" 的位置
    const char *end = strchr(addr, ']');
    // 如果没有找到 "]"，则返回 false
    if (!end) {
        return false;
    }

    // 在 end 中查找 ":" 的位置
    const char *port = strchr(end, ':');
    // 如果没有找到 ":"，则返回 false
    if (!port) {
        return false;
    }

    // 计算 host 的长度，并动态分配内存存储 host
    const auto size = static_cast<size_t>(end - addr);
    char *host        = new char[size]();
    memcpy(host, addr + 1, size - 1);

    // 将 m_host 设置为 host，将 m_port 设置为端口号
    m_host = host;
    m_port = static_cast<uint16_t>(strtol(port + 1, nullptr, 10));
}
    # 返回布尔值 True
    return true;
# 闭合前面的函数定义
```