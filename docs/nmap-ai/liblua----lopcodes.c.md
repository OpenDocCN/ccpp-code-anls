# `nmap\liblua\lopcodes.c`

```
/*
** $Id: lopcodes.c $
** Opcodes for Lua virtual machine
** See Copyright Notice in lua.h
*/

// 定义宏，用于标识 lopcodes.c 文件
#define lopcodes_c
// 定义宏，用于标识 LUA_CORE
#define LUA_CORE

// 包含预编译的头文件 lprefix.h
#include "lprefix.h"

// 包含自定义的头文件 lopcodes.h
#include "lopcodes.h"

// 定义操作码的顺序
/* ORDER OP */

// 定义 Lua 虚拟机操作码的模式数组
LUAI_DDEF const lu_byte luaP_opmodes[NUM_OPCODES] = {
};
```