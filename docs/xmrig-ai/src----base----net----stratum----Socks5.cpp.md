# `xmrig\src\base\net\stratum\Socks5.cpp`

```
/* XMRig
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款
 *   它，无论是许可证的第 3 版还是
 *   （根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include "base/net/stratum/Socks5.h"


xmrig::Client::Socks5::Socks5(Client *client) :
    m_client(client)
{
}


bool xmrig::Client::Socks5::read(const char *data, size_t size)
{
    if (size < m_nextSize) {
        return false;
    }

    if (data[0] == 0x05 && data[1] == 0x00) {
        if (m_state == SentInitialHandshake) {
            connect();
        }
        else {
            m_state = Ready;
        }
    }
    else {
        m_client->close();
    }

    return true;
}


void xmrig::Client::Socks5::handshake()
{
    m_nextSize  = 2;
    m_state     = SentInitialHandshake;

    char buf[3] = { 0x05, 0x01, 0x00 };

    m_client->write(uv_buf_init(buf, sizeof (buf)));
}


bool xmrig::Client::Socks5::isIPv4(const String &host, sockaddr_storage *addr)
{
    return uv_ip4_addr(host.data(), 0, reinterpret_cast<sockaddr_in *>(addr)) == 0;
}


bool xmrig::Client::Socks5::isIPv6(const String &host, sockaddr_storage *addr)
{
   return uv_ip6_addr(host.data(), 0, reinterpret_cast<sockaddr_in6 *>(addr)) == 0;
}


void xmrig::Client::Socks5::connect()
{
    m_nextSize  = 5;
    m_state     = SentFinalHandshake;
    // 获取客户端连接池中的主机信息
    const auto &host = m_client->pool().host();
    // 创建一个存储数据的缓冲区
    std::vector<uint8_t> buf;
    // 创建一个存储地址信息的结构体
    sockaddr_storage addr{};

    // 如果主机是 IPv4 地址
    if (isIPv4(host, &addr)) {
        // 调整缓冲区大小
        buf.resize(10);
        // 设置协议版本为 IPv4
        buf[3] = 0x01;
        // 将地址信息拷贝到缓冲区中
        memcpy(buf.data() + 4, &reinterpret_cast<sockaddr_in *>(&addr)->sin_addr, 4);
    }
    // 如果主机是 IPv6 地址
    else if (isIPv6(host, &addr)) {
        // 调整缓冲区大小
        buf.resize(22);
        // 设置协议版本为 IPv6
        buf[3] = 0x04;
        // 将地址信息拷贝到缓冲区中
        memcpy(buf.data() + 4, &reinterpret_cast<sockaddr_in6 *>(&addr)->sin6_addr, 16);
    }
    // 如果主机不是 IPv4 或 IPv6 地址
    else {
        // 调整缓冲区大小
        buf.resize(host.size() + 7);
        // 设置协议版本为域名
        buf[3] = 0x03;
        // 设置主机名长度
        buf[4] = static_cast<uint8_t>(host.size());
        // 将主机名拷贝到缓冲区中
        memcpy(buf.data() + 5, host.data(), host.size());
    }

    // 设置协议版本
    buf[0] = 0x05;
    // 设置认证方式
    buf[1] = 0x01;
    // 保留字段
    buf[2] = 0x00;

    // 获取端口号并转换为网络字节序
    const uint16_t port = htons(m_client->pool().port());
    // 将端口号拷贝到缓冲区中
    memcpy(buf.data() + (buf.size() - sizeof(port)), &port, sizeof(port));

    // 将缓冲区中的数据写入客户端连接
    m_client->write(uv_buf_init(reinterpret_cast<char *>(buf.data()), buf.size()));
# 闭合前面的函数定义
```