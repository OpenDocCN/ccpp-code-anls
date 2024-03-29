# `nmap\nsock\tests\test-common.h`

```cpp
/*
 * Nsock regression test suite
 * Same license as nmap -- see https://nmap.org/book/man-legal.html
 */
// 包含必要的头文件
#ifndef __TEST_COMMON_H
#define __TEST_COMMON_H

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>
#include <assert.h>
#include <nsock.h>

// 定义端口号常量
#define PORT_UDP    55234
#define PORT_TCP    55235
#define PORT_TCPSSL 55236

// 定义断言宏
#define __ASSERT_BASE(stmt)    do { \
        if (!(stmt)) { \
            fprintf(stderr, "(%s:%d) Assertion failed: " #stmt "\n", \
                    __FILE__, __LINE__); \
            return -EINVAL; \
        } \
    } while (0)

#define AssertNonNull(a)        __ASSERT_BASE((a) != NULL);
#define AssertEqual(a, b)       __ASSERT_BASE((a) == (b));
#define AssertStrEqual(a, b)    __ASSERT_BASE(strcmp((a), (b)) == 0);

// 定义测试用例结构体
struct test_case {
  const char *t_name;
  int (*t_setup)(void **tdata);
  int (*t_run)(void *tdata);
  int (*t_teardown)(void *tdata);
};

// 获取测试用例名称
static inline const char *get_test_name(const struct test_case *test) {
  return test->t_name;
}

// 执行测试用例的设置阶段
static inline int test_setup(const struct test_case *test, void **tdata) {
  int rc;

  assert(test);

  if (test->t_setup)
    rc = test->t_setup(tdata);
  else
    rc = 0;

  return rc;
}

// 执行测试用例的运行阶段
static inline int test_run(const struct test_case *test, void *tdata) {
  int rc;

  assert(test);

  if (test->t_run)
    rc = test->t_run(tdata);
  else
    rc = 0;

  return rc;
}

// 执行测试用例的清理阶段
static inline int test_teardown(const struct test_case *test, void *tdata) {
  int rc;

  assert(test);

  if (test->t_teardown)
    rc = test->t_teardown(tdata);
  else
    rc = 0;

  return rc;
}

#endif /* ^__TEST_COMMON_H */
```