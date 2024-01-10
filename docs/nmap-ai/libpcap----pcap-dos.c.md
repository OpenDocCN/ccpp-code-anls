# `nmap\libpcap\pcap-dos.c`

```
/*
 *  This file is part of DOS-libpcap
 *  Ported to DOS/DOSX by G. Vanem <gvanem@yahoo.no>
 *
 *  pcap-dos.c: Interface to PKTDRVR, NDIS2 and 32-bit pmode
 *              network drivers.
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <signal.h>
#include <float.h>
#include <fcntl.h>
#include <limits.h> /* for INT_MAX */
#include <io.h>

#if defined(USE_32BIT_DRIVERS)
  #include "msdos/pm_drvr/pmdrvr.h"
  #include "msdos/pm_drvr/pci.h"
  #include "msdos/pm_drvr/bios32.h"
  #include "msdos/pm_drvr/module.h"
  #include "msdos/pm_drvr/3c501.h"
  #include "msdos/pm_drvr/3c503.h"
  #include "msdos/pm_drvr/3c509.h"
  #include "msdos/pm_drvr/3c59x.h"
  #include "msdos/pm_drvr/3c515.h"
  #include "msdos/pm_drvr/3c90x.h"
  #include "msdos/pm_drvr/3c575_cb.h"
  #include "msdos/pm_drvr/ne.h"
  #include "msdos/pm_drvr/wd.h"
  #include "msdos/pm_drvr/accton.h"
  #include "msdos/pm_drvr/cs89x0.h"
  #include "msdos/pm_drvr/rtl8139.h"
  #include "msdos/pm_drvr/ne2k-pci.h"
#endif

#include "pcap.h"
#include "pcap-dos.h"
#include "pcap-int.h"
#include "msdos/pktdrvr.h"

#ifdef USE_NDIS2
#include "msdos/ndis2.h"
#endif

#include <arpa/inet.h>
#include <net/if.h>
#include <net/if_arp.h>
#include <net/if_ether.h>
#include <net/if_packe.h>
#include <tcp.h>

#if defined(USE_32BIT_DRIVERS)
  #define FLUSHK()       do { _printk_safe = 1; _printk_flush(); } while (0)
  #define NDIS_NEXT_DEV  &rtl8139_dev

  static char *rx_pool = NULL;
  static void init_32bit (void);

  static int  pktq_init     (struct rx_ringbuf *q, int size, int num, char *pool);
  static int  pktq_check    (struct rx_ringbuf *q);
  static int  pktq_inc_out  (struct rx_ringbuf *q);
  static int  pktq_in_index (struct rx_ringbuf *q) LOCKED_FUNC;
  static void pktq_clear    (struct rx_ringbuf *q) LOCKED_FUNC;

  static struct rx_elem *pktq_in_elem  (struct rx_ringbuf *q) LOCKED_FUNC;
  static struct rx_elem *pktq_out_elem (struct rx_ringbuf *q);
#else
  #define FLUSHK()      ((void)0)  // 定义宏 FLUSHK()，不执行任何操作
  #define NDIS_NEXT_DEV  NULL       // 定义宏 NDIS_NEXT_DEV，赋值为 NULL
#endif

/*
 * Internal variables/functions in Watt-32
 */
extern WORD  _pktdevclass;  // 声明外部变量 _pktdevclass，类型为 WORD
extern BOOL  _eth_is_init;  // 声明外部变量 _eth_is_init，类型为 BOOL
extern int   _w32_dynamic_host;  // 声明外部变量 _w32_dynamic_host，类型为 int
extern int   _watt_do_exit;  // 声明外部变量 _watt_do_exit，类型为 int
extern int   _watt_is_init;  // 声明外部变量 _watt_is_init，类型为 int
extern int   _w32__bootp_on, _w32__dhcp_on, _w32__rarp_on, _w32__do_mask_req;  // 声明外部变量 _w32__bootp_on, _w32__dhcp_on, _w32__rarp_on, _w32__do_mask_req，类型为 int
extern void (*_w32_usr_post_init) (void);  // 声明外部变量 _w32_usr_post_init，类型为指向无返回值、无参数的函数指针
extern void (*_w32_print_hook)();  // 声明外部变量 _w32_print_hook，类型为指向无返回值、无参数的函数指针

extern void dbug_write (const char *);  // 声明外部函数 dbug_write，参数类型为指向常量字符的指针
extern int  pkt_get_mtu (void);  // 声明外部函数 pkt_get_mtu，无参数

static int ref_count = 0;  // 声明静态变量 ref_count，类型为 int，赋值为 0

static u_long mac_count    = 0;  // 声明静态变量 mac_count，类型为 u_long，赋值为 0
static u_long filter_count = 0;  // 声明静态变量 filter_count，类型为 u_long，赋值为 0

static volatile BOOL exc_occured = 0;  // 声明静态变量 exc_occured，类型为 volatile BOOL，赋值为 0

static struct device *handle_to_device [20];  // 声明静态变量 handle_to_device，类型为指向结构体 device 的指针数组，长度为 20

static int  pcap_activate_dos (pcap_t *p);  // 声明静态函数 pcap_activate_dos，参数类型为 pcap_t 指针，返回类型为 int
static int  pcap_read_dos (pcap_t *p, int cnt, pcap_handler callback, u_char *data);  // 声明静态函数 pcap_read_dos，参数类型为 pcap_t 指针、int、pcap_handler、u_char 指针，返回类型为 int
static void pcap_cleanup_dos (pcap_t *p);  // 声明静态函数 pcap_cleanup_dos，参数类型为 pcap_t 指针，无返回值
static int  pcap_stats_dos (pcap_t *p, struct pcap_stat *ps);  // 声明静态函数 pcap_stats_dos，参数类型为 pcap_t 指针、指向结构体 pcap_stat 的指针，返回类型为 int
static int  pcap_sendpacket_dos (pcap_t *p, const void *buf, size_t len);  // 声明静态函数 pcap_sendpacket_dos，参数类型为 pcap_t 指针、指向常量 void 的指针、size_t，返回类型为 int
static int  pcap_setfilter_dos (pcap_t *p, struct bpf_program *fp);  // 声明静态函数 pcap_setfilter_dos，参数类型为 pcap_t 指针、指向结构体 bpf_program 的指针，返回类型为 int

static int  ndis_probe (struct device *dev);  // 声明静态函数 ndis_probe，参数类型为指向结构体 device 的指针，返回类型为 int
static int  pkt_probe  (struct device *dev);  // 声明静态函数 pkt_probe，参数类型为指向结构体 device 的指针，返回类型为 int

static void close_driver (void);  // 声明静态函数 close_driver，无参数，无返回值
static int  init_watt32 (struct pcap *pcap, const char *dev_name, char *err_buf);  // 声明静态函数 init_watt32，参数类型为指向结构体 pcap 的指针、指向常量字符的指针、字符指针，返回类型为 int
static int  first_init (const char *name, char *ebuf, int promisc);  // 声明静态函数 first_init，参数类型为指向常量字符的指针、字符指针、int，返回类型为 int

static void watt32_recv_hook (u_char *dummy, const struct pcap_pkthdr *pcap, const u_char *buf);  // 声明静态函数 watt32_recv_hook，参数类型为 u_char 指针、指向结构体 pcap_pkthdr 的指针、指向常量 u_char 的指针

/*
 * These are the device we always support
 */
static struct device ndis_dev = {  // 声明静态结构体 device 变量 ndis_dev
              "ndis",  // 设备名称
              "NDIS2 LanManager",  // 设备描述
              0,0,0,0,0,0,0,  // 设备属性
              NDIS_NEXT_DEV,  // NULL or a 32-bit device
              ndis_probe  // 设备探测函数
            };

static struct device pkt_dev = {  // 声明静态结构体 device 变量 pkt_dev
              "pkt",  // 设备名称
              "Packet-Driver",  // 设备描述
              0,0,0,0,0,0,0,  // 设备属性
              &ndis_dev,  // 指向 ndis_dev 的指针
              pkt_probe  // 设备探测函数
            };
// 根据文件描述符获取设备指针
static struct device *get_device (int fd)
{
  // 如果文件描述符小于等于0或者大于等于设备指针数组的大小，则返回空指针
  if (fd <= 0 || fd >= sizeof(handle_to_device)/sizeof(handle_to_device[0]))
     return (NULL);
  // 返回对应文件描述符的设备指针
  return handle_to_device [fd-1];
}

/*
 * MS-DOS 上用于捕获数据的私有数据
 */
struct pcap_dos {
    void (*wait_proc)(void); /* 在等待时调用的过程 */
    struct pcap_stat stat;
};

// 创建接口的函数
pcap_t *pcap_create_interface (const char *device _U_, char *ebuf)
{
    pcap_t *p;

    // 调用通用的创建函数，传入 MS-DOS 的私有数据结构
    p = PCAP_CREATE_COMMON(ebuf, struct pcap_dos);
    if (p == NULL)
        return (NULL);

    // 设置激活操作为 MS-DOS 特定的激活函数
    p->activate_op = pcap_activate_dos;
    return (p);
}

/*
 * 用于打开 MAC 驱动程序，用于捕获网络数据包
 */
static int pcap_activate_dos (pcap_t *pcap)
{
  // 如果设置了监控模式，则在 DOS 上不支持监控模式
  if (pcap->opt.rfmon) {
    return (PCAP_ERROR_RFMON_NOTSUP);
  }

  // 将负的快照值、快照值为0或者大于最大允许值的快照值，都设置为最大允许值
  if (pcap->snapshot <= 0 || pcap->snapshot > MAXIMUM_SNAPLEN)
    pcap->snapshot = MAXIMUM_SNAPLEN;

  // 如果快照值小于以太网最小值加8，则设置为以太网最小值加8
  if (pcap->snapshot < ETH_MIN+8)
      pcap->snapshot = ETH_MIN+8;

  // 如果快照值大于以太网最大值，则设置为以太网最大值
  if (pcap->snapshot > ETH_MAX)   /* 静默接受并截断大的 MTU */
      pcap->snapshot = ETH_MAX;

  // 设置链路类型为以太网
  pcap->linktype          = DLT_EN10MB;  /* !! */
  // 设置清理操作为 MS-DOS 特定的清理函数
  pcap->cleanup_op        = pcap_cleanup_dos;
  // 设置读取操作为 MS-DOS 特定的读取函数
  pcap->read_op           = pcap_read_dos;
  // 设置统计操作为 MS-DOS 特定的统计函数
  pcap->stats_op          = pcap_stats_dos;
  // 设置发送数据包操作为 MS-DOS 特定的发送函数
  pcap->inject_op         = pcap_sendpacket_dos;
  // 设置过滤操作为 MS-DOS 特定的过滤函数
  pcap->setfilter_op      = pcap_setfilter_dos;
  // 设置方向操作为 NULL，未实现
  pcap->setdirection_op   = NULL;  /* 未实现 */
  // 设置文件描述符为递增的引用计数
  pcap->fd                = ++ref_count;

  // 设置缓冲区大小为以太网最大值加100
  pcap->bufsize = ETH_MAX+100;     /* 添加一些余量 */
  // 分配缓冲区内存
  pcap->buffer = calloc (pcap->bufsize, 1);

  // 如果文件描述符为1，表示第一次调用
  if (pcap->fd == 1)  /* 第一次调用 */
  {
    // 如果初始化watt32失败，或者第一次初始化失败，则返回错误
    if (!init_watt32(pcap, pcap->opt.device, pcap->errbuf) ||
        !first_init(pcap->opt.device, pcap->errbuf, pcap->opt.promisc))
    {
      /* XXX - free pcap->buffer? */
      // 释放pcap->buffer内存
      return (PCAP_ERROR);
    }
    // 在程序退出时调用close_driver函数
    atexit (close_driver);
  }
  // 如果活动设备的名称与pcap的设备名称不相同
  else if (stricmp(active_dev->name,pcap->opt.device))
  {
    // 格式化错误信息到pcap->errbuf
    snprintf (pcap->errbuf, PCAP_ERRBUF_SIZE,
                   "Cannot use different devices simultaneously "
                   "(`%s' vs. `%s')", active_dev->name, pcap->opt.device);
    /* XXX - free pcap->buffer? */
    // 释放pcap->buffer内存
    return (PCAP_ERROR);
  }
  // 将文件描述符对应的设备存储到handle_to_device数组中
  handle_to_device [pcap->fd-1] = active_dev;
  // 返回0表示成功
  return (0);
}
/*
 * 从接收队列中轮询并调用 pcap 回调处理程序处理数据包。
 */
