# `nmap\libssh2\os400\include\stdio.h`

```cpp
/*
 * 版权声明
 * 版权所有（C）2015 Patrick Monnerat, D+H <patrick.monnerat@dh.com>
 *
 * 在源代码和二进制形式下重新分发和使用，无论是否经过修改，都是允许的，前提是满足以下条件：
 *   1. 源代码的再分发必须保留上述版权声明、此条件列表和以下免责声明。
 *   2. 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、此条件列表和以下免责声明。
 *   3. 未经特定事先书面许可，不得使用版权所有者的名称或其他贡献者的名称来认可或推广从本软件衍生的产品。
 *
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）产生的任何理论，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责。
 */

#ifndef LIBSSH2_STDIO_H
#define LIBSSH2_STDIO_H

/*
 * <stdio.h> 包装器。
 * 其目标是重新定义 snprintf/vsnprintf，因为 QADRT 不支持这些函数。
 */

#include <qadrt.h>

#if __ILEC400_TGTVRM__ >= 710
# include_next <stdio.h>
#elif __ILEC400_TGTVRM__ >= 510
# ifndef __SRCSTMF__
#  include <QADRT/h/stdio>
# else
// 包含标准输入输出头文件
#  include </QIBM/ProdData/qadrt/include/stdio.h>
# endif
#endif

// 声明 OS/400 平台下的 vsnprintf 函数
extern int  _libssh2_os400_vsnprintf(char *dst, size_t len,
                                     const char *fmt, va_list args);
// 声明 OS/400 平台下的 snprintf 函数
extern int  _libssh2_os400_snprintf(char *dst, size_t len,
                                    const char *fmt, ...);

// 如果未禁用 QADRT 扩展，则定义 vsnprintf 和 snprintf 宏
#ifndef LIBSSH2_DISABLE_QADRT_EXT
# define vsnprintf(dst, len, fmt, args)                                     \
                        _libssh2_os400_vsnprintf((dst), (len), (fmt), (args))
# define snprintf       _libssh2_os400_snprintf
#endif

#endif

/* vim: set expandtab ts=4 sw=4: */
```