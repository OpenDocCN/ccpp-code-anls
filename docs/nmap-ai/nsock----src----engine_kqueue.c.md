# `nmap\nsock\src\engine_kqueue.c`

```
/* $Id$ */

#ifdef HAVE_CONFIG_H
#include "nsock_config.h"
#endif

#if HAVE_KQUEUE

#include <sys/types.h>
#include <sys/event.h>
#include <sys/time.h>
#include <errno.h>

#include "nsock_internal.h"
#include "nsock_log.h"

#if HAVE_PCAP
#include "nsock_pcap.h"
#endif

#define INITIAL_EV_COUNT  128


/* --- ENGINE INTERFACE PROTOTYPES --- */
// 初始化 kqueue 引擎
static int kqueue_init(struct npool *nsp);
// 销毁 kqueue 引擎
static void kqueue_destroy(struct npool *nsp);
// 注册 I/O 事件到 kqueue 引擎
static int kqueue_iod_register(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev);
// 从 kqueue 引擎注销 I/O 事件
static int kqueue_iod_unregister(struct npool *nsp, struct niod *iod);
// 修改 kqueue 引擎中的 I/O 事件
static int kqueue_iod_modify(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev_set, int ev_clr);
// kqueue 引擎的事件循环
static int kqueue_loop(struct npool *nsp, int msec_timeout);

extern struct io_operations posix_io_operations;

/* ---- ENGINE DEFINITION ---- */
// kqueue 引擎的定义
struct io_engine engine_kqueue = {
  "kqueue",
  kqueue_init,
  kqueue_destroy,
  kqueue_iod_register,
  kqueue_iod_unregister,
  kqueue_iod_modify,
  kqueue_loop,
  &posix_io_operations
};


/* --- INTERNAL PROTOTYPES --- */
// 遍历事件列表
static void iterate_through_event_lists(struct npool *nsp, int evcount);

/* defined in nsock_core.c */
// 处理 I/O 事件
void process_iod_events(struct npool *nsp, struct niod *nsi, int ev);
// 处理事件
void process_event(struct npool *nsp, gh_list_t *evlist, struct nevent *nse, int ev);
// 处理过期事件
void process_expired_events(struct npool *nsp);
#if HAVE_PCAP
#ifndef PCAP_CAN_DO_SELECT
// 在非 select 模式下读取 pcap
int pcap_read_on_nonselect(struct npool *nsp);
#endif
#endif

/* defined in nsock_event.c */
// 更新首个事件
void update_first_events(struct nevent *nse);


extern struct timeval nsock_tod;


/*
 * Engine specific data structure
 */
// kqueue 引擎特定的数据结构
struct kqueue_engine_info {
  int kqfd;
  int maxfd;
  size_t evlen;
  struct kevent *events;
};
// 初始化 kqueue 引擎信息
int kqueue_init(struct npool *nsp) {
  // 分配 kqueue 引擎信息结构体的内存空间
  struct kqueue_engine_info *kinfo;
  kinfo = (struct kqueue_engine_info *)safe_malloc(sizeof(struct kqueue_engine_info));

  // 创建 kqueue 文件描述符
  kinfo->kqfd = kqueue();
  // 初始化最大文件描述符为 -1
  kinfo->maxfd = -1;
  // 初始化事件长度为初始事件数量
  kinfo->evlen = INITIAL_EV_COUNT;
  // 分配事件数组的内存空间
  kinfo->events = (struct kevent *)safe_malloc(INITIAL_EV_COUNT * sizeof(struct kevent));

  // 将 kinfo 结构体指针赋值给引擎数据
  nsp->engine_data = (void *)kinfo;

  return 1;
}

// 销毁 kqueue 引擎信息
void kqueue_destroy(struct npool *nsp) {
  // 获取引擎数据中的 kinfo 结构体指针
  struct kqueue_engine_info *kinfo = (struct kqueue_engine_info *)nsp->engine_data;

  // 断言 kinfo 不为空
  assert(kinfo != NULL);
  // 关闭 kqueue 文件描述符
  close(kinfo->kqfd);
  // 释放事件数组的内存空间
  free(kinfo->events);
  // 释放 kinfo 结构体的内存空间
  free(kinfo);
}

// 注册 I/O 事件到 kqueue 引擎
int kqueue_iod_register(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev) {
  // 获取引擎数据中的 kinfo 结构体指针
  struct kqueue_engine_info *kinfo = (struct kqueue_engine_info *)nsp->engine_data;

  // 断言 iod 未注册
  assert(!IOD_PROPGET(iod, IOD_REGISTERED));

  // 设置 iod 的注册状态为已注册
  IOD_PROPSET(iod, IOD_REGISTERED);
  // 设置 iod 的监视事件为空
  iod->watched_events = EV_NONE;

  // 修改 iod 的监视事件
  kqueue_iod_modify(nsp, iod, nse, ev, EV_NONE);

  // 如果 iod 的文件描述符大于最大文件描述符，则更新最大文件描述符
  if (nsock_iod_get_sd(iod) > kinfo->maxfd)
    kinfo->maxfd = nsock_iod_get_sd(iod);

  return 1;
}

// 从 kqueue 引擎中注销 I/O 事件
int kqueue_iod_unregister(struct npool *nsp, struct niod *iod) {
  // 获取引擎数据中的 kinfo 结构体指针
  struct kqueue_engine_info *kinfo = (struct kqueue_engine_info *)nsp->engine_data;

  /* 一些 IOD 可能在此处被注销，如果它们与立即完成的事件相关联 */
  if (IOD_PROPGET(iod, IOD_REGISTERED)) {
    // 修改 iod 的监视事件
    kqueue_iod_modify(nsp, iod, NULL, EV_NONE, EV_READ|EV_WRITE);
    // 清除 iod 的注册状态
    IOD_PROPCLR(iod, IOD_REGISTERED);

    // 如果 iod 的文件描述符等于最大文件描述符，则减小最大文件描述符
    if (nsock_iod_get_sd(iod) == kinfo->maxfd)
      kinfo->maxfd--;
  }
  // 设置 iod 的监视事件为空
  iod->watched_events = EV_NONE;
  return 1;
}

// 定义宏，用于设置事件标志
#define EV_SETFLAG(_set, _ev) (((_set) & (_ev)) ? (EV_ADD|EV_ENABLE) : (EV_ADD|EV_DISABLE))
int kqueue_iod_modify(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev_set, int ev_clr) {
  // 创建 kevent 结构体数组，用于存储事件
  struct kevent kev[2];
  int new_events, i;
  // 获取 kqueue 引擎信息
  struct kqueue_engine_info *kinfo = (struct kqueue_engine_info *)nsp->engine_data;

  // 断言 ev_set 和 ev_clr 没有重叠的事件
  assert((ev_set & ev_clr) == 0);
  // 断言 IOD 已经注册
  assert(IOD_PROPGET(iod, IOD_REGISTERED));

  // 计算新的事件集合
  new_events = iod->watched_events;
  new_events |= ev_set;
  new_events &= ~ev_clr;

  // 如果新的事件集合和之前的一样，则无需修改，直接返回
  if (new_events == iod->watched_events)
    return 1; /* nothing to do */

  i = 0;
  // 如果读事件有变化，则设置 kevent 结构体
  if ((ev_set ^ ev_clr) & EV_READ) {
    EV_SET(&kev[i], nsock_iod_get_sd(iod), EVFILT_READ, EV_SETFLAG(ev_set, EV_READ), 0, 0, (void *)iod);
    i++;
  }
  // 如果写事件有变化，则设置 kevent 结构体
  if ((ev_set ^ ev_clr) & EV_WRITE) {
    EV_SET(&kev[i], nsock_iod_get_sd(iod), EVFILT_WRITE, EV_SETFLAG(ev_set, EV_WRITE), 0, 0, (void *)iod);
    i++;
  }

  // 如果有事件变化，则调用 kevent 更新事件
  if (i > 0 && kevent(kinfo->kqfd, kev, i, NULL, 0, NULL) < 0)
    fatal("Unable to update events for IOD #%lu: %s", iod->id, strerror(errno));

  // 更新 IOD 的事件集合
  iod->watched_events = new_events;
  return 1;
}

int kqueue_loop(struct npool *nsp, int msec_timeout) {
  int results_left = 0;
  int event_msecs; /* msecs before an event goes off */
  int combined_msecs;
  struct timespec ts, *ts_p;
  int sock_err = 0;
  // 获取 kqueue 引擎信息
  struct kqueue_engine_info *kinfo = (struct kqueue_engine_info *)nsp->engine_data;

  // 断言超时时间大于等于 -1
  assert(msec_timeout >= -1);

  // 如果没有待处理的事件，则直接返回
  if (nsp->events_pending == 0)
    return 0; /* No need to wait on 0 events ... */

  // 如果活跃的 IOD 数量大于事件数组的长度，则重新分配事件数组的内存空间
  if (gh_list_count(&nsp->active_iods) > kinfo->evlen) {
    kinfo->evlen = gh_list_count(&nsp->active_iods) * 2;
    kinfo->events = (struct kevent *)safe_realloc(kinfo->events, kinfo->evlen * sizeof(struct kevent));
  }

  do {
    struct nevent *nse;

    nsock_log_debug_all("wait for events");

    // 获取下一个将要超时的事件
    nse = next_expirable_event(nsp);
    if (!nse)
      event_msecs = -1; /* None of the events specified a timeout */
    else
      // 计算距离下一个事件超时的时间
      event_msecs = MAX(0, TIMEVAL_MSEC_SUBTRACT(nse->timeout, nsock_tod));

#if HAVE_PCAP
#ifndef PCAP_CAN_DO_SELECT
    /* 在无法使用select()函数的系统上捕获数据包时，强制设置低超时时间 */
    if (gh_list_count(&nsp->pcap_read_events) > 0)  // 检查是否有待处理的pcap读取事件
      if (event_msecs > PCAP_POLL_INTERVAL)  // 如果事件超时时间大于PCAP_POLL_INTERVAL
        event_msecs = PCAP_POLL_INTERVAL;  // 将事件超时时间设置为PCAP_POLL_INTERVAL
#endif
#endif

    /* We cast to unsigned because we want -1 to be very high (since it means no
     * timeout) */
    // 将值转换为无符号整数，因为我们希望-1表示非常高的值（因为它表示没有超时）
    combined_msecs = MIN((unsigned)event_msecs, (unsigned)msec_timeout);

    /* Set up the timeval pointer we will give to kevent() */
    // 设置我们将传递给kevent()的时间值指针
    memset(&ts, 0, sizeof(struct timespec));
    if (combined_msecs >= 0) {
      ts.tv_sec = combined_msecs / 1000;
      ts.tv_nsec = (combined_msecs % 1000) * 1000000L;
      ts_p = &ts;
    } else {
      ts_p = NULL;
    }

#if HAVE_PCAP
#ifndef PCAP_CAN_DO_SELECT
    /* do non-blocking read on pcap devices that doesn't support select()
     * If there is anything read, just leave this loop. */
    // 在不支持select()的pcap设备上进行非阻塞读取
    // 如果有任何内容被读取，就离开这个循环
    if (pcap_read_on_nonselect(nsp)) {
      /* okay, something was read. */
      // 好的，有东西被读取了
    } else
#endif
#endif
    {
      results_left = kevent(kinfo->kqfd, NULL, 0, kinfo->events, kinfo->evlen, ts_p);
      if (results_left == -1)
        sock_err = socket_errno();
    }

    gettimeofday(&nsock_tod, NULL); /* Due to kevent delay */
    // 由于kevent延迟，获取当前时间
  } while (results_left == -1 && sock_err == EINTR); /* repeat only if signal occurred */
  // 只有在发生信号时才重复

  if (results_left == -1 && sock_err != EINTR) {
    nsock_log_error("nsock_loop error %d: %s", sock_err, socket_strerror(sock_err));
    nsp->errnum = sock_err;
    return -1;
  }

  iterate_through_event_lists(nsp, results_left);
  // 遍历事件列表

  return 1;
}


/* ---- INTERNAL FUNCTIONS ---- */

static inline int get_evmask(struct niod *nsi, const struct kevent *kev) {
  int evmask = EV_NONE;

  /* generate the corresponding event mask with nsock event flags */
  // 使用nsock事件标志生成相应的事件掩码
  if (kev->flags & EV_ERROR) {
    evmask |= EV_EXCEPT;

    if (kev->data == EPIPE && (nsi->watched_events & EV_READ))
      evmask |= EV_READ;
  } else {
    switch (kev->filter) {
      case EVFILT_READ:
        evmask |= EV_READ;
        break;

      case EVFILT_WRITE:
        evmask |= EV_WRITE;
        break;

      default:
        fatal("Unsupported filter value: %d\n", (int)kev->filter);
    }
  }
  return evmask;
}
/* 遍历所有事件列表（如connect_events, read_events, timer_events等），并对已完成的事件（由于超时、I/O等）采取行动 */
void iterate_through_event_lists(struct npool *nsp, int evcount) {
  int n;
  struct kqueue_engine_info *kinfo = (struct kqueue_engine_info *)nsp->engine_data;
  struct niod *nsi;

  for (n = 0; n < evcount; n++) {
    struct kevent *kev = &kinfo->events[n];

    nsi = (struct niod *)kev->udata;

    /* 处理该 IOD 的所有待处理事件 */
    process_iod_events(nsp, nsi, get_evmask(nsi, kev));

    IOD_PROPSET(nsi, IOD_PROCESSED);
  }

  for (n = 0; n < evcount; n++) {
    struct kevent *kev = &kinfo->events[n];

    nsi = (struct niod *)kev->udata;

    if (nsi->state == NSIOD_STATE_DELETED) {
      if (IOD_PROPGET(nsi, IOD_PROCESSED)) {
        IOD_PROPCLR(nsi, IOD_PROCESSED);
        gh_list_remove(&nsp->active_iods, &nsi->nodeq);
        gh_list_prepend(&nsp->free_iods, &nsi->nodeq);
      }
    }
  }

  /* 遍历定时器和已过期事件 */
  process_expired_events(nsp);
}

#endif /* HAVE_KQUEUE */
```