# `nmap\libdnet-stripped\src\rand.c`

```cpp
/*
 * rand.c
 *
 * Pseudorandom number generation, based on OpenBSD arc4random().
 *
 * Copyright (c) 2000 Dug Song <dugsong@monkey.org>
 * Copyright (c) 1996 David Mazieres <dm@lcs.mit.edu>
 *
 * $Id: rand.c 587 2005-02-15 06:37:07Z dugsong $
 */

#include "config.h"

#ifdef _WIN32
/* XXX */
# undef _WIN32_WINNT
# define _WIN32_WINNT _WIN32_WINNT_WIN7
# include <bcrypt.h>
# pragma comment(lib, "bcrypt.lib")
# define inline __inline
#else
# include <sys/types.h>
# include <sys/time.h>
# include <unistd.h>
#endif
#include <fcntl.h>
#include <stdlib.h>
#include <string.h>

#include "dnet.h"

struct rand_handle {
    uint8_t         i;  // 用于随机数生成的变量
    uint8_t         j;  // 用于随机数生成的变量
    uint8_t         s[256];  // 用于随机数生成的变量
    u_char        *tmp;  // 临时存储随机数的指针
    int         tmplen;  // 临时存储随机数的长度
};

static inline void
rand_init(rand_t *rand)
{
    int i;
    
    for (i = 0; i < 256; i++)
        rand->s[i] = i;  // 初始化随机数生成器
    rand->i = rand->j = 0;  // 初始化随机数生成器
}

static inline void
rand_addrandom(rand_t *rand, u_char *buf, int len)
{
    int i;
    uint8_t si;
    
    rand->i--;  // 更新随机数生成器状态
    for (i = 0; i < 256; i++) {
        rand->i = (rand->i + 1);  // 更新随机数生成器状态
        si = rand->s[rand->i];  // 更新随机数生成器状态
        rand->j = (rand->j + si + buf[i % len]);  // 更新随机数生成器状态
        rand->s[rand->i] = rand->s[rand->j];  // 更新随机数生成器状态
        rand->s[rand->j] = si;  // 更新随机数生成器状态
    }
    rand->j = rand->i;  // 更新随机数生成器状态
}

rand_t *
rand_open(void)
{
    rand_t *r;
    u_char seed[256];
#ifdef _WIN32
    if (STATUS_SUCCESS != BCryptGenRandom(NULL, seed, sizeof(seed), BCRYPT_USE_SYSTEM_PREFERRED_RNG))
      return NULL;
#else
    struct timeval *tv = (struct timeval *)seed;
    int fd;

    if ((fd = open("/dev/arandom", O_RDONLY)) != -1 ||
        (fd = open("/dev/urandom", O_RDONLY)) != -1) {
        read(fd, seed + sizeof(*tv), sizeof(seed) - sizeof(*tv));  // 从随机设备中读取种子
        close(fd);
    }
    gettimeofday(tv, NULL);  // 获取当前时间作为种子
#endif
    if ((r = malloc(sizeof(*r))) != NULL) {
        rand_init(r);  // 初始化随机数生成器
        rand_addrandom(r, seed, 128);  // 添加种子到随机数生成器
        rand_addrandom(r, seed + 128, 128);  // 添加种子到随机数生成器
        r->tmp = NULL;  // 初始化临时存储随机数的指针
        r->tmplen = 0;  // 初始化临时存储随机数的长度
    }
    return (r);  // 返回随机数生成器
}

static uint8_t
# 从随机数生成器中获取一个字节
rand_getbyte(rand_t *r)
{
    uint8_t si, sj;

    # 更新 i 的值
    r->i = (r->i + 1);
    # 获取 s[i] 的值
    si = r->s[r->i];
    # 更新 j 的值
    r->j = (r->j + si);
    # 获取 s[j] 的值
    sj = r->s[r->j];
    # 交换 s[i] 和 s[j] 的值
    r->s[r->i] = sj;
    r->s[r->j] = si;
    # 返回 (s[i] + s[j]) & 0xff 处的值
    return (r->s[(si + sj) & 0xff]);
}

# 从随机数生成器中获取随机数填充到指定的缓冲区
int
rand_get(rand_t *r, void *buf, size_t len)
{
    u_char *p;
    u_int i;

    # 遍历缓冲区，填充随机数
    for (p = buf, i = 0; i < len; i++) {
        p[i] = rand_getbyte(r);
    }
    return (0);
}

# 重新初始化随机数生成器，并用指定的数据填充
int
rand_set(rand_t *r, const void *buf, size_t len)
{
    # 重新初始化随机数生成器
    rand_init(r);
    # 用指定的数据填充随机数生成器
    rand_addrandom(r, (u_char *)buf, len);
    rand_addrandom(r, (u_char *)buf, len);
    return (0);
}

# 向随机数生成器中添加指定的数据
int
rand_add(rand_t *r, const void *buf, size_t len)
{
    # 向随机数生成器中添加指定的数据
    rand_addrandom(r, (u_char *)buf, len);
    return (0);
}

# 从随机数生成器中获取一个 8 位无符号整数
uint8_t
rand_uint8(rand_t *r)
{
    # 从随机数生成器中获取一个字节
    return (rand_getbyte(r));
}

# 从随机数生成器中获取一个 16 位无符号整数
uint16_t
rand_uint16(rand_t *r)
{
    uint16_t val;

    # 从随机数生成器中获取一个字节，并左移 8 位
    val = rand_getbyte(r) << 8;
    # 从随机数生成器中获取一个字节，并与 val 进行按位或运算
    val |= rand_getbyte(r);
    return (val);
}

# 从随机数生成器中获取一个 32 位无符号整数
uint32_t
rand_uint32(rand_t *r)
{
    uint32_t val;

    # 从随机数生成器中获取一个字节，并左移 24 位
    val = rand_getbyte(r) << 24;
    # 从随机数生成器中获取一个字节，并左移 16 位，与 val 进行按位或运算
    val |= rand_getbyte(r) << 16;
    # 从随机数生成器中获取一个字节，并左移 8 位，与 val 进行按位或运算
    val |= rand_getbyte(r) << 8;
    # 从随机数生成器中获取一个字节，并与 val 进行按位或运算
    val |= rand_getbyte(r);
    return (val);
}

# 对指定的数组进行洗牌
int
rand_shuffle(rand_t *r, void *base, size_t nmemb, size_t size)
{
    u_char *save, *src, *dst, *start = (u_char *)base;
    u_int i, j;

    # 如果数组长度小于 2，则直接返回
    if (nmemb < 2)
        return (0);
    
    # 如果随机数生成器的临时缓冲区长度小于指定的大小
    if ((u_int)r->tmplen < size) {
        # 如果临时缓冲区为空，则分配指定大小的内存
        if (r->tmp == NULL) {
            if ((save = malloc(size)) == NULL)
                return (-1);
        } else if ((save = realloc(r->tmp, size)) == NULL)
            return (-1);
        
        r->tmp = save;
        r->tmplen = size;
    } else
        save = r->tmp;
    
    # 对数组进行洗牌
    for (i = 0; i < nmemb; i++) {
        if ((j = rand_uint32(r) % (nmemb - 1)) != i) {
            src = start + (size * i);
            dst = start + (size * j);
            memcpy(save, dst, size);
            memcpy(dst, src, size);
            memcpy(src, save, size);
        }
    }
    return (0);
}

# 关闭随机数生成器
rand_t *
rand_close(rand_t *r)
{
    # 如果指针 r 不为空
    if (r != NULL) {
        # 如果 r 指向的 tmp 不为空，则释放其内存
        if (r->tmp != NULL)
            free(r->tmp);
        # 释放指针 r 指向的内存
        free(r);
    }
    # 返回空指针
    return (NULL);
# 闭合前面的函数定义
```