# `nmap\libdnet-stripped\src\fw-none.c`

```
/*
 * fw-none.c
 * 
 * 版权所有 (c) 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: fw-none.c 208 2002-01-20 21:23:28Z dugsong $
 */

#include "config.h"

#include <errno.h>
#include <stdio.h>
#include <stdlib.h>

#include "dnet.h"

// 打开防火墙，但是此处什么也不做，直接返回空指针
fw_t *
fw_open(void)
{
    // 设置错误码为不支持的系统调用
    errno = ENOSYS;
    return (NULL);
}

// 向防火墙添加规则，但是此处什么也不做，直接返回-1
int
fw_add(fw_t *f, const struct fw_rule *rule)
{
    // 设置错误码为不支持的系统调用
    errno = ENOSYS;
    return (-1);
}

// 从防火墙删除规则，但是此处什么也不做，直接返回-1
int
fw_delete(fw_t *f, const struct fw_rule *rule)
{
    // 设置错误码为不支持的系统调用
    errno = ENOSYS;
    return (-1);
}

// 循环处理防火墙规则，但是此处什么也不做，直接返回-1
int
fw_loop(fw_t *f, fw_handler callback, void *arg)
{
    // 设置错误码为不支持的系统调用
    errno = ENOSYS;
    return (-1);
}

// 关闭防火墙，但是此处什么也不做，直接返回空指针
fw_t *
fw_close(fw_t *f)
{
    return (NULL);
}
```