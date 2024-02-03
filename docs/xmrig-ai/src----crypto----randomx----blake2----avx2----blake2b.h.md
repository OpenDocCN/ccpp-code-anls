# `xmrig\src\crypto\randomx\blake2\avx2\blake2b.h`

```cpp
# 如果 BLAKE2_AVX2_BLAKE2B_H 还没有被定义，则定义 BLAKE2_AVX2_BLAKE2B_H
#ifndef BLAKE2_AVX2_BLAKE2B_H
# 包含标准库的头文件，用于定义 size_t 类型
#include <stddef.h>
# 如果是 C++ 代码，则使用 extern "C" 包裹
#if defined(__cplusplus)
extern "C" {
#endif
# 定义一个名为 blake2b_avx2 的函数，接受四个参数：输出指针，输出长度，输入指针，输入长度
int blake2b_avx2(void* out, size_t outlen, const void* in, size_t inlen);
# 如果是 C++ 代码，则结束 extern "C" 块
#if defined(__cplusplus)
}
#endif
# 结束条件编译指令
#endif
```