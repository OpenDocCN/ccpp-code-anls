# `nmap\ncat\ncat_proxy.c`

```
/* $Id$ */

#include "base64.h"  // 包含 base64 相关的头文件
#include "http.h"  // 包含 http 相关的头文件
#include "nsock.h"  // 包含 nsock 相关的头文件
#include "ncat.h"  // 包含 ncat 相关的头文件
#include "sys_wrap.h"  // 包含系统封装相关的头文件

#ifndef WIN32
#include <unistd.h>  // 如果不是 Windows 平台，包含 unistd.h 头文件
#endif

#ifndef WIN32
/* SIG_CHLD handler */
static void proxyreaper(int signo)  // 定义 SIG_CHLD 信号处理函数
{
    while (waitpid(-1, NULL, WNOHANG) > 0)  // 循环等待子进程退出
        ;
}
#endif

/* send a '\0'-terminated string. */
static int send_string(struct fdinfo *fdn, const char *s)  // 发送以 '\0' 结尾的字符串
{
    return fdinfo_send(fdn, s, strlen(s));  // 调用 fdinfo_send 函数发送字符串
}

static void http_server_handler(int c);  // 定义 http 服务器处理函数
static int send_proxy_authenticate(struct fdinfo *fdn, int stale);  // 发送代理身份验证信息
static char *http_code2str(int code);  // 将 HTTP 状态码转换为字符串

static void fork_handler(int s, int c);  // 定义 fork 处理函数

static int handle_connect(struct socket_buffer *client_sock,
    struct http_request *request);  // 处理 CONNECT 方法
static int handle_method(struct socket_buffer *client_sock,
    struct http_request *request);  // 处理 HTTP 方法

static int check_auth(const struct http_request *request,
    const struct http_credentials *credentials, int *stale);  // 检查身份验证信息
/*
 * Simple forking HTTP proxy. It is an HTTP/1.0 proxy with knowledge of
 * HTTP/1.1. (The things lacking for HTTP/1.1 are the chunked transfer encoding
 * and the expect mechanism.) The proxy supports the CONNECT, GET, HEAD, and
 * POST methods. It supports Basic and Digest authentication of clients (use the
 * --proxy-auth option).
 *
 * HTTP/1.1 is defined in RFC 2616. Many comments refer to that document.
 * http://tools.ietf.org/html/rfc2616
 *
 * HTTP authentication is discussed in RFC 2617.
 * http://tools.ietf.org/html/rfc2617
 *
 * The CONNECT method is documented in an Internet draft and is specified as the
 * way to proxy HTTPS in RFC 2817, section 5.
 * http://tools.ietf.org/html/draft-luotonen-web-proxy-tunneling-01
 * http://tools.ietf.org/html/rfc2817#section-5
 *
 * The CONNECT method is not limited to HTTP, but is potentially capable of
 * connecting to any TCP port on any host. The proxy connection is requested
 * with an HTTP request, but after that, the proxy does no interpretation of the
 * data passing through it. See section 6 of the above mentioned draft for the
 * security implications.
 */
int ncat_http_server(void)
{
    // 定义变量
    int c, i, j;
    int listen_socket[NUM_LISTEN_ADDRS];
    socklen_t sslen;
    union sockaddr_u conn;
    unsigned int num_sockets;

    // 在非 Windows 系统下，设置信号处理函数 proxyreaper
#ifndef WIN32
    Signal(SIGCHLD, proxyreaper);
#endif

    // 如果支持 HTTP Digest 认证，初始化密钥
#if HAVE_HTTP_DIGEST
    http_digest_init_secret();
#endif

    // 如果支持 OpenSSL，且启用 SSL，设置 SSL 监听
#ifdef HAVE_OPENSSL
    if (o.ssl)
        setup_ssl_listen(SSLv23_server_method());
#endif

    // 清空监听套接字列表
    for (i = 0; i < NUM_LISTEN_ADDRS; i++)
        listen_socket[i] = -1;

    // 设置用于选择监听套接字的文件描述符集合
    fd_set listen_fds;
    fd_list_t listen_fdlist;
    FD_ZERO(&listen_fds);
    init_fdlist(&listen_fdlist, num_listenaddrs);

    // 在每个地址上监听，设置用于 select 的列表
    num_sockets = 0;
    # 遍历监听地址列表
    for (i = 0; i < num_listenaddrs; i++) {
        # 调用 do_listen 函数创建监听套接字，并将其存储在 listen_socket 数组中
        listen_socket[num_sockets] = do_listen(SOCK_STREAM, IPPROTO_TCP, &listenaddrs[i]);
        # 如果创建监听套接字失败
        if (listen_socket[num_sockets] == -1) {
            # 如果调试模式开启，记录错误日志
            if (o.debug > 0)
                logdebug("do_listen(\"%s\"): %s\n", socktop(&listenaddrs[i], 0), socket_strerror(socket_errno()));
            # 继续下一次循环
            continue;
        }

        # 设置监听套接字为非阻塞模式
        /* make us not block on accepts in weird cases. See ncat_listen.c:209 */
        unblock_socket(listen_socket[num_sockets]);

        # 设置 select 集合和最大文件描述符
        /* setup select sets and max fd */
        checked_fd_set(listen_socket[num_sockets], &listen_fds);
        add_fd(&listen_fdlist, listen_socket[num_sockets]);

        # 增加已创建的监听套接字数量
        num_sockets++;
    }
    # 如果没有成功创建任何监听套接字
    if (num_sockets == 0) {
        # 如果只有一个监听地址，记录错误日志并退出程序
        if (num_listenaddrs == 1)
            bye("Unable to open listening socket on %s: %s", socktop(&listenaddrs[0], 0), socket_strerror(socket_errno()));
        # 如果有多个监听地址，记录错误日志并退出程序
        else
            bye("Unable to open any listening sockets.");
    }
    // 无限循环，用于处理多个连接
    for (;;) {
        // 创建一个文件描述符集合
        fd_set read_fds;

        // 获取连接存储的大小
        sslen = sizeof(conn.storage);
        /*
         * 通过 select 函数获取可以通信的套接字列表
         */
        if (o.debug > 1)
            logdebug("selecting, fdmax %d\n", listen_fdlist.fdmax);
        // 将监听套接字集合赋值给 read_fds
        read_fds = listen_fds;

        // 调用 fselect 函数，获取准备好的文件描述符数量
        int fds_ready = fselect(listen_fdlist.fdmax + 1, &read_fds, NULL, NULL, NULL);

        if (o.debug > 1)
            logdebug("select returned %d fds ready\n", fds_ready);

        // 如果没有准备好的文件描述符，触发空闲超时
        if (fds_ready == 0)
            bye("Idle timeout expired (%d ms).", o.idletimeout);

        // 遍历文件描述符，直到有准备好的文件描述符
        for (i = 0; i <= listen_fdlist.fdmax && fds_ready > 0; i++) {
            /* Loop through descriptors until there is something ready */
            if (!checked_fd_isset(i, &read_fds))
                continue;

            /* Check each listening socket */
            for (j = 0; j < num_sockets; j++) {
                // 如果当前文件描述符是监听套接字
                if (i == listen_socket[j]) {
                    fds_ready--;
                    // 接受连接请求，获取客户端套接字
                    c = accept(i, &conn.sockaddr, &sslen);

                    // 如果接受连接失败
                    if (c == -1) {
                        if (errno == EINTR)
                            continue;
                        die("accept");
                    }

                    // 如果不允许访问
                    if (!allow_access(&conn)) {
                        Close(c);
                        continue;
                    }
                    // 如果调试级别大于 1，记录处理程序的分叉
                    if (o.debug > 1)
                        logdebug("forking handler for %d\n", i);
                    // 分叉处理程序
                    fork_handler(i, c);
                }
            }
        }
    }
    // 返回 0
    return 0;
}
#ifdef WIN32
/* 在 Windows 上，我们实际上不是 fork，而是启动一个线程。 */

static DWORD WINAPI handler_thread_func(void *data)
{
    http_server_handler(*((int *) data));
    free(data);

    return 0;
}

static void fork_handler(int s, int c)
{
    int *data;
    HANDLE thread;

    data = (int *) safe_malloc(sizeof(int));
    *data = c;
    thread = CreateThread(NULL, 0, handler_thread_func, data, 0, NULL);
    if (thread == NULL) {
        if (o.verbose)
            logdebug("Error in CreateThread: %d\n", GetLastError());
        free(data);
        return;
    }
    CloseHandle(thread);
}
#else
static void fork_handler(int s, int c)
{
    int rc;

    rc = fork();
    if (rc == -1) {
        return;
    } else if (rc == 0) {
        Close(s);

        if (!o.debug) {
            Close(STDIN_FILENO);
            Close(STDOUT_FILENO);
            Close(STDERR_FILENO);
        }

        http_server_handler(c);
        exit(0);
    } else {
        Close(c);
    }
}
#endif

/* 这是我们能处理的方法之一吗？ */
static int method_is_known(const char *method)
{
    return strcmp(method, "CONNECT") == 0
        || strcmp(method, "GET") == 0
        || strcmp(method, "HEAD") == 0
        || strcmp(method, "POST") == 0;
}

static void http_server_handler(int c)
{
    int code;
    struct socket_buffer sock;
    struct http_request request;
    char *buf;

    socket_buffer_init(&sock, c);
#if HAVE_OPENSSL
    if (o.ssl) {
        sock.fdn.ssl = new_ssl(sock.fdn.fd);
        if (SSL_accept(sock.fdn.ssl) != 1) {
            loguser("Failed SSL connection: %s\n",
                ERR_error_string(ERR_get_error(), NULL));
            fdinfo_close(&sock.fdn);
            return;
        }
    }
#endif

    code = http_read_request_line(&sock, &buf);
    if (code != 0) {
        if (o.verbose)
            logdebug("Error reading Request-Line.\n");
        send_string(&sock.fdn, http_code2str(code));
        fdinfo_close(&sock.fdn);
        return;
    }
    # 如果调试级别大于1，则记录请求行
    if (o.debug > 1)
        logdebug("Request-Line: %s", buf);
    # 解析请求行，将结果保存在request中
    code = http_parse_request_line(buf, &request);
    # 释放buf所指向的内存
    free(buf);
    # 如果解析请求行出错
    if (code != 0) {
        # 如果设置了详细输出，则记录错误信息
        if (o.verbose)
            logdebug("Error parsing Request-Line.\n");
        # 发送错误码给客户端
        send_string(&sock.fdn, http_code2str(code));
        # 关闭文件描述符
        fdinfo_close(&sock.fdn);
        # 返回
        return;
    }

    # 如果请求方法未知
    if (!method_is_known(request.method)) {
        # 如果调试级别大于1，则记录错误信息
        if (o.debug > 1)
            logdebug("Bad method: %s.\n", request.method);
        # 释放request中的内存
        http_request_free(&request);
        # 发送错误码给客户端
        send_string(&sock.fdn, http_code2str(405));
        # 关闭文件描述符
        fdinfo_close(&sock.fdn);
        # 返回
        return;
    }

    # 读取请求头部
    code = http_read_header(&sock, &buf);
    # 如果读取请求头部出错
    if (code != 0) {
        # 如果设置了详细输出，则记录错误信息
        if (o.verbose)
            logdebug("Error reading header.\n");
        # 释放request中的内存
        http_request_free(&request);
        # 发送错误码给客户端
        send_string(&sock.fdn, http_code2str(code));
        # 关闭文件描述符
        fdinfo_close(&sock.fdn);
        # 返回
        return;
    }
    # 如果调试级别大于1，则记录请求头部
    if (o.debug > 1)
        logdebug("Header:\n%s", buf);
    # 解析请求头部，将结果保存在request中
    code = http_request_parse_header(&request, buf);
    # 释放buf所指向的内存
    free(buf);
    # 如果解析请求头部出错
    if (code != 0) {
        # 如果设置了详细输出，则记录错误信息
        if (o.verbose)
            logdebug("Error parsing header.\n");
        # 释放request中的内存
        http_request_free(&request);
        # 发送错误码给客户端
        send_string(&sock.fdn, http_code2str(code));
        # 关闭文件描述符
        fdinfo_close(&sock.fdn);
        # 返回
        return;
    }

    # 检查认证信息
    # 如果存在代理授权信息
    if (o.proxy_auth) {
        # 定义一个结构体变量用于存储代理认证信息
        struct http_credentials credentials;
        # 定义变量用于存储认证结果和是否过期
        int ret, stale;

        # 如果无法获取代理认证信息或者解析错误
        if (http_header_get_proxy_credentials(request.header, &credentials) == NULL) {
            # 发送代理认证请求
            send_proxy_authenticate(&sock.fdn, 0);
            # 释放请求对象内存
            http_request_free(&request);
            # 关闭套接字
            fdinfo_close(&sock.fdn);
            # 返回
            return;
        }

        # 检查认证信息
        ret = check_auth(&request, &credentials, &stale);
        # 释放认证信息内存
        http_credentials_free(&credentials);
        # 如果认证失败
        if (!ret) {
            # 发送代理认证请求
            send_proxy_authenticate(&sock.fdn, stale);
            # 释放请求对象内存
            http_request_free(&request);
            # 关闭套接字
            fdinfo_close(&sock.fdn);
            # 返回
            return;
        }
    }

    # 如果请求方法是 CONNECT
    if (strcmp(request.method, "CONNECT") == 0) {
        # 处理 CONNECT 方法
        code = handle_connect(&sock, &request);
    } 
    # 如果请求方法是 GET、HEAD 或 POST
    else if (strcmp(request.method, "GET") == 0
        || strcmp(request.method, "HEAD") == 0
        || strcmp(request.method, "POST") == 0) {
        # 处理对应的方法
        code = handle_method(&sock, &request);
    } 
    # 其他情况
    else {
        # 返回 500 错误码
        code = 500;
    }
    # 释放请求对象内存
    http_request_free(&request);

    # 如果返回码不为 0
    if (code != 0) {
        # 发送错误码对应的字符串
        send_string(&sock.fdn, http_code2str(code));
        # 关闭套接字
        fdinfo_close(&sock.fdn);
        # 返回
        return;
    }

    # 关闭套接字
    fdinfo_close(&sock.fdn);
}

static int handle_connect(struct socket_buffer *client_sock,
    struct http_request *request)
{
    union sockaddr_u su;  // 定义一个联合体变量 su，用于存储地址信息
    size_t sslen = sizeof(su.storage);  // 获取地址信息的大小
    int maxfd, s, rc;  // 定义整型变量 maxfd, s, rc
    char *line;  // 定义字符指针变量 line
    size_t len;  // 定义大小变量 len
    fd_set m, r;  // 定义文件描述符集合变量 m, r

    if (request->uri.port == -1) {  // 如果请求中的端口号为-1
        if (o.verbose)  // 如果设置了详细输出
            logdebug("No port number in CONNECT URI.\n");  // 输出日志信息
        return 400;  // 返回400错误
    }
    if (o.debug > 1)  // 如果调试级别大于1
        logdebug("CONNECT to %s:%d.\n", request->uri.host, request->uri.port);  // 输出连接信息

    rc = resolve(request->uri.host, request->uri.port, &su.storage, &sslen, o.af);  // 解析主机名和端口号
    if (rc != 0) {  // 如果解析失败
        if (o.debug) {  // 如果设置了调试模式
            logdebug("Can't resolve name \"%s\": %s.\n",
                request->uri.host, gai_strerror(rc));  // 输出解析失败的日志信息
        }
        return 504;  // 返回504错误
    }

    s = Socket(su.storage.ss_family, SOCK_STREAM, IPPROTO_TCP);  // 创建一个套接字

    if (connect(s, &su.sockaddr, sslen) == -1) {  // 如果连接失败
        if (o.debug)  // 如果设置了调试模式
            logdebug("Can't connect to %s: %s.\n", socktop(&su, sslen), socket_strerror(socket_errno()));  // 输出连接失败的日志信息
        Close(s);  // 关闭套接字
        return 504;  // 返回504错误
    }

    send_string(&client_sock->fdn, http_code2str(200));  // 发送状态码给客户端

    /* Clear out whatever is left in the socket buffer. The client may have
       already sent the first part of its request to the origin server. */
    line = socket_buffer_remainder(client_sock, &len);  // 获取套接字缓冲区中剩余的数据
    if (send(s, line, len, 0) < 0) {  // 如果发送剩余数据失败
        if (o.debug)  // 如果设置了调试模式
            logdebug("Error sending %lu leftover bytes: %s.\n", (unsigned long) len, strerror(errno));  // 输出发送失败的日志信息
        Close(s);  // 关闭套接字
        return 0;  // 返回0
    }

    maxfd = client_sock->fdn.fd < s ? s : client_sock->fdn.fd;  // 获取最大的文件描述符
    FD_ZERO(&m);  // 清空文件描述符集合 m
    checked_fd_set(client_sock->fdn.fd, &m);  // 将客户端套接字的文件描述符加入集合 m
    checked_fd_set(s, &m);  // 将新创建的套接字的文件描述符加入集合 m

    errno = 0;  // 设置错误号为0
    # 当套接字没有错误或者套接字错误为中断时，执行循环
    while (!socket_errno() || socket_errno() == EINTR) {
        # 定义一个长度为 DEFAULT_TCP_BUF_LEN 的字符数组 buf
        char buf[DEFAULT_TCP_BUF_LEN];
        # 定义变量 len 和 rc，用于存储长度和返回值
        int len, rc;

        # 将 r 的值赋给 m
        r = m;

        # 使用 fselect 函数监视套接字的可读性
        fselect(maxfd + 1, &r, NULL, NULL, NULL);

        # 将 buf 数组清零
        zmem(buf, sizeof(buf));

        # 如果客户端套接字的文件描述符在 r 中被设置
        if (checked_fd_isset(client_sock->fdn.fd, &r)) {
            # 执行循环
            do {
                # 执行循环，直到成功接收数据或者出现中断错误
                do {
                    len = fdinfo_recv(&client_sock->fdn, buf, sizeof(buf));
                } while (len == -1 && socket_errno() == EINTR);
                # 如果接收到的数据长度小于等于0，则跳转到 end 标签处
                if (len <= 0)
                    goto end;

                # 执行循环，直到成功发送数据或者出现中断错误
                do {
                    rc = send(s, buf, len, 0);
                } while (rc == -1 && socket_errno() == EINTR);
                # 如果发送数据出现错误，则跳转到 end 标签处
                if (rc == -1)
                    goto end;
            } while (fdinfo_pending(&client_sock->fdn));
        }

        # 如果套接字 s 在 r 中被设置
        if (checked_fd_isset(s, &r)) {
            # 执行循环，直到成功接收数据或者出现中断错误
            do {
                len = recv(s, buf, sizeof(buf), 0);
            } while (len == -1 && socket_errno() == EINTR);
            # 如果接收到的数据长度小于等于0，则跳转到 end 标签处
            if (len <= 0)
                goto end;

            # 执行循环，直到成功发送数据或者出现中断错误
            do {
                rc = fdinfo_send(&client_sock->fdn, buf, len);
            } while (rc == -1 && socket_errno() == EINTR);
            # 如果发送数据出现错误，则跳转到 end 标签处
            if (rc == -1)
                goto end;
        }
    }
end:

    close(s);  # 关闭套接字s

    return 0;  # 返回0
}

static int do_transaction(struct http_request *request,
    struct socket_buffer *client_sock, struct socket_buffer *server_sock);

/* Generic handler for GET, HEAD, and POST methods. */
static int handle_method(struct socket_buffer *client_sock,
    struct http_request *request)
{
    struct socket_buffer server_sock;  # 定义服务器套接字缓冲区
    union sockaddr_u su;  # 定义sockaddr_u联合体su
    size_t sslen = sizeof(su.storage);  # 定义sslen为su.storage的大小
    int code;  # 定义整型变量code
    int s, rc;  # 定义整型变量s和rc

    if (strcmp(request->uri.scheme, "http") != 0) {  # 如果请求的URI scheme不是http
        if (o.verbose)  # 如果o.verbose为真
            logdebug("Unknown scheme in URI: %s.\n", request->uri.scheme);  # 记录调试信息
        return 400;  # 返回400
    }
    if (request->uri.port == -1) {  # 如果请求的URI端口为-1
        if (o.verbose)  # 如果o.verbose为真
            logdebug("Unknown port in URI.\n");  # 记录调试信息
        return 400;  # 返回400
    }

    rc = resolve(request->uri.host, request->uri.port, &su.storage, &sslen, o.af);  # 解析URI的主机和端口
    if (rc != 0) {  # 如果解析失败
        if (o.debug) {  # 如果o.debug为真
            logdebug("Can't resolve name %s:%d: %s.\n",
                request->uri.host, request->uri.port, gai_strerror(rc));  # 记录调试信息
        }
        return 504;  # 返回504
    }

    /* RFC 2616, section 5.1.2: "In order to avoid request loops, a proxy MUST
       be able to recognize all of its server names, including any aliases,
       local variations, and the numeric IP address. */
    if (request->uri.port == o.portno && addr_is_local(&su)) {  # 如果请求的URI端口等于o.portno并且地址是本地地址
        if (o.verbose)  # 如果o.verbose为真
            logdebug("Proxy loop detected: %s:%d\n", request->uri.host, request->uri.port);  # 记录调试信息
        return 403;  # 返回403
    }

    s = Socket(su.storage.ss_family, SOCK_STREAM, IPPROTO_TCP);  # 创建套接字s

    if (connect(s, &su.sockaddr, sslen) == -1) {  # 如果连接失败
        if (o.debug)  # 如果o.debug为真
            logdebug("Can't connect to %s: %s.\n", socktop(&su, sslen), socket_strerror(socket_errno()));  # 记录调试信息
        Close(s);  # 关闭套接字s
        return 504;  # 返回504
    }

    socket_buffer_init(&server_sock, s);  # 初始化服务器套接字缓冲区

    code = do_transaction(request, client_sock, &server_sock);  # 进行事务处理

    fdinfo_close(&server_sock.fdn);  # 关闭服务器套接字缓冲区的文件描述符

    if (code != 0)  # 如果返回值不为0
        return code;  # 返回返回值

    return 0;  # 返回0
}

/* Do a GET, HEAD, or POST transaction. */
static int do_transaction(struct http_request *request,
    struct socket_buffer *client_sock, struct socket_buffer *server_sock)
{
    char buf[BUFSIZ]; // 创建一个缓冲区数组，用于存储临时数据
    struct http_response response; // 创建一个 HTTP 响应结构体
    char *line; // 创建一个指向字符的指针变量
    char *request_str, *response_str; // 创建两个指向字符的指针变量
    size_t len; // 创建一个用于存储长度的变量
    int code, n; // 创建两个整型变量

    /* We don't handle the chunked transfer encoding, which in the absence of a
       Content-Length is the only way we know the end of a request body. RFC
       2616, section 4.4 says, "If a request contains a message-body and a
       Content-Length is not given, the server SHOULD respond with 400 (bad
       request) if it cannot determine the length of the message, or with 411
       (length required) if it wishes to insist on receiving a valid
       Content-Length." */
    // 如果是 POST 请求且没有设置 Content-Length，则返回 400 错误
    if (strcmp(request->method, "POST") == 0 && !request->content_length_set) {
        if (o.debug)
            logdebug("POST request with no Content-Length.\n");
        return 400;
    }

    /* The version we use to talk to the server. */
    // 设置与服务器通信的 HTTP 版本为 1.0
    request->version = HTTP_10;

    /* Remove headers that only apply to our connection with the client. */
    // 移除只适用于与客户端连接的头部信息
    code = http_header_remove_hop_by_hop(&request->header);
    if (code != 0) {
        if (o.verbose)
            logdebug("Error removing hop-by-hop headers.\n");
        return code;
    }

    /* Build the Host header. */
    // 构建 Host 头部信息
    if (request->uri.port == -1 || request->uri.port == 80)
        n = Snprintf(buf, sizeof(buf), "%s", request->uri.host);
    else
        n = Snprintf(buf, sizeof(buf), "%s:%d", request->uri.host, request->uri.port);
    if (n < 0 || n >= sizeof(buf)) {
        /* Request Entity Too Large. */
        return 501;
    }
    request->header = http_header_set(request->header, "Host", buf);

    request->header = http_header_set(request->header, "Connection", "close");

    /* Send the request to the server. */
    // 将请求发送到服务器
    request_str = http_request_to_string(request, &len);
    n = send(server_sock->fdn.fd, request_str, len, 0);
    free(request_str);
}
    # 如果请求体长度小于0，则返回504错误
    if (n < 0)
        return 504;
    # 发送请求体，如果有的话。计数到Content-Length
    while (request->bytes_transferred < request->content_length) {
        # 从客户端套接字读取数据到缓冲区，最多读取sizeof(buf)大小的数据，或者Content-Length减去已传输字节数的大小
        n = socket_buffer_read(client_sock, buf, MIN(sizeof(buf), request->content_length - request->bytes_transferred));
        # 如果读取出错，则返回504错误
        if (n < 0)
            return 504;
        # 如果没有读取到数据，则跳出循环
        if (n == 0)
            break;
        # 更新已传输字节数
        request->bytes_transferred += n;
        # 将数据发送到服务器套接字
        n = send(server_sock->fdn.fd, buf, n, 0);
        # 如果发送出错，则返回504错误
        if (n < 0)
            return 504;
    }
    # 如果调试模式开启且已传输字节数小于Content-Length，则记录日志
    if (o.debug && request->bytes_transferred < request->content_length)
        logdebug("Received only %lu request body bytes (Content-Length was %lu).\n", request->bytes_transferred, request->content_length);

    # 读取响应状态行
    code = http_read_status_line(server_sock, &line);
    # 如果调试模式大于1，则记录状态行日志
    if (o.debug > 1)
        logdebug("Status-Line: %s", line);
    # 如果读取状态行出错，则记录错误日志并返回0
    if (code != 0) {
        if (o.verbose)
            logdebug("Error reading Status-Line.\n");
        return 0;
    }
    # 解析状态行
    code = http_parse_status_line(line, &response);
    free(line);
    # 如果解析状态行出错，则记录错误日志并返回0
    if (code != 0) {
        if (o.verbose)
            logdebug("Error parsing Status-Line.\n");
        return 0;
    }

    # 读取响应头
    code = http_read_header(server_sock, &line);
    # 如果读取响应头出错，则记录错误日志并返回0
    if (code != 0) {
        if (o.verbose)
            logdebug("Error reading header.\n");
        return 0;
    }
    # 如果调试模式大于1，则记录响应头日志
    if (o.debug > 1)
        logdebug("Response header:\n%s", line);

    # 解析响应头
    code = http_response_parse_header(&response, line);
    free(line);
    # 如果解析响应头出错，则记录错误日志并返回0
    if (code != 0) {
        if (o.verbose)
            logdebug("Error parsing response header.\n");
        return 0;
    }

    # 设置与客户端通信的HTTP版本为HTTP_10
    response.version = HTTP_10;

    # 移除只适用于与服务器连接的头部
    code = http_header_remove_hop_by_hop(&response.header);
    # 如果移除头部出错，则记录错误日志并返回错误码
    if (code != 0) {
        if (o.verbose)
            logdebug("Error removing hop-by-hop headers.\n");
        return code;
    }
    # 设置响应头中的 Connection 字段为 close，表示在发送完响应后关闭连接
    response.header = http_header_set(response.header, "Connection", "close");

    # 将响应转换为字符串形式
    response_str = http_response_to_string(&response, &len);
    # 将响应字符串发送给客户端
    n = fdinfo_send(&client_sock->fdn, response_str, len);
    # 释放响应字符串的内存
    free(response_str);
    # 如果发送失败，则释放响应对象的内存并返回 504 错误码
    if (n < 0) {
        http_response_free(&response);
        return 504;
    }
    # 如果 Content-Length 为 0，则读取直到连接关闭；否则读取指定长度的数据
    while (!response.content_length_set
        || response.bytes_transferred < response.content_length) {
        size_t count;

        count = sizeof(buf);
        # 如果 Content-Length 已设置，则计算剩余需要读取的长度
        if (response.content_length_set) {
            size_t remaining = response.content_length - response.bytes_transferred;
            # 如果剩余长度小于缓冲区大小，则设置 count 为剩余长度
            if (remaining < count)
                count = remaining;
        }
        # 从服务器套接字中读取数据到缓冲区
        n = socket_buffer_read(server_sock, buf, count);
        # 如果读取失败或者没有数据可读，则跳出循环
        if (n <= 0)
            break;
        # 更新已传输的字节数
        response.bytes_transferred += n;
        # 将读取的数据发送给客户端
        n = fdinfo_send(&client_sock->fdn, buf, n);
        # 如果发送失败，则跳出循环
        if (n < 0)
            break;
    }

    # 释放响应对象的内存
    http_response_free(&response);

    # 返回 0 表示成功
    return 0;
# 发送一个407代理身份验证要求的响应
static int send_proxy_authenticate(struct fdinfo *fdn, int stale)
{
    char *buf = NULL;
    size_t size = 0, offset = 0;
    int n;

    # 添加HTTP响应头部信息
    strbuf_append_str(&buf, &size, &offset, "HTTP/1.0 407 Proxy Authentication Required\r\n");
    strbuf_append_str(&buf, &size, &offset, "Proxy-Authenticate: Basic realm=\"Ncat\"\r\n");
    # 如果支持HTTP摘要认证，则添加摘要认证头部信息
#if HAVE_HTTP_DIGEST
    {
        char *hdr;

        hdr = http_digest_proxy_authenticate("Ncat", stale);
        strbuf_sprintf(&buf, &size, &offset, "Proxy-Authenticate: %s\r\n", hdr);
        free(hdr);
    }
#endif
    strbuf_append_str(&buf, &size, &offset, "\r\n");

    # 如果调试级别大于1，则记录响应信息
    if (o.debug > 1)
        logdebug("RESPONSE:\n%s", buf);

    # 发送响应信息
    n = send_string(fdn, buf);
    free(buf);

    return n;
}

# 将HTTP状态码转换为字符串
static char *http_code2str(int code)
{
    # 根据状态码返回对应的HTTP响应字符串
    switch (code) {
    case 200:
        return "HTTP/1.0 200 OK\r\n\r\n";
    case 400:
        return "HTTP/1.0 400 Bad Request\r\n\r\n";
    case 403:
        return "HTTP/1.0 403 Forbidden\r\n\r\n";
    case 405:
        # 返回405状态码对应的响应字符串，包括允许的方法
        return "\
HTTP/1.0 405 Method Not Allowed\r\n\
Allow: CONNECT, GET, HEAD, POST\r\n\
\r\n";
    case 413:
        return "HTTP/1.0 413 Request Entity Too Large\r\n\r\n";
    case 501:
        return "HTTP/1.0 501 Not Implemented\r\n\r\n";
    case 504:
        return "HTTP/1.0 504 Gateway Timeout\r\n\r\n";
    default:
        return "HTTP/1.0 500 Internal Server Error\r\n\r\n";
    }

    return NULL;
}

# 检查身份验证信息是否有效
static int check_auth(const struct http_request *request,
    const struct http_credentials *credentials, int *stale)
{
    # 如果没有设置代理身份验证信息，则直接返回成功
    if (o.proxy_auth == NULL)
        return 1;
    # 将指针 *stale 的值设置为 0
    *stale = 0;

    # 如果认证方案是基本认证
    if (credentials->scheme == AUTH_BASIC) {
        # 声明一个指针变量 expected
        char *expected;
        # 声明一个整型变量 cmp
        int cmp;

        # 如果基本认证的用户名密码为空，则返回 0
        if (credentials->u.basic == NULL)
            return 0;

        # 对接收到的密码不进行解码，而是对期望的密码进行编码，然后比较编码后的字符串
        expected = b64enc((unsigned char *) o.proxy_auth, strlen(o.proxy_auth));
        cmp = strcmp(expected, credentials->u.basic);
        free(expected);

        # 返回比较结果是否为 0
        return cmp == 0;
    }
#if HAVE_HTTP_DIGEST
    # 如果使用的是 HTTP 摘要认证
    else if (credentials->scheme == AUTH_DIGEST) {
        # 定义变量
        char *username, *password;
        char *proxy_auth;
        struct timeval nonce_tv, now;
        int nonce_age;
        int ret;

        /* Split up the proxy auth argument. */
        # 复制代理认证信息
        proxy_auth = Strdup(o.proxy_auth);
        # 获取用户名和密码
        username = proxy_auth;
        password = strchr(proxy_auth, ':');
        # 如果没有找到密码，则释放内存并返回 0
        if (password == NULL) {
            free(proxy_auth);
            return 0;
        }
        *password++ = '\0';
        # 检查摘要认证的用户名和密码是否正确
        ret = http_digest_check_credentials(username, "Ncat", password,
            request->method, credentials);
        free(proxy_auth);

        # 如果认证失败，则返回 0
        if (!ret)
            return 0;

        /* The nonce checks out as one we issued and it matches what we expect
           given the credentials. Now check if it's too old. */
        # 检查摘要认证的随机数是否过期
        if (credentials->u.digest.nonce == NULL
            || http_digest_nonce_time(credentials->u.digest.nonce, &nonce_tv) == -1)
            return 0;
        gettimeofday(&now, NULL);
        if (TIMEVAL_AFTER(nonce_tv, now))
            return 0;
        nonce_age = TIMEVAL_SEC_SUBTRACT(now, nonce_tv);

        # 如果随机数过期，则返回 0
        if (nonce_age > HTTP_DIGEST_NONCE_EXPIRY) {
            if (o.verbose)
                loguser("Nonce is %d seconds old; rejecting.\n", nonce_age);
            *stale = 1;
            return 0;
        }

        /* To prevent replays, here we should additionally check against a list
           of recently used nonces, where "recently used nonce" is one that has
           been used to successfully authenticate within the last
           HTTP_DIGEST_NONCE_EXPIRY seconds. (Older than that and we don't need
           to keep it in the list, because the expiry test above will catch it.
           This isn't supported because the fork-and-process architecture of the
           proxy server makes it hard for us to change state in the parent
           process from here in the child. */

        # 防止重放攻击，这里应该对最近使用的随机数列表进行额外检查
        return 1;
    }
#endif
    else {
        return 0;
    }
    # 代码块结束
# 闭合前面的函数定义
}
```