# `nmap\libpcap\missing\strlcpy.c`

```cpp
/*    $OpenBSD: pcap_strlcpy.c,v 1.12 2015/01/15 03:54:12 millert Exp $    */
# 包含文件版本信息

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
# 版权声明和许可声明

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
 * Copy string src to buffer dst of size dsize.  At most dsize-1
 * chars will be copied.  Always NUL terminates (unless dsize == 0).
 * Returns strlen(src); if retval >= dsize, truncation occurred.
 */
size_t
pcap_strlcpy(char * restrict dst, const char * restrict src, size_t dsize)
{
    const char *osrc = src;
    size_t nleft = dsize;
    # 定义变量

    /* Copy as many bytes as will fit. */
    if (nleft != 0) {
        while (--nleft != 0) {
            if ((*dst++ = *src++) == '\0')
                break;
        }
    }
    # 复制尽可能多的字节

    /* Not enough room in dst, add NUL and traverse rest of src. */
    if (nleft == 0) {
        if (dsize != 0)
            *dst = '\0';        /* NUL-terminate dst */
        while (*src++)
            ;
    }
    # 如果目标缓冲区空间不足，添加NUL并遍历源字符串

    return(src - osrc - 1);    /* count does not include NUL */
    # 返回源字符串长度，不包括NUL
}
# 函数结束
```