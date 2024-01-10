# `nmap\libz\zutil.c`

```
/* zutil.c -- 压缩库的目标相关实用函数
 * 版权所有 (C) 1995-2017 Jean-loup Gailly
 * 发行和使用条件，请参见 zlib.h 中的版权声明
 */

/* @(#) $Id$ */

#include "zutil.h"
#ifndef Z_SOLO
#  include "gzguts.h"
#endif

// 错误消息数组
z_const char * const z_errmsg[10] = {
    (z_const char *)"need dictionary",     /* Z_NEED_DICT       2  */
    (z_const char *)"stream end",          /* Z_STREAM_END      1  */
    (z_const char *)"",                    /* Z_OK              0  */
    (z_const char *)"file error",          /* Z_ERRNO         (-1) */
    (z_const char *)"stream error",        /* Z_STREAM_ERROR  (-2) */
    (z_const char *)"data error",          /* Z_DATA_ERROR    (-3) */
    (z_const char *)"insufficient memory", /* Z_MEM_ERROR     (-4) */
    (z_const char *)"buffer error",        /* Z_BUF_ERROR     (-5) */
    (z_const char *)"incompatible version",/* Z_VERSION_ERROR (-6) */
    (z_const char *)""
};

// 返回 zlib 版本号
const char * ZEXPORT zlibVersion()
{
    return ZLIB_VERSION;
}

// 返回 zlib 编译标志
uLong ZEXPORT zlibCompileFlags()
{
    uLong flags;

    flags = 0;
    switch ((int)(sizeof(uInt))) {
    case 2:     break;
    case 4:     flags += 1;     break;
    case 8:     flags += 2;     break;
    default:    flags += 3;
    }
    switch ((int)(sizeof(uLong))) {
    case 2:     break;
    case 4:     flags += 1 << 2;        break;
    case 8:     flags += 2 << 2;        break;
    default:    flags += 3 << 2;
    }
    switch ((int)(sizeof(voidpf))) {
    case 2:     break;
    case 4:     flags += 1 << 4;        break;
    case 8:     flags += 2 << 4;        break;
    default:    flags += 3 << 4;
    }
    switch ((int)(sizeof(z_off_t))) {
    case 2:     break;
    case 4:     flags += 1 << 6;        break;
    case 8:     flags += 2 << 6;        break;
    default:    flags += 3 << 6;
    }
#ifdef ZLIB_DEBUG
    flags += 1 << 8;
#endif
    /*
#if defined(ASMV) || defined(ASMINF)
    flags += 1 << 9;
#endif
     */
#ifdef ZLIB_WINAPI
    # 将标志位左移10位，用于设置特定的标志位
    flags += 1 << 10;
#endif
#ifdef BUILDFIXED
    # 将标志位左移12位，用于设置特定的标志位
    flags += 1 << 12;
#endif
#ifdef DYNAMIC_CRC_TABLE
    # 将标志位左移13位，用于设置特定的标志位
    flags += 1 << 13;
#endif
#ifdef NO_GZCOMPRESS
    # 将标志位左移16位，用于设置特定的标志位
    flags += 1L << 16;
#endif
#ifdef NO_GZIP
    # 将标志位左移17位，用于设置特定的标志位
    flags += 1L << 17;
#endif
#ifdef PKZIP_BUG_WORKAROUND
    # 将标志位左移20位，用于设置特定的标志位
    flags += 1L << 20;
#endif
#ifdef FASTEST
    # 将标志位左移21位，用于设置特定的标志位
    flags += 1L << 21;
#endif
#if defined(STDC) || defined(Z_HAVE_STDARG_H)
#  ifdef NO_vsnprintf
    # 将标志位左移25位，用于设置特定的标志位
    flags += 1L << 25;
#    ifdef HAS_vsprintf_void
    # 将标志位左移26位，用于设置特定的标志位
    flags += 1L << 26;
#    endif
#  else
#    ifdef HAS_vsnprintf_void
    # 将标志位左移26位，用于设置特定的标志位
    flags += 1L << 26;
#    endif
#  endif
#else
    # 将标志位左移24位，用于设置特定的标志位
    flags += 1L << 24;
#  ifdef NO_snprintf
    # 将标志位左移25位，用于设置特定的标志位
    flags += 1L << 25;
#    ifdef HAS_sprintf_void
    # 将标志位左移26位，用于设置特定的标志位
    flags += 1L << 26;
#    endif
#  else
#    ifdef HAS_snprintf_void
    # 将标志位左移26位，用于设置特定的标志位
    flags += 1L << 26;
#    endif
#  endif
#endif
    # 返回设置好的标志位
    return flags;
}

#ifdef ZLIB_DEBUG
#include <stdlib.h>
#  ifndef verbose
#    define verbose 0
#  endif
int ZLIB_INTERNAL z_verbose = verbose;

void ZLIB_INTERNAL z_error(m)
    char *m;
{
    # 输出错误信息到标准错误流
    fprintf(stderr, "%s\n", m);
    # 退出程序
    exit(1);
}
#endif

/* exported to allow conversion of error code to string for compress() and
 * uncompress()
 */
const char * ZEXPORT zError(err)
    int err;
{
    # 返回错误码对应的错误信息
    return ERR_MSG(err);
}

#if defined(_WIN32_WCE) && _WIN32_WCE < 0x800
    /* The older Microsoft C Run-Time Library for Windows CE doesn't have
     * errno.  We define it as a global variable to simplify porting.
     * Its value is always 0 and should not be used.
     */
    # 定义全局变量errno，用于简化移植，其值始终为0，不应使用
    int errno = 0;
#endif

#ifndef HAVE_MEMCPY

void ZLIB_INTERNAL zmemcpy(dest, source, len)
    Bytef* dest;
    const Bytef* source;
    uInt  len;
{
    # 如果长度为0，则直接返回
    if (len == 0) return;
    # 复制数据
    do {
        *dest++ = *source++; /* ??? to be unrolled */
    } while (--len != 0);
}

int ZLIB_INTERNAL zmemcmp(s1, s2, len)
    const Bytef* s1;
    const Bytef* s2;
    uInt  len;
{
    uInt j;
    # 比较两段数据
    for (j = 0; j < len; j++) {
        if (s1[j] != s2[j]) return 2*(s1[j] > s2[j])-1;
    }
    return 0;
}
void ZLIB_INTERNAL zmemzero(dest, len)
    Bytef* dest;
    uInt  len;
{
    // 如果长度为0，则直接返回
    if (len == 0) return;
    // 循环将目标指针指向的内容清零，直到长度为0
    do {
        *dest++ = 0;  /* ??? to be unrolled */
    } while (--len != 0);
}
#endif

#ifndef Z_SOLO

#ifdef SYS16BIT

#ifdef __TURBOC__
/* Turbo C in 16-bit mode */

#  define MY_ZCALLOC

/* Turbo C malloc() does not allow dynamic allocation of 64K bytes
 * and farmalloc(64K) returns a pointer with an offset of 8, so we
 * must fix the pointer. Warning: the pointer must be put back to its
 * original form in order to free it, use zcfree().
 */

#define MAX_PTR 10
/* 10*64K = 640K */

local int next_ptr = 0;

typedef struct ptr_table_s {
    voidpf org_ptr;
    voidpf new_ptr;
} ptr_table;

local ptr_table table[MAX_PTR];
/* This table is used to remember the original form of pointers
 * to large buffers (64K). Such pointers are normalized with a zero offset.
 * Since MSDOS is not a preemptive multitasking OS, this table is not
 * protected from concurrent access. This hack doesn't work anyway on
 * a protected system like OS/2. Use Microsoft C instead.
 */

voidpf ZLIB_INTERNAL zcalloc(voidpf opaque, unsigned items, unsigned size)
{
    voidpf buf;
    ulg bsize = (ulg)items*size;

    (void)opaque;

    /* If we allocate less than 65520 bytes, we assume that farmalloc
     * will return a usable pointer which doesn't have to be normalized.
     */
    if (bsize < 65520L) {
        buf = farmalloc(bsize);
        if (*(ush*)&buf != 0) return buf;
    } else {
        buf = farmalloc(bsize + 16L);
    }
    if (buf == NULL || next_ptr >= MAX_PTR) return NULL;
    table[next_ptr].org_ptr = buf;

    /* Normalize the pointer to seg:0 */
    *((ush*)&buf+1) += ((ush)((uch*)buf-0) + 15) >> 4;
    *(ush*)&buf = 0;
    table[next_ptr++].new_ptr = buf;
    return buf;
}

void ZLIB_INTERNAL zcfree(voidpf opaque, voidpf ptr)
{
    int n;

    (void)opaque;

    if (*(ush*)&ptr != 0) { /* object < 64K */
        farfree(ptr);
        return;
    }
    /* 寻找原始指针 */
    for (n = 0; n < next_ptr; n++) {
        // 如果指针不等于表中的新指针，则继续循环
        if (ptr != table[n].new_ptr) continue;

        // 释放原始指针所指向的内存
        farfree(table[n].org_ptr);
        // 将后续元素向前移动一个位置
        while (++n < next_ptr) {
            table[n-1] = table[n];
        }
        // 更新表中元素的数量
        next_ptr--;
        // 找到指针并完成操作后直接返回
        return;
    }
    // 如果未找到指定的指针，则抛出断言错误
    Assert(0, "zcfree: ptr not found");
#ifdef __TURBOC__
#endif /* __TURBOC__ */

#ifdef M_I86
/* Microsoft C in 16-bit mode */

#  define MY_ZCALLOC

#if (!defined(_MSC_VER) || (_MSC_VER <= 600))
#  define _halloc  halloc
#  define _hfree   hfree
#endif

// 为 zlib 分配内存
voidpf ZLIB_INTERNAL zcalloc(voidpf opaque, uInt items, uInt size)
{
    (void)opaque;
    return _halloc((long)items, size);
}

// 释放 zlib 内存
void ZLIB_INTERNAL zcfree(voidpf opaque, voidpf ptr)
{
    (void)opaque;
    _hfree(ptr);
}

#endif /* M_I86 */

#endif /* SYS16BIT */


#ifndef MY_ZCALLOC /* Any system without a special alloc function */

#ifndef STDC
extern voidp  malloc OF((uInt size));
extern voidp  calloc OF((uInt items, uInt size));
extern void   free   OF((voidpf ptr));
#endif

// 为 zlib 分配内存
voidpf ZLIB_INTERNAL zcalloc(opaque, items, size)
    voidpf opaque;
    unsigned items;
    unsigned size;
{
    (void)opaque;
    return sizeof(uInt) > 2 ? (voidpf)malloc(items * size) :
                              (voidpf)calloc(items, size);
}

// 释放 zlib 内存
void ZLIB_INTERNAL zcfree(opaque, ptr)
    voidpf opaque;
    voidpf ptr;
{
    (void)opaque;
    free(ptr);
}

#endif /* MY_ZCALLOC */

#endif /* !Z_SOLO */
```