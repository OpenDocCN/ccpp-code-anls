# `nmap\nsock\tests\timer.c`

```cpp
/*
 * Nsock regression test suite
 * Same license as nmap -- see https://nmap.org/book/man-legal.html
 */

#include "test-common.h"
#include <time.h>

#define TIMERS_BUFFLEN  1024


struct timer_test_data {
  nsock_pool nsp;
  nsock_event_id timer_list[TIMERS_BUFFLEN];
  size_t timer_count;
  int stop; /* set to non-zero to stop the test */
};


static void timer_handler(nsock_pool nsp, nsock_event nse, void *tdata);


static void add_timer(struct timer_test_data *ttd, int timeout) {
  nsock_event_id id;

  // 创建一个定时器，并将其 ID 添加到定时器列表中
  id = nsock_timer_create(ttd->nsp, timer_handler, timeout, ttd);
  ttd->timer_list[ttd->timer_count++] = id;
}

static void timer_handler(nsock_pool nsp, nsock_event nse, void *tdata) {
  struct timer_test_data *ttd = (struct timer_test_data *)tdata;
  int rnd, rnd2;

  // 如果事件状态不是成功，则设置停止标志并返回
  if (nse_status(nse) != NSE_STATUS_SUCCESS) {
    ttd->stop = -nsock_pool_get_error(nsp);
    return;
  }

  // 如果定时器数量超过 TIMERS_BUFFLEN - 3，则直接返回
  if (ttd->timer_count > TIMERS_BUFFLEN - 3)
    return;

  // 生成随机数，并根据不同的情况执行相应的操作
  rnd = rand() % ttd->timer_count;
  rnd2 = rand() % 3;

  switch (rnd2) {
    case 0:
      /* Do nothing */
      /* Actually I think I'll create two timers :) */
      // 什么都不做，然后创建两个新的定时器
      add_timer(ttd, rand() % 3000);
      add_timer(ttd, rand() % 3000);
      break;

    case 1:
      /* Try to kill another id (which may or may not be active */
      // 尝试取消另一个 ID 的定时器（可能是活动的，也可能不是）
      nsock_event_cancel(nsp, ttd->timer_list[rnd], rand() % 2);
      break;

    case 2:
      /* Create a new timer */
      // 创建一个新的定时器
      add_timer(ttd, rand() % 3000);
      break;

    default:
      assert(0);
  }
}

static int timer_setup(void **tdata) {
  struct timer_test_data *ttd;

  srand(time(NULL));

  // 分配内存并初始化定时器测试数据结构
  ttd = calloc(1, sizeof(struct timer_test_data));
  if (ttd == NULL)
    return -ENOMEM;

  // 创建一个新的 nsock_pool 对象
  ttd->nsp = nsock_pool_new(NULL);
  AssertNonNull(ttd->nsp);

  *tdata = ttd;
  return 0;
}

static int timer_teardown(void *tdata) {
  struct timer_test_data *ttd = (struct timer_test_data *)tdata;

  if (tdata) {
    // 删除 nsock_pool 对象并释放内存
    nsock_pool_delete(ttd->nsp);
    free(tdata);
  }
  return 0;
}
# 定义一个静态函数，用于计算定时器的总消息数
static int timer_totalmess(void *tdata) {
  # 将传入的数据指针转换为 timer_test_data 结构体指针
  struct timer_test_data *ttd = (struct timer_test_data *)tdata;
  # 定义循环状态枚举变量
  enum nsock_loopstatus loopret;
  # 定义循环次数变量并初始化为 0
  int num_loops = 0;

  # 添加定时器，设置定时时间为 1800 毫秒
  add_timer(ttd, 1800);
  # 添加定时器，设置定时时间为 800 毫秒
  add_timer(ttd, 800);
  # 添加定时器，设置定时时间为 1300 毫秒
  add_timer(ttd, 1300);
  # 添加定时器，设置定时时间为 0 毫秒
  add_timer(ttd, 0);
  # 添加定时器，设置定时时间为 100 毫秒
  add_timer(ttd, 100);

  # 进入循环，最多循环 5 次且 ttd->stop 为假时执行
  while (num_loops++ < 5 && !ttd->stop) {
    # 调用 nsock_loop 函数，设置超时时间为 1500 毫秒
    loopret = nsock_loop(ttd->nsp, 1500);
    # 根据循环返回值进行不同的处理
    switch (loopret) {
      case NSOCK_LOOP_TIMEOUT:
        # 超时情况，无需处理
        /* nothing to do */
        break;

      case NSOCK_LOOP_NOEVENTS:
        # 无事件情况，返回 0
        return 0;

      default:
        # 其他情况，返回错误码
        return -(nsock_pool_get_error(ttd->nsp));
    }
  }
  # 返回 ttd->stop 的值
  return ttd->stop;
}

# 定义一个名为 TestTimer 的测试用例结构体
const struct test_case TestTimer = {
  # 测试用例名称
  .t_name     = "test timer operations",
  # 测试用例初始化函数
  .t_setup    = timer_setup,
  # 测试用例执行函数
  .t_run      = timer_totalmess,
  # 测试用例清理函数
  .t_teardown = timer_teardown
};
```