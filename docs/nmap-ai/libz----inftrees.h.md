# `nmap\libz\inftrees.h`

```cpp
# inftrees.h -- 用于使用inftrees.c的头文件
# 版权所有（C）1995-2005，2010 Mark Adler
# 分发和使用条件，请参见zlib.h中的版权声明

# 警告：此文件不应由应用程序使用。它是压缩库实现的一部分，并且可能会更改。应用程序应仅使用zlib.h。

# 用于解码表的结构。每个条目都提供了执行由索引该表条目的代码请求的操作所需的信息，或者提供了指向另一个表的指针，该表索引代码的更多位。op指示条目是指向另一个表，文字，长度或距离，块结束还是无效代码。对于表指针，op的低四位是该表的索引位数。对于长度或距离，op的低四位是在代码之后获取的额外位数。bits是要从位缓冲区中删除的代码或代码的位数。val是文字，基本长度或距离，或者到下一个表的当前表的偏移量。每个条目是四个字节。
typedef struct {
    unsigned char op;           /* 操作，额外位，表位数 */
    unsigned char bits;         /* 该代码部分的位数 */
    unsigned short val;         /* 表中的偏移量或代码值 */
} code;

# 由inflate_table()设置的op值：
# 00000000 - 文字
# 0000tttt - 表链接，tttt！= 0是表索引位数
# 0001eeee - 长度或距离，eeee是额外位数
# 01100000 - 块结束
# 01000000 - 无效代码
/* 定义动态表的最大大小。代码结构的最大数量为1444，其中字面/长度代码为852，距离代码为592。这些值是通过使用zlib发行版中的程序examples/enough.c进行详尽搜索找到的。该程序的参数是符号数量、初始根表大小和代码的最大位长度。"enough 286 9 15"用于字面/长度代码返回852，"enough 30 6 15"用于距离代码返回592。初始根表大小（9或6）可以在inflate.c和infback.c中的inflate_table()调用的第五个参数中找到。如果根表大小被更改，那么这些最大大小需要重新计算和更新。 */
#define ENOUGH_LENS 852
#define ENOUGH_DISTS 592
#define ENOUGH (ENOUGH_LENS+ENOUGH_DISTS)

/* 用于inflate_table()构建的代码类型 */
typedef enum {
    CODES,
    LENS,
    DISTS
} codetype;

int ZLIB_INTERNAL inflate_table OF((codetype type, unsigned short FAR *lens,
                             unsigned codes, code FAR * FAR *table,
                             unsigned FAR *bits, unsigned short FAR *work));
```