# `nmap\nping\output.h`

```cpp
#ifndef OUTPUT_H
#define OUTPUT_H

#ifdef HAVE_CONFIG_H
#include "nping_config.h"
#else
#ifdef WIN32
#include "nping_winconfig.h"
#endif /* WIN32 */
#endif /* HAVE_CONFIG_H */

#ifdef STDC_HEADERS
#include <stdlib.h>
#endif

#include <stdarg.h>
#include <stdio.h>

#if HAVE_UNISTD_H
#include <unistd.h>
#endif

#define MAX_ERR_STR_LEN 1024      /* 定义最大错误消息长度为1024 */
#define MAX_OUTPUT_STR_LEN 2048   /* 定义最大输出消息长度为2048 */

/* 为了与 Nmap 代码向后兼容 */
int fatal(const char *str, ...)
     __attribute__ ((format (printf, 1, 2)));  /* 标记函数参数，使其具有printf风格的格式化 */

int error(const char *str, ...)
     __attribute__ ((format (printf, 1, 2)));  /* 标记函数参数，使其具有printf风格的格式化 */

/* 为了与 Nmap 代码向后兼容 */
int pfatal(const char *str, ...)
     __attribute__ ((format (printf, 1, 2)));  /* 标记函数参数，使其具有printf风格的格式化 */

int nping_fatal(int level, const char *str, ...)
    __attribute__((format(printf, 2, 3)));  /* 标记函数参数，使其具有printf风格的格式化 */

int nping_warning(int level, const char *str, ...)
    __attribute__((format(printf, 2, 3)));  /* 标记函数参数，使其具有printf风格的格式化 */

int nping_print(int level, const char *str, ...)
    __attribute__((format(printf, 2, 3)));  /* 标记函数参数，使其具有printf风格的格式化 */

#ifdef __cplusplus
extern "C" {
#endif

#ifdef __cplusplus
}
#endif

#endif /* NPING_ERROR_H */
```