# `xmrig\src\backend\cuda\CudaConfig.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   许可证的版本为3，或者
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CUDACONFIG_H
#define XMRIG_CUDACONFIG_H


#include "backend/cuda/CudaLaunchData.h"
#include "backend/common/Threads.h"
#include "backend/cuda/CudaThreads.h"


namespace xmrig {


class CudaConfig
{
public:
    // 默认构造函数
    CudaConfig() = default;

    // 将CudaConfig对象转换为JSON格式
    rapidjson::Value toJSON(rapidjson::Document &doc) const;
    // 获取CUDA设备的启动数据
    std::vector<CudaLaunchData> get(const Miner *miner, const Algorithm &algorithm, const std::vector<CudaDevice> &devices) const;
    // 从JSON值中读取配置
    void read(const rapidjson::Value &value);

    // 返回是否启用CUDA
    inline bool isEnabled() const                               { return m_enabled; }
    // 返回是否应该保存
    inline bool isShouldSave() const                            { return m_shouldSave; }
    // 返回设备提示
    inline const std::vector<uint32_t> &devicesHint() const     { return m_devicesHint; }
    // 返回加载器
    inline const String &loader() const                         { return m_loader; }
    // 返回CUDA线程
    inline const Threads<CudaThreads> &threads() const          { return m_threads; }
    // 返回bfactor
    inline int32_t bfactor() const                              { return m_bfactor; }
    // 返回bsleep
    inline int32_t bsleep() const                               { return m_bsleep; }

#   ifdef XMRIG_FEATURE_NVML
    # 返回是否启用了 Nvml
    inline bool isNvmlEnabled() const                           { return m_nvml; }
    # 返回 Nvml 加载器的引用
    inline const String &nvmlLoader() const                     { return m_nvmlLoader; }
// 结束条件判断
#   endif

// 私有成员变量和方法声明
private:
    void generate(); // 生成方法
    void setDevicesHint(const char *devicesHint); // 设置设备提示方法

    bool m_enabled          = false; // 是否启用
    bool m_shouldSave       = false; // 是否应该保存
    std::vector<uint32_t> m_devicesHint; // 设备提示
    String m_loader; // 加载器
    Threads<CudaThreads> m_threads; // CUDA 线程

// 根据操作系统不同设置不同的参数
#   ifdef _WIN32
    int32_t m_bfactor      = 6; // Windows 下的 bfactor 参数
    int32_t m_bsleep       = 25; // Windows 下的 bsleep 参数
#   else
    int32_t m_bfactor      = 0; // 非 Windows 下的 bfactor 参数
    int32_t m_bsleep       = 0; // 非 Windows 下的 bsleep 参数
#   endif

// 根据特性开启或关闭对应的参数
#   ifdef XMRIG_FEATURE_NVML
    bool m_nvml            = true; // 是否启用 NVML
    String m_nvmlLoader; // NVML 加载器
#   endif
};


} /* namespace xmrig */

// 结束条件判断
#endif /* XMRIG_CUDACONFIG_H */
```