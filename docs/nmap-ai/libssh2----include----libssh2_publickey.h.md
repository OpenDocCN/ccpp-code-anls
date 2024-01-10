# `nmap\libssh2\include\libssh2_publickey.h`

```
/* 版权声明，版权所有，保留所有权利 */
/* 在源代码和二进制形式下重新分发和使用，需满足以下条件 */
/* 在源代码中重新分发时，必须保留上述版权声明、条件列表和以下免责声明 */
/* 在二进制形式中重新分发时，必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明 */
/* 未经特定事先书面许可，不得使用版权所有者或其他贡献者的名称来认可或推广从本软件衍生的产品 */
/* 版权所有者和贡献者提供的本软件是"按原样"提供的，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保 */
/* 在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）产生的任何直接、间接、偶发、特殊、惩罚性或后果性损害，版权所有者或贡献者均不承担责任，即使已被告知可能发生此类损害 */

/* 注意：此包含文件仅用于使用公钥子系统，而不是公钥身份验证。对于身份验证，只需要libssh2.h */
/* 有关公钥子系统的更多信息，请参考IETF草案：secsh-publickey */

/* 如果未定义LIBSSH2_PUBLICKEY_H，则定义为1 */
#ifndef LIBSSH2_PUBLICKEY_H
#define LIBSSH2_PUBLICKEY_H 1

/* 包含libssh2.h头文件 */
#include "libssh2.h"
# 定义 LIBSSH2_PUBLICKEY 结构体
typedef struct _LIBSSH2_PUBLICKEY               LIBSSH2_PUBLICKEY;

# 定义公钥属性结构体
typedef struct _libssh2_publickey_attribute {
    const char *name;  # 属性名
    unsigned long name_len;  # 属性名长度
    const char *value;  # 属性值
    unsigned long value_len;  # 属性值长度
    char mandatory;  # 是否必须
} libssh2_publickey_attribute;

# 定义公钥列表结构体
typedef struct _libssh2_publickey_list {
    unsigned char *packet;  # 用于释放内存
    const unsigned char *name;  # 公钥名称
    unsigned long name_len;  # 公钥名称长度
    const unsigned char *blob;  # 公钥数据
    unsigned long blob_len;  # 公钥数据长度
    unsigned long num_attrs;  # 属性数量
    libssh2_publickey_attribute *attrs;  # 属性数组，需要释放内存
} libssh2_publickey_list;

# 定义宏，用于快速创建公钥属性
# 一般使用第一个宏，但如果属性名和属性值都是字符串字面值，可以使用 _fast() 宏来利用预处理
#define libssh2_publickey_attribute(name, value, mandatory) \
  { (name), strlen(name), (value), strlen(value), (mandatory) },
#define libssh2_publickey_attribute_fast(name, value, mandatory) \
  { (name), sizeof(name) - 1, (value), sizeof(value) - 1, (mandatory) },

# 如果是 C++ 环境，则使用 extern "C"
#ifdef __cplusplus
extern "C" {
#endif

# 公钥子系统
# 初始化公钥对象
LIBSSH2_API LIBSSH2_PUBLICKEY *
libssh2_publickey_init(LIBSSH2_SESSION *session);

# 添加公钥，包括名称、数据、是否覆盖、属性数量和属性数组
LIBSSH2_API int
libssh2_publickey_add_ex(LIBSSH2_PUBLICKEY *pkey,
                         const unsigned char *name,
                         unsigned long name_len,
                         const unsigned char *blob,
                         unsigned long blob_len, char overwrite,
                         unsigned long num_attrs,
                         const libssh2_publickey_attribute attrs[]);
# 宏定义，简化添加公钥的调用
#define libssh2_publickey_add(pkey, name, blob, blob_len, overwrite,    \
                              num_attrs, attrs)                         \
  libssh2_publickey_add_ex((pkey), (name), strlen(name), (blob), (blob_len), \
                           (overwrite), (num_attrs), (attrs))
# 定义一个函数，用于从公钥列表中移除指定的公钥
LIBSSH2_API int libssh2_publickey_remove_ex(LIBSSH2_PUBLICKEY *pkey,
                                            const unsigned char *name,
                                            unsigned long name_len,
                                            const unsigned char *blob,
                                            unsigned long blob_len);
# 定义一个宏，用于调用 libssh2_publickey_remove_ex 函数，传入参数 name 的长度
#define libssh2_publickey_remove(pkey, name, blob, blob_len) \
  libssh2_publickey_remove_ex((pkey), (name), strlen(name), (blob), (blob_len))

# 定义一个函数，用于获取公钥列表
LIBSSH2_API int
libssh2_publickey_list_fetch(LIBSSH2_PUBLICKEY *pkey,
                             unsigned long *num_keys,
                             libssh2_publickey_list **pkey_list);
# 定义一个函数，用于释放公钥列表
LIBSSH2_API void
libssh2_publickey_list_free(LIBSSH2_PUBLICKEY *pkey,
                            libssh2_publickey_list *pkey_list);

# 定义一个函数，用于关闭公钥功能
LIBSSH2_API int libssh2_publickey_shutdown(LIBSSH2_PUBLICKEY *pkey);

# 如果是 C++ 环境，则使用 extern "C"
#ifdef __cplusplus
} /* extern "C" */
#endif

# 结束 LIBSSH2_PUBLICKEY_H 的定义
#endif /* ifndef: LIBSSH2_PUBLICKEY_H */
```