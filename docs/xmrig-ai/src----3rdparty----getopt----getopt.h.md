# `xmrig\src\3rdparty\getopt\getopt.h`

```cpp
#ifndef __GETOPT_H__
/**
 * DISCLAIMER
 * This file is part of the mingw-w64 runtime package.
 *
 * The mingw-w64 runtime package and its code is distributed in the hope that it
 * will be useful but WITHOUT ANY WARRANTY.  ALL WARRANTIES, EXPRESSED OR
 * IMPLIED ARE HEREBY DISCLAIMED.  This includes but is not limited to
 * warranties of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
 */
 /*
 * Copyright (c) 2002 Todd C. Miller <Todd.Miller@courtesan.com>
 *
 * Permission to use, copy, modify, and distribute this software for any
 * purpose with or without fee is hereby granted, provided that the above
 * copyright notice and this permission notice appear in all copies.
 *
 * THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES
 * WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF
 * MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR
 * ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES
 * WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN
 * ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF
 * OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.
 *
 * Sponsored in part by the Defense Advanced Research Projects
 * Agency (DARPA) and Air Force Research Laboratory, Air Force
 * Materiel Command, USAF, under agreement number F39502-99-1-0512.
 */
#endif
/*-
 * 版权声明
 * 本代码源自Dieter Baron和Thomas Klausner为NetBSD Foundation贡献的软件。
 * 在遵守以下条件的情况下，允许以源代码和二进制形式重新分发和使用：
 * 1. 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明。
 * 2. 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明。
 * 本软件由NetBSD Foundation和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。在任何情况下，无论是在合同、严格责任还是侵权（包括疏忽或其他情况）的情况下，基金会或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责，即使已被告知可能发生此类损害。
 */

#pragma warning(disable:4996)  // 禁用特定警告

#define __GETOPT_H__  // 定义__GETOPT_H__

/* 所有头文件都包含此文件。*/
#include <crtdefs.h>
#include <errno.h>
#include <stdlib.h>
#include <string.h>
#include <stdarg.h>
#include <stdio.h>
#include <windows.h>

#ifdef __cplusplus
extern "C" {
#endif

#define    REPLACE_GETOPT        /* use this getopt as the system getopt(3) */

#ifdef REPLACE_GETOPT
int    opterr = 1;        /* if error message should be printed */
int    optind = 1;        /* index into parent argv vector */
int    optopt = '?';        /* character checked for validity */
#undef    optreset        /* see getopt.h */
#define    optreset        __mingw_optreset
int    optreset;        /* reset getopt */
char    *optarg;        /* argument associated with option */
#endif

//extern int optind;        /* index of first non-option in argv      */
//extern int optopt;        /* single option character, as parsed     */
//extern int opterr;        /* flag to enable built-in diagnostics... */
//                /* (user may set to zero, to suppress)    */
//
//extern char *optarg;        /* pointer to argument of current option  */

#define PRINT_ERROR    ((opterr) && (*options != ':'))

#define FLAG_PERMUTE    0x01    /* permute non-options to the end of argv */
#define FLAG_ALLARGS    0x02    /* treat non-options as args to option "-1" */
#define FLAG_LONGONLY    0x04    /* operate as getopt_long_only */

/* return values */
#define    BADCH        (int)'?'
#define    BADARG        ((*options == ':') ? (int)':' : (int)'?')
#define    INORDER     (int)1

#ifndef __CYGWIN__
#define __progname __argv[0]
#else
extern char __declspec(dllimport) *__progname;
#endif

static char EMSG[] = "";

static int getopt_internal(int, char * const *, const char *,
               const struct option *, int *, int);
static int parse_long_options(char * const *, const char *,
                  const struct option *, int *, int);
static int gcd(int, int);
static void permute_args(int, int, int, char * const *);

static char *place = EMSG; /* option letter processing */

/* XXX: set optreset to 1 rather than these two */
static int nonopt_start = -1; /* first non option argument (for permute) */
static int nonopt_end = -1;   /* first option after non options (for permute) */

/* Error messages */
static const char recargchar[] = "option requires an argument -- %c";
static const char recargstring[] = "option requires an argument -- %s";
// 定义常量字符串，表示不明确的选项
static const char ambig[] = "ambiguous option -- %.*s";
// 定义常量字符串，表示选项不接受参数
static const char noarg[] = "option doesn't take an argument -- %.*s";
// 定义常量字符串，表示未知选项
static const char illoptchar[] = "unknown option -- %c";
// 定义常量字符串，表示未知选项
static const char illoptstring[] = "unknown option -- %s";

// 定义函数_vwarnx，用于在标准错误流中输出格式化的警告信息
static void
_vwarnx(const char *fmt,va_list ap)
{
  // 输出程序名称到标准错误流
  (void)fprintf(stderr,"%s: ",__progname);
  // 如果格式不为空，则使用可变参数列表输出格式化字符串到标准错误流
  if (fmt != NULL)
    (void)vfprintf(stderr,fmt,ap);
  // 输出换行符到标准错误流
  (void)fprintf(stderr,"\n");
}

// 定义函数warnx，用于在标准错误流中输出格式化的警告信息
static void
warnx(const char *fmt,...)
{
  va_list ap;
  // 初始化可变参数列表
  va_start(ap,fmt);
  // 调用_vwarnx函数输出警告信息
  _vwarnx(fmt,ap);
  // 结束可变参数列表的使用
  va_end(ap);
}

/*
 * 计算a和b的最大公约数
 */
static int
gcd(int a, int b)
{
    int c;

    // 计算a对b的取模
    c = a % b;
    // 当c不为0时循环
    while (c != 0) {
        a = b;
        b = c;
        c = a % b;
    }

    return (b);
}

/*
 * 交换从nonopt_start到nonopt_end的块与从nonopt_end到opt_end的块
 * （保持每个块中参数的顺序不变）
 */
static void
permute_args(int panonopt_start, int panonopt_end, int opt_end,
    char * const *nargv)
{
    int cstart, cyclelen, i, j, ncycle, nnonopts, nopts, pos;
    char *swap;

    /*
     * 计算块的长度和循环的数量和大小
     */
    nnonopts = panonopt_end - panonopt_start;
    nopts = opt_end - panonopt_end;
    ncycle = gcd(nnonopts, nopts);
    cyclelen = (opt_end - panonopt_start) / ncycle;

    for (i = 0; i < ncycle; i++) {
        cstart = panonopt_end+i;
        pos = cstart;
        for (j = 0; j < cyclelen; j++) {
            if (pos >= panonopt_end)
                pos -= nnonopts;
            else
                pos += nopts;
            swap = nargv[pos];
            // 强制类型转换，交换参数位置
            ((char **) nargv)[pos] = nargv[cstart];
            // 强制类型转换，交换参数位置
            ((char **)nargv)[cstart] = swap;
        }
    }
}

#ifdef REPLACE_GETOPT
/*
 * getopt --
 *    解析argc/argv参数向量
 *
 * [最终将替换BSD getopt]
 */
int
// 定义 getopt 函数，接受参数个数、参数数组和选项字符串
getopt(int nargc, char * const *nargv, const char *options)
{

    /*
     * 我们不向 getopt_internal() 传递 FLAG_PERMUTE，因为 BSD 的 getopt(3)（不像 GNU）从来没有这样做过。
     *
     * 此外，由于许多特权程序在放弃特权之前调用 getopt()，因此保持事情尽可能简单（和无错误）是有意义的。
     */
    return (getopt_internal(nargc, nargv, options, NULL, NULL, 0));
}
#endif /* REPLACE_GETOPT */

//extern int getopt(int nargc, char * const *nargv, const char *options);

#ifdef _BSD_SOURCE
/*
 * BSD 添加了非标准的 `optreset' 特性，用于重新初始化 `getopt' 解析。我们支持此特性，用于宣称其 BSD 继承权的应用程序，在包含此头文件之前；然而，为了保持可移植性，建议开发人员避免使用它。
 */
