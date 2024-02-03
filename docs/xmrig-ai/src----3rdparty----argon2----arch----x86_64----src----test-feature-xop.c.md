# `xmrig\src\3rdparty\argon2\arch\x86_64\src\test-feature-xop.c`

```cpp
# 包含 x86 指令集的头文件
#include <x86intrin.h>

# 定义一个函数 function_xop，接受一个指向 __m128i 类型的指针 dst，一个指向 __m128i 类型的指针 a，一个整型变量 b
void function_xop(__m128i *dst, const __m128i *a, int b)
{
    # 使用 _mm_roti_epi64 函数对 a 指向的数据进行 64 位整数的循环左移操作，结果存入 dst 指向的位置
    *dst = _mm_roti_epi64(*a, b);
}

# 主函数
int main(void) { return 0; }
```