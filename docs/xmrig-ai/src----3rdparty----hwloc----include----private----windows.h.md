# `xmrig\src\3rdparty\hwloc\include\private\windows.h`

```
/*
 * 版权声明，版权归 Université Bordeaux 所有，保留所有权利
 * 版权归 2020-2022 Inria 所有，保留所有权利
 * 请参阅顶层目录中的 COPYING 文件
 */

// 如果 _ANONYMOUS_UNION 未定义且编译器为 GCC，则定义 _ANONYMOUS_UNION 为 __extension__
#ifndef _ANONYMOUS_UNION
#ifdef __GNUC__
#define _ANONYMOUS_UNION __extension__
// 如果编译器不是 GCC，则将 _ANONYMOUS_UNION 定义为空
#else
#define _ANONYMOUS_UNION
#endif /* __GNUC__ */
#endif /* _ANONYMOUS_UNION */

// 如果 _ANONYMOUS_STRUCT 未定义且编译器为 GCC，则定义 _ANONYMOUS_STRUCT 为 __extension__
#ifndef _ANONYMOUS_STRUCT
#ifdef __GNUC__
#define _ANONYMOUS_STRUCT __extension__
// 如果编译器不是 GCC，则将 _ANONYMOUS_STRUCT 定义为空
#else
#define _ANONYMOUS_STRUCT
#endif /* __GNUC__ */
#endif /* _ANONYMOUS_STRUCT */

// 定义 DUMMYUNIONNAME 为空
#define DUMMYUNIONNAME
// 定义 DUMMYSTRUCTNAME 为空
#define DUMMYSTRUCTNAME
```