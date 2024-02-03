# `nmap\libpcap\fmtutils.h`

```cpp
/*
 * 版权声明，版权归加利福尼亚大学理事会所有
 * 允许以源代码或二进制形式进行再发布和使用，需满足以下条件：
 * 1. 源代码的再发布必须保留上述版权声明、条件列表和以下免责声明
 * 2. 二进制形式的再发布必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 * 3. 所有提及此软件特性或使用的广告材料必须显示以下声明：
 *    本产品包含由劳伦斯伯克利实验室的计算机系统工程组开发的软件
 * 4. 未经特定事先书面许可，不得使用大学或实验室的名称来认可或推广由此软件衍生的产品
 *
 * 本软件由理事会和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。在任何情况下，理事会或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性的损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）承担责任，无论是合同责任、严格责任还是侵权行为（包括疏忽或其他方式）引起的，即使已被告知可能发生此类损害。
 */

#ifndef fmtutils_h
#define    fmtutils_h

#include <stdarg.h>    /* 声明可变参数函数 */

#include "pcap/funcattrs.h"

#ifdef __cplusplus
#ifdef __cplusplus
}
#endif

// 如果是 C++ 环境，结束 extern "C" 块


#endif

// 结束 extern "C" 块


void    pcap_fmt_set_encoding(unsigned int);

// 声明一个名为 pcap_fmt_set_encoding 的函数，接受一个无符号整数参数


void    pcap_fmt_errmsg_for_errno(char *, size_t, int, PCAP_FORMAT_STRING(const char *), ...) PCAP_PRINTFLIKE(4, 5);

// 声明一个名为 pcap_fmt_errmsg_for_errno 的函数，接受一个字符指针、一个 size_t 类型的整数、一个整数和一个格式化字符串作为参数，以及可变参数列表


void    pcap_vfmt_errmsg_for_errno(char *, size_t, int, PCAP_FORMAT_STRING(const char *), va_list) PCAP_PRINTFLIKE(4, 0);

// 声明一个名为 pcap_vfmt_errmsg_for_errno 的函数，接受一个字符指针、一个 size_t 类型的整数、一个整数、一个格式化字符串和一个 va_list 类型的参数列表作为参数


#ifdef _WIN32
void    pcap_fmt_errmsg_for_win32_err(char *, size_t, DWORD, PCAP_FORMAT_STRING(const char *), ...) PCAP_PRINTFLIKE(4, 5);

// 如果是在 Windows 环境下，声明一个名为 pcap_fmt_errmsg_for_win32_err 的函数，接受一个字符指针、一个 size_t 类型的整数、一个 DWORD 类型的整数、一个格式化字符串和可变参数列表作为参数


void    pcap_vfmt_errmsg_for_win32_err(char *, size_t, DWORD, PCAP_FORMAT_STRING(const char *), va_list) PCAP_PRINTFLIKE(4, 0);

// 如果是在 Windows 环境下，声明一个名为 pcap_vfmt_errmsg_for_win32_err 的函数，接受一个字符指针、一个 size_t 类型的整数、一个 DWORD 类型的整数、一个格式化字符串和一个 va_list 类型的参数列表作为参数


#endif

// 结束 Windows 环境下的条件编译块


#endif

// 结束头文件的条件编译块
```