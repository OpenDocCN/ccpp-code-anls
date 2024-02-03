# `nmap\libpcap\llc.h`

```cpp
/*
 * 版权声明
 * 版权所有（c）1993年，1994年，1997年
 * 加利福尼亚大学理事会。保留所有权利。
 *
 * 在源代码和二进制形式中重新分发和使用，无论是否经过修改，都是允许的，前提是：（1）源代码分发
 * 保留上述版权声明和本段文字的完整性，（2）包含二进制代码的分发包括上述版权声明
 * 和本段文字的完整性在文档或其他提供的材料中，（3）所有宣传材料提及
 * 此软件的特性或使用显示以下声明：
 * “本产品包括由加利福尼亚大学，劳伦斯伯克利实验室及其贡献者开发的软件。”不得使用加利福尼亚大学的名称
 * 或其贡献者的名称来认可或推广从本软件衍生的产品，除非事先得到明确的书面许可。
 * 本软件按“原样”提供，没有任何明示或暗示的保证，
 * 包括但不限于，适销性和特定用途的暗示保证。
 */

/*
 * LLC头部信息的定义。
 */

#define    LLC_U_FMT    3
#define    LLC_GSAP    1
#define    LLC_IG            1 /* 个体 / 组 */
#define LLC_S_FMT    1

#define    LLC_U_POLL    0x10
#define    LLC_IS_POLL    0x0100
#define    LLC_XID_FI    0x81

#define LLC_U_CMD_MASK    0xef
#define    LLC_UI        0x03
#define    LLC_UA        0x63
#define    LLC_DISC    0x43
#define    LLC_DM        0x0f
#define    LLC_SABME    0x6f
#define    LLC_TEST    0xe3
#define    LLC_XID        0xaf
#define    LLC_FRMR    0x87

#define LLC_S_CMD_MASK    0x0f
#define    LLC_RR        0x0001
#define    LLC_RNR        0x0005
#define    LLC_REJ        0x0009

#define LLC_IS_NR(is)    (((is) >> 9) & 0x7f)
#define LLC_I_NS(is)    (((is) >> 1) & 0x7f)

/*
 * 802.2 LLC SAP值。
 */
# 如果未定义 LLCSAP_NULL，则将其定义为 0x00
#ifndef LLCSAP_NULL
#define    LLCSAP_NULL        0x00
#endif
# 如果未定义 LLCSAP_GLOBAL，则将其定义为 0xff
#ifndef LLCSAP_GLOBAL
#define    LLCSAP_GLOBAL        0xff
#endif
# 如果未定义 LLCSAP_8021B_I，则将其定义为 0x02
#ifndef LLCSAP_8021B_I
#define    LLCSAP_8021B_I        0x02
#endif
# 如果未定义 LLCSAP_8021B_G，则将其定义为 0x03
#ifndef LLCSAP_8021B_G
#define    LLCSAP_8021B_G        0x03
#endif
# 如果未定义 LLCSAP_IP，则将其定义为 0x06
#ifndef LLCSAP_IP
#define    LLCSAP_IP        0x06
#endif
# 如果未定义 LLCSAP_PROWAYNM，则将其定义为 0x0e
#ifndef LLCSAP_PROWAYNM
#define    LLCSAP_PROWAYNM        0x0e
#endif
# 如果未定义 LLCSAP_8021D，则将其定义为 0x42
#ifndef LLCSAP_8021D
#define    LLCSAP_8021D        0x42
#endif
# 如果未定义 LLCSAP_RS511，则将其定义为 0x4e
#ifndef LLCSAP_RS511
#define    LLCSAP_RS511        0x4e
#endif
# 如果未定义 LLCSAP_ISO8208，则将其定义为 0x7e
#ifndef LLCSAP_ISO8208
#define    LLCSAP_ISO8208        0x7e
#endif
# 如果未定义 LLCSAP_PROWAY，则将其定义为 0x8e
#ifndef LLCSAP_PROWAY
#define    LLCSAP_PROWAY        0x8e
#endif
# 如果未定义 LLCSAP_SNAP，则将其定义为 0xaa
#ifndef LLCSAP_SNAP
#define    LLCSAP_SNAP        0xaa
#endif
# 如果未定义 LLCSAP_IPX，则将其定义为 0xe0
#ifndef LLCSAP_IPX
#define LLCSAP_IPX        0xe0
#endif
# 如果未定义 LLCSAP_NETBEUI，则将其定义为 0xf0
#ifndef LLCSAP_NETBEUI
#define LLCSAP_NETBEUI        0xf0
#endif
# 如果未定义 LLCSAP_ISONS，则将其定义为 0xfe
#ifndef LLCSAP_ISONS
#define    LLCSAP_ISONS        0xfe
#endif
```