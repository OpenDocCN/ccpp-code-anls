# `xmrig\src\backend\cuda\wrappers\CudaLib.cpp`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是基于希望它有用而分发的，
 *   但没有任何保证；甚至没有暗示的保证适用于特定目的。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <stdexcept>
#include <uv.h>


#include "backend/cuda/wrappers/CudaLib.h"
#include "base/io/Env.h"
#include "base/io/log/Log.h"
#include "base/kernel/Process.h"
#include "crypto/rx/RxAlgo.h"


namespace xmrig {


enum Version : uint32_t
{
    ApiVersion,
    DriverVersion,
    RuntimeVersion
};


static uv_lib_t cudaLib;

#if defined(__APPLE__)
static String defaultLoader = "libxmrig-cuda.dylib";
#elif defined(_WIN32)
static String defaultLoader = "xmrig-cuda.dll";
#else
static String defaultLoader = "libxmrig-cuda.so";
#endif


static const char *kAlloc                               = "alloc";
static const char *kCnHash                              = "cnHash";
static const char *kDeviceCount                         = "deviceCount";
static const char *kDeviceInfo                          = "deviceInfo";
static const char *kDeviceInfo_v2                       = "deviceInfo_v2";
static const char *kDeviceInit                          = "deviceInit";
static const char *kDeviceInt                           = "deviceInt";
static const char *kDeviceName                          = "deviceName";
// 定义静态常量字符串，表示不同的设备属性或函数名称
static const char *kDeviceUint                          = "deviceUint";
static const char *kDeviceUlong                         = "deviceUlong";
static const char *kInit                                = "init";
static const char *kKawPowHash                          = "kawPowHash";
static const char *kKawPowPrepare_v2                    = "kawPowPrepare_v2";
static const char *kKawPowStopHash                      = "kawPowStopHash";
static const char *kLastError                           = "lastError";
static const char *kPluginVersion                       = "pluginVersion";
static const char *kRelease                             = "release";
static const char *kRxHash                              = "rxHash";
static const char *kRxPrepare                           = "rxPrepare";
static const char *kSetJob                              = "setJob";
static const char *kSetJob_v2                           = "setJob_v2";
static const char *kVersion                             = "version";

// 使用别名定义不同的函数指针类型
using alloc_t                                           = nvid_ctx * (*)(uint32_t, int32_t, int32_t);
using cnHash_t                                          = bool (*)(nvid_ctx *, uint32_t, uint64_t, uint64_t, uint32_t *, uint32_t *);
using deviceCount_t                                     = uint32_t (*)();
using deviceInfo_t                                      = bool (*)(nvid_ctx *, int32_t, int32_t, uint32_t, int32_t);
using deviceInfo_v2_t                                   = bool (*)(nvid_ctx *, int32_t, int32_t, const char *, int32_t);
using deviceInit_t                                      = bool (*)(nvid_ctx *);
using deviceInt_t                                       = int32_t (*)(nvid_ctx *, CudaLib::DeviceProperty);
using deviceName_t                                      = const char * (*)(nvid_ctx *);
using deviceUint_t                                      = uint32_t (*)(nvid_ctx *, CudaLib::DeviceProperty);
# 定义函数指针类型 deviceUlong_t，指向返回 uint64_t 类型的函数，参数为 nvid_ctx *, CudaLib::DeviceProperty
using deviceUlong_t                                     = uint64_t (*)(nvid_ctx *, CudaLib::DeviceProperty);
# 定义函数指针类型 init_t，指向返回 void 类型的函数，无参数
using init_t                                            = void (*)();
# 定义函数指针类型 kawPowHash_t，指向返回 bool 类型的函数，参数为 nvid_ctx *, uint8_t*, uint64_t, uint32_t *, uint32_t *, uint32_t *
using kawPowHash_t                                      = bool (*)(nvid_ctx *, uint8_t*, uint64_t, uint32_t *, uint32_t *, uint32_t *);
# 定义函数指针类型 kawPowPrepare_v2_t，指向返回 bool 类型的函数，参数为 nvid_ctx *, const void *, size_t, const void *, size_t, uint32_t, const uint64_t*
using kawPowPrepare_v2_t                                = bool (*)(nvid_ctx *, const void *, size_t, const void *, size_t, uint32_t, const uint64_t*);
# 定义函数指针类型 kawPowStopHash_t，指向返回 bool 类型的函数，参数为 nvid_ctx *
using kawPowStopHash_t                                  = bool (*)(nvid_ctx *);
# 定义函数指针类型 lastError_t，指向返回 const char * 类型的函数，参数为 nvid_ctx *
using lastError_t                                       = const char * (*)(nvid_ctx *);
# 定义函数指针类型 pluginVersion_t，指向返回 const char * 类型的函数，无参数
using pluginVersion_t                                   = const char * (*)();
# 定义函数指针类型 release_t，指向返回 void 类型的函数，参数为 nvid_ctx *
using release_t                                         = void (*)(nvid_ctx *);
# 定义函数指针类型 rxHash_t，指向返回 bool 类型的函数，参数为 nvid_ctx *, uint32_t, uint64_t, uint32_t *, uint32_t *
using rxHash_t                                          = bool (*)(nvid_ctx *, uint32_t, uint64_t, uint32_t *, uint32_t *);
# 定义函数指针类型 rxPrepare_t，指向返回 bool 类型的函数，参数为 nvid_ctx *, const void *, size_t, bool, uint32_t
using rxPrepare_t                                       = bool (*)(nvid_ctx *, const void *, size_t, bool, uint32_t);
# 定义函数指针类型 setJob_t，指向返回 bool 类型的函数，参数为 nvid_ctx *, const void *, size_t, uint32_t
using setJob_t                                          = bool (*)(nvid_ctx *, const void *, size_t, uint32_t);
# 定义函数指针类型 setJob_v2_t，指向返回 bool 类型的函数，参数为 nvid_ctx *, const void *, size_t, const char *
using setJob_v2_t                                       = bool (*)(nvid_ctx *, const void *, size_t, const char *);
# 定义函数指针类型 version_t，指向返回 uint32_t 类型的函数，参数为 Version
using version_t                                         = uint32_t (*)(Version);

# 初始化静态变量 pAlloc，指向 nullptr
static alloc_t pAlloc                                   = nullptr;
# 初始化静态变量 pCnHash，指向 nullptr
static cnHash_t pCnHash                                 = nullptr;
# 初始化静态变量 pDeviceCount，指向 nullptr
static deviceCount_t pDeviceCount                       = nullptr;
# 初始化静态变量 pDeviceInfo，指向 nullptr
static deviceInfo_t pDeviceInfo                         = nullptr;
# 初始化静态变量 pDeviceInfo_v2，指向 nullptr
static deviceInfo_v2_t pDeviceInfo_v2                   = nullptr;
# 初始化静态变量 pDeviceInit，指向 nullptr
static deviceInit_t pDeviceInit                         = nullptr;
# 初始化静态变量 pDeviceInt，指向 nullptr
static deviceInt_t pDeviceInt                           = nullptr;
# 初始化静态变量 pDeviceName，指向 nullptr
static deviceName_t pDeviceName                         = nullptr;
# 初始化静态变量 pDeviceUint，指向 nullptr
static deviceUint_t pDeviceUint                         = nullptr;
// 声明静态指针变量，初始化为nullptr
static deviceUlong_t pDeviceUlong                       = nullptr;
static init_t pInit                                     = nullptr;
static kawPowHash_t pKawPowHash                         = nullptr;
static kawPowPrepare_v2_t pKawPowPrepare_v2             = nullptr;
static kawPowStopHash_t pKawPowStopHash                 = nullptr;
static lastError_t pLastError                           = nullptr;
static pluginVersion_t pPluginVersion                   = nullptr;
static release_t pRelease                               = nullptr;
static rxHash_t pRxHash                                 = nullptr;
static rxPrepare_t pRxPrepare                           = nullptr;
static setJob_t pSetJob                                 = nullptr;
static setJob_v2_t pSetJob_v2                           = nullptr;
static version_t pVersion                               = nullptr;

// 定义宏，用于动态加载库中的函数指针
#define DLSYM(x) if (uv_dlsym(&cudaLib, k##x, reinterpret_cast<void**>(&p##x)) == -1) { throw std::runtime_error(std::string("symbol not found: ") + k##x); }

// 初始化静态成员变量
bool CudaLib::m_initialized = false;
bool CudaLib::m_ready       = false;
String CudaLib::m_error;
String CudaLib::m_loader;

// xmrig 命名空间
} // namespace xmrig

