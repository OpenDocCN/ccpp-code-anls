# `xmrig\src\base\kernel\config\BaseConfig.cpp`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新分发或修改
 * 根据GNU通用公共许可证的条款发布
 * 由自由软件基金会发布，许可证的第3版或
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


#include "base/kernel/config/BaseConfig.h"  // 导入BaseConfig类的头文件
#include "3rdparty/rapidjson/document.h"    // 导入rapidjson库的document头文件
#include "base/io/json/Json.h"              // 导入Json类的头文件
#include "base/io/log/Log.h"                // 导入Log类的头文件
#include "base/io/log/Tags.h"               // 导入Tags类的头文件
#include "base/kernel/interfaces/IJsonReader.h"  // 导入IJsonReader接口的头文件
#include "base/net/dns/Dns.h"               // 导入Dns类的头文件
#include "version.h"                        // 导入version头文件


#include <algorithm>                       // 导入算法库
#include <cassert>                         // 导入断言库
#include <cstdio>                          // 导入C标准输入输出库
#include <cstdlib>                         // 导入C标准库
#include <cstring>                         // 导入C字符串库
#include <uv.h>                            // 导入libuv库


#ifdef XMRIG_FEATURE_TLS
#   include <openssl/opensslv.h>           // 如果启用了TLS功能，则导入OpenSSL版本头文件
#endif

#ifdef XMRIG_FEATURE_HWLOC
#   include "backend/cpu/Cpu.h"            // 如果启用了HWLOC功能，则导入Cpu类的头文件
#endif


namespace xmrig {


const char *BaseConfig::kApi            = "api";           // 定义BaseConfig类的静态成员变量kApi
const char *BaseConfig::kApiId          = "id";            // 定义BaseConfig类的静态成员变量kApiId
const char *BaseConfig::kApiWorkerId    = "worker-id";     // 定义BaseConfig类的静态成员变量kApiWorkerId
const char *BaseConfig::kAutosave       = "autosave";      // 定义BaseConfig类的静态成员变量kAutosave
const char *BaseConfig::kBackground     = "background";    // 定义BaseConfig类的静态成员变量kBackground
const char *BaseConfig::kColors         = "colors";        // 定义BaseConfig类的静态成员变量kColors
const char *BaseConfig::kDryRun         = "dry-run";       // 定义BaseConfig类的静态成员变量kDryRun
const char *BaseConfig::kHttp           = "http";          // 定义BaseConfig类的静态成员变量kHttp
const char *BaseConfig::kLogFile        = "log-file";      // 定义BaseConfig类的静态成员变量kLogFile
const char *BaseConfig::kPrintTime      = "print-time";    // 定义BaseConfig类的静态成员变量kPrintTime
const char *BaseConfig::kSyslog         = "syslog";        // 定义BaseConfig类的静态成员变量kSyslog
// 定义静态常量字符串，表示配置文件中的标题、用户代理、详细信息和监视选项
const char *BaseConfig::kTitle          = "title";
const char *BaseConfig::kUserAgent      = "user-agent";
const char *BaseConfig::kVerbose        = "verbose";
const char *BaseConfig::kWatch          = "watch";

#ifdef XMRIG_FEATURE_TLS
// 如果启用了 TLS 功能，则定义 TLS 常量字符串
const char *BaseConfig::kTls            = "tls";
#endif

// 命名空间 xmrig
} // namespace xmrig

