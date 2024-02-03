# `nmap\nse_lpeg.h`

```cpp
# 如果未定义 LPEG，则定义 LPEG
#ifndef LPEG
# 定义 LPEG
#define LPEG
# 定义 LPEGLIBNAME 为 "lpeg"
#define LPEGLIBNAME "lpeg"
# 声明 luaopen_lpeg 函数，返回一个整数
LUALIB_API int luaopen_lpeg (lua_State *L);
# 结束条件编译指令
#endif
```