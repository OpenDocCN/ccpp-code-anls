# `xmrig\src\crypto\randomx\reciprocal.c`

```cpp
/*
    版权声明
    版权所有（c）2018-2019，tevador <tevador@gmail.com>

    保留所有权利。

    在源代码和二进制形式中重新分发和使用，无论是否修改，都必须满足以下条件：
    * 源代码的再分发必须保留上述版权声明、此条件列表和以下免责声明。
    * 二进制形式的再分发必须在提供的文档和/或其他材料中重现上述版权声明、此条件列表和以下免责声明。
    * 未经特定事先书面许可，不得使用版权持有人或其贡献者的名称来认可或推广从本软件衍生的产品。

    本软件由版权持有人和贡献者“按原样”提供，任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保，均不承担责任。无论在任何情况下，版权持有人或贡献者均不对任何直接、间接、偶然、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）承担责任，无论是在合同、严格责任或侵权（包括疏忽或其他方式）的任何理论下，即使已被告知可能发生此类损害的可能性。

*/

#include <assert.h>
#include "crypto/randomx/reciprocal.h"

/*
    计算 rcp = 2**x / divisor，其中 x 是最大的整数，使得 rcp < 2**64。
    divisor 不能为 0 或 2 的幂

    等效的 x86 汇编代码（divisor 在 rcx 寄存器中）：

    mov edx, 1
    mov r8, rcx
    xor eax, eax
    bsr rcx, rcx
    shl rdx, cl
    div r8
    ret

*/
uint64_t randomx_reciprocal(uint64_t divisor) {

    // 断言 divisor 不为 0
    assert(divisor != 0);
    # 定义一个常量，值为2的63次方，用于计算商和余数
    const uint64_t p2exp63 = 1ULL << 63;

    # 计算商和余数
    uint64_t quotient = p2exp63 / divisor, remainder = p2exp63 % divisor;

    # 计算除数的最高位
    unsigned bsr = 0; //highest set bit in divisor

    # 循环计算除数的最高位
    for (uint64_t bit = divisor; bit > 0; bit >>= 1)
        bsr++;

    # 进行除法运算，计算商和余数
    for (unsigned shift = 0; shift < bsr; shift++) {
        if (remainder >= divisor - remainder) {
            quotient = quotient * 2 + 1;
            remainder = remainder * 2 - divisor;
        }
        else {
            quotient = quotient * 2;
            remainder = remainder * 2;
        }
    }

    # 返回商
    return quotient;
# 如果没有定义 RANDOMX_HAVE_FAST_RECIPROCAL 宏
#if !RANDOMX_HAVE_FAST_RECIPROCAL
    # 调用 randomx_reciprocal 函数计算除法的倒数，并返回结果
    uint64_t randomx_reciprocal_fast(uint64_t divisor) {
        return randomx_reciprocal(divisor);
    }
# 结束条件编译指令
#endif
```