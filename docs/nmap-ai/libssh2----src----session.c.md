# `nmap\libssh2\src\session.c`

```cpp
/*
 * 版权声明和许可声明
 * 作者：Sara Golemon <sarag@libssh2.org>，Daniel Stenberg，Simon Josefsson <simon@josefsson.org>
 * 版权所有
 *
 * 在源代码和二进制形式下的再发布和使用，无论是否经过修改，都是允许的，只要满足以下条件：
 *   1. 源代码的再发布必须保留上述版权声明、条件列表和以下免责声明。
 *   2. 二进制形式的再发布必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明。
 *   3. 未经特定事先书面许可，不得使用版权所有者的名称或其他贡献者的名称来认可或推广从本软件派生的产品。
 *
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途适用性的暗示担保。在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）责任，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责，即使已被告知可能发生此类损害的可能性。
 */

#include "libssh2_priv.h"  // 包含 libssh2 私有头文件
#include <errno.h>  // 包含错误码头文件
#ifdef HAVE_UNISTD_H
#include <unistd.h>  // 如果有 unistd.h 头文件，则包含
#endif
#include <stdlib.h>  // 包含标准库头文件
#include <fcntl.h>  // 包含文件控制头文件

#ifdef HAVE_GETTIMEOFDAY
#include <sys/time.h>  // 如果有 gettimeofday 函数，则包含系统时间头文件
#endif
#ifdef HAVE_ALLOCA_H
#include <alloca.h>  // 如果有 alloca 函数，则包含动态分配内存头文件
#endif

#include "transport.h"  // 包含传输层头文件
 */ 
/* libssh2_default_alloc
 */
static
LIBSSH2_ALLOC_FUNC(libssh2_default_alloc)
{
    // 忽略抽象参数，分配指定大小的内存块
    (void) abstract;
    return malloc(count);
}

/* libssh2_default_free
 */
static
LIBSSH2_FREE_FUNC(libssh2_default_free)
{
    // 忽略抽象参数，释放指定的内存块
    (void) abstract;
    free(ptr);
}

/* libssh2_default_realloc
 */
static
LIBSSH2_REALLOC_FUNC(libssh2_default_realloc)
{
    // 忽略抽象参数，重新分配指定大小的内存块
    (void) abstract;
    return realloc(ptr, count);
}

/*
 * banner_receive
 *
 * Wait for a hello from the remote host
 * Allocate a buffer and store the banner in session->remote.banner
 * Returns: 0 on success, LIBSSH2_ERROR_EAGAIN if read would block, negative
 * on failure
 */
static int
banner_receive(LIBSSH2_SESSION * session)
{
    int ret;
    int banner_len;

    // 如果会话的状态为闲置，则初始化横幅长度和状态
    if(session->banner_TxRx_state == libssh2_NB_state_idle) {
        banner_len = 0;
        session->banner_TxRx_state = libssh2_NB_state_created;
    }
    // 否则，使用已发送的横幅长度
    else {
        banner_len = session->banner_TxRx_total_send;
    }
}
    while((banner_len < (int) sizeof(session->banner_TxRx_banner)) &&
           ((banner_len == 0)
            || (session->banner_TxRx_banner[banner_len - 1] != '\n'))) {
        char c = '\0';

        /* 没有接收到数据块！ */
        session->socket_block_directions &= ~LIBSSH2_SESSION_BLOCK_INBOUND;

        // 接收数据
        ret = LIBSSH2_RECV(session, &c, 1,
                            LIBSSH2_SOCKET_RECV_FLAGS(session));
        if(ret < 0) {
            if(session->api_block_mode || (ret != -EAGAIN))
                /* 在非阻塞模式下忽略 EAGAIN */
                _libssh2_debug(session, LIBSSH2_TRACE_SOCKET,
                               "Error recving %d bytes: %d", 1, -ret);
        }
        else
            _libssh2_debug(session, LIBSSH2_TRACE_SOCKET,
                           "Recved %d bytes banner", ret);

        if(ret < 0) {
            if(ret == -EAGAIN) {
                session->socket_block_directions =
                    LIBSSH2_SESSION_BLOCK_INBOUND;
                session->banner_TxRx_total_send = banner_len;
                return LIBSSH2_ERROR_EAGAIN;
            }

            /* 出现某种错误 */
            session->banner_TxRx_state = libssh2_NB_state_idle;
            session->banner_TxRx_total_send = 0;
            return LIBSSH2_ERROR_SOCKET_RECV;
        }

        if(ret == 0) {
            session->socket_state = LIBSSH2_SOCKET_DISCONNECTED;
            return LIBSSH2_ERROR_SOCKET_DISCONNECT;
        }

        if(c == '\0') {
            /* SSH 横幅中不允许出现 NULL 字符 */
            session->banner_TxRx_state = libssh2_NB_state_idle;
            session->banner_TxRx_total_send = 0;
            return LIBSSH2_ERROR_BANNER_RECV;
        }

        session->banner_TxRx_banner[banner_len++] = c;
    }

    // 去除横幅末尾的换行符和回车符
    while(banner_len &&
           ((session->banner_TxRx_banner[banner_len - 1] == '\n') ||
            (session->banner_TxRx_banner[banner_len - 1] == '\r'))) {
        banner_len--;
    }
    /* 从这里开始，我们已经完成了对远程服务器的握手 */
    session->banner_TxRx_state = libssh2_NB_state_idle;
    session->banner_TxRx_total_send = 0;

    // 如果没有接收到远程服务器的欢迎信息，则返回错误
    if(!banner_len)
        return LIBSSH2_ERROR_BANNER_RECV;

    // 如果会话中已经存在远程服务器的欢迎信息，则释放内存
    if(session->remote.banner)
        LIBSSH2_FREE(session, session->remote.banner);

    // 为远程服务器的欢迎信息分配内存空间
    session->remote.banner = LIBSSH2_ALLOC(session, banner_len + 1);
    // 如果内存分配失败，则返回错误
    if(!session->remote.banner) {
        return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                              "Error allocating space for remote banner");
    }
    // 将接收到的远程服务器的欢迎信息复制到会话中
    memcpy(session->remote.banner, session->banner_TxRx_banner, banner_len);
    session->remote.banner[banner_len] = '\0';
    // 打印接收到的远程服务器的欢迎信息
    _libssh2_debug(session, LIBSSH2_TRACE_TRANS, "Received Banner: %s",
                   session->remote.banner);
    // 返回无错误
    return LIBSSH2_ERROR_NONE;
# 发送默认横幅，或者通过 libssh2_setopt_string 设置的横幅
# 如果会阻塞，则返回 LIBSSH2_ERROR_EAGAIN，如果是这样，应该在可能发送更多数据时再次调用此函数
# 并且应该使用相同的参数调用此函数，直到返回零或失败为止
static int
banner_send(LIBSSH2_SESSION * session)
{
    char *banner = (char *) LIBSSH2_SSH_DEFAULT_BANNER_WITH_CRLF;  # 设置默认横幅
    int banner_len = sizeof(LIBSSH2_SSH_DEFAULT_BANNER_WITH_CRLF) - 1;  # 计算默认横幅的长度
    ssize_t ret;  # 定义返回值
#ifdef LIBSSH2DEBUG
    char banner_dup[256];  # 定义用于调试输出的横幅副本
#endif

    if(session->banner_TxRx_state == libssh2_NB_state_idle) {  # 如果横幅发送接收状态为闲置
        if(session->local.banner) {  # 如果本地横幅存在
            # setopt_string 将给出我们的 \r\n 字符
            banner_len = strlen((char *) session->local.banner);  # 计算本地横幅的长度
            banner = (char *) session->local.banner;  # 设置横幅为本地横幅
        }
#ifdef LIBSSH2DEBUG
        # 为避免在调试输出中发送 CRLF 而进行的修改
        if(banner_len < 256) {  # 如果横幅长度小于 256
            memcpy(banner_dup, banner, banner_len - 2);  # 复制横幅内容到横幅副本
            banner_dup[banner_len - 2] = '\0';  # 在横幅副本末尾添加结束符
        }
        else {
            memcpy(banner_dup, banner, 255);  # 复制横幅内容的前 255 个字符到横幅副本
            banner_dup[255] = '\0';  # 在横幅副本末尾添加结束符
        }

        _libssh2_debug(session, LIBSSH2_TRACE_TRANS, "Sending Banner: %s",  # 调试输出发送的横幅内容
                       banner_dup);
#endif

        session->banner_TxRx_state = libssh2_NB_state_created;  # 设置横幅发送接收状态为已创建
    }

    # 没有传出的阻塞！
    session->socket_block_directions &= ~LIBSSH2_SESSION_BLOCK_OUTBOUND;  # 清除传出阻塞标志位

    ret = LIBSSH2_SEND(session,  # 发送横幅数据
                        banner + session->banner_TxRx_total_send,  # 从当前发送位置开始发送横幅数据
                        banner_len - session->banner_TxRx_total_send,  # 计算剩余需要发送的横幅数据长度
                        LIBSSH2_SOCKET_SEND_FLAGS(session));  # 获取发送标志
    # 如果发送的字节数小于 0，则记录错误信息
    if(ret < 0)
        _libssh2_debug(session, LIBSSH2_TRACE_SOCKET,
                       "Error sending %d bytes: %d",
                       banner_len - session->banner_TxRx_total_send, -ret);
    # 如果发送的字节数大于等于 0，则记录发送成功的信息
    else
        _libssh2_debug(session, LIBSSH2_TRACE_SOCKET,
                       "Sent %d/%d bytes at %p+%d", ret,
                       banner_len - session->banner_TxRx_total_send,
                       banner, session->banner_TxRx_total_send);

    # 如果发送的字节数不等于待发送的字节数
    if(ret != (banner_len - session->banner_TxRx_total_send)) {
        # 如果发送的字节数大于等于 0 或者等于 -EAGAIN，则保存已发送的部分并返回 EAGAIN 错误
        if(ret >= 0 || ret == -EAGAIN) {
            session->socket_block_directions =
                LIBSSH2_SESSION_BLOCK_OUTBOUND;
            if(ret > 0)
                session->banner_TxRx_total_send += ret;
            return LIBSSH2_ERROR_EAGAIN;
        }
        # 如果发送的字节数小于 0 且不是 EAGAIN，则重置状态并返回 SOCKET_RECV 错误
        session->banner_TxRx_state = libssh2_NB_state_idle;
        session->banner_TxRx_total_send = 0;
        return LIBSSH2_ERROR_SOCKET_RECV;
    }

    # 将状态设置回空闲
    session->banner_TxRx_state = libssh2_NB_state_idle;
    session->banner_TxRx_total_send = 0;

    # 返回成功
    return 0;
/*
 * session_nonblock() sets the given socket to either blocking or
 * non-blocking mode based on the 'nonblock' boolean argument. This function
 * is copied from the libcurl sources with permission.
 */
static int
session_nonblock(libssh2_socket_t sockfd,   /* operate on this */
                 int nonblock /* TRUE or FALSE */ )
{
#undef SETBLOCK
#define SETBLOCK 0
#ifdef HAVE_O_NONBLOCK
    /* most recent unix versions */
    int flags;

    // 获取当前文件描述符的状态标志
    flags = fcntl(sockfd, F_GETFL, 0);
    // 如果需要设置为非阻塞模式，则添加 O_NONBLOCK 标志
    if(nonblock)
        return fcntl(sockfd, F_SETFL, flags | O_NONBLOCK);
    // 如果需要设置为阻塞模式，则移除 O_NONBLOCK 标志
    else
        return fcntl(sockfd, F_SETFL, flags & (~O_NONBLOCK));
#undef SETBLOCK
#define SETBLOCK 1
#endif

#if defined(HAVE_FIONBIO) && (SETBLOCK == 0)
    /* older unix versions and VMS*/
    int flags;

    // 根据 nonblock 参数设置文件描述符的状态
    flags = nonblock;
    return ioctl(sockfd, FIONBIO, &flags);
#undef SETBLOCK
#define SETBLOCK 2
#endif

#if defined(HAVE_IOCTLSOCKET) && (SETBLOCK == 0)
    /* Windows? */
    unsigned long flags;
    flags = nonblock;

    // 根据 nonblock 参数设置套接字的状态
    return ioctlsocket(sockfd, FIONBIO, &flags);
#undef SETBLOCK
#define SETBLOCK 3
#endif

#if defined(HAVE_IOCTLSOCKET_CASE) && (SETBLOCK == 0)
    /* presumably for Amiga */
    // 根据 nonblock 参数设置套接字的状态
    return IoctlSocket(sockfd, FIONBIO, (long) nonblock);
#undef SETBLOCK
#define SETBLOCK 4
#endif

#if defined(HAVE_SO_NONBLOCK) && (SETBLOCK == 0)
    /* BeOS */
    long b = nonblock ? 1 : 0;
    // 根据 nonblock 参数设置套接字的状态
    return setsockopt(sockfd, SOL_SOCKET, SO_NONBLOCK, &b, sizeof(b));
#undef SETBLOCK
#define SETBLOCK 5
#endif

#ifdef HAVE_DISABLED_NONBLOCKING
    // 如果禁用了非阻塞模式，则直接返回成功
    return 0;                   /* returns success */
#undef SETBLOCK
#define SETBLOCK 6
#endif

#if(SETBLOCK == 0)
#error "no non-blocking method was found/used/set"
#endif
}

/*
 * get_socket_nonblocking()
 *
 * gets the given blocking or non-blocking state of the socket.
 */
static int
get_socket_nonblocking(int sockfd)
{                               /* operate on this */
#undef GETBLOCK
#define GETBLOCK 0
#ifdef HAVE_O_NONBLOCK
    /* most recent unix versions */
    # 获取套接字的当前状态标志
    int flags = fcntl(sockfd, F_GETFL, 0);

    # 如果获取失败，假设出现错误时是阻塞状态，返回1
    if(flags == -1) {
        /* Assume blocking on error */
        return 1;
    }
    # 返回套接字是否为非阻塞状态
    return (flags & O_NONBLOCK);
#if defined(WSAEWOULDBLOCK) && (GETBLOCK == 0)
    /* 如果定义了 WSAEWOULDBLOCK 并且 GETBLOCK 的值为 0，则执行以下代码块 */
    unsigned int option_value;  // 声明一个无符号整数变量 option_value
    socklen_t option_len = sizeof(option_value);  // 声明一个 socklen_t 类型的变量 option_len，并初始化为 option_value 的大小

    if(getsockopt
        (sockfd, SOL_SOCKET, SO_ERROR, (void *) &option_value, &option_len)) {
        /* 如果 getsockopt 函数调用失败，则假定为阻塞状态，并返回 1 */
        return 1;
    }
    /* 返回 option_value 的整数值 */
    return (int) option_value;
#undef GETBLOCK
#define GETBLOCK 2
#endif

#if defined(HAVE_SO_NONBLOCK) && (GETBLOCK == 0)
    /* 如果定义了 HAVE_SO_NONBLOCK 并且 GETBLOCK 的值为 0，则执行以下代码块 */
    long b;  // 声明一个长整型变量 b
    if(getsockopt(sockfd, SOL_SOCKET, SO_NONBLOCK, &b, sizeof(b))) {
        /* 如果 getsockopt 函数调用失败，则假定为阻塞状态，并返回 1 */
        return 1;
    }
    /* 返回 b 的整数值 */
    return (int) b;
#undef GETBLOCK
#define GETBLOCK 5
#endif

#if defined(SO_STATE) && defined(__VMS) && (GETBLOCK == 0)
    /* 如果定义了 SO_STATE 和 __VMS 并且 GETBLOCK 的值为 0，则执行以下代码块 */
    size_t sockstat = 0;  // 声明一个大小为 size_t 的变量 sockstat，并初始化为 0
    int    callstat = 0;  // 声明一个整型变量 callstat，并初始化为 0
    size_t size = sizeof(int);  // 声明一个大小为 size_t 的变量 size，并初始化为 int 类型的大小

    callstat = getsockopt(sockfd, SOL_SOCKET, SO_STATE, (char *)&sockstat, &size);  // 调用 getsockopt 函数，获取 sockfd 的 SO_STATE 状态，并将结果存入 sockstat 中
    if(callstat == -1) return 0;  // 如果调用失败，则返回 0
    if((sockstat&SS_NBIO) != 0) return 1;  // 如果 sockstat 的 SS_NBIO 位不为 0，则返回 1
    return 0;  // 否则返回 0
#undef GETBLOCK
#define GETBLOCK 6
#endif

#ifdef HAVE_DISABLED_NONBLOCKING
    return 1;  // 返回 1，表示阻塞状态
#undef GETBLOCK
#define GETBLOCK 7
#endif

#if(GETBLOCK == 0)
#error "no non-blocking method was found/used/get"
#endif
}

/* libssh2_session_banner_set
 * Set the local banner to use in the server handshake.
 */
LIBSSH2_API int
libssh2_session_banner_set(LIBSSH2_SESSION * session, const char *banner)
{
    size_t banner_len = banner ? strlen(banner) : 0;  // 如果 banner 存在，则获取其长度，否则长度为 0

    if(session->local.banner) {
        LIBSSH2_FREE(session, session->local.banner);  // 释放 session 的 local.banner
        session->local.banner = NULL;  // 将 session 的 local.banner 置为 NULL
    }

    if(!banner_len)
        return 0;  // 如果 banner 长度为 0，则返回 0

    session->local.banner = LIBSSH2_ALLOC(session, banner_len + 3);  // 为 session 的 local.banner 分配内存，大小为 banner_len + 3
    if(!session->local.banner) {
        return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                              "Unable to allocate memory for local banner");  // 如果分配内存失败，则返回错误信息
    }
    // 将 banner 的内容复制到 session->local.banner 中
    memcpy(session->local.banner, banner, banner_len);

    // 在调试输出中，设置本地横幅的格式
    // 将本地横幅设置为以 '\0' 结尾的字符串
    session->local.banner[banner_len] = '\0';
    _libssh2_debug(session, LIBSSH2_TRACE_TRANS, "Setting local Banner: %s",
                   session->local.banner);
    // 在本地横幅的末尾添加回车和换行符
    session->local.banner[banner_len++] = '\r';
    session->local.banner[banner_len++] = '\n';
    session->local.banner[banner_len] = '\0';

    // 返回 0，表示成功
    return 0;
/* libssh2_banner_set
 * 设置本地标语。已弃用版本
 */
LIBSSH2_API int
libssh2_banner_set(LIBSSH2_SESSION * session, const char *banner)
{
    // 调用 libssh2_session_banner_set 函数设置会话的标语
    return libssh2_session_banner_set(session, banner);
}

/*
 * libssh2_session_init_ex
 *
 * 分配并初始化一个 libssh2 会话结构。允许在调用程序具有自己的内存管理器的情况下使用 malloc 回调。允许定义一些但不是所有的 malloc 回调。还可以可选地传递一个指针值以发送到回调函数（这样它们就知道是谁在请求）
 */
LIBSSH2_API LIBSSH2_SESSION *
libssh2_session_init_ex(LIBSSH2_ALLOC_FUNC((*my_alloc)),
                        LIBSSH2_FREE_FUNC((*my_free)),
                        LIBSSH2_REALLOC_FUNC((*my_realloc)), void *abstract)
{
    LIBSSH2_ALLOC_FUNC((*local_alloc)) = libssh2_default_alloc;
    LIBSSH2_FREE_FUNC((*local_free)) = libssh2_default_free;
    LIBSSH2_REALLOC_FUNC((*local_realloc)) = libssh2_default_realloc;
    LIBSSH2_SESSION *session;

    if(my_alloc) {
        local_alloc = my_alloc;
    }
    if(my_free) {
        local_free = my_free;
    }
    if(my_realloc) {
        local_realloc = my_realloc;
    }

    // 分配会话结构的内存空间
    session = local_alloc(sizeof(LIBSSH2_SESSION), &abstract);
    if(session) {
        // 将会话结构的内存空间清零
        memset(session, 0, sizeof(LIBSSH2_SESSION));
        session->alloc = local_alloc;
        session->free = local_free;
        session->realloc = local_realloc;
        session->send = _libssh2_send;
        session->recv = _libssh2_recv;
        session->abstract = abstract;
        session->api_timeout = 0; /* 默认情况下无超时的 API */
        session->api_block_mode = 1; /* 默认情况下为阻塞 API */
        _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
                       "分配新的会话资源");
        _libssh2_init_if_needed();
    }
    return session;
}
/*
 * libssh2_session_callback_set
 *
 * 设置（或重置）回调函数
 * 返回先前的地址
 *
 * 警告：此函数依赖于我们可以将函数指针强制转换为void指针，这在ISO C中是不允许的！
 */
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wpedantic"
LIBSSH2_API void *
libssh2_session_callback_set(LIBSSH2_SESSION * session,
                             int cbtype, void *callback)
{
    void *oldcb;

    switch(cbtype) {
    case LIBSSH2_CALLBACK_IGNORE:
        oldcb = session->ssh_msg_ignore;
        session->ssh_msg_ignore = callback;
        return oldcb;

    case LIBSSH2_CALLBACK_DEBUG:
        oldcb = session->ssh_msg_debug;
        session->ssh_msg_debug = callback;
        return oldcb;

    case LIBSSH2_CALLBACK_DISCONNECT:
        oldcb = session->ssh_msg_disconnect;
        session->ssh_msg_disconnect = callback;
        return oldcb;

    case LIBSSH2_CALLBACK_MACERROR:
        oldcb = session->macerror;
        session->macerror = callback;
        return oldcb;

    case LIBSSH2_CALLBACK_X11:
        oldcb = session->x11;
        session->x11 = callback;
        return oldcb;

    case LIBSSH2_CALLBACK_SEND:
        oldcb = session->send;
        session->send = callback;
        return oldcb;

    case LIBSSH2_CALLBACK_RECV:
        oldcb = session->recv;
        session->recv = callback;
        return oldcb;
    }
    _libssh2_debug(session, LIBSSH2_TRACE_TRANS, "Setting Callback %d",
                   cbtype);

    return NULL;
}
#pragma GCC diagnostic pop

/*
 * _libssh2_wait_socket()
 *
 * 等待套接字上的操作的实用函数。当准备好再次运行时返回0，或在超时时返回错误。
 */
int _libssh2_wait_socket(LIBSSH2_SESSION *session, time_t start_time)
{
    int rc;
    int seconds_to_next;
    int dir;
    int has_timeout;
    long ms_to_next = 0;
    long elapsed_ms;
    /* 由于 libssh2 在调用此函数之前通常会在内部设置 EAGAIN，我们可以通过在此函数中重置错误代码来减少一些混淆，从而降低将 EAGAIN 存储为错误代码的风险 */
    session->err_code = LIBSSH2_ERROR_NONE;

    // 发送保持活动消息，获取下一次发送消息的时间间隔
    rc = libssh2_keepalive_send(session, &seconds_to_next);
    if(rc)
        return rc;

    // 将秒转换为毫秒
    ms_to_next = seconds_to_next * 1000;

    // 确定等待的方向
    dir = libssh2_session_block_directions(session);

    // 如果没有等待的方向，则设置超时为 1 秒，以避免无限循环
    if(!dir) {
        _libssh2_debug(session, LIBSSH2_TRACE_SOCKET,
                       "Nothing to wait for in wait_socket");
        ms_to_next = 1000;
    }

    // 如果设置了 API 超时，并且下一次发送消息的时间为 0 或者毫秒数大于 API 超时时间，则返回 API 超时错误
    if(session->api_timeout > 0 &&
        (seconds_to_next == 0 ||
         ms_to_next > session->api_timeout)) {
        time_t now = time(NULL);
        elapsed_ms = (long)(1000*difftime(now, start_time));
        if(elapsed_ms > session->api_timeout) {
            return _libssh2_error(session, LIBSSH2_ERROR_TIMEOUT,
                                  "API timeout expired");
        }
        ms_to_next = (session->api_timeout - elapsed_ms);
        has_timeout = 1;
    }
    // 如果下一次发送消息的时间大于 0，则设置超时标志为 1
    else if(ms_to_next > 0) {
        has_timeout = 1;
    }
    // 否则，超时标志为 0
    else
        has_timeout = 0;
#ifdef HAVE_POLL
    {
        // 如果系统支持 poll 函数，则使用 pollfd 结构体数组
        struct pollfd sockets[1];

        // 设置第一个文件描述符为会话的套接字文件描述符
        sockets[0].fd = session->socket_fd;
        sockets[0].events = 0;
        sockets[0].revents = 0;

        // 如果是入站阻塞，则设置对应的事件为可读
        if(dir & LIBSSH2_SESSION_BLOCK_INBOUND)
            sockets[0].events |= POLLIN;

        // 如果是出站阻塞，则设置对应的事件为可写
        if(dir & LIBSSH2_SESSION_BLOCK_OUTBOUND)
            sockets[0].events |= POLLOUT;

        // 使用 poll 函数进行等待，如果有超时则使用 ms_to_next，否则使用 -1
        rc = poll(sockets, 1, has_timeout?ms_to_next: -1);
    }
#else
    {
        // 如果系统不支持 poll 函数，则使用 fd_set 和 select 函数
        fd_set rfd;
        fd_set wfd;
        fd_set *writefd = NULL;
        fd_set *readfd = NULL;
        struct timeval tv;

        // 设置超时时间
        tv.tv_sec = ms_to_next / 1000;
        tv.tv_usec = (ms_to_next - tv.tv_sec*1000) * 1000;

        // 如果是入站阻塞，则设置对应的文件描述符和读集合
        if(dir & LIBSSH2_SESSION_BLOCK_INBOUND) {
            FD_ZERO(&rfd);
            FD_SET(session->socket_fd, &rfd);
            readfd = &rfd;
        }

        // 如果是出站阻塞，则设置对应的文件描述符和写集合
        if(dir & LIBSSH2_SESSION_BLOCK_OUTBOUND) {
            FD_ZERO(&wfd);
            FD_SET(session->socket_fd, &wfd);
            writefd = &wfd;
        }

        // 使用 select 函数进行等待，如果有超时则使用 tv，否则使用 NULL
        rc = select(session->socket_fd + 1, readfd, writefd, NULL,
                    has_timeout ? &tv : NULL);
    }
#endif
    // 如果返回值为 0，表示超时
    if(rc == 0) {
        return _libssh2_error(session, LIBSSH2_ERROR_TIMEOUT,
                              "Timed out waiting on socket");
    }
    // 如果返回值小于 0，表示出现错误
    if(rc < 0) {
        return _libssh2_error(session, LIBSSH2_ERROR_TIMEOUT,
                              "Error waiting on socket");
    }

    // 返回 0，表示准备再次尝试
    return 0; /* ready to try again */
}

static int
session_startup(LIBSSH2_SESSION *session, libssh2_socket_t sock)
{
    int rc;
    # 如果会话的启动状态为闲置
    if(session->startup_state == libssh2_NB_state_idle) {
        # 输出调试信息，显示会话的启动状态和套接字的编号
        _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
                       "session_startup for socket %d", sock);
        # 如果套接字编号无效
        if(LIBSSH2_INVALID_SOCKET == sock) {
            # 返回错误信息，表示提供了无效的套接字
            return _libssh2_error(session, LIBSSH2_ERROR_BAD_SOCKET,
                                  "Bad socket provided");
        }
        # 将套接字编号赋值给会话对象的套接字文件描述符
        session->socket_fd = sock;

        # 保存套接字的原始阻塞状态
        session->socket_prev_blockstate =
            !get_socket_nonblocking(session->socket_fd);

        # 如果套接字原来是阻塞状态，则改为非阻塞状态
        if(session->socket_prev_blockstate) {
            rc = session_nonblock(session->socket_fd, 1);
            if(rc) {
                # 返回错误信息，表示无法将套接字的阻塞状态改为非阻塞
                return _libssh2_error(session, rc,
                                      "Failed changing socket's "
                                      "blocking state to non-blocking");
            }
        }

        # 将会话的启动状态设置为已创建
        session->startup_state = libssh2_NB_state_created;
    }

    # 如果会话的启动状态为已创建
    if(session->startup_state == libssh2_NB_state_created) {
        # 发送协议版本信息，并返回结果
        rc = banner_send(session);
        if(rc == LIBSSH2_ERROR_EAGAIN)
            return rc;
        else if(rc) {
            # 返回错误信息，表示发送协议版本信息失败
            return _libssh2_error(session, rc,
                                  "Failed sending banner");
        }
        # 将会话的启动状态设置为已发送
        session->startup_state = libssh2_NB_state_sent;
        # 将会话的协议版本信息收发状态设置为闲置
        session->banner_TxRx_state = libssh2_NB_state_idle;
    }

    # 如果会话的启动状态为已发送
    if(session->startup_state == libssh2_NB_state_sent) {
        # 循环接收协议版本信息
        do {
            rc = banner_receive(session);
            if(rc == LIBSSH2_ERROR_EAGAIN)
                return rc;
            else if(rc)
                return _libssh2_error(session, rc,
                                      "Failed getting banner");
        } while(strncmp("SSH-", (char *)session->remote.banner, 4));

        # 将会话的启动状态设置为已发送1
        session->startup_state = libssh2_NB_state_sent1;
    }
    # 如果会话的启动状态为发送状态1
    if(session->startup_state == libssh2_NB_state_sent1) {
        # 调用_libssh2_kex_exchange函数进行密钥交换，并更新启动密钥状态
        rc = _libssh2_kex_exchange(session, 0, &session->startup_key_state);
        # 如果返回值为LIBSSH2_ERROR_EAGAIN，则表示需要再次调用该函数
        if(rc == LIBSSH2_ERROR_EAGAIN)
            return rc;
        # 如果返回值不为0，则返回错误信息
        else if(rc)
            return _libssh2_error(session, rc,
                                  "Unable to exchange encryption keys");

        # 更新启动状态为发送状态2
        session->startup_state = libssh2_NB_state_sent2;
    }

    # 如果会话的启动状态为发送状态2
    if(session->startup_state == libssh2_NB_state_sent2) {
        # 输出调试信息，请求用户认证服务
        _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
                       "Requesting userauth service");

        # 请求用户认证服务
        session->startup_service[0] = SSH_MSG_SERVICE_REQUEST;
        _libssh2_htonu32(session->startup_service + 1,
                         sizeof("ssh-userauth") - 1);
        memcpy(session->startup_service + 5, "ssh-userauth",
               sizeof("ssh-userauth") - 1);

        # 更新启动状态为发送状态3
        session->startup_state = libssh2_NB_state_sent3;
    }

    # 如果会话的启动状态为发送状态3
    if(session->startup_state == libssh2_NB_state_sent3) {
        # 发送请求用户认证服务的消息
        rc = _libssh2_transport_send(session, session->startup_service,
                                     sizeof("ssh-userauth") + 5 - 1,
                                     NULL, 0);
        # 如果返回值为LIBSSH2_ERROR_EAGAIN，则表示需要再次发送消息
        if(rc == LIBSSH2_ERROR_EAGAIN)
            return rc;
        # 如果返回值不为0，则返回错误信息
        else if(rc) {
            return _libssh2_error(session, rc,
                                  "Unable to ask for ssh-userauth service");
        }

        # 更新启动状态为发送状态4
        session->startup_state = libssh2_NB_state_sent4;
    }
    # 如果会话的启动状态为libssh2_NB_state_sent4
    if(session->startup_state == libssh2_NB_state_sent4) {
        # 要求接收SSH_MSG_SERVICE_ACCEPT消息
        rc = _libssh2_packet_require(session, SSH_MSG_SERVICE_ACCEPT,
                                     &session->startup_data,
                                     &session->startup_data_len, 0, NULL, 0,
                                     &session->startup_req_state);
        # 如果出现错误，返回错误码
        if(rc)
            return rc;

        # 如果启动数据长度小于5，返回协议错误
        if(session->startup_data_len < 5) {
            return _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                  "Unexpected packet length");
        }

        # 获取服务长度
        session->startup_service_length =
            _libssh2_ntohu32(session->startup_data + 1);

        # 如果服务长度不等于"ssh-userauth"的长度，或者服务名称不是"ssh-userauth"，返回协议错误
        if((session->startup_service_length != (sizeof("ssh-userauth") - 1))
            || strncmp("ssh-userauth", (char *) session->startup_data + 5,
                       session->startup_service_length)) {
            LIBSSH2_FREE(session, session->startup_data);
            session->startup_data = NULL;
            return _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                  "Invalid response received from server");
        }
        # 释放启动数据内存
        LIBSSH2_FREE(session, session->startup_data);
        session->startup_data = NULL;

        # 设置启动状态为libssh2_NB_state_idle
        session->startup_state = libssh2_NB_state_idle;

        # 返回成功
        return 0;
    }

    # 如果不是libssh2_NB_state_sent4状态，返回无效参数错误
    return LIBSSH2_ERROR_INVAL;
}

/*
 * libssh2_session_handshake()
 *
 * session: LIBSSH2_SESSION struct allocated and owned by the calling program
 * sock:    *must* be populated with an opened and connected socket.
 *
 * Returns: 0 on success, or non-zero on failure
 */
LIBSSH2_API int
libssh2_session_handshake(LIBSSH2_SESSION *session, libssh2_socket_t sock)
{
    int rc;

    BLOCK_ADJUST(rc, session, session_startup(session, sock) );

    return rc;
}

/*
 * libssh2_session_startup()
 *
 * DEPRECATED. Use libssh2_session_handshake() instead! This function is not
 * portable enough.
 *
 * session: LIBSSH2_SESSION struct allocated and owned by the calling program
 * sock:    *must* be populated with an opened and connected socket.
 *
 * Returns: 0 on success, or non-zero on failure
 */
LIBSSH2_API int
libssh2_session_startup(LIBSSH2_SESSION *session, int sock)
{
    return libssh2_session_handshake(session, (libssh2_socket_t) sock);
}

/*
 * libssh2_session_free
 *
 * Frees the memory allocated to the session
 * Also closes and frees any channels attached to this session
 */
static int
session_free(LIBSSH2_SESSION *session)
{
    int rc;
    LIBSSH2_PACKET *pkg;
    LIBSSH2_CHANNEL *ch;
    LIBSSH2_LISTENER *l;
    int packets_left = 0;

    if(session->free_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
                       "Freeing session resource",
                       session->remote.banner);

        session->free_state = libssh2_NB_state_created;
    }

    if(session->free_state == libssh2_NB_state_created) {
        while((ch = _libssh2_list_first(&session->channels))) {

            rc = _libssh2_channel_free(ch);
            if(rc == LIBSSH2_ERROR_EAGAIN)
                return rc;
        }

        session->free_state = libssh2_NB_state_sent;
    }
    # 如果会话的自由状态为已发送，则执行以下操作
    if(session->free_state == libssh2_NB_state_sent) {
        # 当会话的监听器列表不为空时，执行以下操作
        while((l = _libssh2_list_first(&session->listeners))) {
            # 取消通道转发
            rc = _libssh2_channel_forward_cancel(l);
            # 如果返回值为 LIBSSH2_ERROR_EAGAIN，则立即返回该值
            if(rc == LIBSSH2_ERROR_EAGAIN)
                return rc;
        }

        # 将会话的自由状态设置为已发送1
        session->free_state = libssh2_NB_state_sent1;
    }

    # 如果会话的状态包含 LIBSSH2_STATE_NEWKEYS，则执行以下操作
    if(session->state & LIBSSH2_STATE_NEWKEYS) {
        /* hostkey */
        # 如果会话的主机密钥存在且具有析构函数，则执行析构函数
        if(session->hostkey && session->hostkey->dtor) {
            session->hostkey->dtor(session, &session->server_hostkey_abstract);
        }

        /* Client to Server */
        /* crypt */
        # 如果会话的本地加密存在且具有析构函数，则执行析构函数
        if(session->local.crypt && session->local.crypt->dtor) {
            session->local.crypt->dtor(session,
                                       &session->local.crypt_abstract);
        }
        /* comp */
        # 如果会话的本地压缩存在且具有析构函数，则执行析构函数
        if(session->local.comp && session->local.comp->dtor) {
            session->local.comp->dtor(session, 1,
                                      &session->local.comp_abstract);
        }
        /* mac */
        # 如果会话的本地消息认证码存在且具有析构函数，则执行析构函数
        if(session->local.mac && session->local.mac->dtor) {
            session->local.mac->dtor(session, &session->local.mac_abstract);
        }

        /* Server to Client */
        /* crypt */
        # 如果会话的远程加密存在且具有析构函数，则执行析构函数
        if(session->remote.crypt && session->remote.crypt->dtor) {
            session->remote.crypt->dtor(session,
                                        &session->remote.crypt_abstract);
        }
        /* comp */
        # 如果会话的远程压缩存在且具有析构函数，则执行析构函数
        if(session->remote.comp && session->remote.comp->dtor) {
            session->remote.comp->dtor(session, 0,
                                       &session->remote.comp_abstract);
        }
        /* mac */
        # 如果会话的远程消息认证码存在且具有析构函数，则执行析构函数
        if(session->remote.mac && session->remote.mac->dtor) {
            session->remote.mac->dtor(session, &session->remote.mac_abstract);
        }

        /* session_id */
        # 如果会话的会话ID存在，则释放其内存
        if(session->session_id) {
            LIBSSH2_FREE(session, session->session_id);
        }
    }

    /* Free banner(s) */
    # 释放横幅（banner）的内存
    // 如果会话的远程横幅存在，则释放远程横幅内存
    if(session->remote.banner) {
        LIBSSH2_FREE(session, session->remote.banner);
    }
    // 如果会话的本地横幅存在，则释放本地横幅内存
    if(session->local.banner) {
        LIBSSH2_FREE(session, session->local.banner);
    }

    /* 释放首选项 */
    // 如果会话的密钥交换首选项存在，则释放密钥交换首选项内存
    if(session->kex_prefs) {
        LIBSSH2_FREE(session, session->kex_prefs);
    }
    // 如果会话的主机密钥首选项存在，则释放主机密钥首选项内存
    if(session->hostkey_prefs) {
        LIBSSH2_FREE(session, session->hostkey_prefs);
    }

    // 如果会话的本地密钥交换初始化数据存在，则释放本地密钥交换初始化数据内存
    if(session->local.kexinit) {
        LIBSSH2_FREE(session, session->local.kexinit);
    }
    // 如果会话的本地加密首选项存在，则释放本地加密首选项内存
    if(session->local.crypt_prefs) {
        LIBSSH2_FREE(session, session->local.crypt_prefs);
    }
    // 如果会话的本地消息认证码首选项存在，则释放本地消息认证码首选项内存
    if(session->local.mac_prefs) {
        LIBSSH2_FREE(session, session->local.mac_prefs);
    }
    // 如果会话的本地压缩首选项存在，则释放本地压缩首选项内存
    if(session->local.comp_prefs) {
        LIBSSH2_FREE(session, session->local.comp_prefs);
    }
    // 如果会话的本地语言首选项存在，则释放本地语言首选项内存
    if(session->local.lang_prefs) {
        LIBSSH2_FREE(session, session->local.lang_prefs);
    }

    // 如果会话的远程密钥交换初始化数据存在，则释放远程密钥交换初始化数据内存
    if(session->remote.kexinit) {
        LIBSSH2_FREE(session, session->remote.kexinit);
    }
    // 如果会话的远程加密首选项存在，则释放远程加密首选项内存
    if(session->remote.crypt_prefs) {
        LIBSSH2_FREE(session, session->remote.crypt_prefs);
    }
    // 如果会话的远程消息认证码首选项存在，则释放远程消息认证码首选项内存
    if(session->remote.mac_prefs) {
        LIBSSH2_FREE(session, session->remote.mac_prefs);
    }
    // 如果会话的远程压缩首选项存在，则释放远程压缩首选项内存
    if(session->remote.comp_prefs) {
        LIBSSH2_FREE(session, session->remote.comp_prefs);
    }
    // 如果会话的远程语言首选项存在，则释放远程语言首选项内存
    if(session->remote.lang_prefs) {
        LIBSSH2_FREE(session, session->remote.lang_prefs);
    }

    /*
     * 确保释放状态变量中使用的所有内存
     */
    // 如果会话的密钥交换初始化数据存在，则释放密钥交换初始化数据内存
    if(session->kexinit_data) {
        LIBSSH2_FREE(session, session->kexinit_data);
    }
    // 如果会话的启动数据存在，则释放启动数据内存
    if(session->startup_data) {
        LIBSSH2_FREE(session, session->startup_data);
    }
    // 如果会话的用户认证列表数据存在，则释放用户认证列表数据内存
    if(session->userauth_list_data) {
        LIBSSH2_FREE(session, session->userauth_list_data);
    }
    // 如果会话的用户认证密码数据存在，则释放用户认证密码数据内存
    if(session->userauth_pswd_data) {
        LIBSSH2_FREE(session, session->userauth_pswd_data);
    }
    // 如果会话的用户认证新密码数据存在，则释放用户认证新密码数据内存
    if(session->userauth_pswd_newpw) {
        LIBSSH2_FREE(session, session->userauth_pswd_newpw);
    }
    # 如果会话中存在用户认证主机数据包，则释放该数据包
    if(session->userauth_host_packet) {
        LIBSSH2_FREE(session, session->userauth_host_packet);
    }
    # 如果会话中存在用户认证主机方法，则释放该方法
    if(session->userauth_host_method) {
        LIBSSH2_FREE(session, session->userauth_host_method);
    }
    # 如果会话中存在用户认证主机数据，则释放该数据
    if(session->userauth_host_data) {
        LIBSSH2_FREE(session, session->userauth_host_data);
    }
    # 如果会话中存在用户认证公钥数据，则释放该数据
    if(session->userauth_pblc_data) {
        LIBSSH2_FREE(session, session->userauth_pblc_data);
    }
    # 如果会话中存在用户认证公钥数据包，则释放该数据包
    if(session->userauth_pblc_packet) {
        LIBSSH2_FREE(session, session->userauth_pblc_packet);
    }
    # 如果会话中存在用户认证公钥方法，则释放该方法
    if(session->userauth_pblc_method) {
        LIBSSH2_FREE(session, session->userauth_pblc_method);
    }
    # 如果会话中存在用户认证键盘交互数据，则释放该数据
    if(session->userauth_kybd_data) {
        LIBSSH2_FREE(session, session->userauth_kybd_data);
    }
    # 如果会话中存在用户认证键盘交互数据包，则释放该数据包
    if(session->userauth_kybd_packet) {
        LIBSSH2_FREE(session, session->userauth_kybd_packet);
    }
    # 如果会话中存在用户认证键盘交互认证指令，则释放该指令
    if(session->userauth_kybd_auth_instruction) {
        LIBSSH2_FREE(session, session->userauth_kybd_auth_instruction);
    }
    # 如果会话中存在打开数据包，则释放该数据包
    if(session->open_packet) {
        LIBSSH2_FREE(session, session->open_packet);
    }
    # 如果会话中存在打开数据，则释放该数据
    if(session->open_data) {
        LIBSSH2_FREE(session, session->open_data);
    }
    # 如果会话中存在直接消息，则释放该消息
    if(session->direct_message) {
        LIBSSH2_FREE(session, session->direct_message);
    }
    # 如果会话中存在转发监听数据包，则释放该数据包
    if(session->fwdLstn_packet) {
        LIBSSH2_FREE(session, session->fwdLstn_packet);
    }
    # 如果会话中存在密钥初始化数据，则释放该数据
    if(session->pkeyInit_data) {
        LIBSSH2_FREE(session, session->pkeyInit_data);
    }
    # 如果会话中存在SCP接收命令，则释放该命令
    if(session->scpRecv_command) {
        LIBSSH2_FREE(session, session->scpRecv_command);
    }
    # 如果会话中存在SCP发送命令，则释放该命令
    if(session->scpSend_command) {
        LIBSSH2_FREE(session, session->scpSend_command);
    }
    # 如果会话中存在SFTP初始化SFTP，则释放该SFTP
    if(session->sftpInit_sftp) {
        LIBSSH2_FREE(session, session->sftpInit_sftp);
    }

    # 如果会话中存在数据包总数，则释放数据包的载荷
    if(session->packet.total_num) {
        LIBSSH2_FREE(session, session->packet.payload);
    }

    # 清理所有剩余的数据包
    # 当会话中还有未处理的数据包时
    while((pkg = _libssh2_list_first(&session->packets))) {
        packets_left++;  # 计算剩余的数据包数量
        _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
            "packet left with id %d", pkg->data[0]);  # 输出剩余数据包的 ID
        /* unlink the node */
        _libssh2_list_remove(&pkg->node);  # 移除节点

        /* free */
        LIBSSH2_FREE(session, pkg->data);  # 释放数据包的内存
        LIBSSH2_FREE(session, pkg);  # 释放数据包节点的内存
    }
    _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
         "Extra packets left %d", packets_left);  # 输出剩余的额外数据包数量

    if(session->socket_prev_blockstate) {
        /* if the socket was previously blocking, put it back so */
        rc = session_nonblock(session->socket_fd, 0);  # 如果套接字之前是阻塞的，则将其恢复为阻塞状态
        if(rc) {
            _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
             "unable to reset socket's blocking state");  # 输出无法重置套接字阻塞状态的错误信息
        }
    }

    if(session->server_hostkey) {
        LIBSSH2_FREE(session, session->server_hostkey);  # 释放服务器主机密钥的内存
    }

    /* error string */
    if(session->err_msg &&
       ((session->err_flags & LIBSSH2_ERR_FLAG_DUP) != 0)) {
        LIBSSH2_FREE(session, (char *)session->err_msg);  # 如果存在错误消息且为重复错误，则释放错误消息的内存
    }

    LIBSSH2_FREE(session, session);  # 释放会话的内存

    return 0;  # 返回 0 表示成功
}

/*
 * libssh2_session_free
 *
 * 释放分配给会话的内存
 * 同时关闭和释放附加到此会话的任何通道
 */
LIBSSH2_API int
libssh2_session_free(LIBSSH2_SESSION * session)
{
    int rc;

    BLOCK_ADJUST(rc, session, session_free(session) );

    return rc;
}

/*
 * libssh2_session_disconnect_ex
 */
static int
session_disconnect(LIBSSH2_SESSION *session, int reason,
                   const char *description,
                   const char *lang)
{
    unsigned char *s;
    unsigned long descr_len = 0, lang_len = 0;
    int rc;

    if(session->disconnect_state == libssh2_NB_state_idle) {
        _libssh2_debug(session, LIBSSH2_TRACE_TRANS,
                       "Disconnecting: reason=%d, desc=%s, lang=%s", reason,
                       description, lang);
        if(description)
            descr_len = strlen(description);

        if(lang)
            lang_len = strlen(lang);

        if(descr_len > 256)
            return _libssh2_error(session, LIBSSH2_ERROR_INVAL,
                                  "too long description");

        /* 13 = packet_type(1) + reason code(4) + descr_len(4) + lang_len(4) */
        session->disconnect_data_len = descr_len + lang_len + 13;

        s = session->disconnect_data;

        *(s++) = SSH_MSG_DISCONNECT;
        _libssh2_store_u32(&s, reason);
        _libssh2_store_str(&s, description, descr_len);
        /* store length only, lang is sent separately */
        _libssh2_store_u32(&s, lang_len);

        session->disconnect_state = libssh2_NB_state_created;
    }

    rc = _libssh2_transport_send(session, session->disconnect_data,
                                 session->disconnect_data_len,
                                 (unsigned char *)lang, lang_len);
    if(rc == LIBSSH2_ERROR_EAGAIN)
        return rc;

    session->disconnect_state = libssh2_NB_state_idle;

    return 0;
}

/*
 * libssh2_session_disconnect_ex
 */
LIBSSH2_API int
# 断开会话连接，设置状态为非密钥交换状态
libssh2_session_disconnect_ex(LIBSSH2_SESSION *session, int reason,
                              const char *desc, const char *lang)
{
    int rc;
    session->state &= ~LIBSSH2_STATE_EXCHANGING_KEYS;
    # 调整块，调用session_disconnect函数，将结果存储在rc中
    BLOCK_ADJUST(rc, session,
                 session_disconnect(session, reason, desc, lang));

    return rc;
}

/* libssh2_session_methods
 *
 * 返回method_type的当前活动方法
 *
 * 注意：当前lang_cs和lang_sc始终设置为空字符串，而不管实际协商的字符串。字符串不应该被释放
 */
LIBSSH2_API const char *
libssh2_session_methods(LIBSSH2_SESSION * session, int method_type)
{
    /* 所有方法都有char *name作为它们的第一个元素 */
    const LIBSSH2_KEX_METHOD *method = NULL;

    switch(method_type) {
    case LIBSSH2_METHOD_KEX:
        method = session->kex;
        break;

    case LIBSSH2_METHOD_HOSTKEY:
        method = (LIBSSH2_KEX_METHOD *) session->hostkey;
        break;

    case LIBSSH2_METHOD_CRYPT_CS:
        method = (LIBSSH2_KEX_METHOD *) session->local.crypt;
        break;

    case LIBSSH2_METHOD_CRYPT_SC:
        method = (LIBSSH2_KEX_METHOD *) session->remote.crypt;
        break;

    case LIBSSH2_METHOD_MAC_CS:
        method = (LIBSSH2_KEX_METHOD *) session->local.mac;
        break;

    case LIBSSH2_METHOD_MAC_SC:
        method = (LIBSSH2_KEX_METHOD *) session->remote.mac;
        break;

    case LIBSSH2_METHOD_COMP_CS:
        method = (LIBSSH2_KEX_METHOD *) session->local.comp;
        break;

    case LIBSSH2_METHOD_COMP_SC:
        method = (LIBSSH2_KEX_METHOD *) session->remote.comp;
        break;

    case LIBSSH2_METHOD_LANG_CS:
        return "";
        # 返回空字符串

    case LIBSSH2_METHOD_LANG_SC:
        return "";
        # 返回空字符串

    default:
        _libssh2_error(session, LIBSSH2_ERROR_INVAL,
                       "Invalid parameter specified for method_type");
        return NULL;
    }
    # 如果没有协商到方法，则记录错误信息并返回空
    if(!method) {
        # 调用_libssh2_error函数记录错误信息，错误码为LIBSSH2_ERROR_METHOD_NONE，错误信息为"No method negotiated"
        _libssh2_error(session, LIBSSH2_ERROR_METHOD_NONE,
                       "No method negotiated");
        # 返回空指针
        return NULL;
    }
    # 返回协商到的方法的名称
    return method->name;
/* libssh2_session_abstract
 * Retrieve a pointer to the abstract property
 */
LIBSSH2_API void **
libssh2_session_abstract(LIBSSH2_SESSION * session)
{
    // 返回指向抽象属性的指针
    return &session->abstract;
}

/* libssh2_session_last_error
 *
 * Returns error code and populates an error string into errmsg If want_buf is
 * non-zero then the string placed into errmsg must be freed by the calling
 * program. Otherwise it is assumed to be owned by libssh2
 */
LIBSSH2_API int
libssh2_session_last_error(LIBSSH2_SESSION * session, char **errmsg,
                           int *errmsg_len, int want_buf)
{
    size_t msglen = 0;

    /* No error to report */
    // 如果没有错误需要报告
    if(!session->err_code) {
        if(errmsg) {
            if(want_buf) {
                // 如果需要缓冲区，则分配内存并将其置为空
                *errmsg = LIBSSH2_ALLOC(session, 1);
                if(*errmsg) {
                    **errmsg = 0;
                }
            }
            else {
                // 否则假定由libssh2拥有
                *errmsg = (char *) "";
            }
        }
        if(errmsg_len) {
            *errmsg_len = 0;
        }
        return 0;
    }

    if(errmsg) {
        const char *error = session->err_msg ? session->err_msg : "";

        msglen = strlen(error);

        if(want_buf) {
            // 如果需要缓冲区，则复制错误字符串以便调用程序拥有它
            *errmsg = LIBSSH2_ALLOC(session, msglen + 1);
            if(*errmsg) {
                memcpy(*errmsg, error, msglen);
                (*errmsg)[msglen] = 0;
            }
        }
        else
            *errmsg = (char *)error;
    }

    if(errmsg_len) {
        *errmsg_len = msglen;
    }

    return session->err_code;
}

/* libssh2_session_last_errno
 *
 * Returns error code
 */
LIBSSH2_API int
libssh2_session_last_errno(LIBSSH2_SESSION * session)
{
    // 返回错误代码
    return session->err_code;
}
/* libssh2_session_set_last_error
 *
 * 设置会话的内部错误代码。
 *
 * 此函数专门供高级语言包装器（如Python或Perl）使用，这些包装器可能扩展库的功能，同时仍依赖于其错误报告机制。
 */
LIBSSH2_API int
libssh2_session_set_last_error(LIBSSH2_SESSION* session,
                               int errcode,
                               const char *errmsg)
{
    return _libssh2_error_flags(session, errcode, errmsg,
                                LIBSSH2_ERR_FLAG_DUP);
}

/* Libssh2_session_flag
 *
 * 设置/获取会话标志
 *
 * 返回错误代码。
 */
LIBSSH2_API int
libssh2_session_flag(LIBSSH2_SESSION * session, int flag, int value)
{
    switch(flag) {
    case LIBSSH2_FLAG_SIGPIPE:
        session->flag.sigpipe = value;
        break;
    case LIBSSH2_FLAG_COMPRESS:
        session->flag.compress = value;
        break;
    default:
        /* 未知标志 */
        return LIBSSH2_ERROR_INVAL;
    }

    return LIBSSH2_ERROR_NONE;
}

/* _libssh2_session_set_blocking
 *
 * 设置会话的阻塞模式开启或关闭，并在调用此函数时返回先前的状态。注意，此函数不会改变实际涉及的套接字的状态。
 */
int
_libssh2_session_set_blocking(LIBSSH2_SESSION *session, int blocking)
{
    int bl = session->api_block_mode;
    _libssh2_debug(session, LIBSSH2_TRACE_CONN,
                   "Setting blocking mode %s", blocking?"ON":"OFF");
    session->api_block_mode = blocking;

    return bl;
}

/* libssh2_session_set_blocking
 *
 * 设置通道的阻塞模式开启或关闭，类似于套接字的fcntl(fd, F_SETFL, O_NONBLOCK);类型命令
 */
LIBSSH2_API void
libssh2_session_set_blocking(LIBSSH2_SESSION * session, int blocking)
{
    (void) _libssh2_session_set_blocking(session, blocking);
}

/* libssh2_session_get_blocking
 *
 * 返回会话的阻塞模式开启或关闭
 */
LIBSSH2_API int
# 获取会话的阻塞模式
libssh2_session_get_blocking(LIBSSH2_SESSION * session)
{
    return session->api_block_mode;
}

/* libssh2_session_set_timeout
 *
 * 设置会话的阻塞模式超时时间（以毫秒为单位），设置为0表示禁用超时
 */
LIBSSH2_API void
libssh2_session_set_timeout(LIBSSH2_SESSION * session, long timeout)
{
    session->api_timeout = timeout;
}

/* libssh2_session_get_timeout
 *
 * 返回会话的超时时间，如果禁用则返回0
 */
LIBSSH2_API long
libssh2_session_get_timeout(LIBSSH2_SESSION * session)
{
    return session->api_timeout;
}

/*
 * libssh2_poll_channel_read
 *
 * 如果通道上没有等待的数据，则返回0，如果有可用数据则返回非0
 */
LIBSSH2_API int
libssh2_poll_channel_read(LIBSSH2_CHANNEL *channel, int extended)
{
    LIBSSH2_SESSION *session;
    LIBSSH2_PACKET *packet;

    if(!channel)
        return LIBSSH2_ERROR_BAD_USE;

    session = channel->session;
    packet = _libssh2_list_first(&session->packets);

    while(packet) {
        if(packet->data_len < 5) {
            return _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                                  "Packet too small");
        }

        if(channel->local.id == _libssh2_ntohu32(packet->data + 1)) {
            if(extended == 1 &&
                (packet->data[0] == SSH_MSG_CHANNEL_EXTENDED_DATA
                 || packet->data[0] == SSH_MSG_CHANNEL_DATA)) {
                return 1;
            }
            else if(extended == 0 &&
                    packet->data[0] == SSH_MSG_CHANNEL_DATA) {
                return 1;
            }
            # 否则 - 没有任何类型的数据准备好被读取
        }
        packet = _libssh2_list_next(&packet->node);
    }

    return 0;
}

/*
 * poll_channel_write
 *
 * 如果写入通道会阻塞，则返回0，如果可以写入数据而不会阻塞则返回非0
 */
static inline int
poll_channel_write(LIBSSH2_CHANNEL * channel)
{
    return channel->local.window_size ? 1 : 0;
}
/* poll_listener_queued
 *
 * 返回0，如果没有连接等待被接受
 * 返回非0，如果有一个或多个连接可用
 */
static inline int
poll_listener_queued(LIBSSH2_LISTENER * listener)
{
    // 如果队列中有连接等待被接受，则返回1，否则返回0
    return _libssh2_list_first(&listener->queue) ? 1 : 0;
}

/*
 * libssh2_poll
 *
 * 对活动进行套接字、通道和监听器的轮询
 */
LIBSSH2_API int
libssh2_poll(LIBSSH2_POLLFD * fds, unsigned int nfds, long timeout)
{
    long timeout_remaining;
    unsigned int i, active_fds;
#ifdef HAVE_POLL
    LIBSSH2_SESSION *session = NULL;
#ifdef HAVE_ALLOCA
    // 使用alloca分配内存来存储pollfd结构体数组
    struct pollfd *sockets = alloca(sizeof(struct pollfd) * nfds);
#else
    // 如果系统不支持alloca，则使用固定大小的数组
    struct pollfd sockets[256];

    if(nfds > 256)
        /* 没有alloca的系统使用固定大小的数组，如果真的需要，至少如果编译器是C99兼容的话，这个问题可以解决 */
        return -1;
#endif
    /* 设置套接字进行轮询 */
    for(i = 0; i < nfds; i++) {
        fds[i].revents = 0;
        switch(fds[i].type) {
        case LIBSSH2_POLLFD_SOCKET:
            sockets[i].fd = fds[i].fd.socket;
            sockets[i].events = fds[i].events;
            sockets[i].revents = 0;
            break;

        case LIBSSH2_POLLFD_CHANNEL:
            sockets[i].fd = fds[i].fd.channel->session->socket_fd;
            sockets[i].events = POLLIN;
            sockets[i].revents = 0;
            if(!session)
                session = fds[i].fd.channel->session;
            break;

        case LIBSSH2_POLLFD_LISTENER:
            sockets[i].fd = fds[i].fd.listener->session->socket_fd;
            sockets[i].events = POLLIN;
            sockets[i].revents = 0;
            if(!session)
                session = fds[i].fd.listener->session;
            break;

        default:
            if(session)
                _libssh2_error(session, LIBSSH2_ERROR_INVALID_POLL_TYPE,
                               "Invalid descriptor passed to libssh2_poll()");
            return -1;
        }
    }
#elif defined(HAVE_SELECT)
    // 如果定义了 HAVE_SELECT，则执行以下代码块
    LIBSSH2_SESSION *session = NULL;
    libssh2_socket_t maxfd = 0;
    fd_set rfds, wfds;
    struct timeval tv;

    // 清空读取和写入文件描述符集合
    FD_ZERO(&rfds);
    FD_ZERO(&wfds);
    // 遍历文件描述符数组
    for(i = 0; i < nfds; i++) {
        // 重置事件标志
        fds[i].revents = 0;
        switch(fds[i].type) {
        case LIBSSH2_POLLFD_SOCKET:
            // 如果有可读事件，则将文件描述符加入到读取文件描述符集合中
            if(fds[i].events & LIBSSH2_POLLFD_POLLIN) {
                FD_SET(fds[i].fd.socket, &rfds);
                // 更新最大文件描述符
                if(fds[i].fd.socket > maxfd)
                    maxfd = fds[i].fd.socket;
            }
            // 如果有可写事件，则将文件描述符加入到写入文件描述符集合中
            if(fds[i].events & LIBSSH2_POLLFD_POLLOUT) {
                FD_SET(fds[i].fd.socket, &wfds);
                // 更新最大文件描述符
                if(fds[i].fd.socket > maxfd)
                    maxfd = fds[i].fd.socket;
            }
            break;

        case LIBSSH2_POLLFD_CHANNEL:
            // 将通道对应的会话的文件描述符加入到读取文件描述符集合中
            FD_SET(fds[i].fd.channel->session->socket_fd, &rfds);
            // 更新最大文件描述符
            if(fds[i].fd.channel->session->socket_fd > maxfd)
                maxfd = fds[i].fd.channel->session->socket_fd;
            // 如果会话为空，则将当前通道对应的会话赋值给 session
            if(!session)
                session = fds[i].fd.channel->session;
            break;

        case LIBSSH2_POLLFD_LISTENER:
            // 将监听器对应的会话的文件描述符加入到读取文件描述符集合中
            FD_SET(fds[i].fd.listener->session->socket_fd, &rfds);
            // 更新最大文件描述符
            if(fds[i].fd.listener->session->socket_fd > maxfd)
                maxfd = fds[i].fd.listener->session->socket_fd;
            // 如果会话为空，则将当前监听器对应的会话赋值给 session
            if(!session)
                session = fds[i].fd.listener->session;
            break;

        default:
            // 如果会话不为空，则抛出无效的描述符错误
            if(session)
                _libssh2_error(session, LIBSSH2_ERROR_INVALID_POLL_TYPE,
                               "Invalid descriptor passed to libssh2_poll()");
            return -1;
        }
    }
#else
    /* No select() or poll()
     * no sockets structure to setup
     */

    // 没有 select() 或 poll()，不需要设置任何套接字结构
    timeout = 0;
#endif /* HAVE_POLL or HAVE_SELECT */

    // 设置剩余超时时间
    timeout_remaining = timeout;
    // 执行循环
    do {
#if defined(HAVE_POLL) || defined(HAVE_SELECT)
        int sysret;
#ifdef HAVE_POLL
#ifdef HAVE_LIBSSH2_GETTIMEOFDAY
        {
            // 如果平台支持 gettimeofday 函数，则使用该函数计算 poll 函数的超时时间
            struct timeval tv_begin, tv_end;

            // 获取开始时间
            _libssh2_gettimeofday((struct timeval *) &tv_begin, NULL);
            // 调用 poll 函数
            sysret = poll(sockets, nfds, timeout_remaining);
            // 获取结束时间
            _libssh2_gettimeofday((struct timeval *) &tv_end, NULL);
            // 计算剩余超时时间
            timeout_remaining -= (tv_end.tv_sec - tv_begin.tv_sec) * 1000;
            timeout_remaining -= (tv_end.tv_usec - tv_begin.tv_usec) / 1000;
        }
#else
        /* 如果平台不支持 gettimeofday 函数，
         * 则将调用设置为非阻塞，并立即返回
         */
        // 调用 poll 函数
        sysret = poll(sockets, nfds, timeout_remaining);
        // 将剩余超时时间设置为 0
        timeout_remaining = 0;
#endif /* HAVE_GETTIMEOFDAY */

        // 如果有文件描述符就绪
        if(sysret > 0) {
            // 遍历所有文件描述符
            for(i = 0; i < nfds; i++) {
                // 根据文件描述符类型进行不同的处理
                switch(fds[i].type) {
                case LIBSSH2_POLLFD_SOCKET:
                    // 将就绪的事件赋值给 revents，并清空 sockets[i].revents
                    fds[i].revents = sockets[i].revents;
                    sockets[i].revents = 0; /* In case we loop again, be
                                               nice */
                    // 如果 revents 不为 0，则增加活跃文件描述符的计数
                    if(fds[i].revents) {
                        active_fds++;
                    }
                    break;
                case LIBSSH2_POLLFD_CHANNEL:
                    // 如果 sockets[i].events 中包含 POLLIN 事件
                    if(sockets[i].events & POLLIN) {
                        /* Spin session until no data available */
                        // 循环读取通道中的数据，直到没有数据可读
                        while(_libssh2_transport_read(fds[i].fd.
                                                      channel->session)
                              > 0);
                    }
                    // 如果 sockets[i].revents 中包含 POLLHUP 事件
                    if(sockets[i].revents & POLLHUP) {
                        // 将通道关闭和会话关闭的事件添加到 revents 中
                        fds[i].revents |=
                            LIBSSH2_POLLFD_CHANNEL_CLOSED |
                            LIBSSH2_POLLFD_SESSION_CLOSED;
                    }
                    // 清空 sockets[i].revents
                    sockets[i].revents = 0;
                    break;
                case LIBSSH2_POLLFD_LISTENER:
                    // 如果 sockets[i].events 中包含 POLLIN 事件
                    if(sockets[i].events & POLLIN) {
                        /* Spin session until no data available */
                        // 循环读取监听器中的数据，直到没有数据可读
                        while(_libssh2_transport_read(fds[i].fd.
                                                      listener->session)
                              > 0);
                    }
                    // 如果 sockets[i].revents 中包含 POLLHUP 事件
                    if(sockets[i].revents & POLLHUP) {
                        // 将监听器关闭和会话关闭的事件添加到 revents 中
                        fds[i].revents |=
                            LIBSSH2_POLLFD_LISTENER_CLOSED |
                            LIBSSH2_POLLFD_SESSION_CLOSED;
                    }
                    // 清空 sockets[i].revents
                    sockets[i].revents = 0;
                    break;
                }
            }
        }
# 如果定义了 HAVE_SELECT 宏
#elif defined(HAVE_SELECT)
    # 设置 select 函数的超时时间为 timeout_remaining 毫秒
    tv.tv_sec = timeout_remaining / 1000;
    tv.tv_usec = (timeout_remaining % 1000) * 1000;
    # 如果平台支持 gettimeofday 函数
    # 获取当前时间 tv_begin
    # 调用 select 函数
    # 获取当前时间 tv_end
    # 计算经过的时间，并更新 timeout_remaining
#ifdef HAVE_LIBSSH2_GETTIMEOFDAY
    {
        struct timeval tv_begin, tv_end;
        _libssh2_gettimeofday((struct timeval *) &tv_begin, NULL);
        sysret = select(maxfd + 1, &rfds, &wfds, NULL, &tv);
        _libssh2_gettimeofday((struct timeval *) &tv_end, NULL);
        timeout_remaining -= (tv_end.tv_sec - tv_begin.tv_sec) * 1000;
        timeout_remaining -= (tv_end.tv_usec - tv_begin.tv_usec) / 1000;
    }
#else
    # 如果平台不支持 gettimeofday 函数
    # 调用 select 函数，并将超时时间设为非阻塞
    sysret = select(maxfd + 1, &rfds, &wfds, NULL, &tv);
    # 将 timeout_remaining 设为 0
    timeout_remaining = 0;
#endif

        if(sysret > 0) {
            for(i = 0; i < nfds; i++) {
                switch(fds[i].type) {
                case LIBSSH2_POLLFD_SOCKET:
                    if(FD_ISSET(fds[i].fd.socket, &rfds)) {
                        fds[i].revents |= LIBSSH2_POLLFD_POLLIN;
                    }
                    if(FD_ISSET(fds[i].fd.socket, &wfds)) {
                        fds[i].revents |= LIBSSH2_POLLFD_POLLOUT;
                    }
                    if(fds[i].revents) {
                        active_fds++;
                    }
                    break;

                case LIBSSH2_POLLFD_CHANNEL:
                    if(FD_ISSET(fds[i].fd.channel->session->socket_fd,
                                &rfds)) {
                        /* Spin session until no data available */
                        while(_libssh2_transport_read(fds[i].fd.
                                                      channel->session)
                              > 0);
                    }
                    break;

                case LIBSSH2_POLLFD_LISTENER:
                    if(FD_ISSET
                        (fds[i].fd.listener->session->socket_fd, &rfds)) {
                        /* Spin session until no data available */
                        while(_libssh2_transport_read(fds[i].fd.
                                                      listener->session)
                              > 0);
                    }
                    break;
                }
            }
        }
#endif /* else no select() or poll() -- timeout (and by extension
        * timeout_remaining) will be equal to 0 */
    } while((timeout_remaining > 0) && !active_fds);

    return active_fds;
}

/*
 * libssh2_session_block_directions
 *
 * Get blocked direction when a function returns LIBSSH2_ERROR_EAGAIN
 * Returns LIBSSH2_SOCKET_BLOCK_INBOUND if recv() blocked
 * or LIBSSH2_SOCKET_BLOCK_OUTBOUND if send() blocked
 */
LIBSSH2_API int
# 返回会话的套接字阻塞方向
libssh2_session_block_directions(LIBSSH2_SESSION *session)
{
    return session->socket_block_directions;
}

/* libssh2_session_banner_get
 * 获取远程横幅（服务器ID字符串）
 */

LIBSSH2_API const char *
libssh2_session_banner_get(LIBSSH2_SESSION *session)
{
    /* 避免会话为空时出现核心转储 */
    if(NULL == session)
        return NULL;

    # 如果远程横幅为空，则返回空
    if(NULL == session->remote.banner)
        return NULL;

    # 返回远程横幅
    return (const char *) session->remote.banner;
}
```