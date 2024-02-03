# `xmrig\src\base\tools\cryptonote\BlockTemplate.h`

```cpp
/*
 * XMRig
 * 版权所有 (c) 2012-2013 The Cryptonote developers
 * 版权所有 (c) 2014-2021 The Monero Project
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 * 无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有暗示的保证
 * 商品性或适用于特定目的。有关更多详细信息，请参见
 * GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_BLOCKTEMPLATE_H
#define XMRIG_BLOCKTEMPLATE_H


#include "3rdparty/rapidjson/fwd.h"
#include "base/crypto/Coin.h"
#include "base/tools/Buffer.h"
#include "base/tools/String.h"
#include "base/tools/Span.h"


namespace xmrig {


class BlockTemplate
{
public:
    static constexpr size_t kHashSize       = 32;
    static constexpr size_t kKeySize        = 32;
    static constexpr size_t kNonceSize      = 4;
    static constexpr size_t kSignatureSize  = 64;

#   ifdef XMRIG_PROXY_PROJECT
    static constexpr bool kCalcHashes       = true;
#   else
    static constexpr bool kCalcHashes       = false;
#   endif

    enum Offset : uint32_t {
        NONCE_OFFSET,
        MINER_TX_PREFIX_OFFSET,
        MINER_TX_PREFIX_END_OFFSET,
        EPH_PUBLIC_KEY_OFFSET,
        TX_EXTRA_OFFSET,
        TX_PUBKEY_OFFSET,
        TX_EXTRA_NONCE_OFFSET,
        TX_EXTRA_MERGE_MINING_TAG_OFFSET,
        OFFSET_COUNT
    };

    inline const Coin &coin() const                         { return m_coin; }
    // 返回数据块的指针
    inline const uint8_t *blob() const                      { return m_blob.data(); }
    // 返回偏移量处的数据块的指针
    inline const uint8_t *blob(Offset offset) const         { return m_blob.data() + m_offsets[offset]; }
    // 返回偏移量的大小
    inline size_t offset(Offset offset) const               { return m_offsets[offset]; }
    // 返回数据块的大小
    inline size_t size() const                              { return m_blob.size(); }

    // 区块头
    inline uint8_t majorVersion() const                     { return m_version.first; }
    // 返回次要版本号
    inline uint8_t minorVersion() const                     { return m_version.second; }
    // 返回时间戳
    inline uint64_t timestamp() const                       { return m_timestamp; }
    // 返回上一个区块的ID
    inline const Span &prevId() const                       { return m_prevId; }
    // 返回矿工的随机数
    inline const uint8_t *nonce() const                     { return blob(NONCE_OFFSET); }

    // Wownero矿工签名
    inline bool hasMinerSignature() const                   { return !m_minerSignature.empty(); }
    // 返回矿工签名
    inline const Span &minerSignature() const               { return m_minerSignature; }
    // 返回投票信息
    inline const uint8_t *vote() const                      { return m_vote; }

    // 矿工交易
    inline uint64_t txVersion() const                       { return m_txVersion; }
    // 返回解锁时间
    inline uint64_t unlockTime() const                      { return m_unlockTime; }
    // 返回输入数量
    inline uint64_t numInputs() const                       { return m_numInputs; }
    // 返回输入类型
    inline uint8_t inputType() const                        { return m_inputType; }
    // 返回高度
    inline uint64_t height() const                          { return m_height; }
    // 返回输出数量
    inline uint64_t numOutputs() const                      { return m_numOutputs; }
    // 返回金额
    inline uint64_t amount() const                          { return m_amount; }
    // 返回输出类型
    inline uint64_t outputType() const                      { return m_outputType; }
    // 返回临时公钥
    inline const Span &ephPublicKey() const                 { return m_ephPublicKey; }
    // 返回交易额外随机数
    inline const Span &txExtraNonce() const                 { return m_txExtraNonce; }
    // 返回交易合并挖矿标签的引用
    inline const Span &txMergeMiningTag() const             { return m_txMergeMiningTag; }

    // 交易哈希数量
    inline uint64_t numHashes() const                       { return m_numHashes; }
    // 返回哈希数组
    inline const Buffer &hashes() const                     { return m_hashes; }
    // 返回矿工交易默克尔树分支
    inline const Buffer &minerTxMerkleTreeBranch() const    { return m_minerTxMerkleTreeBranch; }
    // 返回根哈希
    inline const uint8_t *rootHash() const                  { return m_rootHash; }

    // 生成用于哈希计算的数据块
    inline Buffer generateHashingBlob() const
    {
        Buffer out;
        generateHashingBlob(out);

        return out;
    }

    // 计算矿工交易哈希
    static void calculateMinerTxHash(const uint8_t *prefix_begin, const uint8_t *prefix_end, uint8_t *hash);
    // 计算根哈希
    static void calculateRootHash(const uint8_t *prefix_begin, const uint8_t *prefix_end, const Buffer &miner_tx_merkle_tree_branch, uint8_t *root_hash);

    // 解析区块模板
    bool parse(const Buffer &blocktemplate, const Coin &coin, bool hashes = kCalcHashes);
    // 解析区块模板
    bool parse(const char *blocktemplate, size_t size, const Coin &coin, bool hashes);
    // 解析区块模板
    bool parse(const rapidjson::Value &blocktemplate, const Coin &coin, bool hashes = kCalcHashes);
    // 解析区块模板
    bool parse(const String &blocktemplate, const Coin &coin, bool hashes = kCalcHashes);
    // 计算默克尔树哈希
    void calculateMerkleTreeHash();
    // 生成用于哈希计算的数据块
    void generateHashingBlob(Buffer &out) const;
// 定义私有成员变量，表示最小大小为76
private:
    static constexpr size_t kMinSize = 76;

    // 设置偏移量对应的数值
    inline void setOffset(Offset offset, size_t value)  { m_offsets[offset] = static_cast<uint32_t>(value); }

    // 解析函数，传入参数表示是否需要计算哈希值
    bool parse(bool hashes);

    // 区块数据的缓冲区
    Buffer m_blob;
    // 区块的币种
    Coin m_coin;
    // 偏移量数组
    uint32_t m_offsets[OFFSET_COUNT]{};

    // 区块的版本号和子版本号
    std::pair<uint8_t, uint8_t> m_version;
    // 区块的时间戳
    uint64_t m_timestamp    = 0;
    // 上一个区块的ID
    Span m_prevId;

    // 矿工的签名
    Span m_minerSignature;
    // 投票信息
    uint8_t m_vote[2]{};

    // 交易的版本号和解锁时间
    uint64_t m_txVersion    = 0;
    uint64_t m_unlockTime   = 0;
    // 输入交易的数量
    uint64_t m_numInputs    = 0;
    // 输入交易的类型
    uint8_t m_inputType     = 0;
    // 区块的高度
    uint64_t m_height       = 0;
    // 输出交易的数量
    uint64_t m_numOutputs   = 0;
    // 交易金额
    uint64_t m_amount       = 0;
    // 输出交易的类型
    uint8_t m_outputType    = 0;
    // 临时公钥
    Span m_ephPublicKey;
    // 视图标签
    uint8_t m_viewTag       = 0;
    // 额外数据的大小
    uint64_t m_extraSize    = 0;
    // 交易额外的随机数
    Span m_txExtraNonce;
    // 交易合并挖矿标签
    Span m_txMergeMiningTag = 0;
    // 哈希值的数量
    uint64_t m_numHashes    = 0;
    // 哈希值的缓冲区
    Buffer m_hashes;
    // 矿工交易的默克尔树分支
    Buffer m_minerTxMerkleTreeBranch;
    // 区块的根哈希值
    uint8_t m_rootHash[kHashSize]{};
};


} /* namespace xmrig */


#endif /* XMRIG_BLOCKTEMPLATE_H */
```