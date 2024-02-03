# `xmrig\src\backend\opencl\OclThread.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有暗示的保证
 * 商品性或适用于特定目的。有关更多详细信息，请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_OCLTHREAD_H
#define XMRIG_OCLTHREAD_H


#include "3rdparty/rapidjson/fwd.h"


#include <bitset>
#include <vector>


namespace xmrig {


class OclThread
{
public:
    // 删除默认构造函数
    OclThread() = delete;
    // 构造函数，初始化成员变量
    OclThread(uint32_t index, uint32_t intensity, uint32_t worksize, uint32_t stridedIndex, uint32_t memChunk, uint32_t threads, uint32_t unrollFactor) :
        m_threads(threads, -1),
        m_index(index),
        m_memChunk(memChunk),
        m_stridedIndex(stridedIndex),
        m_unrollFactor(unrollFactor),
        m_worksize(worksize)
    {
        setIntensity(intensity);
    }

#   ifdef XMRIG_ALGO_RANDOMX
    // 构造函数，初始化成员变量
    OclThread(uint32_t index, uint32_t intensity, uint32_t worksize, uint32_t threads, bool gcnAsm, bool datasetHost, uint32_t bfactor) :
        m_datasetHost(datasetHost),
        m_gcnAsm(gcnAsm),
        m_fields(2),
        m_threads(threads, -1),
        m_bfactor(bfactor),
        m_index(index),
        m_memChunk(0),
        m_stridedIndex(0),
        m_worksize(worksize)
    {
        setIntensity(intensity);
    }
#   endif

#   ifdef XMRIG_ALGO_KAWPOW
    # 定义 OclThread 类的构造函数，接受四个参数：index, intensity, worksize, threads
    OclThread(uint32_t index, uint32_t intensity, uint32_t worksize, uint32_t threads) :
        # 初始化 m_fields 属性为 8
        m_fields(8),
        # 初始化 m_threads 属性为长度为 threads 的数组，初始值为 -1
        m_threads(threads, -1),
        # 初始化 m_index 属性为传入的 index 参数
        m_index(index),
        # 初始化 m_memChunk 属性为 0
        m_memChunk(0),
        # 初始化 m_stridedIndex 属性为 0
        m_stridedIndex(0),
        # 初始化 m_unrollFactor 属性为 1
        m_unrollFactor(1),
        # 初始化 m_worksize 属性为传入的 worksize 参数
        m_worksize(worksize)
    {
        # 调用 setIntensity 方法，传入 intensity 参数
        setIntensity(intensity);
    }
// 结束条件
    OclThread(const rapidjson::Value &value);

    // 返回是否为汇编
    inline bool isAsm() const                               { return m_gcnAsm; }
    // 返回是否为数据集主机
    inline bool isDatasetHost() const                       { return m_datasetHost; }
    // 返回是否有效
    inline bool isValid() const                             { return m_intensity > 0; }
    // 返回线程向量
    inline const std::vector<int64_t> &threads() const      { return m_threads; }
    // 返回 B 因子
    inline uint32_t bfactor() const                         { return m_bfactor; }
    // 返回索引
    inline uint32_t index() const                           { return m_index; }
    // 返回强度
    inline uint32_t intensity() const                       { return m_intensity; }
    // 返回内存块大小
    inline uint32_t memChunk() const                        { return m_memChunk; }
    // 返回分步索引
    inline uint32_t stridedIndex() const                    { return m_stridedIndex; }
    // 返回展开因子
    inline uint32_t unrollFactor() const                    { return m_unrollFactor; }
    // 返回工作大小
    inline uint32_t worksize() const                        { return m_worksize; }

    // 不等于运算符重载
    inline bool operator!=(const OclThread &other) const    { return !isEqual(other); }
    // 等于运算符重载
    inline bool operator==(const OclThread &other) const    { return isEqual(other); }

    // 判断是否相等
    bool isEqual(const OclThread &other) const;
    // 转换为 JSON 格式
    rapidjson::Value toJSON(rapidjson::Document &doc) const;

private:
    // 字段枚举
    enum Fields {
        STRIDED_INDEX_FIELD,
        RANDOMX_FIELDS,
        KAWPOW_FIELDS,
        FIELD_MAX
    };

    // 设置强度
    inline void setIntensity(uint32_t intensity)            { m_intensity = intensity / m_worksize * m_worksize; }

    // 数据集主机标志
    bool m_datasetHost              = false;
    // 是否为 GCN 汇编
    bool m_gcnAsm                   = true;
    // 字段位集合
    std::bitset<FIELD_MAX> m_fields = 1;
    // 线程向量
    std::vector<int64_t> m_threads;
    // B 因子
    uint32_t m_bfactor              = 6;
    // 索引
    uint32_t m_index                = 0;
    // 强度
    uint32_t m_intensity            = 0;
    // 内存块大小
    uint32_t m_memChunk             = 2;
    // 分步索引
    uint32_t m_stridedIndex         = 2;
    // 展开因子
    uint32_t m_unrollFactor         = 8;
    // 工作大小
    uint32_t m_worksize             = 0;
};


} /* namespace xmrig */
#endif /* XMRIG_OCLTHREAD_H */

这行代码是C++中的预处理指令，用于结束一个条件编译块。在这里，它表示结束对XMRIG_OCLTHREAD_H宏的条件编译。
```