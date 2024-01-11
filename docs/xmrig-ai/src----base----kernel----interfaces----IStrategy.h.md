# `xmrig\src\base\kernel\interfaces\IStrategy.h`

```
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据 GNU 通用公共许可证的条款发布，由
 * 自由软件基金会发布的版本 3 或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_ISTRATEGY_H
#define XMRIG_ISTRATEGY_H


#include <cstdint>


namespace xmrig {


class Algorithm;
class IClient;
class JobResult;
class ProxyUrl;


class IStrategy
{
public:
    virtual ~IStrategy() = default;

    virtual bool isActive() const                      = 0;  // 检查策略是否处于活动状态
    virtual IClient *client() const                    = 0;  // 获取客户端
    virtual int64_t submit(const JobResult &result)    = 0;  // 提交作业结果
    virtual void connect()                             = 0;  // 连接
    virtual void resume()                              = 0;  // 恢复
    virtual void setAlgo(const Algorithm &algo)        = 0;  // 设置算法
    virtual void setProxy(const ProxyUrl &proxy)       = 0;  // 设置代理
    virtual void stop()                                = 0;  // 停止
    // 定义一个纯虚函数，用于在派生类中实现，表示在给定时间点执行某些操作
    virtual void tick(uint64_t now)                    = 0;
}; 
// 结束了 xmrig 命名空间的定义

} /* namespace xmrig */
// 声明当前代码块中的内容属于 xmrig 命名空间

#endif // XMRIG_ISTRATEGY_H
// 结束了对 XMRIG_ISTRATEGY_H 的条件编译声明
```