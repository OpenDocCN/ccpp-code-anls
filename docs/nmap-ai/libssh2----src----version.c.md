# `nmap\libssh2\src\version.c`

```cpp
/*
  版权声明，版权所有，禁止未经许可的再分发和修改
  在源代码和二进制形式下允许再分发和修改，但需要满足以下条件：
  - 源代码的再分发必须保留上述版权声明、条件列表和下面的免责声明
  - 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和下面的免责声明
  - 未经特定书面许可，不得使用版权所有者的名称或其他贡献者的名称来认可或推广从本软件衍生的产品
  本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的适用性的担保。无论在任何情况下，版权所有者或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性的损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失，或业务中断）负责，无论是在合同、严格责任或侵权行为（包括疏忽或其他方式）的任何理论下，即使已被告知可能发生此类损害的可能性。
*/

#include "libssh2_priv.h"

/*
  libssh2_version() 的使用示例：
  if (!libssh2_version(LIBSSH2_VERSION_NUM)) {
    fprintf (stderr, "Runtime libssh2 version too old!\n");
    exit(1);
  }
*/
LIBSSH2_API
const char *libssh2_version(int req_version_num)
{
    // 如果请求的版本号小于等于当前版本号，返回当前版本号
    if(req_version_num <= LIBSSH2_VERSION_NUM)
        return LIBSSH2_VERSION;
    # 返回空值，表示这个库不适用！
    return NULL; /* this is not a suitable library! */
# 闭合前面的函数定义
```