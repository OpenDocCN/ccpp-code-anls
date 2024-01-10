# `nmap\libdnet-stripped\src\eth-pfilt.c`

```
#include "config.h"
// 包含自定义的配置文件

#include <sys/types.h>
#include <sys/time.h>
#include <sys/ioctl.h>
// 包含系统类型、时间和输入输出控制相关的头文件

#include <net/if.h>
#include <net/pfilt.h>
// 包含网络接口和数据包过滤相关的头文件

#include <fcntl.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
// 包含文件控制、标准库、字符串处理和系统调用相关的头文件

#include "dnet.h"
// 包含自定义的网络库头文件

struct eth_handle {
    int    fd;
    int    sock;
    char    device[16];
};
// 定义了一个结构体 eth_handle，包含文件描述符、套接字和设备名称

eth_t *
eth_open(const char *device)
{
    struct eth_handle *e;
    int fd;
    // 定义了一个指向 eth_handle 结构体的指针 e 和一个整型变量 fd

    if ((e = calloc(1, sizeof(*e))) != NULL) {
        // 分配了一个大小为 eth_handle 结构体大小的内存空间，并将其赋值给 e
        strlcpy(e->device, device, sizeof(e->device));
        // 将传入的设备名称拷贝到结构体中的设备名称字段
        if ((e->fd = pfopen(e->device, O_WRONLY)) < 0 ||
            (e->sock = socket(AF_INET, SOCK_DGRAM, 0)) < 0)
            // 如果打开设备失败或者创建套接字失败
            e = eth_close(e);
            // 调用 eth_close 函数关闭设备并释放内存
    }
    return (e);
    // 返回 eth_handle 结构体指针
}

int
eth_get(eth_t *e, eth_addr_t *ea)
{
    struct ifdevea ifd;
    // 定义了一个 ifdevea 结构体

    strlcpy(ifd.ifr_name, e->device, sizeof(ifd.ifr_name));
    // 将结构体中的设备名称字段赋值为传入的设备名称
    if (ioctl(e->sock, SIOCRPHYSADDR, &ifd) < 0)
        // 如果获取物理地址失败
        return (-1);
        // 返回错误
    memcpy(ea, ifd.current_pa, ETH_ADDR_LEN);
    // 将获取到的物理地址拷贝到传入的地址结构体中
    return (0);
    // 返回成功
}

int
eth_set(eth_t *e, const eth_addr_t *ea)
{
    struct ifdevea ifd;
    // 定义了一个 ifdevea 结构体

    strlcpy(ifd.ifr_name, e->device, sizeof(ifd.ifr_name));
    // 将结构体中的设备名称字段赋值为传入的设备名称
    memcpy(ifd.current_pa, ea, ETH_ADDR_LEN);
    // 将传入的地址拷贝到结构体中的当前物理地址字段
    return (ioctl(e->sock, SIOCSPHYSADDR, &ifd));
    // 调用 ioctl 函数设置物理地址
}

ssize_t
eth_send(eth_t *e, const void *buf, size_t len)
{
    return (write(e->fd, buf, len));
    // 调用 write 函数发送数据
}

eth_t *
eth_close(eth_t *e)
{
    if (e != NULL) {
        // 如果传入的指针不为空
        if (e->fd >= 0)
            // 如果文件描述符大于等于 0
            close(e->fd);
            // 关闭文件描述符
        if (e->sock >= 0)
            // 如果套接字大于等于 0
            close(e->sock);
            // 关闭套接字
        free(e);
        // 释放内存
    }
    return (NULL);
    // 返回空指针
}
```