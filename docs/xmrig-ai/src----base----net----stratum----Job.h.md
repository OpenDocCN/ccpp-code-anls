# `xmrig\src\base\net\stratum\Job.h`

```
/* XMRig
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
 *   本程序是自由软件：您可以重新发布或修改
 *   根据 GNU 通用公共许可证的条款，由
 *   自由软件基金会发布的版本 3 或
 *   （在您的选择下）任何更高版本。
 *
 *   本程序是希望它有用的分发
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多详情请参见
 *   GNU 通用公共许可证。
 *
 *   您应该已经收到 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_JOB_H
#define XMRIG_JOB_H


#include <cstddef>
#include <cstdint>


#include "base/crypto/Algorithm.h"
#include "base/tools/Buffer.h"
#include "base/tools/String.h"


namespace xmrig {


class Job
{
public:
    // 最大 blob 大小为 84（75 固定 + 9 可变），对齐到 96。https://github.com/xmrig/xmrig/issues/1 感谢 fireice-uk。
    // SECOR 增加了对 blob 大小的要求：https://github.com/xmrig/xmrig/issues/913
    // Haven（XHV）离岸增加了通过将 pricing_record 结构（192 字节）添加到 block_header 来增加要求。
    // 将其舍入到 408（136*3）以便在 OpenCL 中进行方便的 keccak 计算
    // 定义常量 kMaxBlobSize，表示 Blob 的最大大小为 408
    static constexpr const size_t kMaxBlobSize = 408;
    // 定义常量 kMaxSeedSize，表示 Seed 的最大大小为 32
    static constexpr const size_t kMaxSeedSize = 32;
    
    // 默认构造函数
    Job() = default;
    // 带参数的构造函数，参数包括是否为 Nicehash、算法和客户端 ID
    Job(bool nicehash, const Algorithm &algorithm, const String &clientId);
    
    // 拷贝构造函数，用于拷贝另一个 Job 对象
    inline Job(const Job &other)        { copy(other); }
    // 移动构造函数，用于移动另一个 Job 对象
    inline Job(Job &&other) noexcept    { move(std::move(other)); }
    
    // 默认析构函数
    ~Job() = default;
    
    // 判断当前 Job 对象是否与另一个 Job 对象相等
    bool isEqual(const Job &other) const;
    // 判断当前 Job 对象的 Blob 是否与另一个 Job 对象的 Blob 相等
    bool isEqualBlob(const Job &other) const;
    // 设置当前 Job 对象的 Blob
    bool setBlob(const char *blob);
    // 设置当前 Job 对象的 Seed Hash
    bool setSeedHash(const char *hash);
    // 设置当前 Job 对象的 Target
    bool setTarget(const char *target);
    // 设置当前 Job 对象的难度
    void setDiff(uint64_t diff);
    // 设置当前 Job 对象的 Signature Key
    void setSigKey(const char *sig_key);
    
    // 返回当前 Job 对象是否为 Nicehash
    inline bool isNicehash() const                      { return m_nicehash; }
    // 判断当前 Job 对象是否有效
    inline bool isValid() const                         { return (m_size > 0 && m_diff > 0) || !m_poolWallet.isEmpty(); }
    // 设置当前 Job 对象的 ID
    inline bool setId(const char *id)                   { return m_id = id; }
    // 返回当前 Job 对象的算法
    inline const Algorithm &algorithm() const           { return m_algorithm; }
    // 返回当前 Job 对象的 Seed
    inline const Buffer &seed() const                   { return m_seed; }
    // 返回当前 Job 对象的客户端 ID
    inline const String &clientId() const               { return m_clientId; }
    // 返回当前 Job 对象的 Extra Nonce
    inline const String &extraNonce() const             { return m_extraNonce; }
    // 返回当前 Job 对象的 ID
    inline const String &id() const                     { return m_id; }
    // 返回当前 Job 对象的矿池钱包地址
    inline const String &poolWallet() const             { return m_poolWallet; }
    // 返回当前 Job 对象的 Nonce
    inline const uint32_t *nonce() const                { return reinterpret_cast<const uint32_t*>(m_blob + nonceOffset()); }
    // 返回当前 Job 对象的 Blob
    inline const uint8_t *blob() const                  { return m_blob; }
    // 返回当前 Job 对象的 Nonce Offset
    int32_t nonceOffset() const;
    // 返回当前 Job 对象的 Nonce 大小
    inline size_t nonceSize() const                     { return (algorithm().family() == Algorithm::KAWPOW) ?  8 :  4; }
    // 返回当前 Job 对象的大小
    inline size_t size() const                          { return m_size; }
    // 返回当前 Job 对象的 Nonce
    inline uint32_t *nonce()                            { return reinterpret_cast<uint32_t*>(m_blob + nonceOffset()); }
    // 返回后端类型
    inline uint32_t backend() const                     { return m_backend; }
    // 返回难度
    inline uint64_t diff() const                        { return m_diff; }
    // 返回高度
    inline uint64_t height() const                      { return m_height; }
    // 返回随机数掩码
    inline uint64_t nonceMask() const                   { return isNicehash() ? 0xFFFFFFULL : (nonceSize() == sizeof(uint64_t) ? (static_cast<uint64_t>(-1LL) >> (extraNonce().size() * 4)) : 0xFFFFFFFFULL); }
    // 返回目标值
    inline uint64_t target() const                      { return m_target; }
    // 返回数据块
    inline uint8_t *blob()                              { return m_blob; }
    // 返回固定字节
    inline uint8_t fixedByte() const                    { return *(m_blob + 42); }
    // 返回索引
    inline uint8_t index() const                        { return m_index; }
    // 重置数据
    inline void reset()                                 { m_size = 0; m_diff = 0; }
    // 设置算法
    inline void setAlgorithm(const Algorithm::Id id)    { m_algorithm = id; }
    // 设置算法
    inline void setAlgorithm(const char *algo)          { m_algorithm = algo; }
    // 设置后端类型
    inline void setBackend(uint32_t backend)            { m_backend = backend; }
    // 设置客户端ID
    inline void setClientId(const String &id)           { m_clientId = id; }
    // 设置额外的随机数
    inline void setExtraNonce(const String &extraNonce) { m_extraNonce = extraNonce; }
    // 设置高度
    inline void setHeight(uint64_t height)              { m_height = height; }
    // 设置索引
    inline void setIndex(uint8_t index)                 { m_index = index; }
    // 设置矿池钱包地址
    inline void setPoolWallet(const String &poolWallet) { m_poolWallet = poolWallet; }
#   ifdef XMRIG_PROXY_PROJECT
    # 如果定义了 XMRIG_PROXY_PROJECT，则以下代码块有效

    # 返回原始数据块的指针
    inline char *rawBlob()                              { return m_rawBlob; }
    # 返回常量原始数据块的指针
    inline const char *rawBlob() const                  { return m_rawBlob; }
    # 返回常量原始目标值的指针
    inline const char *rawTarget() const                { return m_rawTarget; }
    # 返回常量原始种子哈希值的引用
    inline const String &rawSeedHash() const            { return m_rawSeedHash; }
    # 返回常量原始签名密钥的引用
    inline const String &rawSigKey() const              { return m_rawSigKey; }
#   endif

    # 将目标值转换为难度值
    static inline uint64_t toDiff(uint64_t target)      { return target ? (0xFFFFFFFFFFFFFFFFULL / target) : 0; }

    # 判断是否不等于另一个 Job 对象
    inline bool operator!=(const Job &other) const      { return !isEqual(other); }
    # 判断是否等于另一个 Job 对象
    inline bool operator==(const Job &other) const      { return isEqual(other); }
    # 赋值运算符重载，用于复制另一个 Job 对象
    inline Job &operator=(const Job &other)             { copy(other); return *this; }
    # 移动赋值运算符重载，用于移动另一个 Job 对象
    inline Job &operator=(Job &&other) noexcept         { move(std::move(other)); return *this; }

#   ifdef XMRIG_FEATURE_BENCHMARK
    # 如果定义了 XMRIG_FEATURE_BENCHMARK，则以下代码块有效

    # 返回基准测试大小
    inline uint32_t benchSize() const                   { return m_benchSize; }
    # 设置基准测试大小
    inline void setBenchSize(uint32_t size)             { m_benchSize = size; }
#   endif

#   ifdef XMRIG_PROXY_PROJECT
    # 如果定义了 XMRIG_PROXY_PROJECT，则以下代码块有效

    # 判断是否有视图标签
    inline bool hasViewTag() const                      { return m_hasViewTag; }

    # 设置花费密钥
    void setSpendSecretKey(const uint8_t* key);
    # 设置矿工交易
    void setMinerTx(const uint8_t* begin, const uint8_t* end, size_t minerTxEphPubKeyOffset, size_t minerTxPubKeyOffset, size_t minerTxExtraNonceOffset, size_t minerTxExtraNonceSize, const Buffer& minerTxMerkleTreeBranch, bool hasViewTag);
    # 在矿工交易中设置视图标签
    void setViewTagInMinerTx(uint8_t view_tag);
    # 在矿工交易中设置额外的随机数
    void setExtraNonceInMinerTx(uint32_t extra_nonce);
    # 生成签名数据
    void generateSignatureData(String& signatureData, uint8_t& view_tag) const;
    # 生成哈希数据块
    void generateHashingBlob(String& blob) const;
#   else
    # 如果未定义 XMRIG_PROXY_PROJECT，则以下代码块有效

    # 返回临时密钥
    inline const uint8_t* ephSecretKey() const { return m_hasMinerSignature ? m_ephSecretKey : nullptr; }

    # 设置临时密钥对
    inline void setEphemeralKeys(const uint8_t *pub_key, const uint8_t *sec_key)
    {
        // 设置 m_hasMinerSignature 为 true
        m_hasMinerSignature = true;
        // 将 pub_key 的内容复制到 m_ephPublicKey 中，大小为 m_ephSecretKey 的大小
        memcpy(m_ephPublicKey, pub_key, sizeof(m_ephSecretKey));
        // 将 sec_key 的内容复制到 m_ephSecretKey 中，大小为 m_ephSecretKey 的大小
        memcpy(m_ephSecretKey, sec_key, sizeof(m_ephSecretKey));
    }

    // 定义一个函数 generateMinerSignature，接受一个指向 blob 的指针，大小为 size，将结果存储在 out_sig 中
    void generateMinerSignature(const uint8_t* blob, size_t size, uint8_t* out_sig) const;
// 返回是否具有矿工签名
inline bool hasMinerSignature() const { return m_hasMinerSignature; }

// 获取交易数量
uint32_t getNumTransactions() const;

private:
// 复制另一个作业对象的数据
void copy(const Job &other);
// 移动另一个作业对象的数据
void move(Job &&other);

// 算法类型
Algorithm m_algorithm;
// 是否为 NiceHash
bool m_nicehash     = false;
// 种子数据
Buffer m_seed;
// 大小
size_t m_size       = 0;
// 客户端 ID
String m_clientId;
// 额外的 nonce
String m_extraNonce;
// ID
String m_id;
// 矿池钱包地址
String m_poolWallet;
// 后端
uint32_t m_backend  = 0;
// 难度
uint64_t m_diff     = 0;
// 高度
uint64_t m_height   = 0;
// 目标
uint64_t m_target   = 0;
// 数据块
uint8_t m_blob[kMaxBlobSize]{ 0 };
// 索引
uint8_t m_index     = 0;

#   ifdef XMRIG_PROXY_PROJECT
// 原始数据块
char m_rawBlob[kMaxBlobSize * 2 + 8]{};
// 原始目标
char m_rawTarget[24]{};
// 原始种子哈希
String m_rawSeedHash;
// 原始签名密钥
String m_rawSigKey;

// 矿工签名
uint8_t m_spendSecretKey[32]{};
uint8_t m_viewSecretKey[32]{};
uint8_t m_spendPublicKey[32]{};
uint8_t m_viewPublicKey[32]{};
mutable Buffer m_minerTxPrefix;
size_t m_minerTxEphPubKeyOffset = 0;
size_t m_minerTxPubKeyOffset = 0;
size_t m_minerTxExtraNonceOffset = 0;
size_t m_minerTxExtraNonceSize = 0;
Buffer m_minerTxMerkleTreeBranch;
bool m_hasViewTag = false;
#   else
// 矿工签名
uint8_t m_ephPublicKey[32]{};
uint8_t m_ephSecretKey[32]{};
#   endif

// 是否具有矿工签名
bool m_hasMinerSignature = false;

#   ifdef XMRIG_FEATURE_BENCHMARK
// 基准测试大小
uint32_t m_benchSize = 0;
#   endif
};


} /* namespace xmrig */


#endif /* XMRIG_JOB_H */
```