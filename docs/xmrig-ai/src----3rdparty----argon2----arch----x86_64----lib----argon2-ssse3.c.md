# `xmrig\src\3rdparty\argon2\arch\x86_64\lib\argon2-ssse3.c`

```cpp
#include "argon2-ssse3.h"  // 包含 SSSE3 版本的 Argon2 头文件

#ifdef HAVE_SSSE3  // 如果支持 SSSE3 指令集
#include <string.h>  // 包含字符串处理函数的头文件

#ifdef __GNUC__  // 如果是 GCC 编译器
#   include <x86intrin.h>  // 包含 SSSE3 指令集的头文件
#else
#   include <intrin.h>  // 包含 SSSE3 指令集的头文件
#endif

#define r16 (_mm_setr_epi8( \  // 定义一个 128 位整数，用于进行位移操作
     2,  3,  4,  5,  6,  7,  0,  1, \
    10, 11, 12, 13, 14, 15,  8,  9))

#define r24 (_mm_setr_epi8( \  // 定义一个 128 位整数，用于进行位移操作
     3,  4,  5,  6,  7,  0,  1,  2, \
    11, 12, 13, 14, 15,  8,  9, 10))

#define ror64_16(x) _mm_shuffle_epi8((x), r16)  // 定义一个宏，用于进行 64 位整数的位移操作
#define ror64_24(x) _mm_shuffle_epi8((x), r24)  // 定义一个宏，用于进行 64 位整数的位移操作
#define ror64_32(x) _mm_shuffle_epi32((x), _MM_SHUFFLE(2, 3, 0, 1))  // 定义一个宏，用于进行 64 位整数的位移操作
#define ror64_63(x) \  // 定义一个宏，用于进行 64 位整数的位移操作
    _mm_xor_si128(_mm_srli_epi64((x), 63), _mm_add_epi64((x), (x)))

static __m128i f(__m128i x, __m128i y)  // 定义一个静态函数，接受两个 128 位整数参数
{
    __m128i z = _mm_mul_epu32(x, y);  // 计算两个 64 位整数的乘积
    return _mm_add_epi64(_mm_add_epi64(x, y), _mm_add_epi64(z, z));  // 返回四个 64 位整数的和
}

#define G1(A0, B0, C0, D0, A1, B1, C1, D1) \  // 定义一个宏，接受八个 128 位整数参数
    do { \  // 开始一个 do-while 循环
        A0 = f(A0, B0); \  // 调用函数 f，对 A0 和 B0 进行计算
        A1 = f(A1, B1); \  // 调用函数 f，对 A1 和 B1 进行计算
\
        D0 = _mm_xor_si128(D0, A0); \  // 对 D0 和 A0 进行按位异或操作
        D1 = _mm_xor_si128(D1, A1); \  // 对 D1 和 A1 进行按位异或操作
\
        D0 = ror64_32(D0); \  // 对 D0 进行 32 位的位移操作
        D1 = ror64_32(D1); \  // 对 D1 进行 32 位的位移操作
\
        C0 = f(C0, D0); \  // 调用函数 f，对 C0 和 D0 进行计算
        C1 = f(C1, D1); \  // 调用函数 f，对 C1 和 D1 进行计算
\
        B0 = _mm_xor_si128(B0, C0); \  // 对 B0 和 C0 进行按位异或操作
        B1 = _mm_xor_si128(B1, C1); \  // 对 B1 和 C1 进行按位异或操作
\
        B0 = ror64_24(B0); \  // 对 B0 进行 24 位的位移操作
        B1 = ror64_24(B1); \  // 对 B1 进行 24 位的位移操作
    } while ((void)0, 0)  // 结束 do-while 循环

#define G2(A0, B0, C0, D0, A1, B1, C1, D1) \  // 定义一个宏，接受八个 128 位整数参数
    do { \  // 开始一个 do-while 循环
        A0 = f(A0, B0); \  // 调用函数 f，对 A0 和 B0 进行计算
        A1 = f(A1, B1); \  // 调用函数 f，对 A1 和 B1 进行计算
\
        D0 = _mm_xor_si128(D0, A0); \  // 对 D0 和 A0 进行按位异或操作
        D1 = _mm_xor_si128(D1, A1); \  // 对 D1 和 A1 进行按位异或操作
\
        D0 = ror64_16(D0); \  // 对 D0 进行 16 位的位移操作
        D1 = ror64_16(D1); \  // 对 D1 进行 16 位的位移操作
\
        C0 = f(C0, D0); \  // 调用函数 f，对 C0 和 D0 进行计算
        C1 = f(C1, D1); \  // 调用函数 f，对 C1 和 D1 进行计算
\
        B0 = _mm_xor_si128(B0, C0); \  // 对 B0 和 C0 进行按位异或操作
        B1 = _mm_xor_si128(B1, C1); \  // 对 B1 和 C1 进行按位异或操作
\
        B0 = ror64_63(B0); \  // 对 B0 进行 63 位的位移操作
        B1 = ror64_63(B1); \  // 对 B1 进行 63 位的位移操作
    } while ((void)0, 0)  // 结束 do-while 循环

#define DIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1) \  // 定义一个宏，接受八个 128 位整数参数
    do { \  // 开始一个 do-while 循环
        __m128i t0 = _mm_alignr_epi8(B1, B0, 8); \  // 对 B1 和 B0 进行字节对齐操作
        __m128i t1 = _mm_alignr_epi8(B0, B1, 8); \  // 对 B0 和 B1 进行字节对齐操作
        B0 = t0; \  // 将 t0 赋值给 B0
        B1 = t1; \  // 将 t1 赋值给 B1
#define UNDIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1) \ 
    do { \  # 定义宏UNDIAGONALIZE，接受8个参数，并开始一个do-while循环
        __m128i t0 = _mm_alignr_epi8(B0, B1, 8); \  # 使用SSE3指令_mm_alignr_epi8对B0和B1进行字节对齐操作，结果存储在t0中
        __m128i t1 = _mm_alignr_epi8(B1, B0, 8); \  # 使用SSE3指令_mm_alignr_epi8对B1和B0进行字节对齐操作，结果存储在t1中
        B0 = t0; \  # 将t0的值赋给B0
        B1 = t1; \  # 将t1的值赋给B1

        t0 = _mm_alignr_epi8(D0, D1, 8); \  # 使用SSE3指令_mm_alignr_epi8对D0和D1进行字节对齐操作，结果存储在t0中
        t1 = _mm_alignr_epi8(D1, D0, 8); \  # 使用SSE3指令_mm_alignr_epi8对D1和D0进行字节对齐操作，结果存储在t1中
        D0 = t1; \  # 将t1的值赋给D0
        D1 = t0; \  # 将t0的值赋给D1
    } while ((void)0, 0)  # 结束do-while循环，使用逗号操作符返回0

#define BLAKE2_ROUND(A0, A1, B0, B1, C0, C1, D0, D1) \  # 定义宏BLAKE2_ROUND，接受8个参数，并开始一个do-while循环
    do { \  # 开始一个do-while循环
        G1(A0, B0, C0, D0, A1, B1, C1, D1); \  # 调用G1函数，传入8个参数
        G2(A0, B0, C0, D0, A1, B1, C1, D1); \  # 调用G2函数，传入8个参数

        DIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1); \  # 调用DIAGONALIZE宏，传入8个参数

        G1(A0, B0, C1, D0, A1, B1, C0, D1); \  # 调用G1函数，传入8个参数
        G2(A0, B0, C1, D0, A1, B1, C0, D1); \  # 调用G2函数，传入8个参数

        UNDIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1); \  # 调用UNDIAGONALIZE宏，传入8个参数
    } while ((void)0, 0)  # 结束do-while循环，使用逗号操作符返回0

#include "argon2-template-128.h"  # 包含头文件argon2-template-128.h

void xmrig_ar2_fill_segment_ssse3(const argon2_instance_t *instance, argon2_position_t position)  # 定义函数xmrig_ar2_fill_segment_ssse3，接受两个参数
{  # 函数体开始
    fill_segment_128(instance, position);  # 调用fill_segment_128函数，传入两个参数
}  # 函数体结束

extern int cpu_flags_has_ssse3(void);  # 声明函数cpu_flags_has_ssse3，无参数，返回整型
int xmrig_ar2_check_ssse3(void) { return cpu_flags_has_ssse3(); }  # 定义函数xmrig_ar2_check_ssse3，返回cpu_flags_has_ssse3函数的结果

#else  # 如果前面的条件不成立，执行以下代码

void xmrig_ar2_fill_segment_ssse3(const argon2_instance_t *instance, argon2_position_t position) {}  # 定义函数xmrig_ar2_fill_segment_ssse3，无操作
int xmrig_ar2_check_ssse3(void) { return 0; }  # 定义函数xmrig_ar2_check_ssse3，返回0
#endif  # 结束条件编译
```