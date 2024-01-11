# `xmrig\src\3rdparty\hwloc\include\hwloc\autogen\config.h`

```
/* 版权声明，版权归 CNRS、Inria、Université Bordeaux 和 Cisco Systems, Inc. 所有，保留所有权利 */
/* 配置文件 */

#ifndef HWLOC_CONFIG_H
#define HWLOC_CONFIG_H

/* 定义 HWLOC 版本号 */
#define HWLOC_VERSION "2.9.0"
#define HWLOC_VERSION_MAJOR 2
#define HWLOC_VERSION_MINOR 9
#define HWLOC_VERSION_RELEASE 0
#define HWLOC_VERSION_GREEK ""

/* 定义空宏 __hwloc_restrict 和内联函数宏 __hwloc_inline */
#define __hwloc_restrict
#define __hwloc_inline __inline

/* 定义一些属性宏 */
#define __hwloc_attribute_unused
#define __hwloc_attribute_malloc
#define __hwloc_attribute_const
#define __hwloc_attribute_pure
#define __hwloc_attribute_deprecated
#define __hwloc_attribute_may_alias
#define __hwloc_attribute_warn_unused_result

/* 如果有 'windows.h' 头文件，则定义为 1 */
#define HWLOC_HAVE_WINDOWS_H 1
#define hwloc_pid_t HANDLE
#define hwloc_thread_t HANDLE

/* 包含必要的 Windows 头文件 */
#include <windows.h>
#include <BaseTsd.h>
typedef DWORDLONG hwloc_uint64_t;

/* 根据动态链接或静态链接定义 HWLOC_DECLSPEC */
#if defined( _USRDLL ) /* dynamic linkage */
#if defined( DECLSPEC_EXPORTS )
#define HWLOC_DECLSPEC __declspec(dllexport)
#else
#define HWLOC_DECLSPEC __declspec(dllimport)
#endif
#else /* static linkage */
#define HWLOC_DECLSPEC
#endif

/* 是否需要重新定义所有 hwloc 公共符号 */
#define HWLOC_SYM_TRANSFORM 0

/* hwloc 符号前缀 */
#define HWLOC_SYM_PREFIX hwloc_

/* 大写形式的 hwloc 符号前缀 */
#define HWLOC_SYM_PREFIX_CAPS HWLOC_

#endif /* HWLOC_CONFIG_H */
```