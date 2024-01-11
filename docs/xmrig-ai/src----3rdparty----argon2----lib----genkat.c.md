# `xmrig\src\3rdparty\argon2\lib\genkat.c`

```
/*
 * Argon2源代码包
 *
 * 由Daniel Dinu和Dmitry Khovratovich编写，2015年
 *
 * 本作品根据知识共享CC0 1.0许可证/放弃许可证授权。
 *
 * 您应该已经收到了CC0公共领域奉献的副本
 * 这个软件。如果没有，请参见
 * <http://creativecommons.org/publicdomain/zero/1.0/>。
 */

#include <inttypes.h>
#include <stdio.h>

#include "genkat.h"

void initial_kat(const uint8_t *blockhash, const argon2_context *context,
                 argon2_type type) {
    unsigned i;
    # 检查 blockhash 和 context 是否都不为空
    if (blockhash != NULL && context != NULL) {
        # 打印分隔线
        printf("=======================================\n");

        # 打印算法类型和版本号
        printf("%s version number %d\n", argon2_type2string(type, 1),
               context->version);

        # 打印分隔线
        printf("=======================================\n");

        # 打印内存、迭代次数、并行度和标签长度
        printf("Memory: %u KiB, Iterations: %u, Parallelism: %u lanes, Tag "
               "length: %u bytes\n",
               context->m_cost, context->t_cost, context->lanes,
               context->outlen);

        # 打印密码长度和密码内容（如果未清除）
        printf("Password[%u]: ", context->pwdlen);
        if (context->flags & ARGON2_FLAG_CLEAR_PASSWORD) {
            printf("CLEARED\n");
        } else {
            for (i = 0; i < context->pwdlen; ++i) {
                printf("%2.2x ", ((unsigned char *)context->pwd)[i]);
            }
            printf("\n");
        }

        # 打印盐值长度和盐值内容
        printf("Salt[%u]: ", context->saltlen);
        for (i = 0; i < context->saltlen; ++i) {
            printf("%2.2x ", ((unsigned char *)context->salt)[i]);
        }
        printf("\n");

        # 打印密钥长度和密钥内容（如果未清除）
        printf("Secret[%u]: ", context->secretlen);
        if (context->flags & ARGON2_FLAG_CLEAR_SECRET) {
            printf("CLEARED\n");
        } else {
            for (i = 0; i < context->secretlen; ++i) {
                printf("%2.2x ", ((unsigned char *)context->secret)[i]);
            }
            printf("\n");
        }

        # 打印关联数据长度和关联数据内容
        printf("Associated data[%u]: ", context->adlen);
        for (i = 0; i < context->adlen; ++i) {
            printf("%2.2x ", ((unsigned char *)context->ad)[i]);
        }
        printf("\n");

        # 打印预哈希摘要内容
        printf("Pre-hashing digest: ");
        for (i = 0; i < ARGON2_PREHASH_DIGEST_LENGTH; ++i) {
            printf("%2.2x ", ((unsigned char *)blockhash)[i]);
        }
        printf("\n");
    }
// 打印输出标签和其对应的十六进制值
void print_tag(const void *out, uint32_t outlen) {
    unsigned i;
    // 如果输出不为空
    if (out != NULL) {
        // 打印标签
        printf("Tag: ");

        // 遍历输出的每个字节，打印其十六进制值
        for (i = 0; i < outlen; ++i) {
            printf("%2.2x ", ((uint8_t *)out)[i]);
        }

        printf("\n");
    }
}

// 执行内部的KAT（Known Answer Test）测试，打印每个pass后的内存块状态
void internal_kat(const argon2_instance_t *instance, uint32_t pass) {
    // 如果实例不为空
    if (instance != NULL) {
        uint32_t i, j;
        // 打印pass后的内存块状态
        printf("\n After pass %u:\n", pass);

        // 遍历每个内存块
        for (i = 0; i < instance->memory_blocks; ++i) {
            // 计算每个内存块包含的字数
            uint32_t how_many_words =
                (instance->memory_blocks > ARGON2_QWORDS_IN_BLOCK)
                    ? 1
                    : ARGON2_QWORDS_IN_BLOCK;

            // 遍历每个字，打印其十六进制值
            for (j = 0; j < how_many_words; ++j)
                printf("Block %.4u [%3u]: %016" PRIx64 "\n", i, j,
                       instance->memory[i].v[j]);
        }
    }
}
```