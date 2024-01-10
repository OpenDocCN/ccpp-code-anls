# `nmap\libpcap\pcap\bpf.h`

```
/*
 * 版权声明，版权归加利福尼亚大学校董会所有
 * 本代码源自Stanford/CMU enet数据包过滤器(net/enet.c)，作为4.3BSD的一部分进行分发，并由Steven McCanne和Van Jacobson在劳伦斯伯克利实验室贡献代码给伯克利。
 * 在源代码和二进制形式的再发布和使用时，需满足以下条件：
 * 1. 源代码的再发布必须保留上述版权声明、条件列表和以下免责声明。
 * 2. 二进制形式的再发布必须在文档和/或其他提供的材料中复制上述版权声明、条件列表和以下免责声明。
 * 3. 未经特定书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件派生的产品。
 * 本软件由校董和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保。在任何情况下，校董或贡献者对于任何直接、间接、附带、特殊、示范性或后果性的损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）概不负责，无论是在合同、严格责任或侵权行为（包括疏忽或其他方式）的任何责任理论下，即使已被告知可能发生此类损害。
 */
/*
 * 这是 libpcap 的精简版本的 bpf.h；它只包括代码生成器和用户空间 BPF 解释器所需的内容，以及设置过滤器等的 libpcap API。
 *
 * "pcap-bpf.c" 将包含本地操作系统版本，因为它涉及操作系统的 BPF 实现。
 *
 * 至少有两个通过 Google Code Search 找到的程序明确包含了 <pcap/bpf.h>（即使 <pcap.h>/<pcap/pcap.h> 会为你包含它），所以将那些内容移动到 <pcap/pcap.h> 会破坏一些程序的构建。
 */

/*
 * 如果我们已经包含了 <net/bpf.h>，就不要重新定义这些内容。
 * 我们假设 <net/bpf.h> 中有类似 BSD 风格的多重包含保护，这对于除了最古老的 FreeBSD 和 NetBSD 版本之外的所有版本都是正确的，或者 Tru64 UNIX 风格的多重包含保护（或者至少是 Tru64 UNIX 5.x 风格；我没有早期版本来检查），或者 AIX 风格的多重包含保护（或者至少是 AIX 5.x 风格；我没有早期版本来检查），或者 QNX 风格的多重包含保护（根据 GitHub pull request #394）。
 *
 * 我们相信它们会以与我们的定义源代码兼容和二进制兼容的方式定义结构、宏和类型。
 *
 * 我们不检查 BPF_MAJOR_VERSION，因为它由 <linux/filter.h> 定义，而一些也包含 pcap.h 的程序直接或间接地包含了 <linux/filter.h>，而 <linux/filter.h> 并没有定义我们需要的内容。我们 *确实* 防止 <linux/filter.h> 为 BPF 代码本身定义各种宏；<linux/filter.h> 中写道
 *
 *    尽量使这些值和结构与 BSD 类似，特别是需要匹配的 BPF 代码定义，以便您可以共享过滤器
 *
 * 因此，我们相信它们会以与我们的定义源代码兼容和二进制兼容的方式定义它们。
 *
 * 这也提供了我们自己的多重包含保护。
 */
# 如果未定义以下宏，则定义 lib_pcap_bpf_h
#if !defined(_NET_BPF_H_) && !defined(_NET_BPF_H_INCLUDED) && !defined(_BPF_H_) && !defined(_H_BPF) && !defined(lib_pcap_bpf_h)
#define lib_pcap_bpf_h

#include <pcap/funcattrs.h>  # 包含 pcap 库的 funcattrs.h 文件
#include <pcap/dlt.h>  # 包含 pcap 库的 dlt.h 文件

#ifdef __cplusplus  # 如果是 C++ 代码
extern "C" {  # 使用 C 语言的链接规则
#endif

/* BSD style release date */
#define BPF_RELEASE 199606  # 定义 BPF_RELEASE 为 199606

#ifdef MSDOS /* must be 32-bit */  # 如果是 MSDOS 系统
typedef long          bpf_int32;  # 定义 bpf_int32 为 long 类型
typedef unsigned long bpf_u_int32;  # 定义 bpf_u_int32 为 unsigned long 类型
#else  # 如果不是 MSDOS 系统
typedef    int bpf_int32;  # 定义 bpf_int32 为 int 类型
typedef    u_int bpf_u_int32;  # 定义 bpf_u_int32 为 u_int 类型
#endif

/*
 * Alignment macros.  BPF_WORDALIGN rounds up to the next
 * even multiple of BPF_ALIGNMENT.
 *
 * Tcpdump's print-pflog.c uses this, so we define it here.
 */
#ifndef __NetBSD__  # 如果不是 NetBSD 系统
#define BPF_ALIGNMENT sizeof(bpf_int32)  # 定义 BPF_ALIGNMENT 为 bpf_int32 类型的大小
#else  # 如果是 NetBSD 系统
#define BPF_ALIGNMENT sizeof(long)  # 定义 BPF_ALIGNMENT 为 long 类型的大小
#endif
#define BPF_WORDALIGN(x) (((x)+(BPF_ALIGNMENT-1))&~(BPF_ALIGNMENT-1))  # 定义 BPF_WORDALIGN 宏，将 x 向上舍入到下一个 BPF_ALIGNMENT 的倍数

/*
 * Structure for "pcap_compile()", "pcap_setfilter()", etc..
 */
struct bpf_program {  # 定义结构体 bpf_program
    u_int bf_len;  # 无符号整型 bf_len
    struct bpf_insn *bf_insns;  # 指向 bpf_insn 结构体的指针 bf_insns
};

/*
 * The instruction encodings.
 *
 * Please inform tcpdump-workers@lists.tcpdump.org if you use any
 * of the reserved values, so that we can note that they're used
 * (and perhaps implement it in the reference BPF implementation
 * and encourage its implementation elsewhere).
 */

/*
 * The upper 8 bits of the opcode aren't used. BSD/OS used 0x8000.
 */

/* instruction classes */
#define BPF_CLASS(code) ((code) & 0x07)  # 定义 BPF_CLASS 宏，获取指令类别
#define        BPF_LD        0x00  # 载入指令
#define        BPF_LDX        0x01  # 载入 X 寄存器指令
#define        BPF_ST        0x02  # 存储指令
#define        BPF_STX        0x03  # 存储 X 寄存器指令
#define        BPF_ALU        0x04  # 算术逻辑指令
#define        BPF_JMP        0x05  # 跳转指令
#define        BPF_RET        0x06  # 返回指令
#define        BPF_MISC    0x07  # 其他指令

/* ld/ldx fields */
#define BPF_SIZE(code)    ((code) & 0x18)  # 定义 BPF_SIZE 宏，获取载入大小
#define        BPF_W        0x00  # 字长载入
#define        BPF_H        0x08  # 半字长载入
#define        BPF_B        0x10  # 字节长载入
/*                0x18    reserved; used by BSD/OS */
#define BPF_MODE(code)    ((code) & 0xe0)  # 定义 BPF_MODE 宏，获取载入模式
#define        BPF_IMM    0x00  # 立即数模式
#define        BPF_ABS        0x20  # 绝对地址模式
# 定义 BPF 指令的一些常量
#define        BPF_IND        0x40
#define        BPF_MEM        0x60
#define        BPF_LEN        0x80
#define        BPF_MSH        0xa0
# 0xc0 保留，被 BSD/OS 使用
# 0xe0 保留，被 BSD/OS 使用

# alu/jmp 字段
# 通过 BPF 指令获取操作码
#define BPF_OP(code)    ((code) & 0xf0)
#define        BPF_ADD        0x00
#define        BPF_SUB        0x10
#define        BPF_MUL        0x20
#define        BPF_DIV        0x30
#define        BPF_OR        0x40
#define        BPF_AND        0x50
#define        BPF_LSH        0x60
#define        BPF_RSH        0x70
#define        BPF_NEG        0x80
#define        BPF_MOD        0x90
#define        BPF_XOR        0xa0
# 0xb0 保留
# 0xc0 保留
# 0xd0 保留
# 0xe0 保留
# 0xf0 保留

# 跳转指令
#define        BPF_JA        0x00
#define        BPF_JEQ        0x10
#define        BPF_JGT        0x20
#define        BPF_JGE        0x30
#define        BPF_JSET    0x40
# 0x50 保留，被 BSD/OS 使用
# 0x60 保留
# 0x70 保留
# 0x80 保留
# 0x90 保留
# 0xa0 保留
# 0xb0 保留
# 0xc0 保留
# 0xd0 保留
# 0xe0 保留
# 0xf0 保留
# 通过 BPF 指令获取源操作数
#define BPF_SRC(code)    ((code) & 0x08)
#define        BPF_K        0x00
#define        BPF_X        0x08

# 返回指令 - BPF_K 和 BPF_X 同样适用
# 通过 BPF 指令获取返回值
#define BPF_RVAL(code)    ((code) & 0x18)
#define        BPF_A        0x10
# 0x18 保留

# 其他
# 通过 BPF 指令获取杂项操作码
#define BPF_MISCOP(code) ((code) & 0xf8)
#define        BPF_TAX        0x00
# 0x08 保留
# 0x10 保留
# 0x18 保留
/* #define    BPF_COP        0x20    NetBSD "coprocessor" extensions */
/*                0x28    reserved */
/*                0x30    reserved */
/*                0x38    reserved */
/* #define    BPF_COPX    0x40    NetBSD "coprocessor" extensions */
/*                    also used on BSD/OS */
/*                0x48    reserved */
/*                0x50    reserved */
/*                0x58    reserved */
/*                0x60    reserved */
/*                0x68    reserved */
/*                0x70    reserved */
/*                0x78    reserved */
#define        BPF_TXA        0x80
/*                0x88    reserved */
/*                0x90    reserved */
/*                0x98    reserved */
/*                0xa0    reserved */
/*                0xa8    reserved */
/*                0xb0    reserved */
/*                0xb8    reserved */
/*                0xc0    reserved; used on BSD/OS */
/*                0xc8    reserved */
/*                0xd0    reserved */
/*                0xd8    reserved */
/*                0xe0    reserved */
/*                0xe8    reserved */
/*                0xf0    reserved */
/*                0xf8    reserved */

/*
 * The instruction data structure.
 */
struct bpf_insn {
    u_short    code;  // 16位无符号整数，表示BPF指令的操作码
    u_char    jt;    // 8位无符号整数，表示跳转的偏移量（true）
    u_char    jf;    // 8位无符号整数，表示跳转的偏移量（false）
    bpf_u_int32 k;  // 32位无符号整数，表示常数值
};

/*
 * Macros for insn array initializers.
 *
 * In case somebody's included <linux/filter.h>, or something else that
 * gives the kernel's definitions of BPF statements, get rid of its
 * definitions, so we can supply ours instead.  If some kernel's
 * definitions aren't *binary-compatible* with what BPF has had
 * since it first sprung from the brows of Van Jacobson and Steve
 * McCanne, that kernel should be fixed.
 */
#ifdef BPF_STMT
#undef BPF_STMT  // 取消之前定义的BPF_STMT宏
#endif
#define BPF_STMT(code, k) { (u_short)(code), 0, 0, k }  // 定义BPF_STMT宏，用于初始化BPF指令数组
#ifdef BPF_JUMP
#undef BPF_JUMP  // 取消之前定义的BPF_JUMP宏
#endif
#define BPF_JUMP(code, k, jt, jf) { (u_short)(code), jt, jf, k }  // 定义BPF_JUMP宏，用于初始化BPF跳转指令数组

PCAP_AVAILABLE_0_4
# 定义了一个名为bpf_filter的函数，接受BPF指令、数据包指针、数据包长度和数据包最大长度作为参数，返回一个无符号整数
PCAP_API u_int    bpf_filter(const struct bpf_insn *, const u_char *, u_int, u_int);

# 在0.6版本中可用，定义了一个名为bpf_validate的函数，接受BPF指令和长度作为参数，返回一个整数
PCAP_AVAILABLE_0_6
PCAP_API int    bpf_validate(const struct bpf_insn *f, int len);

# 在0.4版本中可用，定义了一个名为bpf_image的函数，接受BPF指令和长度作为参数，返回一个字符指针
PCAP_AVAILABLE_0_4
PCAP_API char    *bpf_image(const struct bpf_insn *, int);

# 在0.6版本中可用，定义了一个名为bpf_dump的函数，接受BPF程序和整数作为参数，无返回值
PCAP_AVAILABLE_0_6
PCAP_API void    bpf_dump(const struct bpf_program *, int);

/*
 * 用于BPF_LD|BPF_MEM和BPF_ST的临时内存单词数
 */
#define BPF_MEMWORDS 16

# 如果是C++代码，则结束extern "C"块
#ifdef __cplusplus
}
#endif

# 如果没有定义_NET_BPF_H_、_BPF_H_、_H_BPF或lib_pcap_bpf_h，则结束条件编译
#endif /* !defined(_NET_BPF_H_) && !defined(_BPF_H_) && !defined(_H_BPF) && !defined(lib_pcap_bpf_h) */
```