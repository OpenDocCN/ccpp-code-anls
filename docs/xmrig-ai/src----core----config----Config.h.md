# `xmrig\src\core\config\Config.h`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 要么是许可证的第3版，要么（在您的选择下）是任何以后的版本。
 *
 * 本程序是希望它有用的，
 * 但没有任何保证；甚至没有暗示的保证
 * 商品性或适用于特定目的。有关更多详细信息，请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CONFIG_H
#define XMRIG_CONFIG_H


#include <cstdint>


#include "3rdparty/rapidjson/fwd.h"
#include "backend/cpu/CpuConfig.h"
#include "base/kernel/config/BaseConfig.h"
#include "base/tools/Object.h"


namespace xmrig {


class ConfigPrivate;
class CudaConfig;
class IThread;
class OclConfig;
class RxConfig;


class Config : public BaseConfig
{
public:
    XMRIG_DISABLE_COPY_MOVE(Config);

    static const char *kPauseOnBattery;
    static const char *kPauseOnActive;

#   ifdef XMRIG_FEATURE_OPENCL
    static const char *kOcl;
#   endif

#   ifdef XMRIG_FEATURE_CUDA
    static const char *kCuda;
#   endif

#   if defined(XMRIG_FEATURE_NVML) || defined (XMRIG_FEATURE_ADL)
    static const char *kHealthPrintTime;
#   endif

#   ifdef XMRIG_FEATURE_DMI
    static const char *kDMI;
#   endif

    Config();
    ~Config() override;

    inline bool isPauseOnActive() const { return idleTime() > 0; }

    bool isPauseOnBattery() const;
    const CpuConfig &cpu() const;
    uint32_t idleTime() const;

#   ifdef XMRIG_FEATURE_OPENCL
    const OclConfig &cl() const;
#   endif

#   ifdef XMRIG_FEATURE_CUDA
    const CudaConfig &cuda() const;
// 结束条件判断
#   endif

// 如果定义了 XMRIG_ALGO_RANDOMX，则返回 RxConfig 对象的引用
#   ifdef XMRIG_ALGO_RANDOMX
    const RxConfig &rx() const;
#   endif

// 如果定义了 XMRIG_FEATURE_NVML 或 XMRIG_FEATURE_ADL，则返回健康打印时间
// 否则返回 0
#   if defined(XMRIG_FEATURE_NVML) || defined (XMRIG_FEATURE_ADL)
    uint32_t healthPrintTime() const;
#   else
    uint32_t healthPrintTime() const        { return 0; }
#   endif

// 如果定义了 XMRIG_FEATURE_DMI，则返回是否支持 DMI
// 否则返回 false
#   ifdef XMRIG_FEATURE_DMI
    bool isDMI() const;
#   else
    static constexpr inline bool isDMI()    { return false; }
#   endif

// 返回是否应该保存配置
    bool isShouldSave() const;
    // 从 JSON 读取配置
    bool read(const IJsonReader &reader, const char *fileName) override;
    // 获取配置的 JSON 格式
    void getJSON(rapidjson::Document &doc) const override;

// 私有成员变量
private:
    ConfigPrivate *d_ptr;
};


} /* namespace xmrig */


#endif /* XMRIG_CONFIG_H */
```