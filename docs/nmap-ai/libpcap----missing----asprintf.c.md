# `nmap\libpcap\missing\asprintf.c`

```
#include <stdio.h>  // 包含标准输入输出库
#include <stdlib.h>  // 包含标准库
#include <stdarg.h>  // 包含可变参数列表的库

#include "portability.h"  // 包含自定义的可移植性库

/*
 * vasprintf() and asprintf() for platforms with a C99-compliant
 * snprintf() - so that, if you format into a 1-byte buffer, it
 * will return how many characters it would have produced had
 * it been given an infinite-sized buffer.
 */
int
pcap_vasprintf(char **strp, const char *format, va_list args)
{
    char buf;  // 定义一个字符型变量buf
    int len;  // 定义一个整型变量len
    size_t str_size;  // 定义一个大小为size_t的变量str_size
    char *str;  // 定义一个字符指针变量str
    int ret;  // 定义一个整型变量ret
    /*
     * XXX - C99标准在第7.19.6.5节中说：“nprintf函数”：
     *
     *    snprintf函数等同于fprintf，不同之处在于输出被写入一个数组（由参数s指定）而不是流中。如果n为零，则不写入任何内容，s可以是空指针。否则，超过第n-1个字符的输出将被丢弃而不是被写入数组，并且在实际写入数组的字符末尾写入一个空字符。
     *
     *        ...
     *
     *    snprintf函数返回将被写入的字符数，如果n足够大，则不计入终止空字符，或者如果发生编码错误则返回负值。因此，只有当返回值为非负且小于n时，才完全写入了以空字符结尾的输出。
     *
     * 这并没有完全说明，如果传递了空缓冲区指针和零计数，它是否会返回将被写入的字符数。
     *
     * 而且，即使C99 *实际上*说它必须工作，例如在Solaris 8中它并不工作 - 对于NULL/0，它返回-1，但对于1字节缓冲区返回正确的字符计数。
     *
     * 因此，我们传递一个一字符指针，以便找出这个格式和这些参数需要多少字符，而不会生成比我们需要的更多的字符。
     *
     * （它可能会在GNU libc或各种BSD libc中工作，但这完全不重要，因为这些通常已经有asprintf()，因此甚至*不需要*这段代码；这是为了在那些没有asprintf()的UN*X中使用。）
     */
    # 使用给定的格式和参数将格式化的字符串写入缓冲区，并返回写入的字符数
    len = vsnprintf(&buf, sizeof buf, format, args);
    # 如果返回-1，表示出错，将指针指向NULL，并返回-1
    if (len == -1) {
        *strp = NULL;
        return (-1);
    }
    # 计算字符串的大小，包括结尾的空字符
    str_size = len + 1;
    # 分配足够大小的内存来存储字符串
    str = malloc(str_size);
    # 如果分配内存失败，将指针指向NULL，并返回-1
    if (str == NULL) {
        *strp = NULL;
        return (-1);
    }
    # 将格式化的字符串写入分配的内存中，并返回写入的字符数
    ret = vsnprintf(str, str_size, format, args);
    # 如果返回-1，表示出错，释放内存，将指针指向NULL，并返回-1
    if (ret == -1) {
        free(str);
        *strp = NULL;
        return (-1);
    }
    # 将指针指向新分配的内存
    *strp = str;
    '''
     * vsnprintf() shouldn't truncate the string, as we have
     * allocated a buffer large enough to hold the string, so its
     * return value should be the number of characters written.
     '''
    # 返回写入的字符数
    return (ret);
# 结束当前函数的定义
}

# 定义一个返回整数类型的函数，函数名为 pcap_asprintf，参数包括一个指向指针的指针和一个常量字符串格式化参数
int
pcap_asprintf(char **strp, const char *format, ...)
{
    # 声明一个变量参数列表
    va_list args;
    # 声明一个整数类型的变量 ret
    int ret;

    # 初始化变量参数列表
    va_start(args, format);
    # 调用 pcap_vasprintf 函数，将结果赋值给 ret
    ret = pcap_vasprintf(strp, format, args);
    # 结束变量参数列表
    va_end(args);
    # 返回 ret 的值
    return (ret);
}
```