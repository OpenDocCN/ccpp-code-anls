# `nmap\nsock\src\engine_iocp.c`

```
/* $Id$ */
# 定义代码版本信息

#if WIN32
#include "nsock_winconfig.h"
#endif
# 如果是 Windows 系统，包含 Windows 相关配置文件

#if HAVE_IOCP
# 如果支持 IOCP

#include <Winsock2.h>
#include <Mswsock.h>
# 包含 Windows Socket API 和 Microsoft Windows Sockets 2.0 Service Provider

#include "nsock_internal.h"
#include "nsock_log.h"
# 包含内部定义和日志相关的头文件

#if HAVE_PCAP
#include "nsock_pcap.h"
#endif
# 如果支持 PCAP，包含 PCAP 相关头文件

/* --- ENGINE INTERFACE PROTOTYPES --- */
# 引擎接口原型定义
static int iocp_init(struct npool *nsp);
static void iocp_destroy(struct npool *nsp);
static int iocp_iod_register(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev);
static int iocp_iod_unregister(struct npool *nsp, struct niod *iod);
static int iocp_iod_modify(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev_set, int ev_clr);
static int iocp_loop(struct npool *nsp, int msec_timeout);
# 定义 IOCP 引擎的接口函数原型

int iocp_iod_connect(struct npool *nsp, int sockfd, const struct sockaddr *addr, socklen_t addrlen);
int iocp_iod_read(struct npool *nsp, int sockfd, void *buf, size_t len, int flags, struct sockaddr *src_addr, socklen_t *addrlen);
int iocp_iod_write(struct npool *nsp, int sockfd, const void *buf, size_t len, int flags, const struct sockaddr *dest_addr, socklen_t addrlen);
# 定义 IOCP 引擎的连接、读取、写入函数

struct io_operations iocp_io_operations = {
  iocp_iod_connect,
  iocp_iod_read,
  iocp_iod_write
};
# 定义 IOCP 引擎的 IO 操作结构体

/* ---- ENGINE DEFINITION ---- */
# 引擎定义部分
struct io_engine engine_iocp = {
  "iocp",
  iocp_init,
  iocp_destroy,
  iocp_iod_register,
  iocp_iod_unregister,
  iocp_iod_modify,
  iocp_loop,
  &iocp_io_operations
};
# 定义 IOCP 引擎结构体

/*
* Engine specific data structure
*/
# 引擎特定数据结构
struct iocp_engine_info {
  /* The handle to the Completion Port*/
  HANDLE iocp;
  # 完成端口的句柄

  /* We put the current eov to be processed here in order to be retrieved by nsock_core */
  struct extended_overlapped *eov;
  # 将要处理的当前 eov 放在这里，以便被 nsock_core 检索

  /* The overlapped_entry list used to retrieve completed packets from the port */
  OVERLAPPED_ENTRY *eov_list;
  unsigned long capacity;
  # 用于从端口检索完成的数据包的 overlapped_entry 列表

  /* How many Completion Packets we actually retreieved */
  unsigned long entries_removed;
  # 实际检索到的完成包数量

  gh_list_t active_eovs;
  gh_list_t free_eovs;
};
# 定义 IOCP 引擎特定数据结构
# 定义了一个扩展的重叠结构体，用于重叠操作
struct extended_overlapped {
  /* 用于重叠操作的重叠结构 */
  OVERLAPPED ov;

  /* 在启动操作时是否出现错误？将错误代码放在这里并将其发送到主循环 */
  int err;

  /* 事件可能已经过期并被回收，我们不能信任指向nevent结构的指针来告诉我们真正的nevent */
  nsock_event_id nse_id;

  /* 事件的指针 */
  struct nevent *nse;

  /* 用于WSARecv/WSASend */
  WSABUF wsabuf;

  /* 这是我们将在其中读取数据的缓冲区 */
  char *readbuf;

  /* 结构npool跟踪已分配的EOV，以便在删除msp时销毁它们。这个指针使得在必要时很容易从分配的列表中删除这个扩展的重叠结构 */
  gh_lnode_t nodeq;
};

/* --- 内部原型 --- */
static void iterate_through_event_lists(struct npool *nsp);
static void iterate_through_pcap_events(struct npool *nsp);
static void terminate_overlapped_event(struct npool *nsp, struct nevent *nse);
static void initiate_overlapped_event(struct npool *nsp, struct nevent *nse);
static int get_overlapped_result(struct npool *nsp, int fd, const void *buffer, size_t count);
static void force_operation(struct npool *nsp, struct nevent *nse);
static void free_eov(struct npool *nsp, struct extended_overlapped *eov);
static int map_faulty_errors(int err);

/* 在nsock_core.c中定义 */
void process_iod_events(struct npool *nsp, struct niod *nsi, int ev);
void process_event(struct npool *nsp, gh_list_t *evlist, struct nevent *nse, int ev);
void process_expired_events(struct npool *nsp);
#if HAVE_PCAP
#ifndef PCAP_CAN_DO_SELECT
int pcap_read_on_nonselect(struct npool *nsp);
#endif
#endif

/* 在nsock_event.c中定义 */
void update_first_events(struct nevent *nse);


extern struct timeval nsock_tod;
# 初始化 IOCP 引擎，将其信息存储在给定的 npool 结构中
int iocp_init(struct npool *nsp) {
  # 分配内存以存储 IOCP 引擎信息
  struct iocp_engine_info *iinfo;
  iinfo = (struct iocp_engine_info *)safe_malloc(sizeof(struct iocp_engine_info));

  # 初始化活跃的扩展重叠结构链表
  gh_list_init(&iinfo->active_eovs);
  # 初始化空闲的扩展重叠结构链表
  gh_list_init(&iinfo->free_eovs);

  # 创建一个 I/O 完成端口对象
  iinfo->iocp = CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, NULL, 0);
  # 设置容量为 10
  iinfo->capacity = 10;
  iinfo->eov = NULL;
  iinfo->entries_removed = 0;
  # 分配内存以存储 OVERLAPPED_ENTRY 结构数组
  iinfo->eov_list = (OVERLAPPED_ENTRY *)safe_malloc(iinfo->capacity * sizeof(OVERLAPPED_ENTRY));
  # 将 IOCP 引擎信息存储在给定的 npool 结构中
  nsp->engine_data = (void *)iinfo;

  # 返回 1 表示初始化成功
  return 1;
}

# 销毁 IOCP 引擎
void iocp_destroy(struct npool *nsp) {
  # 获取存储在 npool 结构中的 IOCP 引擎信息
  struct iocp_engine_info *iinfo = (struct iocp_engine_info *)nsp->engine_data;

  # 断言 IOCP 引擎信息不为空
  assert(iinfo != NULL);

  struct extended_overlapped *eov;
  gh_lnode_t *current;

  # 释放活跃的扩展重叠结构链表中的资源
  while ((current = gh_list_pop(&iinfo->active_eovs))) {
    eov = container_of(current, struct extended_overlapped, nodeq);
    if (eov->readbuf) {
      free(eov->readbuf);
      eov->readbuf = NULL;
    }
    free(eov);
  }

  # 释放空闲的扩展重叠结构链表中的资源
  while ((current = gh_list_pop(&iinfo->free_eovs))) {
    eov = container_of(current, struct extended_overlapped, nodeq);
    free(eov);
  }

  # 释放活跃的扩展重叠结构链表
  gh_list_free(&iinfo->active_eovs);
  # 释放空闲的扩展重叠结构链表
  gh_list_free(&iinfo->free_eovs);

  # 关闭 I/O 完成端口对象
  CloseHandle(iinfo->iocp);
  # 释放 OVERLAPPED_ENTRY 结构数组的内存
  free(iinfo->eov_list);

  # 释放 IOCP 引擎信息的内存
  free(iinfo);
}

# 注册 I/O 完成端口对象
int iocp_iod_register(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev) {
  # 获取存储在 npool 结构中的 IOCP 引擎信息
  struct iocp_engine_info *iinfo = (struct iocp_engine_info *)nsp->engine_data;
  HANDLE result;

  # 断言 I/O 完成端口对象未注册
  assert(!IOD_PROPGET(iod, IOD_REGISTERED));
  # 设置被监视的事件
  iod->watched_events = ev;
  # 将 I/O 完成端口对象与 I/O 完成端口关联
  result = CreateIoCompletionPort((HANDLE)iod->sd, iinfo->iocp, NULL, 0);
  # 断言关联成功
  assert(result);

  # 设置 I/O 完成端口对象已注册
  IOD_PROPSET(iod, IOD_REGISTERED);

  # 初始化重叠事件
  initiate_overlapped_event(nsp, nse);

  # 返回 1 表示注册成功
  return 1;
}

# 取消 I/O 完成端口对象的注册
int iocp_iod_unregister(struct npool *nsp, struct niod *iod) {

  if (IOD_PROPGET(iod, IOD_REGISTERED)) {
    # 取消 I/O 完成端口对象上的所有未完成的操作
    CancelIo((HANDLE)iod->sd);
    # 设置IOD对象的属性为IOD_REGISTERED
    IOD_PROPCLR(iod, IOD_REGISTERED);
  }
  # 返回1，表示成功
  return 1;
}

int iocp_iod_modify(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev_set, int ev_clr) {
  int new_events;
  struct iocp_engine_info *iinfo = (struct iocp_engine_info *)nsp->engine_data;

  assert((ev_set & ev_clr) == 0);  // 确保设置和清除的事件不重叠
  assert(IOD_PROPGET(iod, IOD_REGISTERED));  // 确保 niod 已注册

  new_events = iod->watched_events;  // 获取当前 niod 监视的事件
  new_events |= ev_set;  // 添加新的需要监视的事件
  new_events &= ~ev_clr;  // 移除不需要监视的事件

  if (ev_set != EV_NONE)
    initiate_overlapped_event(nsp, nse);  // 如果有设置事件，则发起重叠事件
  else if (ev_clr != EV_NONE)
    terminate_overlapped_event(nsp, nse);  // 如果有清除事件，则终止重叠事件

  if (new_events == iod->watched_events)
    return 1; /* nothing to do */  // 如果新的事件和之前监视的事件相同，则无需操作

  iod->watched_events = new_events;  // 更新 niod 监视的事件

  return 1;  // 返回操作成功
}

int iocp_loop(struct npool *nsp, int msec_timeout) {
  int event_msecs; /* msecs before an event goes off */  // 事件超时的毫秒数
  int combined_msecs;
  int sock_err = 0;
  BOOL bRet;
  unsigned long total_events;
  struct iocp_engine_info *iinfo = (struct iocp_engine_info *)nsp->engine_data;

  assert(msec_timeout >= -1);  // 确保超时时间大于等于-1

  if (nsp->events_pending == 0)
    return 0; /* No need to wait on 0 events ... */  // 如果没有待处理的事件，则无需等待

  struct nevent *nse;

  /* Make sure the preallocated space for the retrieved events is big enough */
  total_events = gh_list_count(&nsp->connect_events) + gh_list_count(&nsp->read_events) + gh_list_count(&nsp->write_events);  // 计算所有事件的总数
  if (iinfo->capacity < total_events) {
    iinfo->capacity *= 2;  // 扩大容量
    iinfo->eov_list = (OVERLAPPED_ENTRY *)safe_realloc(iinfo->eov_list, iinfo->capacity * sizeof(OVERLAPPED_ENTRY));  // 重新分配内存
  }

  nsock_log_debug_all("wait for events");  // 记录调试信息

  nse = next_expirable_event(nsp);  // 获取下一个将要超时的事件
  if (!nse)
    event_msecs = -1; /* None of the events specified a timeout */  // 如果没有事件指定超时，则设置为-1
  else
    event_msecs = MAX(0, TIMEVAL_MSEC_SUBTRACT(nse->timeout, nsock_tod));  // 计算距离下一个事件超时的时间

#if HAVE_PCAP
#ifndef PCAP_CAN_DO_SELECT
  /* Force a low timeout when capturing packets on systems where
  * the pcap descriptor is not select()able. */
  if (gh_list_count(&nsp->pcap_read_events) > 0)
  if (event_msecs > PCAP_POLL_INTERVAL)
    event_msecs = PCAP_POLL_INTERVAL;  // 在不支持 select() 的系统上捕获数据包时，强制设置低超时时间
#endif
#endif

  /* 将事件超时时间和毫秒超时时间中较小的值赋给combined_msecs，使用无符号类型是为了让-1表示非常大的值（因为它表示没有超时） */
  combined_msecs = MIN((unsigned)event_msecs, (unsigned)msec_timeout);

#if HAVE_PCAP
#ifndef PCAP_CAN_DO_SELECT
  /* 对不支持select()的pcap设备进行非阻塞读取
  * 如果有任何内容被读取，就离开这个循环 */
  if (pcap_read_on_nonselect(nsp)) {
    /* 好的，有内容被读取 */
    gettimeofday(&nsock_tod, NULL);
    iterate_through_pcap_events(nsp);
  }
  else
#endif
#endif
  {
    /* 在调用GetQueuedCompletionStatusEx之前，重置这些值是必需的 */
    iinfo->entries_removed = 0;
    memset(iinfo->eov_list, 0, iinfo->capacity * sizeof(OVERLAPPED_ENTRY));
    bRet = GetQueuedCompletionStatusEx(iinfo->iocp, iinfo->eov_list, iinfo->capacity, &iinfo->entries_removed, combined_msecs, FALSE);

    gettimeofday(&nsock_tod, NULL); /* 由于iocp延迟 */
    if (!bRet) {
      sock_err = socket_errno();
      if (!iinfo->eov && sock_err != WAIT_TIMEOUT) {
        nsock_log_error("nsock_loop error %d: %s", sock_err, socket_strerror(sock_err));
        nsp->errnum = sock_err;
        return -1;
      }
    }
  }

  iterate_through_event_lists(nsp);

  return 1;
}


/* ---- INTERNAL FUNCTIONS ---- */

#if HAVE_PCAP
/* 单独遍历pcap事件，因为这些事件不在iocp_engine_info中被跟踪 */
void iterate_through_pcap_events(struct npool *nsp) {
  gh_lnode_t *current, *next, *last;

  last = gh_list_last_elem(&nsp->active_iods);

  for (current = gh_list_first_elem(&nsp->active_iods);
       current != NULL && gh_lnode_prev(current) != last;
       current = next) {
    struct niod *nsi = container_of(current, struct niod, nodeq);

    if (nsi->pcap && nsi->state != NSIOD_STATE_DELETED && nsi->events_pending)
    {
      process_iod_events(nsp, nsi, EV_READ);
    }

    next = gh_lnode_next(current);
    # 如果当前 IOD 状态为已删除
    if (nsi->state == NSIOD_STATE_DELETED) {
      # 从活跃 IOD 列表中移除当前 IOD
      gh_list_remove(&nsp->active_iods, current);
      # 将当前 IOD 放入空闲 IOD 列表的开头
      gh_list_prepend(&nsp->free_iods, current);
    }
  }
}
#endif

/* 遍历所有事件列表（如 connect_events、read_events、timer_events 等），对已完成的事件（由于超时、I/O 等）进行操作 */
void iterate_through_event_lists(struct npool *nsp) {
  // 获取 IOCP 引擎信息
  struct iocp_engine_info *iinfo = (struct iocp_engine_info *)nsp->engine_data;

  for (unsigned long i = 0; i < iinfo->entries_removed; i++) {

    // 获取当前处理的 extended_overlapped 结构体
    iinfo->eov = (struct extended_overlapped *)iinfo->eov_list[i].lpOverlapped;
    /* 不能依赖 iinfo->entries_removed 告诉我们要处理的事件的真实数量 */
    if (!iinfo->eov || !iinfo->eov->nse)
      continue;

    /* 检查是否来自已取消的操作 */
    if (iinfo->eov->nse->id != iinfo->eov->nse_id ||
        iinfo->eov->nse->event_done) {
      free_eov(nsp, iinfo->eov);
      iinfo->eov = NULL;
      continue;
    }

    // 检查是否完成了重叠 I/O 操作
    if (!HasOverlappedIoCompleted((OVERLAPPED *)iinfo->eov))
      continue;

    // 获取相关的 niod 和 nevent 结构体
    struct niod *nsi = iinfo->eov->nse->iod;
    struct nevent *nse = iinfo->eov->nse;
    gh_list_t *evlist = NULL;
    int ev = 0;

    switch (nse->type) {
      case NSE_TYPE_CONNECT:
      case NSE_TYPE_CONNECT_SSL:
        ev = EV_READ;
        evlist = &nsp->connect_events;
        break;
      case NSE_TYPE_READ:
        ev = EV_READ;
        evlist = &nsp->read_events;
        break;
      case NSE_TYPE_WRITE:
        ev = EV_WRITE;
        evlist = &nsp->write_events;
        break;
    }

    /* 设置连接错误，以便 nsock_core 在 handle_connect_result 中处理 */
    if (nse->type == NSE_TYPE_CONNECT || nse->type == NSE_TYPE_CONNECT_SSL) {
      setsockopt(nse->iod->sd, SOL_SOCKET, SO_UPDATE_CONNECT_CONTEXT, NULL, 0);
      DWORD dwRes;
      if (!GetOverlappedResult((HANDLE)nse->iod->sd, (LPOVERLAPPED)iinfo->eov, &dwRes, FALSE)) {
        int err = map_faulty_errors(socket_errno());
        if (err)
          setsockopt(nse->iod->sd, SOL_SOCKET, SO_ERROR, (char *)&err, sizeof(err));
      }
    }
    # 处理事件，传入命名空间、事件列表、I/O 状态和事件
    process_event(nsp, evlist, nse, ev);

    # 如果事件已完成
    if (nse->event_done) {
      # 事件已完成，从事件列表中移除，并更新每种事件类型的首个事件的 I/O 指针
      update_first_events(nse);
      gh_list_remove(evlist, &nse->nodeq_io);
      gh_list_append(&nsp->free_events, &nse->nodeq_io);

      # 如果事件有超时时间，则从超时堆中移除
      if (nse->timeout.tv_sec)
        gh_heap_remove(&nsp->expirables, &nse->expire);
    } else
      # 否则，初始化重叠事件
      initiate_overlapped_event(nsp, nse);

    # 如果 I/O 状态为已删除
    if (nsi->state == NSIOD_STATE_DELETED) {
      # 从活动 I/O 状态列表中移除，并将其添加到空闲 I/O 状态列表中
      gh_list_remove(&nsp->active_iods, &nsi->nodeq);
      gh_list_prepend(&nsp->free_iods, &nsi->nodeq);
    }

    # 重置事件溢出值
    iinfo->eov = NULL;
  }

  # 迭代计时器和已过期事件
  process_expired_events(nsp);
static int errcode_is_failure(int err) {
#ifndef WIN32
  // 检查错误码是否表示失败，排除 EINTR、EAGAIN 和 EBUSY
  return err != EINTR && err != EAGAIN && err != EBUSY;
#else
  // 检查错误码是否表示失败，排除 EINTR、EAGAIN、WSA_IO_PENDING 和 ERROR_NETNAME_DELETED
  return err != EINTR && err != EAGAIN && err != WSA_IO_PENDING && err != ERROR_NETNAME_DELETED;
#endif
}

static int map_faulty_errors(int err) {
  // 将一些错误码映射为对应的 Windows 错误码
  switch (err) {
    case ERROR_NETWORK_UNREACHABLE: return WSAENETUNREACH;
    case ERROR_HOST_UNREACHABLE: return WSAEHOSTUNREACH;
    case ERROR_CONNECTION_REFUSED: return WSAECONNREFUSED;
    case ERROR_SEM_TIMEOUT: return WSAETIMEDOUT;
  }
  return err;
}

static struct extended_overlapped *new_eov(struct npool *nsp, struct nevent *nse) {
  struct extended_overlapped *eov;
  gh_lnode_t *lnode;
  struct iocp_engine_info *iinfo = (struct iocp_engine_info *)nsp->engine_data;

  // 从空闲的 extended_overlapped 链表中取出一个节点
  lnode = gh_list_pop(&iinfo->free_eovs);
  if (!lnode)
    // 如果链表为空，则分配新的 extended_overlapped 结构
    eov = (struct extended_overlapped *)safe_malloc(sizeof(struct extended_overlapped));
  else
    // 如果链表不为空，则使用链表中的节点
    eov = container_of(lnode, struct extended_overlapped, nodeq);

  // 将 extended_overlapped 结构清零
  memset(eov, 0, sizeof(struct extended_overlapped));
  nse->eov = eov;
  eov->nse = nse;
  eov->nse_id = nse->id;
  eov->err = 0;
  // 将 extended_overlapped 结构加入到活动的 extended_overlapped 链表中
  gh_list_prepend(&iinfo->active_eovs, &eov->nodeq);

  // 如果是读操作，并且 readbuf 为空，并且不是 SSL 连接，则分配读缓冲区
  if (nse->type == NSE_TYPE_READ && !eov->readbuf && !nse->iod->ssl)
    eov->readbuf = (char*)safe_malloc(READ_BUFFER_SZ * sizeof(char));

  return eov;
}

/* This needs to be called after getting the overlapped event in */
static void free_eov(struct npool *nsp, struct extended_overlapped *eov) {
  struct iocp_engine_info *iinfo = (struct iocp_engine_info *)nsp->engine_data;
  struct nevent *nse = eov->nse;

  // 从活动的 extended_overlapped 链表中移除节点
  gh_list_remove(&iinfo->active_eovs, &eov->nodeq);

  // 如果 readbuf 不为空，则释放内存
  if (eov->readbuf) {
    free(eov->readbuf);
    eov->readbuf = NULL;
  }

  // 将节点加入到空闲的 extended_overlapped 链表中
  gh_list_prepend(&iinfo->free_eovs, &eov->nodeq);

  eov->nse = NULL;
  if (nse)
    nse->eov = NULL;
}
# 调用连接重叠函数，传入网络池和网络事件
static void call_connect_overlapped(struct npool *nsp, struct nevent *nse) {
  BOOL ok;  # 布尔类型变量，用于存储操作是否成功的状态
  DWORD numBytes = 0;  # 用于存储字节数的变量
  int one = 1;  # 用于存储整数1的变量
  SOCKET sock = nse->iod->sd;  # 获取套接字描述符
  GUID guid = WSAID_CONNECTEX;  # 创建GUID对象
  struct sockaddr_in addr;  # 创建IPv4套接字地址结构
  LPFN_CONNECTEX ConnectExPtr = NULL;  # 创建指向ConnectEx函数的指针
  struct iocp_engine_info *iinfo = (struct iocp_engine_info *)nse->iod->nsp->engine_data;  # 获取IOCP引擎信息
  struct extended_overlapped *eov = new_eov(nsp, nse);  # 创建扩展重叠结构体对象
  int ret;  # 用于存储函数返回值的变量
  struct sockaddr_storage *ss = &nse->iod->peer;  # 获取对等方套接字地址存储结构的指针
  size_t sslen = nse->iod->peerlen;  # 获取对等方套接字地址存储结构的长度

  if (nse->iod->lastproto != IPPROTO_TCP) {  # 如果上一个协议不是TCP
    if (connect(sock, (struct sockaddr *)ss, sslen) == -1) {  # 进行连接
      int err = socket_errno();  # 获取套接字错误码
      nse->event_done = 1;  # 设置事件完成标志
      nse->status = NSE_STATUS_ERROR;  # 设置事件状态为错误
      nse->errnum = err;  # 设置错误码
    } else {
      force_operation(nsp, nse);  # 强制执行操作
    }
    return;  # 返回
  }

  ret = WSAIoctl(sock, SIO_GET_EXTENSION_FUNCTION_POINTER,
    (void*)&guid, sizeof(guid), (void*)&ConnectExPtr, sizeof(ConnectExPtr),
    &numBytes, NULL, NULL);  # 调用WSAIoctl函数获取ConnectEx函数指针
  if (ret)  # 如果返回值不为0
    fatal("Error initiating event type(%d)", nse->type);  # 输出错误信息

  ret = setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, (const char *)&one, sizeof(one));  # 设置套接字选项
  if (ret == -1) {  # 如果返回值为-1
    int err = socket_errno();  # 获取套接字错误码
    nse->event_done = 1;  # 设置事件完成标志
    nse->status = NSE_STATUS_ERROR;  # 设置事件状态为错误
    nse->errnum = err;  # 设置错误码
    return;  # 返回
  }

  /* ConnectEx doesn't automatically bind the socket */
  memset(&addr, 0, sizeof(addr));  # 将addr结构体清零
  addr.sin_family = AF_INET;  # 设置地址族为IPv4
  addr.sin_addr.s_addr = INADDR_ANY;  # 设置IP地址为任意地址
  addr.sin_port = 0;  # 设置端口为0
  if (!nse->iod->locallen) {  # 如果本地地址长度为0
    ret = bind(sock, (SOCKADDR*)&addr, sizeof(addr));  # 绑定套接字
    if (ret) {  # 如果返回值不为0
      int err = socket_errno();  # 获取套接字错误码
      nse->event_done = 1;  # 设置事件完成标志
      nse->status = NSE_STATUS_ERROR;  # 设置事件状态为错误
      nse->errnum = err;  # 设置错误码
      return;  # 返回
    }
  }

  ok = ConnectExPtr(sock, (SOCKADDR*)ss, sslen, NULL, 0, NULL, (LPOVERLAPPED)eov);  # 调用ConnectEx函数
  if (!ok) {  # 如果返回值为假
    int err = socket_errno();  # 获取套接字错误码
    if (err != ERROR_IO_PENDING) {  # 如果错误码不是ERROR_IO_PENDING
      nse->event_done = 1;  # 设置事件完成标志
      nse->status = NSE_STATUS_ERROR;  # 设置事件状态为错误
      nse->errnum = err;  # 设置错误码
    }
  }
}
# 调用异步读取函数，处理传入的事件结构体
static void call_read_overlapped(struct nevent *nse) {
  # 初始化标志位和错误码
  DWORD flags = 0;
  int err = 0;
  # 获取IOCP引擎信息
  struct iocp_engine_info *iinfo = (struct iocp_engine_info *)nse->iod->nsp->engine_data;

  # 创建扩展的overlapped结构体
  struct extended_overlapped *eov = new_eov(nse->iod->nsp, nse);

  # 设置wsabuf的buf和len属性
  eov->wsabuf.buf = eov->readbuf;
  eov->wsabuf.len = READ_BUFFER_SZ;

  # 调用WSARecvFrom函数进行异步读取
  err = WSARecvFrom(nse->iod->sd, &eov->wsabuf, 1, NULL, &flags,
    (struct sockaddr *)&nse->iod->peer, (LPINT)&nse->iod->peerlen, (LPOVERLAPPED)eov, NULL);
  # 处理错误情况
  if (err) {
    err = socket_errno();
    if (errcode_is_failure(err)) {
      # 如果错误码是ERROR_PORT_UNREACHABLE，则将eov的err属性设置为ECONNREFUSED，否则设置为err
      eov->err = (err == ERROR_PORT_UNREACHABLE ? ECONNREFUSED : err);
      /* Send the error to the main loop to be picked up by the appropriate handler */
      # 将错误信息发送到主循环，以便由适当的处理程序接收
      BOOL bRet = PostQueuedCompletionStatus(iinfo->iocp, -1, (ULONG_PTR)nse->iod, (LPOVERLAPPED)eov);
      # 如果发送失败，则触发致命错误
      if (!bRet)
        fatal("Error initiating event type(%d)", nse->type);
    }
  }
}

# 调用异步写入函数，处理传入的事件结构体
static void call_write_overlapped(struct nevent *nse) {
  int err;
  char *str;
  int bytesleft;
  # 获取IOCP引擎信息
  struct iocp_engine_info *iinfo = (struct iocp_engine_info *)nse->iod->nsp->engine_data;

  # 创建扩展的overlapped结构体
  struct extended_overlapped *eov = new_eov(nse->iod->nsp, nse);

  # 获取待写入数据的起始地址和剩余字节数
  str = fs_str(&nse->iobuf) + nse->writeinfo.written_so_far;
  bytesleft = fs_length(&nse->iobuf) - nse->writeinfo.written_so_far;

  # 设置wsabuf的buf和len属性
  eov->wsabuf.buf = str;
  eov->wsabuf.len = bytesleft;

  # 根据目标地址的协议簇选择调用WSASend或WSASendTo函数进行异步写入
  if (nse->writeinfo.dest.ss_family == AF_UNSPEC)
    err = WSASend(nse->iod->sd, &eov->wsabuf, 1, NULL, 0, (LPWSAOVERLAPPED)eov, NULL);
  else
    err = WSASendTo(nse->iod->sd, &eov->wsabuf, 1, NULL, 0,
    (struct sockaddr *)&nse->writeinfo.dest, (int)nse->writeinfo.destlen,
    (LPWSAOVERLAPPED)eov, NULL);
  # 处理错误情况
  if (err) {
    err = socket_errno();
    # 如果错误码表示失败
    if (errcode_is_failure(err)) {
      # 将错误码存储到事件对象中
      eov->err = err;
      # 将错误发送到主循环，由适当的处理程序接收
      BOOL bRet = PostQueuedCompletionStatus(iinfo->iocp, -1, (ULONG_PTR)nse->iod, (LPOVERLAPPED)eov);
      # 如果发送失败，则触发致命错误并打印错误信息
      if (!bRet)
        fatal("Error initiating event type(%d)", nse->type);
    }
  }
/* Anything that isn't an overlapped operation uses this to get processed by the main loop */
static void force_operation(struct npool *nsp, struct nevent *nse) {
  BOOL bRet;
  struct extended_overlapped *eov;

  struct iocp_engine_info *iinfo = (struct iocp_engine_info *)nsp->engine_data;
  // 创建一个新的扩展重叠结构体
  eov = new_eov(nse->iod->nsp, nse);

  // 将完成状态信息放入完成端口队列
  bRet = PostQueuedCompletionStatus(iinfo->iocp, 0, (ULONG_PTR)nse->iod, (LPOVERLAPPED)eov);
  if (!bRet)
    // 如果失败，输出错误信息
    fatal("Error initiating event type(%d)", nse->type);
}

/* Either initiate a I/O read or force a SSL_read */
static void initiate_read(struct npool *nsp, struct nevent *nse) {
  if (!nse->iod->ssl)
    // 如果不是 SSL 连接，调用读取重叠操作
    call_read_overlapped(nse);
  else
    // 否则，强制执行操作
    force_operation(nsp, nse);
}

/* Either initiate a I/O write or force a SSL_write */
static void initiate_write(struct npool *nsp, struct nevent *nse) {
  if (!nse->iod->ssl)
    // 如果不是 SSL 连接，调用写入重叠操作
    call_write_overlapped(nse);
  else
    // 否则，强制执行操作
    force_operation(nsp, nse);
}

/* Force a PCAP read */
static void initiate_pcap_read(struct npool *nsp, struct nevent *nse) {
  // 强制执行操作
  force_operation(nsp, nse);
}

static void initiate_connect(struct npool *nsp, struct nevent *nse) {
  int sslconnect_inprogress = 0;
  struct iocp_engine_info *iinfo = (struct iocp_engine_info *)nsp->engine_data;

#if HAVE_OPENSSL
  sslconnect_inprogress = nse->type == NSE_TYPE_CONNECT_SSL && nse->iod &&
    (nse->sslinfo.ssl_desire == SSL_ERROR_WANT_READ ||
    nse->sslinfo.ssl_desire == SSL_ERROR_WANT_WRITE);
#endif

  if (sslconnect_inprogress)
    // 如果 SSL 连接正在进行中，强制执行操作
    force_operation(nsp, nse);
  else
    // 否则，调用连接重叠操作
    call_connect_overlapped(nsp, nse);
}

/* Start the overlapped I/O operation */
static void initiate_overlapped_event(struct npool *nsp, struct nevent *nse) {
  if (nse->eov)
    // 如果已经存在重叠事件，终止它
    terminate_overlapped_event(nsp, nse);

  switch (nse->type) {
  case NSE_TYPE_CONNECT:
  case NSE_TYPE_CONNECT_SSL:
    // 根据事件类型进行相应的操作
    initiate_connect(nsp, nse);
    break;
  case NSE_TYPE_READ:
    initiate_read(nsp, nse);
    break;
  case NSE_TYPE_WRITE:
    initiate_write(nsp, nse);
    break;
#if HAVE_PCAP
  case NSE_TYPE_PCAP_READ:
    initiate_pcap_read(nsp, nse);
    break;
#endif
  default: fatal("Event type(%d) not supported by engine IOCP\n", nse->type);
  }
}

/* Terminate an overlapped I/O operation that expired */
static void terminate_overlapped_event(struct npool *nsp, struct nevent *nse) {
  bool eov_done = true;

  if (nse->eov) {
    if (!HasOverlappedIoCompleted((LPOVERLAPPED)nse->eov)) {
      CancelIoEx((HANDLE)nse->iod->sd, (LPOVERLAPPED)nse->eov);
      eov_done = false;
    }

    if (eov_done)
      free_eov(nsp, nse->eov);
  }
}

/* Retrieve the amount of bytes transferred or set the appropriate error */
static int get_overlapped_result(struct npool *nsp, int fd, const void *buffer, size_t count) {
  char *buf = (char *)buffer;
  DWORD dwRes = 0;
  int err;
  struct iocp_engine_info *iinfo = (struct iocp_engine_info *)nsp->engine_data;

  struct extended_overlapped *eov = iinfo->eov;
  struct nevent *nse = eov->nse;

  /* If the operation failed at initialization, set the error for nsock_core.c to see */
  if (eov->err) {
    SetLastError(map_faulty_errors(eov->err));
    return -1;
  }

  if (!GetOverlappedResult((HANDLE)fd, (LPOVERLAPPED)eov, &dwRes, FALSE)) {
    err = socket_errno();
    if (errcode_is_failure(err)) {
      SetLastError(map_faulty_errors(err));
      return -1;
    }
  }

  if (nse->type == NSE_TYPE_READ && buf)
    memcpy(buf, eov->wsabuf.buf, dwRes);

  return dwRes;
}

int iocp_iod_connect(struct npool *nsp, int sockfd, const struct sockaddr *addr, socklen_t addrlen) {
  return 0;
}

int iocp_iod_read(struct npool *nsp, int sockfd, void *buf, size_t len, int flags, struct sockaddr *src_addr, socklen_t *addrlen) {
  return get_overlapped_result(nsp, sockfd, buf, len);
}

int iocp_iod_write(struct npool *nsp, int sockfd, const void *buf, size_t len, int flags, const struct sockaddr *dest_addr, socklen_t addrlen) {
  return get_overlapped_result(nsp, sockfd, buf, len);
}

#endif /* HAVE_IOCP */
```