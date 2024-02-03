# `nmap\libdnet-stripped\src\strlcpy.c`

```cpp
/*
 * 以下是一个字符串拷贝函数的实现
 * 该函数的作用是将源字符串拷贝到目标字符串中，最多拷贝 siz-1 个字符
 * 并且始终在目标字符串末尾添加 NUL 终止符（除非 siz == 0）
 * 如果返回的拷贝长度大于等于 siz，则表示发生了截断
 */
size_t
strlcpy(dst, src, siz)
    # 定义一个指向字符的指针变量
    char *dst;
    # 定义一个指向常量字符的指针变量
    const char *src;
    # 定义一个表示大小的变量
    size_t siz;
{
    // 定义指向目标字符串的指针
    register char *d = dst;
    // 定义指向源字符串的指针
    register const char *s = src;
    // 定义要复制的最大字节数
    register size_t n = siz;

    /* 复制尽可能多的字节 */
    if (n != 0 && --n != 0) {
        do {
            // 将源字符串的内容复制到目标字符串中，直到遇到空字符或者复制完所有字节
            if ((*d++ = *s++) == 0)
                break;
        } while (--n != 0);
    }

    /* 目标字符串空间不足，添加空字符并继续遍历源字符串 */
    if (n == 0) {
        if (siz != 0)
            *d = '\0';        /* 在目标字符串末尾添加空字符 */
        while (*s++)
            ;
    }

    // 返回复制的字节数（不包括空字符）
    return(s - src - 1);    /* 计数不包括空字符 */
}
```