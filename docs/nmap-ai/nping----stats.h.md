# `nmap\nping\stats.h`

```cpp
#ifndef __STATS_H__
#define __STATS_H__ 1

#include <stdio.h>  // 包含标准输入输出库
#include <stdlib.h>  // 包含标准库函数
#include <stdarg.h>  // 包含可变参数列表的宏
#include <string.h>  // 包含字符串处理函数
#include <errno.h>  // 包含错误码定义
#include <ctype.h>  // 包含字符处理函数
#include "nping.h"  // 包含自定义头文件

#ifndef WIN32
#include <sys/types.h>  // 包含系统类型定义

#if HAVE_NETINET_IN_H
#include <netinet/in.h>  // 包含网络地址结构定义
#endif

#include <sys/mman.h>  // 包含内存映射函数
#include "nping_config.h"  // 包含自定义配置文件
#endif

#if HAVE_UNISTD_H
#include <unistd.h>  // 包含 POSIX 系统服务函数
#endif

#if TIME_WITH_SYS_TIME
# include <sys/time.h>  // 包含时间相关函数
# include <time.h>  // 包含时间函数
#else
# if HAVE_SYS_TIME_H
#  include <sys/time.h>  // 包含时间相关函数
# else
#  include <time.h>  // 包含时间函数
# endif
#endif

/* Make sure we define a 64bit integer type */
#ifndef u64_t
    #if WIN32
      typedef unsigned __int64 u64_t;  // 定义 64 位无符号整型
    #else
      typedef unsigned long long u64_t;  // 定义 64 位无符号整型
    #endif
#endif

/* Timeval subtraction in microseconds */
#define TIMEVAL_SUBTRACT(a,b) (((a).tv_sec - (b).tv_sec) * 1000000 + (a).tv_usec - (b).tv_usec)  // 定义时间差值（微秒）
/* Timeval subtract in milliseconds */
#define TIMEVAL_MSEC_SUBTRACT(a,b) ((((a).tv_sec - (b).tv_sec) * 1000) + ((a).tv_usec - (b).tv_usec) / 1000)  // 定义时间差值（毫秒）
/* Timeval subtract in seconds; truncate towards zero */
#define TIMEVAL_SEC_SUBTRACT(a,b) ((a).tv_sec - (b).tv_sec + (((a).tv_usec < (b).tv_usec) ? - 1 : 0))  // 定义时间差值（秒）
/* Timeval subtract in fractional seconds; convert to float */
#define TIMEVAL_FSEC_SUBTRACT(a,b) ((a).tv_sec - (b).tv_sec + (((a).tv_usec - (b).tv_usec)/1000000.0))  // 定义时间差值（秒，浮点数）

class NpingTimer {

  private:
    struct timeval start_tv;  // 定义开始时间
    struct timeval stop_tv;  // 定义结束时间

  public:
    NpingTimer();  // 构造函数
    ~NpingTimer();  // 析构函数
    void reset();  // 重置计时器
    int start();  // 开始计时
    int stop();  // 停止计时
    double elapsed(struct timeval *now=NULL);  // 返回经过的时间
    bool is_started();  // 判断计时器是否已开始
    bool is_stopped();  // 判断计时器是否已停止

  private:
      bool timeval_set(const struct timeval *tv);  // 设置时间
};

class NpingStats {

  private:
    u64_t packets_sent;  // 发送的数据包数量
    u64_t packets_received;  // 接收的数据包数量
    u64_t packets_echoed;  // 回显的数据包数量

    u64_t bytes_sent;  // 发送的字节数
    u64_t bytes_received;  // 接收的字节数
    u64_t bytes_echoed;  // 回显的字节数

    u32 echo_clients_served;  // 服务的回显客户端数量

    NpingTimer tx_timer;  /* Timer for packet transmission.         */  // 用于数据包传输的计时器
    NpingTimer rx_timer;  /* 用于数据包接收的计时器。*/
    NpingTimer run_timer; /* 用于测量 Nping 执行时间的计时器。*/

 public:
    NpingStats();  /* NpingStats 类的构造函数。*/
    ~NpingStats();  /* NpingStats 类的析构函数。*/

    void reset();  /* 重置统计数据。*/

    int addSentPacket(u32 len);  /* 增加发送数据包的数量和长度。*/
    int addRecvPacket(u32 len);  /* 增加接收数据包的数量和长度。*/
    int addEchoedPacket(u32 len);  /* 增加回显数据包的数量和长度。*/
    int addEchoClientServed();  /* 增加回显客户端服务数量。*/

    int startClocks();  /* 启动计时器。*/
    int stopClocks();  /* 停止计时器。*/

    int startTxClock();  /* 启动发送计时器。*/
    int stopTxClock();  /* 停止发送计时器。*/

    int startRxClock();  /* 启动接收计时器。*/
    int stopRxClock();  /* 停止接收计时器。*/

    int startRuntime();  /* 启动运行时间计时器。*/
    int stopRuntime();  /* 停止运行时间计时器。*/

    double elapsedTx();  /* 获取发送时间。*/
    double elapsedRx();  /* 获取接收时间。*/
    double elapsedRuntime(struct timeval *now=NULL);  /* 获取运行时间。*/

    u64_t getSentPackets();  /* 获取发送数据包数量。*/
    u64_t getSentBytes();  /* 获取发送数据包字节数。*/

    u64_t getRecvPackets();  /* 获取接收数据包数量。*/
    u64_t getRecvBytes();  /* 获取接收数据包字节数。*/

    u64_t getEchoedPackets();  /* 获取回显数据包数量。*/
    u64_t getEchoedBytes();  /* 获取回显数据包字节数。*/
    u32 getEchoClientsServed();  /* 获取回显客户端服务数量。*/

    u64_t getLostPackets();  /* 获取丢失数据包数量。*/
    double getLostPacketPercentage();  /* 获取丢失数据包百分比。*/
    double getLostPacketPercentage100();  /* 获取丢失数据包百分比（乘以100）。*/

    u64_t getUnmatchedPackets();  /* 获取不匹配数据包数量。*/
    double getUnmatchedPacketPercentage();  /* 获取不匹配数据包百分比。*/
    double getUnmatchedPacketPercentage100();  /* 获取不匹配数据包百分比（乘以100）。*/

    double getOverallTxPacketRate();  /* 获取总体发送数据包速率。*/
    double getOverallTxByteRate();  /* 获取总体发送字节速率。*/

    double getOverallRxPacketRate();  /* 获取总体接收数据包速率。*/
    double getOverallRxByteRate();  /* 获取总体接收字节速率。*/
# 结束 C++ 头文件的定义
};

# 结束条件编译指令，防止头文件被重复包含
#endif /* __STATS_H__ */
```