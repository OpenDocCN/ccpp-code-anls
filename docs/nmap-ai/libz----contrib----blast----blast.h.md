# `nmap\libz\contrib\blast\blast.h`

```cpp
/* blast.h -- interface for blast.c
  声明 blast.c 的接口
  Copyright (C) 2003, 2012, 2013 Mark Adler
  版权声明
  version 1.3, 24 Aug 2013
  版本信息

  This software is provided 'as-is', without any express or implied
  warranty.  In no event will the author be held liable for any damages
  arising from the use of this software.
  软件免责声明

  Permission is granted to anyone to use this software for any purpose,
  including commercial applications, and to alter it and redistribute it
  freely, subject to the following restrictions:
  授权声明

  1. The origin of this software must not be misrepresented; you must not
     claim that you wrote the original software. If you use this software
     in a product, an acknowledgment in the product documentation would be
     appreciated but is not required.
  2. Altered source versions must be plainly marked as such, and must not be
     misrepresented as being the original software.
  3. This notice may not be removed or altered from any source distribution.
  使用限制声明

  Mark Adler    madler@alumni.caltech.edu
  作者信息
 */


/*
 * blast() decompresses the PKWare Data Compression Library (DCL) compressed
 * format.  It provides the same functionality as the explode() function in
 * that library.  (Note: PKWare overused the "implode" verb, and the format
 * used by their library implode() function is completely different and
 * incompatible with the implode compression method supported by PKZIP.)
 * blast() 函数用于解压缩 PKWare 数据压缩库 (DCL) 的压缩格式。它提供了与该库中的 explode() 函数相同的功能。
 * (注意：PKWare 过度使用了 "implode" 动词，他们的库中 implode() 函数使用的格式与 PKZIP 支持的 implode 压缩方法完全不同且不兼容。)

 * The binary mode for stdio functions should be used to assure that the
 * compressed data is not corrupted when read or written.  For example:
 * fopen(..., "rb") and fopen(..., "wb").
 * 应该使用stdio函数的二进制模式来确保在读取或写入时不会损坏压缩数据。例如：fopen(..., "rb") 和 fopen(..., "wb")。
 */


typedef unsigned (*blast_in)(void *how, unsigned char **buf);
typedef int (*blast_out)(void *how, unsigned char *buf, unsigned len);
/* Definitions for input/output functions passed to blast().  See below for
 * what the provided functions need to do.
 * 传递给 blast() 的输入/输出函数的定义。下面是提供的函数需要做什么。

 */


int blast(blast_in infun, void *inhow, blast_out outfun, void *outhow,
          unsigned *left, unsigned char **in);
/* Function prototype for blast() with input/output functions and additional
 * parameters. See below for what the function does.
 * blast() 的函数原型，带有输入/输出函数和额外的参数。下面是函数的功能。
 */
/* 使用提供的 infun() 和 outfun() 调用来将输入解压缩到输出。
 * 成功时，blast() 的返回值为零。如果源数据出现错误，即不符合正确格式，则返回负值。
 * 如果没有足够的输入可用或没有足够的输出空间，则返回正错误值。
 *
 * 输入函数被调用为：len = infun(how, &buf)，其中 buf 被 infun() 设置为指向输入缓冲区，infun() 返回那里可用的字节数。
 * 如果 infun() 返回零，则 blast() 返回输入错误。（blast() 仅在需要时才请求输入。）inhow 用于应用程序向 infun() 传递输入描述符（如果需要）。
 *
 * 如果 left 和 in 非空且在调用 blast() 时 *left 不为零，则在使用 infun() 之前会消耗 *in 中的 *left 字节作为输入。
 *
 * 输出函数被调用为：err = outfun(how, buf, len)，其中要写入的字节为 buf[0..len-1]。如果 err 不为零，则 blast() 返回输出错误。outfun() 总是以 len <= 4096 调用。outhow 用于应用程序向 outfun() 传递输出描述符（如果需要）。
 *
 * 如果有任何未使用的输入，则 *left 被设置为已读取的字节数，*in 指向它们。否则 *left 被设置为零，*in 被设置为 NULL。如果 left 或 in 为空，则它们不被设置。
 *
 * 返回代码为：
 *   2:  在完成解压缩之前用尽了输入
 *   1:  在完成解压缩之前发生输出错误
 *   0:  成功解压缩
 *  -1:  文字标志不为零或一
 *  -2:  字典大小不在 4 到 6 之间
 *  -3:  距离太远
 *
 * 在 blast.c 的底部有一个使用 blast() 的示例程序，可以通过定义 TEST 来编译生成一个命令行解压缩过滤器。
 */
```