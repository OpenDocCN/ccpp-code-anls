# `nmap\libpcap\pcap-septel.c`

```
/*
 * pcap-septel.c: Packet capture interface for Intel/Septel card.
 *
 * Authors: Gilbert HOYEK (gil_hoyek@hotmail.com), Elias M. KHOURY
 * (+961 3 485243)
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <sys/param.h>

#include <stdlib.h>
#include <string.h>
#include <errno.h>

#include "pcap-int.h"

#include <netinet/in.h>
#include <sys/mman.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>

#include <msg.h>
#include <ss7_inc.h>
#include <sysgct.h>
#include <pack.h>
#include <system.h>

#include "pcap-septel.h"

static int septel_stats(pcap_t *p, struct pcap_stat *ps);
static int septel_getnonblock(pcap_t *p);
static int septel_setnonblock(pcap_t *p, int nonblock);

/*
 * Private data for capturing on Septel devices.
 */
struct pcap_septel {
    struct pcap_stat stat;
}

/*
 *  Read at most max_packets from the capture queue and call the callback
 *  for each of them. Returns the number of packets handled, -1 if an
 *  error occurred, or -2 if we were told to break out of the loop.
 */
static int septel_read(pcap_t *p, int cnt, pcap_handler callback, u_char *user) {

  struct pcap_septel *ps = p->priv;
  HDR *h;
  MSG *m;
  int processed = 0 ;
  int t = 0 ;

  /* identifier for the message queue of the module(upe) from which we are capturing
   * packets.These IDs are defined in system.txt . By default it is set to 0x2d
   * so change it to 0xdd for technical reason and therefore the module id for upe becomes:
   * LOCAL        0xdd           * upe - Example user part task */
  unsigned int id = 0xdd;

  /* process the packets */
  do  {

    unsigned short packet_len = 0;
    int caplen = 0;
    int counter = 0;
    struct pcap_pkthdr   pcap_header;
    u_char *dp ;

    /*
     * Has "pcap_breakloop()" been called?
     */
loop:
    # 如果标志位指示需要中断循环
    if (p->break_loop) {
      # 是的 - 清除指示中断循环的标志，并返回 -2 表示被告知中断循环
      p->break_loop = 0;
      return -2;
    }

    # 重复直到读取到一个数据包
    # 当消息为空时意味着：
    # 当队列中没有数据包或者队列中的所有数据包都已经被读取时
    do  {
      # 在非阻塞模式下接收数据包
      # GCT_grab 在 septel 库软件中定义
      h = GCT_grab(id);

      # 将接收到的数据包转换为消息类型
      m = (MSG*)h;
      # 在这里添加一个计数器以避免无限循环
      # 这将导致我们的捕获程序 GUI 在等待数据包时冻结
      counter++ ;

    }
    while  ((m == NULL)&& (counter< 100)) ;
    if (m != NULL) {
      // 如果消息不为空
      t = h->type ;
      // 将消息类型赋值给变量t

      /* catch only messages with type = 0xcf00 or 0x8f01 corresponding to ss7 messages*/
      /* XXX = why not use API_MSG_TX_REQ for 0xcf00 and API_MSG_RX_IND
       * for 0x8f01? */
      // 仅捕获类型为0xcf00或0x8f01的消息，对应于ss7消息
      // 为什么不使用API_MSG_TX_REQ代表0xcf00，使用API_MSG_RX_IND代表0x8f01？
      if ((t != 0xcf00) && (t != 0x8f01)) {
        // 如果消息类型不是0xcf00或0x8f01，则释放消息并跳转到循环开始处
        relm(h);
        goto loop ;
      }

      /* XXX - is API_MSG_RX_IND for an MTP2 or MTP3 message? */
      // API_MSG_RX_IND是用于MTP2还是MTP3消息？
      dp = get_param(m);/* get pointer to MSG parameter area (m->param) */
      // 获取消息参数区域的指针（m->param）
      packet_len = m->len;
      // 将消息长度赋值给变量packet_len
      caplen =  p->snapshot ;
      // 将快照长度赋值给变量caplen

      if (caplen > packet_len) {
        // 如果快照长度大于消息长度
        caplen = packet_len;
        // 则将快照长度设置为消息长度
      }
      /* Run the packet filter if there is one. */
      // 如果存在数据包过滤器，则运行数据包过滤器
      if ((p->fcode.bf_insns == NULL) || pcap_filter(p->fcode.bf_insns, dp, packet_len, caplen)) {
        // 如果过滤器为空或者通过数据包过滤器

        /*  get a time stamp , consisting of :
         *
         *  pcap_header.ts.tv_sec:
         *  ----------------------
         *   a UNIX format time-in-seconds when he packet was captured,
         *   i.e. the number of seconds since Epoch time (January 1,1970, 00:00:00 GMT)
         *
         *  pcap_header.ts.tv_usec :
         *  ------------------------
         *   the number of microseconds since that second
         *   when the packet was captured
         */
        // 获取时间戳，包括：
        // pcap_header.ts.tv_sec:
        // UNIX格式的时间戳，即捕获数据包时的秒数，即自1970年1月1日00:00:00 GMT以来的秒数
        // pcap_header.ts.tv_usec:
        // 自捕获数据包以来的微秒数
        (void)gettimeofday(&pcap_header.ts, NULL);
        // 获取当前时间并赋值给pcap_header.ts

        /* Fill in our own header data */
        // 填充我们自己的头部数据
        pcap_header.caplen = caplen;
        // 设置捕获长度
        pcap_header.len = packet_len;
        // 设置数据包长度

        /* Count the packet. */
        // 统计数据包
        ps->stat.ps_recv++;

        /* Call the user supplied callback function */
        // 调用用户提供的回调函数
        callback(user, &pcap_header, dp);

        processed++ ;
        // 处理数量加一
      }
      /* after being processed the packet must be
       *released in order to receive another one */
      // 处理完数据包后，必须释放以接收另一个数据包
      relm(h);
    }else
      processed++;
      // 否则处理数量加一

  }
  while (processed < cnt) ;
  // 当处理数量小于指定数量时循环

  return processed ;
  // 返回处理数量
}

static int
septel_inject(pcap_t *handle, const void *buf _U_, int size _U_)
{
  // 设置错误信息，表示 Septel 卡不支持发送数据包
  pcap_strlcpy(handle->errbuf, "Sending packets isn't supported on Septel cards",
          PCAP_ERRBUF_SIZE);
  // 返回 -1，表示发送数据包不支持
  return (-1);
}

/*
 *  Activate a handle for a live capture from the given Septel device.  Always pass a NULL device
 *  The promisc flag is ignored because Septel cards have built-in tracing.
 *  The timeout is also ignored as it is not supported in hardware.
 *
 *  See also pcap(3).
 */
static pcap_t *septel_activate(pcap_t* handle) {
  /* 初始化 pcap 结构的一些组件 */
  handle->linktype = DLT_MTP2;

  /*
   * 将负的快照值（无效）、快照值为 0（未指定）或大于正常最大值的值，转换为最大允许的值。
   *
   * 如果某些应用程序确实*需要*更大的快照长度，我们应该增加 MAXIMUM_SNAPLEN。
   */
  if (handle->snapshot <= 0 || handle->snapshot > MAXIMUM_SNAPLEN)
    handle->snapshot = MAXIMUM_SNAPLEN;

  handle->bufsize = 0;

  /*
   * "select()" 和 "poll()" 在 Septel 队列上不起作用
   */
  handle->selectable_fd = -1;

  handle->read_op = septel_read;
  handle->inject_op = septel_inject;
  handle->setfilter_op = install_bpf_program;
  handle->set_datalink_op = NULL; /* 无法更改数据链路类型 */
  handle->getnonblock_op = septel_getnonblock;
  handle->setnonblock_op = septel_setnonblock;
  handle->stats_op = septel_stats;

  return 0;
}

pcap_t *septel_create(const char *device, char *ebuf, int *is_ours) {
    const char *cp;
    pcap_t *p;

    /* 是否看起来像 Septel 设备？ */
    cp = strrchr(device, '/');
    if (cp == NULL)
        cp = device;
    if (strcmp(cp, "septel") != 0) {
        /* 不是 "septel" */
        *is_ours = 0;
        return NULL;
    }

    /* 好的，很可能是我们的设备。 */
    *is_ours = 1;

    p = PCAP_CREATE_COMMON(ebuf, struct pcap_septel);
    if (p == NULL)
        return NULL;
    # 设置 p 指针的 activate_op 属性为 septel_activate 函数
    p->activate_op = septel_activate;
    '''
     * 设置这些属性为前置，这样，即使我们的客户在我们激活之前尝试设置非阻塞模式，
     * 或者查询非阻塞模式的状态，他们会得到一个错误，而不是在以后使用时设置非阻塞模式选项。
     '''
    # 设置 p 指针的 getnonblock_op 属性为 septel_getnonblock 函数
    p->getnonblock_op = septel_getnonblock;
    # 设置 p 指针的 setnonblock_op 属性为 septel_setnonblock 函数
    p->setnonblock_op = septel_setnonblock;
    # 返回 p 指针
    return p;
}

static int septel_stats(pcap_t *p, struct pcap_stat *ps) {
  // 获取指向 pcap_septel 结构的指针
  struct pcap_septel *handlep = p->priv;
  /*handlep->stat.ps_recv = 0;*/
  /*handlep->stat.ps_drop = 0;*/

  // 将 handlep->stat 的值赋给 ps
  *ps = handlep->stat;

  // 返回 0 表示成功
  return 0;
}


int
septel_findalldevs(pcap_if_list_t *devlistp, char *errbuf)
{
  /*
   * XXX - do the notions of "up", "running", or "connected" apply here?
   */
  // 如果无法添加 septel 设备到 devlistp 中，则返回 -1
  if (add_dev(devlistp,"septel",0,"Intel/Septel device",errbuf) == NULL)
    return -1;
  return 0;
}


/*
 * We don't support non-blocking mode.  I'm not sure what we'd
 * do to support it and, given that we don't support select()/
 * poll()/epoll_wait()/kevent() etc., it probably doesn't
 * matter.
 */
static int
septel_getnonblock(pcap_t *p)
{
  // 输出错误信息，表示不支持非阻塞模式
  fprintf(p->errbuf, PCAP_ERRBUF_SIZE, "Non-blocking mode not supported on Septel devices");
  return (-1);
}

static int
septel_setnonblock(pcap_t *p, int nonblock _U_)
{
  // 输出错误信息，表示不支持非阻塞模式
  fprintf(p->errbuf, PCAP_ERRBUF_SIZE, "Non-blocking mode not supported on Septel devices");
  return (-1);
}

#ifdef SEPTEL_ONLY
/*
 * This libpcap build supports only Septel cards, not regular network
 * interfaces.
 */

/*
 * There are no regular interfaces, just Septel interfaces.
 */
// 返回 0 表示成功
int
pcap_platform_finddevs(pcap_if_list_t *devlistp, char *errbuf)
{
  return (0);
}

/*
 * Attempts to open a regular interface fail.
 */
// 返回 NULL，表示无法打开常规接口
pcap_t *
pcap_create_interface(const char *device, char *errbuf)
{
  snprintf(errbuf, PCAP_ERRBUF_SIZE,
                "This version of libpcap only supports Septel cards");
  return (NULL);
}

/*
 * Libpcap version string.
 */
// 返回 libpcap 版本信息，表示仅支持 Septel 设备
const char *
pcap_lib_version(void)
{
  return (PCAP_VERSION_STRING " (Septel-only)");
}
#endif
```