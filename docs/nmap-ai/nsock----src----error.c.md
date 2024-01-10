# `nmap\nsock\src\error.c`

```
/* $Id$ */

#include "error.h"

#include <stdio.h>
#include <stdlib.h>
#include <stdarg.h>

// 当程序发生致命错误时，打印错误信息并退出程序
void fatal(char *fmt, ...) {
  va_list ap;

  // 初始化可变参数列表
  va_start(ap, fmt);
  // 清空标准输出缓冲区，将格式化的错误信息输出到标准错误流
  fflush(stdout);
  vfprintf(stderr, fmt, ap);
  // 输出换行符
  fprintf(stderr, "\n");
  // 结束可变参数列表的使用
  va_end(ap);

  // 退出程序
  exit(1);
}

// 当发生系统调用错误时，打印错误信息并退出程序
void pfatal(char *fmt, ...) {
  va_list ap;

  // 初始化可变参数列表
  va_start(ap, fmt);
  // 清空标准输出缓冲区，将格式化的错误信息输出到标准错误流
  fflush(stdout);
  vfprintf(stderr, fmt, ap);
  // 输出冒号和系统调用错误信息
  fprintf(stderr, ": ");
  perror("");
  // 输出换行符
  fprintf(stderr, "\n");
  // 结束可变参数列表的使用
  va_end(ap);

  // 退出程序
  exit(1);
}
```