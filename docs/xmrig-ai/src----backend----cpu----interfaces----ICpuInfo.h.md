# `xmrig\src\backend\cpu\interfaces\ICpuInfo.h`

```
/* XMRig
 * 版权所有 (c) 2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2023 XMRig       <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据 GNU 通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版或
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CPUINFO_H
#define XMRIG_CPUINFO_H


#include "backend/cpu/CpuThreads.h"
#include "base/crypto/Algorithm.h"
#include "base/tools/Object.h"
#include "crypto/common/Assembly.h"


#ifdef XMRIG_FEATURE_HWLOC
using hwloc_const_bitmap_t  = const struct hwloc_bitmap_s *;
using hwloc_topology_t      = struct hwloc_topology *;
#endif


namespace xmrig {


class ICpuInfo
{
public:
    XMRIG_DISABLE_COPY_MOVE(ICpuInfo)

    enum Vendor : uint32_t {
        VENDOR_UNKNOWN,
        VENDOR_INTEL,
        VENDOR_AMD
    };

    enum Arch : uint32_t {
        ARCH_UNKNOWN,
        ARCH_ZEN,
        ARCH_ZEN_PLUS,
        ARCH_ZEN2,
        ARCH_ZEN3,
        ARCH_ZEN4
    };

    enum MsrMod : uint32_t {
        MSR_MOD_NONE,
        MSR_MOD_RYZEN_17H,
        MSR_MOD_RYZEN_19H,
        MSR_MOD_RYZEN_19H_ZEN4,
        MSR_MOD_INTEL,
        MSR_MOD_CUSTOM,
        MSR_MOD_MAX
    };

#   define MSR_NAMES_LIST "none", "ryzen_17h", "ryzen_19h", "ryzen_19h_zen4", "intel", "custom"
    # 定义一个枚举类型 Flag，使用 uint32_t 作为底层数据类型
    enum Flag : uint32_t {
        FLAG_AES,       # 表示支持 AES 指令集
        FLAG_VAES,      # 表示支持 VAES 指令集
        FLAG_AVX,       # 表示支持 AVX 指令集
        FLAG_AVX2,      # 表示支持 AVX2 指令集
        FLAG_AVX512F,   # 表示支持 AVX512F 指令集
        FLAG_BMI2,      # 表示支持 BMI2 指令集
        FLAG_OSXSAVE,   # 表示支持 OSXSAVE 指令集
        FLAG_PDPE1GB,   # 表示支持 PDPE1GB 指令集
        FLAG_SSE2,      # 表示支持 SSE2 指令集
        FLAG_SSSE3,     # 表示支持 SSSE3 指令集
        FLAG_SSE41,     # 表示支持 SSE41 指令集
        FLAG_XOP,       # 表示支持 XOP 指令集
        FLAG_POPCNT,    # 表示支持 POPCNT 指令集
        FLAG_CAT_L3,    # 表示支持 CAT_L3 指令集
        FLAG_VM,        # 表示支持 VM 指令集
        FLAG_MAX        # 表示枚举类型的最大值
    };

    # 默认构造函数，不做任何操作
    ICpuInfo()          = default;
    # 虚析构函数，不做任何操作
    virtual ~ICpuInfo() = default;
#   if defined(__x86_64__) || defined(_M_AMD64) || defined (__arm64__) || defined (__aarch64__)
    # 如果定义了__x86_64__或_M_AMD64或__arm64__或__aarch64__，则返回true，表示是64位系统
    inline constexpr static bool is64bit() { return true; }
#   else
    # 否则返回false，表示不是64位系统
    inline constexpr static bool is64bit() { return false; }
#   endif

    # 返回架构类型
    virtual Arch arch() const                                                       = 0;
    # 返回汇编类型
    virtual Assembly::Id assembly() const                                           = 0;
    # 检查是否具有指定的特性
    virtual bool has(Flag feature) const                                            = 0;
    # 检查是否具有AES指令集
    virtual bool hasAES() const                                                     = 0;
    # 检查是否具有VAES指令集
    virtual bool hasVAES() const                                                    = 0;
    # 检查是否具有AVX指令集
    virtual bool hasAVX() const                                                     = 0;
    # 检查是否具有AVX2指令集
    virtual bool hasAVX2() const                                                    = 0;
    # 检查是否具有BMI2指令集
    virtual bool hasBMI2() const                                                    = 0;
    # 检查是否具有CatL3指令集
    virtual bool hasCatL3() const                                                   = 0;
    # 检查是否具有1GB页面
    virtual bool hasOneGbPages() const                                              = 0;
    # 检查是否具有XOP指令集
    virtual bool hasXOP() const                                                     = 0;
    # 检查是否是虚拟机
    virtual bool isVM() const                                                       = 0;
    # 检查是否有jcc错误
    virtual bool jccErratum() const                                                 = 0;
    # 返回后端类型
    virtual const char *backend() const                                             = 0;
    # 返回CPU品牌
    virtual const char *brand() const                                               = 0;
    # 返回CPU核心数
    virtual const std::vector<int32_t> &units() const                               = 0;
    # 返回指定算法的线程数
    virtual CpuThreads threads(const Algorithm &algorithm, uint32_t limit) const    = 0;
    # 返回MSR修改类型
    virtual MsrMod msrMod() const                                                   = 0;
    # 将CPU信息转换为JSON格式
    virtual rapidjson::Value toJSON(rapidjson::Document &doc) const                 = 0;
    # 返回虚拟核心数量
    virtual size_t cores() const                                                    = 0;
    # 返回L2缓存大小
    virtual size_t L2() const                                                       = 0;
    # 返回L3缓存大小
    virtual size_t L3() const                                                       = 0;
    # 返回节点数量
    virtual size_t nodes() const                                                    = 0;
    # 返回处理器包数量
    virtual size_t packages() const                                                 = 0;
    # 返回线程数量
    virtual size_t threads() const                                                  = 0;
    # 返回处理器供应商
    virtual Vendor vendor() const                                                   = 0;
    # 返回处理器型号
    virtual uint32_t model() const                                                  = 0;
#   ifdef XMRIG_FEATURE_HWLOC
    // 如果定义了 XMRIG_FEATURE_HWLOC，则执行以下代码
    virtual bool membind(hwloc_const_bitmap_t nodeset)                              = 0;
    // 虚拟函数，用于将内存绑定到指定的 NUMA 节点
    virtual const std::vector<uint32_t> &nodeset() const                            = 0;
    // 虚拟函数，返回节点集合的引用
    virtual hwloc_topology_t topology() const                                       = 0;
    // 虚拟函数，返回硬件拓扑结构的引用
#   endif
};


} // namespace xmrig
// 结束 xmrig 命名空间


#endif // XMRIG_CPUINFO_H
// 结束 XMRIG_CPUINFO_H 头文件的条件编译
```