// 从 JSON 读取配置信息
bool xmrig::BaseConfig::read(const IJsonReader &reader, const char *fileName)
{
    // 将文件名存储到成员变量中
    m_fileName = fileName;

    // 如果 JSON 为空，则返回 false
    if (reader.isEmpty()) {
        return false;
    }

    // 从 JSON 中读取各项配置信息，并存储到对应的成员变量中
    m_autoSave          = reader.getBool(kAutosave, m_autoSave);
    m_background        = reader.getBool(kBackground, m_background);
    m_dryRun            = reader.getBool(kDryRun, m_dryRun);
    m_syslog            = reader.getBool(kSyslog, m_syslog);
    m_watch             = reader.getBool(kWatch, m_watch);
    m_logFile           = reader.getString(kLogFile);
    m_userAgent         = reader.getString(kUserAgent);
    m_printTime         = std::min(reader.getUint(kPrintTime, m_printTime), 3600U);
    m_title             = reader.getValue(kTitle);

#   ifdef XMRIG_FEATURE_TLS
    // 如果启用了 TLS 功能，则从 JSON 中读取 TLS 配置信息
    m_tls = reader.getValue(kTls);
#   endif

    // 设置日志颜色
    Log::setColors(reader.getBool(kColors, Log::isColors()));
    // 设置详细信息
    setVerbose(reader.getValue(kVerbose));

    // 从 JSON 中读取 API 配置信息
    const auto &api = reader.getObject(kApi);
    if (api.IsObject()) {
        m_apiId       = Json::getString(api, kApiId);
        m_apiWorkerId = Json::getString(api, kApiWorkerId);
    }

    // 从 JSON 中加载 HTTP 配置信息
    m_http.load(reader.getObject(kHttp));
    // 从 JSON 中加载矿池配置信息
    m_pools.load(reader);
    // 设置 DNS 配置
    Dns::set(reader.getObject(DnsConfig::kField));

    // 返回是否成功加载了至少一个矿池的配置信息
    return m_pools.active() > 0;
}

// 保存配置信息到文件
bool xmrig::BaseConfig::save()
{
    // 如果文件名为空，则返回 false
    if (m_fileName.isNull()) {
        return false;
    }

    // 创建 JSON 文档并将配置信息存储到其中
    rapidjson::Document doc;
    getJSON(doc);

    // 将 JSON 文档保存到文件中
    if (Json::save(m_fileName, doc)) {
        // 输出日志，表示配置信息已保存到文件中
        LOG_NOTICE("%s " WHITE_BOLD("configuration saved to: \"%s\""), Tags::config(), m_fileName.data());
        return true;
    }

    return false;
}

// 打印版本信息
void xmrig::BaseConfig::printVersions()
{
    // 创建一个缓冲区
    char buf[256] = { 0 };
}
#   if defined(__clang__)
    # 如果是使用 clang 编译器
    snprintf(buf, sizeof buf, "clang/%d.%d.%d", __clang_major__, __clang_minor__, __clang_patchlevel__);
#   elif defined(__GNUC__)
    # 如果是使用 GCC 编译器
    snprintf(buf, sizeof buf, "gcc/%d.%d.%d", __GNUC__, __GNUC_MINOR__, __GNUC_PATCHLEVEL__);
#   elif defined(_MSC_VER)
    # 如果是使用 MSVC 编译器
    snprintf(buf, sizeof buf, "MSVC/%d", MSVC_VERSION);
#   endif

    # 打印关于应用程序的信息，包括应用程序名称、版本、编译器信息、操作系统、架构和位数
    Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-13s") CYAN_BOLD("%s/%s") WHITE_BOLD(" %s") WHITE_BOLD(" (built for %s") WHITE_BOLD(" %s,") WHITE_BOLD(" %s)"), "ABOUT", APP_NAME, APP_VERSION, buf, APP_OS, APP_ARCH, APP_BITS);

    std::string libs;

#   if defined(XMRIG_FEATURE_TLS)
    {
#       if defined(LIBRESSL_VERSION_TEXT)
        # 如果使用 LibreSSL
        snprintf(buf, sizeof buf, "LibreSSL/%s ", LIBRESSL_VERSION_TEXT + 9);
        libs += buf;
#       elif defined(OPENSSL_VERSION_TEXT)
        # 如果使用 OpenSSL
        constexpr const char *v = &OPENSSL_VERSION_TEXT[8];
        snprintf(buf, sizeof buf, "OpenSSL/%.*s ", static_cast<int>(strchr(v, ' ') - v), v);
        libs += buf;
#       endif
    }
#   endif

#   if defined(XMRIG_FEATURE_HWLOC)
    # 如果使用了硬件定位库
    libs += Cpu::info()->backend();
#   endif

    # 打印关于使用的库的信息，包括 libuv 版本和其他库的信息
    Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-13slibuv/%s %s"), "LIBS", uv_version_string(), libs.c_str());
}


void xmrig::BaseConfig::setVerbose(const rapidjson::Value &value)
{
    # 设置日志的详细程度
    if (value.IsBool()) {
        Log::setVerbose(value.GetBool() ? 1 : 0);
    }
    else if (value.IsUint()) {
        Log::setVerbose(value.GetUint());
    }
}
```