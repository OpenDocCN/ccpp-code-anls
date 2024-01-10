# `nmap\libpcap\missing\win_asprintf.c`

```
#include <stdio.h>
#include <stdlib.h>
#include <stdarg.h>

#include "portability.h"

int
pcap_vasprintf(char **strp, const char *format, va_list args)
{
    int len;  // 用于存储格式化后的字符串长度
    size_t str_size;  // 存储字符串的大小
    char *str;  // 存储格式化后的字符串
    int ret;  // 存储格式化函数的返回值

    len = _vscprintf(format, args);  // 获取格式化后的字符串长度
    if (len == -1) {
        *strp = NULL;  // 如果长度为-1，表示出错，将结果指针置为NULL
        return (-1);  // 返回-1表示出错
    }
    str_size = len + 1;  // 计算字符串大小
    str = malloc(str_size);  // 分配字符串大小的内存空间
    if (str == NULL) {
        *strp = NULL;  // 如果分配内存失败，将结果指针置为NULL
        return (-1);  // 返回-1表示出错
    }
    ret = vsnprintf(str, str_size, format, args);  // 格式化字符串
    if (ret == -1) {
        free(str);  // 如果格式化出错，释放内存
        *strp = NULL;  // 将结果指针置为NULL
        return (-1);  // 返回-1表示出错
    }
    *strp = str;  // 将格式化后的字符串赋值给结果指针
    /*
     * vsnprintf() shouldn't truncate the string, as we have
     * allocated a buffer large enough to hold the string, so its
     * return value should be the number of characters printed.
     */
    return (ret);  // 返回格式化后的字符串长度
}

int
pcap_asprintf(char **strp, const char *format, ...)
{
    va_list args;  // 定义可变参数列表
    int ret;  // 存储格式化函数的返回值

    va_start(args, format);  // 初始化可变参数列表
    ret = pcap_vasprintf(strp, format, args);  // 调用格式化函数
    va_end(args);  // 结束可变参数列表
    return (ret);  // 返回格式化函数的返回值
}
```