# `nmap\libz\examples\gzjoin.c`

```
/* gzjoin -- command to join gzip files into one gzip file
   gzip文件合并命令

  Copyright (C) 2004, 2005, 2012 Mark Adler, all rights reserved
  版权声明

  version 1.2, 14 Aug 2012
  版本信息

  This software is provided 'as-is', without any express or implied
  warranty.  In no event will the author be held liable for any damages
  arising from the use of this software.
  软件免责声明

  Permission is granted to anyone to use this software for any purpose,
  including commercial applications, and to alter it and redistribute it
  freely, subject to the following restrictions:
  许可声明

  1. The origin of this software must not be misrepresented; you must not
     claim that you wrote the original software. If you use this software
     in a product, an acknowledgment in the product documentation would be
     appreciated but is not required.
  2. Altered source versions must be plainly marked as such, and must not be
     misrepresented as being the original software.
  3. This notice may not be removed or altered from any source distribution.
  许可限制

  Mark Adler    madler@alumni.caltech.edu
  作者信息
 */

/*
 * Change history:
 * 变更历史

 * 1.0  11 Dec 2004     - First version
 * 1.1  12 Jun 2005     - Changed ssize_t to long for portability
 * 1.2  14 Aug 2012     - Clean up for z_const usage
 * 版本变更历史
 */
/*
   gzjoin接受命令行上的一个或多个gzip文件，并写出一个单个的gzip文件，该文件解压缩后将得到来自各个gzip文件的未压缩数据的连接。gzjoin在不重新压缩任何数据的情况下完成此操作，并且无需计算连接的未压缩数据的新crc32。但是，gzjoin必须解压缩所有输入数据，以便找到需要修改以连接流的压缩数据中的位。

   gzjoin对输入gzip文件不进行完整性检查，除了检查gzip头并解压缩压缩数据。否则假定它们是完整和正确的。

   每个gzip文件之间的连接至少会删除前一个尾部和后一个头部约18个字节，并且平均会向压缩数据中插入约三个字节以连接流。输出的gzip文件具有最小的十个字节的gzip头，没有文件名或修改时间。

   该程序是为了说明inflate()的Z_BLOCK选项和crc32_combine()函数的使用而编写的。gzjoin将无法与早于1.2.3版本的zlib一起编译。
 */

#include <stdio.h>      /* fputs(), fprintf(), fwrite(), putc() */
#include <stdlib.h>     /* exit(), malloc(), free() */
#include <fcntl.h>      /* open() */
#include <unistd.h>     /* close(), read(), lseek() */
#include "zlib.h"
    /* crc32(), crc32_combine(), inflateInit2(), inflate(), inflateEnd() */

#define local static

/* 以错误退出（返回一个值以允许在表达式中使用） */
local int bail(char *why1, char *why2)
{
    fprintf(stderr, "gzjoin error: %s%s, output incomplete\n", why1, why2);
    exit(1);
    return 0;
}

/* -- 带有对缓冲区的访问的简单缓冲文件输入 -- */

#define CHUNK 32768         /* 必须是2的幂并适合于无符号整数 */

/* 二进制缓冲输入文件类型 */
typedef struct {
    // 文件名，用于错误消息
    char *name;
    // 文件描述符
    int fd;
    // 下一个位置剩余的字节数
    unsigned left;
    // 下一个要读取的字节
    unsigned char *next;
    // 分配的长度为 CHUNK 的缓冲区
    unsigned char *buf;
/* 定义一个结构体类型 bin，用于表示缓冲文件 */
} bin;

/* 关闭缓冲文件并释放分配的内存 */
local void bclose(bin *in)
{
    /* 检查输入指针是否为空 */
    if (in != NULL) {
        /* 如果文件描述符不为-1，则关闭文件 */
        if (in->fd != -1)
            close(in->fd);
        /* 如果缓冲区不为空，则释放内存 */
        if (in->buf != NULL)
            free(in->buf);
        /* 释放结构体内存 */
        free(in);
    }
}

/* 打开一个输入缓冲文件，返回一个指向 bin 类型的指针，如果失败则返回 NULL */
local bin *bopen(char *name)
{
    bin *in;

    /* 分配内存给结构体 */
    in = malloc(sizeof(bin));
    /* 如果分配内存失败，则返回 NULL */
    if (in == NULL)
        return NULL;
    /* 分配缓冲区内存 */
    in->buf = malloc(CHUNK);
    /* 打开文件，并获取文件描述符 */
    in->fd = open(name, O_RDONLY, 0);
    /* 如果缓冲区为空或文件描述符为-1，则关闭文件并释放内存，返回 NULL */
    if (in->buf == NULL || in->fd == -1) {
        bclose(in);
        return NULL;
    }
    /* 初始化剩余字节数为0，下一个指针指向缓冲区开头，记录文件名 */
    in->left = 0;
    in->next = in->buf;
    in->name = name;
    return in;
}

/* 从文件加载缓冲区，读取错误返回-1，成功返回0或1，1表示已到达文件末尾 */
local int bload(bin *in)
{
    long len;

    /* 如果输入指针为空，则返回-1 */
    if (in == NULL)
        return -1;
    /* 如果剩余字节数不为0，则返回0 */
    if (in->left != 0)
        return 0;
    /* 重置下一个指针指向缓冲区开头 */
    in->next = in->buf;
    do {
        /* 从文件读取数据到缓冲区，记录读取的字节数 */
        len = (long)read(in->fd, in->buf + in->left, CHUNK - in->left);
        /* 如果读取错误，则返回-1 */
        if (len < 0)
            return -1;
        /* 更新剩余字节数 */
        in->left += (unsigned)len;
    } while (len != 0 && in->left < CHUNK);
    /* 如果读取到文件末尾，则返回1，否则返回0 */
    return len == 0 ? 1 : 0;
}

/* 从文件获取一个字节，如果到达文件末尾则退出 */
#define bget(in) (in->left ? 0 : bload(in), \
                  in->left ? (in->left--, *(in->next)++) : \
                    bail("unexpected end of file on ", in->name))

/* 从文件获取一个四字节的小端无符号整数 */
local unsigned long bget4(bin *in)
{
    unsigned long val;

    /* 从文件获取一个字节 */
    val = bget(in);
    /* 从文件获取一个字节并左移8位，与上一步结果相加 */
    val += (unsigned long)(bget(in)) << 8;
    val += (unsigned long)(bget(in)) << 16;
    val += (unsigned long)(bget(in)) << 24;
    return val;
}

/* 跳过文件中的字节数 */
local void bskip(bin *in, unsigned skip)
{
    /* 检查输入指针是否为空 */
    if (in == NULL)
        return;

    /* 简单情况 -- 在缓冲区中跳过字节数 */
    if (skip <= in->left) {
        in->left -= skip;
        in->next += skip;
        return;
    }
    /* 跳过缓冲区中的内容，丢弃缓冲区的内容 */
    skip -= in->left;
    in->left = 0;

    /* 寻找超过 CHUNK 字节的倍数 */
    if (skip > CHUNK) {
        unsigned left;

        left = skip & (CHUNK - 1);
        if (left == 0) {
            /* 精确的块数：寻找到最后一块减去一个字节以检查文件结束 */
            lseek(in->fd, skip - 1, SEEK_CUR);
            if (read(in->fd, in->buf, 1) != 1)
                bail("unexpected end of file on ", in->name);
            return;
        }

        /* 跳过整数块，更新 skip 为余数 */
        lseek(in->fd, skip - left, SEEK_CUR);
        skip = left;
    }

    /* 读取更多输入并跳过剩余部分 */
    bload(in);
    if (skip > in->left)
        bail("unexpected end of file on ", in->name);
    in->left -= skip;
    in->next += skip;
/* -- end of buffered input functions -- */
/* 缓冲输入函数结束 */

/* skip the gzip header from file in */
/* 跳过文件中的 gzip 头部 */

local void gzhead(bin *in)
{
    int flags;

    /* verify gzip magic header and compression method */
    /* 验证 gzip 魔术头和压缩方法 */
    if (bget(in) != 0x1f || bget(in) != 0x8b || bget(in) != 8)
        bail(in->name, " is not a valid gzip file");

    /* get and verify flags */
    /* 获取并验证标志位 */
    flags = bget(in);
    if ((flags & 0xe0) != 0)
        bail("unknown reserved bits set in ", in->name);

    /* skip modification time, extra flags, and os */
    /* 跳过修改时间、额外标志和操作系统 */
    bskip(in, 6);

    /* skip extra field if present */
    /* 如果存在，跳过额外字段 */
    if (flags & 4) {
        unsigned len;

        len = bget(in);
        len += (unsigned)(bget(in)) << 8;
        bskip(in, len);
    }

    /* skip file name if present */
    /* 如果存在，跳过文件名 */
    if (flags & 8)
        while (bget(in) != 0)
            ;

    /* skip comment if present */
    /* 如果存在，跳过注释 */
    if (flags & 16)
        while (bget(in) != 0)
            ;

    /* skip header crc if present */
    /* 如果存在，跳过头部 CRC */
    if (flags & 2)
        bskip(in, 2);
}

/* write a four-byte little-endian unsigned integer to out */
/* 将四字节小端无符号整数写入输出 */
local void put4(unsigned long val, FILE *out)
{
    putc(val & 0xff, out);
    putc((val >> 8) & 0xff, out);
    putc((val >> 16) & 0xff, out);
    putc((val >> 24) & 0xff, out);
}

/* Load up zlib stream from buffered input, bail if end of file */
/* 从缓冲输入加载 zlib 流，如果文件结束则退出 */
local void zpull(z_streamp strm, bin *in)
{
    if (in->left == 0)
        bload(in);
    if (in->left == 0)
        bail("unexpected end of file on ", in->name);
    strm->avail_in = in->left;
    strm->next_in = in->next;
}

/* Write header for gzip file to out and initialize trailer. */
/* 将 gzip 文件的头部写入输出并初始化尾部 */
local void gzinit(unsigned long *crc, unsigned long *tot, FILE *out)
{
    fwrite("\x1f\x8b\x08\0\0\0\0\0\0\xff", 1, 10, out);
    *crc = crc32(0L, Z_NULL, 0);
    *tot = 0;
}
# 从名称复制压缩数据，如果clr为真，则清除最后一个块的最后一个位，然后根据需要添加空块以达到字节边界。
# 如果clr为假，则最后一个块变为输出的最后一个块，并写入gzip尾部。crc和tot维护输出的crc和长度（模2^32）用于尾部。
# 结果的gzip文件写入到out中。在第一次调用gzcopy()之前必须调用gzinit()来写入gzip头并初始化crc和tot。
local void gzcopy(char *name, int clr, unsigned long *crc, unsigned long *tot,
                  FILE *out)
{
    int ret;                /* zlib函数的返回值 */
    int pos;                /* "最后一个块"位在字节中的位置 */
    int last;               /* 如果处理最后一个块，则为真 */
    bin *in;                /* 缓冲输入文件 */
    unsigned char *start;   /* 缓冲区中压缩数据的起始位置 */
    unsigned char *junk;    /* 用于未压缩数据的缓冲区 -- 丢弃 */
    z_off_t len;            /* 未压缩数据的长度（支持 > 4 GB） */
    z_stream strm;          /* zlib解压缩流 */

    /* 打开gzip文件并跳过头部 */
    in = bopen(name);
    if (in == NULL)
        bail("could not open ", name);
    gzhead(in);

    /* 为未压缩数据分配缓冲区并初始化原始解压缩流 */
    junk = malloc(CHUNK);
    strm.zalloc = Z_NULL;
    strm.zfree = Z_NULL;
    strm.opaque = Z_NULL;
    strm.avail_in = 0;
    strm.next_in = Z_NULL;
    ret = inflateInit2(&strm, -15);
    if (junk == NULL || ret != Z_OK)
        bail("out of memory", "");

    /* 解压缩并复制压缩数据，如果请求，则清除最后一个块的最后一个位 */
    len = 0;
    zpull(&strm, in);
    start = in->next;
    last = start[0] & 1;
    if (last && clr)
        start[0] &= ~1;
    strm.avail_out = 0;
    for (;;) {
        /* 无限循环，直到条件中断 */
        /* 如果输入已使用并且输出已完成，则写入已使用的输入并获取更多 */
        if (strm.avail_in == 0 && strm.avail_out != 0) {
            fwrite(start, 1, strm.next_in - start, out);
            start = in->buf;
            in->left = 0;
            zpull(&strm, in);
        }

        /* 解压缩 -- 当到达块结束时提前返回 */
        strm.avail_out = CHUNK;
        strm.next_out = junk;
        ret = inflate(&strm, Z_BLOCK);
        switch (ret) {
        case Z_MEM_ERROR:
            bail("out of memory", "");
        case Z_DATA_ERROR:
            bail("invalid compressed data in ", in->name);
        }

        /* 更新未压缩数据的长度 */
        len += CHUNK - strm.avail_out;

        /* 检查块边界（只有在块被复制出来时才会得到这个） */
        if (strm.data_type & 128) {
            /* 如果这是最后一个块，那么结束 */
            if (last)
                break;

            /* 最后一个字节中未使用的位数 */
            pos = strm.data_type & 7;

            /* 找到下一个最后一个块位 */
            if (pos != 0) {
                /* 下一个最后一个块位在最后一个使用的字节中 */
                pos = 0x100 >> pos;
                last = strm.next_in[-1] & pos;
                if (last && clr)
                    in->buf[strm.next_in - in->buf - 1] &= ~pos;
            }
            else {
                /* 下一个最后一个块位在下一个未使用的字节中 */
                if (strm.avail_in == 0) {
                    /* 还没有那个字节 -- 获取它 */
                    fwrite(start, 1, strm.next_in - start, out);
                    start = in->buf;
                    in->left = 0;
                    zpull(&strm, in);
                }
                last = strm.next_in[0] & 1;
                if (last && clr)
                    in->buf[strm.next_in - in->buf] &= ~1;
            }
        }
    }

    /* 更新缓冲区中未使用的输入 */
    in->left = strm.avail_in;
    in->next = in->buf + (strm.next_in - in->buf);
    # 将输入缓冲区中已使用的部分复制到in->next，以便后续处理

    /* copy used input, write empty blocks to get to byte boundary */
    pos = strm.data_type & 7;
    fwrite(start, 1, in->next - start - 1, out);
    # 将已使用的输入数据复制到输出流中，同时写入空块以达到字节边界
    last = in->next[-1];
    if (pos == 0 || !clr)
        /* already at byte boundary, or last file: write last byte */
        putc(last, out);
    else {
        /* append empty blocks to last byte */
        last &= ((0x100 >> pos) - 1);       /* assure unused bits are zero */
        if (pos & 1) {
            /* odd -- append an empty stored block */
            putc(last, out);
            if (pos == 1)
                putc(0, out);               /* two more bits in block header */
            fwrite("\0\0\xff\xff", 1, 4, out);
        }
        else {
            /* even -- append 1, 2, or 3 empty fixed blocks */
            switch (pos) {
            case 6:
                putc(last | 8, out);
                last = 0;
            case 4:
                putc(last | 0x20, out);
                last = 0;
            case 2:
                putc(last | 0x80, out);
                putc(0, out);
            }
        }
    }

    /* update crc and tot */
    *crc = crc32_combine(*crc, bget4(in), len);
    *tot += (unsigned long)len;
    # 更新CRC和总长度

    /* clean up */
    inflateEnd(&strm);
    free(junk);
    bclose(in);
    # 清理资源，释放内存

    /* write trailer if this is the last gzip file */
    if (!clr) {
        put4(*crc, out);
        put4(*tot, out);
    }
    # 如果这是最后一个gzip文件，则写入trailer
/* join the gzip files on the command line, write result to stdout */
int main(int argc, char **argv)
{
    unsigned long crc, tot;     /* running crc and total uncompressed length */

    /* skip command name */
    argc--;
    argv++;

    /* show usage if no arguments */
    if (argc == 0) {
        fputs("gzjoin usage: gzjoin f1.gz [f2.gz [f3.gz ...]] > fjoin.gz\n",
              stderr);
        return 0;
    }

    /* join gzip files on command line and write to stdout */
    gzinit(&crc, &tot, stdout);
    while (argc--)
        gzcopy(*argv++, argc, &crc, &tot, stdout);

    /* done */
    return 0;
}
```