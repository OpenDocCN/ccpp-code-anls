# `xmrig\src\3rdparty\libethash\ethash.h`

```
/*
  这个文件是 ethash 的一部分。

  ethash 是自由软件：您可以根据自由软件基金会发布的 GNU 通用公共许可证的第 3 版或（根据您的选择）任何更高版本的条款重新分发和/或修改它。

  ethash 是希望它能有用的自由软件，但没有任何担保；甚至没有适销性或特定用途的隐含担保。有关更多详细信息，请参见 GNU 通用公共许可证。

  您应该已经收到了 GNU 通用公共许可证的副本。如果没有，请参见 <http://www.gnu.org/licenses/>。
*/

/** @file ethash.h
* @date 2015
*/
#pragma once

#include <stdint.h>
#include <stdbool.h>
#include <string.h>
#include <stddef.h>

#define ETHASH_REVISION 23
#define ETHASH_DATASET_BYTES_INIT 1073741824U // 2**30
#define ETHASH_DATASET_BYTES_GROWTH 8388608U  // 2**23
#define ETHASH_CACHE_BYTES_INIT 1073741824U // 2**24
#define ETHASH_CACHE_BYTES_GROWTH 131072U  // 2**17
#define ETHASH_EPOCH_LENGTH 30000U
#define ETHASH_MIX_BYTES 128
#define ETHASH_HASH_BYTES 64
#define ETHASH_DATASET_PARENTS 256
#define ETHASH_CACHE_ROUNDS 3
#define ETHASH_ACCESSES 64
#define ETHASH_DAG_MAGIC_NUM_SIZE 8
#define ETHASH_DAG_MAGIC_NUM 0xFEE1DEADBADDCAFE

#ifdef __cplusplus
extern "C" {
#endif

/// Type of a seedhash/blockhash e.t.c.
typedef struct ethash_h256 { uint8_t b[32]; } ethash_h256_t;

// convenience macro to statically initialize an h256_t
// usage:
// ethash_h256_t a = ethash_h256_static_init(1, 2, 3, ... )
// have to provide all 32 values. If you don't provide all the rest
// will simply be unitialized (not guranteed to be 0)
#define ethash_h256_static_init(...)            \
    { {__VA_ARGS__} }

struct ethash_light;
typedef struct ethash_light* ethash_light_t;
struct ethash_full;
typedef struct ethash_full* ethash_full_t;
typedef int(*ethash_callback_t)(unsigned);

typedef struct ethash_return_value {
    ethash_h256_t result;
*/
    # 定义一个名为 mix_hash 的变量，类型为 ethash_h256_t，用于存储混合哈希值
    ethash_h256_t mix_hash;
    # 定义一个名为 success 的变量，类型为 bool，用于表示操作是否成功
    bool success;
/**
 * 定义了一个结构体 ethash_return_value_t
 */
} ethash_return_value_t;

/**
 * 分配并初始化一个新的 ethash_light 处理程序
 *
 * @param block_number   要创建处理程序的块号
 * @return               新分配的 ethash_light 处理程序，或者在使用 @ref ethash_compute_cache_nodes() 的参数出错时返回 NULL
 */
ethash_light_t ethash_light_new(uint64_t block_number);
/**
 */
bool ethash_compute_cache_nodes(
    void* nodes,
    uint64_t cache_size,
    ethash_h256_t const* seed
);
/**
 * 释放先前分配的 ethash_light 处理程序
 * @param light        要释放的 light 处理程序
 */
void ethash_light_delete(ethash_light_t light);
/**
 * 计算轻客户端数据
 *
 * @param light          轻客户端处理程序
 * @param header_hash    要打包到混合中的头哈希
 * @param nonce          要打包到混合中的随机数
 * @return               一个包含返回值的 ethash_return_value_t 对象
 */
ethash_return_value_t ethash_light_compute(
    ethash_light_t light,
    ethash_h256_t const header_hash,
    uint64_t nonce
);
/**
 * 分配并初始化一个新的 ethash_full 处理程序
 *
 * @param light         包含缓存的 light 处理程序。
 * @param callback      一个带有 @ref ethash_callback_t 签名的回调函数
 *                      它接受一个无符号整数，用于显示 DAG 计算的进度。
 *                      如果一切顺利，回调应该返回 0。
 *                      如果返回一个非零值，则 DAG 生成将停止。
 *                      请注意，进度值为 100 意味着 DAG 创建几乎完成，
 *                      并且该函数很快将成功返回。
 *                      这并不意味着函数已经成功返回。
 * @return              新分配的 ethash_full 处理程序，或者在使用 @ref ethash_compute_full_data() 的时候出现 ERRNOMEM 或无效参数时返回 NULL
 */
ethash_full_t ethash_full_new(ethash_light_t light, ethash_callback_t callback);

/**
 * 释放先前分配的 ethash_full 处理程序
 * @param full    要释放的 light 处理程序
 */
void ethash_full_delete(ethash_full_t full);

/**
 * 计算完整的客户端数据
 *
 * @param full           完整的客户端处理程序
 * @param header_hash    要打包到 mix 中的头哈希
 * @param nonce          要打包到 mix 中的 nonce
 * @return               一个 ethash_return_value 对象，用于保存返回值
 */
ethash_return_value_t ethash_full_compute(
    ethash_full_t full,
    ethash_h256_t const header_hash,
    uint64_t nonce
);

/**
 * 获取完整 DAG 数据的指针
 */
void const* ethash_full_dag(ethash_full_t full);

/**
 * 获取 DAG 数据的大小
 */
uint64_t ethash_full_dag_size(ethash_full_t full);

/**
 * 计算给定时期的 seedhash
 */
ethash_h256_t ethash_get_seedhash(uint64_t epoch);

/**
 * ProgPoW 的 KeccakF800
 */
void ethash_keccakf800(uint32_t state[25]);

#ifdef __cplusplus
}
#endif
```