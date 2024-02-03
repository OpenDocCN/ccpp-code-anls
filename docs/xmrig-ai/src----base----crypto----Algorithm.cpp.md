# `xmrig\src\base\crypto\Algorithm.cpp`

```cpp
/* XMRig
 * 版权所有（C）2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有（C）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（C）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发或修改它，
 * 由自由软件基金会发布的版本3的许可证，或者
 * （在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/crypto/Algorithm.h"
#include "3rdparty/rapidjson/document.h"
#include "base/tools/String.h"


#include <cassert>
#include <cstdio>
#include <cstdlib>
#include <map>


#ifdef _MSC_VER
#   define strcasecmp  _stricmp
#endif


namespace xmrig {


const char *Algorithm::kINVALID         = "invalid";
const char *Algorithm::kCN              = "cn";
const char *Algorithm::kCN_0            = "cn/0";
const char *Algorithm::kCN_1            = "cn/1";
const char *Algorithm::kCN_2            = "cn/2";
const char *Algorithm::kCN_R            = "cn/r";
const char *Algorithm::kCN_FAST         = "cn/fast";
const char *Algorithm::kCN_HALF         = "cn/half";
const char *Algorithm::kCN_XAO          = "cn/xao";
const char *Algorithm::kCN_RTO          = "cn/rto";
const char *Algorithm::kCN_RWZ          = "cn/rwz";
const char *Algorithm::kCN_ZLS          = "cn/zls";
const char *Algorithm::kCN_DOUBLE       = "cn/double";
const char *Algorithm::kCN_CCX          = "cn/ccx";

#ifdef XMRIG_ALGO_CN_LITE
const char *Algorithm::kCN_LITE         = "cn-lite";
// 定义常量字符串，表示不同的算法类型
const char *Algorithm::kCN_LITE_0       = "cn-lite/0";
const char *Algorithm::kCN_LITE_1       = "cn-lite/1";
#endif

#ifdef XMRIG_ALGO_CN_HEAVY
const char *Algorithm::kCN_HEAVY        = "cn-heavy";
const char *Algorithm::kCN_HEAVY_0      = "cn-heavy/0";
const char *Algorithm::kCN_HEAVY_TUBE   = "cn-heavy/tube";
const char *Algorithm::kCN_HEAVY_XHV    = "cn-heavy/xhv";
#endif

#ifdef XMRIG_ALGO_CN_PICO
const char *Algorithm::kCN_PICO         = "cn-pico";
const char *Algorithm::kCN_PICO_0       = "cn-pico";
const char *Algorithm::kCN_PICO_TLO     = "cn-pico/tlo";
#endif

#ifdef XMRIG_ALGO_CN_FEMTO
const char *Algorithm::kCN_UPX2         = "cn/upx2";
#endif

#ifdef XMRIG_ALGO_RANDOMX
const char *Algorithm::kRX              = "rx";
const char *Algorithm::kRX_0            = "rx/0";
const char *Algorithm::kRX_WOW          = "rx/wow";
const char *Algorithm::kRX_ARQ          = "rx/arq";
const char *Algorithm::kRX_GRAFT        = "rx/graft";
const char *Algorithm::kRX_SFX          = "rx/sfx";
const char *Algorithm::kRX_KEVA         = "rx/keva";
#endif

#ifdef XMRIG_ALGO_ARGON2
const char *Algorithm::kAR2             = "argon2";
const char *Algorithm::kAR2_CHUKWA      = "argon2/chukwa";
const char *Algorithm::kAR2_CHUKWA_V2   = "argon2/chukwav2";
const char *Algorithm::kAR2_WRKZ        = "argon2/ninja";
#endif

#ifdef XMRIG_ALGO_KAWPOW
const char *Algorithm::kKAWPOW          = "kawpow";
const char *Algorithm::kKAWPOW_RVN      = "kawpow";
#endif

#ifdef XMRIG_ALGO_GHOSTRIDER
const char* Algorithm::kGHOSTRIDER      = "ghostrider";
const char* Algorithm::kGHOSTRIDER_RTM  = "ghostrider";
#endif

// 定义宏，用于将算法类型和其对应的字符串名字关联起来
#define ALGO_NAME(ALGO)         { Algorithm::ALGO, Algorithm::k##ALGO }
#define ALGO_ALIAS(ALGO, NAME)  { NAME, Algorithm::ALGO }
#define ALGO_ALIAS_AUTO(ALGO)   { Algorithm::k##ALGO, Algorithm::ALGO }

// 定义静态常量映射表，将算法类型和其对应的字符串名字关联起来
static const std::map<uint32_t, const char *> kAlgorithmNames = {
    ALGO_NAME(CN_0),
    ALGO_NAME(CN_1),
    ALGO_NAME(CN_2),
    ALGO_NAME(CN_R),
    ALGO_NAME(CN_FAST),
    # 定义算法名称为 CN_HALF
    ALGO_NAME(CN_HALF),
    # 定义算法名称为 CN_XAO
    ALGO_NAME(CN_XAO),
    # 定义算法名称为 CN_RTO
    ALGO_NAME(CN_RTO),
    # 定义算法名称为 CN_RWZ
    ALGO_NAME(CN_RWZ),
    # 定义算法名称为 CN_ZLS
    ALGO_NAME(CN_ZLS),
    # 定义算法名称为 CN_DOUBLE
    ALGO_NAME(CN_DOUBLE),
    # 定义算法名称为 CN_CCX
    ALGO_NAME(CN_CCX),
#   ifdef XMRIG_ALGO_CN_LITE
    # 如果定义了 XMRIG_ALGO_CN_LITE，则执行以下代码
    ALGO_NAME(CN_LITE_0),
    # 定义 CN_LITE_0 算法名称
    ALGO_NAME(CN_LITE_1),
    # 定义 CN_LITE_1 算法名称
#   endif
    # 结束 XMRIG_ALGO_CN_LITE 的条件编译

#   ifdef XMRIG_ALGO_CN_HEAVY
    # 如果定义了 XMRIG_ALGO_CN_HEAVY，则执行以下代码
    ALGO_NAME(CN_HEAVY_0),
    # 定义 CN_HEAVY_0 算法名称
    ALGO_NAME(CN_HEAVY_TUBE),
    # 定义 CN_HEAVY_TUBE 算法名称
    ALGO_NAME(CN_HEAVY_XHV),
    # 定义 CN_HEAVY_XHV 算法名称
#   endif
    # 结束 XMRIG_ALGO_CN_HEAVY 的条件编译

#   ifdef XMRIG_ALGO_CN_PICO
    # 如果定义了 XMRIG_ALGO_CN_PICO，则执行以下代码
    ALGO_NAME(CN_PICO_0),
    # 定义 CN_PICO_0 算法名称
    ALGO_NAME(CN_PICO_TLO),
    # 定义 CN_PICO_TLO 算法名称
#   endif
    # 结束 XMRIG_ALGO_CN_PICO 的条件编译

#   ifdef XMRIG_ALGO_CN_FEMTO
    # 如果定义了 XMRIG_ALGO_CN_FEMTO，则执行以下代码
    ALGO_NAME(CN_UPX2),
    # 定义 CN_UPX2 算法名称
#   endif
    # 结束 XMRIG_ALGO_CN_FEMTO 的条件编译

#   ifdef XMRIG_ALGO_RANDOMX
    # 如果定义了 XMRIG_ALGO_RANDOMX，则执行以下代码
    ALGO_NAME(RX_0),
    # 定义 RX_0 算法名称
    ALGO_NAME(RX_WOW),
    # 定义 RX_WOW 算法名称
    ALGO_NAME(RX_ARQ),
    # 定义 RX_ARQ 算法名称
    ALGO_NAME(RX_GRAFT),
    # 定义 RX_GRAFT 算法名称
    ALGO_NAME(RX_SFX),
    # 定义 RX_SFX 算法名称
    ALGO_NAME(RX_KEVA),
    # 定义 RX_KEVA 算法名称
#   endif
    # 结束 XMRIG_ALGO_RANDOMX 的条件编译

#   ifdef XMRIG_ALGO_ARGON2
    # 如果定义了 XMRIG_ALGO_ARGON2，则执行以下代码
    ALGO_NAME(AR2_CHUKWA),
    # 定义 AR2_CHUKWA 算法名称
    ALGO_NAME(AR2_CHUKWA_V2),
    # 定义 AR2_CHUKWA_V2 算法名称
    ALGO_NAME(AR2_WRKZ),
    # 定义 AR2_WRKZ 算法名称
#   endif
    # 结束 XMRIG_ALGO_ARGON2 的条件编译

#   ifdef XMRIG_ALGO_KAWPOW
    # 如果定义了 XMRIG_ALGO_KAWPOW，则执行以下代码
    ALGO_NAME(KAWPOW_RVN),
    # 定义 KAWPOW_RVN 算法名称
#   endif
    # 结束 XMRIG_ALGO_KAWPOW 的条件编译

#   ifdef XMRIG_ALGO_GHOSTRIDER
    # 如果定义了 XMRIG_ALGO_GHOSTRIDER，则执行以下代码
    ALGO_NAME(GHOSTRIDER_RTM),
    # 定义 GHOSTRIDER_RTM 算法名称
#   endif
    # 结束 XMRIG_ALGO_GHOSTRIDER 的条件编译

};

# 定义一个结构体 aliasCompare
struct aliasCompare
{
   inline bool operator()(const char *a, const char *b) const   { return strcasecmp(a, b) < 0; }
};

# 定义一个常量映射表 kAlgorithmAliases
static const std::map<const char *, Algorithm::Id, aliasCompare> kAlgorithmAliases = {
    ALGO_ALIAS_AUTO(CN_0),          ALGO_ALIAS(CN_0,            "cryptonight/0"),
                                    ALGO_ALIAS(CN_0,            "cryptonight"),
                                    ALGO_ALIAS(CN_0,            "cn"),
    ALGO_ALIAS_AUTO(CN_1),          ALGO_ALIAS(CN_1,            "cryptonight/1"),
                                    ALGO_ALIAS(CN_1,            "cryptonight-monerov7"),
                                    ALGO_ALIAS(CN_1,            "cryptonight_v7"),
    ALGO_ALIAS_AUTO(CN_2),          ALGO_ALIAS(CN_2,            "cryptonight/2"),
                                    ALGO_ALIAS(CN_2,            "cryptonight-monerov8"),
                                    ALGO_ALIAS(CN_2,            "cryptonight_v8"),
    # 自动设置算法别名为CN_FAST，同时设置CN_FAST的其他别名
    ALGO_ALIAS_AUTO(CN_FAST),       ALGO_ALIAS(CN_FAST,         "cryptonight/fast"),
                                    ALGO_ALIAS(CN_FAST,         "cryptonight/msr"),
                                    ALGO_ALIAS(CN_FAST,         "cn/msr"),
    # 自动设置算法别名为CN_R，同时设置CN_R的其他别名
    ALGO_ALIAS_AUTO(CN_R),          ALGO_ALIAS(CN_R,            "cryptonight/r"),
                                    ALGO_ALIAS(CN_R,            "cryptonight_r"),
    # 自动设置算法别名为CN_XAO，同时设置CN_XAO的其他别名
    ALGO_ALIAS_AUTO(CN_XAO),        ALGO_ALIAS(CN_XAO,          "cryptonight/xao"),
                                    ALGO_ALIAS(CN_XAO,          "cryptonight_alloy"),
    # 自动设置算法别名为CN_HALF，同时设置CN_HALF的其他别名
    ALGO_ALIAS_AUTO(CN_HALF),       ALGO_ALIAS(CN_HALF,         "cryptonight/half"),
    # 自动设置算法别名为CN_RTO，同时设置CN_RTO的其他别名
    ALGO_ALIAS_AUTO(CN_RTO),        ALGO_ALIAS(CN_RTO,          "cryptonight/rto"),
    # 自动设置算法别名为CN_RWZ，同时设置CN_RWZ的其他别名
    ALGO_ALIAS_AUTO(CN_RWZ),        ALGO_ALIAS(CN_RWZ,          "cryptonight/rwz"),
    # 自动设置算法别名为CN_ZLS，同时设置CN_ZLS的其他别名
    ALGO_ALIAS_AUTO(CN_ZLS),        ALGO_ALIAS(CN_ZLS,          "cryptonight/zls"),
    # 自动设置算法别名为CN_DOUBLE，同时设置CN_DOUBLE的其他别名
    ALGO_ALIAS_AUTO(CN_DOUBLE),     ALGO_ALIAS(CN_DOUBLE,       "cryptonight/double"),
    # 自动设置算法别名为CN_CCX，同时设置CN_CCX的其他别名
    ALGO_ALIAS_AUTO(CN_CCX),        ALGO_ALIAS(CN_CCX,          "cryptonight/ccx"),
                                    ALGO_ALIAS(CN_CCX,          "cryptonight/conceal"),
                                    ALGO_ALIAS(CN_CCX,          "cn/conceal"),
# 如果定义了 XMRIG_ALGO_CN_LITE，则执行以下代码块
    # 自动设置 CN_LITE_0 的别名
    ALGO_ALIAS_AUTO(CN_LITE_0),     
    # 设置 CN_LITE_0 的别名为 "cryptonight-lite/0"
    ALGO_ALIAS(CN_LITE_0,       "cryptonight-lite/0"),
    # 设置 CN_LITE_0 的别名为 "cryptonight-lite"
    ALGO_ALIAS(CN_LITE_0,       "cryptonight-lite"),
    # 设置 CN_LITE_0 的别名为 "cryptonight-light"
    ALGO_ALIAS(CN_LITE_0,       "cryptonight-light"),
    # 设置 CN_LITE_0 的别名为 "cn-lite"
    ALGO_ALIAS(CN_LITE_0,       "cn-lite"),
    # 设置 CN_LITE_0 的别名为 "cn-light"
    ALGO_ALIAS(CN_LITE_0,       "cn-light"),
    # 设置 CN_LITE_0 的别名为 "cryptonight_lite"
    ALGO_ALIAS(CN_LITE_0,       "cryptonight_lite"),
    # 自动设置 CN_LITE_1 的别名
    ALGO_ALIAS_AUTO(CN_LITE_1),     
    # 设置 CN_LITE_1 的别名为 "cryptonight-lite/1"
    ALGO_ALIAS(CN_LITE_1,       "cryptonight-lite/1"),
    # 设置 CN_LITE_1 的别名为 "cryptonight-aeonv7"
    ALGO_ALIAS(CN_LITE_1,       "cryptonight-aeonv7"),
    # 设置 CN_LITE_1 的别名为 "cryptonight_lite_v7"
    ALGO_ALIAS(CN_LITE_1,       "cryptonight_lite_v7"),
# 如果定义了 XMRIG_ALGO_CN_HEAVY，则执行以下代码块
    # 自动设置 CN_HEAVY_0 的别名
    ALGO_ALIAS_AUTO(CN_HEAVY_0),    
    # 设置 CN_HEAVY_0 的别名为 "cryptonight-heavy/0"
    ALGO_ALIAS(CN_HEAVY_0,      "cryptonight-heavy/0"),
    # 设置 CN_HEAVY_0 的别名为 "cryptonight-heavy"
    ALGO_ALIAS(CN_HEAVY_0,      "cryptonight-heavy"),
    # 设置 CN_HEAVY_0 的别名为 "cn-heavy"
    ALGO_ALIAS(CN_HEAVY_0,      "cn-heavy"),
    # 设置 CN_HEAVY_0 的别名为 "cryptonight_heavy"
    ALGO_ALIAS(CN_HEAVY_0,      "cryptonight_heavy"),
    # 自动设置 CN_HEAVY_XHV 的别名
    ALGO_ALIAS_AUTO(CN_HEAVY_XHV),  
    # 设置 CN_HEAVY_XHV 的别名为 "cryptonight-heavy/xhv"
    ALGO_ALIAS(CN_HEAVY_XHV,    "cryptonight-heavy/xhv"),
    # 设置 CN_HEAVY_XHV 的别名为 "cryptonight_haven"
    ALGO_ALIAS(CN_HEAVY_XHV,    "cryptonight_haven"),
    # 自动设置 CN_HEAVY_TUBE 的别名
    ALGO_ALIAS_AUTO(CN_HEAVY_TUBE), 
    # 设置 CN_HEAVY_TUBE 的别名为 "cryptonight-heavy/tube"
    ALGO_ALIAS(CN_HEAVY_TUBE,   "cryptonight-heavy/tube"),
    # 设置 CN_HEAVY_TUBE 的别名为 "cryptonight-bittube2"
    ALGO_ALIAS(CN_HEAVY_TUBE,   "cryptonight-bittube2"),
# 如果定义了 XMRIG_ALGO_CN_PICO
    # 自动设置算法别名为 CN_PICO_0
    ALGO_ALIAS_AUTO(CN_PICO_0),
    # 为算法 CN_PICO_0 设置别名 "cryptonight-pico"
    ALGO_ALIAS(CN_PICO_0,       "cryptonight-pico"),
    # 为算法 CN_PICO_0 设置别名 "cn-pico/0"
    ALGO_ALIAS(CN_PICO_0,       "cn-pico/0"),
    # 为算法 CN_PICO_0 设置别名 "cryptonight-pico/trtl"
    ALGO_ALIAS(CN_PICO_0,       "cryptonight-pico/trtl"),
    # 为算法 CN_PICO_0 设置别名 "cn-pico/trtl"
    ALGO_ALIAS(CN_PICO_0,       "cn-pico/trtl"),
    # 为算法 CN_PICO_0 设置别名 "cryptonight-turtle"
    ALGO_ALIAS(CN_PICO_0,       "cryptonight-turtle"),
    # 为算法 CN_PICO_0 设置别名 "cn-trtl"
    ALGO_ALIAS(CN_PICO_0,       "cn-trtl"),
    # 为算法 CN_PICO_0 设置别名 "cryptonight-ultralite"
    ALGO_ALIAS(CN_PICO_0,       "cryptonight-ultralite"),
    # 为算法 CN_PICO_0 设置别名 "cn-ultralite"
    ALGO_ALIAS(CN_PICO_0,       "cn-ultralite"),
    # 为算法 CN_PICO_0 设置别名 "cryptonight_turtle"
    ALGO_ALIAS(CN_PICO_0,       "cryptonight_turtle"),
    # 为算法 CN_PICO_0 设置别名 "cn_turtle"
    ALGO_ALIAS(CN_PICO_0,       "cn_turtle"),
    # 自动设置算法别名为 CN_PICO_TLO
    ALGO_ALIAS_AUTO(CN_PICO_TLO),
    # 为算法 CN_PICO_TLO 设置别名 "cryptonight-pico/tlo"
    ALGO_ALIAS(CN_PICO_TLO,     "cryptonight-pico/tlo"),
    # 为算法 CN_PICO_TLO 设置别名 "cryptonight/ultra"
    ALGO_ALIAS(CN_PICO_TLO,     "cryptonight/ultra"),
    # 为算法 CN_PICO_TLO 设置别名 "cn/ultra"
    ALGO_ALIAS(CN_PICO_TLO,     "cn/ultra"),
    # 为算法 CN_PICO_TLO 设置别名 "cryptonight-talleo"
    ALGO_ALIAS(CN_PICO_TLO,     "cryptonight-talleo"),
    # 为算法 CN_PICO_TLO 设置别名 "cn-talleo"
    ALGO_ALIAS(CN_PICO_TLO,     "cn-talleo"),
    # 为算法 CN_PICO_TLO 设置别名 "cryptonight_talleo"
    ALGO_ALIAS(CN_PICO_TLO,     "cryptonight_talleo"),
    # 为算法 CN_PICO_TLO 设置别名 "cn_talleo"
    ALGO_ALIAS(CN_PICO_TLO,     "cn_talleo"),
#   endif
#   ifdef XMRIG_ALGO_CN_FEMTO
    # 如果定义了XMRIG_ALGO_CN_FEMTO，则执行以下代码
    ALGO_ALIAS_AUTO(CN_UPX2),       # 自动为CN_UPX2生成别名
    ALGO_ALIAS(CN_UPX2,         "cryptonight/upx2"),  # 为CN_UPX2生成别名
                                    ALGO_ALIAS(CN_UPX2,         "cn-extremelite/upx2"),  # 为CN_UPX2生成别名
                                    ALGO_ALIAS(CN_UPX2,         "cryptonight-upx/2"),  # 为CN_UPX2生成别名
#   endif
#   ifdef XMRIG_ALGO_RANDOMX
    # 如果定义了XMRIG_ALGO_RANDOMX，则执行以下代码
    ALGO_ALIAS_AUTO(RX_0),          # 自动为RX_0生成别名
    ALGO_ALIAS(RX_0,            "randomx/0"),  # 为RX_0生成别名
                                    ALGO_ALIAS(RX_0,            "randomx/test"),  # 为RX_0生成别名
                                    ALGO_ALIAS(RX_0,            "rx/test"),  # 为RX_0生成别名
                                    ALGO_ALIAS(RX_0,            "randomx"),  # 为RX_0生成别名
                                    ALGO_ALIAS(RX_0,            "rx"),  # 为RX_0生成别名
    ALGO_ALIAS_AUTO(RX_WOW),        # 自动为RX_WOW生成别名
                                    ALGO_ALIAS(RX_WOW,          "randomx/wow"),  # 为RX_WOW生成别名
                                    ALGO_ALIAS(RX_WOW,          "randomwow"),  # 为RX_WOW生成别名
    ALGO_ALIAS_AUTO(RX_ARQ),        # 自动为RX_ARQ生成别名
                                    ALGO_ALIAS(RX_ARQ,          "randomx/arq"),  # 为RX_ARQ生成别名
                                    ALGO_ALIAS(RX_ARQ,          "randomarq"),  # 为RX_ARQ生成别名
    ALGO_ALIAS_AUTO(RX_GRAFT),      # 自动为RX_GRAFT生成别名
                                    ALGO_ALIAS(RX_GRAFT,        "randomx/graft"),  # 为RX_GRAFT生成别名
                                    ALGO_ALIAS(RX_GRAFT,        "randomgraft"),  # 为RX_GRAFT生成别名
    ALGO_ALIAS_AUTO(RX_SFX),        # 自动为RX_SFX生成别名
                                    ALGO_ALIAS(RX_SFX,          "randomx/sfx"),  # 为RX_SFX生成别名
                                    ALGO_ALIAS(RX_SFX,          "randomsfx"),  # 为RX_SFX生成别名
    ALGO_ALIAS_AUTO(RX_KEVA),       # 自动为RX_KEVA生成别名
                                    ALGO_ALIAS(RX_KEVA,         "randomx/keva"),  # 为RX_KEVA生成别名
                                    ALGO_ALIAS(RX_KEVA,         "randomkeva"),  # 为RX_KEVA生成别名
#   endif
#   ifdef XMRIG_ALGO_ARGON2
    # 如果定义了XMRIG_ALGO_ARGON2，则执行以下代码
    ALGO_ALIAS_AUTO(AR2_CHUKWA),    # 自动为AR2_CHUKWA生成别名
    ALGO_ALIAS(AR2_CHUKWA,      "chukwa"),  # 为AR2_CHUKWA生成别名
    ALGO_ALIAS_AUTO(AR2_CHUKWA_V2), # 自动为AR2_CHUKWA_V2生成别名
    ALGO_ALIAS(AR2_CHUKWA,      "chukwav2"),  # 为AR2_CHUKWA_V2生成别名
    ALGO_ALIAS_AUTO(AR2_WRKZ),      # 自动为AR2_WRKZ生成别名
    ALGO_ALIAS(AR2_WRKZ,        "argon2/wrkz"),  # 为AR2_WRKZ生成别名
#   endif
#   ifdef XMRIG_ALGO_KAWPOW
    # 如果定义了XMRIG_ALGO_KAWPOW，则执行以下代码
    ALGO_ALIAS_AUTO(KAWPOW_RVN),    # 自动为KAWPOW_RVN生成别名
    ALGO_ALIAS(KAWPOW_RVN,      "kawpow/rvn"),  # 为KAWPOW_RVN生成别名
#   endif
#   ifdef XMRIG_ALGO_GHOSTRIDER
    # 自动为 GHOSTRIDER_RTM 添加别名
    ALGO_ALIAS_AUTO(GHOSTRIDER_RTM), 
    # 为 GHOSTRIDER_RTM 添加别名 "ghostrider/rtm"
    ALGO_ALIAS(GHOSTRIDER_RTM, "ghostrider/rtm"),
    # 为 GHOSTRIDER_RTM 添加别名 "gr"
    ALGO_ALIAS(GHOSTRIDER_RTM, "gr"),
// 结束条件
#   endif
};


} /* namespace xmrig */


// 从 JSON 值构造算法对象
xmrig::Algorithm::Algorithm(const rapidjson::Value &value) :
    m_id(parse(value.GetString()))
{
}


// 从 ID 构造算法对象
xmrig::Algorithm::Algorithm(uint32_t id) :
    m_id(kAlgorithmNames.count(id) ? static_cast<Id>(id) : INVALID)
{
}


// 返回算法名称
const char *xmrig::Algorithm::name() const
{
    // 如果算法无效，则返回无效名称
    if (!isValid()) {
        return kINVALID;
    }

    // 断言确保算法名称存在
    assert(kAlgorithmNames.count(m_id));
    const auto it = kAlgorithmNames.find(m_id);

    // 如果存在算法名称，则返回名称，否则返回无效名称
    return it != kAlgorithmNames.end() ? it->second : kINVALID;
}


// 将算法对象转换为 JSON 值
rapidjson::Value xmrig::Algorithm::toJSON() const
{
    using namespace rapidjson;

    // 如果算法有效，则返回名称对应的 JSON 值，否则返回空值
    return isValid() ? Value(StringRef(name())) : Value(kNullType);
}


// 将算法对象转换为 JSON 值
rapidjson::Value xmrig::Algorithm::toJSON(rapidjson::Document &) const
{
    return toJSON();
}


// 解析算法名称并返回对应的 ID
xmrig::Algorithm::Id xmrig::Algorithm::parse(const char *name)
{
    // 如果名称为空或长度小于1，则返回无效 ID
    if (name == nullptr || strlen(name) < 1) {
        return INVALID;
    }

    const auto it = kAlgorithmAliases.find(name);

    // 如果存在对应的别名，则返回对应的 ID，否则返回无效 ID
    return it != kAlgorithmAliases.end() ? it->second : INVALID;
}


// 返回算法数量
size_t xmrig::Algorithm::count()
{
    return kAlgorithmNames.size();
}


// 返回所有算法对象的向量，可根据过滤条件进行筛选
std::vector<xmrig::Algorithm> xmrig::Algorithm::all(const std::function<bool(const Algorithm &algo)> &filter)
{
    static const std::vector<Id> order = {
        CN_0, CN_1, CN_2, CN_R, CN_FAST, CN_HALF, CN_XAO, CN_RTO, CN_RWZ, CN_ZLS, CN_DOUBLE, CN_CCX,
        CN_LITE_0, CN_LITE_1,
        CN_HEAVY_0, CN_HEAVY_TUBE, CN_HEAVY_XHV,
        CN_PICO_0, CN_PICO_TLO,
        CN_UPX2,
        RX_0, RX_WOW, RX_ARQ, RX_GRAFT, RX_SFX, RX_KEVA,
        AR2_CHUKWA, AR2_CHUKWA_V2, AR2_WRKZ,
        KAWPOW_RVN,
        GHOSTRIDER_RTM
    };

    Algorithms out;
    out.reserve(count());

    // 按照指定顺序遍历算法，根据过滤条件筛选并加入结果向量
    for (const Id algo : order) {
        if (kAlgorithmNames.count(algo) && (!filter || filter(algo))) {
            out.emplace_back(algo);
        }
    }

    return out;
}
```