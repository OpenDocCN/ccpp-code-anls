# `nmap\libz\examples\zran.c`

```cpp
/* zran.c -- zlib/gzip流索引和随机访问的示例
 * 版权所有 (C) 2005, 2012, 2018 Mark Adler
 * 有关分发和使用条件，请参阅zlib.h中的版权声明
 * 版本 1.2  2018年10月14日  Mark Adler */

/* 版本历史:
 1.0  2005年5月29日  第一个版本
 1.1  2012年9月29日  修复内存重新分配错误
 1.2  2018年10月14日  处理具有多个成员的gzip流
                   添加一个头文件以便在应用程序中使用
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "zlib.h"
#include "zran.h"

#define WINSIZE 32768U      /* 滑动窗口大小 */
#define CHUNK 16384         /* 文件输入缓冲区大小 */

/* 访问点条目。*/
struct point {
    off_t out;          /* 未压缩数据中对应的偏移量 */
    off_t in;           /* 第一个完整字节的输入文件偏移量 */
    int bits;           /* 来自in-1处字节的位数（1-7），或者为0 */
    unsigned char window[WINSIZE];  /* 未压缩数据的前32K */
};

/* 请参阅zran.h中的注释。*/
void deflate_index_free(struct deflate_index *index)
{
    if (index != NULL) {
        free(index->list);
        free(index);
    }
}

/* 向访问点列表添加一个条目。如果内存不足，则释放现有列表并返回NULL。
   index->gzip是point条目中索引的分配大小，直到deflate_index_build()返回时，gzip被设置为指示gzip文件或否的值。
 */
static struct deflate_index *addpoint(struct deflate_index *index, int bits,
                                      off_t in, off_t out, unsigned left,
                                      unsigned char *window)
{
    struct point *next;

    /* 如果列表为空，则创建它（从八个点开始） */
    # 如果索引为空，分配内存并初始化
    if (index == NULL) {
        index = malloc(sizeof(struct deflate_index));
        if (index == NULL) return NULL;
        index->list = malloc(sizeof(struct point) << 3);
        if (index->list == NULL) {
            free(index);
            return NULL;
        }
        index->gzip = 8;
        index->have = 0;
    }

    # 如果列表已满，扩大列表的大小
    else if (index->have == index->gzip) {
        index->gzip <<= 1;
        next = realloc(index->list, sizeof(struct point) * index->gzip);
        if (next == NULL) {
            deflate_index_free(index);
            return NULL;
        }
        index->list = next;
    }

    # 填充条目并增加已有条目数量
    next = (struct point *)(index->list) + index->have;
    next->bits = bits;
    next->in = in;
    next->out = out;
    if (left)
        memcpy(next->window, window + WINSIZE - left, left);
    if (left < WINSIZE)
        memcpy(next->window + left, window, WINSIZE - left);
    index->have++;

    # 返回列表，可能已重新分配
    return index;
}

/* 在 zran.h 中查看注释。*/
int deflate_index_build(FILE *in, off_t span, struct deflate_index **built)
{
    int ret;
    int gzip = 0;               /* 如果读取的是 gzip 文件，则为真 */
    off_t totin, totout;        /* 我们自己的总计数器，避免 4GB 限制 */
    off_t last;                 /* 上一个访问点的 totout 值 */
    struct deflate_index *index;    /* 正在生成的访问点 */
    z_stream strm;
    unsigned char input[CHUNK];
    unsigned char window[WINSIZE];

    /* 初始化 inflate */
    strm.zalloc = Z_NULL;
    strm.zfree = Z_NULL;
    strm.opaque = Z_NULL;
    strm.avail_in = 0;
    strm.next_in = Z_NULL;
    ret = inflateInit2(&strm, 47);      /* 自动 zlib 或 gzip 解码 */
    if (ret != Z_OK)
        return ret;

    /* 解压输入，维护一个滑动窗口，并构建一个索引 -- 这也使用 gzip 或 zlib 流中的检查信息验证压缩数据的完整性 */
    totin = totout = last = 0;
    index = NULL;               /* 将由 first addpoint() 分配 */
    strm.avail_out = 0;
    } while (ret != Z_STREAM_END);

    /* 清理并返回索引（释放列表中未使用的条目） */
    (void)inflateEnd(&strm);
    index->list = realloc(index->list, sizeof(struct point) * index->have);
    index->gzip = gzip;
    index->length = totout;
    *built = index;
    return index->have;

    /* 返回错误 */
  deflate_index_build_error:
    (void)inflateEnd(&strm);
    deflate_index_free(index);
    return ret;
}

/* 在 zran.h 中查看注释。*/
int deflate_index_extract(FILE *in, struct deflate_index *index, off_t offset,
                          unsigned char *buf, int len)
{
    int ret, skip;
    z_stream strm;
    struct point *here;
    unsigned char input[CHUNK];
    unsigned char discard[WINSIZE];

    /* 只有在有合理操作时才继续 */
    if (len < 0)
        return 0;

    /* 找到流中开始的位置 */
    here = index->list;
    # 将索引中的已有数据数量赋值给 ret
    ret = index->have;
    # 当 ret 大于 0 且 here[1].out 小于等于 offset 时，执行循环
    while (--ret && here[1].out <= offset)
        # 移动到下一个位置
        here++;

    # 初始化文件和解压缩状态，以便从那里开始
    strm.zalloc = Z_NULL;
    strm.zfree = Z_NULL;
    strm.opaque = Z_NULL;
    strm.avail_in = 0;
    strm.next_in = Z_NULL;
    # 初始化解压缩流，使用原始解压缩
    ret = inflateInit2(&strm, -15);         /* raw inflate */
    # 如果初始化失败，则返回错误码
    if (ret != Z_OK)
        return ret;
    # 移动文件指针到指定位置
    ret = fseeko(in, here->in - (here->bits ? 1 : 0), SEEK_SET);
    # 如果移动失败，则跳转到 deflate_index_extract_ret 标签处
    if (ret == -1)
        goto deflate_index_extract_ret;
    # 如果有额外的位，则读取一个字节并进行预处理
    if (here->bits) {
        ret = getc(in);
        # 如果读取失败，则返回相应的错误码
        if (ret == -1) {
            ret = ferror(in) ? Z_ERRNO : Z_DATA_ERROR;
            goto deflate_index_extract_ret;
        }
        # 进行预处理
        (void)inflatePrime(&strm, here->bits, ret >> (8 - here->bits));
    }
    # 设置解压缩流的字典
    (void)inflateSetDictionary(&strm, here->window, WINSIZE);

    # 跳过未压缩的字节，直到达到偏移量，然后满足请求
    offset -= here->out;
    strm.avail_in = 0;
    skip = 1;                               /* while skipping to offset */
    # 执行循环，直到 skip 为假
    } while (skip);

    # 计算偏移量后读取的未压缩字节数
    ret = skip ? 0 : len - strm.avail_out;

    # 清理并返回读取的字节数，或者返回负错误码
  deflate_index_extract_ret:
    # 结束解压缩流
    (void)inflateEnd(&strm);
    # 返回结果
    return ret;
}
#ifdef TEST

#define SPAN 1048576L       /* desired distance between access points */  // 定义期望的访问点之间的距离
#define LEN 16384           /* number of bytes to extract */  // 要提取的字节数

/* 通过处理命令行提供的文件，并从未压缩的输出的2/3处提取LEN字节，将其写入stdout，演示deflate_index_build()和deflate_index_extract()的使用。可以将偏移量作为第二个参数提供，这样可以从那里提取数据。 */
int main(int argc, char **argv)
{
    int len;
    off_t offset = -1;
    FILE *in;
    struct deflate_index *index = NULL;
    unsigned char buf[LEN];

    /* 打开输入文件 */
    if (argc < 2 || argc > 3) {
        fprintf(stderr, "usage: zran file.gz [offset]\n");
        return 1;
    }
    in = fopen(argv[1], "rb");
    if (in == NULL) {
        fprintf(stderr, "zran: could not open %s for reading\n", argv[1]);
        return 1;
    }

    /* 获取可选的偏移量 */
    if (argc == 3) {
        char *end;
        offset = strtoll(argv[2], &end, 10);
        if (*end || offset < 0) {
            fprintf(stderr, "zran: %s is not a valid offset\n", argv[2]);
            return 1;
        }
    }

    /* 构建索引 */
    len = deflate_index_build(in, SPAN, &index);
    if (len < 0) {
        fclose(in);
        switch (len) {
        case Z_MEM_ERROR:
            fprintf(stderr, "zran: out of memory\n");
            break;
        case Z_DATA_ERROR:
            fprintf(stderr, "zran: compressed data error in %s\n", argv[1]);
            break;
        case Z_ERRNO:
            fprintf(stderr, "zran: read error on %s\n", argv[1]);
            break;
        default:
            fprintf(stderr, "zran: error %d while building index\n", len);
        }
        return 1;
    }
    fprintf(stderr, "zran: built index with %d access points\n", len);

    /* 通过从任意偏移量读取一些字节来使用索引 */
    # 如果偏移量为-1，则将偏移量设置为索引长度的2/3
    if (offset == -1)
        offset = (index->length << 1) / 3;
    # 从输入流中提取数据到缓冲区
    len = deflate_index_extract(in, index, offset, buf, LEN);
    # 如果提取失败，则输出错误信息
    if (len < 0)
        fprintf(stderr, "zran: extraction failed: %s error\n",
                len == Z_MEM_ERROR ? "out of memory" : "input corrupted");
    # 如果提取成功，则将数据写入标准输出，并输出提取的字节数和偏移量
    else {
        fwrite(buf, 1, len, stdout);
        fprintf(stderr, "zran: extracted %d bytes at %llu\n", len, offset);
    }

    /* 清理并退出 */
    # 释放索引内存
    deflate_index_free(index);
    # 关闭输入流
    fclose(in);
    # 返回0表示成功
    return 0;
}
# 结束一个代码块或函数的定义
#endif
# 结束一个条件编译指令的定义
```