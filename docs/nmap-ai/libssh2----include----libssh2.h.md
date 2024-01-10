# `nmap\libssh2\include\libssh2.h`

```
# 版权声明和许可声明
# 作者和贡献者的版权声明
# 允许在源代码和二进制形式下重新分发和使用，但需要满足一定条件
# 在源代码的重新分发中需要保留版权声明、条件列表和免责声明
# 在二进制形式的重新分发中需要在文档和/或其他提供的材料中复制版权声明、条件列表和免责声明
# 未经特定事先书面许可，不得使用版权所有者或其他贡献者的名称来认可或推广基于此软件的产品
# 本软件由版权所有者和贡献者按原样提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的适用性的担保
# 无论在任何情况下，版权所有者或贡献者均不对任何直接、间接、偶然、特殊、惩罚性或后果性的损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）承担责任，无论是合同责任、严格责任还是侵权行为（包括疏忽或其他方式）的任何理论，即使已被告知可能发生此类损害
# libssh2 项目及其贡献者的版权声明
#ifndef LIBSSH2_H
#define LIBSSH2_H 1
# 定义 libssh2 版权声明
#define LIBSSH2_COPYRIGHT "2004-2019 The libssh2 project and its contributors."
/* 定义 libssh2 的版本号为 "1.10.0" */
#define LIBSSH2_VERSION "1.10.0"

/* 定义 libssh2 的主版本号为 1 */
#define LIBSSH2_VERSION_MAJOR 1
/* 定义 libssh2 的次版本号为 10 */
#define LIBSSH2_VERSION_MINOR 10
/* 定义 libssh2 的修订版本号为 0 */
#define LIBSSH2_VERSION_PATCH 0

/* 定义 libssh2 的版本号的数字形式为 0x010a00 */
#define LIBSSH2_VERSION_NUM 0x010a00

/*
 * 定义 libssh2 源代码包创建的日期和时间
 * 日期格式应该遵循 "Mon Feb 12 11:35:33 UTC 2007" 的模板
 */
#define LIBSSH2_TIMESTAMP "Sun 29 Aug 2021 08:37:50 PM UTC"

#ifndef RC_INVOKED

#ifdef __cplusplus
extern "C" {
#endif
#ifdef _WIN32
# include <basetsd.h>
# include <winsock2.h>
#endif

#include <stddef.h>
#include <string.h>
#include <sys/stat.h>
#include <sys/types.h>

/* 从 CFLAGS 或调用应用程序中允许使用替代的 API 前缀 */
#ifndef LIBSSH2_API
# ifdef LIBSSH2_WIN32
#  ifdef _WINDLL
#   ifdef LIBSSH2_LIBRARY
#    定义 LIBSSH2_API 为 __declspec(dllexport)
#   else
#    定义 LIBSSH2_API 为 __declspec(dllimport)
#ifdef LIBSSH2_LIBRARY
#   define LIBSSH2_API __declspec(dllexport)
#  else
#   define LIBSSH2_API
#  endif
# else /* !LIBSSH2_WIN32 */
#  define LIBSSH2_API
# endif /* LIBSSH2_WIN32 */
#endif /* LIBSSH2_API */

#ifdef HAVE_SYS_UIO_H
# include <sys/uio.h>
#endif

#if (defined(NETWARE) && !defined(__NOVELL_LIBC__))
# include <sys/bsdskt.h>
typedef unsigned char uint8_t;
typedef unsigned short int uint16_t;
typedef unsigned int uint32_t;
typedef int int32_t;
typedef unsigned long long uint64_t;
typedef long long int64_t;
#endif

#ifdef _MSC_VER
typedef unsigned char uint8_t;
typedef unsigned short int uint16_t;
typedef unsigned int uint32_t;
typedef __int32 int32_t;
typedef __int64 int64_t;
typedef unsigned __int64 uint64_t;
typedef unsigned __int64 libssh2_uint64_t;
typedef __int64 libssh2_int64_t;
#if (!defined(HAVE_SSIZE_T) && !defined(ssize_t))
typedef SSIZE_T ssize_t;
#define HAVE_SSIZE_T
#endif
#else
#include <stdint.h>
typedef unsigned long long libssh2_uint64_t;
typedef long long libssh2_int64_t;
#endif

#ifdef WIN32
typedef SOCKET libssh2_socket_t;
#define LIBSSH2_INVALID_SOCKET INVALID_SOCKET
#else /* !WIN32 */
typedef int libssh2_socket_t;
#define LIBSSH2_INVALID_SOCKET -1
#endif /* WIN32 */

/*
 * Determine whether there is small or large file support on windows.
 */

#if defined(_MSC_VER) && !defined(_WIN32_WCE)
#  if (_MSC_VER >= 900) && (_INTEGRAL_MAX_BITS >= 64)
#    define LIBSSH2_USE_WIN32_LARGE_FILES
#  else
#    define LIBSSH2_USE_WIN32_SMALL_FILES
#  endif
#endif

#if defined(__MINGW32__) && !defined(LIBSSH2_USE_WIN32_LARGE_FILES)
#  define LIBSSH2_USE_WIN32_LARGE_FILES
#endif

#if defined(__WATCOMC__) && !defined(LIBSSH2_USE_WIN32_LARGE_FILES)
#  define LIBSSH2_USE_WIN32_LARGE_FILES
#endif

#if defined(__POCC__)
#  undef LIBSSH2_USE_WIN32_LARGE_FILES
#endif

#if defined(_WIN32) && !defined(LIBSSH2_USE_WIN32_LARGE_FILES) && \
    !defined(LIBSSH2_USE_WIN32_SMALL_FILES)
#  define LIBSSH2_USE_WIN32_SMALL_FILES
#endif

/*
 * Large file (>2Gb) support using WIN32 functions.
 */
#ifdef LIBSSH2_USE_WIN32_LARGE_FILES
#  include <io.h>
#  include <sys/types.h>
#  include <sys/stat.h>
#  define LIBSSH2_STRUCT_STAT_SIZE_FORMAT    "%I64d"
typedef struct _stati64 libssh2_struct_stat;
typedef __int64 libssh2_struct_stat_size;
#endif

/*
 * Small file (<2Gb) support using WIN32 functions.
 */

#ifdef LIBSSH2_USE_WIN32_SMALL_FILES
#  include <sys/types.h>
#  include <sys/stat.h>
#  ifndef _WIN32_WCE
#    define LIBSSH2_STRUCT_STAT_SIZE_FORMAT    "%d"
typedef struct _stat libssh2_struct_stat;
typedef off_t libssh2_struct_stat_size;
#  endif
#endif

#ifndef LIBSSH2_STRUCT_STAT_SIZE_FORMAT
#  ifdef __VMS
/* We have to roll our own format here because %z is a C99-ism we don't
   have. */
#    if __USE_OFF64_T || __USING_STD_STAT
#      define LIBSSH2_STRUCT_STAT_SIZE_FORMAT      "%Ld"
#    else
#      define LIBSSH2_STRUCT_STAT_SIZE_FORMAT      "%d"
#    endif
#  else
#    define LIBSSH2_STRUCT_STAT_SIZE_FORMAT      "%zd"
#  endif
typedef struct stat libssh2_struct_stat;
typedef off_t libssh2_struct_stat_size;
#endif

/* Part of every banner, user specified or not */
#define LIBSSH2_SSH_BANNER                  "SSH-2.0-libssh2_" LIBSSH2_VERSION

#define LIBSSH2_SSH_DEFAULT_BANNER            LIBSSH2_SSH_BANNER
#define LIBSSH2_SSH_DEFAULT_BANNER_WITH_CRLF  LIBSSH2_SSH_DEFAULT_BANNER "\r\n"

/* Default generate and safe prime sizes for
   diffie-hellman-group-exchange-sha1 */
#define LIBSSH2_DH_GEX_MINGROUP     2048
#define LIBSSH2_DH_GEX_OPTGROUP     4096
#define LIBSSH2_DH_GEX_MAXGROUP     8192

#define LIBSSH2_DH_MAX_MODULUS_BITS 16384

/* Defaults for pty requests */
#define LIBSSH2_TERM_WIDTH      80
#define LIBSSH2_TERM_HEIGHT     24
#define LIBSSH2_TERM_WIDTH_PX   0
#define LIBSSH2_TERM_HEIGHT_PX  0

/* 1/4 second */
#define LIBSSH2_SOCKET_POLL_UDELAY      250000
/* 0.25 * 120 == 30 seconds */
#define LIBSSH2_SOCKET_POLL_MAXLOOPS    120

/* Maximum size to allow a payload to compress to, plays it safe by falling
   short of spec limits */
/* 定义最大压缩包大小 */
#define LIBSSH2_PACKET_MAXCOMP      32000

/* 定义最大解压缩包大小，比规范要求的更大 */
#define LIBSSH2_PACKET_MAXDECOMP    40000

/* 定义最大入站压缩包大小，比规范限制更大 */
#define LIBSSH2_PACKET_MAXPAYLOAD   40000

/* 分配内存的回调函数 */
#define LIBSSH2_ALLOC_FUNC(name)   void *name(size_t count, void **abstract)
#define LIBSSH2_REALLOC_FUNC(name) void *name(void *ptr, size_t count, \
                                              void **abstract)
#define LIBSSH2_FREE_FUNC(name)    void name(void *ptr, void **abstract)

/* 定义用户交互式键盘认证提示结构体 */
typedef struct _LIBSSH2_USERAUTH_KBDINT_PROMPT
{
    char *text;
    unsigned int length;
    unsigned char echo;
} LIBSSH2_USERAUTH_KBDINT_PROMPT;

/* 定义用户交互式键盘认证响应结构体 */
typedef struct _LIBSSH2_USERAUTH_KBDINT_RESPONSE
{
    char *text;
    unsigned int length;
} LIBSSH2_USERAUTH_KBDINT_RESPONSE;

/* 'publickey' 认证回调函数 */
#define LIBSSH2_USERAUTH_PUBLICKEY_SIGN_FUNC(name) \
  int name(LIBSSH2_SESSION *session, unsigned char **sig, size_t *sig_len, \
           const unsigned char *data, size_t data_len, void **abstract)

/* 'keyboard-interactive' 认证回调函数 */
#define LIBSSH2_USERAUTH_KBDINT_RESPONSE_FUNC(name_) \
 void name_(const char *name, int name_len, const char *instruction, \
            int instruction_len, int num_prompts, \
            const LIBSSH2_USERAUTH_KBDINT_PROMPT *prompts,              \
            LIBSSH2_USERAUTH_KBDINT_RESPONSE *responses, void **abstract)

/* 特殊 SSH 数据包的回调函数 */
#define LIBSSH2_IGNORE_FUNC(name) \
 void name(LIBSSH2_SESSION *session, const char *message, int message_len, \
           void **abstract)

#define LIBSSH2_DEBUG_FUNC(name) \
 void name(LIBSSH2_SESSION *session, int always_display, const char *message, \
           int message_len, const char *language, int language_len, \
           void **abstract)
# 定义一个宏，用于声明一个断开连接的回调函数
#define LIBSSH2_DISCONNECT_FUNC(name) \
 void name(LIBSSH2_SESSION *session, int reason, const char *message, \
           int message_len, const char *language, int language_len, \
           void **abstract)

# 定义一个宏，用于声明一个密码更改请求的回调函数
#define LIBSSH2_PASSWD_CHANGEREQ_FUNC(name) \
 void name(LIBSSH2_SESSION *session, char **newpw, int *newpw_len, \
           void **abstract)

# 定义一个宏，用于声明一个 MAC 错误的回调函数
#define LIBSSH2_MACERROR_FUNC(name) \
 int name(LIBSSH2_SESSION *session, const char *packet, int packet_len, \
          void **abstract)

# 定义一个宏，用于声明一个 X11 连接打开的回调函数
#define LIBSSH2_X11_OPEN_FUNC(name) \
 void name(LIBSSH2_SESSION *session, LIBSSH2_CHANNEL *channel, \
           const char *shost, int sport, void **abstract)

# 定义一个宏，用于声明一个通道关闭的回调函数
#define LIBSSH2_CHANNEL_CLOSE_FUNC(name) \
  void name(LIBSSH2_SESSION *session, void **session_abstract, \
            LIBSSH2_CHANNEL *channel, void **channel_abstract)

# 定义一个宏，用于声明一个接收数据的回调函数
#define LIBSSH2_RECV_FUNC(name)                                         \
    ssize_t name(libssh2_socket_t socket,                               \
                 void *buffer, size_t length,                           \
                 int flags, void **abstract)
                 
# 定义一个宏，用于声明一个发送数据的回调函数
#define LIBSSH2_SEND_FUNC(name)                                         \
    ssize_t name(libssh2_socket_t socket,                               \
                 const void *buffer, size_t length,                     \
                 int flags, void **abstract)

# 定义 libssh2_session_callback_set() 函数的常量
#define LIBSSH2_CALLBACK_IGNORE             0
#define LIBSSH2_CALLBACK_DEBUG              1
#define LIBSSH2_CALLBACK_DISCONNECT         2
#define LIBSSH2_CALLBACK_MACERROR           3
#define LIBSSH2_CALLBACK_X11                4
#define LIBSSH2_CALLBACK_SEND               5
#define LIBSSH2_CALLBACK_RECV               6

# 定义 libssh2_session_method_pref() 函数的常量
#define LIBSSH2_METHOD_KEX          0
#define LIBSSH2_METHOD_HOSTKEY      1
#define LIBSSH2_METHOD_CRYPT_CS     2
#define LIBSSH2_METHOD_CRYPT_SC     3
#define LIBSSH2_METHOD_MAC_CS       4
# 定义 SSH2 方法的 MAC_SC（消息认证码），COMP_CS（压缩算法），COMP_SC（压缩算法），LANG_CS（语言），LANG_SC（语言）
#define LIBSSH2_METHOD_MAC_SC       5
#define LIBSSH2_METHOD_COMP_CS      6
#define LIBSSH2_METHOD_COMP_SC      7
#define LIBSSH2_METHOD_LANG_CS      8
#define LIBSSH2_METHOD_LANG_SC      9

# 定义 SSH2 的标志
#define LIBSSH2_FLAG_SIGPIPE        1  # SIGPIPE 标志
#define LIBSSH2_FLAG_COMPRESS       2  # 压缩标志

# 定义 SSH2 会话、通道、监听器、已知主机、代理的结构体
typedef struct _LIBSSH2_SESSION                     LIBSSH2_SESSION;
typedef struct _LIBSSH2_CHANNEL                     LIBSSH2_CHANNEL;
typedef struct _LIBSSH2_LISTENER                    LIBSSH2_LISTENER;
typedef struct _LIBSSH2_KNOWNHOSTS                  LIBSSH2_KNOWNHOSTS;
typedef struct _LIBSSH2_AGENT                       LIBSSH2_AGENT;

# 定义 SSH2 的轮询文件描述符结构体
typedef struct _LIBSSH2_POLLFD {
    unsigned char type;  # 文件描述符类型，可以是 SOCKET、CHANNEL、LISTENER
    union {
        libssh2_socket_t socket;  # 套接字文件描述符，通过系统的 select() 调用检查
        LIBSSH2_CHANNEL *channel;  # 通过检查内部状态来检查的通道
        LIBSSH2_LISTENER *listener;  # 仅通过读取来检查的监听器，是否有传入连接等待接受
    } fd;
    unsigned long events;  # 请求的事件
    unsigned long revents;  # 返回的事件
} LIBSSH2_POLLFD;

# 定义 SSH2 轮询文件描述符的类型
#define LIBSSH2_POLLFD_SOCKET       1  # 套接字类型
#define LIBSSH2_POLLFD_CHANNEL      2  # 通道类型
#define LIBSSH2_POLLFD_LISTENER     3  # 监听器类型

# 注意：Win32 实际上没有 poll() 实现，因此其中一些值是用 select() 数据模拟的
# SSH2 轮询文件描述符的事件/返回事件，尽可能与 sys/poll.h 匹配
#define LIBSSH2_POLLFD_POLLIN           0x0001  # 可以读取数据或连接可用
#define LIBSSH2_POLLFD_POLLPRI          0x0002  # 优先级数据可用于读取，仅适用于套接字
# 定义了不同的标志位，用于表示不同的事件类型
#define LIBSSH2_POLLFD_POLLEXT          0x0002 /* 扩展数据可供读取 -- 仅限通道 */
#define LIBSSH2_POLLFD_POLLOUT          0x0004 /* 可以写入 -- 套接字/通道 */
/* 仅限 revents */
#define LIBSSH2_POLLFD_POLLERR          0x0008 /* 错误条件 -- 套接字 */
#define LIBSSH2_POLLFD_POLLHUP          0x0010 /* 挂断/EOF -- 套接字 */
#define LIBSSH2_POLLFD_SESSION_CLOSED   0x0010 /* 会话断开 */
#define LIBSSH2_POLLFD_POLLNVAL         0x0020 /* 无效请求 -- 仅限套接字 */
#define LIBSSH2_POLLFD_POLLEX           0x0040 /* 异常条件 -- 套接字/Win32 */
#define LIBSSH2_POLLFD_CHANNEL_CLOSED   0x0080 /* 通道断开 */
#define LIBSSH2_POLLFD_LISTENER_CLOSED  0x0080 /* 监听器断开 */

#define HAVE_LIBSSH2_SESSION_BLOCK_DIRECTION
/* 阻塞方向类型 */
#define LIBSSH2_SESSION_BLOCK_INBOUND                  0x0001
#define LIBSSH2_SESSION_BLOCK_OUTBOUND                 0x0002

/* 哈希类型 */
#define LIBSSH2_HOSTKEY_HASH_MD5                            1
#define LIBSSH2_HOSTKEY_HASH_SHA1                           2
#define LIBSSH2_HOSTKEY_HASH_SHA256                         3

/* 主机密钥类型 */
#define LIBSSH2_HOSTKEY_TYPE_UNKNOWN            0
#define LIBSSH2_HOSTKEY_TYPE_RSA                1
#define LIBSSH2_HOSTKEY_TYPE_DSS                2
#define LIBSSH2_HOSTKEY_TYPE_ECDSA_256          3
#define LIBSSH2_HOSTKEY_TYPE_ECDSA_384          4
#define LIBSSH2_HOSTKEY_TYPE_ECDSA_521          5
#define LIBSSH2_HOSTKEY_TYPE_ED25519            6

/* 断开连接代码（由SSH协议定义） */
#define SSH_DISCONNECT_HOST_NOT_ALLOWED_TO_CONNECT          1
#define SSH_DISCONNECT_PROTOCOL_ERROR                       2
#define SSH_DISCONNECT_KEY_EXCHANGE_FAILED                  3
# 定义 SSH 断开连接的保留代码
#define SSH_DISCONNECT_RESERVED                             4
# 定义 SSH 断开连接的 MAC 错误代码
#define SSH_DISCONNECT_MAC_ERROR                            5
# 定义 SSH 断开连接的压缩错误代码
#define SSH_DISCONNECT_COMPRESSION_ERROR                    6
# 定义 SSH 断开连接的服务不可用代码
#define SSH_DISCONNECT_SERVICE_NOT_AVAILABLE                7
# 定义 SSH 断开连接的协议版本不支持代码
#define SSH_DISCONNECT_PROTOCOL_VERSION_NOT_SUPPORTED       8
# 定义 SSH 断开连接的主机密钥不可验证代码
#define SSH_DISCONNECT_HOST_KEY_NOT_VERIFIABLE              9
# 定义 SSH 断开连接的连接丢失代码
#define SSH_DISCONNECT_CONNECTION_LOST                      10
# 定义 SSH 断开连接的应用程序断开代码
#define SSH_DISCONNECT_BY_APPLICATION                       11
# 定义 SSH 断开连接的连接过多代码
#define SSH_DISCONNECT_TOO_MANY_CONNECTIONS                 12
# 定义 SSH 断开连接的用户取消认证代码
#define SSH_DISCONNECT_AUTH_CANCELLED_BY_USER               13
# 定义 SSH 断开连接的没有更多认证方法可用代码
#define SSH_DISCONNECT_NO_MORE_AUTH_METHODS_AVAILABLE       14
# 定义 SSH 断开连接的非法用户名代码
#define SSH_DISCONNECT_ILLEGAL_USER_NAME                    15

# 错误代码（由 libssh2 定义）
# 定义 libssh2 错误代码为无错误
#define LIBSSH2_ERROR_NONE                      0

# 该库曾经在代码中的许多地方使用 -1 作为通用错误返回值，随着时间的推移，这被转换为 LIBSSH2_ERROR_SOCKET_NONE 使用。由于这是一个通用错误代码，目标是永远不要返回此代码，而是确保使用更准确和描述性的错误代码。
# 定义 libssh2 错误代码为无套接字
#define LIBSSH2_ERROR_SOCKET_NONE               -1
# 定义 libssh2 错误代码为接收横幅失败
#define LIBSSH2_ERROR_BANNER_RECV               -2
# 定义 libssh2 错误代码为发送横幅失败
#define LIBSSH2_ERROR_BANNER_SEND               -3
# 定义 libssh2 错误代码为无效的 MAC
#define LIBSSH2_ERROR_INVALID_MAC               -4
# 定义 libssh2 错误代码为密钥交换失败
#define LIBSSH2_ERROR_KEX_FAILURE               -5
# 定义 libssh2 错误代码为分配失败
#define LIBSSH2_ERROR_ALLOC                     -6
# 定义 libssh2 错误代码为套接字发送失败
#define LIBSSH2_ERROR_SOCKET_SEND               -7
# 定义 libssh2 错误代码为密钥交换失败
#define LIBSSH2_ERROR_KEY_EXCHANGE_FAILURE      -8
# 定义 libssh2 错误代码为超时
#define LIBSSH2_ERROR_TIMEOUT                   -9
# 定义 libssh2 错误代码为主机密钥初始化失败
#define LIBSSH2_ERROR_HOSTKEY_INIT              -10
# 定义 libssh2 错误代码为主机密钥签名失败
#define LIBSSH2_ERROR_HOSTKEY_SIGN              -11
# 定义 libssh2 错误代码为解密失败
#define LIBSSH2_ERROR_DECRYPT                   -12
# 定义 libssh2 错误代码为套接字断开连接
#define LIBSSH2_ERROR_SOCKET_DISCONNECT         -13
# 定义 libssh2 错误代码为协议错误
#define LIBSSH2_ERROR_PROTO                     -14
# 定义 libssh2 错误代码为密码过期
#define LIBSSH2_ERROR_PASSWORD_EXPIRED          -15
# 定义 LIBSSH2_ERROR_FILE 常量，数值为 -16，表示文件错误
#define LIBSSH2_ERROR_FILE                      -16
# 定义 LIBSSH2_ERROR_METHOD_NONE 常量，数值为 -17，表示没有指定方法错误
#define LIBSSH2_ERROR_METHOD_NONE               -17
# 定义 LIBSSH2_ERROR_AUTHENTICATION_FAILED 常量，数值为 -18，表示认证失败错误
#define LIBSSH2_ERROR_AUTHENTICATION_FAILED     -18
# 定义 LIBSSH2_ERROR_PUBLICKEY_UNRECOGNIZED 常量，数值为 LIBSSH2_ERROR_AUTHENTICATION_FAILED，表示公钥未识别错误
#define LIBSSH2_ERROR_PUBLICKEY_UNRECOGNIZED    \
    LIBSSH2_ERROR_AUTHENTICATION_FAILED
# 定义 LIBSSH2_ERROR_PUBLICKEY_UNVERIFIED 常量，数值为 -19，表示公钥未验证错误
#define LIBSSH2_ERROR_PUBLICKEY_UNVERIFIED      -19
# 定义 LIBSSH2_ERROR_CHANNEL_OUTOFORDER 常量，数值为 -20，表示通道顺序错误
#define LIBSSH2_ERROR_CHANNEL_OUTOFORDER        -20
# 定义 LIBSSH2_ERROR_CHANNEL_FAILURE 常量，数值为 -21，表示通道失败错误
#define LIBSSH2_ERROR_CHANNEL_FAILURE           -21
# 定义 LIBSSH2_ERROR_CHANNEL_REQUEST_DENIED 常量，数值为 -22，表示通道请求被拒绝错误
#define LIBSSH2_ERROR_CHANNEL_REQUEST_DENIED    -22
# 定义 LIBSSH2_ERROR_CHANNEL_UNKNOWN 常量，数值为 -23，表示未知通道错误
#define LIBSSH2_ERROR_CHANNEL_UNKNOWN           -23
# 定义 LIBSSH2_ERROR_CHANNEL_WINDOW_EXCEEDED 常量，数值为 -24，表示通道窗口超出错误
#define LIBSSH2_ERROR_CHANNEL_WINDOW_EXCEEDED   -24
# 定义 LIBSSH2_ERROR_CHANNEL_PACKET_EXCEEDED 常量，数值为 -25，表示通道数据包超出错误
#define LIBSSH2_ERROR_CHANNEL_PACKET_EXCEEDED   -25
# 定义 LIBSSH2_ERROR_CHANNEL_CLOSED 常量，数值为 -26，表示通道关闭错误
#define LIBSSH2_ERROR_CHANNEL_CLOSED            -26
# 定义 LIBSSH2_ERROR_CHANNEL_EOF_SENT 常量，数值为 -27，表示通道发送 EOF 错误
#define LIBSSH2_ERROR_CHANNEL_EOF_SENT          -27
# 定义 LIBSSH2_ERROR_SCP_PROTOCOL 常量，数值为 -28，表示SCP协议错误
#define LIBSSH2_ERROR_SCP_PROTOCOL              -28
# 定义 LIBSSH2_ERROR_ZLIB 常量，数值为 -29，表示ZLIB错误
#define LIBSSH2_ERROR_ZLIB                      -29
# 定义 LIBSSH2_ERROR_SOCKET_TIMEOUT 常量，数值为 -30，表示套接字超时错误
#define LIBSSH2_ERROR_SOCKET_TIMEOUT            -30
# 定义 LIBSSH2_ERROR_SFTP_PROTOCOL 常量，数值为 -31，表示SFTP协议错误
#define LIBSSH2_ERROR_SFTP_PROTOCOL             -31
# 定义 LIBSSH2_ERROR_REQUEST_DENIED 常量，数值为 -32，表示请求被拒绝错误
#define LIBSSH2_ERROR_REQUEST_DENIED            -32
# 定义 LIBSSH2_ERROR_METHOD_NOT_SUPPORTED 常量，数值为 -33，表示方法不支持错误
#define LIBSSH2_ERROR_METHOD_NOT_SUPPORTED      -33
# 定义 LIBSSH2_ERROR_INVAL 常量，数值为 -34，表示无效错误
#define LIBSSH2_ERROR_INVAL                     -34
# 定义 LIBSSH2_ERROR_INVALID_POLL_TYPE 常量，数值为 -35，表示无效的轮询类型错误
#define LIBSSH2_ERROR_INVALID_POLL_TYPE         -35
# 定义 LIBSSH2_ERROR_PUBLICKEY_PROTOCOL 常量，数值为 -36，表示公钥协议错误
#define LIBSSH2_ERROR_PUBLICKEY_PROTOCOL        -36
# 定义 LIBSSH2_ERROR_EAGAIN 常量，数值为 -37，表示再试错误
#define LIBSSH2_ERROR_EAGAIN                    -37
# 定义 LIBSSH2_ERROR_BUFFER_TOO_SMALL 常量，数值为 -38，表示缓冲区太小错误
#define LIBSSH2_ERROR_BUFFER_TOO_SMALL          -38
# 定义 LIBSSH2_ERROR_BAD_USE 常量，数值为 -39，表示错误的使用错误
#define LIBSSH2_ERROR_BAD_USE                   -39
# 定义 LIBSSH2_ERROR_COMPRESS 常量，数值为 -40，表示压缩错误
#define LIBSSH2_ERROR_COMPRESS                  -40
# 定义 LIBSSH2_ERROR_OUT_OF_BOUNDARY 常量，数值为 -41，表示超出边界错误
#define LIBSSH2_ERROR_OUT_OF_BOUNDARY           -41
# 定义 LIBSSH2_ERROR_AGENT_PROTOCOL 常量，数值为 -42，表示代理协议错误
#define LIBSSH2_ERROR_AGENT_PROTOCOL            -42
# 定义 LIBSSH2_ERROR_SOCKET_RECV 常量，数值为 -43，表示套接字接收错误
#define LIBSSH2_ERROR_SOCKET_RECV               -43
# 定义 LIBSSH2_ERROR_ENCRYPT 常量，数值为 -44，表示加密错误
#define LIBSSH2_ERROR_ENCRYPT                   -44
# 定义 LIBSSH2_ERROR_BAD_SOCKET 常量，数值为 -45，表示错误的套接字错误
#define LIBSSH2_ERROR_BAD_SOCKET                -45
# 定义 LIBSSH2_ERROR_KNOWN_HOSTS 常量，数值为 -46，表示已知主机错误
#define LIBSSH2_ERROR_KNOWN_HOSTS               -46
# 定义 LIBSSH2_ERROR_CHANNEL_WINDOW_FULL 常量，数值为 -47，表示通道窗口已满错误
#define LIBSSH2_ERROR_CHANNEL_WINDOW_FULL       -47
# 定义 LIBSSH2_ERROR_KEYFILE_AUTH_FAILED 常量，数值为 -48，表示密钥文件认证失败错误
#define LIBSSH2_ERROR_KEYFILE_AUTH_FAILED       -48
# 定义 LIBSSH2_ERROR_RANDGEN 常量，数值为 -49，表示随机生成错误
#define LIBSSH2_ERROR_RANDGEN                   -49

# 定义 LIBSSH2_ERROR_BANNER_NONE 常量，数值为 LIBSSH2_ERROR_BANNER_RECV，表示没有横幅错误
#define LIBSSH2_ERROR_BANNER_NONE LIBSSH2_ERROR_BANNER_RECV

# 全局 API
#define LIBSSH2_INIT_NO_CRYPTO        0x0001
# 定义了一个名为LIBSSH2_INIT_NO_CRYPTO的常量，数值为0x0001

/*
 * libssh2_init()
 *
 * 初始化libssh2函数。通常会初始化加密库。它使用全局状态，不是线程安全的 --
 * 你必须确保这个函数不会被并发调用。
 *
 * 标志可以是：
 * 0:                              正常初始化
 * LIBSSH2_INIT_NO_CRYPTO:         不初始化加密库（例如，对于OpenSSL，不调用OPENSSL_add_cipher_algoritms()）
 *
 * 如果成功返回0，如果出错返回负值。
 */
LIBSSH2_API int libssh2_init(int flags);
# 初始化libssh2函数，根据传入的标志进行初始化

/*
 * libssh2_exit()
 *
 * 退出libssh2函数并释放所有内部使用的内存。
 */
LIBSSH2_API void libssh2_exit(void);
# 退出libssh2函数并释放所有内部使用的内存

/*
 * libssh2_free()
 *
 * 释放先前调用libssh2函数分配的内存。
 */
LIBSSH2_API void libssh2_free(LIBSSH2_SESSION *session, void *ptr);
# 释放先前调用libssh2函数分配的内存

/*
 * libssh2_session_supported_algs()
 *
 * 用支持的加密算法列表填充algs。成功时返回非负数（支持的算法数量），失败时返回负数（错误代码）。
 *
 * 注意：成功时，algs在不再需要时必须被释放（通过调用libssh2_free）
 */
LIBSSH2_API int libssh2_session_supported_algs(LIBSSH2_SESSION* session,
                                               int method_type,
                                               const char ***algs);
# 获取会话支持的加密算法列表

/* 会话API */
LIBSSH2_API LIBSSH2_SESSION *
libssh2_session_init_ex(LIBSSH2_ALLOC_FUNC((*my_alloc)),
                        LIBSSH2_FREE_FUNC((*my_free)),
                        LIBSSH2_REALLOC_FUNC((*my_realloc)), void *abstract);
# 初始化会话，使用指定的内存分配和释放函数

#define libssh2_session_init() libssh2_session_init_ex(NULL, NULL, NULL, NULL)
# 定义一个宏，简化会话初始化的调用

LIBSSH2_API void **libssh2_session_abstract(LIBSSH2_SESSION *session);
# 获取会话的抽象数据

LIBSSH2_API void *libssh2_session_callback_set(LIBSSH2_SESSION *session,
                                               int cbtype, void *callback);
# 设置会话的回调函数
# 设置会话的欢迎横幅
LIBSSH2_API int libssh2_session_banner_set(LIBSSH2_SESSION *session,
                                           const char *banner);
# 设置会话的欢迎横幅
LIBSSH2_API int libssh2_banner_set(LIBSSH2_SESSION *session,
                                   const char *banner);

# 启动 SSH 会话
LIBSSH2_API int libssh2_session_startup(LIBSSH2_SESSION *session, int sock);
# 进行 SSH 会话握手
LIBSSH2_API int libssh2_session_handshake(LIBSSH2_SESSION *session,
                                          libssh2_socket_t sock);
# 断开 SSH 会话连接
LIBSSH2_API int libssh2_session_disconnect_ex(LIBSSH2_SESSION *session,
                                              int reason,
                                              const char *description,
                                              const char *lang);
# 宏定义，用于断开 SSH 会话连接
#define libssh2_session_disconnect(session, description) \
  libssh2_session_disconnect_ex((session), SSH_DISCONNECT_BY_APPLICATION, \
                                (description), "")

# 释放 SSH 会话资源
LIBSSH2_API int libssh2_session_free(LIBSSH2_SESSION *session);

# 获取主机密钥的哈希值
LIBSSH2_API const char *libssh2_hostkey_hash(LIBSSH2_SESSION *session,
                                             int hash_type);

# 获取会话的主机密钥
LIBSSH2_API const char *libssh2_session_hostkey(LIBSSH2_SESSION *session,
                                                size_t *len, int *type);

# 设置会话的方法偏好
LIBSSH2_API int libssh2_session_method_pref(LIBSSH2_SESSION *session,
                                            int method_type,
                                            const char *prefs);
# 获取会话支持的方法
LIBSSH2_API const char *libssh2_session_methods(LIBSSH2_SESSION *session,
                                                int method_type);
# 获取会话的最后一个错误信息
LIBSSH2_API int libssh2_session_last_error(LIBSSH2_SESSION *session,
                                           char **errmsg,
                                           int *errmsg_len, int want_buf);
# 获取会话的最后一个错误码
LIBSSH2_API int libssh2_session_last_errno(LIBSSH2_SESSION *session);
# 设置会话的最后错误信息
LIBSSH2_API int libssh2_session_set_last_error(LIBSSH2_SESSION* session,
                                               int errcode,
                                               const char *errmsg);
# 设置会话的阻塞方向
LIBSSH2_API int libssh2_session_block_directions(LIBSSH2_SESSION *session);

# 设置会话的标志
LIBSSH2_API int libssh2_session_flag(LIBSSH2_SESSION *session, int flag,
                                     int value);
# 获取会话的欢迎信息
LIBSSH2_API const char *libssh2_session_banner_get(LIBSSH2_SESSION *session);

/* 用户认证 API */
# 列出用户的认证方式
LIBSSH2_API char *libssh2_userauth_list(LIBSSH2_SESSION *session,
                                        const char *username,
                                        unsigned int username_len);
# 检查用户是否已认证
LIBSSH2_API int libssh2_userauth_authenticated(LIBSSH2_SESSION *session);

# 使用密码进行用户认证
LIBSSH2_API int
libssh2_userauth_password_ex(LIBSSH2_SESSION *session,
                             const char *username,
                             unsigned int username_len,
                             const char *password,
                             unsigned int password_len,
                             LIBSSH2_PASSWD_CHANGEREQ_FUNC
                             ((*passwd_change_cb)));

# 宏定义，使用密码进行用户认证
#define libssh2_userauth_password(session, username, password) \
 libssh2_userauth_password_ex((session), (username),           \
                              (unsigned int)strlen(username),  \
                              (password), (unsigned int)strlen(password), NULL)

# 使用公钥文件进行用户认证
LIBSSH2_API int
libssh2_userauth_publickey_fromfile_ex(LIBSSH2_SESSION *session,
                                       const char *username,
                                       unsigned int username_len,
                                       const char *publickey,
                                       const char *privatekey,
                                       const char *passphrase);

# 宏定义，使用公钥文件进行用户认证
#define libssh2_userauth_publickey_fromfile(session, username, publickey, \
                                            privatekey, passphrase)     \
    # 使用公钥和私钥进行用户身份验证
    libssh2_userauth_publickey_fromfile_ex((session), (username),       \
                                           (unsigned int)strlen(username), \
                                           (publickey),                 \
                                           (privatekey), (passphrase))
# 使用公钥进行用户身份验证
LIBSSH2_API int
libssh2_userauth_publickey(LIBSSH2_SESSION *session,  # SSH会话对象
                           const char *username,  # 用户名
                           const unsigned char *pubkeydata,  # 公钥数据
                           size_t pubkeydata_len,  # 公钥数据长度
                           LIBSSH2_USERAUTH_PUBLICKEY_SIGN_FUNC  # 公钥签名回调函数
                           ((*sign_callback)),  # 签名回调函数指针
                           void **abstract);  # 抽象指针

# 使用基于主机的公钥进行用户身份验证
LIBSSH2_API int
libssh2_userauth_hostbased_fromfile_ex(LIBSSH2_SESSION *session,  # SSH会话对象
                                       const char *username,  # 用户名
                                       unsigned int username_len,  # 用户名长度
                                       const char *publickey,  # 公钥文件路径
                                       const char *privatekey,  # 私钥文件路径
                                       const char *passphrase,  # 密码短语
                                       const char *hostname,  # 主机名
                                       unsigned int hostname_len,  # 主机名长度
                                       const char *local_username,  # 本地用户名
                                       unsigned int local_username_len);  # 本地用户名长度

# 宏定义，使用基于主机的公钥进行用户身份验证
#define libssh2_userauth_hostbased_fromfile(session, username, publickey, \
                                            privatekey, passphrase, hostname) \
 libssh2_userauth_hostbased_fromfile_ex((session), (username), \
                                        (unsigned int)strlen(username), \
                                        (publickey),                    \
                                        (privatekey), (passphrase),     \
                                        (hostname),                     \
                                        (unsigned int)strlen(hostname), \
                                        (username),                     \
                                        (unsigned int)strlen(username))

LIBSSH2_API int
# 从内存中的公钥和私钥进行用户身份验证
libssh2_userauth_publickey_frommemory(LIBSSH2_SESSION *session,
                                      const char *username,
                                      size_t username_len,
                                      const char *publickeyfiledata,
                                      size_t publickeyfiledata_len,
                                      const char *privatekeyfiledata,
                                      size_t privatekeyfiledata_len,
                                      const char *passphrase);

/*
 * response_callback 由库填充提示数组，但客户端必须分配和填充单独的响应。
 * 响应数组已经分配。响应数据将在回调返回后由 libssh2 释放，但在后续回调调用之前。
 */
LIBSSH2_API int
libssh2_userauth_keyboard_interactive_ex(LIBSSH2_SESSION* session,
                                         const char *username,
                                         unsigned int username_len,
                                         LIBSSH2_USERAUTH_KBDINT_RESPONSE_FUNC(
                                                       (*response_callback)));

# 使用键盘交互方式进行用户身份验证
#define libssh2_userauth_keyboard_interactive(session, username,        \
                                              response_callback)        \
    libssh2_userauth_keyboard_interactive_ex((session), (username),     \
                                             (unsigned int)strlen(username), \
                                             (response_callback))

# 轮询函数，用于等待 I/O 事件
LIBSSH2_API int libssh2_poll(LIBSSH2_POLLFD *fds, unsigned int nfds,
                             long timeout);

/* 通道 API */

# 默认通道窗口大小
#define LIBSSH2_CHANNEL_WINDOW_DEFAULT  (2*1024*1024)
# 默认通道数据包大小
#define LIBSSH2_CHANNEL_PACKET_DEFAULT  32768
# 通道最小调整值
#define LIBSSH2_CHANNEL_MINADJUST       1024

/* 扩展数据处理 */

# 扩展数据正常处理
#define LIBSSH2_CHANNEL_EXTENDED_DATA_NORMAL        0
# 忽略扩展数据
#define LIBSSH2_CHANNEL_EXTENDED_DATA_IGNORE        1
# 定义常量，表示扩展数据合并
#define LIBSSH2_CHANNEL_EXTENDED_DATA_MERGE         2

# 定义常量，表示扩展数据类型为标准错误输出
#define SSH_EXTENDED_DATA_STDERR 1

# 定义常量，表示在读写操作中会出现阻塞
# 返回值为 LIBSSH2_ERROR_EAGAIN
#define LIBSSH2CHANNEL_EAGAIN LIBSSH2_ERROR_EAGAIN

# 打开一个新的通道，返回通道对象
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_channel_open_ex(LIBSSH2_SESSION *session, const char *channel_type,
                        unsigned int channel_type_len,
                        unsigned int window_size, unsigned int packet_size,
                        const char *message, unsigned int message_len);

# 宏定义，打开一个会话类型的通道
#define libssh2_channel_open_session(session) \
  libssh2_channel_open_ex((session), "session", sizeof("session") - 1, \
                          LIBSSH2_CHANNEL_WINDOW_DEFAULT, \
                          LIBSSH2_CHANNEL_PACKET_DEFAULT, NULL, 0)

# 打开一个直接的 TCP/IP 通道，返回通道对象
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_channel_direct_tcpip_ex(LIBSSH2_SESSION *session, const char *host,
                                int port, const char *shost, int sport);
# 宏定义，打开一个直接的 TCP/IP 通道
#define libssh2_channel_direct_tcpip(session, host, port) \
  libssh2_channel_direct_tcpip_ex((session), (host), (port), "127.0.0.1", 22)

# 监听传入的连接请求，返回监听器对象
LIBSSH2_API LIBSSH2_LISTENER *
libssh2_channel_forward_listen_ex(LIBSSH2_SESSION *session, const char *host,
                                  int port, int *bound_port,
                                  int queue_maxsize);
# 宏定义，监听传入的连接请求
#define libssh2_channel_forward_listen(session, port) \
 libssh2_channel_forward_listen_ex((session), NULL, (port), NULL, 16)

# 取消传入连接的转发
LIBSSH2_API int libssh2_channel_forward_cancel(LIBSSH2_LISTENER *listener);

# 接受传入连接的转发，返回通道对象
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_channel_forward_accept(LIBSSH2_LISTENER *listener);

# 设置通道的环境变量
LIBSSH2_API int libssh2_channel_setenv_ex(LIBSSH2_CHANNEL *channel,
                                          const char *varname,
                                          unsigned int varname_len,
                                          const char *value,
                                          unsigned int value_len);
# 定义一个宏，用于设置通道环境变量
#define libssh2_channel_setenv(channel, varname, value)                 \
    libssh2_channel_setenv_ex((channel), (varname),                     \
                              (unsigned int)strlen(varname), (value),   \
                              (unsigned int)strlen(value))

# 请求使用身份验证代理进行认证
LIBSSH2_API int libssh2_channel_request_auth_agent(LIBSSH2_CHANNEL *channel);

# 请求分配一个伪终端
LIBSSH2_API int libssh2_channel_request_pty_ex(LIBSSH2_CHANNEL *channel,
                                               const char *term,
                                               unsigned int term_len,
                                               const char *modes,
                                               unsigned int modes_len,
                                               int width, int height,
                                               int width_px, int height_px);
# 定义一个宏，用于请求分配一个伪终端
#define libssh2_channel_request_pty(channel, term)                      \
    libssh2_channel_request_pty_ex((channel), (term),                   \
                                   (unsigned int)strlen(term),          \
                                   NULL, 0,                             \
                                   LIBSSH2_TERM_WIDTH,                  \
                                   LIBSSH2_TERM_HEIGHT,                 \
                                   LIBSSH2_TERM_WIDTH_PX,               \
                                   LIBSSH2_TERM_HEIGHT_PX)

# 请求设置伪终端的大小
LIBSSH2_API int libssh2_channel_request_pty_size_ex(LIBSSH2_CHANNEL *channel,
                                                    int width, int height,
                                                    int width_px,
                                                    int height_px);
# 定义一个宏，用于请求设置伪终端的大小
#define libssh2_channel_request_pty_size(channel, width, height) \
  libssh2_channel_request_pty_size_ex((channel), (width), (height), 0, 0)
# 请求在 SSH 通道上启用 X11 转发
LIBSSH2_API int libssh2_channel_x11_req_ex(LIBSSH2_CHANNEL *channel,
                                           int single_connection,
                                           const char *auth_proto,
                                           const char *auth_cookie,
                                           int screen_number);
# 定义一个宏，用于简化 X11 请求的调用
#define libssh2_channel_x11_req(channel, screen_number) \
 libssh2_channel_x11_req_ex((channel), 0, NULL, NULL, (screen_number))

# 在 SSH 通道上启动一个 shell 进程
LIBSSH2_API int libssh2_channel_process_startup(LIBSSH2_CHANNEL *channel,
                                                const char *request,
                                                unsigned int request_len,
                                                const char *message,
                                                unsigned int message_len);
# 定义一个宏，用于简化启动 shell 进程的调用
#define libssh2_channel_shell(channel) \
  libssh2_channel_process_startup((channel), "shell", sizeof("shell") - 1, \
                                  NULL, 0)
# 定义一个宏，用于简化执行命令的调用
#define libssh2_channel_exec(channel, command) \
  libssh2_channel_process_startup((channel), "exec", sizeof("exec") - 1, \
                                  (command), (unsigned int)strlen(command))
# 定义一个宏，用于简化启动子系统的调用
#define libssh2_channel_subsystem(channel, subsystem) \
  libssh2_channel_process_startup((channel), "subsystem",              \
                                  sizeof("subsystem") - 1, (subsystem), \
                                  (unsigned int)strlen(subsystem))

# 从 SSH 通道中读取数据
LIBSSH2_API ssize_t libssh2_channel_read_ex(LIBSSH2_CHANNEL *channel,
                                            int stream_id, char *buf,
                                            size_t buflen);
# 定义一个宏，用于简化从通道中读取数据的调用
#define libssh2_channel_read(channel, buf, buflen) \
  libssh2_channel_read_ex((channel), 0, (buf), (buflen))
# 定义一个宏，用于简化从标准错误流中读取数据的调用
#define libssh2_channel_read_stderr(channel, buf, buflen) \
  libssh2_channel_read_ex((channel), SSH_EXTENDED_DATA_STDERR, (buf), (buflen))
# 定义一个函数，用于轮询通道以读取数据
LIBSSH2_API int libssh2_poll_channel_read(LIBSSH2_CHANNEL *channel,
                                          int extended);

# 定义一个函数，用于获取通道的读取窗口大小
LIBSSH2_API unsigned long
libssh2_channel_window_read_ex(LIBSSH2_CHANNEL *channel,
                               unsigned long *read_avail,
                               unsigned long *window_size_initial);
# 定义一个宏，用于简化获取通道的读取窗口大小的操作
#define libssh2_channel_window_read(channel) \
  libssh2_channel_window_read_ex((channel), NULL, NULL)

# 定义一个函数，用于调整通道的接收窗口大小（已废弃，不建议使用）
LIBSSH2_API unsigned long
libssh2_channel_receive_window_adjust(LIBSSH2_CHANNEL *channel,
                                      unsigned long adjustment,
                                      unsigned char force);

# 定义一个函数，用于调整通道的接收窗口大小（已废弃，不建议使用）
LIBSSH2_API int
libssh2_channel_receive_window_adjust2(LIBSSH2_CHANNEL *channel,
                                       unsigned long adjustment,
                                       unsigned char force,
                                       unsigned int *storewindow);

# 定义一个函数，用于向通道写入数据
LIBSSH2_API ssize_t libssh2_channel_write_ex(LIBSSH2_CHANNEL *channel,
                                             int stream_id, const char *buf,
                                             size_t buflen);
# 定义一个宏，用于简化向通道写入数据的操作
#define libssh2_channel_write(channel, buf, buflen) \
  libssh2_channel_write_ex((channel), 0, (buf), (buflen))
# 定义一个宏，用于向标准错误通道写入数据
#define libssh2_channel_write_stderr(channel, buf, buflen)              \
    libssh2_channel_write_ex((channel), SSH_EXTENDED_DATA_STDERR,       \
                             (buf), (buflen))

# 定义一个函数，用于获取通道的写入窗口大小
LIBSSH2_API unsigned long
libssh2_channel_window_write_ex(LIBSSH2_CHANNEL *channel,
                                unsigned long *window_size_initial);
# 定义一个宏，用于简化获取通道的写入窗口大小的操作
#define libssh2_channel_window_write(channel) \
  libssh2_channel_window_write_ex((channel), NULL)

# 定义一个函数，用于设置会话的阻塞模式
LIBSSH2_API void libssh2_session_set_blocking(LIBSSH2_SESSION* session,
                                              int blocking);
# 定义一个函数，用于获取会话的阻塞模式
LIBSSH2_API int libssh2_session_get_blocking(LIBSSH2_SESSION* session);
# 设置通道是否为阻塞模式
LIBSSH2_API void libssh2_channel_set_blocking(LIBSSH2_CHANNEL *channel,
                                              int blocking);

# 设置会话超时时间
LIBSSH2_API void libssh2_session_set_timeout(LIBSSH2_SESSION* session,
                                             long timeout);
# 获取会话超时时间
LIBSSH2_API long libssh2_session_get_timeout(LIBSSH2_SESSION* session);

# 处理扩展数据，已废弃，不建议使用
LIBSSH2_API void libssh2_channel_handle_extended_data(LIBSSH2_CHANNEL *channel,
                                                      int ignore_mode);
# 处理扩展数据，已废弃，不建议使用
LIBSSH2_API int libssh2_channel_handle_extended_data2(LIBSSH2_CHANNEL *channel,
                                                      int ignore_mode);

# 忽略扩展数据，为了与版本0.1兼容
# 未来使用应该直接使用libssh2_channel_handle_extended_data()
# 如果传入LIBSSH2_CHANNEL_EXTENDED_DATA_MERGE，扩展数据将从标准数据通道中读取（FIFO）
/* DEPRECATED */
#define libssh2_channel_ignore_extended_data(channel, ignore) \
  libssh2_channel_handle_extended_data((channel),                       \
                                       (ignore) ?                       \
                                       LIBSSH2_CHANNEL_EXTENDED_DATA_IGNORE : \
                                       LIBSSH2_CHANNEL_EXTENDED_DATA_NORMAL)

# 刷新扩展数据
#define LIBSSH2_CHANNEL_FLUSH_EXTENDED_DATA     -1
# 刷新所有数据
#define LIBSSH2_CHANNEL_FLUSH_ALL               -2
LIBSSH2_API int libssh2_channel_flush_ex(LIBSSH2_CHANNEL *channel,
                                         int streamid);
# 刷新通道数据
#define libssh2_channel_flush(channel) libssh2_channel_flush_ex((channel), 0)
# 刷新标准错误通道数据
#define libssh2_channel_flush_stderr(channel) \
 libssh2_channel_flush_ex((channel), SSH_EXTENDED_DATA_STDERR)

# 获取通道的退出状态
LIBSSH2_API int libssh2_channel_get_exit_status(LIBSSH2_CHANNEL* channel);
# 获取通道的退出信号
LIBSSH2_API int libssh2_channel_get_exit_signal(LIBSSH2_CHANNEL* channel,
                                                char **exitsignal,
                                                size_t *exitsignal_len,
                                                char **errmsg,
                                                size_t *errmsg_len,
                                                char **langtag,
                                                size_t *langtag_len);
# 发送 EOF 到通道
LIBSSH2_API int libssh2_channel_send_eof(LIBSSH2_CHANNEL *channel);
# 检查通道是否已经到达 EOF
LIBSSH2_API int libssh2_channel_eof(LIBSSH2_CHANNEL *channel);
# 等待通道到达 EOF
LIBSSH2_API int libssh2_channel_wait_eof(LIBSSH2_CHANNEL *channel);
# 关闭通道
LIBSSH2_API int libssh2_channel_close(LIBSSH2_CHANNEL *channel);
# 等待通道关闭
LIBSSH2_API int libssh2_channel_wait_closed(LIBSSH2_CHANNEL *channel);
# 释放通道资源
LIBSSH2_API int libssh2_channel_free(LIBSSH2_CHANNEL *channel);

# libssh2_scp_recv 已经被废弃，不要使用
LIBSSH2_API LIBSSH2_CHANNEL *libssh2_scp_recv(LIBSSH2_SESSION *session,
                                              const char *path,
                                              struct stat *sb);
# 在 Windows 上支持大于 2GB 的文件时，请使用 libssh2_scp_recv2
LIBSSH2_API LIBSSH2_CHANNEL *libssh2_scp_recv2(LIBSSH2_SESSION *session,
                                               const char *path,
                                               libssh2_struct_stat *sb);
# 发送文件到远程主机
LIBSSH2_API LIBSSH2_CHANNEL *libssh2_scp_send_ex(LIBSSH2_SESSION *session,
                                                 const char *path, int mode,
                                                 size_t size, long mtime,
                                                 long atime);
# 发送大于 2GB 的文件到远程主机
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_scp_send64(LIBSSH2_SESSION *session, const char *path, int mode,
                   libssh2_int64_t size, time_t mtime, time_t atime);
# 宏定义，用于发送文件到远程主机
#define libssh2_scp_send(session, path, mode, size) \
  libssh2_scp_send_ex((session), (path), (mode), (size), 0, 0)
# 定义一个函数，用于将 base64 编码的字符串解码为二进制数据
LIBSSH2_API int libssh2_base64_decode(LIBSSH2_SESSION *session, char **dest,
                                      unsigned int *dest_len,
                                      const char *src, unsigned int src_len);

# 定义一个函数，用于获取 libssh2 库的版本信息
LIBSSH2_API
const char *libssh2_version(int req_version_num);

# 定义一个宏，表示是否支持 libssh2_knownhost API，值为 0x010101，表示自 1.1.1 版本开始支持
#define HAVE_LIBSSH2_KNOWNHOST_API 0x010101 /* since 1.1.1 */
# 定义一个宏，表示是否支持 libssh2_version API，值为 0x010100，表示自 1.1 版本开始支持
#define HAVE_LIBSSH2_VERSION_API   0x010100 /* libssh2_version since 1.1 */

# 定义一个结构体，表示 libssh2_knownhost 对象
struct libssh2_knownhost {
    unsigned int magic;  /* 存储在库中的魔术值 */
    void *node; /* 内部表示此主机的句柄 */
    char *name; /* 如果没有明文主机名，则为 NULL */
    char *key;  /* 以 base64/可打印格式表示的密钥 */
    int typemask;  /* 类型掩码 */
};

# 定义一个函数，用于初始化已知主机的集合，返回指向集合的指针
LIBSSH2_API LIBSSH2_KNOWNHOSTS *
libssh2_knownhost_init(LIBSSH2_SESSION *session);

# 定义一个函数，用于向已知主机的集合中添加主机及其关联的密钥
# type 参数指定给定主机和密钥的格式
# 如果选择 'sha1' 作为类型，则必须提供盐到盐参数。这也是 base64 编码的。
# 如果使用自定义类型，则忽略盐，并且在 libssh2_knownhost_check() 函数中检查时，必须提供预先哈希的主机。
# 如果密钥以 NULL 结尾的 base64 编码字符串提供，则可以省略 keylen 参数（为零）。
LIBSSH2_API int libssh2_knownhost_add(LIBSSH2_KNOWNHOSTS *hosts, const char *host, const char *salt,
                                      const char *key, size_t keylen, int typemask);

# 定义一个宏，表示主机格式（2 位）
#define LIBSSH2_KNOWNHOST_TYPE_MASK    0xffff
# 定义一个宏，表示已知主机的类型为 plain（明文）
#define LIBSSH2_KNOWNHOST_TYPE_PLAIN   1
# 定义一个宏，表示已知主机的类型为 sha1（始终为 base64 编码）
#define LIBSSH2_KNOWNHOST_TYPE_SHA1    2
# 定义一个宏，表示已知主机的类型为 custom（自定义哈希）
#define LIBSSH2_KNOWNHOST_TYPE_CUSTOM  3

# 定义一个宏，表示密钥格式（2 位）
#define LIBSSH2_KNOWNHOST_KEYENC_MASK     (3<<16)
#define LIBSSH2_KNOWNHOST_KEYENC_RAW      (1<<16)  // 定义已知主机密钥编码为原始格式
#define LIBSSH2_KNOWNHOST_KEYENC_BASE64   (2<<16)  // 定义已知主机密钥编码为Base64格式

/* type of key (4 bits) */
#define LIBSSH2_KNOWNHOST_KEY_MASK         (15<<18)  // 定义已知主机密钥类型的掩码
#define LIBSSH2_KNOWNHOST_KEY_SHIFT        18  // 定义已知主机密钥类型的偏移量
#define LIBSSH2_KNOWNHOST_KEY_RSA1         (1<<18)  // 定义已知主机密钥类型为RSA1
#define LIBSSH2_KNOWNHOST_KEY_SSHRSA       (2<<18)  // 定义已知主机密钥类型为SSH-RSA
#define LIBSSH2_KNOWNHOST_KEY_SSHDSS       (3<<18)  // 定义已知主机密钥类型为SSH-DSS
#define LIBSSH2_KNOWNHOST_KEY_ECDSA_256    (4<<18)  // 定义已知主机密钥类型为ECDSA-256
#define LIBSSH2_KNOWNHOST_KEY_ECDSA_384    (5<<18)  // 定义已知主机密钥类型为ECDSA-384
#define LIBSSH2_KNOWNHOST_KEY_ECDSA_521    (6<<18)  // 定义已知主机密钥类型为ECDSA-521
#define LIBSSH2_KNOWNHOST_KEY_ED25519      (7<<18)  // 定义已知主机密钥类型为ED25519
#define LIBSSH2_KNOWNHOST_KEY_UNKNOWN      (15<<18)  // 定义已知主机密钥类型为未知类型

LIBSSH2_API int
libssh2_knownhost_add(LIBSSH2_KNOWNHOSTS *hosts,
                      const char *host,
                      const char *salt,
                      const char *key, size_t keylen, int typemask,
                      struct libssh2_knownhost **store);
/*
 * libssh2_knownhost_addc
 *
 * Add a host and its associated key to the collection of known hosts.
 *
 * Takes a comment argument that may be NULL.  A NULL comment indicates
 * there is no comment and the entry will end directly after the key
 * when written out to a file.  An empty string "" comment will indicate an
 * empty comment which will cause a single space to be written after the key.
 *
 * The 'type' argument specifies on what format the given host and keys are:
 *
 * plain  - ascii "hostname.domain.tld"
 * sha1   - SHA1(<salt> <host>) base64-encoded!
 * custom - another hash
 *
 * If 'sha1' is selected as type, the salt must be provided to the salt
 * argument. This too base64 encoded.
 *
 * The SHA-1 hash is what OpenSSH can be told to use in known_hosts files.  If
 * a custom type is used, salt is ignored and you must provide the host
 * pre-hashed when checking for it in the libssh2_knownhost_check() function.
 *
 * The keylen parameter may be omitted (zero) if the key is provided as a
 * NULL-terminated base64-encoded string.
 */

LIBSSH2_API int
# 添加一个已知主机到已知主机集合中
libssh2_knownhost_addc(LIBSSH2_KNOWNHOSTS *hosts,
                       const char *host,
                       const char *salt,
                       const char *key, size_t keylen,
                       const char *comment, size_t commentlen, int typemask,
                       struct libssh2_knownhost **store);

/*
 * libssh2_knownhost_check
 *
 * 检查主机及其关联的密钥是否在已知主机集合中
 *
 * 类型是给定主机名的类型/格式
 *
 * plain  - ascii "hostname.domain.tld"
 * custom - 预先哈希的 base64 编码。注意这不能使用任何盐。
 *
 *
 * 如果你不关心这个信息，'knownhost' 可以设置为 NULL
 *
 * 返回：
 *
 * LIBSSH2_KNOWNHOST_CHECK_* 值，见下文
 *
 */

#define LIBSSH2_KNOWNHOST_CHECK_MATCH    0
#define LIBSSH2_KNOWNHOST_CHECK_MISMATCH 1
#define LIBSSH2_KNOWNHOST_CHECK_NOTFOUND 2
#define LIBSSH2_KNOWNHOST_CHECK_FAILURE  3

LIBSSH2_API int
libssh2_knownhost_check(LIBSSH2_KNOWNHOSTS *hosts,
                        const char *host, const char *key, size_t keylen,
                        int typemask,
                        struct libssh2_knownhost **knownhost);

/* 这个函数与上面的函数相同，但还接受一个端口参数，允许 libssh2 进行更好的检查 */
LIBSSH2_API int
libssh2_knownhost_checkp(LIBSSH2_KNOWNHOSTS *hosts,
                         const char *host, int port,
                         const char *key, size_t keylen,
                         int typemask,
                         struct libssh2_knownhost **knownhost);

/*
 * libssh2_knownhost_del
 *
 * 从已知主机集合中删除一个主机。'entry' 结构体是通过调用 libssh2_knownhost_check() 获取的。
 *
 */
LIBSSH2_API int
libssh2_knownhost_del(LIBSSH2_KNOWNHOSTS *hosts,
                      struct libssh2_knownhost *entry);

/*
 * libssh2_knownhost_free
 *
 * 释放整个已知主机集合。
 *
 */
LIBSSH2_API void
# 释放已分配的已知主机列表
libssh2_knownhost_free(LIBSSH2_KNOWNHOSTS *hosts);

'''
 * libssh2_knownhost_readline()
 *
 * 传入一个文件的一行内容，让 libssh2 读取这一行。
 *
 * LIBSSH2_KNOWNHOST_FILE_OPENSSH 是唯一支持的类型。
 *
 */
LIBSSH2_API int
libssh2_knownhost_readline(LIBSSH2_KNOWNHOSTS *hosts,
                           const char *line, size_t len, int type);

'''
 * libssh2_knownhost_readfile
 *
 * 从给定文件中添加主机+密钥对。
 *
 * 返回负值表示错误或成功添加的主机数量。
 *
 * 当前实现只知道一种 'type'（openssh），其他类型保留供将来使用。
 */

#define LIBSSH2_KNOWNHOST_FILE_OPENSSH 1

LIBSSH2_API int
libssh2_knownhost_readfile(LIBSSH2_KNOWNHOSTS *hosts,
                           const char *filename, int type);

'''
 * libssh2_knownhost_writeline()
 *
 * 请求 libssh2 将已知主机转换为输出行以进行存储。
 *
 * 注意，如果给定的输出缓冲区太小而无法容纳所需的输出，则此函数返回 LIBSSH2_ERROR_BUFFER_TOO_SMALL。
 *
 * 当前实现只知道一种 'type'（openssh），其他类型保留供将来使用。
 *
 */
LIBSSH2_API int
libssh2_knownhost_writeline(LIBSSH2_KNOWNHOSTS *hosts,
                            struct libssh2_knownhost *known,
                            char *buffer, size_t buflen,
                            size_t *outlen, /* the amount of written data */
                            int type);

'''
 * libssh2_knownhost_writefile
 *
 * 将主机+密钥对写入给定文件。
 *
 * 当前实现只知道一种 'type'（openssh），其他类型保留供将来使用。
 */

LIBSSH2_API int
libssh2_knownhost_writefile(LIBSSH2_KNOWNHOSTS *hosts,
                            const char *filename, int type);
# 定义 libssh2_knownhost_get() 函数，用于遍历已知主机列表，获取下一个或第一个已知主机
# 参数 hosts: 已知主机列表
# 参数 store: 存储已知主机信息的指针
# 参数 prev: 上一个已知主机的指针，传入 NULL 获取第一个已知主机
# 返回值:
# 0 - 存储了一个良好的主机在 'store' 中
# 1 - 已知主机列表结束
# [负数] - 错误
LIBSSH2_API int
libssh2_knownhost_get(LIBSSH2_KNOWNHOSTS *hosts,
                      struct libssh2_knownhost **store,
                      struct libssh2_knownhost *prev);

# 定义 libssh2_agent_publickey 结构体，用于存储代理公钥信息
struct libssh2_agent_publickey {
    unsigned int magic;              /* 由库存储的魔术值 */
    void *node;     /* 内部密钥表示的句柄 */
    unsigned char *blob;           /* 公钥 blob */
    size_t blob_len;               /* 公钥 blob 的长度 */
    char *comment;                 /* 可打印格式的注释 */
};

# 定义 libssh2_agent_init() 函数，用于初始化一个 ssh-agent 句柄，返回句柄指针
# 参数 session: SSH 会话
# 返回值: 初始化的 ssh-agent 句柄指针
LIBSSH2_API LIBSSH2_AGENT *
libssh2_agent_init(LIBSSH2_SESSION *session);

# 定义 libssh2_agent_connect() 函数，用于连接到一个 ssh-agent
# 参数 agent: ssh-agent 句柄
# 返回值: 0 - 连接成功，负数 - 错误
LIBSSH2_API int
libssh2_agent_connect(LIBSSH2_AGENT *agent);

# 定义 libssh2_agent_list_identities() 函数，用于请求 ssh-agent 列出身份
# 参数 agent: ssh-agent 句柄
# 返回值: 0 - 请求成功，负数 - 错误
LIBSSH2_API int
libssh2_agent_list_identities(LIBSSH2_AGENT *agent);

# 定义 libssh2_agent_get_identity() 函数，用于遍历内部公钥列表，获取下一个或第一个公钥
# 参数 agent: ssh-agent 句柄
# 参数 store: 存储公钥信息的指针
# 参数 prev: 上一个公钥的指针，传入 NULL 获取第一个公钥
# 返回值:
# 0 - 存储了一个良好的公钥在 'store' 中
# 1 - 公钥列表结束
# [负数] - 错误
LIBSSH2_API int
libssh2_agent_get_identity(LIBSSH2_AGENT *agent,
               struct libssh2_agent_publickey **store,
               struct libssh2_agent_publickey *prev);
/*
 * libssh2_agent_userauth()
 *
 * 使用 ssh-agent 协助进行公钥用户认证。
 *
 * 如果成功则返回 0，否则返回负值表示错误。
 */
LIBSSH2_API int
libssh2_agent_userauth(LIBSSH2_AGENT *agent,
               const char *username,
               struct libssh2_agent_publickey *identity);

/*
 * libssh2_agent_disconnect()
 *
 * 关闭与 ssh-agent 的连接。
 *
 * 如果成功则返回 0，否则返回负值表示错误。
 */
LIBSSH2_API int
libssh2_agent_disconnect(LIBSSH2_AGENT *agent);

/*
 * libssh2_agent_free()
 *
 * 释放 ssh-agent 句柄。此函数还会释放内部的公钥集合。
 */
LIBSSH2_API void
libssh2_agent_free(LIBSSH2_AGENT *agent);

/*
 * libssh2_agent_set_identity_path()
 *
 * 允许设置自定义的代理身份验证套接字路径，超出 SSH_AUTH_SOCK 环境变量。
 *
 */
LIBSSH2_API void
libssh2_agent_set_identity_path(LIBSSH2_AGENT *agent,
                                const char *path);

/*
 * libssh2_agent_get_identity_path()
 *
 * 如果设置了自定义代理身份验证套接字路径，则返回该路径。
 *
 */
LIBSSH2_API const char *
libssh2_agent_get_identity_path(LIBSSH2_AGENT *agent);

/*
 * libssh2_keepalive_config()
 *
 * 设置发送保持活动消息的频率。WANT_REPLY 表示保持活动消息是否应该请求服务器的响应。
 * INTERVAL 是可以在没有任何 I/O 的情况下经过的秒数，使用 0（默认值）来禁用保持活动消息。
 * 为了避免一些繁忙循环的特殊情况，如果指定间隔为 1，则将视为 2。
 *
 * 非阻塞应用程序负责使用 libssh2_keepalive_send() 发送保持活动消息。
 */
LIBSSH2_API void libssh2_keepalive_config(LIBSSH2_SESSION *session,
                                          int want_reply,
                                          unsigned interval);
/*
 * libssh2_keepalive_send()
 *
 * Send a keepalive message if needed.  SECONDS_TO_NEXT indicates how
 * many seconds you can sleep after this call before you need to call
 * it again.  Returns 0 on success, or LIBSSH2_ERROR_SOCKET_SEND on
 * I/O errors.
 */
// 发送心跳消息以保持连接，SECONDS_TO_NEXT 表示在下次调用之前可以休眠多少秒。成功返回 0，I/O 错误返回 LIBSSH2_ERROR_SOCKET_SEND
LIBSSH2_API int libssh2_keepalive_send(LIBSSH2_SESSION *session,
                                       int *seconds_to_next);

/* NOTE NOTE NOTE
   libssh2_trace() has no function in builds that aren't built with debug
   enabled
 */
// 注意：在没有启用调试的构建中，libssh2_trace() 函数没有功能
LIBSSH2_API int libssh2_trace(LIBSSH2_SESSION *session, int bitmask);
#define LIBSSH2_TRACE_TRANS (1<<1)
#define LIBSSH2_TRACE_KEX   (1<<2)
#define LIBSSH2_TRACE_AUTH  (1<<3)
#define LIBSSH2_TRACE_CONN  (1<<4)
#define LIBSSH2_TRACE_SCP   (1<<5)
#define LIBSSH2_TRACE_SFTP  (1<<6)
#define LIBSSH2_TRACE_ERROR (1<<7)
#define LIBSSH2_TRACE_PUBLICKEY (1<<8)
#define LIBSSH2_TRACE_SOCKET (1<<9)

typedef void (*libssh2_trace_handler_func)(LIBSSH2_SESSION*,
                                           void *,
                                           const char *,
                                           size_t);
LIBSSH2_API int libssh2_trace_sethandler(LIBSSH2_SESSION *session,
                                         void *context,
                                         libssh2_trace_handler_func callback);

#ifdef __cplusplus
} /* extern "C" */
#endif

#endif /* !RC_INVOKED */

#endif /* LIBSSH2_H */
```