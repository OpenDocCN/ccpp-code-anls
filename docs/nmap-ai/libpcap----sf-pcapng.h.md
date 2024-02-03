# `nmap\libpcap\sf-pcapng.h`

```cpp
/*
 * 版权声明，版权归加利福尼亚大学所有
 * 保留所有权利。
 *
 * 允许在源代码和二进制形式下进行再发布和使用，无论是否进行修改，只要满足以下条件：
 * (1) 源代码发布必须保留以上版权声明和本段完整内容，
 * (2) 包含二进制代码的发布必须在文档或其他提供的材料中包含以上版权声明和本段完整内容，
 * (3) 所有提及此软件特性或使用的广告材料必须显示以下声明：
 * “本产品包括由加利福尼亚大学，劳伦斯伯克利实验室及其贡献者开发的软件。”
 * 未经事先书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品。
 * 本软件按“原样”提供，不附带任何明示或暗示的保证，包括但不限于对适销性和特定用途的暗示保证。
 *
 * sf-pcapng.h - pcapng文件格式特定的例程
 *
 * 用于读取pcapng保存文件。
 */

#ifndef sf_pcapng_h
#define    sf_pcapng_h

extern pcap_t *pcap_ng_check_header(const uint8_t *magic, FILE *fp,
    u_int precision, char *errbuf, int *err);

#endif
```