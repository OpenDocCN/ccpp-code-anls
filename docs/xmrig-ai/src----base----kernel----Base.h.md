# `xmrig\src\base\kernel\Base.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   许可证的版本为3，或者（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_BASE_H
#define XMRIG_BASE_H


#include "3rdparty/rapidjson/fwd.h"
#include "base/api/interfaces/IApiListener.h"
#include "base/kernel/interfaces/IConfigListener.h"
#include "base/kernel/interfaces/IWatcherListener.h"
#include "base/tools/Object.h"


namespace xmrig {


class Api;
class BasePrivate;
class Config;
class IBaseListener;
class Process;


class Base : public IWatcherListener, public IApiListener
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(Base)

    // 构造函数，接受一个Process指针
    Base(Process *process);
    // 析构函数
    ~Base() override;

    // 虚函数，返回是否准备就绪
    virtual bool isReady() const;
    // 虚函数，初始化
    virtual int init();
    // 虚函数，启动
    virtual void start();
    // 虚函数，停止
    virtual void stop();

    // 返回Api指针
    Api *api() const;
    // 返回是否在后台运行
    bool isBackground() const;
    // 重新加载配置
    bool reload(const rapidjson::Value &json);
    // 返回Config指针
    Config *config() const;
    // 添加监听器
    void addListener(IBaseListener *listener);

protected:
    // 文件改变时的回调函数
    void onFileChanged(const String &fileName) override;

#   ifdef XMRIG_FEATURE_API
    // API请求的回调函数
    void onRequest(IApiRequest &request) override;
#   endif

private:
    BasePrivate *d_ptr;
};


} /* namespace xmrig */


#endif /* XMRIG_BASE_H */
```