// 初始化函数，用于加载 CUDA 库
bool xmrig::CudaLib::init(const char *fileName)
{
    // 如果未初始化，则进行初始化
    if (!m_initialized) {
        m_initialized = true;
        // 设置加载器文件名
        m_loader      = fileName == nullptr ? defaultLoader : Env::expand(fileName);

        // 打开 CUDA 库
        if (!open()) {
            return false;
        }

        // 加载 CUDA 库中的函数
        try {
            load();
        } catch (std::exception &ex) {
            // 捕获异常，设置错误信息
            m_error = (std::string(m_loader) + ": " + ex.what()).c_str();

            return false;
        }

        // 设置为就绪状态
        m_ready = true;
    }

    return m_ready;
}

// 获取最后的错误信息
const char *xmrig::CudaLib::lastError() noexcept
{
    return m_error;
}

// 关闭 CUDA 库
void xmrig::CudaLib::close()
{
    uv_dlclose(&cudaLib);
}

// 计算哈希值
bool xmrig::CudaLib::cnHash(nvid_ctx *ctx, uint32_t startNonce, uint64_t height, uint64_t target, uint32_t *rescount, uint32_t *resnonce)
    # 返回调用 pCnHash 函数的结果
    return pCnHash(ctx, startNonce, height, target, rescount, resnonce);
