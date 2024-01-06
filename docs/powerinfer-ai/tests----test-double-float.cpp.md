# `PowerInfer\tests\test-double-float.cpp`

```
// 这些测试可能需要很长时间！
// 它们旨在证明 ggml.c 中各种函数的 double 转换为 float 不会影响结果。
// 这是通过检查所有有限（非 NaN，非无穷大）的浮点数来实现的。

// 取消 NDEBUG 宏定义，启用断言
#undef NDEBUG
#include <cassert>
// 如果不是 RISC-V、s390 或 ARM NEON 架构，包含 immintrin.h 头文件
#if !defined(__riscv) && !defined(__s390__) && !defined(__ARM_NEON)
#include <immintrin.h>
#endif
// 包含数学函数头文件
#include <cmath>
// 包含整型数据类型头文件
#include <cstdint>
// 包含字符串处理头文件
#include <cstring>

// 忽略编译器关于 double 提升的警告
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wdouble-promotion"

// 定义 ggml.c::quantize_row_q4_0_reference 函数的内联版本，将 float 类型参数四舍五入为 uint8_t 类型
inline static uint8_t round_orig(float v0) { return ((int8_t) (round(v0))) + 8; }

// 定义 ggml.c::ggml_silu_f32 函数
// 定义一个静态内联函数，实现原始的Sigmoid激活函数
inline static float silu_orig(float x) {
    return x/(1.0 + exp(-x));
}

// 取消之前的GCC诊断设置

// 定义一个静态内联函数，实现四舍五入的功能，并返回一个8位无符号整数
inline static uint8_t round_float(float v0) { return (int8_t)roundf(v0) + 8; }

// 定义一个静态内联函数，实现浮点数版本的Sigmoid激活函数
inline static float silu_float(float x) {
    return x/(1.0f + expf(-x));
}

// 主函数
int main(void) {
    // 初始化一个32位无符号整数，并赋值为最大值
    uint32_t x = UINT32_MAX;
    // 循环开始
    do {
        // 定义一个浮点数变量f，并将x的值拷贝到f中
        float f;
        memcpy(&f, &x, sizeof(x));
        // 断言：f不是有限的，或者原始四舍五入函数的结果等于浮点数四舍五入函数的结果
        assert(!std::isfinite(f) || (round_orig(f) == round_float(f)));
    } while (x--);
    // 使用 do-while 循环，先执行一次循环体，然后检查条件是否满足，如果满足则继续执行循环体

#ifdef __F16C__
    // 如果定义了 __F16C__ 宏，则执行以下代码块
    // GELU 和 SILU 实现使用 FP16 查找表
    // 在将结果转换为 FP16 后，原始结果和仅使用浮点数的结果并不相等
    // GELU 本身就是一个近似值（tanh），这里没有进行测试
    // 对于 SILU，验证结果至少是最接近的浮点数，如果 FP16 值不匹配
    for (x = 0; x <= UINT16_MAX; x++) {
        // 将 x 转换为单精度浮点数
        float f = _cvtsh_ss(x);
        // 计算原始 SILU 函数的结果
        const float so = silu_orig(f);
        // 计算使用浮点数的 SILU 函数的结果
        const float sf = silu_float(f);
        // 断言：原始结果和浮点数结果相等，或者浮点数结果是最接近的浮点数之一
        assert(   (_cvtss_sh(so, 0) == _cvtss_sh(sf, 0))
               || (nextafterf(so, sf) == sf)
               || (nextafterf(sf, so) == so));
    }
#endif
}
```