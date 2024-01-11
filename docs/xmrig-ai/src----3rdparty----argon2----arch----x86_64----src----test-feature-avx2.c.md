# `xmrig\src\3rdparty\argon2\arch\x86_64\src\test-feature-avx2.c`

```
# 包含 x86 指令集的头文件
#include <x86intrin.h>

# 定义一个使用 AVX2 指令集的函数，对两个 __m256i 类型的参数进行按位异或操作，并将结果存储到目标参数中
void function_avx2(__m256i *dst, const __m256i *a, const __m256i *b)
{
    *dst = _mm256_xor_si256(*a, *b);
}

# 主函数，返回整型值
int main(void) { return 0; }
```