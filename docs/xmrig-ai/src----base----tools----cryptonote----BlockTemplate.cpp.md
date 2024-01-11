# `xmrig\src\base\tools\cryptonote\BlockTemplate.cpp`

```
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
 * 适销性或特定用途的保证。更多详情请参见
 * GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include "base/tools/cryptonote/BlockTemplate.h"
#include "3rdparty/rapidjson/document.h"
#include "base/crypto/keccak.h"
#include "base/tools/cryptonote/BlobReader.h"
#include "base/tools/Cvt.h"

void xmrig::BlockTemplate::calculateMinerTxHash(const uint8_t *prefix_begin, const uint8_t *prefix_end, uint8_t *hash)
{
    uint8_t hashes[kHashSize * 3];

    // 计算 3 个部分哈希

    // 1. 前缀
    keccak(prefix_begin, static_cast<int>(prefix_end - prefix_begin), hashes, kHashSize);

    // 2. 基本 RCT，矿工交易中的单个 0 字节
    static const uint8_t known_second_hash[kHashSize] = {
        188,54,120,158,122,30,40,20,54,70,66,41,130,143,129,125,102,18,247,180,119,214,101,145,255,150,169,224,100,188,201,138
    };
    memcpy(hashes + kHashSize, known_second_hash, kHashSize);

    // 3. 可剪切 RCT，在矿工交易中为空
    memset(hashes + kHashSize * 2, 0, kHashSize);

    // 计算矿工交易哈希
    keccak(hashes, sizeof(hashes), hash, kHashSize);
}
// 计算区块模板的根哈希值，参数包括前缀数据范围、矿工交易默克尔树分支、根哈希值
void xmrig::BlockTemplate::calculateRootHash(const uint8_t *prefix_begin, const uint8_t *prefix_end, const Buffer &miner_tx_merkle_tree_branch, uint8_t *root_hash)
{
    // 计算矿工交易哈希值
    calculateMinerTxHash(prefix_begin, prefix_end, root_hash);

    // 遍历矿工交易默克尔树分支，计算根哈希值
    for (size_t i = 0; i < miner_tx_merkle_tree_branch.size(); i += kHashSize) {
        uint8_t h[kHashSize * 2];

        // 复制根哈希值和默克尔树分支的哈希值到临时数组
        memcpy(h, root_hash, kHashSize);
        memcpy(h + kHashSize, miner_tx_merkle_tree_branch.data() + i, kHashSize);

        // 使用 Keccak 算法计算哈希值
        keccak(h, kHashSize * 2, root_hash, kHashSize);
    }
}

// 计算默克尔树哈希值
void xmrig::BlockTemplate::calculateMerkleTreeHash()
{
    // 清空矿工交易默克尔树分支
    m_minerTxMerkleTreeBranch.clear();

    // 获取哈希数量
    const uint64_t count = m_numHashes + 1;
    const uint8_t *h = m_hashes.data();

    // 如果哈希数量为1，直接复制哈希值到根哈希值
    if (count == 1) {
        memcpy(m_rootHash, h, kHashSize);
    }
    // 如果哈希数量为2，计算矿工交易默克尔树分支并计算根哈希值
    else if (count == 2) {
        m_minerTxMerkleTreeBranch.insert(m_minerTxMerkleTreeBranch.end(), h + kHashSize, h + kHashSize * 2);
        keccak(h, kHashSize * 2, m_rootHash, kHashSize);
    }
}
    else {
        // 初始化变量 i, j, cnt
        size_t i    = 0;
        size_t j    = 0;
        size_t cnt  = 0;

        // 循环计算 cnt 的值，使其成为大于等于 count 的最小的 2 的幂次方
        for (i = 0, cnt = 1; cnt <= count; ++i, cnt <<= 1) {}

        // 将 cnt 除以 2，得到上一步计算的值的一半
        cnt >>= 1;

        // 预留空间，用于存储哈希值
        m_minerTxMerkleTreeBranch.reserve(kHashSize * (i - 1));

        // 创建缓冲区 ints，拷贝哈希值数据
        Buffer ints(cnt * kHashSize);
        memcpy(ints.data(), h, (cnt * 2 - count) * kHashSize);

        // 循环计算哈希值
        for (i = cnt * 2 - count, j = cnt * 2 - count; j < cnt; i += 2, ++j) {
            // 如果 i 等于 0，则插入哈希值到 m_minerTxMerkleTreeBranch
            if (i == 0) {
                m_minerTxMerkleTreeBranch.insert(m_minerTxMerkleTreeBranch.end(), h + kHashSize, h + kHashSize * 2);
            }
            // 计算哈希值
            keccak(h + i * kHashSize, kHashSize * 2, ints.data() + j * kHashSize, kHashSize);
        }

        // 循环计算哈希值，直到 cnt 小于等于 2
        while (cnt > 2) {
            cnt >>= 1;
            for (i = 0, j = 0; j < cnt; i += 2, ++j) {
                // 如果 i 等于 0，则插入哈希值到 m_minerTxMerkleTreeBranch
                if (i == 0) {
                    m_minerTxMerkleTreeBranch.insert(m_minerTxMerkleTreeBranch.end(), ints.data() + kHashSize, ints.data() + kHashSize * 2);
                }
                // 计算哈希值
                keccak(ints.data() + i * kHashSize, kHashSize * 2, ints.data() + j * kHashSize, kHashSize);
            }
        }

        // 插入哈希值到 m_minerTxMerkleTreeBranch
        m_minerTxMerkleTreeBranch.insert(m_minerTxMerkleTreeBranch.end(), ints.data() + kHashSize, ints.data() + kHashSize * 2);
        // 计算根哈希值
        keccak(ints.data(), kHashSize * 2, m_rootHash, kHashSize);
    }
# 解析区块模板的内容，根据给定的区块模板、货币类型和哈希值标志
bool xmrig::BlockTemplate::parse(const Buffer &blocktemplate, const Coin &coin, bool hashes)
{
    # 如果区块模板的大小小于最小大小要求，则返回 false
    if (blocktemplate.size() < kMinSize) {
        return false;
    }

    # 将区块模板赋值给成员变量 m_blob
    m_blob  = blocktemplate;
    # 将货币类型赋值给成员变量 m_coin
    m_coin  = coin;
    # 初始化返回值为 false
    bool rc = false;

    try {
        # 调用 parse 函数解析区块模板内容，并根据哈希值标志进行处理
        rc = parse(hashes);
    } catch (...) {}

    # 返回解析结果
    return rc;
}

# 解析区块模板的内容，根据给定的区块模板数据、大小、货币类型和哈希值标志
bool xmrig::BlockTemplate::parse(const char *blocktemplate, size_t size, const Coin &coin, bool hashes)
{
    # 如果区块模板的大小小于最小大小的两倍，或者无法将十六进制转换为二进制，则返回 false
    if (size < (kMinSize * 2) || !Cvt::fromHex(m_blob, blocktemplate, size)) {
        return false;
    }

    # 将货币类型赋值给成员变量 m_coin
    m_coin  = coin;
    # 初始化返回值为 false
    bool rc = false;

    try {
        # 调用 parse 函数解析区块模板内容，并根据哈希值标志进行处理
        rc = parse(hashes);
    } catch (...) {}

    # 返回解析结果
    return rc;
}

# 解析区块模板的内容，根据给定的区块模板 JSON 对象、货币类型和哈希值标志
bool xmrig::BlockTemplate::parse(const rapidjson::Value &blocktemplate, const Coin &coin, bool hashes)
{
    # 判断区块模板是否为字符串，并调用相应的 parse 函数进行解析
    return blocktemplate.IsString() && parse(blocktemplate.GetString(), blocktemplate.GetStringLength(), coin, hashes);
}

# 解析区块模板的内容，根据给定的区块模板字符串、货币类型和哈希值标志
bool xmrig::BlockTemplate::parse(const String &blocktemplate, const Coin &coin, bool hashes)
{
    # 调用相应的 parse 函数解析区块模板内容
    return parse(blocktemplate.data(), blocktemplate.size(), coin, hashes);
}

# 生成用于哈希计算的区块模板内容
void xmrig::BlockTemplate::generateHashingBlob(Buffer &out) const
{
    # 清空输出缓冲区
    out.clear();
    # 预留输出缓冲区的空间
    out.reserve(offset(MINER_TX_PREFIX_OFFSET) + kHashSize + 3);

    # 将区块模板的部分内容添加到输出缓冲区
    out.assign(m_blob.begin(), m_blob.begin() + offset(MINER_TX_PREFIX_OFFSET));
    # 将根哈希值添加到输出缓冲区
    out.insert(out.end(), m_rootHash, m_rootHash + kHashSize);

    # 计算并添加 k 值到输出缓冲区
    uint64_t k = m_numHashes + 1;
    while (k >= 0x80) {
        out.emplace_back((static_cast<uint8_t>(k) & 0x7F) | 0x80);
        k >>= 7;
    }
    out.emplace_back(static_cast<uint8_t>(k));
}

# 解析区块模板的内容，根据给定的哈希值标志
bool xmrig::BlockTemplate::parse(bool hashes)
{
    # 创建 BlobReader 对象，用于读取区块模板的内容
    BlobReader<true> ar(m_blob.data(), m_blob.size());

    # 读取区块头部信息
    ar(m_version.first);
    ar(m_version.second);
    ar(m_timestamp);
    ar(m_prevId, kHashSize);
    # 设置偏移量并跳过 nonce 部分
    setOffset(NONCE_OFFSET, ar.index());
    ar.skip(kNonceSize);

    # Wownero 区块模板从版本 18 开始包含矿工签名
    // 如果币种为WOWNERO并且主版本号大于等于18
    if (m_coin == Coin::WOWNERO && majorVersion() >= 18) {
        // 序列化矿工签名和投票信息
        ar(m_minerSignature, kSignatureSize);
        ar(m_vote);
    }

    // 如果币种为ZEPHYR
    if (m_coin == Coin::ZEPHYR) {
        // 创建一个长度为120的pricing_record数组，并进行序列化
        uint8_t pricing_record[120];
        ar(pricing_record);
    }

    // 矿工交易开始
    // 前缀开始
    // 设置偏移量为矿工交易前缀的偏移量，序列化交易版本、解锁时间和输入数量
    setOffset(MINER_TX_PREFIX_OFFSET, ar.index());
    ar(m_txVersion);
    ar(m_unlockTime);
    ar(m_numInputs);

    // 输入数量必须为1
    if (m_numInputs != 1) {
        return false;
    }

    // 序列化输入类型
    ar(m_inputType);

    // 输入类型必须为txin_gen (0xFF)
    if (m_inputType != 0xFF) {
        return false;
    }

    // 序列化高度和输出数量
    ar(m_height);
    ar(m_numOutputs);

    // 如果币种为ZEPHYR
    if (m_coin == Coin::ZEPHYR) {
        // 如果输出数量小于2，则返回false
        if (m_numOutputs < 2) {
            return false;
        }
    }
    // 如果币种不为ZEPHYR，输出数量必须为1
    else if (m_numOutputs != 1) {
        return false;
    }

    // 序列化金额和输出类型
    ar(m_amount);
    ar(m_outputType);

    // 输出类型必须为txout_to_key (2)或txout_to_tagged_key (3)
    if ((m_outputType != 2) && (m_outputType != 3)) {
        return false;
    }

    // 设置偏移量为临时公钥的偏移量，序列化临时公钥
    setOffset(EPH_PUBLIC_KEY_OFFSET, ar.index());
    ar(m_ephPublicKey, kKeySize);

    // 如果币种为ZEPHYR
    if (m_coin == Coin::ZEPHYR) {
        // 如果输出类型不为2，则返回false
        if (m_outputType != 2) {
            return false;
        }

        // 序列化资产类型长度和跳过相应长度的数据
        uint64_t asset_type_len;
        ar(asset_type_len);
        ar.skip(asset_type_len);
        // 序列化视图标签
        ar(m_viewTag);

        // 遍历输出数量，序列化金额、输出类型、公钥和跳过相应长度的数据
        for (uint64_t k = 1; k < m_numOutputs; ++k) {
            uint64_t amount2;
            ar(amount2);

            uint8_t output_type2;
            ar(output_type2);
            if (output_type2 != 2) {
                return false;
            }

            Span key2;
            ar(key2, kKeySize);

            ar(asset_type_len);
            ar.skip(asset_type_len);

            uint8_t view_tag2;
            ar(view_tag2);
        }
    }
    // 如果币种不为ZEPHYR且输出类型为3，序列化视图标签
    else if (m_outputType == 3) {
        ar(m_viewTag);
    }

    // 序列化额外数据的大小
    ar(m_extraSize);

    // 设置偏移量为交易额外数据的偏移量，创建BlobReader对象并传入相应参数
    setOffset(TX_EXTRA_OFFSET, ar.index());
    BlobReader<true> ar_extra(blob(TX_EXTRA_OFFSET), m_extraSize);
    # 跳过额外数据的大小
    ar.skip(m_extraSize);

    # 设置公钥偏移量为第一次
    bool pubkey_offset_first = true;

    # 当额外数据的索引小于额外数据的大小时执行循环
    while (ar_extra.index() < m_extraSize) {
        uint64_t extra_tag  = 0;
        uint64_t size       = 0;

        # 从额外数据中读取额外标签
        ar_extra(extra_tag);

        # 根据额外标签执行不同的操作
        switch (extra_tag) {
        case 0x01: // TX_EXTRA_TAG_PUBKEY
            # 如果是第一次遇到公钥偏移量，则设置公钥偏移量，并跳过公钥数据
            if (pubkey_offset_first) {
                pubkey_offset_first = false;
                setOffset(TX_PUBKEY_OFFSET, offset(TX_EXTRA_OFFSET) + ar_extra.index());
            }
            ar_extra.skip(kKeySize);
            break;

        case 0x02: // TX_EXTRA_NONCE
            # 从额外数据中读取大小，并设置偏移量，然后读取交易额外数据
            ar_extra(size);
            setOffset(TX_EXTRA_NONCE_OFFSET, offset(TX_EXTRA_OFFSET) + ar_extra.index());
            ar_extra(m_txExtraNonce, size);
            break;

        case 0x03: // TX_EXTRA_MERGE_MINING_TAG
            # 从额外数据中读取大小，并设置偏移量，然后读取合并挖矿标签数据
            ar_extra(size);
            setOffset(TX_EXTRA_MERGE_MINING_TAG_OFFSET, offset(TX_EXTRA_OFFSET) + ar_extra.index());
            ar_extra(m_txMergeMiningTag, size);
            break;

        default:
            return false; // TODO(SChernykh): handle other tags
        }
    }

    # 如果币种是 ZEPHYR，则读取定价记录高度、燃烧数量和铸造数量
    if (m_coin == Coin::ZEPHYR) {
        uint64_t pricing_record_height, amount_burnt, amount_minted;
        ar(pricing_record_height);
        ar(amount_burnt);
        ar(amount_minted);
    }

    # 设置矿工交易前缀结束偏移量
    setOffset(MINER_TX_PREFIX_END_OFFSET, ar.index());
    # 前缀结束

    # 读取环签名类型
    uint8_t vin_rct_type = 0;
    ar(vin_rct_type);

    # 环签名类型必须为 RCTTypeNull (0)
    if (vin_rct_type != 0) {
        return false;
    }

    # 设置矿工交易结束偏移量
    const size_t miner_tx_end = ar.index();
    # 矿工交易结束

    # 矿工交易后必须有且仅有一个值为0的字节
    if ((miner_tx_end != offset(MINER_TX_PREFIX_END_OFFSET) + 1) || (*blob(MINER_TX_PREFIX_END_OFFSET) != 0)) {
        return false;
    }

    # 读取其他交易哈希数量
    ar(m_numHashes);
    # 如果哈希值存在
    if (hashes) {
        # 调整哈希数组大小，以便容纳新的哈希值
        m_hashes.resize((m_numHashes + 1) * kHashSize);
        # 计算矿工交易哈希值
        calculateMinerTxHash(blob(MINER_TX_PREFIX_OFFSET), blob(MINER_TX_PREFIX_END_OFFSET), m_hashes.data());

        # 遍历哈希数组，从第一个哈希值开始
        for (uint64_t i = 1; i <= m_numHashes; ++i) {
            # 创建一个 Span 对象 h
            Span h;
            # 从输入流中读取 kHashSize 大小的数据到 h
            ar(h, kHashSize);
            # 将 h 的数据复制到哈希数组中的指定位置
            memcpy(m_hashes.data() + i * kHashSize, h.data(), kHashSize);
        }

        # 计算 Merkle 树的哈希值
        calculateMerkleTreeHash();
    }

    # 返回 true
    return true;
# 闭合前面的函数定义
```