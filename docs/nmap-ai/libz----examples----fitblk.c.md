# `nmap\libz\examples\fitblk.c`

```
/* fitblk.c: 压缩输出以适应指定大小的示例
   无版权 -- 提供给公众领域
   版本 1.1  2004年11月25日  Mark Adler */

/* 版本历史:
   1.0  2004年11月24日  第一个版本
   1.1  2004年11月25日  将 deflateInit2() 更改为 deflateInit()
                     使用固定大小的栈分配原始缓冲区
                     简化将压缩移动到子例程的代码
                     对内部错误使用 assert()
                     添加方法的详细描述
 */

#include <stdio.h>
#include <stdlib.h>
#include <assert.h>
#include "zlib.h"

#define local static

/* 打印错误信息并退出 */
local void quit(char *why)
{
    fprintf(stderr, "fitblk abort: %s\n", why);
    exit(1);
}

#define RAWLEN 4096    /* 中间未压缩缓冲区的大小 */

/* 从文件压缩到 def 直到提供的缓冲区已满或到达输入的末尾; 返回最后一个 deflate() 返回值，如果文件读取错误则返回 Z_ERRNO */
local int partcompress(FILE *in, z_streamp def)
{
    int ret, flush;
    unsigned char raw[RAWLEN];

    flush = Z_NO_FLUSH;
    do {
        def->avail_in = fread(raw, 1, RAWLEN, in);
        if (ferror(in))
            return Z_ERRNO;
        def->next_in = raw;
        if (feof(in))
            flush = Z_FINISH;
        ret = deflate(def, flush);
        assert(ret != Z_STREAM_ERROR);
    } while (def->avail_out != 0 && flush == Z_NO_FLUSH);
    return ret;
}

/* 从 inf 的输入重新压缩到 def 的输出; 在调用之前，设置 inf 的输入和 def 的输出;
   返回最后一个 deflate() 返回值，如果 inflate() 在需要时无法分配足够的内存，则返回 Z_MEM_ERROR */
local int recompress(z_streamp inf, z_streamp def)
{
    int ret, flush;
    unsigned char raw[RAWLEN];

    flush = Z_NO_FLUSH;
    // 进行解压缩操作
    inf->avail_out = RAWLEN;  // 设置输出缓冲区大小
    inf->next_out = raw;  // 设置输出缓冲区位置
    ret = inflate(inf, Z_NO_FLUSH);  // 调用解压缩函数
    assert(ret != Z_STREAM_ERROR && ret != Z_DATA_ERROR && ret != Z_NEED_DICT);  // 检查解压缩返回值
    if (ret == Z_MEM_ERROR)  // 如果返回内存错误，则直接返回
        return ret;

    // 压缩解压缩后的内容，直到完成或没有足够的空间
    def->avail_in = RAWLEN - inf->avail_out;  // 设置输入缓冲区大小
    def->next_in = raw;  // 设置输入缓冲区位置
    if (inf->avail_out != 0)  // 如果输出缓冲区不为空
        flush = Z_FINISH;  // 设置压缩结束标志
    ret = deflate(def, flush);  // 调用压缩函数
    assert(ret != Z_STREAM_ERROR);  // 检查压缩返回值
} while (ret != Z_STREAM_END && def->avail_out != 0);  // 当未完成压缩且输出缓冲区不为空时循环
return ret;  // 返回压缩结果
}  // 结束 main 函数

#define EXCESS 256      /* empirically determined stream overage */  // 定义一个常量 EXCESS，表示流的过量大小
#define MARGIN 8        /* amount to back off for completion */  // 定义一个常量 MARGIN，表示用于完成的回退量

/* compress from stdin to fixed-size block on stdout */  // 从标准输入压缩到固定大小的块上输出到标准输出
int main(int argc, char **argv)  // 主函数，接受命令行参数
{
    int ret;                /* return code */  // 返回值
    unsigned size;          /* requested fixed output block size */  // 请求的固定输出块大小
    unsigned have;          /* bytes written by deflate() call */  // deflate() 调用写入的字节数
    unsigned char *blk;     /* intermediate and final stream */  // 中间和最终流
    unsigned char *tmp;     /* close to desired size stream */  // 接近期望大小的流
    z_stream def, inf;      /* zlib deflate and inflate states */  // zlib 压缩和解压状态

    /* get requested output size */  // 获取请求的输出大小
    if (argc != 2)  // 如果参数个数不为 2
        quit("need one argument: size of output block");  // 退出并提示需要一个参数：输出块的大小
    ret = strtol(argv[1], argv + 1, 10);  // 将参数转换为整数
    if (argv[1][0] != 0)  // 如果参数的第一个字符不为 0
        quit("argument must be a number");  // 退出并提示参数必须是一个数字
    if (ret < 8)            /* 8 is minimum zlib stream size */  // 如果 ret 小于 8
        quit("need positive size of 8 or greater");  // 退出并提示需要大于等于 8 的正数大小
    size = (unsigned)ret;  // 将 ret 转换为无符号整数赋值给 size

    /* allocate memory for buffers and compression engine */  // 为缓冲区和压缩引擎分配内存
    blk = malloc(size + EXCESS);  // 分配 size + EXCESS 大小的内存给 blk
    def.zalloc = Z_NULL;  // 设置 def 的内存分配函数为 Z_NULL
    def.zfree = Z_NULL;  // 设置 def 的内存释放函数为 Z_NULL
    def.opaque = Z_NULL;  // 设置 def 的不透明指针为 Z_NULL
    ret = deflateInit(&def, Z_DEFAULT_COMPRESSION);  // 初始化压缩引擎
    if (ret != Z_OK || blk == NULL)  // 如果返回值不为 Z_OK 或者 blk 为 NULL
        quit("out of memory");  // 退出并提示内存不足

    /* compress from stdin until output full, or no more input */  // 从标准输入压缩直到输出完全，或者没有更多输入
    def.avail_out = size + EXCESS;  // 设置可用输出大小
    def.next_out = blk;  // 设置输出缓冲区
    ret = partcompress(stdin, &def);  // 调用 partcompress 函数进行压缩
    if (ret == Z_ERRNO)  // 如果返回值为 Z_ERRNO
        quit("error reading input");  // 退出并提示读取输入时出错

    /* if it all fit, then size was undersubscribed -- done! */  // 如果全部适合，那么大小被订阅不足 - 完成！
    if (ret == Z_STREAM_END && def.avail_out >= EXCESS) {
        /* 如果压缩结束并且输出缓冲区中剩余空间大于等于EXCESS */
        have = size + EXCESS - def.avail_out;
        /* 计算实际写入输出流的数据大小 */
        if (fwrite(blk, 1, have, stdout) != have || ferror(stdout))
            quit("error writing output");
        /* 将数据块写入标准输出流，如果出错则退出程序 */

        /* 清理并将结果打印到标准错误流 */
        ret = deflateEnd(&def);
        assert(ret != Z_STREAM_ERROR);
        free(blk);
        fprintf(stderr,
                "%u bytes unused out of %u requested (all input)\n",
                size - have, size);
        return 0;
    }

    /* 如果数据没有完全写入输出缓冲区 -- 设置重新压缩 */
    inf.zalloc = Z_NULL;
    inf.zfree = Z_NULL;
    inf.opaque = Z_NULL;
    inf.avail_in = 0;
    inf.next_in = Z_NULL;
    ret = inflateInit(&inf);
    tmp = malloc(size + EXCESS);
    if (ret != Z_OK || tmp == NULL)
        quit("out of memory");
    ret = deflateReset(&def);
    assert(ret != Z_STREAM_ERROR);

    /* 进行第一次重新压缩，接近正确的大小 */
    inf.avail_in = size + EXCESS;
    inf.next_in = blk;
    def.avail_out = size + EXCESS;
    def.next_out = tmp;
    ret = recompress(&inf, &def);
    if (ret == Z_MEM_ERROR)
        quit("out of memory");

    /* 设置下一次重新压缩 */
    ret = inflateReset(&inf);
    assert(ret != Z_STREAM_ERROR);
    ret = deflateReset(&def);
    assert(ret != Z_STREAM_ERROR);

    /* 进行第二次和最终的重新压缩（第三次压缩） */
    inf.avail_in = size - MARGIN;   /* 确保流能够完成 */
    inf.next_in = tmp;
    def.avail_out = size;
    def.next_out = blk;
    ret = recompress(&inf, &def);
    if (ret == Z_MEM_ERROR)
        quit("out of memory");
    assert(ret == Z_STREAM_END);    /* 否则MARGIN太小 */

    /* 完成 -- 将数据块写入标准输出流 */
    have = size - def.avail_out;
    if (fwrite(blk, 1, have, stdout) != have || ferror(stdout))
        quit("error writing output");

    /* 清理并将结果打印到标准错误流 */
    free(tmp);
    ret = inflateEnd(&inf);
    # 确保返回值不是 Z_STREAM_ERROR
    assert(ret != Z_STREAM_ERROR);
    # 结束压缩流
    ret = deflateEnd(&def);
    # 确保返回值不是 Z_STREAM_ERROR
    assert(ret != Z_STREAM_ERROR);
    # 释放内存
    free(blk);
    # 打印未使用的字节数、请求的字节数和输入的字节数到标准错误流
    fprintf(stderr,
            "%u bytes unused out of %u requested (%lu input)\n",
            size - have, size, def.total_in);
    # 返回 0 表示成功
    return 0;
# 闭合前面的函数定义
```