# 获取设备信息，返回布尔值
bool xmrig::CudaLib::deviceInfo(nvid_ctx *ctx, int32_t blocks, int32_t threads, const Algorithm &algorithm, int32_t dataset_host) noexcept
{
    # 将算法转换为对应的算法对象
    const Algorithm algo = RxAlgo::id(algorithm);
    
    # 如果存在设备信息函数，则调用该函数并返回结果
    if (pDeviceInfo) {
        return pDeviceInfo(ctx, blocks, threads, algo, dataset_host);
    }
    
    # 否则调用另一个设备信息函数并返回结果
    return pDeviceInfo_v2(ctx, blocks, threads, algo.isValid() ? algo.name() : nullptr, dataset_host);
}

# 初始化设备，返回布尔值
bool xmrig::CudaLib::deviceInit(nvid_ctx *ctx) noexcept
{
    # 调用设备初始化函数并返回结果
    return pDeviceInit(ctx);
}

# 计算 RxHash，返回布尔值
bool xmrig::CudaLib::rxHash(nvid_ctx *ctx, uint32_t startNonce, uint64_t target, uint32_t *rescount, uint32_t *resnonce) noexcept
{
    # 调用 RxHash 函数并返回结果
    return pRxHash(ctx, startNonce, target, rescount, resnonce);
}

# 准备 RxHash 所需的数据，返回布尔值
bool xmrig::CudaLib::rxPrepare(nvid_ctx *ctx, const void *dataset, size_t datasetSize, bool dataset_host, uint32_t batchSize) noexcept
{
    # 调用 RxPrepare 函数并返回结果
    return pRxPrepare(ctx, dataset, datasetSize, dataset_host, batchSize);
}

# 计算 KawPowHash，返回布尔值
bool xmrig::CudaLib::kawPowHash(nvid_ctx *ctx, uint8_t* job_blob, uint64_t target, uint32_t *rescount, uint32_t *resnonce, uint32_t *skipped_hashes) noexcept
{
    # 调用 KawPowHash 函数并返回结果
    return pKawPowHash(ctx, job_blob, target, rescount, resnonce, skipped_hashes);
}

# 准备 KawPowHash 所需的数据，返回布尔值
bool xmrig::CudaLib::kawPowPrepare(nvid_ctx *ctx, const void* cache, size_t cache_size, const void* dag_precalc, size_t dag_size, uint32_t height, const uint64_t* dag_sizes) noexcept
{
    # 调用 KawPowPrepare_v2 函数并返回结果
    return pKawPowPrepare_v2(ctx, cache, cache_size, dag_precalc, dag_size, height, dag_sizes);
}

# 停止 KawPowHash 计算，返回布尔值
bool xmrig::CudaLib::kawPowStopHash(nvid_ctx *ctx) noexcept
{
    # 调用 KawPowStopHash 函数并返回结果
    return pKawPowStopHash(ctx);
}

# 设置作业信息，返回布尔值
bool xmrig::CudaLib::setJob(nvid_ctx *ctx, const void *data, size_t size, const Algorithm &algorithm) noexcept
{
    # 将算法转换为对应的算法对象
    const Algorithm algo = RxAlgo::id(algorithm);
    
    # 如果存在设置作业函数，则调用该函数并返回结果
    if (pSetJob) {
        return pSetJob(ctx, data, size, algo);
    }
    
    # 否则调用另一个设置作业函数并返回结果
    return pSetJob_v2(ctx, data, size, algo.name());
}

# 获取设备名称，返回字符指针
const char *xmrig::CudaLib::deviceName(nvid_ctx *ctx) noexcept
{
    # 调用获取设备名称函数并返回结果
    return pDeviceName(ctx);
}

# 获取最后的错误信息，返回字符指针
const char *xmrig::CudaLib::lastError(nvid_ctx *ctx) noexcept
{
    # 返回获取最后错误信息函数的结果
    # 返回上下文中的最后一个错误
    return pLastError(ctx);
}

// 获取 CUDA 插件版本号
const char *xmrig::CudaLib::pluginVersion() noexcept
{
    return pPluginVersion();
}

// 获取设备整型属性
int32_t xmrig::CudaLib::deviceInt(nvid_ctx *ctx, DeviceProperty property) noexcept
{
    return pDeviceInt(ctx, property);
}

