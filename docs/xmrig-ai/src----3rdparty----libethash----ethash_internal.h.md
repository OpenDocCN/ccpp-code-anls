# `xmrig\src\3rdparty\libethash\ethash_internal.h`

```cpp
#pragma once
#include "endian.h"
#include "ethash.h"
#include <stdio.h>

#define ENABLE_SSE 0

#if defined(_M_X64) && ENABLE_SSE
#include <smmintrin.h>
#endif

#ifdef __cplusplus
extern "C" {
#endif

// compile time settings
#define NODE_WORDS (64/4)  // 定义节点字长为64位，每个字长为4字节
#define MIX_WORDS (ETHASH_MIX_BYTES/4)  // 定义混合数据的字长为ETHASH_MIX_BYTES的四分之一
#define MIX_NODES (MIX_WORDS / NODE_WORDS)  // 计算混合数据中节点的数量

#include <stdint.h>

typedef union node {
    uint8_t bytes[NODE_WORDS * 4];  // 节点的字节数组
    uint32_t words[NODE_WORDS];  // 节点的32位字数组
    uint64_t double_words[NODE_WORDS / 2];  // 节点的64位双字数组

#if defined(_M_X64) && ENABLE_SSE
    __m128i xmm[NODE_WORDS/4];  // 如果启用SSE指令集，则使用__m128i类型的数组
#endif

} node;

static inline uint8_t ethash_h256_get(ethash_h256_t const* hash, unsigned int i)
{
    return hash->b[i];  // 获取hash中第i个字节的值
}

static inline void ethash_h256_set(ethash_h256_t* hash, unsigned int i, uint8_t v)
{
    hash->b[i] = v;  // 设置hash中第i个字节的值为v
}

static inline void ethash_h256_reset(ethash_h256_t* hash)
{
    memset(hash, 0, 32);  // 将hash的内容全部置为0
}

// Returns if hash is less than or equal to boundary (2^256/difficulty)
static inline bool ethash_check_difficulty(
    ethash_h256_t const* hash,
    ethash_h256_t const* boundary
)
{
    // Boundary is big endian
    for (int i = 0; i < 32; i++) {
        if (ethash_h256_get(hash, i) == ethash_h256_get(boundary, i)) {
            continue;  // 如果hash和boundary的第i个字节相等，则继续下一次循环
        }
        return ethash_h256_get(hash, i) < ethash_h256_get(boundary, i);  // 如果hash的第i个字节小于boundary的第i个字节，则返回true
    }
    return true;  // 如果所有字节都相等，则返回true
}

/**
 *  Difficulty quick check for POW preverification
 *
 * @param header_hash      The hash of the header
 * @param nonce            The block's nonce
 * @param mix_hash         The mix digest hash
 * @param boundary         The boundary is defined as (2^256 / difficulty)
 * @return                 true for succesful pre-verification and false otherwise
 */
bool ethash_quick_check_difficulty(
    ethash_h256_t const* header_hash,
    uint64_t const nonce,
    ethash_h256_t const* mix_hash,
    ethash_h256_t const* boundary
);

struct ethash_light {
    void* cache;
    uint64_t cache_size;
    uint64_t block_number;

    // Used for fast division
    # 定义一个32位无符号整数变量，用于存储父节点的数量
    uint32_t num_parent_nodes;
    # 定义一个32位无符号整数变量，用于存储倒数值
    uint32_t reciprocal;
    # 定义一个32位无符号整数变量，用于存储增量值
    uint32_t increment;
    # 定义一个32位无符号整数变量，用于存储移位值
    uint32_t shift;
// 结构体 ethash_full 包含文件指针、文件大小和数据节点
struct ethash_full {
    FILE* file; // 文件指针
    uint64_t file_size; // 文件大小
    node* data; // 数据节点
};

/**
 * 分配并初始化一个新的 ethash_light 处理程序。内部版本
 *
 * @param cache_size    缓存大小（以字节为单位）
 * @param seed          在计算缓存节点时要使用的块种子哈希
 * @return              新分配的 ethash_light 处理程序，或者在使用 @ref ethash_compute_cache_nodes() 的 ERRNOMEM 或无效参数时返回 NULL
 */
ethash_light_t ethash_light_new_internal(uint64_t cache_size, ethash_h256_t const* seed);

/**
 * 计算轻客户端数据。内部版本
 *
 * @param light          轻客户端处理程序
 * @param full_size      完整数据的大小（以字节为单位）
 * @param header_hash    要打包到混合中的头哈希
 * @param nonce          要打包到混合中的随机数
 * @return               结果哈希
 */
ethash_return_value_t ethash_light_compute_internal(
    ethash_light_t light,
    uint64_t full_size,
    ethash_h256_t const header_hash,
    uint64_t nonce
);
/**
 * 分配并初始化一个新的ethash_full处理程序。内部版本。
 *
 * @param dirname        放置DAG文件的目录。
 * @param seedhash       区块的种子哈希。用于DAG文件命名。
 * @param full_size      完整数据的大小（以字节为单位）。
 * @param cache          要使用的缓存对象，使用@ref ethash_cache_new()分配。
 *                       如果此函数成功，ethash_full_t将拥有缓存的内存所有权，并在删除时释放它。
 *                       如果不成功，则用户仍然必须自行处理缓存的释放。
 * @param callback       具有@ref ethash_callback_t签名的回调函数
 *                       它接受一个无符号整数，可以显示DAG计算的进度。
 *                       如果一切顺利，回调应返回0。
 *                       如果返回非零值，则DAG生成将停止。
 * @return               新分配的ethash_full处理程序，或在@ref ethash_compute_full_data()使用的ERRNOMEM或无效参数的情况下返回NULL。
 */
ethash_full_t ethash_full_new_internal(
    char const* dirname,
    ethash_h256_t const seed_hash,
    uint64_t full_size,
    ethash_light_t const light,
    ethash_callback_t callback
);

void ethash_calculate_dag_item(
    node* const ret,
    uint32_t node_index,
    uint32_t num_parents,
    ethash_light_t const cache
);

void ethash_calculate_dag_item_opt(
    node* const ret,
    uint32_t node_index,
    uint32_t num_parents,
    ethash_light_t const cache
);

void ethash_calculate_dag_item4_opt(
    node* ret,
    uint32_t node_index,
    uint32_t num_parents,
    ethash_light_t const cache
);

void ethash_quick_hash(
    ethash_h256_t* return_hash,
    ethash_h256_t const* header_hash,
    const uint64_t nonce,
    ethash_h256_t const* mix_hash
);
// 声明一个函数，用于获取指定区块号的数据大小
uint64_t ethash_get_datasize(uint64_t const block_number);
// 声明一个函数，用于获取指定区块号的缓存大小
uint64_t ethash_get_cachesize(uint64_t const block_number);

/**
 * 计算完整节点内存的数据
 *
 * @param mem         指向ethash full内存的指针
 * @param full_size   完整数据的大小（以字节为单位）
 * @param cache       用于计算的缓存对象
 * @param callback    回调函数。有关详细信息，请参阅@ref ethash_full_new()
 * @return            如果一切顺利返回true，否则返回false表示参数无效
 */
bool ethash_compute_full_data(
    void* mem,
    uint64_t full_size,
    ethash_light_t const light,
    ethash_callback_t callback
);

#ifdef __cplusplus
}
#endif
```