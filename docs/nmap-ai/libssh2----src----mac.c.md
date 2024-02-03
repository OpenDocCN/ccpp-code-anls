# `nmap\libssh2\src\mac.c`

```cpp
/*
 * 版权声明和许可条件
 */
/* Copyright (c) 2004-2007, Sara Golemon <sarag@libssh2.org>
 * All rights reserved.
 *
 * Redistribution and use in source and binary forms,
 * with or without modification, are permitted provided
 * that the following conditions are met:
 *
 *   Redistributions of source code must retain the above
 *   copyright notice, this list of conditions and the
 *   following disclaimer.
 *
 *   Redistributions in binary form must reproduce the above
 *   copyright notice, this list of conditions and the following
 *   disclaimer in the documentation and/or other materials
 *   provided with the distribution.
 *
 *   Neither the name of the copyright holder nor the names
 *   of any other contributors may be used to endorse or
 *   promote products derived from this software without
 *   specific prior written permission.
 *
 * THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND
 * CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES,
 * INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES
 * OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
 * ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR
 * CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
 * SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING,
 * BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
 * SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
 * INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY,
 * WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
 * NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE
 * USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY
 * OF SUCH DAMAGE.
 */

#include "libssh2_priv.h"
#include "mac.h"

#ifdef LIBSSH2_MAC_NONE
/* mac_none_MAC
 * Minimalist MAC: No MAC
 */
static int
mac_none_MAC(LIBSSH2_SESSION * session, unsigned char *buf,
             uint32_t seqno, const unsigned char *packet,
             uint32_t packet_len, const unsigned char *addtl,
             uint32_t addtl_len, void **abstract)
{
    // 在这里实现了一个名为mac_none_MAC的函数，用于处理没有MAC的情况
    # 返回整数值0
        return 0;
# 定义一个不使用 MAC 的方法，名称为 "none"
static LIBSSH2_MAC_METHOD mac_method_none = {
    "none",  # MAC 方法名称
    0,        # MAC 密钥长度
    0,        # MAC 哈希值长度
    NULL,     # 初始化方法
    mac_none_MAC,  # 计算 MAC 哈希值的方法
    NULL      # 清理方法
};
#endif /* LIBSSH2_MAC_NONE */

/* mac_method_common_init
 * 初始化简单的 MAC 方法
 */
static int
mac_method_common_init(LIBSSH2_SESSION * session, unsigned char *key,
                       int *free_key, void **abstract)
{
    *abstract = key;  # 将密钥赋值给抽象指针
    *free_key = 0;     # 设置是否需要释放密钥的标志
    (void) session;    # 忽略会话参数

    return 0;  # 返回初始化成功
}

/* mac_method_common_dtor
 * 清理简单的 MAC 方法
 */
static int
mac_method_common_dtor(LIBSSH2_SESSION * session, void **abstract)
{
    if(*abstract) {
        LIBSSH2_FREE(session, *abstract);  # 释放抽象指针指向的内存
    }
    *abstract = NULL;  # 将抽象指针置为空

    return 0;  # 返回清理成功
}

#if LIBSSH2_HMAC_SHA512
/* mac_method_hmac_sha512_hash
 * 使用完整的 sha512 值计算哈希
 */
static int
mac_method_hmac_sha2_512_hash(LIBSSH2_SESSION * session,
                          unsigned char *buf, uint32_t seqno,
                          const unsigned char *packet,
                          uint32_t packet_len,
                          const unsigned char *addtl,
                          uint32_t addtl_len, void **abstract)
{
    libssh2_hmac_ctx ctx;  # 定义 HMAC 上下文
    unsigned char seqno_buf[4];  # 用于存储序列号的缓冲区
    (void) session;  # 忽略会话参数

    _libssh2_htonu32(seqno_buf, seqno);  # 将序列号转换为网络字节顺序

    libssh2_hmac_ctx_init(ctx);  # 初始化 HMAC 上下文
    libssh2_hmac_sha512_init(&ctx, *abstract, 64);  # 使用 sha512 初始化 HMAC 上下文
    libssh2_hmac_update(ctx, seqno_buf, 4);  # 更新 HMAC 上下文的数据
    libssh2_hmac_update(ctx, packet, packet_len);  # 更新 HMAC 上下文的数据
    if(addtl && addtl_len) {
        libssh2_hmac_update(ctx, addtl, addtl_len);  # 如果有额外数据，则更新 HMAC 上下文的数据
    }
    libssh2_hmac_final(ctx, buf);  # 计算最终的 HMAC 值
    libssh2_hmac_cleanup(&ctx);  # 清理 HMAC 上下文

    return 0;  # 返回计算成功
}

static const LIBSSH2_MAC_METHOD mac_method_hmac_sha2_512 = {
    "hmac-sha2-512",  # MAC 方法名称
    64,               # MAC 密钥长度
    64,               # MAC 哈希值长度
    mac_method_common_init,  # 初始化方法
    mac_method_hmac_sha2_512_hash,  # 计算 MAC 哈希值的方法
    mac_method_common_dtor,  # 清理方法
};
#endif

#if LIBSSH2_HMAC_SHA256
/* mac_method_hmac_sha256_hash
 * 使用完整的 sha256 值计算哈希
 */
static int
# 计算使用 HMAC-SHA2-256 哈希方法的 MAC（消息认证码）
def mac_method_hmac_sha2_256_hash(LIBSSH2_SESSION * session,
                          unsigned char *buf, uint32_t seqno,
                          const unsigned char *packet,
                          uint32_t packet_len,
                          const unsigned char *addtl,
                          uint32_t addtl_len, void **abstract)
{
    libssh2_hmac_ctx ctx;  # 初始化 HMAC 上下文
    unsigned char seqno_buf[4];  # 创建一个长度为4的字节数组
    (void) session;  # 忽略 session 参数

    _libssh2_htonu32(seqno_buf, seqno);  # 将 seqno 转换为网络字节序并存储到 seqno_buf 中

    libssh2_hmac_ctx_init(ctx);  # 初始化 HMAC 上下文
    libssh2_hmac_sha256_init(&ctx, *abstract, 32);  # 使用 HMAC-SHA2-256 初始化 HMAC 上下文
    libssh2_hmac_update(ctx, seqno_buf, 4);  # 更新 HMAC 上下文，添加 seqno_buf 的内容
    libssh2_hmac_update(ctx, packet, packet_len);  # 更新 HMAC 上下文，添加 packet 的内容
    if(addtl && addtl_len) {  # 如果 addtl 和 addtl_len 都存在
        libssh2_hmac_update(ctx, addtl, addtl_len);  # 更新 HMAC 上下文，添加 addtl 的内容
    }
    libssh2_hmac_final(ctx, buf);  # 完成 HMAC 计算，将结果存储到 buf 中
    libssh2_hmac_cleanup(&ctx);  # 清理 HMAC 上下文

    return 0;  # 返回 0
}

# 定义使用 HMAC-SHA2-256 哈希方法的 MAC 方法
static const LIBSSH2_MAC_METHOD mac_method_hmac_sha2_256 = {
    "hmac-sha2-256",  # MAC 方法的名称
    32,  # 哈希输出的长度
    32,  # HMAC 上下文的长度
    mac_method_common_init,  # 初始化 MAC 方法的函数
    mac_method_hmac_sha2_256_hash,  # 计算哈希的函数
    mac_method_common_dtor,  # 清理 MAC 方法的函数
};
#endif

# 计算使用 HMAC-SHA1 哈希方法的 MAC
static int
mac_method_hmac_sha1_hash(LIBSSH2_SESSION * session,
                          unsigned char *buf, uint32_t seqno,
                          const unsigned char *packet,
                          uint32_t packet_len,
                          const unsigned char *addtl,
                          uint32_t addtl_len, void **abstract)
{
    libssh2_hmac_ctx ctx;  # 初始化 HMAC 上下文
    unsigned char seqno_buf[4];  # 创建一个长度为4的字节数组
    (void) session;  # 忽略 session 参数

    _libssh2_htonu32(seqno_buf, seqno);  # 将 seqno 转换为网络字节序并存储到 seqno_buf 中

    libssh2_hmac_ctx_init(ctx);  # 初始化 HMAC 上下文
    libssh2_hmac_sha1_init(&ctx, *abstract, 20);  # 使用 HMAC-SHA1 初始化 HMAC 上下文
    libssh2_hmac_update(ctx, seqno_buf, 4);  # 更新 HMAC 上下文，添加 seqno_buf 的内容
    libssh2_hmac_update(ctx, packet, packet_len);  # 更新 HMAC 上下文，添加 packet 的内容
    if(addtl && addtl_len) {  # 如果 addtl 和 addtl_len 都存在
        libssh2_hmac_update(ctx, addtl, addtl_len);  # 更新 HMAC 上下文，添加 addtl 的内容
    }
    libssh2_hmac_final(ctx, buf);  # 完成 HMAC 计算，将结果存储到 buf 中
    libssh2_hmac_cleanup(&ctx);  # 清理 HMAC 上下文

    return 0;  # 返回 0
}

# 定义使用 HMAC-SHA1 哈希方法的 MAC 方法
static const LIBSSH2_MAC_METHOD mac_method_hmac_sha1 = {
    "hmac-sha1",  # MAC 方法的名称
    20,  # 哈希输出的长度
    20,  # HMAC 上下文的长度
    # 定义了三个函数名，分别是mac_method_common_init, mac_method_hmac_sha1_hash, mac_method_common_dtor
/* mac_method_hmac_sha1_96_hash
 * 使用 SHA1 值的前 96 位计算哈希
 */
static int
mac_method_hmac_sha1_96_hash(LIBSSH2_SESSION * session,
                             unsigned char *buf, uint32_t seqno,
                             const unsigned char *packet,
                             uint32_t packet_len,
                             const unsigned char *addtl,
                             uint32_t addtl_len, void **abstract)
{
    unsigned char temp[SHA_DIGEST_LENGTH];

    // 使用通用的 HMAC-SHA1 哈希方法计算哈希值
    mac_method_hmac_sha1_hash(session, temp, seqno, packet, packet_len,
                              addtl, addtl_len, abstract);
    // 将计算得到的哈希值的前 96 位复制到 buf 中
    memcpy(buf, (char *) temp, 96 / 8);

    return 0;
}

// 定义 HMAC-SHA1-96 的 MAC 方法
static const LIBSSH2_MAC_METHOD mac_method_hmac_sha1_96 = {
    "hmac-sha1-96",
    12, // key_length
    20, // hash_length
    mac_method_common_init, // init
    mac_method_hmac_sha1_96_hash, // hash
    mac_method_common_dtor, // dtor
};

#if LIBSSH2_MD5
/* mac_method_hmac_md5_hash
 * 使用完整的 MD5 值计算哈希
 */
static int
mac_method_hmac_md5_hash(LIBSSH2_SESSION * session, unsigned char *buf,
                         uint32_t seqno,
                         const unsigned char *packet,
                         uint32_t packet_len,
                         const unsigned char *addtl,
                         uint32_t addtl_len, void **abstract)
{
    libssh2_hmac_ctx ctx;
    unsigned char seqno_buf[4];
    (void) session;

    _libssh2_htonu32(seqno_buf, seqno);

    // 初始化 HMAC 上下文
    libssh2_hmac_ctx_init(ctx);
    // 使用 MD5 初始化 HMAC 上下文
    libssh2_hmac_md5_init(&ctx, *abstract, 16);
    // 更新 HMAC 上下文的数据
    libssh2_hmac_update(ctx, seqno_buf, 4);
    libssh2_hmac_update(ctx, packet, packet_len);
    if(addtl && addtl_len) {
        libssh2_hmac_update(ctx, addtl, addtl_len);
    }
    // 完成 HMAC 计算
    libssh2_hmac_final(ctx, buf);
    // 清理 HMAC 上下文
    libssh2_hmac_cleanup(&ctx);

    return 0;
}

// 定义 HMAC-MD5 的 MAC 方法
static const LIBSSH2_MAC_METHOD mac_method_hmac_md5 = {
    "hmac-md5",
    16, // key_length
    16, // hash_length
    mac_method_common_init, // init
    mac_method_hmac_md5_hash, // hash
    mac_method_common_dtor, // dtor
};
/* mac_method_hmac_md5_96_hash
 * 使用 MD5 值的前 96 位计算哈希值
 */
static int
mac_method_hmac_md5_96_hash(LIBSSH2_SESSION * session,
                            unsigned char *buf, uint32_t seqno,
                            const unsigned char *packet,
                            uint32_t packet_len,
                            const unsigned char *addtl,
                            uint32_t addtl_len, void **abstract)
{
    unsigned char temp[MD5_DIGEST_LENGTH];
    // 调用 mac_method_hmac_md5_hash 方法计算哈希值
    mac_method_hmac_md5_hash(session, temp, seqno, packet, packet_len,
                             addtl, addtl_len, abstract);
    // 将计算得到的哈希值的前 96 位复制到 buf 中
    memcpy(buf, (char *) temp, 96 / 8);
    return 0;
}

// 定义了使用 hmac-md5-96 算法的 MAC 方法
static const LIBSSH2_MAC_METHOD mac_method_hmac_md5_96 = {
    "hmac-md5-96",
    12,
    16,
    mac_method_common_init,
    mac_method_hmac_md5_96_hash,
    mac_method_common_dtor,
};
#endif /* LIBSSH2_MD5 */

#if LIBSSH2_HMAC_RIPEMD
/* mac_method_hmac_ripemd160_hash
 * 使用 ripemd160 值计算哈希值
 */
static int
mac_method_hmac_ripemd160_hash(LIBSSH2_SESSION * session,
                               unsigned char *buf, uint32_t seqno,
                               const unsigned char *packet,
                               uint32_t packet_len,
                               const unsigned char *addtl,
                               uint32_t addtl_len,
                               void **abstract)
{
    libssh2_hmac_ctx ctx;
    unsigned char seqno_buf[4];
    (void) session;

    _libssh2_htonu32(seqno_buf, seqno);

    // 初始化 HMAC 上下文
    libssh2_hmac_ctx_init(ctx);
    // 使用 ripemd160 算法初始化 HMAC 上下文
    libssh2_hmac_ripemd160_init(&ctx, *abstract, 20);
    // 更新 HMAC 上下文的数据
    libssh2_hmac_update(ctx, seqno_buf, 4);
    libssh2_hmac_update(ctx, packet, packet_len);
    if(addtl && addtl_len) {
        libssh2_hmac_update(ctx, addtl, addtl_len);
    }
    // 完成 HMAC 计算
    libssh2_hmac_final(ctx, buf);
    // 清理 HMAC 上下文
    libssh2_hmac_cleanup(&ctx);

    return 0;
}

// 定义了使用 hmac-ripemd160 算法的 MAC 方法
static const LIBSSH2_MAC_METHOD mac_method_hmac_ripemd160 = {
    "hmac-ripemd160",
    20,
    20,
    mac_method_common_init,
    mac_method_hmac_ripemd160_hash,  # 定义了一个变量 mac_method_hmac_ripemd160_hash
    mac_method_common_dtor,  # 定义了一个变量 mac_method_common_dtor
// 定义了一个名为mac_method_hmac_ripemd160_openssh_com的静态常量，表示使用hmac-ripemd160@openssh.com方法
static const LIBSSH2_MAC_METHOD mac_method_hmac_ripemd160_openssh_com = {
    "hmac-ripemd160@openssh.com", // 方法名称
    20, // 输入块大小
    20, // 输出块大小
    mac_method_common_init, // 初始化函数
    mac_method_hmac_ripemd160_hash, // 哈希函数
    mac_method_common_dtor, // 析构函数
};
#endif /* LIBSSH2_HMAC_RIPEMD */

// 定义了一个名为mac_methods的数组，包含了各种加密方法
static const LIBSSH2_MAC_METHOD *mac_methods[] = {
#if LIBSSH2_HMAC_SHA256
    &mac_method_hmac_sha2_256, // 如果支持HMAC-SHA256，则添加该方法
#endif
#if LIBSSH2_HMAC_SHA512
    &mac_method_hmac_sha2_512, // 如果支持HMAC-SHA512，则添加该方法
#endif
    &mac_method_hmac_sha1, // 添加HMAC-SHA1方法
    &mac_method_hmac_sha1_96, // 添加HMAC-SHA1-96方法
#if LIBSSH2_MD5
    &mac_method_hmac_md5, // 如果支持MD5，则添加HMAC-MD5方法
    &mac_method_hmac_md5_96, // 如果支持MD5，则添加HMAC-MD5-96方法
#endif
#if LIBSSH2_HMAC_RIPEMD
    &mac_method_hmac_ripemd160, // 如果支持HMAC-RIPEMD160，则添加该方法
    &mac_method_hmac_ripemd160_openssh_com, // 如果支持HMAC-RIPEMD160，则添加该方法
#endif /* LIBSSH2_HMAC_RIPEMD */
#ifdef LIBSSH2_MAC_NONE
    &mac_method_none, // 如果支持无加密方法，则添加该方法
#endif /* LIBSSH2_MAC_NONE */
    NULL // 结束标志
};

// 返回加密方法数组的指针
const LIBSSH2_MAC_METHOD **
_libssh2_mac_methods(void)
{
    return mac_methods;
}
```