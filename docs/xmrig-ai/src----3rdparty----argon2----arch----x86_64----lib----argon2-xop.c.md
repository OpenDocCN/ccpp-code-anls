# `xmrig\src\3rdparty\argon2\arch\x86_64\lib\argon2-xop.c`

```
#include "argon2-xop.h"  // 包含自定义的头文件 argon2-xop.h

#ifdef HAVE_XOP  // 如果定义了 HAVE_XOP 宏
#include <string.h>  // 包含标准库头文件 string.h

#ifdef __GNUC__  // 如果是 GNU 编译器
#   include <x86intrin.h>  // 包含 x86 平台特定的头文件
#else  // 如果不是 GNU 编译器
#   include <intrin.h>  // 包含通用的内部函数头文件
#endif

#define ror64(x, c) _mm_roti_epi64((x), -(c))  // 定义宏 ror64，用于对 x 进行 c 位的循环右移

static __m128i f(__m128i x, __m128i y)  // 定义静态函数 f，接受两个 __m128i 类型的参数 x 和 y
{
    __m128i z = _mm_mul_epu32(x, y);  // 计算 x 和 y 的无符号 32 位乘法
    return _mm_add_epi64(_mm_add_epi64(x, y), _mm_add_epi64(z, z));  // 返回 x+y+z+z 的结果
}

#define G1(A0, B0, C0, D0, A1, B1, C1, D1) \  // 定义宏 G1，接受 8 个参数
    do { \  // 开始 do-while 循环
        A0 = f(A0, B0); \  // 调用函数 f，更新 A0
        A1 = f(A1, B1); \  // 调用函数 f，更新 A1
\
        D0 = _mm_xor_si128(D0, A0); \  // 对 D0 和 A0 进行异或操作
        D1 = _mm_xor_si128(D1, A1); \  // 对 D1 和 A1 进行异或操作
\
        D0 = ror64(D0, 32); \  // 对 D0 进行 32 位的循环右移
        D1 = ror64(D1, 32); \  // 对 D1 进行 32 位的循环右移
\
        C0 = f(C0, D0); \  // 调用函数 f，更新 C0
        C1 = f(C1, D1); \  // 调用函数 f，更新 C1
\
        B0 = _mm_xor_si128(B0, C0); \  // 对 B0 和 C0 进行异或操作
        B1 = _mm_xor_si128(B1, C1); \  // 对 B1 和 C1 进行异或操作
\
        B0 = ror64(B0, 24); \  // 对 B0 进行 24 位的循环右移
        B1 = ror64(B1, 24); \  // 对 B1 进行 24 位的循环右移
    } while ((void)0, 0)  // 结束 do-while 循环

#define G2(A0, B0, C0, D0, A1, B1, C1, D1) \  // 定义宏 G2，接受 8 个参数
    do { \  // 开始 do-while 循环
        A0 = f(A0, B0); \  // 调用函数 f，更新 A0
        A1 = f(A1, B1); \  // 调用函数 f，更新 A1
\
        D0 = _mm_xor_si128(D0, A0); \  // 对 D0 和 A0 进行异或操作
        D1 = _mm_xor_si128(D1, A1); \  // 对 D1 和 A1 进行异或操作
\
        D0 = ror64(D0, 16); \  // 对 D0 进行 16 位的循环右移
        D1 = ror64(D1, 16); \  // 对 D1 进行 16 位的循环右移
\
        C0 = f(C0, D0); \  // 调用函数 f，更新 C0
        C1 = f(C1, D1); \  // 调用函数 f，更新 C1
\
        B0 = _mm_xor_si128(B0, C0); \  // 对 B0 和 C0 进行异或操作
        B1 = _mm_xor_si128(B1, C1); \  // 对 B1 和 C1 进行异或操作
\
        B0 = ror64(B0, 63); \  // 对 B0 进行 63 位的循环右移
        B1 = ror64(B1, 63); \  // 对 B1 进行 63 位的循环右移
    } while ((void)0, 0)  // 结束 do-while 循环

#define DIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1) \  // 定义宏 DIAGONALIZE，接受 8 个参数
    do { \  // 开始 do-while 循环
        __m128i t0 = _mm_alignr_epi8(B1, B0, 8); \  // 对 B1 和 B0 进行字节对齐
        __m128i t1 = _mm_alignr_epi8(B0, B1, 8); \  // 对 B0 和 B1 进行字节对齐
        B0 = t0; \  // 更新 B0
        B1 = t1; \  // 更新 B1
\
        t0 = _mm_alignr_epi8(D1, D0, 8); \  // 对 D1 和 D0 进行字节对齐
        t1 = _mm_alignr_epi8(D0, D1, 8); \  // 对 D0 和 D1 进行字节对齐
        D0 = t1; \  // 更新 D0
        D1 = t0; \  // 更新 D1
    } while ((void)0, 0)  // 结束 do-while 循环

#define UNDIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1) \  // 定义宏 UNDIAGONALIZE，接受 8 个参数
    do { \  // 开始 do-while 循环
        __m128i t0 = _mm_alignr_epi8(B0, B1, 8); \  // 对 B0 和 B1 进行字节对齐
        __m128i t1 = _mm_alignr_epi8(B1, B0, 8); \  // 对 B1 和 B0 进行字节对齐
        B0 = t0; \  // 更新 B0
        B1 = t1; \  // 更新 B1
\
        t0 = _mm_alignr_epi8(D0, D1, 8); \  // 对 D0 和 D1 进行字节对齐
        t1 = _mm_alignr_epi8(D1, D0, 8); \  // 对 D1 和 D0 进行字节对齐
        D0 = t1; \  // 更新 D0
        D1 = t0; \  // 更新 D1
    } while ((void)0, 0)  // 结束 do-while 循环
#define BLAKE2_ROUND(A0, A1, B0, B1, C0, C1, D0, D1) \  // 定义 BLAKE2_ROUND 宏，接受8个参数
    do { \  // 执行以下代码块
        G1(A0, B0, C0, D0, A1, B1, C1, D1); \  // 调用 G1 函数
        G2(A0, B0, C0, D0, A1, B1, C1, D1); \  // 调用 G2 函数
\
        DIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1); \  // 调用 DIAGONALIZE 函数
\
        G1(A0, B0, C1, D0, A1, B1, C0, D1); \  // 调用 G1 函数
        G2(A0, B0, C1, D0, A1, B1, C0, D1); \  // 调用 G2 函数
\
        UNDIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1); \  // 调用 UNDIAGONALIZE 函数
    } while ((void)0, 0)  // 循环条件为假，即只执行一次

#include "argon2-template-128.h"  // 包含 argon2-template-128.h 文件

void xmrig_ar2_fill_segment_xop(const argon2_instance_t *instance, argon2_position_t position)  // 定义函数 xmrig_ar2_fill_segment_xop，接受两个参数
{
    fill_segment_128(instance, position);  // 调用 fill_segment_128 函数
}

extern int cpu_flags_has_xop(void);  // 声明函数 cpu_flags_has_xop，无参数，返回整型
int xmrig_ar2_check_xop(void) { return cpu_flags_has_xop(); }  // 定义函数 xmrig_ar2_check_xop，返回 cpu_flags_has_xop 函数的结果

#else

void xmrig_ar2_fill_segment_xop(const argon2_instance_t *instance, argon2_position_t position) {}  // 定义函数 xmrig_ar2_fill_segment_xop，无操作
int xmrig_ar2_check_xop(void) { return 0; }  // 定义函数 xmrig_ar2_check_xop，返回0

#endif  // 结束条件编译指令
```