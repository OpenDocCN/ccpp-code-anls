# `nmap\libpcap\pcap-sita.h`

```cpp
/*
 * pcap-sita.h: SITA WAN设备的数据包捕获接口
 * 
 * 作者: Fulko Hew (fulko.hew@sita.aero) (+1 905 6815570);
 */

// 解析主机文件，返回错误信息
extern int acn_parse_hosts_file(char *errbuf);

// 查找所有设备，返回错误信息
extern int acn_findalldevs(char *errbuf);
```