static int
pcap_read_one (pcap_t *p, pcap_handler callback, u_char *data)
{
  // 获取 pcap_t 结构体中的私有数据
  struct pcap_dos *pd = p->priv;
  // 定义 pcap 包头和时间结构体
  struct pcap_pkthdr pcap;
  struct timeval     now, expiry = { 0,0 };
  int    rx_len = 0;

  // 如果设置了超时时间
  if (p->opt.timeout > 0)
  {
    // 获取当前时间
    gettimeofday2 (&now, NULL);
    // 计算超时时间
    expiry.tv_usec = now.tv_usec + 1000UL * p->opt.timeout;
    expiry.tv_sec  = now.tv_sec;
    // 处理超过 1 秒的情况
    while (expiry.tv_usec >= 1000000L)
    {
      expiry.tv_usec -= 1000000L;
      expiry.tv_sec++;
    }
  }

  // 循环直到发生异常
  while (!exc_occured)
  {
    volatile struct device *dev; /* 可能会被信号处理程序重置 */

    // 获取设备
    dev = get_device (p->fd);
    // 如果设备不存在，跳出循环
    if (!dev)
       break;

    // 断言设备有复制或者查看接收缓冲区的能力
    PCAP_ASSERT (dev->copy_rx_buf || dev->peek_rx_buf);
    FLUSHK();

    /* 如果驱动程序具有零拷贝接收功能，则查看队列，过滤它，执行回调并释放缓冲区。 */
    if (dev->peek_rx_buf)
    {
      PCAP_ASSERT (dev->release_rx_buf);
      rx_len = (*dev->peek_rx_buf) (&p->buffer);
    }
    else
    {
      rx_len = (*dev->copy_rx_buf) (p->buffer, p->snapshot);
    }

    // 如果接收到数据包
    if (rx_len > 0)
    {
      mac_count++;

      FLUSHK();

      // 设置 pcap 包头的长度和实际长度
      pcap.caplen = min (rx_len, p->snapshot);
      pcap.len    = rx_len;

      // 如果有回调函数并且通过过滤器
      if (callback &&
          (!p->fcode.bf_insns || pcap_filter(p->fcode.bf_insns, p->buffer, pcap.len, pcap.caplen)))
      {
        filter_count++;

        /* 修复!! 应该是到达时间，而不是捕获时间。 */
        gettimeofday2 (&pcap.ts, NULL);
        (*callback) (data, &pcap, p->buffer);
      }

      // 如果设备有释放接收缓冲区的能力
      if (dev->release_rx_buf)
        (*dev->release_rx_buf) (p->buffer);

      // 如果启用了数据包调试
      if (pcap_pkt_debug > 0)
      {
        if (callback == watt32_recv_hook)
             dbug_write ("pcap_recv_hook\n");
        else dbug_write ("pcap_read_op\n");
      }
      FLUSHK();
      return (1);
    }

    /* 是否调用了 "pcap_breakloop()"? */
    # 如果标志位指示需要中断循环
    if (p->break_loop) {
      # 是的 - 清除指示已中断循环的标志，并返回 -2 表示我们被告知中断循环
      p->break_loop = 0;
      return (-2);
    }

    # 如果不需要等待数据包或者 pcap_cleanup_dos() 从 SIGINT 处理程序中调用，立即退出循环
    if (p->opt.timeout <= 0 || (volatile int)p->fd <= 0)
       break;

    # 获取当前时间
    gettimeofday2 (&now, NULL);

    # 如果当前时间超过了到期时间，退出循环
    if (timercmp(&now, &expiry, >))
       break;
#ifndef DJGPP
    kbhit();    /* 如果不是 DJGPP 编译器，则调用 kbhit() 函数，这个函数会占用大量 CPU 资源 */
#endif

    if (pd->wait_proc)
      (*pd->wait_proc)();     /* 如果 wait_proc 不为空，则调用对应的函数，这个函数用于让出 CPU 资源 */
  }

  if (rx_len < 0)            /* 如果接收出现错误 */
  {
    pd->stat.ps_drop++;      /* 增加接收错误的统计数量 */
#ifdef USE_32BIT_DRIVERS
    if (pcap_pkt_debug > 1)
       printk ("pkt-err %s\n", pktInfo.error);  /* 如果启用了 32 位驱动并且调试级别大于 1，则打印接收错误信息 */
#endif
    return (-1);  /* 返回接收错误 */
  }
  return (0);  /* 返回正常接收 */
}

