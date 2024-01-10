# `nmap\libpcap\pcap-dpdk.h`

```
/*
 * 版权所有（C）2018 jingle YANG。
 *
 * 在源代码和二进制形式中重新分发和使用，无论是否经过修改，
 * 都是允许的，前提是满足以下条件：
 *
 *   1. 源代码的重新分发必须保留上述版权声明、此条件列表和以下免责声明。
 *   2. 以二进制形式重新分发时，必须在文档和/或其他提供的材料中重现上述版权声明、此条件列表和以下免责声明。
 *
 * 本软件由作者和贡献者“按原样”提供，不提供任何明示或暗示的担保，
 * 包括但不限于对适销性和特定用途的暗示担保。无论在任何情况下，
 * 作者或贡献者均不对任何直接、间接、偶然、特殊、惩罚性或后果性的损害
 * （包括但不限于替代商品或服务的采购；使用、数据或利润的损失；或业务中断）
 * 负责，无论是在合同责任、严格责任或侵权行为（包括疏忽或其他方式）的任何理论下产生的。
 * 使用本软件的任何方式，即使已被告知可能会发生此类损害，也不承担任何责任。
 */

// 使用给定的参数创建一个基于 DPDK 的 pcap 对象
pcap_t *pcap_dpdk_create(const char *dev, char *errbuf, int *promisc);

// 在给定的设备列表中查找基于 DPDK 的网络接口
int pcap_dpdk_findalldevs(pcap_if_list_t *devlistp, char *errbuf);
```