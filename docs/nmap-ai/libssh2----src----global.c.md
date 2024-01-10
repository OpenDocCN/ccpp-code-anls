# `nmap\libssh2\src\global.c`

```
/*
 * 版权声明，版权所有
 * 允许在源代码和二进制形式下重新分发和使用，需满足以下条件：
 * 1. 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明。
 * 2. 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明。
 * 未经特定事先书面许可，不得使用版权所有者的名称或其他贡献者的名称来认可或推广从本软件派生的产品。
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保。
 * 在任何情况下，无论是在合同、严格责任还是侵权（包括疏忽或其他方式）的情况下，版权所有者或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责，即使已被告知可能发生此类损害。
 */

#include "libssh2_priv.h"

// 初始化 libssh2 库的标志和状态
static int _libssh2_initialized = 0;
static int _libssh2_init_flags = 0;

// 初始化 libssh2 库
LIBSSH2_API int
libssh2_init(int flags)
{
    // 如果 libssh2 未初始化且不包含 LIBSSH2_INIT_NO_CRYPTO 标志
    if(_libssh2_initialized == 0 && !(flags & LIBSSH2_INIT_NO_CRYPTO)) {
        // 初始化加密库
        libssh2_crypto_init();
    }

    // 增加 libssh2 初始化计数
    _libssh2_initialized++;
    # 将给定的标志位与_libssh2_init_flags进行按位或操作，并将结果赋值给_libssh2_init_flags
    _libssh2_init_flags |= flags;
    
    # 返回整数值0，表示函数执行成功
    return 0;
# 退出 libssh2 库
LIBSSH2_API void
libssh2_exit(void)
{
    # 如果 libssh2 未初始化，则直接返回
    if(_libssh2_initialized == 0)
        return;

    # 减少 libssh2 初始化计数
    _libssh2_initialized--;

    # 如果 libssh2 初始化计数为 0 且初始化标志不包含 LIBSSH2_INIT_NO_CRYPTO，则退出加密模块
    if(_libssh2_initialized == 0 &&
       !(_libssh2_init_flags & LIBSSH2_INIT_NO_CRYPTO)) {
        libssh2_crypto_exit();
    }

    # 返回
    return;
}

# 如果 libssh2 未初始化，则调用 libssh2_init 进行初始化
void
_libssh2_init_if_needed(void)
{
    if(_libssh2_initialized == 0)
        (void)libssh2_init (0);
}
```