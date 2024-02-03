# `nmap\osscan2.h`

```cpp
#ifndef OSSCAN2_H
#define OSSCAN2_H

#include "nbase.h"
#include <dnet.h>
#include <pcap.h>

#include <vector>
#include <list>
#include "timing.h"
#include "osscan.h"
class FingerPrintResultsIPv4;
class Target;


/******************************************************************************
 * CONSTANT DEFINITIONS                                                       *
 ******************************************************************************/

/* 我们通常尝试的次数。如果目标看起来是一个好的指纹提交候选，可能会增加这个次数，或者如果用户给出了 --max-os-tries 选项，则可能会减少这个次数 */
#define STANDARD_OS2_TRIES 2

// 发送到单个主机的探测之间等待的最小（和目标）时间，以毫秒为单位。
#define OS_PROBE_DELAY 25

// 发送到单个主机的序列探测之间等待的目标时间，以毫秒为单位。理想情况下是500ms，因为常见的2Hz时间戳频率。少于500ms，我们可能看不到TS计数器的任何变化（即使我们看到了，它也变得不太准确）。大于500ms，我们有风险出现两个变化（即使我们只有一个，它也变得不太准确）。因此，我们在探测之间延迟100MS，留下500MS在第一个和第六个之间。
#define OS_SEQ_PROBE_DELAY 100

/* 我们发送多少个SYN数据包来对TCP序列进行探测？ */
#define NUM_SEQ_SAMPLES 6

/* TCP时间戳序列 */
#define TS_SEQ_UNKNOWN 0
#define TS_SEQ_ZERO 1 /* 我们收到的时间戳中至少有一个是0 */
#define TS_SEQ_2HZ 2
#define TS_SEQ_100HZ 3
#define TS_SEQ_1000HZ 4
#define TS_SEQ_OTHER_NUM 5
#define TS_SEQ_UNSUPPORTED 6 /* 系统没有发送时间戳回来 */

#define IPID_SEQ_UNKNOWN 0
#define IPID_SEQ_INCR 1  /* 每次简单递增一个 */
#define IPID_SEQ_BROKEN_INCR 2 /* 愚蠢的MS -- 忘记了htons()，所以在小端平台上每次计数256 */
#define IPID_SEQ_RPI 3 /* 每次增加一个“随机”正增量 */
#define IPID_SEQ_RD 4 /* 似乎使用“随机”分布来选择 IPID（意味着它可以增加或减少） */
#define IPID_SEQ_CONSTANT 5 /* 包含一个或多个连续的重复值 */
#define IPID_SEQ_ZERO 6 /* 每个返回的数据包的 IP.ID 都为 0（例如 Linux 2.4 这样做） */
#define IPID_SEQ_INCR_BY_2 7 /* 每次简单地增加两个 */

/******************************************************************************
 * 类型和结构定义                                                         *
 ******************************************************************************/

struct seq_info {
  int responses;
  int ts_seqclass; /* nmap.h 中的 TS_SEQ_* 定义 */
  int ipid_seqclass; /* nmap.h 中的 IPID_SEQ_* 定义 */
  u32 seqs[NUM_SEQ_SAMPLES];
  u32 timestamps[NUM_SEQ_SAMPLES];
  int index;
  u16 ipids[NUM_SEQ_SAMPLES];
  time_t lastboot; /* 0 表示未知 */
};

/* 不同类型的 Ipids。 */
struct ipid_info {
  u32 tcp_ipids[NUM_SEQ_SAMPLES];
  u32 tcp_closed_ipids[NUM_SEQ_SAMPLES];
  u32 icmp_ipids[NUM_SEQ_SAMPLES];
};

struct udpprobeinfo {
  u16 iptl;
  u16 ipid;
  u16 ipck;
  u16 sport;
  u16 dport;
  u16 udpck;
  u16 udplen;
  u8 patternbyte;
  struct in_addr target;
};

typedef enum OFProbeType {
  OFP_UNSET,
  OFP_TSEQ,
  OFP_TOPS,
  OFP_TECN,
  OFP_T1_7,
  OFP_TICMP,
  OFP_TUDP
} OFProbeType;

/******************************************************************************
 * 函数原型                                                         *
 ******************************************************************************/

int get_initial_ttl_guess(u8 ttl);

int identify_sequence(int numSamples, u32 *ipid_diffs, int islocalhost, int allipideqz);
int get_diffs(u32 *ipid_diffs, int numSamples, const u32 *ipids, int islocalhost);
int get_ipid_sequence_16(int numSamples, const u32 *ipids, int islocalhost);
/* 获取32位IPID序列 */
int get_ipid_sequence_32(int numSamples, const u32 *ipids, int islocalhost);

/* 将IPID类别转换为ASCII字符串 */
const char *ipidclass2ascii(int seqclass);

/* 将时间戳序列类别转换为ASCII字符串 */
const char *tsseqclass2ascii(int seqclass);

/* 将TCP序列预测困难度索引转换为困难度字符串 */
const char *seqidx2difficultystr(unsigned long idx);

/******************************************************************************
 * CLASS DEFINITIONS                                                          *
 ******************************************************************************/

/* 表示一个OS检测探测 */
class OFProbe;

/* 主机OS扫描统计 */
class HostOsScanStats;

/* 主机OS扫描 */
class HostOsScan;

/* 主机OS扫描信息 */
class HostOsScanInfo;

/* OS扫描信息 */
class OsScanInfo;

/** 表示一个OS检测探测。它不包含发送到目标的实际数据包，但包含足够的信息来生成它（如探测类型和其子ID）。它还存储时间信息。 */
class OFProbe {

 public:
  OFProbe();

  /* 当前探测类型的文字字符串 */
  const char *typestr() const;

  /* 探测类型：用于什么OS指纹测试？ */
  OFProbeType type;

  /* 用于区分不同的TCP/UDP/ICMP的探测的子ID */
  int subid;

  /* 这个探测的尝试（重传）次数 */
  int tryno;

  /* 由于数据包发送速率限制，数据包可能在重新发送之前被超时一段时间 */
  bool retransmitted;

  struct timeval sent;

  /* 如果这是一个重传（tryno > 0），则记录上一个探测发送的时间 */
  struct timeval prevSent;
};

/* 存储在扫描轮次中正在扫描的主机的状态 */
class HostOsScanStats;

/* 这些是整个目标组的统计数据 */
class ScanStats {

 public:
  ScanStats();  /* 构造函数，用于初始化对象 */
  bool sendOK() const; /* 如果系统表示发送正常，则返回true */

  struct ultra_timing_vals timing;  /* 用于存储超级定时值的结构体 */
  struct timeout_info to;      /* 存储往返时间/超时信息的结构体 */
  int num_probes_active;       /* 活动探测的总数 */
  int num_probes_sent;         /* 总共发送的探测数量 */
  int num_probes_sent_at_last_wait;  /* 上次等待时发送的探测数量 */
};


/* 执行扫描任务的类，设置并使用主机在主机的HostOsScanStats中的状态。 */
};



/* 维护不完整的HostOsScanInfo的链表。 */
class OsScanInfo {

 public:
  OsScanInfo(std::vector<Target *> &Targets);  /* 构造函数，接受目标的向量并初始化对象 */
  ~OsScanInfo();  /* 析构函数，用于清理对象 */

  float starttime;  /* 扫描开始时间 */

  /* 如果从中删除，最好也调整nextI（或者在之后调用resetHostIterator()）。不要让这个列表变为空，然后再添加，否则可能会搞乱nextI（我不确定） */
  std::list<HostOsScanInfo *> incompleteHosts;  /* 不完整的主机扫描信息的链表 */

  unsigned int numIncompleteHosts() const {return incompleteHosts.size();}  /* 返回不完整主机的数量 */
  HostOsScanInfo *findIncompleteHost(const struct sockaddr_storage *ss);  /* 查找不完整的主机扫描信息 */

  /* 不完整主机的循环缓冲区。nextIncompleteHost()给出下一个。第一次调用时，它将给出列表中的第一个主机。如果incompleteHosts为空，则返回NULL。 */
  HostOsScanInfo *nextIncompleteHost();  /* 返回下一个不完整的主机 */

  /* 重置用于nextIncompleteHost()的主机迭代器到开头。如果从incompleteHosts中删除主机，请立即调用此函数 */
  void resetHostIterator() { nextI = incompleteHosts.begin(); }  /* 重置主机迭代器 */

  int removeCompletedHosts();  /* 删除已完成的主机 */

 private:
  unsigned int numInitialTargets;  /* 初始目标的数量 */
  std::list<HostOsScanInfo *>::iterator nextI;  /* 下一个主机的迭代器 */
};


/* 主机的整体操作系统扫描信息：
 *  - 每次扫描轮次得到的指纹；
 *  - 这些指纹的匹配结果。
 *  - 它是否超时/完成？
 *  - ... */
class HostOsScanInfo {

 public:
  // 构造函数，接受目标对象和操作系统扫描信息对象作为参数
  HostOsScanInfo(Target *t, OsScanInfo *OSI);
  // 析构函数
  ~HostOsScanInfo();

  Target *target;       /* The target                                  */
  FingerPrintResultsIPv4 *FPR;
  OsScanInfo *OSI;      /* The OSI which contains this HostOsScanInfo  */
  FingerPrint **FPs;    /* Fingerprints of the host                    */
  FingerPrintResultsIPv4 *FP_matches; /* Fingerprint-matching results      */
  bool timedOut;        /* Did it time out?                            */
  bool isCompleted;     /* Has the OS detection been completed?        */
  HostOsScanStats *hss; /* Scan status of the host in one scan round   */
};


/** This is the class that performs OS detection (both IPv4 and IPv6).
  * Using it is simple, just call os_scan() passing a list of targets.
  * The results of the detection will be stored inside the supplied
  * target objects. */
class OSScan {

 private:
  // 分块并执行扫描的私有方法，接受目标对象列表和地址族作为参数
  int chunk_and_do_scan(std::vector<Target *> &Targets, int family);
  // 对 IPv4 地址进行操作系统扫描的私有方法，接受目标对象列表作为参数
  int os_scan_ipv4(std::vector<Target *> &Targets);
  // 对 IPv6 地址进行操作系统扫描的私有方法，接受目标对象列表作为参数
  int os_scan_ipv6(std::vector<Target *> &Targets);

  public:
   // 构造函数
   OSScan();
   // 析构函数
   ~OSScan();
   // 重置方法
   void reset();
   // 对目标对象列表进行操作系统扫描的方法
   int os_scan(std::vector<Target *> &Targets);
};

#endif /*OSSCAN2_H*/
```