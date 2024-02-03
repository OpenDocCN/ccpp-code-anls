# `nmap\libssh2\os400\include\sys\socket.h`

```cpp
/*
 * 版权声明
 * 版权所有（C）2015 Patrick Monnerat, D+H <patrick.monnerat@dh.com>
 *
 * 在源代码和二进制形式下重新分发和使用，无论是否经过修改，都是允许的，只要满足以下条件：
 *
 *   1. 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明。
 *   2. 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明。
 *   3. 未经特定事先书面许可，不得使用版权所有者或其他贡献者的名称来认可或推广从本软件衍生的产品。
 *
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）产生的任何理论，版权所有者或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责。
 */

#ifndef LIBSSH2_SYS_SOCKET_H
#define LIBSSH2_SYS_SOCKET_H

/*
 * <sys/socket.h> 包装器
 * 重新定义 connect()
 */

#include <qadrt.h>

#ifndef _QADRT_LT
# define _QADRT_LT <
#endif
#ifndef _QADRT_GT
# define _QADRT_GT >
#endif

#ifdef QADRT_SYSINC
# include _QADRT_LT QADRT_SYSINC/sys/socket.h _QADRT_GT
#elif __ILEC400_TGTVRM__ >= 710
# include_next <sys/socket.h>  # 包含下一个 sys/socket.h 文件
#elif !defined(__SRCSTMF__)  # 如果未定义 __SRCSTMF__ 宏
# include <QSYSINC/sys/socket>  # 包含 QSYSINC/sys/socket 文件
#else  # 否则
# include </QIBM/include/sys/socket.h>  # 包含 /QIBM/include/sys/socket.h 文件
#endif  # 结束条件编译

extern int  _libssh2_os400_connect(int sd,  # 声明 _libssh2_os400_connect 函数，参数为 sd
                                   struct sockaddr * destaddr, int addrlen);  # 结构体 sockaddr 指针和 addrlen 参数

#ifndef LIBSSH2_DISABLE_QADRT_EXT  # 如果未定义 LIBSSH2_DISABLE_QADRT_EXT 宏
#define connect(sd, addr, len)  _libssh2_os400_connect((sd), (addr), (len))  # 定义 connect 宏，调用 _libssh2_os400_connect 函数
#endif  # 结束条件编译

#endif  # 结束条件编译

/* vim: set expandtab ts=4 sw=4: */  # 设置 vim 缩进和制表符
```