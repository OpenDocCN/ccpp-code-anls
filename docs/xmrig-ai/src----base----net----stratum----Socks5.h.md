# `xmrig\src\base\net\stratum\Socks5.h`

```cpp
/* XMRig
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证适用于特定用途。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_SOCKS5_H
#define XMRIG_SOCKS5_H


#include "base/net/stratum/Client.h"


namespace xmrig {


class Client::Socks5
{
public:
    // 构造函数，初始化 Socks5 对象
    Socks5(Client *client);

    // 返回当前状态是否为 Ready
    inline bool isReady() const     { return m_state == Ready; }

    // 读取数据并处理
    bool read(const char *data, size_t size);
    // 进行握手
    void handshake();

private:
    // 定义状态枚举
    enum State {
        Created,
        SentInitialHandshake,
        SentFinalHandshake,
        Ready
    };

    // 判断是否为 IPv4 地址
    static bool isIPv4(const String &host, sockaddr_storage *addr);
    // 判断是否为 IPv6 地址
    static bool isIPv6(const String &host, sockaddr_storage *addr);

    // 进行连接
    void connect();

    // 客户端指针
    Client *m_client;
    // 下一个数据包的大小
    size_t m_nextSize   = 0;
    // 当前状态
    State m_state       = Created;
};


} /* namespace xmrig */


#endif /* XMRIG_SOCKS5_H */
```