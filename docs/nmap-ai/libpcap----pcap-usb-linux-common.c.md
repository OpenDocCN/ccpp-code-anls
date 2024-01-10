# `nmap\libpcap\pcap-usb-linux-common.c`

```
/*
 * 版权声明
 * 1993年至1997年，加利福尼亚大学理事会保留所有权利
 * 允许在源代码和二进制形式下进行再发布和使用，无论是否进行修改：
 * (1) 源代码发布必须保留上述版权声明和本段完整内容
 * (2) 包含二进制代码的发布必须在文档或其他提供的材料中包含上述版权声明和本段完整内容
 * (3) 所有提及此软件特性或使用的广告材料必须显示以下声明：
 * “本产品包括由加利福尼亚大学劳伦斯伯克利实验室及其贡献者开发的软件。”
 * 未经特定事先书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件派生的产品。
 * 本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。
 * pcap-usb-linux-common.c - 用于处理 Linux USB 捕获的通用代码
 */

#include "pcap/pcap.h"
#include "pcap/usb.h"

#include "pcap-usb-linux-common.h"

/*
 * 从 Linux USB 内存映射捕获机制提供的数据中计算出，如果捕获机制在末尾截断了任何数据，那么本应提供的数据包长度
 * 将“未截断长度”字段设置为该值
 */
void
fix_linux_usb_mmapped_length(struct pcap_pkthdr *pkth, const u_char *bp)
{
    const pcap_usb_header_mmapped *hdr;
    u_int bytes_left;

    /*
     * 所有调用此例程的调用者必须确保 pkth->caplen >= sizeof (pcap_usb_header_mmapped)
     */
    bytes_left = pkth->caplen;
    # 从剩余的字节数中减去 pcap_usb_header_mmapped 结构的大小
    bytes_left -= sizeof (pcap_usb_header_mmapped);
    # 将 bp 强制转换为 pcap_usb_header_mmapped 结构，并赋值给 hdr
    hdr = (const pcap_usb_header_mmapped *) bp;
    }
# 闭合前面的函数定义
```