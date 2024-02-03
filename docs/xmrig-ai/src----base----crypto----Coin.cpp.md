# `xmrig\src\base\crypto\Coin.cpp`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证适用于特定目的。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include "base/crypto/Coin.h"  // 导入加密货币相关的头文件
#include "3rdparty/rapidjson/document.h"  // 导入 rapidjson 库的头文件
#include "base/io/json/Json.h"  // 导入 JSON 相关的头文件
#include "base/io/log/Log.h"  // 导入日志相关的头文件

#include <cstring>  // 导入 C 字符串操作相关的头文件

#ifdef _MSC_VER
#   define strcasecmp _stricmp  // 如果是 MSVC 编译器，则定义 strcasecmp 为 _stricmp
#endif

namespace xmrig {

// 定义加密货币信息结构体
struct CoinInfo
{
    const Algorithm::Id algorithm;  // 加密算法 ID
    const char *code;  // 加密货币代码
    const char *name;  // 加密货币名称
    const uint64_t target;  // 目标值
    const uint64_t units;  // 单位
    const char *tag;  // 标签
};

// 定义加密货币信息数组
static const CoinInfo coinInfo[] = {
    { Algorithm::INVALID,         nullptr,    nullptr,        0,      0,              nullptr },  // 无效加密货币
    { Algorithm::RX_0,            "XMR",      "Monero",       120,    1000000000000,  YELLOW_BG_BOLD( WHITE_BOLD_S " monero  ") },  // Monero 加密货币信息
    { Algorithm::CN_R,            "SUMO",     "Sumokoin",     240,    1000000000,     BLUE_BG_BOLD(   WHITE_BOLD_S " sumo    ") },  // Sumokoin 加密货币信息
    { Algorithm::RX_ARQ,          "ARQ",      "ArQmA",        120,    1000000000,     BLUE_BG_BOLD(   WHITE_BOLD_S " arqma   ") },  // ArQmA 加密货币信息
    { Algorithm::RX_GRAFT,        "GRFT",     "Graft",        120,    10000000000,    BLUE_BG_BOLD(   WHITE_BOLD_S " graft   ") },  // Graft 加密货币信息
    # 使用Algorithm::RX_KEVA算法，代号为"KVA"，名称为"Kevacoin"，初始值为0，最大值为0，颜色为洋红色背景加粗白色字体的" keva    "
    { Algorithm::RX_KEVA,         "KVA",      "Kevacoin",     0,      0,              MAGENTA_BG_BOLD(WHITE_BOLD_S " keva    ") },
    # 使用Algorithm::KAWPOW_RVN算法，代号为"RVN"，名称为"Ravencoin"，初始值为0，最大值为0，颜色为蓝色背景加粗白色字体的" raven   "
    { Algorithm::KAWPOW_RVN,      "RVN",      "Ravencoin",    0,      0,              BLUE_BG_BOLD(   WHITE_BOLD_S " raven   ") },
    # 使用Algorithm::RX_WOW算法，代号为"WOW"，名称为"Wownero"，初始值为300，最大值为100000000000，颜色为洋红色背景加粗白色字体的" wownero "
    { Algorithm::RX_WOW,          "WOW",      "Wownero",      300,    100000000000,   MAGENTA_BG_BOLD(WHITE_BOLD_S " wownero ") },
    # 使用Algorithm::RX_0算法，代号为"ZEPH"，名称为"Zephyr"，初始值为120，最大值为1000000000000，颜色为蓝色背景加粗白色字体的" zephyr  "
    { Algorithm::RX_0,            "ZEPH",     "Zephyr",       120,    1000000000000,  BLUE_BG_BOLD(   WHITE_BOLD_S " zephyr  ") },
};

// 使用静态断言检查Coin::MAX是否等于coinInfo数组的大小
static_assert(Coin::MAX == sizeof(coinInfo) / sizeof(coinInfo[0]), "size mismatch");

// 初始化禁用币种、币种字段和未知币种的字符串常量
const char *Coin::kDisabled = "DISABLED_COIN";
const char *Coin::kField    = "coin";
const char *Coin::kUnknown  = "UNKNOWN_COIN";

// 命名空间结束
} /* namespace xmrig */

// 从JSON值构造Coin对象
xmrig::Coin::Coin(const rapidjson::Value &value)
{
    // 如果值是字符串，则解析为币种ID
    if (value.IsString()) {
        m_id = parse(value.GetString());
    }
    // 如果值是对象且非空，则从对象中获取币种字段的值并解析为币种ID
    else if (value.IsObject() && !value.ObjectEmpty()) {
        m_id = parse(Json::getString(value, kField));
    }
}

// 获取币种对应的算法
xmrig::Algorithm xmrig::Coin::algorithm(uint8_t) const
{
    return coinInfo[m_id].algorithm;
}

// 获取币种对应的代码
const char *xmrig::Coin::code() const
{
    return coinInfo[m_id].code;
}

// 获取币种对应的名称
const char *xmrig::Coin::name() const
{
    return coinInfo[m_id].name;
}

// 将币种信息转换为JSON值
rapidjson::Value xmrig::Coin::toJSON() const
{
    using namespace rapidjson;

    // 如果币种有效，则返回其代码的JSON值，否则返回空值
    return isValid() ? Value(StringRef(code())) : Value(kNullType);
}

// 获取币种对应的目标值
uint64_t xmrig::Coin::target(uint8_t) const
{
    return coinInfo[m_id].target;
}

// 获取币种对应的单位值
uint64_t xmrig::Coin::units() const
{
    return coinInfo[m_id].units;
}

// 解析币种名称并返回对应的币种ID
xmrig::Coin::Id xmrig::Coin::parse(const char *name)
{
    // 如果名称为空或长度小于3，则返回INVALID
    if (name == nullptr || strlen(name) < 3) {
        return INVALID;
    }

    // 遍历币种数组，通过代码或名称匹配来获取对应的币种ID
    for (uint32_t i = 1; i < MAX; ++i) {
        if (strcasecmp(name, coinInfo[i].code) == 0 || strcasecmp(name, coinInfo[i].name) == 0) {
            return static_cast<Id>(i);
        }
    }

    // 如果未匹配到任何币种，则返回INVALID
    return INVALID;
}

// 获取币种对应的标签
const char *xmrig::Coin::tag(Id id)
{
    return coinInfo[id].tag;
}
```