# `nmap\nsock\tests\connect.c`

```cpp
/*
 * Nsock regression test suite
 * Same license as nmap -- see https://nmap.org/book/man-legal.html
 */

#include "test-common.h"


struct connect_test_data {
  nsock_pool nsp;  // 定义 nsock_pool 结构体变量
  nsock_iod  nsi;  // 定义 nsock_iod 结构体变量
  int connect_result;  // 定义连接结果变量
};


static void connect_handler(nsock_pool nsp, nsock_event nse, void *udata) {
  struct connect_test_data *ctd;  // 定义 connect_test_data 结构体指针变量

  ctd = (struct connect_test_data *)nsock_pool_get_udata(nsp);  // 获取 nsock_pool 中的用户数据

  switch(nse_status(nse)) {  // 根据事件状态进行处理
    case NSE_STATUS_SUCCESS:  // 连接成功
      ctd->connect_result = 0;  // 连接结果为 0
      break;

    case NSE_STATUS_ERROR:  // 连接出错
      ctd->connect_result = -(nse_errorcode(nse));  // 连接结果为错误码的负值
      break;

    case NSE_STATUS_TIMEOUT:  // 连接超时
      ctd->connect_result = -ETIMEDOUT;  // 连接结果为超时错误码
      break;

    default:  // 其他情况
      ctd->connect_result = -EINVAL;  // 连接结果为无效参数错误码
      break;
  }
}

static int connect_setup(void **tdata) {
  struct connect_test_data *ctd;  // 定义 connect_test_data 结构体指针变量

  ctd = calloc(1, sizeof(struct connect_test_data));  // 分配内存空间
  if (ctd == NULL)
    return -ENOMEM;  // 分配内存失败

  ctd->nsp = nsock_pool_new(ctd);  // 创建新的 nsock_pool 对象
  AssertNonNull(ctd->nsp);  // 断言 nsock_pool 对象非空

  ctd->nsi = nsock_iod_new(ctd->nsp, NULL);  // 创建新的 nsock_iod 对象
  AssertNonNull(ctd->nsi);  // 断言 nsock_iod 对象非空

  *tdata = ctd;  // 将 ctd 赋值给 tdata
  return 0;
}

static int connect_teardown(void *tdata) {
  struct connect_test_data *ctd = (struct connect_test_data *)tdata;  // 定义 connect_test_data 结构体指针变量

  if (tdata) {
    nsock_iod_delete(ctd->nsi, NSOCK_PENDING_SILENT); /* nsock_pool_delete would also handle it */
    nsock_pool_delete(ctd->nsp);  // 删除 nsock_pool 对象
    free(tdata);  // 释放内存空间
  }
  return 0;
}

static int connect_tcp(void *tdata) {
  struct connect_test_data *ctd = (struct connect_test_data *)tdata;  // 定义 connect_test_data 结构体指针变量
  struct sockaddr_in peer;  // 定义 sockaddr_in 结构体变量

  memset(&peer, 0, sizeof(peer));  // 将 peer 结构体变量清零
  peer.sin_family = AF_INET;  // 设置地址族为 IPv4
  inet_aton("127.0.0.1", &peer.sin_addr);  // 将字符串形式的 IPv4 地址转换为二进制形式

  nsock_connect_tcp(ctd->nsp, ctd->nsi, connect_handler, 4000, NULL,
                    (struct sockaddr *)&peer, sizeof(peer), PORT_TCP);  // 发起 TCP 连接请求

  nsock_loop(ctd->nsp, 4000);  // 运行 nsock 事件循环
  return ctd->connect_result;  // 返回连接结果
}
static int connect_tcp_failure(void *tdata) {
  // 将传入的数据转换为 connect_test_data 结构体指针
  struct connect_test_data *ctd = (struct connect_test_data *)tdata;
  // 创建一个用于存储对端地址信息的 sockaddr_in 结构体
  struct sockaddr_in peer;

  // 将 peer 结构体清零
  memset(&peer, 0, sizeof(peer));
  // 设置 peer 结构体的地址族为 AF_INET
  peer.sin_family = AF_INET;
  // 将字符串形式的 IP 地址转换为二进制形式，并存储在 peer 结构体中
  inet_aton("127.0.0.1", &peer.sin_addr);

  /* pass in addrlen == 0 to force connect(2) to fail */
  // 调用 nsock_connect_tcp 函数，传入参数，强制 connect(2) 失败
  nsock_connect_tcp(ctd->nsp, ctd->nsi, connect_handler, 4000, NULL,
                    (struct sockaddr *)&peer, 0, PORT_TCP);

  // 运行事件循环，等待连接结果
  nsock_loop(ctd->nsp, 4000);
  // 断言连接结果为 -EINVAL
  AssertEqual(ctd->connect_result, -EINVAL);
  // 返回 0
  return 0;
}


const struct test_case TestConnectTCP = {
  .t_name     = "simple tcp connection",
  .t_setup    = connect_setup,
  .t_run      = connect_tcp,
  .t_teardown = connect_teardown
};

const struct test_case TestConnectFailure = {
  .t_name     = "tcp connection failure case",
  .t_setup    = connect_setup,
  .t_run      = connect_tcp_failure,
  .t_teardown = connect_teardown
};
```