# `nmap\libdnet-stripped\include\dnet\addr.h`

```cpp
/*
 * addr.h
 *
 * Network address operations.
 *
 * Copyright (c) 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: addr.h 404 2003-02-27 03:44:55Z dugsong $
 */

#ifndef DNET_ADDR_H
#define DNET_ADDR_H

#define ADDR_TYPE_NONE        0    /* 未设置地址 */
#define    ADDR_TYPE_ETH        1    /* 以太网 */
#define    ADDR_TYPE_IP        2    /* 互联网协议 v4 */
#define    ADDR_TYPE_IP6        3    /* 互联网协议 v6 */

struct addr {
    uint16_t        addr_type;  // 地址类型
    uint16_t        addr_bits;  // 地址位数
    union {
        eth_addr_t    __eth;  // 以太网地址
        ip_addr_t    __ip;  // IPv4地址
        ip6_addr_t    __ip6;  // IPv6地址
        
        uint8_t        __data8[16];  // 8位数据
        uint16_t    __data16[8];  // 16位数据
        uint32_t    __data32[4];  // 32位数据
    } __addr_u;
};
#define addr_eth    __addr_u.__eth  // 以太网地址
#define addr_ip        __addr_u.__ip  // IPv4地址
#define addr_ip6    __addr_u.__ip6  // IPv6地址
#define addr_data8    __addr_u.__data8  // 8位数据
#define addr_data16    __addr_u.__data16  // 16位数据
#define addr_data32    __addr_u.__data32  // 32位数据

#define addr_pack(addr, type, bits, data, len) do {    \
    (addr)->addr_type = type;            // 设置地址类型
    (addr)->addr_bits = bits;            // 设置地址位数
    memmove((addr)->addr_data8, (char *)data, len);    // 将数据移动到地址数据中
} while (0)

__BEGIN_DECLS
int     addr_cmp(const struct addr *a, const struct addr *b);  // 比较两个地址

int     addr_bcast(const struct addr *a, struct addr *b);  // 广播地址
int     addr_net(const struct addr *a, struct addr *b);  // 网络地址

char    *addr_ntop(const struct addr *src, char *dst, size_t size);  // 将地址转换为可读的字符串形式
int     addr_pton(const char *src, struct addr *dst);  // 将可读的字符串形式转换为地址

char    *addr_ntoa(const struct addr *a);  // 将地址转换为点分十进制字符串形式
#define     addr_aton    addr_pton  // 将点分十进制字符串形式转换为地址

int     addr_ntos(const struct addr *a, struct sockaddr *sa);  // 将地址转换为套接字地址
int     addr_ston(const struct sockaddr *sa, struct addr *a);  // 将套接字地址转换为地址

int     addr_btos(uint16_t bits, struct sockaddr *sa);  // 将位数转换为套接字地址
int     addr_stob(const struct sockaddr *sa, uint16_t *bits);  // 将套接字地址转换为位数

int     addr_btom(uint16_t bits, void *mask, size_t size);  // 将位数转换为掩码
int     addr_mtob(const void *mask, size_t size, uint16_t *bits);  // 将掩码转换为位数
__END_DECLS

#endif /* DNET_ADDR_H */
```