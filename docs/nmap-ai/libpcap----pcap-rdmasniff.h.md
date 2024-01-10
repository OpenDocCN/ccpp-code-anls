# `nmap\libpcap\pcap-rdmasniff.h`

```
// 创建一个新的 rdmasniff 会话，用于捕获指定设备的数据包
pcap_t *rdmasniff_create(const char *device, char *ebuf, int *is_ours);

// 查找所有可用的网络设备，并将它们添加到设备列表中
int rdmasniff_findalldevs(pcap_if_list_t *devlistp, char *err_str);
```