# `stable-diffusion.cpp\util.h`

```
#ifndef __UTIL_H__
#define __UTIL_H__

#include <cstdint>
#include <string>

#include "stable-diffusion.h"

// 判断字符串是否以指定后缀结束
bool ends_with(const std::string& str, const std::string& ending);
// 判断字符串是否以指定前缀开始
bool starts_with(const std::string& str, const std::string& start);

// 格式化字符串
std::string format(const char* fmt, ...);

// 替换字符串中所有指定字符为目标字符
void replace_all_chars(std::string& str, char target, char replacement);

// 判断文件是否存在
bool file_exists(const std::string& filename);
// 判断路径是否为目录
bool is_directory(const std::string& path);
// 获取完整路径
std::string get_full_path(const std::string& dir, const std::string& filename);

// 将 UTF-8 字符串转换为 UTF-32 字符串
std::u32string utf8_to_utf32(const std::string& utf8_str);
// 将 UTF-32 字符串转换为 UTF-8 字符串
std::string utf32_to_utf8(const std::u32string& utf32_str);
// 将 Unicode 值转换为 UTF-32 字符
std::u32string unicode_value_to_utf32(int unicode_value);

// 获取路径中的基本文件名
std::string sd_basename(const std::string& path);

// 拼接路径
std::string path_join(const std::string& p1, const std::string& p2);

// 打印进度信息
void pretty_progress(int step, int steps, float time);

// 打印日志信息
void log_printf(sd_log_level_t level, const char* file, int line, const char* format, ...);

// 去除字符串两端的空格
std::string trim(const std::string& s);

// 定义日志级别宏，便于打印日志信息
#define LOG_DEBUG(format, ...) log_printf(SD_LOG_DEBUG, __FILE__, __LINE__, format, ##__VA_ARGS__)
#define LOG_INFO(format, ...) log_printf(SD_LOG_INFO, __FILE__, __LINE__, format, ##__VA_ARGS__)
#define LOG_WARN(format, ...) log_printf(SD_LOG_WARN, __FILE__, __LINE__, format, ##__VA_ARGS__)
#define LOG_ERROR(format, ...) log_printf(SD_LOG_ERROR, __FILE__, __LINE__, format, ##__VA_ARGS__)
#endif  // __UTIL_H__
```