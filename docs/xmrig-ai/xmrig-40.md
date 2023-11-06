# xmrig源码解析 40

# `src/3rdparty/hwloc/src/topology.c`

这段代码定义了一个头文件名为 "private/autogen/config.h" 的文件。它包含了一些定义，然后在其中定義了一個名为 "config" 的常量。

```cpp
#define _ATFILE_SOURCE
#include <assert.h>
#include <sys/types.h>
#ifdef HAVE_DIRENT_H
```

接着，它包含了一些导入库文件的路径，这些文件都是用于定义和实现文件操作的。
```cpp
#include "private/autogen/config.h"
```

然而，在文件的主体部分之前，它还包含了一个名为 "config.h" 的文件，这个文件可能是一个客户端头文件，它包含了实现文件操作的函数和定义。但是，这个文件的具体内容并没有在当前的条文中被实现。
```cpp
#define _ATFILE_SOURCE
#include <assert.h>
#include <sys/types.h>
```

另外，它还包含了一个名为 "config" 的常量，但是它的具体值没有给出，我们无法知道这个常量最终会被定义成什么。


```cpp
/*
 * Copyright © 2009 CNRS
 * Copyright © 2009-2022 Inria.  All rights reserved.
 * Copyright © 2009-2012, 2020 Université Bordeaux
 * Copyright © 2009-2011 Cisco Systems, Inc.  All rights reserved.
 * Copyright © 2022 IBM Corporation.  All rights reserved.
 * See COPYING in top-level directory.
 */

#include "private/autogen/config.h"

#define _ATFILE_SOURCE
#include <assert.h>
#include <sys/types.h>
#ifdef HAVE_DIRENT_H
```



该代码是一个C语言程序，它包括了多个头文件和函数声明。以下是对每个部分的作用的解释：

1. `#include <dirent.h>` 和 `#include <unistd.h>`：这两个头文件是用于输入输出函数和标准输入输出的。`<dirent.h>` 是用于输入输出函数的，它定义了几个常用的输入输出函数，如 `cdup`, `chmod`, `makedirs`, etc. `<unistd.h>` 是用于标准输入输出的，它定义了几个常用的标准输入输出函数，如 `write`, `read`, ` unlink`, etc.

2. `#ifdef HAVE_UNISTD_H` 和 `#endif`：这两个指令是用于判断是否支持使用 `<unistd.h>` 中的函数或头文件。如果 `HAVE_UNISTD_H` 成立，则使用 `<unistd.h>` 中定义的函数或头文件，否则不要使用。

3. `#include <string.h>` 和 `#include <errno.h>`：这两个头文件是用于处理字符串和错误信息的。`<string.h>` 是用于处理字符串的，它定义了一些字符串操作的函数，如 `strlen`, `strcpy`, `strcat`, etc. `<errno.h>` 是用于处理错误信息的，它定义了一些与错误信息有关的函数，如 `errno`, `strerror`, etc.

4. `#include <stdio.h>` 和 `#include <sys/stat.h>`：这两个头文件是用于输入输出和文件操作的。`<stdio.h>` 是用于输入输出的，它定义了几个常用的输入输出函数，如 `write`, `read`, ` unlink`, etc. `<sys/stat.h>` 是用于文件操作的，它定义了几个文件操作的函数，如 `stat`, `fstat`, ` mv`, etc.

5. `#include <fcntl.h>` 和 `#include <limits.h>`：这两个头文件是用于文件输入输出的。`<fcntl.h>` 是用于文件输入输出的，它定义了几个文件操作的函数，如 `open`, ` close`, ` lstat`, etc. `<limits.h>` 是用于设置输入输出的上限，它定义了一些常量，如 `MAX_PATH_LENGTH`, `IMAX_NORMAL`, etc.

6. `#include "hwloc.h"` 和 `#include "private/private.h"`：这两个头文件是用于硬件配置和私有函数的。`hwloc.h` 是用于硬件配置的，它定义了一些硬件配置的函数，如 `hwloc_get_name`, `hwloc_get_model`, etc. `private/private.h` 是用于私有函数的，它定义了一些私有函数的函数名和参数，如 `hide_to_std_domain`, `set_private_max_size` 等。


```cpp
#include <dirent.h>
#endif
#ifdef HAVE_UNISTD_H
#include <unistd.h>
#endif
#include <string.h>
#include <errno.h>
#include <stdio.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <limits.h>
#include <float.h>

#include "hwloc.h"
#include "private/private.h"
```

这段代码是一个嵌入式系统中的编译时预处理指令链。它通过检查系统是否支持可移植的软件编译器（Mach），如果支持，则包含Mach和相关的头文件。

具体来说，这段代码包含以下几部分：

1. `#include "private/debug.h"` 和 `#include "private/misc.h"`：这两部分包含的是调试和测试相关的头文件，由于调试器和测试程序在系统启动前无法运行，因此需要放在预处理指令链中。

2. `#ifdef HAVE_MACH_MACH_INIT_H` 和 `#ifdef HAVE_MACH_INIT_H`：这两部分用于检查系统是否支持Mach工具链。如果系统支持Mach，则包含`<mach/mach_init.h>`和`<mach/mach_host.h>`头文件。

3. `#ifdef HAVE_MACH_MACH_HOST_H`：这一部分包含的是Mach主机模式的头文件，也是用于检查系统是否支持Mach工具链。

4. `#ifdef HAVE_SYS_PARAM_H`：这一部分包含的是系统参数的定义，通常在系统启动时使用。

综上所述，这段代码的作用是检查系统是否支持可移植的软件编译器（Mach），并包含Mach相关头文件以进行调试和测试。


```cpp
#include "private/debug.h"
#include "private/misc.h"

#ifdef HAVE_MACH_MACH_INIT_H
#include <mach/mach_init.h>
#endif
#ifdef HAVE_MACH_INIT_H
#include <mach_init.h>
#endif
#ifdef HAVE_MACH_MACH_HOST_H
#include <mach/mach_host.h>
#endif

#ifdef HAVE_SYS_PARAM_H
#include <sys/param.h>
```

这段代码是一个if语句的备份，其中如果当前环境中存在名为HWLOC_WIN_SYS的系统架构，则包含头文件<windows.h>。这个系统架构在`#ifdef HWLOC_WIN_SYS`时被预先设置。

否则，如果当前环境中存在名为HWLOC_HAVE_LEVELZERO的系统架构，则包含头文件<sys/sysctl.h>。这个系统架构在`#ifdef HWLOC_HAVE_LEVELZERO`时被预先设置，但是如果当前环境中不存在这个系统架构，则不会加载这个头文件。

如果当前环境中存在名为HWLOC_WIN_SYS的系统架构，则做一些特定的事情。如果当前环境中存在名为HWLOC_HAVE_LEVELZERO的系统架构，则开启ZES_ENABLE_SYSMAN系统架构，这意味着Sysman功能将被启用。

否则，如果当前环境中存在名为HWLOC_WIN_SYS的系统架构，则做一些特定的事情。如果当前环境中存在名为HWLOC_HAVE_LEVELZERO的系统架构，则开启ZES_ENABLE_SYSMAN系统架构，这意味着Sysman功能将被启用。

否则，如果当前环境中存在名为HWLOC_WIN_SYS的系统架构，则做一些特定的事情。如果当前环境中存在名为HWLOC_HAVE_LEVELZERO的系统架构，则开启ZES_ENABLE_SYSMAN系统架构，这意味着Sysman功能将被启用。

否则，如果当前环境中存在名为HWLOC_WIN_SYS的系统架构，则做一些特定的事情。如果当前环境中存在名为HWLOC_HAVE_LEVELZERO的系统架构，则开启ZES_ENABLE_SYSMAN系统架构，这意味着Sysman功能将被启用。

否则，如果当前环境中存在名为HWLOC_WIN_SYS的系统架构，则做一些特定的事情。


```cpp
#endif

#ifdef HAVE_SYS_SYSCTL_H
#include <sys/sysctl.h>
#endif

#ifdef HWLOC_WIN_SYS
#include <windows.h>
#endif


#ifdef HWLOC_HAVE_LEVELZERO
/*
 * Define ZES_ENABLE_SYSMAN=1 early so that the LevelZero backend gets Sysman enabled.
 *
 * Only if the levelzero was enabled in this build so that we don't enable sysman
 * for external levelzero users when hwloc doesn't need it. If somebody ever loads
 * an external levelzero plugin in a hwloc library built without levelzero (unlikely),
 * he may have to manually set ZES_ENABLE_SYSMAN=1.
 *
 * Use the constructor if supported and/or the Windows DllMain callback.
 * Do it in the main hwloc library instead of the levelzero component because
 * the latter could be loaded later as a plugin.
 *
 * L0 seems to be using getenv() to check this variable on Windows
 * (at least in the Intel Compute-Runtime of March 2021),
 * but setenv() doesn't seem to exist on Windows, hence use putenv() to set the variable.
 *
 * For the record, Get/SetEnvironmentVariable() is not exactly the same as getenv/putenv():
 * - getenv() doesn't see what was set with SetEnvironmentVariable()
 * - GetEnvironmentVariable() doesn't see putenv() in cygwin (while it does in MSVC and MinGW).
 * Hence, if L0 ever switches from getenv() to GetEnvironmentVariable(),
 * it will break in cygwin, we'll have to use both putenv() and SetEnvironmentVariable().
 * Hopefully L0 will provide a way to enable Sysman without env vars before it happens.
 */
```

这段代码定义了一个名为 "hwloc_constructor" 的函数，用于在程序运行时初始化 HWLOC 库。

首先，通过判断操作系统是否支持 ZES(高性能系统管理)框架，如果操作系统不支持 ZES，则需要手动初始化。如果操作系统支持 ZES，则不需要手动初始化。

接下来，对于 Windows 系统，还需要在函数内输出 "ZES_ENABLE_SYSMAN=1"，这是因为 Windows 系统需要手动调用 putenv 函数来设置环境变量。

最后，对于 DLL 加载，如果函数的调用是为了进程加载，则需要输出 "ZES_ENABLE_SYSMAN=1"，这是因为 Windows 系统需要加载 DLL 并设置环境变量。

DllMain 函数用于在 DLL 加载时执行一些操作，包括设置 ZES 是否启用，设置 Enable sysman 环境变量等。


```cpp
#if HWLOC_HAVE_ATTRIBUTE_CONSTRUCTOR
static void hwloc_constructor(void) __attribute__((constructor));
static void hwloc_constructor(void)
{
  if (!getenv("ZES_ENABLE_SYSMAN"))
#ifdef HWLOC_WIN_SYS
    putenv("ZES_ENABLE_SYSMAN=1");
#else
    setenv("ZES_ENABLE_SYSMAN", "1", 1);
#endif
}
#endif
#ifdef HWLOC_WIN_SYS
BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpReserved)
{
  if (fdwReason == DLL_PROCESS_ATTACH) {
    if (!getenv("ZES_ENABLE_SYSMAN"))
      /* Windows does not have a setenv, so use putenv. */
      putenv((char *) "ZES_ENABLE_SYSMAN=1");
  }
  return TRUE;
}
```



这段代码是关于 hwloc_topology_abi_check() 的函数。

函数的作用是检查给定的 topology 是否与 hwloc_topology_abi 兼容，然后返回 0 或 -1。

函数中首先通过 HWLOC_API_VERSION 函数获取当前硬件loc 库的 api 版本，然后使用 topology->topology_abi 检查 topology 是否支持 hwloc_topology_abi。

如果 topology 不支持 hwloc_topology_abi，则函数返回 -1，否则返回 0。


```cpp
#endif
#endif /* HWLOC_HAVE_LEVELZERO */


unsigned hwloc_get_api_version(void)
{
  return HWLOC_API_VERSION;
}

int hwloc_topology_abi_check(hwloc_topology_t topology)
{
  return topology->topology_abi != HWLOC_TOPOLOGY_ABI ? -1 : 0;
}

/* callers should rather use wrappers HWLOC_SHOW_ALL_ERRORS() and HWLOC_SHOW_CRITICAL_ERRORS() for clarity */
```

这段代码是一个静态函数，称为 `hwloc_hide_errors()`。它的作用是隐藏操作系统中的错误输出，具体设置如下：

1. 在函数定义之前，定义了两个静态变量 `hide` 和 `checked`，初始值都为0。

2. 如果当前函数被调用，而且上文中未检查过，则执行以下操作：

  a. 获取环境变量 "HWLOC_HIDE_ERRORS"，并检查是否已经设置。

  b. 如果 "HWLOC_HIDE_ERRORS" 已经被设置，则将 `hide` 设置为所获取的数值。

  c. 如果 "HWLOC_DEBUG" 已经设置为1，并且上文中未设置 `HWLOC_DEBUG_VERBOSE`，则执行以下操作：

    d. 获取环境变量 "HWLOC_DEBUG_VERBOSE"，并检查是否已经设置。

    e. 如果 "HWLOC_DEBUG_VERBOSE" 已经被设置，则将 `hide` 设置为 0。否则，将 `hide` 设置为 1。

f. 如果 "HWLOC_DEBUG" 设置为 0，则将 `hide` 设置为 1，以显示所有调试信息。


```cpp
int hwloc_hide_errors(void)
{
  static int hide = 1; /* only show critical errors by default. lstopo will show others */
  static int checked = 0;
  if (!checked) {
    const char *envvar = getenv("HWLOC_HIDE_ERRORS");
    if (envvar) {
      hide = atoi(envvar);
#ifdef HWLOC_DEBUG
    } else {
      /* if debug is enabled and HWLOC_DEBUG_VERBOSE isn't forced to 0,
       * show all errors jus like we show all debug messages.
       */
      envvar = getenv("HWLOC_DEBUG_VERBOSE");
      if (!envvar || atoi(envvar))
        hide = 0;
```

这段代码是一个 C 语言中的函数，它有几个作用：

1. 定义了一个名为 "hide" 的变量，但并没有给它赋予任何实际值。
2. 将 "checked" 变量设置为 1。
3. 定义了一个名为 "report_insert_error_format_obj" 的函数，这个函数接受三个参数：
	* "buf"：一个字符数组，用于存储错误信息；
	* "buflen"：一个整数，用于存储 "buf" 数组长度；
	* "obj"：一个 hwloc_obj_t 类型的对象，用于获取插入的上下文信息。
	* 函数的作用是格式化 "obj" 对象的信息，将错误信息添加到 "buf" 中。

具体来说，函数内部首先定义了一个字符串类型变量 "typestr"，用于存储 "obj" 对象的类型信息。接着定义了一个字符串类型变量 "cpusetstr"，用于存储 "obj" 对象中所有 CPU 集的哈希值。然后，根据 "obj" 对象是否支持节点设置，定义了一个字符串类型变量 "nodesetstr"，用于存储 "obj" 对象中所有节点集的哈希值。最后，根据 "obj" 对象操作系统索引和 "cpuset" 和 "nodeset" 是否有变化，将错误信息添加到 "buf" 中。


```cpp
#endif
    }
    checked = 1;
  }
  return hide;
}


/* format the obj info to print in error messages */
static void
report_insert_error_format_obj(char *buf, size_t buflen, hwloc_obj_t obj)
{
  char typestr[64];
  char *cpusetstr;
  char *nodesetstr = NULL;

  hwloc_obj_type_snprintf(typestr, sizeof(typestr), obj, 0);
  hwloc_bitmap_asprintf(&cpusetstr, obj->cpuset);
  if (obj->nodeset) /* may be missing during insert */
    hwloc_bitmap_asprintf(&nodesetstr, obj->nodeset);
  if (obj->os_index != HWLOC_UNKNOWN_INDEX)
    snprintf(buf, buflen, "%s (P#%u cpuset %s%s%s)",
             typestr, obj->os_index, cpusetstr,
             nodesetstr ? " nodeset " : "",
             nodesetstr ? nodesetstr : "");
  else
    snprintf(buf, buflen, "%s (cpuset %s%s%s)",
             typestr, cpusetstr,
             nodesetstr ? " nodeset " : "",
             nodesetstr ? nodesetstr : "");
  free(cpusetstr);
  free(nodesetstr);
}

```

这段代码是一个静态函数，名为 report_insert_error，其作用是打印一条有关操作系统错误信息的警告消息，并报告任何成功插入新数据到哈威位置（hwloc）对象中时出现的无效信息。

具体来说，这段代码首先检查两个参数：一个是已插入的新数据（new）的对象，另一个是已存在的旧数据（old）的对象，以及一个字符串（reason）用于指定在操作系统中出现的错误信息。

接着，代码块内部使用 if 语句进行条件判断。如果条件为真，即操作系统中出现了错误信息，而且已经报告过这个错误，那么代码将执行以下操作：

1. 在屏幕上输出 "*************************************************************" 和当前哈威位置操作系统的版本信息；
2. 在屏幕上输出一条有关新数据无效信息的警告消息，其中包括新数据和旧数据的格式化信息；
3. 在屏幕上输出 "Failed with: " 和新数据的格式化信息，同时输出 "coming from: " 和旧数据的格式化信息；
4. 在屏幕上输出哈威位置操作系统的帮助文档中关于这个问题的链接；
5. 如果操作系统中未报告过这个错误，那么代码将报告这个错误信息，以便用户参考。

最后，代码返回一个静态整数类型的变量，表示成功报告了该错误信息，0表示失败。


```cpp
static void report_insert_error(hwloc_obj_t new, hwloc_obj_t old, const char *msg, const char *reason)
{
  static int reported = 0;

  if (reason && !reported && HWLOC_SHOW_CRITICAL_ERRORS()) {
    char newstr[512];
    char oldstr[512];
    report_insert_error_format_obj(newstr, sizeof(newstr), new);
    report_insert_error_format_obj(oldstr, sizeof(oldstr), old);

    fprintf(stderr, "****************************************************************************\n");
    fprintf(stderr, "* hwloc %s received invalid information from the operating system.\n", HWLOC_VERSION);
    fprintf(stderr, "*\n");
    fprintf(stderr, "* Failed with: %s\n", msg);
    fprintf(stderr, "* while inserting %s at %s\n", newstr, oldstr);
    fprintf(stderr, "* coming from: %s\n", reason);
    fprintf(stderr, "*\n");
    fprintf(stderr, "* The following FAQ entry in the hwloc documentation may help:\n");
    fprintf(stderr, "*   What should I do when hwloc reports \"operating system\" warnings?\n");
    fprintf(stderr, "* Otherwise please report this error message to the hwloc user's mailing list,\n");
```

这段代码是一个用于输出在 hwloc-gather-topology 脚本中定义的系统调用和与之相关的topology信息的代码。具体来说，代码的作用是输出以下内容：

```cpp
* 如果在定义了 HWLOC_LINUX_SYS 头文件的情况下，输出一些关于系统调用和文件的声明。
* 否则输出一些关于topology信息的声明，并提示 hwloc 将忽略任何无效的topology信息，并继续输出。
* 输出 "hwloc will now ignore this invalid topology information and continue."。
* 输出 "****************************************************************************"。
* 如果 hwloc-gather-topology 脚本中定义了 HUSE_SYSCTLBYNAME 函数，则调用该函数来获取指定系统调用对应的 topology 信息，并输出。
```
具体来说，代码首先定义了一个名为 "hwloc-gather-topology.c" 的头文件，其中包含了一系列用于输出 topology 信息的函数和定义。然后，在主函数中，先判断是否定义了 HWLOC_LINUX_SYS 头文件，如果没有，则输出一些与文件和系统相关的信息。如果已经定义了头文件，则调用 hwloc-gather-topology.c 中的函数来获取 topology 信息，并输出一些信息。如果 topology 信息存在问题，则输出一条错误消息并跳过该错误。最后，通过一系列输出，告诉用户 hwloc 已经准备好继续执行topology 操作。


```cpp
#ifdef HWLOC_LINUX_SYS
    fprintf(stderr, "* along with the files generated by the hwloc-gather-topology script.\n");
#else
    fprintf(stderr, "* along with any relevant topology information from your platform.\n");
#endif
    fprintf(stderr, "* \n");
    fprintf(stderr, "* hwloc will now ignore this invalid topology information and continue.\n");
    fprintf(stderr, "****************************************************************************\n");
    reported = 1;
  }
}

#if defined(HAVE_SYSCTLBYNAME)
int hwloc_get_sysctlbyname(const char *name, int64_t *ret)
{
  union {
    int32_t i32;
    int64_t i64;
  } n;
  size_t size = sizeof(n);
  if (sysctlbyname(name, &n, &size, NULL, 0))
    return -1;
  switch (size) {
    case sizeof(n.i32):
      *ret = n.i32;
      break;
    case sizeof(n.i64):
      *ret = n.i64;
      break;
    default:
      return -1;
  }
  return 0;
}
```

这段代码是一个 C 语言函数，名为 `hwloc_get_sysctl()`，定义在 `hwloc.h` 文件中。它用于获取指定系统调用 `sysctl()` 的返回值。

具体来说，这段代码实现了一个名为 `hwloc_get_sysctl()` 的函数，它接受三个参数：一个字符型数组 `name`，一个字符型字数组 `namelen`，以及一个指向整型或整型字段的 `ret` 指针，分别用于表示取得的服务器硬件配置和设置的返回值和参数值。

函数实现中，首先定义了一个名为 `n` 的联合体，用于存储服务器硬件配置或设置的值。接着定义了一个 `size_t` 类型的变量 `size`，用于存储 `name` 和 `namelen` 组成的字符串长度。

接下来，函数实现了一个 `if` 语句，判断 `sysctl()` 函数是否支持指定的系统调用，并返回相应的结果。如果支持，则实现了一个 switch 语句，根据 `size` 变量的值选择执行哪个系统调用，并取得对应的返回值。如果 `size` 不确定或者系统调用不支持，函数返回一个负数表示失败。

最后，函数返回 0，表示成功执行了系统调用并取得了正确的返回值。


```cpp
#endif

#if defined(HAVE_SYSCTL)
int hwloc_get_sysctl(int name[], unsigned namelen, int64_t *ret)
{
  union {
    int32_t i32;
    int64_t i64;
  } n;
  size_t size = sizeof(n);
  if (sysctl(name, namelen, &n, &size, NULL, 0))
    return -1;
  switch (size) {
    case sizeof(n.i32):
      *ret = n.i32;
      break;
    case sizeof(n.i64):
      *ret = n.i64;
      break;
    default:
      return -1;
  }
  return 0;
}
```

这段代码是一个头文件，其中包含一个函数 `hwloc_fallback_nbprocessors()`。

这个函数的作用是返回操作系统提供的处理器数量，即使topology->is_thissystem为假(即这个系统不是windows)，也仍然假设topology->is_thissystem为真，以便在将来需要支持在线下CPU的系统上运行。

函数的实现基于两个条件：

1. 如果定义了`HWLOC_FALLBACK_NBPROCESSORS_INCLUDE_OFFLINE`位，那么函数将会尝试获取一个包含所有可用的CPU的数组，即使这些CPU不能在线下，也可能来自包含多个CPU的物理机器。
2. 如果定义了`HWLOC_FALLBACK_CB`宏，那么函数将会尝试从系统调用中获得当前系统的处理器数量。这个函数在windows实现中由`topology-windows.c`文件提供。

总结起来，这个函数是一个倒入函数，它返回操作系统提供的处理器数量，以便在需要支持在线下CPU的系统上运行。


```cpp
#endif

/* Return the OS-provided number of processors.
 * Assumes topology->is_thissystem is true.
 */
#ifndef HWLOC_WIN_SYS /* The windows implementation is in topology-windows.c */
int
hwloc_fallback_nbprocessors(unsigned flags) {
  int n;

  if (flags & HWLOC_FALLBACK_NBPROCESSORS_INCLUDE_OFFLINE) {
    /* try to get all CPUs for Linux and Solaris that can handle offline CPUs */
#if HAVE_DECL__SC_NPROCESSORS_CONF
    n = sysconf(_SC_NPROCESSORS_CONF);
#elif HAVE_DECL__SC_NPROC_CONF
    n = sysconf(_SC_NPROC_CONF);
```

这段代码是一个条件分支，其作用是判断给定的变量n是否为-1，如果是，则执行下面的代码，否则不执行。

具体来说，如果给定的n不是-1，则说明系统支持多核处理器，于是就尝试使用系统变量_SC_NPROCESSORS_ONLN或者_SC_NPROC_ONLN获取在线CPU的数量。如果这两个系统变量都无法获取，则执行下一个语句，即使用系统变量_SC_NPROCESSORS_CONF或者_SC_NPROC_CONF获取当前系统的多核处理器数量。如果这两个系统变量也都无法获取，则执行最后的语句，即输出-1，表明系统不支持多核处理器。

这段代码可以用来获取在线CPU的数量，并且在一些Linux发行版中可能需要使用系统调用来获取该值。


```cpp
#else
    n = -1;
#endif
    if (n != -1)
      return n;
  }

  /* try getting only online CPUs, or whatever we can get */
#if HAVE_DECL__SC_NPROCESSORS_ONLN
  n = sysconf(_SC_NPROCESSORS_ONLN);
#elif HAVE_DECL__SC_NPROC_ONLN
  n = sysconf(_SC_NPROC_ONLN);
#elif HAVE_DECL__SC_NPROCESSORS_CONF
  n = sysconf(_SC_NPROCESSORS_CONF);
#elif HAVE_DECL__SC_NPROC_CONF
  n = sysconf(_SC_NPROC_CONF);
```

这段代码检查系统是否支持输出主机基本信息，包括CPU数量。它首先检查定义是否为真（通过`#elif`条件语句），然后检查是否支持输出主机基本信息。如果是，它定义了一个`info`结构体，然后调用`host_info`函数获取它所支持的最大CPU数（通过`count`参数），最后通过`n`变量获取CPU数量。如果不是，则表示操作系统不支持获取主机基本信息，返回-1。


```cpp
#elif defined(HAVE_HOST_INFO) && HAVE_HOST_INFO
  struct host_basic_info info;
  mach_msg_type_number_t count = HOST_BASIC_INFO_COUNT;
  host_info(mach_host_self(), HOST_BASIC_INFO, (integer_t*) &info, &count);
  n = info.avail_cpus;
#elif defined(HAVE_SYSCTLBYNAME)
  int64_t nn;
  if (hwloc_get_sysctlbyname("hw.ncpu", &nn))
    nn = -1;
  n = nn;
#elif defined(HAVE_SYSCTL) && HAVE_DECL_CTL_HW && HAVE_DECL_HW_NCPU
  static int name[2] = {CTL_HW, HW_NCPU};
  int64_t nn;
  if (hwloc_get_sysctl(name, sizeof(name)/sizeof(*name), &nn))
    n = -1;
  n = nn;
```

这段代码是一个C语言中的函数，名为`hwloc_fallback_memsize`，定义在`hwloc_ fallback_memsize.h`文件中。它的作用是返回一个整数类型的`int64_t`类型的系统内存最大值，同时如果系统不支持`hwloc_fallback_memsize`函数，将会输出一个警告信息。

具体来说，这段代码首先定义了一个名为`n`的整数变量，其初始值为-1。然后通过`#ifdef`和`#elif`语句判断系统是否支持`hwloc_fallback_memsize`函数。如果系统不支持该函数，将会输出一个警告信息，其内容将基于`HAVE_HOST_INFO`和`HAVE_HOST_INFO`两个预设标志中的任意一个。

接着，在`#elif`语句中，通过`mach_msg_type_number_t`类型变量`count`来获取`host_basic_info`结构中与系统内存相关的信息，然后通过`host_info`函数获取该信息，最后通过`info.memory_size`获取到系统内存的最大值。最后，将`size`的值赋给整数变量`n`，并返回`size`的值作为最终结果。


```cpp
#else
#ifdef __GNUC__
#warning No known way to discover number of available processors on this system
#endif
  n = -1;
#endif
  return n;
}

int64_t
hwloc_fallback_memsize(void) {
  int64_t size;
#if defined(HAVE_HOST_INFO) && HAVE_HOST_INFO
  struct host_basic_info info;
  mach_msg_type_number_t count = HOST_BASIC_INFO_COUNT;
  host_info(mach_host_self(), HOST_BASIC_INFO, (integer_t*) &info, &count);
  size = info.memory_size;
```

这段代码定义了一系列哈希表（Hash Table）以实现对系统控制卡（SYSCTL）的访问控制。哈希表的键（key）是描述硬件内存（HWMEM）的参数，值为相应的索引，用于访问特定硬件内存区域。

具体来说，这段代码定义了以下哈希表：

1. `name[2]` 存储系统控制卡中的 `CTL_HW` 和 `HW_MEMSIZE64` 两个常量。这两个常量分别表示控制卡的名称（如： ctl）和硬件内存大小（如： memsize64）。
2. `name[2]` 存储系统控制卡中的 `CTL_HW` 和 `HW_REALMEM64` 两个常量。这两个常量分别表示控制卡的名称（如： ctl）和硬件内存大小（如： remem64）。
3. `name[2]` 存储系统控制卡中的 `CTL_HW` 和 `HW_PHYSMEM64` 两个常量。这两个常量分别表示控制卡的名称（如： ctl）和硬件内存类型（如： memsize64）。
4. `name[2]` 存储系统控制卡中的 `CTL_HW` 和 `HW_USERMEM64` 两个常量。这两个常量分别表示控制卡的名称（如： ctl）和用户内存大小（如： usermem64）。
5. `name[2]` 存储系统控制卡中的 `CTL_HW` 和 `HW_MEMSIZE` 两个常量。这两个常量分别表示控制卡的名称（如： ctl）和硬件内存大小（如： memsize）。
6. `name[2]` 存储系统控制卡中的 `CTL_HW` 和 `HW_REALMEM` 两个常量。这两个常量分别表示控制卡的名称（如： ctl）和硬件内存大小（如： remem）。
7. `name[2]` 存储系统控制卡中的 `CTL_HW` 和 `HW_PHYSMEM` 两个常量。这两个常量分别表示控制卡的名称（如： ctl）和硬件内存类型（如： memsize）。

`HAVE_SYSCTL`、`HAVE_DECL_CTL_HW` 和 `HAVE_DECL_HW_MEMSIZE64` 是系统控制卡的定义，而 `HAVE_DECL_HW_REALMEM64` 等则不是。


```cpp
#elif defined(HAVE_SYSCTL) && HAVE_DECL_CTL_HW && (HAVE_DECL_HW_REALMEM64 || HAVE_DECL_HW_MEMSIZE64 || HAVE_DECL_HW_PHYSMEM64 || HAVE_DECL_HW_USERMEM64 || HAVE_DECL_HW_REALMEM || HAVE_DECL_HW_MEMSIZE || HAVE_DECL_HW_PHYSMEM || HAVE_DECL_HW_USERMEM)
#if HAVE_DECL_HW_MEMSIZE64
  static int name[2] = {CTL_HW, HW_MEMSIZE64};
#elif HAVE_DECL_HW_REALMEM64
  static int name[2] = {CTL_HW, HW_REALMEM64};
#elif HAVE_DECL_HW_PHYSMEM64
  static int name[2] = {CTL_HW, HW_PHYSMEM64};
#elif HAVE_DECL_HW_USERMEM64
  static int name[2] = {CTL_HW, HW_USERMEM64};
#elif HAVE_DECL_HW_MEMSIZE
  static int name[2] = {CTL_HW, HW_MEMSIZE};
#elif HAVE_DECL_HW_REALMEM
  static int name[2] = {CTL_HW, HW_REALMEM};
#elif HAVE_DECL_HW_PHYSMEM
  static int name[2] = {CTL_HW, HW_PHYSMEM};
```

这段代码是一个C语言程序，它定义了一个名为`name`的静态整型数组，其值为`CTL_HW`和`HW_USERMEM`。数组长度为`sizeof(name)/sizeof(*name)`，即`2`。

接下来，程序根据`hwloc_get_sysctl`函数的返回值，判断是否支持使用系统调用`sysctl`。如果不支持，程序将`size`设置为`-1`。如果支持，程序使用`hwloc_get_sysctlbyname`函数，根据指定的硬件内存使用器名称，尝试获取相应的物理内存大小。如果所有尝试的名称返回的值都是有效的，则程序返回物理内存大小。否则，程序将`size`设置为`-1`。

最后，程序返回上面计算得到的物理内存大小。


```cpp
#elif HAVE_DECL_HW_USERMEM
  static int name[2] = {CTL_HW, HW_USERMEM};
#endif
  if (hwloc_get_sysctl(name, sizeof(name)/sizeof(*name), &size))
    size = -1;
#elif defined(HAVE_SYSCTLBYNAME)
  if (hwloc_get_sysctlbyname("hw.memsize", &size) &&
      hwloc_get_sysctlbyname("hw.realmem", &size) &&
      hwloc_get_sysctlbyname("hw.physmem", &size) &&
      hwloc_get_sysctlbyname("hw.usermem", &size))
      size = -1;
#else
  size = -1;
#endif
  return size;
}
```

这段代码是用来设置硬件加速器（HWLOC）中的PU（处理器单元）级别的设置。代码中包含两个循环，用于遍历所有的CPU和创建CPU设置对象。

在循环过程中，首先定义一个名为obj的硬件对象，然后定义一个名为oscpu的局部变量，用于跟踪当前正在设置的CPU。接下来，创建一个名为obj->cpuset的硬件对象，并使用hwloc_bitmap_alloc()函数来分配一个字节码映射，该映射将对应于每个CPU的设置。然后，将该设置对象仅设置为当前CPU的设置，并输出相关信息。

接着，进行循环，以将所有CPU设置为当前PU的设置，并输出设置的对象名称。在循环内部，可以使用hwloc__insert_object_by_cpuset()函数将设置对象插入到topology链中，以便在运行时进行查询。


```cpp
#endif /* !HWLOC_WIN_SYS */

/*
 * Use the given number of processors to set a PU level.
 */
void
hwloc_setup_pu_level(struct hwloc_topology *topology,
		     unsigned nb_pus)
{
  struct hwloc_obj *obj;
  unsigned oscpu,cpu;

  hwloc_debug("%s", "\n\n * CPU cpusets *\n\n");
  for (cpu=0,oscpu=0; cpu<nb_pus; oscpu++)
    {
      obj = hwloc_alloc_setup_object(topology, HWLOC_OBJ_PU, oscpu);
      obj->cpuset = hwloc_bitmap_alloc();
      hwloc_bitmap_only(obj->cpuset, oscpu);

      hwloc_debug_2args_bitmap("cpu %u (os %u) has cpuset %s\n",
		 cpu, oscpu, obj->cpuset);
      hwloc__insert_object_by_cpuset(topology, NULL, obj, "core:pulevel");

      cpu++;
    }
}

```

这两行代码定义了两个名为`for_each_child_safe`和`for_each_memory_child_safe`的宏，用于遍历一个parent中的子元素，并确保在遍历过程中不会访问越界或释放资源。

在`for_each_child_safe`中，首先定义了一个名为`pchild`的指针变量，该变量指向`parent`的第一个子元素，然后通过循环遍历该指针所指向的子元素，每次将子元素的值存储在`child`中，接着检查当前子元素是否被删除，如果是，则将`pchild`指向该子元素的下一个子元素，否则将`child`指向下一个子元素。最后，该宏还定义了一个名为`child`的指针变量，用于访问当前子元素。

在`for_each_memory_child_safe`中，定义了与`for_each_child_safe`类似的宏，但使用的是`memory_first_child`代替了`first_child`。该宏中，首先定义了一个名为`pchild`的指针变量，该变量指向`parent`的第一个子元素，然后通过循环遍历该指针所指向的子元素，每次将子元素的值存储在`child`中，接着检查当前子元素是否被删除，如果是，则将`pchild`指向该子元素的下一个子元素，否则将`child`指向下一个子元素。最后，该宏还定义了一个名为`child`的指针变量，用于访问当前子元素。


```cpp
/* Traverse children of a parent in a safe way: reread the next pointer as
 * appropriate to prevent crash on child deletion:  */
#define for_each_child_safe(child, parent, pchild) \
  for (pchild = &(parent)->first_child, child = *pchild; \
       child; \
       /* Check whether the current child was not dropped.  */ \
       (*pchild == child ? pchild = &(child->next_sibling) : NULL), \
       /* Get pointer to next child.  */ \
        child = *pchild)
#define for_each_memory_child_safe(child, parent, pchild) \
  for (pchild = &(parent)->memory_first_child, child = *pchild; \
       child; \
       /* Check whether the current child was not dropped.  */ \
       (*pchild == child ? pchild = &(child->next_sibling) : NULL), \
       /* Get pointer to next child.  */ \
        child = *pchild)
```

这两行代码定义了两个名为`for_each_io_child_safe`和`for_each_misc_child_safe`的函数，它们都接受三个参数：`child`、`parent`和`pchild`。

这两个函数的目的是分别遍历`parent`中的`io_first_child`和`misc_first_child`，并获取它们的下一个子节点，然后将当前节点加入到`pchild`中。通过双重循环，函数首先检查当前节点是否已经被删除，如果是，则将`pchild`指向下一个兄弟节点，如果不是，则将`child`指向下一个子节点。

具体来说，这两行代码可以看作是在遍历`parent`中的子节点，并将其复制到`pchild`中，以使得`parent`中的每个子节点都有对应的`pchild`。


```cpp
#define for_each_io_child_safe(child, parent, pchild) \
  for (pchild = &(parent)->io_first_child, child = *pchild; \
       child; \
       /* Check whether the current child was not dropped.  */ \
       (*pchild == child ? pchild = &(child->next_sibling) : NULL), \
       /* Get pointer to next child.  */ \
        child = *pchild)
#define for_each_misc_child_safe(child, parent, pchild) \
  for (pchild = &(parent)->misc_first_child, child = *pchild; \
       child; \
       /* Check whether the current child was not dropped.  */ \
       (*pchild == child ? pchild = &(child->next_sibling) : NULL), \
       /* Get pointer to next child.  */ \
        child = *pchild)

```

This is a C function that takes an object file and its attributes and
snprintf(/* format string */, ...) the object file's attributes, including its type, index,
subtype, cpuset, nodeset, complete_cpuset, and complete_nodeset. It also
supports the addition of a object's name and subtype, as well as its arity. The
object file should have an "object" header that includes the
object's index and a "time" header that includes the object's creation
time.

The function takes a single argument, which is the object file, and
retorts a string containing the object's attributes formatted as a
"Verse Bold名著： [object file name]
[object file attributes]". This string will be used on the
output when the object is printed to the console or terminal.

The function first initializes some variables to be used in the
snprintf() calls that follow, such as the width of the output, the current
indentation level, and a character buffer for the attribute name.

Next, the function loops through the object file's attributes,
snprintf() each attribute with the appropriate indentation level and format
the attribute as necessary using the hwloc_obj_attr_snprintf() function.

For each attribute, the function either adds its own
object name or subtype, or sets a character buffer for the attribute
name using hwloc_obj_attr_snprintf() with the appropriate format string.

Finally, the function includes some formatting and indentation
lines to add some formatting and indentation to the output,
following the example usage of the function provided in the


```cpp
#ifdef HWLOC_DEBUG
/* Just for debugging.  */
static void
hwloc_debug_print_object(int indent __hwloc_attribute_unused, hwloc_obj_t obj)
{
  char type[64], idx[12], attr[1024], *cpuset = NULL;
  hwloc_debug("%*s", 2*indent, "");
  hwloc_obj_type_snprintf(type, sizeof(type), obj, 1);
  if (obj->os_index != HWLOC_UNKNOWN_INDEX)
    snprintf(idx, sizeof(idx), "#%u", obj->os_index);
  else
    *idx = '\0';
  hwloc_obj_attr_snprintf(attr, sizeof(attr), obj, " ", 1);
  hwloc_debug("%s%s%s%s%s", type, idx, *attr ? "(" : "", attr, *attr ? ")" : "");
  if (obj->name)
    hwloc_debug(" name \"%s\"", obj->name);
  if (obj->subtype)
    hwloc_debug(" subtype \"%s\"", obj->subtype);
  if (obj->cpuset) {
    hwloc_bitmap_asprintf(&cpuset, obj->cpuset);
    hwloc_debug(" cpuset %s", cpuset);
    free(cpuset);
  }
  if (obj->complete_cpuset) {
    hwloc_bitmap_asprintf(&cpuset, obj->complete_cpuset);
    hwloc_debug(" complete %s", cpuset);
    free(cpuset);
  }
  if (obj->nodeset) {
    hwloc_bitmap_asprintf(&cpuset, obj->nodeset);
    hwloc_debug(" nodeset %s", cpuset);
    free(cpuset);
  }
  if (obj->complete_nodeset) {
    hwloc_bitmap_asprintf(&cpuset, obj->complete_nodeset);
    hwloc_debug(" completeN %s", cpuset);
    free(cpuset);
  }
  if (obj->arity)
    hwloc_debug(" arity %u", obj->arity);
  hwloc_debug("%s", "\n");
}

```

这段代码定义了一个名为 `hwloc_debug_print_objects` 的静态函数，它的参数是一个整型变量 `indent` 和一个 `hwloc_obj_t` 类型的对象 `obj`。函数内部使用 `hwloc_obj_t` 类型，说明它是一个 `hwloc_object` 类型的别名，可能是为了和 `hwloc_debug_print_object` 函数区分。

函数的作用是打印指定对象的路径偏移和各个子对象的路径偏移，并使用不同格式化级别。具体来说，当 `hwloc_debug_enabled()` 不小于 2 时，打印的格式包括：

1. 打印对象本身，使用 `hwloc_obj_t` 的 `print_object` 函数；
2. 打印对象的子对象，使用 `hwloc_debug_print_objects` 函数，格式为 `indent + 1`，递归调用自身；
3. 打印对象的 I/O 子对象，使用 `hwloc_debug_print_objects` 函数，格式为 `indent + 1`，递归调用自身；
4. 打印对象的内存子对象，使用 `hwloc_debug_print_objects` 函数，格式为 `indent + 1`，递归调用自身；
5. 打印对象的其它子对象（包括 `hwloc_object` 和 `hwloc_string` 类型），格式为 `indent + 1`，递归调用自身。


```cpp
static void
hwloc_debug_print_objects(int indent __hwloc_attribute_unused, hwloc_obj_t obj)
{
  if (hwloc_debug_enabled() >= 2) {
    hwloc_obj_t child;
    hwloc_debug_print_object(indent, obj);
    for_each_child (child, obj)
      hwloc_debug_print_objects(indent + 1, child);
    for_each_memory_child (child, obj)
      hwloc_debug_print_objects(indent + 1, child);
    for_each_io_child (child, obj)
      hwloc_debug_print_objects(indent + 1, child);
    for_each_misc_child (child, obj)
      hwloc_debug_print_objects(indent + 1, child);
  }
}
```

这段代码定义了两个头文件：hwloc_debug_print_object和hwloc_debug_print_objects。这两个头文件都包含一个do-while循环，内部还有一个if语句。

如果定义了HWLOC_DEBUG，这两个头文件才会被编译和输出。否则，它们不会被编译，也不会输出任何内容。

hwloc_free_infos是一个函数，它接收一个hwloc_info_s结构体类型的参数infos，以及一个整数count。这个函数用于释放给定的infos结构体中的name和value成员。


```cpp
#else /* !HWLOC_DEBUG */
#define hwloc_debug_print_object(indent, obj) do { /* nothing */ } while (0)
#define hwloc_debug_print_objects(indent, obj) do { /* nothing */ } while (0)
#endif /* !HWLOC_DEBUG */

void hwloc__free_infos(struct hwloc_info_s *infos, unsigned count)
{
  unsigned i;
  for(i=0; i<count; i++) {
    free(infos[i].name);
    free(infos[i].value);
  }
  free(infos);
}

```

这段代码定义了一个名为"hwloc__add_info"的函数，其作用是向一个"hwloc_info_s"结构体数组中添加一个名为"info"的元素。该函数接受三个参数：一个指向"hwloc_info_s"结构体数组的指针 infosp，一个指向整数的 countp，以及两个字符串参数 name 和 value，分别表示要添加的名称和值。函数的主要部分如下：

1. 首先定义了一个名为 count 的整数变量，用于保存添加新元素后数组中元素的个数，即新元素个数。

2. 接着定义了一个名为 infos 的指向 "hwloc_info_s" 结构体数组的指针变量，用于保存数组元素的地址。

3. 定义了一个名为 ObjectInfoAlloc 的宏，表示定义了一个允许使用该宏的变量为 8。这个宏将会在定义的函数中用于计算允许分配给数组的元素数量。

4. 计算允许分配的元素数量后，如果计算得到的 count 不等，则执行以下操作：首先尝试从数组中分配一块内存，如果内存分配失败，则执行一个带参数 out_with_array 的局部函数，该函数将输出错误信息并跳转到 out_with_name 处。然后尝试从数组中分配元素，如果尝试成功，则执行以下操作：将 infos 指向分配的内存，并将 count 自增，最后将 *countp 指向当前元素个数。

5. 定义了一个名为 out_with_name 的局部函数，用于在尝试从数组中分配元素时，成功创建并设置新元素的名称，然后释放内存。

6. 定义了一个名为 out_with_array 的局部函数，用于在尝试从数组中分配元素时，输出错误信息。

7. 在函数中主要执行以下操作：首先，根据 ObjectInfoAlloc 宏计算允许分配的元素数量，然后创建一个新的 "hwloc_info_s" 结构体元素，并将它的 name 和 value 分别设置为传入的名称和值。接着，如果尝试从数组中分配内存失败，则输出错误信息并跳转到 out_with_array 处。如果尝试成功，则设置 count 自增，并将 infos 指向分配的内存，最后将 *countp 指向当前元素个数，以便更新数组中元素的引用。


```cpp
int hwloc__add_info(struct hwloc_info_s **infosp, unsigned *countp, const char *name, const char *value)
{
  unsigned count = *countp;
  struct hwloc_info_s *infos = *infosp;
#define OBJECT_INFO_ALLOC 8
  /* nothing allocated initially, (re-)allocate by multiple of 8 */
  unsigned alloccount = (count + 1 + (OBJECT_INFO_ALLOC-1)) & ~(OBJECT_INFO_ALLOC-1);
  if (count != alloccount) {
    struct hwloc_info_s *tmpinfos = realloc(infos, alloccount*sizeof(*infos));
    if (!tmpinfos)
      /* failed to allocate, ignore this info */
      goto out_with_array;
    *infosp = infos = tmpinfos;
  }
  infos[count].name = strdup(name);
  if (!infos[count].name)
    goto out_with_array;
  infos[count].value = strdup(value);
  if (!infos[count].value)
    goto out_with_name;
  *countp = count+1;
  return 0;

 out_with_name:
  free(infos[count].name);
 out_with_array:
  /* don't bother reducing the array */
  return -1;
}

```

这段代码是一个名为`hwloc__add_info_nodup`的函数，它接受一个指向结构体`hwloc_info_s`的指针`infosp`，一个指向整数的变量`countp`，以及一个字符串`name`和一个字符串`value`，函数的作用是判断给定的名字和值是否与已有信息中的名字和值相同，如果是，则替换掉已有的值，并返回0；如果不是，则返回-1。

具体实现中，首先将`infosp`和`countp`指向的信息结构体和计数值存入`infos`和`count`中，然后遍历`infos`数组，对于每个`hwloc_info_s`结构体，首先比较它名字和给定的`name`是否相同，如果相同，则检查`replace`是否为真，如果是，则用新的值替换已有的值，并释放旧的值，最后将`hwloc_add_info`函数返回，返回0表示成功，-1表示失败。


```cpp
int hwloc__add_info_nodup(struct hwloc_info_s **infosp, unsigned *countp,
			  const char *name, const char *value,
			  int replace)
{
  struct hwloc_info_s *infos = *infosp;
  unsigned count = *countp;
  unsigned i;
  for(i=0; i<count; i++) {
    if (!strcmp(infos[i].name, name)) {
      if (replace) {
	char *new = strdup(value);
	if (!new)
	  return -1;
	free(infos[i].value);
	infos[i].value = new;
      }
      return 0;
    }
  }
  return hwloc__add_info(infosp, countp, name, value);
}

```

This is a function that seems to be managing the allocation of object information in a large-scale distributed storage system. The function takes in a few parameters, including the memory regions associated with the object information (`dst_infos` and `src_infos`), the current object information allocation count (`dst_count`), and the maximum number of object information that can be allocated (`OBJECT_INFO_ALLOC`).

The function first checks if the memory regions associated with the object information are already allocated, and if not, it dynamically allocates the memory regions using the `realloc` function.

Then, the function iterates through the current object information and copies its name and value to the corresponding memory region associated with the object information in the `dst_infos` array.

Finally, the function sets the pointer to the current object information allocation count to `dst_count` and returns 0 to indicate success. If the allocation fails, the function returns `-1` to indicate failure.

Note that the function does not handle the case where the `OBJECT_INFO_ALLOC` parameter is not defined or is too large, which could cause the program to crash or behave unexpectedly.


```cpp
int hwloc__move_infos(struct hwloc_info_s **dst_infosp, unsigned *dst_countp,
		      struct hwloc_info_s **src_infosp, unsigned *src_countp)
{
  unsigned dst_count = *dst_countp;
  struct hwloc_info_s *dst_infos = *dst_infosp;
  unsigned src_count = *src_countp;
  struct hwloc_info_s *src_infos = *src_infosp;
  unsigned i;
#define OBJECT_INFO_ALLOC 8
  /* nothing allocated initially, (re-)allocate by multiple of 8 */
  unsigned alloccount = (dst_count + src_count + (OBJECT_INFO_ALLOC-1)) & ~(OBJECT_INFO_ALLOC-1);
  if (dst_count != alloccount) {
    struct hwloc_info_s *tmp_infos = realloc(dst_infos, alloccount*sizeof(*dst_infos));
    if (!tmp_infos)
      /* Failed to realloc, ignore the appended infos */
      goto drop;
    dst_infos = tmp_infos;
  }
  for(i=0; i<src_count; i++, dst_count++) {
    dst_infos[dst_count].name = src_infos[i].name;
    dst_infos[dst_count].value = src_infos[i].value;
  }
  *dst_infosp = dst_infos;
  *dst_countp = dst_count;
  free(src_infos);
  *src_infosp = NULL;
  *src_countp = 0;
  return 0;

 drop:
  /* drop src infos, don't modify dst_infos at all */
  for(i=0; i<src_count; i++) {
    free(src_infos[i].name);
    free(src_infos[i].value);
  }
  free(src_infos);
  *src_infosp = NULL;
  *src_countp = 0;
  return -1;
}

```

这两函数的主要作用是管理 `hwloc_tma` 结构体中的 `info` 成员变量。

`hwloc_obj_add_info` 函数接受一个 `hwloc_obj_t` 类型的对象以及一个指向字符串的指针和一个字符串，它代表了一个 `hwloc_info_s` 结构体。函数首先调用 `hwloc__add_info` 函数，然后返回调用结果。如果调用成功，该函数将返回 0；如果调用失败，该函数将返回 -1。

`hwloc__tma_dup_infos` 函数接受一个 `hwloc_tma` 类型的实例，一个指向 `hwloc_info_s` 结构体的指针，一个整数和一个整数。函数首先调用 `hwloc_tma_calloc` 函数，以分配足够的内存空间用于存储新的 `hwloc_info_s` 结构体。然后，函数遍历旧的 `hwloc_info_s` 结构体，将每个旧的 `name` 和 `value` 字段复制到新分配的内存中。最后，函数将新分配的内存返回给调用者，如果没有内存泄漏，则返回 0。如果调用失败，例如分配内存失败，函数将返回 -1。


```cpp
int hwloc_obj_add_info(hwloc_obj_t obj, const char *name, const char *value)
{
  return hwloc__add_info(&obj->infos, &obj->infos_count, name, value);
}

/* This function may be called with topology->tma set, it cannot free() or realloc() */
int hwloc__tma_dup_infos(struct hwloc_tma *tma,
                         struct hwloc_info_s **newip, unsigned *newcp,
                         struct hwloc_info_s *oldi, unsigned oldc)
{
  struct hwloc_info_s *newi;
  unsigned i, j;
  newi = hwloc_tma_calloc(tma, oldc * sizeof(*newi));
  if (!newi)
    return -1;
  for(i=0; i<oldc; i++) {
    newi[i].name = hwloc_tma_strdup(tma, oldi[i].name);
    newi[i].value = hwloc_tma_strdup(tma, oldi[i].value);
    if (!newi[i].name || !newi[i].value)
      goto failed;
  }
  *newip = newi;
  *newcp = oldc;
  return 0;

 failed:
  assert(!tma || !tma->dontfree); /* this tma cannot fail to allocate */
  for(j=0; j<=i; j++) {
    free(newi[i].name);
    free(newi[i].value);
  }
  free(newi);
  *newip = NULL;
  return -1;
}

```

这段代码是一个名为 `hwloc__free_object_contents` 的函数，它是 `hwloc` 库中的一个函数。

这段代码的作用是释放一个 `hwloc_obj_t` 类型的对象，它包含了一个 `HWLOC_OBJ_NUMANODE` 类型的成员变量 `obj`。函数会在释放对象时执行一些操作，包括：

1. 如果 `obj` 的类型是 `HWLOC_OBJ_NUMANODE`，函数会释放它所有的 `numanode.page_types` 成员。
2. 否则，函数会释放 `obj` 的 `infos` 成员及其相关的 `infos_count` 成员。
3. 函数会释放 `obj` 的 `attr` 成员及其相关的 `free_attr` 函数指针。
4. 函数会释放 `obj` 的 `children` 成员及其相关的 `free_children` 函数指针。
5. 函数会释放 `obj` 的 `subtype` 成员。
6. 函数会释放 `obj` 的 `name` 成员。
7. 函数会释放 `obj` 所在的 `cpuset` 中的 `hwloc_bitmap_free` 函数。
8. 函数会释放 `obj` 所在的 `complete_cpuset` 中的 `hwloc_bitmap_free` 函数。
9. 函数会释放 `obj` 所在的 `nodeset` 中的 `hwloc_bitmap_free` 函数。
10. 函数会释放 `obj` 所在的 `complete_nodeset` 中的 `hwloc_bitmap_free` 函数。




```cpp
static void
hwloc__free_object_contents(hwloc_obj_t obj)
{
  switch (obj->type) {
  case HWLOC_OBJ_NUMANODE:
    free(obj->attr->numanode.page_types);
    break;
  default:
    break;
  }
  hwloc__free_infos(obj->infos, obj->infos_count);
  free(obj->attr);
  free(obj->children);
  free(obj->subtype);
  free(obj->name);
  hwloc_bitmap_free(obj->cpuset);
  hwloc_bitmap_free(obj->complete_cpuset);
  hwloc_bitmap_free(obj->nodeset);
  hwloc_bitmap_free(obj->complete_nodeset);
}

```



这段代码定义了两个函数，hwloc_free_unlinked_object和hwloc_replace_linked_object。

hwloc_free_unlinked_object函数的作用是释放一个不再被任何其他hwloc_obj_t引用到的对象，并将其内容和指针 free掉。

hwloc_replace_linked_object函数的作用是复制一个已经存在的对象的新内容和指针，使得新对象可以被hwloc_free_unlinked_object函数释放。新旧对象和新旧指针都会被复制。新旧对象的树指针会被复制，并更新对象的父节点和子节点。新旧对象的内存和 io 指针也会被复制，并更新对象的树深度和总内存使用量。

hwloc_replace_linked_object函数的实现中，首先通过hwloc__free_object_contents函数释放旧对象的内容和指针，然后使用free函数释放旧对象本身。接着，通过memcpy函数将新对象的内容复制到旧对象中，最后使用memset函数将新对象初始化为0，以便于hwloc_free_unlinked_object函数进行内存对齐。

hwloc_free_unlinked_object函数和hwloc_replace_linked_object函数的作用是不同的，hwloc_free_unlinked_object函数主要用于释放不再被任何其他hwloc_obj_t引用到的对象，而hwloc_replace_linked_object函数主要用于复制一个已经存在的对象的新内容和指针，使得新对象可以被hwloc_free_unlinked_object函数释放。


```cpp
/* Free an object and all its content.  */
void
hwloc_free_unlinked_object(hwloc_obj_t obj)
{
  hwloc__free_object_contents(obj);
  free(obj);
}

/* Replace old with contents of new object, and make new freeable by the caller.
 * Requires reconnect (for siblings pointers and group depth),
 * fixup of sets (only the main cpuset was likely compared before merging),
 * and update of total_memory and group depth.
 */
static void
hwloc_replace_linked_object(hwloc_obj_t old, hwloc_obj_t new)
{
  /* drop old fields */
  hwloc__free_object_contents(old);
  /* copy old tree pointers to new */
  new->parent = old->parent;
  new->next_sibling = old->next_sibling;
  new->first_child = old->first_child;
  new->memory_first_child = old->memory_first_child;
  new->io_first_child = old->io_first_child;
  new->misc_first_child = old->misc_first_child;
  /* copy new contents to old now that tree pointers are OK */
  memcpy(old, new, sizeof(*old));
  /* clear new to that we may free it */
  memset(new, 0,sizeof(*new));
}

```

这段代码定义了一个名为 `unlink_and_free_object_and_children` 的函数，它的作用是移除一个对象及其子对象，并释放内存。

该函数接受一个 `hwloc_obj_t` 类型的参数 `pobj`，代表要处理的对象。函数内部使用 `for_each_child_safe` 函数遍历对象的每个子对象，递归地调用 `unlink_and_free_object_and_children` 函数，将子对象也处理干净。最后，函数将对象的超兄弟指针 `next_sibling` 更新为对象的下一个兄弟对象，并使用 `hwloc_free_unlinked_object` 函数释放对象内存，实现从父对象中移除对象及其子对象的操作。

该函数的作用主要是在对象的生命周期结束时或者在对象被创建早期进行，例如在对象被用作数据结构时进行。对象及其子对象将被释放，但不会影响对象本身的数据或引用。


```cpp
/* Remove an object and its children from its parent and free them.
 * Only updates next_sibling/first_child pointers,
 * so may only be used during early discovery or during destroy.
 */
static void
unlink_and_free_object_and_children(hwloc_obj_t *pobj)
{
  hwloc_obj_t obj = *pobj, child, *pchild;

  for_each_child_safe(child, obj, pchild)
    unlink_and_free_object_and_children(pchild);
  for_each_memory_child_safe(child, obj, pchild)
    unlink_and_free_object_and_children(pchild);
  for_each_io_child_safe(child, obj, pchild)
    unlink_and_free_object_and_children(pchild);
  for_each_misc_child_safe(child, obj, pchild)
    unlink_and_free_object_and_children(pchild);

  *pobj = obj->next_sibling;
  hwloc_free_unlinked_object(obj);
}

```

这两段代码定义了 `hwloc_free_object_and_children()` 和 `hwloc_free_object_siblings_and_children()` 函数，它们的作用是释放由 `hwloc_obj_t` 类型的对象及其子对象和相邻对象组成的链表。

这两段代码实现了解决链表问题的一种简单方法，即通过递归遍历链表，调用 `unlink_and_free_object_and_children()` 函数来释放链表中的对象。在释放对象之后，如果链表中仍有其他对象，则递归调用 `hwloc_free_object_siblings_and_children()` 函数继续释放这些对象。这样做可以确保所有对象及其子对象和相邻对象都被正确释放，不会留下任何内存泄漏。


```cpp
/* Free an object and its children without unlinking from parent.
 */
void
hwloc_free_object_and_children(hwloc_obj_t obj)
{
  unlink_and_free_object_and_children(&obj);
}

/* Free an object, its next siblings and their children without unlinking from parent.
 */
void
hwloc_free_object_siblings_and_children(hwloc_obj_t obj)
{
  while (obj)
    unlink_and_free_object_and_children(&obj);
}

```

这段代码定义了一个名为 `insert_siblings_list` 的函数，它接受三个参数：

1. `firstp`：一个指向要插入新兄弟姐妹的指针；
2. `firstnew`：一个非空的兄弟列表，其中第一个元素是要插入的新的兄弟姐妹；
3. `newparent`：一个指向新父对象的指针。

函数的作用是，将 `firstnew` 插入到 `firstp` 所指向对象的相邻对象的位置，并将 `firstnew` 的新父指向设置为 `newparent`。

函数返回的是 `firstnew` 的下一个兄弟对象的指针。


```cpp
/* insert the (non-empty) list of sibling starting at firstnew as new children of newparent,
 * and return the address of the pointer to the next one
 */
static hwloc_obj_t *
insert_siblings_list(hwloc_obj_t *firstp, hwloc_obj_t firstnew, hwloc_obj_t newparent)
{
  hwloc_obj_t tmp;
  assert(firstnew);
  *firstp = tmp = firstnew;
  tmp->parent = newparent;
  while (tmp->next_sibling) {
    tmp = tmp->next_sibling;
    tmp->parent = newparent;
  }
  return &tmp->next_sibling;
}

```

这段代码的作用是实现将一个名为“firstnew”的新列表追加到名为“firstp”的旧列表的起始位置，并将新列表中的每个元素设置为“newparent”的子列表。同时，在更新其他相关的变量之后，还可能被用于在早期或晚期发现（更新prev_sibling和sibling_rank）。

具体来说，代码首先通过循环遍历新列表的每个元素，然后将这些元素的后继指针设置为新列表的父列表。接下来，计算新列表中的元素长度，并在旧列表中从旧列表的起始位置开始更新其索引。最后，将新列表的起始位置设置为旧列表的起始位置，使得新列表的元素在旧列表中的位置与它们的父列表元素的索引相等。


```cpp
/* Take the new list starting at firstnew and prepend it to the old list starting at *firstp,
 * and mark the new children as children of newparent.
 * May be used during early or late discovery (updates prev_sibling and sibling_rank).
 * List firstnew must be non-NULL.
 */
static void
prepend_siblings_list(hwloc_obj_t *firstp, hwloc_obj_t firstnew, hwloc_obj_t newparent)
{
  hwloc_obj_t *tmpp, tmp, last;
  unsigned length;

  /* update parent pointers and find the length and end of the new list */
  for(length = 0, tmpp = &firstnew, last = NULL ; *tmpp; length++, last = *tmpp, tmpp = &((*tmpp)->next_sibling))
    (*tmpp)->parent = newparent;

  /* update sibling_rank */
  for(tmp = *firstp; tmp; tmp = tmp->next_sibling)
    tmp->sibling_rank += length; /* if it wasn't initialized yet, it'll be overwritten later */

  /* place the existing list at the end of the new one */
  *tmpp = *firstp;
  if (*firstp)
    (*firstp)->prev_sibling = last;

  /* use the beginning of the new list now */
  *firstp = firstnew;
}

```

这段代码的作用是实现将一个新列表（firstnew）附加到另一个列表（firstp）的父节点上，并设置新列表的孩子们为新的父节点（newparent）。这有助于在发现早期或晚期时更新前兄弟节点和兄弟排名。

具体来说，代码首先通过遍历新列表的每个元素，以及新父节点的前一个兄弟节点，找到新列表的起始点和长度。然后，代码根据新列表的起始点和长度，更新新前兄弟节点和兄弟排名，将新列表的起始点设置为前一个兄弟节点的下一个兄弟节点。最后，代码将新列表的起始点设置为第一个父节点的下一个兄弟节点，如果新列表为空，则将其设置为最后一个父节点。


```cpp
/* Take the new list starting at firstnew and append it to the old list starting at *firstp,
 * and mark the new children as children of newparent.
 * May be used during early or late discovery (updates prev_sibling and sibling_rank).
 */
static void
append_siblings_list(hwloc_obj_t *firstp, hwloc_obj_t firstnew, hwloc_obj_t newparent)
{
  hwloc_obj_t *tmpp, tmp, last;
  unsigned length;

  /* find the length and end of the existing list */
  for(length = 0, tmpp = firstp, last = NULL ; *tmpp; length++, last = *tmpp, tmpp = &((*tmpp)->next_sibling));

  /* update parent pointers and sibling_rank */
  for(tmp = firstnew; tmp; tmp = tmp->next_sibling) {
    tmp->parent = newparent;
    tmp->sibling_rank += length; /* if it wasn't set yet, it'll be overwritten later */
  }

  /* place new list at the end of the old one */
  *tmpp = firstnew;
  if (firstnew)
    firstnew->prev_sibling = last;
}

```

This function appears to be a part of the `hwloc` library, which seems to be a library for managing hardware resources on Linux systems.

The function takes an object of the specified `hwloc__obj_type_is_memory` type, which is either `hwloc_obj_type_is_memory` or `hwloc_obj_type_is_discrete_device`, and returns the object's parent object, if it is a `hwloc_obj_type_is_memory` object, or the entire object if it is not a `hwloc_obj_type_is_memory` object.

The function first checks if the object is a `hwloc_obj_type_is_memory` object and, if it is, it inserts the object's children, if any, as new siblings below the parent object. If it is not a `hwloc_obj_type_is_memory` object, the function simply inserts the object into the parent object's list of children.

The function then checks if the object has any children of its own and, if it does, it appends them to the parent object's list of children. If it does not have any children, the function sets the parent object as the object's parent object, and appends the object to the list of children of the parent object.

Finally, the function frees the object's memory and other resources, if it has any.


```cpp
/* Remove an object from its parent and free it.
 * Only updates next_sibling/first_child pointers,
 * so may only be used during early discovery.
 *
 * Children are inserted in the parent.
 * If children should be inserted somewhere else (e.g. when merging with a child),
 * the caller should move them before calling this function.
 */
static void
unlink_and_free_single_object(hwloc_obj_t *pparent)
{
  hwloc_obj_t old = *pparent;
  hwloc_obj_t *lastp;

  if (old->type == HWLOC_OBJ_MISC) {
    /* Misc object */

    /* no normal children */
    assert(!old->first_child);
    /* no memory children */
    assert(!old->memory_first_child);
    /* no I/O children */
    assert(!old->io_first_child);

    if (old->misc_first_child)
      /* insert old misc object children as new siblings below parent instead of old */
      lastp = insert_siblings_list(pparent, old->misc_first_child, old->parent);
    else
      lastp = pparent;
    /* append old siblings back */
    *lastp = old->next_sibling;

  } else if (hwloc__obj_type_is_io(old->type)) {
    /* I/O object */

    /* no normal children */
    assert(!old->first_child);
    /* no memory children */
    assert(!old->memory_first_child);

    if (old->io_first_child)
      /* insert old I/O object children as new siblings below parent instead of old */
      lastp = insert_siblings_list(pparent, old->io_first_child, old->parent);
    else
      lastp = pparent;
    /* append old siblings back */
    *lastp = old->next_sibling;

    /* append old Misc children to parent */
    if (old->misc_first_child)
      append_siblings_list(&old->parent->misc_first_child, old->misc_first_child, old->parent);

  } else if (hwloc__obj_type_is_memory(old->type)) {
    /* memory object */

    /* no normal children */
    assert(!old->first_child);
    /* no I/O children */
    assert(!old->io_first_child);

    if (old->memory_first_child)
      /* insert old memory object children as new siblings below parent instead of old */
      lastp = insert_siblings_list(pparent, old->memory_first_child, old->parent);
    else
      lastp = pparent;
    /* append old siblings back */
    *lastp = old->next_sibling;

    /* append old Misc children to parent */
    if (old->misc_first_child)
      append_siblings_list(&old->parent->misc_first_child, old->misc_first_child, old->parent);

  } else {
    /* Normal object */

    if (old->first_child)
      /* insert old object children as new siblings below parent instead of old */
      lastp = insert_siblings_list(pparent, old->first_child, old->parent);
    else
      lastp = pparent;
    /* append old siblings back */
    *lastp = old->next_sibling;

    /* append old memory, I/O and Misc children to parent
     * old->parent cannot be NULL (removing root), misc children should have been moved by the caller earlier.
     */
    if (old->memory_first_child)
      append_siblings_list(&old->parent->memory_first_child, old->memory_first_child, old->parent);
    if (old->io_first_child)
      append_siblings_list(&old->parent->io_first_child, old->io_first_child, old->parent);
    if (old->misc_first_child)
      append_siblings_list(&old->parent->misc_first_child, old->misc_first_child, old->parent);
  }

  hwloc_free_unlinked_object(old);
}

```

This function appears to be a part of a larger object-oriented programming language. It is used to insert a child object into its parent object's children array. The function takes various arguments, such as the object the child object is being inserted into, the parent object, and various indices related to the parent object's children array. It checks whether the child object is a leaf object, a memory object, a io object, or a miscellaneous object, and sets the parent object's children array accordingly. If the child object is a leaf object, it is inserted as the first child. If the child object is a memory object, it is inserted as the first child, and the parent object's memory array is modified to point to the first child. If the child object is an io object, it is inserted as the first child, and the parent object's io array is modified to point to the first child. If the child object is a miscellaneous object, it is inserted as the first child. If the child object cannot be inserted, it returns an error.


```cpp
/* This function may use a tma, it cannot free() or realloc() */
static int
hwloc__duplicate_object(struct hwloc_topology *newtopology,
			struct hwloc_obj *newparent,
			struct hwloc_obj *newobj,
			struct hwloc_obj *src)
{
  struct hwloc_tma *tma = newtopology->tma;
  hwloc_obj_t *level;
  unsigned level_width;
  size_t len;
  unsigned i;
  hwloc_obj_t child, prev;
  int err = 0;

  /* either we're duplicating to an already allocated new root, which has no newparent,
   * or we're duplicating to a non-yet allocated new non-root, which will have a newparent.
   */
  assert(!newparent == !!newobj);

  if (!newobj) {
    newobj = hwloc_alloc_setup_object(newtopology, src->type, src->os_index);
    if (!newobj)
      return -1;
  }

  /* duplicate all non-object-pointer fields */
  newobj->logical_index = src->logical_index;
  newobj->depth = src->depth;
  newobj->sibling_rank = src->sibling_rank;

  newobj->type = src->type;
  newobj->os_index = src->os_index;
  newobj->gp_index = src->gp_index;
  newobj->symmetric_subtree = src->symmetric_subtree;

  if (src->name)
    newobj->name = hwloc_tma_strdup(tma, src->name);
  if (src->subtype)
    newobj->subtype = hwloc_tma_strdup(tma, src->subtype);
  newobj->userdata = src->userdata;

  newobj->total_memory = src->total_memory;

  memcpy(newobj->attr, src->attr, sizeof(*newobj->attr));

  if (src->type == HWLOC_OBJ_NUMANODE && src->attr->numanode.page_types_len) {
    len = src->attr->numanode.page_types_len * sizeof(struct hwloc_memory_page_type_s);
    newobj->attr->numanode.page_types = hwloc_tma_malloc(tma, len);
    memcpy(newobj->attr->numanode.page_types, src->attr->numanode.page_types, len);
  }

  newobj->cpuset = hwloc_bitmap_tma_dup(tma, src->cpuset);
  newobj->complete_cpuset = hwloc_bitmap_tma_dup(tma, src->complete_cpuset);
  newobj->nodeset = hwloc_bitmap_tma_dup(tma, src->nodeset);
  newobj->complete_nodeset = hwloc_bitmap_tma_dup(tma, src->complete_nodeset);

  hwloc__tma_dup_infos(tma, &newobj->infos, &newobj->infos_count, src->infos, src->infos_count);

  /* find our level */
  if (src->depth < 0) {
    i = HWLOC_SLEVEL_FROM_DEPTH(src->depth);
    level = newtopology->slevels[i].objs;
    level_width = newtopology->slevels[i].nbobjs;
    /* deal with first/last pointers of special levels, even if not really needed */
    if (!newobj->logical_index)
      newtopology->slevels[i].first = newobj;
    if (newobj->logical_index == newtopology->slevels[i].nbobjs - 1)
      newtopology->slevels[i].last = newobj;
  } else {
    level = newtopology->levels[src->depth];
    level_width = newtopology->level_nbobjects[src->depth];
  }
  /* place us for real */
  assert(newobj->logical_index < level_width);
  level[newobj->logical_index] = newobj;
  /* link to already-inserted cousins */
  if (newobj->logical_index > 0 && level[newobj->logical_index-1]) {
    newobj->prev_cousin = level[newobj->logical_index-1];
    level[newobj->logical_index-1]->next_cousin = newobj;
  }
  if (newobj->logical_index < level_width-1 && level[newobj->logical_index+1]) {
    newobj->next_cousin = level[newobj->logical_index+1];
    level[newobj->logical_index+1]->prev_cousin = newobj;
  }

  /* prepare for children */
  if (src->arity) {
    newobj->children = hwloc_tma_malloc(tma, src->arity * sizeof(*newobj->children));
    if (!newobj->children)
      return -1;
  }
  newobj->arity = src->arity;
  newobj->memory_arity = src->memory_arity;
  newobj->io_arity = src->io_arity;
  newobj->misc_arity = src->misc_arity;

  /* actually insert children now */
  for_each_child(child, src) {
    err = hwloc__duplicate_object(newtopology, newobj, NULL, child);
    if (err < 0)
      goto out_with_children;
  }
  for_each_memory_child(child, src) {
    err = hwloc__duplicate_object(newtopology, newobj, NULL, child);
    if (err < 0)
      return err;
  }
  for_each_io_child(child, src) {
    err = hwloc__duplicate_object(newtopology, newobj, NULL, child);
    if (err < 0)
      goto out_with_children;
  }
  for_each_misc_child(child, src) {
    err = hwloc__duplicate_object(newtopology, newobj, NULL, child);
    if (err < 0)
      goto out_with_children;
  }

 out_with_children:

  /* link children if all of them where inserted */
  if (!err) {
    /* only next_sibling is set by insert_by_parent().
     * sibling_rank was set above.
     */
    if (newobj->arity) {
      newobj->children[0]->prev_sibling = NULL;
      for(i=1; i<newobj->arity; i++)
	newobj->children[i]->prev_sibling = newobj->children[i-1];
      newobj->last_child = newobj->children[newobj->arity-1];
    }
    if (newobj->memory_arity) {
      child = newobj->memory_first_child;
      prev = NULL;
      while (child) {
	child->prev_sibling = prev;
	prev = child;
	child = child->next_sibling;
      }
    }
    if (newobj->io_arity) {
      child = newobj->io_first_child;
      prev = NULL;
      while (child) {
	child->prev_sibling = prev;
	prev = child;
	child = child->next_sibling;
      }
    }
    if (newobj->misc_arity) {
      child = newobj->misc_first_child;
      prev = NULL;
      while (child) {
	child->prev_sibling = prev;
	prev = child;
	child = child->next_sibling;
      }
    }
  }

  /* some children insertion may have failed, but some children may have been inserted below us already.
   * keep inserting ourself and let the caller clean the entire tree if we return an error.
   */

  if (newparent) {
    /* no need to check the children insert order here, the source topology
     * is supposed to be OK already, and we have debug asserts.
     */
    hwloc_insert_object_by_parent(newtopology, newparent, newobj);

    /* place us inside our parent children array */
    if (hwloc__obj_type_is_normal(newobj->type))
      newparent->children[newobj->sibling_rank] = newobj;
  }

  return err;
}

```

This code appears to be processing object hierarchies and allocating memory for objects at different levels of the hierarchy. The code is using a recursive approach to handle object hierarchies with multiple levels of indirection.

The code is first defining a set of constants called `new_levels` and a variable called `new_nbobjects` to keep track of the number of objects at each level of the hierarchy. It is then creating a recursive relationship between the objects at each level, with the root object having a NULL value for its `level_nbobjects` array.

The code then creates a loop that goes through each object at each level and creates a corresponding object in the hierarchy. It does this by first allocating space for the object on the local machine using the `hwloc_tma_calloc` function, and then creates the object using the `hwloc_tma_calloc` function.

The code then recursively creates any missing children of the current object and initializes their `nbobjs` field with the `nb_levels_allocated` field of the current object.

Finally, the code sets the modified flag on the object to `true` to indicate that it has been modified since it was last created, and sets the `backends` field to `NULL` to indicate that the object does not have a backend.


```cpp
static int
hwloc__topology_init (struct hwloc_topology **topologyp, unsigned nblevels, struct hwloc_tma *tma);

/* This function may use a tma, it cannot free() or realloc() */
int
hwloc__topology_dup(hwloc_topology_t *newp,
		    hwloc_topology_t old,
		    struct hwloc_tma *tma)
{
  hwloc_topology_t new;
  hwloc_obj_t newroot;
  hwloc_obj_t oldroot = hwloc_get_root_obj(old);
  unsigned i;
  int err;

  if (!old->is_loaded) {
    errno = EINVAL;
    return -1;
  }

  err = hwloc__topology_init(&new, old->nb_levels_allocated, tma);
  if (err < 0)
    goto out;

  new->flags = old->flags;
  memcpy(new->type_filter, old->type_filter, sizeof(old->type_filter));
  new->is_thissystem = old->is_thissystem;
  new->is_loaded = 1;
  new->pid = old->pid;
  new->next_gp_index = old->next_gp_index;

  memcpy(&new->binding_hooks, &old->binding_hooks, sizeof(old->binding_hooks));

  memcpy(new->support.discovery, old->support.discovery, sizeof(*old->support.discovery));
  memcpy(new->support.cpubind, old->support.cpubind, sizeof(*old->support.cpubind));
  memcpy(new->support.membind, old->support.membind, sizeof(*old->support.membind));
  memcpy(new->support.misc, old->support.misc, sizeof(*old->support.misc));

  new->allowed_cpuset = hwloc_bitmap_tma_dup(tma, old->allowed_cpuset);
  new->allowed_nodeset = hwloc_bitmap_tma_dup(tma, old->allowed_nodeset);

  new->userdata_export_cb = old->userdata_export_cb;
  new->userdata_import_cb = old->userdata_import_cb;
  new->userdata_not_decoded = old->userdata_not_decoded;

  assert(!old->machine_memory.local_memory);
  assert(!old->machine_memory.page_types_len);
  assert(!old->machine_memory.page_types);

  for(i = HWLOC_OBJ_TYPE_MIN; i < HWLOC_OBJ_TYPE_MAX; i++)
    new->type_depth[i] = old->type_depth[i];

  /* duplicate levels and we'll place objects there when duplicating objects */
  new->nb_levels = old->nb_levels;
  assert(new->nb_levels_allocated >= new->nb_levels);
  for(i=1 /* root level already allocated */ ; i<new->nb_levels; i++) {
    new->level_nbobjects[i] = old->level_nbobjects[i];
    new->levels[i] = hwloc_tma_calloc(tma, new->level_nbobjects[i] * sizeof(*new->levels[i]));
  }
  for(i=0; i<HWLOC_NR_SLEVELS; i++) {
    new->slevels[i].nbobjs = old->slevels[i].nbobjs;
    if (new->slevels[i].nbobjs)
      new->slevels[i].objs = hwloc_tma_calloc(tma, new->slevels[i].nbobjs * sizeof(*new->slevels[i].objs));
  }

  /* recursively duplicate object children */
  newroot = hwloc_get_root_obj(new);
  err = hwloc__duplicate_object(new, NULL, newroot, oldroot);
  if (err < 0)
    goto out_with_topology;

  err = hwloc_internal_distances_dup(new, old);
  if (err < 0)
    goto out_with_topology;

  err = hwloc_internal_memattrs_dup(new, old);
  if (err < 0)
    goto out_with_topology;

  err = hwloc_internal_cpukinds_dup(new, old);
  if (err < 0)
    goto out_with_topology;

  /* we connected everything during duplication */
  new->modified = 0;

  /* no need to duplicate backends, topology is already loaded */
  new->backends = NULL;
  new->get_pci_busid_cpuset_backend = NULL;

```

这段代码是一个C函数，定义了一个名为`hwloc_debug`的函数。该函数的作用是：

1. 如果当前目录下的`HWLOC_DEBUG_CHECK`环境变量存在，则执行`hwloc_topology_check`函数；
2. 如果`HWLOC_DEBUG_CHECK`环境变量不存在，则执行`hwloc_topology_destroy`函数并返回-1；
3. 在函数内部，创建一个新的`hwloc_topology_t`结构体变量`new`，将其赋值为`new`；
4. 返回0。

函数`hwloc_debug`的作用是检查当前目录下的`HWLOC_DEBUG_CHECK`环境变量是否存在，如果不存在，则执行`hwloc_topology_destroy`函数并返回-1，以使函数能够正确地检测到`hwloc_topology`的使用情况。


```cpp
#ifndef HWLOC_DEBUG
  if (getenv("HWLOC_DEBUG_CHECK"))
#endif
    hwloc_topology_check(new);

  *newp = new;
  return 0;

 out_with_topology:
  assert(!tma || !tma->dontfree); /* this tma cannot fail to allocate */
  hwloc_topology_destroy(new);
 out:
  return -1;
}

```

这段代码是一个名为`hwloc_topology_dup`的函数，属于`hwloc_topology_t`类型的别处。它的作用是复制`hwloc_topology_t`类型的数据，将其复制到名为`newp`的变量中，同时将原始参数`old`作为参数传递给函数。

具体来说，这段代码的主要作用是确保`hwloc_topology_t`类型的数据在复制过程中不会出现错误，并为`hwloc_topology_init()`函数提供正确的复制品。通过这种途径，可以确保在将数据复制到别处时，可以创建一个与原始数据具有相同结构的整体。


```cpp
int
hwloc_topology_dup(hwloc_topology_t *newp,
		   hwloc_topology_t old)
{
  return hwloc__topology_dup(newp, old, NULL);
}

/* WARNING: The indexes of this array MUST match the ordering that of
   the obj_order_type[] array, below.  Specifically, the values must
   be laid out such that:

       obj_order_type[obj_type_order[N]] = N

   for all HWLOC_OBJ_* values of N.  Put differently:

       obj_type_order[A] = B

   where the A values are in order of the hwloc_obj_type_t enum, and
   the B values are the corresponding indexes of obj_order_type.

   We can't use C99 syntax to initialize this in a little safer manner
   -- bummer.  :-(

   Correctness is asserted in hwloc_topology_init() when debug is enabled.
   */
```

这段代码定义了一个名为 `obj_type_order` 的静态常量数组，用于指定不同硬件设备的 obj 类型优先级。这个数组包含了各种硬件设备的 obj 类型，如处理器、芯片组、内存、硬盘驱动器等等。这些 obj 类型的优先级是按照从高到低的顺序排列的，因此， `obj_type_order[0]` 对应的是 `HWLOC_OBJ_MACHINE`, `obj_type_order[4]` 对应的是 `HWLOC_OBJ_PACKAGE`，以此类推。

这个静态常量数组可以被用来维护程序中各种硬件设备 obj 类型的优先级信息，帮助程序正确地操作系统中的硬件资源，并确保在程序更新时不会降低硬件设备的优先级。


```cpp
/***** Make sure you update obj_type_priority[] below as well. *****/
static const unsigned obj_type_order[] = {
    /* first entry is HWLOC_OBJ_MACHINE */  0,
    /* next entry is HWLOC_OBJ_PACKAGE */  4,
    /* next entry is HWLOC_OBJ_CORE */     14,
    /* next entry is HWLOC_OBJ_PU */       18,
    /* next entry is HWLOC_OBJ_L1CACHE */  12,
    /* next entry is HWLOC_OBJ_L2CACHE */  10,
    /* next entry is HWLOC_OBJ_L3CACHE */  8,
    /* next entry is HWLOC_OBJ_L4CACHE */  7,
    /* next entry is HWLOC_OBJ_L5CACHE */  6,
    /* next entry is HWLOC_OBJ_L1ICACHE */ 13,
    /* next entry is HWLOC_OBJ_L2ICACHE */ 11,
    /* next entry is HWLOC_OBJ_L3ICACHE */ 9,
    /* next entry is HWLOC_OBJ_GROUP */    1,
    /* next entry is HWLOC_OBJ_NUMANODE */ 3,
    /* next entry is HWLOC_OBJ_BRIDGE */   15,
    /* next entry is HWLOC_OBJ_PCI_DEVICE */  16,
    /* next entry is HWLOC_OBJ_OS_DEVICE */   17,
    /* next entry is HWLOC_OBJ_MISC */     19,
    /* next entry is HWLOC_OBJ_MEMCACHE */ 2,
    /* next entry is HWLOC_OBJ_DIE */      5
};

```

这段代码定义了一个名为`obj_order_type`的静态常量数组，用于表示hwloc_obj_type_t类型的不同组件。这个数组包含了13个不同的`HWLOC_OBJ_`类型，每个`HWLOC_OBJ_`类型都有一个对应的`obj_order_type`值。

这个数组的目的是为了在程序开发过程中，对hwloc_obj_type_t类型的组件进行统一的命名和说明。通过这个数，开发人员可以使用预定义的名称来理解组件的行为和用途，而不需要记住它们的实际类型。例如，如果组件的类型实现了某个特定的函数或方法，开发人员可以通过这个数来了解它的作用，而不是需要了解具体的实现细节。


```cpp
#ifndef NDEBUG /* only used in debug check assert if !NDEBUG */
static const hwloc_obj_type_t obj_order_type[] = {
  HWLOC_OBJ_MACHINE,
  HWLOC_OBJ_GROUP,
  HWLOC_OBJ_MEMCACHE,
  HWLOC_OBJ_NUMANODE,
  HWLOC_OBJ_PACKAGE,
  HWLOC_OBJ_DIE,
  HWLOC_OBJ_L5CACHE,
  HWLOC_OBJ_L4CACHE,
  HWLOC_OBJ_L3CACHE,
  HWLOC_OBJ_L3ICACHE,
  HWLOC_OBJ_L2CACHE,
  HWLOC_OBJ_L2ICACHE,
  HWLOC_OBJ_L1CACHE,
  HWLOC_OBJ_L1ICACHE,
  HWLOC_OBJ_CORE,
  HWLOC_OBJ_BRIDGE,
  HWLOC_OBJ_PCI_DEVICE,
  HWLOC_OBJ_OS_DEVICE,
  HWLOC_OBJ_PU,
  HWLOC_OBJ_MISC /* Misc is always a leaf */
};
```

这段代码定义了一个名为 `obj_type_priority` 的数组，用于在对象合并时确定优先级。这个数组的默认值为 `Machine/NUMANode/PU/PCIDev/OSDev`、`Core`、`Package` 和 `Die`。数组中的元素按照从高到低的顺序排列，以最小化合并无用子对象时可能出现的优先级冲突。

在 `obj_type_priority` 数组被使用之前，它已经在 `merge_useless_child` 函数中进行了更新。如果两个对象具有相同的父对象，则这个函数会尝试合并对象，并将优先级设置为最高。这样可以确保在合并对象时，即使两个对象具有相同的父对象，也会使用最高的优先级来决定哪个对象被保留下来。

数组中的具体元素如下：

- `Machine/NUMANode/PU/PCIDev/OSDev` 排在了最前面，因为这些对象通常不会被用于对象的合并，所以不会出现优先级冲突。
- `Core` 排在了第二个，因为它是所有选项卡中最重要的一个，所以一定会被用于合并。
- `Package` 排在了第三个，因为它是所有支持选项卡中最重要的一个，所以一定会被用于合并。
- `Die` 排在了第四个，因为它是所有 `File` 类型对象的父对象，所以一定会被用于合并。
- `Cage` 排在了第五个，`Instruction Cache` 排在了第六个，和 `Always` 排在了第七个。这三者都是 `Instruction` 类型的对象，它们的优先级相同。所以它们都会被用于合并，但是如果两个或更多个 `Instruction` 类型的对象被合并，则它们可能会出现优先级冲突。这个数组确保了在合并对象时，最高的优先级总是被保留下来。
- `Group/Misc/Bridge` 排在了第八个。

最后，还有一些注释，指出哪些类型的对象可能不会使用这个数组。


```cpp
#endif
/***** Make sure you update obj_type_priority[] below as well. *****/

/* priority to be used when merging identical parent/children object
 * (in merge_useless_child), keep the highest priority one.
 *
 * Always keep Machine/NUMANode/PU/PCIDev/OSDev
 * then Core
 * then Package
 * then Die
 * then Cache,
 * then Instruction Caches
 * then always drop Group/Misc/Bridge.
 *
 * Some type won't actually ever be involved in such merging.
 */
```

这段代码定义了一个名为 obj_type_priority 的静态变量，它是一个整型数组，包含了不同硬件组件的优先级。这个数组是为了在程序中选择正确的硬件组件而设计的，程序会根据不同的参数来选择最优的硬件组件。

这个数组的元素个数为 13，每个元素代表了一个不同的硬件组件。其中，第一行的元素 HWLOC_OBJ_MACHINE 的优先级最高，为 90，表示这是最重要的硬件组件。其余的元素的优先级根据它们的硬件组件类型而有所不同，如 HWLOC_OBJ_PACKAGE 的优先级为 40,HWLOC_OBJ_CORE 的优先级为 60,HWLOC_OBJ_L1CACHE 和 HWLOC_OBJ_L2CACHE 的优先级都为 20,HWLOC_OBJ_L3CACHE 和 HWLOC_OBJ_L1ICACHE 的优先级也都为 20。

这个静态变量的作用是告诉程序不同硬件组件的优先级，以便在程序设计时能够根据需要来选择正确的硬件组件。


```cpp
/***** Make sure you update this array when changing the list of types. *****/
static const int obj_type_priority[] = {
  /* first entry is HWLOC_OBJ_MACHINE */     90,
  /* next entry is HWLOC_OBJ_PACKAGE */     40,
  /* next entry is HWLOC_OBJ_CORE */        60,
  /* next entry is HWLOC_OBJ_PU */          100,
  /* next entry is HWLOC_OBJ_L1CACHE */     20,
  /* next entry is HWLOC_OBJ_L2CACHE */     20,
  /* next entry is HWLOC_OBJ_L3CACHE */     20,
  /* next entry is HWLOC_OBJ_L4CACHE */     20,
  /* next entry is HWLOC_OBJ_L5CACHE */     20,
  /* next entry is HWLOC_OBJ_L1ICACHE */    19,
  /* next entry is HWLOC_OBJ_L2ICACHE */    19,
  /* next entry is HWLOC_OBJ_L3ICACHE */    19,
  /* next entry is HWLOC_OBJ_GROUP */       0,
  /* next entry is HWLOC_OBJ_NUMANODE */    100,
  /* next entry is HWLOC_OBJ_BRIDGE */      0,
  /* next entry is HWLOC_OBJ_PCI_DEVICE */  100,
  /* next entry is HWLOC_OBJ_OS_DEVICE */   100,
  /* next entry is HWLOC_OBJ_MISC */        0,
  /* next entry is HWLOC_OBJ_MEMCACHE */    19,
  /* next entry is HWLOC_OBJ_DIE */         30
};

```

这段代码定义了一个名为 `hwloc_compare_types` 的函数，其作用是比較兩個 `hwloc_obj_type_t` 类型的object，並返回一個整數。

函数有两个參數，一個是 `type1`，另一個是 `type2`，它們分別代表要比較的兩個object的類型。

函数首先定義了一個名為 `obj_type_order` 的變數，其作用是存儲 `hwloc_obj_type_t` 的型別對應的序號。然後，函数判斷 `type1` 和 `type2` 是否為正常的 `hwloc_obj_type_t`，如果不是，就返回 `HWLOC_TYPE_UNORDERED`。如果 `type1` 和 `type2` 都為正常的 `hwloc_obj_type_t`，並且 `type2` 不是 `HWLOC_OBJ_MACHINE`，就返回 `HWLOC_TYPE_ORDERED`。

最後，函数返回 `obj_type_order[type1] - obj_type_order[type2]`，即 `hwloc_compare_types` 的返回值。


```cpp
int hwloc_compare_types (hwloc_obj_type_t type1, hwloc_obj_type_t type2)
{
  unsigned order1 = obj_type_order[type1];
  unsigned order2 = obj_type_order[type2];

  /* only normal objects are comparable. others are only comparable with machine */
  if (!hwloc__obj_type_is_normal(type1)
      && hwloc__obj_type_is_normal(type2) && type2 != HWLOC_OBJ_MACHINE)
    return HWLOC_TYPE_UNORDERED;
  if (!hwloc__obj_type_is_normal(type2)
      && hwloc__obj_type_is_normal(type1) && type1 != HWLOC_OBJ_MACHINE)
    return HWLOC_TYPE_UNORDERED;

  return order1 - order2;
}

```

以下是关于 `hwloc_type_cmp` 的函数原型，实现了 `hwloc_type_cmp` 的函数实现。
```cppc
static enum hwloc_obj_cmp_e
hwloc_type_cmp(hwloc_obj_t obj1, hwloc_obj_t obj2)
{
 hwloc_obj_type_t type1 = obj1->type;
 hwloc_obj_type_t type2 = obj2->type;
 int compare;

 compare = hwloc_compare_types(type1, type2);
 if (compare == HWLOC_TYPE_UNORDERED)
   return HWLOC_OBJ_DIFFERENT; /* we cannot do better */
 if (compare > 0)
   return HWLOC_OBJ_INCLUDED;
 if (compare < 0)
   return HWLOC_OBJ_CONTAINS;

 if (obj1->type == HWLOC_OBJ_GROUP
     && (obj1->attr->group.kind != obj2->attr->group.kind
	  || obj1->attr->group.subkind != obj2->attr->group.subkind))
   return HWLOC_OBJ_DIFFERENT; /* we cannot do better */

 return HWLOC_OBJ_EQUAL;
}
```
这个函数接受两个 `hwloc_obj_t` 类型的参数，比较两个参数的类型是否相同，如果不同，返回 `HWLOC_OBJ_DIFFERENT`。如果两个类型的比较结果为正数，那么这两个参数对应的 `hwloc_obj_t` 类型的属性中，是否包含的子属性值的范围是否相同。如果两个类型的比较结果为负数，那么这两个参数对应的 `hwloc_obj_t` 类型的属性中，是否包含的子属性值的范围是否包含。如果两个类型的比较结果为0，那么这两个参数对应的 `hwloc_obj_t` 类型的属性中，包含的子属性值的范围是否相等。


```cpp
enum hwloc_obj_cmp_e {
  HWLOC_OBJ_EQUAL = HWLOC_BITMAP_EQUAL,			/**< \brief Equal */
  HWLOC_OBJ_INCLUDED = HWLOC_BITMAP_INCLUDED,		/**< \brief Strictly included into */
  HWLOC_OBJ_CONTAINS = HWLOC_BITMAP_CONTAINS,		/**< \brief Strictly contains */
  HWLOC_OBJ_INTERSECTS = HWLOC_BITMAP_INTERSECTS,	/**< \brief Intersects, but no inclusion! */
  HWLOC_OBJ_DIFFERENT = HWLOC_BITMAP_DIFFERENT		/**< \brief No intersection */
};

static enum hwloc_obj_cmp_e
hwloc_type_cmp(hwloc_obj_t obj1, hwloc_obj_t obj2)
{
  hwloc_obj_type_t type1 = obj1->type;
  hwloc_obj_type_t type2 = obj2->type;
  int compare;

  compare = hwloc_compare_types(type1, type2);
  if (compare == HWLOC_TYPE_UNORDERED)
    return HWLOC_OBJ_DIFFERENT; /* we cannot do better */
  if (compare > 0)
    return HWLOC_OBJ_INCLUDED;
  if (compare < 0)
    return HWLOC_OBJ_CONTAINS;

  if (obj1->type == HWLOC_OBJ_GROUP
      && (obj1->attr->group.kind != obj2->attr->group.kind
	  || obj1->attr->group.subkind != obj2->attr->group.subkind))
    return HWLOC_OBJ_DIFFERENT; /* we cannot do better */

  return HWLOC_OBJ_EQUAL;
}

```

这段代码定义了一个名为 `hwloc_obj_cmp_sets` 的函数，用于比较两个 `hwloc_obj_t` 类型的对象，根据 `cpu_set` 是否完成来比较它们。函数的参数有两个，分别代表要比较的两个 `hwloc_obj_t` 对象。

函数首先检查要比较的 `cpu_set` 是否完成，如果是，则将两个 `set` 都初始化为相同的值，否则将两个 `cpu_set` 分别初始化为相同的对象。

接下来，函数检查两个 `set` 是否都为空集（即 `hwloc_bitmap_iszero` 返回 `true`）。如果是，函数将返回 `hwloc_bitmap_compare_inclusion`，否则返回 `HWLOC_OBJ_DIFFERENT`。

函数的作用是用于比较两个 `hwloc_obj_t` 类型的对象，根据 `cpu_set` 是否完成来比较它们。如果两个对象在 `cpu_set` 上有交集，并且两个 `set` 都不是空集，则返回最小的 `set`。否则，返回 `HWLOC_OBJ_DIFFERENT`。


```cpp
/*
 * How to compare objects based on cpusets.
 */
static int
hwloc_obj_cmp_sets(hwloc_obj_t obj1, hwloc_obj_t obj2)
{
  hwloc_bitmap_t set1, set2;

  assert(!hwloc__obj_type_is_special(obj1->type));
  assert(!hwloc__obj_type_is_special(obj2->type));

  /* compare cpusets first */
  if (obj1->complete_cpuset && obj2->complete_cpuset) {
    set1 = obj1->complete_cpuset;
    set2 = obj2->complete_cpuset;
  } else {
    set1 = obj1->cpuset;
    set2 = obj2->cpuset;
  }
  if (set1 && set2 && !hwloc_bitmap_iszero(set1) && !hwloc_bitmap_iszero(set2))
    return hwloc_bitmap_compare_inclusion(set1, set2);

  return HWLOC_OBJ_DIFFERENT;
}

```

这段代码定义了一个名为 `hwloc__object_cpusets_compare_first` 的函数，用于比较两个二叉树中的 CPU 集合。

函数有两个参数 `obj1` 和 `obj2`，它们都是 `hwloc_obj_t` 类型的对象，代表了一个二叉树中的一个节点。函数的作用是在两个节点之间比较 CPU 集合，以确定哪个节点对应的二叉树优先级更高。

函数首先检查两个节点是否都定义了 `complete_cpuset` 成员。如果是，函数将直接比较这两个成员。如果不是，函数将检查每个节点是否都定义了 `cpuset` 成员。如果是，函数将比较这两个成员，以确定哪个节点对应的二叉树优先级更高。

如果两个节点都没有定义 `complete_cpuset` 或 `cpuset` 成员，函数将直接返回 0，表示两个节点对应的二叉树优先级相同。

在比较完 `complete_cpuset` 和 `cpuset` 成员之后，如果两个节点都定义了 `complete_cpuset` 成员，函数将返回 `hwloc_bitmap_compare_first` 函数的返回值，这个函数将两个二叉树对应的 CPU 集合进行比较。如果两个节点都定义了 `cpuset` 成员，函数将同样返回 `hwloc_bitmap_compare_first` 函数的返回值，这个函数将两个二叉树对应的 CPU 集合进行比较。


```cpp
/* Compare object cpusets based on complete_cpuset if defined (always correctly ordered),
 * or fallback to the main cpusets (only correctly ordered during early insert before disallowed bits are cleared).
 *
 * This is the sane way to compare object among a horizontal level.
 */
int
hwloc__object_cpusets_compare_first(hwloc_obj_t obj1, hwloc_obj_t obj2)
{
  if (obj1->complete_cpuset && obj2->complete_cpuset)
    return hwloc_bitmap_compare_first(obj1->complete_cpuset, obj2->complete_cpuset);
  else if (obj1->cpuset && obj2->cpuset)
    return hwloc_bitmap_compare_first(obj1->cpuset, obj2->cpuset);
  return 0;
}

```

This code appears to be a part of an embedded system that manages the memory usage of different hardware locations. It is used to update the attributes of an object when it is loaded or updated.

The code defines several case statements that check the type of the new object and handle it accordingly. If the object is a L1 cache, L2 cache, or L3 cache, the code checks if the corresponding attribute is present in the old object. If not, it sets the attribute to be equal to the new one.

If the object is a hardware location with a specific cache type, the code sets the attribute to be equal to the new one. It also sets the size, linesize, and associativity of the cache, if they are not present in the old object.

If the object is not a hardware location or the type is not recognized, the code falls back to the default behavior.

It is important to note that the code is assumes to be分期 load, the given code is using the information that it is loaded later than the other object, so it is loading the attributes of the object that will be used later.


```cpp
/*
 * How to insert objects into the topology.
 *
 * Note: during detection, only the first_child and next_sibling pointers are
 * kept up to date.  Others are computed only once topology detection is
 * complete.
 */

/* merge new object attributes in old.
 * use old if defined, otherwise use new.
 */
static void
merge_insert_equal(hwloc_obj_t new, hwloc_obj_t old)
{
  if (old->os_index == HWLOC_UNKNOWN_INDEX)
    old->os_index = new->os_index;

  if (new->infos_count) {
    /* FIXME: dedup */
    hwloc__move_infos(&old->infos, &old->infos_count,
		      &new->infos, &new->infos_count);
  }

  if (new->name && !old->name) {
    old->name = new->name;
    new->name = NULL;
  }
  if (new->subtype && !old->subtype) {
    old->subtype = new->subtype;
    new->subtype = NULL;
  }

  /* Ignore userdata. It will be NULL before load().
   * It may be non-NULL if alloc+insert_group() after load().
   */

  switch(new->type) {
  case HWLOC_OBJ_NUMANODE:
    if (new->attr->numanode.local_memory && !old->attr->numanode.local_memory) {
      /* no memory in old, use new memory */
      old->attr->numanode.local_memory = new->attr->numanode.local_memory;
      free(old->attr->numanode.page_types);
      old->attr->numanode.page_types_len = new->attr->numanode.page_types_len;
      old->attr->numanode.page_types = new->attr->numanode.page_types;
      new->attr->numanode.page_types = NULL;
      new->attr->numanode.page_types_len = 0;
    }
    /* old->attr->numanode.total_memory will be updated by propagate_total_memory() */
    break;
  case HWLOC_OBJ_L1CACHE:
  case HWLOC_OBJ_L2CACHE:
  case HWLOC_OBJ_L3CACHE:
  case HWLOC_OBJ_L4CACHE:
  case HWLOC_OBJ_L5CACHE:
  case HWLOC_OBJ_L1ICACHE:
  case HWLOC_OBJ_L2ICACHE:
  case HWLOC_OBJ_L3ICACHE:
    if (!old->attr->cache.size)
      old->attr->cache.size = new->attr->cache.size;
    if (!old->attr->cache.linesize)
      old->attr->cache.size = new->attr->cache.linesize;
    if (!old->attr->cache.associativity)
      old->attr->cache.size = new->attr->cache.linesize;
    break;
  default:
    break;
  }
}

```

This function appears to merge memory groups. It does so by first checking whether the new object is a memory group, and if it is, it checks whether the group kind is the same as the old group's kind. If the kind are different, it keeps the memory group as is. If the kind are the same, it compares the subkinds of the new and old groups. If the new group has a smaller subkind, it merges the groups by replacing the old group with the new one.


```cpp
/* returns the result of merge, or NULL if not merged */
static __hwloc_inline hwloc_obj_t
hwloc__insert_try_merge_group(hwloc_topology_t topology, hwloc_obj_t old, hwloc_obj_t new)
{
  if (new->type == HWLOC_OBJ_GROUP && old->type == HWLOC_OBJ_GROUP) {
    /* which group do we keep? */
    if (new->attr->group.dont_merge) {
      if (old->attr->group.dont_merge)
	/* nobody wants to be merged */
	return NULL;

      /* keep the new one, it doesn't want to be merged */
      hwloc_replace_linked_object(old, new);
      topology->modified = 1;
      return new;

    } else {
      if (old->attr->group.dont_merge)
	/* keep the old one, it doesn't want to be merged */
	return old;

      /* compare subkinds to decide which group to keep */
      if (new->attr->group.kind < old->attr->group.kind) {
        /* keep smaller kind */
	hwloc_replace_linked_object(old, new);
        topology->modified = 1;
      }
      return old;
    }
  }

  if (new->type == HWLOC_OBJ_GROUP && !new->attr->group.dont_merge) {

    if (old->type == HWLOC_OBJ_PU && new->attr->group.kind == HWLOC_GROUP_KIND_MEMORY)
      /* Never merge Memory groups with PU, we don't want to attach Memory under PU */
      return NULL;

    /* Remove the Group now. The normal ignore code path wouldn't tell us whether the Group was removed or not,
     * while some callers need to know (at least hwloc_topology_insert_group()).
     */
    return old;

  } else if (old->type == HWLOC_OBJ_GROUP && !old->attr->group.dont_merge) {

    if (new->type == HWLOC_OBJ_PU && old->attr->group.kind == HWLOC_GROUP_KIND_MEMORY)
      /* Never merge Memory groups with PU, we don't want to attach Memory under PU */
      return NULL;

    /* Replace the Group with the new object contents
     * and let the caller free the new object
     */
    hwloc_replace_linked_object(old, new);
    topology->modified = 1;
    return old;

  } else {
    /* cannot merge */
    return NULL;
  }
}

```

This is a function that modifys the topology of the CUR and OBJ objects. It takes in an object, which is either a topology object or a CUR object, and modifies it to become part of another topology object. It does this by first finding where the object belongs in the current topology object, and then re-ordering the children of the object, if necessary.

The function has several arguments:

* `cur`: the CUR object to modify.
* `setres`: a boolean indicating whether the modification was done to equal the memory location.
* `obj`: the object to modify.
* `child`: the child object of the `obj` object.
* `topology`: the topology object to modify.

The function returns a pointer to the modified `obj`.

Note: This function assumes that the `setres` argument is not `HWLOC_OBJ_EQUAL`, in which case the function will not modify the object and will return.


```cpp
/*
 * The main insertion routine, only used for CPU-side object (normal types)
 * uisng cpuset only (or complete_cpuset).
 *
 * Try to insert OBJ in CUR, recurse if needed.
 * Returns the object if it was inserted,
 * the remaining object it was merged,
 * NULL if failed to insert.
 */
static struct hwloc_obj *
hwloc___insert_object_by_cpuset(struct hwloc_topology *topology, hwloc_obj_t cur, hwloc_obj_t obj,
			        const char *reason)
{
  hwloc_obj_t child, next_child = NULL, tmp;
  /* These will always point to the pointer to their next last child. */
  hwloc_obj_t *cur_children = &cur->first_child;
  hwloc_obj_t *obj_children = &obj->first_child;
  /* Pointer where OBJ should be put */
  hwloc_obj_t *putp = NULL; /* OBJ position isn't found yet */

  assert(!hwloc__obj_type_is_memory(obj->type));

  /* Iteration with prefetching to be completely safe against CHILD removal.
   * The list is already sorted by cpuset, and there's no intersection between siblings.
   */
  for (child = cur->first_child, child ? next_child = child->next_sibling : NULL;
       child;
       child = next_child, child ? next_child = child->next_sibling : NULL) {

    int res = hwloc_obj_cmp_sets(obj, child);
    int setres = res;

    if (res == HWLOC_OBJ_EQUAL) {
      hwloc_obj_t merged = hwloc__insert_try_merge_group(topology, child, obj);
      if (merged)
	return merged;
      /* otherwise compare actual types to decide of the inclusion */
      res = hwloc_type_cmp(obj, child);
    }

    switch (res) {
      case HWLOC_OBJ_EQUAL:
	/* Two objects with same type.
	 * Groups are handled above.
	 */
	merge_insert_equal(obj, child);
	/* Already present, no need to insert.  */
	return child;

      case HWLOC_OBJ_INCLUDED:
	/* OBJ is strictly contained is some child of CUR, go deeper.  */
	return hwloc___insert_object_by_cpuset(topology, child, obj, reason);

      case HWLOC_OBJ_INTERSECTS:
        report_insert_error(obj, child, "intersection without inclusion", reason);
	goto putback;

      case HWLOC_OBJ_DIFFERENT:
        /* OBJ should be a child of CUR before CHILD, mark its position if not found yet. */
	if (!putp && hwloc__object_cpusets_compare_first(obj, child) < 0)
	  /* Don't insert yet, there could be intersect errors later */
	  putp = cur_children;
	/* Advance cur_children.  */
	cur_children = &child->next_sibling;
	break;

      case HWLOC_OBJ_CONTAINS:
	/* OBJ contains CHILD, remove CHILD from CUR */
	*cur_children = child->next_sibling;
	child->next_sibling = NULL;
	/* Put CHILD in OBJ */
	*obj_children = child;
	obj_children = &child->next_sibling;
	child->parent = obj;
	if (setres == HWLOC_OBJ_EQUAL) {
	  obj->memory_first_child = child->memory_first_child;
	  child->memory_first_child = NULL;
	  for(tmp=obj->memory_first_child; tmp; tmp = tmp->next_sibling)
	    tmp->parent = obj;
	}
	break;
    }
  }
  /* cur/obj_children points to last CUR/OBJ child next_sibling pointer, which must be NULL. */
  assert(!*obj_children);
  assert(!*cur_children);

  /* Put OBJ where it belongs, or in last in CUR's children.  */
  if (!putp)
    putp = cur_children;
  obj->next_sibling = *putp;
  *putp = obj;
  obj->parent = cur;

  topology->modified = 1;
  return obj;

 putback:
  /* OBJ cannot be inserted.
   * Put-back OBJ children in CUR and return an error.
   */
  if (putp)
    cur_children = putp; /* No need to try to insert before where OBJ was supposed to go */
  else
    cur_children = &cur->first_child; /* Start from the beginning */
  /* We can insert in order, but there can be holes in the middle. */
  while ((child = obj->first_child) != NULL) {
    /* Remove from OBJ */
    obj->first_child = child->next_sibling;
    /* Find child position in CUR, and reinsert it. */
    while (*cur_children && hwloc__object_cpusets_compare_first(*cur_children, child) < 0)
      cur_children = &(*cur_children)->next_sibling;
    child->next_sibling = *cur_children;
    *cur_children = child;
    child->parent = cur;
  }
  return NULL;
}

```

这段代码定义了一个名为 `hwloc__find_obj_covering_memory_cpuset` 的函数，它的功能是查找一个内存分配区域，该区域与给定的 CPU 集是否相等。如果找到了这样的区域，函数将返回该区域；否则，函数将返回根对象。

该函数与 `hwloc_get_obj_covering_cpuset()` 函数的主要区别在于，它首先查找父对象中的 CPU 集，而不是从根对象开始查找。这意味着即使根对象的 CPU 集未被设置，也可以在父对象中插入它。另外，该函数返回第一个与给定 CPU 集相等的子对象，而不是遍历子对象以查找相等对象。

总之，该函数是用于在给定的 topology 对象中查找与指定 CPU 集相等的内存分配区域，以便于在 hwloc_topology 对象中进行内存分配。


```cpp
/* this differs from hwloc_get_obj_covering_cpuset() by:
 * - not looking at the parent cpuset first, which means we can insert
 *   below root even if root PU bits are not set yet (PU are inserted later).
 * - returning the first child that exactly matches instead of walking down in case
 *   of identical children.
 */
static struct hwloc_obj *
hwloc__find_obj_covering_memory_cpuset(struct hwloc_topology *topology, hwloc_obj_t parent, hwloc_bitmap_t cpuset)
{
  hwloc_obj_t child = hwloc_get_child_covering_cpuset(topology, cpuset, parent);
  if (!child)
    return parent;
  if (child && hwloc_bitmap_isequal(child->cpuset, cpuset))
    return child;
  return hwloc__find_obj_covering_memory_cpuset(topology, child, cpuset);
}

```

This is a function definition for `root_remove_group` which removes a `Group` object from the tree rooted at `root`, given a list of `Group` objects and the object you want to remove.

It works by first checking if the `Group` object to be removed is topologically distinct from the root, and then checking if it is a member of the root's `cpuset`. If it is not topologically distinct from the root, it returns the `Group` object.

If the `Group` is not topologically distinct from the root, it returns the `Group` object.

If the `Group` is not a member of the root's `cpuset`, it creates a new intermediate `Group` and returns it.

If the `Group` is not inserted into the tree, it returns the `Group` object.

It is important to note that the function assumes that the input `topology` is valid and that the `Group` object has been initialized properly.


```cpp
static struct hwloc_obj *
hwloc__find_insert_memory_parent(struct hwloc_topology *topology, hwloc_obj_t obj,
                                 const char *reason)
{
  hwloc_obj_t parent, group, result;

  if (hwloc_bitmap_iszero(obj->cpuset)) {
    /* CPU-less go in dedicated group below root */
    parent = topology->levels[0][0];

  } else {
    /* find the highest obj covering the cpuset */
    parent = hwloc__find_obj_covering_memory_cpuset(topology, topology->levels[0][0], obj->cpuset);
    if (!parent) {
      /* fallback to root */
      parent = hwloc_get_root_obj(topology);
    }

    if (parent->type == HWLOC_OBJ_PU) {
      /* Never attach to PU, try parent */
      parent = parent->parent;
      assert(parent);
    }

    /* TODO: if root->cpuset was updated earlier, we would be sure whether the group will remain identical to root */
    if (parent != topology->levels[0][0] && hwloc_bitmap_isequal(parent->cpuset, obj->cpuset))
      /* that parent is fine */
      return parent;
  }

  if (!hwloc_filter_check_keep_object_type(topology, HWLOC_OBJ_GROUP))
    /* even if parent isn't perfect, we don't want an intermediate group */
    return parent;

  /* need to insert an intermediate group for attaching the NUMA node */
  group = hwloc_alloc_setup_object(topology, HWLOC_OBJ_GROUP, HWLOC_UNKNOWN_INDEX);
  if (!group)
    /* failed to create the group, fallback to larger parent */
    return parent;

  group->attr->group.kind = HWLOC_GROUP_KIND_MEMORY;
  group->cpuset = hwloc_bitmap_dup(obj->cpuset);
  group->complete_cpuset = hwloc_bitmap_dup(obj->complete_cpuset);
  /* we could duplicate nodesets too but hwloc__insert_object_by_cpuset()
   * doesn't actually need it. and it could prevent future calls from reusing
   * that groups for other NUMA nodes.
   */
  if (!group->cpuset != !obj->cpuset
      || !group->complete_cpuset != !obj->complete_cpuset) {
    /* failed to create the group, fallback to larger parent */
    hwloc_free_unlinked_object(group);
    return parent;
  }

  result = hwloc__insert_object_by_cpuset(topology, parent, group, reason);
  if (!result) {
    /* failed to insert, fallback to larger parent */
    return parent;
  }

  assert(result == group);
  return group;
}

```

This function appears to be part of a higher-level system that manages NUMA nodes and their associated memory objects. It takes in an object, a current node, and an error reason, and returns either the same object or a new object that has been attached to the current node via a process of adding its nodeset to the existing NUMA node.

The function first checks if the object and the current node are identical，如果是， the function reports an error and returns NULL. If the objects are not identical, the function then checks if the current node is a memcache, and if it is, it checks if the depth of the memcache is the same as the depth of the object's cache. If the conditions are not met, the function inserts the object into the memcache. If the depth of the memcache is greater than the depth of the object's cache, the function moves the object higher in the hierarchy.

If the current node is not a memcache, the function first checks if the object's type is also a memcache. If it is not, the function then checks if the current node is a memcache or a high-level object like a hardware location (hwloc). If the current node is a memcache or a high-level object, the function reports the error and returns NULL. If the current node is not a memcache or a high-level object, the function appends the object to the end of the list and returns it.

The function also modifys the current node's parent to point to the new object, and if the parent is not a memcache, it also modifies the topology to reflect the change.


```cpp
/* only works for MEMCACHE and NUMAnode with a single bit in nodeset */
static hwloc_obj_t
hwloc___attach_memory_object_by_nodeset(struct hwloc_topology *topology, hwloc_obj_t parent,
					hwloc_obj_t obj, const char *reason)
{
  hwloc_obj_t *curp = &parent->memory_first_child;
  unsigned first = hwloc_bitmap_first(obj->nodeset);

  while (*curp) {
    hwloc_obj_t cur = *curp;
    unsigned curfirst = hwloc_bitmap_first(cur->nodeset);

    if (first < curfirst) {
      /* insert before cur */
      obj->next_sibling = cur;
      *curp = obj;
      obj->memory_first_child = NULL;
      obj->parent = parent;
      topology->modified = 1;
      return obj;
    }

    if (first == curfirst) {
      /* identical nodeset */
      if (obj->type == HWLOC_OBJ_NUMANODE) {
	if (cur->type == HWLOC_OBJ_NUMANODE) {
	  /* identical NUMA nodes? ignore the new one */
          report_insert_error(obj, cur, "NUMAnodes with identical nodesets", reason);
	  return NULL;
	}
	assert(cur->type == HWLOC_OBJ_MEMCACHE);
	/* insert the new NUMA node below that existing memcache */
	return hwloc___attach_memory_object_by_nodeset(topology, cur, obj, reason);

      } else {
	assert(obj->type == HWLOC_OBJ_MEMCACHE);
	if (cur->type == HWLOC_OBJ_MEMCACHE) {
	  if (cur->attr->cache.depth == obj->attr->cache.depth)
	    /* memcache with same nodeset and depth, ignore the new one */
	    return NULL;
	  if (cur->attr->cache.depth > obj->attr->cache.depth)
	    /* memcache with higher cache depth is actually *higher* in the hierarchy
	     * (depth starts from the NUMA node).
	     * insert the new memcache below the existing one
	     */
	    return hwloc___attach_memory_object_by_nodeset(topology, cur, obj, reason);
	}
	/* insert the memcache above the existing memcache or numa node */
	obj->next_sibling = cur->next_sibling;
	cur->next_sibling = NULL;
	obj->memory_first_child = cur;
	cur->parent = obj;
	*curp = obj;
	obj->parent = parent;
	topology->modified = 1;
	return obj;
      }
    }

    curp = &cur->next_sibling;
  }

  /* append to the end of the list */
  obj->next_sibling = NULL;
  *curp = obj;
  obj->memory_first_child = NULL;
  obj->parent = parent;
  topology->modified = 1;
  return obj;
}

```

这段代码的作用是实现了一个名为 `hwloc__attach_memory_object()` 的函数，用于将一个给定的内存对象挂载到指定的normal parent上。

函数接收三个参数：`topology`，表示要 attach 的内存对象的topology结构；`parent`，表示要 attach的内存对象的normal parent；`obj`，表示要 attach的内存对象；`reason`，表示给内存对象附加的理由，例如挂载到正常parent上。

函数先检查给定的内存对象 `obj` 是否存在于normal parent上，如果是，则检查该内存对象的nodeset是否与normal parent的nodeset相同。如果不是，则需要先初始化nodeset，或者检查该nodeset是否包含整个normal parent的nodeset。

如果已经检查完所有参数，并且确定要attach的内存对象存在于normal parent上，则检查给定的nodeset是否与normal parent的nodeset相同。如果不是，则函数返回一个空指针。

最后，函数会尝试在CPU-side parent上传播nodeset。


```cpp
/* Attach the given memory object below the given normal parent.
 *
 * Only the nodeset is used to find the location inside memory children below parent.
 *
 * Nodeset inclusion inside the given memory hierarchy is guaranteed by this function,
 * but nodesets are not propagated to CPU-side parent yet. It will be done by
 * propagate_nodeset() later.
 */
struct hwloc_obj *
hwloc__attach_memory_object(struct hwloc_topology *topology, hwloc_obj_t parent,
			    hwloc_obj_t obj, const char *reason)
{
  hwloc_obj_t result;

  assert(parent);
  assert(hwloc__obj_type_is_normal(parent->type));

  /* Check the nodeset */
  if (!obj->nodeset || hwloc_bitmap_iszero(obj->nodeset))
    return NULL;
  /* Initialize or check the complete nodeset */
  if (!obj->complete_nodeset) {
    obj->complete_nodeset = hwloc_bitmap_dup(obj->nodeset);
  } else if (!hwloc_bitmap_isincluded(obj->nodeset, obj->complete_nodeset)) {
    return NULL;
  }
  /* Neither ACPI nor Linux support multinode mscache */
  assert(hwloc_bitmap_weight(obj->nodeset) == 1);

```

这段代码的作用是检查在给一个 NUMA 网络中的节点添加完好的数据结构后，整个数据结构在多大程度上影响了它的层次结构和位置。如果发现整个数据结构与预期的不同，那么就需要释放原始数据结构。

具体来说，这段代码首先检查如果不使用 `hwloc_nodeset_is_in_same_set` 函数，那么整个数据结构将无法正确地集成到系统的时间复杂度中。因此，我们需要使用 `hwloc_nodeset_is_in_same_set` 函数来比较当前节点和父节点的 CPU 集合是否在同一个集合中。

接下来，代码会尝试从系统时间复杂度中添加节点，并使用 `hwloc_bitmap_copy` 函数将父节点的 CPU 集合复制到当前对象的 CPU 集合中，以便在添加新的节点时能够正确地设置其层次结构和位置。

最后，如果添加整个数据结构失败，或者已经将数据结构集成到了系统时间复杂度中，那么就需要释放原始数据结构，并返回结果。


```cpp
#if 0
  /* TODO: enable this instead of hack in fixup_sets once NUMA nodes are inserted late */
  /* copy the parent cpuset in case it's larger than expected.
   * we could also keep the cpuset smaller than the parent and say that a normal-parent
   * can have multiple memory children with smaller cpusets.
   * However, the user decided the ignore Groups, so hierarchy/locality loss is expected.
   */
  hwloc_bitmap_copy(obj->cpuset, parent->cpuset);
  hwloc_bitmap_copy(obj->complete_cpuset, parent->complete_cpuset);
#endif

  result = hwloc___attach_memory_object_by_nodeset(topology, parent, obj, reason);
  if (result == obj) {
    /* Add the bit to the top sets, and to the parent CPU-side object */
    if (obj->type == HWLOC_OBJ_NUMANODE) {
      hwloc_bitmap_set(topology->levels[0][0]->nodeset, obj->os_index);
      hwloc_bitmap_set(topology->levels[0][0]->complete_nodeset, obj->os_index);
    }
  }
  if (result != obj) {
    /* either failed to insert, or got merged, free the original object */
    hwloc_free_unlinked_object(obj);
  }
  return result;
}

```

这段代码是一个hwloc_obj的结构体，用于在给定的topology中插入一个对象obj。当insert_object_by_cpuset函数被调用时，它将检查obj是否满足hwloc_obj_type_is_special类型的条件，如果不是，则说明该对象不属于特殊类型，可以进行正常的插入操作。

另外，如果insert_object_by_cpuset函数的第一个参数topology指向的topology结构中，已存在与obj具有相同type的topology，则函数将直接返回obj，无需插入。

如果insert_object_by_cpuset函数的第一个参数topology中，存在一个由非空set（normal或complete，cpuset或nodeset）组成的topology，则函数将尝试使用topology中指定set中的所有set中的元素来设置obj，并将其返回。如果topology中指定的set中所有元素都是空集，则表示可以插入一个空对象，在这种情况下，函数将在不抛出任何错误的情况下返回obj。

如果topology中指定的set中存在非空set，则函数将在set中查找与obj具有相同type的set，并尝试使用set中的元素来设置obj。如果set中包含的元素数量与obj的元素数量不同，则函数将在set中具有相同type的set中插入obj，而不是使用set中所有元素。

如果尝试插入obj时出现任何错误，则函数将在不抛出任何错误的情况下返回null。


```cpp
/* insertion routine that lets you change the error reporting callback */
struct hwloc_obj *
hwloc__insert_object_by_cpuset(struct hwloc_topology *topology, hwloc_obj_t root,
			       hwloc_obj_t obj, const char *reason)
{
  struct hwloc_obj *result;

#ifdef HWLOC_DEBUG
  assert(!hwloc__obj_type_is_special(obj->type));

  /* we need at least one non-NULL set (normal or complete, cpuset or nodeset) */
  assert(obj->cpuset || obj->complete_cpuset || obj->nodeset || obj->complete_nodeset);
  /* we support the case where all of them are empty.
   * it may happen when hwloc__find_insert_memory_parent()
   * inserts a Group for a CPU-less NUMA-node.
   */
```

这段代码是一个 C 语言函数，它的作用是插入一个二进制数据块（Object）到二进制数据块的根数据集中。如果二进制数据块已经被插入到根数据集中，则这段代码会检查是否可以继续在根数据集中插入。如果可以插入，则插入二进制数据块并返回其根指针；如果不能插入，则释放二进制数据块和根指针，并返回 NULL。

具体来说，这段代码可以分为以下几个步骤：

1. 检查要插入的二进制数据块是否已经存在于内存中，如果是，则直接返回 NULL；
2. 检查二进制数据块的类型是否为内存，如果是，则尝试在当前节点（由 topology 提供的链表）中查找插入内存数据块的根节点，如果找到了，则插入数据块并返回根指针；
3. 如果没有找到，则在当前节点开始遍历，直到找到根节点或者遍历结束；
4. 如果已经遍历完所有的节点，仍然没有找到数据块，则插入数据块到链表的根节点；
5. 检查二进制数据块是否存在于根数据集中，如果是，则检查它是否已经被设置为 1，如果是，则说明数据块已经被设置为二进制数据块的所有节点，可以继续向根数据集中插入；
6. 否则，说明插入失败，需要释放数据块和根指针并返回 NULL。


```cpp
#endif

  if (hwloc__obj_type_is_memory(obj->type)) {
    if (!root) {
      root = hwloc__find_insert_memory_parent(topology, obj, reason);
      if (!root) {
	hwloc_free_unlinked_object(obj);
	return NULL;
      }
    }
    return hwloc__attach_memory_object(topology, root, obj, reason);
  }

  if (!root)
    /* Start at the top. */
    root = topology->levels[0][0];

  result = hwloc___insert_object_by_cpuset(topology, root, obj, reason);
  if (result && result->type == HWLOC_OBJ_PU) {
      /* Add the bit to the top sets */
      if (hwloc_bitmap_isset(result->cpuset, result->os_index))
	hwloc_bitmap_set(topology->levels[0][0]->cpuset, result->os_index);
      hwloc_bitmap_set(topology->levels[0][0]->complete_cpuset, result->os_index);
  }
  if (result != obj) {
    /* either failed to insert, or got merged, free the original object */
    hwloc_free_unlinked_object(obj);
  }
  return result;
}

```

This function appears to be a part of a higher-level object allocator that manages memory-bound data structures with a topology-based organization. It takes an object struct and adds it to the appropriate list of children based on its type, and optionally updates the topology of the object.

The function first checks the current bitmap of the highest level (topology level 0) to see if the object has already been added and the topology bits for the parent and child nodesets are set. If the topology bit for the object is not set, the function adds the object to the end of the list and sets the topology bit for the parent and child nodesets to indicate that the object is a top-level object.

If the object is a memory-bound object, the function checks if the topology bit for the memory set is set. If it is not set, the function appends the object to the end of the memory list and sets the topology bit for the parent and child nodesets to indicate that the object is a top-level object.

If the object is a process or a user-level I/O object, the function appends it to the end of the list and sets the topology bit for the parent and child nodesets to indicate that the object is a top-level object.

The function also performs some additional processing if the object is a process:

* If the object is a process, it checks if the topology bit for the process set is set. If it is not set, the function sets the topology bit for the parent and child nodesets to indicate that the object is a top-level process.
* If the object is a user-level I/O object, it sets the topology bit for the parent and child nodesets to indicate that the object is a top-level I/O object.

Finally, the function updates the topology of the object by setting the modified bit to indicate that the object has been modified and the parent object is no longer the top object.

If the function fails to add the object to the parent or child nodeset due to conflicts with other objects in the same topology, it does not raise any errors and simply returns, without modifying the topology of the object.


```cpp
/* the default insertion routine warns in case of error.
 * it's used by most backends */
void
hwloc_insert_object_by_parent(struct hwloc_topology *topology, hwloc_obj_t parent, hwloc_obj_t obj)
{
  hwloc_obj_t *current;

  if (obj->type == HWLOC_OBJ_MISC) {
    /* Append to the end of the Misc list */
    for (current = &parent->misc_first_child; *current; current = &(*current)->next_sibling);
  } else if (hwloc__obj_type_is_io(obj->type)) {
    /* Append to the end of the I/O list */
    for (current = &parent->io_first_child; *current; current = &(*current)->next_sibling);
  } else if (hwloc__obj_type_is_memory(obj->type)) {
    /* Append to the end of the memory list */
    for (current = &parent->memory_first_child; *current; current = &(*current)->next_sibling);
    /* Add the bit to the top sets */
    if (obj->type == HWLOC_OBJ_NUMANODE) {
      if (hwloc_bitmap_isset(obj->nodeset, obj->os_index))
	hwloc_bitmap_set(topology->levels[0][0]->nodeset, obj->os_index);
      hwloc_bitmap_set(topology->levels[0][0]->complete_nodeset, obj->os_index);
    }
  } else {
    /* Append to the end of the list.
     * The caller takes care of inserting children in the right cpuset order, without intersection between them.
     * Duplicating doesn't need to check the order since the source topology is supposed to be OK already.
     * XML reorders if needed, and fails on intersecting siblings.
     * Other callers just insert random objects such as I/O or Misc, no cpuset issue there.
     */
    for (current = &parent->first_child; *current; current = &(*current)->next_sibling);
    /* Add the bit to the top sets */
    if (obj->type == HWLOC_OBJ_PU) {
      if (hwloc_bitmap_isset(obj->cpuset, obj->os_index))
	hwloc_bitmap_set(topology->levels[0][0]->cpuset, obj->os_index);
      hwloc_bitmap_set(topology->levels[0][0]->complete_cpuset, obj->os_index);
    }
  }

  *current = obj;
  obj->parent = parent;
  obj->next_sibling = NULL;
  topology->modified = 1;
}

```

该代码定义了一个名为`hwloc_obj_t`的硬件设备对象类型，以及一个名为`hwloc_topology_t`的topology结构体。`hwloc_alloc_setup_object`函数使用`hwloc_topology_t`作为其输入参数，并返回一个硬件设备对象`obj`。

此函数首先调用`hwloc_tma_malloc`函数来在topology的tma中分配内存，该函数将返回一个指向内存分配位置的指针。如果内存分配成功，则将`obj`初始化为`hwloc_obj_type_t`类型的零值，并设置其`type`为所需的对象类型，`os_index`为指定的操作系统索引，`gp_index`为topology中下一个gp索引的位置，`attr`是一个指向分配的内存的指针。如果分配内存失败，则释放内存并返回`NULL`。

如果`hwloc_topology_t`中包含了`dontfree`成员，则调用`hwloc_tma_malloc`函数时，分配的内存将在`dontfree`成员中释放。


```cpp
hwloc_obj_t
hwloc_alloc_setup_object(hwloc_topology_t topology,
			 hwloc_obj_type_t type, unsigned os_index)
{
  struct hwloc_obj *obj = hwloc_tma_malloc(topology->tma, sizeof(*obj));
  if (!obj)
    return NULL;
  memset(obj, 0, sizeof(*obj));
  obj->type = type;
  obj->os_index = os_index;
  obj->gp_index = topology->next_gp_index++;
  obj->attr = hwloc_tma_malloc(topology->tma, sizeof(*obj->attr));
  if (!obj->attr) {
    assert(!topology->tma || !topology->tma->dontfree); /* this tma cannot fail to allocate */
    free(obj);
    return NULL;
  }
  memset(obj->attr, 0, sizeof(*obj->attr));
  /* do not allocate the cpuset here, let the caller do it */
  return obj;
}

```

这段代码定义了一个名为 `hwloc_obj_t` 的结构体，表示硬件位置（hwloc）中的对象。接着定义了一个名为 `hwloc_topology_alloc_group_object` 的函数，该函数接受一个 `hwloc_topology` 的结构体作为参数，返回一个指向 `hwloc_topology_obj_t` 结构的指针，该结构体包含了 `hwloc_topology` 中的所有已知成员。

函数首先检查传入的 `topology` 结构体是否已加载，如果不加载，函数将返回一个错误码。然后检查 `topology` 中的 `adopted_shmem_addr` 是否为真，如果是，函数将返回一个错误码。否则，函数将返回一个指向 `hwloc_alloc_setup_object` 的函数指针，该函数接受 `topology` 和 `HWLOC_OBJ_GROUP` 作为参数，并尝试设置 `hwloc_obj_group` 类型的对象。


```cpp
hwloc_obj_t
hwloc_topology_alloc_group_object(struct hwloc_topology *topology)
{
  if (!topology->is_loaded) {
    /* this could actually work, see insert() below */
    errno = EINVAL;
    return NULL;
  }
  if (topology->adopted_shmem_addr) {
    errno = EPERM;
    return NULL;
  }
  return hwloc_alloc_setup_object(topology, HWLOC_OBJ_GROUP, HWLOC_UNKNOWN_INDEX);
}

```

This code appears to be a part of a larger software system that manages power and resource usage across a cluster of machines.

The function `hwloc_get_next_obj_by_type` returns the next object of a particular type along the topology tree rooted at `topology`. If there is no such object, it returns NULL.

The function `hwloc_bitmap_isset` checks if the bitmap of a given nodeset is set to 1.

The function `hwloc_bitmap_or` ORs the contents of the given `cpuset` with the contents of the bitmap.

The function `hwloc_obj_cmp_sets` compares the `cpuset` of a given object with the `cpuset` of the root object. If the objects are included in the same `root` object, the function `hwloc__insert_object_by_cpuset` inserts the object into the `root` object. If the objects are not included in the same `root` object, the function `root` is the new parent object.

The function `hwloc__insert_object_by_cpuset` inserts the object with the given `cpuset` into the root object.

The function `hwloc_topology_reconnect` reconnects the levels of the topology tree if the topology has a hierarchical structure.

The function `hwloc_set_group_depth` sets the depth of the root object.

The function `hwloc_propagate_symmetric_subtree` propagates the symmetric subtree of the topology up to the given depth.

The function `hwloc_obj_add_children_sets` adds the children of the given object to its `children_sets` property.

It is not clear from the provided code what the purpose of all of these functions is, or how they are part of a larger software system.


```cpp
static void hwloc_propagate_symmetric_subtree(hwloc_topology_t topology, hwloc_obj_t root);
static void propagate_total_memory(hwloc_obj_t obj);
static void hwloc_set_group_depth(hwloc_topology_t topology);
static void hwloc_connect_children(hwloc_obj_t parent);
static int hwloc_connect_levels(hwloc_topology_t topology);
static int hwloc_connect_special_levels(hwloc_topology_t topology);

hwloc_obj_t
hwloc_topology_insert_group_object(struct hwloc_topology *topology, hwloc_obj_t obj)
{
  hwloc_obj_t res, root;
  int cmp;

  if (!topology->is_loaded) {
    /* this could actually work, we would just need to disable connect_children/levels below */
    hwloc_free_unlinked_object(obj);
    errno = EINVAL;
    return NULL;
  }
  if (topology->adopted_shmem_addr) {
    errno = EPERM;
    return NULL;
  }

  if (topology->type_filter[HWLOC_OBJ_GROUP] == HWLOC_TYPE_FILTER_KEEP_NONE) {
    hwloc_free_unlinked_object(obj);
    errno = EINVAL;
    return NULL;
  }

  root = hwloc_get_root_obj(topology);
  if (obj->cpuset)
    hwloc_bitmap_and(obj->cpuset, obj->cpuset, root->cpuset);
  if (obj->complete_cpuset)
    hwloc_bitmap_and(obj->complete_cpuset, obj->complete_cpuset, root->complete_cpuset);
  if (obj->nodeset)
    hwloc_bitmap_and(obj->nodeset, obj->nodeset, root->nodeset);
  if (obj->complete_nodeset)
    hwloc_bitmap_and(obj->complete_nodeset, obj->complete_nodeset, root->complete_nodeset);

  if ((!obj->cpuset || hwloc_bitmap_iszero(obj->cpuset))
      && (!obj->complete_cpuset || hwloc_bitmap_iszero(obj->complete_cpuset))) {
    /* we'll insert by cpuset, so build cpuset from the nodeset */
    hwloc_const_bitmap_t nodeset = obj->nodeset ? obj->nodeset : obj->complete_nodeset;
    hwloc_obj_t numa;

    if ((!obj->nodeset || hwloc_bitmap_iszero(obj->nodeset))
	&& (!obj->complete_nodeset || hwloc_bitmap_iszero(obj->complete_nodeset))) {
      hwloc_free_unlinked_object(obj);
      errno = EINVAL;
      return NULL;
    }

    if (!obj->cpuset) {
      obj->cpuset = hwloc_bitmap_alloc();
      if (!obj->cpuset) {
	hwloc_free_unlinked_object(obj);
	return NULL;
      }
    }

    numa = NULL;
    while ((numa = hwloc_get_next_obj_by_type(topology, HWLOC_OBJ_NUMANODE, numa)) != NULL)
      if (hwloc_bitmap_isset(nodeset, numa->os_index))
	hwloc_bitmap_or(obj->cpuset, obj->cpuset, numa->cpuset);
  }
  /* FIXME insert by nodeset to group NUMAs even if CPUless? */

  cmp = hwloc_obj_cmp_sets(obj, root);
  if (cmp == HWLOC_OBJ_INCLUDED) {
    res = hwloc__insert_object_by_cpuset(topology, NULL, obj, NULL /* do not show errors on stdout */);
  } else {
    /* just merge root */
    res = root;
  }

  if (!res)
    return NULL;

  if (res != obj && res->type != HWLOC_OBJ_GROUP)
    /* merged, not into a Group, nothing to update */
    return res;

  /* res == obj means that the object was inserted.
   * We need to reconnect levels, fill all its cpu/node sets,
   * compute its total memory, group depth, etc.
   *
   * res != obj usually means that our new group was merged into an
   * existing object, no need to recompute anything.
   * However, if merging with an existing group, depending on their kinds,
   * the contents of obj may overwrite the contents of the old group.
   * This requires reconnecting levels, filling sets, recomputing total memory, etc.
   */

  /* properly inserted */
  hwloc_obj_add_children_sets(res);
  if (hwloc_topology_reconnect(topology, 0) < 0)
    return NULL;

  hwloc_propagate_symmetric_subtree(topology, topology->levels[0][0]);
  hwloc_set_group_depth(topology);

```

这段代码是一个 C 语言函数，名为 `hwloc_topology_insert_misc_object`。它属于 `hwloc_topology_profile` 类别，用于在 `hwloc_topology` 结构中插入一个名为 `MiscObject` 的对象。

具体来说，这段代码在以下步骤中做了什么呢：

1. 如果当前进程配置了 `HWLOC_DEBUG_CHECK` 环境变量，那么会首先检查一下是否有相关的配置，然后调用 `hwloc_topology_check` 函数。
2. 如果 `hwloc_topology_check` 函数成功返回，那么会继续执行以下操作：
	1. 尝试从环境变量中获取 `MiscObject` 父对象的索引，如果没有找到，则表示系统不支持插入这个对象，需要抛出错误并返回 `NULL`。
	2. 如果 `MiscObject` 父对象成功被找到，那么会将其插入到 `topology` 结构中，使用 `hwloc_insert_object_by_parent` 函数将其插入到 `parent` 对象所在的层级。
	3. 调用 `hwloc_topology_reconnect` 函数，将 `topology` 结构中的对象重新连接，使得它们能够通信。注意，这个步骤在 `hwloc_topology_insert_misc_object` 函数中是没有定义的，需要在调用它的函数中进行定义。


```cpp
#ifndef HWLOC_DEBUG
  if (getenv("HWLOC_DEBUG_CHECK"))
#endif
    hwloc_topology_check(topology);

  return res;
}

hwloc_obj_t
hwloc_topology_insert_misc_object(struct hwloc_topology *topology, hwloc_obj_t parent, const char *name)
{
  hwloc_obj_t obj;

  if (topology->type_filter[HWLOC_OBJ_MISC] == HWLOC_TYPE_FILTER_KEEP_NONE) {
    errno = EINVAL;
    return NULL;
  }

  if (!topology->is_loaded) {
    errno = EINVAL;
    return NULL;
  }
  if (topology->adopted_shmem_addr) {
    errno = EPERM;
    return NULL;
  }

  obj = hwloc_alloc_setup_object(topology, HWLOC_OBJ_MISC, HWLOC_UNKNOWN_INDEX);
  if (name)
    obj->name = strdup(name);

  hwloc_insert_object_by_parent(topology, parent, obj);

  /* FIXME: only connect misc parent children and misc level,
   * but this API is likely not performance critical anyway
   */
  hwloc_topology_reconnect(topology, 0);

```

这段代码是一个 C 语言函数，定义在 `hwloc_debug.h` 头文件中。它通过使用环境变量 `HWLOC_DEBUG_CHECK` 来判断是否需要对调试信息进行输出。

函数首先检查 `HWLOC_DEBUG_CHECK` 环境变量是否已经被设置，如果已经被设置，则执行函数体内的内容。函数体中包含两个步骤：

1. 尝试从 `topology` 头文件中获取 `hwloc_topology_check` 函数的返回值，如果已经定义好的函数可以被调用，则调用它并进行一些检查。
2. 如果 `hwloc_topology_check` 函数返回结果，则继续执行后续操作，否则递归调用 `hwloc_get_highest_obj_covering_complete_cpuset` 函数，该函数将在所有子节点中查找与给定 `set` 完全相同的 `hwloc_obj_t` 对象，并返回该对象。

函数 `hwloc_get_highest_obj_covering_complete_cpuset` 在 `hwloc_topology_t` 头文件中定义，它接收 `topology` 头文件和给定的 `hwloc_const_cpuset_t` `set` 作为参数。函数的作用与上面定义的 `hwloc_get_root_obj` 函数类似，只是它会在 `hwloc_topology_check` 函数返回结果后继续寻找最佳子节点，从而返回最佳对象。


```cpp
#ifndef HWLOC_DEBUG
  if (getenv("HWLOC_DEBUG_CHECK"))
#endif
    hwloc_topology_check(topology);

  return obj;
}

/* assuming set is included in the topology complete_cpuset
 * and all objects have a proper complete_cpuset,
 * return the best one containing set.
 * if some object are equivalent (same complete_cpuset), return the highest one.
 */
static hwloc_obj_t
hwloc_get_highest_obj_covering_complete_cpuset (hwloc_topology_t topology, hwloc_const_cpuset_t set)
{
  hwloc_obj_t current = hwloc_get_root_obj(topology);
  hwloc_obj_t child;

  if (hwloc_bitmap_isequal(set, current->complete_cpuset))
    /* root cpuset is exactly what we want, no need to look at children, we want the highest */
    return current;

 recurse:
  /* find the right child */
  for_each_child(child, current) {
    if (hwloc_bitmap_isequal(set, child->complete_cpuset))
      /* child puset is exactly what we want, no need to look at children, we want the highest */
      return child;
    if (!hwloc_bitmap_iszero(child->complete_cpuset) && hwloc_bitmap_isincluded(set, child->complete_cpuset))
      break;
  }

  if (child) {
    current = child;
    goto recurse;
  }

  /* no better child */
  return current;
}

```

This function appears to retrieve a completecpuset for a given topology. It works by first getting the highest object covering the given cpuset, and then checking if it's a valid group. If it is, it returns that object. If not, it checks if there's any other object that covers the cpuset and if it's not a conflicting group. If it is not a valid group, it checks if it's a normal case or a conflicting case. If it's a normal case, it returns the object. If it's a conflicting case, it inserts an intermediate group, and then returns that group. If the group can't get merged or if it's inserted too late, it returns the highest object covering the cpuset.


```cpp
hwloc_obj_t
hwloc_find_insert_io_parent_by_complete_cpuset(struct hwloc_topology *topology, hwloc_cpuset_t cpuset)
{
  hwloc_obj_t group_obj, largeparent, parent;

  /* restrict to the existing complete cpuset to avoid errors later */
  hwloc_bitmap_and(cpuset, cpuset, hwloc_topology_get_complete_cpuset(topology));
  if (hwloc_bitmap_iszero(cpuset))
    /* remaining cpuset is empty, invalid */
    return NULL;

  largeparent = hwloc_get_highest_obj_covering_complete_cpuset(topology, cpuset);
  if (hwloc_bitmap_isequal(largeparent->complete_cpuset, cpuset)
      || !hwloc_filter_check_keep_object_type(topology, HWLOC_OBJ_GROUP))
    /* Found a valid object (normal case) */
    return largeparent;

  /* we need to insert an intermediate group */
  group_obj = hwloc_alloc_setup_object(topology, HWLOC_OBJ_GROUP, HWLOC_UNKNOWN_INDEX);
  if (!group_obj)
    /* Failed to insert the exact Group, fallback to largeparent */
    return largeparent;

  group_obj->complete_cpuset = hwloc_bitmap_dup(cpuset);
  hwloc_bitmap_and(cpuset, cpuset, hwloc_topology_get_topology_cpuset(topology));
  group_obj->cpuset = hwloc_bitmap_dup(cpuset);
  group_obj->attr->group.kind = HWLOC_GROUP_KIND_IO;
  parent = hwloc__insert_object_by_cpuset(topology, largeparent, group_obj, "topology:io_parent");
  if (!parent)
    /* Failed to insert the Group, maybe a conflicting cpuset */
    return largeparent;

  /* Group couldn't get merged or we would have gotten the right largeparent earlier */
  assert(parent == group_obj);

  /* Group inserted without being merged, everything OK, setup its sets */
  hwloc_obj_add_children_sets(group_obj);

  return parent;
}

```

这段代码定义了一个名为 `hwloc_memory_page_type_compare` 的函数，它的参数为两个指向 `hwloc_memory_page_type_s` 结构体的指针 `_a` 和 `_b`。

这个函数的作用是用来比较两个 `hwloc_memory_page_type_s` 结构体的大小关系，用于输出 `hwloc_memory_page_type_compare` 函数时需要传递给用户的一个参数。

具体来说，这个函数首先通过 `_a` 和 `_b` 两个指针，获取到两个 `hwloc_memory_page_type_s` 结构体，然后对两个结构体的大小关系进行比较。如果 `b` 的 `size` 字段为 `NULL`，表示这个 `hwloc_memory_page_type_s` 结构体没有对应的 `size` 字段，那么函数返回一个负的整数，表示 `_a` 中的 `size` 字段大于 `_b` 中的 `size` 字段。否则，如果 `a` 的 `size` 字段大于 `b` 的 `size` 字段，那么函数返回一个正的整数，表示 `_a` 中的 `size` 字段小于 `_b` 中的 `size` 字段。最后，如果两个 `size` 字段相等，函数返回 0。

这个 `hwloc_memory_page_type_compare` 函数在 `hwloc_system_ monitor` 中被使用，用于比较不同 `hwloc_memory_page_type_s` 结构体的大小，以便输出一个整数表示哪个 `hwloc_memory_page_type_s` 结构体更大。


```cpp
static int hwloc_memory_page_type_compare(const void *_a, const void *_b)
{
  const struct hwloc_memory_page_type_s *a = _a;
  const struct hwloc_memory_page_type_s *b = _b;
  /* consider 0 as larger so that 0-size page_type go to the end */
  if (!b->size)
    return -1;
  /* don't cast a-b in int since those are ullongs */
  if (b->size == a->size)
    return 0;
  return a->size < b->size ? -1 : 1;
}

/* Propagate memory counts */
static void
```

这段代码是一个名为`propagate_total_memory`的函数，属于`hwloc_obj_t`类型的数据结构。它用于将一个硬件资源对象（`hwloc_obj_t`）中的总内存数向上传播到其所有子资源上，并将所有子资源中的内存数累加到最终的结果中。

在函数内部，首先重置对象的总内存数，使其为0。然后，使用`for_each_child`和`for_each_memory_child`循环遍历对象的子资源，递归调用`propagate_total_memory`函数，并将子资源返回的内存数累加到当前对象的内存数中。

接下来，代码中有一些判断，这些判断会在遍历子资源后，对对象的总内存数进行修改。具体来说，当对象类型为`HWLOC_OBJ_NUMANODE`时，会累加到`obj->total_memory`中；当`obj->attr->numanode.page_types_len`存在时，会对子资源的类型进行排序；当`obj->attr->numanode.page_types`中包含页类型`XML`时，會在插入对象之后添加页类型。


```cpp
propagate_total_memory(hwloc_obj_t obj)
{
  hwloc_obj_t child;
  unsigned i;

  /* reset total before counting local and children memory */
  obj->total_memory = 0;

  /* Propagate memory up. */
  for_each_child(child, obj) {
    propagate_total_memory(child);
    obj->total_memory += child->total_memory;
  }
  for_each_memory_child(child, obj) {
    propagate_total_memory(child);
    obj->total_memory += child->total_memory;
  }
  /* No memory under I/O or Misc */

  if (obj->type == HWLOC_OBJ_NUMANODE) {
    obj->total_memory += obj->attr->numanode.local_memory;

    if (obj->attr->numanode.page_types_len) {
      /* By the way, sort the page_type array.
       * Cannot do it on insert since some backends (e.g. XML) add page_types after inserting the object.
       */
      qsort(obj->attr->numanode.page_types, obj->attr->numanode.page_types_len, sizeof(*obj->attr->numanode.page_types), hwloc_memory_page_type_compare);
      /* Ignore 0-size page_types, they are at the end */
      for(i=obj->attr->numanode.page_types_len; i>=1; i--)
	if (obj->attr->numanode.page_types[i-1].size)
	  break;
      obj->attr->numanode.page_types_len = i;
    }
  }
}

```

This is a C code that describes an iterative process for inserting memory objects into a hierarchical树. The specificity of this code is its use of memory object handles, which allows for the insertion of memory objects at any level of the hierarchy.

The code starts by iterating through the children of a given node, and for each child, it recursively checks for the presence of a memory object and, if it is, updates its correspondingcpuset and nodeset. Then, it checks if the memory object is complete and, if not, it creates a copy of the memory object and copies its completion set to the corresponding complete set for the parent node.

If the memory object is not a complete set, the code also checks if it is a memory object and, if it is, it copies itscpuset to the corresponding child node and itscomplete_cpuset to the corresponding complete set for the parent node.

The code then proceeds to the next step of the recursion, where it checks if the given node is a memory object and, if it is, it updates its corresponding sets and, if it is not, it sets the in_memory_list flag to 1 and continues the iteration.

Finally, the code includes a switch-case block that, if the node is a memory object and the memory object is a child of a node with a memory object that has been inserted recently, the code copies the memory object'scpuset to the memory object'scpuset and sets the in_memory_list flag to 1.


```cpp
/* Now that root sets are ready, propagate them to children
 * by allocating missing sets and restricting existing ones.
 */
static void
fixup_sets(hwloc_obj_t obj)
{
  int in_memory_list;
  hwloc_obj_t child;

  child = obj->first_child;
  in_memory_list = 0;
  /* iterate over normal children first, we'll come back for memory children later */

  /* FIXME: if memory objects are inserted late, we should update their cpuset and complete_cpuset at insertion instead of here */
 iterate:
  while (child) {
    /* our cpuset must be included in our parent's one */
    hwloc_bitmap_and(child->cpuset, child->cpuset, obj->cpuset);
    hwloc_bitmap_and(child->nodeset, child->nodeset, obj->nodeset);
    /* our complete_cpuset must be included in our parent's one, but can be larger than our cpuset */
    if (child->complete_cpuset) {
      hwloc_bitmap_and(child->complete_cpuset, child->complete_cpuset, obj->complete_cpuset);
    } else {
      child->complete_cpuset = hwloc_bitmap_dup(child->cpuset);
    }
    if (child->complete_nodeset) {
      hwloc_bitmap_and(child->complete_nodeset, child->complete_nodeset, obj->complete_nodeset);
    } else {
      child->complete_nodeset = hwloc_bitmap_dup(child->nodeset);
    }

    if (hwloc_obj_type_is_memory(child->type)) {
      /* update memory children cpusets in case some CPU-side parent was removed */
      hwloc_bitmap_copy(child->cpuset, obj->cpuset);
      hwloc_bitmap_copy(child->complete_cpuset, obj->complete_cpuset);
    }

    fixup_sets(child);
    child = child->next_sibling;
  }

  /* switch to memory children list if any */
  if (!in_memory_list && obj->memory_first_child) {
    child = obj->memory_first_child;
    in_memory_list = 1;
    goto iterate;
  }

  /* No sets in I/O or Misc */
}

```

这段代码定义了一个名为`hwloc_obj_add_other_obj_sets`的函数，它的作用是设置一个`hwloc_obj_t`对象的子对象。

函数接受两个参数：`dst`表示需要设置的子对象的句柄，`src`表示设置子对象的源对象。函数内部定义了一个名为`ADD_OTHER_OBJ_SET`的函数，用于设置子对象。

`ADD_OTHER_OBJ_SET`函数的实现如下：
```cppperl
if ((src)->_set) {
   if (!(dst)->_set) {
       (dst)->_set = hwloc_bitmap_alloc();
   }
   hwloc_bitmap_or((dst)->_set, (src)->_set, (src)->_set);
}
```
首先检查`src`对象是否设置，如果设置，则检查`dst`对象是否设置。如果`dst`对象未设置，则创建一个`hwloc_bitmap_t`对象并将其设置为`hwloc_bitmap_alloc`函数的返回值。然后，使用`hwloc_bitmap_or`函数将`src`对象的设置和`dst`对象的设置进行按位与操作，得到新的设置。这样，当`dst`对象也设置时，子对象将使用现有的设置，否则需要创建一个新的设置。

根据定义，该函数的作用是设置`dst`对象为给定句柄的子对象的设置，如果`src`对象已设置，则检查`dst`对象是否设置，如果`dst`对象未设置，则创建一个新的设置并将其与`src`对象的设置进行按位与操作。


```cpp
/* Setup object cpusets/nodesets by OR'ing its children. */
int
hwloc_obj_add_other_obj_sets(hwloc_obj_t dst, hwloc_obj_t src)
{
#define ADD_OTHER_OBJ_SET(_dst, _src, _set)			\
  if ((_src)->_set) {						\
    if (!(_dst)->_set)						\
      (_dst)->_set = hwloc_bitmap_alloc();			\
    hwloc_bitmap_or((_dst)->_set, (_dst)->_set, (_src)->_set);	\
  }
  ADD_OTHER_OBJ_SET(dst, src, cpuset);
  ADD_OTHER_OBJ_SET(dst, src, complete_cpuset);
  ADD_OTHER_OBJ_SET(dst, src, nodeset);
  ADD_OTHER_OBJ_SET(dst, src, complete_nodeset);
  return 0;
}

```

这段代码定义了一个名为 `hwloc_obj_add_children_sets` 的函数，属于 `hwloc_obj_t` 类别，用于将一个 `hwloc_obj_t` 类型的对象添加到其子对象中。

该函数接收一个 `hwloc_obj_t` 类型的参数 `obj`，并使用一系列循环遍历该参数的所有子对象。在每次循环中，函数调用另一个名为 `hwloc_obj_add_other_obj_sets` 的函数，将参数 `obj` 和其子对象中的其他对象设置为彼此的属性。

在这段注释中，说明了一下该函数的作用。首先，它将一个 `hwloc_obj_t` 类型的对象添加到其子对象中。接着，使用一个循环遍历所有的子对象，并调用 `hwloc_obj_add_other_obj_sets` 函数，将参数 `obj` 和子对象中的其他对象设置为彼此的属性。最后，说明该函数不需要考虑 `Misc` 类型的子对象，因为它们不包含 `publicUid` 成员，因此不需要进行调整。


```cpp
int
hwloc_obj_add_children_sets(hwloc_obj_t obj)
{
  hwloc_obj_t child;
  for_each_child(child, obj) {
    hwloc_obj_add_other_obj_sets(obj, child);
  }
  /* No need to look at Misc children, they contain no PU. */
  return 0;
}

/* CPU objects are inserted by cpusets, we know their cpusets are properly included.
 * We just need fixup_sets() to make sure they aren't too wide.
 *
 * Within each memory hierarchy, nodeset are consistent as well.
 * However they must be propagated to their CPU-side parents.
 *
 * A memory object nodeset consists of NUMA nodes below it.
 * A normal object nodeset consists in NUMA nodes attached to any
 * of its children or parents.
 */
```

This function appears to be used to add a local nodeset to an existing `hwloc` object, such as an `hwloc_device`. The `loc_bitmap_zero` function appears to be used to create a bitmap that represents the local nodeset of the `hwloc_device`. The local nodeset is then propagated to the CPU children, and to the children that are not already part of the `hwloc_device` topology.


```cpp
static void
propagate_nodeset(hwloc_obj_t obj)
{
  hwloc_obj_t child;

  /* Start our nodeset from the parent one.
   * It was emptied at root, and it's being filled with local nodes
   * in that branch of the tree as we recurse down.
   */
  if (!obj->nodeset)
    obj->nodeset = hwloc_bitmap_alloc();
  if (obj->parent)
    hwloc_bitmap_copy(obj->nodeset, obj->parent->nodeset);
  else
    hwloc_bitmap_zero(obj->nodeset);

  /* Don't clear complete_nodeset, just make sure it contains nodeset.
   * We cannot clear the complete_nodeset at root and rebuild it down because
   * some bits may correspond to offline/disallowed NUMA nodes missing in the topology.
   */
  if (!obj->complete_nodeset)
    obj->complete_nodeset = hwloc_bitmap_dup(obj->nodeset);
  else
    hwloc_bitmap_or(obj->complete_nodeset, obj->complete_nodeset, obj->nodeset);

  /* now add our local nodeset */
  for_each_memory_child(child, obj) {
    /* add memory children nodesets to ours */
    hwloc_bitmap_or(obj->nodeset, obj->nodeset, child->nodeset);
    hwloc_bitmap_or(obj->complete_nodeset, obj->complete_nodeset, child->complete_nodeset);
    /* no need to recurse because hwloc__attach_memory_object()
     * makes sure nodesets are consistent within each memory hierarchy.
     */
  }

  /* Propagate our nodeset to CPU children. */
  for_each_child(child, obj) {
    propagate_nodeset(child);
  }

  /* Propagate CPU children specific nodesets back to us.
   *
   * We cannot merge these two loops because we don't want to first child
   * nodeset to be propagated back to us and then down to the second child.
   * Each child may have its own local nodeset,
   * each of them is propagated to us, but not to other children.
   */
  for_each_child(child, obj) {
    hwloc_bitmap_or(obj->nodeset, obj->nodeset, child->nodeset);
    hwloc_bitmap_or(obj->complete_nodeset, obj->complete_nodeset, child->complete_nodeset);
  }

  /* No nodeset under I/O or Misc */

}

```

这段代码的作用是移除操作系统中无法被任何进程访问的设置。它主要针对Linux系统，通过`hwloc_topology_t`和`hwloc_obj_t`来表示硬件和对象的类型。

在函数开始时，创建了一个名为`child`的临时变量，以及一个名为`topology`的`hwloc_topology_t`对象和一个名为`obj`的`hwloc_obj_t`对象。

接着，代码使用`hwloc_bitmap_and`函数对目标对象的`cpuset`和`nodeset`进行与操作，将操作系统中所有未被分配的设置也加入到这两个集合中。这样，在函数内部，所有没有被使用的设置都会被加入到这两个集合中。

接下来，通过`for_each_child`循环遍历目标对象的子对象，并递归调用`remove_unused_sets`函数。这样，就可以确保所有没有被使用的设置都会被正确地移除。

最后，在循环的最后部分，添加了一个`/*`注释。这个注释表示该函数不会在I/O或内存子对象上执行`remove_unused_sets`函数，因为这些子对象在系统启动时可能已经被设置。


```cpp
static void
remove_unused_sets(hwloc_topology_t topology, hwloc_obj_t obj)
{
  hwloc_obj_t child;

  hwloc_bitmap_and(obj->cpuset, obj->cpuset, topology->allowed_cpuset);
  hwloc_bitmap_and(obj->nodeset, obj->nodeset, topology->allowed_nodeset);

  for_each_child(child, obj)
    remove_unused_sets(topology, child);
  for_each_memory_child(child, obj)
    remove_unused_sets(topology, child);
  /* No cpuset under I/O or Misc */
}

```

这段代码是一个名为 `hwloc__filter_bridges` 的函数，属于 `hwloc_topology_t` 类的成员函数。它的作用是过滤 `hwloc_topology_t` 结构中的 I/O 子树，并对深度进行递归。

具体来说，代码首先遍历 I/O 子树，然后对于每个 I/O 子节点，递归地处理它的子节点，直到达到函数设定的最大深度。在处理子节点的过程中，根据不同的过滤策略，对子节点进行不同的处理，如保留、删除或更新。最终，函数会在 depths 为 0（即处理完成后）或者不需要处理时，设置 `topology->modified` 属性为 1，表明发生了修改。


```cpp
static void
hwloc__filter_bridges(hwloc_topology_t topology, hwloc_obj_t root, unsigned depth)
{
  hwloc_obj_t child, *pchild;

  /* filter I/O children and recurse */
  for_each_io_child_safe(child, root, pchild) {
    enum hwloc_type_filter_e filter = topology->type_filter[child->type];

    /* recurse into grand-children */
    hwloc__filter_bridges(topology, child, depth+1);

    child->attr->bridge.depth = depth;

    /* remove bridges that have no child,
     * and pci-to-non-pci bridges (pcidev) that no child either.
     * keep NVSwitch since they may be used in NVLink matrices.
     */
    if (filter == HWLOC_TYPE_FILTER_KEEP_IMPORTANT
	&& !child->io_first_child
        && (child->type == HWLOC_OBJ_BRIDGE
            || (child->type == HWLOC_OBJ_PCI_DEVICE && (child->attr->pcidev.class_id >> 8) == 0x06
                && (!child->subtype || strcmp(child->subtype, "NVSwitch"))))) {
      unlink_and_free_single_object(pchild);
      topology->modified = 1;
    }
  }
}

```



这是一个C语言的函数，定义在hwloc_filter_bridges.hwloc_obj_t类型的库中。

hwloc_filter_bridges函数的作用是处理硬件布局中的桥接类对象，用于将具有相同数据类型的桥接类对象连接在一起，以形成一个更大的桥接类对象。该函数接受两个参数：topology表示硬件布局的拓扑结构，parent表示要遍历的桥接对象的父对象。函数内部通过递归调用hwloc_filter_bridges函数来将相同的桥接类对象连接在一起，然后将子桥接对象与父桥接对象进行连接。

hwloc__reorder_children函数的作用是将硬件布局中的桥接类对象按照其父子关系重新排列，以重新分配系统中所有桥接类对象的优先级。该函数接受一个参数：parent，表示要遍历的硬件布局对象的父对象。函数内部通过循环遍历所有桥接类对象，并使用三个指针变量prev、child和children，分别代表当前层、子桥接对象和父桥接对象的父对象。在循环过程中，首先将当前桥接对象（child）添加到子桥接对象（children）的前面，然后将子桥接对象（children）添加到父桥接对象（parent）的前面。接着，通过不断递归调用hwloc__filter_bridges函数，以重新排列整个系统中所有桥接类对象的优先级。最后，在遍历结束后，将父桥接对象（parent）的first_child属性设置为当前层桥接对象的引用，以便在父对象中查找子桥接对象。

注意：在该代码中，对于Misc和I/O类型的桥接类对象，排序规则的特殊处理没有给出明确的说明。


```cpp
static void
hwloc_filter_bridges(hwloc_topology_t topology, hwloc_obj_t parent)
{
  hwloc_obj_t child = parent->first_child;
  while (child) {
    hwloc_filter_bridges(topology, child);
    child = child->next_sibling;
  }

  hwloc__filter_bridges(topology, parent, 0);
}

void
hwloc__reorder_children(hwloc_obj_t parent)
{
  /* move the children list on the side */
  hwloc_obj_t *prev, child, children = parent->first_child;
  parent->first_child = NULL;
  while (children) {
    /* dequeue child */
    child = children;
    children = child->next_sibling;
    /* find where to enqueue it */
    prev = &parent->first_child;
    while (*prev && hwloc__object_cpusets_compare_first(child, *prev) > 0)
      prev = &((*prev)->next_sibling);
    /* enqueue */
    child->next_sibling = *prev;
    *prev = child;
  }
  /* No ordering to enforce for Misc or I/O children. */
}

```

这段代码的作用是移除具有 emptyCPUTSET 和 emptyNodesET 的对象，以及具有 I/O 子对象的虚拟机。在移除时，代码会首先检查对象是否具有正常类型，如果是，则检查该对象是否有空集。如果是，则不需要移动该对象。如果不是，则继续移除。

代码中使用 for_each_child_safe 和 for_each_memory_child_safe 函数遍历对象的所有子对象，如果找到了空集或 I/O 子对象，将跳过这些子对象。在 if 语句中，如果发现对象是 normal 类型，但是其 CPUTSET 或 NodesET 中有一个为空，则需要移动该对象。在 if 语句中，如果发现对象是 memory 类型，并且其 NodesET 中为空，则需要移动该对象。

代码中还包含一个 remove_empty 函数，该函数接收一个 hwloc_topology_t 类型的顶图和 hwloc_obj_t 类型的对象，该函数会首先检查对象是否具有 normal 类型，如果是，则检查该对象是否有空集。如果是，则不需要移动该对象。如果不是，则继续移除。


```cpp
/* Remove all normal children whose cpuset is empty,
 * and memory children whose nodeset is empty.
 * Also don't remove objects that have I/O children, but ignore Misc.
 */
static void
remove_empty(hwloc_topology_t topology, hwloc_obj_t *pobj)
{
  hwloc_obj_t obj = *pobj, child, *pchild;

  for_each_child_safe(child, obj, pchild)
    remove_empty(topology, pchild);
  for_each_memory_child_safe(child, obj, pchild)
    remove_empty(topology, pchild);
  /* No cpuset under I/O or Misc */

  if (obj->first_child /* only remove if all children were removed above, so that we don't remove parents of NUMAnode */
      || obj->memory_first_child /* only remove if no memory attached there */
      || obj->io_first_child /* only remove if no I/O is attached there */)
    /* ignore Misc */
    return;

  if (hwloc__obj_type_is_normal(obj->type)) {
    if (!hwloc_bitmap_iszero(obj->cpuset))
      return;
  } else {
    assert(hwloc__obj_type_is_memory(obj->type));
    if (!hwloc_bitmap_iszero(obj->nodeset))
      return;
  }

  hwloc_debug("%s", "\nRemoving empty object ");
  hwloc_debug_print_object(0, obj);
  unlink_and_free_single_object(pobj);
  topology->modified = 1;
}

```

这段代码的作用是重置 HWLOC  topology 中的类型深度（depth），并保持原始 topology 中的结构。主要实现了以下两个主要功能：

1. 在函数中，通过遍历 topology 中的所有类型深度，将其重置为 HWLOC_TYPE_DEPTH_UNKNOWN。这主要是在遍历过程中，如果发现某个类型深度存在，则将其类型属性设置为 HWLOC_TYPE_DEPTH_UNKNOWN。

2. 通过传入一个整数（i）来控制类型合并。在函数中，对于传入的 i，根据 topology 中的 level_nbobjects（级别数组）判断是否继续合并该层。如果当前层存在一个或多个 don't_merge 属性，设置返回值为 1，否则继续合并。


```cpp
/* reset type depth before modifying levels (either reconnecting or filtering/keep_structure) */
static void
hwloc_reset_normal_type_depths(hwloc_topology_t topology)
{
  unsigned i;
  for (i=HWLOC_OBJ_TYPE_MIN; i<=HWLOC_OBJ_GROUP; i++)
    topology->type_depth[i] = HWLOC_TYPE_DEPTH_UNKNOWN;
  /* type contiguity is asserted in topology_check() */
  topology->type_depth[HWLOC_OBJ_DIE] = HWLOC_TYPE_DEPTH_UNKNOWN;
}

static int
hwloc_dont_merge_group_level(hwloc_topology_t topology, unsigned i)
{
  unsigned j;

  /* Don't merge some groups in that level? */
  for(j=0; j<topology->level_nbobjects[i]; j++)
    if (topology->levels[i][j]->attr->group.dont_merge)
      return 1;

  return 0;
}

```

这段代码定义了一个名为 `hwloc_compare_levels_structure` 的函数，它比较了 `hwloc_topology_t` 结构中的第 `i` 级和第 `i-1` 级层之间的关系。

具体来说，函数接收一个 `hwloc_topology_t` 结构的实例，以及一个 `unsigned` 类型的整数 `i`。函数首先检查 `topology->levels[i][0]->type` 是否为 `HWLOC_OBJ_PU`，如果是，则认为层 $i$ 是最低的，返回 `0`。然后，函数检查 `topology->level_nbobjects[i-1]` 和 `topology->level_nbobjects[i]` 是否相等，如果不相等，则认为层 $i$ 和 $i-1$ 不同，返回 `-1`。

接下来，函数遍历层 $i` 内的所有对象，检查它们的父对象是否属于同一层，如果不是，则返回 `-1`。对于每个对象，函数还检查它的 `arity` 是否为 `1`，如果不是，则返回 `-1`。最后，如果 `topology->levels[i][0]->memory_arity` 为 `true`，但是层 $i$ 内的所有对象都具有父对象，则认为层 $i$ 内的对象是不可达的，返回 `-1`。

综上所述，函数 `hwloc_compare_levels_structure` 比较了层 $i$ 和层 $i-1$ 之间的关系，返回 `0`，如果层 $i$ 是最低的，返回 `-1`，否则返回 `-1`。


```cpp
/* compare i-th and i-1-th levels structure */
static int
hwloc_compare_levels_structure(hwloc_topology_t topology, unsigned i)
{
  int checkmemory = (topology->levels[i][0]->type == HWLOC_OBJ_PU);
  unsigned j;

  if (topology->level_nbobjects[i-1] != topology->level_nbobjects[i])
    return -1;

  for(j=0; j<topology->level_nbobjects[i]; j++) {
    if (topology->levels[i-1][j] != topology->levels[i][j]->parent)
      return -1;
    if (topology->levels[i-1][j]->arity != 1)
      return -1;
    if (checkmemory && topology->levels[i-1][j]->memory_arity)
      /* don't merge PUs if there's memory above */
      return -1;
  }
  /* same number of objects with arity 1 above, no problem */
  return 0;
}

```

This function appears to be part of the `hwloc_topology_ reconnect()`
function. It rebuilds the topology object by connecting all the nodes
and updating the object and type depths, if some levels were removed.

It does this by first disconnecting all the special levels (i.e. levels with
multiple children), and then connecting all the remaining special levels
(i.e. levels with only one child).

If some levels were removed, the function updates the object and type depths
for those levels, and also updates the modified flag to reflect that some
levels have been removed.

It does all this by first storing the modified flag in a global
variable, and then using that flag to decide which levels to modify.

If the reconnection fails, the function returns -1.


```cpp
/* return > 0 if any level was removed.
 * performs its own reconnect internally if needed
 */
static int
hwloc_filter_levels_keep_structure(hwloc_topology_t topology)
{
  unsigned i, j;
  int res = 0;

  if (topology->modified) {
    /* WARNING: hwloc_topology_reconnect() is duplicated partially here
     * and at the end of this function:
     * - we need normal levels before merging.
     * - and we'll need to update special levels after merging.
     */
    hwloc_connect_children(topology->levels[0][0]);
    if (hwloc_connect_levels(topology) < 0)
      return -1;
  }

  /* start from the bottom since we'll remove intermediate levels */
  for(i=topology->nb_levels-1; i>0; i--) {
    int replacechild = 0, replaceparent = 0;
    hwloc_obj_t obj1 = topology->levels[i-1][0];
    hwloc_obj_t obj2 = topology->levels[i][0];
    hwloc_obj_type_t type1 = obj1->type;
    hwloc_obj_type_t type2 = obj2->type;

    /* Check whether parents and/or children can be replaced */
    if (topology->type_filter[type1] == HWLOC_TYPE_FILTER_KEEP_STRUCTURE) {
      /* Parents can be ignored in favor of children.  */
      replaceparent = 1;
      if (type1 == HWLOC_OBJ_GROUP && hwloc_dont_merge_group_level(topology, i-1))
	replaceparent = 0;
    }
    if (topology->type_filter[type2] == HWLOC_TYPE_FILTER_KEEP_STRUCTURE) {
      /* Children can be ignored in favor of parents.  */
      replacechild = 1;
      if (type1 == HWLOC_OBJ_GROUP && hwloc_dont_merge_group_level(topology, i))
	replacechild = 0;
    }
    if (!replacechild && !replaceparent)
      /* no ignoring */
      continue;
    /* Decide which one to actually replace */
    if (replaceparent && replacechild) {
      /* If both may be replaced, look at obj_type_priority */
      if (obj_type_priority[type1] >= obj_type_priority[type2])
	replaceparent = 0;
      else
	replacechild = 0;
    }
    /* Are these levels actually identical? */
    if (hwloc_compare_levels_structure(topology, i) < 0)
      continue;
    hwloc_debug("may merge levels #%u=%s and #%u=%s\n",
		i-1, hwloc_obj_type_string(type1), i, hwloc_obj_type_string(type2));

    /* OK, remove intermediate objects from the tree. */
    for(j=0; j<topology->level_nbobjects[i]; j++) {
      hwloc_obj_t parent = topology->levels[i-1][j];
      hwloc_obj_t child = topology->levels[i][j];
      unsigned k;
      if (replacechild) {
	/* move child's children to parent */
	parent->first_child = child->first_child;
	parent->last_child = child->last_child;
	parent->arity = child->arity;
	free(parent->children);
	parent->children = child->children;
	child->children = NULL;
	/* update children parent */
	for(k=0; k<parent->arity; k++)
	  parent->children[k]->parent = parent;
	/* append child memory/io/misc children to parent */
	if (child->memory_first_child) {
	  append_siblings_list(&parent->memory_first_child, child->memory_first_child, parent);
	  parent->memory_arity += child->memory_arity;
	}
	if (child->io_first_child) {
	  append_siblings_list(&parent->io_first_child, child->io_first_child, parent);
	  parent->io_arity += child->io_arity;
	}
	if (child->misc_first_child) {
	  append_siblings_list(&parent->misc_first_child, child->misc_first_child, parent);
	  parent->misc_arity += child->misc_arity;
	}
	hwloc_free_unlinked_object(child);
      } else {
	/* replace parent with child in grand-parent */
	if (parent->parent) {
	  parent->parent->children[parent->sibling_rank] = child;
	  child->sibling_rank = parent->sibling_rank;
	  if (!parent->sibling_rank) {
	    parent->parent->first_child = child;
	    /* child->prev_sibling was already NULL, child was single */
	  } else {
	    child->prev_sibling = parent->parent->children[parent->sibling_rank-1];
	    child->prev_sibling->next_sibling = child;
	  }
	  if (parent->sibling_rank == parent->parent->arity-1) {
	    parent->parent->last_child = child;
	    /* child->next_sibling was already NULL, child was single */
	  } else {
	    child->next_sibling = parent->parent->children[parent->sibling_rank+1];
	    child->next_sibling->prev_sibling = child;
	  }
	  /* update child parent */
	  child->parent = parent->parent;
	} else {
	  /* make child the new root */
	  topology->levels[0][0] = child;
	  child->parent = NULL;
	}
	/* prepend parent memory/io/misc children to child */
	if (parent->memory_first_child) {
	  prepend_siblings_list(&child->memory_first_child, parent->memory_first_child, child);
	  child->memory_arity += parent->memory_arity;
	}
	if (parent->io_first_child) {
	  prepend_siblings_list(&child->io_first_child, parent->io_first_child, child);
	  child->io_arity += parent->io_arity;
	}
	if (parent->misc_first_child) {
	  prepend_siblings_list(&child->misc_first_child, parent->misc_first_child, child);
	  child->misc_arity += parent->misc_arity;
	}
	hwloc_free_unlinked_object(parent);
	/* prev/next_sibling will be updated below in another loop */
      }
    }
    if (replaceparent && i>1) {
      /* Update sibling list within modified parent->parent arrays */
      for(j=0; j<topology->level_nbobjects[i]; j++) {
	hwloc_obj_t child = topology->levels[i][j];
	unsigned rank = child->sibling_rank;
	child->prev_sibling = rank > 0 ? child->parent->children[rank-1] : NULL;
	child->next_sibling = rank < child->parent->arity-1 ? child->parent->children[rank+1] : NULL;
      }
    }

    /* Update levels so that the next reconnect isn't confused */
    if (replaceparent) {
      /* Removing level i-1, so move levels [i..nb_levels-1] to [i-1..] */
      free(topology->levels[i-1]);
      memmove(&topology->levels[i-1],
	      &topology->levels[i],
	      (topology->nb_levels-i)*sizeof(topology->levels[i]));
      memmove(&topology->level_nbobjects[i-1],
	      &topology->level_nbobjects[i],
	      (topology->nb_levels-i)*sizeof(topology->level_nbobjects[i]));
      hwloc_debug("removed parent level %s at depth %u\n",
		  hwloc_obj_type_string(type1), i-1);
    } else {
      /* Removing level i, so move levels [i+1..nb_levels-1] and later to [i..] */
      free(topology->levels[i]);
      memmove(&topology->levels[i],
	      &topology->levels[i+1],
	      (topology->nb_levels-1-i)*sizeof(topology->levels[i]));
      memmove(&topology->level_nbobjects[i],
	      &topology->level_nbobjects[i+1],
	      (topology->nb_levels-1-i)*sizeof(topology->level_nbobjects[i]));
      hwloc_debug("removed child level %s at depth %u\n",
		  hwloc_obj_type_string(type2), i);
    }
    topology->level_nbobjects[topology->nb_levels-1] = 0;
    topology->levels[topology->nb_levels-1] = NULL;
    topology->nb_levels--;

    res++;
  }

  if (res > 0) {
    /* Update object and type depths if some levels were removed */
    hwloc_reset_normal_type_depths(topology);
    for(i=0; i<topology->nb_levels; i++) {
      hwloc_obj_type_t type = topology->levels[i][0]->type;
      for(j=0; j<topology->level_nbobjects[i]; j++)
	topology->levels[i][j]->depth = (int)i;
      if (topology->type_depth[type] == HWLOC_TYPE_DEPTH_UNKNOWN)
	topology->type_depth[type] = (int)i;
      else
	topology->type_depth[type] = HWLOC_TYPE_DEPTH_MULTIPLE;
    }
  }


  if (res > 0 || topology-> modified) {
    /* WARNING: hwloc_topology_reconnect() is duplicated partially here
     * and at the beginning of this function.
     * If we merged some levels, some child+parent special children lisst
     * may have been merged, hence specials level might need reordering,
     * So reconnect special levels only here at the end
     * (it's not needed at the beginning of this function).
     */
    if (hwloc_connect_special_levels(topology) < 0)
      return -1;
    topology->modified = 0;
  }

  return 0;
}

```

This is a C function for determining if a given array is symmetric or not. It takes an array and a topology as input and returns a boolean value.

The function has several error-handling blocks to handle cases where the input is not in the expected format. For example, if the topology is not a tree, the function will return immediately.

The function first checks if the input array is empty or has only one element. If it is empty or has only one element, it is not possible to determine if the array is symmetric or not, so the function will exit early.

If the input array has more than one element, the function checks if it is symmetric. If it is not symmetric, the function will exit early.

If the input array is symmetric, the function will check if all levels of the array are equal in all levels. If any level is not equal, the function will exit early.

If the input array is not symmetric and all levels are equal, the function will exit early.

If the input array is not symmetric, all levels are not equal, and the function will exit early.

If the input array is not symmetric, some levels are not equal, and the function will exit early.

If the input array is not symmetric, some levels are not equal, and the function will exit early.

If the input array is not symmetric, some levels are not equal, and the function will exit early.

If the input array is not symmetric, some levels are not equal, and the function will exit early.

If the input array is not symmetric, some levels are not equal, and the function will exit early.

If the input array is not symmetric, some levels are not equal, and the function will exit early.

If the input array is not symmetric, some levels are not equal, and the function will exit early。


```cpp
static void
hwloc_propagate_symmetric_subtree(hwloc_topology_t topology, hwloc_obj_t root)
{
  hwloc_obj_t child;
  unsigned arity = root->arity;
  hwloc_obj_t *array;
  int ok;

  /* assume we're not symmetric by default */
  root->symmetric_subtree = 0;

  /* if no child, we are symmetric */
  if (!arity)
    goto good;

  /* FIXME ignore memory just like I/O and Misc? */

  /* look at normal children only, I/O and Misc are ignored.
   * return if any child is not symmetric.
   */
  ok = 1;
  for_each_child(child, root) {
    hwloc_propagate_symmetric_subtree(topology, child);
    if (!child->symmetric_subtree)
      ok = 0;
  }
  if (!ok)
    return;
  /* Misc and I/O children do not care about symmetric_subtree */

  /* if single child is symmetric, we're good */
  if (arity == 1)
    goto good;

  /* now check that children subtrees are identical.
   * just walk down the first child in each tree and compare their depth and arities
   */
  array = malloc(arity * sizeof(*array));
  if (!array)
    return;
  memcpy(array, root->children, arity * sizeof(*array));
  while (1) {
    unsigned i;
    /* check current level arities and depth */
    for(i=1; i<arity; i++)
      if (array[i]->depth != array[0]->depth
	  || array[i]->arity != array[0]->arity) {
	free(array);
	return;
      }
    if (!array[0]->arity)
      /* no more children level, we're ok */
      break;
    /* look at first child of each element now */
    for(i=0; i<arity; i++)
      array[i] = array[i]->first_child;
  }
  free(array);

  /* everything went fine, we're symmetric */
 good:
  root->symmetric_subtree = 1;
}

```

这段代码定义了一个名为`hwloc_set_group_depth`的静态函数，其作用是设置指定分层结构中对象属性的组层深度。

具体来说，代码遍历分层结构中的每一层，对于每一层，首先检查该层是否包含一个名为`HWLOC_OBJ_GROUP`的节点，如果是，则执行以下操作：

1. 遍历该层中所有对象的层次结构，跳过第一层（即根节点）。
2. 对于每一对象，获取其`attr->group`属性的`depth`成员，将其设置为组层深度。
3. 增加该层的组层深度。

在`hwloc_connect_levels`函数中，调用了一次`hwloc_set_group_depth`函数，设置了根层的组层深度为0。此外，由于`hwloc_connect_levels`函数也设置了一些其他层数的层，因此其返回时这些层数的组层深度也会被设置。


```cpp
static void hwloc_set_group_depth(hwloc_topology_t topology)
{
  unsigned groupdepth = 0;
  unsigned i, j;
  for(i=0; i<topology->nb_levels; i++)
    if (topology->levels[i][0]->type == HWLOC_OBJ_GROUP) {
      for (j = 0; j < topology->level_nbobjects[i]; j++)
	topology->levels[i][j]->attr->group.depth = groupdepth;
      groupdepth++;
    }
}

/*
 * Initialize handy pointers in the whole topology.
 * The topology only had first_child and next_sibling pointers.
 * When this funtions return, all parent/children pointers are initialized.
 * The remaining fields (levels, cousins, logical_index, depth, ...) will
 * be setup later in hwloc_connect_levels().
 *
 * Can be called several times, so may have to update the array.
 */
```

This is a C function that performs an operation on a HWLArchetype object, specifically on the OK (Output Keyword) property. It takes an HWLArchetype object, and it is structured as follows:

1. It checks if the OK property is already set. If it is, it implies that the archetype already has a valid OK property.
2. If the OK property is not set, it checks if the archetype has too many children and needs to allocate more memory. If so, it allocates more memory and refills the OK property by setting the parent's OK property to the child.
3. If the OK property is not set and the archetype still has a valid OK property, it creates a children list for the OK property by setting the parent's OK property to the archetype's first child, and then connects it to the parent using the hwloc\_connect\_children() function.
4. If the OK property is not set and the archetype still has a valid OK property, it creates a children list for the IO property by setting the parent's IO property to the archetype's first child, and then connects it to the parent using the hwloc\_connect\_children() function.
5. If the OK property is not set and the archetype still has a valid OK property, it creates a children list for the Misc property by setting the parent's Misc property to the archetype's first child, and then connects it to the parent using the hwloc\_connect\_children() function.

Note that the function also checks for the case where the OK property is not set and the archetype does not have any children, in which case it sets the OK property to the archetype's first child.


```cpp
static void
hwloc_connect_children(hwloc_obj_t parent)
{
  unsigned n, oldn = parent->arity;
  hwloc_obj_t child, prev_child;
  int ok;

  /* Main children list */

  ok = 1;
  prev_child = NULL;
  for (n = 0, child = parent->first_child;
       child;
       n++,   prev_child = child, child = child->next_sibling) {
    child->sibling_rank = n;
    child->prev_sibling = prev_child;
    /* already OK in the array? */
    if (n >= oldn || parent->children[n] != child)
      ok = 0;
    /* recurse */
    hwloc_connect_children(child);
  }
  parent->last_child = prev_child;
  parent->arity = n;
  if (!n) {
    /* no need for an array anymore */
    free(parent->children);
    parent->children = NULL;
    goto memory;
  }
  if (ok)
    /* array is already OK (even if too large) */
    goto memory;

  /* alloc a larger array if needed */
  if (oldn < n) {
    free(parent->children);
    parent->children = malloc(n * sizeof(*parent->children));
  }
  /* refill */
  for (n = 0, child = parent->first_child;
       child;
       n++,   child = child->next_sibling) {
    parent->children[n] = child;
  }



 memory:
  /* Memory children list */

  prev_child = NULL;
  for (n = 0, child = parent->memory_first_child;
       child;
       n++,   prev_child = child, child = child->next_sibling) {
    child->parent = parent;
    child->sibling_rank = n;
    child->prev_sibling = prev_child;
    hwloc_connect_children(child);
  }
  parent->memory_arity = n;

  /* I/O children list */

  prev_child = NULL;
  for (n = 0, child = parent->io_first_child;
       child;
       n++,   prev_child = child, child = child->next_sibling) {
    child->parent = parent;
    child->sibling_rank = n;
    child->prev_sibling = prev_child;
    hwloc_connect_children(child);
  }
  parent->io_arity = n;

  /* Misc children list */

  prev_child = NULL;
  for (n = 0, child = parent->misc_first_child;
       child;
       n++,   prev_child = child, child = child->next_sibling) {
    child->parent = parent;
    child->sibling_rank = n;
    child->prev_sibling = prev_child;
    hwloc_connect_children(child);
  }
  parent->misc_arity = n;
}

```

这段代码定义了一个名为 `find_same_type` 的函数，它接收两个参数，一个是根对象（`hwloc_obj_t` 类型），另一个是要查找的对象（`hwloc_obj_t` 类型）。函数返回一个整数，表示要查找的对象是否存在于根对象中，并且它们具有相同的类型。

函数首先遍历根对象的子对象，对于每个子对象，它使用 `hwloc_type_cmp` 函数比较子对象的类型是否与要查找的对象的类型相同。如果是，函数返回 1，否则继续遍历。在遍历过程中，如果找到要查找的对象，函数将返回 1。


```cpp
/*
 * Check whether there is an object strictly below ROOT that has the same type as OBJ
 */
static int
find_same_type(hwloc_obj_t root, hwloc_obj_t obj)
{
  hwloc_obj_t child;

  for_each_child (child, root) {
    if (hwloc_type_cmp(child, obj) == HWLOC_OBJ_EQUAL)
      return 1;
    if (find_same_type(child, obj))
      return 1;
  }

  return 0;
}

```

这段代码是一个名为 `hwloc_build_level_from_list` 的函数，它接收一个名为 `struct hwloc_special_level_s` 的结构体，并返回一个整数类型的 `hwloc_build_level_t` 变量。

函数的主要目的是根据传入的结构体 `slevel`，建立一个整数级别的 `hwloc_special_level_t` 变量，该变量可以用来指示程序绘图区域所在的层次。

具体来说，函数首先通过 `obj` 指针遍历到结构体数组的第一个元素，并记录下该元素的逻辑索引和 `next_cousin` 指针。然后，函数遍历数组元素，记录下每个元素的逻辑索引、 `next_cousin` 指针以及该元素在数组中的偏移量。最后，函数计算出数组中元素的数量，并返回该数量作为 `hwloc_special_level_t` 变量的值。

如果函数成功执行并且输入的 `slevel` 结构体中包含至少一个有效的元素，它将申请内存以容纳数组元素，并初始化每个元素的逻辑索引、 `next_cousin` 指针以及该元素在数组中的偏移量。这样，就可以在程序中创建一个整数级别的 `hwloc_special_level_t` 变量，该变量可以用来指示程序绘图区域所在的层次。


```cpp
static int
hwloc_build_level_from_list(struct hwloc_special_level_s *slevel)
{
  unsigned i, nb;
  struct hwloc_obj * obj;

  /* count */
  obj = slevel->first;
  i = 0;
  while (obj) {
    i++;
    obj = obj->next_cousin;
  }
  nb = i;

  if (nb) {
    /* allocate and fill level */
    slevel->objs = malloc(nb * sizeof(struct hwloc_obj *));
    if (!slevel->objs)
      return -1;

    obj = slevel->first;
    i = 0;
    while (obj) {
      obj->logical_index = i;
      slevel->objs[i] = obj;
      i++;
      obj = obj->next_cousin;
    }
  }

  slevel->nbobjs = nb;
  return 0;
}

```

这段代码定义了一个名为 `hwloc_append_special_object` 的函数，其作用是将传入的 `obj` 对象加入到某个 `level` 对象的列表中。

函数的具体实现如下：

1. 如果 `level` 对象中已经存在了第一个特殊对象，则执行以下操作：
  1. 将被继承的先前邻居对象的指针存储到 `obj` 对象的 `prev_cousin` 成员中。
  2. 将继承的先前邻居对象后续的邻居对象的指针存储到 `obj` 对象的 `prev_cousin->next_cousin` 成员中。
  3. 将 `obj` 对象存储到 `level` 对象的最后一个特殊对象的位置。

2. 如果 `level` 对象中还没有第一个特殊对象，则执行以下操作：
  1. 将 `obj` 对象存储到 `level` 对象的第一个特殊对象的位置。
  2. 将 `obj` 对象存储到 `level` 对象的最后一个特殊对象的位置。
  3. 将 `obj` 对象存储到 `level` 对象的前一个特殊对象的位置。

另外，如果 `level` 对象中已经存在了多个特殊对象，则需要执行以下操作：

1. 如果 `obj` 对象是第一个特殊对象，则执行以下操作：
  1. 将 `obj` 对象存储到 `level` 对象的第一个特殊对象的位置。

2. 如果 `obj` 对象不是第一个特殊对象，则执行以下操作：
  1. 如果 `obj` 对象是后续的特殊对象，则将其指针存储到 `level` 对象的后续特殊对象指针的列表中。
  2. 如果 `obj` 对象不是后续的特殊对象，则将其指针存储到 `level` 对象的后续特殊对象指针的列表中。


```cpp
static void
hwloc_append_special_object(struct hwloc_special_level_s *level, hwloc_obj_t obj)
{
  if (level->first) {
    obj->prev_cousin = level->last;
    obj->prev_cousin->next_cousin = obj;
    level->last = obj;
  } else {
    obj->prev_cousin = NULL;
    level->first = level->last = obj;
  }
}

/* Append special objects to their lists */
static void
```

This function appears to be a part of a higher-level object management system. It takes in an object header object, and depending on the object type, it either inserts it into a specific list or in the main lists. It also appears to be doing some recursive calls to I/O or Misc children of the object, but it is not clear where or how it is obtaining these children.


```cpp
hwloc_list_special_objects(hwloc_topology_t topology, hwloc_obj_t obj)
{
  hwloc_obj_t child;

  if (obj->type == HWLOC_OBJ_NUMANODE) {
    obj->next_cousin = NULL;
    obj->depth = HWLOC_TYPE_DEPTH_NUMANODE;
    /* Insert the main NUMA node list */
    hwloc_append_special_object(&topology->slevels[HWLOC_SLEVEL_NUMANODE], obj);

    /* Recurse, NUMA nodes only have Misc children */
    for_each_misc_child(child, obj)
      hwloc_list_special_objects(topology, child);

  } else if (obj->type == HWLOC_OBJ_MEMCACHE) {
    obj->next_cousin = NULL;
    obj->depth = HWLOC_TYPE_DEPTH_MEMCACHE;
    /* Insert the main MemCache list */
    hwloc_append_special_object(&topology->slevels[HWLOC_SLEVEL_MEMCACHE], obj);

    /* Recurse, MemCaches have NUMA nodes or Misc children */
    for_each_memory_child(child, obj)
      hwloc_list_special_objects(topology, child);
    for_each_misc_child(child, obj)
      hwloc_list_special_objects(topology, child);

  } else if (obj->type == HWLOC_OBJ_MISC) {
    obj->next_cousin = NULL;
    obj->depth = HWLOC_TYPE_DEPTH_MISC;
    /* Insert the main Misc list */
    hwloc_append_special_object(&topology->slevels[HWLOC_SLEVEL_MISC], obj);
    /* Recurse, Misc only have Misc children */
    for_each_misc_child(child, obj)
      hwloc_list_special_objects(topology, child);

  } else if (hwloc__obj_type_is_io(obj->type)) {
    obj->next_cousin = NULL;

    if (obj->type == HWLOC_OBJ_BRIDGE) {
      obj->depth = HWLOC_TYPE_DEPTH_BRIDGE;
      /* Insert in the main bridge list */
      hwloc_append_special_object(&topology->slevels[HWLOC_SLEVEL_BRIDGE], obj);

    } else if (obj->type == HWLOC_OBJ_PCI_DEVICE) {
      obj->depth = HWLOC_TYPE_DEPTH_PCI_DEVICE;
      /* Insert in the main pcidev list */
      hwloc_append_special_object(&topology->slevels[HWLOC_SLEVEL_PCIDEV], obj);

    } else if (obj->type == HWLOC_OBJ_OS_DEVICE) {
      obj->depth = HWLOC_TYPE_DEPTH_OS_DEVICE;
      /* Insert in the main osdev list */
      hwloc_append_special_object(&topology->slevels[HWLOC_SLEVEL_OSDEV], obj);
    }

    /* Recurse, I/O only have I/O and Misc children */
    for_each_io_child(child, obj)
      hwloc_list_special_objects(topology, child);
    for_each_misc_child(child, obj)
      hwloc_list_special_objects(topology, child);

  } else {
    /* Recurse */
    for_each_child(child, obj)
      hwloc_list_special_objects(topology, child);
    for_each_memory_child(child, obj)
      hwloc_list_special_objects(topology, child);
    for_each_io_child(child, obj)
      hwloc_list_special_objects(topology, child);
    for_each_misc_child(child, obj)
      hwloc_list_special_objects(topology, child);
  }
}

```

这段代码定义了一个名为 `hwloc_connect_special_levels` 的函数，它属于 `hwloc_topology_t` 结构体的成员函数。

该函数的作用是：

1. 释放 `topology` 指向的 `hwloc_slevel` 结构体中的所有 `slevels` 成员，这些成员是内存中的内存块。
2. 将 `topology` 指向的 `hwloc_slevel` 结构体初始化为一个空 `hwloc_slevel` 结构体。
3. 调用 `hwloc_list_special_objects` 函数，将 `topology` 指向的 `hwloc_slevel` 结构体中的 `level0` 成员开始对应的 `slevels` 数组中的所有元素设置为 `NULL`。
4. 遍历 `topology` 指向的 `hwloc_slevel` 结构体中的所有 `slevels` 成员，如果 `hwloc_build_level_from_list` 函数返回负数，说明有错误，返回 0，否则继续遍历。
5. 返回 0，表示所有操作成功完成。


```cpp
/* Build Memory, I/O and Misc levels */
static int
hwloc_connect_special_levels(hwloc_topology_t topology)
{
  unsigned i;

  for(i=0; i<HWLOC_NR_SLEVELS; i++)
    free(topology->slevels[i].objs);
  memset(&topology->slevels, 0, sizeof(topology->slevels));

  hwloc_list_special_objects(topology, topology->levels[0][0]);

  for(i=0; i<HWLOC_NR_SLEVELS; i++) {
    if (hwloc_build_level_from_list(&topology->slevels[i]) < 0)
      return -1;
  }

  return 0;
}

```

This function appears to be a part of the hwloc library, which appears to be a library for managing the memory and nodata on Linux systems.

The function appears to check if the memory layout for the topology object has been successfully allocated, and if not, it attempts to allocate memory for the topology object's level arrays and return an error code.

If the memory layout is successful, the function adds a new level to the topology object's level array, adds the new level's worth of objects to the topology object's level_nbobjects array, and sets the n_taken_objs and n_new_objs fields accordingly.

Finally, the function returns 0 on success, or an error code on failure.


```cpp
/*
 * Do the remaining work that hwloc_connect_children() did not do earlier.
 * Requires object arity and children list to be properly initialized (by hwloc_connect_children()).
 */
static int
hwloc_connect_levels(hwloc_topology_t topology)
{
  unsigned l, i=0;
  hwloc_obj_t *objs, *taken_objs, *new_objs, top_obj, root;
  unsigned n_objs, n_taken_objs, n_new_objs;

  /* reset non-root levels (root was initialized during init and will not change here) */
  for(l=1; l<topology->nb_levels; l++)
    free(topology->levels[l]);
  memset(topology->levels+1, 0, (topology->nb_levels-1)*sizeof(*topology->levels));
  memset(topology->level_nbobjects+1, 0, (topology->nb_levels-1)*sizeof(*topology->level_nbobjects));
  topology->nb_levels = 1;

  /* initialize all non-IO/non-Misc depths to unknown */
  hwloc_reset_normal_type_depths(topology);

  /* initialize root type depth */
  root = topology->levels[0][0];
  root->depth = 0;
  topology->type_depth[root->type] = 0;
  /* root level */
  root->logical_index = 0;
  root->prev_cousin = NULL;
  root->next_cousin = NULL;
  /* root as a child of nothing */
  root->parent = NULL;
  root->sibling_rank = 0;
  root->prev_sibling = NULL;
  root->next_sibling = NULL;

  /* Start with children of the whole system.  */
  n_objs = topology->levels[0][0]->arity;
  objs = malloc(n_objs * sizeof(objs[0]));
  if (!objs) {
    errno = ENOMEM;
    return -1;
  }
  memcpy(objs, topology->levels[0][0]->children, n_objs*sizeof(objs[0]));

  /* Keep building levels while there are objects left in OBJS.  */
  while (n_objs) {
    /* At this point, the objs array contains only objects that may go into levels */

    /* First find which type of object is the topmost.
     * Don't use PU if there are other types since we want to keep PU at the bottom.
     */

    /* Look for the first non-PU object, and use the first PU if we really find nothing else */
    for (i = 0; i < n_objs; i++)
      if (objs[i]->type != HWLOC_OBJ_PU)
        break;
    top_obj = i == n_objs ? objs[0] : objs[i];

    /* See if this is actually the topmost object */
    for (i = 0; i < n_objs; i++) {
      if (hwloc_type_cmp(top_obj, objs[i]) != HWLOC_OBJ_EQUAL) {
	if (find_same_type(objs[i], top_obj)) {
	  /* OBJS[i] is strictly above an object of the same type as TOP_OBJ, so it
	   * is above TOP_OBJ.  */
	  top_obj = objs[i];
	}
      }
    }

    /* Now peek all objects of the same type, build a level with that and
     * replace them with their children.  */

    /* allocate enough to take all current objects and an ending NULL */
    taken_objs = malloc((n_objs+1) * sizeof(taken_objs[0]));
    if (!taken_objs) {
      free(objs);
      errno = ENOMEM;
      return -1;
    }

    /* allocate enough to keep all current objects or their children */
    n_new_objs = 0;
    for (i = 0; i < n_objs; i++) {
      if (objs[i]->arity)
	n_new_objs += objs[i]->arity;
      else
	n_new_objs++;
    }
    new_objs = malloc(n_new_objs * sizeof(new_objs[0]));
    if (!new_objs) {
      free(objs);
      free(taken_objs);
      errno = ENOMEM;
      return -1;
    }

    /* now actually take these objects */
    n_new_objs = 0;
    n_taken_objs = 0;
    for (i = 0; i < n_objs; i++)
      if (hwloc_type_cmp(top_obj, objs[i]) == HWLOC_OBJ_EQUAL) {
	/* Take it, add main children.  */
	taken_objs[n_taken_objs++] = objs[i];
	if (objs[i]->arity)
	  memcpy(&new_objs[n_new_objs], objs[i]->children, objs[i]->arity * sizeof(new_objs[0]));
	n_new_objs += objs[i]->arity;
      } else {
	/* Leave it.  */
	new_objs[n_new_objs++] = objs[i];
      }

    if (!n_new_objs) {
      free(new_objs);
      new_objs = NULL;
    }

    /* Ok, put numbers in the level and link cousins.  */
    for (i = 0; i < n_taken_objs; i++) {
      taken_objs[i]->depth = (int) topology->nb_levels;
      taken_objs[i]->logical_index = i;
      if (i) {
	taken_objs[i]->prev_cousin = taken_objs[i-1];
	taken_objs[i-1]->next_cousin = taken_objs[i];
      }
    }
    taken_objs[0]->prev_cousin = NULL;
    taken_objs[n_taken_objs-1]->next_cousin = NULL;

    /* One more level!  */
    hwloc_debug("--- %s level", hwloc_obj_type_string(top_obj->type));
    hwloc_debug(" has number %u\n\n", topology->nb_levels);

    if (topology->type_depth[top_obj->type] == HWLOC_TYPE_DEPTH_UNKNOWN)
      topology->type_depth[top_obj->type] = (int) topology->nb_levels;
    else
      topology->type_depth[top_obj->type] = HWLOC_TYPE_DEPTH_MULTIPLE; /* mark as unknown */

    taken_objs[n_taken_objs] = NULL;

    if (topology->nb_levels == topology->nb_levels_allocated) {
      /* extend the arrays of levels */
      void *tmplevels, *tmpnbobjs;
      tmplevels = realloc(topology->levels,
			  2 * topology->nb_levels_allocated * sizeof(*topology->levels));
      tmpnbobjs = realloc(topology->level_nbobjects,
			  2 * topology->nb_levels_allocated * sizeof(*topology->level_nbobjects));
      if (!tmplevels || !tmpnbobjs) {
        if (HWLOC_SHOW_CRITICAL_ERRORS())
          fprintf(stderr, "hwloc: failed to realloc level arrays to %u\n", topology->nb_levels_allocated * 2);

	/* if one realloc succeeded, make sure the caller will free the new buffer */
	if (tmplevels)
	  topology->levels = tmplevels;
	if (tmpnbobjs)
	  topology->level_nbobjects = tmpnbobjs;
	/* the realloc that failed left topology->level_foo untouched, will be freed by the caller */

	free(objs);
	free(taken_objs);
	free(new_objs);
	errno = ENOMEM;
	return -1;
      }
      topology->levels = tmplevels;
      topology->level_nbobjects = tmpnbobjs;
      memset(topology->levels + topology->nb_levels_allocated,
	     0, topology->nb_levels_allocated * sizeof(*topology->levels));
      memset(topology->level_nbobjects + topology->nb_levels_allocated,
	     0, topology->nb_levels_allocated * sizeof(*topology->level_nbobjects));
      topology->nb_levels_allocated *= 2;
    }
    /* add the new level */
    topology->level_nbobjects[topology->nb_levels] = n_taken_objs;
    topology->levels[topology->nb_levels] = taken_objs;

    topology->nb_levels++;

    free(objs);

    /* Switch to new_objs */
    objs = new_objs;
    n_objs = n_new_objs;
  }

  /* It's empty now.  */
  free(objs);

  return 0;
}

```

这段代码是一个名为 `hwloc_topology_reconnect` 的函数，它是 `hwloc_topology_loop` 函数的补态函数（并发症函数）。它用于重新连接一个 `hwloc_topology` 结构体，该结构体包含了一个 `hwloc_topology_loop` 函数和一个 `hwloc_topology_selfloop` 函数。

该函数首先检查传入的 `flags` 是否为 `0`，如果是，则表示不需要进行任何操作，返回 `-1`。否则，函数将检查 `topology` 是否被修改过，如果是，则返回 `0`。接下来，函数调用 `hwloc_connect_children` 函数连接第一个级别的 `level`，然后调用 `hwloc_connect_levels` 函数连接所有级别的 `level`。如果以上两个函数中的任意一个返回 `-1`，则整个函数返回 `-1`。最后，函数将 `topology` 结构体的 `modified` 标记设置为 `0`，然后返回 `0`。

总之，该函数的主要作用是确保 `hwloc_topology` 结构体中的所有 `level` 都能够正确地连接起来。


```cpp
int
hwloc_topology_reconnect(struct hwloc_topology *topology, unsigned long flags)
{
  /* WARNING: when updating this function, the replicated code must
   * also be updated inside hwloc_filter_levels_keep_structure()
   */

  if (flags) {
    errno = EINVAL;
    return -1;
  }
  if (!topology->modified)
    return 0;

  hwloc_connect_children(topology->levels[0][0]);

  if (hwloc_connect_levels(topology) < 0)
    return -1;

  if (hwloc_connect_special_levels(topology) < 0)
    return -1;

  topology->modified = 0;

  return 0;
}

```

这段代码定义了一个名为 `hwloc_debug_insert_osdev_sorted` 的函数，它是 `hwloc_obj_t` 类的成员函数。

这个函数的作用是在文件系统中对 `.h` 或 `.栈` 文件（可能还有 `.heap` 和 `.sys` 文件）的 `osdev` 设备队列进行排序。这个排序的依据是 `osdev` 设备的 `name` 属性，而不是 `.h` 文件系统的索引顺序或者 `.stack` 文件的索引顺序。

在排序过程中，函数首先定义了一个指针变量 `pcur`，并将其指向队列的头结点。然后，函数遍历队列中的每个结点，并比较当前结点的 `name` 属性和目标结点的 `name` 属性。如果当前结点的 `name` 属性小于目标结点的 `name` 属性，则将当前结点的 `next_sibling` 指针指向目标结点，并将 `queue` 指向当前结点。

最后，函数将 `queue` 指向了排序后的 `osdev` 设备队列，并返回 `queue`。


```cpp
/* for regression testing, make sure the order of io devices
 * doesn't change with the dentry order in the filesystem
 *
 * Only needed for OSDev for now.
 */
static hwloc_obj_t
hwloc_debug_insert_osdev_sorted(hwloc_obj_t queue, hwloc_obj_t obj)
{
  hwloc_obj_t *pcur = &queue;
  while (*pcur && strcmp((*pcur)->name, obj->name) < 0)
    pcur = &((*pcur)->next_sibling);
  obj->next_sibling = *pcur;
  *pcur = obj;
  return queue;
}

```

这段代码定义了一个名为`hwloc_debug_sort_children`的函数，属于`hwloc_obj_t`类的成员函数。

该函数接收一个`hwloc_obj_t`类型的根对象（根对象），然后对其`io_first_child`成员进行判断，如果根对象中存在一个`hwloc_obj_t`类型的设备对象，则执行以下操作：

1. 从`root`的`io_first_child`成员中获取一个`hwloc_obj_t`类型的空闲对象（如果存在），以及一个指向`hwloc_obj_t`类型的空闲对象的指针`pchild`。
2. 将`pchild`指向的`hwloc_obj_t`类型设置为非操作系统设备类型。
3. 使用`hwloc_debug_insert_osdev_sorted`函数将该设备对象插入到`osdevqueue`队列中，按照设备的初始化时间进行排序。
4. 将`pchild`指向的`hwloc_obj_t`类型设置为`osdevqueue`。

接下来，该函数会递归执行以下操作：

1. 对于每个子对象`child`，首先获取其`type`成员，如果`type`不是`HWLOC_OBJ_OS_DEVICE`，则执行以下操作：

	1. 从`root`的`io_first_child`成员中获取一个指向`hwloc_obj_t`类型的空闲对象（如果存在），并将其指向的`next_sibling`成员设置为新的子对象`child`的下一个兄弟对象。
	2. 如果子对象`child`的`type`是`HWLOC_OBJ_OS_DEVICE`，则执行以下操作：
		1. 从`root`的`io_first_child`成员中获取一个指向`hwloc_obj_t`类型的空闲对象（如果存在），并将其插入到`osdevqueue`队列中，按照设备的初始化时间进行排序。
		2. 将子对象`child`的`next_sibling`成员设置为`NULL`。
		3. 递归执行`hwloc_debug_sort_children`函数，将其子对象也按照类似的方式进行排序。

该函数仅在存在`HWLOC_OBJ_OS_DEVICE`类型的设备对象时起作用，因此在`hwloc_obj_t`类型的根对象中，可能存在一些`HWLOC_OBJ_OS_DEVICE`类型的设备对象，如果没有这样的设备对象，则该函数将不会执行任何操作。


```cpp
static void
hwloc_debug_sort_children(hwloc_obj_t root)
{
  hwloc_obj_t child;

  if (root->io_first_child) {
    hwloc_obj_t osdevqueue, *pchild;

    pchild = &root->io_first_child;
    osdevqueue = NULL;
    while ((child = *pchild) != NULL) {
      if (child->type != HWLOC_OBJ_OS_DEVICE) {
	/* keep non-osdev untouched */
	pchild = &child->next_sibling;
	continue;
      }

      /* dequeue this child */
      *pchild = child->next_sibling;
      child->next_sibling = NULL;

      /* insert in osdev queue in order */
      osdevqueue = hwloc_debug_insert_osdev_sorted(osdevqueue, child);
    }

    /* requeue the now-sorted osdev queue */
    *pchild = osdevqueue;
  }

  /* Recurse */
  for_each_child(child, root)
    hwloc_debug_sort_children(child);
  for_each_memory_child(child, root)
    hwloc_debug_sort_children(child);
  for_each_io_child(child, root)
    hwloc_debug_sort_children(child);
  /* no I/O under Misc */
}

```

这段代码定义了一个名为 `hwloc_alloc_root_sets` 的函数，属于 `hwloc_obj_t` 类的成员函数。

该函数的主要作用是初始化一个 `hwloc_obj_t` 类型的 `root` 对象，确保所有的 `set` 初始化为 NULL，并且在某些情况下，通过调用该函数，所有 `set` 都可以在同一个时间被初始化为某个特定的 `set` 值。

具体来说，该函数首先检查 `root` 对象中的 `cpuset` 是否为 `NULL`，如果是，则执行下面的语句：

```cpp
   root->cpuset = hwloc_bitmap_alloc();
```

如果是，则执行下面的语句：

```cpp
   root->complete_cpuset = hwloc_bitmap_alloc();
```

如果是，则执行下面的语句：

```cpp
   root->nodeset = hwloc_bitmap_alloc();
```

如果是，则执行下面的语句：

```cpp
   root->complete_nodeset = hwloc_bitmap_alloc();
```

在这些情况中，如果所有的 `set` 都为 `NULL`，函数也会正常执行。如果任何一个 `set` 不是 `NULL`，则该函数会为该 `set` 创建一个包含所有 `CPUSET` 的 `hwloc_bitmap` 对象，这样所有 `set` 将共享相同的 `CPUSET` 值。

此外，在一些情况下，该函数可能会被 `hwloc_obj_t` 的父类或某些其他后继类所调用，以便在初始化 `set` 之前执行一些操作。


```cpp
void hwloc_alloc_root_sets(hwloc_obj_t root)
{
  /*
   * All sets are initially NULL.
   *
   * At least one backend should call this function to initialize all sets at once.
   * XML uses it lazily in case only some sets were given in the XML import.
   *
   * Other backends can check root->cpuset != NULL to see if somebody
   * discovered things before them.
   */
  if (!root->cpuset)
     root->cpuset = hwloc_bitmap_alloc();
  if (!root->complete_cpuset)
     root->complete_cpuset = hwloc_bitmap_alloc();
  if (!root->nodeset)
    root->nodeset = hwloc_bitmap_alloc();
  if (!root->complete_nodeset)
    root->complete_nodeset = hwloc_bitmap_alloc();
}

```

这段代码定义了一个名为 `hwloc_discover_by_phase` 的函数，属于 `hwloc_topology` 和 `hwloc_disc_status` 结构体的成员。函数的作用是发现 HWLOC 系统中当前相位的设备，并将结果打印出来。

函数的实现分为以下几个步骤：

1. 获取要遍历的 HWLOC 顶级结构体和相应的Disc Status。
2. 遍历顶级结构体的后继。
3. 如果发现当前相位的设备，则跳出循环。
4. 如果当前相位的设备没有被找到，则继续遍历。
5. 如果当前相位的设备没有被发现，则打印相应的错误信息并继续遍历。
6. 打印最高层级中所有设备的名称。

函数的输入参数为 `topology` 和 `dstatus`，分别为 HWLOC 顶级结构和 Disc Status。函数的输出为空。


```cpp
static void
hwloc_discover_by_phase(struct hwloc_topology *topology,
			struct hwloc_disc_status *dstatus,
			const char *phasename __hwloc_attribute_unused)
{
  struct hwloc_backend *backend;
  hwloc_debug("%s phase discovery...\n", phasename);
  for(backend = topology->backends; backend; backend = backend->next) {
    if (dstatus->phase & dstatus->excluded_phases)
      break;
    if (!(backend->phases & dstatus->phase))
      continue;
    if (!backend->discover)
      continue;
    hwloc_debug("%s phase discovery in component %s...\n", phasename, backend->component->name);
    backend->discover(backend, dstatus);
    hwloc_debug_print_objects(0, topology->levels[0][0]);
  }
}

```

This function appears to be a higher-level version of a topology-specific utility function that performs some additional work to enable or enhance the use of HWLOC.

It does this by first checking if the topology already has a NUMA node, and if it does, it aborts and returns -1. If it does not, it then creates a new NUMA node and sets the topology level to it.

Next, it checks if the topology contains any structures with the HWLOC_TYPE_FILTER_KEEP_STRUCTURE type and removes any such levels.

Then, it accumulates the children memory in the `total_memory` field, and sets up a symmetric subtree for the root level.

Finally, it sets some identification attributes for the topology, such as the `hwlocVersion` and `ProcessName`, and adds a `hwlocVersion` for the topology if it is not set from an XML file.

It also applies the group depth to the topology.

Note that the `hwloc_debug_print_objects` and `hwloc_debug_print_objects` functions are defined earlier in the code, and are used here for logging purposes to print the contents of the topology.

It is important to note that this function calls several other functions and needs to be integrated with them for it to work properly.


```cpp
/* Main discovery loop */
static int
hwloc_discover(struct hwloc_topology *topology,
	       struct hwloc_disc_status *dstatus)
{
  const char *env;

  topology->modified = 0; /* no need to reconnect yet */

  topology->allowed_cpuset = hwloc_bitmap_alloc_full();
  topology->allowed_nodeset = hwloc_bitmap_alloc_full();

  /* discover() callbacks should use hwloc_insert to add objects initialized
   * through hwloc_alloc_setup_object.
   * For node levels, nodeset and memory must be initialized.
   * For cache levels, memory and type/depth must be initialized.
   * For group levels, depth must be initialized.
   */

  /* There must be at least a PU object for each logical processor, at worse
   * produced by hwloc_setup_pu_level()
   */

  /* To be able to just use hwloc__insert_object_by_cpuset to insert the object
   * in the topology according to the cpuset, the cpuset field must be
   * initialized.
   */

  /* A priori, All processors are visible in the topology, and allowed
   * for the application.
   *
   * - If some processors exist but topology information is unknown for them
   *   (and thus the backend couldn't create objects for them), they should be
   *   added to the complete_cpuset field of the lowest object where the object
   *   could reside.
   *
   * - If some processors are not allowed for the application (e.g. for
   *   administration reasons), they should be dropped from the allowed_cpuset
   *   field.
   *
   * The same applies to the node sets complete_nodeset and allowed_cpuset.
   *
   * If such field doesn't exist yet, it can be allocated, and initialized to
   * zero (for complete), or to full (for allowed). The values are
   * automatically propagated to the whole tree after detection.
   */

  if (topology->backend_phases & HWLOC_DISC_PHASE_GLOBAL) {
    /* usually, GLOBAL is alone.
     * but HWLOC_ANNOTATE_GLOBAL_COMPONENTS=1 allows optional ANNOTATE steps.
     */
    struct hwloc_backend *global_backend = topology->backends;
    assert(global_backend);
    assert(global_backend->phases == HWLOC_DISC_PHASE_GLOBAL);

    /*
     * Perform the single-component-based GLOBAL discovery
     */
    hwloc_debug("GLOBAL phase discovery...\n");
    hwloc_debug("GLOBAL phase discovery with component %s...\n", global_backend->component->name);
    dstatus->phase = HWLOC_DISC_PHASE_GLOBAL;
    global_backend->discover(global_backend, dstatus);
    hwloc_debug_print_objects(0, topology->levels[0][0]);
  }
  /* Don't explicitly ignore other phases, in case there's ever
   * a need to bring them back.
   * The component with usually exclude them by default anyway.
   * Except if annotating global components is explicitly requested.
   */

  if (topology->backend_phases & HWLOC_DISC_PHASE_CPU) {
    /*
     * Discover CPUs first
     */
    dstatus->phase = HWLOC_DISC_PHASE_CPU;
    hwloc_discover_by_phase(topology, dstatus, "CPU");
  }

  if (!(topology->backend_phases & (HWLOC_DISC_PHASE_GLOBAL|HWLOC_DISC_PHASE_CPU))) {
    hwloc_debug("No GLOBAL or CPU component phase found\n");
    /* we'll fail below */
  }

  /* One backend should have called hwloc_alloc_root_sets()
   * and set bits during PU and NUMA insert.
   */
  if (!topology->levels[0][0]->cpuset || hwloc_bitmap_iszero(topology->levels[0][0]->cpuset)) {
    hwloc_debug("%s", "No PU added by any CPU or GLOBAL component phase\n");
    errno = EINVAL;
    return -1;
  }

  /*
   * Memory-specific discovery
   */
  if (topology->backend_phases & HWLOC_DISC_PHASE_MEMORY) {
    dstatus->phase = HWLOC_DISC_PHASE_MEMORY;
    hwloc_discover_by_phase(topology, dstatus, "MEMORY");
  }

  if (/* check if getting the sets of locally allowed resources is possible */
      topology->binding_hooks.get_allowed_resources
      && topology->is_thissystem
      /* check whether it has been done already */
      && !(dstatus->flags & HWLOC_DISC_STATUS_FLAG_GOT_ALLOWED_RESOURCES)
      /* check whether it was explicitly requested */
      && ((topology->flags & HWLOC_TOPOLOGY_FLAG_THISSYSTEM_ALLOWED_RESOURCES) != 0
	  || ((env = getenv("HWLOC_THISSYSTEM_ALLOWED_RESOURCES")) != NULL && atoi(env)))) {
    /* OK, get the sets of locally allowed resources */
    topology->binding_hooks.get_allowed_resources(topology);
    dstatus->flags |= HWLOC_DISC_STATUS_FLAG_GOT_ALLOWED_RESOURCES;
  }

  /* If there's no NUMA node, add one with all the memory.
   * root->complete_nodeset wouldn't be empty if any NUMA was ever added:
   * - insert_by_cpuset() adds bits whe PU/NUMA are added.
   * - XML takes care of sanitizing nodesets.
   */
  if (hwloc_bitmap_iszero(topology->levels[0][0]->complete_nodeset)) {
    hwloc_obj_t node;
    hwloc_debug("%s", "\nAdd missing single NUMA node\n");
    node = hwloc_alloc_setup_object(topology, HWLOC_OBJ_NUMANODE, 0);
    node->cpuset = hwloc_bitmap_dup(topology->levels[0][0]->cpuset);
    node->nodeset = hwloc_bitmap_alloc();
    /* other nodesets will be filled below */
    hwloc_bitmap_set(node->nodeset, 0);
    memcpy(&node->attr->numanode, &topology->machine_memory, sizeof(topology->machine_memory));
    memset(&topology->machine_memory, 0, sizeof(topology->machine_memory));
    hwloc__insert_object_by_cpuset(topology, NULL, node, "core:defaultnumanode");
  } else {
    /* if we're sure we found all NUMA nodes without their sizes (x86 backend?),
     * we could split topology->total_memory in all of them.
     */
    free(topology->machine_memory.page_types);
    memset(&topology->machine_memory, 0, sizeof(topology->machine_memory));
  }

  hwloc_debug("%s", "\nFixup root sets\n");
  hwloc_bitmap_and(topology->levels[0][0]->cpuset, topology->levels[0][0]->cpuset, topology->levels[0][0]->complete_cpuset);
  hwloc_bitmap_and(topology->levels[0][0]->nodeset, topology->levels[0][0]->nodeset, topology->levels[0][0]->complete_nodeset);

  hwloc_bitmap_and(topology->allowed_cpuset, topology->allowed_cpuset, topology->levels[0][0]->cpuset);
  hwloc_bitmap_and(topology->allowed_nodeset, topology->allowed_nodeset, topology->levels[0][0]->nodeset);

  hwloc_debug("%s", "\nPropagate sets\n");
  /* cpuset are already there thanks to the _by_cpuset insertion,
   * but nodeset have to be propagated below and above NUMA nodes
   */
  propagate_nodeset(topology->levels[0][0]);
  /* now fixup parent/children sets */
  fixup_sets(topology->levels[0][0]);

  hwloc_debug_print_objects(0, topology->levels[0][0]);

  if (!(topology->flags & HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED)) {
    hwloc_debug("%s", "\nRemoving unauthorized sets from all sets\n");
    remove_unused_sets(topology, topology->levels[0][0]);
    hwloc_debug_print_objects(0, topology->levels[0][0]);
  }

  /* see if we should ignore the root now that we know how many children it has */
  if (!hwloc_filter_check_keep_object(topology, topology->levels[0][0])
      && topology->levels[0][0]->first_child && !topology->levels[0][0]->first_child->next_sibling) {
    hwloc_obj_t oldroot = topology->levels[0][0];
    hwloc_obj_t newroot = oldroot->first_child;
    /* switch to the new root */
    newroot->parent = NULL;
    topology->levels[0][0] = newroot;
    /* move oldroot memory/io/misc children before newroot children */
    if (oldroot->memory_first_child)
      prepend_siblings_list(&newroot->memory_first_child, oldroot->memory_first_child, newroot);
    if (oldroot->io_first_child)
      prepend_siblings_list(&newroot->io_first_child, oldroot->io_first_child, newroot);
    if (oldroot->misc_first_child)
      prepend_siblings_list(&newroot->misc_first_child, oldroot->misc_first_child, newroot);
    /* destroy oldroot and use the new one */
    hwloc_free_unlinked_object(oldroot);
  }

  /*
   * All object cpusets and nodesets are properly set now.
   */

  /* Now connect handy pointers to make remaining discovery easier. */
  hwloc_debug("%s", "\nOk, finished tweaking, now connect\n");
  if (hwloc_topology_reconnect(topology, 0) < 0)
    return -1;
  hwloc_debug_print_objects(0, topology->levels[0][0]);

  /*
   * Additional discovery
   */
  hwloc_pci_discovery_prepare(topology);

  if (topology->backend_phases & HWLOC_DISC_PHASE_PCI) {
    dstatus->phase = HWLOC_DISC_PHASE_PCI;
    hwloc_discover_by_phase(topology, dstatus, "PCI");
  }
  if (topology->backend_phases & HWLOC_DISC_PHASE_IO) {
    dstatus->phase = HWLOC_DISC_PHASE_IO;
    hwloc_discover_by_phase(topology, dstatus, "IO");
  }
  if (topology->backend_phases & HWLOC_DISC_PHASE_MISC) {
    dstatus->phase = HWLOC_DISC_PHASE_MISC;
    hwloc_discover_by_phase(topology, dstatus, "MISC");
  }
  if (topology->backend_phases & HWLOC_DISC_PHASE_ANNOTATE) {
    dstatus->phase = HWLOC_DISC_PHASE_ANNOTATE;
    hwloc_discover_by_phase(topology, dstatus, "ANNOTATE");
  }

  hwloc_pci_discovery_exit(topology); /* pci needed up to annotate */

  if (getenv("HWLOC_DEBUG_SORT_CHILDREN"))
    hwloc_debug_sort_children(topology->levels[0][0]);

  /* Remove some stuff */

  hwloc_debug("%s", "\nRemoving bridge objects if needed\n");
  hwloc_filter_bridges(topology, topology->levels[0][0]);
  hwloc_debug_print_objects(0, topology->levels[0][0]);

  hwloc_debug("%s", "\nRemoving empty objects\n");
  remove_empty(topology, &topology->levels[0][0]);
  if (!topology->levels[0][0]) {
    if (HWLOC_SHOW_CRITICAL_ERRORS())
      fprintf(stderr, "hwloc: Topology became empty, aborting!\n");
    return -1;
  }
  if (hwloc_bitmap_iszero(topology->levels[0][0]->cpuset)) {
    if (HWLOC_SHOW_CRITICAL_ERRORS())
      fprintf(stderr, "hwloc: Topology does not contain any PU, aborting!\n");
    return -1;
  }
  if (hwloc_bitmap_iszero(topology->levels[0][0]->nodeset)) {
    if (HWLOC_SHOW_CRITICAL_ERRORS())
      fprintf(stderr, "hwloc: Topology does not contain any NUMA node, aborting!\n");
    return -1;
  }
  hwloc_debug_print_objects(0, topology->levels[0][0]);

  hwloc_debug("%s", "\nRemoving levels with HWLOC_TYPE_FILTER_KEEP_STRUCTURE\n");
  if (hwloc_filter_levels_keep_structure(topology) < 0)
    return -1;
  /* takes care of reconnecting children/levels internally,
   * because it needs normal levels.
   * and it's often needed below because of Groups inserted for I/Os anyway */
  hwloc_debug_print_objects(0, topology->levels[0][0]);

  /* accumulate children memory in total_memory fields (only once parent is set) */
  hwloc_debug("%s", "\nPropagate total memory up\n");
  propagate_total_memory(topology->levels[0][0]);

  /* setup the symmetric_subtree attribute */
  hwloc_propagate_symmetric_subtree(topology, topology->levels[0][0]);

  /* apply group depths */
  hwloc_set_group_depth(topology);

  /* add some identification attributes if not loading from XML */
  if (topology->backends
      && strcmp(topology->backends->component->name, "xml")
      && !getenv("HWLOC_DONT_ADD_VERSION_INFO")) {
    char *value;
    /* add a hwlocVersion */
    hwloc_obj_add_info(topology->levels[0][0], "hwlocVersion", HWLOC_VERSION);
    /* add a ProcessName */
    value = hwloc_progname(topology);
    if (value) {
      hwloc_obj_add_info(topology->levels[0][0], "ProcessName", value);
      free(value);
    }
  }

  return 0;
}

```

This code appears to be setting up a hardware location object (HLOC) for a machine, with a specific depth and a硬盘中包含的设备类型。HLOC是Linux系统中的一个硬件设备驱动程序，用于管理计算机硬件设备的挂载点和依赖关系。

首先，根据topology结构中的sane values to type_depth，可以确定类型深度（type_depth）的值。然后，根据topology结构中的sane values to type_depth，可以确定已知的类型和深度，并将其初始化为HWLOC_SLEVEL_FROM_DEPTH(HWLOC_TYPE_DEPTH_PCI_DEVICE)和HWLOC_SLEVEL_FROM_DEPTH(HWLOC_TYPE_DEPTH_OS_DEVICE)。

接下来，根据topology结构中的sane values to type_depth，创建一个适当的机器对象（root_obj），并将其设置为该层的根对象（levels[0][0]）。最后，根据topology结构中的sane values to type_depth，将其他层的对象设置为它们的合适类型和深度。


```cpp
/* To be called before discovery is actually launched,
 * Resets everything in case a previous load initialized some stuff.
 */
void
hwloc_topology_setup_defaults(struct hwloc_topology *topology)
{
  struct hwloc_obj *root_obj;

  /* reset support */
  memset(&topology->binding_hooks, 0, sizeof(topology->binding_hooks));
  memset(topology->support.discovery, 0, sizeof(*topology->support.discovery));
  memset(topology->support.cpubind, 0, sizeof(*topology->support.cpubind));
  memset(topology->support.membind, 0, sizeof(*topology->support.membind));
  memset(topology->support.misc, 0, sizeof(*topology->support.misc));

  /* Only the System object on top by default */
  topology->next_gp_index = 1; /* keep 0 as an invalid value */
  topology->nb_levels = 1; /* there's at least SYSTEM */
  topology->levels[0] = hwloc_tma_malloc (topology->tma, sizeof (hwloc_obj_t));
  topology->level_nbobjects[0] = 1;

  /* Machine-wide memory */
  topology->machine_memory.local_memory = 0;
  topology->machine_memory.page_types_len = 0;
  topology->machine_memory.page_types = NULL;

  /* Allowed stuff */
  topology->allowed_cpuset = NULL;
  topology->allowed_nodeset = NULL;

  /* NULLify other special levels */
  memset(&topology->slevels, 0, sizeof(topology->slevels));
  /* assert the indexes of special levels */
  HWLOC_BUILD_ASSERT(HWLOC_SLEVEL_NUMANODE == HWLOC_SLEVEL_FROM_DEPTH(HWLOC_TYPE_DEPTH_NUMANODE));
  HWLOC_BUILD_ASSERT(HWLOC_SLEVEL_MISC == HWLOC_SLEVEL_FROM_DEPTH(HWLOC_TYPE_DEPTH_MISC));
  HWLOC_BUILD_ASSERT(HWLOC_SLEVEL_BRIDGE == HWLOC_SLEVEL_FROM_DEPTH(HWLOC_TYPE_DEPTH_BRIDGE));
  HWLOC_BUILD_ASSERT(HWLOC_SLEVEL_PCIDEV == HWLOC_SLEVEL_FROM_DEPTH(HWLOC_TYPE_DEPTH_PCI_DEVICE));
  HWLOC_BUILD_ASSERT(HWLOC_SLEVEL_OSDEV == HWLOC_SLEVEL_FROM_DEPTH(HWLOC_TYPE_DEPTH_OS_DEVICE));
  HWLOC_BUILD_ASSERT(HWLOC_SLEVEL_MEMCACHE == HWLOC_SLEVEL_FROM_DEPTH(HWLOC_TYPE_DEPTH_MEMCACHE));

  /* sane values to type_depth */
  hwloc_reset_normal_type_depths(topology);
  topology->type_depth[HWLOC_OBJ_NUMANODE] = HWLOC_TYPE_DEPTH_NUMANODE;
  topology->type_depth[HWLOC_OBJ_MISC] = HWLOC_TYPE_DEPTH_MISC;
  topology->type_depth[HWLOC_OBJ_BRIDGE] = HWLOC_TYPE_DEPTH_BRIDGE;
  topology->type_depth[HWLOC_OBJ_PCI_DEVICE] = HWLOC_TYPE_DEPTH_PCI_DEVICE;
  topology->type_depth[HWLOC_OBJ_OS_DEVICE] = HWLOC_TYPE_DEPTH_OS_DEVICE;
  topology->type_depth[HWLOC_OBJ_MEMCACHE] = HWLOC_TYPE_DEPTH_MEMCACHE;

  /* Create the actual machine object, but don't touch its attributes yet
   * since the OS backend may still change the object into something else
   * (for instance System)
   */
  root_obj = hwloc_alloc_setup_object(topology, HWLOC_OBJ_MACHINE, 0);
  topology->levels[0][0] = root_obj;
}

```

This is a C function that initializes a topology object. The topology object is used to
maintain information about the system's hardware and is used to
filter and manipulate topology information.

The function takes several arguments:

- `topology`: The root topology object. This object should already
be initialized.
- `userdata`: The user-provided data that will be stored in the
topology object. This can be used to pass data between topology functions.
- `topology_abi`: The topology ABI (Application Binary Interface). This
interface should be passed to `hwloc_topology_setup_defaults`.
- `adopted_shmem_addr`: The memory address of the shared memory that has
been adopted by the topology. This can be used to store data
between topology functions.
- `adopted_shmem_length`: The size of the shared memory that has been
adopted by the topology.
- `support`: The `topology_support` structure that should
be passed to `hwloc_topology_setup_defaults`. This structure
should already have a `discovery` field that is not NULL and a
`cpubind` field that is not NULL.
- `membind`: The `topology_membind` field that should
be passed to `hwloc_topology_setup_defaults`. This field should
already have a `misc` field that is not NULL.
- `nb_levels_allocated`: The number of levels that have been allocated
in the topology. This should be passed to `hwloc_levels_get_default`.
- `levels`: A pointer to an array of `level_nbobjects` that
should be passed to `hwloc_level_nbobjects_get_default`.
- `level_nbobjects`: A pointer to an array of `hwloc_topology_level_nbobjects`
that should be passed to `hwloc_topology_level_nbobjects_get_default`.

The function initializes the topology object and sets it to the first object in the hierarchy. It also sets the topology ABI and creates the shared memory that has been adopted by the topology.

The function returns nothing but the pointer to the topology object.


```cpp
static void hwloc__topology_filter_init(struct hwloc_topology *topology);

/* This function may use a tma, it cannot free() or realloc() */
static int
hwloc__topology_init (struct hwloc_topology **topologyp,
		      unsigned nblevels,
		      struct hwloc_tma *tma)
{
  struct hwloc_topology *topology;

  topology = hwloc_tma_malloc (tma, sizeof (struct hwloc_topology));
  if(!topology)
    return -1;

  topology->tma = tma;

  hwloc_components_init(); /* uses malloc without tma, but won't need it since dup() caller already took a reference */
  hwloc_topology_components_init(topology);
  hwloc_pci_discovery_init(topology); /* make sure both dup() and load() get sane variables */

  /* Setup topology context */
  topology->is_loaded = 0;
  topology->flags = 0;
  topology->is_thissystem = 1;
  topology->pid = 0;
  topology->userdata = NULL;
  topology->topology_abi = HWLOC_TOPOLOGY_ABI;
  topology->adopted_shmem_addr = NULL;
  topology->adopted_shmem_length = 0;

  topology->support.discovery = hwloc_tma_malloc(tma, sizeof(*topology->support.discovery));
  topology->support.cpubind = hwloc_tma_malloc(tma, sizeof(*topology->support.cpubind));
  topology->support.membind = hwloc_tma_malloc(tma, sizeof(*topology->support.membind));
  topology->support.misc = hwloc_tma_malloc(tma, sizeof(*topology->support.misc));

  topology->nb_levels_allocated = nblevels; /* enough for default 10 levels = Mach+Pack+Die+NUMA+L3+L2+L1d+L1i+Co+PU */
  topology->levels = hwloc_tma_calloc(tma, topology->nb_levels_allocated * sizeof(*topology->levels));
  topology->level_nbobjects = hwloc_tma_calloc(tma, topology->nb_levels_allocated * sizeof(*topology->level_nbobjects));

  hwloc__topology_filter_init(topology);

  hwloc_internal_distances_init(topology);
  hwloc_internal_memattrs_init(topology);
  hwloc_internal_cpukinds_init(topology);

  topology->userdata_export_cb = NULL;
  topology->userdata_import_cb = NULL;
  topology->userdata_not_decoded = 0;

  /* Make the topology look like something coherent but empty */
  hwloc_topology_setup_defaults(topology);

  *topologyp = topology;
  return 0;
}

```

这段代码定义了一个名为 `hwloc_topology_init` 的函数和名为 `hwloc_topology_set_pid` 的函数。它们属于 `hwloc_topology.h` 文件。

`hwloc_topology_init` 函数的目的是初始化硬件布局 topology，设置时钟树顶端点为 16，表示使用 16 个时钟树级别。函数的返回值表示初始化是否成功。

`hwloc_topology_set_pid` 函数的目的是设置布局的时钟树根节点的 ID，设置时钟树根节点为指定的 ID。函数的第一个参数是一个 `hwloc_topology` 类型的布局对象，第二个参数是一个 `hwloc_pid_t` 类型的 ID，表示要设置的时钟树根节点。函数的返回值表示设置是否成功，若成功，则返回 0，否则返回一个负值。


```cpp
int
hwloc_topology_init (struct hwloc_topology **topologyp)
{
  return hwloc__topology_init(topologyp,
			      16, /* 16 is enough for default 10 levels = Mach+Pack+Die+NUMA+L3+L2+L1d+L1i+Co+PU */
			      NULL); /* no TMA for normal topologies, too many allocations to fix */
}

int
hwloc_topology_set_pid(struct hwloc_topology *topology __hwloc_attribute_unused,
                       hwloc_pid_t pid __hwloc_attribute_unused)
{
  if (topology->is_loaded) {
    errno = EBUSY;
    return -1;
  }

  /* this does *not* change the backend */
```

这段代码是一个C语言函数，名为`hwloc_topology_set_synthetic`。它用于设置硬件定位（hwloc）中的合成设置。

首先，代码检查是否在Linux系统的顶端（topology）中设置过该设置。如果是，则将`topology->pid`的值设置为当前正在运行的PID（process ID），并将0返回。否则，定义一个错误代码（errno）和-1的返回值。

接着，函数`hwloc_topology_set_synthetic`，用于设置合成设置。它首先检查`topology`是否已加载（is_loaded）。如果是，则执行接下来的错误代码（errno）。否则，调用`hwloc_disc_component_force_enable`函数，传递给`topology`的引用、0作为API，设置描述信息（description）为设置的合成设置，并将返回值存储在`ret`中。


```cpp
#ifdef HWLOC_LINUX_SYS
  topology->pid = pid;
  return 0;
#else /* HWLOC_LINUX_SYS */
  errno = ENOSYS;
  return -1;
#endif /* HWLOC_LINUX_SYS */
}

int
hwloc_topology_set_synthetic(struct hwloc_topology *topology, const char *description)
{
  if (topology->is_loaded) {
    errno = EBUSY;
    return -1;
  }

  return hwloc_disc_component_force_enable(topology,
					   0 /* api */,
					   "synthetic",
					   description, NULL, NULL);
}

```

这段代码是一个用于设置硬件抽象层（HAL）中顶级拓扑结构（Topology）的XML文件的函数。这个函数的作用是确保在从给定的XML路径装载顶级拓扑结构之前，已启用了相应的API。

具体来说，代码首先检查给定的topology结构是否已经加载过。如果是，函数将返回-1，因为已经有一个加载的topology结构，不需要额外的操作。

否则，函数调用hwloc_disc_component_force_enable函数，设置api为0，并指定xml路径。这样，函数将启用API以从xml文件中读取顶级拓扑结构。

最后，函数返回0，表示成功设置xml文件并启用相应的API。


```cpp
int
hwloc_topology_set_xml(struct hwloc_topology *topology,
		       const char *xmlpath)
{
  if (topology->is_loaded) {
    errno = EBUSY;
    return -1;
  }

  return hwloc_disc_component_force_enable(topology,
					   0 /* api */,
					   "xml",
					   xmlpath, NULL, NULL);
}

```

这段代码定义了一个名为 `hwloc_topology_set_xmlbuffer` 的函数，其作用是设置一个 XML 文件的缓冲区，以供 `hwloc_disc_component_force_enable` 函数使用。

具体来说，这段代码包含以下几行：

1. 定义了一个名为 `topology` 的结构体，其中包含一个指向 XML 文件的指针，一个 XML 文件的大小，以及一个布尔值，表示是否已经加载过 XML 文件。

2. 定义了一个名为 `xmlbuffer` 的字符指针，用于存储 XML 文件的内容。

3. 定义了一个名为 `size` 的整数，用于表示 XML 文件的大小。

4. 接下来是函数体，该体实现了 `hwloc_topology_set_xmlbuffer` 的功能：

a. 首先检查传入的 `topology` 是否已经被加载过 XML 文件，如果是，则返回 EBUSY，否则继续执行。

b. 调用 `hwloc_disc_component_force_enable` 函数，传递三个参数：`topology`、0（表示使用 API 而不是用户空间模式）、"xml" 和 `xmlbuffer`，最后一个参数是一个指向 `size` 所表示字节数目的指针。

c. 调用 `hwloc_disc_component_force_enable` 函数之后，如果 `topology->is_loaded` 为假，那么函数将返回 -1，否则成功设置 XML 文件的缓冲区。


```cpp
int
hwloc_topology_set_xmlbuffer(struct hwloc_topology *topology,
                             const char *xmlbuffer,
                             int size)
{
  if (topology->is_loaded) {
    errno = EBUSY;
    return -1;
  }

  return hwloc_disc_component_force_enable(topology,
					   0 /* api */,
					   "xml", NULL,
					   xmlbuffer, (void*) (uintptr_t) size);
}

```

This function appears to check for certain topology flags and handle if they are present or not. If the flags are not present or their value is incorrect, it returns -1.

If the flags are present and do not include disallowed or THISSYSTEM bindings, it seems to be fine and returns 0.

If the flags include disallowed or THISSYSTEM bindings, it returns -1.

If the flags include RESTRICT_TO_CPUBINDING or RESTRICT_TO_MEMBINDING, it returns -1 if the binding is required, but the function does not seem to handle the error in any way.

Overall, it appears to be a simple function that checks for certain topology flags and handles if they are present or not.


```cpp
int
hwloc_topology_set_flags (struct hwloc_topology *topology, unsigned long flags)
{
  if (topology->is_loaded) {
    /* actually harmless */
    errno = EBUSY;
    return -1;
  }

  if (flags & ~(HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED
                |HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM
                |HWLOC_TOPOLOGY_FLAG_THISSYSTEM_ALLOWED_RESOURCES
                |HWLOC_TOPOLOGY_FLAG_IMPORT_SUPPORT
                |HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_CPUBINDING
                |HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_MEMBINDING
                |HWLOC_TOPOLOGY_FLAG_DONT_CHANGE_BINDING
                |HWLOC_TOPOLOGY_FLAG_NO_DISTANCES
                |HWLOC_TOPOLOGY_FLAG_NO_MEMATTRS
                |HWLOC_TOPOLOGY_FLAG_NO_CPUKINDS)) {
    errno = EINVAL;
    return -1;
  }

  if ((flags & (HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_CPUBINDING|HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM)) == HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_CPUBINDING) {
    /* RESTRICT_TO_CPUBINDING requires THISSYSTEM for binding */
    errno = EINVAL;
    return -1;
  }
  if ((flags & (HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_MEMBINDING|HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM)) == HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_MEMBINDING) {
    /* RESTRICT_TO_MEMBINDING requires THISSYSTEM for binding */
    errno = EINVAL;
    return -1;
  }

  topology->flags = flags;
  return 0;
}

```



The `hwloc_topology_get_flags()` function retrieves the flags of a `hwloc_topology` structure. The structure contains various flags indicating the location and characteristics of the hardware resource.

The `hwloc_obj_type_t` enumeration includes the possible object types for the `hwloc_topology` structure. The `topology->flags` field can be accessed using the `enum_find_index()` function to determine the index of the corresponding flag in the enumeration.

The `hwloc__topology_filter_init()` function is used to filter the topology structure based on certain criteria. It takes a single parameter of the `struct hwloc_topology` type and initializes the filter for each type of the object. The filter flags are stored in the `topology->type_filter` array, and the function handles various types of filters, such as ones for L1, L2, and L3 cache, Memcached, and PCI devices.

The initial call to `hwloc__topology_filter_init()` is made when the `hwloc_topology_create()` function is called. It is used to reset the filter flags for each type of the object to its default state (which is `HWLOC_TYPE_FILTER_KEEP_ALL` for all types).


```cpp
unsigned long
hwloc_topology_get_flags (struct hwloc_topology *topology)
{
  return topology->flags;
}

static void
hwloc__topology_filter_init(struct hwloc_topology *topology)
{
  hwloc_obj_type_t type;
  /* Only ignore useless cruft by default */
  for(type = HWLOC_OBJ_TYPE_MIN; type < HWLOC_OBJ_TYPE_MAX; type++)
    topology->type_filter[type] = HWLOC_TYPE_FILTER_KEEP_ALL;
  topology->type_filter[HWLOC_OBJ_L1ICACHE] = HWLOC_TYPE_FILTER_KEEP_NONE;
  topology->type_filter[HWLOC_OBJ_L2ICACHE] = HWLOC_TYPE_FILTER_KEEP_NONE;
  topology->type_filter[HWLOC_OBJ_L3ICACHE] = HWLOC_TYPE_FILTER_KEEP_NONE;
  topology->type_filter[HWLOC_OBJ_MEMCACHE] = HWLOC_TYPE_FILTER_KEEP_NONE;
  topology->type_filter[HWLOC_OBJ_GROUP] = HWLOC_TYPE_FILTER_KEEP_STRUCTURE;
  topology->type_filter[HWLOC_OBJ_MISC] = HWLOC_TYPE_FILTER_KEEP_NONE;
  topology->type_filter[HWLOC_OBJ_BRIDGE] = HWLOC_TYPE_FILTER_KEEP_NONE;
  topology->type_filter[HWLOC_OBJ_PCI_DEVICE] = HWLOC_TYPE_FILTER_KEEP_NONE;
  topology->type_filter[HWLOC_OBJ_OS_DEVICE] = HWLOC_TYPE_FILTER_KEEP_NONE;
}

```

该函数名为 `hwloc__topology_set_type_filter`，属于 `hwloc_topology_` 结构的成员函数。

它的作用是设置给定的 `topology` 结构中的 `type` 成员的 `filter` 值。具体的实现过程如下：

1. 首先检查 `topology` 结构中 `type` 成员的值，如果为 `HWLOC_OBJ_PU`、`HWLOC_OBJ_NUMANODE` 或 `HWLOC_OBJ_MACHINE`，则需要检查 `filter` 是否为 `HWLOC_TYPE_FILTER_KEEP_ALL`。如果是，函数返回 `0`，否则继续执行下面的判断。
2. 如果 `topology` 结构中的 `type` 成员不是 `HWLOC_OBJ_PU`、`HWLOC_OBJ_NUMANODE` 或 `HWLOC_OBJ_MACHINE`，则需要判断给定的 `filter` 是否为 `HWLOC_TYPE_FILTER_KEEP_STRUCTURE`。如果是，函数返回 `EINVAL`，否则继续执行下面的判断。
3. 如果 `topology` 结构中的 `type` 成员是 `HWLOC_OBJ_GROUP`，则需要判断给定的 `filter` 是否为 `HWLOC_TYPE_FILTER_KEEP_ALL`。如果是，函数返回 `EINVAL`，否则继续执行下面的判断。
4. 如果 `topology` 结构中的 `type` 成员不是 `HWLOC_OBJ_GROUP` 且给定的 `filter` 不是 `HWLOC_TYPE_FILTER_KEEP_IMPORTANT`，则函数将 `filter` 设置为 `HWLOC_TYPE_FILTER_KEEP_ALL`，并返回 `0`。否则，函数返回 `-1`，表明出错。

该函数在 `hwloc_topology_` 结构中具有重要的地位，因为它负责管理 `topology` 结构中的 `type` 成员的 `filter` 值。通过该函数，可以对 `topology` 结构进行灵活的自定义，以适应不同的 `hwloc_topology_` 对象的设置需要。


```cpp
static int
hwloc__topology_set_type_filter(struct hwloc_topology *topology, hwloc_obj_type_t type, enum hwloc_type_filter_e filter)
{
  if (type == HWLOC_OBJ_PU || type == HWLOC_OBJ_NUMANODE || type == HWLOC_OBJ_MACHINE) {
    if (filter != HWLOC_TYPE_FILTER_KEEP_ALL) {
      /* we need the Machine, PU and NUMA levels */
      errno = EINVAL;
      return -1;
    }
  } else if (hwloc__obj_type_is_special(type)) {
    if (filter == HWLOC_TYPE_FILTER_KEEP_STRUCTURE) {
      /* I/O and Misc are outside of the main topology structure, makes no sense. */
      errno = EINVAL;
      return -1;
    }
  } else if (type == HWLOC_OBJ_GROUP) {
    if (filter == HWLOC_TYPE_FILTER_KEEP_ALL) {
      /* Groups are always ignored, at least keep_structure */
      errno = EINVAL;
      return -1;
    }
  }

  /* "important" just means "all" for non-I/O non-Misc */
  if (!hwloc__obj_type_is_special(type) && filter == HWLOC_TYPE_FILTER_KEEP_IMPORTANT)
    filter = HWLOC_TYPE_FILTER_KEEP_ALL;

  topology->type_filter[type] = filter;
  return 0;
}

```

这段代码是一个名为 `hwloc_topology_set_type_filter` 的函数，属于 `hwloc_topology` 系列的库。它的作用是设置给定的 `topology` 对象中的 `type` 成员为指定的 `enum hwloc_type_filter_e` 值，然后检查是否可以设置，成功则返回 0，失败则返回一个负数。

函数的参数包括：

* `topology`：指向 `hwloc_topology` 结构的指针，该结构可能包含其他辅助函数，如 `hwloc_util_topology_loop` 等。
* `type`：`enum hwloc_type_filter_e` 类型的参数，用于指定要设置的类型。
* `filter`：同上，用于指定要应用的过滤规则。

该函数的实现基于 `hwloc_topology` 和 `enum` 系列的提供。通过这两个库可以方便地设置和检查 `topology` 对象的属性和 `enum` 中定义的值。


```cpp
int
hwloc_topology_set_type_filter(struct hwloc_topology *topology, hwloc_obj_type_t type, enum hwloc_type_filter_e filter)
{
  HWLOC_BUILD_ASSERT(HWLOC_OBJ_TYPE_MIN == 0);
  if ((unsigned) type >= HWLOC_OBJ_TYPE_MAX) {
    errno = EINVAL;
    return -1;
  }
  if (topology->is_loaded) {
    errno = EBUSY;
    return -1;
  }
  return hwloc__topology_set_type_filter(topology, type, filter);
}

```

这两段代码是用于设置一个名为 "hwloc\_topology" 的哈威格式（HWLC）中的所有类型（即布局）的类型筛选（filter）。通过这两段代码，可以允许或阻止在 "hwloc\_topology" 中使用的不同类型的布局。

在第一段代码中，`hwloc_topology_set_all_types_filter`函数接收一个 `struct hwloc_topology` 类型的上下文和一个 `enum hwloc_type_filter_e` 类型的过滤器（使用 `filter` 参数指定）并返回 0。如果上下文未加载，函数返回 -1，因为无法设置任何内容。

在第二段代码中，`hwloc_topology_set_cache_types_filter`函数接收一个 `hwloc_topology_t` 类型的上下文和一个 `enum hwloc_type_filter_e` 类型的过滤器（使用 `filter` 参数指定），并返回 0。在 L1 缓存层次结构和 L3 缓存层次结构中设置过滤器。


```cpp
int
hwloc_topology_set_all_types_filter(struct hwloc_topology *topology, enum hwloc_type_filter_e filter)
{
  hwloc_obj_type_t type;
  if (topology->is_loaded) {
    errno = EBUSY;
    return -1;
  }
  for(type = HWLOC_OBJ_TYPE_MIN; type < HWLOC_OBJ_TYPE_MAX; type++)
    hwloc__topology_set_type_filter(topology, type, filter);
  return 0;
}

int
hwloc_topology_set_cache_types_filter(hwloc_topology_t topology, enum hwloc_type_filter_e filter)
{
  unsigned i;
  for(i=HWLOC_OBJ_L1CACHE; i<HWLOC_OBJ_L3ICACHE; i++)
    hwloc_topology_set_type_filter(topology, (hwloc_obj_type_t) i, filter);
  return 0;
}

```

这两段代码是用来设置硬件布局（hwloc_topology_t）中的输入/输出（io）类型过滤器（enum hwloc_type_filter_e）的。

在函数hwloc_topology_set_icache_types_filter中，首先定义了一个枚举类型hwloc_type_filter_e，用来表示输入/输出（io）类型过滤器的类型。然后使用一个for循环，从HWLOC_OBJ_L1ICACHE开始，遍历所有的输入/输出类型，将类型为i的hwloc_obj_type_t对象传递给函数指针filter，实现输入/输出类型过滤器的设置。最后，函数返回0，表示设置成功。

在函数hwloc_topology_set_io_types_filter中，同样定义了一个枚举类型hwloc_type_filter_e，用来表示输入/输出（io）类型过滤器的类型。然后使用一个for循环，从HWLOC_OBJ_BRIDGE开始，遍历所有的输入/输出类型，将类型为HWLOC_OBJ_BRIDGE的hwloc_obj_type_t对象传递给函数指针filter，实现输入/输出类型过滤器的设置。最后，函数返回0，表示设置成功。


```cpp
int
hwloc_topology_set_icache_types_filter(hwloc_topology_t topology, enum hwloc_type_filter_e filter)
{
  unsigned i;
  for(i=HWLOC_OBJ_L1ICACHE; i<HWLOC_OBJ_L3ICACHE; i++)
    hwloc_topology_set_type_filter(topology, (hwloc_obj_type_t) i, filter);
  return 0;
}

int
hwloc_topology_set_io_types_filter(hwloc_topology_t topology, enum hwloc_type_filter_e filter)
{
  hwloc_topology_set_type_filter(topology, HWLOC_OBJ_BRIDGE, filter);
  hwloc_topology_set_type_filter(topology, HWLOC_OBJ_PCI_DEVICE, filter);
  hwloc_topology_set_type_filter(topology, HWLOC_OBJ_OS_DEVICE, filter);
  return 0;
}

```

这两段代码是用于在hwloc_topology结构体中进行类型枚举和过滤。`hwloc_topology_get_type_filter`函数接收一个hwloc_topology结构体和一个enum hwloc_type_filter_e类型的指针，用于获取topology结构体中具有指定type类型的元素，并将其存储在filterp指向的内存位置。如果type参数不匹配，函数将返回-1并输出错误码。`hwloc_topology_clear`函数用于释放topology结构体中的内存，并将其free。在函数中，对于每个topology层，free掉了该层所有对象和子层。在topology层的free函数中，释放了该层对象的内存，并使用topology->allowed_cpuset和topology->allowed_nodeset指针，将允许的CPU设置和允许的节点设置为free内存。


```cpp
int
hwloc_topology_get_type_filter(struct hwloc_topology *topology, hwloc_obj_type_t type, enum hwloc_type_filter_e *filterp)
{
  HWLOC_BUILD_ASSERT(HWLOC_OBJ_TYPE_MIN == 0);
  if ((unsigned) type >= HWLOC_OBJ_TYPE_MAX) {
    errno = EINVAL;
    return -1;
  }
  *filterp = topology->type_filter[type];
  return 0;
}

void
hwloc_topology_clear (struct hwloc_topology *topology)
{
  /* no need to set to NULL after free() since callers will call setup_defaults() or just destroy the rest of the topology */
  unsigned l;
  hwloc_internal_cpukinds_destroy(topology);
  hwloc_internal_distances_destroy(topology);
  hwloc_internal_memattrs_destroy(topology);
  hwloc_free_object_and_children(topology->levels[0][0]);
  hwloc_bitmap_free(topology->allowed_cpuset);
  hwloc_bitmap_free(topology->allowed_nodeset);
  for (l=0; l<topology->nb_levels; l++)
    free(topology->levels[l]);
  for(l=0; l<HWLOC_NR_SLEVELS; l++)
    free(topology->slevels[l].objs);
  free(topology->machine_memory.page_types);
}

```

这段代码定义了一个名为 `hwloc_topology_destroy` 的函数，其作用是释放给定的 `hwloc_topology` 结构中的内存并销毁其对应的 `hwloc_topology_device` 结构。

具体来说，函数首先检查给定的 `topology` 结构中是否已经继承自 `hwloc_topology_device` 类型，如果是，则调用一个名为 `hwloc__topology_disadopt` 的函数，该函数会禁止使用该结构对应的 `hwloc_topology_device` 类型，然后返回。否则，函数会执行以下操作：

1. 禁用给定 `topology` 结构中的所有后端；
2. 调用 `hwloc_topology_components_fini` 函数和 `hwloc_topology_clear` 函数，清除 `topology` 结构中的组件和数据；
3. 释放给定的 `topology` 结构中的所有指针，包括 `levels` 链表、`level_nbobjects` 计数器和 `support` 结构中的所有成员；
4. 释放给定的 `hwloc_topology_device` 结构中的所有指针。

该函数在 `hwloc_topology_destroy` 函数中首次被调用时，会创建一个新的 `hwloc_topology` 结构，并且 `topology->adopted_shmem_addr` 字段指示该结构已经继承自 `hwloc_topology_device` 类型。在这种情况下，函数会执行上述操作，然后返回。在后续调用中，如果 `topology` 结构中的 `adopted_shmem_addr` 字段为 `true`，则说明该结构已经继承自 `hwloc_topology_device` 类型，函数不会执行上述操作，而是直接返回。


```cpp
void
hwloc_topology_destroy (struct hwloc_topology *topology)
{
  if (topology->adopted_shmem_addr) {
    hwloc__topology_disadopt(topology);
    return;
  }

  hwloc_backends_disable_all(topology);
  hwloc_topology_components_fini(topology);
  hwloc_components_fini();

  hwloc_topology_clear(topology);

  free(topology->levels);
  free(topology->level_nbobjects);

  free(topology->support.discovery);
  free(topology->support.cpubind);
  free(topology->support.membind);
  free(topology->support.misc);
  free(topology);
}

```

It looks like you are trying to discover the hardware location(s) of a system, such as a GPU or a妙臂。 这个代码段似乎在做什么呢？

首先，它定义了一个名为 topology 的结构体，其中包含了一些关于硬件位置的信息。 它包含一个名为 backends 的成员变量，表示正在寻找的硬件位置的背景进程。

然后，它检查当前正在寻找的硬件位置是否为真（即，从环境中获取了 "HWLOC\_SYNTHETIC" 或 "HWLOC\_XMLFILE" 环境变量）。如果是，它将启用环境设置，以便在背景进程上使用。

此外，如果 topology 对象的 backends 成员变量为真，它将尝试使用 topology 中的 backends 系统。 它将检查操作系统是否支持设置允许状态，以确定要允许哪些设置。

最后，它似乎还包含一些错误处理代码，但这些代码似乎没有被使用。


```cpp
int
hwloc_topology_load (struct hwloc_topology *topology)
{
  struct hwloc_disc_status dstatus;
  const char *env;
  int err;

  if (topology->is_loaded) {
    errno = EBUSY;
    return -1;
  }

  /* initialize envvar-related things */
  hwloc_internal_distances_prepare(topology);
  hwloc_internal_memattrs_prepare(topology);

  if (getenv("HWLOC_XML_USERDATA_NOT_DECODED"))
    topology->userdata_not_decoded = 1;

  /* Ignore variables if HWLOC_COMPONENTS is set. It will be processed later */
  if (!getenv("HWLOC_COMPONENTS")) {
    /* Only apply variables if we have not changed the backend yet.
     * Only the first one will be kept.
     * Check for FSROOT first since it's for debugging so likely needs to override everything else.
     * Check for XML last (that's the one that may be set system-wide by administrators)
     * so that it's only used if other variables are not set,
     * to allow users to override easily.
     */
    if (!topology->backends) {
      const char *fsroot_path_env = getenv("HWLOC_FSROOT");
      if (fsroot_path_env)
	hwloc_disc_component_force_enable(topology,
					  1 /* env force */,
					  "linux",
					  NULL /* backend will getenv again */, NULL, NULL);
    }
    if (!topology->backends) {
      const char *cpuid_path_env = getenv("HWLOC_CPUID_PATH");
      if (cpuid_path_env)
	hwloc_disc_component_force_enable(topology,
					  1 /* env force */,
					  "x86",
					  NULL /* backend will getenv again */, NULL, NULL);
    }
    if (!topology->backends) {
      const char *synthetic_env = getenv("HWLOC_SYNTHETIC");
      if (synthetic_env)
	hwloc_disc_component_force_enable(topology,
					  1 /* env force */,
					  "synthetic",
					  synthetic_env, NULL, NULL);
    }
    if (!topology->backends) {
      const char *xmlpath_env = getenv("HWLOC_XMLFILE");
      if (xmlpath_env)
	hwloc_disc_component_force_enable(topology,
					  1 /* env force */,
					  "xml",
					  xmlpath_env, NULL, NULL);
    }
  }

  dstatus.excluded_phases = 0;
  dstatus.flags = 0; /* did nothing yet */

  env = getenv("HWLOC_ALLOW");
  if (env && !strcmp(env, "all"))
    /* don't retrieve the sets of allowed resources */
    dstatus.flags |= HWLOC_DISC_STATUS_FLAG_GOT_ALLOWED_RESOURCES;

  /* instantiate all possible other backends now */
  hwloc_disc_components_enable_others(topology);
  /* now that backends are enabled, update the thissystem flag and some callbacks */
  hwloc_backends_is_thissystem(topology);
  hwloc_backends_find_callbacks(topology);
  /*
   * Now set binding hooks according to topology->is_thissystem
   * and what the native OS backend offers.
   */
  hwloc_set_binding_hooks(topology);

  /* actual topology discovery */
  err = hwloc_discover(topology, &dstatus);
  if (err < 0)
    goto out;

```

This is a function that appears to perform hardware location targeting during the discovery phase of a HWLOC (Highway Land Location) tool.

It does this by first creating a bitmap that represents the hardware location for the CPU binding, and then checking if the binding is allowed. If the binding is allowed, it also filters out the backends that should not be mapped to the CPU binding during discovery.

If the topology does not have any hardware location restrictions and the backends are allowed to be mapped to the CPU binding, it creates a bitmap that represents the hardware location for the backends.

If the topology has hardware location restrictions and the backends are allowed to be mapped to the CPU binding, it sets the backend as dirty and sets the discovery mode to TWEAK (Two Weeks Away).

It also似乎在设置完硬件位置限制和允许后，还发送一个通知给操作系统，告诉它已准备好进行硬件位置定位。


```cpp
#ifndef HWLOC_DEBUG
  if (getenv("HWLOC_DEBUG_CHECK"))
#endif
    hwloc_topology_check(topology);

  /* Rank cpukinds */
  hwloc_internal_cpukinds_rank(topology);

  /* Mark distances objs arrays as invalid since we may have removed objects
   * from the topology after adding the distances (remove_empty, etc).
   * It would be hard to actually verify whether it's needed.
   */
  hwloc_internal_distances_invalidate_cached_objs(topology);
  /* And refresh distances so that multithreaded concurrent distances_get()
   * don't refresh() concurrently (disallowed).
   */
  hwloc_internal_distances_refresh(topology);

  /* Same for memattrs */
  hwloc_internal_memattrs_need_refresh(topology);
  hwloc_internal_memattrs_refresh(topology);
  hwloc_internal_memattrs_guess_memory_tiers(topology);

  topology->is_loaded = 1;

  if (topology->flags & HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_CPUBINDING) {
    /* FIXME: filter directly in backends during the discovery.
     * Only x86 does it because binding may cause issues on Windows.
     */
    hwloc_bitmap_t set = hwloc_bitmap_alloc();
    if (set) {
      err = hwloc_get_cpubind(topology, set, HWLOC_CPUBIND_STRICT);
      if (!err)
        hwloc_topology_restrict(topology, set, 0);
      hwloc_bitmap_free(set);
    }
  }
  if (topology->flags & HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_MEMBINDING) {
    /* FIXME: filter directly in backends during the discovery.
     */
    hwloc_bitmap_t set = hwloc_bitmap_alloc();
    hwloc_membind_policy_t policy;
    if (set) {
      err = hwloc_get_membind(topology, set, &policy, HWLOC_MEMBIND_STRICT | HWLOC_MEMBIND_BYNODESET);
      if (!err)
        hwloc_topology_restrict(topology, set, HWLOC_RESTRICT_FLAG_BYNODESET);
      hwloc_bitmap_free(set);
    }
  }

  if (topology->backend_phases & HWLOC_DISC_PHASE_TWEAK) {
    dstatus.phase = HWLOC_DISC_PHASE_TWEAK;
    hwloc_discover_by_phase(topology, &dstatus, "TWEAK");
  }

  return 0;

 out:
  hwloc_pci_discovery_exit(topology);
  hwloc_topology_clear(topology);
  hwloc_topology_setup_defaults(topology);
  hwloc_backends_disable_all(topology);
  return -1;
}

```

这段代码的作用是检查给定的对象(Object)，如果它是一个硬件资源(如CPU或GPU)，并且它的顶片(Topology)中存在子顶片(Drop)，则使用给定的选项(Options)对它进行限制。

具体来说，这段代码会按照给定的选项对对象进行限制，包括限制它的CPU或GPU的集合，以及限制它的输入或输出。如果对象是一个硬件资源，但是它的顶片中不存在子顶片，则不会对它进行任何限制。如果对象已经被连接到其他对象中，则代码会解引用它，并使用topology中当前状态的最后修改时间来更新它。

如果给定的选项中包含“allow_object_in_cpuset_旗号”，则不会在给定的CPU集合中放置该对象。相反，如果包含“force_element_in_cpuset_旗号”，则会在指定的CPU集合中放置该对象，即使它的元素集合与给定的CPU集合不同。


```cpp
/* adjust object cpusets according the given droppedcpuset,
 * drop object whose cpuset becomes empty and that have no children,
 * and propagate NUMA node removal as nodeset changes in parents.
 */
static void
restrict_object_by_cpuset(hwloc_topology_t topology, unsigned long flags, hwloc_obj_t *pobj,
			  hwloc_bitmap_t droppedcpuset, hwloc_bitmap_t droppednodeset)
{
  hwloc_obj_t obj = *pobj, child, *pchild;
  int modified = 0;

  if (hwloc_bitmap_intersects(obj->complete_cpuset, droppedcpuset)) {
    hwloc_bitmap_andnot(obj->cpuset, obj->cpuset, droppedcpuset);
    hwloc_bitmap_andnot(obj->complete_cpuset, obj->complete_cpuset, droppedcpuset);
    modified = 1;
  } else {
    if ((flags & HWLOC_RESTRICT_FLAG_REMOVE_CPULESS)
	&& hwloc_bitmap_iszero(obj->complete_cpuset)) {
      /* we're empty, there's a NUMAnode below us, it'll be removed this time */
      modified = 1;
    }
    /* nodeset cannot intersect unless cpuset intersects or is empty */
    if (droppednodeset)
      assert(!hwloc_bitmap_intersects(obj->complete_nodeset, droppednodeset)
	     || hwloc_bitmap_iszero(obj->complete_cpuset));
  }
  if (droppednodeset) {
    hwloc_bitmap_andnot(obj->nodeset, obj->nodeset, droppednodeset);
    hwloc_bitmap_andnot(obj->complete_nodeset, obj->complete_nodeset, droppednodeset);
  }

  if (modified) {
    for_each_child_safe(child, obj, pchild)
      restrict_object_by_cpuset(topology, flags, pchild, droppedcpuset, droppednodeset);
    /* if some hwloc_bitmap_first(child->complete_cpuset) changed, children might need to be reordered */
    hwloc__reorder_children(obj);

    for_each_memory_child_safe(child, obj, pchild)
      restrict_object_by_cpuset(topology, flags, pchild, droppedcpuset, droppednodeset);
    /* local NUMA nodes have the same cpusets, no need to reorder them */

    /* Nothing to restrict under I/O or Misc */
  }

  if (!obj->first_child && !obj->memory_first_child /* arity not updated before connect_children() */
      && hwloc_bitmap_iszero(obj->cpuset)
      && (obj->type != HWLOC_OBJ_NUMANODE || (flags & HWLOC_RESTRICT_FLAG_REMOVE_CPULESS))) {
    /* remove object */
    hwloc_debug("%s", "\nRemoving object during restrict by cpuset");
    hwloc_debug_print_object(0, obj);

    if (!(flags & HWLOC_RESTRICT_FLAG_ADAPT_IO)) {
      hwloc_free_object_siblings_and_children(obj->io_first_child);
      obj->io_first_child = NULL;
    }
    if (!(flags & HWLOC_RESTRICT_FLAG_ADAPT_MISC)) {
      hwloc_free_object_siblings_and_children(obj->misc_first_child);
      obj->misc_first_child = NULL;
    }
    assert(!obj->first_child);
    assert(!obj->memory_first_child);
    unlink_and_free_single_object(pobj);
    topology->modified = 1;
  }
}

```

This code appears to restrict the object based on the `nodeset` of the `obj` alias. It does this by first checking if the `obj` has a `first_child` and if it does not have a `memory_first_child`. If the `obj` does not have a `first_child`, it checks if it is a `HWLOC_OBJ_PU`. If the `obj` is a `HWLOC_OBJ_PU`, it checks if the `complete_cpuset` of the `obj` has changed. If it has, the children of the `obj` may need to be reordered.

The function then iterates through all the memory children of the `obj` and checks the `droppedcpuset` and `droppednodeset` of each child. It also, if the `obj` is a `HWLOC_OBJ_PU`, checks if the `complete_cpuset` of the `obj` has changed and if some of their nodes were removed. If the `complete_cpuset` has changed or some nodes were removed, it reorders the children of the `obj`.

If the `obj` is not a `HWLOC_OBJ_PU`, it is not necessary to check if the `complete_cpuset` of the `obj` has changed or if some nodes were removed, as nothing in this case will affect the order of the children.


```cpp
/* adjust object nodesets according the given droppednodeset,
 * drop object whose nodeset becomes empty and that have no children,
 * and propagate PU removal as cpuset changes in parents.
 */
static void
restrict_object_by_nodeset(hwloc_topology_t topology, unsigned long flags, hwloc_obj_t *pobj,
			   hwloc_bitmap_t droppedcpuset, hwloc_bitmap_t droppednodeset)
{
  hwloc_obj_t obj = *pobj, child, *pchild;
  int modified = 0;

  if (hwloc_bitmap_intersects(obj->complete_nodeset, droppednodeset)) {
    hwloc_bitmap_andnot(obj->nodeset, obj->nodeset, droppednodeset);
    hwloc_bitmap_andnot(obj->complete_nodeset, obj->complete_nodeset, droppednodeset);
    modified = 1;
  } else {
    if ((flags & HWLOC_RESTRICT_FLAG_REMOVE_MEMLESS)
	&& hwloc_bitmap_iszero(obj->complete_nodeset)) {
      /* we're empty, there's a PU below us, it'll be removed this time */
      modified = 1;
    }
    /* cpuset cannot intersect unless nodeset intersects or is empty */
    if (droppedcpuset)
      assert(!hwloc_bitmap_intersects(obj->complete_cpuset, droppedcpuset)
	     || hwloc_bitmap_iszero(obj->complete_nodeset));
  }
  if (droppedcpuset) {
    hwloc_bitmap_andnot(obj->cpuset, obj->cpuset, droppedcpuset);
    hwloc_bitmap_andnot(obj->complete_cpuset, obj->complete_cpuset, droppedcpuset);
  }

  if (modified) {
    for_each_child_safe(child, obj, pchild)
      restrict_object_by_nodeset(topology, flags, pchild, droppedcpuset, droppednodeset);
    if (flags & HWLOC_RESTRICT_FLAG_REMOVE_MEMLESS)
      /* cpuset may have changed above where some NUMA nodes were removed.
       * if some hwloc_bitmap_first(child->complete_cpuset) changed, children might need to be reordered */
      hwloc__reorder_children(obj);

    for_each_memory_child_safe(child, obj, pchild)
      restrict_object_by_nodeset(topology, flags, pchild, droppedcpuset, droppednodeset);
    /* FIXME: we may have to reorder CPU-less groups of NUMA nodes if some of their nodes were removed */

    /* Nothing to restrict under I/O or Misc */
  }

  if (!obj->first_child && !obj->memory_first_child /* arity not updated before connect_children() */
      && hwloc_bitmap_iszero(obj->nodeset)
      && (obj->type != HWLOC_OBJ_PU || (flags & HWLOC_RESTRICT_FLAG_REMOVE_MEMLESS))) {
    /* remove object */
    hwloc_debug("%s", "\nRemoving object during restrict by nodeset");
    hwloc_debug_print_object(0, obj);

    if (!(flags & HWLOC_RESTRICT_FLAG_ADAPT_IO)) {
      hwloc_free_object_siblings_and_children(obj->io_first_child);
      obj->io_first_child = NULL;
    }
    if (!(flags & HWLOC_RESTRICT_FLAG_ADAPT_MISC)) {
      hwloc_free_object_siblings_and_children(obj->misc_first_child);
      obj->misc_first_child = NULL;
    }
    assert(!obj->first_child);
    assert(!obj->memory_first_child);
    unlink_and_free_single_object(pobj);
    topology->modified = 1;
  }
}

```

hwloc_filter_levels_take_recursive_errors(topology, &topology->errors, &errors);
hwloc_remove_with_default_constructor(topology, "");
if (errors < 0)
 goto out;
```cpp
out:
hwloc_bitmap_free(droppedcpuset);
hwloc_bitmap_free(droppednodeset);
einfo
{"error_string", "Error looking at the object that caused the failure", A_ALPHA(0)}
```
}
```cpp

This function handles the case where a node set is dropped and should be propagated through the entire tree to ensure that the levels are updated correctly.

First, we check if the node set is empty and if it is, we free the bitmap and return an error code.

If the node set is not empty, we first clear the bitmap and return an error code if the bitmap is empty.

We then recursively filter the set and any connected components based on the topology level.

We also drop the node set and check for any remaining node sets by checking if the bitmap is empty or if the node set is empty.

Finally, we handle the object that caused the failure by updating the internal distances and memo



```
int
hwloc_topology_restrict(struct hwloc_topology *topology, hwloc_const_bitmap_t set, unsigned long flags)
{
  hwloc_bitmap_t droppedcpuset, droppednodeset;

  if (!topology->is_loaded) {
    errno = EINVAL;
    return -1;
  }
  if (topology->adopted_shmem_addr) {
    errno = EPERM;
    return -1;
  }

  if (flags & ~(HWLOC_RESTRICT_FLAG_REMOVE_CPULESS
		|HWLOC_RESTRICT_FLAG_ADAPT_MISC|HWLOC_RESTRICT_FLAG_ADAPT_IO
		|HWLOC_RESTRICT_FLAG_BYNODESET|HWLOC_RESTRICT_FLAG_REMOVE_MEMLESS)) {
    errno = EINVAL;
    return -1;
  }

  if (flags & HWLOC_RESTRICT_FLAG_BYNODESET) {
    /* cannot use CPULESS with BYNODESET */
    if (flags & HWLOC_RESTRICT_FLAG_REMOVE_CPULESS) {
      errno = EINVAL;
      return -1;
    }
  } else {
    /* cannot use MEMLESS without BYNODESET */
    if (flags & HWLOC_RESTRICT_FLAG_REMOVE_MEMLESS) {
      errno = EINVAL;
      return -1;
    }
  }

  /* make sure we'll keep something in the topology */
  if (((flags & HWLOC_RESTRICT_FLAG_BYNODESET) && !hwloc_bitmap_intersects(set, topology->allowed_nodeset))
      || (!(flags & HWLOC_RESTRICT_FLAG_BYNODESET) && !hwloc_bitmap_intersects(set, topology->allowed_cpuset))) {
    errno = EINVAL; /* easy failure, just don't touch the topology */
    return -1;
  }

  droppedcpuset = hwloc_bitmap_alloc();
  droppednodeset = hwloc_bitmap_alloc();
  if (!droppedcpuset || !droppednodeset) {
    hwloc_bitmap_free(droppedcpuset);
    hwloc_bitmap_free(droppednodeset);
    return -1;
  }

  if (flags & HWLOC_RESTRICT_FLAG_BYNODESET) {
    /* nodeset to clear */
    hwloc_bitmap_not(droppednodeset, set);
    /* cpuset to clear */
    if (flags & HWLOC_RESTRICT_FLAG_REMOVE_MEMLESS) {
      hwloc_obj_t pu = hwloc_get_obj_by_type(topology, HWLOC_OBJ_PU, 0);
      assert(pu);
      do {
	/* PU will be removed if cpuset gets or was empty */
	if (hwloc_bitmap_iszero(pu->cpuset)
	    || hwloc_bitmap_isincluded(pu->nodeset, droppednodeset))
	  hwloc_bitmap_set(droppedcpuset, pu->os_index);
	pu = pu->next_cousin;
      } while (pu);

      /* check we're not removing all PUs */
      if (hwloc_bitmap_isincluded(topology->allowed_cpuset, droppedcpuset)) {
	errno = EINVAL; /* easy failure, just don't touch the topology */
	hwloc_bitmap_free(droppedcpuset);
	hwloc_bitmap_free(droppednodeset);
	return -1;
      }
    }
    /* remove cpuset if empty */
    if (!(flags & HWLOC_RESTRICT_FLAG_REMOVE_MEMLESS)
	|| hwloc_bitmap_iszero(droppedcpuset)) {
      hwloc_bitmap_free(droppedcpuset);
      droppedcpuset = NULL;
    }

    /* now recurse to filter sets and drop things */
    restrict_object_by_nodeset(topology, flags, &topology->levels[0][0], droppedcpuset, droppednodeset);
    hwloc_bitmap_andnot(topology->allowed_nodeset, topology->allowed_nodeset, droppednodeset);
    if (droppedcpuset)
      hwloc_bitmap_andnot(topology->allowed_cpuset, topology->allowed_cpuset, droppedcpuset);

  } else {
    /* cpuset to clear */
    hwloc_bitmap_not(droppedcpuset, set);
    /* nodeset to clear */
    if (flags & HWLOC_RESTRICT_FLAG_REMOVE_CPULESS) {
      hwloc_obj_t node = hwloc_get_obj_by_type(topology, HWLOC_OBJ_NUMANODE, 0);
      assert(node);
      do {
	/* node will be removed if nodeset gets or was empty */
	if (hwloc_bitmap_iszero(node->cpuset)
	    || hwloc_bitmap_isincluded(node->cpuset, droppedcpuset))
	  hwloc_bitmap_set(droppednodeset, node->os_index);
	node = node->next_cousin;
      } while (node);

      /* check we're not removing all NUMA nodes */
      if (hwloc_bitmap_isincluded(topology->allowed_nodeset, droppednodeset)) {
	errno = EINVAL; /* easy failure, just don't touch the topology */
	hwloc_bitmap_free(droppedcpuset);
	hwloc_bitmap_free(droppednodeset);
	return -1;
      }
    }
    /* remove nodeset if empty */
    if (!(flags & HWLOC_RESTRICT_FLAG_REMOVE_CPULESS)
	|| hwloc_bitmap_iszero(droppednodeset)) {
      hwloc_bitmap_free(droppednodeset);
      droppednodeset = NULL;
    }

    /* now recurse to filter sets and drop things */
    restrict_object_by_cpuset(topology, flags, &topology->levels[0][0], droppedcpuset, droppednodeset);
    hwloc_bitmap_andnot(topology->allowed_cpuset, topology->allowed_cpuset, droppedcpuset);
    if (droppednodeset)
      hwloc_bitmap_andnot(topology->allowed_nodeset, topology->allowed_nodeset, droppednodeset);
  }

  hwloc_bitmap_free(droppedcpuset);
  hwloc_bitmap_free(droppednodeset);

  if (hwloc_filter_levels_keep_structure(topology) < 0) /* takes care of reconnecting internally */
    goto out;

  /* some objects may have disappeared, we need to update distances objs arrays */
  hwloc_internal_distances_invalidate_cached_objs(topology);
  hwloc_internal_memattrs_need_refresh(topology);

  hwloc_propagate_symmetric_subtree(topology, topology->levels[0][0]);
  propagate_total_memory(topology->levels[0][0]);
  hwloc_internal_cpukinds_restrict(topology);

```cpp

这段代码是一个 C 语言函数，定义在 `hwloc_debug.h` 头文件中。它主要作用是检查定义在 `hwloc_debug.h` 中的 `HWLOC_DEBUG` 环境变量是否设置为 `1`，如果是，则执行下面的操作，否则执行下面的操作。

操作一是检查 `HWLOC_DEBUG_CHECK` 环境变量是否被定义，如果没有定义，则执行下面的操作。

操作二是执行 `hwloc_topology_check` 函数，这个函数用于检查并设置 HWLOC 拓扑结构。

操作三是如果 `HWLOC_DEBUG_CHECK` 环境变量设置为 `1`，则执行操作二，即先调用 `hwloc_topology_clear` 函数清除之前的拓扑结构，再调用 `hwloc_topology_setup_defaults` 函数设置拓扑结构的默认值，最后返回 `0`。

操作四是如果 `HWLOC_DEBUG_CHECK` 环境变量设置为 `0` 或上述操作三返回 `-1`，则执行操作五，即不执行任何操作，返回 `-1`。


```
#ifndef HWLOC_DEBUG
  if (getenv("HWLOC_DEBUG_CHECK"))
#endif
    hwloc_topology_check(topology);

  return 0;

 out:
  /* unrecoverable failure, re-init the topology */
   hwloc_topology_clear(topology);
   hwloc_topology_setup_defaults(topology);
   return -1;
}

int
```cpp

This function appears to be a part of a system's topology binding mechanism, which handles the interaction between the topology and the underlying hardware. It is called `allow_resources()`.

The function takes in a single argument, `topology`, which is an instance of a `Topology` class that represents the topology for a particular system.

The function first checks if `topology` is a device-specific topology and then checks if there is a `device_id` attribute set on topology. If there is no `device_id`, the function falls back to the default behavior which is to assume that the topology is a full topology.

If the topology is a device-specific topology, the function then checks if the `binding_hooks` attribute has a `get_allowed_resources()` method. If it does, the function retrieves a list of allowed resources from the `topology` instance and then checks if the `backend` returned any valid resources. If the `backend` returns no resources, the function returns an error with an error code.

If the topology is a full topology, the function then initializes the bitmap for the `allowed_cpuset` and `allowed_nodeset` to the same value as `topology->allowed_cpuset` and `topology->allowed_nodeset`, respectively.

The function then makes sure that the backend has returned a valid set of resources by first checking if the `hwloc_bitmap_intersects()` function returns a non-empty bitmap if the `topology->allowed_cpuset` and `cpuset` are different. If the bitmap is not empty, the function sets the `topology->allowed_cpuset` to the same value as `topology->allowed_cpuset`.

Finally, the function checks if the `topology->binding_hooks` has a `get_allowed_resources()` method and if it does, the function calls the `get_allowed_resources()` method and sets the `topology->allowed_nodeset` and `topology->allowed_cpuset` as described in the comment above.


```
hwloc_topology_allow(struct hwloc_topology *topology,
		     hwloc_const_cpuset_t cpuset, hwloc_const_nodeset_t nodeset,
		     unsigned long flags)
{
  if (!topology->is_loaded)
    goto einval;

  if (topology->adopted_shmem_addr) {
    errno = EPERM;
    goto error;
  }

  if (!(topology->flags & HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED))
    goto einval;

  if (flags & ~(HWLOC_ALLOW_FLAG_ALL|HWLOC_ALLOW_FLAG_LOCAL_RESTRICTIONS|HWLOC_ALLOW_FLAG_CUSTOM))
    goto einval;

  switch (flags) {
  case HWLOC_ALLOW_FLAG_ALL: {
    if (cpuset || nodeset)
      goto einval;
    hwloc_bitmap_copy(topology->allowed_cpuset, hwloc_get_root_obj(topology)->complete_cpuset);
    hwloc_bitmap_copy(topology->allowed_nodeset, hwloc_get_root_obj(topology)->complete_nodeset);
    break;
  }
  case HWLOC_ALLOW_FLAG_LOCAL_RESTRICTIONS: {
    if (cpuset || nodeset)
      goto einval;
    if (!topology->is_thissystem)
      goto einval;
    if (!topology->binding_hooks.get_allowed_resources) {
      errno = ENOSYS;
      goto error;
    }
    topology->binding_hooks.get_allowed_resources(topology);
    /* make sure the backend returned something sane (Linux cpusets may return offline PUs in some cases) */
    hwloc_bitmap_and(topology->allowed_cpuset, topology->allowed_cpuset, hwloc_get_root_obj(topology)->cpuset);
    hwloc_bitmap_and(topology->allowed_nodeset, topology->allowed_nodeset, hwloc_get_root_obj(topology)->nodeset);
    break;
  }
  case HWLOC_ALLOW_FLAG_CUSTOM: {
    if (cpuset) {
      /* keep the intersection with the full topology cpuset, if not empty */
      if (!hwloc_bitmap_intersects(hwloc_get_root_obj(topology)->cpuset, cpuset))
	goto einval;
      hwloc_bitmap_and(topology->allowed_cpuset, hwloc_get_root_obj(topology)->cpuset, cpuset);
    }
    if (nodeset) {
      /* keep the intersection with the full topology nodeset, if not empty */
      if (!hwloc_bitmap_intersects(hwloc_get_root_obj(topology)->nodeset, nodeset))
	goto einval;
      hwloc_bitmap_and(topology->allowed_nodeset, hwloc_get_root_obj(topology)->nodeset, nodeset);
    }
    break;
  }
  default:
    goto einval;
  }

  return 0;

 einval:
  errno = EINVAL;
 error:
  return -1;
}

```cpp



这两段代码定义了两个名为 `hwloc_topology_refresh` 和 `hwloc_topology_is_thissystem` 的函数。它们属于 `hwloc_topology` 结构体的成员函数。

1. `hwloc_topology_refresh` 函数的作用是刷新 `topology` 结构体的内部状态，包括：

  - `hwloc_internal_cpukinds_rank` 函数会输出 `topology` 中的所有 CPU 设备节点，并按照它们的 CPU 类型进行排名。
  - `hwloc_internal_distances_refresh` 函数会输出 `topology` 中的所有网络设备节点，并按照它们的网络距离进行刷新。
  - `hwloc_internal_memattrs_refresh` 函数会输出 `topology` 中的所有内存控制器节点，并按照它们的内存属性进行刷新。

  返回值为 0，表示 `hwloc_topology_refresh` 函数成功刷新了 `topology` 结构体。

2. `hwloc_topology_is_thissystem` 函数的作用是检查给定的 `topology` 结构体是否为当前系统。它返回一个布尔值，表示给定的 `topology` 结构体是否与当前系统相同。

这两段代码都是定义在 `hwloc_topology.h` 文件中，是在 Linux 系统的 `hwloc` 包内部定义的函数。


```
int
hwloc_topology_refresh(struct hwloc_topology *topology)
{
  hwloc_internal_cpukinds_rank(topology);
  hwloc_internal_distances_refresh(topology);
  hwloc_internal_memattrs_refresh(topology);
  return 0;
}

int
hwloc_topology_is_thissystem(struct hwloc_topology *topology)
{
  return topology->is_thissystem;
}

```cpp



这段代码定义了三个函数，用于获取和设置 HWLOC_TOPOLOGY 结构体中 topology 的不同方面，并实现 topology 中的一个用户数据设置。

第一个函数 `hwloc_topology_get_depth(struct hwloc_topology *topology)` 返回 topology 中的分层结构层数，它由 `topology->nb_levels` 计算得出。

第二个函数 `hwloc_topology_get_support(struct hwloc_topology *topology)` 返回 topology 中的支持结构，它指向了 topology 的支持结构体。

第三个函数 `hwloc_topology_set_userdata(struct hwloc_topology *topology, const void *userdata)` 允许用户数据通过 `topology->userdata` 指针进行设置。

该代码根据 `hwloc_topology_get_depth(topology)` 和 `hwloc_topology_get_support(topology)` 函数可以得到一个 depth 0 的 topology，即 topology 的根节点。

`hwloc_topology_set_userdata(topology, userdata)` 函数用于设置 topology 中的用户数据，通过指针 `userdata` 实现。这个设置可以在 topology 的创建或某些修改过程中进行，例如，在 `hwloc_topology_generate_l2(topology)` 函数中。


```
int
hwloc_topology_get_depth(struct hwloc_topology *topology)
{
  return (int) topology->nb_levels;
}

const struct hwloc_topology_support *
hwloc_topology_get_support(struct hwloc_topology * topology)
{
  return &topology->support;
}

void hwloc_topology_set_userdata(struct hwloc_topology * topology, const void *userdata)
{
  topology->userdata = (void *) userdata;
}

```cpp



这三段代码是用于获取 `hwloc_topology` 结构体的用户数据、完成CPU集和当前CPU集的方法。

`hwloc_topology_get_userdata()` 函数返回 `topology` 结构体的 `userdata` 成员的地址，即 `topology->userdata` 的地址。

`hwloc_topology_get_complete_cpuset()` 函数返回 `topology` 结构体的 `complete_cpuset` 成员的地址，即 `hwloc_get_root_obj(topology)->complete_cpuset` 的地址。这里使用了 `hwloc_get_root_obj()` 函数来获取 `topology` 结构体的根对象。

`hwloc_topology_get_topology_cpuset()` 函数返回 `topology` 结构体的 `cpuset` 成员的地址，即 `hwloc_get_root_obj(topology)->cpuset` 的地址。

这三段代码的具体作用是用于从 `hwloc_topology` 结构体中获取一些与 `topology` 相关的成员，包括 `userdata`、`complete_cpuset` 和 `cpuset`。


```
void * hwloc_topology_get_userdata(struct hwloc_topology * topology)
{
  return topology->userdata;
}

hwloc_const_cpuset_t
hwloc_topology_get_complete_cpuset(hwloc_topology_t topology)
{
  return hwloc_get_root_obj(topology)->complete_cpuset;
}

hwloc_const_cpuset_t
hwloc_topology_get_topology_cpuset(hwloc_topology_t topology)
{
  return hwloc_get_root_obj(topology)->cpuset;
}

```cpp

这三段代码都是属于 hwloc_topology_t 类型的函数，用于获取顶端网络中的所有合法的 CPU 集、顶端链或者顶端集合。

1. hwloc_const_cpuset_t：返回顶端网络中的所有合法的 CPU 集。这个函数的实现是通过调用 hwloc_topology_get_allowed_cpuset 函数获取顶端网络允许的 CPU 集，然后返回给调用者。

2. hwloc_const_nodeset_t：返回顶端网络中的所有合法的顶端链或者顶端集合。这个函数的实现是通过调用 hwloc_topology_get_complete_nodeset 函数获取顶端网络的完整顶端链或者顶端集合，然后返回给调用者。

3. hwloc_const_nodeset_t：返回顶端网络中的所有合法的顶端集合。这个函数的实现是通过调用 hwloc_topology_get_topology_nodeset 函数获取顶端网络的完整顶端集合，然后返回给调用者。

这三段代码共同作用于获取顶端网络中的所有合法的元素，包括 CPU 集、顶端链和顶端集合。这些函数在 hwloc_topology_t 类型的顶端网络中使用，通过调用不同的函数，可以实现对顶端网络元素的安全访问和维护。


```
hwloc_const_cpuset_t
hwloc_topology_get_allowed_cpuset(hwloc_topology_t topology)
{
  return topology->allowed_cpuset;
}

hwloc_const_nodeset_t
hwloc_topology_get_complete_nodeset(hwloc_topology_t topology)
{
  return hwloc_get_root_obj(topology)->complete_nodeset;
}

hwloc_const_nodeset_t
hwloc_topology_get_topology_nodeset(hwloc_topology_t topology)
{
  return hwloc_get_root_obj(topology)->nodeset;
}

```cpp

这段代码定义了一个名为 `hwloc_const_nodeset_t` 的常量类型 `hwloc_const_nodeset_t`，它用于表示一个节点集中的节点。然后，定义了一个名为 `hwloc_topology_get_allowed_nodeset` 的函数，它接受一个名为 `topology` 的 `hwloc_topology_t` 类型的参数，并返回节点集中允许的节点集合。

接下来的注释中，定义了一个名为 `hwloc__check_child_siblings` 的函数，它接受一个名为 `parent` 的 `hwloc_obj_t` 类型的参数，一个名为 `array` 的 `unsigned charp` 类型的参数，和一个名为 `i` 的 `unsigned` 类型的参数，函数的作用是检查子节点与当前节点的相邻关系。函数首先检查当前节点是否与父节点相同，然后检查当前节点是否是数组中的某个节点，接着检查父节点是否是前一个节点的相邻节点，如果前一个节点是父节点的一个相邻节点，检查当前节点是否是前一个相邻节点的后一个节点，以此类推。最后，函数还检查前一个节点是否是当前节点的父节点，如果不是，则前一个节点需要与当前节点相邻。


```
hwloc_const_nodeset_t
hwloc_topology_get_allowed_nodeset(hwloc_topology_t topology)
{
  return topology->allowed_nodeset;
}


/****************
 * Debug Checks *
 ****************/

#ifndef NDEBUG /* assert only enabled if !NDEBUG */

static void
hwloc__check_child_siblings(hwloc_obj_t parent, hwloc_obj_t *array,
			    unsigned arity, unsigned i,
			    hwloc_obj_t child, hwloc_obj_t prev)
{
  assert(child->parent == parent);

  assert(child->sibling_rank == i);
  if (array)
    assert(child == array[i]);

  if (prev)
    assert(prev->next_sibling == child);
  assert(child->prev_sibling == prev);

  if (!i)
    assert(child->prev_sibling == NULL);
  else
    assert(child->prev_sibling != NULL);

  if (i == arity-1)
    assert(child->next_sibling == NULL);
  else
    assert(child->next_sibling != NULL);
}

```cpp

这段代码是一个名为`hwloc__check_object`的函数，它是`hwloc_topology_t`结构体中`hwloc_check_object`函数的一部分。这个函数的主要作用是检查一个父对象下面的子对象，确保它们都是正常的，并且它们在树中的深度是正确的。

具体来说，这个函数接受一个指向`hwloc_topology_t`结构体的指针`topology`，一个表示`hwloc_obj_t`对象的`hwloc_bitmap_t`类型的指针`gp_indexes`，以及一个父`hwloc_obj_t`对象的引用`parent`。它首先检查父对象是否有任何真正的子对象，如果没有，就返回。然后，它检查父对象是否真的有子对象，如果是，就继续检查它的子对象是否都是正常的，它们的深度是否符合预期，以及它们是否都有正确的`hwloc_obj_t`类型。最后，它还会检查父对象的儿子对象是否都在正确的`hwloc_topology_t`子对象中，确保所有的子对象都正常，然后就返回。


```
static void
hwloc__check_object(hwloc_topology_t topology, hwloc_bitmap_t gp_indexes, hwloc_obj_t obj);

/* check children between a parent object */
static void
hwloc__check_normal_children(hwloc_topology_t topology, hwloc_bitmap_t gp_indexes, hwloc_obj_t parent)
{
  hwloc_obj_t child, prev;
  unsigned j;

  if (!parent->arity) {
    /* check whether that parent has no children for real */
    assert(!parent->children);
    assert(!parent->first_child);
    assert(!parent->last_child);
    return;
  }
  /* check whether that parent has children for real */
  assert(parent->children);
  assert(parent->first_child);
  assert(parent->last_child);

  /* sibling checks */
  for(prev = NULL, child = parent->first_child, j = 0;
      child;
      prev = child, child = child->next_sibling, j++) {
    /* normal child */
    assert(hwloc__obj_type_is_normal(child->type));
    /* check depth */
    assert(child->depth > parent->depth);
    /* check siblings */
    hwloc__check_child_siblings(parent, parent->children, parent->arity, j, child, prev);
    /* recurse */
    hwloc__check_object(topology, gp_indexes, child);
  }
  /* check arity */
  assert(j == parent->arity);

  assert(parent->first_child == parent->children[0]);
  assert(parent->last_child == parent->children[parent->arity-1]);

  /* no normal children below a PU */
  if (parent->type == HWLOC_OBJ_PU)
    assert(!parent->arity);
}

```cpp

This code snippet appears to be a part of an implementation of an object that manages cpuset. The function in this snippet is meant to be used by other functions in the same object.

The code starts by defining an assert statement that checks if the object's parent has a cpuset. If it does, the function then checks if the object's cpuset is equal to the parent's cpuset, and then attempts to verify that the object's arity is empty.

If the cpuset is not the same, the function checks if the object's type is a memory object. If it is, the function checks if the memory object has the same cpuset as the parent's cpuset. If it does not, the function attempts to allocate a memory object with the same cpuset and checks if the allocation was successful.

If the object's type is not a memory object, the function checks if the object's children have the same cpuset as the parent's cpuset. This is done by first checking if the object's children have the same arity as the parent, and if they do not, the function checks if the children' cpusets are equal to the parent's cpuset.

Finally, the function checks if the children' cpusets are properly ordered and empty cpusets are not allowed in the children.

Note that the code is implementing the cpuset of an object that manages memory objects and other memory children, so the children of the object would be memory objects and not real-world objects, and the function would be used to keep track of the memory objects that belong to the object.


```
static void
hwloc__check_children_cpusets(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_obj_t obj)
{
  /* we already checked in the caller that objects have either all sets or none */
  hwloc_obj_t child;
  int prev_first, prev_empty;

  if (obj->type == HWLOC_OBJ_PU) {
    /* PU cpuset is just itself, with no normal children */
    assert(hwloc_bitmap_weight(obj->cpuset) == 1);
    assert(hwloc_bitmap_first(obj->cpuset) == (int) obj->os_index);
    assert(hwloc_bitmap_weight(obj->complete_cpuset) == 1);
    assert(hwloc_bitmap_first(obj->complete_cpuset) == (int) obj->os_index);
    if (!(topology->flags & HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED)) {
      assert(hwloc_bitmap_isset(topology->allowed_cpuset, (int) obj->os_index));
    }
    assert(!obj->arity);
  } else if (hwloc__obj_type_is_memory(obj->type)) {
    /* memory object cpuset is equal to its parent */
    assert(hwloc_bitmap_isequal(obj->parent->cpuset, obj->cpuset));
    assert(!obj->arity);
  } else if (!hwloc__obj_type_is_special(obj->type)) {
    hwloc_bitmap_t set;
    /* other obj cpuset is an exclusive OR of normal children, except for PUs */
    set = hwloc_bitmap_alloc();
    for_each_child(child, obj) {
      assert(!hwloc_bitmap_intersects(set, child->cpuset));
      hwloc_bitmap_or(set, set, child->cpuset);
    }
    assert(hwloc_bitmap_isequal(set, obj->cpuset));
    hwloc_bitmap_free(set);
  }

  /* check that memory children have same cpuset */
  for_each_memory_child(child, obj)
    assert(hwloc_bitmap_isequal(obj->cpuset, child->cpuset));

  /* check that children complete_cpusets are properly ordered, empty ones may be anywhere
   * (can be wrong for main cpuset since removed PUs can break the ordering).
   */
  prev_first = -1; /* -1 works fine with first comparisons below */
  prev_empty = 0; /* no empty cpuset in previous children */
  for_each_child(child, obj) {
    int first = hwloc_bitmap_first(child->complete_cpuset);
    if (first >= 0) {
      assert(!prev_empty); /* no objects with CPU after objects without CPU */
      assert(prev_first < first);
    } else {
      prev_empty = 1;
    }
    prev_first = first;
  }
}

```cpp

这段代码是一个名为 `hwloc__check_memory_children` 的函数，属于 `hwloc_topology_t` 类的成员函数。它的作用是检查给定的 `hwloc_topology_t` 结构中，所有的 `hwloc_obj_t` 对象是否都符合某种特定的条件，然后输出检查结果。

具体来说，这段代码的作用如下：

1. 首先检查给定的 `parent` 对象是否有一个 `memory_arity` 属性，如果是，则执行后续的代码。
2. 如果 `parent` 对象没有 `memory_arity` 属性，则执行以下操作：
   a. 检查 `parent` 对象是否有一个 `memory_first_child` 属性，如果是，则执行后续的代码。
   b. 如果 `parent` 对象有 `memory_first_child` 属性，则执行以下操作：
      i. 检查给定的 `child` 对象是否是一个 `hwloc_obj_t` 类型的内存，如果是，并且 `hwloc__obj_type_is_memory(child->type)` 返回 `true`，则执行后续的代码。
      ii. 递归地检查给定的 `child` 对象的 `next_sibling` 属性，直到找到一个 `hwloc_obj_t` 类型的内存并且 `hwloc__obj_type_is_memory(child->type)` 返回 `true`。
      iii. 检查给定的 `child` 对象是否是 `hwloc_topology_t` 结构中的 `hwloc_numanode` 类型，如果是，并且 `hwloc_topology_t` 结构的 `memory_arity` 属性为 `true`，则执行后续的代码。
      x. 如果 `hwloc__obj_type_is_memory(child->type)` 返回 `false` 或者 `hwloc_topology_t` 结构的 `memory_arity` 属性不是 `true`，则输出 "Memory not found"。

这段代码的输出结果为 "Memory not found"。


```
static void
hwloc__check_memory_children(hwloc_topology_t topology, hwloc_bitmap_t gp_indexes, hwloc_obj_t parent)
{
  unsigned j;
  hwloc_obj_t child, prev;

  if (!parent->memory_arity) {
    /* check whether that parent has no children for real */
    assert(!parent->memory_first_child);
    return;
  }
  /* check whether that parent has children for real */
  assert(parent->memory_first_child);

  for(prev = NULL, child = parent->memory_first_child, j = 0;
      child;
      prev = child, child = child->next_sibling, j++) {
    assert(hwloc__obj_type_is_memory(child->type));
    /* check siblings */
    hwloc__check_child_siblings(parent, NULL, parent->memory_arity, j, child, prev);
    /* only Memory and Misc children, recurse */
    assert(!child->first_child);
    assert(!child->io_first_child);
    hwloc__check_object(topology, gp_indexes, child);
  }
  /* check arity */
  assert(j == parent->memory_arity);

  /* no memory children below a NUMA node */
  if (parent->type == HWLOC_OBJ_NUMANODE)
    assert(!parent->memory_arity);
}

```cpp

这段代码是一个名为 `hwloc__check_io_children` 的函数，属于 `hwloc_topology_ t` 系列的成员函数。它的作用是检查给定的 `hwloc_topology_t` 结构中，某个根 `hwloc_obj_t` 对象的所有 I/O 子对象的层次结构是否正确。

具体来说，函数首先检查给定的根对象是否没有子对象，如果是，就返回。然后，函数依次检查根对象的每个 I/O 子对象，以及它的所有相邻 I/O 子对象的层次结构。在检查每个子对象时，函数会先检查它是否为 I/O 类型，如果是，就继续检查它的兄弟姐妹对象是否也是 I/O 类型，以及它的第一个和第二个子对象（如果存在）是否都是 I/O 类型。如果子对象既不是 I/O 类型，也不是它的兄弟姐妹对象，函数就会调用 itself 继续检查。在所有 I/O 子对象都被正确检查之后，函数会再次检查给定的根对象是否满足某种特定的 I/O arity 属性，如果是，就返回。


```
static void
hwloc__check_io_children(hwloc_topology_t topology, hwloc_bitmap_t gp_indexes, hwloc_obj_t parent)
{
  unsigned j;
  hwloc_obj_t child, prev;

  if (!parent->io_arity) {
    /* check whether that parent has no children for real */
    assert(!parent->io_first_child);
    return;
  }
  /* check whether that parent has children for real */
  assert(parent->io_first_child);

  for(prev = NULL, child = parent->io_first_child, j = 0;
      child;
      prev = child, child = child->next_sibling, j++) {
    /* all children must be I/O */
    assert(hwloc__obj_type_is_io(child->type));
    /* check siblings */
    hwloc__check_child_siblings(parent, NULL, parent->io_arity, j, child, prev);
    /* only I/O and Misc children, recurse */
    assert(!child->first_child);
    assert(!child->memory_first_child);
    hwloc__check_object(topology, gp_indexes, child);
  }
  /* check arity */
  assert(j == parent->io_arity);
}

```cpp

这段代码是一个名为 `hwloc__check_misc_children` 的函数，属于 `hwloc_topology_t` 类的成员函数。它的作用是检查给定的 `hwloc_topology_t` 结构中，某个根 `hwloc_obj_t` 对象的所有 `hwloc_bitmap_t` 索引 children 是否都符合 `HWLOC_OBJ_MISC` 类型。如果是，那么这些 children 都可以被认为是 "自由孩子"，可以自由地去掉 parent 中的 children，并且不会对 parent 的子树造成任何影响。如果不是，那么就无法去掉这些 children。

具体实现中，首先判断给定的 parent 对象是否没有 children，如果是，那么直接返回。接着，判断 parent 对象是否至少有一个 children，如果是，那么就执行 children 的 check_children 函数，这个函数会递归地检查 children 中的每一个 child，确保所有的 children 都是 "自由孩子"。如果 parent 对象中 children 的个数是奇数，那么 children 中的每一个 child 都会被认为是 "半自由孩子"，需要去除。


```
static void
hwloc__check_misc_children(hwloc_topology_t topology, hwloc_bitmap_t gp_indexes, hwloc_obj_t parent)
{
  unsigned j;
  hwloc_obj_t child, prev;

  if (!parent->misc_arity) {
    /* check whether that parent has no children for real */
    assert(!parent->misc_first_child);
    return;
  }
  /* check whether that parent has children for real */
  assert(parent->misc_first_child);

  for(prev = NULL, child = parent->misc_first_child, j = 0;
      child;
      prev = child, child = child->next_sibling, j++) {
    /* all children must be Misc */
    assert(child->type == HWLOC_OBJ_MISC);
    /* check siblings */
    hwloc__check_child_siblings(parent, NULL, parent->misc_arity, j, child, prev);
    /* only Misc children, recurse */
    assert(!child->first_child);
    assert(!child->memory_first_child);
    assert(!child->io_first_child);
    hwloc__check_object(topology, gp_indexes, child);
  }
  /* check arity */
  assert(j == parent->misc_arity);
}

```cpp

This function appears to check if a hardware location has the specified properties and, if so, asserts that the corresponding node sets and cache types are also included.

It seems to be checking if the location is a cache node, and if it is, it checks the cache type and depth against the specified type and then checks if it's an instruction cache or data cache.

Then, it checks if the location is a node set, and if it is, it checks the total memory usage and checks if any children are referenced by the node set.

Finally, it checks if the location has children, and if it does, it checks the children's properties and also checks if there's a complete cpuset for the node set.


```
static void
hwloc__check_object(hwloc_topology_t topology, hwloc_bitmap_t gp_indexes, hwloc_obj_t obj)
{
  hwloc_uint64_t total_memory;
  hwloc_obj_t child;

  assert(!hwloc_bitmap_isset(gp_indexes, obj->gp_index));
  hwloc_bitmap_set(gp_indexes, obj->gp_index);

  HWLOC_BUILD_ASSERT(HWLOC_OBJ_TYPE_MIN == 0);
  assert((unsigned) obj->type < HWLOC_OBJ_TYPE_MAX);

  assert(hwloc_filter_check_keep_object(topology, obj));

  /* check that sets and depth */
  if (hwloc__obj_type_is_special(obj->type)) {
    assert(!obj->cpuset);
    if (obj->type == HWLOC_OBJ_BRIDGE)
      assert(obj->depth == HWLOC_TYPE_DEPTH_BRIDGE);
    else if (obj->type == HWLOC_OBJ_PCI_DEVICE)
      assert(obj->depth == HWLOC_TYPE_DEPTH_PCI_DEVICE);
    else if (obj->type == HWLOC_OBJ_OS_DEVICE)
      assert(obj->depth == HWLOC_TYPE_DEPTH_OS_DEVICE);
    else if (obj->type == HWLOC_OBJ_MISC)
      assert(obj->depth == HWLOC_TYPE_DEPTH_MISC);
  } else {
    assert(obj->cpuset);
    if (obj->type == HWLOC_OBJ_NUMANODE)
      assert(obj->depth == HWLOC_TYPE_DEPTH_NUMANODE);
    else if (obj->type == HWLOC_OBJ_MEMCACHE)
      assert(obj->depth == HWLOC_TYPE_DEPTH_MEMCACHE);
    else
      assert(obj->depth >= 0);
  }

  /* group depth cannot be -1 anymore in v2.0+ */
  if (obj->type == HWLOC_OBJ_GROUP) {
    assert(obj->attr->group.depth != (unsigned) -1);
  }

  /* there's other cpusets and nodesets if and only if there's a main cpuset */
  assert(!!obj->cpuset == !!obj->complete_cpuset);
  assert(!!obj->cpuset == !!obj->nodeset);
  assert(!!obj->nodeset == !!obj->complete_nodeset);

  /* check that complete/inline sets are larger than the main sets */
  if (obj->cpuset) {
    assert(hwloc_bitmap_isincluded(obj->cpuset, obj->complete_cpuset));
    assert(hwloc_bitmap_isincluded(obj->nodeset, obj->complete_nodeset));
  }

  /* check cache type/depth vs type */
  if (hwloc__obj_type_is_cache(obj->type)) {
    if (hwloc__obj_type_is_icache(obj->type))
      assert(obj->attr->cache.type == HWLOC_OBJ_CACHE_INSTRUCTION);
    else if (hwloc__obj_type_is_dcache(obj->type))
      assert(obj->attr->cache.type == HWLOC_OBJ_CACHE_DATA
	     || obj->attr->cache.type == HWLOC_OBJ_CACHE_UNIFIED);
    else
      assert(0);
    assert(hwloc_cache_type_by_depth_type(obj->attr->cache.depth, obj->attr->cache.type) == obj->type);
  }

  /* check total memory */
  total_memory = 0;
  if (obj->type == HWLOC_OBJ_NUMANODE)
    total_memory += obj->attr->numanode.local_memory;
  for_each_child(child, obj) {
    total_memory += child->total_memory;
  }
  for_each_memory_child(child, obj) {
    total_memory += child->total_memory;
  }
  assert(total_memory == obj->total_memory);

  /* check children */
  hwloc__check_normal_children(topology, gp_indexes, obj);
  hwloc__check_memory_children(topology, gp_indexes, obj);
  hwloc__check_io_children(topology, gp_indexes, obj);
  hwloc__check_misc_children(topology, gp_indexes, obj);
  hwloc__check_children_cpusets(topology, obj);
  /* nodesets are checked during another recursion with state below */
}

```cpp

This is a description of a recursive function called `parent_contributions` which takes an object `obj` and its children `child` and calculates the父贡献 of each child in the tree.

It starts by creating a bitmap called `myset` which is a copy of the nodeset of the current object. It then gets the bitmap of the parent node set of the current object and compares it with the bitmap `myset`. If they intersect, it means that the local node set of the current object has some elements that are also in the parent node set, and it将这些 elements in `myset`.

Next, it recursively calls itself with the local node set of the current object and the parent node set of the current object, passing the bitmap of the current object and the parent node set to the function.

It repeats this process for each child of the current object, getting the bitmap of the local node set and the parent node set for each child, and comparing them to the bitmap `myset`. If they intersect, it means that the local node set of the current object has some elements that are also in the parent node set, and it将这些 elements in `myset`.

If the bitmap of the local node set does not intersect with the parent node set, it means that the local node set is only present in the parent node set, and it adds this to `myset`.

Finally, it checks that the bitmap `myset` contains all the elements of the local node set and that the local node set is a combination of the parent node set and the children's node sets. If this condition is not met, it sets the `prev_first` to -1, which means that the parent node set is unknown.

This function can be useful for understanding how parents contribute to their children's node sets.


```
static void
hwloc__check_nodesets(hwloc_topology_t topology, hwloc_obj_t obj, hwloc_bitmap_t parentset)
{
  hwloc_obj_t child;
  int prev_first;

  if (obj->type == HWLOC_OBJ_NUMANODE) {
    /* NUMANODE nodeset is just itself, with no memory/normal children */
    assert(hwloc_bitmap_weight(obj->nodeset) == 1);
    assert(hwloc_bitmap_first(obj->nodeset) == (int) obj->os_index);
    assert(hwloc_bitmap_weight(obj->complete_nodeset) == 1);
    assert(hwloc_bitmap_first(obj->complete_nodeset) == (int) obj->os_index);
    if (!(topology->flags & HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED)) {
      assert(hwloc_bitmap_isset(topology->allowed_nodeset, (int) obj->os_index));
    }
    assert(!obj->arity);
    assert(!obj->memory_arity);
    assert(hwloc_bitmap_isincluded(obj->nodeset, parentset));
  } else {
    hwloc_bitmap_t myset;
    hwloc_bitmap_t childset;

    /* the local nodeset is an exclusive OR of memory children */
    myset = hwloc_bitmap_alloc();
    for_each_memory_child(child, obj) {
      assert(!hwloc_bitmap_intersects(myset, child->nodeset));
      hwloc_bitmap_or(myset, myset, child->nodeset);
    }
    /* the local nodeset cannot intersect with parents' local nodeset */
    assert(!hwloc_bitmap_intersects(myset, parentset));
    hwloc_bitmap_or(parentset, parentset, myset);
    hwloc_bitmap_free(myset);
    /* parentset now contains parent+local contribution */

    /* for each children, recurse to check/get its contribution */
    childset = hwloc_bitmap_alloc();
    for_each_child(child, obj) {
      hwloc_bitmap_t set = hwloc_bitmap_dup(parentset); /* don't touch parentset, we don't want to propagate the first child contribution to other children */
      hwloc__check_nodesets(topology, child, set);
      /* extract this child contribution */
      hwloc_bitmap_andnot(set, set, parentset);
      /* save it */
      assert(!hwloc_bitmap_intersects(childset, set));
      hwloc_bitmap_or(childset, childset, set);
      hwloc_bitmap_free(set);
    }
    /* combine child contribution into parentset */
    assert(!hwloc_bitmap_intersects(parentset, childset));
    hwloc_bitmap_or(parentset, parentset, childset);
    hwloc_bitmap_free(childset);
    /* now check that our nodeset is combination of parent, local and children */
    assert(hwloc_bitmap_isequal(obj->nodeset, parentset));
  }

  /* check that children complete_nodesets are properly ordered, empty ones may be anywhere
   * (can be wrong for main nodeset since removed PUs can break the ordering).
   */
  prev_first = -1; /* -1 works fine with first comparisons below */
  for_each_memory_child(child, obj) {
    int first = hwloc_bitmap_first(child->complete_nodeset);
    assert(prev_first < first);
    prev_first = first;
  }
}

```cpp

This code appears to be testing the equality of an object with a specific type with itself. It is using a depth-first search (DFS) algorithm to navigate through the object hierarchy, starting at a specific depth and following the specified directories (either PUS or NUMA).

The code first checks that the object is an equal object with a specific type (HWLOC_OBJ_NUMANODE). If it is, it checks that the object has a complete nodeset and that it is a numbered node (i.e., an NUMA node).

Then, it starts a depth-first search from the top layer of the hierarchy, starting at a specific depth and following the specified directories. It checks for the first object of the level, and then, if it doesn't find it, it checks that the object is not present.

Finally, it checks that the object is present at the last depth in the hierarchy, and that it has the correct CPU-set and node set.


```
static void
hwloc__check_level(struct hwloc_topology *topology, int depth,
		   hwloc_obj_t first, hwloc_obj_t last)
{
  unsigned width = hwloc_get_nbobjs_by_depth(topology, depth);
  struct hwloc_obj *prev = NULL;
  hwloc_obj_t obj;
  unsigned j;

  /* check each object of the level */
  for(j=0; j<width; j++) {
    obj = hwloc_get_obj_by_depth(topology, depth, j);
    /* check that the object is corrected placed horizontally and vertically */
    assert(obj);
    assert(obj->depth == depth);
    assert(obj->logical_index == j);
    /* check that all objects in the level have the same type */
    if (prev) {
      assert(hwloc_type_cmp(obj, prev) == HWLOC_OBJ_EQUAL);
      assert(prev->next_cousin == obj);
    }
    assert(obj->prev_cousin == prev);

    /* check that PUs and NUMA nodes have correct cpuset/nodeset */
    if (obj->type == HWLOC_OBJ_NUMANODE) {
      assert(hwloc_bitmap_weight(obj->complete_nodeset) == 1);
      assert(hwloc_bitmap_first(obj->complete_nodeset) == (int) obj->os_index);
    }
    prev = obj;
  }
  if (prev)
    assert(prev->next_cousin == NULL);

  if (width) {
    /* check first object of the level */
    obj = hwloc_get_obj_by_depth(topology, depth, 0);
    assert(obj);
    assert(!obj->prev_cousin);
    /* check type */
    assert(hwloc_get_depth_type(topology, depth) == obj->type);
    assert(depth == hwloc_get_type_depth(topology, obj->type)
	   || HWLOC_TYPE_DEPTH_MULTIPLE == hwloc_get_type_depth(topology, obj->type));
    /* check last object of the level */
    obj = hwloc_get_obj_by_depth(topology, depth, width-1);
    assert(obj);
    assert(!obj->next_cousin);
  }

  if (depth < 0) {
    assert(first == hwloc_get_obj_by_depth(topology, depth, 0));
    assert(last == hwloc_get_obj_by_depth(topology, depth, width-1));
  } else {
    assert(!first);
    assert(!last);
  }

  /* check last+1 object of the level */
  obj = hwloc_get_obj_by_depth(topology, depth, width);
  assert(!obj);
}

```cpp

This function appears to verify that the object at index `topology->index` in the given topology object is a valid object that meets certain criteria. It does this by first checking that the object is a topology object (`topology->type == HWLOC_OBJECT_TYPE_ topology`), that it has a cpuset that matches the allowed cpuset defined in the topology (`topology->allowed_cpuset == topology->cpuset`), and that it has a depth equal to the specified depth (`topology->depth == 2`).

Then, it recursively checks that each level in the topology has a valid depth (`hwloc__check_level`).

Finally, it checks that the object is a leaf object (`topology->flags & HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED == 0`) and that it has a root object (`topology->parent == NULL` and `topology->cpuset == topology->cpuset`).


```
/* check a whole topology structure */
void
hwloc_topology_check(struct hwloc_topology *topology)
{
  struct hwloc_obj *obj;
  hwloc_bitmap_t gp_indexes, set;
  hwloc_obj_type_t type;
  unsigned i;
  int j, depth;

  /* make sure we can use ranges to check types */

  /* hwloc__obj_type_is_{,d,i}cache() want cache types to be ordered like this */
  HWLOC_BUILD_ASSERT(HWLOC_OBJ_L2CACHE == HWLOC_OBJ_L1CACHE + 1);
  HWLOC_BUILD_ASSERT(HWLOC_OBJ_L3CACHE == HWLOC_OBJ_L2CACHE + 1);
  HWLOC_BUILD_ASSERT(HWLOC_OBJ_L4CACHE == HWLOC_OBJ_L3CACHE + 1);
  HWLOC_BUILD_ASSERT(HWLOC_OBJ_L5CACHE == HWLOC_OBJ_L4CACHE + 1);
  HWLOC_BUILD_ASSERT(HWLOC_OBJ_L1ICACHE == HWLOC_OBJ_L5CACHE + 1);
  HWLOC_BUILD_ASSERT(HWLOC_OBJ_L2ICACHE == HWLOC_OBJ_L1ICACHE + 1);
  HWLOC_BUILD_ASSERT(HWLOC_OBJ_L3ICACHE == HWLOC_OBJ_L2ICACHE + 1);

  /* hwloc__obj_type_is_normal(), hwloc__obj_type_is_memory(), hwloc__obj_type_is_io(), hwloc__obj_type_is_special()
   * and hwloc_reset_normal_type_depths()
   * want special types to be ordered like this, after all normal types.
   */
  HWLOC_BUILD_ASSERT(HWLOC_OBJ_NUMANODE   + 1 == HWLOC_OBJ_BRIDGE);
  HWLOC_BUILD_ASSERT(HWLOC_OBJ_BRIDGE     + 1 == HWLOC_OBJ_PCI_DEVICE);
  HWLOC_BUILD_ASSERT(HWLOC_OBJ_PCI_DEVICE + 1 == HWLOC_OBJ_OS_DEVICE);
  HWLOC_BUILD_ASSERT(HWLOC_OBJ_OS_DEVICE  + 1 == HWLOC_OBJ_MISC);
  HWLOC_BUILD_ASSERT(HWLOC_OBJ_MISC       + 1 == HWLOC_OBJ_MEMCACHE);
  HWLOC_BUILD_ASSERT(HWLOC_OBJ_MEMCACHE   + 1 == HWLOC_OBJ_DIE);
  HWLOC_BUILD_ASSERT(HWLOC_OBJ_DIE        + 1 == HWLOC_OBJ_TYPE_MAX);

  /* make sure order and priority arrays have the right size */
  HWLOC_BUILD_ASSERT(sizeof(obj_type_order)/sizeof(*obj_type_order) == HWLOC_OBJ_TYPE_MAX);
  HWLOC_BUILD_ASSERT(sizeof(obj_order_type)/sizeof(*obj_order_type) == HWLOC_OBJ_TYPE_MAX);
  HWLOC_BUILD_ASSERT(sizeof(obj_type_priority)/sizeof(*obj_type_priority) == HWLOC_OBJ_TYPE_MAX);

  /* make sure group are not entirely ignored */
  assert(topology->type_filter[HWLOC_OBJ_GROUP] != HWLOC_TYPE_FILTER_KEEP_ALL);

  /* make sure order arrays are coherent */
  for(type=HWLOC_OBJ_TYPE_MIN; type<HWLOC_OBJ_TYPE_MAX; type++)
    assert(obj_order_type[obj_type_order[type]] == type);
  for(i=HWLOC_OBJ_TYPE_MIN; i<HWLOC_OBJ_TYPE_MAX; i++)
    assert(obj_type_order[obj_order_type[i]] == i);

  depth = hwloc_topology_get_depth(topology);

  assert(!topology->modified);

  /* check that first level is Machine.
   * Root object cannot be ignored. And Machine can only be merged into PU,
   * but there must be a NUMA node below Machine, and it cannot be below PU.
   */
  assert(hwloc_get_depth_type(topology, 0) == HWLOC_OBJ_MACHINE);

  /* check that last level is PU and that it doesn't have memory */
  assert(hwloc_get_depth_type(topology, depth-1) == HWLOC_OBJ_PU);
  assert(hwloc_get_nbobjs_by_depth(topology, depth-1) > 0);
  for(i=0; i<hwloc_get_nbobjs_by_depth(topology, depth-1); i++) {
    obj = hwloc_get_obj_by_depth(topology, depth-1, i);
    assert(obj);
    assert(obj->type == HWLOC_OBJ_PU);
    assert(!obj->memory_first_child);
  }
  /* check that other levels are not PU or Machine */
  for(j=1; j<depth-1; j++) {
    assert(hwloc_get_depth_type(topology, j) != HWLOC_OBJ_PU);
    assert(hwloc_get_depth_type(topology, j) != HWLOC_OBJ_MACHINE);
  }

  /* check normal levels */
  for(j=0; j<depth; j++) {
    int d;
    type = hwloc_get_depth_type(topology, j);
    assert(type != HWLOC_OBJ_NUMANODE);
    assert(type != HWLOC_OBJ_MEMCACHE);
    assert(type != HWLOC_OBJ_PCI_DEVICE);
    assert(type != HWLOC_OBJ_BRIDGE);
    assert(type != HWLOC_OBJ_OS_DEVICE);
    assert(type != HWLOC_OBJ_MISC);
    d = hwloc_get_type_depth(topology, type);
    assert(d == j || d == HWLOC_TYPE_DEPTH_MULTIPLE);
  }

  /* check type depths, even if there's no such level */
  for(type=HWLOC_OBJ_TYPE_MIN; type<HWLOC_OBJ_TYPE_MAX; type++) {
    int d;
    d = hwloc_get_type_depth(topology, type);
    if (type == HWLOC_OBJ_NUMANODE) {
      assert(d == HWLOC_TYPE_DEPTH_NUMANODE);
      assert(hwloc_get_depth_type(topology, d) == HWLOC_OBJ_NUMANODE);
    } else if (type == HWLOC_OBJ_MEMCACHE) {
      assert(d == HWLOC_TYPE_DEPTH_MEMCACHE);
      assert(hwloc_get_depth_type(topology, d) == HWLOC_OBJ_MEMCACHE);
    } else if (type == HWLOC_OBJ_BRIDGE) {
      assert(d == HWLOC_TYPE_DEPTH_BRIDGE);
      assert(hwloc_get_depth_type(topology, d) == HWLOC_OBJ_BRIDGE);
    } else if (type == HWLOC_OBJ_PCI_DEVICE) {
      assert(d == HWLOC_TYPE_DEPTH_PCI_DEVICE);
      assert(hwloc_get_depth_type(topology, d) == HWLOC_OBJ_PCI_DEVICE);
    } else if (type == HWLOC_OBJ_OS_DEVICE) {
      assert(d == HWLOC_TYPE_DEPTH_OS_DEVICE);
      assert(hwloc_get_depth_type(topology, d) == HWLOC_OBJ_OS_DEVICE);
    } else if (type == HWLOC_OBJ_MISC) {
      assert(d == HWLOC_TYPE_DEPTH_MISC);
      assert(hwloc_get_depth_type(topology, d) == HWLOC_OBJ_MISC);
    } else {
      assert(d >=0 || d == HWLOC_TYPE_DEPTH_UNKNOWN || d == HWLOC_TYPE_DEPTH_MULTIPLE);
    }
  }

  /* top-level specific checks */
  assert(hwloc_get_nbobjs_by_depth(topology, 0) == 1);
  obj = hwloc_get_root_obj(topology);
  assert(obj);
  assert(!obj->parent);
  assert(obj->cpuset);
  assert(!obj->depth);

  /* check that allowed sets are larger than the main sets */
  if (topology->flags & HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED) {
    assert(hwloc_bitmap_isincluded(topology->allowed_cpuset, obj->cpuset));
    assert(hwloc_bitmap_isincluded(topology->allowed_nodeset, obj->nodeset));
  } else {
    assert(hwloc_bitmap_isequal(topology->allowed_cpuset, obj->cpuset));
    assert(hwloc_bitmap_isequal(topology->allowed_nodeset, obj->nodeset));
  }

  /* check each level */
  for(j=0; j<depth; j++)
    hwloc__check_level(topology, j, NULL, NULL);
  for(j=0; j<HWLOC_NR_SLEVELS; j++)
    hwloc__check_level(topology, HWLOC_SLEVEL_TO_DEPTH(j), topology->slevels[j].first, topology->slevels[j].last);

  /* recurse and check the tree of children, and type-specific checks */
  gp_indexes = hwloc_bitmap_alloc(); /* TODO prealloc to topology->next_gp_index */
  hwloc__check_object(topology, gp_indexes, obj);
  hwloc_bitmap_free(gp_indexes);

  /* recurse and check the nodesets of children */
  set = hwloc_bitmap_alloc();
  hwloc__check_nodesets(topology, obj, set);
  hwloc_bitmap_free(set);
}

```cpp

这段代码是一个C语言函数，名为`hwloc_topology_check`，属于`hwloc_topology`系列的函数。其作用是检查`topology`结构中所有成员的类型是否正确。

具体来说，如果`topology`结构中的成员类型都是`void`，那么函数不会做任何操作；如果有一个成员是`void`，那么函数会打印一条错误消息并退出。因此，该函数的作用是作为一种简单的错误处理机制，用于在程序运行时检查`topology`结构中成员的类型是否正确。


```
#else /* NDEBUG */

void
hwloc_topology_check(struct hwloc_topology *topology __hwloc_attribute_unused)
{
}

#endif /* NDEBUG */

```