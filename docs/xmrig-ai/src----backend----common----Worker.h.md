# `xmrig\src\backend\common\Worker.h`

```
/* XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据GNU通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版，或者
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_WORKER_H
#define XMRIG_WORKER_H


#include "backend/common/interfaces/IWorker.h"


namespace xmrig {


class Worker : public IWorker
{
public:
    Worker(size_t id, int64_t affinity, int priority);

    size_t threads() const override                         { return 1; }

protected:
    inline int64_t affinity() const                         { return m_affinity; }
    inline size_t id() const override                       { return m_id; }
    inline uint32_t node() const                            { return m_node; }

    uint64_t m_count                = 0;

private:
    const int64_t m_affinity;
    const size_t m_id;
    uint32_t m_node                 = 0;
};


} // namespace xmrig


#endif /* XMRIG_WORKER_H */
```