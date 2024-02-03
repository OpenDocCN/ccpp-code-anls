# `xmrig\src\base\net\stratum\BaseClient.cpp`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据GNU通用公共许可证的条款发布
 * 由自由软件基金会发布，许可证的第3版
 * （在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/net/stratum/BaseClient.h"  // 引入BaseClient头文件
#include "3rdparty/fmt/core.h"  // 引入fmt库的core头文件
#include "3rdparty/rapidjson/document.h"  // 引入rapidjson库的document头文件
#include "base/io/Env.h"  // 引入Env头文件
#include "base/io/log/Log.h"  // 引入Log头文件
#include "base/io/log/Tags.h"  // 引入Tags头文件
#include "base/kernel/interfaces/IClientListener.h"  // 引入IClientListener头文件
#include "base/net/stratum/SubmitResult.h"  // 引入SubmitResult头文件

namespace xmrig {

int64_t BaseClient::m_sequence = 1;  // 初始化BaseClient类的静态成员m_sequence为1

} /* namespace xmrig */

xmrig::BaseClient::BaseClient(int id, IClientListener *listener) :  // BaseClient类的构造函数
    m_listener(listener),  // 初始化m_listener成员
    m_id(id)  // 初始化m_id成员
{
}

void xmrig::BaseClient::setPool(const Pool &pool)  // 设置Pool对象的函数
{
    if (!pool.isValid()) {  // 如果pool对象无效，则返回
        return;
    }

    m_pool      = pool;  // 初始化m_pool成员
    m_user      = Env::expand(pool.user());  // 初始化m_user成员
    m_password  = Env::expand(pool.password());  // 初始化m_password成员
    m_rigId     = Env::expand(pool.rigId());  // 初始化m_rigId成员
    m_tag       = fmt::format("{} " CYAN_BOLD("{}"), Tags::network(), m_pool.url().data());  // 初始化m_tag成员
}

bool xmrig::BaseClient::handleResponse(int64_t id, const rapidjson::Value &result, const rapidjson::Value &error)  // 处理响应的函数
{
    if (id == 1) {  // 如果id为1，则返回false
        return false;
    }

    auto it = m_callbacks.find(id);  // 查找id对应的回调函数
    # 检查是否存在指定的键值对
    if (it != m_callbacks.end()) {
        # 计算时间间隔，单位为毫秒
        const uint64_t elapsed = Chrono::steadyMSecs() - it->second.ts;

        # 如果返回的结果是一个对象
        if (error.IsObject()) {
            # 调用回调函数，传入错误对象、false和时间间隔
            it->second.callback(error, false, elapsed);
        }
        # 如果返回的结果不是一个对象
        else {
            # 调用回调函数，传入结果对象、true和时间间隔
            it->second.callback(result, true, elapsed);
        }

        # 从回调函数映射中删除指定的键值对
        m_callbacks.erase(it);

        # 返回true
        return true;
    }

    # 如果不存在指定的键值对，则返回false
    return false;
# 检查是否有与给定ID对应的结果
bool xmrig::BaseClient::handleSubmitResponse(int64_t id, const char *error)
{
    # 在结果映射中查找给定ID对应的结果
    auto it = m_results.find(id);
    # 如果找到了对应的结果
    if (it != m_results.end()) {
        # 标记结果为已完成
        it->second.done();
        # 调用监听器的onResultAccepted方法，通知结果被接受
        m_listener->onResultAccepted(this, it->second, error);
        # 从结果映射中删除该结果
        m_results.erase(it);

        # 返回true，表示处理成功
        return true;
    }

    # 如果未找到对应的结果，返回false
    return false;
}
```