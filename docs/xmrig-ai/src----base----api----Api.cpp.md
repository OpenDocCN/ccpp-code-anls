# `xmrig\src\base\api\Api.cpp`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款
 *   许可证的版本为 3，或者（按您的选择）任何以后的版本。
 *
 *   本程序是希望它有用的分发
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多详情请参见
 *   GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include <uv.h>


#include "base/api/Api.h"
#include "base/api/interfaces/IApiListener.h"
#include "base/api/requests/HttpApiRequest.h"
#include "base/crypto/keccak.h"
#include "base/io/Env.h"
#include "base/io/json/Json.h"
#include "base/kernel/Base.h"
#include "base/tools/Chrono.h"
#include "base/tools/Cvt.h"
#include "core/config/Config.h"
#include "core/Controller.h"
#include "version.h"


#ifdef XMRIG_FEATURE_HTTP
#   include "base/api/Httpd.h"
#endif


#include <thread>


namespace xmrig {


static rapidjson::Value getResources(rapidjson::Document &doc)
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();

    size_t rss = 0;
    // 获取当前进程的常驻内存大小
    uv_resident_set_memory(&rss);

    // 创建 JSON 对象
    Value out(kObjectType);
    Value memory(kObjectType);
    Value load_average(kArrayType);

    // 添加内存信息到 JSON 对象
    memory.AddMember("free",                uv_get_free_memory(), allocator);
    memory.AddMember("total",               uv_get_total_memory(), allocator);
    memory.AddMember("resident_set_memory", static_cast<uint64_t>(rss), allocator);

    double loadavg[3] = { 0.0 };
    // 获取系统负载信息
    uv_loadavg(loadavg);
    # 遍历 loadavg 数组中的每个值，将其规范化后添加到 load_average 对象中
    for (double value : loadavg) {
        load_average.PushBack(Json::normalize(value, true), allocator);
    }

    # 将 memory 对象添加到 out 对象中，使用 allocator 进行内存分配
    out.AddMember("memory", memory, allocator);
    # 将 load_average 对象添加到 out 对象中，使用 allocator 进行内存分配
    out.AddMember("load_average", load_average, allocator);
    # 将硬件并发数添加到 out 对象中，使用 allocator 进行内存分配
    out.AddMember("hardware_concurrency", std::thread::hardware_concurrency(), allocator);

    # 返回 out 对象
    return out;
} // namespace xmrig

// 构造函数，初始化成员变量
xmrig::Api::Api(Base *base) :
    m_base(base), // 初始化 m_base 成员变量
    m_timestamp(Chrono::currentMSecsSinceEpoch()) // 初始化 m_timestamp 成员变量为当前时间

{
    base->addListener(this); // 将当前对象添加为 base 的监听器

    genId(base->config()->apiId()); // 生成 ID

}

// 析构函数
xmrig::Api::~Api()
{
#   ifdef XMRIG_FEATURE_HTTP
    delete m_httpd; // 删除 m_httpd 对象
#   endif
}

// 处理 HTTP 请求
void xmrig::Api::request(const HttpData &req)
{
    HttpApiRequest request(req, m_base->config()->http().isRestricted()); // 创建 HttpApiRequest 对象

    exec(request); // 执行请求
}

// 启动 API
void xmrig::Api::start()
{
    genWorkerId(m_base->config()->apiWorkerId()); // 生成 worker ID

#   ifdef XMRIG_FEATURE_HTTP
    m_httpd = new Httpd(m_base); // 创建 Httpd 对象
    m_httpd->start(); // 启动 HTTP 服务
#   endif
}

// 停止 API
void xmrig::Api::stop()
{
#   ifdef XMRIG_FEATURE_HTTP
    m_httpd->stop(); // 停止 HTTP 服务
#   endif
}

// 定时任务
void xmrig::Api::tick()
{
#   ifdef XMRIG_FEATURE_HTTP
    if (m_httpd->isBound() || !m_base->config()->http().isEnabled()) {
        return; // 如果 HTTP 服务已绑定或者未启用，则直接返回
    }

    if (++m_ticks % 10 == 0) { // 每 10 次执行一次
        m_ticks = 0; // 重置计数
        m_httpd->start(); // 启动 HTTP 服务
    }
#   endif
}

// 配置变更时的处理
void xmrig::Api::onConfigChanged(Config *config, Config *previousConfig)
{
    if (config->apiId() != previousConfig->apiId()) {
        genId(config->apiId()); // 生成 ID
    }

    if (config->apiWorkerId() != previousConfig->apiWorkerId()) {
        genWorkerId(config->apiWorkerId()); // 生成 worker ID
    }
}

