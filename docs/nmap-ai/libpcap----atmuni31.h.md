# `nmap\libpcap\atmuni31.h`

```
/*
 * 版权所有（c）1997年Yen Yen Lim和北达科他州立大学
 * 保留所有权利。
 *
 * 在源代码和二进制形式中重新分发和使用，无论是否经过修改，都是允许的，只要满足以下条件：
 * 1. 源代码的再分发必须保留上述版权声明、此条件列表和以下免责声明。
 * 2. 以二进制形式重新分发时，必须在文档和/或其他提供的材料中复制上述版权声明、此条件列表和以下免责声明。
 * 3. 所有提及此软件特性或使用的广告材料必须显示以下确认：
 *      本产品包括Yen Yen Lim和北达科他州立大学开发的软件
 * 4. 未经特定事先书面许可，不得使用作者的姓名来认可或推广从本软件衍生的产品。
 *
 * 本软件由作者“按原样”提供，任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保，都是被拒绝的。在任何情况下，作者都不对任何直接、间接、偶然、特殊、惩罚性或后果性的损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）承担责任，无论是在合同、严格责任或侵权行为（包括疏忽或其他方式）的任何责任理论下，即使已被告知可能发生此类损害的可能性。
 */

/* 基于ATM Forum的UNI3.1标准 */

/* 基于VPI=0和以下VCI的ATM流量类型 */
#define VCI_PPC            0x05    /* 点对点信号消息 */
#define VCI_BCC            0x02    /* 广播信号消息 */
# 定义VCI_OAMF4SC常量，表示Segment OAM F4流单元
#define VCI_OAMF4SC        0x03    /* Segment OAM F4 flow cell */
# 定义VCI_OAMF4EC常量，表示End-to-end OAM F4流单元
#define VCI_OAMF4EC        0x04    /* End-to-end OAM F4 flow cell */
# 定义VCI_METAC常量，表示Meta信号消息
#define VCI_METAC        0x01    /* Meta signal msg */
# 定义VCI_ILMIC常量，表示ILMI消息
#define VCI_ILMIC        0x10    /* ILMI msg */

# 定义Q.2931信令消息
#define CALL_PROCEED        0x02    /* call proceeding */
#define CONNECT            0x07    /* connect */
#define CONNECT_ACK        0x0f    /* connect_ack */
#define SETUP            0x05    /* setup */
#define RELEASE            0x4d    /* release */
#define RELEASE_DONE        0x5a    /* release_done */
#define RESTART            0x46    /* restart */
#define RESTART_ACK        0x4e    /* restart ack */
#define STATUS            0x7d    /* status */
#define STATUS_ENQ        0x75    /* status ack */
#define ADD_PARTY        0x80    /* add party */
#define ADD_PARTY_ACK        0x81    /* add party ack */
#define ADD_PARTY_REJ        0x82    /* add party rej */
#define DROP_PARTY        0x83    /* drop party */
#define DROP_PARTY_ACK        0x84    /* drop party ack */

# 定义信令消息中的信息元素参数
#define CAUSE            0x08    /* cause */
#define ENDPT_REF        0x54    /* endpoint reference */
#define AAL_PARA        0x58    /* ATM adaptation layer parameters */
#define TRAFF_DESCRIP        0x59    /* atm traffic descriptors */
#define CONNECT_ID        0x5a    /* connection identifier */
#define QOS_PARA        0x5c    /* quality of service parameters */
#define B_HIGHER        0x5d    /* broadband higher layer information */
#define B_BEARER        0x5e    /* broadband bearer capability */
#define B_LOWER            0x5f    /* broadband lower information */
#define CALLING_PARTY        0x6c    /* calling party number */
#define CALLED_PARTY        0x70    /* called party nmber */

# 定义Q2931常量，表示Q.2931信令
#define Q2931            0x09

# 定义Q.2931信令的一般消息格式中的协议位置偏移量
#define PROTO_POS       0    /* offset of protocol discriminator */
# 定义调用引用值的偏移量
#define CALL_REF_POS    2    /* offset of call reference value */
# 定义消息类型的偏移量
#define MSG_TYPE_POS    5    /* offset of message type */
# 定义消息长度的偏移量
#define MSG_LEN_POS     7    /* offset of message length */
# 定义第一个信息元素的偏移量
#define IE_BEGIN_POS    9    /* offset of first information element */

# 定义信令消息的格式
# 消息类型的偏移量
#define TYPE_POS    0
# 消息长度的偏移量
#define LEN_POS        2
# 第一个字段的偏移量
#define FIELD_BEGIN_POS 4
```