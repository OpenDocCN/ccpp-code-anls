# `nmap\nsock\src\nsock_core.c`

```cpp
/* $Id$ */

#include "nsock_internal.h"  // 包含内部 nsock 头文件
#include "gh_list.h"  // 包含 gh_list 头文件
#include "filespace.h"  // 包含 filespace 头文件
#include "nsock_log.h"  // 包含 nsock_log 头文件

#include <assert.h>  // 包含断言头文件
#if HAVE_ERRNO_H
#include <errno.h>  // 包含错误处理头文件
#endif
#if HAVE_SYS_TYPES_H
#include <sys/types.h>  // 包含系统类型头文件
#endif
#if HAVE_SYS_SOCKET_H
#include <sys/socket.h>  // 包含套接字头文件
#endif
#if HAVE_NETINET_IN_H
#include <netinet/in.h>  // 包含互联网地址族头文件
#endif
#if HAVE_ARPA_INET_H
#include <arpa/inet.h>  // 包含互联网地址转换头文件
#endif
#if HAVE_STRING_H
#include <string.h>  // 包含字符串处理头文件
#endif

#include "netutils.h"  // 包含网络工具头文件

#if HAVE_PCAP
#include "nsock_pcap.h"  // 如果支持 pcap，则包含 nsock_pcap 头文件
#endif

/* Nsock time of day -- we update this at least once per nsock_loop round (and
 * after most calls that are likely to block).  Other nsock files should grab
 * this */
struct timeval nsock_tod;  // 定义 nsock_tod 结构体变量，用于记录时间

/* Internal function defined in nsock_event.c
 * Update the nse->iod first events, assuming nse is about to be deleted */
void update_first_events(struct nevent *nse);  // 声明内部函数 update_first_events，用于更新事件

/* Each iod has a count of pending socket reads, socket writes, and pcap reads.
 * When a descriptor's count is nonzero, its bit must be set in the appropriate
 * master fd_set, and when the count is zero the bit must be cleared. What we
 * are simulating is an fd_set with a counter for each socket instead of just an
 * on/off switch. The fd_set's bits aren't enough by itself because a descriptor
 * may for example have two reads pending at once, and the bit must not be
 * cleared after the first is completed.
 * The socket_count_* functions return the event to transmit to update_events() */
int socket_count_zero(struct niod *iod, struct npool *ms) {  // 定义函数 socket_count_zero，用于清零计数并注销事件
  iod->readsd_count = 0;  // 读取计数清零
  iod->writesd_count = 0;  // 写入计数清零
#if HAVE_PCAP
  iod->readpcapsd_count = 0;  // 如果支持 pcap，则 pcap 读取计数清零
#endif
  return nsock_engine_iod_unregister(ms, iod);  // 返回注销事件的结果
}

static int socket_count_read_inc(struct niod *iod) {  // 定义静态函数 socket_count_read_inc，用于增加读取计数
  assert(iod->readsd_count >= 0);  // 断言读取计数大于等于0
  iod->readsd_count++;  // 增加读取计数
  return EV_READ;  // 返回读取事件
}

static int socket_count_read_dec(struct niod *iod) {  // 定义静态函数 socket_count_read_dec，用于减少读取计数
  assert(iod->readsd_count > 0);  // 断言读取计数大于0
  iod->readsd_count--;  // 减少读取计数
  return (iod->readsd_count == 0) ? EV_READ : EV_NONE;  // 如果读取计数为0，则返回读取事件，否则返回无事件
}
static int socket_count_write_inc(struct niod *iod) {
  // 确保写入事件计数大于等于0
  assert(iod->writesd_count >= 0);
  // 增加写入事件计数
  iod->writesd_count++;
  // 返回写入事件
  return EV_WRITE;
}

static int socket_count_write_dec(struct niod *iod) {
  // 确保写入事件计数大于0
  assert(iod->writesd_count > 0);
  // 减少写入事件计数
  iod->writesd_count--;
  // 如果计数为0，则返回EV_WRITE，否则返回EV_NONE
  return (iod->writesd_count == 0) ? EV_WRITE : EV_NONE;
}

#if HAVE_PCAP
static int socket_count_readpcap_inc(struct niod *iod) {
  // 确保读取PCAP事件计数大于等于0
  assert(iod->readpcapsd_count >= 0);
  // 增加读取PCAP事件计数
  iod->readpcapsd_count++;
  // 返回读取事件
  return EV_READ;
}

static int socket_count_readpcap_dec(struct niod *iod) {
  // 确保读取PCAP事件计数大于0
  assert(iod->readpcapsd_count > 0);
  // 减少读取PCAP事件计数
  iod->readpcapsd_count--;
  // 如果计数为0，则返回EV_READ，否则返回EV_NONE
  return (iod->readpcapsd_count == 0) ? EV_READ : EV_NONE;
}
#endif

#if HAVE_OPENSSL
/* 根据nse->sslinfo.ssl_desire的当前值，在nse->iod上调用socket_count_read_dec或socket_count_write_dec。 */
static int socket_count_dec_ssl_desire(struct nevent *nse) {
  // 确保nse->iod->ssl不为空
  assert(nse->iod->ssl != NULL);
  // 确保nse->sslinfo.ssl_desire的值为SSL_ERROR_WANT_READ或SSL_ERROR_WANT_WRITE
  assert(nse->sslinfo.ssl_desire == SSL_ERROR_WANT_READ ||
         nse->sslinfo.ssl_desire == SSL_ERROR_WANT_WRITE);

  // 如果nse->sslinfo.ssl_desire的值为SSL_ERROR_WANT_READ，则调用socket_count_read_dec，否则调用socket_count_write_dec
  if (nse->sslinfo.ssl_desire == SSL_ERROR_WANT_READ)
    return socket_count_read_dec(nse->iod);
  else
    return socket_count_write_dec(nse->iod);
}
#endif

static int should_clear_ev_read(const struct niod *iod, int ev_set) {
  // 如果ev_set中包含EV_READ，并且readpcapsd_count和readsd_count都为0，则返回true，否则返回false
  return (ev_set & EV_READ) &&
#if HAVE_PCAP
         !iod->readpcapsd_count &&
#endif
         !iod->readsd_count;
}

static int should_clear_ev_write(const struct niod *iod, int ev_set) {
  // 如果ev_set中包含EV_WRITE，并且writesd_count为0，则返回true，否则返回false
  return (ev_set & EV_WRITE) && !iod->writesd_count;
}

/* 更新IO引擎应该监视的给定IOD的事件。
 *
 * ev_inc是一组事件，其中事件计数应增加。因此，IO引擎将监视此IOD的这些事件。
 *
 * ev_dec是一组事件，其中事件计数应减少。
 * 如果此计数达到零，则IO引擎将不再监视此IOD的事件。
 */
static void update_events(struct niod * iod, struct npool *ms, struct nevent *nse, int ev_inc, int ev_dec) {
  int setmask, clrmask, ev_temp;

  /* 过滤出同时属于两个集合的事件。*/
  ev_temp = ev_inc ^ ev_dec;
  ev_inc = ev_inc & ev_temp;
  ev_dec = ev_dec & ev_temp;

  setmask = ev_inc;
  clrmask = EV_NONE;

  if (should_clear_ev_read(iod, ev_dec))
    clrmask |= EV_READ;

  if (should_clear_ev_write(iod, ev_dec))
    clrmask |= EV_WRITE;

  /* EV_EXCEPT 是系统设置的，不能被移除 */
  if (ev_inc & EV_EXCEPT)
    nsock_log_info("无效的事件设置，不需要指定 EV_EXCEPT");

  if (ev_dec & EV_EXCEPT)
    nsock_log_info("无效的事件设置，拒绝清除 EV_EXCEPT");

  if (!IOD_PROPGET(iod, IOD_REGISTERED)) {
    assert(clrmask == EV_NONE);
    nsock_engine_iod_register(ms, iod, nse, setmask);
  } else {
    nsock_engine_iod_modify(ms, iod, nse, setmask, clrmask);
  }
}

/* 为给定的 IOD 添加一个新事件。nevents 存储在单独的事件列表中（在 nsock 池中），并且在每个列表中按 IOD 进行分组。

   此函数将事件添加到给定 IOD 的第一个相似事件之前，或者如果没有相似事件，则将其追加到列表的末尾。

   注意，将事件添加到相似事件之前对于可重入性很重要，因为它将防止新事件在其添加后立即在事件循环中被处理。
*/
static int iod_add_event(struct niod *iod, struct nevent *nse) {
  struct npool *nsp = iod->nsp;

  switch (nse->type) {
    case NSE_TYPE_CONNECT:
    case NSE_TYPE_CONNECT_SSL:
      if (iod->first_connect)
        gh_list_insert_before(&nsp->connect_events,
                              iod->first_connect, &nse->nodeq_io);
      else
        gh_list_append(&nsp->connect_events, &nse->nodeq_io);
      iod->first_connect = &nse->nodeq_io;
      break;
    # 如果事件类型是读取
    case NSE_TYPE_READ:
      # 如果是第一个读取事件
      if (iod->first_read)
        # 在第一个读取事件之前插入当前事件
        gh_list_insert_before(&nsp->read_events, iod->first_read, &nse->nodeq_io);
      else
        # 在读取事件列表末尾添加当前事件
        gh_list_append(&nsp->read_events, &nse->nodeq_io);
      # 将当前事件设置为第一个读取事件
      iod->first_read = &nse->nodeq_io;
      break;

    # 如果事件类型是写入
    case NSE_TYPE_WRITE:
      # 如果是第一个写入事件
      if (iod->first_write)
        # 在第一个写入事件之前插入当前事件
        gh_list_insert_before(&nsp->write_events, iod->first_write, &nse->nodeq_io);
      else
        # 在写入事件列表末尾添加当前事件
        gh_list_append(&nsp->write_events, &nse->nodeq_io);
      # 将当前事件设置为第一个写入事件
      iod->first_write = &nse->nodeq_io;
      break;
#if HAVE_PCAP
    // 如果支持 pcap
    case NSE_TYPE_PCAP_READ: {
      // 如果事件类型是 pcap 读取
      char add_read = 0, add_pcap_read = 0;

#if PCAP_BSD_SELECT_HACK
      // 如果是 BSD 系统，同时将事件添加到读取和 pcap 读取列表中
      add_read = add_pcap_read = 1;
#else
      // 如果不是 BSD 系统
      if (((mspcap *)iod->pcap)->pcap_desc >= 0) {
        // 如果 pcap 描述符大于等于 0
        add_read = 1;
      } else {
        // 否则
        add_pcap_read = 1;
      }
#endif
      // 如果需要添加到读取列表
      if (add_read) {
        // 如果当前事件是第一个读取事件
        if (iod->first_read)
          // 将当前事件插入到读取事件列表中
          gh_list_insert_before(&nsp->read_events, iod->first_read, &nse->nodeq_io);
        else
          // 将当前事件追加到读取事件列表中
          gh_list_append(&nsp->read_events, &nse->nodeq_io);
        // 更新当前事件为第一个读取事件
        iod->first_read = &nse->nodeq_io;
      }
      // 如果需要添加到 pcap 读取列表
      if (add_pcap_read) {
        // 如果当前事件是第一个 pcap 读取事件
        if (iod->first_pcap_read)
          // 将当前事件插入到 pcap 读取事件列表中
          gh_list_insert_before(&nsp->pcap_read_events, iod->first_pcap_read,
                                &nse->nodeq_pcap);
        else
          // 将当前事件追加到 pcap 读取事件列表中
          gh_list_append(&nsp->pcap_read_events, &nse->nodeq_pcap);
        // 更新当前事件为第一个 pcap 读取事件
        iod->first_pcap_read = &nse->nodeq_pcap;
      }
      // 结束当前事件处理
      break;
    }
#endif

    // 默认情况
    default:
      // 报错，未知的事件类型
      fatal("Unknown event type (%d) for IOD #%lu\n", nse->type, iod->id);
  }

  // 返回 0
  return 0;
}

/* 为每个主要事件类型（读取、写入、连接、定时器等）定义一个处理函数 -- 当事件有新信息可用时调用处理程序。
   处理程序根据可用的任何新信息对事件进行任何必要的更新。
   如果事件准备好传递，处理程序设置 nse->event_done 并填写相关的事件字段（状态、错误号）。
   处理程序还负责特定于事件类型的拆卸（例如从 select/poll 列表中清除套接字描述符）。
   如果 event_done 未设置，则在有更多信息或事件超时的情况下再次调用处理程序 */
/* 事件类型处理程序--每个处理程序的前三个参数都是相同的：
 * struct npool *ms struct nevent *nse -- 我们有新信息的事件 enum nse_status --
 * 调用原因，通常是 NSE_STATUS_SUCCESS（通常表示成功的 I/O 调用）或 NSE_STATUS_TIMEOUT 或 NSE_STATUS_CANCELLED
 *
 *  一些事件类型处理程序有其他参数，特定于它们的需求。所有处理程序都可以假定调用函数已经检查了 select 或 poll 是否显示它们的描述符是可读/可写的（适当的）。
 *
 *  思想是每个处理程序将处理特定于它的内容，而调用函数将处理可以泛化为分派/删除等的所有事件的内容。但是调用函数可能使用特定于类型的信息来确定是否应该调用处理程序（以节省 CPU 时间）。 */

/* handle_connect_results 假设 select 或 poll 已经显示描述符是活动的 */
void handle_connect_result(struct npool *ms, struct nevent *nse, enum nse_status status) {
  int optval;
  socklen_t optlen = sizeof(int);
  struct niod *iod = nse->iod;
  assert(iod != NULL);
#if HAVE_OPENSSL
  int sslerr;
  int rc = 0;
  int sslconnect_inprogress = nse->type == NSE_TYPE_CONNECT_SSL && nse->iod &&
    (nse->sslinfo.ssl_desire == SSL_ERROR_WANT_READ ||
     nse->sslinfo.ssl_desire == SSL_ERROR_WANT_WRITE);
  SSL_CTX *sslctx = NULL;
#else
  int sslconnect_inprogress = 0;
#endif

  if (status == NSE_STATUS_TIMEOUT || status == NSE_STATUS_CANCELLED) {
    nse->status = status;
    nse->event_done = 1;
  } else if (sslconnect_inprogress) {
    /* 什么也不做 */
  } else if (status == NSE_STATUS_SUCCESS) {
    /* 首先，我们想确定套接字是否真的已连接 */
    if (getsockopt(iod->sd, SOL_SOCKET, SO_ERROR, (char *)&optval, &optlen) != 0)
      optval = socket_errno(); /* 愚蠢的 Solaris */
    # 如果 optval 等于 0，则将 nse 的状态设置为成功
    if (optval == 0) {
        nse->status = NSE_STATUS_SUCCESS;
    }
    # 否则，将 nse 的状态设置为错误，并将错误码设置为 optval
    else {
        nse->status = NSE_STATUS_ERROR;
        nse->errnum = optval;
    }

    # 现在针对 SSL 情况进行特殊处理，其中 TCP 连接成功
    if (nse->type == NSE_TYPE_CONNECT_SSL &&
        nse->status == NSE_STATUS_SUCCESS) {
#if HAVE_OPENSSL
      // 如果有 OpenSSL 支持，则根据最后一个协议选择 SSL 上下文
      sslctx = iod->lastproto == IPPROTO_UDP ? ms->dtlsctx : ms->sslctx;
      // 断言 SSL 上下文不为空
      assert(sslctx != NULL);
      /* 如果 iod->ssl 存在，则重用它。如果设置了，这是在没有设置 SSL_OP_NO_SSLv2 选项的情况下进行连接的第二次尝试。 */
      if (iod->ssl == NULL) {
        // 创建一个新的 SSL 对象
        iod->ssl = SSL_new(sslctx);
        // 如果创建失败，则输出错误信息并终止程序
        if (!iod->ssl)
          fatal("SSL_new failed: %s", ERR_error_string(ERR_get_error(), NULL));
      }

#if HAVE_SSL_SET_TLSEXT_HOST_NAME
      /* 避免在 DTLS 中发送 SNI 扩展，因为许多服务器不允许分段的 ClientHello 消息。 */
      if (iod->hostname != NULL && iod->lastproto != IPPROTO_UDP) {
        // 设置 TLS 扩展主机名
        if (SSL_set_tlsext_host_name(iod->ssl, iod->hostname) != 1)
          fatal("SSL_set_tlsext_host_name failed: %s", ERR_error_string(ERR_get_error(), NULL));
      }
#endif

      /* 将新的 SSL 与连接的套接字关联。它将继承 sd 的非阻塞特性 */
      if (SSL_set_fd(iod->ssl, iod->sd) != 1)
        fatal("SSL_set_fd failed: %s", ERR_error_string(ERR_get_error(), NULL));

      /* 事件未完成 -- 需要在下面执行 SSL 连接 */
      nse->sslinfo.ssl_desire = SSL_ERROR_WANT_CONNECT;
#endif
    } else {
      /* 这不是 SSL 连接（在这种情况下我们总是完成的），或者 SSL 下面的 TCP 连接() 失败（在这种情况下我们也是完成的） */
      nse->event_done = 1;
    }
  } else {
    fatal("Unknown status (%d)", status);
  }

  /* 此时 TCP 连接已完成，无论成功与否。因此，减少在 nsock_pool_add_event 中增加的读/写监听计数。在 SSL 情况下，根据 SSL_connect 返回的 SSL_ERROR_WANT_READ 或 SSL_ERROR_WANT_WRITE 错误，我们可能会增加计数中的一个。在这种情况下，我们将重新进入此函数，但我们不希望再次执行此块。 */
  if (iod->sd != -1 && !sslconnect_inprogress) {
    int ev = EV_NONE;
    # 将事件值与读取套接字计数的结果进行按位或操作，并将结果赋给ev变量
    ev |= socket_count_read_dec(iod);
    # 将事件值与写入套接字计数的结果进行按位或操作，并将结果赋给ev变量
    ev |= socket_count_write_dec(iod);
    # 调用update_events函数，更新套接字的事件，传入参数iod为套接字，ms为毫秒数，nse为纳秒数，EV_NONE为无事件，ev为事件值
    update_events(iod, ms, nse, EV_NONE, ev);
  }
#if HAVE_OPENSSL
  // 如果系统支持 OpenSSL
  if (nse->type == NSE_TYPE_CONNECT_SSL && !nse->event_done) {
    // 如果当前连接类型为 SSL 并且事件未完成
    /* Lets now start/continue/finish the connect! */
    // 开始/继续/完成连接操作
    if (iod->ssl_session) {
      // 如果存在 SSL 会话，将其设置到当前 SSL 连接
      rc = SSL_set_session(iod->ssl, iod->ssl_session);
      // 如果设置会话失败，记录错误日志
      if (rc == 0)
        nsock_log_error("Uh-oh: SSL_set_session() failed - please tell dev@nmap.org");
      iod->ssl_session = NULL; /* No need for this any more */
      // 清空 SSL 会话，不再需要
    }

    /* If this is a reinvocation of handle_connect_result, clear out the listen
     * bits that caused it, based on the previous SSL desire. */
    // 如果这是 handle_connect_result 的重新调用，根据之前的 SSL 期望清除导致它的监听位
    if (sslconnect_inprogress) {
      // 如果 SSL 连接正在进行中
      int ev;
      // 定义事件变量
      ev = socket_count_dec_ssl_desire(nse);
      // 减少 SSL 期望的套接字计数
      update_events(iod, ms, nse, EV_NONE, ev);
      // 更新事件
    }

    rc = SSL_connect(iod->ssl);
    // 开始 SSL 连接
    if (rc == 1) {
      // 如果连接成功
      /* Woop!  Connect is done! */
      // 连接完成
      nse->event_done = 1;
      // 设置事件完成标志
      /* Check that certificate verification was okay, if requested. */
      // 检查证书验证是否成功
      if (nsi_ssl_post_connect_verify(iod)) {
        // 如果证书验证成功
        nse->status = NSE_STATUS_SUCCESS;
        // 设置连接状态为成功
      } else {
        // 如果证书验证失败
        nsock_log_error("certificate verification error for EID %li: %s",
                        nse->id, ERR_error_string(ERR_get_error(), NULL));
        // 记录证书验证错误日志
        nse->status = NSE_STATUS_ERROR;
        // 设置连接状态为错误
      }
    } else {
#if SSL_OP_NO_SSLv2 != 0
      long options = SSL_get_options(iod->ssl);
#endif
      // 如果 SSL 不支持 SSLv2
      sslerr = SSL_get_error(iod->ssl, rc);
      // 获取 SSL 错误码
      if (sslerr == SSL_ERROR_WANT_READ) {
        // 如果 SSL 需要读操作
        nse->sslinfo.ssl_desire = sslerr;
        // 设置 SSL 期望为读操作
        socket_count_read_inc(iod);
        // 增加读操作的套接字计数
        update_events(iod, ms, nse, EV_READ, EV_NONE);
        // 更新事件
      } else if (sslerr == SSL_ERROR_WANT_WRITE) {
        // 如果 SSL 需要写操作
        nse->sslinfo.ssl_desire = sslerr;
        // 设置 SSL 期望为写操作
        socket_count_write_inc(iod);
        // 增加写操作的套接字计数
        update_events(iod, ms, nse, EV_WRITE, EV_NONE);
        // 更新事件
#if SSL_OP_NO_SSLv2 != 0
      # 如果 SSL_OP_NO_SSLv2 不等于 0
      } else if (iod->lastproto != IPPROTO_UDP && !(options & SSL_OP_NO_SSLv2)) {
        # 如果上一个协议不是UDP，并且选项中不包含SSL_OP_NO_SSLv2
        int saved_ev;
        # 定义保存事件的变量

        /* SSLv3-only and TLSv1-only servers can't be connected to when the
         * SSL_OP_NO_SSLv2 option is not set, which is the case when the pool
         * was initialized with nsock_pool_ssl_init_max_speed. Try reconnecting
         * with SSL_OP_NO_SSLv2. Never downgrade a NO_SSLv2 connection to one
         * that might use SSLv2. */
        # SSLv3-only 和 TLSv1-only 服务器在未设置SSL_OP_NO_SSLv2选项时无法连接，这是在使用nsock_pool_ssl_init_max_speed初始化池时的情况。尝试重新连接使用SSL_OP_NO_SSLv2。永远不要将NO_SSLv2连接降级为可能使用SSLv2的连接。
        nsock_log_info("EID %li reconnecting with SSL_OP_NO_SSLv2", nse->id);
        # 记录日志信息，重新连接使用SSL_OP_NO_SSLv2

        saved_ev = iod->watched_events;
        # 保存当前的事件
        nsock_engine_iod_unregister(ms, iod);
        # 取消注册当前的 I/O 引擎
        close(iod->sd);
        # 关闭当前的套接字
        nsock_connect_internal(ms, nse, SOCK_STREAM, iod->lastproto, &iod->peer,
                               iod->peerlen, nsock_iod_get_peerport(iod));
        # 内部连接
        nsock_engine_iod_register(ms, iod, nse, saved_ev);
        # 注册 I/O 引擎

        /* Use SSL_free here because SSL_clear keeps session info, which
         * doesn't work when changing SSL versions (as we're clearly trying to
         * do by adding SSL_OP_NO_SSLv2). */
        # 在这里使用 SSL_free，因为 SSL_clear 会保留会话信息，当更改 SSL 版本时（如我们明显尝试通过添加 SSL_OP_NO_SSLv2 来做的）这是行不通的。
        SSL_free(iod->ssl);
        # 释放 SSL 对象
        iod->ssl = SSL_new(ms->sslctx);
        # 创建一个新的 SSL 对象
        if (!iod->ssl)
          fatal("SSL_new failed: %s", ERR_error_string(ERR_get_error(), NULL));
        # 如果 SSL 对象创建失败，则输出错误信息

        SSL_set_options(iod->ssl, options | SSL_OP_NO_SSLv2);
        # 设置 SSL 选项
        socket_count_read_inc(nse->iod);
        # 读取套接字计数增加
        socket_count_write_inc(nse->iod);
        # 写入套接字计数增加
        update_events(iod, ms, nse, EV_READ|EV_WRITE, EV_NONE);
        # 更新事件
        nse->sslinfo.ssl_desire = SSL_ERROR_WANT_CONNECT;
        # 设置 SSL 期望连接错误
#endif
      } else {
        nsock_log_info("EID %li %s",
                       nse->id, ERR_error_string(ERR_get_error(), NULL));
        # 记录日志信息
        nse->event_done = 1;
        # 事件完成标志设为1
        nse->status = NSE_STATUS_ERROR;
        # 设置状态为错误
        nse->errnum = EIO;
        # 设置错误号为 EIO
      }
    }
  }
#endif
}

static int errcode_is_failure(int err) {
#ifndef WIN32
  return err != EINTR && err != EAGAIN && err != EBUSY;
  # 如果错误不是 EINTR、EAGAIN 和 EBUSY，则返回真
#else
  return err != EINTR && err != EAGAIN;
#endif
}

void handle_write_result(struct npool *ms, struct nevent *nse, enum nse_status status) {
  int bytesleft;  // 定义剩余字节数变量
  char *str;  // 定义字符指针变量
  int res;  // 定义结果变量
  int err;  // 定义错误变量
  struct niod *iod = nse->iod;  // 定义 niod 结构体指针变量，指向 nevent 结构体的 iod 成员

  if (status == NSE_STATUS_TIMEOUT || status == NSE_STATUS_CANCELLED) {  // 如果状态为超时或取消
    nse->event_done = 1;  // 设置事件完成标志为 1
    nse->status = status;  // 设置状态为当前状态
  } else if (status == NSE_STATUS_SUCCESS) {  // 如果状态为成功
    str = fs_str(&nse->iobuf) + nse->writeinfo.written_so_far;  // 获取写入数据的字符串
    bytesleft = fs_length(&nse->iobuf) - nse->writeinfo.written_so_far;  // 计算剩余字节数
    if (nse->writeinfo.written_so_far > 0)  // 如果已经写入的字节数大于 0
      assert(bytesleft > 0);  // 断言剩余字节数大于 0
#if HAVE_OPENSSL
    if (iod->ssl)  // 如果使用了 SSL
      res = SSL_write(iod->ssl, str, bytesleft);  // 使用 SSL 写入数据
    else
#endif
      res = ms->engine->io_operations->iod_write(ms, nse->iod->sd, str, bytesleft, 0, (struct sockaddr *)&nse->writeinfo.dest, (int)nse->writeinfo.destlen);  // 否则使用非 SSL 写入数据
    if (res == bytesleft) {  // 如果写入字节数等于剩余字节数
      nse->event_done = 1;  // 设置事件完成标志为 1
      nse->status = NSE_STATUS_SUCCESS;  // 设置状态为成功
    } else if (res >= 0) {  // 如果写入字节数大于等于 0
      nse->writeinfo.written_so_far += res;  // 更新已写入字节数
    } else {  // 否则
      assert(res == -1);  // 断言写入结果为 -1
      if (iod->ssl) {  // 如果使用了 SSL
#if HAVE_OPENSSL
        err = SSL_get_error(iod->ssl, res);  // 获取 SSL 错误
        if (err == SSL_ERROR_WANT_READ) {  // 如果 SSL 错误为需要读取
          int evclr;  // 定义事件清除变量

          evclr = socket_count_dec_ssl_desire(nse);  // 减少 SSL 期望计数
          socket_count_read_inc(iod);  // 增加读取计数
          update_events(iod, ms, nse, EV_READ, evclr);  // 更新事件为读取事件
          nse->sslinfo.ssl_desire = err;  // 设置 SSL 期望值
        } else if (err == SSL_ERROR_WANT_WRITE) {  // 如果 SSL 错误为需要写入
          int evclr;  // 定义事件清除变量

          evclr = socket_count_dec_ssl_desire(nse);  // 减少 SSL 期望计数
          socket_count_write_inc(iod);  // 增加写入计数
          update_events(iod, ms, nse, EV_WRITE, evclr);  // 更新事件为写入事件
          nse->sslinfo.ssl_desire = err;  // 设置 SSL 期望值
        } else {  // 否则
          /* Unexpected error */  // 意外错误
          nse->event_done = 1;  // 设置事件完成标志为 1
          nse->status = NSE_STATUS_ERROR;  // 设置状态为错误
          nse->errnum = EIO;  // 设置错误号为输入输出错误
        }
#endif
      } else {
        // 获取套接字错误码
        err = socket_errno();
        // 如果错误码表示失败，则设置事件完成标志、状态为错误、错误码
        if (errcode_is_failure(err)) {
          nse->event_done = 1;
          nse->status = NSE_STATUS_ERROR;
          nse->errnum = err;
        }
      }
    }

    // 如果读取成功，更新写入计数
    if (res >= 0)
      nse->iod->write_count += res;
  }

  // 如果事件完成且套接字描述符不为-1
  if (nse->event_done && nse->iod->sd != -1) {
    int ev = EV_NONE;

#if HAVE_OPENSSL
    // 如果使用了 OpenSSL，更新事件
    if (nse->iod->ssl != NULL)
      ev |= socket_count_dec_ssl_desire(nse);
    else
#endif
      // 否则，更新写入事件
      ev |= socket_count_write_dec(nse->iod);
    update_events(nse->iod, ms, nse, EV_NONE, ev);
  }
}

// 处理定时器结果
void handle_timer_result(struct npool *ms, struct nevent *nse, enum nse_status status) {
  /* Ooh this is a hard job :) */
  // 设置事件完成标志、状态
  nse->event_done = 1;
  nse->status = status;
}

/* 如果出错返回-1，否则返回新写入的字节数 */
static int do_actual_read(struct npool *ms, struct nevent *nse) {
  char buf[READ_BUFFER_SZ];
  int buflen = 0;
  struct niod *iod = nse->iod;
  int err = 0;
  int max_chunk = NSOCK_READ_CHUNK_SIZE;
  int startlen = fs_length(&nse->iobuf);
  int enotsock = 0;  /* Did we get ENOTSOCK once? */

  // 如果读取类型为 NSOCK_READBYTES，设置最大读取块大小
  if (nse->readinfo.read_type == NSOCK_READBYTES)
    max_chunk = nse->readinfo.num;

  // 如果不使用 SSL
  if (!iod->ssl) {
    // 循环读取数据
    do {
      buflen = read(iod->sd, buf, READ_BUFFER_SZ);
      if (buflen > 0) {
        // 写入数据到缓冲区
        fs_write(&nse->iobuf, buf, buflen);
      } else if (buflen == 0) {
        // 读取结束，设置事件完成标志
        nse->event_done = 1;
      } else {
        // 出错处理
        err = socket_errno();
        if (err != EINTR && err != EAGAIN) {
          nse->event_done = 1;
          nse->status = NSE_STATUS_ERROR;
          nse->errnum = err;
          return -1;
        }
      }
    } while (buflen > 0 || (buflen == -1 && err == EINTR));

    if (buflen == -1) {
      if (err != EINTR && err != EAGAIN) {
        nse->event_done = 1;
        nse->status = NSE_STATUS_ERROR;
        nse->errnum = err;
        return -1;
      }
    }
  } else {
#if HAVE_OPENSSL
    /* OpenSSL read */
    // 如果使用 OpenSSL，进行 OpenSSL 读取
    // ...
#endif
  }
}
    # 当从 SSL 连接中读取数据时，循环执行直到读取的数据长度为 0
    while ((buflen = SSL_read(iod->ssl, buf, sizeof(buf))) > 0) {

      # 将读取的数据追加到输入缓冲区中，如果失败则设置错误状态并返回 -1
      if (fs_cat(&nse->iobuf, buf, buflen) == -1) {
        nse->event_done = 1;
        nse->status = NSE_STATUS_ERROR;
        nse->errnum = ENOMEM;
        return -1;
      }

      # 如果输入缓冲区中的数据长度超过了最大允许的长度，则返回当前数据长度减去起始长度
      if (fs_length(&nse->iobuf) > max_chunk)
        return fs_length(&nse->iobuf) - startlen;
    }

    # 如果读取数据时出现错误
    if (buflen == -1) {
      # 获取 SSL 错误码
      err = SSL_get_error(iod->ssl, buflen);
      # 如果是需要继续读取操作，则更新事件并设置 SSL 读取状态
      if (err == SSL_ERROR_WANT_READ) {
        int evclr;
        evclr = socket_count_dec_ssl_desire(nse);
        socket_count_read_inc(iod);
        update_events(iod, ms, nse, EV_READ, evclr);
        nse->sslinfo.ssl_desire = err;
      } 
      # 如果是需要继续写入操作，则更新事件并设置 SSL 写入状态
      else if (err == SSL_ERROR_WANT_WRITE) {
        int evclr;
        evclr = socket_count_dec_ssl_desire(nse);
        socket_count_write_inc(iod);
        update_events(iod, ms, nse, EV_WRITE, evclr);
        nse->sslinfo.ssl_desire = err;
      } 
      # 如果是其他未预期的错误，则设置错误状态并返回 -1
      else {
        nse->event_done = 1;
        nse->status = NSE_STATUS_ERROR;
        nse->errnum = EIO;
        nsock_log_info("SSL_read() failed for reason %s on NSI %li",
                       ERR_error_string(err, NULL), iod->id);
        return -1;
      }
    }
#endif /* HAVE_OPENSSL */
  }

  // 如果缓冲区长度为0
  if (buflen == 0) {
    // 设置事件完成标志和结束标志
    nse->event_done = 1;
    nse->eof = 1;
    // 如果输入输出缓冲区中有数据
    if (fs_length(&nse->iobuf) > 0) {
      // 设置状态为成功并返回数据长度
      nse->status = NSE_STATUS_SUCCESS;
      return fs_length(&nse->iobuf) - startlen;
    } else {
      // 设置状态为文件结束并返回0
      nse->status = NSE_STATUS_EOF;
      return 0;
    }
  }

  // 返回输入输出缓冲区长度减去起始长度
  return fs_length(&nse->iobuf) - startlen;
}


// 处理读取结果的函数
void handle_read_result(struct npool *ms, struct nevent *nse, enum nse_status status) {
  unsigned int count;
  char *str;
  int rc, len;
  struct niod *iod = nse->iod;

  // 如果状态为超时
  if (status == NSE_STATUS_TIMEOUT) {
    // 设置事件完成标志
    nse->event_done = 1;
    // 如果输入输出缓冲区中有数据，设置状态为成功，否则设置状态为超时
    if (fs_length(&nse->iobuf) > 0)
      nse->status = NSE_STATUS_SUCCESS;
    else
      nse->status = NSE_STATUS_TIMEOUT;
  } else if (status == NSE_STATUS_CANCELLED) {
    // 设置状态为取消，并设置事件完成标志
    nse->status = status;
    nse->event_done = 1;
  } else if (status == NSE_STATUS_SUCCESS) {
    // 执行实际读取操作
    rc = do_actual_read(ms, nse);
    /* printf("DBG: Just read %d new bytes%s.\n", rc, iod->ssl? "( SSL!)" : ""); */
  }
    // 如果读取的字节数大于0
    if (rc > 0) {
      // 更新读取计数
      nse->iod->read_count += rc;
      /* 我们决定是否已经读取足够的数据来返回 */
      switch (nse->readinfo.read_type) {
        // 如果是 NSOCK_READ 类型
        case NSOCK_READ:
          // 设置状态为成功
          nse->status = NSE_STATUS_SUCCESS;
          // 标记事件已完成
          nse->event_done = 1;
          break;
        // 如果是 NSOCK_READBYTES 类型
        case NSOCK_READBYTES:
          // 如果缓冲区中的数据长度大于等于指定的字节数
          if (fs_length(&nse->iobuf) >= nse->readinfo.num) {
            // 设置状态为成功
            nse->status = NSE_STATUS_SUCCESS;
            // 标记事件已完成
            nse->event_done = 1;
          }
          /* 否则表示还未完成 */
          break;
        // 如果是 NSOCK_READLINES 类型
        case NSOCK_READLINES:
          /* 让我们计算我们有多少行... */
          count = 0;
          len = fs_length(&nse->iobuf) -1;
          str = fs_str(&nse->iobuf);
          for (count=0; len >= 0; len--) {
            if (str[len] == '\n') {
              count++;
              if ((int)count >= nse->readinfo.num)
                break;
            }
          }
          // 如果行数大于等于指定的行数
          if ((int) count >= nse->readinfo.num) {
            // 标记事件已完成
            nse->event_done = 1;
            // 设置状态为成功
            nse->status = NSE_STATUS_SUCCESS;
          }
          /* 否则表示还未完成 */
          break;
        // 如果是其他未知类型
        default:
          // 报告未知的操作类型
          fatal("Unknown operation type (%d)", (int)nse->readinfo.read_type);
      }
    }
  } else {
    // 报告未知的状态
    fatal("Unknown status (%d)", status);
  }

  /* 如果对于这个 IOD 没有更多的读取操作，我们已经完成了在套接字上的读取
   * 所以我们可以将其从描述符列表中移除... */
  if (nse->event_done && iod->sd >= 0) {
    int ev = EV_NONE;
#if HAVE_OPENSSL
    // 如果使用了 OpenSSL，调用 socket_count_dec_ssl_desire 函数，更新事件状态
    if (nse->iod->ssl != NULL)
      ev |= socket_count_dec_ssl_desire(nse);
    else
#endif
      // 否则，调用 socket_count_read_dec 函数，更新事件状态
      ev |= socket_count_read_dec(nse->iod);
    // 调用 update_events 函数，更新事件状态
    update_events(nse->iod, ms, nse, EV_NONE, ev);
  }
}

#if HAVE_PCAP
// 处理 pcap 读取结果的函数
void handle_pcap_read_result(struct npool *ms, struct nevent *nse, enum nse_status status) {
  struct niod *iod = nse->iod;
  mspcap *mp = (mspcap *)iod->pcap;

  switch (status) {
    // 如果状态为超时，设置事件状态为超时，并标记事件已完成
    case NSE_STATUS_TIMEOUT:
      nse->status = NSE_STATUS_TIMEOUT;
      nse->event_done = 1;
      break;

    // 如果状态为取消，设置事件状态为取消，并标记事件已完成
    case NSE_STATUS_CANCELLED:
      nse->status = NSE_STATUS_CANCELLED;
      nse->event_done = 1;
      break;

    // 如果状态为成功
    case NSE_STATUS_SUCCESS:
      /* check if we already have something read */
      // 检查是否已经有数据被读取
      if (fs_length(&(nse->iobuf)) == 0) {
        // 如果没有数据被读取，设置事件状态为超时，并标记事件未完成
        nse->status = NSE_STATUS_TIMEOUT;
        nse->event_done = 0;
      } else {
        // 如果有数据被读取，设置事件状态为成功，并标记事件已完成
        nse->status = NSE_STATUS_SUCCESS; /* we have full buffer */
        nse->event_done = 1;
      }
      break;

    // 默认情况下，抛出致命错误
    default:
      fatal("Unknown status (%d) for nsock event #%lu", status, nse->id);
  }

  /* If there are no more read events, we are done reading on the socket so we
   * can take it off the descriptor list... */
  // 如果事件已完成，并且 pcap_desc 大于等于 0
  if (nse->event_done && mp->pcap_desc >= 0) {
    int ev;

    // 调用 socket_count_readpcap_dec 函数，更新事件状态
    ev = socket_count_readpcap_dec(iod);
    // 调用 update_events 函数，更新事件状态
    update_events(iod, ms, nse, EV_NONE, ev);
  }
}

/* Returns whether something was read */
// 在非选择模式下进行 pcap 读取
int pcap_read_on_nonselect(struct npool *nsp) {
  gh_lnode_t *current, *next;
  struct nevent *nse;
  int ret = 0;

  // 遍历 pcap_read_events 列表
  for (current = gh_list_first_elem(&nsp->pcap_read_events);
       current != NULL;
       current = next) {
    nse = lnode_nevent2(current);
    // 调用 do_actual_pcap_read 函数，如果有数据被读取，增加 ret 的值
    if (do_actual_pcap_read(nse) == 1) {
      /* something received */
      ret++;
      break;
    }
    next = gh_lnode_next(current);
  }
  return ret;
}
#endif /* HAVE_PCAP */
/* 这是一个非常重要的循环函数，告诉事件引擎启动并开始处理事件。它会一直运行，直到所有事件都被传递（包括从事件处理程序启动的新事件），或者达到了msec_timeout，或者发生了重大错误。如果不想设置最大运行时间，可以使用-1。超时为0将在1个非阻塞循环后返回。nsock循环在返回后可以重新启动。例如，您可以进行一系列15秒的运行，允许您在它们之间做其他事情 */
enum nsock_loopstatus nsock_loop(nsock_pool nsp, int msec_timeout) {
  struct npool *ms = (struct npool *)nsp;
  struct timeval loop_timeout;
  int msecs_left;
  unsigned long loopnum = 0;
  enum nsock_loopstatus quitstatus = NSOCK_LOOP_ERROR;

  gettimeofday(&nsock_tod, NULL);

  if (msec_timeout < -1) {
    ms->errnum = EINVAL;
    return NSOCK_LOOP_ERROR;
  }
  TIMEVAL_MSEC_ADD(loop_timeout, nsock_tod, msec_timeout);
  msecs_left = msec_timeout;

  if (msec_timeout >= 0)
    nsock_log_debug("nsock_loop() started (timeout=%dms). %d events pending",
                    msec_timeout, ms->events_pending);
  else
    nsock_log_debug("nsock_loop() started (no timeout). %d events pending",
                    ms->events_pending);

  while (1) {
    if (ms->quit) {
      /* 通过nsock_loop_quit要求退出循环 */
      ms->quit = 0;
      quitstatus = NSOCK_LOOP_QUIT;
      break;
    }

    if (ms->events_pending == 0) {
      /* 如果没有任何事件挂起，那么在我们退出nsock_loop()之前，就不会创建任何事件。 */
      quitstatus = NSOCK_LOOP_NOEVENTS;
      break;
    }

    if (msec_timeout >= 0) {
      msecs_left = MAX(0, TIMEVAL_MSEC_SUBTRACT(loop_timeout, nsock_tod));
      if (msecs_left == 0 && loopnum > 0) {
        quitstatus = NSOCK_LOOP_TIMEOUT;
        break;
      }
    }
    # 如果 nsock_engine_loop 返回 -1，表示出现错误，更新退出状态并跳出循环
    if (nsock_engine_loop(ms, msecs_left) == -1) {
      quitstatus = NSOCK_LOOP_ERROR;
      break;
    }

    # 获取当前时间，用于记录循环次数
    gettimeofday(&nsock_tod, NULL); /* we do this at end because there is one
                                     * at beginning of function */
    # 循环次数加一
    loopnum++;
  }

  # 返回退出状态
  return quitstatus;
// 处理事件的函数，接受一个指向事件列表的指针，一个指向事件列表的指针，一个指向事件的指针，一个表示事件类型的整数
void process_event(struct npool *nsp, gh_list_t *evlist, struct nevent *nse, int ev) {
  // 检查是否匹配读事件
  int match_r = ev & EV_READ;
  // 检查是否匹配写事件
  int match_w = ev & EV_WRITE;
  // 检查是否匹配异常事件
  int match_x = ev & EV_EXCEPT;
#if HAVE_OPENSSL
  // 如果有 OpenSSL，初始化 desire_r 和 desire_w
  int desire_r = 0, desire_w = 0;
#endif

  // 记录调试日志
  nsock_log_debug_all("Processing event %lu (timeout in %ldms, done=%d)",
                      nse->id,
                      (long)TIMEVAL_MSEC_SUBTRACT(nse->timeout, nsock_tod),
                      nse->event_done);

  // 如果事件未完成
  if (!nse->event_done) {
    // 根据事件类型进行处理
    switch (nse->type) {
      // 如果是连接或 SSL 连接
      case NSE_TYPE_CONNECT:
      case NSE_TYPE_CONNECT_SSL:
        // 如果事件不是 EV_NONE，处理连接结果
        if (ev != EV_NONE)
          handle_connect_result(nsp, nse, NSE_STATUS_SUCCESS);
        // 如果事件超时，处理连接结果
        if (event_timedout(nse))
          handle_connect_result(nsp, nse, NSE_STATUS_TIMEOUT);
        break;

      // 如果是读事件
      case NSE_TYPE_READ:
#if HAVE_OPENSSL
        // 如果有 OpenSSL，检查 SSL desire，并根据匹配的读写事件处理读结果
        desire_r = nse->sslinfo.ssl_desire == SSL_ERROR_WANT_READ;
        desire_w = nse->sslinfo.ssl_desire == SSL_ERROR_WANT_WRITE;
        if (nse->iod->ssl && ((desire_r && match_r) || (desire_w && match_w)))
          handle_read_result(nsp, nse, NSE_STATUS_SUCCESS);
        else
#endif
        // 如果没有 SSL 或匹配了读或异常事件，处理读结果
        if ((!nse->iod->ssl && match_r) || match_x)
          handle_read_result(nsp, nse, NSE_STATUS_SUCCESS);

        // 如果事件超时，处理读结果
        if (event_timedout(nse))
          handle_read_result(nsp, nse, NSE_STATUS_TIMEOUT);

        break;

      // 如果是写事件
      case NSE_TYPE_WRITE:
#if HAVE_OPENSSL
        // 如果有 OpenSSL，检查 SSL desire，并根据匹配的读写事件处理写结果
        desire_r = nse->sslinfo.ssl_desire == SSL_ERROR_WANT_READ;
        desire_w = nse->sslinfo.ssl_desire == SSL_ERROR_WANT_WRITE;
        if (nse->iod->ssl && ((desire_r && match_r) || (desire_w && match_w)))
          handle_write_result(nsp, nse, NSE_STATUS_SUCCESS);
        else
#endif
          // 如果没有 SSL 并且匹配写事件，或者匹配超时事件
          if ((!nse->iod->ssl && match_w) || match_x)
            // 处理写事件结果
            handle_write_result(nsp, nse, NSE_STATUS_SUCCESS);

        // 如果事件超时
        if (event_timedout(nse))
          // 处理写事件结果为超时
          handle_write_result(nsp, nse, NSE_STATUS_TIMEOUT);
        // 退出 switch 语句
        break;

      // 如果事件类型为定时器
      case NSE_TYPE_TIMER:
        // 如果事件超时
        if (event_timedout(nse))
          // 处理定时器事件结果
          handle_timer_result(nsp, nse, NSE_STATUS_SUCCESS);
        // 退出 switch 语句
        break;
#if HAVE_PCAP
      case NSE_TYPE_PCAP_READ:{
        nsock_log_debug_all("PCAP iterating %lu", nse->id);

        if (ev & EV_READ) {
          /* 如果缓冲区为空，进行实际的 pcap 读取 */
          if (fs_length(&(nse->iobuf)) == 0)
            do_actual_pcap_read(nse);
        }

        /* 如果已经接收到数据 */
        if (fs_length(&(nse->iobuf)) > 0)
          handle_pcap_read_result(nsp, nse, NSE_STATUS_SUCCESS);

        /* 如果事件超时 */
        if (event_timedout(nse))
          handle_pcap_read_result(nsp, nse, NSE_STATUS_TIMEOUT);

        #if PCAP_BSD_SELECT_HACK
        /* 如果事件发生，并且处于 BSD_HACK 模式，则此事件被添加到两个队列中。read_event 和 pcap_read_event
         * 当然，我们应该只销毁一次。
         * 我们假设现在处于 read_event，所以只需从 pcap_read_event 中取消链接此事件 */
        if (((mspcap *)nse->iod->pcap)->pcap_desc >= 0
            && nse->event_done
            && evlist == &nsp->read_events) {
          /* 事件已完成，列表是 read_events，并且我们处于 BSD_HACK 模式。
           * 因此从 pcap_read_events 中取消链接事件 */
          update_first_events(nse);
          gh_list_remove(&nsp->pcap_read_events, &nse->nodeq_pcap);

          nsock_log_debug_all("PCAP NSE #%lu: Removing event from PCAP_READ_EVENTS",
                              nse->id);
        }
        if (((mspcap *)nse->iod->pcap)->pcap_desc >= 0
            && nse->event_done
            && evlist == &nsp->pcap_read_events) {
          update_first_events(nse);
          gh_list_remove(&nsp->read_events, &nse->nodeq_io);
          nsock_log_debug_all("PCAP NSE #%lu: Removing event from READ_EVENTS",
                              nse->id);
        }
        #endif
        break;
      }
#endif
      default:
        fatal("Event has unknown type (%d)", nse->type);
    }
  }

  if (nse->event_done) {
    /* 安全性检查：不要返回一个功能性的 SSL iod，而没有设置 SSL 数据结构。 */
    # 如果网络安全事件的类型是连接 SSL 并且状态是成功，则断言 SSL 对象不为空
    if (nse->type == NSE_TYPE_CONNECT_SSL && nse->status == NSE_STATUS_SUCCESS)
      assert(nse->iod->ssl != NULL);

    # 记录调试信息：发送事件
    nsock_log_debug_all("NSE #%lu: Sending event", nse->id);

    # 事件准备好发送，调用事件分发函数并删除事件
    event_dispatch_and_delete(nsp, nse, 1);
  }
}
void process_iod_events(struct npool *nsp, struct niod *nsi, int ev) {
  int i = 0;
  /* 存储每种事件类型的第一个元素的指针地址，而不是存储值，因为连接可能会添加读取事件 */
  gh_lnode_t **start_elems[] = {
    &nsi->first_connect,
    &nsi->first_read,
    &nsi->first_write,
#if HAVE_PCAP
    &nsi->first_pcap_read,
#endif
    NULL
  };
  gh_list_t *evlists[] = {
    &nsp->connect_events,
    &nsp->read_events,
    &nsp->write_events,
#if HAVE_PCAP
    &nsp->pcap_read_events,
#endif
    NULL
  };

  assert(nsp == nsi->nsp);
  nsock_log_debug_all("Processing events on IOD %lu (ev=%d)", nsi->id, ev);

  /* 我们将事件分开处理，因为我们希望按顺序处理它们：连接 => 读取 => 写入 => 定时器，有几个原因：

   *  1) 确保在定时器到期之前已经处理了所有网络 I/O 事件（在数据可用但我们尚未传递事件之前超时会很遗憾）
   
   *  2) connect() 的结果通常会导致在同一周期内处理读取或写入。同样，read() 通常会导致写入。
   */
  for (i = 0; evlists[i] != NULL; i++) {
    gh_lnode_t *current, *next, *last;

    /* 对于每个列表，获取最后一个事件，并不要查看它之后的事件，因为一个事件可能会在同一列表中添加另一个事件，依此类推... */
    last = gh_list_last_elem(evlists[i]);

    for (current = *start_elems[i];
         current != NULL && gh_lnode_prev(current) != last;
         current = next) {
      struct nevent *nse;

#if HAVE_PCAP
      if (evlists[i] == &nsi->nsp->pcap_read_events)
        nse = lnode_nevent2(current);
      else
#endif
        // 获取当前事件的IOD
        nse = lnode_nevent(current);

      /* events are grouped by IOD. Break if we're done with the events for the
       * current IOD */
      // 事件按IOD分组。如果当前IOD的事件处理完毕，则跳出循环
      if (nse->iod != nsi)
        break;

      // 处理事件
      process_event(nsp, evlists[i], nse, ev);
      // 获取下一个事件
      next = gh_lnode_next(current);

      // 如果事件已完成
      if (nse->event_done) {
        /* event is done, remove it from the event list and update IOD pointers
         * to the first events of each kind */
        // 事件完成后，从事件列表中移除，并更新每种类型事件的IOD指针
        update_first_events(nse);
        gh_list_remove(evlists[i], current);
        gh_list_append(&nsp->free_events, &nse->nodeq_io);

        // 如果有超时时间，则从堆中移除
        if (nse->timeout.tv_sec)
          gh_heap_remove(&nsp->expirables, &nse->expire);
      }
    }
  }
}

static int nevent_unref(struct npool *nsp, struct nevent *nse) {
  switch (nse->type) {
    case NSE_TYPE_CONNECT:
    case NSE_TYPE_CONNECT_SSL:
      // 移除连接事件
      gh_list_remove(&nsp->connect_events, &nse->nodeq_io);
      break;

    case NSE_TYPE_READ:
      // 移除读事件
      gh_list_remove(&nsp->read_events, &nse->nodeq_io);
      break;

    case NSE_TYPE_WRITE:
      // 移除写事件
      gh_list_remove(&nsp->write_events, &nse->nodeq_io);
      break;

#if HAVE_PCAP
    case NSE_TYPE_PCAP_READ: {
      char read = 0;
      char pcap = 0;

#if PCAP_BSD_SELECT_HACK
      read = pcap = 1;
#else
      // 根据条件判断是否移除读事件或者PCAP读事件
      if (((mspcap *)nse->iod->pcap)->pcap_desc >= 0)
        read = 1;
      else
        pcap = 1;
#endif /* PCAP_BSD_SELECT_HACK */

      if (read)
        gh_list_remove(&nsp->read_events, &nse->nodeq_io);
      if (pcap)
        gh_list_remove(&nsp->pcap_read_events, &nse->nodeq_pcap);

      break;
    }
#endif /* HAVE_PCAP */

    case NSE_TYPE_TIMER:
      /* Nothing to do */
      // 定时器事件无需处理
      break;

    default:
      // 未知事件类型，报错
      fatal("Unknown event type %d", nse->type);
  }
  // 将事件添加到空闲事件列表中
  gh_list_append(&nsp->free_events, &nse->nodeq_io);
  return 0;
}

void process_expired_events(struct npool *nsp) {
  for (;;) {
    gh_hnode_t *hnode;
    struct nevent *nse;

    // 获取堆中最小的事件
    hnode = gh_heap_min(&nsp->expirables);
    if (!hnode)
      break;
    # 通过包含指向结构体的指针hnode，获取结构体nevent的指针nse
    nse = container_of(hnode, struct nevent, expire);
    # 如果事件未超时，则跳出循环
    if (!event_timedout(nse))
      break;

    # 从nsp->expirables堆中移除hnode节点
    gh_heap_remove(&nsp->expirables, hnode);
    # 处理事件，传入参数nsp和nse，事件类型为EV_NONE
    process_event(nsp, NULL, nse, EV_NONE);
    # 断言nse->event_done为真
    assert(nse->event_done);
    # 更新第一个事件
    update_first_events(nse);
    # 释放nse的引用
    nevent_unref(nsp, nse);
  }
/* 调用此函数将导致 nsock_loop 在下一次迭代时以 NSOCK_LOOP_QUIT 的返回值退出 */
void nsock_loop_quit(nsock_pool nsp) {
  // 将 nsp 转换为 npool 结构体指针
  struct npool *ms = (struct npool *)nsp;
  // 设置 quit 标志为 1，表示退出循环
  ms->quit = 1;
}

/* 获取 nsock 库记录的最新时间，nsock 至少在每个事件循环（在 main_loop 中）记录一次时间。
 * 这个函数（通常）不仅避免了系统调用，而且在许多情况下最好使用 nsock 的时间而不是系统时间。
 * 如果在调用时 nsock 从未获取过时间，它将在返回之前获取时间 */
const struct timeval *nsock_gettimeofday() {
  // 如果 nsock_tod 的秒数为 0，则调用 gettimeofday 获取当前时间
  if (nsock_tod.tv_sec == 0)
    gettimeofday(&nsock_tod, NULL);
  // 返回 nsock_tod 的地址
  return &nsock_tod;
}

/* 将事件添加到适当的 nsp 事件列表中，处理诸如调整描述符选择/轮询列表、注册超时值等的工作 */
void nsock_pool_add_event(struct npool *nsp, struct nevent *nse) {
  // 记录调试信息，包括事件的编号和超时时间
  nsock_log_debug("NSE #%lu: Adding event (timeout in %ldms)",
                  nse->id,
                  (long)TIMEVAL_MSEC_SUBTRACT(nse->timeout, nsock_tod));

  // 增加待处理事件的计数
  nsp->events_pending++;

  // 如果事件未完成且具有超时时间，则将其添加到队列中
  if (!nse->event_done && nse->timeout.tv_sec) {
    gh_heap_push(&nsp->expirables, &nse->expire);
  }

  // 根据事件类型执行特定的操作
  switch (nse->type) {
    case NSE_TYPE_CONNECT:
    case NSE_TYPE_CONNECT_SSL:
      if (!nse->event_done) {
        assert(nse->iod->sd >= 0);
        socket_count_read_inc(nse->iod);
        socket_count_write_inc(nse->iod);
        update_events(nse->iod, nsp, nse, EV_READ|EV_WRITE, EV_NONE);
      }
      iod_add_event(nse->iod, nse);
      break;

    case NSE_TYPE_READ:
      if (!nse->event_done) {
        assert(nse->iod->sd >= 0);
        socket_count_read_inc(nse->iod);
        update_events(nse->iod, nsp, nse, EV_READ, EV_NONE);
#if HAVE_OPENSSL
        // 如果支持 OpenSSL，则设置 SSL 错误为需要读取
        if (nse->iod->ssl)
          nse->sslinfo.ssl_desire = SSL_ERROR_WANT_READ;
#endif
      }
      // 将事件添加到 IOD 中
      iod_add_event(nse->iod, nse);
      break;

    case NSE_TYPE_WRITE:
      // 如果事件未完成
      if (!nse->event_done) {
        // 断言 IOD 的套接字描述符大于等于 0
        assert(nse->iod->sd >= 0);
        // 增加 IOD 的写入计数
        socket_count_write_inc(nse->iod);
        // 更新事件
        update_events(nse->iod, nsp, nse, EV_WRITE, EV_NONE);
#if HAVE_OPENSSL
        // 如果支持 OpenSSL，则设置 SSL 错误为需要写入
        if (nse->iod->ssl)
          nse->sslinfo.ssl_desire = SSL_ERROR_WANT_WRITE;
#endif
      }
      // 将事件添加到 IOD 中
      iod_add_event(nse->iod, nse);
      break;

    case NSE_TYPE_TIMER:
      /* 无需操作 */
      break;

#if HAVE_PCAP
    case NSE_TYPE_PCAP_READ: {
      mspcap *mp = (mspcap *)nse->iod->pcap;

      assert(mp);
      // 如果 pcap 描述符大于等于 0（即存在）
      if (mp->pcap_desc >= 0) { 
        // 如果事件未完成
        if (!nse->event_done) {
          // 增加 IOD 的读取 pcap 计数
          socket_count_readpcap_inc(nse->iod);
          // 更新事件
          update_events(nse->iod, nsp, nse, EV_READ, EV_NONE);
        }
        // 记录调试信息
        nsock_log_debug_all("PCAP NSE #%lu: Adding event to READ_EVENTS", nse->id);

        #if PCAP_BSD_SELECT_HACK
        /* 当使用 BSD hack 时，我们必须在 select() 之后执行 pcap_next()。
         * 让我们将此 pcap 插入到两个队列中，可选择和不可选择的。
         * 这将导致在 select() 之前执行 pcap_next_ex() */
        // 记录调试信息
        nsock_log_debug_all("PCAP NSE #%lu: Adding event to PCAP_READ_EVENTS", nse->id);
        #endif
      } else {
        /* pcap 不可选择。将其添加到特定于 pcap 的队列中。 */
        // 记录调试信息
        nsock_log_debug_all("PCAP NSE #%lu: Adding event to PCAP_READ_EVENTS", nse->id);
      }
      // 将事件添加到 IOD 中
      iod_add_event(nse->iod, nse);
      break;
    }
#endif

    default:
      // 报告未知的 nsock 事件类型
      fatal("Unknown nsock event type (%d)", nse->type);
  }

  /* 可能出现事件已经完成的情况。在这种情况下，我们可以立即传递它，即使我们可能不在 nsock_loop() 中。 */
  if (nse->event_done) {
    // 分发并删除事件
    event_dispatch_and_delete(nsp, nse, 1);
    // 更新首个事件
    update_first_events(nse);
    // 减少事件的引用计数
    nevent_unref(nsp, nse);
  }
}
/* 事件已完成，处理程序即将被调用。如果需要，此函数会写出有关事件的跟踪数据 */
void nsock_trace_handler_callback(struct npool *ms, struct nevent *nse) {
  struct niod *nsi; // 定义指向 niod 结构体的指针
  char *str; // 定义指向字符的指针
  int strlength = 0; // 初始化字符长度为 0
  char displaystr[256]; // 定义长度为 256 的字符数组
  char errstr[256]; // 定义长度为 256 的错误信息数组

  if (NsockLogLevel > NSOCK_LOG_INFO) // 如果日志级别大于信息级别，直接返回
    return;

  nsi = nse->iod; // 将事件的 I/O 数据结构赋值给 nsi

  if (nse->status == NSE_STATUS_ERROR) // 如果事件状态为错误
    Snprintf(errstr, sizeof(errstr), "[%s (%d)] ", socket_strerror(nse->errnum),
             nse->errnum); // 格式化错误信息字符串
  else
    errstr[0] = '\0'; // 否则清空错误信息字符串

  /* 一些类型有特殊的跟踪处理 */
  switch (nse->type) {
    case NSE_TYPE_CONNECT:
    case NSE_TYPE_CONNECT_SSL:
      nsock_log_info("Callback: %s %s %sfor EID %li [%s]",
                     nse_type2str(nse->type), nse_status2str(nse->status),
                     errstr, nse->id, get_peeraddr_string(nsi)); // 记录连接类型的回调信息
      break;

    case NSE_TYPE_READ:
      if (nse->status != NSE_STATUS_SUCCESS) { // 如果读取状态不成功
        nsock_log_info("Callback: %s %s %sfor EID %li [%s]",
                       nse_type2str(nse->type), nse_status2str(nse->status),
                       errstr, nse->id, get_peeraddr_string(nsi)); // 记录读取类型的回调信息
      } else {
        str = nse_readbuf(nse, &strlength); // 读取事件的缓冲区内容和长度
        if (strlength < 80) { // 如果长度小于 80
          memcpy(displaystr, ": ", 2); // 复制字符串
          memcpy(displaystr + 2, str, strlength); // 复制字符串
          displaystr[2 + strlength] = '\0'; // 添加结束符
          replacenonprintable(displaystr + 2, strlength, '.'); // 替换不可打印字符
        } else {
          displaystr[0] = '\0'; // 否则清空显示字符串
        }
        nsock_log_info("Callback: %s %s for EID %li [%s] %s(%d bytes)%s",
                       nse_type2str(nse->type), nse_status2str(nse->status),
                       nse->id,
                       get_peeraddr_string(nsi),
                       nse_eof(nse) ? "[EOF]" : "", strlength, displaystr); // 记录读取类型的回调信息
      }
      break;
    # 如果是写入类型的回调
    case NSE_TYPE_WRITE:
      # 记录日志信息，包括回调类型、状态、错误信息、EID和对等地址
      nsock_log_info("Callback: %s %s %sfor EID %li [%s]",
                     nse_type2str(nse->type), nse_status2str(nse->status),
                     errstr, nse->id, get_peeraddr_string(nsi));
      # 结束当前的 case
      break;

    # 如果是定时器类型的回调
    case NSE_TYPE_TIMER:
      # 记录日志信息，包括回调类型、状态、错误信息和EID
      nsock_log_info("Callback: %s %s %sfor EID %li",
                     nse_type2str(nse->type), nse_status2str(nse->status),
                     errstr, nse->id);
      # 结束当前的 case
      break;
#if HAVE_PCAP
    // 如果定义了 HAVE_PCAP，则执行以下代码块
    case NSE_TYPE_PCAP_READ:
      // 打印回调信息，包括类型、状态、错误信息和 ID
      nsock_log_info("Callback: %s %s %sfor EID %li ",
                     nse_type2str(nse->type), nse_status2str(nse->status),
                     errstr, nse->id);
      // 跳出 switch 语句
      break;
#endif

    // 如果不是 NSE_TYPE_PCAP_READ 类型的事件
    default:
      // 输出错误信息，指出 nse 类型无效
      fatal("Invalid nsock event type (%d)", nse->type);
    // 结束 switch 语句
  }
}
```