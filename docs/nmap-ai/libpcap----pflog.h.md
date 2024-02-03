# `nmap\libpcap\pflog.h`

```cpp
/*
 * 版权声明，版权归加利福尼亚大学理事会所有
 */
/*
 * 在源代码和二进制形式中重新分发和使用，需满足以下条件：
 * 1. 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明
 * 2. 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 * 3. 所有提及此软件特性或使用的广告材料必须显示以下声明：
 *    本产品包括加利福尼亚大学伯克利分校及其贡献者开发的软件
 * 4. 未经特定事先书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品
 *
 * 本软件由理事会和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。在任何情况下，理事会或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）承担责任，无论是合同责任、严格责任还是侵权行为（包括疏忽或其他方式）产生的任何理论，即使已被告知可能发生此类损害。
 */
/*
 * pflog 头文件，至少是目前存在的样子
 */
#define PFLOG_IFNAMSIZ        16
#define PFLOG_RULESET_NAME_SIZE    16

/*
 * 方向值
 */
// 定义用于规则的匹配方向的常量
#define PF_INOUT    0
#define PF_IN        1
#define PF_OUT        2
#if defined(__OpenBSD__)
#define PF_FWD        3
#endif

/*
 * 原因数值
 */
#define PFRES_MATCH    0
#define PFRES_BADOFF    1
#define PFRES_FRAG    2
#define PFRES_SHORT    3
#define PFRES_NORM    4
#define PFRES_MEMORY    5
#define PFRES_TS    6
#define PFRES_CONGEST    7
#define PFRES_IPOPTIONS 8
#define PFRES_PROTCKSUM 9
#define PFRES_BADSTATE    10
#define PFRES_STATEINS    11
#define PFRES_MAXSTATES    12
#define PFRES_SRCLIMIT    13
#define PFRES_SYNPROXY    14
#if defined(__FreeBSD__)
#define PFRES_MAPFAILED    15
#elif defined(__NetBSD__)
#define PFRES_STATELOCKED 15
#elif defined(__OpenBSD__)
#define PFRES_TRANSLATE    15
#define PFRES_NOROUTE    16
#elif defined(__APPLE__)
#define PFRES_DUMMYNET  15
#endif

/*
 * 动作数值
 */
#define PF_PASS            0
#define PF_DROP            1
#define PF_SCRUB        2
#define PF_NOSCRUB        3
#define PF_NAT            4
#define PF_NONAT        5
#define PF_BINAT        6
#define PF_NOBINAT        7
#define PF_RDR            8
#define PF_NORDR        9
#define PF_SYNPROXY_DROP    10
#if defined(__FreeBSD__)
#define PF_DEFER        11
#elif defined(__OpenBSD__)
#define PF_DEFER        11
#define PF_MATCH        12
#define PF_DIVERT        13
#define PF_RT            14
#define PF_AFRT            15
#elif defined(__APPLE__)
#define PF_DUMMYNET        11
#define PF_NODUMMYNET        12
#define PF_NAT64        13
#define PF_NONAT64        14
#endif

// 定义用于存储 IP 地址的结构
struct pf_addr {
    union {
        struct in_addr        v4;
        struct in6_addr        v6;
        uint8_t            addr8[16];
        uint16_t        addr16[8];
        uint32_t        addr32[4];
    } pfa;            /* 128-bit address */
#define v4    pfa.v4
#define v6    pfa.v6
#define addr8    pfa.addr8
#define addr16    pfa.addr16
#define addr32    pfa.addr32
};

// 定义用于存储 pflog 头部信息的结构
struct pfloghdr {
    uint8_t        length;
    uint8_t        af;
    uint8_t        action;
    # 声明一个无符号8位整数变量reason
    uint8_t        reason;
    # 声明一个长度为PFLOG_IFNAMSIZ的字符数组ifname
    char        ifname[PFLOG_IFNAMSIZ];
    # 声明一个长度为PFLOG_RULESET_NAME_SIZE的字符数组ruleset
    char        ruleset[PFLOG_RULESET_NAME_SIZE];
    # 声明一个无符号32位整数变量rulenr
    uint32_t    rulenr;
    # 声明一个无符号32位整数变量subrulenr
    uint32_t    subrulenr;
    # 声明一个无符号32位整数变量uid
    uint32_t    uid;
    # 声明一个有符号32位整数变量pid
    int32_t        pid;
    # 声明一个无符号32位整数变量rule_uid
    uint32_t    rule_uid;
    # 声明一个有符号32位整数变量rule_pid
    int32_t        rule_pid;
    # 声明一个无符号8位整数变量dir
    uint8_t        dir;
#if defined(__OpenBSD__)
    // 如果定义了 __OpenBSD__，则定义三个 uint8_t 类型的变量
    uint8_t        rewritten;
    uint8_t        naf;
    uint8_t        pad[1];
#else
    // 如果未定义 __OpenBSD__，则定义一个包含3个元素的 uint8_t 类型的数组
    uint8_t        pad[3];
#endif
#if defined(__FreeBSD__)
    // 如果定义了 __FreeBSD__，则定义一个 uint32_t 类型的变量和一个 uint8_t 类型的变量，以及一个包含3个元素的 uint8_t 类型的数组
    uint32_t    ridentifier;
    uint8_t        reserve;
    uint8_t        pad2[3];
#elif defined(__OpenBSD__)
    // 如果定义了 __OpenBSD__，则定义两个 struct pf_addr 类型的变量和两个 uint16_t 类型的变量
    struct pf_addr    saddr;
    struct pf_addr    daddr;
    uint16_t    sport;
    uint16_t    dport;
#endif
};
```