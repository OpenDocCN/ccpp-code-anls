# `nmap\libpcap\pcap-airpcap.h`

```
/*
 * 版权声明，版权所有
 */
pcap_t *airpcap_create(const char *, char *, int *);
/*
 * 创建一个 AirPcap 设备
 */
int airpcap_findalldevs(pcap_if_list_t *devlistp, char *errbuf);
/*
 * 查找所有的 AirPcap 设备
 */
int device_is_airpcap(const char *device, char *ebuf);
/*
 * 检查设备是否为 AirPcap 设备
 */
```