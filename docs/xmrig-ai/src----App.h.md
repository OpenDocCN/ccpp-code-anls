# `xmrig\src\App.h`

```
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 * 要么是许可证的第3版，要么（在您的选择下）是任何以后的版本。
 *
 * 本程序是希望它有用，但没有任何担保；甚至没有暗示的担保。
 * 有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_APP_H
#define XMRIG_APP_H

#include "base/kernel/interfaces/IConsoleListener.h"
#include "base/kernel/interfaces/ISignalListener.h"
#include "base/tools/Object.h"

#include <memory>

namespace xmrig {

class Console;
class Controller;
class Network;
class Process;
class Signals;

class App : public IConsoleListener, public ISignalListener
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(App)  // 禁用默认的拷贝和移动构造函数

    App(Process *process);  // 构造函数，接受一个Process对象作为参数
    ~App() override;  // 虚析构函数

    int exec();  // 执行函数，返回整型值

protected:
    void onConsoleCommand(char command) override;  // 重写IConsoleListener接口的onConsoleCommand函数
    void onSignal(int signum) override;  // 重写ISignalListener接口的onSignal函数

private:
    bool background(int &rc);  // 后台函数，接受一个整型引用参数
    void close();  // 关闭函数

    std::shared_ptr<Console> m_console;  // Console类的智能指针成员变量
    # 创建一个指向Controller对象的shared_ptr智能指针
    std::shared_ptr<Controller> m_controller;
    # 创建一个指向Signals对象的shared_ptr智能指针
    std::shared_ptr<Signals> m_signals;
}; 
// 结束 xmrig 命名空间

} /* namespace xmrig */
// 结束 namespace xmrig

#endif /* XMRIG_APP_H */
// 结束条件编译指令，检查是否定义了 XMRIG_APP_H，如果没有则包含该头文件，防止重复包含
```