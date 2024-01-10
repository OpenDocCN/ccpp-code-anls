# `nmap\libdnet-stripped\src\eth-ndd.c`

```
/*
 * eth-ndd.c
 *
 * 版权所有 (c) 2001 Dug Song <dugsong@monkey.org>
 *
 * $Id: eth-ndd.c 547 2005-01-25 21:30:40Z dugsong $
 */

#include "config.h"

#include <sys/types.h>
#include <sys/socket.h>
#include <sys/ndd_var.h>
#include <sys/kinfo.h>

#include <assert.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#include "dnet.h"

struct eth_handle {
    char    device[16];  // 以太网设备名称
    int    fd;  // 文件描述符
};

eth_t *
eth_open(const char *device)
{
    struct sockaddr_ndd_8022 sa;  // NDD 802.2 协议的套接字地址结构
    eth_t *e;  // 以太网句柄
    
    if ((e = calloc(1, sizeof(*e))) == NULL)  // 分配以太网句柄内存
        return (NULL);

    if ((e->fd = socket(AF_NDD, SOCK_DGRAM, NDD_PROT_ETHER)) < 0)  // 创建以太网套接字
        return (eth_close(e));
    
    sa.sndd_8022_family = AF_NDD;  // 设置套接字地址结构的协议族
    sa.sndd_8022_len = sizeof(sa);  // 设置套接字地址结构的长度
    sa.sndd_8022_filtertype = NS_ETHERTYPE;  // 设置过滤类型为以太网类型
    sa.sndd_8022_ethertype = 0;  // 设置以太网类型
    sa.sndd_8022_filterlen = sizeof(struct ns_8022);  // 设置过滤器长度
    strlcpy(sa.sndd_8022_nddname, device, sizeof(sa.sndd_8022_nddname));  // 设置以太网设备名称
    
    if (bind(e->fd, (struct sockaddr *)&sa, sizeof(sa)) < 0)  // 将套接字绑定到指定地址
        return (eth_close(e));
    
    if (connect(e->fd, (struct sockaddr *)&sa, sizeof(sa)) < 0)  // 将套接字连接到指定地址
        return (eth_close(e));
    
    /* XXX - SO_BROADCAST needed? */  // 是否需要广播
    
    return (e);  // 返回以太网句柄
}

ssize_t
eth_send(eth_t *e, const void *buf, size_t len)
{
    return (write(e->fd, buf, len));  // 发送数据到以太网
}

eth_t *
eth_close(eth_t *e)
{
    if (e != NULL) {
        if (e->fd >= 0)
            close(e->fd);  // 关闭套接字
        free(e);  // 释放以太网句柄内存
    }
    return (NULL);  // 返回空指针
}

int
eth_get(eth_t *e, eth_addr_t *ea)
{
    struct kinfo_ndd *nddp;  // NDD 内核信息结构
    int size;  // 大小
    void *end;  // 结尾
    
    if ((size = getkerninfo(KINFO_NDD, 0, 0, 0)) == 0) {  // 获取 NDD 内核信息大小
        errno = ENOENT;  // 设置错误码为不存在
        return (-1);  // 返回错误
    } else if (size < 0)
        return (-1);
    
    if ((nddp = malloc(size)) == NULL)  // 分配 NDD 内核信息内存
        return (-1);
                     
    if (getkerninfo(KINFO_NDD, nddp, &size, 0) < 0) {  // 获取 NDD 内核信息
        free(nddp);  // 释放内存
        return (-1);  // 返回错误
    }
    # 遍历指针 nddp 到指定的内存块末尾
    for (end = (void *)nddp + size; (void *)nddp < end; nddp++) {
        # 如果当前 nddp 指向的设备别名与给定设备名相同，或者设备名相同
        if (strcmp(nddp->ndd_alias, e->device) == 0 ||
            strcmp(nddp->ndd_name, e->device) == 0) {
            # 将 nddp 指向的设备地址拷贝到 ea 指向的内存中
            memcpy(ea, nddp->ndd_addr, sizeof(*ea));
        }
    }
    # 释放 nddp 指向的内存块
    free(nddp);
    
    # 如果 nddp 指向的内存块超出了指定的内存块末尾
    if ((void *)nddp >= end) {
        # 设置错误码为 ESRCH
        errno = ESRCH;
        # 返回 -1
        return (-1);
    }
    # 返回 0
    return (0);
# 结束函数定义
}

# 设置以太网地址的函数
int
eth_set(eth_t *e, const eth_addr_t *ea)
{
    # 设置错误码为不支持的系统调用
    errno = ENOSYS;
    # 返回-1表示设置失败
    return (-1);
}
```