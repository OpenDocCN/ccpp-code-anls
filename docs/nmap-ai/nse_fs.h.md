# `nmap\nse_fs.h`

```cpp
#ifndef NSE_FS
// 如果 NSE_FS 未定义，则定义 NSE_FS
#define NSE_FS
// 定义 LFSLIBNAME 为 "lfs"
#define LFSLIBNAME "lfs"
// 声明 luaopen_lfs 函数，返回一个整数，接受一个 lua_State 指针参数
LUALIB_API int luaopen_lfs (lua_State *L);
// 结束条件编译指令
#endif
```