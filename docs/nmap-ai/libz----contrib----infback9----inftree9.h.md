# `nmap\libz\contrib\infback9\inftree9.h`

```cpp
/* inftree9.h -- header to use inftree9.c
 * Copyright (C) 1995-2008 Mark Adler
 * For conditions of distribution and use, see copyright notice in zlib.h
 */
# 用于包含 inftree9.c 的头文件，版权信息和使用条件
# 版权归 Mark Adler 所有，具体的发布和使用条件请参见 zlib.h 中的版权声明

/* WARNING: this file should *not* be used by applications. It is
   part of the implementation of the compression library and is
   subject to change. Applications should only use zlib.h.
 */
# 警告：此文件不应被应用程序使用。它是压缩库实现的一部分，可能会发生变化。应用程序应该只使用 zlib.h。

/* Structure for decoding tables.  Each entry provides either the
   information needed to do the operation requested by the code that
   indexed that table entry, or it provides a pointer to another
   table that indexes more bits of the code.  op indicates whether
   the entry is a pointer to another table, a literal, a length or
   distance, an end-of-block, or an invalid code.  For a table
   pointer, the low four bits of op is the number of index bits of
   that table.  For a length or distance, the low four bits of op
   is the number of extra bits to get after the code.  bits is
   the number of bits in this code or part of the code to drop off
   of the bit buffer.  val is the actual byte to output in the case
   of a literal, the base length or distance, or the offset from
   the current table to the next table.  Each entry is four bytes. */
# 解码表的结构。每个条目提供执行由索引该表条目的代码请求的操作所需的信息，或者提供指向另一个索引代码更多位的表的指针。op指示条目是指向另一个表、字面值、长度或距离、块结束或无效代码。对于表指针，op的低四位是该表的索引位数。对于长度或距离，op的低四位是代码后面的额外位数。bits是代码或代码部分中的位数，用于从位缓冲区中删除。val是字面值、基本长度或距离，或者从当前表到下一个表的偏移量的实际字节。每个条目占四个字节。

typedef struct {
    unsigned char op;           /* operation, extra bits, table bits */
    unsigned char bits;         /* bits in this part of the code */
    unsigned short val;         /* offset in table or code value */
} code;
# 用于解码表的结构体，包括操作、额外位、表位数、代码部分的位数、表中的偏移量或代码值

/* op values as set by inflate_table():
    00000000 - literal
    0000tttt - table link, tttt != 0 is the number of table index bits
    100eeeee - length or distance, eeee is the number of extra bits
    01100000 - end of block
    01000000 - invalid code
 */
# 由 inflate_table() 设置的 op 值：
# 00000000 - 字面值
# 0000tttt - 表链接，tttt != 0 是表索引位数
# 100eeeee - 长度或距离，eeee 是额外位数
# 01100000 - 块结束
# 01000000 - 无效代码
/* 定义动态表的最大大小。代码结构的最大数量为1446，其中852为文字/长度代码的总和，594为距离代码的总和。这些值是通过使用 zlib 发行版中的程序 examples/enough.c 进行详尽搜索得到的。该程序的参数是符号的数量、初始根表大小和代码的最大位长度。"enough 286 9 15" 用于文字/长度代码返回852，"enough 32 6 15" 用于距离代码返回594。初始根表大小（9或6）可以在 infback9.c 中的 inflate_table() 调用的第五个参数中找到。如果更改根表大小，则需要重新计算和更新这些最大大小。 */
#define ENOUGH_LENS 852
#define ENOUGH_DISTS 594
#define ENOUGH (ENOUGH_LENS+ENOUGH_DISTS)

/* 用于构建 inflate_table9() 的代码类型 */
typedef enum {
    CODES,
    LENS,
    DISTS
} codetype;

/* 声明 inflate_table9 函数 */
extern int inflate_table9 OF((codetype type, unsigned short FAR *lens,
                             unsigned codes, code FAR * FAR *table,
                             unsigned FAR *bits, unsigned short FAR *work));
```