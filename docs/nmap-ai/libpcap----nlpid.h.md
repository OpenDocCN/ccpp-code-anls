# `nmap\libpcap\nlpid.h`

```
/*
 * 版权声明
 *    1996年 Juniper Networks, Inc.  保留所有权利。
 *
 * 在源代码和二进制形式下重新分发和使用，无论是否经过修改，都是允许的，前提是：（1）源代码分发中保留以上版权声明和本段完整内容，（2）包含二进制代码的分发在文档或其他提供的材料中包含以上版权声明和本段完整内容。未经特定事先书面许可，不得使用 Juniper Networks 的名称来认可或推广从本软件衍生的产品。
 *
 * 本软件按原样提供，没有任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。
 */

/* 一些系统中缺少的类型 */

/*
 * 网络层协议标识符
 */
#ifndef ISO8473_CLNP
#define ISO8473_CLNP        0x81
#endif
#ifndef    ISO9542_ESIS
#define    ISO9542_ESIS        0x82
#endif
#ifndef ISO9542X25_ESIS
#define ISO9542X25_ESIS        0x8a
#endif
#ifndef    ISO10589_ISIS
#define    ISO10589_ISIS        0x83
#endif
/*
 * 这实际上不属于 nlpid.h 文件
 * 但是我们需要它来生成漂亮的
 * 与 IS-IS 相关的 BPF 过滤器
 */
#define ISIS_L1_LAN_IIH      15
#define ISIS_L2_LAN_IIH      16
#define ISIS_PTP_IIH         17
#define ISIS_L1_LSP          18
#define ISIS_L2_LSP          20
#define ISIS_L1_CSNP         24
#define ISIS_L2_CSNP         25
#define ISIS_L1_PSNP         26
#define ISIS_L2_PSNP         27

#ifndef ISO8878A_CONS
#define    ISO8878A_CONS        0x84
#endif
#ifndef    ISO10747_IDRP
#define    ISO10747_IDRP        0x85
#endif
```