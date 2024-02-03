# `xmrig\src\3rdparty\argon2\arch\x86_64\src\test-feature-sse2.c`

```cpp
# 包含 x86 平台的 SSE2 指令集头文件
#include <x86intrin.h>

# 定义一个使用 SSE2 指令集的函数，计算两个 __m128i 类型的参数的按位异或结果，并存储到目标参数中
void function_sse2(__m128i *dst, const __m128i *a, const __m128i *b)
{
    *dst = _mm_xor_si128(*a, *b);
}

# 主函数
int main(void) { return 0; }
```