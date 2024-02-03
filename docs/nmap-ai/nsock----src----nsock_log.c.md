# `nmap\nsock\src\nsock_log.c`

```cpp
/* $Id$ */

// 包含必要的头文件
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <assert.h>

// 包含自定义的头文件
#include "nsock_internal.h"
#include "nsock_log.h"

// 定义一个用于输出日志到标准错误流的函数
static void nsock_stderr_logger(const struct nsock_log_rec *rec);

// 外部变量声明
extern struct timeval nsock_tod;

// 初始化默认日志级别和日志输出函数
nsock_loglevel_t    NsockLogLevel = NSOCK_LOG_ERROR;
nsock_logger_t      NsockLogger   = nsock_stderr_logger;

// 设置日志输出函数
void nsock_set_log_function(nsock_logger_t logger) {
  if (logger != NULL)
    NsockLogger = logger;
  else
    NsockLogger = nsock_stderr_logger;

  nsock_log_debug("Registered external logging function: %p", NsockLogger);
}

// 获取当前日志级别
nsock_loglevel_t nsock_get_loglevel(void) {
  return NsockLogLevel;
}

// 设置日志级别
void nsock_set_loglevel(nsock_loglevel_t loglevel) {
  NsockLogLevel = loglevel;
  nsock_log_debug("Set log level to %s", nsock_loglevel2str(loglevel));
}

// 输出日志到标准错误流的函数实现
void nsock_stderr_logger(const struct nsock_log_rec *rec) {
  fprintf(stderr, "libnsock %s(): %s\n", rec->func, rec->msg);
}

// 内部日志输出函数，支持可变参数
void __nsock_log_internal(nsock_loglevel_t loglevel, const char *file, int line,
                          const char *func, const char *format, ...) {
  struct nsock_log_rec rec;
  va_list args;
  int rc;

  va_start(args, format);

  rec.level = loglevel;
  rec.time = nsock_tod;
  rec.file = file;
  rec.line = line;
  rec.func = func;

  rc = vasprintf(&rec.msg, format, args);
  if (rc >= 0) {
    NsockLogger(&rec);
    free(rec.msg);
  }
  va_end(args);
}
```