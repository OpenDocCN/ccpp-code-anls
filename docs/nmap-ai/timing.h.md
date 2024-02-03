# `nmap\timing.h`

```cpp
/* $Id$ */

#ifndef NMAP_TIMING_H
#define NMAP_TIMING_H

#if TIME_WITH_SYS_TIME
# include <sys/time.h>
# include <time.h>
#else
# if HAVE_SYS_TIME_H
#  include <sys/time.h>
# else
#  include <time.h>
# endif
#endif

#include <nbase.h> /* u32 */

/* 基于 RFC2581 中的 TCP 拥塞控制技术。*/
struct ultra_timing_vals {
  double cwnd; /* 拥塞窗口 - 以探测为单位 */
  int ssthresh; /* 当超过此阈值时，模式从慢启动变为拥塞避免 */
  /* 如果每个探测都产生回复，我们期望收到的回复数量。这几乎等同于发送的探测总数，但在收到回复或探测超时之前不会递增。这个值和
     num_replies_received 用于缩放拥塞窗口的增量。 */
  int num_replies_expected;
  /* 我们收到的任何类型探测的回复数量。 */
  int num_replies_received;
  /* 此定时结构的更新次数（通常是数据包接收次数）。 */
  int num_updates;
  /* 上次调整数值的时间（通常只希望基于该调整后发送的探测来再次调整，以防突然的大量丢包破坏定时）。初始化为当前时间 */
  struct timeval last_drop;

  double cc_scale(const struct scan_performance_vars *perf);
  void ack(const struct scan_performance_vars *perf, double scale = 1.0);
  void drop(unsigned in_flight,
    const struct scan_performance_vars *perf, const struct timeval *now);
  void drop_group(unsigned in_flight,
    const struct scan_performance_vars *perf, const struct timeval *now);
};

/* 这些主要是 ultra_timing_vals 的初始化器。 */
# 定义了一个结构体，用于存储扫描性能相关的变量
struct scan_performance_vars {
  int low_cwnd;  /* 允许的最低拥塞窗口大小 */
  int host_initial_cwnd; /* 单个主机的初始拥塞窗口大小 */
  int group_initial_cwnd; /* 所有主机作为一个组的初始拥塞窗口大小 */
  int max_cwnd; /* 最多允许有这么多个探测同时进行 */
  int slow_incr; /* 慢启动模式下，每个响应增加的探测数量 */
  int ca_incr; /* 拥塞避免模式下，每个（大致）往返时间增加的探测数量 */
  int cc_scale_max; /* 拥塞窗口增加的最大比例因子 */
  int initial_ssthresh; /* 初始慢启动门限 */
  double group_drop_cwnd_divisor; /* 如果发生任何数据包丢失，所有主机组的拥塞窗口除以这个值 */
  double group_drop_ssthresh_divisor; /* 如果发生任何数据包丢失，用于降低组的慢启动门限 */
  double host_drop_ssthresh_divisor; /* 如果发生任何数据包丢失，用于降低主机的慢启动门限 */

  /* 在填充全局 NmapOps 表后进行初始化 */
  void init();
};

# 定义了一个结构体，用于存储超时信息
struct timeout_info {
  int srtt; /* 平滑的往返时间估计（微秒） */
  int rttvar; /* 往返时间方差 */
  int timeout; /* 当前超时阈值（微秒） */
};

# 调用这个函数来初始化一个新分配的 timeout_info 结构体的值
void initialize_timeout_info(struct timeout_info *to);

# 与 adjust_timeouts() 相同，但允许指定接收时间（这可能是因为它在一段时间前接收到，也可能是为了效率，因为调用者已经知道当前时间）
void adjust_timeouts2(const struct timeval *sent,
                      const struct timeval *received,
                      struct timeout_info *to);
/* 根据最新探测所花费的时间调整超时数值。更新我们的往返时间平均值等。 */
void adjust_timeouts(struct timeval sent, struct timeout_info *to);

#define DEFAULT_CURRENT_RATE_HISTORY 5.0

/* 如果需要，睡眠以确保它不会在小于 o.send_delay 的时间内被调用两次。如果传入非空的 tv，将记录睡眠后的时间 */
void enforce_scan_delay(struct timeval *tv);

/* 这个类用于测量某个数量的当前和平均速率。 */
class RateMeter {
  public:
    RateMeter(double current_rate_history = DEFAULT_CURRENT_RATE_HISTORY);

    void start(const struct timeval *now = NULL);
    void stop(const struct timeval *now = NULL);
    void update(double amount, const struct timeval *now = NULL);
    double getOverallRate(const struct timeval *now = NULL) const;
    double getCurrentRate(const struct timeval *now = NULL, bool update = true);
    double getTotal(void) const;
    double elapsedTime(const struct timeval *now = NULL) const;

  private:
    /* 计算“当前”速率时要回溯多少秒。 */
    double current_rate_history;

    /* 记录此计量器开始记录的时间。 */
    struct timeval start_tv;
    /* 记录此计量器停止记录的时间。 */
    struct timeval stop_tv;
    /* 上次更新当前样本速率的时间。 */
    struct timeval last_update_tv;

    double total;
    double current_rate;

    static bool isSet(const struct timeval *tv);
};

/* 一个测量数据包和字节速率的 RateMeter 的特化。 */
class PacketRateMeter {
  public:
    PacketRateMeter(double current_rate_history = DEFAULT_CURRENT_RATE_HISTORY);

    void start(const struct timeval *now = NULL);
    void stop(const struct timeval *now = NULL);
    void update(u32 len, const struct timeval *now = NULL);
    double getOverallPacketRate(const struct timeval *now = NULL) const;
    # 获取当前数据包速率，可以传入时间参数，是否更新速率
    double getCurrentPacketRate(const struct timeval *now = NULL, bool update = true);
    # 获取总体字节速率，可以传入时间参数
    double getOverallByteRate(const struct timeval *now = NULL) const;
    # 获取当前字节速率，可以传入时间参数，是否更新速率
    double getCurrentByteRate(const struct timeval *now = NULL, bool update = true);
    # 获取数据包数量
    unsigned long long getNumPackets(void) const;
    # 获取字节数量
    unsigned long long getNumBytes(void) const;

  private:
    # 数据包速率计量器
    RateMeter packet_rate_meter;
    # 字节速率计量器
    RateMeter byte_rate_meter;
};

class ScanProgressMeter {
 public:
  /* 保存 stypestr 的副本，以便在打印统计信息时使用 */
  ScanProgressMeter(const char *stypestr);
  ~ScanProgressMeter();
  /* 决定是否可能打印时间统计报告。对打印频率和详细程度有严格限制，因此最好在计算进度信息之前先检查这一点。如果调用者没有当前时间，则 now 可以为 NULL。即使此函数返回 true，也不意味着下一个 printStatsIfNecessary 总是会打印内容。这取决于时间估计是否发生了变化，而这个函数甚至不知道这一点。 */
  bool mayBePrinted(const struct timeval *now);

  /* 打印此扫描预计何时完成的估计。只有在 mayBePrinted() 为 true 且估计发生了显着变化时才会这样做。返回是否打印了一行。 */
  bool printStatsIfNecessary(double perc_done, const struct timeval *now);

  /* 打印此扫描预计何时完成的估计。 */
  bool printStats(double perc_done, const struct timeval *now);

  /* 打印此任务已完成。 */
  bool endTask(const struct timeval *now, const char *additional_info) { return beginOrEndTask(now, additional_info, false); }

  struct timeval begin; /* 实例化此 ScanProgressMeter 的时间 */
 private:
  struct timeval last_print_test; /* 上次调用 printStatsIfNecessary 的时间 */
  struct timeval last_print; /* 最近一次打印 ETC 的时间 */
  char *scantypestr;
  struct timeval last_est; /* 最新的已打印估计 */

  bool beginOrEndTask(const struct timeval *now, const char *additional_info, bool beginning);
};

#endif /* NMAP_TIMING_H */
```