# `nmap\libpcap\sockutils.c`

```cpp
/*
 * 版权声明，版权所有
 * NetGroup, 意大利都灵理工大学（2002-2003）
 * 禁止未经许可的源代码和二进制形式的再发布和使用
 *
 * 在满足以下条件的情况下，允许对源代码和二进制形式进行再发布和使用：
 *
 * 1. 源代码的再发布必须保留上述版权声明、条件列表和以下免责声明。
 * 2. 二进制形式的再发布必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明。
 * 3. 不得使用都灵理工大学的名称或其贡献者的名称，以认可或推广从本软件衍生的产品，除非事先得到特定的书面许可。
 *
 * 本软件由版权所有者和贡献者按原样提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。在任何情况下，无论是在合同、严格责任还是侵权行为（包括疏忽或其他情况）的任何理论下，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性的损害（包括但不限于替代商品或服务的采购、使用损失、数据或利润的损失或业务中断）负责，即使已被告知可能发生此类损害。
 *
 */
#ifdef HAVE_CONFIG_H
#include <config.h>
#endif
/*
 * \file sockutils.c
 *
 * The goal of this file is to provide a common set of primitives for socket
 * manipulation.
 *
 * Although the socket interface defined in the RFC 2553 (and its updates)
 * is excellent, there are still differences between the behavior of those
 * routines on UN*X and Windows, and between UN*Xes.
 *
 * These calls provide an interface similar to the socket interface, but
 * that hides the differences between operating systems.  It does not
 * attempt to significantly improve on the socket interface in other
 * ways.
 */

#include "ftmacros.h"

#include <string.h>
#include <errno.h>    /* for the errno variable */
#include <stdio.h>    /* for the stderr file */
#include <stdlib.h>    /* for malloc() and free() */
#include <limits.h>    /* for INT_MAX */

#include "pcap-int.h"

#include "sockutils.h"
#include "portability.h"

#ifdef _WIN32
  /*
   * Winsock initialization.
   *
   * Ask for Winsock 2.2.
   */
  #define WINSOCK_MAJOR_VERSION 2
  #define WINSOCK_MINOR_VERSION 2

  static int sockcount = 0;    /*!< Variable that allows calling the WSAStartup() only one time */
#endif

/* Some minor differences between UNIX and Win32 */
#ifdef _WIN32
  #define SHUT_WR SD_SEND    /* The control code for shutdown() is different in Win32 */
#endif

/* Size of the buffer that has to keep error messages */
#define SOCK_ERRBUF_SIZE 1024

/* Constants; used in order to keep strings here */
#define SOCKET_NO_NAME_AVAILABLE "No name available"
#define SOCKET_NO_PORT_AVAILABLE "No port available"
#define SOCKET_NAME_NULL_DAD "Null address (possibly DAD Phase)"
/*
 * On UN*X, send() and recv() return ssize_t.
 *
 * On Windows, send() and recv() return an int.
 *
 *   With MSVC, there *is* no ssize_t.
 *
 *   With MinGW, there is an ssize_t type; it is either an int (32 bit)
 *   or a long long (64 bit).
 *
 * So, on Windows, if we don't have ssize_t defined, define it as an
 * int, so we can use it, on all platforms, as the type of variables
 * that hold the return values from send() and recv().
 */
#if defined(_WIN32) && !defined(_SSIZE_T_DEFINED)
typedef int ssize_t;
#endif

/****************************************************
 *                                                  *
 * Locally defined functions                        *
 *                                                  *
 ****************************************************/

static int sock_ismcastaddr(const struct sockaddr *saddr);

/****************************************************
 *                                                  *
 * Function bodies                                  *
 *                                                  *
 ****************************************************/

#ifdef FUZZING_BUILD_MODE_UNSAFE_FOR_PRODUCTION
const uint8_t *fuzzBuffer;
size_t fuzzSize;
size_t fuzzPos;

void sock_initfuzz(const uint8_t *Data, size_t Size) {
    fuzzPos = 0;
    fuzzSize = Size;
    fuzzBuffer = Data;
}

static int fuzz_recv(char *bufp, int remaining) {
    if (remaining > fuzzSize - fuzzPos) {
        remaining = fuzzSize - fuzzPos;
    }
    if (fuzzPos < fuzzSize) {
        memcpy(bufp, fuzzBuffer + fuzzPos, remaining);
    }
    fuzzPos += remaining;
    return remaining;
}
#endif

int sock_geterrcode(void)
{
#ifdef _WIN32
    return GetLastError();
#else
    return errno;
#endif
}

/*
 * Format an error message given an errno value (UN*X) or a Winsock error
 * (Windows).
 */
void sock_vfmterrmsg(char *errbuf, size_t errbuflen, int errcode,
    const char *fmt, va_list ap)
{
    if (errbuf == NULL)
        return;

#ifdef _WIN32
    # 调用函数pcap_vfmt_errmsg_for_win32_err，传入参数errbuf, errbuflen, errcode, fmt, ap
#else
    # 如果不满足条件，则执行以下代码
    pcap_vfmt_errmsg_for_errno(errbuf, errbuflen, errcode,
        fmt, ap);
#endif
}

# 格式化套接字错误消息
void sock_fmterrmsg(char *errbuf, size_t errbuflen, int errcode,
    const char *fmt, ...)
{
    va_list ap;

    # 初始化参数列表
    va_start(ap, fmt);
    # 调用格式化套接字错误消息的函数
    sock_vfmterrmsg(errbuf, errbuflen, errcode, fmt, ap);
    # 结束参数列表
    va_end(ap);
}

/*
 * 格式化最后一个套接字错误的错误消息
 */
void sock_geterrmsg(char *errbuf, size_t errbuflen, const char *fmt, ...)
{
    va_list ap;

    # 初始化参数列表
    va_start(ap, fmt);
    # 调用格式化套接字错误消息的函数
    sock_vfmterrmsg(errbuf, errbuflen, sock_geterrcode(), fmt, ap);
    # 结束参数列表
    va_end(ap);
}

/*
 * 错误类型
 *
 * 这些按照它们可能是“潜在”问题的可能性排序，
 * 因此，给定地址族中给定地址的低评级错误不应覆盖该族中另一个地址的高评级错误，
 * 高评级错误应覆盖低评级错误。
 */
typedef enum {
    SOCK_CONNERR,        /* 连接错误 */
    SOCK_HOSTERR,        /* 主机错误 */
    SOCK_NETERR,        /* 网络错误 */
    SOCK_AFNOTSUPERR,    /* 地址族不支持错误 */
    SOCK_UNKNOWNERR,    /* 未知错误 */
    SOCK_NOERR        /* 无错误 */
} sock_errtype;

static sock_errtype sock_geterrtype(int errcode)
{
    switch (errcode) {

#ifdef _WIN32
    case WSAECONNRESET:
    case WSAECONNABORTED:
    case WSAECONNREFUSED:
#else
    case ECONNRESET:
    case ECONNABORTED:
    case ECONNREFUSED:
#endif
        '''
         * 连接错误；这意味着问题可能是远程机器上没有设置服务器，
         * 或者已设置，但它只支持IPv4或IPv6，而我们正在尝试错误的地址族。
         *
         * 这些覆盖所有其他错误，因为它们表明，即使在另一个尝试中发生了其他问题，
         * 即使其他问题得到解决，这个问题可能也不会起作用。
         '''
        return (SOCK_CONNERR);

#ifdef _WIN32
    # 如果发生网络不可达错误，则执行以下代码
    case WSAENETUNREACH:
    # 如果发生连接超时错误，则执行以下代码
    case WSAETIMEDOUT:
    # 如果发生主机宕机错误，则执行以下代码
    case WSAEHOSTDOWN:
    # 如果发生主机不可达错误，则执行以下代码
    case WSAEHOSTUNREACH:
#else
    case ENETUNREACH:
    case ETIMEDOUT:
    case EHOSTDOWN:
    case EHOSTUNREACH:
#endif
        '''
         * 网络错误可能是特定于 IPv4、IPv6 或两者都存在。
         *
         * 不要覆盖连接错误，但覆盖其他所有错误。
         '''
        return (SOCK_HOSTERR);

#ifdef _WIN32
    case WSAENETDOWN:
    case WSAENETRESET:
#else
    case ENETDOWN:
    case ENETRESET:
#endif
        '''
         * 网络错误；这意味着我们不知道远程机器上是否设置了服务器，
         * 而且我们没有理由相信 IPv6 比 IPv4 更好或更差。
         *
         * 这些可能表示本地故障，例如接口已关闭。
         *
         * 不要覆盖连接错误或主机错误，但覆盖其他所有错误。
         '''
        return (SOCK_NETERR);

#ifdef _WIN32
    case WSAEAFNOSUPPORT:
#else
    case EAFNOSUPPORT:
#endif
        '''
         * "地址族不受支持" 可能意味着 "没有 IPv6 可用！"。
         *
         * 不要覆盖连接错误、主机错误或网络错误
         * (如果不支持该地址族，我们不应该得到任何这些错误)，
         * 但覆盖其他所有错误。
         '''
        return (SOCK_AFNOTSUPERR);

    default:
        '''
         * 其他任何错误。
         *
         * 不要覆盖任何错误。
         '''
        return (SOCK_UNKNOWNERR);
    }
}
/*
 * \brief This function initializes the socket mechanism if it hasn't
 * already been initialized or reinitializes it after it has been
 * cleaned up.
 *
 * On UN*Xes, it doesn't need to do anything; on Windows, it needs to
 * initialize Winsock.
 *
 * \param errbuf: a pointer to an user-allocated buffer that will contain
 * the complete error message. This buffer has to be at least 'errbuflen'
 * in length. It can be NULL; in this case no error message is supplied.
 *
 * \param errbuflen: length of the buffer that will contains the error.
 * The error message cannot be larger than 'errbuflen - 1' because the
 * last char is reserved for the string terminator.
 *
 * \return '0' if everything is fine, '-1' if some errors occurred. The
 * error message is returned in the buffer pointed to by 'errbuf' variable.
 */
#ifdef _WIN32
int sock_init(char *errbuf, int errbuflen)
{
    if (sockcount == 0)
    {
        WSADATA wsaData;            /* helper variable needed to initialize Winsock */

        if (WSAStartup(MAKEWORD(WINSOCK_MAJOR_VERSION,
            WINSOCK_MINOR_VERSION), &wsaData) != 0)
        {
            if (errbuf)
                snprintf(errbuf, errbuflen, "Failed to initialize Winsock\n");

            WSACleanup();

            return -1;
        }
    }

    sockcount++;
    return 0;
}
#else
int sock_init(char *errbuf _U_, int errbuflen _U_)
{
    /*
     * Nothing to do on UN*Xes.
     */
    return 0;
}
#endif

/*
 * \brief This function cleans up the socket mechanism if we have no
 * sockets left open.
 *
 * On UN*Xes, it doesn't need to do anything; on Windows, it needs
 * to clean up Winsock.
 *
 * \return No error values.
 */
void sock_cleanup(void)
{
#ifdef _WIN32
    sockcount--;

    if (sockcount == 0)
        WSACleanup();
#endif
}

/*
 * \brief It checks if the sockaddr variable contains a multicast address.
 *
 * \return '0' if the address is multicast, '-1' if it is not.
 */
static int sock_ismcastaddr(const struct sockaddr *saddr)
{
    # 如果地址族为 IPv4
    if (saddr->sa_family == PF_INET)
    {
        # 将地址结构转换为 IPv4 地址结构
        struct sockaddr_in *saddr4 = (struct sockaddr_in *) saddr;
        # 如果是 IPv4 组播地址，则返回 0
        if (IN_MULTICAST(ntohl(saddr4->sin_addr.s_addr))) return 0;
        # 否则返回 -1
        else return -1;
    }
    # 如果地址族为 IPv6
    else
    {
        # 将地址结构转换为 IPv6 地址结构
        struct sockaddr_in6 *saddr6 = (struct sockaddr_in6 *) saddr;
        # 如果是 IPv6 组播地址，则返回 0
        if (IN6_IS_ADDR_MULTICAST(&saddr6->sin6_addr)) return 0;
        # 否则返回 -1
        else return -1;
    }
}

// 定义一个结构体，包含地址信息、错误码和错误类型
struct addr_status {
    struct addrinfo *info;
    int errcode;
    sock_errtype errtype;
};

/*
 * 按照 IPv4 地址和 IPv6 地址进行排序
 */
static int compare_addrs_to_try_by_address_family(const void *a, const void *b)
{
    const struct addr_status *addr_a = (const struct addr_status *)a;
    const struct addr_status *addr_b = (const struct addr_status *)b;

    return addr_a->info->ai_family - addr_b->info->ai_family;
}

/*
 * 按照错误类型排序，同一错误类型下按照错误码排序，同一错误码下按照 IPv4 地址和 IPv6 地址排序
 */
static int compare_addrs_to_try_by_status(const void *a, const void *b)
{
    const struct addr_status *addr_a = (const struct addr_status *)a;
    const struct addr_status *addr_b = (const struct addr_status *)b;

    if (addr_a->errtype == addr_b->errtype)
    {
        if (addr_a->errcode == addr_b->errcode)
        {
            return addr_a->info->ai_family - addr_b->info->ai_family;
        }
        return addr_a->errcode - addr_b->errcode;
    }

    return addr_a->errtype - addr_b->errtype;
}

// 创建套接字
static SOCKET sock_create_socket(struct addrinfo *addrinfo, char *errbuf,
    int errbuflen)
{
    SOCKET sock;
#ifdef SO_NOSIGPIPE
    int on = 1;
#endif

    // 使用地址信息创建套接字
    sock = socket(addrinfo->ai_family, addrinfo->ai_socktype,
        addrinfo->ai_protocol);
    if (sock == INVALID_SOCKET)
    {
        // 如果创建套接字失败，获取错误信息并返回无效套接字
        sock_geterrmsg(errbuf, errbuflen, "socket() failed");
        return INVALID_SOCKET;
    }

    /*
     * 如果有 SO_NOSIGPIPE 选项，禁用 SIGPIPE 信号
     */
#ifdef SO_NOSIGPIPE
    if (setsockopt(sock, SOL_SOCKET, SO_NOSIGPIPE, (char *)&on,
        sizeof (int)) == -1)
    {
        // 如果设置 SO_NOSIGPIPE 失败，获取错误信息并关闭套接字，返回无效套接字
        sock_geterrmsg(errbuf, errbuflen,
            "setsockopt(SO_NOSIGPIPE) failed");
        closesocket(sock);
        return INVALID_SOCKET;
    # 闭合前面的代码块，表示函数定义结束
#endif
    return sock;
}

/*
 * \brief It initializes a network connection both from the client and the server side.
 *
 * In case of a client socket, this function calls socket() and connect().
 * In the meanwhile, it checks for any socket error.
 * If an error occurs, it writes the error message into 'errbuf'.
 *
 * In case of a server socket, the function calls socket(), bind() and listen().
 *
 * This function is usually preceded by the sock_initaddress().
 *
 * \param host: for client sockets, the host name to which we're trying
 * to connect.
 *
 * \param addrinfo: pointer to an addrinfo variable which will be used to
 * open the socket and such. This variable is the one returned by the previous call to
 * sock_initaddress().
 *
 * \param server: '1' if this is a server socket, '0' otherwise.
 *
 * \param nconn: number of the connections that are allowed to wait into the listen() call.
 * This value has no meanings in case of a client socket.
 *
 * \param errbuf: a pointer to an user-allocated buffer that will contain the complete
 * error message. This buffer has to be at least 'errbuflen' in length.
 * It can be NULL; in this case the error cannot be printed.
 *
 * \param errbuflen: length of the buffer that will contains the error. The error message cannot be
 * larger than 'errbuflen - 1' because the last char is reserved for the string terminator.
 *
 * \return the socket that has been opened (that has to be used in the following sockets calls)
 * if everything is fine, INVALID_SOCKET if some errors occurred. The error message is returned
 * in the 'errbuf' variable.
 */
SOCKET sock_open(const char *host, struct addrinfo *addrinfo, int server, int nconn, char *errbuf, int errbuflen)
{
    SOCKET sock;

    /* This is a server socket */
    if (server)
        {
            // 定义整型变量 on
            int on;

            /*
             * 尝试创建套接字。
             */
            // 调用函数创建套接字，传入地址信息和错误缓冲区
            sock = sock_create_socket(addrinfo, errbuf, errbuflen);
            // 如果创建失败，返回无效的套接字
            if (sock == INVALID_SOCKET)
            {
                return INVALID_SOCKET;
            }

            /*
             * 允许新的服务器在旧服务器退出后绑定套接字，即使残留套接字仍然存在。
             *
             * 不将错误视为失败。
             */
            // 设置 on 变量为 1
            on = 1;
            // 设置套接字选项，允许地址重用
            (void)setsockopt(sock, SOL_SOCKET, SO_REUSEADDR,
                (char *)&on, sizeof (on));
        }
#if defined(IPV6_V6ONLY) || defined(IPV6_BINDV6ONLY)
        /*
         * 强制使用仅IPv6地址。
         *
         * RFC 3493指出，您可以在IPv6套接字上支持IPv4：
         *
         *    https://tools.ietf.org/html/rfc3493#section-3.7
         *
         * 并且这是默认行为。这意味着，如果我们首先创建一个绑定到“任何”地址的IPv6套接字，实际上也绑定到IPv4的“任何”地址，因此当我们创建一个IPv4套接字并尝试将其绑定到IPv4的“任何”地址时，会出现EADDRINUSE。
         *
         * 并非所有网络堆栈都支持IPv6套接字上的IPv4；早期的Windows堆栈不支持它，OpenBSD堆栈出于安全原因不支持它（请参阅OpenBSD inet6(4)手册页）。因此，我们不希望依赖这种行为。
         *
         * 因此，我们尝试禁用它，使用RFC 3493中的IPV6_V6ONLY选项：
         *
         *    https://tools.ietf.org/html/rfc3493#section-5.3
         *
         * 或者旧版UNIX中的IPV6_BINDV6ONLY选项。
         */
#ifndef IPV6_V6ONLY
  /* 对于旧系统 */
  #define IPV6_V6ONLY IPV6_BINDV6ONLY
#endif /* IPV6_V6ONLY */
        if (addrinfo->ai_family == PF_INET6)
        {
            on = 1;
            if (setsockopt(sock, IPPROTO_IPV6, IPV6_V6ONLY,
                (char *)&on, sizeof (int)) == -1)
            {
                if (errbuf)
                    snprintf(errbuf, errbuflen, "setsockopt(IPV6_V6ONLY)");
                closesocket(sock);
                return INVALID_SOCKET;
            }
        }
#endif /* defined(IPV6_V6ONLY) || defined(IPV6_BINDV6ONLY) */

        /* 如果地址是多播地址，需要在这里放置适当的 Win32 代码 */
        if (bind(sock, addrinfo->ai_addr, (int) addrinfo->ai_addrlen) != 0)
        {
            // 绑定失败时关闭套接字并返回无效套接字
            sock_geterrmsg(errbuf, errbuflen, "bind() failed");
            closesocket(sock);
            return INVALID_SOCKET;
        }

        if (addrinfo->ai_socktype == SOCK_STREAM)
            if (listen(sock, nconn) == -1)
            {
                // 监听失败时关闭套接字并返回无效套接字
                sock_geterrmsg(errbuf, errbuflen,
                    "listen() failed");
                closesocket(sock);
                return INVALID_SOCKET;
            }

        /* 服务器端结束 */
        return sock;
    }
    else    /* 我们是客户端 */
#ifdef FUZZING_BUILD_MODE_UNSAFE_FOR_PRODUCTION
            break;
    }
}

/*
 * \brief 关闭当前 (TCP 和 UDP) 套接字连接。
 *
 * 此函数发送 shutdown() 在套接字上以禁用 send() 调用
 * (而 recv() 调用仍然允许)。然后关闭套接字。
 *
 * \param sock: 要关闭的连接的套接字标识符。
 *
 * \param errbuf: 指向用户分配的缓冲区的指针，该缓冲区将包含完整的错误消息。
 * 此缓冲区的长度必须至少为 'errbuflen'。
 * 它可以为 NULL；在这种情况下，无法打印错误。
 *
 * \param errbuflen: 将包含错误的缓冲区的长度。错误消息不能
 * 大于 'errbuflen - 1'，因为最后一个字符保留给字符串终结符。
 *
 * \return 如果一切正常，则返回 '0'，如果发生错误，则返回 '-1'。错误消息在 'errbuf' 变量中返回。
 */
int sock_close(SOCKET sock, char *errbuf, int errbuflen)
{
    /*
     * SHUT_WR: 禁止对 send 函数的后续调用。
     * 对于 TCP 套接字，在发送和服务器确认所有数据后，将发送 FIN。
     */
    if (shutdown(sock, SHUT_WR))
    {
        // 获取套接字错误信息并存储到 errbuf 中，如果失败则返回 "shutdown() feiled"
        sock_geterrmsg(errbuf, errbuflen, "shutdown() feiled");
        // 无论如何都关闭套接字
        closesocket(sock);
        // 返回 -1
        return -1;
    }

    // 关闭套接字
    closesocket(sock);
    // 返回 0
    return 0;
/*
 * gai_strerror()存在一些问题：
 *
 * 1）在Windows上，Microsoft明确表示它不是线程安全的；
 * 2）在UNIX上，单一UNIX规范并没有说它是线程安全的，因此一个实现可能对未知的错误代码使用静态缓冲区；
 * 3）对于最可能的错误EAI_NONAME，几个平台上的错误消息都非常糟糕（“nodename nor servname provided, or not known”？通常会是“not known”，而不是“oopsie，我为主机名和服务名传递了空指针”，更不用说他们忘记了“neither”）；
 *
 * 因此我们自己来实现。
 */
static void
get_gai_errstring(char *errbuf, int errbuflen, const char *prefix, int err,
    const char *hostname, const char *portname)
{
    char hostport[PCAP_ERRBUF_SIZE];

    if (hostname != NULL && portname != NULL)
        snprintf(hostport, PCAP_ERRBUF_SIZE, "host and port %s:%s",
            hostname, portname);
    else if (hostname != NULL)
        snprintf(hostport, PCAP_ERRBUF_SIZE, "host %s",
            hostname);
    else if (portname != NULL)
        snprintf(hostport, PCAP_ERRBUF_SIZE, "port %s",
            portname);
    else
        snprintf(hostport, PCAP_ERRBUF_SIZE, "<no host or port!>");
    switch (err)
    {
#ifdef EAI_ADDRFAMILY
        case EAI_ADDRFAMILY:
            snprintf(errbuf, errbuflen,
                "%sAddress family for %s not supported",
                prefix, hostport);
            break;
        # 如果解析主机名或端口时出现 EAI_AGAIN 错误
        case EAI_AGAIN:
            # 格式化错误信息，提示无法解析主机名或端口
            snprintf(errbuf, errbuflen,
                "%s%s could not be resolved at this time",
                prefix, hostport);
            break;

        # 如果解析主机名或端口时出现 EAI_BADFLAGS 错误
        case EAI_BADFLAGS:
            # 格式化错误信息，提示 ai_flags 参数的值无效
            snprintf(errbuf, errbuflen,
                "%sThe ai_flags parameter for looking up %s had an invalid value",
                prefix, hostport);
            break;

        # 如果解析主机名或端口时出现 EAI_FAIL 错误
        case EAI_FAIL:
            # 格式化错误信息，提示解析时发生不可恢复的错误
            snprintf(errbuf, errbuflen,
                "%sA non-recoverable error occurred when attempting to resolve %s",
                prefix, hostport);
            break;

        # 如果解析主机名或端口时出现 EAI_FAMILY 错误
        case EAI_FAMILY:
            # 格式化错误信息，提示地址族未被识别
            snprintf(errbuf, errbuflen,
                "%sThe address family for looking up %s was not recognized",
                prefix, hostport);
            break;

        # 如果解析主机名或端口时出现 EAI_MEMORY 错误
        case EAI_MEMORY:
            # 格式化错误信息，提示内存不足
            snprintf(errbuf, errbuflen,
                "%sOut of memory trying to allocate storage when looking up %s",
                prefix, hostport);
            break;

        # 如果解析主机名或端口时出现 EAI_NODATA 错误
#if defined(EAI_NODATA) && EAI_NODATA != EAI_NONAME
        case EAI_NODATA:
            # 格式化错误信息，提示无地址与主机名或端口相关联
            snprintf(errbuf, errbuflen,
                "%sNo address associated with %s",
                prefix, hostport);
            break;
#endif

        # 处理 EAI_NONAME 错误情况
        case EAI_NONAME:
            # 格式化错误信息，指示主机名无法解析
            snprintf(errbuf, errbuflen,
                "%sThe %s couldn't be resolved",
                prefix, hostport);
            break;

        # 处理 EAI_SERVICE 错误情况
        case EAI_SERVICE:
            # 格式化错误信息，指示服务值在查找时未被识别
            snprintf(errbuf, errbuflen,
                "%sThe service value specified when looking up %s as not recognized for the socket type",
                prefix, hostport);
            break;

        # 处理 EAI_SOCKTYPE 错误情况
        case EAI_SOCKTYPE:
            # 格式化错误信息，指示套接字类型在查找时未被识别
            snprintf(errbuf, errbuflen,
                "%sThe socket type specified when looking up %s as not recognized",
                prefix, hostport);
            break;

#ifdef EAI_SYSTEM
        # 处理 EAI_SYSTEM 错误情况
        case EAI_SYSTEM:
            '''
             * 假设为 UNIX 系统。
             '''
            # 格式化错误信息，指示查找主机名时发生错误
            pcap_fmt_errmsg_for_errno(errbuf, errbuflen, errno,
                "%sAn error occurred when looking up %s",
                prefix, hostport);
            break;
#endif

#ifdef EAI_BADHINTS
        # 处理 EAI_BADHINTS 错误情况
        case EAI_BADHINTS:
            # 格式化错误信息，指示查找主机名时的提示值无效
            snprintf(errbuf, errbuflen,
                "%sInvalid value for hints when looking up %s",
                prefix, hostport);
            break;
#endif

#ifdef EAI_PROTOCOL
        # 处理 EAI_PROTOCOL 错误情况
        case EAI_PROTOCOL:
            # 格式化错误信息，指示查找主机名时解析的协议未知
            snprintf(errbuf, errbuflen,
                "%sResolved protocol when looking up %s is unknown",
                prefix, hostport);
            break;
#endif

#ifdef EAI_OVERFLOW
        # 处理 EAI_OVERFLOW 错误情况
        case EAI_OVERFLOW:
            # 格式化错误信息，指示查找主机名时参数缓冲区溢出
            snprintf(errbuf, errbuflen,
                "%sArgument buffer overflow when looking up %s",
                prefix, hostport);
            break;
#endif

        # 处理默认情况
        default:
            # 格式化错误信息，指示在查找主机名时发生 getaddrinfo() 错误
            snprintf(errbuf, errbuflen,
                "%sgetaddrinfo() error %d when looking up %s",
                prefix, err, hostport);
            break;
    }
}

# 初始化套接字地址
int sock_initaddress(const char *host, const char *port,
    struct addrinfo *hints, struct addrinfo **addrinfo, char *errbuf, int errbuflen)
{
    int retval;
    /*
     * 允许主机和端口都为空，但是 getaddrinfo() 不保证会这样做；
     * 为了处理这种情况，如果端口为空，我们将"0"作为端口号提供。
     *
     * 这样可以从 get_gai_errstring() 获得更好的错误消息，
     * 因为如果没有指定端口，这些消息就不会提到端口有问题。
     */
    retval = getaddrinfo(host, port == NULL ? "0" : port, hints, addrinfo);
    if (retval != 0)
    {
        if (errbuf)
        {
            if (host != NULL && port != NULL) {
                /*
                 * 尝试只使用主机名，以区分"主机有问题"和"端口有问题"。
                 */
                int try_retval;

                try_retval = getaddrinfo(host, NULL, hints,
                    addrinfo);
                if (try_retval == 0) {
                    /*
                     * 只使用主机名就成功了，所以假设问题在端口上。
                     *
                     * 首先释放地址信息。
                     */
                    freeaddrinfo(*addrinfo);
                    get_gai_errstring(errbuf, errbuflen,
                        "", retval, NULL, port);
                } else {
                    /*
                     * 只使用主机名不成功，所以假设问题在主机上。
                     */
                    get_gai_errstring(errbuf, errbuflen,
                        "", retval, host, NULL);
                }
            } else {
                /*
                 * 主机或端口中有一个为空，所以无法确定问题所在。
                 */
                get_gai_errstring(errbuf, errbuflen, "",
                    retval, host, port);
            }
        }
        return -1;
    }
    /*
     * \warning SOCKET: 我应该检查所有的accept()，以便绑定到所有地址，以防addrinfo有多个指针
     */

    /*
     * 此软件仅支持PF_INET和PF_INET6。
     *
     * XXX - 我们是否应该只检查至少*一个*地址是PF_INET或PF_INET6，并且在使用列表时，
     * 忽略所有既不是的地址？ (IPX支持吗？ :-))
     */
    if (((*addrinfo)->ai_family != PF_INET) &&
        ((*addrinfo)->ai_family != PF_INET6))
    {
        if (errbuf)
            snprintf(errbuf, errbuflen, "getaddrinfo(): socket type not supported");
        freeaddrinfo(*addrinfo);
        *addrinfo = NULL;
        return -1;
    }

    /*
     * 你不能在多播（或广播）TCP上进行操作。
     */
    if (((*addrinfo)->ai_socktype == SOCK_STREAM) &&
        (sock_ismcastaddr((*addrinfo)->ai_addr) == 0))
    {
        if (errbuf)
            snprintf(errbuf, errbuflen, "getaddrinfo(): multicast addresses are not valid when using TCP streams");
        freeaddrinfo(*addrinfo);
        *addrinfo = NULL;
        return -1;
    }

    return 0;
}
/*
 * \brief It sends the amount of data contained into 'buffer' on the given socket.
 *
 * This function basically calls the send() socket function and it checks that all
 * the data specified in 'buffer' (of size 'size') will be sent. If an error occurs,
 * it writes the error message into 'errbuf'.
 * In case the socket buffer does not have enough space, it loops until all data
 * has been sent.
 *
 * \param socket: the connected socket currently opened.
 *
 * \param buffer: a char pointer to a user-allocated buffer in which data is contained.
 *
 * \param size: number of bytes that have to be sent.
 *
 * \param errbuf: a pointer to an user-allocated buffer that will contain the complete
 * error message. This buffer has to be at least 'errbuflen' in length.
 * It can be NULL; in this case the error cannot be printed.
 *
 * \param errbuflen: length of the buffer that will contains the error. The error message cannot be
 * larger than 'errbuflen - 1' because the last char is reserved for the string terminator.
 *
 * \return '0' if everything is fine, '-1' if an error other than
 * "connection reset" or "peer has closed the receive side" occurred,
 * '-2' if we got one of those errors.
 * For errors, an error message is returned in the 'errbuf' variable.
 */
int sock_send(SOCKET sock, SSL *ssl _U_NOSSL_, const char *buffer, size_t size,
    char *errbuf, int errbuflen)
{
    int remaining;  // remaining bytes to be sent
    ssize_t nsent;  // number of bytes sent

    // check if the size is larger than the maximum integer value
    if (size > INT_MAX)
    {
        // if error buffer is provided, write the error message
        if (errbuf)
        {
            snprintf(errbuf, errbuflen,
                "Can't send more than %u bytes with sock_send",
                INT_MAX);
        }
        return -1;  // return -1 for error
    }
    remaining = (int)size;  // set remaining to size

    // loop until all data has been sent
    do {
#ifdef HAVE_OPENSSL
        // if SSL is enabled, call ssl_send function and return its result
        if (ssl) return ssl_send(ssl, buffer, remaining, errbuf, errbuflen);
#endif

#ifdef FUZZING_BUILD_MODE_UNSAFE_FOR_PRODUCTION
        nsent = remaining;  // set nsent to remaining
#else
#ifdef MSG_NOSIGNAL
        /*
         * 如果定义了 MSG_NOSIGNAL，则使用该标志发送数据，以防止在流式套接字上发生错误时收到 SIGPIPE 信号，
         * 当对端断开连接时，仍然会返回 EPIPE 错误。
         */
        nsent = send(sock, buffer, remaining, MSG_NOSIGNAL);
#else
        nsent = send(sock, buffer, remaining, 0);
#endif
#endif //FUZZING_BUILD_MODE_UNSAFE_FOR_PRODUCTION

        if (nsent == -1)
        {
            /*
             * 如果客户端在我们发送数据时关闭了连接，就不需要将其记录为错误。
             */
            int errcode;

#ifdef _WIN32
            errcode = GetLastError();
            if (errcode == WSAECONNRESET ||
                errcode == WSAECONNABORTED)
            {
                /*
                 * 在 Winsock 中，当尝试在对端关闭接收端的连接上发送数据时，WSAECONNABORTED 似乎是返回的错误。
                 */
                return -2;
            }
            sock_fmterrmsg(errbuf, errbuflen, errcode,
                "send() failed");
#else
            errcode = errno;
            if (errcode == ECONNRESET || errcode == EPIPE)
            {
                /*
                 * 在 UNIX 上，当尝试在对端关闭接收端的连接上发送数据时，EPIPE 是返回的错误。
                 */
                return -2;
            }
            sock_fmterrmsg(errbuf, errbuflen, errcode,
                "send() failed");
#endif
            return -1;
        }

        remaining -= nsent;
        buffer += nsent;
    } while (remaining != 0);

    return 0;
}

int sock_bufferize(const void *data, int size, char *outbuf, int *offset, int totsize, int checkonly, char *errbuf, int errbuflen)
{
    if ((*offset + size) > totsize)
    {
        // 如果错误缓冲区存在，则将错误信息写入缓冲区
        if (errbuf)
            snprintf(errbuf, errbuflen, "Not enough space in the temporary send buffer.");
        // 返回错误代码 -1
        return -1;
    }

    // 如果不是仅检查模式，则将数据从源地址复制到目标地址
    if (!checkonly)
        memcpy(outbuf + (*offset), data, size);

    // 偏移量增加数据大小
    (*offset) += size;

    // 返回成功代码 0
    return 0;
}
// 从套接字接收数据，支持 SSL，返回接收到的数据大小
int sock_recv(SOCKET sock, SSL *ssl _U_NOSSL_, void *buffer, size_t size,
    int flags, char *errbuf, int errbuflen)
{
    int recv_flags = 0;  // 接收标志位
    char *bufp = buffer;  // 缓冲区指针
    int remaining;  // 剩余数据大小
    ssize_t nread;  // 实际读取的数据大小

    if (size == 0)  // 如果要接收的数据大小为0，则直接返回0
    {
        return 0;
    }
    if (size > INT_MAX)  // 如果要接收的数据大小超过INT_MAX，则返回错误
    {
        if (errbuf)
        {
            snprintf(errbuf, errbuflen,
                "Can't read more than %u bytes with sock_recv",
                INT_MAX);
        }
        return -1;
    }

    if (flags & SOCK_MSG_PEEK)  // 如果接收标志位包含SOCK_MSG_PEEK，则设置接收标志位为MSG_PEEK
        recv_flags |= MSG_PEEK;

    bufp = (char *) buffer;  // 重新设置缓冲区指针
    remaining = (int) size;  // 设置剩余数据大小为要接收的数据大小

    /*
     * We don't use MSG_WAITALL because it's not supported in
     * Win32.
     */
    for (;;) {  // 无限循环，直到接收到指定大小的数据
#ifdef FUZZING_BUILD_MODE_UNSAFE_FOR_PRODUCTION
        nread = fuzz_recv(bufp, remaining);  // 使用模糊测试接收数据
#elif defined(HAVE_OPENSSL)
        if (ssl)  // 如果使用SSL
        {
            /*
             * XXX - what about MSG_PEEK?
             */
            nread = ssl_recv(ssl, bufp, remaining, errbuf, errbuflen);  // 使用SSL接收数据
            if (nread == -2) return -1;  // 如果SSL接收出错，则返回-1
        }
        else
            nread = recv(sock, bufp, remaining, recv_flags);  // 否则使用普通套接字接收数据
#else
        nread = recv(sock, bufp, remaining, recv_flags);  // 使用普通套接字接收数据
#endif

        if (nread == -1)  // 如果接收出错
        {
#ifndef _WIN32
            if (errno == EINTR)  // 如果是中断错误
                return -3;  // 返回-3
#endif
            # 如果接收数据失败，将错误信息存储到errbuf中并返回-1
            sock_geterrmsg(errbuf, errbuflen, "recv() failed");
            return -1;
        }

        # 如果接收到的数据大小为0
        if (nread == 0)
        {
            # 如果设置了EOF_IS_ERROR标志，或者剩余数据大小不等于请求的大小
            if ((flags & SOCK_EOF_IS_ERROR) ||
                (remaining != (int) size))
            {
                '''
                 * 要么我们已经读取了一些数据，
                 * 要么我们总是应该在EOF时返回错误。
                 '''
                # 如果errbuf不为空，将连接终止的错误信息存储到errbuf中
                if (errbuf)
                {
                    snprintf(errbuf, errbuflen,
                        "The other host terminated the connection.");
                }
                return -1;
            }
            else
                return 0;
        }

        '''
         * 我们是想读取请求的数量，还是只返回我们得到的？
         '''
        # 如果没有设置RECEIVEALL_YES标志
        if (!(flags & SOCK_RECEIVEALL_YES))
        {
            '''
             * 只返回我们得到的。
             '''
            return (int) nread;
        }

        # 移动缓冲区指针和更新剩余数据大小
        bufp += nread;
        remaining -= nread;

        # 如果剩余数据大小为0，返回请求的大小
        if (remaining == 0)
            return (int) size;
    }
}

'''
 * 从套接字接收数据报文。
 *
 * 成功时返回数据报文的大小，出错时返回-1。
 '''
int sock_recv_dgram(SOCKET sock, SSL *ssl _U_NOSSL_, void *buffer, size_t size,
    char *errbuf, int errbuflen)
{
    ssize_t nread;
#ifndef _WIN32
    struct msghdr message;
    struct iovec iov;
#endif

    # 如果请求的大小为0，直接返回0
    if (size == 0)
    {
        return 0;
    }
    # 如果请求的大小超过INT_MAX
    if (size > INT_MAX)
    {
        # 如果errbuf不为空，将错误信息存储到errbuf中
        if (errbuf)
        {
            snprintf(errbuf, errbuflen,
                "Can't read more than %u bytes with sock_recv_dgram",
                INT_MAX);
        }
        return -1;
    }

#ifdef HAVE_OPENSSL
    // TODO: DTLS
    # 如果ssl不为空，返回DTLS未实现的错误信息
    if (ssl)
    {
        snprintf(errbuf, errbuflen, "DTLS not implemented yet");
        return -1;
    }
#endif
    /*
     * 这应该是一个数据报套接字，因此我们应该在一个recv()或recvmsg()调用中获取整个数据报，而不需要循环。
     */
#ifdef _WIN32
    // 如果是 Windows 平台
    nread = recv(sock, buffer, (int)size, 0);
    // 如果接收失败
    if (nread == SOCKET_ERROR)
    {
        /*
         * 引用 MSDN 对 recv() 的文档，
         * "如果数据报或消息大于指定的缓冲区，缓冲区将填充
         * 数据报的第一部分，并且 recv 会生成错误 WSAEMSGSIZE。
         * 对于不可靠的协议（例如 UDP），多余的数据将丢失..."
         *
         * 所以如果消息大于我们提供的缓冲区，多余的数据将被丢弃，
         * 并且我们会报告一个错误。
         */
        sock_fmterrmsg(errbuf, errbuflen, sock_geterrcode(),
            "recv() failed");
        return -1;
    }
#else /* _WIN32 */
    /*
     * 单一 UNIX 规范表示，针对面向消息的协议的套接字的 recv() 将丢弃
     * 多余的数据。它并没有指示接收将失败，例如，EMSGSIZE。
     *
     * 因此，我们使用 recvmsg()，这似乎是在接收消息时获取“消息被截断”指示的
     * 唯一方法。
     */
    message.msg_name = NULL;    // 我们不关心消息的来源
    message.msg_namelen = 0;
    iov.iov_base = buffer;
    iov.iov_len = size;
    message.msg_iov = &iov;
    message.msg_iovlen = 1;
#ifdef HAVE_STRUCT_MSGHDR_MSG_CONTROL
    message.msg_control = NULL;    // 我们不关心控制信息
    message.msg_controllen = 0;
#endif
#ifdef HAVE_STRUCT_MSGHDR_MSG_FLAGS
    message.msg_flags = 0;
#endif
#ifdef FUZZING_BUILD_MODE_UNSAFE_FOR_PRODUCTION
    nread = fuzz_recv(buffer, size);
#else
    nread = recvmsg(sock, &message, 0);
#endif
    // 如果接收失败
    if (nread == -1)
    {
        if (errno == EINTR)
            return -3;
        sock_geterrmsg(errbuf, errbuflen, "recv() failed");
        return -1;
    }
#ifdef HAVE_STRUCT_MSGHDR_MSG_FLAGS
    /*
     * 如果消息标志中包含 MSG_TRUNC，表示接收到的消息比指定的缓冲区大小要大。
     * 
     * 根据 Microsoft 文档的暗示，在类似情况下应该将此报告为错误。
     * 
     * 使用 snprintf 将错误信息写入 errbuf，并返回 -1 表示错误。
     */
    if (message.msg_flags & MSG_TRUNC)
    {
        /*
         * 消息比指定的缓冲区大小要大。
         * 
         * 将此报告为错误，因为 Microsoft 文档暗示在类似情况下应该这样做。
         */
        snprintf(errbuf, errbuflen, "recv(): Message too long");
        return -1;
    }
#endif /* HAVE_STRUCT_MSGHDR_MSG_FLAGS */
#endif /* _WIN32 */

/*
 * The size we're reading fits in an int, so the return value
 * will fit in an int.
 */
// 返回的读取大小适合于 int 类型，因此返回值也适合于 int 类型。
return (int)nread;
}

/*
 * \brief It discards N bytes that are currently waiting to be read on the current socket.
 *
 * This function is useful in case we receive a message we cannot understand (e.g.
 * wrong version number when receiving a network packet), so that we have to discard all
 * data before reading a new message.
 *
 * This function will read 'size' bytes from the socket and discard them.
 * It defines an internal buffer in which data will be copied; however, in case
 * this buffer is not large enough, it will cycle in order to read everything as well.
 *
 * \param sock: the connected socket currently opened.
 *
 * \param size: number of bytes that have to be discarded.
 *
 * \param errbuf: a pointer to an user-allocated buffer that will contain the complete
 * error message. This buffer has to be at least 'errbuflen' in length.
 * It can be NULL; in this case the error cannot be printed.
 *
 * \param errbuflen: length of the buffer that will contains the error. The error message cannot be
 * larger than 'errbuflen - 1' because the last char is reserved for the string terminator.
 *
 * \return '0' if everything is fine, '-1' if some errors occurred.
 * The error message is returned in the 'errbuf' variable.
 */
// 丢弃当前套接字上当前等待读取的 N 个字节。
// 这个函数在我们收到一个无法理解的消息时很有用（例如，在接收网络数据包时收到错误的版本号），因此我们必须在读取新消息之前丢弃所有数据。
// 这个函数将从套接字中读取 'size' 个字节并丢弃它们。
// 它定义了一个内部缓冲区，数据将被复制到其中；但是，如果这个缓冲区不够大，它将循环以便读取所有内容。
int sock_discard(SOCKET sock, SSL *ssl, int size, char *errbuf, int errbuflen)
{
#define TEMP_BUF_SIZE 32768

char buffer[TEMP_BUF_SIZE];        /* network buffer, to be used when the message is discarded */
    /*
     * 当需要丢弃消息时，静态分配内存避免了每次都需要调用 'malloc()'
     * 我们认为32KB的缓冲区对于大多数应用来说已经足够了；
     * 如果不够的话，"while"循环通过多次调用sockrecv()来丢弃消息。
     * 我们不想创建一个更大的变量，因为这会导致程序在某些平台上退出（例如BSD）
     */
    while (size > TEMP_BUF_SIZE)
    {
        // 如果消息大小大于临时缓冲区大小，通过多次调用sock_recv()来丢弃消息
        if (sock_recv(sock, ssl, buffer, TEMP_BUF_SIZE, SOCK_RECEIVEALL_YES, errbuf, errbuflen) == -1)
            return -1;

        // 减去已经处理的消息大小
        size -= TEMP_BUF_SIZE;
    }

    /*
     * 如果仍然有数据需要丢弃
     * 在这种情况下，数据可以适应临时缓冲区
     */
    if (size)
    {
        // 如果还有数据需要丢弃，且数据大小小于临时缓冲区大小，直接丢弃
        if (sock_recv(sock, ssl, buffer, size, SOCK_RECEIVEALL_YES, errbuf, errbuflen) == -1)
            return -1;
    }

    // 返回0表示成功
    return 0;
/*
 * \brief 检查一个主机（由sockaddr_storage结构标识）是否属于“允许列表”。
 *
 * 此函数在accept()调用之后非常有用，用于检查连接的主机是否被允许连接到本机。为此，我们有一个缓冲区，保存着允许主机的列表；此函数将连接主机的sockaddr_storage结构与此主机列表进行比较，并在主机包含在此列表中时返回'0'。
 *
 * \param hostlist: 指向包含允许主机列表的字符串的指针。
 *
 * \param sep: 保存主机列表中主机之间使用的分隔符的字符串（例如空格字符）。
 *
 * \param from: 一个sockaddr_storage结构，就像它被accept()调用返回的那样。
 *
 * \param errbuf: 指向用户分配的缓冲区的指针，该缓冲区将包含完整的错误消息。此缓冲区的长度必须至少为'errbuflen'。
 * 它可以为NULL；在这种情况下，无法打印错误。
 *
 * \param errbuflen: 将包含错误的缓冲区的长度。错误消息不能大于'errbuflen - 1'，因为最后一个字符保留给字符串终结符。
 *
 * \return 返回值：
 * - 如果主机列表为空，则返回'1'
 * - 如果主机属于主机列表（因此允许连接），则返回'0'
 * - 如果主机不属于主机列表（因此不允许连接），则返回'-1'
 * - 如果出现错误，则返回'-2'。错误消息在'errbuf'变量中返回。
 */
int sock_check_hostlist(char *hostlist, const char *sep, struct sockaddr_storage *from, char *errbuf, int errbuflen)
{
    /* 检查连接的主机是否在允许列表中 */
    if ((hostlist) && (hostlist[0]))
    }

    /* 没有主机列表，因此必须返回'空列表' */
    return 1;
}
/*
 * \brief 比较两个包含在两个sockaddr_storage结构中的地址。
 *
 * 此函数用于比较两个地址，给定它们的内部表示，即sockaddr_storage结构。
 *
 * 这两个结构不需要是sockaddr_storage；您可以有'sockaddr_in'和sockaddr_in6，适当地转换为函数接口的兼容性。
 *
 * 如果两个地址匹配，此函数将返回'0'，如果不匹配，则返回'-1'。
 *
 * \param first: 一个sockaddr_storage结构（例如通过accept()调用返回的结构），包含要比较的第一个地址。
 *
 * \param second: 包含要比较的第二个地址的sockaddr_storage结构。
 *
 * \return 如果地址相等，则返回'0'，如果它们不同，则返回'-1'。
 */
int sock_cmpaddr(struct sockaddr_storage *first, struct sockaddr_storage *second)
{
    if (first->ss_family == second->ss_family)
    {
        if (first->ss_family == AF_INET)
        {
            if (memcmp(&(((struct sockaddr_in *) first)->sin_addr),
                &(((struct sockaddr_in *) second)->sin_addr),
                sizeof(struct in_addr)) == 0)
                return 0;
        }
        else /* address family is AF_INET6 */
        {
            if (memcmp(&(((struct sockaddr_in6 *) first)->sin6_addr),
                &(((struct sockaddr_in6 *) second)->sin6_addr),
                sizeof(struct in6_addr)) == 0)
                return 0;
        }
    }

    return -1;
}
/*
 * \brief 获取系统为此套接字选择的地址/端口（对于已连接的套接字）。
 *
 * 用于返回服务器在本地机器上为我们的套接字选择的地址和端口。仅适用于：
 * - 已连接的套接字
 * - 服务器套接字
 *
 * 在未连接的客户端套接字上，它不起作用，因为系统仅在套接字调用send()时动态选择端口。
 *
 * \param sock：当前打开的已连接套接字。
 *
 * \param address：包含函数将返回的地址。此缓冲区必须由用户正确分配。地址可以是文字形式或数字形式，具体取决于'Flags'的值。
 *
 * \param addrlen：'address'缓冲区的长度。
 *
 * \param port：包含函数将返回的端口。此缓冲区必须由用户正确分配。
 *
 * \param portlen：'port'缓冲区的长度。
 *
 * \param flags：一组标志（定义在getnameinfo()标准套接字函数中的标志），确定结果地址是否必须是数字形式/文字形式等。
 *
 * \param errbuf：指向用户分配的缓冲区的指针，该缓冲区将包含完整的错误消息。此缓冲区的长度必须至少为'errbuflen'。
 * 它可以为NULL；在这种情况下，无法打印错误。
 *
 * \param errbuflen：将包含错误的缓冲区的长度。错误消息的长度不能大于'errbuflen - 1'，因为最后一个字符保留给字符串终止符。
 *
 * \return 如果此函数成功，则返回'-1'，否则返回'0'。
 * 对应的地址和端口将返回到'address'和'port'缓冲区中。无论如何，返回的字符串都以'0'结尾。
 *
 * \warning 如果套接字使用无连接协议，则在套接字上发生I/O之前，地址可能不可用。
 */
# 获取套接字的本地地址和端口信息
int sock_getmyinfo(SOCKET sock, char *address, int addrlen, char *port, int portlen, int flags, char *errbuf, int errbuflen)
{
    // 存储套接字的地址信息
    struct sockaddr_storage mysockaddr;
    // 存储套接字地址结构的长度
    socklen_t sockaddrlen;

    // 获取套接字地址结构的长度
    sockaddrlen = sizeof(struct sockaddr_storage);

    // 获取套接字的本地地址和端口信息
    if (getsockname(sock, (struct sockaddr *) &mysockaddr, &sockaddrlen) == -1)
    {
        // 获取错误信息并返回
        sock_geterrmsg(errbuf, errbuflen, "getsockname() failed");
        return 0;
    }

    /* 返回触发错误的主机的数值地址 */
    return sock_getascii_addrport(&mysockaddr, address, addrlen, port, portlen, flags, errbuf, errbuflen);
}

// 获取套接字地址结构的数值地址和端口信息
int sock_getascii_addrport(const struct sockaddr_storage *sockaddr, char *address, int addrlen, char *port, int portlen, int flags, char *errbuf, size_t errbuflen)
{
    // 存储套接字地址结构的长度
    socklen_t sockaddrlen;
    // 保存返回值的变量
    int retval;

    // 初始化返回值
    retval = -1;

    // 根据操作系统设置套接字地址结构的长度
#ifdef _WIN32
    if (sockaddr->ss_family == AF_INET)
        sockaddrlen = sizeof(struct sockaddr_in);
    else
        sockaddrlen = sizeof(struct sockaddr_in6);
#else
    sockaddrlen = sizeof(struct sockaddr_storage);
#endif

    // 检查是否需要获取文字形式的地址
    if ((flags & NI_NUMERICHOST) == 0)
    {
        // 检查是否为IPv6地址的未指定地址
        if ((sockaddr->ss_family == AF_INET6) &&
            (memcmp(&((struct sockaddr_in6 *) sockaddr)->sin6_addr, "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0", sizeof(struct in6_addr)) == 0))
        {
            // 如果是未指定地址，设置地址为NULL_DAD并返回
            if (address)
                pcap_strlcpy(address, SOCKET_NAME_NULL_DAD, addrlen);
            return retval;
        }
    }

    // 获取套接字地址结构的数值地址和端口信息
    if (getnameinfo((struct sockaddr *) sockaddr, sockaddrlen, address, addrlen, port, portlen, flags) != 0)
    {
        /* 如果用户想要接收错误消息 */
        if (errbuf)
        {
            // 获取套接字错误消息并存储到errbuf中
            sock_geterrmsg(errbuf, errbuflen,
                "getnameinfo() failed");
            // 确保errbuf以null结尾
            errbuf[errbuflen - 1] = 0;
        }

        // 如果存在地址参数
        if (address)
        {
            // 将SOCKET_NO_NAME_AVAILABLE复制到address中
            pcap_strlcpy(address, SOCKET_NO_NAME_AVAILABLE, addrlen);
            // 确保address以null结尾
            address[addrlen - 1] = 0;
        }

        // 如果存在端口参数
        if (port)
        {
            // 将SOCKET_NO_PORT_AVAILABLE复制到port中
            pcap_strlcpy(port, SOCKET_NO_PORT_AVAILABLE, portlen);
            // 确保port以null结尾
            port[portlen - 1] = 0;
        }

        // 返回值设为0
        retval = 0;
    }

    // 返回结果值
    return retval;
}

// 将地址转换为网络地址结构
int sock_present2network(const char *address, struct sockaddr_storage *sockaddr, int addr_family, char *errbuf, int errbuflen)
{
    int retval;
    struct addrinfo *addrinfo;
    struct addrinfo hints;

    // 清空 hints 结构体
    memset(&hints, 0, sizeof(hints));

    // 设置地址族
    hints.ai_family = addr_family;

    // 初始化地址信息
    if ((retval = sock_initaddress(address, "22222" /* fake port */, &hints, &addrinfo, errbuf, errbuflen)) == -1)
        return 0;

    // 拷贝地址信息到 sockaddr
    if (addrinfo->ai_family == PF_INET)
        memcpy(sockaddr, addrinfo->ai_addr, sizeof(struct sockaddr_in));
    else
        memcpy(sockaddr, addrinfo->ai_addr, sizeof(struct sockaddr_in6));

    // 如果有多个地址返回，则返回错误
    if (addrinfo->ai_next != NULL)
    {
        freeaddrinfo(addrinfo);

        if (errbuf)
            snprintf(errbuf, errbuflen, "More than one socket requested; using the first one returned");
        return -2;
    }

    // 释放地址信息
    freeaddrinfo(addrinfo);
    return -1;
}
```