# define optreset  __mingw_optreset
extern int optreset;
#endif
#ifdef __cplusplus
}
#endif
/*
 * POSIX 要求在 `unistd.h' 中指定 `getopt' API；因此，`unistd.h' 包含了此头文件。然而，当以这种方式包含时，我们不希望暴露 `getopt_long' 或 `getopt_long_only' API。因此，在此关闭标准的 __GETOPT_H__ 声明块，并在 *不是* __UNISTD_H_SOURCED__ 时打开额外的 __GETOPT_LONG_H__ 特定块，以声明扩展的 API。
 */
#endif /* !defined(__GETOPT_H__) */

#if !defined(__UNISTD_H_SOURCED__) && !defined(__GETOPT_LONG_H__)
#define __GETOPT_LONG_H__

#ifdef __cplusplus
extern "C" {
#endif

// 结构体 option，用于指定长格式选项
struct option        /* specification for a long form option...    */
{
  const char *name;        /* option name, without leading hyphens */
  int         has_arg;        /* does it take an argument?        */
  int        *flag;        /* where to save its status, or NULL    */
  int         val;        /* its associated status value        */
};

enum            /* permitted values for its `has_arg' field...    */
{
  no_argument = 0,          /* option never takes an argument    */
  required_argument,        /* option always requires an argument    */
  optional_argument        /* option may take an argument        */
};

/*
 * parse_long_options --
 *    Parse long options in argc/argv argument vector.
 * Returns -1 if short_too is set and the option does not match long_options.
 */
static int
parse_long_options(char * const *nargv, const char *options,
    const struct option *long_options, int *idx, int short_too)
{
    char *current_argv, *has_equal;
    size_t current_argv_len;
    int i, ambiguous, match;

#define IDENTICAL_INTERPRETATION(_x, _y)                                \
    (long_options[(_x)].has_arg == long_options[(_y)].has_arg &&    \
     long_options[(_x)].flag == long_options[(_y)].flag &&          \
     long_options[(_x)].val == long_options[(_y)].val)

    current_argv = place;  /* 设置当前参数为 place */
    match = -1;  /* 匹配结果初始化为-1 */
    ambiguous = 0;  /* 歧义标志初始化为0 */

    optind++;  /* 增加 optind 的值 */

    if ((has_equal = strchr(current_argv, '=')) != NULL) {
        /* argument found (--option=arg) */
        current_argv_len = has_equal - current_argv;  /* 计算参数长度 */
        has_equal++;
    } else
        current_argv_len = strlen(current_argv);  /* 计算参数长度 */

    for (i = 0; long_options[i].name; i++) {
        /* find matching long option */
        if (strncmp(current_argv, long_options[i].name,
            current_argv_len))
            continue;  /* 如果当前参数与选项名不匹配，则继续下一个选项 */

        if (strlen(long_options[i].name) == current_argv_len) {
            /* exact match */
            match = i;  /* 完全匹配，记录匹配的选项索引 */
            ambiguous = 0;  /* 歧义标志置为0 */
            break;
        }
        /*
         * If this is a known short option, don't allow
         * a partial match of a single character.
         */
        if (short_too && current_argv_len == 1)
            continue;  /* 如果允许短选项，并且当前参数长度为1，则继续下一个选项 */

        if (match == -1)    /* partial match */
            match = i;  /* 记录部分匹配的选项索引 */
        else if (!IDENTICAL_INTERPRETATION(i, match))
            ambiguous = 1;  /* 如果匹配不唯一，则设置歧义标志为1 */
    }
    if (ambiguous) {
        /* 如果存在歧义的缩写 */
        if (PRINT_ERROR)
            warnx(ambig, (int)current_argv_len,
                 current_argv);
        optopt = 0;
        return (BADCH);
    }
    if (match != -1) {        /* 找到选项 */
        if (long_options[match].has_arg == no_argument
            && has_equal) {
            if (PRINT_ERROR)
                warnx(noarg, (int)current_argv_len,
                     current_argv);
            /*
             * XXX: GNU sets optopt to val regardless of flag
             */
            if (long_options[match].flag == NULL)
                optopt = long_options[match].val;
            else
                optopt = 0;
            return (BADARG);
        }
        if (long_options[match].has_arg == required_argument ||
            long_options[match].has_arg == optional_argument) {
            if (has_equal)
                optarg = has_equal;
            else if (long_options[match].has_arg ==
                required_argument) {
                /*
                 * 可选参数不使用下一个 nargv
                 */
                optarg = nargv[optind++];
            }
        }
        if ((long_options[match].has_arg == required_argument)
            && (optarg == NULL)) {
            /*
             * 缺少参数；前导 ':' 表示不应生成错误。
             */
            if (PRINT_ERROR)
                warnx(recargstring,
                    current_argv);
            /*
             * XXX: GNU sets optopt to val regardless of flag
             */
            if (long_options[match].flag == NULL)
                optopt = long_options[match].val;
            else
                optopt = 0;
            --optind;
            return (BADARG);
        }
    } else {            /* 未知选项 */
        if (short_too) {
            --optind;
            return (-1);
        }
        if (PRINT_ERROR)
            warnx(illoptstring, current_argv);
        optopt = 0;
        return (BADCH);
    }
    if (idx)
        *idx = match;
    if (long_options[match].flag) {
        *long_options[match].flag = long_options[match].val;
        return (0);
    } else
        return (long_options[match].val);
/*
 * getopt_internal --
 *    解析 argc/argv 参数向量。被用户级别的程序调用。
 */
static int
getopt_internal(int nargc, char * const *nargv, const char *options,
    const struct option *long_options, int *idx, int flags)
{
    char *oli;                /* 选项字母列表索引 */
    int optchar, short_too;
    static int posixly_correct = -1;

    if (options == NULL)
        return (-1);

    /*
     * XXX 一些 GNU 程序（比如 cvs）将 optind 设置为 0 而不是使用 optreset。需要解决这个问题。
     */
    if (optind == 0)
        optind = optreset = 1;

    /*
     * 如果 POSIXLY_CORRECT 被设置或者选项字符串以 '+' 开头，则禁用 GNU 扩展。
     *
     * CV, 2009-12-14: 如果 optind == 0 或者 optreset != 0，为了 GNU 兼容性，重新检查 POSIXLY_CORRECT。
     */
    if (posixly_correct == -1 || optreset != 0)
        posixly_correct = (getenv("POSIXLY_CORRECT") != NULL);
    if (*options == '-')
        flags |= FLAG_ALLARGS;
    else if (posixly_correct || *options == '+')
        flags &= ~FLAG_PERMUTE;
    if (*options == '+' || *options == '-')
        options++;

    optarg = NULL;
    if (optreset)
        nonopt_start = nonopt_end = -1;
start:
    }

    /*
     * 如果：
     *  1) 我们传递了一些长选项
     *  2) 参数不只是 "-"
     *  3) 要么参数以 -- 开头，要么我们是在使用 getopt_long_only()
     */
    # 如果长选项不为空且当前位置不是参数的结尾，并且当前位置是'-'或者标志包含FLAG_LONGONLY
    if (long_options != NULL && place != nargv[optind] &&
        (*place == '-' || (flags & FLAG_LONGONLY))) {
        # 禁止解析短选项
        short_too = 0;
        # 如果当前位置是'-'，则跳过
        if (*place == '-')
            place++;        /* --foo long option */
        # 如果当前位置不是':'并且在选项中找到当前位置的字符，则该选项也可能是短选项
        else if (*place != ':' && strchr(options, *place) != NULL)
            short_too = 1;        /* could be short option too */

        # 解析长选项
        optchar = parse_long_options(nargv, options, long_options,
            idx, short_too);
        # 如果解析成功，则将当前位置置为空，并返回解析结果
        if (optchar != -1) {
            place = EMSG;
            return (optchar);
        }
    }

    # 如果当前位置的字符是':'，或者是'-'并且下一个位置不为空，或者在选项中找不到当前位置的字符
    if ((optchar = (int)*place++) == (int)':' ||
        (optchar == (int)'-' && *place != '\0') ||
        (oli = (char*)strchr(options, optchar)) == NULL) {
        '''
         * 如果用户指定了"-"并且'-'没有在选项中列出，
         * 则根据 POSIX 返回-1（非选项）。
         * 否则，它是一个未知的选项字符（或':'）。
         '''
        # 如果当前位置的字符是'-'并且下一个位置为空，则返回-1
        if (optchar == (int)'-' && *place == '\0')
            return (-1);
        # 如果当前位置为空，则增加optind
        if (!*place)
            ++optind;
        # 如果PRINT_ERROR为真，则打印错误信息
        if (PRINT_ERROR)
            warnx(illoptchar, optchar);
        # 设置optopt为当前位置的字符，并返回BADCH
        optopt = optchar;
        return (BADCH);
    }
    # 如果长选项不为空且当前位置的字符是'W'并且oli的下一个字符是';'
    if (long_options != NULL && optchar == 'W' && oli[1] == ';') {
        # -W长选项
        # 如果当前位置不为空，则什么也不做
        if (*place)            /* no space */
            /* NOTHING */;
        # 如果optind增加后大于等于nargc，则当前位置为空，打印错误信息，设置optopt并返回BADARG
        else if (++optind >= nargc) {    /* no arg */
            place = EMSG;
            if (PRINT_ERROR)
                warnx(recargchar, optchar);
            optopt = optchar;
            return (BADARG);
        # 否则，当前位置为nargv[optind]
        } else                /* white space */
            place = nargv[optind];
        # 解析长选项，将当前位置置为空，并返回解析结果
        optchar = parse_long_options(nargv, options, long_options,
            idx, 0);
        place = EMSG;
        return (optchar);
    }
    # 如果oli的下一个字符不是':'，则不需要参数
    if (*++oli != ':') {            /* doesn't take argument */
        # 如果当前位置为空，则增加optind
        if (!*place)
            ++optind;
    } else {                /* 如果有（可选的）参数 */
        optarg = NULL;       /* 参数值初始化为空 */
        if (*place)            /* 如果没有空白字符 */
            optarg = place;    /* 参数值为当前位置 */
        else if (oli[1] != ':') {    /* 参数不是可选的 */
            if (++optind >= nargc) {    /* 没有参数 */
                place = EMSG;    /* 位置置为错误信息 */
                if (PRINT_ERROR)
                    warnx(recargchar, optchar);    /* 打印警告信息 */
                optopt = optchar;    /* 设置选项字符 */
                return (BADARG);    /* 返回参数错误 */
            } else
                optarg = nargv[optind];    /* 参数值为下一个参数 */
        }
        place = EMSG;    /* 位置置为错误信息 */
        ++optind;    /* 下一个参数索引加一 */
    }
    /* 返回选项字符 */
    return (optchar);
// 结束 C++ 的 extern "C" 块
#ifdef __cplusplus
}
#endif

// 结束 ifndef 条件编译块
#endif /* !defined(__UNISTD_H_SOURCED__) && !defined(__GETOPT_LONG_H__) */

// 如果没有定义 HAVE_DECL_GETOPT，则定义为 1，用于兼容性
#ifndef HAVE_DECL_GETOPT
# define HAVE_DECL_GETOPT    1
#endif

// getopt_long_only 函数，解析 argc/argv 参数向量
int
getopt_long_only(int nargc, char * const *nargv, const char *options,
    const struct option *long_options, int *idx)
{
    // 调用 getopt_internal 函数，传入参数和标志 FLAG_PERMUTE|FLAG_LONGONLY
    return (getopt_internal(nargc, nargv, options, long_options, idx,
        FLAG_PERMUTE|FLAG_LONGONLY));
}

// getopt_long 函数，解析 argc/argv 参数向量
int
getopt_long(int nargc, char * const *nargv, const char *options,
    const struct option *long_options, int *idx)
{
    // 调用 getopt_internal 函数，传入参数和标志 FLAG_PERMUTE
    return (getopt_internal(nargc, nargv, options, long_options, idx,
        FLAG_PERMUTE));
}
```