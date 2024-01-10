# `nmap\nsock\tests\tests_main.c`

```
/*
 * Nsock regression test suite
 * Same license as nmap -- see https://nmap.org/book/man-legal.html
 */

#include "test-common.h"  // 包含测试公共函数的头文件
#include "string.h"  // 包含字符串处理函数的头文件

#ifndef WIN32
  #define RESET       "\033[0m"  // 定义 ANSI 控制码，用于重置终端颜色
  #define BOLDRED     "\033[1m\033[31m"  // 定义 ANSI 控制码，用于设置粗体红色
  #define BOLDGREEN   "\033[1m\033[32m"  // 定义 ANSI 控制码，用于设置粗体绿色
  #define TEST_FAILED "[" BOLDRED "FAILED" RESET "]"  // 定义测试失败的提示信息
  #define TEST_OK     "[" BOLDGREEN "OK" RESET "]"  // 定义测试成功的提示信息
#else
  /* WIN32 terminal has no ANSI driver */
  #define TEST_FAILED "[FAILED]"  // 定义测试失败的提示信息
  #define TEST_OK     "[OK]"  // 定义测试成功的提示信息
#endif

/* socket_strerror() comes from nbase
 * Declared here to work around a silly inclusion issue until I can fix it. */
char *socket_strerror(int errnum);  // 声明 socket_strerror 函数，用于获取错误信息

extern const struct test_case TestPoolUserData;  // 声明测试用例 TestPoolUserData
extern const struct test_case TestTimer;  // 声明测试用例 TestTimer
extern const struct test_case TestLogLevels;  // 声明测试用例 TestLogLevels
extern const struct test_case TestErrLevels;  // 声明测试用例 TestErrLevels
extern const struct test_case TestConnectTCP;  // 声明测试用例 TestConnectTCP
extern const struct test_case TestConnectFailure;  // 声明测试用例 TestConnectFailure
extern const struct test_case TestGHLists;  // 声明测试用例 TestGHLists
extern const struct test_case TestGHHeaps;  // 声明测试用例 TestGHHeaps
extern const struct test_case TestHeapOrdering;  // 声明测试用例 TestHeapOrdering
extern const struct test_case TestProxyParse;  // 声明测试用例 TestProxyParse
extern const struct test_case TestCancelTCP;  // 声明测试用例 TestCancelTCP
extern const struct test_case TestCancelUDP;  // 声明测试用例 TestCancelUDP
#ifdef HAVE_OPENSSL
extern const struct test_case TestCancelSSL;  // 声明测试用例 TestCancelSSL
#endif

static const struct test_case *TestCases[] = {  // 定义测试用例数组
  /* ---- basic.c */
  &TestPoolUserData,  // 添加测试用例 TestPoolUserData 到数组
  /* ---- timer.c */
  &TestTimer,  // 添加测试用例 TestTimer 到数组
  /* ---- logs.c */
  &TestLogLevels,  // 添加测试用例 TestLogLevels 到数组
  &TestErrLevels,  // 添加测试用例 TestErrLevels 到数组
  /* ---- connect.c */
  &TestConnectTCP,  // 添加测试用例 TestConnectTCP 到数组
  &TestConnectFailure,  // 添加测试用例 TestConnectFailure 到数组
  /* ---- ghlists.c */
  &TestGHLists,  // 添加测试用例 TestGHLists 到数组
  /* ---- ghheaps.c */
  &TestGHHeaps,  // 添加测试用例 TestGHHeaps 到数组
  &TestHeapOrdering,  // 添加测试用例 TestHeapOrdering 到数组
  /* ---- proxychain.c */
  &TestProxyParse,  // 添加测试用例 TestProxyParse 到数组
  /* ---- cancel.c */
  &TestCancelTCP,  // 添加测试用例 TestCancelTCP 到数组
  &TestCancelUDP,  // 添加测试用例 TestCancelUDP 到数组
#ifdef HAVE_OPENSSL
  &TestCancelSSL,  // 添加测试用例 TestCancelSSL 到数组
#endif
  NULL  // 数组结束标志
};

static int test_case_run(const struct test_case *test) {  // 定义测试用例运行函数
  int rc;  // 定义整型变量 rc，用于存储返回值
  void *tdata = NULL;  // 定义空指针变量 tdata

  rc = test_setup(test, &tdata);  // 调用测试设置函数，初始化测试用例数据
  if (rc)  // 如果返回值不为 0
    return rc;  // 返回返回值

  rc = test_run(test, tdata);  // 调用测试运行函数，执行测试用例
  if (rc)  // 如果返回值不为 0
    # 返回变量 rc 的值
    return rc;

  # 调用 test_teardown 函数，传入参数 test 和 tdata，并返回其结果
  return test_teardown(test, tdata);
#ifdef WIN32
// 在 Windows 平台下初始化 Winsock
static int win_init(void) {
  WSADATA data;
  int rc;

  // 启动 Winsock
  rc = WSAStartup(MAKEWORD(2, 2), &data);
  // 如果启动失败，输出错误信息并退出
  if (rc)
    fatal("Failed to start winsock: %s\n", socket_strerror(rc));

  return 0;
}
#endif

// 主函数
int main(int ac, char **av) {
  int rc, i;

  /* simple "do we have ssl" check for run_tests.sh */
  // 简单检查是否有 SSL
  if (ac == 2 && !strncmp(av[1], "--ssl", 5)) {
#ifdef HAVE_SSL
    return 0;
#else
    return 1;
#endif
  }

#ifdef WIN32
  // 在 Windows 平台下初始化 Winsock
  win_init();
#endif

  // 遍历测试用例数组
  for (i = 0; TestCases[i] != NULL; i++) {
    const struct test_case *current = TestCases[i];
    const char *name = get_test_name(current);

    // 输出测试用例名称
    printf("%-48s", name);
    fflush(stdout);
    // 运行测试用例
    rc = test_case_run(current);
    // 如果运行失败，输出错误信息并退出
    if (rc) {
      printf(TEST_FAILED " (%s)\n", socket_strerror(-rc));
      break;
    }
    // 输出测试通过信息
    printf(TEST_OK "\n");
  }
  return rc;
}
```