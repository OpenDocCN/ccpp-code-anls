# `nmap\nse_nsock.h`

```cpp
# 如果未定义 NMAP_LUA_NSOCK_H，则定义 NMAP_LUA_NSOCK_H
#ifndef NMAP_LUA_NSOCK_H
# 定义 NMAP_LUA_NSOCK_H 包含 nse_lua.h 头文件
#define NMAP_LUA_NSOCK_H

#include "nse_lua.h"
# 声明 luaopen_nsock 函数，返回一个整数
LUALIB_API int luaopen_nsock (lua_State *);
# 结束条件编译指令
#endif
```