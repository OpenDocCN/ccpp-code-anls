# `nmap\libpcap\bpf_image.c`

```
/*
 * 版权声明，版权归加利福尼亚大学所有
 * 允许在源代码和二进制形式下进行再发布和使用，无论是否进行修改
 * 1. 源代码发布必须保留以上版权声明和本段完整内容
 * 2. 包含二进制代码的发布必须在文档或其他提供的材料中包含以上版权声明和本段完整内容
 * 3. 所有提及此软件特性或使用的广告材料必须显示以下声明：
 *    “本产品包含由加利福尼亚大学洛伦斯伯克利实验室及其贡献者开发的软件。”
 *    未经事先书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品
 * 本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <pcap-types.h>

#include <stdio.h>
#include <string.h>

#ifdef __linux__
#include <linux/types.h>
#include <linux/if_packet.h>
#include <linux/filter.h>

/*
 * 我们希望使用我们的版本的这些 #define，而不是 Linux 的版本
 * （两者应该是相同的；如果不是，就有问题；所有 BPF 实现应该是我们的源代码兼容的超集）
 */
#undef BPF_STMT
#undef BPF_JUMP
#endif

#include "pcap-int.h"

#ifdef HAVE_OS_PROTO_H
#include "os-proto.h"
#endif

#ifdef SKF_AD_OFF
/*
 * 指向特殊的 Linux BPF 位置的偏移的符号名称
 */
static const char *offsets[SKF_AD_MAX] = {
#ifdef SKF_AD_PROTOCOL
    [SKF_AD_PROTOCOL] = "proto",
#endif
#ifdef SKF_AD_PKTTYPE
    [SKF_AD_PKTTYPE] = "type",
#endif
#ifdef SKF_AD_IFINDEX
    [SKF_AD_IFINDEX] = "ifidx",  // 如果定义了 SKF_AD_IFINDEX，则将其映射为字符串"ifidx"
#endif
#ifdef SKF_AD_NLATTR
    [SKF_AD_NLATTR] = "nla",  // 如果定义了 SKF_AD_NLATTR，则将其映射为字符串"nla"
#endif
#ifdef SKF_AD_NLATTR_NEST
    [SKF_AD_NLATTR_NEST] = "nlan",  // 如果定义了 SKF_AD_NLATTR_NEST，则将其映射为字符串"nlan"
#endif
#ifdef SKF_AD_MARK
    [SKF_AD_MARK] = "mark",  // 如果定义了 SKF_AD_MARK，则将其映射为字符串"mark"
#endif
#ifdef SKF_AD_QUEUE
    [SKF_AD_QUEUE] = "queue",  // 如果定义了 SKF_AD_QUEUE，则将其映射为字符串"queue"
#endif
#ifdef SKF_AD_HATYPE
    [SKF_AD_HATYPE] = "hatype",  // 如果定义了 SKF_AD_HATYPE，则将其映射为字符串"hatype"
#endif
#ifdef SKF_AD_RXHASH
    [SKF_AD_RXHASH] = "rxhash",  // 如果定义了 SKF_AD_RXHASH，则将其映射为字符串"rxhash"
#endif
#ifdef SKF_AD_CPU
    [SKF_AD_CPU] = "cpu",  // 如果定义了 SKF_AD_CPU，则将其映射为字符串"cpu"
#endif
#ifdef SKF_AD_ALU_XOR_X
    [SKF_AD_ALU_XOR_X] = "xor_x",  // 如果定义了 SKF_AD_ALU_XOR_X，则将其映射为字符串"xor_x"
#endif
#ifdef SKF_AD_VLAN_TAG
    [SKF_AD_VLAN_TAG] = "vlan_tci",  // 如果定义了 SKF_AD_VLAN_TAG，则将其映射为字符串"vlan_tci"
#endif
#ifdef SKF_AD_VLAN_TAG_PRESENT
    [SKF_AD_VLAN_TAG_PRESENT] = "vlanp",  // 如果定义了 SKF_AD_VLAN_TAG_PRESENT，则将其映射为字符串"vlanp"
#endif
#ifdef SKF_AD_PAY_OFFSET
    [SKF_AD_PAY_OFFSET] = "poff",  // 如果定义了 SKF_AD_PAY_OFFSET，则将其映射为字符串"poff"
#endif
#ifdef SKF_AD_RANDOM
    [SKF_AD_RANDOM] = "random",  // 如果定义了 SKF_AD_RANDOM，则将其映射为字符串"random"
#endif
#ifdef SKF_AD_VLAN_TPID
    [SKF_AD_VLAN_TPID] = "vlan_tpid"  // 如果定义了 SKF_AD_VLAN_TPID，则将其映射为字符串"vlan_tpid"
#endif
};
#endif

static void
bpf_print_abs_load_operand(char *buf, size_t bufsize, const struct bpf_insn *p)
{
#ifdef SKF_AD_OFF
    const char *sym;

    /*
     * It's an absolute load.
     * Is the offset a special Linux offset that we know about?
     */
    if (p->k >= (bpf_u_int32)SKF_AD_OFF &&
        p->k < (bpf_u_int32)(SKF_AD_OFF + SKF_AD_MAX) &&
        (sym = offsets[p->k - (bpf_u_int32)SKF_AD_OFF]) != NULL) {
        /*
         * Yes.  Print the offset symbolically.
         */
        (void)snprintf(buf, bufsize, "[%s]", sym);  // 如果偏移量是特殊的 Linux 偏移量，则以符号形式打印偏移量
    } else
#endif
        (void)snprintf(buf, bufsize, "[%d]", p->k);  // 否则以数字形式打印偏移量
}

char *
bpf_image(const struct bpf_insn *p, int n)
{
    const char *op;
    static char image[256];
    char operand_buf[64];
    const char *operand;

    switch (p->code) {

    default:
        op = "unimp";
        (void)snprintf(operand_buf, sizeof operand_buf, "0x%x", p->code);  // 默认情况下，操作码为"unimp"，操作数为十六进制形式的操作码
        operand = operand_buf;
        break;

    case BPF_RET|BPF_K:
        op = "ret";
        (void)snprintf(operand_buf, sizeof operand_buf, "#%d", p->k);  // 如果操作码为 BPF_RET|BPF_K，则操作为"ret"，操作数为十进制形式的 k 值
        operand = operand_buf;
        break;
    # 如果操作码是返回指令，设置操作符为"ret"，操作数为空
    case BPF_RET|BPF_A:
        op = "ret";
        operand = "";
        break;

    # 如果操作码是从绝对地址加载字（4字节），设置操作符为"ld"，调用函数获取操作数
    case BPF_LD|BPF_W|BPF_ABS:
        op = "ld";
        bpf_print_abs_load_operand(operand_buf, sizeof operand_buf, p);
        operand = operand_buf;
        break;

    # 如果操作码是从绝对地址加载半字（2字节），设置操作符为"ldh"，调用函数获取操作数
    case BPF_LD|BPF_H|BPF_ABS:
        op = "ldh";
        bpf_print_abs_load_operand(operand_buf, sizeof operand_buf, p);
        operand = operand_buf;
        break;

    # 如果操作码是从绝对地址加载字节（1字节），设置操作符为"ldb"，调用函数获取操作数
    case BPF_LD|BPF_B|BPF_ABS:
        op = "ldb";
        bpf_print_abs_load_operand(operand_buf, sizeof operand_buf, p);
        operand = operand_buf;
        break;

    # 如果操作码是加载数据包长度，设置操作符为"ld"，操作数为"#pktlen"
    case BPF_LD|BPF_W|BPF_LEN:
        op = "ld";
        operand = "#pktlen";
        break;

    # 如果操作码是从索引地址加载字（4字节），设置操作符为"ld"，构造操作数字符串
    case BPF_LD|BPF_W|BPF_IND:
        op = "ld";
        (void)snprintf(operand_buf, sizeof operand_buf, "[x + %d]", p->k);
        operand = operand_buf;
        break;

    # 如果操作码是从索引地址加载半字（2字节），设置操作符为"ldh"，构造操作数字符串
    case BPF_LD|BPF_H|BPF_IND:
        op = "ldh";
        (void)snprintf(operand_buf, sizeof operand_buf, "[x + %d]", p->k);
        operand = operand_buf;
        break;

    # 如果操作码是从索引地址加载字节（1字节），设置操作符为"ldb"，构造操作数字符串
    case BPF_LD|BPF_B|BPF_IND:
        op = "ldb";
        (void)snprintf(operand_buf, sizeof operand_buf, "[x + %d]", p->k);
        operand = operand_buf;
        break;

    # 如果操作码是加载立即数，设置操作符为"ld"，构造操作数字符串
    case BPF_LD|BPF_IMM:
        op = "ld";
        (void)snprintf(operand_buf, sizeof operand_buf, "#0x%x", p->k);
        operand = operand_buf;
        break;

    # 如果操作码是加载立即数到X寄存器，设置操作符为"ldx"，构造操作数字符串
    case BPF_LDX|BPF_IMM:
        op = "ldx";
        (void)snprintf(operand_buf, sizeof operand_buf, "#0x%x", p->k);
        operand = operand_buf;
        break;

    # 如果操作码是加载立即数的高4位到X寄存器，设置操作符为"ldxb"，构造操作数字符串
    case BPF_LDX|BPF_MSH|BPF_B:
        op = "ldxb";
        (void)snprintf(operand_buf, sizeof operand_buf, "4*([%d]&0xf)", p->k);
        operand = operand_buf;
        break;

    # 如果操作码是从内存加载数据，设置操作符为"ld"，构造操作数字符串
    case BPF_LD|BPF_MEM:
        op = "ld";
        (void)snprintf(operand_buf, sizeof operand_buf, "M[%d]", p->k);
        operand = operand_buf;
        break;
    # 如果操作码是加载指令并且操作数是内存，设置操作符为"ldx"
    case BPF_LDX|BPF_MEM:
        op = "ldx";
        # 格式化操作数缓冲区，将"M[操作数]"写入缓冲区
        (void)snprintf(operand_buf, sizeof operand_buf, "M[%d]", p->k);
        operand = operand_buf;
        break;

    # 如果操作码是存储指令，设置操作符为"st"
    case BPF_ST:
        op = "st";
        # 格式化操作数缓冲区，将"M[操作数]"写入缓冲区
        (void)snprintf(operand_buf, sizeof operand_buf, "M[%d]", p->k);
        operand = operand_buf;
        break;

    # 如果操作码是存储指令并且操作数是内存，设置操作符为"stx"
    case BPF_STX:
        op = "stx";
        # 格式化操作数缓冲区，将"M[操作数]"写入缓冲区
        (void)snprintf(operand_buf, sizeof operand_buf, "M[%d]", p->k);
        operand = operand_buf;
        break;

    # 如果操作码是跳转指令并且跳转类型是绝对跳转，设置操作符为"ja"
    case BPF_JMP|BPF_JA:
        op = "ja";
        # 格式化操作数缓冲区，将"n + 1 + 操作数"写入缓冲区
        (void)snprintf(operand_buf, sizeof operand_buf, "%d", n + 1 + p->k);
        operand = operand_buf;
        break;

    # 如果操作码是跳转指令并且跳转类型是大于并且操作数是立即数，设置操作符为"jgt"
    case BPF_JMP|BPF_JGT|BPF_K:
        op = "jgt";
        # 格式化操作数缓冲区，将"#0x操作数"写入缓冲区
        (void)snprintf(operand_buf, sizeof operand_buf, "#0x%x", p->k);
        operand = operand_buf;
        break;

    # 如果操作码是跳转指令并且跳转类型是大于等于并且操作数是立即数，设置操作符为"jge"
    case BPF_JMP|BPF_JGE|BPF_K:
        op = "jge";
        # 格式化操作数缓冲区，将"#0x操作数"写入缓冲区
        (void)snprintf(operand_buf, sizeof operand_buf, "#0x%x", p->k);
        operand = operand_buf;
        break;

    # 如果操作码是跳转指令并且跳转类型是等于并且操作数是立即数，设置操作符为"jeq"
    case BPF_JMP|BPF_JEQ|BPF_K:
        op = "jeq";
        # 格式化操作数缓冲区，将"#0x操作数"写入缓冲区
        (void)snprintf(operand_buf, sizeof operand_buf, "#0x%x", p->k);
        operand = operand_buf;
        break;

    # 如果操作码是跳转指令并且跳转类型是设置并且操作数是立即数，设置操作符为"jset"
    case BPF_JMP|BPF_JSET|BPF_K:
        op = "jset";
        # 格式化操作数缓冲区，将"#0x操作数"写入缓冲区
        (void)snprintf(operand_buf, sizeof operand_buf, "#0x%x", p->k);
        operand = operand_buf;
        break;

    # 如果操作码是跳转指令并且跳转类型是大于并且操作数是寄存器，设置操作符为"jgt"
    case BPF_JMP|BPF_JGT|BPF_X:
        op = "jgt";
        # 设置操作数为"x"
        operand = "x";
        break;

    # 如果操作码是跳转指令并且跳转类型是大于等于并且操作数是寄存器，设置操作符为"jge"
    case BPF_JMP|BPF_JGE|BPF_X:
        op = "jge";
        # 设置操作数为"x"
        operand = "x";
        break;

    # 如果操作码是跳转指令并且跳转类型是等于并且操作数是寄存器，设置操作符为"jeq"
    case BPF_JMP|BPF_JEQ|BPF_X:
        op = "jeq";
        # 设置操作数为"x"
        operand = "x";
        break;

    # 如果操作码是跳转指令并且跳转类型是设置并且操作数是寄存器，设置操作符为"jset"
    case BPF_JMP|BPF_JSET|BPF_X:
        op = "jset";
        # 设置操作数为"x"
        operand = "x";
        break;

    # 如果操作码是算术逻辑指令并且操作类型是加法并且操作数是寄存器，设置操作符为"add"
    case BPF_ALU|BPF_ADD|BPF_X:
        op = "add";
        # 设置操作数为"x"
        operand = "x";
        break;

    # 如果操作码是算术逻辑指令并且操作类型是减法并且操作数是寄存器，设置操作符为"sub"
    case BPF_ALU|BPF_SUB|BPF_X:
        op = "sub";
        # 设置操作数为"x"
        operand = "x";
        break;

    # 如果操作码是算术逻辑指令并且操作类型是乘法并且操作数是寄存器，设置操作符为"mul"
    case BPF_ALU|BPF_MUL|BPF_X:
        op = "mul";
        # 设置操作数为"x"
        operand = "x";
        break;
    # 如果操作码是 ALU、DIV、X 的组合，则操作为除法，操作数为 x
    case BPF_ALU|BPF_DIV|BPF_X:
        op = "div";
        operand = "x";
        break;

    # 如果操作码是 ALU、MOD、X 的组合，则操作为取模，操作数为 x
    case BPF_ALU|BPF_MOD|BPF_X:
        op = "mod";
        operand = "x";
        break;

    # 如果操作码是 ALU、AND、X 的组合，则操作为按位与，操作数为 x
    case BPF_ALU|BPF_AND|BPF_X:
        op = "and";
        operand = "x";
        break;

    # 如果操作码是 ALU、OR、X 的组合，则操作为按位或，操作数为 x
    case BPF_ALU|BPF_OR|BPF_X:
        op = "or";
        operand = "x";
        break;

    # 如果操作码是 ALU、XOR、X 的组合，则操作为按位异或，操作数为 x
    case BPF_ALU|BPF_XOR|BPF_X:
        op = "xor";
        operand = "x";
        break;

    # 如果操作码是 ALU、LSH、X 的组合，则操作为逻辑左移，操作数为 x
    case BPF_ALU|BPF_LSH|BPF_X:
        op = "lsh";
        operand = "x";
        break;

    # 如果操作码是 ALU、RSH、X 的组合，则操作为逻辑右移，操作数为 x
    case BPF_ALU|BPF_RSH|BPF_X:
        op = "rsh";
        operand = "x";
        break;

    # 如果操作码是 ALU、ADD、K 的组合，则操作为加法，操作数为常数 k
    case BPF_ALU|BPF_ADD|BPF_K:
        op = "add";
        (void)snprintf(operand_buf, sizeof operand_buf, "#%d", p->k);
        operand = operand_buf;
        break;

    # 如果操作码是 ALU、SUB、K 的组合，则操作为减法，操作数为常数 k
    case BPF_ALU|BPF_SUB|BPF_K:
        op = "sub";
        (void)snprintf(operand_buf, sizeof operand_buf, "#%d", p->k);
        operand = operand_buf;
        break;

    # 如果操作码是 ALU、MUL、K 的组合，则操作为乘法，操作数为常数 k
    case BPF_ALU|BPF_MUL|BPF_K:
        op = "mul";
        (void)snprintf(operand_buf, sizeof operand_buf, "#%d", p->k);
        operand = operand_buf;
        break;

    # 如果操作码是 ALU、DIV、K 的组合，则操作为除法，操作数为常数 k
    case BPF_ALU|BPF_DIV|BPF_K:
        op = "div";
        (void)snprintf(operand_buf, sizeof operand_buf, "#%d", p->k);
        operand = operand_buf;
        break;

    # 如果操作码是 ALU、MOD、K 的组合，则操作为取模，操作数为常数 k
    case BPF_ALU|BPF_MOD|BPF_K:
        op = "mod";
        (void)snprintf(operand_buf, sizeof operand_buf, "#%d", p->k);
        operand = operand_buf;
        break;

    # 如果操作码是 ALU、AND、K 的组合，则操作为按位与，操作数为常数 k
    case BPF_ALU|BPF_AND|BPF_K:
        op = "and";
        (void)snprintf(operand_buf, sizeof operand_buf, "#0x%x", p->k);
        operand = operand_buf;
        break;

    # 如果操作码是 ALU、OR、K 的组合，则操作为按位或，操作数为常数 k
    case BPF_ALU|BPF_OR|BPF_K:
        op = "or";
        (void)snprintf(operand_buf, sizeof operand_buf, "#0x%x", p->k);
        operand = operand_buf;
        break;

    # 如果操作码是 ALU、XOR、K 的组合，则操作为按位异或，操作数为常数 k
    case BPF_ALU|BPF_XOR|BPF_K:
        op = "xor";
        (void)snprintf(operand_buf, sizeof operand_buf, "#0x%x", p->k);
        operand = operand_buf;
        break;
    # 如果操作码是 ALU 和 LSH 以及 K，则设置操作为 lsh
    case BPF_ALU|BPF_LSH|BPF_K:
        op = "lsh";
        # 将操作数的值格式化为字符串并存储在操作数缓冲区中
        (void)snprintf(operand_buf, sizeof operand_buf, "#%d", p->k);
        operand = operand_buf;
        break;

    # 如果操作码是 ALU 和 RSH 以及 K，则设置操作为 rsh
    case BPF_ALU|BPF_RSH|BPF_K:
        op = "rsh";
        # 将操作数的值格式化为字符串并存储在操作数缓冲区中
        (void)snprintf(operand_buf, sizeof operand_buf, "#%d", p->k);
        operand = operand_buf;
        break;

    # 如果操作码是 ALU 和 NEG，则设置操作为 neg
    case BPF_ALU|BPF_NEG:
        op = "neg";
        # 操作数为空字符串
        operand = "";
        break;

    # 如果操作码是 MISC 和 TAX，则设置操作为 tax
    case BPF_MISC|BPF_TAX:
        op = "tax";
        # 操作数为空字符串
        operand = "";
        break;

    # 如果操作码是 MISC 和 TXA，则设置操作为 txa
    case BPF_MISC|BPF_TXA:
        op = "txa";
        # 操作数为空字符串
        operand = "";
        break;
    }

    # 如果指令类别是跳转指令，并且操作码不是跳转绝对地址指令
    if (BPF_CLASS(p->code) == BPF_JMP && BPF_OP(p->code) != BPF_JA) {
        # 格式化输出指令信息，包括编号、操作、操作数、跳转真条件和跳转假条件
        (void)snprintf(image, sizeof image,
                  "(%03d) %-8s %-16s jt %d\tjf %d",
                  n, op, operand, n + 1 + p->jt, n + 1 + p->jf);
    } else {
        # 格式化输出指令信息，包括编号、操作、操作数
        (void)snprintf(image, sizeof image,
                  "(%03d) %-8s %s",
                  n, op, operand);
    }
    # 返回指令信息字符串
    return image;
# 闭合前面的函数定义
```