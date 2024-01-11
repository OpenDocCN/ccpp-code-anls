# `xmrig\src\backend\opencl\wrappers\OclDevice.h`

```
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。更多细节请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_OCLDEVICE_H
#define XMRIG_OCLDEVICE_H


#include "backend/common/misc/PciTopology.h"
#include "backend/opencl/wrappers/OclVendor.h"
#include "base/tools/String.h"

#include <algorithm>
#include <vector>


using cl_device_id      = struct _cl_device_id *;
using cl_platform_id    = struct _cl_platform_id *;


namespace xmrig {


class Algorithm;
class OclThreads;


class OclDevice
{
public:
    enum Type {
        Unknown,
        Baffin,
        Ellesmere,
        Polaris,
        Lexa,
        Vega_10,
        Vega_20,
        Raven,
        Navi_10,
        Navi_12,
        Navi_14,
        Navi_21
    };

    // 删除默认构造函数
    OclDevice() = delete;
    // 构造函数，初始化设备索引、设备ID和平台ID
    OclDevice(uint32_t index, cl_device_id id, cl_platform_id platform);

    // 返回可打印的设备名称
    String printableName() const;
    // 返回设备时钟频率
    uint32_t clock() const;
    // 生成算法和线程相关的内容
    void generate(const Algorithm &algorithm, OclThreads &threads) const;

    // 判断设备是否有效
    inline bool isValid() const                 { return m_id != nullptr && m_platform != nullptr; }
    // 返回设备ID
    inline cl_device_id id() const              { return m_id; }
    // 返回平台供应商
    inline const String &platformVendor() const { return m_platformVendor; }
    # 返回平台供应商的 ID
    inline OclVendor platformVendorId() const   { return m_vendorId; }
    
    # 返回 PCI 拓扑结构
    inline const PciTopology &topology() const  { return m_topology; }
    
    # 返回板卡名称，如果为空则返回设备名称
    inline const String &board() const          { return m_board.isNull() ? m_name : m_board; }
    
    # 返回设备名称
    inline const String &name() const           { return m_name; }
    
    # 返回供应商名称
    inline const String &vendor() const         { return m_vendor; }
    
    # 返回供应商 ID
    inline OclVendor vendorId() const           { return m_vendorId; }
    
    # 返回设备支持的扩展
    inline const String &extensions() const     { return m_extensions; }
    
    # 返回设备类型
    inline Type type() const                    { return m_type; }
    
    # 返回计算单元数量
    inline uint32_t computeUnits() const        { return m_computeUnits; }
    
    # 返回可用内存大小，取最小值为最大内存分配大小和全局内存大小
    inline size_t freeMemSize() const           { return std::min(maxMemAllocSize(), globalMemSize()); }
    
    # 返回全局内存大小
    inline size_t globalMemSize() const         { return m_globalMemory; }
    
    # 返回最大内存分配大小
    inline size_t maxMemAllocSize() const       { return m_maxMemoryAlloc; }
    
    # 返回设备索引
    inline uint32_t index() const               { return m_index; }
// 如果定义了 XMRIG_FEATURE_API，则声明一个函数toJSON，用于将对象转换为JSON格式
#ifdef XMRIG_FEATURE_API
    void toJSON(rapidjson::Value &out, rapidjson::Document &doc) const;
#endif

private:
    // 设备ID
    cl_device_id m_id = nullptr;
    // 平台ID
    cl_platform_id m_platform = nullptr;
    // 平台供应商
    const String m_platformVendor;
    // 设备板名称
    String m_board;
    // 设备名称
    const String m_name;
    // 设备供应商
    const String m_vendor;
    // 设备扩展
    String m_extensions;
    // 最大内存分配
    const size_t m_maxMemoryAlloc = 0;
    // 全局内存
    const size_t m_globalMemory = 0;
    // 计算单元数量
    const uint32_t m_computeUnits = 1;
    // 索引
    const uint32_t m_index = 0;
    // 平台供应商ID
    OclVendor m_platformVendorId = OCL_VENDOR_UNKNOWN;
    // 设备供应商ID
    OclVendor m_vendorId = OCL_VENDOR_UNKNOWN;
    // PCI拓扑
    PciTopology m_topology;
    // 设备类型
    Type m_type = Unknown;
};


} // namespace xmrig

#endif /* XMRIG_OCLDEVICE_H */
```