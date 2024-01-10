# `nmap\nse_libssh2.h`

```
# 如果 LIBSSH2 未定义，则定义 LIBSSH2
#ifndef LIBSSH2
# 定义 LIBSSH2
#define LIBSSH2
# 定义 LIBSSH2LIBNAME 为 "libssh2"
#define LIBSSH2LIBNAME "libssh2"
# 声明 luaopen_libssh2 函数，该函数接受一个 lua_State 指针参数，并返回一个整型值
int luaopen_libssh2 (lua_State *L);
# 结束条件编译指令
#endif
```