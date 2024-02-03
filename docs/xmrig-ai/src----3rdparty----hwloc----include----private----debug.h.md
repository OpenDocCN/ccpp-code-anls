# `xmrig\src\3rdparty\hwloc\include\private\debug.h`

```cpp
/*
 * 版权所有 © 2009 CNRS
 * 版权所有 © 2009-2020 Inria
 * 版权所有 © 2009, 2011 Université Bordeaux
 * 版权所有 © 2011 Cisco Systems, Inc.
 * 请参阅顶层目录中的 COPYING 文件。
 */

/* 配置文件 */

#ifndef HWLOC_DEBUG_H
#define HWLOC_DEBUG_H

#include "private/autogen/config.h"
#include "private/misc.h"

#ifdef HWLOC_DEBUG
#include <stdarg.h>
#include <stdio.h>
#endif

#ifdef ANDROID
extern void JNIDebug(char *text);
#endif

/* 编译时断言 */
#define HWLOC_BUILD_ASSERT(condition) ((void)sizeof(char[1 - 2*!(condition)]))

#ifdef HWLOC_DEBUG
static __hwloc_inline int hwloc_debug_enabled(void)
{
  static int checked = 0;
  static int enabled = 1;
  if (!checked) {
    const char *env = getenv("HWLOC_DEBUG_VERBOSE");
    if (env)
      enabled = atoi(env);
    if (enabled)
      fprintf(stderr, "hwloc verbose debug enabled, may be disabled with HWLOC_DEBUG_VERBOSE=0 in the environment.\n");
    checked = 1;
  }
  return enabled;
}
#endif

static __hwloc_inline void hwloc_debug(const char *s __hwloc_attribute_unused, ...) __hwloc_attribute_format(printf, 1, 2);
static __hwloc_inline void hwloc_debug(const char *s __hwloc_attribute_unused, ...)
{
#ifdef HWLOC_DEBUG
  if (hwloc_debug_enabled()) {
#ifdef ANDROID
    char buffer[256];
#endif
    va_list ap;
    va_start(ap, s);
#ifdef ANDROID
    vsprintf(buffer, s, ap);
    JNIDebug(buffer);
#else
    vfprintf(stderr, s, ap);
#endif
    va_end(ap);
  }
#endif
}

#ifdef HWLOC_DEBUG
#define hwloc_debug_bitmap(fmt, bitmap) do { \
if (hwloc_debug_enabled()) { \
  char *s; \
  hwloc_bitmap_asprintf(&s, bitmap); \
  hwloc_debug(fmt, s); \
  free(s); \
} } while (0)
#define hwloc_debug_1arg_bitmap(fmt, arg1, bitmap) do { \
if (hwloc_debug_enabled()) { \
  char *s; \
  hwloc_bitmap_asprintf(&s, bitmap); \
  hwloc_debug(fmt, arg1, s); \
  free(s); \
} } while (0)
#define hwloc_debug_2args_bitmap(fmt, arg1, arg2, bitmap) do { \
#ifdef HWLOC_DEBUG_H
// 如果调试功能已启用
#define hwloc_debug_bitmap(s, bitmap) do { \
  // 声明一个字符串指针
  char *s; \
  // 将位图转换为字符串格式，并赋值给s
  hwloc_bitmap_asprintf(&s, bitmap); \
  // 调试输出格式化字符串和参数，以及位图的字符串格式
  hwloc_debug(fmt, arg1, arg2, s); \
  // 释放字符串指针所指向的内存
  free(s); \
} while (0)
// 如果调试功能未启用，则定义空的宏
#else
#define hwloc_debug_bitmap(s, bitmap) do { } while(0)
#define hwloc_debug_1arg_bitmap(s, arg1, bitmap) do { } while(0)
#define hwloc_debug_2args_bitmap(s, arg1, arg2, bitmap) do { } while(0)
#endif
// 结束条件编译
#endif /* HWLOC_DEBUG_H */
```