# `nmap\libdnet-stripped\src\route-none.c`

```cpp
/*
 * route-none.c
 *
 * Copyright (c) 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: route-none.c 260 2002-02-04 04:03:45Z dugsong $
 */

#include "config.h"

#include <sys/types.h>

#include <errno.h>
#include <stdio.h>
#include <stdlib.h>

#include "dnet.h"

// 打开路由表，但是此函数未实现，直接返回错误
route_t *
route_open(void)
{
    errno = ENOSYS; // 设置错误码为不支持的系统调用
    return (NULL); // 返回空指针
}

// 向路由表中添加路由条目，但是此函数未实现，直接返回错误
int
route_add(route_t *r, const struct route_entry *entry)
{
    errno = ENOSYS; // 设置错误码为不支持的系统调用
    return (-1); // 返回-1表示失败
}

// 从路由表中删除路由条目，但是此函数未实现，直接返回错误
int
route_delete(route_t *r, const struct route_entry *entry)
{
    errno = ENOSYS; // 设置错误码为不支持的系统调用
    return (-1); // 返回-1表示失败
}

// 从路由表中获取路由条目，但是此函数未实现，直接返回错误
int
route_get(route_t *r, struct route_entry *entry)
{
    errno = ENOSYS; // 设置错误码为不支持的系统调用
    return (-1); // 返回-1表示失败
}

// 遍历路由表中的路由条目，但是此函数未实现，直接返回错误
int
route_loop(route_t *r, route_handler callback, void *arg)
{
    errno = ENOSYS; // 设置错误码为不支持的系统调用
    return (-1); // 返回-1表示失败
}

// 关闭路由表，直接返回空指针
route_t *
route_close(route_t *r)
{
    return (NULL); // 返回空指针
}
```