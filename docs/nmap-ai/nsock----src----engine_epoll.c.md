# `nmap\nsock\src\engine_epoll.c`

```cpp
/* $Id$ */

#ifdef HAVE_CONFIG_H
#include "nsock_config.h"
#endif

#if HAVE_EPOLL

#include <sys/epoll.h>
#include <errno.h>

#include "nsock_internal.h"
#include "nsock_log.h"

#if HAVE_PCAP
#include "nsock_pcap.h"
#endif

#define INITIAL_EV_COUNT  128

#define EPOLL_R_FLAGS (EPOLLIN | EPOLLPRI)
#define EPOLL_W_FLAGS EPOLLOUT

/* EPOLLRDHUP was introduced later and might be unavailable on older systems. */
#ifndef EPOLLRDHUP
  #define EPOLLRDHUP 0
#endif
#define EPOLL_X_FLAGS (EPOLLERR | EPOLLRDHUP| EPOLLHUP)


/* --- ENGINE INTERFACE PROTOTYPES --- */
// 初始化 epoll 引擎
static int epoll_init(struct npool *nsp);
// 销毁 epoll 引擎
static void epoll_destroy(struct npool *nsp);
// 注册 I/O 事件到 epoll 引擎
static int epoll_iod_register(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev);
// 从 epoll 引擎注销 I/O 事件
static int epoll_iod_unregister(struct npool *nsp, struct niod *iod);
// 修改 epoll 引擎中的 I/O 事件
static int epoll_iod_modify(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev_set, int ev_clr);
// epoll 引擎的事件循环
static int epoll_loop(struct npool *nsp, int msec_timeout);

extern struct io_operations posix_io_operations;

/* ---- ENGINE DEFINITION ---- */
// epoll 引擎的定义
struct io_engine engine_epoll = {
  "epoll",
  epoll_init,
  epoll_destroy,
  epoll_iod_register,
  epoll_iod_unregister,
  epoll_iod_modify,
  epoll_loop,
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
// 在非 select 模式下读取 pcap 数据
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
struct epoll_engine_info {
  /* 定义 epoll 引擎信息的结构体 */
  int epfd;  /* 对应 epoll 实例的文件描述符 */
  int evlen;  /* 我们可以处理的 epoll_events 的数量 */
  struct epoll_event *events;  /* epoll 事件的列表，如果需要的话会重新调整大小（当轮询大量 IODs 时） */
};


int epoll_init(struct npool *nsp) {
  struct epoll_engine_info *einfo;

  einfo = (struct epoll_engine_info *)safe_malloc(sizeof(struct epoll_engine_info));

  einfo->epfd = epoll_create(10); /* 参数被忽略 */
  einfo->evlen = INITIAL_EV_COUNT;
  einfo->events = (struct epoll_event *)safe_malloc(einfo->evlen * sizeof(struct epoll_event));

  nsp->engine_data = (void *)einfo;

  return 1;
}

void epoll_destroy(struct npool *nsp) {
  struct epoll_engine_info *einfo = (struct epoll_engine_info *)nsp->engine_data;

  assert(einfo != NULL);
  close(einfo->epfd);
  free(einfo->events);
  free(einfo);
}

int epoll_iod_register(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev) {
  int sd;
  struct epoll_event epev;
  struct epoll_engine_info *einfo = (struct epoll_engine_info *)nsp->engine_data;

  assert(!IOD_PROPGET(iod, IOD_REGISTERED));

  iod->watched_events = ev;

  memset(&epev, 0x00, sizeof(struct epoll_event));
  epev.events = EPOLLET;
  epev.data.ptr = (void *)iod;

  if (ev & EV_READ)
    epev.events |= EPOLL_R_FLAGS;
  if (ev & EV_WRITE)
    epev.events |= EPOLL_W_FLAGS;

  sd = nsock_iod_get_sd(iod);
  if (epoll_ctl(einfo->epfd, EPOLL_CTL_ADD, sd, &epev) < 0)
    fatal("Unable to register IOD #%lu: %s", iod->id, strerror(errno));

  IOD_PROPSET(iod, IOD_REGISTERED);
  return 1;
}

int epoll_iod_unregister(struct npool *nsp, struct niod *iod) {
  iod->watched_events = EV_NONE;

  /* 如果与立即完成的事件相关联的一些 IODs 可以在这里注销 */
  if (IOD_PROPGET(iod, IOD_REGISTERED)) {
    struct epoll_engine_info *einfo = (struct epoll_engine_info *)nsp->engine_data;
    int sd;

    sd = nsock_iod_get_sd(iod);
    # 从 epoll 实例中删除指定的文件描述符
    epoll_ctl(einfo->epfd, EPOLL_CTL_DEL, sd, NULL);
    # 清除指定 I/O 描述符的注册标志
    IOD_PROPCLR(iod, IOD_REGISTERED);
  }
  # 返回 1，表示操作成功
  return 1;
}

这是一个函数的结束标记


int epoll_iod_modify(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev_set, int ev_clr) {

定义一个名为epoll_iod_modify的函数，接受npool结构体指针nsp，niod结构体指针iod，nevent结构体指针nse，整型ev_set和ev_clr作为参数


  int sd;
  struct epoll_event epev;
  int new_events;
  struct epoll_engine_info *einfo = (struct epoll_engine_info *)nsp->engine_data;

声明整型变量sd，结构体epoll_event类型的epev，整型变量new_events，以及epoll_engine_info结构体指针einfo，并将nsp->engine_data强制转换为epoll_engine_info类型


  assert((ev_set & ev_clr) == 0);
  assert(IOD_PROPGET(iod, IOD_REGISTERED));

使用断言确保ev_set和ev_clr的按位与结果为0，以及IOD_PROPGET(iod, IOD_REGISTERED)的返回值为真


  memset(&epev, 0x00, sizeof(struct epoll_event));
  epev.events = EPOLLET;
  epev.data.ptr = (void *)iod;

使用0x00填充epev的内存，设置epev的events为EPOLLET，将epev的data.ptr指向iod


  new_events = iod->watched_events;
  new_events |= ev_set;
  new_events &= ~ev_clr;

将new_events设置为iod->watched_events，然后使用按位或和按位与操作更新new_events的值


  if (new_events == iod->watched_events)
    return 1; /* nothing to do */

如果new_events等于iod->watched_events，则返回1，表示没有需要执行的操作


  iod->watched_events = new_events;

将iod->watched_events更新为new_events的值


  if (iod->watched_events & EV_READ)
    epev.events |= EPOLL_R_FLAGS;
  if (iod->watched_events & EV_WRITE)
    epev.events |= EPOLL_W_FLAGS;

根据iod->watched_events的值，更新epev的events字段


  sd = nsock_iod_get_sd(iod);

调用nsock_iod_get_sd函数获取iod的sd值，并赋给sd变量


  if (epoll_ctl(einfo->epfd, EPOLL_CTL_MOD, sd, &epev) < 0)
    fatal("Unable to update events for IOD #%lu: %s", iod->id, strerror(errno));

使用epoll_ctl函数更新einfo->epfd指定的epoll实例中的sd的事件为epev指定的事件，如果失败则调用fatal函数输出错误信息


  return 1;

返回1表示操作成功


}

int epoll_loop(struct npool *nsp, int msec_timeout) {

定义一个名为epoll_loop的函数，接受npool结构体指针nsp和整型msec_timeout作为参数


  int results_left = 0;
  int event_msecs; /* msecs before an event goes off */
  int combined_msecs;
  int sock_err = 0;
  unsigned int iod_count;
  struct epoll_engine_info *einfo = (struct epoll_engine_info *)nsp->engine_data;

声明整型变量results_left，event_msecs，combined_msecs，sock_err，以及无符号整型变量iod_count，以及epoll_engine_info结构体指针einfo，并将nsp->engine_data强制转换为epoll_engine_info类型


  assert(msec_timeout >= -1);

使用断言确保msec_timeout大于等于-1


  if (nsp->events_pending == 0)
    return 0; /* No need to wait on 0 events ... */

如果nsp->events_pending等于0，则返回0，表示不需要等待事件


  iod_count = gh_list_count(&nsp->active_iods);
  if (iod_count > einfo->evlen) {
    einfo->evlen = iod_count * 2;
    einfo->events = (struct epoll_event *)safe_realloc(einfo->events, einfo->evlen * sizeof(struct epoll_event));
  }

获取nsp->active_iods链表的元素个数赋给iod_count，如果iod_count大于einfo->evlen，则将einfo->evlen更新为iod_count的两倍，并重新分配einfo->events的内存空间


  do {
    struct nevent *nse;

    nsock_log_debug_all("wait for events");

    nse = next_expirable_event(nsp);
    if (!nse)
      event_msecs = -1; /* None of the events specified a timeout */
    else
      event_msecs = MAX(0, TIMEVAL_MSEC_SUBTRACT(nse->timeout, nsock_tod));

循环执行以下操作：获取下一个将要过期的事件赋给nse，如果nse为空，则将event_msecs设置为-1，否则将event_msecs设置为nse->timeout和nsock_tod之间的差值


#if HAVE_PCAP
#ifndef PCAP_CAN_DO_SELECT

条件编译指令，根据HAVE_PCAP和PCAP_CAN_DO_SELECT的定义情况决定是否编译以下代码
    /* 在无法使用select()函数的系统上，强制设置低超时时间来捕获数据包 */
    if (gh_list_count(&nsp->pcap_read_events) > 0)  // 如果pcap读取事件列表中有事件
      if (event_msecs > PCAP_POLL_INTERVAL)  // 如果事件的毫秒数大于PCAP_POLL_INTERVAL
        event_msecs = PCAP_POLL_INTERVAL;  // 将事件的毫秒数设置为PCAP_POLL_INTERVAL
#endif
#endif

    /* We cast to unsigned because we want -1 to be very high (since it means no
     * timeout) */
    // 将变量转换为无符号整数，因为我们希望-1表示非常高的值（因为它表示没有超时）
    combined_msecs = MIN((unsigned)event_msecs, (unsigned)msec_timeout);

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
      results_left = epoll_wait(einfo->epfd, einfo->events, einfo->evlen, combined_msecs);
      if (results_left == -1)
        sock_err = socket_errno();
    }

    gettimeofday(&nsock_tod, NULL); /* Due to epoll delay */
    // 获取当前时间，由于epoll延迟
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
static inline int get_evmask(struct epoll_engine_info *einfo, int n) {
  int evmask = EV_NONE;

  if (einfo->events[n].events & EPOLL_R_FLAGS)
    evmask |= EV_READ;
  if (einfo->events[n].events & EPOLL_W_FLAGS)
    evmask |= EV_WRITE;
  if (einfo->events[n].events & EPOLL_X_FLAGS)
    evmask |= EV_EXCEPT;

  return evmask;
}

/* Iterate through all the event lists (such as connect_events, read_events,
 * timer_events, etc) and take action for those that have completed (due to
 * timeout, i/o, etc) */
// 遍历所有事件列表（如connect_events, read_events, timer_events等），并对已完成的事件（由于超时、I/O等）采取行动
void iterate_through_event_lists(struct npool *nsp, int evcount) {
  struct epoll_engine_info *einfo = (struct epoll_engine_info *)nsp->engine_data;
  int n;

  for (n = 0; n < evcount; n++) {
    struct niod *nsi = (struct niod *)einfo->events[n].data.ptr;

    assert(nsi);

    /* process all the pending events for this IOD */
    // 处理此I/O设备的所有待处理事件
    process_iod_events(nsp, nsi, get_evmask(einfo, n));
    # 如果当前 I/O 状态为已删除
    if (nsi->state == NSIOD_STATE_DELETED) {
      # 从活跃的 I/O 列表中移除当前 I/O
      gh_list_remove(&nsp->active_iods, &nsi->nodeq);
      # 将当前 I/O 放入空闲的 I/O 列表中
      gh_list_prepend(&nsp->free_iods, &nsi->nodeq);
    }
  }

  # 遍历定时器和已过期的事件
  process_expired_events(nsp);
}
# 结束条件编译指令，如果定义了 HAVE_EPOLL，则包含以下代码块
#endif /* HAVE_EPOLL */
# 结束条件编译指令，标志着条件编译的结束
```