static int
pcap_read_dos (pcap_t *p, int cnt, pcap_handler callback, u_char *data)
{
  int rc, num = 0;

  /*
   * 这个函数可能处理超过 INT_MAX 个数据包，这会导致数据包计数溢出，要么看起来像是负数，导致返回一个看起来像是错误的值，要么溢出回到正数领域，导致返回一个太低的计数。
   * 因此，如果数据包计数是无限的，我们将其截断为 INT_MAX；这个函数不应该无限处理数据包，所以这不是一个问题。
   */
  if (PACKET_COUNT_IS_UNLIMITED(cnt))
    cnt = INT_MAX;

  while (num <= cnt)
  {
    if (p->fd <= 0)
       return (-1);  /* 如果文件描述符小于等于 0，则返回错误 */
    rc = pcap_read_one (p, callback, data);  /* 调用 pcap_read_one 函数读取一个数据包 */
    if (rc > 0)
       num++;  /* 如果成功读取一个数据包，则计数加一 */
    if (rc < 0)
       break;  /* 如果读取出错，则跳出循环 */
    _w32_os_yield();  /* 允许生成 SIGINT 信号，让出 CPU 资源给 Win95/NT */
  }
  return (num);  /* 返回读取的数据包数量 */
}

/*
 * 返回网络统计信息
 */
