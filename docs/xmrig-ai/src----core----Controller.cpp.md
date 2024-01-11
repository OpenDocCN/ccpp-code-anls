# `xmrig\src\core\Controller.cpp`

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
 *   但没有任何保证；甚至没有暗示的保证
 *   商品性或适用于特定目的。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "core/Controller.h"  // 包含Controller类的头文件
#include "backend/cpu/Cpu.h"   // 包含Cpu类的头文件
#include "core/config/Config.h"  // 包含Config类的头文件
#include "core/Miner.h"  // 包含Miner类的头文件
#include "crypto/common/VirtualMemory.h"  // 包含VirtualMemory类的头文件
#include "net/Network.h"  // 包含Network类的头文件


#ifdef XMRIG_FEATURE_API
#   include "base/api/Api.h"  // 如果定义了XMRIG_FEATURE_API，则包含Api类的头文件
#   include "hw/api/HwApi.h"  // 如果定义了XMRIG_FEATURE_API，则包含HwApi类的头文件
#endif


#include <cassert>  // 包含断言的头文件


xmrig::Controller::Controller(Process *process) :  // Controller类的构造函数，参数为Process类型指针
    Base(process)  // 调用基类Base的构造函数
{
}


xmrig::Controller::~Controller()  // Controller类的析构函数
{
    VirtualMemory::destroy();  // 销毁VirtualMemory对象
}


int xmrig::Controller::init()  // 初始化函数
{
    Base::init();  // 调用基类Base的init函数

    VirtualMemory::init(config()->cpu().memPoolSize(), config()->cpu().hugePageSize());  // 初始化VirtualMemory对象

    m_network = std::make_shared<Network>(this);  // 创建Network对象的共享指针，传入this指针

#   ifdef XMRIG_FEATURE_API
    m_hwApi = std::make_shared<HwApi>();  // 如果定义了XMRIG_FEATURE_API，则创建HwApi对象的共享指针
    api()->addListener(m_hwApi.get());  // 调用api()函数，添加HwApi对象的监听器
#   endif

    return 0;  // 返回0
}


void xmrig::Controller::start()  // 开始函数
{
    Base::start();  // 调用基类Base的start函数

    m_miner = std::make_shared<Miner>(this);  // 创建Miner对象的共享指针，传入this指针

    network()->connect();  // 调用network()函数，连接网络
}


void xmrig::Controller::stop()  // 停止函数
{
    Base::stop();  // 调用基类Base的stop函数

    m_network.reset();  // 重置m_network对象

    m_miner->stop();  // 停止Miner对象
    m_miner.reset();  // 重置m_miner对象
}


xmrig::Miner *xmrig::Controller::miner() const  // 获取Miner对象的函数
{
    assert(m_miner);  // 断言m_miner对象存在

    return m_miner.get();  // 返回m_miner对象
}
# 返回网络指针
xmrig::Network *xmrig::Controller::network() const
{
    # 断言网络指针不为空
    assert(m_network);
    # 返回网络指针
    return m_network.get();
}

# 执行命令
void xmrig::Controller::execCommand(char command) const
{
    # 执行矿工命令
    miner()->execCommand(command);
    # 执行网络命令
    network()->execCommand(command);
}
```