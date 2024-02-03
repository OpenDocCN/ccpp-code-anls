# `nmap\nse_ssl_cert.h`

```cpp
# 定义条件编译指令，防止头文件被重复包含
#ifndef NMAP_SSL_CERT_H
#define NMAP_SSL_CERT_H

# 声明函数 l_get_ssl_certificate，接受 Lua 状态机指针作为参数，返回整型
int l_get_ssl_certificate(lua_State *L);
# 声明函数 l_parse_ssl_certificate，接受 Lua 状态机指针作为参数，返回整型
int l_parse_ssl_certificate(lua_State *L);
# 声明函数 nse_nsock_init_ssl_cert，接受 Lua 状态机指针作为参数，无返回值
void nse_nsock_init_ssl_cert(lua_State *L);

# 结束条件编译指令
#endif
```