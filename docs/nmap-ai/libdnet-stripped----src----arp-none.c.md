# `nmap\libdnet-stripped\src\arp-none.c`

```cpp
/*
 * arp-none.c
 *
 * 版权所有 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: arp-none.c 252 2002-02-02 04:15:57Z dugsong $
 */

#include "config.h"

#include <sys/types.h>

#include <errno.h>
#include <stdio.h>
#include <stdlib.h>

#include "dnet.h"

// 打开 ARP 表，但是此函数未实现，返回错误码 ENOSYS
arp_t *
arp_open(void)
{
    errno = ENOSYS;
    return (NULL);
}

// 向 ARP 表中添加条目，但是此函数未实现，返回错误码 ENOSYS
int
arp_add(arp_t *a, const struct arp_entry *entry)
{
    errno = ENOSYS;
    return (-1);
}

// 从 ARP 表中删除条目，但是此函数未实现，返回错误码 ENOSYS
int
arp_delete(arp_t *a, const struct arp_entry *entry)
{
    errno = ENOSYS;
    return (-1);
}

// 从 ARP 表中获取条目，但是此函数未实现，返回错误码 ENOSYS
int
arp_get(arp_t *a, struct arp_entry *entry)
{
    errno = ENOSYS;
    return (-1);
}

// 遍历 ARP 表中的条目，但是此函数未实现，返回错误码 ENOSYS
int
arp_loop(arp_t *a, arp_handler callback, void *arg)
{
    errno = ENOSYS;
    return (-1);
}

// 关闭 ARP 表，返回 NULL
arp_t *
arp_close(arp_t *a)
{
    return (NULL);
}
```