static int pcap_stats_dos (pcap_t *p, struct pcap_stat *ps)
{
  struct net_device_stats *stats;
  struct pcap_dos         *pd;
  struct device           *dev = p ? get_device(p->fd) : NULL;

  if (!dev)
  {
    strcpy (p->errbuf, "illegal pcap handle");  /* 如果设备为空，则设置错误信息为非法 pcap 句柄 */
    return (-1);  /* 返回错误 */
  }

  if (!dev->get_stats || (stats = (*dev->get_stats)(dev)) == NULL)
  {
    strcpy (p->errbuf, "device statistics not available");  /* 如果设备没有获取统计信息的函数，或者获取统计信息失败，则设置错误信息为设备统计信息不可用 */
    # 返回-1，表示出错
    return (-1);
  }

  # 清空缓冲区
  FLUSHK();

  # 获取指向私有数据的指针
  pd = p->priv;
  # 设置接收数据包数和丢弃数据包数
  pd->stat.ps_recv   = stats->rx_packets;
  pd->stat.ps_drop  += stats->rx_missed_errors;
  # 设置接口丢弃数据包数，包括队列满和硬件错误
  pd->stat.ps_ifdrop = stats->rx_dropped +  /* queue full */
                         stats->rx_errors;    /* HW errors */
  # 如果ps不为空，将私有数据的统计信息拷贝到ps指向的位置
  if (ps)
     *ps = pd->stat;

  # 返回0，表示成功
  return (0);
/*
 * 返回详细的网络/设备统计信息。
 * 可能在调用 'dev->close' 后调用。
 */
int pcap_stats_ex (pcap_t *p, struct pcap_stat_ex *se)
{
  // 获取与 pcap_t 对象关联的设备
  struct device *dev = p ? get_device (p->fd) : NULL;

  // 如果设备不存在或者没有获取统计信息的函数，则返回错误
  if (!dev || !dev->get_stats)
  {
    pcap_strlcpy (p->errbuf, "detailed device statistics not available",
             PCAP_ERRBUF_SIZE);
    return (-1);
  }

  // 如果设备名称以 "pkt" 开头，则返回错误
  if (!strnicmp(dev->name,"pkt",3))
  {
    pcap_strlcpy (p->errbuf, "pktdrvr doesn't have detailed statistics",
             PCAP_ERRBUF_SIZE);
    return (-1);
  }
  // 复制设备的统计信息到指定的结构体中
  memcpy (se, (*dev->get_stats)(dev), sizeof(*se));
  return (0);
}

/*
 * 简单地存储 pcap_read_dos() 回调的过滤代码
 * 有一天，过滤代码可以传递给活动设备（pkt_rx1.s 或 32位设备中断处理程序）。
 */
static int pcap_setfilter_dos (pcap_t *p, struct bpf_program *fp)
{
  // 如果 pcap_t 对象不存在，则返回错误
  if (!p)
     return (-1);
  // 复制过滤代码到 pcap_t 对象中
  p->fcode = *fp;
  return (0);
}

/*
 * 返回 pcap_read_dos() 中接收的数据包数量
 */
u_long pcap_mac_packets (void)
{
  return (mac_count);
}

/*
 * 返回 pcap_read_dos() 中通过过滤器的数据包数量
 */
u_long pcap_filter_packets (void)
{
  return (filter_count);
}

/*
 * 关闭 pcap 设备。对于离线捕获不会调用此函数。
 */
static void pcap_cleanup_dos (pcap_t *p)
{
  struct pcap_dos *pd;

  // 如果没有发生异常
  if (!exc_occured)
  {
    // 获取与 pcap_t 对象关联的 pcap_dos 结构体
    pd = p->priv;
    // 如果获取设备的统计信息失败，则将丢弃的数据包数量设置为 0
    if (pcap_stats(p,NULL) < 0)
       pd->stat.ps_drop = 0;
    // 如果获取不到设备，则直接返回
    if (!get_device(p->fd))
       return;

    // 将设备句柄从全局数组中移除
    handle_to_device [p->fd-1] = NULL;
    p->fd = 0;
    // 如果引用计数大于 0，则递减引用计数
    if (ref_count > 0)
        ref_count--;
    // 如果引用计数大于 0，则直接返回
    if (ref_count > 0)
       return;
  }
  // 关闭驱动程序
  close_driver();
  /* XXX - call pcap_cleanup_live_common? */
}

/*
 * 返回第一个网络接口的名称，如果找不到则返回 NULL。
 */
char *pcap_lookupdev (char *ebuf)
{
  struct device *dev;

  // 初始化 32 位驱动程序
#ifdef USE_32BIT_DRIVERS
  init_32bit();
#endif

  // 遍历设备链表，查找第一个可用的网络接口
  for (dev = (struct device*)dev_base; dev; dev = dev->next)
  {
    // 确保设备有探测函数
    PCAP_ASSERT (dev->probe);

    // 如果设备可以被探测到，则返回设备名称
    if ((*dev->probe)(dev))
    {
      FLUSHK();  // 调用 FLUSHK 函数
      probed_dev = (struct device*) dev; // 将 dev 赋值给 probed_dev，作为最后一次探测到的设备
      return (char*) dev->name;  // 返回设备的名称
    }
  }

  if (ebuf)
     strcpy (ebuf, "No driver found");  // 如果 ebuf 不为空，则将 "No driver found" 复制到 ebuf 中
  return (NULL);  // 返回空指针
}

/*
 * 从 Watt-32 获取本地网络和子网掩码。
 */
int pcap_lookupnet (const char *device, bpf_u_int32 *localnet,
                    bpf_u_int32 *netmask, char *errbuf)
{
  DWORD mask, net;

  // 如果 Watt-32 尚未初始化，则返回错误信息
  if (!_watt_is_init)
  {
    strcpy (errbuf, "pcap_open_offline() or pcap_activate() must be "
                    "called first");
    return (-1);
  }

  // 获取子网掩码和网络地址
  mask  = _w32_sin_mask;
  net = my_ip_addr & mask;
  if (net == 0)
  {
    // 如果网络地址为 0，则根据子网掩码判断网络类别
    if (IN_CLASSA(*netmask))
       net = IN_CLASSA_NET;
    else if (IN_CLASSB(*netmask))
       net = IN_CLASSB_NET;
    else if (IN_CLASSC(*netmask))
       net = IN_CLASSC_NET;
    else
    {
      snprintf (errbuf, PCAP_ERRBUF_SIZE, "inet class for 0x%lx unknown", mask);
      return (-1);
    }
  }
  // 将网络地址和子网掩码转换为网络字节序
  *localnet = htonl (net);
  *netmask = htonl (mask);

  ARGSUSED (device);
  return (0);
}

/*
 * 获取所有存在且可探测的接口列表。
 * 出错时返回 -1，否则返回 0。
 * 如果没有接口处于启用状态且可打开，则列表可能为空。
 */
int pcap_platform_finddevs  (pcap_if_list_t *devlistp, char *errbuf)
{
  struct device     *dev;
  pcap_if_t *curdev;
#if 0   /* Pkt drivers should have no addresses */
  struct sockaddr_in sa_ll_1, sa_ll_2;
  struct sockaddr   *addr, *netmask, *broadaddr, *dstaddr;
#endif
  int       ret = 0;
  int       found = 0;

  // 遍历设备列表
  for (dev = (struct device*)dev_base; dev; dev = dev->next)
  {
    // 确保设备的探测函数存在
    PCAP_ASSERT (dev->probe);

    // 如果设备无法探测，则继续下一个设备
    if (!(*dev->probe)(dev))
       continue;

    // 确保设备的关闭函数存在，并调用关闭函数
    PCAP_ASSERT (dev->close);  /* set by probe routine */
    FLUSHK();
    (*dev->close) (dev);

    /*
     * XXX - find out whether it's up or running?  Does that apply here?
     * Can we find out if anything's plugged into the adapter, if it's
     * a wired device, and set PCAP_IF_CONNECTION_STATUS_CONNECTED
     * or PCAP_IF_CONNECTION_STATUS_DISCONNECTED?
     */
    // 将设备信息添加到设备列表中
    if ((curdev = add_dev(devlistp, dev->name, 0,
                dev->long_name, errbuf)) == NULL)
    {
      ret = -1;
      break;
    }
    # 初始化变量 found 为 1，表示找到了
    found = 1;
#if 0   /* Pkt drivers should have no addresses */
    // 如果为 0，则表示数据包驱动程序不应该有地址
    memset (&sa_ll_1, 0, sizeof(sa_ll_1));
    // 将 sa_ll_1 的内存内容全部置为 0
    memset (&sa_ll_2, 0, sizeof(sa_ll_2));
    // 将 sa_ll_2 的内存内容全部置为 0
    sa_ll_1.sin_family = AF_INET;
    // 设置 sa_ll_1 的地址族为 AF_INET
    sa_ll_2.sin_family = AF_INET;
    // 设置 sa_ll_2 的地址族为 AF_INET

    addr      = (struct sockaddr*) &sa_ll_1;
    // 将 addr 指向 sa_ll_1 的地址
    netmask   = (struct sockaddr*) &sa_ll_1;
    // 将 netmask 指向 sa_ll_1 的地址
    dstaddr   = (struct sockaddr*) &sa_ll_1;
    // 将 dstaddr 指向 sa_ll_1 的地址
    broadaddr = (struct sockaddr*) &sa_ll_2;
    // 将 broadaddr 指向 sa_ll_2 的地址
    memset (&sa_ll_2.sin_addr, 0xFF, sizeof(sa_ll_2.sin_addr));
    // 将 sa_ll_2 的地址全部置为 0xFF

    if (add_addr_to_dev(curdev, addr, sizeof(*addr),
                        netmask, sizeof(*netmask),
                        broadaddr, sizeof(*broadaddr),
                        dstaddr, sizeof(*dstaddr), errbuf) < 0)
    {
      ret = -1;
      break;
    }
#endif
  }
  // 结束 if 0 代码块

  if (ret == 0 && !found)
     strcpy (errbuf, "No drivers found");
  // 如果 ret 为 0 且未找到驱动程序，则将 "No drivers found" 复制到 errbuf 中

  return (ret);
  // 返回 ret 的值
}

/*
 * pcap_assert() is mainly used for debugging
 */
void pcap_assert (const char *what, const char *file, unsigned line)
{
  FLUSHK();
  // 刷新键盘缓冲区
  fprintf (stderr, "%s (%u): Assertion \"%s\" failed\n",
           file, line, what);
  // 输出断言失败的信息
  close_driver();
  // 关闭驱动程序
  _exit (-1);
  // 退出程序
}

/*
 * For pcap_offline_read(): wait and yield between printing packets
 * to simulate the pace packets where actually recorded.
 */
void pcap_set_wait (pcap_t *p, void (*yield)(void), int wait)
{
  if (p)
  {
    struct pcap_dos *pd = p->priv;
    // 获取 pcap_t 结构体中的私有数据

    pd->wait_proc  = yield;
    // 设置 pd 的等待处理函数为 yield
    p->opt.timeout = wait;
    // 设置 p 的超时时间为 wait
  }
}

/*
 * Initialise a named network device.
 */
static struct device *
open_driver (const char *dev_name, char *ebuf, int promisc)
{
  struct device *dev;
  // 定义一个设备结构体指针 dev

  for (dev = (struct device*)dev_base; dev; dev = dev->next)
  {
    PCAP_ASSERT (dev->name);
    // 断言 dev 的名称不为空

    if (strcmp (dev_name,dev->name))
       continue;
    // 如果 dev_name 与 dev 的名称不相等，则继续循环

    if (!probed_dev)   /* user didn't call pcap_lookupdev() first */
    // 如果没有探测到设备，则执行以下代码
    {
      # 断言设备的探测函数不为空
      PCAP_ASSERT (dev->probe);

      # 如果设备的探测函数返回值为假
      if (!(*dev->probe)(dev))    /* call the xx_probe() function */
      {
        # 格式化错误信息到错误缓冲区
        snprintf (ebuf, PCAP_ERRBUF_SIZE, "failed to detect device `%s'", dev_name);
        # 返回空指针
        return (NULL);
      }
      # 设备已经探测成功，可以使用
      probed_dev = dev;  /* device is probed okay and may be used */
    }
    # 如果设备不等于已探测的设备
    else if (dev != probed_dev)
    {
      # 跳转到未探测标签
      goto not_probed;
    }

    # 刷新内核缓冲区
    FLUSHK();

    # 选择接收的流量
    if (promisc)
         dev->flags |=  (IFF_ALLMULTI | IFF_PROMISC);
    else dev->flags &= ~(IFF_ALLMULTI | IFF_PROMISC);

    # 断言设备的打开函数不为空
    PCAP_ASSERT (dev->open);

    # 如果设备的打开函数返回值为假
    if (!(*dev->open)(dev))
    {
      # 格式化错误信息到错误缓冲区
      snprintf (ebuf, PCAP_ERRBUF_SIZE, "failed to activate device `%s'", dev_name);
      # 如果存在数据包信息的错误并且设备名称以"pkt"开头
      if (pktInfo.error && !strncmp(dev->name,"pkt",3))
      {
        # 追加错误信息到错误缓冲区
        strcat (ebuf, ": ");
        strcat (ebuf, pktInfo.error);
      }
      # 返回空指针
      return (NULL);
    }

    # 一些设备需要这个操作来在混杂模式下操作
    if (promisc && dev->set_multicast_list)
       (*dev->set_multicast_list) (dev);

    # 记住我们的活动设备
    active_dev = dev;   /* remember our active device */
    # 跳出循环
    break;
  }

  # 在设备基本列表中未匹配到'dev_name'
  if (!dev)
  {
    # 格式化错误信息到错误缓冲区
    snprintf (ebuf, PCAP_ERRBUF_SIZE, "device `%s' not supported", dev_name);
    # 返回空指针
    return (NULL);
  }
not_probed:
  # 如果设备没有被探测到，则返回错误信息
  if (!probed_dev)
  {
    snprintf (ebuf, PCAP_ERRBUF_SIZE, "device `%s' not probed", dev_name);
    return (NULL);
  }
  # 返回设备
  return (dev);
}

/*
 * 反初始化 MAC 驱动程序。
 * 将接收模式设置回默认模式。
 */
static void close_driver (void)
{
  /* !!todo: loop over all 'handle_to_device[]' ? */
  # 获取当前活动设备
  struct device *dev = active_dev;

  # 如果设备存在且有关闭函数，则调用关闭函数
  if (dev && dev->close)
  {
    (*dev->close) (dev);
    FLUSHK();
  }

  # 将活动设备置为空
  active_dev = NULL;

#ifdef USE_32BIT_DRIVERS
  # 如果 rx_pool 存在，则释放其内存并置为空
  if (rx_pool)
  {
    k_free (rx_pool);
    rx_pool = NULL;
  }
  # 如果设备存在，则调用 pcibios_exit() 函数
  if (dev)
     pcibios_exit();
#endif
}


#ifdef __DJGPP__
# 设置信号处理函数
static void setup_signals (void (*handler)(int))
{
  signal (SIGSEGV,handler);
  signal (SIGILL, handler);
  signal (SIGFPE, handler);
}

# 异常处理函数
static void exc_handler (int sig)
{
#ifdef USE_32BIT_DRIVERS
  # 如果活动设备的中断号大于 0，则执行以下操作
  if (active_dev->irq > 0)    /* excludes IRQ 0 */
  {
    disable_irq (active_dev->irq);
    irq_eoi_cmd (active_dev->irq);
    _printk_safe = 1;
  }
#endif

  # 根据信号类型执行相应的操作
  switch (sig)
  {
    case SIGSEGV:
         fputs ("Catching SIGSEGV.\n", stderr);
         break;
    case SIGILL:
         fputs ("Catching SIGILL.\n", stderr);
         break;
    case SIGFPE:
         _fpreset();
         fputs ("Catching SIGFPE.\n", stderr);
         break;
    default:
         fprintf (stderr, "Catching signal %d.\n", sig);
  }
  # 设置异常发生标志为 1
  exc_occured = 1;
  # 调用关闭驱动函数
  close_driver();
}
#endif  /* __DJGPP__ */


/*
 * 为第一个调用 pcap_activate() 的客户端打开 pcap 设备
 */
static int first_init (const char *name, char *ebuf, int promisc)
{
  struct device *dev;

#ifdef USE_32BIT_DRIVERS
  # 分配接收缓冲区内存
  rx_pool = k_calloc (RECEIVE_BUF_SIZE, RECEIVE_QUEUE_SIZE);
  # 如果内存分配失败，则返回错误信息
  if (!rx_pool)
  {
    strcpy (ebuf, "Not enough memory (Rx pool)");
    return (0);
  }
#endif

#ifdef __DJGPP__
  # 设置信号处理函数
  setup_signals (exc_handler);
#endif

#ifdef USE_32BIT_DRIVERS
  # 初始化 32 位驱动程序
  init_32bit();
#endif

  # 打开驱动程序
  dev = open_driver (name, ebuf, promisc);
  # 如果打开失败，则执行以下操作
  if (!dev)
  {
#ifdef USE_32BIT_DRIVERS
    # 释放 rx_pool 内存并置为空
    k_free (rx_pool);
    rx_pool = NULL;
#endif

#ifdef __DJGPP__
    # 设置信号处理函数为默认处理方式
    setup_signals (SIG_DFL);
#endif
    return (0);
  }

#ifdef USE_32BIT_DRIVERS
  '''
   * 如果驱动程序不是16位的“pkt/ndis”驱动程序（在其探测处理程序中设置了'copy_rx_buf'），则为32位设备初始化近内存环形缓冲区。
   '''
  if (dev->copy_rx_buf == NULL)
  {
    dev->get_rx_buf     = get_rxbuf;
    dev->peek_rx_buf    = peek_rxbuf;
    dev->release_rx_buf = release_rxbuf;
    pktq_init (&dev->queue, RECEIVE_BUF_SIZE, RECEIVE_QUEUE_SIZE, rx_pool);
  }
#endif
  return (1);
}

#ifdef USE_32BIT_DRIVERS
static void init_32bit (void)
{
  static int init_pci = 0;

  if (!_printk_file)
     _printk_init (64*1024, NULL); /* 调用 atexit(printk_exit) */

  if (!init_pci)
     (void)pci_init();             /* 初始化 BIOS32+PCI 接口 */
  init_pci = 1;
}
#endif


'''
 * 用于与 pcap 一起使用 Watt-32 的挂钩函数
 '''
static char rxbuf [ETH_MAX+100]; /* 带有一些余量的接收缓冲区 */
static WORD etype;
static pcap_t pcap_save;

static void watt32_recv_hook (u_char *dummy, const struct pcap_pkthdr *pcap,
                              const u_char *buf)
{
  '''修复我：假设仅适用以太网 II'''
  struct ether_header *ep = (struct ether_header*) buf;

  memcpy (rxbuf, buf, pcap->caplen);
  etype = ep->ether_type;
  ARGSUSED (dummy);
}

#if (WATTCP_VER >= 0x0224)
'''
 * Watt-32 用于轮询数据包的函数。
 * 即它被设置为绕过 _eth_arrived()
 '''
static void *pcap_recv_hook (WORD *type)
{
  int len = pcap_read_dos (&pcap_save, 1, watt32_recv_hook, NULL);

  if (len < 0)
     return (NULL);

  *type = etype;
  return (void*) &rxbuf;
}

'''
 * 此函数由 Watt-32 调用（通过 _eth_xmit_hook）。
 * 如果调用了 dbug_init()，我们应该跟踪发送的数据包。
 '''
static int pcap_xmit_hook (const void *buf, unsigned len)
{
  # 定义变量 rc 并初始化为 0
  int rc = 0;

  # 如果 pcap_pkt_debug 大于 0，则调用 dbug_write 函数输出信息
  if (pcap_pkt_debug > 0)
     dbug_write ("pcap_xmit_hook: ");

  # 如果 active_dev 存在且 active_dev->xmit 存在，则调用 active_dev->xmit 函数发送数据包
  if (active_dev && active_dev->xmit)
     if ((*active_dev->xmit) (active_dev, buf, len) > 0)
        rc = len;

  # 如果 pcap_pkt_debug 大于 0，则根据 rc 的值调用 dbug_write 函数输出信息
  if (pcap_pkt_debug > 0)
     dbug_write (rc ? "ok\n" : "fail\n");
  # 返回发送数据包的结果
  return (rc);
}
#endif

# 定义函数 pcap_sendpacket_dos，用于发送数据包
static int pcap_sendpacket_dos (pcap_t *p, const void *buf, size_t len)
{
  # 获取与 pcap_t 对应的设备结构体
  struct device *dev = p ? get_device(p->fd) : NULL;

  # 如果设备结构体不存在或者设备结构体的 xmit 函数不存在，则返回 -1
  if (!dev || !dev->xmit)
     return (-1);
  # 调用设备结构体的 xmit 函数发送数据包
  return (*dev->xmit) (dev, buf, len);
}

# 定义函数 pcap_init_hook，用于初始化 pcap
static void pcap_init_hook (void)
{
  # 关闭 BOOTP/DHCP/RARP 等功能
  _w32__bootp_on = _w32__dhcp_on = _w32__rarp_on = 0;
  _w32__do_mask_req = 0;
  _w32_dynamic_host = 0;
  # 如果存在前一个初始化钩子函数，则调用它
  if (prev_post_hook)
    (*prev_post_hook)();
}

# 定义函数 null_print，用于屏蔽来自 sock_init() 的 PRINT 消息
static void null_print (void) {}

# 定义函数 init_watt32，用于初始化 Watt-32
static int init_watt32 (struct pcap *pcap, const char *dev_name, char *err_buf)
{
  char *env;
  int   rc, MTU, has_ip_addr;
  int   using_pktdrv = 1;

  # 如果 _watt_is_init 已经初始化，则调用 sock_exit() 重新初始化
  if (_watt_is_init)
     sock_exit();

  # 获取环境变量 PCAP_TRACE 的值，如果大于 0 并且 pcap_pkt_debug 小于 0，则初始化调试输出
  env = getenv ("PCAP_TRACE");
  if (env && atoi(env) > 0 &&
      pcap_pkt_debug < 0)   /* if not already set */
  {
    dbug_init();
  // 将环境变量转换为整数，并赋值给 pcap_pkt_debug
  pcap_pkt_debug = atoi (env);

  // 防止 sock_init() 调用 exit()
  _watt_do_exit      = 0;
  // 保存先前的 post_hook 函数指针
  prev_post_hook     = _w32_usr_post_init;
  // 将 pcap_init_hook 函数指针赋值给 _w32_usr_post_init
  _w32_usr_post_init = pcap_init_hook;
  // 将 null_print 函数指针赋值给 _w32_print_hook
  _w32_print_hook    = null_print;

  // 如果 dev_name 存在且不以 "pkt" 开头，则将 using_pktdrv 置为 FALSE
  if (dev_name && strncmp(dev_name,"pkt",3))
     using_pktdrv = FALSE;

  // 初始化套接字库，将返回值赋给 rc
  rc = sock_init();
  // 判断 IP 地址是否分配成功，并将结果赋给 has_ip_addr
  has_ip_addr = (rc != 8);

  // 如果不使用 pktdrv 或者 IP 地址未分配成功
  if (!using_pktdrv || !has_ip_addr)
  {
    // 设置默认的 IP 地址和子网掩码
    static const char myip[] = "192.168.0.1";
    static const char mask[] = "255.255.255.0";
    printf ("Just guessing, using IP %s and netmask %s\n", myip, mask);
    // 将默认的 IP 地址转换为网络字节序并赋值给 my_ip_addr
    my_ip_addr    = aton (myip);
    // 将默认的子网掩码转换为网络字节序并赋值给 _w32_sin_mask
    _w32_sin_mask = aton (mask);
  }
  // 如果 rc 不为 0 且使用了 pktdrv
  else if (rc && using_pktdrv)
  {
    // 将错误信息格式化并存储到 err_buf 中
    snprintf (err_buf, PCAP_ERRBUF_SIZE, "sock_init() failed, code %d", rc);
    // 返回 0
    return (0);
  }

  // 设置 _eth_arrived() 的接收钩子函数
#if (WATTCP_VER >= 0x0224)
  # 如果 WATTCP 版本大于等于 0x0224，则设置以太网接收和发送钩子为 pcap_recv_hook 和 pcap_xmit_hook
  _eth_recv_hook = pcap_recv_hook;
  _eth_xmit_hook = pcap_xmit_hook;
#endif

  /* 释放在 pkt_init() 中分配的 pkt-drvr 句柄。
   * 上述钩子应该使用在 open_driver() 中重新打开的句柄
   */
  if (using_pktdrv)
  {
    _eth_release();
/*  _eth_is_init = 1; */  /* hack to get Rx/Tx-hooks in Watt-32 working */
  }

  // 复制 pcap 结构体的内容到 pcap_save 结构体
  memcpy (&pcap_save, pcap, sizeof(pcap_save));
  // 获取 MTU 值
  MTU = pkt_get_mtu();
  // 设置 pcap_save 结构体的 bf_insns 为 NULL
  pcap_save.fcode.bf_insns = NULL;
  // 获取以太网硬件类型
  pcap_save.linktype       = _eth_get_hwtype (NULL, NULL);
  // 设置快照长度为 MTU 或者 ETH_MAX（假设为 1514）
  pcap_save.snapshot       = MTU > 0 ? MTU : ETH_MAX;

  // 防止使用 resolve() 和 resolve_ip()
  last_nameserver = 0;
  // 返回 1
  return (1);
}

// 设置 EISA_bus 变量为 0
int EISA_bus = 0;  /* Where is natural place for this? */

/*
 * 应用程序配置钩子，用于设置各种驱动程序参数。
 */
static const struct config_table debug_tab[] = {
            // 设置 pcap_pkt_debug 变量
            { "PKT.DEBUG",       ARG_ATOI,   &pcap_pkt_debug    },
            // 设置 NULL
            { "PKT.VECTOR",      ARG_ATOX_W, NULL               },
            // 设置 NULL
            { "NDIS.DEBUG",      ARG_ATOI,   NULL               },
#ifdef USE_32BIT_DRIVERS
            { "3C503.DEBUG",     ARG_ATOI,   &ei_debug          },  // 设置 3C503 网卡的调试标志
            { "3C503.IO_BASE",   ARG_ATOX_W, &el2_dev.base_addr },  // 设置 3C503 网卡的 I/O 基地址
            { "3C503.MEMORY",    ARG_ATOX_W, &el2_dev.mem_start },  // 设置 3C503 网卡的内存地址
            { "3C503.IRQ",       ARG_ATOI,   &el2_dev.irq       },  // 设置 3C503 网卡的中断请求
            { "3C505.DEBUG",     ARG_ATOI,   NULL               },  // 设置 3C505 网卡的调试标志
            { "3C505.BASE",      ARG_ATOX_W, NULL               },  // 设置 3C505 网卡的基地址
            { "3C507.DEBUG",     ARG_ATOI,   NULL               },  // 设置 3C507 网卡的调试标志
            { "3C509.DEBUG",     ARG_ATOI,   &el3_debug         },  // 设置 3C509 网卡的调试标志
            { "3C509.ILOOP",     ARG_ATOI,   &el3_max_loop      },  // 设置 3C509 网卡的最大循环数
            { "3C529.DEBUG",     ARG_ATOI,   NULL               },  // 设置 3C529 网卡的调试标志
            { "3C575.DEBUG",     ARG_ATOI,   &debug_3c575       },  // 设置 3C575 网卡的调试标志
            { "3C59X.DEBUG",     ARG_ATOI,   &vortex_debug      },  // 设置 3C59X 网卡的调试标志
            { "3C59X.IFACE0",    ARG_ATOI,   &vortex_options[0] },  // 设置 3C59X 网卡的接口0
            { "3C59X.IFACE1",    ARG_ATOI,   &vortex_options[1] },  // 设置 3C59X 网卡的接口1
            { "3C59X.IFACE2",    ARG_ATOI,   &vortex_options[2] },  // 设置 3C59X 网卡的接口2
            { "3C59X.IFACE3",    ARG_ATOI,   &vortex_options[3] },  // 设置 3C59X 网卡的接口3
            { "3C90X.DEBUG",     ARG_ATOX_W, &tc90xbc_debug     },  // 设置 3C90X 网卡的调试标志
            { "ACCT.DEBUG",      ARG_ATOI,   &ethpk_debug       },  // 设置 ACCT 网卡的调试标志
            { "CS89.DEBUG",      ARG_ATOI,   &cs89_debug        },  // 设置 CS89 网卡的调试标志
            { "RTL8139.DEBUG",   ARG_ATOI,   &rtl8139_debug     },  // 设置 RTL8139 网卡的调试标志
        /*  { "RTL8139.FDUPLEX", ARG_ATOI,   &rtl8139_options   }, */  // 设置 RTL8139 网卡的全双工模式
            { "SMC.DEBUG",       ARG_ATOI,   &ei_debug          },  // 设置 SMC 网卡的调试标志
        /*  { "E100.DEBUG",      ARG_ATOI,   &e100_debug        }, */  // 设置 E100 网卡的调试标志
            { "PCI.DEBUG",       ARG_ATOI,   &pci_debug         },  // 设置 PCI 网卡的调试标志
            { "BIOS32.DEBUG",    ARG_ATOI,   &bios32_debug      },  // 设置 BIOS32 的调试标志
            { "IRQ.DEBUG",       ARG_ATOI,   &irq_debug         },  // 设置中断请求的调试标志
            { "TIMER.IRQ",       ARG_ATOI,   &timer_irq         },  // 设置定时器的中断请求
#endif
            { NULL }  // 结束标志
          };
/*
 * pcap_config_hook()是应用程序配置处理的扩展。使用Watt-32的配置表函数。
 */
int pcap_config_hook (const char *keyword, const char *value)
{
  return parse_config_table (debug_tab, NULL, keyword, value);
}

/*
 * 支持设备的链表
 */
struct device       *active_dev = NULL;      /* 我们已经打开的设备 */
struct device       *probed_dev = NULL;      /* 我们已经探测的设备 */
const struct device *dev_base   = &pkt_dev;  /* 网络设备列表 */

/*
 * PKTDRVR设备函数
 */
int pcap_pkt_debug = -1;

static void pkt_close (struct device *dev)
{
  BOOL okay = PktExitDriver();

  if (pcap_pkt_debug > 1)
     fprintf (stderr, "pkt_close(): %d\n", okay);

  if (dev->priv)
     free (dev->priv);
  dev->priv = NULL;
}

static int pkt_open (struct device *dev)
{
  PKT_RX_MODE mode;

  if (dev->flags & IFF_PROMISC)
       mode = PDRX_ALL_PACKETS;
  else mode = PDRX_BROADCAST;

  if (!PktInitDriver(mode))
     return (0);

  PktResetStatistics (pktInfo.handle);
  PktQueueBusy (FALSE);
  return (1);
}

static int pkt_xmit (struct device *dev, const void *buf, int len)
{
  struct net_device_stats *stats = (struct net_device_stats*) dev->priv;

  if (pcap_pkt_debug > 0)
     dbug_write ("pcap_xmit\n");

  if (!PktTransmit(buf,len))
  {
    stats->tx_errors++;
    return (0);
  }
  return (len);
}

static void *pkt_stats (struct device *dev)
{
  struct net_device_stats *stats = (struct net_device_stats*) dev->priv;

  if (!stats || !PktSessStatistics(pktInfo.handle))
     return (NULL);

  stats->rx_packets       = pktStat.inPackets;
  stats->rx_errors        = pktStat.lost;
  stats->rx_missed_errors = PktRxDropped();
  return (stats);
}

static int pkt_probe (struct device *dev)
{
  // 如果没有找到网络包搜索驱动，则返回 0
  if (!PktSearchDriver())
     return (0);

  // 设置设备的打开、发送、关闭、获取统计信息等函数指针
  dev->open           = pkt_open;
  dev->xmit           = pkt_xmit;
  dev->close          = pkt_close;
  dev->get_stats      = pkt_stats;
  dev->copy_rx_buf    = PktReceive;  /* farmem peek and copy routine */
  dev->get_rx_buf     = NULL;
  dev->peek_rx_buf    = NULL;
  dev->release_rx_buf = NULL;
  // 为设备的私有数据分配内存
  dev->priv           = calloc (sizeof(struct net_device_stats), 1);
  // 如果内存分配失败，则返回 0
  if (!dev->priv)
     return (0);
  // 返回 1
  return (1);
}

/*
 * NDIS device functions
 */
// 关闭 NDIS 设备
static void ndis_close (struct device *dev)
{
#ifdef USE_NDIS2
  NdisShutdown();
#endif
  ARGSUSED (dev);
}

// 打开 NDIS 设备
static int ndis_open (struct device *dev)
{
  // 检查设备是否处于混杂模式
  int promisc = (dev->flags & IFF_PROMISC);

#ifdef USE_NDIS2
  // 如果 NDIS 初始化失败，则返回 0
  if (!NdisInit(promisc))
     return (0);
  // 返回 1
  return (1);
#else
  ARGSUSED (promisc);
  // 返回 0
  return (0);
#endif
}

// 获取 NDIS 设备的统计信息
static void *ndis_stats (struct device *dev)
{
  // 静态变量，用于存储设备的统计信息
  static struct net_device_stats stats;

  /* to-do */
  ARGSUSED (dev);
  // 返回设备的统计信息
  return (&stats);
}

// 探测 NDIS 设备
static int ndis_probe (struct device *dev)
{
#ifdef USE_NDIS2
  // 如果 NDIS 打开失败，则返回 0
  if (!NdisOpen())
     return (0);
#endif

  // 设置设备的打开、关闭、获取统计信息等函数指针
  dev->open           = ndis_open;
  dev->xmit           = NULL;
  dev->close          = ndis_close;
  dev->get_stats      = ndis_stats;
  dev->copy_rx_buf    = NULL;       /* to-do */
  dev->get_rx_buf     = NULL;       /* upcall is from rmode driver */
  dev->peek_rx_buf    = NULL;
  dev->release_rx_buf = NULL;
  // 返回 0
  return (0);
}

/*
 * Search & probe for supported 32-bit (pmode) pcap devices
 */
// 如果使用 32 位驱动，则搜索和探测支持的设备
#if defined(USE_32BIT_DRIVERS)

// 定义 EtherLink II 设备结构
struct device el2_dev LOCKED_VAR = {
              "3c503",
              "EtherLink II",
              0,
              0,0,0,0,0,0,
              NULL,
              el2_probe
            };

// 定义 EtherLink III 设备结构
struct device el3_dev LOCKED_VAR = {
              "3c509",
              "EtherLink III",
              0,
              0,0,0,0,0,0,
              &el2_dev,
              el3_probe
            };
# 定义名为 tc515_dev 的设备结构体，包含设备名称、描述、以及指向其他设备结构体的指针和设备探测函数
struct device tc515_dev LOCKED_VAR = {
              "3c515",
              "EtherLink PCI",
              0,
              0,0,0,0,0,0,
              &el3_dev,
              tc515_probe
            };

# 定义名为 tc59_dev 的设备结构体，包含设备名称、描述、以及指向其他设备结构体的指针和设备探测函数
struct device tc59_dev LOCKED_VAR = {
              "3c59x",
              "EtherLink PCI",
              0,
              0,0,0,0,0,0,
              &tc515_dev,
              tc59x_probe
            };

# 定义名为 tc90xbc_dev 的设备结构体，包含设备名称、描述、以及指向其他设备结构体的指针和设备探测函数
struct device tc90xbc_dev LOCKED_VAR = {
              "3c90x",
              "EtherLink 90X",
              0,
              0,0,0,0,0,0,
              &tc59_dev,
              tc90xbc_probe
            };

# 定义名为 wd_dev 的设备结构体，包含设备名称、描述、以及指向其他设备结构体的指针和设备探测函数
struct device wd_dev LOCKED_VAR = {
              "wd",
              "Westen Digital",
              0,
              0,0,0,0,0,0,
              &tc90xbc_dev,
              wd_probe
            };

# 定义名为 ne_dev 的设备结构体，包含设备名称、描述、以及指向其他设备结构体的指针和设备探测函数
struct device ne_dev LOCKED_VAR = {
              "ne",
              "NEx000",
              0,
              0,0,0,0,0,0,
              &wd_dev,
              ne_probe
            };

# 定义名为 acct_dev 的设备结构体，包含设备名称、描述、以及指向其他设备结构体的指针和设备探测函数
struct device acct_dev LOCKED_VAR = {
              "acct",
              "Accton EtherPocket",
              0,
              0,0,0,0,0,0,
              &ne_dev,
              ethpk_probe
            };

# 定义名为 cs89_dev 的设备结构体，包含设备名称、描述、以及指向其他设备结构体的指针和设备探测函数
struct device cs89_dev LOCKED_VAR = {
              "cs89",
              "Crystal Semiconductor",
              0,
              0,0,0,0,0,0,
              &acct_dev,
              cs89x0_probe
            };

# 定义名为 rtl8139_dev 的设备结构体，包含设备名称、描述、以及指向其他设备结构体的指针和设备探测函数
struct device rtl8139_dev LOCKED_VAR = {
              "rtl8139",
              "RealTek PCI",
              0,
              0,0,0,0,0,0,
              &cs89_dev,
              rtl8139_probe     /* dev->probe routine */
            };

# 定义名为 peek_rxbuf 的函数，用于从接收缓冲区中获取数据
# 注意：队列元素不会被复制，只会返回指向队列元素的指针
int peek_rxbuf (BYTE **buf)
{
  // 定义指向接收队列头尾的指针
  struct rx_elem *tail, *head;

  // 检查接收队列是否有效
  PCAP_ASSERT (pktq_check (&active_dev->queue));

  // 禁用中断
  DISABLE();
  // 从接收队列中取出一个元素
  tail = pktq_out_elem (&active_dev->queue);
  // 获取接收队列中的第一个元素
  head = pktq_in_elem (&active_dev->queue);
  // 启用中断
  ENABLE();

  // 如果队列不为空
  if (head != tail)
  {
    // 检查队尾元素的大小是否小于队列元素大小减去4和2
    PCAP_ASSERT (tail->size < active_dev->queue.elem_size-4-2);

    // 将指向队尾数据的指针赋值给buf
    *buf = &tail->data[0];
    // 返回队尾元素的大小
    return (tail->size);
  }
  // 如果队列为空，将buf指向空，并返回0
  *buf = NULL;
  return (0);
}

/*
 * 释放上面查看的缓冲区
 */
int release_rxbuf (BYTE *buf)
{
#ifndef NDEBUG
  // 从接收队列中取出一个元素
  struct rx_elem *tail = pktq_out_elem (&active_dev->queue);
  // 断言buf指向队尾数据的指针
  PCAP_ASSERT (&tail->data[0] == buf);
#else
  // 如果是发布版本，使用ARGSUSED宏
  ARGSUSED (buf);
#endif
  // 增加队尾索引
  pktq_inc_out (&active_dev->queue);
  return (1);
}

/*
 * get_rxbuf()例程（在锁定代码中）从IRQ处理程序中调用以请求缓冲区。
 * 中断被禁用，我们有32kB的堆栈。
 */
BYTE *get_rxbuf (int len)
{
  int idx;

  // 如果长度小于最小以太网帧长度或大于最大以太网帧长度，返回空
  if (len < ETH_MIN || len > ETH_MAX)
     return (NULL);

  // 获取接收队列的输入索引
  idx = pktq_in_index (&active_dev->queue);

#ifdef DEBUG
  {
    // 静态变量fan_idx LOCKED_VAR初始化为0
    static int fan_idx LOCKED_VAR = 0;
    // 在屏幕上显示动画
    writew ("-\\|/"[fan_idx++] | (15 << 8),      /* white on black colour */
            0xB8000 + 2*79);  /* upper-right corner, 80-col colour screen */
    fan_idx &= 3;
  }
/* writew (idx + '0' + 0x0F00, 0xB8000 + 2*78); */
#endif

  // 如果输入索引不等于队尾索引
  if (idx != active_dev->queue.out_index)
  {
    // 获取接收队列的头元素
    struct rx_elem *head = pktq_in_elem (&active_dev->queue);
    // 设置头元素的大小为len
    head->size = len;
    // 设置队列的输入索引为idx
    active_dev->queue.in_index = idx;
    // 返回指向头元素数据的指针
    return (&head->data[0]);
  }

  /* !!to-do: drop 25% of the oldest element
   */
  // 清空接收队列
  pktq_clear (&active_dev->queue);
  return (NULL);
}

/*
 * 用于接收网络驱动程序的数据包的简单环形缓冲区队列处理程序。
 */
#define PKTQ_MARKER  0xDEADBEEF

static int pktq_check (struct rx_ringbuf *q)
{
#ifndef NDEBUG
  int   i;
  char *buf;
#endif

  // 如果队列为空或者未初始化，返回0
  if (!q || !q->num_elem || !q->buf_start)
     return (0);

#ifndef NDEBUG
  // 初始化buf为队列的起始地址
  buf = q->buf_start;

  // 遍历队列中的元素
  for (i = 0; i < q->num_elem; i++)
  {
    // 更新buf指向下一个元素
    buf += q->elem_size;
    # 检查缓冲区中前一个 DWORD 大小的数据是否等于 PKTQ_MARKER
    if (*(DWORD*)(buf - sizeof(DWORD)) != PKTQ_MARKER)
       # 如果不相等，则返回 0
       return (0);
  }
#endif
  return (1);
}

static int pktq_init (struct rx_ringbuf *q, int size, int num, char *pool)
{
  int i;

  q->elem_size = size;  # 设置队列元素的大小
  q->num_elem  = num;   # 设置队列中元素的数量
  q->buf_start = pool;  # 设置队列的起始位置
  q->in_index  = 0;     # 设置队列的输入索引
  q->out_index = 0;     # 设置队列的输出索引

  PCAP_ASSERT (size >= sizeof(struct rx_elem) + sizeof(DWORD));  # 断言队列元素大小大于等于结构体 rx_elem 和 DWORD 的大小之和
  PCAP_ASSERT (num);  # 断言队列元素数量不为0
  PCAP_ASSERT (pool);  # 断言队列起始位置不为空

  for (i = 0; i < num; i++)
  {
#if 0
    struct rx_elem *elem = (struct rx_elem*) pool;

    /* assert dword aligned elements
     */
    PCAP_ASSERT (((unsigned)(&elem->data[0]) & 3) == 0);
#endif
    pool += size;  # 更新队列起始位置
    *(DWORD*) (pool - sizeof(DWORD)) = PKTQ_MARKER;  # 在队列起始位置的前一个位置设置 PKTQ_MARKER
  }
  return (1);
}

/*
 * Increment the queue 'out_index' (tail).
 * Check for wraps.
 */
static int pktq_inc_out (struct rx_ringbuf *q)
{
  q->out_index++;  # 输出索引加1
  if (q->out_index >= q->num_elem)  # 如果输出索引超过了队列元素数量
      q->out_index = 0;  # 则将输出索引重置为0
  return (q->out_index);  # 返回输出索引
}

/*
 * Return the queue's next 'in_index' (head).
 * Check for wraps.
 */
static int pktq_in_index (struct rx_ringbuf *q)
{
  volatile int index = q->in_index + 1;  # 将输入索引加1赋值给临时变量 index

  if (index >= q->num_elem)  # 如果临时变量 index 超过了队列元素数量
      index = 0;  # 则将临时变量 index 重置为0
  return (index);  # 返回临时变量 index
}

/*
 * Return the queue's head-buffer.
 */
static struct rx_elem *pktq_in_elem (struct rx_ringbuf *q)
{
  return (struct rx_elem*) (q->buf_start + (q->elem_size * q->in_index));  # 返回队列头部的缓冲区
}

/*
 * Return the queue's tail-buffer.
 */
static struct rx_elem *pktq_out_elem (struct rx_ringbuf *q)
{
  return (struct rx_elem*) (q->buf_start + (q->elem_size * q->out_index));  # 返回队列尾部的缓冲区
}

/*
 * Clear the queue ring-buffer by setting head=tail.
 */
static void pktq_clear (struct rx_ringbuf *q)
{
  q->in_index = q->out_index;  # 通过将输入索引设置为输出索引来清空队列环形缓冲区
}

/*
 * Symbols that must be linkable for "gcc -O0"
 */
#undef __IOPORT_H  # 取消定义 __IOPORT_H 符号
#undef __DMA_H  # 取消定义 __DMA_H 符号

#define extern  # 定义 extern 符号为空
#define __inline__  # 定义 __inline__ 符号为空

#include "msdos/pm_drvr/ioport.h"  # 包含 msdos/pm_drvr/ioport.h 头文件
#include "msdos/pm_drvr/dma.h"  # 包含 msdos/pm_drvr/dma.h 头文件

#endif /* USE_32BIT_DRIVERS */

/*
 * Libpcap version string.
 */
const char *
pcap_lib_version(void)
{
  return ("DOS-" PCAP_VERSION_STRING);  # 返回 Libpcap 版本字符串
}
```