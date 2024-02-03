# `nmap\libssh2\src\wincng.c`

```cpp
/*
 * 版权声明，版权所有
 *
 * 在源代码和二进制形式下，允许进行再发布和修改
 * 前提是满足以下条件：
 *   1. 源代码的再发布必须保留上述版权声明、条件列表和以下免责声明
 *   2. 二进制形式的再发布必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 *   3. 未经特定事先书面许可，不得使用版权所有者的名称或其他贡献者的名称来认可或推广从本软件派生的产品
 *
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，
 * 包括但不限于适销性和特定用途的暗示担保。无论在任何情况下，版权所有者或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责，无论是合同责任、严格责任还是侵权行为（包括疏忽或其他方式）产生的任何理论，即使已被告知可能发生此类损害。
 */

#include "libssh2_priv.h"

#ifdef LIBSSH2_WINCNG /* 只有在使用wincng构建时才编译 */

/* 用于针对w64 mingw-runtime包进行交叉编译所需 */
#if defined(_WIN32_WINNT) && (_WIN32_WINNT < 0x0600)
#undef _WIN32_WINNT
#endif
#ifndef _WIN32_WINNT
#define _WIN32_WINNT 0x0600
#endif
/* specify the required libraries for dependencies using MSVC */
#ifdef _MSC_VER
#pragma comment(lib, "bcrypt.lib")
#ifdef HAVE_LIBCRYPT32
#pragma comment(lib, "crypt32.lib")
#endif
#endif

#include <windows.h>  // 包含 Windows API 头文件
#include <bcrypt.h>  // 包含 Windows Cryptography API 头文件
#include <ntstatus.h>  // 包含 Windows NT 状态码头文件
#include <math.h>  // 包含数学函数头文件
#include "misc.h"  // 包含自定义的杂项函数头文件

#ifdef HAVE_STDLIB_H
#include <stdlib.h>  // 包含标准库头文件
#endif
#ifdef HAVE_LIBCRYPT32
#include <wincrypt.h>  // 包含 Windows 加密 API 头文件
#endif

#define PEM_RSA_HEADER "-----BEGIN RSA PRIVATE KEY-----"  // 定义 PEM RSA 私钥头部
#define PEM_RSA_FOOTER "-----END RSA PRIVATE KEY-----"  // 定义 PEM RSA 私钥尾部
#define PEM_DSA_HEADER "-----BEGIN DSA PRIVATE KEY-----"  // 定义 PEM DSA 私钥头部
#define PEM_DSA_FOOTER "-----END DSA PRIVATE KEY-----"  // 定义 PEM DSA 私钥尾部

/*******************************************************************/
/*
 * Windows CNG backend: Missing definitions (for MinGW[-w64])
 */
#ifndef BCRYPT_SUCCESS
#define BCRYPT_SUCCESS(Status) (((NTSTATUS)(Status)) >= 0)  // 定义 BCRYPT_SUCCESS 宏
#endif

#ifndef BCRYPT_RNG_ALGORITHM
#define BCRYPT_RNG_ALGORITHM L"RNG"  // 定义 BCRYPT 随机数生成算法
#endif

#ifndef BCRYPT_MD5_ALGORITHM
#define BCRYPT_MD5_ALGORITHM L"MD5"  // 定义 BCRYPT MD5 算法
#endif

#ifndef BCRYPT_SHA1_ALGORITHM
#define BCRYPT_SHA1_ALGORITHM L"SHA1"  // 定义 BCRYPT SHA1 算法
#endif

#ifndef BCRYPT_SHA256_ALGORITHM
#define BCRYPT_SHA256_ALGORITHM L"SHA256"  // 定义 BCRYPT SHA256 算法
#endif

#ifndef BCRYPT_SHA384_ALGORITHM
#define BCRYPT_SHA384_ALGORITHM L"SHA384"  // 定义 BCRYPT SHA384 算法
#endif

#ifndef BCRYPT_SHA512_ALGORITHM
#define BCRYPT_SHA512_ALGORITHM L"SHA512"  // 定义 BCRYPT SHA512 算法
#endif

#ifndef BCRYPT_RSA_ALGORITHM
#define BCRYPT_RSA_ALGORITHM L"RSA"  // 定义 BCRYPT RSA 算法
#endif

#ifndef BCRYPT_DSA_ALGORITHM
#define BCRYPT_DSA_ALGORITHM L"DSA"  // 定义 BCRYPT DSA 算法
#endif

#ifndef BCRYPT_AES_ALGORITHM
#define BCRYPT_AES_ALGORITHM L"AES"  // 定义 BCRYPT AES 算法
#endif

#ifndef BCRYPT_RC4_ALGORITHM
#define BCRYPT_RC4_ALGORITHM L"RC4"  // 定义 BCRYPT RC4 算法
#endif

#ifndef BCRYPT_3DES_ALGORITHM
#define BCRYPT_3DES_ALGORITHM L"3DES"  // 定义 BCRYPT 3DES 算法
#endif

#ifndef BCRYPT_DH_ALGORITHM
#define BCRYPT_DH_ALGORITHM L"DH"  // 定义 BCRYPT DH 算法
#endif

/* BCRYPT_KDF_RAW_SECRET is available from Windows 8.1 and onwards */
#ifndef BCRYPT_KDF_RAW_SECRET
#define BCRYPT_KDF_RAW_SECRET L"TRUNCATE"  // 定义 BCRYPT 原始密钥派生算法
#endif

#ifndef BCRYPT_ALG_HANDLE_HMAC_FLAG
#define BCRYPT_ALG_HANDLE_HMAC_FLAG 0x00000008  // 定义 BCRYPT HMAC 标志
#endif
// 如果未定义 BCRYPT_DSA_PUBLIC_BLOB，则定义为 DSAPUBLICBLOB
#ifndef BCRYPT_DSA_PUBLIC_BLOB
#define BCRYPT_DSA_PUBLIC_BLOB L"DSAPUBLICBLOB"
#endif

// 如果未定义 BCRYPT_DSA_PUBLIC_MAGIC，则定义为 0x42505344
#ifndef BCRYPT_DSA_PUBLIC_MAGIC
#define BCRYPT_DSA_PUBLIC_MAGIC 0x42505344 /* DSPB */
#endif

// 如果未定义 BCRYPT_DSA_PRIVATE_BLOB，则定义为 DSAPRIVATEBLOB
#ifndef BCRYPT_DSA_PRIVATE_BLOB
#define BCRYPT_DSA_PRIVATE_BLOB L"DSAPRIVATEBLOB"
#endif

// 如果未定义 BCRYPT_DSA_PRIVATE_MAGIC，则定义为 0x56505344
#ifndef BCRYPT_DSA_PRIVATE_MAGIC
#define BCRYPT_DSA_PRIVATE_MAGIC 0x56505344 /* DSPV */
#endif

// 如果未定义 BCRYPT_RSAPUBLIC_BLOB，则定义为 RSAPUBLICBLOB
#ifndef BCRYPT_RSAPUBLIC_BLOB
#define BCRYPT_RSAPUBLIC_BLOB L"RSAPUBLICBLOB"
#endif

// 如果未定义 BCRYPT_RSAPUBLIC_MAGIC，则定义为 0x31415352
#ifndef BCRYPT_RSAPUBLIC_MAGIC
#define BCRYPT_RSAPUBLIC_MAGIC 0x31415352 /* RSA1 */
#endif

// 如果未定义 BCRYPT_RSAFULLPRIVATE_BLOB，则定义为 RSAFULLPRIVATEBLOB
#ifndef BCRYPT_RSAFULLPRIVATE_BLOB
#define BCRYPT_RSAFULLPRIVATE_BLOB L"RSAFULLPRIVATEBLOB"
#endif

// 如果未定义 BCRYPT_RSAFULLPRIVATE_MAGIC，则定义为 0x33415352
#ifndef BCRYPT_RSAFULLPRIVATE_MAGIC
#define BCRYPT_RSAFULLPRIVATE_MAGIC 0x33415352 /* RSA3 */
#endif

// 如果未定义 BCRYPT_KEY_DATA_BLOB，则定义为 KeyDataBlob
#ifndef BCRYPT_KEY_DATA_BLOB
#define BCRYPT_KEY_DATA_BLOB L"KeyDataBlob"
#endif

// 如果未定义 BCRYPT_MESSAGE_BLOCK_LENGTH，则定义为 MessageBlockLength
#ifndef BCRYPT_MESSAGE_BLOCK_LENGTH
#define BCRYPT_MESSAGE_BLOCK_LENGTH L"MessageBlockLength"
#endif

// 如果未定义 BCRYPT_NO_KEY_VALIDATION，则定义为 0x00000008
#ifndef BCRYPT_NO_KEY_VALIDATION
#define BCRYPT_NO_KEY_VALIDATION 0x00000008
#endif

// 如果未定义 BCRYPT_BLOCK_PADDING，则定义为 0x00000001
#ifndef BCRYPT_BLOCK_PADDING
#define BCRYPT_BLOCK_PADDING 0x00000001
#endif

// 如果未定义 BCRYPT_PAD_NONE，则定义为 0x00000001
#ifndef BCRYPT_PAD_NONE
#define BCRYPT_PAD_NONE 0x00000001
#endif

// 如果未定义 BCRYPT_PAD_PKCS1，则定义为 0x00000002
#ifndef BCRYPT_PAD_PKCS1
#define BCRYPT_PAD_PKCS1 0x00000002
#endif

// 如果未定义 BCRYPT_PAD_OAEP，则定义为 0x00000004
#ifndef BCRYPT_PAD_OAEP
#define BCRYPT_PAD_OAEP 0x00000004
#endif

// 如果未定义 BCRYPT_PAD_PSS，则定义为 0x00000008
#ifndef BCRYPT_PAD_PSS
#define BCRYPT_PAD_PSS 0x00000008
#endif

// 如果未定义 CRYPT_STRING_ANY，则定义为 0x00000007
#ifndef CRYPT_STRING_ANY
#define CRYPT_STRING_ANY 0x00000007
#endif

// 如果未定义 LEGACY_RSAPRIVATE_BLOB，则定义为 CAPIPRIVATEBLOB
#ifndef LEGACY_RSAPRIVATE_BLOB
#define LEGACY_RSAPRIVATE_BLOB L"CAPIPRIVATEBLOB"
#endif

// 如果未定义 PKCS_RSA_PRIVATE_KEY，则定义为 (LPCSTR)43
#ifndef PKCS_RSA_PRIVATE_KEY
#define PKCS_RSA_PRIVATE_KEY (LPCSTR)43
#endif

/*******************************************************************/
/*
 * Windows CNG backend: Generic functions
 */

// 定义 libssh2_wincng_ctx 结构体的实例 _libssh2_wincng
struct _libssh2_wincng_ctx _libssh2_wincng;

// 初始化 libssh2_wincng_ctx 结构体的实例 _libssh2_wincng
void
_libssh2_wincng_init(void)
{
    int ret;

    // 将 _libssh2_wincng 实例的内存清零
    memset(&_libssh2_wincng, 0, sizeof(_libssh2_wincng));
    # 使用 BCryptOpenAlgorithmProvider 函数打开随机数生成器算法提供程序，并将句柄存储在 _libssh2_wincng.hAlgRNG 中
    ret = BCryptOpenAlgorithmProvider(&_libssh2_wincng.hAlgRNG,
                                      BCRYPT_RNG_ALGORITHM, NULL, 0);
    # 如果返回值不是 BCRYPT_SUCCESS，则将 _libssh2_wincng.hAlgRNG 置为 NULL
    if(!BCRYPT_SUCCESS(ret)) {
        _libssh2_wincng.hAlgRNG = NULL;
    }

    # 使用 BCryptOpenAlgorithmProvider 函数打开 MD5 哈希算法提供程序，并将句柄存储在 _libssh2_wincng.hAlgHashMD5 中
    ret = BCryptOpenAlgorithmProvider(&_libssh2_wincng.hAlgHashMD5,
                                      BCRYPT_MD5_ALGORITHM, NULL, 0);
    # 如果返回值不是 BCRYPT_SUCCESS，则将 _libssh2_wincng.hAlgHashMD5 置为 NULL
    if(!BCRYPT_SUCCESS(ret)) {
        _libssh2_wincng.hAlgHashMD5 = NULL;
    }

    # 使用 BCryptOpenAlgorithmProvider 函数打开 SHA1 哈希算法提供程序，并将句柄存储在 _libssh2_wincng.hAlgHashSHA1 中
    ret = BCryptOpenAlgorithmProvider(&_libssh2_wincng.hAlgHashSHA1,
                                      BCRYPT_SHA1_ALGORITHM, NULL, 0);
    # 如果返回值不是 BCRYPT_SUCCESS，则将 _libssh2_wincng.hAlgHashSHA1 置为 NULL
    if(!BCRYPT_SUCCESS(ret)) {
        _libssh2_wincng.hAlgHashSHA1 = NULL;
    }

    # 使用 BCryptOpenAlgorithmProvider 函数打开 SHA256 哈希算法提供程序，并将句柄存储在 _libssh2_wincng.hAlgHashSHA256 中
    ret = BCryptOpenAlgorithmProvider(&_libssh2_wincng.hAlgHashSHA256,
                                      BCRYPT_SHA256_ALGORITHM, NULL, 0);
    # 如果返回值不是 BCRYPT_SUCCESS，则将 _libssh2_wincng.hAlgHashSHA256 置为 NULL
    if(!BCRYPT_SUCCESS(ret)) {
        _libssh2_wincng.hAlgHashSHA256 = NULL;
    }

    # 使用 BCryptOpenAlgorithmProvider 函数打开 SHA384 哈希算法提供程序，并将句柄存储在 _libssh2_wincng.hAlgHashSHA384 中
    ret = BCryptOpenAlgorithmProvider(&_libssh2_wincng.hAlgHashSHA384,
                                      BCRYPT_SHA384_ALGORITHM, NULL, 0);
    # 如果返回值不是 BCRYPT_SUCCESS，则将 _libssh2_wincng.hAlgHashSHA384 置为 NULL
    if(!BCRYPT_SUCCESS(ret)) {
        _libssh2_wincng.hAlgHashSHA384 = NULL;
    }

    # 使用 BCryptOpenAlgorithmProvider 函数打开 SHA512 哈希算法提供程序，并将句柄存储在 _libssh2_wincng.hAlgHashSHA512 中
    ret = BCryptOpenAlgorithmProvider(&_libssh2_wincng.hAlgHashSHA512,
                                      BCRYPT_SHA512_ALGORITHM, NULL, 0);
    # 如果返回值不是 BCRYPT_SUCCESS，则将 _libssh2_wincng.hAlgHashSHA512 置为 NULL
    if(!BCRYPT_SUCCESS(ret)) {
        _libssh2_wincng.hAlgHashSHA512 = NULL;
    }

    # 使用 BCryptOpenAlgorithmProvider 函数打开 MD5 HMAC 算法提供程序，并将句柄存储在 _libssh2_wincng.hAlgHmacMD5 中
    ret = BCryptOpenAlgorithmProvider(&_libssh2_wincng.hAlgHmacMD5,
                                      BCRYPT_MD5_ALGORITHM, NULL,
                                      BCRYPT_ALG_HANDLE_HMAC_FLAG);
    # 如果返回值不是 BCRYPT_SUCCESS，则将 _libssh2_wincng.hAlgHmacMD5 置为 NULL
    if(!BCRYPT_SUCCESS(ret)) {
        _libssh2_wincng.hAlgHmacMD5 = NULL;
    }

    # 使用 BCryptOpenAlgorithmProvider 函数打开 SHA1 HMAC 算法提供程序，并将句柄存储在 _libssh2_wincng.hAlgHmacSHA1 中
    ret = BCryptOpenAlgorithmProvider(&_libssh2_wincng.hAlgHmacSHA1,
                                      BCRYPT_SHA1_ALGORITHM, NULL,
                                      BCRYPT_ALG_HANDLE_HMAC_FLAG);
    # 如果返回值不是 BCRYPT_SUCCESS，则将 _libssh2_wincng.hAlgHmacSHA1 置为 NULL
    if(!BCRYPT_SUCCESS(ret)) {
        _libssh2_wincng.hAlgHmacSHA1 = NULL;
    }
    # 使用 BCryptOpenAlgorithmProvider 函数打开 HMAC-SHA256 算法提供程序，并将句柄保存在 _libssh2_wincng.hAlgHmacSHA256 中
    ret = BCryptOpenAlgorithmProvider(&_libssh2_wincng.hAlgHmacSHA256,
                                      BCRYPT_SHA256_ALGORITHM, NULL,
                                      BCRYPT_ALG_HANDLE_HMAC_FLAG);
    # 如果返回值不是 BCRYPT_SUCCESS，则将 _libssh2_wincng.hAlgHmacSHA256 置为 NULL
    if(!BCRYPT_SUCCESS(ret)) {
        _libssh2_wincng.hAlgHmacSHA256 = NULL;
    }
    # 使用 BCryptOpenAlgorithmProvider 函数打开 HMAC-SHA384 算法提供程序，并将句柄保存在 _libssh2_wincng.hAlgHmacSHA384 中
    ret = BCryptOpenAlgorithmProvider(&_libssh2_wincng.hAlgHmacSHA384,
                                      BCRYPT_SHA384_ALGORITHM, NULL,
                                      BCRYPT_ALG_HANDLE_HMAC_FLAG);
    # 如果返回值不是 BCRYPT_SUCCESS，则将 _libssh2_wincng.hAlgHmacSHA384 置为 NULL
    if(!BCRYPT_SUCCESS(ret)) {
        _libssh2_wincng.hAlgHmacSHA384 = NULL;
    }
    # 使用 BCryptOpenAlgorithmProvider 函数打开 HMAC-SHA512 算法提供程序，并将句柄保存在 _libssh2_wincng.hAlgHmacSHA512 中
    ret = BCryptOpenAlgorithmProvider(&_libssh2_wincng.hAlgHmacSHA512,
                                      BCRYPT_SHA512_ALGORITHM, NULL,
                                      BCRYPT_ALG_HANDLE_HMAC_FLAG);
    # 如果返回值不是 BCRYPT_SUCCESS，则将 _libssh2_wincng.hAlgHmacSHA512 置为 NULL
    if(!BCRYPT_SUCCESS(ret)) {
        _libssh2_wincng.hAlgHmacSHA512 = NULL;
    }
    # 使用 BCryptOpenAlgorithmProvider 函数打开 RSA 算法提供程序，并将句柄保存在 _libssh2_wincng.hAlgRSA 中
    ret = BCryptOpenAlgorithmProvider(&_libssh2_wincng.hAlgRSA,
                                      BCRYPT_RSA_ALGORITHM, NULL, 0);
    # 如果返回值不是 BCRYPT_SUCCESS，则将 _libssh2_wincng.hAlgRSA 置为 NULL
    if(!BCRYPT_SUCCESS(ret)) {
        _libssh2_wincng.hAlgRSA = NULL;
    }
    # 使用 BCryptOpenAlgorithmProvider 函数打开 DSA 算法提供程序，并将句柄保存在 _libssh2_wincng.hAlgDSA 中
    ret = BCryptOpenAlgorithmProvider(&_libssh2_wincng.hAlgDSA,
                                      BCRYPT_DSA_ALGORITHM, NULL, 0);
    # 如果返回值不是 BCRYPT_SUCCESS，则将 _libssh2_wincng.hAlgDSA 置为 NULL
    if(!BCRYPT_SUCCESS(ret)) {
        _libssh2_wincng.hAlgDSA = NULL;
    }
    # 使用 BCryptOpenAlgorithmProvider 函数打开 AES-CBC 算法提供程序，并将句柄保存在 _libssh2_wincng.hAlgAES_CBC 中
    ret = BCryptOpenAlgorithmProvider(&_libssh2_wincng.hAlgAES_CBC,
                                      BCRYPT_AES_ALGORITHM, NULL, 0);
    # 如果返回值是 BCRYPT_SUCCESS，则设置 AES-CBC 算法的链模式为 CBC
    if(BCRYPT_SUCCESS(ret)) {
        ret = BCryptSetProperty(_libssh2_wincng.hAlgAES_CBC,
                                BCRYPT_CHAINING_MODE,
                                (PBYTE)BCRYPT_CHAIN_MODE_CBC,
                                sizeof(BCRYPT_CHAIN_MODE_CBC), 0);
        # 如果设置链模式失败，则关闭 AES-CBC 算法提供程序，并将 _libssh2_wincng.hAlgAES_CBC 置为 NULL
        if(!BCRYPT_SUCCESS(ret)) {
            ret = BCryptCloseAlgorithmProvider(_libssh2_wincng.hAlgAES_CBC, 0);
            if(BCRYPT_SUCCESS(ret)) {
                _libssh2_wincng.hAlgAES_CBC = NULL;
            }
        }
    }
    # 打开 AES_ECB 算法提供程序，并将其句柄存储在 _libssh2_wincng.hAlgAES_ECB 中
    ret = BCryptOpenAlgorithmProvider(&_libssh2_wincng.hAlgAES_ECB,
                                      BCRYPT_AES_ALGORITHM, NULL, 0);
    # 如果成功打开算法提供程序
    if(BCRYPT_SUCCESS(ret)) {
        # 设置 AES_ECB 算法的属性为 ECB 模式
        ret = BCryptSetProperty(_libssh2_wincng.hAlgAES_ECB,
                                BCRYPT_CHAINING_MODE,
                                (PBYTE)BCRYPT_CHAIN_MODE_ECB,
                                sizeof(BCRYPT_CHAIN_MODE_ECB), 0);
        # 如果设置属性不成功
        if(!BCRYPT_SUCCESS(ret)) {
            # 关闭 AES_ECB 算法提供程序
            ret = BCryptCloseAlgorithmProvider(_libssh2_wincng.hAlgAES_ECB, 0);
            # 如果成功关闭算法提供程序
            if(BCRYPT_SUCCESS(ret)) {
                # 将 _libssh2_wincng.hAlgAES_ECB 置为 NULL
                _libssh2_wincng.hAlgAES_ECB = NULL;
            }
        }
    }

    # 打开 RC4_NA 算法提供程序，并将其句柄存储在 _libssh2_wincng.hAlgRC4_NA 中
    ret = BCryptOpenAlgorithmProvider(&_libssh2_wincng.hAlgRC4_NA,
                                      BCRYPT_RC4_ALGORITHM, NULL, 0);
    # 如果成功打开算法提供程序
    if(BCRYPT_SUCCESS(ret)) {
        # 设置 RC4_NA 算法的属性为 NA 模式
        ret = BCryptSetProperty(_libssh2_wincng.hAlgRC4_NA,
                                BCRYPT_CHAINING_MODE,
                                (PBYTE)BCRYPT_CHAIN_MODE_NA,
                                sizeof(BCRYPT_CHAIN_MODE_NA), 0);
        # 如果设置属性不成功
        if(!BCRYPT_SUCCESS(ret)) {
            # 关闭 RC4_NA 算法提供程序
            ret = BCryptCloseAlgorithmProvider(_libssh2_wincng.hAlgRC4_NA, 0);
            # 如果成功关闭算法提供程序
            if(BCRYPT_SUCCESS(ret)) {
                # 将 _libssh2_wincng.hAlgRC4_NA 置为 NULL
                _libssh2_wincng.hAlgRC4_NA = NULL;
            }
        }
    }

    # 打开 3DES_CBC 算法提供程序，并将其句柄存储在 _libssh2_wincng.hAlg3DES_CBC 中
    ret = BCryptOpenAlgorithmProvider(&_libssh2_wincng.hAlg3DES_CBC,
                                      BCRYPT_3DES_ALGORITHM, NULL, 0);
    # 如果成功打开算法提供程序
    if(BCRYPT_SUCCESS(ret)) {
        # 设置 3DES_CBC 算法的属性为 CBC 模式
        ret = BCryptSetProperty(_libssh2_wincng.hAlg3DES_CBC,
                                BCRYPT_CHAINING_MODE,
                                (PBYTE)BCRYPT_CHAIN_MODE_CBC,
                                sizeof(BCRYPT_CHAIN_MODE_CBC), 0);
        # 如果设置属性不成功
        if(!BCRYPT_SUCCESS(ret)) {
            # 关闭 3DES_CBC 算法提供程序
            ret = BCryptCloseAlgorithmProvider(_libssh2_wincng.hAlg3DES_CBC,
                                               0);
            # 如果成功关闭算法提供程序
            if(BCRYPT_SUCCESS(ret)) {
                # 将 _libssh2_wincng.hAlg3DES_CBC 置为 NULL
                _libssh2_wincng.hAlg3DES_CBC = NULL;
            }
        }
    }
    # 调用 BCryptOpenAlgorithmProvider 函数打开 Diffie-Hellman 算法提供程序，并将结果存储在 _libssh2_wincng.hAlgDH 中
    ret = BCryptOpenAlgorithmProvider(&_libssh2_wincng.hAlgDH, BCRYPT_DH_ALGORITHM, NULL, 0);
    # 如果 ret 不是 BCRYPT_SUCCESS，则将 _libssh2_wincng.hAlgDH 设置为 NULL
    if(!BCRYPT_SUCCESS(ret)) {
        _libssh2_wincng.hAlgDH = NULL;
    }
void
_libssh2_wincng_free(void)
{
    // 如果随机数生成器句柄存在，则关闭对应的算法提供者
    if(_libssh2_wincng.hAlgRNG)
        (void)BCryptCloseAlgorithmProvider(_libssh2_wincng.hAlgRNG, 0);
    // 如果 MD5 哈希句柄存在，则关闭对应的算法提供者
    if(_libssh2_wincng.hAlgHashMD5)
        (void)BCryptCloseAlgorithmProvider(_libssh2_wincng.hAlgHashMD5, 0);
    // 如果 SHA1 哈希句柄存在，则关闭对应的算法提供者
    if(_libssh2_wincng.hAlgHashSHA1)
        (void)BCryptCloseAlgorithmProvider(_libssh2_wincng.hAlgHashSHA1, 0);
    // 如果 SHA256 哈希句柄存在，则关闭对应的算法提供者
    if(_libssh2_wincng.hAlgHashSHA256)
        (void)BCryptCloseAlgorithmProvider(_libssh2_wincng.hAlgHashSHA256, 0);
    // 如果 SHA384 哈希句柄存在，则关闭对应的算法提供者
    if(_libssh2_wincng.hAlgHashSHA384)
        (void)BCryptCloseAlgorithmProvider(_libssh2_wincng.hAlgHashSHA384, 0);
    // 如果 SHA512 哈希句柄存在，则关闭对应的算法提供者
    if(_libssh2_wincng.hAlgHashSHA512)
        (void)BCryptCloseAlgorithmProvider(_libssh2_wincng.hAlgHashSHA512, 0);
    // 如果 HMAC-MD5 句柄存在，则关闭对应的算法提供者
    if(_libssh2_wincng.hAlgHmacMD5)
        (void)BCryptCloseAlgorithmProvider(_libssh2_wincng.hAlgHmacMD5, 0);
    // 如果 HMAC-SHA1 句柄存在，则关闭对应的算法提供者
    if(_libssh2_wincng.hAlgHmacSHA1)
        (void)BCryptCloseAlgorithmProvider(_libssh2_wincng.hAlgHmacSHA1, 0);
    // 如果 HMAC-SHA256 句柄存在，则关闭对应的算法提供者
    if(_libssh2_wincng.hAlgHmacSHA256)
        (void)BCryptCloseAlgorithmProvider(_libssh2_wincng.hAlgHmacSHA256, 0);
    // 如果 HMAC-SHA384 句柄存在，则关闭对应的算法提供者
    if(_libssh2_wincng.hAlgHmacSHA384)
        (void)BCryptCloseAlgorithmProvider(_libssh2_wincng.hAlgHmacSHA384, 0);
    // 如果 HMAC-SHA512 句柄存在，则关闭对应的算法提供者
    if(_libssh2_wincng.hAlgHmacSHA512)
        (void)BCryptCloseAlgorithmProvider(_libssh2_wincng.hAlgHmacSHA512, 0);
    // 如果 RSA 句柄存在，则关闭对应的算法提供者
    if(_libssh2_wincng.hAlgRSA)
        (void)BCryptCloseAlgorithmProvider(_libssh2_wincng.hAlgRSA, 0);
    // 如果 DSA 句柄存在，则关闭对应的算法提供者
    if(_libssh2_wincng.hAlgDSA)
        (void)BCryptCloseAlgorithmProvider(_libssh2_wincng.hAlgDSA, 0);
    // 如果 AES-CBC 句柄存在，则关闭对应的算法提供者
    if(_libssh2_wincng.hAlgAES_CBC)
        (void)BCryptCloseAlgorithmProvider(_libssh2_wincng.hAlgAES_CBC, 0);
    // 如果 RC4-NA 句柄存在，则关闭对应的算法提供者
    if(_libssh2_wincng.hAlgRC4_NA)
        (void)BCryptCloseAlgorithmProvider(_libssh2_wincng.hAlgRC4_NA, 0);
    // 如果 3DES-CBC 句柄存在，则关闭对应的算法提供者
    if(_libssh2_wincng.hAlg3DES_CBC)
        (void)BCryptCloseAlgorithmProvider(_libssh2_wincng.hAlg3DES_CBC, 0);
    // 如果 DH 句柄存在，则关闭对应的算法提供者
    if(_libssh2_wincng.hAlgDH)
        (void)BCryptCloseAlgorithmProvider(_libssh2_wincng.hAlgDH, 0);
}
    # 使用memset函数将_libssh2_wincng的内存空间初始化为0，大小为sizeof(_libssh2_wincng)
    memset(&_libssh2_wincng, 0, sizeof(_libssh2_wincng));
}

int
_libssh2_wincng_random(void *buf, int len)
{
    int ret;

    // 使用 Windows CNG API 生成随机数填充到指定的缓冲区
    ret = BCryptGenRandom(_libssh2_wincng.hAlgRNG, buf, len, 0);

    // 如果生成随机数成功，则返回 0，否则返回 -1
    return BCRYPT_SUCCESS(ret) ? 0 : -1;
}

static void
_libssh2_wincng_safe_free(void *buf, int len)
{
#ifndef LIBSSH2_CLEAR_MEMORY
    (void)len;
#endif

    // 如果未定义 LIBSSH2_CLEAR_MEMORY，则直接返回
    if(!buf)
        return;

#ifdef LIBSSH2_CLEAR_MEMORY
    // 如果定义了 LIBSSH2_CLEAR_MEMORY，并且长度大于 0，则使用 SecureZeroMemory 清空缓冲区
    if(len > 0)
        SecureZeroMemory(buf, len);
#endif

    // 释放缓冲区
    free(buf);
}

/* Copy a big endian set of bits from src to dest.
 * if the size of src is smaller than dest then pad the "left" (MSB)
 * end with zeroes and copy the bits into the "right" (LSB) end. */
static void
memcpy_with_be_padding(unsigned char *dest, unsigned long dest_len,
                       unsigned char *src, unsigned long src_len)
{
    // 如果目标长度大于源长度，则使用 0 填充目标缓冲区的左侧
    if(dest_len > src_len) {
        memset(dest, 0, dest_len - src_len);
    }
    // 将源缓冲区的内容复制到目标缓冲区
    memcpy((dest + dest_len) - src_len, src, src_len);
}

static int
round_down(int number, int multiple)
{
    // 将给定的数字向下舍入到最接近的倍数
    return (number / multiple) * multiple;
}

/*******************************************************************/
/*
 * Windows CNG backend: Hash functions
 */

int
_libssh2_wincng_hash_init(_libssh2_wincng_hash_ctx *ctx,
                          BCRYPT_ALG_HANDLE hAlg, unsigned long hashlen,
                          unsigned char *key, unsigned long keylen)
{
    BCRYPT_HASH_HANDLE hHash;
    unsigned char *pbHashObject;
    unsigned long dwHashObject, dwHash, cbData;
    int ret;

    // 获取哈希算法的长度
    ret = BCryptGetProperty(hAlg, BCRYPT_HASH_LENGTH,
                            (unsigned char *)&dwHash,
                            sizeof(dwHash),
                            &cbData, 0);
    // 如果获取失败或者长度不匹配，则返回 -1
    if((!BCRYPT_SUCCESS(ret)) || dwHash != hashlen) {
        return -1;
    }

    // 获取哈希对象的长度
    ret = BCryptGetProperty(hAlg, BCRYPT_OBJECT_LENGTH,
                            (unsigned char *)&dwHashObject,
                            sizeof(dwHashObject),
                            &cbData, 0);
    // 如果获取失败，则返回 -1
    if(!BCRYPT_SUCCESS(ret)) {
        return -1;
    }
    # 分配内存以存储哈希对象
    pbHashObject = malloc(dwHashObject);
    # 如果内存分配失败，则返回-1
    if(!pbHashObject) {
        return -1;
    }
    
    # 使用BCryptCreateHash函数创建哈希对象
    ret = BCryptCreateHash(hAlg, &hHash,
                           pbHashObject, dwHashObject,
                           key, keylen, 0);
    # 如果创建哈希对象失败，则释放内存并返回-1
    if(!BCRYPT_SUCCESS(ret)) {
        _libssh2_wincng_safe_free(pbHashObject, dwHashObject);
        return -1;
    }
    
    # 将哈希对象和相关信息存储在上下文中
    ctx->hHash = hHash;
    ctx->pbHashObject = pbHashObject;
    ctx->dwHashObject = dwHashObject;
    ctx->cbHash = dwHash;
    
    # 返回0表示成功
    return 0;
}

int
_libssh2_wincng_hash_update(_libssh2_wincng_hash_ctx *ctx,
                            const unsigned char *data, unsigned long datalen)
{
    int ret;

    // 使用 BCryptHashData 函数更新哈希上下文中的哈希值
    ret = BCryptHashData(ctx->hHash, (unsigned char *)data, datalen, 0);

    // 如果 BCryptHashData 函数执行成功，则返回 0，否则返回 -1
    return BCRYPT_SUCCESS(ret) ? 0 : -1;
}

int
_libssh2_wincng_hash_final(_libssh2_wincng_hash_ctx *ctx,
                           unsigned char *hash)
{
    int ret;

    // 使用 BCryptFinishHash 函数完成哈希计算并将结果存储在 hash 中
    ret = BCryptFinishHash(ctx->hHash, hash, ctx->cbHash, 0);

    // 销毁哈希对象
    BCryptDestroyHash(ctx->hHash);
    ctx->hHash = NULL;

    // 释放哈希对象的内存
    _libssh2_wincng_safe_free(ctx->pbHashObject, ctx->dwHashObject);
    ctx->pbHashObject = NULL;
    ctx->dwHashObject = 0;

    // 如果 BCryptFinishHash 函数执行成功，则返回 0，否则返回 -1
    return BCRYPT_SUCCESS(ret) ? 0 : -1;
}

int
_libssh2_wincng_hash(unsigned char *data, unsigned long datalen,
                     BCRYPT_ALG_HANDLE hAlg,
                     unsigned char *hash, unsigned long hashlen)
{
    _libssh2_wincng_hash_ctx ctx;
    int ret;

    // 初始化哈希上下文
    ret = _libssh2_wincng_hash_init(&ctx, hAlg, hashlen, NULL, 0);
    if(!ret) {
        // 更新哈希值
        ret = _libssh2_wincng_hash_update(&ctx, data, datalen);
        // 完成哈希计算
        ret |= _libssh2_wincng_hash_final(&ctx, hash);
    }

    return ret;
}


/*******************************************************************/
/*
 * Windows CNG backend: HMAC functions
 */

int
_libssh2_wincng_hmac_final(_libssh2_wincng_hash_ctx *ctx,
                           unsigned char *hash)
{
    int ret;

    // 使用 BCryptFinishHash 函数完成哈希计算并将结果存储在 hash 中
    ret = BCryptFinishHash(ctx->hHash, hash, ctx->cbHash, 0);

    // 如果 BCryptFinishHash 函数执行成功，则返回 0，否则返回 -1
    return BCRYPT_SUCCESS(ret) ? 0 : -1;
}

void
_libssh2_wincng_hmac_cleanup(_libssh2_wincng_hash_ctx *ctx)
{
    // 销毁哈希对象
    BCryptDestroyHash(ctx->hHash);
    ctx->hHash = NULL;

    // 释放哈希对象的内存
    _libssh2_wincng_safe_free(ctx->pbHashObject, ctx->dwHashObject);
    ctx->pbHashObject = NULL;
    ctx->dwHashObject = 0;
}


/*******************************************************************/
/*
 * Windows CNG backend: Key functions
 */

int
# 验证使用 SHA1 算法的密钥签名
_libssh2_wincng_key_sha1_verify(_libssh2_wincng_key_ctx *ctx,  # 使用的密钥上下文
                                const unsigned char *sig,  # 签名数据
                                unsigned long sig_len,  # 签名数据长度
                                const unsigned char *m,  # 待验证的数据
                                unsigned long m_len,  # 待验证的数据长度
                                unsigned long flags)  # 标志位
{
    BCRYPT_PKCS1_PADDING_INFO paddingInfoPKCS1;  # PKCS1 填充信息
    void *pPaddingInfo;  # 填充信息指针
    unsigned char *data, *hash;  # 数据和哈希值
    unsigned long datalen, hashlen;  # 数据长度和哈希值长度
    int ret;  # 返回值

    datalen = m_len;  # 设置数据长度
    data = malloc(datalen);  # 分配数据内存空间
    if(!data) {  # 如果分配失败
        return -1;  # 返回错误
    }

    hashlen = SHA_DIGEST_LENGTH;  # 设置哈希值长度
    hash = malloc(hashlen);  # 分配哈希值内存空间
    if(!hash) {  # 如果分配失败
        free(data);  # 释放数据内存空间
        return -1;  # 返回错误
    }

    memcpy(data, m, datalen);  # 将待验证的数据复制到数据中

    ret = _libssh2_wincng_hash(data, datalen,  # 使用哈希函数计算哈希值
                               _libssh2_wincng.hAlgHashSHA1,
                               hash, hashlen);

    _libssh2_wincng_safe_free(data, datalen);  # 安全释放数据内存空间

    if(ret) {  # 如果计算哈希值失败
        _libssh2_wincng_safe_free(hash, hashlen);  # 安全释放哈希值内存空间
        return -1;  # 返回错误
    }

    datalen = sig_len;  # 设置数据长度为签名数据长度
    data = malloc(datalen);  # 分配数据内存空间
    if(!data) {  # 如果分配失败
        _libssh2_wincng_safe_free(hash, hashlen);  # 安全释放哈希值内存空间
        return -1;  # 返回错误
    }

    if(flags & BCRYPT_PAD_PKCS1) {  # 如果标志位包含 BCRYPT_PAD_PKCS1
        paddingInfoPKCS1.pszAlgId = BCRYPT_SHA1_ALGORITHM;  # 设置填充信息的算法 ID
        pPaddingInfo = &paddingInfoPKCS1;  # 填充信息指针指向 PKCS1 填充信息
    }
    else
        pPaddingInfo = NULL;  # 否则填充信息指针为空

    memcpy(data, sig, datalen);  # 将签名数据复制到数据中

    ret = BCryptVerifySignature(ctx->hKey, pPaddingInfo,  # 使用密钥验证签名
                                hash, hashlen, data, datalen, flags);

    _libssh2_wincng_safe_free(hash, hashlen);  # 安全释放哈希值内存空间
    _libssh2_wincng_safe_free(data, datalen);  # 安全释放数据内存空间

    return BCRYPT_SUCCESS(ret) ? 0 : -1;  # 返回验证结果
}

#ifdef HAVE_LIBCRYPT32
static int
# 加载 PEM 格式的私钥文件内容到内存中
_libssh2_wincng_load_pem(LIBSSH2_SESSION *session,
                         const char *filename,
                         const char *passphrase,
                         const char *headerbegin,
                         const char *headerend,
                         unsigned char **data,
                         unsigned int *datalen)
{
    # 打开指定文件，以只读文本模式
    FILE *fp;
    int ret;

    fp = fopen(filename, FOPEN_READTEXT);
    # 如果文件打开失败，返回错误码 -1
    if(!fp) {
        return -1;
    }

    # 解析 PEM 格式的文件内容，获取数据和数据长度
    ret = _libssh2_pem_parse(session, headerbegin, headerend,
                             passphrase,
                             fp, data, datalen);

    # 关闭文件
    fclose(fp);

    # 返回解析结果
    return ret;
}

# 加载私钥文件内容到内存中
static int
_libssh2_wincng_load_private(LIBSSH2_SESSION *session,
                             const char *filename,
                             const char *passphrase,
                             unsigned char **ppbEncoded,
                             unsigned long *pcbEncoded,
                             int tryLoadRSA, int tryLoadDSA)
{
    unsigned char *data = NULL;
    unsigned int datalen = 0;
    int ret = -1;

    # 尝试加载 RSA 类型的私钥文件
    if(ret && tryLoadRSA) {
        ret = _libssh2_wincng_load_pem(session, filename, passphrase,
                                       PEM_RSA_HEADER, PEM_RSA_FOOTER,
                                       &data, &datalen);
    }

    # 尝试加载 DSA 类型的私钥文件
    if(ret && tryLoadDSA) {
        ret = _libssh2_wincng_load_pem(session, filename, passphrase,
                                       PEM_DSA_HEADER, PEM_DSA_FOOTER,
                                       &data, &datalen);
    }

    # 如果成功加载私钥文件内容，将数据和数据长度保存到指定的指针中
    if(!ret) {
        *ppbEncoded = data;
        *pcbEncoded = datalen;
    }

    # 返回加载结果
    return ret;
}

static int
# 加载私钥数据到内存中
_libssh2_wincng_load_private_memory(LIBSSH2_SESSION *session,
                                    const char *privatekeydata,
                                    size_t privatekeydata_len,
                                    const char *passphrase,
                                    unsigned char **ppbEncoded,
                                    unsigned long *pcbEncoded,
                                    int tryLoadRSA, int tryLoadDSA)
{
    unsigned char *data = NULL;  # 初始化一个空的数据指针
    unsigned int datalen = 0;  # 初始化数据长度为0
    int ret = -1;  # 初始化返回值为-1

    (void)passphrase;  # 忽略传入的passphrase参数

    if(ret && tryLoadRSA) {  # 如果ret不为0且tryLoadRSA为真
        ret = _libssh2_pem_parse_memory(session,  # 调用_libssh2_pem_parse_memory函数
                                        PEM_RSA_HEADER, PEM_RSA_FOOTER,
                                        privatekeydata, privatekeydata_len,
                                        &data, &datalen);  # 将解析后的数据和长度存储到data和datalen中
    }

    if(ret && tryLoadDSA) {  # 如果ret不为0且tryLoadDSA为真
        ret = _libssh2_pem_parse_memory(session,  # 调用_libssh2_pem_parse_memory函数
                                        PEM_DSA_HEADER, PEM_DSA_FOOTER,
                                        privatekeydata, privatekeydata_len,
                                        &data, &datalen);  # 将解析后的数据和长度存储到data和datalen中
    }

    if(!ret) {  # 如果ret为0
        *ppbEncoded = data;  # 将data赋值给ppbEncoded
        *pcbEncoded = datalen;  # 将datalen赋值给pcbEncoded
    }

    return ret;  # 返回ret的值
}

# 解码ASN.1编码的数据
static int
_libssh2_wincng_asn_decode(unsigned char *pbEncoded,
                           unsigned long cbEncoded,
                           LPCSTR lpszStructType,
                           unsigned char **ppbDecoded,
                           unsigned long *pcbDecoded)
{
    unsigned char *pbDecoded = NULL;  # 初始化一个空的解码数据指针
    unsigned long cbDecoded = 0;  # 初始化解码数据长度为0
    int ret;  # 初始化返回值

    ret = CryptDecodeObjectEx(X509_ASN_ENCODING | PKCS_7_ASN_ENCODING,  # 调用CryptDecodeObjectEx函数解码数据
                              lpszStructType,
                              pbEncoded, cbEncoded, 0, NULL,
                              NULL, &cbDecoded);
    if(!ret) {  # 如果ret为0
        return -1;  # 返回-1
    }

    pbDecoded = malloc(cbDecoded);  # 分配内存给解码数据
    if(!pbDecoded) {  # 如果分配内存失败
        return -1;  # 返回-1
    }
    # 使用 CryptDecodeObjectEx 函数对编码的数据进行解码
    ret = CryptDecodeObjectEx(X509_ASN_ENCODING | PKCS_7_ASN_ENCODING,
                              lpszStructType,
                              pbEncoded, cbEncoded, 0, NULL,
                              pbDecoded, &cbDecoded);
    # 如果解码失败
    if(!ret) {
        # 释放已分配的内存
        _libssh2_wincng_safe_free(pbDecoded, cbDecoded);
        # 返回错误代码
        return -1;
    }

    # 将解码后的数据指针赋值给传入的指针
    *ppbDecoded = pbDecoded;
    # 将解码后的数据长度赋值给传入的指针
    *pcbDecoded = cbDecoded;

    # 返回成功代码
    return 0;
# 将大整数从小端序转换为大端序
static int
_libssh2_wincng_bn_ltob(unsigned char *pbInput,
                        unsigned long cbInput,
                        unsigned char **ppbOutput,
                        unsigned long *pcbOutput)
{
    unsigned char *pbOutput;  # 定义输出的字节数组
    unsigned long cbOutput, index, offset, length;  # 定义输出字节数组长度，索引，偏移量和长度

    if(cbInput < 1) {  # 如果输入的字节数小于1
        return 0;  # 返回0
    }

    offset = 0;  # 偏移量初始化为0
    length = cbInput - 1;  # 长度初始化为输入字节数减1
    cbOutput = cbInput;  # 输出字节数初始化为输入字节数
    if(pbInput[length] & (1 << 7)) {  # 如果输入的最后一个字节的最高位为1
        offset++;  # 偏移量加1
        cbOutput += offset;  # 输出字节数增加偏移量
    }

    pbOutput = (unsigned char *)malloc(cbOutput);  # 分配输出字节数组的内存空间
    if(!pbOutput) {  # 如果分配内存失败
        return -1;  # 返回-1
    }

    pbOutput[0] = 0;  # 输出字节数组的第一个字节初始化为0
    for(index = 0; ((index + offset) < cbOutput)
                    && (index < cbInput); index++) {  # 循环遍历输入和输出字节数组
        pbOutput[index + offset] = pbInput[length - index];  # 将输入字节数组的内容按照大端序存入输出字节数组
    }


    *ppbOutput = pbOutput;  # 将输出字节数组的地址赋给ppbOutput指针
    *pcbOutput = cbOutput;  # 将输出字节数组的长度赋给pcbOutput指针

    return 0;  # 返回0
}

# ASN.1解码大整数
static int
_libssh2_wincng_asn_decode_bn(unsigned char *pbEncoded,
                              unsigned long cbEncoded,
                              unsigned char **ppbDecoded,
                              unsigned long *pcbDecoded)
{
    unsigned char *pbDecoded = NULL, *pbInteger;  # 定义解码后的字节数组和整数字节数组
    unsigned long cbDecoded = 0, cbInteger;  # 定义解码后的字节数组长度和整数字节数组长度
    int ret;  # 定义返回值

    ret = _libssh2_wincng_asn_decode(pbEncoded, cbEncoded,  # 调用ASN.1解码函数
                                     X509_MULTI_BYTE_UINT,
                                     &pbInteger, &cbInteger);
    if(!ret) {  # 如果解码成功
        ret = _libssh2_wincng_bn_ltob(((PCRYPT_DATA_BLOB)pbInteger)->pbData,  # 调用大整数小端序转大端序函数
                                      ((PCRYPT_DATA_BLOB)pbInteger)->cbData,
                                      &pbDecoded, &cbDecoded);
        if(!ret) {  # 如果转换成功
            *ppbDecoded = pbDecoded;  # 将解码后的字节数组地址赋给ppbDecoded指针
            *pcbDecoded = cbDecoded;  # 将解码后的字节数组长度赋给pcbDecoded指针
        }
        _libssh2_wincng_safe_free(pbInteger, cbInteger);  # 释放整数字节数组的内存空间
    }

    return ret;  # 返回ret值
}

static int
# 解码 ASN.1 编码的数据
# 参数：
#   pbEncoded: 待解码的数据
#   cbEncoded: 待解码数据的长度
#   prpbDecoded: 解码后的数据
#   prcbDecoded: 解码后数据的长度
#   pcbCount: 解码后数据的数量
_libssh2_wincng_asn_decode_bns(unsigned char *pbEncoded,
                               unsigned long cbEncoded,
                               unsigned char ***prpbDecoded,
                               unsigned long **prcbDecoded,
                               unsigned long *pcbCount)
{
    # 用于存储解码后的数据结构
    PCRYPT_DER_BLOB pBlob;
    unsigned char *pbDecoded, **rpbDecoded;
    unsigned long cbDecoded, *rcbDecoded, index, length;
    int ret;

    # 调用 _libssh2_wincng_asn_decode 函数进行解码
    ret = _libssh2_wincng_asn_decode(pbEncoded, cbEncoded,
                                     X509_SEQUENCE_OF_ANY,
                                     &pbDecoded, &cbDecoded);
    # 如果返回值为假
    if(!ret) {
        # 获取解码后数据的长度
        length = ((PCRYPT_DATA_BLOB)pbDecoded)->cbData;

        # 分配解码后数据的字节数组
        rpbDecoded = malloc(sizeof(PBYTE) * length);
        # 如果分配成功
        if(rpbDecoded) {
            # 分配解码后数据长度的数组
            rcbDecoded = malloc(sizeof(DWORD) * length);
            # 如果分配成功
            if(rcbDecoded) {
                # 遍历解码后数据的每个元素
                for(index = 0; index < length; index++) {
                    # 获取解码后数据的指针
                    pBlob = &((PCRYPT_DER_BLOB)
                              ((PCRYPT_DATA_BLOB)pbDecoded)->pbData)[index];
                    # 调用解码函数
                    ret = _libssh2_wincng_asn_decode_bn(pBlob->pbData,
                                                        pBlob->cbData,
                                                        &rpbDecoded[index],
                                                        &rcbDecoded[index]);
                    # 如果解码失败则跳出循环
                    if(ret)
                        break;
                }

                # 如果解码成功
                if(!ret) {
                    # 将解码后数据的指针、长度和计数返回
                    *prpbDecoded = rpbDecoded;
                    *prcbDecoded = rcbDecoded;
                    *pcbCount = length;
                }
                # 如果解码失败
                else {
                    # 释放已分配的内存
                    for(length = 0; length < index; length++) {
                        _libssh2_wincng_safe_free(rpbDecoded[length],
                                                  rcbDecoded[length]);
                        rpbDecoded[length] = NULL;
                        rcbDecoded[length] = 0;
                    }
                    free(rpbDecoded);
                    free(rcbDecoded);
                }
            }
            # 如果分配失败
            else {
                free(rpbDecoded);
                ret = -1;
            }
        }
        # 如果分配失败
        else {
            ret = -1;
        }

        # 释放解码前数据的内存
        _libssh2_wincng_safe_free(pbDecoded, cbDecoded);
    }

    # 返回结果
    return ret;
    // 结束条件判断，如果没有大数数据，则返回0
    if(!bignum)
        return 0;

    // 减去一个字节，用于存储大数的长度
    length--;

    // 初始化偏移量为0
    offset = 0;
    // 循环直到找到第一个非零字节的偏移量
    while(!(*(bignum + offset)) && (offset < length))
        offset++;

    // 增加一个字节，用于存储大数的长度
    length++;

    // 返回大数的长度减去偏移量
    return length - offset;
}

/*******************************************************************/
/*
 * Windows CNG backend: RSA functions
 */

// 创建新的 RSA 上下文
int
_libssh2_wincng_rsa_new(libssh2_rsa_ctx **rsa,
                        const unsigned char *edata,
                        unsigned long elen,
                        const unsigned char *ndata,
                        unsigned long nlen,
                        const unsigned char *ddata,
                        unsigned long dlen,
                        const unsigned char *pdata,
                        unsigned long plen,
                        const unsigned char *qdata,
                        unsigned long qlen,
                        const unsigned char *e1data,
                        unsigned long e1len,
                        const unsigned char *e2data,
                        unsigned long e2len,
                        const unsigned char *coeffdata,
                        unsigned long coefflen)
{
    BCRYPT_KEY_HANDLE hKey; // BCRYPT 密钥句柄
    BCRYPT_RSAKEY_BLOB *rsakey; // BCRYPT RSA 密钥结构
    LPCWSTR lpszBlobType; // BLOB 类型
    unsigned char *key; // 密钥
    unsigned long keylen, offset, mlen, p1len = 0, p2len = 0; // 密钥长度、偏移量、m长度、p1长度、p2长度
    int ret; // 返回值

    // 计算 m 的长度
    mlen = max(_libssh2_wincng_bn_size(ndata, nlen),
               _libssh2_wincng_bn_size(ddata, dlen));
    // 计算密钥长度
    offset = sizeof(BCRYPT_RSAKEY_BLOB);
    keylen = offset + elen + mlen;
    // 如果存在 d 数据且长度大于0
    if(ddata && dlen > 0) {
        // 计算 p1 和 p2 的长度
        p1len = max(_libssh2_wincng_bn_size(pdata, plen),
                    _libssh2_wincng_bn_size(e1data, e1len));
        p2len = max(_libssh2_wincng_bn_size(qdata, qlen),
                    _libssh2_wincng_bn_size(e2data, e2len));
        // 更新密钥长度
        keylen += p1len * 3 + p2len * 2 + mlen;
    // 分配内存空间，用于存储密钥
    key = malloc(keylen);
    // 如果分配内存失败，返回错误代码-1
    if(!key) {
        return -1;
    }

    // 将密钥内存空间初始化为0
    memset(key, 0, keylen);

    // 设置 RSA 密钥的一些属性，如长度、公钥指数、模数长度等
    rsakey = (BCRYPT_RSAKEY_BLOB *)key;
    rsakey->BitLength = mlen * 8;
    rsakey->cbPublicExp = elen;
    rsakey->cbModulus = mlen;

    // 将公钥指数数据拷贝到密钥内存空间中
    memcpy(key + offset, edata, elen);
    offset += elen;

    // 根据情况将模数数据拷贝到密钥内存空间中
    if(nlen < mlen)
        memcpy(key + offset + mlen - nlen, ndata, nlen);
    else
        memcpy(key + offset, ndata + nlen - mlen, mlen);

    // 如果存在私钥数据，则将私钥数据拷贝到密钥内存空间中
    if(ddata && dlen > 0) {
        offset += mlen;

        // 根据情况将 p 数据拷贝到密钥内存空间中
        if(plen < p1len)
            memcpy(key + offset + p1len - plen, pdata, plen);
        else
            memcpy(key + offset, pdata + plen - p1len, p1len);
        offset += p1len;

        // 根据情况将 q 数据拷贝到密钥内存空间中
        if(qlen < p2len)
            memcpy(key + offset + p2len - qlen, qdata, qlen);
        else
            memcpy(key + offset, qdata + qlen - p2len, p2len);
        offset += p2len;

        // 根据情况将 e1 数据拷贝到密钥内存空间中
        if(e1len < p1len)
            memcpy(key + offset + p1len - e1len, e1data, e1len);
        else
            memcpy(key + offset, e1data + e1len - p1len, p1len);
        offset += p1len;

        // 根据情况将 e2 数据拷贝到密钥内存空间中
        if(e2len < p2len)
            memcpy(key + offset + p2len - e2len, e2data, e2len);
        else
            memcpy(key + offset, e2data + e2len - p2len, p2len);
        offset += p2len;

        // 根据情况将 coeff 数据拷贝到密钥内存空间中
        if(coefflen < p1len)
            memcpy(key + offset + p1len - coefflen, coeffdata, coefflen);
        else
            memcpy(key + offset, coeffdata + coefflen - p1len, p1len);
        offset += p1len;

        // 根据情况将私钥数据拷贝到密钥内存空间中
        if(dlen < mlen)
            memcpy(key + offset + mlen - dlen, ddata, dlen);
        else
            memcpy(key + offset, ddata + dlen - mlen, mlen);

        // 设置密钥类型为 RSA 私钥类型
        lpszBlobType = BCRYPT_RSAFULLPRIVATE_BLOB;
        // 设置 RSA 密钥的魔数
        rsakey->Magic = BCRYPT_RSAFULLPRIVATE_MAGIC;
        // 设置 RSA 密钥的素数1长度和素数2长度
        rsakey->cbPrime1 = p1len;
        rsakey->cbPrime2 = p2len;
    }
    // 如果条件不成立，设置 lpszBlobType 为 BCRYPT_RSAPUBLIC_BLOB
    // 设置 rsakey 的 Magic 字段为 BCRYPT_RSAPUBLIC_MAGIC
    // 设置 rsakey 的 cbPrime1 和 cbPrime2 字段为 0
    else {
        lpszBlobType = BCRYPT_RSAPUBLIC_BLOB;
        rsakey->Magic = BCRYPT_RSAPUBLIC_MAGIC;
        rsakey->cbPrime1 = 0;
        rsakey->cbPrime2 = 0;
    }

    // 导入密钥对，根据返回值判断是否成功
    ret = BCryptImportKeyPair(_libssh2_wincng.hAlgRSA, NULL, lpszBlobType,
                              &hKey, key, keylen, 0);
    // 如果导入密钥对不成功，释放资源并返回错误
    if(!BCRYPT_SUCCESS(ret)) {
        _libssh2_wincng_safe_free(key, keylen);
        return -1;
    }

    // 分配内存给 rsa 指针
    *rsa = malloc(sizeof(libssh2_rsa_ctx));
    // 如果内存分配不成功，释放资源并返回错误
    if(!(*rsa)) {
        BCryptDestroyKey(hKey);
        _libssh2_wincng_safe_free(key, keylen);
        return -1;
    }

    // 设置 (*rsa) 的 hKey、pbKeyObject 和 cbKeyObject 字段
    (*rsa)->hKey = hKey;
    (*rsa)->pbKeyObject = key;
    (*rsa)->cbKeyObject = keylen;

    // 返回成功
    return 0;
#ifdef HAVE_LIBCRYPT32
// 如果系统支持 libcrypt32 库，则定义静态函数 _libssh2_wincng_rsa_new_private_parse
static int
_libssh2_wincng_rsa_new_private_parse(libssh2_rsa_ctx **rsa,
                                      LIBSSH2_SESSION *session,
                                      unsigned char *pbEncoded,
                                      unsigned long cbEncoded)
{
    BCRYPT_KEY_HANDLE hKey; // 定义 BCRYPT_KEY_HANDLE 类型的变量 hKey
    unsigned char *pbStructInfo; // 定义指向无符号字符的指针变量 pbStructInfo
    unsigned long cbStructInfo; // 定义无符号长整型变量 cbStructInfo
    int ret; // 定义整型变量 ret

    (void)session; // 忽略未使用的参数 session

    // 调用 _libssh2_wincng_asn_decode 函数解码 pbEncoded，得到结构信息和结构信息的长度
    ret = _libssh2_wincng_asn_decode(pbEncoded, cbEncoded,
                                     PKCS_RSA_PRIVATE_KEY,
                                     &pbStructInfo, &cbStructInfo);

    _libssh2_wincng_safe_free(pbEncoded, cbEncoded); // 安全释放 pbEncoded 的内存

    if(ret) { // 如果 ret 不为 0
        return -1; // 返回 -1
    }

    // 导入 RSA 私钥对
    ret = BCryptImportKeyPair(_libssh2_wincng.hAlgRSA, NULL,
                              LEGACY_RSAPRIVATE_BLOB, &hKey,
                              pbStructInfo, cbStructInfo, 0);
    if(!BCRYPT_SUCCESS(ret)) { // 如果导入失败
        _libssh2_wincng_safe_free(pbStructInfo, cbStructInfo); // 安全释放 pbStructInfo 的内存
        return -1; // 返回 -1
    }

    // 分配内存给 rsa
    *rsa = malloc(sizeof(libssh2_rsa_ctx));
    if(!(*rsa)) { // 如果分配失败
        BCryptDestroyKey(hKey); // 销毁 hKey
        _libssh2_wincng_safe_free(pbStructInfo, cbStructInfo); // 安全释放 pbStructInfo 的内存
        return -1; // 返回 -1
    }

    (*rsa)->hKey = hKey; // 将 hKey 赋值给 rsa 的 hKey 成员
    (*rsa)->pbKeyObject = pbStructInfo; // 将 pbStructInfo 赋值给 rsa 的 pbKeyObject 成员
    (*rsa)->cbKeyObject = cbStructInfo; // 将 cbStructInfo 赋值给 rsa 的 cbKeyObject 成员

    return 0; // 返回 0
}
#endif /* HAVE_LIBCRYPT32 */

// 定义函数 _libssh2_wincng_rsa_new_private，用于加载 RSA 私钥
int
_libssh2_wincng_rsa_new_private(libssh2_rsa_ctx **rsa,
                                LIBSSH2_SESSION *session,
                                const char *filename,
                                const unsigned char *passphrase)
{
#ifdef HAVE_LIBCRYPT32
    unsigned char *pbEncoded; // 定义指向无符号字符的指针变量 pbEncoded
    unsigned long cbEncoded; // 定义无符号长整型变量 cbEncoded
    int ret; // 定义整型变量 ret

    (void)session; // 忽略未使用的参数 session

    // 调用 _libssh2_wincng_load_private 函数加载 RSA 私钥
    ret = _libssh2_wincng_load_private(session, filename,
                                       (const char *)passphrase,
                                       &pbEncoded, &cbEncoded, 1, 0);
    if(ret) { // 如果加载失败
        return -1; // 返回 -1
    }
    # 调用_libssh2_wincng_rsa_new_private_parse函数，传入参数rsa, session, pbEncoded, cbEncoded，并返回结果
    return _libssh2_wincng_rsa_new_private_parse(rsa, session, pbEncoded, cbEncoded);
#else
    // 如果不是在 Windows CNG 后端，则忽略以下代码
    (void)rsa;
    (void)filename;
    (void)passphrase;

    // 返回错误信息，表示在 Windows CNG 后端不支持该方法
    return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                          "Unable to load RSA key from private key file: "
                          "Method unsupported in Windows CNG backend");
#endif /* HAVE_LIBCRYPT32 */
}

// 从内存中加载私钥并创建 RSA 对象
int
_libssh2_wincng_rsa_new_private_frommemory(libssh2_rsa_ctx **rsa,
                                           LIBSSH2_SESSION *session,
                                           const char *filedata,
                                           size_t filedata_len,
                                           unsigned const char *passphrase)
{
#ifdef HAVE_LIBCRYPT32
    unsigned char *pbEncoded;
    unsigned long cbEncoded;
    int ret;

    (void)session;

    // 调用函数从内存中加载私钥
    ret = _libssh2_wincng_load_private_memory(session, filedata, filedata_len,
                                              (const char *)passphrase,
                                              &pbEncoded, &cbEncoded, 1, 0);
    if(ret) {
        return -1;
    }

    // 调用函数解析私钥并创建 RSA 对象
    return _libssh2_wincng_rsa_new_private_parse(rsa, session,
                                                 pbEncoded, cbEncoded);
#else
    // 如果不是在 Windows CNG 后端，则忽略以下代码
    (void)rsa;
    (void)filedata;
    (void)filedata_len;
    (void)passphrase;

    // 返回错误信息，表示在 Windows CNG 后端不支持该方法
    return _libssh2_error(session, LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                          "Unable to extract private key from memory: "
                          "Method unsupported in Windows CNG backend");
#endif /* HAVE_LIBCRYPT32 */
}

// 使用 SHA1 算法验证签名
int
_libssh2_wincng_rsa_sha1_verify(libssh2_rsa_ctx *rsa,
                                const unsigned char *sig,
                                unsigned long sig_len,
                                const unsigned char *m,
                                unsigned long m_len)
{
    // 调用函数使用 SHA1 算法验证签名
    return _libssh2_wincng_key_sha1_verify(rsa, sig, sig_len, m, m_len,
                                           BCRYPT_PAD_PKCS1);
}

int
# 使用 Windows CNG 进行 RSA-SHA1 签名
_libssh2_wincng_rsa_sha1_sign(LIBSSH2_SESSION *session,
                              libssh2_rsa_ctx *rsa,
                              const unsigned char *hash,
                              size_t hash_len,
                              unsigned char **signature,
                              size_t *signature_len)
{
    BCRYPT_PKCS1_PADDING_INFO paddingInfo;  # 定义 PKCS1 填充信息结构体
    unsigned char *data, *sig;  # 定义数据和签名的指针
    unsigned long cbData, datalen, siglen;  # 定义数据长度和签名长度
    int ret;  # 定义返回值

    datalen = (unsigned long)hash_len;  # 将哈希长度转换为无符号长整型
    data = malloc(datalen);  # 分配内存空间存储数据
    if(!data) {  # 如果内存分配失败
        return -1;  # 返回错误码
    }

    paddingInfo.pszAlgId = BCRYPT_SHA1_ALGORITHM;  # 设置填充信息的哈希算法为 SHA1

    memcpy(data, hash, datalen);  # 将哈希数据复制到数据指针中

    ret = BCryptSignHash(rsa->hKey, &paddingInfo,  # 使用私钥对数据进行签名
                         data, datalen, NULL, 0,
                         &cbData, BCRYPT_PAD_PKCS1);
    if(BCRYPT_SUCCESS(ret)) {  # 如果签名成功
        siglen = cbData;  # 获取签名长度
        sig = LIBSSH2_ALLOC(session, siglen);  # 分配内存空间存储签名
        if(sig) {  # 如果内存分配成功
            ret = BCryptSignHash(rsa->hKey, &paddingInfo,  # 再次使用私钥对数据进行签名
                                 data, datalen, sig, siglen,
                                 &cbData, BCRYPT_PAD_PKCS1);
            if(BCRYPT_SUCCESS(ret)) {  # 如果签名成功
                *signature_len = siglen;  # 设置签名长度
                *signature = sig;  # 设置签名数据
            }
            else {
                LIBSSH2_FREE(session, sig);  # 释放签名内存空间
            }
        }
        else
            ret = STATUS_NO_MEMORY;  # 如果内存分配失败，设置错误码
    }

    _libssh2_wincng_safe_free(data, datalen);  # 安全释放数据内存空间

    return BCRYPT_SUCCESS(ret) ? 0 : -1;  # 根据签名结果返回相应的错误码
}

# 释放 RSA 上下文
void
_libssh2_wincng_rsa_free(libssh2_rsa_ctx *rsa)
{
    if(!rsa)  # 如果 RSA 上下文为空
        return;  # 直接返回

    BCryptDestroyKey(rsa->hKey);  # 销毁 RSA 密钥
    rsa->hKey = NULL;  # 将 RSA 密钥指针置空

    _libssh2_wincng_safe_free(rsa->pbKeyObject, rsa->cbKeyObject);  # 安全释放 RSA 密钥对象内存空间
    _libssh2_wincng_safe_free(rsa, sizeof(libssh2_rsa_ctx));  # 安全释放 RSA 上下文内存空间
}


/*******************************************************************/
/*
 * Windows CNG backend: DSA functions
 */

#if LIBSSH2_DSA
int
# 创建一个新的 DSA 密钥对
def _libssh2_wincng_dsa_new(libssh2_dsa_ctx **dsa,
                            const unsigned char *pdata,
                            unsigned long plen,
                            const unsigned char *qdata,
                            unsigned long qlen,
                            const unsigned char *gdata,
                            unsigned long glen,
                            const unsigned char *ydata,
                            unsigned long ylen,
                            const unsigned char *xdata,
                            unsigned long xlen)
{
    # 定义变量
    BCRYPT_KEY_HANDLE hKey;
    BCRYPT_DSA_KEY_BLOB *dsakey;
    LPCWSTR lpszBlobType;
    unsigned char *key;
    unsigned long keylen, offset, length;
    int ret;

    # 计算密钥长度
    length = max(max(_libssh2_wincng_bn_size(pdata, plen),
                     _libssh2_wincng_bn_size(gdata, glen)),
                 _libssh2_wincng_bn_size(ydata, ylen));
    offset = sizeof(BCRYPT_DSA_KEY_BLOB);
    keylen = offset + length * 3;
    if(xdata && xlen > 0)
        keylen += 20;

    # 分配内存
    key = malloc(keylen);
    if(!key) {
        return -1;
    }

    # 初始化内存
    memset(key, 0, keylen);

    # 设置 DSA 密钥结构
    dsakey = (BCRYPT_DSA_KEY_BLOB *)key;
    dsakey->cbKey = length;
    memset(dsakey->Count, -1, sizeof(dsakey->Count));
    memset(dsakey->Seed, -1, sizeof(dsakey->Seed));

    # 复制 qdata 到 dsakey->q
    if(qlen < 20)
        memcpy(dsakey->q + 20 - qlen, qdata, qlen);
    else
        memcpy(dsakey->q, qdata + qlen - 20, 20);

    # 复制 pdata 到 key
    if(plen < length)
        memcpy(key + offset + length - plen, pdata, plen);
    else
        memcpy(key + offset, pdata + plen - length, length);
    offset += length;

    # 复制 gdata 到 key
    if(glen < length)
        memcpy(key + offset + length - glen, gdata, glen);
    else
        memcpy(key + offset, gdata + glen - length, length);
    offset += length;

    # 复制 ydata 到 key
    if(ylen < length)
        memcpy(key + offset + length - ylen, ydata, ylen);
    else
        memcpy(key + offset, ydata + ylen - length, length);
    # 如果 xdata 存在且 xlen 大于 0
    if(xdata && xlen > 0) {
        # 偏移量增加 length
        offset += length;

        # 如果 xlen 小于 20，则将 xdata 复制到 key 的末尾 20-xlen 的位置
        if(xlen < 20)
            memcpy(key + offset + 20 - xlen, xdata, xlen);
        # 否则，将 xdata 的末尾 20 个字节复制到 key 的偏移位置
        else
            memcpy(key + offset, xdata + xlen - 20, 20);

        # 设置 lpszBlobType 为 BCRYPT_DSA_PRIVATE_BLOB
        lpszBlobType = BCRYPT_DSA_PRIVATE_BLOB;
        # 设置 dsakey 的 dwMagic 为 BCRYPT_DSA_PRIVATE_MAGIC
        dsakey->dwMagic = BCRYPT_DSA_PRIVATE_MAGIC;
    }
    # 如果 xdata 不存在或者 xlen 小于等于 0
    else {
        # 设置 lpszBlobType 为 BCRYPT_DSA_PUBLIC_BLOB
        lpszBlobType = BCRYPT_DSA_PUBLIC_BLOB;
        # 设置 dsakey 的 dwMagic 为 BCRYPT_DSA_PUBLIC_MAGIC
        dsakey->dwMagic = BCRYPT_DSA_PUBLIC_MAGIC;
    }

    # 导入密钥对到 hKey
    ret = BCryptImportKeyPair(_libssh2_wincng.hAlgDSA, NULL, lpszBlobType,
                              &hKey, key, keylen, 0);
    # 如果导入密钥对不成功
    if(!BCRYPT_SUCCESS(ret)) {
        # 释放 key 的内存
        _libssh2_wincng_safe_free(key, keylen);
        # 返回 -1
        return -1;
    }

    # 分配内存给 dsa
    *dsa = malloc(sizeof(libssh2_dsa_ctx));
    # 如果分配内存不成功
    if(!(*dsa)) {
        # 销毁 hKey
        BCryptDestroyKey(hKey);
        # 释放 key 的内存
        _libssh2_wincng_safe_free(key, keylen);
        # 返回 -1
        return -1;
    }

    # 设置 (*dsa) 的 hKey 为 hKey
    (*dsa)->hKey = hKey;
    # 设置 (*dsa) 的 pbKeyObject 为 key
    (*dsa)->pbKeyObject = key;
    # 设置 (*dsa) 的 cbKeyObject 为 keylen
    (*dsa)->cbKeyObject = keylen;

    # 返回 0
    return 0;
#ifdef HAVE_LIBCRYPT32
// 如果系统支持 libcrypt32 库，则定义静态函数 _libssh2_wincng_dsa_new_private_parse
static int
_libssh2_wincng_dsa_new_private_parse(libssh2_dsa_ctx **dsa,
                                      LIBSSH2_SESSION *session,
                                      unsigned char *pbEncoded,
                                      unsigned long cbEncoded)
{
    unsigned char **rpbDecoded; // 定义指向指针的指针，用于存储解码后的数据
    unsigned long *rcbDecoded, index, length; // 定义指向无符号长整型的指针和变量 index、length
    int ret; // 定义整型变量 ret，用于存储函数返回值

    (void)session; // 忽略未使用的参数 session

    // 调用 _libssh2_wincng_asn_decode_bns 函数解码 pbEncoded，将解码后的数据存储在 rpbDecoded 和 rcbDecoded 中
    ret = _libssh2_wincng_asn_decode_bns(pbEncoded, cbEncoded,
                                         &rpbDecoded, &rcbDecoded, &length);

    // 安全释放 pbEncoded 和 cbEncoded
    _libssh2_wincng_safe_free(pbEncoded, cbEncoded);

    // 如果解码失败，则返回 -1
    if(ret) {
        return -1;
    }

    // 根据解码后的数据长度判断是否为 DSA 密钥
    if(length == 6) {
        // 调用 _libssh2_wincng_dsa_new 函数创建 DSA 密钥
        ret = _libssh2_wincng_dsa_new(dsa,
                                      rpbDecoded[1], rcbDecoded[1],
                                      rpbDecoded[2], rcbDecoded[2],
                                      rpbDecoded[3], rcbDecoded[3],
                                      rpbDecoded[4], rcbDecoded[4],
                                      rpbDecoded[5], rcbDecoded[5]);
    }
    else {
        ret = -1; // 如果长度不符合 DSA 密钥的要求，则返回 -1
    }

    // 循环释放解码后的数据
    for(index = 0; index < length; index++) {
        _libssh2_wincng_safe_free(rpbDecoded[index], rcbDecoded[index]);
        rpbDecoded[index] = NULL;
        rcbDecoded[index] = 0;
    }

    free(rpbDecoded); // 释放 rpbDecoded 指针
    free(rcbDecoded); // 释放 rcbDecoded 指针

    return ret; // 返回函数执行结果
}
#endif /* HAVE_LIBCRYPT32 */

// 定义函数 _libssh2_wincng_dsa_new_private，用于创建 DSA 密钥
int
_libssh2_wincng_dsa_new_private(libssh2_dsa_ctx **dsa,
                                LIBSSH2_SESSION *session,
                                const char *filename,
                                const unsigned char *passphrase)
{
#ifdef HAVE_LIBCRYPT32
    unsigned char *pbEncoded; // 定义指向无符号字符的指针，用于存储编码后的数据
    unsigned long cbEncoded; // 定义无符号长整型变量，用于存储编码后的数据长度
    int ret; // 定义整型变量 ret，用于存储函数返回值

    // 调用 _libssh2_wincng_load_private 函数加载私钥文件，获取编码后的数据和数据长度
    ret = _libssh2_wincng_load_private(session, filename,
                                       (const char *)passphrase,
                                       &pbEncoded, &cbEncoded, 0, 1);
    // 如果加载私钥文件失败，则返回 -1
    if(ret) {
        return -1;
    }
    # 调用_libssh2_wincng_dsa_new_private_parse函数，传入参数dsa, session, pbEncoded, cbEncoded，并返回结果
    return _libssh2_wincng_dsa_new_private_parse(dsa, session, pbEncoded, cbEncoded);
#else
    // 如果不支持 Windows CNG 后端，则忽略以下代码
    (void)dsa;
    (void)filename;
    (void)passphrase;
    // 返回错误信息，说明在 Windows CNG 后端不支持加载 DSA 密钥
    return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                          "Unable to load DSA key from private key file: "
                          "Method unsupported in Windows CNG backend");
#endif /* HAVE_LIBCRYPT32 */
}

int
// 从内存中加载 DSA 私钥
_libssh2_wincng_dsa_new_private_frommemory(libssh2_dsa_ctx **dsa,
                                           LIBSSH2_SESSION *session,
                                           const char *filedata,
                                           size_t filedata_len,
                                           unsigned const char *passphrase)
{
#ifdef HAVE_LIBCRYPT32
    unsigned char *pbEncoded;
    unsigned long cbEncoded;
    int ret;
    // 调用函数从内存中加载私钥
    ret = _libssh2_wincng_load_private_memory(session, filedata, filedata_len,
                                              (const char *)passphrase,
                                              &pbEncoded, &cbEncoded, 0, 1);
    // 如果加载失败，则返回错误
    if(ret) {
        return -1;
    }
    // 调用函数解析私钥并创建 DSA 上下文
    return _libssh2_wincng_dsa_new_private_parse(dsa, session,
                                                 pbEncoded, cbEncoded);
#else
    // 如果不支持 Windows CNG 后端，则忽略以下代码
    (void)dsa;
    (void)filedata;
    (void)filedata_len;
    (void)passphrase;
    // 返回错误信息，说明在 Windows CNG 后端不支持从内存中提取私钥
    return _libssh2_error(session, LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                          "Unable to extract private key from memory: "
                          "Method unsupported in Windows CNG backend");
#endif /* HAVE_LIBCRYPT32 */
}

int
// 使用 SHA1 验证 DSA 签名
_libssh2_wincng_dsa_sha1_verify(libssh2_dsa_ctx *dsa,
                                const unsigned char *sig_fixed,
                                const unsigned char *m,
                                unsigned long m_len)
{
    // 调用函数使用 SHA1 验证 DSA 签名
    return _libssh2_wincng_key_sha1_verify(dsa, sig_fixed, 40, m, m_len, 0);
}

int
# 使用 libssh2_dsa_ctx 结构体和哈希值对 DSA-SHA1 签名进行操作
_libssh2_wincng_dsa_sha1_sign(libssh2_dsa_ctx *dsa,
                              const unsigned char *hash,
                              unsigned long hash_len,
                              unsigned char *sig_fixed)
{
    unsigned char *data, *sig;  # 声明指向无符号字符的指针变量
    unsigned long cbData, datalen, siglen;  # 声明无符号长整型变量
    int ret;  # 声明整型变量

    datalen = hash_len;  # 将哈希值长度赋给 datalen
    data = malloc(datalen);  # 分配 datalen 大小的内存空间给 data
    if(!data) {  # 如果 data 为空
        return -1;  # 返回 -1
    }

    memcpy(data, hash, datalen);  # 将哈希值拷贝到 data

    ret = BCryptSignHash(dsa->hKey, NULL, data, datalen,
                         NULL, 0, &cbData, 0);  # 使用 BCryptSignHash 对哈希值进行签名
    if(BCRYPT_SUCCESS(ret)) {  # 如果签名成功
        siglen = cbData;  # 将签名长度赋给 siglen
        if(siglen == 40) {  # 如果签名长度为 40
            sig = malloc(siglen);  # 分配 siglen 大小的内存空间给 sig
            if(sig) {  # 如果 sig 不为空
                ret = BCryptSignHash(dsa->hKey, NULL, data, datalen,
                                     sig, siglen, &cbData, 0);  # 使用 BCryptSignHash 对哈希值进行签名
                if(BCRYPT_SUCCESS(ret)) {  # 如果签名成功
                    memcpy(sig_fixed, sig, siglen);  # 将签名拷贝到 sig_fixed
                }

                _libssh2_wincng_safe_free(sig, siglen);  # 释放 sig 占用的内存空间
            }
            else
                ret = STATUS_NO_MEMORY;  # 如果 sig 为空，返回内存不足的状态
        }
        else
            ret = STATUS_NO_MEMORY;  # 如果签名长度不为 40，返回内存不足的状态
    }

    _libssh2_wincng_safe_free(data, datalen);  # 释放 data 占用的内存空间

    return BCRYPT_SUCCESS(ret) ? 0 : -1;  # 如果签名成功，返回 0，否则返回 -1
}

# 释放 libssh2_dsa_ctx 结构体占用的内存空间
void
_libssh2_wincng_dsa_free(libssh2_dsa_ctx *dsa)
{
    if(!dsa)  # 如果 dsa 为空
        return;  # 直接返回

    BCryptDestroyKey(dsa->hKey);  # 销毁 DSA 密钥
    dsa->hKey = NULL;  # 将 DSA 密钥置空

    _libssh2_wincng_safe_free(dsa->pbKeyObject, dsa->cbKeyObject);  # 释放 DSA 密钥对象占用的内存空间
    _libssh2_wincng_safe_free(dsa, sizeof(libssh2_dsa_ctx));  # 释放 libssh2_dsa_ctx 结构体占用的内存空间
}
#endif


/*******************************************************************/
/*
 * Windows CNG backend: Key functions
 */

#ifdef HAVE_LIBCRYPT32
static unsigned long
_libssh2_wincng_pub_priv_write(unsigned char *key,
                               unsigned long offset,
                               const unsigned char *bignum,
                               const unsigned long length)
{
    _libssh2_htonu32(key + offset, length);  # 将长度转换为网络字节顺序，并写入 key
    offset += 4;  # 偏移量增加 4

    memcpy(key + offset, bignum, length);  # 将 bignum 拷贝到 key 的偏移位置
    offset += length;  # 偏移量增加 bignum 的长度
    # 返回变量 offset 的值
    return offset;
    # 解析 Windows CNG 公钥/私钥文件
    static int
    _libssh2_wincng_pub_priv_keyfile_parse(LIBSSH2_SESSION *session,
                                           unsigned char **method,
                                           size_t *method_len,
                                           unsigned char **pubkeydata,
                                           size_t *pubkeydata_len,
                                           unsigned char *pbEncoded,
                                           unsigned long cbEncoded)
    {
        unsigned char **rpbDecoded;
        unsigned long *rcbDecoded;
        unsigned char *key = NULL, *mth = NULL;
        unsigned long keylen = 0, mthlen = 0;
        unsigned long index, offset, length;
        int ret;

        # 调用 _libssh2_wincng_asn_decode_bns 函数解码 ASN.1 编码的数据
        ret = _libssh2_wincng_asn_decode_bns(pbEncoded, cbEncoded,
                                             &rpbDecoded, &rcbDecoded, &length);

        # 释放原始编码数据的内存
        _libssh2_wincng_safe_free(pbEncoded, cbEncoded);

        # 如果解码失败，返回错误
        if(ret) {
            return -1;
        }

        # 如果长度为 9，表示是私钥 RSA 密钥
        if(length == 9) { 
            mthlen = 7;
            # 分配内存并拷贝 "ssh-rsa" 到 mth 中
            mth = LIBSSH2_ALLOC(session, mthlen);
            if(mth) {
                memcpy(mth, "ssh-rsa", mthlen);
            }
            else {
                ret = -1;
            }

            # 计算私钥长度并分配内存
            keylen = 4 + mthlen + 4 + rcbDecoded[2] + 4 + rcbDecoded[1];
            key = LIBSSH2_ALLOC(session, keylen);
            if(key) {
                # 将数据写入私钥缓冲区
                offset = _libssh2_wincng_pub_priv_write(key, 0, mth, mthlen);
                offset = _libssh2_wincng_pub_priv_write(key, offset,
                                                        rpbDecoded[2],
                                                        rcbDecoded[2]);
                _libssh2_wincng_pub_priv_write(key, offset,
                                               rpbDecoded[1],
                                               rcbDecoded[1]);
            }
            else {
                ret = -1;
            }
        }
    }
    # 如果长度为6，表示是私有的DSA密钥
    else if(length == 6) { /* private DSA key */
        # 设置方法长度为7
        mthlen = 7;
        # 分配内存给方法
        mth = LIBSSH2_ALLOC(session, mthlen);
        # 如果成功分配内存，则将"ssh-dss"复制到方法中
        if(mth) {
            memcpy(mth, "ssh-dss", mthlen);
        }
        # 如果分配内存失败，则返回-1
        else {
            ret = -1;
        }

        # 计算密钥长度
        keylen = 4 + mthlen + 4 + rcbDecoded[1] + 4 + rcbDecoded[2]
                            + 4 + rcbDecoded[3] + 4 + rcbDecoded[4];
        # 分配内存给密钥
        key = LIBSSH2_ALLOC(session, keylen);
        # 如果成功分配内存，则按顺序将解码后的数据写入密钥中
        if(key) {
            offset = _libssh2_wincng_pub_priv_write(key, 0, mth, mthlen);

            offset = _libssh2_wincng_pub_priv_write(key, offset,
                                                    rpbDecoded[1],
                                                    rcbDecoded[1]);

            offset = _libssh2_wincng_pub_priv_write(key, offset,
                                                    rpbDecoded[2],
                                                    rcbDecoded[2]);

            offset = _libssh2_wincng_pub_priv_write(key, offset,
                                                    rpbDecoded[3],
                                                    rcbDecoded[3]);

            _libssh2_wincng_pub_priv_write(key, offset,
                                           rpbDecoded[4],
                                           rcbDecoded[4]);
        }
        # 如果分配内存失败，则返回-1
        else {
            ret = -1;
        }

    }
    # 如果长度不为6，则返回-1
    else {
        ret = -1;
    }

    # 释放解码后的数据
    for(index = 0; index < length; index++) {
        _libssh2_wincng_safe_free(rpbDecoded[index], rcbDecoded[index]);
        rpbDecoded[index] = NULL;
        rcbDecoded[index] = 0;
    }

    # 释放解码后的数据数组
    free(rpbDecoded);
    free(rcbDecoded);

    # 如果ret不为0，则释放内存，否则将方法和密钥赋值给指针
    if(ret) {
        if(mth)
            LIBSSH2_FREE(session, mth);
        if(key)
            LIBSSH2_FREE(session, key);
    }
    else {
        *method = mth;
        *method_len = mthlen;
        *pubkeydata = key;
        *pubkeydata_len = keylen;
    }

    # 返回ret
    return ret;
}
#endif /* HAVE_LIBCRYPT32 */

int
_libssh2_wincng_pub_priv_keyfile(LIBSSH2_SESSION *session,
                                 unsigned char **method,
                                 size_t *method_len,
                                 unsigned char **pubkeydata,
                                 size_t *pubkeydata_len,
                                 const char *privatekey,
                                 const char *passphrase)
{
#ifdef HAVE_LIBCRYPT32
    // 声明变量
    unsigned char *pbEncoded;
    unsigned long cbEncoded;
    int ret;

    // 调用_libssh2_wincng_load_private函数加载私钥
    ret = _libssh2_wincng_load_private(session, privatekey, passphrase,
                                       &pbEncoded, &cbEncoded, 1, 1);
    // 如果加载失败，返回-1
    if(ret) {
        return -1;
    }

    // 调用_libssh2_wincng_pub_priv_keyfile_parse函数解析公私钥文件
    return _libssh2_wincng_pub_priv_keyfile_parse(session, method, method_len,
                                                  pubkeydata, pubkeydata_len,
                                                  pbEncoded, cbEncoded);
#else
    // 如果没有支持的库，则忽略参数并返回错误信息
    (void)method;
    (void)method_len;
    (void)pubkeydata;
    (void)pubkeydata_len;
    (void)privatekey;
    (void)passphrase;

    return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                          "Unable to load public key from private key file: "
                          "Method unsupported in Windows CNG backend");
#endif /* HAVE_LIBCRYPT32 */
}

int
_libssh2_wincng_pub_priv_keyfilememory(LIBSSH2_SESSION *session,
                                       unsigned char **method,
                                       size_t *method_len,
                                       unsigned char **pubkeydata,
                                       size_t *pubkeydata_len,
                                       const char *privatekeydata,
                                       size_t privatekeydata_len,
                                       const char *passphrase)
{
#ifdef HAVE_LIBCRYPT32
    // 声明变量
    unsigned char *pbEncoded;
    unsigned long cbEncoded;
    int ret;
    # 调用_libssh2_wincng_load_private_memory函数，加载私钥数据到内存中
    ret = _libssh2_wincng_load_private_memory(session, privatekeydata,
                                              privatekeydata_len, passphrase,
                                              &pbEncoded, &cbEncoded, 1, 1);
    # 如果加载私钥数据失败，返回-1
    if(ret) {
        return -1;
    }
    
    # 调用_libssh2_wincng_pub_priv_keyfile_parse函数，解析公私钥文件
    return _libssh2_wincng_pub_priv_keyfile_parse(session, method, method_len,
                                                  pubkeydata, pubkeydata_len,
                                                  pbEncoded, cbEncoded);
#else
    // 忽略未使用的参数
    (void)method;
    (void)method_len;
    (void)pubkeydata_len;
    (void)pubkeydata;
    (void)privatekeydata;
    (void)privatekeydata_len;
    (void)passphrase;

    // 返回错误信息
    return _libssh2_error(session, LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
               "Unable to extract public key from private key in memory: "
               "Method unsupported in Windows CNG backend");
#endif /* HAVE_LIBCRYPT32 */
}

/*******************************************************************/
/*
 * Windows CNG backend: Cipher functions
 */

int
_libssh2_wincng_cipher_init(_libssh2_cipher_ctx *ctx,
                            _libssh2_cipher_type(type),
                            unsigned char *iv,
                            unsigned char *secret,
                            int encrypt)
{
    BCRYPT_KEY_HANDLE hKey;
    BCRYPT_KEY_DATA_BLOB_HEADER *header;
    unsigned char *pbKeyObject, *pbIV, *key, *pbCtr, *pbIVCopy;
    unsigned long dwKeyObject, dwIV, dwCtrLength, dwBlockLength,
                  cbData, keylen;
    int ret;

    // 忽略未使用的参数
    (void)encrypt;

    // 获取密钥对象长度
    ret = BCryptGetProperty(*type.phAlg, BCRYPT_OBJECT_LENGTH,
                            (unsigned char *)&dwKeyObject,
                            sizeof(dwKeyObject),
                            &cbData, 0);
    if(!BCRYPT_SUCCESS(ret)) {
        return -1;
    }

    // 获取加密块长度
    ret = BCryptGetProperty(*type.phAlg, BCRYPT_BLOCK_LENGTH,
                            (unsigned char *)&dwBlockLength,
                            sizeof(dwBlockLength),
                            &cbData, 0);
    if(!BCRYPT_SUCCESS(ret)) {
        return -1;
    }

    // 分配密钥对象内存
    pbKeyObject = malloc(dwKeyObject);
    if(!pbKeyObject) {
        return -1;
    }

    // 分配密钥内存
    keylen = sizeof(BCRYPT_KEY_DATA_BLOB_HEADER) + type.dwKeyLength;
    key = malloc(keylen);
    if(!key) {
        free(pbKeyObject);
        return -1;
    }

    // 设置密钥数据头部信息
    header = (BCRYPT_KEY_DATA_BLOB_HEADER *)key;
    header->dwMagic = BCRYPT_KEY_DATA_BLOB_MAGIC;
    header->dwVersion = BCRYPT_KEY_DATA_BLOB_VERSION1;
    # 设置 header->cbKeyData 为 type.dwKeyLength
    header->cbKeyData = type.dwKeyLength;

    # 将 secret 复制到 key 中，偏移 sizeof(BCRYPT_KEY_DATA_BLOB_HEADER) 个字节
    memcpy(key + sizeof(BCRYPT_KEY_DATA_BLOB_HEADER),
           secret, type.dwKeyLength);

    # 导入密钥到加密提供程序
    ret = BCryptImportKey(*type.phAlg, NULL, BCRYPT_KEY_DATA_BLOB, &hKey,
                          pbKeyObject, dwKeyObject, key, keylen, 0);

    # 释放 key 的内存
    _libssh2_wincng_safe_free(key, keylen);

    # 如果导入密钥不成功，则释放内存并返回错误
    if(!BCRYPT_SUCCESS(ret)) {
        _libssh2_wincng_safe_free(pbKeyObject, dwKeyObject);
        return -1;
    }

    # 初始化 pbIV, pbCtr, dwIV, dwCtrLength
    pbIV = NULL;
    pbCtr = NULL;
    dwIV = 0;
    dwCtrLength = 0;

    # 如果需要使用 IV 或者是计数器模式
    if(type.useIV || type.ctrMode) {
        # 分配内存给 pbIVCopy
        pbIVCopy = malloc(dwBlockLength);
        # 如果内存分配失败，则释放内存并返回错误
        if(!pbIVCopy) {
            BCryptDestroyKey(hKey);
            _libssh2_wincng_safe_free(pbKeyObject, dwKeyObject);
            return -1;
        }
        # 将 iv 复制到 pbIVCopy
        memcpy(pbIVCopy, iv, dwBlockLength);

        # 如果是计数器模式，则设置 pbCtr 和 dwCtrLength
        if(type.ctrMode) {
            pbCtr = pbIVCopy;
            dwCtrLength = dwBlockLength;
        }
        # 如果需要使用 IV，则设置 pbIV 和 dwIV
        else if(type.useIV) {
            pbIV = pbIVCopy;
            dwIV = dwBlockLength;
        }
    }

    # 设置 ctx 结构体的各个成员
    ctx->hKey = hKey;
    ctx->pbKeyObject = pbKeyObject;
    ctx->pbIV = pbIV;
    ctx->pbCtr = pbCtr;
    ctx->dwKeyObject = dwKeyObject;
    ctx->dwIV = dwIV;
    ctx->dwBlockLength = dwBlockLength;
    ctx->dwCtrLength = dwCtrLength;

    # 返回成功
    return 0;
}
int
_libssh2_wincng_cipher_crypt(_libssh2_cipher_ctx *ctx,
                             _libssh2_cipher_type(type),
                             int encrypt,
                             unsigned char *block,
                             size_t blocklen)
{
    unsigned char *pbOutput, *pbInput;  // 声明指向无符号字符的指针变量
    unsigned long cbOutput, cbInput;  // 声明无符号长整型变量
    int ret;  // 声明整型变量

    (void)type;  // 忽略未使用的参数

    cbInput = (unsigned long)blocklen;  // 将块长度转换为无符号长整型

    if(type.ctrMode) {  // 如果类型为计数器模式
        pbInput = ctx->pbCtr;  // 输入指针指向上下文的计数器
    }
    else {
        pbInput = block;  // 否则输入指针指向块
    }

    if(encrypt || type.ctrMode) {  // 如果是加密或者是计数器模式
        ret = BCryptEncrypt(ctx->hKey, pbInput, cbInput, NULL,
                            ctx->pbIV, ctx->dwIV, NULL, 0, &cbOutput, 0);  // 使用密钥和初始化向量加密数据
    }
    else {
        ret = BCryptDecrypt(ctx->hKey, pbInput, cbInput, NULL,
                            ctx->pbIV, ctx->dwIV, NULL, 0, &cbOutput, 0);  // 使用密钥和初始化向量解密数据
    }
    if(BCRYPT_SUCCESS(ret)) {  // 如果加密或解密成功
        pbOutput = malloc(cbOutput);  // 分配内存用于输出数据
        if(pbOutput) {  // 如果内存分配成功
            if(encrypt || type.ctrMode) {  // 如果是加密或者是计数器模式
                ret = BCryptEncrypt(ctx->hKey, pbInput, cbInput, NULL,
                                    ctx->pbIV, ctx->dwIV,
                                    pbOutput, cbOutput, &cbOutput, 0);  // 再次使用密钥和初始化向量加密数据
            }
            else {
                ret = BCryptDecrypt(ctx->hKey, pbInput, cbInput, NULL,
                                    ctx->pbIV, ctx->dwIV,
                                    pbOutput, cbOutput, &cbOutput, 0);  // 再次使用密钥和初始化向量解密数据
            }
            if(BCRYPT_SUCCESS(ret)) {  // 如果再次加密或解密成功
                if(type.ctrMode) {  // 如果是计数器模式
                    _libssh2_xor_data(block, block, pbOutput, blocklen);  // 对块进行异或操作
                    _libssh2_aes_ctr_increment(ctx->pbCtr, ctx->dwCtrLength);  // 对计数器进行增量操作
                }
                else {
                    memcpy(block, pbOutput, cbOutput);  // 复制输出数据到块
                }
            }

            _libssh2_wincng_safe_free(pbOutput, cbOutput);  // 释放输出数据的内存
        }
        else
            ret = STATUS_NO_MEMORY;  // 如果内存分配失败，则返回内存不足的错误状态
    }

    return BCRYPT_SUCCESS(ret) ? 0 : -1;  // 如果加密或解密成功则返回0，否则返回-1
}

void
_libssh2_wincng_cipher_dtor(_libssh2_cipher_ctx *ctx)  // 密码上下文的析构函数
{
    # 销毁密钥
    BCryptDestroyKey(ctx->hKey);
    ctx->hKey = NULL;

    # 释放内存并重置指针
    _libssh2_wincng_safe_free(ctx->pbKeyObject, ctx->dwKeyObject);
    ctx->pbKeyObject = NULL;
    ctx->dwKeyObject = 0;

    # 释放内存并重置指针
    _libssh2_wincng_safe_free(ctx->pbIV, ctx->dwBlockLength);
    ctx->pbIV = NULL;
    ctx->dwBlockLength = 0;

    # 释放内存并重置指针
    _libssh2_wincng_safe_free(ctx->pbCtr, ctx->dwCtrLength);
    ctx->pbCtr = NULL;
    ctx->dwCtrLength = 0;
}

/*******************************************************************/
/*
 * Windows CNG backend: BigNumber functions
 */

# 初始化大数
_libssh2_bn *
_libssh2_wincng_bignum_init(void)
{
    _libssh2_bn *bignum;

    bignum = (_libssh2_bn *)malloc(sizeof(_libssh2_bn));
    if(bignum) {
        bignum->bignum = NULL;
        bignum->length = 0;
    }

    return bignum;
}

# 调整大数的大小
static int
_libssh2_wincng_bignum_resize(_libssh2_bn *bn, unsigned long length)
{
    unsigned char *bignum;

    if(!bn)
        return -1;

    if(length == bn->length)
        return 0;

    # 清空内存
    if(bn->bignum && bn->length > 0 && length < bn->length) {
        SecureZeroMemory(bn->bignum + length, bn->length - length);
    }

    bignum = realloc(bn->bignum, length);
    if(!bignum)
        return -1;

    bn->bignum = bignum;
    bn->length = length;

    return 0;
}

# 生成随机大数
static int
_libssh2_wincng_bignum_rand(_libssh2_bn *rnd, int bits, int top, int bottom)
{
    unsigned char *bignum;
    unsigned long length;

    if(!rnd)
        return -1;

    length = (unsigned long) (ceil(((double)bits) / 8.0) *
                              sizeof(unsigned char));
    if(_libssh2_wincng_bignum_resize(rnd, length))
        return -1;

    bignum = rnd->bignum;

    if(_libssh2_wincng_random(bignum, length))
        return -1;

    # 计算最高字节中的有效位数
    bits %= 8;
    if(bits == 0)
        bits = 8;

    # 填充最高字节的零填充
    bignum[0] &= ((1 << bits) - 1);

    # 设置最高字节中的最高位
    # 如果最高位为0，则将最高位的值设为1左移(bits-1)位的结果
    if(top == 0)
        bignum[0] |= (1 << (bits - 1));
    # 如果最高位为1，则将最高位的值设为3左移(bits-2)位的结果
    else if(top == 1)
        bignum[0] |= (3 << (bits - 2));

    # 通过设置最低有效字节的第一位，使其变为奇数
    if(bottom)
        bignum[length - 1] |= 1;

    # 返回0，表示执行成功
    return 0;
    # 定义静态函数_libssh2_wincng_bignum_mod_exp，实现大数的模幂运算
    static int
    _libssh2_wincng_bignum_mod_exp(_libssh2_bn *r,
                                   _libssh2_bn *a,
                                   _libssh2_bn *p,
                                   _libssh2_bn *m)
    {
        # 定义变量
        BCRYPT_KEY_HANDLE hKey;
        BCRYPT_RSAKEY_BLOB *rsakey;
        unsigned char *key, *bignum;
        unsigned long keylen, offset, length;
        int ret;

        # 如果输入参数为空，则返回错误
        if(!r || !a || !p || !m)
            return -1;

        # 计算 RSA 密钥的长度
        offset = sizeof(BCRYPT_RSAKEY_BLOB);
        keylen = offset + p->length + m->length;

        # 分配内存空间用于存储 RSA 密钥
        key = malloc(keylen);
        if(!key)
            return -1;

        # 设置 RSA 密钥的各个字段
        rsakey = (BCRYPT_RSAKEY_BLOB *)key;
        rsakey->Magic = BCRYPT_RSAPUBLIC_MAGIC;
        rsakey->BitLength = m->length * 8;
        rsakey->cbPublicExp = p->length;
        rsakey->cbModulus = m->length;
        rsakey->cbPrime1 = 0;
        rsakey->cbPrime2 = 0;

        # 将 p 的大数数据拷贝到 key 中
        memcpy(key + offset, p->bignum, p->length);
        offset += p->length;

        # 将 m 的大数数据拷贝到 key 中
        memcpy(key + offset, m->bignum, m->length);
        offset = 0;

        # 导入 RSA 密钥对
        ret = BCryptImportKeyPair(_libssh2_wincng.hAlgRSA, NULL,
                                  BCRYPT_RSAPUBLIC_BLOB, &hKey, key, keylen, 0);
    }
    # 检查加密操作是否成功
    if(BCRYPT_SUCCESS(ret)) {
        # 使用密钥对数据进行加密
        ret = BCryptEncrypt(hKey, a->bignum, a->length, NULL, NULL, 0,
                            NULL, 0, &length, BCRYPT_PAD_NONE);
        # 如果加密成功
        if(BCRYPT_SUCCESS(ret)) {
            # 调整目标数据的大小
            if(!_libssh2_wincng_bignum_resize(r, length)) {
                # 计算目标数据的大小
                length = max(a->length, length);
                # 分配内存
                bignum = malloc(length);
                # 如果内存分配成功
                if(bignum) {
                    # 使用大端填充方式将源数据复制到目标数据
                    memcpy_with_be_padding(bignum, length,
                                           a->bignum, a->length);
                    # 使用密钥对目标数据进行加密
                    ret = BCryptEncrypt(hKey, bignum, length, NULL, NULL, 0,
                                        r->bignum, r->length, &offset,
                                        BCRYPT_PAD_NONE);
                    # 释放目标数据的内存
                    _libssh2_wincng_safe_free(bignum, length);
                    # 如果加密成功
                    if(BCRYPT_SUCCESS(ret)) {
                        # 调整目标数据的大小
                        _libssh2_wincng_bignum_resize(r, offset);
                    }
                }
                # 如果内存分配失败
                else
                    # 设置错误状态
                    ret = STATUS_NO_MEMORY;
            }
            # 如果调整目标数据大小失败
            else
                # 设置错误状态
                ret = STATUS_NO_MEMORY;
        }
        # 销毁密钥对象
        BCryptDestroyKey(hKey);
    }
    # 释放密钥内存
    _libssh2_wincng_safe_free(key, keylen);
    # 返回加密操作是否成功
    return BCRYPT_SUCCESS(ret) ? 0 : -1;
}

int
_libssh2_wincng_bignum_set_word(_libssh2_bn *bn, unsigned long word)
{
    unsigned long offset, number, bits, length;

    if(!bn)
        return -1;

    bits = 0;
    number = word;
    while(number >>= 1)  // 右移一位，计算 word 的二进制位数
        bits++;
    bits++;

    length = (unsigned long) (ceil(((double)bits) / 8.0) *  // 计算所需的字节数
                              sizeof(unsigned char));
    if(_libssh2_wincng_bignum_resize(bn, length))  // 调整 bn 的大小
        return -1;

    for(offset = 0; offset < length; offset++)  // 将 word 按字节存入 bn
        bn->bignum[offset] = (word >> (offset * 8)) & 0xff;

    return 0;
}

unsigned long
_libssh2_wincng_bignum_bits(const _libssh2_bn *bn)
{
    unsigned char number;
    unsigned long offset, length, bits;

    if(!bn || !bn->bignum || !bn->length)  // 检查 bn 是否有效
        return 0;

    offset = 0;
    length = bn->length - 1;
    while(!bn->bignum[offset] && offset < length)  // 计算 bn 的二进制位数
        offset++;

    bits = (length - offset) * 8;
    number = bn->bignum[offset];
    while(number >>= 1)
        bits++;
    bits++;

    return bits;
}

void
_libssh2_wincng_bignum_from_bin(_libssh2_bn *bn, unsigned long len,
                                const unsigned char *bin)
{
    unsigned char *bignum;
    unsigned long offset, length, bits;

    if(!bn || !bin || !len)  // 检查参数是否有效
        return;

    if(_libssh2_wincng_bignum_resize(bn, len))  // 调整 bn 的大小
        return;

    memcpy(bn->bignum, bin, len);  // 将二进制数据复制到 bn

    bits = _libssh2_wincng_bignum_bits(bn);  // 计算 bn 的二进制位数
    length = (unsigned long) (ceil(((double)bits) / 8.0) *  // 计算所需的字节数
                              sizeof(unsigned char));

    offset = bn->length - length;
    if(offset > 0) {
        memmove(bn->bignum, bn->bignum + offset, length);  // 移动数据到起始位置

#ifdef LIBSSH2_CLEAR_MEMORY
        SecureZeroMemory(bn->bignum + length, offset);  // 清空多余的内存
#endif

        bignum = realloc(bn->bignum, length);  // 重新分配内存
        if(bignum) {
            bn->bignum = bignum;
            bn->length = length;
        }
    }
}

void
_libssh2_wincng_bignum_to_bin(const _libssh2_bn *bn, unsigned char *bin)
{
    # 检查指针变量 bin、bn 和 bn->bignum 是否都存在且 bn->length 大于 0
    if(bin && bn && bn->bignum && bn->length > 0) {
        # 将 bn->bignum 指向的内存块的数据复制到 bin 指向的内存块中
        memcpy(bin, bn->bignum, bn->length);
    }
/* 释放大数对象的内存空间 */
void
_libssh2_wincng_bignum_free(_libssh2_bn *bn)
{
    if(bn) {
        if(bn->bignum) {
            _libssh2_wincng_safe_free(bn->bignum, bn->length);
            bn->bignum = NULL;
        }
        bn->length = 0;
        _libssh2_wincng_safe_free(bn, sizeof(_libssh2_bn));
    }
}

/*******************************************************************/
/*
 * Windows CNG backend: Diffie-Hellman support.
 */

/* 初始化 Diffie-Hellman 上下文 */
void
_libssh2_dh_init(_libssh2_dh_ctx *dhctx)
{
    /* 从客户端获取随机数 */
    dhctx->bn = NULL;
    dhctx->dh_handle = NULL;
    dhctx->dh_params = NULL;
}

/* 销毁 Diffie-Hellman 上下文 */
void
_libssh2_dh_dtor(_libssh2_dh_ctx *dhctx)
{
    if(dhctx->dh_handle) {
        BCryptDestroyKey(dhctx->dh_handle);
        dhctx->dh_handle = NULL;
    }
    if(dhctx->dh_params) {
        /* 由于公共 dh_params 以明文形式共享，因此此处不需要安全地清零它们 */
        free(dhctx->dh_params);
        dhctx->dh_params = NULL;
    }
    if(dhctx->bn) {
        _libssh2_wincng_bignum_free(dhctx->bn);
        dhctx->bn = NULL;
    }
}

/* 生成 Diffie-Hellman 密钥对 */
int
_libssh2_dh_key_pair(_libssh2_dh_ctx *dhctx, _libssh2_bn *public,
                     _libssh2_bn *g, _libssh2_bn *p, int group_order)
{
    const int hasAlgDHwithKDF = _libssh2_wincng.hasAlgDHwithKDF;

    /* 生成 x 和 e */
    dhctx->bn = _libssh2_wincng_bignum_init();
    if(!dhctx->bn)
        return -1;
    if(_libssh2_wincng_bignum_rand(dhctx->bn, group_order * 8 - 1, 0, -1))
        return -1;
    if(_libssh2_wincng_bignum_mod_exp(public, g, dhctx->bn, p))
        return -1;

    return 0;
}
/* 从先前创建的上下文`*dhctx'、来自对方的公钥`f'和与上下文创建时相同的素数`p'计算Diffie-Hellman密钥。结果存储在`secret'中。成功返回0，否则返回-1。 */
int
_libssh2_dh_secret(_libssh2_dh_ctx *dhctx, _libssh2_bn *secret,
                   _libssh2_bn *f, _libssh2_bn *p)
{
out:
        if(peer_public) {
            BCryptDestroyKey(peer_public);  // 如果peer_public存在，则销毁它
        }
        if(agreement) {
            BCryptDestroySecret(agreement);  // 如果agreement存在，则销毁它
        }
        if(status == STATUS_NOT_SUPPORTED &&
           _libssh2_wincng.hasAlgDHwithKDF == -1) {
            goto fb; /* 回退到基于RSA的实现 */
        }
        return BCRYPT_SUCCESS(status) ? 0 : -1;  // 如果status为BCRYPT_SUCCESS，则返回0，否则返回-1
    }

fb:
    /* 计算共享密钥 */
    return _libssh2_wincng_bignum_mod_exp(secret, f, dhctx->bn, p);  // 调用_libssh2_wincng_bignum_mod_exp函数计算共享密钥
}

#endif /* LIBSSH2_WINCNG */
```