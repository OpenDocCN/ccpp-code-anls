# `nmap\nsock\tests\basic.c`

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

// 设置测试环境
static int basic_setup(void **tdata) {
  struct basic_test_data *btd;

  // 分配内存给基本测试数据结构
  btd = calloc(1, sizeof(struct basic_test_data));
  if (btd == NULL)
    return -ENOMEM;

  // 创建新的nsock池
  btd->nsp = nsock_pool_new(NULL);

  *tdata = btd;
  return 0;
}

// 清理测试环境
static int basic_teardown(void *tdata) {
  struct basic_test_data *btd = (struct basic_test_data *)tdata;

  if (tdata) {
    // 删除nsock池
    nsock_pool_delete(btd->nsp);
    free(tdata);
  }
  return 0;
}

// 测试用户数据
static int basic_udata(void *tdata) {
  struct basic_test_data *btd = (struct basic_test_data *)tdata;

  // 断言nsock池的用户数据为空
  AssertEqual(nsock_pool_get_udata(btd->nsp), NULL);
  // 设置nsock池的用户数据为btd
  nsock_pool_set_udata(btd->nsp, btd);
  // 断言nsock池的用户数据为btd
  AssertEqual(nsock_pool_get_udata(btd->nsp), btd);
  return 0;
}

// 定义测试用例
const struct test_case TestPoolUserData = {
  .t_name     = "nsock池用户数据",
  .t_setup    = basic_setup,
  .t_run      = basic_udata,
  .t_teardown = basic_teardown
};
```