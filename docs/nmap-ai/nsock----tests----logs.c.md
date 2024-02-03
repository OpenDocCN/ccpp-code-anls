# `nmap\nsock\tests\logs.c`

```cpp
/*
 * Nsock regression test suite
 * Same license as nmap -- see https://nmap.org/book/man-legal.html
 */

#include "test-common.h"


struct log_test_data {
  nsock_pool nsp;  // 定义结构体成员 nsp，表示 nsock_pool 对象
  nsock_loglevel_t current_level;  // 定义结构体成员 current_level，表示当前日志级别
  unsigned int got_dbgfull: 1;  // 定义结构体成员 got_dbgfull，表示是否收到了 NSOCK_LOG_DBG_ALL 级别的日志
  unsigned int got_dbg: 1;  // 定义结构体成员 got_dbg，表示是否收到了 NSOCK_LOG_DBG 级别的日志
  unsigned int got_info: 1;  // 定义结构体成员 got_info，表示是否收到了 NSOCK_LOG_INFO 级别的日志
  unsigned int got_error: 1;  // 定义结构体成员 got_error，表示是否收到了 NSOCK_LOG_ERROR 级别的日志
  unsigned int total;  // 定义结构体成员 total，表示收到的日志总数
  int errcode;  // 定义结构体成员 errcode，表示错误代码
};

static struct log_test_data *GlobalLTD;  // 定义全局变量 GlobalLTD，指向 log_test_data 结构体

static void log_handler(const struct nsock_log_rec *rec) {  // 定义日志处理函数 log_handler
  GlobalLTD->total++;  // 收到日志总数加一
  switch(rec->level) {  // 根据日志级别进行判断
    case NSOCK_LOG_DBG_ALL:  // 如果是 NSOCK_LOG_DBG_ALL 级别的日志
      GlobalLTD->got_dbgfull = 1;  // 设置 got_dbgfull 为 1
      break;

    case NSOCK_LOG_DBG:  // 如果是 NSOCK_LOG_DBG 级别的日志
      GlobalLTD->got_dbg = 1;  // 设置 got_dbg 为 1
      break;

    case NSOCK_LOG_INFO:  // 如果是 NSOCK_LOG_INFO 级别的日志
      GlobalLTD->got_info = 1;  // 设置 got_info 为 1
      break;

    case NSOCK_LOG_ERROR:  // 如果是 NSOCK_LOG_ERROR 级别的日志
      GlobalLTD->got_error = 1;  // 设置 got_error 为 1
      break;

    default:  // 如果是其他未知的日志级别
      fprintf(stderr, "UNEXPECTED LOG LEVEL (%d)!\n", (int)rec->level);  // 输出错误信息
      GlobalLTD->errcode = -EINVAL;  // 设置错误代码为 -EINVAL
  }
}

static void nop_handler(nsock_pool nsp, nsock_event nse, void *udata) {  // 定义空处理函数 nop_handler
}

static int check_loglevel(struct log_test_data *ltd, nsock_loglevel_t level) {  // 定义检查日志级别的函数 check_loglevel
  int rc = 0;  // 定义返回值 rc，初始化为 0
  nsock_event_id id;  // 定义事件 ID

  nsock_set_loglevel(level);  // 设置日志级别为指定级别

  ltd->current_level = level;  // 更新当前日志级别为指定级别

  // 重置各种日志标记和计数
  ltd->got_dbgfull = 0;
  ltd->got_dbg     = 0;
  ltd->got_info    = 0;
  ltd->got_error   = 0;
  ltd->total   = 0;
  ltd->errcode = 0;

  id = nsock_timer_create(ltd->nsp, nop_handler, 200, NULL);  // 创建定时器事件
  nsock_event_cancel(ltd->nsp, id, 0);  // 取消定时器事件

  if (ltd->errcode)  // 如果有错误代码
    return ltd->errcode;  // 返回错误代码

  if (ltd->total < 1)  // 如果收到的日志总数小于 1
    return -EINVAL;  // 返回 -EINVAL

  return rc;  // 返回结果
}

static int check_errlevel(struct log_test_data *ltd, nsock_loglevel_t level) {  // 定义检查错误级别的函数 check_errlevel
  nsock_event_id id;  // 定义事件 ID

  nsock_set_loglevel(level);  // 设置日志级别为指定级别

  ltd->current_level = level;  // 更新当前日志级别为指定级别

  // 重置各种日志标记和计数
  ltd->got_dbgfull = 0;
  ltd->got_dbg     = 0;
  ltd->got_info    = 0;
  ltd->got_error   = 0;
  ltd->total   = 0;
  ltd->errcode = 0;

  id = nsock_timer_create(ltd->nsp, nop_handler, 200, NULL);  // 创建定时器事件
  nsock_event_cancel(ltd->nsp, id, 0);  // 取消定时器事件

  if (ltd->errcode)  // 如果有错误代码
    return ltd->errcode;  // 返回错误代码

  if (ltd->total > 0)  // 如果收到的日志总数大于 0
    # 返回错误码 -EINVAL
    return -EINVAL;

  # 返回成功状态吗 0
  return 0;
# 设置日志测试数据的初始化
static int log_setup(void **tdata) {
  # 分配内存以存储日志测试数据
  struct log_test_data *ltd;
  ltd = calloc(1, sizeof(struct log_test_data));
  # 检查内存分配是否成功
  if (ltd == NULL)
    return -ENOMEM;
  # 创建新的网络套接字池
  ltd->nsp = nsock_pool_new(ltd);
  # 断言网络套接字池不为空
  AssertNonNull(ltd->nsp);
  # 将全局日志测试数据指针指向当前测试数据
  *tdata = GlobalLTD = ltd;
  return 0;
}

# 清理日志测试数据
static int log_teardown(void *tdata) {
  # 将测试数据转换为日志测试数据结构
  struct log_test_data *ltd = (struct log_test_data *)tdata;
  # 如果测试数据存在
  if (tdata) {
    # 删除网络套接字池
    nsock_pool_delete(ltd->nsp);
    # 释放测试数据内存
    free(tdata);
  }
  # 设置日志函数为空
  nsock_set_log_function(NULL);
  # 将全局日志测试数据指针设置为空
  GlobalLTD = NULL;
  return 0;
}

# 检查标准日志级别
static int log_check_std_levels(void *tdata) {
  # 将测试数据转换为日志测试数据结构
  struct log_test_data *ltd = (struct log_test_data *)tdata;
  # 定义日志级别和返回码
  nsock_loglevel_t lvl;
  int rc = 0;
  # 设置日志函数为日志处理函数
  nsock_set_log_function(log_handler);
  # 遍历标准日志级别
  for (lvl = NSOCK_LOG_DBG_ALL; lvl < NSOCK_LOG_ERROR; lvl++) {
    # 检查日志级别
    rc = check_loglevel(ltd, lvl);
    # 如果返回码不为0，则返回返回码
    if (rc)
      return rc;
  }
  return 0;
}

# 检查错误日志级别
static int log_check_err_levels(void *tdata) {
  # 将测试数据转换为日志测试数据结构
  struct log_test_data *ltd = (struct log_test_data *)tdata;
  # 定义日志级别和返回码
  nsock_loglevel_t lvl;
  int rc = 0;
  # 设置日志函数为日志处理函数
  nsock_set_log_function(log_handler);
  # 遍历错误日志级别
  for (lvl = NSOCK_LOG_ERROR; lvl <= NSOCK_LOG_NONE; lvl++) {
    # 检查错误日志级别
    rc = check_errlevel(ltd, NSOCK_LOG_ERROR);
    # 如果返回码不为0，则返回返回码
    if (rc)
      return rc;
  }
  return 0;
}

# 定义测试用例 TestLogLevels
const struct test_case TestLogLevels = {
  .t_name     = "set standard log levels",
  .t_setup    = log_setup,
  .t_run      = log_check_std_levels,
  .t_teardown = log_teardown
};

# 定义测试用例 TestErrLevels
const struct test_case TestErrLevels = {
  .t_name     = "check error log levels",
  .t_setup    = log_setup,
  .t_run      = log_check_err_levels,
  .t_teardown = log_teardown
};
```