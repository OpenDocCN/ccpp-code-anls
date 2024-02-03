# `nmap\libdnet-stripped\src\eth-bsd.c`

```cpp
/*
 * eth-bsd.c
 *
 * 版权所有 (c) 2001 Dug Song <dugsong@monkey.org>
 *
 * $Id: eth-bsd.c 630 2006-02-02 04:17:39Z dugsong $
 */

#include "config.h"

#include <sys/param.h>
#include <sys/types.h>
#include <sys/ioctl.h>
#include <sys/socket.h>
#include <sys/time.h>
#if defined(HAVE_SYS_SYSCTL_H) && defined(HAVE_ROUTE_RT_MSGHDR)
#include <sys/sysctl.h>
#include <net/route.h>
#include <net/if_dl.h>
#endif
#include <net/bpf.h>
#include <net/if.h>

#include <assert.h>
#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#include "dnet.h"

// 定义一个结构体用于存储以太网句柄
struct eth_handle {
    int    fd; // 文件描述符
    char    device[16]; // 设备名称
};

// 打开以太网设备
eth_t *
eth_open(const char *device)
{
    struct ifreq ifr; // 网络接口请求结构体
    char file[32]; // 文件名
    eth_t *e; // 以太网句柄
    int i;

    // 分配以太网句柄内存
    if ((e = calloc(1, sizeof(*e))) != NULL) {
        for (i = 0; i < 128; i++) {
            snprintf(file, sizeof(file), "/dev/bpf%d", i); // 格式化文件名
            /* This would be O_WRONLY, but Mac OS X 10.6 has a bug
               where that prevents other users of the interface
               from seeing incoming traffic, even in other
               processes. */
            e->fd = open(file, O_RDWR); // 以读写方式打开文件
            if (e->fd != -1 || errno != EBUSY) // 如果文件打开成功或者错误码不是设备忙
                break;
        }
        if (e->fd < 0) // 如果文件描述符小于0
            return (eth_close(e)); // 关闭以太网句柄
        
        memset(&ifr, 0, sizeof(ifr)); // 将 ifr 结构体清零
        strlcpy(ifr.ifr_name, device, sizeof(ifr.ifr_name)); // 将设备名称拷贝到 ifr 结构体中
        
        if (ioctl(e->fd, BIOCSETIF, (char *)&ifr) < 0) // 设置 BPF 设备的接口
            return (eth_close(e)); // 关闭以太网句柄
#ifdef BIOCSHDRCMPLT
        i = 1;
        if (ioctl(e->fd, BIOCSHDRCMPLT, &i) < 0) // 设置 BPF 设备的头部完整性
            return (eth_close(e)); // 关闭以太网句柄
#endif
        strlcpy(e->device, device, sizeof(e->device)); // 将设备名称拷贝到以太网句柄中
    }
    return (e); // 返回以太网句柄
}

// 发送数据
ssize_t
eth_send(eth_t *e, const void *buf, size_t len)
{
    return (write(e->fd, buf, len)); // 写入数据到文件描述符
}

// 关闭以太网句柄
eth_t *
eth_close(eth_t *e)
{
    if (e != NULL) {
        if (e->fd >= 0) // 如果文件描述符大于等于0
            close(e->fd); // 关闭文件描述符
        free(e); // 释放以太网句柄内存
    }
    return (NULL); // 返回空指针
}
#if defined(HAVE_SYS_SYSCTL_H) && defined(HAVE_ROUTE_RT_MSGHDR)
# 如果定义了 HAVE_SYS_SYSCTL_H 和 HAVE_ROUTE_RT_MSGHDR，则执行以下代码块
int
eth_get(eth_t *e, eth_addr_t *ea)
{
    # 定义各种需要使用的结构体和变量
    struct if_msghdr *ifm;
    struct sockaddr_dl *sdl;
    struct addr ha;
    u_char *p, *buf;
    size_t len;
    int mib[] = { CTL_NET, AF_ROUTE, 0, AF_LINK, NET_RT_IFLIST, 0 };

    # 获取网络接口信息的长度
    if (sysctl(mib, 6, NULL, &len, NULL, 0) < 0)
        return (-1);
    
    # 分配内存空间
    if ((buf = malloc(len)) == NULL)
        return (-1);
    
    # 获取网络接口信息
    if (sysctl(mib, 6, buf, &len, NULL, 0) < 0) {
        free(buf);
        return (-1);
    }
    # 遍历网络接口信息
    for (p = buf; p < buf + len; p += ifm->ifm_msglen) {
        ifm = (struct if_msghdr *)p;
        sdl = (struct sockaddr_dl *)(ifm + 1);
        
        # 判断网络接口信息类型和地址类型
        if (ifm->ifm_type != RTM_IFINFO ||
            (ifm->ifm_addrs & RTA_IFP) == 0)
            continue;
        
        # 判断地址家族和地址长度
        if (sdl->sdl_family != AF_LINK || sdl->sdl_nlen == 0 ||
            memcmp(sdl->sdl_data, e->device, sdl->sdl_nlen) != 0)
            continue;
        
        # 将地址转换为网络字节顺序
        if (addr_ston((struct sockaddr *)sdl, &ha) == 0)
            break;
    }
    # 释放内存空间
    free(buf);
    
    # 判断是否找到对应的网络接口信息
    if (p >= buf + len) {
        errno = ESRCH;
        return (-1);
    }
    # 将地址信息拷贝到指定的地址结构体中
    memcpy(ea, &ha.addr_eth, sizeof(*ea));
    
    return (0);
}
#else
# 如果未定义 HAVE_SYS_SYSCTL_H 和 HAVE_ROUTE_RT_MSGHDR，则执行以下代码块
int
eth_get(eth_t *e, eth_addr_t *ea)
{
    # 设置错误码并返回
    errno = ENOSYS;
    return (-1);
}
#endif

#if defined(SIOCSIFLLADDR)
# 如果定义了 SIOCSIFLLADDR，则执行以下代码块
int
eth_set(eth_t *e, const eth_addr_t *ea)
{
    # 定义各种需要使用的结构体和变量
    struct ifreq ifr;
    struct addr ha;

    # 设置地址类型和地址位数
    ha.addr_type = ADDR_TYPE_ETH;
    ha.addr_bits = ETH_ADDR_BITS;
    # 将地址信息拷贝到指定的地址结构体中
    memcpy(&ha.addr_eth, ea, ETH_ADDR_LEN);
    
    # 初始化网络接口请求结构体
    memset(&ifr, 0, sizeof(ifr));
    # 将网络接口名称拷贝到请求结构体中
    strlcpy(ifr.ifr_name, e->device, sizeof(ifr.ifr_name));
    # 将地址信息转换为网络字节顺序
    addr_ntos(&ha, &ifr.ifr_addr);
    
    # 设置网络接口的链路地址
    return (ioctl(e->fd, SIOCSIFLLADDR, &ifr));
}
#else
# 如果未定义 SIOCSIFLLADDR，则执行以下代码块
int
eth_set(eth_t *e, const eth_addr_t *ea)
{
    # 设置错误码并返回
    errno = ENOSYS;
    return (-1);
}
#endif
```