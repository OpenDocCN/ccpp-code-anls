# `nmap\libz\test\infcover.c`

```
/* infcover.c -- test zlib's inflate routines with full code coverage
 * Copyright (C) 2011, 2016 Mark Adler
 * For conditions of distribution and use, see copyright notice in zlib.h
 */

/* to use, do: ./configure --cover && make cover */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <assert.h>
#include "zlib.h"

/* get definition of internal structure so we can mess with it (see pull()),
   and so we can call inflate_trees() (see cover5()) */
#define ZLIB_INTERNAL
#include "inftrees.h"
#include "inflate.h"

#define local static

/* -- memory tracking routines -- */
/*
   这些内存跟踪例程提供给 zlib，并跟踪所有 zlib 的分配和释放，检查 LIFO 操作，保持总请求字节的当前和最大值，可选择设置总内存的限制，并在完成后检查内存泄漏。

   它们的使用如下：

   z_stream strm;
   mem_setup(&strm)         初始化内存跟踪并设置 zalloc、zfree 和 opaque 成员，以便在 strm 上对所有 zlib 操作使用内存跟踪
   mem_limit(&strm, limit)  设置对总字节请求的限制 -- 超过此限制的请求将导致分配失败（返回 NULL）-- 将限制设置为零表示没有限制，这是 mem_setup() 后的默认值
   mem_used(&strm, "msg")   打印到 stderr "msg" 和已使用的总字节数
   mem_high(&strm, "msg")   打印到 stderr "msg" 和最大值
   mem_done(&strm, "msg")   结束内存跟踪，释放所有用于跟踪的分配以及泄漏的 zlib 块（如果有）。如果出现异常情况，比如泄漏的块、非 FIFO 释放或释放未分配的地址，则会向 stderr 打印 "msg" 和有关问题的信息。如果一切正常，则不会打印任何内容。mem_done 会将 strm 成员重置为 Z_NULL，以便在下一次使用 strm 进行 zlib 初始化时使用默认的内存分配例程。
 */

/* 这些项目被串在一起形成一个链表，每个分配一个 */
struct mem_item {
    // 指向分配内存的指针
    void *ptr;                  
    // 请求分配的内存大小
    size_t size;                
    // 指向列表中下一项的指针，如果没有则为NULL
    struct mem_item *next;      
};

/* 这个结构位于链表的根部，用于跟踪统计信息 */
struct mem_zone {
    struct mem_item *first;     /* 指向列表中的第一项，或者为 NULL */
    size_t total, highwater;    /* 总分配量，以及最大总分配量 */
    size_t limit;               /* 内存分配限制，如果没有限制则为 0 */
    int notlifo, rogue;         /* 非LIFO释放和异常释放的计数 */
};

/* 传递给 zlib 的内存分配例程 */
local void *mem_alloc(void *mem, unsigned count, unsigned size)
{
    void *ptr;
    struct mem_item *item;
    struct mem_zone *zone = mem;
    size_t len = count * (size_t)size;

    /* 引发分配失败 */
    if (zone == NULL || (zone->limit && zone->total + len > zone->limit))
        return NULL;

    /* 使用标准库进行分配，用非零值填充内存以确保代码不依赖于零值 */
    ptr = malloc(len);
    if (ptr == NULL)
        return NULL;
    memset(ptr, 0xa5, len);

    /* 为列表创建一个新项 */
    item = malloc(sizeof(struct mem_item));
    if (item == NULL) {
        free(ptr);
        return NULL;
    }
    item->ptr = ptr;
    item->size = len;

    /* 将项插入到列表的开头 */
    item->next = zone->first;
    zone->first = item;

    /* 更新统计信息 */
    zone->total += item->size;
    if (zone->total > zone->highwater)
        zone->highwater = zone->total;

    /* 返回分配的内存 */
    return ptr;
}

/* 传递给 zlib 的内存释放例程 */
local void mem_free(void *mem, void *ptr)
{
    struct mem_item *item, *next;
    struct mem_zone *zone = mem;

    /* 如果没有区域，直接释放 */
    if (zone == NULL) {
        free(ptr);
        return;
    }

    /* 将 next 指向与 ptr 匹配的项，如果找不到则为 NULL -- 如果找到则从链表中移除该项 */
    next = zone->first;
    # 如果下一个节点存在
    if (next) {
        # 如果下一个节点的指针等于当前指针，将其从链表中移除
        if (next->ptr == ptr)
            zone->first = next->next;   /* first one is it, remove from list */
        else {
            # 否则，搜索链表
            do {                        /* search the linked list */
                item = next;
                next = item->next;
            } while (next != NULL && next->ptr != ptr);
            # 如果找到，从链表中移除
            if (next) {                 /* if found, remove from linked list */
                item->next = next->next;
                zone->notlifo++;        /* not a LIFO free */
            }

        }
    }

    # 如果找到，更新统计信息并释放该节点
    if (next) {
        zone->total -= next->size;
        free(next);
    }

    # 如果未找到，更新异常节点计数
    else
        zone->rogue++;

    # 无论如何，使用标准库函数进行释放
    free(ptr);
# 为监控设置一个受控内存分配空间，将流参数设置为受控例程，opaque指向该空间
local void mem_setup(z_stream *strm)
{
    // 分配内存空间用于存储监控信息
    struct mem_zone *zone;
    zone = malloc(sizeof(struct mem_zone));
    // 确保内存分配成功
    assert(zone != NULL);
    zone->first = NULL;
    zone->total = 0;
    zone->highwater = 0;
    zone->limit = 0;
    zone->notlifo = 0;
    zone->rogue = 0;
    // 将空间指针赋值给流的opaque属性
    strm->opaque = zone;
    // 设置内存分配函数
    strm->zalloc = mem_alloc;
    // 设置内存释放函数
    strm->zfree = mem_free;
}

# 设置总内存分配限制，或者设置为0以移除限制
local void mem_limit(z_stream *strm, size_t limit)
{
    // 获取流的opaque属性，即内存分配空间
    struct mem_zone *zone = strm->opaque;
    // 设置内存分配限制
    zone->limit = limit;
}

# 显示当前总请求分配的内存大小（以字节为单位）
local void mem_used(z_stream *strm, char *prefix)
{
    // 获取流的opaque属性，即内存分配空间
    struct mem_zone *zone = strm->opaque;
    // 打印当前总分配的内存大小
    fprintf(stderr, "%s: %lu allocated\n", prefix, zone->total);
}

# 显示内存高水位线的分配大小（以字节为单位）
local void mem_high(z_stream *strm, char *prefix)
{
    // 获取流的opaque属性，即内存分配空间
    struct mem_zone *zone = strm->opaque;
    // 打印内存高水位线的分配大小
    fprintf(stderr, "%s: %lu high water mark\n", prefix, zone->highwater);
}

# 释放内存分配空间 -- 如果有任何意外情况，请通知
local void mem_done(z_stream *strm, char *prefix)
{
    int count = 0;
    struct mem_item *item, *next;
    // 获取流的opaque属性，即内存分配空间
    struct mem_zone *zone = strm->opaque;

    /* 显示高水位线 */
    mem_high(strm, prefix);

    /* 释放剩余的分配和项目结构，如果有的话 */
    item = zone->first;
    while (item != NULL) {
        free(item->ptr);
        next = item->next;
        free(item);
        item = next;
        count++;
    }

    /* 对任何意外情况发出警报 */
    if (count || zone->total)
        fprintf(stderr, "** %s: %lu bytes in %d blocks not freed\n",
                prefix, zone->total, count);
    if (zone->notlifo)
        fprintf(stderr, "** %s: %d frees not LIFO\n", prefix, zone->notlifo);
}
    # 如果 zone->rogue 存在，则向标准错误流输出未识别的释放数量
    if (zone->rogue)
        fprintf(stderr, "** %s: %d frees not recognized\n",
                prefix, zone->rogue);

    # 释放 zone 指针指向的内存
    free(zone);
    # 将 strm->opaque 设置为 Z_NULL
    strm->opaque = Z_NULL;
    # 将 strm->zalloc 设置为 Z_NULL
    strm->zalloc = Z_NULL;
    # 将 strm->zfree 设置为 Z_NULL
    strm->zfree = Z_NULL;
/* -- inflate test routines -- */

/* 解码十六进制字符串，将 *len 设置为长度，in[] 设置为字节。这种解码方式很宽松，因为十六进制数字可以相邻，这种情况下两个相邻的数字写入一个字节。或者它们可以被任何非十六进制字符分隔，其中分隔符被忽略，除非一个单个的十六进制数字后面跟着一个分隔符，这种情况下该单个数字写入一个字节。返回的数据是分配的，最终必须被释放。如果内存不足，则返回 NULL。如果不需要长度，则 len 可以为 NULL。 */
local unsigned char *h2b(const char *hex, unsigned *len)
{
    unsigned char *in, *re;
    unsigned next, val;

    in = malloc((strlen(hex) + 1) >> 1);  // 分配内存，用于存储解码后的字节
    if (in == NULL)
        return NULL;  // 如果内存分配失败，则返回 NULL
    next = 0;  // 初始化下一个位置
    val = 1;  // 初始化值
    do {
        if (*hex >= '0' && *hex <= '9')
            val = (val << 4) + *hex - '0';  // 如果是数字，则进行解码
        else if (*hex >= 'A' && *hex <= 'F')
            val = (val << 4) + *hex - 'A' + 10;  // 如果是大写字母，则进行解码
        else if (*hex >= 'a' && *hex <= 'f')
            val = (val << 4) + *hex - 'a' + 10;  // 如果是小写字母，则进行解码
        else if (val != 1 && val < 32)  // 一个数字后面跟着分隔符
            val += 240;  // 让它看起来像两个数字
        if (val > 255) {  // 有两个数字
            in[next++] = val & 0xff;  // 保存解码后的字节
            val = 1;  // 重新开始
        }
    } while (*hex++);  // 通过循环处理以 null 结尾的字符串
    if (len != NULL)
        *len = next;  // 如果需要长度，则设置长度
    re = realloc(in, next);  // 重新分配内存
    return re == NULL ? in : re;  // 如果重新分配失败，则返回原始内存，否则返回重新分配后的内存
}
# 通用的 inflate() 运行函数，其中 hex 是十六进制输入数据，what 是要包含在错误消息中的文本，step 是每次调用 inflate() 时要提供的输入数据量，或者为零以提供全部数据，win 是传递给 inflateInit2() 的窗口位参数，len 是输出缓冲区的大小，err 是预期从第一个 inflate() 调用返回的错误代码（第二个 inflate() 调用预期返回 Z_STREAM_END）。如果 win 为 47，则使用 inflateGetHeader() 收集头部信息。如果 zlib 流正在寻找字典，则提供一个空字典。inflate() 运行直到消耗完所有输入数据。
local void inf(char *hex, char *what, unsigned step, int win, unsigned len, int err)
{
    int ret;
    unsigned have;
    unsigned char *in, *out;
    z_stream strm, copy;
    gz_header head;

    # 设置内存
    mem_setup(&strm);
    strm.avail_in = 0;
    strm.next_in = Z_NULL;
    # 初始化 inflate
    ret = inflateInit2(&strm, win);
    if (ret != Z_OK) {
        # 完成内存操作
        mem_done(&strm, what);
        return;
    }
    # 分配输出缓冲区
    out = malloc(len);                          assert(out != NULL);
    if (win == 47) {
        # 设置头部信息
        head.extra = out;
        head.extra_max = len;
        head.name = out;
        head.name_max = len;
        head.comment = out;
        head.comm_max = len;
        # 获取头部信息
        ret = inflateGetHeader(&strm, &head);   assert(ret == Z_OK);
    }
    # 将十六进制转换为二进制
    in = h2b(hex, &have);                       assert(in != NULL);
    if (step == 0 || step > have)
        step = have;
    strm.avail_in = step;
    have -= step;
    strm.next_in = in;
}
    // 使用 do-while 循环执行以下操作，直到满足条件才退出循环
    do {
        // 设置输出缓冲区的大小
        strm.avail_out = len;
        // 设置输出缓冲区的起始位置
        strm.next_out = out;
        // 解压缩数据
        ret = inflate(&strm, Z_NO_FLUSH);       assert(err == 9 || ret == err);
        // 如果返回值不是 Z_OK、Z_BUF_ERROR 或 Z_NEED_DICT，则跳出循环
        if (ret != Z_OK && ret != Z_BUF_ERROR && ret != Z_NEED_DICT)
            break;
        // 如果需要字典，则执行以下操作
        if (ret == Z_NEED_DICT) {
            // 设置输入字典
            ret = inflateSetDictionary(&strm, in, 1);
                                                assert(ret == Z_DATA_ERROR);
            // 限制内存使用
            mem_limit(&strm, 1);
            // 设置输出字典
            ret = inflateSetDictionary(&strm, out, 0);
                                                assert(ret == Z_MEM_ERROR);
            // 限制内存使用
            mem_limit(&strm, 0);
            // 设置解压缩模式为字典模式
            ((struct inflate_state *)strm.state)->mode = DICT;
            // 设置输出字典
            ret = inflateSetDictionary(&strm, out, 0);
                                                assert(ret == Z_OK);
            // 继续解压缩数据
            ret = inflate(&strm, Z_NO_FLUSH);   assert(ret == Z_BUF_ERROR);
        }
        // 复制解压缩状态
        ret = inflateCopy(&copy, &strm);        assert(ret == Z_OK);
        // 结束解压缩
        ret = inflateEnd(&copy);                assert(ret == Z_OK);
        // 设置 err 为 9，下次循环不关心
        err = 9;                        /* don't care next time around */
        // 更新已处理的输入数据大小
        have += strm.avail_in;
        // 设置输入缓冲区的大小
        strm.avail_in = step > have ? have : step;
        // 更新已处理的输入数据大小
        have -= strm.avail_in;
    } while (strm.avail_in);
    // 释放输入缓冲区的内存
    free(in);
    // 释放输出缓冲区的内存
    free(out);
    // 重置解压缩状态
    ret = inflateReset2(&strm, -8);             assert(ret == Z_OK);
    // 结束解压缩
    ret = inflateEnd(&strm);                    assert(ret == Z_OK);
    // 释放内存
    mem_done(&strm, what);
# 定义一个函数，用于覆盖 inflate.c 中 inflate() 函数之前的所有代码
local void cover_support(void)
{
    int ret;  # 声明一个整型变量 ret
    z_stream strm;  # 声明一个 z_stream 结构体变量 strm

    mem_setup(&strm);  # 调用 mem_setup() 函数对 strm 进行内存设置
    strm.avail_in = 0;  # 设置 strm 的 avail_in 属性为 0
    strm.next_in = Z_NULL;  # 设置 strm 的 next_in 属性为 Z_NULL
    ret = inflateInit(&strm);  # 调用 inflateInit() 函数对 strm 进行初始化，并将返回值赋给 ret
    assert(ret == Z_OK);  # 使用 assert() 函数判断 ret 是否等于 Z_OK
    mem_used(&strm, "inflate init");  # 调用 mem_used() 函数对 strm 进行内存使用记录
    ret = inflatePrime(&strm, 5, 31);  # 调用 inflatePrime() 函数对 strm 进行预处理，并将返回值赋给 ret
    assert(ret == Z_OK);  # 使用 assert() 函数判断 ret 是否等于 Z_OK
    ret = inflatePrime(&strm, -1, 0);  # 再次调用 inflatePrime() 函数对 strm 进行预处理，并将返回值赋给 ret
    assert(ret == Z_OK);  # 使用 assert() 函数判断 ret 是否等于 Z_OK
    ret = inflateSetDictionary(&strm, Z_NULL, 0);  # 调用 inflateSetDictionary() 函数对 strm 设置字典，并将返回值赋给 ret
    assert(ret == Z_STREAM_ERROR);  # 使用 assert() 函数判断 ret 是否等于 Z_STREAM_ERROR
    ret = inflateEnd(&strm);  # 调用 inflateEnd() 函数结束对 strm 的处理，并将返回值赋给 ret
    assert(ret == Z_OK);  # 使用 assert() 函数判断 ret 是否等于 Z_OK
    mem_done(&strm, "prime");  # 调用 mem_done() 函数对 strm 进行内存处理记录

    # 调用 inf() 函数进行特定情况的测试
    inf("63 0", "force window allocation", 0, -15, 1, Z_OK);
    inf("63 18 5", "force window replacement", 0, -8, 259, Z_OK);
    inf("63 18 68 30 d0 0 0", "force split window update", 4, -8, 259, Z_OK);
    inf("3 0", "use fixed blocks", 0, -15, 1, Z_STREAM_END);
    inf("", "bad window size", 0, 1, 0, Z_STREAM_ERROR);

    mem_setup(&strm);  # 再次调用 mem_setup() 函数对 strm 进行内存设置
    strm.avail_in = 0;  # 设置 strm 的 avail_in 属性为 0
    strm.next_in = Z_NULL;  # 设置 strm 的 next_in 属性为 Z_NULL
    ret = inflateInit_(&strm, ZLIB_VERSION - 1, (int)sizeof(z_stream));  # 调用 inflateInit_() 函数对 strm 进行初始化，并将返回值赋给 ret
    assert(ret == Z_VERSION_ERROR);  # 使用 assert() 函数判断 ret 是否等于 Z_VERSION_ERROR
    mem_done(&strm, "wrong version");  # 调用 mem_done() 函数对 strm 进行内存处理记录

    strm.avail_in = 0;  # 设置 strm 的 avail_in 属性为 0
    strm.next_in = Z_NULL;  # 设置 strm 的 next_in 属性为 Z_NULL
    ret = inflateInit(&strm);  # 再次调用 inflateInit() 函数对 strm 进行初始化，并将返回值赋给 ret
    assert(ret == Z_OK);  # 使用 assert() 函数判断 ret 是否等于 Z_OK
    ret = inflateEnd(&strm);  # 调用 inflateEnd() 函数结束对 strm 的处理，并将返回值赋给 ret
    assert(ret == Z_OK);  # 使用 assert() 函数判断 ret 是否等于 Z_OK
    fputs("inflate built-in memory routines\n", stderr);  # 在标准错误流中输出一条消息
}

# 定义一个函数，用于覆盖所有 inflate() 头部和尾部情况以及在 inflate() 之后的所有代码
local void cover_wrap(void)
{
    int ret;  # 声明一个整型变量 ret
    z_stream strm, copy;  # 声明两个 z_stream 结构体变量 strm 和 copy
    unsigned char dict[257];  # 声明一个长度为 257 的无符号字符数组 dict

    ret = inflate(Z_NULL, 0);  # 调用 inflate() 函数对 Z_NULL 进行解压缩，并将返回值赋给 ret
    assert(ret == Z_STREAM_ERROR);  # 使用 assert() 函数判断 ret 是否等于 Z_STREAM_ERROR
    ret = inflateEnd(Z_NULL);  # 调用 inflateEnd() 函数结束对 Z_NULL 的处理，并将返回值赋给 ret
    assert(ret == Z_STREAM_ERROR);  # 使用 assert() 函数判断 ret 是否等于 Z_STREAM_ERROR
    ret = inflateCopy(Z_NULL, Z_NULL);  # 调用 inflateCopy() 函数对 Z_NULL 进行复制，并将返回值赋给 ret
    assert(ret == Z_STREAM_ERROR);  # 使用 assert() 函数判断 ret 是否等于 Z_STREAM_ERROR
    fputs("inflate bad parameters\n", stderr);  # 在标准错误流中输出一条消息
}
    # 调用inf函数，传入参数为16进制字符串、错误信息、以及其他参数，检查gzip方法是否正确
    inf("1f 8b 0 0", "bad gzip method", 0, 31, 0, Z_DATA_ERROR);
    # 调用inf函数，传入参数为16进制字符串、错误信息、以及其他参数，检查gzip标志位是否正确
    inf("1f 8b 8 80", "bad gzip flags", 0, 31, 0, Z_DATA_ERROR);
    # 调用inf函数，传入参数为16进制字符串、错误信息、以及其他参数，检查zlib方法是否正确
    inf("77 85", "bad zlib method", 0, 15, 0, Z_DATA_ERROR);
    # 调用inf函数，传入参数为16进制字符串、错误信息、以及其他参数，设置头部的窗口大小
    inf("8 99", "set window size from header", 0, 0, 0, Z_OK);
    # 调用inf函数，传入参数为16进制字符串、错误信息、以及其他参数，检查zlib窗口大小是否正确
    inf("78 9c", "bad zlib window size", 0, 8, 0, Z_DATA_ERROR);
    # 调用inf函数，传入参数为16进制字符串、错误信息、以及其他参数，检查adler32校验
    inf("78 9c 63 0 0 0 1 0 1", "check adler32", 0, 15, 1, Z_STREAM_END);
    # 调用inf函数，传入参数为16进制字符串、错误信息、以及其他参数，检查头部crc是否正确
    inf("1f 8b 8 1e 0 0 0 0 0 0 1 0 0 0 0 0 0", "bad header crc", 0, 47, 1, Z_DATA_ERROR);
    # 调用inf函数，传入参数为16进制字符串、错误信息、以及其他参数，检查gzip长度是否正确
    inf("1f 8b 8 2 0 0 0 0 0 0 1d 26 3 0 0 0 0 0 0 0 0 0", "check gzip length", 0, 47, 0, Z_STREAM_END);
    # 调用inf函数，传入参数为16进制字符串、错误信息、以及其他参数，检查zlib头部校验是否正确
    inf("78 90", "bad zlib header check", 0, 47, 0, Z_DATA_ERROR);
    # 调用inf函数，传入参数为16进制字符串、错误信息、以及其他参数，需要字典
    inf("8 b8 0 0 0 1", "need dictionary", 0, 8, 0, Z_NEED_DICT);
    # 调用inf函数，传入参数为16进制字符串、错误信息、以及其他参数，计算adler32校验
    inf("78 9c 63 0", "compute adler32", 0, 15, 1, Z_OK);

    # 初始化strm结构体
    mem_setup(&strm);
    # 设置输入数据的长度为0
    strm.avail_in = 0;
    # 设置输入数据的指针为空
    strm.next_in = Z_NULL;
    # 初始化inflate结构体，使用默认的窗口大小
    ret = inflateInit2(&strm, -8);
    # 设置输入数据的长度为2
    strm.avail_in = 2;
    # 设置输入数据的指针为16进制字符串"\x63"
    strm.next_in = (void *)"\x63";
    # 设置输出数据的长度为1
    strm.avail_out = 1;
    # 设置输出数据的指针为ret
    strm.next_out = (void *)&ret;
    # 设置内存限制为1
    mem_limit(&strm, 1);
    # 解压缩数据，检查返回值是否为Z_MEM_ERROR
    ret = inflate(&strm, Z_NO_FLUSH);           assert(ret == Z_MEM_ERROR);
    # 解压缩数据，检查返回值是否为Z_MEM_ERROR
    ret = inflate(&strm, Z_NO_FLUSH);           assert(ret == Z_MEM_ERROR);
    # 取消内存限制
    mem_limit(&strm, 0);
    # 将dict数组的前257个字节设置为0
    memset(dict, 0, 257);
    # 设置字典，长度为257
    ret = inflateSetDictionary(&strm, dict, 257);
                                                assert(ret == Z_OK);
    # 设置内存限制，大小为(inflate_state结构体大小的两倍) + 256
    mem_limit(&strm, (sizeof(struct inflate_state) << 1) + 256);
    # 预留空间，长度为16
    ret = inflatePrime(&strm, 16, 0);           assert(ret == Z_OK);
    # 设置输入数据的长度为2
    strm.avail_in = 2;
    # 设置输入数据的指针为16进制字符串"\x80"
    strm.next_in = (void *)"\x80";
    # 同步查找压缩数据的开始位置
    ret = inflateSync(&strm);                   assert(ret == Z_DATA_ERROR);
    # 解压缩数据，检查返回值是否为Z_STREAM_ERROR
    ret = inflate(&strm, Z_NO_FLUSH);           assert(ret == Z_STREAM_ERROR);
    # 设置输入数据的长度为4
    strm.avail_in = 4;
    # 设置输入数据的指针为16进制字符串"\0\0\xff\xff"
    strm.next_in = (void *)"\0\0\xff\xff";
    # 同步查找压缩数据的开始位置
    ret = inflateSync(&strm);                   assert(ret == Z_OK);
    # 检查是否为同步点
    (void)inflateSyncPoint(&strm);
    # 复制inflate结构体
    ret = inflateCopy(&copy, &strm);            assert(ret == Z_MEM_ERROR);
    # 取消内存限制
    mem_limit(&strm, 0);
    # 使用inflateUndermine函数解压缩数据流，设置参数1表示强制产生数据错误，返回值应该是Z_DATA_ERROR
    ret = inflateUndermine(&strm, 1);           assert(ret == Z_DATA_ERROR);
    # 使用inflateMark函数标记当前解压缩位置
    (void)inflateMark(&strm);
    # 结束解压缩过程，返回值应该是Z_OK
    ret = inflateEnd(&strm);                    assert(ret == Z_OK);
    # 手动触发内存错误，释放解压缩过程中使用的内存
    mem_done(&strm, "miscellaneous, force memory errors");
}

/* input and output functions for inflateBack() */
# 定义一个函数，用于从输入中获取数据
local unsigned pull(void *desc, unsigned char **buf)
{
    static unsigned int next = 0;
    static unsigned char dat[] = {0x63, 0, 2, 0};
    struct inflate_state *state;

    if (desc == Z_NULL) {
        next = 0;
        return 0;   /* no input (already provided at next_in) */
    }
    state = (void *)((z_stream *)desc)->state;
    if (state != Z_NULL)
        state->mode = SYNC;     /* force an otherwise impossible situation */
    return next < sizeof(dat) ? (*buf = dat + next++, 1) : 0;
}

# 定义一个函数，用于向输出中推送数据
local int push(void *desc, unsigned char *buf, unsigned len)
{
    buf += len;
    return desc != Z_NULL;      /* force error if desc not null */
}

/* cover inflateBack() up to common deflate data cases and after those */
# 定义一个函数，用于测试inflateBack()函数的各种情况
local void cover_back(void)
{
    int ret;
    z_stream strm;
    unsigned char win[32768];

    ret = inflateBackInit_(Z_NULL, 0, win, 0, 0);
                                                assert(ret == Z_VERSION_ERROR);
    ret = inflateBackInit(Z_NULL, 0, win);      assert(ret == Z_STREAM_ERROR);
    ret = inflateBack(Z_NULL, Z_NULL, Z_NULL, Z_NULL, Z_NULL);
                                                assert(ret == Z_STREAM_ERROR);
    ret = inflateBackEnd(Z_NULL);               assert(ret == Z_STREAM_ERROR);
    fputs("inflateBack bad parameters\n", stderr);

    mem_setup(&strm);
    ret = inflateBackInit(&strm, 15, win);      assert(ret == Z_OK);
    strm.avail_in = 2;
    strm.next_in = (void *)"\x03";
    ret = inflateBack(&strm, pull, Z_NULL, push, Z_NULL);
                                                assert(ret == Z_STREAM_END);
        /* force output error */
    strm.avail_in = 3;
    strm.next_in = (void *)"\x63\x00";
    ret = inflateBack(&strm, pull, Z_NULL, push, &strm);
                                                assert(ret == Z_BUF_ERROR);
        /* force mode error by mucking with state */
}
    # 使用inflateBack函数解压缩数据，使用pull函数从输入缓冲区中获取数据，使用push函数将解压缩后的数据写入输出缓冲区，Z_NULL表示不使用回调函数
    ret = inflateBack(&strm, pull, &strm, push, Z_NULL);
    # 断言ret等于Z_STREAM_ERROR，即判断inflateBack函数是否返回Z_STREAM_ERROR
    assert(ret == Z_STREAM_ERROR);
    
    # 使用inflateBackEnd函数结束解压缩过程
    ret = inflateBackEnd(&strm);
    # 断言ret等于Z_OK，即判断inflateBackEnd函数是否返回Z_OK
    assert(ret == Z_OK);
    
    # 使用mem_done函数释放inflateBack函数使用的内存
    mem_done(&strm, "inflateBack bad state");

    # 使用inflateBackInit函数初始化解压缩过程，参数15表示窗口大小，win表示窗口数据
    ret = inflateBackInit(&strm, 15, win);
    # 断言ret等于Z_OK，即判断inflateBackInit函数是否返回Z_OK
    assert(ret == Z_OK);
    
    # 使用inflateBackEnd函数结束解压缩过程
    ret = inflateBackEnd(&strm);
    # 断言ret等于Z_OK，即判断inflateBackEnd函数是否返回Z_OK
    assert(ret == Z_OK);
    
    # 在标准错误流中输出提示信息
    fputs("inflateBack built-in memory routines\n", stderr);
# 定义一个名为 try 的函数，接受十六进制数据、标识符和错误码作为参数
local int try(char *hex, char *id, int err)
{
    int ret;  # 定义整型变量 ret
    unsigned len, size;  # 定义无符号整型变量 len 和 size
    unsigned char *in, *out, *win;  # 定义无符号字符指针变量 in, out, win
    char *prefix;  # 定义字符指针变量 prefix
    z_stream strm;  # 定义 z_stream 结构体变量 strm

    /* convert to hex */
    in = h2b(hex, &len);  # 将十六进制数据转换为二进制数据，并将长度存储在 len 中
    assert(in != NULL);  # 断言二进制数据不为空

    /* allocate work areas */
    size = len << 3;  # 计算 size 的值
    out = malloc(size);  # 分配大小为 size 的内存空间给 out
    assert(out != NULL);  # 断言 out 不为空
    win = malloc(32768);  # 分配大小为 32768 的内存空间给 win
    assert(win != NULL);  # 断言 win 不为空
    prefix = malloc(strlen(id) + 6);  # 分配大小为标识符长度加 6 的内存空间给 prefix
    assert(prefix != NULL);  # 断言 prefix 不为空

    /* first with inflate */
    strcpy(prefix, id);  # 将 id 复制到 prefix
    strcat(prefix, "-late");  # 将 "-late" 连接到 prefix
    mem_setup(&strm);  # 调用 mem_setup 函数，传入 strm 的地址
    strm.avail_in = 0;  # 设置 strm 的 avail_in 为 0
    strm.next_in = Z_NULL;  # 设置 strm 的 next_in 为 Z_NULL
    ret = inflateInit2(&strm, err < 0 ? 47 : -15);  # 调用 inflateInit2 函数，传入 strm 和错误码
    assert(ret == Z_OK);  # 断言 ret 等于 Z_OK
    strm.avail_in = len;  # 设置 strm 的 avail_in 为 len
    strm.next_in = in;  # 设置 strm 的 next_in 为 in
    do {
        strm.avail_out = size;  # 设置 strm 的 avail_out 为 size
        strm.next_out = out;  # 设置 strm 的 next_out 为 out
        ret = inflate(&strm, Z_TREES);  # 调用 inflate 函数，传入 strm 和 Z_TREES
        assert(ret != Z_STREAM_ERROR && ret != Z_MEM_ERROR);  # 断言 ret 不等于 Z_STREAM_ERROR 和 Z_MEM_ERROR
        if (ret == Z_DATA_ERROR || ret == Z_NEED_DICT)  # 如果 ret 等于 Z_DATA_ERROR 或 Z_NEED_DICT
            break;  # 跳出循环
    } while (strm.avail_in || strm.avail_out == 0);  # 循环条件
    if (err) {  # 如果 err 不为 0
        assert(ret == Z_DATA_ERROR);  # 断言 ret 等于 Z_DATA_ERROR
        assert(strcmp(id, strm.msg) == 0);  # 断言 id 与 strm.msg 相等
    }
    inflateEnd(&strm);  # 调用 inflateEnd 函数，传入 strm
    mem_done(&strm, prefix);  # 调用 mem_done 函数，传入 strm 和 prefix

    /* then with inflateBack */
    if (err >= 0) {  # 如果 err 大于等于 0
        strcpy(prefix, id);  # 将 id 复制到 prefix
        strcat(prefix, "-back");  # 将 "-back" 连接到 prefix
        mem_setup(&strm);  # 调用 mem_setup 函数，传入 strm 的地址
        ret = inflateBackInit(&strm, 15, win);  # 调用 inflateBackInit 函数，传入 strm、15 和 win
        assert(ret == Z_OK);  # 断言 ret 等于 Z_OK
        strm.avail_in = len;  # 设置 strm 的 avail_in 为 len
        strm.next_in = in;  # 设置 strm 的 next_in 为 in
        ret = inflateBack(&strm, pull, Z_NULL, push, Z_NULL);  # 调用 inflateBack 函数，传入 strm、pull 和 push
        assert(ret != Z_STREAM_ERROR);  # 断言 ret 不等于 Z_STREAM_ERROR
        if (err) {  # 如果 err 不为 0
            assert(ret == Z_DATA_ERROR);  # 断言 ret 等于 Z_DATA_ERROR
            assert(strcmp(id, strm.msg) == 0);  # 断言 id 与 strm.msg 相等
        }
        inflateBackEnd(&strm);  # 调用 inflateBackEnd 函数，传入 strm
        mem_done(&strm, prefix);  # 调用 mem_done 函数，传入 strm 和 prefix
    }

    /* clean up */
    free(prefix);  # 释放 prefix 的内存空间
    free(win);  # 释放 win 的内存空间
    free(out);  # 释放 out 的内存空间
    free(in);  # 释放 in 的内存空间
    return ret;  # 返回 ret
}

/* cover deflate data cases in both inflate() and inflateBack() */
# 定义一个名为 cover_inflate 的函数
local void cover_inflate(void)
{
    # 调用 try 函数，传入参数 "0 0 0 0 0" 和 "invalid stored block lengths"，并传入 1 作为第三个参数
    try("0 0 0 0 0", "invalid stored block lengths", 1);
    # 调用 try 函数，传入参数 "3 0" 和 "fixed"，并传入 0 作为第三个参数
    try("3 0", "fixed", 0);
    # 调用 try 函数，传入参数 "6" 和 "invalid block type"，并传入 1 作为第三个参数
    try("6", "invalid block type", 1);
    # 调用 try 函数，传入参数 "1 1 0 fe ff 0" 和 "stored"，并传入 0 作为第三个参数
    try("1 1 0 fe ff 0", "stored", 0);
    # 调用 try 函数，传入参数 "fc 0 0" 和 "too many length or distance symbols"，并传入 1 作为第三个参数
    try("fc 0 0", "too many length or distance symbols", 1);
    # 调用 try 函数，传入参数 "4 0 fe ff" 和 "invalid code lengths set"，并传入 1 作为第三个参数
    try("4 0 fe ff", "invalid code lengths set", 1);
    # 调用 try 函数，传入参数 "4 0 24 49 0" 和 "invalid bit length repeat"，并传入 1 作为第三个参数
    try("4 0 24 49 0", "invalid bit length repeat", 1);
    # 调用 try 函数，传入参数 "4 0 24 e9 ff ff" 和 "invalid bit length repeat"，并传入 1 作为第三个参数
    try("4 0 24 e9 ff ff", "invalid bit length repeat", 1);
    # 调用 try 函数，传入参数 "4 0 24 e9 ff 6d" 和 "invalid code -- missing end-of-block"，并传入 1 作为第三个参数
    try("4 0 24 e9 ff 6d", "invalid code -- missing end-of-block", 1);
    # 调用 try 函数，传入参数 "4 80 49 92 24 49 92 24 71 ff ff 93 11 0" 和 "invalid literal/lengths set"，并传入 1 作为第三个参数
    try("4 80 49 92 24 49 92 24 71 ff ff 93 11 0", "invalid literal/lengths set", 1);
    # 调用 try 函数，传入参数 "4 80 49 92 24 49 92 24 f b4 ff ff c3 84" 和 "invalid distances set"，并传入 1 作为第三个参数
    try("4 80 49 92 24 49 92 24 f b4 ff ff c3 84", "invalid distances set", 1);
    # 调用 try 函数，传入参数 "4 c0 81 8 0 0 0 0 20 7f eb b 0 0" 和 "invalid literal/length code"，并传入 1 作为第三个参数
    try("4 c0 81 8 0 0 0 0 20 7f eb b 0 0", "invalid literal/length code", 1);
    # 调用 try 函数，传入参数 "2 7e ff ff" 和 "invalid distance code"，并传入 1 作为第三个参数
    try("2 7e ff ff", "invalid distance code", 1);
    # 调用 try 函数，传入参数 "c c0 81 0 0 0 0 0 90 ff 6b 4 0" 和 "invalid distance too far back"，并传入 1 作为第三个参数
    try("c c0 81 0 0 0 0 0 90 ff 6b 4 0", "invalid distance too far back", 1);

    /* also trailer mismatch just in inflate() */
    # 调用 try 函数，传入参数 "1f 8b 8 0 0 0 0 0 0 0 3 0 0 0 0 1" 和 "incorrect data check"，并传入 -1 作为第三个参数
    try("1f 8b 8 0 0 0 0 0 0 0 3 0 0 0 0 1", "incorrect data check", -1);
    # 调用 try 函数，传入参数 "1f 8b 8 0 0 0 0 0 0 0 3 0 0 0 0 0 0 0 0 1" 和 "incorrect length check"，并传入 -1 作为第三个参数
    try("1f 8b 8 0 0 0 0 0 0 0 3 0 0 0 0 0 0 0 0 1", "incorrect length check", -1);
    # 调用 try 函数，传入参数 "5 c0 21 d 0 0 0 80 b0 fe 6d 2f 91 6c" 和 "pull 17"，并传入 0 作为第三个参数
    try("5 c0 21 d 0 0 0 80 b0 fe 6d 2f 91 6c", "pull 17", 0);
    # 调用 try 函数，传入参数 "5 e0 81 91 24 cb b2 2c 49 e2 f 2e 8b 9a 47 56 9f fb fe ec d2 ff 1f" 和 "long code"，并传入 0 作为第三个参数
    try("5 e0 81 91 24 cb b2 2c 49 e2 f 2e 8b 9a 47 56 9f fb fe ec d2 ff 1f", "long code", 0);
    # 调用 try 函数，传入参数 "ed c0 1 1 0 0 0 40 20 ff 57 1b 42 2c 4f" 和 "length extra"，并传入 0 作为第三个参数
    try("ed c0 1 1 0 0 0 40 20 ff 57 1b 42 2c 4f", "length extra", 0);
    # 调用 try 函数，传入参数 "ed cf c1 b1 2c 47 10 c4 30 fa 6f 35 1d 1 82 59 3d fb be 2e 2a fc f c" 和 "long distance and extra"，并传入 0 作为第三个参数
    try("ed cf c1 b1 2c 47 10 c4 30 fa 6f 35 1d 1 82 59 3d fb be 2e 2a fc f c", "long distance and extra", 0);
    # 调用 try 函数，传入参数 "ed c0 81 0 0 0 0 80 a0 fd a9 17 a9 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 "
    # "0 0 0
    /* 我们需要直接调用 inflate_table() 以表现出不够的错误，因为 zlib 保证足够总是足够的 */
    for (bits = 0; bits < 15; bits++)
        lens[bits] = (unsigned short)(bits + 1);
    lens[15] = 15;
    next = table;
    bits = 15;
    ret = inflate_table(DISTS, lens, 16, &next, &bits, work);
                                                assert(ret == 1);
    next = table;
    bits = 1;
    ret = inflate_table(DISTS, lens, 16, &next, &bits, work);
                                                assert(ret == 1);
    fputs("inflate_table not enough errors\n", stderr);
# cover remaining inffast.c decoding and window copying
local void cover_fast(void):
    # 覆盖剩余的inffast.c解码和窗口复制
    inf("e5 e0 81 ad 6d cb b2 2c c9 01 1e 59 63 ae 7d ee fb 4d fd b5 35 41 68"
        " ff 7f 0f 0 0 0", "fast length extra bits", 0, -8, 258, Z_DATA_ERROR);
    # 解码长度的额外位
    inf("25 fd 81 b5 6d 59 b6 6a 49 ea af 35 6 34 eb 8c b9 f6 b9 1e ef 67 49"
        " 50 fe ff ff 3f 0 0", "fast distance extra bits", 0, -8, 258,
        Z_DATA_ERROR);
    # 解码距离的额外位
    inf("3 7e 0 0 0 0 0", "fast invalid distance code", 0, -8, 258,
        Z_DATA_ERROR);
    # 无效的距离码
    inf("1b 7 0 0 0 0 0", "fast invalid literal/length code", 0, -8, 258,
        Z_DATA_ERROR);
    # 无效的文字/长度码
    inf("d c7 1 ae eb 38 c 4 41 a0 87 72 de df fb 1f b8 36 b1 38 5d ff ff 0",
        "fast 2nd level codes and too far back", 0, -8, 258, Z_DATA_ERROR);
    # 第二级码和太远的情况
    inf("63 18 5 8c 10 8 0 0 0 0", "very common case", 0, -8, 259, Z_OK);
    # 非常常见的情况
    inf("63 60 60 18 c9 0 8 18 18 18 26 c0 28 0 29 0 0 0",
        "contiguous and wrap around window", 6, -8, 259, Z_OK);
    # 连续和环绕窗口
    inf("63 0 3 0 0 0 0 0", "copy direct from output", 0, -8, 259,
        Z_STREAM_END);
    # 直接从输出复制
}

int main(void):
    # 打印zlib版本信息
    fprintf(stderr, "%s\n", zlibVersion());
    # 覆盖支持
    cover_support();
    # 覆盖包装
    cover_wrap();
    # 覆盖返回
    cover_back();
    # 覆盖膨胀
    cover_inflate();
    # 覆盖树
    cover_trees();
    # 覆盖快速
    cover_fast();
    # 返回0
    return 0;
```