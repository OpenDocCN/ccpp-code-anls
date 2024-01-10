# `nmap\nsock\tests\cancel.c`

```
/*
 * Nsock回归测试套件
 * 与nmap具有相同的许可证--请参阅https://nmap.org/book/man-legal.html
 */

#include "test-common.h"

// 定义基本测试数据结构
struct basic_test_data {
  nsock_pool nsp;
};

// 取消处理程序，用于处理取消事件
static void cancel_handler(nsock_pool nsp, nsock_event nse, void *udata) {
  int *ev_done = (int *)udata;

  // 如果事件状态为取消，则设置ev_done为1
  if (nse_status(nse) == NSE_STATUS_CANCELLED)
    *ev_done = 1;
}

// 取消设置，用于初始化测试数据
static int cancel_setup(void **tdata) {
  struct basic_test_data *btd;

  // 分配内存以存储基本测试数据
  btd = calloc(1, sizeof(struct basic_test_data));
  if (btd == NULL)
    return -ENOMEM;

  // 创建新的nsock池
  btd->nsp = nsock_pool_new(NULL);

  *tdata = btd;
  return 0;
}

// 取消拆卸，用于清理测试数据
static int cancel_teardown(void *tdata) {
  struct basic_test_data *btd = (struct basic_test_data *)tdata;

  if (tdata) {
    // 删除nsock池并释放内存
    nsock_pool_delete(btd->nsp);
    free(tdata);
  }
  return 0;
}

// 取消TCP运行，用于执行TCP取消测试
static int cancel_tcp_run(void *tdata) {
  struct basic_test_data *btd = (struct basic_test_data *)tdata;
  struct sockaddr_in peer;
  nsock_iod iod;
  nsock_event_id id;
  int done = 0;

  // 创建新的nsock I/O数据结构
  iod = nsock_iod_new(btd->nsp, NULL);
  AssertNonNull(iod);

  // 初始化peer结构
  memset(&peer, 0, sizeof(peer));
  peer.sin_family = AF_INET;
  inet_aton("127.0.0.1", &peer.sin_addr);

  // 发起TCP连接，并设置取消处理程序和超时时间
  id = nsock_connect_tcp(btd->nsp, iod, cancel_handler, 4000, (void *)&done,
                         (struct sockaddr *)&peer, sizeof(peer), PORT_TCP);
  // 取消TCP连接事件
  nsock_event_cancel(btd->nsp, id, 1);

  // 删除nsock I/O数据结构
  nsock_iod_delete(iod, NSOCK_PENDING_SILENT);

  // 根据done值判断测试结果
  return (done == 1) ? 0 : -ENOEXEC;
}
# 取消 UDP 运行的函数
static int cancel_udp_run(void *tdata) {
  # 将传入的数据转换为基本测试数据结构
  struct basic_test_data *btd = (struct basic_test_data *)tdata;
  # 创建一个新的 I/O 数据结构
  struct sockaddr_in peer;
  nsock_iod iod;
  nsock_event_id id;
  int done = 0;

  # 使用网络套接字库创建一个新的 I/O 数据结构
  iod = nsock_iod_new(btd->nsp, NULL);
  # 断言 I/O 数据结构非空
  AssertNonNull(iod);

  # 将 peer 结构体清零，并设置地址族为 AF_INET
  memset(&peer, 0, sizeof(peer));
  peer.sin_family = AF_INET;
  # 将字符串形式的 IP 地址转换为二进制形式，并存储在 peer.sin_addr 中
  inet_aton("127.0.0.1", &peer.sin_addr);

  # 使用网络套接字库创建一个 UDP 连接事件，并返回事件 ID
  id = nsock_connect_udp(btd->nsp, iod, cancel_handler, (void *)&done,
                         (struct sockaddr *)&peer, sizeof(peer), PORT_UDP);
  # 取消指定的事件
  nsock_event_cancel(btd->nsp, id, 1);

  # 删除 I/O 数据结构
  nsock_iod_delete(iod, NSOCK_PENDING_SILENT);

  # 如果 done 等于 1，则返回 0，否则返回 -ENOEXEC
  return (done == 1) ? 0 : -ENOEXEC;
}

# 取消 SSL 运行的函数
static int cancel_ssl_run(void *tdata) {
  # 将传入的数据转换为基本测试数据结构
  struct basic_test_data *btd = (struct basic_test_data *)tdata;
  # 创建一个新的 I/O 数据结构
  struct sockaddr_in peer;
  nsock_iod iod;
  nsock_event_id id;
  int done = 0;

  # 使用网络套接字库创建一个新的 I/O 数据结构
  iod = nsock_iod_new(btd->nsp, NULL);
  # 断言 I/O 数据结构非空
  AssertNonNull(iod);

  # 将 peer 结构体清零，并设置地址族为 AF_INET
  memset(&peer, 0, sizeof(peer));
  peer.sin_family = AF_INET;
  # 将字符串形式的 IP 地址转换为二进制形式，并存储在 peer.sin_addr 中
  inet_aton("127.0.0.1", &peer.sin_addr);

  # 使用网络套接字库创建一个 SSL 连接事件，并返回事件 ID
  id = nsock_connect_ssl(btd->nsp, iod, cancel_handler, 4000, (void *)&done,
                         (struct sockaddr *)&peer, sizeof(peer), IPPROTO_TCP,
                         PORT_TCPSSL, NULL);
  # 取消指定的事件
  nsock_event_cancel(btd->nsp, id, 1);

  # 删除 I/O 数据结构
  nsock_iod_delete(iod, NSOCK_PENDING_SILENT);

  # 如果 done 等于 1，则返回 0，否则返回 -ENOEXEC
  return (done == 1) ? 0 : -ENOEXEC;
}

# 取消 TCP 运行的测试用例
const struct test_case TestCancelTCP = {
  .t_name     = "schedule and cancel TCP connect",
  .t_setup    = cancel_setup,
  .t_run      = cancel_tcp_run,
  .t_teardown = cancel_teardown
};

# 取消 UDP 运行的测试用例
const struct test_case TestCancelUDP = {
  .t_name     = "schedule and cancel UDP pseudo-connect",
  .t_setup    = cancel_setup,
  .t_run      = cancel_udp_run,
  .t_teardown = cancel_teardown
};

# 取消 SSL 运行的测试用例
const struct test_case TestCancelSSL = {
  .t_name     = "schedule and cancel SSL connect",
  .t_setup    = cancel_setup,
  .t_run      = cancel_ssl_run,
  .t_teardown = cancel_teardown
};
```