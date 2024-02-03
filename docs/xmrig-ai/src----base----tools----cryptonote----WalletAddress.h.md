# `xmrig\src\base\tools\cryptonote\WalletAddress.h`

```cpp
// XMRig程序的版权声明
// 包含必要的头文件
#ifndef XMRIG_WALLETADDRESS_H
#define XMRIG_WALLETADDRESS_H

#include "base/tools/String.h"
#include "base/crypto/Coin.h"

// 命名空间定义
namespace xmrig {

// 钱包地址类
class WalletAddress
{
public:
    // 网络类型枚举
    enum Net : uint32_t {
        MAINNET,
        TESTNET,
        STAGENET
    };

    // 地址类型枚举
    enum Type : uint32_t {
        PUBLIC,
        INTEGRATED,
        SUBADDRESS
    };

    // 钱包地址的常量定义
    constexpr static size_t kKeySize        = 32;
    constexpr static size_t kMaxSize        = 256;
    constexpr static size_t kMinDataSize    = 69;
    constexpr static size_t kMinSize        = 95;

    // 默认构造函数
    WalletAddress() = default;
    // 构造函数，根据地址和大小解码
    inline WalletAddress(const char *address, size_t size)  { decode(address, size); }
    // 构造函数，根据地址解码
    inline WalletAddress(const char *address)               { decode(address); }
    // 构造函数，根据JSON值解码
    inline WalletAddress(const rapidjson::Value &address)   { decode(address); }
    // 构造函数，根据字符串解码
    inline WalletAddress(const String &address)             { decode(address); }

    // 解码函数，根据地址解码
    inline bool decode(const char *address)                 { return decode(address, strlen(address)); }
    // 使用给定的地址解码，调用重载的 decode 函数
    inline bool decode(const String &address)               { return decode(address, address.size()); }
    // 检查地址是否有效，标签大于 0 并且数据大小大于等于最小大小
    inline bool isValid() const                             { return m_tag > 0 && m_data.size() >= kMinSize; }
    // 返回地址数据
    inline const char *data() const                         { return m_data; }
    // 返回地址对应的加密货币类型
    inline const Coin &coin() const                         { return tagInfo(m_tag).coin; }
    // 返回地址对应的公开花费密钥
    inline const uint8_t *spendKey() const                  { return m_publicSpendKey; }
    // 返回地址对应的公开查看密钥
    inline const uint8_t *viewKey() const                   { return m_publicViewKey; }
    // 返回地址对应的网络类型
    inline Net net() const                                  { return tagInfo(m_tag).net; }
    // 返回地址对应的类型
    inline Type type() const                                { return tagInfo(m_tag).type; }
    // 返回地址对应的 RPC 端口号
    inline uint16_t rpcPort() const                         { return tagInfo(m_tag).rpcPort; }
    // 返回地址对应的 ZMQ 端口号
    inline uint16_t zmqPort() const                         { return tagInfo(m_tag).zmqPort; }
    // 返回地址对应的标签
    inline uint64_t tag() const                             { return m_tag; }
    
    // 使用给定的地址和大小解码
    bool decode(const char *address, size_t size);
    // 使用给定的 JSON 对象解码
    bool decode(const rapidjson::Value &address);
    // 返回网络类型的名称
    const char *netName() const;
    // 返回地址类型的名称
    const char *typeName() const;
    // 将地址转换为 JSON 格式
    rapidjson::Value toJSON(rapidjson::Document &doc) const;
// 如果定义了 XMRIG_FEATURE_API，则声明一个名为 toAPI 的函数，该函数接受一个 rapidjson::Document 对象并返回 rapidjson::Value 对象
#ifdef XMRIG_FEATURE_API
    rapidjson::Value toAPI(rapidjson::Document &doc) const;
#endif

// 私有成员变量和结构体定义
private:
    struct TagInfo
    {
        const Coin coin;  // 币种
        const Net net;  // 网络类型
        const Type type;  // 类型
        const uint16_t rpcPort;  // RPC 端口
        const uint16_t zmqPort;  // ZMQ 端口
    };

    // 静态函数，返回指定标签的信息
    static const TagInfo &tagInfo(uint64_t tag);

    String m_data;  // 数据
    uint64_t m_tag  = 0;  // 标签
    uint8_t m_checksum[4]{};  // 校验和
    uint8_t m_publicSpendKey[kKeySize]{};  // 公开花费密钥
    uint8_t m_publicViewKey[kKeySize]{};  // 公开查看密钥
};


} /* namespace xmrig */

#endif /* XMRIG_WALLETADDRESS_H */
```