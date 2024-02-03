# `nmap\libpcap\pcap-snf.h`

```cpp
// 创建一个新的数据包捕获会话，传入参数为文件名、过滤规则和错误码指针，返回一个 pcap_t 类型的指针
pcap_t *snf_create(const char *, char *, int *);

// 查找所有可用的网络设备，并将它们存储在 devlistp 中，传入参数为设备列表和错误信息字符串，返回一个整型值
int snf_findalldevs(pcap_if_list_t *devlistp, char *errbuf);
```