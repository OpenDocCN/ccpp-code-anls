# `nmap\libz\examples\zpipe.c`

```
/* zpipe.c: zlib的inflate()和deflate()的正确使用示例
   不受版权保护 -- 提供给公众领域
   版本 1.4  2005年12月11日  Mark Adler */

/* 版本历史:
   1.0  2004年10月30日  第一个版本
   1.1  2004年11月8日   为未使用的返回值添加void转换
                        使用switch语句处理inflate()的返回值
   1.2  2004年11月9日   添加断言以记录zlib的保证
   1.3  2005年4月6日    在inf()中删除不正确的断言
   1.4  2005年12月11日  添加hack以避免MSDOS的换行转换
                        避免一些编译器对输入和输出缓冲区的警告
 */

#include <stdio.h>
#include <string.h>
#include <assert.h>
#include "zlib.h"

#if defined(MSDOS) || defined(OS2) || defined(WIN32) || defined(__CYGWIN__)
#  include <fcntl.h>
#  include <io.h>
#  define SET_BINARY_MODE(file) setmode(fileno(file), O_BINARY)
#else
#  define SET_BINARY_MODE(file)
#endif

#define CHUNK 16384

/* 从源文件压缩到目标文件，直到源文件结束。
   def()在成功时返回Z_OK，在无法分配内存进行处理时返回Z_MEM_ERROR，
   在提供无效的压缩级别时返回Z_STREAM_ERROR，
   在zlib.h的版本和链接库的版本不匹配时返回Z_VERSION_ERROR，
   或者在读取或写入文件时出现错误时返回Z_ERRNO。 */
int def(FILE *source, FILE *dest, int level)
{
    int ret, flush;
    unsigned have;
    z_stream strm;
    unsigned char in[CHUNK];
    unsigned char out[CHUNK];

    /* 分配deflate状态 */
    strm.zalloc = Z_NULL;
    strm.zfree = Z_NULL;
    strm.opaque = Z_NULL;
    ret = deflateInit(&strm, level);
    if (ret != Z_OK)
        return ret;

    /* 压缩直到文件结束 */
    // 使用 do-while 循环来处理压缩数据，直到所有数据都被处理完毕
    do {
        // 从输入源读取数据到输入缓冲区
        strm.avail_in = fread(in, 1, CHUNK, source);
        // 如果读取输入源出错，则结束压缩并返回错误
        if (ferror(source)) {
            (void)deflateEnd(&strm);
            return Z_ERRNO;
        }
        // 如果已到达输入源文件末尾，则设置压缩模式为结束
        flush = feof(source) ? Z_FINISH : Z_NO_FLUSH;
        // 设置输入缓冲区
        strm.next_in = in;

        /* 在输入上运行 deflate()，直到输出缓冲区不再满，如果所有输入都已读取，则完成压缩 */
        do {
            // 设置输出缓冲区大小
            strm.avail_out = CHUNK;
            // 设置输出缓冲区
            strm.next_out = out;
            // 运行 deflate()，没有错误返回值
            ret = deflate(&strm, flush);
            // 确保状态没有被破坏
            assert(ret != Z_STREAM_ERROR);
            // 计算已经压缩的数据大小
            have = CHUNK - strm.avail_out;
            // 将压缩后的数据写入目标文件，如果出错则结束压缩并返回错误
            if (fwrite(out, 1, have, dest) != have || ferror(dest)) {
                (void)deflateEnd(&strm);
                return Z_ERRNO;
            }
        } while (strm.avail_out == 0);
        // 确保所有输入都被使用
        assert(strm.avail_in == 0);

        /* 当处理完文件中的最后一部分数据时结束循环 */
    } while (flush != Z_FINISH);
    // 确保流已经完成
    assert(ret == Z_STREAM_END);

    /* 清理并返回 */
    (void)deflateEnd(&strm);
    return Z_OK;
/* 从源文件解压缩到目标文件，直到流结束或到达文件结尾。
   如果成功，inf() 返回 Z_OK；如果无法为处理分配内存，则返回 Z_MEM_ERROR；
   如果压缩数据无效或不完整，则返回 Z_DATA_ERROR；
   如果 zlib.h 的版本与链接的库的版本不匹配，则返回 Z_VERSION_ERROR；
   如果读取或写入文件时出现错误，则返回 Z_ERRNO。 */
int inf(FILE *source, FILE *dest)
{
    int ret;  // 存储函数返回值
    unsigned have;  // 已经处理的输出数据大小
    z_stream strm;  // zlib 解压缩流
    unsigned char in[CHUNK];  // 输入缓冲区
    unsigned char out[CHUNK];  // 输出缓冲区

    /* 分配解压缩状态 */
    strm.zalloc = Z_NULL;
    strm.zfree = Z_NULL;
    strm.opaque = Z_NULL;
    strm.avail_in = 0;
    strm.next_in = Z_NULL;
    ret = inflateInit(&strm);  // 初始化解压缩流
    if (ret != Z_OK)
        return ret;

    /* 解压缩直到压缩流结束或文件结束 */
    do {
        strm.avail_in = fread(in, 1, CHUNK, source);  // 从源文件读取数据到输入缓冲区
        if (ferror(source)) {
            (void)inflateEnd(&strm);  // 释放解压缩流
            return Z_ERRNO;
        }
        if (strm.avail_in == 0)
            break;
        strm.next_in = in;

        /* 在输出缓冲区不满的情况下运行 inflate() */
        do {
            strm.avail_out = CHUNK;  // 设置输出缓冲区大小
            strm.next_out = out;  // 设置输出缓冲区位置
            ret = inflate(&strm, Z_NO_FLUSH);  // 解压缩数据
            assert(ret != Z_STREAM_ERROR);  /* 状态未被破坏 */
            switch (ret) {
            case Z_NEED_DICT:
                ret = Z_DATA_ERROR;     /* 然后继续执行 */
            case Z_DATA_ERROR:
            case Z_MEM_ERROR:
                (void)inflateEnd(&strm);  // 释放解压缩流
                return ret;
            }
            have = CHUNK - strm.avail_out;  // 计算已经处理的输出数据大小
            if (fwrite(out, 1, have, dest) != have || ferror(dest)) {
                (void)inflateEnd(&strm);  // 释放解压缩流
                return Z_ERRNO;
            }
        } while (strm.avail_out == 0);

        /* 当 inflate() 表示完成时结束 */
    } while (ret != Z_STREAM_END);

    /* 清理并返回 */
    (void)inflateEnd(&strm);  // 释放解压缩流
    # 如果返回值 ret 等于 Z_STREAM_END，则返回 Z_OK，否则返回 Z_DATA_ERROR
    return ret == Z_STREAM_END ? Z_OK : Z_DATA_ERROR;
/* 报告 zlib 或 I/O 错误 */
void zerr(int ret)
{
    // 输出错误信息前缀
    fputs("zpipe: ", stderr);
    switch (ret) {
    case Z_ERRNO:
        // 如果从 stdin 读取时出错，输出错误信息
        if (ferror(stdin))
            fputs("error reading stdin\n", stderr);
        // 如果向 stdout 写入时出错，输出错误信息
        if (ferror(stdout))
            fputs("error writing stdout\n", stderr);
        break;
    case Z_STREAM_ERROR:
        // 输出无效的压缩级别错误信息
        fputs("invalid compression level\n", stderr);
        break;
    case Z_DATA_ERROR:
        // 输出无效或不完整的解压缩数据错误信息
        fputs("invalid or incomplete deflate data\n", stderr);
        break;
    case Z_MEM_ERROR:
        // 输出内存不足错误信息
        fputs("out of memory\n", stderr);
        break;
    case Z_VERSION_ERROR:
        // 输出 zlib 版本不匹配错误信息
        fputs("zlib version mismatch!\n", stderr);
    }
}

/* 从 stdin 压缩或解压缩到 stdout */
int main(int argc, char **argv)
{
    int ret;

    /* 避免进行行尾转换 */
    SET_BINARY_MODE(stdin);
    SET_BINARY_MODE(stdout);

    /* 如果没有参数，则进行压缩 */
    if (argc == 1) {
        // 调用 def 函数进行压缩
        ret = def(stdin, stdout, Z_DEFAULT_COMPRESSION);
        // 如果返回值不是 Z_OK，则调用 zerr 函数输出错误信息
        if (ret != Z_OK)
            zerr(ret);
        return ret;
    }

    /* 如果指定了 -d 参数，则进行解压缩 */
    else if (argc == 2 && strcmp(argv[1], "-d") == 0) {
        // 调用 inf 函数进行解压缩
        ret = inf(stdin, stdout);
        // 如果返回值不是 Z_OK，则调用 zerr 函数输出错误信息
        if (ret != Z_OK)
            zerr(ret);
        return ret;
    }

    /* 否则，报告使用方法 */
    else {
        // 输出 zpipe 的使用方法
        fputs("zpipe usage: zpipe [-d] < source > dest\n", stderr);
        return 1;
    }
}
```