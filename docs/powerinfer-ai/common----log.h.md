# `PowerInfer\common\log.h`

```
// 防止头文件被多次包含
#pragma once

// 包含必要的头文件
#include <chrono>       // 用于时间相关操作
#include <cstring>      // 用于字符串操作
#include <sstream>      // 用于字符串流操作
#include <iostream>     // 用于标准输入输出
#include <thread>       // 用于多线程操作
#include <vector>       // 用于向量操作
#include <algorithm>    // 用于算法操作
#include <cinttypes>    // 用于整数类型操作

// --------------------------------
//
// Basic usage:
//
// --------
//
//  The LOG() and LOG_TEE() macros are ready to go by default
//   they do not require any initialization.
//
// 以下是基本用法的说明：
// LOG() 和 LOG_TEE() 宏默认情况下即可使用
// 它们不需要任何初始化。
// LOGLN()和LOG_TEELN()是自动在日志字符串末尾包含\n字符的变体。
//
// LOG()的行为与printf完全相同，默认写入日志文件。
// LOG_TEE()另外还会打印到屏幕上（模仿Unix的tee命令）。
//
// 默认日志文件名为
// "llama.<threadID>.log"
// 默认LOG_TEE()的次要输出目标为
// stderr
//
// 可以使用以下函数动态禁用或启用日志：
// log_disable()
// 和
// log_enable()
//
// 可以使用以下函数更改日志目标：
// log_set_target( string )
// 通过字符串文件名创建和打开，或重新打开文件
// 或
// 设置日志输出目标，可以指向 stderr、stdout 或任何有效的 FILE* 文件处理器。
//
// --------
//
// 基本用法结束。
//
// --------------------------------

// 指定日志输出目标。
// 默认情况下使用 log_handler() 写入 "llama.log" 日志文件
// 可以通过定义 LOG_TARGET 来更改日志输出目标
// 例如：
//
// #define LOG_TARGET (一个有效的 FILE*)
// #include "log.h"
//
// 或者可以简单地重定向到 stdout 或 stderr
// 例如：
//
// 定义日志输出目标为标准错误输出流
// 包含日志头文件
//
// 也可以将日志输出目标重定向到不同的函数，例如：
//
// 定义日志输出目标为 log_handler_diffrent() 函数
// 包含日志头文件
//
// 定义 log_handler_diffrent() 函数，返回标准错误输出流
//
// 或者：
//
// 定义日志输出目标为 log_handler_another_one("somelog.log") 函数
// 包含日志头文件
//
// 定义 log_handler_another_one() 函数，接受文件名参数
// 定义静态的日志文件指针，默认为nullptr
static FILE* logfile = nullptr;
// 如果日志文件指针为空
if( !logfile )
{
    // 打开日志文件
    fopen(...)
}
// 返回日志文件指针
return logfile

// 如果未定义LOG_TARGET，则使用log_handler()作为日志目标
#ifndef LOG_TARGET
    #define LOG_TARGET log_handler()
#endif

// 如果未定义LOG_TEE_TARGET，则使用stderr作为日志复制目标
#ifndef LOG_TEE_TARGET
    #define LOG_TEE_TARGET stderr
#endif

// 用于同步日志配置状态的实用程序
// 由于 std::optional 只在 c++17 中引入，所以使用枚举 LogTriState 代替
enum LogTriState
{
    LogTriStateSame,  // 表示状态相同
    LogTriStateFalse,  // 表示状态为假
    LogTriStateTrue  // 表示状态为真
};

// 获取类似于 "pid" 的唯一进程 ID，并在创建日志文件时使用
inline std::string log_get_pid()
{
   static std::string pid;  // 静态变量，用于保存进程 ID
   if (pid.empty())  // 如果进程 ID 为空
   {
       // std::this_thread::get_id() 是获取 "进程 ID" 最通用的方法
       // 它并不同于 "pid"，但足够唯一，可以解决多个实例尝试写入同一日志的问题
       std::stringstream ss;  // 创建字符串流
       ss << std::this_thread::get_id();  // 将当前线程 ID 写入字符串流
       pid = ss.str();  // 将字符串流转换为字符串并保存为进程 ID
// 生成具有基于线程ID的唯一ID的日志文件名的实用函数
// 调用 log_filename_generator("llama", "log") 会创建一个字符串 "llama.<number>.log"，其中 number 是当前线程的运行时ID
// 使用宏定义 log_filename_generator(log_file_basename, log_file_extension) 调用 log_filename_generator_impl(LogTriStateSame, log_file_basename, log_file_extension)

// 内部函数，不要使用
// 生成日志文件名的内部函数实现
inline std::string log_filename_generator_impl(LogTriState multilog, const std::string & log_file_basename, const std::string & log_file_extension)
{
    static bool _multilog = false;

    // 如果 multilog 不等于 LogTriStateSame，则将 _multilog 设置为 multilog 是否为 LogTriStateTrue
    if (multilog != LogTriStateSame)
    {
        _multilog = multilog == LogTriStateTrue;
    }
// 创建一个字符串流对象
std::stringstream buf;

// 将 log_file_basename 添加到字符串流中
buf << log_file_basename;

// 如果 _multilog 为真，则在字符串流中添加一个点和进程 ID
if (_multilog)
{
    buf << ".";
    buf << log_get_pid();
}

// 在字符串流中添加一个点和 log_file_extension
buf << ".";
buf << log_file_extension;

// 返回字符串流中的内容作为字符串
return buf.str();
}

// 如果 LOG_DEFAULT_FILE_NAME 未定义，则定义为调用 log_filename_generator 函数生成的默认文件名
#ifndef LOG_DEFAULT_FILE_NAME
    #define LOG_DEFAULT_FILE_NAME log_filename_generator("llama", "log")
#endif

// 用于将 #define 值转换为字符串文字的实用程序
// 定义 LOG_STRINGIZE1 宏，将参数 s 转换为字符串
#define LOG_STRINGIZE1(s) #s
// 定义 LOG_STRINGIZE 宏，将参数 s 转换为字符串
#define LOG_STRINGIZE(s) LOG_STRINGIZE1(s)

// 定义 LOG_TEE_TARGET_STRING 宏，将 LOG_TEE_TARGET 转换为字符串
#define LOG_TEE_TARGET_STRING LOG_STRINGIZE(LOG_TEE_TARGET)

// 允许禁用时间戳
// 如果要禁用，定义 LOG_NO_TIMESTAMPS
// 例如：
// #define LOG_NO_TIMESTAMPS
// #include "log.h"
#ifndef LOG_NO_TIMESTAMPS
    #ifndef _MSC_VER
        // 定义 LOG_TIMESTAMP_FMT 宏，时间戳格式
        #define LOG_TIMESTAMP_FMT "[%" PRIu64 "] "
        // 定义 LOG_TIMESTAMP_VAL 宏，时间戳值
        #define LOG_TIMESTAMP_VAL , (std::chrono::duration_cast<std::chrono::duration<std::uint64_t>>(std::chrono::system_clock::now().time_since_epoch())).count()
    #else
        // 定义 LOG_TIMESTAMP_FMT 宏，时间戳格式
        #define LOG_TIMESTAMP_FMT "[%" PRIu64 "] "
#ifdef LOG_TIMESTAMP
    // 如果定义了LOG_TIMESTAMP，则定义时间戳格式为64位整数，并获取当前时间戳
    #ifndef _MSC_VER
        #define LOG_TIMESTAMP_FMT "[%" PRIu64 "] "
        #define LOG_TIMESTAMP_VAL , (std::chrono::duration_cast<std::chrono::duration<std::uint64_t>>(std::chrono::system_clock::now().time_since_epoch())).count()
    #else
        #define LOG_TIMESTAMP_FMT "[%" PRIu64 "] "
        #define LOG_TIMESTAMP_VAL , (std::chrono::duration_cast<std::chrono::duration<std::uint64_t>>(std::chrono::system_clock::now().time_since_epoch())).count()
    #endif
#else
    // 如果未定义LOG_TIMESTAMP，则时间戳格式为空字符串
    #define LOG_TIMESTAMP_FMT "%s"
    #define LOG_TIMESTAMP_VAL ,""
#endif

#ifdef LOG_TEE_TIMESTAMPS
    // 如果定义了LOG_TEE_TIMESTAMPS，则定义时间戳格式为64位整数，并获取当前时间戳
    #ifndef _MSC_VER
        #define LOG_TEE_TIMESTAMP_FMT "[%" PRIu64 "] "
        #define LOG_TEE_TIMESTAMP_VAL , (std::chrono::duration_cast<std::chrono::duration<std::uint64_t>>(std::chrono::system_clock::now().time_since_epoch())).count()
    #else
        #define LOG_TEE_TIMESTAMP_FMT "[%" PRIu64 "] "
        #define LOG_TEE_TIMESTAMP_VAL , (std::chrono::duration_cast<std::chrono::duration<std::uint64_t>>(std::chrono::system_clock::now().time_since_epoch())).count()
    #endif
#else
    // 如果未定义LOG_TEE_TIMESTAMPS，则时间戳格式为空字符串
    #define LOG_TEE_TIMESTAMP_FMT "%s"
    #define LOG_TEE_TIMESTAMP_VAL ,""
#endif
// 允许禁用文件/行/函数前缀
// 要禁用，定义LOG_NO_FILE_LINE_FUNCTION如下：
//
// #define LOG_NO_FILE_LINE_FUNCTION
// #include "log.h"
//
#ifndef LOG_NO_FILE_LINE_FUNCTION
    #ifndef _MSC_VER
        // 定义带有文件名、行号、函数名的日志格式
        #define LOG_FLF_FMT "[%24s:%5d][%24s] "
        // 定义文件名、行号、函数名的值
        #define LOG_FLF_VAL , __FILE__, __LINE__, __FUNCTION__
    #else
        // 定义带有文件名、行号、函数名的日志格式
        #define LOG_FLF_FMT "[%24s:%5ld][%24s] "
        // 定义文件名、行号、函数名的值
        #define LOG_FLF_VAL , __FILE__, __LINE__, __FUNCTION__
    #endif
// 如果禁用了文件/行/函数前缀，则使用简单的日志格式
#else
    #define LOG_FLF_FMT "%s"
    // 空值
    #define LOG_FLF_VAL ,""
#endif
#ifdef LOG_TEE_FILE_LINE_FUNCTION
    // 如果定义了 LOG_TEE_FILE_LINE_FUNCTION，则进行以下操作
    #ifndef _MSC_VER
        // 如果不是 MSC 编译器，则定义 LOG_TEE_FLF_FMT 和 LOG_TEE_FLF_VAL
        #define LOG_TEE_FLF_FMT "[%24s:%5d][%24s] "
        #define LOG_TEE_FLF_VAL , __FILE__, __LINE__, __FUNCTION
    #else
        // 如果是 MSC 编译器，则定义 LOG_TEE_FLF_FMT 和 LOG_TEE_FLF_VAL
        #define LOG_TEE_FLF_FMT "[%24s:%5ld][%24s] "
        #define LOG_TEE_FLF_VAL , __FILE__, __LINE__, __FUNCTION
    #endif
#else
    // 如果未定义 LOG_TEE_FILE_LINE_FUNCTION，则定义 LOG_TEE_FLF_FMT 和 LOG_TEE_FLF_VAL
    #define LOG_TEE_FLF_FMT "%s"
    #define LOG_TEE_FLF_VAL ,""
#endif

// INTERNAL, DO NOT USE
//  USE LOG() INSTEAD
//
#ifndef _MSC_VER
    // 如果不是 MSC 编译器，则定义 LOG_IMPL 宏
    #define LOG_IMPL(str, ...)                                                                                      \
    do {                                                                                                            \
        // 如果 LOG_TARGET 不为空，则执行以下操作
        if (LOG_TARGET != nullptr)                                                                                  \
// 如果不是在 Windows 平台下编译，则定义 LOG_IMPL 宏
#ifndef _MSC_VER
// 定义 LOG_IMPL 宏，用于输出日志信息
#define LOG_IMPL(str, ...)                                                                                           
// do-while 循环，用于执行日志输出操作
do {                                                                                                                 
    // 如果日志输出目标不为空
    if (LOG_TARGET != nullptr)                                                                                       
    {                                                                                                                
        // 格式化输出日志信息到指定目标
        fprintf(LOG_TARGET, LOG_TIMESTAMP_FMT LOG_FLF_FMT str "%s" LOG_TIMESTAMP_VAL LOG_FLF_VAL "", ##__VA_ARGS__); 
        // 刷新输出缓冲区
        fflush(LOG_TARGET);                                                                                          
    }                                                                                                                
} while (0)
// 如果在 Windows 平台下编译，则定义 LOG_IMPL 宏
#else
// 定义 LOG_IMPL 宏，用于输出日志信息
#define LOG_IMPL(str, ...)                                                                                           
// do-while 循环，用于执行日志输出操作
do {                                                                                                                 
    // 如果日志输出目标不为空
    if (LOG_TARGET != nullptr)                                                                                       
    {                                                                                                                
        // 格式化输出日志信息到指定目标
        fprintf(LOG_TARGET, LOG_TIMESTAMP_FMT LOG_FLF_FMT str "%s" LOG_TIMESTAMP_VAL LOG_FLF_VAL, __VA_ARGS__); 
        // 刷新输出缓冲区
        fflush(LOG_TARGET);                                                                                          
    }                                                                                                                
} while (0)
#endif
// 内部使用，不要直接使用，使用 LOG_TEE() 替代
// 如果不是在 Windows 平台下编译
#ifndef _MSC_VER
    # 定义日志输出的宏，带有可变参数
    # 如果 LOG_TARGET 不为空，则将日志输出到 LOG_TARGET
    # 如果 LOG_TARGET 不为空且不是 stdout 或 stderr，并且 LOG_TEE_TARGET 不为空，则将日志输出到 LOG_TEE_TARGET
    # 如果未定义，则将日志输出到 LOG_TARGET
    # 最后将日志内容刷新到输出流
// 如果日志目标不为空且不是标准输出或标准错误，并且日志复制目标不为空
// 则将日志输出到日志复制目标，并刷新
// 这里使用了宏定义的可变参数，类似于printf函数
// 通过do-while(0)的技巧来确保宏定义的完整性
// 这里的\是用来连接多行代码的
} while (0)
#endif

// 最后一个参数'\0'是为了绕过ISO C++11对可变宏的警告
// 使得宏可以像printf一样被调用

// 主要的LOG宏
// 行为类似于printf，并且支持相同的参数
// 在不同的编译器下有不同的实现
#ifndef _MSC_VER
    #define LOG(...) LOG_IMPL(__VA_ARGS__, "")
#else
    #define LOG(str, ...) LOG_IMPL("%s" str, "", __VA_ARGS__, "")
// 结束符号
#endif

// 主要的 TEE 宏
// 与 LOG 相同
// 同时写入 stderr
//
// 可以像 LOG_TARGET 一样更改次要目标
// 通过定义 LOG_TEE_TARGET
//
#ifndef _MSC_VER
    #define LOG_TEE(...) LOG_TEE_IMPL(__VA_ARGS__, "")
#else
    #define LOG_TEE(str, ...) LOG_TEE_IMPL("%s" str, "", __VA_ARGS__, "")
#endif

// 自动换行的 LOG 宏变体
#ifndef _MSC_VER
    #define LOGLN(...) LOG_IMPL(__VA_ARGS__, "\n")
    #define LOG_TEELN(...) LOG_TEE_IMPL(__VA_ARGS__, "\n")
// 如果条件不成立，则定义 LOGLN 和 LOG_TEELN 宏
#else
    #define LOGLN(str, ...) LOG_IMPL("%s" str, "", __VA_ARGS__, "\n")
    #define LOG_TEELN(str, ...) LOG_TEE_IMPL("%s" str, "", __VA_ARGS__, "\n")
#endif

// 内部使用，不要直接使用
{
    // 静态变量，用于标记是否已经初始化
    static bool _initialized = false;
    // 静态变量，用于标记是否追加写入
    static bool _append = false;
    // 静态变量，用于标记是否禁用日志
    static bool _disabled = filename.empty() && target == nullptr;
    // 静态变量，保存当前日志文件名
    static std::string log_current_filename{filename};
    // 静态变量，保存当前日志目标
    static FILE *log_current_target{target};
    // 静态变量，保存日志文件指针
    static FILE *logfile = nullptr;

    // 如果发生改变
    if (change)
    {
        // 如果追加标志不等于 LogTriStateSame
        if (append != LogTriStateSame)
        {
            // 更新追加标志
            _append = append == LogTriStateTrue;
            // 返回日志文件指针
            return logfile;
        }

        // 如果禁用状态为真
        if (disable == LogTriStateTrue)
        {
            // 禁用主要目标
            _disabled = true;
        }
        // 如果先前已禁用，只能启用，并保留先前的目标
        else if (disable == LogTriStateFalse)
        {
            // 取消禁用
            _disabled = false;
        }
        // 否则，处理参数
        else if (log_current_filename != filename || log_current_target != target)
        {
            // 重置初始化状态
            _initialized = false;
        }
    }

    // 如果禁用状态为真
    if (_disabled)
    {
        // 如果日志被禁用，则返回空指针
        return nullptr;
    }

    if (_initialized)
    {
        // 如果已经初始化，则返回日志文件，如果日志文件为空，则返回标准错误输出流
        return logfile ? logfile : stderr;
    }

    // 进行（重新）初始化
    if (target != nullptr)
    {
        // 如果日志文件不为空且不是标准输出流或标准错误输出流，则关闭日志文件
        if (logfile != nullptr && logfile != stdout && logfile != stderr)
        {
            fclose(logfile);
        }

        // 设置当前日志文件名为默认文件名
        log_current_filename = LOG_DEFAULT_FILE_NAME;
        // 将当前目标赋值给日志当前目标
        log_current_target = target;

        // 将目标赋值给日志文件
        logfile = target;
    }
    else
    {
        // 如果当前日志文件名不等于传入的文件名
        if (log_current_filename != filename)
        {
            // 如果日志文件不为空且不等于标准输出和标准错误输出
            if (logfile != nullptr && logfile != stdout && logfile != stderr)
            {
                // 关闭日志文件
                fclose(logfile);
            }
        }

        // 打开文件名对应的文件，以追加或写入模式
        logfile = fopen(filename.c_str(), _append ? "a" : "w");
    }

    // 如果日志文件为空
    if (!logfile)
    {
        // 验证文件是否被打开，否则回退到标准错误输出
// 将标准错误流赋值给logfile变量
logfile = stderr;

// 使用fprintf函数将错误信息输出到标准错误流，包括文件名和错误信息
fprintf(stderr, "Failed to open logfile '%s' with error '%s'\n", filename.c_str(), std::strerror(errno));
// 刷新标准错误流
fflush(stderr);

// 在这一点上，让init标志为true，并让目标回退到标准错误流，否则我们会重复打开已经不成功的文件
// 在这里我们让init标志为true，目标回退到标准错误流，否则我们会重复打开已经不成功的文件

// 将_initialized标志设置为true
_initialized = true;

// 如果logfile存在，则返回logfile，否则返回标准错误流
return logfile ? logfile : stderr;
}

// 内部函数，不要使用
{
    // 调用log_handler1_impl函数，并返回其结果
    return log_handler1_impl(change, append, disable, filename, target);
}

// 在运行时完全禁用日志
// 使LOG()和LOG_TEE()不产生输出，直到重新启用。
#define log_disable() log_disable_impl()

// 内部使用，不要使用
inline FILE *log_disable_impl()
{
    return log_handler1_impl(true, LogTriStateSame, LogTriStateTrue);
}

// 在运行时启用日志。
#define log_enable() log_enable_impl()

// 内部使用，不要使用
inline FILE *log_enable_impl()
{
    return log_handler1_impl(true, LogTriStateSame, LogTriStateFalse);
}

// 设置日志的目标，可以是文件名或FILE*指针（stdout、stderr或任何有效的FILE*）
// 定义宏，将 log_set_target(target) 转发到 log_set_target_impl(target)
#define log_set_target(target) log_set_target_impl(target)

// 内部使用，不要直接调用
// 使用文件名设置日志目标
inline FILE *log_set_target_impl(const std::string & filename) { return log_handler1_impl(true, LogTriStateSame, LogTriStateSame, filename); }
// 内部使用，不要直接调用
// 使用文件指针设置日志目标
inline FILE *log_set_target_impl(FILE *target) { return log_handler2_impl(true, LogTriStateSame, LogTriStateSame, target); }

// 内部使用，不要直接调用
// 获取日志处理器
inline FILE *log_handler() { return log_handler1_impl(); }

// 启用或禁用为每次运行创建单独的日志文件
// 只能在第一次使用日志之前调用
#define log_multilog(enable) log_filename_generator_impl((enable) ? LogTriStateTrue : LogTriStateFalse, "", "")
// 启用或禁用日志文件的追加模式
// 只能在第一次使用日志之前调用
#define log_append(enable) log_append_impl(enable)
// 内部使用，不要直接调用
// 启用或禁用日志文件的追加模式
inline FILE *log_append_impl(bool enable)
{
    return log_handler1_impl(true, enable ? LogTriStateTrue : LogTriStateFalse, LogTriStateSame);
}
// 定义一个内联函数 log_test
inline void log_test()
{
    // 禁用日志记录
    log_disable();
    // 记录日志到无处，因为日志已被禁用
    LOG("01 Hello World to nobody, because logs are disabled!\n");
    // 启用日志记录
    log_enable();
    // 记录日志到默认输出，使用 LOG_STRINGIZE 宏将 LOG_TARGET 转换为字符串
    LOG("02 Hello World to default output, which is \"%s\" ( Yaaay, arguments! )!\n", LOG_STRINGIZE(LOG_TARGET));
    // 记录日志到默认输出和 LOG_TEE_TARGET_STRING 所指定的位置
    LOG_TEE("03 Hello World to **both** default output and " LOG_TEE_TARGET_STRING "!\n");
    // 将日志输出目标设置为 stderr
    log_set_target(stderr);
    // 记录日志到 stderr
    LOG("04 Hello World to stderr!\n");
    // 记录日志到默认输出和 stderr，但防止重复打印到 stderr
    LOG_TEE("05 Hello World TEE with double printing to stderr prevented!\n");
    // 将日志输出目标设置为默认的日志文件名
    log_set_target(LOG_DEFAULT_FILE_NAME);
    // 记录日志到默认的日志文件
    LOG("06 Hello World to default log file!\n");
    // 将日志输出目标设置为 stdout
    log_set_target(stdout);
    // 记录日志到 stdout
    LOG("07 Hello World to stdout!\n");
    // 将日志输出目标设置为默认的日志文件名
    log_set_target(LOG_DEFAULT_FILE_NAME);
    // 记录日志到默认的日志文件
    LOG("08 Hello World to default log file again!\n");
    // 禁用日志记录
    log_disable();
    // 记录日志到虚空
    LOG("09 Hello World _1_ into the void!\n");
    // 启用日志记录
    log_enable();
}
// 输出日志信息，包括 "Hello World back from the void ( you should not see _1_ in the log or the output )!\n"
LOG("10 Hello World back from the void ( you should not see _1_ in the log or the output )!\n");
// 禁用日志记录
log_disable();
// 设置日志记录目标为 "llama.anotherlog.log"
log_set_target("llama.anotherlog.log");
// 输出日志信息，但由于日志记录已被禁用，不会被记录
LOG("11 Hello World _2_ to nobody, new target was selected but logs are still disabled!\n");
// 启用日志记录
log_enable();
// 输出日志信息，日志将被记录在新文件中
LOG("12 Hello World this time in a new file ( you should not see _2_ in the log or the output )?\n");
// 设置日志记录目标为 "llama.yetanotherlog.log"
log_set_target("llama.yetanotherlog.log");
// 输出日志信息，日志将被记录在另一个新文件中
LOG("13 Hello World this time in yet new file?\n");
// 设置日志记录目标为根据给定前缀和后缀生成的自动命名文件
log_set_target(log_filename_generator("llama_autonamed", "log"));
// 输出日志信息，日志将被记录在自动生成的文件中
LOG("14 Hello World in log with generated filename!\n");
#ifdef _MSC_VER
    // 输出日志信息，同时将信息输出到标准输出
    LOG_TEE("15 Hello msvc TEE without arguments\n");
    // 输出日志信息，同时将信息输出到标准输出，并包括参数
    LOG_TEE("16 Hello msvc TEE with (%d)(%s) arguments\n", 1, "test");
    // 输出日志信息，同时将信息输出到标准输出并换行
    LOG_TEELN("17 Hello msvc TEELN without arguments\n");
    // 输出日志信息，同时将信息输出到标准输出并换行，包括参数
    LOG_TEELN("18 Hello msvc TEELN with (%d)(%s) arguments\n", 1, "test");
    // 输出日志信息
    LOG("19 Hello msvc LOG without arguments\n");
    // 输出日志信息，包括参数
    LOG("20 Hello msvc LOG with (%d)(%s) arguments\n", 1, "test");
    // 输出日志信息并换行
    LOGLN("21 Hello msvc LOGLN without arguments\n");
    // 输出日志信息并换行，包括参数
    LOGLN("22 Hello msvc LOGLN with (%d)(%s) arguments\n", 1, "test");
#endif
// 检查参数是否为"--log-test"，如果是则调用log_test()函数并返回true
inline bool log_param_single_parse(const std::string & param)
{
    if ( param == "--log-test")
    {
        log_test();
        return true;
    }

    // 检查参数是否为"--log-disable"，如果是则调用log_disable()函数并返回true
    if ( param == "--log-disable")
    {
        log_disable();
        return true;
    }

    // 检查参数是否为"--log-enable"，如果是则调用log_enable()函数并返回true
    if ( param == "--log-enable")
    {
        log_enable();
        return true;
    }

    // 如果参数为"--log-new"，则调用log_multilog函数并返回true
    if (param == "--log-new")
    {
        log_multilog(true);
        return true;
    }

    // 如果参数为"--log-append"，则调用log_append函数并返回true
    if (param == "--log-append")
    {
        log_append(true);
        return true;
    }

    // 默认返回false
    return false;
}

// 解析日志参数对
inline bool log_param_pair_parse(bool check_but_dont_parse, const std::string & param, const std::string & next = std::string())
{
    // 如果参数为"--log-file"，则...
// 如果不需要检查但不解析，则设置日志目标为日志文件名生成器生成的文件名
if (!check_but_dont_parse)
{
    log_set_target(log_filename_generator(next.empty() ? "unnamed" : next, "log"));
}
// 返回 true
return true;
}

// 返回 false
return false;
}

// 打印日志使用说明
inline void log_print_usage()
{
    // 打印日志选项
    printf("log options:\n");
    /* 格式
    printf("  -h, --help            show this help message and exit\n");*/
    /* 空格
    printf("__-param----------------Description\n");*/
    // 运行简单的日志测试
    printf("  --log-test            Run simple logging test\n");
// 打印命令行参数，用于日志配置
printf("  --log-disable         Disable trace logs\n");
printf("  --log-enable          Enable trace logs\n");
printf("  --log-file            Specify a log filename (without extension)\n");
printf("  --log-new             Create a separate new log file on start. "
                               "Each log file will have unique name: \"<name>.<ID>.log\"\n");
printf("  --log-append          Don't truncate the old log file.\n");
}

// 宏定义，用于打印命令行参数
#define log_dump_cmdline(argc, argv) log_dump_cmdline_impl(argc, argv)

// 内部函数，用于打印命令行参数
// 注意：不要直接调用此函数
inline void log_dump_cmdline_impl(int argc, char **argv)
{
    // 创建一个字符串流
    std::stringstream buf;
    // 遍历命令行参数
    for (int i = 0; i < argc; ++i)
    {
        // 如果参数中包含空格，则在参数两端加上双引号
        if (std::string(argv[i]).find(' ') != std::string::npos)
        {
            buf << " \"" << argv[i] <<"\"";
        }
        else
        {
            // 如果不是第一个参数，向缓冲区中添加空格和参数值
            buf << " " << argv[i];
        }
    }
    // 打印命令行参数
    LOGLN("Cmd:%s", buf.str().c_str());
}

// 定义宏，将变量转换为字符串
#define log_tostr(var) log_var_to_string_impl(var).c_str()

// 将布尔变量转换为字符串
inline std::string log_var_to_string_impl(bool var)
{
    return var ? "true" : "false";
}

// 将字符串变量转换为字符串
inline std::string log_var_to_string_impl(std::string var)
{
    return var;
}
// 将整数向量转换为字符串表示
inline std::string log_var_to_string_impl(const std::vector<int> & var)
{
    // 创建一个字符串流对象
    std::stringstream buf;
    // 在字符串流中添加左括号
    buf << "[ ";
    // 初始化一个布尔变量，用于判断是否是第一个元素
    bool first = true;
    // 遍历整数向量
    for (auto e : var)
    {
        // 如果是第一个元素，则不添加逗号
        if (first)
        {
            first = false;
        }
        // 如果不是第一个元素，则在字符串流中添加逗号
        else
        {
            buf << ", ";
        }
        // 在字符串流中添加当前整数的字符串表示
        buf << std::to_string(e);
    }
    // 在字符串流中添加右括号
    buf << " ]";

    // 返回字符串流中的内容
    return buf.str();
}
// 定义一个模板函数，将上下文和令牌转换为漂亮的字符串
template <typename C, typename T>
inline std::string LOG_TOKENS_TOSTR_PRETTY(const C & ctx, const T & tokens)
{
    // 创建一个字符串流对象
    std::stringstream buf;
    // 在字符串流中添加起始标记
    buf << "[ ";

    // 初始化一个布尔变量，用于判断是否是第一个令牌
    bool first = true;
    // 遍历令牌集合
    for (const auto &token : tokens)
    {
        // 如果不是第一个令牌，则在字符串流中添加逗号和空格
        if (!first) {
            buf << ", ";
        } else {
            // 如果是第一个令牌，则将first设置为false
            first = false;
        }

        // 调用llama_token_to_piece函数将令牌转换为字符串
        auto detokenized = llama_token_to_piece(ctx, token);

        // 删除字符串中的特定字符
        detokenized.erase(
// 使用 remove_if 函数移除 detokenized 中不可打印字符
std::remove_if(
    detokenized.begin(),
    detokenized.end(),
    [](const unsigned char c) { return !std::isprint(c); }),
    detokenized.end());

// 将 detokenized 和 token 的值以字符串形式添加到 buf 中
buf
    << "'" << detokenized << "'"
    << ":" << std::to_string(token);
}

// 在 buf 中添加结束标记
buf << " ]";

// 返回 buf 的字符串形式
return buf.str();
}

// 定义一个模板函数，将 ctx 和 batch 的内容以漂亮的格式添加到 buf 中
template <typename C, typename B>
inline std::string LOG_BATCH_TOSTR_PRETTY(const C & ctx, const B & batch)
{
    // 创建一个字符串流对象 buf
    std::stringstream buf;
    // 在 buf 中添加起始标记
    buf << "[ ";
    # 定义一个布尔变量，用于标记是否是第一个元素
    bool first = true;
    # 遍历 batch 中的 token
    for (int i = 0; i < batch.n_tokens; ++i)
    {
        # 如果不是第一个元素，向 buf 中添加逗号和空格
        if (!first) {
            buf << ", ";
        } 
        # 如果是第一个元素，将 first 设为 false
        else {
            first = false;
        }

        # 将 token 转换为字符串
        auto detokenized = llama_token_to_piece(ctx, batch.token[i]);

        # 移除不可打印字符
        detokenized.erase(
            std::remove_if(
                detokenized.begin(),
                detokenized.end(),
                [](const unsigned char c) { return !std::isprint(c); }),
            detokenized.end());

        # 将处理后的字符串添加到 buf 中
        buf
// 将一系列信息拼接成一个字符串，包括索引、token、位置、序列ID、logits等信息
<< "\n" << std::to_string(i)
<< ":token '" << detokenized << "'"
<< ":pos " << std::to_string(batch.pos[i])
<< ":n_seq_id  " << std::to_string(batch.n_seq_id[i])
<< ":seq_id " << std::to_string(batch.seq_id[i][0])
<< ":logits " << std::to_string(batch.logits[i]);

// 将拼接好的信息加入到缓冲区中
buf << " ]";

// 返回缓冲区中的字符串
return buf.str();

// 如果定义了LOG_DISABLE_LOGS，则将LOG和LOGLN宏定义为空，不执行任何操作
#ifdef LOG_DISABLE_LOGS
#undef LOG
#define LOG(...) // dummy stub
#undef LOGLN
#define LOGLN(...) // dummy stub

// 如果定义了LOG_DISABLE_LOGS，则将LOG_TEE宏定义为空，不执行任何操作
#undef LOG_TEE
// 重新定义LOG_TEE宏，将其转换为普通的fprintf函数
#define LOG_TEE(...) fprintf(stderr, __VA_ARGS__)

// 重新定义LOG_TEELN宏，将其转换为普通的fprintf函数
#define LOG_TEELN(...) fprintf(stderr, __VA_ARGS__)

// 取消LOG_DISABLE宏的定义，将其定义为空的宏
#define LOG_DISABLE()

// 取消LOG_ENABLE宏的定义，将其定义为空的宏
#define LOG_ENABLE()

// 取消LOG_ENABLE宏的定义，将其定义为空的宏
#define LOG_ENABLE()

// 取消LOG_SET_TARGET宏的定义，将其定义为空的宏
#define LOG_SET_TARGET(...)

// 取消LOG_DUMP_CMDLINE宏的定义，将其定义为空的宏
#define LOG_DUMP_CMDLINE(...)
// 如果定义了 LOG_DISABLE_LOGS 宏，则禁用日志功能。
```