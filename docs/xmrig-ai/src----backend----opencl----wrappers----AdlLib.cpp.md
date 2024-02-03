# `xmrig\src\backend\opencl\wrappers\AdlLib.cpp`

```cpp
/* XMRig
 * 版权所有（c）2008-2018 高级微处理器公司
 * 版权所有（c）2018-2021 SChernykh                    <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig                        <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发或修改它
 *   由自由软件基金会发布，无论是许可证的第3版还是
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是在希望它有用的情况下分发的，
 *   但没有任何保证；甚至没有暗示的保证适用于特定目的。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <stdexcept>
#include <uv.h>


#include "backend/opencl/wrappers/AdlLib.h"
#include "3rdparty/adl/adl_sdk.h"
#include "3rdparty/adl/adl_structures.h"
#include "backend/opencl/wrappers/OclDevice.h"


namespace xmrig {


static std::vector<AdapterInfo> adapters;  // 静态变量，存储适配器信息
static uv_lib_t adlLib;  // 静态变量，存储ADL库的句柄


static const char *kSymbolNotFound                          = "symbol not found";  // 定义常量字符串
static const char *kADL_Main_Control_Create                 = "ADL_Main_Control_Create";  // 定义常量字符串
static const char *kADL_Main_Control_Destroy                = "ADL_Main_Control_Destroy";  // 定义常量字符串
static const char *kADL_Adapter_NumberOfAdapters_Get        = "ADL_Adapter_NumberOfAdapters_Get";  // 定义常量字符串
static const char *kADL_Adapter_AdapterInfo_Get             = "ADL_Adapter_AdapterInfo_Get";  // 定义常量字符串
static const char *kADL2_Overdrive_Caps                     = "ADL2_Overdrive_Caps";  // 定义常量字符串
static const char *kADL2_OverdriveN_FanControl_Get          = "ADL2_OverdriveN_FanControl_Get";  // 定义常量字符串
static const char *kADL2_New_QueryPMLogData_Get             = "ADL2_New_QueryPMLogData_Get";  // 定义常量字符串
# 定义常量字符串，用于标识不同的功能
static const char *kADL2_OverdriveN_Temperature_Get         = "ADL2_OverdriveN_Temperature_Get";
static const char *kADL2_OverdriveN_PerformanceStatus_Get   = "ADL2_OverdriveN_PerformanceStatus_Get";
static const char *kADL2_Overdrive6_CurrentPower_Get        = "ADL2_Overdrive6_CurrentPower_Get";

# 定义函数指针类型，用于指向对应的函数
using ADL_MAIN_CONTROL_CREATE               = int (*)(ADL_MAIN_MALLOC_CALLBACK, int);
using ADL_MAIN_CONTROL_DESTROY              = int (*)();
using ADL_ADAPTER_NUMBEROFADAPTERS_GET      = int (*)(int *);
using ADL_ADAPTER_ADAPTERINFO_GET           = int (*)(LPAdapterInfo, int);
using ADL2_OVERDRIVE_CAPS                   = int (*)(ADL_CONTEXT_HANDLE, int, int *, int *, int *);
using ADL2_OVERDRIVEN_FANCONTROL_GET        = int (*)(ADL_CONTEXT_HANDLE, int, ADLODNFanControl *);
using ADL2_NEW_QUERYPMLOGDATA_GET           = int (*)(ADL_CONTEXT_HANDLE, int, ADLPMLogDataOutput *);
using ADL2_OVERDRIVEN_TEMPERATURE_GET       = int (*)(ADL_CONTEXT_HANDLE, int, int, int *);
using ADL2_OVERDRIVEN_PERFORMANCESTATUS_GET = int (*)(ADL_CONTEXT_HANDLE, int, ADLODNPerformanceStatus *);
using ADL2_OVERDRIVE6_CURRENTPOWER_GET      = int (*)(ADL_CONTEXT_HANDLE, int, int, int *);

# 定义函数指针变量，用于指向对应的函数
ADL_MAIN_CONTROL_CREATE                 ADL_Main_Control_Create                 = nullptr;
ADL_MAIN_CONTROL_DESTROY                ADL_Main_Control_Destroy                = nullptr;
ADL_ADAPTER_NUMBEROFADAPTERS_GET        ADL_Adapter_NumberOfAdapters_Get        = nullptr;
ADL_ADAPTER_ADAPTERINFO_GET             ADL_Adapter_AdapterInfo_Get             = nullptr;
ADL2_OVERDRIVE_CAPS                     ADL2_Overdrive_Caps                     = nullptr;
ADL2_OVERDRIVEN_FANCONTROL_GET          ADL2_OverdriveN_FanControl_Get          = nullptr;
ADL2_NEW_QUERYPMLOGDATA_GET             ADL2_New_QueryPMLogData_Get             = nullptr;
ADL2_OVERDRIVEN_TEMPERATURE_GET         ADL2_OverdriveN_Temperature_Get         = nullptr;
ADL2_OVERDRIVEN_PERFORMANCESTATUS_GET   ADL2_OverdriveN_PerformanceStatus_Get   = nullptr;
// 定义指向 ADL2_Overdrive6_CurrentPower_Get 函数的指针
ADL2_OVERDRIVE6_CURRENTPOWER_GET        ADL2_Overdrive6_CurrentPower_Get        = nullptr;

// 定义宏 DLSYM，用于加载动态链接库中的函数指针
#define DLSYM(x) if (uv_dlsym(&adlLib, k##x, reinterpret_cast<void**>(&(x))) == -1) { throw std::runtime_error(kSymbolNotFound); }

// 初始化静态成员变量
bool AdlLib::m_initialized         = false;
bool AdlLib::m_ready               = false;

// 分配内存的回调函数
static void * __stdcall ADL_Main_Memory_Alloc(int iSize)
{
    return malloc(iSize); // NOLINT(cppcoreguidelines-no-malloc, hicpp-no-malloc)
}

// 匹配拓扑结构的辅助函数
static inline bool matchTopology(const PciTopology &topology, const AdapterInfo &adapter)
{
    return adapter.iBusNumber > -1 && topology.bus() == adapter.iBusNumber && topology.device() == adapter.iDeviceNumber && topology.function() == adapter.iFunctionNumber;
}

// 获取风扇信息的辅助函数
static void getFan_v7(const AdapterInfo &adapter, AdlHealth &health)
{
    ADLODNFanControl data;
    memset(&data, 0, sizeof(ADLODNFanControl));

    if (ADL2_OverdriveN_FanControl_Get(nullptr, adapter.iAdapterIndex, &data) == ADL_OK) {
        health.rpm = data.iCurrentFanSpeed;
    }
}

// 获取温度信息的辅助函数
static void getTemp_v7(const AdapterInfo &adapter, AdlHealth &health)
{
    int temp = 0;
    if (ADL2_OverdriveN_Temperature_Get(nullptr, adapter.iAdapterIndex, 1, &temp) == ADL_OK) {
        health.temperature = temp / 1000;
    }
}

// 获取时钟信息的辅助函数
static void getClocks_v7(const AdapterInfo &adapter, AdlHealth &health)
{
    ADLODNPerformanceStatus data;
    memset(&data, 0, sizeof(ADLODNPerformanceStatus));

    if (ADL2_OverdriveN_PerformanceStatus_Get(nullptr, adapter.iAdapterIndex, &data) == ADL_OK) {
        health.clock    = data.iCoreClock / 100;
        health.memClock = data.iMemoryClock / 100;
    }
}

// 获取功耗信息的辅助函数
static void getPower_v7(const AdapterInfo &adapter, AdlHealth &health)
{
    int power = 0;
    if (ADL2_Overdrive6_CurrentPower_Get && ADL2_Overdrive6_CurrentPower_Get(nullptr, adapter.iAdapterIndex, 0, &power) == ADL_OK) {
        health.power = static_cast<uint32_t>(power / 256.0);
    }
}
static void getSensorsData_v8(const AdapterInfo &adapter, AdlHealth &health)
{
    // 如果 ADL2_New_QueryPMLogData_Get 为空，则返回
    if (!ADL2_New_QueryPMLogData_Get) {
        return;
    }

    // 初始化并清零 ADLPMLogDataOutput 结构体
    ADLPMLogDataOutput data;
    memset(&data, 0, sizeof(ADLPMLogDataOutput));

    // 如果无法获取 PM 日志数据，则返回
    if (ADL2_New_QueryPMLogData_Get(nullptr, adapter.iAdapterIndex, &data) != ADL_OK) {
        return;
    }

    // 使用 lambda 表达式获取传感器数值
    auto sensorValue = [&data](ADLSensorType type) { return data.sensors[type].supported ? data.sensors[type].value : 0; };

    // 获取各项健康数据
    health.clock        = sensorValue(PMLOG_CLK_GFXCLK);
    health.memClock     = sensorValue(PMLOG_CLK_MEMCLK);
    health.power        = sensorValue(PMLOG_ASIC_POWER);
    health.rpm          = sensorValue(PMLOG_FAN_RPM);
    health.temperature  = sensorValue(PMLOG_TEMPERATURE_HOTSPOT);
}


} // namespace xmrig


bool xmrig::AdlLib::init()
{
    // 如果未初始化，则加载动态链接库并进行初始化
    if (!m_initialized) {
        m_ready       = dlopen() && load();
        m_initialized = true;
    }

    return m_ready;
}


const char *xmrig::AdlLib::lastError() noexcept
{
    // 返回最后一个错误信息
    return uv_dlerror(&adlLib);
}


void xmrig::AdlLib::close()
{
    // 如果已准备就绪，则销毁主控制
    if (m_ready) {
        ADL_Main_Control_Destroy();
    }

    // 关闭动态链接库
    uv_dlclose(&adlLib);
}


AdlHealth xmrig::AdlLib::health(const OclDevice &device)
{
    // 如果未准备就绪或设备厂商不是 AMD，则返回空的 AdlHealth 对象
    if (!isReady() || device.vendorId() != OCL_VENDOR_AMD) {
        return {};
    }

    int supported   = 0;
    int enabled     = 0;
    int version     = 0;
    AdlHealth health;
    // 遍历适配器列表，对每个适配器进行操作
    for (const auto &adapter : adapters) {
        // 检查设备拓扑结构是否匹配当前适配器
        if (matchTopology(device.topology(), adapter)) {
            // 检查适配器是否支持 Overdrive 功能，并获取相关信息
            if (ADL2_Overdrive_Caps(nullptr, adapter.iAdapterIndex, &supported, &enabled, &version) != ADL_OK) {
                // 如果获取失败，则跳过当前适配器，继续下一个
                continue;
            }

            // 根据 Overdrive 版本不同，执行不同的操作
            if (version == 7) {
                // 如果版本为 7，则获取风扇、温度、时钟和功耗信息
                getFan_v7(adapter, health);
                getTemp_v7(adapter, health);
                getClocks_v7(adapter, health);
                getPower_v7(adapter, health);
            }
            else if (version == 8) {
                // 如果版本为 8，则获取传感器数据
                getSensorsData_v8(adapter, health);
            }

            // 找到匹配的适配器后，结束循环
            break;
        }
    }

    // 返回健康信息
    return health;
# 定义了一个名为dlopen的函数，返回一个布尔值
bool xmrig::AdlLib::dlopen()
{
    # 调用uv_dlopen函数加载"atiadlxx.dll"库，并将结果存储到adlLib指针中，如果返回值为0则表示加载成功
    return uv_dlopen("atiadlxx.dll", &adlLib) == 0;
}

# 定义了一个名为load的函数，返回一个布尔值
bool xmrig::AdlLib::load()
{
    try {
        # 尝试动态加载ADL_Main_Control_Create函数
        DLSYM(ADL_Main_Control_Create);
        # 尝试动态加载ADL_Main_Control_Destroy函数
        DLSYM(ADL_Main_Control_Destroy);
        # 尝试动态加载ADL_Adapter_NumberOfAdapters_Get函数
        DLSYM(ADL_Adapter_NumberOfAdapters_Get);
        # 尝试动态加载ADL_Adapter_AdapterInfo_Get函数
        DLSYM(ADL_Adapter_AdapterInfo_Get);
        # 尝试动态加载ADL2_Overdrive_Caps函数
        DLSYM(ADL2_Overdrive_Caps);
        # 尝试动态加载ADL2_OverdriveN_FanControl_Get函数
        DLSYM(ADL2_OverdriveN_FanControl_Get);
        # 尝试动态加载ADL2_OverdriveN_Temperature_Get函数
        DLSYM(ADL2_OverdriveN_Temperature_Get);
        # 尝试动态加载ADL2_OverdriveN_PerformanceStatus_Get函数
        DLSYM(ADL2_OverdriveN_PerformanceStatus_Get);
    } catch (std::exception &ex) {
        # 如果发生异常，则返回false
        return false;
    }

    try {
        # 尝试动态加载ADL2_Overdrive6_CurrentPower_Get函数
        DLSYM(ADL2_Overdrive6_CurrentPower_Get);
        # 尝试动态加载ADL2_New_QueryPMLogData_Get函数
        DLSYM(ADL2_New_QueryPMLogData_Get);
    } catch (std::exception &ex) {}

    # 调用ADL_Main_Control_Create函数，如果返回值不等于ADL_OK，则返回false
    if (ADL_Main_Control_Create(ADL_Main_Memory_Alloc, 1) != ADL_OK) {
        return false;
    }

    # 定义一个整型变量count，并初始化为0
    int count = 0;
    # 调用ADL_Adapter_NumberOfAdapters_Get函数，如果返回值不等于ADL_OK，则返回false
    if (ADL_Adapter_NumberOfAdapters_Get(&count) != ADL_OK) {
        return false;
    }

    # 如果count等于0，则返回false
    if (count == 0) {
        return false;
    }

    # 调整adapters的大小为count
    adapters.resize(count);
    # 计算adapters的大小，并用0填充
    const size_t size = sizeof(adapters[0]) * adapters.size();
    memset(adapters.data(), 0, size);

    # 调用ADL_Adapter_AdapterInfo_Get函数，如果返回值不等于ADL_OK，则返回false
    return ADL_Adapter_AdapterInfo_Get(adapters.data(), size) == ADL_OK;
}
```