# `xmrig\src\backend\cpu\platform\HwlocCpuInfo.h`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2023 XMRig       <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据 GNU 通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版或
 *   （根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_HWLOCCPUINFO_H
#define XMRIG_HWLOCCPUINFO_H


#include "backend/cpu/platform/BasicCpuInfo.h"


using hwloc_obj_t = struct hwloc_obj *;


namespace xmrig {


class HwlocCpuInfo : public BasicCpuInfo
{
public:
    XMRIG_DISABLE_COPY_MOVE(HwlocCpuInfo)

    HwlocCpuInfo();
    ~HwlocCpuInfo() override;

protected:
    bool membind(hwloc_const_bitmap_t nodeset) override;
    CpuThreads threads(const Algorithm &algorithm, uint32_t limit) const override;

    inline const char *backend() const override                     { return m_backend; }
    inline const std::vector<uint32_t> &nodeset() const override    { return m_nodeset; }
    inline hwloc_topology_t topology() const override               { return m_topology; }
    inline size_t cores() const override                            { return m_cores; }
    inline size_t L2() const override                               { return m_cache[2]; }
    inline size_t L3() const override                               { return m_cache[3]; }
    inline size_t nodes() const override                            { return m_nodes; }
    inline size_t packages() const override                         { return m_packages; }
// 声明私有成员函数，返回指定算法和限制条件下的 CPU 线程
CpuThreads allThreads(const Algorithm &algorithm, uint32_t limit) const;

// 处理顶层缓存，获取指定算法和限制条件下的 CPU 线程
void processTopLevelCache(hwloc_obj_t cache, const Algorithm &algorithm, CpuThreads &threads, size_t limit) const;

// 设置线程数量
void setThreads(size_t threads);

// 声明私有成员变量
char m_backend[20]          = { 0 }; // 存储后端信息的字符数组
hwloc_topology_t m_topology = nullptr; // 存储硬件拓扑信息的指针
size_t m_cache[5]           = { 0 }; // 存储缓存信息的数组
size_t m_cores              = 0; // 存储核心数量
size_t m_nodes              = 0; // 存储节点数量
size_t m_packages           = 0; // 存储包数量
std::vector<uint32_t> m_nodeset; // 存储节点集合的向量
```