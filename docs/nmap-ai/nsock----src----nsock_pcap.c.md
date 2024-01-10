# `nmap\nsock\src\nsock_pcap.c`

```
/* $Id$ */

#include "nsock.h"  // 包含 nsock 头文件
#include "nsock_internal.h"  // 包含 nsock 内部头文件
#include "nsock_log.h"  // 包含 nsock 日志头文件

#include <limits.h>  // 包含 limits.h 头文件
#if HAVE_SYS_IOCTL_H
#include <sys/ioctl.h>  // 如果有 sys/ioctl.h 头文件，则包含
#endif
#if HAVE_NET_BPF_H
#ifdef _AIX
/* Prevent bpf.h from redefining the DLT_ values to their IFT_ values. (See
 * similar comment in libpcap/pcap-bpf.c.) */
#undef _AIX
#include <net/bpf.h>  // 如果有 net/bpf.h 头文件，则包含
#define _AIX
#else
#include <net/bpf.h>  // 如果有 net/bpf.h 头文件，则包含
#endif
#endif

#include "nsock_pcap.h"  // 包含 nsock_pcap 头文件

extern struct timeval nsock_tod;  // 声明 nsock_tod 结构体变量

#if HAVE_PCAP  // 如果有 pcap 库

#ifndef PCAP_NETMASK_UNKNOWN
/* libpcap before 1.1.1 (e.g. WinPcap) doesn't handle this specially, so just use 0 netmask */
#define PCAP_NETMASK_UNKNOWN 0  // 如果没有定义 PCAP_NETMASK_UNKNOWN，则定义为 0
#endif

#define PCAP_OPEN_MAX_RETRIES   3  // 定义 PCAP_OPEN_MAX_RETRIES 为 3

#define PCAP_FAILURE_EXPL_MESSAGE  \  // 定义 PCAP_FAILURE_EXPL_MESSAGE 为以下字符串
    "There are several possible reasons for this, " \
    "depending on your operating system:\n" \
    "LINUX: If you are getting Socket type not supported, " \
    "try modprobe af_packet or recompile your kernel with PACKET enabled.\n" \
    "*BSD:  If you are getting device not configured, you need to recompile " \
    "your kernel with Berkeley Packet Filter support." \
    "If you are getting No such file or directory, try creating the device " \
    "(eg cd /dev; MAKEDEV <device>; or use mknod).\n" \
    "*WINDOWS:  Nmap only supports ethernet interfaces on Windows for most " \
    "operations because Microsoft disabled raw sockets as of Windows XP SP2. " \
    "Depending on the reason for this error, it is possible that the " \
    "--unprivileged command-line argument will help.\n" \
    "SOLARIS:  If you are trying to scan localhost and getting "\
    "'/dev/lo0: No such file or directory', complain to Sun.  "\
    "I don't think Solaris can support advanced localhost scans.  "\
    "You can probably use \"-PN -sT localhost\" though.\n\n"
# 设置 pcap 过滤器
static int nsock_pcap_set_filter(struct npool *nsp, pcap_t *pt, const char *device,
                                 const char *bpf) {
  struct bpf_program fcode;  # 定义 bpf 编译后的程序
  int rc;  # 定义返回结果

  rc = pcap_compile(pt, &fcode, (char *)bpf, 1, PCAP_NETMASK_UNKNOWN);  # 编译过滤器
  if (rc) {  # 如果编译失败
    nsock_log_error("Error compiling pcap filter: %s", pcap_geterr(pt));  # 记录错误信息
    return rc;  # 返回错误码
  }

  rc = pcap_setfilter(pt, &fcode);  # 设置过滤器
  if (rc) {  # 如果设置失败
    nsock_log_error("Failed to set the pcap filter: %s", pcap_geterr(pt));  # 记录错误信息
  }

  pcap_freecode(&fcode);  # 释放编译后的程序
  return rc;  # 返回结果
}

# 获取第三层偏移量
static int nsock_pcap_get_l3_offset(pcap_t *pt, int *dl) {
  int datalink;  # 定义数据链路类型
  unsigned int offset = 0;  # 定义偏移量，默认为 0

  /* New packet capture device, need to recompute offset */
  if ((datalink = pcap_datalink(pt)) < 0)  # 获取数据链路类型，如果失败
    fatal("Cannot obtain datalink information: %s", pcap_geterr(pt));  # 输出错误信息并退出程序

  /* XXX NOTE:
   * if a new offset ever exceeds the current max (24),
   * adjust MAX_LINK_HEADERSZ in libnetutil/netutil.h
   */
  switch (datalink) {  # 根据数据链路类型进行偏移量计算
    case DLT_EN10MB: offset = 14; break;  # 以太网
    case DLT_IEEE802: offset = 22; break;  # IEEE 802
    #ifdef __amigaos__
    case DLT_MIAMI: offset = 16; break;  # Miami
    #endif
    #ifdef DLT_LOOP
    case DLT_LOOP:
    #endif
    case DLT_NULL: offset = 4; break;  # 空链路

    case DLT_SLIP:
    #ifdef DLT_SLIP_BSDOS
    case DLT_SLIP_BSDOS:
    #endif
    #if (FREEBSD || OPENBSD || NETBSD || BSDI || MACOSX)
    offset = 16;break;  # FreeBSD、OpenBSD、NetBSD、BSDI、MacOSX
    #else
    offset = 24;break; /* Anyone use this??? */  # 默认偏移量
    #endif

    case DLT_PPP:
    #ifdef DLT_PPP_BSDOS
    case DLT_PPP_BSDOS:
    #endif
    #ifdef DLT_PPP_SERIAL
    case DLT_PPP_SERIAL:
    #endif
    #ifdef DLT_PPP_ETHER
    case DLT_PPP_ETHER:
    #endif
    #if (FREEBSD || OPENBSD || NETBSD || BSDI || MACOSX)
    offset = 4;break;  # FreeBSD、OpenBSD、NetBSD、BSDI、MacOSX
    #else
      #ifdef SOLARIS
        offset = 8;break;  # Solaris
      #else
        offset = 24;break; /* Anyone use this? */  # 默认偏移量
      #endif /* ifdef solaris */
    #endif /* if freebsd || openbsd || netbsd || bsdi */
    #ifdef DLT_RAW
    case DLT_RAW: offset = 0; break;  # 原始数据链路
    #endif /* DLT_RAW */
    # 根据数据链路类型设置偏移量
    case DLT_FDDI: offset = 21; break;
    #ifdef DLT_ENC
    case DLT_ENC: offset = 12; break;
    #endif /* DLT_ENC */
    #ifdef DLT_LINUX_SLL
    case DLT_LINUX_SLL: offset = 16; break;
    #endif
    #ifdef DLT_IPNET
    case DLT_IPNET: offset = 24; break;
    #endif /* DLT_IPNET */

    # 默认情况下，如果数据链路类型未知，则输出错误信息并终止程序
    default: /* Sorry, link type is unknown. */
      fatal("Unknown datalink type %d.\n", datalink);
  }
  # 如果 dl 指针不为空，则将数据链路类型赋值给它
  if (dl)
    *dl = datalink;
  # 返回偏移量
  return (offset);
/* Convert new nsiod to pcap descriptor. Other parameters have
 * the same meaning as for pcap_open_live in pcap(3).
 *   device   : pcap-style device name
 *   snaplen  : size of packet to be copied to handler
 *   promisc  : whether to open device in promiscuous mode
 *   bpf_fmt   : berkeley filter
 * return value: NULL if everything was okay, or error string
 * if error occurred. */
int nsock_pcap_open(nsock_pool nsp, nsock_iod nsiod, const char *pcap_device,
                    int snaplen, int promisc, const char *bpf_fmt, ...) {
  // 将 nsiod 转换为 pcap 描述符
  struct niod *nsi = (struct niod *)nsiod;
  struct npool *ms = (struct npool *)nsp;
  mspcap *mp = (mspcap *)nsi->pcap;
  char errbuf[PCAP_ERRBUF_SIZE];
  char bpf[4096];
  va_list ap;
  int failed, datalink;
  int rc;

#ifdef PCAP_CAN_DO_SELECT
#if PCAP_BSD_SELECT_HACK
  /* MacOsX reports error if to_ms is too big (like INT_MAX) with error
   * FAILED. Reported error: BIOCSRTIMEOUT: Invalid argument
   * INT_MAX/6 (=357913941) seems to be working... */
  int to_ms = 357913941;
#else
  int to_ms = 200;
#endif /* PCAP_BSD_SELECT_HACK */

#else
  int to_ms = 1;
#endif

  // 获取当前时间
  gettimeofday(&nsock_tod, NULL);

  // 检查是否已经打开了 pcap 设备
  if (mp) {
    nsock_log_error("This nsi already has pcap device opened");
    return -1;
  }

  // 分配内存给 mp
  mp = (mspcap *)safe_zalloc(sizeof(mspcap));
  nsi->pcap = (void *)mp;

  // 解析 bpf_fmt 中的格式化字符串
  va_start(ap, bpf_fmt);
  rc = Vsnprintf(bpf, sizeof(bpf), bpf_fmt, ap);
  va_end(ap);

  // 检查格式化后的字符串长度是否超过限制
  if (rc >= (int)sizeof(bpf)) {
    nsock_log_error("Too-large bpf filter argument");
    return -1;
  }

  // 打印 PCAP 请求的信息
  nsock_log_info("PCAP requested on device '%s' with berkeley filter '%s' "
                 "(promisc=%i snaplen=%i to_ms=%i) (IOD #%li)",
                 pcap_device,bpf, promisc, snaplen, to_ms, nsi->id);

#ifdef __amigaos__
  // Amiga 没有 pcap_create
  // TODO: Amiga 上的 Nmap 是否仍然可用？
  // 在 Amiga 上打开 pcap 设备
  mp->pt = pcap_open_live(pcap_device, snaplen, promisc, to_ms, errbuf);
  // 如果打开失败
  if (!mp->pt) {
    # 使用 nsock_log_error 函数记录错误日志，包括 pcap_open_live 函数的参数和错误信息
    nsock_log_error("pcap_open_live(%s, %d, %d, %d) failed with error: %s",
        pcap_device, snaplen, promisc, to_ms, errbuf);
    # 使用 nsock_log_error 函数记录错误日志，包括 PCAP_FAILURE_EXPL_MESSAGE
    nsock_log_error(PCAP_FAILURE_EXPL_MESSAGE);
    # 使用 nsock_log_error 函数记录错误日志，提示无法打开 pcap，需要 root 权限
    nsock_log_error("Can't open pcap! Are you root?");
    # 返回错误代码 -1
    return -1;
#else
  // 如果不是上述情况，则创建一个新的 pcap 对象
  mp->pt = pcap_create(pcap_device, errbuf);
  // 如果创建失败，则记录错误信息并返回 -1
  if (!mp->pt) {
    nsock_log_error("pcap_create(%s) failed with error: %s", pcap_device, errbuf);
    nsock_log_error(PCAP_FAILURE_EXPL_MESSAGE);
    nsock_log_error("Can't open pcap! Are you root?");
    return -1;
  }

// 定义一个宏，用于设置 pcap 参数
#define MY_PCAP_SET(func, p_t, val) do {\
  failed = func(p_t, val);\
  if (failed) {\
    nsock_log_error(#func "(%d) FAILED: %d.", val, failed);\
    pcap_close(p_t);\
    return -1;\
  }\
} while(0);

  // 使用宏设置抓包长度
  MY_PCAP_SET(pcap_set_snaplen, mp->pt, snaplen);
  // 使用宏设置混杂模式
  MY_PCAP_SET(pcap_set_promisc, mp->pt, promisc);
  // 使用宏设置超时时间
  MY_PCAP_SET(pcap_set_timeout, mp->pt, to_ms);
#ifdef HAVE_PCAP_SET_IMMEDIATE_MODE
  // 如果支持立即模式，则使用宏设置
  MY_PCAP_SET(pcap_set_immediate_mode, mp->pt, 1);
#endif

  // 激活 pcap 对象
  failed = pcap_activate(mp->pt);
  // 如果激活失败，则记录错误信息并返回 -1
  if (failed < 0) {
    nsock_log_error("pcap_activate(%s) FAILED: %s.", pcap_device, pcap_geterr(mp->pt));
    pcap_close(mp->pt);
    mp->pt = NULL;
    return -1;
  }
  // 如果激活有警告，则记录警告信息
  else if (failed > 0) {
    nsock_log_error("pcap_activate(%s) WARNING: %s.", pcap_device, pcap_geterr(mp->pt));
  }
#endif /* not __amigaos__ */

  // 设置过滤器
  rc = nsock_pcap_set_filter(ms, mp->pt, pcap_device, bpf);
  // 如果设置过滤器失败，则返回错误码
  if (rc)
    return rc;

  // 获取数据链路层偏移量
  mp->l3_offset = nsock_pcap_get_l3_offset(mp->pt, &datalink);
  // 设置抓包长度
  mp->snaplen = snaplen;
  // 设置数据链路类型
  mp->datalink = datalink;
  // 复制 pcap 设备名称
  mp->pcap_device = strdup(pcap_device);
#ifdef PCAP_CAN_DO_SELECT
  // 获取可选择的文件描述符
  mp->pcap_desc = pcap_get_selectable_fd(mp->pt);
#else
  // 如果不支持获取可选择的文件描述符，则设置为 -1
  mp->pcap_desc = -1;
#endif
  // 读取套接字计数初始化为 0
  mp->readsd_count = 0;

#ifndef HAVE_PCAP_SET_IMMEDIATE_MODE
  // 如果不支持立即模式，则已经由 pcap_set_immediate_mode 处理
#ifdef BIOCIMMEDIATE
  /* 如果不设置这个 ioctl，在一些系统上（例如 BSD，尽管这取决于版本），非阻塞模式下会缓冲数据包，并且只有在缓冲区满时才返回它们。设置这个 ioctl 会使每个数据包立即被传送。这就是 Linux 默认的工作方式。参见 libpcap/pcap-bpf.c 中关于设置 BIOCIMMEDIATE 的注释。 */
  if (mp->pcap_desc != -1) {
    int immediate = 1;

    if (ioctl(mp->pcap_desc, BIOCIMMEDIATE, &immediate) < 0)
      fatal("Cannot set BIOCIMMEDIATE on pcap descriptor");
  }
#elif defined WIN32
  /* 我们希望尽快收到任何响应 */
  pcap_setmintocopy(mp->pt, 0);
#endif
#endif

  /* 设置设备为非阻塞模式 */
  rc = pcap_setnonblock(mp->pt, 1, errbuf);
  if (rc) {

    /* 我无法在 pcap 上执行 select()！
     * 阻塞 + 无法使用 select 是致命的 */
#ifndef PCAP_BSD_SELECT_HACK
    if (mp->pcap_desc < 0)
#endif
    {
      nsock_log_error("Failed to set pcap descriptor on device %s "
                      "to nonblocking mode: %s", pcap_device, errbuf);
      return -1;
    }
    /* 在其他情况下，我们可以接受阻塞的 pcap */
    nsock_log_info("Failed to set pcap descriptor on device %s "
                   "to nonblocking state: %s", pcap_device, errbuf);
  }

  if (NsockLogLevel <= NSOCK_LOG_INFO) {
    #if PCAP_BSD_SELECT_HACK
      int bsd_select_hack = 1;
    #else
      int bsd_select_hack = 0;
    #endif

    #if PCAP_RECV_TIMEVAL_VALID
      int recv_timeval_valid = 1;
    #else
      int recv_timeval_valid = 0;
    #endif

    nsock_log_info("PCAP created successfully on device '%s' "
                   "(pcap_desc=%i bsd_hack=%i to_valid=%i l3_offset=%i) (IOD #%li)",
                   pcap_device, mp->pcap_desc, bsd_select_hack,
                   recv_timeval_valid, mp->l3_offset, nsi->id);
  }
  return 0;
}

/* 请求精确捕获一个数据包。 */
# 从 nsock_pool 中读取数据包，处理事件
nsock_event_id nsock_pcap_read_packet(nsock_pool nsp, nsock_iod nsiod,
                                      nsock_ev_handler handler,
                                      int timeout_msecs, void *userdata) {
  # 将 nsiod 转换为 struct niod 类型
  struct niod *nsi = (struct niod *)nsiod;
  # 将 nsp 转换为 struct npool 类型
  struct npool *ms = (struct npool *)nsp;
  # 定义 nevent 结构体指针 nse
  struct nevent *nse;

  # 创建一个新的事件，类型为 NSE_TYPE_PCAP_READ
  nse = event_new(ms, NSE_TYPE_PCAP_READ, nsi, timeout_msecs, handler, userdata);
  # 断言事件非空
  assert(nse);

  # 记录日志，表示从 IOD #nsi->id 读取了 EID #nse->id 的 pcap 数据包
  nsock_log_info("Pcap read request from IOD #%li  EID %li", nsi->id, nse->id);

  # 将事件添加到 nsock_pool 中
  nsock_pool_add_event(ms, nse);

  # 返回事件的 id
  return nse->id;
}

/* Remember that pcap descriptor is in nonblocking state. */
# 执行实际的 pcap 读取操作
int do_actual_pcap_read(struct nevent *nse) {
  # 将 nse->iod->pcap 转换为 mspcap 类型
  mspcap *mp = (mspcap *)nse->iod->pcap;
  # 定义 nsock_pcap 结构体 npp 和 nsock_pcap 指针 n
  nsock_pcap npp;
  nsock_pcap *n;
  # 定义 pcap 数据包头指针 pkt_header 和数据包内容指针 pkt_data
  struct pcap_pkthdr *pkt_header;
  const unsigned char *pkt_data = NULL;
  # 定义返回值 rc
  int rc;

  # 将 npp 的内存清零
  memset(&npp, 0, sizeof(nsock_pcap));

  # 记录调试日志，表示执行了 PCAP 测试
  nsock_log_debug_all("PCAP %s TEST (IOD #%li) (EID #%li)",
                      __func__, nse->iod->id, nse->id);

  # 断言 nse->iobuf 的长度为 0
  assert(fs_length(&(nse->iobuf)) == 0);

  # 从 pcap 中读取下一个数据包
  rc = pcap_next_ex(mp->pt, &pkt_header, &pkt_data);
  switch (rc) {
    case 1: /* read good packet  */
      # 如果读取到有效的数据包
      # 判断平台是否支持 PCAP_RECV_TIMEVAL_VALID
      # 如果支持，将 npp.ts 设置为 pkt_header->ts
      # 如果不支持，将 npp.ts 设置为当前时间
      npp.ts     = pkt_header->ts;
      npp.len    = pkt_header->len;
      npp.caplen = pkt_header->caplen;
      npp.packet = pkt_data;

      # 将 npp 和 pkt_data 添加到 nse->iobuf 中
      fs_cat(&(nse->iobuf), (char *)&npp, sizeof(npp));
      fs_cat(&(nse->iobuf), (char *)pkt_data, npp.caplen);
      # 将 n 指向 nse->iobuf 的数据
      n = (nsock_pcap *)fs_str(&(nse->iobuf));
      n->packet = (unsigned char *)fs_str(&(nse->iobuf)) + sizeof(npp);

      # 记录调试日志，表示读取了 pcap 数据包
      nsock_log_debug_all("PCAP %s READ (IOD #%li) (EID #%li) size=%i",
                          __func__, nse->iod->id, nse->id, pkt_header->caplen);
      rc = 1;
      break;

    case 0: /* timeout */
      # 如果超时，将 rc 设置为 0
      rc = 0;
      break;
    # 当返回值为-1时，表示pcap_next_ex()读取pcap文件时发生致命错误，输出错误信息并终止程序
    case -1: /* error */
      fatal("pcap_next_ex() fatal error while reading from pcap: %s\n",
            pcap_geterr(mp->pt));
      break;

    # 当返回值为-2时，表示从保存文件中没有更多的数据包可读取，或者默认情况下，输出意外的返回码并终止程序
    case -2: /* no more packets in savefile (if reading from one) */
    default:
      fatal("Unexpected return code from pcap_next_ex! (%d)\n", rc);
  }

  # 返回pcap_next_ex()的返回值
  return rc;
# 读取 pcap 数据包内容，并将结果存储在指定的参数中
void nse_readpcap(nsock_event nsev, const unsigned char **l2_data, size_t *l2_len,
                  const unsigned char **l3_data, size_t *l3_len,
                  size_t *packet_len, struct timeval *ts) {
  # 将 nsev 转换为 nevent 结构体
  struct nevent *nse = (struct nevent *)nsev;
  # 从 nevent 结构体中获取 niod 结构体
  struct niod  *iod = nse->iod;
  # 从 niod 结构体中获取 pcap 结构体
  mspcap *mp = (mspcap *)iod->pcap;
  # 定义 nsock_pcap 结构体
  nsock_pcap *n;
  # 定义 l2 数据长度和 l3 数据长度
  size_t l2l;
  size_t l3l;

  # 将 nse->iobuf 转换为 nsock_pcap 结构体
  n = (nsock_pcap *)fs_str(&(nse->iobuf));
  # 如果 iobuf 长度小于 nsock_pcap 结构体的长度，则将 l2_data、l2_len、l3_data、l3_len、packet_len 设置为 NULL 或 0
  if (fs_length(&(nse->iobuf)) < sizeof(nsock_pcap)) {
    if (l2_data)
      *l2_data = NULL;
    if (l2_len)
      *l2_len = 0;
    if (l3_data)
      *l3_data = NULL;
    if (l3_len)
      *l3_len = 0;
    if (packet_len)
      *packet_len = 0;
    return;
  }

  # 计算 l2 数据长度和 l3 数据长度
  l2l = MIN(mp->l3_offset, n->caplen);
  l3l = MAX(0, n->caplen-mp->l3_offset);

  # 根据条件将 l2_data、l2_len、l3_data、l3_len、packet_len、ts 设置为相应的值
  if (l2_data)
    *l2_data = n->packet;
  if (l2_len)
    *l2_len = l2l;
  if (l3_data)
    *l3_data = (l3l > 0) ? n->packet+l2l : NULL;
  if (l3_len)
    *l3_len = l3l;
  if (packet_len)
    *packet_len = n->len;
  if (ts)
    *ts = n->ts;
  return;
}

# 获取 niod 结构体中的 pcap 数据链路类型
int nsock_iod_linktype(nsock_iod iod) {
  # 将 iod 转换为 niod 结构体
  struct niod *nsi = (struct niod *)iod;
  # 从 niod 结构体中获取 pcap 结构体
  mspcap *mp = (mspcap *)nsi->pcap;

  # 断言 pcap 结构体存在
  assert(mp);
  # 返回 pcap 数据链路类型
  return (mp->datalink);
}

# 判断 niod 结构体中是否存在 pcap 数据
int nsock_iod_is_pcap(nsock_iod iod) {
  # 将 iod 转换为 niod 结构体
  struct niod *nsi = (struct niod *)iod;
  # 从 niod 结构体中获取 pcap 结构体
  mspcap *mp = (mspcap *)nsi->pcap;

  # 返回是否存在 pcap 数据
  return (mp != NULL);
}

#endif /* HAVE_PCAP */
```