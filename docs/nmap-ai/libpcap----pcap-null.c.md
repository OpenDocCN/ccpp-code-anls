# `nmap\libpcap\pcap-null.c`

```cpp
/*
 * 版权声明
 * 版权所有 (c) 1994, 1995, 1996
 * 加利福尼亚大学理事会。保留所有权利。
 *
 * 在源代码和二进制形式下的再发布和使用，无论是否经过修改，都是允许的，前提是：(1) 源代码发布
 * 保留上述版权声明和本段文字的完整性，(2) 包含二进制代码的发布包括上述版权声明和本段文字的完整性
 * 在文档或其他提供的材料中，并且 (3) 所有提及此软件特性或使用的广告材料都显示以下声明：
 * “本产品包括由加利福尼亚大学劳伦斯伯克利实验室及其贡献者开发的软件。”未经事先书面许可，
 * 不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品。
 * 本软件按“原样”提供，没有任何明示或暗示的保证，包括但不限于对适销性和特定用途的暗示保证。
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <string.h>

#include "pcap-int.h"

static char nosup[] = "live packet capture not supported on this system";

// 创建一个捕获接口的 pcap_t 结构体，但在此系统上不支持
pcap_t *
pcap_create_interface(const char *device _U_, char *ebuf)
{
    (void)pcap_strlcpy(ebuf, nosup, PCAP_ERRBUF_SIZE);
    return (NULL);
}

// 在此系统上找不到可捕获的接口
int
pcap_platform_finddevs(pcap_if_list_t *devlistp _U_, char *errbuf _U_)
{
    /*
     * 没有可以捕获的接口。
     */
    return (0);
}

#ifdef _WIN32
// 在 Windows 系统上查找网络接口的网络地址和子网掩码，但在此系统上不支持
int
pcap_lookupnet(const char *device _U_, bpf_u_int32 *netp _U_,
    bpf_u_int32 *maskp _U_, char *errbuf)
{
    (void)pcap_strlcpy(errbuf, nosup, PCAP_ERRBUF_SIZE);
    return (-1);
}
#endif

/*
 * Libpcap 版本字符串。
 */
const char *
pcap_lib_version(void)
{
    return (PCAP_VERSION_STRING);
}
```