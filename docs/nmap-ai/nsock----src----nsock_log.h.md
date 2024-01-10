# `nmap\nsock\src\nsock_log.h`

```
#ifndef NSOCK_LOG_H
#define NSOCK_LOG_H

#include "nsock.h"

extern nsock_loglevel_t NsockLogLevel;  // 声明全局变量 NsockLogLevel，用于记录日志级别
extern nsock_logger_t   NsockLogger;    // 声明全局变量 NsockLogger，用于记录日志

// 定义宏 NSOCK_LOG_WRAP，用于包装日志记录操作
#define NSOCK_LOG_WRAP(lvl, ...)  \
    do { \
        if (NsockLogger && (lvl) >= NsockLogLevel) {  // 如果存在日志记录器并且级别大于等于设定的日志级别
            __nsock_log_internal((lvl), __FILE__, __LINE__, __func__, __VA_ARGS__);  // 调用内部日志记录函数
        } \
    } while (0)

// 定义内联函数 nsock_loglevel2str，用于将日志级别转换为字符串
static inline const char *nsock_loglevel2str(nsock_loglevel_t level)
{
  switch (level) {
    case NSOCK_LOG_DBG_ALL:
      return "FULL DEBUG";
    case NSOCK_LOG_DBG:
      return "DEBUG";
    case NSOCK_LOG_INFO:
      return "INFO";
    case NSOCK_LOG_ERROR:
      return "ERROR";
    default:
      return "???";
  }
}

/* -- Internal logging macros -- */

// 定义宏 nsock_log_debug_all，用于记录最详细的调试信息
#define nsock_log_debug_all(...) NSOCK_LOG_WRAP(NSOCK_LOG_DBG_ALL, __VA_ARGS__)

// 定义宏 nsock_log_debug，用于记录详细的调试信息
#define nsock_log_debug(...)     NSOCK_LOG_WRAP(NSOCK_LOG_DBG, __VA_ARGS__)

// 定义宏 nsock_log_info，用于记录高级别的调试信息
#define nsock_log_info(...)      NSOCK_LOG_WRAP(NSOCK_LOG_INFO, __VA_ARGS__)

// 定义宏 nsock_log_error，用于记录错误信息
#define nsock_log_error(...)     NSOCK_LOG_WRAP(NSOCK_LOG_ERROR, __VA_ARGS__)

// 声明内部函数 __nsock_log_internal，用于实际记录日志
void __nsock_log_internal(nsock_loglevel_t loglevel, const char *file, int line,
                          const char *func, const char *format, ...)
                          __attribute__((format (printf, 5, 6)));  // 使用属性声明格式化字符串

#endif /* NSOCK_LOG_H */
```