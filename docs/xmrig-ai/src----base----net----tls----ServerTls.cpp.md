# `xmrig\src\base\net\tls\ServerTls.cpp`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（按您的选择）任何以后的版本。
 *
 *   本程序是希望它有用的，但没有任何保证；甚至没有适销性或特定用途的暗示保证。有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include "base/net/tls/ServerTls.h"


#include <algorithm>
#include <cassert>
#include <cstring>
#include <openssl/ssl.h>


xmrig::ServerTls::ServerTls(SSL_CTX *ctx) :
    m_ctx(ctx)
{
}


xmrig::ServerTls::~ServerTls()
{
    if (m_ssl) {
        SSL_free(m_ssl);
    }
}


bool xmrig::ServerTls::isHTTP(const char *data, size_t size)
{
    assert(size > 0);

    static const char test[6] = "GET /";

    return size > 0 && memcmp(data, test, std::min(size, sizeof(test) - 1)) == 0;
}


bool xmrig::ServerTls::isTLS(const char *data, size_t size)
{
    assert(size > 0);

    static const uint8_t test[3] = { 0x16, 0x03, 0x01 };

    return size > 0 && memcmp(data, test, std::min(size, sizeof(test))) == 0;
}


bool xmrig::ServerTls::send(const char *data, size_t size)
{
    SSL_write(m_ssl, data, size);  // 使用 SSL 连接发送数据

    return write(m_write);  // 返回写入结果
}


void xmrig::ServerTls::read(const char *data, size_t size)
{
    if (!m_ssl) {
        m_ssl = SSL_new(m_ctx);  // 如果 SSL 对象不存在，则创建一个新的 SSL 对象

        m_write = BIO_new(BIO_s_mem());  // 创建一个新的内存 BIO 对象
        m_read  = BIO_new(BIO_s_mem());  // 创建一个新的内存 BIO 对象

        SSL_set_accept_state(m_ssl);  // 设置 SSL 对象为接收状态
        SSL_set_bio(m_ssl, m_read, m_write);  // 设置 SSL 对象的读写 BIO
    }


    BIO_write(m_read, data, size);  // 将数据写入读取 BIO
    # 如果 SSL 握手未完成
    if (!SSL_is_init_finished(m_ssl)) {
        # 执行 SSL 握手
        const int rc = SSL_do_handshake(m_ssl);
        
        # 如果握手返回值小于 0 并且 SSL 错误为 SSL_ERROR_WANT_READ
        if (rc < 0 && SSL_get_error(m_ssl, rc) == SSL_ERROR_WANT_READ) {
            # 写入数据
            write(m_write);
        } 
        # 如果握手成功
        else if (rc == 1) {
            # 写入数据
            write(m_write);
            
            # 设置就绪状态为 true
            m_ready = true;
            # 读取数据
            read();
        }
        # 如果握手失败
        else {
            # 关闭连接
            shutdown();
        }
        
        # 返回
        return;
    }
    
    # 读取数据
    read();
# 读取服务器端的数据
void xmrig::ServerTls::read()
{
    # 创建一个静态的字符数组，用于存储读取的数据
    static char buf[16384]{};

    # 初始化已读取的字节数为0
    int bytes_read = 0;
    # 当成功读取数据时，循环执行读取和解析操作
    while ((bytes_read = SSL_read(m_ssl, buf, sizeof(buf))) > 0) {
        # 调用parse函数对读取的数据进行解析
        parse(buf, bytes_read);
    }
}
```