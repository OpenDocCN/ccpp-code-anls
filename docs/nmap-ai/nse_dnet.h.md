# `nmap\nse_dnet.h`

```
# 如果未定义 NMAP_LUA_DNET_H，则定义 NMAP_LUA_DNET_H
#ifndef NMAP_LUA_DNET_H
# 定义 NMAP_LUA_DNET_H
#define NMAP_LUA_DNET_H

# 导出 luaopen_dnet 函数，使其在 Lua 中可用
LUALIB_API int luaopen_dnet (lua_State *L);

# 结束条件编译指令
#endif
```