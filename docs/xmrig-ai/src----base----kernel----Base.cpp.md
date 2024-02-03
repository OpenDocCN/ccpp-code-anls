# `xmrig\src\base\kernel\Base.cpp`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据GNU通用公共许可证的条款发布，由
 * 自由软件基金会发布的许可证的第3版，或者
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

#include <cassert>
#include <memory>


#include "base/kernel/Base.h"
#include "base/io/json/Json.h"
#include "base/io/json/JsonChain.h"
#include "base/io/log/backends/ConsoleLog.h"
#include "base/io/log/backends/FileLog.h"
#include "base/io/log/Log.h"
#include "base/io/log/Tags.h"
#include "base/io/Watcher.h"
#include "base/kernel/interfaces/IBaseListener.h"
#include "base/kernel/Platform.h"
#include "base/kernel/Process.h"
#include "base/net/tools/NetBuffer.h"
#include "core/config/Config.h"
#include "core/config/ConfigTransform.h"
#include "version.h"


#ifdef HAVE_SYSLOG_H
#   include "base/io/log/backends/SysLog.h"
#endif


#ifdef XMRIG_FEATURE_API
#   include "base/api/Api.h"
#   include "base/api/interfaces/IApiRequest.h"

namespace xmrig {

static const char *kConfigPathV1 = "/1/config";
static const char *kConfigPathV2 = "/2/config";

} // namespace xmrig
#endif


#ifdef XMRIG_FEATURE_EMBEDDED_CONFIG
#   include "core/config/Config_default.h"
#endif


namespace xmrig {


class BasePrivate
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(BasePrivate)


    inline explicit BasePrivate(Process *process)
*/
    {
        // 初始化日志系统
        Log::init();

        // 加载进程配置信息
        config = load(process);
    }


    // BasePrivate 类的析构函数
    inline ~BasePrivate()
    {
// 如果定义了 XMRIG_FEATURE_API，则删除 api 对象
#ifdef XMRIG_FEATURE_API
    delete api;
#endif

// 删除 config 对象
delete config;
// 删除 watcher 对象
delete watcher;

// 销毁 NetBuffer 对象
NetBuffer::destroy();
}

// 从 JsonChain 对象中读取配置信息，并存储到 config 对象中
inline static bool read(const JsonChain &chain, std::unique_ptr<Config> &config)
{
    // 创建一个新的 Config 对象
    config = std::unique_ptr<Config>(new Config());

    // 调用 Config 对象的 read 方法，将配置信息存储到 config 对象中
    return config->read(chain, chain.fileName());
}

// 用新的配置替换原有的配置
inline void replace(Config *newConfig)
{
    // 保存原有的配置对象
    Config *previousConfig = config;
    // 使用新的配置对象替换原有的配置对象
    config = newConfig;

    // 遍历 listeners 列表，调用 onConfigChanged 方法通知配置变更
    for (IBaseListener *listener : listeners) {
        listener->onConfigChanged(config, previousConfig);
    }

    // 删除原有的配置对象
    delete previousConfig;
}

// 初始化 api、config、listeners、watcher 对象
Api *api            = nullptr;
Config *config      = nullptr;
std::vector<IBaseListener *> listeners;
Watcher *watcher    = nullptr;

// 静态方法，从进程中加载配置信息
inline static Config *load(Process *process)
{
    // 创建 JsonChain 和 ConfigTransform 对象
    JsonChain chain;
    ConfigTransform transform;
    std::unique_ptr<Config> config;

    // 从进程中加载配置信息
    ConfigTransform::load(chain, process, transform);

    // 调用 read 方法，将配置信息存储到 config 对象中
    if (read(chain, config)) {
        return config.release();
    }

    // 添加默认配置文件路径，并读取配置信息
    chain.addFile(Process::location(Process::DataLocation, "config.json"));
    if (read(chain, config)) {
        return config.release();
    }

    // 添加默认配置文件路径，并读取配置信息
    chain.addFile(Process::location(Process::HomeLocation,  "." APP_ID ".json"));
    if (read(chain, config)) {
        return config.release();
    }

    // 添加默认配置文件路径，并读取配置信息
    chain.addFile(Process::location(Process::HomeLocation, ".config" XMRIG_DIR_SEPARATOR APP_ID ".json"));
    if (read(chain, config)) {
        return config.release();
    }

    // 如果定义了 XMRIG_FEATURE_EMBEDDED_CONFIG，则添加默认配置信息，并读取配置
#ifdef XMRIG_FEATURE_EMBEDDED_CONFIG
    chain.addRaw(default_config);

    if (read(chain, config)) {
        return config.release();
    }
#endif

    return nullptr;
}
};

} // namespace xmrig

// 构造函数，初始化 BasePrivate 对象
xmrig::Base::Base(Process *process)
    : d_ptr(new BasePrivate(process))
{
}

// 析构函数，删除 BasePrivate 对象
xmrig::Base::~Base()
{
    delete d_ptr;
}
# 检查是否配置指针不为空，返回布尔值
bool xmrig::Base::isReady() const
{
    return d_ptr->config != nullptr;
}


