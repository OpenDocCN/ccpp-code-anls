# `nmap\libssh2\src\kex.c`

```
/*
 * 版权声明，版权所有
 * 作者：Sara Golemon <sarag@libssh2.org> (2004-2007)
 * 作者：Daniel Stenberg <daniel@haxx.se> (2010-2019)
 * 
 * 在源代码和二进制形式下的重新分发和使用均被允许，无论是否经过修改，但需要满足以下条件：
 *   - 源代码的重新分发必须保留上述版权声明、条件列表和以下免责声明
 *   - 二进制形式的重新分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 *   - 未经特定事先书面许可，不得使用版权所有者的名称或其他贡献者的名称来认可或推广基于此软件的产品
 * 
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的适用性担保。
 * 在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）产生的任何损害，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责，即使已被告知可能发生此类损害的可能性。
 */

#include "libssh2_priv.h"

#include "transport.h"
#include "comp.h"
#include "mac.h"

#include <assert.h>

/* 如果未定义 SHA1_DIGEST_LENGTH，则使用 SHA_DIGEST_LENGTH */
#ifndef SHA1_DIGEST_LENGTH
#define SHA1_DIGEST_LENGTH SHA_DIGEST_LENGTH
#endif

/* TODO: 将此代码改为内联，并处理分配失败的情况 */
/* 定义一个宏，用于在 kex_method_diffie_hellman_group1_sha1_key_exchange 中调用
   该宏用于根据不同的椭圆曲线类型，调用不同长度的哈希函数 */

#define LIBSSH2_KEX_METHOD_EC_SHA_VALUE_HASH(value, reqlen, version)        \
    {                                                                       \
        if(type == LIBSSH2_EC_CURVE_NISTP256) {                             \
            LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(256, value, reqlen, version); \
        }                                                                   \
        else if(type == LIBSSH2_EC_CURVE_NISTP384) {                        \
            LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(384, value, reqlen, version); \
        }                                                                   \
        else if(type == LIBSSH2_EC_CURVE_NISTP521) {                        \
            LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(512, value, reqlen, version); \
        }                                                                   \
    }                                                                       \


/* 定义一个宏，用于根据不同的哈希类型，创建相应长度的哈希上下文，并分配内存空间 */
#define LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(digest_type, value,               \
                                          reqlen, version)                  \
{                                                                           \
    libssh2_sha##digest_type##_ctx hash;                                    \
    unsigned long len = 0;                                                  \
    if(!(value)) {                                                          \
        value = LIBSSH2_ALLOC(session,                                      \
                              reqlen + SHA##digest_type##_DIGEST_LENGTH);   \
    }                                                                       \
    # 如果 value 存在，则执行循环，直到长度达到所需长度
    if(value)                                                               \
        while(len < (unsigned long)reqlen) {                                \
            # 初始化指定类型的 SHA 哈希对象
            libssh2_sha##digest_type##_init(&hash);                         \
            # 更新哈希对象的数据，包括交换状态的 k_value 和 h_sig_comp
            libssh2_sha##digest_type##_update(hash,                         \
                                              exchange_state->k_value,      \
                                              exchange_state->k_value_len); \
            libssh2_sha##digest_type##_update(hash,                         \
                                              exchange_state->h_sig_comp,   \
                                         SHA##digest_type##_DIGEST_LENGTH); \
            # 如果长度大于 0，则更新哈希对象的数据为 value 的前 len 个字节
            if(len > 0) {                                                   \
                libssh2_sha##digest_type##_update(hash, value, len);        \
            }                                                               \
            # 否则，更新哈希对象的数据为 version 和 session_id
            else {                                                          \
                libssh2_sha##digest_type##_update(hash, (version), 1);      \
                libssh2_sha##digest_type##_update(hash, session->session_id,\
                                                  session->session_id_len); \
            }                                                               \
            # 完成哈希计算，将结果存储到 value 的 len 位置
            libssh2_sha##digest_type##_final(hash, (value) + len);          \
            # 更新长度
            len += SHA##digest_type##_DIGEST_LENGTH;                        \
        }                                                                   \
/*!
 * @note The following are wrapper functions used by diffie_hellman_sha_algo().
 * TODO: Switch backend SHA macros to functions to allow function pointers
 * @discussion Ideally these would be function pointers but the backend macros
 * don't allow it so we have to wrap them up in helper functions
 */

// 初始化 SHA 算法上下文
static void _libssh2_sha_algo_ctx_init(int sha_algo, void *ctx)
{
    // 根据不同的 SHA 算法类型进行初始化
    if(sha_algo == 512) {
        libssh2_sha512_init((libssh2_sha512_ctx*)ctx);
    }
    else if(sha_algo == 384) {
        libssh2_sha384_init((libssh2_sha384_ctx*)ctx);
    }
    else if(sha_algo == 256) {
        libssh2_sha256_init((libssh2_sha256_ctx*)ctx);
    }
    else if(sha_algo == 1) {
        libssh2_sha1_init((libssh2_sha1_ctx*)ctx);
    }
    else {
        assert(0);
    }
}

// 更新 SHA 算法上下文
static void _libssh2_sha_algo_ctx_update(int sha_algo, void *ctx,
                                         void *data, size_t len)
{
    // 根据不同的 SHA 算法类型进行更新
    if(sha_algo == 512) {
        libssh2_sha512_ctx *_ctx = (libssh2_sha512_ctx*)ctx;
        libssh2_sha512_update(*_ctx, data, len);
    }
    else if(sha_algo == 384) {
        libssh2_sha384_ctx *_ctx = (libssh2_sha384_ctx*)ctx;
        libssh2_sha384_update(*_ctx, data, len);
    }
    else if(sha_algo == 256) {
        libssh2_sha256_ctx *_ctx = (libssh2_sha256_ctx*)ctx;
        libssh2_sha256_update(*_ctx, data, len);
    }
    else if(sha_algo == 1) {
        libssh2_sha1_ctx *_ctx = (libssh2_sha1_ctx*)ctx;
        libssh2_sha1_update(*_ctx, data, len);
    }
    else {
        #if LIBSSH2DEBUG
        assert(0);
        #endif
    }
}

// 完成 SHA 算法计算，获取哈希值
static void _libssh2_sha_algo_ctx_final(int sha_algo, void *ctx,
                                        void *hash)
{
    // 根据不同的 SHA 算法类型进行最终计算
    if(sha_algo == 512) {
        libssh2_sha512_ctx *_ctx = (libssh2_sha512_ctx*)ctx;
        libssh2_sha512_final(*_ctx, hash);
    }
    else if(sha_algo == 384) {
        libssh2_sha384_ctx *_ctx = (libssh2_sha384_ctx*)ctx;
        libssh2_sha384_final(*_ctx, hash);
    }
    // 其他情况省略
}
    # 如果哈希算法为256位
    else if(sha_algo == 256) {
        # 将上下文转换为256位SHA上下文
        libssh2_sha256_ctx *_ctx = (libssh2_sha256_ctx*)ctx;
        # 完成256位SHA哈希计算
        libssh2_sha256_final(*_ctx, hash);
    }
    # 如果哈希算法为1
    else if(sha_algo == 1) {
        # 将上下文转换为1位SHA上下文
        libssh2_sha1_ctx *_ctx = (libssh2_sha1_ctx*)ctx;
        # 完成1位SHA哈希计算
        libssh2_sha1_final(*_ctx, hash);
    }
    # 如果以上条件都不满足
    else {
#if LIBSSH2DEBUG
        // 如果定义了 LIBSSH2DEBUG，则断言失败
        assert(0);
#endif
    }
}

static void _libssh2_sha_algo_value_hash(int sha_algo,
                                      LIBSSH2_SESSION *session,
                                      kmdhgGPshakex_state_t *exchange_state,
                                      unsigned char **data, size_t data_len,
                                      const unsigned char *version)
{
    // 根据不同的 SHA 算法值进行哈希计算
    if(sha_algo == 512) {
        LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(512, *data, data_len, version);
    }
    else if(sha_algo == 384) {
        LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(384, *data, data_len, version);
    }
    else if(sha_algo == 256) {
        LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(256, *data, data_len, version);
    }
    else if(sha_algo == 1) {
        LIBSSH2_KEX_METHOD_SHA_VALUE_HASH(1, *data, data_len, version);
    }
    else {
#if LIBSSH2DEBUG
        // 如果定义了 LIBSSH2DEBUG，则断言失败
        assert(0);
#endif
    }
}


/*!
 * @function diffie_hellman_sha_algo
 * @abstract Diffie Hellman Key Exchange, Group Agnostic,
 * SHA Algorithm Agnostic
 * @result 0 on success, error code on failure
 */
static int diffie_hellman_sha_algo(LIBSSH2_SESSION *session,
                                   _libssh2_bn *g,
                                   _libssh2_bn *p,
                                   int group_order,
                                   int sha_algo_value,
                                   void *exchange_hash_ctx,
                                   unsigned char packet_type_init,
                                   unsigned char packet_type_reply,
                                   unsigned char *midhash,
                                   unsigned long midhash_len,
                                   kmdhgGPshakex_state_t *exchange_state)
{
    int ret = 0;
    int rc;

    int digest_len = 0;

    // 根据 SHA 算法值确定摘要长度
    if(sha_algo_value == 512)
        digest_len = SHA512_DIGEST_LENGTH;
    else if(sha_algo_value == 384)
        digest_len = SHA384_DIGEST_LENGTH;
    # 如果哈希算法值为256，则设置摘要长度为SHA256摘要长度
    else if(sha_algo_value == 256)
        digest_len = SHA256_DIGEST_LENGTH;
    # 如果哈希算法值为1，则设置摘要长度为SHA1摘要长度
    else if(sha_algo_value == 1)
        digest_len = SHA1_DIGEST_LENGTH;
    # 如果哈希算法值不是256也不是1，则返回未实现的错误信息
    else {
        ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                            "sha algo value is unimplemented");
        goto clean_exit;
    }

    }

    # 如果交换状态为libssh2_NB_state_created
    if(exchange_state->state == libssh2_NB_state_created) {
        # 发送KEX初始化消息
        rc = _libssh2_transport_send(session, exchange_state->e_packet,
                                     exchange_state->e_packet_len,
                                     NULL, 0);
        # 如果返回值为LIBSSH2_ERROR_EAGAIN，则表示需要再次调用该函数
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 如果返回值不为0，则返回无法发送KEX初始化消息的错误信息
        else if(rc) {
            ret = _libssh2_error(session, rc,
                                 "Unable to send KEX init message");
            goto clean_exit;
        }
        # 设置交换状态为libssh2_NB_state_sent
        exchange_state->state = libssh2_NB_state_sent;
    }
    # 如果交换状态为已发送
    if(exchange_state->state == libssh2_NB_state_sent) {
        # 如果会话中存在乐观的密钥交换初始化
        if(session->burn_optimistic_kexinit) {
            # 服务器发送的第一个密钥交换数据包会被忽略
            # 因为这个猜测最初是错误的，所以我们需要默默地忽略它
            int burn_type;

            # 输出调试信息，等待错误猜测的密钥交换数据包
            _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                           "Waiting for badly guessed KEX packet "
                           "(to be ignored)");
            # 烧毁错误猜测的密钥交换数据包
            burn_type =
                _libssh2_packet_burn(session, &exchange_state->burn_state);
            # 如果返回值为需要再次尝试，则直接返回
            if(burn_type == LIBSSH2_ERROR_EAGAIN) {
                return burn_type;
            }
            # 如果返回值小于等于0，表示接收数据包失败
            else if(burn_type <= 0) {
                # 设置返回值为接收数据包失败的返回值
                ret = burn_type;
                # 跳转到清理退出的位置
                goto clean_exit;
            }
            # 将会话中的乐观密钥交换初始化标志位设为0
            session->burn_optimistic_kexinit = 0;

            # 输出调试信息，显示烧毁的数据包类型
            _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                           "Burnt packet of type: %02x",
                           (unsigned int) burn_type);
        }

        # 将交换状态设置为已发送1
        exchange_state->state = libssh2_NB_state_sent1;
    }
    # 如果交换状态为已发送第一个数据包
    if(exchange_state->state == libssh2_NB_state_sent1) {
        # 等待密钥交换回复
        struct string_buf buf;  # 定义一个字符串缓冲区结构体
        size_t host_key_len;  # 定义主机密钥长度变量

        # 要求接收密钥交换回复数据包
        rc = _libssh2_packet_require(session, packet_type_reply,
                                     &exchange_state->s_packet,
                                     &exchange_state->s_packet_len, 0, NULL,
                                     0, &exchange_state->req_state);
        # 如果返回值为 LIBSSH2_ERROR_EAGAIN，表示需要再次调用该函数
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 如果返回值不为 0，表示出现错误
        if(rc) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_TIMEOUT,
                                 "Timed out waiting for KEX reply");
            goto clean_exit;
        }

        # 解析 KEXDH_REPLY 数据包
        if(exchange_state->s_packet_len < 5) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                 "Unexpected packet length");
            goto clean_exit;
        }

        # 将数据包内容赋值给字符串缓冲区
        buf.data = exchange_state->s_packet;
        buf.len = exchange_state->s_packet_len;
        buf.dataptr = buf.data;
        buf.dataptr++;  # 指针向前移动一个位置，跳过类型字段

        # 如果会话中已存在服务器主机密钥，则释放其内存
        if(session->server_hostkey)
            LIBSSH2_FREE(session, session->server_hostkey);

        # 复制字符串缓冲区中的主机密钥到会话中
        if(_libssh2_copy_string(session, &buf, &(session->server_hostkey),
                                &host_key_len)) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                 "Could not copy host key");
            goto clean_exit;
        }

        # 更新服务器主机密钥的长度
        session->server_hostkey_len = (uint32_t)host_key_len;
#if LIBSSH2_MD5
        {
            // 如果支持 MD5
            libssh2_md5_ctx fingerprint_ctx;

            // 初始化 MD5 上下文
            if(libssh2_md5_init(&fingerprint_ctx)) {
                // 更新 MD5 上下文
                libssh2_md5_update(fingerprint_ctx, session->server_hostkey,
                                   session->server_hostkey_len);
                // 完成 MD5 计算
                libssh2_md5_final(fingerprint_ctx,
                                  session->server_hostkey_md5);
                // 标记 MD5 计算结果有效
                session->server_hostkey_md5_valid = TRUE;
            }
            else {
                // 标记 MD5 计算结果无效
                session->server_hostkey_md5_valid = FALSE;
            }
        }
#ifdef LIBSSH2DEBUG
        {
            // 如果定义了 LIBSSH2DEBUG
            char fingerprint[50], *fprint = fingerprint;
            int i;
            // 生成 MD5 指纹
            for(i = 0; i < 16; i++, fprint += 3) {
                snprintf(fprint, 4, "%02x:", session->server_hostkey_md5[i]);
            }
            *(--fprint) = '\0';
            // 输出 MD5 指纹
            _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                           "Server's MD5 Fingerprint: %s", fingerprint);
        }
#endif /* LIBSSH2DEBUG */
#endif /* ! LIBSSH2_MD5 */

        {
            // 初始化 SHA1 上下文
            libssh2_sha1_ctx fingerprint_ctx;

            // 如果支持 SHA1
            if(libssh2_sha1_init(&fingerprint_ctx)) {
                // 更新 SHA1 上下文
                libssh2_sha1_update(fingerprint_ctx, session->server_hostkey,
                                    session->server_hostkey_len);
                // 完成 SHA1 计算
                libssh2_sha1_final(fingerprint_ctx,
                                   session->server_hostkey_sha1);
                // 标记 SHA1 计算结果有效
                session->server_hostkey_sha1_valid = TRUE;
            }
            else {
                // 标记 SHA1 计算结果无效
                session->server_hostkey_sha1_valid = FALSE;
            }
        }
#ifdef LIBSSH2DEBUG
        {
            // 如果定义了 LIBSSH2DEBUG
            char fingerprint[64], *fprint = fingerprint;
            int i;

            // 生成 SHA1 指纹
            for(i = 0; i < 20; i++, fprint += 3) {
                snprintf(fprint, 4, "%02x:", session->server_hostkey_sha1[i]);
            }
            *(--fprint) = '\0';
            // 输出 SHA1 指纹
            _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                           "Server's SHA1 Fingerprint: %s", fingerprint);
        }
#ifndef LIBSSH2DEBUG */
        // 如果未定义 LIBSSH2DEBUG，则执行以下代码块
        {
            // 创建 SHA256 上下文对象
            libssh2_sha256_ctx fingerprint_ctx;

            // 如果成功初始化 SHA256 上下文对象
            if(libssh2_sha256_init(&fingerprint_ctx)) {
                // 更新 SHA256 上下文对象，使用服务器主机密钥和长度
                libssh2_sha256_update(fingerprint_ctx, session->server_hostkey,
                                      session->server_hostkey_len);
                // 完成 SHA256 计算，将结果存储到会话对象的 server_hostkey_sha256 属性中
                libssh2_sha256_final(fingerprint_ctx,
                                     session->server_hostkey_sha256);
                // 设置服务器主机密钥 SHA256 值为有效
                session->server_hostkey_sha256_valid = TRUE;
            }
            else {
                // 如果初始化 SHA256 上下文对象失败，则将服务器主机密钥 SHA256 值设置为无效
                session->server_hostkey_sha256_valid = FALSE;
            }
        }
#ifdef LIBSSH2DEBUG
        // 如果定义了 LIBSSH2DEBUG，则执行以下代码块
        {
            // 创建指向字符的指针变量 base64Fingerprint，并初始化为 NULL
            char *base64Fingerprint = NULL;
            // 对服务器主机密钥 SHA256 值进行 Base64 编码，结果存储到 base64Fingerprint 中
            _libssh2_base64_encode(session,
                                   (const char *)
                                   session->server_hostkey_sha256,
                                   SHA256_DIGEST_LENGTH, &base64Fingerprint);
            // 如果 base64Fingerprint 不为 NULL
            if(base64Fingerprint != NULL) {
                // 输出服务器的 SHA256 指纹值
                _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                               "Server's SHA256 Fingerprint: %s",
                               base64Fingerprint);
                // 释放 base64Fingerprint 的内存空间
                LIBSSH2_FREE(session, base64Fingerprint);
            }
        }
#ifdef LIBSSH2_DH_GEX_NEW
            // 将 LIBSSH2_DH_GEX_MINGROUP 转换为网络字节顺序，并存储到 exchange_state->h_sig_comp 中
            _libssh2_htonu32(exchange_state->h_sig_comp,
                             LIBSSH2_DH_GEX_MINGROUP);
            // 将 LIBSSH2_DH_GEX_OPTGROUP 转换为网络字节顺序，并存储到 exchange_state->h_sig_comp + 4 中
            _libssh2_htonu32(exchange_state->h_sig_comp + 4,
                             LIBSSH2_DH_GEX_OPTGROUP);
            // 将 LIBSSH2_DH_GEX_MAXGROUP 转换为网络字节顺序，并存储到 exchange_state->h_sig_comp + 8 中
            _libssh2_htonu32(exchange_state->h_sig_comp + 8,
                             LIBSSH2_DH_GEX_MAXGROUP);
            // 更新 SHA 算法上下文对象，使用 exchange_state->h_sig_comp 中的值
            _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                         exchange_state->h_sig_comp, 12);
#else
            // 将 LIBSSH2_DH_GEX_OPTGROUP 转换为网络字节顺序，并存储到 exchange_state->h_sig_comp 中
            _libssh2_htonu32(exchange_state->h_sig_comp,
                             LIBSSH2_DH_GEX_OPTGROUP);
            // 更新 SHA 算法上下文对象，使用 exchange_state->h_sig_comp 中的值
            _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                         exchange_state->h_sig_comp, 4);
#endif
        }

        if(midhash) {
            // 如果存在中间哈希值，则更新交换哈希上下文
            _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                         midhash, midhash_len);
        }

        // 更新交换哈希上下文，使用交换状态的加密数据
        _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                     exchange_state->e_packet + 1,
                                     exchange_state->e_packet_len - 1);

        // 将交换状态的 f_value_len 转换为网络字节顺序，并更新交换哈希上下文
        _libssh2_htonu32(exchange_state->h_sig_comp,
                         exchange_state->f_value_len);
        _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                     exchange_state->h_sig_comp, 4);
        // 更新交换哈希上下文，使用交换状态的 f_value
        _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                     exchange_state->f_value,
                                     exchange_state->f_value_len);

        // 更新交换哈希上下文，使用交换状态的 k_value
        _libssh2_sha_algo_ctx_update(sha_algo_value, exchange_hash_ctx,
                                     exchange_state->k_value,
                                     exchange_state->k_value_len);

        // 完成交换哈希上下文，将结果存储到 h_sig_comp 中
        _libssh2_sha_algo_ctx_final(sha_algo_value, exchange_hash_ctx,
                                    exchange_state->h_sig_comp);

        // 验证主机密钥签名
        if(session->hostkey->
           sig_verify(session, exchange_state->h_sig,
                      exchange_state->h_sig_len, exchange_state->h_sig_comp,
                      digest_len, &session->server_hostkey_abstract)) {
            // 如果无法验证主机密钥签名，则返回错误
            ret = _libssh2_error(session, LIBSSH2_ERROR_HOSTKEY_SIGN,
                                 "Unable to verify hostkey signature");
            // 跳转到清理退出的位置
            goto clean_exit;
        }

        // 发送 NEWKEYS 消息
        _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Sending NEWKEYS message");
        exchange_state->c = SSH_MSG_NEWKEYS;

        // 更新交换状态为已发送第二步
        exchange_state->state = libssh2_NB_state_sent2;
    }
    # 如果交换状态为已发送第二步
    if(exchange_state->state == libssh2_NB_state_sent2) {
        # 发送NEWKEYS消息
        rc = _libssh2_transport_send(session, &exchange_state->c, 1, NULL, 0);
        # 如果返回值为LIBSSH2_ERROR_EAGAIN，表示需要再次调用该函数
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 如果返回值不为0，表示发送NEWKEYS消息失败
        else if(rc) {
            # 设置错误信息并跳转到清理退出的标签
            ret = _libssh2_error(session, rc, "Unable to send NEWKEYS message");
            goto clean_exit;
        }

        # 更新交换状态为已发送第三步
        exchange_state->state = libssh2_NB_state_sent3;
    }

    # 清理退出标签，释放资源
    clean_exit:
    # 销毁交换状态中的DH对象
    libssh2_dh_dtor(&exchange_state->x);
    # 释放交换状态中的e值
    _libssh2_bn_free(exchange_state->e);
    exchange_state->e = NULL;
    # 释放交换状态中的f值
    _libssh2_bn_free(exchange_state->f);
    exchange_state->f = NULL;
    # 释放交换状态中的k值
    _libssh2_bn_free(exchange_state->k);
    exchange_state->k = NULL;
    # 释放交换状态中的上下文
    _libssh2_bn_ctx_free(exchange_state->ctx);
    exchange_state->ctx = NULL;

    # 如果交换状态中存在e_packet，则释放其内存
    if(exchange_state->e_packet) {
        LIBSSH2_FREE(session, exchange_state->e_packet);
        exchange_state->e_packet = NULL;
    }

    # 如果交换状态中存在s_packet，则释放其内存
    if(exchange_state->s_packet) {
        LIBSSH2_FREE(session, exchange_state->s_packet);
        exchange_state->s_packet = NULL;
    }

    # 如果交换状态中存在k_value，则释放其内存
    if(exchange_state->k_value) {
        LIBSSH2_FREE(session, exchange_state->k_value);
        exchange_state->k_value = NULL;
    }

    # 更新交换状态为空闲状态
    exchange_state->state = libssh2_NB_state_idle;

    # 返回结果
    return ret;
/* kex_method_diffie_hellman_group1_sha1_key_exchange
 * 使用 SHA1 的 Diffie-Hellman Group1（实际上是 Group2）密钥交换
 */
static int
kex_method_diffie_hellman_group1_sha1_key_exchange(LIBSSH2_SESSION *session,
                                                   key_exchange_state_low_t
                                                   * key_state)
{
    static const unsigned char p_value[128] = {
        // Diffie-Hellman Group1 的素数 p
        0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF,
        0xC9, 0x0F, 0xDA, 0xA2, 0x21, 0x68, 0xC2, 0x34,
        0xC4, 0xC6, 0x62, 0x8B, 0x80, 0xDC, 0x1C, 0xD1,
        0x29, 0x02, 0x4E, 0x08, 0x8A, 0x67, 0xCC, 0x74,
        0x02, 0x0B, 0xBE, 0xA6, 0x3B, 0x13, 0x9B, 0x22,
        0x51, 0x4A, 0x08, 0x79, 0x8E, 0x34, 0x04, 0xDD,
        0xEF, 0x95, 0x19, 0xB3, 0xCD, 0x3A, 0x43, 0x1B,
        0x30, 0x2B, 0x0A, 0x6D, 0xF2, 0x5F, 0x14, 0x37,
        0x4F, 0xE1, 0x35, 0x6D, 0x6D, 0x51, 0xC2, 0x45,
        0xE4, 0x85, 0xB5, 0x76, 0x62, 0x5E, 0x7E, 0xC6,
        0xF4, 0x4C, 0x42, 0xE9, 0xA6, 0x37, 0xED, 0x6B,
        0x0B, 0xFF, 0x5C, 0xB6, 0xF4, 0x06, 0xB7, 0xED,
        0xEE, 0x38, 0x6B, 0xFB, 0x5A, 0x89, 0x9F, 0xA5,
        0xAE, 0x9F, 0x24, 0x11, 0x7C, 0x4B, 0x1F, 0xE6,
        0x49, 0x28, 0x66, 0x51, 0xEC, 0xE6, 0x53, 0x81,
        0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF
    };

    int ret;
    libssh2_sha1_ctx exchange_hash_ctx;

    if(key_state->state == libssh2_NB_state_idle) {
        /* g == 2 */
        // 初始化 p 和 g
        key_state->p = _libssh2_bn_init_from_bin(); /* SSH2 定义的值 (p_value) */
        key_state->g = _libssh2_bn_init();      /* SSH2 定义的值 (2) */

        /* 初始化 P 和 G */
        _libssh2_bn_set_word(key_state->g, 2);
        _libssh2_bn_from_bin(key_state->p, 128, p_value);

        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Initiating Diffie-Hellman Group1 Key Exchange");

        key_state->state = libssh2_NB_state_created;
    }
    # 使用 Diffie-Hellman 算法和 SHA 算法进行密钥交换
    ret = diffie_hellman_sha_algo(session, key_state->g, key_state->p, 128, 1,
                                  (void *)&exchange_hash_ctx,
                                  SSH_MSG_KEXDH_INIT, SSH_MSG_KEXDH_REPLY,
                                  NULL, 0, &key_state->exchange_state)
    # 如果返回值为 LIBSSH2_ERROR_EAGAIN，则表示需要再次调用该函数
    if(ret == LIBSSH2_ERROR_EAGAIN) {
        return ret
    }
    
    # 释放 key_state 中的 p 和 g
    _libssh2_bn_free(key_state->p)
    key_state->p = NULL
    _libssh2_bn_free(key_state->g)
    key_state->g = NULL
    # 将 key_state 的状态设置为 libssh2_NB_state_idle
    key_state->state = libssh2_NB_state_idle
    
    # 返回 ret
    return ret
# 定义了一个名为kex_method_diffie_hellman_group14_key_exchange的函数，用于执行Diffie-Hellman Group14密钥交换，并且包含了哈希函数回调
# 函数指针类型，用于表示哈希函数的参数和返回值
typedef int (*diffie_hellman_hash_func_t)(LIBSSH2_SESSION *,
                                          _libssh2_bn *,
                                          _libssh2_bn *,
                                          int,
                                          int,
                                          void *,
                                          unsigned char,
                                          unsigned char,
                                          unsigned char *,
                                          unsigned long,
                                          kmdhgGPshakex_state_t *);
# kex_method_diffie_hellman_group14_key_exchange函数，用于执行Diffie-Hellman Group14密钥交换
static int
kex_method_diffie_hellman_group14_key_exchange(LIBSSH2_SESSION *session,
                                               key_exchange_state_low_t
                                               * key_state,
                                               int sha_algo_value,
                                               void *exchange_hash_ctx,
                                               diffie_hellman_hash_func_t
                                               hashfunc)
{
    # 定义一个包含256个无符号字符的常量数组，用于存储数值
    static const unsigned char p_value[256] = {
        # 依次初始化数组中的256个元素，每个元素占用一个字节
        0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF,
        0xC9, 0x0F, 0xDA, 0xA2, 0x21, 0x68, 0xC2, 0x34,
        # ...（省略部分初始化元素）
        0x8A, 0xAC, 0xAA, 0x68,
        0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF
    };
    # 定义一个整型变量 ret
    int ret;
    # 如果密钥状态为闲置状态
    if(key_state->state == libssh2_NB_state_idle) {
        # 从二进制数据初始化 p 值
        key_state->p = _libssh2_bn_init_from_bin(); /* SSH2 defined value
                                                       (p_value) */
        # 初始化 g 值为 2
        key_state->g = _libssh2_bn_init();      /* SSH2 defined value (2) */

        /* g == 2 */
        /* 初始化 P 和 G */
        _libssh2_bn_set_word(key_state->g, 2);
        _libssh2_bn_from_bin(key_state->p, 256, p_value);

        # 输出调试信息
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Initiating Diffie-Hellman Group14 Key Exchange");

        # 将密钥状态设置为已创建
        key_state->state = libssh2_NB_state_created;
    }
    # 使用哈希函数计算交换数据的哈希值
    ret = hashfunc(session, key_state->g, key_state->p,
                256, sha_algo_value, exchange_hash_ctx, SSH_MSG_KEXDH_INIT,
                SSH_MSG_KEXDH_REPLY, NULL, 0, &key_state->exchange_state);
    # 如果返回值为需要再次调用，则直接返回
    if(ret == LIBSSH2_ERROR_EAGAIN) {
        return ret;
    }

    # 将密钥状态设置为闲置状态
    key_state->state = libssh2_NB_state_idle;
    # 释放 p 值的内存
    _libssh2_bn_free(key_state->p);
    key_state->p = NULL;
    # 释放 g 值的内存
    _libssh2_bn_free(key_state->g);
    key_state->g = NULL;

    # 返回结果
    return ret;
/* kex_method_diffie_hellman_group14_sha1_key_exchange
 * 使用 SHA1 的 Diffie-Hellman Group14 密钥交换
 */
static int
kex_method_diffie_hellman_group14_sha1_key_exchange(LIBSSH2_SESSION *session,
                                                    key_exchange_state_low_t
                                                    * key_state)
{
    // 创建 SHA1 上下文
    libssh2_sha1_ctx ctx;
    // 调用 kex_method_diffie_hellman_group14_key_exchange 函数，使用 SHA1 算法
    return kex_method_diffie_hellman_group14_key_exchange(session,
                                                    key_state, 1,
                                                    &ctx,
                                                    diffie_hellman_sha_algo);
}

/* kex_method_diffie_hellman_group14_sha256_key_exchange
 * 使用 SHA256 的 Diffie-Hellman Group14 密钥交换
 */
static int
kex_method_diffie_hellman_group14_sha256_key_exchange(LIBSSH2_SESSION *session,
                                                      key_exchange_state_low_t
                                                      * key_state)
{
    // 创建 SHA256 上下文
    libssh2_sha256_ctx ctx;
    // 调用 kex_method_diffie_hellman_group14_key_exchange 函数，使用 SHA256 算法
    return kex_method_diffie_hellman_group14_key_exchange(session,
                                                    key_state, 256,
                                                    &ctx,
                                                    diffie_hellman_sha_algo);
}

/* kex_method_diffie_hellman_group16_sha512_key_exchange
* 使用 SHA512 的 Diffie-Hellman Group16 密钥交换
*/
static int
kex_method_diffie_hellman_group16_sha512_key_exchange(LIBSSH2_SESSION *session,
                                                      key_exchange_state_low_t
                                                      * key_state)

{
    // 定义 512 位的 p 值
    static const unsigned char p_value[512] = {
    0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xC9, 0x0F, 0xDA, 0xA2,
    0x21, 0x68, 0xC2, 0x34, 0xC4, 0xC6, 0x62, 0x8B, 0x80, 0xDC, 0x1C, 0xD1,
    0x29, 0x02, 0x4E, 0x08, 0x8A, 0x67, 0xCC, 0x74, 0x02, 0x0B, 0xBE, 0xA6,
    # 这是一串十六进制数，可能是用作密码、密钥或者其他加密相关的数据
    0x3B, 0x13, 0x9B, 0x22, 0x51, 0x4A, 0x08, 0x79, 0x8E, 0x34, 0x04, 0xDD,
    0xEF, 0x95, 0x19, 0xB3, 0xCD, 0x3A, 0x43, 0x1B, 0x30, 0x2B, 0x0A, 0x6D,
    0xF2, 0x5F, 0x14, 0x37, 0x4F, 0xE1, 0x35, 0x6D, 0x6D, 0x51, 0xC2, 0x45,
    0xE4, 0x85, 0xB5, 0x76, 0x62, 0x5E, 0x7E, 0xC6, 0xF4, 0x4C, 0x42, 0xE9,
    0xA6, 0x37, 0xED, 0x6B, 0x0B, 0xFF, 0x5C, 0xB6, 0xF4, 0x06, 0xB7, 0xED,
    0xEE, 0x38, 0x6B, 0xFB, 0x5A, 0x89, 0x9F, 0xA5, 0xAE, 0x9F, 0x24, 0x11,
    0x7C, 0x4B, 0x1F, 0xE6, 0x49, 0x28, 0x66, 0x51, 0xEC, 0xE4, 0x5B, 0x3D,
    0xC2, 0x00, 0x7C, 0xB8, 0xA1, 0x63, 0xBF, 0x05, 0x98, 0xDA, 0x48, 0x36,
    0x1C, 0x55, 0xD3, 0x9A, 0x69, 0x16, 0x3F, 0xA8, 0xFD, 0x24, 0xCF, 0x5F,
    0x83, 0x65, 0x5D, 0x23, 0xDC, 0xA3, 0xAD, 0x96, 0x1C, 0x62, 0xF3, 0x56,
    0x20, 0x85, 0x52, 0xBB, 0x9E, 0xD5, 0x29, 0x07, 0x70, 0x96, 0x96, 0x6D,
    0x67, 0x0C, 0x35, 0x4E, 0x4A, 0xBC, 0x98, 0x04, 0xF1, 0x74, 0x6C, 0x08,
    0xCA, 0x18, 0x21, 0x7C, 0x32, 0x90, 0x5E, 0x46, 0x2E, 0x36, 0xCE, 0x3B,
    0xE3, 0x9E, 0x77, 0x2C, 0x18, 0x0E, 0x86, 0x03, 0x9B, 0x27, 0x83, 0xA2,
    0xEC, 0x07, 0xA2, 0x8F, 0xB5, 0xC5, 0x5D, 0xF0, 0x6F, 0x4C, 0x52, 0xC9,
    0xDE, 0x2B, 0xCB, 0xF6, 0x95, 0x58, 0x17, 0x18, 0x39, 0x95, 0x49, 0x7C,
    0xEA, 0x95, 0x6A, 0xE5, 0x15, 0xD2, 0x26, 0x18, 0x98, 0xFA, 0x05, 0x10,
    0x15, 0x72, 0x8E, 0x5A, 0x8A, 0xAA, 0xC4, 0x2D, 0xAD, 0x33, 0x17, 0x0D,
    0x04, 0x50, 0x7A, 0x33, 0xA8, 0x55, 0x21, 0xAB, 0xDF, 0x1C, 0xBA, 0x64,
    0xEC, 0xFB, 0x85, 0x04, 0x58, 0xDB, 0xEF, 0x0A, 0x8A, 0xEA, 0x71, 0x57,
    0x5D, 0x06, 0x0C, 0x7D, 0xB3, 0x97, 0x0F, 0x85, 0xA6, 0xE1, 0xE4, 0xC7,
    0xAB, 0xF5, 0xAE, 0x8C, 0xDB, 0x09, 0x33, 0xD7, 0x1E, 0x8C, 0x94, 0xE0,
    0x4A, 0x25, 0x61, 0x9D, 0xCE, 0xE3, 0xD2, 0x26, 0x1A, 0xD2, 0xEE, 0x6B,
    0xF1, 0x2F, 0xFA, 0x06, 0xD9, 0x8A, 0x08, 0x64, 0xD8, 0x76, 0x02, 0x73,
    0x3E, 0xC8, 0x6A, 0x64, 0x52, 0x1F, 0x2B, 0x18, 0x17, 0x7B, 0x20, 0x0C,
    0xBB, 0xE1, 0x17, 0x57, 0x7A, 0x61, 0x5D, 0x6C, 0x77, 0x09, 0x88, 0xC0,
    # 定义一个包含十六进制数的数组
    0xBA, 0xD9, 0x46, 0xE2, 0x08, 0xE2, 0x4F, 0xA0, 0x74, 0xE5, 0xAB, 0x31,
    0x43, 0xDB, 0x5B, 0xFC, 0xE0, 0xFD, 0x10, 0x8E, 0x4B, 0x82, 0xD1, 0x20,
    0xA9, 0x21, 0x08, 0x01, 0x1A, 0x72, 0x3C, 0x12, 0xA7, 0x87, 0xE6, 0xD7,
    0x88, 0x71, 0x9A, 0x10, 0xBD, 0xBA, 0x5B, 0x26, 0x99, 0xC3, 0x27, 0x18,
    0x6A, 0xF4, 0xE2, 0x3C, 0x1A, 0x94, 0x68, 0x34, 0xB6, 0x15, 0x0B, 0xDA,
    0x25, 0x83, 0xE9, 0xCA, 0x2A, 0xD4, 0x4C, 0xE8, 0xDB, 0xBB, 0xC2, 0xDB,
    0x04, 0xDE, 0x8E, 0xF9, 0x2E, 0x8E, 0xFC, 0x14, 0x1F, 0xBE, 0xCA, 0xA6,
    0x28, 0x7C, 0x59, 0x47, 0x4E, 0x6B, 0xC0, 0x5D, 0x99, 0xB2, 0x96, 0x4F,
    0xA0, 0x90, 0xC3, 0xA2, 0x23, 0x3B, 0xA1, 0x86, 0x51, 0x5B, 0xE7, 0xED,
    0x1F, 0x61, 0x29, 0x70, 0xCE, 0xE2, 0xD7, 0xAF, 0xB8, 0x1B, 0xDD, 0x76,
    0x21, 0x70, 0x48, 0x1C, 0xD0, 0x06, 0x91, 0x27, 0xD5, 0xB0, 0x5A, 0xA9,
    0x93, 0xB4, 0xEA, 0x98, 0x8D, 0x8F, 0xDD, 0xC1, 0x86, 0xFF, 0xB7, 0xDC,
    0x90, 0xA6, 0xC0, 0x8F, 0x4D, 0xF4, 0x35, 0xC9, 0x34, 0x06, 0x31, 0x99,
    0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF
    };
    
    # 定义一个整型变量 ret
    int ret;
    # 定义一个 libssh2_sha512_ctx 结构体变量 exchange_hash_ctx
    
    # 如果 key_state 的状态为 libssh2_NB_state_idle
    if(key_state->state == libssh2_NB_state_idle) {
        # 从二进制数据初始化一个大数 p
        key_state->p = _libssh2_bn_init_from_bin(); /* SSH2 defined value
                                                       (p_value) */
        # 初始化一个大数 g
        key_state->g = _libssh2_bn_init();      /* SSH2 defined value (2) */
    
        # g == 2
        # 初始化 P 和 G
        _libssh2_bn_set_word(key_state->g, 2);
        _libssh2_bn_from_bin(key_state->p, 512, p_value);
    
        # 输出日志信息，表示正在启动 Diffie-Hellman Group16 密钥交换
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Initiating Diffie-Hellman Group16 Key Exchange");
    
        # 将 key_state 的状态设置为 libssh2_NB_state_created
        key_state->state = libssh2_NB_state_created;
    }
    # 使用 Diffie-Hellman 算法和 SHA 算法进行密钥交换
    ret = diffie_hellman_sha_algo(session, key_state->g, key_state->p, 512,
                                  512, (void *)&exchange_hash_ctx,
                                  SSH_MSG_KEXDH_INIT, SSH_MSG_KEXDH_REPLY,
                                  NULL, 0, &key_state->exchange_state)
    # 如果返回值为 LIBSSH2_ERROR_EAGAIN，则表示需要再次调用该函数
    if(ret == LIBSSH2_ERROR_EAGAIN) {
        return ret
    }
    
    # 设置 key_state 的状态为 libssh2_NB_state_idle
    key_state->state = libssh2_NB_state_idle
    # 释放 key_state 中的 p 和 g
    _libssh2_bn_free(key_state->p)
    key_state->p = NULL
    _libssh2_bn_free(key_state->g)
    key_state->g = NULL
    
    # 返回 ret
    return ret
# 定义 Diffie-Hellman Group18 Key Exchange using SHA512 的方法
static int
kex_method_diffie_hellman_group18_sha512_key_exchange(LIBSSH2_SESSION *session,
                                                      key_exchange_state_low_t
                                                      * key_state)
{
    # 定义 Diffie-Hellman Group18 Key Exchange 使用的 p 值
    static const unsigned char p_value[1024] = {
    # p 值的具体数值
    0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xC9, 0x0F, 0xDA, 0xA2,
    0x21, 0x68, 0xC2, 0x34, 0xC4, 0xC6, 0x62, 0x8B, 0x80, 0xDC, 0x1C, 0xD1,
    0x29, 0x02, 0x4E, 0x08, 0x8A, 0x67, 0xCC, 0x74, 0x02, 0x0B, 0xBE, 0xA6,
    0x3B, 0x13, 0x9B, 0x22, 0x51, 0x4A, 0x08, 0x79, 0x8E, 0x34, 0x04, 0xDD,
    0xEF, 0x95, 0x19, 0xB3, 0xCD, 0x3A, 0x43, 0x1B, 0x30, 0x2B, 0x0A, 0x6D,
    0xF2, 0x5F, 0x14, 0x37, 0x4F, 0xE1, 0x35, 0x6D, 0x6D, 0x51, 0xC2, 0x45,
    0xE4, 0x85, 0xB5, 0x76, 0x62, 0x5E, 0x7E, 0xC6, 0xF4, 0x4C, 0x42, 0xE9,
    0xA6, 0x37, 0xED, 0x6B, 0x0B, 0xFF, 0x5C, 0xB6, 0xF4, 0x06, 0xB7, 0xED,
    0xEE, 0x38, 0x6B, 0xFB, 0x5A, 0x89, 0x9F, 0xA5, 0xAE, 0x9F, 0x24, 0x11,
    0x7C, 0x4B, 0x1F, 0xE6, 0x49, 0x28, 0x66, 0x51, 0xEC, 0xE4, 0x5B, 0x3D,
    0xC2, 0x00, 0x7C, 0xB8, 0xA1, 0x63, 0xBF, 0x05, 0x98, 0xDA, 0x48, 0x36,
    0x1C, 0x55, 0xD3, 0x9A, 0x69, 0x16, 0x3F, 0xA8, 0xFD, 0x24, 0xCF, 0x5F,
    0x83, 0x65, 0x5D, 0x23, 0xDC, 0xA3, 0xAD, 0x96, 0x1C, 0x62, 0xF3, 0x56,
    0x20, 0x85, 0x52, 0xBB, 0x9E, 0xD5, 0x29, 0x07, 0x70, 0x96, 0x96, 0x6D,
    0x67, 0x0C, 0x35, 0x4E, 0x4A, 0xBC, 0x98, 0x04, 0xF1, 0x74, 0x6C, 0x08,
    0xCA, 0x18, 0x21, 0x7C, 0x32, 0x90, 0x5E, 0x46, 0x2E, 0x36, 0xCE, 0x3B,
    0xE3, 0x9E, 0x77, 0x2C, 0x18, 0x0E, 0x86, 0x03, 0x9B, 0x27, 0x83, 0xA2,
    0xEC, 0x07, 0xA2, 0x8F, 0xB5, 0xC5, 0x5D, 0xF0, 0x6F, 0x4C, 0x52, 0xC9,
    0xDE, 0x2B, 0xCB, 0xF6, 0x95, 0x58, 0x17, 0x18, 0x39, 0x95, 0x49, 0x7C,
    0xEA, 0x95, 0x6A, 0xE5, 0x15, 0xD2, 0x26, 0x18, 0x98, 0xFA, 0x05, 0x10,
    0x15, 0x72, 0x8E, 0x5A, 0x8A, 0xAA, 0xC4, 0x2D, 0xAD, 0x33, 0x17, 0x0D,
    # 这是一串十六进制数，可能是用作密码、密钥或者其他数据的编码
    0x04, 0x50, 0x7A, 0x33, 0xA8, 0x55, 0x21, 0xAB, 0xDF, 0x1C, 0xBA, 0x64,
    0xEC, 0xFB, 0x85, 0x04, 0x58, 0xDB, 0xEF, 0x0A, 0x8A, 0xEA, 0x71, 0x57,
    5D, 0x06, 0x0C, 0x7D, 0xB3, 0x97, 0x0F, 0x85, 0xA6, 0xE1, 0xE4, 0xC7,
    0xAB, 0xF5, 0xAE, 0x8C, 0xDB, 0x09, 0x33, 0xD7, 0x1E, 0x8C, 0x94, 0xE0,
    0x4A, 0x25, 0x61, 0x9D, 0xCE, 0xE3, 0xD2, 0x26, 0x1A, 0xD2, 0xEE, 0x6B,
    0xF1, 0x2F, 0xFA, 0x06, 0xD9, 0x8A, 0x08, 0x64, 0xD8, 0x76, 0x02, 0x73,
    0x3E, 0xC8, 0x6A, 0x64, 0x52, 0x1F, 0x2B, 0x18, 0x17, 0x7B, 0x20, 0x0C,
    0xBB, 0xE1, 0x17, 0x57, 0x7A, 0x61, 0x5D, 0x6C, 0x77, 0x09, 0x88, 0xC0,
    0xBA, 0xD9, 0x46, 0xE2, 0x08, 0xE2, 0x4F, 0xA0, 0x74, 0xE5, 0xAB, 0x31,
    0x43, 0xDB, 0x5B, 0xFC, 0xE0, 0xFD, 0x10, 0x8E, 0x4B, 0x82, 0xD1, 0x20,
    0xA9, 0x21, 0x08, 0x01, 0x1A, 0x72, 0x3C, 0x12, 0xA7, 0x87, 0xE6, 0xD7,
    0x88, 0x71, 0x9A, 0x10, 0xBD, 0xBA, 0x5B, 0x26, 0x99, 0xC3, 0x27, 0x18,
    0x6A, 0xF4, 0xE2, 0x3C, 0x1A, 0x94, 0x68, 0x34, 0xB6, 0x15, 0x0B, 0xDA,
    0x25, 0x83, 0xE9, 0xCA, 0x2A, 0xD4, 0x4C, 0xE8, 0xDB, 0xBB, 0xC2, 0xDB,
    0x04, 0xDE, 0x8E, 0xF9, 0x2E, 0x8E, 0xFC, 0x14, 0x1F, 0xBE, 0xCA, 0xA6,
    0x28, 0x7C, 0x59, 0x47, 0x4E, 0x6B, 0xC0, 0x5D, 0x99, 0xB2, 0x96, 0x4F,
    0xA0, 0x90, 0xC3, 0xA2, 0x23, 0x3B, 0xA1, 0x86, 0x51, 0x5B, 0xE7, 0xED,
    0x1F, 0x61, 0x29, 0x70, 0xCE, 0xE2, 0xD7, 0xAF, 0xB8, 0x1B, 0xDD, 0x76,
    0x21, 0x70, 0x48, 0x1C, 0xD0, 0x06, 0x91, 0x27, 0xD5, 0xB0, 0x5A, 0xA9,
    0x93, 0xB4, 0xEA, 0x98, 0x8D, 0x8F, 0xDD, 0xC1, 0x86, 0xFF, 0xB7, 0xDC,
    0x90, 0xA6, 0xC0, 0x8F, 0x4D, F4, 0x35, 0xC9, 0x34, 0x02, 0x84, 0x92,
    0x36, 0xC3, 0xFA, 0xB4, 0xD2, 0x7C, 0x70, 0x26, 0xC1, 0xD4, 0xDC, 0xB2,
    0x60, 0x26, 0x46, 0xDE, 0xC9, 0x75, 0x1E, 0x76, 0x3D, 0xBA, 0x37, 0xBD,
    0xF8, 0xFF, 0x94, 0x06, 0xAD, 0x9E, 0x53, 0x0E, 0xE5, 0xDB, 0x38, 0x2F,
    0x41, 0x30, 0x01, 0xAE, 0xB0, 0x6A, 0x53, 0xED, 0x90, 0x27, 0xD8, 0x31,
    0x17, 0x97, 0x27, 0xB0, 0x86, 0x5A, 0x89, 0x18, 0xDA, 0x3E, 0xDB, 0xEB,
    # 这是一串十六进制数，可能是用作密码、密钥或者其他数据的编码
    0xCF, 0x9B, 0x14, 0xED, 0x44, 0xCE, 0x6C, 0xBA, 0xCE, 0xD4, 0xBB, 0x1B,
    0xDB, 0x7F, 0x14, 0x47, 0xE6, 0xCC, 0x25, 0x4B, 0x33, 0x20, 0x51, 0x51,
    0x2B, 0xD7, 0xAF, 0x42, 0x6F, 0xB8, 0xF4, 0x01, 0x37, 0x8C, 0xD2, 0xBF,
    0x59, 0x83, 0xCA, 0x01, 0xC6, 0x4B, 0x92, 0xEC, 0xF0, 0x32, 0xEA, 0x15,
    0xD1, 0x72, 0x1D, 0x03, 0xF4, 0x82, 0xD7, 0xCE, 0x6E, 0x74, 0xFE, 0xF6,
    0xD5, 0x5E, 0x70, 0x2F, 0x46, 0x98, 0x0C, 0x82, 0xB5, 0xA8, 0x40, 0x31,
    0x90, 0x0B, 0x1C, 0x9E, 0x59, 0xE7, 0xC9, 0x7F, 0xBE, 0xC7, 0xE8, 0xF3,
    0x23, 0xA9, 0x7A, 0x7E, 0x36, 0xCC, 0x88, 0xBE, 0x0F, 0x1D, 0x45, 0xB7,
    0xFF, 0x58, 0x5A, 0xC5, 0x4B, 0xD4, 0x07, 0xB2, 0x2B, 0x41, 0x54, 0xAA,
    0xCC, 0x8F, 0x6D, 0x7E, 0xBF, 0x48, 0xE1, 0xD8, 0x14, 0xCC, 0x5E, 0xD2,
    0x0F, 0x80, 0x37, 0xE0, 0xA7, 0x97, 0x15, 0xEE, 0xF2, 0x9B, 0xE3, 0x28,
    0x06, 0xA1, 0xD5, 0x8B, 0xB7, 0xC5, 0xDA, 0x76, 0xF5, 0x50, 0xAA, 0x3D,
    0x8A, 0x1F, 0xBF, 0xF0, 0xEB, 0x19, 0xCC, 0xB1, 0xA3, 0x13, 0xD5, 0x5C,
    0xDA, 0x56, 0xC9, 0xEC, 0x2E, 0xF2, 0x96, 0x32, 0x38, 0x7F, 0xE8, 0xD7,
    0x6E, 0x3C, 0x04, 0x68, 0x04, 0x3E, 0x8F, 0x66, 0x3F, 0x48, 0x60, 0xEE,
    0x12, 0xBF, 0x2D, 0x5B, 0x0B, 0x74, 0x74, 0xD6, 0xE6, 0x94, 0xF9, 0x1E,
    0x6D, 0xBE, 0x11, 0x59, 0x74, 0xA3, 0x92, 0x6F, 0x12, 0xFE, 0xE5, 0xE4,
    0x38, 0x77, 0x7C, 0xB6, 0xA9, 0x32, 0xDF, 0x8C, 0xD8, 0xBE, 0xC4, 0xD0,
    0x73, 0xB9, 0x31, 0xBA, 0x3B, 0xC8, 0x32, 0xB6, 0x8D, 0x9D, 0xD3, 0x00,
    0x74, 0x1F, 0xA7, 0xBF, 0x8A, 0xFC, 0x47, 0xED, 0x25, 0x76, 0xF6, 0x93,
    0x6B, 0xA4, 0x24, 0x66, 0x3A, 0xAB, 0x63, 0x9C, 0x5A, 0xE4, 0xF5, 0x68,
    0x34, 0x23, 0xB4, 0x74, 0x2B, 0xF1, 0xC9, 0x78, 0x23, 0x8F, 0x16, 0xCB,
    0xE3, 0x9D, 0x65, 0x2D, 0xE3, 0xFD, 0xB8, 0xBE, 0xFC, 0x84, 0x8A, 0xD9,
    0x22, 0x22, 0x2E, 0x04, 0xA4, 0x03, 0x7C, 0x07, 0x13, 0xEB, 0x57, 0xA8,
    0x1A, 0x23, 0xF0, 0xC7, 0x34, 0x73, 0xFC, 0x64, 0x6C, 0xEA, 0x30, 0x6B,
    0x4B, 0xCB, 0xC8, 0x86, 0x2F, 0x83, 0x85, 0xDD, 0xFA, 0x9D, 0x4B, 0x7F,
    # 定义一个包含十六进制数的数组
    0xA2, 0xC0, 0x87, 0xE8, 0x79, 0x68, 0x33, 0x03, 0xED, 0x5B, 0xDD, 0x3A,
    0x06, 0x2B, 0x3C, 0xF5, 0xB3, 0xA2, 0x78, 0xA6, 0x6D, 0x2A, 0x13, 0xF8,
    0x3F, 0x44, 0xF8, 0x2D, 0xDF, 0x31, 0x0E, 0xE0, 0x74, 0xAB, 0x6A, 0x36,
    0x45, 0x97, 0xE8, 0x99, 0xA0, 0x25, 0x5D, 0xC1, 0x64, 0xF3, 0x1C, 0xC5,
    0x08, 0x46, 0x85, 0x1D, 0xF9, 0xAB, 0x48, 0x19, 0x5D, 0xED, 0x7E, 0xA1,
    0xB1, 0xD5, 0x10, 0xBD, 0x7E, 0xE7, 0x4D, 0x73, 0xFA, 0xF3, 0x6B, 0xC3,
    0x1E, 0xCF, 0xA2, 0x68, 0x35, 0x90, 0x46, 0xF4, 0xEB, 0x87, 0x9F, 0x92,
    0x40, 0x09, 0x43, 0x8B, 0x48, 0x1C, 0x6C, 0xD7, 0x88, 0x9A, 0x00, 0x2E,
    0xD5, 0xEE, 0x38, 0x2B, 0xC9, 0x19, 0x0D, 0xA6, 0xFC, 0x02, 0x6E, 0x47,
    0x95, 0x58, 0xE4, 0x47, 0x56, 0x77, 0xE9, 0xAA, 0x9E, 0x30, 0x50, 0xE2,
    0x76, 0x56, 0x94, 0xDF, 0xC8, 0x1F, 0x56, 0xE8, 0x80, 0xB9, 0x6E, 0x71,
    0x60, 0xC9, 0x80, 0xDD, 0x98, 0xED, 0xD3, 0xDF, 0xFF, 0xFF, 0xFF, 0xFF,
    0xFF, 0xFF, 0xFF, 0xFF
    };
    
    # 定义一个整型变量 ret
    int ret;
    # 定义一个 SHA512 上下文对象 exchange_hash_ctx
    libssh2_sha512_ctx exchange_hash_ctx;
    
    # 如果 key_state 的状态为 libssh2_NB_state_idle
    if(key_state->state == libssh2_NB_state_idle) {
        # 从二进制数据初始化 P 值
        key_state->p = _libssh2_bn_init_from_bin(); /* SSH2 defined value
                                                       (p_value) */
        # 初始化 G 值
        key_state->g = _libssh2_bn_init();      /* SSH2 defined value (2) */
    
        # g == 2
        # 初始化 P 和 G
        _libssh2_bn_set_word(key_state->g, 2);
        _libssh2_bn_from_bin(key_state->p, 1024, p_value);
    
        # 输出日志信息
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Initiating Diffie-Hellman Group18 Key Exchange");
    
        # 设置 key_state 的状态为 libssh2_NB_state_created
        key_state->state = libssh2_NB_state_created;
    }
    
    # 调用 diffie_hellman_sha_algo 函数进行密钥交换
    ret = diffie_hellman_sha_algo(session, key_state->g, key_state->p, 1024,
                                  512, (void *)&exchange_hash_ctx,
                                  SSH_MSG_KEXDH_INIT, SSH_MSG_KEXDH_REPLY,
                                  NULL, 0, &key_state->exchange_state);
    # 如果返回值为 LIBSSH2_ERROR_EAGAIN，则返回 ret
    if(ret == LIBSSH2_ERROR_EAGAIN) {
        return ret;
    }
    # 将 key_state 结构体中的 state 状态设置为 libssh2_NB_state_idle
    key_state->state = libssh2_NB_state_idle;
    # 释放 key_state 结构体中的 p 成员所指向的内存，并将其指针设置为 NULL
    _libssh2_bn_free(key_state->p);
    key_state->p = NULL;
    # 释放 key_state 结构体中的 g 成员所指向的内存，并将其指针设置为 NULL
    _libssh2_bn_free(key_state->g);
    key_state->g = NULL;

    # 返回 ret 变量的值
    return ret;
/* kex_method_diffie_hellman_group_exchange_sha1_key_exchange
 * Diffie-Hellman Group Exchange Key Exchange using SHA1
 * Negotiates random(ish) group for secret derivation
 */
static int
kex_method_diffie_hellman_group_exchange_sha1_key_exchange
(LIBSSH2_SESSION * session, key_exchange_state_low_t * key_state)
{
    int ret = 0;
    int rc;

    if(key_state->state == libssh2_NB_state_idle) {
        key_state->p = _libssh2_bn_init_from_bin();  // 从二进制数据初始化大数 p
        key_state->g = _libssh2_bn_init_from_bin();  // 从二进制数据初始化大数 g
        /* Ask for a P and G pair */
#ifdef LIBSSH2_DH_GEX_NEW
        key_state->request[0] = SSH_MSG_KEX_DH_GEX_REQUEST;  // 设置请求消息类型为 Diffie-Hellman Group Exchange Request
        _libssh2_htonu32(key_state->request + 1, LIBSSH2_DH_GEX_MINGROUP);  // 将最小组大小转换为网络字节序并存入请求消息
        _libssh2_htonu32(key_state->request + 5, LIBSSH2_DH_GEX_OPTGROUP);  // 将可选组大小转换为网络字节序并存入请求消息
        _libssh2_htonu32(key_state->request + 9, LIBSSH2_DH_GEX_MAXGROUP);  // 将最大组大小转换为网络字节序并存入请求消息
        key_state->request_len = 13;  // 设置请求消息长度
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Initiating Diffie-Hellman Group-Exchange "
                       "(New Method)");  // 调试信息，表示正在启动 Diffie-Hellman Group-Exchange (New Method)
#else
        key_state->request[0] = SSH_MSG_KEX_DH_GEX_REQUEST_OLD;  // 设置请求消息类型为旧版 Diffie-Hellman Group Exchange Request
        _libssh2_htonu32(key_state->request + 1, LIBSSH2_DH_GEX_OPTGROUP);  // 将可选组大小转换为网络字节序并存入请求消息
        key_state->request_len = 5;  // 设置请求消息长度
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Initiating Diffie-Hellman Group-Exchange "
                       "(Old Method)");  // 调试信息，表示正在启动 Diffie-Hellman Group-Exchange (Old Method)
#endif

        key_state->state = libssh2_NB_state_created;  // 设置状态为已创建
    }

    if(key_state->state == libssh2_NB_state_created) {
        rc = _libssh2_transport_send(session, key_state->request,
                                     key_state->request_len, NULL, 0);  // 发送请求消息
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;  // 如果返回值为 EAGAIN，表示需要再次调用该函数
        }
        else if(rc) {
            ret = _libssh2_error(session, rc,
                                 "Unable to send Group Exchange Request");  // 如果返回值不为 0，表示发送请求消息失败
            goto dh_gex_clean_exit;  // 跳转到清理退出的标签
        }

        key_state->state = libssh2_NB_state_sent;  // 设置状态为已发送
    }
    // 如果密钥状态为已发送
    if(key_state->state == libssh2_NB_state_sent) {
        // 要求接收 SSH_MSG_KEX_DH_GEX_GROUP 消息
        rc = _libssh2_packet_require(session, SSH_MSG_KEX_DH_GEX_GROUP,
                                     &key_state->data, &key_state->data_len,
                                     0, NULL, 0, &key_state->req_state);
        // 如果返回值为 LIBSSH2_ERROR_EAGAIN，表示需要再次调用该函数
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        // 如果返回值不为 0，表示出现错误
        else if(rc) {
            // 设置错误信息并跳转到清理退出的标签
            ret = _libssh2_error(session, rc,
                                 "Timeout waiting for GEX_GROUP reply");
            goto dh_gex_clean_exit;
        }

        // 更新密钥状态为已发送1
        key_state->state = libssh2_NB_state_sent1;
    }
    # 如果密钥状态为已发送第一步
    if(key_state->state == libssh2_NB_state_sent1) {
        # 定义变量 p_len 和 g_len
        size_t p_len, g_len;
        # 定义指针 p 和 g
        unsigned char *p, *g;
        # 定义结构体变量 buf
        struct string_buf buf;
        # 定义交换哈希上下文 exchange_hash_ctx
        libssh2_sha1_ctx exchange_hash_ctx;

        # 如果密钥状态中的数据长度小于9
        if(key_state->data_len < 9) {
            # 返回意外的密钥长度错误
            ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                 "Unexpected key length");
            # 跳转到清理退出标签
            goto dh_gex_clean_exit;
        }

        # 将密钥状态中的数据赋值给 buf 的 data 和 dataptr，数据长度赋值给 buf 的 len
        buf.data = key_state->data;
        buf.dataptr = buf.data;
        buf.len = key_state->data_len;

        # 将 buf 的 dataptr 指针向后移动一位，指向大数
        buf.dataptr++; /* increment to big num */

        # 从 buf 中获取大数 p 的字节数组
        if(_libssh2_get_bignum_bytes(&buf, &p, &p_len)) {
            # 返回意外的值错误
            ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                 "Unexpected value");
            # 跳转到清理退出标签
            goto dh_gex_clean_exit;
        }

        # 从 buf 中获取大数 g 的字节数组
        if(_libssh2_get_bignum_bytes(&buf, &g, &g_len)) {
            # 返回意外的值错误
            ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                 "Unexpected value");
            # 跳转到清理退出标签
            goto dh_gex_clean_exit;
        }

        # 将字节数组 p 转换为大数存储在 key_state->p 中
        _libssh2_bn_from_bin(key_state->p, p_len, p);
        # 将字节数组 g 转换为大数存储在 key_state->g 中
        _libssh2_bn_from_bin(key_state->g, g_len, g);

        # 调用 diffie_hellman_sha_algo 函数进行密钥交换
        ret = diffie_hellman_sha_algo(session, key_state->g, key_state->p,
                                      p_len, 1,
                                      (void *)&exchange_hash_ctx,
                                      SSH_MSG_KEX_DH_GEX_INIT,
                                      SSH_MSG_KEX_DH_GEX_REPLY,
                                      key_state->data + 1,
                                      key_state->data_len - 1,
                                      &key_state->exchange_state);
        # 如果返回值为需要再次调用，则直接返回
        if(ret == LIBSSH2_ERROR_EAGAIN) {
            return ret;
        }

        # 释放 key_state->data 的内存
        LIBSSH2_FREE(session, key_state->data);
    }

  # 清理退出标签，将密钥状态设置为空闲状态，释放内存并返回结果
  dh_gex_clean_exit:
    key_state->state = libssh2_NB_state_idle;
    _libssh2_bn_free(key_state->g);
    key_state->g = NULL;
    _libssh2_bn_free(key_state->p);
    key_state->p = NULL;

    return ret;
# 定义 Diffie-Hellman Group Exchange Key Exchange using SHA256 方法
# 用于协商秘密派生的随机组
static int
kex_method_diffie_hellman_group_exchange_sha256_key_exchange
(LIBSSH2_SESSION * session, key_exchange_state_low_t * key_state)
{
    int ret = 0;
    int rc;

    # 如果 key_state 的状态为 idle
    if(key_state->state == libssh2_NB_state_idle) {
        # 初始化 p 和 g
        key_state->p = _libssh2_bn_init();
        key_state->g = _libssh2_bn_init();
        # 请求 P 和 G 对
        # 如果支持新方法
#ifdef LIBSSH2_DH_GEX_NEW
        key_state->request[0] = SSH_MSG_KEX_DH_GEX_REQUEST;
        _libssh2_htonu32(key_state->request + 1, LIBSSH2_DH_GEX_MINGROUP);
        _libssh2_htonu32(key_state->request + 5, LIBSSH2_DH_GEX_OPTGROUP);
        _libssh2_htonu32(key_state->request + 9, LIBSSH2_DH_GEX_MAXGROUP);
        key_state->request_len = 13;
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Initiating Diffie-Hellman Group-Exchange "
                       "(New Method SHA256)");
# 如果不支持新方法
#else
        key_state->request[0] = SSH_MSG_KEX_DH_GEX_REQUEST_OLD;
        _libssh2_htonu32(key_state->request + 1, LIBSSH2_DH_GEX_OPTGROUP);
        key_state->request_len = 5;
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Initiating Diffie-Hellman Group-Exchange "
                       "(Old Method SHA256)");
#endif

        # 设置状态为 created
        key_state->state = libssh2_NB_state_created;
    }
    # 如果密钥状态为已创建，则发送请求
    if(key_state->state == libssh2_NB_state_created) {
        # 调用_libssh2_transport_send函数发送请求，并获取返回状态
        rc = _libssh2_transport_send(session, key_state->request,
                                     key_state->request_len, NULL, 0);
        # 如果返回状态为LIBSSH2_ERROR_EAGAIN，则表示需要再次调用该函数
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 如果返回状态不为LIBSSH2_ERROR_EAGAIN，则表示发送请求出错
        else if(rc) {
            # 调用_libssh2_error函数生成错误信息，并跳转到清理退出的标签
            ret = _libssh2_error(session, rc,
                                 "Unable to send "
                                 "Group Exchange Request SHA256");
            goto dh_gex_clean_exit;
        }

        # 更新密钥状态为已发送
        key_state->state = libssh2_NB_state_sent;
    }

    # 如果密钥状态为已发送，则接收数据
    if(key_state->state == libssh2_NB_state_sent) {
        # 调用_libssh2_packet_require函数接收数据，并获取返回状态
        rc = _libssh2_packet_require(session, SSH_MSG_KEX_DH_GEX_GROUP,
                                     &key_state->data, &key_state->data_len,
                                     0, NULL, 0, &key_state->req_state);
        # 如果返回状态为LIBSSH2_ERROR_EAGAIN，则表示需要再次调用该函数
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 如果返回状态不为LIBSSH2_ERROR_EAGAIN，则表示接收数据出错
        else if(rc) {
            # 调用_libssh2_error函数生成错误信息，并跳转到清理退出的标签
            ret = _libssh2_error(session, rc,
                                 "Timeout waiting for GEX_GROUP reply SHA256");
            goto dh_gex_clean_exit;
        }

        # 更新密钥状态为已发送1
        key_state->state = libssh2_NB_state_sent1;
    }
    # 如果密钥状态为已发送第一步
    if(key_state->state == libssh2_NB_state_sent1) {
        unsigned char *p, *g;  # 定义指向无符号字符的指针p和g
        size_t p_len, g_len;  # 定义p和g的长度
        struct string_buf buf;  # 定义字符串缓冲区buf
        libssh2_sha256_ctx exchange_hash_ctx;  # 定义SHA256上下文exchange_hash_ctx

        # 如果密钥状态的数据长度小于9
        if(key_state->data_len < 9) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                 "Unexpected key length");  # 返回意外的密钥长度错误
            goto dh_gex_clean_exit;  # 跳转到dh_gex_clean_exit标签
        }

        buf.data = key_state->data;  # 将密钥状态的数据赋值给buf的数据
        buf.dataptr = buf.data;  # 将buf的数据指针指向buf的数据
        buf.len = key_state->data_len;  # 将buf的长度设置为密钥状态的数据长度

        buf.dataptr++; /* increment to big num */  # 数据指针向后移动一个位置，指向大数

        # 从buf中获取大数p的字节，并赋值给p和p_len
        if(_libssh2_get_bignum_bytes(&buf, &p, &p_len)) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                 "Unexpected value");  # 返回意外的值错误
            goto dh_gex_clean_exit;  # 跳转到dh_gex_clean_exit标签
        }

        # 从buf中获取大数g的字节，并赋值给g和g_len
        if(_libssh2_get_bignum_bytes(&buf, &g, &g_len)) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                 "Unexpected value");  # 返回意外的值错误
            goto dh_gex_clean_exit;  # 跳转到dh_gex_clean_exit标签
        }

        # 将p和g的字节转换为大数
        _libssh2_bn_from_bin(key_state->p, p_len, p);
        _libssh2_bn_from_bin(key_state->g, g_len, g);

        # 使用Diffie-Hellman算法计算SHA散列
        ret = diffie_hellman_sha_algo(session, key_state->g, key_state->p,
                                      p_len, 256,
                                      (void *)&exchange_hash_ctx,
                                      SSH_MSG_KEX_DH_GEX_INIT,
                                      SSH_MSG_KEX_DH_GEX_REPLY,
                                      key_state->data + 1,
                                      key_state->data_len - 1,
                                      &key_state->exchange_state);
        # 如果返回值为需要再次调用，则直接返回
        if(ret == LIBSSH2_ERROR_EAGAIN) {
            return ret;
        }

        LIBSSH2_FREE(session, key_state->data);  # 释放密钥状态的数据内存
    }

  dh_gex_clean_exit:  # dh_gex_clean_exit标签
    key_state->state = libssh2_NB_state_idle;  # 设置密钥状态为空闲状态
    _libssh2_bn_free(key_state->g);  # 释放密钥状态的g
    key_state->g = NULL;  # 将密钥状态的g设置为NULL
    _libssh2_bn_free(key_state->p);  # 释放密钥状态的p
    key_state->p = NULL;  # 将密钥状态的p设置为NULL

    return ret;  # 返回结果
/* LIBSSH2_KEX_METHOD_EC_SHA_HASH_CREATE_VERIFY
 *
 * 宏，用于创建和验证具有给定摘要字节的 EC SHA 哈希
 *
 * 负载格式：
 *
 * 字符串   V_C，客户端标识字符串（不包括 CR 和 LF）
 * 字符串   V_S，服务器标识字符串（不包括 CR 和 LF）
 * 字符串   I_C，客户端的 SSH_MSG_KEXINIT 负载
 * 字符串   I_S，服务器的 SSH_MSG_KEXINIT 负载
 * 字符串   K_S，服务器的公共主机密钥
 * 字符串   Q_C，客户端的临时公钥八位字符串
 * 字符串   Q_S，服务器的临时公钥八位字符串
 * mpint    K，   共享密钥
 *
 */

#define LIBSSH2_KEX_METHOD_EC_SHA_HASH_CREATE_VERIFY(digest_type)       \
{                                                                       \
    libssh2_sha##digest_type##_ctx ctx;                                 \
    exchange_state->exchange_hash = (void *)&ctx;                       \
    libssh2_sha##digest_type##_init(&ctx);                              \
    if(session->local.banner) {                                         \
        _libssh2_htonu32(exchange_state->h_sig_comp,                    \
                         strlen((char *) session->local.banner) - 2);   \
        libssh2_sha##digest_type##_update(ctx,                          \
                                          exchange_state->h_sig_comp, 4); \
        libssh2_sha##digest_type##_update(ctx,                          \
                                          (char *) session->local.banner, \
                                          strlen((char *)               \
                                                 session->local.banner) \
                                          - 2);                         \
    }                                                                   \
    // 如果不是第一次交换密钥，则将默认 SSH 横幅的长度转换为网络字节顺序，更新到哈希计算上下文中
    else {                                                              
        _libssh2_htonu32(exchange_state->h_sig_comp,                    
                         sizeof(LIBSSH2_SSH_DEFAULT_BANNER) - 1);       
        libssh2_sha##digest_type##_update(ctx,                          
                                          exchange_state->h_sig_comp, 4); 
        libssh2_sha##digest_type##_update(ctx,                          
                                          LIBSSH2_SSH_DEFAULT_BANNER,   
                                          sizeof(LIBSSH2_SSH_DEFAULT_BANNER) 
                                          - 1);                         
    }                                                                   
                                                                        
    // 将远程服务器横幅的长度转换为网络字节顺序，更新到哈希计算上下文中
    _libssh2_htonu32(exchange_state->h_sig_comp,                        
                     strlen((char *) session->remote.banner));          
    libssh2_sha##digest_type##_update(ctx,                              
                                      exchange_state->h_sig_comp, 4);   
    libssh2_sha##digest_type##_update(ctx,                              
                                      session->remote.banner,           
                                      strlen((char *)                   
                                             session->remote.banner));  
                                                                        
    // 将本地密钥交换初始化数据的长度转换为网络字节顺序，更新到哈希计算上下文中
    _libssh2_htonu32(exchange_state->h_sig_comp,                        
                     session->local.kexinit_len);                       
    libssh2_sha##digest_type##_update(ctx,                              
                                      exchange_state->h_sig_comp, 4);   
    # 更新上下文中的哈希值，使用本地密钥交换初始化数据
    libssh2_sha##digest_type##_update(ctx,                              \
                                      session->local.kexinit,           \
                                      session->local.kexinit_len);      \
                                                                        \
    # 将远程密钥交换初始化数据的长度转换为网络字节顺序，并更新哈希值
    _libssh2_htonu32(exchange_state->h_sig_comp,                        \
                     session->remote.kexinit_len);                      \
    libssh2_sha##digest_type##_update(ctx,                              \
                                      exchange_state->h_sig_comp, 4);   \
    # 更新上下文中的哈希值，使用远程密钥交换初始化数据
    libssh2_sha##digest_type##_update(ctx,                              \
                                      session->remote.kexinit,          \
                                      session->remote.kexinit_len);     \
                                                                        \
    # 将服务器主机密钥的长度转换为网络字节顺序，并更新哈希值
    _libssh2_htonu32(exchange_state->h_sig_comp,                        \
                     session->server_hostkey_len);                      \
    libssh2_sha##digest_type##_update(ctx,                              \
                                      exchange_state->h_sig_comp, 4);   \
    # 更新上下文中的哈希值，使用服务器主机密钥数据
    libssh2_sha##digest_type##_update(ctx,                              \
                                      session->server_hostkey,          \
                                      session->server_hostkey_len);     \
                                                                        \
    # 将公钥的长度转换为网络字节顺序，并更新哈希值
    _libssh2_htonu32(exchange_state->h_sig_comp,                        \
                     public_key_len);                                   \
    libssh2_sha##digest_type##_update(ctx,                              \
                                      exchange_state->h_sig_comp, 4);   \
    # 使用指定的摘要类型更新上下文和公钥的数据
    libssh2_sha##digest_type##_update(ctx,                              \
                                      public_key,                       \
                                      public_key_len);                  \
    # 将服务器公钥长度转换为网络字节顺序，并更新上下文
    _libssh2_htonu32(exchange_state->h_sig_comp,                        \
                     server_public_key_len);                            \
    # 使用指定的摘要类型更新上下文和服务器公钥的数据
    libssh2_sha##digest_type##_update(ctx,                              \
                                      exchange_state->h_sig_comp, 4);   \
    libssh2_sha##digest_type##_update(ctx,                              \
                                      server_public_key,                \
                                      server_public_key_len);           \
    # 使用指定的摘要类型更新上下文和交换状态的 k_value 数据
    libssh2_sha##digest_type##_update(ctx,                              \
                                      exchange_state->k_value,          \
                                      exchange_state->k_value_len);     \
    # 完成指定摘要类型的计算，并将结果存储在 h_sig_comp 中
    libssh2_sha##digest_type##_final(ctx, exchange_state->h_sig_comp);  \
    # 验证服务器签名
    if(session->hostkey->                                               \
       sig_verify(session, exchange_state->h_sig,                       \
                  exchange_state->h_sig_len, exchange_state->h_sig_comp, \
                  SHA##digest_type##_DIGEST_LENGTH,                     \
                  &session->server_hostkey_abstract)) {                 \
        rc = -1;                                                        \
    }                                                                   \
# 如果 LIBSSH2_ECDSA 被定义

# 定义函数 kex_session_ecdh_curve_type，用于返回用于密钥交换的 EC 曲线类型
static int
kex_session_ecdh_curve_type(const char *name, libssh2_curve_type *out_type)
{
    int ret = 0;
    libssh2_curve_type type;

    # 如果输入的名称为空，则返回错误
    if(name == NULL)
        return -1;

    # 根据输入的名称判断曲线类型
    if(strcmp(name, "ecdh-sha2-nistp256") == 0)
        type = LIBSSH2_EC_CURVE_NISTP256;
    else if(strcmp(name, "ecdh-sha2-nistp384") == 0)
        type = LIBSSH2_EC_CURVE_NISTP384;
    else if(strcmp(name, "ecdh-sha2-nistp521") == 0)
        type = LIBSSH2_EC_CURVE_NISTP521;
    else {
        ret = -1;
    }

    # 如果返回值为 0 并且输出类型不为空，则将类型赋给输出类型
    if(ret == 0 && out_type) {
        *out_type = type;
    }

    return ret;
}

# 定义函数 ecdh_sha2_nistp，用于椭圆曲线 Diffie Hellman 密钥交换
static int ecdh_sha2_nistp(LIBSSH2_SESSION *session, libssh2_curve_type type,
                           unsigned char *data, size_t data_len,
                           unsigned char *public_key,
                           size_t public_key_len, _libssh2_ec_key *private_key,
                           kmdhgGPshakex_state_t *exchange_state)
{
    int ret = 0;
    int rc;

    # 如果数据长度小于 5，则返回错误
    if(data_len < 5) {
        ret = _libssh2_error(session, LIBSSH2_ERROR_HOSTKEY_INIT,
                            "Host key data is too short");
        return ret;
    }

    # 如果交换状态为闲置状态
    if(exchange_state->state == libssh2_NB_state_idle) {

        # 设置初始值
        exchange_state->k = _libssh2_bn_init();

        exchange_state->state = libssh2_NB_state_created;
    }
}
    # 如果交换状态为已创建
    if(exchange_state->state == libssh2_NB_state_created) {
        # 解析INIT回复数据

        # 服务器公钥K_S
        unsigned char *server_public_key;
        size_t server_public_key_len;
        struct string_buf buf;

        # 将数据和数据长度封装到buf中
        buf.data = data;
        buf.len = data_len;
        buf.dataptr = buf.data;
        buf.dataptr++; # 跳过数据包类型

         # 复制服务器公钥到session->server_hostkey，并获取服务器公钥长度
         if(_libssh2_copy_string(session, &buf, &(session->server_hostkey),
                                &server_public_key_len)) {
             ret = _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for a copy "
                                  "of the host key");
            goto clean_exit;
        }

        # 设置服务器公钥长度
        session->server_hostkey_len = (uint32_t)server_public_key_len;
    }
#if LIBSSH2_MD5
        {
            // 如果支持 MD5
            libssh2_md5_ctx fingerprint_ctx;

            // 初始化 MD5 上下文
            if(libssh2_md5_init(&fingerprint_ctx)) {
                // 更新 MD5 上下文
                libssh2_md5_update(fingerprint_ctx, session->server_hostkey,
                                   session->server_hostkey_len);
                // 完成 MD5 计算
                libssh2_md5_final(fingerprint_ctx,
                                  session->server_hostkey_md5);
                // 标记 MD5 计算结果有效
                session->server_hostkey_md5_valid = TRUE;
            }
            else {
                // 标记 MD5 计算结果无效
                session->server_hostkey_md5_valid = FALSE;
            }
        }
#ifdef LIBSSH2DEBUG
        {
            // 如果定义了 LIBSSH2DEBUG
            char fingerprint[50], *fprint = fingerprint;
            int i;
            // 生成 MD5 指纹
            for(i = 0; i < 16; i++, fprint += 3) {
                snprintf(fprint, 4, "%02x:", session->server_hostkey_md5[i]);
            }
            *(--fprint) = '\0';
            // 输出 MD5 指纹
            _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                           "Server's MD5 Fingerprint: %s", fingerprint);
        }
#endif /* LIBSSH2DEBUG */
#endif /* ! LIBSSH2_MD5 */

        {
            // 初始化 SHA1 上下文
            libssh2_sha1_ctx fingerprint_ctx;

            // 如果支持 SHA1
            if(libssh2_sha1_init(&fingerprint_ctx)) {
                // 更新 SHA1 上下文
                libssh2_sha1_update(fingerprint_ctx, session->server_hostkey,
                                    session->server_hostkey_len);
                // 完成 SHA1 计算
                libssh2_sha1_final(fingerprint_ctx,
                                   session->server_hostkey_sha1);
                // 标记 SHA1 计算结果有效
                session->server_hostkey_sha1_valid = TRUE;
            }
            else {
                // 标记 SHA1 计算结果无效
                session->server_hostkey_sha1_valid = FALSE;
            }
        }
#ifdef LIBSSH2DEBUG
        {
            // 如果定义了 LIBSSH2DEBUG
            char fingerprint[64], *fprint = fingerprint;
            int i;

            // 生成 SHA1 指纹
            for(i = 0; i < 20; i++, fprint += 3) {
                snprintf(fprint, 4, "%02x:", session->server_hostkey_sha1[i]);
            }
            *(--fprint) = '\0';
            // 输出 SHA1 指纹
            _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                           "Server's SHA1 Fingerprint: %s", fingerprint);
        }
#endif /* LIBSSH2DEBUG */

        /* 计算服务器的公钥的 SHA256 指纹 */
        {
            libssh2_sha256_ctx fingerprint_ctx;

            // 初始化 SHA256 上下文
            if(libssh2_sha256_init(&fingerprint_ctx)) {
                // 更新 SHA256 上下文，计算服务器的公钥的 SHA256 指纹
                libssh2_sha256_update(fingerprint_ctx, session->server_hostkey,
                                      session->server_hostkey_len);
                // 完成 SHA256 计算，将结果存储在 session->server_hostkey_sha256 中
                libssh2_sha256_final(fingerprint_ctx,
                                     session->server_hostkey_sha256);
                session->server_hostkey_sha256_valid = TRUE;
            }
            else {
                session->server_hostkey_sha256_valid = FALSE;
            }
        }
#ifdef LIBSSH2DEBUG
        {
            char *base64Fingerprint = NULL;
            // 将服务器的 SHA256 指纹进行 Base64 编码
            _libssh2_base64_encode(session,
                                   (const char *)
                                   session->server_hostkey_sha256,
                                   SHA256_DIGEST_LENGTH, &base64Fingerprint);
            // 如果成功编码，则输出调试信息
            if(base64Fingerprint != NULL) {
                _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                               "Server's SHA256 Fingerprint: %s",
                               base64Fingerprint);
                LIBSSH2_FREE(session, base64Fingerprint);
            }
        }
    }

    // 发送 NEWKEYS 消息
    if(exchange_state->state == libssh2_NB_state_sent) {
        // 发送 NEWKEYS 消息
        rc = _libssh2_transport_send(session, &exchange_state->c, 1, NULL, 0);
        // 如果是 EAGAIN 错误，则返回
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        // 如果有其他错误，则输出错误信息并跳转到清理退出
        else if(rc) {
            ret = _libssh2_error(session, rc,
                                 "Unable to send NEWKEYS message");
            goto clean_exit;
        }

        exchange_state->state = libssh2_NB_state_sent2;
    }

    }

clean_exit:
    // 释放资源并重置状态
    _libssh2_bn_free(exchange_state->k);
    exchange_state->k = NULL;

    if(exchange_state->k_value) {
        LIBSSH2_FREE(session, exchange_state->k_value);
        exchange_state->k_value = NULL;
    }

    exchange_state->state = libssh2_NB_state_idle;

    // 返回结果
    return ret;
}
/* kex_method_ecdh_key_exchange
 *
 * 椭圆曲线Diffie Hellman密钥交换
 * 支持基于协商的ecdh方法的SHA256/384/512哈希
 *
 */

static int
kex_method_ecdh_key_exchange
(LIBSSH2_SESSION * session, key_exchange_state_low_t * key_state)
{
    int ret = 0;  // 返回值初始化为0
    int rc = 0;  // 返回码初始化为0
    unsigned char *s;  // 定义一个无符号字符指针
    libssh2_curve_type type;  // 定义一个libssh2_curve_type类型的变量

    if(key_state->state == libssh2_NB_state_idle) {  // 如果状态为idle
        key_state->public_key_oct = NULL;  // 将公钥置为空
        key_state->state = libssh2_NB_state_created;  // 将状态设置为created
    }

    if(key_state->state == libssh2_NB_state_created) {  // 如果状态为created
        rc = kex_session_ecdh_curve_type(session->kex->name, &type);  // 调用函数获取曲线类型

        if(rc != 0) {  // 如果返回码不为0
            ret = _libssh2_error(session, -1,
                                 "Unknown KEX nistp curve type");  // 设置错误信息
            goto ecdh_clean_exit;  // 跳转到清理退出的标签
        }

        rc = _libssh2_ecdsa_create_key(session, &key_state->private_key,
                                       &key_state->public_key_oct,
                                       &key_state->public_key_oct_len, type);  // 创建私钥和公钥

        if(rc != 0) {  // 如果返回码不为0
            ret = _libssh2_error(session, rc,
                                 "Unable to create private key");  // 设置错误信息
            goto ecdh_clean_exit;  // 跳转到清理退出的标签
        }

        key_state->request[0] = SSH2_MSG_KEX_ECDH_INIT;  // 设置请求消息的第一个字节
        s = key_state->request + 1;  // 将s指向请求消息的第二个字节
        _libssh2_store_str(&s, (const char *)key_state->public_key_oct,
                           key_state->public_key_oct_len);  // 存储公钥到请求消息中
        key_state->request_len = key_state->public_key_oct_len + 5;  // 设置请求消息的长度

        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                       "Initiating ECDH SHA2 NISTP256");  // 调试信息

        key_state->state = libssh2_NB_state_sent;  // 设置状态为sent
    }
    # 如果密钥状态为已发送，则发送请求
    if(key_state->state == libssh2_NB_state_sent) {
        rc = _libssh2_transport_send(session, key_state->request,
                                     key_state->request_len, NULL, 0);
        # 如果返回值为需要再次调用，则返回该值
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 如果返回值不为0，则生成错误信息并跳转到清理退出标签
        else if(rc) {
            ret = _libssh2_error(session, rc,
                                 "Unable to send ECDH_INIT");
            goto ecdh_clean_exit;
        }

        # 更新密钥状态为已发送1
        key_state->state = libssh2_NB_state_sent1;
    }

    # 如果密钥状态为已发送1，则等待接收回复
    if(key_state->state == libssh2_NB_state_sent1) {
        rc = _libssh2_packet_require(session, SSH2_MSG_KEX_ECDH_REPLY,
                                     &key_state->data, &key_state->data_len,
                                     0, NULL, 0, &key_state->req_state);
        # 如果返回值为需要再次调用，则返回该值
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 如果返回值不为0，则生成错误信息并跳转到清理退出标签
        else if(rc) {
            ret = _libssh2_error(session, rc,
                                 "Timeout waiting for ECDH_REPLY reply");
            goto ecdh_clean_exit;
        }

        # 更新密钥状态为已发送2
        key_state->state = libssh2_NB_state_sent2;
    }

    # 如果密钥状态为已发送2，则进行密钥交换
    if(key_state->state == libssh2_NB_state_sent2) {

        # 获取密钥交换协议的曲线类型
        (void)kex_session_ecdh_curve_type(session->kex->name, &type);

        # 执行椭圆曲线 Diffie-Hellman 密钥交换
        ret = ecdh_sha2_nistp(session, type, key_state->data,
                              key_state->data_len,
                              (unsigned char *)key_state->public_key_oct,
                              key_state->public_key_oct_len,
                              key_state->private_key,
                              &key_state->exchange_state);

        # 如果返回值为需要再次调用，则返回该值
        if(ret == LIBSSH2_ERROR_EAGAIN) {
            return ret;
        }

        # 释放密钥交换数据
        LIBSSH2_FREE(session, key_state->data);
    }
# 清理 ECDH 密钥交换状态并退出
ecdh_clean_exit:
    # 如果公钥字节流存在，则释放内存并将指针置为空
    if(key_state->public_key_oct) {
        LIBSSH2_FREE(session, key_state->public_key_oct);
        key_state->public_key_oct = NULL;
    }
    # 如果私钥存在，则释放内存并将指针置为空
    if(key_state->private_key) {
        _libssh2_ecdsa_free(key_state->private_key);
        key_state->private_key = NULL;
    }
    # 将密钥状态设置为闲置状态
    key_state->state = libssh2_NB_state_idle;
    # 返回结果
    return ret;
}

#endif /*LIBSSH2_ECDSA*/

#if LIBSSH2_ED25519

/* curve25519_sha256
 * 椭圆曲线密钥交换
 */

static int
curve25519_sha256(LIBSSH2_SESSION *session, unsigned char *data,
                  size_t data_len,
                  unsigned char public_key[LIBSSH2_ED25519_KEY_LEN],
                  unsigned char private_key[LIBSSH2_ED25519_KEY_LEN],
                  kmdhgGPshakex_state_t *exchange_state)
{
    int ret = 0;
    int rc;
    int public_key_len = LIBSSH2_ED25519_KEY_LEN;
    # 如果数据长度小于5，则返回错误
    if(data_len < 5) {
        return _libssh2_error(session, LIBSSH2_ERROR_HOSTKEY_INIT,
                              "Data is too short");
    }
    # 如果密钥交换状态为闲置状态
    if(exchange_state->state == libssh2_NB_state_idle) {
        # 设置初始值
        exchange_state->k = _libssh2_bn_init();
        # 将密钥交换状态设置为已创建状态
        exchange_state->state = libssh2_NB_state_created;
    }
    # 如果交换状态为已创建
    if(exchange_state->state == libssh2_NB_state_created) {
        # 解析INIT回复数据
        unsigned char *server_public_key, *server_host_key;
        size_t server_public_key_len, hostkey_len;
        struct string_buf buf;

        # 如果数据长度小于5
        if(data_len < 5) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                 "Unexpected key length");
            goto clean_exit;
        }

        # 将数据封装成字符串缓冲区
        buf.data = data;
        buf.len = data_len;
        buf.dataptr = buf.data;
        buf.dataptr++; # 跳过数据包类型

        # 获取服务器主机密钥和主机密钥长度
        if(_libssh2_get_string(&buf, &server_host_key, &hostkey_len)) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                 "Unexpected key length");
            goto clean_exit;
        }

        # 设置会话的服务器主机密钥长度和服务器主机密钥
        session->server_hostkey_len = (uint32_t)hostkey_len;
        session->server_hostkey = LIBSSH2_ALLOC(session,
                                                session->server_hostkey_len);
        # 如果无法分配内存用于主机密钥的副本
        if(!session->server_hostkey) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                 "Unable to allocate memory for a copy "
                                 "of the host key");
            goto clean_exit;
        }

        # 复制服务器主机密钥到会话的服务器主机密钥
        memcpy(session->server_hostkey, server_host_key,
               session->server_hostkey_len);
#if LIBSSH2_MD5
        {
            // 如果支持 MD5
            libssh2_md5_ctx fingerprint_ctx;

            // 初始化 MD5 上下文
            if(libssh2_md5_init(&fingerprint_ctx)) {
                // 更新 MD5 上下文
                libssh2_md5_update(fingerprint_ctx, session->server_hostkey,
                                   session->server_hostkey_len);
                // 完成 MD5 计算
                libssh2_md5_final(fingerprint_ctx,
                                  session->server_hostkey_md5);
                // 标记 MD5 计算结果有效
                session->server_hostkey_md5_valid = TRUE;
            }
            else {
                // 标记 MD5 计算结果无效
                session->server_hostkey_md5_valid = FALSE;
            }
        }
#ifdef LIBSSH2DEBUG
        {
            // 如果定义了 LIBSSH2DEBUG
            char fingerprint[50], *fprint = fingerprint;
            int i;
            // 遍历 MD5 计算结果，生成指纹字符串
            for(i = 0; i < 16; i++, fprint += 3) {
                snprintf(fprint, 4, "%02x:", session->server_hostkey_md5[i]);
            }
            *(--fprint) = '\0';
            // 输出服务器的 MD5 指纹
            _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                             "Server's MD5 Fingerprint: %s", fingerprint);
        }
#endif /* LIBSSH2DEBUG */
#endif /* ! LIBSSH2_MD5 */

        {
            // 初始化 SHA1 上下文
            libssh2_sha1_ctx fingerprint_ctx;

            // 如果支持 SHA1
            if(libssh2_sha1_init(&fingerprint_ctx)) {
                // 更新 SHA1 上下文
                libssh2_sha1_update(fingerprint_ctx, session->server_hostkey,
                                    session->server_hostkey_len);
                // 完成 SHA1 计算
                libssh2_sha1_final(fingerprint_ctx,
                                   session->server_hostkey_sha1);
                // 标记 SHA1 计算结果有效
                session->server_hostkey_sha1_valid = TRUE;
            }
            else {
                // 标记 SHA1 计算结果无效
                session->server_hostkey_sha1_valid = FALSE;
            }
        }
#ifdef LIBSSH2DEBUG
        {
            // 创建一个字符数组来存储指纹，初始化指针指向该数组
            char fingerprint[64], *fprint = fingerprint;
            int i;

            // 循环遍历服务器主机密钥的 SHA1 值，将其格式化为十六进制字符串
            for(i = 0; i < 20; i++, fprint += 3) {
                snprintf(fprint, 4, "%02x:", session->server_hostkey_sha1[i]);
            }
            // 将最后一个冒号替换为结束符
            *(--fprint) = '\0';
            // 输出服务器的 SHA1 指纹
            _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                             "Server's SHA1 Fingerprint: %s", fingerprint);
        }
#endif /* LIBSSH2DEBUG */

        /* SHA256 */
        {
            // 创建 SHA256 上下文
            libssh2_sha256_ctx fingerprint_ctx;

            // 如果初始化 SHA256 上下文成功
            if(libssh2_sha256_init(&fingerprint_ctx)) {
                // 更新 SHA256 上下文，使用服务器主机密钥的数据
                libssh2_sha256_update(fingerprint_ctx, session->server_hostkey,
                                      session->server_hostkey_len);
                // 完成 SHA256 计算，将结果存储在 session->server_hostkey_sha256 中
                libssh2_sha256_final(fingerprint_ctx,
                                     session->server_hostkey_sha256);
                // 设置服务器主机密钥的 SHA256 值为有效
                session->server_hostkey_sha256_valid = TRUE;
            }
            else {
                // 如果初始化 SHA256 上下文失败，将服务器主机密钥的 SHA256 值标记为无效
                session->server_hostkey_sha256_valid = FALSE;
            }
        }
#ifdef LIBSSH2DEBUG
        {
            // 创建一个指向 base64 指纹的指针
            char *base64Fingerprint = NULL;
            // 将服务器主机密钥的 SHA256 值进行 base64 编码
            _libssh2_base64_encode(session,
                                   (const char *)
                                   session->server_hostkey_sha256,
                                   SHA256_DIGEST_LENGTH, &base64Fingerprint);
            // 如果 base64 编码成功
            if(base64Fingerprint != NULL) {
                // 输出服务器的 SHA256 指纹
                _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                               "Server's SHA256 Fingerprint: %s",
                               base64Fingerprint);
                // 释放 base64 指纹的内存
                LIBSSH2_FREE(session, base64Fingerprint);
            }
        }
    }
    # 如果交换状态为已发送
    if(exchange_state->state == libssh2_NB_state_sent) {
        # 调用_libssh2_transport_send函数发送数据
        rc = _libssh2_transport_send(session, &exchange_state->c, 1, NULL, 0);
        # 如果返回值为LIBSSH2_ERROR_EAGAIN，表示需要再次调用该函数
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            # 返回rc值
            return rc;
        }
        # 如果返回值不为0，表示发送失败
        else if(rc) {
            # 调用_libssh2_error函数生成错误信息
            ret = _libssh2_error(session, rc, "Unable to send NEWKEYS message");
            # 跳转到clean_exit标签处
            goto clean_exit;
        }

        # 修改交换状态为已发送2
        exchange_state->state = libssh2_NB_state_sent2;
    }

    # 结束if语句块
    }
# 清理退出：释放资源并返回结果
clean_exit:
    # 释放交换状态中的 k 值
    _libssh2_bn_free(exchange_state->k);
    exchange_state->k = NULL;

    # 如果存在 k_value，则释放并置空
    if(exchange_state->k_value) {
        LIBSSH2_FREE(session, exchange_state->k_value);
        exchange_state->k_value = NULL;
    }

    # 将交换状态设置为闲置状态
    exchange_state->state = libssh2_NB_state_idle;

    # 返回结果
    return ret;
}

/* kex_method_curve25519_key_exchange
 *
 * Elliptic Curve X25519 Key Exchange with SHA256 hash
 *
 */

# 定义椭圆曲线 X25519 密钥交换方法
static int
kex_method_curve25519_key_exchange
(LIBSSH2_SESSION * session, key_exchange_state_low_t * key_state)
{
    int ret = 0;
    int rc = 0;

    # 如果状态为闲置，则进行初始化
    if(key_state->state == libssh2_NB_state_idle) {

        # 置空公钥
        key_state->public_key_oct = NULL;
        # 设置状态为已创建
        key_state->state = libssh2_NB_state_created;
    }

    # 如果状态为已创建
    if(key_state->state == libssh2_NB_state_created) {
        unsigned char *s = NULL;

        # 比较密钥交换算法名称
        rc = strcmp(session->kex->name, "curve25519-sha256@libssh.org");
        if(rc != 0)
            rc = strcmp(session->kex->name, "curve25519-sha256");

        # 如果不匹配，则返回错误
        if(rc != 0) {
            ret = _libssh2_error(session, -1,
                                 "Unknown KEX curve25519 curve type");
            goto clean_exit;
        }

        # 创建新的 curve25519 密钥对
        rc = _libssh2_curve25519_new(session,
                                     &key_state->curve25519_public_key,
                                     &key_state->curve25519_private_key);

        # 如果创建失败，则返回错误
        if(rc != 0) {
            ret = _libssh2_error(session, rc,
                                 "Unable to create private key");
            goto clean_exit;
        }

        # 设置请求消息类型为 SSH2_MSG_KEX_ECDH_INIT
        key_state->request[0] = SSH2_MSG_KEX_ECDH_INIT;
        s = key_state->request + 1;
        # 存储 curve25519 公钥到请求消息中
        _libssh2_store_str(&s, (const char *)key_state->curve25519_public_key,
                           LIBSSH2_ED25519_KEY_LEN);
        key_state->request_len = LIBSSH2_ED25519_KEY_LEN + 5;

        # 调试信息：初始化 curve25519 SHA2
        _libssh2_debug(session, LIBSSH2_TRACE_KEX,
                        "Initiating curve25519 SHA2");

        # 设置状态为已发送
        key_state->state = libssh2_NB_state_sent;
    }
}
    # 如果密钥状态为已发送，则发送请求数据
    if(key_state->state == libssh2_NB_state_sent) {
        rc = _libssh2_transport_send(session, key_state->request,
                                     key_state->request_len, NULL, 0);
        # 如果返回值为需要再次调用，则直接返回
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 如果返回值不为0，则生成错误信息并跳转到清理退出
        else if(rc) {
            ret = _libssh2_error(session, rc,
                                 "Unable to send ECDH_INIT");
            goto clean_exit;
        }

        # 更新密钥状态为已发送1
        key_state->state = libssh2_NB_state_sent1;
    }

    # 如果密钥状态为已发送1，则等待接收回复数据
    if(key_state->state == libssh2_NB_state_sent1) {
        rc = _libssh2_packet_require(session, SSH2_MSG_KEX_ECDH_REPLY,
                                     &key_state->data, &key_state->data_len,
                                     0, NULL, 0, &key_state->req_state);
        # 如果返回值为需要再次调用，则直接返回
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 如果返回值不为0，则生成错误信息并跳转到清理退出
        else if(rc) {
            ret = _libssh2_error(session, rc,
                                 "Timeout waiting for ECDH_REPLY reply");
            goto clean_exit;
        }

        # 更新密钥状态为已发送2
        key_state->state = libssh2_NB_state_sent2;
    }

    # 如果密钥状态为已发送2，则进行密钥交换
    if(key_state->state == libssh2_NB_state_sent2) {

        # 调用密钥交换函数，生成共享密钥
        ret = curve25519_sha256(session, key_state->data, key_state->data_len,
                                key_state->curve25519_public_key,
                                key_state->curve25519_private_key,
                                &key_state->exchange_state);

        # 如果返回值为需要再次调用，则直接返回
        if(ret == LIBSSH2_ERROR_EAGAIN) {
            return ret;
        }

        # 释放数据内存
        LIBSSH2_FREE(session, key_state->data);
    }
// 清理退出：释放内存并将状态设置为闲置
clean_exit:

    // 如果存在 curve25519 公钥，则清零并释放内存
    if(key_state->curve25519_public_key) {
        _libssh2_explicit_zero(key_state->curve25519_public_key,
                               LIBSSH2_ED25519_KEY_LEN);
        LIBSSH2_FREE(session, key_state->curve25519_public_key);
        key_state->curve25519_public_key = NULL;
    }

    // 如果存在 curve25519 私钥，则清零并释放内存
    if(key_state->curve25519_private_key) {
        _libssh2_explicit_zero(key_state->curve25519_private_key,
                               LIBSSH2_ED25519_KEY_LEN);
        LIBSSH2_FREE(session, key_state->curve25519_private_key);
        key_state->curve25519_private_key = NULL;
    }

    // 将状态设置为闲置
    key_state->state = libssh2_NB_state_idle;

    // 返回结果
    return ret;
}


#endif /*LIBSSH2_ED25519*/


// 定义不同的密钥交换方法
#define LIBSSH2_KEX_METHOD_FLAG_REQ_ENC_HOSTKEY     0x0001
#define LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY    0x0002

// 定义 diffie-hellman-group1-sha1 密钥交换方法
static const LIBSSH2_KEX_METHOD kex_method_diffie_helman_group1_sha1 = {
    "diffie-hellman-group1-sha1",
    kex_method_diffie_hellman_group1_sha1_key_exchange,
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
};

// 定义 diffie-hellman-group14-sha1 密钥交换方法
static const LIBSSH2_KEX_METHOD kex_method_diffie_helman_group14_sha1 = {
    "diffie-hellman-group14-sha1",
    kex_method_diffie_hellman_group14_sha1_key_exchange,
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
};

// 定义 diffie-hellman-group14-sha256 密钥交换方法
static const LIBSSH2_KEX_METHOD kex_method_diffie_helman_group14_sha256 = {
    "diffie-hellman-group14-sha256",
    kex_method_diffie_hellman_group14_sha256_key_exchange,
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
};

// 定义 diffie-hellman-group16-sha512 密钥交换方法
static const LIBSSH2_KEX_METHOD kex_method_diffie_helman_group16_sha512 = {
    "diffie-hellman-group16-sha512",
    kex_method_diffie_hellman_group16_sha512_key_exchange,
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
};

// 定义 diffie-hellman-group18-sha512 密钥交换方法
static const LIBSSH2_KEX_METHOD kex_method_diffie_helman_group18_sha512 = {
    "diffie-hellman-group18-sha512",
    kex_method_diffie_hellman_group18_sha512_key_exchange,
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
};

// 定义 diffie-hellman-group-exchange-sha1 密钥交换方法
static const LIBSSH2_KEX_METHOD
kex_method_diffie_helman_group_exchange_sha1 = {
    # 定义字符串 "diffie-hellman-group-exchange-sha1"
    "diffie-hellman-group-exchange-sha1",
    # 使用 diffie_hellman_group_exchange_sha1_key_exchange 方法进行密钥交换
    kex_method_diffie_hellman_group_exchange_sha1_key_exchange,
    # 设置标志位，要求对主机密钥进行签名验证
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
// 定义了一个静态常量，表示 diffie-hellman-group-exchange-sha256 方法的密钥交换方式
static const LIBSSH2_KEX_METHOD
kex_method_diffie_helman_group_exchange_sha256 = {
    "diffie-hellman-group-exchange-sha256",
    kex_method_diffie_hellman_group_exchange_sha256_key_exchange,
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
};

// 如果支持 ECDSA，则定义了 ecdh-sha2-nistp256 方法的密钥交换方式
#if LIBSSH2_ECDSA
static const LIBSSH2_KEX_METHOD
kex_method_ecdh_sha2_nistp256 = {
    "ecdh-sha2-nistp256",
    kex_method_ecdh_key_exchange,
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
};

// 如果支持 ECDSA，则定义了 ecdh-sha2-nistp384 方法的密钥交换方式
static const LIBSSH2_KEX_METHOD
kex_method_ecdh_sha2_nistp384 = {
    "ecdh-sha2-nistp384",
    kex_method_ecdh_key_exchange,
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
};

// 如果支持 ECDSA，则定义了 ecdh-sha2-nistp521 方法的密钥交换方式
static const LIBSSH2_KEX_METHOD
kex_method_ecdh_sha2_nistp521 = {
    "ecdh-sha2-nistp521",
    kex_method_ecdh_key_exchange,
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
};
#endif

// 如果支持 ED25519，则定义了 curve25519-sha256@libssh.org 方法的密钥交换方式
#if LIBSSH2_ED25519
static const LIBSSH2_KEX_METHOD
kex_method_ssh_curve25519_sha256_libssh = {
    "curve25519-sha256@libssh.org",
    kex_method_curve25519_key_exchange,
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
};

// 如果支持 ED25519，则定义了 curve25519-sha256 方法的密钥交换方式
static const LIBSSH2_KEX_METHOD
kex_method_ssh_curve25519_sha256 = {
    "curve25519-sha256",
    kex_method_curve25519_key_exchange,
    LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY,
};
#endif

// 定义了 libssh2_kex_methods 数组，包含了支持的密钥交换方式
static const LIBSSH2_KEX_METHOD *libssh2_kex_methods[] = {
#if LIBSSH2_ED25519
    &kex_method_ssh_curve25519_sha256,
    &kex_method_ssh_curve25519_sha256_libssh,
#endif
#if LIBSSH2_ECDSA
    &kex_method_ecdh_sha2_nistp256,
    &kex_method_ecdh_sha2_nistp384,
    &kex_method_ecdh_sha2_nistp521,
#endif
    &kex_method_diffie_helman_group_exchange_sha256,
    &kex_method_diffie_helman_group16_sha512,
    &kex_method_diffie_helman_group18_sha512,
    &kex_method_diffie_helman_group14_sha256,
    &kex_method_diffie_helman_group14_sha1,
    &kex_method_diffie_helman_group1_sha1,
    &kex_method_diffie_helman_group_exchange_sha1,
  NULL
};

// 定义了 LIBSSH2_COMMON_METHOD 结构体，包含了一个字符串指针
typedef struct _LIBSSH2_COMMON_METHOD
{
    const char *name;
} LIBSSH2_COMMON_METHOD;
/* kex_method_strlen
 * 计算特定方法列表生成的字符串的长度
 * 包括每个单独方法的 strlen() 之和加 1（逗号）- 1
 * （因为最后一个逗号没有使用）
 * 又一个糟糕的编码实践的迹象。假装你没看到这个。
 */
static size_t
kex_method_strlen(LIBSSH2_COMMON_METHOD ** method)
{
    size_t len = 0;

    if(!method || !*method) {
        return 0;
    }

    while(*method && (*method)->name) {
        len += strlen((*method)->name) + 1;
        method++;
    }

    return len - 1;
}



/* kex_method_list
 * 在 buf 中生成格式化的偏好列表
 */
static size_t
kex_method_list(unsigned char *buf, size_t list_strlen,
                LIBSSH2_COMMON_METHOD ** method)
{
    _libssh2_htonu32(buf, list_strlen);
    buf += 4;

    if(!method || !*method) {
        return 4;
    }

    while(*method && (*method)->name) {
        int mlen = strlen((*method)->name);
        memcpy(buf, (*method)->name, mlen);
        buf += mlen;
        *(buf++) = ',';
        method++;
    }

    return list_strlen + 4;
}



#define LIBSSH2_METHOD_PREFS_LEN(prefvar, defaultvar)           \
    ((prefvar) ? strlen(prefvar) :                              \
     kex_method_strlen((LIBSSH2_COMMON_METHOD**)(defaultvar)))

#define LIBSSH2_METHOD_PREFS_STR(buf, prefvarlen, prefvar, defaultvar)  \
    if(prefvar) {                                                       \
        _libssh2_htonu32((buf), (prefvarlen));                          \
        buf += 4;                                                       \
        memcpy((buf), (prefvar), (prefvarlen));                         \
        buf += (prefvarlen);                                            \
    }                                                                   \
    # 如果不满足前面的条件，执行以下代码块
    else {                                                              \
        # 将 kex_method_list 函数的返回值添加到 buf 中
        buf += kex_method_list((buf), (prefvarlen),                     \
                               (LIBSSH2_COMMON_METHOD**)(defaultvar));  \
    }
/* kexinit
 * 发送 SSH_MSG_KEXINIT 数据包
 */
static int kexinit(LIBSSH2_SESSION * session)
{
    /* 62 = packet_type(1) + cookie(16) + first_packet_follows(1) +
       reserved(4) + length longs(40) */
    // 计算数据包长度
    size_t data_len = 62;
    size_t kex_len, hostkey_len = 0;
    size_t crypt_cs_len, crypt_sc_len;
    size_t comp_cs_len, comp_sc_len;
    size_t mac_cs_len, mac_sc_len;
    size_t lang_cs_len, lang_sc_len;
    unsigned char *data, *s;
    int rc;

#ifdef LIBSSH2DEBUG
        {
            /* Funnily enough, they'll all "appear" to be '\0' terminated */
            // 调试模式下输出发送的各部分数据
            unsigned char *p = data + 21;   /* type(1) + cookie(16) + len(4) */
            _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Sent KEX: %s", p);
            p += kex_len + 4;
            _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Sent HOSTKEY: %s", p);
            p += hostkey_len + 4;
            _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Sent CRYPT_CS: %s", p);
            p += crypt_cs_len + 4;
            _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Sent CRYPT_SC: %s", p);
            p += crypt_sc_len + 4;
            _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Sent MAC_CS: %s", p);
            p += mac_cs_len + 4;
            _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Sent MAC_SC: %s", p);
            p += mac_sc_len + 4;
            _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Sent COMP_CS: %s", p);
            p += comp_cs_len + 4;
            _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Sent COMP_SC: %s", p);
            p += comp_sc_len + 4;
            _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Sent LANG_CS: %s", p);
            p += lang_cs_len + 4;
            _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Sent LANG_SC: %s", p);
            p += lang_sc_len + 4;
        }
#endif /* LIBSSH2DEBUG */

        // 设置状态为已创建
        session->kexinit_state = libssh2_NB_state_created;
    }
    else {
        // 如果不是第一次发送KEXINIT数据，则将session->kexinit_data和session->kexinit_data_len赋给data和data_len
        data = session->kexinit_data;
        data_len = session->kexinit_data_len;
        // 清空session->kexinit_data和session->kexinit_data_len，以确保后续不会出现双重释放的情况
        session->kexinit_data = NULL;
        session->kexinit_data_len = 0;
    }

    // 发送KEXINIT数据
    rc = _libssh2_transport_send(session, data, data_len, NULL, 0);
    // 如果返回值是LIBSSH2_ERROR_EAGAIN，表示需要再次发送KEXINIT数据
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        // 将data和data_len重新赋给session->kexinit_data和session->kexinit_data_len
        session->kexinit_data = data;
        session->kexinit_data_len = data_len;
        return rc;
    }
    // 如果返回值不是LIBSSH2_ERROR_EAGAIN
    else if(rc) {
        // 释放data所指向的内存
        LIBSSH2_FREE(session, data);
        // 将kexinit_state设置为libssh2_NB_state_idle
        session->kexinit_state = libssh2_NB_state_idle;
        // 返回错误信息
        return _libssh2_error(session, rc, "Unable to send KEXINIT packet to remote host");
    }

    // 如果session->local.kexinit不为空
    if(session->local.kexinit) {
        // 释放session->local.kexinit所指向的内存
        LIBSSH2_FREE(session, session->local.kexinit);
    }

    // 将data赋给session->local.kexinit，将data_len赋给session->local.kexinit_len
    session->local.kexinit = data;
    session->local.kexinit_len = data_len;

    // 将kexinit_state设置为libssh2_NB_state_idle
    session->kexinit_state = libssh2_NB_state_idle;

    // 返回0
    return 0;
/* kex_agree_instr
 * Kex specific variant of strstr()
 * Needle must be precede by BOL or ',', and followed by ',' or EOL
 */
static unsigned char *
kex_agree_instr(unsigned char *haystack, unsigned long haystack_len,
                const unsigned char *needle, unsigned long needle_len)
{
    unsigned char *s;  // 定义指向 unsigned char 类型的指针变量 s
    unsigned char *end_haystack;  // 定义指向 unsigned char 类型的指针变量 end_haystack
    unsigned long left;  // 定义 unsigned long 类型的变量 left

    if(haystack == NULL || needle == NULL) {  // 如果 haystack 或 needle 为空指针
        return NULL;  // 返回空指针
    }

    /* Haystack too short to bother trying */
    if(haystack_len < needle_len || needle_len == 0) {  // 如果 haystack 长度小于 needle 长度或者 needle 长度为 0
        return NULL;  // 返回空指针
    }

    s = haystack;  // 将 haystack 赋值给 s
    end_haystack = &haystack[haystack_len];  // 将 haystack 的地址加上 haystack 长度赋值给 end_haystack
    left = end_haystack - s;  // 计算 left 的值

    /* Needle at start of haystack */
    if((strncmp((char *) haystack, (char *) needle, needle_len) == 0) &&  // 如果 needle 在 haystack 开头
        (needle_len == haystack_len || haystack[needle_len] == ',')) {  // 并且 needle 长度等于 haystack 长度或者 haystack[needle_len] 等于 ','
        return haystack;  // 返回 haystack
    }

    /* Search until we run out of comas or we run out of haystack,
       whichever comes first */
    while((s = (unsigned char *) memchr((char *) s, ',', left))) {  // 在 haystack 中搜索逗号，直到没有逗号或者 haystack 用完
        /* Advance buffer past coma if we can */
        left = end_haystack - s;  // 更新 left 的值
        if((left >= 1) && (left <= haystack_len) && (left > needle_len)) {  // 如果 left 符合条件
            s++;  // s 向前移动一位
        }
        else {
            return NULL;  // 返回空指针
        }

        /* Needle at X position */
        if((strncmp((char *) s, (char *) needle, needle_len) == 0) &&  // 如果 needle 在 X 位置
            (((s - haystack) + needle_len) == haystack_len  // 并且 s 到 haystack 的距离加上 needle 长度等于 haystack 长度
             || s[needle_len] == ',')) {  // 或者 s[needle_len] 等于 ','
            return s;  // 返回 s
        }
    }

    return NULL;  // 返回空指针
}



/* kex_get_method_by_name
 */
static const LIBSSH2_COMMON_METHOD *
kex_get_method_by_name(const char *name, size_t name_len,
                       const LIBSSH2_COMMON_METHOD ** methodlist)
{
    while(*methodlist) {  // 当 methodlist 不为空
        if((strlen((*methodlist)->name) == name_len) &&  // 如果 methodlist 的 name 长度等于 name_len
            (strncmp((*methodlist)->name, name, name_len) == 0)) {  // 并且 methodlist 的 name 与 name 相等
            return *methodlist;  // 返回 methodlist
        }
        methodlist++;  // 指针后移
    }
    return NULL;  // 返回空指针
}
/* kex_agree_hostkey
 * Agree on a Hostkey which works with this kex
 */
static int kex_agree_hostkey(LIBSSH2_SESSION * session,
                             unsigned long kex_flags,
                             unsigned char *hostkey, unsigned long hostkey_len)
{
    const LIBSSH2_HOSTKEY_METHOD **hostkeyp = libssh2_hostkey_methods();
    unsigned char *s;

    // 检查会话中是否存在主机密钥偏好列表
    if(session->hostkey_prefs) {
        s = (unsigned char *) session->hostkey_prefs;

        // 遍历主机密钥偏好列表
        while(s && *s) {
            // 查找逗号分隔的方法名
            unsigned char *p = (unsigned char *) strchr((char *) s, ',');
            size_t method_len = (p ? (size_t)(p - s) : strlen((char *) s));
            // 检查是否有匹配的主机密钥方法
            if(kex_agree_instr(hostkey, hostkey_len, s, method_len)) {
                // 根据方法名获取主机密钥方法
                const LIBSSH2_HOSTKEY_METHOD *method =
                    (const LIBSSH2_HOSTKEY_METHOD *)
                    kex_get_method_by_name((char *) s, method_len,
                                           (const LIBSSH2_COMMON_METHOD **)
                                           hostkeyp);

                if(!method) {
                    /* Invalid method -- Should never be reached */
                    return -1;
                }

                /* So far so good, but does it suit our purposes? (Encrypting
                   vs Signing) */
                // 检查主机密钥方法是否符合加密要求
                if(((kex_flags & LIBSSH2_KEX_METHOD_FLAG_REQ_ENC_HOSTKEY) ==
                     0) || (method->encrypt)) {
                    /* Either this hostkey can do encryption or this kex just
                       doesn't require it */
                    // 检查主机密钥方法是否符合签名要求
                    if(((kex_flags & LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY)
                         == 0) || (method->sig_verify)) {
                        /* Either this hostkey can do signing or this kex just
                           doesn't require it */
                        // 设置会话的主机密钥方法
                        session->hostkey = method;
                        return 0;
                    }
                }
            }

            // 移动到下一个方法名
            s = p ? p + 1 : NULL;
        }
        return -1;
    }
}
    # 当 hostkeyp 不为空且指向的 hostkeyp 的 name 不为空时，执行循环
    while(hostkeyp && (*hostkeyp) && (*hostkeyp)->name) {
        # 调用 kex_agree_instr 函数，协商密钥交换算法
        s = kex_agree_instr(hostkey, hostkey_len,
                            (unsigned char *) (*hostkeyp)->name,
                            strlen((*hostkeyp)->name));
        # 如果协商成功
        if(s) {
            # 检查是否符合我们的目的（加密 vs 签名）
            if(((kex_flags & LIBSSH2_KEX_METHOD_FLAG_REQ_ENC_HOSTKEY) == 0) ||
                ((*hostkeyp)->encrypt)) {
                # 如果主机密钥可以进行加密，或者此 kex 不需要加密
                if(((kex_flags & LIBSSH2_KEX_METHOD_FLAG_REQ_SIGN_HOSTKEY) ==
                     0) || ((*hostkeyp)->sig_verify)) {
                    # 如果主机密钥可以进行签名，或者此 kex 不需要签名
                    # 设置会话的主机密钥为当前 hostkeyp，并返回成功
                    session->hostkey = *hostkeyp;
                    return 0;
                }
            }
        }
        # 指向下一个 hostkeyp
        hostkeyp++;
    }
    # 如果循环结束仍未找到符合条件的主机密钥，则返回失败
    return -1;
/* kex_agree_kex_hostkey
 * Agree on a Key Exchange method and a hostkey encoding type
 */
static int kex_agree_kex_hostkey(LIBSSH2_SESSION * session, unsigned char *kex,
                                 unsigned long kex_len, unsigned char *hostkey,
                                 unsigned long hostkey_len)
{
    const LIBSSH2_KEX_METHOD **kexp = libssh2_kex_methods;
    unsigned char *s;

    // 检查会话中是否存在密钥交换偏好
    if(session->kex_prefs) {
        s = (unsigned char *) session->kex_prefs;

        // 遍历密钥交换偏好列表
        while(s && *s) {
            unsigned char *q, *p = (unsigned char *) strchr((char *) s, ',');
            size_t method_len = (p ? (size_t)(p - s) : strlen((char *) s));
            // 在密钥交换偏好列表中查找与当前密钥交换方法匹配的方法
            q = kex_agree_instr(kex, kex_len, s, method_len);
            if(q) {
                const LIBSSH2_KEX_METHOD *method = (const LIBSSH2_KEX_METHOD *)
                    kex_get_method_by_name((char *) s, method_len,
                                           (const LIBSSH2_COMMON_METHOD **)
                                           kexp);

                if(!method) {
                    /* Invalid method -- Should never be reached */
                    // 无效的方法，不应该到达这里
                    return -1;
                }

                /* We've agreed on a key exchange method,
                 * Can we agree on a hostkey that works with this kex?
                 */
                // 如果找到了适合当前密钥交换方法的主机密钥，则进行协商
                if(kex_agree_hostkey(session, method->flags, hostkey,
                                      hostkey_len) == 0) {
                    session->kex = method;
                    if(session->burn_optimistic_kexinit && (kex == q)) {
                        /* Server sent an optimistic packet, and client agrees
                         * with preference cancel burning the first KEX_INIT
                         * packet that comes in */
                        // 服务器发送了一个乐观的数据包，并且客户端同意取消发送第一个KEX_INIT数据包
                        session->burn_optimistic_kexinit = 0;
                    }
                    return 0;
                }
            }

            s = p ? p + 1 : NULL;
        }
        return -1;
    }
}
    # 当指针指向的表达式不为空且表达式的名称不为空时执行循环
    while(*kexp && (*kexp)->name) {
        # 调用 kex_agree_instr 函数，比较协商的密钥交换方法是否一致
        s = kex_agree_instr(kex, kex_len,
                            (unsigned char *) (*kexp)->name,
                            strlen((*kexp)->name));
        # 如果协商成功
        if(s) {
            # 检查是否可以协商出与密钥交换方法相匹配的主机密钥
            if(kex_agree_hostkey(session, (*kexp)->flags, hostkey,
                                  hostkey_len) == 0) {
                # 将会话的密钥交换方法设置为当前协商成功的方法
                session->kex = *kexp;
                # 如果服务器发送了一个乐观的数据包，并且客户端同意首选项，则取消燃烧第一个 KEX_INIT 数据包
                if(session->burn_optimistic_kexinit && (kex == s)) {
                    session->burn_optimistic_kexinit = 0;
                }
                # 返回成功
                return 0;
            }
        }
        # 指针指向下一个表达式
        kexp++;
    }
    # 返回失败
    return -1;
# kex_agree_crypt 函数，用于协商加密算法
static int kex_agree_crypt(LIBSSH2_SESSION * session,
                           libssh2_endpoint_data *endpoint,
                           unsigned char *crypt,
                           unsigned long crypt_len)
{
    # 获取可用的加密算法列表
    const LIBSSH2_CRYPT_METHOD **cryptp = libssh2_crypt_methods();
    unsigned char *s;

    # 忽略未使用的参数
    (void) session;

    # 如果端点有指定加密算法偏好
    if(endpoint->crypt_prefs) {
        # 获取端点指定的加密算法偏好
        s = (unsigned char *) endpoint->crypt_prefs;

        # 遍历加密算法偏好列表
        while(s && *s) {
            # 查找逗号分隔的加密算法名称
            unsigned char *p = (unsigned char *) strchr((char *) s, ',');
            size_t method_len = (p ? (size_t)(p - s) : strlen((char *) s));

            # 如果找到匹配的加密算法
            if(kex_agree_instr(crypt, crypt_len, s, method_len)) {
                # 获取匹配的加密算法对象
                const LIBSSH2_CRYPT_METHOD *method =
                    (const LIBSSH2_CRYPT_METHOD *)
                    kex_get_method_by_name((char *) s, method_len,
                                           (const LIBSSH2_COMMON_METHOD **)
                                           cryptp);

                # 如果找不到匹配的加密算法对象，返回错误
                if(!method) {
                    /* Invalid method -- Should never be reached */
                    return -1;
                }

                # 设置端点的加密算法
                endpoint->crypt = method;
                return 0;
            }

            # 继续遍历下一个加密算法偏好
            s = p ? p + 1 : NULL;
        }
        return -1;
    }

    # 如果端点没有指定加密算法偏好，则遍历所有可用的加密算法
    while(*cryptp && (*cryptp)->name) {
        # 检查是否有匹配的加密算法
        s = kex_agree_instr(crypt, crypt_len,
                            (unsigned char *) (*cryptp)->name,
                            strlen((*cryptp)->name));
        if(s) {
            # 设置端点的加密算法
            endpoint->crypt = *cryptp;
            return 0;
        }
        # 继续遍历下一个加密算法
        cryptp++;
    }

    # 如果没有匹配的加密算法，返回错误
    return -1;
}

# kex_agree_mac 函数，用于协商消息认证哈希算法
static int kex_agree_mac(LIBSSH2_SESSION * session,
                         libssh2_endpoint_data * endpoint, unsigned char *mac,
                         unsigned long mac_len)
{
    # 获取可用的消息认证哈希算法列表
    const LIBSSH2_MAC_METHOD **macp = _libssh2_mac_methods();
    unsigned char *s;
    # 忽略未使用的参数
    (void) session;
    # 如果端点的 MAC 首选项存在
    if(endpoint->mac_prefs) {
        # 将端点的 MAC 首选项转换为无符号字符指针
        s = (unsigned char *) endpoint->mac_prefs;

        # 当 s 存在且不为空时循环
        while(s && *s) {
            # 查找逗号的位置，计算方法长度
            unsigned char *p = (unsigned char *) strchr((char *) s, ',');
            size_t method_len = (p ? (size_t)(p - s) : strlen((char *) s));

            # 如果协商达成 MAC 方法
            if(kex_agree_instr(mac, mac_len, s, method_len)) {
                # 通过方法名称获取 MAC 方法
                const LIBSSH2_MAC_METHOD *method = (const LIBSSH2_MAC_METHOD *)
                    kex_get_method_by_name((char *) s, method_len,
                                           (const LIBSSH2_COMMON_METHOD **)
                                           macp);

                # 如果方法不存在
                if(!method) {
                    # 无效的方法 -- 不应该到达这里
                    return -1;
                }

                # 设置端点的 MAC 方法并返回
                endpoint->mac = method;
                return 0;
            }

            # 更新 s 的位置
            s = p ? p + 1 : NULL;
        }
        # 返回 -1
        return -1;
    }

    # 当 MAC 方法存在时循环
    while(*macp && (*macp)->name) {
        # 协商达成 MAC 方法
        s = kex_agree_instr(mac, mac_len, (unsigned char *) (*macp)->name,
                            strlen((*macp)->name));
        # 如果协商成功
        if(s) {
            # 设置端点的 MAC 方法并返回
            endpoint->mac = *macp;
            return 0;
        }
        # 更新 MAC 方法指针
        macp++;
    }

    # 返回 -1
    return -1;
# kex_agree_comp 函数，用于协商压缩方案
static int kex_agree_comp(LIBSSH2_SESSION *session,
                          libssh2_endpoint_data *endpoint, unsigned char *comp,
                          unsigned long comp_len)
{
    # 获取会话支持的压缩方法
    const LIBSSH2_COMP_METHOD **compp = _libssh2_comp_methods(session);
    unsigned char *s;
    (void) session;

    # 如果端点有压缩偏好
    if(endpoint->comp_prefs) {
        # 获取端点的压缩偏好
        s = (unsigned char *) endpoint->comp_prefs;

        # 遍历压缩偏好列表
        while(s && *s) {
            # 查找逗号分隔的压缩方法
            unsigned char *p = (unsigned char *) strchr((char *) s, ',');
            size_t method_len = (p ? (size_t)(p - s) : strlen((char *) s));

            # 如果找到匹配的压缩方法
            if(kex_agree_instr(comp, comp_len, s, method_len)) {
                # 获取匹配的压缩方法
                const LIBSSH2_COMP_METHOD *method =
                    (const LIBSSH2_COMP_METHOD *)
                    kex_get_method_by_name((char *) s, method_len,
                                           (const LIBSSH2_COMMON_METHOD **)
                                           compp);

                # 如果找不到匹配的压缩方法
                if(!method) {
                    # 无效的方法 -- 不应该到达这里
                    return -1;
                }

                # 设置端点的压缩方法
                endpoint->comp = method;
                return 0;
            }

            # 更新压缩偏好列表
            s = p ? p + 1 : NULL;
        }
        return -1;
    }

    # 如果端点没有压缩偏好，使用会话支持的压缩方法
    while(*compp && (*compp)->name) {
        s = kex_agree_instr(comp, comp_len, (unsigned char *) (*compp)->name,
                            strlen((*compp)->name));
        if(s) {
            endpoint->comp = *compp;
            return 0;
        }
        compp++;
    }

    return -1;
}


# TODO: 当处于服务器模式时，需要颠倒这个逻辑
# 客户端最终决定“协商方法”
# kex_agree_methods 函数，决定每个参与方提供的方法中使用哪个具体方法
static int kex_agree_methods(LIBSSH2_SESSION * session, unsigned char *data,
                             unsigned data_len)
{
    # 定义指向各种密钥和数据的指针，以及它们的长度
    unsigned char *kex, *hostkey, *crypt_cs, *crypt_sc, *comp_cs, *comp_sc,
        *mac_cs, *mac_sc;
    size_t kex_len, hostkey_len, crypt_cs_len, crypt_sc_len, comp_cs_len;
    size_t comp_sc_len, mac_cs_len, mac_sc_len;
    struct string_buf buf;

    # 如果数据长度小于17，则返回-1
    if(data_len < 17)
        return -1;

    # 将数据指针指向数据的起始位置
    buf.data = (unsigned char *)data;
    buf.len = data_len;
    buf.dataptr = buf.data;
    buf.dataptr++; /* advance past packet type */

    # 跳过16个字节的cookie，不用担心，它在kexinit字段中保留
    buf.dataptr += 16;

    # 定位每个字符串并将其存储到相应的指针中
    if(_libssh2_get_string(&buf, &kex, &kex_len))
        return -1;
    if(_libssh2_get_string(&buf, &hostkey, &hostkey_len))
        return -1;
    if(_libssh2_get_string(&buf, &crypt_cs, &crypt_cs_len))
        return -1;
    if(_libssh2_get_string(&buf, &crypt_sc, &crypt_sc_len))
        return -1;
    if(_libssh2_get_string(&buf, &mac_cs, &mac_cs_len))
        return -1;
    if(_libssh2_get_string(&buf, &mac_sc, &mac_sc_len))
        return -1;
    if(_libssh2_get_string(&buf, &comp_cs, &comp_cs_len))
        return -1;
    if(_libssh2_get_string(&buf, &comp_sc, &comp_sc_len))
        return -1;

    # 如果服务器发送了一个乐观的数据包，假设它猜错了
    # 如果猜测被确定为正确（由kex_agree_kex_hostkey确定）
    # 则将此标志重置为零，以便不被忽略
    if(_libssh2_check_length(&buf, 1)) {
        session->burn_optimistic_kexinit = *(buf.dataptr++);
    }
    else {
        return -1;
    }

    # 下一个数据包中的uint32都是零（保留）

    # 如果协商密钥和主机密钥失败，则返回-1
    if(kex_agree_kex_hostkey(session, kex, kex_len, hostkey, hostkey_len)) {
        return -1;
    }

    # 如果协商加密算法失败，则返回-1
    if(kex_agree_crypt(session, &session->local, crypt_cs, crypt_cs_len)
       || kex_agree_crypt(session, &session->remote, crypt_sc,
                          crypt_sc_len)) {
        return -1;
    }
    # 如果本地和远程会话都同意使用给定的消息认证码算法，则返回-1
    if(kex_agree_mac(session, &session->local, mac_cs, mac_cs_len) ||
        kex_agree_mac(session, &session->remote, mac_sc, mac_sc_len)) {
        return -1;
    }

    # 如果本地和远程会话都同意使用给定的压缩算法，则返回-1
    if(kex_agree_comp(session, &session->local, comp_cs, comp_cs_len) ||
        kex_agree_comp(session, &session->remote, comp_sc, comp_sc_len)) {
        return -1;
    }
#if 0
    // 如果 libssh2_kex_agree_lang 函数返回非零，表示协商失败，返回-1
    if(libssh2_kex_agree_lang(session, &session->local, lang_cs, lang_cs_len)
        || libssh2_kex_agree_lang(session, &session->remote, lang_sc,
                                  lang_sc_len)) {
        return -1;
    }
#endif

    // 输出协商成功的密钥交换方法
    _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Agreed on KEX method: %s",
                   session->kex->name);
    // 输出协商成功的主机密钥方法
    _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Agreed on HOSTKEY method: %s",
                   session->hostkey->name);
    // 输出协商成功的加密算法（客户端到服务器）
    _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Agreed on CRYPT_CS method: %s",
                   session->local.crypt->name);
    // 输出协商成功的加密算法（服务器到客户端）
    _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Agreed on CRYPT_SC method: %s",
                   session->remote.crypt->name);
    // 输出协商成功的消息认证码算法（客户端到服务器）
    _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Agreed on MAC_CS method: %s",
                   session->local.mac->name);
    // 输出协商成功的消息认证码算法（服务器到客户端）
    _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Agreed on MAC_SC method: %s",
                   session->remote.mac->name);
    // 输出协商成功的压缩算法（客户端到服务器）
    _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Agreed on COMP_CS method: %s",
                   session->local.comp->name);
    // 输出协商成功的压缩算法（服务器到客户端）
    _libssh2_debug(session, LIBSSH2_TRACE_KEX, "Agreed on COMP_SC method: %s",
                   session->remote.comp->name);

    // 返回成功
    return 0;
}



/* _libssh2_kex_exchange
 * Exchange keys
 * Returns 0 on success, non-zero on failure
 *
 * Returns some errors without _libssh2_error()
 */
// 密钥交换函数，返回0表示成功，非零表示失败
int
_libssh2_kex_exchange(LIBSSH2_SESSION * session, int reexchange,
                      key_exchange_state_t * key_state)
{
    int rc = 0;
    int retcode;

    // 设置会话状态为密钥交换活跃状态
    session->state |= LIBSSH2_STATE_KEX_ACTIVE;
    # 如果 key_state 的状态为 libssh2_NB_state_idle
    if(key_state->state == libssh2_NB_state_idle) {
        # 防止在 packet_add() 中出现循环
        session->state |= LIBSSH2_STATE_EXCHANGING_KEYS;

        # 如果需要重新交换密钥
        if(reexchange) {
            # 重置会话的密钥交换对象
            session->kex = NULL;

            # 如果会话的主机密钥存在且有析构函数
            if(session->hostkey && session->hostkey->dtor) {
                # 调用主机密钥的析构函数
                session->hostkey->dtor(session, &session->server_hostkey_abstract);
            }
            # 重置会话的主机密钥对象
            session->hostkey = NULL;
        }

        # 将 key_state 的状态设置为 libssh2_NB_state_created
        key_state->state = libssh2_NB_state_created;
    }
    # 如果 key_state 的状态不为 libssh2_NB_state_idle
    else {
        # 将 key_state 的状态设置为 libssh2_NB_state_sent2
        key_state->state = libssh2_NB_state_sent2;
    }

    # 如果 rc 为 0 且会话的密钥交换对象存在
    if(rc == 0 && session->kex) {
        # 如果 key_state 的状态为 libssh2_NB_state_sent2
        if(key_state->state == libssh2_NB_state_sent2) {
            # 调用会话的密钥交换对象的 exchange_keys 方法
            retcode = session->kex->exchange_keys(session, &key_state->key_state_low);
            # 如果返回值为 LIBSSH2_ERROR_EAGAIN
            if(retcode == LIBSSH2_ERROR_EAGAIN) {
                # 清除会话的状态中的 LIBSSH2_STATE_KEX_ACTIVE 标志位
                session->state &= ~LIBSSH2_STATE_KEX_ACTIVE;
                # 返回 retcode
                return retcode;
            }
            # 如果返回值不为 0
            else if(retcode) {
                # 设置 rc 为 LIBSSH2_ERROR_KEY_EXCHANGE_FAILURE
                rc = _libssh2_error(session, LIBSSH2_ERROR_KEY_EXCHANGE_FAILURE, "Unrecoverable error exchanging keys");
            }
        }
    }

    # 释放会话的本地 kexinit 缓冲区
    if(session->local.kexinit) {
        LIBSSH2_FREE(session, session->local.kexinit);
        session->local.kexinit = NULL;
    }
    # 释放会话的远程 kexinit 缓冲区
    if(session->remote.kexinit) {
        LIBSSH2_FREE(session, session->remote.kexinit);
        session->remote.kexinit = NULL;
    }

    # 清除会话的状态中的 LIBSSH2_STATE_KEX_ACTIVE 和 LIBSSH2_STATE_EXCHANGING_KEYS 标志位
    session->state &= ~LIBSSH2_STATE_KEX_ACTIVE;
    session->state &= ~LIBSSH2_STATE_EXCHANGING_KEYS;

    # 将 key_state 的状态设置为 libssh2_NB_state_idle
    key_state->state = libssh2_NB_state_idle;

    # 返回 rc
    return rc;
/* libssh2_session_method_pref
 * 设置首选方法
 */
LIBSSH2_API int
libssh2_session_method_pref(LIBSSH2_SESSION * session, int method_type,
                            const char *prefs)
{
    char **prefvar, *s, *newprefs;
    int prefs_len = strlen(prefs);
    const LIBSSH2_COMMON_METHOD **mlist;

    switch(method_type) {
    case LIBSSH2_METHOD_KEX:
        prefvar = &session->kex_prefs;
        mlist = (const LIBSSH2_COMMON_METHOD **) libssh2_kex_methods;
        break;

    case LIBSSH2_METHOD_HOSTKEY:
        prefvar = &session->hostkey_prefs;
        mlist = (const LIBSSH2_COMMON_METHOD **) libssh2_hostkey_methods();
        break;

    case LIBSSH2_METHOD_CRYPT_CS:
        prefvar = &session->local.crypt_prefs;
        mlist = (const LIBSSH2_COMMON_METHOD **) libssh2_crypt_methods();
        break;

    case LIBSSH2_METHOD_CRYPT_SC:
        prefvar = &session->remote.crypt_prefs;
        mlist = (const LIBSSH2_COMMON_METHOD **) libssh2_crypt_methods();
        break;

    case LIBSSH2_METHOD_MAC_CS:
        prefvar = &session->local.mac_prefs;
        mlist = (const LIBSSH2_COMMON_METHOD **) _libssh2_mac_methods();
        break;

    case LIBSSH2_METHOD_MAC_SC:
        prefvar = &session->remote.mac_prefs;
        mlist = (const LIBSSH2_COMMON_METHOD **) _libssh2_mac_methods();
        break;

    case LIBSSH2_METHOD_COMP_CS:
        prefvar = &session->local.comp_prefs;
        mlist = (const LIBSSH2_COMMON_METHOD **)
            _libssh2_comp_methods(session);
        break;

    case LIBSSH2_METHOD_COMP_SC:
        prefvar = &session->remote.comp_prefs;
        mlist = (const LIBSSH2_COMMON_METHOD **)
            _libssh2_comp_methods(session);
        break;

    case LIBSSH2_METHOD_LANG_CS:
        prefvar = &session->local.lang_prefs;
        mlist = NULL;
        break;

    case LIBSSH2_METHOD_LANG_SC:
        prefvar = &session->remote.lang_prefs;
        mlist = NULL;
        break;
    # 默认情况下，返回无效参数指定的方法类型的错误
    default:
        return _libssh2_error(session, LIBSSH2_ERROR_INVAL,
                              "Invalid parameter specified for method_type");
    }

    # 为方法偏好分配内存空间
    s = newprefs = LIBSSH2_ALLOC(session, prefs_len + 1);
    if(!newprefs) {
        # 如果无法分配内存空间，则返回分配空间给方法偏好时出错的错误
        return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                              "Error allocated space for method preferences");
    }
    # 将方法偏好复制到新分配的内存空间中
    memcpy(s, prefs, prefs_len + 1);

    # 遍历方法偏好列表，检查每个方法是否被支持
    while(s && *s && mlist) {
        # 查找逗号分隔的方法名称
        char *p = strchr(s, ',');
        int method_len = p ? (p - s) : (int) strlen(s);

        # 如果方法不被支持，则从偏好列表中删除该方法
        if(!kex_get_method_by_name(s, method_len, mlist)) {
            # 删除不支持的方法
            if(p) {
                memcpy(s, p + 1, strlen(s) - method_len);
            }
            else {
                if(s > newprefs) {
                    *(--s) = '\0';
                }
                else {
                    *s = '\0';
                }
            }
        }
        else {
            # 如果方法被支持，则继续检查下一个方法
            s = p ? (p + 1) : NULL;
        }
    }

    # 如果偏好列表为空，则释放内存空间并返回请求的方法当前不被支持的错误
    if(!*newprefs) {
        LIBSSH2_FREE(session, newprefs);
        return _libssh2_error(session, LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                              "The requested method(s) are not currently "
                              "supported");
    }

    # 如果原始偏好列表不为空，则释放其内存空间
    if(*prefvar) {
        LIBSSH2_FREE(session, *prefvar);
    }
    # 将新的偏好列表赋值给原始偏好列表
    *prefvar = newprefs;

    # 返回成功
    return 0;
/*
 * libssh2_session_supported_algs()
 * 在成功时返回返回的算法数量（正数），失败时返回负数
 */

LIBSSH2_API int libssh2_session_supported_algs(LIBSSH2_SESSION* session,
                                               int method_type,
                                               const char ***algs)
{
    unsigned int i;
    unsigned int j;
    unsigned int ialg;
    const LIBSSH2_COMMON_METHOD **mlist;

    /* 防止由于对 NULL 的解引用而导致核心转储 */
    if(NULL == algs)
        return _libssh2_error(session, LIBSSH2_ERROR_BAD_USE,
                              "algs must not be NULL");

    switch(method_type) {
    case LIBSSH2_METHOD_KEX:
        mlist = (const LIBSSH2_COMMON_METHOD **) libssh2_kex_methods;
        break;

    case LIBSSH2_METHOD_HOSTKEY:
        mlist = (const LIBSSH2_COMMON_METHOD **) libssh2_hostkey_methods();
        break;

    case LIBSSH2_METHOD_CRYPT_CS:
    case LIBSSH2_METHOD_CRYPT_SC:
        mlist = (const LIBSSH2_COMMON_METHOD **) libssh2_crypt_methods();
        break;

    case LIBSSH2_METHOD_MAC_CS:
    case LIBSSH2_METHOD_MAC_SC:
        mlist = (const LIBSSH2_COMMON_METHOD **) _libssh2_mac_methods();
        break;

    case LIBSSH2_METHOD_COMP_CS:
    case LIBSSH2_METHOD_COMP_SC:
        mlist = (const LIBSSH2_COMMON_METHOD **)
            _libssh2_comp_methods(session);
        break;

    default:
        return _libssh2_error(session, LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                              "Unknown method type");
    }  /* switch */

    /* 奇怪的情况 */
    if(NULL == mlist)
        return _libssh2_error(session, LIBSSH2_ERROR_INVAL,
                              "No algorithm found");
}
    /*
      mlist is looped through twice. The first time to find the number od
      supported algorithms (needed to allocate the proper size of array) and
      the second time to actually copy the pointers.  Typically this function
      will not be called often (typically at the beginning of a session) and
      the number of algorithms (i.e. number of iterations in one loop) will
      not be high (typically it will not exceed 20) for quite a long time.

      So double looping really shouldn't be an issue and it is definitely a
      better solution than reallocation several times.
    */

    /* count the number of supported algorithms */
    for(i = 0, ialg = 0; NULL != mlist[i]; i++) {
        /* do not count fields with NULL name */
        if(mlist[i]->name)
            ialg++;
    }

    /* weird situation, no algorithm found */
    if(0 == ialg)
        return _libssh2_error(session, LIBSSH2_ERROR_INVAL,
                              "No algorithm found");

    /* allocate buffer */
    *algs = (const char **) LIBSSH2_ALLOC(session, ialg*sizeof(const char *));
    if(NULL == *algs) {
        return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                              "Memory allocation failed");
    }
    /* Past this point *algs must be deallocated in case of an error!! */

    /* copy non-NULL pointers only */
    for(i = 0, j = 0; NULL != mlist[i] && j < ialg; i++) {
        if(NULL == mlist[i]->name) {
            /* maybe a weird situation but if it occurs, do not include NULL
               pointers */
            continue;
        }

        /* note that [] has higher priority than * (dereferencing) */
        (*algs)[j++] = mlist[i]->name;
    }

    /* correct number of pointers copied? (test the code above) */
    # 如果 j 不等于 ialg，则执行以下操作
    if(j != ialg) {
        # 释放缓冲区
        LIBSSH2_FREE(session, (void *)*algs);
        # 将指针 algs 指向的内存设置为 NULL
        *algs = NULL;

        # 返回内部错误，并终止程序
        return _libssh2_error(session, LIBSSH2_ERROR_BAD_USE,
                              "Internal error");
    }

    # 返回 ialg 的值
    return ialg;
# 闭合前面的函数定义
```