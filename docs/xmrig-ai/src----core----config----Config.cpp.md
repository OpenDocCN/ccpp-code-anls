# `xmrig\src\core\config\Config.cpp`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据GNU通用公共许可证的条款发布，由
 * 自由软件基金会发布的版本3或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <algorithm>
#include <cinttypes>
#include <cstring>
#include <uv.h>


#include "core/config/Config.h"
#include "3rdparty/rapidjson/document.h"
#include "backend/cpu/Cpu.h"
#include "base/io/log/Log.h"
#include "base/kernel/interfaces/IJsonReader.h"
#include "base/net/dns/Dns.h"
#include "crypto/common/Assembly.h"


#ifdef XMRIG_ALGO_RANDOMX
#   include "crypto/rx/RxConfig.h"
#endif


#ifdef XMRIG_FEATURE_OPENCL
#   include "backend/opencl/OclConfig.h"
#endif


#ifdef XMRIG_FEATURE_CUDA
#   include "backend/cuda/CudaConfig.h"
#endif


namespace xmrig {


constexpr static uint32_t kIdleTime     = 60U;


const char *Config::kPauseOnBattery     = "pause-on-battery";
const char *Config::kPauseOnActive      = "pause-on-active";


#ifdef XMRIG_FEATURE_OPENCL
const char *Config::kOcl                = "opencl";
#endif

#ifdef XMRIG_FEATURE_CUDA
const char *Config::kCuda               = "cuda";
#endif

#if defined(XMRIG_FEATURE_NVML) || defined (XMRIG_FEATURE_ADL)
const char *Config::kHealthPrintTime    = "health-print-time";
#endif

#ifdef XMRIG_FEATURE_DMI
const char *Config::kDMI                = "dmi";
#endif


class ConfigPrivate
{
public:
    # 定义一个布尔变量，表示是否在电池模式下暂停
    bool pauseOnBattery = false;
    # 定义一个CpuConfig类型的变量cpu，用于存储CPU配置信息
    CpuConfig cpu;
    # 定义一个32位的无符号整数变量idleTime，用于存储空闲时间
    uint32_t idleTime   = 0;
// 如果定义了 XMRIG_ALGO_RANDOMX，则创建 RxConfig 对象
    RxConfig rx;
// 如果定义了 XMRIG_FEATURE_OPENCL，则创建 OclConfig 对象
    OclConfig cl;
// 如果定义了 XMRIG_FEATURE_CUDA，则创建 CudaConfig 对象
    CudaConfig cuda;
// 如果定义了 XMRIG_FEATURE_NVML 或 XMRIG_FEATURE_ADL，则设置健康打印时间为60秒
    uint32_t healthPrintTime = 60U;
// 如果定义了 XMRIG_FEATURE_DMI，则设置 dmi 为 true
    bool dmi = true;

// 设置空闲时间，根据传入的值进行判断
    void setIdleTime(const rapidjson::Value &value)
    {
        if (value.IsBool()) {
            idleTime = value.GetBool() ? kIdleTime : 0U;
        }
        else if (value.IsUint()) {
            idleTime = value.GetUint();
        }
    }
};

} // namespace xmrig

// Config 构造函数
xmrig::Config::Config() :
    d_ptr(new ConfigPrivate())
{
}

// Config 析构函数
xmrig::Config::~Config()
{
    delete d_ptr;
}

// 返回是否在电池模式下暂停
bool xmrig::Config::isPauseOnBattery() const
{
    return d_ptr->pauseOnBattery;
}

// 返回 CPU 配置
const xmrig::CpuConfig &xmrig::Config::cpu() const
{
    return d_ptr->cpu;
}

// 返回空闲时间
uint32_t xmrig::Config::idleTime() const
{
    return d_ptr->idleTime * 1000U;
}

// 如果定义了 XMRIG_FEATURE_OPENCL，则返回 OclConfig 对象
const xmrig::OclConfig &xmrig::Config::cl() const
{
    return d_ptr->cl;
}

// 如果定义了 XMRIG_FEATURE_CUDA，则返回 CudaConfig 对象
const xmrig::CudaConfig &xmrig::Config::cuda() const
{
    return d_ptr->cuda;
}

