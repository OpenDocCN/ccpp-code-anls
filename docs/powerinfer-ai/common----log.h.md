# `PowerInfer\common\log.h`

```cpp
#pragma once

#include <chrono>  // 包含时间库
#include <cstring>  // 包含字符串处理库
#include <sstream>  // 包含字符串流库
#include <iostream>  // 包含输入输出流库
#include <thread>  // 包含线程库
#include <vector>  // 包含向量库
#include <algorithm>  // 包含算法库
#include <cinttypes>  // 包含整数类型库

// --------------------------------
//
// Basic usage:
//
// --------
//
//  The LOG() and LOG_TEE() macros are ready to go by default
//   they do not require any initialization.
//
//  LOGLN() and LOG_TEELN() are variants which automatically
//   include \n character at the end of the log string.
//
//  LOG() behaves exactly like printf, by default writing to a logfile.
//  LOG_TEE() additionally, prints to the screen too ( mimics Unix tee command ).
//
//  Default logfile is named
//   "llama.<threadID>.log"
//  Default LOG_TEE() secondary output target is
//   stderr
//
//  Logs can be dynamically disabled or enabled using functions:
//   log_disable()
//  and
//   log_enable()
//
//  A log target can be changed with:
//   log_set_target( string )
//    creating and opening, or re-opening a file by string filename
//  or
//   log_set_target( FILE* )
//    allowing to point at stderr, stdout, or any valid FILE* file handler.
//
// --------
//
// End of Basic usage.
//
// --------------------------------

// Specifies a log target.
//  default uses log_handler() with "llama.log" log file
//  this can be changed, by defining LOG_TARGET
//  like so:
//
//  #define LOG_TARGET (a valid FILE*)
//  #include "log.h"
//
//  or it can be simply redirected to stdout or stderr
//  like so:
//
//  #define LOG_TARGET stderr
//  #include "log.h"
//
//  The log target can also be redirected to a diffrent function
//  like so:
//
//  #define LOG_TARGET log_handler_diffrent()
//  #include "log.h"
//
//  FILE* log_handler_diffrent()
//  {
//      return stderr;
//  }
//
//  or:
//
//  #define LOG_TARGET log_handler_another_one("somelog.log")
//  #include "log.h"
//
//  FILE* log_handler_another_one(char*filename)
//  {
//      static FILE* logfile = nullptr;
//      (...)
//      if( !logfile )
//      {
// 定义 LOG_TARGET 宏，如果未定义则使用 log_handler() 函数
#ifndef LOG_TARGET
    #define LOG_TARGET log_handler()
#endif

// 定义 LOG_TEE_TARGET 宏，如果未定义则使用 stderr
#ifndef LOG_TEE_TARGET
    #define LOG_TEE_TARGET stderr
#endif

// 枚举类型，用于同步日志配置状态
// 由于 std::optional 仅在 c++17 中引入，因此使用枚举类型来代替
enum LogTriState
{
    LogTriStateSame,
    LogTriStateFalse,
    LogTriStateTrue
};

// 获取类似于 "pid" 的唯一进程 ID，并在创建日志文件时使用
inline std::string log_get_pid()
{
   static std::string pid;
   if (pid.empty())
   {
       // 使用 std::this_thread::get_id() 获取一个 "process id"
       // 它并不同于 "pid"，但足够唯一，可以解决多个实例尝试写入相同日志的问题
       std::stringstream ss;
       ss << std::this_thread::get_id();
       pid = ss.str();
   }

   return pid;
}

// 用于生成带有基于线程 ID 的唯一 ID 的日志文件名的实用函数
// 调用 log_filename_generator("llama", "log") 会创建一个字符串 "llama.<number>.log"
// 其中的数字是当前线程的运行时 ID
#define log_filename_generator(log_file_basename, log_file_extension) log_filename_generator_impl(LogTriStateSame, log_file_basename, log_file_extension)

// 内部使用，不要使用
inline std::string log_filename_generator_impl(LogTriState multilog, const std::string & log_file_basename, const std::string & log_file_extension)
{
    static bool _multilog = false;

    if (multilog != LogTriStateSame)
    {
        _multilog = multilog == LogTriStateTrue;
    }

    std::stringstream buf;

    buf << log_file_basename;
    if (_multilog)
    {
        buf << ".";
        buf << log_get_pid();
    }
    buf << ".";
    buf << log_file_extension;

    return buf.str();
}

// 定义 LOG_DEFAULT_FILE_NAME 宏，如果未定义则使用 log_filename_generator("llama", "log") 函数
#ifndef LOG_DEFAULT_FILE_NAME
    #define LOG_DEFAULT_FILE_NAME log_filename_generator("llama", "log")
#endif
// 将 #define 值转换为字符串文字的实用程序，以便我们可以将定义为 stderr 的值打印为 "stderr"，等等。
#define LOG_STRINGIZE1(s) #s
#define LOG_STRINGIZE(s) LOG_STRINGIZE1(s)

// 将 LOG_TEE_TARGET 转换为字符串
#define LOG_TEE_TARGET_STRING LOG_STRINGIZE(LOG_TEE_TARGET)

// 允许禁用时间戳。
// 要禁用，定义 LOG_NO_TIMESTAMPS 如下所示：
//
// #define LOG_NO_TIMESTAMPS
// #include "log.h"
//
#ifndef LOG_NO_TIMESTAMPS
    #ifndef _MSC_VER
        // 定义时间戳格式
        #define LOG_TIMESTAMP_FMT "[%" PRIu64 "] "
        // 定义时间戳值
        #define LOG_TIMESTAMP_VAL , (std::chrono::duration_cast<std::chrono::duration<std::uint64_t>>(std::chrono::system_clock::now().time_since_epoch())).count()
    #else
        // 定义时间戳格式
        #define LOG_TIMESTAMP_FMT "[%" PRIu64 "] "
        // 定义时间戳值
        #define LOG_TIMESTAMP_VAL , (std::chrono::duration_cast<std::chrono::duration<std::uint64_t>>(std::chrono::system_clock::now().time_since_epoch())).count()
    #endif
#else
    // 定义无时间戳格式
    #define LOG_TIMESTAMP_FMT "%s"
    // 定义无时间戳值
    #define LOG_TIMESTAMP_VAL ,""
#endif

#ifdef LOG_TEE_TIMESTAMPS
    #ifndef _MSC_VER
        // 定义时间戳格式
        #define LOG_TEE_TIMESTAMP_FMT "[%" PRIu64 "] "
        // 定义时间戳值
        #define LOG_TEE_TIMESTAMP_VAL , (std::chrono::duration_cast<std::chrono::duration<std::uint64_t>>(std::chrono::system_clock::now().time_since_epoch())).count()
    #else
        // 定义时间戳格式
        #define LOG_TEE_TIMESTAMP_FMT "[%" PRIu64 "] "
        // 定义时间戳值
        #define LOG_TEE_TIMESTAMP_VAL , (std::chrono::duration_cast<std::chrono::duration<std::uint64_t>>(std::chrono::system_clock::now().time_since_epoch())).count()
    #endif
#else
    // 定义无时间戳格式
    #define LOG_TEE_TIMESTAMP_FMT "%s"
    // 定义无时间戳值
    #define LOG_TEE_TIMESTAMP_VAL ,""
#endif

// 允许禁用文件/行/函数前缀
// 要禁用，定义 LOG_NO_FILE_LINE_FUNCTION 如下所示：
//
// #define LOG_NO_FILE_LINE_FUNCTION
// #include "log.h"
//
#ifndef LOG_NO_FILE_LINE_FUNCTION
    #ifndef _MSC_VER
        // 定义文件/行/函数前缀格式
        #define LOG_FLF_FMT "[%24s:%5d][%24s] "
        // 定义文件/行/函数前缀值
        #define LOG_FLF_VAL , __FILE__, __LINE__, __FUNCTION__
    #else
        #define LOG_FLF_FMT "[%24s:%5ld][%24s] "  // 定义日志格式字符串，包含文件名、行号、函数名
        #define LOG_FLF_VAL , __FILE__, __LINE__, __FUNCTION__  // 定义日志格式值，包含当前文件名、行号、函数名
    #endif
// 如果没有定义 LOG_TEE_FILE_LINE_FUNCTION，则定义 LOG_FLF_FMT 为 "%s"，LOG_FLF_VAL 为空字符串
#define LOG_FLF_FMT "%s"
#define LOG_FLF_VAL ,""

// 如果定义了 LOG_TEE_FILE_LINE_FUNCTION
#ifdef LOG_TEE_FILE_LINE_FUNCTION
    // 如果不是在 Visual Studio 编译环境下，则定义 LOG_TEE_FLF_FMT 为 "[%24s:%5d][%24s] "，LOG_TEE_FLF_VAL 包含 __FILE__、__LINE__、__FUNCTION__
    #ifndef _MSC_VER
        #define LOG_TEE_FLF_FMT "[%24s:%5d][%24s] "
        #define LOG_TEE_FLF_VAL , __FILE__, __LINE__, __FUNCTION__
    // 如果是在 Visual Studio 编译环境下，则定义 LOG_TEE_FLF_FMT 为 "[%24s:%5ld][%24s] "，LOG_TEE_FLF_VAL 包含 __FILE__、__LINE__、__FUNCTION__
    #else
        #define LOG_TEE_FLF_FMT "[%24s:%5ld][%24s] "
        #define LOG_TEE_FLF_VAL , __FILE__, __LINE__, __FUNCTION__
    #endif
// 如果没有定义 LOG_TEE_FILE_LINE_FUNCTION，则定义 LOG_TEE_FLF_FMT 为 "%s"，LOG_TEE_FLF_VAL 为空字符串
#else
    #define LOG_TEE_FLF_FMT "%s"
    #define LOG_TEE_FLF_VAL ,""
#endif

// 内部使用，不要直接使用，使用 LOG() 替代
#ifndef _MSC_VER
    // 定义 LOG_IMPL 宏，使用 fprintf 将日志输出到 LOG_TARGET，包括时间戳、文件名、行号、函数名等信息
    #define LOG_IMPL(str, ...)                                                                                      \
    do {                                                                                                            \
        if (LOG_TARGET != nullptr)                                                                                  \
        {                                                                                                           \
            fprintf(LOG_TARGET, LOG_TIMESTAMP_FMT LOG_FLF_FMT str "%s" LOG_TIMESTAMP_VAL LOG_FLF_VAL, __VA_ARGS__); \
            fflush(LOG_TARGET);                                                                                     \
        }                                                                                                           \
    } while (0)
// 如果是在 Visual Studio 编译环境下，则定义 LOG_IMPL 为空
#else
    #define LOG_IMPL(str, ...)                                                                                           \
    # 使用 do-while 循环，目的是为了能够使用 if 语句和多条语句组成一个语句块
    do {                                                                                                                 \
        # 如果日志目标不为空，则执行以下操作
        if (LOG_TARGET != nullptr)                                                                                       \
        {                                                                                                                \
            # 将日志信息按照指定格式写入日志目标
            fprintf(LOG_TARGET, LOG_TIMESTAMP_FMT LOG_FLF_FMT str "%s" LOG_TIMESTAMP_VAL LOG_FLF_VAL "", ##__VA_ARGS__); \
            # 刷新日志目标，确保日志信息被写入
            fflush(LOG_TARGET);                                                                                          \
        }                                                                                                                \
    } while (0)  # 使用 do-while 循环的目的是为了能够使用 if 语句和多条语句组成一个语句块，然后使用 while(0) 来结束循环
#endif

// 内部使用，不要直接使用
// 使用 LOG_TEE() 替代
//
#ifndef _MSC_VER
    #define LOG_TEE_IMPL(str, ...)                                                                                                      \
    do {                                                                                                                                \
        // 如果 LOG_TARGET 不为空，则将日志输出到 LOG_TARGET
        if (LOG_TARGET != nullptr)                                                                                                      \
        {                                                                                                                               \
            fprintf(LOG_TARGET, LOG_TIMESTAMP_FMT LOG_FLF_FMT str "%s" LOG_TIMESTAMP_VAL LOG_FLF_VAL, __VA_ARGS__);                     \
            fflush(LOG_TARGET);                                                                                                         \
        }                                                                                                                               \
        // 如果 LOG_TARGET 不为空且不是 stdout 或 stderr，并且 LOG_TEE_TARGET 不为空，则将日志输出到 LOG_TEE_TARGET
        if (LOG_TARGET != nullptr && LOG_TARGET != stdout && LOG_TARGET != stderr && LOG_TEE_TARGET != nullptr)                         \
        {                                                                                                                               \
            fprintf(LOG_TEE_TARGET, LOG_TEE_TIMESTAMP_FMT LOG_TEE_FLF_FMT str "%s" LOG_TEE_TIMESTAMP_VAL LOG_TEE_FLF_VAL, __VA_ARGS__); \
            fflush(LOG_TEE_TARGET);                                                                                                     \
        }                                                                                                                               \
    } while (0)
#else
    // 如果是 MSC_VER 编译器，则定义为空
    #define LOG_TEE_IMPL(str, ...)                                                                                                           \
    // 使用 do-while 循环，目的是为了使用多个 if 条件语句，保证每个条件都能执行相同的代码块
    do {
        // 如果 LOG_TARGET 不为空指针，则将日志信息写入 LOG_TARGET 指向的文件中
        if (LOG_TARGET != nullptr) {
            // 使用 fprintf 将格式化的日志信息写入 LOG_TARGET 指向的文件中
            fprintf(LOG_TARGET, LOG_TIMESTAMP_FMT LOG_FLF_FMT str "%s" LOG_TIMESTAMP_VAL LOG_FLF_VAL "", ##__VA_ARGS__);
            // 刷新文件流，确保日志信息被写入文件中
            fflush(LOG_TARGET);
        }
        // 如果 LOG_TARGET 不为空指针，并且不指向标准输出或标准错误，并且 LOG_TEE_TARGET 不为空指针
        if (LOG_TARGET != nullptr && LOG_TARGET != stdout && LOG_TARGET != stderr && LOG_TEE_TARGET != nullptr) {
            // 使用 fprintf 将格式化的日志信息写入 LOG_TEE_TARGET 指向的文件中
            fprintf(LOG_TEE_TARGET, LOG_TEE_TIMESTAMP_FMT LOG_TEE_FLF_FMT str "%s" LOG_TEE_TIMESTAMP_VAL LOG_TEE_FLF_VAL "", ##__VA_ARGS__);
            // 刷新文件流，确保日志信息被写入文件中
            fflush(LOG_TEE_TARGET);
        }
    } while (0)
#endif

// 作为最后一个参数的 '\0' 是一个技巧，用于绕过愚蠢的
// "warning: ISO C++11 requires at least one argument for the "..." in a variadic macro"
// 这样我们就可以有一个单一的宏，可以像 printf 一样调用。

// 主要的 LOG 宏。
// 表现得像 printf，并且支持相同的参数。
//
#ifndef _MSC_VER
    #define LOG(...) LOG_IMPL(__VA_ARGS__, "")
#else
    #define LOG(str, ...) LOG_IMPL("%s" str, "", __VA_ARGS__, "")
#endif

// 主要的 TEE 宏。
// 做的事情和 LOG 一样
// 同时写入 stderr。
//
// 次要目标可以像 LOG_TARGET 一样改变
// 通过定义 LOG_TEE_TARGET
//
#ifndef _MSC_VER
    #define LOG_TEE(...) LOG_TEE_IMPL(__VA_ARGS__, "")
#else
    #define LOG_TEE(str, ...) LOG_TEE_IMPL("%s" str, "", __VA_ARGS__, "")
#endif

// 带自动换行的 LOG 宏变体。
#ifndef _MSC_VER
    #define LOGLN(...) LOG_IMPL(__VA_ARGS__, "\n")
    #define LOG_TEELN(...) LOG_TEE_IMPL(__VA_ARGS__, "\n")
#else
    #define LOGLN(str, ...) LOG_IMPL("%s" str, "", __VA_ARGS__, "\n")
    #define LOG_TEELN(str, ...) LOG_TEE_IMPL("%s" str, "", __VA_ARGS__, "\n")
#endif

// 内部使用，不要使用
inline FILE *log_handler1_impl(bool change = false, LogTriState append = LogTriStateSame, LogTriState disable = LogTriStateSame, const std::string & filename = LOG_DEFAULT_FILE_NAME, FILE *target = nullptr)
{
    static bool _initialized = false;
    static bool _append = false;
    static bool _disabled = filename.empty() && target == nullptr;
    static std::string log_current_filename{filename};
    static FILE *log_current_target{target};
    static FILE *logfile = nullptr;

    if (change)
    {
        // 如果追加标志不等于LogTriStateSame
        if (append != LogTriStateSame)
        {
            // 将_append设置为是否追加的布尔值，然后返回日志文件
            _append = append == LogTriStateTrue;
            return logfile;
        }
    
        // 如果禁用标志为LogTriStateTrue
        if (disable == LogTriStateTrue)
        {
            // 禁用主要目标
            _disabled = true;
        }
        // 如果之前已禁用，只能启用，并保留先前的目标
        else if (disable == LogTriStateFalse)
        {
            // 取消禁用
            _disabled = false;
        }
        // 否则，处理参数
        else if (log_current_filename != filename || log_current_target != target)
        {
            // 未初始化
            _initialized = false;
        }
    }
    
    // 如果已禁用
    if (_disabled)
    {
        // 日志已禁用
        return nullptr;
    }
    
    // 如果已初始化
    if (_initialized)
    {
        // 如果日志文件存在，则返回日志文件，否则返回标准错误输出
        return logfile ? logfile : stderr;
    }
    
    // 进行（重新）初始化
    if (target != nullptr)
    {
        // 如果日志文件存在且不是标准输出或标准错误输出，则关闭日志文件
        if (logfile != nullptr && logfile != stdout && logfile != stderr)
        {
            fclose(logfile);
        }
    
        // 设置当前文件名和目标
        log_current_filename = LOG_DEFAULT_FILE_NAME;
        log_current_target = target;
    
        // 将目标设置为日志文件
        logfile = target;
    }
    else
    {
        // 如果当前文件名不等于指定文件名
        if (log_current_filename != filename)
        {
            // 如果日志文件存在且不是标准输出或标准错误输出，则关闭日志文件
            if (logfile != nullptr && logfile != stdout && logfile != stderr)
            {
                fclose(logfile);
            }
        }
    
        // 打开指定文件名的文件，如果追加标志为真则以追加模式打开，否则以写模式打开
        logfile = fopen(filename.c_str(), _append ? "a" : "w");
    }
    
    // 如果日志文件为空
    if (!logfile)
    {
        // 将日志文件设置为标准错误输出
        logfile = stderr;
    
        // 输出打开日志文件失败的错误信息
        fprintf(stderr, "Failed to open logfile '%s' with error '%s'\n", filename.c_str(), std::strerror(errno));
        fflush(stderr);
    
        // 在这一点上，让初始化标志为真，并让目标回退到标准错误输出
        // 否则我们会重复打开已经不成功的文件
    }
    
    // 设置已初始化标志为真
    _initialized = true;
    
    // 如果日志文件存在，则返回日志文件，否则返回标准错误输出
    return logfile ? logfile : stderr;
    }
// INTERNAL, DO NOT USE
// 内部使用，不要直接调用
inline FILE *log_handler2_impl(bool change = false, LogTriState append = LogTriStateSame, LogTriState disable = LogTriStateSame, FILE *target = nullptr, const std::string & filename = LOG_DEFAULT_FILE_NAME)
{
    // 调用 log_handler1_impl 函数，返回结果
    return log_handler1_impl(change, append, disable, filename, target);
}

// Disables logs entirely at runtime.
// 在运行时完全禁用日志记录
// 使 LOG() 和 LOG_TEE() 不产生任何输出，直到重新启用
#define log_disable() log_disable_impl()

// INTERNAL, DO NOT USE
// 内部使用，不要直接调用
inline FILE *log_disable_impl()
{
    // 调用 log_handler1_impl 函数，返回结果
    return log_handler1_impl(true, LogTriStateSame, LogTriStateTrue);
}

// Enables logs at runtime.
// 在运行时启用日志记录
#define log_enable() log_enable_impl()

// INTERNAL, DO NOT USE
// 内部使用，不要直接调用
inline FILE *log_enable_impl()
{
    // 调用 log_handler1_impl 函数，返回结果
    return log_handler1_impl(true, LogTriStateSame, LogTriStateFalse);
}

// Sets target fir logs, either by a file name or FILE* pointer (stdout, stderr, or any valid FILE*)
// 设置日志记录的目标，可以是文件名或 FILE* 指针（stdout、stderr 或任何有效的 FILE*）
#define log_set_target(target) log_set_target_impl(target)

// INTERNAL, DO NOT USE
// 内部使用，不要直接调用
inline FILE *log_set_target_impl(const std::string & filename) { return log_handler1_impl(true, LogTriStateSame, LogTriStateSame, filename); }
inline FILE *log_set_target_impl(FILE *target) { return log_handler2_impl(true, LogTriStateSame, LogTriStateSame, target); }

// INTERNAL, DO NOT USE
// 内部使用，不要直接调用
inline FILE *log_handler() { return log_handler1_impl(); }

// Enable or disable creating separate log files for each run.
//  can ONLY be invoked BEFORE first log use.
// 启用或禁用为每次运行创建单独的日志文件
// 只能在第一次使用日志之前调用
#define log_multilog(enable) log_filename_generator_impl((enable) ? LogTriStateTrue : LogTriStateFalse, "", "")
// Enable or disable append mode for log file.
//  can ONLY be invoked BEFORE first log use.
// 启用或禁用日志文件的追加模式
// 只能在第一次使用日志之前调用
#define log_append(enable) log_append_impl(enable)
// INTERNAL, DO NOT USE
// 内部使用，不要直接调用
inline FILE *log_append_impl(bool enable)
{
    // 调用 log_handler1_impl 函数，返回结果
    return log_handler1_impl(true, enable ? LogTriStateTrue : LogTriStateFalse, LogTriStateSame);
}

inline void log_test()
{
    // 禁用日志记录
    log_disable();
    // 输出日志，因为日志已被禁用，所以不会产生任何输出
    LOG("01 Hello World to nobody, because logs are disabled!\n");
    // 启用日志记录
    log_enable();
}
    # 输出带参数的字符串到默认输出，参数是 LOG_TARGET 的字符串化
    LOG("02 Hello World to default output, which is \"%s\" ( Yaaay, arguments! )!\n", LOG_STRINGIZE(LOG_TARGET));
    # 输出带参数的字符串到默认输出和 LOG_TEE_TARGET_STRING
    LOG_TEE("03 Hello World to **both** default output and " LOG_TEE_TARGET_STRING "!\n");
    # 设置输出目标为 stderr
    log_set_target(stderr);
    # 输出字符串到 stderr
    LOG("04 Hello World to stderr!\n");
    # 输出字符串到 stderr，并且防止双重打印到 stderr
    LOG_TEE("05 Hello World TEE with double printing to stderr prevented!\n");
    # 设置输出目标为默认的日志文件名
    log_set_target(LOG_DEFAULT_FILE_NAME);
    # 输出字符串到默认的日志文件
    LOG("06 Hello World to default log file!\n");
    # 设置输出目标为 stdout
    log_set_target(stdout);
    # 输出字符串到 stdout
    LOG("07 Hello World to stdout!\n");
    # 设置输出目标为默认的日志文件名
    log_set_target(LOG_DEFAULT_FILE_NAME);
    # 输出字符串到默认的日志文件
    LOG("08 Hello World to default log file again!\n");
    # 禁用日志记录
    log_disable();
    # 输出字符串到虚空
    LOG("09 Hello World _1_ into the void!\n");
    # 启用日志记录
    log_enable();
    # 输出字符串到默认输出
    LOG("10 Hello World back from the void ( you should not see _1_ in the log or the output )!\n");
    # 禁用日志记录
    log_disable();
    # 设置输出目标为 "llama.anotherlog.log"
    log_set_target("llama.anotherlog.log");
    # 输出字符串到新的输出目标，但日志记录仍然被禁用
    LOG("11 Hello World _2_ to nobody, new target was selected but logs are still disabled!\n");
    # 启用日志记录
    log_enable();
    # 输出字符串到新的输出目标，但不应该在日志或输出中看到 _2_
    LOG("12 Hello World this time in a new file ( you should not see _2_ in the log or the output )?\n");
    # 设置输出目标为 "llama.yetanotherlog.log"
    log_set_target("llama.yetanotherlog.log");
    # 输出字符串到新的输出目标
    LOG("13 Hello World this time in yet new file?\n");
    # 设置输出目标为生成的日志文件名
    log_set_target(log_filename_generator("llama_autonamed", "log"));
    # 输出字符串到生成的日志文件名
    LOG("14 Hello World in log with generated filename!\n");
#ifdef _MSC_VER
    // 使用LOG_TEE宏打印带参数的日志信息
    LOG_TEE("15 Hello msvc TEE without arguments\n");
    // 使用LOG_TEE宏打印带参数的日志信息
    LOG_TEE("16 Hello msvc TEE with (%d)(%s) arguments\n", 1, "test");
    // 使用LOG_TEELN宏打印不带参数的日志信息
    LOG_TEELN("17 Hello msvc TEELN without arguments\n");
    // 使用LOG_TEELN宏打印带参数的日志信息
    LOG_TEELN("18 Hello msvc TEELN with (%d)(%s) arguments\n", 1, "test");
    // 使用LOG宏打印不带参数的日志信息
    LOG("19 Hello msvc LOG without arguments\n");
    // 使用LOG宏打印带参数的日志信息
    LOG("20 Hello msvc LOG with (%d)(%s) arguments\n", 1, "test");
    // 使用LOGLN宏打印不带参数的日志信息
    LOGLN("21 Hello msvc LOGLN without arguments\n");
    // 使用LOGLN宏打印带参数的日志信息
    LOGLN("22 Hello msvc LOGLN with (%d)(%s) arguments\n", 1, "test");
#endif
}

// 解析单个参数并执行相应操作
inline bool log_param_single_parse(const std::string & param)
{
    // 如果参数为"--log-test"，执行log_test函数并返回true
    if ( param == "--log-test")
    {
        log_test();
        return true;
    }

    // 如果参数为"--log-disable"，执行log_disable函数并返回true
    if ( param == "--log-disable")
    {
        log_disable();
        return true;
    }

    // 如果参数为"--log-enable"，执行log_enable函数并返回true
    if ( param == "--log-enable")
    {
        log_enable();
        return true;
    }

    // 如果参数为"--log-new"，执行log_multilog函数并返回true
    if (param == "--log-new")
    {
        log_multilog(true);
        return true;
    }

    // 如果参数为"--log-append"，执行log_append函数并返回true
    if (param == "--log-append")
    {
        log_append(true);
        return true;
    }

    // 如果参数不匹配以上任何情况，返回false
    return false;
}

// 解析一对参数并执行相应操作
inline bool log_param_pair_parse(bool check_but_dont_parse, const std::string & param, const std::string & next = std::string())
{
    // 如果参数为"--log-file"
    if ( param == "--log-file")
    {
        // 如果不仅检查参数但不解析，执行log_filename_generator函数生成日志文件名
        if (!check_but_dont_parse)
        {
            log_set_target(log_filename_generator(next.empty() ? "unnamed" : next, "log"));
        }

        return true;
    }

    // 如果参数不匹配以上情况，返回false
    return false;
}

// 打印日志使用说明
inline void log_print_usage()
{
    // 打印日志选项
    printf("log options:\n");
    // 打印简要说明
    printf("  --log-test            Run simple logging test\n");
    printf("  --log-disable         Disable trace logs\n");
    printf("  --log-enable          Enable trace logs\n");
    printf("  --log-file            Specify a log filename (without extension)\n");
}
    # 打印提示信息，说明在启动时创建一个新的日志文件，每个日志文件都会有唯一的名称：“<name>.<ID>.log”
    printf("  --log-new             Create a separate new log file on start. "
                                   "Each log file will have unique name: \"<name>.<ID>.log\"\n");
    # 打印提示信息，说明不要截断旧的日志文件
    printf("  --log-append          Don't truncate the old log file.\n");
// 定义宏，将 log_dump_cmdline 转发到 log_dump_cmdline_impl
#define log_dump_cmdline(argc, argv) log_dump_cmdline_impl(argc, argv)

// 内部使用，不要直接调用
// 将命令行参数数组转换为字符串并输出日志
inline void log_dump_cmdline_impl(int argc, char **argv)
{
    // 创建一个字符串流对象
    std::stringstream buf;
    // 遍历命令行参数数组
    for (int i = 0; i < argc; ++i)
    {
        // 如果参数中包含空格，则在参数两侧加上双引号
        if (std::string(argv[i]).find(' ') != std::string::npos)
        {
            buf << " \"" << argv[i] <<"\"";
        }
        else
        {
            buf << " " << argv[i];
        }
    }
    // 输出日志
    LOGLN("Cmd:%s", buf.str().c_str());
}

// 定义宏，将 log_tostr 转发到 log_var_to_string_impl
#define log_tostr(var) log_var_to_string_impl(var).c_str()

// 内部使用，将布尔类型转换为字符串
inline std::string log_var_to_string_impl(bool var)
{
    return var ? "true" : "false";
}

// 内部使用，将字符串类型转换为字符串
inline std::string log_var_to_string_impl(std::string var)
{
    return var;
}

// 内部使用，将整型向量转换为字符串
inline std::string log_var_to_string_impl(const std::vector<int> & var)
{
    std::stringstream buf;
    buf << "[ ";
    bool first = true;
    for (auto e : var)
    {
        if (first)
        {
            first = false;
        }
        else
        {
            buf << ", ";
        }
        buf << std::to_string(e);
    }
    buf << " ]";

    return buf.str();
}

// 模板函数，将 token 序列转换为字符串并输出日志
template <typename C, typename T>
inline std::string LOG_TOKENS_TOSTR_PRETTY(const C & ctx, const T & tokens)
{
    std::stringstream buf;
    buf << "[ ";

    bool first = true;
    for (const auto &token : tokens)
    {
        if (!first) {
            buf << ", ";
        } else {
            first = false;
        }

        auto detokenized = llama_token_to_piece(ctx, token);

        detokenized.erase(
            std::remove_if(
                detokenized.begin(),
                detokenized.end(),
                [](const unsigned char c) { return !std::isprint(c); }),
            detokenized.end());

        buf
            << "'" << detokenized << "'"
            << ":" << std::to_string(token);
    }
    buf << " ]";

    return buf.str();
}

// 模板函数，将 batch 转换为字符串并输出日志
template <typename C, typename B>
inline std::string LOG_BATCH_TOSTR_PRETTY(const C & ctx, const B & batch)
{
    std::stringstream buf;
    # 将字符串 "[ " 添加到缓冲区中
    buf << "[ ";

    # 初始化一个布尔变量，用于标记是否是第一个元素
    bool first = true;
    # 遍历批处理中的所有标记
    for (int i = 0; i < batch.n_tokens; ++i)
    {
        # 如果不是第一个元素，则在缓冲区中添加逗号和空格
        if (!first) {
            buf << ", ";
        } else {
            # 如果是第一个元素，则将 first 设置为 false
            first = false;
        }

        # 将标记转换为对应的字符串
        auto detokenized = llama_token_to_piece(ctx, batch.token[i]);

        # 删除不可打印字符
        detokenized.erase(
            std::remove_if(
                detokenized.begin(),
                detokenized.end(),
                [](const unsigned char c) { return !std::isprint(c); }),
            detokenized.end());

        # 将标记的信息添加到缓冲区中
        buf
            << "\n" << std::to_string(i)
            << ":token '" << detokenized << "'"
            << ":pos " << std::to_string(batch.pos[i])
            << ":n_seq_id  " << std::to_string(batch.n_seq_id[i])
            << ":seq_id " << std::to_string(batch.seq_id[i][0])
            << ":logits " << std::to_string(batch.logits[i]);
    }
    # 在缓冲区中添加 "]"
    buf << " ]";

    # 返回缓冲区中的字符串
    return buf.str();
#ifdef LOG_DISABLE_LOGS
// 如果定义了 LOG_DISABLE_LOGS，则执行以下代码

#undef LOG
// 取消 LOG 宏的定义

#define LOG(...) // dummy stub
// 用空的宏定义替换 LOG 宏

#undef LOGLN
// 取消 LOGLN 宏的定义

#define LOGLN(...) // dummy stub
// 用空的宏定义替换 LOGLN 宏

#undef LOG_TEE
// 取消 LOG_TEE 宏的定义

#define LOG_TEE(...) fprintf(stderr, __VA_ARGS__) // convert to normal fprintf
// 用 fprintf 函数替换 LOG_TEE 宏

#undef LOG_TEELN
// 取消 LOG_TEELN 宏的定义

#define LOG_TEELN(...) fprintf(stderr, __VA_ARGS__) // convert to normal fprintf
// 用 fprintf 函数替换 LOG_TEELN 宏

#undef LOG_DISABLE
// 取消 LOG_DISABLE 宏的定义

#define LOG_DISABLE() // dummy stub
// 用空的宏定义替换 LOG_DISABLE 宏

#undef LOG_ENABLE
// 取消 LOG_ENABLE 宏的定义

#define LOG_ENABLE() // dummy stub
// 用空的宏定义替换 LOG_ENABLE 宏

#undef LOG_ENABLE
// 取消 LOG_ENABLE 宏的定义

#define LOG_ENABLE() // dummy stub
// 用空的宏定义替换 LOG_ENABLE 宏

#undef LOG_SET_TARGET
// 取消 LOG_SET_TARGET 宏的定义

#define LOG_SET_TARGET(...) // dummy stub
// 用空的宏定义替换 LOG_SET_TARGET 宏

#undef LOG_DUMP_CMDLINE
// 取消 LOG_DUMP_CMDLINE 宏的定义

#define LOG_DUMP_CMDLINE(...) // dummy stub
// 用空的宏定义替换 LOG_DUMP_CMDLINE 宏

#endif // LOG_DISABLE_LOGS
// 结束条件编译指令
```