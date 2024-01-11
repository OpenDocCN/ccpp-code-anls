# `xmrig\src\backend\cuda\CudaThread.h`

```
/* XMRig
 * 版权所有（C）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（C）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，
 *   但没有任何保证；甚至没有暗示的保证适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CUDATHREAD_H
#define XMRIG_CUDATHREAD_H


using nvid_ctx = struct nvid_ctx;


#include "3rdparty/rapidjson/fwd.h"


namespace xmrig {


class CudaThread
{
public:
    // 删除默认构造函数
    CudaThread() = delete;
    // 从rapidjson::Value对象构造CudaThread对象
    CudaThread(const rapidjson::Value &value);
    // 使用索引和nvid_ctx指针构造CudaThread对象
    CudaThread(uint32_t index, nvid_ctx *ctx);

    // 检查CudaThread对象是否有效
    inline bool isValid() const                              { return m_blocks > 0 && m_threads > 0; }
    // 返回bfactor值
    inline int32_t bfactor() const                           { return static_cast<int32_t>(m_bfactor); }
    // 返回blocks值
    inline int32_t blocks() const                            { return m_blocks; }
    // 返回bsleep值
    inline int32_t bsleep() const                            { return static_cast<int32_t>(m_bsleep); }
    // 返回datasetHost值
    inline int32_t datasetHost() const                       { return m_datasetHost; }
    // 返回threads值
    inline int32_t threads() const                           { return m_threads; }
    // 返回affinity值
    inline int64_t affinity() const                          { return m_affinity; }
    // 返回index值
    inline uint32_t index() const                            { return m_index; }

    // 重载不等于运算符
    inline bool operator!=(const CudaThread &other) const    { return !isEqual(other); }
    # 定义重载运算符，判断两个CudaThread对象是否相等
    inline bool operator==(const CudaThread &other) const    { return isEqual(other); }

    # 判断当前CudaThread对象是否与另一个CudaThread对象相等
    bool isEqual(const CudaThread &other) const;

    # 将当前CudaThread对象转换为JSON格式，存入rapidjson::Document对象中
    rapidjson::Value toJSON(rapidjson::Document &doc) const;
# 定义私有成员变量，用于存储块数、数据集主机、线程数、亲和性和索引
private:
    int32_t m_blocks        = 0;  // 存储块数，默认为0
    int32_t m_datasetHost   = -1; // 存储数据集主机，默认为-1
    int32_t m_threads       = 0;  // 存储线程数，默认为0
    int64_t m_affinity      = -1; // 存储亲和性，默认为-1
    uint32_t m_index        = 0;  // 存储索引，默认为0

#   ifdef _WIN32
    uint32_t m_bfactor      = 6;  // 如果是Windows系统，存储bfactor，默认为6
    uint32_t m_bsleep       = 25; // 如果是Windows系统，存储bsleep，默认为25
#   else
    uint32_t m_bfactor      = 0;  // 如果不是Windows系统，存储bfactor，默认为0
    uint32_t m_bsleep       = 0;  // 如果不是Windows系统，存储bsleep，默认为0
#   endif
};


} /* namespace xmrig */


#endif /* XMRIG_CUDATHREAD_H */
```