# `xmrig\src\base\net\tls\ServerTls.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，
 *   但没有任何保证；甚至没有适销性或特定用途的暗示保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#ifndef XMRIG_SERVERTLS_H
#define XMRIG_SERVERTLS_H


使用BIO       = 结构体bio_st;
使用SSL       = 结构体ssl_st;
使用SSL_CTX   = 结构体ssl_ctx_st;



包括 "base/tools/Object.h"


在xmrig命名空间中


类ServerTls
{
public:
    禁用默认的拷贝和移动构造函数
    ServerTls(SSL_CTX *ctx);
    虚析构函数
    virtual ~ServerTls();

    静态方法，判断数据是否为HTTP
    static bool isHTTP(const char *data, size_t size);
    静态方法，判断数据是否为TLS
    static bool isTLS(const char *data, size_t size);

    发送数据
    bool send(const char *data, size_t size);
    读取数据
    void read(const char *data, size_t size);

protected:
    纯虚函数，写入数据到BIO
    virtual bool write(BIO *bio)                = 0;
    纯虚函数，解析数据
    virtual void parse(char *data, size_t size) = 0;
    纯虚函数，关闭连接
    virtual void shutdown()                     = 0;

private:
    读取数据
    void read();

    读取BIO
    BIO *m_read     = nullptr;
    写入BIO
    BIO *m_write    = nullptr;
    连接是否就绪
    bool m_ready    = false;
    SSL对象
    SSL *m_ssl      = nullptr;
    SSL_CTX对象
    SSL_CTX *m_ctx;
};


} // 在xmrig命名空间中
#endif /* XMRIG_SERVERTLS_H */
```