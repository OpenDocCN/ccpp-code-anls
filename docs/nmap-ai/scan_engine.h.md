# `nmap\scan_engine.h`

```
/* $Id$ */

#ifndef SCAN_ENGINE_H
#define SCAN_ENGINE_H

#include "scan_lists.h"
#include "probespec.h"

#include <dnet.h>

#include "timing.h"

#include <pcap.h>
#include <list>
#include <vector>
#include <set>
#include <algorithm>
class Target;

/* 3rd generation Nmap scanning function.  Handles most Nmap port scan types */
void ultra_scan(std::vector<Target *> &Targets, const struct scan_lists *ports,
                stype scantype, struct timeout_info *to = NULL);

/* Determines an ideal number of hosts to be scanned (port scan, os
   scan, version detection, etc.) in parallel after the ping scan is
   completed.  This is a balance between efficiency (more hosts in
   parallel often reduces scan time per host) and results latency (you
   need to wait for all hosts to finish before Nmap can spit out the
   results).  Memory consumption usually also increases with the
   number of hosts scanned in parallel, though rarely to significant
   levels. */
int determineScanGroupSize(int hosts_scanned_so_far,
                           const struct scan_lists *ports);

class UltraScanInfo;

struct ppkt { /* Beginning of ICMP Echo/Timestamp header         */
  u8 type;
  u8 code;
  u16 checksum;
  u16 id;
  u16 seq;
};

class ConnectProbe {
public:
  ConnectProbe();
  ~ConnectProbe();
  int sd; /* Socket descriptor used for connection.  -1 if not valid. */
};

struct IPExtraProbeData_icmp {
  u16 ident;
};

struct IPExtraProbeData_tcp {
  u16 sport;
  u32 seq; /* host byte order (like the other fields */
};

struct IPExtraProbeData_udp {
  u16 sport;
};

struct IPExtraProbeData_sctp {
  u16 sport;
  u32 vtag;
};

struct IPExtraProbeData {
  u32 ipid; /* host byte order */
  union {
    struct IPExtraProbeData_icmp icmp;
    struct IPExtraProbeData_tcp tcp;
    struct IPExtraProbeData_udp udp;
    struct IPExtraProbeData_sctp sctp;
  } pd;
};
union _tryno_u {
  struct {
  u8 isPing : 1; // 是否为 ping，而不是扫描探针？
  u8 seqnum : 7; // 序列号，范围为 0-127
  } fields;
  u8 opaque;
};
typedef union _tryno_u tryno_t;

/* 目前至少，我将像使用结构体一样直接访问所有数据成员 */
class UltraProbe {
public:
  UltraProbe();
  ~UltraProbe();
  enum UPType { UP_UNSET, UP_IP, UP_CONNECT, UP_ARP, UP_ND } type; /* 这个探针的类型 */

  /* 将此 UltraProbe 设置为 UP_IP 类型，并创建并初始化内部的 IPProbe。由于 pspec.type 在只有 ippacket 的情况下是模棱两可的（例如，一个 TCP 包可以是 PS_PROTO 或 PS_TCP），因此设置 IP 时需要相关的 probespec */
  void setIP(const u8 *ippacket, u32 iplen, const probespec *pspec);
  /* 将此 UltraProbe 设置为 UP_CONNECT 类型，准备连接到给定的端口号 */
  void setConnect(u16 portno);
  /* 传递一个 ARP 包，包括以太网头。必须是 42 字节 */
  void setARP(const u8 *arppkt, u32 arplen);
  void setND(const u8 *ndpkt, u32 ndlen);
  // 以下 4 个访问器都以主机字节顺序返回
  // 如果是 TCP、UDP 或 SCTP，则返回源端口号
  u16 sport() const;
  // 如果是 TCP、UDP 或 SCTP，则返回目标端口号
  u16 dport() const;
  u32 ipid() const {
    return probes.IP.ipid;
  }
  u16 icmpid() const; // 如果协议是 ICMP，则返回 ICMP 标识
  u32 tcpseq() const; // 如果协议是 TCP，则返回 TCP 序列号
  u32 sctpvtag() const; // 如果协议是 SCTP，则返回 SCTP vtag
  /* 协议号，例如 IPPROTO_TCP、IPPROTO_UDP 等 */
  u8 protocol() const {
    return mypspec.proto;
  }
  ConnectProbe *CP() const {
    return probes.CP;  // 如果类型为 UP_CONNECT
  }
  // Arpprobe 被移除，因为没有使用。
  //  ArpProbe *AP() { return probes.AP; } // 如果是 UP_ARP
  // 通过读取 probespec 的适当字段，返回协议号，例如 IPPROTO_TCP 或 IPPROTO_UDP。

  /* 获取有关探针的一般详细信息 */
  const probespec *pspec() const {
  // 返回 mypspec 的引用
  return &mypspec;
}

/* 如果给定的 tryno 与此探测匹配，则返回 true */
bool check_tryno(u8 tryno) const {
  return tryno == this->tryno.opaque;
}

/* 从数据包中检查协议/端口匹配的辅助函数 */
bool check_proto_port(u8 proto, u16 sport_or_icmpid, u16 dport) const;

/* tryno/pingseq，取决于这个探测是 ping 还是扫描探测 */
tryno_t tryno; // 这个探测的尝试（重传）编号
/* 如果为 true，由于超时，探测被认为不再活动，但可能会保留一段时间，以防回复延迟到来 */
bool timedout;
/* 由于数据包发送速率限制，数据包可能在重新发送之前被超时一段时间 */
bool retransmitted;

struct timeval sent;
/* 如果这是一个重传（tryno > 0），则记录上一个探测发送的时间 */
struct timeval prevSent;
bool isPing() const {
  return tryno.fields.isPing;
}
u8 get_tryno() const {
  return tryno.fields.seqnum;
}
// 定义了一个私有成员变量 mypspec，用于存储探测规范信息
// 该成员变量将由适当的 set* 函数填充
private:
  probespec mypspec; /* Filled in by the appropriate set* function */
  // 定义了一个联合体 probes，用于存储不同类型的探测数据
  union {
    IPExtraProbeData IP;
    ConnectProbe *CP;
    //    ArpProbe *AP;
  } probes;
};

// 定义了一个全局的 ConnectScanInfo 类，用于存储连接扫描的全局信息
class ConnectScanInfo {
public:
  ConnectScanInfo();
  ~ConnectScanInfo();

  // 监视一个套接字描述符（将其添加到 fd_sets 和 maxValidSD 中）
  // 如果套接字描述符之前不存在于列表中，则返回 true；如果尝试监视已经被监视的套接字描述符，则返回 false
  bool watchSD(int sd);

  // 从 fd_sets 和 maxValidSD 中清除套接字描述符
  // 如果套接字描述符在列表中，则返回 true；如果尝试清除不存在于列表中的套接字描述符，则返回 false
  bool clearSD(int sd);

  // 尝试获取一个适合于 select() 的套接字，如果成功则返回 true；如果失败则返回 false
  bool sendOK();
  int maxValidSD; // fd_sets 中的最大套接字描述符
  fd_set fds_read; // 读取套接字描述符的集合
  fd_set fds_write; // 写入套接字描述符的集合
  fd_set fds_except; // 异常套接字描述符的集合
  int numSDs; // 正在监视的套接字描述符的数量
  int getSocket(); // 获取套接字描述符
private:
  int nextSD; // 下一个套接字描述符
  int maxSocketsAllowed; // 一次性创建的套接字描述符数量上限
};

class HostScanStats;

// GroupScanStats 类，用于存储整个目标组的 ultra_scan() 统计信息
class GroupScanStats {
};

// send_delay_nfo 结构体，用于存储发送延迟的相关信息
struct send_delay_nfo {
  unsigned int delayms; // 两次探测之间的延迟时间（毫秒）
  // 自上次延迟更改以来的成功和丢弃的探测数量
  unsigned int goodRespSinceDelayChanged;
  unsigned int droppedRespSinceDelayChanged;
  struct timeval last_boost; // 最近一次增加 delayms 的时间，初始化为创建时间
};

// 用于测试速率限制，发送特定重传次数的第一个数据包之间存在延迟
// 这些值有助于跟踪延迟
// 定义了一个结构体 rate_limit_detection_nfo，用于存储速率限制检测相关的信息
struct rate_limit_detection_nfo {
  unsigned int max_tryno_sent; /* 已发送的最大尝试次数（从0开始） */
  bool rld_waiting; /* 是否由于速率限制而当前正在等待？ */
  struct timeval rld_waittime; /* 如果由于速率限制在等待，那么何时可以发送？ */
};

/* 适用于组中单个目标主机的 ultra_scan() 统计信息 */
class HostScanStats {
    // 必须适应7位：tryno.fields.seqnum
    nxtpseq = (nxtpseq + 1) % 0x80;
    return nxtpseq;
  }
  /* 这是产生有用结果（如端口状态更改）的最高尝试次数 */
  unsigned int max_successful_tryno;
  int ports_finished; /* 已确定的该主机的端口数量 */
  int numprobes_sent; /* 发送到该主机的端口探测数量（不包括ping，但包括重传） */
  /* 提高该主机的扫描延迟，通常是因为检测到太多数据包丢失 */
  void boostScanDelay();
  struct send_delay_nfo sdn;
  struct rate_limit_detection_nfo rld;

private:
  u8 nxtpseq; /* 下一个扫描ping序列号 */
};

/* 一些特定于 ultra_scan 的额外性能调优参数 */
struct ultra_scan_performance_vars : public scan_performance_vars {
  /* 当成功的ping响应返回时，它被视为这么多“正常”响应，因为需要ping的事实意味着我们没有得到太多输入 */
  int ping_magnifier;
  /* 如果从目标主机没有收到响应，尝试发送扫描ping的时间间隔 */
  int pingtime;
  unsigned int tryno_cap; /* 允许的最大尝试次数（从零开始） */

  void init();
};

struct HssPredicate {
public:
  int operator() (const HostScanStats *lhs, const HostScanStats *rhs) const;
  static struct sockaddr_storage *ss;
};

class UltraScanInfo {
public:
  UltraScanInfo();
  UltraScanInfo(std::vector<Target *> &Targets, const struct scan_lists *pts, stype scantype) {
  // 初始化函数，用于初始化目标、扫描类型等参数
  Init(Targets, pts, scantype);
}
// 析构函数
~UltraScanInfo();
/* 如果使用默认构造函数创建对象，必须调用Init */
void Init(std::vector<Target *> &Targets, const struct scan_lists *pts, stype scantp);

// 返回每个主机的探测数量
unsigned int numProbesPerHost() const;

/* 与组统计信息和每个不完整主机的hstats协商，确定是否可以立即发送任何探测。
   如果可以立即发送，则返回true。如果tv非空，则填充下次可以发送探测的时间（如果函数返回true，则为现在） */
bool sendOK(struct timeval *tv) const;
stype scantype;
bool tcp_scan; /* scantype是TCP扫描的一种类型 */
bool udp_scan;
bool sctp_scan; /* scantype是SCTP扫描的一种类型 */
bool prot_scan;
bool ping_scan; /* 包括传统ping扫描和arp扫描 */
bool ping_scan_arp; /* 仅包括arp ping扫描 */
bool ping_scan_nd; /* 仅包括ND ping扫描 */
bool noresp_open_scan; /* 无响应是否意味着端口开放 */

/* massping状态 */
/* 如果ping_scan为true（除非ping_scan_arp也为true），则这是要使用的ping技术集（ICMP、原始ICMP、TCP连接、原始TCP或原始UDP） */
struct {
    // 定义一个结构体，包含用于扫描的不同类型的标志位
    struct {
      unsigned int rawicmpscan: 1,  // 是否进行原始 ICMP 扫描
        connecttcpscan: 1,  // 是否进行连接式 TCP 扫描
        rawtcpscan: 1,  // 是否进行原始 TCP 扫描
        rawudpscan: 1,  // 是否进行原始 UDP 扫描
        rawsctpscan: 1,  // 是否进行原始 SCTP 扫描
        rawprotoscan: 1;  // 是否进行原始协议扫描
    } ptech;

    // 声明一个函数，用于检查是否进行了原始扫描
    bool isRawScan() const;

    // 用于保存时间信息，可以用于避免调用 gettimeofday()
    struct timeval now;

    // 用于保存组扫描的统计信息
    GroupScanStats *gstats;

    // 用于保存超高性能扫描的变量
    struct ultra_scan_performance_vars perf;

    // 一个循环缓冲区，保存未完成扫描的主机
    HostScanStats *nextIncompleteHost();

    // 移除已完成扫描的主机，并移除已超过生命周期的主机
    int removeCompletedHosts();

    // 在未完成和已完成的列表中根据 IP 地址查找主机扫描统计信息
    HostScanStats *findHost(struct sockaddr_storage *ss) const;

    // 获取扫描完成的比例
    double getCompletionFraction() const;

    // 获取未完成主机的数量
    unsigned int numIncompleteHosts() const {
      return incompleteHosts.size();
    }

    // 检查未完成主机列表是否为空
    bool incompleteHostsEmpty() const {
      return incompleteHosts.empty();
    }

    // 检查未完成主机的数量是否小于指定值
    bool numIncompleteHostsLessThan(unsigned int n) const;

    // 获取初始主机的数量
    unsigned int numInitialHosts() const {
    // 返回初始目标数量
    return numInitialTargets;
  }

  // 记录整体速率
  void log_overall_rates(int logt) const;
  // 记录当前速率
  void log_current_rates(int logt, bool update = true);

  /* 任何修改 incompleteHosts（从中删除元素）的函数可能需要操作 nextI */
  // 未完成主机的多重集合
  std::multiset<HostScanStats *, HssPredicate> incompleteHosts;
  /* 主机在完成后从 incompleteHosts 移动到 completedHosts。我们保留它们是因为有时候响应会在我们认为主机已完成之后非常晚地返回。 */
  // 已完成主机的多重集合
  std::multiset<HostScanStats *, HssPredicate> completedHosts;
  /* 我们上次遍历 completedHosts 来移除主机的时间 */
  // 上次移除已完成主机的时间
  struct timeval lastCompletedHostRemoval;

  // 扫描进度计量器
  ScanProgressMeter *SPM;
  // 发送速率计量器
  PacketRateMeter send_rate_meter;
  // 端口列表
  const struct scan_lists *ports;
  // 原始套接字描述符
  int rawsd;
  // 数据包捕获描述符
  pcap_t *pd;
  // 以太网描述符
  eth_t *ethsd;
  // 序列号掩码，用于在序列号中编码值。在 UltraScanInfo::Init() 中随机设置
  u32 seqmask;
  // 基础端口
  u16 base_port;
  // 返回源套接字地址
  const struct sockaddr_storage *SourceSockAddr() const { return &sourceSockAddr; }
// 用于存储初始目标数量的变量
unsigned int numInitialTargets;
// 用于迭代访问主机扫描统计信息的多重集合迭代器
std::multiset<HostScanStats *, HssPredicate>::iterator nextI;
// 保存源地址信息的结构体
struct sockaddr_storage sourceSockAddr;
/* 在源端口中编码每个探测的尝试次数等探测信息，用于使用端口的协议（当 o.magic_port_set 为真时，将使用请求的源端口）。
   尝试次数被编码为相对于 base_port 的偏移量，base_port 是一个基本的源端口号（参见 sport_encode 和 sport_decode）。
   为了避免将上一次 ultra_scan 调用的延迟响应错误地解释为当前调用中相同端口的响应，每次运行 ultra_scan 时，我们都会将 base_port 增加一个足够大的偏移量。 */
/* 必须选择 base_port，以便在不超过 16 位的情况下添加一个 8 位值（tryno）。我们对最大的素数 N 进行模增量，使得 33000 + N + 256 < 65536，这确保没有重叠的循环。 */
// 最接近不超过 65536 - 256 - 33000 的素数：
#define PRIME_32K 32261
/* 将 base_port 更改为一个安全端口范围内的新数字，这个范围不太可能与附近的过去或将来的 ultra_scan 调用发生冲突。 */
static u16 increment_base_port() {
  static u16 g_base_port = 33000 + get_random_uint() % PRIME_32K;
  g_base_port = 33000 + (g_base_port - 33000 + 256) % PRIME_32K;
  return g_base_port;
}
/* 是否为整个组或单个主机存储时间统计信息的枚举类型 */
enum ultra_timing_type { TIMING_HOST, TIMING_GROUP };

// 将扫描类型转换为ASCII字符串
const char *pspectype2ascii(int type);

// 更新端口探测状态
void ultrascan_port_probe_update(UltraScanInfo *USI, HostScanStats *hss,
                                 std::list<UltraProbe *>::iterator probeI,
                                 int newstate, struct timeval *rcvdtime,
                                 bool adjust_timing_hint = true);

// 更新主机探测状态
void ultrascan_host_probe_update(UltraScanInfo *USI, HostScanStats *hss,
                                        std::list<UltraProbe *>::iterator probeI,
                                        int newstate, struct timeval *rcvdtime,
                                        bool adjust_timing_hint = true);

// 更新ping探测状态
void ultrascan_ping_update(UltraScanInfo *USI, HostScanStats *hss,
                                  std::list<UltraProbe *>::iterator probeI,
                                  struct timeval *rcvdtime,
                                  bool adjust_timing = true);
#endif /* SCAN_ENGINE_H */
```