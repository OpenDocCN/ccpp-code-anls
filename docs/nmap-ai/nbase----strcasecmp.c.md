# `nmap\nbase\strcasecmp.c`

```cpp
/* $Id$ */
// 如果没有定义 HAVE_STRCASECMP 或者 HAVE_STRNCASECMP，则包含必要的头文件和声明
#if !defined(HAVE_STRCASECMP) || !defined(HAVE_STRNCASECMP)
#include <stdlib.h>
#include <string.h>
#include "nbase.h"
#endif

// 如果没有定义 HAVE_STRCASECMP，则定义 strcasecmp 函数
#ifndef HAVE_STRCASECMP
int strcasecmp(const char *s1, const char *s2)
{
    int i, ret;
    char *cp1, *cp2;

    // 分配内存并复制字符串 s1 和 s2
    cp1 = safe_malloc(strlen(s1) + 1);
    cp2 = safe_malloc(strlen(s2) + 1);

    // 将字符串转换为小写
    for (i = 0; i < strlen(s1) + 1; i++)
        cp1[i] = tolower((int) (unsigned char) s1[i]);
    for (i = 0; i < strlen(s2) + 1; i++)
        cp2[i] = tolower((int) (unsigned char) s2[i]);

    // 比较转换后的字符串
    ret = strcmp(cp1, cp2);

    // 释放内存
    free(cp1);
    free(cp2);

    return ret;
}
#endif

// 如果没有定义 HAVE_STRNCASECMP，则定义 strncasecmp 函数
#ifndef HAVE_STRNCASECMP
int strncasecmp(const char *s1, const char *s2, size_t n)
{
    int i, ret;
    char *cp1, *cp2;

    // 分配内存并复制字符串 s1 和 s2
    cp1 = safe_malloc(strlen(s1) + 1);
    cp2 = safe_malloc(strlen(s2) + 1);

    // 将字符串转换为小写
    for (i = 0; i < strlen(s1) + 1; i++)
        cp1[i] = tolower((int) (unsigned char) s1[i]);
    for (i = 0; i < strlen(s2) + 1; i++)
        cp2[i] = tolower((int) (unsigned char) s2[i]);

    // 比较转换后的字符串的前 n 个字符
    ret = strncmp(cp1, cp2, n);

    // 释放内存
    free(cp1);
    free(cp2);

    return ret;
}
#endif
```