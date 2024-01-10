# `nmap\libdnet-stripped\src\strsep.c`

```
/*    $OpenBSD: strsep.c,v 1.3 1997/08/20 04:28:14 millert Exp $    */
/*-
 * 版权所有（c）1990, 1993
 *    加利福尼亚大学董事会。保留所有权利。
 *
 * 在源代码和二进制形式中重新分发和使用，无论是否经过修改，
 * 都必须满足以下条件：
 * 1. 源代码的再分发必须保留上述版权
 *    通知，此条件列表和以下免责声明。
 * 2. 以二进制形式重新分发必须在
 *    文档和/或随附分发的其他材料中复制上述版权
 *    通知，此条件列表和以下免责声明。
 * 3. 所有宣传材料提及此软件的特性或使用
 *    必须显示以下确认：
 *    本产品包括由加利福尼亚大学开发的软件
 *    加利福尼亚大学伯克利分校及其贡献者。
 * 4. 未经特定事先书面许可，不得使用大学的名称
 *    或其贡献者的名称来认可或推广从本软件衍生的产品。
 *
 * 此软件由董事会和贡献者“按原样”提供
 * 任何明示或暗示的担保，包括但不限于
 * 对适销性和特定用途的暗示担保
 * 不承担责任。在任何情况下，董事会或贡献者都不承担责任
 * 由使用本软件引起的任何直接、间接、附带、特殊、惩罚性或后果性的损害
 * （包括但不限于替代商品的采购
 * 或服务；使用、数据或利润的损失；或业务中断）
 * 无论是因合同、严格责任还是侵权（包括疏忽或其他方式）引起的
 * 出于使用本软件的可能性而产生的任何责任理论。
 */
#include <string.h>
#include <stdio.h>

#if defined(LIBC_SCCS) && !defined(lint)
#if 0
static char sccsid[] = "@(#)strsep.c    8.1 (Berkeley) 6/4/93";
#else
static char *rcsid = "$OpenBSD: strsep.c,v 1.3 1997/08/20 04:28:14 millert Exp $";
#endif
#endif /* LIBC_SCCS and not lint */

/*
 * 从字符串 *stringp 中获取下一个标记，其中标记是由分隔符 delim 分隔的可能为空的字符串。
 *
 * 在 *stringp 中写入 NUL 以结束标记。
 * delim 在每次调用时都不需要保持不变。
 * 返回时，*stringp 指向最后一个写入的 NUL（如果可能还有更多标记），或者为 NULL（如果绝对没有更多标记）。
 *
 * 如果 *stringp 为 NULL，则 strsep 返回 NULL。
 */
char *
strsep(stringp, delim)
    register char **stringp;
    register const char *delim;
{
    register char *s;
    register const char *spanp;
    register int c, sc;
    char *tok;

    if ((s = *stringp) == NULL)
        return (NULL);
    for (tok = s;;) {
        c = *s++;
        spanp = delim;
        do {
            if ((sc = *spanp++) == c) {
                if (c == 0)
                    s = NULL;
                else
                    s[-1] = 0;
                *stringp = s;
                return (tok);
            }
        } while (sc != 0);
    }
    /* NOTREACHED */
}
```