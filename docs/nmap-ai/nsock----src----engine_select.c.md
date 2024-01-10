# `nmap\nsock\src\engine_select.c`

```
/* $Id$ */

#ifndef WIN32
#include <sys/select.h>
#endif

#include <errno.h>

#include "nsock_internal.h"
#include "nsock_log.h"

#if HAVE_PCAP
#include "nsock_pcap.h"
#endif

extern struct io_operations posix_io_operations;


/* --- ENGINE INTERFACE PROTOTYPES --- */
// 初始化 select 引擎
static int select_init(struct npool *nsp);
// 销毁 select 引擎
static void select_destroy(struct npool *nsp);
// 注册 I/O 事件到 select 引擎
static int select_iod_register(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev);
// 从 select 引擎注销 I/O 事件
static int select_iod_unregister(struct npool *nsp, struct niod *iod);
// 修改 select 引擎中的 I/O 事件
static int select_iod_modify(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev_set, int ev_clr);
// 运行 select 引擎的事件循环
static int select_loop(struct npool *nsp, int msec_timeout);


/* ---- ENGINE DEFINITION ---- */
// select 引擎的定义
struct io_engine engine_select = {
  "select",
  select_init,
  select_destroy,
  select_iod_register,
  select_iod_unregister,
  select_iod_modify,
  select_loop,
  &posix_io_operations
};


/* --- INTERNAL PROTOTYPES --- */
// 遍历事件列表
static void iterate_through_event_lists(struct npool *nsp);

/* defined in nsock_core.c */
// 处理事件
void process_event(struct npool *nsp, gh_list_t *evlist, struct nevent *nse, int ev);
// 处理 I/O 事件
void process_iod_events(struct npool *nsp, struct niod *nsi, int ev);
// 处理过期事件
void process_expired_events(struct npool *nsp);

#if HAVE_PCAP
#ifndef PCAP_CAN_DO_SELECT
// 在非 select 上进行 pcap 读取
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
struct select_engine_info {
  /* Descriptors which have pending READ events */
  fd_set fds_master_r;  // 用于存储有待处理读事件的描述符集合

  /* Descriptors we are trying to WRITE to */
  fd_set fds_master_w;  // 用于存储正在尝试写入的描述符集合

  /* Looking for exceptional events -- used with connect */
  fd_set fds_master_x;  // 用于寻找异常事件的描述符集合，与连接一起使用

  /* For keeping track of the select results */
  fd_set fds_results_r, fds_results_w, fds_results_x;  // 用于跟踪 select 结果的描述符集合

  /* The highest sd we have set in any of our fd_set's (max_sd + 1 is used in
   * select() calls).  Note that it can be -1, when there are no valid sockets */
  int max_sd;  // 我们在任何 fd_set 中设置的最高描述符，select() 调用中使用 max_sd + 1。注意，当没有有效套接字时，它可以是 -1
};


int select_init(struct npool *nsp) {
  struct select_engine_info *sinfo;

  sinfo = (struct select_engine_info *)safe_malloc(sizeof(struct select_engine_info));  // 分配内存给 select_engine_info 结构体

  FD_ZERO(&sinfo->fds_master_r);  // 清空 fds_master_r 集合
  FD_ZERO(&sinfo->fds_master_w);  // 清空 fds_master_w 集合
  FD_ZERO(&sinfo->fds_master_x);  // 清空 fds_master_x 集合

  sinfo->max_sd = -1;  // 将 max_sd 初始化为 -1

  nsp->engine_data = (void *)sinfo;  // 将 sinfo 赋值给 nsp 的 engine_data

  return 1;  // 返回初始化成功
}

void select_destroy(struct npool *nsp) {
  assert(nsp->engine_data != NULL);  // 断言 engine_data 不为空
  free(nsp->engine_data);  // 释放 engine_data 的内存
}

int select_iod_register(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev) {
  assert(!IOD_PROPGET(iod, IOD_REGISTERED));  // 断言 iod 未注册

  iod->watched_events = ev;  // 设置 iod 的 watched_events 为 ev
  select_iod_modify(nsp, iod, nse, ev, EV_NONE);  // 调用 select_iod_modify 函数
  IOD_PROPSET(iod, IOD_REGISTERED);  // 设置 iod 的注册属性
  return 1;  // 返回注册成功
}

int select_iod_unregister(struct npool *nsp, struct niod *iod) {
  struct select_engine_info *sinfo = (struct select_engine_info *)nsp->engine_data;  // 获取 nsp 的 engine_data

  iod->watched_events = EV_NONE;  // 将 iod 的 watched_events 设置为 EV_NONE

  /* some IODs can be unregistered here if they're associated to an event that was
   * immediately completed */
  if (IOD_PROPGET(iod, IOD_REGISTERED)) {  // 如果 iod 已注册
#if HAVE_PCAP
    if (iod->pcap) {  // 如果 iod 有 pcap
      int sd = ((mspcap *)iod->pcap)->pcap_desc;  // 获取 pcap 的描述符
      if (sd >= 0) {  // 如果描述符大于等于 0
        checked_fd_clr(sd, &sinfo->fds_master_r);  // 清除 fds_master_r 中的描述符
        checked_fd_clr(sd, &sinfo->fds_results_r);  // 清除 fds_results_r 中的描述符
      }
    } else
#endif
    # 清除主控文件描述符集合中的读取、写入和执行权限
    checked_fd_clr(iod->sd, &sinfo->fds_master_r);
    checked_fd_clr(iod->sd, &sinfo->fds_master_w);
    checked_fd_clr(iod->sd, &sinfo->fds_master_x);
    # 清除结果文件描述符集合中的读取、写入和执行权限
    checked_fd_clr(iod->sd, &sinfo->fds_results_r);
    checked_fd_clr(iod->sd, &sinfo->fds_results_w);
    checked_fd_clr(iod->sd, &sinfo->fds_results_x);

    # 如果最大文件描述符等于当前文件描述符，则将最大文件描述符减一
    if (sinfo->max_sd == iod->sd)
      sinfo->max_sd--;

    # 清除 IOD_REGISTERED 属性
    IOD_PROPCLR(iod, IOD_REGISTERED);
  }
  # 返回 1，表示执行成功
  return 1;
}

int select_iod_modify(struct npool *nsp, struct niod *iod, struct nevent *nse, int ev_set, int ev_clr) {
  int sd;
  struct select_engine_info *sinfo = (struct select_engine_info *)nsp->engine_data;

  assert((ev_set & ev_clr) == 0);  # 确保设置和清除的事件不会同时发生

  iod->watched_events |= ev_set;  # 将设置的事件添加到已监视的事件中
  iod->watched_events &= ~ev_clr;  # 从已监视的事件中清除需要清除的事件

  ev_set |= EV_EXCEPT;  # 添加异常事件到设置的事件中
  ev_clr &= ~EV_EXCEPT;  # 从清除的事件中移除异常事件

  sd = nsock_iod_get_sd(iod);  # 获取 I/O 描述符

  /* -- set events -- */
  if (ev_set & EV_READ)
    checked_fd_set(sd, &sinfo->fds_master_r);  # 如果设置了读事件，将描述符添加到读事件的主集合中

  if (ev_set & EV_WRITE)
    checked_fd_set(sd, &sinfo->fds_master_w);  # 如果设置了写事件，将描述符添加到写事件的主集合中

  if (ev_set & EV_EXCEPT)
    checked_fd_set(sd, &sinfo->fds_master_x);  # 如果设置了异常事件，将描述符添加到异常事件的主集合中

  /* -- clear events -- */
  if (ev_clr & EV_READ)
    checked_fd_clr(sd, &sinfo->fds_master_r);  # 如果清除了读事件，将描述符从读事件的主集合中移除

  if (ev_clr & EV_WRITE)
    checked_fd_clr(sd, &sinfo->fds_master_w);  # 如果清除了写事件，将描述符从写事件的主集合中移除

  if (ev_clr & EV_EXCEPT)
    checked_fd_clr(sd, &sinfo->fds_master_x);  # 如果清除了异常事件，将描述符从异常事件的主集合中移除

  /* -- update max_sd -- */
  if (ev_set != EV_NONE)
    sinfo->max_sd = MAX(sinfo->max_sd, sd);  # 如果设置了事件，更新最大描述符
  else if (ev_clr != EV_NONE && iod->events_pending == 1 && (sinfo->max_sd == sd))
    sinfo->max_sd--;  # 如果清除了事件，并且只有一个待处理的事件，并且最大描述符是当前描述符，则减少最大描述符

  return 1;  # 返回成功
}

int select_loop(struct npool *nsp, int msec_timeout) {
  int results_left = 0;
  int event_msecs; /* msecs before an event goes off */
  int combined_msecs;
  int sock_err = 0;
  struct timeval select_tv;
  struct timeval *select_tv_p;
  struct select_engine_info *sinfo = (struct select_engine_info *)nsp->engine_data;

  assert(msec_timeout >= -1);  # 确保超时时间大于等于-1

  if (nsp->events_pending == 0)
    return 0; /* No need to wait on 0 events ... */  # 如果没有待处理的事件，直接返回0

  do {
    struct nevent *nse;

    nsock_log_debug_all("wait for events");  # 记录等待事件

    nse = next_expirable_event(nsp);  # 获取下一个将要超时的事件
    if (!nse)
      event_msecs = -1; /* None of the events specified a timeout */  # 如果没有事件指定超时时间，则设置为-1
    else
      event_msecs = MAX(0, TIMEVAL_MSEC_SUBTRACT(nse->timeout, nsock_tod));  # 计算距离下一个事件超时的时间

#if HAVE_PCAP
#ifndef PCAP_CAN_DO_SELECT
    /* Force a low timeout when capturing packets on systems where
     * the pcap descriptor is not select()able. */
    # 如果 nsp->pcap_read_events 列表的计数大于 0
    if (gh_list_count(&nsp->pcap_read_events))
      # 如果 event_msecs 大于 PCAP_POLL_INTERVAL
      if (event_msecs > PCAP_POLL_INTERVAL)
        # 将 event_msecs 设置为 PCAP_POLL_INTERVAL
        event_msecs = PCAP_POLL_INTERVAL;
#endif
#endif

    /* 将 combined_msecs 设置为 event_msecs 和 msec_timeout 中较小的一个，且转换为无符号数（因为我们希望 -1 被视为非常大的值，表示没有超时） */
    combined_msecs = MIN((unsigned)event_msecs, (unsigned)msec_timeout);

    /* 设置将传递给 select() 的 timeval 指针 */
    memset(&select_tv, 0, sizeof(select_tv));
    if (combined_msecs > 0) {
      select_tv.tv_sec = combined_msecs / 1000;
      select_tv.tv_usec = (combined_msecs % 1000) * 1000;
      select_tv_p = &select_tv;
    } else if (combined_msecs == 0) {
      /* 我们希望 tv_sec 和 tv_usec 都为零，但它们已经在 bzero 中被设置为零 */
      select_tv_p = &select_tv;
    } else {
      assert(combined_msecs == -1);
      select_tv_p = NULL;
    }

#if HAVE_PCAP
#ifndef PCAP_CAN_DO_SELECT
    /* 对不支持 select() 的 pcap 设备进行非阻塞读取
     * 如果有任何内容被读取，就离开这个循环 */
    if (pcap_read_on_nonselect(nsp)) {
      /* 好的，有内容被读取 */
    } else
#endif
#endif
    {
      /* 设置 select 的描述符 */
      sinfo->fds_results_r = sinfo->fds_master_r;
      sinfo->fds_results_w = sinfo->fds_master_w;
      sinfo->fds_results_x = sinfo->fds_master_x;

      results_left = fselect(sinfo->max_sd + 1, &sinfo->fds_results_r,
                             &sinfo->fds_results_w, &sinfo->fds_results_x, select_tv_p);

      if (results_left == -1)
        sock_err = socket_errno();
    }

    gettimeofday(&nsock_tod, NULL); /* 由于 select 延迟 */
  } while (results_left == -1 && sock_err == EINTR); /* 只有在发生信号时才重复 */

  if (results_left == -1 && sock_err != EINTR) {
    nsock_log_error("nsock_loop error %d: %s", sock_err, socket_strerror(sock_err));
    nsp->errnum = sock_err;
    return -1;
  }

  iterate_through_event_lists(nsp);

  return 1;
}


/* ---- 内部函数 ---- */
static inline int get_evmask(const struct npool *nsp, const struct niod *nsi) {
  // 获取指向 select_engine_info 结构体的指针
  struct select_engine_info *sinfo = (struct select_engine_info *)nsp->engine_data;
  int sd, evmask;

  evmask = EV_NONE;

#if HAVE_PCAP
#ifndef PCAP_CAN_DO_SELECT
  if (nsi->pcap) {
    /* 对于非阻塞读取，始终假定可读。我们无法检查 checked_fd_isset，因为我们没有 pcap_desc。 */
    evmask |= EV_READ;
    return evmask;
  }
#endif
#endif

#if HAVE_PCAP
  if (nsi->pcap)
    sd = ((mspcap *)nsi->pcap)->pcap_desc;
  else
#endif
    sd = nsi->sd;

  assert(sd >= 0);

  // 检查是否可读、可写、异常，并设置相应的事件掩码
  if (checked_fd_isset(sd, &sinfo->fds_results_r))
    evmask |= EV_READ;
  if (checked_fd_isset(sd, &sinfo->fds_results_w))
    evmask |= EV_WRITE;
  if (checked_fd_isset(sd, &sinfo->fds_results_x))
    evmask |= EV_EXCEPT;

  return evmask;
}

/* 遍历所有事件列表（如 connect_events, read_events, timer_events 等），并对已完成的事件（由于超时、I/O 等）采取行动 */
void iterate_through_event_lists(struct npool *nsp) {
  gh_lnode_t *current, *next, *last;

  last = gh_list_last_elem(&nsp->active_iods);

  for (current = gh_list_first_elem(&nsp->active_iods);
       current != NULL && gh_lnode_prev(current) != last;
       current = next) {
    struct niod *nsi = container_of(current, struct niod, nodeq);

    // 如果状态不是已删除且有待处理事件，则处理 I/O 事件
    if (nsi->state != NSIOD_STATE_DELETED && nsi->events_pending)
      process_iod_events(nsp, nsi, get_evmask(nsp, nsi));

    next = gh_lnode_next(current);
    // 如果状态是已删除，则从活动 I/O 列表中移除，并添加到空闲 I/O 列表中
    if (nsi->state == NSIOD_STATE_DELETED) {
      gh_list_remove(&nsp->active_iods, current);
      gh_list_prepend(&nsp->free_iods, current);
    }
  }

  /* 遍历定时器和已过期事件 */
  process_expired_events(nsp);
}
```