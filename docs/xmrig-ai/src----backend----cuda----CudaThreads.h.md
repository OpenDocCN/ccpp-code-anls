# `xmrig\src\backend\cuda\CudaThreads.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的，
 * 但没有任何保证；甚至没有暗示的保证
 * 商品性或适用于特定目的。有关更多详细信息，请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CUDATHREADS_H
#define XMRIG_CUDATHREADS_H


#include <vector>


#include "backend/cuda/CudaThread.h"
#include "backend/cuda/wrappers/CudaDevice.h"


namespace xmrig {


class CudaThreads
{
public:
    // 默认构造函数
    CudaThreads() = default;
    // 从JSON值构造函数
    CudaThreads(const rapidjson::Value &value);
    // 从CUDA设备和算法构造函数
    CudaThreads(const std::vector<CudaDevice> &devices, const Algorithm &algorithm);

    // 判断是否为空
    inline bool isEmpty() const                              { return m_data.empty(); }
    // 返回数据
    inline const std::vector<CudaThread> &data() const       { return m_data; }
    // 返回数据数量
    inline size_t count() const                              { return m_data.size(); }
    // 添加线程
    inline void add(const CudaThread &thread)                { m_data.push_back(thread); }
    // 预留容量
    inline void reserve(size_t capacity)                     { m_data.reserve(capacity); }

    // 不等于运算符重载
    inline bool operator!=(const CudaThreads &other) const   { return !isEqual(other); }
    // 等于运算符重载
    inline bool operator==(const CudaThreads &other) const   { return isEqual(other); }

    // 判断是否相等
    bool isEqual(const CudaThreads &other) const;
    // 转换为JSON值
    rapidjson::Value toJSON(rapidjson::Document &doc) const;

private:
    std::vector<CudaThread> m_data;
};
} /* namespace xmrig */  // 结束 xmrig 命名空间

#endif /* XMRIG_CUDATHREADS_H */  // 结束 XMRIG_CUDATHREADS_H 的条件编译
```