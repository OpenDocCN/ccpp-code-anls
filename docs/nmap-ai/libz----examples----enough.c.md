# `nmap\libz\examples\enough.c`

```
/* enough.c -- determine the maximum size of inflate's Huffman code tables over
 * all possible valid and complete prefix codes, subject to a length limit.
 * Copyright (C) 2007, 2008, 2012, 2018 Mark Adler
 * Version 1.5  5 August 2018  Mark Adler
 */
// 确定在长度限制的情况下，对所有可能的有效和完整的前缀代码，确定inflate的Huffman编码表的最大大小
// 版权所有 (C) 2007, 2008, 2012, 2018 Mark Adler
// 版本 1.5  2018年8月5日  Mark Adler

/* Version history:
   1.0   3 Jan 2007  First version (derived from codecount.c version 1.4)
   1.1   4 Jan 2007  Use faster incremental table usage computation
                     Prune examine() search on previously visited states
   1.2   5 Jan 2007  Comments clean up
                     As inflate does, decrease root for short codes
                     Refuse cases where inflate would increase root
   1.3  17 Feb 2008  Add argument for initial root table size
                     Fix bug for initial root table size == max - 1
                     Use a macro to compute the history index
   1.4  18 Aug 2012  Avoid shifts more than bits in type (caused endless loop!)
                     Clean up comparisons of different types
                     Clean up code indentation
   1.5   5 Aug 2018  Clean up code style, formatting, and comments
                     Show all the codes for the maximum, and only the maximum
 */
// 版本历史：
// 1.0   2007年1月3日  第一个版本 (派生自codecount.c版本1.4)
// 1.1   2007年1月4日  使用更快的增量表使用计算
//                     在先前访问的状态上修剪examine()搜索
// 1.2   2007年1月5日  注释清理
//                     与inflate一样，减少短代码的根
//                     拒绝inflate会增加根的情况
// 1.3   2008年2月17日  为初始根表大小添加参数
//                     修复初始根表大小== max - 1的错误
//                     使用宏计算历史索引
// 1.4   2012年8月18日  避免类型中的位移超过位数 (导致无限循环!)
//                     清理不同类型的比较
//                     清理代码缩进
// 1.5   2018年8月5日  清理代码风格、格式和注释
//                     显示最大值的所有代码，仅显示最大值

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdarg.h>
#include <stdint.h>
#include <assert.h>

#define local static

// Special data types.
typedef uintmax_t big_t;    // code counting的类型
#define PRIbig "ju"         // big_t的printf格式
typedef uintmax_t code_t;   // bit pattern counting的类型
struct tab {                // been-here check的类型
    size_t len;             // 位向量的分配长度（以八位字节为单位）
    char *vec;              // 位向量的分配空间
};
# 用于保存结果的数组num[]，使用以下三个索引：

      syms: 剩余待编码的符号数量
      left: 长度为len的可用位模式数量
      len: 当前被分配的代码的位数

   当保存结果时，这些索引受到以下限制：

      syms: 3..totsym (totsym == 待编码的总符号数)
      left: 2..syms - 1，但只有偶数（所以syms == 8 -> 2, 4, 6）
      len: 1..max - 1（max == 代码的最大长度，以位为单位）

   syms == 2不会被保存，因为这会立即导致一个单一的代码。left必须是偶数，因为它表示当前长度下可用位模式的数量，这是前一个长度下数量的两倍。left在syms-1结束，因为left == syms会立即导致一个单一的代码。（left > sym是不允许的，因为这会导致一个不完整的代码。）len小于max，因为当len == max时，代码会立即完成。

   通过计算三个索引的偏移量来计算数组的索引，第一个索引（syms）最外层，最后一个索引（len）最内层。我们使用长度为max-1的列表为len索引构建数组，每个符号有syms-3个这样的列表。有totsym-2个这样的列表，每个列表的长度都是sym的函数。查看map()中索引的计算和main()中数组大小的计算。

   对于限制为15位代码的286个符号的deflate示例，数组有284,284个条目，占用2.17 MB的8字节big_t。保存结果分配的空间超过一半实际被使用--并非所有可能的三元组都在生成有效的前缀代码时被达到。
/* The array for tracking visited states, done[], is itself indexed identically
   to the num[] array as described above for the (syms, left, len) triplet.
   Each element in the array is further indexed by the (mem, rem) doublet,
   where mem is the amount of inflate table space used so far, and rem is the
   remaining unused entries in the current inflate sub-table. Each indexed
   element is simply one bit indicating whether the state has been visited or
   not. Since the ranges for mem and rem are not known a priori, each bit
   vector is of a variable size, and grows as needed to accommodate the visited
   states. mem and rem are used to calculate a single index in a triangular
   array. Since the range of mem is expected in the default case to be about
   ten times larger than the range of rem, the array is skewed to reduce the
   memory usage, with eight times the range for mem than for rem. See the
   calculations for offset and bit in been_here() for the details.

   For the deflate example of 286 symbols limited to 15-bit codes, the bit
   vectors grow to total 5.5 MB, in addition to the 4.3 MB done array itself.
 */
 
// Type for a variable-length, allocated string.
typedef struct {
    char *str;          // pointer to allocated string
    size_t size;        // size of allocation
    size_t len;         // length of string, not including terminating zero
} string_t;

// Clear a string_t.
local void string_clear(string_t *s) {
    s->str[0] = 0;
    s->len = 0;
}

// Initialize a string_t.
local void string_init(string_t *s) {
    s->size = 16;
    s->str = malloc(s->size);
    assert(s->str != NULL && "out of memory");
    string_clear(s);
}

// Release the allocation of a string_t.
local void string_free(string_t *s) {
    free(s->str);
    s->str = NULL;
    s->size = 0;
    s->len = 0;
}

// Save the results of printf with fmt and the subsequent argument list to s.
// Each call appends to s. The allocated space for s is increased as needed.
// 格式化字符串并将结果追加到字符串对象中
local void string_printf(string_t *s, char *fmt, ...) {
    va_list ap;  // 定义可变参数列表
    va_start(ap, fmt);  // 初始化可变参数列表
    size_t len = s->len;  // 获取字符串对象的长度
    int ret = vsnprintf(s->str + len, s->size - len, fmt, ap);  // 格式化字符串并将结果追加到字符串对象中
    assert(ret >= 0 && "out of memory");  // 断言，确保内存分配成功
    s->len += ret;  // 更新字符串对象的长度
    if (s->size < s->len + 1) {  // 如果字符串对象的大小不足以容纳新的内容
        do {
            s->size <<= 1;  // 扩大字符串对象的大小
            assert(s->size != 0 && "overflow");  // 断言，确保没有发生溢出
        } while (s->size < s->len + 1);
        s->str = realloc(s->str, s->size);  // 重新分配字符串对象的内存空间
        assert(s->str != NULL && "out of memory");  // 断言，确保内存分配成功
        vsnprintf(s->str + len, s->size - len, fmt, ap);  // 重新格式化字符串并将结果追加到字符串对象中
    }
    va_end(ap);  // 结束可变参数列表的使用
}

// 全局变量，避免递归传播常量或常量指针
struct {
    int max;            // 代码的最大允许位长度
    int root;           // 基本代码表的位长度
    int large;          // 到目前为止最大的代码表
    size_t size;        // num 和 done 数组中的元素数量
    big_t tot;          // 具有最大表大小的代码总数
    string_t out;       // 最大表大小的子代码显示
    int *code;          // 每个位长度分配的符号数量
    big_t *num;         // 用于代码计数的保存结果数组
    struct tab *done;   // 已评估状态的数组
} g;

// num[] 和 done[] 的索引函数
local inline size_t map(int syms, int left, int len) {
    return ((size_t)((syms - 1) >> 1) * ((syms - 2) >> 1) +
            (left >> 1) - 1) * (g.max - 1) +
           len - 1;
}

// 清理全局变量中分配的空间
local void cleanup(void) {
    if (g.done != NULL) {
        for (size_t n = 0; n < g.size; n++)
            if (g.done[n].len)
                free(g.done[n].vec);
        g.size = 0;
        free(g.done);   g.done = NULL;
    }
    free(g.num);    g.num = NULL;
    free(g.code);   g.code = NULL;
    string_free(&g.out);
}

// 返回使用长度为 len 到 max（包括）的位模式编码 syms 符号，具有左侧位模式的可能前缀代码数量
// len unused -- return -1 if there is an overflow in the counting. Keep a
// record of previous results in num to prevent repeating the same calculation.
// 如果计数溢出，则返回-1。在num中记录先前的结果，以防止重复计算。

local big_t count(int syms, int left, int len) {
    // see if only one possible code
    // 检查是否只有一种可能的编码
    if (syms == left)
        return 1;

    // note and verify the expected state
    // 记录并验证预期状态
    assert(syms > left && left > 0 && len < g.max);

    // see if we've done this one already
    // 检查是否已经完成了这个
    size_t index = map(syms, left, len);
    big_t got = g.num[index];
    if (got)
        return got;         // we have -- return the saved result
        // 如果有，返回保存的结果

    // we need to use at least this many bit patterns so that the code won't be
    // incomplete at the next length (more bit patterns than symbols)
    // 我们至少需要使用这么多位模式，以便代码在下一个长度时不会不完整（比符号多的位模式）
    int least = (left << 1) - syms;
    if (least < 0)
        least = 0;

    // we can use at most this many bit patterns, lest there not be enough
    // available for the remaining symbols at the maximum length (if there were
    // no limit to the code length, this would become: most = left - 1)
    // 我们最多可以使用这么多位模式，以免在最大长度时剩余的符号没有足够的位模式可用（如果代码长度没有限制，这将变为：most = left - 1）
    int most = (((code_t)left << (g.max - len)) - syms) /
               (((code_t)1 << (g.max - len)) - 1);

    // count all possible codes from this juncture and add them up
    // 从这个交汇点开始计算所有可能的代码并将它们相加
    big_t sum = 0;
    for (int use = least; use <= most; use++) {
        got = count(syms - use, (left - use) << 1, len + 1);
        sum += got;
        if (got == (big_t)-1 || sum < got)      // overflow
            return (big_t)-1;
    }

    // verify that all recursive calls are productive
    // 验证所有递归调用都是有效的
    assert(sum != 0);

    // save the result and return it
    // 保存结果并返回
    g.num[index] = sum;
    return sum;
}

// Return true if we've been here before, set to true if not. Set a bit in a
// bit vector to indicate visiting this state. Each (syms,len,left) state has a
// variable size bit vector indexed by (mem,rem). The bit vector is lengthened
// as needed to allow setting the (mem,rem) bit.
// 如果我们之前来过这里，则返回true，如果没有，则设置为true。在位向量中设置一个位来指示访问此状态。每个（syms，len，left）状态都有一个由（mem，rem）索引的可变大小位向量。根据需要扩展位向量以允许设置（mem，rem）位。

local int been_here(int syms, int left, int len, int mem, int rem) {
    // 根据(syms, left, len)映射到向量的索引，根据(mem, rem)映射到向量中的位
    size_t index = map(syms, left, len);
    mem -= 1 << g.root;             // mem总是包括根表
    mem >>= 1;                      // mem和rem总是偶数
    rem >>= 1;
    size_t offset = (mem >> 3) + rem;
    offset = ((offset * (offset + 1)) >> 1) + rem;
    int bit = 1 << (mem & 7);

    // 检查是否已经访问过这里
    size_t length = g.done[index].len;
    if (offset < length && (g.done[index].vec[offset] & bit) != 0)
        return 1;       // 已经访问过这里！

    // 我们以前没有访问过这里 -- 设置位来表示我们现在已经访问过

    // 检查是否需要扩展向量以设置位
    if (length <= offset) {
        // 如果已经有一个向量，扩大它，将附加空间清零
        char *vector;
        if (length) {
            do {
                length <<= 1;
            } while (length <= offset);
            vector = realloc(g.done[index].vec, length);
            assert(vector != NULL && "内存不足");
            memset(vector + g.done[index].len, 0, length - g.done[index].len);
        }

        // 否则，我们需要创建一个新的向量并将其清零
        else {
            length = 16;
            while (length <= offset)
                length <<= 1;
            vector = calloc(length, 1);
            assert(vector != NULL && "内存不足");
        }

        // 安装新的向量
        g.done[index].len = length;
        g.done[index].vec = vector;
    }

    // 设置位
    g.done[index].vec[offset] |= bit;
    return 0;
// 检查给定节点（syms，len，left）的所有可能的代码。计算构建inflate解码表所需的内存量，其中到目前为止使用的代码结构数量为mem，当前子表中剩余的代码数量为rem。
local void examine(int syms, int left, int len, int mem, int rem) {
    // 看看我们是否有一个完整的代码
    if (syms == left) {
        // 设置最后一个代码条目
        g.code[len] = left;

        // 完成计算此代码使用的内存
        while (rem < left) {
            left -= rem;
            rem = 1 << (len - g.root);
            mem += rem;
        }
        assert(rem == left);

        // 如果这是最大值，则显示子代码
        if (mem >= g.large) {
            // 如果这是一个新的最大值，则更新最大值并清除先前最大值的打印子代码
            if (mem > g.large) {
                g.large = mem;
                string_clear(&g.out);
            }

            // 计算此子代码的起始状态
            syms = 0;
            left = 1 << g.max;
            for (int bits = g.max; bits > g.root; bits--) {
                syms += g.code[bits];
                left -= g.code[bits];
                assert((left & 1) == 0);
                left >>= 1;
            }

            // 将起始状态和生成的子代码打印到g.out
            string_printf(&g.out, "<%u, %u, %u>:",
                          syms, g.root + 1, ((1 << g.root) - left) << 1);
            for (int bits = g.root + 1; bits <= g.max; bits++)
                if (g.code[bits])
                    string_printf(&g.out, " %d[%d]", g.code[bits], bits);
            string_printf(&g.out, "\n");
        }

        // 在递归中回退时删除条目
        g.code[len] = 0;
        return;
    }

    // 如果可能的话，修剪树
    if (been_here(syms, left, len, mem, rem))
        return;
}
    // 计算至少需要这么多位模式，以便在下一个长度时不会出现不完整的编码（比符号多的位模式）
    int least = (left << 1) - syms;
    if (least < 0)
        least = 0;

    // 最多可以使用这么多位模式，以免在最大长度时剩余的符号不够（如果编码长度没有限制，这将变为：most = left - 1）
    int most = (((code_t)left << (g.max - len)) - syms) /
               (((code_t)1 << (g.max - len)) - 1);

    // 占用至少的表空间，根据需要创建新的子表
    int use = least;
    while (rem < use) {
        use -= rem;
        rem = 1 << (len - g.root);
        mem += rem;
    }
    rem -= use;

    // 从这里开始检查编码，随着进行更新表空间
    for (use = least; use <= most; use++) {
        g.code[len] = use;
        examine(syms - use, (left - use) << 1, len + 1,
                mem + (rem ? 1 << (len - g.root) : 0), rem << 1);
        if (rem == 0) {
            rem = 1 << (len - g.root);
            mem += rem;
        }
        rem--;
    }

    // 在递归回退时删除条目
    g.code[len] = 0;
// 检查所有以 root + 1 位开始的子代码。只查看有效的中间代码状态（syms、left、len）。对于每个完成的代码，计算解压缩所需的内存量来构建解码表。找到所需的最大内存量，并显示需要该最大内存量的代码。
local void enough(int syms) {
    // 清除代码
    for (int n = 0; n <= g.max; n++)
        g.code[n] = 0;

    // 查看所有（root + 1）位及更长的代码
    string_clear(&g.out);           // 清空保存的结果
    g.large = 1 << g.root;          // 基本表
    if (g.root < g.max)             // 否则，只有一个基本表
        for (int n = 3; n <= syms; n++)
            for (int left = 2; left < n; left += 2) {
                // 查看所有可达的（root + 1）位节点，以及结果代码（在 root + 2 或更多时完成）
                size_t index = map(n, left, g.root + 1);
                if (g.root + 1 < g.max && g.num[index]) // 可达节点
                    examine(n, left, g.root + 1, 1 << g.root, 0);

                // 还要查看在 root 位代码上完成 root + 1 位（不保存在 num 中，因为完成），以防万一
                if (g.num[index - 1] && n <= left << 1)
                    examine((n - left) << 1, (n - left) << 1, g.root + 1,
                            1 << g.root, 0);
            }

    // 完成
    printf("maximum of %d table entries for root = %d\n", g.large, g.root);
    fputs(g.out.str, stdout);
}

// 检查并显示给定符号的最大数、初始根表大小和最大代码长度（以位为单位）的可能前缀代码的总数 -- 这些是按照命令参数的顺序。默认值分别为 286、9 和 15，用于 deflate 字面/长度代码。对于每个编码符号数从两到
// 设置全局变量以便清理
g.code = NULL;
g.num = NULL;
g.done = NULL;
string_init(&g.out);

// 获取参数，默认为deflate字面长度代码
int syms = 286;
g.root = 9;
g.max = 15;
if (argc > 1) {
    syms = atoi(argv[1]);
    if (argc > 2) {
        g.root = atoi(argv[2]);
        if (argc > 3)
            g.max = atoi(argv[3]);
    }
}
if (argc > 4 || syms < 2 || g.root < 1 || g.max < 1) {
    fputs("invalid arguments, need: [sym >= 2 [root >= 1 [max >= 1]]]\n",
          stderr);
    return 1;
}

// 如果不限制代码长度，最长为syms - 1
if (g.max > syms - 1)
    g.max = syms - 1;

// 确定code_t中的位数
int bits = 0;
for (code_t word = 1; word; word <<= 1)
    bits++;

// 确保most的计算不会溢出
if (g.max > bits || (code_t)(syms - 2) >= ((code_t)-1 >> (g.max - 1))) {
    fputs("abort: code length too long for internal types\n", stderr);
    return 1;
}

// 拒绝不可能的代码请求
if ((code_t)(syms - 1) > ((code_t)1 << g.max) - 1) {
    fprintf(stderr, "%d symbols cannot be coded in %d bits\n",
            syms, g.max);
    return 1;
}

// 分配代码向量
g.code = calloc(g.max + 1, sizeof(int));
    # 确保 g.code 不为空，如果为空则表示内存不足
    assert(g.code != NULL && "out of memory");

    // 确定保存结果数组的大小，检查是否溢出，分配并清空数组（使用 calloc() 将所有元素设置为零）
    if (syms == 2)              // 当且仅当 max == 1 时
        g.num = NULL;           // 不需要保存任何结果
    else:
        g.size = syms >> 1;
        int n = (syms - 1) >> 1;
        assert(g.size <= (size_t)-1 / n && "overflow");
        g.size *= n;
        n = g.max - 1;
        assert(g.size <= (size_t)-1 / n && "overflow");
        g.size *= n;
        g.num = calloc(g.size, sizeof(big_t));
        assert(g.num != NULL && "out of memory");

    // 计算所有符号数量的可能编码，累加计数
    big_t sum = 0;
    for (int n = 2; n <= syms; n++) {
        big_t got = count(n, 2, 1);
        sum += got;
        assert(got != (big_t)-1 && sum >= got && "overflow");
    }
    printf("%"PRIbig" total codes for 2 to %d symbols", sum, syms);
    if (g.max < syms - 1)
        printf(" (%d-bit length limit)\n", g.max);
    else
        puts(" (no length limit)");

    // 为 been_here() 分配并清空 done 数组
    if (syms == 2)
        g.done = NULL;
    else {
        g.done = calloc(g.size, sizeof(struct tab));
        assert(g.done != NULL && "out of memory");
    }

    // 查找并显示最大的解压表使用情况
    if (g.root > g.max)             // 将 root 降低到最大长度
        g.root = g.max;
    if ((code_t)syms < ((code_t)1 << (g.root + 1)))
        enough(syms);
    else
        fputs("cannot handle minimum code lengths > root", stderr);

    // 完成
    cleanup();
    return 0;
# 闭合前面的函数定义
```