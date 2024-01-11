# `xmrig\src\crypto\rx\RxMsr.cpp`

```
/* XMRig
 * 版权所有 (c) 2018-2019 tevador     <tevador@gmail.com>
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它，
 *   其发布由自由软件基金会发布，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，但没有任何保证；甚至没有暗示的保证适用于特定目的。
 *   有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本。
 *   如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include "crypto/rx/RxMsr.h"
#include "backend/cpu/Cpu.h"
#include "backend/cpu/CpuThread.h"
#include "base/io/log/Log.h"
#include "base/tools/Chrono.h"
#include "crypto/rx/RxConfig.h"
#include "hw/msr/Msr.h"


#include <algorithm>
#include <set>


namespace xmrig {


bool RxMsr::m_cacheQoS      = false;  // 初始化静态成员变量 m_cacheQoS 为 false
bool RxMsr::m_enabled       = false;  // 初始化静态成员变量 m_enabled 为 false
bool RxMsr::m_initialized   = false;  // 初始化静态成员变量 m_initialized 为 false


static MsrItems items;  // 创建静态变量 items


#ifdef XMRIG_OS_WIN
static constexpr inline int32_t get_cpu(int32_t)        { return -1; }  // 如果是 Windows 系统，定义内联函数 get_cpu 返回 -1
#else
static constexpr inline int32_t get_cpu(int32_t cpu)    { return cpu; }  // 如果不是 Windows 系统，定义内联函数 get_cpu 返回传入的参数 cpu
#endif


static bool wrmsr(const MsrItems &preset, const std::vector<CpuThread> &threads, bool cache_qos, bool save)
{
    auto msr = Msr::get();  // 获取 Msr 对象
    if (!msr) {  // 如果获取失败
        return false;  // 返回 false
    }
    # 如果 save 为真，则执行以下代码块
    if (save) {
        # 预先分配 preset 大小的空间
        items.reserve(preset.size());

        # 遍历 preset 中的每个元素
        for (const auto &i : preset) {
            # 从 msr 对象中读取 i.reg() 对应的数据
            auto item = msr->read(i.reg());
            # 如果读取的数据无效，则清空 items 并返回 false
            if (!item.isValid()) {
                items.clear();

                return false;
            }

            # 打印日志，输出 Msr::tag()、i.reg()、item.value()、MsrItem::maskedValue(item.value(), i.value(), i.mask()) 的值
            LOG_VERBOSE("%s " CYAN_BOLD("0x%08" PRIx32) CYAN(":0x%016" PRIx64) CYAN_BOLD(" -> 0x%016" PRIx64), Msr::tag(), i.reg(), item.value(), MsrItem::maskedValue(item.value(), i.value(), i.mask()));

            # 将 item 添加到 items 中
            items.emplace_back(item);
        }
    }

    # 定义一个存储 int32_t 类型的集合 cacheEnabled
    std::set<int32_t> cacheEnabled;
    # 定义一个布尔变量 cacheQoSDisabled，并初始化为 threads 是否为空
    bool cacheQoSDisabled = threads.empty();

    # 如果 cache_qos 为真，则执行以下代码块
    if (cache_qos) {
        # 获取 CPU 信息中的 units
        const auto &units = Cpu::info()->units();

        # 遍历 threads 中的每个元素
        for (const auto &t : threads) {
            # 将 t.affinity() 转换为 int32_t 类型，并赋值给 affinity
            const auto affinity = static_cast<int32_t>(t.affinity());

            # 如果某个线程没有设置亲和性或者亲和性设置错误，则禁用 cache QoS
            if (affinity < 0 || std::find(units.begin(), units.end(), affinity) == units.end()) {
                cacheQoSDisabled = true;

                # 打印警告日志，输出 Msr::tag() 的值
                LOG_WARN("%s " YELLOW_BOLD("cache QoS can only be enabled when all mining threads have affinity set"), Msr::tag());
                # 跳出循环
                break;
            }

            # 将 affinity 添加到 cacheEnabled 中
            cacheEnabled.insert(affinity);
        }
    }
    return msr->write([&msr, &preset, cache_qos, &cacheEnabled, cacheQoSDisabled](int32_t cpu) {
        # 使用 lambda 表达式定义一个函数，该函数接受一个 int32_t 类型的参数 cpu，并返回一个 bool 类型的值
        for (const auto &item : preset) {
            # 遍历 preset 列表中的每个元素，将其赋值给 item
            if (!msr->write(item, get_cpu(cpu))) {
                # 如果调用 msr 对象的 write 方法返回 false，则返回 false
                return false;
            }
        }

        if (!cache_qos) {
            # 如果 cache_qos 为 false，则返回 true
            return true;
        }

        // Assign Class Of Service 0 to current CPU core (default, full L3 cache available)
        if (cacheQoSDisabled || cacheEnabled.count(cpu)) {
            # 如果 cacheQoSDisabled 为 true 或者 cacheEnabled 中包含 cpu，则调用 msr 对象的 write 方法，将 0xC8F 写入到当前 CPU 核心
            return msr->write(0xC8F, 0, get_cpu(cpu));
        }

        // Disable L3 cache for Class Of Service 1
        if (!msr->write(0xC91, 0, get_cpu(cpu))) {
            # 如果调用 msr 对象的 write 方法返回 false，则继续执行下面的代码
            // Some CPUs don't let set it to all zeros
            if (!msr->write(0xC91, 1, get_cpu(cpu))) {
                # 如果调用 msr 对象的 write 方法返回 false，则返回 false
                return false;
            }
        }

        // Assign Class Of Service 1 to current CPU core
        return msr->write(0xC8F, 1ULL << 32, get_cpu(cpu));
    });
} // namespace xmrig


// 初始化函数，用于初始化 RxMsr 对象
bool xmrig::RxMsr::init(const RxConfig &config, const std::vector<CpuThread> &threads)
{
    // 如果已经初始化过，则返回是否启用的状态
    if (isInitialized()) {
        return isEnabled();
    }

    // 设置初始化标志为 true，启用标志为 false
    m_initialized = true;
    m_enabled     = false;

    // 获取配置中的 MSR 预设值
    const auto &preset = config.msrPreset();
    // 如果预设值为空，则返回 false
    if (preset.empty()) {
        return false;
    }

    // 获取当前时间戳
    const uint64_t ts = Chrono::steadyMSecs();
    // 获取配置中的 cacheQoS 值
    m_cacheQoS        = config.cacheQoS();

    // 如果 cacheQoS 为 true 且当前 CPU 不支持 cat_l3，则输出警告信息，并将 cacheQoS 设置为 false
    if (m_cacheQoS && !Cpu::info()->hasCatL3()) {
        LOG_WARN("%s " YELLOW_BOLD("this CPU doesn't support cat_l3, cache QoS is unavailable"), Msr::tag());

        m_cacheQoS = false;
    }

    // 如果成功设置了 MSR 寄存器的值，则输出成功信息
    if ((m_enabled = wrmsr(preset, threads, m_cacheQoS, config.rdmsr()))) {
        LOG_NOTICE("%s " GREEN_BOLD("register values for \"%s\" preset have been set successfully") BLACK_BOLD(" (%" PRIu64 " ms)"), Msr::tag(), config.msrPresetName(), Chrono::steadyMSecs() - ts);
    }
    // 否则输出失败信息
    else {
        LOG_ERR("%s " RED_BOLD("FAILED TO APPLY MSR MOD, HASHRATE WILL BE LOW"), Msr::tag());
    }

    // 返回是否启用的状态
    return isEnabled();
}


// 销毁函数，用于销毁 RxMsr 对象
void xmrig::RxMsr::destroy()
{
    // 如果未初始化，则直接返回
    if (!isInitialized()) {
        return;
    }

    // 设置初始化标志为 false，启用标志为 false
    m_initialized = false;
    m_enabled     = false;

    // 如果 items 列表为空，则直接返回
    if (items.empty()) {
        return;
    }

    // 获取当前时间戳
    const uint64_t ts = Chrono::steadyMSecs();

    // 如果无法恢复初始状态，则输出错误信息
    if (!wrmsr(items, std::vector<CpuThread>(), m_cacheQoS, false)) {
        LOG_ERR("%s " RED_BOLD("failed to restore initial state" BLACK_BOLD(" (%" PRIu64 " ms)")), Msr::tag(), Chrono::steadyMSecs() - ts);
    }
}
```