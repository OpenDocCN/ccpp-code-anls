# `nmap\nsock\tests\proxychain.c`

```
/*
 * Nsock regression test suite
 * Same license as nmap -- see https://nmap.org/book/man-legal.html
 */

#include "test-common.h"
#include "../src/nsock_log.h"

struct proxy_test_data {
  int tn;
  nsock_pool nsp;
};
static struct proxy_test_data *GlobalTD;

#define END_OF_TESTS -1
#define GOOD 0
#define BAD 1
struct proxy_test {
  int ttype;
  const char *input;
};

};

static int parser_test(void *testdata) {
  int result = 0;
  struct proxy_test_data *ptd = (struct proxy_test_data *)testdata;
  const struct proxy_test *pt = &Tests[ptd->tn];
  while (pt->ttype != END_OF_TESTS) {
    nsock_proxychain pxc = NULL;
    if (pt->ttype == BAD)
      nsock_log_info("Expected failure:");
    int ret = nsock_proxychain_new(pt->input, &pxc, NULL);
    nsock_log_debug("Test %d result: %d", ptd->tn, ret);
    if (ret > 0) {
      if (pt->ttype == BAD) {
        fprintf(stderr, "Proxy Test #%d: Failed to reject bad input: %s\n", ptd->tn, pt->input);
        result = -1;
      }
      nsock_proxychain_delete(pxc);
    }
    else if (ret <= 0 && pt->ttype == GOOD) {
      fprintf(stderr, "Proxy Test #%d: Failed to parse good input: %s\n", ptd->tn, pt->input);
      result = -2;
    }
    ptd->tn++;
    pt = &Tests[ptd->tn];
  }
  return result;
}

static void log_handler(const struct nsock_log_rec *rec) {
  /* Only print log messages if we expect the test to succeed. */
  if (Tests[GlobalTD->tn].ttype == GOOD) {
    fprintf(stderr, "Proxy Test #%d: %s(): %s\n", GlobalTD->tn, rec->func, rec->msg);
  }
}

static int proxy_setup(void **tdata) {
  struct proxy_test_data *ptd = calloc(1, sizeof(struct proxy_test_data));  // 分配内存空间用于存储代理测试数据
  if (ptd == NULL)
    return -ENOMEM;

  ptd->nsp = nsock_pool_new(ptd);  // 创建新的套接字池
  AssertNonNull(ptd->nsp);  // 断言套接字池不为空

  nsock_set_log_function(log_handler);  // 设置日志处理函数

  *tdata = GlobalTD = ptd;  // 将全局测试数据指针指向新创建的代理测试数据
  return 0;
}

static int proxy_teardown(void *tdata) {
  struct proxy_test_data *ptd = (struct proxy_test_data *)tdata;

  if (tdata) {
    nsock_pool_delete(ptd->nsp);  // 删除套接字池
    释放 tdata 指针指向的内存空间
    free(tdata);
  }
  设置日志函数为 NULL，即不再记录日志
  nsock_set_log_function(NULL);
  将 GlobalTD 指针指向 NULL，释放内存空间
  GlobalTD = NULL;
  返回 0，表示函数执行成功
  return 0;
# 定义一个名为TestProxyParse的测试用例结构体
const struct test_case TestProxyParse = {
  # 测试用例的名称为"test nsock proxychain parsing"
  .t_name     = "test nsock proxychain parsing",
  # 设置测试用例的初始化函数为proxy_setup
  .t_setup    = proxy_setup,
  # 设置测试用例的运行函数为parser_test
  .t_run      = parser_test,
  # 设置测试用例的清理函数为proxy_teardown
  .t_teardown = proxy_teardown
};
```