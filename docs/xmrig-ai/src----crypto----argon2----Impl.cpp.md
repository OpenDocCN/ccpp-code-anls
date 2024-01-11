# `xmrig\src\crypto\argon2\Impl.cpp`

```
/*
 * 以下是 XMRig 的版权声明，列出了各个作者的版权信息
 * 程序遵循 GNU 通用公共许可证发布，允许自由传播和修改
 * 无任何担保，详细条款请参考 GNU 通用公共许可证
 * 如果没有收到许可证的副本，请访问 http://www.gnu.org/licenses/
 */

#include "3rdparty/argon2.h"  // 引入 argon2 头文件
#include "base/tools/String.h"  // 引入 String 类的头文件
#include "crypto/argon2/Impl.h"  // 引入 argon2 实现的头文件

namespace xmrig {
    static bool selected = false;  // 静态布尔变量 selected 初始化为 false
    static String implName;  // 静态 String 对象 implName

} // namespace xmrig

extern "C" {
    // 声明四个外部函数，用于检查不同的 CPU 指令集支持情况
    extern int xmrig_ar2_check_avx512f();
    extern int xmrig_ar2_check_avx2();
    extern int xmrig_ar2_check_ssse3();
    extern int xmrig_ar2_check_sse2();
}

// argon2 实现的 select 函数
bool xmrig::argon2::Impl::select(const String &nameHint, bool benchmark) {
    // 如果 selected 为 false
    if (!selected) {
# 如果定义了__x86_64__或者_M_AMD64，则执行以下代码块
        auto hint = nameHint;  # 声明一个自动变量hint，并将nameHint赋值给它

        if (hint.isEmpty() && !benchmark) {  # 如果hint为空并且不是基准测试模式
            if (xmrig_ar2_check_avx512f()) {  # 检查是否支持AVX-512F指令集
                hint = "AVX-512F";  # 如果支持，则将hint设置为"AVX-512F"
            }
            else if (xmrig_ar2_check_avx2()) {  # 如果不支持AVX-512F，检查是否支持AVX2指令集
                hint = "AVX2";  # 如果支持，则将hint设置为"AVX2"
            }
            else if (xmrig_ar2_check_ssse3()) {  # 如果不支持AVX2，检查是否支持SSSE3指令集
                hint = "SSSE3";  # 如果支持，则将hint设置为"SSSE3"
            }
            else if (xmrig_ar2_check_sse2()) {  # 如果不支持SSSE3，检查是否支持SSE2指令集
                hint = "SSE2";  # 如果支持，则将hint设置为"SSE2"
            }
        }

        if (!hint.isEmpty()) {  # 如果hint不为空
            argon2_select_impl_by_name(hint);  # 根据hint选择实现
        }
#       endif

        selected = true;  # 设置selected为true
        implName = argon2_get_impl_name();  # 获取实现的名称

        return true;  # 返回true
    }

    return false;  # 如果不满足条件，返回false
}


const xmrig::String &xmrig::argon2::Impl::name()
{
    return implName;  # 返回实现的名称
}
```