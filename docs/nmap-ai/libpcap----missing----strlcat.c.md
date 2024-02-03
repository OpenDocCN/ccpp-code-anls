# `nmap\libpcap\missing\strlcat.c`

```cpp
/*    $OpenBSD: pcap_strlcat.c,v 1.15 2015/03/02 21:41:08 millert Exp $    */
# 包含版本信息的注释

/*
 * Copyright (c) 1998, 2015 Todd C. Miller <Todd.Miller@courtesan.com>
 *
 * Permission to use, copy, modify, and distribute this software for any
 * purpose with or without fee is hereby granted, provided that the above
 * copyright notice and this permission notice appear in all copies.
 *
 * THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES
 * WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF
 * MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR
 * ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES
 * WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN
 * ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF
 * OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.
 */
# 版权声明和许可信息

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif
# 如果有配置文件，则包含配置文件

#include <stddef.h>
#include <string.h>
# 包含标准库头文件

#include "portability.h"
# 包含自定义的可移植性头文件

/*
 * Appends src to string dst of size dsize (unlike strncat, dsize is the
 * full size of dst, not space left).  At most dsize-1 characters
 * will be copied.  Always NUL terminates (unless dsize <= strlen(dst)).
 * Returns strlen(src) + MIN(dsize, strlen(initial dst)).
 * If retval >= dsize, truncation occurred.
 */
# 函数注释，说明了函数的作用和返回值

size_t
pcap_strlcat(char * restrict dst, const char * restrict src, size_t dsize)
{
    const char *odst = dst;
    const char *osrc = src;
    size_t n = dsize;
    size_t dlen;
    # 声明变量

    /* Find the end of dst and adjust bytes left but don't go past end. */
    while (n-- != 0 && *dst != '\0')
        dst++;
    dlen = dst - odst;
    n = dsize - dlen;
    # 查找目标字符串的末尾并调整剩余的字节数

    if (n-- == 0)
        return(dlen + strlen(src));
    while (*src != '\0') {
        if (n != 0) {
            *dst++ = *src;
            n--;
        }
        src++;
    }
    *dst = '\0';
    # 如果剩余的字节数为0，则返回目标字符串长度加上源字符串长度；否则将源字符串拼接到目标字符串末尾

    return(dlen + (src - osrc));    /* count does not include NUL */
    # 返回拼接后的字符串长度
}
```