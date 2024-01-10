# `nmap\ncat\util.c`

```
/* $Id$ */

#include "sys_wrap.h"  // 包含系统封装的头文件
#include "util.h"  // 包含工具函数的头文件
#include "ncat.h"  // 包含ncat的头文件
#include "nbase.h"  // 包含nbase的头文件
#include "sockaddr_u.h"  // 包含sockaddr_u的头文件

#include <stdio.h>  // 标准输入输出库
#ifdef WIN32
#include <iphlpapi.h>  // Windows平台的IP辅助库
#endif
#include <stdlib.h>  // 标准库
#include <stdarg.h>  // 提供了一个宏 va_list 和三个函数，用于支持可变参数的函数
#include <string.h>  // 字符串处理库
#include <stddef.h>  // 标准定义库

#if HAVE_SYS_STAT_H
#include <sys/stat.h>  // 系统状态库
#endif
#if HAVE_FCNTL_H
#include <fcntl.h>  // 文件控制库
#endif
#if HAVE_UNISTD_H
#include <unistd.h>  // Unix标准库
#endif

#if HAVE_LINUX_VM_SOCKETS_H
#include <linux/vm_sockets.h>  // Linux虚拟内存套接字库
#endif

/* 安全地将两个size_t相加 */
size_t sadd(size_t l, size_t r)
{
    size_t t;

    t = l + r;
    if (t < l)
        bye("integer overflow %lu + %lu.", (u_long) l, (u_long) r);  // 如果相加后的值小于其中一个操作数，抛出异常
    return t;
}

/* 安全地将两个size_t相乘 */
size_t smul(size_t l, size_t r)
{
    size_t t;

    t = l * r;
    if (l && t / l != r)
        bye("integer overflow %lu * %lu.", (u_long) l, (u_long) r);  // 如果相乘后的值不等于其中一个操作数，抛出异常
    return t;
}

#ifdef WIN32
void windows_init()
{
    WORD werd;
    WSADATA data;

    werd = MAKEWORD(2, 2);
    if ((WSAStartup(werd, &data)) != 0)
        bye("Failed to start WinSock.");  // 如果启动WinSock失败，抛出异常
}
#endif

/* 用于打印调试或诊断消息，避免污染用户流 */
void loguser(const char *fmt, ...)
{
    va_list ap;

    fprintf(stderr, "%s: ", NCAT_NAME);  // 打印Ncat名称
    va_start(ap, fmt);
    vfprintf(stderr, fmt, ap);  // 格式化打印消息
    va_end(ap);
    fflush(stderr);  // 刷新标准错误流
}

/* 记录用户消息，不带“Ncat:”前缀，以允许构建一系列字符串的行 */
void loguser_noprefix(const char *fmt, ...)
{
    va_list ap;

    va_start(ap, fmt);
    vfprintf(stderr, fmt, ap);  // 格式化打印消息
    va_end(ap);
    fflush(stderr);  // 刷新标准错误流
}

void logdebug(const char *fmt, ...)
{
    va_list ap;

    fprintf(stderr, "NCAT DEBUG: ");  // 打印调试消息前缀
    va_start(ap, fmt);
    vfprintf(stderr, fmt, ap);  // 格式化打印消息
    va_end(ap);
    fflush(stderr);  // 刷新标准错误流
}

void logtest(const char *fmt, ...)
{
    va_list ap;

    fprintf(stderr, "NCAT TEST: ");  // 打印测试消息前缀
    va_start(ap, fmt);
    vfprintf(stderr, fmt, ap);  // 格式化打印消息
    va_end(ap);
    fflush(stderr);  // 刷新标准错误流
}
/* Exit status 2 indicates a program error other than a network error. */
void die(char *err)
{
#ifdef WIN32
  int error_number;
  char *strerror_s;
  error_number = GetLastError();
  FormatMessage(FORMAT_MESSAGE_ALLOCATE_BUFFER|FORMAT_MESSAGE_FROM_SYSTEM,
      NULL, error_number, MAKELANGID (LANG_NEUTRAL, SUBLANG_DEFAULT),
      (LPTSTR) &strerror_s,  0, NULL);
  fprintf(stderr, "%s: %s\n", err, strerror_s);
  HeapFree(GetProcessHeap(), 0, strerror_s);
#else
    perror(err);
#endif
    fflush(stderr);
    exit(2);
}

/* adds newline for you */
void bye(const char *fmt, ...)
{
    va_list ap;

    fprintf(stderr, "%s: ", NCAT_NAME);
    va_start(ap, fmt);
    vfprintf(stderr, fmt, ap);
    va_end(ap);
    fprintf(stderr, " QUITTING.\n");
    fflush(stderr);

    exit(2);
}

/* zero out some mem, bzero() is deprecated */
void zmem(void *mem, size_t n)
{
    memset(mem, 0, n);
}

/* Append n bytes starting at s to a malloc-allocated buffer. Reallocates the
   buffer and updates the variables to make room if necessary. */
int strbuf_append(char **buf, size_t *size, size_t *offset, const char *s, size_t n)
{
    ncat_assert(*offset <= *size);

    if (n >= *size - *offset) {
        *size += n + 1;
        *buf = (char *) safe_realloc(*buf, *size);
    }

    memcpy(*buf + *offset, s, n);
    *offset += n;
    (*buf)[*offset] = '\0';

    return n;
}

/* Append a '\0'-terminated string as with strbuf_append. */
int strbuf_append_str(char **buf, size_t *size, size_t *offset, const char *s)
{
    return strbuf_append(buf, size, offset, s, strlen(s));
}

/* Do a sprintf at the given offset into a malloc-allocated buffer. Reallocates
   the buffer and updates the variables to make room if necessary. */
int strbuf_sprintf(char **buf, size_t *size, size_t *offset, const char *fmt, ...)
{
    va_list va;
    int n;

    ncat_assert(*offset <= *size);

    if (*buf == NULL) {
        *size = 1;
        *buf = (char *) safe_malloc(*size);
    }
    # 无限循环，直到条件被打破
    for (;;) {
        # 初始化可变参数列表
        va_start(va, fmt);
        # 格式化字符串并将结果存储到缓冲区中
        n = Vsnprintf(*buf + *offset, *size - *offset, fmt, va);
        # 结束可变参数列表的使用
        va_end(va);
        # 如果格式化失败，重新分配缓冲区大小
        if (n < 0)
            *size = MAX(*size, 1) * 2;
        # 如果格式化结果超出缓冲区大小，增加缓冲区大小
        else if (n >= *size - *offset)
            *size += n + 1;
        # 格式化成功，跳出循环
        else
            break;
        # 重新分配缓冲区大小
        *buf = (char *) safe_realloc(*buf, *size);
    }
    # 更新偏移量
    *offset += n;

    # 返回格式化结果
    return n;
/* 如果给定地址是本地地址，则返回 true */
int addr_is_local(const union sockaddr_u *su)
{
    // 初始化地址信息结构体和其他变量
    struct addrinfo hints = { 0 }, *addrs, *addr;
    char hostname[128];

    /* 检查回环地址 */
    if (su->storage.ss_family == AF_INET) {
        if ((ntohl(su->in.sin_addr.s_addr) & 0xFF000000UL) == 0x7F000000UL)
            return 1;
        if (ntohl(su->in.sin_addr.s_addr) == 0x00000000UL)
            return 1;
    }
#ifdef HAVE_IPV6
    else if (su->storage.ss_family == AF_INET6) {
        if (memcmp(&su->in6.sin6_addr, &in6addr_any, sizeof(su->in6.sin6_addr)) == 0
            || memcmp(&su->in6.sin6_addr, &in6addr_loopback, sizeof(su->in6.sin6_addr)) == 0)
            return 1;
    }
#endif

    /* 检查分配给本地主机名的地址 */
    if (gethostname(hostname, sizeof(hostname)) == -1)
        return 0;
    hints.ai_family = su->storage.ss_family;
    if (getaddrinfo(hostname, NULL, &hints, &addrs) != 0)
        return 0;
    for (addr = addrs; addr != NULL; addr = addr->ai_next) {
        union sockaddr_u addr_su;

        if (addr->ai_family != su->storage.ss_family)
            continue;
        if (addr->ai_addrlen > sizeof(addr_su)) {
            bye("getaddrinfo returned oversized address (%lu > %lu)",
                (unsigned long) addr->ai_addrlen, (unsigned long) sizeof(addr_su));
        }
        memcpy(&addr_su, addr->ai_addr, addr->ai_addrlen);
        if (su->storage.ss_family == AF_INET) {
            if (su->in.sin_addr.s_addr == addr_su.in.sin_addr.s_addr)
                break;
        } else if (su->storage.ss_family == AF_INET6) {
            if (memcmp(&su->in6.sin6_addr, &addr_su.in6.sin6_addr, sizeof(su->in6.sin6_addr)) == 0)
                break;
        }
    }
    if (addr != NULL) {
        freeaddrinfo(addrs);
        return 1;
    } else {
        return 0;
    }
}
/* 将 sockaddr_u 转换为字符串表示形式。由于返回的是静态缓冲区，因此不是线程安全的，只能在类似 printf() 的调用中使用一次。如果 ss_len 为 0，则可能是未知的。 */
const char *socktop(const union sockaddr_u *su, socklen_t ss_len)
{
    static char buf[INET6_ADDRSTRLEN + sizeof(union sockaddr_u)];  // 静态缓冲区，用于存储字符串表示形式
    size_t size = sizeof(buf);  // 缓冲区大小

    switch (su->storage.ss_family) {  // 根据地址族类型进行切换
#if HAVE_SYS_UN_H
        case AF_UNIX:  // UNIX 域套接字
            ncat_assert(ss_len <= sizeof(struct sockaddr_un));  // 断言，确保长度不超过 sockaddr_un 结构体的大小
            if (ss_len == sizeof(sa_family_t)) {
                /* 未命名套接字 */
                Strncpy(buf, "(unnamed socket)", sizeof(buf));  // 将字符串复制到缓冲区
            }
            else {
                if (ss_len < sizeof(sa_family_t)) {
                    /* 套接字路径不一定有效，但我们会尝试。 */
                    size = sizeof(su->un.sun_path);  // 设置缓冲区大小为套接字路径的大小
                }
                else {
                    /* 我们将在 size + 1 处添加空终止符，以防它丢失。 */
                    size = MIN(sizeof(buf) - 1, ss_len - offsetof(struct sockaddr_un, sun_path));  // 设置缓冲区大小为最小值
                }
                if (su->un.sun_path[0] == '\0') {
                    /* 抽象套接字（Linux 扩展） */
                    memcpy(buf, su->un.sun_path + 1, size - 1);  // 复制套接字路径到缓冲区
                    Strncpy(buf + size, " (abstract socket)", sizeof(buf) - size);  // 将字符串追加到缓冲区末尾
                }
                else {
                    memcpy(buf, su->un.sun_path, size);  // 复制套接字路径到缓冲区
                    buf[size+1] = '\0';  // 添加空终止符
                }
                /* 如果我们得到了垃圾数据，使其安全。 */
                replacenonprintable(buf, strlen(buf), '?');  // 替换非打印字符
            }
            break;
#endif
#ifdef HAVE_LINUX_VM_SOCKETS_H
        case AF_VSOCK:  // VSOCK 套接字
            Snprintf(buf, sizeof(buf), "%u:%u", su->vm.svm_cid, su->vm.svm_port);  // 格式化字符串并将结果存储在缓冲区中
            break;
#endif
        case AF_INET:
            # 如果地址族是 AF_INET，使用 inet_socktop() 和 inet_port() 函数格式化 IP 地址和端口号
            Snprintf(buf, sizeof(buf), "%s:%hu", inet_socktop(su), inet_port(su));
            break;
        case AF_INET6:
            # 如果地址族是 AF_INET6，使用 inet_socktop() 和 inet_port() 函数格式化 IP 地址和端口号
            Snprintf(buf, sizeof(buf), "[%s]:%hu", inet_socktop(su), inet_port(su));
            break;
        default:
            # 如果地址族不是 AF_INET 或 AF_INET6，返回空指针
            return NULL;
            break;
    }
    # 返回格式化后的 IP 地址和端口号
    return buf;
}

/* Converts an IP address given in a sockaddr_u to an IPv4 or
   IPv6 IP address string.  Since a static buffer is returned, this is
   not thread-safe and can only be used once in calls like printf()
*/
const char *inet_socktop(const union sockaddr_u *su)
{
    # 创建一个静态缓冲区来存储 IP 地址字符串
    static char buf[INET6_ADDRSTRLEN + 1];
    void *addr;

    # 根据地址族类型获取地址
    if (su->storage.ss_family == AF_INET)
        addr = (void *) &su->in.sin_addr;
#if HAVE_IPV6
    else if (su->storage.ss_family == AF_INET6)
        addr = (void *) &su->in6.sin6_addr;
#endif
    else
        # 如果地址族类型无效，抛出异常
        bye("Invalid address family passed to inet_socktop().");

    # 将地址转换为可读的 IP 地址字符串
    if (inet_ntop(su->storage.ss_family, addr, buf, sizeof(buf)) == NULL) {
        bye("Failed to convert address to presentation format!  Error: %s.",
            strerror(socket_errno()));
    }

    # 返回 IP 地址字符串
    return buf;
}

/* Returns the port number in HOST BYTE ORDER based on the su's family */
unsigned short inet_port(const union sockaddr_u *su)
{
    switch (su->storage.ss_family) {
        case AF_INET:
            # 如果地址族是 AF_INET，返回端口号（主机字节顺序）
            return ntohs(su->in.sin_port);
            break;
#if HAVE_IPV6
        case AF_INET6:
            # 如果地址族是 AF_INET6，返回端口号（主机字节顺序）
            return ntohs(su->in6.sin6_port);
            break;
#endif
        default:
            # 如果地址族类型无效，抛出异常
            bye("Invalid address family passed to inet_port().");
            break;
    }
    # 如果无法确定地址族类型，返回 0
    return 0;
}

/* Return a listening socket after setting various characteristics on it.
   Returns -1 on error. */
int do_listen(int type, int proto, const union sockaddr_u *srcaddr_u)
{
    int sock = 0, option_on = 1;
    size_t sa_len;

    # 如果类型不是 SOCK_STREAM 或 SOCK_DGRAM，返回错误
    if (type != SOCK_STREAM && type != SOCK_DGRAM)
        return -1;
    /* 我们需要一个可以被子进程继承的套接字，在 ncat_exec_win.c 中使用，用于 --exec 和 --sh-exec。inheritable_socket 来自于 nbase。*/
    // 使用 inheritable_socket 函数创建一个套接字，根据地址家族、类型和协议来确定
    sock = inheritable_socket(srcaddr_u->storage.ss_family, type, proto);
    // 如果创建套接字失败，返回 -1
    if (sock < 0)
        return -1;

    // 设置套接字选项，允许地址重用
    Setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, &option_on, sizeof(int));
/* IPPROTO_IPV6 is defined in Visual C++ only when _WIN32_WINNT >= 0x501.
   Nbase's nbase_winunix.h defines _WIN32_WINNT to a lower value for
   compatibility with older versions of Windows. This code disables IPv6 sockets
   that also receive IPv4 connections. This is the default on Windows anyway so
   it doesn't make a difference.
   http://support.microsoft.com/kb/950688
   http://msdn.microsoft.com/en-us/library/bb513665
*/
#ifdef IPPROTO_IPV6
#ifdef IPV6_V6ONLY
    // 如果地址族是 AF_INET6，设置 IPV6_V6ONLY 选项，告诉它不要尝试绑定到 IPV4
    if (srcaddr_u->storage.ss_family == AF_INET6) {
        int set = 1;
        if (setsockopt(sock, IPPROTO_IPV6, IPV6_V6ONLY, &set, sizeof(set)) == -1)
            die("Unable to set IPV6 socket to bind only to IPV6");
    }
#endif
#endif

    // 获取地址结构体的长度
    sa_len = get_socklen(srcaddr_u);

    // 将套接字绑定到指定地址
    if (bind(sock, &srcaddr_u->sockaddr, sa_len) < 0) {
        bye("bind to %s: %s.", socktop(srcaddr_u, sa_len),
                socket_strerror(socket_errno()));
    }

    // 如果是流式套接字，开始监听连接
    if (type == SOCK_STREAM)
        Listen(sock, BACKLOG);

    // 如果设置了详细输出，记录监听的地址
    if (o.verbose) {
        loguser("Listening on %s\n", socktop(srcaddr_u, sa_len));
    }
    // 如果是测试模式，记录监听操作
    if (o.test)
        logtest("LISTEN\n");

    // 返回套接字
    return sock;
}

/* Only used in proxy connect functions, so doesn't need to support address
 * families that don't support proxying like AF_UNIX and AF_VSOCK */
int do_connect(int type)
{
    int sock = 0;

    // 如果不是流式套接字或数据报套接字，返回错误
    if (type != SOCK_STREAM && type != SOCK_DGRAM)
        return -1;

    /* We need a socket that can be inherited by child processes in
       ncat_exec_win.c, for --exec and --sh-exec. inheritable_socket is from
       nbase. */
    // 获取可继承的套接字
    sock = inheritable_socket(targetaddrs->addr.storage.ss_family, type, 0);

    // 如果有指定源地址，将套接字绑定到指定地址
    if (srcaddr.storage.ss_family != AF_UNSPEC) {
        size_t sa_len = get_socklen(&srcaddr);

        if (bind(sock, &srcaddr.sockaddr, sa_len) < 0) {
            bye("bind to %s: %s.", socktop(&srcaddr, sa_len),
                    socket_strerror(socket_errno()));
        }
    }
    # 如果套接字不等于-1，则执行以下操作
    if (sock != -1) {
        # 尝试连接到目标地址
        if (connect(sock, &targetaddrs->addr.sockaddr, (int) targetaddrs->addrlen) != -1)
            # 如果连接成功，则返回套接字
            return sock;
        # 如果连接出现非阻塞错误或者资源暂时不可用错误，则返回套接字
        else if (socket_errno() == EINPROGRESS || socket_errno() == EAGAIN)
            return sock;
    }
    # 如果套接字为-1或者连接失败，则返回-1
    return -1;
}

unsigned char *buildsrcrte(struct in_addr dstaddr, struct in_addr routes[],
                  int numroutes, int ptr, size_t *len)
{
    int x;
    unsigned char *opts, *p;

    *len = (numroutes + 1) * sizeof(struct in_addr) + 4;  // 计算需要分配的内存空间大小

    if (numroutes > 8)  // 如果路由数量大于8，则报错
        bye("Bad number of routes passed to buildsrcrte().");

    opts = (unsigned char *) safe_malloc(*len);  // 分配内存空间
    p = opts;  // 指针指向分配的内存空间

    zmem(opts, *len);  // 将分配的内存空间清零

    *p++ = 0x01; /* IPOPT_NOP, for alignment */  // 设置选项字段为IPOPT_NOP，用于对齐
    *p++ = 0x83; /* IPOPT_LSRR */  // 设置选项字段为IPOPT_LSRR
    *p++ = (char) (*len - 1); /* subtract nop */  // 设置选项字段长度
    *p++ = (char) ptr;  // 设置指针字段

    for (x = 0; x < numroutes; x++) {  // 遍历路由数组
        memcpy(p, &routes[x], sizeof(routes[x]));  // 将路由信息复制到内存空间中
        p += sizeof(routes[x]);  // 指针移动到下一个位置
    }

    memcpy(p, &dstaddr, sizeof(dstaddr));  // 将目标地址信息复制到内存空间中

    return opts;  // 返回构建好的选项字段
}

int allow_access(const union sockaddr_u *su)
{
    /* A host not in the allow set is denied, but only if the --allow or
       --allowfile option was given. */
    if (o.allow && !addrset_contains(o.allowset, &su->sockaddr))  // 如果不在允许的地址集合中，则拒绝访问
        return 0;
    if (addrset_contains(o.denyset, &su->sockaddr))  // 如果在拒绝的地址集合中，则拒绝访问
        return 0;

    return 1;  // 允许访问
}

/*
 * Fills the given timeval struct with proper
 * values based on the given time in milliseconds.
 * The pointer to timeval struct must NOT be NULL.
 */
void ms_to_timeval(struct timeval *tv, long ms)
{
    tv->tv_sec = ms / 1000;  // 将毫秒转换为秒
    tv->tv_usec = (ms - (tv->tv_sec * 1000)) * 1000;  // 计算微秒
}

/*
 * ugly code to maintain our list of fds so we can have proper fdmax for
 * select().  really this should be generic list code, not this silly bit of
 * stupidity. -sean
 */

/* add an fdinfo to our list */
int add_fdinfo(fd_list_t *fdl, struct fdinfo *s)
{
    if (fdl->nfds >= fdl->maxfds)  // 如果当前文件描述符数量大于等于最大文件描述符数量，则返回错误
        return -1;

    fdl->fds[fdl->nfds] = *s;  // 将文件描述符信息添加到列表中

    fdl->nfds++;  // 文件描述符数量加一

    if (s->fd > fdl->fdmax)  // 如果当前文件描述符大于最大文件描述符，则更新最大文件描述符
        fdl->fdmax = s->fd;

    if (o.debug > 1)  // 如果调试级别大于1
        logdebug("Added fd %d to list, nfds %d, maxfd %d\n", s->fd, fdl->nfds, fdl->fdmax);  // 记录日志
    return 0;  // 返回成功
}
/* Add a descriptor to the list. Use this when you are only adding to the list
 * for the side effect of increasing fdmax, and don't care about fdinfo
 * members. */
// 向列表中添加描述符。当您只是为了增加fdmax而向列表添加时使用此函数，不关心fdinfo成员。

int add_fd(fd_list_t *fdl, int fd)
{
    struct fdinfo info = { 0 }; // 创建一个fdinfo结构体变量info，并初始化为0

    info.fd = fd; // 将传入的fd赋值给info结构体的fd成员

    return add_fdinfo(fdl, &info); // 调用add_fdinfo函数，将fdl和info的地址作为参数传入
}

/* remove a descriptor from our list */
// 从列表中移除描述符
int rm_fd(fd_list_t *fdl, int fd)
{
    int x = 0, last = fdl->nfds; // 初始化变量x为0，last为fdl中的nfds成员
    int found = -1; // 初始化变量found为-1，用于标记是否找到了要移除的fd
    int newfdmax = 0; // 初始化变量newfdmax为0，用于记录新的最大fd

    /* make sure we have a list */
    // 确保我们有一个列表
    if (last == 0)
        bye("Program bug: Trying to remove fd from list with no fds."); // 如果列表中没有fd，则输出错误信息并退出程序

    /* find the fd in the list */
    // 在列表中查找fd
    for (x = 0; x < last; x++) {
        struct fdinfo *fdi = &fdl->fds[x]; // 获取fdl中fds数组的第x个元素的地址
        if (fdi->fd == fd) { // 如果找到了要移除的fd
            found = x; // 标记找到的位置
            /* If it's not the max, we can bail early. */
            // 如果不是最大的fd，我们可以提前退出
            if (fd < fdl->fdmax) {
                newfdmax = fdl->fdmax; // 更新新的最大fd
                break; // 退出循环
            }
        }
        else if (fdi->fd > newfdmax) // 如果当前fd大于新的最大fd
            newfdmax = fdi->fd; // 更新新的最大fd
    }
    fdl->fdmax = newfdmax; // 更新fdl中的fdmax成员为新的最大fd

    /* make sure we found it */
    // 确保我们找到了要移除的fd
    if (found < 0)
        bye("Program bug: fd (%d) not on list.", fd); // 如果没有找到要移除的fd，则输出错误信息并退出程序

    /* remove it, does nothing if (last == 1) */
    // 移除fd，如果last等于1则不执行任何操作
    if (o.debug > 1)
        logdebug("Swapping fd[%d] (%d) with fd[%d] (%d)\n",
                 found, fdl->fds[found].fd, last - 1, fdl->fds[last - 1].fd); // 输出调试信息
    fdl->fds[found] = fdl->fds[last - 1]; // 将最后一个fd替换到要移除的位置
    fdl->state++; // 更新fdl的状态

    fdl->nfds--; // 更新fdl中的nfds成员

    if (o.debug > 1)
        logdebug("Removed fd %d from list, nfds %d, maxfd %d\n", fd, fdl->nfds, fdl->fdmax); // 输出调试信息
    return 0; // 返回0表示移除成功
}

/* find the max descriptor in our list */
// 在列表中找到最大的描述符
int get_maxfd(fd_list_t *fdl)
{
    int x = 0, max = -1, nfds = fdl->nfds; // 初始化变量x为0，max为-1，nfds为fdl中的nfds成员

    for (x = 0; x < nfds; x++) // 遍历fdl中的fds数组
        if (fdl->fds[x].fd > max) // 如果当前fd大于max
            max = fdl->fds[x].fd; // 更新max为当前fd

    return max; // 返回最大的fd
}

struct fdinfo *get_fdinfo(const fd_list_t *fdl, int fd)
{
    int x;

    for (x = 0; x < fdl->nfds; x++) // 遍历fdl中的fds数组
        if (fdl->fds[x].fd == fd) // 如果找到了要查找的fd
            return &fdl->fds[x]; // 返回该fd的地址

    return NULL; // 如果没有找到，则返回NULL
}
// 初始化文件描述符列表
void init_fdlist(fd_list_t *fdl, int maxfds)
{
    // 为文件描述符列表分配内存空间
    fdl->fds = (struct fdinfo *) Calloc(maxfds, sizeof(struct fdinfo));
    // 初始化文件描述符数量为0
    fdl->nfds = 0;
    // 初始化最大文件描述符为-1
    fdl->fdmax = -1;
    // 设置最大文件描述符数量
    fdl->maxfds = maxfds;
    // 初始化状态为0
    fdl->state = 0;

    // 如果调试级别大于1，则记录初始化文件描述符列表的最大文件描述符数量
    if (o.debug > 1)
        logdebug("Initialized fdlist with %d maxfds\n", maxfds);
}

// 释放文件描述符列表
void free_fdlist(fd_list_t *fdl)
{
    // 释放文件描述符列表的内存空间
    free(fdl->fds);
    // 重置文件描述符数量为0
    fdl->nfds = 0;
    // 重置最大文件描述符为-1
    fdl->fdmax = -1;
    // 重置状态为0
    fdl->state = 0;
}

/*  如果需要对EOL序列进行更改以符合--crlf，则dst将被填充为修改后的src，len将相应调整，返回值将为非零。
 *  state用于跟踪跨多次调用此函数的行尾。在第一次调用时，state应该是一个包含0的int指针。之后，保持传递相同的指针。不同的逻辑流应使用不同的状态指针。
 *  如果未进行更改，则返回0 - len和dst将保持不变。
 */
int fix_line_endings(char *src, int *len, char **dst, int *state)
{
    int fix_count;
    int i, j;
    int num_bytes = *len;
    int prev_state = *state;

    /* *state为真当且仅当前一个块的最后一个字节为\r。 */
    if (num_bytes > 0)
        *state = (src[num_bytes - 1] == '\r');

    // 获取不匹配\r的\n的数量
    fix_count = 0;
    for (i = 0; i < num_bytes; i++) {
        if (src[i] == '\n' && ((i == 0) ? !prev_state : src[i - 1] != '\r'))
            fix_count++;
    }
    if (fix_count <= 0)
        return 0;

    // 现在插入匹配的\r
    *dst = (char *) safe_malloc(num_bytes + fix_count);
    j = 0;

    for (i = 0; i < num_bytes; i++) {
        if (src[i] == '\n' && ((i == 0) ? !prev_state : src[i - 1] != '\r')) {
            memcpy(*dst + j, "\r\n", 2);
            j += 2;
        } else {
            memcpy(*dst + j, src + i, 1);
            j++;
        }
    }
    *len += fix_count;

    return 1;
}
/*
 * next_protos_parse函数将逗号分隔的字符串解析成一个适合传递给SSL_CTX_set_next_protos_advertised的格式的字符串
 *   outlen: (输出) 成功时设置为结果缓冲区的长度
 *   err: 失败时为NULL
 *   in: 类似于"abc,def,ghi"的以NULL结尾的字符串
 *
 *   返回: malloc分配的缓冲区，失败时为NULL
 */
unsigned char *next_protos_parse(size_t *outlen, const char *in)
{
    size_t len;
    unsigned char *out;
    size_t i, start = 0;

    len = strlen(in);
    if (len >= 65535)
        return NULL;

    out = (unsigned char *)safe_malloc(strlen(in) + 1);
    for (i = 0; i <= len; ++i) {
        if (i == len || in[i] == ',') {
            if (i - start > 255) {
                free(out);
                return NULL;
            }
            out[start] = i - start;
            start = i + 1;
        } else
            out[i + 1] = in[i];
    }

    *outlen = len + 1;
    return out;
}
```