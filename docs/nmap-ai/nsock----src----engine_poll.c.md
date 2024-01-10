# `nmap\nsock\src\engine_poll.c`

```
/* $Id$ */

#ifndef WIN32
/* 如果不是在 Windows 环境下，则允许使用 POLLRDHUP */
#define _GNU_SOURCE
#endif

#ifdef HAVE_CONFIG_H
#include "nsock_config.h"
#elif WIN32
#include "nsock_winconfig.h"
#endif

#if HAVE_POLL

#include <errno.h>

#ifndef WIN32
#include <poll.h>
#else
#include <Winsock2.h>
#endif /* ^WIN32 */

#include "nsock_internal.h"
#include "nsock_log.h"

#if HAVE_PCAP
#include "nsock_pcap.h"
#endif

#define EV_LIST_INIT_SIZE 1024

#ifdef WIN32
  #define Poll    WSAPoll
  #define POLLFD  WSAPOLLFD
#else
  #define Poll    poll
  #define POLLFD  struct pollfd
#endif

#ifdef WIN32
  #define POLL_R_FLAGS (POLLIN)
#else
  #define POLL_R_FLAGS (POLLIN | POLLPRI)
#endif /* WIN32 */

#define POLL_W_FLAGS POLLOUT
#ifdef POLLRDHUP
  #define POLL_X_FLAGS (POLLERR | POLLHUP | POLLRDHUP)
#else
  /* POLLRDHUP 是后来引入的，可能在旧系统上不可用。 */
  #define POLL_X_FLAGS (POLLERR | POLLHUP)
#endif /* POLLRDHUP */

extern struct io_operations posix_io_operations;

/* --- ENGINE INTERFACE PROTOTYPES --- */
static int poll_init(struct npool *nsp);
static void poll_destroy(struct npool *nsp);
static int poll_iod_register(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev);
static int poll_iod_unregister(struct npool *nsp, struct niod *iod);
static int poll_iod_modify(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev_set, int ev_clr);
static int poll_loop(struct npool *nsp, int msec_timeout);


/* ---- ENGINE DEFINITION ---- */
struct io_engine engine_poll = {
  "poll",
  poll_init,
  poll_destroy,
  poll_iod_register,
  poll_iod_unregister,
  poll_iod_modify,
  poll_loop,
  &posix_io_operations
};


/* --- INTERNAL PROTOTYPES --- */
static void iterate_through_event_lists(struct npool *nsp);

/* 在 nsock_core.c 中定义 */
void process_iod_events(struct npool *nsp, struct niod *nsi, int ev);
void process_event(struct npool *nsp, gh_list_t *evlist, struct nevent *nse, int ev);
void process_expired_events(struct npool *nsp);
#if HAVE_PCAP
#ifndef PCAP_CAN_DO_SELECT
int pcap_read_on_nonselect(struct npool *nsp);
#endif
#endif

// 如果定义了 HAVE_PCAP 并且未定义 PCAP_CAN_DO_SELECT，则声明 pcap_read_on_nonselect 函数


/* defined in nsock_event.c */
void update_first_events(struct nevent *nse);

// 声明 update_first_events 函数，该函数在 nsock_event.c 文件中定义


extern struct timeval nsock_tod;

// 声明外部变量 nsock_tod，类型为 struct timeval


/*
 * Engine specific data structure
 */
struct poll_engine_info {
  int capacity;
  int max_fd;
  /* index of the highest poll event */
  POLLFD *events;
};

// 定义 poll_engine_info 结构体，包含 capacity、max_fd 和 events 三个成员变量


static inline int lower_max_fd(struct poll_engine_info *pinfo) {
  do {
    pinfo->max_fd--;
  } while (pinfo->max_fd >= 0 && pinfo->events[pinfo->max_fd].fd == -1);

  return pinfo->max_fd;
}

// 定义内联函数 lower_max_fd，用于降低 max_fd 的值，直到找到一个有效的文件描述符


static inline int evlist_grow(struct poll_engine_info *pinfo) {
  int i;

  i = pinfo->capacity;

  if (pinfo->capacity == 0) {
    pinfo->capacity = EV_LIST_INIT_SIZE;
    pinfo->events = (POLLFD *)safe_malloc(sizeof(POLLFD) * pinfo->capacity);
  } else {
    pinfo->capacity *= 2;
    pinfo->events = (POLLFD *)safe_realloc(pinfo->events, sizeof(POLLFD) * pinfo->capacity);
  }

  while (i < pinfo->capacity) {
    pinfo->events[i].fd = -1;
    pinfo->events[i].events = 0;
    pinfo->events[i].revents = 0;
    i++;
  }
  return pinfo->capacity;
}

// 定义内联函数 evlist_grow，用于增加事件列表的容量，并初始化新增的事件结构


int poll_init(struct npool *nsp) {
  struct poll_engine_info *pinfo;

  pinfo = (struct poll_engine_info *)safe_malloc(sizeof(struct poll_engine_info));
  pinfo->capacity = 0;
  pinfo->max_fd = -1;
  evlist_grow(pinfo);

  nsp->engine_data = (void *)pinfo;

  return 1;
}

// 初始化 poll 引擎，分配内存并初始化相关数据结构


void poll_destroy(struct npool *nsp) {
  struct poll_engine_info *pinfo = (struct poll_engine_info *)nsp->engine_data;

  assert(pinfo != NULL);
  free(pinfo->events);
  free(pinfo);
}

// 销毁 poll 引擎，释放相关内存


int poll_iod_register(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev) {
  struct poll_engine_info *pinfo = (struct poll_engine_info *)nsp->engine_data;
  int sd;

  assert(!IOD_PROPGET(iod, IOD_REGISTERED));

  iod->watched_events = ev;

  sd = nsock_iod_get_sd(iod);
  while (pinfo->capacity < sd + 1)

// 注册 I/O 处理器，设置相关事件并获取文件描述符
    # 扩展事件列表以容纳新的文件描述符
    evlist_grow(pinfo);

  # 设置事件结构体中的文件描述符
  pinfo->events[sd].fd = sd;
  # 初始化事件结构体中的事件和触发事件
  pinfo->events[sd].events = 0;
  pinfo->events[sd].revents = 0;

  # 更新最大文件描述符
  pinfo->max_fd = MAX(pinfo->max_fd, sd);

  # 如果事件包含读取事件
  if (ev & EV_READ)
    # 设置事件结构体中的事件为读取事件的标志
    pinfo->events[sd].events |= POLL_R_FLAGS;
  # 如果事件包含写入事件
  if (ev & EV_WRITE)
    # 设置事件结构体中的事件为写入事件的标志
    pinfo->events[sd].events |= POLL_W_FLAGS;
#ifndef WIN32
  // 如果不是在 Windows 平台下
  if (ev & EV_EXCEPT)
    // 如果事件包含异常事件
    pinfo->events[sd].events |= POLL_X_FLAGS;
#endif

  // 设置 IOD 的属性为已注册
  IOD_PROPSET(iod, IOD_REGISTERED);
  // 返回 1，表示成功
  return 1;
}

// 取消注册 IOD
int poll_iod_unregister(struct npool *nsp, struct niod *iod) {
  // 将 IOD 的监视事件设置为无
  iod->watched_events = EV_NONE;

  /* 一些 IOD 可能会在这里被注销，如果它们关联的事件立即完成 */
  if (IOD_PROPGET(iod, IOD_REGISTERED)) {
    // 获取与 npool 关联的 poll_engine_info 结构体
    struct poll_engine_info *pinfo = (struct poll_engine_info *)nsp->engine_data;
    int sd;

    // 获取 IOD 对应的套接字描述符
    sd = nsock_iod_get_sd(iod);
    // 将事件数组中对应套接字描述符的事件设置为无
    pinfo->events[sd].fd = -1;
    pinfo->events[sd].events = 0;
    pinfo->events[sd].revents = 0;

    // 如果最大套接字描述符等于当前套接字描述符
    if (pinfo->max_fd == sd)
      // 降低最大套接字描述符
      lower_max_fd(pinfo);

    // 清除 IOD 的注册属性
    IOD_PROPCLR(iod, IOD_REGISTERED);
  }
  // 返回 1，表示成功
  return 1;
}

// 修改 IOD 的监视事件
int poll_iod_modify(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev_set, int ev_clr) {
  int sd;
  int new_events;
  struct poll_engine_info *pinfo = (struct poll_engine_info *)nsp->engine_data;

  // 断言 ev_set 和 ev_clr 的交集为空
  assert((ev_set & ev_clr) == 0);
  // 断言 IOD 已经注册
  assert(IOD_PROPGET(iod, IOD_REGISTERED));

  // 计算新的监视事件
  new_events = iod->watched_events;
  new_events |= ev_set;
  new_events &= ~ev_clr;

  // 如果新的监视事件和原来的一样，直接返回 1
  if (new_events == iod->watched_events)
    return 1; /* 没有需要做的 */

  // 更新 IOD 的监视事件
  iod->watched_events = new_events;

  // 获取 IOD 对应的套接字描述符
  sd = nsock_iod_get_sd(iod);

  // 设置事件数组中对应套接字描述符的事件为无
  pinfo->events[sd].fd = sd;
  pinfo->events[sd].events = 0;

  // 重新生成 IOD 的当前事件集合
  if (iod->watched_events & EV_READ)
    pinfo->events[sd].events |= POLL_R_FLAGS;
  if (iod->watched_events & EV_WRITE)
    pinfo->events[sd].events |= POLL_W_FLAGS;

  // 返回 1，表示成功
  return 1;
}

// 轮询事件
int poll_loop(struct npool *nsp, int msec_timeout) {
  int results_left = 0;
  int event_msecs; /* 事件触发前的毫秒数 */
  int combined_msecs;
  int sock_err = 0;
  struct poll_engine_info *pinfo = (struct poll_engine_info *)nsp->engine_data;

  // 断言毫秒超时大于等于 -1
  assert(msec_timeout >= -1);

  // 如果没有待处理的事件，直接返回 0
  if (nsp->events_pending == 0)
    return 0; /* 不需要等待 0 个事件... */

  do {
    struct nevent *nse;
    # 打印调试信息，等待事件发生
    nsock_log_debug_all("wait for events");

    # 获取下一个可过期事件
    nse = next_expirable_event(nsp);
    # 如果没有可过期事件，则将事件毫秒数设置为-1
    if (!nse)
      event_msecs = -1; /* None of the events specified a timeout */
    # 否则，计算当前时间和下一个可过期事件的时间差，作为事件的毫秒数
    else
      event_msecs = MAX(0, TIMEVAL_MSEC_SUBTRACT(nse->timeout, nsock_tod));
#if HAVE_PCAP
#ifndef PCAP_CAN_DO_SELECT
    /* 如果系统不支持 pcap 描述符的 select() 操作，则强制设置低超时时间 */
    if (gh_list_count(&nsp->pcap_read_events) > 0)
      if (event_msecs > PCAP_POLL_INTERVAL)
        event_msecs = PCAP_POLL_INTERVAL;
#endif
#endif

    /* 将 event_msecs 和 msec_timeout 转换为无符号数，因为我们希望 -1 被视为非常大的值（因为它表示没有超时） */
    combined_msecs = MIN((unsigned)event_msecs, (unsigned)msec_timeout);

#if HAVE_PCAP
#ifndef PCAP_CAN_DO_SELECT
    /* 在不支持 select() 操作的 pcap 设备上进行非阻塞读取
     * 如果有任何内容被读取，就离开循环 */
    if (pcap_read_on_nonselect(nsp)) {
      /* 好的，有内容被读取 */
    } else
#endif
#endif
    {
      results_left = Poll(pinfo->events, pinfo->max_fd + 1, combined_msecs);
      if (results_left == -1)
        sock_err = socket_errno();
    }

    gettimeofday(&nsock_tod, NULL); /* 由于 poll 延迟 */
  } while (results_left == -1 && sock_err == EINTR); /* 只有在发生信号时才重复 */

  if (results_left == -1 && sock_err != EINTR) {
#ifdef WIN32
    for (int i = 0; sock_err != EINVAL || i <= pinfo->max_fd; i++) {
      if (sock_err != EINVAL || pinfo->events[i].fd != -1) {
#endif
        nsock_log_error("nsock_loop error %d: %s", sock_err, socket_strerror(sock_err));
        nsp->errnum = sock_err;
        return -1;
#ifdef WIN32
      }
    }
#endif
  }

  iterate_through_event_lists(nsp);

  return 1;
}


/* ---- INTERNAL FUNCTIONS ---- */

static inline int get_evmask(struct npool *nsp, struct niod *nsi) {
  struct poll_engine_info *pinfo = (struct poll_engine_info *)nsp->engine_data;
  int sd, evmask = EV_NONE;
  POLLFD *pev;

  if (nsi->state != NSIOD_STATE_DELETED
      && nsi->events_pending
      && IOD_PROPGET(nsi, IOD_REGISTERED)) {

#if HAVE_PCAP
      if (nsi->pcap)
        sd = ((mspcap *)nsi->pcap)->pcap_desc;
      else
#endif
        sd = nsi->sd;  // 获取当前事件的文件描述符

      assert(sd < pinfo->capacity);  // 断言文件描述符小于 pinfo 的容量
      pev = &pinfo->events[sd];  // 获取文件描述符对应的事件信息

      if (pev->revents & POLL_R_FLAGS)  // 如果事件包含可读标志
        evmask |= EV_READ;  // 将可读事件添加到事件掩码中
      if (pev->revents & POLL_W_FLAGS)  // 如果事件包含可写标志
        evmask |= EV_WRITE;  // 将可写事件添加到事件掩码中
      if (pev->events && (pev->revents & POLL_X_FLAGS))  // 如果事件不为空且包含异常标志
        evmask |= EV_EXCEPT;  // 将异常事件添加到事件掩码中
  }
  return evmask;  // 返回事件掩码
}

/* Iterate through all the event lists (such as connect_events, read_events,
 * timer_events, etc) and take action for those that have completed (due to
 * timeout, i/o, etc) */
void iterate_through_event_lists(struct npool *nsp) {
  gh_lnode_t *current, *next, *last;  // 定义链表节点指针

  last = gh_list_last_elem(&nsp->active_iods);  // 获取活跃 I/O 事件链表的最后一个元素

  for (current = gh_list_first_elem(&nsp->active_iods);  // 遍历活跃 I/O 事件链表
       current != NULL && gh_lnode_prev(current) != last;  // 当前节点不为空且前一个节点不是最后一个节点时
       current = next) {  // 更新当前节点为下一个节点
    struct niod *nsi = container_of(current, struct niod, nodeq);  // 获取当前节点对应的 niod 结构体指针

    process_iod_events(nsp, nsi, get_evmask(nsp, nsi));  // 处理 I/O 事件

    next = gh_lnode_next(current);  // 获取下一个节点
    if (nsi->state == NSIOD_STATE_DELETED) {  // 如果当前 niod 的状态为已删除
      gh_list_remove(&nsp->active_iods, current);  // 从活跃 I/O 事件链表中移除当前节点
      gh_list_prepend(&nsp->free_iods, current);  // 将当前节点添加到空闲 I/O 事件链表的开头
    }
  }

  /* iterate through timers and expired events */
  process_expired_events(nsp);  // 处理过期事件
}

#endif /* HAVE_POLL */
```