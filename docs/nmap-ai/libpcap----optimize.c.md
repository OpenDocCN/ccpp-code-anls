# `nmap\libpcap\optimize.c`

```cpp
/*
 * 版权声明，版权归加利福尼亚大学所有
 * 允许在源代码和二进制形式下进行再发布和使用，无论是否进行修改
 * 在源代码发布中保留以上版权声明和本段文字，包括二进制代码的发布中
 * 在文档或其他提供的材料中包含以上版权声明和本段文字
 * 所有提及此软件特性或使用的广告材料都必须显示以下声明：
 * “本产品包含由加利福尼亚大学，劳伦斯伯克利实验室及其贡献者开发的软件。”
 * 未经特定事先书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品
 * 本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保
 * BPF代码中间表示的优化模块
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <pcap-types.h>

#include <stdio.h>
#include <stdlib.h>
#include <memory.h>
#include <setjmp.h>
#include <string.h>
#include <limits.h> /* for SIZE_MAX */
#include <errno.h>

#include "pcap-int.h"

#include "gencode.h"
#include "optimize.h"
#include "diag-control.h"

#ifdef HAVE_OS_PROTO_H
#include "os-proto.h"
#endif

#ifdef BDEBUG
/*
 * 过滤表达式优化器的内部“调试打印”标志
 * 只有在定义了BDEBUG时，才会存在打印代码，因此只有在定义了BDEBUG时才定义该标志和设置它的例程
 */
static int pcap_optimizer_debug;
/*
 * 设置优化器调试标志的例程。
 *
 * 这是为 libpcap 开发人员准备的，不是为了一般使用。
 * 如果你想在程序中设置这些标志，你将不得不自己声明这个例程，在 Windows 上带有适当的 DLL 导入属性；
 * 它没有在任何头文件中声明，并且不会在 libpcap 提供的任何头文件中声明。
 */
PCAP_API void pcap_set_optimizer_debug(int value);

PCAP_API_DEF void
pcap_set_optimizer_debug(int value)
{
    pcap_optimizer_debug = value;
}

/*
 * 用于过滤表达式优化器的内部“打印点图”标志。
 * 只有在定义了 BDEBUG 时，才会出现打印该内容的代码，因此该标志和设置它的例程只有在定义了 BDEBUG 时才会定义。
 */
static int pcap_print_dot_graph;

/*
 * 设置该标志的例程。
 *
 * 这是为 libpcap 开发人员准备的，不是为了一般使用。
 * 如果你想在程序中设置这些标志，你将不得不自己声明这个例程，在 Windows 上带有适当的 DLL 导入属性；
 * 它没有在任何头文件中声明，并且不会在 libpcap 提供的任何头文件中声明。
 */
PCAP_API void pcap_set_print_dot_graph(int value);

PCAP_API_DEF void
pcap_set_print_dot_graph(int value)
{
    pcap_print_dot_graph = value;
}

#endif

/*
 * lowest_set_bit()。
 *
 * 以 32 位整数作为参数。
 *
 * 如果传递的值非零，则从零开始计数返回最低位的索引。
 *
 * 如果传递零，则结果取决于平台和编译器。
 * 不要让它接触光线，不要给它水，不要在午夜后喂它，也不要将零传递给它。
 *
 * 这与字中尾随零的计数相同。
 */
#if PCAP_IS_AT_LEAST_GNUC_VERSION(3,4)
  /*
   * GCC 3.4 及更高版本；我们有 __builtin_ctz()。
   */
  #define lowest_set_bit(mask) ((u_int)__builtin_ctz(mask))
#elif defined(_MSC_VER)
  /*
   * Visual Studio；我们仅支持 2005 及更高版本，因此使用 _BitScanForward()。
   */
#ifdef __cplusplus
#include <intrin.h>
#endif

#ifndef __clang__
#pragma intrinsic(_BitScanForward)
#endif

static __forceinline u_int
lowest_set_bit(int mask)
{
    unsigned long bit;

    /*
     * Don't sign-extend mask if long is longer than int.
     * (It's currently not, in MSVC, even on 64-bit platforms, but....)
     */
    // 使用_BitScanForward函数查找mask中最低位的1的位置
    if (_BitScanForward(&bit, (unsigned int)mask) == 0)
        abort();    /* mask is zero */
    return (u_int)bit;
}
#elif defined(MSDOS) && defined(__DJGPP__)
  /*
   * MS-DOS with DJGPP, which declares ffs() in <string.h>, which
   * we've already included.
   */
  // 对于MS-DOS平台，使用ffs函数找到最低位的1的位置
  #define lowest_set_bit(mask)    ((u_int)(ffs((mask)) - 1))
#elif (defined(MSDOS) && defined(__WATCOMC__)) || defined(STRINGS_H_DECLARES_FFS)
  /*
   * MS-DOS with Watcom C, which has <strings.h> and declares ffs() there,
   * or some other platform (UN*X conforming to a sufficient recent version
   * of the Single UNIX Specification).
   */
  // 对于MS-DOS平台，使用ffs函数找到最低位的1的位置
  #include <strings.h>
  #define lowest_set_bit(mask)    (u_int)((ffs((mask)) - 1))
#else
/*
 * None of the above.
 * Use a perfect-hash-function-based function.
 */
// 使用基于完美哈希函数的方法找到最低位的1的位置
static u_int
lowest_set_bit(int mask)
{
    unsigned int v = (unsigned int)mask;

    static const u_int MultiplyDeBruijnBitPosition[32] = {
        0, 1, 28, 2, 29, 14, 24, 3, 30, 22, 20, 15, 25, 17, 4, 8,
        31, 27, 13, 23, 21, 19, 16, 7, 26, 12, 18, 6, 11, 5, 10, 9
    };

    /*
     * We strip off all but the lowermost set bit (v & ~v),
     * and perform a minimal perfect hash on it to look up the
     * number of low-order zero bits in a table.
     *
     * See:
     *
     *    http://7ooo.mooo.com/text/ComputingTrailingZerosHOWTO.pdf
     *
     *    http://supertech.csail.mit.edu/papers/debruijn.pdf
     */
    return (MultiplyDeBruijnBitPosition[((v & -v) * 0x077CB531U) >> 27]);
}
#endif

/*
 * Represents a deleted instruction.
 */
// 定义一个表示删除指令的宏
#define NOP -1
/*
 * 为使用-定义值注册数字。
 * 0到BPF_MEMWORDS-1表示相应的临时存储器位置。
 * A_ATOM是累加器，X_ATOM是索引寄存器。
 */
#define A_ATOM BPF_MEMWORDS
#define X_ATOM (BPF_MEMWORDS+1)

/*
 * 此定义用于表示*累加器和x寄存器在使用-定义计算中*。
 * 目前，使用-定义代码假设每条指令只有一个定义。
 */
#define AX_ATOM N_ATOMS

/*
 * 这些数据结构用于Cocke和Shwarz风格的值编号方案。
 * 由于流图是无环的，退出值可以从节点的前驱传播，前提是它是唯一定义的。
 */
struct valnode {
    int code;
    bpf_u_int32 v0, v1;
    int val;        /* 值编号 */
    struct valnode *next;
};

/* 用加载立即数操作码映射的整数常量。 */
#define K(i) F(opt_state, BPF_LD|BPF_IMM|BPF_W, i, 0U)

struct vmapinfo {
    int is_const;
    bpf_u_int32 const_val;
};

typedef struct {
    /*
     * 在出现错误时跳转的位置。
     */
    jmp_buf top_ctx;

    /*
     * 放置错误消息的缓冲区。
     */
    char *errbuf;

    /*
     * 一个标志，表示需要进一步优化。
     * 迭代传递会继续，直到给定的传递不再产生代码简化或分支移动。
     */
    int done;

    /*
     * XXX - 检测什么都不做的循环，只是重复的AND/OR上拉和边移动。
     * 如果连续100次都只是这样做，将其视为我们处于一个循环中的迹象，
     * 每次传递只是在代码中移动，并且最终回到原始配置。
     *
     * XXX - 我们需要一种非启发式的方法来检测或防止这样的循环。
     */
    int non_branch_movement_performed;
    # 定义一个无符号整数变量，表示控制流图中的块数，保证大于0，因为至少有一个RET指令
    u_int n_blocks;
    # 定义一个指向block结构体指针的数组
    struct block **blocks;
    # 定义一个无符号整数变量，表示控制流图中的边数，保证大于0，因为是块数的两倍
    u_int n_edges;
    # 定义一个指向edge结构体指针的数组
    struct edge **edges;

    '''
    用位向量集表示支配者。
    我们将集合大小向上取整到下一个二的幂。
    '''
    # 定义一个无符号整数变量，表示位向量的32位字的数量，保证大于0
    u_int nodewords;
    # 定义一个无符号整数变量，表示位向量的32位字的数量，保证大于0
    u_int edgewords;
    # 定义一个指向block结构体指针的数组
    struct block **levels;
    # 定义一个指向bpf_u_int32类型的指针，表示空间
    bpf_u_int32 *space;
#define BITS_PER_WORD (8*sizeof(bpf_u_int32))
/*
 * 定义每个字的位数
 */
#define SET_MEMBER(p, a) \
((p)[(unsigned)(a) / BITS_PER_WORD] & ((bpf_u_int32)1 << ((unsigned)(a) % BITS_PER_WORD)))
/*
 * 如果a在uset {p}中，则返回真
 */

#define SET_INSERT(p, a) \
(p)[(unsigned)(a) / BITS_PER_WORD] |= ((bpf_u_int32)1 << ((unsigned)(a) % BITS_PER_WORD))
/*
 * 将'a'添加到uset p中
 */

#define SET_DELETE(p, a) \
(p)[(unsigned)(a) / BITS_PER_WORD] &= ~((bpf_u_int32)1 << ((unsigned)(a) % BITS_PER_WORD))
/*
 * 从uset p中删除'a'
 */

#define SET_INTERSECT(a, b, n)\
{\
    register bpf_u_int32 *_x = a, *_y = b;\
    register u_int _n = n;\
    do *_x++ &= *_y++; while (--_n != 0);\
}
/*
 * a := a 与 b 的交集
 * n 必须保证大于 0
 */

#define SET_SUBTRACT(a, b, n)\
{\
    register bpf_u_int32 *_x = a, *_y = b;\
    register u_int _n = n;\
    do *_x++ &=~ *_y++; while (--_n != 0);\
}
/*
 * a := a - b
 * n 必须保证大于 0
 */

#define SET_UNION(a, b, n)\
{\
    register bpf_u_int32 *_x = a, *_y = b;\
    register u_int _n = n;\
    do *_x++ |= *_y++; while (--_n != 0);\
}
/*
 * a := a 与 b 的并集
 * n 必须保证大于 0
 */

    uset all_dom_sets;
    uset all_closure_sets;
    uset all_edge_sets;
/*
 * 定义三个uset集合

#define MODULUS 213
    struct valnode *hashtbl[MODULUS];
    bpf_u_int32 curval;
    bpf_u_int32 maxval;
/*
 * 定义MODULUS和两个bpf_u_int32类型的变量

    struct vmapinfo *vmap;
    struct valnode *vnode_base;
    struct valnode *next_vnode;
} opt_state_t;
/*
 * 定义结构体opt_state_t

typedef struct {
    /*
     * Place to longjmp to on an error.
     */
    jmp_buf top_ctx;
    /*
     * 在出现错误时跳转的位置
     */

    char *errbuf;
    /*
     * 用于存放错误消息的缓冲区
     */

    struct bpf_insn *fstart;
    struct bpf_insn *ftail;
    /*
     * 用于将代码转换为BPF所需的数组形式的指针
     */
} conv_state_t;
/*
 * 定义结构体conv_state_t

static void opt_init(opt_state_t *, struct icode *);
/*
 * 定义函数opt_init
 */
# 清理选项状态
static void opt_cleanup(opt_state_t *);

# 输出错误信息
static void PCAP_NORETURN opt_error(opt_state_t *, const char *, ...) PCAP_PRINTFLIKE(2, 3);

# 内部块
static void intern_blocks(opt_state_t *, struct icode *);

# 查找入边
static void find_inedges(opt_state_t *, struct block *);

# 如果定义了 BDEBUG，则输出调试信息
#ifdef BDEBUG
static void opt_dump(opt_state_t *, struct icode *);
#endif

# 如果未定义 MAX，则定义 MAX 为两者中的较大值
#ifndef MAX
#define MAX(a,b) ((a)>(b)?(a):(b))
#endif

# 递归查找级别
static void find_levels_r(opt_state_t *opt_state, struct icode *ic, struct block *b)
{
    int level;

    # 如果块已标记，则返回
    if (isMarked(ic, b))
        return;

    # 标记块
    Mark(ic, b);
    b->link = 0;

    # 如果有真分支
    if (JT(b)) {
        find_levels_r(opt_state, ic, JT(b));
        find_levels_r(opt_state, ic, JF(b));
        level = MAX(JT(b)->level, JF(b)->level) + 1;
    } else
        level = 0;
    b->level = level;
    b->link = opt_state->levels[level];
    opt_state->levels[level] = b;
}

# 查找级别
static void find_levels(opt_state_t *opt_state, struct icode *ic)
{
    # 将 opt_state->levels[] 数组初始化为 0
    memset((char *)opt_state->levels, 0, opt_state->n_blocks * sizeof(*opt_state->levels));
    # 取消所有块的标记
    unMarkAll(ic);
    # 递归查找级别
    find_levels_r(opt_state, ic, ic->root);
}

# 查找支配关系
static void find_dom(opt_state_t *opt_state, struct block *root)
{
    u_int i;
    int level;
    struct block *b;
    bpf_u_int32 *x;

    # 初始化集合包含所有节点
    x = opt_state->all_dom_sets;
    i = opt_state->n_blocks * opt_state->nodewords;
    while (i != 0) {
        --i;
        *x++ = 0xFFFFFFFFU;
    }
    # 根节点开始为空
    for (i = opt_state->nodewords; i != 0;) {
        --i;
        root->dom[i] = 0;
    }
}
    /* 从根节点到最高级别是找到的最高级别。 */
    for (level = root->level; level >= 0; --level) {
        /* 遍历当前级别的所有基本块 */
        for (b = opt_state->levels[level]; b; b = b->link) {
            /* 将当前基本块的 ID 插入到支配集合中 */
            SET_INSERT(b->dom, b->id);
            /* 如果当前基本块的 JT 指针为 0，则跳过 */
            if (JT(b) == 0)
                continue;
            /* 计算当前基本块的 JT 指针和支配集合的交集 */
            SET_INTERSECT(JT(b)->dom, b->dom, opt_state->nodewords);
            /* 计算当前基本块的 JF 指针和支配集合的交集 */
            SET_INTERSECT(JF(b)->dom, b->dom, opt_state->nodewords);
        }
    }
}

static void
propedom(opt_state_t *opt_state, struct edge *ep)
{
    // 将当前边的 ID 插入到边的支配集合中
    SET_INSERT(ep->edom, ep->id);
    // 如果当前边有后继
    if (ep->succ) {
        // 计算当前边的支配集合与后继边的支配集合的交集
        SET_INTERSECT(ep->succ->et.edom, ep->edom, opt_state->edgewords);
        // 计算当前边的支配集合与后继边的反向支配集合的交集
        SET_INTERSECT(ep->succ->ef.edom, ep->edom, opt_state->edgewords);
    }
}

/*
 * 计算边的支配者。
 * 假设图已经被分层，并且前驱已经建立。
 */
static void
find_edom(opt_state_t *opt_state, struct block *root)
{
    u_int i;
    uset x;
    int level;
    struct block *b;

    x = opt_state->all_edge_sets;
    /*
     * 在 opt_init() 中，我们已经确保了乘积不会溢出。
     */
    for (i = opt_state->n_edges * opt_state->edgewords; i != 0; ) {
        --i;
        x[i] = 0xFFFFFFFFU;
    }

    /* root->level 是找到的最高级别。 */
    memset(root->et.edom, 0, opt_state->edgewords * sizeof(*(uset)0));
    memset(root->ef.edom, 0, opt_state->edgewords * sizeof(*(uset)0));
    for (level = root->level; level >= 0; --level) {
        for (b = opt_state->levels[level]; b != 0; b = b->link) {
            propedom(opt_state, &b->et);
            propedom(opt_state, &b->ef);
        }
    }
}

/*
 * 找到流图的反向传递闭包。这些集合是反向的，因为我们找到达到给定节点的节点集合，而不是可以被节点到达的节点集合。
 *
 * 假设图已经被分层。
 */
static void
find_closure(opt_state_t *opt_state, struct block *root)
{
    int level;
    struct block *b;

    /*
     * 将集合初始化为不包含任何节点。
     */
    memset((char *)opt_state->all_closure_sets, 0,
          opt_state->n_blocks * opt_state->nodewords * sizeof(*opt_state->all_closure_sets));

    /* root->level 是找到的最高级别。 */
    # 从根节点的层级开始向上遍历
    for (level = root->level; level >= 0; --level) {
        # 遍历当前层级的所有节点
        for (b = opt_state->levels[level]; b; b = b->link) {
            # 将当前节点的 ID 插入到闭包中
            SET_INSERT(b->closure, b->id);
            # 如果当前节点的真分支为空，则跳过
            if (JT(b) == 0)
                continue;
            # 将当前节点的真分支的闭包与当前节点的闭包进行合并
            SET_UNION(JT(b)->closure, b->closure, opt_state->nodewords);
            # 将当前节点的假分支的闭包与当前节点的闭包进行合并
            SET_UNION(JF(b)->closure, b->closure, opt_state->nodewords);
        }
    }
# 返回使用寄存器的编号
# 如果使用了 A 寄存器，则返回 ATOM_A
# 如果使用了 X 寄存器，则返回 ATOM_X
# 如果 A 和 X 都被使用了，则返回 AX_ATOM
# 如果使用了临时存储器位置，则返回临时存储器位置的编号（例如，M[0]的编号为0）
# 如果以上情况都没有发生，则返回-1
# 实现可能应该改为数组访问
static int
atomuse(struct stmt *s)
{
    register int c = s->code;

    if (c == NOP)
        return -1;

    switch (BPF_CLASS(c)) {

    case BPF_RET:
        return (BPF_RVAL(c) == BPF_A) ? A_ATOM :
            (BPF_RVAL(c) == BPF_X) ? X_ATOM : -1;

    case BPF_LD:
    case BPF_LDX:
        '''
        如果内存位置少于2^31，则s->k应该可以转换为int而不会出现问题
        '''
        return (BPF_MODE(c) == BPF_IND) ? X_ATOM :
            (BPF_MODE(c) == BPF_MEM) ? (int)s->k : -1;

    case BPF_ST:
        return A_ATOM;

    case BPF_STX:
        return X_ATOM;

    case BPF_JMP:
    case BPF_ALU:
        if (BPF_SRC(c) == BPF_X)
            return AX_ATOM;
        return A_ATOM;

    case BPF_MISC:
        return BPF_MISCOP(c) == BPF_TXA ? X_ATOM : A_ATOM;
    }
    abort();
    # 不会执行到这里
}

# 返回由's'定义的寄存器编号
# 我们假设单个stmt不能定义多个寄存器
# 如果没有定义寄存器，则返回-1
# 实现可能应该改为数组访问
static int
atomdef(struct stmt *s)
{
    if (s->code == NOP)
        return -1;

    switch (BPF_CLASS(s->code)) {

    case BPF_LD:
    case BPF_ALU:
        return A_ATOM;

    case BPF_LDX:
        return X_ATOM;

    case BPF_ST:
    case BPF_STX:
        return s->k;

    case BPF_MISC:
        return BPF_MISCOP(s->code) == BPF_TAX ? X_ATOM : A_ATOM;
    }
    return -1;
}
/*
 * 计算由 'b' 使用、定义和杀死的寄存器集合。
 *
 * "使用" 表示 'b' 中的语句在 'b' 中的任何语句定义之前使用寄存器，即它使用前一个块的值留在该寄存器中。
 * "定义" 表示 'b' 中的语句定义它。
 * "杀死" 表示 'b' 中的语句在 'b' 中的任何语句使用它之前定义它，即它杀死前一个块的值留在该寄存器中。
 */
static void
compute_local_ud(struct block *b)
{
    struct slist *s;
    atomset def = 0, use = 0, killed = 0;
    int atom;

    for (s = b->stmts; s; s = s->next) {
        if (s->s.code == NOP)
            continue;
        atom = atomuse(&s->s);
        if (atom >= 0) {
            if (atom == AX_ATOM) {
                if (!ATOMELEM(def, X_ATOM))
                    use |= ATOMMASK(X_ATOM);
                if (!ATOMELEM(def, A_ATOM))
                    use |= ATOMMASK(A_ATOM);
            }
            else if (atom < N_ATOMS) {
                if (!ATOMELEM(def, atom))
                    use |= ATOMMASK(atom);
            }
            else
                abort();
        }
        atom = atomdef(&s->s);
        if (atom >= 0) {
            if (!ATOMELEM(use, atom))
                killed |= ATOMMASK(atom);
            def |= ATOMMASK(atom);
        }
    }
    if (BPF_CLASS(b->s.code) == BPF_JMP) {
        /*
         * XXX - what about RET?
         */
        atom = atomuse(&b->s);
        if (atom >= 0) {
            if (atom == AX_ATOM) {
                if (!ATOMELEM(def, X_ATOM))
                    use |= ATOMMASK(X_ATOM);
                if (!ATOMELEM(def, A_ATOM))
                    use |= ATOMMASK(A_ATOM);
            }
            else if (atom < N_ATOMS) {
                if (!ATOMELEM(def, atom))
                    use |= ATOMMASK(atom);
            }
            else
                abort();
        }
    }

    b->def = def;
    b->kill = killed;
}
    # 将变量 b 的成员变量 in_use 设置为 use 的值
    b->in_use = use;
}
/*
 * Assume graph is already leveled.
 */
static void
find_ud(opt_state_t *opt_state, struct block *root)
{
    int i, maxlevel;
    struct block *p;

    /*
     * root->level is the highest level no found;
     * count down from there.
     */
    maxlevel = root->level;
    // 从最高级别开始向下计数
    for (i = maxlevel; i >= 0; --i)
        for (p = opt_state->levels[i]; p; p = p->link) {
            compute_local_ud(p);
            p->out_use = 0;
        }

    // 从1级别到最高级别
    for (i = 1; i <= maxlevel; ++i) {
        for (p = opt_state->levels[i]; p; p = p->link) {
            p->out_use |= JT(p)->in_use | JF(p)->in_use;
            p->in_use |= p->out_use &~ p->kill;
        }
    }
}
static void
init_val(opt_state_t *opt_state)
{
    // 初始化值
    opt_state->curval = 0;
    opt_state->next_vnode = opt_state->vnode_base;
    // 将vmap数组初始化为0
    memset((char *)opt_state->vmap, 0, opt_state->maxval * sizeof(*opt_state->vmap));
    // 将hashtbl数组初始化为0
    memset((char *)opt_state->hashtbl, 0, sizeof opt_state->hashtbl);
}

/*
 * Because we really don't have an IR, this stuff is a little messy.
 *
 * This routine looks in the table of existing value number for a value
 * with generated from an operation with the specified opcode and
 * the specified values.  If it finds it, it returns its value number,
 * otherwise it makes a new entry in the table and returns the
 * value number of that entry.
 */
static bpf_u_int32
F(opt_state_t *opt_state, int code, bpf_u_int32 v0, bpf_u_int32 v1)
{
    u_int hash;
    bpf_u_int32 val;
    struct valnode *p;

    // 计算哈希值
    hash = (u_int)code ^ (v0 << 4) ^ (v1 << 8);
    hash %= MODULUS;

    // 在hashtbl中查找指定的值
    for (p = opt_state->hashtbl[hash]; p; p = p->next)
        if (p->code == code && p->v0 == v0 && p->v1 == v1)
            return p->val;
}
    /*
     * 如果未找到值，则分配一个新值，并为其分配一个新的值编号。
     *
     * opt_state->curval 最初为 0，表示 VAL_UNKNOWN；我们在使用它作为新值编号之前递增它，这意味着我们永远不会分配 VAL_UNKNOWN。
     *
     * XXX - 除非溢出，但我们可能不会有 2^32-1 个值；我们将 32 位视为有效无限。
     */
    val = ++opt_state->curval;
    if (BPF_MODE(code) == BPF_IMM &&
        (BPF_CLASS(code) == BPF_LD || BPF_CLASS(code) == BPF_LDX)) {
        opt_state->vmap[val].const_val = v0;
        opt_state->vmap[val].is_const = 1;
    }
    p = opt_state->next_vnode++;
    p->val = val;
    p->code = code;
    p->v0 = v0;
    p->v1 = v1;
    p->next = opt_state->hashtbl[hash];
    opt_state->hashtbl[hash] = p;

    return val;
}

static inline void
vstore(struct stmt *s, bpf_u_int32 *valp, bpf_u_int32 newval, int alter)
{
    // 如果需要修改并且新值不是未知，并且当前值等于新值，则将语句代码设置为 NOP
    if (alter && newval != VAL_UNKNOWN && *valp == newval)
        s->code = NOP;
    else
        // 否则将当前值设置为新值
        *valp = newval;
}

/*
 * 对二元操作符进行常量折叠。
 * （一元操作符在其他地方处理。）
 */
static void
fold_op(opt_state_t *opt_state, struct stmt *s, bpf_u_int32 v0, bpf_u_int32 v1)
{
    bpf_u_int32 a, b;

    // 获取第一个操作数的常量值
    a = opt_state->vmap[v0].const_val;
    // 获取第二个操作数的常量值
    b = opt_state->vmap[v1].const_val;

    switch (BPF_OP(s->code)) {
    case BPF_ADD:
        // 加法
        a += b;
        break;

    case BPF_SUB:
        // 减法
        a -= b;
        break;

    case BPF_MUL:
        // 乘法
        a *= b;
        break;

    case BPF_DIV:
        // 除法
        if (b == 0)
            opt_error(opt_state, "division by zero");
        a /= b;
        break;

    case BPF_MOD:
        // 取模
        if (b == 0)
            opt_error(opt_state, "modulus by zero");
        a %= b;
        break;

    case BPF_AND:
        // 与操作
        a &= b;
        break;

    case BPF_OR:
        // 或操作
        a |= b;
        break;

    case BPF_XOR:
        // 异或操作
        a ^= b;
        break;

    case BPF_LSH:
        /*
         * C 中超出类型宽度的左移是未定义的；我们将其视为将所有位移出。
         *
         * XXX - BPF 解释器不检查这一点，因此其行为取决于运行的处理器的行为。
         * 有些处理器会将所有位移出，有些处理器则不会进行位移。
         */
        if (b < 32)
            // 左移操作
            a <<= b;
        else
            a = 0;
        break;
    case BPF_RSH:
        /*
         * 如果要右移的位数大于类型的宽度，在 C 语言中是未定义的；
         * 我们将把它视为将所有位移出去。
         *
         * XXX - BPF 解释器不检查这一点，因此其行为取决于运行的处理器的行为。
         * 有些处理器会将所有位移出去，有些处理器则不进行位移。
         */
        if (b < 32)
            a >>= b;  // 如果要右移的位数小于32，则执行右移操作
        else
            a = 0;  // 否则将结果置为0
        break;

    default:
        abort();  // 默认情况下，终止程序
    }
    s->k = a;  // 将结果保存到 s->k 中
    s->code = BPF_LD|BPF_IMM;  // 设置 s->code 的值为 BPF_LD|BPF_IMM
    /*
     * XXX - 优化器循环检测。
     */
    opt_state->non_branch_movement_performed = 1;  // 设置优化器状态的非分支移动标志为1
    opt_state->done = 0;  // 设置优化器状态的完成标志为0
    # 返回下一个非空操作指令
    static inline struct slist *this_op(struct slist *s)
    {
        while (s != 0 && s->s.code == NOP)
            s = s->next;
        return s;
    }

    # 交换条件跳转的两个分支
    static void opt_not(struct block *b)
    {
        struct block *tmp = JT(b);

        JT(b) = JF(b);
        JF(b) = tmp;
    }

    # 对指定基本块进行优化
    static void opt_peep(opt_state_t *opt_state, struct block *b)
    {
        struct slist *s;
        struct slist *next, *last;
        bpf_u_int32 val;

        s = b->stmts;
        if (s == 0)
            return;

        last = s;
        }
        # 如果块末尾的比较是与常量的相等比较，并且没有人使用块末尾的 A 寄存器中的值，
        # 并且比较之前的操作是算术操作，有时可以进行优化
        }
        # jset #0        ->   never
        # jset #ffffffff ->   always
        if (b->s.code == (BPF_JMP|BPF_K|BPF_JSET)) {
            if (b->s.k == 0)
                JT(b) = JF(b);
            if (b->s.k == 0xffffffffU)
                JF(b) = JT(b);
        }
        # 如果我们正在与索引寄存器进行比较，并且索引寄存器是已知常量，我们可以直接与该常量进行比较
        val = b->val[X_ATOM];
        if (opt_state->vmap[val].is_const && BPF_SRC(b->s.code) == BPF_X) {
            bpf_u_int32 v = opt_state->vmap[val].const_val;
            b->s.code &= ~BPF_X;
            b->s.k = v;
        }
        # 如果累加器是已知常量，我们可以计算比较结果
        val = b->val[A_ATOM];
    }
    # 如果值是常量并且指令类型是立即数
    if (opt_state->vmap[val].is_const && BPF_SRC(b->s.code) == BPF_K) {
        # 获取常量值
        bpf_u_int32 v = opt_state->vmap[val].const_val;
        # 根据指令类型进行不同的操作
        switch (BPF_OP(b->s.code)) {

        case BPF_JEQ:
            # 判断是否相等
            v = v == b->s.k;
            break;

        case BPF_JGT:
            # 判断是否大于
            v = v > b->s.k;
            break;

        case BPF_JGE:
            # 判断是否大于等于
            v = v >= b->s.k;
            break;

        case BPF_JSET:
            # 按位与
            v &= b->s.k;
            break;

        default:
            # 终止程序
            abort();
        }
        # 如果跳转条件不相等
        if (JF(b) != JT(b)) {
            # 优化器循环检测
            opt_state->non_branch_movement_performed = 1;
            opt_state->done = 0;
        }
        # 根据条件跳转
        if (v)
            JF(b) = JT(b);
        else
            JT(b) = JF(b);
    }
/*
 * 计算's'表达式的符号值，并更新值表'val'中它定义的任何内容。如果'alter'为真，进行各种优化。
 * 如果符号评估和代码转换没有合并在一起，这段代码会更清晰。
 */
static void
opt_stmt(opt_state_t *opt_state, struct stmt *s, bpf_u_int32 val[], int alter)
{
    int op;
    bpf_u_int32 v;

    switch (s->code) {

    case BPF_LD|BPF_ABS|BPF_W:
    case BPF_LD|BPF_ABS|BPF_H:
    case BPF_LD|BPF_ABS|BPF_B:
        v = F(opt_state, s->code, s->k, 0L);
        vstore(s, &val[A_ATOM], v, alter);
        break;

    case BPF_LD|BPF_IND|BPF_W:
    case BPF_LD|BPF_IND|BPF_H:
    case BPF_LD|BPF_IND|BPF_B:
        v = val[X_ATOM];
        if (alter && opt_state->vmap[v].is_const) {
            s->code = BPF_LD|BPF_ABS|BPF_SIZE(s->code);
            s->k += opt_state->vmap[v].const_val;
            v = F(opt_state, s->code, s->k, 0L);
            /*
             * XXX - 优化器循环检测。
             */
            opt_state->non_branch_movement_performed = 1;
            opt_state->done = 0;
        }
        else
            v = F(opt_state, s->code, s->k, v);
        vstore(s, &val[A_ATOM], v, alter);
        break;

    case BPF_LD|BPF_LEN:
        v = F(opt_state, s->code, 0L, 0L);
        vstore(s, &val[A_ATOM], v, alter);
        break;

    case BPF_LD|BPF_IMM:
        v = K(s->k);
        vstore(s, &val[A_ATOM], v, alter);
        break;

    case BPF_LDX|BPF_IMM:
        v = K(s->k);
        vstore(s, &val[X_ATOM], v, alter);
        break;

    case BPF_LDX|BPF_MSH|BPF_B:
        v = F(opt_state, s->code, s->k, 0L);
        vstore(s, &val[X_ATOM], v, alter);
        break;
    # 如果操作码是 ALU 类型且是 NEG 操作
    case BPF_ALU|BPF_NEG:
        # 如果 alter 为真且 val[A_ATOM] 对应的常量值存在
        if (alter && opt_state->vmap[val[A_ATOM]].is_const) {
            # 将操作码改为加载立即数
            s->code = BPF_LD|BPF_IMM;
            '''
             * 以无符号算术方式执行取反操作；这是现代 BPF 引擎的做法，
             * 并且可以保证所有可能的值都可以被取反。
             * （是的，取反 0x80000000，即有符号 32 位补码的最小值，结果是 0x80000000，
             * 所以它仍然是负数，但是我们应该在这里执行所有无符号算术操作，
             * 以匹配现代 BPF 引擎的做法。）
             *
             * 将其表达为 0U - （无符号值），这样我们就不会收到有关对无符号值取反的编译器警告，
             * 也不会收到关于对 0x80000000 取反结果未定义的 UBSan 警告。
             '''
            s->k = 0U - opt_state->vmap[val[A_ATOM]].const_val;
            # 将 val[A_ATOM] 更新为常量值对应的立即数
            val[A_ATOM] = K(s->k);
        }
        else
            # 否则，根据操作码和 val[A_ATOM] 的值执行操作
            val[A_ATOM] = F(opt_state, s->code, val[A_ATOM], 0L);
        # 结束当前 case
        break;

    # 如果操作码是 ALU 类型且是 ADD、SUB、MUL、DIV、MOD、AND、OR、XOR、LSH 操作之一
    case BPF_ALU|BPF_ADD|BPF_K:
    case BPF_ALU|BPF_SUB|BPF_K:
    case BPF_ALU|BPF_MUL|BPF_K:
    case BPF_ALU|BPF_DIV|BPF_K:
    case BPF_ALU|BPF_MOD|BPF_K:
    case BPF_ALU|BPF_AND|BPF_K:
    case BPF_ALU|BPF_OR|BPF_K:
    case BPF_ALU|BPF_XOR|BPF_K:
    case BPF_ALU|BPF_LSH|BPF_K:
    # 如果操作码是 ALU、右移、常数的组合
    case BPF_ALU|BPF_RSH|BPF_K:
        # 获取操作码
        op = BPF_OP(s->code);
        # 如果需要修改
        if (alter) {
            # 如果常数为0
            if (s->k == 0) {
                """
                优化常数为0的操作。
                不要优化掉 "sub #0"，因为后面可能需要修复生成的数学代码。
                如果除以0或者取模0，则失败。
                """
                if (op == BPF_ADD ||
                    op == BPF_LSH || op == BPF_RSH ||
                    op == BPF_OR || op == BPF_XOR) {
                    s->code = NOP;
                    break;
                }
                if (op == BPF_MUL || op == BPF_AND) {
                    s->code = BPF_LD|BPF_IMM;
                    val[A_ATOM] = K(s->k);
                    break;
                }
                if (op == BPF_DIV)
                    opt_error(opt_state,
                        "division by zero");
                if (op == BPF_MOD)
                    opt_error(opt_state,
                        "modulus by zero");
            }
            # 如果常数在值映射中是常数
            if (opt_state->vmap[val[A_ATOM]].is_const) {
                fold_op(opt_state, s, val[A_ATOM], K(s->k));
                val[A_ATOM] = K(s->k);
                break;
            }
        }
        # 将操作码应用于值
        val[A_ATOM] = F(opt_state, s->code, val[A_ATOM], K(s->k));
        break;

    # 如果操作码是 ALU、加法、X寄存器的组合
    case BPF_ALU|BPF_ADD|BPF_X:
    # 如果操作码是 ALU、减法、X寄存器的组合
    case BPF_ALU|BPF_SUB|BPF_X:
    # 如果操作码是 ALU、乘法、X寄存器的组合
    case BPF_ALU|BPF_MUL|BPF_X:
    # 如果操作码是 ALU、除法、X寄存器的组合
    case BPF_ALU|BPF_DIV|BPF_X:
    # 如果操作码是 ALU、取模、X寄存器的组合
    case BPF_ALU|BPF_MOD|BPF_X:
    # 如果操作码是 ALU、按位与、X寄存器的组合
    case BPF_ALU|BPF_AND|BPF_X:
    # 如果操作码是 ALU、按位或、X寄存器的组合
    case BPF_ALU|BPF_OR|BPF_X:
    # 如果操作码是 ALU、按位异或、X寄存器的组合
    case BPF_ALU|BPF_XOR|BPF_X:
    # 如果操作码是 ALU、左移、X寄存器的组合
    case BPF_ALU|BPF_LSH|BPF_X:
    # 如果操作码是 MISC、将A寄存器的值存储到X寄存器
    case BPF_MISC|BPF_TXA:
        # 将A寄存器的值存储到X寄存器
        vstore(s, &val[A_ATOM], val[X_ATOM], alter);
        break;
    # 如果指令是从内存加载数据
    case BPF_LD|BPF_MEM:
        # 从值数组中获取指定索引的值
        v = val[s->k];
        # 如果需要修改并且该值在优化状态的值映射中是常量
        if (alter && opt_state->vmap[v].is_const) {
            # 将指令改为立即数加载
            s->code = BPF_LD|BPF_IMM;
            # 将指令的索引设置为该常量值
            s->k = opt_state->vmap[v].const_val;
            """
            优化器循环检测。
            """
            # 标记非分支移动已执行
            opt_state->non_branch_movement_performed = 1;
            # 标记优化未完成
            opt_state->done = 0;
        }
        # 将加载的值存储到指定的寄存器中
        vstore(s, &val[A_ATOM], v, alter);
        # 退出switch语句
        break;

    # 如果指令是将A寄存器的值传送到X寄存器
    case BPF_MISC|BPF_TAX:
        # 将A寄存器的值存储到X寄存器
        vstore(s, &val[X_ATOM], val[A_ATOM], alter);
        # 退出switch语句
        break;

    # 如果指令是从内存加载数据到X寄存器
    case BPF_LDX|BPF_MEM:
        # 从值数组中获取指定索引的值
        v = val[s->k];
        # 如果需要修改并且该值在优化状态的值映射中是常量
        if (alter && opt_state->vmap[v].is_const) {
            # 将指令改为立即数加载到X寄存器
            s->code = BPF_LDX|BPF_IMM;
            # 将指令的索引设置为该常量值
            s->k = opt_state->vmap[v].const_val;
            """
            优化器循环检测。
            """
            # 标记非分支移动已执行
            opt_state->non_branch_movement_performed = 1;
            # 标记优化未完成
            opt_state->done = 0;
        }
        # 将加载的值存储到X寄存器
        vstore(s, &val[X_ATOM], v, alter);
        # 退出switch语句
        break;

    # 如果指令是将A寄存器的值存储到内存
    case BPF_ST:
        # 将A寄存器的值存储到指定的内存位置
        vstore(s, &val[s->k], val[A_ATOM], alter);
        # 退出switch语句
        break;

    # 如果指令是将X寄存器的值存储到内存
    case BPF_STX:
        # 将X寄存器的值存储到指定的内存位置
        vstore(s, &val[s->k], val[X_ATOM], alter);
        # 退出switch语句
        break;
    }
}
# 定义函数 deadstmt，接受优化状态、语句和最后一个语句的数组作为参数
static void
deadstmt(opt_state_t *opt_state, register struct stmt *s, register struct stmt *last[])
{
    register int atom;

    # 获取语句中使用的原子
    atom = atomuse(s);
    # 如果使用的原子大于等于 0
    if (atom >= 0) {
        # 如果使用的原子是 AX_ATOM
        if (atom == AX_ATOM) {
            # 将 X_ATOM 和 A_ATOM 的最后一个语句置为 0
            last[X_ATOM] = 0;
            last[A_ATOM] = 0;
        }
        # 否则
        else
            # 将对应原子的最后一个语句置为 0
            last[atom] = 0;
    }
    # 获取语句中定义的原子
    atom = atomdef(s);
    # 如果定义的原子大于等于 0
    if (atom >= 0) {
        # 如果对应原子的最后一个语句存在
        if (last[atom]) {
            # 优化器循环检测
            opt_state->non_branch_movement_performed = 1;
            opt_state->done = 0;
            # 将对应原子的最后一个语句的代码置为 NOP
            last[atom]->code = NOP;
        }
        # 将对应原子的最后一个语句置为当前语句
        last[atom] = s;
    }
}

# 定义函数 opt_deadstores，接受优化状态和块作为参数
static void
opt_deadstores(opt_state_t *opt_state, register struct block *b)
{
    register struct slist *s;
    register int atom;
    struct stmt *last[N_ATOMS];

    # 将最后一个语句的数组初始化为 0
    memset((char *)last, 0, sizeof last);

    # 遍历块中的语句
    for (s = b->stmts; s != 0; s = s->next)
        # 对每个语句调用 deadstmt 函数
        deadstmt(opt_state, &s->s, last);
    # 对块中的语句调用 deadstmt 函数
    deadstmt(opt_state, &b->s, last);

    # 遍历原子数组
    for (atom = 0; atom < N_ATOMS; ++atom)
        # 如果对应原子的最后一个语句存在且不在块的输出使用中
        if (last[atom] && !ATOMELEM(b->out_use, atom)) {
            # 将对应原子的最后一个语句的代码置为 NOP
            last[atom]->code = NOP;
            # 优化器循环检测
            opt_state->non_branch_movement_performed = 1;
            opt_state->done = 0;
        }
}

# 定义函数 opt_blk，接受优化状态、块和是否执行语句的标志作为参数
static void
opt_blk(opt_state_t *opt_state, struct block *b, int do_stmts)
{
    struct slist *s;
    struct edge *p;
    int i;
    bpf_u_int32 aval, xval;

    # 如果条件为假，则执行以下代码
    # for 循环遍历块中的语句
    #     如果语句的代码类别是 BPF_JMP，则将 do_stmts 置为 0 并跳出循环
    # endif

    # 初始化原子值
    p = b->in_edges;
    # 如果没有前驱，则块中所有值都是未定义的
    memset((char *)b->val, 0, sizeof(b->val));
}
    } else {
        /*
         * 从我们的前任那里继承值。
         *
         * 首先，从沿着第一条边到达这个节点的前任那里获取值。
         */
        memcpy((char *)b->val, (char *)p->pred->val, sizeof(b->val));
        /*
         * 现在看看所有其他到达这个节点的节点。
         * 如果沿着那条边的前任，一个寄存器的值与我们的值不同（即，控制路径正在合并，并且合并路径为该寄存器分配了不同的值），则给该寄存器赋予未定义的值0。
         */
        while ((p = p->next) != NULL) {
            for (i = 0; i < N_ATOMS; ++i)
                if (b->val[i] != p->pred->val[i])
                    b->val[i] = 0;
        }
    }
    aval = b->val[A_ATOM];
    xval = b->val[X_ATOM];
    for (s = b->stmts; s; s = s->next)
        opt_stmt(opt_state, &s->s, b->val, do_stmts);
    /*
     * 这是一个特殊情况：如果我们不使用这个代码块中的任何内容，并且我们用累加器或索引寄存器加载一个已经存在的值，
     * 或者如果这个代码块是一个返回语句，那么就可以消除所有语句。
     *
     * XXX - 如果它执行存储操作怎么办？可能属于“如果我们不使用这个代码块中的任何内容”的范畴，
     * 即，如果我们使用了这个代码块设置的任何不同值的内存位置，那么我们就使用了这个代码块的内容。
     *
     * XXX - 为什么使用这个代码块的内容很重要？如果累加器或索引寄存器的值没有改变，即使我们使用了那个值，也没关系吗？
     *
     * XXX - 如果我们用不同的值加载累加器，并且代码块以条件分支结束，显然我们不能消除它，因为分支取决于那个值。
     * 对于索引寄存器，只有当测试是针对索引寄存器的值而不是一个常数时，条件分支才依赖于索引寄存器的值；
     * 如果没有任何东西使用我们放入索引寄存器的值，并且我们没有针对索引寄存器的值进行测试，
     * 并且没有其他阻止我们消除这个代码块的问题，那么我们可以消除它吗？
     */
    if (do_stmts &&
        ((b->out_use == 0 &&
          aval != VAL_UNKNOWN && b->val[A_ATOM] == aval &&
          xval != VAL_UNKNOWN && b->val[X_ATOM] == xval) ||
         BPF_CLASS(b->s.code) == BPF_RET)) {
        if (b->stmts != 0) {
            b->stmts = 0;
            /*
             * XXX - 优化器循环检测。
             */
            opt_state->non_branch_movement_performed = 1;
            opt_state->done = 0;
        }
    } else {
        opt_peep(opt_state, b);
        opt_deadstores(opt_state, b);
    }
    /*
     * 为分支优化器设置值。
     */
    # 如果指令类型是常数类型
    if (BPF_SRC(b->s.code) == BPF_K)
        # 将常数值赋给 b->oval
        b->oval = K(b->s.k);
    # 否则
    else
        # 将 X_ATOM 对应的值赋给 b->oval
        b->oval = b->val[X_ATOM];
    # 将 b->s.code 赋给 b->et.code
    b->et.code = b->s.code;
    # 将 b->s.code 的相反数赋给 b->ef.code
    b->ef.code = -b->s.code;
}
/*
 * 如果在'succ'退出时使用的任何寄存器具有与'b'对应的退出值不同的情况下返回true。
 */
static int
use_conflict(struct block *b, struct block *succ)
{
    int atom;
    atomset use = succ->out_use;

    if (use == 0)
        return 0;

    for (atom = 0; atom < N_ATOMS; ++atom)
        if (ATOMELEM(use, atom))
            if (b->val[atom] != succ->val[atom])
                return 1;
    return 0;
}

/*
 * 给定一个作为边的后继的块，以及支配该边的边，如果该块是替换后者边的后继的候选块，则返回指向该块的子块的指针，如果第一个块的子块都不是候选块，则返回NULL。
 */
static struct block *
fold_edge(struct block *child, struct edge *ep)
{
    int sense;
    bpf_u_int32 aval0, aval1, oval0, oval1;
    int code = ep->code;

    if (code < 0) {
        /*
         * 这条边是“如果为假则跳转”的边。
         */
        code = -code;
        sense = 0;
    } else {
        /*
         * 这条边是“如果为真则跳转”的边。
         */
        sense = 1;
    }

    /*
     * 如果我们手头的块末尾的分支的操作码与我们手头的边对应的分支的操作码不同，则这些分支的测试不是测试相同的条件，因此第一个块跳转的块不是替换边的后继的候选块。
     */
    if (child->s.code != code)
        return 0;

    aval0 = child->val[A_ATOM];
    oval0 = child->oval;
    aval1 = ep->pred->val[A_ATOM];
    oval1 = ep->pred->oval;
    /*
     * 如果从前驱边的退出时A寄存器的值与后继块的退出时A寄存器的值不同，
     * 则第一个块分支到的块不是替换后继边的候选块。
     */
    if (aval0 != aval1)
        return 0;

    if (oval0 == oval1)
        /*
         * 分支指令的操作数相同，因此分支测试相同条件，
         * 如果通过真分支到达这里，则结果为真，否则为假。
         */
        return sense ? JT(child) : JF(child);

    if (sense && code == (BPF_JMP|BPF_JEQ|BPF_K))
        /*
         * 在这一点上，我们只知道如果我们通过了真分支，那么它是与常量的相等比较。
         *
         * 也就是说，如果我们通过了真分支，并且分支是与常量的相等比较，我们知道累加器包含该常量。
         * 如果我们通过了假分支，或者比较不是与常量进行的，我们不知道累加器中的内容。
         *
         * 我们依赖于不同的常量具有不同的值编号这一事实。
         */
        return JF(child);

    return 0;
}
/*
 * 如果我们可以使这条边直接指向边的当前后继的子节点，那么就这样做。
 */
static void
opt_j(opt_state_t *opt_state, struct edge *ep)
{
    register u_int i, k;
    register struct block *target;

    /*
     * 这条边是否指向一个块，如果在其末尾的测试成功，它会转到DAG的叶节点，即返回语句？
     * 如果是这样，就没有什么可以优化的。
     */
    if (JT(ep->succ) == 0)
        return;

    /*
     * 这条边是否指向一个块，该块反过来无论测试结果如何都会转到相同的块？
     */
    if (JT(ep->succ) == JF(ep->succ)) {
        /*
         * 可以消除公共分支目标，前提是没有数据依赖关系。
         *
         * 检查从这条边的后继转到的块的出口处使用的任何寄存器，在那一点上的值是否与从这条边的前驱出口处的值不同。如果没有，那么这条边的前驱可以直接转到这条边的后继所指向的块，绕过这条边的后继，因为这条边的后继没有进行任何计算，其结果与其前面的块不同，并且没有进行任何测试其结果的重要性。
         */
        if (!use_conflict(ep->pred, JT(ep->succ))) {
            /*
             * 没有冲突。
             * 使这条边指向后继的块。
             *
             * XXX - 优化器循环检测。
             */
            opt_state->non_branch_movement_performed = 1;
            opt_state->done = 0;
            ep->succ = JT(ep->succ);
        }
    }
    /*
     * 对于每个边支配者，如果匹配了该边的后继节点，就将该边的后继节点提升为其孙子节点。
     *
     * XXX 这里我们违反了集合抽象，而是选择了一个相当高效的循环。
     */
 top:
    for (i = 0; i < opt_state->edgewords; ++i) {
        /* 在支配者位图中的第i个字 */
        register bpf_u_int32 x = ep->edom[i];

        while (x != 0) {
            /* 找到该字中的下一个支配者，并标记为已找到 */
            k = lowest_set_bit(x);
            x &=~ ((bpf_u_int32)1 << k);
            k += i * BITS_PER_WORD;

            target = fold_edge(ep->succ, opt_state->edges[k]);
            /*
             * 我们有一个候选者可以替换ep的后继节点。
             *
             * 检查是否存在数据依赖性，如果我们移动边将会违反节点之间的数据依赖性；
             * 即，如果在候选者的退出点上使用的任何寄存器在那一点的值与我们退出边的前驱节点时的值不同，那么就会违反数据依赖性。
             */
            if (target != 0 && !use_conflict(ep->pred, target)) {
                /*
                 * 替换ep的后继节点是安全的；这样做，并且注意到我们至少做了一次改变。
                 *
                 * XXX - 这是优化器陷入无限循环时发生的操作之一。
                 */
                opt_state->done = 0;
                ep->succ = target;
                if (JT(target) != 0)
                    /*
                     * 除非我们遇到叶子节点，否则重新开始。
                     */
                    goto top;
                return;
            }
        }
    }
}
/*
 * 这个函数和 and_pullup() 是否是 BPF+ 论文中第 6.1.2 节中描述的内容？
 * 注意，这里是查看块支配者，而不是边支配者。
 * 不认为是。
 * "A or B" 编译成
 *          A
 *       t / \ f
 *        /   B
 *       / t / \ f
 *      \   /
 *       \ /
 *        X
 */
static void
or_pullup(opt_state_t *opt_state, struct block *b)
{
    bpf_u_int32 val;
    int at_top;
    struct block *pull;
    struct block **diffp, **samep;
    struct edge *ep;

    ep = b->in_edges;
    if (ep == 0)
        return;

    /*
     * 确保每个前驱加载相同的值。
     * XXX 为什么？
     */
    val = ep->pred->val[A_ATOM];
    for (ep = ep->next; ep != 0; ep = ep->next)
        if (val != ep->pred->val[A_ATOM])
            return;

    /*
     * 对于进入此块的边列表中的第一条边，查看该边的前驱是通过真分支还是假分支到达这里。
     */
    if (JT(b->in_edges->pred) == b)
        diffp = &JT(b->in_edges->pred);    /* jt */
    else
        diffp = &JF(b->in_edges->pred);    /* jf */

    /*
     * diffp 是指向块的指针的指针。
     *
     * 沿着假链向下查看，尽可能远地查看，确保每个跳转比较都与原始块做的一样。
     *
     * 如果在到达不同的跳转比较之前到达底部，就退出。这里没什么可做的。XXX - 不，这个版本是在检查离开块的值；这是来自 BPF+ 的 pullup 程序。
     */
    at_top = 1;
    for (;;) {
        /*
         * 如果这不会有任何进展，那就结束
         */
        if (*diffp == 0)
            return;

        /*
         * 如果前任节点不会去和我们一样的地方，那就结束
         *
         * 这个块的真边指向和 b 的真边指向的位置一样吗？
         */
        if (JT(*diffp) != JT(b))
            return;

        /*
         * 如果这个节点不是那个节点的支配者，那就结束
         *
         * b 是否支配 diffp？
         */
        if (!SET_MEMBER((*diffp)->dom, b->id))
            return;

        /*
         * 如果那个节点的 A 值不等于上面的 A 值，就跳出循环
         */
        if ((*diffp)->val[A_ATOM] != val)
            break;

        /*
         * 获取那个节点的 JF
         * 走向假路径
         */
        diffp = &JF(*diffp);
        at_top = 0;
    }

    /*
     * 现在我们在 b 下面的链中找到了一个不同的跳转比较，继续向下搜索，直到找到另一个查看原始值的跳转比较。
     * 这个跳转比较应该被提升。
     */
    samep = &JF(*diffp);
    for (;;) {
        /*
         * 如果这不会有任何进展，那就结束了 XXX
         */
        if (*samep == 0)
            return;

        /*
         * 如果前驱节点不会去和我们去的地方一样，那就结束了 XXX
         */
        if (JT(*samep) != JT(b))
            return;

        /*
         * 如果这个节点不是那个节点的支配者，那就结束了 XXX
         *
         * 节点 b 是否支配节点 samep？
         */
        if (!SET_MEMBER((*samep)->dom, b->id))
            return;

        /*
         * 如果这个节点的 A 值和上面的 A 值一样，那就跳出循环 XXX
         */
        if ((*samep)->val[A_ATOM] == val)
            break;

        /* XXX 需要检查 dp0 和 dp1 之间是否存在数据依赖关系。
           目前，代码生成器不会产生这样的依赖关系。 */
        samep = &JF(*samep);
    }
#ifdef notdef
    /* 如果定义了 notdef，则执行以下代码块 */
    /* XXX This doesn't cover everything. */
    /* 注意：这里并没有覆盖所有情况 */
    for (i = 0; i < N_ATOMS; ++i)
        /* 遍历 N_ATOMS，检查是否有不同的值 */
        if ((*samep)->val[i] != pred->val[i])
            /* 如果同一个节点的值与前一个节点的值不同，则返回 */
            return;
#endif
    /* Pull up the node. */
    /* 拉起节点 */
    pull = *samep;
    *samep = JF(pull);
    JF(pull) = *diffp;

    /*
     * At the top of the chain, each predecessor needs to point at the
     * pulled up node.  Inside the chain, there is only one predecessor
     * to worry about.
     */
    /*
     * 在链的顶部，每个前驱节点都需要指向拉起的节点。在链内部，只需要关注一个前驱节点。
     */
    if (at_top) {
        for (ep = b->in_edges; ep != 0; ep = ep->next) {
            if (JT(ep->pred) == b)
                JT(ep->pred) = pull;
            else
                JF(ep->pred) = pull;
        }
    }
    else
        *diffp = pull;

    /*
     * XXX - this is one of the operations that happens when the
     * optimizer gets into one of those infinite loops.
     */
    /* 注意：这是优化器陷入无限循环时发生的操作之一 */
    opt_state->done = 0;
}

static void
and_pullup(opt_state_t *opt_state, struct block *b)
{
    bpf_u_int32 val;
    int at_top;
    struct block *pull;
    struct block **diffp, **samep;
    struct edge *ep;

    ep = b->in_edges;
    if (ep == 0)
        return;

    /*
     * Make sure each predecessor loads the same value.
     */
    /* 确保每个前驱节点加载相同的值 */
    val = ep->pred->val[A_ATOM];
    for (ep = ep->next; ep != 0; ep = ep->next)
        if (val != ep->pred->val[A_ATOM])
            return;

    if (JT(b->in_edges->pred) == b)
        diffp = &JT(b->in_edges->pred);
    else
        diffp = &JF(b->in_edges->pred);

    at_top = 1;
    for (;;) {
        if (*diffp == 0)
            return;

        if (JF(*diffp) != JF(b))
            return;

        if (!SET_MEMBER((*diffp)->dom, b->id))
            return;

        if ((*diffp)->val[A_ATOM] != val)
            break;

        diffp = &JT(*diffp);
        at_top = 0;
    }
    samep = &JT(*diffp);
}
    # 进入无限循环
    for (;;) {
        # 如果指针 samep 指向的值为 0，则返回
        if (*samep == 0)
            return;

        # 如果指针 samep 指向的结构体中的 JF 值不等于 b 结构体中的 JF 值，则返回
        if (JF(*samep) != JF(b))
            return;

        # 如果 b 结构体的 id 不在指针 samep 指向的结构体中的 dom 集合中，则返回
        if (!SET_MEMBER((*samep)->dom, b->id))
            return;

        # 如果指针 samep 指向的结构体中的 val 数组中的 A_ATOM 元素的值等于 val，则跳出循环
        if ((*samep)->val[A_ATOM] == val)
            break;

        /* XXX 需要检查 diffp 和 samep 之间是否存在数据依赖关系
           目前，代码生成器不会产生这样的依赖关系。 */
        # 更新指针 samep 指向的地址为 JT(*samep) 的地址
        samep = &JT(*samep);
    }
#ifdef notdef
    /* 如果未定义，执行以下操作 */
    /* XXX This doesn't cover everything. */
    /* 这里并未覆盖所有情况 */
    for (i = 0; i < N_ATOMS; ++i)
        /* 遍历N_ATOMS个元素 */
        if ((*samep)->val[i] != pred->val[i])
            /* 如果相同指针的第i个值不等于pred指针的第i个值 */
            return;
            /* 返回 */
#endif
    /* 拉起节点 */
    pull = *samep;
    *samep = JT(pull);
    JT(pull) = *diffp;

    /*
     * 在链的顶部，每个前驱节点都需要指向拉起的节点。在链的内部，只需要担心一个前驱节点。
     */
    if (at_top) {
        for (ep = b->in_edges; ep != 0; ep = ep->next) {
            if (JT(ep->pred) == b)
                JT(ep->pred) = pull;
            else
                JF(ep->pred) = pull;
        }
    }
    else
        *diffp = pull;

    /*
     * XXX - this is one of the operations that happens when the
     * optimizer gets into one of those infinite loops.
     */
    /* 这是优化器陷入无限循环时发生的操作之一 */
    opt_state->done = 0;
}

static void
opt_blks(opt_state_t *opt_state, struct icode *ic, int do_stmts)
{
    int i, maxlevel;
    struct block *p;

    init_val(opt_state);
    maxlevel = ic->root->level;

    find_inedges(opt_state, ic->root);
    for (i = maxlevel; i >= 0; --i)
        for (p = opt_state->levels[i]; p; p = p->link)
            opt_blk(opt_state, p, do_stmts);

    if (do_stmts)
        /*
         * No point trying to move branches; it can't possibly
         * make a difference at this point.
         *
         * XXX - this might be after we detect a loop where
         * we were just looping infinitely moving branches
         * in such a fashion that we went through two or more
         * versions of the machine code, eventually returning
         * to the first version.  (We're really not doing a
         * full loop detection, we're just testing for two
         * passes in a row where we do nothing but
         * move branches.)
         */
        /* 没有必要尝试移动分支；在这一点上可能不会有任何区别 */
        /* 这可能是在我们检测到一个循环之后，我们只是无限地移动分支，以至于我们通过两个或更多版本的机器代码，最终返回到第一个版本。(我们并不是真正做一个完整的循环检测，我们只是测试连续两次我们除了移动分支之外什么也不做的情况。) */
        return;

    /*
     * Is this what the BPF+ paper describes in sections 6.1.1,
     * 6.1.2, and 6.1.3?
     */
    /* 这是否是BPF+论文中描述的6.1.1、6.1.2和6.1.3节中的内容？ */
}
    # 遍历每个层级的节点，从第一层到最大层级
    for (i = 1; i <= maxlevel; ++i) {
        # 遍历当前层级的节点
        for (p = opt_state->levels[i]; p; p = p->link) {
            # 对当前节点的 true 分支进行优化
            opt_j(opt_state, &p->et);
            # 对当前节点的 false 分支进行优化
            opt_j(opt_state, &p->ef);
        }
    }

    # 查找每个节点的入边
    find_inedges(opt_state, ic->root);
    # 再次遍历每个层级的节点，从第一层到最大层级
    for (i = 1; i <= maxlevel; ++i) {
        # 遍历当前层级的节点
        for (p = opt_state->levels[i]; p; p = p->link) {
            # 对当前节点进行 OR 操作的优化
            or_pullup(opt_state, p);
            # 对当前节点进行 AND 操作的优化
            and_pullup(opt_state, p);
        }
    }
# 将父节点添加到子节点的入边列表中
static inline void
link_inedge(struct edge *parent, struct block *child)
{
    parent->next = child->in_edges;
    child->in_edges = parent;
}

# 查找每个节点的入边
static void
find_inedges(opt_state_t *opt_state, struct block *root)
{
    u_int i;
    int level;
    struct block *b;

    # 将所有节点的入边列表初始化为空
    for (i = 0; i < opt_state->n_blocks; ++i)
        opt_state->blocks[i]->in_edges = 0;

    """
    遍历图，将每条边添加到其后继节点的前驱列表中。
    跳过叶子节点（即级别为0的节点）。
    """
    for (level = root->level; level > 0; --level) {
        for (b = opt_state->levels[level]; b != 0; b = b->link) {
            link_inedge(&b->et, JT(b));
            link_inedge(&b->ef, JF(b));
        }
    }
}

# 优化根节点
static void
opt_root(struct block **b)
{
    struct slist *tmp, *s;

    s = (*b)->stmts;
    (*b)->stmts = 0;
    while (BPF_CLASS((*b)->s.code) == BPF_JMP && JT(*b) == JF(*b))
        *b = JT(*b);

    tmp = (*b)->stmts;
    if (tmp != 0)
        sappend(s, tmp);
    (*b)->stmts = s;

    """
    如果根节点是返回节点，则没有执行任何语句的意义
    （因为bpf机器没有副作用）。
    """
    if (BPF_CLASS((*b)->s.code) == BPF_RET)
        (*b)->stmts = 0;
}

# 优化循环
static void
opt_loop(opt_state_t *opt_state, struct icode *ic, int do_stmts)
{

#ifdef BDEBUG
    if (pcap_optimizer_debug > 1 || pcap_print_dot_graph) {
        printf("opt_loop(root, %d) begin\n", do_stmts);
        opt_dump(opt_state, ic);
    }
#endif

    """
    优化器循环检测。
    """
    int loop_count = 0;
    for (;;) {
        opt_state->done = 1;
        """
        优化器循环检测。
        """
        opt_state->non_branch_movement_performed = 0;
        find_levels(opt_state, ic);
        find_dom(opt_state, ic->root);
        find_closure(opt_state, ic->root);
        find_ud(opt_state, ic->root);
        find_edom(opt_state, ic->root);
        opt_blks(opt_state, ic, do_stmts);
#ifdef BDEBUG
        // 如果定义了 BDEBUG 宏，并且调试级别大于1或者需要打印图形表示，则输出调试信息
        if (pcap_optimizer_debug > 1 || pcap_print_dot_graph) {
            printf("opt_loop(root, %d) bottom, done=%d\n", do_stmts, opt_state->done);
            opt_dump(opt_state, ic);
        }
#endif

        /*
         * 在这个优化器过程中是否有任何操作？
         */
        if (opt_state->done) {
            /*
             * 没有，所以我们已经达到了一个固定点。
             * 我们完成了。
             */
            break;
        }

        /*
         * XXX - 在这个过程中除了分支移动之外还有其他操作吗？
         */
        if (opt_state->non_branch_movement_performed) {
            /*
             * 是的。清除任何循环检测计数器；
             * 我们正在取得某种形式的进展（假设我们不能陷入循环做*其他*优化...）。
             */
            loop_count = 0;
        } else {
            /*
             * 没有 - 增加计数器，并在达到100时退出。
             */
            loop_count++;
            if (loop_count >= 100) {
                /*
                 * 我们已经做了100次分支移动以外的事情；我们可能
                 * 处于一个循环中，永远不会达到一个固定点。
                 *
                 * XXX - 是的，我们真的需要一种非启发式的方法来检测循环。
                 */
                opt_state->done = 1;
                break;
            }
        }
    }
}

/*
 * 在其dag表示中优化过滤器代码。
 * 成功返回0，错误返回-1。
 */
int
bpf_optimize(struct icode *ic, char *errbuf)
{
    opt_state_t opt_state;

    memset(&opt_state, 0, sizeof(opt_state));
    opt_state.errbuf = errbuf;
    opt_state.non_branch_movement_performed = 0;
    if (setjmp(opt_state.top_ctx)) {
        opt_cleanup(&opt_state);
        return -1;
    }
    opt_init(&opt_state, ic);
    # 调用 opt_loop 函数，传入 opt_state、ic 和 0 作为参数
    opt_loop(&opt_state, ic, 0);
    # 再次调用 opt_loop 函数，传入 opt_state、ic 和 1 作为参数
    opt_loop(&opt_state, ic, 1);
    # 调用 intern_blocks 函数，传入 opt_state 和 ic 作为参数
    intern_blocks(&opt_state, ic);
#ifdef BDEBUG
    // 如果定义了 BDEBUG 宏，并且 pcap_optimizer_debug 大于 1 或者 pcap_print_dot_graph 为真，则打印提示信息和优化状态
    if (pcap_optimizer_debug > 1 || pcap_print_dot_graph) {
        printf("after intern_blocks()\n");
        opt_dump(&opt_state, ic);
    }
#endif
    // 对代码块进行优化
    opt_root(&ic->root);
#ifdef BDEBUG
    // 如果定义了 BDEBUG 宏，并且 pcap_optimizer_debug 大于 1 或者 pcap_print_dot_graph 为真，则打印提示信息和优化状态
    if (pcap_optimizer_debug > 1 || pcap_print_dot_graph) {
        printf("after opt_root()\n");
        opt_dump(&opt_state, ic);
    }
#endif
    // 清理优化状态
    opt_cleanup(&opt_state);
    // 返回 0 表示成功
    return 0;
}

// 标记指定代码块中的节点
static void
make_marks(struct icode *ic, struct block *p)
{
    // 如果代码块中的节点未被标记，则进行标记
    if (!isMarked(ic, p)) {
        Mark(ic, p);
        // 如果当前节点的操作码不是 BPF_RET，则递归标记其子节点
        if (BPF_CLASS(p->s.code) != BPF_RET) {
            make_marks(ic, JT(p));
            make_marks(ic, JF(p));
        }
    }
}

/*
 * 标记代码数组，使得 isMarked(ic->cur_mark, i) 仅对存活的节点返回 true
 */
static void
mark_code(struct icode *ic)
{
    // 当前标记值加一
    ic->cur_mark += 1;
    // 从根节点开始标记代码块
    make_marks(ic, ic->root);
}

/*
 * 判断两个语句列表是否从数据包中加载相同的值到累加器中
 */
static int
eq_slist(struct slist *x, struct slist *y)
{
    for (;;) {
        // 跳过 NOP 操作码
        while (x && x->s.code == NOP)
            x = x->next;
        while (y && y->s.code == NOP)
            y = y->next;
        // 如果其中一个列表为空，则返回另一个列表是否为空的结果
        if (x == 0)
            return y == 0;
        if (y == 0)
            return x == 0;
        // 如果两个语句的操作码或者参数不相等，则返回 false
        if (x->s.code != y->s.code || x->s.k != y->s.k)
            return 0;
        x = x->next;
        y = y->next;
    }
}

// 判断两个代码块是否相等
static inline int
eq_blk(struct block *b0, struct block *b1)
{
    // 如果两个代码块的操作码、参数、true 分支和 false 分支相等，则比较语句列表是否相等
    if (b0->s.code == b1->s.code &&
        b0->s.k == b1->s.k &&
        b0->et.succ == b1->et.succ &&
        b0->ef.succ == b1->ef.succ)
        return eq_slist(b0->stmts, b1->stmts);
    return 0;
}

// 对代码块进行内部优化
static void
intern_blocks(opt_state_t *opt_state, struct icode *ic)
{
    struct block *p;
    u_int i, j;
    int done1; /* 不要遮蔽全局变量 */
 top:
    done1 = 1;
    // 将所有代码块的链接置为 0
    for (i = 0; i < opt_state->n_blocks; ++i)
        opt_state->blocks[i]->link = 0;

    // 标记代码块
    mark_code(ic);
    # 从最后一个块开始向前遍历
    for (i = opt_state->n_blocks - 1; i != 0; ) {
        # 减小 i 的值
        --i;
        # 如果块未标记，则跳过
        if (!isMarked(ic, opt_state->blocks[i]))
            continue;
        # 从当前块的下一个块开始向后遍历
        for (j = i + 1; j < opt_state->n_blocks; ++j) {
            # 如果块未标记，则跳过
            if (!isMarked(ic, opt_state->blocks[j]))
                continue;
            # 如果两个块相等
            if (eq_blk(opt_state->blocks[i], opt_state->blocks[j])) {
                # 将当前块的链接指向下一个块的链接，如果下一个块的链接为空，则指向下一个块
                opt_state->blocks[i]->link = opt_state->blocks[j]->link ?
                    opt_state->blocks[j]->link : opt_state->blocks[j];
                # 跳出循环
                break;
            }
        }
    }
    # 从第一个块开始向后遍历
    for (i = 0; i < opt_state->n_blocks; ++i) {
        # 获取当前块
        p = opt_state->blocks[i];
        # 如果当前块的 JT 域为 0，则跳过
        if (JT(p) == 0)
            continue;
        # 如果当前块的 JT 域的链接存在
        if (JT(p)->link) {
            # 将 done1 设为 0
            done1 = 0;
            # 将当前块的 JT 域指向其链接
            JT(p) = JT(p)->link;
        }
        # 如果当前块的 JF 域的链接存在
        if (JF(p)->link) {
            # 将 done1 设为 0
            done1 = 0;
            # 将当前块的 JF 域指向其链接
            JF(p) = JF(p)->link;
        }
    }
    # 如果 done1 为假，则跳转到 top 标签处
    if (!done1)
        goto top;
}

static void
opt_cleanup(opt_state_t *opt_state)
{
    // 释放优化状态结构体中的内存空间
    free((void *)opt_state->vnode_base);
    free((void *)opt_state->vmap);
    free((void *)opt_state->edges);
    free((void *)opt_state->space);
    free((void *)opt_state->levels);
    free((void *)opt_state->blocks);
}

/*
 * 用于优化器错误处理。
 */
static void PCAP_NORETURN
opt_error(opt_state_t *opt_state, const char *fmt, ...)
{
    va_list ap;

    // 如果错误缓冲区不为空，则格式化错误信息并存储到错误缓冲区中
    if (opt_state->errbuf != NULL) {
        va_start(ap, fmt);
        (void)vsnprintf(opt_state->errbuf,
            PCAP_ERRBUF_SIZE, fmt, ap);
        va_end(ap);
    }
    // 跳转到错误处理的上下文
    longjmp(opt_state->top_ctx, 1);
    /* NOTREACHED */
#ifdef _AIX
    PCAP_UNREACHABLE
#endif /* _AIX */
}

/*
 * 返回's'中的语句数。
 */
static u_int
slength(struct slist *s)
{
    u_int n = 0;

    for (; s; s = s->next)
        // 如果语句的代码不是NOP，则计数加一
        if (s->s.code != NOP)
            ++n;
    return n;
}

/*
 * 返回由'p'可达的节点数。
 * 所有节点应该最初都是未标记的。
 */
static int
count_blocks(struct icode *ic, struct block *p)
{
    // 如果p为0或已标记，则返回0
    if (p == 0 || isMarked(ic, p))
        return 0;
    // 标记p
    Mark(ic, p);
    // 递归计算由JT(p)和JF(p)可达的节点数，并加上1
    return count_blocks(ic, JT(p)) + count_blocks(ic, JF(p)) + 1;
}

/*
 * 对流图进行深度优先搜索，对基本块进行编号，并将它们输入到'blocks'数组中。
 */
static void
number_blks_r(opt_state_t *opt_state, struct icode *ic, struct block *p)
{
    u_int n;

    // 如果p为0或已标记，则返回
    if (p == 0 || isMarked(ic, p))
        return;

    // 标记p
    Mark(ic, p);
    n = opt_state->n_blocks++;
    // 如果块数达到最大值，则报错
    if (opt_state->n_blocks == 0) {
        /*
         * Overflow.
         */
        opt_error(opt_state, "filter is too complex to optimize");
    }
    // 给p分配一个id，并将其存储到blocks数组中
    p->id = n;
    opt_state->blocks[n] = p;

    // 递归处理JT(p)和JF(p)
    number_blks_r(opt_state, ic, JT(p));
    number_blks_r(opt_state, ic, JF(p));
}
/*
 * 返回流图中由 'p' 可达的语句数。
 * 在调用之前，节点应该是未标记的。
 *
 * 注意，“stmts”指的是“指令”，这包括
 *
 *    'p' 中的副作用语句 (slength(p->stmts));
 *
 *    'p' 的真分支中的语句 (count_stmts(JT(p)));
 *
 *    'p' 的假分支中的语句 (count_stmts(JF(p)));
 *
 *    条件跳转本身 (1);
 *
 *    如果真分支需要，额外的长跳转 (p->longjt);
 *
 *    如果假分支需要，额外的长跳转 (p->longjf).
 */
static u_int
count_stmts(struct icode *ic, struct block *p)
{
    u_int n;

    if (p == 0 || isMarked(ic, p))
        return 0;
    Mark(ic, p);
    n = count_stmts(ic, JT(p)) + count_stmts(ic, JF(p));
    return slength(p->stmts) + n + 1 + p->longjt + p->longjf;
}

/*
 * 分配内存。所有分配都在优化开始之前完成。从块和/或语句的总数计算所有数据结构大小的线性界限。
 */
static void
opt_init(opt_state_t *opt_state, struct icode *ic)
{
    bpf_u_int32 *p;
    int i, n, max_stmts;
    u_int product;
    size_t block_memsize, edge_memsize;

    /*
     * 首先，计算块的数量，这样我们就可以分配一个数组来映射块号到块。
     * 然后，将块放入数组中。
     */
    unMarkAll(ic);
    n = count_blocks(ic, ic->root);
    opt_state->blocks = (struct block **)calloc(n, sizeof(*opt_state->blocks));
    if (opt_state->blocks == NULL)
        opt_error(opt_state, "malloc");
    unMarkAll(ic);
    opt_state->n_blocks = 0;
    number_blks_r(opt_state, ic, ic->root);

    /*
     * 这“不应该发生”。
     */
    if (opt_state->n_blocks == 0)
        opt_error(opt_state, "filter has no instructions; please report this as a libpcap issue");

    opt_state->n_edges = 2 * opt_state->n_blocks;
}
    # 如果边的数量除以2不等于块的数量，表示溢出，输出错误信息
    if ((opt_state->n_edges / 2) != opt_state->n_blocks) {
        /*
         * Overflow.
         */
        opt_error(opt_state, "filter is too complex to optimize");
    }
    # 为边分配内存空间
    opt_state->edges = (struct edge **)calloc(opt_state->n_edges, sizeof(*opt_state->edges));
    # 如果内存分配失败，输出错误信息
    if (opt_state->edges == NULL) {
        opt_error(opt_state, "malloc");
    }

    /*
     * The number of levels is bounded by the number of nodes.
     */
    # 为块分配内存空间
    opt_state->levels = (struct block **)calloc(opt_state->n_blocks, sizeof(*opt_state->levels));
    # 如果内存分配失败，输出错误信息
    if (opt_state->levels == NULL) {
        opt_error(opt_state, "malloc");
    }

    # 计算边所需的字数和块所需的字数
    opt_state->edgewords = opt_state->n_edges / BITS_PER_WORD + 1;
    opt_state->nodewords = opt_state->n_blocks / BITS_PER_WORD + 1;

    /*
     * Make sure opt_state->n_blocks * opt_state->nodewords fits
     * in a u_int; we use it as a u_int number-of-iterations
     * value.
     */
    # 确保 opt_state->n_blocks * opt_state->nodewords 可以放入 u_int 中，用作迭代次数
    product = opt_state->n_blocks * opt_state->nodewords;
    # 如果不符合条件，输出错误信息
    if ((product / opt_state->n_blocks) != opt_state->nodewords) {
        /*
         * XXX - just punt and don't try to optimize?
         * In practice, this is unlikely to happen with
         * a normal filter.
         */
        opt_error(opt_state, "filter is too complex to optimize");
    }

    /*
     * Make sure the total memory required for that doesn't
     * overflow.
     */
    # 计算块内存大小，并确保不会溢出
    block_memsize = (size_t)2 * product * sizeof(*opt_state->space);
    # 如果不符合条件，输出错误信息
    if ((block_memsize / product) != 2 * sizeof(*opt_state->space)) {
        opt_error(opt_state, "filter is too complex to optimize");
    }

    /*
     * Make sure opt_state->n_edges * opt_state->edgewords fits
     * in a u_int; we use it as a u_int number-of-iterations
     * value.
     */
    # 确保 opt_state->n_edges * opt_state->edgewords 可以放入 u_int 中，用作迭代次数
    product = opt_state->n_edges * opt_state->edgewords;
    # 如果不符合条件，输出错误信息
    if ((product / opt_state->n_edges) != opt_state->edgewords) {
        opt_error(opt_state, "filter is too complex to optimize");
    }
    /*
     * 确保所需的总内存不会溢出。
     */
    edge_memsize = (size_t)product * sizeof(*opt_state->space);
    if (edge_memsize / product != sizeof(*opt_state->space)) {
        opt_error(opt_state, "filter is too complex to optimize");
    }

    /*
     * 确保两者所需的总内存不会溢出。
     */
    if (block_memsize > SIZE_MAX - edge_memsize) {
        opt_error(opt_state, "filter is too complex to optimize");
    }

    /* 分配空间给 opt_state->space，并进行错误检查 */
    opt_state->space = (bpf_u_int32 *)malloc(block_memsize + edge_memsize);
    if (opt_state->space == NULL) {
        opt_error(opt_state, "malloc");
    }
    p = opt_state->space;
    opt_state->all_dom_sets = p;
    for (i = 0; i < n; ++i) {
        opt_state->blocks[i]->dom = p;
        p += opt_state->nodewords;
    }
    opt_state->all_closure_sets = p;
    for (i = 0; i < n; ++i) {
        opt_state->blocks[i]->closure = p;
        p += opt_state->nodewords;
    }
    opt_state->all_edge_sets = p;
    for (i = 0; i < n; ++i) {
        register struct block *b = opt_state->blocks[i];

        b->et.edom = p;
        p += opt_state->edgewords;
        b->ef.edom = p;
        p += opt_state->edgewords;
        b->et.id = i;
        opt_state->edges[i] = &b->et;
        b->ef.id = opt_state->n_blocks + i;
        opt_state->edges[opt_state->n_blocks + i] = &b->ef;
        b->et.pred = b;
        b->ef.pred = b;
    }
    max_stmts = 0;
    for (i = 0; i < n; ++i)
        max_stmts += slength(opt_state->blocks[i]->stmts) + 1;
    /*
     * 每条语句最多分配 3 个值编号，因此这是我们所需的 valnodes 的上限。
     */
    opt_state->maxval = 3 * max_stmts;
    opt_state->vmap = (struct vmapinfo *)calloc(opt_state->maxval, sizeof(*opt_state->vmap));
    if (opt_state->vmap == NULL) {
        opt_error(opt_state, "malloc");
    }
    # 为 opt_state 结构体中的 vnode_base 成员分配内存空间，大小为 maxval 乘以 vnode_base 结构体的大小
    opt_state->vnode_base = (struct valnode *)calloc(opt_state->maxval, sizeof(*opt_state->vnode_base));
    # 如果分配内存失败，则输出错误信息
    if (opt_state->vnode_base == NULL) {
        opt_error(opt_state, "malloc");
    }
}

/*
 * This is only used when supporting optimizer debugging. It is
 * global state, so do *not* do more than one compile in parallel
 * and expect it to provide meaningful information.
 */
#ifdef BDEBUG
int bids[NBIDS];
#endif

static void PCAP_NORETURN conv_error(conv_state_t *, const char *, ...)
    PCAP_PRINTFLIKE(2, 3);

/*
 * Returns true if successful.  Returns false if a branch has
 * an offset that is too large.  If so, we have marked that
 * branch so that on a subsequent iteration, it will be treated
 * properly.
 */
static int
convert_code_r(conv_state_t *conv_state, struct icode *ic, struct block *p)
{
    struct bpf_insn *dst;  // 指向目标 BPF 指令的指针
    struct slist *src;  // 指向源 slist 的指针
    u_int slen;  // slist 的长度
    u_int off;  // 偏移量
    struct slist **offset = NULL;  // slist 的偏移量数组

    if (p == 0 || isMarked(ic, p))  // 如果 p 为 0 或者已经标记过了
        return (1);  // 返回 1
    Mark(ic, p);  // 标记 p

    if (convert_code_r(conv_state, ic, JF(p)) == 0)  // 如果 JF(p) 转换失败
        return (0);  // 返回 0
    if (convert_code_r(conv_state, ic, JT(p)) == 0)  // 如果 JT(p) 转换失败
        return (0);  // 返回 0

    slen = slength(p->stmts);  // 获取 p->stmts 的长度
    dst = conv_state->ftail -= (slen + 1 + p->longjt + p->longjf);
        /* inflate length by any extra jumps */
        // 根据额外的跳转扩展长度

    p->offset = (int)(dst - conv_state->fstart);  // 计算偏移量

    /* generate offset[] for convenience  */
    if (slen) {  // 如果长度不为 0
        offset = (struct slist **)calloc(slen, sizeof(struct slist *));  // 分配内存
        if (!offset) {  // 如果分配失败
            conv_error(conv_state, "not enough core");  // 报错
            /*NOTREACHED*/
        }
    }
    src = p->stmts;  // 指向 p->stmts
    for (off = 0; off < slen && src; off++) {  // 遍历 p->stmts
#if 0
        printf("off=%d src=%x\n", off, src);
#endif
        offset[off] = src;  // 将 src 存入 offset
        src = src->next;  // 指向下一个
    }

    off = 0;  // 偏移量初始化为 0
    for (src = p->stmts; src; src = src->next) {  // 遍历 p->stmts
        if (src->s.code == NOP)  // 如果指令为 NOP
            continue;  // 继续下一次循环
        dst->code = (u_short)src->s.code;  // 设置目标指令的指令码
        dst->k = src->s.k;  // 设置目标指令的 k 值

        /* fill block-local relative jump */
        if (BPF_CLASS(src->s.code) != BPF_JMP || src->s.code == (BPF_JMP|BPF_JA)) {  // 如果不是跳转指令或者是无条件跳转指令
        # 如果源码的跳转目标不为空，则释放偏移量内存，报告非法跳转目标错误
        if (src->s.jt || src->s.jf) {
            free(offset);
            conv_error(conv_state, "illegal jmp destination");
            /*NOTREACHED*/
        }
        # 跳转到 filled 标签处
        goto filled;

        # 如果 off 等于 slen - 2，则跳转到 filled 标签处
        if (off == slen - 2)    /*???*/
            goto filled;

        {
        u_int i;
        int jt, jf;
        const char ljerr[] = "%s for block-local relative jump: off=%d";

        # 如果定义了宏 0，则打印源码的指令、偏移量、跳转目标 jt 和 jf
        printf("code=%x off=%d %x %x\n", src->s.code,
            off, src->s.jt, src->s.jf);

        # 如果源码的跳转目标 jt 或 jf 为空，则释放偏移量内存，报告无跳转目标错误
        if (!src->s.jt || !src->s.jf) {
            free(offset);
            conv_error(conv_state, ljerr, "no jmp destination", off);
            /*NOTREACHED*/
        }

        jt = jf = 0;
        # 遍历偏移量数组
        for (i = 0; i < slen; i++) {
            # 如果偏移量数组中的值等于源码的跳转目标 jt
            if (offset[i] == src->s.jt) {
                # 如果 jt 已经被设置过，则释放偏移量内存，报告多个匹配错误
                if (jt) {
                    free(offset);
                    conv_error(conv_state, ljerr, "multiple matches", off);
                    /*NOTREACHED*/
                }

                # 如果跳转偏移量超出范围，则释放偏移量内存，报告跳转超出范围错误
                if (i - off - 1 >= 256) {
                    free(offset);
                    conv_error(conv_state, ljerr, "out-of-range jump", off);
                    /*NOTREACHED*/
                }
                # 设置目标的跳转偏移量为 i - off - 1
                dst->jt = (u_char)(i - off - 1);
                jt++;
            }
            # 如果偏移量数组中的值等于源码的跳转目标 jf
            if (offset[i] == src->s.jf) {
                # 如果 jf 已经被设置过，则释放偏移量内存，报告多个匹配错误
                if (jf) {
                    free(offset);
                    conv_error(conv_state, ljerr, "multiple matches", off);
                    /*NOTREACHED*/
                }
                # 如果跳转偏移量超出范围，则释放偏移量内存，报告跳转超出范围错误
                if (i - off - 1 >= 256) {
                    free(offset);
                    conv_error(conv_state, ljerr, "out-of-range jump", off);
                    /*NOTREACHED*/
                }
                # 设置目标的跳转偏移量为 i - off - 1
                dst->jf = (u_char)(i - off - 1);
                jf++;
            }
        }
        # 如果 jt 或 jf 为空，则释放偏移量内存，报告未找到目标错误
        if (!jt || !jf) {
            free(offset);
            conv_error(conv_state, ljerr, "no destination found", off);
            /*NOTREACHED*/
        }
        }
    filled:
        // 填充指令，将目标指针和偏移指针分别向前移动一位
        ++dst;
        ++off;
    }
    // 如果偏移指针不为空，则释放其内存
    if (offset)
        free(offset);

#ifdef BDEBUG
    // 如果定义了 BDEBUG 宏，则将指令的 ID 加 1 存入 bids 数组
    if (dst - conv_state->fstart < NBIDS)
        bids[dst - conv_state->fstart] = p->id + 1;
#endif
    // 将目标指令的 code 字段设置为 p 指令的 s.code
    dst->code = (u_short)p->s.code;
    // 将目标指令的 k 字段设置为 p 指令的 s.k
    dst->k = p->s.k;
    // 如果 p 指令有 JT 字段
    if (JT(p)) {
        /* 插入的额外跳转次数 */
        u_char extrajmps = 0;
        // 计算跳转偏移量 off
        off = JT(p)->offset - (p->offset + slen) - 1;
        // 如果 off 大于等于 256
        if (off >= 256) {
            /* 偏移量过大，需要添加一个跳转指令 */
            if (p->longjt == 0) {
                /* 标记此指令并重试 */
                p->longjt++;
                return(0);
            }
            // 设置目标指令的 jt 字段为 extrajmps
            dst->jt = extrajmps;
            extrajmps++;
            // 设置下一个指令的 code 字段为 BPF_JMP|BPF_JA
            dst[extrajmps].code = BPF_JMP|BPF_JA;
            // 设置下一个指令的 k 字段为 off - extrajmps
            dst[extrajmps].k = off - extrajmps;
        }
        else
            // 设置目标指令的 jt 字段为 off
            dst->jt = (u_char)off;
        // 计算跳转偏移量 off
        off = JF(p)->offset - (p->offset + slen) - 1;
        // 如果 off 大于等于 256
        if (off >= 256) {
            /* 偏移量过大，需要添加一个跳转指令 */
            if (p->longjf == 0) {
                /* 标记此指令并重试 */
                p->longjf++;
                return(0);
            }
            /* F 跳转到下一个跳转指令 */
            /* 如果插入了两个跳转指令，F 跳转到第二个 */
            // 设置目标指令的 jf 字段为 extrajmps
            dst->jf = extrajmps;
            extrajmps++;
            // 设置下一个指令的 code 字段为 BPF_JMP|BPF_JA
            dst[extrajmps].code = BPF_JMP|BPF_JA;
            // 设置下一个指令的 k 字段为 off - extrajmps
            dst[extrajmps].k = off - extrajmps;
        }
        else
            // 设置目标指令的 jf 字段为 off
            dst->jf = (u_char)off;
    }
    // 返回 1
    return (1);
}
/*
 * 将流图中间表示转换为BPF数组表示。将*lenp设置为指令的数量。
 *
 * 这个例程不会泄漏fp指向的内存。在返回fp之前，它*不能*释放fp；这样做是毫无意义的，
 * 因为icode_to_fcode()的返回值指向的BPF数组必须是有效的 - 它被返回以在bpf_program结构中使用。
 *
 * 如果看起来icode_to_fcode()在泄漏，问题可能是使用pcap_compile()的程序在完成时未释放BPF程序中的内存 - 泄漏在程序中，而不是在分配内存的例程中。
 * （类似地，如果一个程序调用fopen()而从未在FILE *上调用fclose()，它将泄漏FILE结构；泄漏不在fopen()中，而是在程序中。）更改程序以在完成过滤程序时使用pcap_freecode()。参见pcap手册页。
 */
struct bpf_insn *
icode_to_fcode(struct icode *ic, struct block *root, u_int *lenp,
    char *errbuf)
{
    u_int n;
    struct bpf_insn *fp;
    conv_state_t conv_state;

    conv_state.fstart = NULL;
    conv_state.errbuf = errbuf;
    if (setjmp(conv_state.top_ctx) != 0) {
        free(conv_state.fstart);
        return NULL;
    }

    /*
     * 循环执行convert_code_r()，直到没有剩余具有太大偏移的分支。
     */
    for (;;) {
        unMarkAll(ic);
        n = *lenp = count_stmts(ic, root);

        fp = (struct bpf_insn *)malloc(sizeof(*fp) * n);
        if (fp == NULL) {
        (void)snprintf(errbuf, PCAP_ERRBUF_SIZE,
            "malloc");
        return NULL;
        }
        memset((char *)fp, 0, sizeof(*fp) * n);
        conv_state.fstart = fp;
        conv_state.ftail = fp + n;

        unMarkAll(ic);
        if (convert_code_r(&conv_state, ic, root))
        break;
        free(fp);
    }

    return fp;
}

/*
 * 用于iconv_to_fconv()的错误。
 */
# 定义一个函数，用于处理转换错误，函数名为 PCAP_NORETURN
def conv_error(conv_state_t *conv_state, const char *fmt, ...):
    # 声明一个变量列表
    va_list ap
    # 初始化变量列表
    va_start(ap, fmt)
    # 将格式化的字符串写入 conv_state->errbuf 中
    (void)vsnprintf(conv_state->errbuf, PCAP_ERRBUF_SIZE, fmt, ap)
    # 结束变量列表的使用
    va_end(ap)
    # 跳转到 conv_state->top_ctx 所指向的位置
    longjmp(conv_state->top_ctx, 1)
    # 不会执行到这里
    /* NOTREACHED */
    # 如果是在 AIX 平台下
    # 则执行 PCAP_UNREACHABLE
    #endif /* _AIX */

# 定义一个函数，用于安装 BPF 程序到 pcap_t 的 fcode 成员中
int install_bpf_program(pcap_t *p, struct bpf_program *fp):
    # 声明一个变量，用于存储程序的大小
    size_t prog_size
    # 验证程序是否有效
    if (!pcap_validate_filter(fp->bf_insns, fp->bf_len)):
        # 如果程序无效，则将错误信息写入 p->errbuf 中
        snprintf(p->errbuf, sizeof(p->errbuf), "BPF program is not valid")
        # 返回 -1
        return (-1)
    # 释放已安装的程序
    pcap_freecode(&p->fcode)
    # 计算程序的大小
    prog_size = sizeof(*fp->bf_insns) * fp->bf_len
    # 将程序的长度赋值给 p->fcode.bf_len
    p->fcode.bf_len = fp->bf_len
    # 分配内存空间给 p->fcode.bf_insns
    p->fcode.bf_insns = (struct bpf_insn *)malloc(prog_size)
    # 如果分配内存失败
    if (p->fcode.bf_insns == NULL):
        # 将错误信息写入 p->errbuf 中
        pcap_fmt_errmsg_for_errno(p->errbuf, sizeof(p->errbuf), errno, "malloc")
        # 返回 -1
        return (-1)
    # 将 fp->bf_insns 中的内容复制到 p->fcode.bf_insns 中
    memcpy(p->fcode.bf_insns, fp->bf_insns, prog_size)
    # 返回 0
    return (0)

# 如果定义了 BDEBUG
# 则定义一个函数，用于将节点信息输出到文件中
static void dot_dump_node(struct icode *ic, struct block *block, struct bpf_program *prog, FILE *out):
    # 声明变量
    int icount, noffset
    int i
    # 如果 block 为空或者已标记过
    if (block == NULL || isMarked(ic, block)):
        # 直接返回
        return
    # 标记 block
    Mark(ic, block)
    # 计算语句数量和偏移量
    icount = slength(block->stmts) + 1 + block->longjt + block->longjf
    noffset = min(block->offset + icount, (int)prog->bf_len)
    # 将节点信息输出到文件中
    fprintf(out, "\tblock%u [shape=ellipse, id=\"block-%u\" label=\"BLOCK%u\\n", block->id, block->id, block->id)
    for (i = block->offset; i < noffset; i++):
        fprintf(out, "\\n%s", bpf_image(prog->bf_insns + i, i))
    # 将字符串写入输出文件，用于显示工具提示
    fprintf(out, "\" tooltip=\"");
    # 遍历块的值数组，如果不是未知值，则将值和索引写入输出文件
    for (i = 0; i < BPF_MEMWORDS; i++)
        if (block->val[i] != VAL_UNKNOWN)
            fprintf(out, "val[%d]=%d ", i, block->val[i]);
    # 将特定索引的值写入输出文件
    fprintf(out, "val[A]=%d ", block->val[A_ATOM]);
    fprintf(out, "val[X]=%d", block->val[X_ATOM]);
    # 写入结束引号
    fprintf(out, "\"");
    # 如果块的 JT 指针为空，则在输出文件中添加 peripheries=2
    if (JT(block) == NULL)
        fprintf(out, ", peripheries=2");
    # 写入结束括号和分号
    fprintf(out, "];\n");

    # 调用 dot_dump_node 函数，将块的 JT 指针指向的节点转换为 DOT 语言并写入输出文件
    dot_dump_node(ic, JT(block), prog, out);
    # 调用 dot_dump_node 函数，将块的 JF 指针指向的节点转换为 DOT 语言并写入输出文件
    dot_dump_node(ic, JF(block), prog, out);
# 定义一个静态函数，用于输出图形化的控制流图
static void
dot_dump_edge(struct icode *ic, struct block *block, FILE *out)
{
    # 如果块为空或者已经标记过，则返回
    if (block == NULL || isMarked(ic, block))
        return;
    # 标记当前块
    Mark(ic, block);

    # 如果当前块有真分支和假分支
    if (JT(block)) {
        # 输出真分支的连接关系
        fprintf(out, "\t\"block%u\":se -> \"block%u\":n [label=\"T\"]; \n",
                block->id, JT(block)->id);
        # 输出假分支的连接关系
        fprintf(out, "\t\"block%u\":sw -> \"block%u\":n [label=\"F\"]; \n",
               block->id, JF(block)->id);
    }
    # 递归输出真分支的连接关系
    dot_dump_edge(ic, JT(block), out);
    # 递归输出假分支的连接关系
    dot_dump_edge(ic, JF(block), out);
}

/* 用 graphviz/DOT 语言输出控制流图
 * 在控制流图中，显示块的代码，退出时每个寄存器的值索引，以及跳转关系
 *
 * 例如 BPF `ip src host 1.1.1.1' 的 DOT 示例为:
    digraph BPF {
    block0 [shape=ellipse, id="block-0" label="BLOCK0\n\n(000) ldh      [12]\n(001) jeq      #0x800           jt 2    jf 5" tooltip="val[A]=0 val[X]=0"];
    block1 [shape=ellipse, id="block-1" label="BLOCK1\n\n(002) ld       [26]\n(003) jeq      #0x1010101       jt 4    jf 5" tooltip="val[A]=0 val[X]=0"];
    block2 [shape=ellipse, id="block-2" label="BLOCK2\n\n(004) ret      #68" tooltip="val[A]=0 val[X]=0", peripheries=2];
    block3 [shape=ellipse, id="block-3" label="BLOCK3\n\n(005) ret      #0" tooltip="val[A]=0 val[X]=0", peripheries=2];
    "block0":se -> "block1":n [label="T"];
    "block0":sw -> "block3":n [label="F"];
    "block1":se -> "block2":n [label="T"];
    "block1":sw -> "block3":n [label="F"];
    }
 *
 * 安装 graphviz 在 https://www.graphviz.org/ 后，将其保存为 bpf.dot
 * 运行 `dot -Tpng -O bpf.dot' 以绘制图形
 */
static int
dot_dump(struct icode *ic, char *errbuf)
{
    struct bpf_program f;
    FILE *out = stdout;

    # 将 bids 数组初始化为 0
    memset(bids, 0, sizeof bids);
    # 将 icode 转换为 fcode，并将结果赋给 f.bf_insns
    f.bf_insns = icode_to_fcode(ic, ic->root, &f.bf_len, errbuf);
    # 如果转换失败，则返回 -1
    if (f.bf_insns == NULL)
        return -1;

    # 输出图形化的控制流图的开始部分
    fprintf(out, "digraph BPF {\n");
    # 取消所有块的标记
    unMarkAll(ic);
    # 输出控制流图的节点和边
    dot_dump_node(ic, ic->root, &f, out);
    # 取消所有块的标记
    unMarkAll(ic);
    # 将图形化表示的控制流图输出到文件中
    dot_dump_edge(ic, ic->root, out);
    # 向输出文件中写入结束语句
    fprintf(out, "}\n");

    # 释放动态分配的指令缓冲区内存
    free((char *)f.bf_insns);
    # 返回 0 表示程序正常结束
    return 0;
# 定义一个静态函数，用于将中间代码转换为 BPF 字节码并输出
static int
plain_dump(struct icode *ic, char *errbuf)
{
    # 定义一个 BPF 程序结构体
    struct bpf_program f;

    # 将 bids 数组清零
    memset(bids, 0, sizeof bids);
    # 将中间代码转换为 BPF 字节码
    f.bf_insns = icode_to_fcode(ic, ic->root, &f.bf_len, errbuf);
    # 如果转换失败，则返回 -1
    if (f.bf_insns == NULL)
        return -1;
    # 输出 BPF 字节码
    bpf_dump(&f, 1);
    # 输出一个换行符
    putchar('\n');
    # 释放 BPF 字节码内存
    free((char *)f.bf_insns);
    # 返回成功
    return 0;
}

# 定义一个静态函数，用于根据配置输出优化后的中间代码或 DOT 格式的 CFG 图
static void
opt_dump(opt_state_t *opt_state, struct icode *ic)
{
    # 定义变量
    int status;
    char errbuf[PCAP_ERRBUF_SIZE];

    # 如果需要输出 DOT 格式的 CFG 图，则调用 dot_dump 函数
    if (pcap_print_dot_graph)
        status = dot_dump(ic, errbuf);
    # 否则调用 plain_dump 函数
    else
        status = plain_dump(ic, errbuf);
    # 如果输出失败，则输出错误信息
    if (status == -1)
        opt_error(opt_state, "opt_dump: icode_to_fcode failed: %s", errbuf);
}
#endif
```