# `xmrig\src\base\crypto\Algorithm.h`

```
#ifndef XMRIG_ALGORITHM_H
#define XMRIG_ALGORITHM_H

#include <functional>  // 包含函数对象的头文件
#include <vector>  // 包含向量的头文件

#include "3rdparty/rapidjson/fwd.h"  // 包含第三方库 rapidjson 的前向声明

namespace xmrig {

class Algorithm  // 定义 Algorithm 类
{
public:
    // Changes in following file may required if this enum changed:
    //
    // src/backend/opencl/cl/cn/algorithm.cl
    //
    // Id encoding:
    // 1 byte: family
    // 1 byte: L3 memory as power of 2 (if applicable).
    // 1 byte: L2 memory for RandomX algorithms as power of 2, base variant for CryptoNight algorithms or 0x00.
    // 1 byte: extra variant (coin) id.
    };

    enum Family : uint32_t {  // 定义枚举类型 Family
        UNKNOWN         = 0,  // 未知类型
        CN_ANY          = 0x63000000,  // CN_ANY 类型
        CN              = 0x63150000,  // CN 类型
        CN_LITE         = 0x63140000,  // CN_LITE 类型
        CN_HEAVY        = 0x63160000,  // CN_HEAVY 类型
        CN_PICO         = 0x63120000,  // CN_PICO 类型
        CN_FEMTO        = 0x63110000,  // CN_FEMTO 类型
        RANDOM_X        = 0x72000000,  // RANDOM_X 类型
        ARGON2          = 0x61000000,  // ARGON2 类型
        KAWPOW          = 0x6b000000,  // KAWPOW 类型
        GHOSTRIDER      = 0x6c000000  // GHOSTRIDER 类型
    };

    static const char *kINVALID;  // 静态成员变量 kINVALID
    static const char *kCN;  // 静态成员变量 kCN
    # 定义静态常量字符串指针 kCN_0
    static const char *kCN_0;
    # 定义静态常量字符串指针 kCN_1
    static const char *kCN_1;
    # 定义静态常量字符串指针 kCN_2
    static const char *kCN_2;
    # 定义静态常量字符串指针 kCN_R
    static const char *kCN_R;
    # 定义静态常量字符串指针 kCN_FAST
    static const char *kCN_FAST;
    # 定义静态常量字符串指针 kCN_HALF
    static const char *kCN_HALF;
    # 定义静态常量字符串指针 kCN_XAO
    static const char *kCN_XAO;
    # 定义静态常量字符串指针 kCN_RTO
    static const char *kCN_RTO;
    # 定义静态常量字符串指针 kCN_RWZ
    static const char *kCN_RWZ;
    # 定义静态常量字符串指针 kCN_ZLS
    static const char *kCN_ZLS;
    # 定义静态常量字符串指针 kCN_DOUBLE
    static const char *kCN_DOUBLE;
    # 定义静态常量字符串指针 kCN_CCX
    static const char *kCN_CCX;
#   ifdef XMRIG_ALGO_CN_LITE
    // 如果定义了 XMRIG_ALGO_CN_LITE，则声明以下常量指针
    static const char *kCN_LITE;
    static const char *kCN_LITE_0;
    static const char *kCN_LITE_1;
#   endif

#   ifdef XMRIG_ALGO_CN_HEAVY
    // 如果定义了 XMRIG_ALGO_CN_HEAVY，则声明以下常量指针
    static const char *kCN_HEAVY;
    static const char *kCN_HEAVY_0;
    static const char *kCN_HEAVY_TUBE;
    static const char *kCN_HEAVY_XHV;
#   endif

#   ifdef XMRIG_ALGO_CN_PICO
    // 如果定义了 XMRIG_ALGO_CN_PICO，则声明以下常量指针
    static const char *kCN_PICO;
    static const char *kCN_PICO_0;
    static const char *kCN_PICO_TLO;
#   endif

#   ifdef XMRIG_ALGO_CN_FEMTO
    // 如果定义了 XMRIG_ALGO_CN_FEMTO，则声明以下常量指针
    static const char *kCN_UPX2;
#   endif

#   ifdef XMRIG_ALGO_RANDOMX
    // 如果定义了 XMRIG_ALGO_RANDOMX，则声明以下常量指针
    static const char *kRX;
    static const char *kRX_0;
    static const char *kRX_WOW;
    static const char *kRX_ARQ;
    static const char *kRX_GRAFT;
    static const char *kRX_SFX;
    static const char *kRX_KEVA;
#   endif

#   ifdef XMRIG_ALGO_ARGON2
    // 如果定义了 XMRIG_ALGO_ARGON2，则声明以下常量指针
    static const char *kAR2;
    static const char *kAR2_CHUKWA;
    static const char *kAR2_CHUKWA_V2;
    static const char *kAR2_WRKZ;
#   endif

#   ifdef XMRIG_ALGO_KAWPOW
    // 如果定义了 XMRIG_ALGO_KAWPOW，则声明以下常量指针
    static const char *kKAWPOW;
    static const char *kKAWPOW_RVN;
#   endif

#   ifdef XMRIG_ALGO_GHOSTRIDER
    // 如果定义了 XMRIG_ALGO_GHOSTRIDER，则声明以下常量指针
    static const char* kGHOSTRIDER;
    static const char* kGHOSTRIDER_RTM;
#   endif

    // 默认构造函数
    inline Algorithm() = default;
    // 根据算法名称构造函数
    inline Algorithm(const char *algo) : m_id(parse(algo))  {}
    // 根据算法 ID 构造函数
    inline Algorithm(Id id) : m_id(id)                      {}
    // 根据 JSON 值构造函数
    Algorithm(const rapidjson::Value &value);
    // 根据 ID 构造函数
    Algorithm(uint32_t id);

    // 判断是否为 CN 算法
    static inline constexpr bool isCN(Id id)                { return (id & 0xff000000) == CN_ANY; }
    // 获取基础算法 ID
    static inline constexpr Id base(Id id)                  { return isCN(id) ? static_cast<Id>(CN_0 | (id & 0xff00)) : INVALID; }
    // 获取算法的 L2 缓存大小
    static inline constexpr size_t l2(Id id)                { return family(id) == RANDOM_X ? (1U << ((id >> 8) & 0xff)) : 0U; }
    // 获取算法的 L3 缓存大小
    static inline constexpr size_t l3(Id id)                { return 1ULL << ((id >> 16) & 0xff); }
    // 返回算法的家族标识
    static inline constexpr uint32_t family(Id id)          { return id & (isCN(id) ? 0xffff0000 : 0xff000000); }

    // 检查算法是否为CN（中国）家族
    inline bool isCN() const                                { return isCN(m_id); }
    // 检查算法是否与另一个算法相等
    inline bool isEqual(const Algorithm &other) const       { return m_id == other.m_id; }
    // 检查算法是否有效
    inline bool isValid() const                             { return m_id != INVALID && family() > UNKNOWN; }
    // 返回算法的基本标识
    inline Id base() const                                  { return base(m_id); }
    // 返回算法的标识
    inline Id id() const                                    { return m_id; }
    // 返回算法的L2大小
    inline size_t l2() const                                { return l2(m_id); }
    // 返回算法的家族标识
    inline uint32_t family() const                          { return family(m_id); }
    // 返回算法的最小强度
    inline uint32_t minIntensity() const                    { return ((m_id == GHOSTRIDER_RTM) ? 8 : 1); };
    // 返回算法的最大强度
    inline uint32_t maxIntensity() const                    { return isCN() ? 5 : ((m_id == GHOSTRIDER_RTM) ? 8 : 1); };

    // 返回算法的L3大小
    inline size_t l3() const                                { return l3(m_id); }

    // 检查算法是否不等于指定标识
    inline bool operator!=(Algorithm::Id id) const          { return m_id != id; }
    // 检查算法是否不等于另一个算法
    inline bool operator!=(const Algorithm &other) const    { return !isEqual(other); }
    // 检查算法是否等于指定标识
    inline bool operator==(Algorithm::Id id) const          { return m_id == id; }
    // 检查算法是否等于另一个算法
    inline bool operator==(const Algorithm &other) const    { return isEqual(other); }
    // 转换算法为其标识
    inline operator Algorithm::Id() const                   { return m_id; }

    // 返回算法的名称
    const char *name() const;
    // 将算法转换为JSON格式
    rapidjson::Value toJSON() const;
    // 将算法转换为JSON格式，并添加到指定的JSON文档中
    rapidjson::Value toJSON(rapidjson::Document &doc) const;

    // 解析算法名称并返回其标识
    static Id parse(const char *name);
    // 返回算法的数量
    static size_t count();
    // 返回所有满足条件的算法
    static std::vector<Algorithm> all(const std::function<bool(const Algorithm &algo)> &filter = nullptr);
# 声明私有成员变量 Id，并初始化为 INVALID
private:
    Id m_id = INVALID;
# 使用别名声明 Algorithms 为 std::vector<Algorithm> 类型
using Algorithms = std::vector<Algorithm>;
# 结束命名空间 xmrig
} /* namespace xmrig */
# 结束头文件定义
#endif /* XMRIG_ALGORITHM_H */
```