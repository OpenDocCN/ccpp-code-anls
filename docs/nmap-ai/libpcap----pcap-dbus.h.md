# `nmap\libpcap\pcap-dbus.h`

```cpp
// 创建一个新的 pcap_t 对象，用于捕获指定接口的数据包
pcap_t *dbus_create(const char *source, char *errbuf, int *status);

// 在指定的设备列表中查找所有可用的网络接口，并将结果存储在 devlistp 中
int dbus_findalldevs(pcap_if_list_t *devlistp, char *errbuf);
```