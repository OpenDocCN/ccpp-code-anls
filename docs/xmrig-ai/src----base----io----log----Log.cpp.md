# `xmrig\src\base\io\log\Log.cpp`

```
/* XMRig
 * 版权所有（c）2019      Spudz76     <https://github.com/Spudz76>
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发或修改它
 *   由自由软件基金会发布，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，但没有任何保证；甚至没有暗示的保证适用于特定目的。有关更多详细信息，请参见GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifdef WIN32
#   include <winsock2.h>
#   include <windows.h>
#endif


#include <algorithm>
#include <cassert>
#include <cstring>
#include <ctime>
#include <mutex>
#include <string>
#include <uv.h>
#include <vector>


#include "base/io/log/Log.h"
#include "base/kernel/interfaces/ILogBackend.h"
#include "base/tools/Chrono.h"
#include "base/tools/Object.h"


namespace xmrig {


static const char *colors_map[] = {
    RED_BOLD_S,    // EMERG
    RED_BOLD_S,    // ALERT
    RED_BOLD_S,    // CRIT
    RED_S,         // ERR
    YELLOW_S,      // WARNING
    WHITE_BOLD_S,  // NOTICE
    nullptr,       // INFO
#   ifdef WIN32
    BLACK_BOLD_S   // DEBUG
#   else
    BRIGHT_BLACK_S // DEBUG
#   endif
};



class LogPrivate
{
public:
    XMRIG_DISABLE_COPY_MOVE(LogPrivate)


    LogPrivate() = default;


    inline ~LogPrivate()
    {
        for (auto backend : m_backends) {
            delete backend;
        }
    }


    inline void add(ILogBackend *backend) { m_backends.push_back(backend); }


    void print(Log::Level level, const char *fmt, va_list args)
    {
        // 初始化变量 size 和 offset
        size_t size   = 0;
        size_t offset = 0;

        // 使用互斥锁保护临界区
        std::lock_guard<std::mutex> lock(m_mutex);

        // 如果日志处于后台模式且后端为空，则直接返回
        if (Log::isBackground() && m_backends.empty()) {
            return;
        }

        // 获取时间戳
        const uint64_t ts = timestamp(level, size, offset);
        // 设置日志颜色
        color(level, size);

        // 格式化日志消息
        const int rc = vsnprintf(m_buf + size, sizeof (m_buf) - offset - 32, fmt, args);
        // 如果格式化失败，则直接返回
        if (rc < 0) {
            return;
        }

        // 更新 size
        size += std::min(static_cast<size_t>(rc), sizeof (m_buf) - offset - 32);
        // 添加换行符
        endl(size);

        // 创建字符串 txt，并初始化索引 i
        std::string txt(m_buf);
        size_t i = 0;
        // 移除 ANSI 转义序列
        while ((i = txt.find(CSI)) != std::string::npos) {
            txt.erase(i, txt.find('m', i) - i + 1);
        }

        // 如果后端不为空，则遍历后端并打印日志
        if (!m_backends.empty()) {
            for (auto backend : m_backends) {
                backend->print(ts, level, m_buf, offset, size, true);
                backend->print(ts, level, txt.c_str(), offset ? (offset - 11) : 0, txt.size(), false);
            }
        }
        // 如果后端为空，则将日志输出到标准输出
        else {
            fputs(txt.c_str(), stdout);
            fflush(stdout);
        }
    }
private:
    // 内联函数，用于获取时间戳，并根据日志级别设置大小和偏移量
    inline uint64_t timestamp(Log::Level level, size_t &size, size_t &offset)
    {
        // 获取当前时间的毫秒数
        const uint64_t ms = Chrono::currentMSecsSinceEpoch();

        // 如果日志级别为NONE，则直接返回时间戳
        if (level == Log::NONE) {
            return ms;
        }

        // 将毫秒数转换为秒数
        time_t now = ms / 1000;
        tm stime{};

#       ifdef _WIN32
        // Windows平台下获取本地时间
        localtime_s(&stime, &now);
#       else
        // 非Windows平台下获取本地时间
        localtime_r(&now, &stime);
#       endif

        // 格式化时间戳并写入缓冲区
        const int rc = snprintf(m_buf, sizeof(m_buf) - 1, "[%d-%02d-%02d %02d:%02d:%02d" BLACK_BOLD(".%03d") "] ",
                                stime.tm_year + 1900,
                                stime.tm_mon + 1,
                                stime.tm_mday,
                                stime.tm_hour,
                                stime.tm_min,
                                stime.tm_sec,
                                static_cast<int>(ms % 1000)
                                );

        // 如果格式化成功，则设置大小和偏移量
        if (rc > 0) {
            size = offset = static_cast<size_t>(rc);
        }

        // 返回时间戳
        return ms;
    }

    // 内联函数，根据日志级别设置颜色
    inline void color(Log::Level level, size_t &size)
    {
        // 如果日志级别为NONE，则直接返回
        if (level == Log::NONE) {
            return;
        }

        // 获取对应日志级别的颜色
        const char *color = colors_map[level];
        // 如果颜色为空，则直接返回
        if (color == nullptr) {
            return;
        }

        // 获取颜色字符串的长度，并将颜色字符串复制到缓冲区中
        const size_t s = strlen(color);
        memcpy(m_buf + size, color, s);

        // 更新大小
        size += s;
    }

    // 内联函数，添加换行符到缓冲区中
    inline void endl(size_t &size)
    {
#       ifdef _WIN32
        // Windows平台下添加回车换行符到缓冲区中
        memcpy(m_buf + size, CLEAR "\r\n", 7);
        size += 6;
#       else
        // 非Windows平台下添加换行符到缓冲区中
        memcpy(m_buf + size, CLEAR "\n", 6);
        size += 5;
#       endif
    }

    // 日志缓冲区
    char m_buf[Log::kMaxBufferSize]{};
    // 互斥锁
    std::mutex m_mutex;
    // 日志后端的容器
    std::vector<ILogBackend*> m_backends;
};

// 静态成员变量初始化
bool Log::m_background      = false;
bool Log::m_colors          = true;
LogPrivate *Log::d          = nullptr;
uint32_t Log::m_verbose     = 0;

} /* namespace xmrig */

// 添加日志后端
void xmrig::Log::add(ILogBackend *backend)
{
    // 断言日志私有对象不为空
    assert(d != nullptr);

    // 如果日志私有对象不为空，则添加日志后端
    if (d) {
        d->add(backend);
    }
}
// 销毁 Log 对象
void xmrig::Log::destroy()
{
    // 释放 Log 对象的内存
    delete d;
    // 将指针置为空
    d = nullptr;
}


// 初始化 Log 对象
void xmrig::Log::init()
{
    // 为 Log 对象分配内存
    d = new LogPrivate();
}


// 打印日志信息
void xmrig::Log::print(const char *fmt, ...)
{
    // 如果 Log 对象为空，则直接返回
    if (!d) {
        return;
    }

    // 初始化可变参数列表
    va_list args{};
    va_start(args, fmt);

    // 调用 LogPrivate 类的 print 方法打印日志信息
    d->print(NONE, fmt, args);

    // 结束可变参数列表
    va_end(args);
}


// 打印指定级别的日志信息
void xmrig::Log::print(Level level, const char *fmt, ...)
{
    // 如果 Log 对象为空，则直接返回
    if (!d) {
        return;
    }

    // 初始化可变参数列表
    va_list args{};
    va_start(args, fmt);

    // 调用 LogPrivate 类的 print 方法打印指定级别的日志信息
    d->print(level, fmt, args);

    // 结束可变参数列表
    va_end(args);
}
```