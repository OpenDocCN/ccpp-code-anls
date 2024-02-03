# `stable-diffusion.cpp\util.cpp`

```cpp
// 包含必要的头文件
#include "util.h"
#include <stdarg.h>
#include <algorithm>
#include <codecvt>
#include <fstream>
#include <locale>
#include <sstream>
#include <string>
#include <thread>
#include <unordered_set>
#include <vector>

// 如果是苹果系统，包含相关头文件
#if defined(__APPLE__) && defined(__MACH__)
#include <sys/sysctl.h>
#include <sys/types.h>
#endif

// 如果不是 Windows 系统，包含相关头文件
#if !defined(_WIN32)
#include <sys/ioctl.h>
#include <unistd.h>
#endif

// 包含自定义的头文件
#include "ggml/ggml.h"
#include "stable-diffusion.h"

// 判断字符串是否以指定字符串结尾
bool ends_with(const std::string& str, const std::string& ending) {
    if (str.length() >= ending.length()) {
        return (str.compare(str.length() - ending.length(), ending.length(), ending) == 0);
    } else {
        return false;
    }
}

// 判断字符串是否以指定字符串开头
bool starts_with(const std::string& str, const std::string& start) {
    if (str.find(start) == 0) {
        return true;
    }
    return false;
}

// 替换字符串中所有指定字符为另一个字符
void replace_all_chars(std::string& str, char target, char replacement) {
    for (size_t i = 0; i < str.length(); ++i) {
        if (str[i] == target) {
            str[i] = replacement;
        }
    }
}

// 格式化字符串
std::string format(const char* fmt, ...) {
    va_list ap;
    va_list ap2;
    va_start(ap, fmt);
    va_copy(ap2, ap);
    int size = vsnprintf(NULL, 0, fmt, ap);
    std::vector<char> buf(size + 1);
    int size2 = vsnprintf(buf.data(), size + 1, fmt, ap2);
    va_end(ap2);
    va_end(ap);
    return std::string(buf.data(), size);
}

// 如果是 Windows 系统，包含相关头文件
#ifdef _WIN32  // code for windows
#include <windows.h>

// 判断文件是否存在
bool file_exists(const std::string& filename) {
    DWORD attributes = GetFileAttributesA(filename.c_str());
    return (attributes != INVALID_FILE_ATTRIBUTES && !(attributes & FILE_ATTRIBUTE_DIRECTORY));
}

// 判断路径是否为目录
bool is_directory(const std::string& path) {
    DWORD attributes = GetFileAttributesA(path.c_str());
    return (attributes != INVALID_FILE_ATTRIBUTES && (attributes & FILE_ATTRIBUTE_DIRECTORY));
}

// 获取完整路径
std::string get_full_path(const std::string& dir, const std::string& filename) {
    std::string full_path = dir + "\\" + filename;
    // 定义 WIN32_FIND_DATA 结构体变量，用于存储查找到的文件信息
    WIN32_FIND_DATA find_file_data;
    // 使用 FindFirstFile 函数查找指定路径下的第一个文件，并将文件信息存储在 find_file_data 中
    HANDLE hFind = FindFirstFile(full_path.c_str(), &find_file_data);

    // 如果找到文件
    if (hFind != INVALID_HANDLE_VALUE) {
        // 关闭文件查找句柄
        FindClose(hFind);
        // 返回完整路径
        return full_path;
    } else {
        // 如果未找到文件，则返回空字符串
        return "";
    }
// 如果不是 Windows 系统，则包含 dirent.h 和 sys/stat.h 头文件
#else  // Unix
#include <dirent.h>
#include <sys/stat.h>

// 检查文件是否存在
bool file_exists(const std::string& filename) {
    struct stat buffer;
    return (stat(filename.c_str(), &buffer) == 0 && S_ISREG(buffer.st_mode));
}

// 检查路径是否为目录
bool is_directory(const std::string& path) {
    struct stat buffer;
    return (stat(path.c_str(), &buffer) == 0 && S_ISDIR(buffer.st_mode));
}

// 获取完整路径
std::string get_full_path(const std::string& dir, const std::string& filename) {
    // 打开目录
    DIR* dp = opendir(dir.c_str());

    if (dp != nullptr) {
        struct dirent* entry;

        // 遍历目录中的文件
        while ((entry = readdir(dp)) != nullptr) {
            // 检查文件名是否匹配
            if (strcasecmp(entry->d_name, filename.c_str()) == 0) {
                closedir(dp);
                return dir + "/" + entry->d_name;
            }
        }

        closedir(dp);
    }

    return "";
}

#endif

// 获取物理核心数
// 代码来源：https://github.com/ggerganov/llama.cpp/blob/master/examples/common.cpp
// 许可证：https://github.com/ggerganov/llama.cpp/blob/master/LICENSE
int32_t get_num_physical_cores() {
#ifdef __linux__
    // 枚举线程的同级，条目数即为核心数
    std::unordered_set<std::string> siblings;
    for (uint32_t cpu = 0; cpu < UINT32_MAX; ++cpu) {
        std::ifstream thread_siblings("/sys/devices/system/cpu" + std::to_string(cpu) + "/topology/thread_siblings");
        if (!thread_siblings.is_open()) {
            break;  // 没有更多的 CPU
        }
        std::string line;
        if (std::getline(thread_siblings, line)) {
            siblings.insert(line);
        }
    }
    if (siblings.size() > 0) {
        return static_cast<int32_t>(siblings.size());
    }
// 如果是苹果系统
#elif defined(__APPLE__) && defined(__MACH__)
    int32_t num_physical_cores;
    size_t len = sizeof(num_physical_cores);
    int result = sysctlbyname("hw.perflevel0.physicalcpu", &num_physical_cores, &len, NULL, 0);
    if (result == 0) {
        return num_physical_cores;
    }
    # 调用系统函数sysctlbyname获取物理CPU核心数，将结果存储在num_physical_cores中
    result = sysctlbyname("hw.physicalcpu", &num_physical_cores, &len, NULL, 0);
    # 如果sysctlbyname函数执行成功，返回获取到的物理CPU核心数
    if (result == 0) {
        return num_physical_cores;
    }
#ifdef defined(_WIN32)
    // 如果是 Windows 平台，需要实现相应的功能
    // TODO: Implement
#endif

    // 获取系统支持的线程数
    unsigned int n_threads = std::thread::hardware_concurrency();
    // 根据线程数确定并返回合适的线程数量
    return n_threads > 0 ? (n_threads <= 4 ? n_threads : n_threads / 2) : 4;
}

// 将 UTF-8 字符串转换为 UTF-32 字符串
std::u32string utf8_to_utf32(const std::string& utf8_str) {
    std::wstring_convert<std::codecvt_utf8<char32_t>, char32_t> converter;
    return converter.from_bytes(utf8_str);
}

// 将 UTF-32 字符串转换为 UTF-8 字符串
std::string utf32_to_utf8(const std::u32string& utf32_str) {
    std::wstring_convert<std::codecvt_utf8<char32_t>, char32_t> converter;
    return converter.to_bytes(utf32_str);
}

// 将 Unicode 值转换为 UTF-32 字符串
std::u32string unicode_value_to_utf32(int unicode_value) {
    std::u32string utf32_string = {static_cast<char32_t>(unicode_value)};
    return utf32_string;
}

// 获取路径中的基本文件名
std::string sd_basename(const std::string& path) {
    // 查找路径中最后一个 '/' 的位置
    size_t pos = path.find_last_of('/');
    if (pos != std::string::npos) {
        return path.substr(pos + 1);
    }
    // 查找路径中最后一个 '\\' 的位置
    pos = path.find_last_of('\\');
    if (pos != std::string::npos) {
        return path.substr(pos + 1);
    }
    // 如果没有找到 '/' 或 '\\'，直接返回原路径
    return path;
}

// 拼接两个路径
std::string path_join(const std::string& p1, const std::string& p2) {
    if (p1.empty()) {
        return p2;
    }

    if (p2.empty()) {
        return p1;
    }

    // 判断 p1 的最后一个字符是否为 '/' 或 '\\'
    if (p1[p1.length() - 1] == '/' || p1[p1.length() - 1] == '\\') {
        return p1 + p2;
    }

    return p1 + "/" + p2;
}

// 显示进度条
void pretty_progress(int step, int steps, float time) {
    std::string progress = "  |";
    int max_progress     = 50;
    int32_t current      = (int32_t)(step * 1.f * max_progress / steps);
    for (int i = 0; i < 50; i++) {
        if (i > current) {
            progress += " ";
        } else if (i == current && i != max_progress - 1) {
            progress += ">";
        } else {
            progress += "=";
        }
    }
    progress += "|";
    // 格式化输出进度条和时间信息
    printf(time > 1.0f ? "\r%s %i/%i - %.2fs/it" : "\r%s %i/%i - %.2fit/s",
           progress.c_str(), step, steps,
           time > 1.0f || time == 0 ? time : (1.0f / time));
    fflush(stdout);  // 刷新输出缓冲区，用于 Linux 平台
    # 如果当前步骤等于总步骤数
    if (step == steps) {
        # 打印换行符
        printf("\n");
    }
// 左侧大括号，表示函数或代码块的开始
}

// 去除字符串左侧空格的函数
std::string ltrim(const std::string& s) {
    // 使用 lambda 表达式查找第一个非空格字符
    auto it = std::find_if(s.begin(), s.end(), [](int ch) {
        return !std::isspace(ch);
    });
    // 返回去除左侧空格后的字符串
    return std::string(it, s.end());
}

// 去除字符串右侧空格的函数
std::string rtrim(const std::string& s) {
    // 使用 lambda 表达式查找第一个非空格字符
    auto it = std::find_if(s.rbegin(), s.rend(), [](int ch) {
        return !std::isspace(ch);
    });
    // 返回去除右侧空格后的字符串
    return std::string(s.begin(), it.base());
}

// 去除字符串两侧空格的函数
std::string trim(const std::string& s) {
    // 调用 rtrim 和 ltrim 函数去除两侧空格
    return rtrim(ltrim(s));
}

// 静态变量，用于存储日志回调函数和数据
static sd_log_cb_t sd_log_cb = NULL;
void* sd_log_cb_data         = NULL;

// 定义日志缓冲区大小
#define LOG_BUFFER_SIZE 1024

// 打印日志的函数
void log_printf(sd_log_level_t level, const char* file, int line, const char* format, ...) {
    va_list args;
    va_start(args, format);

    // 根据日志级别设置日志字符串
    const char* level_str = "DEBUG";
    if (level == SD_LOG_INFO) {
        level_str = "INFO ";
    } else if (level == SD_LOG_WARN) {
        level_str = "WARN ";
    } else if (level == SD_LOG_ERROR) {
        level_str = "ERROR";
    }

    // 静态日志缓冲区
    static char log_buffer[LOG_BUFFER_SIZE];

    // 格式化日志字符串
    int written = snprintf(log_buffer, LOG_BUFFER_SIZE, "[%s] %s:%-4d - ", level_str, sd_basename(file).c_str(), line);

    // 如果写入成功，继续格式化日志内容
    if (written >= 0 && written < LOG_BUFFER_SIZE) {
        vsnprintf(log_buffer + written, LOG_BUFFER_SIZE - written, format, args);
        strncat(log_buffer, "\n", LOG_BUFFER_SIZE - strlen(log_buffer) - 1);
    }

    // 如果设置了日志回调函数，调用回调函数
    if (sd_log_cb) {
        sd_log_cb(level, log_buffer, sd_log_cb_data);
    }

    va_end(args);
}

// 设置日志回调函数
void sd_set_log_callback(sd_log_cb_t cb, void* data) {
    sd_log_cb      = cb;
    sd_log_cb_data = data;
}

// 获取系统信息的函数
const char* sd_get_system_info() {
    // 静态字符数组用于存储系统信息
    static char buffer[1024];
    std::stringstream ss;
    ss << "System Info: \n";
    ss << "    BLAS = " << ggml_cpu_has_blas() << std::endl;
    ss << "    SSE3 = " << ggml_cpu_has_sse3() << std::endl;
    ss << "    AVX = " << ggml_cpu_has_avx() << std::endl;
    ss << "    AVX2 = " << ggml_cpu_has_avx2() << std::endl;
    ss << "    AVX512 = " << ggml_cpu_has_avx512() << std::endl;
    // 将 AVX512_VBMI 的信息添加到输出流中
    ss << "    AVX512_VBMI = " << ggml_cpu_has_avx512_vbmi() << std::endl;
    // 将 AVX512_VNNI 的信息添加到输出流中
    ss << "    AVX512_VNNI = " << ggml_cpu_has_avx512_vnni() << std::endl;
    // 将 FMA 的信息添加到输出流中
    ss << "    FMA = " << ggml_cpu_has_fma() << std::endl;
    // 将 NEON 的信息添加到输出流中
    ss << "    NEON = " << ggml_cpu_has_neon() << std::endl;
    // 将 ARM_FMA 的信息添加到输出流中
    ss << "    ARM_FMA = " << ggml_cpu_has_arm_fma() << std::endl;
    // 将 F16C 的信息添加到输出流中
    ss << "    F16C = " << ggml_cpu_has_f16c() << std::endl;
    // 将 FP16_VA 的信息添加到输出流中
    ss << "    FP16_VA = " << ggml_cpu_has_fp16_va() << std::endl;
    // 将 WASM_SIMD 的信息添加到输出流中
    ss << "    WASM_SIMD = " << ggml_cpu_has_wasm_simd() << std::endl;
    // 将 VSX 的信息添加到输出流中
    ss << "    VSX = " << ggml_cpu_has_vsx() << std::endl;
    // 将输出流中的内容格式化为字符串并存储到 buffer 中
    snprintf(buffer, sizeof(buffer), "%s", ss.str().c_str());
    // 返回存储了 CPU 信息的 buffer
    return buffer;
# 返回一个指向字符串常量的指针，该字符串是根据给定的枚举类型转换为对应的字符串类型的函数
const char* sd_type_name(enum sd_type_t type) {
    # 调用 ggml_type_name 函数，将枚举类型转换为 ggml_type 类型，再转换为对应的字符串类型
    return ggml_type_name((ggml_type)type);
}
```