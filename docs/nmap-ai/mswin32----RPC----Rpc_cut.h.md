# `nmap\mswin32\RPC\Rpc_cut.h`

```cpp
/*
 * 版权声明
 * 1999年，2000年，都灵理工大学。保留所有权利。
 *
 * 在源代码和二进制形式下重新分发和使用，无论是否经过修改，都是允许的，前提是：（1）源代码分发中保留以上版权声明和本段完整内容，（2）包含二进制代码的分发在文档或其他提供的材料中包含以上版权声明和本段完整内容，（3）所有提及此软件特性或使用的广告材料都显示以下声明：“本产品包括都灵理工大学及其贡献者开发的软件。”未经特定事先书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品。
 * 本软件按“原样”提供，不附带任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。
 */

// 定义无符号长整型
#define u_long unsigned long
// 定义无符号整型
#define u_int unsigned int
// 定义无符号短整型
#define u_short unsigned short
// 定义枚举类型为整型
#define enum_t int
// 定义字符指针类型
#define caddr_t char*

// 定义不透明认证结构体
struct opaque_auth {
    enum_t    oa_flavor;        // 认证的类型
    caddr_t    oa_base;        // 更多认证信息的地址
    u_int    oa_length;        // 不超过最大认证字节数
};

// 定义接受的回复结构体
struct accepted_reply {
    struct opaque_auth    ar_verf;  // 认证信息
    enum accept_stat    ar_stat;  // 接受状态
    union {
        struct {
            u_long    low;  // 低位
            u_long    high;  // 高位
        } AR_versions;  // 版本信息
        /*struct {
            caddr_t    where;
            xdrproc_t proc;
        } AR_results;
        /* 和许多其他空情况 */
    } ru;
#define    ar_results    ru.AR_results  // 结果
#define    ar_vers        ru.AR_versions  // 版本
};

// 定义拒绝的回复结构体
struct rejected_reply {
    enum reject_stat rj_stat;  // 拒绝状态
    # 定义一个联合体，可以同时存储两种不同类型的数据
    union {
        # 定义一个结构体，包含两个无符号长整型变量low和high
        struct {
            u_long low;
            u_long high;
        } RJ_versions;
        # 定义一个枚举类型变量RJ_why，表示认证失败的原因
        enum auth_stat RJ_why;  /* why authentication did not work */
    } ru;
// 定义结构体 reply_body，包含枚举类型 reply_stat 和联合体 ru
struct reply_body {
    enum reply_stat rp_stat; // 声明枚举类型变量 rp_stat
    union {
        struct accepted_reply RP_ar; // 声明结构体变量 RP_ar
        struct rejected_reply RP_dr; // 声明结构体变量 RP_dr
    } ru; // 声明联合体变量 ru
#define    rp_acpt    ru.RP_ar // 定义宏 rp_acpt，表示 ru.RP_ar
#define    rp_rjct    ru.RP_dr // 定义宏 rp_rjct，表示 ru.RP_dr
};

// 定义结构体 call_body，包含多个 u_long 类型的变量和两个结构体变量
struct call_body {
    u_long cb_rpcvers;    /* must be equal to two */ // 声明 u_long 类型变量 cb_rpcvers
    u_long cb_prog; // 声明 u_long 类型变量 cb_prog
    u_long cb_vers; // 声明 u_long 类型变量 cb_vers
    u_long cb_proc; // 声明 u_long 类型变量 cb_proc
    struct opaque_auth cb_cred; // 声明结构体变量 cb_cred
    struct opaque_auth cb_verf; /* protocol specific - provided by client */ // 声明结构体变量 cb_verf
};

// 定义结构体 rpc_msg，包含多个 u_long 类型的变量和一个枚举类型 rm_direction 和一个联合体 ru
struct rpc_msg {
    u_long            rm_xid; // 声明 u_long 类型变量 rm_xid
    enum msg_type        rm_direction; // 声明枚举类型变量 rm_direction
    union {
        struct call_body RM_cmb; // 声明结构体变量 RM_cmb
        struct reply_body RM_rmb; // 声明结构体变量 RM_rmb
    } ru; // 声明联合体变量 ru
#define    rm_call        ru.RM_cmb // 定义宏 rm_call，表示 ru.RM_cmb
#define    rm_reply    ru.RM_rmb // 定义宏 rm_reply，表示 ru.RM_rmb
};

// 定义多个宏，表示不同的状态和错误码
#define    MSG_ACCEPTED 0
#define MSG_DENIED 1
#define SUCCESS 0
#define PROG_UNAVAIL 1
#define PROG_MISMATCH 2
#define PROC_UNAVAIL 3
#define GARBAGE_ARGS 4
#define SYSTEM_ERR 5

// 定义多个宏，表示不同的端口和程序版本
#define PMAPPORT        ((u_short)111)
#define PMAPPROG        ((u_long)100000)
#define PMAPVERS        ((u_long)2)
#define PMAPVERS_PROTO        ((u_long)2)
#define PMAPVERS_ORIG        ((u_long)1)
#define PMAPPROC_NULL        ((u_long)0)
#define PMAPPROC_SET        ((u_long)1)
#define PMAPPROC_UNSET        ((u_long)2)
#define PMAPPROC_GETPORT    ((u_long)3)
#define PMAPPROC_DUMP        ((u_long)4)
#define PMAPPROC_CALLIT        ((u_long)5)

// 定义两个宏，表示消息类型
#define CALL 0
#define REPLY 1
```