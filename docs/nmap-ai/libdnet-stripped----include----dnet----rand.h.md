# `nmap\libdnet-stripped\include\dnet\rand.h`

```
/*
 * rand.h
 *
 * Pseudo-random number generation, based on OpenBSD arc4random().
 *
 * Copyright (c) 2000 Dug Song <dugsong@monkey.org>
 * Copyright (c) 1996 David Mazieres <dm@lcs.mit.edu>
 *
 * $Id: rand.h 340 2002-04-07 19:01:25Z dugsong $
 */

#ifndef DNET_RAND_H
#define DNET_RAND_H

// 定义 rand_handle 结构体类型
typedef struct rand_handle rand_t;

__BEGIN_DECLS
// 打开一个伪随机数生成器
rand_t    *rand_open(void);

// 从伪随机数生成器中获取随机数
int     rand_get(rand_t *r, void *buf, size_t len);

// 设置伪随机数生成器的种子
int     rand_set(rand_t *r, const void *seed, size_t len);

// 向伪随机数生成器中添加数据
int     rand_add(rand_t *r, const void *buf, size_t len);

// 生成一个 8 位无符号整数
uint8_t     rand_uint8(rand_t *r);

// 生成一个 16 位无符号整数
uint16_t rand_uint16(rand_t *r);

// 生成一个 32 位无符号整数
uint32_t rand_uint32(rand_t *r);

// 对数组进行乱序排列
int     rand_shuffle(rand_t *r, void *base, size_t nmemb, size_t size);

// 关闭伪随机数生成器
rand_t    *rand_close(rand_t *r);
__END_DECLS

#endif /* DNET_RAND_H */
```