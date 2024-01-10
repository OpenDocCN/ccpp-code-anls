# `nmap\libpcap\sslutils.c`

```
"""
版权声明
"""
"""
导入必要的模块
"""
#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#ifdef HAVE_OPENSSL
#include <stdlib.h>

#include "portability.h"

#include "sslutils.h"

# 定义存储私钥的文件名
static const char *ssl_keyfile = "";   //!< file containing the private key in PEM format
# 定义存储服务器证书的文件名
static const char *ssl_certfile = "";  //!< file containing the server's certificate in PEM format
// 定义一个指向包含客户端信任的 CA 列表的文件的指针
static const char *ssl_rootfile = "";  //!< file containing the list of CAs trusted by the client
// TODO: a way to set ssl_rootfile from the command line, or an envvar?

// TODO: lock?
// 定义一个 SSL 上下文的指针
static SSL_CTX *ctx;

// 设置证书文件路径
void ssl_set_certfile(const char *certfile)
{
    ssl_certfile = certfile;
}

// 设置密钥文件路径
void ssl_set_keyfile(const char *keyfile)
{
    ssl_keyfile = keyfile;
}

// 初始化 SSL 上下文
int ssl_init_once(int is_server, int enable_compression, char *errbuf, size_t errbuflen)
{
    // 定义一个静态变量，用于标记是否已经初始化
    static int inited = 0;
    if (inited) return 0;

    // 初始化 SSL 库
    SSL_library_init();
    SSL_load_error_strings();
    OpenSSL_add_ssl_algorithms();
    if (enable_compression)
        SSL_COMP_get_compression_methods();

    // 根据是否是服务器端选择 SSL 方法
    SSL_METHOD const *meth =
        is_server ? SSLv23_server_method() : SSLv23_client_method();
    // 创建 SSL 上下文
    ctx = SSL_CTX_new(meth);
    if (! ctx)
    {
        // 如果无法获取新的 SSL 上下文，则输出错误信息并跳转到 die 标签
        snprintf(errbuf, errbuflen, "Cannot get a new SSL context: %s", ERR_error_string(ERR_get_error(), NULL));
        goto die;
    }

    // 设置 SSL 上下文的模式
    SSL_CTX_set_mode(ctx, SSL_MODE_AUTO_RETRY);

    if (is_server)
    {
        // 如果是服务器端，设置证书文件和私钥文件
        char const *certfile = ssl_certfile[0] ? ssl_certfile : "cert.pem";
        if (1 != SSL_CTX_use_certificate_file(ctx, certfile, SSL_FILETYPE_PEM))
        {
            snprintf(errbuf, errbuflen, "Cannot read certificate file %s: %s", certfile, ERR_error_string(ERR_get_error(), NULL));
            goto die;
        }

        char const *keyfile = ssl_keyfile[0] ? ssl_keyfile : "key.pem";
        if (1 != SSL_CTX_use_PrivateKey_file(ctx, keyfile, SSL_FILETYPE_PEM))
        {
            snprintf(errbuf, errbuflen, "Cannot read private key file %s: %s", keyfile, ERR_error_string(ERR_get_error(), NULL));
            goto die;
        }
    }
    else
    // ...
    {
        // 如果 SSL 根证书文件存在
        if (ssl_rootfile[0])
        {
            // 加载 SSL 根证书文件的位置到 SSL 上下文中
            if (! SSL_CTX_load_verify_locations(ctx, ssl_rootfile, 0))
            {
                // 如果加载失败，将错误信息写入 errbuf，并跳转到 die 标签处
                snprintf(errbuf, errbuflen, "Cannot read CA list from %s", ssl_rootfile);
                goto die;
            }
        }
        // 如果 SSL 根证书文件不存在
        else
        {
            // 设置 SSL 上下文的验证方式为 SSL_VERIFY_NONE
            SSL_CTX_set_verify(ctx, SSL_VERIFY_NONE, NULL);
        }
    }
#if 0
    // 如果无法从随机文件中加载随机数据，则返回错误信息并跳转到die标签处
    if (! RAND_load_file(RANDOM, 1024*1024))
    {
        snprintf(errbuf, errbuflen, "Cannot init random");
        goto die;
    }

    // 如果是服务器端，设置会话 ID 上下文
    if (is_server)
    {
        SSL_CTX_set_session_id_context(ctx, (void *)&s_server_session_id_context, sizeof(s_server_session_id_context));
    }
#endif

    // 初始化标志位
    inited = 1;
    // 返回成功
    return 0;

die:
    // 返回失败
    return -1;
}

// SSL 升级函数，根据参数创建 SSL 对象并进行 SSL 握手
SSL *ssl_promotion(int is_server, SOCKET s, char *errbuf, size_t errbuflen)
{
    // 如果 SSL 初始化失败，则返回空指针
    if (ssl_init_once(is_server, 1, errbuf, errbuflen) < 0) {
        return NULL;
    }

    // 创建 SSL 对象
    SSL *ssl = SSL_new(ctx); // TODO: also a DTLS context
    // 将 SSL 对象与套接字关联
    SSL_set_fd(ssl, (int)s);

    // 如果是服务器端，执行 SSL 握手
    if (is_server) {
        if (SSL_accept(ssl) <= 0) {
            snprintf(errbuf, errbuflen, "SSL_accept(): %s",
                    ERR_error_string(ERR_get_error(), NULL));
            return NULL;
        }
    } else {
        // 如果是客户端，执行 SSL 握手
        if (SSL_connect(ssl) <= 0) {
            snprintf(errbuf, errbuflen, "SSL_connect(): %s",
                    ERR_error_string(ERR_get_error(), NULL));
            return NULL;
        }
    }

    // 返回 SSL 对象
    return ssl;
}

// 结束使用 SSL 句柄；关闭连接并释放句柄
void ssl_finish(SSL *ssl)
{
    // 发送关闭连接的警告并释放 SSL 句柄
    SSL_shutdown(ssl);
    SSL_free(ssl);
}

// SSL 发送函数，向 SSL 连接发送数据
// 返回值：0 表示成功，-1 表示错误但连接已关闭（-2）
int ssl_send(SSL *ssl, char const *buffer, int size, char *errbuf, size_t errbuflen)
{
    // 使用 SSL 写入数据
    int status = SSL_write(ssl, buffer, size);
    // 如果写入成功
    if (status > 0)
    {
        // 如果 SSL_write() 成功写入了完整的内容，返回 0
        return 0;
    }
    else
    {
        // 获取 SSL 错误码
        int ssl_err = SSL_get_error(ssl, status); // TODO: does it pop the error?
        // 如果 SSL 错误码为 SSL_ERROR_ZERO_RETURN，返回 -2
        if (ssl_err == SSL_ERROR_ZERO_RETURN)
        {
            return -2;
        }
        // 如果 SSL 错误码为 SSL_ERROR_SYSCALL，执行以下操作
        else if (ssl_err == SSL_ERROR_SYSCALL)
        {
#ifndef _WIN32
            // 如果不是在 Windows 平台，并且发生连接重置或管道断开错误，则返回 -2
            if (errno == ECONNRESET || errno == EPIPE) return -2;
#endif
        }
        // 格式化错误信息到 errbuf 中，返回 -1
        snprintf(errbuf, errbuflen, "SSL_write(): %s",
            ERR_error_string(ERR_get_error(), NULL));
        return -1;
    }
}

// 返回读取的字节数，如果发生系统错误返回 -1，如果发生 SSL 错误返回 -2
int ssl_recv(SSL *ssl, char *buffer, int size, char *errbuf, size_t errbuflen)
{
    // 读取 SSL 连接中的数据
    int status = SSL_read(ssl, buffer, size);
    if (status <= 0)
    {
        // 获取 SSL 错误码
        int ssl_err = SSL_get_error(ssl, status);
        if (ssl_err == SSL_ERROR_ZERO_RETURN)
        {
            // SSL 连接关闭，返回 0
            return 0;
        }
        else if (ssl_err == SSL_ERROR_SYSCALL)
        {
            // 发生系统错误，返回 -1
            return -1;
        }
        else
        {
            // 不应该发生的情况，格式化错误信息到 errbuf 中，返回 -2
            snprintf(errbuf, errbuflen, "SSL_read(): %s",
                ERR_error_string(ERR_get_error(), NULL));
            return -2;
        }
    }
    else
    {
        // 返回读取的字节数
        return status;
    }
}

#endif // HAVE_OPENSSL
```