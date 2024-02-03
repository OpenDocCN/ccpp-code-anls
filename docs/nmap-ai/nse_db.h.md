# `nmap\nse_db.h`

```cpp
#ifndef NSE_DB
// 如果 NSE_DB 未定义，则定义 NSE_DB
#define NSE_DB

// 定义 NSE_DBLIBNAME 为 "nmapdb"
#define NSE_DBLIBNAME "nmapdb"
// 声明 luaopen_db 函数，返回一个整数，接受一个 lua_State 指针作为参数
LUALIB_API int luaopen_db (lua_State *L);

#endif
```