# 初始化函数
int xmrig::Base::init()
{
#   ifdef XMRIG_FEATURE_API
    # 如果定义了 XMRIG_FEATURE_API，则创建 Api 对象并添加监听器
    d_ptr->api = new Api(this);
    d_ptr->api->addListener(this);
#   endif

    # 调用 Platform 类的 init 函数，传入配置的用户代理
    Platform::init(config()->userAgent());

    # 如果是后台运行，则设置日志为后台模式
    if (isBackground()) {
        Log::setBackground(true);
    }
    # 否则，添加一个控制台日志
    else {
        Log::add(new ConsoleLog(config()->title()));
    }

    # 如果配置了日志文件，则添加一个文件日志
    if (config()->logFile()) {
        Log::add(new FileLog(config()->logFile()));
    }

#   ifdef HAVE_SYSLOG_H
    # 如果定义了 HAVE_SYSLOG_H 并且配置了使用系统日志，则添加一个系统日志
    if (config()->isSyslog()) {
        Log::add(new SysLog());
    }
#   endif

    return 0;
}


# 启动函数
void xmrig::Base::start()
{
#   ifdef XMRIG_FEATURE_API
    # 如果定义了 XMRIG_FEATURE_API，则启动 API
    api()->start();
#   endif

    # 如果配置了应该保存，则保存配置
    if (config()->isShouldSave()) {
        config()->save();
    }

    # 如果配置了监视文件变化，则创建一个 Watcher 对象
    if (config()->isWatch()) {
        d_ptr->watcher = new Watcher(config()->fileName(), this);
    }
}


# 停止函数
void xmrig::Base::stop()
{
#   ifdef XMRIG_FEATURE_API
    # 如果定义了 XMRIG_FEATURE_API，则停止 API
    api()->stop();
#   endif

    # 删除 Watcher 对象
    delete d_ptr->watcher;
    d_ptr->watcher = nullptr;
}


# 返回 Api 指针
xmrig::Api *xmrig::Base::api() const
{
    assert(d_ptr->api != nullptr);

    return d_ptr->api;
}


# 检查是否后台运行
bool xmrig::Base::isBackground() const
{
    return d_ptr->config && d_ptr->config->isBackground();
}


# 重新加载配置
bool xmrig::Base::reload(const rapidjson::Value &json)
{
    # 从 JSON 值创建 JsonReader 对象
    JsonReader reader(json);
    # 如果读取为空，则返回 false
    if (reader.isEmpty()) {
        return false;
    }

    # 创建一个新的配置对象，并从 JsonReader 中读取配置
    auto config = new Config();
    if (!config->read(reader, d_ptr->config->fileName())) {
        delete config;

        return false;
    }

    # 保存配置，并检查是否需要监视文件变化
    const bool saved = config->save();

    # 如果配置了监视文件变化，并且 Watcher 对象存在，并且保存成功，则返回 true
    if (config->isWatch() && d_ptr->watcher && saved) {
        delete config;

        return true;
    }

    # 替换当前配置为新配置
    d_ptr->replace(config);

    return true;
}


# 返回配置指针
xmrig::Config *xmrig::Base::config() const
{
    assert(d_ptr->config != nullptr);

    return d_ptr->config;
}


# 添加监听器
void xmrig::Base::addListener(IBaseListener *listener)
{
    d_ptr->listeners.push_back(listener);
}


# 文件变化时的回调函数
void xmrig::Base::onFileChanged(const String &fileName)
{
    # 记录警告日志，指示文件被更改，重新加载配置
    LOG_WARN("%s " YELLOW("\"%s\" was changed, reloading configuration"), Tags::config(), fileName.data());

    # 创建一个 JsonChain 对象，添加文件名
    JsonChain chain;
    chain.addFile(fileName);

    # 创建一个新的 Config 对象
    auto config = new Config();

    # 如果无法从 JsonChain 对象中读取配置文件内容
    if (!config->read(chain, chain.fileName())) {
        # 记录错误日志，指示重新加载配置失败
        LOG_ERR("%s " RED("reloading failed"), Tags::config());

        # 释放 Config 对象的内存
        delete config;
        # 退出函数
        return;
    }

    # 用新的配置对象替换原有的配置对象
    d_ptr->replace(config);
#ifdef XMRIG_FEATURE_API
void xmrig::Base::onRequest(IApiRequest &request)
{
    // 如果请求方法为 GET
    if (request.method() == IApiRequest::METHOD_GET) {
        // 如果请求的 URL 为 kConfigPathV1 或 kConfigPathV2
        if (request.url() == kConfigPathV1 || request.url() == kConfigPathV2) {
            // 如果请求被限制
            if (request.isRestricted()) {
                // 返回 403 状态码
                return request.done(403);
            }

            // 接受请求
            request.accept();
            // 获取配置信息并以 JSON 格式返回
            config()->getJSON(request.doc());
        }
    }
    // 如果请求方法为 PUT 或 POST
    else if (request.method() == IApiRequest::METHOD_PUT || request.method() == IApiRequest::METHOD_POST) {
        // 如果请求的 URL 为 kConfigPathV1 或 kConfigPathV2
        if (request.url() == kConfigPathV1 || request.url() == kConfigPathV2) {
            // 接受请求
            request.accept();

            // 如果重新加载配置信息失败
            if (!reload(request.json())) {
                // 返回 400 状态码
                return request.done(400);
            }

            // 返回 204 状态码
            request.done(204);
        }
    }
}
#endif
```