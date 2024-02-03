# `xmrig\src\backend\cuda\wrappers\CudaLib.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有对适销性或特定用途的隐含保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CUDALIB_H
#define XMRIG_CUDALIB_H


using nvid_ctx = struct nvid_ctx;


#include "backend/cuda/wrappers/CudaDevice.h"
#include "base/tools/String.h"


#include <vector>
#include <string>


namespace xmrig {


class CudaLib
{
public:
    enum DeviceProperty : uint32_t
    {
        DeviceId,
        DeviceAlgorithm,
        DeviceArchMajor,
        DeviceArchMinor,
        DeviceSmx,
        DeviceBlocks,
        DeviceThreads,
        DeviceBFactor,
        DeviceBSleep,
        DeviceClockRate,
        DeviceMemoryClockRate,
        DeviceMemoryTotal,
        DeviceMemoryFree,
        DevicePciBusID,
        DevicePciDeviceID,
        DevicePciDomainID,
        DeviceDatasetHost,
    };

    static bool init(const char *fileName = nullptr);  // 初始化CUDA库，可选参数为文件名
    static const char *lastError() noexcept;  // 返回最后一个错误信息
    static void close();  // 关闭CUDA库

    static inline bool isInitialized()    { return m_initialized; }  // 返回CUDA库是否已初始化
    static inline bool isReady() noexcept { return m_ready; }  // 返回CUDA库是否准备就绪
    static inline const String &loader()  { return m_loader; }  // 返回CUDA库加载器
    # 定义一个静态函数，用于计算 cnHash
    static bool cnHash(nvid_ctx *ctx, uint32_t startNonce, uint64_t height, uint64_t target, uint32_t *rescount, uint32_t *resnonce);
    
    # 定义一个静态函数，用于获取设备信息
    static bool deviceInfo(nvid_ctx *ctx, int32_t blocks, int32_t threads, const Algorithm &algorithm, int32_t dataset_host = -1) noexcept;
    
    # 定义一个静态函数，用于初始化设备
    static bool deviceInit(nvid_ctx *ctx) noexcept;
    
    # 定义一个静态函数，用于计算 rxHash
    static bool rxHash(nvid_ctx *ctx, uint32_t startNonce, uint64_t target, uint32_t *rescount, uint32_t *resnonce) noexcept;
    
    # 定义一个静态函数，用于准备 rxHash 所需的数据
    static bool rxPrepare(nvid_ctx *ctx, const void *dataset, size_t datasetSize, bool dataset_host, uint32_t batchSize) noexcept;
    
    # 定义一个静态函数，用于计算 kawPowHash
    static bool kawPowHash(nvid_ctx *ctx, uint8_t* job_blob, uint64_t target, uint32_t *rescount, uint32_t *resnonce, uint32_t *skipped_hashes) noexcept;
    
    # 定义一个静态函数，用于准备 kawPowHash 所需的数据
    static bool kawPowPrepare(nvid_ctx *ctx, const void* cache, size_t cache_size, const void* dag_precalc, size_t dag_size, uint32_t height, const uint64_t* dag_sizes) noexcept;
    
    # 定义一个静态函数，用于停止 kawPowHash 的计算
    static bool kawPowStopHash(nvid_ctx *ctx) noexcept;
    
    # 定义一个静态函数，用于设置作业
    static bool setJob(nvid_ctx *ctx, const void *data, size_t size, const Algorithm &algorithm) noexcept;
    
    # 定义一个静态函数，用于获取设备名称
    static const char *deviceName(nvid_ctx *ctx) noexcept;
    
    # 定义一个静态函数，用于获取最后的错误信息
    static const char *lastError(nvid_ctx *ctx) noexcept;
    
    # 定义一个静态函数，用于获取插件版本
    static const char *pluginVersion() noexcept;
    
    # 定义一个静态函数，用于获取设备整型属性
    static int32_t deviceInt(nvid_ctx *ctx, DeviceProperty property) noexcept;
    
    # 定义一个静态函数，用于分配内存并初始化设备上下文
    static nvid_ctx *alloc(uint32_t id, int32_t bfactor, int32_t bsleep) noexcept;
    
    # 定义一个静态函数，用于获取版本信息
    static std::string version(uint32_t version);
    
    # 定义一个静态函数，用于获取设备列表
    static std::vector<CudaDevice> devices(int32_t bfactor, int32_t bsleep, const std::vector<uint32_t> &hints) noexcept;
    
    # 定义一个静态函数，用于获取设备数量
    static uint32_t deviceCount() noexcept;
    
    # 定义一个静态函数，用于获取设备无符号整型属性
    static uint32_t deviceUint(nvid_ctx *ctx, DeviceProperty property) noexcept;
    
    # 定义一个静态函数，用于获取驱动版本
    static uint32_t driverVersion() noexcept;
    
    # 定义一个静态函数，用于获取运行时版本
    static uint32_t runtimeVersion() noexcept;
    
    # 定义一个静态函数，用于获取设备无符号长整型属性
    static uint64_t deviceUlong(nvid_ctx *ctx, DeviceProperty property) noexcept;
    
    # 定义一个静态函数，用于释放设备上下文
    static void release(nvid_ctx *ctx) noexcept;
# 声明私有成员和静态成员函数
private:
    static bool open();  # 声明静态成员函数 open()，返回布尔类型
    static void load();  # 声明静态成员函数 load()，无返回值

    static bool m_initialized;  # 声明静态布尔类型成员变量 m_initialized
    static bool m_ready;  # 声明静态布尔类型成员变量 m_ready
    static String m_error;  # 声明静态字符串类型成员变量 m_error
    static String m_loader;  # 声明静态字符串类型成员变量 m_loader
};


} // namespace xmrig  # 结束 xmrig 命名空间


#endif /* XMRIG_CUDALIB_H */  # 结束头文件定义
```