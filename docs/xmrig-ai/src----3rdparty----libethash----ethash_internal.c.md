# `xmrig\src\3rdparty\libethash\ethash_internal.c`

```
/*
  这个文件是 ethash 的一部分。

  ethash 是自由软件：您可以根据自由软件基金会发布的 GNU 通用公共许可证的第 3 版或（根据您的选择）更高版本的条款重新分发和/或修改它。

  ethash 是希望它能够有用的，但没有任何担保；甚至没有对适销性或特定用途适用性的暗示担保。有关更多详细信息，请参见 GNU 通用公共许可证。

  您应该已经收到了 GNU 通用公共许可证的副本，与 cpp-ethereum 一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
*/
/** @file internal.c
* @author Tim Hughes <tim@twistedfury.com>
* @author Matthew Wampler-Doty
* @date 2015
*/

#include <assert.h>  // 包含断言库
#include <inttypes.h>  // 包含整数类型库
#include <stddef.h>  // 包含标准定义库
#include <errno.h>  // 包含错误库
#include <math.h>  // 包含数学库
#include <stdlib.h>  // 包含标准库
#include "ethash.h"  // 包含 ethash 头文件
#include "fnv.h"  // 包含 fnv 头文件
#include "endian.h"  // 包含 endian 头文件
#include "ethash_internal.h"  // 包含 ethash_internal 头文件
#include "data_sizes.h"  // 包含 data_sizes 头文件
#include "base/crypto/sha3.h"  // 包含 sha3 头文件

#if defined(_M_X64) || defined(__x86_64__) || defined(__SSE2__)  // 如果定义了这些宏
    #ifdef __GNUC__  // 如果定义了 GNU C 编译器
        #include <x86intrin.h>  // 包含 x86intrin 头文件
    #else
        #include <intrin.h>  // 包含 intrin 头文件
    #endif

    #define kp_prefetch(x) _mm_prefetch((x), _MM_HINT_T0);  // 定义 kp_prefetch 宏
#else
    #define kp_prefetch(x)  // 定义 kp_prefetch 宏为空
#endif

#define SHA3_256(a, b, c) sha3_HashBuffer(256, SHA3_FLAGS_KECCAK, b, c, a, 32)  // 定义 SHA3_256 宏
#define SHA3_512(a, b, c) sha3_HashBuffer(512, SHA3_FLAGS_KECCAK, b, c, a, 64)  // 定义 SHA3_512 宏

uint64_t ethash_get_datasize(uint64_t const block_number)  // 定义函数 ethash_get_datasize
{
    assert(block_number / ETHASH_EPOCH_LENGTH < 2048);  // 断言，确保块号除以 ETHASH_EPOCH_LENGTH 小于 2048
    return dag_sizes[block_number / ETHASH_EPOCH_LENGTH];  // 返回 dag_sizes 中对应索引的值
}

uint64_t ethash_get_cachesize(uint64_t const block_number)  // 定义函数 ethash_get_cachesize
{
    assert(block_number / ETHASH_EPOCH_LENGTH < 2048);  // 断言，确保块号除以 ETHASH_EPOCH_LENGTH 小于 2048
    return cache_sizes[block_number / ETHASH_EPOCH_LENGTH];  // 返回 cache_sizes 中对应索引的值
}

// 遵循 Sergio 的 "STRICT MEMORY HARD HASHING FUNCTIONS"（2014）
// https://bitslog.files.wordpress.com/2013/12/memohash-v0-3.pdf
// SeqMemoHash(s, R, N)
# 计算以太坊缓存节点的布尔函数
bool ethash_compute_cache_nodes(
    void* nodes_ptr,  # 缓存节点的指针
    uint64_t cache_size,  # 缓存大小
    ethash_h256_t const* seed  # 以太坊种子哈希
)
{
    if (cache_size % sizeof(node) != 0) {  # 如果缓存大小不是节点大小的整数倍
        return false;  # 返回假
    }
    uint32_t const num_nodes = (uint32_t) (cache_size / sizeof(node));  # 计算节点数量

    node* nodes = (node*)nodes_ptr;  # 将节点指针转换为节点数组
    SHA3_512(nodes[0].bytes, (uint8_t*)seed, 32);  # 使用种子哈希计算节点的 SHA3-512 哈希值

    for (uint32_t i = 1; i != num_nodes; ++i) {  # 遍历节点数组
        SHA3_512(nodes[i].bytes, nodes[i - 1].bytes, 64);  # 使用前一个节点的哈希值计算当前节点的 SHA3-512 哈希值
    }

    for (uint32_t j = 0; j != ETHASH_CACHE_ROUNDS; j++) {  # 循环执行缓存轮数
        for (uint32_t i = 0; i != num_nodes; i++) {  # 遍历节点数组
            uint32_t const idx = nodes[i].words[0] % num_nodes;  # 计算节点索引
            node data;  # 创建节点数据
            data = nodes[(num_nodes - 1 + i) % num_nodes];  # 获取节点数据
            for (uint32_t w = 0; w != NODE_WORDS; ++w) {  # 遍历节点数据的字
                data.words[w] ^= nodes[idx].words[w];  # 对节点数据进行异或操作
            }
            SHA3_512(nodes[i].bytes, data.bytes, sizeof(data));  # 使用节点数据计算节点的 SHA3-512 哈希值
        }
    }

    // 现在执行端点转换
    fix_endian_arr32(nodes->words, num_nodes * NODE_WORDS);  # 执行端点转换
    return true;  # 返回真
}

void ethash_calculate_dag_item(
    node* const ret,  # DAG 项目的返回节点
    uint32_t node_index,  # 节点索引
    uint32_t num_parents,  # 父节点数量
    ethash_light_t const light  # 以太坊轻客户端
)
{
    uint32_t num_parent_nodes = (uint32_t) (light->cache_size / sizeof(node));  # 计算父节点数量
    node const* cache_nodes = (node const *) light->cache;  # 将轻客户端缓存转换为节点数组
    node const* init = &cache_nodes[node_index % num_parent_nodes];  # 获取初始节点
    memcpy(ret, init, sizeof(node));  # 复制初始节点到返回节点
    ret->words[0] ^= node_index;  # 对返回节点的第一个字进行异或操作
    SHA3_512(ret->bytes, ret->bytes, sizeof(node));  # 使用返回节点计算 SHA3-512 哈希值
#if defined(_M_X64) && ENABLE_SSE
    __m128i const fnv_prime = _mm_set1_epi32(FNV_PRIME);  # 设置 FNV 素数
    __m128i xmm0 = ret->xmm[0];  # 获取返回节点的 xmm0 寄存器
    __m128i xmm1 = ret->xmm[1];  # 获取返回节点的 xmm1 寄存器
    __m128i xmm2 = ret->xmm[2];  # 获取返回节点的 xmm2 寄存器
    __m128i xmm3 = ret->xmm[3];  # 获取返回节点的 xmm3 寄存器
#endif

    for (uint32_t i = 0; i != num_parents; ++i) {  # 遍历父节点
        uint32_t parent_index = fnv_hash(node_index ^ i, ret->words[i % NODE_WORDS]) % num_parent_nodes;  # 计算父节点索引
        node const *parent = &cache_nodes[parent_index];  # 获取父节点
#if defined(_M_X64) && ENABLE_SSE
        {
            # 如果定义了_M_X64并且启用了SSE，则执行以下代码块
            xmm0 = _mm_mullo_epi32(xmm0, fnv_prime);
            xmm1 = _mm_mullo_epi32(xmm1, fnv_prime);
            xmm2 = _mm_mullo_epi32(xmm2, fnv_prime);
            xmm3 = _mm_mullo_epi32(xmm3, fnv_prime);
            xmm0 = _mm_xor_si128(xmm0, parent->xmm[0]);
            xmm1 = _mm_xor_si128(xmm1, parent->xmm[1]);
            xmm2 = _mm_xor_si128(xmm2, parent->xmm[2]);
            xmm3 = _mm_xor_si128(xmm3, parent->xmm[3]);

            # 将值写入ret，因为这些值用于计算索引
            ret->xmm[0] = xmm0;
            ret->xmm[1] = xmm1;
            ret->xmm[2] = xmm2;
            ret->xmm[3] = xmm3;
        }
#else
        {
            # 如果未定义_M_X64或未启用SSE，则执行以下代码块
            for (unsigned w = 0; w != NODE_WORDS; ++w) {
                ret->words[w] = fnv_hash(ret->words[w], parent->words[w]);
            }
        }
#endif
    }
    SHA3_512(ret->bytes, ret->bytes, sizeof(node));
}

static inline uint32_t fast_mod(uint64_t a, uint64_t d, uint64_t r, uint64_t i, uint64_t s)
{
    # 快速计算取模运算
    const uint32_t q = ((a + i) * r) >> s;
    return a - q * d;
}

void ethash_calculate_dag_item_opt(
    node* const ret,
    uint32_t node_index,
    uint32_t num_parents,
    ethash_light_t const light
)
{
    # 从轻客户端缓存中获取节点数据
    node const* cache_nodes = (node const*)light->cache;
    node const* init = &cache_nodes[fast_mod(node_index, light->num_parent_nodes, light->reciprocal, light->increment, light->shift)];
    # 将init的内容复制到ret中
    memcpy(ret, init, sizeof(node));
    ret->words[0] ^= node_index;
    SHA3_512(ret->bytes, ret->bytes, sizeof(node));

    for (uint32_t i = 0; i != num_parents; ++i) {
        # 计算父节点的索引
        uint32_t parent_index = fast_mod(fnv_hash(node_index ^ i, ret->words[i % NODE_WORDS]), light->num_parent_nodes, light->reciprocal, light->increment, light->shift);
        node const* parent = &cache_nodes[parent_index];
        for (unsigned w = 0; w != NODE_WORDS; ++w) {
            # 计算哈希值
            ret->words[w] = fnv_hash(ret->words[w], parent->words[w]);
        }
    }
}
    # 使用 SHA3_512 算法对 ret->bytes 中的数据进行哈希计算
    SHA3_512(ret->bytes, ret->bytes, sizeof(node));
}

void ethash_calculate_dag_item4_opt(
    node* ret,  // 用于存储计算结果的节点指针
    uint32_t node_index,  // 当前节点的索引
    uint32_t num_parents,  // 父节点的数量
    ethash_light_t const light  // ethash_light_t 结构体
)
{
    node const* cache_nodes = (node const*)light->cache;  // 从 ethash_light_t 结构体中获取缓存节点数组的指针

    for (size_t i = 0; i < 4; ++i) {  // 循环4次
        node const* init = &cache_nodes[fast_mod(node_index + i, light->num_parent_nodes, light->reciprocal, light->increment, light->shift)];  // 计算初始节点的索引
        memcpy(ret + i, init, sizeof(node));  // 将初始节点的数据复制到结果节点数组中
        ret[i].words[0] ^= node_index + i;  // 对结果节点的第一个字进行异或操作
        SHA3_512(ret[i].bytes, ret[i].bytes, sizeof(node));  // 对结果节点进行 SHA3-512 哈希计算
    }

    for (uint32_t i = 0; i != num_parents; ++i) {  // 循环父节点的数量次
        node const* parent[4];  // 父节点数组

        for (uint32_t j = 0; j < 4; ++j) {  // 循环4次
            const uint32_t parent_index = fast_mod(fnv_hash((node_index + j) ^ i, ret[j].words[i % NODE_WORDS]), light->num_parent_nodes, light->reciprocal, light->increment, light->shift);  // 计算父节点的索引
            parent[j] = &cache_nodes[parent_index];  // 获取父节点的指针
            kp_prefetch(parent[j]);  // 预取父节点数据
        }

        for (unsigned w = 0; w != NODE_WORDS; ++w) ret[0].words[w] = fnv_hash(ret[0].words[w], parent[0]->words[w]);  // 对结果节点进行哈希计算
        for (unsigned w = 0; w != NODE_WORDS; ++w) ret[1].words[w] = fnv_hash(ret[1].words[w], parent[1]->words[w]);  // 对结果节点进行哈希计算
        for (unsigned w = 0; w != NODE_WORDS; ++w) ret[2].words[w] = fnv_hash(ret[2].words[w], parent[2]->words[w]);  // 对结果节点进行哈希计算
        for (unsigned w = 0; w != NODE_WORDS; ++w) ret[3].words[w] = fnv_hash(ret[3].words[w], parent[3]->words[w]);  // 对结果节点进行哈希计算
    }

    for (size_t i = 0; i < 4; ++i) {  // 循环4次
        SHA3_512(ret[i].bytes, ret[i].bytes, sizeof(node));  // 对结果节点进行 SHA3-512 哈希计算
    }
}

bool ethash_compute_full_data(
    void* mem,  // 内存指针
    uint64_t full_size,  // 数据的总大小
    ethash_light_t const light,  // ethash_light_t 结构体
    ethash_callback_t callback  // 回调函数
)
{
    if (full_size % (sizeof(uint32_t) * MIX_WORDS) != 0 ||  // 如果数据大小不能被 (sizeof(uint32_t) * MIX_WORDS) 整除
        (full_size % sizeof(node)) != 0) {  // 或者数据大小不能被 sizeof(node) 整除
        return false;  // 返回 false
    }
    uint32_t const max_n = (uint32_t)(full_size / sizeof(node));  // 计算最大节点数量
    node* full_nodes = (node*) mem;  // 将内存指针转换为节点指针
    double const progress_change = 1.0f / max_n;  // 进度变化值
    double progress = 0.0f;  // 初始进度为0
    // 计算完整节点
    for (uint32_t n = 0; n != max_n; ++n) {
        // 如果存在回调函数，并且达到了进度的百分之一，则调用回调函数
        if (callback &&
            n % (max_n / 100) == 0 &&
            callback((unsigned int)(ceil(progress * 100.0f))) != 0) {
            // 如果回调函数返回非零值，则返回 false
            return false;
        }
        // 更新进度
        progress += progress_change;
        // 计算 ETHASH 数据集的父节点，存储在 full_nodes 数组中
        ethash_calculate_dag_item(&(full_nodes[n]), n, ETHASH_DATASET_PARENTS, light);
    }
    // 循环结束后返回 true
    return true;
    # 定义一个静态函数，用于计算 ethash 哈希
    static bool ethash_hash(
        ethash_return_value_t* ret,  # 用于存储计算结果的返回值指针
        node const* full_nodes,  # 完整节点数据的指针
        ethash_light_t const light,  # ethash 轻客户端数据
        uint64_t full_size,  # 完整节点数据的大小
        ethash_h256_t const header_hash,  # 区块头哈希值
        uint64_t const nonce  # 区块的随机数
    )
    {
        # 如果完整节点数据大小不是 MIX_WORDS 的整数倍，则返回 false
        if (full_size % MIX_WORDS != 0) {
            return false;
        }

        // 将哈希和随机数打包到 s_mix 的前 40 个字节中
        assert(sizeof(node) * 8 == 512);
        node s_mix[MIX_NODES + 1];
        memcpy(s_mix[0].bytes, &header_hash, 32);
        fix_endian64(s_mix[0].double_words[4], nonce);

        // 计算 sha3-512 哈希并复制到 mix 中
        SHA3_512(s_mix->bytes, s_mix->bytes, 40);
        fix_endian_arr32(s_mix[0].words, 16);

        node* const mix = s_mix + 1;
        for (uint32_t w = 0; w != MIX_WORDS; ++w) {
            mix->words[w] = s_mix[0].words[w % NODE_WORDS];
        }

        unsigned const page_size = sizeof(uint32_t) * MIX_WORDS;
        unsigned const num_full_pages = (unsigned) (full_size / page_size);

        // 遍历 ETHASH_ACCESSES 次
        for (unsigned i = 0; i != ETHASH_ACCESSES; ++i) {
            // 计算索引值
            uint32_t const index = fnv_hash(s_mix->words[0] ^ i, mix->words[i % MIX_WORDS]) % num_full_pages;

            // 遍历 MIX_NODES 次
            for (unsigned n = 0; n != MIX_NODES; ++n) {
                node const* dag_node;
                node tmp_node;
                // 如果存在完整节点数据，则使用完整节点数据
                if (full_nodes) {
                    dag_node = &full_nodes[MIX_NODES * index + n];
                } else {
                    // 否则计算 DAG 数据
                    ethash_calculate_dag_item(&tmp_node, index * MIX_NODES + n, ETHASH_DATASET_PARENTS, light);
                    dag_node = &tmp_node;
                }
            }
        }
    }
#if defined(_M_X64) && ENABLE_SSE
            {
                // 如果定义了_M_X64并且启用了SSE，则使用SSE指令集进行计算
                __m128i fnv_prime = _mm_set1_epi32(FNV_PRIME);
                __m128i xmm0 = _mm_mullo_epi32(fnv_prime, mix[n].xmm[0]);
                __m128i xmm1 = _mm_mullo_epi32(fnv_prime, mix[n].xmm[1]);
                __m128i xmm2 = _mm_mullo_epi32(fnv_prime, mix[n].xmm[2]);
                __m128i xmm3 = _mm_mullo_epi32(fnv_prime, mix[n].xmm[3]);
                mix[n].xmm[0] = _mm_xor_si128(xmm0, dag_node->xmm[0]);
                mix[n].xmm[1] = _mm_xor_si128(xmm1, dag_node->xmm[1]);
                mix[n].xmm[2] = _mm_xor_si128(xmm2, dag_node->xmm[2]);
                mix[n].xmm[3] = _mm_xor_si128(xmm3, dag_node->xmm[3]);
            }
            #else
            {
                // 如果不满足上述条件，则使用循环计算
                for (unsigned w = 0; w != NODE_WORDS; ++w) {
                    mix[n].words[w] = fnv_hash(mix[n].words[w], dag_node->words[w]);
                }
            }
#endif
        }

    }

    // 压缩mix
    for (uint32_t w = 0; w != MIX_WORDS; w += 4) {
        uint32_t reduction = mix->words[w + 0];
        reduction = reduction * FNV_PRIME ^ mix->words[w + 1];
        reduction = reduction * FNV_PRIME ^ mix->words[w + 2];
        reduction = reduction * FNV_PRIME ^ mix->words[w + 3];
        mix->words[w / 4] = reduction;
    }

    fix_endian_arr32(mix->words, MIX_WORDS / 4);
    memcpy(&ret->mix_hash, mix->bytes, 32);
    // 最终Keccak哈希
    SHA3_256(&ret->result, s_mix->bytes, 64 + 32); // Keccak-256(s + compressed_mix)
    return true;
}

void ethash_quick_hash(
    ethash_h256_t* return_hash,
    ethash_h256_t const* header_hash,
    uint64_t nonce,
    ethash_h256_t const* mix_hash
)
{
    uint8_t buf[64 + 32];
    memcpy(buf, header_hash, 32);
    fix_endian64_same(nonce);
    memcpy(&(buf[32]), &nonce, 8);
    SHA3_512(buf, buf, 40);
    memcpy(&(buf[64]), mix_hash, 32);
    SHA3_256(return_hash, buf, 64 + 32);
}

ethash_h256_t ethash_get_seedhash(uint64_t epoch)
{
    ethash_h256_t ret;
    # 重置 ret 变量，使用 ethash_h256_reset 函数
    ethash_h256_reset(&ret);
    # 循环执行 epoch 次
    for (uint32_t i = 0; i < epoch; ++i)
        # 使用 SHA3_256 函数对 ret 变量进行哈希计算，结果覆盖原 ret 变量
        SHA3_256(&ret, (uint8_t*)&ret, 32);
    # 返回计算结果
    return ret;
# 检查以太坊区块的难度是否符合要求
bool ethash_quick_check_difficulty(
    ethash_h256_t const* header_hash,  # 区块头哈希值的指针
    uint64_t const nonce,  # 区块的随机数
    ethash_h256_t const* mix_hash,  # 混合哈希值的指针
    ethash_h256_t const* boundary  # 边界值的指针
)
{
    ethash_h256_t return_hash;  # 返回的哈希值
    ethash_quick_hash(&return_hash, header_hash, nonce, mix_hash);  # 使用区块头哈希值、随机数和混合哈希值计算返回哈希值
    return ethash_check_difficulty(&return_hash, boundary);  # 检查返回哈希值是否符合给定的边界值
}

# 创建一个新的以太坊轻客户端
ethash_light_t ethash_light_new_internal(uint64_t cache_size, ethash_h256_t const* seed)
{
    struct ethash_light *ret;  # 以太坊轻客户端的指针
    ret = (struct ethash_light*)calloc(sizeof(*ret), 1);  # 分配内存空间给以太坊轻客户端
    if (!ret) {  # 如果分配内存失败
        return NULL;  # 返回空指针
    }
    ret->cache = malloc((size_t)cache_size);  # 分配内存空间给缓存
    if (!ret->cache) {  # 如果分配内存失败
        goto fail_free_light;  # 跳转到释放以太坊轻客户端内存的标签
    }
    node* nodes = (node*)ret->cache;  # 将缓存转换为节点
    if (!ethash_compute_cache_nodes(nodes, cache_size, seed)) {  # 计算缓存节点
        goto fail_free_cache_mem;  # 如果计算失败，跳转到释放缓存内存的标签
    }
    ret->cache_size = cache_size;  # 设置缓存大小
    return ret;  # 返回以太坊轻客户端指针

fail_free_cache_mem:  # 释放缓存内存的标签
    free(ret->cache);  # 释放缓存内存
fail_free_light:  # 释放以太坊轻客户端内存的标签
    free(ret);  # 释放以太坊轻客户端内存
    return NULL;  # 返回空指针
}

# 创建一个新的以太坊轻客户端
ethash_light_t ethash_light_new(uint64_t block_number)
{
    ethash_h256_t seedhash = ethash_get_seedhash(block_number / ETHASH_EPOCH_LENGTH);  # 获取种子哈希值
    ethash_light_t ret;  # 以太坊轻客户端的指针
    ret = ethash_light_new_internal(ethash_get_cachesize(block_number), &seedhash);  # 创建新的以太坊轻客户端
    ret->block_number = block_number;  # 设置区块编号
    return ret;  # 返回以太坊轻客户端指针
}

# 删除以太坊轻客户端
void ethash_light_delete(ethash_light_t light)
{
    if (light->cache) {  # 如果缓存存在
        free(light->cache);  # 释放缓存内存
    }
    free(light);  # 释放以太坊轻客户端内存
}

# 计算以太坊轻客户端的哈希值
ethash_return_value_t ethash_light_compute_internal(
    ethash_light_t light,  # 以太坊轻客户端的指针
    uint64_t full_size,  # 完整大小
    ethash_h256_t const header_hash,  # 区块头哈希值的指针
    uint64_t nonce  # 随机数
)
{
    ethash_return_value_t ret;  # 返回值
    ret.success = true;  # 设置成功标志为真
    if (!ethash_hash(&ret, NULL, light, full_size, header_hash, nonce)) {  # 如果计算哈希值失败
        ret.success = false;  # 设置成功标志为假
    }
    return ret;  # 返回返回值
}

# 计算以太坊轻客户端的哈希值
ethash_return_value_t ethash_light_compute(
    ethash_light_t light,  # 以太坊轻客户端的指针
    ethash_h256_t const header_hash,  # 区块头哈希值的指针
    uint64_t nonce  # 随机数
)
{
    uint64_t full_size = ethash_get_datasize(light->block_number);  # 获取完整大小
    # 调用名为 ethash_light_compute_internal 的函数，传入参数 light, full_size, header_hash, nonce，并返回结果
    return ethash_light_compute_internal(light, full_size, header_hash, nonce);
# 计算完整的 ethash，返回计算结果
ethash_return_value_t ethash_full_compute(
    ethash_full_t full,  # ethash_full_t 类型的参数 full
    ethash_h256_t const header_hash,  # ethash_h256_t 类型的参数 header_hash
    uint64_t nonce  # uint64_t 类型的参数 nonce
)
{
    ethash_return_value_t ret;  # 声明一个 ethash_return_value_t 类型的变量 ret
    ret.success = true;  # 设置 ret 的 success 属性为 true
    if (!ethash_hash(  # 如果 ethash_hash 函数返回值为假
        &ret,  # 传入 ret 的地址
        (node const*)full->data,  # 将 full->data 转换为 node const* 类型
        NULL,  # 传入 NULL
        full->file_size,  # 传入 full->file_size
        header_hash,  # 传入 header_hash
        nonce)) {  # 传入 nonce
        ret.success = false;  # 设置 ret 的 success 属性为 false
    }
    return ret;  # 返回 ret
}

# 返回 ethash_full_t 类型的 full 的 data 属性
void const* ethash_full_dag(ethash_full_t full)
{
    return full->data;  # 返回 full 的 data 属性
}

# 返回 ethash_full_t 类型的 full 的 file_size 属性
uint64_t ethash_full_dag_size(ethash_full_t full)
{
    return full->file_size;  # 返回 full 的 file_size 属性
}
```