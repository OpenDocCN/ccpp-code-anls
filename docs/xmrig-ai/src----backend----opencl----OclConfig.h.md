# `xmrig\src\backend\opencl\OclConfig.h`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 它可以是许可证的第3版，也可以是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 商品性或适用于特定目的。有关更多详细信息，请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_OCLCONFIG_H
#define XMRIG_OCLCONFIG_H


#include "backend/common/Threads.h"
#include "backend/opencl/OclLaunchData.h"
#include "backend/opencl/OclThreads.h"
#include "backend/opencl/wrappers/OclPlatform.h"


namespace xmrig {


class OclConfig
{
public:
    // 构造函数
    OclConfig();

    // 获取平台
    OclPlatform platform() const;
    // 转换为JSON格式
    rapidjson::Value toJSON(rapidjson::Document &doc) const;
    // 获取OpenCL启动数据
    std::vector<OclLaunchData> get(const Miner *miner, const Algorithm &algorithm, const OclPlatform &platform, const std::vector<OclDevice> &devices) const;
    // 从JSON值中读取配置
    void read(const rapidjson::Value &value);

    // 是否启用缓存
    inline bool isCacheEnabled() const                  { return m_cache; }
    // 是否启用
    inline bool isEnabled() const                       { return m_enabled; }
    // 是否应该保存
    inline bool isShouldSave() const                    { return m_shouldSave; }
    // 加载器
    inline const String &loader() const                 { return m_loader; }
    // 线程
    inline const Threads<OclThreads> &threads() const   { return m_threads; }

#   ifdef XMRIG_FEATURE_ADL
    // 是否启用ADL
    inline bool isAdlEnabled() const                    { return m_adl; }
#   endif

private:
    // 生成配置
    void generate();
    # 设置设备提示信息，参数为设备提示字符串
    void setDevicesHint(const char *devicesHint);
    
    # 缓存标志，默认为true
    bool m_cache         = true;
    # 启用标志，默认为false
    bool m_enabled       = false;
    # 是否应该保存标志，默认为false
    bool m_shouldSave    = false;
    # 设备提示信息的向量
    std::vector<uint32_t> m_devicesHint;
    # 加载器字符串
    String m_loader;
    # OclThreads类型的线程集合
    Threads<OclThreads> m_threads;
#   ifndef XMRIG_OS_APPLE
    // 如果不是苹果操作系统，则定义 setPlatform 函数
    void setPlatform(const rapidjson::Value &platform);

    // 定义平台供应商字符串
    String m_platformVendor;
    // 定义平台索引，默认为0
    uint32_t m_platformIndex = 0;
#   endif

#   ifdef XMRIG_FEATURE_ADL
    // 如果支持 ADL 特性，则定义 m_adl 变量，默认为 true
    bool m_adl          = true;
#   endif
};


} /* namespace xmrig */


#endif /* XMRIG_OCLCONFIG_H */
```