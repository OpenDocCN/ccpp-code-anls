# `nmap\libpcap\bpf_dump.c`

```
/*
 * 版权声明
 * 版权所有 (c) 1992, 1993, 1994, 1995, 1996
 * 加利福尼亚大学理事会。保留所有权利。
 *
 * 在源代码和二进制形式下的再发布和使用，无论是否经过修改，都是允许的，前提是：(1) 源代码发布
 * 保留上述版权声明和本段文字的完整性，(2) 包含二进制代码的发布在文档或其他提供的材料中保留上述版权声明
 * 和本段文字的完整性，(3) 所有提及此软件特性或使用的广告材料都显示以下声明：
 * “本产品包括由加利福尼亚大学劳伦斯伯克利实验室及其贡献者开发的软件。”未经事先书面许可，不得使用
 * 大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品。
 * 本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <pcap.h>
#include <stdio.h>

#include "optimize.h"

void
bpf_dump(const struct bpf_program *p, int option)
{
    const struct bpf_insn *insn;
    int i;
    int n = p->bf_len;

    insn = p->bf_insns;
    if (option > 2) {
        printf("%d\n", n);
        for (i = 0; i < n; ++insn, ++i) {
            printf("%u %u %u %u\n", insn->code,
                   insn->jt, insn->jf, insn->k);
        }
        return ;
    }
    if (option > 1) {
        for (i = 0; i < n; ++insn, ++i)
            printf("{ 0x%x, %d, %d, 0x%08x },\n",
                   insn->code, insn->jt, insn->jf, insn->k);
        return;
    }
    for (i = 0; i < n; ++insn, ++i) {
#ifdef BDEBUG
        // 如果定义了 BDEBUG 并且当前索引小于 NBIDS 并且 bids[i] 大于 0，则打印方括号和对应的值减一
        if (i < NBIDS && bids[i] > 0)
            printf("[%02d]", bids[i] - 1);
        // 否则打印双短横线
        else
            printf(" -- ");
#endif
        // 打印基于指令和索引的图像
        puts(bpf_image(insn, i));
    }
}
```