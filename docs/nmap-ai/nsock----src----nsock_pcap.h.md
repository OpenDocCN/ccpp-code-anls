# `nmap\nsock\src\nsock_pcap.h`

```
/* $Id$ */

#ifndef NSOCK_PCAP_H
#define NSOCK_PCAP_H

#include "nsock_internal.h"
#ifdef HAVE_PCAP

#include "pcap.h"

#include <string.h>
#include <stdarg.h>

/*
 * 从 pcap 描述符读取数据的三种可能方式：
 *  - 在描述符上使用 select()：
 *      这当然是最好的方式，但是有一些系统不支持，比如 WIN32。这在 Linux 上效果很好。
 *
 *  - 使用 select() + 一些技巧：
 *      这是为旧的 bsd 系统设计的技巧，
 *      描述符 *必须* 设置为非阻塞模式。
 *
 *   - 不使用 select()：
 *      这是为 WIN32 和其他一些系统设计的，这些系统从 pcap_get_selectable_fd() 返回描述符 -1。
 *      在这种情况下，描述符 *必须* 设置为非阻塞模式。
 *      如果失败了，那么我们无法从该系统进行任何嗅探。
 *
 * 无论哪种方式，我们都尝试将描述符设置为非阻塞模式。
 */

/* 返回系统是否正确支持 pcap_get_selectable_fd() */
#if !defined(WIN32) && !defined(SOLARIS_BPF_PCAP_CAPTURE)
#define PCAP_CAN_DO_SELECT 1
#endif

/* 在一些系统（比如 Windows）中，pcap 描述符不可选择。
 * 因此，我们不能只是在其上使用 select() 并期望它唤醒我们并传递数据包，而是需要连续轮询它。
 * 这个定义设置了轮询频率，以毫秒为单位，用于确定是否有捕获的数据包。
 * 请注意，这仅在未定义 PCAP_CAN_DO_SELECT 时使用，因此对像 Linux 这样的系统没有影响。
 */
#define PCAP_POLL_INTERVAL 2
/* Note that on most versions of most BSDs (including Mac OS X) select() and
 * poll() do not work correctly on BPF devices; pcap_get_selectable_fd() will
 * return a file descriptor on most of those versions (the exceptions being
 * FreeBSD 4.3 and 4.4), a simple select() or poll() will not return even after
 * a timeout specified in pcap_open_live() expires. To work around this, an
 * application that uses select() or poll() to wait for packets to arrive must
 * put the pcap_t in non-blocking mode, and must arrange that the select() or
 * poll() have a timeout less than or equal to the timeout specified in
 * pcap_open_live(), and must try to read packets after that timeout expires,
 * regardless of whether select() or poll() indicated that the file descriptor
 * for the pcap_t is ready to be read or not. (That workaround will not work in
 * FreeBSD 4.3 and later; however, in FreeBSD 4.6 and later, select() and poll()
 * work correctly on BPF devices, so the workaround isn't necessary, although it
 * does no harm.)
 */
#if defined(MACOSX) || defined(FREEBSD) || defined(OPENBSD)
/* Well, now select() is not receiving any pcap events on MACOSX, but maybe it
 * will someday :) in both cases. It never hurts to enable this feature. It just
 * has performance penalty. */
#define PCAP_BSD_SELECT_HACK 1
#endif

/* Returns whether the packet receive time value obtained from libpcap
 * (and thus by readip_pcap()) should be considered valid.  When
 * invalid (Windows and Amiga), readip_pcap returns the time you called it. */
#if !defined(WIN32) && !defined(__amigaos__)
#define PCAP_RECV_TIMEVAL_VALID 1
#endif


typedef struct{
  pcap_t *pt;
  int pcap_desc;
  /* Like the corresponding member in iod, when this reaches 0 we stop
   * watching the socket for readability. */
  int readsd_count;
  int datalink;
  int l3_offset;
  int snaplen;
  char *pcap_device;
} mspcap;
# 定义了一个结构体，用于存储抓包数据的相关信息
typedef struct{
  // 时间戳
  struct timeval ts;
  // 抓取的数据包的实际长度
  int caplen;
  // 数据包的总长度
  int len;
  // 指向抓取的数据包的指针，长度为caplen字节
  const unsigned char *packet;  /* caplen bytes */
} nsock_pcap;

// 执行实际的抓包读取操作的函数声明
int do_actual_pcap_read(struct nevent *nse);

#endif /* HAVE_PCAP */
#endif /* NSOCK_PCAP_H */
```