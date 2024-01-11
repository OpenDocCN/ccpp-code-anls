# `xmrig\src\crypto\randomx\reciprocal.h`

```
/*
版权声明，禁止再分发和使用，以及免责声明
*/

#pragma once

#include <stdint.h>

#if defined(_M_X64) || defined(__x86_64__)
// 如果是 64 位系统，启用快速倒数计算
#define RANDOMX_HAVE_FAST_RECIPROCAL 1
#else
// 如果不是 64 位系统，禁用快速倒数计算
#define RANDOMX_HAVE_FAST_RECIPROCAL 0
#endif

#if defined(__cplusplus)
extern "C" {
#endif

// 定义倒数函数的接口
uint64_t randomx_reciprocal(uint64_t);
uint64_t randomx_reciprocal_fast(uint64_t);

#if defined(__cplusplus)
}
#endif
```