# `nmap\libdnet-stripped\src\eth-dlpi.c`

```
/*
 * eth-dlpi.c
 *
 * Based on Neal Nuckolls' 1992 "How to Use DLPI" paper.
 *
 * Copyright (c) 2001 Dug Song <dugsong@monkey.org>
 *
 * $Id: eth-dlpi.c 560 2005-02-10 16:48:36Z dugsong $
 */

#include "config.h"

#include <sys/types.h>
#ifdef HAVE_SYS_BUFMOD_H
#include <sys/bufmod.h>
#endif
#ifdef HAVE_SYS_DLPI_H
#include <sys/dlpi.h>
#elif defined(HAVE_SYS_DLPIHDR_H)
#include <sys/dlpihdr.h>
#endif
#ifdef HAVE_SYS_DLPI_EXT_H
#include <sys/dlpi_ext.h>
#endif
#include <sys/stream.h>

#include <assert.h>
#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stropts.h>
#include <unistd.h>

#include "dnet.h"

#ifndef INFTIM
#define INFTIM    -1
#endif

struct eth_handle {
    int    fd;
    int    sap_len;
};

static int
dlpi_msg(int fd, union DL_primitives *dlp, int rlen, int flags,
    int ack, int alen, int size)
{
    struct strbuf ctl;

    ctl.maxlen = 0;
    ctl.len = rlen;
    ctl.buf = (caddr_t)dlp;
    
    if (putmsg(fd, &ctl, NULL, flags) < 0)
        return (-1);
    
    ctl.maxlen = size;
    ctl.len = 0;
    
    flags = 0;

    if (getmsg(fd, &ctl, NULL, &flags) < 0)
        return (-1);
    
    if (dlp->dl_primitive != ack || ctl.len < alen)
        return (-1);
    
    return (0);
}

#if defined(DLIOCRAW) || defined(HAVE_SYS_DLPIHDR_H)
static int
strioctl(int fd, int cmd, int len, char *dp)
{
    struct strioctl str;
    
    str.ic_cmd = cmd;
    str.ic_timout = INFTIM;
    str.ic_len = len;
    str.ic_dp = dp;
    
    if (ioctl(fd, I_STR, &str) < 0)
        return (-1);
    
    return (str.ic_len);
}
#endif

#ifdef HAVE_SYS_DLPIHDR_H
/* XXX - OSF1 is nuts */
#define ND_GET    ('N' << 8 + 0)

static int
eth_match_ppa(eth_t *e, const char *device)
{
    char *p, dev[16], buf[256];
    int len, ppa;

    strlcpy(buf, "dl_ifnames", sizeof(buf));
    
    if ((len = strioctl(e->fd, ND_GET, sizeof(buf), buf)) < 0)
        return (-1);
    # 从缓冲区开始遍历，直到遍历完整个缓冲区
    for (p = buf; p < buf + len; p += strlen(p) + 1) {
        # 初始化ppa为-1
        ppa = -1;
        # 从当前位置p开始，按照指定格式读取字符串和整数，如果读取的数量不是2，则跳出循环
        if (sscanf(p, "%s (PPA %d)\n", dev, &ppa) != 2)
            break;
        # 如果读取的设备名和指定的设备名相同，则跳出循环
        if (strcmp(dev, device) == 0)
            break;
    }
    # 返回ppa的值
    return (ppa);
}
#else
static char *
dev_find_ppa(char *dev)
{
    char *p;

    p = dev + strlen(dev);  # 将指针 p 指向字符串 dev 的末尾
    while (p > dev && strchr("0123456789", *(p - 1)) != NULL)  # 当 p 指向的字符是数字时，向前移动指针 p
        p--;
    if (*p == '\0')  # 如果 p 指向的是字符串结束符，则返回空指针
        return NULL;

    return p;  # 返回指针 p
}
#endif

eth_t *
eth_open(const char *device)
{
    union DL_primitives *dlp;  # 定义 DL_primitives 联合体指针变量 dlp
    uint32_t buf[8192];  # 定义长度为 8192 的 uint32_t 类型数组 buf
    char *p, dev[16];  # 定义字符指针 p 和长度为 16 的字符数组 dev
    eth_t *e;  # 定义 eth_t 结构体指针变量 e
    int ppa;  # 定义整型变量 ppa

    if ((e = calloc(1, sizeof(*e))) == NULL)  # 分配内存给 e，大小为 eth_t 结构体的大小，如果分配失败则返回空指针
        return (NULL);

#ifdef HAVE_SYS_DLPIHDR_H
    if ((e->fd = open("/dev/streams/dlb", O_RDWR)) < 0)  # 打开 /dev/streams/dlb 设备文件，以读写方式打开，如果失败则关闭 e 并返回空指针
        return (eth_close(e));
    
    if ((ppa = eth_match_ppa(e, device)) < 0) {  # 调用 eth_match_ppa 函数，如果返回值小于 0
        errno = ESRCH;  # 设置错误码为 ESRCH
        return (eth_close(e));  # 关闭 e 并返回空指针
    }
#else
    e->fd = -1;  # 将 e 的文件描述符设置为 -1
    snprintf(dev, sizeof(dev), "/dev/%s", device);  # 将设备名格式化为 /dev/ 开头的字符串，存入 dev
    if ((p = dev_find_ppa(dev)) == NULL) {  # 调用 dev_find_ppa 函数，如果返回空指针
        errno = EINVAL;  # 设置错误码为 EINVAL
        return (eth_close(e));  # 关闭 e 并返回空指针
    }
    ppa = atoi(p);  # 将 p 指向的字符串转换为整型数，存入 ppa
    *p = '\0';  # 将 p 指向的位置设置为空字符

    if ((e->fd = open(dev, O_RDWR)) < 0) {  # 以读写方式打开 dev 设备文件，如果失败
        snprintf(dev, sizeof(dev), "/dev/%s", device);  # 格式化设备名为 /dev/ 开头的字符串，存入 dev
        if ((e->fd = open(dev, O_RDWR)) < 0) {  # 以读写方式打开 dev 设备文件，如果失败
            snprintf(dev, sizeof(dev), "/dev/net/%s", device);  # 格式化设备名为 /dev/net/ 开头的字符串，存入 dev
            if ((e->fd = open(dev, O_RDWR)) < 0)  # 以读写方式打开 dev 设备文件，如果失败
                return (eth_close(e));  # 关闭 e 并返回空指针
        }
    }
#endif
    dlp = (union DL_primitives *)buf;  # 将 buf 强制转换为 DL_primitives 联合体指针类型，赋值给 dlp
    dlp->info_req.dl_primitive = DL_INFO_REQ;  # 设置 dlp 指向的联合体中的 info_req 的 dl_primitive 字段为 DL_INFO_REQ
    
    if (dlpi_msg(e->fd, dlp, DL_INFO_REQ_SIZE, RS_HIPRI,  # 调用 dlpi_msg 函数，如果返回值小于 0
        DL_INFO_ACK, DL_INFO_ACK_SIZE, sizeof(buf)) < 0)
        return (eth_close(e));  # 关闭 e 并返回空指针
    
    e->sap_len = dlp->info_ack.dl_sap_length;  # 将 dlp 指向的联合体中的 info_ack 的 dl_sap_length 字段赋值给 e 的 sap_len 字段
    
    if (dlp->info_ack.dl_provider_style == DL_STYLE2) {  # 如果 dlp 指向的联合体中的 info_ack 的 dl_provider_style 字段等于 DL_STYLE2
        dlp->attach_req.dl_primitive = DL_ATTACH_REQ;  # 设置 dlp 指向的联合体中的 attach_req 的 dl_primitive 字段为 DL_ATTACH_REQ
        dlp->attach_req.dl_ppa = ppa;  # 将 ppa 赋值给 dlp 指向的联合体中的 attach_req 的 dl_ppa 字段
        
        if (dlpi_msg(e->fd, dlp, DL_ATTACH_REQ_SIZE, 0,  # 调用 dlpi_msg 函数，如果返回值小于 0
            DL_OK_ACK, DL_OK_ACK_SIZE, sizeof(buf)) < 0)
            return (eth_close(e));  # 关闭 e 并返回空指针
    }
    memset(&dlp->bind_req, 0, DL_BIND_REQ_SIZE);  # 将 dlp 指向的联合体中的 bind_req 字段清零
    dlp->bind_req.dl_primitive = DL_BIND_REQ;  # 设置 dlp 指向的联合体中的 bind_req 的 dl_primitive 字段为 DL_BIND_REQ
#ifdef DL_HP_RAWDLS
    dlp->bind_req.dl_sap = 24;    /* from HP-UX DLPI programmers guide */  # 设置 dlp 指向的联合体中的 bind_req 的 dl_sap 字段为 24
    # 设置dl_service_mode字段为DL_HP_RAWDLS
    dlp->bind_req.dl_service_mode = DL_HP_RAWDLS;
#else
    // 如果条件不成立，设置数据链路层的SAP为DL_ETHER
    dlp->bind_req.dl_sap = DL_ETHER;
    // 设置数据链路层的服务模式为DL_CLDLS
    dlp->bind_req.dl_service_mode = DL_CLDLS;
#endif
    // 如果调用dlpi_msg函数失败，则关闭以太网连接并返回
    if (dlpi_msg(e->fd, dlp, DL_BIND_REQ_SIZE, 0,
        DL_BIND_ACK, DL_BIND_ACK_SIZE, sizeof(buf)) < 0)
        return (eth_close(e));
#ifdef DLIOCRAW
    // 如果支持DLIOCRAW，则设置以太网连接为原始模式
    if (strioctl(e->fd, DLIOCRAW, 0, NULL) < 0)
        return (eth_close(e));
#endif
    // 返回以太网连接
    return (e);
}

ssize_t
eth_send(eth_t *e, const void *buf, size_t len)
{
#if defined(DLIOCRAW)
    // 如果支持DLIOCRAW，则直接写入数据到以太网连接
    return (write(e->fd, buf, len));
#else
    // 定义DL_primitives联合体指针dlp
    union DL_primitives *dlp;
    // 定义strbuf类型的ctl和data
    struct strbuf ctl, data;
    // 定义eth_hdr类型的指针eth
    struct eth_hdr *eth;
    // 定义uint32_t类型的ctlbuf数组
    uint32_t ctlbuf[8192];
    // 定义长度为4的u_char类型的sap数组
    u_char sap[4] = { 0, 0, 0, 0 };
    // 定义整型变量dlen
    int dlen;

    // 将ctlbuf强制转换为DL_primitives类型的指针赋值给dlp
    dlp = (union DL_primitives *)ctlbuf;
#ifdef DL_HP_RAWDATA_REQ
    // 如果支持DL_HP_RAWDATA_REQ，则设置dl_primitive为DL_HP_RAWDATA_REQ，dlen为DL_HP_RAWDATA_REQ_SIZE
    dlp->dl_primitive = DL_HP_RAWDATA_REQ;
    dlen = DL_HP_RAWDATA_REQ_SIZE;
#else
    // 否则设置unitdata_req的相关属性
    dlp->unitdata_req.dl_primitive = DL_UNITDATA_REQ;
    dlp->unitdata_req.dl_dest_addr_length = ETH_ADDR_LEN;
    dlp->unitdata_req.dl_dest_addr_offset = DL_UNITDATA_REQ_SIZE;
    dlp->unitdata_req.dl_priority.dl_min =
        dlp->unitdata_req.dl_priority.dl_max = 0;
    dlen = DL_UNITDATA_REQ_SIZE;
#endif
    // 将buf强制转换为eth_hdr类型的指针赋值给eth
    eth = (struct eth_hdr *)buf;
    // 将eth中的eth_type转换为网络字节顺序，并赋值给sap的前两个字节
    *(uint16_t *)sap = ntohs(eth->eth_type);
    
    /* XXX - DLSAP setup logic from ISC DHCP */
    // 设置ctl的maxlen为0，len为dlen + ETH_ADDR_LEN + e->sap_len的绝对值，buf为ctlbuf的地址
    ctl.maxlen = 0;
    ctl.len = dlen + ETH_ADDR_LEN + abs(e->sap_len);
    ctl.buf = (char *)ctlbuf;
    
    // 根据e->sap_len的正负情况，拷贝sap和eth->eth_dst.data到ctlbuf中
    if (e->sap_len >= 0) {
        memcpy(ctlbuf + dlen, sap, e->sap_len);
        memcpy(ctlbuf + dlen + e->sap_len,
            eth->eth_dst.data, ETH_ADDR_LEN);
    } else {
        memcpy(ctlbuf + dlen, eth->eth_dst.data, ETH_ADDR_LEN);
        memcpy(ctlbuf + dlen + ETH_ADDR_LEN, sap, abs(e->sap_len));
    }
    // 设置data的maxlen为0，len为传入的数据长度，buf为传入的数据buf
    data.maxlen = 0;
    data.len = len;
    data.buf = (char *)buf;

    // 将ctl和data发送到以太网连接，如果失败则返回-1
    if (putmsg(e->fd, &ctl, &data, 0) < 0)
        return (-1);

    // 返回发送的数据长度
    return (len);
#endif
}

eth_t *
eth_close(eth_t *e)
{
    // 如果e不为空，则关闭以太网连接并释放内存
    if (e != NULL) {
        if (e->fd >= 0)
            close(e->fd);
        free(e);
    }
    // 返回空指针
    return (NULL);
}

int
# 获取以太网地址
eth_get(eth_t *e, eth_addr_t *ea)
{
    # 定义一个 DL_primitives 类型的联合体指针
    union DL_primitives *dlp;
    # 定义一个长度为2048的无符号字符数组
    u_char buf[2048];
    
    # 将 buf 转换为 DL_primitives 类型的指针
    dlp = (union DL_primitives *)buf;
    # 设置物理地址请求的数据结构
    dlp->physaddr_req.dl_primitive = DL_PHYS_ADDR_REQ;
    dlp->physaddr_req.dl_addr_type = DL_CURR_PHYS_ADDR;

    # 如果获取物理地址请求失败，则返回-1
    if (dlpi_msg(e->fd, dlp, DL_PHYS_ADDR_REQ_SIZE, 0,
        DL_PHYS_ADDR_ACK, DL_PHYS_ADDR_ACK_SIZE, sizeof(buf)) < 0)
        return (-1);

    # 将获取到的以太网地址拷贝到 ea 中
    memcpy(ea, buf + dlp->physaddr_ack.dl_addr_offset, sizeof(*ea));
    
    # 返回0表示成功
    return (0);
}

# 设置以太网地址
int
eth_set(eth_t *e, const eth_addr_t *ea)
{
    # 定义一个 DL_primitives 类型的联合体指针
    union DL_primitives *dlp;
    # 定义一个长度为2048的无符号字符数组
    u_char buf[2048];

    # 将 buf 转换为 DL_primitives 类型的指针
    dlp = (union DL_primitives *)buf;
    # 设置设置物理地址请求的数据结构
    dlp->set_physaddr_req.dl_primitive = DL_SET_PHYS_ADDR_REQ;
    dlp->set_physaddr_req.dl_addr_length = ETH_ADDR_LEN;
    dlp->set_physaddr_req.dl_addr_offset = DL_SET_PHYS_ADDR_REQ_SIZE;

    # 将以太网地址拷贝到 buf 中
    memcpy(buf + DL_SET_PHYS_ADDR_REQ_SIZE, ea, sizeof(*ea));
    
    # 返回设置以太网地址的结果
    return (dlpi_msg(e->fd, dlp, DL_SET_PHYS_ADDR_REQ_SIZE + ETH_ADDR_LEN,
        0, DL_OK_ACK, DL_OK_ACK_SIZE, sizeof(buf)));
}
```