# `nmap\libpcap\sunatmpos.h`

```
/*
 * 版权声明，版权归 Yen Yen Lim 和 North Dakota State University 所有
 * 保留所有权利
 *
 * 在源代码和二进制形式下重新分发和使用，无论是否经过修改，都是允许的，但必须满足以下条件：
 * 1. 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明
 * 2. 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 * 3. 所有宣传材料提及此软件的特性或使用都必须显示以下声明：
 *      本产品包含 Yen Yen Lim 和 North Dakota State University 开发的软件
 * 4. 未经特定事先书面许可，不得使用作者的姓名来认可或推广从本软件衍生的产品
 *
 * 本软件由作者提供"按原样"，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。在任何情况下，作者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）承担责任，无论是合同责任、严格责任还是侵权行为（包括疏忽或其他方式）引起的任何责任，即使已被告知可能发生此类损害的可能性。
 */

/* ATM 数据包的 SunATM 头部 */
#define SUNATM_DIR_POS        0   // 方向位置
#define SUNATM_VPI_POS        1   // VPI 位置
#define SUNATM_VCI_POS        2   // VCI 位置
#define SUNATM_PKT_BEGIN_POS    4   // ATM 数据包的起始位置
# 在 SUNATM_DIR_POS 的字节的最低四位中，定义了协议类型的数值
# 表示 LANE 协议
#define PT_LANE        0x01    /* LANE */
# 表示 LLC 封装
#define PT_LLC        0x02    /* LLC encapsulation */
# 表示 ILMI 协议
#define PT_ILMI        0x05    /* ILMI */
# 表示 Q.SAAL 协议
#define PT_QSAAL    0x06    /* Q.SAAL */
```