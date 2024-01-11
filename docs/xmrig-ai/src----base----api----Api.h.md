# `xmrig\src\base\api\Api.h`

```
/* XMRig
 * 版权所有 (c) 2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款
 *   许可证的版本，或者（在您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有适销性或特定用途的暗示保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_API_H
#define XMRIG_API_H


#include <vector>


#include "base/kernel/interfaces/IBaseListener.h"
#include "base/tools/String.h"


namespace xmrig {


class Base;
class Httpd;
class HttpData;
class IApiListener;
class IApiRequest;
class String;


class Api : public IBaseListener
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(Api)

    explicit Api(Base *base);
    ~Api() override;

    inline const char *id() const                   { return m_id; }
    inline const char *workerId() const             { return m_workerId; }
    inline void addListener(IApiListener *listener) { m_listeners.push_back(listener); }

    void request(const HttpData &req);
    void start();
    void stop();
    void tick();

protected:
    void onConfigChanged(Config *config, Config *previousConfig) override;

private:
    void exec(IApiRequest &request);
    void genId(const String &id);
    void genWorkerId(const String &id);

    Base *m_base;
    char m_id[32]{}; // 用于存储 ID 的字符数组
    const uint64_t m_timestamp; // 时间戳
    Httpd *m_httpd  = nullptr; // HTTP 服务器对象
    std::vector<IApiListener *> m_listeners; // API 监听器列表
    String m_workerId; // 工作 ID
    uint8_t m_ticks = 0; // 计数器
};


} // namespace xmrig
#endif // XMRIG_API_H

这行代码是C++中的预处理指令，用于结束一个条件编译块。在这里，它用于结束对XMRIG_API_H宏的条件编译。
```