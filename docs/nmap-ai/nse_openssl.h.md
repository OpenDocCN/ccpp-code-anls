# `nmap\nse_openssl.h`

```cpp
#ifndef OPENSSLLIB
// 如果未定义 OPENSSLLIB，则定义 OPENSSLLIB
#define OPENSSLLIBNAME "openssl"
// 定义 OPENSSLLIBNAME 为 "openssl"
LUALIB_API int luaopen_openssl(lua_State *L);
// 声明 luaopen_openssl 函数
#include <openssl/bn.h>
// 包含 openssl/bn.h 头文件
int nse_pushbn( lua_State *L, BIGNUM *num, bool should_free);
// 声明 nse_pushbn 函数
#endif
// 结束条件编译指令
```