// 如果定义了 XMRIG_ALGO_RANDOMX，则返回 RxConfig 对象
const xmrig::RxConfig &xmrig::Config::rx() const
{
    return d_ptr->rx;
}

// 如果定义了 XMRIG_FEATURE_NVML 或 XMRIG_FEATURE_ADL，则返回健康打印时间
uint32_t xmrig::Config::healthPrintTime() const
{
    return d_ptr->healthPrintTime;
}

// 如果定义了 XMRIG_FEATURE_DMI，则返回是否启用 DMI
bool xmrig::Config::isDMI() const
{
    return d_ptr->dmi;
}

// 返回是否应该保存配置
bool xmrig::Config::isShouldSave() const
{
    if (!isAutoSave()) {
        return false;
    }

// 如果定义了 XMRIG_FEATURE_OPENCL，并且 OclConfig 对象需要保存，则返回 true
    if (cl().isShouldSave()) {
        return true;
    }

// 如果定义了 XMRIG_FEATURE_CUDA，并且 CudaConfig 对象需要保存，则返回 true
    if (cuda().isShouldSave()) {
        return true;
    }

// 如果需要升级或者 CPU 配置需要保存，则返回 true
    return (m_upgrade || cpu().isShouldSave());
}

// 从 JSON 读取配置
bool xmrig::Config::read(const IJsonReader &reader, const char *fileName)
{
    // 如果 BaseConfig::read() 返回 false，则返回 false
    if (!BaseConfig::read(reader, fileName)) {
        return false;
    }

    // 从配置文件中读取布尔值，并赋给 d_ptr->pauseOnBattery
    d_ptr->pauseOnBattery = reader.getBool(kPauseOnBattery, d_ptr->pauseOnBattery);
    // 从配置文件中读取值，并赋给 d_ptr->setIdleTime
    d_ptr->setIdleTime(reader.getValue(kPauseOnActive));

    // 如果定义了 XMRIG_ALGO_RANDOMX，则执行以下代码块
#   ifdef XMRIG_ALGO_RANDOMX
    // 如果 !d_ptr->rx.read() 返回 false，则将 m_upgrade 设置为 true
    if (!d_ptr->rx.read(reader.getValue(RxConfig::kField))) {
        m_upgrade = true;
    }
#   endif

    // 如果定义了 XMRIG_FEATURE_OPENCL，并且 pools().isBenchmark() 返回 false，则执行以下代码块
#   ifdef XMRIG_FEATURE_OPENCL
    if (!pools().isBenchmark()) {
        // 从配置文件中读取值，并赋给 d_ptr->cl
        d_ptr->cl.read(reader.getValue(kOcl));
    }
#   endif

    // 如果定义了 XMRIG_FEATURE_CUDA，并且 pools().isBenchmark() 返回 false，则执行以下代码块
#   ifdef XMRIG_FEATURE_CUDA
    if (!pools().isBenchmark()) {
        // 从配置文件中读取值，并赋给 d_ptr->cuda
        d_ptr->cuda.read(reader.getValue(kCuda));
    }
#   endif

    // 如果定义了 XMRIG_FEATURE_NVML 或 XMRIG_FEATURE_ADL，则从配置文件中读取值，并赋给 d_ptr->healthPrintTime
#   if defined(XMRIG_FEATURE_NVML) || defined (XMRIG_FEATURE_ADL)
    d_ptr->healthPrintTime = reader.getUint(kHealthPrintTime, d_ptr->healthPrintTime);
#   endif

    // 如果定义了 XMRIG_FEATURE_DMI，则从配置文件中读取布尔值，并赋给 d_ptr->dmi
#   ifdef XMRIG_FEATURE_DMI
    d_ptr->dmi = reader.getBool(kDMI, d_ptr->dmi);
#   endif

    // 返回 true
    return true;
}


