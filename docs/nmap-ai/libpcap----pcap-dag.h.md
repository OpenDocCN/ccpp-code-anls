# `nmap\libpcap\pcap-dag.h`

```
/*
 * pcap-dag.c: Endace DAG卡的数据包捕获接口。
 *
 * 该代码的功能尽可能模拟了pcap-linux的功能。只有在同时编译DAG卡代码和其他类型设备代码时才需要该代码。
 *
 * 作者：Richard Littin, Sean Irvine ({richard,sean}@reeltwo.com)
 */

// 创建一个DAG捕获实例
pcap_t *dag_create(const char *, char *, int *);

// 查找所有DAG设备并将它们添加到设备列表中
int dag_findalldevs(pcap_if_list_t *devlistp, char *errbuf);
```