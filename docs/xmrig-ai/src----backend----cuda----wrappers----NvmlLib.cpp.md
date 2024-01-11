# `xmrig\src\backend\cuda\wrappers\NvmlLib.cpp`

```
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据 GNU 通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版，或者
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include <stdexcept>
#include <uv.h>


#include "backend/cuda/wrappers/NvmlLib.h"
#include "backend/cuda/wrappers/nvml_lite.h"
#include "base/io/log/Log.h"



namespace xmrig {


static uv_lib_t nvmlLib;


static const char *kNvmlDeviceGetClockInfo                          = "nvmlDeviceGetClockInfo";
static const char *kNvmlDeviceGetCount                              = "nvmlDeviceGetCount_v2";
static const char *kNvmlDeviceGetFanSpeed                           = "nvmlDeviceGetFanSpeed";
static const char *kNvmlDeviceGetFanSpeed_v2                        = "nvmlDeviceGetFanSpeed_v2";
static const char *kNvmlDeviceGetHandleByIndex                      = "nvmlDeviceGetHandleByIndex_v2";
static const char *kNvmlDeviceGetPciInfo                            = "nvmlDeviceGetPciInfo_v2";
static const char *kNvmlDeviceGetPowerUsage                         = "nvmlDeviceGetPowerUsage";
static const char *kNvmlDeviceGetTemperature                        = "nvmlDeviceGetTemperature";
static const char *kNvmlInit                                        = "nvmlInit_v2";
static const char *kNvmlShutdown                                    = "nvmlShutdown";
// 定义常量字符串，用于标识不同的 NVML 函数
static const char *kNvmlSystemGetDriverVersion                      = "nvmlSystemGetDriverVersion";
static const char *kNvmlSystemGetNVMLVersion                        = "nvmlSystemGetNVMLVersion";
static const char *kSymbolNotFound                                  = "symbol not found";

// 定义函数指针，用于动态加载 NVML 库中的函数
static nvmlReturn_t (*pNvmlDeviceGetClockInfo)(nvmlDevice_t device, uint32_t type, uint32_t *clock)                         = nullptr;
static nvmlReturn_t (*pNvmlDeviceGetCount)(uint32_t *deviceCount)                                                           = nullptr;
static nvmlReturn_t (*pNvmlDeviceGetFanSpeed_v2)(nvmlDevice_t device, uint32_t fan, uint32_t *speed)                        = nullptr;
static nvmlReturn_t (*pNvmlDeviceGetFanSpeed)(nvmlDevice_t device, uint32_t *speed)                                         = nullptr;
static nvmlReturn_t (*pNvmlDeviceGetHandleByIndex)(uint32_t index, nvmlDevice_t *device)                                    = nullptr;
static nvmlReturn_t (*pNvmlDeviceGetPciInfo)(nvmlDevice_t device, nvmlPciInfo_t *pci)                                       = nullptr;
static nvmlReturn_t (*pNvmlDeviceGetPowerUsage)(nvmlDevice_t device, uint32_t *power)                                       = nullptr;
static nvmlReturn_t (*pNvmlDeviceGetTemperature)(nvmlDevice_t device, uint32_t sensorType, uint32_t *temp)                  = nullptr;
static nvmlReturn_t (*pNvmlInit)()                                                                                          = nullptr;
static nvmlReturn_t (*pNvmlShutdown)()                                                                                      = nullptr;
static nvmlReturn_t (*pNvmlSystemGetDriverVersion)(char *version, uint32_t length)                                          = nullptr;
static nvmlReturn_t (*pNvmlSystemGetNVMLVersion)(char *version, uint32_t length)                                            = nullptr;
// 定义宏，用于动态加载符号
#define DLSYM(x) if (uv_dlsym(&nvmlLib, k##x, reinterpret_cast<void**>(&p##x)) == -1) { throw std::runtime_error(kSymbolNotFound); }

// NvmlLib 类的静态成员变量初始化
bool NvmlLib::m_initialized         = false;
bool NvmlLib::m_ready               = false;
char NvmlLib::m_driverVersion[80]   = { 0 };
char NvmlLib::m_nvmlVersion[80]     = { 0 };
String NvmlLib::m_loader;

// xmrig 命名空间
} // namespace xmrig

// 初始化 NvmlLib
bool xmrig::NvmlLib::init(const char *fileName)
{
    // 如果未初始化，则进行初始化
    if (!m_initialized) {
        m_loader      = fileName;
        m_ready       = dlopen() && load();
        m_initialized = true;
    }

    return m_ready;
}

// 获取最后的错误信息
const char *xmrig::NvmlLib::lastError() noexcept
{
    return uv_dlerror(&nvmlLib);
}

// 关闭 NvmlLib
void xmrig::NvmlLib::close()
{
    // 如果已经准备好，则关闭 Nvml
    if (m_ready) {
        pNvmlShutdown();
    }

    uv_dlclose(&nvmlLib);
}

// 分配设备
bool xmrig::NvmlLib::assign(std::vector<CudaDevice> &devices)
{
    uint32_t count = 0;
    // 获取设备数量
    if (pNvmlDeviceGetCount(&count) != NVML_SUCCESS) {
        return false;
    }

    // 遍历设备
    for (uint32_t i = 0; i < count; i++) {
        nvmlDevice_t nvmlDevice = nullptr;
        // 获取设备句柄
        if (pNvmlDeviceGetHandleByIndex(i, &nvmlDevice) != NVML_SUCCESS) {
            continue;
        }

        nvmlPciInfo_t pci;
        // 获取设备的 PCI 信息
        if (pNvmlDeviceGetPciInfo(nvmlDevice, &pci) != NVML_SUCCESS) {
            continue;
        }

        // 遍历设备列表，将 Nvml 设备分配给对应的 Cuda 设备
        for (auto &device : devices) {
            if (device.topology().bus() == pci.bus && device.topology().device() == pci.device) {
                device.setNvmlDevice(nvmlDevice);
            }
        }
    }

    return true;
}

// 获取设备健康信息
NvmlHealth xmrig::NvmlLib::health(nvmlDevice_t device)
{
    if (!device) {
        return {};
    }

    NvmlHealth health;
    // 获取设备温度、功耗、时钟信息
    pNvmlDeviceGetTemperature(device, NVML_TEMPERATURE_GPU, &health.temperature);
    pNvmlDeviceGetPowerUsage(device, &health.power);
    pNvmlDeviceGetClockInfo(device, NVML_CLOCK_SM, &health.clock);
    pNvmlDeviceGetClockInfo(device, NVML_CLOCK_MEM, &health.memClock);

    // 将功耗转换为瓦特
    if (health.power) {
        health.power /= 1000;
    }
}
    # 定义一个无符号32位整数变量speed，初始化为0
    uint32_t speed = 0;

    # 如果pNvmlDeviceGetFanSpeed_v2函数存在
    if (pNvmlDeviceGetFanSpeed_v2) {
        # 定义一个无符号32位整数变量i，初始化为0
        uint32_t i = 0;

        # 当调用pNvmlDeviceGetFanSpeed_v2函数成功时，循环执行以下操作
        while (pNvmlDeviceGetFanSpeed_v2(device, i, &speed) == NVML_SUCCESS) {
            # 将获取到的风扇速度数据添加到health.fanSpeed向量中
            health.fanSpeed.push_back(speed);
            # i自增1
            ++i;
        }

    }
    # 如果pNvmlDeviceGetFanSpeed_v2函数不存在
    else {
        # 调用pNvmlDeviceGetFanSpeed函数获取风扇速度数据
        pNvmlDeviceGetFanSpeed(device, &speed);

        # 将获取到的风扇速度数据添加到health.fanSpeed向量中
        health.fanSpeed.push_back(speed);
    }

    # 返回health对象
    return health;
}

bool xmrig::NvmlLib::dlopen()
{
    // 如果加载器不为空，则尝试使用 uv_dlopen 加载 nvmlLib，并返回结果
    if (!m_loader.isNull()) {
        return uv_dlopen(m_loader, &nvmlLib) == 0;
    }

#   ifdef _WIN32
    // 如果是 Windows 平台，则尝试加载 nvml.dll，加载成功则返回 true
    if (uv_dlopen("nvml.dll", &nvmlLib) == 0) {
        return true;
    }

    // 如果加载失败，则尝试使用环境变量获取 nvml.dll 的路径
    char path[MAX_PATH] = { 0 };
    ExpandEnvironmentStringsA("%PROGRAMFILES%\\NVIDIA Corporation\\NVSMI\\nvml.dll", path, sizeof(path));

    // 使用获取到的路径加载 nvml.dll，加载成功则返回 true
    return uv_dlopen(path, &nvmlLib) == 0;
#   else
    // 如果不是 Windows 平台，则尝试加载 libnvidia-ml.so，加载成功则返回 true
    return uv_dlopen("libnvidia-ml.so", &nvmlLib) == 0;
#   endif
}

bool xmrig::NvmlLib::load()
{
    try {
        // 尝试使用 DLSYM 加载一系列函数
        DLSYM(NvmlDeviceGetClockInfo);
        DLSYM(NvmlDeviceGetCount);
        DLSYM(NvmlDeviceGetFanSpeed);
        DLSYM(NvmlDeviceGetHandleByIndex);
        DLSYM(NvmlDeviceGetPciInfo);
        DLSYM(NvmlDeviceGetPowerUsage);
        DLSYM(NvmlDeviceGetTemperature);
        DLSYM(NvmlInit);
        DLSYM(NvmlShutdown);
        DLSYM(NvmlSystemGetDriverVersion);
        DLSYM(NvmlSystemGetNVMLVersion);
    } catch (std::exception &ex) {
        // 如果加载函数出现异常，则返回 false
        return false;
    }

    // 使用 uv_dlsym 加载 kNvmlDeviceGetFanSpeed_v2 函数
    uv_dlsym(&nvmlLib, kNvmlDeviceGetFanSpeed_v2, reinterpret_cast<void**>(&pNvmlDeviceGetFanSpeed_v2));

    // 如果 pNvmlInit() 返回值不是 NVML_SUCCESS，则返回 false
    if (pNvmlInit() != NVML_SUCCESS) {
        return false;
    }

    // 获取驱动版本和 NVML 版本
    pNvmlSystemGetDriverVersion(m_driverVersion, sizeof(m_driverVersion));
    pNvmlSystemGetNVMLVersion(m_nvmlVersion, sizeof(m_nvmlVersion));

    // 加载成功，返回 true
    return true;
}
```