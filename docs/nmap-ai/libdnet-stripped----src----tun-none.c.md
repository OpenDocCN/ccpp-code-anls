# `nmap\libdnet-stripped\src\tun-none.c`

```cpp
/*
 * tun-none.c
 *
 * 版权所有 (c) 2001 Dug Song <dugsong@monkey.org>
 *
 * $Id: tun-none.c 548 2005-01-30 06:01:57Z dugsong $
 */

#include "config.h"

#include <sys/types.h>

#include <errno.h>
#include <stdio.h>
#include <stdlib.h>

#include "dnet.h"

// 打开一个虚拟网络接口，但是此处仅返回错误
tun_t *
tun_open(struct addr *src, struct addr *dst, int mtu)
{
    // 设置错误码为 ENOSYS
    errno = ENOSYS;
    // 返回空指针
    return (NULL);
}

// 获取虚拟网络接口的名称，但是此处仅返回错误
const char *
tun_name(tun_t *tun)
{
    // 设置错误码为 ENOSYS
    errno = ENOSYS;
    // 返回空指针
    return (NULL);
}

// 获取虚拟网络接口的文件描述符，但是此处仅返回错误
int
tun_fileno(tun_t *tun)
{
    // 设置错误码为 ENOSYS
    errno = ENOSYS;
    // 返回 -1
    return (-1);
}

// 发送数据到虚拟网络接口，但是此处仅返回错误
ssize_t
tun_send(tun_t *tun, const void *buf, size_t size)
{
    // 设置错误码为 ENOSYS
    errno = ENOSYS;
    // 返回 -1
    return (-1);
}

// 从虚拟网络接口接收数据，但是此处仅返回错误
ssize_t
tun_recv(tun_t *tun, void *buf, size_t size)
{
    // 设置错误码为 ENOSYS
    errno = ENOSYS;
    // 返回 -1
    return (-1);
}

// 关闭虚拟网络接口，但是此处仅返回空指针
tun_t *
tun_close(tun_t *tun)
{
    // 返回空指针
    return (NULL);
}
```