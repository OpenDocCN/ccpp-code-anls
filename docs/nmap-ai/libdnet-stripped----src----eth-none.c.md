# `nmap\libdnet-stripped\src\eth-none.c`

```
/*
 * eth-none.c
 *
 * 版权所有 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: eth-none.c 547 2005-01-25 21:30:40Z dugsong $
 */

#include "config.h"

#include <sys/types.h>

#include <errno.h>
#include <stdio.h>
#include <stdlib.h>

#include "dnet.h"

// 打开以太网设备，但是此处仅返回错误
eth_t *
eth_open(const char *device)
{
    // 设置错误码为 ENOSYS
    errno = ENOSYS;
    // 返回空指针
    return (NULL);
}

// 发送数据到以太网设备，但是此处仅返回错误
ssize_t
eth_send(eth_t *e, const void *buf, size_t len)
{
    // 设置错误码为 ENOSYS
    errno = ENOSYS;
    // 返回 -1 表示发送失败
    return (-1);
}

// 关闭以太网设备，但是此处仅返回错误
eth_t *
eth_close(eth_t *e)
{
    // 返回空指针
    return (NULL);
}

// 获取以太网设备的地址，但是此处仅返回错误
int
eth_get(eth_t *e, eth_addr_t *ea)
{
    // 设置错误码为 ENOSYS
    errno = ENOSYS;
    // 返回 -1 表示获取失败
    return (-1);
}

// 设置以太网设备的地址，但是此处仅返回错误
int
eth_set(eth_t *e, const eth_addr_t *ea)
{
    // 设置错误码为 ENOSYS
    errno = ENOSYS;
    // 返回 -1 表示设置失败
    return (-1);
}
```