# `nmap\libpcap\missing\getopt.c`

```
/*
 * 版权声明，版权归加利福尼亚大学理事会所有
 * 允许在源代码和二进制形式下进行再发布和使用，无论是否经过修改
 * 需满足以下条件：
 * 1. 源代码的再发布必须保留上述版权声明、条件列表和以下免责声明
 * 2. 二进制形式的再发布必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 * 3. 所有宣传材料提及本软件的特性或使用必须显示以下声明：
 *    本产品包括由加利福尼亚大学伯克利分校及其贡献者开发的软件
 * 4. 未经特定事先书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品
 *
 * 本软件由理事会和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保
 * 在任何情况下，理事会或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性的损害承担责任
 * 包括但不限于替代商品或服务的采购、使用损失、数据或利润的损失、业务中断等
 * 无论是合同责任、严格责任还是侵权行为（包括疏忽或其他方式）引起的任何责任，即使已被告知可能发生此类损害
 */

#if defined(LIBC_SCCS) && !defined(lint)
static char sccsid[] = "@(#)getopt.c    8.3 (Berkeley) 4/27/95";
#endif /* LIBC_SCCS and not lint */
#include <stdio.h>  // 包含标准输入输出库
#include <stdlib.h>  // 包含标准库
#include <string.h>  // 包含字符串处理库

#include "getopt.h"  // 包含自定义的 getopt 库

int    opterr = 1,        /* if error message should be printed */
    optind = 1,        /* index into parent argv vector */
    optopt,            /* character checked for validity */
    optreset;        /* reset getopt */
char    *optarg;        /* argument associated with option */

#define    BADCH    (int)'?'  // 定义无效选项字符
#define    BADARG    (int)':'  // 定义缺少参数错误
#define    EMSG    ""  // 空字符串

/*
 * getopt --
 *    Parse argc/argv argument vector.
 */
int
getopt(int nargc, char * const *nargv, const char *ostr)
{
    char *cp;
    static char *__progname;
    static char *place = EMSG;        /* option letter processing */
    char *oli;                /* option letter list index */

    if (__progname == NULL) {
        if ((cp = strrchr(nargv[0], '/')) != NULL)
            __progname = cp + 1;
        else
            __progname = nargv[0];
    }
    if (optreset || !*place) {        /* update scanning pointer */
        optreset = 0;
        if (optind >= nargc || *(place = nargv[optind]) != '-') {
            place = EMSG;
            return (-1);
        }
        if (place[1] && *++place == '-') {    /* found "--" */
            ++optind;
            place = EMSG;
            return (-1);
        }
    }
    optopt = (int)*place++;
    if (optopt == (int)':') {        /* option letter okay? */
        if (!*place)
            ++optind;
        if (opterr && *ostr != ':')
            (void)fprintf(stderr,
                "%s: illegal option -- %c\n", __progname, optopt);
        return (BADCH);
    }
    oli = strchr(ostr, optopt);
    if (!oli) {
        /*
         * 如果用户没有指定'-'作为选项，
         * 假设它意味着-1。
         */
        if (optopt == (int)'-')
            return (-1);
        if (!*place)
            ++optind;
        if (opterr && *ostr != ':')
            (void)fprintf(stderr,
                "%s: illegal option -- %c\n", __progname, optopt);
        return (BADCH);
    }
    if (*++oli != ':') {            /* 不需要参数 */
        optarg = NULL;
        if (!*place)
            ++optind;
    }
    else {                    /* 需要一个参数 */
        if (*place)            /* 没有空白 */
            optarg = place;
        else if (nargc <= ++optind) {    /* 没有参数 */
            place = EMSG;
            if (*ostr == ':')
                return (BADARG);
            if (opterr)
                (void)fprintf(stderr,
                    "%s: option requires an argument -- %c\n",
                    __progname, optopt);
            return (BADCH);
        }
        else                /* 有空白 */
            optarg = nargv[optind];
        place = EMSG;
        ++optind;
    }
    return (optopt);            /* 返回选项字母 */
# 闭合函数或代码块的结束标志
```