# `xmrig\src\backend\cpu\CpuConfig.h`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 要么许可证的第3版，要么
 * （在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CPUCONFIG_H
#define XMRIG_CPUCONFIG_H


#include "backend/common/Threads.h"
#include "backend/cpu/CpuLaunchData.h"
#include "backend/cpu/CpuThreads.h"
#include "crypto/common/Assembly.h"


namespace xmrig {


class CpuConfig
{
public:
    enum AesMode {
        AES_AUTO,
        AES_HW,
        AES_SOFT
    };

    static const char *kEnabled;  // 声明静态常量指针kEnabled
    static const char *kField;  // 声明静态常量指针kField
    static const char *kHugePages;  // 声明静态常量指针kHugePages
    static const char *kHugePagesJit;  // 声明静态常量指针kHugePagesJit
    static const char *kHwAes;  // 声明静态常量指针kHwAes
    static const char *kMaxThreadsHint;  // 声明静态常量指针kMaxThreadsHint
    static const char *kMemoryPool;  // 声明静态常量指针kMemoryPool
    static const char *kPriority;  // 声明静态常量指针kPriority
    static const char *kYield;  // 声明静态常量指针kYield

#   ifdef XMRIG_FEATURE_ASM
    static const char *kAsm;  // 声明静态常量指针kAsm
#   endif

#   ifdef XMRIG_ALGO_ARGON2
    static const char *kArgon2Impl;  // 声明静态常量指针kArgon2Impl
#   endif

    CpuConfig() = default;  // 默认构造函数

    bool isHwAES() const;  // 返回是否启用硬件AES加密
    rapidjson::Value toJSON(rapidjson::Document &doc) const;  // 将CpuConfig对象转换为JSON格式
    size_t memPoolSize() const;  // 返回内存池大小
    std::vector<CpuLaunchData> get(const Miner *miner, const Algorithm &algorithm) const;  // 获取CPU启动数据
    void read(const rapidjson::Value &value);  // 从JSON值中读取数据

    inline bool isEnabled() const                       { return m_enabled; }  // 内联函数，返回是否启用
    # 返回是否启用了大页内存
    inline bool isHugePages() const                     { return m_hugePageSize > 0; }
    # 返回是否启用了大页内存的即时编译
    inline bool isHugePagesJit() const                  { return m_hugePagesJit; }
    # 返回是否应该保存
    inline bool isShouldSave() const                    { return m_shouldSave; }
    # 返回是否启用了线程让步
    inline bool isYield() const                         { return m_yield; }
    # 返回程序集对象的引用
    inline const Assembly &assembly() const             { return m_assembly; }
    # 返回Argon2实现的引用
    inline const String &argon2Impl() const             { return m_argon2Impl; }
    # 返回CPU线程对象的引用
    inline const Threads<CpuThreads> &threads() const   { return m_threads; }
    # 返回优先级
    inline int priority() const                         { return m_priority; }
    # 返回大页内存的大小
    inline size_t hugePageSize() const                  { return m_hugePageSize * 1024U; }
    # 返回限制值
    inline uint32_t limit() const                       { return m_limit; }
// 默认的巨大页面大小为2048KB
constexpr static size_t kDefaultHugePageSizeKb  = 2048U;
// 1GB页面大小为1048576KB
constexpr static size_t kOneGbPageSizeKb        = 1048576U;

// 生成函数声明
void generate();
// 设置AES模式
void setAesMode(const rapidjson::Value &value);
// 设置巨大页面
void setHugePages(const rapidjson::Value &value);
// 设置内存池
void setMemoryPool(const rapidjson::Value &value);

// 设置优先级的内联函数
inline void setPriority(int priority)   { m_priority = (priority >= -1 && priority <= 5) ? priority : -1; }

// AES模式
AesMode m_aes           = AES_AUTO;
// 汇编
Assembly m_assembly;
// 是否启用
bool m_enabled          = true;
// 是否使用巨大页面的JIT
bool m_hugePagesJit     = false;
// 是否应该保存
bool m_shouldSave       = false;
// 是否让出CPU
bool m_yield            = true;
// 内存池
int m_memoryPool        = 0;
// 优先级
int m_priority          = -1;
// 巨大页面大小
size_t m_hugePageSize   = kDefaultHugePageSizeKb;
// Argon2实现
String m_argon2Impl;
// 线程
Threads<CpuThreads> m_threads;
// 限制
uint32_t m_limit        = 100;
};
```