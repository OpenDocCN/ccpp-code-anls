# `nmap\nsock\src\nsock_timers.c`

```cpp
/* $Id$ */

#include "nsock_internal.h"
#include "nsock_log.h"

extern struct timeval nsock_tod;

/* Send back an NSE_TYPE_TIMER after the number of milliseconds specified.  Of
 * course it can also return due to error, cancellation, etc. */
// 创建一个定时器事件，在指定的毫秒数后发送 NSE_TYPE_TIMER 事件。当然，也可能因为错误、取消等原因返回。
nsock_event_id nsock_timer_create(nsock_pool ms_pool, nsock_ev_handler handler,
                                  int timeout_msecs, void *userdata) {
  struct npool *nsp = (struct npool *)ms_pool;
  struct nevent *nse;

  // 使用给定的毫秒池和处理程序创建一个新的事件
  nse = event_new(nsp, NSE_TYPE_TIMER, NULL, timeout_msecs, handler, userdata);
  assert(nse);

  // 记录日志，显示定时器创建的信息，包括多少毫秒后以及事件ID
  nsock_log_info("Timer created - %dms from now.  EID %li", timeout_msecs,
                 nse->id);

  // 将事件添加到毫秒池中
  nsock_pool_add_event(nsp, nse);

  // 返回事件ID
  return nse->id;
}
```