# `nmap\liblua\lualib.h`

```
/*
** $Id: lualib.h $
** Lua标准库
** 请参阅lua.h中的版权声明
*/

#ifndef lualib_h
#define lualib_h

#include "lua.h"

/* 环境变量名称的版本后缀 */
#define LUA_VERSUFFIX          "_" LUA_VERSION_MAJOR "_" LUA_VERSION_MINOR

// 打开基础库
LUAMOD_API int (luaopen_base) (lua_State *L);

#define LUA_COLIBNAME    "coroutine"
// 打开协程库
LUAMOD_API int (luaopen_coroutine) (lua_State *L);

#define LUA_TABLIBNAME    "table"
// 打开表库
LUAMOD_API int (luaopen_table) (lua_State *L);

#define LUA_IOLIBNAME    "io"
// 打开IO库
LUAMOD_API int (luaopen_io) (lua_State *L);

#define LUA_OSLIBNAME    "os"
// 打开OS库
LUAMOD_API int (luaopen_os) (lua_State *L);

#define LUA_STRLIBNAME    "string"
// 打开字符串库
LUAMOD_API int (luaopen_string) (lua_State *L);

#define LUA_UTF8LIBNAME    "utf8"
// 打开UTF-8库
LUAMOD_API int (luaopen_utf8) (lua_State *L);

#define LUA_MATHLIBNAME    "math"
// 打开数学库
LUAMOD_API int (luaopen_math) (lua_State *L);

#define LUA_DBLIBNAME    "debug"
// 打开调试库
LUAMOD_API int (luaopen_debug) (lua_State *L);

#define LUA_LOADLIBNAME    "package"
// 打开包管理库
LUAMOD_API int (luaopen_package) (lua_State *L);

/* 打开所有先前的库 */
LUALIB_API void (luaL_openlibs) (lua_State *L);

#endif
```