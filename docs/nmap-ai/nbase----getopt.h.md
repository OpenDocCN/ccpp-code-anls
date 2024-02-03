# `nmap\nbase\getopt.h`

```cpp
/*
 *  my_getopt.h - interface to my re-implementation of getopt.
 *  Copyright 1997, 2000, 2001, 2002, 2006, Benjamin Sittler
 *
 *  Permission is hereby granted, free of charge, to any person
 *  obtaining a copy of this software and associated documentation
 *  files (the "Software"), to deal in the Software without
 *  restriction, including without limitation the rights to use, copy,
 *  modify, merge, publish, distribute, sublicense, and/or sell copies
 *  of the Software, and to permit persons to whom the Software is
 *  furnished to do so, subject to the following conditions:
 *
 *  The above copyright notice and this permission notice shall be
 *  included in all copies or substantial portions of the Software.
 *
 *  THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
 *  EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
 *  MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
 *  NONINFRINGEMENT.  IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT
 *  HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY,
 *  WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
 *  OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
 *  DEALINGS IN THE SOFTWARE.
 */

/* $Id$ */

// 如果未定义 MY_GETOPT_H_INCLUDED，则定义 MY_GETOPT_H_INCLUDED
#ifndef MY_GETOPT_H_INCLUDED
#define MY_GETOPT_H_INCLUDED

// 如果包含了配置文件，则包含 nbase_config.h，否则根据 WIN32 包含 nbase_winconfig.h
#if HAVE_CONFIG_H
#include "nbase_config.h"
#else
#ifdef WIN32
#include "nbase_winconfig.h"
#endif /* WIN32 */
#endif /* HAVE_CONFIG_H */

#ifdef __cplusplus
extern "C" {
#endif

// 重置参数解析器为初始值
extern int getopt_reset(void);

// 如果未定义 HAVE_GETOPT，则定义 UNIX 风格的短参数解析器
#ifndef HAVE_GETOPT
extern int getopt(int argc, char * argv[], const char *opts);
#endif /* HAVE_GETOPT */

// 定义全局变量 optind, opterr, optopt, optarg
extern int optind, opterr, optopt;
extern char *optarg;

// 定义结构体 option，包含 name, has_arg, flag, val 四个成员
struct option {
  const char *name;
  int has_arg;
  int *flag;
  int val;
};

// 为 has_arg 定义可读性更好的值
#undef no_argument
#define no_argument 0
#undef required_argument
#define required_argument 1
/* 取消定义 optional_argument 宏 */
#undef optional_argument
/* 定义 optional_argument 宏为 2 */
#define optional_argument 2

/* GNU-style long-argument parsers */

/* 解析命令行参数，包括长选项 */
extern int getopt_long(int argc, char * argv[], const char *shortopts,
                       const struct option *longopts, int *longind);

/* 解析命令行参数，只包括长选项 */
extern int getopt_long_only(int argc, char * argv[], const char *shortopts,
                            const struct option *longopts, int *longind);

/* 内部函数，用于解析命令行参数 */
extern int _getopt_internal(int argc, char * argv[], const char *shortopts,
                            const struct option *longopts, int *longind,
                            int long_only);

#ifdef __cplusplus
}
#endif

#endif /* MY_GETOPT_H_INCLUDED */
```