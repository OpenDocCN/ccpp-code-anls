# `xmrig\src\crypto\randomx\instructions_portable.cpp`

```
/*
版权所有 (c) 2018-2019, tevador <tevador@gmail.com>

保留所有权利。

在源代码和二进制形式下，无论是否修改，只要满足以下条件，就可以重新分发和使用：
    * 必须保留上述版权声明、此条件列表和以下免责声明。
    * 在二进制形式下，必须在文档和/或其他提供的材料中复制上述版权声明、此条件列表和以下免责声明。
    * 未经特定事先书面许可，不得使用版权持有人的名称或贡献者的名称来认可或推广从本软件派生的产品。

本软件由版权持有人和贡献者 "按原样" 提供，任何明示或暗示的保证，包括但不限于对适销性和适用性的暗示保证，都被拒绝。无论在任何情况下，版权持有人或贡献者都不对任何直接、间接、偶然、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润损失，或业务中断）承担任何责任，无论是因合同、严格责任还是侵权行为（包括疏忽或其他）而引起的，即使事先被告知此类损害的可能性。
*/



#include <cfenv> // 引入cfenv库，用于处理浮点数异常
#include <cmath> // 引入cmath库，用于数学计算
#include "crypto/randomx/common.hpp" // 引入common.hpp文件，用于随机数生成
#include "crypto/randomx/intrin_portable.h" // 引入intrin_portable.h文件，用于处理可移植的底层指令
#include "crypto/randomx/blake2/endian.h" // 引入endian.h文件，用于处理字节序

#if defined(__SIZEOF_INT128__)
    typedef unsigned __int128 uint128_t; // 定义unsigned __int128类型为uint128_t
    typedef __int128 int128_t; // 定义__int128类型为int128_t
    uint64_t mulh(uint64_t a, uint64_t b) { // 定义mulh函数，用于计算两个64位无符号整数的乘积的高64位
        return ((uint128_t)a * b) >> 64; // 返回两个64位无符号整数的乘积的高64位
    }
    int64_t smulh(int64_t a, int64_t b) { // 定义smulh函数，用于计算两个64位有符号整数的乘积的高64位
        return ((int128_t)a * b) >> 64; // 返回两个64位有符号整数的乘积的高64位
    }
    # 定义宏HAVE_MULH，表示系统支持无符号整数乘法的高位结果
    #define HAVE_MULH
    
    # 定义宏HAVE_SMULH，表示系统支持有符号整数乘法的高位结果
    #define HAVE_SMULH
    
    
    这段代码是在定义两个宏，分别表示系统是否支持无符号整数乘法的高位结果和有符号整数乘法的高位结果。
#endif

#if defined(_MSC_VER)
    #define HAS_VALUE(X) X ## 0  // 定义宏，用于连接参数和数字0
    #define EVAL_DEFINE(X) HAS_VALUE(X)  // 定义宏，用于评估参数
    #include <intrin.h>  // 包含内联汇编函数的头文件
    #include <stdlib.h>  // 包含标准库函数的头文件

    uint64_t rotl64(uint64_t x, unsigned int c) {  // 定义64位左循环移位函数
        return _rotl64(x, c);  // 调用内联汇编函数执行左循环移位操作
    }
    uint64_t rotr64(uint64_t x, unsigned int c) {  // 定义64位右循环移位函数
        return _rotr64(x, c);  // 调用内联汇编函数执行右循环移位操作
    }
    #define HAVE_ROTL64  // 定义宏，表示已经有了64位左循环移位函数
    #define HAVE_ROTR64  // 定义宏，表示已经有了64位右循环移位函数

    #if EVAL_DEFINE(__MACHINEARM64_X64(1))  // 评估参数是否为ARM64或X64
        uint64_t mulh(uint64_t a, uint64_t b) {  // 定义64位乘法高位函数
            return __umulh(a, b);  // 调用内联汇编函数执行64位乘法高位操作
        }
        #define HAVE_MULH  // 定义宏，表示已经有了64位乘法高位函数
    #endif

    #if EVAL_DEFINE(__MACHINEX64(1))  // 评估参数是否为X64
        int64_t smulh(int64_t a, int64_t b) {  // 定义有符号64位乘法高位函数
            int64_t hi;
            _mul128(a, b, &hi);  // 调用内联汇编函数执行64位乘法操作
            return hi;  // 返回乘法结果的高位
        }
        #define HAVE_SMULH  // 定义宏，表示已经有了有符号64位乘法高位函数
    #endif

    static void setRoundMode_(uint32_t mode) {  // 定义设置舍入模式的函数
        _controlfp(mode, _MCW_RC);  // 调用内联汇编函数设置浮点数舍入模式
    }
    #define HAVE_SETROUNDMODE_IMPL  // 定义宏，表示已经有了设置舍入模式的函数
#endif

#ifndef HAVE_ROTR64  // 如果没有64位右循环移位函数
    uint64_t rotr64(uint64_t a, unsigned int b) {  // 定义64位右循环移位函数
        return (a >> b) | (a << (-b & 63));  // 执行64位右循环移位操作
    }
    #define HAVE_ROTR64  // 定义宏，表示已经有了64位右循环移位函数
#endif

#ifndef HAVE_ROTL64  // 如果没有64位左循环移位函数
    uint64_t rotl64(uint64_t a, unsigned int b) {  // 定义64位左循环移位函数
        return (a << b) | (a >> (-b & 63));  // 执行64位左循环移位操作
    }
    #define HAVE_ROTL64  // 定义宏，表示已经有了64位左循环移位函数
#endif

#ifndef HAVE_MULH  // 如果没有64位乘法高位函数
    #define LO(x) ((x)&0xffffffff)  // 定义宏，获取64位数的低32位
    #define HI(x) ((x)>>32)  // 定义宏，获取64位数的高32位
    uint64_t mulh(uint64_t a, uint64_t b) {  // 定义64位乘法高位函数
        uint64_t ah = HI(a), al = LO(a);  // 获取a的高32位和低32位
        uint64_t bh = HI(b), bl = LO(b);  // 获取b的高32位和低32位
        uint64_t x00 = al * bl;  // 计算a的低32位和b的低32位的乘积
        uint64_t x01 = al * bh;  // 计算a的低32位和b的高32位的乘积
        uint64_t x10 = ah * bl;  // 计算a的高32位和b的低32位的乘积
        uint64_t x11 = ah * bh;  // 计算a的高32位和b的高32位的乘积
        uint64_t m1 = LO(x10) + LO(x01) + HI(x00);  // 计算中间结果m1
        uint64_t m2 = HI(x10) + HI(x01) + LO(x11) + HI(m1);  // 计算中间结果m2
        uint64_t m3 = HI(x11) + HI(m2);  // 计算中间结果m3

        return (m3 << 32) + LO(m2);  // 返回64位乘法高位的结果
    }
    #define HAVE_MULH  // 定义宏，表示已经有了64位乘法高位函数
#endif

#ifndef HAVE_SMULH  // 如果没有有符号64位乘法高位函数
    int64_t smulh(int64_t a, int64_t b) {  // 定义有符号64位乘法高位函数
        int64_t hi = mulh(a, b);  // 调用64位乘法高位函数获取结果
        if (a < 0LL) hi -= b;  // 如果a为负数，则减去b
        if (b < 0LL) hi -= a;  // 如果b为负数，则减去a
        return hi;  // 返回有符号64位乘法高位的结果
    }
    #define HAVE_SMULH  // 定义宏，表示已经有了有符号64位乘法高位函数
#endif

#ifdef RANDOMX_DEFAULT_FENV
# 如果没有定义 HAVE_SETROUNDMODE_IMPL，则定义一个名为 setRoundMode_ 的静态函数，用于设置浮点数的舍入模式
static void setRoundMode_(uint32_t mode) {
    # 使用 fesetround 函数设置浮点数的舍入模式
    fesetround(mode);
}

# 重置浮点数状态，将舍入模式设置为最接近模式，并设置双精度精度为 53 位（如果平台需要）
void rx_reset_float_state() {
    # 调用 setRoundMode_ 函数，将舍入模式设置为最接近模式
    setRoundMode_(FE_TONEAREST);
    # 调用 rx_set_double_precision 函数，将双精度精度设置为 53 位（如果平台需要）
    rx_set_double_precision();
}

# 设置浮点数的舍入模式
void rx_set_rounding_mode(uint32_t mode) {
    # 根据传入的模式值进行判断
    switch (mode & 3) {
    # 如果模式是 RoundDown，则调用 setRoundMode_ 函数，将舍入模式设置为向下舍入
    case RoundDown:
        setRoundMode_(FE_DOWNWARD);
        break;
    # 如果模式是 RoundUp，则调用 setRoundMode_ 函数，将舍入模式设置为向上舍入
    case RoundUp:
        setRoundMode_(FE_UPWARD);
        break;
    # 如果模式是 RoundToZero，则调用 setRoundMode_ 函数，将舍入模式设置为朝零舍入
    case RoundToZero:
        setRoundMode_(FE_TOWARDZERO);
        break;
    # 如果模式是 RoundToNearest，则调用 setRoundMode_ 函数，将舍入模式设置为最接近模式
    case RoundToNearest:
        setRoundMode_(FE_TONEAREST);
        break;
    # 如果模式不是上述四种情况，则表示代码逻辑错误，输出错误信息
    default:
        UNREACHABLE;
    }
}

#endif

#ifdef RANDOMX_USE_X87

#ifdef _M_IX86

# 设置双精度精度为 53 位（如果平台需要）
void rx_set_double_precision() {
    _control87(_PC_53, _MCW_PC);
}

#elif defined(__i386)

# 设置双精度精度为 53 位（如果平台需要）
void rx_set_double_precision() {
    # 定义一个名为 x87cw 的 16 位无符号整数变量
    uint16_t volatile x87cw;
    # 使用汇编指令将 x87cw 的值设置为当前 x87 控制字的值
    asm volatile("fstcw %0" : "=m" (x87cw));
    # 将 x87cw 的第 9、10 位清零
    x87cw &= ~0x300;
    # 将 x87cw 的第 9 位设置为 1
    x87cw |= 0x200;
    # 使用汇编指令将当前 x87 控制字的值设置为 x87cw 的值
    asm volatile("fldcw %0" : : "m" (x87cw));
}

#endif

#endif //RANDOMX_USE_X87

# 定义一个名为 double_ser_t 的联合体，包含一个双精度浮点数和一个 64 位无符号整数
union double_ser_t {
    double f;
    uint64_t i;
};

# 加载地址 addr 处的 64 位数据，并将其转换为双精度浮点数返回
double loadDoublePortable(const void* addr) {
    # 定义一个 double_ser_t 类型的变量 ds
    double_ser_t ds;
    # 将 ds 的 i 成员设置为调用 load64 函数加载地址 addr 处的 64 位数据
    ds.i = load64(addr);
    # 返回 ds 的 f 成员，即转换后的双精度浮点数
    return ds.f;
}
```