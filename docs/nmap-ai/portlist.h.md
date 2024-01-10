# `nmap\portlist.h`

```
/* $Id$ */

// 防止头文件被重复包含
#ifndef PORTLIST_H
#define PORTLIST_H

// 包含必要的头文件
#include "nbase.h"
#ifndef NOLUA
#include "nse_main.h"
#endif

#include "portreasons.h"

#include <vector>

// 定义端口状态常量
/* port states */
#define PORT_UNKNOWN 0
#define PORT_CLOSED 1
#define PORT_OPEN 2
#define PORT_FILTERED 3
#define PORT_TESTING 4
#define PORT_FRESH 5
#define PORT_UNFILTERED 6
#define PORT_OPENFILTERED 7 /* Like udp/fin/xmas/null/ipproto scan with no response */
#define PORT_CLOSEDFILTERED 8 /* Idle scan */
#define PORT_HIGHEST_STATE 9 /* ***IMPORTANT -- BUMP THIS UP WHEN STATES ARE
                                ADDED *** */
// 将端口状态转换为字符串
const char *statenum2str(int state);

// 定义协议类型常量
#define TCPANDUDPANDSCTP IPPROTO_MAX
#define UDPANDSCTP (IPPROTO_MAX + 1)

// 定义服务探测状态枚举
enum serviceprobestate {
  PROBESTATE_INITIAL=1, // No probes started yet
  PROBESTATE_NULLPROBE, // Is working on the NULL Probe
  PROBESTATE_MATCHINGPROBES, // Is doing matching probe(s)
  PROBESTATE_NONMATCHINGPROBES, // The above failed, is checking nonmatches
  PROBESTATE_FINISHED_HARDMATCHED, // Yay!  Found a match
  PROBESTATE_FINISHED_SOFTMATCHED, // Well, a soft match anyway
  PROBESTATE_FINISHED_NOMATCH, // D'oh!  Failed to find the service.
  PROBESTATE_FINISHED_TCPWRAPPED, // We think the port is blocked via tcpwrappers
  PROBESTATE_EXCLUDED, // The port has been excluded from the scan
  PROBESTATE_INCOMPLETE // failed to complete (error, host timeout, etc.)
};

// 定义服务探测类型枚举
enum service_detection_type { SERVICE_DETECTION_TABLE, SERVICE_DETECTION_PROBED };

// 定义服务隧道类型枚举
enum service_tunnel_type { SERVICE_TUNNEL_NONE, SERVICE_TUNNEL_SSL };

// 将一些常见的TCP端口移动到端口列表的开头，以加快某些扫描的速度
// 在进行端口随机化之后，这可以防止端口总是以相同的顺序出现
void random_port_cheat(u16 *ports, int portcount);
// 定义了一个结构体 serviceDeductions
struct serviceDeductions {
  // 构造函数
  serviceDeductions();
  // 释放需要释放的字符串，并将所有指针设置为 null
  void erase();
  // 填充完整版本字符串
  void populateFullVersionString(char *buf, size_t n) const;

  const char *name; // 如果无法确定，则为 NULL
  // 置信度是一个从 0（最不确定）到 10（最确定）的数字，表示服务检测的准确程度
  int name_confidence;
  // 如果我们无法确定，这 6 个中的任何一个都可以为 NULL
  char *product;
  char *version;
  char *extrainfo;
  char *hostname;
  char *ostype;
  char *devicetype;
  std::vector<char *> cpe;
  // SERVICE_TUNNEL_NONE 或 SERVICE_TUNNEL_SSL
  enum service_tunnel_type service_tunnel;
  // 如果应该给用户一个服务指纹来提交，这里就是它。否则为 NULL
  char *service_fp;
  enum service_detection_type dtype; // 上面的定义
};

// 定义了一个类 Port
class Port {
 friend class PortList;

 public:
  // 构造函数
  Port();
  // 释放服务，如果 del_service 为真
  void freeService(bool del_service);
  // 释放脚本结果
  void freeScriptResults(void);
  // 获取 Nmap 服务名称
  void getNmapServiceName(char *namebuf, int buflen) const;

  u16 portno;
  u8 proto;
  u8 state;
  state_reason_t reason;

#ifndef NOLUA
  ScriptResults scriptResults;
#endif

 private:
  /* This is allocated only on demand by PortList::setServiceProbeResults
     to save memory for the many closed or filtered ports that don't need it. */
  serviceDeductions *service;
};

/* 需要的枚举来访问一些数组。这些值
 * 不应该直接使用。使用 INPROTO2PORTLISTPROTO 宏 */
enum portlist_proto {    // PortList Protocols
  PORTLIST_PROTO_TCP    = 0,
  PORTLIST_PROTO_UDP    = 1,
  PORTLIST_PROTO_SCTP    = 2,
  PORTLIST_PROTO_IP    = 3,
  PORTLIST_PROTO_MAX    = 4
};

#ifndef NOLUA
  // 添加脚本结果
  void addScriptResult(u16 portno, int protocol, ScriptResult *sr);
};

#endif
```