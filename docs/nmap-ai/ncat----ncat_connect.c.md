# `nmap\ncat\ncat_connect.c`

```cpp
/* $Id$ */

#include "base64.h"  // 包含 base64 相关的头文件
#include "nsock.h"  // 包含 nsock 相关的头文件
#include "ncat.h"  // 包含 ncat 相关的头文件
#include "util.h"  // 包含 util 相关的头文件
#include "sys_wrap.h"  // 包含 sys_wrap 相关的头文件

#include "nbase.h"  // 包含 nbase 相关的头文件
#include "http.h"  // 包含 http 相关的头文件

#ifndef WIN32
#include <unistd.h>  // 包含 POSIX 标准的头文件
#include <netdb.h>  // 包含网络数据库操作的头文件
#endif
#include <stddef.h>  // 包含标准定义的头文件
#include <stdlib.h>  // 包含标准库的头文件
#include <string.h>  // 包含字符串操作的头文件
#include <stdio.h>  // 包含标准输入输出的头文件

#ifdef HAVE_OPENSSL
#include <openssl/ssl.h>  // 包含 OpenSSL 的 SSL 协议相关的头文件
#include <openssl/err.h>  // 包含 OpenSSL 的错误处理相关的头文件

/* Deprecated in OpenSSL 3.0 */
#if OPENSSL_VERSION_NUMBER >= 0x30000000L
# define SSL_get_peer_certificate SSL_get1_peer_certificate  // 定义 SSL_get_peer_certificate 为 SSL_get1_peer_certificate
#endif
#endif

#ifdef WIN32
/* Define missing constant for shutdown(2).
 * See:
 * http://msdn.microsoft.com/en-us/library/windows/desktop/ms740481%28v=vs.85%29.aspx
 */
#define SHUT_WR SD_SEND  // 定义 SHUT_WR 为 SD_SEND
#endif

struct conn_state {
    nsock_iod sock_nsi;  // 定义网络套接字的状态
    nsock_iod stdin_nsi;  // 定义标准输入的状态
    nsock_event_id idle_timer_event_id;  // 定义空闲定时器事件的 ID
    int crlf_state;  // 定义回车换行的状态
};

static struct conn_state cs = {
    NULL,  // 初始化网络套接字状态为 NULL
    NULL,  // 初始化标准输入状态为 NULL
    0,  // 初始化空闲定时器事件 ID 为 0
    0  // 初始化回车换行状态为 0
};

static void try_nsock_connect(nsock_pool nsp, struct sockaddr_list *conn_addr);  // 定义尝试 nsock 连接的函数
static void connect_handler(nsock_pool nsp, nsock_event evt, void *data);  // 定义连接处理程序的函数
static void post_connect(nsock_pool nsp, nsock_iod iod);  // 定义连接后处理的函数
static void read_stdin_handler(nsock_pool nsp, nsock_event evt, void *data);  // 定义读取标准输入处理程序的函数
static void read_socket_handler(nsock_pool nsp, nsock_event evt, void *data);  // 定义读取套接字处理程序的函数
static void write_socket_handler(nsock_pool nsp, nsock_event evt, void *data);  // 定义写入套接字处理程序的函数
static void idle_timer_handler(nsock_pool nsp, nsock_event evt, void *data);  // 定义空闲定时器处理程序的函数
static void refresh_idle_timer(nsock_pool nsp);  // 定义刷新空闲定时器的函数

#ifdef HAVE_OPENSSL
/* This callback is called for every certificate in a chain. ok is true if
   OpenSSL's internal verification has verified the certificate. We don't change
   anything about the verification, we only need access to the certificates to
   provide diagnostics. */
static int verify_callback(int ok, X509_STORE_CTX *store)  // 定义验证回调函数
{
    X509 *cert = X509_STORE_CTX_get_current_cert(store);  // 获取当前证书
    int err = X509_STORE_CTX_get_error(store);  // 获取错误码
    /* 根据详细程度打印主题、颁发者和指纹。 */
    if ((!ok && o.verbose) || o.debug > 1) {
        char digest_buf[SHA1_STRING_LENGTH + 1];
        char *fp;

        loguser("Subject: ");
        X509_NAME_print_ex_fp(stderr, X509_get_subject_name(cert), 0, XN_FLAG_COMPAT);
        loguser_noprefix("\n");
        loguser("Issuer: ");
        X509_NAME_print_ex_fp(stderr, X509_get_issuer_name(cert), 0, XN_FLAG_COMPAT);
        loguser_noprefix("\n");

        fp = ssl_cert_fp_str_sha1(cert, digest_buf, sizeof(digest_buf));
        ncat_assert(fp == digest_buf);
        loguser("SHA-1 fingerprint: %s\n", digest_buf);
    }

    if (!ok && o.verbose) {
        loguser("Certificate verification failed (%s).\n",
            X509_verify_cert_error_string(err));
    }

    return ok;
}
# 设置 SSL 上下文的选项
static void set_ssl_ctx_options(SSL_CTX *ctx)
{
    # 如果未指定 SSL 信任文件，则加载默认的 CA 证书
    if (o.ssltrustfile == NULL) {
        ssl_load_default_ca_certs(ctx);
    } else {
        # 如果指定了 SSL 信任文件，并且开启了调试模式，则记录使用的信任 CA 证书文件
        if (o.debug)
            logdebug("Using trusted CA certificates from %s.\n", o.ssltrustfile);
        # 加载指定的信任 CA 证书文件
        if (SSL_CTX_load_verify_locations(ctx, o.ssltrustfile, NULL) != 1) {
            bye("Could not load trusted certificates from %s.\n%s",
                o.ssltrustfile, ERR_error_string(ERR_get_error(), NULL));
        }
    }

    # 如果开启了 SSL 验证，则设置 SSL 上下文的验证选项
    if (o.sslverify) {
        SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER, verify_callback);
    } else {
        # 否则，仍然检查验证状态并报告
        SSL_CTX_set_verify(ctx, SSL_VERIFY_NONE, verify_callback);
        # 如果开启了 SSL 并且开启了调试模式，则记录不进行证书验证
        if (o.ssl && o.debug)
            logdebug("Not doing certificate verification.\n");
    }

    # 如果指定了 SSL 证书和私钥文件，则加载它们
    if (o.sslcert != NULL && o.sslkey != NULL) {
        if (SSL_CTX_use_certificate_file(ctx, o.sslcert, SSL_FILETYPE_PEM) != 1)
            bye("SSL_CTX_use_certificate_file(): %s.", ERR_error_string(ERR_get_error(), NULL));
        if (SSL_CTX_use_PrivateKey_file(ctx, o.sslkey, SSL_FILETYPE_PEM) != 1)
            bye("SSL_CTX_use_Privatekey_file(): %s.", ERR_error_string(ERR_get_error(), NULL));
    } else {
        # 否则，如果 SSL 证书和私钥文件不完整，则报错
        if ((o.sslcert == NULL) != (o.sslkey == NULL))
            bye("The --ssl-key and --ssl-cert options must be used together.");
    }
    # 如果未指定 SSL 密码套件，则设置默认的 OpenSSL 密码套件
    if (o.sslciphers == NULL) {
      if (!SSL_CTX_set_cipher_list(ctx, "ALL:!aNULL:!eNULL:!LOW:!EXP:!RC4:!MD5:@STRENGTH"))
        bye("Unable to set OpenSSL cipher list: %s", ERR_error_string(ERR_get_error(), NULL));
    }
    else {
      # 否则，设置指定的 SSL 密码套件
      printf("setting ciphers: %s\n", o.sslciphers);
      if (!SSL_CTX_set_cipher_list(ctx, o.sslciphers))
        bye("Unable to set OpenSSL cipher list: %s", ERR_error_string(ERR_get_error(), NULL));
    }

    # 如果支持 ALPN，则进行相应处理
#ifdef HAVE_ALPN_SUPPORT
    # 如果存在 SSL ALPN 字符串
    if (o.sslalpn) {
        # 定义 ALPN 长度和 ALPN 字节流
        size_t alpn_len;
        unsigned char *alpn = next_protos_parse(&alpn_len, o.sslalpn);

        # 如果无法解析 ALPN 字符串，则退出并打印错误信息
        if (alpn == NULL)
            bye("Could not parse ALPN string");

        # 如果开启了调试模式，则打印 ALPN 字符串
        if (o.debug)
            logdebug("Using ALPN String %s\n", o.sslalpn);

        # 设置 SSL 上下文的 ALPN 协议，如果失败则释放内存并退出并打印错误信息
        if (SSL_CTX_set_alpn_protos(ctx, alpn, alpn_len) != 0){
            free(alpn);
            bye("SSL_CTX_set_alpn_protos: %s.", ERR_error_string(ERR_get_error(), NULL));
        }

        # 释放 ALPN 字节流内存
        free(alpn);
    }
#endif
}
#endif

/* 根据详细程度，打印连接建立的消息。 */
static void connect_report(nsock_iod nsi)
{
    union sockaddr_u peer;
    zmem(&peer, sizeof(peer.storage));

    nsock_iod_get_communication_info(nsi, NULL, NULL, NULL, &peer.sockaddr,
                                     sizeof(peer.storage));
    if (o.verbose) {
        char peer_str[INET6_ADDRSTRLEN + sizeof(union sockaddr_u)] = {0};
        if (o.proxytype) {
            Snprintf(peer_str, sizeof(peer_str), "%s:%u", o.target, o.portno);
        }
        else {
            Strncpy(peer_str, socktop(&peer, 0), sizeof(peer_str));
        }
#ifdef HAVE_OPENSSL
        if (nsock_iod_check_ssl(nsi)) {
            X509 *cert;
            X509_NAME *subject;
            char digest_buf[SHA1_STRING_LENGTH + 1];
            char *fp;

            loguser("SSL connection to %s.", peer_str);

            cert = SSL_get_peer_certificate((SSL *)nsock_iod_get_ssl(nsi));
            ncat_assert(cert != NULL);

            subject = X509_get_subject_name(cert);
            if (subject != NULL) {
                char buf[256];
                int n;

                n = X509_NAME_get_text_by_NID(subject, NID_organizationName, buf, sizeof(buf));
                if (n >= 0 && n <= sizeof(buf) - 1)
                    loguser_noprefix(" %s", buf);
            }

            loguser_noprefix("\n");

            fp = ssl_cert_fp_str_sha1(cert, digest_buf, sizeof(digest_buf));
            ncat_assert(fp == digest_buf);
            loguser("SHA-1 fingerprint: %s\n", digest_buf);
        } else
#endif
        {
            loguser("Connected to %s.\n", peer_str);
        }
    }
}

/* 类似于inet_socktop，但它将IPv6地址放在方括号中。 */
static const char *sock_to_url(char *host_str, unsigned short port)
{
    static char buf[512];
    # 根据主机地址字符串获取地址族，并根据不同的地址族格式化主机地址和端口号
    switch(getaddrfamily(host_str)) {
       # 如果地址族为-1或1，使用普通格式化方式
       case -1:
       case 1:
           Snprintf(buf, sizeof(buf), "%s:%hu", host_str, port);
           break;
       # 如果地址族为2，使用带有方括号的格式化方式
       case 2:
           Snprintf(buf, sizeof(buf), "[%s]:%hu", host_str, port);
    }
    # 返回格式化后的主机地址和端口号
    return buf;
# 在连接请求行中追加 CONNECT 方法和目标主机的 HTTP 版本信息
static int append_connect_request_line(char **buf, size_t *size, size_t *offset,
    char* host_str,unsigned short port)
{
    return strbuf_sprintf(buf, size, offset, "CONNECT %s HTTP/1.0\r\n",
        sock_to_url(host_str,port));
}

# 生成 HTTP 连接请求的字符串
static char *http_connect_request(char* host_str, unsigned short port, int *n)
{
    char *buf = NULL;
    size_t size = 0, offset = 0;

    # 调用函数追加连接请求行
    append_connect_request_line(&buf, &size, &offset, host_str, port);
    # 追加换行符
    strbuf_append_str(&buf, &size, &offset, "\r\n");
    # 将请求字符串的长度赋值给 n
    *n = offset;

    return buf;
}

# 生成带有代理认证信息的 HTTP 连接请求的字符串
static char *http_connect_request_auth(char* host_str, unsigned short port, int *n,
    struct http_challenge *challenge)
{
    char *buf = NULL;
    size_t size = 0, offset = 0;

    # 调用函数追加连接请求行
    append_connect_request_line(&buf, &size, &offset, host_str, port);
    # 追加代理认证信息
    strbuf_append_str(&buf, &size, &offset, "Proxy-Authorization:");
    if (challenge->scheme == AUTH_BASIC) {
        char *auth_str;

        # 对代理认证信息进行 Base64 编码
        auth_str = b64enc((unsigned char *) o.proxy_auth, strlen(o.proxy_auth));
        # 格式化追加认证信息到请求字符串
        strbuf_sprintf(&buf, &size, &offset, " Basic %s\r\n", auth_str);
        free(auth_str);
#if HAVE_HTTP_DIGEST
    } else if (challenge->scheme == AUTH_DIGEST) {
        char *proxy_auth;
        char *username, *password;
        char *response_hdr;

        # 分割代理认证参数
        proxy_auth = Strdup(o.proxy_auth);
        username = proxy_auth;
        password = strchr(proxy_auth, ':');
        if (password == NULL) {
            free(proxy_auth);
            return NULL;
        }
        *password++ = '\0';
        # 生成 HTTP 摘要认证信息并追加到请求字符串
        response_hdr = http_digest_proxy_authorization(challenge,
            username, password, "CONNECT", sock_to_url(o.target,o.portno));
        if (response_hdr == NULL) {
            free(proxy_auth);
            return NULL;
        }
        strbuf_append_str(&buf, &size, &offset, response_hdr);
        free(proxy_auth);
        free(response_hdr);
#endif
    } else {
        bye("Unknown authentication type.");
    }
    # 将"\r\n"字符串追加到strbuf中
    strbuf_append_str(&buf, &size, &offset, "\r\n");
    # 将offset的值赋给n指针所指向的变量
    *n = offset;
    # 返回buf作为函数的结果
    return buf;
# 返回经过代理协商后可用的套接字描述符，如果出现任何错误则返回-1。如果在协商后通过代理接收到任何字节，将它们写入标准输出。
static int do_proxy_http(void)
{
    struct socket_buffer sockbuf;  # 定义套接字缓冲区结构体
    char *request;  # 定义请求字符串指针
    char *status_line, *header;  # 定义状态行和头部字符串指针
    char *remainder;  # 定义剩余字符串指针
    size_t len;  # 定义长度
    int sd, code;  # 定义套接字描述符和代码
    int n;  # 定义整型变量n
    char *target;  # 定义目标字符串指针
    union sockaddr_u addr;  # 定义联合体地址结构
    size_t sslen;  # 定义长度
    char addrstr[INET6_ADDRSTRLEN];  # 定义地址字符串数组

    request = NULL;  # 初始化请求字符串指针为空
    status_line = NULL;  # 初始化状态行字符串指针为空
    header = NULL;  # 初始化头部字符串指针为空

    sd = do_connect(SOCK_STREAM);  # 调用do_connect函数，返回套接字描述符
    if (sd == -1) {  # 如果套接字描述符为-1
        loguser("Proxy connection failed: %s.\n", socket_strerror(socket_errno()));  # 记录用户日志，输出代理连接失败的错误信息
        return -1;  # 返回-1
    }

    if (proxyresolve(o.target, 0, &addr.storage, &sslen, o.af)) {  # 如果代理解析失败
        /* target resolution has failed, possibly because it is disabled */
        if (!(o.proxydns & PROXYDNS_REMOTE)) {  # 如果代理DNS不是远程
            loguser("Error: Failed to resolve host %s locally.\n", o.target);  # 记录用户日志，输出本地解析主机失败的错误信息
            goto bail;  # 跳转到bail标签
        }
        if (o.verbose)  # 如果是冗长模式
            loguser("Host %s will be resolved by the proxy.\n", o.target);  # 记录用户日志，输出主机将由代理解析的信息
        target = o.target;  # 目标为o.target
    } else {  # 否则
        /* addr is now populated with either sockaddr_in or sockaddr_in6 */
        Strncpy(addrstr, inet_socktop(&addr), sizeof(addrstr));  # 将地址转换为字符串
        target = addrstr;  # 目标为地址字符串
        if (o.verbose && getaddrfamily(o.target) == -1)  # 如果是冗长模式且获取地址家族失败
            loguser("Host %s locally resolved to %s.\n", o.target, target);  # 记录用户日志，输出本地解析主机到目标的信息
    }

    /* First try a request with no authentication. */
    request = http_connect_request(target, o.portno, &n);  # 生成HTTP连接请求
    if (send(sd, request, n, 0) < 0) {  # 如果发送请求失败
        loguser("Error sending proxy request: %s.\n", socket_strerror(socket_errno()));  # 记录用户日志，输出发送代理请求失败的错误信息
        goto bail;  # 跳转到bail标签
    }
    free(request);  # 释放请求字符串指针
    request = NULL;  # 请求字符串指针置空

    socket_buffer_init(&sockbuf, sd);  # 初始化套接字缓冲区

    if (http_read_status_line(&sockbuf, &status_line) != 0) {  # 如果读取状态行失败
        loguser("Error reading proxy response Status-Line.\n");  # 记录用户日志，输出读取代理响应状态行失败的错误信息
        goto bail;  # 跳转到bail标签
    }
    code = http_parse_status_line_code(status_line);  # 解析状态行代码
    # 如果调试模式开启，记录代理返回的状态码
    if (o.debug)
      logdebug("Proxy returned status code %d.\n", code);
    # 释放 status_line 指针指向的内存，并将其置为 NULL
    free(status_line);
    status_line = NULL;
    # 如果读取代理响应头部出现错误，记录错误信息并跳转到标签 bail
    if (http_read_header(&sockbuf, &header) != 0) {
        loguser("Error reading proxy response header.\n");
        goto bail;
    }
    # 如果状态码为407且代理授权不为空
    if (code == 407 && o.proxy_auth != NULL) {
        # 声明一个指向http_header结构的指针h，一个http_challenge结构的challenge
        struct http_header *h;
        struct http_challenge challenge;

        # 关闭当前的套接字连接
        close(sd);
        sd = -1;

        # 如果解析代理响应头部出错
        if (http_parse_header(&h, header) != 0) {
            loguser("Error parsing proxy response header.\n");
            # 跳转到bail标签处
            goto bail;
        }
        # 释放header指向的内存
        free(header);
        header = NULL;

        # 如果获取Proxy-Authenticate挑战出错
        if (http_header_get_proxy_challenge(h, &challenge) == NULL) {
            loguser("Error getting Proxy-Authenticate challenge.\n");
            # 释放h指向的内存
            http_header_free(h);
            # 跳转到bail标签处
            goto bail;
        }
        # 释放h指向的内存
        http_header_free(h);

        # 重新建立一个TCP连接
        sd = do_connect(SOCK_STREAM);
        # 如果连接失败
        if (sd == -1) {
            loguser("Proxy reconnection failed: %s.\n", socket_strerror(socket_errno()));
            # 跳转到bail标签处
            goto bail;
        }

        # 构建带有授权信息的连接请求
        request = http_connect_request_auth(target, o.portno, &n, &challenge);
        # 如果构建请求头部出错
        if (request == NULL) {
            loguser("Error building Proxy-Authorization header.\n");
            # 释放challenge指向的内存
            http_challenge_free(&challenge);
            # 跳转到bail标签处
            goto bail;
        }
        # 如果调试模式开启，记录重新连接的请求头部
        if (o.debug)
          logdebug("Reconnection header:\n%s", request);
        # 如果发送请求失败
        if (send(sd, request, n, 0) < 0) {
            loguser("Error sending proxy request: %s.\n", socket_strerror(socket_errno()));
            # 释放challenge指向的内存
            http_challenge_free(&challenge);
            # 跳转到bail标签处
            goto bail;
        }
        # 释放request指向的内存
        free(request);
        request = NULL;
        # 释放challenge指向的内存
        http_challenge_free(&challenge);

        # 初始化套接字缓冲区
        socket_buffer_init(&sockbuf, sd);

        # 如果读取代理响应状态行出错
        if (http_read_status_line(&sockbuf, &status_line) != 0) {
            loguser("Error reading proxy response Status-Line.\n");
            # 跳转到bail标签处
            goto bail;
        }
        # 解析状态行中的状态码
        code = http_parse_status_line_code(status_line);
        # 如果调试模式开启，记录代理返回的状态码
        if (o.debug)
          logdebug("Proxy returned status code %d.\n", code);
        # 释放status_line指向的内存
        free(status_line);
        status_line = NULL;
        # 如果读取代理响应头部出错
        if (http_read_header(&sockbuf, &header) != 0) {
            loguser("Error reading proxy response header.\n");
            # 跳转到bail标签处
            goto bail;
        }
    }
    # 如果状态码不等于200，则记录日志并跳转到bail标签处
    if (code != 200) {
        loguser("Proxy returned status code %d.\n", code);
        goto bail;
    }

    # 释放header指针所指向的内存，并将其置为NULL
    free(header);
    header = NULL;

    # 获取socket_buffer_remainder函数返回的剩余数据和长度
    remainder = socket_buffer_remainder(&sockbuf, &len);
    # 将剩余数据写入标准输出
    Write(STDOUT_FILENO, remainder, len);

    # 返回sd变量的值
    return sd;
# 定义一个标签，用于在函数中跳转到此处进行错误处理
bail:
    # 如果套接字描述符不为-1，则关闭套接字
    if (sd != -1)
        close(sd);
    # 如果请求不为空，则释放请求内存
    if (request != NULL)
        free(request);
    # 如果状态行不为空，则释放状态行内存
    if (status_line != NULL)
        free(status_line);
    # 如果头部不为空，则释放头部内存
    if (header != NULL)
        free(header);
    # 返回-1，表示发生错误
    return -1;
}

# SOCKS4a 支持
# 在代理协商后返回一个可用的套接字描述符，如果发生任何错误则返回-1
static int do_proxy_socks4(void)
{
    # 定义一个长度为8的字符数组，用于存储 SOCKS4 消息
    char socksbuf[8];
    # 定义一个 SOCKS4 数据结构体
    struct socks4_data socks4msg;
    # 定义数据长度变量
    size_t datalen;
    # 如果代理认证不为空，则使用代理认证，否则使用空字符串
    char *username = o.proxy_auth != NULL ? o.proxy_auth : "";
    # 定义一个地址联合体
    union sockaddr_u addr;
    # 定义地址长度变量
    size_t sslen;
    # 定义套接字描述符
    int sd;

    # 如果目标地址的地址族为2（IPv6），则打印错误信息并返回-1
    if (getaddrfamily(o.target) == 2) {
        loguser("Error: IPv6 addresses are not supported with Socks4.\n");
        return -1;
    }

    # 进行 SOCK_STREAM 类型的连接，并将返回的套接字描述符赋值给 sd
    sd = do_connect(SOCK_STREAM);
    # 如果连接失败，则打印错误信息并返回连接失败的套接字描述符
    if (sd == -1) {
        loguser("Proxy connection failed: %s.\n", socket_strerror(socket_errno()));
        return sd;
    }

    # 如果设置了详细输出，则打印连接到代理的信息
    if (o.verbose) {
        loguser("Connected to proxy %s:%hu\n", inet_socktop(&targetaddrs->addr),
            inet_port(&targetaddrs->addr));
    }

    # 填充 SOCKS4 数据结构体
    zmem(&socks4msg, sizeof(socks4msg));
    socks4msg.version = SOCKS4_VERSION;
    socks4msg.type = SOCKS_CONNECT;
    socks4msg.port = htons(o.portno);

    # 如果用户名长度超过 SOCKS4 消息中的数据部分长度，则打印错误信息，关闭套接字并返回-1
    if (strlen(username) >= sizeof(socks4msg.data)) {
        loguser("Error: username is too long.\n");
        close(sd);
        return -1;
    }
    # 将用户名复制到 SOCKS4 消息的数据部分，并更新数据长度
    strcpy(socks4msg.data, username);
    datalen = strlen(username) + 1;
    # 如果目标地址解析失败，可能是因为被禁用了
    if (proxyresolve(o.target, 0, &addr.storage, &sslen, AF_INET)) {
        # 如果未使用远程代理DNS，则记录错误信息并关闭连接
        if (!(o.proxydns & PROXYDNS_REMOTE)) {
            loguser("Error: Failed to resolve host %s locally.\n", o.target);
            close(sd);
            return -1;
        }
        # 如果启用了详细日志，则记录目标地址将由代理解析
        if (o.verbose)
            loguser("Host %s will be resolved by the proxy.\n", o.target);
        # 设置 SOCKS4 消息的地址为 0.0.0.1
        socks4msg.address = inet_addr("0.0.0.1");
        # 如果数据长度加上目标地址长度超过了消息数据的大小，则记录错误信息并关闭连接
        if (datalen + strlen(o.target) >= sizeof(socks4msg.data)) {
            loguser("Error: host name is too long.\n");
            close(sd);
            return -1;
        }
        # 将目标地址拷贝到消息数据中
        strcpy(socks4msg.data + datalen, o.target);
        datalen += strlen(o.target) + 1;
    } else {
        # 如果目标地址解析成功，则将 SOCKS4 消息的地址设置为解析得到的地址
        socks4msg.address = addr.in.sin_addr.s_addr;
        # 如果启用了详细日志且目标地址的地址族为 -1，则记录目标地址的本地解析结果
        if (o.verbose && getaddrfamily(o.target) == -1)
            loguser("Host %s locally resolved to %s.\n", o.target,
                inet_socktop(&addr));
    }

    # 发送 SOCKS4 请求消息，消息长度为消息头加上数据长度
    if (send(sd, (char *)&socks4msg, offsetof(struct socks4_data, data) + datalen, 0) < 0) {
        # 记录发送代理请求时的错误信息，并关闭连接
        loguser("Error: sending proxy request: %s.\n", socket_strerror(socket_errno()));
        close(sd);
        return -1;
    }

    # 从缓冲区中接收 8 字节的 SOCKS4 响应消息
    if (recv(sd, socksbuf, 8, 0) < 0) {
        # 记录从代理接收到的响应消息过短的错误信息，并关闭连接
        loguser("Error: short response from proxy.\n");
        close(sd);
        return -1;
    }

    # 如果连接未关闭且 SOCKS4 响应消息的第二个字节不是连接成功的标志，则记录代理连接失败的错误信息，并关闭连接
    if (sd != -1 && socksbuf[1] != SOCKS4_CONN_ACC) {
        loguser("Proxy connection failed.\n");
        close(sd);
        return -1;
    }

    # 返回连接描述符
    return sd;
/* SOCKS5 support
 * Return a usable socket descriptor after
 * proxy negotiation, or -1 on any error.
 */
static int do_proxy_socks5(void)
{
    // 定义 SOCKS5 连接消息结构体
    struct socks5_connect socks5msg;
    // 将代理端口号转换为网络字节顺序
    uint16_t proxyport = htons(o.portno);
    // 定义 SOCKS5 请求消息缓冲区
    char socksbuf[4];
    // 定义套接字描述符
    int sd;
    // 目标地址长度、目标地址长度、目标地址长度、目标地址长度
    size_t dstlen, targetlen;
    // 定义 SOCKS5 请求消息结构体
    struct socks5_request socks5msg2;
    // 定义 SOCKS5 认证消息结构体
    struct socks5_auth socks5auth;
    // 用户名指针、密码指针
    char *uptr, *pptr;
    // 认证消息长度、用户名长度、密码长度
    size_t authlen, ulen, plen;
    // 地址联合体
    union sockaddr_u addr;
    // 地址缓冲区长度
    size_t sslen;
    // 地址缓冲区
    void *addrbuf;
    // 地址长度
    size_t addrlen;
    // 绑定地址长度
    size_t bndaddrlen;
    // 绑定地址缓冲区
    char bndaddr[SOCKS5_DST_MAXLEN + 2]; /* IPv4/IPv6/hostname and port */

    // 进行 SOCK_STREAM 类型的连接
    sd = do_connect(SOCK_STREAM);
    // 如果连接失败，则记录错误信息并返回 -1
    if (sd == -1) {
        loguser("Proxy connection failed: %s.\n", socket_strerror(socket_errno()));
        return sd;
    }

    // 如果设置了详细模式，则记录连接到代理的信息
    if (o.verbose) {
        loguser("Connected to proxy %s:%hu\n", inet_socktop(&targetaddrs->addr),
            inet_port(&targetaddrs->addr));
    }

    // 清空 SOCKS5 连接消息结构体
    zmem(&socks5msg,sizeof(socks5msg));
    // 设置 SOCKS5 协议版本号
    socks5msg.ver = SOCKS5_VERSION;
    // 设置认证方法数量为 0
    socks5msg.nmethods = 0;
    // 设置认证方法为无认证
    socks5msg.methods[socks5msg.nmethods++] = SOCKS5_AUTH_NONE;

    // 如果设置了代理认证，则添加用户名密码认证方法
    if (o.proxy_auth)
        socks5msg.methods[socks5msg.nmethods++] = SOCKS5_AUTH_USERPASS;

    // 发送代理请求消息
    if (send(sd, (char *)&socks5msg, offsetof(struct socks5_connect, methods) + socks5msg.nmethods, 0) < 0) {
        loguser("Error: proxy request: %s.\n", socket_strerror(socket_errno()));
        close(sd);
        return -1;
    }

    // 接收连接响应消息，长度为两个字节
    if (recv(sd, socksbuf, 2, 0) < 0) {
        loguser("Error: malformed connect response from proxy.\n");
        close(sd);
        return -1;
    }

    // 检查连接响应消息的 SOCKS 版本号
    if (socksbuf[0] != SOCKS5_VERSION) {
        loguser("Error: wrong SOCKS version in connect response.\n");
        close(sd);
        return -1;
    }

    // 清空 SOCKS5 请求消息结构体
    zmem(&socks5msg2,sizeof(socks5msg2));
    // 设置 SOCKS5 协议版本号
    socks5msg2.ver = SOCKS5_VERSION;
    // 设置命令为连接
    socks5msg2.cmd = SOCKS_CONNECT;
    // 保留字段为 0
    socks5msg2.rsv = 0;
    # 如果目标地址解析失败，可能是因为被禁用了
    if (proxyresolve(o.target, 0, &addr.storage, &sslen, o.af)) {
        # 如果未使用远程代理DNS，则记录错误信息并关闭连接
        if (!(o.proxydns & PROXYDNS_REMOTE)) {
            loguser("Error: Failed to resolve host %s locally.\n", o.target);
            close(sd);
            return -1;
        }
        # 如果启用了详细日志，则记录目标地址将由代理解析
        if (o.verbose)
            loguser("Host %s will be resolved by the proxy.\n", o.target);
        # 设置 SOCKS5 消息的地址类型为名称
        socks5msg2.atyp = SOCKS5_ATYP_NAME;
        # 计算目标地址的长度
        targetlen = strlen(o.target);
        # 如果目标地址长度超过最大长度限制，则记录错误信息并关闭连接
        if (targetlen > SOCKS5_DST_MAXLEN){
            loguser("Error: hostname length exceeds %d.\n", SOCKS5_DST_MAXLEN);
            close(sd);
            return -1;
        }
        dstlen = 0;
        # 将目标地址长度和地址数据拷贝到 SOCKS5 消息中
        socks5msg2.dst[dstlen++] = targetlen;
        memcpy(socks5msg2.dst + dstlen, o.target, targetlen);
        dstlen += targetlen;
    } else {
        # 如果地址已经解析为 sockaddr_in 或 sockaddr_in6
        switch (addr.sockaddr.sa_family) {
            # 如果是 IPv4 地址，则设置 SOCKS5 消息的地址类型为 IPv4
            case AF_INET:
                socks5msg2.atyp = SOCKS5_ATYP_IPv4;
                addrbuf = &addr.in.sin_addr;
                addrlen = 4;
                break;
            # 如果是 IPv6 地址，则设置 SOCKS5 消息的地址类型为 IPv6
            case AF_INET6:
                socks5msg2.atyp = SOCKS5_ATYP_IPv6;
                addrbuf = &addr.in6.sin6_addr;
                addrlen = 16;
                break;
            # 如果地址类型不是 IPv4 或 IPv6，则断言失败
            default:
                ncat_assert(0);
        }
        # 将地址数据拷贝到 SOCKS5 消息中
        memcpy(socks5msg2.dst, addrbuf, addrlen);
        dstlen = addrlen;
        # 如果启用了详细日志且目标地址的地址族为 -1，则记录本地解析的信息
        if (o.verbose && getaddrfamily(o.target) == -1)
            loguser("Host %s locally resolved to %s.\n", o.target,
                inet_socktop(&addr));
    }

    # 将代理端口拷贝到 SOCKS5 消息中
    memcpy(socks5msg2.dst + dstlen, &proxyport, 2);
    dstlen += 2;

    # 发送 SOCKS5 消息到代理服务器
    if (send(sd, (char *) &socks5msg2, offsetof(struct socks5_request , dst) + dstlen, 0) < 0) {
        loguser("Error: sending proxy request: %s.\n", socket_strerror(socket_errno()));
        close(sd);
        return -1;
    }
    # 如果接收到的数据小于0，表示接收出错，记录错误信息并关闭连接，返回-1
    if (recv(sd, socksbuf, 4, 0) < 0) {
        loguser("Error: malformed request response from proxy.\n");
        close(sd);
        return -1;
    }

    # 如果接收到的数据的第一个字节不是SOCKS5版本号，记录错误信息并关闭连接，返回-1
    if (socksbuf[0] != SOCKS5_VERSION) {
        loguser("Error: wrong SOCKS version in request response.\n");
        close(sd);
        return -1;
    }

    # 根据接收到的第二个字节进行不同的处理
    switch(socksbuf[1]) {
        case 0:
            # 如果是0，表示连接成功，记录信息
            if (o.verbose)
                loguser("connection succeeded.\n");
            break;
        case 1:
            # 如果是1，表示SOCKS服务器一般性错误，记录错误信息并关闭连接，返回-1
            loguser("Error: general SOCKS server failure.\n");
            close(sd);
            return -1;
        case 2:
            # 如果是2，表示连接不被规则集允许，记录错误信息并关闭连接，返回-1
            loguser("Error: connection not allowed by ruleset.\n");
            close(sd);
            return -1;
        case 3:
            # 如果是3，表示网络不可达，记录错误信息并关闭连接，返回-1
            loguser("Error: Network unreachable.\n");
            close(sd);
            return -1;
        case 4:
            # 如果是4，表示主机不可达，记录错误信息并关闭连接，返回-1
            loguser("Error: Host unreachable.\n");
            close(sd);
            return -1;
        case 5:
            # 如果是5，表示连接被拒绝，记录错误信息并关闭连接，返回-1
            loguser("Error: Connection refused.\n");
            close(sd);
            return -1;
        case 6:
            # 如果是6，表示TTL过期，记录错误信息并关闭连接，返回-1
            loguser("Error: TTL expired.\n");
            close(sd);
            return -1;
        case 7:
            # 如果是7，表示命令不被支持，记录错误信息并关闭连接，返回-1
            loguser("Error: Command not supported.\n");
            close(sd);
            return -1;
        case 8:
            # 如果是8，表示地址类型不被支持，记录错误信息并关闭连接，返回-1
            loguser("Error: Address type not supported.\n");
            close(sd);
            return -1;
        default:
            # 如果是其他值，表示回复中有未分配的值，记录错误信息并关闭连接，返回-1
            loguser("Error: unassigned value in the reply.\n");
            close(sd);
            return -1;
    }

    # 根据接收到的第四个字节进行不同的处理
    switch (socksbuf[3]) {
    case SOCKS5_ATYP_IPv4:
        # 如果是IPv4地址类型，绑定地址长度为4+2
        bndaddrlen = 4 + 2;
        break;
    case SOCKS5_ATYP_IPv6:
        # 如果是IPv6地址类型，绑定地址长度为16+2
        bndaddrlen = 16 + 2;
        break;
    case SOCKS5_ATYP_NAME:
        # 如果是域名地址类型，接收下一个字节，计算绑定地址长度
        if (recv(sd, socksbuf, 1, 0) < 0) {
            loguser("Error: malformed request response from proxy.\n");
            close(sd);
            return -1;
        }
        bndaddrlen = (unsigned char)socksbuf[0] + 2;
        break;
    # 默认情况下，如果代理绑定地址类型无效，则记录错误信息并关闭套接字，返回-1
    default:
        loguser("Error: invalid proxy bind address type.\n");
        close(sd);
        return -1;
    }

    # 如果从代理接收的请求响应格式不正确，则记录错误信息并关闭套接字，返回-1
    if (recv(sd, bndaddr, bndaddrlen, 0) < 0) {
        loguser("Error: malformed request response from proxy.\n");
        close(sd);
        return -1;
    }

    # 返回套接字
    return(sd);
}

# 创建一个新的 nsock_iod 对象
static nsock_iod new_iod(nsock_pool mypool) {
   # 使用给定的 nsock_pool 创建一个新的 nsock_iod 对象
   nsock_iod nsi = nsock_iod_new(mypool, NULL);
   # 如果创建失败，则退出程序并打印错误信息
   if (nsi == NULL)
     bye("Failed to create nsock_iod.");
   # 设置 nsock_iod 对象的主机名
   if (nsock_iod_set_hostname(nsi, o.sslservername) == -1)
     bye("Failed to set hostname on iod.");

   # 根据源地址的协议类型进行不同的处理
   switch (srcaddr.storage.ss_family) {
     case AF_UNSPEC:
       break;
     case AF_INET:
       nsock_iod_set_localaddr(nsi, &srcaddr.storage,
           sizeof(srcaddr.in));
       break;
#ifdef AF_INET6
     case AF_INET6:
       nsock_iod_set_localaddr(nsi, &srcaddr.storage,
           sizeof(srcaddr.in6));
       break;
#endif
#if HAVE_SYS_UN_H
     case AF_UNIX:
       nsock_iod_set_localaddr(nsi, &srcaddr.storage,
           SUN_LEN((struct sockaddr_un *)&srcaddr.storage));
       break;
#endif
     default:
       nsock_iod_set_localaddr(nsi, &srcaddr.storage,
           sizeof(srcaddr.storage));
       break;
   }

   # 如果存在源路由信息，则设置 IP 选项
   if (o.numsrcrtes) {
     unsigned char *ipopts = NULL;
     size_t ipoptslen = 0;

     if (o.af != AF_INET)
       bye("Sorry, -g can only currently be used with IPv4.");
     ipopts = buildsrcrte(targetaddrs->addr.in.sin_addr, o.srcrtes, o.numsrcrtes, o.srcrteptr, &ipoptslen);

     nsock_iod_set_ipoptions(nsi, ipopts, ipoptslen);
     free(ipopts); /* Nsock has its own copy */
   }
   # 返回创建的 nsock_iod 对象
   return nsi;
}

# ncat 连接函数
int ncat_connect(void)
{
    nsock_pool mypool;
    int rc;

    # 如果未指定 nsock_engine，则使用默认的 "select" 引擎
    if (!o.nsock_engine)
        nsock_set_default_engine("select");

    # 创建一个 nsock_pool 对象
    if ((mypool = nsock_pool_new(NULL)) == NULL)
        bye("Failed to create nsock_pool.");

    # 根据调试级别设置 nsock 日志级别
    if (o.debug >= 6)
        nsock_set_loglevel(NSOCK_LOG_DBG_ALL);
    else if (o.debug >= 3)
        nsock_set_loglevel(NSOCK_LOG_DBG);
    else if (o.debug >= 1)
        nsock_set_loglevel(NSOCK_LOG_INFO);
    # 如果条件不成立，则设置日志级别为错误
    else
        nsock_set_loglevel(NSOCK_LOG_ERROR);

    # 允许连接到广播地址
    nsock_pool_set_broadcast(mypool, 1);
#ifdef HAVE_OPENSSL
#ifdef HAVE_DTLS_CLIENT_METHOD
    // 如果支持 OpenSSL 并且支持 DTLS 客户端方法
    if(o.proto == IPPROTO_UDP)
        // 如果协议是 UDP，则设置 SSL 上下文选项为 DTLS
        set_ssl_ctx_options((SSL_CTX *) nsock_pool_dtls_init(mypool, 0));
    else
#endif
        // 否则设置 SSL 上下文选项为普通 SSL
        set_ssl_ctx_options((SSL_CTX *) nsock_pool_ssl_init(mypool, 0));
#endif

    // 如果没有代理类型
    if (!o.proxytype) {
#if HAVE_SYS_UN_H
        /* For DGRAM UNIX socket we have to use source socket */
        // 对于 DGRAM UNIX 套接字，必须使用源套接字
        if (o.af == AF_UNIX && o.proto == IPPROTO_UDP)
        {
            // 如果地址族是 UNIX 并且协议是 UDP
            if (srcaddr.storage.ss_family != AF_UNIX) {
                char *tmp_name = NULL;
#if HAVE_MKSTEMP
              char *tmpdir = getenv("TMPDIR");
              size_t size=0, offset=0;
              // 根据环境变量 TMPDIR 或默认路径创建临时文件名
              strbuf_sprintf(&tmp_name, &size, &offset, "%s/ncat.XXXXXX",
                  tmpdir ? tmpdir : "/tmp");
              // 如果创建临时文件名失败，则退出程序
              if (mkstemp(tmp_name) == -1) {
                bye("Failed to create name for temporary DGRAM source Unix domain socket (mkstemp).");
              }
              // 删除临时文件
              unlink(tmp_name);
#else
                /* If no source socket was specified, we have to create temporary one. */
                // 如果没有指定源套接字，则创建临时套接字
                if ((tmp_name = tempnam(NULL, "ncat.")) == NULL)
                    bye("Failed to create name for temporary DGRAM source Unix domain socket (tempnam).");
#endif

                // 初始化源地址为临时文件名
                NCAT_INIT_SUN(&srcaddr, tmp_name);
                free (tmp_name);
            }

            // 如果启用了详细输出，则记录使用的源 DGRAM Unix 域套接字
            if (o.verbose)
                loguser("[%s] used as source DGRAM Unix domain socket.\n", srcaddr.un.sun_path);
        }
#endif
        /* A non-proxy connection. Create an iod for a new socket. */
        // 非代理连接。为新套接字创建一个 iod
        cs.sock_nsi = new_iod(mypool);
#if HAVE_SYS_UN_H
        # 如果系统支持 AF_UNIX 地址族
        if (o.af == AF_UNIX) {
            # 如果协议是 IPPROTO_UDP
            if (o.proto == IPPROTO_UDP) {
                # 使用 UNIX 套接字连接到目标地址，并设置数据报模式
                nsock_connect_unixsock_datagram(mypool, cs.sock_nsi, connect_handler, NULL,
                                                &targetaddrs->addr.sockaddr,
                                                SUN_LEN((struct sockaddr_un *)&targetaddrs->addr.sockaddr));
            } else {
                # 使用 UNIX 套接字连接到目标地址，并设置流模式
                nsock_connect_unixsock_stream(mypool, cs.sock_nsi, connect_handler, o.conntimeout,
                                              NULL, &targetaddrs->addr.sockaddr,
                                              SUN_LEN((struct sockaddr_un *)&targetaddrs->addr.sockaddr));
            }
        } else
#endif
        {
            # 将连接添加到第一个解析的地址
            try_nsock_connect(mypool, targetaddrs);
        }
    } else {
        /* 如果是代理连接。 */
        static int connect_socket;  // 声明静态变量，用于保存连接的套接字

        if (strcmp(o.proxytype, "http") == 0) {  // 如果代理类型是 HTTP
            connect_socket = do_proxy_http();  // 调用 HTTP 代理连接函数
        } else if (strcmp(o.proxytype, "socks4") == 0) {  // 如果代理类型是 SOCKS4
            connect_socket = do_proxy_socks4();  // 调用 SOCKS4 代理连接函数
        } else if (strcmp(o.proxytype, "socks5") == 0) {  // 如果代理类型是 SOCKS5
            connect_socket = do_proxy_socks5();  // 调用 SOCKS5 代理连接函数
        }

        if (connect_socket == -1)  // 如果连接套接字为-1，代表连接失败
        {
            nsock_pool_delete(mypool);  // 删除套接字池
            return 1;  // 返回1，代表连接失败
        }

        /* 一旦代理协商完成，Nsock 将控制套接字。 */
        cs.sock_nsi = nsock_iod_new2(mypool, connect_socket, NULL);  // 创建新的 Nsock IOD 对象
        if (nsock_iod_set_hostname(cs.sock_nsi, o.sslservername) == -1)  // 如果设置主机名失败
            bye("Failed to set hostname on iod.");  // 输出错误信息并退出程序
        if (o.ssl)  // 如果使用 SSL
        {
            /* connect_handler 创建 stdin_nsi 并调用 post_connect */
            nsock_reconnect_ssl(mypool, cs.sock_nsi, connect_handler, o.conntimeout, NULL, NULL);  // 重新连接并使用 SSL
        }
        else  // 如果不使用 SSL
        {
            /* 为 nsp->stdin 创建 IOD */
            if ((cs.stdin_nsi = nsock_iod_new2(mypool, 0, NULL)) == NULL)  // 如果创建 stdin IOD 失败
                bye("Failed to create stdin nsiod.");  // 输出错误信息并退出程序

            post_connect(mypool, cs.sock_nsi);  // 调用 post_connect 函数
        }
    }

    /* 连接 */
    rc = nsock_loop(mypool, -1);  // 运行套接字池中的事件循环，-1表示一直运行直到所有事件完成

    free_sockaddr_list(targetaddrs);  // 释放目标地址列表的内存

    if (o.verbose) {  // 如果设置了详细输出
        struct timeval end_time;  // 声明结束时间结构体
        double time;  // 声明时间变量
        gettimeofday(&end_time, NULL);  // 获取当前时间
        time = TIMEVAL_MSEC_SUBTRACT(end_time, start_time) / 1000.0;  // 计算时间差并转换为秒
        loguser("%lu bytes sent, %lu bytes received in %.2f seconds.\n",
            nsock_iod_get_write_count(cs.sock_nsi),
            nsock_iod_get_read_count(cs.sock_nsi), time);  // 输出发送和接收的字节数以及时间
    }
#if HAVE_SYS_UN_H
    # 如果系统支持 UNIX 域套接字
    if (o.af == AF_UNIX && o.proto == IPPROTO_UDP) {
        # 如果地址族是 AF_UNIX 并且协议是 IPPROTO_UDP
        if (o.verbose)
            # 如果设置了详细输出，记录删除源 DGRAM Unix 域套接字的操作
            loguser("Deleting source DGRAM Unix domain socket. [%s]\n", srcaddr.un.sun_path);
        # 删除源地址对应的 Unix 域套接字
        unlink(srcaddr.un.sun_path);
    }
#endif

    # 从套接字池中删除套接字
    nsock_pool_delete(mypool);

    # 如果返回值是 NSOCK_LOOP_ERROR，则返回 1，否则返回 0
    return rc == NSOCK_LOOP_ERROR ? 1 : 0;
}

static void try_nsock_connect(nsock_pool nsp, struct sockaddr_list *conn_addr)
{
#ifdef HAVE_OPENSSL
    # 如果支持 OpenSSL
    if (o.ssl) {
        # 进行 SSL 连接
        nsock_connect_ssl(nsp, cs.sock_nsi, connect_handler,
                          o.conntimeout, (void *)conn_addr->next,
                          &conn_addr->addr.sockaddr, conn_addr->addrlen,
                          o.proto, inet_port(&conn_addr->addr),
                          NULL);
    }
    else
#endif
#ifdef HAVE_LINUX_VM_SOCKETS_H
    # 如果支持 Linux VM 套接字
    if (o.af == AF_VSOCK) {
        # 如果地址族是 AF_VSOCK
        if (o.proto == IPPROTO_UDP) {
            # 如果协议是 IPPROTO_UDP，进行 VSOCK 数据报套接字连接
            nsock_connect_vsock_datagram(nsp, cs.sock_nsi, connect_handler,
                    (void *)conn_addr->next, &conn_addr->addr.sockaddr,
                    conn_addr->addrlen, conn_addr->addr.vm.svm_port);
        } else {
            # 否则进行 VSOCK 流套接字连接
            nsock_connect_vsock_stream(nsp, cs.sock_nsi, connect_handler,
                    o.conntimeout, (void *)conn_addr->next,
                    &conn_addr->addr.sockaddr, conn_addr->addrlen,
                    conn_addr->addr.vm.svm_port);
        }
    }
    else
#endif
    # 如果协议是 IPPROTO_UDP，进行 UDP 套接字连接
    if (o.proto == IPPROTO_UDP) {
        nsock_connect_udp(nsp, cs.sock_nsi, connect_handler, (void *)conn_addr->next,
                          &conn_addr->addr.sockaddr, conn_addr->addrlen,
                          inet_port(&conn_addr->addr));
    }
    else if (o.proto == IPPROTO_SCTP) {
        # 如果协议是 IPPROTO_SCTP，进行 SCTP 套接字连接
        nsock_connect_sctp(nsp, cs.sock_nsi, connect_handler,
                          o.conntimeout, (void *)conn_addr->next,
                          &conn_addr->addr.sockaddr, conn_addr->addrlen,
                          inet_port(&conn_addr->addr));
    }
    # 如果前面的条件都不满足，则执行以下代码块
    else {
        # 使用 TCP 协议连接指定的套接字地址，并指定连接处理函数
        nsock_connect_tcp(nsp, cs.sock_nsi, connect_handler,
                          o.conntimeout, (void *)conn_addr->next,
                          &conn_addr->addr.sockaddr, conn_addr->addrlen,
                          inet_port(&conn_addr->addr));
    }
}
# 定义一个静态函数，用于向目标发送空的 UDP 数据包
static void send_udp_null(nsock_pool nsp)
{
  # 定义一个空的 UDP 数据包
  char *NULL_PROBE = "\0";
  # 数据包长度为1
  int length = 1;
  # 向目标发送 UDP 数据包
  nsock_write(nsp, cs.sock_nsi, write_socket_handler, -1, NULL, NULL_PROBE, length);
}

# 定义连接处理函数
static void connect_handler(nsock_pool nsp, nsock_event evt, void *data)
{
    # 获取事件状态和类型
    enum nse_status status = nse_status(evt);
    enum nse_type type = nse_type(evt);
    # 获取下一个地址的信息
    struct sockaddr_list *next_addr = (struct sockaddr_list *)data;

    # 断言连接类型为连接或连接 SSL
    ncat_assert(type == NSE_TYPE_CONNECT || type == NSE_TYPE_CONNECT_SSL);

    # 如果连接出现错误或超时
    if (status == NSE_STATUS_ERROR || status == NSE_STATUS_TIMEOUT) {
        # 如果还有更多的解析地址，尝试连接到下一个地址
        if (next_addr != NULL) {
            # 如果设置了详细输出，记录连接失败的信息
            if (o.verbose) {
                union sockaddr_u peer;
                zmem(&peer, sizeof(peer.storage));
                nsock_iod_get_communication_info(cs.sock_nsi, NULL, NULL, NULL,
                    &peer.sockaddr, sizeof(peer.storage));
                loguser("Connection to %s failed: %s.\n", inet_socktop(&peer),
                    (status == NSE_STATUS_TIMEOUT)
                    ? nse_status2str(status)
                    : socket_strerror(nse_errorcode(evt)));
                loguser("Trying next address...\n");
            }
            # 删除旧的 IOD 并为下一个地址创建一个新的 IOD
            nsock_iod_delete(cs.sock_nsi, NSOCK_PENDING_NOTIFY);
            cs.sock_nsi = new_iod(nsp);

            # 尝试连接到下一个地址
            try_nsock_connect(nsp, next_addr);
            return;
        }
        else {
            # 释放目标地址列表，如果设置了非零字节或详细输出，记录连接失败的信息并退出
            free_sockaddr_list(targetaddrs);
            if (!o.zerobyte||o.verbose)
              loguser("%s.\n",
                  (status == NSE_STATUS_TIMEOUT)
                  ? nse_status2str(status)
                  : socket_strerror(nse_errorcode(evt)));
            exit(1);
        }
    } else {
        # 断言连接状态为成功
        ncat_assert(status == NSE_STATUS_SUCCESS);
    }

#ifdef HAVE_OPENSSL
    # 如果套接字的 SSL 校验通过
    if (nsock_iod_check_ssl(cs.sock_nsi)) {
        # 检查域名。如果需要，ssl_post_connect_check 会打印错误信息。
        if (!ssl_post_connect_check((SSL *)nsock_iod_get_ssl(cs.sock_nsi), o.sslservername))
            # 如果证书验证出错，退出程序并打印错误信息
            bye("Certificate verification error.");
    }
#endif

    # 连接报告
    connect_report(cs.sock_nsi);
    # 如果协议不是UDP且zerobyte为真，则退出循环
    if (o.proto != IPPROTO_UDP && o.zerobyte):
      nsock_loop_quit(nsp)

    # 为nsp->stdin创建IOD
    # 如果无法创建，则输出错误信息并退出程序
    if ((cs.stdin_nsi = nsock_iod_new2(nsp, 0, NULL)) == NULL)
        bye("Failed to create stdin nsiod.");

    # 处理连接后的操作
    post_connect(nsp, nse_iod(evt));
}

# 如果适用，处理--exec，否则启动初始读取事件并设置空闲超时
static void post_connect(nsock_pool nsp, nsock_iod iod)
{
    # 执行的命令
    if (o.cmdexec):
        struct fdinfo info;

        info.fd = nsock_iod_get_sd(iod);
        # 如果有OpenSSL，则获取SSL信息
        # 将Nsock的非阻塞套接字转换为普通阻塞套接字
        # 可能会在非阻塞套接字上写入速度过快而导致EAGAIN错误
        # 执行命令
        # 开始初始读取
        if (!o.sendonly && !o.zerobyte)
            nsock_read(nsp, cs.sock_nsi, read_socket_handler, -1, NULL);

        if (!o.recvonly && !o.zerobyte)
            nsock_readbytes(nsp, cs.stdin_nsi, read_stdin_handler, -1, NULL, 0);

        if (o.zerobyte && o.proto==IPPROTO_UDP)
          send_udp_null(nsp);

        # --idle-timeout选项表示在一定的空闲时间后退出
        # 在这里启动一个定时器，并在每次读取事件时重置它
        if (o.idletimeout > 0):
            cs.idle_timer_event_id =
                nsock_timer_create(nsp, idle_timer_handler, o.idletimeout, NULL);
    }
}

static void read_stdin_handler(nsock_pool nsp, nsock_event evt, void *data)
{
    enum nse_status status = nse_status(evt);
    enum nse_type type = nse_type(evt);
    char *buf, *tmp = NULL;
    int nbytes;

    # 断言事件类型为读取
    ncat_assert(type == NSE_TYPE_READ);

    # 如果状态为EOF
    if (status == NSE_STATUS_EOF):
        # 如果不是noshutdown，则执行以下操作
#ifdef HAVE_OPENSSL
            // 如果支持 OpenSSL，并且 SSL 对象存在，则执行 SSL 关闭操作
            SSL *ssl = NULL;
            if (o.ssl && NULL != (ssl = (SSL *)nsock_iod_get_ssl(cs.sock_nsi))) {
                SSL_shutdown(ssl);
            }
            else
#endif
                // 否则执行关闭操作
                shutdown(nsock_iod_get_sd(cs.sock_nsi), SHUT_WR);
        }
        /* In --send-only mode or non-TCP mode, exit after EOF on stdin. */
        // 如果是 --send-only 模式或者非 TCP 模式，则在标准输入结束后退出
        if (o.proto != IPPROTO_TCP || (o.proto == IPPROTO_TCP && o.sendonly))
            nsock_loop_quit(nsp);
        return;
    } else if (status == NSE_STATUS_ERROR) {
        // 如果状态为错误，则记录错误信息并退出
        loguser("%s.\n", socket_strerror(nse_errorcode(evt)));
        exit(1);
    } else if (status == NSE_STATUS_TIMEOUT) {
        // 如果状态为超时，则记录超时信息并退出
        loguser("%s.\n", nse_status2str(status));
        exit(1);
    } else if (status == NSE_STATUS_CANCELLED || status == NSE_STATUS_KILL) {
        // 如果状态为取消或者终止，则直接返回
        return;
    } else {
        // 否则断言状态为成功
        ncat_assert(status == NSE_STATUS_SUCCESS);
    }

    // 从事件中读取数据到缓冲区
    buf = nse_readbuf(evt, &nbytes);

    /* read from stdin */
    // 如果设置了行延迟，则延迟一定时间
    if (o.linedelay)
        ncat_delay_timer(o.linedelay);

    // 如果设置了 CRLF 转换，则执行转换操作
    if (o.crlf) {
        if (fix_line_endings(buf, &nbytes, &tmp, &cs.crlf_state))
            buf = tmp;
    }

    // 将缓冲区的数据写入套接字
    nsock_write(nsp, cs.sock_nsi, write_socket_handler, -1, NULL, buf, nbytes);
    // 记录发送的数据
    ncat_log_send(buf, nbytes);

    // 如果临时缓冲区存在，则释放内存
    if (tmp)
        free(tmp);

    // 刷新空闲定时器
    refresh_idle_timer(nsp);
}

// 读取套接字事件处理函数
static void read_socket_handler(nsock_pool nsp, nsock_event evt, void *data)
{
    // 获取事件状态和类型
    enum nse_status status = nse_status(evt);
    enum nse_type type = nse_type(evt);
    char *buf;
    int nbytes;

    // 断言事件类型为读取
    ncat_assert(type == NSE_TYPE_READ);

    // 如果状态为 EOF
    if (status == NSE_STATUS_EOF) {
#ifdef WIN32
        _close(STDOUT_FILENO);
#else
        Close(STDOUT_FILENO);
#endif
        /* In --recv-only mode or non-TCP mode, exit after EOF on the socket. */
        // 如果是 --recv-only 模式或者非 TCP 模式，则在套接字结束后退出
        if (o.proto != IPPROTO_TCP || (o.proto == IPPROTO_TCP && o.recvonly))
            nsock_loop_quit(nsp);
        return;
    } else if (status == NSE_STATUS_ERROR) {
        // 如果状态为错误，且不是零字节或者需要显示详细信息，则记录错误信息并退出程序
        if (!o.zerobyte||o.verbose)
          loguser("%s.\n", socket_strerror(nse_errorcode(evt)));
        exit(1);
    } else if (status == NSE_STATUS_TIMEOUT) {
        // 如果状态为超时，则记录超时信息并退出程序
        loguser("%s.\n", nse_status2str(status));
        exit(1);
    } else if (status == NSE_STATUS_CANCELLED || status == NSE_STATUS_KILL) {
        // 如果状态为取消或者终止，则直接返回
        return;
    } else {
        // 否则，断言状态为成功
        ncat_assert(status == NSE_STATUS_SUCCESS);
    }

    // 从事件中读取数据到缓冲区
    buf = nse_readbuf(evt, &nbytes);

    // 如果设置了行延迟，则延迟指定时间
    if (o.linedelay)
        ncat_delay_timer(o.linedelay);

    // 如果设置了 telnet，则执行 telnet 操作
    if (o.telnet)
        dotelnet(nsock_iod_get_sd(nse_iod(evt)), (unsigned char *) buf, nbytes);

    /* 将套接字数据写入标准输出 */
    Write(STDOUT_FILENO, buf, nbytes);
    // 记录接收到的数据
    ncat_log_recv(buf, nbytes);

    // 从套接字读取数据，并指定读取数据的处理函数
    nsock_readbytes(nsp, cs.sock_nsi, read_socket_handler, -1, NULL, 0);

    // 刷新空闲计时器
    refresh_idle_timer(nsp);
# 定义一个静态函数，用于处理写入套接字的事件
static void write_socket_handler(nsock_pool nsp, nsock_event evt, void *data)
{
    # 获取事件的状态和类型
    enum nse_status status = nse_status(evt);
    enum nse_type type = nse_type(evt);

    # 确保事件类型为写入类型
    ncat_assert(type == NSE_TYPE_WRITE);

    # 如果写入套接字发生错误，则记录错误信息并退出程序
    if (status == NSE_STATUS_ERROR) {
        loguser("%s.\n", socket_strerror(nse_errorcode(evt)));
        exit(1);
    } 
    # 如果写入套接字超时，则记录超时信息并退出程序
    else if (status == NSE_STATUS_TIMEOUT) {
        loguser("%s.\n", nse_status2str(status));
        exit(1);
    } 
    # 如果事件被取消或终止，则直接返回
    else if (status == NSE_STATUS_CANCELLED || status == NSE_STATUS_KILL) {
        return;
    } 
    # 否则，确保事件状态为成功
    else {
        ncat_assert(status == NSE_STATUS_SUCCESS);
    }

    # 如果设置了zerobyte选项且协议为UDP，则确保协议为UDP，并开始读取套接字数据
    if (o.zerobyte){
      ncat_assert(o.proto == IPPROTO_UDP);
      nsock_read(nsp, cs.sock_nsi, read_socket_handler, -1, NULL);
      return;
    }
    # 写入套接字成功后，允许从标准输入读取更多数据
    nsock_readbytes(nsp, cs.stdin_nsi, read_stdin_handler, -1, NULL, 0);
}

# 定义一个静态函数，用于处理空闲定时器事件
static void idle_timer_handler(nsock_pool nsp, nsock_event evt, void *data)
{
    # 获取事件的状态和类型
    enum nse_status status = nse_status(evt);
    enum nse_type type = nse_type(evt);

    # 确保事件类型为定时器类型
    ncat_assert(type == NSE_TYPE_TIMER);

    # 如果事件被取消或终止，则直接返回
    if (status == NSE_STATUS_CANCELLED || status == NSE_STATUS_KILL)
        return;

    # 确保事件状态为成功
    ncat_assert(status == NSE_STATUS_SUCCESS);

    # 如果设置了zerobyte选项且协议为UDP，则记录UDP数据包发送成功，并退出循环
    if (o.zerobyte&&o.proto==IPPROTO_UDP){
      if (o.verbose)
        loguser("UDP packet sent successfully\n");
      nsock_loop_quit(nsp);
      return;
    }

    # 记录空闲超时已过期的信息，并退出程序
    loguser("Idle timeout expired (%d ms).\n", o.idletimeout);

    exit(1);
}

# 定义一个静态函数，用于刷新空闲定时器
static void refresh_idle_timer(nsock_pool nsp)
{
    # 如果空闲超时时间小于等于0，则直接返回
    if (o.idletimeout <= 0)
        return;
    # 取消之前的空闲定时器事件，并创建新的空闲定时器事件
    nsock_event_cancel(nsp, cs.idle_timer_event_id, 0);
    cs.idle_timer_event_id =
        nsock_timer_create(nsp, idle_timer_handler, o.idletimeout, NULL);
}
```