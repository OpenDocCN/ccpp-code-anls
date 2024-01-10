# `nmap\libpcap\pcap-dos.h`

```
/*
 * 内部细节，用于在 DOS 上使用 libpcap。
 * 32 位目标：djgpp、Pharlap 或 DOS4GW。
 */

#ifndef __PCAP_DOS_H
#define __PCAP_DOS_H

#ifdef __DJGPP__
#include <pc.h>    /* 简单的非 conio kbhit */
#else
#include <conio.h>
#endif

typedef int            BOOL;  /* 布尔类型 */
typedef unsigned char  BYTE;  /* 无符号字符 */
typedef unsigned short WORD;  /* 无符号短整型 */
typedef unsigned long  DWORD; /* 无符号长整型 */
typedef BYTE           ETHER[6];  /* 以太网地址 */

#define ETH_ALEN       sizeof(ETHER)   /* 以太网地址长度 */
#define ETH_HLEN       (2*ETH_ALEN+2)  /* 以太网头部长度 */
#define ETH_MTU        1500  /* 以太网最大传输单元 */
#define ETH_MIN        60    /* 以太网最小长度 */
#define ETH_MAX        (ETH_MTU+ETH_HLEN)  /* 以太网最大长度 */

#ifndef TRUE
  #define TRUE   1  /* 真值 */
  #define FALSE  0  /* 假值 */
#endif

#define PHARLAP  1  /* Pharlap 环境 */
#define DJGPP    2  /* djgpp 环境 */
#define DOS4GW   4  /* DOS4GW 环境 */

#ifdef __DJGPP__
  #undef  DOSX
  #define DOSX DJGPP
#endif

#ifdef __WATCOMC__
  #undef  DOSX
  #define DOSX DOS4GW
#endif

#ifdef __HIGHC__
  #include <pharlap.h>
  #undef  DOSX
  #define DOSX PHARLAP
  #define inline
#else
  typedef unsigned int UINT;
#endif


#if defined(__GNUC__) || defined(__HIGHC__)
  typedef unsigned long long  uint64;  /* 64 位无符号长整型 */
  typedef unsigned long long  QWORD;   /* 64 位无符号长整型 */
#endif

#if defined(__WATCOMC__)
  typedef unsigned __int64  uint64;  /* 64 位无符号长整型 */
  typedef unsigned __int64  QWORD;   /* 64 位无符号长整型 */
#endif

#define ARGSUSED(x)  (void) x  /* 使用参数，避免编译器警告 */

#if defined (__SMALL__) || defined(__LARGE__)
  #define DOSX 0
#elif !defined(DOSX)
  #error DOSX 未定义；1 = PharLap，2 = djgpp，4 = DOS4GW
#endif

#ifdef __HIGHC__
#define min(a,b) _min(a,b)  /* 返回 a 和 b 中的最小值 */
#define max(a,b) _max(a,b)  /* 返回 a 和 b 中的最大值 */
#endif

#ifndef min
#define min(a,b) ((a) < (b) ? (a) : (b))  /* 返回 a 和 b 中的最小值 */
#endif

#ifndef max
#define max(a,b) ((a) < (b) ? (b) : (a))  /* 返回 a 和 b 中的最大值 */
#endif

#if !defined(_U_) && defined(__GNUC__)
#define _U_  __attribute__((unused))  /* 标记为未使用的变量 */
#endif

#ifndef _U_
#define _U_  /* 未使用的变量标记 */
#endif
#if defined(USE_32BIT_DRIVERS)
  #include "msdos/pm_drvr/lock.h"  // 包含锁定相关的头文件

  #ifndef RECEIVE_QUEUE_SIZE
  #define RECEIVE_QUEUE_SIZE  60  // 如果未定义接收队列大小，则设置默认值为60
  #endif

  #ifndef RECEIVE_BUF_SIZE
  #define RECEIVE_BUF_SIZE   (ETH_MAX+20)  // 如果未定义接收缓冲区大小，则设置默认值为(ETH_MAX+20)
  #endif

  extern struct device el2_dev     LOCKED_VAR;  /* 3Com EtherLink II */  // 声明3Com EtherLink II设备
  extern struct device el3_dev     LOCKED_VAR;  /*      EtherLink III */  // 声明EtherLink III设备
  extern struct device tc59_dev    LOCKED_VAR;  /* 3Com Vortex Card (?) */  // 声明3Com Vortex Card (?)设备
  extern struct device tc515_dev   LOCKED_VAR;  // 声明tc515_dev设备
  extern struct device tc90x_dev   LOCKED_VAR;  // 声明tc90x_dev设备
  extern struct device tc90bcx_dev LOCKED_VAR;  // 声明tc90bcx_dev设备
  extern struct device wd_dev      LOCKED_VAR;  // 声明wd_dev设备
  extern struct device ne_dev      LOCKED_VAR;  // 声明ne_dev设备
  extern struct device acct_dev    LOCKED_VAR;  // 声明acct_dev设备
  extern struct device cs89_dev    LOCKED_VAR;  // 声明cs89_dev设备
  extern struct device rtl8139_dev LOCKED_VAR;  // 声明rtl8139_dev设备

  struct rx_ringbuf {
         volatile int in_index;   /* queue index head */  // 接收环形缓冲区的队列索引头
         int          out_index;  /* queue index tail */  // 接收环形缓冲区的队列索引尾
         int          elem_size;  /* size of each element */  // 每个元素的大小
         int          num_elem;   /* number of elements */  // 元素的数量
         char        *buf_start;  /* start of buffer pool */  // 缓冲池的起始位置
       };

  struct rx_elem {
         DWORD size;              /* size copied to this element */  // 复制到该元素的大小
         BYTE  data[ETH_MAX+10];  /* add some margin. data[0] should be */  // 添加一些边距。data[0]应该是
       };                         /* dword aligned */

  extern BYTE *get_rxbuf     (int len) LOCKED_FUNC;  // 获取接收缓冲区
  extern int   peek_rxbuf    (BYTE **buf);  // 查看接收缓冲区
  extern int   release_rxbuf (BYTE  *buf);  // 释放接收缓冲区

#endif

extern struct device       *active_dev  LOCKED_VAR;  // 声明锁定的活动设备
extern const struct device *dev_base    LOCKED_VAR;  // 声明锁定的设备基址
extern struct device       *probed_dev;  // 声明probed_dev

extern int pcap_pkt_debug;  // 声明pcap_pkt_debug变量

extern void _w32_os_yield (void); /* Watt-32's misc.c */  // 声明_w32_os_yield函数

#ifdef NDEBUG
  #define PCAP_ASSERT(x) ((void)0)  // 如果定义了NDEBUG，则定义PCAP_ASSERT宏为空
#else
  // 定义一个函数，用于在条件不满足时触发断言
  void pcap_assert (const char *what, const char *file, unsigned line);

  // 定义一个宏，用于在条件不满足时触发断言
  #define PCAP_ASSERT(x) do { \
                           if (!(x)) \
                              pcap_assert (#x, __FILE__, __LINE__); \
                         } while (0)
#endif

#endif  /* __PCAP_DOS_H */
```