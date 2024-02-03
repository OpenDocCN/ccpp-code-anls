# `nmap\libz\examples\gun.c`

```cpp
/* gun.c -- simple gunzip to give an example of the use of inflateBack()
 * Copyright (C) 2003, 2005, 2008, 2010, 2012 Mark Adler
 * For conditions of distribution and use, see copyright notice in zlib.h
   Version 1.7  12 August 2012  Mark Adler */

/* Version history:
   1.0  16 Feb 2003  First version for testing of inflateBack()
   1.1  21 Feb 2005  Decompress concatenated gzip streams
                     Remove use of "this" variable (C++ keyword)
                     Fix return value for in()
                     Improve allocation failure checking
                     Add typecasting for void * structures
                     Add -h option for command version and usage
                     Add a bunch of comments
   1.2  20 Mar 2005  Add Unix compress (LZW) decompression
                     Copy file attributes from input file to output file
   1.3  12 Jun 2005  Add casts for error messages [Oberhumer]
   1.4   8 Dec 2006  LZW decompression speed improvements
   1.5   9 Feb 2008  Avoid warning in latest version of gcc
   1.6  17 Jan 2010  Avoid signed/unsigned comparison warnings
   1.7  12 Aug 2012  Update for z_const usage in zlib 1.2.8
 */

这段代码是一个简单的gunzip程序，用来展示inflateBack()函数的使用示例。它包含了版权信息和版本历史。
/*
   gun [ -t ] [ name ... ]

   decompresses the data in the named gzip files.  If no arguments are given,
   gun will decompress from stdin to stdout.  The names must end in .gz, -gz,
   .z, -z, _z, or .Z.  The uncompressed data will be written to a file name
   with the suffix stripped.  On success, the original file is deleted.  On
   failure, the output file is deleted.  For most failures, the command will
   continue to process the remaining names on the command line.  A memory
   allocation failure will abort the command.  If -t is specified, then the
   listed files or stdin will be tested as gzip files for integrity (without
   checking for a proper suffix), no output will be written, and no files
   will be deleted.

   Like gzip, gun allows concatenated gzip streams and will decompress them,
   writing all of the uncompressed data to the output.  Unlike gzip, gun allows
   an empty file on input, and will produce no error writing an empty output
   file.

   gun will also decompress files made by Unix compress, which uses LZW
   compression.  These files are automatically detected by virtue of their
   magic header bytes.  Since the end of Unix compress stream is marked by the
   end-of-file, they cannot be concatenated.  If a Unix compress stream is
   encountered in an input file, it is the last stream in that file.

   Like gunzip and uncompress, the file attributes of the original compressed
   file are maintained in the final uncompressed file, to the extent that the
   user permissions allow it.

   On my Mac OS X PowerPC G4, gun is almost twice as fast as gunzip (version
   1.2.4) is on the same file, when gun is linked with zlib 1.2.2.  Also the
   LZW decompression provided by gun is about twice as fast as the standard
   Unix uncompress command.
 */

/* external functions and related types and constants */
#include <stdio.h>          /* fprintf() */
#include <stdlib.h>         /* malloc(), free() */
#include <string.h>         /* strerror(), strcmp(), strlen(), memcpy() */  // 包含字符串处理函数的头文件
#include <errno.h>          /* errno */  // 包含错误处理函数的头文件
#include <fcntl.h>          /* open() */  // 包含文件控制函数的头文件
#include <unistd.h>         /* read(), write(), close(), chown(), unlink() */  // 包含文件 I/O 函数的头文件
#include <sys/types.h>
#include <sys/stat.h>       /* stat(), chmod() */  // 包含文件状态函数的头文件
#include <utime.h>          /* utime() */  // 包含文件时间函数的头文件
#include "zlib.h"           /* inflateBackInit(), inflateBack(), */  // 包含 zlib 库的头文件，用于数据压缩和解压缩

/* function declaration */
#define local static  // 定义宏，用于声明静态函数

/* buffer constants */
#define SIZE 32768U         /* input and output buffer sizes */  // 定义输入和输出缓冲区的大小
#define PIECE 16384         /* limits i/o chunks for 16-bit int case */  // 限制 16 位整数情况下的 I/O 块大小

/* structure for infback() to pass to input function in() -- it maintains the
   input file and a buffer of size SIZE */
struct ind {
    int infile;
    unsigned char *inbuf;
};

/* Load input buffer, assumed to be empty, and return bytes loaded and a
   pointer to them.  read() is called until the buffer is full, or until it
   returns end-of-file or error.  Return 0 on error. */
local unsigned in(void *in_desc, z_const unsigned char **buf)
{
    int ret;
    unsigned len;
    unsigned char *next;
    struct ind *me = (struct ind *)in_desc;

    next = me->inbuf;
    *buf = next;
    len = 0;
    do {
        ret = PIECE;
        if ((unsigned)ret > SIZE - len)
            ret = (int)(SIZE - len);
        ret = (int)read(me->infile, next, ret);
        if (ret == -1) {
            len = 0;
            break;
        }
        next += ret;
        len += ret;
    } while (ret != 0 && len < SIZE);
    return len;
}

/* structure for infback() to pass to output function out() -- it maintains the
   output file, a running CRC-32 check on the output and the total number of
   bytes output, both for checking against the gzip trailer.  (The length in
   the gzip trailer is stored modulo 2^32, so it's ok if a long is 32 bits and
   the output is greater than 4 GB.) */
struct outd {
    # 定义一个整型变量outfile
    int outfile;
    # 定义一个整型变量check，用于标记是否检查crc和total
    int check;                  
    # 定义一个无符号长整型变量crc，用于存储CRC值
    unsigned long crc;
    # 定义一个无符号长整型变量total，用于存储总数值
    unsigned long total;
/* Write output buffer and update the CRC-32 and total bytes written.  write()
   is called until all of the output is written or an error is encountered.
   On success out() returns 0.  For a write failure, out() returns 1.  If the
   output file descriptor is -1, then nothing is written.
 */
local int out(void *out_desc, unsigned char *buf, unsigned len)
{
    // 定义变量ret，结构体指针me指向out_desc
    int ret;
    struct outd *me = (struct outd *)out_desc;

    // 如果需要校验，则更新CRC-32和总字节数
    if (me->check) {
        me->crc = crc32(me->crc, buf, len);
        me->total += len;
    }
    // 如果输出文件描述符不为-1，则写入数据
    if (me->outfile != -1)
        do {
            ret = PIECE;
            // 如果ret大于len，则ret=len
            if ((unsigned)ret > len)
                ret = (int)len;
            // 写入数据，更新ret、buf和len
            ret = (int)write(me->outfile, buf, ret);
            // 如果写入失败，则返回1
            if (ret == -1)
                return 1;
            buf += ret;
            len -= ret;
        } while (len != 0);
    // 返回0表示写入成功
    return 0;
}

/* next input byte macro for use inside lunpipe() and gunpipe() */
#define NEXT() (have ? 0 : (have = in(indp, &next)), \
                last = have ? (have--, (int)(*next++)) : -1)

/* memory for gunpipe() and lunpipe() --
   the first 256 entries of prefix[] and suffix[] are never used, could
   have offset the index, but it's faster to waste the memory */
unsigned char inbuf[SIZE];              /* input buffer */
unsigned char outbuf[SIZE];             /* output buffer */
unsigned short prefix[65536];           /* index to LZW prefix string */
unsigned char suffix[65536];            /* one-character LZW suffix */
unsigned char match[65280 + 2];         /* buffer for reversed match or gzip
                                           32K sliding window */

/* throw out what's left in the current bits byte buffer (this is a vestigial
   aspect of the compressed data format derived from an implementation that
   made use of a special VAX machine instruction!) */
#define FLUSHCODE() \
    # 使用 do-while 循环执行以下代码块
    do { \
        # 初始化 left 为 0
        left = 0; \
        # 初始化 rem 为 0
        rem = 0; \
        # 如果 chunk 大于 have
        if (chunk > have) { \
            # 减去 have，并将 have 置为 0
            chunk -= have; \
            have = 0; \
            # 如果 NEXT() 返回 -1，则跳出循环
            if (NEXT() == -1) \
                break; \
            # 减去一个 chunk
            chunk--; \
            # 如果 chunk 仍大于 have
            if (chunk > have) { \
                # 将 chunk 和 have 置为 0，跳出循环
                chunk = have = 0; \
                break; \
            } \
        } \
        # 减去 chunk
        have -= chunk; \
        # 将 next 增加 chunk
        next += chunk; \
        # 将 chunk 置为 0
        chunk = 0; \
    } while (0)
# 从输入文件中解压缩一个压缩（LZW）文件到输出文件。已经读取并验证了压缩魔术头（两个字节）。在 next 中有缓冲的输入字节。strm 用于将错误信息传递回 gunpipe()。

# lunpipe() 在成功时返回 Z_OK，在意外的文件结束、读取错误或写入错误时返回 Z_BUF_ERROR（写入错误由 strm->next_in 不等于 Z_NULL 表示），或在无效输入时返回 Z_DATA_ERROR。
local int lunpipe(unsigned have, z_const unsigned char *next, struct ind *indp,
                  int outfile, z_stream *strm)
{
    int last;                   /* 上次由 NEXT() 读取的字节，如果是 EOF 则为 -1 */
    unsigned chunk;             /* 当前块中剩余的字节数 */
    int left;                   /* 剩余的位数 */
    unsigned rem;               /* 输入中未使用的位 */
    int bits;                   /* 当前代码的位数 */
    unsigned code;              /* 代码，表遍历索引 */
    unsigned mask;              /* 当前位数代码的掩码 */
    int max;                    /* 该流的每个代码的最大位数 */
    unsigned flags;             /* 压缩标志，然后是块压缩标志 */
    unsigned end;               /* 前缀/后缀表中的最后一个有效条目 */
    unsigned temp;              /* 当前代码 */
    unsigned prev;              /* 前一个代码 */
    unsigned final;             /* 为前一个代码写入的最后一个字符 */
    unsigned stack;             /* 反转字符串的下一个位置 */
    unsigned outcnt;            /* 输出缓冲区中的字节数 */
    struct outd outd;           /* 输出结构 */
    unsigned char *p;

    /* 设置输出 */
    outd.outfile = outfile;
    outd.check = 0;

    /* 处理剩余的压缩头部 -- 一个标志字节 */
    flags = NEXT()
    if (last == -1)
        return Z_BUF_ERROR
    if (flags & 0x60):
        strm->msg = (char *)"unknown lzw flags set"
        return Z_DATA_ERROR
    # 从标志位中获取最大值，与 0x1f 进行按位与操作
    max = flags & 0x1f;
    # 如果最大值小于 9 或大于 16，则返回数据错误
    if (max < 9 || max > 16) {
        strm->msg = (char *)"lzw bits out of range";
        return Z_DATA_ERROR;
    }
    # 如果最大值等于 9，则将其设置为 10
    if (max == 9)                           /* 9 doesn't really mean 9 */
        max = 10;
    # 从标志位中获取压缩块标志位
    flags &= 0x80;                          /* true if block compress */

    # 清空表
    bits = 9;
    mask = 0x1ff;
    end = flags ? 256 : 255;

    # 设置：获取第一个 9 位编码，这是第一个解压缩的字节，但在下一个编码之前不创建表项
    if (NEXT() == -1)                       /* no compressed data is ok */
        return Z_OK;
    final = prev = (unsigned)last;          /* low 8 bits of code */
    if (NEXT() == -1)                       /* missing a bit */
        return Z_BUF_ERROR;
    if (last & 1) {                         /* code must be < 256 */
        strm->msg = (char *)"invalid lzw code";
        return Z_DATA_ERROR;
    }
    rem = (unsigned)last >> 1;              /* remaining 7 bits */
    left = 7;
    chunk = bits - 2;                       /* 7 bytes left in this chunk */
    outbuf[0] = (unsigned char)final;       /* write first decompressed byte */
    outcnt = 1;

    # 解码编码
    stack = 0;
    }
}

/* 从输入文件解压缩一个 gzip 文件到输出文件。假定 strm 已经成功初始化为 inflateBackInit()。输入文件可能由一系列 gzip 流组成，如果是这样，所有这些流都将被解压缩到输出文件中。如果 outfile 是 -1，则检查 gzip 流的完整性，并且不写入任何内容。

   返回值是一个 zlib 错误代码：如果内存不足则返回 Z_MEM_ERROR，如果头部或压缩数据无效则返回 Z_DATA_ERROR，如果尾部的 CRC-32 校验或长度不匹配则返回 Z_DATA_ERROR，如果输入过早结束或发生写入错误则返回 Z_BUF_ERROR，如果跟在有效的 gzip 流后面的是垃圾（不是另一个 gzip 流）则返回 Z_ERRNO。
 */
local int gunpipe(z_stream *strm, int infile, int outfile)
{
    int ret, first, last;
    unsigned have, flags, len;
    z_const unsigned char *next = NULL;
    struct ind ind, *indp;
    struct outd outd;

    /* 设置输入缓冲区 */
    ind.infile = infile;
    ind.inbuf = inbuf;
    indp = &ind;

    /* 解压缩串联的 gzip 流 */
    have = 0;                               /* 尚未读取任何输入数据 */
    first = 1;                              /* 寻找第一个 gzip 头部 */
    strm->next_in = Z_NULL;                 /* 因此 Z_BUF_ERROR 表示 EOF */
    }

    /* 清理并返回 */
    return ret;
}

/* 尽力复制文件属性，从 from -> to。这是最大的努力，因此不报告任何错误。模式位，包括 suid、sgid 和粘性位都被复制（如果允许），所有者的用户 id 和组 id 被复制（再次如果允许），访问和修改时间被复制。 */
local void copymeta(char *from, char *to)
{
    struct stat was;
    struct utimbuf when;

    /* 获取 from 的 Unix 元数据，如果不是常规文件则返回 */
    if (stat(from, &was) != 0 || (was.st_mode & S_IFMT) != S_IFREG)
        return;

    /* 设置 to 的模式位，忽略错误 */
    (void)chmod(to, was.st_mode & 07777);
    # 复制所有者的用户和组，忽略错误
    (void)chown(to, was.st_uid, was.st_gid);

    # 复制访问和修改时间，忽略错误
    when.actime = was.st_atime;
    when.modtime = was.st_mtime;
    (void)utime(to, &when);
}
/* 解压缩 inname 文件到 outnname 文件，如果 test 为真，则只解压缩而不写入，并检查 gzip 尾部以确保完整性。如果 inname 为 NULL 或空字符串，则从 stdin 读取。如果 outname 为 NULL 或空字符串，则写入 stdout。strm 是预初始化的 inflateBack 结构。在适当的时候，从 inname 复制文件属性到 outname。

   gunzip() 如果出现内存不足错误或从 gunpipe() 返回意外的返回码，则返回 1。否则返回 0。
 */
local int gunzip(z_stream *strm, char *inname, char *outname, int test)
{
    int ret;
    int infile, outfile;

    /* 打开文件 */
    if (inname == NULL || *inname == 0) {
        inname = "-";
        infile = 0;     /* stdin */
    }
    else {
        infile = open(inname, O_RDONLY, 0);
        if (infile == -1) {
            fprintf(stderr, "gun cannot open %s\n", inname);
            return 0;
        }
    }
    if (test)
        outfile = -1;
    else if (outname == NULL || *outname == 0) {
        outname = "-";
        outfile = 1;    /* stdout */
    }
    else {
        outfile = open(outname, O_CREAT | O_TRUNC | O_WRONLY, 0666);
        if (outfile == -1) {
            close(infile);
            fprintf(stderr, "gun cannot create %s\n", outname);
            return 0;
        }
    }
    errno = 0;

    /* 解压缩 */
    ret = gunpipe(strm, infile, outfile);
    if (outfile > 2) close(outfile);
    if (infile > 2) close(infile);

    /* 解释结果 */
    switch (ret) {
    case Z_OK:
    case Z_ERRNO:
        if (infile > 2 && outfile > 2) {
            copymeta(inname, outname);          /* 复制属性 */
            unlink(inname);
        }
        if (ret == Z_ERRNO)
            fprintf(stderr, "gun warning: trailing garbage ignored in %s\n",
                    inname);
        break;
    # 如果数据错误，则删除输出文件（如果输出文件描述符大于2），并打印错误信息
    case Z_DATA_ERROR:
        if (outfile > 2) unlink(outname);
        fprintf(stderr, "gun data error on %s: %s\n", inname, strm->msg);
        break;
    # 如果内存错误，则删除输出文件（如果输出文件描述符大于2），并打印错误信息，然后返回1
    case Z_MEM_ERROR:
        if (outfile > 2) unlink(outname);
        fprintf(stderr, "gun out of memory error--aborting\n");
        return 1;
    # 如果缓冲区错误，则删除输出文件（如果输出文件描述符大于2），根据情况打印不同的错误信息
    case Z_BUF_ERROR:
        if (outfile > 2) unlink(outname);
        if (strm->next_in != Z_NULL) {
            fprintf(stderr, "gun write error on %s: %s\n",
                    outname, strerror(errno));
        }
        else if (errno) {
            fprintf(stderr, "gun read error on %s: %s\n",
                    inname, strerror(errno));
        }
        else {
            fprintf(stderr, "gun unexpected end of file on %s\n",
                    inname);
        }
        break;
    # 默认情况下，删除输出文件（如果输出文件描述符大于2），并打印内部错误信息，然后返回1
    default:
        if (outfile > 2) unlink(outname);
        fprintf(stderr, "gun internal error--aborting\n");
        return 1;
    # 返回0表示没有错误
    }
    return 0;
/* 处理 gun 命令行参数。请参阅源文件开头附近的命令语法。 */
int main(int argc, char **argv)
{
    int ret, len, test;
    char *outname;
    unsigned char *window;
    z_stream strm;

    /* 为重复使用初始化 inflateBack 状态 */
    window = match;                         /* 重用 LZW 匹配缓冲区 */
    strm.zalloc = Z_NULL;
    strm.zfree = Z_NULL;
    strm.opaque = Z_NULL;
    ret = inflateBackInit(&strm, 15, window);
    if (ret != Z_OK) {
        fprintf(stderr, "gun 内存不足错误--中止\n");
        return 1;
    }

    /* 将每个文件解压缩到去除后缀名的相同名称 */
    argc--;
    argv++;
    test = 0;
    if (argc && strcmp(*argv, "-h") == 0) {
        fprintf(stderr, "gun 1.6 (17 Jan 2010)\n");
        fprintf(stderr, "版权所有 (C) 2003-2010 Mark Adler\n");
        fprintf(stderr, "用法: gun [-t] [file1.gz [file2.Z ...]]\n");
        return 0;
    }
    if (argc && strcmp(*argv, "-t") == 0) {
        test = 1;
        argc--;
        argv++;
    }
    # 如果参数个数不为零，则执行循环
    if (argc)
        do {
            # 如果测试标志为真，则输出文件名为空
            if (test)
                outname = NULL;
            # 否则，根据文件名的后缀确定输出文件名
            else {
                len = (int)strlen(*argv);
                # 如果文件名以".gz"或"-gz"结尾，则去掉后缀
                if (strcmp(*argv + len - 3, ".gz") == 0 ||
                    strcmp(*argv + len - 3, "-gz") == 0)
                    len -= 3;
                # 如果文件名以".z"、"-z"、"_z"或".Z"结尾，则去掉后缀
                else if (strcmp(*argv + len - 2, ".z") == 0 ||
                    strcmp(*argv + len - 2, "-z") == 0 ||
                    strcmp(*argv + len - 2, "_z") == 0 ||
                    strcmp(*argv + len - 2, ".Z") == 0)
                    len -= 2;
                # 否则，输出错误信息并跳过当前文件
                else {
                    fprintf(stderr, "gun error: no gz type on %s--skipping\n",
                            *argv);
                    continue;
                }
                # 分配内存并复制文件名到输出文件名
                outname = malloc(len + 1);
                # 如果分配内存失败，则输出错误信息并终止程序
                if (outname == NULL) {
                    fprintf(stderr, "gun out of memory error--aborting\n");
                    ret = 1;
                    break;
                }
                memcpy(outname, *argv, len);
                outname[len] = 0;
            }
            # 调用gunzip函数解压文件，并传入相应参数
            ret = gunzip(&strm, *argv, outname, test);
            # 如果输出文件名不为空，则释放内存
            if (outname != NULL) free(outname);
            # 如果解压失败，则跳出循环
            if (ret) break;
        } while (argv++, --argc);
    # 如果参数个数为零，则调用gunzip函数，传入相应参数
    else
        ret = gunzip(&strm, NULL, NULL, test);

    # 清理内存
    inflateBackEnd(&strm);
    # 返回解压结果
    return ret;
# 闭合前面的函数定义
```