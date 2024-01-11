# `xmrig\src\3rdparty\argon2\arch\x86_64\src\test-feature-avx512f.c`

```
# 包含 x86 指令集的头文件
#include <x86intrin.h>

# 定义一个函数，使用 AVX-512 指令集对输入的 __m512i 类型数据进行操作
void function_avx512f(__m512i *dst, const __m512i *a)
{
    # 使用 AVX-512 指令对输入数据进行 64 位元素级别的循环右移操作
    *dst = _mm512_ror_epi64(*a, 57);
}

# 主函数
int main(void) { return 0; }
```