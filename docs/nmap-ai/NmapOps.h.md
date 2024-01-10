# `nmap\NmapOps.h`

```
/* $Id$ */

#ifndef NMAP_OPS_H
#define NMAP_OPS_H

#include "nmap.h" /* MAX_DECOYS */ // 包含 nmap.h 文件，用于获取最大伪装数量
#include "scan_lists.h" // 包含 scan_lists.h 文件
#include "output.h" /* LOG_NUM_FILES */ // 包含 output.h 文件，用于获取日志文件数量
#include <nbase.h> // 包含 nbase.h 文件
#include <nsock.h> // 包含 nsock.h 文件
#include <string> // 包含 string 库
#include <map> // 包含 map 库
#include <vector> // 包含 vector 库

struct FingerPrintDB; // 声明 FingerPrintDB 结构体
struct FingerMatch; // 声明 FingerMatch 结构体

class NmapOps {
 public:
  NmapOps(); // 构造函数
  ~NmapOps(); // 析构函数
  void ReInit(); // 重新初始化类为默认状态
  void setaf(int af) { addressfamily = af; } // 设置地址族
  int af() { return addressfamily; } // 获取地址族
  // no setpf() because it is based on setaf() values
  int pf(); // 获取协议族
  /* Returns 0 for success, nonzero if no source has been set or any other
     failure */
  int SourceSockAddr(struct sockaddr_storage *ss, size_t *ss_len); // 设置源地址
  /* Returns a const pointer to the source address if set, or NULL if unset. */
  const struct sockaddr_storage *SourceSockAddr() const; // 返回源地址的常量指针
  /* Note that it is OK to pass in a sockaddr_in or sockaddr_in6 casted
     to sockaddr_storage */
  void setSourceSockAddr(struct sockaddr_storage *ss, size_t ss_len); // 设置源地址
// 返回对象实例化或上次重新初始化的时间
const struct timeval *getStartTime() { return &start_time; }
// 返回自getStartTime()以来的秒数。当前时间是一个可选参数，以避免额外的gettimeofday()调用。
float TimeSinceStart(const struct timeval *now=NULL);

// 如果至少一个选择的扫描类型是TCP，则返回true
bool TCPScan();
// 如果至少一个选择的扫描类型是UDP，则返回true
bool UDPScan();
// 如果至少一个选择的扫描类型是SCTP，则返回true
bool SCTPScan();

// 如果至少一个选择的扫描类型使用原始数据包，则返回true。
// 它目前不包括诸如TCP SYN ping扫描这样的情况，这取决于用户是否是root或者是否使用IPv6。
// 在那些不一定使用RawScan的情况下，它将返回false。
bool RawScan();
// 验证选项是否合理和一致。如果不合理，函数可能会退出Nmap或进行小的调整（静默地或者向用户发出警告）。
void ValidateOptions();
// 是否为root用户
int isr00t;
// 是否有pcap函数（在Windows上可能为false）。
bool have_pcap;
// 调试级别
u8 debugging;
// 是否正在恢复
bool resuming;

// 发送数据包的选项
#define PACKET_SEND_NOPREF 1
#define PACKET_SEND_ETH_WEAK 2
#define PACKET_SEND_ETH_STRONG 4
#define PACKET_SEND_ETH 6
#define PACKET_SEND_IP_WEAK 8
#define PACKET_SEND_IP_STRONG 16
#define PACKET_SEND_IP 24
  /* 定义一个常量，表示发送 IP 数据包 */

  /* 我们应该如何发送原始 IP 数据包？Nmap通常可以使用以太网或原始 IP 套接字。哪种方法更好取决于平台和目标。_STRONG 优先级意味着每当可能时 Nmap 应该使用首选方法（显然并非总是可能的 - 在 PPP 连接上发送以太网帧是行不通的）。当另一种类型根本不起作用时，这是有用的。_WEAK 优先级意味着在另一种类型更有效的情况下，Nmap 可能会使用另一种类型。例如，即使 pref 是 SEND_IP_WEAK，Nmap 仍将对本地网络进行 ARP ping 扫描。 */
  int sendpref;
  /* 返回是否启用数据包跟踪 */
  bool packetTrace() { return (debugging >= 3)? true : pTrace;  }
  /* 返回是否启用版本跟踪 */
  bool versionTrace() { return packetTrace()? true : vTrace;  }
#ifndef NOLUA
  /* 返回是否启用脚本跟踪 */
  bool scriptTrace() { return packetTrace()? true : scripttrace; }
#ifndef NOLUA
  bool script;
  char *scriptargs;
  char *scriptargsfile;
  bool scriptversion;
  bool scripttrace;
  bool scriptupdatedb;
  bool scripthelp;
  double scripttimeout;
  /* 选择要执行的脚本 */
  void chooseScripts(char* argument);
  /* 已选择的脚本列表 */
  std::vector<std::string> chosenScripts;
#endif

  /* ip options used in build_*_raw() */
  // 用于在 build_*_raw() 中使用的 IP 选项
  u8 *ipoptions;
  int ipoptionslen;
  int ipopt_firsthop;    // offset in ipoptions where is first hop for source/strict routing
  // ipoptions 中用于源/严格路由的第一个跳转的偏移量
  int ipopt_lasthop;  // offset in ipoptions where is space for targets ip for source/strict routing
  // ipoptions 中用于源/严格路由的目标 IP 的空间的偏移量

  // Statistics Options set in nmap.cc
  // 在 nmap.cc 中设置的统计选项
  unsigned int numhosts_scanned;
  unsigned int numhosts_up;
  int numhosts_scanning;
  stype current_scantype;
  bool noninteractive;
  char *locale;

  bool release_memory;    /* suggest to release memory before quitting. used to find memory leaks. */
  // 建议在退出前释放内存，用于查找内存泄漏

 private:
  int max_os_tries;
  int max_rtt_timeout;
  int min_rtt_timeout;
  int initial_rtt_timeout;
  unsigned int max_retransmissions;
  unsigned int max_tcp_scan_delay;
  unsigned int max_udp_scan_delay;
  unsigned int max_sctp_scan_delay;
  unsigned int min_host_group_sz;
  unsigned int max_host_group_sz;
  void Initialize();
  int addressfamily; /*  Address family:  AF_INET or AF_INET6 */
  // 地址族：AF_INET 或 AF_INET6
  struct sockaddr_storage sourcesock;
  size_t sourcesocklen;
  struct timeval start_time;
  bool pTrace; // Whether packet tracing has been enabled
  bool vTrace; // Whether version tracing has been enabled
  bool xsl_stylesheet_set;
  char *xsl_stylesheet;
  u8 spoof_mac[6];
  bool spoof_mac_set;
};

#endif
```