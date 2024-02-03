# `xmrig\src\crypto\ghostrider\sph_fugue.h`

```cpp
#ifndef SPH_FUGUE_H__  // 如果未定义 SPH_FUGUE_H__，则包含以下内容
#define SPH_FUGUE_H__  // 定义 SPH_FUGUE_H__

#include <stddef.h>  // 包含标准库stddef.h
#include "sph_types.h"  // 包含自定义头文件sph_types.h

#ifdef __cplusplus  // 如果是 C++ 代码
extern "C"{  // 使用 C 语言的方式进行编译
#endif

#define SPH_SIZE_fugue224   224  // 定义宏 SPH_SIZE_fugue224 为 224
#define SPH_SIZE_fugue256   256  // 定义宏 SPH_SIZE_fugue256 为 256
#define SPH_SIZE_fugue384   384  // 定义宏 SPH_SIZE_fugue384 为 384
#define SPH_SIZE_fugue512   512  // 定义宏 SPH_SIZE_fugue512 为 512

typedef struct {  // 定义结构体 sph_fugue_context
#ifndef DOXYGEN_IGNORE  // 如果不是 Doxygen 文档生成器
    sph_u32 partial;  // 32 位无符号整数 partial
    unsigned partial_len;  // 无符号整数 partial_len
    unsigned round_shift;  // 无符号整数 round_shift
    sph_u32 S[36];  // 36 个 32 位无符号整数的数组 S
#if SPH_64  // 如果是 64 位系统
    sph_u64 bit_count;  // 64 位无符号整数 bit_count
#else  // 如果不是 64 位系统
    sph_u32 bit_count_high, bit_count_low;  // 32 位无符号整数 bit_count_high 和 bit_count_low
#endif
#endif
} sph_fugue_context;  // 结构体 sph_fugue_context 的别名

typedef sph_fugue_context sph_fugue224_context;  // sph_fugue224_context 是 sph_fugue_context 的别名
typedef sph_fugue_context sph_fugue256_context;  // sph_fugue256_context 是 sph_fugue_context 的别名
typedef sph_fugue_context sph_fugue384_context;  // sph_fugue384_context 是 sph_fugue_context 的别名
typedef sph_fugue_context sph_fugue512_context;  // sph_fugue512_context 是 sph_fugue_context 的别名

void sph_fugue224_init(void *cc);  // 定义函数 sph_fugue224_init，参数为指向 void 类型的指针 cc
void sph_fugue224(void *cc, const void *data, size_t len);  // 定义函数 sph_fugue224，参数为指向 void 类型的指针 cc、指向常量 void 类型的指针 data 和 size_t 类型的 len
void sph_fugue224_close(void *cc, void *dst);  // 定义函数 sph_fugue224_close，参数为指向 void 类型的指针 cc 和指向 void 类型的指针 dst
void sph_fugue224_addbits_and_close(  // 定义函数 sph_fugue224_addbits_and_close
    void *cc, unsigned ub, unsigned n, void *dst);  // 参数为指向 void 类型的指针 cc、无符号整数 ub、无符号整数 n 和指向 void 类型的指针 dst

// 后续函数定义与上述类似，不再赘述

#ifdef __cplusplus  // 如果是 C++ 代码
}
#endif

#endif  // 结束条件，结束宏定义
```