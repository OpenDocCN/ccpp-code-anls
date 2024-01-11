# `xmrig\src\3rdparty\argon2\lib\argon2.c`

```
/*
 * Argon2源代码包
 *
 * 由Daniel Dinu和Dmitry Khovratovich编写，2015年
 *
 * 本作品根据知识共享CC0 1.0许可证/豁免授权发布。
 *
 * 您应该已经收到了CC0公共领域奉献的副本
 * 这个软件。如果没有，请参见
 * <http://creativecommons.org/publicdomain/zero/1.0/>。
 */

#include <string.h>
#include <stdlib.h>
#include <stdio.h>

#include "3rdparty/argon2.h"
#include "encoding.h"
#include "core.h"

// 将Argon2类型转换为字符串
const char *argon2_type2string(argon2_type type, int uppercase) {
    switch (type) {
        case Argon2_d:
            return uppercase ? "Argon2d" : "argon2d";
        case Argon2_i:
            return uppercase ? "Argon2i" : "argon2i";
        case Argon2_id:
            return uppercase ? "Argon2id" : "argon2id";
    }

    return NULL;
}

// 计算内存块和段长度
static void argon2_compute_memory_blocks(uint32_t *memory_blocks,
                                         uint32_t *segment_length,
                                         uint32_t m_cost, uint32_t lanes)
{
    /* 最小内存块 = 8L块，其中L是通道数 */
    *memory_blocks = m_cost;
    if (*memory_blocks < 2 * ARGON2_SYNC_POINTS * lanes) {
        *memory_blocks = 2 * ARGON2_SYNC_POINTS * lanes;
    }

    *segment_length = *memory_blocks / (lanes * ARGON2_SYNC_POINTS);
    /* 确保所有段具有相等的长度 */
    *memory_blocks = *segment_length * (lanes * ARGON2_SYNC_POINTS);
}

// 计算Argon2的内存大小
size_t argon2_memory_size(uint32_t m_cost, uint32_t parallelism) {
    uint32_t memory_blocks, segment_length;
    argon2_compute_memory_blocks(&memory_blocks, &segment_length, m_cost,
                                 parallelism);
    return memory_blocks * ARGON2_BLOCK_SIZE;
}

// 分配Argon2上下文的内存
int argon2_ctx_mem(argon2_context *context, argon2_type type, void *memory,
                   size_t memory_size) {
    /* 1. 验证所有输入 */
    int result = xmrig_ar2_validate_inputs(context);
    uint32_t memory_blocks, segment_length;
    // 声明一个名为instance的argon2_instance_t类型的变量

    // 如果result不等于ARGON2_OK，则返回result
    if (ARGON2_OK != result) {
        return result;
    }

    // 如果type不是Argon2_d、Argon2_i和Argon2_id中的任何一个，则返回ARGON2_INCORRECT_TYPE

    /* 2. 对齐内存大小 */
    // 计算内存块和段长度
    argon2_compute_memory_blocks(&memory_blocks, &segment_length,
                                 context->m_cost, context->lanes);

    // 检查内存大小是否足够：
    if (memory != NULL && (memory_size % ARGON2_BLOCK_SIZE != 0 ||
                           memory_size / ARGON2_BLOCK_SIZE < memory_blocks)) {
        return ARGON2_MEMORY_ALLOCATION_ERROR;
    }

    // 设置instance的各个属性
    instance.version = context->version;
    instance.memory = (block *)memory;
    instance.passes = context->t_cost;
    instance.memory_blocks = memory_blocks;
    instance.segment_length = segment_length;
    instance.lane_length = segment_length * ARGON2_SYNC_POINTS;
    instance.lanes = context->lanes;
    instance.threads = context->threads;
    instance.type = type;
    instance.print_internals = !!(context->flags & ARGON2_FLAG_GENKAT);
    instance.keep_memory = memory != NULL;

    // 如果instance的线程数大于实例的lanes数，则将线程数设置为lanes数
    if (instance.threads > instance.lanes) {
        instance.threads = instance.lanes;
    }

    /* 3. 初始化：哈希输入，分配内存，填充第一个块 */
    // 初始化：哈希输入，分配内存，填充第一个块
    result = xmrig_ar2_initialize(&instance, context);

    // 如果result不等于ARGON2_OK，则返回result
    if (ARGON2_OK != result) {
        return result;
    }

    /* 4. 填充内存 */
    // 填充内存
    result = xmrig_ar2_fill_memory_blocks(&instance);

    // 如果result不等于ARGON2_OK，则返回result
    if (ARGON2_OK != result) {
        return result;
    }
    /* 5. 完成 */
    // 完成最后的处理
    xmrig_ar2_finalize(context, &instance);

    // 返回ARGON2_OK
    return ARGON2_OK;
}

int argon2_ctx(argon2_context *context, argon2_type type) {
    return argon2_ctx_mem(context, type, NULL, 0);
}

int argon2_hash(const uint32_t t_cost, const uint32_t m_cost,
                const uint32_t parallelism, const void *pwd,
                const size_t pwdlen, const void *salt, const size_t saltlen,
                void *hash, const size_t hashlen, char *encoded,
                const size_t encodedlen, argon2_type type,
                const uint32_t version){

    argon2_context context;  // 定义 argon2 上下文
    int result;  // 定义结果变量
    uint8_t *out;  // 定义输出变量

    if (pwdlen > ARGON2_MAX_PWD_LENGTH) {  // 如果密码长度超过最大限制
        return ARGON2_PWD_TOO_LONG;  // 返回密码过长错误
    }

    if (saltlen > ARGON2_MAX_SALT_LENGTH) {  // 如果盐值长度超过最大限制
        return ARGON2_SALT_TOO_LONG;  // 返回盐值过长错误
    }

    if (hashlen > ARGON2_MAX_OUTLEN) {  // 如果哈希值长度超过最大限制
        return ARGON2_OUTPUT_TOO_LONG;  // 返回哈希值过长错误
    }

    if (hashlen < ARGON2_MIN_OUTLEN) {  // 如果哈希值长度小于最小限制
        return ARGON2_OUTPUT_TOO_SHORT;  // 返回哈希值过短错误
    }

    out = malloc(hashlen);  // 分配哈希值长度大小的内存空间
    if (!out) {  // 如果内存分配失败
        return ARGON2_MEMORY_ALLOCATION_ERROR;  // 返回内存分配错误
    }

    context.out = (uint8_t *)out;  // 设置输出指针
    context.outlen = (uint32_t)hashlen;  // 设置输出长度
    context.pwd = CONST_CAST(uint8_t *)pwd;  // 设置密码
    context.pwdlen = (uint32_t)pwdlen;  // 设置密码长度
    context.salt = CONST_CAST(uint8_t *)salt;  // 设置盐值
    context.saltlen = (uint32_t)saltlen;  // 设置盐值长度
    context.secret = NULL;  // 设置密钥为空
    context.secretlen = 0;  // 设置密钥长度为0
    context.ad = NULL;  // 设置附加数据为空
    context.adlen = 0;  // 设置附加数据长度为0
    context.t_cost = t_cost;  // 设置时间成本
    context.m_cost = m_cost;  // 设置内存成本
    context.lanes = parallelism;  // 设置并行度
    context.threads = parallelism;  // 设置线程数
    context.allocate_cbk = NULL;  // 设置分配回调为空
    context.free_cbk = NULL;  // 设置释放回调为空
    context.flags = ARGON2_DEFAULT_FLAGS;  // 设置标志
    context.version = version;  // 设置版本号

    result = argon2_ctx(&context, type);  // 调用 argon2 上下文函数

    if (result != ARGON2_OK) {  // 如果结果不是成功
        xmrig_ar2_clear_internal_memory(out, hashlen);  // 清除内存
        free(out);  // 释放内存
        return result;  // 返回结果
    }

    /* if raw hash requested, write it */
    if (hash) {  // 如果请求原始哈希值
        memcpy(hash, out, hashlen);  // 复制哈希值
    }

    /* if encoding requested, write it */
}
    # 检查是否存在编码后的数据，并且长度不为0
    if (encoded && encodedlen) {
        # 如果编码字符串失败，则清空内存并返回编码失败
        if (encode_string(encoded, encodedlen, &context, type) != ARGON2_OK) {
            xmrig_ar2_clear_internal_memory(out, hashlen); /* wipe buffers if error */
            xmrig_ar2_clear_internal_memory(encoded, encodedlen);
            free(out);
            return ARGON2_ENCODING_FAIL;
        }
    }
    # 清空内存
    xmrig_ar2_clear_internal_memory(out, hashlen);
    # 释放内存
    free(out);

    # 返回成功
    return ARGON2_OK;
// 使用 Argon2i 算法对输入进行哈希，并返回编码后的哈希值
int argon2i_hash_encoded(const uint32_t t_cost, const uint32_t m_cost,
                         const uint32_t parallelism, const void *pwd,
                         const size_t pwdlen, const void *salt,
                         const size_t saltlen, const size_t hashlen,
                         char *encoded, const size_t encodedlen) {

    return argon2_hash(t_cost, m_cost, parallelism, pwd, pwdlen, salt, saltlen,
                       NULL, hashlen, encoded, encodedlen, Argon2_i,
                       ARGON2_VERSION_NUMBER);
}

// 使用 Argon2i 算法对输入进行哈希，并返回原始的哈希值
int argon2i_hash_raw(const uint32_t t_cost, const uint32_t m_cost,
                     const uint32_t parallelism, const void *pwd,
                     const size_t pwdlen, const void *salt,
                     const size_t saltlen, void *hash, const size_t hashlen) {

    return argon2_hash(t_cost, m_cost, parallelism, pwd, pwdlen, salt, saltlen,
                       hash, hashlen, NULL, 0, Argon2_i, ARGON2_VERSION_NUMBER);
}

// 使用 Argon2d 算法对输入进行哈希，并返回编码后的哈希值
int argon2d_hash_encoded(const uint32_t t_cost, const uint32_t m_cost,
                         const uint32_t parallelism, const void *pwd,
                         const size_t pwdlen, const void *salt,
                         const size_t saltlen, const size_t hashlen,
                         char *encoded, const size_t encodedlen) {

    return argon2_hash(t_cost, m_cost, parallelism, pwd, pwdlen, salt, saltlen,
                       NULL, hashlen, encoded, encodedlen, Argon2_d,
                       ARGON2_VERSION_NUMBER);
}

// 使用 Argon2d 算法对输入进行哈希，并返回原始的哈希值
int argon2d_hash_raw(const uint32_t t_cost, const uint32_t m_cost,
                     const uint32_t parallelism, const void *pwd,
                     const size_t pwdlen, const void *salt,
                     const size_t saltlen, void *hash, const size_t hashlen) {

    return argon2_hash(t_cost, m_cost, parallelism, pwd, pwdlen, salt, saltlen,
                       hash, hashlen, NULL, 0, Argon2_d, ARGON2_VERSION_NUMBER);
}
// 使用 Argon2id 算法对输入的密码进行哈希，并将结果以编码形式返回
int argon2id_hash_encoded(const uint32_t t_cost, const uint32_t m_cost,
                          const uint32_t parallelism, const void *pwd,
                          const size_t pwdlen, const void *salt,
                          const size_t saltlen, const size_t hashlen,
                          char *encoded, const size_t encodedlen) {

    return argon2_hash(t_cost, m_cost, parallelism, pwd, pwdlen, salt, saltlen,
                       NULL, hashlen, encoded, encodedlen, Argon2_id,
                       ARGON2_VERSION_NUMBER);
}

// 使用 Argon2id 算法对输入的密码进行哈希，并将原始哈希值返回
int argon2id_hash_raw(const uint32_t t_cost, const uint32_t m_cost,
                      const uint32_t parallelism, const void *pwd,
                      const size_t pwdlen, const void *salt,
                      const size_t saltlen, void *hash, const size_t hashlen) {
    return argon2_hash(t_cost, m_cost, parallelism, pwd, pwdlen, salt, saltlen,
                       hash, hashlen, NULL, 0, Argon2_id,
                       ARGON2_VERSION_NUMBER);
}

// 使用 Argon2id 算法对输入的密码进行哈希，并将原始哈希值返回，同时指定内存分配方式
int argon2id_hash_raw_ex(const uint32_t t_cost, const uint32_t m_cost,
                         const uint32_t parallelism, const void *pwd,
                         const size_t pwdlen, const void *salt,
                         const size_t saltlen, void *hash, const size_t hashlen, void *memory) {
    // 创建 Argon2 上下文
    argon2_context context;

    // 设置上下文中的输出哈希值和长度
    context.out = (uint8_t *)hash;
    context.outlen = (uint32_t)hashlen;
    // 设置上下文中的密码和长度
    context.pwd = CONST_CAST(uint8_t *)pwd;
    context.pwdlen = (uint32_t)pwdlen;
    // 设置上下文中的盐和长度
    context.salt = CONST_CAST(uint8_t *)salt;
    context.saltlen = (uint32_t)saltlen;
    // 设置上下文中的其他参数
    context.secret = NULL;
    context.secretlen = 0;
    context.ad = NULL;
    context.adlen = 0;
    context.t_cost = t_cost;
    context.m_cost = m_cost;
    context.lanes = parallelism;
    context.threads = parallelism;
    context.allocate_cbk = NULL;
    context.free_cbk = NULL;
    context.flags = ARGON2_DEFAULT_FLAGS;
    context.version = ARGON2_VERSION_NUMBER;
    # 调用 argon2_ctx_mem 函数，传入参数 context, Argon2_id, memory, m_cost * 1024，并返回结果
    return argon2_ctx_mem(&context, Argon2_id, memory, m_cost * 1024);
/* 
   比较两个字节数组的内容是否相等
   如果相等返回0，否则返回非0值
*/
static int argon2_compare(const uint8_t *b1, const uint8_t *b2, size_t len) {
    size_t i;
    uint8_t d = 0U;

    for (i = 0U; i < len; i++) {
        d |= b1[i] ^ b2[i];  // 逐个比较两个字节数组的内容
    }
    return (int)((1 & ((d - 1) >> 8)) - 1);  // 返回比较结果
}

/* 
   验证密码是否匹配给定的编码
   如果匹配返回0，否则返回错误码
*/
int argon2_verify(const char *encoded, const void *pwd, const size_t pwdlen,
                  argon2_type type) {

    argon2_context ctx;
    uint8_t *desired_result = NULL;

    int ret = ARGON2_OK;

    size_t encoded_len;
    uint32_t max_field_len;

    if (pwdlen > ARGON2_MAX_PWD_LENGTH) {
        return ARGON2_PWD_TOO_LONG;  // 密码长度过长
    }

    if (encoded == NULL) {
        return ARGON2_DECODING_FAIL;  // 编码为空
    }

    encoded_len = strlen(encoded);
    if (encoded_len > UINT32_MAX) {
        return ARGON2_DECODING_FAIL;  // 编码长度过长
    }

    /* No field can be longer than the encoded length */
    max_field_len = (uint32_t)encoded_len;

    ctx.saltlen = max_field_len;
    ctx.outlen = max_field_len;

    ctx.salt = malloc(ctx.saltlen);  // 分配盐的内存空间
    ctx.out = malloc(ctx.outlen);  // 分配输出的内存空间
    if (!ctx.salt || !ctx.out) {
        ret = ARGON2_MEMORY_ALLOCATION_ERROR;  // 内存分配错误
        goto fail;
    }

    ctx.pwd = (uint8_t *)pwd;
    ctx.pwdlen = (uint32_t)pwdlen;

    ret = decode_string(&ctx, encoded, type);  // 解码编码字符串
    if (ret != ARGON2_OK) {
        goto fail;
    }

    /* Set aside the desired result, and get a new buffer. */
    desired_result = ctx.out;  // 保存期望的结果
    ctx.out = malloc(ctx.outlen);  // 分配新的输出缓冲区
    if (!ctx.out) {
        ret = ARGON2_MEMORY_ALLOCATION_ERROR;  // 内存分配错误
        goto fail;
    }

    ret = argon2_verify_ctx(&ctx, (char *)desired_result, type);  // 验证密码是否匹配
    if (ret != ARGON2_OK) {
        goto fail;
    }

fail:
    free(ctx.salt);  // 释放盐的内存空间
    free(ctx.out);  // 释放输出的内存空间
    free(desired_result);  // 释放期望的结果的内存空间

    return ret;  // 返回验证结果
}

/* 
   验证密码是否匹配给定的编码（argon2i类型）
*/
int argon2i_verify(const char *encoded, const void *pwd, const size_t pwdlen) {

    return argon2_verify(encoded, pwd, pwdlen, Argon2_i);  // 调用argon2_verify函数
}

/* 
   验证密码是否匹配给定的编码（argon2d类型）
*/
int argon2d_verify(const char *encoded, const void *pwd, const size_t pwdlen) {

    return argon2_verify(encoded, pwd, pwdlen, Argon2_d);  // 调用argon2_verify函数
}
# 验证 argon2id 加密的密码是否匹配
int argon2id_verify(const char *encoded, const void *pwd, const size_t pwdlen) {
    return argon2_verify(encoded, pwd, pwdlen, Argon2_id);
}

# 使用 argon2d 加密算法对密码进行加密
int argon2d_ctx(argon2_context *context) {
    return argon2_ctx(context, Argon2_d);
}

# 使用 argon2i 加密算法对密码进行加密
int argon2i_ctx(argon2_context *context) {
    return argon2_ctx(context, Argon2_i);
}

# 使用 argon2id 加密算法对密码进行加密
int argon2id_ctx(argon2_context *context) {
    return argon2_ctx(context, Argon2_id);
}

# 验证 argon2 加密算法加密的密码是否匹配
int argon2_verify_ctx(argon2_context *context, const char *hash, argon2_type type) {
    # 调用 argon2_ctx 函数对密码进行加密
    int ret = argon2_ctx(context, type);
    if (ret != ARGON2_OK) {
        return ret;
    }

    # 比较加密后的密码和原始密码是否匹配
    if (argon2_compare((uint8_t *)hash, context->out, context->outlen)) {
        return ARGON2_VERIFY_MISMATCH;
    }

    return ARGON2_OK;
}

# 使用 argon2d 加密算法验证密码是否匹配
int argon2d_verify_ctx(argon2_context *context, const char *hash) {
    return argon2_verify_ctx(context, hash, Argon2_d);
}

# 使用 argon2i 加密算法验证密码是否匹配
int argon2i_verify_ctx(argon2_context *context, const char *hash) {
    return argon2_verify_ctx(context, hash, Argon2_i);
}

# 使用 argon2id 加密算法验证密码是否匹配
int argon2id_verify_ctx(argon2_context *context, const char *hash) {
    return argon2_verify_ctx(context, hash, Argon2_id);
}

# 根据错误代码返回相应的错误信息
const char *argon2_error_message(int error_code) {
    switch (error_code) {
    case ARGON2_OK:
        return "OK";
    case ARGON2_OUTPUT_PTR_NULL:
        return "Output pointer is NULL";
    case ARGON2_OUTPUT_TOO_SHORT:
        return "Output is too short";
    case ARGON2_OUTPUT_TOO_LONG:
        return "Output is too long";
    case ARGON2_PWD_TOO_SHORT:
        return "Password is too short";
    case ARGON2_PWD_TOO_LONG:
        return "Password is too long";
    case ARGON2_SALT_TOO_SHORT:
        return "Salt is too short";
    case ARGON2_SALT_TOO_LONG:
        return "Salt is too long";
    case ARGON2_AD_TOO_SHORT:
        return "Associated data is too short";
    case ARGON2_AD_TOO_LONG:
        return "Associated data is too long";
    case ARGON2_SECRET_TOO_SHORT:
        return "Secret is too short";
    # 如果密钥过长，则返回"Secret is too long"
    case ARGON2_SECRET_TOO_LONG:
        return "Secret is too long";
    # 如果时间成本太小，则返回"Time cost is too small"
    case ARGON2_TIME_TOO_SMALL:
        return "Time cost is too small";
    # 如果时间成本太大，则返回"Time cost is too large"
    case ARGON2_TIME_TOO_LARGE:
        return "Time cost is too large";
    # 如果内存成本太小，则返回"Memory cost is too small"
    case ARGON2_MEMORY_TOO_LITTLE:
        return "Memory cost is too small";
    # 如果内存成本太大，则返回"Memory cost is too large"
    case ARGON2_MEMORY_TOO_MUCH:
        return "Memory cost is too large";
    # 如果并行处理线程数太少，则返回"Too few lanes"
    case ARGON2_LANES_TOO_FEW:
        return "Too few lanes";
    # 如果并行处理线程数太多，则返回"Too many lanes"
    case ARGON2_LANES_TOO_MANY:
        return "Too many lanes";
    # 如果密码指针为空，但密码长度不为0，则返回"Password pointer is NULL, but password length is not 0"
    case ARGON2_PWD_PTR_MISMATCH:
        return "Password pointer is NULL, but password length is not 0";
    # 如果盐指针为空，但盐长度不为0，则返回"Salt pointer is NULL, but salt length is not 0"
    case ARGON2_SALT_PTR_MISMATCH:
        return "Salt pointer is NULL, but salt length is not 0";
    # 如果密钥指针为空，但密钥长度不为0，则返回"Secret pointer is NULL, but secret length is not 0"
    case ARGON2_SECRET_PTR_MISMATCH:
        return "Secret pointer is NULL, but secret length is not 0";
    # 如果相关数据指针为空，但相关数据长度不为0，则返回"Associated data pointer is NULL, but ad length is not 0"
    case ARGON2_AD_PTR_MISMATCH:
        return "Associated data pointer is NULL, but ad length is not 0";
    # 如果内存分配错误，则返回"Memory allocation error"
    case ARGON2_MEMORY_ALLOCATION_ERROR:
        return "Memory allocation error";
    # 如果释放内存回调为空，则返回"The free memory callback is NULL"
    case ARGON2_FREE_MEMORY_CBK_NULL:
        return "The free memory callback is NULL";
    # 如果分配内存回调为空，则返回"The allocate memory callback is NULL"
    case ARGON2_ALLOCATE_MEMORY_CBK_NULL:
        return "The allocate memory callback is NULL";
    # 如果参数不正确，则返回"Argon2_Context context is NULL"
    case ARGON2_INCORRECT_PARAMETER:
        return "Argon2_Context context is NULL";
    # 如果类型不正确，则返回"There is no such version of Argon2"
    case ARGON2_INCORRECT_TYPE:
        return "There is no such version of Argon2";
    # 如果输出指针不匹配，则返回"Output pointer mismatch"
    case ARGON2_OUT_PTR_MISMATCH:
        return "Output pointer mismatch";
    # 如果线程数太少，则返回"Not enough threads"
    case ARGON2_THREADS_TOO_FEW:
        return "Not enough threads";
    # 如果线程数太多，则返回"Too many threads"
    case ARGON2_THREADS_TOO_MANY:
        return "Too many threads";
    # 如果缺少参数，则返回"Missing arguments"
    case ARGON2_MISSING_ARGS:
        return "Missing arguments";
    # 如果编码失败，则返回"Encoding failed"
    case ARGON2_ENCODING_FAIL:
        return "Encoding failed";
    # 如果解码失败，则返回"Decoding failed"
    case ARGON2_DECODING_FAIL:
        return "Decoding failed";
    # 如果线程失败，则返回"Threading failure"
    case ARGON2_THREAD_FAIL:
        return "Threading failure";
    # 如果解码长度失败，则返回"Some of encoded parameters are too long or too short"
    case ARGON2_DECODING_LENGTH_FAIL:
        return "Some of encoded parameters are too long or too short";
    # 如果返回的错误码是ARGON2_VERIFY_MISMATCH，则密码与提供的哈希不匹配
    case ARGON2_VERIFY_MISMATCH:
        return "The password does not match the supplied hash";
    # 如果返回的错误码是其他值，则是未知的错误码
    default:
        return "Unknown error code";
    }
# 计算并返回使用 Argon2 加密算法编码后的长度
size_t argon2_encodedlen(uint32_t t_cost, uint32_t m_cost, uint32_t parallelism,
                         uint32_t saltlen, uint32_t hashlen, argon2_type type) {
    # 返回编码后的长度，包括固定字符串长度和各个参数的长度
    return strlen("$$v=$m=,t=,p=$$") + strlen(argon2_type2string(type, 0)) +
            numlen(t_cost) + numlen(m_cost) + numlen(parallelism) +
            b64len(saltlen) + b64len(hashlen) + numlen(ARGON2_VERSION_NUMBER) +
            1;
}
```