# `xmrig\src\Summary.cpp`

```
/* XMRig
 * 版权所有 (c) 2018-2022 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2022 XMRig       <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据 GNU 通用公共许可证的条款发布
 *   由自由软件基金会发布的许可证的第3版或
 *   （根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include <cinttypes>
#include <cstdio>
#include <uv.h>


#include "backend/cpu/Cpu.h"
#include "base/io/log/Log.h"
#include "base/net/stratum/Pool.h"
#include "core/config/Config.h"
#include "core/Controller.h"
#include "crypto/common/Assembly.h"
#include "crypto/common/VirtualMemory.h"
#include "Summary.h"
#include "version.h"


#ifdef XMRIG_FEATURE_DMI
#   include "hw/dmi/DmiReader.h"
#endif


#ifdef XMRIG_ALGO_RANDOMX
#   include "crypto/rx/RxConfig.h"
#endif


namespace xmrig {


#ifdef XMRIG_OS_WIN
static constexpr const char *kHugepagesSupported = GREEN_BOLD("permission granted");
#else
static constexpr const char *kHugepagesSupported = GREEN_BOLD("supported");
#endif


#ifdef XMRIG_FEATURE_ASM
static const char *coloredAsmNames[] = {
    RED_BOLD("none"),
    "auto",
    GREEN_BOLD("intel"),
    GREEN_BOLD("ryzen"),
    GREEN_BOLD("bulldozer")
};


inline static const char *asmName(Assembly::Id assembly)
{
    return coloredAsmNames[assembly];
}
#endif


static void print_pages(const Config *config)
{
    # 使用Log类的print方法打印信息，使用绿色加粗的"*"作为前缀，然后使用白色加粗的格式化字符串填充输出
    # 格式化字符串包括左对齐的字符串"%-13s"和另一个字符串"%s"
    # 第一个字符串是"HUGE PAGES"，表示巨大页面的信息
    # 第二个字符串是根据条件判断输出不同的内容，如果config->cpu().isHugePages()为真，则判断VirtualMemory::isHugepagesAvailable()的真假，输出不同的内容
    # 如果config->cpu().isHugePages()为假，则输出"disabled"加粗的红色字符串
    Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-13s") "%s",
               "HUGE PAGES", config->cpu().isHugePages() ? (VirtualMemory::isHugepagesAvailable() ? kHugepagesSupported : RED_BOLD("unavailable")) : RED_BOLD("disabled"));
#   ifdef XMRIG_ALGO_RANDOMX
#   ifdef XMRIG_OS_LINUX
    # 打印1GB页面的可用性，如果可用则打印是否在配置中启用，否则打印不可用
    Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-13s") "%s",
               "1GB PAGES", (VirtualMemory::isOneGbPagesAvailable() ? (config->rx().isOneGbPages() ? kHugepagesSupported : YELLOW_BOLD("disabled")) : YELLOW_BOLD("unavailable")));
#   else
    # 如果不是Linux系统，则打印1GB页面不可用
    Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-13s") "%s", "1GB PAGES", YELLOW_BOLD("unavailable"));
#   endif
#   endif
}

# 定义打印CPU信息的函数
static void print_cpu(const Config *)
{
    # 获取CPU信息
    const auto info = Cpu::info();

    # 打印CPU品牌、包数量、是否64位、是否支持AES指令集、是否虚拟机
    Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-13s%s (%zu)") " %s %sAES%s",
               "CPU",
               info->brand(),
               info->packages(),
               ICpuInfo::is64bit()    ? GREEN_BOLD("64-bit") : RED_BOLD("32-bit"),
               info->hasAES()         ? GREEN_BOLD_S : RED_BOLD_S "-",
               info->isVM()           ? RED_BOLD_S " VM" : ""
               );
#   if defined(XMRIG_FEATURE_HWLOC)
    # 如果定义了XMRIG_FEATURE_HWLOC，则打印L2缓存、L3缓存、核心数、线程数、NUMA节点数
    Log::print(WHITE_BOLD("   %-13s") BLACK_BOLD("L2:") WHITE_BOLD("%.1f MB") BLACK_BOLD(" L3:") WHITE_BOLD("%.1f MB")
               CYAN_BOLD(" %zu") "C" BLACK_BOLD("/") CYAN_BOLD("%zu") "T"
               BLACK_BOLD(" NUMA:") CYAN_BOLD("%zu"),
               "",
               info->L2() / 1048576.0,
               info->L3() / 1048576.0,
               info->cores(),
               info->threads(),
               info->nodes()
               );
#   else
    # 如果未定义XMRIG_FEATURE_HWLOC，则只打印线程数
    Log::print(WHITE_BOLD("   %-13s") BLACK_BOLD("threads:") CYAN_BOLD("%zu"), "", info->threads());
#   endif
}

# 定义打印内存信息的函数
static void print_memory(const Config *config)
{
    # 定义1GB的大小
    constexpr size_t oneGiB = 1024U * 1024U * 1024U;
    # 获取空闲内存和总内存
    const auto freeMem      = static_cast<double>(uv_get_free_memory());
    const auto totalMem     = static_cast<double>(uv_get_total_memory());

    # 计算内存使用百分比
    const double percent = freeMem > 0 ? ((totalMem - freeMem) / totalMem * 100.0) : 100.0;
    # 使用 Log 类的 print 方法打印带有颜色和格式的信息
    Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-13s") CYAN_BOLD("%.1f/%.1f") CYAN(" GB") BLACK_BOLD(" (%.0f%%)"),
               # 打印 MEMORY 字样
               "MEMORY",
               # 打印已使用内存和总内存的GB数值
               (totalMem - freeMem) / oneGiB,
               totalMem / oneGiB,
               # 打印内存使用百分比
               percent
               );
# 如果定义了 XMRIG_FEATURE_DMI
if (!config->isDMI()) {
    return;
}

# 创建 DmiReader 对象
DmiReader reader;
# 读取 DMI 数据
if (!reader.read()) {
    return;
}

# 判断是否打印空的内存信息
const bool printEmpty = reader.memory().size() <= 8;

# 遍历内存信息
for (const auto &memory : reader.memory()) {
    # 如果内存信息无效，则跳过
    if (!memory.isValid()) {
        continue;
    }

    # 如果内存大小不为0，则打印内存信息
    if (memory.size()) {
        Log::print(WHITE_BOLD("   %-13s") "%s: " CYAN_BOLD("%" PRIu64) CYAN(" GB ") WHITE_BOLD("%s @ %" PRIu64 " MHz ") BLACK_BOLD("%s"),
                   "", memory.id().data(), memory.size() / oneGiB, memory.type(), memory.speed() / 1000000ULL, memory.product().data());
    }
    # 如果内存大小为0且需要打印空的内存信息，则打印空的内存信息
    else if (printEmpty) {
        Log::print(WHITE_BOLD("   %-13s") "%s: " BLACK_BOLD("<empty>"), "", memory.id().data());
    }
}

# 获取主板信息
const auto &board = Cpu::info()->isVM() ? reader.system() : reader.board();

# 如果主板信息有效，则打印主板信息
if (board.isValid()) {
    Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-13s") WHITE_BOLD("%s") " - " WHITE_BOLD("%s"), "MOTHERBOARD", board.vendor().data(), board.product().data());
}

# 如果定义了 XMRIG_FEATURE_ASM
if (config->cpu().assembly() == Assembly::AUTO) {
    # 获取 CPU 支持的汇编类型
    const Assembly assembly = Cpu::info()->assembly();
    # 打印自动选择的汇编类型
    Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-13sauto:%s"), "ASSEMBLY", asmName(assembly));
}
else {
    # 打印指定的汇编类型
    Log::print(GREEN_BOLD(" * ") WHITE_BOLD("%-13s%s"), "ASSEMBLY", asmName(config->cpu().assembly()));
}
    # 如果日志支持颜色输出
    if (Log::isColors()) {
        # 打印带有颜色的命令提示信息
        Log::print(GREEN_BOLD(" * ") WHITE_BOLD("COMMANDS     ") MAGENTA_BG_BOLD("h") WHITE_BOLD("ashrate, ")
                                                                 MAGENTA_BG_BOLD("p") WHITE_BOLD("ause, ")
                                                                 MAGENTA_BG_BOLD("r") WHITE_BOLD("esume, ")
                                                                 WHITE_BOLD("re") MAGENTA_BG(WHITE_BOLD_S "s") WHITE_BOLD("ults, ")
                                                                 MAGENTA_BG_BOLD("c") WHITE_BOLD("onnection")
                   );
    }
    # 如果日志不支持颜色输出
    else {
        # 打印不带颜色的命令提示信息
        Log::print(" * COMMANDS     'h' hashrate, 'p' pause, 'r' resume, 's' results, 'c' connection");
    }
} // 结束 xmrig 命名空间

void xmrig::Summary::print(Controller *controller)
{
    // 获取控制器的配置信息
    const auto config = controller->config();

    // 打印版本信息
    config->printVersions();
    // 打印页面信息
    print_pages(config);
    // 打印 CPU 信息
    print_cpu(config);
    // 打印内存信息
    print_memory(config);
    // 打印线程信息
    print_threads(config);
    // 打印连接池信息
    config->pools().print();

    // 打印命令信息
    print_commands(config);
}
```