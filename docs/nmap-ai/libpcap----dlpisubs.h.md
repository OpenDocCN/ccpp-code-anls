# `nmap\libpcap\dlpisubs.h`

```
#ifndef dlpisubs_h
#define    dlpisubs_h

#ifdef __cplusplus
extern "C" {
#endif

/*
 * Private data for capturing on DLPI devices.
 */
struct pcap_dlpi {
#ifdef HAVE_LIBDLPI
    dlpi_handle_t dlpi_hd;  // DLPI 设备的句柄
#endif /* HAVE_LIBDLPI */
#ifdef DL_HP_RAWDLS
    int send_fd;  // 发送文件描述符
#endif /* DL_HP_RAWDLS */

    struct pcap_stat stat;  // pcap 统计信息
};

/*
 * Functions defined by dlpisubs.c.
 */
int pcap_stats_dlpi(pcap_t *, struct pcap_stat *);  // 获取 DLPI 设备的统计信息
int pcap_process_pkts(pcap_t *, pcap_handler, u_char *, int, u_char *, int);  // 处理 DLPI 设备的数据包
int pcap_process_mactype(pcap_t *, u_int);  // 处理 MAC 类型
#ifdef HAVE_SYS_BUFMOD_H
int pcap_conf_bufmod(pcap_t *, int);  // 配置缓冲模块
#endif
int pcap_alloc_databuf(pcap_t *);  // 分配数据缓冲区
int strioctl(int, int, int, char *);  // 字符串 I/O 控制

#ifdef __cplusplus
}
#endif

#endif
```