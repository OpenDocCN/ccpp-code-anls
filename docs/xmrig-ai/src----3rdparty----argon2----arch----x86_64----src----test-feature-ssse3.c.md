# `xmrig\src\3rdparty\argon2\arch\x86_64\src\test-feature-ssse3.c`

```
#include <x86intrin.h>  // 包含 x86 SIMD 指令集的头文件

void function_ssse3(__m128i *dst, const __m128i *a, const __m128i *b)
{
    *dst = _mm_shuffle_epi8(*a, *b);  // 使用 SSSE3 指令对 a 和 b 进行按位混洗，并将结果存储到 dst 中
}

int main(void) { return 0; }  // 主函数，返回 0 表示正常退出
```