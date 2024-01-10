# `nmap\libdnet-stripped\src\blob.c`

```
/*
 * blob.c
 *
 * Copyright (c) 2002 Dug Song <dugsong@monkey.org>
 *
 * $Id: blob.c 615 2006-01-08 16:06:49Z dugsong $
 */

#ifdef _WIN32
#include "dnet_winconfig.h"
#else
#include "config.h"
#endif

#include <ctype.h>
#include <stdarg.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "dnet.h"

static void    *(*bl_malloc)(size_t) = malloc;
static void    *(*bl_realloc)(void *, size_t) = realloc;
static void     (*bl_free)(void *) = free;
static int       bl_size = BUFSIZ;

static int       fmt_D(int, int, blob_t *, va_list *);
static int       fmt_H(int, int, blob_t *, va_list *);
static int       fmt_b(int, int, blob_t *, va_list *);
static int       fmt_c(int, int, blob_t *, va_list *);
static int       fmt_d(int, int, blob_t *, va_list *);
static int       fmt_h(int, int, blob_t *, va_list *);
static int       fmt_s(int, int, blob_t *, va_list *);

static void       print_hexl(blob_t *);

static blob_fmt_cb blob_ascii_fmt[] = {
    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,
    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,
    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,
    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,
    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,
    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,
    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,
    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,
    NULL,    NULL,    NULL,    NULL,    fmt_D,    NULL,    NULL,    NULL,
    fmt_H,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,
    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,
    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,
    NULL,    NULL,    fmt_b,    fmt_c,    fmt_d,    NULL,    NULL,    NULL,
    fmt_h,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,    NULL,
    # 这是一个包含多个空值和一个字符串格式化的列表
    # 在这个列表中，除了第四个元素是一个字符串格式化之外，其他都是空值
};

// 定义一个结构体，包含名称和打印函数指针
struct blob_printer {
    char      *name;
    void     (*print)(blob_t *);
} blob_printers[] = {
    { "hexl",    print_hexl },  // 初始化结构体数组，包含名称和对应的打印函数
    { NULL,        NULL },  // 结构体数组的结束标志
};

// 创建一个新的 blob_t 对象
blob_t *
blob_new(void)
{
    blob_t *b;

    // 分配内存给 blob_t 对象
    if ((b = bl_malloc(sizeof(*b))) != NULL) {
        b->off = b->end = 0;  // 初始化偏移和结束位置
        b->size = bl_size;  // 设置大小
        // 分配内存给 base
        if ((b->base = bl_malloc(b->size)) == NULL) {
            bl_free(b);  // 释放内存
            b = NULL;
        }
    }
    return (b);  // 返回 blob_t 对象
}

// 为 blob_t 对象保留空间
static int
blob_reserve(blob_t *b, int len)
{
    void *p;
    int nsize;

    // 如果当前大小不足以容纳 len 长度的数据
    if (b->size < b->end + len) {
        if (b->size == 0)
            return (-1);  // 如果当前大小为 0，返回错误

        // 计算新的大小
        if ((nsize = b->end + len) > bl_size)
            nsize = ((nsize / bl_size) + 1) * bl_size;
        
        // 重新分配内存
        if ((p = bl_realloc(b->base, nsize)) == NULL)
            return (-1);  // 重新分配失败，返回错误
        
        b->base = p;  // 更新 base 指针
        b->size = nsize;  // 更新大小
    }
    b->end += len;  // 更新结束位置
    
    return (0);  // 返回成功
}

// 从 blob_t 对象中读取数据
int
blob_read(blob_t *b, void *buf, int len)
{
    if (b->end - b->off < len)
        len = b->end - b->off;  // 如果剩余长度不足，只读取剩余长度
    
    memcpy(buf, b->base + b->off, len);  // 复制数据到 buf
    b->off += len;  // 更新偏移位置
    
    return (len);  // 返回读取的长度
}

// 向 blob_t 对象中写入数据
int
blob_write(blob_t *b, const void *buf, int len)
{
    if (b->off + len <= b->end ||
        blob_reserve(b, b->off + len - b->end) == 0) {
        memcpy(b->base + b->off, (u_char *)buf, len);  // 复制数据到 base
        b->off += len;  // 更新偏移位置
        return (len);  // 返回写入的长度
    }
    return (-1);  // 写入失败，返回错误
}

// 向 blob_t 对象中插入数据
int
blob_insert(blob_t *b, const void *buf, int len)
{
    if (blob_reserve(b, len) == 0 && b->size) {
        if (b->end - b->off > 0)
            memmove( b->base + b->off + len, b->base + b->off, b->end - b->off);  // 移动数据
        memcpy(b->base + b->off, buf, len);  // 复制数据到 base
        b->off += len;  // 更新偏移位置
        return (len);  // 返回插入的长度
    }
    return (-1);  // 插入失败，返回错误
}

// 从 blob_t 对象中删除数据
int
blob_delete(blob_t *b, void *buf, int len)
{
    if (b->off + len <= b->end && b->size) {
        if (buf != NULL)
            memcpy(buf, b->base + b->off, len);  // 复制数据到 buf
        memmove(b->base + b->off, b->base + b->off + len, b->end - (b->off + len));  // 移动数据
        b->end -= len;  // 更新结束位置
        return (len);  // 返回删除的长度
    }
    # 返回值为 -1
    return (-1);
# 定义一个静态函数，用于将数据按照指定格式打包到 blob_t 结构中
static int
blob_fmt(blob_t *b, int pack, const char *fmt, va_list *ap)
{
    # 定义格式化回调函数指针和临时变量
    blob_fmt_cb fmt_cb;
    char *p;
    int len;

    # 遍历格式字符串
    for (p = (char *)fmt; *p != '\0'; p++) {
        # 如果遇到 % 符号，表示需要进行格式化
        if (*p == '%') {
            p++;
            # 判断是否有指定长度
            if (isdigit((int) (unsigned char) *p)) {
                len = strtol(p, &p, 10);
            } else if (*p == '*') {
                len = va_arg(*ap, int);
                p++;
            } else
                len = 0;
            
            # 根据格式字符获取对应的格式化回调函数
            if ((fmt_cb = blob_ascii_fmt[(int)*p]) == NULL)
                return (-1);

            # 调用格式化回调函数进行格式化
            if ((*fmt_cb)(pack, len, b, ap) < 0)
                return (-1);
        } else {
            # 如果不是 % 符号，则直接处理字符
            if (pack) {
                if (b->off + 1 < b->end ||
                    blob_reserve(b, b->off + 1 - b->end) == 0)
                    b->base[b->off++] = *p;
                else
                    return (-1);
            } else {
                if (b->base[b->off++] != *p)
                    return (-1);
            }
        }
    }
    return (0);
}

# 将数据按照指定格式打包到 blob_t 结构中
int
blob_pack(blob_t *b, const char *fmt, ...)
{
    va_list ap;
    va_start(ap, fmt);
    return (blob_fmt(b, 1, fmt, &ap));
}

# 从 blob_t 结构中按照指定格式解包数据
int
blob_unpack(blob_t *b, const char *fmt, ...)
{
    va_list ap;
    va_start(ap, fmt);
    return (blob_fmt(b, 0, fmt, &ap));
}

# 移动 blob_t 结构中的偏移量
int
blob_seek(blob_t *b, int off, int whence)
{
    if (whence == SEEK_CUR)
        off += b->off;
    else if (whence == SEEK_END)
        off += b->end;

    if (off < 0 || off > b->end)
        return (-1);
    
    return ((b->off = off));
}

# 在 blob_t 结构中查找指定数据的索引
int
blob_index(blob_t *b, const void *buf, int len)
{
    int i;

    for (i = b->off; i <= b->end - len; i++) {
        if (memcmp(b->base + i, buf, len) == 0)
            return (i);
    }
    return (-1);
}

# 在 blob_t 结构中反向查找指定数据的索引
int
blob_rindex(blob_t *b, const void *buf, int len)
{
    int i;

    for (i = b->end - len; i >= 0; i--) {
        if (memcmp(b->base + i, buf, len) == 0)
            return (i);
    }
    return (-1);
}

# 打印 blob_t 结构中的数据
int
blob_print(blob_t *b, char *style, int len)
    # 声明一个指向结构体的指针变量bp
    struct blob_printer *bp;
    
    # 遍历blob_printers数组中的每个元素，直到遇到name为NULL的元素
    for (bp = blob_printers; bp->name != NULL; bp++) {
        # 检查当前元素的name是否与给定的style相等
        if (strcmp(bp->name, style) == 0)
            # 如果相等，则调用当前元素的print函数，打印b的内容
            bp->print(b);
    }
    
    # 返回0，表示执行成功
    return (0);
}

这是一个函数的结束标志。


int
blob_sprint(blob_t *b, char *style, int len, char *dst, int size)
{
    return (0);
}

定义了一个名为blob_sprint的函数，返回类型为int，接受blob_t类型指针b，char类型指针style，int类型len，char类型指针dst和int类型size作为参数。函数体内部直接返回0。


blob_t *
blob_free(blob_t *b)
{
    if (b->size)
        bl_free(b->base);
    bl_free(b);
    return (NULL);
}

定义了一个名为blob_free的函数，返回类型为blob_t指针，接受blob_t类型指针b作为参数。如果b的size不为0，则释放b的base指针所指向的内存，然后释放b指针所指向的内存，最后返回NULL。


int
blob_register_alloc(size_t size, void *(bmalloc)(size_t),
    void (*bfree)(void *), void *(*brealloc)(void *, size_t))
{
    bl_size = size;
    if (bmalloc != NULL)
        bl_malloc = bmalloc;
    if (bfree != NULL)
        bl_free = bfree;
    if (brealloc != NULL)
        bl_realloc = brealloc;
    return (0);
}

定义了一个名为blob_register_alloc的函数，返回类型为int，接受size_t类型size和三个函数指针作为参数。将size赋值给bl_size，如果bmalloc不为NULL，则将bmalloc赋值给bl_malloc，如果bfree不为NULL，则将bfree赋值给bl_free，如果brealloc不为NULL，则将brealloc赋值给bl_realloc，最后返回0。


int
blob_register_pack(char c, blob_fmt_cb fmt_cb)
{
    if (blob_ascii_fmt[(int)c] == NULL) {
        blob_ascii_fmt[(int)c] = fmt_cb;
        return (0);
    }
    return (-1);
}

定义了一个名为blob_register_pack的函数，返回类型为int，接受char类型c和blob_fmt_cb类型的fmt_cb作为参数。如果blob_ascii_fmt[(int)c]为NULL，则将fmt_cb赋值给blob_ascii_fmt[(int)c]，并返回0，否则返回-1。

以下是一系列名为fmt_开头的静态函数，它们用于处理不同类型的数据的打包和解包操作。每个函数都接受pack、len、blob_t类型指针b和va_list类型指针ap作为参数，根据pack和len的值进行相应的操作，并返回0或-1。
    # 如果 pack 为真，则执行以下代码块
    if (pack) {
        # 从可变参数列表中获取一个 int 类型的参数，并将其写入到二进制流中
        uint8_t n = va_arg(*ap, int);
        return (blob_write(b, &n, sizeof(n)));
    } 
    # 如果 pack 为假，则执行以下代码块
    else {
        # 从可变参数列表中获取一个 uint8_t 类型的参数，并将其从二进制流中读取出来
        uint8_t *n = va_arg(*ap, uint8_t *);
        return (blob_read(b, n, sizeof(*n)));
    }
# 定义一个处理整数类型数据的格式化函数，参数为是否打包、长度、数据流、可变参数列表
static int
fmt_d(int pack, int len, blob_t *b, va_list *ap)
{
    # 如果长度不为0，则返回-1
    if (len) return (-1);
    
    # 如果打包为真
    if (pack) {
        # 从可变参数列表中获取一个无符号32位整数
        uint32_t n = va_arg(*ap, uint32_t);
        # 将该整数写入数据流中
        return (blob_write(b, &n, sizeof(n)));
    } else {
        # 否则从可变参数列表中获取一个无符号32位整数指针
        uint32_t *n = va_arg(*ap, uint32_t *);
        # 从数据流中读取一个无符号32位整数
        return (blob_read(b, n, sizeof(*n)));
    }
}

# 定义一个处理16位整数类型数据的格式化函数，参数为是否打包、长度、数据流、可变参数列表
static int
fmt_h(int pack, int len, blob_t *b, va_list *ap)
{
    # 如果长度不为0，则返回-1
    if (len) return (-1);
    
    # 如果打包为真
    if (pack) {
        # 从可变参数列表中获取一个16位整数
        uint16_t n = va_arg(*ap, int);
        # 将该整数写入数据流中
        return (blob_write(b, &n, sizeof(n)));
    } else {
        # 否则从可变参数列表中获取一个16位整数指针
        uint16_t *n = va_arg(*ap, uint16_t *);
        # 从数据流中读取一个16位整数
        return (blob_read(b, n, sizeof(*n)));
    }
}

# 定义一个处理字符串类型数据的格式化函数，参数为是否打包、长度、数据流、可变参数列表
static int
fmt_s(int pack, int len, blob_t *b, va_list *ap)
{
    # 从可变参数列表中获取一个字符指针
    char *p = va_arg(*ap, char *);
    # 初始化一个字符c为'\0'，以及整数i和end
    char c = '\0';
    int i, end;
    
    # 如果打包为真
    if (pack) {
        # 如果长度大于0
        if (len > 0) {
            # 如果最后一个字符不是'\0'，则将其替换为'\0'
            if ((c = p[len - 1]) != '\0')
                p[len - 1] = '\0';
        } else
            # 否则将长度设置为字符串长度加1
            len = strlen(p) + 1;
        
        # 将字符串写入数据流中
        if (blob_write(b, p, len) > 0) {
            # 如果之前替换了最后一个字符，则恢复
            if (c != '\0')
                p[len - 1] = c;
            return (len);
        }
    } else {
        # 如果长度小于等于0，则返回-1
        if (len <= 0) return (-1);

        # 如果数据流剩余长度小于要读取的长度，则取两者中较小的值
        if ((end = b->end - b->off) < len)
            end = len;
        
        # 遍历数据流，将数据读取到字符串中
        for (i = 0; i < end; i++) {
            if ((p[i] = b->base[b->off + i]) == '\0') {
                b->off += i + 1;
                return (i);
            }
        }
    }
    return (-1);
}

# 定义一个打印16进制数据的函数，参数为数据流
static void
print_hexl(blob_t *b)
{
    u_int i, j, jm, len;
    u_char *p;
    int c;

    # 获取数据流的起始位置和长度
    p = b->base + b->off;
    len = b->end - b->off;
    
    # 打印换行符
    printf("\n");
    # 以每16个字节为一组进行循环
    for (i = 0; i < len; i += 0x10) {
        # 打印当前行的起始地址
        printf("  %04x: ", (u_int)(i + b->off));
        # 计算当前行剩余的字节数
        jm = len - i;
        # 如果剩余字节数大于16，则设置为16，否则为剩余字节数
        jm = jm > 16 ? 16 : jm;
        
        # 打印当前行的16进制数据
        for (j = 0; j < jm; j++) {
            printf((j % 2) ? "%02x " : "%02x", (u_int)p[i + j]);
        }
        # 如果当前行不足16个字节，用空格填充
        for (; j < 16; j++) {
            printf((j % 2) ? "   " : "  ");
        }
        # 打印空格
        
        # 打印当前行的可打印字符，不可打印字符用'.'代替
        for (j = 0; j < jm; j++) {
            c = p[i + j];
            printf("%c", isprint(c) ? c : '.');
        }
        # 换行
        printf("\n");
    }
# 闭合前面的函数定义
```