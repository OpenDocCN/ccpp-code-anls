# `nmap\nsock\src\nsock_pool.c`

```cpp
/* $Id$ */
// 定义文件标识符

#include "nsock_internal.h"
#include "nsock_log.h"
#include "gh_list.h"
#include "netutils.h"

#include <string.h>
#ifdef HAVE_SYS_TIME_H
#include <sys/time.h>
#endif
#ifdef HAVE_UNISTD_H
#include <unistd.h>
#endif
#include <signal.h>

#if HAVE_SYS_RESOURCE_H
#include <sys/resource.h>
#endif

extern struct timeval nsock_tod;

/* To use this library, the first thing they must do is create a pool
 * so we do the initialization during the first pool creation */
static int nsocklib_initialized = 0;
// 初始化 nsock 库标志

/* defined in nsock_engines.h */
struct io_engine *get_io_engine(void);
// 获取 I/O 引擎

/* ---- INTERNAL FUNCTIONS PROTOTYPES ---- */
static void nsock_library_initialize(void);
// 内部函数原型：初始化 nsock 库
/* --------------------------------------- */


/* This next function returns the errno style error code -- which is only
 * valid if the status NSOCK_LOOP_ERROR was returned by nsock_loop() */
int nsock_pool_get_error(nsock_pool nsp) {
  struct npool *mt = (struct npool *)nsp;
  return mt->errnum;
}
// 获取错误码

/* Sometimes it is useful to store a pointer to information inside
 * the NSP so you can retrieve it during a callback. */
void nsock_pool_set_udata(nsock_pool nsp, void *data) {
  struct npool *mt = (struct npool *)nsp;
  mt->userdata = data;
}
// 设置用户数据

/* And the define above wouldn't make much sense if we didn't have a way
 * to retrieve that data ... */
void *nsock_pool_get_udata(nsock_pool nsp) {
  struct npool *mt = (struct npool *)nsp;
  return mt->userdata;
}
// 获取用户数据

/* Turns on or off broadcast support on new sockets. Default is off (0, false)
 * set in nsock_pool_new(). Any non-zero (true) value sets SO_BROADCAST on all new
 * sockets (value of optval will be used directly in the setsockopt() call */
void nsock_pool_set_broadcast(nsock_pool nsp, int optval) {
  struct npool *mt = (struct npool *)nsp;
  mt->broadcast = optval;
}
// 设置广播支持

/* Sets the name of the interface for new sockets to bind to. */
// 设置新套接字绑定的接口名称
# 设置 nsock_pool 的设备
void nsock_pool_set_device(nsock_pool nsp, const char *device) {
  # 将 nsp 转换为 npool 结构体
  struct npool *mt = (struct npool *)nsp;
  # 设置设备
  mt->device = device;
}

# 比较可过期事件的函数
static int expirable_cmp(gh_hnode_t *n1, gh_hnode_t *n2) {
  # 定义两个可过期事件指针
  struct nevent *nse1;
  struct nevent *nse2;

  # 将 n1 和 n2 转换为 nevent 结构体
  nse1 = container_of(n1, struct nevent, expire);
  nse2 = container_of(n2, struct nevent, expire);

  # 比较两个事件的超时时间
  return (TIMEVAL_BEFORE(nse1->timeout, nse2->timeout)) ? 1 : 0;
}

# 创建 nsock_pool 的函数
nsock_pool nsock_pool_new(void *userdata) {
  # 定义 nsp 结构体指针
  struct npool *nsp;

  # 如果 nsock 库未初始化，则初始化
  if (!nsocklib_initialized) {
    nsock_library_initialize();
    nsocklib_initialized = 1;
  }

  # 分配内存并初始化 nsp 结构体
  nsp = (struct npool *)safe_malloc(sizeof(*nsp));
  memset(nsp, 0, sizeof(*nsp));

  # 获取当前时间
  gettimeofday(&nsock_tod, NULL);

  # 设置 userdata
  nsp->userdata = userdata;

  # 获取 IO 引擎并初始化 nsp
  nsp->engine = get_io_engine();
  nsock_engine_init(nsp);

  # 初始化 IO 事件列表
  gh_list_init(&nsp->connect_events);
  gh_list_init(&nsp->read_events);
  gh_list_init(&nsp->write_events);
#if HAVE_PCAP
  gh_list_init(&nsp->pcap_read_events);
#endif

  # 初始化定时器堆
  gh_heap_init(&nsp->expirables, expirable_cmp);

  # 初始化 IOD 列表
  gh_list_init(&nsp->active_iods);

  # 初始化缓存
  gh_list_init(&nsp->free_iods);
  gh_list_init(&nsp->free_events);

  # 设置下一个事件序列号
  nsp->next_event_serial = 1;

  # 初始化设备为 NULL
  nsp->device = NULL;

#if HAVE_OPENSSL
  # 初始化 SSL 上下文和 DTLS 上下文
  nsp->sslctx = NULL;
  nsp->dtlsctx = NULL;
#endif

  # 初始化 px_chain
  nsp->px_chain = NULL;

  # 返回 nsp
  return (nsock_pool)nsp;
}

# 如果 nsock_pool_new 返回成功，则在使用完毕后必须释放 nsp 以节省内存（有时还有套接字）。在此调用之后，nsp 可能不再使用。任何待处理事件都会收到 NSE_STATUS_KILL 回调，并且所有未完成的 IOD 都会被删除。
# 删除给定的 nsock_pool 对象
void nsock_pool_delete(nsock_pool ms_pool) {
  # 将传入的 nsock_pool 转换为 npool 结构体指针
  struct npool *nsp = (struct npool *)ms_pool;
  struct nevent *nse;
  struct niod *nsi;
  int i;
  gh_lnode_t *current, *next;
  # 定义包含事件列表的数组
  gh_list_t *event_lists[] = {
    &nsp->connect_events,
    &nsp->read_events,
    &nsp->write_events,
#if HAVE_PCAP
    &nsp->pcap_read_events,
#endif
    NULL
  };

  # 断言 nsp 不为空
  assert(nsp);

  # 遍历所有事件列表，发送 NSE_STATUS_KILL
  for (i = 0; event_lists[i] != NULL; i++) {
    while (gh_list_count(event_lists[i]) > 0) {
      # 从事件列表中弹出节点
      gh_lnode_t *lnode = gh_list_pop(event_lists[i]);

      # 断言节点不为空
      assert(lnode);

#if HAVE_PCAP
      # 如果是 pcap 读事件列表，则使用 lnode_nevent2 转换为 nevent 结构体指针
      if (event_lists[i] == &nsp->pcap_read_events)
        nse = lnode_nevent2(lnode);
      else
#endif
        # 否则使用 lnode_nevent 转换为 nevent 结构体指针
        nse = lnode_nevent(lnode);

      # 断言 nevent 不为空
      assert(nse);

      # 设置 nevent 的状态为 NSE_STATUS_KILL
      nse->status = NSE_STATUS_KILL;
      # 调用 nsock_trace_handler_callback 函数
      nsock_trace_handler_callback(nsp, nse);
      # 调用 nevent 的处理函数
      nse->handler(nsp, nse, nse->userdata);

      # 如果 nevent 的 iod 不为空
      if (nse->iod) {
        # 减少 iod 的待处理事件数量
        nse->iod->events_pending--;
        # 断言待处理事件数量大于等于 0
        assert(nse->iod->events_pending >= 0);
      }
      # 删除事件
      event_delete(nsp, nse);
    }
    # 释放事件列表
    gh_list_free(event_lists[i]);
  }

  # 处理定时器，它们不在事件列表中
  while (gh_heap_count(&nsp->expirables) > 0) {
    gh_hnode_t *hnode;

    # 从堆中弹出节点
    hnode = gh_heap_pop(&nsp->expirables);
    # 将节点转换为 nevent 结构体指针
    nse = container_of(hnode, struct nevent, expire);

    # 如果是定时器类型的事件
    if (nse->type == NSE_TYPE_TIMER) {
      # 设置 nevent 的状态为 NSE_STATUS_KILL
      nse->status = NSE_STATUS_KILL;
      # 调用 nsock_trace_handler_callback 函数
      nsock_trace_handler_callback(nsp, nse);
      # 调用 nevent 的处理函数
      nse->handler(nsp, nse, nse->userdata);
      # 删除事件
      event_delete(nsp, nse);
      # 将 nevent 添加到空闲事件列表中
      gh_list_append(&nsp->free_events, &nse->nodeq_io);
    }
  }

  # 释放堆
  gh_heap_free(&nsp->expirables);

  # 遍历活跃的 niod 结构体
  for (current = gh_list_first_elem(&nsp->active_iods);
       current != NULL;
       current = next) {
    next = gh_lnode_next(current);
    # 将节点转换为 niod 结构体指针
    nsi = container_of(current, struct niod, nodeq);

    # 删除 niod
    nsock_iod_delete(nsi, NSOCK_PENDING_ERROR);

    # 从活跃的 niod 列表中移除当前节点
    gh_list_remove(&nsp->active_iods, current);
  # 将节点插入到链表的头部
  gh_list_prepend(&nsp->free_iods, &nsi->nodeq);
  }

  # 释放所有在空闲 IOD 列表中的内存
  while ((current = gh_list_pop(&nsp->free_iods))) {
    # 从当前节点中获取 niod 结构体指针
    nsi = container_of(current, struct niod, nodeq);
    # 释放 niod 结构体指针指向的内存
    free(nsi);
  }

  # 释放所有在空闲事件列表中的内存
  while ((current = gh_list_pop(&nsp->free_events))) {
    # 从当前节点中获取 nse 结构体指针
    nse = lnode_nevent(current);
    # 释放 nse 结构体指针指向的内存
    free(nse);
  }

  # 释放活跃 IOD 列表的内存
  gh_list_free(&nsp->active_iods);
  # 释放空闲 IOD 列表的内存
  gh_list_free(&nsp->free_iods);
  # 释放空闲事件列表的内存
  gh_list_free(&nsp->free_events);

  # 销毁网络套接字引擎
  nsock_engine_destroy(nsp);
#if HAVE_OPENSSL
  // 如果有 OpenSSL，执行 nsp_ssl_cleanup 函数清理 nsp 对象
  nsp_ssl_cleanup(nsp);
#endif

  // 释放 nsp 对象的内存
  free(nsp);
}

void nsock_library_initialize(void) {
#ifndef WIN32
  rlim_t res;

  /* We want to make darn sure the evil SIGPIPE is ignored */
  // 忽略 SIGPIPE 信号
  signal(SIGPIPE, SIG_IGN);

  /* And we're gonna need sockets -- LOTS of sockets ... */
  // 最大化文件描述符限制
  res = maximize_fdlimit();
  // 断言文件描述符限制大于 7
  assert(res > 7);
#endif
  return;
}
```