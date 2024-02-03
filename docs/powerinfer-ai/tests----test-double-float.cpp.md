# `PowerInfer\tests\test-double-float.cpp`

```cpp
// 这些测试可能需要很长时间！
// 它们旨在证明 ggml.c 中各种函数的 double 到 float 转换不会影响结果。
// 这是通过检查所有有限（非 NaN，非无穷大）的浮点数来实现的。

// 取消 NDEBUG 宏定义
#undef NDEBUG
#include <cassert>
// 如果不是 __riscv、__s390__、__ARM_NEON，则包含 immintrin.h 头文件
#if !defined(__riscv) && !defined(__s390__) && !defined(__ARM_NEON)
#include <immintrin.h>
#endif
#include <cmath>
#include <cstdint>
#include <cstring>

// 忽略 -Wdouble-promotion 警告
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wdouble-promotion"

// ggml.c::quantize_row_q4_0_reference
// 内联函数，将 float 类型的 v0 四舍五入为 uint8_t 类型
inline static uint8_t round_orig(float v0) { return ((int8_t) (round(v0))) + 8; }

// ggml.c::ggml_silu_f32
// 内联函数，计算 SILU 函数的原始实现
inline static float silu_orig(float x) {
    return x/(1.0 + exp(-x));
}

// 恢复之前忽略的警告
#pragma GCC diagnostic pop

// ggml.c::quantize_row_q4_0_reference
// 内联函数，将 float 类型的 v0 四舍五入为 uint8_t 类型
inline static uint8_t round_float(float v0) { return (int8_t)roundf(v0) + 8; }

// ggml.c::ggml_silu_f32
// 内联函数，计算 SILU 函数的 float 实现
inline static float silu_float(float x) {
    return x/(1.0f + expf(-x));
}

// 主函数
int main(void) {
    // 初始化 x 为 UINT32_MAX
    uint32_t x = UINT32_MAX;
    // 循环，直到 x 为 0
    do {
        // 定义 float 类型的变量 f，并将 x 的值拷贝给它
        float f;
        memcpy(&f, &x, sizeof(x));
        // 断言：如果 f 不是有限的，或者 round_orig(f) 等于 round_float(f)，则断言成功
        assert(!std::isfinite(f) || (round_orig(f) == round_float(f)));
    } while (x--);

    // 如果定义了 __F16C__ 宏
#ifdef __F16C__
    // GELU 和 SILU 实现使用 FP16 查找表。
    // 在将结果转换为 FP16 后，原始结果和仅使用 float 的结果并不相等。
    // GELU 是一个近似值（tanh），这里不进行测试。
    // 对于 SILU，验证结果至少是最接近的浮点数，如果 FP16 值不匹配。
    for (x = 0; x <= UINT16_MAX; x++) {
        // 将 x 转换为 float 类型的 f
        float f = _cvtsh_ss(x);
        // 计算原始 SILU 函数和 float 实现的结果
        const float so = silu_orig(f);
        const float sf = silu_float(f);
        // 断言：如果 _cvtss_sh(so, 0) 等于 _cvtss_sh(sf, 0)，
        // 或者 nextafterf(so, sf) 等于 sf，或者 nextafterf(sf, so) 等于 so，则断言成功
        assert(   (_cvtss_sh(so, 0) == _cvtss_sh(sf, 0))
               || (nextafterf(so, sf) == sf)
               || (nextafterf(sf, so) == so));
    }
#endif
}
```