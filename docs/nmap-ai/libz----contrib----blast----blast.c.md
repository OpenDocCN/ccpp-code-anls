# `nmap\libz\contrib\blast\blast.c`

```
/* blast.c
 * 版权所有 (C) 2003, 2012, 2013 Mark Adler
 * 发行和使用条件，请参见 blast.h 中的版权声明
 * 版本 1.3, 2013年8月24日
 *
 * blast.c 用于解压由 PKWare 压缩库压缩的数据。
 * 此函数提供类似于 PKWare 库的 explode() 函数的功能，因此命名为 "blast"。
 *
 * 此解压缩器基于 Ben Rudiak-Gould 在 2001年8月13日在 comp.compression 上提供的优秀格式描述。
 * 有趣的是，Ben 在帖子中提供的示例是不正确的。距离 110001 应该是 111000。当更正后，示例字节流变为：
 *
 *    00 04 82 24 25 8f 80 7f
 *
 * 它解压缩为 "AIAIAIAIAIAIA"（不带引号）。
 */

/*
 * 变更历史:
 *
 * 1.0  2003年2月12日     - 第一个版本
 * 1.1  2003年2月16日     - 修复了对 > 4GB 未压缩数据的距离检查
 * 1.2  2012年10月24日    - 添加有关在 stdio 中使用二进制模式的说明
 *                        - 修复了不同符号整数的比较
 * 1.3  2013年8月24日     - 从 blast() 返回未使用的输入
 *                        - 修复测试代码以正确报告未使用的输入
 *                        - 启用向 blast() 提供初始输入的功能
 */

#include <stddef.h>             /* 用于 NULL */
#include <setjmp.h>             /* 用于 setjmp()、longjmp() 和 jmp_buf */
#include "blast.h"              /* blast() 的原型 */

#define local static            /* 用于本地函数定义 */
#define MAXBITS 13              /* 最大代码长度 */
#define MAXWIN 4096             /* 最大窗口大小 */

/* 输入和输出状态 */
struct state {
    /* 输入状态 */
    blast_in infun;             /* 用户提供的输入函数 */
    void *inhow;                /* 传递给 infun() 的不透明信息 */
    unsigned char *in;          /* 下一个输入位置 */
    unsigned left;              /* 在 in 中可用的输入 */
    # 位缓冲区
    int bitbuf;                 /* bit buffer */
    # 位缓冲区中的位数
    int bitcnt;                 /* number of bits in bit buffer */

    # 用于 bits() 和 decode() 函数的输入限制错误返回状态
    jmp_buf env;                /* input limit error return state for bits() and decode() */

    # 输出状态
    blast_out outfun;           /* output function provided by user */
    # 传递给 outfun() 函数的不透明信息
    void *outhow;               /* opaque information passed to outfun() */
    # out[] 数组中下一个写入位置的索引
    unsigned next;              /* index of next write location in out[] */
    # 用于检查距离的标志（对于前4K）
    int first;                  /* true to check distances (for first 4K) */
    # 输出缓冲区和滑动窗口
    unsigned char out[MAXWIN];  /* output buffer and sliding window */
/*
 * 从输入流中返回所需的位数。这总是会在缓冲区中留下少于八位。
 * 当 need == 0 时，bits() 函数能正常工作。
 *
 * 格式说明：
 *
 * - 位被存储在字节中，从最低有效位到最高有效位。因此，位从位缓冲区的底部被丢弃，使用右移，新的字节被追加到位缓冲区的顶部，使用左移。
 */
local int bits(struct state *s, int need)
{
    int val;            /* 位累加器 */

    /* 将至少 need 位加载到 val 中 */
    val = s->bitbuf;
    while (s->bitcnt < need) {
        if (s->left == 0) {
            s->left = s->infun(s->inhow, &(s->in));
            if (s->left == 0) longjmp(s->env, 1);       /* 输入流耗尽 */
        }
        val |= (int)(*(s->in)++) << s->bitcnt;          /* 加载八位 */
        s->left--;
        s->bitcnt += 8;
    }

    /* 丢弃所需的位并更新缓冲区，始终保留零到七位 */
    s->bitbuf = val >> need;
    s->bitcnt -= need;

    /* 返回所需的位，将高于所需位的位清零 */
    return val & ((1 << need) - 1);
}

/*
 * 哈夫曼编码解码表。count[1..MAXBITS] 是每个长度的符号数，在规范编码中按顺序递增。
 * symbol[] 是按规范顺序的符号值，条目数是 count[] 中所有计数的总和。解码过程可以在下面的 decode() 函数中看到。
 */
struct huffman {
    short *count;       /* 每个长度的符号数 */
    short *symbol;      /* 按规范顺序的符号 */
};
/*
 * 使用霍夫曼表 h 从流 s 中解码一个代码。如果出现错误，则返回符号或负值。如果所有长度都为零，即空代码，或者代码不完整并收到无效代码，则在读取 MAXBITS 位后返回 -9。
 *
 * 格式说明：
 *
 * - 在压缩数据中，代码与相同长度的简单整数顺序相对于位反转。因此，在下面，从压缩数据中逐位提取位，并使用这些位构建代码值，以便与流中的相反顺序进行简单的整数比较以进行解码。
 *
 * - 最短长度的第一个代码是全为1。相同长度的后续代码只是前一个代码的整数递减。当移动到更长的长度时，会在代码后附加一个一位。对于完整的代码，最长长度的最后一个代码将全为零。为了支持这种顺序，解码过程中提取的位被反转，以应用更“自然”的顺序，从全零开始递增。
 */
local int decode(struct state *s, struct huffman *h)
{
    int len;            /* 当前代码中的位数 */
    int code;           /* 被解码的长度位 */
    int first;          /* 长度为 len 的第一个代码 */
    int count;          /* 长度为 len 的代码数量 */
    int index;          /* 符号表中长度为 len 的第一个代码的索引 */
    int bitbuf;         /* 来自流的位 */
    int left;           /* 下一个或待处理的剩余位数 */
    short *next;        /* 下一个代码的数量 */

    bitbuf = s->bitbuf;
    left = s->bitcnt;
    code = first = index = 0;
    len = 1;
    next = h->count + 1;
    while (1) {
        // 无限循环，直到条件被打破
        while (left--) {
            // 当 left 不为 0 时，执行循环
            code |= (bitbuf & 1) ^ 1;   /* invert code */
            // 将 bitbuf 的最低位与 1 进行异或操作，然后取反，将结果赋值给 code
            bitbuf >>= 1;
            // 将 bitbuf 右移一位
            count = *next++;
            // 将 next 指向的值赋给 count，然后 next 指针向后移动一位
            if (code < first + count) { /* if length len, return symbol */
                // 如果 code 小于 first + count，则返回对应的符号
                s->bitbuf = bitbuf;
                // 将 bitbuf 的值赋给 s->bitbuf
                s->bitcnt = (s->bitcnt - len) & 7;
                // 更新 s->bitcnt 的值
                return h->symbol[index + (code - first)];
                // 返回对应的符号
            }
            index += count;             /* else update for next length */
            // 更新 index 的值
            first += count;
            // 更新 first 的值
            first <<= 1;
            // 将 first 左移一位
            code <<= 1;
            // 将 code 左移一位
            len++;
            // len 自增
        }
        left = (MAXBITS+1) - len;
        // 计算 left 的值
        if (left == 0) break;
        // 如果 left 为 0，则跳出循环
        if (s->left == 0) {
            // 如果 s->left 为 0
            s->left = s->infun(s->inhow, &(s->in));
            // 调用 s->infun 函数，更新 s->left 的值
            if (s->left == 0) longjmp(s->env, 1);       /* out of input */
            // 如果 s->left 为 0，则跳转到 longjmp 函数
        }
        bitbuf = *(s->in)++;
        // 将 s->in 指向的值赋给 bitbuf，然后 s->in 指针向后移动一位
        s->left--;
        // s->left 自减
        if (left > 8) left = 8;
        // 如果 left 大于 8，则将 left 的值设为 8
    }
    return -9;                          /* ran out of codes */
    // 循环结束后返回 -9
}
/*
 * 给定一个重复代码长度列表 rep[0..n-1]，其中每个字节都是一个计数（高四位+1）和一个代码长度（低四位），生成代码长度列表。
 * 这种压缩可以减小对象代码的大小。然后，给定表示 n 个符号的规范哈夫曼编码的代码长度列表 length[0..n-1]，构造解码这些代码所需的表。
 * 这些表是每个长度的代码数量，以及按长度排序的符号，保留它们在每个长度内的原始顺序。返回值为零表示完整的代码集，负数表示超额的代码集，正数表示不完整的代码集。
 * 如果返回值为零或正数，则可以使用这些表，但如果返回值为负数，则不能使用这些表。
 * 如果返回值为零，则使用该表的 decode() 不可能返回错误--任何足够的位流都将解析为一个符号。
 * 如果返回值为正数，则使用该表的 decode() 可能对接收到的超出不完整长度的代码返回错误。
 */
local int construct(struct huffman *h, const unsigned char *rep, int n)
{
    int symbol;         /* 当遍历 length[] 时的当前符号 */
    int len;            /* 当遍历 h->count[] 时的当前长度 */
    int left;           /* 当前长度左侧可能的代码数量 */
    short offs[MAXBITS+1];      /* 每个长度的符号表中的偏移量 */
    short length[256];  /* 代码长度 */

    /* 将紧凑的重复计数转换为符号位长度列表 */
    symbol = 0;
    do {
        len = *rep++;
        left = (len >> 4) + 1;
        len &= 15;
        do {
            length[symbol++] = len;
        } while (--left);
    } while (--n);
    n = symbol;

    /* 计算每个长度的代码数量 */
    for (len = 0; len <= MAXBITS; len++)
        h->count[len] = 0;
    for (symbol = 0; symbol < n; symbol++)
        (h->count[length[symbol]])++;   /* 遍历长度数组，统计每个长度出现的次数 */
    if (h->count[0] == n)               /* 如果长度为0的次数等于总数n，表示没有编码 */
        return 0;                       /* 完整，但解码会失败 */

    /* 检查长度集合是否超额订阅或不完整 */
    left = 1;                           /* 零长度可能的编码 */
    for (len = 1; len <= MAXBITS; len++) {
        left <<= 1;                     /* 多一个比特位，编码数量翻倍 */
        left -= h->count[len];          /* 从可能的编码数量中减去实际出现的数量 */
        if (left < 0) return left;      /* 超额订阅--返回负数 */
    }                                   /* left > 0 表示不完整 */

    /* 为每个长度生成符号表的偏移量以进行排序 */
    offs[1] = 0;
    for (len = 1; len < MAXBITS; len++)
        offs[len + 1] = offs[len] + h->count[len];

    /*
     * 按长度和符号顺序将符号放入表中进行排序
     */
    for (symbol = 0; symbol < n; symbol++)
        if (length[symbol] != 0)
            h->symbol[offs[length[symbol]]++] = symbol;

    /* 完整集合返回零，不完整集合返回正数 */
    return left;
# 解码 PKWare 压缩库流的函数
# 格式说明：
# - 第一个字节为0表示文字未编码，为1表示文字已编码。第二个字节为4、5或6，表示距离代码中额外位数的数量。这是字典大小减去六的以2为底的对数。
# - 压缩数据是文字和长度/距离对的组合，以结束代码终止。文字可以是Huffman编码或未编码的字节。长度/距离对是一个编码长度，后跟一个编码距离，表示在未压缩数据中出现在当前位置再次出现的字符串。
# - 位在文字或长度/距离对之前指示接下来的内容，0表示文字，1表示长度/距离。
# - 如果文字未编码，则接下来的八位是文字，按照流中的正常位顺序，即不需要位反转。类似地，长度额外位或距离额外位也不需要位反转。
# - 文字字节直接写入输出。长度/距离对是一个指令，将先前未压缩的字节复制到输出。复制是从输出流中的距禿字节开始，复制长度字节。
# - 不允许指向输出数据开始之前的距离。
# - 允许和常见的重叠复制，其中长度大于距离。例如，距离为1，长度为518，只是将最后一个字节复制518次。距禿为4，长度为12，将最后四个字节复制三次。一个简单的前向复制，忽略长度是否大于距禿，可以正确实现这一点。
本地变量，表示文字是否编码
    int lit;
本地变量，表示log2(字典大小) - 6
    int dict;
    int symbol;         /* 解码的符号，距离的额外位 */
    int len;            /* 复制的长度 */
    unsigned dist;      /* 复制的距离 */
    int copy;           /* 复制计数器 */
    unsigned char *from, *to;   /* 复制指针 */
    static int virgin = 1;                              /* 仅构建一次表 */
    static short litcnt[MAXBITS+1], litsym[256];        /* 字面码内存 */
    static short lencnt[MAXBITS+1], lensym[16];         /* 长度码内存 */
    static short distcnt[MAXBITS+1], distsym[64];       /* 距离码内存 */
    static struct huffman litcode = {litcnt, litsym};   /* 长度码 */
    static struct huffman lencode = {lencnt, lensym};   /* 长度码 */
    static struct huffman distcode = {distcnt, distsym};/* 距离码 */
        /* 字面码的位长度 */
    static const unsigned char litlen[] = {
        11, 124, 8, 7, 28, 7, 188, 13, 76, 4, 10, 8, 12, 10, 12, 10, 8, 23, 8,
        9, 7, 6, 7, 8, 7, 6, 55, 8, 23, 24, 12, 11, 7, 9, 11, 12, 6, 7, 22, 5,
        7, 24, 6, 11, 9, 6, 7, 22, 7, 11, 38, 7, 9, 8, 25, 11, 8, 11, 9, 12,
        8, 12, 5, 38, 5, 38, 5, 11, 7, 5, 6, 21, 6, 10, 53, 8, 7, 24, 10, 27,
        44, 253, 253, 253, 252, 252, 252, 13, 12, 45, 12, 45, 12, 61, 12, 45,
        44, 173};
        /* 长度码 0..15 的位长度 */
    static const unsigned char lenlen[] = {2, 35, 36, 53, 38, 23};
        /* 距离码 0..63 的位长度 */
    static const unsigned char distlen[] = {2, 20, 53, 230, 247, 151, 248};
    static const short base[16] = {     /* 长度码的基数 */
        3, 2, 4, 5, 6, 7, 8, 9, 10, 12, 16, 24, 40, 72, 136, 264};
    static const char extra[16] = {     /* 长度码的额外位 */
        0, 0, 0, 0, 0, 0, 0, 0, 1, 2, 3, 4, 5, 6, 7, 8};

    /* 设置解码表（一次性操作，可能不是线程安全的） */
    # 如果是第一次执行，则构造litcode、lencode、distcode三个数据结构
    if (virgin) {
        construct(&litcode, litlen, sizeof(litlen));
        construct(&lencode, lenlen, sizeof(lenlen));
        construct(&distcode, distlen, sizeof(distlen));
        virgin = 0;
    }

    # 读取头部信息
    # 读取8位bit作为lit
    lit = bits(s, 8);
    # 如果lit大于1，则返回-1
    if (lit > 1) return -1;
    # 读取8位bit作为dict
    dict = bits(s, 8);
    # 如果dict小于4或大于6，则返回-2
    if (dict < 4 || dict > 6) return -2;

    # 解码字面量和长度/距离对
    do {
        // 如果当前位为1
        if (bits(s, 1)) {
            /* get length */
            // 解码获取长度
            symbol = decode(s, &lencode);
            len = base[symbol] + bits(s, extra[symbol]);
            // 如果长度为519，则结束
            if (len == 519) break;              /* end code */

            /* get distance */
            // 如果长度为2，则设置symbol为2，否则为dict
            symbol = len == 2 ? 2 : dict;
            dist = decode(s, &distcode) << symbol;
            dist += bits(s, symbol);
            dist++;
            // 如果是第一次并且距离太远，则返回错误
            if (s->first && dist > s->next)
                return -3;              /* distance too far back */

            /* copy length bytes from distance bytes back */
            // 从距离处复制长度字节到当前位置
            do {
                to = s->out + s->next;
                from = to - dist;
                copy = MAXWIN;
                if (s->next < dist) {
                    from += copy;
                    copy = dist;
                }
                copy -= s->next;
                if (copy > len) copy = len;
                len -= copy;
                s->next += copy;
                do {
                    *to++ = *from++;
                } while (--copy);
                // 如果当前位置达到最大窗口大小
                if (s->next == MAXWIN) {
                    if (s->outfun(s->outhow, s->out, s->next)) return 1;
                    s->next = 0;
                    s->first = 0;
                }
            } while (len != 0);
        }
        else {
            /* get literal and write it */
            // 获取字面值并写入
            symbol = lit ? decode(s, &litcode) : bits(s, 8);
            s->out[s->next++] = symbol;
            // 如果当前位置达到最大窗口大小
            if (s->next == MAXWIN) {
                if (s->outfun(s->outhow, s->out, s->next)) return 1;
                s->next = 0;
                s->first = 0;
            }
        }
    } while (1);
    return 0;
/* See comments in blast.h */
/* 根据 blast.h 文件中的注释进行解释 */

int blast(blast_in infun, void *inhow, blast_out outfun, void *outhow,
          unsigned *left, unsigned char **in)
{
    /* 定义输入/输出状态结构体 */
    struct state s;
    /* 返回值 */
    int err;

    /* 初始化输入状态 */
    s.infun = infun;
    s.inhow = inhow;
    if (left != NULL && *left) {
        s.left = *left;
        s.in = *in;
    }
    else
        s.left = 0;
    s.bitbuf = 0;
    s.bitcnt = 0;

    /* 初始化输出状态 */
    s.outfun = outfun;
    s.outhow = outhow;
    s.next = 0;
    s.first = 1;

    /* 如果 bits() 或 decode() 尝试读取超出可用输入，则返回 */
    if (setjmp(s.env) != 0)             /* 如果通过 longjmp() 回到这里，则跳过 decomp()，返回错误 */
        err = 2;
    else
        err = decomp(&s);               /* 解压缩 */

    /* 返回未使用的输入 */
    if (left != NULL)
        *left = s.left;
    if (in != NULL)
        *in = s.left ? s.in : NULL;

    /* 写入任何剩余的输出并根据需要更新错误代码 */
    if (err != 1 && s.next && s.outfun(s.outhow, s.out, s.next) && err == 0)
        err = 1;
    return err;
}

#ifdef TEST
/* Example of how to use blast() */
/* 使用 blast() 的示例 */

#include <stdio.h>
#include <stdlib.h>

#define CHUNK 16384

local unsigned inf(void *how, unsigned char **buf)
{
    static unsigned char hold[CHUNK];

    *buf = hold;
    return fread(hold, 1, CHUNK, (FILE *)how);
}

local int outf(void *how, unsigned char *buf, unsigned len)
{
    return fwrite(buf, 1, len, (FILE *)how) != len;
}

/* Decompress a PKWare Compression Library stream from stdin to stdout */
/* 从 stdin 解压缩 PKWare 压缩库流到 stdout */
int main(void)
{
    int ret;
    unsigned left;

    /* 解压缩到 stdout */
    left = 0;
    ret = blast(inf, stdin, outf, stdout, &left, NULL);
    if (ret != 0)
        fprintf(stderr, "blast error: %d\n", ret);

    /* 计算任何剩余的字节 */
    while (getchar() != EOF)
        left++;
}
    # 如果 left 不为 0，则向标准错误输出未使用的输入字节数的警告信息
    if (left)
        fprintf(stderr, "blast warning: %u unused bytes of input\n", left);

    # 返回 blast() 函数的错误码
    return ret;
}
# 结束条件编译指令的代码块
#endif
# 结束条件编译指令
```