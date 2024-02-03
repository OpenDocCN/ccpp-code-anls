# `xmrig\src\base\crypto\Coin.h`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 它可以是许可证的第3版，也可以是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的适用性。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_COIN_H
#define XMRIG_COIN_H


#include "3rdparty/rapidjson/fwd.h"
#include "base/crypto/Algorithm.h"


namespace xmrig {


class Coin
{
public:
    enum Id : uint32_t {
        INVALID,
        MONERO,
        SUMO,
        ARQMA,
        GRAFT,
        KEVA,
        RAVEN,
        WOWNERO,
        ZEPHYR,
        MAX
    };

    static const char *kDisabled;
    static const char *kField;
    static const char *kUnknown;

    Coin() = default;
    Coin(const rapidjson::Value &value);
    inline Coin(const char *name) : m_id(parse(name))                           {}  // 通过名称构造Coin对象
    inline Coin(Id id) : m_id(id)                                               {}  // 通过ID构造Coin对象
    inline Coin(uint32_t id) : m_id(id < MAX ? static_cast<Id>(id) : INVALID)   {}  // 通过ID值构造Coin对象


    inline bool isEqual(const Coin &other) const                                { return m_id == other.m_id; }  // 判断两个Coin对象是否相等
    inline bool isValid() const                                                 { return m_id != INVALID; }  // 判断Coin对象是否有效
    inline Id id() const                                                        { return m_id; }  // 返回Coin对象的ID
    // 返回标签
    inline const char *tag() const                                              { return tag(m_id); }
    // 将无符号整数金额转换为浮点数金额
    inline double decimal(uint64_t amount) const                                { return static_cast<double>(amount) / units(); }
    
    // 返回算法
    Algorithm algorithm(uint8_t blobVersion = 255) const;
    // 返回代码
    const char *code() const;
    // 返回名称
    const char *name() const;
    // 转换为 JSON 格式
    rapidjson::Value toJSON() const;
    // 返回目标
    uint64_t target(uint8_t blobVersion = 255) const;
    // 返回单位
    uint64_t units() const;
    
    // 不等于运算符重载
    inline bool operator!=(Id id) const                                         { return m_id != id; }
    // 不等于运算符重载
    inline bool operator!=(const Coin &other) const                             { return !isEqual(other); }
    // 小于运算符重载
    inline bool operator<(Id id) const                                          { return m_id < id; }
    // 小于运算符重载
    inline bool operator<(const Coin &other) const                              { return m_id < other.m_id; }
    // 等于运算符重载
    inline bool operator==(Id id) const                                         { return m_id == id; }
    // 等于运算符重载
    inline bool operator==(const Coin &other) const                             { return isEqual(other); }
    // 类型转换运算符重载
    inline operator Id() const                                                  { return m_id; }
    
    // 解析名称并返回 ID
    static Id parse(const char *name);
    // 返回标签
    static const char *tag(Id id);
# 私有成员部分开始
private:
    # 初始化一个ID变量，初始值为INVALID
    Id m_id = INVALID;
};  # 类的私有成员部分结束
# 命名空间xmrig结束
} /* namespace xmrig */
# 宏定义结束
#endif /* XMRIG_COIN_H */
```