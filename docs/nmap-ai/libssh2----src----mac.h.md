# `nmap\libssh2\src\mac.h`

```cpp
#ifndef __LIBSSH2_MAC_H
#define __LIBSSH2_MAC_H
/* 定义宏，避免重复包含该文件 */
/* 定义宏，避免重复包含该文件 */

/* 版权声明 */
/* 版权声明 */

/* 定义结构体_LIBSSH2_MAC_METHOD */
struct _LIBSSH2_MAC_METHOD
{
    const char *name;  /* MAC 方法的名称 */

    /* MAC 数据包的长度 */
    int mac_len;

    /* 完整性密钥长度 */
    int key_len;

    /* 消息认证码哈希算法的初始化函数指针 */
    int (*init) (LIBSSH2_SESSION * session, unsigned char *key, int *free_key,
                 void **abstract);
};
    # 定义一个指向函数的指针，该函数接受多个参数并返回整型值
    int (*hash) (LIBSSH2_SESSION * session, unsigned char *buf,
                 uint32_t seqno, const unsigned char *packet,
                 uint32_t packet_len, const unsigned char *addtl,
                 uint32_t addtl_len, void **abstract);
    # 定义一个指向函数的指针，该函数接受多个参数并返回整型值
    int (*dtor) (LIBSSH2_SESSION * session, void **abstract);
# 结构体声明，用于定义LIBSSH2_MAC_METHOD类型
typedef struct _LIBSSH2_MAC_METHOD LIBSSH2_MAC_METHOD;

# 声明_libssh2_mac_methods函数，返回指向指针数组的指针
const LIBSSH2_MAC_METHOD **_libssh2_mac_methods(void);

# 结束宏定义__LIBSSH2_MAC_H
#endif /* __LIBSSH2_MAC_H */
```