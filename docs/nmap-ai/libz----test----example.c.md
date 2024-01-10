# `nmap\libz\test\example.c`

```
/* example.c -- zlib压缩库的用法示例
 * 版权所有 (C) 1995-2006, 2011, 2016 Jean-loup Gailly
 * 发行和使用条件，请参见 zlib.h 中的版权声明
 */

/* @(#) $Id$ */

#include "zlib.h"  // 包含 zlib 压缩库的头文件
#include <stdio.h>  // 包含标准输入输出库的头文件

#ifdef STDC
#  include <string.h>  // 包含字符串处理库的头文件
#  include <stdlib.h>  // 包含标准库的头文件
#endif

#if defined(VMS) || defined(RISCOS)
#  define TESTFILE "foo-gz"  // 定义测试文件名
#else
#  define TESTFILE "foo.gz"  // 定义测试文件名
#endif

#define CHECK_ERR(err, msg) { \  // 定义宏，用于检查错误并输出错误信息
    if (err != Z_OK) { \
        fprintf(stderr, "%s error: %d\n", msg, err); \
        exit(1); \
    } \
}

static z_const char hello[] = "hello, hello!";  // 定义静态常量字符串 hello

static const char dictionary[] = "hello";  // 定义常量字符串 dictionary
static uLong dictId;    /* Adler32 value of the dictionary */  // 定义 Adler32 校验值

void test_deflate       OF((Byte *compr, uLong comprLen));  // 声明函数 test_deflate
void test_inflate       OF((Byte *compr, uLong comprLen,  // 声明函数 test_inflate
                            Byte *uncompr, uLong uncomprLen));
void test_large_deflate OF((Byte *compr, uLong comprLen,  // 声明函数 test_large_deflate
                            Byte *uncompr, uLong uncomprLen));
void test_large_inflate OF((Byte *compr, uLong comprLen,  // 声明函数 test_large_inflate
                            Byte *uncompr, uLong uncomprLen));
void test_flush         OF((Byte *compr, uLong *comprLen));  // 声明函数 test_flush
void test_sync          OF((Byte *compr, uLong comprLen,  // 声明函数 test_sync
                            Byte *uncompr, uLong uncomprLen));
void test_dict_deflate  OF((Byte *compr, uLong comprLen));  // 声明函数 test_dict_deflate
void test_dict_inflate  OF((Byte *compr, uLong comprLen,  // 声明函数 test_dict_inflate
                            Byte *uncompr, uLong uncomprLen));
int  main               OF((int argc, char *argv[]));  // 声明主函数 main

#ifdef Z_SOLO

void *myalloc OF((void *, unsigned, unsigned));  // 声明自定义内存分配函数
void myfree OF((void *, void *));  // 声明自定义内存释放函数

void *myalloc(q, n, m)  // 实现自定义内存分配函数
    void *q;
    unsigned n, m;
{
    (void)q;
    return calloc(n, m);
}

void myfree(void *q, void *p)  // 实现自定义内存释放函数
{
    (void)q;
    free(p);
}

static alloc_func zalloc = myalloc;  // 定义自定义内存分配函数指针
static free_func zfree = myfree;  // 定义自定义内存释放函数指针
#else /* !Z_SOLO */
# 如果没有定义 Z_SOLO 宏，则执行以下代码

static alloc_func zalloc = (alloc_func)0;
# 定义静态变量 zalloc，用于分配内存，初始值为 0
static free_func zfree = (free_func)0;
# 定义静态变量 zfree，用于释放内存，初始值为 0

void test_compress      OF((Byte *compr, uLong comprLen,
                            Byte *uncompr, uLong uncomprLen));
# 声明函数 test_compress，用于测试压缩和解压缩

void test_gzio          OF((const char *fname,
                            Byte *uncompr, uLong uncomprLen));
# 声明函数 test_gzio，用于测试读写 .gz 文件

/* ===========================================================================
 * Test compress() and uncompress()
 */
# 测试 compress() 和 uncompress() 函数
void test_compress(compr, comprLen, uncompr, uncomprLen)
    Byte *compr, *uncompr;
    uLong comprLen, uncomprLen;
# 定义函数 test_compress，接受压缩数据、压缩数据长度、解压数据和解压数据长度作为参数
{
    int err;
    uLong len = (uLong)strlen(hello)+1;
    # 计算字符串 hello 的长度，并加 1

    err = compress(compr, &comprLen, (const Bytef*)hello, len);
    # 使用 compress() 函数对字符串 hello 进行压缩
    CHECK_ERR(err, "compress");
    # 检查压缩是否出错

    strcpy((char*)uncompr, "garbage");
    # 将字符串 "garbage" 复制到 uncompr 指向的内存中

    err = uncompress(uncompr, &uncomprLen, compr, comprLen);
    # 使用 uncompress() 函数对压缩数据进行解压
    CHECK_ERR(err, "uncompress");
    # 检查解压是否出错

    if (strcmp((char*)uncompr, hello)) {
        fprintf(stderr, "bad uncompress\n");
        exit(1);
    } else {
        printf("uncompress(): %s\n", (char *)uncompr);
    }
    # 如果解压后的数据与原始数据不相同，则输出错误信息，否则输出解压后的数据
}

/* ===========================================================================
 * Test read/write of .gz files
 */
# 测试读写 .gz 文件
void test_gzio(fname, uncompr, uncomprLen)
    const char *fname; /* compressed file name */
    Byte *uncompr;
    uLong uncomprLen;
# 定义函数 test_gzio，接受压缩文件名、解压数据和解压数据长度作为参数
{
#ifdef NO_GZCOMPRESS
    fprintf(stderr, "NO_GZCOMPRESS -- gz* functions cannot compress\n");
# 如果定义了 NO_GZCOMPRESS 宏，则输出错误信息
#else
    int err;
    int len = (int)strlen(hello)+1;
    # 计算字符串 hello 的长度，并加 1
    gzFile file;
    z_off_t pos;

    file = gzopen(fname, "wb");
    # 以写入模式打开压缩文件
    if (file == NULL) {
        fprintf(stderr, "gzopen error\n");
        exit(1);
    }
    gzputc(file, 'h');
    # 写入字符 'h' 到文件中
    if (gzputs(file, "ello") != 4) {
        fprintf(stderr, "gzputs err: %s\n", gzerror(file, &err));
        exit(1);
    }
    # 写入字符串 "ello" 到文件中
    if (gzprintf(file, ", %s!", "hello") != 8) {
        fprintf(stderr, "gzprintf err: %s\n", gzerror(file, &err));
        exit(1);
    }
    # 格式化输出 ", hello!" 到文件中
    gzseek(file, 1L, SEEK_CUR); /* add one zero byte */
    # 在当前位置后移 1 个字节
    gzclose(file);
    # 关闭文件

    file = gzopen(fname, "rb");
    # 以读取模式打开压缩文件
    # 如果文件指针为空，输出错误信息并退出程序
    if (file == NULL) {
        fprintf(stderr, "gzopen error\n");
        exit(1);
    }
    # 将字符串 "garbage" 复制到 uncompr 指向的内存中
    strcpy((char*)uncompr, "garbage");

    # 从文件中读取指定长度的数据到 uncompr 指向的内存中，如果读取长度不等于 len，则输出错误信息并退出程序
    if (gzread(file, uncompr, (unsigned)uncomprLen) != len) {
        fprintf(stderr, "gzread err: %s\n", gzerror(file, &err));
        exit(1);
    }
    # 如果读取的数据不等于 hello 字符串，则输出错误信息并退出程序；否则输出读取的数据
    if (strcmp((char*)uncompr, hello)) {
        fprintf(stderr, "bad gzread: %s\n", (char*)uncompr);
        exit(1);
    } else {
        printf("gzread(): %s\n", (char*)uncompr);
    }

    # 从当前位置向前移动 8 个字节，如果移动的位置不等于 6 或者当前位置不等于移动后的位置，则输出错误信息并退出程序
    pos = gzseek(file, -8L, SEEK_CUR);
    if (pos != 6 || gztell(file) != pos) {
        fprintf(stderr, "gzseek error, pos=%ld, gztell=%ld\n",
                (long)pos, (long)gztell(file));
        exit(1);
    }

    # 从文件中读取一个字符，如果读取的字符不是空格，则输出错误信息并退出程序
    if (gzgetc(file) != ' ') {
        fprintf(stderr, "gzgetc error\n");
        exit(1);
    }

    # 将字符空格推回到文件中，如果推回的字符不是空格，则输出错误信息并退出程序
    if (gzungetc(' ', file) != ' ') {
        fprintf(stderr, "gzungetc error\n");
        exit(1);
    }

    # 从文件中读取一行数据到 uncompr 指向的内存中，如果读取的数据长度不等于 7，则输出错误信息并退出程序
    gzgets(file, (char*)uncompr, (int)uncomprLen);
    if (strlen((char*)uncompr) != 7) { /* " hello!" */
        fprintf(stderr, "gzgets err after gzseek: %s\n", gzerror(file, &err));
        exit(1);
    }
    # 如果读取的数据不等于 hello 字符串的子串，则输出错误信息并退出程序；否则输出读取的数据
    if (strcmp((char*)uncompr, hello + 6)) {
        fprintf(stderr, "bad gzgets after gzseek\n");
        exit(1);
    } else {
        printf("gzgets() after gzseek: %s\n", (char*)uncompr);
    }

    # 关闭文件
    gzclose(file);
#endif
}

#endif /* Z_SOLO */

/* ===========================================================================
 * 使用小缓冲区测试 deflate() 函数
 */
void test_deflate(compr, comprLen)
    Byte *compr;
    uLong comprLen;
{
    z_stream c_stream; /* 压缩流 */
    int err;
    uLong len = (uLong)strlen(hello)+1;

    c_stream.zalloc = zalloc;
    c_stream.zfree = zfree;
    c_stream.opaque = (voidpf)0;

    err = deflateInit(&c_stream, Z_DEFAULT_COMPRESSION);
    CHECK_ERR(err, "deflateInit");

    c_stream.next_in  = (z_const unsigned char *)hello;
    c_stream.next_out = compr;

    while (c_stream.total_in != len && c_stream.total_out < comprLen) {
        c_stream.avail_in = c_stream.avail_out = 1; /* 强制使用小缓冲区 */
        err = deflate(&c_stream, Z_NO_FLUSH);
        CHECK_ERR(err, "deflate");
    }
    /* 完成流，仍然强制使用小缓冲区 */
    for (;;) {
        c_stream.avail_out = 1;
        err = deflate(&c_stream, Z_FINISH);
        if (err == Z_STREAM_END) break;
        CHECK_ERR(err, "deflate");
    }

    err = deflateEnd(&c_stream);
    CHECK_ERR(err, "deflateEnd");
}

/* ===========================================================================
 * 使用小缓冲区测试 inflate() 函数
 */
void test_inflate(compr, comprLen, uncompr, uncomprLen)
    Byte *compr, *uncompr;
    uLong comprLen, uncomprLen;
{
    int err;
    z_stream d_stream; /* 解压缩流 */

    strcpy((char*)uncompr, "garbage");

    d_stream.zalloc = zalloc;
    d_stream.zfree = zfree;
    d_stream.opaque = (voidpf)0;

    d_stream.next_in  = compr;
    d_stream.avail_in = 0;
    d_stream.next_out = uncompr;

    err = inflateInit(&d_stream);
    CHECK_ERR(err, "inflateInit");
    # 当解压缩的输出总量小于未压缩长度并且输入总量小于压缩长度时执行循环
    while (d_stream.total_out < uncomprLen && d_stream.total_in < comprLen) {
        # 强制使用小缓冲区
        d_stream.avail_in = d_stream.avail_out = 1; /* force small buffers */
        # 解压缩数据流，直到输出缓冲区满或输入缓冲区为空
        err = inflate(&d_stream, Z_NO_FLUSH);
        # 如果解压缩到达流的末尾，则跳出循环
        if (err == Z_STREAM_END) break;
        # 检查解压缩过程中是否出现错误
        CHECK_ERR(err, "inflate");
    }

    # 结束解压缩过程
    err = inflateEnd(&d_stream);
    # 检查解压缩结束时是否出现错误
    CHECK_ERR(err, "inflateEnd");

    # 比较解压缩后的数据和预期的数据是否相同
    if (strcmp((char*)uncompr, hello)) {
        # 如果不相同，则输出错误信息并退出程序
        fprintf(stderr, "bad inflate\n");
        exit(1);
    } else {
        # 如果相同，则输出解压缩后的数据
        printf("inflate(): %s\n", (char *)uncompr);
    }
/* ===========================================================================
 * Test deflate() with large buffers and dynamic change of compression level
 */
// 测试使用大缓冲区和动态改变压缩级别的deflate()
void test_large_deflate(compr, comprLen, uncompr, uncomprLen)
    Byte *compr, *uncompr;
    uLong comprLen, uncomprLen;
{
    z_stream c_stream; /* compression stream */ 
    // 定义压缩流
    int err;
    // 定义错误变量

    c_stream.zalloc = zalloc;
    // 分配内存函数
    c_stream.zfree = zfree;
    // 释放内存函数
    c_stream.opaque = (voidpf)0;
    // 透明指针

    err = deflateInit(&c_stream, Z_BEST_SPEED);
    // 初始化压缩流
    CHECK_ERR(err, "deflateInit");
    // 检查错误

    c_stream.next_out = compr;
    // 设置输出缓冲区
    c_stream.avail_out = (uInt)comprLen;
    // 设置输出缓冲区大小

    /* At this point, uncompr is still mostly zeroes, so it should compress
     * very well:
     */
    // 在这一点上，uncompr仍然大部分是零，所以它应该压缩得很好
    c_stream.next_in = uncompr;
    // 设置输入缓冲区
    c_stream.avail_in = (uInt)uncomprLen;
    // 设置输入缓冲区大小
    err = deflate(&c_stream, Z_NO_FLUSH);
    // 压缩数据
    CHECK_ERR(err, "deflate");
    // 检查错误
    if (c_stream.avail_in != 0) {
        fprintf(stderr, "deflate not greedy\n");
        exit(1);
    }
    // 如果输入缓冲区不为空，则输出错误信息并退出

    /* Feed in already compressed data and switch to no compression: */
    // 输入已经压缩的数据并切换到无压缩
    deflateParams(&c_stream, Z_NO_COMPRESSION, Z_DEFAULT_STRATEGY);
    // 设置压缩参数
    c_stream.next_in = compr;
    // 设置输入缓冲区
    c_stream.avail_in = (uInt)comprLen/2;
    // 设置输入缓冲区大小
    err = deflate(&c_stream, Z_NO_FLUSH);
    // 压缩数据
    CHECK_ERR(err, "deflate");
    // 检查错误

    /* Switch back to compressing mode: */
    // 切换回压缩模式
    deflateParams(&c_stream, Z_BEST_COMPRESSION, Z_FILTERED);
    // 设置压缩参数
    c_stream.next_in = uncompr;
    // 设置输入缓冲区
    c_stream.avail_in = (uInt)uncomprLen;
    // 设置输入缓冲区大小
    err = deflate(&c_stream, Z_NO_FLUSH);
    // 压缩数据
    CHECK_ERR(err, "deflate");
    // 检查错误

    err = deflate(&c_stream, Z_FINISH);
    // 压缩结束
    if (err != Z_STREAM_END) {
        fprintf(stderr, "deflate should report Z_STREAM_END\n");
        exit(1);
    }
    // 如果压缩没有正常结束，则输出错误信息并退出
    err = deflateEnd(&c_stream);
    // 结束压缩流
    CHECK_ERR(err, "deflateEnd");
    // 检查错误
}

/* ===========================================================================
 * Test inflate() with large buffers
 */
// 使用大缓冲区测试inflate()
void test_large_inflate(compr, comprLen, uncompr, uncomprLen)
    Byte *compr, *uncompr;
    uLong comprLen, uncomprLen;
{
    int err;
    // 定义错误变量
    # 创建解压缩流对象
    z_stream d_stream; /* decompression stream */

    # 将字符串"garbage"复制到uncompr中
    strcpy((char*)uncompr, "garbage");

    # 设置解压缩流对象的内存分配函数、释放函数和不透明数据
    d_stream.zalloc = zalloc;
    d_stream.zfree = zfree;
    d_stream.opaque = (voidpf)0;

    # 设置解压缩流对象的输入数据和输入数据长度
    d_stream.next_in  = compr;
    d_stream.avail_in = (uInt)comprLen;

    # 初始化解压缩流对象
    err = inflateInit(&d_stream);
    CHECK_ERR(err, "inflateInit");

    # 循环解压缩数据
    for (;;) {
        # 设置解压缩流对象的输出数据和输出数据长度，丢弃输出
        d_stream.next_out = uncompr;            /* discard the output */
        d_stream.avail_out = (uInt)uncomprLen;
        # 解压缩数据
        err = inflate(&d_stream, Z_NO_FLUSH);
        # 如果解压缩到达流的末尾，则跳出循环
        if (err == Z_STREAM_END) break;
        CHECK_ERR(err, "large inflate");
    }

    # 结束解压缩流对象
    err = inflateEnd(&d_stream);
    CHECK_ERR(err, "inflateEnd");

    # 检查解压缩后的数据长度是否符合预期
    if (d_stream.total_out != 2*uncomprLen + comprLen/2) {
        fprintf(stderr, "bad large inflate: %ld\n", d_stream.total_out);
        exit(1);
    } else {
        printf("large_inflate(): OK\n");
    }
}

/* ===========================================================================
 * 使用全刷新测试deflate()
 */
void test_flush(compr, comprLen)
    Byte *compr;
    uLong *comprLen;
{
    z_stream c_stream; /* 压缩流 */
    int err;
    uInt len = (uInt)strlen(hello)+1;

    c_stream.zalloc = zalloc;  // 分配函数
    c_stream.zfree = zfree;    // 释放函数
    c_stream.opaque = (voidpf)0;

    err = deflateInit(&c_stream, Z_DEFAULT_COMPRESSION);  // 初始化压缩流
    CHECK_ERR(err, "deflateInit");  // 检查错误

    c_stream.next_in  = (z_const unsigned char *)hello;  // 输入数据
    c_stream.next_out = compr;  // 输出数据
    c_stream.avail_in = 3;  // 输入数据的长度
    c_stream.avail_out = (uInt)*comprLen;  // 输出数据的长度
    err = deflate(&c_stream, Z_FULL_FLUSH);  // 压缩数据
    CHECK_ERR(err, "deflate");  // 检查错误

    compr[3]++; /* 强制在第一个压缩块中出现错误 */
    c_stream.avail_in = len - 3;  // 剩余输入数据的长度

    err = deflate(&c_stream, Z_FINISH);  // 完成压缩
    if (err != Z_STREAM_END) {
        CHECK_ERR(err, "deflate");  // 检查错误
    }
    err = deflateEnd(&c_stream);  // 结束压缩
    CHECK_ERR(err, "deflateEnd");  // 检查错误

    *comprLen = c_stream.total_out;  // 压缩后的数据长度
}

/* ===========================================================================
 * 测试inflateSync()
 */
void test_sync(compr, comprLen, uncompr, uncomprLen)
    Byte *compr, *uncompr;
    uLong comprLen, uncomprLen;
{
    int err;
    z_stream d_stream; /* 解压缩流 */

    strcpy((char*)uncompr, "garbage");  // 将"garbage"复制到uncompr中

    d_stream.zalloc = zalloc;  // 分配函数
    d_stream.zfree = zfree;  // 释放函数
    d_stream.opaque = (voidpf)0;

    d_stream.next_in  = compr;  // 输入数据
    d_stream.avail_in = 2; /* 仅读取zlib头部 */

    err = inflateInit(&d_stream);  // 初始化解压缩流
    CHECK_ERR(err, "inflateInit");  // 检查错误

    d_stream.next_out = uncompr;  // 输出数据
    d_stream.avail_out = (uInt)uncomprLen;  // 输出数据的长度

    err = inflate(&d_stream, Z_NO_FLUSH);  // 解压数据
    CHECK_ERR(err, "inflate");  // 检查错误

    d_stream.avail_in = (uInt)comprLen-2;   /* 读取所有压缩数据 */
    err = inflateSync(&d_stream);           /* 但跳过损坏的部分 */
    CHECK_ERR(err, "inflateSync");  // 检查错误

    err = inflate(&d_stream, Z_FINISH);  // 完成解压
    # 如果解压缩操作没有到达流的末尾，报告错误并退出程序
    if (err != Z_STREAM_END) {
        fprintf(stderr, "inflate should report Z_STREAM_END\n");
        exit(1);
    }
    # 结束解压缩操作
    err = inflateEnd(&d_stream);
    CHECK_ERR(err, "inflateEnd");

    # 打印解压缩后的数据
    printf("after inflateSync(): hel%s\n", (char *)uncompr);
}

/* ===========================================================================
 * 使用预设字典测试 deflate()
 */
void test_dict_deflate(compr, comprLen)
    Byte *compr;
    uLong comprLen;
{
    z_stream c_stream; /* 压缩流 */
    int err;

    c_stream.zalloc = zalloc;  // 设置内存分配函数
    c_stream.zfree = zfree;  // 设置内存释放函数
    c_stream.opaque = (voidpf)0;  // 设置私有数据

    err = deflateInit(&c_stream, Z_BEST_COMPRESSION);  // 初始化压缩流
    CHECK_ERR(err, "deflateInit");  // 检查错误

    err = deflateSetDictionary(&c_stream,
                (const Bytef*)dictionary, (int)sizeof(dictionary));  // 设置预设字典
    CHECK_ERR(err, "deflateSetDictionary");  // 检查错误

    dictId = c_stream.adler;  // 获取字典标识
    c_stream.next_out = compr;  // 设置输出缓冲区
    c_stream.avail_out = (uInt)comprLen;  // 设置输出缓冲区大小

    c_stream.next_in = (z_const unsigned char *)hello;  // 设置输入数据
    c_stream.avail_in = (uInt)strlen(hello)+1;  // 设置输入数据大小

    err = deflate(&c_stream, Z_FINISH);  // 压缩数据
    if (err != Z_STREAM_END) {  // 检查压缩是否完成
        fprintf(stderr, "deflate should report Z_STREAM_END\n");
        exit(1);
    }
    err = deflateEnd(&c_stream);  // 结束压缩
    CHECK_ERR(err, "deflateEnd");  // 检查错误
}

/* ===========================================================================
 * 使用预设字典测试 inflate()
 */
void test_dict_inflate(compr, comprLen, uncompr, uncomprLen)
    Byte *compr, *uncompr;
    uLong comprLen, uncomprLen;
{
    int err;
    z_stream d_stream; /* 解压缩流 */

    strcpy((char*)uncompr, "garbage");  // 初始化解压缩数据

    d_stream.zalloc = zalloc;  // 设置内存分配函数
    d_stream.zfree = zfree;  // 设置内存释放函数
    d_stream.opaque = (voidpf)0;  // 设置私有数据

    d_stream.next_in  = compr;  // 设置输入数据
    d_stream.avail_in = (uInt)comprLen;  // 设置输入数据大小

    err = inflateInit(&d_stream);  // 初始化解压缩流
    CHECK_ERR(err, "inflateInit");  // 检查错误

    d_stream.next_out = uncompr;  // 设置输出缓冲区
    d_stream.avail_out = (uInt)uncomprLen;  // 设置输出缓冲区大小
    # 无限循环，直到条件被打破
    for (;;) {
        # 使用无压缩操作来解压缩数据流
        err = inflate(&d_stream, Z_NO_FLUSH);
        # 如果解压缩到达流的末尾，跳出循环
        if (err == Z_STREAM_END) break;
        # 如果需要字典来解压缩
        if (err == Z_NEED_DICT) {
            # 检查字典标识符是否匹配
            if (d_stream.adler != dictId) {
                # 如果不匹配，输出错误信息并退出程序
                fprintf(stderr, "unexpected dictionary");
                exit(1);
            }
            # 设置解压缩所需的字典
            err = inflateSetDictionary(&d_stream, (const Bytef*)dictionary,
                                       (int)sizeof(dictionary));
        }
        # 检查解压缩操作是否出错
        CHECK_ERR(err, "inflate with dict");
    }

    # 结束解压缩操作
    err = inflateEnd(&d_stream);
    CHECK_ERR(err, "inflateEnd");

    # 检查解压缩后的数据是否与指定字符串相同
    if (strcmp((char*)uncompr, hello)) {
        # 如果不相同，输出错误信息并退出程序
        fprintf(stderr, "bad inflate with dict\n");
        exit(1);
    } else {
        # 如果相同，输出解压缩后的数据
        printf("inflate with dictionary: %s\n", (char *)uncompr);
    }
/* ===========================================================================
 * Usage:  example [output.gz  [input.gz]]
 */
// 主函数，接受命令行参数，执行示例程序
int main(argc, argv)
    int argc;
    char *argv[];
{
    Byte *compr, *uncompr;
    uLong comprLen = 10000*sizeof(int); /* don't overflow on MSDOS */
    uLong uncomprLen = comprLen;
    static const char* myVersion = ZLIB_VERSION;

    // 检查链接的 zlib 版本是否兼容
    if (zlibVersion()[0] != myVersion[0]) {
        fprintf(stderr, "incompatible zlib version\n");
        exit(1);

    } else if (strcmp(zlibVersion(), ZLIB_VERSION) != 0) {
        fprintf(stderr, "warning: different zlib version linked: %s\n",
                zlibVersion());
    }

    // 打印 zlib 版本和编译标志
    printf("zlib version %s = 0x%04x, compile flags = 0x%lx\n",
            ZLIB_VERSION, ZLIB_VERNUM, zlibCompileFlags());

    // 分配内存给压缩和解压缩的数据
    compr    = (Byte*)calloc((uInt)comprLen, 1);
    uncompr  = (Byte*)calloc((uInt)uncomprLen, 1);
    /* compr and uncompr are cleared to avoid reading uninitialized
     * data and to ensure that uncompr compresses well.
     */
    // 检查内存分配是否成功
    if (compr == Z_NULL || uncompr == Z_NULL) {
        printf("out of memory\n");
        exit(1);
    }

#ifdef Z_SOLO
    (void)argc;
    (void)argv;
#else
    // 执行压缩测试
    test_compress(compr, comprLen, uncompr, uncomprLen);

    // 执行文件压缩测试
    test_gzio((argc > 1 ? argv[1] : TESTFILE),
              uncompr, uncomprLen);
#endif

    // 执行压缩测试
    test_deflate(compr, comprLen);
    // 执行解压缩测试
    test_inflate(compr, comprLen, uncompr, uncomprLen);

    // 执行大数据压缩测试
    test_large_deflate(compr, comprLen, uncompr, uncomprLen);
    // 执行大数据解压缩测试
    test_large_inflate(compr, comprLen, uncompr, uncomprLen);

    // 执行压缩数据刷新测试
    test_flush(compr, &comprLen);
    // 执行数据同步测试
    test_sync(compr, comprLen, uncompr, uncomprLen);
    comprLen = uncomprLen;

    // 执行字典压缩测试
    test_dict_deflate(compr, comprLen);
    // 执行字典解压缩测试
    test_dict_inflate(compr, comprLen, uncompr, uncomprLen);

    // 释放内存
    free(compr);
    free(uncompr);

    return 0;
}
```