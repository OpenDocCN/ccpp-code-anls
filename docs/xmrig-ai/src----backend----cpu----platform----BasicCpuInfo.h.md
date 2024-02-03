# `xmrig\src\backend\cpu\platform\BasicCpuInfo.h`

```cpp
// XMRig程序的版权声明
// 版权声明和许可证信息

// 防止头文件重复包含
#ifndef XMRIG_BASICCPUINFO_H
#define XMRIG_BASICCPUINFO_H

// 包含ICpuInfo接口的头文件
#include "backend/cpu/interfaces/ICpuInfo.h"

// 包含bitset头文件
#include <bitset>

// 命名空间xmrig
namespace xmrig {

// BasicCpuInfo类，继承自ICpuInfo接口
class BasicCpuInfo : public ICpuInfo
{
public:
    // 构造函数
    BasicCpuInfo();

protected:
    // 返回后端信息
    const char *backend() const override;
    // 返回CPU线程信息
    CpuThreads threads(const Algorithm &algorithm, uint32_t limit) const override;
    // 将CPU信息转换为JSON格式
    rapidjson::Value toJSON(rapidjson::Document &doc) const override;

    // 返回CPU架构
    inline Arch arch() const override                           { return m_arch; }
    // 返回CPU汇编类型
    inline Assembly::Id assembly() const override               { return m_assembly; }
    // 检查CPU是否具有特定标志位
    inline bool has(Flag flag) const override                   { return m_flags.test(flag); }
    // 检查CPU是否支持AES指令集
    inline bool hasAES() const override                         { return has(FLAG_AES); }
    // 检查CPU是否支持VAES指令集
    inline bool hasVAES() const override                        { return has(FLAG_VAES); }
    // 检查CPU是否支持AVX指令集
    inline bool hasAVX() const override                         { return has(FLAG_AVX); }
    // 检查CPU是否支持AVX2指令集
    inline bool hasAVX2() const override                        { return has(FLAG_AVX2); }
    // 检查是否支持BMI2指令集，并返回结果
    inline bool hasBMI2() const override                        { return has(FLAG_BMI2); }
    // 检查是否支持CatL3指令集，并返回结果
    inline bool hasCatL3() const override                       { return has(FLAG_CAT_L3); }
    // 检查是否支持OneGbPages，并返回结果
    inline bool hasOneGbPages() const override                  { return has(FLAG_PDPE1GB); }
    // 检查是否支持XOP指令集，并返回结果
    inline bool hasXOP() const override                         { return has(FLAG_XOP); }
    // 检查是否是虚拟机，并返回结果
    inline bool isVM() const override                           { return has(FLAG_VM); }
    // 返回是否存在jccErratum
    inline bool jccErratum() const override                     { return m_jccErratum; }
    // 返回品牌信息
    inline const char *brand() const override                   { return m_brand; }
    // 返回单元信息
    inline const std::vector<int32_t> &units() const override   { return m_units; }
    // 返回MSR修改信息
    inline MsrMod msrMod() const override                       { return m_msrMod; }
    // 返回核心数量
    inline size_t cores() const override                        { return 0; }
    // 返回L2缓存大小
    inline size_t L2() const override                           { return 0; }
    // 返回L3缓存大小
    inline size_t L3() const override                           { return 0; }
    // 返回节点数量
    inline size_t nodes() const override                        { return 0; }
    // 返回处理器包数量
    inline size_t packages() const override                     { return 1; }
    // 返回线程数量
    inline size_t threads() const override                      { return m_threads; }
    // 返回供应商信息
    inline Vendor vendor() const override                       { return m_vendor; }
    // 返回模型信息
    inline uint32_t model() const override
    {
// 如果不是 XMRIG_ARM 宏定义，则返回 m_model 变量的值
        return m_model;
// 如果是 XMRIG_ARM 宏定义，则返回 0
        return 0;
    }

    // 初始化变量
    Arch m_arch             = ARCH_UNKNOWN;
    bool m_jccErratum       = false;
    char m_brand[64 + 6]{}; // 初始化长度为 70 的字符数组
    size_t m_threads        = 0; // 初始化线程数为 0
    std::vector<int32_t> m_units; // 初始化整型向量
    Vendor m_vendor         = VENDOR_UNKNOWN; // 初始化供应商为未知

private:
// 如果不是 XMRIG_ARM 宏定义，则定义以下变量
    uint32_t m_procInfo     = 0; // 初始化处理器信息为 0
    uint32_t m_family       = 0; // 初始化家族信息为 0
    uint32_t m_model        = 0; // 初始化模型信息为 0
    uint32_t m_stepping     = 0; // 初始化步进信息为 0

    Assembly m_assembly     = Assembly::NONE; // 初始化汇编为 NONE
    MsrMod m_msrMod         = MSR_MOD_NONE; // 初始化 MsrMod 为 NONE
    std::bitset<FLAG_MAX> m_flags; // 初始化位集合

}; // 结构体定义结束

} // namespace xmrig

#endif // XMRIG_BASICCPUINFO_H
```