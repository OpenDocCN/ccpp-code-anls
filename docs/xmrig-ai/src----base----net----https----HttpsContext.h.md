# `xmrig\src\base\net\https\HttpsContext.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   许可证的版本为3或（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，
 *   但没有任何保证；甚至没有暗示的保证。
 *   有关更多详细信息，请参见GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#ifndef XMRIG_HTTPSCONTEXT_H
#define XMRIG_HTTPSCONTEXT_H


using BIO = struct bio_st;
using SSL = struct ssl_st;


#include "base/net/http/HttpContext.h"
#include "base/net/tls/ServerTls.h"


namespace xmrig {


class TlsContext;


class HttpsContext : public HttpContext, public ServerTls
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(HttpsContext)

    // 构造函数，接受TLS上下文和HTTP监听器的弱引用
    HttpsContext(TlsContext *tls, const std::weak_ptr<IHttpListener> &listener);
    // 析构函数
    ~HttpsContext() override;

    // 向数据中追加字符
    void append(char *data, size_t size);

protected:
    // 重写ServerTls的方法，向BIO写入数据
    bool write(BIO *bio) override;
    // 重写ServerTls的方法，解析数据
    void parse(char *data, size_t size) override;
    // 重写ServerTls的方法，关闭连接
    void shutdown() override;

    // 重写HttpContext的方法，向连接写入数据
    void write(std::string &&data, bool close) override;

private:
    // TLS模式枚举
    enum TlsMode : uint32_t {
      TLS_AUTO,
      TLS_OFF,
      TLS_ON
    };

    // 是否关闭连接
    bool m_close    = false;
    // TLS模式
    TlsMode m_mode  = TLS_AUTO;
};


} // namespace xmrig


#endif // XMRIG_HTTPSCONTEXT_H
```