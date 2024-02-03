# `xmrig\src\crypto\cn\c_blake256.h`

```cpp
#ifndef _BLAKE256_H_
#define _BLAKE256_H_

#include <stdint.h>

typedef struct {
  uint32_t h[8], s[4], t[2];  // 定义包含8个32位整数的数组h，4个32位整数的数组s，2个32位整数的数组t的结构体
  int buflen, nullt;  // 定义整型变量buflen和nullt
  uint8_t buf[64];  // 定义包含64个8位整数的数组buf
} state;  // 定义名为state的结构体类型

typedef struct {
  state inner;  // 包含一个名为inner的state结构体的结构体
  state outer;  // 包含一个名为outer的state结构体的结构体
} hmac_state;  // 定义名为hmac_state的结构体类型

void blake256_init(state *);  // 声明blake256_init函数，接受一个state类型指针参数
void blake224_init(state *);  // 声明blake224_init函数，接受一个state类型指针参数

void blake256_update(state *, const uint8_t *, uint64_t);  // 声明blake256_update函数，接受一个state类型指针参数，一个指向8位整数的常量指针参数，一个64位整数参数
void blake224_update(state *, const uint8_t *, uint64_t);  // 声明blake224_update函数，接受一个state类型指针参数，一个指向8位整数的常量指针参数，一个64位整数参数

void blake256_final(state *, uint8_t *);  // 声明blake256_final函数，接受一个state类型指针参数，一个指向8位整数的指针参数
void blake224_final(state *, uint8_t *);  // 声明blake224_final函数，接受一个state类型指针参数，一个指向8位整数的指针参数

void blake256_hash(uint8_t *, const uint8_t *, uint64_t);  // 声明blake256_hash函数，接受一个指向8位整数的指针参数，一个指向8位整数的常量指针参数，一个64位整数参数
void blake224_hash(uint8_t *, const uint8_t *, uint64_t);  // 声明blake224_hash函数，接受一个指向8位整数的指针参数，一个指向8位整数的常量指针参数，一个64位整数参数

/* HMAC functions: */

void hmac_blake256_init(hmac_state *, const uint8_t *, uint64_t);  // 声明hmac_blake256_init函数，接受一个hmac_state类型指针参数，一个指向8位整数的常量指针参数，一个64位整数参数
void hmac_blake224_init(hmac_state *, const uint8_t *, uint64_t);  // 声明hmac_blake224_init函数，接受一个hmac_state类型指针参数，一个指向8位整数的常量指针参数，一个64位整数参数

void hmac_blake256_update(hmac_state *, const uint8_t *, uint64_t);  // 声明hmac_blake256_update函数，接受一个hmac_state类型指针参数，一个指向8位整数的常量指针参数，一个64位整数参数
void hmac_blake224_update(hmac_state *, const uint8_t *, uint64_t);  // 声明hmac_blake224_update函数，接受一个hmac_state类型指针参数，一个指向8位整数的常量指针参数，一个64位整数参数

void hmac_blake256_final(hmac_state *, uint8_t *);  // 声明hmac_blake256_final函数，接受一个hmac_state类型指针参数，一个指向8位整数的指针参数
void hmac_blake224_final(hmac_state *, uint8_t *);  // 声明hmac_blake224_final函数，接受一个hmac_state类型指针参数，一个指向8位整数的指针参数

void hmac_blake256_hash(uint8_t *, const uint8_t *, uint64_t, const uint8_t *, uint64_t);  // 声明hmac_blake256_hash函数，接受一个指向8位整数的指针参数，一个指向8位整数的常量指针参数，两个64位整数参数
void hmac_blake224_hash(uint8_t *, const uint8_t *, uint64_t, const uint8_t *, uint64_t);  // 声明hmac_blake224_hash函数，接受一个指向8位整数的指针参数，一个指向8位整数的常量指针参数，两个64位整数参数

#endif /* _BLAKE256_H_ */
```