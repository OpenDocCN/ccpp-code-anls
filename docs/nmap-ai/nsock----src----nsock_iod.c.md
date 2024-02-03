# `nmap\nsock\src\nsock_iod.c`

```cpp
/* $Id$ */

#include "nsock.h"  // 包含 nsock 库的头文件
#include "nsock_internal.h"  // 包含 nsock 内部实现的头文件
#include "nsock_log.h"  // 包含 nsock 日志的头文件
#include "gh_list.h"  // 包含通用哈希列表的头文件
#include "netutils.h"  // 包含网络工具的头文件

#if HAVE_PCAP
#include "nsock_pcap.h"  // 如果支持 pcap，则包含 pcap 的头文件
#endif

#include <string.h>  // 包含字符串操作的头文件


/* nsock_iod is like a "file descriptor" for the nsock library. You use it to
 * request events. And here is how you create an nsock_iod. nsock_iod_new returns
 * NULL if the iod cannot be allocated. Pass NULL as userdata if you don't want
 * to immediately associate any user data with the iod. */
nsock_iod nsock_iod_new(nsock_pool nsockp, void *userdata) {
  return nsock_iod_new2(nsockp, -1, userdata);  // 调用 nsock_iod_new2 函数创建一个新的 nsock_iod
}

/* This version allows you to associate an existing sd with the msi so that you
 * can read/write it using the nsock infrastructure.  For example, you may want
 * to watch for data from STDIN_FILENO at the same time as you read/write
 * various sockets.  STDIN_FILENO is a special case, however. Any other sd is
 * dup()ed, so you may close or otherwise manipulate your copy.  The duped copy
 * will be destroyed when the nsi is destroyed. */
nsock_iod nsock_iod_new2(nsock_pool nsockp, int sd, void *userdata) {
  struct npool *nsp = (struct npool *)nsockp;  // 将 nsock_pool 转换为 npool 结构体
  gh_lnode_t *lnode;  // 定义通用哈希列表节点
  struct niod *nsi;  // 定义 niod 结构体

  lnode = gh_list_pop(&nsp->free_iods);  // 从空闲的 iod 列表中弹出一个节点
  if (!lnode) {  // 如果节点为空
    nsi = (struct niod *)safe_malloc(sizeof(*nsi));  // 分配 niod 结构体的内存空间
    memset(nsi, 0, sizeof(*nsi));  // 将 niod 结构体的内存空间清零
  } else {
    nsi = container_of(lnode, struct niod, nodeq);  // 将节点转换为 niod 结构体
  }

  if (sd == -1) {  // 如果 sd 为 -1
    nsi->sd = -1;  // 设置 niod 结构体的 sd 为 -1
    nsi->state = NSIOD_STATE_INITIAL;  // 设置 niod 结构体的状态为初始状态
  } else if (sd == STDIN_FILENO) {  // 如果 sd 为标准输入文件描述符
    nsi->sd = STDIN_FILENO;  // 设置 niod 结构体的 sd 为标准输入文件描述符
    nsi->state = NSIOD_STATE_UNKNOWN;  // 设置 niod 结构体的状态为未知状态
  } else {  // 其他情况
    nsi->sd = dup_socket(sd);  // 复制 sd 的套接字
    if (nsi->sd == -1) {  // 如果复制失败
      free(nsi);  // 释放 niod 结构体的内存空间
      return NULL;  // 返回空指针
    }
    unblock_socket(nsi->sd);  // 设置套接字为非阻塞模式
    nsi->state = NSIOD_STATE_UNKNOWN;  // 设置 niod 结构体的状态为未知状态
  }

  nsi->first_connect = NULL;  // 初始化 niod 结构体的首次连接为 NULL
  nsi->first_read = NULL;  // 初始化 niod 结构体的首次读取为 NULL
  nsi->first_write = NULL;  // 初始化 niod 结构体的首次写入为 NULL
#if HAVE_PCAP
  nsi->first_pcap_read = NULL;  // 如果支持 pcap，则初始化 niod 结构体的首次 pcap 读取为 NULL
  nsi->readpcapsd_count = 0;  // 如果支持 pcap，则初始化 niod 结构体的 pcap 读取套接字计数为 0
#endif
  // 重置读取数据的计数
  nsi->readsd_count = 0;
  // 重置写入数据的计数
  nsi->write_count = 0;

  // 设置用户数据
  nsi->userdata = userdata;
  // 设置网络池指针
  nsi->nsp = (struct npool *)nsockp;

  // 初始化标志位
  nsi->_flags = 0;

  // 重置读取数据的计数
  nsi->read_count = 0;
  // 重置写入数据的计数
  nsi->write_count = 0;

  // 主机名置空
  nsi->hostname = NULL;

  // IP 选项置空
  nsi->ipopts = NULL;
  nsi->ipoptslen = 0;

  // 如果支持 OpenSSL，则初始化 SSL 会话
#if HAVE_OPENSSL
  nsi->ssl_session = NULL;
#endif

  // 如果存在代理链，则创建代理链上下文
  if (nsp->px_chain) {
    nsi->px_ctx = proxy_chain_context_new(nsp);
  } else {
    nsi->px_ctx = NULL;
  }

  // 设置 IOD 的 ID
  nsi->id = nsp->next_iod_serial++;
  if (nsi->id == 0)
    nsi->id = nsp->next_iod_serial++;

  /* nsp 保持跟踪活动的 IOD，以便在删除时删除它们 */
  gh_list_append(&nsp->active_iods, &nsi->nodeq);

  // 记录日志
  nsock_log_info("nsock_iod_new (IOD #%lu)", nsi->id);

  // 返回新创建的 IOD
  return (nsock_iod)nsi;
}

/* Defined in nsock_core.c. */
// 检查套接字计数是否为零
int socket_count_zero(struct niod *iod, struct npool *ms);

/* 如果 nsock_iod_new 返回成功，当使用完 IOD 时必须释放它以节省内存（在某些情况下还可以释放套接字）。
 * 在此调用之后，nsockiod 可能不再可用 -- 需要使用 nsock_iod_new() 创建一个新的。
 * pending_response 告诉如何处理任何挂起在此 nsock_iod 上的事件。
 * 这可以是 NSOCK_PENDING_NOTIFY（向每个事件发送 KILL 通知）、NSOCK_PENDING_SILENT（不向被杀死的事件发送通知），
 * 或 NSOCK_PENDING_ERROR（打印错误消息并退出程序） */
void nsock_iod_delete(nsock_iod nsockiod, enum nsock_del_mode pending_response) {
#if HAVE_PCAP
#define NUM_EVT_TYPES 4
#else
#define NUM_EVT_TYPES 3
#endif
  // 将 nsockiod 转换为 niod 指针
  struct niod *nsi = (struct niod *)nsockiod;
  // 事件列表数组
  gh_lnode_t *evlist_ar[NUM_EVT_TYPES];
  // 对应列表数组
  gh_list_t *corresp_list[NUM_EVT_TYPES];
  int i;
  gh_lnode_t *current, *next;

  // 断言 nsi 不为空
  assert(nsi);

  // 如果 nsi 的状态已经标记为删除，则避免破坏重入性，直接返回
  if (nsi->state == NSIOD_STATE_DELETED) {
    /* This nsi is already marked as deleted, will probably be removed from the
     * list very soon. Just return to avoid breaking reentrancy. */
    # 返回空值
    return;
  }

  # 记录日志，标记删除的 IOD 编号
  nsock_log_info("nsock_iod_delete (IOD #%lu)", nsi->id);

  # 如果有待处理的事件
  if (nsi->events_pending > 0) {
    # 如果在处理事件时删除了 struct niod 结构体，则报错
    if (pending_response == NSOCK_PENDING_ERROR)
      fatal("nsock_iod_delete called with argument NSOCK_PENDING_ERROR on a nsock_iod that has %d pending event(s) associated with it", nsi->events_pending);

    # 确保 pending_response 的值为 NSOCK_PENDING_NOTIFY 或 NSOCK_PENDING_SILENT
    assert(pending_response == NSOCK_PENDING_NOTIFY || pending_response == NSOCK_PENDING_SILENT);

    # 将待处理的事件存储到事件列表中
    evlist_ar[0] = nsi->first_connect;
    evlist_ar[1] = nsi->first_read;
    evlist_ar[2] = nsi->first_write;
#if HAVE_PCAP
    // 如果有 pcap，将第三个事件列表设置为第一个 pcap 读取事件
    evlist_ar[3] = nsi->first_pcap_read;
#endif

    // 将对应列表的第一个元素设置为连接事件列表
    corresp_list[0] = &nsi->nsp->connect_events;
    // 将对应列表的第二个元素设置为读取事件列表
    corresp_list[1] = &nsi->nsp->read_events;
    // 将对应列表的第三个元素设置为写入事件列表
    corresp_list[2] = &nsi->nsp->write_events;
#if HAVE_PCAP
    // 如果有 pcap，将对应列表的第四个元素设置为 pcap 读取事件列表
    corresp_list[3] = &nsi->nsp->pcap_read_events;
#endif

    // 遍历事件类型数组，直到事件挂起数为 0 或者遍历完所有事件类型
    for (i = 0; i < NUM_EVT_TYPES && nsi->events_pending > 0; i++) {
      for (current = evlist_ar[i]; current != NULL; current = next) {
        struct nevent *nse;

        next = gh_lnode_next(current);
        nse = lnode_nevent(current);

        // 如果事件的 IOD 不等于当前 IOD，跳出循环
        if (nse->iod != nsi)
          break;

        // 删除事件
        nevent_delete(nsi->nsp, nse, corresp_list[i], current, pending_response == NSOCK_PENDING_NOTIFY);
      }
    }
  }

  // 如果事件挂起数不为 0，输出错误信息
  if (nsi->events_pending != 0)
    fatal("Trying to delete NSI, but could not find %d of the purportedly pending events on that IOD.\n", nsi->events_pending);

  // 确保不再选择此套接字，以防套接字计数已经减少到零
  if (nsi->sd >= 0)
    socket_count_zero(nsi, nsi->nsp);

  // 释放主机名内存
  free(nsi->hostname);

#if HAVE_OPENSSL
  // 关闭任何 SSL 资源
  if (nsi->ssl) {
    // 如果有 SSL 会话，释放 SSL 会话
#if 0
    if (nsi->ssl_session)
    SSL_SESSION_free(nsi->ssl_session);
    nsi->ssl_session = NULL;
#endif

    // 如果 SSL 关闭失败，输出错误信息
    if (SSL_shutdown(nsi->ssl) == -1) {
      nsock_log_info("nsock_iod_delete: SSL shutdown failed (%s) on NSI %li",
                     ERR_reason_error_string(SSL_get_error(nsi->ssl, -1)), nsi->id);
    }

    // 释放 SSL 对象
    SSL_free(nsi->ssl);
    nsi->ssl = NULL;
  }
#endif

  // 如果套接字大于等于 0 且不等于标准输入文件描述符，关闭套接字
  if (nsi->sd >= 0 && nsi->sd != STDIN_FILENO) {
    close(nsi->sd);
    # 将 nsi 结构体中的 sd 成员变量设置为 -1
    nsi->sd = -1;
  }

  # 将 nsi 结构体中的 state 成员变量设置为 NSIOD_STATE_DELETED
  nsi->state = NSIOD_STATE_DELETED;
  # 将 nsi 结构体中的 userdata 成员变量设置为 NULL
  nsi->userdata = NULL;

  # 如果 nsi 结构体中的 ipoptslen 成员变量不为 0，则释放 ipopts 指向的内存空间
  if (nsi->ipoptslen)
    free(nsi->ipopts);
#if HAVE_PCAP
  // 如果支持 pcap
  if (nsi->pcap){
    // 如果 pcap 存在
    mspcap *mp = (mspcap *)nsi->pcap;
    // 将 nsi->pcap 转换为 mspcap 类型的指针 mp

    if (mp->pt){
      // 如果 mp->pt 存在
      pcap_close(mp->pt);
      // 关闭 pcap
      mp->pt = NULL;
      // 将 mp->pt 置为 NULL
    }
    if (mp->pcap_desc) {
      // 如果 mp->pcap_desc 存在
      /* pcap_close() will close the associated pcap descriptor */
      // pcap_close() 将关闭相关的 pcap 描述符
      mp->pcap_desc = -1;
      // 将 mp->pcap_desc 置为 -1
    }
    if (mp->pcap_device) {
      // 如果 mp->pcap_device 存在
      free(mp->pcap_device);
      // 释放 mp->pcap_device
      mp->pcap_device = NULL;
      // 将 mp->pcap_device 置为 NULL
    }
    free(mp);
    // 释放 mp
    nsi->pcap = NULL;
    // 将 nsi->pcap 置为 NULL
  }
#endif

  if (nsi->px_ctx)
    // 如果 nsi->px_ctx 存在
    proxy_chain_context_delete(nsi->px_ctx);
    // 删除代理链上下文

/* Returns the ID of an nsock_iod . This ID is always unique amongst ids for a
 * given nspool (unless you blow through billions of them). */
// 返回 nsock_iod 的 ID。该 ID 在给定 nspool 中始终是唯一的（除非你用完了数十亿个 ID）。
unsigned long nsock_iod_id(nsock_iod nsockiod) {
  // 断言 nsockiod 不为空
  assert(nsockiod);
  // 返回 nsockiod 的 id
  return ((struct niod *)nsockiod)->id;
}

/* Returns the SSL object inside an nsock_iod, or NULL if unset. */
// 返回 nsock_iod 中的 SSL 对象，如果未设置则返回 NULL。
nsock_ssl nsock_iod_get_ssl(nsock_iod iod) {
#if HAVE_OPENSSL
  // 如果支持 OpenSSL
  return ((struct niod *)iod)->ssl;
  // 返回 iod 中的 ssl 对象
#else
  return NULL;
  // 否则返回 NULL
#endif
}

/* Returns the SSL_SESSION of an nsock_iod.
 * Increments its usage count if inc_ref is not zero. */
// 返回 nsock_iod 的 SSL_SESSION。如果 inc_ref 不为零，则增加其使用计数。
nsock_ssl_session nsock_iod_get_ssl_session(nsock_iod iod, int inc_ref) {
#if HAVE_OPENSSL
  // 如果支持 OpenSSL
  if (inc_ref)
    // 如果 inc_ref 不为零
    return SSL_get1_session(((struct niod *)iod)->ssl);
    // 返回 SSL_get1_session(((struct niod *)iod)->ssl)
  else
    // 否则
    return SSL_get0_session(((struct niod *)iod)->ssl);
    // 返回 SSL_get0_session(((struct niod *)iod)->ssl)
#else
  return NULL;
  // 否则返回 NULL
#endif
}

/* sets the ssl session of an nsock_iod, increments usage count. The session
 * should not have been set yet (as no freeing is done) */
// 设置 nsock_iod 的 SSL 会话，增加使用计数。会话不应该已经被设置（因为没有释放）。
#if HAVE_OPENSSL
void nsi_set_ssl_session(struct niod *iod, SSL_SESSION *sessid) {
  // 如果支持 OpenSSL
  if (sessid) {
    // 如果 sessid 存在
    iod->ssl_session = sessid;
    // 将 iod->ssl_session 设置为 sessid
    /* No reference counting for the copy stored briefly in nsiod */
    // 在 nsiod 中存储的副本不进行引用计数
  }
}
#endif

/* Sometimes it is useful to store a pointer to information inside the struct niod so
 * you can retrieve it during a callback. */
// 有时在 struct niod 中存储信息的指针是有用的，这样在回调期间可以检索它。
void nsock_iod_set_udata(nsock_iod iod, void *udata) {
  // 断言 iod 不为空
  assert(iod);
  // 将 iod->userdata 设置为 udata
  ((struct niod *)iod)->userdata = udata;
}

/* And the function above wouldn't make much sense if we didn't have a way to
 * retrieve that data... */
// 如果没有一种方法来检索数据，上面的函数就没有太多意义...
// 获取指定 nsock_iod 对象的用户数据
void *nsock_iod_get_udata(nsock_iod iod) {
  // 断言确保 nsock_iod 对象存在
  assert(iod);
  // 返回 nsock_iod 对象中的 userdata
  return ((struct niod *)iod)->userdata;
}

/* 如果一个 NSI 通过 SSL 进行通信，则返回 1，否则返回 0 */
int nsock_iod_check_ssl(nsock_iod iod) {
  // 如果 nsock_iod 对象中的 ssl 存在，则返回 1，否则返回 0
  return (((struct niod *)iod)->ssl) ? 1 : 0;
}

/* 返回远程对等端口（如果不可用则返回-1）。注意返回值是一个整数，这样-1可以与65535区分开。端口以主机字节顺序返回。 */
int nsock_iod_get_peerport(nsock_iod iod) {
  // 将 nsock_iod 对象转换为 struct niod 类型
  struct niod *nsi = (struct niod *)iod;
  int fam;

  // 如果 peerlen 小于等于0，则返回-1
  if (nsi->peerlen <= 0)
    return -1;

  // 获取对等端口的地址族
  fam = ((struct sockaddr_in *)&nsi->peer)->sin_family;

  // 如果地址族为 AF_INET，则返回对等端口的网络字节顺序
  if (fam == AF_INET)
    return ntohs(((struct sockaddr_in *)&nsi->peer)->sin_port);
#if HAVE_IPV6
  // 如果地址族为 AF_INET6，则返回对等端口的网络字节顺序
  else if (fam == AF_INET6)
    return ntohs(((struct sockaddr_in6 *)&nsi->peer)->sin6_port);
#endif

  // 如果地址族不是 AF_INET 或 AF_INET6，则返回-1
  return -1;
}

/* 在连接之前设置要绑定的本地地址 */
int nsock_iod_set_localaddr(nsock_iod iod, struct sockaddr_storage *ss,
                            size_t sslen) {
  // 将 nsock_iod 对象转换为 struct niod 类型
  struct niod *nsi = (struct niod *)iod;

  // 断言确保 nsi 存在
  assert(nsi);

  // 如果 sslen 大于 ns1->local 的大小，则返回-1
  if (sslen > sizeof(nsi->local))
    return -1;

  // 将 ss 复制到 ns1->local，并设置 locallen 为 sslen
  memcpy(&nsi->local, ss, sslen);
  nsi->locallen = sslen;
  return 0;
}

/* 设置在连接之前应用的 IPv4 选项。它会复制选项，因此如果需要，可以释放你自己的。此副本在销毁 iod 时被释放。 */
int nsock_iod_set_ipoptions(nsock_iod iod, void *opts, size_t optslen) {
  // 将 nsock_iod 对象转换为 struct niod 类型
  struct niod *nsi = (struct niod *)iod;

  // 断言确保 nsi 存在
  assert(nsi);

  // 如果 optslen 大于44，则返回-1
  if (optslen > 44)
    return -1;

  // 分配 optslen 大小的内存给 nsi->ipopts，并将 opts 复制到 nsi->ipopts，设置 ipoptslen 为 optslen
  nsi->ipopts = safe_malloc(optslen);
  memcpy(nsi->ipopts, opts, optslen);
  nsi->ipoptslen = optslen;
  return 0;
}
/* 获取 nsock_iod 中的 socket 描述符，用于执行一些合理的操作，比如设置 socket 接收缓冲区大小 */
int nsock_iod_get_sd(nsock_iod iod) {
  // 将 nsock_iod 转换为 niod 结构体
  struct niod *nsi = (struct niod *)iod;

  // 断言 nsi 不为空
  assert(nsi);

  // 如果有 pcap，返回 pcap 描述符，否则返回 nsi 中的描述符
#if HAVE_PCAP
  if (nsi->pcap)
    return ((mspcap *)nsi->pcap)->pcap_desc;
  else
#endif
    return nsi->sd;
}

/* 获取 nsock_iod 中的读取次数 */
unsigned long nsock_iod_get_read_count(nsock_iod iod){
  // 断言 iod 不为空
  assert(iod);
  return ((struct niod *)iod)->read_count;
}

/* 获取 nsock_iod 中的写入次数 */
unsigned long nsock_iod_get_write_count(nsock_iod iod){
  // 断言 iod 不为空
  assert(iod);
  return ((struct niod *)iod)->write_count;
}

/* 设置 nsock_iod 中的主机名 */
int nsock_iod_set_hostname(nsock_iod iod, const char *hostname) {
  // 将 nsock_iod 转换为 niod 结构体
  struct niod *nsi = (struct niod *)iod;

  // 如果 nsi 中的主机名不为空，释放内存
  if (nsi->hostname != NULL)
    free(nsi->hostname);

  // 复制主机名到 nsi 中的主机名
  nsi->hostname = strdup(hostname);
  // 如果复制失败，返回 -1
  if (nsi->hostname == NULL)
    return -1;

  // 设置成功，返回 0
  return 0;
}
```