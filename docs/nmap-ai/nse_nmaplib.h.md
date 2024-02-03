# `nmap\nse_nmaplib.h`

```cpp
#ifndef NSE_NMAPLIB
// 如果 NSE_NMAPLIB 未定义，则定义 NSE_NMAPLIB
#define NSE_NMAPLIBNAME  "nmap"
// 定义 NSE_NMAPLIBNAME 为 "nmap"
class Target;
// 声明 Target 类
class Port;
// 声明 Port 类

int luaopen_nmap(lua_State* l);
// 声明 luaopen_nmap 函数，接受 lua_State 指针参数并返回整型
#define NSE_NUM_HOSTINFO_FIELDS 17
// 定义 NSE_NUM_HOSTINFO_FIELDS 为 17
void set_hostinfo(lua_State* l, Target* currenths);
// 声明 set_hostinfo 函数，接受 lua_State 指针和 Target 指针参数，无返回值
#define NSE_NUM_PORTINFO_FIELDS 7
// 定义 NSE_NUM_PORTINFO_FIELDS 为 7
void set_portinfo(lua_State* l, const Target *target, const Port* port);
// 声明 set_portinfo 函数，接受 lua_State 指针、const Target 指针和 const Port 指针参数，无返回值

#endif
// 结束条件编译指令
```