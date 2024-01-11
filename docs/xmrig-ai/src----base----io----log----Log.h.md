# `xmrig\src\base\io\log\Log.h`

```
/* XMRig
 * 版权所有（c）2019      Spudz76     <https://github.com/Spudz76>
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发或修改它
 *   该许可证由自由软件基金会发布，可以选择遵循许可证的第3版或
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，但没有任何保证；甚至没有暗示的保证
 *   商品性或适用于特定目的。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_LOG_H
#define XMRIG_LOG_H


#include <cstddef>
#include <cstdint>


namespace xmrig {


class ILogBackend;
class LogPrivate;


class Log
{
public:
    enum Level : int {
        NONE = -1,  // 无日志级别
        EMERG,      // 系统不可用
        ALERT,      // 需要立即采取行动
        CRIT,       // 严重情况
        ERR,        // 错误情况
        WARNING,    // 警告情况
        NOTICE,     // 正常但重要的情况
        INFO,       // 信息性
        DEBUG,      // 调试级别的消息
    };

    constexpr static size_t kMaxBufferSize = 16384;  // 最大缓冲区大小

    static void add(ILogBackend *backend);  // 添加日志后端
    static void destroy();  // 销毁日志
    static void init();  // 初始化日志
    static void print(const char *fmt, ...);  // 打印日志
    static void print(Level level, const char *fmt, ...);  // 打印指定级别的日志

    static inline bool isBackground()                   { return m_background; }  // 是否后台运行
    static inline bool isColors()                       { return m_colors; }  // 是否支持颜色
    static inline bool isVerbose()                      { return m_verbose > 0; }  // 是否详细输出
    # 返回当前的详细程度设置
    static inline uint32_t verbose()                    { return m_verbose; }
    # 设置程序是否在后台运行
    static inline void setBackground(bool background)   { m_background = background; }
    # 设置程序是否使用彩色输出
    static inline void setColors(bool colors)           { m_colors = colors; }
    # 设置程序的详细程度
    static inline void setVerbose(uint32_t verbose)     { m_verbose = verbose; }
// 声明私有静态成员变量，用于控制日志输出的背景颜色、文字颜色和详细程度
private:
    static bool m_background;
    static bool m_colors;
    static LogPrivate *d;
    static uint32_t m_verbose;
};

// 定义 ANSI 控制序列的宏，用于控制终端输出的颜色
#define CSI                 "\x1B["     // 控制序列引导符（ANSI 规范名称）
#define CLEAR               CSI "0m"    // 关闭所有属性
#define BRIGHT_BLACK_S      CSI "0;90m" // 亮黑色
#define BLACK_S             CSI "0;30m"

#ifdef XMRIG_OS_APPLE
#   define BLACK_BOLD_S     CSI "0;37m"
#else
#   define BLACK_BOLD_S     CSI "1;30m" // 另一种灰色
#endif

#define RED_S               CSI "0;31m"
#define RED_BOLD_S          CSI "1;31m"
#define GREEN_S             CSI "0;32m"
#define GREEN_BOLD_S        CSI "1;32m"
#define YELLOW_S            CSI "0;33m"
#define YELLOW_BOLD_S       CSI "1;33m"
#define BLUE_S              CSI "0;34m"
#define BLUE_BOLD_S         CSI "1;34m"
#define MAGENTA_S           CSI "0;35m"
#define MAGENTA_BOLD_S      CSI "1;35m"
#define CYAN_S              CSI "0;36m"
#define CYAN_BOLD_S         CSI "1;36m"
#define WHITE_S             CSI "0;37m" // 另一种浅灰色
#define WHITE_BOLD_S        CSI "1;37m" // 白色

#define RED_BG_BOLD_S       CSI "41;1m" // 红色背景
#define GREEN_BG_BOLD_S     CSI "42;1m" // 绿色背景
#define YELLOW_BG_BOLD_S    CSI "43;1m" // 黄色背景
#define BLUE_BG_S           CSI "44m"    // 蓝色背景
#define BLUE_BG_BOLD_S      CSI "44;1m" // 亮蓝色背景
#define MAGENTA_BG_S        CSI "45m"    // 洋红色背景
#define MAGENTA_BG_BOLD_S   CSI "45;1m" // 亮洋红色背景
#define CYAN_BG_S           CSI "46m"    // 青色背景
#define CYAN_BG_BOLD_S      CSI "46;1m" // 亮青色背景

// 封装颜色宏，用于在输出文本时改变颜色
#define BLACK(x)            BLACK_S x CLEAR
#define BLACK_BOLD(x)       BLACK_BOLD_S x CLEAR
#define RED(x)              RED_S x CLEAR
#define RED_BOLD(x)         RED_BOLD_S x CLEAR
#define GREEN(x)            GREEN_S x CLEAR
#define GREEN_BOLD(x)       GREEN_BOLD_S x CLEAR
#define YELLOW(x)           YELLOW_S x CLEAR
#define YELLOW_BOLD(x)      YELLOW_BOLD_S x CLEAR
#define BLUE(x)             BLUE_S x CLEAR
#define BLUE_BOLD(x)        BLUE_BOLD_S x CLEAR
#define MAGENTA(x)          MAGENTA_S x CLEAR  // 定义一个宏，将参数 x 用 MAGENTA_S 包围，并在末尾添加 CLEAR
#define MAGENTA_BOLD(x)     MAGENTA_BOLD_S x CLEAR  // 定义一个宏，将参数 x 用 MAGENTA_BOLD_S 包围，并在末尾添加 CLEAR
#define CYAN(x)             CYAN_S x CLEAR  // 定义一个宏，将参数 x 用 CYAN_S 包围，并在末尾添加 CLEAR
#define CYAN_BOLD(x)        CYAN_BOLD_S x CLEAR  // 定义一个宏，将参数 x 用 CYAN_BOLD_S 包围，并在末尾添加 CLEAR
#define WHITE(x)            WHITE_S x CLEAR  // 定义一个宏，将参数 x 用 WHITE_S 包围，并在末尾添加 CLEAR
#define WHITE_BOLD(x)       WHITE_BOLD_S x CLEAR  // 定义一个宏，将参数 x 用 WHITE_BOLD_S 包围，并在末尾添加 CLEAR

#define RED_BG_BOLD(x)      RED_BG_BOLD_S x CLEAR  // 定义一个宏，将参数 x 用 RED_BG_BOLD_S 包围，并在末尾添加 CLEAR
#define GREEN_BG_BOLD(x)    GREEN_BG_BOLD_S x CLEAR  // 定义一个宏，将参数 x 用 GREEN_BG_BOLD_S 包围，并在末尾添加 CLEAR
#define YELLOW_BG_BOLD(x)   YELLOW_BG_BOLD_S x CLEAR  // 定义一个宏，将参数 x 用 YELLOW_BG_BOLD_S 包围，并在末尾添加 CLEAR
#define BLUE_BG(x)          BLUE_BG_S x CLEAR  // 定义一个宏，将参数 x 用 BLUE_BG_S 包围，并在末尾添加 CLEAR
#define BLUE_BG_BOLD(x)     BLUE_BG_BOLD_S x CLEAR  // 定义一个宏，将参数 x 用 BLUE_BG_BOLD_S 包围，并在末尾添加 CLEAR
#define MAGENTA_BG(x)       MAGENTA_BG_S x CLEAR  // 定义一个宏，将参数 x 用 MAGENTA_BG_S 包围，并在末尾添加 CLEAR
#define MAGENTA_BG_BOLD(x)  MAGENTA_BG_BOLD_S x CLEAR  // 定义一个宏，将参数 x 用 MAGENTA_BG_BOLD_S 包围，并在末尾添加 CLEAR
#define CYAN_BG(x)          CYAN_BG_S x CLEAR  // 定义一个宏，将参数 x 用 CYAN_BG_S 包围，并在末尾添加 CLEAR
#define CYAN_BG_BOLD(x)     CYAN_BG_BOLD_S x CLEAR  // 定义一个宏，将参数 x 用 CYAN_BG_BOLD_S 包围，并在末尾添加 CLEAR

#define LOG_EMERG(x, ...)   xmrig::Log::print(xmrig::Log::EMERG,   x, ##__VA_ARGS__)  // 定义一个宏，调用 Log 类的 print 方法，传入 EMERG 级别和参数 x
#define LOG_ALERT(x, ...)   xmrig::Log::print(xmrig::Log::ALERT,   x, ##__VA_ARGS__)  // 定义一个宏，调用 Log 类的 print 方法，传入 ALERT 级别和参数 x
#define LOG_CRIT(x, ...)    xmrig::Log::print(xmrig::Log::CRIT,    x, ##__VA_ARGS__)  // 定义一个宏，调用 Log 类的 print 方法，传入 CRIT 级别和参数 x
#define LOG_ERR(x, ...)     xmrig::Log::print(xmrig::Log::ERR,     x, ##__VA_ARGS__)  // 定义一个宏，调用 Log 类的 print 方法，传入 ERR 级别和参数 x
#define LOG_WARN(x, ...)    xmrig::Log::print(xmrig::Log::WARNING, x, ##__VA_ARGS__)  // 定义一个宏，调用 Log 类的 print 方法，传入 WARNING 级别和参数 x
#define LOG_NOTICE(x, ...)  xmrig::Log::print(xmrig::Log::NOTICE,  x, ##__VA_ARGS__)  // 定义一个宏，调用 Log 类的 print 方法，传入 NOTICE 级别和参数 x
#define LOG_INFO(x, ...)    xmrig::Log::print(xmrig::Log::INFO,    x, ##__VA_ARGS__)  // 定义一个宏，调用 Log 类的 print 方法，传入 INFO 级别和参数 x
#define LOG_VERBOSE(x, ...) if (xmrig::Log::verbose() > 0) { xmrig::Log::print(xmrig::Log::INFO, x, ##__VA_ARGS__); }  // 如果 Log 的 verbose 等级大于 0，则调用 Log 类的 print 方法，传入 INFO 级别和参数 x
#define LOG_V1(x, ...)      if (xmrig::Log::verbose() > 0) { xmrig::Log::print(xmrig::Log::INFO, x, ##__VA_ARGS__); }  // 如果 Log 的 verbose 等级大于 0，则调用 Log 类的 print 方法，传入 INFO 级别和参数 x
#define LOG_V2(x, ...)      if (xmrig::Log::verbose() > 1) { xmrig::Log::print(xmrig::Log::INFO, x, ##__VA_ARGS__); }  // 如果 Log 的 verbose 等级大于 1，则调用 Log 类的 print 方法，传入 INFO 级别和参数 x
#define LOG_V3(x, ...)      if (xmrig::Log::verbose() > 2) { xmrig::Log::print(xmrig::Log::INFO, x, ##__VA_ARGS__); }  // 如果 Log 的 verbose 等级大于 2，则调用 Log 类的 print 方法，传入 INFO 级别和参数 x
#define LOG_V4(x, ...)      if (xmrig::Log::verbose() > 3) { xmrig::Log::print(xmrig::Log::INFO, x, ##__VA_ARGS__); }  // 如果 Log 的 verbose 等级大于 3，则调用 Log 类的 print 方法，传入 INFO 级别和参数 x
// 定义一个宏，如果日志的详细程度大于4，则打印信息
#define LOG_V5(x, ...)      if (xmrig::Log::verbose() > 4) { xmrig::Log::print(xmrig::Log::INFO, x, ##__VA_ARGS__); }

#ifdef APP_DEBUG
// 如果定义了APP_DEBUG，则定义LOG_DEBUG宏为打印DEBUG级别的日志
#   define LOG_DEBUG(x, ...) xmrig::Log::print(xmrig::Log::DEBUG, x, ##__VA_ARGS__)
#else
// 否则定义LOG_DEBUG宏为空
#   define LOG_DEBUG(x, ...)
#endif

#if defined(APP_DEBUG) || defined(APP_DEVEL)
// 如果定义了APP_DEBUG或APP_DEVEL，则定义LOG_DEBUG_ERR宏为打印ERR级别的日志，定义LOG_DEBUG_WARN宏为打印WARNING级别的日志
#   define LOG_DEBUG_ERR(x, ...)  xmrig::Log::print(xmrig::Log::ERR,     x, ##__VA_ARGS__)
#   define LOG_DEBUG_WARN(x, ...) xmrig::Log::print(xmrig::Log::WARNING, x, ##__VA_ARGS__)
#else
// 否则定义LOG_DEBUG_ERR和LOG_DEBUG_WARN宏为空
#   define LOG_DEBUG_ERR(x, ...)
#   define LOG_DEBUG_WARN(x, ...)
#endif

} /* namespace xmrig */

// 结束xmrig命名空间

#endif /* XMRIG_LOG_H */
// 结束条件编译，结束文件```
```