# `nmap\nbase\nbase_rnd.c`

```cpp
/* $Id$ */

#include "nbase.h"  // 包含自定义的头文件 nbase.h
#include <errno.h>  // 包含错误处理相关的头文件
#include <string.h>  // 包含字符串处理相关的头文件
#include <stdio.h>  // 包含标准输入输出相关的头文件
#include <stdlib.h>  // 包含标准库相关的头文件
#include <fcntl.h>  // 包含文件控制相关的头文件
#if HAVE_SYS_TIME_H
#include <sys/time.h>  // 如果有 sys/time.h 头文件，则包含
#endif /* HAV_SYS_TIME_H */
#ifdef WIN32
#include <wincrypt.h>  // 如果是 WIN32 平台，则包含 wincrypt.h 头文件
#endif /* WIN32 */

/* data for our random state */
struct nrand_handle {
  u8    i, j, s[256], *tmp;  // 定义结构体 nrand_handle，包含 i、j、s 数组和 tmp 指针
  int   tmplen;  // 定义 tmplen 变量
};
typedef struct nrand_handle nrand_h;  // 定义 nrand_h 为 struct nrand_handle 的别名

static void nrand_addrandom(nrand_h *rand, u8 *buf, int len) {
  int i;
  u8 si;

  /* Mix entropy in buf with s[]...
   *
   * This is the ARC4 key-schedule.  It is rather poor and doesn't mix
   * the key in very well.  This causes a bias at the start of the stream.
   * To eliminate most of this bias, the first N bytes of the stream should
   * be dropped.
   */
  rand->i--;  // 对 rand->i 进行自减操作
  for (i = 0; i < 256; i++) {
    rand->i = (rand->i + 1);  // 对 rand->i 进行自增操作
    si = rand->s[rand->i];  // 获取 rand->s[rand->i] 的值
    rand->j = (rand->j + si + buf[i % len]);  // 对 rand->j 进行赋值操作
    rand->s[rand->i] = rand->s[rand->j];  // 对 rand->s[rand->i] 进行赋值操作
    rand->s[rand->j] = si;  // 对 rand->s[rand->j] 进行赋值操作
  }
  rand->j = rand->i;  // 对 rand->j 进行赋值操作
}

static u8 nrand_getbyte(nrand_h *r) {
  u8 si, sj;

  /* This is the core of ARC4 and provides the pseudo-randomness */
  r->i = (r->i + 1);  // 对 r->i 进行自增操作
  si = r->s[r->i];  // 获取 r->s[r->i] 的值
  r->j = (r->j + si);  // 对 r->j 进行赋值操作
  sj = r->s[r->j];  // 获取 r->s[r->j] 的值
  r->s[r->i] = sj; /* The start of the the swap */  // 对 r->s[r->i] 进行赋值操作
  r->s[r->j] = si; /* The other half of the swap */  // 对 r->s[r->j] 进行赋值操作
  return (r->s[(si + sj) & 0xff]);  // 返回计算后的值
}

int nrand_get(nrand_h *r, void *buf, size_t len) {
  u8 *p;
  size_t i;

  /* Hand out however many bytes were asked for */
  for (p = buf, i = 0; i < len; i++) {
    p[i] = nrand_getbyte(r);  // 调用 nrand_getbyte 函数，将结果赋值给 p[i]
  }
  return (0);  // 返回 0
}

void nrand_init(nrand_h *r) {
  u8 seed[256]; /* Starts out with "random" stack data */
  int i;

  /* Gather seed entropy with best the OS has to offer */
#ifdef WIN32
  HCRYPTPROV hcrypt = 0;  // 定义 HCRYPTPROV 类型的变量 hcrypt，并赋值为 0

  CryptAcquireContext(&hcrypt, NULL, NULL, PROV_RSA_FULL, CRYPT_VERIFYCONTEXT);  // 调用 CryptAcquireContext 函数
  CryptGenRandom(hcrypt, sizeof(seed), seed);  // 调用 CryptGenRandom 函数
  CryptReleaseContext(hcrypt, 0);  // 调用 CryptReleaseContext 函数
#else
  // 定义指向时间结构体的指针tv，指向整型的指针pid，整型fd
  struct timeval *tv = (struct timeval *)seed;
  int *pid = (int *)(seed + sizeof(*tv));
  int fd;

  // 获取当前时间并填充seed的最低位
  gettimeofday(tv, NULL); /* fill lowest seed[] with time */
  // 获取当前进程的PID并填充seed的次低位
  *pid = getpid();        /* fill next lowest seed[] with pid */

  // 尝试使用操作系统提供的熵来填充状态的其余部分
  if ((fd = open("/dev/urandom", O_RDONLY)) != -1 ||
      (fd = open("/dev/arandom", O_RDONLY)) != -1) {
    ssize_t n;
    do {
      errno = 0;
      // 从文件描述符fd中读取数据，填充seed的剩余部分
      n = read(fd, seed + sizeof(*tv) + sizeof(*pid),
               sizeof(seed) - sizeof(*tv) - sizeof(*pid));
    } while (n < 0 && errno == EINTR);
    close(fd);
  }
#endif

  // 用初始值填充状态的句柄
  for (i = 0; i < 256; i++) { r->s[i] = i; };
  r->i = r->j = 0;

  // 使用seed的下半部分数据来增加熵
  nrand_addrandom(r, seed, 128); /* lower half of seed data for entropy */
  // 使用seed的上半部分数据来增加熵
  nrand_addrandom(r, seed + 128, 128); /* Now use upper half */
  r->tmp = NULL;
  r->tmplen = 0;

  // 这个流将开始有偏差。丢弃1K的流
  nrand_get(r, seed, 256); nrand_get(r, seed, 256);
  nrand_get(r, seed, 256); nrand_get(r, seed, 256);
}

// 获取随机字节
int get_random_bytes(void *buf, int numbytes) {
  static nrand_h state;
  static int state_init = 0;

  // 如果需要，进行初始化
  if (!state_init) {
    nrand_init(&state);
    state_init = 1;
  }

  // 填充缓冲区
  nrand_get(&state, buf, numbytes);

  return 0;
}

// 获取随机整数
int get_random_int() {
  int i;
  get_random_bytes(&i, sizeof(int));
  return i;
}

// 获取无符号随机整数
unsigned int get_random_uint() {
  unsigned int i;
  get_random_bytes(&i, sizeof(unsigned int));
  return i;
}

// 获取64位无符号随机整数
u64 get_random_u64() {
  u64 i;
  get_random_bytes(&i, sizeof(i));
  return i;
}

// 获取32位无符号随机整数
u32 get_random_u32() {
  u32 i;
  get_random_bytes(&i, sizeof(i));
  return i;
}

// 获取16位无符号随机整数
u16 get_random_u16() {
  u16 i;
  get_random_bytes(&i, sizeof(i));
  return i;
}

// 获取8位无符号随机整数
u8 get_random_u8() {
  u8 i;
  get_random_bytes(&i, sizeof(i));
  return i;
}
# 生成一个无符号短整型随机数
unsigned short get_random_ushort() {
  # 定义一个无符号短整型变量s
  unsigned short s;
  # 调用get_random_bytes函数生成一个无符号短整型随机数，并将其赋值给s
  get_random_bytes(&s, sizeof(unsigned short));
  # 返回生成的随机数s
  return s;
}

/* 这个函数是神奇的 ;-)
 *
 * 有时候Nmap想要生成看起来随机但没有重复的IP地址。强大的RC4生成器
 * 不能用于这个目的，因为如果获取足够多的IP地址，它可能会生成重复的。
 *
 * 这个函数利用了LCG不会在其周期内重复的事实。LCG有一些非常糟糕的属性，
 * 因此这个函数会额外工作，尝试调整LCG的输出，使其具有非常好的统计特性但不会重复。
 * 所使用的调整大部分是即兴而为，但基本上是基于良好的想法，并经过了适度的测试。参见下面的链接和推理。
 */
u32 get_random_unique_u32() {
  # 定义静态变量state, tweak1, tweak2, tweak3和state_init
  static u32 state, tweak1, tweak2, tweak3;
  static int state_init = 0;
  u32 output;

  /* 如果需要，进行初始化 */
  if (!state_init) {
    # 生成state, tweak1, tweak2, tweak3的随机值
    get_random_bytes(&state, sizeof(state));
    get_random_bytes(&tweak1, sizeof(tweak1));
    get_random_bytes(&tweak2, sizeof(tweak2));
    get_random_bytes(&tweak3, sizeof(tweak3));
  }
}
```