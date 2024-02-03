# `nmap\libpcap\pcap-septel.h`

```cpp
/*
 * pcap-septel.c: 用于 Intel Septel 卡的数据包捕获接口
 *
 * 该代码的功能尽可能模仿 pcap-linux 的功能。只有在同时编译 Intel/Septel 卡代码和其他类型设备代码时才需要该代码。
 *
 * 作者: Gilbert HOYEK (gil_hoyek@hotmail.com), Elias M. KHOURY (+961 3 485343);
 */

// 创建一个 Septel 数据包捕获对象
pcap_t *septel_create(const char *device, char *ebuf, int *is_ours);

// 查找所有 Septel 设备并将它们添加到设备列表中
int septel_findalldevs(pcap_if_list_t *devlistp, char *errbuf);
```