# `nmap\nsock\include\nsock.h`

```cpp
/* $Id$ */

#ifndef NSOCK_H
#define NSOCK_H

/* Keep assert() defined for security reasons */
#undef NDEBUG

#ifndef WIN32
#include "nsock_config.h"
#else
#include "nsock_winconfig.h"
#endif

#include <stdio.h>
#include <sys/types.h>
#ifndef WIN32
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <sys/time.h>
#else
#include <winsock2.h>   /* for struct timeval... */
#endif

#if HAVE_SYS_UN_H
#include <sys/un.h>

#ifndef SUN_LEN
#include <string.h>
#define SUN_LEN(ptr) ((sizeof(*(ptr)) - sizeof((ptr)->sun_path))    \
                      + strlen((ptr)->sun_path))
#endif
#endif  /* HAVE_SYS_UN_H */

#if HAVE_LINUX_VM_SOCKETS_H
#include <linux/vm_sockets.h>
#endif

#ifdef __cplusplus
extern "C" {
#endif

/* The read calls will generally return after reading at least this
 * much data so that the caller can process it and so that the
 * connection spewing data doesn't monopolize resources. The caller
 * can always initiate another read request to ask for more. */
#define NSOCK_READ_CHUNK_SIZE 0x8FFFF

struct npool;
struct niod;
struct nevent;
struct proxy_chain;

/* ------------------- TYPEDEFS ------------------- */

/* nsock_pool, nsock_iod, and nsock_event are opaque objects that should
 * only be accessed using the appropriate accessor functions (described below). */

/* An nsock_pool aggregates and manages events and i/o descriptors */
typedef struct npool *nsock_pool;

/* nsock_iod is an I/O descriptor -- you create it and then use it to
 * make calls to do connect()s, read()s, write()s, etc. A single IOD can handle
 * multiple event calls, but only one at a time. Also the event calls must be in
 * a "reasonable" order. For example, you might start with nsock_connect_tcp()
 * followed by a bunch of nsock_read* and nsock_write* calls.  Then you either
 * destroy the iod for good with nsock_iod_delete() and allocate a new one via
 * nsock_iod_new for your next connection. */
typedef struct niod *nsock_iod;
/* An event is created when you do various calls (for reading, writing,
 * connecting, timers, etc) and is provided back to you in the callback when the
 * call completes/fails. It is automatically destroyed after the callback */
typedef struct nevent *nsock_event;

/* Provided by calls which (internally) create an nsock_event.  This allows you
 * to cancel the event */
typedef unsigned long nsock_event_id;

/* This is used to save SSL sessionids between SSL connections */
typedef void *nsock_ssl_session;
typedef void *nsock_ssl_ctx;
typedef void *nsock_ssl;

typedef struct proxy_chain *nsock_proxychain;


/* Logging-related data structures */

typedef enum {
  /* --
   * Actual message priority values */
  NSOCK_LOG_DBG_ALL,
  NSOCK_LOG_DBG,
  NSOCK_LOG_INFO,
  NSOCK_LOG_ERROR,
  /* --
   * No messages are issued by nsock with this value.
   * Users can therefore set loglevel to NSOCK_LOG_NONE
   * to disable logging */
  NSOCK_LOG_NONE
} nsock_loglevel_t;

struct nsock_log_rec {
  /* Message emission time */
  struct timeval time;
  /* Message log level */
  nsock_loglevel_t level;
  /* Source file */
  const char *file;
  /* Statement line in nsock source */
  int line;
  /* Function that emitted the message */
  const char *func;
  /* Actual log message */
  char *msg;
};

/* Nsock logging function. This function receives all nsock log records whose
 * level is greater than or equal to nsp loglevel. The rec structure is
 * allocated and freed by nsock. */
typedef void (*nsock_logger_t)(const struct nsock_log_rec *rec);


/* ------------------- PROTOTYPES ------------------- */
# 这是一个非常重要的循环函数，告诉事件引擎启动并开始处理事件。它将一直持续，直到所有事件都被传递（包括从事件处理程序启动的新事件），或者达到了msec_timeout，或者发生了重大错误。如果不想设置它运行的最长时间，则使用-1。超时为0将在1个非阻塞循环后返回。nsock循环可以在返回后重新启动。例如，您可以进行一系列15秒的运行，允许您在它们之间做其他事情。或者您可以安排一个定时器每15秒调用您回来。
enum nsock_loopstatus {
  NSOCK_LOOP_NOEVENTS = 2,  # 没有事件
  NSOCK_LOOP_TIMEOUT,  # 超时
  NSOCK_LOOP_ERROR,  # 错误
  NSOCK_LOOP_QUIT  # 退出
};

# 调用此函数将导致nsock_loop在其下一个迭代中以NSOCK_LOOP_QUIT的返回值退出。
void nsock_loop_quit(nsock_pool nsp);

# 此下一个函数返回errno样式的错误代码——仅当nsock_loop()返回NSOCK_LOOP_ERROR状态时才有效。
int nsock_pool_get_error(nsock_pool nsp);

# 获取nsock_iod的SSL对象
nsock_ssl nsock_iod_get_ssl(nsock_iod nsockiod);

# 注意，如果inc_ref不为零，nsock_iod_get_ssl_session将增加SSL_SESSION的使用计数，因为nsock在销毁IOD时会释放它。任何调用函数/等都要对其进行SSL_SESSION_free()。传递inc_ref=0不会增加，仅用于信息目的。
nsock_ssl_session nsock_iod_get_ssl_session(nsock_iod nsockiod, int inc_ref);

# 有时在NSP中存储指向信息的指针是有用的，这样您就可以在回调期间检索它。
void nsock_pool_set_udata(nsock_pool nsp, void *data);

# 如果没有一种方法来检索该数据，上面的函数就没有多大意义...
void *nsock_pool_get_udata(nsock_pool nsp);
/* Turns on or off broadcast support on new sockets. Default is off (0, false)
 * set in nsock_pool_new(). Any non-zero (true) value sets SO_BROADCAST on all
 * new sockets (value of optval will be used directly in the setsockopt() call). */
void nsock_pool_set_broadcast(nsock_pool nsp, int optval);

/* Sets the name of the interface for new sockets to bind to. */
void nsock_pool_set_device(nsock_pool nsp, const char *device);

/* Initializes an Nsock pool to create SSL connections. This sets an internal
 * SSL_CTX, which is like a template that sets options for all connections that
 * are made from it. Returns the SSL_CTX so you can set your own options.
 *
 * Use the NSOCK_SSL_MAX_SPEED to emphasize speed over security.
 * Insecure ciphers are used when they are faster and no certificate
 * verification is done.
 *
 * Returns the SSL_CTX so you can set your own options.
 * By default, do no server certificate verification. To enable it, do
 * something like:
 *    SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER, NULL);
 *
 *  on the SSL_CTX returned. If you do, it is then up to the application to
 *  load trusted certificates with SSL_CTX_load_verify_locations or
 *  SSL_CTX_set_default_verify_paths, or else every connection will fail. It
 *  is also up to the application to do any further checks such as domain name
 *  validation. */
#define NSOCK_SSL_MAX_SPEED (1 << 0)
nsock_ssl_ctx nsock_pool_ssl_init(nsock_pool ms_pool, int flags);

/* Initializes an Nsock pool to create a DTLS connect. This sets and internal
 * SSL_CTX, which is like a template that sets options for all connections that
 * are made from it. Returns the SSL_CTX so tyou can set your own options.
 *
 * Functionally similar to nsock_pool_ssl_init, just for the DTLS */
nsock_ssl_ctx nsock_pool_dtls_init(nsock_pool ms_pool, int flags);
# 强制使用给定的 IO 引擎
# engine 参数是一个以零结尾的字符串，将由库的 strup() 函数处理
# 此函数不执行有效性检查，请注意，在调用 nsock_pool_new() 之前，如果提供了无效/不可用的引擎名称，将导致致命错误
# 传入 NULL 以重置为默认值（使用最有效的可用引擎）
# 函数在成功时返回 0，错误时返回 -1
int nsock_set_default_engine(char *engine);

# 获取可用引擎的逗号分隔列表
const char *nsock_list_engines(void);

# 创建 nsock_pool 的方法
# 分配、初始化并返回一个 nsock_pool 事件聚合器
# 在发生错误的情况下，将返回 NULL
# 如果不希望立即关联任何用户数据，请传入 NULL
nsock_pool nsock_pool_new(void *udata);

# 如果 nsock_pool_new 返回成功，则在使用完毕后必须释放 nsp 以节省内存（以及在某些情况下，套接字）
# 在此调用之后，nsp 可能不再可用
# 任何待处理事件都会收到 NSE_STATUS_KILL 回调，并删除所有未完成的 IODs
void nsock_pool_delete(nsock_pool nsp);

# 日志子系统：设置自定义日志函数
# 一个空的日志记录器将重置默认（stderr）日志记录器
# （参见 nsock_logger_t 类型定义）
void nsock_set_log_function(nsock_logger_t logger);

# 获取日志级别
nsock_loglevel_t nsock_get_loglevel(void);
# 设置日志级别
void nsock_set_loglevel(nsock_loglevel_t loglevel);

# 解析代理链描述字符串并构建相应的 nsock_proxychain 对象
# 如果传入了可选的 nsock_pool 参数，则将其关联到链对象
# 另一种方法是传入 nsp=NULL 并手动调用 nsock_pool_set_proxychain()
# 无论采取何种方式，都必须由调用者删除链对象，使用 proxychain_delete()
# 成功时返回 1，失败时返回 -1
int nsock_proxychain_new(const char *proxystr, nsock_proxychain *chain, nsock_pool nspool);
/* If nsock_proxychain_new() returned success, caller has to free the chain
 * object using this function. */
// 如果 nsock_proxychain_new() 返回成功，调用者必须使用此函数释放链对象

void nsock_proxychain_delete(nsock_proxychain chain);
// 释放之前创建的代理链对象

/* Assign a previously created proxychain object to a nsock pool. After this,
 * new connections requests will be issued through the chain of proxies (if
 * possible). This only applies to nsock_iod created *after* the call to
 * nsock_pool_set_proxychain(). Existing nsock_iod will connect as normal. */
// 将之前创建的代理链对象分配给 nsock 池。之后，新的连接请求将通过代理链发出（如果可能的话）。这仅适用于在调用 nsock_pool_set_proxychain() 之后创建的 nsock_iod。现有的 nsock_iod 将像往常一样连接。
int nsock_pool_set_proxychain(nsock_pool nspool, nsock_proxychain chain);

/* nsock_event handles a single event.  Its ID is generally returned when the
 * event is created, and the event itself is included in callbacks
 *
 * ---------------------------------------------------------------------------
 * IF YOU ADD NEW NSE_TYPES YOU MUST INCREASE TYPE_CODE_NUM_BITS SO THAT IT IS
 * ALWAYS log2(maximum_nse_type_value + 1)
 * --------------------------------------------------------------------------- */
// nsock_event 处理单个事件。通常在创建事件时返回其 ID，并且事件本身包含在回调中
// 如果添加新的 NSE_TYPES，必须增加 TYPE_CODE_NUM_BITS，以便它始终是 log2(maximum_nse_type_value + 1)

#define TYPE_CODE_NUM_BITS 3
enum nse_type {
  NSE_TYPE_CONNECT = 0,
  NSE_TYPE_CONNECT_SSL = 1,
  NSE_TYPE_READ = 2,
  NSE_TYPE_WRITE = 3,
  NSE_TYPE_TIMER = 4,
  NSE_TYPE_PCAP_READ = 5,
  NSE_TYPE_MAX = 6,
};  /* At some point I was considering a NSE_TYPE_START and NSE_TYPE_CUSTOM */
// 定义枚举类型 nse_type，包括不同的事件类型

/* Find the type of an event that spawned a callback */
// 查找生成回调的事件类型
enum nse_type nse_type(nsock_event nse);

/* Takes an nse_type (as returned by nse_type()) and returns a static string name
 * that you can use for printing, etc. */
// 获取 nse_type（由 nse_type() 返回）并返回一个静态字符串名称，可用于打印等
const char *nse_type2str(enum nse_type type);

/* Did the event succeed?  What is the status? */
// 事件是否成功？状态是什么？
# 定义枚举类型 nse_status，表示网络事件的状态
enum nse_status {
  NSE_STATUS_NONE = 0,  /* 用户不应该看到这个状态 */
  NSE_STATUS_SUCCESS,   /* 一切正常！ */
  NSE_STATUS_ERROR,     /* 出现问题，请检查 nse_errorcode() */
  NSE_STATUS_TIMEOUT,   /* 异步调用超过了指定的超时时间 */
  NSE_STATUS_CANCELLED, /* 有人取消了事件（通过调用 nsock_event_cancel()） */
  NSE_STATUS_KILL,      /* 事件已被终止，通常意味着 nspool 被删除，应该释放已分配的资源并退出 */
  NSE_STATUS_EOF,       /* 我们收到了 EOF 但没有数据，如果我们先收到数据，则报告 SUCCESS（参见 nse_eof()） */
  NSE_STATUS_PROXYERROR
};

# 定义函数 nse_status，返回事件的状态
enum nse_status nse_status(nsock_event nse);

# 定义函数 nse_status2str，根据状态返回静态字符串名称，用于打印等
const char *nse_status2str(enum nse_status status);

# 定义函数 nse_eof，判断是否在读取时收到了 EOF，比查看状态更好的方式，因为有时在收到 EOF 前可能已经读取了一些数据
int nse_eof(nsock_event nse);

# 定义函数 nse_errorcode，返回类似 errno 的错误码，仅当状态为 NSE_STATUS_ERROR 时有效
int nse_errorcode(nsock_event nse);

# 每个事件都有一个唯一的 ID，在程序执行期间（对于给定的 nsock_pool）是唯一的，除非超过 500,000,000 个
nsock_event_id nse_id(nsock_event nse);
# 如果进行了读取请求，并且结果是STATUS_SUCCESS，这个函数提供了读取的缓冲区以及读取的字符数
# 缓冲区不应该被修改或释放。它不能保证以NUL结尾，甚至可能包含NUL字符
char *nse_readbuf(nsock_event nse, int *nbytes);

# 获取与事件相关联的nsock_iod（见下文）。请注意，某些事件（例如定时器）没有与它们相关联的nsock_iod
nsock_iod nse_iod(nsock_event nse);

# nsock_iod类似于nsock库的“文件描述符”。您可以使用它来请求事件。以下是如何创建nsock_iod。如果无法分配iod，则nsock_iod_new返回NULL。如果您不想立即将任何用户数据与IOD关联，可以将NULL作为udata传递
nsock_iod nsock_iod_new(nsock_pool nsockp, void *udata);

# 此版本允许您将现有的sd与msi关联，以便您可以使用nsock基础设施进行读/写。例如，您可能希望同时监视来自STDIN_FILENO的数据，同时读/写各种套接字。但是，STDIN_FILENO是一个特殊情况。任何其他sd都会被dup()，因此您可以关闭或以其他方式操作您的副本。复制的副本将在IOD被销毁时被销毁
nsock_iod nsock_iod_new2(nsock_pool nsockp, int sd, void *udata);

# 如果nsock_iod_new返回成功，则在使用完毕后必须释放iod以节省内存（在某些情况下，也是套接字）。在此调用之后，nsockiod可能不再使用--您需要使用nsock_iod_new()创建一个新的
# pending_response告诉如何处理任何挂起在此nsock_iod上的事件。这可以是NSOCK_PENDING_NOTIFY（向每个事件发送KILL通知），NSOCK_PENDING_SILENT（不向被杀死的事件发送通知），或NSOCK_PENDING_ERROR（打印错误消息并退出程序）
# 定义枚举类型，表示删除模式
enum nsock_del_mode {
  NSOCK_PENDING_NOTIFY,  # 待通知状态
  NSOCK_PENDING_SILENT,  # 待静默状态
  NSOCK_PENDING_ERROR,   # 待错误状态
};

# 删除指定的 nsock_iod 对象
void nsock_iod_delete(nsock_iod iod, enum nsock_del_mode pending_response);

# 在 nsiod 中存储指向信息的指针，以便在回调期间检索
void nsock_iod_set_udata(nsock_iod iod, void *udata);

# 获取存储在 nsiod 中的数据
void *nsock_iod_get_udata(nsock_iod iod);

# 获取套接字描述符，通常用于设置套接字接收缓冲区大小
int nsock_iod_get_sd(nsock_iod iod);

# 返回 nsock_iod 的 ID，对于给定的 nspool，此 ID 始终是唯一的
unsigned long nsock_iod_id(nsock_iod iod);

# 返回接收的数据包字节数
unsigned long nsock_iod_get_read_count(nsock_iod iod);

# 返回发送的数据包字节数
unsigned long nsock_iod_get_write_count(nsock_iod iod);

# 检查 nsock_iod 是否通过 SSL 进行通信，返回 1 表示是，0 表示否
int nsock_iod_check_ssl(nsock_iod iod);

# 返回远程对等端口（如果不可用则返回-1），端口以主机字节顺序返回
int nsock_iod_get_peerport(nsock_iod iod);

# 在连接之前设置要绑定的本地地址
int nsock_iod_set_localaddr(nsock_iod iod, struct sockaddr_storage *ss, size_t sslen);
# 设置在连接之前应用的 IPv4 选项。它会复制选项，所以如果需要的话可以释放你自己的选项。这个副本在 iod 销毁时会被释放
int nsock_iod_set_ipoptions(nsock_iod iod, void *ipopts, size_t ipoptslen);

# 返回该 nsi 最后一次通信（或通信尝试）涉及的主机/端口/协议信息。通过 "涉及" 我的意思是像建立（或尝试建立）连接或通过未连接的 nsock_iod 发送 UDP 数据报。AF 是地址族（AF_INET 或 AF_INET6），Protocol 是 IPPROTO_TCP 或 IPPROTO_UDP。对于不需要的信息，传递 NULL。如果请求的任何信息不可用，则返回 0，并将不可用的套接字置零。如果请求但不可用的协议或 af，则将其设置为 -1（并返回 0）。传递的指针必须为 NULL 或指向已分配的地址空间。sockaddr 成员实际上应该是 sockaddr_storage、sockaddr_in6 或 sockaddr_in，其 socklen 应适当设置（例如，如果传递的是 sizeof(sockaddr_storage)）
int nsock_iod_get_communication_info(nsock_iod iod, int *protocol, int *af,
                                     struct sockaddr *local,
                                     struct sockaddr *remote, size_t socklen);

# 设置远程主机的主机名，用于在这种情况下。目前仅在 SSL 连接中用于服务器名称指示。
int nsock_iod_set_hostname(nsock_iod iod, const char *hostname);

# 事件创建函数
# ---
# 这些函数请求异步通知事件完成。处理程序永远不会在事件创建调用期间同步回调（这会导致太多难以调试的错误，而且我们也不希望人们在实际调用 nsock_loop 之前就处理回调）
/*
 * 这些函数通常需要5个初始参数：
 * 
 *   nsock_pool mst:
 *     描述已安排的事件等的 nsock_pool
 * 
 *   nsock_iod  nsiod:
 *     应在请求中使用的 I/O 描述符。请注意，定时器事件不需要此参数，因为它们不使用 iod。您可以在 nsock_event 的回调中从 nsock_event 获取它。
 * 
 *   nsock_ev_handler handler:
 *     当触发您的事件（或超时，或出现错误等）时，您希望系统调用的函数。函数应该是这种形式：void funcname(nsock_pool nsp, nsock_event nse, void *userdata)
 * 
 *   int timeout_msecs:
 *     请求的超时时间（以毫秒为单位）。如果在指定的超时时间内请求未完成（或在少数情况下未启动），将使用 TIMEOUT 状态调用处理程序，并且请求将被中止。
 * 
 *   void *userdata:
 *     返回的 nsock_event 可以选择与其关联的指针。您可以在这里设置该指针。如果您不想要一个，只需传递 NULL。
 * 
 *   这些函数返回一个 nsock_event_id，如果需要，可以用于取消事件。
 */
typedef void (*nsock_ev_handler)(nsock_pool, nsock_event, void *);

/* 初始化一个未连接的 UDP 套接字。 */
int nsock_setup_udp(nsock_pool nsp, nsock_iod ms_iod, int af);

#if HAVE_SYS_UN_H

/* 请求到同一系统的 UNIX 域套接字连接（通过套接字路径）。
 * 此函数连接到类型为 SOCK_STREAM 的套接字。ss 应该是一个 sockaddr_storage，根据情况是 sockaddr_un（就像您将要传递给 connect 的内容一样）。sslen 应该是您传递的结构的大小。 */
# 请求通过 UNIX 域套接字流连接到同一系统（通过套接字路径）的事件
# 这个函数连接到类型为 SOCK_DGRAM 的套接字。ss 应该是一个 sockaddr_storage，根据情况也可以是一个 sockaddr_un（就像你传递给 connect 的那样）。sslen 应该是你传递的结构体的大小。
nsock_event_id nsock_connect_unixsock_stream(nsock_pool nsp, nsock_iod nsiod, nsock_ev_handler handler,
                                             int timeout_msecs, void *userdata, struct sockaddr *ss,
                                             size_t sslen);

/* 请求通过 UNIX 域套接字数据报连接到同一系统的事件。ss 应该是一个 sockaddr_storage，根据情况也可以是一个 sockaddr_un（就像你传递给 connect 的那样）。sslen 应该是你传递的结构体的大小。 */
nsock_event_id nsock_connect_unixsock_datagram(nsock_pool nsp, nsock_iod nsiod, nsock_ev_handler handler,
                                               void *userdata, struct sockaddr *ss, size_t sslen);
#endif /* HAVE_SYS_UN_H */

#if HAVE_LINUX_VM_SOCKETS_H
/* 请求通过 vsock 流连接到另一个系统的事件。ss 应该是一个 sockaddr_storage 或 sockaddr_vm，根据情况也可以是一个 sockaddr_vm（就像你传递给 connect 的那样）。sslen 应该是你传递的结构体的大小。 */
nsock_event_id nsock_connect_vsock_stream(nsock_pool nsp, nsock_iod ms_iod,
                                          nsock_ev_handler handler,
                                          int timeout_msecs, void *userdata,
                                          struct sockaddr *saddr, size_t sslen,
                                          unsigned int port);
/* 请求使用 vsock 数据报“连接”到另一个系统。由于这是一个数据报套接字，实际上并不发送任何数据包。目的地 CID 和端口只是与 nsiod 关联（实际上会调用操作系统的 connect() 调用）。然后可以在套接字上使用正常的 nsock 写入调用。由于这个调用总是在下一个机会调用您的回调函数，所以没有超时。拥有连接的数据报套接字的优势（与只使用 sendto() 指定地址相比）是我们现在可以使用一致的写入/读取调用来处理流和数据报套接字，操作系统会自动丢弃来自非伙伴的接收数据包，并且操作系统可以提供异步错误（参见《Unix 网络编程》第224页）。ss 应该是一个 sockaddr_storage 或 sockaddr_vm，视情况而定（就像您传递给 connect 的那样）。sslen 应该是您传递的结构体的大小。 */
nsock_event_id nsock_connect_vsock_datagram(nsock_pool nsp, nsock_iod nsiod,
                                            nsock_ev_handler handler,
                                            void *userdata,
                                            struct sockaddr *saddr,
                                            size_t sslen, unsigned int port);
#endif /* HAVE_LINUX_VM_SOCKETS_H */

/* 请求使用 TCP 连接到另一个系统（通过 IP 地址）。in_addr 是正常的网络字节顺序，但端口号应该以主机字节顺序给出。ss 应该是一个 sockaddr_storage、sockaddr_in6 或 sockaddr_in，视情况而定（就像您传递给 connect 的那样）。sslen 应该是您传递的结构体的大小。 */
nsock_event_id nsock_connect_tcp(nsock_pool nsp, nsock_iod nsiod, nsock_ev_handler handler, int timeout_msecs,
                                 void *userdata, struct sockaddr *ss, size_t sslen, unsigned short port);
/* 请求与另一个系统（通过IP地址）建立SCTP关联。in_addr是正常的网络字节顺序，但端口号应以主机字节顺序给出。ss应该是一个sockaddr_storage、sockaddr_in6或sockaddr_in，与传递给connect的内容相同。sslen应该是传递的结构的大小。 */
nsock_event_id nsock_connect_sctp(nsock_pool nsp, nsock_iod nsiod, nsock_ev_handler handler, int timeout_msecs,
                                  void *userdata, struct sockaddr *ss, size_t sslen, unsigned short port);

/* 请求与另一个系统（通过IP地址）建立UDP“连接”。in_addr是正常的网络字节顺序，但端口号应以主机字节顺序给出。由于这是UDP，实际上并没有发送任何数据包。目的地IP和端口只是与nsiod关联（实际上会调用OS的connect()）。然后可以在套接字上使用正常的nsock写入调用。由于这个调用总是在下一个机会调用您的回调函数，所以没有超时。与使用sendto()指定地址相比，使用连接的UDP套接字的优势是我们现在可以使用一致的写入/读取调用来处理TCP/UDP，非合作伙伴的接收数据包会被操作系统自动丢弃，并且操作系统可以提供异步错误（参见Unix网络编程pp224）。ss应该是一个sockaddr_storage、sockaddr_in6或sockaddr_in，与传递给connect的内容相同。sslen应该是传递的结构的大小。 */
nsock_event_id nsock_connect_udp(nsock_pool nsp, nsock_iod nsiod, nsock_ev_handler handler, void *userdata,
                                 struct sockaddr *ss, size_t sslen, unsigned short port);
# 请求与另一个系统（通过IP地址）建立 SSL over TCP/SCTP 连接。
# in_addr 使用正常的网络字节顺序，但端口号应该使用主机字节顺序。
# 此函数只有在建立连接并完成初始 SSL 协商后才会回调。
# 从那时起，您可以使用正常的读/写调用，解密将会透明地进行。
# ss 应该是一个适当的 sockaddr_storage、sockaddr_in6 或 sockaddr_in（就像您传递给 connect 的那样）。
# sslen 应该是您传入的结构的大小。
nsock_event_id nsock_connect_ssl(nsock_pool nsp, nsock_iod nsiod, nsock_ev_handler handler, int timeout_msecs,
                                 void *userdata, struct sockaddr *ss, size_t sslen, int proto, unsigned short port, nsock_ssl_session ssl_session);

# 请求在已建立的 TCP/SCTP 连接上进行 SSL 连接。
# nsiod 必须是已经使用 nsock_connect_tcp 或 nsock_connect_sctp 连接到目标的套接字。
# 所有参数的含义与 'nsock_connect_ssl' 中的相同。
nsock_event_id nsock_reconnect_ssl(nsock_pool nsp, nsock_iod nsiod,
                                   nsock_ev_handler handler, int timeout_msecs, void *userdata, nsock_ssl_session ssl_session);

# 读取最多 nlines 行（以 \n 结尾，当然包括 \r\n），或直到 EOF，或直到超时，以先到者为准。
# 请注意，如果至少读取了 1 个字符，EOF 或超时的情况下将返回 NSE_STATUS_SUCCESS。
# 还要注意，您可能会收到超过 'nlines' 的内容，我们只是在“至少”读取了 'nlines' 后停止。
nsock_event_id nsock_readlines(nsock_pool nsp, nsock_iod nsiod,
                               nsock_ev_handler handler, int timeout_msecs, void *userdata, int nlines);

# 与上面相同，只是它尝试读取至少 'nbytes' 而不是 'nlines'。
# 读取指定数量的字节数据，返回事件 ID
nsock_event_id nsock_readbytes(nsock_pool nsp, nsock_iod nsiod, nsock_ev_handler handler, int timeout_msecs, void *userdata, int nbytes);

# 最简单的读取函数 -- 当读取到任何内容时返回 NSE_STATUS_SUCCESS，否则根据情况返回超时、文件结束或错误
nsock_event_id nsock_read(nsock_pool nsp, nsock_iod nsiod, nsock_ev_handler handler, int timeout_msecs, void *userdata);

# 向套接字写入一些数据。如果在 timeout_msecs 内写入未完成，将返回 NSE_STATUS_TIMEOUT。如果您提供了以 NUL 结尾的数据，可以选择为 datalen 传递 -1，nsock_write 将自行计算长度
nsock_event_id nsock_write(nsock_pool nsp, nsock_iod nsiod, nsock_ev_handler handler, int timeout_msecs, void *userdata, const char *data, int datalen);

# 与 nsock_write 相同，除了您可以使用 printf 样式的格式，并且只能用于 ASCII 字符串
nsock_event_id nsock_printf(nsock_pool nsp, nsock_iod nsiod, nsock_ev_handler handler, int timeout_msecs, void *userdata, char *format, ... );

# 在指定的毫秒数后发送一个 NSE_TYPE_TIMER。当然，它也可能因错误、取消等原因返回
nsock_event_id nsock_timer_create(nsock_pool nsp, nsock_ev_handler handler, int timeout_msecs, void *userdata);
/* 取消一个事件（比如定时器或读取请求）。如果 notify 不为零，请求者将会收到一个事件取消的状态，发送到给定的处理程序。但在某些情况下，这是没有必要的（比如如果删除它的函数是创建它的函数），在这种情况下，可以传递 0 来跳过这一步。如果事件未找到，则此函数返回零，否则返回非零值 */
int nsock_event_cancel(nsock_pool ms_pool, nsock_event_id id, int notify );

/* 获取由 nsock 库记录的最新时间，该库至少在每个事件循环中（在 main_loop 中）记录一次。这个函数（通常）不仅避免了系统调用，而且在许多情况下，最好使用 nsock 的时间而不是系统时间。如果在调用它时 nsock 从未获取过时间，它将在返回之前获取时间 */
const struct timeval *nsock_gettimeofday();

#ifdef HAVE_PCAP
/* 打开 pcap 设备并将其连接到 nsp。其他参数的含义与 pcap_open_live 中的含义相同。
 *
 *   device: pcap 风格的设备名称
 *   snaplen: 要复制到处理程序的数据包大小
 *   promisc: 是否以混杂模式打开设备
 *   bpf_fmt: 伯克利过滤器
 *
 * 返回值：如果一切正常，返回 0，如果发生错误，则返回错误代码。 */
int nsock_pcap_open(nsock_pool nsp, nsock_iod nsiod, const char *pcap_device,
                    int snaplen, int promisc, const char *bpf_fmt, ...);

/* 从 pcap 中请求精确捕获一个数据包。
 * 有关参数描述，请参阅 nsock_read()。 */
nsock_event_id nsock_pcap_read_packet(nsock_pool nsp, nsock_iod nsiod,
                                      nsock_ev_handler handler,
                                      int timeout_msecs, void *userdata);
/* Gets packet data. This should be called after successful receipt of packet
 * to get packet.  If you're not interested in some values, just pass NULL
 * instead of valid pointer.
 * l3_data is just after l2_data in buffer. Feel free to treat l2_data as one
 * buffer with size of (l2_len + l3_len).
 * Ts time is fixed for systems that don't support proper timing, like Windows.
 * So TS is pointing to time when packet was received or to the time _after_.
 * As a result you'll get longer times than you should, but it's safer to
 * think that host is a bit further.
 * */
void nse_readpcap(nsock_event nsee, const unsigned char **l2_data,
                  size_t *l2_len, const unsigned char **l3_data, size_t *l3_len,
                  size_t *packet_len, struct timeval *ts);

/* Well. Just pcap-style datalink.
 * Like DLT_EN10MB or DLT_SLIP. Check in pcap(3) manpage. */
int nsock_iod_linktype(nsock_iod iod);

/* Is this nsiod a pcap descriptor? */
int nsock_iod_is_pcap(nsock_iod iod);

#endif /* HAVE_PCAP */

#ifdef __cplusplus
} /* End of 'extern "C"' */
#endif

#endif /* NSOCK_H */
```