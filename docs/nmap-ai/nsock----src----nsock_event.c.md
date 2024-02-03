# `nmap\nsock\src\nsock_event.c`

```cpp
/* $Id$ */

#include "nsock_internal.h"  // 包含 nsock_internal.h 头文件
#include "nsock_log.h"  // 包含 nsock_log.h 头文件
#include "gh_list.h"  // 包含 gh_list.h 头文件

#if HAVE_PCAP
#include "nsock_pcap.h"  // 如果定义了 HAVE_PCAP，则包含 nsock_pcap.h 头文件
#endif

#include <string.h>  // 包含 string.h 头文件

extern struct timeval nsock_tod;  // 声明 nsock_tod 结构体变量的外部链接

/* Find the type of an event that spawned a callback */
enum nse_type nse_type(nsock_event nse) {  // 定义函数 nse_type，返回值类型为 enum nse_type，参数为 nsock_event nse
  struct nevent *me = (struct nevent *)nse;  // 将 nse 强制转换为 struct nevent 类型，并赋值给 me
  return me->type;  // 返回 me 的 type 成员
}

enum nse_status nse_status(nsock_event nse) {  // 定义函数 nse_status，返回值类型为 enum nse_status，参数为 nsock_event nse
  struct nevent *me = (struct nevent *)nse;  // 将 nse 强制转换为 struct nevent 类型，并赋值给 me
  return me->status;  // 返回 me 的 status 成员
}

int nse_eof(nsock_event nse) {  // 定义函数 nse_eof，返回值类型为 int，参数为 nsock_event nse
  struct nevent *me = (struct nevent *)nse;  // 将 nse 强制转换为 struct nevent 类型，并赋值给 me
  return me->eof;  // 返回 me 的 eof 成员
}

/* Obtains the nsock_iod (see below) associated with the event.  Note that
 * some events (such as timers) don't have an nsock_iod associated with them */
nsock_iod nse_iod(nsock_event ms_event) {  // 定义函数 nse_iod，返回值类型为 nsock_iod，参数为 nsock_event ms_event
  struct nevent *nse = (struct nevent *)ms_event;  // 将 ms_event 强制转换为 struct nevent 类型，并赋值给 nse
  return (nsock_iod) nse->iod;  // 返回 nse 的 iod 成员
}

/* This next function returns the errno style error code -- which is only valid
 * if the status is NSE_STATUS_ERROR */
int nse_errorcode(nsock_event nse) {  // 定义函数 nse_errorcode，返回值类型为 int，参数为 nsock_event nse
  struct nevent *me = (struct nevent *)nse;  // 将 nse 强制转换为 struct nevent 类型，并赋值给 me
  return me->errnum;  // 返回 me 的 errnum 成员
}

/* Every event has an ID which will be unique throughout the program's execution
 * unless you use (literally) billions of them */
nsock_event_id nse_id(nsock_event nse) {  // 定义函数 nse_id，返回值类型为 nsock_event_id，参数为 nsock_event nse
  struct nevent *me = (struct nevent *)nse;  // 将 nse 强制转换为 struct nevent 类型，并赋值给 me
  return me->id;  // 返回 me 的 id 成员
}

/* If you did a read request, and the result was STATUS_SUCCESS, this function
 * provides the buffer that was read in as well as the number of chars read.
 * The buffer should not be modified or free'd */
char *nse_readbuf(nsock_event nse, int *nbytes) {  // 定义函数 nse_readbuf，返回值类型为 char*，参数为 nsock_event nse 和 int* nbytes
  struct nevent *me = (struct nevent *)nse;  // 将 nse 强制转换为 struct nevent 类型，并赋值给 me

  if (nbytes)  // 如果 nbytes 不为 NULL
    *nbytes = fs_length(&(me->iobuf));  // 将 me 的 iobuf 成员的长度赋值给 *nbytes
  return fs_str(&(me->iobuf));  // 返回 me 的 iobuf 成员的字符串
}

static void first_ev_next(struct nevent *nse, gh_lnode_t **first, int nodeq2) {  // 定义静态函数 first_ev_next，返回类型为 void，参数为 struct nevent *nse, gh_lnode_t **first, int nodeq2
  if (!first || !*first)  // 如果 first 为 NULL 或者 *first 为 NULL
    return;  // 返回

  if (&nse->nodeq_io == *first || &nse->nodeq_pcap == *first) {  // 如果 nse 的 nodeq_io 或者 nodeq_pcap 成员等于 *first
    gh_lnode_t *next;  // 声明 gh_lnode_t 类型的指针变量 next

    next = gh_lnode_next(*first);  // 将 *first 的下一个节点赋值给 next
    # 如果存在下一个事件节点
    if (next) {
      # 声明一个新的事件结构体指针
      struct nevent *newevent;

      # 如果存在第二个事件队列
      if (nodeq2)
        # 将下一个节点转换为第二个事件队列的事件
        newevent = lnode_nevent2(next);
      else
        # 将下一个节点转换为事件
        newevent = lnode_nevent(next);

      # 如果新事件的 IOD 等于当前节点的 IOD
      if (newevent->iod == nse->iod)
        # 将当前节点指向下一个节点
        *first = next;
      else
        # 将当前节点指向空
        *first = NULL;
    } else {
      # 如果不存在下一个节点，则将当前节点指向空
      *first = NULL;
    }
  }
}

void update_first_events(struct nevent *nse) {
  // 根据事件ID类型进行不同的处理
  switch (get_event_id_type(nse->id)) {
    // 如果是连接事件或SSL连接事件，调用first_ev_next函数处理
    case NSE_TYPE_CONNECT:
    case NSE_TYPE_CONNECT_SSL:
      first_ev_next(nse, &nse->iod->first_connect, 0);
      break;

    // 如果是读事件，调用first_ev_next函数处理
    case NSE_TYPE_READ:
      first_ev_next(nse, &nse->iod->first_read, 0);
      break;

    // 如果是写事件，调用first_ev_next函数处理
    case NSE_TYPE_WRITE:
      first_ev_next(nse, &nse->iod->first_write, 0);
      break;

    // 如果支持PCAP，且是PCAP读事件，调用first_ev_next函数处理
#if HAVE_PCAP
    case NSE_TYPE_PCAP_READ:
      first_ev_next(nse, &nse->iod->first_read, 0);
      first_ev_next(nse, &nse->iod->first_pcap_read, 1);
      break;
#endif

    // 如果是定时器事件，不做任何处理
    case NSE_TYPE_TIMER:
      /* nothing to do */
      break;

    // 其他情况报错
    default:
      fatal("Bogus event type in update_first_events");
      break;
  }
}

/* 取消事件（如定时器或读请求）。如果notify为非零，请求者将收到一个事件取消状态的通知，发送到给定的处理程序。
 * 但在某些情况下，这是没有必要的（比如如果删除它的函数是创建它的函数），在这种情况下，可以传递0来跳过这一步。
 * 如果事件未找到，则此函数返回零，否则返回非零。 */
int nsock_event_cancel(nsock_pool ms_pool, nsock_event_id id, int notify) {
  struct npool *nsp = (struct npool *)ms_pool;
  enum nse_type type;
  unsigned int i;
  gh_list_t *event_list = NULL, *event_list2 = NULL;
  gh_lnode_t *current, *next;
  struct nevent *nse = NULL;

  assert(nsp);

  type = get_event_id_type(id);
  nsock_log_info("Event #%li (type %s) cancelled", id, nse_type2str(type));

  /* 首先确定事件所在的列表 */
  switch (type) {
    // 如果是连接事件或SSL连接事件，设置event_list为nsp->connect_events
    case NSE_TYPE_CONNECT:
    case NSE_TYPE_CONNECT_SSL:
      event_list = &nsp->connect_events;
      break;

    // 如果是读事件，设置event_list为nsp->read_events
    case NSE_TYPE_READ:
      event_list = &nsp->read_events;
      break;

    // 如果是写事件，设置event_list为nsp->write_events
    case NSE_TYPE_WRITE:
      event_list = &nsp->write_events;
      break;
    # 如果事件类型是定时器
    case NSE_TYPE_TIMER:
      # 遍历所有即将过期的事件
      for (i = 0; i < gh_heap_count(&nsp->expirables); i++) {
        # 获取即将过期的事件的节点
        gh_hnode_t *hnode;
        hnode = gh_heap_find(&nsp->expirables, i);
        # 将节点转换为事件结构
        nse = container_of(hnode, struct nevent, expire);
        # 如果找到了指定 id 的事件，则删除该事件并返回结果
        if (nse->id == id)
          return nevent_delete(nsp, nse, NULL, NULL, notify);
      }
      # 如果没有找到指定 id 的事件，则返回 0
      return 0;
#if HAVE_PCAP
    // 如果定义了 HAVE_PCAP，则执行以下代码块
    case NSE_TYPE_PCAP_READ:
      // 设置 event_list 指向 nsp 的 read_events
      event_list  = &nsp->read_events;
      // 设置 event_list2 指向 nsp 的 pcap_read_events
      event_list2 = &nsp->pcap_read_events;
      // 跳出 switch 语句
      break;
#endif

    // 默认情况
    default:
      // 输出错误信息并终止程序
      fatal("Bogus event type in nsock_event_cancel"); break;
  }

  /* 现在我们尝试在列表中找到事件 */
  // 遍历 event_list 列表
  for (current = gh_list_first_elem(event_list); current != NULL; current = next) {
    next = gh_lnode_next(current);
    // 获取当前节点对应的事件
    nse = lnode_nevent(current);
    // 如果事件的 id 与给定 id 相等，则跳出循环
    if (nse->id == id)
      break;
  }

  // 如果 current 为 NULL 并且 event_list2 不为空
  if (current == NULL && event_list2) {
    // 将 event_list 指向 event_list2
    event_list = event_list2;
    // 再次遍历 event_list 列表
    for (current = gh_list_first_elem(event_list); current != NULL; current = next) {
      next = gh_lnode_next(current);
      // 获取当前节点对应的事件
      nse = lnode_nevent2(current);
      // 如果事件的 id 与给定 id 相等，则跳出循环
      if (nse->id == id)
        break;
    }
  }
  // 如果 current 为 NULL，则返回 0
  if (current == NULL)
    return 0;

  // 调用 nevent_delete 函数删除事件
  return nevent_delete(nsp, nse, event_list, current, notify);
}

/* 一个用于取消事件的内部函数，当你已经有一个指向结构 nevent 的指针时使用
 * （如果你只有一个 ID，则使用 nsock_event_cancel）。传入的 event_list 应该对应于事件的类型。
 * 例如，对于 NSE_TYPE_READ，你应该传入 &nsp->read_events;。elem 是 event_list 中持有事件的列表元素。
 * 如果你希望拥有事件的程序被通知事件已被取消，则传入一个非零值给 notify */
int nevent_delete(struct npool *nsp, struct nevent *nse, gh_list_t *event_list,
                   gh_lnode_t *elem, int notify) {
  // 如果事件已经被标记为即将删除，则返回 0
  if (nse->event_done) {
    return 0;
  }

  // 输出信息，标记事件即将被取消
  nsock_log_info("%s on event #%li (type %s)", __func__, nse->id,
                 nse_type2str(nse->type));

  /* 现在我们找到了事件...我们按照正常流程取消它 */
  // 根据事件类型执行相应的操作
  switch (nse->type) {
    case NSE_TYPE_CONNECT:
    # 如果是 SSL 连接类型，处理连接结果为取消状态
    case NSE_TYPE_CONNECT_SSL:
      handle_connect_result(nsp, nse, NSE_STATUS_CANCELLED);
      break;

    # 如果是读取类型，处理读取结果为取消状态
    case NSE_TYPE_READ:
      handle_read_result(nsp, nse, NSE_STATUS_CANCELLED);
      break;

    # 如果是写入类型，处理写入结果为取消状态
    case NSE_TYPE_WRITE:
      handle_write_result(nsp, nse, NSE_STATUS_CANCELLED);
      break;

    # 如果是定时器类型，处理定时器结果为取消状态
    case NSE_TYPE_TIMER:
      handle_timer_result(nsp, nse, NSE_STATUS_CANCELLED);
      break;
#if HAVE_PCAP
    // 如果定义了 HAVE_PCAP，则执行以下代码块
    case NSE_TYPE_PCAP_READ:
      // 如果事件类型为 NSE_TYPE_PCAP_READ，则调用 handle_pcap_read_result 函数，传入参数 nsp, nse, NSE_STATUS_CANCELLED
      handle_pcap_read_result(nsp, nse, NSE_STATUS_CANCELLED);
      // 跳出 switch 语句
      break;
#endif

    // 默认情况
    default:
      // 抛出致命错误，提示无效的 nsock 事件类型
      fatal("Invalid nsock event type (%d)", nse->type);
  }

  // 断言事件已完成
  assert(nse->event_done);

  // 如果超时时间大于 0 秒
  if (nse->timeout.tv_sec)
    // 从 expirables 堆中移除 nse->expire
    gh_heap_remove(&nsp->expirables, &nse->expire);

  // 如果 event_list 不为空
  if (event_list) {
    // 更新首个事件
    update_first_events(nse);
    // 从 event_list 中移除 elem
    gh_list_remove(event_list, elem);
  }

  // 将 nse->nodeq_io 追加到 free_events 列表中
  gh_list_append(&nsp->free_events, &nse->nodeq_io);

  // 记录调试日志，提示正在从列表中移除事件
  nsock_log_debug_all("NSE #%lu: Removing event from list", nse->id);

#if HAVE_PCAP
#if PCAP_BSD_SELECT_HACK
  // 如果定义了 HAVE_PCAP 和 PCAP_BSD_SELECT_HACK
  if (nse->type == NSE_TYPE_PCAP_READ) {
    // 记录调试日志，提示取消测试
    nsock_log_debug_all("PCAP NSE #%lu: CANCEL TEST pcap=%p read=%p curr=%p sd=%i",
                        nse->id, &nsp->pcap_read_events, &nsp->read_events,
                        event_list,((mspcap *)nse->iod->pcap)->pcap_desc);

    /* 如果事件发生，并且我们处于 BSD_HACK 模式，则此事件已添加到两个队列中。
     * read_event 和 pcap_read_event，当然我们应该只销毁一次。
     * 我们假设现在处于 read_event，所以只需从 pcap_read_event 中取消链接此事件 */
    if (((mspcap *)nse->iod->pcap)->pcap_desc >= 0 && event_list == &nsp->read_events) {
      // 事件已完成，列表为 read_events，我们处于 BSD_HACK 模式。因此从 pcap_read_events 中取消链接事件
      gh_list_remove(&nsp->pcap_read_events, &nse->nodeq_pcap);
      // 记录调试日志，提示从 PCAP_READ_EVENTS 中移除事件
      nsock_log_debug_all("PCAP NSE #%lu: Removing event from PCAP_READ_EVENTS", nse->id);
    }

    if (((mspcap *)nse->iod->pcap)->pcap_desc >= 0 && event_list == &nsp->pcap_read_events) {
      // 事件已完成，列表为 read_events，我们处于 BSD_HACK 模式。因此从 read_events 中取消链接事件
      gh_list_remove(&nsp->read_events, &nse->nodeq_io);
      // 记录调试日志，提示从 READ_EVENTS 中移除事件
      nsock_log_debug_all("PCAP NSE #%lu: Removing event from READ_EVENTS", nse->id);
    }
  }
#endif
#endif
  // 调用 event_dispatch_and_delete 函数，传入参数 nsp, nse, notify，并返回 1
  event_dispatch_and_delete(nsp, nse, notify);
  // 返回 1
  return 1;
}
/* 调整各种统计数据，分派事件处理程序（如果 notify 不为零），然后删除事件。此函数不会从任何可能存在的列表中删除事件（例如 nsp->read_list 等）。在调用此函数时，nse->event_done 必须为 true */
void event_dispatch_and_delete(struct npool *nsp, struct nevent *nse, int notify) {
  assert(nsp);
  assert(nse);

  assert(nse->event_done);

  nsp->events_pending--;
  assert(nsp->events_pending >= 0);

  if (nse->iod) {
    nse->iod->events_pending--;
    assert(nse->iod->events_pending >= 0);
  }

  if (notify) {
    nsock_trace_handler_callback(nsp, nse);
    nse->handler(nsp, nse, nse->userdata);
  }

  /* FIXME: We should be updating stats here ... */

  /* 现在我们破坏事件... */
  event_delete(nsp, nse);
}

/* 好吧 - 我们的想法是，我们希望类型包含在最右边的两位中，而序列号在最左边的 30 或 62 位中。但是，我们还希望确保在事件数量过多的情况下正确的环绕。一个“正确”环绕的定义是，它从最高数值回到一（而不是零），因为我们不希望事件编号永远为零。 */
nsock_event_id get_new_event_id(struct npool *ms, enum nse_type type) {
  int type_code = (int)type;
  unsigned long serial = ms->next_event_serial++;
  unsigned long max_serial_allowed;
  int shiftbits;

  assert(type < NSE_TYPE_MAX);

  shiftbits = sizeof(nsock_event_id) * 8 - TYPE_CODE_NUM_BITS;
  max_serial_allowed = ((unsigned long)1 << shiftbits) - 1;
  if (serial == max_serial_allowed) {
    /* 那么下一个序列号将为一，因为零是被禁止的 */
    ms->next_event_serial = 1;
  }

  return (serial << TYPE_CODE_NUM_BITS) | type_code;
}

/* 获取事件 ID 并返回类型（NSE_TYPE_CONNECT 等） */
enum nse_type get_event_id_type(nsock_event_id event_id) {
  return (enum nse_type)((event_id & ((1 << TYPE_CODE_NUM_BITS) - 1)));
}
/* 创建一个新的事件结构体 -- 必须使用 event_delete 函数在使用完后删除，除非返回 NULL（表示失败）。如果 struct niod 和 userdata 不可用，可以传入 NULL */
struct nevent *event_new(struct npool *nsp, enum nse_type type,
                           struct niod *iod, int timeout_msecs,
                           nsock_ev_handler handler, void *userdata) {
  struct nevent *nse;
  gh_lnode_t *lnode;

  /* 获取当前时间，用于计算超时 */
  gettimeofday(&nsock_tod, NULL);

  if (iod) {
    iod->events_pending++;
    assert(iod->state != NSIOD_STATE_DELETED);
  }

  /* 首先检查是否有可用的事件结构体在空闲列表中... */
  lnode = gh_list_pop(&nsp->free_events);
  if (!lnode)
    nse = (struct nevent *)safe_malloc(sizeof(*nse));
  else
    nse = lnode_nevent(lnode);

  memset(nse, 0, sizeof(*nse));

  nse->id = get_new_event_id(nsp, type);
  nse->type = type;
  nse->status = NSE_STATUS_NONE;
  gh_hnode_invalidate(&nse->expire);
#if HAVE_OPENSSL
  nse->sslinfo.ssl_desire = SSL_ERROR_NONE;
#endif

  if (type == NSE_TYPE_READ || type ==  NSE_TYPE_WRITE)
    filespace_init(&(nse->iobuf), 1024);

#if HAVE_PCAP
  if (type == NSE_TYPE_PCAP_READ) {
    mspcap *mp;
    int sz;

    assert(iod != NULL);
    mp = (mspcap *)iod->pcap;
    assert(mp);

    sz = mp->snaplen+1 + sizeof(nsock_pcap);
    filespace_init(&(nse->iobuf), sz);
  }
#endif

  if (timeout_msecs != -1) {
    assert(timeout_msecs >= 0);
    TIMEVAL_MSEC_ADD(nse->timeout, nsock_tod, timeout_msecs);
  }

  nse->iod = iod;
  nse->handler = handler;
  nse->userdata = userdata;

  if (nse->iod == NULL)
    nsock_log_debug("%s (IOD #NULL) (EID #%li)", __func__, nse->id);
  else
    nsock_log_debug("%s (IOD #%li) (EID #%li)", __func__, nse->iod->id,
                    nse->id);
  return nse;
}
/* 释放使用 event_new 分配的 struct nevent 结构，包括所有内部资源。
 * 注意 - 我们假设 nse->iod->events_pending（如果存在）已经被递减（在 event_dispatch_and_delete 中完成） - 
 * 所以如果直接调用 event_delete()，请记得这样做 */
void event_delete(struct npool *nsp, struct nevent *nse) {
  if (nse->iod == NULL)
    nsock_log_debug("%s (IOD #NULL) (EID #%li)", __func__, nse->id);
  else
    nsock_log_debug("%s (IOD #%li) (EID #%li)", __func__, nse->iod->id, nse->id);

  /* 首先，如果需要，释放其中的 IOBuf */
  if (nse->type == NSE_TYPE_READ || nse->type ==  NSE_TYPE_WRITE) {
    fs_free(&nse->iobuf);
  }
  #if HAVE_PCAP
  if (nse->type == NSE_TYPE_PCAP_READ) {
    fs_free(&nse->iobuf);
    nsock_log_debug_all("PCAP removed %lu", nse->id);
  }
  #endif

  /* 现在将事件重新添加到空闲池中 */
  nse->event_done = 1;
}

/* 获取 nse_type（由 nse_type() 返回）并返回一个静态字符串名称，可用于打印等 */
const char *nse_type2str(enum nse_type type) {
  switch (type) {
    case NSE_TYPE_CONNECT: return "CONNECT";
    case NSE_TYPE_CONNECT_SSL: return "SSL-CONNECT";
    case NSE_TYPE_READ: return "READ";
    case NSE_TYPE_WRITE: return "WRITE";
    case NSE_TYPE_TIMER: return "TIMER";
    case NSE_TYPE_PCAP_READ: return "READ-PCAP";
    default:
      return "UNKNOWN!";
  }
}

/* 获取 nse_status（由 nse_status() 返回）并返回一个静态字符串名称，可用于打印等 */
const char *nse_status2str(enum nse_status status) {
  switch (status) {
    case NSE_STATUS_NONE: return "NONE";
    case NSE_STATUS_SUCCESS: return "SUCCESS";
    case NSE_STATUS_ERROR: return "ERROR";
    case NSE_STATUS_TIMEOUT: return "TIMEOUT";
    case NSE_STATUS_CANCELLED: return "CANCELLED";
    case NSE_STATUS_KILL: return "KILL";
    case NSE_STATUS_EOF: return "EOF";
    case NSE_STATUS_PROXYERROR: return "PROXY ERROR";
  }
}
    # 如果没有匹配的 case，则返回 "UNKNOWN!"
    default:
      return "UNKNOWN!";
  }
# 判断事件是否超时的函数
int event_timedout(struct nevent *nse) {
  # 如果事件已经完成，则返回0
  if (nse->event_done)
    return 0;
  
  # 如果设置了超时时间，并且当前时间已经超过了超时时间，则返回1，否则返回0
  return (nse->timeout.tv_sec && !TIMEVAL_AFTER(nse->timeout, nsock_tod));
}
```