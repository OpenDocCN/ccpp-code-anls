# `nmap\libpcap\pcap\namedb.h`

```
/*
 * 版权声明，版权归加利福尼亚大学理事会所有
 * 允许在源代码和二进制形式下进行再发布和使用，无论是否经过修改
 * 需满足以下条件：
 * 1. 源代码的再发布必须保留上述版权声明、条件列表和以下免责声明
 * 2. 二进制形式的再发布必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 * 3. 所有宣传材料提及本软件的特性或使用必须显示以下声明：
 *    本产品包含由劳伦斯伯克利实验室计算机系统工程组开发的软件
 * 4. 未经特定事先书面许可，不得使用大学或实验室的名称来认可或推广从本软件衍生的产品
 *
 * 本软件由理事会和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保
 * 在任何情况下，理事会或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性的损害承担责任（包括但不限于替代商品或服务的采购、使用损失、数据或利润损失、业务中断等），无论是合同责任、严格责任还是侵权行为（包括疏忽或其他方式）引起的，即使已被告知可能发生此类损害
 */
#ifndef lib_pcap_namedb_h
#define lib_pcap_namedb_h

#ifdef __cplusplus
extern "C" {
#endif
/*
 * 由 pcap_next_etherent() 返回
 * XXX 这些内容不属于这个接口，但是这个库已经必须执行名称到地址的转换，所以在一些不支持 /etc/ethers 的系统上，
 * 我们导出这些钩子，因为它们已经被一些应用程序（如 tcpdump）使用，并且已经在一些提供 libpcap 的操作系统中被标记为已导出（如 Debian）。
 */
struct pcap_etherent {
    u_char addr[6];  // 地址
    char name[122];   // 名称
};
#ifndef PCAP_ETHERS_FILE
#define PCAP_ETHERS_FILE "/etc/ethers"  // /etc/ethers 文件路径
#endif
PCAP_API struct    pcap_etherent *pcap_next_etherent(FILE *);  // 获取下一个以太网地址
PCAP_API u_char *pcap_ether_hostton(const char*);  // 主机名到地址
PCAP_API u_char *pcap_ether_aton(const char *);  // 字符串到地址

PCAP_API
PCAP_DEPRECATED("this is not reentrant; use 'pcap_nametoaddrinfo' instead")
bpf_u_int32 **pcap_nametoaddr(const char *);  // 名称到地址
PCAP_API struct addrinfo *pcap_nametoaddrinfo(const char *);  // 名称到地址信息
PCAP_API bpf_u_int32 pcap_nametonetaddr(const char *);  // 名称到网络地址

PCAP_API int    pcap_nametoport(const char *, int *, int *);  // 名称到端口
PCAP_API int    pcap_nametoportrange(const char *, int *, int *, int *);  // 名称到端口范围
PCAP_API int    pcap_nametoproto(const char *);  // 名称到协议
PCAP_API int    pcap_nametoeproto(const char *);  // 名称到扩展协议
PCAP_API int    pcap_nametollc(const char *);  // 名称到 LLC
/*
 * 如果协议未知，则返回 PROTO_UNDEF。
 * 此外，pcap_nametoport() 返回端口号和协议。
 * 如果 /etc/services 中存在歧义的条目（即域可以是 tcp 或 udp），则返回 PROTO_UNDEF。
 */
#define PROTO_UNDEF        -1

#ifdef __cplusplus
}
#endif

#endif
```