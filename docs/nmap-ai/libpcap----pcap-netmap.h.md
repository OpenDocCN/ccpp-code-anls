# `nmap\libpcap\pcap-netmap.h`

```
// 创建一个 netmap 抓包会话，传入参数为文件名、模式和错误指针
pcap_t *pcap_netmap_create(const char *, char *, int *);
// 在 netmap 上查找所有设备，并将结果存储在 devlistp 中，错误信息存储在 errbuf 中
int pcap_netmap_findalldevs(pcap_if_list_t *devlistp, char *errbuf);
```