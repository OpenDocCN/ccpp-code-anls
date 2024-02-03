# `nmap\libz\examples\gzappend.c`

```cpp
/* gzappend -- command to append to a gzip file
   gzappend -- 用于向 gzip 文件追加内容的命令

  Copyright (C) 2003, 2012 Mark Adler, all rights reserved
  版权所有 2003 年，2012 年 Mark Adler，保留所有权利
  version 1.2, 11 Oct 2012
  版本 1.2，2012 年 10 月 11 日

  This software is provided 'as-is', without any express or implied
  warranty.  In no event will the author be held liable for any damages
  arising from the use of this software.
  本软件按原样提供，没有任何明示或暗示的保证。作者不对因使用本软件而产生的任何损害承担责任。

  Permission is granted to anyone to use this software for any purpose,
  including commercial applications, and to alter it and redistribute it
  freely, subject to the following restrictions:
  任何人都被授予使用本软件的权限，包括商业应用，并可以自由修改和重新分发，但需遵守以下限制：

  1. The origin of this software must not be misrepresented; you must not
     claim that you wrote the original software. If you use this software
     in a product, an acknowledgment in the product documentation would be
     appreciated but is not required.
     本软件的来源不得被歪曲；您不得声称自己编写了原始软件。如果您在产品中使用本软件，产品文档中的致谢将不是必需的，但会受到赞赏。
  2. Altered source versions must be plainly marked as such, and must not be
     misrepresented as being the original software.
     修改后的源代码版本必须明确标记，并且不得被误传为原始软件。
  3. This notice may not be removed or altered from any source distribution.
     本通知不得从任何源分发中删除或更改。

  Mark Adler    madler@alumni.caltech.edu
 */
/*
 * Change history:
 *
 * 1.0  19 Oct 2003     - First version
 * 1.1   4 Nov 2003     - Expand and clarify some comments and notes
 *                      - Add version and copyright to help
 *                      - Send help to stdout instead of stderr
 *                      - Add some preemptive typecasts
 *                      - Add L to constants in lseek() calls
 *                      - Remove some debugging information in error messages
 *                      - Use new data_type definition for zlib 1.2.1
 *                      - Simplify and unify file operations
 *                      - Finish off gzip file in gztack()
 *                      - Use deflatePrime() instead of adding empty blocks
 *                      - Keep gzip file clean on appended file read errors
 *                      - Use in-place rotate instead of auxiliary buffer
 *                        (Why you ask?  Because it was fun to write!)
 * 1.2  11 Oct 2012     - Fix for proper z_const usage
 *                      - Check for input buffer malloc failure
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include "zlib.h"

#define local static
#define LGCHUNK 14
#define CHUNK (1U << LGCHUNK)
#define DSIZE 32768U

/* print an error message and terminate with extreme prejudice */
local void bye(char *msg1, char *msg2)
{
    // 打印错误消息并终止程序
    fprintf(stderr, "gzappend error: %s%s\n", msg1, msg2);
    exit(1);
}

/* return the greatest common divisor of a and b using Euclid's algorithm,
   modified to be fast when one argument much greater than the other, and
   coded to avoid unnecessary swapping */
local unsigned gcd(unsigned a, unsigned b)
{
    unsigned c;

    while (a && b)
        if (a > b) {
            c = b;
            while (a - c >= c)
                c <<= 1;
            a -= c;
        }
        else {
            c = a;
            while (b - c >= c)
                c <<= 1;
            b -= c;
        }
    return a + b;
}
# 将列表 list[0..len-1] 左旋转 rot 个位置，原地操作
def rotate(list, len, rot):
    # 定义临时变量
    tmp = 0
    # 定义循环次数
    cycles = 0
    # 定义指针变量
    start, last, to, from = None, None, None, None

    # 规范化 rot 并处理退化情况
    if len < 2:
        return
    if rot >= len:
        rot %= len
    if rot == 0:
        return

    # 指向列表中最后一个条目
    last = list + (len - 1)

    # 简单左移一位
    if rot == 1:
        tmp = list[0]
        list[0:len-1] = list[1:]
        last[0] = tmp
        return

    # 简单右移一位
    if rot == len - 1:
        tmp = last[0]
        list[1:] = list[0:len-1]
        list[0] = tmp
        return

    # 否则，作为一组循环进行原地旋转
    cycles = gcd(len, rot)  # 循环次数
    while True:
        start = from = list + cycles  # 起始索引是任意的
        tmp = from[0]  # 保存要被覆盖的条目
        while True:
            to = from  # 循环中的下一步
            from += rot  # 向右移动 rot 个位置
            if from > last:
                from -= len  # （指针最好不要回绕）
            if from == start:
                break  # 除了一个条目外，全部移动
            to[0] = from[0]  # 左移
        to[0] = tmp  # 完成循环
        cycles -= 1
        if cycles == 0:
            break

# 用于 gzip 文件读取操作的结构
class file:
    def __init__(self, fd, size, left, buf, next, name):
        self.fd = fd  # 文件描述符
        self.size = size  # buf 的字节数
        self.left = left  # 下一个可用的字节数
        self.buf = buf  # 缓冲区
        self.next = next  # 缓冲区中的下一个字节
        self.name = name  # 错误消息的文件名

# 重新加载缓冲区
def readin(in):
    len = 0
    # 从输入文件描述符中读取数据到输入缓冲区，读取长度为 1 左移输入大小位
    len = read(in->fd, in->buf, 1 << in->size);
    # 如果读取长度为 -1，表示读取出错，调用 bye 函数并输出错误信息和输入文件名
    if (len == -1) bye("error reading ", in->name);
    # 将读取长度转换为无符号整数，赋值给输入剩余长度
    in->left = (unsigned)len;
    # 将输入缓冲区的下一个位置指针指向缓冲区起始位置
    in->next = in->buf;
    # 返回读取长度
    return len;
/* read from file in, exit if end-of-file */
local int readmore(file *in)
{
    // 如果从文件中读取的内容为0，则退出程序并打印错误信息
    if (readin(in) == 0) bye("unexpected end of ", in->name);
    // 返回0表示正常结束
    return 0;
}

#define read1(in) (in->left == 0 ? readmore(in) : 0, \
                   in->left--, *(in->next)++)

/* skip over n bytes of in */
local void skip(file *in, unsigned n)
{
    unsigned bypass;

    // 如果需要跳过的字节数大于文件中剩余的字节数
    if (n > in->left) {
        // 计算需要绕过的字节数
        n -= in->left;
        bypass = n & ~((1U << in->size) - 1);
        // 如果需要绕过的字节数不为0，则进行文件指针的偏移
        if (bypass) {
            if (lseek(in->fd, (off_t)bypass, SEEK_CUR) == -1)
                bye("seeking ", in->name);
            n -= bypass;
        }
        // 读取更多内容
        readmore(in);
        // 如果需要跳过的字节数大于文件中剩余的字节数，则退出程序并打印错误信息
        if (n > in->left)
            bye("unexpected end of ", in->name);
    }
    // 更新文件中剩余的字节数和文件指针位置
    in->left -= n;
    in->next += n;
}

/* read a four-byte unsigned integer, little-endian, from in */
unsigned long read4(file *in)
{
    unsigned long val;

    // 从文件中读取四个字节的无符号整数，按小端模式解析
    val = read1(in);
    val += (unsigned)read1(in) << 8;
    val += (unsigned long)read1(in) << 16;
    val += (unsigned long)read1(in) << 24;
    return val;
}

/* skip over gzip header */
local void gzheader(file *in)
{
    int flags;
    unsigned n;

    // 如果文件头不是gzip格式，则退出程序并打印错误信息
    if (read1(in) != 31 || read1(in) != 139) bye(in->name, " not a gzip file");
    // 如果压缩方法不是8（deflate），则退出程序并打印错误信息
    if (read1(in) != 8) bye("unknown compression method in", in->name);
    flags = read1(in);
    // 如果头部标志位有未知的设置，则退出程序并打印错误信息
    if (flags & 0xe0) bye("unknown header flags set in", in->name);
    // 跳过6个字节
    skip(in, 6);
    // 如果标志位中有4，则跳过指定字节数
    if (flags & 4) {
        n = read1(in);
        n += (unsigned)(read1(in)) << 8;
        skip(in, n);
    }
    // 如果标志位中有8，则跳过直到遇到0为止
    if (flags & 8) while (read1(in) != 0) ;
    // 如果标志位中有16，则跳过直到遇到0为止
    if (flags & 16) while (read1(in) != 0) ;
    // 如果标志位中有2，则跳过2个字节
    if (flags & 2) skip(in, 2);
}

/* decompress gzip file "name", return strm with a deflate stream ready to
   continue compression of the data in the gzip file, and return a file
   descriptor pointing to where to write the compressed data -- the deflate
   stream is initialized to compress using level "level" */
local int gzscan(char *name, z_stream *strm, int level)
{
    int ret, lastbit, left, full;
    // 省略部分代码
}
    # 已经读取的字节数
    unsigned have;
    # CRC 校验值
    unsigned long crc, tot;
    # 窗口缓冲区
    unsigned char *window;
    # 上一次的偏移量和结束位置
    off_t lastoff, end;
    # 文件对象
    file gz;

    # 打开 gzip 文件
    gz.name = name;
    gz.fd = open(name, O_RDWR, 0);
    # 如果文件打开失败，则退出并打印错误信息
    if (gz.fd == -1) bye("cannot open ", name);
    # 分配内存给缓冲区
    gz.buf = malloc(CHUNK);
    # 如果内存分配失败，则退出并打印错误信息
    if (gz.buf == NULL) bye("out of memory", "");
    # 设置缓冲区大小
    gz.size = LGCHUNK;
    # 剩余未读取的字节数
    gz.left = 0;

    # 跳过 gzip 头部
    gzheader(&gz);

    # 准备解压缩
    # 分配内存给窗口缓冲区
    window = malloc(DSIZE);
    # 如果内存分配失败，则退出并打印错误信息
    if (window == NULL) bye("out of memory", "");
    # 设置解压缩参数
    strm->zalloc = Z_NULL;
    strm->zfree = Z_NULL;
    strm->opaque = Z_NULL;
    ret = inflateInit2(strm, -15);
    # 如果初始化失败，则退出并打印错误信息
    if (ret != Z_OK) bye("out of memory", " or library mismatch");

    # 解压缩 deflate 流，保存附加信息
    lastbit = 0;
    # 记录上一次的偏移量
    lastoff = lseek(gz.fd, 0L, SEEK_CUR) - gz.left;
    # 未使用的字节数
    left = 0;
    # 设置输入数据的可用字节数和指针
    strm->avail_in = gz.left;
    strm->next_in = gz.next;
    # 初始化 CRC 校验值
    crc = crc32(0L, Z_NULL, 0);
    # 已经读取的字节数和完整的字节数
    have = full = 0;
    do {
        /* 如果需要，获取更多输入 */
        if (strm->avail_in == 0) {
            readmore(&gz);
            strm->avail_in = gz.left;
            strm->next_in = gz.next;
        }

        /* 设置输出到滑动窗口的下一个可用部分 */
        strm->avail_out = DSIZE - have;
        strm->next_out = window + have;

        /* 解压并检查错误 */
        ret = inflate(strm, Z_BLOCK);
        if (ret == Z_STREAM_ERROR) bye("internal stream error!", "");
        if (ret == Z_MEM_ERROR) bye("out of memory", "");
        if (ret == Z_DATA_ERROR)
            bye("invalid compressed data--format violated in", name);

        /* 更新 crc 和滑动窗口指针 */
        crc = crc32(crc, window + have, DSIZE - have - strm->avail_out);
        if (strm->avail_out)
            have = DSIZE - strm->avail_out;
        else {
            have = 0;
            full = 1;
        }

        /* 处理块结束 */
        if (strm->data_type & 128) {
            if (strm->data_type & 64)
                left = strm->data_type & 0x1f;
            else {
                lastbit = strm->data_type & 0x1f;
                lastoff = lseek(gz.fd, 0L, SEEK_CUR) - strm->avail_in;
            }
        }
    } while (ret != Z_STREAM_END);
    inflateEnd(strm);
    gz.left = strm->avail_in;
    gz.next = strm->next_in;

    /* 保存压缩数据的结束位置 */
    end = lseek(gz.fd, 0L, SEEK_CUR) - gz.left;

    /* 检查 gzip 尾部并保存总长度用于压缩 */
    if (crc != read4(&gz))
        bye("invalid compressed data--crc mismatch in ", name);
    tot = strm->total_out;
    if ((tot & 0xffffffffUL) != read4(&gz))
        bye("invalid compressed data--length mismatch in", name);

    /* 如果不是文件结尾，发出警告 */
    if (gz.left || readin(&gz))
        fprintf(stderr,
            "gzappend warning: junk at end of gzip file overwritten\n");

    /* 清除最后一个块位 */
    # 将文件指针移动到指定位置
    lseek(gz.fd, lastoff - (lastbit != 0), SEEK_SET);
    # 如果读取的字节数不等于1，则输出错误信息并退出
    if (read(gz.fd, gz.buf, 1) != 1) bye("reading after seek on ", name);
    # 对读取的字节进行位运算
    *gz.buf = (unsigned char)(*gz.buf ^ (1 << ((8 - lastbit) & 7)));
    # 将文件指针移动到当前位置的前一个位置
    lseek(gz.fd, -1L, SEEK_CUR);
    # 如果写入的字节数不等于1，则输出错误信息并退出
    if (write(gz.fd, gz.buf, 1) != 1) bye("writing after seek to ", name);

    # 如果窗口已满，从窗口中构建字典
    if (full) {
        rotate(window, DSIZE, have);
        have = DSIZE;
    }

    # 使用窗口、crc、total_in和剩余位设置deflate流
    ret = deflateInit2(strm, level, Z_DEFLATED, -15, 8, Z_DEFAULT_STRATEGY);
    # 如果初始化失败，则输出错误信息并退出
    if (ret != Z_OK) bye("out of memory", "");
    # 设置deflate流的字典
    deflateSetDictionary(strm, window, have);
    strm->adler = crc;
    strm->total_in = tot;
    # 如果有剩余位，则进行相关操作
    if (left) {
        lseek(gz.fd, --end, SEEK_SET);
        if (read(gz.fd, gz.buf, 1) != 1) bye("reading after seek on ", name);
        deflatePrime(strm, 8 - left, *gz.buf);
    }
    lseek(gz.fd, end, SEEK_SET);

    # 清理并返回文件描述符
    free(window);
    free(gz.buf);
    return gz.fd;
# 将文件“name”附加到gzip文件gd中，使用deflate流strm -- 如果last为true，则在最后完成deflate流
def gztack(char *name, int gd, z_stream *strm, int last):
    # 定义变量
    int fd, len, ret;
    unsigned left;
    unsigned char *in, *out;

    # 打开要压缩和附加的文件
    fd = 0;
    if (name != NULL):
        fd = open(name, O_RDONLY, 0);
        if (fd == -1):
            fprintf(stderr, "gzappend warning: %s not found, skipping ...\n",
                    name);

    # 分配缓冲区
    in = malloc(CHUNK);
    out = malloc(CHUNK);
    if (in == NULL || out == NULL):
        bye("out of memory", "");

    # 压缩输入文件并附加到gzip文件
    do:
        # 获取更多输入
        len = read(fd, in, CHUNK);
        if (len == -1):
            fprintf(stderr,
                    "gzappend warning: error reading %s, skipping rest ...\n",
                    name);
            len = 0;
        strm->avail_in = (unsigned)len;
        strm->next_in = in;
        if (len) strm->adler = crc32(strm->adler, in, (unsigned)len);

        # 压缩并写入所有可用的输出
        do:
            strm->avail_out = CHUNK;
            strm->next_out = out;
            ret = deflate(strm, last && len == 0 ? Z_FINISH : Z_NO_FLUSH);
            left = CHUNK - strm->avail_out;
            while (left):
                len = write(gd, out + CHUNK - strm->avail_out - left, left);
                if (len == -1):
                    bye("writing gzip file", "");
                left -= (unsigned)len;
        while (strm->avail_out == 0 && ret != Z_STREAM_END);
    while (len != 0);

    # 在最后一个条目后写入尾部
    # 如果已经处理完最后一个数据块
    if (last) {
        # 结束压缩流
        deflateEnd(strm);
        # 将 Adler-32 校验值的每个字节存入输出数组
        out[0] = (unsigned char)(strm->adler);
        out[1] = (unsigned char)(strm->adler >> 8);
        out[2] = (unsigned char)(strm->adler >> 16);
        out[3] = (unsigned char)(strm->adler >> 24);
        # 将输入数据的总字节数的每个字节存入输出数组
        out[4] = (unsigned char)(strm->total_in);
        out[5] = (unsigned char)(strm->total_in >> 8);
        out[6] = (unsigned char)(strm->total_in >> 16);
        out[7] = (unsigned char)(strm->total_in >> 24);
        # 设置输出数据的长度为 8
        len = 8;
        # 循环写入数据，直到所有数据都写入完成
        do {
            # 将数据写入文件描述符，并更新剩余长度
            ret = write(gd, out + 8 - len, len);
            # 如果写入失败，输出错误信息并退出程序
            if (ret == -1) bye("writing gzip file", "");
            len -= ret;
        } while (len);
        # 关闭文件描述符
        close(gd);
    }

    /* 清理并返回 */
    # 释放输出数组的内存
    free(out);
    # 释放输入数组的内存
    free(in);
    # 如果文件描述符大于 0，关闭文件描述符
    if (fd > 0) close(fd);
/* 处理压缩级别选项（如果存在），扫描 gzip 文件，并追加指定的文件，如果命令行中没有提供其他文件名，则追加来自标准输入的数据 -- gzip 文件必须是可写和可寻址的 */
int main(int argc, char **argv)
{
    int gd, level; // 定义整型变量 gd 和 level
    z_stream strm; // 定义 zlib 压缩流对象

    /* 忽略命令名 */
    argc--; argv++;

    /* 如果没有参数，则提供用法 */
    if (*argv == NULL) {
        printf(
            "gzappend 1.2 (11 Oct 2012) Copyright (C) 2003, 2012 Mark Adler\n"
               );
        printf(
            "usage: gzappend [-level] file.gz [ addthis [ andthis ... ]]\n");
        return 0;
    }

    /* 设置压缩级别 */
    level = Z_DEFAULT_COMPRESSION;
    if (argv[0][0] == '-') {
        if (argv[0][1] < '0' || argv[0][1] > '9' || argv[0][2] != 0)
            bye("invalid compression level", "");
        level = argv[0][1] - '0';
        if (*++argv == NULL) bye("no gzip file name after options", "");
    }

    /* 准备追加到 gzip 文件 */
    gd = gzscan(*argv++, &strm, level);

    /* 追加命令行中的文件，如果没有则来自标准输入 */
    if (*argv == NULL)
        gztack(NULL, gd, &strm, 1);
    else
        do {
            gztack(*argv, gd, &strm, argv[1] == NULL);
        } while (*++argv != NULL);
    return 0;
}
```