// 执行 API 请求
void xmrig::Api::exec(IApiRequest &request)
{
    using namespace rapidjson;

    if (request.type() == IApiRequest::REQ_SUMMARY) { // 如果请求类型为总结
        auto &allocator = request.doc().GetAllocator(); // 获取分配器

        request.accept(); // 接受请求

        auto &reply = request.reply(); // 获取回复对象
        reply.AddMember("id",         StringRef(m_id),       allocator); // 添加 ID
        reply.AddMember("worker_id",  m_workerId.toJSON(), allocator); // 添加 worker ID
        reply.AddMember("uptime",     (Chrono::currentMSecsSinceEpoch() - m_timestamp) / 1000, allocator); // 添加运行时间
        reply.AddMember("restricted", request.isRestricted(), allocator); // 添加是否受限
        reply.AddMember("resources",  getResources(request.doc()), allocator); // 添加资源信息

        Value features(kArrayType); // 创建特性数组
# 如果定义了 XMRIG_FEATURE_API，则将字符串 "api" 添加到 features 数组中
        features.PushBack("api", allocator);
# 如果定义了 XMRIG_FEATURE_ASM，则将字符串 "asm" 添加到 features 数组中
        features.PushBack("asm", allocator);
# 如果定义了 XMRIG_FEATURE_HTTP，则将字符串 "http" 添加到 features 数组中
        features.PushBack("http", allocator);
# 如果定义了 XMRIG_FEATURE_HWLOC，则将字符串 "hwloc" 添加到 features 数组中
        features.PushBack("hwloc", allocator);
# 如果定义了 XMRIG_FEATURE_TLS，则将字符串 "tls" 添加到 features 数组中
        features.PushBack("tls", allocator);
# 如果定义了 XMRIG_FEATURE_OPENCL，则将字符串 "opencl" 添加到 features 数组中
        features.PushBack("opencl", allocator);
# 如果定义了 XMRIG_FEATURE_CUDA，则将字符串 "cuda" 添加到 features 数组中
        features.PushBack("cuda", allocator);
# 将 features 数组添加到 reply 对象中
        reply.AddMember("features", features, allocator);
    }

    # 遍历监听器列表，对每个监听器调用 onRequest 方法
    for (IApiListener *listener : m_listeners) {
        listener->onRequest(request);

        # 如果请求已完成，则返回
        if (request.isDone()) {
            return;
        }
    }

    # 根据请求的状态，调用 done 方法
    request.done(request.isNew() ? 404 : 200);
}


# 生成 ID 的方法，根据给定的 ID 字符串
void xmrig::Api::genId(const String &id)
{
    # 将 m_id 数组清零
    memset(m_id, 0, sizeof(m_id));

    # 如果给定的 ID 长度大于 0，则将其拷贝到 m_id 数组中
    if (id.size() > 0) {
        strncpy(m_id, id.data(), sizeof(m_id) - 1);
        return;
    }

    # 获取系统网络接口地址列表
    uv_interface_address_t *interfaces = nullptr;
    int count = 0;

    # 如果获取失败，则返回
    if (uv_interface_addresses(&interfaces, &count) < 0) {
        return;
    }
    // 遍历接口列表，i 从 0 到 count-1
    for (int i = 0; i < count; i++) {
        // 检查接口是否非内部接口且地址族为 AF_INET
        if (!interfaces[i].is_internal && interfaces[i].address.address4.sin_family == AF_INET) {
            // 创建一个长度为 200 的哈希数组
            uint8_t hash[200];
            // 获取物理地址的大小
            const size_t addrSize = sizeof(interfaces[i].phys_addr);
            // 计算输入数据的大小
            const size_t inSize   = (sizeof(APP_KIND) - 1) + addrSize + sizeof(uint16_t);
            // 获取配置文件中 HTTP 端口号
            const auto port       = static_cast<uint16_t>(m_base->config()->http().port());

            // 创建一个长度为 inSize 的输入数组，并初始化为 0
            auto *input = new uint8_t[inSize]();
            // 将端口号拷贝到输入数组中
            memcpy(input, &port, sizeof(uint16_t));
            // 将物理地址拷贝到输入数组中
            memcpy(input + sizeof(uint16_t), interfaces[i].phys_addr, addrSize);
            // 将应用程序类型拷贝到输入数组中
            memcpy(input + sizeof(uint16_t) + addrSize, APP_KIND, (sizeof(APP_KIND) - 1));

            // 对输入数组进行哈希运算，结果存储在 hash 数组中
            keccak(input, inSize, hash);
            // 将哈希结果转换为十六进制字符串，存储在 m_id 中
            Cvt::toHex(m_id, sizeof(m_id), hash, 8);

            // 释放输入数组的内存
            delete [] input;
            // 跳出循环
            break;
        }
    }

    // 释放接口地址列表的内存
    uv_free_interface_addresses(interfaces, count);
# 生成工作人员ID的函数，参数为字符串类型的ID
void xmrig::Api::genWorkerId(const String &id)
{
    # 将传入的ID进行环境变量扩展，并赋值给m_workerId
    m_workerId = Env::expand(id);
    # 如果m_workerId为空，则将其赋值为当前主机名
    if (m_workerId.isEmpty()) {
        m_workerId = Env::hostname();
    }
}
```