# `nmap\libz\examples\gznorm.c`

```
/* gznorm.c -- normalize a gzip stream
 * 对 gzip 流进行规范化
 * Copyright (C) 2018 Mark Adler
 * 版权所有 2018 Mark Adler
 * For conditions of distribution and use, see copyright notice in zlib.h
 * 分发和使用条件，请参阅 zlib.h 中的版权声明
 * Version 1.0  7 Oct 2018  Mark Adler
 * 版本 1.0  2018年10月7日  Mark Adler */

// gznorm takes a gzip stream, potentially containing multiple members, and
// converts it to a gzip stream with a single member. In addition the gzip
// header is normalized, removing the file name and time stamp, and setting the
// other header contents (XFL, OS) to fixed values. gznorm does not recompress
// the data, so it is fast, but no advantage is gained from the history that
// could be available across member boundaries.
// gznorm 接受一个 gzip 流，可能包含多个成员，并将其转换为只包含一个成员的 gzip 流。此外，gzip 头被规范化，删除文件名和时间戳，并将其他头内容（XFL、OS）设置为固定值。gznorm 不会重新压缩数据，因此速度很快，但无法从可能跨成员边界可用的历史中获得优势。

#include <stdio.h>      // fread, fwrite, putc, fflush, ferror, fprintf,
                        // vsnprintf, stdout, stderr, NULL, FILE
#include <stdlib.h>     // malloc, free
#include <string.h>     // strerror
#include <errno.h>      // errno
#include <stdarg.h>     // va_list, va_start, va_end
#include "zlib.h"       // inflateInit2, inflate, inflateReset, inflateEnd,
                        // z_stream, z_off_t, crc32_combine, Z_NULL, Z_BLOCK,
                        // Z_OK, Z_STREAM_END, Z_BUF_ERROR, Z_DATA_ERROR,
                        // Z_MEM_ERROR

#if defined(MSDOS) || defined(OS2) || defined(WIN32) || defined(__CYGWIN__)
#  include <fcntl.h>
#  include <io.h>
#  define SET_BINARY_MODE(file) setmode(fileno(file), O_BINARY)
#else
#  define SET_BINARY_MODE(file)
#endif

#define local static

// printf to an allocated string. Return the string, or NULL if the printf or
// allocation fails.
// 将 printf 输出到分配的字符串。返回字符串，如果 printf 或分配失败则返回 NULL。
local char *aprintf(char *fmt, ...) {
    // Get the length of the result of the printf.
    // 获取 printf 结果的长度
    va_list args;
    va_start(args, fmt);
    int len = vsnprintf(NULL, 0, fmt, args);
    va_end(args);
    if (len < 0)
        return NULL;

    // Allocate the required space and printf to it.
    // 分配所需的空间并将 printf 输出到其中
    char *str = malloc(len + 1);
    if (str == NULL)
        return NULL;
    va_start(args, fmt);
    vsnprintf(str, len + 1, fmt, args);
    va_end(args);
    return str;
// 返回一个错误，将分配的错误消息放入 *err 中。在已结束的状态上执行 inflateEnd()，或者状态设置为 Z_NULL，是允许的。
#define BYE(...) \
    do { \
        inflateEnd(&strm); \
        *err = aprintf(__VA_ARGS__); \
        return 1; \
    } while (0)

// 缓冲读取和解压缩的块大小。堆栈上将分配两倍于这么多字节的空间。必须适合无符号整数。
#define CHUNK 16384

// 从输入流中读取一个 gzip 流，并将等效的标准化 gzip 流写入输出流。如果没有输入，则将写入一个空的 gzip 流。如果成功，返回 0，并将 *err 设置为 NULL。发生错误时，返回 1，其中错误的详细信息在 *err 中返回，即分配的字符串的指针。
//
// 输入可以是具有多个 gzip 成员的流，这些成员转换为输出的单个 gzip 成员。每个 gzip 成员都在 deflate 块的级别上进行解压缩。这使得清除最后一个块位，将压缩数据移位以连接到前一个成员的压缩数据，该数据可以在任意位边界结束，并识别存储的块以将其重新同步到字节边界。deflate 压缩数据以 10 位空的固定块终止。如果输入的任何成员以 10 位空的固定块结尾，则该块将从流中切除。这避免了为每个标准化附加空的固定块，并确保 gzip_normalize 第二次应用不会更改输入。存储块标题后和最终 deflate 块后的填充位都被强制为零。
local int gzip_normalize(FILE *in, FILE *out, char **err) {
    // 初始化 inflate 引擎以处理 gzip 成员
    z_stream strm;
    strm.zalloc = Z_NULL;
    strm.zfree = Z_NULL;
    strm.opaque = Z_NULL;
    strm.avail_in = 0;
    strm.next_in = Z_NULL;
    # 初始化压缩流，使用默认的压缩级别和标准的 gzip 格式
    if (inflateInit2(&strm, 15 + 16) != Z_OK)
        BYE("out of memory");

    # 处理输入的 gzip 流时的状态
    enum {              // BETWEEN -> HEAD -> BLOCK -> TAIL -> BETWEEN -> ...
        BETWEEN,        // 在两个 gzip 成员之间（必须以这个状态结束）
        HEAD,           // 读取 gzip 头部
        BLOCK,          // 读取 deflate 块
        TAIL            // 读取 gzip 尾部
    } state = BETWEEN;              // 当前正在处理的组件
    unsigned long crc = 0;          // 未压缩数据的累积 CRC
    unsigned long len = 0;          // 未压缩数据的累积长度
    unsigned long buf = 0;          // num 位的 deflate 流位缓冲
    int num = 0;                    // buf 中的位数（从底部开始）

    # 写入一个标准的 gzip 头部（没有修改时间、文件名、注释、额外块或额外标志，并且 OS 标记为未知）
    fwrite("\x1f\x8b\x08\0\0\0\0\0\0\xff", 1, 10, out);

    # 处理输入的 gzip 流，直到到达输入的末尾、遇到无效输入或遇到 I/O 错误
    int more;                       // 如果没有到达输入的末尾，则为 true
    } while (more);

    # 完成解压缩引擎
    inflateEnd(&strm);

    # 验证输入的有效性
    if (state != BETWEEN)
        BYE("input invalid: incomplete gzip stream");

    # 写入剩余的 deflate 流位，后跟一个终止的 deflate 固定块
    buf += (unsigned long)3 << num;
    putc(buf, out);
    putc(buf >> 8, out);
    if (num > 6)
        putc(0, out);

    # 写入 gzip 尾部，即 CRC 和未压缩长度模 2^32，都以小端序写入
    putc(crc, out);
    putc(crc >> 8, out);
    putc(crc >> 16, out);
    putc(crc >> 24, out);
    putc(len, out);
    putc(len >> 8, out);
    putc(len >> 16, out);
    putc(len >> 24, out);
    fflush(out);
    // 检查是否有任何输入/输出错误
    if (ferror(in) || ferror(out))
        // 如果有错误，输出错误信息并退出
        BYE("i/o error: %s", strerror(errno));

    // 如果没有错误，将错误指针置为空
    *err = NULL;
    // 返回 0 表示一切正常
    return 0;
// 主函数，将标准输入的 gzip 流进行规范化处理，并将结果写入标准输出
int main(void) {
    // 避免在一些操作系统上出现换行符转换的问题
    SET_BINARY_MODE(stdin);
    SET_BINARY_MODE(stdout);

    // 从标准输入规范化到标准输出，如果出错返回1，正常返回0
    char *err;
    int ret = gzip_normalize(stdin, stdout, &err);
    if (ret)
        fprintf(stderr, "gznorm error: %s\n", err);
    free(err);
    return ret;
}
```