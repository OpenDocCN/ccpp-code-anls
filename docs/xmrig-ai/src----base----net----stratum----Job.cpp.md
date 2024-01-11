# `xmrig\src\base\net\stratum\Job.cpp`

```
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有 2019      Howard Chu  <https://github.com/hyc>
 * 版权所有 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 * 要么是许可证的第3版，要么（在您的选择下）是任何更高版本。
 *
 * 本程序是希望它有用，但没有任何担保；甚至没有暗示的担保
 * 商品性或适用于特定目的。有关更多详细信息，请参见
 * GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include <cassert>
#include <cstring>


#include "base/net/stratum/Job.h"
#include "base/tools/Alignment.h"
#include "base/tools/Buffer.h"
#include "base/tools/Cvt.h"
#include "base/tools/cryptonote/BlockTemplate.h"
#include "base/tools/cryptonote/Signatures.h"
#include "base/crypto/keccak.h"


// 构造函数，初始化成员变量
xmrig::Job::Job(bool nicehash, const Algorithm &algorithm, const String &clientId) :
    m_algorithm(algorithm),
    m_nicehash(nicehash),
    m_clientId(clientId)
{
}


// 判断两个Job对象是否相等
bool xmrig::Job::isEqual(const Job &other) const
{
    return m_id == other.m_id && m_clientId == other.m_clientId && isEqualBlob(other) && m_target == other.m_target;
}
// 检查两个 Job 对象的 m_size 和 m_blob 是否相等
bool xmrig::Job::isEqualBlob(const Job &other) const
{
    return (m_size == other.m_size) && (memcmp(m_blob, other.m_blob, m_size) == 0);
}

// 设置 Job 对象的 m_blob
bool xmrig::Job::setBlob(const char *blob)
{
    // 如果 blob 为空指针，则返回 false
    if (!blob) {
        return false;
    }

    // 计算 blob 的长度
    size_t size = strlen(blob);
    // 如果长度为奇数，则返回 false
    if (size % 2 != 0) {
        return false;
    }

    size /= 2;

    // 计算最小的 m_blob 大小
    const size_t minSize = nonceOffset() + nonceSize();
    // 如果 size 小于最小大小或者大于 m_blob 的大小，则返回 false
    if (size < minSize || size >= sizeof(m_blob)) {
        return false;
    }

    // 将 blob 转换为十六进制并存储到 m_blob 中
    if (!Cvt::fromHex(m_blob, sizeof(m_blob), blob, size * 2)) {
        return false;
    }

    // 如果读取到的 nonce 不为 0 且 m_nicehash 为 false，则将 m_nicehash 设置为 true
    if (readUnaligned(nonce()) != 0 && !m_nicehash) {
        m_nicehash = true;
    }

#   ifdef XMRIG_PROXY_PROJECT
    // 如果定义了 XMRIG_PROXY_PROJECT，则将 m_rawBlob 清零并将 blob 复制到 m_rawBlob 中
    memset(m_rawBlob, 0, sizeof(m_rawBlob));
    memcpy(m_rawBlob, blob, size * 2);
#   endif

    // 设置 m_size 为 size，并返回 true
    m_size = size;
    return true;
}

// 设置 Job 对象的 m_seed
bool xmrig::Job::setSeedHash(const char *hash)
{
    // 如果 hash 为空指针或者长度不等于 kMaxSeedSize * 2，则返回 false
    if (!hash || (strlen(hash) != kMaxSeedSize * 2)) {
        return false;
    }

#   ifdef XMRIG_PROXY_PROJECT
    // 如果定义了 XMRIG_PROXY_PROJECT，则将 m_rawSeedHash 设置为 hash
    m_rawSeedHash = hash;
#   endif

    // 将 hash 转换为十六进制并存储到 m_seed 中
    m_seed = Cvt::fromHex(hash, kMaxSeedSize * 2);

    // 如果 m_seed 不为空，则返回 true，否则返回 false
    return !m_seed.empty();
}

// 设置 Job 对象的 m_target
bool xmrig::Job::setTarget(const char *target)
{
    // 如果 target 为空指针，则返回 false
    if (!target) {
        return false;
    }

    // 将 target 转换为十六进制并存储到 raw 中，计算 raw 的长度
    const auto raw    = Cvt::fromHex(target, strlen(target));
    const size_t size = raw.size();

    // 根据 raw 的长度设置 m_target 的值
    if (size == 4) {
        m_target = 0xFFFFFFFFFFFFFFFFULL / (0xFFFFFFFFULL / uint64_t(*reinterpret_cast<const uint32_t *>(raw.data()));
    }
    else if (size == 8) {
        m_target = *reinterpret_cast<const uint64_t *>(raw.data());
    }
    else {
        return false;
    }

#   ifdef XMRIG_PROXY_PROJECT
    // 如果定义了 XMRIG_PROXY_PROJECT，则将 m_rawTarget 清零并将 target 复制到 m_rawTarget 中
    assert(sizeof(m_rawTarget) > (size * 2));
    memset(m_rawTarget, 0, sizeof(m_rawTarget));
    memcpy(m_rawTarget, target, std::min(size * 2, sizeof(m_rawTarget)));
#   endif

    // 根据 m_target 计算 m_diff，并返回 true
    m_diff = toDiff(m_target);
    return true;
}

// 设置 Job 对象的 m_diff 和 m_target
void xmrig::Job::setDiff(uint64_t diff)
{
    // 设置 m_diff 为 diff，设置 m_target 为对应的值
    m_diff   = diff;
    m_target = toDiff(diff);

#   ifdef XMRIG_PROXY_PROJECT
    // 使用Cvt::toHex函数将m_rawTarget中的数据转换为十六进制格式，并存储到m_target中
    Cvt::toHex(m_rawTarget, sizeof(m_rawTarget), reinterpret_cast<uint8_t *>(&m_target), sizeof(m_target));
#   endif
}


void xmrig::Job::setSigKey(const char *sig_key)
{
    # 定义 sig_key 的长度
    constexpr const size_t size = 64;

    # 如果 sig_key 为空或者长度不等于 size*2，则返回
    if (!sig_key || strlen(sig_key) != size * 2) {
        return;
    }

#   ifndef XMRIG_PROXY_PROJECT
    # 将 sig_key 转换成二进制数据
    const auto buf = Cvt::fromHex(sig_key, size * 2);
    # 如果转换后的数据长度等于 size，则设置临时密钥
    if (buf.size() == size) {
        setEphemeralKeys(buf.data(), buf.data() + 32);
    }
#   else
    # 否则，将原始 sig_key 存储起来
    m_rawSigKey = sig_key;
#   endif
}


int32_t xmrig::Job::nonceOffset() const
{
   # 获取算法族
   auto f = algorithm().family();
   # 根据不同的算法族返回不同的 nonce 偏移量
   if (f == Algorithm::KAWPOW)     return 32;
   if (f == Algorithm::GHOSTRIDER) return 76;
   return 39;
}

uint32_t xmrig::Job::getNumTransactions() const
{
    # 如果不是 CN 算法族或者随机算法族，则返回 0
    if (!(m_algorithm.isCN() || m_algorithm.family() == Algorithm::RANDOM_X)) {
        return 0;
    }

    uint32_t num_transactions = 0;

    # 计算预期的交易偏移量
    const size_t expected_tx_offset = (m_algorithm == Algorithm::RX_WOW) ? 141 : 75;

    # 如果大小在预期的交易偏移量范围内，则解析交易数量
    if ((m_size > expected_tx_offset) && (m_size <= expected_tx_offset + 4)) {
        for (size_t i = expected_tx_offset, k = 0; i < m_size; ++i, k += 7) {
            const uint8_t b = m_blob[i];
            num_transactions |= static_cast<uint32_t>(b & 0x7F) << k;
            if ((b & 0x80) == 0) {
                break;
            }
        }
    }

    return num_transactions;
}


void xmrig::Job::copy(const Job &other)
{
    # 复制其他属性的值
    m_algorithm  = other.m_algorithm;
    m_nicehash   = other.m_nicehash;
    m_size       = other.m_size;
    m_clientId   = other.m_clientId;
    m_id         = other.m_id;
    m_backend    = other.m_backend;
    m_diff       = other.m_diff;
    m_height     = other.m_height;
    m_target     = other.m_target;
    m_index      = other.m_index;
    m_seed       = other.m_seed;
    m_extraNonce = other.m_extraNonce;
    m_poolWallet = other.m_poolWallet;

    # 复制 m_blob 数组的值
    memcpy(m_blob, other.m_blob, sizeof(m_blob));

#   ifdef XMRIG_PROXY_PROJECT
    # 复制 m_rawSeedHash 的值
    m_rawSeedHash = other.m_rawSeedHash;
    # 将对象 other 的 m_rawSigKey 的值赋给当前对象的 m_rawSigKey
    m_rawSigKey   = other.m_rawSigKey;

    # 将对象 other 的 m_rawBlob 的值拷贝到当前对象的 m_rawBlob 中，拷贝的长度为 m_rawBlob 的大小
    memcpy(m_rawBlob, other.m_rawBlob, sizeof(m_rawBlob));
    # 将对象 other 的 m_rawTarget 的值拷贝到当前对象的 m_rawTarget 中，拷贝的长度为 m_rawTarget 的大小
    memcpy(m_rawTarget, other.m_rawTarget, sizeof(m_rawTarget));
#   endif
#   ifdef XMRIG_FEATURE_BENCHMARK
    # 如果定义了 XMRIG_FEATURE_BENCHMARK，则将当前对象的 m_benchSize 设置为其他对象的 m_benchSize
    m_benchSize = other.m_benchSize;
#   endif
#   ifdef XMRIG_PROXY_PROJECT
    # 如果定义了 XMRIG_PROXY_PROJECT，则将其他对象的 m_spendSecretKey 复制到当前对象的 m_spendSecretKey
    memcpy(m_spendSecretKey, other.m_spendSecretKey, sizeof(m_spendSecretKey));
    # 将其他对象的 m_viewSecretKey 复制到当前对象的 m_viewSecretKey
    memcpy(m_viewSecretKey, other.m_viewSecretKey, sizeof(m_viewSecretKey));
    # 将其他对象的 m_spendPublicKey 复制到当前对象的 m_spendPublicKey
    memcpy(m_spendPublicKey, other.m_spendPublicKey, sizeof(m_spendPublicKey));
    # 将其他对象的 m_viewPublicKey 复制到当前对象的 m_viewPublicKey
    memcpy(m_viewPublicKey, other.m_viewPublicKey, sizeof(m_viewPublicKey));
    # 将其他对象的 m_minerTxPrefix 复制到当前对象的 m_minerTxPrefix
    m_minerTxPrefix = other.m_minerTxPrefix;
    # 将其他对象的 m_minerTxEphPubKeyOffset 复制到当前对象的 m_minerTxEphPubKeyOffset
    m_minerTxEphPubKeyOffset = other.m_minerTxEphPubKeyOffset;
    # 将其他对象的 m_minerTxPubKeyOffset 复制到当前对象的 m_minerTxPubKeyOffset
    m_minerTxPubKeyOffset = other.m_minerTxPubKeyOffset;
    # 将其他对象的 m_minerTxExtraNonceOffset 复制到当前对象的 m_minerTxExtraNonceOffset
    m_minerTxExtraNonceOffset = other.m_minerTxExtraNonceOffset;
    # 将其他对象的 m_minerTxExtraNonceSize 复制到当前对象的 m_minerTxExtraNonceSize
    m_minerTxExtraNonceSize = other.m_minerTxExtraNonceSize;
    # 将其他对象的 m_minerTxMerkleTreeBranch 复制到当前对象的 m_minerTxMerkleTreeBranch
    m_minerTxMerkleTreeBranch = other.m_minerTxMerkleTreeBranch;
    # 将其他对象的 m_hasViewTag 复制到当前对象的 m_hasViewTag
    m_hasViewTag = other.m_hasViewTag;
#   else
    # 如果未定义 XMRIG_PROXY_PROJECT，则将其他对象的 m_ephPublicKey 复制到当前对象的 m_ephPublicKey
    memcpy(m_ephPublicKey, other.m_ephPublicKey, sizeof(m_ephPublicKey));
    # 将其他对象的 m_ephSecretKey 复制到当前对象的 m_ephSecretKey
    memcpy(m_ephSecretKey, other.m_ephSecretKey, sizeof(m_ephSecretKey));
#   endif
    # 将其他对象的 m_hasMinerSignature 复制到当前对象的 m_hasMinerSignature
    m_hasMinerSignature = other.m_hasMinerSignature;
}

# 将其他对象的属性移动到当前对象
void xmrig::Job::move(Job &&other)
{
    # 将其他对象的 m_algorithm 移动到当前对象的 m_algorithm
    m_algorithm  = other.m_algorithm;
    # 将其他对象的 m_nicehash 移动到当前对象的 m_nicehash
    m_nicehash   = other.m_nicehash;
    # 将其他对象的 m_size 移动到当前对象的 m_size
    m_size       = other.m_size;
    # 将其他对象的 m_clientId 移动到当前对象的 m_clientId
    m_clientId   = std::move(other.m_clientId);
    # 将其他对象的 m_id 移动到当前对象的 m_id
    m_id         = std::move(other.m_id);
    # 将其他对象的 m_backend 移动到当前对象的 m_backend
    m_backend    = other.m_backend;
    # 将其他对象的 m_diff 移动到当前对象的 m_diff
    m_diff       = other.m_diff;
    # 将其他对象的 m_height 移动到当前对象的 m_height
    m_height     = other.m_height;
    # 将其他对象的 m_target 移动到当前对象的 m_target
    m_target     = other.m_target;
    # 将其他对象的 m_index 移动到当前对象的 m_index
    m_index      = other.m_index;
    # 将其他对象的 m_seed 移动到当前对象的 m_seed
    m_seed       = std::move(other.m_seed);
    # 将其他对象的 m_extraNonce 移动到当前对象的 m_extraNonce
    m_extraNonce = std::move(other.m_extraNonce);
    # 将其他对象的 m_poolWallet 移动到当前对象的 m_poolWallet
    m_poolWallet = std::move(other.m_poolWallet);
    # 将其他对象的 m_blob 复制到当前对象的 m_blob
    memcpy(m_blob, other.m_blob, sizeof(m_blob));
    # 将其他对象的 m_size 设置为 0
    other.m_size        = 0;
    # 将其他对象的 m_diff 设置为 0
    other.m_diff        = 0;
    # 将其他对象的 m_algorithm 设置为 Algorithm::INVALID
    other.m_algorithm   = Algorithm::INVALID;
#   ifdef XMRIG_PROXY_PROJECT
    # 将其他对象的 m_rawSeedHash 移动到当前对象的 m_rawSeedHash
    m_rawSeedHash = std::move(other.m_rawSeedHash);
    # 将其他对象的 m_rawSigKey 移动到当前对象的 m_rawSigKey
    m_rawSigKey   = std::move(other.m_rawSigKey);
    # 将其他对象的 m_rawBlob 复制到当前对象的 m_rawBlob
    memcpy(m_rawBlob, other.m_rawBlob, sizeof(m_rawBlob));
    // 使用 memcpy 函数将 other 对象的 m_rawTarget 数组内容复制到当前对象的 m_rawTarget 数组中
    memcpy(m_rawTarget, other.m_rawTarget, sizeof(m_rawTarget));
#   endif
#   ifdef XMRIG_FEATURE_BENCHMARK
    // 如果定义了 XMRIG_FEATURE_BENCHMARK，则将当前对象的 m_benchSize 设置为其他对象的 m_benchSize
    m_benchSize = other.m_benchSize;
#   endif
#   ifdef XMRIG_PROXY_PROJECT
    // 如果定义了 XMRIG_PROXY_PROJECT，则将其他对象的数据复制到当前对象
    memcpy(m_spendSecretKey, other.m_spendSecretKey, sizeof(m_spendSecretKey));
    memcpy(m_viewSecretKey, other.m_viewSecretKey, sizeof(m_viewSecretKey));
    memcpy(m_spendPublicKey, other.m_spendPublicKey, sizeof(m_spendPublicKey));
    memcpy(m_viewPublicKey, other.m_viewPublicKey, sizeof(m_viewPublicKey));
    // 移动或复制其他对象的数据到当前对象
    m_minerTxPrefix             = std::move(other.m_minerTxPrefix);
    m_minerTxEphPubKeyOffset    = other.m_minerTxEphPubKeyOffset;
    m_minerTxPubKeyOffset       = other.m_minerTxPubKeyOffset;
    m_minerTxExtraNonceOffset   = other.m_minerTxExtraNonceOffset;
    m_minerTxExtraNonceSize     = other.m_minerTxExtraNonceSize;
    m_minerTxMerkleTreeBranch   = std::move(other.m_minerTxMerkleTreeBranch);
    m_hasViewTag                = other.m_hasViewTag;
#   else
    // 如果未定义 XMRIG_PROXY_PROJECT，则将其他对象的数据复制到当前对象
    memcpy(m_ephPublicKey, other.m_ephPublicKey, sizeof(m_ephPublicKey));
    memcpy(m_ephSecretKey, other.m_ephSecretKey, sizeof(m_ephSecretKey);
#   endif
    // 将其他对象的 m_hasMinerSignature 复制到当前对象
    m_hasMinerSignature = other.m_hasMinerSignature;
}

#ifdef XMRIG_PROXY_PROJECT
// 设置当前对象的 m_spendSecretKey 为给定的密钥，并根据该密钥计算其他相关密钥
void xmrig::Job::setSpendSecretKey(const uint8_t *key)
{
    m_hasMinerSignature = true;
    memcpy(m_spendSecretKey, key, sizeof(m_spendSecretKey));
    derive_view_secret_key(m_spendSecretKey, m_viewSecretKey);
    secret_key_to_public_key(m_spendSecretKey, m_spendPublicKey);
    secret_key_to_public_key(m_viewSecretKey, m_viewPublicKey);
}

// 设置当前对象的矿工交易相关数据
void xmrig::Job::setMinerTx(const uint8_t *begin, const uint8_t *end, size_t minerTxEphPubKeyOffset, size_t minerTxPubKeyOffset, size_t minerTxExtraNonceOffset, size_t minerTxExtraNonceSize, const Buffer &minerTxMerkleTreeBranch, bool hasViewTag)
{
    m_minerTxPrefix.assign(begin, end);
    m_minerTxEphPubKeyOffset    = minerTxEphPubKeyOffset;
    m_minerTxPubKeyOffset       = minerTxPubKeyOffset;
    m_minerTxExtraNonceOffset   = minerTxExtraNonceOffset;
    # 设置矿工交易额外随机数大小
    m_minerTxExtraNonceSize = minerTxExtraNonceSize;
    # 设置矿工交易默克尔树分支
    m_minerTxMerkleTreeBranch = minerTxMerkleTreeBranch;
    # 设置是否有视图标签
    m_hasViewTag = hasViewTag;
# 设置矿工交易中的视图标签
void xmrig::Job::setViewTagInMinerTx(uint8_t view_tag)
{
    # 将视图标签复制到矿工交易前缀中的指定位置
    memcpy(m_minerTxPrefix.data() + m_minerTxEphPubKeyOffset + 32, &view_tag, 1);
}


# 在矿工交易中设置额外的随机数
void xmrig::Job::setExtraNonceInMinerTx(uint32_t extra_nonce)
{
    # 将额外的随机数复制到矿工交易前缀中的指定位置
    memcpy(m_minerTxPrefix.data() + m_minerTxExtraNonceOffset, &extra_nonce, std::min(m_minerTxExtraNonceSize, sizeof(uint32_t)));
}


# 生成签名数据
void xmrig::Job::generateSignatureData(String &signatureData, uint8_t& view_tag) const
{
    # 获取矿工交易前缀中的临时公钥和交易公钥
    uint8_t* eph_public_key = m_minerTxPrefix.data() + m_minerTxEphPubKeyOffset;
    uint8_t* txkey_pub = m_minerTxPrefix.data() + m_minerTxPubKeyOffset;

    # 生成交易密钥对
    uint8_t txkey_sec[32];
    generate_keys(txkey_pub, txkey_sec);

    # 生成密钥派生
    uint8_t derivation[32];
    generate_key_derivation(m_viewPublicKey, txkey_sec, derivation, &view_tag);
    derive_public_key(derivation, 0, m_spendPublicKey, eph_public_key);

    # 复制数据到缓冲区
    uint8_t buf[32 * 3] = {};
    memcpy(buf, txkey_pub, 32);
    memcpy(buf + 32, eph_public_key, 32);

    # 生成密钥派生
    generate_key_derivation(txkey_pub, m_viewSecretKey, derivation, nullptr);
    derive_secret_key(derivation, 0, m_spendSecretKey, buf + 64);

    # 将缓冲区中的数据转换为十六进制字符串
    signatureData = Cvt::toHex(buf, sizeof(buf));
}


# 生成哈希计算块
void xmrig::Job::generateHashingBlob(String &blob) const
{
    # 计算根哈希
    uint8_t root_hash[32];
    const uint8_t* p = m_minerTxPrefix.data();
    BlockTemplate::calculateRootHash(p, p + m_minerTxPrefix.size(), m_minerTxMerkleTreeBranch, root_hash);

    # 计算根哈希的偏移量
    uint64_t root_hash_offset = nonceOffset() + nonceSize();

    # 如果存在矿工签名，则需要调整根哈希的偏移量
    if (m_hasMinerSignature) {
        root_hash_offset += BlockTemplate::kSignatureSize + 2 /* vote */;
    }

    # 获取原始数据块，并将根哈希插入到指定位置
    blob = rawBlob();
    Cvt::toHex(blob.data() + root_hash_offset * 2, 64, root_hash, BlockTemplate::kHashSize);
}


# 生成矿工签名
void xmrig::Job::generateMinerSignature(const uint8_t* blob, size_t size, uint8_t* out_sig) const
{
    # 复制原始数据块到临时缓冲区
    uint8_t tmp[kMaxBlobSize];
    memcpy(tmp, blob, size);

    # 将签名填充为零
    memset(tmp + nonceOffset() + nonceSize(), 0, BlockTemplate::kSignatureSize);

    # 计算前缀哈希
    uint8_t prefix_hash[32];
    # 使用 xmrig 库中的 keccak 函数对 tmp 进行哈希计算，结果存储在 prefix_hash 中
    xmrig::keccak(tmp, static_cast<int>(size), prefix_hash, sizeof(prefix_hash));
    # 使用 xmrig 库中的 generate_signature 函数生成签名，使用 m_ephPublicKey 和 m_ephSecretKey，结果存储在 out_sig 中
    xmrig::generate_signature(prefix_hash, m_ephPublicKey, m_ephSecretKey, out_sig);
# 结束 C 语言中的条件编译指令
#endif
```