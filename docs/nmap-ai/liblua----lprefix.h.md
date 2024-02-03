# `nmap\liblua\lprefix.h`

```cpp
/*
** $Id: lprefix.h $
** Lua代码必须在任何其他头文件之前定义的一些定义
** 请参阅lua.h中的版权声明
*/

#ifndef lprefix_h
#define lprefix_h


/*
** 允许使用POSIX/XSI功能
*/
#if !defined(LUA_USE_C89)    /* { */

#if !defined(_XOPEN_SOURCE)
#define _XOPEN_SOURCE           600
#elif _XOPEN_SOURCE == 0
#undef _XOPEN_SOURCE  /* 使用-D_XOPEN_SOURCE=0来取消定义 */
#endif

/*
** 允许在gcc和一些其他编译器中操作大文件
*/
#if !defined(LUA_32BITS) && !defined(_FILE_OFFSET_BITS)
#define _LARGEFILE_SOURCE       1
#define _FILE_OFFSET_BITS       64
#endif

#endif                /* } */


/*
** Windows相关内容
*/
#if defined(_WIN32)    /* { */

#if !defined(_CRT_SECURE_NO_WARNINGS)
#define _CRT_SECURE_NO_WARNINGS  /* 避免关于ISO C函数的警告 */
#endif

#endif            /* } */

#endif
```