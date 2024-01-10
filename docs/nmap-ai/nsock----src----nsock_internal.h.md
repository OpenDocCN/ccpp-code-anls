# `nmap\nsock\src\nsock_internal.h`

```
/* $Id$ */

#ifndef NSOCK_INTERNAL_H
#define NSOCK_INTERNAL_H

#include <nbase.h>

#ifdef HAVE_CONFIG_H
#include "nsock_config.h"
#include "nbase_config.h"
#endif

#ifdef WIN32
#include "nbase_winconfig.h"
#include <Winsock2.h>
#endif

#include "gh_list.h"
#include "gh_heap.h"
#include "filespace.h"
#include "nsock.h" /* The public interface -- I need it for some enum defs */
#include "nsock_ssl.h"
#include "nsock_proxy.h"

#if HAVE_SYS_TIME_H
#include <sys/time.h>
#endif
#if HAVE_UNISTD_H
#include <unistd.h>
#endif
#if HAVE_SYS_SOCKET_H
#include <sys/socket.h>
#endif
#if HAVE_NETINET_IN_H
#include <netinet/in.h>
#endif
#if HAVE_ARPA_INET_H
#include <arpa/inet.h
#endif
#if HAVE_STRING_H
#include <string.h>
#endif
#if HAVE_STRINGS_H
#include <strings.h>
#endif
#if HAVE_SYS_UN_H
#include <sys/un.h>
#endif

#ifndef IPPROTO_SCTP
#define IPPROTO_SCTP 132
#endif


/* ------------------- CONSTANTS ------------------- */
#define READ_BUFFER_SZ 8192

enum nsock_read_types {
  NSOCK_READLINES,  // 读取行
  NSOCK_READBYTES,  // 读取字节
  NSOCK_READ        // 读取
};

enum iod_state {
  NSIOD_STATE_DELETED,  // 删除状态
  NSIOD_STATE_INITIAL,  // 初始状态

  /* sd was provided to us in nsock_iod_new2 (see nsock_iod.c) */
  NSIOD_STATE_UNKNOWN,  // 未知状态

  NSIOD_STATE_CONNECTED_TCP,  // TCP 连接状态
  NSIOD_STATE_CONNECTED_UDP  // UDP 连接状态
};

/* XXX: ensure that these values can be OR'ed when adding new ones */
#define EV_NONE   0x00
#define EV_READ   0x01  // 读事件
#define EV_WRITE  0x02  // 写事件
#define EV_EXCEPT 0x04  // 异常事件


/* ------------------- STRUCTURES ------------------- */

struct readinfo {
  enum nsock_read_types read_type;  // 读取类型
  /* num lines; num bytes; whatever (depends on read_type) */
  int num;  // 行数或字节数，取决于读取类型
};

struct writeinfo {
  struct sockaddr_storage dest;  // 目标地址
  size_t destlen;  // 目标地址长度
  /* Number of bytes successfully written */
  int written_so_far;  // 已成功写入的字节数
};

/* Remember that callers of this library should NOT be accessing these
 * fields directly */
# 定义一个结构体 npool，用于管理网络事件和相关数据
struct npool {
  /* 用户数据，如果未设置则为 NULL */
  void *userdata;

  /* IO 引擎的虚拟表 */
  struct io_engine *engine;
  /* IO 引擎的内部数据 */
  void *engine_data;

  /* 活动的网络连接事件 */
  gh_list_t connect_events;
  /* 活动的读取事件 */
  gh_list_t read_events;
  /* 活动的写入事件 */
  gh_list_t write_events;
#if HAVE_PCAP
  /* 活动的 pcap 读取事件 */
  gh_list_t pcap_read_events;
#endif
  /* 过期事件的堆 */
  gh_heap_t expirables;

  /* 活动的 IODs 和相关事件列表 */
  gh_list_t active_iods;

  /* 已释放以供重用的 niod 结构 */
  gh_list_t free_iods;
  /* 删除的事件将被放置在此处以供以后重用 */
  gh_list_t free_events;

  /* 所有列表上挂起事件的数量（总数） */
  int events_pending;

  /* 下一个事件的序列号（用于创建下一个 nsock_event_id） */
  unsigned long next_event_serial;
  /* 下一个要创建的 iod 的序列号 */
  unsigned long next_iod_serial;

  /* 如果 nsock_loop() 返回 NSOCK_LOOP_ERROR，则在此处描述错误（errnum 样式） */
  int errnum;

  /* 如果为 true，则新套接字将设置 SO_BROADCAST */
  int broadcast;

  /* 要绑定的接口；仅在 Linux 上支持使用 SO_BINDTODEVICE sockopt。 */
  const char *device;

  /* 如果为 true，则退出 nsock_loop 的下一个迭代，状态为 NSOCK_LOOP_QUIT。 */
  int quit;

#if HAVE_OPENSSL
  /* SSL 上下文（选项等） */
  SSL_CTX *sslctx;
#ifdef HAVE_DTLS_CLIENT_METHOD
  SSL_CTX *dtlsctx;
#endif
#endif

  /* 可选的代理链（NULL 表示未设置）。每个 NSP 只能设置一次（使用 nsock_proxychain_new() 或 nsock_pool_set_proxychain()）。 */
  struct proxy_chain *px_chain;

};


/* nsock_iod 类似于 nsock 库的“文件描述符”。您可以使用它来请求事件。 */
struct niod {
  /* 与事件相关的套接字描述符 */
  int sd;

  /* 此 iod 上挂起事件的数量 */
  int events_pending;

  /* 挂起的事件 */
  gh_lnode_t *first_connect;
  gh_lnode_t *first_read;
  gh_lnode_t *first_write;
#if HAVE_PCAP
  gh_lnode_t *first_pcap_read;
#endif

  // 读取操作计数
  int readsd_count;
  // 写入操作计数
  int writesd_count;
#if HAVE_PCAP
  // 读取 pcap 数据包操作计数
  int readpcapsd_count;
#endif

  // 监视的事件数量
  int watched_events;

  /* 用于创建 iod 的 npool 结构 */
  struct npool *nsp;

  // 当前状态
  enum iod_state state;

  // 与 sd 连接的主机和端口
  struct sockaddr_storage peer;
  // 用 sd 绑定的本地主机和端口
  struct sockaddr_storage local;

  // peer/local 实际使用的长度
  size_t locallen;
  size_t peerlen;

  // 协议类型
  int lastproto;

  // npool 结构用于跟踪已分配的 NIODs，以便在删除 msp 时销毁它们
  gh_lnode_t nodeq;

#define IOD_REGISTERED  0x01
#define IOD_PROCESSED   0x02    /* engine_kqueue.c 内部使用 */

#define IOD_PROPSET(iod, flag)  ((iod)->_flags |= (flag))
#define IOD_PROPCLR(iod, flag)  ((iod)->_flags &= ~(flag))
#define IOD_PROPGET(iod, flag)  (((iod)->_flags & (flag)) != 0)
  char _flags;

  // 用于 SSL Server Name Indication
  char *hostname;

#if HAVE_OPENSSL
  // SSL 连接
  SSL *ssl;
  // SSL 会话 ID
  SSL_SESSION *ssl_session;
#else
  // 因为代码中有许多 if (nsi->ssl) 情况
  char *ssl;
#endif
  /* 每个 iod 都有一个 id，在同一个 nspool 中始终是唯一的（除非创建了数十亿个） */
  unsigned long id;

  /* 从 sd 中读取的字节数 */
  unsigned long read_count;
  /* 写入 sd 的字节数 */
  unsigned long write_count;

  void *userdata;

  /* 在 connect() 之前设置在套接字上的 IP 选项 */
  void *ipopts;
  int ipoptslen;

  /* 指向 mspcap 结构的指针（仅在包含 pcap 支持时使用） */
  void *pcap;

  struct proxy_chain_context *px_ctx;

};


/* nsock_event_t 处理单个事件。当事件创建时，通常会返回其 ID，并且该事件将包含在回调中 */
struct nevent {
  /* 每个事件都有一个 ID，在给定的 nsock 中是唯一的，除非超过了 5 亿个事件 */
  nsock_event_id id;

  enum nse_type type;
  enum nse_status status;

  /* 对于写事件，这是要写入的数据，对于读事件，这是我们将要读取的数据 */
  struct filespace iobuf;

  /* 事件的超时时间 -- 绝对时间，除非 tv_sec == 0 表示没有超时 */
  struct timeval timeout;

  /* 与读取请求相关的信息 */
  struct readinfo readinfo;
  /* 与写入请求相关的信息 */
  struct writeinfo writeinfo;

#if HAVE_OPENSSL
  struct sslinfo sslinfo;
#endif

  /* 如果返回 NSE_STATUS_ERROR 状态，则必须设置此值 */
  int errnum;

  /* 与事件相关的 nsock I/O 描述符（如果适用） */
  struct niod *iod;

  /* 事件完成时要调用的处理程序 */
  nsock_ev_handler handler;

  /* 在可过期的二叉堆中的位置 */
  gh_hnode_t expire;

  /* 由于某些原因（参见 nsock_pcap.c），在 PCAP_BSD_SELECT_HACK 模式下，我们将 pcap 事件注册为读取和 pcap 读取事件。然后我们需要两个 gh_lnode_t 句柄。为了使代码更简单，我们总是使用 _nodeq_pcap 用于 pcap 读取事件和 _nodeq_io 用于其他事件。
   * 当不在 PCAP_BSD_SELECT_HACK 模式下，我们将两个句柄定义为联合的成员，以优化内存占用。 */
  gh_lnode_t nodeq_io;
  gh_lnode_t nodeq_pcap;

  /* 可选的（如果未设置则为 NULL）指针，传递给处理程序 */
  void *userdata;

  /* 如果此事件已全部填充并准备立即传递，则 event_done 为非零。在意外时间完成事件时使用，我们希望稍后调度它以避免重复的状态更新代码和其他所有那些烦人的东西 */
  unsigned int event_done: 1;
  unsigned int eof: 1;

#if HAVE_IOCP
  struct extended_overlapped *eov;
#endif
};

struct io_operations {
  int(*iod_connect)(struct npool *nsp, int sockfd, const struct sockaddr *addr, socklen_t addrlen);

  int(*iod_read)(struct npool *nsp, int sockfd, void *buf, size_t len, int flags,
    struct sockaddr *src_addr, socklen_t *addrlen);

  int(*iod_write)(struct npool *nsp, int sockfd, const void *buf, size_t len, int flags,
    const struct sockaddr *dest_addr, socklen_t addrlen);
};
# 定义了一个结构体 io_engine，用于表示 I/O 引擎
struct io_engine {
  /* 人类可读的引擎标识符 */
  const char *name;

  /* 引擎构造函数 */
  int (*init)(struct npool *nsp);

  /* 引擎析构函数 */
  void (*destroy)(struct npool *nsp);

  /* 向引擎注册新的 IOD */
  int(*iod_register)(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev);

  /* 移除已注册的 IOD */
  int(*iod_unregister)(struct npool *nsp, struct niod *iod);

  /* 修改已注册的 IOD 的事件。
   *  - ev_set 代表要添加的事件
   *  - ev_clr 代表要删除的事件（如果设置） */
  int (*iod_modify)(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev_set, int ev_clr);

  /* 主引擎循环 */
  int (*loop)(struct npool *nsp, int msec_timeout);

  /* I/O 操作 */
  struct io_operations *io_operations;
};

/* ----------- NSOCK I/O ENGINE CONVENIENCE WRAPPERS ------------ */

# 初始化 NSOCK 引擎
static inline int nsock_engine_init(struct npool *nsp) {
  return nsp->engine->init(nsp);
}

# 销毁 NSOCK 引擎
static inline void nsock_engine_destroy(struct npool *nsp) {
  nsp->engine->destroy(nsp);
  return;
}

# 向 NSOCK 引擎注册新的 IOD
static inline int nsock_engine_iod_register(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev) {
  return nsp->engine->iod_register(nsp, iod, nse, ev);
}

# 移除已注册的 IOD
static inline int nsock_engine_iod_unregister(struct npool *nsp, struct niod *iod) {
  return nsp->engine->iod_unregister(nsp, iod);
}

# 修改已注册的 IOD 的事件
static inline int nsock_engine_iod_modify(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev_set, int ev_clr) {
  return nsp->engine->iod_modify(nsp, iod, nse, ev_set, ev_clr);
}

# NSOCK 引擎的主循环
static inline int nsock_engine_loop(struct npool *nsp, int msec_timeout) {
  return nsp->engine->loop(nsp, msec_timeout);
}

/* ------------------- PROTOTYPES ------------------- */

# 事件是否超时
int event_timedout(struct nevent *nse);

# 获取一个新的 nsock_event_id，给定一个类型
nsock_event_id get_new_event_id(struct npool *nsp, enum nse_type type);
# 根据事件ID返回事件类型（NSE_TYPE_CONNECT等）
enum nse_type get_event_id_type(nsock_event_id event_id);

# 创建一个新的事件结构体，必须使用event_delete稍后删除，除非返回NULL（表示失败）。如果struct niod和userdata不可用，可以传入NULL。
struct nevent *event_new(struct npool *nsp, enum nse_type type, struct niod *iod,
                           int timeout_msecs, nsock_ev_handler handler, void *userdata);

# 用于取消事件的内部函数，当已经有struct nevent指针时使用（如果只有ID，则使用nsock_event_cancel）。传入的event_list应该对应于事件的类型。例如，对于NSE_TYPE_READ，应该传入&iod->read_events。elem是event_list中保存事件的列表元素。如果要通知拥有事件的程序已被取消，则传入非零值。
int nevent_delete(struct npool *nsp, struct nevent *nse, gh_list_t *event_list, gh_lnode_t *elem, int notify);

# 调整各种统计数据，分派事件处理程序（如果notify非零），然后删除事件。此函数不会从任何可能存在的列表中删除事件（例如nsp->read_list等）。调用此函数时，nse->event_done必须为true。
void event_dispatch_and_delete(struct npool *nsp, struct nevent *nse, int notify);

# 释放使用event_new分配的struct nevent，包括所有内部资源。注意 - 我们假设nse->iod->events_pending（如果存在）已经被递减（在event_dispatch_and_delete期间完成）- 因此如果直接调用event_delete()，请记得这样做。
void event_delete(struct npool *nsp, struct nevent *nse);

# 将事件添加到适当的nsp事件列表中，处理诸如调整描述符选择/轮询列表、注册超时值等的工作。
/* 将事件添加到事件池中 */
void nsock_pool_add_event(struct npool *nsp, struct nevent *nse);

/* 内部连接函数，用于处理连接事件 */
void nsock_connect_internal(struct npool *ms, struct nevent *nse, int type, int proto, struct sockaddr_storage *ss, size_t sslen, unsigned int port);

/* 处理连接结果的函数，假设 select 或 poll 已经显示描述符处于活动状态 */
void handle_connect_result(struct npool *ms, struct nevent *nse, enum nse_status status);

/* 处理读取结果的函数 */
void handle_read_result(struct npool *ms, struct nevent *nse, enum nse_status status);

/* 处理写入结果的函数 */
void handle_write_result(struct npool *ms, struct nevent *nse, enum nse_status status);

/* 处理定时器结果的函数 */
void handle_timer_result(struct npool *ms, struct nevent *nse, enum nse_status status);

#if HAVE_PCAP
/* 处理 pcap 读取结果的函数 */
void handle_pcap_read_result(struct npool *ms, struct nevent *nse, enum nse_status status);
#endif

/* 事件已完成，即将调用处理程序。如果需要，此函数会写出有关事件的跟踪数据 */
void nsock_trace_handler_callback(struct npool *ms, struct nevent *nse);

#if HAVE_OPENSSL
/* 设置 nsock_iod 的 SSL 会话，增加使用计数 */
void nsi_set_ssl_session(struct niod *iod, SSL_SESSION *sessid);
#endif

/* 获取下一个即将过期的事件 */
static inline struct nevent *next_expirable_event(struct npool *nsp) {
  gh_hnode_t *hnode;

  hnode = gh_heap_min(&nsp->expirables);
  if (!hnode)
    return NULL;

  return container_of(hnode, struct nevent, expire);
}

/* 从链表节点获取事件 */
static inline struct nevent *lnode_nevent(gh_lnode_t *lnode) {
  return container_of(lnode, struct nevent, nodeq_io);
}

/* 从链表节点获取事件（另一种情况） */
static inline struct nevent *lnode_nevent2(gh_lnode_t *lnode) {
  return container_of(lnode, struct nevent, nodeq_pcap);
}

#endif /* NSOCK_INTERNAL_H */
```