# `nmap\libpcap\pcap-rpcap.h`

```
/*
 * 版权声明，版权归加利福尼亚大学理事会所有
 * 允许以源代码或二进制形式重新分发和使用，无论是否经过修改
 * 必须满足以下条件：
 * 1. 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明
 * 2. 以二进制形式再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 * 3. 所有提及此软件特性或使用的广告材料必须显示以下声明：
 *    本产品包含由劳伦斯伯克利实验室计算机系统工程组开发的软件
 * 4. 未经特定事先书面许可，不得使用大学或实验室的名称来认可或推广从本软件衍生的产品
 *
 * 本软件由理事会和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保
 * 在任何情况下，理事会或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）承担责任，无论是合同责任、严格责任还是侵权行为（包括疏忽或其他方式）引起的，即使已被告知可能发生此类损害
 */
#ifndef pcap_rpcap_h
#define    pcap_rpcap_h

/*
 * "pcap_open()" 的内部接口
 */
// 使用远程方式打开一个远程捕获会话，返回一个 pcap_t 结构体指针
pcap_t *pcap_open_rpcap(const char *source, int snaplen, int flags,
    int read_timeout, struct pcap_rmtauth *auth, char *errbuf);

/*
 * "pcap_findalldevs_ex()" 的内部接口
 * 用于远程方式查找所有网络接口
 */
int pcap_findalldevs_ex_remote(const char *source,
    struct pcap_rmtauth *auth, pcap_if_t **alldevs, char *errbuf);

#endif
```