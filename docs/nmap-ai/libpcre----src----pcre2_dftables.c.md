# `nmap\libpcre\src\pcre2_dftables.c`

```cpp
/*************************************************
*      Perl-Compatible Regular Expressions       *
*************************************************/

/* PCRE is a library of functions to support regular expressions whose syntax
and semantics are as close as possible to those of the Perl 5 language.

                       Written by Philip Hazel
     Original API code Copyright (c) 1997-2012 University of Cambridge
          New API code Copyright (c) 2016-2020 University of Cambridge

----------------------------------------------------------------------------- 
Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

    * Redistributions of source code must retain the above copyright notice,
      this list of conditions and the following disclaimer.

    * Redistributions in binary form must reproduce the above copyright
      notice, this list of conditions and the following disclaimer in the
      documentation and/or other materials provided with the distribution.

    * Neither the name of the University of Cambridge nor the names of its
      contributors may be used to endorse or promote products derived from
      this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE
LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
/* 
   该程序用于生成包含 PCRE2 字符表的文件的支持程序。
   这些表是使用 pcre2_maketables() 函数构建的，该函数是 PCRE2 API 的一部分。
   默认情况下，使用系统的 "C" 区域设置，而不是构建用户设置的区域设置，但可以使用 -L 选项从 LC_ALL 环境变量中选择当前区域设置。
   默认情况下，表以源代码形式写入，但如果给定了 -b，则以二进制形式写入。
*/

#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include <ctype.h>
#include <stdio.h>
#include <string.h>
#include <locale.h>

#define PCRE2_CODE_UNIT_WIDTH 0   /* 必须设置，但在这里不相关 */
#include "pcre2_internal.h"

#define PCRE2_DFTABLES            /* pcre2_maketables.c 注意到这一点 */
#include "pcre2_maketables.c"

static const char *classlist[] =
  {
  "space", "xdigit", "digit", "upper", "lower",
  "word", "graph", "print", "punct", "cntrl"
  };

/*************************************************
*                  Usage                         *
*************************************************/

static void
usage(void)
{
(void)fprintf(stderr,
  "Usage: pcre2_dftables [options] <output file>\n"
  "  -b    Write output in binary (default is source code)\n"
  "  -L    Use locale from LC_ALL (default is \"C\" locale)\n"
  );
}

/*************************************************
*                Entry point                     *
*************************************************/

int main(int argc, char **argv)
{
FILE *f;
int i;
int nclass = 0;
BOOL binary = FALSE;
char *env = (char *)"C";
const unsigned char *tables;
const unsigned char *base_of_tables;

/* Process options */
# 遍历命令行参数，从第二个参数开始
for (i = 1; i < argc; i++)
  {
  # 获取当前参数
  char *arg = argv[i];
  # 如果参数不是以 '-' 开头，则跳出循环
  if (*arg != '-') break;

  # 如果参数是 "-help" 或 "--help"，则调用 usage() 函数并返回 0
  if (strcmp(arg, "-help") == 0 || strcmp(arg, "--help") == 0)
    {
    usage();
    return 0;
    }

  # 如果参数是 "-L"，则设置本地化环境
  else if (strcmp(arg, "-L") == 0)
    {
    if (setlocale(LC_ALL, "") == NULL)
      {
      (void)fprintf(stderr, "pcre2_dftables: setlocale() failed\n");
      return 1;
      }
    env = getenv("LC_ALL");
    }

  # 如果参数是 "-b"，则设置 binary 为 TRUE
  else if (strcmp(arg, "-b") == 0)
    binary = TRUE;

  # 如果参数不是以上任何选项，则输出错误信息并返回 1
  else
    {
    (void)fprintf(stderr, "pcre2_dftables: unrecognized option %s\n", arg);
    return 1;
    }
  }

# 如果循环结束后，参数个数不是 argc - 1，则输出错误信息并返回 1
if (i != argc - 1)
  {
  (void)fprintf(stderr, "pcre2_dftables: one filename argument is required\n");
  return 1;
  }

# 生成表格
tables = maketables();
base_of_tables = tables;

# 打开文件准备写入表格数据
f = fopen(argv[i], "wb");
if (f == NULL)
  {
  fprintf(stderr, "pcre2_dftables: failed to open %s for writing\n", argv[1]);
  return 1;
  }

# 如果指定了 -b 选项，则以二进制形式写入表格数据
if (binary)
  {
  int yield = 0;
  size_t len = fwrite(tables, 1, TABLES_LENGTH, f);
  if (len != TABLES_LENGTH)
    {
    (void)fprintf(stderr, "pcre2_dftables: fwrite() returned wrong length %d "
     "instead of %d\n", (int)len, TABLES_LENGTH);
    yield = 1;
    }
  fclose(f);
  free((void *)base_of_tables);
  return yield;
  }

# 否则，以源代码形式写入表格数据
(void)fprintf(f,
  "/*************************************************\n"
  "*      Perl-Compatible Regular Expressions       *\n"
  "*************************************************/\n\n"
  "/* This file was automatically written by the pcre2_dftables auxiliary\n"
  "program. It contains character tables that are used when no external\n"
  "tables are passed to PCRE2 by the application that calls it. The tables\n"
  "are used only for characters whose code values are less than 256. */\n\n");
// 将注释写入文件，指明表格是在哪个 locale 下生成的
(void)fprintf(f,
  "/* This set of tables was written in the %s locale. */\n\n", env);

// 写入关于 pcre2_ftables 程序的说明
(void)fprintf(f,
  "/* The pcre2_ftables program (which is distributed with PCRE2) can be used\n"
  "to build alternative versions of this file. This is necessary if you are\n"
  "running in an EBCDIC environment, or if you want to default to a different\n"
  "encoding, for example ISO-8859-1. When pcre2_dftables is run, it creates\n"
  "these tables in the \"C\" locale by default. This happens automatically if\n"
  "PCRE2 is configured with --enable-rebuild-chartables. However, you can run\n"
  "pcre2_dftables manually with the -L option to build tables using the LC_ALL\n"
  "locale. */\n\n");

// 在 z/OS 下强制使用 config.h
#if defined NATIVE_ZOS
(void)fprintf(f,
  "/* For z/OS, config.h is forced */\n"
  "#ifndef HAVE_CONFIG_H\n"
  "#define HAVE_CONFIG_H 1\n"
  "#endif\n\n");
#endif

// 写入关于包含 config.h 的说明
(void)fprintf(f,
  "/* The following #include is present because without it gcc 4.x may remove\n"
  "the array definition from the final binary if PCRE2 is built into a static\n"
  "library and dead code stripping is activated. This leads to link errors.\n"
  "Pulling in the header ensures that the array gets flagged as \"someone\n"
  "outside this compilation unit might reference this\" and so it will always\n"
  "be supplied to the linker. */\n\n");

// 根据是否有 config.h 写入相应的 include
(void)fprintf(f,
  "#ifdef HAVE_CONFIG_H\n"
  "#include \"config.h\"\n"
  "#endif\n\n"
  "#include \"pcre2_internal.h\"\n\n");

// 写入默认表格的说明
(void)fprintf(f,
  "const uint8_t PRIV(default_tables)[] = {\n\n"
  "/* This table is a lower casing table. */\n\n");

// 写入 lower casing table 的内容
(void)fprintf(f, "  ");
for (i = 0; i < 256; i++)
  {
  if ((i & 7) == 0 && i != 0) fprintf(f, "\n  ");
  fprintf(f, "%3d", *tables++);
  if (i != 255) fprintf(f, ",");
  }
(void)fprintf(f, ",\n\n");

// 写入 case flipping table 的说明
(void)fprintf(f, "/* This table is a case flipping table. */\n\n");

// 继续写入 case flipping table 的内容
(void)fprintf(f, "  ");
for (i = 0; i < 256; i++)
  {
  // 如果 i 能被 7 整除且不等于 0，则在文件中换行并缩进两个空格
  if ((i & 7) == 0 && i != 0) fprintf(f, "\n  ");
  // 在文件中输出当前指针指向的值，并在后面加上逗号
  fprintf(f, "%3d", *tables++);
  // 如果 i 不等于 255，则在文件中输出逗号
  if (i != 255) fprintf(f, ",");
  }
(void)fprintf(f, ",\n\n");

(void)fprintf(f,
  "/* This table contains bit maps for various character classes. Each map is 32\n"
  "bytes long and the bits run from the least significant end of each byte. The\n"
  "classes that have their own maps are: space, xdigit, digit, upper, lower, word,\n"
  "graph, print, punct, and cntrl. Other classes are built from combinations. */\n\n");

(void)fprintf(f, "  ");
for (i = 0; i < cbit_length; i++)
  {
  // 如果 i 能被 7 整除且不等于 0
  if ((i & 7) == 0 && i != 0)
    {
    // 如果 i 能被 31 整除，则在文件中换行
    if ((i & 31) == 0) (void)fprintf(f, "\n");
    // 如果 i 能被 24 整除且不等于 8，则在文件中输出当前类别的注释
    if ((i & 24) == 8) (void)fprintf(f, "  /* %s */", classlist[nclass++]);
    // 在文件中换行并缩进两个空格
    (void)fprintf(f, "\n  ");
    }
  // 在文件中输出当前指针指向的值，并在后面加上逗号
  (void)fprintf(f, "0x%02x", *tables++);
  // 如果 i 不等于 cbit_length - 1，则在文件中输出逗号
  if (i != cbit_length - 1) (void)fprintf(f, ",");
  }
(void)fprintf(f, ",\n\n");

(void)fprintf(f,
  "/* This table identifies various classes of character by individual bits:\n"
  "  0x%02x   white space character\n"
  "  0x%02x   letter\n"
  "  0x%02x   lower case letter\n"
  "  0x%02x   decimal digit\n"
  "  0x%02x   alphanumeric or '_'\n*/\n\n",
  ctype_space, ctype_letter, ctype_lcletter, ctype_digit, ctype_word);

(void)fprintf(f, "  ");
for (i = 0; i < 256; i++)
  {
  // 如果 i 能被 7 整除且不等于 0
  if ((i & 7) == 0 && i != 0)
    {
    // 在文件中输出当前字符范围的注释
    (void)fprintf(f, " /* ");
    // 如果当前字符可打印，则在注释中输出该字符
    if (isprint(i-8)) (void)fprintf(f, " %c -", i-8);
      else (void)fprintf(f, "%3d-", i-8);
    // 如果当前字符可打印，则在注释中输出该字符
    if (isprint(i-1)) (void)fprintf(f, " %c ", i-1);
      else (void)fprintf(f, "%3d", i-1);
    // 在注释中输出结束符
    (void)fprintf(f, " */\n  ");
    }
  // 在文件中输出当前指针指向的值
  (void)fprintf(f, "0x%02x", *tables++);
  // 如果 i 不等于 255，则在文件中输出逗号
  if (i != 255) (void)fprintf(f, ",");
  }

(void)fprintf(f, "};/* ");
// 如果当前字符可打印，则在注释中输出该字符
if (isprint(i-8)) (void)fprintf(f, " %c -", i-8);
  else (void)fprintf(f, "%3d-", i-8);
// 如果当前字符可打印，则在注释中输出该字符
if (isprint(i-1)) (void)fprintf(f, " %c ", i-1);
  else (void)fprintf(f, "%3d", i-1);
// 在注释中输出结束符
(void)fprintf(f, " */\n\n/* End of pcre2_chartables.c */\n");

fclose(f);
free((void *)base_of_tables);
# 返回整数 0
return 0;
# pcre2_dftables.c 文件结束
/* End of pcre2_dftables.c */
```