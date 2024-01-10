# `nmap\libssh2\src\comp.h`

```
#ifndef __LIBSSH2_COMP_H
#define __LIBSSH2_COMP_H
/* 定义了一个条件编译，如果__LIBSSH2_COMP_H未定义，则定义__LIBSSH2_COMP_H */
/* 以下是版权声明和许可协议，允许在源代码和二进制形式下进行再发布和使用 */
/* 如果再发布源代码，需要保留版权声明、条件列表和免责声明 */
/* 如果再发布二进制形式，需要在文档和/或其他提供的材料中重现版权声明、条件列表和免责声明 */
/* 未经特定事先书面许可，不得使用版权所有者的名称或其他贡献者的名称来认可或推广从本软件派生的产品 */
/* 版权所有者和贡献者提供的本软件是"按原样"提供的，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保 */
/* 在任何情况下，无论是在合同、严格责任还是侵权（包括疏忽或其他方式）的情况下，版权所有者或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责 */
/* 即使已被告知可能发生此类损害，也不得在任何责任理论下对使用本软件产生的任何方式负责 */
#include "libssh2_priv.h"
/* 引入libssh2_priv.h头文件 */

const LIBSSH2_COMP_METHOD **_libssh2_comp_methods(LIBSSH2_SESSION *session);
/* 声明一个返回LIBSSH2_COMP_METHOD指针数组的函数，函数名为_libssh2_comp_methods，参数为LIBSSH2_SESSION指针 */
#endif /* __LIBSSH2_COMP_H */
/* 结束条件编译，如果__LIBSSH2_COMP_H未定义，则定义__LIBSSH2_COMP_H */
```