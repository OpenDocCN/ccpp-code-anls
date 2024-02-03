# `nmap\nsock\src\nsock_connect.c`

```cpp
/* $Id$ */

#include "nsock.h"  // 包含自定义的网络套接字头文件
#include "nsock_internal.h"  // 包含自定义的网络套接字内部头文件
#include "nsock_log.h"  // 包含自定义的网络套接字日志头文件
#include "nsock_proxy.h"  // 包含自定义的网络套接字代理头文件
#include "netutils.h"  // 包含自定义的网络工具头文件

#include <sys/types.h>  // 包含系统类型头文件
#include <errno.h>  // 包含错误处理头文件
#include <string.h>  // 包含字符串处理头文件

// 设置套接字地址
static int mksock_bind_addr(struct npool *ms, struct niod *iod) {
  int rc;
  int one = 1;

  // 设置套接字选项为重用地址
  rc = setsockopt(iod->sd, SOL_SOCKET, SO_REUSEADDR, (const char *)&one, sizeof(one));
  if (rc == -1) {
    int err = socket_errno();

    // 记录设置 SO_REUSEADDR 失败的错误日志
    nsock_log_error("Setting of SO_REUSEADDR failed (#%li): %s (%d)", iod->id,
                    socket_strerror(err), err);
  }

  // 记录绑定到本地地址的信息日志
  nsock_log_info("Binding to %s (IOD #%li)", get_localaddr_string(iod), iod->id);
  // 绑定套接字到本地地址
  rc = bind(iod->sd, (struct sockaddr *)&iod->local, (int) iod->locallen);
  if (rc == -1) {
    int err = socket_errno();

    // 记录绑定失败的错误日志
    nsock_log_error("Bind to %s failed (IOD #%li): %s (%d)",
                    get_localaddr_string(iod), iod->id,
                    socket_strerror(err), err);
  }
  return 0;
}

// 设置 IP 选项
static int mksock_set_ipopts(struct npool *ms, struct niod *iod) {
  int rc;

  errno = 0;
  // 设置套接字的 IP 选项
  rc = setsockopt(iod->sd, IPPROTO_IP, IP_OPTIONS, (const char *)iod->ipopts,
                  iod->ipoptslen);
  if (rc == -1) {
    int err = socket_errno();

    // 记录设置 IP 选项失败的错误日志
    nsock_log_error("Setting of IP options failed (IOD #%li): %s (%d)",
                    iod->id, socket_strerror(err), err);
  }
  return 0;
}

// 绑定设备
static int mksock_bind_device(struct npool *ms, struct niod *iod) {
  int rc;

  // 绑定套接字到指定设备
  rc = socket_bindtodevice(iod->sd, ms->device);
  if (!rc) {
    int err = socket_errno();

    if (err != EPERM)
      // 记录设置 SO_BINDTODEVICE 失败的错误日志
      nsock_log_error("Setting of SO_BINDTODEVICE failed (IOD #%li): %s (%d)",
                      iod->id, socket_strerror(err), err);
    else
      // 记录设置 SO_BINDTODEVICE 失败的调试日志
      nsock_log_debug_all("Setting of SO_BINDTODEVICE failed (IOD #%li): %s (%d)",
                          iod->id, socket_strerror(err), err);
  }
  return 0;
}
static int mksock_set_broadcast(struct npool *ms, struct niod *iod) {
  int rc;
  int one = 1;

  // 设置套接字选项，允许广播
  rc = setsockopt(iod->sd, SOL_SOCKET, SO_BROADCAST,
                  (const char *)&one, sizeof(one));
  if (rc == -1) {
    // 如果设置失败，记录错误信息
    int err = socket_errno();
    nsock_log_error("Setting of SO_BROADCAST failed (IOD #%li): %s (%d)",
                    iod->id, socket_strerror(err), err);
  }
  return 0;
}
/* Create the actual socket (nse->iod->sd) underlying the iod. This unblocks the
 * socket, binds to the localaddr address, sets IP options, and sets the
 * broadcast flag. Trying to change these functions after making this call will
 * not have an effect. This function needs to be called before you try to read
 * or write on the iod. */
static int nsock_make_socket(struct npool *ms, struct niod *iod, int family, int type, int proto) {

  /* inheritable_socket is from nbase */
  // 创建套接字，并将其设置为可继承的
  iod->sd = (int)inheritable_socket(family, type, proto);
  if (iod->sd == -1) {
    // 如果创建套接字失败，记录错误信息
    nsock_log_error("Socket trouble: %s", socket_strerror(socket_errno()));
    return -1;
  }

  // 解除套接字的阻塞状态
  unblock_socket(iod->sd);

  // 记录最后一次使用的协议
  iod->lastproto = proto;

  // 如果有本地地址，绑定到该地址
  if (iod->locallen)
    mksock_bind_addr(ms, iod);

  // 如果有 IP 选项，并且地址族为 AF_INET，设置 IP 选项
  if (iod->ipoptslen && family == AF_INET)
    mksock_set_ipopts(ms, iod);

  // 如果有设备信息，绑定到该设备
  if (ms->device)
    mksock_bind_device(ms, iod);

  // 如果允许广播，并且套接字类型不是 SOCK_STREAM，设置广播
  if (ms->broadcast && type != SOCK_STREAM)
    mksock_set_broadcast(ms, iod);

  /* mksock_* functions can raise warnings/errors
   * but we don't let them stop us for now. */
  return iod->sd;
}

int nsock_setup_udp(nsock_pool nsp, nsock_iod ms_iod, int af) {
  struct npool *ms = (struct npool *)nsp;
  struct niod *nsi = (struct niod *)ms_iod;

  // 断言状态为初始状态或未知状态
  assert(nsi->state == NSIOD_STATE_INITIAL || nsi->state == NSIOD_STATE_UNKNOWN);

  // 记录信息，表示创建了 UDP 未连接套接字
  nsock_log_info("UDP unconnected socket (IOD #%li)", nsi->id);

  // 创建 UDP 套接字
  if (nsock_make_socket(ms, nsi, af, SOCK_DGRAM, IPPROTO_UDP) == -1)
    return -1;

  return nsi->sd;
}
/* 这个函数负责实际的连接请求逻辑，被 nsock_connect_tcp 和 nsock_connect_ssl 等函数所共享 */

void nsock_connect_internal(struct npool *ms, struct nevent *nse, int type, int proto, struct sockaddr_storage *ss, size_t sslen,
                            unsigned int port) {

  struct sockaddr_in *sin;
#if HAVE_IPV6
  struct sockaddr_in6 *sin6;
#endif
  struct niod *iod = nse->iod;

  // 如果代理已启用，并且连接类型为 TCP，并且处理程序不是 nsock_proxy_ev_dispatch
  if (iod->px_ctx   
      && proto == IPPROTO_TCP   
      && (nse->handler != nsock_proxy_ev_dispatch)) {   
    struct proxy_node *current;

    // 记录调试信息
    nsock_log_debug_all("TCP connection request (EID %lu) redirected through proxy chain",
                        (long)nse->id);

    current = iod->px_ctx->px_current;
    assert(current != NULL);

    // 保存原始目标地址和端口
    memcpy(&iod->px_ctx->target_ss, ss, sslen);
    iod->px_ctx->target_sslen = sslen;
    iod->px_ctx->target_port  = port;

    // 重定向连接到代理服务器
    ss    = &current->ss;
    sslen = current->sslen;
    port  = current->port;

    iod->px_ctx->target_handler = nse->handler;
    nse->handler = nsock_proxy_ev_dispatch;

    iod->px_ctx->target_ev_type = nse->type;
    nse->type = NSE_TYPE_CONNECT;
  }

  sin = (struct sockaddr_in *)ss;
#if HAVE_IPV6
  sin6 = (struct sockaddr_in6 *)ss;
#endif

  // 尝试建立连接
  if (nsock_make_socket(ms, iod, ss->ss_family, type, proto) == -1) {
    // 连接失败，设置事件状态和错误信息
    nse->event_done = 1;
    nse->status = NSE_STATUS_ERROR;
    nse->errnum = socket_errno();
  } else {
    // 设置端口号
    if (ss->ss_family == AF_INET) {
      sin->sin_port = htons(port);
    }
#if HAVE_IPV6
    else if (ss->ss_family == AF_INET6) {
      sin6->sin6_port = htons(port);
    }
#endif
#if HAVE_SYS_UN_H
    else if (ss->ss_family == AF_UNIX) {
      /* Unix 套接字无需额外操作 */
    }
#endif
#if HAVE_LINUX_VM_SOCKETS_H
    # 如果套接字地址家族是 AF_VSOCK，则执行以下操作
    else if (ss->ss_family == AF_VSOCK) {
      # 将套接字地址转换为 struct sockaddr_vm 类型
      struct sockaddr_vm *svm = (struct sockaddr_vm *)ss;
      # 设置 struct sockaddr_vm 结构体中的 svm_port 字段为指定的端口号
      svm->svm_port = port;
    }
#else
    else {
      // 如果地址族未知，则输出错误信息
      fatal("Unknown address family %d\n", ss->ss_family);
    }

    // 确保地址长度不超过 peer 结构体的大小
    assert(sslen <= sizeof(iod->peer));
    // 如果传入的地址不是 peer 地址，则将传入的地址复制到 peer 结构体中
    if (&iod->peer != ss)
      memcpy(&iod->peer, ss, sslen);
    // 设置 peer 地址的长度
    iod->peerlen = sslen;

    // 如果连接操作失败，则处理错误情况
    if (ms->engine->io_operations->iod_connect(ms, iod->sd, (struct sockaddr *)ss, sslen) == -1) {
      int err = socket_errno();

      // 如果是 UDP 协议或者错误码不是 EINPROGRESS 和 EAGAIN，则设置事件状态为错误，并记录错误码
      if ((proto == IPPROTO_UDP) || (err != EINPROGRESS && err != EAGAIN)) {
        nse->event_done = 1;
        nse->status = NSE_STATUS_ERROR;
        nse->errnum = err;
      }
    }
    /* The callback handle_connect_result handles the connection once it completes. */
  }
}

#if HAVE_SYS_UN_H

/* Request a UNIX domain sockets connection to the same system (by path to socket).
 * This function connects to the socket of type SOCK_STREAM.  ss should be a
 * sockaddr_storage, sockaddr_un as appropriate (just like what you would pass to
 * connect).  sslen should be the sizeof the structure you are passing in. */
nsock_event_id nsock_connect_unixsock_stream(nsock_pool nsp, nsock_iod nsiod, nsock_ev_handler handler, int timeout_msecs,
                                             void *userdata, struct sockaddr *saddr, size_t sslen) {
  struct niod *nsi = (struct niod *)nsiod;
  struct npool *ms = (struct npool *)nsp;
  struct nevent *nse;
  struct sockaddr_storage *ss = (struct sockaddr_storage *)saddr;

  // 确保连接状态为初始状态或未知状态
  assert(nsi->state == NSIOD_STATE_INITIAL || nsi->state == NSIOD_STATE_UNKNOWN);

  // 创建一个新的事件
  nse = event_new(ms, NSE_TYPE_CONNECT, nsi, timeout_msecs, handler, userdata);
  assert(nse);

  // 输出日志信息，记录连接请求的相关信息
  nsock_log_info("UNIX domain socket (STREAM) connection requested to %s (IOD #%li) EID %li",
                 get_unixsock_path(ss), nsi->id, nse->id);

  // 发起连接操作
  nsock_connect_internal(ms, nse, SOCK_STREAM, 0, ss, sslen, 0);
  // 将事件添加到事件池中
  nsock_pool_add_event(ms, nse);

  // 返回事件的 ID
  return nse->id;

}
/* 请求到同一系统的 UNIX 域套接字连接（通过套接字路径）。
 * 此函数连接到类型为 SOCK_DGRAM 的套接字。ss 应该是一个 sockaddr_storage，如适当的话也可以是 sockaddr_un（就像你传递给 connect 的那样）。
 * sslen 应该是你传递的结构体的大小。 */
nsock_event_id nsock_connect_unixsock_datagram(nsock_pool nsp, nsock_iod nsiod, nsock_ev_handler handler,
                                               void *userdata, struct sockaddr *saddr, size_t sslen) {
  struct niod *nsi = (struct niod *)nsiod;
  struct npool *ms = (struct npool *)nsp;
  struct nevent *nse;
  struct sockaddr_storage *ss = (struct sockaddr_storage *)saddr;

  // 断言，确保 nsiod 的状态为 NSIOD_STATE_INITIAL 或 NSIOD_STATE_UNKNOWN
  assert(nsi->state == NSIOD_STATE_INITIAL || nsi->state == NSIOD_STATE_UNKNOWN);

  // 创建一个新的事件
  nse = event_new(ms, NSE_TYPE_CONNECT, nsi, -1, handler, userdata);
  assert(nse);

  // 记录信息日志，表示请求连接到 UNIX 域套接字（DGRAM）到某个地址
  nsock_log_info("UNIX domain socket (DGRAM) connection requested to %s (IOD #%li) EID %li",
                 get_unixsock_path(ss), nsi->id, nse->id);

  // 内部连接到 UNIX 域套接字
  nsock_connect_internal(ms, nse, SOCK_DGRAM, 0, ss, sslen, 0);
  // 将事件添加到事件池中
  nsock_pool_add_event(ms, nse);

  // 返回事件的 ID
  return nse->id;
}

#endif  /* HAVE_SYS_UN_H */

#if HAVE_LINUX_VM_SOCKETS_H
/* 请求到另一个系统的 vsock 流连接。ss 应该是一个 sockaddr_storage 或 sockaddr_vm，如适当的话也可以是（就像你传递给 connect 的那样）。
 * sslen 应该是你传递的结构体的大小。 */
# 创建一个 vsock 流连接，并返回事件 ID
nsock_event_id nsock_connect_vsock_stream(nsock_pool nsp, nsock_iod ms_iod,
                                          nsock_ev_handler handler,
                                          int timeout_msecs, void *userdata,
                                          struct sockaddr *saddr, size_t sslen,
                                          unsigned int port) {
  # 将 ms_iod 转换为 niod 结构
  struct niod *nsi = (struct niod *)ms_iod;
  # 将 nsp 转换为 npool 结构
  struct npool *ms = (struct npool *)nsp;
  # 创建一个事件对象
  struct nevent *nse;
  # 将 saddr 转换为 sockaddr_storage 结构
  struct sockaddr_storage *ss = (struct sockaddr_storage *)saddr;
  # 将 saddr 转换为 sockaddr_vm 结构
  struct sockaddr_vm *svm = (struct sockaddr_vm *)saddr;

  # 断言 nsi 的状态为初始状态或未知状态
  assert(nsi->state == NSIOD_STATE_INITIAL || nsi->state == NSIOD_STATE_UNKNOWN);

  # 创建一个连接事件
  nse = event_new(ms, NSE_TYPE_CONNECT, nsi, timeout_msecs, handler, userdata);
  # 断言事件对象创建成功
  assert(nse);

  # 记录日志信息，表示请求建立 vsock 流连接
  nsock_log_info("vsock stream connection requested to %u:%u (IOD #%li) EID %li",
                 svm->svm_cid, port, nsi->id, nse->id);

  # 执行实际的 connect() 操作
  nsock_connect_internal(ms, nse, SOCK_STREAM, 0, ss, sslen, port);
  # 将事件添加到事件池中
  nsock_pool_add_event(ms, nse);

  # 返回事件 ID
  return nse->id;
}
/* Request a vsock datagram "connection" to another system.  Since this is a
 * datagram socket, no packets are actually sent.  The destination CID and port
 * are just associated with the nsiod (an actual OS connect() call is made).
 * You can then use the normal nsock write calls on the socket.  There is no
 * timeout since this call always calls your callback at the next opportunity.
 * The advantages to having a connected datagram socket (as opposed to just
 * specifying an address with sendto() are that we can now use a consistent set
 * of write/read calls for stream and datagram sockets, received packets from
 * the non-partner are automatically dropped by the OS, and the OS can provide
 * asynchronous errors (see Unix Network Programming pp224).  ss should be a
 * sockaddr_storage or sockaddr_vm, as appropriate (just like what you would
 * pass to connect).  sslen should be the sizeof the structure you are passing
 * in. */
nsock_event_id nsock_connect_vsock_datagram(nsock_pool nsp, nsock_iod nsiod,
                                            nsock_ev_handler handler,
                                            void *userdata,
                                            struct sockaddr *saddr,
                                            size_t sslen, unsigned int port) {
  struct niod *nsi = (struct niod *)nsiod;  // 将 nsiod 转换为 niod 结构体指针
  struct npool *ms = (struct npool *)nsp;  // 将 nsp 转换为 npool 结构体指针
  struct nevent *nse;  // 定义 nevent 结构体指针
  struct sockaddr_storage *ss = (struct sockaddr_storage *)saddr;  // 将 saddr 转换为 sockaddr_storage 结构体指针
  struct sockaddr_vm *svm = (struct sockaddr_vm *)saddr;  // 将 saddr 转换为 sockaddr_vm 结构体指针

  assert(nsi->state == NSIOD_STATE_INITIAL || nsi->state == NSIOD_STATE_UNKNOWN);  // 断言，确保 nsiod 的状态为 NSIOD_STATE_INITIAL 或 NSIOD_STATE_UNKNOWN

  nse = event_new(ms, NSE_TYPE_CONNECT, nsi, -1, handler, userdata);  // 创建一个新的事件
  assert(nse);  // 断言，确保事件创建成功

  nsock_log_info("vsock dgram connection requested to %u:%u (IOD #%li) EID %li",
                 svm->svm_cid, port, nsi->id, nse->id);  // 记录日志信息，表示请求 vsock 数据报连接

  nsock_connect_internal(ms, nse, SOCK_DGRAM, 0, ss, sslen, port);  // 进行内部连接
  nsock_pool_add_event(ms, nse);  // 将事件添加到事件池中

  return nse->id;  // 返回事件的 ID
}
#endif  /* HAVE_LINUX_VM_SOCKETS_H */
/* 请求与另一个系统（通过 IP 地址）建立 TCP 连接。in_addr 是正常的网络字节顺序，但端口号应该以主机字节顺序给出。ss 应该是一个 sockaddr_storage、sockaddr_in6 或 sockaddr_in，与传递给 connect 的参数相同。sslen 应该是传递的结构体的大小。 */
nsock_event_id nsock_connect_tcp(nsock_pool nsp, nsock_iod ms_iod, nsock_ev_handler handler, int timeout_msecs,
                                 void *userdata, struct sockaddr *saddr, size_t sslen, unsigned short port) {
  struct niod *nsi = (struct niod *)ms_iod;
  struct npool *ms = (struct npool *)nsp;
  struct nevent *nse;
  struct sockaddr_storage *ss = (struct sockaddr_storage *)saddr;

  // 断言，确保 nsi 的状态为 NSIOD_STATE_INITIAL 或 NSIOD_STATE_UNKNOWN
  assert(nsi->state == NSIOD_STATE_INITIAL || nsi->state == NSIOD_STATE_UNKNOWN);

  // 创建一个新的事件对象
  nse = event_new(ms, NSE_TYPE_CONNECT, nsi, timeout_msecs, handler, userdata);
  assert(nse);

  // 记录日志，请求建立 TCP 连接
  nsock_log_info("TCP connection requested to %s:%hu (IOD #%li) EID %li",
                 inet_ntop_ez(ss, sslen), port, nsi->id, nse->id);

  /* 执行实际的 connect() */
  nsock_connect_internal(ms, nse, SOCK_STREAM, IPPROTO_TCP, ss, sslen, port);
  // 将事件添加到事件池中
  nsock_pool_add_event(ms, nse);

  // 返回事件 ID
  return nse->id;
}

/* 请求与另一个系统（通过 IP 地址）建立 SCTP 关联。in_addr 是正常的网络字节顺序，但端口号应该以主机字节顺序给出。ss 应该是一个 sockaddr_storage、sockaddr_in6 或 sockaddr_in，与传递给 connect 的参数相同。sslen 应该是传递的结构体的大小。 */
# 使用 SCTP 协议在指定端口上发起连接，并返回事件 ID
nsock_event_id nsock_connect_sctp(nsock_pool nsp, nsock_iod ms_iod, nsock_ev_handler handler, int timeout_msecs,
                                  void *userdata, struct sockaddr *saddr, size_t sslen, unsigned short port) {

  # 将 ms_iod 转换为 niod 结构体
  struct niod *nsi = (struct niod *)ms_iod;
  # 将 nsp 转换为 npool 结构体
  struct npool *ms = (struct npool *)nsp;
  # 定义事件指针
  struct nevent *nse;
  # 将 saddr 转换为 sockaddr_storage 结构体
  struct sockaddr_storage *ss = (struct sockaddr_storage *)saddr;

  # 断言 nsiod 的状态为初始状态或未知状态
  assert(nsi->state == NSIOD_STATE_INITIAL || nsi->state == NSIOD_STATE_UNKNOWN);

  # 创建连接事件
  nse = event_new(ms, NSE_TYPE_CONNECT, nsi, timeout_msecs, handler, userdata);
  # 断言事件指针存在
  assert(nse);

  # 记录 SCTP 关联请求的信息
  nsock_log_info("SCTP association requested to %s:%hu (IOD #%li) EID %li",
                 inet_ntop_ez(ss, sslen), port, nsi->id, nse->id);

  /* 执行实际的 connect() 操作 */
  nsock_connect_internal(ms, nse, SOCK_STREAM, IPPROTO_SCTP, ss, sslen, port);
  # 将事件添加到事件池中
  nsock_pool_add_event(ms, nse);

  # 返回事件 ID
  return nse->id;
}

/* 请求使用 SSL 在 TCP/SCTP/UDP 上与另一个系统（通过 IP 地址）建立连接。
 * in_addr 使用网络字节顺序，但端口号应以主机字节顺序给出。
 * 此函数仅在建立连接并进行初始 SSL 协商后才会回调。
 * 从那时起，您可以使用正常的读/写调用，并且解密将会透明地进行。
 * ss 应该是一个 sockaddr_storage、sockaddr_in6 或 sockaddr_in，具体取决于您要传递的结构（就像您要传递给 connect 的那样）。
 * sslen 应该是您传递的结构的大小。 */
nsock_event_id nsock_connect_ssl(nsock_pool nsp, nsock_iod nsiod, nsock_ev_handler handler, int timeout_msecs,
                                 void *userdata, struct sockaddr *saddr, size_t sslen, int proto, unsigned short port, nsock_ssl_session ssl_session) {

#ifndef HAVE_OPENSSL
  # 如果没有 OpenSSL 支持，则输出错误信息并退出
  fatal("nsock_connect_ssl called - but nsock was built w/o SSL support.  QUITTING");
  return (nsock_event_id)0; /* UNREACHED */
#else
  // 定义并初始化指向 sockaddr_storage 结构体的指针 ss
  struct sockaddr_storage *ss = (struct sockaddr_storage *)saddr;
  // 定义并初始化指向 niod 结构体的指针 nsi
  struct niod *nsi = (struct niod *)nsiod;
  // 定义并初始化指向 npool 结构体的指针 ms
  struct npool *ms = (struct npool *)nsp;
  // 定义指向 nevent 结构体的指针 nse
  struct nevent *nse;

  // 如果协议为 IPPROTO_UDP
  if (proto == IPPROTO_UDP)
  {
    // 如果 npool 结构体中的 dtlsctx 为空，则初始化 DTLS 上下文
    if (!ms->dtlsctx)
      nsock_pool_dtls_init(ms, 0);
  }
  // 如果协议不为 IPPROTO_UDP
  else
  {
    // 如果 npool 结构体中的 sslctx 为空，则初始化 SSL 上下文
    if (!ms->sslctx)
      nsock_pool_ssl_init(ms, 0);
  }

  // 断言 nsi 的状态为 NSIOD_STATE_INITIAL 或 NSIOD_STATE_UNKNOWN
  assert(nsi->state == NSIOD_STATE_INITIAL || nsi->state == NSIOD_STATE_UNKNOWN);

  // 创建一个新的事件，并返回指向该事件的指针 nse
  nse = event_new(ms, NSE_TYPE_CONNECT_SSL, nsi, timeout_msecs, handler, userdata);
  // 断言 nse 不为空
  assert(nse);

  /* 设置 SSL_SESSION，以便我们可以从会话 ID 重用中受益。*/
  /* 但不适用于 DTLS；在 ClientHello 消息中节省空间 */
  // 如果协议不为 IPPROTO_UDP，则设置 SSL_SESSION
  if (proto != IPPROTO_UDP)
    nsi_set_ssl_session(nsi, (SSL_SESSION *)ssl_session);

  // 如果协议为 IPPROTO_UDP，则记录 DTLS 连接请求的信息
  if (proto == IPPROTO_UDP)
    nsock_log_info("DTLS connection requested to %s:%hu/udp (IOD #%li) EID %li",
                 inet_ntop_ez(ss, sslen), port, nsi->id, nse->id);
  // 如果协议不为 IPPROTO_UDP，则记录 SSL 连接请求的信息
  else
    nsock_log_info("SSL connection requested to %s:%hu/%s (IOD #%li) EID %li",
                 inet_ntop_ez(ss, sslen), port, (proto == IPPROTO_TCP ? "tcp" : "sctp"),
                 nsi->id, nse->id);

  // 执行实际的 connect()
  nsock_connect_internal(ms, nse, (proto == IPPROTO_UDP ? SOCK_DGRAM : SOCK_STREAM), proto, ss, sslen, port);

  // 将事件添加到事件池中
  nsock_pool_add_event(ms, nse);

  // 返回事件 ID
  return nse->id;
#endif /* HAVE_OPENSSL */
}

/* 请求在已建立的连接上进行 SSL 连接。nsiod 必须是已使用 nsock_connect_tcp 或 nsock_connect_sctp 连接到目标的套接字。
 * 所有参数的含义与 'nsock_connect_ssl' 中的含义相同 */
nsock_event_id nsock_reconnect_ssl(nsock_pool nsp, nsock_iod nsiod, nsock_ev_handler handler, int timeout_msecs,
                                   void *userdata, nsock_ssl_session ssl_session) {

#ifndef HAVE_OPENSSL
  // 如果没有 OpenSSL 支持，则输出错误信息并退出
  fatal("nsock_reconnect_ssl called - but nsock was built w/o SSL support.  QUITTING");
  // 返回 0，表示不可达
  return (nsock_event_id) 0; /* UNREACHED */
#else
  // 将 nsiod 转换为 struct niod 类型
  struct niod *nsi = (struct niod *)nsiod;
  // 将 nsp 转换为 struct npool 类型
  struct npool *ms = (struct npool *)nsp;
  // 定义指向结构体 nevent 的指针 nse
  struct nevent *nse;
  /* 对于 DTLS，不支持 nsock_reconnect_ssl（尚未支持？） */
  // 断言最后一个协议不是 IPPROTO_UDP
  assert(nsi->lastproto != IPPROTO_UDP);

  // 如果 ms->sslctx 为空，则初始化 SSL 上下文
  if (!ms->sslctx)
    nsock_pool_ssl_init(ms, 0);

  // 创建一个新的事件，类型为 NSE_TYPE_CONNECT_SSL，传入参数并设置超时时间、处理函数和用户数据
  nse = event_new(ms, NSE_TYPE_CONNECT_SSL, nsi, timeout_msecs, handler, userdata);
  // 断言事件非空
  assert(nse);

  /* 设置 SSL_SESSION，以便我们可以重用会话 ID。 */
  nsi_set_ssl_session(nsi, (SSL_SESSION *)ssl_session);

  // 记录 SSL 重新连接请求的日志信息
  nsock_log_info("SSL reconnection requested (IOD #%li) EID %li",
                 nsi->id, nse->id);

  /* 执行实际的 connect() 操作 */
  nse->event_done = 0;
  nse->status = NSE_STATUS_SUCCESS;
  // 将事件添加到事件池中
  nsock_pool_add_event(ms, nse);

  // 返回事件 ID
  return nse->id;
#endif /* HAVE_OPENSSL */
}

/* 请求与另一个系统（通过 IP 地址）建立 UDP "连接"。in_addr 使用网络字节顺序，但端口号应以主机字节顺序给出。
 * 由于这是 UDP，实际上并不发送任何数据包。目标 IP 和端口只是与 nsiod 关联（实际上会调用 OS 的 connect()）。然后可以在套接字上使用正常的 nsock 写入调用。
 * 由于此调用总是在下一个机会调用您的回调函数，因此没有超时。拥有连接的 UDP 套接字的优势（与仅使用 sendto() 指定地址相比）是我们现在可以使用一致的写入/读取调用进行 TCP/UDP，非伙伴的接收数据包会被操作系统自动丢弃，并且操作系统可以提供异步错误（参见《Unix Network Programming》pp224）。ss 应该是一个 sockaddr_storage、sockaddr_in6 或 sockaddr_in，具体取决于您要传递的结构体（就像您要传递给 connect 的那样）。sslen 应该是您要传递的结构体的大小。 */
# 使用 UDP 协议连接指定的地址和端口，并返回事件 ID
nsock_event_id nsock_connect_udp(nsock_pool nsp, nsock_iod nsiod, nsock_ev_handler handler, void *userdata,
                                 struct sockaddr *saddr, size_t sslen, unsigned short port) {

  # 将 nsiod 转换为 niod 结构体
  struct niod *nsi = (struct niod *)nsiod;
  # 将 nsp 转换为 npool 结构体
  struct npool *ms = (struct npool *)nsp;
  # 创建一个新的事件对象
  struct nevent *nse;
  # 将 saddr 转换为 sockaddr_storage 结构体
  struct sockaddr_storage *ss = (struct sockaddr_storage *)saddr;

  # 断言 nsiod 的状态为初始状态或未知状态
  assert(nsi->state == NSIOD_STATE_INITIAL || nsi->state == NSIOD_STATE_UNKNOWN);

  # 创建一个新的事件对象，类型为 NSE_TYPE_CONNECT
  nse = event_new(ms, NSE_TYPE_CONNECT, nsi, -1, handler, userdata);
  # 断言事件对象创建成功
  assert(nse);

  # 记录 UDP 连接请求的信息
  nsock_log_info("UDP connection requested to %s:%hu (IOD #%li) EID %li",
                 inet_ntop_ez(ss, sslen), port, nsi->id, nse->id);

  # 内部调用连接函数，使用 SOCK_DGRAM 和 IPPROTO_UDP
  nsock_connect_internal(ms, nse, SOCK_DGRAM, IPPROTO_UDP, ss, sslen, port);
  # 将事件对象添加到事件池中
  nsock_pool_add_event(ms, nse);

  # 返回事件 ID
  return nse->id;
}

/* 返回该 nsi 最后一次通信（或通信尝试）涉及的主机/端口/协议信息。
 * "涉及" 指的是建立（或尝试建立）连接或通过未连接的 nsock_iod 发送 UDP 数据报等交互。
 * AF 是地址族（AF_INET 或 AF_INET6），协议是 IPPROTO_TCP 或 IPPROTO_UDP。
 * 对于不需要的信息，传入 NULL。如果请求的任何信息不可用，将返回 0 并将不可用的套接字置零。
 * 如果请求协议或地址族但不可用，它将被设置为 -1（并返回 0）。
 * 传入的指针必须为 NULL 或指向已分配的地址空间。
 * sockaddr 成员应该实际上是 sockaddr_storage、sockaddr_in6 或 sockaddr_in，其长度设置为适当的值（例如 sizeof(sockaddr_storage)）。
 */
int nsock_iod_get_communication_info(nsock_iod iod, int *protocol, int *af,
                                     struct sockaddr *local,
                                     struct sockaddr *remote, size_t socklen) {
  // 将 nsock_iod 类型的参数转换为 niod 结构体类型
  struct niod *nsi = (struct niod *)iod;
  // 初始化返回值为 1
  int ret = 1;
  // 定义存储地址信息的结构体
  struct sockaddr_storage ss;
  // 定义存储地址信息长度的变量
  socklen_t slen = sizeof(ss);
  // 定义存储结果的变量
  int res;

  // 断言地址信息长度大于 0
  assert(socklen > 0);

  // 如果 peerlen 大于 0
  if (nsi->peerlen > 0) {
    // 如果 remote 参数不为空，则将 peer 的地址信息拷贝到 remote 中
    if (remote)
      memcpy(remote, &(nsi->peer), MIN((unsigned)socklen, nsi->peerlen));
    // 如果 protocol 参数不为空，则将 lastproto 赋值给 protocol，并检查是否为 -1
    if (protocol) {
      *protocol = nsi->lastproto;
      if (*protocol == -1) res = 0;
    }
    // 如果 af 参数不为空，则将 peer 的地址族赋值给 af
    if (af) {
      *af = nsi->peer.ss_family;
    }
    // 如果 local 参数不为空
    if (local) {
      // 如果 sd 大于等于 0，则获取本地地址信息并拷贝到 local 中
      if (nsi->sd >= 0) {
        res = getsockname(nsi->sd, (struct sockaddr *)&ss, &slen);
        if (res == -1) {
          memset(local, 0, socklen);
          ret = 0;
        } else {
          assert(slen > 0);
          memcpy(local, &ss, MIN((unsigned)slen, socklen));
        }
      } else {
        // 如果 sd 小于 0，则将 local 清零，并将返回值设为 0
        memset(local, 0, socklen);
        ret = 0;
      }
    }
  } else {
    // 如果 peerlen 不大于 0，则将返回值设为 0
    if (local || remote || protocol || af)
      ret = 0;

    // 如果 remote 参数不为空，则将其清零
    if (remote)
      memset(remote, 0, socklen);

    // 如果 local 参数不为空，则将其清零
    if (local)
      memset(local, 0, socklen);

    // 如果 protocol 参数不为空，则将其赋值为 -1
    if (protocol)
      *protocol = -1;

    // 如果 af 参数不为空，则将其赋值为 -1
    if (af)
      *af = -1;
  }
  // 返回结果值
  return ret;
}
```