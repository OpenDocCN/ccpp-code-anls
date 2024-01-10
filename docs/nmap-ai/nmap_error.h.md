# `nmap\nmap_error.h`

```
/* $Id$ */

// 如果没有定义 NMAP_ERROR_H，则定义 NMAP_ERROR_H
#ifndef NMAP_ERROR_H
#define NMAP_ERROR_H

// 如果有配置文件 nmap_config.h，则包含该文件
#ifdef HAVE_CONFIG_H
#include "nmap_config.h"
// 如果没有配置文件 nmap_config.h 但是在 WIN32 环境下，则包含 nmap_winconfig.h
#else
#ifdef WIN32
#include "nmap_winconfig.h"
#endif /* WIN32 */
#endif /* HAVE_CONFIG_H */

// 包含 nbase.h 文件
#include <nbase.h>

// 如果有标准 C 头文件，则包含 stdlib.h 文件
#ifdef STDC_HEADERS
#include <stdlib.h>
#endif

// 包含可变参数的头文件
#include <stdarg.h>

// 如果有 unistd.h 文件，则包含该文件
#if HAVE_UNISTD_H
#include <unistd.h>
#endif

// 如果是 C++ 环境，则使用 extern "C" 语法
#ifdef __cplusplus
extern "C" {
#endif

// 定义一个不返回值的函数 fatal，参数为格式化字符串和可变参数
NORETURN void fatal(const char *fmt, ...)
     __attribute__ ((format (printf, 1, 2)));
// 定义一个不返回值的函数 error，参数为格式化字符串和可变参数
void error(const char *fmt, ...)
     __attribute__ ((format (printf, 1, 2)));

// 定义一个不返回值的函数 pfatal，参数为格式化字符串和可变参数
NORETURN void pfatal(const char *err, ...)
     __attribute__ ((format (printf, 1, 2)));
// 定义一个不返回值的函数 gh_perror，参数为格式化字符串和可变参数
void gh_perror(const char *err, ...)
     __attribute__ ((format (printf, 1, 2)));

// 如果没有定义 strerror 函数，则定义 strerror 函数
#ifndef HAVE_STRERROR
char *strerror(int errnum);
#endif

// 如果是 C++ 环境，则使用 extern "C" 语法
#ifdef __cplusplus
}
#endif

// 结束条件编译
#endif /* NMAP_ERROR_H */
```