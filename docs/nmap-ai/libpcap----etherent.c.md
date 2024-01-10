# `nmap\libpcap\etherent.c`

```
/*
 * 版权声明，版权归加利福尼亚大学所有
 * 允许在源代码和二进制形式下进行再发布和使用，无论是否进行修改
 * 在满足以下条件的情况下：(1)源代码发布时保留以上版权声明和本段完整内容，(2)包含二进制代码的发布包括以上版权声明和本段完整内容在文档或其他提供的材料中，(3)所有提及此软件特性或使用的广告材料展示以下声明："本产品包含由加利福尼亚大学劳伦斯伯克利实验室及其贡献者开发的软件"。未经特定事先书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品。
 * 本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <pcap-types.h>

#include <memory.h>
#include <stdio.h>
#include <string.h>

#include "pcap-int.h"

#include <pcap/namedb.h>

#ifdef HAVE_OS_PROTO_H
#include "os-proto.h"
#endif

// 跳过文件中的空格
static inline int skip_space(FILE *);

// 跳过文件中的一行
static inline int skip_line(FILE *);

/* 十六进制数字转换为整数 */
static inline u_char
xdtoi(u_char c)
{
    if (c >= '0' && c <= '9')
        return (u_char)(c - '0');
    else if (c >= 'a' && c <= 'f')
        return (u_char)(c - 'a' + 10);
    else
        return (u_char)(c - 'A' + 10);
}

/*
 * 跳过连续的空白字符（空格和制表符）和 LF 之前的任何 CR
 * 当遇到非空白字符或行尾的 LF 时停止
 */
static inline int
skip_space(FILE *f)
{
    int c;

    do {
        c = getc(f);
    // 当字符为空格、制表符或回车时，继续循环
    } while (c == ' ' || c == '\t' || c == '\r');
    
    // 返回当前字符
    return c;
# 跳过文件中的一行内容
static inline int
skip_line(FILE *f)
{
    # 定义变量 c 用于存储读取的字符
    int c;

    # 读取文件内容，直到遇到换行符或文件结束
    do
        c = getc(f);
    while (c != '\n' && c != EOF);

    # 返回读取的字符
    return c;
}

# 读取下一个以太网地址
struct pcap_etherent *
pcap_next_etherent(FILE *fp)
{
    # 定义变量 c 用于存储读取的字符，i 用于循环计数
    register int c, i;
    # 定义变量 d 用于存储读取的字节，bp 用于存储地址名称，namesize 用于存储地址名称的大小
    u_char d;
    char *bp;
    size_t namesize;
    # 定义静态的以太网地址结构体 e
    static struct pcap_etherent e;

    # 将以太网地址结构体 e 的内容初始化为 0
    memset((char *)&e, 0, sizeof(e));
    }
}
```