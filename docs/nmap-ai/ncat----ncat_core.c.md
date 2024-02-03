# `nmap\ncat\ncat_core.c`

```cpp
/* $Id$ */

#include "ncat.h"  // 包含自定义的头文件 ncat.h
#include "util.h"  // 包含自定义的头文件 util.h
#include "sys_wrap.h"  // 包含自定义的头文件 sys_wrap.h

#ifndef WIN32  // 如果不是在 Windows 平台下编译
#include <unistd.h>  // 包含系统头文件 unistd.h
#include <netdb.h>  // 包含系统头文件 netdb.h
#include <sys/socket.h>  // 包含系统头文件 sys/socket.h
#include <netinet/in.h>  // 包含系统头文件 netinet/in.h
#include <arpa/inet.h>  // 包含系统头文件 arpa/inet.h
#endif
#include <stdlib.h>  // 包含系统头文件 stdlib.h
#include <string.h>  // 包含系统头文件 string.h
#include <stdio.h>  // 包含系统头文件 stdio.h
#include <errno.h>  // 包含系统头文件 errno.h
#include <fcntl.h>  // 包含系统头文件 fcntl.h
#include <ctype.h>  // 包含系统头文件 ctype.h
#include <time.h>  // 包含系统头文件 time.h

/* Only two for now because we might have to listen on IPV4 and IPV6 */
union sockaddr_u listenaddrs[NUM_LISTEN_ADDRS];  // 定义一个数组用于存放监听地址
int num_listenaddrs = 0;  // 初始化监听地址数量为 0

union sockaddr_u srcaddr;  // 定义一个联合体用于存放源地址
size_t srcaddrlen;  // 定义一个变量用于存放源地址的长度

struct sockaddr_list *targetaddrs;  // 定义一个指向 sockaddr_list 结构体的指针用于存放目标地址

/* Global options structure. */
struct options o;  // 定义一个全局的选项结构体 o

/* The time the program was started, for exit statistics in connect mode. */
struct timeval start_time;  // 定义一个结构体用于存放程序启动的时间，用于连接模式下的退出统计

/* Initializes global options to their default values. */
void options_init(void)  // 初始化全局选项的默认值的函数
{
    o.verbose = 0;  // 设置 verbose 为 0
    o.debug = 0;  // 设置 debug 为 0
    o.target = NULL;  // 设置 target 为 NULL
    o.af = AF_UNSPEC;  // 设置 af 为 AF_UNSPEC
    o.proto = IPPROTO_TCP;  // 设置 proto 为 IPPROTO_TCP
    o.broker = 0;  // 设置 broker 为 0
    o.listen = 0;  // 设置 listen 为 0
    o.keepopen = 0;  // 设置 keepopen 为 0
    o.sendonly = 0;  // 设置 sendonly 为 0
    o.recvonly = 0;  // 设置 recvonly 为 0
    o.noshutdown = 0;  // 设置 noshutdown 为 0
    o.telnet = 0;  // 设置 telnet 为 0
    o.linedelay = 0;  // 设置 linedelay 为 0
    o.chat = 0;  // 设置 chat 为 0
    o.nodns = 0;  // 设置 nodns 为 0
    o.normlog = NULL;  // 设置 normlog 为 NULL
    o.hexlog = NULL;  // 设置 hexlog 为 NULL
    o.normlogfd = -1;  // 设置 normlogfd 为 -1
    o.hexlogfd = -1;  // 设置 hexlogfd 为 -1
    o.append = 0;  // 设置 append 为 0
    o.idletimeout = 0;  // 设置 idletimeout 为 0
    o.crlf = 0;  // 设置 crlf 为 0
    o.allow = 0;  // 设置 allow 为 0
    o.deny = 0;  // 设置 deny 为 0
    o.allowset = addrset_new();  // 调用 addrset_new() 函数初始化 allowset
    o.denyset = addrset_new();  // 调用 addrset_new() 函数初始化 denyset
    o.httpserver = 0;  // 设置 httpserver 为 0

    o.nsock_engine = 0;  // 设置 nsock_engine 为 0

    o.test = 0;  // 设置 test 为 0

    o.numsrcrtes = 0;  // 设置 numsrcrtes 为 0
    o.srcrteptr = 4;  // 设置 srcrteptr 为 4

    o.conn_limit = -1;  /* Unset. */  // 设置 conn_limit 为 -1，表示未设置
    o.conntimeout = DEFAULT_CONNECT_TIMEOUT;  // 设置 conntimeout 为默认连接超时时间

    o.cmdexec = NULL;  // 设置 cmdexec 为 NULL
    o.execmode = EXEC_PLAIN;  // 设置 execmode 为 EXEC_PLAIN
    o.proxy_auth = NULL;  // 设置 proxy_auth 为 NULL
    o.proxytype = NULL;  // 设置 proxytype 为 NULL
    o.proxyaddr = NULL;  // 设置 proxyaddr 为 NULL
    o.proxydns = PROXYDNS_REMOTE;  // 设置 proxydns 为 PROXYDNS_REMOTE
    o.zerobyte = 0;  // 设置 zerobyte 为 0

#ifdef HAVE_OPENSSL
    o.ssl = 0;  // 设置 ssl 为 0
    o.sslcert = NULL;  // 设置 sslcert 为 NULL
    o.sslkey = NULL;  // 设置 sslkey 为 NULL
    o.sslverify = 0;  // 设置 sslverify 为 0
    o.ssltrustfile = NULL;  // 设置 ssltrustfile 为 NULL
    o.sslciphers = NULL;  // 设置 sslciphers 为 NULL
    o.sslservername = NULL;  // 设置 sslservername 为 NULL
    o.sslalpn = NULL;  // 设置 sslalpn 为 NULL
#endif
}
/* Internal helper for resolve and resolve_numeric. addl_flags is ored into
   hints.ai_flags, so you can add AI_NUMERICHOST.
   sl is a pointer to first element of sockaddr linked list, which is always
   statically allocated. Next list elements are dynamically allocated.
   If multiple_addrs is false then only first address is returned. */
static int resolve_internal(const char *hostname, unsigned short port,
    struct sockaddr_list *sl, int af, int addl_flags, int multiple_addrs)
{
    struct addrinfo hints;  // 定义 addrinfo 结构体变量 hints，用于设置地址解析的参数
    struct addrinfo *result;  // 定义 addrinfo 结构体指针 result，用于存储解析结果
    struct addrinfo *next;  // 定义 addrinfo 结构体指针 next，用于遍历解析结果链表
    struct sockaddr_list **item_ptr = &sl;  // 定义 sockaddr_list 结构体指针的指针 item_ptr，指向 sl 的地址
    struct sockaddr_list *new_item;  // 定义 sockaddr_list 结构体指针 new_item，用于存储新的地址信息
    char portbuf[16];  // 定义字符数组 portbuf，用于存储端口号的字符串形式
    int rc;  // 定义整型变量 rc，用于存储函数返回值

    ncat_assert(hostname != NULL);  // 断言 hostname 不为空

    memset(&hints, 0, sizeof(hints));  // 将 hints 结构体变量清零
    hints.ai_family = af;  // 设置地址族
    hints.ai_socktype = SOCK_DGRAM;  // 设置套接字类型
    hints.ai_flags |= addl_flags;  // 将额外的标志位或运算到 hints.ai_flags 中

    /* Make the port number a string to give to getaddrinfo. */
    rc = Snprintf(portbuf, sizeof(portbuf), "%hu", port);  // 将端口号转换为字符串形式
    ncat_assert(rc >= 0 && (size_t) rc < sizeof(portbuf));  // 断言转换成功

    rc = getaddrinfo(hostname, portbuf, &hints, &result);  // 解析主机名和端口号，获取地址信息
    if (rc != 0)
        return rc;  // 如果解析失败，返回错误码
    if (result == NULL)
        return EAI_NONAME;  // 如果解析结果为空，返回 EAI_NONAME 错误码
    ncat_assert(result->ai_addrlen > 0 && result->ai_addrlen <= (int) sizeof(struct sockaddr_storage));  // 断言地址长度合法
    for (next = result; next != NULL; next = next->ai_next) {  // 遍历解析结果链表
        if (*item_ptr == NULL)  // 如果当前地址信息为空
        {
            *item_ptr = (struct sockaddr_list *)safe_malloc(sizeof(struct sockaddr_list));  // 分配内存给地址信息
            (**item_ptr).next = NULL;  // 设置下一个地址信息为空
        }
        new_item = *item_ptr;  // 将当前地址信息赋值给 new_item
        new_item->addrlen = next->ai_addrlen;  // 设置地址长度
        memcpy(&new_item->addr.storage, next->ai_addr, next->ai_addrlen);  // 复制地址信息
        if (!multiple_addrs)  // 如果不需要多个地址
            break;  // 跳出循环
        item_ptr = &new_item->next;  // 更新地址信息指针
    }
    freeaddrinfo(result);  // 释放地址信息内存

    return 0;  // 返回成功
}
# 使用 getaddrinfo 解析给定的主机名或 IP 地址，并将第一个结果（如果有）存储在 *ss 和 *sslen 中。如果不关心端口的值，将在 *ss 中设置为 0。af 可能是 AF_UNSPEC，在这种情况下，getaddrinfo 可能返回 IPv4 和 IPv6 结果；哪个是第一个取决于系统配置。成功返回 0，失败返回 getaddrinfo 返回的代码（适合传递给 gai_strerror）。当此函数返回 0 时，*ss 和 *sslen 总是定义的。

# 如果全局变量 o.nodns 为 true，则不使用 DNS 解析任何名称。
int resolve(const char *hostname, unsigned short port,
    struct sockaddr_storage *ss, size_t *sslen, int af)
{
    int flags;
    struct sockaddr_list sl;
    int result;

    flags = 0;
    if (o.nodns)
        flags |= AI_NUMERICHOST;

    result = resolve_internal(hostname, port, &sl, af, flags, 0);
    *ss = sl.addr.storage;
    *sslen = sl.addrlen;
    return result;
}

# 使用 getaddrinfo 解析给定的主机名或 IP 地址，并将第一个结果（如果有）存储在 *ss 和 *sslen 中。如果不关心端口的值，将在 *ss 中设置为 0。af 可能是 AF_UNSPEC，在这种情况下，getaddrinfo 可能返回 IPv4 和 IPv6 结果；哪个是第一个取决于系统配置。成功返回 0，失败返回 getaddrinfo 返回的代码（适合传递给 gai_strerror）。当此函数返回 0 时，*ss 和 *sslen 总是定义的。

# 只有当全局变量 o.proxydns 包括 PROXYDNS_LOCAL 时，才使用 DNS 解析主机名。
int proxyresolve(const char *hostname, unsigned short port,
    struct sockaddr_storage *ss, size_t *sslen, int af)
{
    int flags;
    struct sockaddr_list sl;
    int result;

    flags = 0;
    if (!(o.proxydns & PROXYDNS_LOCAL))
        flags |= AI_NUMERICHOST;

    result = resolve_internal(hostname, port, &sl, af, flags, 0);
    *ss = sl.addr.storage;
    # 将变量 sl.addrlen 的值赋给变量 sslen
    *sslen = sl.addrlen;
    # 返回 result 变量的值
    return result;
/* 通过 getaddrinfo 解析给定的主机名或 IP 地址，并将所有结果存储到一个链表中。
   其余行为与 resolve() 相同。 */
int resolve_multi(const char *hostname, unsigned short port,
    struct sockaddr_list *sl, int af)
{
    int flags;

    flags = 0;
    // 如果禁用 DNS 解析，则设置 AI_NUMERICHOST 标志
    if (o.nodns)
        flags |= AI_NUMERICHOST;

    // 调用 resolve_internal 函数进行解析
    return resolve_internal(hostname, port, sl, af, flags, 1);
}

// 释放 sockaddr_list 结构体链表
void free_sockaddr_list(struct sockaddr_list *sl)
{
    struct sockaddr_list *current, *next = sl;
    while (next != NULL) {
        current = next;
        next = current->next;
        free(current);
    }
}

// 关闭 fdinfo 结构体中的文件描述符
int fdinfo_close(struct fdinfo *fdn)
{
#ifdef HAVE_OPENSSL
    // 如果启用了 OpenSSL 并且 fdinfo 结构体中有 SSL 对象，则关闭 SSL 连接
    if (o.ssl && fdn->ssl != NULL) {
        SSL_shutdown(fdn->ssl);
        SSL_free(fdn->ssl);
        fdn->ssl = NULL;
    }
#endif

    // 调用系统函数关闭文件描述符
    return close(fdn->fd);
}

/* 在 fdinfo 上执行 recv 操作，不产生其他副作用。 */
int fdinfo_recv(struct fdinfo *fdn, char *buf, size_t size)
{
    int n;
#ifdef HAVE_OPENSSL
    int err = SSL_ERROR_NONE;
    // 如果启用了 OpenSSL 并且 fdinfo 结构体中有 SSL 对象
    if (o.ssl && fdn->ssl)
    {
        do {
            // 调用 SSL_read 读取数据
            n = SSL_read(fdn->ssl, buf, size);
            /* SSL_read 在某些情况下返回 <=0，比如重新协商。在这些情况下，SSL_get_error
             * 返回 SSL_ERROR_WANT_{READ,WRITE}，我们应该再次尝试 SSL_read。 */
            err = (n <= 0) ? SSL_get_error(fdn->ssl, n) : SSL_ERROR_NONE;
        } while (err == SSL_ERROR_WANT_READ || err == SSL_ERROR_WANT_WRITE);
        switch (err) {
            case SSL_ERROR_NONE:
                break;
            case SSL_ERROR_ZERO_RETURN:
                fdn->lasterr = EOF;
                break;
            default:
                fdn->lasterr = err;
                logdebug("SSL_read error on %d: %s\n", fdn->fd, ERR_error_string(err, NULL));
                break;
        }
        return n;
    }
#endif
    // 调用系统函数 recv 读取数据
    n = recv(fdn->fd, buf, size, 0);
    if (n == 0)
        fdn->lasterr = EOF;
}
    # 如果 n 小于 0，则将最后的错误信息设置为套接字错误码
    else if (n < 0)
        fdn->lasterr = socket_errno();
    # 返回 n 的值
    return n;
}

// 检查是否有待处理的数据在套接字缓冲区中
int fdinfo_pending(struct fdinfo *fdn)
{
#ifdef HAVE_OPENSSL
    // 如果使用了 OpenSSL 并且存在 SSL 对象，则返回 SSL_pending 的结果
    if (o.ssl && fdn->ssl)
        return SSL_pending(fdn->ssl);
#endif
    // 否则返回 0
    return 0;
}

/* 从客户端套接字读取数据到 buf 中，返回读取的字节数，出错时返回 -1。
   这个函数处理延迟、Telnet 协商和日志记录。

   如果有更多未处理的数据，不会被 select 发现，就在 *pending 中存储 1，否则存储 0。
   调用者必须循环处理读取的数据，直到 *pending 为 false。原因是此函数可能调用的 SSL_read
   函数会从套接字缓冲区中取出数据（因此 select 可能不会指示套接字可读），并将其保留在自己的缓冲区中。
   *pending 保存调用 SSL_pending 的结果。参见 http://www.mail-archive.com/openssl-dev@openssl.org/msg24324.html。 */
int ncat_recv(struct fdinfo *fdn, char *buf, size_t size, int *pending)
{
    int n;

    *pending = 0;

    n = fdinfo_recv(fdn, buf, size);

    if (n <= 0)
        return n;

    if (o.linedelay)
        ncat_delay_timer(o.linedelay);
    if (o.telnet)
        dotelnet(fdn->fd, (unsigned char *) buf, n);
    ncat_log_recv(buf, n);

    /* SSL 可以缓存我们的输入，所以再次调用 select() 对我们来说不一定有效。
       向调用者指示必须再次调用此函数以获取更多数据。 */
    *pending = fdinfo_pending(fdn);

    return n;
}

/* 在 fdinfo 上进行发送，不进行任何日志记录或其他副作用。 */
int fdinfo_send(struct fdinfo *fdn, const char *buf, size_t size)
{
    int n;
#ifdef HAVE_OPENSSL
    int err = SSL_ERROR_NONE;
    if (o.ssl && fdn->ssl != NULL)
    {
        // 使用 do-while 循环来执行 SSL_write 操作，直到返回值大于 0
        do {
            // 调用 SSL_write 函数向 SSL 连接写入数据
            n = SSL_write(fdn->ssl, buf, size);
            /* SSL_write 在某些情况下返回 <=0，比如重新协商。在这些情况下，SSL_get_error 返回 SSL_ERROR_WANT_{READ,WRITE}，我们应该再次尝试 SSL_write。 */
            // 如果 SSL_write 返回值 <= 0，则使用 SSL_get_error 获取错误码，否则为 SSL_ERROR_NONE
            err = (n <= 0) ? SSL_get_error(fdn->ssl, n) : SSL_ERROR_NONE;
        } while (err == SSL_ERROR_WANT_READ || err == SSL_ERROR_WANT_WRITE);
        // 如果 err 不等于 SSL_ERROR_NONE，则记录错误信息和错误码
        if (err != SSL_ERROR_NONE) {
            fdn->lasterr = err;
            logdebug("SSL_write error on %d: %s\n", fdn->fd, ERR_error_string(err, NULL));
        }
        // 返回 SSL_write 的结果
        return n;
    }
#endif
    # 发送数据到文件描述符所指向的套接字
    n = send(fdn->fd, buf, size, 0);
    # 如果发送失败，记录错误码
    if (n <= 0)
        fdn->lasterr = socket_errno();
    # 返回发送的字节数
    return n;
}

/* 如果我们发送大量数据，可能会暂时耗尽发送空间，并在发送时得到 EAGAIN 错误。
   临时将套接字转换为阻塞模式，进行发送，然后再次将其转换为非阻塞模式。
   假设套接字一开始就是非阻塞模式；它的副作用是在返回时保持套接字为非阻塞模式。 */
static int blocking_fdinfo_send(struct fdinfo *fdn, const char *buf, size_t size)
{
    int ret;

    # 将套接字设置为阻塞模式
    block_socket(fdn->fd);
    # 进行发送操作
    ret = fdinfo_send(fdn, buf, size);
    # 将套接字设置为非阻塞模式
    unblock_socket(fdn->fd);

    return ret;
}

int ncat_send(struct fdinfo *fdn, const char *buf, size_t size)
{
    int n;

    # 如果只接收数据，则直接返回数据大小
    if (o.recvonly)
        return size;

    # 调用阻塞发送函数
    n = blocking_fdinfo_send(fdn, buf, size);
    # 如果发送失败，直接返回错误码
    if (n <= 0)
        return n;

    # 记录发送日志
    ncat_log_send(buf, size);

    return n;
}

/* 向 fds 中的所有描述符广播消息。如果发送失败，则返回 -1。 */
int ncat_broadcast(fd_set *fds, const fd_list_t *fdlist, const char *msg, size_t size)
{
    struct fdinfo *fdn;
    int i, ret;

    # 如果只接收数据，则直接返回数据大小
    if (o.recvonly)
        return size;

    ret = 0;
    for (i = 0; i < fdlist->nfds; i++) {
        fdn = &fdlist->fds[i];
        # 如果描述符不在 fds 中，则跳过
        if (!checked_fd_isset(fdn->fd, fds))
            continue;

        # 调用阻塞发送函数
        if (blocking_fdinfo_send(fdn, msg, size) <= 0) {
            # 如果发送失败，记录错误信息，并将返回值设为 -1
            if (o.debug > 1)
                logdebug("Error sending to fd %d: %s.\n", fdn->fd, socket_strerror(fdn->lasterr));
            ret = -1;
        }
    }

    # 记录发送日志
    ncat_log_send(msg, size);

    return ret;
}

/* 进行 Telnet 协议的 WILL/WONT DO/DONT 协商 */
void dotelnet(int s, unsigned char *buf, size_t bufsiz)
{
    unsigned char *end = buf + bufsiz, *p;
    unsigned char tbuf[3];
    for (p = buf; buf < end; p++) {
        // 从缓冲区开始遍历直到结束
        if (*p != 255) /* IAC */
            // 如果当前字符不是255，则跳出循环
            break;

        tbuf[0] = *p++;

        /* Answer DONT for WILL or WONT */
        // 如果下一个字符是251或252，则回答DONT
        if (*p == 251 || *p == 252)
            tbuf[1] = 254;

        /* Answer WONT for DO or DONT */
        // 如果下一个字符是253或254，则回答WONT
        else if (*p == 253 || *p == 254)
            tbuf[1] = 252;

        tbuf[2] = *++p;

        // 发送tbuf中的内容
        send(s, (const char *) tbuf, 3, 0);
    }
/* sleep(), usleep(), msleep(), Sleep() -- all together now, "portability".
 *
 * There is no upper or lower limit to the delayval, so if you pass in a short
 * length of time <100ms, then you're likely going to get odd results.
 * This is because the Linux timeslice is 10ms-200ms. So don't expect
 * it to return for at least that long.
 *
 * Block until the specified time has elapsed, then return 1.
 */
// 定义一个延时函数，根据传入的延时值进行阻塞
int ncat_delay_timer(int delayval)
{
    // 定义时间结构体
    struct timeval s;

    // 将延时值转换为秒和微秒
    s.tv_sec = delayval / 1000;
    s.tv_usec = (delayval % 1000) * (long) 1000;

    // 使用 select 函数进行阻塞
    select(0, NULL, NULL, NULL, &s);
    // 返回 1 表示延时完成
    return 1;
}

// 将数据以十六进制格式记录到日志文件
static int ncat_hexdump(int logfd, const char *data, int len);

// 记录发送的数据到日志文件
void ncat_log_send(const char *data, size_t len)
{
    // 如果普通日志文件描述符有效，则写入数据
    if (o.normlogfd != -1)
        Write(o.normlogfd, data, len);

    // 如果十六进制日志文件描述符有效，则调用 ncat_hexdump 函数记录十六进制数据
    if (o.hexlogfd != -1)
        ncat_hexdump(o.hexlogfd, data, len);
}

// 记录接收的数据到日志文件
void ncat_log_recv(const char *data, size_t len)
{
    /* Currently the log formats don't distinguish sends and receives. */
    // 目前的日志格式不区分发送和接收，直接调用发送日志函数记录接收的数据
    ncat_log_send(data, len);
}

/* Convert session data to a neat hexdump logfile */
// 将会话数据转换为整洁的十六进制日志文件
static int ncat_hexdump(int logfd, const char *data, int len)
{
  char *str = NULL;
  // 调用 hexdump 函数将数据转换为十六进制字符串
  str = hexdump((u8 *) data, len);
  if (str) {
    // 如果转换成功，则将字符串写入日志文件
    Write(logfd, str, strlen(str));
    free(str);
  }
  else {
    return 0;
  }
  return 1;
}

/* this function will return in what format the target
 * host is specified. It will return:
 * 1 - for ipv4,
 * 2 - for ipv6,
 * -1 - for hostname
 * this has to work even if there is no IPv6 support on
 * local system, proxy may support it.
 */
// 判断目标主机的地址格式，返回 1 表示 IPv4，返回 2 表示 IPv6，返回 -1 表示主机名
int getaddrfamily(const char *addr)
{
    int ret;
    struct addrinfo hint, *info = 0;

    // 如果地址中包含冒号，则判断为 IPv6
    if (strchr(addr,':'))
      return 2;

    // 初始化地址信息结构体
    zmem(&hint,sizeof(hint));
    hint.ai_family = AF_UNSPEC;
    hint.ai_flags = AI_NUMERICHOST;
    // 获取地址信息
    ret = getaddrinfo(addr, 0, &hint, &info);
    if (ret)
        return -1;
    freeaddrinfo(info);
    return 1;
}

// 设置环境
void setup_environment(struct fdinfo *info)
{
    union sockaddr_u su;
    # 定义一个字符数组，用于存储 IP 地址，长度为 INET6_ADDRSTRLEN
    char ip[INET6_ADDRSTRLEN];
    # 定义一个字符数组，用于存储端口号，长度为 16
    char port[16];
    # 定义一个变量，用于存储地址结构体的长度
    socklen_t alen = sizeof(su);

    # 获取与指定套接字关联的远程端点的地址信息，并存储在 su 中
    if (getpeername(info->fd, &su.sockaddr, &alen) != 0) {
        # 如果获取失败，则打印错误信息并退出
        bye("getpeername failed: %s", socket_strerror(socket_errno()));
    }
#ifdef HAVE_SYS_UN_H
    // 如果支持 UNIX 域套接字
    if (su.sockaddr.sa_family == AF_UNIX) {
        // 设置环境变量 NCAT_REMOTE_ADDR 为 localhost，保持向后兼容性
        setenv_portable("NCAT_REMOTE_ADDR", "localhost");
        // 设置环境变量 NCAT_REMOTE_PORT 为空字符串
        setenv_portable("NCAT_REMOTE_PORT", "");
    } else
#endif
#ifdef HAVE_LINUX_VM_SOCKETS_H
    // 如果支持 Linux VM 套接字
    if (su.sockaddr.sa_family == AF_VSOCK) {
        // 创建一个字符数组用于存储转换后的无符号整数
        char char_u32[11];
        // 将 su.vm.svm_cid 转换为字符串并存储到 char_u32 中
        snprintf(char_u32, sizeof(char_u32), "%u", su.vm.svm_cid);
        // 设置环境变量 NCAT_REMOTE_ADDR 为 char_u32
        setenv_portable("NCAT_REMOTE_ADDR", char_u32);
        // 将 su.vm.svm_port 转换为字符串并存储到 char_u32 中
        snprintf(char_u32, sizeof(char_u32), "%u", su.vm.svm_port);
        // 设置环境变量 NCAT_REMOTE_PORT 为 char_u32
        setenv_portable("NCAT_REMOTE_PORT", char_u32);
    } else
#endif
    // 获取远程主机的 IP 地址和端口号
    if (getnameinfo((struct sockaddr *)&su, alen, ip, sizeof(ip),
            port, sizeof(port), NI_NUMERICHOST | NI_NUMERICSERV) == 0) {
        // 设置环境变量 NCAT_REMOTE_ADDR 为远程主机的 IP 地址
        setenv_portable("NCAT_REMOTE_ADDR", ip);
        // 设置环境变量 NCAT_REMOTE_PORT 为远程主机的端口号
        setenv_portable("NCAT_REMOTE_PORT", port);
    } else {
        // 如果获取失败，则输出错误信息并退出程序
        bye("getnameinfo failed: %s", socket_strerror(socket_errno()));
    }

    // 获取本地套接字的名称
    if (getsockname(info->fd, (struct sockaddr *)&su, &alen) < 0) {
        // 如果获取失败，则输出错误信息并退出程序
        bye("getsockname failed: %s", socket_strerror(socket_errno()));
    }
#ifdef HAVE_SYS_UN_H
    // 如果支持 UNIX 域套接字
    if (su.sockaddr.sa_family == AF_UNIX) {
        // 设置环境变量 NCAT_LOCAL_ADDR 为 localhost，保持向后兼容性
        setenv_portable("NCAT_LOCAL_ADDR", "localhost");
        // 设置环境变量 NCAT_LOCAL_PORT 为空字符串
        setenv_portable("NCAT_LOCAL_PORT", "");
    } else
#endif
#ifdef HAVE_LINUX_VM_SOCKETS_H
    // 如果支持 Linux VM 套接字
    if (su.sockaddr.sa_family == AF_VSOCK) {
        // 创建一个字符数组用于存储转换后的无符号整数
        char char_u32[11];
        // 将 su.vm.svm_cid 转换为字符串并存储到 char_u32 中
        snprintf(char_u32, sizeof(char_u32), "%u", su.vm.svm_cid);
        // 设置环境变量 NCAT_LOCAL_ADDR 为 char_u32
        setenv_portable("NCAT_LOCAL_ADDR", char_u32);
        // 将 su.vm.svm_port 转换为字符串并存储到 char_u32 中
        snprintf(char_u32, sizeof(char_u32), "%u", su.vm.svm_port);
        // 设置环境变量 NCAT_LOCAL_PORT 为 char_u32
        setenv_portable("NCAT_LOCAL_PORT", char_u32);
    } else
#endif
    // 获取本地主机的 IP 地址和端口号
    if (getnameinfo((struct sockaddr *)&su, alen, ip, sizeof(ip),
            port, sizeof(port), NI_NUMERICHOST | NI_NUMERICSERV) == 0) {
        // 设置环境变量 NCAT_LOCAL_ADDR 为本地主机的 IP 地址
        setenv_portable("NCAT_LOCAL_ADDR", ip);
        // 设置环境变量 NCAT_LOCAL_PORT 为本地主机的端口号
        setenv_portable("NCAT_LOCAL_PORT", port);
    } else {
        # 如果getnameinfo失败，则打印错误信息并退出
        bye("getnameinfo failed: %s", socket_strerror(socket_errno()));
    }

    # 根据协议类型设置环境变量NCAT_PROTO
    switch(o.proto) {
        case IPPROTO_TCP:
            setenv_portable("NCAT_PROTO", "TCP");
            break;
        case IPPROTO_SCTP:
            setenv_portable("NCAT_PROTO", "SCTP");
            break;
        case IPPROTO_UDP:
            setenv_portable("NCAT_PROTO", "UDP");
            break;
    }
# 闭合前面的函数定义
```