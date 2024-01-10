# `nmap\libpcap\gencode.h`

```
/*
 * 版权声明，版权归加利福尼亚大学所有
 * 允许在源代码和二进制形式下进行再发布和使用，无论是否进行修改
 * 允许在源代码发布中保留以上版权声明和本段文字，以及在包含二进制代码的发布中在文档或其他提供的材料中保留以上版权声明和本段文字
 * 所有提及此软件特性或使用的广告材料都必须显示以下声明：
 * “本产品包含由加利福尼亚大学、劳伦斯伯克利实验室及其贡献者开发的软件。”
 * 未经特定事先书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品
 * 本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保
 */

#ifndef gencode_h
#define gencode_h

#include "pcap/funcattrs.h"
/*
 * pcap/bpf.h（一个公共头文件）需要 u_char、u_short 和 u_int，这些可以通过 pcap-types.h（一个私有头文件）或 pcap/pcap.h（一个公共头文件）来提供，而 pcap/bpf.h 并不包含其中任何一个。为了简化事情，包含私有头文件，这样这个私有头文件应该可以在其他文件中早期包含时编译通过。
 */
#include "pcap-types.h"
#include "pcap/bpf.h" /* bpf_u_int32 and BPF_MEMWORDS */
/*
 * ATM support:
 * ATM支持：
 *
 * Copyright (c) 1997 Yen Yen Lim and North Dakota State University
 * 版权所有：1997年Yen Yen Lim和北达科他州立大学
 *
 * All rights reserved.
 * 保留所有权利。
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions
 * are met:
 * 可以在源代码和二进制形式中重新分发和使用，无论是否经过修改，前提是满足以下条件：
 *
 * 1. Redistributions of source code must retain the above copyright
 *    notice, this list of conditions and the following disclaimer.
 * 1. 源代码的重新分发必须保留上述版权声明、条件列表和以下免责声明。
 *
 * 2. Redistributions in binary form must reproduce the above copyright
 *    notice, this list of conditions and the following disclaimer in the
 *    documentation and/or other materials provided with the distribution.
 * 2. 以二进制形式重新分发必须在文档和/或其他提供的材料中复制上述版权声明、条件列表和以下免责声明。
 *
 * 3. All advertising materials mentioning features or use of this software
 *    must display the following acknowledgement:
 * 3. 所有提及此软件特性或使用的广告材料必须显示以下声明：
 *      This product includes software developed by Yen Yen Lim and
 *      North Dakota State University
 *      本产品包含Yen Yen Lim和北达科他州立大学开发的软件
 *
 * 4. The name of the author may not be used to endorse or promote products
 *    derived from this software without specific prior written permission.
 * 4. 未经特定事先书面许可，不得使用作者的姓名来代言或推广从本软件衍生的产品。
 *
 * THIS SOFTWARE IS PROVIDED BY THE AUTHOR ``AS IS'' AND ANY EXPRESS OR
 * IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
 * WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
 * DISCLAIMED.  IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY DIRECT,
 * INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES
 * (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
 * SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION)
 * HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT,
 * STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN
 * ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
 * POSSIBILITY OF SUCH DAMAGE.
 * 本软件由作者“按原样”提供，任何明示或暗示的保证，包括但不限于对适销性和特定用途的暗示保证，都是被拒绝的。在任何情况下，作者都不对任何直接、间接、附带、特殊、惩罚性或后果性的损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）承担责任，无论是在合同、严格责任或侵权行为（包括疏忽或其他方式）的任何责任理论下，即使已被告知使用本软件的可能性。
 */

/* Address qualifiers. */
/* 地址限定符。 */

#define Q_HOST        1
#define Q_NET        2
#define Q_PORT        3
#define Q_GATEWAY    4
#define Q_PROTO        5
#define Q_PROTOCHAIN    6
#define Q_PORTRANGE    7

/* Protocol qualifiers. */
/* 协议限定符。 */
# 定义不同类型的链路层协议
#define Q_LINK        1
#define Q_IP        2
#define Q_ARP        3
#define Q_RARP        4
#define Q_SCTP        5
#define Q_TCP        6
#define Q_UDP        7
#define Q_ICMP        8
#define Q_IGMP        9
#define Q_IGRP        10

#define    Q_ATALK        11
#define    Q_DECNET    12
#define    Q_LAT        13
#define Q_SCA        14
#define    Q_MOPRC        15
#define    Q_MOPDL        16

#define Q_IPV6        17
#define Q_ICMPV6    18
#define Q_AH        19
#define Q_ESP        20

#define Q_PIM        21
#define Q_VRRP        22

#define Q_AARP        23

#define Q_ISO        24
#define Q_ESIS        25
#define Q_ISIS        26
#define Q_CLNP        27

#define Q_STP        28

#define Q_IPX        29

#define Q_NETBEUI    30

# IS-IS Levels
#define Q_ISIS_L1       31
#define Q_ISIS_L2       32
# PDU types
#define Q_ISIS_IIH      33
#define Q_ISIS_SNP      34
#define Q_ISIS_CSNP     35
#define Q_ISIS_PSNP     36
#define Q_ISIS_LSP      37

#define Q_RADIO        38

#define Q_CARP        39

# 方向限定符
#define Q_SRC        1
#define Q_DST        2
#define Q_OR        3
#define Q_AND        4
#define Q_ADDR1        5
#define Q_ADDR2        6
#define Q_ADDR3        7
#define Q_ADDR4        8
#define Q_RA        9
#define Q_TA        10

#define Q_DEFAULT    0
#define Q_UNDEF        255

# ATM 类型
#define A_METAC        22    # 元信令电路
#define A_BCC        23    # 广播电路
#define A_OAMF4SC    24    # 段 OAM F4 电路
#define A_OAMF4EC    25    # 端到端 OAM F4 电路
#define A_SC        26    # 信令电路
#define A_ILMIC        27    # ILMI 电路
#define A_OAM        28    # OAM 细胞：仅限 F4
#define A_OAMF4        29    # OAM F4 细胞：段 + 端到端
#define A_LANE        30    # LANE 通信
#define A_LLC        31    # LLC 封装的通信

# 基于 Q.2931 信令协议
#define A_SETUP        41    /* 设置消息 */
#define A_CALLPROCEED    42    /* 呼叫进行中消息 */
#define A_CONNECT    43    /* 连接消息 */
#define A_CONNECTACK    44    /* 连接确认消息 */
#define A_RELEASE    45    /* 释放消息 */
#define A_RELEASE_DONE    46    /* 释放完成消息 */

/* ATM 字段类型 */
#define A_VPI        51
#define A_VCI        52
#define A_PROTOTYPE    53
#define A_MSGTYPE    54
#define A_CALLREFTYPE    55

#define A_CONNECTMSG    70    /* 返回用于建立和销毁交换虚拟连接的 Q.2931 信令消息 */
#define A_METACONNECT    71    /* 返回用于建立和销毁预定义虚拟电路的 Q.2931 信令消息，例如广播电路、oamf4 段电路、oamf4 端到端电路、ILMI 电路或连接信令电路。 */

/* MTP2 类型 */
#define M_FISU        22    /* FISU */
#define M_LSSU        23    /* LSSU */
#define M_MSU        24    /* MSU */

/* MTP2 HSL 类型 */
#define MH_FISU        25    /* HSL 的 FISU */
#define MH_LSSU        26    /* LSSU */
#define MH_MSU        27    /* MSU */

/* MTP3 字段类型 */
#define M_SIO        1
#define M_OPC        2
#define M_DPC        3
#define M_SLS        4

/* 在 MTP2 HSL 情况下的 MTP3 字段类型 */
#define MH_SIO        5
#define MH_OPC        6
#define MH_DPC        7
#define MH_SLS        8


struct slist;

/*
 * 单个语句，对应于块中的指令。
 */
struct stmt {
    int code;        /* 操作码 */
    struct slist *jt;    /* 仅用于块中的相对跳转 */
    struct slist *jf;    /* 仅用于块中的相对跳转 */
    bpf_u_int32 k;        /* k 字段 */
};

struct slist {
    struct stmt s;
    struct slist *next;
};
/*
 * 用位向量表示定义集合。我们假设 TOT_REGISTERS
 * 小于 8*sizeof(atomset)。
 */
typedef bpf_u_int32 atomset;
#define ATOMMASK(n) (1 << (n))  // 定义一个宏，用于生成对应位的掩码
#define ATOMELEM(d, n) (d & ATOMMASK(n))  // 定义一个宏，用于检查位集合中是否存在对应位

/*
 * 一个无界集合。
 */
typedef bpf_u_int32 *uset;

/*
 * 原子实体的总数，包括累加器（A）和索引（X）。
 * 在流分析期间，我们将这些实体都视为相似的。
 */
#define N_ATOMS (BPF_MEMWORDS+2)

/*
 * 程序的控制流图。
 * 这对应于 CFG 中的一条边。
 * 这是一个有向图，所以一条边有一个前驱和一个后继。
 */
struct edge {
    u_int id;
    int code;        /* 与该边对应的分支的操作码 */
    uset edom;
    struct block *succ;    /* 后继顶点 */
    struct block *pred;    /* 前驱顶点 */
    struct edge *next;    /* 用于节点的入边的链接列表 */
};

/*
 * 一个块是 CFG 中的一个顶点。
 * 它有一个语句列表，最后一个语句是一个
 * 分支到后继块。
 */
struct block {
    u_int id;
    struct slist *stmts;    /* 副作用语句 */
    struct stmt s;        /* 分支语句 */
    int mark;
    u_int longjt;        /* jt 分支需要长跳转 */
    u_int longjf;        /* jf 分支需要长跳转 */
    int level;
    int offset;
    int sense;
    struct edge et;        /* 对应于 jt 分支的边 */
    struct edge ef;        /* 对应于 jf 分支的边 */
    struct block *head;
    struct block *link;    /* 优化器使用的链接字段 */
    uset dom;
    uset closure;
    struct edge *in_edges;    /* 作为后继顶点的边的集合（链接列表中的第一条边） */
    atomset def, kill;
    atomset in_use;
    atomset out_use;
    int oval;        /* 分支语句中被测试值的值 ID */
    bpf_u_int32 val[N_ATOMS];
};

/*
 * val[i] 的值为 0 表示该值未知。
 */
#define VAL_UNKNOWN    0

struct arth {
    # 定义指向结构体 block 的指针 b，用于协议检查
    struct block *b;    /* protocol checks */
    # 定义指向结构体 slist 的指针 s，用于语句列表
    struct slist *s;    /* stmt list */
    # 定义整型变量 regno，表示结果的虚拟寄存器编号
    int regno;        /* virtual register number of result */
};

// 定义一个名为 qual 的结构体，包含四个无符号字符类型的成员变量
struct qual {
    unsigned char addr;
    unsigned char proto;
    unsigned char dir;
    unsigned char pad;
};

// 声明一个名为 compiler_state_t 的结构体类型
typedef struct _compiler_state compiler_state_t;

// 声明函数 gen_loadi，接受 compiler_state_t 指针和 bpf_u_int32 类型参数，返回 struct arth 指针
struct arth *gen_loadi(compiler_state_t *, bpf_u_int32);
// 声明函数 gen_load，接受 compiler_state_t 指针、整型参数、struct arth 指针和 bpf_u_int32 类型参数，返回 struct arth 指针
struct arth *gen_load(compiler_state_t *, int, struct arth *, bpf_u_int32);
// 声明函数 gen_loadlen，接受 compiler_state_t 指针，返回 struct arth 指针
struct arth *gen_loadlen(compiler_state_t *);
// 声明函数 gen_neg，接受 compiler_state_t 指针和 struct arth 指针，返回 struct arth 指针
struct arth *gen_neg(compiler_state_t *, struct arth *);
// 声明函数 gen_arth，接受 compiler_state_t 指针、整型参数、两个 struct arth 指针，返回 struct arth 指针
struct arth *gen_arth(compiler_state_t *, int, struct arth *, struct arth *);

// 声明函数 gen_and，接受两个 struct block 指针参数，无返回值
void gen_and(struct block *, struct block *);
// 声明函数 gen_or，接受两个 struct block 指针参数，无返回值
void gen_or(struct block *, struct block *);
// 声明函数 gen_not，接受一个 struct block 指针参数，无返回值
void gen_not(struct block *);

// 声明函数 gen_scode，接受 compiler_state_t 指针、字符串常量和 qual 结构体参数，返回 struct block 指针
struct block *gen_scode(compiler_state_t *, const char *, struct qual);
// 声明函数 gen_ecode，接受 compiler_state_t 指针、字符串常量和 qual 结构体参数，返回 struct block 指针
struct block *gen_ecode(compiler_state_t *, const char *, struct qual);
// 声明函数 gen_acode，接受 compiler_state_t 指针、字符串常量和 qual 结构体参数，返回 struct block 指针
struct block *gen_acode(compiler_state_t *, const char *, struct qual);
// 声明函数 gen_mcode，接受 compiler_state_t 指针、两个字符串常量、bpf_u_int32 类型参数和 qual 结构体参数，返回 struct block 指针
struct block *gen_mcode(compiler_state_t *, const char *, const char *, bpf_u_int32, struct qual);
#ifdef INET6
// 声明函数 gen_mcode6，接受 compiler_state_t 指针、两个字符串常量、bpf_u_int32 类型参数和 qual 结构体参数，返回 struct block 指针
struct block *gen_mcode6(compiler_state_t *, const char *, const char *, bpf_u_int32, struct qual);
#endif
// 声明函数 gen_ncode，接受 compiler_state_t 指针、字符串常量、bpf_u_int32 类型参数和 qual 结构体参数，返回 struct block 指针
struct block *gen_ncode(compiler_state_t *, const char *, bpf_u_int32, struct qual);
// 声明函数 gen_proto_abbrev，接受 compiler_state_t 指针和整型参数，返回 struct block 指针
struct block *gen_proto_abbrev(compiler_state_t *, int);
// 声明函数 gen_relation，接受 compiler_state_t 指针、整型参数、两个 struct arth 指针和整型参数，返回 struct block 指针
struct block *gen_relation(compiler_state_t *, int, struct arth *, struct arth *, int);
// 声明函数 gen_less，接受 compiler_state_t 指针和整型参数，返回 struct block 指针
struct block *gen_less(compiler_state_t *, int);
// 声明函数 gen_greater，接受 compiler_state_t 指针和整型参数，返回 struct block 指针
struct block *gen_greater(compiler_state_t *, int);
// 声明函数 gen_byteop，接受 compiler_state_t 指针、两个整型参数和 bpf_u_int32 类型参数，返回 struct block 指针
struct block *gen_byteop(compiler_state_t *, int, int, bpf_u_int32);
// 声明函数 gen_broadcast，接受 compiler_state_t 指针和整型参数，返回 struct block 指针
struct block *gen_broadcast(compiler_state_t *, int);
// 声明函数 gen_multicast，接受 compiler_state_t 指针和整型参数，返回 struct block 指针
struct block *gen_multicast(compiler_state_t *, int);
// 声明函数 gen_ifindex，接受 compiler_state_t 指针和整型参数，返回 struct block 指针
struct block *gen_ifindex(compiler_state_t *, int);
// 声明函数 gen_inbound，接受 compiler_state_t 指针和整型参数，返回 struct block 指针
struct block *gen_inbound(compiler_state_t *, int);

// 声明函数 gen_llc，接受 compiler_state_t 指针，返回 struct block 指针
struct block *gen_llc(compiler_state_t *);
// 声明函数 gen_llc_i，接受 compiler_state_t 指针，返回 struct block 指针
struct block *gen_llc_i(compiler_state_t *);
// 声明函数 gen_llc_s，接受 compiler_state_t 指针，返回 struct block 指针
struct block *gen_llc_s(compiler_state_t *);
// 声明函数 gen_llc_u，接受 compiler_state_t 指针，返回 struct block 指针
struct block *gen_llc_u(compiler_state_t *);
// 声明函数 gen_llc_s_subtype，接受 compiler_state_t 指针和 bpf_u_int32 类型参数，返回 struct block 指针
struct block *gen_llc_s_subtype(compiler_state_t *, bpf_u_int32);
// 生成LLC_U子类型的块
struct block *gen_llc_u_subtype(compiler_state_t *, bpf_u_int32);

// 生成VLAN标签的块
struct block *gen_vlan(compiler_state_t *, bpf_u_int32, int);
// 生成MPLS标签的块
struct block *gen_mpls(compiler_state_t *, bpf_u_int32, int);

// 生成PPPoE会话数据包的块
struct block *gen_pppoed(compiler_state_t *);
// 生成PPPoE会话数据包的块
struct block *gen_pppoes(compiler_state_t *, bpf_u_int32, int);

// 生成Geneve协议数据包的块
struct block *gen_geneve(compiler_state_t *, bpf_u_int32, int);

// 生成ATM字段代码的块
struct block *gen_atmfield_code(compiler_state_t *, int, bpf_u_int32, int, int);
// 生成ATM类型缩写的块
struct block *gen_atmtype_abbrev(compiler_state_t *, int);
// 生成ATM多播缩写的块
struct block *gen_atmmulti_abbrev(compiler_state_t *, int);

// 生成MTP2类型缩写的块
struct block *gen_mtp2type_abbrev(compiler_state_t *, int);
// 生成MTP3字段代码的块
struct block *gen_mtp3field_code(compiler_state_t *, int, bpf_u_int32, int, int);

// 生成PF接口名称的块
struct block *gen_pf_ifname(compiler_state_t *, const char *);
// 生成PF RNR的块
struct block *gen_pf_rnr(compiler_state_t *, int);
// 生成PF SRNR的块
struct block *gen_pf_srnr(compiler_state_t *, int);
// 生成PF规则集的块
struct block *gen_pf_ruleset(compiler_state_t *, char *);
// 生成PF原因的块
struct block *gen_pf_reason(compiler_state_t *, int);
// 生成PF动作的块
struct block *gen_pf_action(compiler_state_t *, int);

// 生成802.11类型的块
struct block *gen_p80211_type(compiler_state_t *, bpf_u_int32, bpf_u_int32);
// 生成802.11帧控制方向的块
struct block *gen_p80211_fcdir(compiler_state_t *, bpf_u_int32);

/*
 * 表示程序的树状块，以及当前标记。
 * 仅当块的标记等于当前标记时，块才被标记。
 * 不需要遍历代码数组，只需增加'cur_mark'即可。这样可以自动将每个元素取消标记。
 */
#define isMarked(icp, p) ((p)->mark == (icp)->cur_mark)
#define unMarkAll(icp) (icp)->cur_mark += 1
#define Mark(icp, p) ((p)->mark = (icp)->cur_mark)

struct icode {
    struct block *root;
    int cur_mark;
};

// 对icode进行优化
int bpf_optimize(struct icode *, char *);
// 设置错误信息
void bpf_set_error(compiler_state_t *, const char *, ...) PCAP_PRINTFLIKE(2, 3);

// 完成解析
int finish_parse(compiler_state_t *, struct block *);
// 复制字符串
char *sdup(compiler_state_t *, const char *);

// 将icode转换为fcode
struct bpf_insn *icode_to_fcode(struct icode *, struct block *, u_int *,
    char *);  # 这是一个函数声明，声明了一个返回类型为char指针的函数
# 声明一个名为 sappend 的函数，该函数接受两个指向 struct slist 结构体的指针作为参数
void sappend(struct slist *, struct slist *);

# 在旧版本的 Bison 中，可能没有将此声明放在 grammar.h 文件中
# 该函数接受两个参数，一个是指向 void 类型的指针，一个是指向 compiler_state_t 类型的指针
int pcap_parse(void *, compiler_state_t *);

# 定义宏 JT，它接受一个指向结构体的指针 b 作为参数，返回 b 结构体中 et 成员的 succ 成员
# 定义宏 JF，它接受一个指向结构体的指针 b 作为参数，返回 b 结构体中 ef 成员的 succ 成员
# gencode_h 文件的结尾
#endif /* gencode_h */
```