# `nmap\nse_zlib.h`

```
# 如果未定义 ZLIB，则定义 ZLIB
#ifndef ZLIB
# 定义 NSE_ZLIBNAME 为 "zlib"
#define NSE_ZLIBNAME "zlib"
# 声明导出函数 luaopen_zlib，返回类型为 int，参数为 lua_State 指针
LUALIB_API int luaopen_zlib(lua_State *L);
# 结束条件编译指令
#endif
```