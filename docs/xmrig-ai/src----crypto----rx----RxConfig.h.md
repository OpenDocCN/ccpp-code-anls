# `xmrig\src\crypto\rx\RxConfig.h`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以自由地重新分发和/或修改
 *   根据 GNU 通用公共许可证的条款，发布的版本为
 *   3 或者 (根据您的选择) 任何更高版本。
 *
 *   本程序是基于希望它有用的目的分发的，
 *   但没有任何担保；甚至没有适用于特定目的的隐含担保。
 *   有关更多详细信息，请参阅 GNU 通用公共许可证。
 *
 *   如果没有收到 GNU 通用公共许可证的副本，
 *   请参阅 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_RXCONFIG_H
#define XMRIG_RXCONFIG_H


#include "3rdparty/rapidjson/fwd.h"


#ifdef XMRIG_FEATURE_MSR
#   include "hw/msr/MsrItem.h"
#endif


#include <vector>


namespace xmrig {


class RxConfig
{
public:
    enum Mode : uint32_t {
        AutoMode,
        FastMode,
        LightMode,
        ModeMax
    };

    enum ScratchpadPrefetchMode : uint32_t {
        ScratchpadPrefetchOff,
        ScratchpadPrefetchT0,
        ScratchpadPrefetchNTA,
        ScratchpadPrefetchMov,
        ScratchpadPrefetchMax,
    };

    static const char *kCacheQoS;
    static const char *kField;
    static const char *kInit;
    static const char *kInitAVX2;
    static const char *kMode;
    static const char *kOneGbPages;
    static const char *kRdmsr;
    static const char *kScratchpadPrefetchMode;
    static const char *kWrmsr;

#   ifdef XMRIG_FEATURE_HWLOC
    static const char *kNUMA;
#   endif

    // 从 JSON 对象中读取配置
    bool read(const rapidjson::Value &value);
    // 将配置转换为 JSON 对象
    rapidjson::Value toJSON(rapidjson::Document &doc) const;

#   ifdef XMRIG_FEATURE_HWLOC
    // 获取 NUMA 节点集合
    std::vector<uint32_t> nodeset() const;
#   else
    # 定义一个内联函数nodeset，返回一个空的无符号整数向量。
# 定义一个命名空间 xmrig
namespace xmrig {

# 定义一个类 RxConfig
class RxConfig {
public:
    # 定义一个常量字符指针函数 modeName，返回值为 const char *
    const char *modeName() const;
    # 定义一个无符号整型函数 threads，参数 limit 的默认值为 100，返回值为 uint32_t
    uint32_t threads(uint32_t limit = 100) const;

    # 定义一个内联函数 initDatasetAVX2，返回值为 int
    inline int initDatasetAVX2() const  { return m_initDatasetAVX2; }
    # 定义一个内联函数 isOneGbPages，返回值为 bool
    inline bool isOneGbPages() const    { return m_oneGbPages; }
    # 定义一个内联函数 rdmsr，返回值为 bool
    inline bool rdmsr() const           { return m_rdmsr; }
    # 定义一个内联函数 wrmsr，返回值为 bool
    inline bool wrmsr() const           { return m_wrmsr; }
    # 定义一个内联函数 cacheQoS，返回值为 bool
    inline bool cacheQoS() const        { return m_cacheQoS; }
    # 定义一个内联函数 mode，返回值为 Mode
    inline Mode mode() const            { return m_mode; }

    # 定义一个内联函数 scratchpadPrefetchMode，返回值为 ScratchpadPrefetchMode
    inline ScratchpadPrefetchMode scratchpadPrefetchMode() const { return m_scratchpadPrefetchMode; }

    # 如果定义了 XMRIG_FEATURE_MSR
    # 定义一个常量字符指针函数 msrPresetName，返回值为 const char *
    const char *msrPresetName() const;
    # 定义一个常量引用函数 msrPreset，返回值为 const MsrItems &
    const MsrItems &msrPreset() const;
    # 定义一个无符号整型函数 msrMod，返回值为 uint32_t
    uint32_t msrMod() const;
    # 定义一个读取 MSR 寄存器的函数，参数 value 为 rapidjson::Value 类型
    void readMSR(const rapidjson::Value &value);

    # 如果没有定义 XMRIG_FEATURE_MSR
    # 定义一个内联函数 wrmsr，返回值为 false
    bool m_wrmsr = false;

    # 定义一个内联函数 cacheQoS，返回值为 false
    bool m_cacheQoS = false;

    # 定义一个静态函数 readMode，参数 value 为 rapidjson::Value 类型，返回值为 Mode
    static Mode readMode(const rapidjson::Value &value);

    # 定义一个内联函数 isOneGbPages，返回值为 false
    bool m_oneGbPages     = false;
    # 定义一个内联函数 rdmsr，返回值为 true
    bool m_rdmsr          = true;
    # 定义一个整型变量 threads，初始值为 -1
    int m_threads         = -1;
    # 定义一个整型变量 initDatasetAVX2，初始值为 -1
    int m_initDatasetAVX2 = -1;
    # 定义一个 Mode 类型变量 mode，初始值为 AutoMode
    Mode m_mode           = AutoMode;

    # 定义一个 ScratchpadPrefetchMode 类型变量 scratchpadPrefetchMode，初始值为 ScratchpadPrefetchT0
    ScratchpadPrefetchMode m_scratchpadPrefetchMode = ScratchpadPrefetchT0;

    # 如果定义了 XMRIG_FEATURE_HWLOC
    # 定义一个内联函数 numa，返回值为 true
    bool m_numa           = true;
    # 定义一个存储 uint32_t 类型的向量 m_nodeset
    std::vector<uint32_t> m_nodeset;

};

} // namespace xmrig

# 结束定义
#endif // XMRIG_RXCONFIG_H
```