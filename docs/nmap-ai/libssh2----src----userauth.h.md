# `nmap\libssh2\src\userauth.h`

```
#ifndef __LIBSSH2_USERAUTH_H
#define __LIBSSH2_USERAUTH_H
/* 定义宏，用于条件编译，如果__LIBSSH2_USERAUTH_H未定义，则定义__LIBSSH2_USERAUTH_H */
/* 版权声明，版权归属于Sara Golemon <sarag@libssh2.org>和Daniel Stenberg，保留所有权利 */
/* 允许在源代码和二进制形式下重新分发和使用，需满足以下条件 */
/* 在源代码的重新分发中，需保留上述版权声明、条件列表和以下免责声明 */
/* 在二进制形式的重新分发中，需在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明 */
/* 不得使用版权所有者的名称或其他贡献者的名称，未经特定事先书面许可，不得用于认可或推广从本软件衍生的产品 */
/* 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保，不得因使用本软件而产生任何直接、间接、附带、特殊、惩罚性或后果性的损害赔偿责任，无论是合同责任、严格责任还是侵权行为（包括疏忽或其他），即使已被告知可能发生此类损害的可能性 */
int
# 使用公钥进行用户身份验证
_libssh2_userauth_publickey(
    LIBSSH2_SESSION *session,  # SSH会话对象
    const char *username,  # 用户名
    unsigned int username_len,  # 用户名长度
    const unsigned char *pubkeydata,  # 公钥数据
    unsigned long pubkeydata_len,  # 公钥数据长度
    LIBSSH2_USERAUTH_PUBLICKEY_SIGN_FUNC ((*sign_callback)),  # 公钥签名回调函数
    void *abstract  # 抽象数据
);

#endif /* __LIBSSH2_USERAUTH_H */
```