# `nmap\liblua\lctype.c`

```
/*
** $Id: lctype.c $
** 'ctype' functions for Lua
** See Copyright Notice in lua.h
*/

// 定义 lctype.c 文件
#define lctype_c
// 定义 LUA_CORE
#define LUA_CORE

// 包含 lprefix.h 文件
#include "lprefix.h"

// 包含 lctype.h 文件
#include "lctype.h"

// 如果不使用 CTYPE
#if !LUA_USE_CTYPE    /* { */

// 包含 limits.h 文件
#include <limits.h>

// 如果定义了 LUA_UCID
#if defined (LUA_UCID)        /* accept UniCode IDentifiers? */
// 将所有非 ASCII 码点视为字母
#define NONA        0x01
#else
// 默认情况下将非 ASCII 码点视为非字母
#define NONA        0x00
#endif

// 结束条件
};

#endif            /* } */
```