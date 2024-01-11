# `xmrig\src\crypto\randomx\blake2\blamka-round-ref.h`

```
# 版权声明
# 本代码的版权归 tevador <tevador@gmail.com> 所有
# 禁止未经许可的修改、分发和使用
# 如果使用了本代码，必须保留版权声明和许可条件
# 不得使用版权持有人或贡献者的名字来推广派生产品
# 本软件按原样提供，不提供任何明示或暗示的担保
# 版权持有人或贡献者不对任何直接、间接、附带、特殊、惩罚性或后果性损害负责
# 无论是合同、严格责任还是侵权行为，即使事先被告知可能发生此类损害，也不负责任

# 引用的原始代码来自于 Argon2 参考源代码包，使用 CC0 许可证
# https://github.com/P-H-C/phc-winner-argon2
# 版权归 2015 年的 Daniel Dinu、Dmitry Khovratovich、Jean-Philippe Aumasson 和 Samuel Neves 所有

# 定义 BLAKE_ROUND_MKA_H 宏，用于条件编译
#ifndef BLAKE_ROUND_MKA_H
#define BLAKE_ROUND_MKA_H

# 引用所需的头文件
#include "crypto/randomx/blake2/blake2.h"
#include "crypto/randomx/blake2/blake2-impl.h"

# 由 Lyra PHC 团队设计
// 定义一个静态的内联函数，用于执行特定的位运算操作
static FORCE_INLINE uint64_t fBlaMka(uint64_t x, uint64_t y) {
    // 定义一个64位的掩码
    const uint64_t m = UINT64_C(0xFFFFFFFF);
    // 对x和y进行位与运算，并将结果乘以掩码，得到xy的值
    const uint64_t xy = (x & m) * (y & m);
    // 返回x + y + 2 * xy的结果
    return x + y + 2 * xy;
}

// 定义一个宏，用于执行BLAKE2算法的一个轮次的操作
#define G(a, b, c, d)                                                          \
    do {                                                                       \
        // 调用fBlaMka函数，对a和b进行特定的位运算操作
        a = fBlaMka(a, b);                                                     \
        // 对d和a进行异或运算，然后将结果向右循环移动32位
        d = rotr64(d ^ a, 32);                                                 \
        // 调用fBlaMka函数，对c和d进行特定的位运算操作
        c = fBlaMka(c, d);                                                     \
        // 对b和c进行异或运算，然后将结果向右循环移动24位
        b = rotr64(b ^ c, 24);                                                 \
        // 以下类似的操作，依次调用fBlaMka函数和rotr64函数，对a、b、c、d进行位运算操作
        a = fBlaMka(a, b);                                                    
        d = rotr64(d ^ a, 16);                                                 
        c = fBlaMka(c, d);                                                    
        b = rotr64(b ^ c, 63);                                                 
    } while ((void)0, 0)

// 定义一个宏，用于执行BLAKE2算法的多个轮次的操作
#define BLAKE2_ROUND_NOMSG(v0, v1, v2, v3, v4, v5, v6, v7, v8, v9, v10, v11,   \
                           v12, v13, v14, v15)                                 \
    do {                                                                       \
        // 依次调用G宏，对v0到v15进行BLAKE2算法的轮次操作
        G(v0, v4, v8, v12);                                                   
        G(v1, v5, v9, v13);                                                   
        G(v2, v6, v10, v14);                                                  
        G(v3, v7, v11, v15);                                                  
        G(v0, v5, v10, v15);                                                  
        G(v1, v6, v11, v12);                                                  
        G(v2, v7, v8, v13);                                                   
        G(v3, v4, v9, v14);                                                   
    } while ((void)0, 0)

#endif
```