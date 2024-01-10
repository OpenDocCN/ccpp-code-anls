# `nmap\libssh2\os400\libssh2_ccsid.h`

```
/*
 * 版权声明
 * 版权所有（C）2015 Patrick Monnerat, D+H <patrick.monnerat@dh.com>
 *
 * 在源代码和二进制形式下，允许进行重新分发和修改，前提是满足以下条件：
 *   1. 源代码的重新分发必须保留上述版权声明、条件列表和以下免责声明。
 *   2. 二进制形式的重新分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明。
 *   3. 未经特定事先书面许可，不得使用版权所有者的名称或任何其他贡献者的名称来认可或推广从本软件衍生的产品。
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的适用性担保。无论在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）产生的任何损害，版权所有者或贡献者均不对任何直接、间接、附带、特殊、示范性或间接损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责，即使已被告知可能发生此类损害的可能性。
 */

/* CCSID 转换支持 */

#ifndef LIBSSH2_CCSID_H_
#define LIBSSH2_CCSID_H_

#include "libssh2.h"

// 定义 libssh2_string_cache 结构体
typedef struct _libssh2_string_cache    libssh2_string_cache;

// 声明 LIBSSH2_API 宏，返回类型为 char* 的函数
LIBSSH2_API char *
# 从指定的 CCSID 转换字符串为内部编码，并将结果存储在缓存中
libssh2_from_ccsid(LIBSSH2_SESSION *session, libssh2_string_cache **cache,
                   unsigned short ccsid, const char *string, ssize_t inlen,
                   size_t *outlen);
# 从内部编码转换字符串为指定的 CCSID，并将结果存储在缓存中
LIBSSH2_API char *
libssh2_to_ccsid(LIBSSH2_SESSION *session, libssh2_string_cache **cache,
                 unsigned short ccsid, const char *string, ssize_t inlen,
                 size_t *outlen);
# 释放字符串缓存
LIBSSH2_API void
libssh2_release_string_cache(LIBSSH2_SESSION *session,
                             libssh2_string_cache **cache);

#endif

/* vim: set expandtab ts=4 sw=4: */
```