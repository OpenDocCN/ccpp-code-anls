# `nmap\FPEngine.h`

```cpp
/* $Id$ */

#ifndef __FPENGINE_H__
#define __FPENGINE_H__ 1

#include "nsock.h"
#include <vector>
#include "libnetutil/npacket.h"

/* Mention some classes here so we don't have to place the declarations in
 * the right order (otherwise the compiler complains). */
class FPHost;
class FPHost6;
class FPProbe;

class Target;
class FingerPrintResultsIPv6;
struct FingerMatch;

/******************************************************************************
 * CONSTANT DEFINITIONS                                                       *
 ******************************************************************************/

#define NELEMS(a) (sizeof(a) / sizeof((a)[0]))  // 定义一个宏，用于计算数组元素个数

#define NUM_FP_PROBES_IPv6_TCP    13  // 定义IPv6 TCP探测的数量
#define NUM_FP_PROBES_IPv6_ICMPv6 4   // 定义IPv6 ICMPv6探测的数量
#define NUM_FP_PROBES_IPv6_UDP    1   // 定义IPv6 UDP探测的数量
/* Total number of IPv6 OS detection probes. */
#define NUM_FP_PROBES_IPv6 (NUM_FP_PROBES_IPv6_TCP+NUM_FP_PROBES_IPv6_ICMPv6+NUM_FP_PROBES_IPv6_UDP)  // 计算IPv6 OS检测探测的总数量

/* Even with a successful classification, we may not consider a match good if it
   is too different from other members of the class. */
#define FP_NOVELTY_THRESHOLD 15.0  // 定义新颖性阈值

const unsigned int OSDETECT_FLOW_LABEL = 0x12345;  // 定义OS检测流标签

/* Number of timed probes for IPv6 OS scan. This is, the number of probes that
 * have specific timing requirements and need to be processed together. This
 * are the probes that are sent 100ms apart. */
#define NUM_FP_TIMEDPROBES_IPv6 6  // 定义IPv6 OS扫描的定时探测数量

/* Initial congestion window. It is set to the number of timed probes because
 * hosts need to be able to schedule all of them at once. */
#define OSSCAN_INITIAL_CWND (NUM_FP_TIMEDPROBES_IPv6)  // 定义初始拥塞窗口大小

/* Initial Slow Start threshold. It is set to four times the initial CWND. */
#define OSSCAN_INITIAL_SSTHRESH (4 * OSSCAN_INITIAL_CWND)  // 定义初始慢启动阈值

/* Host group size is the number of osscan hosts that are processed in parallel.
 * Note that this osscan engine always keeps a working group of this many hosts.
 * in other words, if one host in the group finishes, another is added to it
 * dynamically. */
# 定义OSSCAN_GROUP_SIZE为10，表示OSSCAN组的大小

/* 初始重传超时时间。这是我们在重新传输原始探测包之前等待探测响应的初始时间。
 * 请注意，这只是初始RTO，仅在尚未进行RTT测量时使用。
 * 每当我们收到探测响应时，实际的RTO都会有所变化。
 * 根据RFC 2988，它设置为3秒（3*10^6微秒）。 */
#define OSSCAN_INITIAL_RTO (3*1000000)

/******************************************************************************
 * 类定义                                                                  *
 ******************************************************************************/

/* 该类处理对网络的访问。它处理数据包传输调度、数据包捕获和拥塞控制。
 * 每个FPHost应该链接到该类的同一个实例，以便全局管理对网络的访问（整个操作系统检测过程）。 */
class FPNetworkControl {

 private:
  nsock_pool nsp;            /* Nsock pool.                                         */  // Nsock池
  nsock_iod pcap_nsi;        /* Nsock Pcap descriptor.                              */  // Nsock Pcap描述符
  nsock_event_id pcap_ev_id; /* Last pcap read event that was scheduled.            */  // 最后一次安排的pcap读取事件
  bool first_pcap_scheduled; /* True if we scheduled the first pcap read event.     */  // 如果我们安排了第一个pcap读取事件，则为真
  bool nsock_init;           /* True if the nsock pool has been initialized.        */  // 如果nsock池已初始化，则为真
  int rawsd;                 /* Raw socket.                                         */  // 原始套接字
  std::vector<FPHost *> callers;  /* List of users of this instance (used for callbacks).*/  // 此实例的用户列表（用于回调）
  int probes_sent;           /* Number of unique probes sent (not retransmissions). */  // 发送的唯一探测数量（不包括重传）
  int responses_recv;        /* Number of probe responses received.                 */  // 收到的探测响应数量
  int probes_timedout;       /* Number of probes that timeout after all retransms.  */  // 所有重传后超时的探测数量
  float cc_cwnd;             /* Current congestion window.                          */  // 当前拥塞窗口
  float cc_ssthresh;         /* Current Slow Start threshold.                       */  // 当前慢启动阈值

  int cc_init();             // 初始化拥塞控制
  int cc_update_sent(int pkts);  // 更新发送的数据包数量
  int cc_report_drop();       // 报告丢包
  int cc_update_received();   // 更新接收的数据包数量

 public:
  FPNetworkControl();         // 构造函数
  ~FPNetworkControl();        // 析构函数
  void init(const char *ifname, devtype iftype);  // 初始化函数
  int register_caller(FPHost *newcaller);  // 注册调用者
  int unregister_caller(FPHost *oldcaller);  // 注销调用者
  int setup_sniffer(const char *iface, const char *bfp_filter);  // 设置嗅探器
  void handle_events();       // 处理事件
  int scheduleProbe(FPProbe *pkt, int in_msecs_time);  // 安排探测
  void probe_transmission_handler(nsock_pool nsp, nsock_event nse, void *arg);  // 探测传输处理程序
  void response_reception_handler(nsock_pool nsp, nsock_event nse, void *arg);  // 响应接收处理程序
  bool request_slots(size_t num_packets);  // 请求插槽
  int cc_report_final_timeout();  // 报告最终超时

};
/*        +-----------+
          | FPEngine  |
          +-----------+
          |           |
          +-----+-----+
                |
        +-------+-------+
        |               |
        |               |
  +-----------+  +-----------+
  | FPEngine4 |  | FPEngine6 |
  +-----------+  +-----------+
  |           |  |           |
  +-----------+  +-----------+ */
/* 这个类是通用的指纹识别引擎。*/

class FPEngine {

 protected:
  size_t osgroup_size;

 public:
  FPEngine();
  virtual ~FPEngine();
  void reset();
  virtual int os_scan(std::vector<Target *> &Targets) = 0;
  const char *bpf_filter(std::vector<Target *> &Targets);

};


/* This class handles IPv6 OS fingerprinting. Using it is very simple, just
 * instance it and then call os_scan() with the list of IPv6 targets to
 * fingerprint. If everything goes well, the internal state of the supplied
 * target objects will be modified to reflect the results of the fingerprinting
 * process. */
/* 这个类处理 IPv6 操作系统指纹识别。使用它非常简单，只需实例化它，然后使用 IPv6 目标列表调用 os_scan() 进行指纹识别。如果一切顺利，提供的目标对象的内部状态将被修改以反映指纹识别过程的结果。*/
class FPEngine6 : public FPEngine {

 private:
  std::vector<FPHost6 *> fphosts; /* Information about each target to fingerprint */
  /* 每个目标的指纹识别信息 */

 public:
  FPEngine6();
  ~FPEngine6();
  void reset();
  int os_scan(std::vector<Target *> &Targets);

};


/*        +----------+
          | FPPacket |
          +----------+
          |          |
          +-----+----+
                |
                |
          +-----------+
          |  FPProbe  |
          +-----------+
          |           |
          +-----+-----+ */
/* 这个类表示用于操作系统指纹识别过程的通用数据包 */
class FPPacket {

 protected:
  PacketElement *pkt;      /* 实际与此 FPPacket 相关的数据包 */
  bool link_eth;           /* 是否需要以太网层 */
  struct eth_nfo eth_hdr;  /* 当 this->link_eth==true 时有效的以太网信息 */
  struct timeval pkt_time; /* 发送或接收数据包的时间 */

  int resetTime();         /* 重置时间 */
  void __reset();          /* 私有重置方法 */

 public:
  FPPacket();              /* 构造函数 */
  ~FPPacket();             /* 析构函数 */
  int setTime(const struct timeval *tv = NULL);  /* 设置时间 */
  struct timeval getTime() const;               /* 获取时间 */
  int setPacket(PacketElement *pkt);            /* 设置数据包 */
  int setEthernet(const u8 *src_mac, const u8 *dst_mac, const char *devname);  /* 设置以太网信息 */
  const struct eth_nfo *getEthernet() const;     /* 获取以太网信息 */
  const PacketElement *getPacket() const;        /* 获取数据包 */
  size_t getLength() const;                      /* 获取数据包长度 */
  u8 *getPacketBuffer(size_t *pkt_len) const;    /* 获取数据包缓冲区 */
  bool is_set() const;                           /* 是否已设置 */

};

/* 这个类表示一个通用的操作系统指纹探测包。换句话说，它表示 Nmap 发送到目标的网络数据包，以获取有关目标 TCP/IP 栈的信息。 */
class FPProbe : public FPPacket {

 private:
   const char *probe_id;    /* 探测包 ID */
   int probe_no;            /* 探测包编号 */
   int retransmissions;     /* 重传次数 */
   int times_replied;       /* 回复次数 */
   bool failed;             /* 是否失败 */
   bool timed;              /* 是否定时 */

 public:
  FPHost *host;            /* 主机信息 */

  FPProbe();               /* 构造函数 */
  ~FPProbe();              /* 析构函数 */
  void reset();            /* 重置方法 */
  bool isResponse(PacketElement *rcvd);  /* 是否为响应包 */
  int setProbeID(const char *id);        /* 设置探测包 ID */
  const char *getProbeID() const;        /* 获取探测包 ID */
  int getRetransmissions() const;        /* 获取重传次数 */
  int incrementRetransmissions();        /* 增加重传次数 */
  int getReplies() const;                 /* 获取回复次数 */
  int incrementReplies();                 /* 增加回复次数 */
  int setTimeSent();                      /* 设置发送时间 */
  int resetTimeSent();                    /* 重置发送时间 */
  struct timeval getTimeSent() const;     /* 获取发送时间 */
  bool probeFailed() const;               /* 探测是否失败 */
  int setFailed();                        /* 设置失败 */
  bool isTimed() const;                   /* 是否定时 */
  int setTimed();                         /* 设置定时 */
  int changeSourceAddress(struct in6_addr *addr);  /* 更改源地址 */

};

/* 这个类表示一个通用的接收到的数据包。 */
struct FPResponse {
  const char *probe_id;    /* 探测包 ID */
  u8 *buf;                 /* 缓冲区 */
  size_t len;              /* 长度 */
  struct timeval senttime, rcvdtime;  /* 发送时间和接收时间 */
  
  FPResponse(const char *probe_id, const u8 *buf, size_t len,  /* 构造函数 */
    // 定义了两个时间结构体变量senttime和rcvdtime
    struct timeval senttime, struct timeval rcvdtime);
    // 析构函数，用于释放对象占用的资源
    ~FPResponse();
/*        +-----------+
          |   FPHost  |
          +-----------+
          |           |
          +-----+-----+
                |
        +-------+-------+
        |               |
        |               |
  +-----------+  +-----------+
  |  FPHost4  |  |  FPHost6  |
  +-----------+  +-----------+
  |           |  |           |
  +-----------+  +-----------+  */
/* 这个类表示要进行指纹识别的通用主机。 */

/* 这个类表示要进行 IPv6 主机的指纹识别。该类异步执行操作系统检测。要使用它，必须定期调用 schedule()，直到 done() 返回 true 为止。之后，status() 将指示主机是否成功与特定操作系统匹配。 */
class FPHost6 : public FPHost {

 private:
  FPProbe fp_probes[NUM_FP_PROBES_IPv6];         /* 要发送的操作系统检测探针。*/
  FPResponse *fp_responses[NUM_FP_PROBES_IPv6];  /* 接收到的响应。            */
  FPResponse *aux_resp[NUM_FP_TIMEDPROBES_IPv6]; /* 用于定时响应的辅助向量 */

  int build_probe_list();
  int set_done_and_wrap_up();

 public:
  FPHost6(Target *tgt, FPNetworkControl *fpnc);
  ~FPHost6();
  void reset();
  void init(Target *tgt, FPNetworkControl *fpnc);
  void finish();
  bool done();
  int schedule();
  int callback(const u8 *pkt, size_t pkt_len, const struct timeval *tv);
  const FPProbe *getProbe(const char *id);
  const FPResponse *getResponse(const char *id);

  void fill_FPR(FingerPrintResultsIPv6 *FPR);

};


/******************************************************************************
 * Nsock handler wrappers.                                                    *
 ******************************************************************************/

void probe_transmission_handler_wrapper(nsock_pool nsp, nsock_event nse, void *arg);
void response_reception_handler_wrapper(nsock_pool nsp, nsock_event nse, void *arg);


std::vector<FingerMatch> load_fp_matches();
#endif /* __FPENGINE_H__ */

这行代码是条件编译指令，用于结束对应的条件编译块。在这里，它表示结束对应的头文件的条件编译块。__FPENGINE_H__ 是条件编译的标识符，用于确保头文件只被包含一次。
```