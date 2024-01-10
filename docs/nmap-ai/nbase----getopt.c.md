# `nmap\nbase\getopt.c`

```
/*
 *  my_getopt.c - my re-implementation of getopt.
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

#include <sys/types.h>
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include "getopt.h"

#if HAVE_CONFIG_H
#include "nbase_config.h"
#else
#ifdef WIN32
#include "nbase_winconfig.h" /* mainly for _CRT_SECURE_NO_DEPRECATE */
#endif /* WIN32 */
#endif /* HAVE_CONFIG_H */

int optind=1, opterr=1, optopt=0;
char *optarg=0;

/* reset argument parser to start-up values */
int getopt_reset(void)
{
    optind = 1;  // 重置参数解析器的起始索引
    opterr = 1;  // 设置错误报告标志
    optopt = 0;  // 设置选项字符
    optarg = 0;  // 重置选项参数
    return 0;  // 返回成功
}


/* this is the plain old UNIX getopt, with GNU-style extensions. */
/* if you're porting some piece of UNIX software, this is all you need. */
/* this supports GNU-style permutation and optional arguments */

static int _getopt(int argc, char * argv[], const char *opts)
{
  # 定义静态变量 charind，用于记录当前处理的参数字符索引
  static int charind=0;
  # 定义指向常量字符的指针 s，以及字符 mode 和 colon_mode
  const char *s;
  char mode, colon_mode;
  # 定义整型变量 off 和 opt，初始化 off 为 0，opt 为 -1
  int off = 0, opt = -1;

  # 如果环境变量 POSIXLY_CORRECT 存在，则设置 colon_mode 和 mode 为 '+'
  if(getenv("POSIXLY_CORRECT")) colon_mode = mode = '+';
  # 否则根据 opts 的值设置 colon_mode 和 mode
  else {
    # 如果 opts 的第一个字符是冒号，则 off 加一
    if((colon_mode = *opts) == ':') off ++;
    # 获取 mode 的值，并根据其值设置 colon_mode
    if(((mode = opts[off]) == '+') || (mode == '-')) {
      off++;
      if((colon_mode != ':') && ((colon_mode = opts[off]) == ':'))
        off ++;
    }
  }
  # 初始化 optarg 为 0
  optarg = 0;
  # 如果 charind 不为 0，则处理当前参数
  if(charind) {
    # 获取当前参数的字符并赋值给 optopt
    optopt = argv[optind][charind];
    # 遍历 opts+off，查找匹配的字符
    for(s=opts+off; *s; s++) if(optopt == *s) {
      charind++;
      # 如果下一个字符是冒号，或者当前字符是 'W' 且下一个字符是分号，则处理参数值
      if((*(++s) == ':') || ((optopt == 'W') && (*s == ';'))) {
        if(argv[optind][charind]) {
          optarg = &(argv[optind++][charind]);
          charind = 0;
        } else if(*(++s) != ':') {
          charind = 0;
          # 如果参数值为空且下一个字符不是冒号，则处理下一个参数
          if(++optind >= argc) {
            if(opterr) fprintf(stderr,
                                "%s: option requires an argument -- %c\n",
                                argv[0], optopt);
            opt = (colon_mode == ':') ? ':' : '?';
            goto my_getopt_ok;
          }
          optarg = argv[optind++];
        }
      }
      opt = optopt;
      goto my_getopt_ok;
    }
    # 如果未找到匹配的字符，则输出错误信息
    if(opterr) fprintf(stderr,
                        "%s: illegal option -- %c\n",
                        argv[0], optopt);
    opt = '?';
    # 如果当前参数的下一个字符为空，则处理下一个参数
    if(argv[optind][++charind] == '\0') {
      optind++;
      charind = 0;
    }
  my_getopt_ok:
    # 如果 charind 不为 0 且当前参数的下一个字符为空，则处理下一个参数
    if(charind && ! argv[optind][charind]) {
      optind++;
      charind = 0;
    }
  } else if((optind >= argc) ||
             ((argv[optind][0] == '-') &&
              (argv[optind][1] == '-') &&
              (argv[optind][2] == '\0'))) {
    optind++;
    opt = -1;
  } else if((argv[optind][0] != '-') ||
             (argv[optind][1] == '\0')) {
    char *tmp;
    int i, j, k;

    # 根据 mode 的值设置 opt
    if(mode == '+') opt = -1;
    else if(mode == '-') {
      optarg = argv[optind++];
      charind = 0;
      opt = 1;
    } else {
      # 遍历命令行参数，找到以'-'开头且不为空的参数
      for(i=j=optind; i<argc; i++) if((argv[i][0] == '-') &&
                                        (argv[i][1] != '\0')) {
        # 将 optind 设置为当前参数的索引，调用 _getopt 函数获取下一个选项
        optind=i;
        opt=_getopt(argc, argv, opts);
        # 将当前参数移到参数列表的最前面
        while(i > j) {
          tmp=argv[--i];
          for(k=i; k+1<optind; k++) argv[k]=argv[k+1];
          argv[--optind]=tmp;
        }
        break;
      }
      # 如果遍历完所有参数后，optind 仍然等于 argc，则将 opt 设置为 -1
      if(i == argc) opt = -1;
    }
  } else {
    # 将 charind 自增，调用 _getopt 函数获取下一个选项
    charind++;
    opt = _getopt(argc, argv, opts);
  }
  # 如果 optind 大于 argc，则将 optind 设置为 argc
  if (optind > argc) optind = argc;
  # 返回获取的选项
  return opt;
/* 这是扩展的 getopt_long{,_only} 函数，具有一些类似 GNU 的扩展功能。实现 _getopt_internal 以便任何期望 GNU libc getopt 调用它的程序。 */

int _getopt_internal(int argc, char * argv[], const char *shortopts,
                     const struct option *longopts, int *longind,
                     int long_only)
{
  char mode, colon_mode = *shortopts;  // 定义变量 mode 和 colon_mode，初始化 colon_mode 为 shortopts 的第一个字符
  int shortoff = 0, opt = -1;  // 定义变量 shortoff 和 opt，初始化 shortoff 为 0，opt 为 -1

  if(getenv("POSIXLY_CORRECT")) colon_mode = mode = '+';  // 如果环境变量 POSIXLY_CORRECT 存在，则将 colon_mode 和 mode 都设置为 '+'
  else {
    if((colon_mode = *shortopts) == ':') shortoff ++;  // 如果 shortopts 的第一个字符是 ':'，则将 colon_mode 设置为 shortopts 的第一个字符，并将 shortoff 加一
    if(((mode = shortopts[shortoff]) == '+') || (mode == '-')) {  // 如果 shortopts 的第 shortoff 位置的字符是 '+' 或 '-'，则将 mode 设置为该字符
      shortoff++;
      if((colon_mode != ':') && ((colon_mode = shortopts[shortoff]) == ':'))  // 如果 colon_mode 不是 ':'，且 shortopts 的第 shortoff 位置的字符是 ':'，则将 colon_mode 设置为该字符，并将 shortoff 加一
        shortoff ++;
    }
  }
  optarg = 0;  // 将全局变量 optarg 设置为 0
  if((optind >= argc) ||  // 如果 optind 大于等于 argc，或者
      ((argv[optind][0] == '-') &&
       (argv[optind][1] == '-') &&
       (argv[optind][2] == '\0'))) {  // argv[optind] 的第一个字符是 '-'，第二个字符是 '-'，第三个字符是 '\0'
    optind++;  // 将 optind 加一
    opt = -1;  // 将 opt 设置为 -1
  } else if((argv[optind][0] != '-') ||  // 如果 argv[optind] 的第一个字符不是 '-'，或者
            (argv[optind][1] == '\0')) {  // argv[optind] 的第二个字符是 '\0'
    char *tmp;  // 定义指针变量 tmp
    int i, j, k;  // 定义整型变量 i、j、k

    opt = -1;  // 将 opt 设置为 -1
    if(mode == '+') return -1;  // 如果 mode 是 '+'，则返回 -1
    else if(mode == '-') {  // 否则，如果 mode 是 '-'
      optarg = argv[optind++];  // 将 optarg 设置为 argv[optind]，并将 optind 加一
      return 1;  // 返回 1
    }
    for(i=j=optind; i<argc; i++) if((argv[i][0] == '-') &&  // 遍历 argv，如果 argv[i] 的第一个字符是 '-'，且
                                    (argv[i][1] != '\0')) {  // argv[i] 的第二个字符不是 '\0'
      optind=i;  // 将 optind 设置为 i
      opt=_getopt_internal(argc, argv, shortopts,  // 调用 _getopt_internal 函数
                           longopts, longind,
                           long_only);
      while(i > j) {  // 循环，直到 i 大于 j
        tmp=argv[--i];  // 将 tmp 设置为 argv[i]，并将 i 减一
        for(k=i; k+1<optind; k++)  // 循环，直到 k+1 小于 optind
          argv[k]=argv[k+1];  // 将 argv[k] 设置为 argv[k+1]
        argv[--optind]=tmp;  // 将 argv[optind] 设置为 tmp，同时将 optind 减一
      }
      break;  // 跳出循环
    }
  } else if((!long_only) && (argv[optind][1] != '-'))  // 否则，如果不是 long_only，且 argv[optind] 的第二个字符不是 '-'
    opt = _getopt(argc, argv, shortopts);  // 调用 _getopt 函数
  else {
    int charind, offset;  // 定义整型变量 charind 和 offset
    int found = 0, ind, hits = 0;  // 定义整型变量 found、ind、hits
    # 检查命令行参数是否为短选项
    if(((optopt = argv[optind][1]) != '-') && ! argv[optind][2]) {
      # 定义变量c
      int c;
      # 设置shortopts的偏移量
      ind = shortoff;
      # 遍历shortopts，查找匹配的选项
      while((c = shortopts[ind++])) {
        # 检查是否为带参数的选项
        if(((shortopts[ind] == ':') ||
            ((c == 'W') && (shortopts[ind] == ';'))) &&
           (shortopts[++ind] == ':'))
          ind ++;
        # 如果匹配到选项，则返回
        if(optopt == c) return _getopt(argc, argv, shortopts);
      }
    }
    # 设置偏移量
    offset = 2 - (argv[optind][1] != '-');
    # 遍历命令行参数，查找匹配的长选项
    for(charind = offset;
        (argv[optind][charind] != '\0') &&
          (argv[optind][charind] != '=');
        charind++);
    # 遍历longopts，查找匹配的长选项
    for(ind = 0; longopts[ind].name && !hits; ind++)
      if((strlen(longopts[ind].name) == (size_t) (charind - offset)) &&
         (strncmp(longopts[ind].name,
                  argv[optind] + offset, charind - offset) == 0))
        found = ind, hits++;
    # 如果没有匹配的长选项，则继续遍历longopts
    if(!hits) for(ind = 0; longopts[ind].name; ind++)
      if(strncmp(longopts[ind].name,
                 argv[optind] + offset, charind - offset) == 0)
        found = ind, hits++;
    # 如果只有一个匹配的长选项
    if(hits == 1) {
      opt = 0;
      # 检查是否有参数
      if(argv[optind][charind] == '=') {
        if(longopts[found].has_arg == 0) {
          opt = '?';
          if(opterr) fprintf(stderr,
                             "%s: option `--%s' doesn't allow an argument\n",
                             argv[0], longopts[found].name);
        } else {
          optarg = argv[optind] + ++charind;
          charind = 0;
        }
      } else if(longopts[found].has_arg == 1) {
        if(++optind >= argc) {
          opt = (colon_mode == ':') ? ':' : '?';
          if(opterr) fprintf(stderr,
                             "%s: option `--%s' requires an argument\n",
                             argv[0], longopts[found].name);
        } else optarg = argv[optind];
      }
      # 如果没有错误
      if(!opt) {
        if (longind) *longind = found;
        if(!longopts[found].flag) opt = longopts[found].val;
        else *(longopts[found].flag) = longopts[found].val;
      }
      optind++;
    } else if(!hits) {
      // 如果没有匹配到选项，则根据情况处理
      if(offset == 1) opt = _getopt(argc, argv, shortopts);
      // 如果偏移量为1，则调用_getopt函数处理选项
      else {
        // 否则将选项设置为'?'，并根据情况输出错误信息
        opt = '?';
        if(opterr) fprintf(stderr,
                           "%s: unrecognized option `%s'\n",
                           argv[0], argv[optind++]);
      }
    } else {
      // 如果匹配到多个选项，则将选项设置为'?'，并根据情况输出错误信息
      opt = '?';
      if(opterr) fprintf(stderr,
                         "%s: option `%s' is ambiguous\n",
                         argv[0], argv[optind++]);
    }
  }
  // 如果偏移量大于参数个数，则将偏移量设置为参数个数
  if (optind > argc) optind = argc;
  // 返回选项
  return opt;
/* 如果没有定义 HAVE_GETOPT，就使用自定义的 getopt 函数，否则使用平台上的版本 */
#ifndef HAVE_GETOPT
int getopt(int argc, char * argv[], const char *opts)
{
  return _getopt(argc, argv, opts);
}
#endif /* HAVE_GETOPT */

/* 使用长选项的 getopt 函数 */
int getopt_long(int argc, char * argv[], const char *shortopts,
                const struct option *longopts, int *longind)
{
  return _getopt_internal(argc, argv, shortopts, longopts, longind, 0);
}

/* 使用长选项的 getopt 函数，但是只接受长选项 */
int getopt_long_only(int argc, char * argv[], const char *shortopts,
                const struct option *longopts, int *longind)
{
  return _getopt_internal(argc, argv, shortopts, longopts, longind, 1);
}
```