# `nmap\libpcap\nametoaddr.c`

```
/*
 * 版权声明，版权归加利福尼亚大学所有
 * 允许在源代码和二进制形式下进行再发布和使用，无论是否进行修改：
 * 1. 源代码发布必须保留以上版权声明和本段完整内容
 * 2. 包含二进制代码的发布必须在文档或其他提供的材料中保留以上版权声明和本段完整内容
 * 3. 所有提及此软件特性或使用的广告材料必须显示以下声明：
 *    “本产品包含由加利福尼亚大学劳伦斯伯克利实验室及其贡献者开发的软件。”
 *    未经事先书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品。
 * 此软件按原样提供，不附带任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。
 *
 * 用于扫描仪的名称到ID转换例程
 * 这些函数不是时间关键的。
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#ifdef DECNETLIB
#include <sys/types.h>
#include <netdnet/dnetdb.h>
#endif

#ifdef _WIN32
  #include <winsock2.h>
  #include <ws2tcpip.h>

  #ifdef INET6
    /*
     * 引用 MSDN 页面上关于 getaddrinfo() 函数的说明
     * 链接：https://msdn.microsoft.com/en-us/library/windows/desktop/ms738520(v=vs.85).aspx
     *
     * "对于 Windows 2000 及更早版本的 getaddrinfo 支持
     * getaddrinfo 函数是在 Windows XP 及更高版本的 Ws2_32.dll 中添加的。
     * 要在早期版本的 Windows 上执行使用此函数的应用程序，需要包含 Ws2tcpip.h 和 Wspiapi.h 文件。
     * 当添加 Wspiapi.h 包含文件时，getaddrinfo 函数被定义为 Wspiapi.h 文件中的 WspiapiGetAddrInfo 内联函数。
     * 在运行时，WspiapiGetAddrInfo 函数以这样一种方式实现，即如果 Ws2_32.dll 或 Wship6.dll（包含 IPv6 技术预览版中 getaddrinfo 的文件）不包括 getaddrinfo，
     * 则基于 Wspiapi.h 头文件中的代码内联实现一个版本的 getaddrinfo。这个内联代码将在不原生支持 getaddrinfo 函数的旧版 Windows 平台上使用。"
     *
     * 我们使用 getaddrinfo() 函数，因此在这里包含 Wspiapi.h。
     */
    #include <wspiapi.h>
    #endif /* INET6 */
#else /* _WIN32 */
  #include <sys/param.h>  // 包含系统参数的头文件
  #include <sys/types.h>  // 包含系统数据类型的头文件
  #include <sys/socket.h>  // 包含套接字相关的头文件
  #include <sys/time.h>  // 包含时间相关的头文件

  #include <netinet/in.h>  // 包含互联网地址族的头文件

  #ifdef HAVE_ETHER_HOSTTON
    #if defined(NET_ETHERNET_H_DECLARES_ETHER_HOSTTON)
      /*
       * OK, just include <net/ethernet.h>.
       */
      #include <net/ethernet.h>  // 如果定义了NET_ETHERNET_H_DECLARES_ETHER_HOSTTON，则包含<net/ethernet.h>
    #elif defined(NETINET_ETHER_H_DECLARES_ETHER_HOSTTON)
      /*
       * OK, just include <netinet/ether.h>
       */
      #include <netinet/ether.h>  // 如果定义了NETINET_ETHER_H_DECLARES_ETHER_HOSTTON，则包含<netinet/ether.h>
    #elif defined(SYS_ETHERNET_H_DECLARES_ETHER_HOSTTON)
      /*
       * OK, just include <sys/ethernet.h>
       */
      #include <sys/ethernet.h>  // 如果定义了SYS_ETHERNET_H_DECLARES_ETHER_HOSTTON，则包含<sys/ethernet.h>
    #elif defined(ARPA_INET_H_DECLARES_ETHER_HOSTTON)
      /*
       * OK, just include <arpa/inet.h>
       */
      #include <arpa/inet.h>  // 如果定义了ARPA_INET_H_DECLARES_ETHER_HOSTTON，则包含<arpa/inet.h>
    #elif defined(NETINET_IF_ETHER_H_DECLARES_ETHER_HOSTTON)
      /*
       * OK, include <netinet/if_ether.h>, after all the other stuff we
       * need to include or define for its benefit.
       */
      #define NEED_NETINET_IF_ETHER_H  // 如果定义了NETINET_IF_ETHER_H_DECLARES_ETHER_HOSTTON，则需要包含<netinet/if_ether.h>
    #else
      /*
       * We'll have to declare it ourselves.
       * If <netinet/if_ether.h> defines struct ether_addr, include
       * it.  Otherwise, define it ourselves.
       */
      #ifdef HAVE_STRUCT_ETHER_ADDR
        #define NEED_NETINET_IF_ETHER_H  // 如果定义了HAVE_STRUCT_ETHER_ADDR，则需要包含<netinet/if_ether.h>
      #else /* HAVE_STRUCT_ETHER_ADDR */
    struct ether_addr {
        unsigned char ether_addr_octet[6];
    };
      #endif /* HAVE_STRUCT_ETHER_ADDR */
    #endif /* what declares ether_hostton() */

    #ifdef NEED_NETINET_IF_ETHER_H
      #include <net/if.h>    /* Needed on some platforms */  // 在一些平台上需要包含<net/if.h>
      #include <netinet/in.h>    /* Needed on some platforms */  // 在一些平台上需要包含<netinet/in.h>
      #include <netinet/if_ether.h>  // 包含<netinet/if_ether.h>
    #endif /* NEED_NETINET_IF_ETHER_H */

    #ifndef HAVE_DECL_ETHER_HOSTTON
      /*
       * No header declares it, so declare it ourselves.
       */
      extern int ether_hostton(const char *, struct ether_addr *);  // 如果没有声明ETHER_HOSTTON，则自己声明
    #endif /* !defined(HAVE_DECL_ETHER_HOSTTON) */
  #endif /* HAVE_ETHER_HOSTTON */

  #include <arpa/inet.h>
  #include <netdb.h>

- 如果未定义 HAVE_DECL_ETHER_HOSTTON，则结束宏定义
- 结束 HAVE_ETHER_HOSTTON 宏定义
- 包含 arpa/inet.h 头文件
- 包含 netdb.h 头文件
#endif /* _WIN32 */

#include <errno.h>  // 包含错误处理相关的头文件
#include <stdlib.h>  // 包含标准库函数的头文件
#include <string.h>  // 包含字符串处理相关的头文件
#include <stdio.h>   // 包含输入输出相关的头文件

#include "pcap-int.h"  // 包含 pcap 库的内部头文件

#include "diag-control.h"  // 包含诊断控制相关的头文件

#include "gencode.h"  // 包含生成代码相关的头文件
#include <pcap/namedb.h>  // 包含 pcap 的命名数据库相关的头文件
#include "nametoaddr.h"  // 包含地址转换相关的头文件

#ifdef HAVE_OS_PROTO_H
#include "os-proto.h"  // 如果定义了 HAVE_OS_PROTO_H，则包含操作系统协议相关的头文件
#endif

#ifndef NTOHL
#define NTOHL(x) (x) = ntohl(x)  // 如果未定义 NTOHL，则定义 NTOHL 宏
#define NTOHS(x) (x) = ntohs(x)  // 如果未定义 NTOHS，则定义 NTOHS 宏
#endif

/*
 *  Convert host name to internet address.
 *  Return 0 upon failure.
 *  XXX - not thread-safe; don't use it inside libpcap.
 */
bpf_u_int32 **
pcap_nametoaddr(const char *name)
{
#ifndef h_addr
    static bpf_u_int32 *hlist[2];  // 如果未定义 h_addr，则定义静态的 bpf_u_int32 数组
#endif
    bpf_u_int32 **p;  // 定义 bpf_u_int32 双重指针
    struct hostent *hp;  // 定义主机信息结构体指针

    /*
     * gethostbyname() is deprecated on Windows, perhaps because
     * it's not thread-safe, or because it doesn't support IPv6,
     * or both.
     *
     * We deprecate pcap_nametoaddr() on all platforms because
     * it's not thread-safe; we supply it for backwards compatibility,
     * so suppress the deprecation warning. We could, I guess,
     * use getaddrinfo() and construct the array ourselves, but
     * that's probably not worth the effort, as that wouldn't make
     * this thread-safe - we can't change the API to require that
     * our caller free the address array, so we still have to reuse
     * a local array.
     */
DIAG_OFF_DEPRECATION  // 关闭过时警告
    if ((hp = gethostbyname(name)) != NULL) {  // 如果根据主机名获取主机信息成功
DIAG_ON_DEPRECATION  // 打开过时警告
#ifndef h_addr
        hlist[0] = (bpf_u_int32 *)hp->h_addr;  // 如果未定义 h_addr，则将主机地址赋值给 hlist[0]
        NTOHL(hp->h_addr);  // 将主机地址转换为网络字节顺序
        return hlist;  // 返回 hlist 数组
#else
        for (p = (bpf_u_int32 **)hp->h_addr_list; *p; ++p)  // 遍历主机地址列表
            NTOHL(**p);  // 将主机地址转换为网络字节顺序
        return (bpf_u_int32 **)hp->h_addr_list;  // 返回主机地址列表
#endif
    }
    else
        return 0;  // 返回 0 表示失败
}

struct addrinfo *
pcap_nametoaddrinfo(const char *name)
{
    struct addrinfo hints, *res;  // 定义地址信息结构体和结果指针
    int error;  // 定义错误变量

    memset(&hints, 0, sizeof(hints));  // 将地址信息结构体清零
    hints.ai_family = PF_UNSPEC;  // 设置地址族为未指定
    hints.ai_socktype = SOCK_STREAM;    /*not really*/  // 设置套接字类型为流式套接字
    hints.ai_protocol = IPPROTO_TCP;    /*not really*/  // 设置协议为 TCP
    # 使用 getaddrinfo 函数获取指定主机名的地址信息，存储在 res 中
    error = getaddrinfo(name, NULL, &hints, &res);
    # 如果出现错误，返回空指针
    if (error)
        return NULL;
    # 否则返回获取的地址信息
    else
        return res;
}
/*
 *  Convert net name to internet address.
 *  Return 0 upon failure.
 *  XXX - not guaranteed to be thread-safe!  See below for platforms
 *  on which it is thread-safe and on which it isn't.
 */
#if defined(_WIN32) || defined(__CYGWIN__)
bpf_u_int32
pcap_nametonetaddr(const char *name _U_)
{
    /*
     * Windows平台没有"getnetbyname()"函数。
     *
     * XXX - 我猜我们可以使用BSD代码来读取C:\Windows\System32\drivers\etc\networks，
     * 假设这是所有我们使用的Windows版本的家园，但是这个文件可能只有99.44%的Windows机器上的回环网络127/24。
     *
     * (天哪，这些天它可能只有99.44%的*UN*X*机器上的回环网络。)
     */
    return 0;
}
#else /* _WIN32 */
bpf_u_int32
pcap_nametonetaddr(const char *name)
{
    /*
     * UN*X.
     */
    struct netent *np;
  #if defined(HAVE_LINUX_GETNETBYNAME_R)
    /*
     * 我们有Linux的可重入的getnetbyname_r()。
     */
    struct netent result_buf;
    char buf[1024];    /* 任意大小 */
    int h_errnoval;
    int err;

    /*
     * 显然，man页面上
     *
     *    http://man7.org/linux/man-pages/man3/getnetbyname_r.3.html
     *
     * 在说
     *
     *    如果函数调用成功获取了一个网络记录，那么*result将指向result_buf；否则，*result将被设置为NULL。
     *
     * 事实上，至少在某些版本的GNU libc中，如果getnetbyname_r()成功，它并不总是被设置。
     */
    np = NULL;
    err = getnetbyname_r(name, &result_buf, buf, sizeof buf, &np,
        &h_errnoval);
    if (err != 0) {
        /*
         * XXX - 动态分配缓冲区，并且如果我们得到ERANGE，就使它变得更大？
         */
        return 0;
    }
  #elif defined(HAVE_SOLARIS_IRIX_GETNETBYNAME_R)
    /*
     * 我们有Solaris和IRIX的可重入的getnetbyname_r()。
     */
    // 声明一个 netent 结构体变量 result_buf
    struct netent result_buf;
    // 声明一个大小为 1024 的字符数组 buf
    char buf[1024];    /* arbitrary size */

    // 如果定义了 HAVE_GNU_GETNETBYNAME_R，则使用 GNU 的 reentrant getnetbyname_r() 函数
    np = getnetbyname_r(name, &result_buf, buf, (int)sizeof buf);
  #elif defined(HAVE_AIX_GETNETBYNAME_R)
    /*
     * 我们有 AIX 的 reentrant getnetbyname_r()。
     */
    // 声明一个 netent 结构体变量 result_buf
    struct netent result_buf;
    // 声明一个 netent_data 结构体变量 net_data
    struct netent_data net_data;

    // 如果 getnetbyname_r() 返回 -1，则将 np 设为 NULL，否则将 np 设为 result_buf 的地址
    if (getnetbyname_r(name, &result_buf, &net_data) == -1)
        np = NULL;
    else
        np = &result_buf;
  #else
    /*
     * 我们没有任何 getnetbyname_r(); 要么我们有一个使用线程特定数据的 getnetbyname()，在这种情况下我们是线程安全的
     * （足够新的 FreeBSD、足够新的基于 Darwin 的操作系统、足够新的 HP-UX、足够新的 Tru64 UNIX），
     * 要么我们有传统的 getnetbyname()（包括当前的 NetBSD 和 OpenBSD），在这种情况下我们不是线程安全的。
     */
    // 使用 getnetbyname() 函数获取网络信息
    np = getnetbyname(name);
  #endif
    // 如果 np 不为 NULL，则返回 np->n_net，否则返回 0
    if (np != NULL)
        return np->n_net;
    else
        return 0;
}
#endif /* _WIN32 */

/*
 * Convert a port name to its port and protocol numbers.
 * We assume only TCP or UDP.
 * Return 0 upon failure.
 */
int
pcap_nametoport(const char *name, int *port, int *proto)
{
    struct addrinfo hints, *res, *ai;
    int error;
    struct sockaddr_in *in4;
#ifdef INET6
    struct sockaddr_in6 *in6;
#endif
    int tcp_port = -1;
    int udp_port = -1;

    /*
     * We check for both TCP and UDP in case there are
     * ambiguous entries.
     */
    memset(&hints, 0, sizeof(hints));  // 初始化 hints 结构体
    hints.ai_family = PF_UNSPEC;  // 设置地址族为未指定
    hints.ai_socktype = SOCK_STREAM;  // 设置套接字类型为流式套接字
    hints.ai_protocol = IPPROTO_TCP;  // 设置协议为 TCP
    error = getaddrinfo(NULL, name, &hints, &res);  // 获取地址信息
    if (error != 0) {
        if (error != EAI_NONAME &&
            error != EAI_SERVICE) {
            /*
             * This is a real error, not just "there's
             * no such service name".
             * XXX - this doesn't return an error string.
             */
            return 0;  // 如果出现错误，返回 0
        }
    } else {
        /*
         * OK, we found it.  Did it find anything?
         */
        for (ai = res; ai != NULL; ai = ai->ai_next) {
            /*
             * Does it have an address?
             */
            if (ai->ai_addr != NULL) {
                /*
                 * Yes.  Get a port number; we're done.
                 */
                if (ai->ai_addr->sa_family == AF_INET) {
                    in4 = (struct sockaddr_in *)ai->ai_addr;
                    tcp_port = ntohs(in4->sin_port);  // 获取 TCP 端口号
                    break;
                }
#ifdef INET6
                if (ai->ai_addr->sa_family == AF_INET6) {
                    in6 = (struct sockaddr_in6 *)ai->ai_addr;
                    tcp_port = ntohs(in6->sin6_port);  // 获取 TCP 端口号
                    break;
                }
#endif
            }
        }
        freeaddrinfo(res);  // 释放地址信息
    }

    memset(&hints, 0, sizeof(hints));  // 重新初始化 hints 结构体
    hints.ai_family = PF_UNSPEC;  // 设置地址族为未指定
    hints.ai_socktype = SOCK_DGRAM;  // 设置套接字类型为数据报套接字
    hints.ai_protocol = IPPROTO_UDP;  // 设置协议为 UDP
    # 调用 getaddrinfo 函数，获取指定名称的地址信息
    error = getaddrinfo(NULL, name, &hints, &res);
    # 如果返回错误码不为 0
    if (error != 0) {
        # 如果错误码不是 EAI_NONAME 和 EAI_SERVICE
        if (error != EAI_NONAME &&
            error != EAI_SERVICE) {
            '''
            这是一个真正的错误，不仅仅是“没有这样的服务名称”。
            XXX - 这不返回错误字符串。
            '''
            # 返回 0，表示出现了真正的错误
            return 0;
        }
    } else {
        '''
        好的，我们找到了。它找到了什么？
        '''
        # 遍历地址信息链表
        for (ai = res; ai != NULL; ai = ai->ai_next) {
            '''
            它有地址吗？
            '''
            # 如果地址信息不为空
            if (ai->ai_addr != NULL) {
                '''
                是的。获取端口号；我们完成了。
                '''
                # 如果地址族是 AF_INET
                if (ai->ai_addr->sa_family == AF_INET) {
                    # 将地址信息转换为 IPv4 地址结构体
                    in4 = (struct sockaddr_in *)ai->ai_addr;
                    # 获取端口号，并转换为主机字节顺序
                    udp_port = ntohs(in4->sin_port);
                    # 跳出循环
                    break;
                }
#ifdef INET6
                // 如果支持IPv6
                if (ai->ai_addr->sa_family == AF_INET6) {
                    // 将地址信息转换为IPv6地址结构
                    in6 = (struct sockaddr_in6 *)ai->ai_addr;
                    // 获取UDP端口号
                    udp_port = ntohs(in6->sin6_port);
                    // 跳出循环
                    break;
                }
#endif
            }
        }
        // 释放地址信息链表
        freeaddrinfo(res);
    }

    /*
     * 需要检查 /etc/services 是否存在歧义的条目。
     * 如果找到歧义的条目，并且端口号相同，则将协议更改为 PROTO_UNDEF，
     * 这样就会同时检查 TCP 和 UDP。
     */
    if (tcp_port >= 0) {
        // 设置端口号为 TCP 端口号
        *port = tcp_port;
        // 设置协议为 IPPROTO_TCP
        *proto = IPPROTO_TCP;
        if (udp_port >= 0) {
            // 如果 UDP 端口号和 TCP 端口号相同
            if (udp_port == tcp_port)
                // 将协议更改为 PROTO_UNDEF
                *proto = PROTO_UNDEF;
#ifdef notdef
            else
                /* 无法处理引用不同端口号的歧义名称。 */
                warning("ambiguous port %s in /etc/services",
                    name);
#endif
        }
        return 1;
    }
    // 如果存在 UDP 端口号
    if (udp_port >= 0) {
        // 设置端口号为 UDP 端口号
        *port = udp_port;
        // 设置协议为 IPPROTO_UDP
        *proto = IPPROTO_UDP;
        return 1;
    }
#if defined(ultrix) || defined(__osf__)
    /* 如果 NFS 不在 /etc/services 中，则进行特殊处理 */
    if (strcmp(name, "nfs") == 0) {
        // 设置端口号为 2049
        *port = 2049;
        // 设置协议为 PROTO_UNDEF
        *proto = PROTO_UNDEF;
        return 1;
    }
#endif
    return 0;
}

/*
 * 将形式为 PPP-PPP 的字符串转换为端口范围的起始和结束端口。
 * 失败时返回 0。
 */
int
pcap_nametoportrange(const char *name, int *port1, int *port2, int *proto)
{
    u_int p1, p2;
    char *off, *cpy;
    int save_proto;
    # 如果传入的name参数不能按照指定格式"%d-%d"解析出两个整数，则执行以下操作
    if (sscanf(name, "%d-%d", &p1, &p2) != 2) {
        # 如果无法分配内存给复制的name字符串，则返回0
        if ((cpy = strdup(name)) == NULL)
            return 0;
        # 如果在复制的name字符串中找不到'-'字符，则释放内存并返回0
        if ((off = strchr(cpy, '-')) == NULL) {
            free(cpy);
            return 0;
        }
        # 将'-'字符替换为字符串结束符'\0'
        *off = '\0';
        # 根据复制的name字符串获取端口号和协议类型，如果失败则释放内存并返回0
        if (pcap_nametoport(cpy, port1, proto) == 0) {
            free(cpy);
            return 0;
        }
        # 保存协议类型
        save_proto = *proto;
        # 根据'-'后面的字符串获取端口号和协议类型，如果失败则释放内存并返回0
        if (pcap_nametoport(off + 1, port2, proto) == 0) {
            free(cpy);
            return 0;
        }
        # 释放内存
        free(cpy);
        # 如果协议类型不等于保存的协议类型，则将协议类型设置为PROTO_UNDEF
        if (*proto != save_proto)
            *proto = PROTO_UNDEF;
    } 
    # 如果能按照指定格式"%d-%d"解析出两个整数，则执行以下操作
    else {
        # 将解析出的两个整数分别赋值给port1和port2，将协议类型设置为PROTO_UNDEF
        *port1 = p1;
        *port2 = p2;
        *proto = PROTO_UNDEF;
    }
    # 返回1
    return 1;
}
/*
 * XXX - not guaranteed to be thread-safe!  See below for platforms
 * on which it is thread-safe and on which it isn't.
 */
int
pcap_nametoproto(const char *str)
{
    struct protoent *p;
  #if defined(HAVE_LINUX_GETNETBYNAME_R)
    /*
     * We have Linux's reentrant getprotobyname_r().
     */
    struct protoent result_buf;
    char buf[1024];    /* arbitrary size */
    int err;

    err = getprotobyname_r(str, &result_buf, buf, sizeof buf, &p);
    if (err != 0) {
        /*
         * XXX - dynamically allocate the buffer, and make it
         * bigger if we get ERANGE back?
         */
        return 0;
    }
  #elif defined(HAVE_SOLARIS_IRIX_GETNETBYNAME_R)
    /*
     * We have Solaris's and IRIX's reentrant getprotobyname_r().
     */
    struct protoent result_buf;
    char buf[1024];    /* arbitrary size */

    p = getprotobyname_r(str, &result_buf, buf, (int)sizeof buf);
  #elif defined(HAVE_AIX_GETNETBYNAME_R)
    /*
     * We have AIX's reentrant getprotobyname_r().
     */
    struct protoent result_buf;
    struct protoent_data proto_data;

    if (getprotobyname_r(str, &result_buf, &proto_data) == -1)
        p = NULL;
    else
        p = &result_buf;
  #else
    /*
     * We don't have any getprotobyname_r(); either we have a
     * getprotobyname() that uses thread-specific data, in which
     * case we're thread-safe (sufficiently recent FreeBSD,
     * sufficiently recent Darwin-based OS, sufficiently recent
     * HP-UX, sufficiently recent Tru64 UNIX, Windows), or we have
     * the traditional getprotobyname() (everything else, including
     * current NetBSD and OpenBSD), in which case we're not
     * thread-safe.
     */
    p = getprotobyname(str);
  #endif
    if (p != 0)
        return p->p_proto;
    else
        return PROTO_UNDEF;
}

#include "ethertype.h"

struct eproto {
    const char *s;
    u_short p;
};
/*
 * 静态数据基础的以太网协议类型。
 * tcpdump 用于导入此数据，并且至少在 Debian 上声明为导出，因此将其作为公共符号，即使我们没有通过在头文件中声明它来正式导出它。
 * （程序*应该*自己做这个，就像 tcpdump 现在做的那样。）
 *
 * 我们在这里声明它，就在定义它之前，以消除编译器可能发出的关于缺少声明的任何警告。
 */
PCAP_API struct eproto eproto_db[];
PCAP_API_DEF struct eproto eproto_db[] = {
    { "aarp", ETHERTYPE_AARP },
    { "arp", ETHERTYPE_ARP },
    { "atalk", ETHERTYPE_ATALK },
    { "decnet", ETHERTYPE_DN },
    { "ip", ETHERTYPE_IP },
#ifdef INET6
    { "ip6", ETHERTYPE_IPV6 },
#endif
    { "lat", ETHERTYPE_LAT },
    { "loopback", ETHERTYPE_LOOPBACK },
    { "mopdl", ETHERTYPE_MOPDL },
    { "moprc", ETHERTYPE_MOPRC },
    { "rarp", ETHERTYPE_REVARP },
    { "sca", ETHERTYPE_SCA },
    { (char *)0, 0 }
};

int
pcap_nametoeproto(const char *s)
{
    struct eproto *p = eproto_db;

    while (p->s != 0) {
        if (strcmp(p->s, s) == 0)
            return p->p;
        p += 1;
    }
    return PROTO_UNDEF;
}

#include "llc.h"

/* 静态数据基础的 LLC 值。 */
static struct eproto llc_db[] = {
    { "iso", LLCSAP_ISONS },
    { "stp", LLCSAP_8021D },
    { "ipx", LLCSAP_IPX },
    { "netbeui", LLCSAP_NETBEUI },
    { (char *)0, 0 }
};

int
pcap_nametollc(const char *s)
{
    struct eproto *p = llc_db;

    while (p->s != 0) {
        if (strcmp(p->s, s) == 0)
            return p->p;
        p += 1;
    }
    return PROTO_UNDEF;
}

/* 十六进制数字转换为 8 位无符号整数。 */
static inline u_char
xdtoi(u_char c)
{
    if (c >= '0' && c <= '9')
        return (u_char)(c - '0');
    else if (c >= 'a' && c <= 'f')
        return (u_char)(c - 'a' + 10);
    else
        return (u_char)(c - 'A' + 10);
}

int
__pcap_atoin(const char *s, bpf_u_int32 *addr)
{
    u_int n;
    int len;

    *addr = 0;
    len = 0;
    # 无限循环，直到条件不满足
    for (;;) {
        # 初始化 n 为 0
        n = 0;
        # 当字符串不为空且不是 '.' 时执行循环
        while (*s && *s != '.') {
            # 如果 n 大于 25，则返回 -1
            if (n > 25) {
                /* 结果将大于 255 */
                return -1;
            }
            # 计算 n 的值
            n = n * 10 + *s++ - '0';
        }
        # 如果 n 大于 255，则返回 -1
        if (n > 255)
            return -1;
        # 将地址左移 8 位，并将 n 的低 8 位加到地址上
        *addr <<= 8;
        *addr |= n & 0xff;
        # 增加长度 8
        len += 8;
        # 如果字符串为空，则返回长度
        if (*s == '\0')
            return len;
        # 移动到下一个字符
        ++s;
    }
    /* 不会执行到这里 */
}
// 定义一个名为__pcap_atodn的函数，参数为const char *s和bpf_u_int32 *addr
int
__pcap_atodn(const char *s, bpf_u_int32 *addr)
{
// 定义宏AREASHIFT为10
#define AREASHIFT 10
// 定义宏AREAMASK为0176000
#define AREAMASK 0176000
// 定义宏NODEMASK为01777
#define NODEMASK 01777

    // 定义无符号整型变量node和area
    u_int node, area;

    // 如果sscanf函数返回值不等于2，则返回0
    if (sscanf(s, "%d.%d", &area, &node) != 2)
        return(0);

    // 将area左移AREASHIFT位并与AREAMASK进行位与操作，结果赋给*addr
    *addr = (area << AREASHIFT) & AREAMASK;
    // 将node与NODEMASK进行位与操作，结果赋给*addr
    *addr |= (node & NODEMASK);

    // 返回32
    return(32);
}

/*
 * Convert 's', which can have the one of the forms:
 *
 *    "xx:xx:xx:xx:xx:xx"
 *    "xx.xx.xx.xx.xx.xx"
 *    "xx-xx-xx-xx-xx-xx"
 *    "xxxx.xxxx.xxxx"
 *    "xxxxxxxxxxxx"
 *
 * (or various mixes of ':', '.', and '-') into a new
 * ethernet address.  Assumes 's' is well formed.
 */
// 定义一个名为pcap_ether_aton的函数，参数为const char *s
u_char *
pcap_ether_aton(const char *s)
{
    // 定义无符号字符指针ep和e
    register u_char *ep, *e;
    // 定义无符号字符d
    register u_char d;

    // 为e分配6个字节的内存空间
    e = ep = (u_char *)malloc(6);
    // 如果e为空，则返回空指针
    if (e == NULL)
        return (NULL);

    // 循环处理s中的字符
    while (*s) {
        // 如果s为冒号、句号或短横线，则s向后移动一位
        if (*s == ':' || *s == '.' || *s == '-')
            s += 1;
        // 将*s转换为对应的整数值，赋给d
        d = xdtoi(*s++);
        // 如果*s为十六进制数字，则将其左移4位并与*s转换为对应的整数值进行位或操作，结果赋给d
        if (PCAP_ISXDIGIT(*s)) {
            d <<= 4;
            d |= xdtoi(*s++);
        }
        // 将d赋给*ep，并ep向后移动一位
        *ep++ = d;
    }

    // 返回e
    return (e);
}

// 如果没有定义HAVE_ETHER_HOSTTON，则执行以下代码块
#ifndef HAVE_ETHER_HOSTTON
/*
 * Roll our own.
 * XXX - not thread-safe, because pcap_next_etherent() isn't thread-
 * safe!  Needs a mutex or a thread-safe pcap_next_etherent().
 */
// 定义一个名为pcap_ether_hostton的函数，参数为const char *name
u_char *
pcap_ether_hostton(const char *name)
{
    // 定义结构体指针ep和无符号字符指针ap
    register struct pcap_etherent *ep;
    register u_char *ap;
    // 定义静态文件指针fp和整型变量init
    static FILE *fp = NULL;
    static int init = 0;

    // 如果init为假，则打开PCAP_ETHERS_FILE文件，并将init加1，如果fp为空，则返回空指针
    if (!init) {
        fp = fopen(PCAP_ETHERS_FILE, "r");
        ++init;
        if (fp == NULL)
            return (NULL);
    } else if (fp == NULL)
        return (NULL);
    else
        rewind(fp);

    // 循环处理pcap_next_etherent(fp)的返回值
    while ((ep = pcap_next_etherent(fp)) != NULL) {
        // 如果ep的name与name相等，则为ap分配6个字节的内存空间，将ep的addr的内容复制到ap，并返回ap
        if (strcmp(ep->name, name) == 0) {
            ap = (u_char *)malloc(6);
            if (ap != NULL) {
                memcpy(ap, ep->addr, 6);
                return (ap);
            }
            break;
        }
    }
    // 返回空指针
    return (NULL);
}
#else
/*
 * 使用操作系统提供的例程。
 * 这应该是线程安全的；API没有静态缓冲区。
 */
u_char *
pcap_ether_hostton(const char *name)
{
    register u_char *ap;
    u_char a[6];
    char namebuf[1024];

    /*
     * 在 AIX 7.1 和 7.2 中：int ether_hostton(char *, struct ether_addr *);
     */
    pcap_strlcpy(namebuf, name, sizeof(namebuf));
    ap = NULL;
    if (ether_hostton(namebuf, (struct ether_addr *)a) == 0) {
        ap = (u_char *)malloc(6);
        if (ap != NULL)
            memcpy((char *)ap, (char *)a, 6);
    }
    return (ap);
}
#endif

/*
 * XXX - 不保证是线程安全的！
 */
int
#ifdef    DECNETLIB
__pcap_nametodnaddr(const char *name, u_short *res)
{
    struct nodeent *getnodebyname();
    struct nodeent *nep;

    nep = getnodebyname(name);
    if (nep == ((struct nodeent *)0))
        return(0);

    memcpy((char *)res, (char *)nep->n_addr, sizeof(unsigned short));
    return(1);
#else
__pcap_nametodnaddr(const char *name _U_, u_short *res _U_)
{
    return(0);
#endif
}
```