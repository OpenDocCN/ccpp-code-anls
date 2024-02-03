# `xmrig\src\base\tools\cryptonote\WalletAddress.cpp`

```cpp
/* XMRig
 * 版权所有 (c) 2012-2013 The Cryptonote developers
 * 版权所有 (c) 2014-2021 The Monero Project
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是基于希望它有用而分发的，但没有任何保证；甚至没有暗示的保证适用于特定目的。有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include "base/tools/cryptonote/WalletAddress.h"
#include "3rdparty/rapidjson/document.h"
#include "base/crypto/keccak.h"
#include "base/tools/Buffer.h"
#include "base/tools/cryptonote/BlobReader.h"
#include "base/tools/cryptonote/umul128.h"
#include "base/tools/Cvt.h"

#include <array>
#include <map>

// 解码钱包地址的函数
bool xmrig::WalletAddress::decode(const char *address, size_t size)
{
    // 定义块大小数组
    static constexpr std::array<int, 9> block_sizes{ 0, 2, 3, 5, 6, 7, 9, 10, 11 };
    // 定义地址字符集
    static constexpr char alphabet[] = "123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz";
    // 计算地址字符集大小
    constexpr size_t alphabet_size = sizeof(alphabet) - 1;

    // 如果地址长度小于最小长度或大于最大长度，则返回假
    if (size < kMinSize || size > kMaxSize) {
        return false;
    }

    // 初始化反向地址字符集数组
    int8_t reverse_alphabet[256];
    memset(reverse_alphabet, -1, sizeof(reverse_alphabet));

    // 填充反向地址字符集数组
    for (size_t i = 0; i < alphabet_size; ++i) {
        reverse_alphabet[static_cast<int>(alphabet[i])] = i;
    }

    // 计算地址长度
    const int len = static_cast<int>(size);
    // 计算完整块数
    const int num_full_blocks = len / block_sizes.back();
    // 计算最后一个数据块的大小
    const int last_block_size = len % block_sizes.back();

    // 初始化最后一个数据块的索引为-1
    int last_block_size_index = -1;

    // 遍历数据块大小的数组，找到最后一个数据块的索引
    for (size_t i = 0; i < block_sizes.size(); ++i) {
        if (block_sizes[i] == last_block_size) {
            last_block_size_index = i;
            break;
        }
    }

    // 如果未找到最后一个数据块的索引，返回false
    if (last_block_size_index < 0) {
        return false;
    }

    // 计算数据大小，包括完整数据块和最后一个数据块
    const size_t data_size = static_cast<size_t>(num_full_blocks) * sizeof(uint64_t) + last_block_size_index;
    
    // 如果数据大小小于最小数据大小，返回false
    if (data_size < kMinDataSize) {
        return false;
    }

    // 初始化数据缓冲区
    Buffer data;
    data.reserve(data_size);

    // 初始化地址数据指针
    const char *address_data = address;

    // 遍历完整数据块和最后一个数据块
    for (int i = 0; i <= num_full_blocks; ++i) {
        uint64_t num = 0;
        uint64_t order = 1;

        // 遍历每个数据块中的字符
        for (int j = ((i < num_full_blocks) ? block_sizes.back() : last_block_size) - 1; j >= 0; --j) {
            // 获取字符对应的数字
            const int digit = reverse_alphabet[static_cast<int>(address_data[j])];
            // 如果字符对应的数字小于0，返回false
            if (digit < 0) {
                return false;
            }

            uint64_t hi;
            // 计算数字的值
            const uint64_t tmp = num + __umul128(order, static_cast<uint64_t>(digit), &hi);
            // 如果计算溢出，返回false
            if ((tmp < num) || hi) {
                return false;
            }

            num = tmp;
            order *= alphabet_size;
        }

        address_data += block_sizes.back();

        // 将数字转换为字节并添加到数据缓冲区
        auto p = reinterpret_cast<const uint8_t*>(&num);
        for (int j = ((i < num_full_blocks) ? static_cast<int>(sizeof(num)) : last_block_size_index) - 1; j >= 0; --j) {
            data.emplace_back(p[j]);
        }
    }

    // 断言数据缓冲区大小等于数据大小
    assert(data.size() == data_size);

    // 从数据缓冲区中读取数据
    BlobReader<false> ar(data.data(), data_size);

    // 读取数据并验证
    if (ar(m_tag) && ar(m_publicSpendKey) && ar(m_publicViewKey) && ar.skip(ar.remaining() - sizeof(m_checksum)) && ar(m_checksum)) {
        // 计算数据的哈希值
        uint8_t md[200];
        keccak(data.data(), data_size - sizeof(m_checksum), md);

        // 如果哈希值与校验和相等，设置数据并返回true
        if (memcmp(m_checksum, md, sizeof(m_checksum)) == 0) {
            m_data = { address, size };
            return true;
        }
    }

    // 重置标签值
    m_tag = 0;
    # 返回布尔值 false
    return false;
// 解码钱包地址，判断地址是否为字符串类型，然后调用另一个解码函数进行解码
bool xmrig::WalletAddress::decode(const rapidjson::Value &address)
{
    return address.IsString() && decode(address.GetString(), address.GetStringLength());
}

// 返回网络名称，根据网络类型返回对应的名称
const char *xmrig::WalletAddress::netName() const
{
    static const std::array<const char *, 3> names = { "mainnet", "testnet", "stagenet" };

    return names[net()];
}

// 返回地址类型名称，根据地址类型返回对应的名称
const char *xmrig::WalletAddress::typeName() const
{
    static const std::array<const char *, 3> names = { "public", "integrated", "subaddress" };

    return names[type()];
}

// 将钱包地址转换为 JSON 格式，如果地址有效则转换为 JSON，否则返回空值
rapidjson::Value xmrig::WalletAddress::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;

    return isValid() ? m_data.toJSON(doc) : Value(kNullType);
}

#ifdef XMRIG_FEATURE_API
// 将钱包地址转换为 API 格式的 JSON，包括地址、类型、网络、端口等信息
rapidjson::Value xmrig::WalletAddress::toAPI(rapidjson::Document &doc) const
{
    using namespace rapidjson;

    if (!isValid()) {
        return Value(kNullType);
    }

    auto &allocator = doc.GetAllocator();
    Value out(kObjectType);
    out.AddMember(StringRef(Coin::kField),  coin().toJSON(), allocator);
    out.AddMember("address",                m_data.toJSON(doc), allocator);
    out.AddMember("type",                   StringRef(typeName()), allocator);
    out.AddMember("net",                    StringRef(netName()), allocator);
    out.AddMember("rpc_port",               rpcPort(), allocator);
    out.AddMember("zmq_port",               zmqPort(), allocator);
    out.AddMember("tag",                    m_tag, allocator);
    out.AddMember("view_key",               Cvt::toHex(m_publicViewKey, kKeySize, doc), allocator);
    out.AddMember("spend_key",              Cvt::toHex(m_publicSpendKey, kKeySize, doc), allocator);
    out.AddMember("checksum",               Cvt::toHex(m_checksum, sizeof(m_checksum), doc), allocator);

    return out;
}
#endif

// 返回标签信息，根据标签查找对应的信息，如果找不到则返回默认的 dummy 信息
const xmrig::WalletAddress::TagInfo &xmrig::WalletAddress::tagInfo(uint64_t tag)
{
    static TagInfo dummy = { Coin::INVALID, MAINNET, PUBLIC, 0, 0 };
    };

    const auto it = tags.find(tag);
    # 如果迭代器 it 等于 tags.end()，则返回 dummy，否则返回 it->second
    return it == tags.end() ? dummy : it->second;
# 闭合前面的函数定义
```