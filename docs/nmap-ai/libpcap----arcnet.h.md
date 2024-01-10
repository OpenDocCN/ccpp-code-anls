# `nmap\libpcap\arcnet.h`

```
/*
 * 版权声明，版权归加利福尼亚大学校董会所有
 * 允许以源代码或二进制形式进行再发布和使用，无论是否经过修改
 * 需满足以下条件：
 * 1. 源代码的再发布必须保留上述版权声明、条件列表和以下免责声明
 * 2. 二进制形式的再发布必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 * 3. 所有宣传材料提及本软件的特性或使用必须显示以下声明：
 *    本产品包含加利福尼亚大学伯克利分校及其贡献者开发的软件
 * 4. 未经特定书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件派生的产品
 *
 * 本软件由校董会和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保
 * 在任何情况下，校董会或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）承担责任
 * 无论是合同责任、严格责任还是侵权行为（包括疏忽或其他方式）引起的任何理论，即使已被告知可能发生此类损害，也不承担任何责任
 *
 * 来源：NetBSD: if_arc.h,v 1.13 1999/11/19 20:41:19 thorpej Exp
 */

/* RFC 1051 */
#define    ARCTYPE_IP_OLD        240    /* IP protocol */
# 定义旧版地址解析协议的类型码
#define    ARCTYPE_ARP_OLD        241    /* address resolution protocol */

# RFC 1201规定的协议类型码
#define    ARCTYPE_IP        212    /* IP protocol */
#define    ARCTYPE_ARP        213    /* address resolution protocol */
#define    ARCTYPE_REVARP        214    /* reverse addr resolution protocol */

# 苹果公司的Appletalk协议类型码
#define    ARCTYPE_ATALK        221    /* Appletalk */

# Banyan Vines协议类型码
#define    ARCTYPE_BANIAN        247    /* Banyan Vines */

# Novell IPX协议类型码
#define    ARCTYPE_IPX        250    /* Novell IPX */

# IPng协议类型码
#define ARCTYPE_INET6        0xc4    /* IPng */

# 根据ANSI/ATA 878.1规定的诊断协议类型码
#define ARCTYPE_DIAGNOSE    0x80    /* as per ANSI/ATA 878.1 */
```