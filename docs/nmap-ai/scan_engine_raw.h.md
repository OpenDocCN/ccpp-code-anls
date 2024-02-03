# `nmap\scan_engine_raw.h`

```cpp
/* $Id$ */

// 防止头文件被重复包含
#ifndef SCAN_ENGINE_RAW_H
#define SCAN_ENGINE_RAW_H

// 包含必要的头文件
#include <nbase.h>
class UltraProbe;
class UltraScanInfo;
class HostScanStats;
#include <vector>

// 声明函数
class Target;

int get_ping_pcap_result(UltraScanInfo *USI, struct timeval *stime);
void begin_sniffer(UltraScanInfo *USI, std::vector<Target *> &Targets);
UltraProbe *sendArpScanProbe(UltraScanInfo *USI, HostScanStats *hss,
                             tryno_t tryno);
UltraProbe *sendNDScanProbe(UltraScanInfo *USI, HostScanStats *hss,
                            tryno_t tryno);
UltraProbe *sendIPScanProbe(UltraScanInfo *USI, HostScanStats *hss,
                            const probespec *pspec, tryno_t tryno);
bool get_arp_result(UltraScanInfo *USI, struct timeval *stime);
bool get_ns_result(UltraScanInfo *USI, struct timeval *stime);
bool get_pcap_result(UltraScanInfo *USI, struct timeval *stime);

// 结束防止头文件被重复包含
#endif
```