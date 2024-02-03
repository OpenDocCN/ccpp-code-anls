# `xmrig\src\3rdparty\argon2\include\argon2.h`

```cpp
/*
 * Argon2源代码包
 *
 * 由Daniel Dinu和Dmitry Khovratovich编写，2015年
 *
 * 本作品根据知识共享CC0 1.0许可证/豁免授权发布。
 *
 * 您应该已经收到了CC0公共领域奉献的副本
 * 与本软件一起。如果没有，请参见
 * <http://creativecommons.org/publicdomain/zero/1.0/>。
 */

#ifndef ARGON2_H
#define ARGON2_H

#include <stdint.h>
#include <stddef.h>
#include <stdio.h>
#include <limits.h>

/* 符号可见性控制 */
#define ARGON2_PUBLIC

#if defined(__cplusplus)
extern "C" {
#endif

/*
 * Argon2输入参数限制
 */

/* 最小和最大的通道数（并行度） */
#define ARGON2_MIN_LANES UINT32_C(1)
#define ARGON2_MAX_LANES UINT32_C(0xFFFFFF)

/* 线程的最小和最大数量 */
#define ARGON2_MIN_THREADS UINT32_C(1)
#define ARGON2_MAX_THREADS UINT32_C(0xFFFFFF)

/* 每个传递中通道之间的同步点数 */
#define ARGON2_SYNC_POINTS UINT32_C(4)

/* 摘要大小的最小和最大值（以字节为单位） */
#define ARGON2_MIN_OUTLEN UINT32_C(4)
#define ARGON2_MAX_OUTLEN UINT32_C(0xFFFFFFFF)

/* 内存块的最小和最大数量（每个BLOCK_SIZE字节） */
#define ARGON2_MIN_MEMORY (2 * ARGON2_SYNC_POINTS) /* 每个切片2个块 */

#define ARGON2_MIN(a, b) ((a) < (b) ? (a) : (b))
/* 最大内存大小是地址空间的一半，最大为2^32块（4 TB） */
#define ARGON2_MAX_MEMORY_BITS                                                 \
    ARGON2_MIN(UINT32_C(32), (sizeof(void *) * CHAR_BIT - 10 - 1))
#define ARGON2_MAX_MEMORY                                                      \
    ARGON2_MIN(UINT32_C(0xFFFFFFFF), UINT64_C(1) << ARGON2_MAX_MEMORY_BITS)

/* 传递的最小和最大数量 */
#define ARGON2_MIN_TIME UINT32_C(1)
#define ARGON2_MAX_TIME UINT32_C(0xFFFFFFFF)

/* 密码长度的最小和最大值（以字节为单位） */
#define ARGON2_MIN_PWD_LENGTH UINT32_C(0)
#define ARGON2_MAX_PWD_LENGTH UINT32_C(0xFFFFFFFF)
/* 最小和最大关联数据长度（以字节为单位） */
#define ARGON2_MIN_AD_LENGTH UINT32_C(0)
#define ARGON2_MAX_AD_LENGTH UINT32_C(0xFFFFFFFF)

/* 最小和最大盐长度（以字节为单位） */
#define ARGON2_MIN_SALT_LENGTH UINT32_C(8)
#define ARGON2_MAX_SALT_LENGTH UINT32_C(0xFFFFFFFF)

/* 最小和最大密钥长度（以字节为单位） */
#define ARGON2_MIN_SECRET UINT32_C(0)
#define ARGON2_MAX_SECRET UINT32_C(0xFFFFFFFF)

/* 用于确定哪些字段被安全擦除的标志（默认 = 不擦除）。 */
#define ARGON2_DEFAULT_FLAGS UINT32_C(0)
#define ARGON2_FLAG_CLEAR_PASSWORD (UINT32_C(1) << 0)
#define ARGON2_FLAG_CLEAR_SECRET (UINT32_C(1) << 1)
#define ARGON2_FLAG_GENKAT (UINT32_C(1) << 3)

/* 全局标志，用于确定是否擦除内部内存缓冲区。此标志
 * 在core.c中定义，默认为1（擦除内部内存）。 */
extern int FLAG_clear_internal_memory;

/* 错误代码 */
typedef enum Argon2_ErrorCodes {
    ARGON2_OK = 0,

    ARGON2_OUTPUT_PTR_NULL = -1,

    ARGON2_OUTPUT_TOO_SHORT = -2,
    ARGON2_OUTPUT_TOO_LONG = -3,

    ARGON2_PWD_TOO_SHORT = -4,
    ARGON2_PWD_TOO_LONG = -5,

    ARGON2_SALT_TOO_SHORT = -6,
    ARGON2_SALT_TOO_LONG = -7,

    ARGON2_AD_TOO_SHORT = -8,
    ARGON2_AD_TOO_LONG = -9,

    ARGON2_SECRET_TOO_SHORT = -10,
    ARGON2_SECRET_TOO_LONG = -11,

    ARGON2_TIME_TOO_SMALL = -12,
    ARGON2_TIME_TOO_LARGE = -13,

    ARGON2_MEMORY_TOO_LITTLE = -14,
    ARGON2_MEMORY_TOO_MUCH = -15,

    ARGON2_LANES_TOO_FEW = -16,
    ARGON2_LANES_TOO_MANY = -17,

    ARGON2_PWD_PTR_MISMATCH = -18,    /* 非零长度的空指针 */
    ARGON2_SALT_PTR_MISMATCH = -19,   /* 非零长度的空指针 */
    ARGON2_SECRET_PTR_MISMATCH = -20, /* 非零长度的空指针 */
    ARGON2_AD_PTR_MISMATCH = -21,     /* 非零长度的空指针 */

    ARGON2_MEMORY_ALLOCATION_ERROR = -22,

    ARGON2_FREE_MEMORY_CBK_NULL = -23,
    ARGON2_ALLOCATE_MEMORY_CBK_NULL = -24,

    ARGON2_INCORRECT_PARAMETER = -25,
    # 定义 ARGON2_INCORRECT_TYPE 常量，数值为 -26
    ARGON2_INCORRECT_TYPE = -26,

    # 定义 ARGON2_OUT_PTR_MISMATCH 常量，数值为 -27
    ARGON2_OUT_PTR_MISMATCH = -27,

    # 定义 ARGON2_THREADS_TOO_FEW 常量，数值为 -28
    ARGON2_THREADS_TOO_FEW = -28,
    # 定义 ARGON2_THREADS_TOO_MANY 常量，数值为 -29
    ARGON2_THREADS_TOO_MANY = -29,

    # 定义 ARGON2_MISSING_ARGS 常量，数值为 -30
    ARGON2_MISSING_ARGS = -30,

    # 定义 ARGON2_ENCODING_FAIL 常量，数值为 -31
    ARGON2_ENCODING_FAIL = -31,

    # 定义 ARGON2_DECODING_FAIL 常量，数值为 -32
    ARGON2_DECODING_FAIL = -32,

    # 定义 ARGON2_THREAD_FAIL 常量，数值为 -33
    ARGON2_THREAD_FAIL = -33,

    # 定义 ARGON2_DECODING_LENGTH_FAIL 常量，数值为 -34
    ARGON2_DECODING_LENGTH_FAIL = -34,

    # 定义 ARGON2_VERIFY_MISMATCH 常量，数值为 -35
    ARGON2_VERIFY_MISMATCH = -35
} argon2_error_codes;

/* Memory allocator types --- for external allocation */
// 为外部分配内存而定义的内存分配器类型
typedef int (*allocate_fptr)(uint8_t **memory, size_t bytes_to_allocate);
typedef void (*deallocate_fptr)(uint8_t *memory, size_t bytes_to_allocate);

/* Argon2 external data structures */

/*
 *****
 * Context: structure to hold Argon2 inputs:
 *  output array and its length,
 *  password and its length,
 *  salt and its length,
 *  secret and its length,
 *  associated data and its length,
 *  number of passes, amount of used memory (in KBytes, can be rounded up a bit)
 *  number of parallel threads that will be run.
 * All the parameters above affect the output hash value.
 * Additionally, two function pointers can be provided to allocate and
 * deallocate the memory (if NULL, memory will be allocated internally).
 * Also, three flags indicate whether to erase password, secret as soon as they
 * are pre-hashed (and thus not needed anymore), and the entire memory
 *****
 * Simplest situation: you have output array out[8], password is stored in
 * pwd[32], salt is stored in salt[16], you do not have keys nor associated
 * data. You need to spend 1 GB of RAM and you run 5 passes of Argon2d with
 * 4 parallel lanes.
 * You want to erase the password, but you're OK with last pass not being
 * erased. You want to use the default memory allocator.
 * Then you initialize:
 Argon2_Context(out,8,pwd,32,salt,16,NULL,0,NULL,0,5,1<<20,4,4,NULL,NULL,true,false,false,false)
 */
// Argon2 上下文结构，用于保存 Argon2 的输入参数
typedef struct Argon2_Context {
    uint8_t *out;    /* output array */  // 输出数组
    uint32_t outlen; /* digest length */  // 摘要长度

    uint8_t *pwd;    /* password array */  // 密码数组
    uint32_t pwdlen; /* password length */  // 密码长度

    uint8_t *salt;    /* salt array */  // 盐数组
    uint32_t saltlen; /* salt length */  // 盐长度

    uint8_t *secret;    /* key array */  // 密钥数组
    uint32_t secretlen; /* key length */  // 密钥长度

    uint8_t *ad;    /* associated data array */  // 关联数据数组
    uint32_t adlen; /* associated data length */  // 关联数据长度

    uint32_t t_cost;  /* number of passes */  // 经过的轮数
    # 请求的内存量（KB）
    uint32_t m_cost;  
    # 转换的数量
    uint32_t lanes;   
    # 最大线程数
    uint32_t threads; 

    # 版本号
    uint32_t version; 

    # 内存分配器的指针
    allocate_fptr allocate_cbk; 
    # 内存释放器的指针
    deallocate_fptr free_cbk;   

    # 布尔选项数组
    uint32_t flags; 
} argon2_context;

/* Argon2 原语类型 */
typedef enum Argon2_type {
    Argon2_d = 0,  // 数据依赖型
    Argon2_i = 1,  // 数据独立型
    Argon2_id = 2  // 混合型
} argon2_type;

/* 算法的版本 */
typedef enum Argon2_version {
    ARGON2_VERSION_10 = 0x10,  // 版本 1.0
    ARGON2_VERSION_13 = 0x13,  // 版本 1.3
    ARGON2_VERSION_NUMBER = ARGON2_VERSION_13  // 使用版本 1.3
} argon2_version;

/*
 * 给定 argon2_type，返回其字符串表示形式
 * @param type 要获取字符串表示形式的 argon2_type
 * @param uppercase 字符串是否首字母大写
 * @return 如果类型无效则返回 NULL，否则返回字符串表示形式
 */
ARGON2_PUBLIC const char *argon2_type2string(argon2_type type, int uppercase);

/*
 * 使用一定程度的并行性执行内存硬哈希
 * @param  context  指向 Argon2 内部结构的指针
 * @return 如果出现错误则返回错误代码，否则返回 ARGON2_OK
 */
ARGON2_PUBLIC int argon2_ctx(argon2_context *context, argon2_type type);

/**
 * 使用 Argon2i 对密码进行哈希，生成编码后的哈希
 * @param t_cost 迭代次数
 * @param m_cost 将内存使用量设置为 m_cost kibibytes
 * @param parallelism 线程数和计算通道数
 * @param pwd 密码指针
 * @param pwdlen 密码字节大小
 * @param salt 盐指针
 * @param saltlen 盐字节大小
 * @param hashlen 哈希的期望长度（字节）
 * @param encoded 要写入编码后的哈希的缓冲区
 * @param encodedlen 缓冲区的大小（编码后哈希的最大大小）
 * @pre   不同的并行性级别将产生不同的结果
 * @pre   如果成功则返回 ARGON2_OK
 */
/**
 * 使用Argon2i对密码进行哈希处理，通过在@hash处分配内存来生成原始哈希
 * @param t_cost 迭代次数
 * @param m_cost 将内存使用量设置为m_cost kibibytes
 * @param parallelism 线程数和计算通道数
 * @param pwd 密码指针
 * @param pwdlen 密码的字节大小
 * @param salt 盐指针
 * @param saltlen 盐的字节大小
 * @param hash 用于写入原始哈希的缓冲区 - 由函数更新
 * @param hashlen 哈希的期望长度（以字节为单位）
 * @pre 不同的并行级别将产生不同的结果
 * @pre 如果成功则返回ARGON2_OK
 */
ARGON2_PUBLIC int argon2i_hash_raw(const uint32_t t_cost, const uint32_t m_cost,
                                   const uint32_t parallelism, const void *pwd,
                                   const size_t pwdlen, const void *salt,
                                   const size_t saltlen, void *hash,
                                   const size_t hashlen);

/**
 * 使用Argon2d对密码进行哈希处理，通过在@encoded处分配内存来生成编码哈希
 * @param t_cost 迭代次数
 * @param m_cost 将内存使用量设置为m_cost kibibytes
 * @param parallelism 线程数和计算通道数
 * @param pwd 密码指针
 * @param pwdlen 密码的字节大小
 * @param salt 盐指针
 * @param saltlen 盐的字节大小
 * @param hashlen 哈希的期望长度（以字节为单位）
 * @param encoded 用于写入编码哈希的缓冲区 - 由函数更新
 * @param encodedlen 编码哈希的最大长度
 */
ARGON2_PUBLIC int argon2d_hash_encoded(const uint32_t t_cost,
                                       const uint32_t m_cost,
                                       const uint32_t parallelism,
                                       const void *pwd, const size_t pwdlen,
                                       const void *salt, const size_t saltlen,
                                       const size_t hashlen, char *encoded,
                                       const size_t encodedlen);
# 使用Argon2算法对原始密码进行哈希处理，返回原始哈希值
ARGON2_PUBLIC int argon2d_hash_raw(const uint32_t t_cost,
                                   const uint32_t m_cost,
                                   const uint32_t parallelism, const void *pwd,
                                   const size_t pwdlen, const void *salt,
                                   const size_t saltlen, void *hash,
                                   const size_t hashlen);

# 使用Argon2算法对原始密码进行哈希处理，返回Base64编码的哈希值
ARGON2_PUBLIC int argon2id_hash_encoded(const uint32_t t_cost,
                                        const uint32_t m_cost,
                                        const uint32_t parallelism,
                                        const void *pwd, const size_t pwdlen,
                                        const void *salt, const size_t saltlen,
                                        const size_t hashlen, char *encoded,
                                        const size_t encodedlen);

# 使用Argon2算法对原始密码进行哈希处理，返回原始哈希值
ARGON2_PUBLIC int argon2id_hash_raw(const uint32_t t_cost,
                                    const uint32_t m_cost,
                                    const uint32_t parallelism, const void *pwd,
                                    const size_t pwdlen, const void *salt,
                                    const size_t saltlen, void *hash,
                                    const size_t hashlen);

# 使用Argon2算法对原始密码进行哈希处理，返回原始哈希值，并且可以指定内存
ARGON2_PUBLIC int argon2id_hash_raw_ex(const uint32_t t_cost,
                                       const uint32_t m_cost,
                                       const uint32_t parallelism, const void *pwd,
                                       const size_t pwdlen, const void *salt,
                                       const size_t saltlen, void *hash,
                                       const size_t hashlen,
                                       void *memory);

# 上述函数的通用底层函数
# 使用 Argon2 算法对密码进行哈希处理
ARGON2_PUBLIC int argon2_hash(const uint32_t t_cost, const uint32_t m_cost,
                              const uint32_t parallelism, const void *pwd,
                              const size_t pwdlen, const void *salt,
                              const size_t saltlen, void *hash,
                              const size_t hashlen, char *encoded,
                              const size_t encodedlen, argon2_type type,
                              const uint32_t version);
/**
 * 验证密码与编码字符串的匹配性
 * 编码字符串受 validate_inputs() 限制
 * @param encoded 编码参数、盐值、哈希值的字符串
 * @param pwd 密码指针
 * @pre 如果成功返回 ARGON2_OK
 */
ARGON2_PUBLIC int argon2i_verify(const char *encoded, const void *pwd,
                                 const size_t pwdlen);

ARGON2_PUBLIC int argon2d_verify(const char *encoded, const void *pwd,
                                 const size_t pwdlen);

ARGON2_PUBLIC int argon2id_verify(const char *encoded, const void *pwd,
                                  const size_t pwdlen);

/* 上述函数的通用函数 */
ARGON2_PUBLIC int argon2_verify(const char *encoded, const void *pwd,
                                const size_t pwdlen, argon2_type type);

/**
 * Argon2d: 根据密码和盐值选择内存块的版本。仅适用于无侧信道的环境！
 * @param  context  当前 Argon2 上下文的指针
 * @return  如果成功返回零，否则返回非零错误代码
 */
ARGON2_PUBLIC int argon2d_ctx(argon2_context *context);

/**
 * Argon2i: 不依赖于密码和盐值选择内存块的版本。适用于侧信道攻击，但如果只使用一次传递，则在权衡攻击方面较差。
 * @param  context  当前 Argon2 上下文的指针
 * @return  如果成功返回零，否则返回非零错误代码
 */
# 定义了一个名为 argon2i_ctx 的函数，接受一个 argon2_context 类型的指针参数，返回一个 int 类型的值
ARGON2_PUBLIC int argon2i_ctx(argon2_context *context);

/**
 * Argon2id: Version of Argon2 where the first half-pass over memory is
 * password-independent, the rest are password-dependent (on the password and
 * salt). OK against side channels (they reduce to 1/2-pass Argon2i), and
 * better with w.r.t. tradeoff attacks (similar to Argon2d).
 *****
 * @param  context  Pointer to current Argon2 context
 * @return  Zero if successful, a non zero error code otherwise
 */
# 定义了一个名为 argon2id_ctx 的函数，接受一个 argon2_context 类型的指针参数，返回一个 int 类型的值
ARGON2_PUBLIC int argon2id_ctx(argon2_context *context);

/**
 * Verify if a given password is correct for Argon2d hashing
 * @param  context  Pointer to current Argon2 context
 * @param  hash  The password hash to verify. The length of the hash is
 * specified by the context outlen member
 * @return  Zero if successful, a non zero error code otherwise
 */
# 定义了一个名为 argon2d_verify_ctx 的函数，接受一个 argon2_context 类型的指针参数和一个 const char 类型的指针参数，返回一个 int 类型的值
ARGON2_PUBLIC int argon2d_verify_ctx(argon2_context *context, const char *hash);

/**
 * Verify if a given password is correct for Argon2i hashing
 * @param  context  Pointer to current Argon2 context
 * @param  hash  The password hash to verify. The length of the hash is
 * specified by the context outlen member
 * @return  Zero if successful, a non zero error code otherwise
 */
# 定义了一个名为 argon2i_verify_ctx 的函数，接受一个 argon2_context 类型的指针参数和一个 const char 类型的指针参数，返回一个 int 类型的值
ARGON2_PUBLIC int argon2i_verify_ctx(argon2_context *context, const char *hash);

/**
 * Verify if a given password is correct for Argon2id hashing
 * @param  context  Pointer to current Argon2 context
 * @param  hash  The password hash to verify. The length of the hash is
 * specified by the context outlen member
 * @return  Zero if successful, a non zero error code otherwise
 */
# 定义了一个名为 argon2id_verify_ctx 的函数，接受一个 argon2_context 类型的指针参数和一个 const char 类型的指针参数，返回一个 int 类型的值
ARGON2_PUBLIC int argon2id_verify_ctx(argon2_context *context, const char *hash);

/* generic function underlying the above ones */
# 定义了一个名为 argon2_verify_ctx 的函数，接受一个 argon2_context 类型的指针参数、一个 const char 类型的指针参数和一个 argon2_type 类型的参数，返回一个 int 类型的值
ARGON2_PUBLIC int argon2_verify_ctx(argon2_context *context, const char *hash, argon2_type type);
/**
 * 获取给定错误代码的关联错误消息
 * @return 与给定错误代码关联的错误消息
 */
ARGON2_PUBLIC const char *argon2_error_message(int error_code);

/**
 * 返回给定输入参数的编码哈希长度
 * @param t_cost  迭代次数
 * @param m_cost  使用的内存量（以 kibibytes 为单位）
 * @param parallelism  线程数；用于计算通道数
 * @param saltlen  盐的大小（以字节为单位）
 * @param hashlen  哈希的大小（以字节为单位）
 * @param type 我们想要获取编码长度的 argon2_type
 * @return  编码哈希的长度（以字节为单位）
 */
ARGON2_PUBLIC size_t argon2_encodedlen(uint32_t t_cost, uint32_t m_cost,
                                       uint32_t parallelism, uint32_t saltlen,
                                       uint32_t hashlen, argon2_type type);

/* 表示 argon2_select_impl 可用： */
#define ARGON2_SELECTABLE_IMPL

/**
 * 选择最快的可用优化实现。
 * @param out 调试输出的文件（例如 stderr；如果不需要调试输出，则传入 NULL）
 * @param prefix 每行前面要打印的内容；NULL 等同于空字符串
 */
ARGON2_PUBLIC void argon2_select_impl();
ARGON2_PUBLIC const char *argon2_get_impl_name();
ARGON2_PUBLIC int argon2_select_impl_by_name(const char *name);

/* 表示支持传递预分配的内存： */
#define ARGON2_PREALLOCATED_MEMORY

ARGON2_PUBLIC size_t argon2_memory_size(uint32_t m_cost, uint32_t parallelism);

/**
 * 执行具有一定并行度的内存硬哈希函数
 * @param context       指向 Argon2 内部结构的指针
 * @param type          Argon2 类型
 * @param memory        用于块的预分配内存（或 NULL）
 * @param memory_size   预分配内存的大小
 * @return 如果出现错误，则返回错误代码，否则返回 ARGON2_OK
 */
# 定义一个名为 argon2_ctx_mem 的函数，返回类型为 ARGON2_PUBLIC int，接受参数 argon2_context *context, argon2_type type, void *memory, size_t memory_size
ARGON2_PUBLIC int argon2_ctx_mem(argon2_context *context, argon2_type type,
                                 void *memory, size_t memory_size);

# 如果是 C++ 环境，则结束 extern "C" 声明
#if defined(__cplusplus)
}
#endif

# 结束头文件的条件编译
#endif
```