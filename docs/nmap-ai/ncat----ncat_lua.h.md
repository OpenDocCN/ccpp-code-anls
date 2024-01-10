# `nmap\ncat\ncat_lua.h`

```
/* $Id$ */

#ifndef _NCAT_LUA_H
#define _NCAT_LUA_H

#include "ncat_config.h"

#ifdef __cplusplus
extern "C" {
#endif

#ifdef HAVE_LUA5_4_LUA_H
  #include <lua5.4/lua.h>
  #include <lua5.4/lauxlib.h>
  #include <lua5.4/lualib.h>
#elif defined HAVE_LUA_5_4_LUA_H
  #include <lua/5.4/lua.h>
  #include <lua/5.4/lauxlib.h>
  #include <lua/5.4/lualib.h>
#elif defined HAVE_LUA_H || defined LUA_INCLUDED
  #include <lua.h>
  #include <lauxlib.h>
  #include <lualib.h>
#elif defined HAVE_LUA_LUA_H
  #include <lua/lua.h>
  #include <lua/lauxlib.h>
  #include <lua/lualib.h>
#endif

#ifdef __cplusplus
}
#endif

// 初始化 Lua 环境
void lua_setup(void);
// 运行 Lua 脚本
void lua_run(void);

#endif
```