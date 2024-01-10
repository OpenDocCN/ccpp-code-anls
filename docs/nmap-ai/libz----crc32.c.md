# `nmap\libz\crc32.c`

```
/* crc32.c -- compute the CRC-32 of a data stream
 * 计算数据流的 CRC-32
 * Copyright (C) 1995-2022 Mark Adler
 * 版权所有 1995-2022 Mark Adler
 * For conditions of distribution and use, see copyright notice in zlib.h
 * 分发和使用条件，请参阅 zlib.h 中的版权声明
 *
 * This interleaved implementation of a CRC makes use of pipelined multiple
 * arithmetic-logic units, commonly found in modern CPU cores. It is due to
 * Kadatch and Jenkins (2010). See doc/crc-doc.1.0.pdf in this distribution.
 * 这种交错实现的 CRC 利用了现代 CPU 核心中常见的流水线多算术逻辑单元。这是由 Kadatch 和 Jenkins (2010) 创造的。请参阅本发行版中的 doc/crc-doc.1.0.pdf。
 */

/* @(#) $Id$ */

/*
  Note on the use of DYNAMIC_CRC_TABLE: there is no mutex or semaphore
  protection on the static variables used to control the first-use generation
  of the crc tables. Therefore, if you #define DYNAMIC_CRC_TABLE, you should
  first call get_crc_table() to initialize the tables before allowing more than
  one thread to use crc32().
  关于使用 DYNAMIC_CRC_TABLE 的注意事项：在用于控制首次生成 crc 表的静态变量上没有互斥锁或信号量保护。因此，如果您定义了 DYNAMIC_CRC_TABLE，您应该先调用 get_crc_table() 来初始化表，然后再允许多个线程使用 crc32()。

  MAKECRCH can be #defined to write out crc32.h. A main() routine is also
  produced, so that this one source file can be compiled to an executable.
  可以将 MAKECRCH 定义为写出 crc32.h。还会生成一个 main() 程序，以便将此源文件编译为可执行文件。
 */

#ifdef MAKECRCH
#  include <stdio.h>
#  ifndef DYNAMIC_CRC_TABLE
#    define DYNAMIC_CRC_TABLE
#  endif /* !DYNAMIC_CRC_TABLE */
#endif /* MAKECRCH */
#include "zutil.h"      /* for Z_U4, Z_U8, z_crc_t, and FAR definitions */

 /*
  A CRC of a message is computed on N braids of words in the message, where
  each word consists of W bytes (4 or 8). If N is 3, for example, then three
  running sparse CRCs are calculated respectively on each braid, at these
  indices in the array of words: 0, 3, 6, ..., 1, 4, 7, ..., and 2, 5, 8, ...
  This is done starting at a word boundary, and continues until as many blocks
  of N * W bytes as are available have been processed. The results are combined
  into a single CRC at the end. For this code, N must be in the range 1..6 and
  W must be 4 or 8. The upper limit on N can be increased if desired by adding
  more #if blocks, extending the patterns apparent in the code. In addition,
  crc32.h would need to be regenerated, if the maximum N value is increased.

  N and W are chosen empirically by benchmarking the execution time on a given
  processor. The choices for N and W below were based on testing on Intel Kaby
  Lake i7, AMD Ryzen 7, ARM Cortex-A57, Sparc64-VII, PowerPC POWER9, and MIPS64
  Octeon II processors. The Intel, AMD, and ARM processors were all fastest
  with N=5, W=8. The Sparc, PowerPC, and MIPS64 were all fastest at N=5, W=4.
  They were all tested with either gcc or clang, all using the -O3 optimization
  level. Your mileage may vary.
 */

/* Define N */
#ifdef Z_TESTN
#  define N Z_TESTN
#else
#  define N 5
#endif
#if N < 1 || N > 6
#  error N must be in 1..6
#endif

/*
  z_crc_t must be at least 32 bits. z_word_t must be at least as long as
  z_crc_t. It is assumed here that z_word_t is either 32 bits or 64 bits, and
  that bytes are eight bits.
 */

/*
  Define W and the associated z_word_t type. If W is not defined, then a
  braided calculation is not used, and the associated tables and code are not
  compiled.
 */
#ifdef Z_TESTW
#  if Z_TESTW-1 != -1
#    define W Z_TESTW
#  endif
#else
#  ifdef MAKECRCH
#    define W 8         /* required for MAKECRCH */
#ifdef W
#  if W == 8 && defined(Z_U8)
     typedef Z_U8 z_word_t;
#  elif defined(Z_U4)
#    undef W
#    define W 4
     typedef Z_U4 z_word_t;
#  else
#    undef W
#  endif
#endif
#ifdef W
#  if W == 8 && defined(Z_U8)
     typedef Z_U8 z_word_t;
#  elif defined(Z_U4)
#    undef W
#    define W 4
     typedef Z_U4 z_word_t;
#  else
#    undef W
#  endif
#endif
/* If available, use the ARM processor CRC32 instruction. */
#if defined(__aarch64__) && defined(__ARM_FEATURE_CRC32) && W == 8
#  define ARMCRC32
#endif
/* Local functions. */
local z_crc_t multmodp OF((z_crc_t a, z_crc_t b));
local z_crc_t x2nmodp OF((z_off64_t n, unsigned k));
#if defined(W) && (!defined(ARMCRC32) || defined(DYNAMIC_CRC_TABLE))
    local z_word_t byte_swap OF((z_word_t word));
#endif
#if defined(W) && !defined(ARMCRC32)
    local z_crc_t crc_word OF((z_word_t data));
    local z_word_t crc_word_big OF((z_word_t data));
#endif
#if defined(W) && (!defined(ARMCRC32) || defined(DYNAMIC_CRC_TABLE))
/*
  Swap the bytes in a z_word_t to convert between little and big endian. Any
  self-respecting compiler will optimize this to a single machine byte-swap
  instruction, if one is available. This assumes that word_t is either 32 bits
  or 64 bits.
 */
local z_word_t byte_swap(word)
    z_word_t word;
{
#  if W == 8
    return
        (word & 0xff00000000000000) >> 56 |
        (word & 0xff000000000000) >> 40 |
        (word & 0xff0000000000) >> 24 |
        (word & 0xff00000000) >> 8 |
        (word & 0xff000000) << 8 |
        (word & 0xff0000) << 24 |
        (word & 0xff00) << 40 |
        (word & 0xff) << 56;
#  else   /* W == 4 */
    return
        (word & 0xff000000) >> 24 |
        (word & 0xff0000) >> 8 |
        (word & 0xff00) << 8 |
        (word & 0xff) << 24;
#  endif
}
#endif
/* CRC polynomial. */
#define POLY 0xedb88320         /* p(x) reflected, with x^32 implied */
#ifdef DYNAMIC_CRC_TABLE
local z_crc_t FAR crc_table[256];
local z_crc_t FAR x2n_table[32];
local void make_crc_table OF((void));
#ifdef W
   # 定义一个大小为256的 crc_big_table 数组
   local z_word_t FAR crc_big_table[256];
   # 定义一个大小为 W*256 的 crc_braid_table 二维数组
   local z_crc_t FAR crc_braid_table[W][256];
   # 定义一个大小为 W*256 的 crc_braid_big_table 二维数组
   local z_word_t FAR crc_braid_big_table[W][256];
   # 定义一个名为 braid 的函数，接受两个 z_crc_t 类型的二维数组和两个整数参数
   local void braid OF((z_crc_t [][256], z_word_t [][256], int, int));
#endif
#ifdef MAKECRCH
   # 定义一个名为 write_table 的函数，接受一个文件指针和一个大小为256的 z_crc_t 数组
   local void write_table OF((FILE *, const z_crc_t FAR *, int));
   # 定义一个名为 write_table32hi 的函数，接受一个文件指针和一个大小为256的 z_word_t 数组
   local void write_table32hi OF((FILE *, const z_word_t FAR *, int));
   # 定义一个名为 write_table64 的函数，接受一个文件指针和一个大小为256的 z_word_t 数组
   local void write_table64 OF((FILE *, const z_word_t FAR *, int));
#endif /* MAKECRCH */

/*
  Define a once() function depending on the availability of atomics. If this is
  compiled with DYNAMIC_CRC_TABLE defined, and if CRCs will be computed in
  multiple threads, and if atomics are not available, then get_crc_table() must
  be called to initialize the tables and must return before any threads are
  allowed to compute or combine CRCs.
 */

/* Definition of once functionality. */
# 定义一个名为 once_t 的结构体
typedef struct once_s once_t;
# 定义一个名为 once 的函数，接受一个 once_t 指针和一个函数指针作为参数
local void once OF((once_t *, void (*)(void)));

/* Check for the availability of atomics. */
# 如果支持 C11 标准并且原子操作可用
#if defined(__STDC__) && __STDC_VERSION__ >= 201112L && \
    !defined(__STDC_NO_ATOMICS__)

#include <stdatomic.h>

/* Structure for once(), which must be initialized with ONCE_INIT. */
# 定义一个结构体 once_s，其中包含一个原子标志和一个原子整型
struct once_s {
    atomic_flag begun;
    atomic_int done;
};
# 定义一个名为 ONCE_INIT 的宏，用于初始化 once_t 结构体
#define ONCE_INIT {ATOMIC_FLAG_INIT, 0}

/*
  Run the provided init() function exactly once, even if multiple threads
  invoke once() at the same time. The state must be a once_t initialized with
  ONCE_INIT.
 */
# 定义一个名为 once 的函数，接受一个 once_t 指针和一个函数指针作为参数
local void once(state, init)
    once_t *state;
    void (*init)(void);
{
    # 如果 done 标志为假
    if (!atomic_load(&state->done)) {
        # 如果 begun 标志为真
        if (atomic_flag_test_and_set(&state->begun))
            # 循环等待 done 标志为真
            while (!atomic_load(&state->done))
                ;
        else {
            # 调用 init 函数
            init();
            # 设置 done 标志为真
            atomic_store(&state->done, 1);
        }
    }
}

#else   /* no atomics */

/* Structure for once(), which must be initialized with ONCE_INIT. */
# 定义一个结构体 once_s，其中包含两个 volatile 整型
struct once_s {
    volatile int begun;
    volatile int done;
};
# 定义一个名为 ONCE_INIT 的宏，用于初始化 once_t 结构体
#define ONCE_INIT {0, 0}
/* Test and set. Alas, not atomic, but tries to minimize the period of
   vulnerability. */
// 定义一个测试并设置的函数，尽量减少漏洞期
local int test_and_set OF((int volatile *));
local int test_and_set(flag)
    int volatile *flag;
{
    int was;

    was = *flag;  // 保存当前标志的值
    *flag = 1;  // 将标志设置为1
    return was;  // 返回之前标志的值
}

/* Run the provided init() function once. This is not thread-safe. */
// 运行提供的init()函数一次，这不是线程安全的
local void once(state, init)
    once_t *state;
    void (*init)(void);
{
    if (!state->done) {  // 如果状态未完成
        if (test_and_set(&state->begun))  // 如果测试并设置返回真
            while (!state->done)  // 当状态未完成时循环
                ;
        else {
            init();  // 运行初始化函数
            state->done = 1;  // 设置状态为完成
        }
    }
}

#endif

/* State for once(). */
// 为once()函数定义状态
local once_t made = ONCE_INIT;
# 生成用于按字节进行32位CRC计算的表格，使用的多项式是 x^32+x^26+x^23+x^22+x^16+x^12+x^11+x^10+x^8+x^7+x^5+x^4+x^2+x+1
# 多项式在GF(2)上的表示是二进制形式，每个系数占一位，最低次幂在最高位
# 多项式相加就是异或操作，多项式乘以x就是右移一位
# CRC计算是使用移位寄存器方法进行乘法和取余
# 表格包含了所有可能的8位值的CRC，可以用于按字节生成CRC

local void make_crc_table()
{
    unsigned i, j, n;
    z_crc_t p;

    /* 初始化字节CRC表 */
    for (i = 0; i < 256; i++) {
        p = i;
        for (j = 0; j < 8; j++)
            p = p & 1 ? (p >> 1) ^ POLY : p >> 1;
        crc_table[i] = p;
#ifdef W
        crc_big_table[i] = byte_swap(p);
#endif
    }

    /* 初始化 x^2^n mod p(x) 表 */
    p = (z_crc_t)1 << 30;         /* x^1 */
    x2n_table[0] = p;
    for (n = 1; n < 32; n++)
        x2n_table[n] = p = multmodp(p, p);

#ifdef W
    /* 初始化编织表 -- 需要 x2n_table[] */
    # 调用名为braid的函数，传入参数crc_braid_table, crc_braid_big_table, N, W
    braid(crc_braid_table, crc_braid_big_table, N, W);
#endif

#ifdef MAKECRCH
    {
        /*
          crc32.h 头文件包含了 32 位和 64 位 z_word_t 的表，因此需要一个 64 位类型可用。
          在这种情况下，z_word_t 必须被定义为 64 位。这段代码还生成并写出了 z_word_t 为 32 位的情况下的表。
         */
#if !defined(W) || W != 8
#  error Need a 64-bit integer type in order to generate crc32.h.
    }
#endif /* MAKECRCH */
}

#ifdef MAKECRCH

/*
   将 table[0..k-1] 中的 32 位值以十六进制逗号分隔的形式每行写入五个到 out 中。
 */
local void write_table(out, table, k)
    FILE *out;
    const z_crc_t FAR *table;
    int k;
{
    int n;

    for (n = 0; n < k; n++)
        fprintf(out, "%s0x%08lx%s", n == 0 || n % 5 ? "" : "    ",
                (unsigned long)(table[n]),
                n == k - 1 ? "" : (n % 5 == 4 ? ",\n" : ", "));
}

/*
   将 table[0..k-1] 中每个值的高 32 位以十六进制逗号分隔的形式每行写入五个到 out 中。
 */
local void write_table32hi(out, table, k)
FILE *out;
const z_word_t FAR *table;
int k;
{
    int n;

    for (n = 0; n < k; n++)
        fprintf(out, "%s0x%08lx%s", n == 0 || n % 5 ? "" : "    ",
                (unsigned long)(table[n] >> 32),
                n == k - 1 ? "" : (n % 5 == 4 ? ",\n" : ", "));
}

/*
  将 table[0..k-1] 中的 64 位值以十六进制逗号分隔的形式每行写入三个到 out 中。
  这假设如果有一个 64 位类型，那么也有一个 long long 整数类型，并且它至少有 64 位。
  如果没有，那么类型转换和格式字符串可以相应地调整。
 */
local void write_table64(out, table, k)
    FILE *out;
    const z_word_t FAR *table;
    int k;
{
    int n;

    for (n = 0; n < k; n++)
        fprintf(out, "%s0x%016llx%s", n == 0 || n % 3 ? "" : "    ",
                (unsigned long long)(table[n]),
                n == k - 1 ? "" : (n % 3 == 2 ? ",\n" : ", "));
}
/* Actually do the deed. */
int main()
{
    // 调用make_crc_table函数
    make_crc_table();
    // 返回0
    return 0;
}

#endif /* MAKECRCH */

#ifdef W
/*
  Generate the little and big-endian braid tables for the given n and z_word_t
  size w. Each array must have room for w blocks of 256 elements.
 */
local void braid(ltl, big, n, w)
    z_crc_t ltl[][256];
    z_word_t big[][256];
    int n;
    int w;
{
    int k;
    z_crc_t i, p, q;
    // 循环生成braid表
    for (k = 0; k < w; k++) {
        // 计算p的值
        p = x2nmodp((n * w + 3 - k) << 3, 0);
        // 初始化ltl和big数组
        ltl[k][0] = 0;
        big[w - 1 - k][0] = 0;
        // 循环生成braid表的值
        for (i = 1; i < 256; i++) {
            ltl[k][i] = q = multmodp(i << 24, p);
            big[w - 1 - k][i] = byte_swap(q);
        }
    }
}
#endif

#else /* !DYNAMIC_CRC_TABLE */
/* ========================================================================
 * Tables for byte-wise and braided CRC-32 calculations, and a table of powers
 * of x for combining CRC-32s, all made by make_crc_table().
 */
#include "crc32.h"
#endif /* DYNAMIC_CRC_TABLE */

/* ========================================================================
 * Routines used for CRC calculation. Some are also required for the table
 * generation above.
 */

/*
  Return a(x) multiplied by b(x) modulo p(x), where p(x) is the CRC polynomial,
  reflected. For speed, this requires that a not be zero.
 */
local z_crc_t multmodp(a, b)
    z_crc_t a;
    z_crc_t b;
{
    z_crc_t m, p;
    // 初始化m和p的值
    m = (z_crc_t)1 << 31;
    p = 0;
    // 循环计算a(x) * b(x) % p(x)
    for (;;) {
        if (a & m) {
            p ^= b;
            if ((a & (m - 1)) == 0)
                break;
        }
        m >>= 1;
        b = b & 1 ? (b >> 1) ^ POLY : b >> 1;
    }
    return p;
}

/*
  Return x^(n * 2^k) modulo p(x). Requires that x2n_table[] has been
  initialized.
 */
local z_crc_t x2nmodp(n, k)
    z_off64_t n;
    unsigned k;
{
    z_crc_t p;
    // 初始化p的值
    p = (z_crc_t)1 << 31;           /* x^0 == 1 */
    // 循环计算x^(n * 2^k) % p(x)
    while (n) {
        if (n & 1)
            p = multmodp(x2n_table[k & 31], p);
        n >>= 1;
        k++;
    }
    return p;
}
/* =========================================================================
 * 此函数可被 crc32() 的汇编版本使用，并且可用于在多线程应用程序中强制生成 CRC 表。
 */
const z_crc_t FAR * ZEXPORT get_crc_table()
{
#ifdef DYNAMIC_CRC_TABLE
    // 如果启用了动态 CRC 表，则使用 once 函数确保只执行一次 make_crc_table 函数
    once(&made, make_crc_table);
#endif /* DYNAMIC_CRC_TABLE */
    // 返回 CRC 表的指针
    return (const z_crc_t FAR *)crc_table;
}

/* =========================================================================
 * 如果可用，则使用 ARM 机器指令。这将比编织计算快大约十倍。此代码不会在运行时检查 CRC 指令的存在。
 * 只有在编译指定具有该指令的 ARM 处理器架构时，__ARM_FEATURE_CRC32 才会被定义。例如，使用 -march=armv8.1-a 或 -march=armv8-a+crc 进行编译，或者如果编译机器具有 crc32 指令，则使用 -march=native。
 */
#ifdef ARMCRC

/*
   经验确定的常数，用于最大化速度。这些值来自于对 Cortex-A57 的测量。实际情况可能有所不同。
 */
#define Z_BATCH 3990                /* 一个批次中的字数 */
#define Z_BATCH_ZEROS 0xa10d3d0c    /* 从 Z_BATCH = 3990 计算得出 */
#define Z_BATCH_MIN 800             /* 最终批次中的最少字数 */

unsigned long ZEXPORT crc32_z(crc, buf, len)
    unsigned long crc;
    const unsigned char FAR *buf;
    z_size_t len;
{
    z_crc_t val;
    z_word_t crc1, crc2;
    const z_word_t *word;
    z_word_t val0, val1, val2;
    z_size_t last, last2, i;
    z_size_t num;

    /* 如果请求返回初始 CRC，则返回初始 CRC */
    if (buf == Z_NULL) return 0;

#ifdef DYNAMIC_CRC_TABLE
    // 如果启用了动态 CRC 表，则使用 once 函数确保只执行一次 make_crc_table 函数
    once(&made, make_crc_table);
#endif /* DYNAMIC_CRC_TABLE */

    /* 预处理 CRC */
    crc = (~crc) & 0xffffffff;

    /* 计算直到字边界的 CRC。 */
    # 当剩余长度大于0且buf的地址不是8的倍数时，执行循环
    while (len && ((z_size_t)buf & 7) != 0) {
        # 长度减一
        len--;
        # 从buf中读取一个字节，赋值给val
        val = *buf++;
        # 使用汇编指令计算CRC32B校验值
        __asm__ volatile("crc32b %w0, %w0, %w1" : "+r"(crc) : "r"(val));
    }

    # 准备计算完整64位字的CRC校验值
    word = (z_word_t const *)buf;
    num = len >> 3;
    len &= 7;

    # 执行三个交错的CRC计算，以实现每个周期一个crc32x指令的吞吐量
    # 每个CRC计算在Z_BATCH个字上进行，然后每组批次后将三个CRC合并为一个CRC
    while (num >= 3 * Z_BATCH) {
        crc1 = 0;
        crc2 = 0;
        for (i = 0; i < Z_BATCH; i++) {
            val0 = word[i];
            val1 = word[i + Z_BATCH];
            val2 = word[i + 2 * Z_BATCH];
            __asm__ volatile("crc32x %w0, %w0, %x1" : "+r"(crc) : "r"(val0));
            __asm__ volatile("crc32x %w0, %w0, %x1" : "+r"(crc1) : "r"(val1));
            __asm__ volatile("crc32x %w0, %w0, %x1" : "+r"(crc2) : "r"(val2));
        }
        word += 3 * Z_BATCH;
        num -= 3 * Z_BATCH;
        crc = multmodp(Z_BATCH_ZEROS, crc) ^ crc1;
        crc = multmodp(Z_BATCH_ZEROS, crc) ^ crc2;
    }

    # 如果剩余的字数足够支付CRC合并的成本，则执行最后一个较小的批次
    last = num / 3;
    if (last >= Z_BATCH_MIN) {
        last2 = last << 1;
        crc1 = 0;
        crc2 = 0;
        for (i = 0; i < last; i++) {
            val0 = word[i];
            val1 = word[i + last];
            val2 = word[i + last2];
            __asm__ volatile("crc32x %w0, %w0, %x1" : "+r"(crc) : "r"(val0));
            __asm__ volatile("crc32x %w0, %w0, %x1" : "+r"(crc1) : "r"(val1));
            __asm__ volatile("crc32x %w0, %w0, %x1" : "+r"(crc2) : "r"(val2));
        }
        word += 3 * last;
        num -= 3 * last;
        val = x2nmodp(last, 6);
        crc = multmodp(val, crc) ^ crc1;
        crc = multmodp(val, crc) ^ crc2;
    }
    /* 计算剩余字的 CRC。*/
    for (i = 0; i < num; i++) {
        val0 = word[i];
        __asm__ volatile("crc32x %w0, %w0, %x1" : "+r"(crc) : "r"(val0));
    }
    word += num;

    /* 完成剩余字节的 CRC。*/
    buf = (const unsigned char FAR *)word;
    while (len) {
        len--;
        val = *buf++;
        __asm__ volatile("crc32b %w0, %w0, %w1" : "+r"(crc) : "r"(val));
    }

    /* 返回 CRC，后处理。*/
    return crc ^ 0xffffffff;
}

#else

#ifdef W

/*
  Return the CRC of the W bytes in the word_t data, taking the
  least-significant byte of the word as the first byte of data, without any pre
  or post conditioning. This is used to combine the CRCs of each braid.
 */
// 计算 word_t 数据中 W 个字节的 CRC，将 word 的最低有效字节作为数据的第一个字节，不进行任何预处理或后处理。用于组合每个编织的 CRC。
local z_crc_t crc_word(data)
    z_word_t data;
{
    int k;
    for (k = 0; k < W; k++)
        data = (data >> 8) ^ crc_table[data & 0xff];
    return (z_crc_t)data;
}

// 计算大端字节序下 word_t 数据中 W 个字节的 CRC
local z_word_t crc_word_big(data)
    z_word_t data;
{
    int k;
    for (k = 0; k < W; k++)
        data = (data << 8) ^
            crc_big_table[(data >> ((W - 1) << 3)) & 0xff];
    return data;
}

#endif

/* ========================================================================= */
// 计算给定缓冲区中指定长度的数据的 CRC32 值
unsigned long ZEXPORT crc32_z(crc, buf, len)
    unsigned long crc;
    const unsigned char FAR *buf;
    z_size_t len;
{
    /* 如果请求，返回初始 CRC。 */
    if (buf == Z_NULL) return 0;

#ifdef DYNAMIC_CRC_TABLE
    once(&made, make_crc_table);
#endif /* DYNAMIC_CRC_TABLE */

    /* 预处理 CRC */
    crc = (~crc) & 0xffffffff;

#ifdef W

    /* 如果提供足够的字节，进行编织 CRC 计算。 */
    # 如果输入的长度大于等于 N * W + W - 1
    if (len >= N * W + W - 1) {
        # 定义变量 blks 用于存储 z_word_t 块的数量
        z_size_t blks;
        # 定义指针变量 words 用于存储 z_word_t 类型的数据
        z_word_t const *words;
        # 定义变量 endian 用于存储字节序信息
        unsigned endian;
        # 定义变量 k

        # 计算 CRC 直到达到 z_word_t 边界
        while (len && ((z_size_t)buf & (W - 1)) != 0) {
            len--;
            crc = (crc >> 8) ^ crc_table[(crc ^ *buf++) & 0xff];
        }

        # 计算尽可能多的 N 个 z_word_t 块的 CRC
        blks = len / (N * W);
        len -= blks * N * W;
        words = (z_word_t const *)buf;

        # 在执行时进行字节序检查，而不是在编译时，因为 ARM 处理器可以在执行时更改字节序。
        # 如果编译器知道字节序，它可以优化掉检查和未使用的分支。
        endian = 1;
        if (*(unsigned char *)&endian) {
            # 小端字节序

            # 定义变量 crc0 用于存储 CRC 值
            z_crc_t crc0;
            # 定义变量 word0 用于存储 z_word_t 类型的数据
            z_word_t word0;
#if N > 1
            // 如果 N 大于 1，则声明变量 crc1 和 word1
            z_crc_t crc1;
            z_word_t word1;
#if N > 2
            // 如果 N 大于 2，则声明变量 crc2 和 word2
            z_crc_t crc2;
            z_word_t word2;
#if N > 3
            // 如果 N 大于 3，则声明变量 crc3 和 word3
            z_crc_t crc3;
            z_word_t word3;
#if N > 4
            // 如果 N 大于 4，则声明变量 crc4 和 word4
            z_crc_t crc4;
            z_word_t word4;
#if N > 5
            // 如果 N 大于 5，则声明变量 crc5 和 word5
            z_crc_t crc5;
            z_word_t word5;
#endif
#endif
#endif
#endif
#endif

            /* Initialize the CRC for each braid. */
            // 初始化第一个 braid 的 CRC
            crc0 = crc;
#if N > 1
            // 如果 N 大于 1，则初始化 crc1
            crc1 = 0;
#if N > 2
            // 如果 N 大于 2，则初始化 crc2
            crc2 = 0;
#if N > 3
            // 如果 N 大于 3，则初始化 crc3
            crc3 = 0;
#if N > 4
            // 如果 N 大于 4，则初始化 crc4
            crc4 = 0;
#if N > 5
            // 如果 N 大于 5，则初始化 crc5
            crc5 = 0;
#endif
#endif
#endif
#endif
#endif

            /*
              Process the first blks-1 blocks, computing the CRCs on each braid
              independently.
             */
            // 处理前 blks-1 个块，独立计算每个 braid 的 CRC
            while (--blks) {
                /* Load the word for each braid into registers. */
                // 将每个 braid 的 word 加载到寄存器中
                word0 = crc0 ^ words[0];
#if N > 1
                // 如果 N 大于 1，则加载 word1
                word1 = crc1 ^ words[1];
#if N > 2
                // 如果 N 大于 2，则加载 word2
                word2 = crc2 ^ words[2];
#if N > 3
                // 如果 N 大于 3，则加载 word3
                word3 = crc3 ^ words[3];
#if N > 4
                // 如果 N 大于 4，则加载 word4
                word4 = crc4 ^ words[4];
#if N > 5
                // 如果 N 大于 5，则加载 word5
                word5 = crc5 ^ words[5];
#endif
#endif
#endif
#endif
#endif
                words += N;

                /* Compute and update the CRC for each word. The loop should
                   get unrolled. */
                // 计算并更新每个 word 的 CRC。循环应该展开。
                crc0 = crc_braid_table[0][word0 & 0xff];
#if N > 1
                // 如果 N 大于 1，则计算并更新 crc1
                crc1 = crc_braid_table[0][word1 & 0xff];
#if N > 2
                // 如果 N 大于 2，则计算并更新 crc2
                crc2 = crc_braid_table[0][word2 & 0xff];
#if N > 3
                // 如果 N 大于 3，则计算并更新 crc3
                crc3 = crc_braid_table[0][word3 & 0xff];
#if N > 4
                // 如果 N 大于 4，则计算并更新 crc4
                crc4 = crc_braid_table[0][word4 & 0xff];
#if N > 5
                // 如果 N 大于 5，则计算并更新 crc5
                crc5 = crc_braid_table[0][word5 & 0xff];
#endif
#endif
#endif
#endif
#endif
                for (k = 1; k < W; k++) {
                    // 对每个 word 计算并更新 CRC。循环应该展开。
                    crc0 ^= crc_braid_table[k][(word0 >> (k << 3)) & 0xff];
#if N > 1
                    // 如果 N 大于 1，则对每个 word1 计算并更新 CRC
                    crc1 ^= crc_braid_table[k][(word1 >> (k << 3)) & 0xff];
#if N > 2
                    # 如果 N 大于 2，则对第二个字进行 CRC 计算并与 crc2 异或
                    crc2 ^= crc_braid_table[k][(word2 >> (k << 3)) & 0xff];
#if N > 3
                    # 如果 N 大于 3，则对第三个字进行 CRC 计算并与 crc3 异或
                    crc3 ^= crc_braid_table[k][(word3 >> (k << 3)) & 0xff];
#if N > 4
                    # 如果 N 大于 4，则对第四个字进行 CRC 计算并与 crc4 异或
                    crc4 ^= crc_braid_table[k][(word4 >> (k << 3)) & 0xff];
#if N > 5
                    # 如果 N 大于 5，则对第五个字进行 CRC 计算并与 crc5 异或
                    crc5 ^= crc_braid_table[k][(word5 >> (k << 3)) & 0xff];
#endif
#endif
#endif
#endif
#endif
                }
            }

            /*
              Process the last block, combining the CRCs of the N braids at the
              same time.
             */
            # 计算最后一个块的 CRC，同时合并 N 条编织的 CRC
            crc = crc_word(crc0 ^ words[0]);
#if N > 1
            # 如果 N 大于 1，则计算第二个编织的 CRC 并与之前的 CRC 异或
            crc = crc_word(crc1 ^ words[1] ^ crc);
#if N > 2
            # 如果 N 大于 2，则计算第三个编织的 CRC 并与之前的 CRC 异或
            crc = crc_word(crc2 ^ words[2] ^ crc);
#if N > 3
            # 如果 N 大于 3，则计算第四个编织的 CRC 并与之前的 CRC 异或
            crc = crc_word(crc3 ^ words[3] ^ crc);
#if N > 4
            # 如果 N 大于 4，则计算第五个编织的 CRC 并与之前的 CRC 异或
            crc = crc_word(crc4 ^ words[4] ^ crc);
#if N > 5
            # 如果 N 大于 5，则计算第六个编织的 CRC 并与之前的 CRC 异或
            crc = crc_word(crc5 ^ words[5] ^ crc);
#endif
#endif
#endif
#endif
#endif
            words += N;
        }
        else {
            /* Big endian. */

            z_word_t crc0, word0, comb;
#if N > 1
            z_word_t crc1, word1;
#if N > 2
            z_word_t crc2, word2;
#if N > 3
            z_word_t crc3, word3;
#if N > 4
            z_word_t crc4, word4;
#if N > 5
            z_word_t crc5, word5;
#endif
#endif
#endif
#endif
#endif

            /* Initialize the CRC for each braid. */
            # 为每个编织初始化 CRC
            crc0 = byte_swap(crc);
#if N > 1
            # 如果 N 大于 1，则初始化第二个编织的 CRC
            crc1 = 0;
#if N > 2
            # 如果 N 大于 2，则初始化第三个编织的 CRC
            crc2 = 0;
#if N > 3
            # 如果 N 大于 3，则初始化第四个编织的 CRC
            crc3 = 0;
#if N > 4
            # 如果 N 大于 4，则初始化第五个编织的 CRC
            crc4 = 0;
#if N > 5
            # 如果 N 大于 5，则初始化第六个编织的 CRC
            crc5 = 0;
#endif
#endif
#endif
#endif
#endif

            /*
              Process the first blks-1 blocks, computing the CRCs on each braid
              independently.
             */
            # 处理前 blks-1 个块，分别计算每个编织的 CRC
            while (--blks) {
                /* Load the word for each braid into registers. */
                # 将每个编织的字加载到寄存器中
                word0 = crc0 ^ words[0];
#if N > 1
                # 如果 N 大于 1，则加载第二个编织的字到寄存器中
                word1 = crc1 ^ words[1];
#if N > 2
                # 如果 N 大于 2，则加载第三个编织的字到寄存器中
                word2 = crc2 ^ words[2];
#if N > 3
                # 如果 N 大于 3，则计算 word3
                word3 = crc3 ^ words[3];
#if N > 4
                # 如果 N 大于 4，则计算 word4
                word4 = crc4 ^ words[4];
#if N > 5
                # 如果 N 大于 5，则计算 word5
                word5 = crc5 ^ words[5];
#endif
#endif
#endif
#endif
#endif
                # 将 words 增加 N
                words += N;

                /* 计算并更新每个 word 的 CRC。循环应该展开。 */
                # 计算 word0 的 CRC
                crc0 = crc_braid_big_table[0][word0 & 0xff];
#if N > 1
                # 如果 N 大于 1，则计算 word1 的 CRC
                crc1 = crc_braid_big_table[0][word1 & 0xff];
#if N > 2
                # 如果 N 大于 2，则计算 word2 的 CRC
                crc2 = crc_braid_big_table[0][word2 & 0xff];
#if N > 3
                # 如果 N 大于 3，则计算 word3 的 CRC
                crc3 = crc_braid_big_table[0][word3 & 0xff];
#if N > 4
                # 如果 N 大于 4，则计算 word4 的 CRC
                crc4 = crc_braid_big_table[0][word4 & 0xff];
#if N > 5
                # 如果 N 大于 5，则计算 word5 的 CRC
                crc5 = crc_braid_big_table[0][word5 & 0xff];
#endif
#endif
#endif
#endif
#endif
                for (k = 1; k < W; k++) {
                    # 对每个 word 的 CRC 进行异或操作
                    crc0 ^= crc_braid_big_table[k][(word0 >> (k << 3)) & 0xff];
#if N > 1
                    crc1 ^= crc_braid_big_table[k][(word1 >> (k << 3)) & 0xff];
#if N > 2
                    crc2 ^= crc_braid_big_table[k][(word2 >> (k << 3)) & 0xff];
#if N > 3
                    crc3 ^= crc_braid_big_table[k][(word3 >> (k << 3)) & 0xff];
#if N > 4
                    crc4 ^= crc_braid_big_table[k][(word4 >> (k << 3)) & 0xff];
#if N > 5
                    crc5 ^= crc_braid_big_table[k][(word5 >> (k << 3)) & 0xff];
#endif
#endif
#endif
#endif
#endif
                }
            }

            /*
              处理最后一个块，同时组合 N 条辫的 CRC。
             */
            # 计算最终的组合 CRC
            comb = crc_word_big(crc0 ^ words[0]);
#if N > 1
            comb = crc_word_big(crc1 ^ words[1] ^ comb);
#if N > 2
            comb = crc_word_big(crc2 ^ words[2] ^ comb);
#if N > 3
            comb = crc_word_big(crc3 ^ words[3] ^ comb);
#if N > 4
            comb = crc_word_big(crc4 ^ words[4] ^ comb);
#if N > 5
            comb = crc_word_big(crc5 ^ words[5] ^ comb);
#endif
#endif
#endif
#endif
#endif
            // 将 N 加到 words 上
            words += N;
            // 对 comb 进行字节交换，并赋值给 crc
            crc = byte_swap(comb);
        }

        /*
          更新指向剩余要处理的字节的指针。
         */
        // 将 words 转换为无符号字符指针，并赋值给 buf
        buf = (unsigned char const *)words;
    }

#endif /* W */

    /* 完成对任何剩余字节的 CRC 计算。 */
    // 当 len 大于等于 8 时，进行循环
    while (len >= 8) {
        len -= 8;
        // 对 crc 和 buf 指向的字节进行异或和位移运算，并赋值给 crc
        crc = (crc >> 8) ^ crc_table[(crc ^ *buf++) & 0xff];
        // 对 crc 和 buf 指向的字节进行异或和位移运算，并赋值给 crc
        crc = (crc >> 8) ^ crc_table[(crc ^ *buf++) & 0xff];
        // 对 crc 和 buf 指向的字节进行异或和位移运算，并赋值给 crc
        crc = (crc >> 8) ^ crc_table[(crc ^ *buf++) & 0xff];
        // 对 crc 和 buf 指向的字节进行异或和位移运算，并赋值给 crc
        crc = (crc >> 8) ^ crc_table[(crc ^ *buf++) & 0xff];
        // 对 crc 和 buf 指向的字节进行异或和位移运算，并赋值给 crc
        crc = (crc >> 8) ^ crc_table[(crc ^ *buf++) & 0xff];
        // 对 crc 和 buf 指向的字节进行异或和位移运算，并赋值给 crc
        crc = (crc >> 8) ^ crc_table[(crc ^ *buf++) & 0xff];
        // 对 crc 和 buf 指向的字节进行异或和位移运算，并赋值给 crc
        crc = (crc >> 8) ^ crc_table[(crc ^ *buf++) & 0xff];
        // 对 crc 和 buf 指向的字节进行异或和位移运算，并赋值给 crc
        crc = (crc >> 8) ^ crc_table[(crc ^ *buf++) & 0xff];
    }
    // 当 len 大于 0 时，进行循环
    while (len) {
        len--;
        // 对 crc 和 buf 指向的字节进行异或和位移运算，并赋值给 crc
        crc = (crc >> 8) ^ crc_table[(crc ^ *buf++) & 0xff];
    }

    /* 返回 CRC，后置条件。 */
    // 返回 crc 与 0xffffffff 进行异或运算的结果
    return crc ^ 0xffffffff;
}

#endif

/* ========================================================================= */
// 计算给定缓冲区的 CRC
unsigned long ZEXPORT crc32(crc, buf, len)
    unsigned long crc;
    const unsigned char FAR *buf;
    uInt len;
{
    // 调用 crc32_z 函数，并返回结果
    return crc32_z(crc, buf, len);
}

/* ========================================================================= */
// 合并两个 CRC
uLong ZEXPORT crc32_combine64(crc1, crc2, len2)
    uLong crc1;
    uLong crc2;
    z_off64_t len2;
{
#ifdef DYNAMIC_CRC_TABLE
    // 如果动态 CRC 表为真，则调用 once 函数，并传入 make_crc_table 函数
    once(&made, make_crc_table);
#endif /* DYNAMIC_CRC_TABLE */
    // 返回对 len2 进行一系列数学运算后与 crc2 进行异或运算的结果
    return multmodp(x2nmodp(len2, 3), crc1) ^ (crc2 & 0xffffffff);
}

/* ========================================================================= */
// 合并两个 CRC
uLong ZEXPORT crc32_combine(crc1, crc2, len2)
    uLong crc1;
    uLong crc2;
    z_off_t len2;
{
    // 调用 crc32_combine64 函数，并返回结果
    return crc32_combine64(crc1, crc2, (z_off64_t)len2);
}

/* ========================================================================= */
// 生成 64 位 CRC
uLong ZEXPORT crc32_combine_gen64(len2)
    z_off64_t len2;
{
#ifdef DYNAMIC_CRC_TABLE
    #ifdef 指令，如果定义了 DYNAMIC_CRC_TABLE，则执行下面的代码
    once(&made, make_crc_table);
    #endif /* DYNAMIC_CRC_TABLE */
#endif /* 如果没有定义 DYNAMIC_CRC_TABLE，则忽略该部分代码 */
    return x2nmodp(len2, 3);
}  # 返回 x2nmodp(len2, 3) 的结果

/* ========================================================================= */
uLong ZEXPORT crc32_combine_gen(len2)
    z_off_t len2;
{
    # 返回 crc32_combine_gen64((z_off64_t)len2) 的结果
    return crc32_combine_gen64((z_off64_t)len2);
}

/* ========================================================================= */
uLong ZEXPORT crc32_combine_op(crc1, crc2, op)
    uLong crc1;
    uLong crc2;
    uLong op;
{
    # 返回 multmodp(op, crc1) 与 (crc2 & 0xffffffff) 的异或结果
    return multmodp(op, crc1) ^ (crc2 & 0xffffffff);
}
```