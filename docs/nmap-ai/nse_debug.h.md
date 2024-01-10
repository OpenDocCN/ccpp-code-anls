# `nmap\nse_debug.h`

```
#ifndef NSE_DEBUG
// 如果 NSE_DEBUG 未定义，则定义 NSE_DEBUG

#define NSE_DEBUG
// 定义 NSE_DEBUG

void value_dump(lua_State *L, int i, int depth_limit);
// 声明 value_dump 函数，用于打印 Lua 值的调试信息

void stack_dump(lua_State *L);
// 声明 stack_dump 函数，用于打印 Lua 栈的调试信息

void lua_state_dump(lua_State *L);
// 声明 lua_state_dump 函数，用于打印 Lua 状态的调试信息

#endif
// 结束条件编译指令
```