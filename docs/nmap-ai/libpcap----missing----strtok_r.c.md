# `nmap\libpcap\missing\strtok_r.c`

```
#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

// 如果有配置文件，则包含配置文件


#include "portability.h"

// 包含可移植性文件


char *
pcap_strtok_r(char *s, const char *delim, char **last)
{
    char *spanp, *tok;

// 定义函数 pcap_strtok_r，接受字符串 s、分隔符 delim 和上一次调用的位置 last 作为参数，返回分割后的字符串
    # 定义整型变量c和sc
    int c, sc;

    # 如果s为NULL并且s指向的内容也为NULL，则返回NULL
    if (s == NULL && (s = *last) == NULL)
        return (NULL);

    '''
     * 跳过（跨过）前导分隔符（s += strspn(s, delim)，类似的操作）。
     '''
# 从字符串中提取下一个标记
cont:
    # 从字符串中取出一个字符
    c = *s++;
    # 遍历分隔符列表，如果当前字符是分隔符之一，则跳转到cont标签
    for (spanp = (char *)delim; (sc = *spanp++) != 0;) {
        if (c == sc)
            goto cont;
    }

    # 如果当前字符是空字符，则表示没有非分隔符字符
    if (c == 0) {        /* no non-delimiter characters */
        *last = NULL;
        return (NULL);
    }
    # 记录当前标记的起始位置
    tok = s - 1;

    # 扫描标记（查找分隔符：s += strcspn(s, delim)，类似的操作）。
    # 注意：delim必须有一个NUL；如果遇到NUL字符，也会停止扫描。
    for (;;) {
        # 取出下一个字符
        c = *s++;
        spanp = (char *)delim;
        do {
            # 如果当前字符是分隔符，则将当前位置设为NUL，记录下一个标记的起始位置，并返回当前标记
            if ((sc = *spanp++) == c) {
                if (c == 0)
                    s = NULL;
                else
                    s[-1] = '\0';
                *last = s;
                return (tok);
            }
        } while (sc != 0);
    }
    # 不会执行到这里
}
```