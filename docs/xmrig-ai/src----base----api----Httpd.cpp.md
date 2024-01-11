# `xmrig\src\base\api\Httpd.cpp`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，
 *   但没有任何保证；甚至没有适销性或特定用途的暗示保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/api/Httpd.h"
#include "3rdparty/llhttp/llhttp.h"
#include "base/api/Api.h"
#include "base/io/log/Log.h"
#include "base/net/http/HttpApiResponse.h"
#include "base/net/http/HttpData.h"
#include "base/net/tools/TcpServer.h"
#include "core/config/Config.h"
#include "core/Controller.h"


#ifdef XMRIG_FEATURE_TLS
#   include "base/net/https/HttpsServer.h"
#else
#   include "base/net/http/HttpServer.h"
#endif


namespace xmrig {

static const char *kAuthorization = "authorization";

#ifdef _WIN32
static const char *favicon = nullptr;
static size_t faviconSize  = 0;
#endif

} // namespace xmrig


// 构造函数，初始化Httpd对象
xmrig::Httpd::Httpd(Base *base) :
    m_base(base)
{
    m_httpListener = std::make_shared<HttpListener>(this);

    base->addListener(this);
}


// 析构函数
xmrig::Httpd::~Httpd() = default;


// 启动HTTP服务
bool xmrig::Httpd::start()
{
    const auto &config = m_base->config()->http();

    // 如果HTTP服务未启用，则返回true
    if (!config.isEnabled()) {
        return true;
    }

    bool tls = false;

#   ifdef XMRIG_FEATURE_TLS
    // 如果支持TLS，则创建HttpsServer对象
    m_http = new HttpsServer(m_httpListener);
    // 设置TLS配置
    tls = m_http->setTls(m_base->config()->tls());
#   else
    // 否则创建HttpServer对象
    m_http = new HttpServer(m_httpListener);
#   endif
    // 创建一个新的 TCP 服务器对象，使用配置文件中的主机和端口号，并指定 HTTP 处理器
    m_server = new TcpServer(config.host(), config.port(), m_http);

    // 绑定服务器到指定的主机和端口，返回绑定结果
    const int rc = m_server->bind();
    // 打印绑定结果信息，包括主机、端口和绑定状态
    Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-13s") CSI "1;%dm%s:%d" " " RED_BOLD("%s"),
               "HTTP API",
               tls ? 32 : 36,
               config.host().data(),
               rc < 0 ? config.port() : rc,
               rc < 0 ? uv_strerror(rc) : ""
               );

    // 如果绑定失败，则停止服务器并返回 false
    if (rc < 0) {
        stop();
        return false;
    }

    // 将绑定成功的端口号转换为 uint16_t 类型，并赋值给 m_port
    m_port = static_cast<uint16_t>(rc);
#ifdef _WIN32
// 如果是在 Windows 平台下
// NOLINTNEXTLINE(cppcoreguidelines-pro-type-cstyle-cast, performance-no-int-to-ptr)
// 忽略下一行代码的 lint 错误
HRSRC src = FindResource(nullptr, MAKEINTRESOURCE(1), RT_ICON);
// 查找资源，获取图标资源句柄
if (src != nullptr) {
    HGLOBAL res = LoadResource(nullptr, src);
    // 加载资源，获取资源句柄
    if (res != nullptr) {
        favicon     = static_cast<const char *>(LockResource(res));
        // 锁定资源，获取资源数据
        faviconSize = SizeofResource(nullptr, src);
        // 获取资源大小
    }
}
#endif

// 返回 true
return true;
}


void xmrig::Httpd::stop()
{
    delete m_server;
    // 释放服务器对象内存
    delete m_http;
    // 释放 HTTP 对象内存

    m_server = nullptr;
    // 将服务器对象指针置空
    m_http   = nullptr;
    // 将 HTTP 对象指针置空
    m_port   = 0;
    // 将端口号置为 0
}



void xmrig::Httpd::onConfigChanged(Config *config, Config *previousConfig)
{
    if (config->http() == previousConfig->http()) {
        // 如果新旧配置的 HTTP 部分相同，则返回
        return;
    }

    stop();
    // 停止 HTTP 服务
    start();
    // 启动 HTTP 服务
}


void xmrig::Httpd::onHttpData(const HttpData &data)
{
    if (data.method == HTTP_OPTIONS) {
        // 如果请求方法为 OPTIONS，则返回空响应
        return HttpApiResponse(data.id()).end();
    }

    if (data.method == HTTP_GET && data.url == "/favicon.ico") {
#ifdef _WIN32
        // 如果是在 Windows 平台下
        if (favicon != nullptr) {
            // 如果图标数据不为空
            HttpResponse response(data.id());
            // 创建 HTTP 响应对象
            response.setHeader(HttpData::kContentType, "image/x-icon");
            // 设置响应头的内容类型为图标类型

            return response.end(favicon, faviconSize);
            // 返回包含图标数据的响应
        }
#endif

        return HttpResponse(data.id(), 404 /* NOT_FOUND */).end();
        // 返回 404 错误响应
    }

    if (data.method > 4) {
        // 如果请求方法大于 4
        return HttpApiResponse(data.id(), 405 /* METHOD_NOT_ALLOWED */).end();
        // 返回 405 错误响应
    }

    const int status = auth(data);
    // 调用 auth 函数验证请求
    if (status != 200) {
        // 如果验证状态不为 200
        return HttpApiResponse(data.id(), status).end();
        // 返回对应状态的错误响应
    }

    if (data.method != HTTP_GET) {
        // 如果请求方法不是 GET
        if (m_base->config()->http().isRestricted()) {
            // 如果 HTTP 配置为受限制
            return HttpApiResponse(data.id(), 403 /* FORBIDDEN */).end();
            // 返回 403 错误响应
        }

        if (!data.headers.count(HttpData::kContentTypeL) || data.headers.at(HttpData::kContentTypeL) != HttpData::kApplicationJson) {
            // 如果请求头中没有内容类型，或者内容类型不是 JSON
            return HttpApiResponse(data.id(), 415 /* UNSUPPORTED_MEDIA_TYPE */).end();
            // 返回 415 错误响应
        }
    }
}
    # 通过指针 m_base 调用 api() 方法，然后调用 request() 方法，传入参数 data
    m_base->api()->request(data);
// 检查是否需要进行身份验证，如果不需要则返回 200，否则返回 401
int xmrig::Httpd::auth(const HttpData &req) const
{
    // 获取 HTTP 配置信息
    const Http &config = m_base->config()->http();

    // 如果请求头中没有包含 Authorization 字段
    if (!req.headers.count(kAuthorization)) {
        return config.isAuthRequired() ? 401 /* UNAUTHORIZED */ : 200;
    }

    // 如果配置中的 token 为空，则返回 401
    if (config.token().isNull()) {
        return 401 /* UNAUTHORIZED */;
    }

    // 获取 Authorization 字段的值
    const std::string &token = req.headers.at(kAuthorization);
    const size_t size        = token.size();

    // 如果 token 的长度小于 8，或者 token 的长度减去 "Bearer " 的长度不等于配置中 token 的长度，或者 token 不以 "Bearer " 开头，则返回 403
    if (token.size() < 8 || config.token().size() != size - 7 || memcmp("Bearer ", token.c_str(), 7) != 0) {
        return 403 /* FORBIDDEN */;
    }

    // 比较 token 中去掉 "Bearer " 后的部分与配置中的 token 是否相等，相等则返回 200，否则返回 403
    return strncmp(config.token().data(), token.c_str() + 7, config.token().size()) == 0 ? 200 : 403 /* FORBIDDEN */;
}
```