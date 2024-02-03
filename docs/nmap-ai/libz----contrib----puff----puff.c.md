# `nmap\libz\contrib\puff\puff.c`

```cpp
/*
 * puff.c
 * 版权所有 (C) 2002-2013 Mark Adler
 * 有关分发和使用条件，请参阅 puff.h 中的版权声明
 * 版本 2.3, 2013年1月21日
 *
 * puff.c 是一个简单的解压缩程序，旨在明确指定 deflate 格式。它并非为了速度而编写，而是为了简单性。作为一个副产品，当小型代码比速度更重要时，这段代码实际上可能是有用的，比如引导应用程序。对于典型的 deflate 数据，zlib 的 inflate() 大约比 puff() 快四倍。在我的机器上，zlib 的 inflate 编译为大约20K，而 puff.c 编译为大约4K（使用 GNU cc 的 PowerPC）。如果使用更快的 decode() 函数，则 puff() 只比 zlib 的 inflate() 慢两倍。
 *
 * 所有动态分配的内存都来自堆栈。所需的堆栈少于2K字节。这段代码与16位 int 兼容，并假定 long 至少为32位。puff.c 使用 short 数据类型（假定为16位）的数组，以节省内存。该代码适用于整数是大端或小端存储的情况。
 *
 * 在下面的注释中是“格式说明”，描述了解压缩过程并记录了格式的一些不太明显的方面。此源代码旨在补充正式描述 deflate 格式的 RFC 1951：
 *
 *    http://www.zlib.org/rfc-deflate.html
 */

#include <setjmp.h>             /* 用于 setjmp()、longjmp() 和 jmp_buf */
#include "puff.h"               /* puff() 的原型 */

#define local static            /* 用于本地函数定义 */

/*
 * 分配和循环的最大值。更改这些值并不有用--它们由 deflate 格式固定。
 */
#define MAXBITS 15              /* 代码中的最大位数 */
#define MAXLCODES 286           /* 字面/长度代码的最大数量 */
#define MAXDCODES 30            /* 距离代码的最大数量 */
/* 定义最大代码长度，用于读取 */
#define MAXCODES (MAXLCODES+MAXDCODES)  /* 最大代码长度 */
#define FIXLCODES 288           /* 固定的字面长度代码数量 */

/* 输入和输出状态 */
struct state {
    /* 输出状态 */
    unsigned char *out;         /* 输出缓冲区 */
    unsigned long outlen;       /* 输出缓冲区可用空间 */
    unsigned long outcnt;       /* 到目前为止写入输出缓冲区的字节数 */

    /* 输入状态 */
    const unsigned char *in;    /* 输入缓冲区 */
    unsigned long inlen;        /* 输入缓冲区可用空间 */
    unsigned long incnt;        /* 到目前为止读取的字节数 */
    int bitbuf;                 /* 位缓冲区 */
    int bitcnt;                 /* 位缓冲区中的位数 */

    /* 用于 bits() 和 decode() 的输入限制错误返回状态 */
    jmp_buf env;
};

/*
 * 从输入流中返回所需的位数。这总是在缓冲区中留下少于八位。当 need == 0 时，bits() 函数可以正常工作。
 *
 * 格式说明：
 *
 * - 位存储在字节中，从最低有效位到最高有效位。因此，从位缓冲区底部丢弃位，使用右移，从位缓冲区顶部附加新字节，使用左移。
 */
local int bits(struct state *s, int need)
{
    long val;           /* 位累加器（最多可使用 20 位） */

    /* 将至少 need 位加载到 val 中 */
    val = s->bitbuf;
    while (s->bitcnt < need) {
        if (s->incnt == s->inlen)
            longjmp(s->env, 1);         /* 输入不足 */
        val |= (long)(s->in[s->incnt++]) << s->bitcnt;  /* 加载八位 */
        s->bitcnt += 8;
    }

    /* 丢弃 need 位并更新缓冲区，始终保留零到七位 */
    s->bitbuf = (int)(val >> need);
    s->bitcnt -= need;

    /* 返回所需的位数，将上面的位清零 */
    return (int)(val & ((1L << need) - 1));
}
/*
 * 处理存储的数据块。
 *
 * 格式说明：
 *
 * - 在两位存储块类型（00）之后，存储块长度和存储字节是按字节对齐的，以便进行快速复制。因此，具有类型的最后一位的字节中剩余的位数，最多七位，将被丢弃。丢弃的位的值未定义，不应与任何期望进行比较。
 *
 * - 存储块长度的第二个取反副本不必进行检查，但最好还是这样做。
 *
 * - 存储块的长度可以为零。有时用于将压缩数据的子集进行字节对齐，以便进行随机访问或部分恢复。
 */
local int stored(struct state *s)
{
    unsigned len;       /* 存储块的长度 */

    /* 丢弃当前字节中剩余的位（假设 s->bitcnt < 8） */
    s->bitbuf = 0;
    s->bitcnt = 0;

    /* 获取长度并检查其一的补码 */
    if (s->incnt + 4 > s->inlen)
        return 2;                               /* 输入不足 */
    len = s->in[s->incnt++];
    len |= s->in[s->incnt++] << 8;
    if (s->in[s->incnt++] != (~len & 0xff) ||
        s->in[s->incnt++] != ((~len >> 8) & 0xff))
        return -2;                              /* 补码不匹配！ */

    /* 从输入复制 len 字节到输出 */
    if (s->incnt + len > s->inlen)
        return 2;                               /* 输入不足 */
    if (s->out != NIL) {
        if (s->outcnt + len > s->outlen)
            return 1;                           /* 输出空间不足 */
        while (len--)
            s->out[s->outcnt++] = s->in[s->incnt++];
    }
    else {                                      /* 仅扫描 */
        s->outcnt += len;
        s->incnt += len;
    }

    /* 完成有效的存储块处理 */
    return 0;
}
/*
 * 哈夫曼编码解码表。count[1..MAXBITS] 是每个长度的符号数量，对于规范编码，按顺序递增。
 * symbol[] 是按规范顺序排列的符号值，条目数是 count[] 中所有计数的总和。解码过程可以在下面的 decode() 函数中看到。
 */
struct huffman {
    short *count;       /* 每个长度的符号数量 */
    short *symbol;      /* 按规范顺序排列的符号 */
};

/*
 * 使用哈夫曼表 h 从流 s 中解码一个代码。如果出现错误，则返回符号或负值。如果所有长度都为零，即空代码，或者代码不完整并收到无效代码，则在读取 MAXBITS 位后返回 -10。
 *
 * 格式说明：
 *
 * - 在压缩数据中存储的代码相对于相同长度的简单整数顺序是位反转的。因此，在下面的代码中，从压缩数据中逐位提取位，并使用它们来构建代码值，与流中的相反顺序，以便进行简单的整数比较以进行解码。基于表的解码方案（如 zlib 中使用的）不需要进行这种反转。
 *
 * - 最短长度的第一个代码是全零。相同长度的后续代码只是前一个代码的整数递增。当移动到更长的长度时，会在代码后附加一个零位。对于完整的代码，最长长度的最后一个代码将是全一。
 *
 * - 不完整的代码由此解码器处理，因为它们在 deflate 格式中是允许的。请参阅 fixed() 和 dynamic() 的格式说明。
 */
#ifdef SLOW
local int decode(struct state *s, const struct huffman *h)
{
    int len;            /* 当前代码中的位数 */
    int code;           /* 被解码的长度位 */
    int first;          /* 长度为 len 的第一个代码 */
    int count;          /* 该长度的编码数量 */
    int index;          /* 该长度在符号表中的第一个编码的索引 */

    code = first = index = 0;  /* 初始化编码、第一个编码和索引 */
    for (len = 1; len <= MAXBITS; len++) {  /* 遍历编码长度 */
        code |= bits(s, 1);             /* 获取下一个比特位 */
        count = h->count[len];          /* 获取该长度的编码数量 */
        if (code - count < first)       /* 如果长度为len，则返回符号 */
            return h->symbol[index + (code - first)];
        index += count;                 /* 更新下一个长度的索引 */
        first += count;                 /* 更新下一个长度的第一个编码 */
        first <<= 1;                    /* 左移一位 */
        code <<= 1;                     /* 左移一位 */
    }
    return -10;                         /* 编码用完了 */
/*
 * A faster version of decode() for real applications of this code.   It's not
 * as readable, but it makes puff() twice as fast.  And it only makes the code
 * a few percent larger.
 */
#else /* !SLOW */
local int decode(struct state *s, const struct huffman *h)
{
    int len;            /* 当前代码中的位数 */
    int code;           /* 被解码的长度为len的代码 */
    int first;          /* 长度为len的第一个代码 */
    int count;          /* 长度为len的代码的数量 */
    int index;          /* 长度为len的第一个代码在符号表中的索引 */
    int bitbuf;         /* 来自流的位 */
    int left;           /* 下一个或剩余要处理的位数 */
    short *next;        /* 下一个代码的数量 */

    bitbuf = s->bitbuf;  /* 从状态结构体中获取位缓冲区 */
    left = s->bitcnt;    /* 从状态结构体中获取剩余位数 */
    code = first = index = 0;  /* 初始化code, first, index */
    len = 1;  /* 初始化长度为1 */
    next = h->count + 1;  /* 初始化下一个代码的数量 */

    while (1) {  /* 进入无限循环 */
        while (left--) {  /* 进入内部循环，处理剩余位数 */
            code |= bitbuf & 1;  /* 将bitbuf的最低位加入code */
            bitbuf >>= 1;  /* 右移一位，丢弃最低位 */
            count = *next++;  /* 获取下一个代码的数量 */
            if (code - count < first) { /* 如果长度为len，返回符号 */
                s->bitbuf = bitbuf;  /* 更新状态结构体中的位缓冲区 */
                s->bitcnt = (s->bitcnt - len) & 7;  /* 更新状态结构体中的剩余位数 */
                return h->symbol[index + (code - first)];  /* 返回符号表中的对应符号 */
            }
            index += count;  /* 更新索引 */
            first += count;  /* 更新第一个代码 */
            first <<= 1;  /* 左移一位，为下一个长度做准备 */
            code <<= 1;  /* 左移一位，为下一个长度做准备 */
            len++;  /* 长度加一 */
        }
        left = (MAXBITS+1) - len;  /* 计算剩余位数 */
        if (left == 0)  /* 如果剩余位数为0，跳出循环 */
            break;
        if (s->incnt == s->inlen)  /* 如果输入已经用完，跳出循环 */
            longjmp(s->env, 1);  /* 长跳转到错误处理 */
        bitbuf = s->in[s->incnt++];  /* 从输入中获取下一个位缓冲区 */
        if (left > 8)  /* 如果剩余位数大于8 */
            left = 8;  /* 将剩余位数设置为8 */
    }
    return -10;  /* 代码用完了，返回-10 */
}
#endif /* SLOW */
/*
 * 给定代码长度列表 length[0..n-1]，表示 n 个符号的一个规范哈夫曼编码，构建解码这些编码所需的表格。
 * 这些表格包括每个长度的代码数量，以及按长度排序的符号，保留它们在每个长度内的原始顺序。
 * 返回值为零表示完整的代码集，负数表示过度订阅的代码集，正数表示不完整的代码集。
 * 如果返回值为零或正数，则可以使用这些表格，但如果返回值为负数，则不能使用这些表格。
 * 如果返回值为零，则使用该表格的 decode() 不可能返回错误--足够的比特流将解析为一个符号。
 * 如果返回值为正数，则使用该表格的 decode() 可能对接收到的超出不完整长度的代码返回错误。
 *
 * decode() 不使用，但用于错误检查，h->count[0] 是不在代码中的 n 个符号的数量。
 * 因此 n - h->count[0] 是代码的数量。这对于检查具有多个符号的不完整代码非常有用，在动态块中这是一个错误。
 *
 * 假设：对于所有 i 在 0..n-1，0 <= length[i] <= MAXBITS
 * 这由 dynamic() 和 fixed() 中 length 数组的构造来保证，并且不是由 construct() 来验证的。
 *
 * 格式说明：
 *
 * - 不完整代码的允许和预期示例包括固定代码之一和任何代码，其中单个符号在 deflate 中编码为一个比特而不是零比特。参见 fixed() 和 dynamic() 的格式说明。
 *
 * - 在给定的代码长度内，符号按照代码位定义的升序顺序保留。
 */
local int construct(struct huffman *h, const short *length, int n)
{
    int symbol;         /* 在遍历 length[] 时的当前符号 */
    int len;            /* 在遍历 h->count[] 时的当前长度 */
    # 当前长度左侧可能的代码数量
    int left;           /* number of possible codes left of current length */
    # 每个长度的符号表中的偏移量
    short offs[MAXBITS+1];      /* offsets in symbol table for each length */

    /* 统计每个长度的代码数量 */
    for (len = 0; len <= MAXBITS; len++)
        h->count[len] = 0;
    for (symbol = 0; symbol < n; symbol++)
        (h->count[length[symbol]])++;   /* assumes lengths are within bounds */
    if (h->count[0] == n)               /* no codes! */
        return 0;                       /* complete, but decode() will fail */

    /* 检查长度集合是否过度订阅或不完整 */
    left = 1;                           /* one possible code of zero length */
    for (len = 1; len <= MAXBITS; len++) {
        left <<= 1;                     /* one more bit, double codes left */
        left -= h->count[len];          /* deduct count from possible codes */
        if (left < 0)
            return left;                /* over-subscribed--return negative */
    }                                   /* left > 0 means incomplete */

    /* 为每个长度生成符号表中的偏移量以进行排序 */
    offs[1] = 0;
    for (len = 1; len < MAXBITS; len++)
        offs[len + 1] = offs[len] + h->count[len];

    /*
     * 按长度和每个长度内的符号顺序将符号放入表中
     */
    for (symbol = 0; symbol < n; symbol++)
        if (length[symbol] != 0)
            h->symbol[offs[length[symbol]]++] = symbol;

    /* 对于完整的集合返回零，对于不完整的集合返回正数 */
    return left;
    /* 解码符号 */
    int symbol;         
    /* 复制的长度 */
    int len;            
    /* 复制的距离 */
    unsigned dist;      
    /* 长度编码的基础大小，用于长度编码 257..285 */
    static const short lens[29] = { 
        3, 4, 5, 6, 7, 8, 9, 10, 11, 13, 15, 17, 19, 23, 27, 31,
        35, 43, 51, 59, 67, 83, 99, 115, 131, 163, 195, 227, 258};
    /* 长度编码的额外位，用于长度编码 257..285 */
    static const short lext[29] = { 
        0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 2, 2, 2, 2,
        3, 3, 3, 3, 4, 4, 4, 4, 5, 5, 5, 5, 0};
    /* 距离编码的基础偏移，用于距离编码 0..29 */
    static const short dists[30] = { 
        1, 2, 3, 4, 5, 7, 9, 13, 17, 25, 33, 49, 65, 97, 129, 193,
        257, 385, 513, 769, 1025, 1537, 2049, 3073, 4097, 6145,
        8193, 12289, 16385, 24577};
    /* 距离编码的额外位，用于距离编码 0..29 */
    static const short dext[30] = { 
        0, 0, 0, 0, 1, 1, 2, 2, 3, 3, 4, 4, 5, 5, 6, 6,
        7, 7, 8, 8, 9, 9, 10, 10, 11, 11,
        12, 12, 13, 13};

    /* 解码文字和长度/距离对 */
    # 进入循环，开始解码
    do {
        # 解码得到符号
        symbol = decode(s, lencode);
        # 如果符号小于0，表示无效符号，返回该符号
        if (symbol < 0)
            return symbol;              /* invalid symbol */
        # 如果符号小于256，表示为字面值：符号即为字节
        if (symbol < 256) {             /* literal: symbol is the byte */
            /* 写出字面值 */
            if (s->out != NIL) {
                # 如果输出缓冲区已满，返回1
                if (s->outcnt == s->outlen)
                    return 1;
                # 将符号写入输出缓冲区
                s->out[s->outcnt] = symbol;
            }
            # 输出缓冲区计数加1
            s->outcnt++;
        }
        # 如果符号大于256，表示为长度
        else if (symbol > 256) {        /* length */
            /* 获取并计算长度 */
            symbol -= 257;
            # 如果长度大于等于29，返回-10，表示无效的固定代码
            if (symbol >= 29)
                return -10;             /* invalid fixed code */
            len = lens[symbol] + bits(s, lext[symbol]);

            /* 获取并检查距离 */
            symbol = decode(s, distcode);
            # 如果符号小于0，表示无效符号，返回该符号
            if (symbol < 0)
                return symbol;          /* invalid symbol */
            # 获取距离并计算
            dist = dists[symbol] + bits(s, dext[symbol]);
#ifndef INFLATE_ALLOW_INVALID_DISTANCE_TOOFAR_ARRR
            // 如果距离太远，则返回错误代码-11
            if (dist > s->outcnt)
                return -11;     /* distance too far back */
#endif

            /* 从距离位置复制长度字节的数据 */
            if (s->out != NIL) {
                // 如果输出计数加上长度超过了输出长度，则返回1
                if (s->outcnt + len > s->outlen)
                    return 1;
                // 循环复制数据
                while (len--) {
                    // 如果距离大于输出计数，则复制0，否则复制距离位置的数据
                    s->out[s->outcnt] =
#ifdef INFLATE_ALLOW_INVALID_DISTANCE_TOOFAR_ARRR
                        dist > s->outcnt ?
                            0 :
#endif
                            s->out[s->outcnt - dist];
                    s->outcnt++;
                }
            }
            else
                // 如果输出为空，则直接增加长度到输出计数
                s->outcnt += len;
        }
    } while (symbol != 256);            /* end of block symbol */

    /* 完成一个有效的固定或动态块 */
    return 0;
}
/*
 * 处理固定代码块。
 *
 * 格式说明：
 *
 * - 这种块类型对于压缩少量数据非常有用，对于动态块中代码描述的大小超过该块的自定义代码的好处。对于固定代码，没有位数用于代码描述。相反，文字/长度代码和距离代码的代码长度是固定的。每个符号的具体长度可以在下面的“for”循环中看到。
 *
 * - 文字/长度代码是完整的，但有两个无效的符号，如果接收到应该导致错误。这不能简单地实现为不完整的代码，因为这两个符号在代码的“中间”。它们的长度为八位，最长的文字/长度代码为九位。因此，代码必须用这些符号构造，并且在解码后必须检测无效的符号。
 *
 * - 固定距离代码也有两个无效的符号，如果接收到应该导致错误。由于所有距离代码的长度都相同，这可以实现为不完整的代码。然后在解码时检测无效代码。
 */
local int fixed(struct state *s)
{
    static int virgin = 1;
    static short lencnt[MAXBITS+1], lensym[FIXLCODES];
    static short distcnt[MAXBITS+1], distsym[MAXDCODES];
    static struct huffman lencode, distcode;

    /* 如果是第一次调用，则构建固定的哈夫曼表（可能不是线程安全的） */
    # 如果是第一次执行该代码块
    if (virgin) {
        # 定义变量symbol
        int symbol;
        # 定义长度为FIXLCODES的short数组lengths

        /* 构建lencode和distcode */
        # 设置lencode的count为lencnt
        lencode.count = lencnt;
        # 设置lencode的symbol为lensym
        lencode.symbol = lensym;
        # 设置distcode的count为distcnt
        distcode.count = distcnt;
        # 设置distcode的symbol为distsym

        /* literal/length table */
        # 对于symbol从0到143的范围，将lengths[symbol]设置为8
        for (symbol = 0; symbol < 144; symbol++)
            lengths[symbol] = 8;
        # 对于symbol从144到255的范围，将lengths[symbol]设置为9
        for (; symbol < 256; symbol++)
            lengths[symbol] = 9;
        # 对于symbol从256到279的范围，将lengths[symbol]设置为7
        for (; symbol < 280; symbol++)
            lengths[symbol] = 7;
        # 对于symbol从280到FIXLCODES的范围，将lengths[symbol]设置为8
        for (; symbol < FIXLCODES; symbol++)
            lengths[symbol] = 8;
        # 使用lengths数组构建lencode
        construct(&lencode, lengths, FIXLCODES);

        /* distance table */
        # 对于symbol从0到MAXDCODES的范围，将lengths[symbol]设置为5
        for (symbol = 0; symbol < MAXDCODES; symbol++)
            lengths[symbol] = 5;
        # 使用lengths数组构建distcode
        construct(&distcode, lengths, MAXDCODES);

        /* 只执行一次 */
        # 将virgin设置为0，表示不是第一次执行该代码块
        virgin = 0;
    }

    /* 解码数据直到遇到结束块代码 */
    # 返回codes函数的执行结果，传入参数s, lencode, distcode
    return codes(s, &lencode, &distcode);
}

local int dynamic(struct state *s)
{
    int nlen, ndist, ncode;             /* descriptor中的长度数量 */
    int index;                          /* lengths[]的索引 */
    int err;                            /* construct()的返回值 */
    short lengths[MAXCODES];            /* 描述符代码长度 */
    short lencnt[MAXBITS+1], lensym[MAXLCODES];         /* lencode内存 */
    short distcnt[MAXBITS+1], distsym[MAXDCODES];       /* distcode内存 */
    struct huffman lencode, distcode;   /* 长度和距离代码 */
    static const short order[19] =      /* code length codes的排列 */
        {16, 17, 18, 0, 8, 7, 9, 6, 10, 5, 11, 4, 12, 3, 13, 2, 14, 1, 15};

    /* 构建lencode和distcode */
    lencode.count = lencnt;
    lencode.symbol = lensym;
    distcode.count = distcnt;
    distcode.symbol = distsym;

    /* 获取每个表中的长度数量，检查长度 */
    nlen = bits(s, 5) + 257;
    ndist = bits(s, 5) + 1;
    ncode = bits(s, 4) + 4;
    if (nlen > MAXLCODES || ndist > MAXDCODES)
        return -3;                      /* 计数错误 */

    /* 读取代码长度代码长度（实际上），缺少的长度为零 */
    for (index = 0; index < ncode; index++)
        lengths[order[index]] = bits(s, 3);
    for (; index < 19; index++)
        lengths[order[index]] = 0;

    /* 为代码长度代码构建Huffman表（临时使用lencode） */
    err = construct(&lencode, lengths, 19);
    if (err != 0)               /* 这里需要完整的代码集 */
        return -4;

    /* 读取长度/字面值和距离代码长度表 */
    index = 0;
    while (index < nlen + ndist) {
        int symbol;             /* 解码后的值 */
        int len;                /* 最后重复的长度 */

        symbol = decode(s, &lencode);
        if (symbol < 0)
            return symbol;          /* 无效的符号 */
        if (symbol < 16)                /* 长度在0..15之间 */
            lengths[index++] = symbol;
        else {                          /* 重复指令 */
            len = 0;                    /* 假设重复的是0 */
            if (symbol == 16) {         /* 重复最后一个长度3..6次 */
                if (index == 0)
                    return -5;          /* 没有最后一个长度！ */
                len = lengths[index - 1];       /* 最后一个长度 */
                symbol = 3 + bits(s, 2);
            }
            else if (symbol == 17)      /* 重复0 3..10次 */
                symbol = 3 + bits(s, 3);
            else                        /* == 18, 重复0 11..138次 */
                symbol = 11 + bits(s, 7);
            if (index + symbol > nlen + ndist)
                return -6;              /* 长度过多！ */
            while (symbol--)            /* 重复最后一个或0的符号次数 */
                lengths[index++] = len;
        }
    }

    /* 检查是否存在块结束码 -- 应该有一个！ */
    if (lengths[256] == 0)
        return -9;

    /* 为文字/长度编码构建哈夫曼表 */
    err = construct(&lencode, lengths, nlen);
    if (err && (err < 0 || nlen != lencode.count[0] + lencode.count[1]))
        return -7;      /* 不完整的编码仅适用于单个长度为1的代码 */

    /* 为距离编码构建哈夫曼表 */
    err = construct(&distcode, lengths + nlen, ndist);
    if (err && (err < 0 || ndist != distcode.count[0] + distcode.count[1]))
        return -8;      /* 不完整的编码仅适用于单个长度为1的代码 */

    /* 解码数据直到块结束码 */
    return codes(s, &lencode, &distcode);
}  // 结束函数定义

int puff(unsigned char *dest,           /* pointer to destination pointer */
         unsigned long *destlen,        /* amount of output space */
         const unsigned char *source,   /* pointer to source data pointer */
         unsigned long *sourcelen)      /* amount of input available */
{
    struct state s;             /* input/output state */
    int last, type;             /* block information */
    int err;                    /* return value */

    /* initialize output state */
    s.out = dest;  // 将目标指针赋值给输出状态的out成员
    s.outlen = *destlen;                /* ignored if dest is NIL */  // 如果dest是NIL，则忽略destlen
    s.outcnt = 0;  // 输出计数初始化为0

    /* initialize input state */
    s.in = source;  // 将源数据指针赋值给输入状态的in成员
    s.inlen = *sourcelen;  // 将输入数据的长度赋值给输入状态的inlen成员
    s.incnt = 0;  // 输入计数初始化为0
    s.bitbuf = 0;  // 位缓冲初始化为0
    s.bitcnt = 0;  // 位计数初始化为0

    /* return if bits() or decode() tries to read past available input */
    if (setjmp(s.env) != 0)             /* if came back here via longjmp() */  // 如果通过longjmp()返回到这里
        err = 2;                        /* then skip do-loop, return error */  // 跳过do循环，返回错误
    else {
        /* process blocks until last block or error */
        do {
            last = bits(&s, 1);         /* one if last block */  // 如果是最后一个块，则为1
            type = bits(&s, 2);         /* block type 0..3 */  // 块类型为0..3
            err = type == 0 ?
                    stored(&s) :
                    (type == 1 ?
                        fixed(&s) :
                        (type == 2 ?
                            dynamic(&s) :
                            -1));       /* type == 3, invalid */  // 如果类型为3，则为无效
            if (err != 0)
                break;                  /* return with error */  // 返回错误
        } while (!last);
    }

    /* update the lengths and return */
    if (err <= 0) {
        *destlen = s.outcnt;  // 更新输出长度
        *sourcelen = s.incnt;  // 更新输入长度
    }
    return err;  // 返回错误码
}
```