// 将配置信息转换为 JSON 格式
void xmrig::Config::getJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;

    // 设置 doc 为一个 JSON 对象
    doc.SetObject();

    // 获取 doc 的分配器
    auto &allocator = doc.GetAllocator();

    // 创建一个名为 api 的 JSON 对象，并添加成员
    Value api(kObjectType);
    api.AddMember(StringRef(kApiId),                    m_apiId.toJSON(), allocator);
    api.AddMember(StringRef(kApiWorkerId),              m_apiWorkerId.toJSON(), allocator);

    // 将 api 添加为 doc 的成员
    doc.AddMember(StringRef(kApi),                      api, allocator);
    // 将 m_http 转换为 JSON 格式，并添加为 doc 的成员
    doc.AddMember(StringRef(kHttp),                     m_http.toJSON(doc), allocator);
    // 添加 isAutoSave() 的布尔值为 doc 的成员
    doc.AddMember(StringRef(kAutosave),                 isAutoSave(), allocator);
    // 添加 isBackground() 的布尔值为 doc 的成员
    doc.AddMember(StringRef(kBackground),               isBackground(), allocator);
    // 添加 Log::isColors() 的布尔值为 doc 的成员
    doc.AddMember(StringRef(kColors),                   Log::isColors(), allocator);
    // 将 title().toJSON() 转换为 JSON 格式，并添加为 doc 的成员
    doc.AddMember(StringRef(kTitle),                    title().toJSON(), allocator);

    // 如果定义了 XMRIG_ALGO_RANDOMX，则将 rx().toJSON() 转换为 JSON 格式，并添加为 doc 的成员
#   ifdef XMRIG_ALGO_RANDOMX
    doc.AddMember(StringRef(RxConfig::kField),          rx().toJSON(doc), allocator);
#   endif
}
    # 向文档中添加成员，成员名为 CpuConfig::kField，值为 cpu().toJSON(doc) 的结果，分配内存给 allocator
    doc.AddMember(StringRef(CpuConfig::kField), cpu().toJSON(doc), allocator);
# 如果定义了 XMRIG_FEATURE_OPENCL，则将 OpenCL 相关信息添加到 JSON 文档中
    doc.AddMember(StringRef(kOcl),                      cl().toJSON(doc), allocator);
# 如果定义了 XMRIG_FEATURE_CUDA，则将 CUDA 相关信息添加到 JSON 文档中
    doc.AddMember(StringRef(kCuda),                     cuda().toJSON(doc), allocator);
# 将日志文件信息添加到 JSON 文档中
    doc.AddMember(StringRef(kLogFile),                  m_logFile.toJSON(), allocator);
# 将矿池信息添加到 JSON 文档中
    m_pools.toJSON(doc, doc);
# 将打印时间信息添加到 JSON 文档中
    doc.AddMember(StringRef(kPrintTime),                printTime(), allocator);
# 如果定义了 XMRIG_FEATURE_NVML 或 XMRIG_FEATURE_ADL，则将健康打印时间信息添加到 JSON 文档中
    doc.AddMember(StringRef(kHealthPrintTime),          healthPrintTime(), allocator);
# 如果定义了 XMRIG_FEATURE_DMI，则将 DMI 信息添加到 JSON 文档中
    doc.AddMember(StringRef(kDMI),                      isDMI(), allocator);
# 将系统日志信息添加到 JSON 文档中
    doc.AddMember(StringRef(kSyslog),                   isSyslog(), allocator);
# 如果定义了 XMRIG_FEATURE_TLS，则将 TLS 信息添加到 JSON 文档中
    doc.AddMember(StringRef(kTls),                      m_tls.toJSON(doc), allocator);
# 将 DNS 配置信息添加到 JSON 文档中
    doc.AddMember(StringRef(DnsConfig::kField),         Dns::config().toJSON(doc), allocator);
# 将用户代理信息添加到 JSON 文档中
    doc.AddMember(StringRef(kUserAgent),                m_userAgent.toJSON(), allocator);
# 将详细信息添加到 JSON 文档中
    doc.AddMember(StringRef(kVerbose),                  Log::verbose(), allocator);
# 将监视信息添加到 JSON 文档中
    doc.AddMember(StringRef(kWatch),                    m_watch, allocator);
# 将电池暂停信息添加到 JSON 文档中
    doc.AddMember(StringRef(kPauseOnBattery),           isPauseOnBattery(), allocator);
# 将活动暂停信息添加到 JSON 文档中
    doc.AddMember(StringRef(kPauseOnActive),            (d_ptr->idleTime == 0U || d_ptr->idleTime == kIdleTime) ? Value(isPauseOnActive()) : Value(d_ptr->idleTime), allocator);
```