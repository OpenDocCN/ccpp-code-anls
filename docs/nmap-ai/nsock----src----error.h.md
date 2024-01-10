# `nmap\nsock\src\error.h`

```
/* $Id$ */

// 如果 ERROR_H 未定义，则定义 ERROR_H
#ifndef ERROR_H
#define ERROR_H

// 如果 HAVE_CONFIG_H 定义了，则包含 nsock_config.h 和 nbase_config.h
#ifdef HAVE_CONFIG_H
#include "nsock_config.h"
#include "nbase_config.h"
#endif

// 如果 WIN32 定义了，则包含 nbase_winconfig.h
#ifdef WIN32
#include "nbase_winconfig.h"
#endif

// 包含标准库头文件
#include <stdarg.h>
#include <stdio.h>
#include <stdlib.h>
// 如果定义了 HAVE_UNISTD_H，则包含 unistd.h
#if HAVE_UNISTD_H
#include <unistd.h>
#endif

// 如果定义了 __GNUC__，则定义 NORETURN 为 __attribute__((noreturn))
// 如果定义了 _MSC_VER，则定义 NORETURN 为 __declspec(noreturn)
// 否则定义 NORETURN 为空
#if defined(__GNUC__)
#define NORETURN __attribute__((noreturn))
#elif defined(_MSC_VER)
#define NORETURN __declspec(noreturn)
#else
#define NORETURN
#endif

// 声明一个函数 fatal，参数为可变参数列表，格式化字符串为第一个参数
NORETURN void fatal(char *fmt, ...)
  __attribute__ ((format (printf, 1, 2)));

// 声明一个函数 pfatal，参数为可变参数列表，格式化字符串为第一个参数
NORETURN void pfatal(char *fmt, ...)
  __attribute__ ((format (printf, 1, 2)));

// 结束条件编译，定义结束
#endif /* ERROR_H */
```