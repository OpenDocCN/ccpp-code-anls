# `nmap\libpcap\sf-pcap.h`

```cpp
# 版权声明，版权所有，禁止未经授权的源代码和二进制形式的再分发和使用
# 允许在源代码和二进制形式的再分发和使用时保留版权声明和完整的段落
# 所有包含二进制代码的分发都需要在文档或其他材料中包含版权声明和完整的段落
# 所有提及此软件特性或使用的广告材料都需要显示以下声明："本产品包含由加利福尼亚大学，劳伦斯伯克利实验室及其贡献者开发的软件"
# 未经特定事先书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品
# 本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保
# sf-pcap.h - libpcap文件格式特定的例程
# 由Jeffrey Mogul, DECWRL提取/创建
# 由Steve McCanne, LBL修改
# 用于将经过过滤的接收数据包头保存到文件中，然后以后读取它们
# 文件中的第一条记录包含了机器相关值的保存值，以便我们可以在任何架构上打印转储文件
# sf_pcap_h的头文件保护
# 检查pcap文件头，返回pcap_t指针
# magic: pcap文件的魔术数字
# fp: 文件指针
# precision: 精度
# errbuf: 错误缓冲区
# err: 错误码
extern pcap_t *pcap_check_header(const uint8_t *magic, FILE *fp,
    u_int precision, char *errbuf, int *err);
```