// 分配 CUDA 设备上下文
nvid_ctx *xmrig::CudaLib::alloc(uint32_t id, int32_t bfactor, int32_t bsleep) noexcept
{
    return pAlloc(id, bfactor, bsleep);
}

// 获取 CUDA 版本号
std::string xmrig::CudaLib::version(uint32_t version)
{
    return std::to_string(version / 1000) + "." + std::to_string((version % 1000) / 10);
}

// 获取 CUDA 设备列表
std::vector<xmrig::CudaDevice> xmrig::CudaLib::devices(int32_t bfactor, int32_t bsleep, const std::vector<uint32_t> &hints) noexcept
{
    // 获取设备数量
    const uint32_t count = deviceCount();
    if (!count) {
        return {};
    }

    std::vector<CudaDevice> out;
    out.reserve(count);

    // 如果提示列表为空，则遍历所有设备
    if (hints.empty()) {
        for (uint32_t i = 0; i < count; ++i) {
            CudaDevice device(i, bfactor, bsleep);
            if (device.isValid()) {
                out.emplace_back(std::move(device));
            }
        }
    }
    // 否则，根据提示列表遍历设备
    else {
        for (const uint32_t i : hints) {
            if (i >= count) {
                continue;
            }

            CudaDevice device(i, bfactor, bsleep);
            if (device.isValid()) {
                out.emplace_back(std::move(device));
            }
        }
    }

    return out;
}

// 获取 CUDA 设备数量
uint32_t xmrig::CudaLib::deviceCount() noexcept
{
    return pDeviceCount();
}

// 获取 CUDA 设备无符号整型属性
uint32_t xmrig::CudaLib::deviceUint(nvid_ctx *ctx, DeviceProperty property) noexcept
{
    return pDeviceUint(ctx, property);
}

// 获取 CUDA 驱动版本号
uint32_t xmrig::CudaLib::driverVersion() noexcept
{
    return pVersion(DriverVersion);
}

// 获取 CUDA 运行时版本号
uint32_t xmrig::CudaLib::runtimeVersion() noexcept
{
    return pVersion(RuntimeVersion);
}

// 获取 CUDA 设备无符号长整型属性
uint64_t xmrig::CudaLib::deviceUlong(nvid_ctx *ctx, DeviceProperty property) noexcept
{
    return pDeviceUlong(ctx, property);
}

// 释放 CUDA 设备上下文
void xmrig::CudaLib::release(nvid_ctx *ctx) noexcept
{
    pRelease(ctx);
}

// 打开 CUDA
bool xmrig::CudaLib::open()
{
    m_error = nullptr;
    # 如果使用uv_dlopen函数加载cudaLib成功，则返回true
    if (uv_dlopen(m_loader, &cudaLib) == 0) {
        return true;
    }
// 如果操作系统是 Linux
#ifdef XMRIG_OS_LINUX
    // 如果加载器是默认加载器
    if (m_loader == defaultLoader) {
        // 获取当前进程的可执行文件位置，并将其赋值给加载器
        m_loader = Process::location(Process::ExeLocation, m_loader);
        // 使用动态链接库加载器加载 CUDA 库，如果成功则返回 true
        if (uv_dlopen(m_loader, &cudaLib) == 0) {
            return true;
        }
    }
#endif

    // 获取 CUDA 库加载错误信息
    m_error = uv_dlerror(&cudaLib);

    // 返回 false
    return false;
}

// 加载 CUDA 库的函数
void xmrig::CudaLib::load()
{
    // 获取 CUDA 库的版本信息
    DLSYM(Version);

    // 获取 CUDA 库的 API 版本，并进行版本检查
    const uint32_t api = pVersion(ApiVersion);
    if (api < 3U || api > 4U) {
        throw std::runtime_error("API version mismatch");
    }

    // 获取 CUDA 库的各个函数指针
    DLSYM(Alloc);
    DLSYM(CnHash);
    DLSYM(DeviceCount);
    DLSYM(DeviceInit);
    DLSYM(DeviceInt);
    DLSYM(DeviceName);
    DLSYM(DeviceUint);
    DLSYM(DeviceUlong);
    DLSYM(Init);
    DLSYM(LastError);
    DLSYM(PluginVersion);
    DLSYM(Release);
    DLSYM(RxHash);
    DLSYM(RxPrepare);
    DLSYM(KawPowHash);
    DLSYM(KawPowPrepare_v2);
    DLSYM(KawPowStopHash);

    // 根据 API 版本获取对应的函数指针
    if (api == 4U) {
        DLSYM(DeviceInfo);
        DLSYM(SetJob);
    }
    else if (api == 3U) {
        DLSYM(DeviceInfo_v2);
        DLSYM(SetJob_v2);
    }

    // 调用 CUDA 库的初始化函数
    pInit();
}
```