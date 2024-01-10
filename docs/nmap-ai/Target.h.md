# `nmap\Target.h`

```
/* $Id$ */

// 防止头文件重复包含
#ifndef TARGET_H
#define TARGET_H

// 包含必要的头文件
#include "nbase.h"
#include "libnetutil/netutil.h" /* devtype */
#ifndef NOLUA
#include "nse_main.h"
#endif
#include "portreasons.h"
#include "portlist.h"
#include "probespec.h"
#include "osscan.h"
#include "osscan2.h"
class FingerPrintResults;

// 包含 C++ 标准库头文件
#include <list>
#include <string>
#include <vector>
#include <time.h> /* time_t */

// 定义 IPv6 地址字符串长度
#ifndef INET6_ADDRSTRLEN
#define INET6_ADDRSTRLEN 46
#endif

// 定义操作系统扫描的标志
enum osscan_flags {
        OS_NOTPERF=0, OS_PERF, OS_PERF_UNREL
};

// 定义主机超时信息结构体
struct host_timeout_nfo {
  unsigned long msecs_used; /* 主机使用的毫秒数 */
  bool toclock_running; /* 时钟是否正在运行 */
  struct timeval toclock_start; /* 时钟启动时间 */
  time_t host_start, host_end; /* 主机的绝对开始和结束时间 */
};

// 定义跟踪路由的结构体
struct TracerouteHop {
  struct sockaddr_storage tag;
  bool timedout;
  std::string name;
  struct sockaddr_storage addr;
  int ttl;
  float rtt; /* 毫秒为单位的往返时间 */

  // 显示跟踪路由信息
  int display_name(char *buf, size_t len) const {
    if (name.empty())
      return Snprintf(buf, len, "%s", inet_ntop_ez(&addr, sizeof(addr)));
    else
      return Snprintf(buf, len, "%s (%s)", name.c_str(), inet_ntop_ez(&addr, sizeof(addr)));
  }
};

// 定义早期服务响应的结构体
struct EarlySvcResponse {
  probespec pspec;
  int len;
  u8 data[1];
};
/* 设置设备名称，以便可以通过deviceName()和deviceFullName()返回。普通名称可能不包括别名限定符，而完整名称可能包括它（例如"eth1:1"）。如果它们非空，则它们将覆盖存储的版本 */
  void setDeviceNames(const char *name, const char *fullname);
  const char *deviceName() const;
  const char *deviceFullName() const;

  int osscanPerformed(void) const;
  void osscanSetFlag(int flag);

  struct seq_info seq;
  int distance;
  enum dist_calc_method distance_calculation_method;
  FingerPrintResults *FPR; /* 操作系统扫描系统获取的指纹结果。 */
  PortList ports;
  std::vector<EarlySvcResponse *> earlySvcResponses;

  int weird_responses; /* 来自其他地址的回显响应，即网络广播地址 */
  int flags; /* HOST_UNKNOWN，HOST_UP或HOST_DOWN。 */
  struct timeout_info to;
  char *hostname; // 如果无法解析或未设置，则为空
  char * targetname; // 如果是命名主机，则是命令行上给定的目标主机的名称

  struct probespec traceroute_probespec;
  std::list <TracerouteHop> traceroute_hops;

  /* 如果此目标的地址来自DNS查找，则未扫描的结果地址列表（有时不止一个）。 */
  std::list<struct sockaddr_storage> unscanned_addrs;

#ifndef NOLUA
  ScriptResults scriptResults;
#endif

  state_reason_t reason; // 用于存储状态原因的变量

  /* 一个已知会收到响应的探测。在扫描过程中用于保存当前的定时 ping 探测类型 */
  probespec pingprobe;
  /* 当接收到 pingprobe 的响应时，端口或协议进入的状态 */
  int pingprobe_state;

  private:
  void FreeInternal(); // 释放对象内部分配的内存的函数
  // 根据目标的 IPv4/IPv6 地址创建一个“presentation”格式的字符串
  void GenerateTargetIPString();
  // 根据源 IPv4/IPv6 地址创建一个“presentation”格式的字符串
  void GenerateSourceIPString();
  struct sockaddr_storage targetsock, sourcesock, nexthopsock; // 目标、源、下一跳的套接字存储结构
  size_t targetsocklen, sourcesocklen, nexthopsocklen; // 目标、源、下一跳套接字的长度
  int directly_connected; // -1 = 未设置；0 = 否；1 = 是
  char targetipstring[INET6_ADDRSTRLEN]; // 目标 IP 地址的字符串表示
  char sourceipstring[INET6_ADDRSTRLEN]; // 源 IP 地址的字符串表示
  mutable char *nameIPBuf; /* 用于 NameIP(void) 函数返回的缓冲区 */
  u8 MACaddress[6], SrcMACaddress[6], NextHopMACaddress[6]; // MAC 地址
  bool MACaddress_set, SrcMACaddress_set, NextHopMACaddress_set; // MAC 地址是否设置的标志
  struct host_timeout_nfo htn; // 主机超时信息
  devtype interface_type; // 接口类型
  char devname[32]; // 设备名称
  char devfullname[32]; // 设备完整名称
  int mtu; // 最大传输单元
  /* 如果未执行操作系统检测，则为 0（OS_NOTPERF）
   * 如果执行了操作系统检测，则为 1（OS_PERF）
   * 如果执行了不可靠的操作系统检测，则为 2（OS_PERF_UNREL） */
  int osscan_flag;
};

#endif /* TARGET_H */
```