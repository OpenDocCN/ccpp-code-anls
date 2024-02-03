# `nmap\nping\Crypto.h`

```cpp
#ifndef __CRYPTO_H__
#define __CRYPTO_H__ 1
#include "nping.h"

#define HMAC_SHA256_CODE_LEN 32
#define AES_BLOCK_SIZE 16
#define AES_KEY_SIZE 16
#define SHA256_HASH_LEN 32

class Crypto {

    public:

        Crypto(); // 构造函数
        ~Crypto(); // 析构函数
        void reset(); // 重置函数

        static int hmac_sha256(u8 *inbuff, size_t inlen, u8 *dst_buff, u8 *key, size_t key_len); // HMAC-SHA256 加密函数
        static int aes128_cbc_encrypt(u8 *inbuff, size_t inlen, u8 *dst_buff, u8 *key, size_t key_len, u8 *iv); // AES-128-CBC 加密函数
        static int aes128_cbc_decrypt(u8 *inbuff, size_t inlen, u8 *dst_buff, u8 *key, size_t key_len, u8 *iv); // AES-128-CBC 解密函数
        static int generateNonce(u8 *dst_buff, size_t bufflen); // 生成随机数函数
        static u8 *deriveKey(const u8 *from, size_t fromlen, size_t *final_len); // 导出密钥函数
};

#endif /* __CRYPTO_H__ */
```