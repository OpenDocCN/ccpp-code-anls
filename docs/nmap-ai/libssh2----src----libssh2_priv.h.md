# `nmap\libssh2\src\libssh2_priv.h`

```cpp
#ifndef __LIBSSH2_PRIV_H
#define __LIBSSH2_PRIV_H
/* 定义宏，用于条件编译，确保该头文件只被包含一次 */
/* 版权声明，包括作者信息和版权声明 */
/* 允许在源码和二进制形式下重新分发和使用，需满足一定条件 */
/* 在二进制形式下，需要在文档和/或其他提供的材料中重现版权声明、条件列表和下面的免责声明 */
/* 不得使用版权所有者的名称或其他贡献者的名称，未经特定事先书面许可，不得用于认可或推广从本软件衍生的产品 */
/* 免责声明，声明本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保 */
/* 在任何责任理论下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式），都不承担任何直接、间接、附带、特殊、惩罚性或后果性的损害赔偿责任 */
/* 定义宏，用于标识为 libssh2 库 */
#include "libssh2_config.h"
/* 如果系统支持 Windows.h，则包含该头文件 */
#ifdef HAVE_WINDOWS_H
#ifndef WIN32_LEAN_AND_MEAN
#define WIN32_LEAN_AND_MEAN
#endif
#include <windows.h>
#undef WIN32_LEAN_AND_MEAN
#endif
/* 如果系统支持 WS2TCPIP.h，则包含该头文件 */
#ifdef HAVE_WS2TCPIP_H
/* 包含 Windows 套接字 API 头文件 */
#include <ws2tcpip.h>
#endif

/* 包含标准输入输出头文件 */
#include <stdio.h>
/* 包含时间处理头文件 */
#include <time.h>

/* 下面的 CPP 块应该只在 session.c 和 packet.c 中，但是 AIX 有对 'events' 和 'revents' 的 #define，
   而我们在 libssh2.h 中使用了这些名称，所以我们需要先包含 AIX 头文件，以确保所有代码都使用一致的这些字段名称。
   虽然可以争论最好的做法是修改 libssh2.h 使用其他名称，但那样会破坏向后兼容性。
*/
#ifdef HAVE_POLL
# include <poll.h>
#else
# if defined(HAVE_SELECT) && !defined(WIN32)
# ifdef HAVE_SYS_SELECT_H
# include <sys/select.h>
# else
# include <sys/time.h>
# include <sys/types.h>
# endif
# endif
#endif

/* 在某些平台上需要 struct iovec */
#ifdef HAVE_SYS_UIO_H
#include <sys/uio.h>
#endif

#ifdef HAVE_SYS_SOCKET_H
# include <sys/socket.h>
#endif
#ifdef HAVE_SYS_IOCTL_H
# include <sys/ioctl.h>
#endif
#ifdef HAVE_INTTYPES_H
#include <inttypes.h>
#endif

/* 包含 libssh2 相关头文件 */
#include "libssh2.h"
#include "libssh2_publickey.h"
#include "libssh2_sftp.h"
#include "misc.h" /* for the linked list stuff */

/* 定义 FALSE 和 TRUE */
#ifndef FALSE
#define FALSE 0
#endif
#ifndef TRUE
#define TRUE 1
#endif

#ifdef _MSC_VER
/* "inline" 关键字只在 C++ 引擎中有效！ */
#define inline __inline
#endif

/* 3DS 似乎没有 iovec */
#if defined(WIN32) || defined(_3DS)

/* 定义 iovec 结构体 */
struct iovec {
    size_t iov_len;
    void *iov_base;
};

#endif

/* 在 WIN32 平台上提供 iovec / writev */
#ifdef WIN32

/* 定义 writev 函数 */
static inline int writev(int sock, struct iovec *iov, int nvecs)
{
    DWORD ret;
    if(WSASend(sock, (LPWSABUF)iov, nvecs, &ret, 0, NULL, NULL) == 0) {
        return ret;
    }
    return -1;
}

#endif /* WIN32 */

#ifdef __OS400__
/* 强制参数类型 */
#define send(s, b, l, f)    send((s), (unsigned char *) (b), (l), (f))
#endif

/* 包含加密相关头文件 */
#include "crypto.h"

#ifdef HAVE_WINSOCK2_H

#include <winsock2.h>
#include <ws2tcpip.h>

#endif

/* 定义 SIZE_MAX */
#ifndef SIZE_MAX
#if _WIN64
#define SIZE_MAX 0xFFFFFFFFFFFFFFFF
#else
#ifndef SIZE_MAX
#define SIZE_MAX 0xFFFFFFFF
#endif
#endif

#ifndef UINT_MAX
#define UINT_MAX 0xFFFFFFFF
#endif

/* 定义最大 SSH 数据包长度为 35000 字节 */
#define MAX_SSH_PACKET_LEN 35000
/* 定义最大 SHA 摘要长度为 SHA512_DIGEST_LENGTH */
#define MAX_SHA_DIGEST_LEN SHA512_DIGEST_LENGTH

/* 定义宏 LIBSSH2_ALLOC，用于分配内存 */
#define LIBSSH2_ALLOC(session, count) \
  session->alloc((count), &(session)->abstract)
/* 定义宏 LIBSSH2_CALLOC，用于分配并清零内存 */
#define LIBSSH2_CALLOC(session, count) _libssh2_calloc(session, count)
/* 定义宏 LIBSSH2_REALLOC，用于重新分配内存 */
#define LIBSSH2_REALLOC(session, ptr, count) \
 ((ptr) ? session->realloc((ptr), (count), &(session)->abstract) : \
 session->alloc((count), &(session)->abstract))
/* 定义宏 LIBSSH2_FREE，用于释放内存 */
#define LIBSSH2_FREE(session, ptr) \
 session->free((ptr), &(session)->abstract)
/* 定义宏 LIBSSH2_IGNORE，用于发送 SSH 忽略消息 */
#define LIBSSH2_IGNORE(session, data, datalen) \
 session->ssh_msg_ignore((session), (data), (datalen), &(session)->abstract)
/* 定义宏 LIBSSH2_DEBUG，用于发送 SSH 调试消息 */
#define LIBSSH2_DEBUG(session, always_display, message, message_len, \
                      language, language_len)    \
    session->ssh_msg_debug((session), (always_display), (message), \
                           (message_len), (language), (language_len), \
                           &(session)->abstract)
/* 定义宏 LIBSSH2_DISCONNECT，用于发送 SSH 断开连接消息 */
#define LIBSSH2_DISCONNECT(session, reason, message, message_len, \
                           language, language_len)                \
    session->ssh_msg_disconnect((session), (reason), (message),   \
                                (message_len), (language), (language_len), \
                                &(session)->abstract)

/* 定义宏 LIBSSH2_MACERROR，用于处理 SSH MAC 错误 */
#define LIBSSH2_MACERROR(session, data, datalen)         \
    session->macerror((session), (data), (datalen), &(session)->abstract)
/* 定义宏 LIBSSH2_X11_OPEN，用于打开 SSH X11 通道 */
#define LIBSSH2_X11_OPEN(channel, shost, sport)          \
    channel->session->x11(((channel)->session), (channel), \
                          (shost), (sport), (&(channel)->session->abstract))
# 定义一个宏，用于关闭 SSH 会话中的通道
#define LIBSSH2_CHANNEL_CLOSE(session, channel)          \
    channel->close_cb((session), &(session)->abstract, \
                      (channel), &(channel)->abstract)

# 定义一个宏，用于发送文件描述符
#define LIBSSH2_SEND_FD(session, fd, buffer, length, flags) \
    (session->send)(fd, buffer, length, flags, &session->abstract)
# 定义一个宏，用于接收文件描述符
#define LIBSSH2_RECV_FD(session, fd, buffer, length, flags) \
    (session->recv)(fd, buffer, length, flags, &session->abstract)

# 定义一个宏，用于发送数据
#define LIBSSH2_SEND(session, buffer, length, flags)  \
    LIBSSH2_SEND_FD(session, session->socket_fd, buffer, length, flags)
# 定义一个宏，用于接收数据
#define LIBSSH2_RECV(session, buffer, length, flags)                    \
    LIBSSH2_RECV_FD(session, session->socket_fd, buffer, length, flags)

# 定义一系列结构体和枚举类型，用于处理非阻塞状态下的 SSH 通信
typedef struct _LIBSSH2_KEX_METHOD LIBSSH2_KEX_METHOD;
typedef struct _LIBSSH2_HOSTKEY_METHOD LIBSSH2_HOSTKEY_METHOD;
typedef struct _LIBSSH2_CRYPT_METHOD LIBSSH2_CRYPT_METHOD;
typedef struct _LIBSSH2_COMP_METHOD LIBSSH2_COMP_METHOD;
typedef struct _LIBSSH2_PACKET LIBSSH2_PACKET;
typedef enum
{
    libssh2_NB_state_idle = 0,
    libssh2_NB_state_allocated,
    libssh2_NB_state_created,
    libssh2_NB_state_sent,
    libssh2_NB_state_sent1,
    libssh2_NB_state_sent2,
    libssh2_NB_state_sent3,
    libssh2_NB_state_sent4,
    libssh2_NB_state_sent5,
    libssh2_NB_state_sent6,
    libssh2_NB_state_sent7,
    libssh2_NB_state_jump1,
    libssh2_NB_state_jump2,
    libssh2_NB_state_jump3,
    libssh2_NB_state_jump4,
    libssh2_NB_state_jump5,
    libssh2_NB_state_end
} libssh2_nonblocking_states;
typedef struct packet_require_state_t
{
    libssh2_nonblocking_states state;
    time_t start;
} packet_require_state_t;
typedef struct packet_requirev_state_t
{
    time_t start;
} packet_requirev_state_t;
typedef struct kmdhgGPshakex_state_t
{
    libssh2_nonblocking_states state;
    unsigned char *e_packet;
    unsigned char *s_packet;
    unsigned char *tmp;
    unsigned char h_sig_comp[MAX_SHA_DIGEST_LEN];
    unsigned char c;
    size_t e_packet_len;
    # 定义一个变量，用于存储数据包的长度
    size_t s_packet_len;
    # 定义一个临时变量
    size_t tmp_len;
    # 定义一个大数运算的上下文
    _libssh2_bn_ctx *ctx;
    # 定义一个 DH 上下文
    _libssh2_dh_ctx x;
    # 定义一个大数 e
    _libssh2_bn *e;
    # 定义一个大数 f
    _libssh2_bn *f;
    # 定义一个大数 k
    _libssh2_bn *k;
    # 定义一个存储 f 值的无符号字符指针
    unsigned char *f_value;
    # 定义一个存储 k 值的无符号字符指针
    unsigned char *k_value;
    # 定义一个存储 h 签名的无符号字符指针
    unsigned char *h_sig;
    # 定义一个存储 f 值长度的变量
    size_t f_value_len;
    # 定义一个存储 k 值长度的变量
    size_t k_value_len;
    # 定义一个存储 h 签名长度的变量
    size_t h_sig_len;
    # 定义一个交换哈希的指针
    void *exchange_hash;
    # 定义一个数据包要求的状态
    packet_require_state_t req_state;
    # 定义一个非阻塞状态
    libssh2_nonblocking_states burn_state;
} kmdhgGPshakex_state_t;  # 结构体 kmdhgGPshakex_state_t 结尾

typedef struct key_exchange_state_low_t
{
    libssh2_nonblocking_states state;  # libssh2 非阻塞状态
    packet_require_state_t req_state;  # 数据包要求状态
    kmdhgGPshakex_state_t exchange_state;  # kmdhgGPshakex_state_t 交换状态
    _libssh2_bn *p;             /* SSH2 defined value (p_value) */  # SSH2 定义的值 p
    _libssh2_bn *g;             /* SSH2 defined value (2) */  # SSH2 定义的值 g
    unsigned char request[256]; /* Must fit EC_MAX_POINT_LEN + data */  # 请求数组，长度为 256
    unsigned char *data;  # 数据指针
    size_t request_len;  # 请求长度
    size_t data_len;  # 数据长度
    _libssh2_ec_key *private_key;   /* SSH2 ecdh private key */  # SSH2 ecdh 私钥
    unsigned char *public_key_oct;  /* SSH2 ecdh public key octal value */  # SSH2 ecdh 公钥八进制值
    size_t public_key_oct_len;      /* SSH2 ecdh public key octal value
                                       length */  # SSH2 ecdh 公钥八进制值长度
    unsigned char *curve25519_public_key; /* curve25519 public key, 32
                                             bytes */  # curve25519 公钥，32 字节
    unsigned char *curve25519_private_key; /* curve25519 private key, 32
                                              bytes */  # curve25519 私钥，32 字节
} key_exchange_state_low_t;  # 结构体 key_exchange_state_low_t

typedef struct key_exchange_state_t
{
    libssh2_nonblocking_states state;  # libssh2 非阻塞状态
    packet_require_state_t req_state;  # 数据包要求状态
    key_exchange_state_low_t key_state_low;  # key_exchange_state_low_t 低级状态
    unsigned char *data;  # 数据指针
    size_t data_len;  # 数据长度
    unsigned char *oldlocal;  # 旧本地指针
    size_t oldlocal_len;  # 旧本地长度
} key_exchange_state_t;  # 结构体 key_exchange_state_t

#define FwdNotReq "Forward not requested"  # 宏定义 FwdNotReq

typedef struct packet_queue_listener_state_t
{
    libssh2_nonblocking_states state;  # libssh2 非阻塞状态
    unsigned char packet[17 + (sizeof(FwdNotReq) - 1)];  # 数据包数组，长度为 17 + (FwdNotReq 长度 - 1)
    unsigned char *host;  # 主机指针
    unsigned char *shost;  # shost 指针
    uint32_t sender_channel;  # 发送通道
    uint32_t initial_window_size;  # 初始窗口大小
    uint32_t packet_size;  # 数据包大小
    uint32_t port;  # 端口
    uint32_t sport;  # sport
    uint32_t host_len;  # 主机长度
    uint32_t shost_len;  # shost 长度
    LIBSSH2_CHANNEL *channel;  # libssh2 通道
} packet_queue_listener_state_t;  # 结构体 packet_queue_listener_state_t

#define X11FwdUnAvil "X11 Forward Unavailable"  # 宏定义 X11FwdUnAvil

typedef struct packet_x11_open_state_t
{
    libssh2_nonblocking_states state;  # libssh2 非阻塞状态
    unsigned char packet[17 + (sizeof(X11FwdUnAvil) - 1)];  # 数据包数组，长度为 17 + (X11FwdUnAvil 长度 - 1)
    unsigned char *shost;  # shost 指针
    uint32_t sender_channel;  # 发送通道
    # 定义一个32位无符号整数变量initial_window_size，用于存储初始窗口大小
    uint32_t initial_window_size;
    # 定义一个32位无符号整数变量packet_size，用于存储数据包大小
    uint32_t packet_size;
    # 定义一个32位无符号整数变量sport，用于存储源端口号
    uint32_t sport;
    # 定义一个32位无符号整数变量shost_len，用于存储源主机长度
    uint32_t shost_len;
    # 定义一个指向LIBSSH2_CHANNEL类型的指针变量channel，用于表示SSH2通道
    LIBSSH2_CHANNEL *channel;
} packet_x11_open_state_t;

# 定义_LIBSSH2_PACKET结构体
struct _LIBSSH2_PACKET
{
    struct list_node node; /* linked list header */

    /* the raw unencrypted payload */
    unsigned char *data;  # 未加密的原始数据
    size_t data_len;  # 数据长度

    /* Where to start reading data from,
     * used for channel data that's been partially consumed */
    size_t data_head;  # 从哪里开始读取数据，用于部分消耗的通道数据
};

# 定义libssh2_channel_data结构体
typedef struct _libssh2_channel_data
{
    /* Identifier */
    uint32_t id;  # 标识符

    /* Limits and restrictions */
    uint32_t window_size_initial, window_size, packet_size;  # 窗口大小和数据包大小限制

    /* Set to 1 when CHANNEL_CLOSE / CHANNEL_EOF sent/received */
    char close, eof, extended_data_ignore_mode;  # 当CHANNEL_CLOSE / CHANNEL_EOF被发送/接收时设置为1
} libssh2_channel_data;

# 定义_LIBSSH2_CHANNEL结构体
struct _LIBSSH2_CHANNEL
{
    struct list_node node;  # 链表节点

    unsigned char *channel_type;  # 通道类型
    unsigned channel_type_len;  # 通道类型长度

    /* channel's program exit status */
    int exit_status;  # 通道的程序退出状态

    /* channel's program exit signal (without the SIG prefix) */
    char *exit_signal;  # 通道的程序退出信号（不包括SIG前缀）

    libssh2_channel_data local, remote;  # 本地和远程通道数据
    /* Amount of bytes to be refunded to receive window (but not yet sent) */
    uint32_t adjust_queue;  # 要退还以接收窗口的字节数（但尚未发送）
    /* Data immediately available for reading */
    uint32_t read_avail;  # 立即可用于读取的数据量

    LIBSSH2_SESSION *session;  # 会话指针

    void *abstract;  # 抽象指针
    LIBSSH2_CHANNEL_CLOSE_FUNC((*close_cb));  # 关闭通道回调函数指针

    /* State variables used in libssh2_channel_setenv_ex() */
    libssh2_nonblocking_states setenv_state;  # 在libssh2_channel_setenv_ex()中使用的状态变量
    unsigned char *setenv_packet;  # 设置环境数据包
    size_t setenv_packet_len;  # 设置环境数据包长度
    unsigned char setenv_local_channel[4];  # 本地通道数据

    packet_requirev_state_t setenv_packet_requirev_state;  # setenv数据包要求状态

    /* State variables used in libssh2_channel_request_pty_ex()
       libssh2_channel_request_pty_size_ex() */
    libssh2_nonblocking_states reqPTY_state;  # 在libssh2_channel_request_pty_ex()中使用的状态变量
    unsigned char reqPTY_packet[41 + 256];  # 请求PTY数据包
    size_t reqPTY_packet_len;  # 请求PTY数据包长度
    unsigned char reqPTY_local_channel[4];  # 本地通道数据

    packet_requirev_state_t reqPTY_packet_requirev_state;  # reqPTY数据包要求状态

    /* State variables used in libssh2_channel_x11_req_ex() */
    libssh2_nonblocking_states reqX11_state;  # 在libssh2_channel_x11_req_ex()中使用的状态变量
    unsigned char *reqX11_packet;  # 请求X11数据包
    size_t reqX11_packet_len;  # 请求X11数据包长度
    # 用于存储本地通道的请求数据
    unsigned char reqX11_local_channel[4];
    # 用于存储通道请求状态的数据结构
    packet_requirev_state_t reqX11_packet_requirev_state;

    # 用于存储 libssh2_channel_process_startup() 中的状态变量
    libssh2_nonblocking_states process_state;
    # 用于存储 libssh2_channel_process_startup() 中的数据包
    unsigned char *process_packet;
    # 用于存储 libssh2_channel_process_startup() 中的数据包长度
    size_t process_packet_len;
    # 用于存储 libssh2_channel_process_startup() 中的本地通道数据
    unsigned char process_local_channel[4];
    # 用于存储 libssh2_channel_process_startup() 中的数据包请求状态
    packet_requirev_state_t process_packet_requirev_state;

    # 用于存储 libssh2_channel_flush_ex() 中的状态变量
    libssh2_nonblocking_states flush_state;
    # 用于存储 libssh2_channel_flush_ex() 中的退款字节数
    size_t flush_refund_bytes;
    # 用于存储 libssh2_channel_flush_ex() 中的刷新字节数
    size_t flush_flush_bytes;

    # 用于存储 libssh2_channel_receive_window_adjust() 中的状态变量
    libssh2_nonblocking_states adjust_state;
    # 用于存储 libssh2_channel_receive_window_adjust() 中的调整数据
    unsigned char adjust_adjust[9];     /* packet_type(1) + channel(4) + adjustment(4) */

    # 用于存储 libssh2_channel_read_ex() 中的状态变量
    libssh2_nonblocking_states read_state;
    # 用于存储 libssh2_channel_read_ex() 中的本地 ID
    uint32_t read_local_id;

    # 用于存储 libssh2_channel_write_ex() 中的状态变量
    libssh2_nonblocking_states write_state;
    # 用于存储 libssh2_channel_write_ex() 中的数据包
    unsigned char write_packet[13];
    # 用于存储 libssh2_channel_write_ex() 中的数据包长度
    size_t write_packet_len;
    # 用于存储 libssh2_channel_write_ex() 中的缓冲区写入字节数
    size_t write_bufwrite;

    # 用于存储 libssh2_channel_close() 中的状态变量
    libssh2_nonblocking_states close_state;
    # 用于存储 libssh2_channel_close() 中的数据包
    unsigned char close_packet[5];

    # 用于存储 libssh2_channel_wait_closedeof() 中的状态变量
    libssh2_nonblocking_states wait_eof_state;

    # 用于存储 libssh2_channel_wait_closed() 中的状态变量
    libssh2_nonblocking_states wait_closed_state;

    # 用于存储 libssh2_channel_free() 中的状态变量
    libssh2_nonblocking_states free_state;

    # 用于存储 libssh2_channel_handle_extended_data2() 中的状态变量
    libssh2_nonblocking_states extData2_state;

    # 用于存储 libssh2_channel_request_auth_agent() 中的状态变量
    libssh2_nonblocking_states req_auth_agent_try_state;
    # 用于存储 libssh2_channel_request_auth_agent() 中的状态变量
    libssh2_nonblocking_states req_auth_agent_state;
    # 用于存储 libssh2_channel_request_auth_agent() 中的数据包
    unsigned char req_auth_agent_packet[36];
    # 用于存储 libssh2_channel_request_auth_agent() 中的数据包长度
    size_t req_auth_agent_packet_len;
    # 定义一个包含4个无符号字符的数组，用于存储请求认证代理本地通道的数据
    unsigned char req_auth_agent_local_channel[4];
    # 定义一个packet_requirev_state_t类型的变量，用于存储请求认证代理的状态信息
    packet_requirev_state_t req_auth_agent_requirev_state;
};

// 定义一个结构体 _LIBSSH2_LISTENER
struct _LIBSSH2_LISTENER
{
    struct list_node node; // 链表节点，用于链接多个监听器

    LIBSSH2_SESSION *session; // 指向 LIBSSH2_SESSION 结构体的指针

    char *host; // 主机名
    int port; // 端口号

    // 用于存储该监听器的通道列表
    struct list_head queue;

    int queue_size; // 队列大小
    int queue_maxsize; // 队列最大大小

    // 用于 libssh2_channel_forward_cancel() 中的状态变量
    libssh2_nonblocking_states chanFwdCncl_state;
    unsigned char *chanFwdCncl_data; // 数据
    size_t chanFwdCncl_data_len; // 数据长度
};

// 定义一个结构体 _libssh2_endpoint_data
typedef struct _libssh2_endpoint_data
{
    unsigned char *banner; // 横幅

    unsigned char *kexinit; // 密钥交换初始化数据
    size_t kexinit_len; // 初始化数据长度

    const LIBSSH2_CRYPT_METHOD *crypt; // 加密方法
    void *crypt_abstract; // 加密方法的抽象数据

    const struct _LIBSSH2_MAC_METHOD *mac; // MAC 方法
    uint32_t seqno; // 序列号
    void *mac_abstract; // MAC 方法的抽象数据

    const LIBSSH2_COMP_METHOD *comp; // 压缩方法
    void *comp_abstract; // 压缩方法的抽象数据

    // 方法偏好设置
    char *crypt_prefs; // 加密方法偏好
    char *mac_prefs; // MAC 方法偏好
    char *comp_prefs; // 压缩方法偏好
    char *lang_prefs; // 语言偏好
} libssh2_endpoint_data;

// 定义 PACKETBUFSIZE 常量
#define PACKETBUFSIZE (1024*16)

// 定义一个结构体 transportpacket
struct transportpacket
{
    // 用于存储传入数据的缓冲区
    unsigned char buf[PACKETBUFSIZE];
    unsigned char init[5]; // 传入数据流的前 5 个字节，仍然加密
    size_t writeidx; // 下一个写入缓冲区的位置
    size_t readidx; // 下一个从缓冲区读取的位置
    uint32_t packet_length; // 从网络数据中读取的最新 packet_length
    uint8_t padding_length; // 从网络数据中读取的最新 padding_length
    size_t data_num; // 到目前为止已经读取的总数据量
};
    # 总包大小，以字节为单位。一个完整的包包括数据包长度、填充长度、4个字节和消息认证码长度
    size_t total_num;       

    # 指向一个由 LIBSSH2_ALLOC() 分配的区域的指针，用于写入解密后的数据
    unsigned char *payload; 

    # 写指针，指向当前正在写入解密数据的 payload 区域
    unsigned char *wptr;    

    # 用于传出数据的区域，最大长度为 MAX_SSH_PACKET_LEN
    unsigned char outbuf[MAX_SSH_PACKET_LEN]; 

    # outbuf 的大小，以字节为单位
    int ototal_num;         

    # 原始数据的指针
    const unsigned char *odata; 

    # 存储在 outbuf 中的原始数据的大小
    size_t olen;            

    # 已发送的字节数
    size_t osent;           
};

// 定义了一个结构体 _LIBSSH2_PUBLICKEY，用于存储公钥相关的信息
struct _LIBSSH2_PUBLICKEY
{
    LIBSSH2_CHANNEL *channel;  // 通道
    uint32_t version;  // 版本号

    /* libssh2_publickey_packet_receive() 中使用的状态变量 */
    libssh2_nonblocking_states receive_state;  // 接收状态
    unsigned char *receive_packet;  // 接收数据包
    size_t receive_packet_len;  // 接收数据包长度

    /* libssh2_publickey_add_ex() 中使用的状态变量 */
    libssh2_nonblocking_states add_state;  // 添加状态
    unsigned char *add_packet;  // 添加数据包
    unsigned char *add_s;  // 添加 s

    /* libssh2_publickey_remove_ex() 中使用的状态变量 */
    libssh2_nonblocking_states remove_state;  // 移除状态
    unsigned char *remove_packet;  // 移除数据包
    unsigned char *remove_s;  // 移除 s

    /* libssh2_publickey_list_fetch() 中使用的状态变量 */
    libssh2_nonblocking_states listFetch_state;  // 列表获取状态
    unsigned char *listFetch_s;  // 列表获取 s
    unsigned char listFetch_buffer[12];  // 列表获取缓冲区
    unsigned char *listFetch_data;  // 列表获取数据
    size_t listFetch_data_len;  // 列表获取数据长度
};

// 定义了一个常量 LIBSSH2_SCP_RESPONSE_BUFLEN，值为 256
#define LIBSSH2_SCP_RESPONSE_BUFLEN     256

// 定义了一个结构体 flags，用于存储标志位
struct flags {
    int sigpipe;  // SIGPIPE 标志
    int compress; // 压缩标志
};

// 定义了一个结构体 _LIBSSH2_SESSION，用于存储会话相关的信息
struct _LIBSSH2_SESSION
{
    /* 内存管理回调 */
    void *abstract;  // 抽象指针
    LIBSSH2_ALLOC_FUNC((*alloc));  // 分配内存回调函数
    LIBSSH2_REALLOC_FUNC((*realloc));  // 重新分配内存回调函数
    LIBSSH2_FREE_FUNC((*free));  // 释放内存回调函数

    /* 其他回调 */
    LIBSSH2_IGNORE_FUNC((*ssh_msg_ignore));  // 忽略消息回调函数
    LIBSSH2_DEBUG_FUNC((*ssh_msg_debug));  // 调试消息回调函数
    LIBSSH2_DISCONNECT_FUNC((*ssh_msg_disconnect));  // 断开消息回调函数
    LIBSSH2_MACERROR_FUNC((*macerror));  // MAC 错误回调函数
    LIBSSH2_X11_OPEN_FUNC((*x11));  // X11 打开回调函数
    LIBSSH2_SEND_FUNC((*send));  // 发送回调函数
    LIBSSH2_RECV_FUNC((*recv));  // 接收回调函数

    /* 方法偏好 -- NULL 表示 "加载顺序" */
    char *kex_prefs;  // 密钥交换偏好
    char *hostkey_prefs;  // 主机密钥偏好

    int state;  // 状态

    /* 标志选项 */
    struct flags flag;  // 标志位

    /* 协商的密钥交换方法 */
    const LIBSSH2_KEX_METHOD *kex;  // 密钥交换方法
    unsigned int burn_optimistic_kexinit:1;  // 乐观的 kexinit 标志

    unsigned char *session_id;  // 会话 ID
    uint32_t session_id_len;  // 会话 ID 长度

    /* 如果请求阻塞 API 行为，则设置为 TRUE */
    int api_block_mode;  // API 阻塞模式
};
    # 用于指定阻塞 API 行为时的超时时间
    long api_timeout;

    # 服务器的公钥
    const LIBSSH2_HOSTKEY_METHOD *hostkey;
    void *server_hostkey_abstract;

    # 在服务器模式下使用 libssh2_session_hostkey() 设置
    # 或在客户端模式下从服务器中读取（例如在 KEXDH_INIT 中）
    unsigned char *server_hostkey;
    uint32_t server_hostkey_len;
#if LIBSSH2_MD5
    // 如果支持 MD5，则定义服务器主机密钥的 MD5 值和有效性标志
    unsigned char server_hostkey_md5[MD5_DIGEST_LENGTH];
    int server_hostkey_md5_valid;
#endif                          /* ! LIBSSH2_MD5 */
    // 定义服务器主机密钥的 SHA1 值和有效性标志
    unsigned char server_hostkey_sha1[SHA_DIGEST_LENGTH];
    int server_hostkey_sha1_valid;

    // 定义服务器主机密钥的 SHA256 值和有效性标志
    unsigned char server_hostkey_sha256[SHA256_DIGEST_LENGTH];
    int server_hostkey_sha256_valid;

    // 远程端点数据
    libssh2_endpoint_data remote;

    // 本地端点数据
    libssh2_endpoint_data local;

    // 输入数据链表，有时候接收到的数据包不是我们需要的数据包
    struct list_head packets;

    // 活跃的连接通道
    struct list_head channels;

    // 下一个通道
    uint32_t next_channel;

    // 监听器列表
    struct list_head listeners; /* list of LIBSSH2_LISTENER structs */

    // 实际的 I/O 套接字
    libssh2_socket_t socket_fd;
    int socket_state;
    int socket_block_directions;
    int socket_prev_blockstate; /* 在调用 libssh2_session_startup() 时存储套接字阻塞状态 */

    // 错误跟踪
    const char *err_msg;
    int err_code;
    int err_flags;

    // 用于数据包级别读取的结构成员
    struct transportpacket packet;
#ifdef LIBSSH2DEBUG
    // 要显示的调试/跟踪消息
    int showmask;
    // 用于显示跟踪消息的回调函数
    libssh2_trace_handler_func tracehandler;
    // 跟踪处理程序的上下文
    void *tracehandler_context;
#endif

    // 在 libssh2_banner_send() 中使用的状态变量
    libssh2_nonblocking_states banner_TxRx_state;
    char banner_TxRx_banner[256];
    ssize_t banner_TxRx_total_send;

    // 在 libssh2_kexinit() 中使用的状态变量
    libssh2_nonblocking_states kexinit_state;
    unsigned char *kexinit_data;
    size_t kexinit_data_len;

    // 在 libssh2_session_startup() 中使用的状态变量
    # 定义启动状态变量
    libssh2_nonblocking_states startup_state;
    # 启动数据的指针
    unsigned char *startup_data;
    # 启动数据的长度
    size_t startup_data_len;
    # SSH 服务的名称
    unsigned char startup_service[sizeof("ssh-userauth") + 5 - 1];
    # SSH 服务名称的长度
    size_t startup_service_length;
    # 启动请求状态
    packet_require_state_t startup_req_state;
    # 启动密钥交换状态
    key_exchange_state_t startup_key_state;

    # 在 libssh2_session_free() 中使用的状态变量
    libssh2_nonblocking_states free_state;

    # 在 libssh2_session_disconnect_ex() 中使用的状态变量
    libssh2_nonblocking_states disconnect_state;
    # 断开连接时的数据
    unsigned char disconnect_data[256 + 13];
    # 断开连接数据的长度
    size_t disconnect_data_len;

    # 在 libssh2_packet_read() 中使用的状态变量
    libssh2_nonblocking_states readPack_state;
    # 读取的数据是否加密
    int readPack_encrypted;

    # 在 libssh2_userauth_list() 中使用的状态变量
    libssh2_nonblocking_states userauth_list_state;
    # 用户认证列表的数据指针
    unsigned char *userauth_list_data;
    # 用户认证列表数据的长度
    size_t userauth_list_data_len;
    # 用户认证列表的数据包要求状态
    packet_requirev_state_t userauth_list_packet_requirev_state;

    # 在 libssh2_userauth_password_ex() 中使用的状态变量
    libssh2_nonblocking_states userauth_pswd_state;
    # 用户认证密码的数据指针
    unsigned char *userauth_pswd_data;
    # 用户认证密码的第一个字符
    unsigned char userauth_pswd_data0;
    # 用户认证密码数据的长度
    size_t userauth_pswd_data_len;
    # 用户认证密码的新密码
    char *userauth_pswd_newpw;
    # 用户认证密码新密码的长度
    int userauth_pswd_newpw_len;
    # 用户认证密码的数据包要求状态
    packet_requirev_state_t userauth_pswd_packet_requirev_state;

    # 在 libssh2_userauth_hostbased_fromfile_ex() 中使用的状态变量
    libssh2_nonblocking_states userauth_host_state;
    # 用户认证基于主机的数据指针
    unsigned char *userauth_host_data;
    # 用户认证基于主机数据的长度
    size_t userauth_host_data_len;
    # 用户认证基于主机的数据包
    unsigned char *userauth_host_packet;
    # 用户认证基于主机数据包的长度
    size_t userauth_host_packet_len;
    # 用户认证基于主机的方法
    unsigned char *userauth_host_method;
    # 用户认证基于主机方法的长度
    size_t userauth_host_method_len;
    # 用户认证基于主机的状态
    unsigned char *userauth_host_s;
    # 用户认证基于主机数据包的要求状态
    packet_requirev_state_t userauth_host_packet_requirev_state;

    # 在 libssh2_userauth_publickey_fromfile_ex() 中使用的状态变量
    libssh2_nonblocking_states userauth_pblc_state;
    # 用户认证公钥的数据指针
    unsigned char *userauth_pblc_data;
    # 用户认证公钥数据的长度
    size_t userauth_pblc_data_len;
    # 定义指向用户认证公钥数据包的指针
    unsigned char *userauth_pblc_packet;
    # 用户认证公钥数据包的长度
    size_t userauth_pblc_packet_len;
    # 定义指向用户认证公钥方法的指针
    unsigned char *userauth_pblc_method;
    # 用户认证公钥方法的长度
    size_t userauth_pblc_method_len;
    # 定义指向用户认证公钥的指针
    unsigned char *userauth_pblc_s;
    # 定义指向用户认证公钥的指针
    unsigned char *userauth_pblc_b;
    # 用户认证公钥数据包的状态
    packet_requirev_state_t userauth_pblc_packet_requirev_state;
    
    # libssh2_userauth_keyboard_interactive_ex() 中使用的状态变量
    libssh2_nonblocking_states userauth_kybd_state;
    # 用户认证键盘交互数据
    unsigned char *userauth_kybd_data;
    # 用户认证键盘交互数据的长度
    size_t userauth_kybd_data_len;
    # 用户认证键盘交互数据包
    unsigned char *userauth_kybd_packet;
    # 用户认证键盘交互数据包的长度
    size_t userauth_kybd_packet_len;
    # 用户认证键盘交互认证名称的长度
    unsigned int userauth_kybd_auth_name_len;
    # 用户认证键盘交互认证名称
    char *userauth_kybd_auth_name;
    # 用户认证键盘交互认证指令的长度
    unsigned userauth_kybd_auth_instruction_len;
    # 用户认证键盘交互认证指令
    char *userauth_kybd_auth_instruction;
    # 用户认证键盘交互提示的数量
    unsigned int userauth_kybd_num_prompts;
    # 用户认证键盘交互认证失败标志
    int userauth_kybd_auth_failure;
    # 用户认证键盘交互提示数组
    LIBSSH2_USERAUTH_KBDINT_PROMPT *userauth_kybd_prompts;
    # 用户认证键盘交互响应数组
    LIBSSH2_USERAUTH_KBDINT_RESPONSE *userauth_kybd_responses;
    # 用户认证键盘交互数据包的状态
    packet_requirev_state_t userauth_kybd_packet_requirev_state;
    
    # libssh2_channel_open_ex() 中使用的状态变量
    libssh2_nonblocking_states open_state;
    # libssh2_channel_open_ex() 中使用的数据包状态
    packet_requirev_state_t open_packet_requirev_state;
    # libssh2_channel_open_ex() 中使用的通道
    LIBSSH2_CHANNEL *open_channel;
    # libssh2_channel_open_ex() 中使用的数据包
    unsigned char *open_packet;
    # libssh2_channel_open_ex() 中使用的数据包长度
    size_t open_packet_len;
    # libssh2_channel_open_ex() 中使用的数据
    unsigned char *open_data;
    # libssh2_channel_open_ex() 中使用的数据长度
    size_t open_data_len;
    # libssh2_channel_open_ex() 中使用的本地通道
    uint32_t open_local_channel;
    
    # libssh2_channel_direct_tcpip_ex() 中使用的状态变量
    libssh2_nonblocking_states direct_state;
    # libssh2_channel_direct_tcpip_ex() 中使用的消息
    unsigned char *direct_message;
    # libssh2_channel_direct_tcpip_ex() 中使用的主机长度
    size_t direct_host_len;
    # libssh2_channel_direct_tcpip_ex() 中使用的源主机长度
    size_t direct_shost_len;
    # libssh2_channel_direct_tcpip_ex() 中使用的消息长度
    size_t direct_message_len;
    
    # libssh2_channel_forward_listen_ex() 中使用的状态变量
    libssh2_nonblocking_states fwdLstn_state;
    # libssh2_channel_forward_listen_ex() 中使用的数据包
    unsigned char *fwdLstn_packet;
    # libssh2_channel_forward_listen_ex() 中使用的主机长度
    uint32_t fwdLstn_host_len;
    # libssh2_channel_forward_listen_ex() 中使用的数据包长度
    uint32_t fwdLstn_packet_len;
    # libssh2_channel_forward_listen_ex() 中使用的数据包状态
    packet_requirev_state_t fwdLstn_packet_requirev_state;
    
    # libssh2_publickey_init() 中使用的状态变量
    libssh2_nonblocking_states pkeyInit_state;
    # 初始化公钥指针
    LIBSSH2_PUBLICKEY *pkeyInit_pkey;
    # 初始化通道指针
    LIBSSH2_CHANNEL *pkeyInit_channel;
    # 初始化数据指针
    unsigned char *pkeyInit_data;
    # 初始化数据长度
    size_t pkeyInit_data_len;
    # 初始化缓冲区，用于存储数据
    /* 19 = packet_len(4) + version_len(4) + "version"(7) + version_num(4) */
    unsigned char pkeyInit_buffer[19];
    # 已发送的缓冲区数据长度
    size_t pkeyInit_buffer_sent; /* how much of buffer that has been sent */

    # 用于libssh2_packet_add()函数的状态变量
    libssh2_nonblocking_states packAdd_state;
    # 在EAGAIN状态下保持通道
    LIBSSH2_CHANNEL *packAdd_channelp; /* keeper of the channel during EAGAIN states */
    # 用于packet_queue_listener_state_t的状态变量
    packet_queue_listener_state_t packAdd_Qlstn_state;
    # 用于packet_x11_open_state_t的状态变量
    packet_x11_open_state_t packAdd_x11open_state;

    # 用于fullpacket()函数的状态变量
    libssh2_nonblocking_states fullpacket_state;
    # MAC状态
    int fullpacket_macstate;
    # 负载长度
    size_t fullpacket_payload_len;
    # 数据包类型
    int fullpacket_packet_type;

    # 用于libssh2_sftp_init()函数的状态变量
    libssh2_nonblocking_states sftpInit_state;
    # 初始化SFTP指针
    LIBSSH2_SFTP *sftpInit_sftp;
    # 初始化通道指针
    LIBSSH2_CHANNEL *sftpInit_channel;
    # SFTP缓冲区，用于存储数据
    unsigned char sftpInit_buffer[9];   /* sftp_header(5){excludes request_id} + version_id(4) */
    # 已发送的SFTP缓冲区数据长度
    int sftpInit_sent; /* number of bytes from the buffer that have been sent */

    # 用于libssh2_scp_recv() / libssh_scp_recv2()函数的状态变量
    libssh2_nonblocking_states scpRecv_state;
    # SCP命令指针
    unsigned char *scpRecv_command;
    # SCP命令长度
    size_t scpRecv_command_len;
    # SCP响应缓冲区，用于存储数据
    unsigned char scpRecv_response[LIBSSH2_SCP_RESPONSE_BUFLEN];
    # SCP响应数据长度
    size_t scpRecv_response_len;
    # SCP模式
    long scpRecv_mode;
#if defined(HAVE_LONGLONG) && defined(HAVE_STRTOLL)
    /* 如果定义了 HAVE_LONGLONG 和 HAVE_STRTOLL，则使用 long long 类型和 strtoll 函数来解析数字 */
    long long scpRecv_size;
#define scpsize_strtol strtoll
#elif defined(HAVE_STRTOI64)
    /* 如果定义了 HAVE_STRTOI64，则使用 __int64 类型和 _strtoi64 函数来解析数字 */
    __int64 scpRecv_size;
#define scpsize_strtol _strtoi64
#else
    /* 否则使用 long 类型和 strtol 函数来解析数字 */
    long scpRecv_size;
#define scpsize_strtol strtol
#endif
    /* 定义变量 */
    long scpRecv_mtime;
    long scpRecv_atime;
    LIBSSH2_CHANNEL *scpRecv_channel;

    /* libssh2_scp_send_ex() 中使用的状态变量 */
    libssh2_nonblocking_states scpSend_state;
    unsigned char *scpSend_command;
    size_t scpSend_command_len;
    unsigned char scpSend_response[LIBSSH2_SCP_RESPONSE_BUFLEN];
    size_t scpSend_response_len;
    LIBSSH2_CHANNEL *scpSend_channel;

    /* keepalive.c 中使用的保持连接变量 */
    int keepalive_interval;
    int keepalive_want_reply;
    time_t keepalive_last_sent;
};

/* session.state 的位标志 */
#define LIBSSH2_STATE_EXCHANGING_KEYS   0x00000001
#define LIBSSH2_STATE_NEWKEYS           0x00000002
#define LIBSSH2_STATE_AUTHENTICATED     0x00000004
#define LIBSSH2_STATE_KEX_ACTIVE        0x00000008

/* session.flag 的辅助宏 */
#ifdef MSG_NOSIGNAL
#define LIBSSH2_SOCKET_SEND_FLAGS(session)              \
    (((session)->flag.sigpipe) ? 0 : MSG_NOSIGNAL)
#define LIBSSH2_SOCKET_RECV_FLAGS(session)              \
    (((session)->flag.sigpipe) ? 0 : MSG_NOSIGNAL)
#else
/* 如果 MSG_NOSIGNAL 没有定义，则无法阻止 SIGPIPE 信号 */
#define LIBSSH2_SOCKET_SEND_FLAGS(session)      0
#define LIBSSH2_SOCKET_RECV_FLAGS(session)      0
#endif

/* --------- */

/* libssh2 可扩展的 SSH API，最终希望通过 .so/.dll 文件加载额外的方法 */

struct _LIBSSH2_KEX_METHOD
{
    const char *name;

    /* 密钥交换，填充 session->* 并在成功时返回 0，错误时返回非 0 */
    int (*exchange_keys) (LIBSSH2_SESSION * session,
                          key_exchange_state_low_t * key_state);

    long flags;
};

struct _LIBSSH2_HOSTKEY_METHOD
{
    const char *name;
    # 定义一个无符号长整型变量 hash_len

    # 定义一个函数指针 init，用于初始化会话和主机密钥数据
    int (*init) (LIBSSH2_SESSION * session, const unsigned char *hostkey_data,
                 size_t hostkey_data_len, void **abstract);

    # 定义一个函数指针 initPEM，用于从 PEM 文件初始化会话和私钥数据
    int (*initPEM) (LIBSSH2_SESSION * session, const char *privkeyfile,
                    unsigned const char *passphrase, void **abstract);

    # 定义一个函数指针 initPEMFromMemory，用于从内存中的 PEM 数据初始化会话和私钥数据
    int (*initPEMFromMemory) (LIBSSH2_SESSION * session,
                              const char *privkeyfiledata,
                              size_t privkeyfiledata_len,
                              unsigned const char *passphrase,
                              void **abstract);

    # 定义一个函数指针 sig_verify，用于验证签名
    int (*sig_verify) (LIBSSH2_SESSION * session, const unsigned char *sig,
                       size_t sig_len, const unsigned char *m,
                       size_t m_len, void **abstract);

    # 定义一个函数指针 signv，用于对数据进行签名
    int (*signv) (LIBSSH2_SESSION * session, unsigned char **signature,
                  size_t *signature_len, int veccount,
                  const struct iovec datavec[], void **abstract);

    # 定义一个函数指针 encrypt，用于对数据进行加密
    int (*encrypt) (LIBSSH2_SESSION * session, unsigned char **dst,
                    size_t *dst_len, const unsigned char *src,
                    size_t src_len, void **abstract);

    # 定义一个函数指针 dtor，用于销毁会话
    int (*dtor) (LIBSSH2_SESSION * session, void **abstract);
};

// 定义加密方法的结构体
struct _LIBSSH2_CRYPT_METHOD
{
    const char *name; // 加密方法的名称
    const char *pem_annotation; // PEM 注释
    int blocksize; // 块大小

    /* iv and key sizes (-1 for variable length) */
    int iv_len; // 初始化向量长度
    int secret_len; // 密钥长度

    long flags; // 标志

    // 初始化加密方法
    int (*init) (LIBSSH2_SESSION * session,
                 const LIBSSH2_CRYPT_METHOD * method, unsigned char *iv,
                 int *free_iv, unsigned char *secret, int *free_secret,
                 int encrypt, void **abstract);
    // 加密数据
    int (*crypt) (LIBSSH2_SESSION * session, unsigned char *block,
                  size_t blocksize, void **abstract);
    // 析构函数
    int (*dtor) (LIBSSH2_SESSION * session, void **abstract);

      _libssh2_cipher_type(algo);
};

// 定义压缩方法的结构体
struct _LIBSSH2_COMP_METHOD
{
    const char *name; // 压缩方法的名称
    int compress; /* 1 if it does compress, 0 if it doesn't */ // 是否支持压缩
    int use_in_auth; /* 1 if compression should be used in userauth */ // 是否在用户认证中使用压缩
    // 初始化压缩方法
    int (*init) (LIBSSH2_SESSION *session, int compress, void **abstract);
    // 压缩数据
    int (*comp) (LIBSSH2_SESSION *session,
                 unsigned char *dest,
                 size_t *dest_len,
                 const unsigned char *src,
                 size_t src_len,
                 void **abstract);
    // 解压数据
    int (*decomp) (LIBSSH2_SESSION *session,
                   unsigned char **dest,
                   size_t *dest_len,
                   size_t payload_limit,
                   const unsigned char *src,
                   size_t src_len,
                   void **abstract);
    // 析构函数
    int (*dtor) (LIBSSH2_SESSION * session, int compress, void **abstract);
};

#ifdef LIBSSH2DEBUG
// 调试函数
void _libssh2_debug(LIBSSH2_SESSION * session, int context, const char *format,
                    ...);
#else
#if (defined(__STDC_VERSION__) && (__STDC_VERSION__ >= 199901L)) ||     \
    defined(__GNUC__)
/* C99 supported and also by older GCC */
#define _libssh2_debug(x,y,z,...) do {} while (0)
#else
/* no gcc and not C99, do static and hopefully inline */
static inline void
# 调试函数，用于输出调试信息
_libssh2_debug(LIBSSH2_SESSION * session, int context, const char *format, ...)
{
    // 忽略参数 session
    (void)session;
    // 忽略参数 context
    (void)context;
    // 忽略参数 format
    (void)format;
}
#endif
#endif

// 定义套接字状态常量
#define LIBSSH2_SOCKET_UNKNOWN                   1
#define LIBSSH2_SOCKET_CONNECTED                 0
#define LIBSSH2_SOCKET_DISCONNECTED             -1

// 初始数据包状态，在 MAC 检查之前
#define LIBSSH2_MAC_UNCONFIRMED                  1
// 当 MAC 类型为 "none" 时（协议初始化阶段），所有数据包被视为 "confirmed"
#define LIBSSH2_MAC_CONFIRMED                    0
// 发生严重错误
#define LIBSSH2_MAC_INVALID                     -1

// _libssh2_error_flags 的标志
// 错误消息在堆上分配
#define LIBSSH2_ERR_FLAG_DUP                     1

// SSH 数据包类型 -- 由互联网草案定义
// 传输层
#define SSH_MSG_DISCONNECT                          1
#define SSH_MSG_IGNORE                              2
#define SSH_MSG_UNIMPLEMENTED                       3
#define SSH_MSG_DEBUG                               4
#define SSH_MSG_SERVICE_REQUEST                     5
#define SSH_MSG_SERVICE_ACCEPT                      6

#define SSH_MSG_KEXINIT                             20
#define SSH_MSG_NEWKEYS                             21

// diffie-hellman-group1-sha1
#define SSH_MSG_KEXDH_INIT                          30
#define SSH_MSG_KEXDH_REPLY                         31

// diffie-hellman-group-exchange-sha1 和 diffie-hellman-group-exchange-sha256
#define SSH_MSG_KEX_DH_GEX_REQUEST_OLD              30
#define SSH_MSG_KEX_DH_GEX_REQUEST                  34
#define SSH_MSG_KEX_DH_GEX_GROUP                    31
#define SSH_MSG_KEX_DH_GEX_INIT                     32
#define SSH_MSG_KEX_DH_GEX_REPLY                    33

// ecdh
#define SSH2_MSG_KEX_ECDH_INIT                      30
#define SSH2_MSG_KEX_ECDH_REPLY                     31

// 用户认证
#define SSH_MSG_USERAUTH_REQUEST                    50
# 定义 SSH 用户认证失败消息的消息类型
#define SSH_MSG_USERAUTH_FAILURE                    51
# 定义 SSH 用户认证成功消息的消息类型
#define SSH_MSG_USERAUTH_SUCCESS                    52
# 定义 SSH 用户认证横幅消息的消息类型
#define SSH_MSG_USERAUTH_BANNER                     53

# "public key" 方法的消息类型
#define SSH_MSG_USERAUTH_PK_OK                      60
# "password" 方法的消息类型
#define SSH_MSG_USERAUTH_PASSWD_CHANGEREQ           60
# "keyboard-interactive" 方法的消息类型
#define SSH_MSG_USERAUTH_INFO_REQUEST               60
# 用户认证信息响应消息的消息类型
#define SSH_MSG_USERAUTH_INFO_RESPONSE              61

# 通道相关消息类型
#define SSH_MSG_GLOBAL_REQUEST                      80
#define SSH_MSG_REQUEST_SUCCESS                     81
#define SSH_MSG_REQUEST_FAILURE                     82

#define SSH_MSG_CHANNEL_OPEN                        90
#define SSH_MSG_CHANNEL_OPEN_CONFIRMATION           91
#define SSH_MSG_CHANNEL_OPEN_FAILURE                92
#define SSH_MSG_CHANNEL_WINDOW_ADJUST               93
#define SSH_MSG_CHANNEL_DATA                        94
#define SSH_MSG_CHANNEL_EXTENDED_DATA               95
#define SSH_MSG_CHANNEL_EOF                         96
#define SSH_MSG_CHANNEL_CLOSE                       97
#define SSH_MSG_CHANNEL_REQUEST                     98
#define SSH_MSG_CHANNEL_SUCCESS                     99
#define SSH_MSG_CHANNEL_FAILURE                     100

# 在 SSH_MSG_CHANNEL_OPEN_FAILURE 消息中返回的错误代码
# (参见 RFC4254)
#define SSH_OPEN_ADMINISTRATIVELY_PROHIBITED 1
#define SSH_OPEN_CONNECT_FAILED              2
#define SSH_OPEN_UNKNOWN_CHANNELTYPE         3
#define SSH_OPEN_RESOURCE_SHORTAGE           4

# 从 libssh2_socket_t 接收数据的函数
ssize_t _libssh2_recv(libssh2_socket_t socket, void *buffer,
                      size_t length, int flags, void **abstract);
# 向 libssh2_socket_t 发送数据的函数
ssize_t _libssh2_send(libssh2_socket_t socket, const void *buffer,
                      size_t length, int flags, void **abstract);

# 用于等待更多数据到达时使用的通用超时时间（以秒为单位）
#define LIBSSH2_READ_TIMEOUT 60 /* generic timeout in seconds used when
                                   waiting for more data to arrive */
# 定义 libssh2_kex_exchange 函数，用于执行密钥交换
int _libssh2_kex_exchange(LIBSSH2_SESSION * session, int reexchange,
                          key_exchange_state_t * state);

# 公开加密方法结构体的接口
const LIBSSH2_CRYPT_METHOD **libssh2_crypt_methods(void);
# 公开主机密钥方法结构体的接口
const LIBSSH2_HOSTKEY_METHOD **libssh2_hostkey_methods(void);

# 在 misc.c 文件中定义 _libssh2_bcrypt_pbkdf 函数，用于执行 PBKDF2 密码推导函数
int _libssh2_bcrypt_pbkdf(const char *pass,
                          size_t passlen,
                          const uint8_t *salt,
                          size_t saltlen,
                          uint8_t *key,
                          size_t keylen,
                          unsigned int rounds);

# 在 pem.c 文件中定义 _libssh2_pem_parse 函数，用于解析 PEM 格式的数据
int _libssh2_pem_parse(LIBSSH2_SESSION * session,
                       const char *headerbegin,
                       const char *headerend,
                       const unsigned char *passphrase,
                       FILE * fp, unsigned char **data, unsigned int *datalen);
# 在 pem.c 文件中定义 _libssh2_pem_parse_memory 函数，用于解析内存中的 PEM 格式数据
int _libssh2_pem_parse_memory(LIBSSH2_SESSION * session,
                              const char *headerbegin,
                              const char *headerend,
                              const char *filedata, size_t filedata_len,
                              unsigned char **data, unsigned int *datalen);

# 在 pem.c 文件中定义 _libssh2_openssh_pem_parse 函数，用于解析 OpenSSL 格式的密钥
int
_libssh2_openssh_pem_parse(LIBSSH2_SESSION * session,
                           const unsigned char *passphrase,
                           FILE * fp, struct string_buf **decrypted_buf);
# 在 pem.c 文件中定义 _libssh2_openssh_pem_parse_memory 函数，用于解析内存中的 OpenSSL 格式密钥
int
_libssh2_openssh_pem_parse_memory(LIBSSH2_SESSION * session,
                                  const unsigned char *passphrase,
                                  const char *filedata, size_t filedata_len,
                                  struct string_buf **decrypted_buf);

# 在 pem.c 文件中定义 _libssh2_pem_decode_sequence 函数，用于解析 PEM 格式数据中的序列
int _libssh2_pem_decode_sequence(unsigned char **data, unsigned int *datalen);
# 在 pem.c 文件中定义 _libssh2_pem_decode_integer 函数，用于解析 PEM 格式数据中的整数

int _libssh2_pem_decode_integer(unsigned char **data, unsigned int *datalen,
                                unsigned char **i, unsigned int *ilen);

# 在 global.c 文件中
// 如果需要的话初始化 libssh2
void _libssh2_init_if_needed(void);

// 定义一个宏，用于计算数组的大小
#define ARRAY_SIZE(a) (sizeof ((a)) / sizeof ((a)[0]))

// 根据不同的编译器定义 libssh2_int64_t 类型在 printf() 中的输出格式
#if defined(__BORLANDC__) || defined(_MSC_VER) || defined(__MINGW32__)
#define LIBSSH2_INT64_T_FORMAT "I64d"
#else
#define LIBSSH2_INT64_T_FORMAT "lld"
#endif

// 在 Windows 中，默认的文件模式是文本模式，但应用程序可以覆盖它。因此我们需要显式指定它
#if defined(WIN32) || defined(MSDOS)
#define FOPEN_READTEXT "rt"
#define FOPEN_WRITETEXT "wt"
#define FOPEN_APPENDTEXT "at"
// 在 Cygwin 中，当 WIN32 未定义时，有特定的行为需要处理
#elif defined(__CYGWIN__)
// 为了让输出具有 LF 的行结束，并与其他 Cygwin 实用程序兼容，我们需要指定 'w' 模式
#define FOPEN_READTEXT "rt"
#define FOPEN_WRITETEXT "w"
#define FOPEN_APPENDTEXT "a"
// 其他情况下，使用默认的读写模式
#else
#define FOPEN_READTEXT "r"
#define FOPEN_WRITETEXT "w"
#define FOPEN_APPENDTEXT "a"
#endif

#endif /* __LIBSSH2_PRIV_H */
```