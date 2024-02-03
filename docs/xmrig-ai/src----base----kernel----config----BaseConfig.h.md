# `xmrig\src\base\kernel\config\BaseConfig.h`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新分发或修改
 * 根据GNU通用公共许可证的条款发布，由
 * 自由软件基金会发布的许可证的第3版，或
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

#ifndef XMRIG_BASECONFIG_H
#define XMRIG_BASECONFIG_H


#include "base/kernel/config/Title.h"
#include "base/kernel/interfaces/IConfig.h"
#include "base/net/http/Http.h"
#include "base/net/stratum/Pools.h"


#ifdef XMRIG_FEATURE_TLS
#   include "base/net/tls/TlsConfig.h"
#endif


namespace xmrig {


class IJsonReader;


class BaseConfig : public IConfig
{
public:
    static const char *kApi;
    static const char *kApiId;
    static const char *kApiWorkerId;
    static const char *kAutosave;
    static const char *kBackground;
    static const char *kColors;
    static const char *kDryRun;
    static const char *kHttp;
    static const char *kLogFile;
    static const char *kPrintTime;
    static const char *kSyslog;
    static const char *kTitle;
    static const char *kUserAgent;
    static const char *kVerbose;
    static const char *kWatch;

#   ifdef XMRIG_FEATURE_TLS
    static const char *kTls;
#   endif

    BaseConfig() = default;

    // 返回是否自动保存配置
    inline bool isAutoSave() const                          { return m_autoSave; }
    // 返回是否后台运行
    inline bool isBackground() const                        { return m_background; }
    # 返回成员变量 m_dryRun 的值，表示是否为干运行模式
    inline bool isDryRun() const                            { return m_dryRun; }
    # 返回成员变量 m_syslog 的值，表示是否启用系统日志
    inline bool isSyslog() const                            { return m_syslog; }
    # 返回成员变量 m_logFile 的值，表示日志文件名
    inline const char *logFile() const                      { return m_logFile.data(); }
    # 返回成员变量 m_userAgent 的值，表示用户代理信息
    inline const char *userAgent() const                    { return m_userAgent.data(); }
    # 返回成员变量 m_http 的值，表示 HTTP 对象
    inline const Http &http() const                         { return m_http; }
    # 返回成员变量 m_pools 的值，表示连接池对象
    inline const Pools &pools() const                       { return m_pools; }
    # 返回成员变量 m_apiId 的值，表示 API ID
    inline const String &apiId() const                      { return m_apiId; }
    # 返回成员变量 m_apiWorkerId 的值，表示 API 工作 ID
    inline const String &apiWorkerId() const                { return m_apiWorkerId; }
    # 返回成员变量 m_title 的值，表示标题对象
    inline const Title &title() const                       { return m_title; }
    # 返回成员变量 m_printTime 的值，表示打印时间
    inline uint32_t printTime() const                       { return m_printTime; }
// 如果定义了 XMRIG_FEATURE_TLS，则返回 TLS 配置
#ifdef XMRIG_FEATURE_TLS
    inline const TlsConfig &tls() const                     { return m_tls; }
#endif

// 返回是否为监视模式
inline bool isWatch() const override                    { return m_watch && !m_fileName.isNull(); }
// 返回文件名
inline const String &fileName() const override          { return m_fileName; }
// 设置文件名
inline void setFileName(const char *fileName) override  { m_fileName = fileName; }

// 从 JSON 读取数据到内存
bool read(const IJsonReader &reader, const char *fileName) override;
// 保存数据到文件
bool save() override;

// 打印版本信息
static void printVersions();

// 以下为保护成员变量
protected:
    bool m_autoSave         = true;
    bool m_background       = false;
    bool m_dryRun           = false;
    bool m_syslog           = false;
    bool m_upgrade          = false;
    bool m_watch            = true;
    Http m_http;
    Pools m_pools;
    String m_apiId;
    String m_apiWorkerId;
    String m_fileName;
    String m_logFile;
    String m_userAgent;
    Title m_title;
    uint32_t m_printTime    = 60;

// 如果定义了 XMRIG_FEATURE_TLS，则包含 TLS 配置
#ifdef XMRIG_FEATURE_TLS
    TlsConfig m_tls;
#endif

// 以下为私有成员变量
private:
    // 设置详细信息
    static void setVerbose(const rapidjson::Value &value);
};


} // namespace xmrig

// 结束 if 预处理指令，结束文件
#endif /* XMRIG_BASECONFIG_H */
```