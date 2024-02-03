# `nmap\libdnet-stripped\include\dnet\tun.h`

```cpp
/*
 * tun.h
 *
 * Network tunnel device.
 *
 * Copyright (c) 2001 Dug Song <dugsong@monkey.org>
 *
 * $Id: tun.h 547 2005-01-25 21:30:40Z dugsong $
 */

// 防止头文件重复包含
#ifndef DNET_TUN_H
#define DNET_TUN_H

// 定义 tun_t 结构体
typedef struct tun    tun_t;

// 声明函数 tun_open，用于打开网络隧道设备
__BEGIN_DECLS
tun_t       *tun_open(struct addr *src, struct addr *dst, int mtu);
// 获取网络隧道设备的文件描述符
int        tun_fileno(tun_t *tun);
// 获取网络隧道设备的名称
const char *tun_name(tun_t *tun);
// 发送数据到网络隧道设备
ssize_t        tun_send(tun_t *tun, const void *buf, size_t size);
// 从网络隧道设备接收数据
ssize_t        tun_recv(tun_t *tun, void *buf, size_t size);
// 关闭网络隧道设备
tun_t       *tun_close(tun_t *tun);
__END_DECLS

#endif /* DNET_TUN_H */
```