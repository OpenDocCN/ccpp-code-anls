# `nmap\scan_engine_connect.h`

```cpp
/* $Id$ */ 
// 定义了一个宏，用于标识版本号或者其他信息

#ifndef SCAN_ENGINE_CONNECT_H
// 如果未定义 SCAN_ENGINE_CONNECT_H，则进行以下操作
#define SCAN_ENGINE_CONNECT_H
// 定义 SCAN_ENGINE_CONNECT_H

#include <nbase.h>
// 包含 nbase.h 头文件

class UltraProbe;
// 声明 UltraProbe 类
class UltraScanInfo;
// 声明 UltraScanInfo 类
class HostScanStats;
// 声明 HostScanStats 类

UltraProbe *sendConnectScanProbe(UltraScanInfo *USI, HostScanStats *hss,
                                 u16 destport, tryno_t tryno);
// 声明 sendConnectScanProbe 函数，接受 UltraScanInfo 指针、HostScanStats 指针、u16 类型的 destport 和 tryno_t 类型的 tryno，并返回 UltraProbe 指针
bool do_one_select_round(UltraScanInfo *USI, struct timeval *stime);
// 声明 do_one_select_round 函数，接受 UltraScanInfo 指针和 struct timeval 指针 stime，并返回布尔值

#endif
// 结束条件编译指令
```