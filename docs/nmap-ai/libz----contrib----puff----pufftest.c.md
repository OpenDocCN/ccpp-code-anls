# `nmap\libz\contrib\puff\pufftest.c`

```
/*
 * pufftest.c
 * 版权所有 (C) 2002-2013 Mark Adler
 * 有关分发和使用条件，请参阅 puff.h 中的版权声明
 * 版本 2.3, 2013年1月21日
 */

/* puff() 的使用示例。

   用法: puff [-w] [-f] [-nnn] file
          ... | puff [-w] [-f] [-nnn]

   其中 file 是包含压缩数据的输入文件，nnn 是在解压缩之前要跳过的输入字节数
   （例如，跳过 zlib 或 gzip 头部），-w 用于将解压缩的数据写入标准输出。
   -f 用于覆盖测试，导致 pufftest 因输出空间不足而失败（-f 像 -w 一样进行写入，
   因此不需要 -w）。 */

#include <stdio.h>
#include <stdlib.h>
#include "puff.h"

#if defined(MSDOS) || defined(OS2) || defined(WIN32) || defined(__CYGWIN__)
#  include <fcntl.h>
#  include <io.h>
#  define SET_BINARY_MODE(file) setmode(fileno(file), O_BINARY)
#else
#  define SET_BINARY_MODE(file)
#endif

#define local static

/* 返回大小乘以大约 2 的立方根，保持结果为 1、3 或 5 的 2 的幂次方倍数 --
   结果始终大于 size，直到结果为无符号长整型的最大值为止。这对于使重新分配
   小于实际数据的约 33% 很有用。 */
local size_t bythirds(size_t size)
{
    int n;
    size_t m;

    m = size;
    for (n = 0; m; n++)
        m >>= 1;
    if (n < 3)
        return size + 1;
    n -= 3;
    m = size >> n;
    m += m == 6 ? 2 : 1;
    m <<= n;
    return m > size ? m : (size_t)(-1);
}

/* 读取输入文件 *name，如果 name 为 NULL，则读取标准输入，读入分配的内存中。
   重新分配为更大的缓冲区，直到整个文件都被读入。返回指向分配数据的指针，
   如果存在内存分配失败，则返回 NULL。*len 是从输入文件中读取的数据字节数
   （即使 load() 返回 NULL）。如果输入文件为空或无法打开或读取，则 *len 为零。 */
local void *load(const char *name, size_t *len)
{
    size_t size;
    // 声明指针变量buf和swap，以及文件指针变量in
    void *buf, *swap;
    FILE *in;

    // 初始化len为0，分配4096字节的内存给buf
    *len = 0;
    buf = malloc(size = 4096);
    // 如果内存分配失败，则返回NULL
    if (buf == NULL)
        return NULL;
    // 如果name为NULL，则打开标准输入流，否则以二进制只读方式打开文件
    in = name == NULL ? stdin : fopen(name, "rb");
    // 如果文件打开成功
    if (in != NULL) {
        // 循环读取文件内容
        for (;;) {
            // 从文件中读取数据到buf中，更新len的值
            *len += fread((char *)buf + *len, 1, size - *len, in);
            // 如果读取的数据小于size，则跳出循环
            if (*len < size) break;
            // 根据bythirds函数调整size的大小
            size = bythirds(size);
            // 如果size等于len，或者重新分配内存失败，则释放buf并跳出循环
            if (size == *len || (swap = realloc(buf, size)) == NULL) {
                free(buf);
                buf = NULL;
                break;
            }
            // 更新buf为重新分配的内存
            buf = swap;
        }
        // 关闭文件
        fclose(in);
    }
    // 返回buf
    return buf;
int main(int argc, char **argv)
{
    int ret, put = 0, fail = 0;  // 定义整型变量ret, put, fail，并初始化put和fail为0
    unsigned skip = 0;  // 定义无符号整型变量skip，并初始化为0
    char *arg, *name = NULL;  // 定义字符指针变量arg和name，并将name初始化为NULL
    unsigned char *source = NULL, *dest;  // 定义无符号字符指针变量source和dest，并将source初始化为NULL
    size_t len = 0;  // 定义size_t类型变量len，并初始化为0
    unsigned long sourcelen, destlen;  // 定义无符号长整型变量sourcelen和destlen

    /* process arguments */
    while (arg = *++argv, --argc)  // 处理命令行参数
        if (arg[0] == '-') {  // 如果参数以"-"开头
            if (arg[1] == 'w' && arg[2] == 0)  // 如果参数为"-w"
                put = 1;  // 将put设置为1
            else if (arg[1] == 'f' && arg[2] == 0)  // 如果参数为"-f"
                fail = 1, put = 1;  // 将fail设置为1，put设置为1
            else if (arg[1] >= '0' && arg[1] <= '9')  // 如果参数为数字
                skip = (unsigned)atoi(arg + 1);  // 将参数转换为无符号整型并赋值给skip
            else {
                fprintf(stderr, "invalid option %s\n", arg);  // 输出错误信息到标准错误流
                return 3;  // 返回3
            }
        }
        else if (name != NULL) {  // 如果name不为空
            fprintf(stderr, "only one file name allowed\n");  // 输出错误信息到标准错误流
            return 3;  // 返回3
        }
        else
            name = arg;  // 将参数赋值给name
    source = load(name, &len);  // 调用load函数加载文件内容到source，并将文件长度存储到len
    if (source == NULL) {  // 如果source为空
        fprintf(stderr, "memory allocation failure\n");  // 输出错误信息到标准错误流
        return 4;  // 返回4
    }
    if (len == 0) {  // 如果文件长度为0
        fprintf(stderr, "could not read %s, or it was empty\n",
                name == NULL ? "<stdin>" : name);  // 输出错误信息到标准错误流
        free(source);  // 释放source指向的内存
        return 3;  // 返回3
    }
    if (skip >= len) {  // 如果skip大于等于文件长度
        fprintf(stderr, "skip request of %d leaves no input\n", skip);  // 输出错误信息到标准错误流
        free(source);  // 释放source指向的内存
        return 3;  // 返回3
    }

    /* test inflate data with offset skip */
    len -= skip;  // 文件长度减去skip
    sourcelen = (unsigned long)len;  // 将len转换为无符号长整型并赋值给sourcelen
    ret = puff(NIL, &destlen, source + skip, &sourcelen);  // 调用puff函数对source进行解压缩
    if (ret)  // 如果ret不为0
        fprintf(stderr, "puff() failed with return code %d\n", ret);  // 输出错误信息到标准错误流
    else {
        fprintf(stderr, "puff() succeeded uncompressing %lu bytes\n", destlen);  // 输出解压缩成功信息到标准错误流
        if (sourcelen < len) fprintf(stderr, "%lu compressed bytes unused\n",
                                     len - sourcelen);  // 如果解压缩后的长度小于原始长度，输出未使用的压缩字节数到标准错误流
    }

    /* if requested, inflate again and write decompressed data to stdout */
}
    # 如果put为真且ret为0，则执行以下操作
    if (put && ret == 0) {
        # 如果fail为真，则将destlen右移一位
        if (fail)
            destlen >>= 1;
        # 分配destlen大小的内存空间给dest
        dest = malloc(destlen);
        # 如果dest为空，则输出内存分配失败的错误信息，释放source内存，返回4
        if (dest == NULL) {
            fprintf(stderr, "memory allocation failure\n");
            free(source);
            return 4;
        }
        # 解压source中skip偏移位置开始的数据，存入dest中，解压后的数据长度存入destlen
        puff(dest, &destlen, source + skip, &sourcelen);
        # 设置标准输出为二进制模式
        SET_BINARY_MODE(stdout);
        # 将dest中的数据写入标准输出，写入长度为destlen
        fwrite(dest, 1, destlen, stdout);
        # 释放dest内存
        free(dest);
    }

    /* 清理 */
    # 释放source内存
    free(source);
    # 返回ret
    return ret;
# 闭合前面的函数定义
```