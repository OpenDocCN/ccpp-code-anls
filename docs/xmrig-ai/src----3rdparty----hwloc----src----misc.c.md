# `xmrig\src\3rdparty\hwloc\src\misc.c`

```
/*
 * 版权声明
 * 版权所有 © 2009 CNRS
 * 版权所有 © 2009-2020 Inria
 * 版权所有 © 2009-2010 Université Bordeaux
 * 版权所有 © 2009-2018 Cisco Systems, Inc.
 * 请参阅顶层目录中的 COPYING 文件。
 */

#include "private/autogen/config.h"  // 包含自动生成的配置文件
#include "private/private.h"  // 包含私有头文件
#include "private/misc.h"  // 包含私有杂项函数

#include <stdarg.h>  // 包含可变参数列表相关的头文件
#ifdef HAVE_SYS_UTSNAME_H
#include <sys/utsname.h>  // 包含系统信息相关的头文件
#endif
#include <stdlib.h>  // 包含标准库函数的头文件
#include <string.h>  // 包含字符串处理函数的头文件
#include <stdio.h>  // 包含输入输出函数的头文件
#include <errno.h>  // 包含错误处理相关的头文件
#include <ctype.h>  // 包含字符处理相关的头文件

#ifdef HAVE_PROGRAM_INVOCATION_NAME
#include <errno.h>  // 包含错误处理相关的头文件
extern char *program_invocation_name;  // 外部声明程序调用名
#endif
#ifdef HAVE___PROGNAME
extern char *__progname;  // 外部声明程序名
#endif

#ifndef HWLOC_HAVE_CORRECT_SNPRINTF
int hwloc_snprintf(char *str, size_t size, const char *format, ...)  // 定义格式化输出函数
{
  int ret;  // 定义返回值
  va_list ap;  // 定义可变参数列表
  static char bin;  // 定义静态字符变量
  size_t fakesize;  // 定义伪大小
  char *fakestr;  // 定义伪字符串

  /* 一些系统在 str == NULL 时会崩溃 */
  if (!size) {
    str = &bin;  // 如果大小为0，则将 str 指向静态字符变量
    size = 1;  // 将大小设置为1
  }

  va_start(ap, format);  // 开始可变参数列表
  ret = vsnprintf(str, size, format, ap);  // 格式化输出到字符串
  va_end(ap);  // 结束可变参数列表

  if (ret >= 0 && (size_t) ret != size-1)  // 如果返回值大于等于0且不等于 size-1
    return ret;  // 返回返回值

  /* vsnprintf 返回 size-1 或 -1。这可能是一个报告写入数据而不是实际所需空间的系统。尝试增加缓冲区大小以获得后者。 */
  fakesize = size;  // 将伪大小设置为 size
  fakestr = NULL;  // 将伪字符串设置为 NULL
  do {
    fakesize *= 2;  // 将伪大小乘以2
    free(fakestr);  // 释放伪字符串
    fakestr = malloc(fakesize);  // 分配伪大小的内存
    if (NULL == fakestr)  // 如果伪字符串为 NULL
      return -1;  // 返回-1
    va_start(ap, format);  // 开始可变参数列表
    errno = 0;  // 将错误号设置为0
    ret = vsnprintf(fakestr, fakesize, format, ap);  // 格式化输出到伪字符串
    va_end(ap);  // 结束可变参数列表
  } while ((size_t) ret == fakesize-1 || (ret < 0 && (!errno || errno == ERANGE)));  // 循环直到返回值等于伪大小-1或小于0且错误号为0或错误号为 ERANGE

  if (ret >= 0 && size) {
    if (size > (size_t) ret+1)
      size = ret+1;
    memcpy(str, fakestr, size-1);
    str[size-1] = 0;
  }
  free(fakestr);  // 释放伪字符串

  return ret;  // 返回返回值
}
#endif

void hwloc_add_uname_info(struct hwloc_topology *topology __hwloc_attribute_unused,
              void *cached_uname __hwloc_attribute_unused)
{
#ifdef HAVE_UNAME
  // 定义结构体变量_utsname和指向结构体的指针utsname
  struct utsname _utsname, *utsname;

  // 如果已经有"OSName"信息，则不再添加
  if (hwloc_obj_get_info_by_name(topology->levels[0][0], "OSName"))
    return;

  // 如果已经缓存了uname信息，则使用缓存的信息，否则调用uname获取信息
  if (cached_uname)
    utsname = (struct utsname *) cached_uname;
  else {
    utsname = &_utsname;
    if (uname(utsname) < 0)
      return;
  }

  // 如果sysname不为空，则添加"OSName"信息
  if (*utsname->sysname)
    hwloc_obj_add_info(topology->levels[0][0], "OSName", utsname->sysname);
  // 如果release不为空，则添加"OSRelease"信息
  if (*utsname->release)
    hwloc_obj_add_info(topology->levels[0][0], "OSRelease", utsname->release);
  // 如果version不为空，则添加"OSVersion"信息
  if (*utsname->version)
    hwloc_obj_add_info(topology->levels[0][0], "OSVersion", utsname->version);
  // 如果nodename不为空，则添加"HostName"信息
  if (*utsname->nodename)
    hwloc_obj_add_info(topology->levels[0][0], "HostName", utsname->nodename);
  // 如果machine不为空，则添加"Architecture"信息
  if (*utsname->machine)
    hwloc_obj_add_info(topology->levels[0][0], "Architecture", utsname->machine);
#endif /* HAVE_UNAME */
}

char *
hwloc_progname(struct hwloc_topology *topology __hwloc_attribute_unused)
{
#if (defined HAVE_DECL_GETMODULEFILENAME) && HAVE_DECL_GETMODULEFILENAME
  // 定义变量name和local_basename
  char name[256], *local_basename;
  // 获取当前模块的文件名
  unsigned res = GetModuleFileName(NULL, name, sizeof(name));
  // 如果获取失败或文件名长度超过256，则返回NULL
  if (res == sizeof(name) || !res)
    return NULL;
  // 获取文件名中的基本名称
  local_basename = strrchr(name, '\\');
  if (!local_basename)
    local_basename = name;
  else
    local_basename++;
  // 返回基本名称的副本
  return strdup(local_basename);
#else /* !HAVE_GETMODULEFILENAME */
  const char *name, *local_basename;
  // 根据不同系统获取程序名
#if HAVE_DECL_GETPROGNAME
  name = getprogname(); /* FreeBSD, NetBSD, some Solaris */
#elif HAVE_DECL_GETEXECNAME
  name = getexecname(); /* Solaris */
#elif defined HAVE_PROGRAM_INVOCATION_NAME
  name = program_invocation_name; /* Glibc. BGQ CNK. */
  // 移除路径信息，只保留文件名
#elif defined HAVE___PROGNAME
  name = __progname; /* fallback for most unix, used for OpenBSD */
#else
  // 其他系统的获取程序名的方法
  /* TODO: _NSGetExecutablePath(path, &size) on Darwin */
  /* TODO: AIX, HPUX */
  name = NULL;
#endif
  // 如果程序名为空，则返回NULL
  if (!name)
    # 返回空指针
    return NULL;
  # 在文件名中查找最后一个斜杠，返回指向该位置的指针
  local_basename = strrchr(name, '/');
  # 如果没有找到斜杠，将本地基本名称设置为文件名
  if (!local_basename)
    local_basename = name;
  # 如果找到斜杠，将本地基本名称设置为斜杠后的字符串
  else
    local_basename++;
  # 复制本地基本名称的字符串，并返回指向新分配的内存的指针
  return strdup(local_basename);
#endif /* !HAVE_GETMODULEFILENAME */
}

这是C/C++代码中的预处理指令，用于结束一个条件编译块。在这里，它结束了一个条件编译块，然后是一个右括号，表示代码块的结束。
```