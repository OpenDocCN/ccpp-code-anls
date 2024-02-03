# `nmap\nsock\examples\nsock_test_timers.c`

```cpp
#include "nsock.h"

#include <stdio.h>
#include <stdlib.h>
#include <sys.socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <netdb.h>
#include <unistd.h>
#include <stdlib.h>
#include <time.h>
#include <assert.h>

nsock_event_id ev_ids[2048];  // 定义一个包含2048个元素的事件ID数组

int num_ids = 0;  // 初始化事件ID数量为0

nsock_event_id request_timer(nsock_pool nsp, nsock_ev_handler handler, int timeout_msecs, void *userdata) {
  nsock_event_id id;  // 定义一个事件ID变量

  id = nsock_timer_create(nsp, handler, timeout_msecs, userdata);  // 创建一个定时器事件
  printf("%ld: Created timer ID %li for %d ms from now\n", time(NULL), id, timeout_msecs);  // 打印创建定时器事件的信息

  return id;  // 返回创建的定时器事件ID
}

int try_cancel_timer(nsock_pool nsp, int idx, int notify) {
  int res;  // 定义一个结果变量

  printf("%ld:Attempting to cancel id %li (idx %d) %s notify.\n", time(NULL), ev_ids[idx], idx, ((notify) ? "WITH" : "WITHOUT"));  // 打印尝试取消定时器事件的信息
  res = nsock_event_cancel(nsp, ev_ids[idx], notify);  // 尝试取消定时器事件
  printf("Kill of %li %s\n", ev_ids[idx], (res == 0) ? "FAILED" : "SUCCEEDED");  // 打印取消定时器事件的结果
  return res;  // 返回取消定时器事件的结果
}

void timer_handler(nsock_pool nsp, nsock_event nse, void *mydata) {
  enum nse_status status = nse_status(nse);  // 获取事件状态
  enum nse_type type = nse_type(nse);  // 获取事件类型
  int rnd, rnd2;  // 定义两个随机数变量

  printf("%ld:timer_handler: Received callback of type %s; status %s; id %li\n", time(NULL), nse_type2str(type), nse_status2str(status), nse_id(nse));  // 打印定时器处理函数的回调信息

  rnd = rand() % num_ids;  // 生成一个小于事件ID数量的随机数
  rnd2 = rand() % 3;  // 生成一个小于3的随机数

  if (num_ids > (sizeof(ev_ids) / sizeof(nsock_event_id)) - 3) {  // 如果事件ID数量超过数组长度减3
    printf("\n\nSUCCEEDED DUE TO CREATING ENOUGH EVENTS THAT IT WAS GOING TO OVERFLOW MY BUFFER :)\n\n");  // 打印成功创建足够多的事件导致溢出缓冲区的信息
    exit(0);  // 退出程序
  }

  if (status == NSE_STATUS_SUCCESS) {  // 如果事件状态为成功
    switch (rnd2) {  // 根据随机数rnd2的值进行不同的操作
    case 0:
      /* do nothing */
      /* Actually I think I'll create two timers :) */
      ev_ids[num_ids++] = request_timer(nsp, timer_handler, rand() % 3000, NULL);  // 创建两个定时器事件并将它们的ID存储到数组中
      ev_ids[num_ids++] = request_timer(nsp, timer_handler, rand() % 3000, NULL);
      break;
    case 1:
      /* Kill another id (which may or may not be active */
      try_cancel_timer(nsp, rnd, rand() % 2);  // 尝试取消另一个定时器事件
      break;
    # 如果 case 的值为 2，则执行以下操作
    case 2:
      # 创建一个新的定时器，并将其 ID 存入 ev_ids 数组中
      ev_ids[num_ids++] = request_timer(nsp, timer_handler, rand() % 3000, NULL);
      # 跳出 switch 语句
      break;
    # 如果 case 的值不为 2，则执行以下操作
    default:
      # 断言，如果程序执行到这里，说明出现了意料之外的情况
      assert(0);
    }
  }
# 主函数，接受命令行参数
int main(int argc, char *argv[]) {
  # 创建一个 nsock_pool 对象
  nsock_pool nsp;
  # 定义循环状态变量
  enum nsock_loopstatus loopret;
  # 记录循环次数
  int num_loops = 0;

  # 设置随机数种子
  srand(time(NULL));
  # 创建一个新的 nsock_pool 对象
  if ((nsp = nsock_pool_new(NULL)) == NULL) {
    # 如果创建失败，输出错误信息并退出程序
    fprintf(stderr, "Failed to create new pool.  QUITTING.\n");
    exit(1);
  }

  # 请求定时器，并将事件 ID 存入数组
  ev_ids[num_ids++] = request_timer(nsp, timer_handler, 1800, NULL);
  ev_ids[num_ids++] = request_timer(nsp, timer_handler, 800, NULL);
  ev_ids[num_ids++] = request_timer(nsp, timer_handler, 1300, NULL);
  ev_ids[num_ids++] = request_timer(nsp, timer_handler, 0, NULL);
  ev_ids[num_ids++] = request_timer(nsp, timer_handler, 100, NULL);

  # 进入循环，最多执行 5 次
  while (num_loops++ < 5) {
    # 执行 nsock_loop 函数，最长等待时间为 1500 毫秒
    loopret = nsock_loop(nsp, 1500);
    # 根据循环状态输出相应信息
    if (loopret == NSOCK_LOOP_TIMEOUT)
      printf("Finished l00p #%d due to l00p timeout :)  I may do another\n", num_loops);
    else if (loopret == NSOCK_LOOP_NOEVENTS) {
      printf("SUCCESS -- NO EVENTS LEFT\n");
      # 退出程序
      exit(0);
    } else {
      printf("nsock_loop FAILED!\n");
      # 退出程序
      exit(1);
    }
  }
  # 输出信息
  printf("Trying to kill my msp!\n");
  # 删除 nsock_pool 对象
  nsock_pool_delete(nsp);
  # 输出信息
  printf("SUCCESS -- completed %d l00ps.\n", num_loops);

  # 返回 0 表示程序正常结束
  return 0;
}
```