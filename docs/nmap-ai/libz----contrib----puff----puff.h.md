# `nmap\libz\contrib\puff\puff.h`

```
/*
  puff.h
  版权所有 (C) 2002-2013 Mark Adler
  版本 2.3, 2013年1月21日

  本软件按原样提供，不提供任何明示或暗示的保证。作者不对使用本软件而产生的任何损害承担责任。

  授予任何人使用本软件的权限，包括商业应用，并且可以自由修改和重新分发，但需遵守以下限制：

  1. 不得歪曲本软件的原始来源；不得声称您编写了原始软件。如果您在产品中使用本软件，产品文档中的致谢将不是必需的，但会受到赞赏。
  2. 修改后的源代码必须明确标记，并且不得歪曲为原始软件。
  3. 任何来源分发的代码中不得删除或更改本通知。

  Mark Adler    madler@alumni.caltech.edu
 */

/*
 * 有关目的和用法，请参阅puff.c。
 */
#ifndef NIL
#  define NIL ((unsigned char *)0)      /* 用于无输出选项 */
#endif

int puff(unsigned char *dest,           /* 指向目标指针的指针 */
         unsigned long *destlen,        /* 输出空间的大小 */
         const unsigned char *source,   /* 指向源数据指针 */
         unsigned long *sourcelen);     /* 可用输入的大小 */
```