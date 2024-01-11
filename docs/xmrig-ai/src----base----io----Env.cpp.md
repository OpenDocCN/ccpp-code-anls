# `xmrig\src\base\io\Env.cpp`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据GNU通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版，或者
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/io/Env.h"
#include "base/kernel/Process.h"
#include "version.h"


#include <regex>
#include <uv.h>
#include <map>


#ifndef _WIN32
#   include <unistd.h>
#endif


#ifndef UV_MAXHOSTNAMESIZE
#   ifdef MAXHOSTNAMELEN
#       define UV_MAXHOSTNAMESIZE (MAXHOSTNAMELEN + 1)
#   else
#       define UV_MAXHOSTNAMESIZE 256
#   endif
#endif


namespace xmrig {


#ifdef XMRIG_FEATURE_ENV
static std::map<String, String> variables;


static void createVariables()
{
    // 插入XMRIG_VERSION变量
    variables.insert({ "XMRIG_VERSION",  APP_VERSION });
    // 插入XMRIG_KIND变量
    variables.insert({ "XMRIG_KIND",     APP_KIND });
    // 插入XMRIG_HOSTNAME变量
    variables.insert({ "XMRIG_HOSTNAME", Env::hostname() });
    // 插入XMRIG_EXE变量
    variables.insert({ "XMRIG_EXE",      Process::exepath() });
    // 插入XMRIG_EXE_DIR变量
    variables.insert({ "XMRIG_EXE_DIR",  Process::location(Process::ExeLocation) });
    // 插入XMRIG_CWD变量
    variables.insert({ "XMRIG_CWD",      Process::location(Process::CwdLocation) });
    // 插入XMRIG_HOME_DIR变量
    variables.insert({ "XMRIG_HOME_DIR", Process::location(Process::HomeLocation) });
    // 插入XMRIG_TEMP_DIR变量
    variables.insert({ "XMRIG_TEMP_DIR", Process::location(Process::TempLocation) });
    // 插入XMRIG_DATA_DIR变量
    variables.insert({ "XMRIG_DATA_DIR", Process::location(Process::DataLocation) });
    # 定义字符串变量 hostname，并赋值为 "HOSTNAME"
    String hostname = "HOSTNAME";
    # 如果环境变量中不存在 hostname，则执行以下操作
    if (!getenv(hostname)) { // NOLINT(concurrency-mt-unsafe)
        # 将 hostname 插入到 variables 中，并使用 Env::hostname() 的返回值作为值
        variables.insert({ std::move(hostname), Env::hostname() });
    }
} // 结束 xmrig 命名空间

#endif

} // 结束 xmrig 命名空间

// 根据给定的字符串和额外的映射，扩展环境变量
xmrig::String xmrig::Env::expand(const char *in, const std::map<String, String> &extra)
{
#   ifdef XMRIG_FEATURE_ENV
    // 如果输入为空指针，则返回空字符串
    if (in == nullptr) {
        return {};
    }

    // 将输入的字符数组转换为 std::string
    std::string text(in);
    // 如果字符串长度小于4，则返回原始字符串
    if (text.size() < 4) {
        return text.c_str();
    }

    // 定义用于匹配环境变量的正则表达式
    static const std::regex env_re{R"--(\$\{([^}]+)\})--"};

    // 创建存储环境变量的映射
    std::map<std::string, String> vars;

    // 遍历字符串中匹配的环境变量
    for (std::sregex_iterator i = std::sregex_iterator(text.begin(), text.end(), env_re); i != std::sregex_iterator(); ++i) {
        std::smatch m = *i;
        const auto var = m.str();

        // 如果映射中已经存在该环境变量，则跳过
        if (vars.count(var)) {
            continue;
        }

        // 将环境变量及其值插入映射中
        vars.insert({ var, get(m[1].str().c_str(), extra) });
    }

    // 替换字符串中的环境变量
    for (const auto &kv : vars) {
        if (kv.second.isNull()) {
            continue;
        }

        size_t pos = 0;
        while ((pos = text.find(kv.first, pos)) != std::string::npos) {
            text.replace(pos, kv.first.size(), kv.second);
            pos += kv.second.size();
        }
    }

    // 返回扩展后的字符串
    return text.c_str();
#   else
    // 如果未定义 XMRIG_FEATURE_ENV，则返回原始输入字符串
    return in;
#   endif
}

// 根据给定的名称和额外的映射，获取环境变量的值
xmrig::String xmrig::Env::get(const String &name, const std::map<String, String> &extra)
{
#   ifdef XMRIG_FEATURE_ENV
    // 如果环境变量映射为空，则创建环境变量映射
    if (variables.empty()) {
        createVariables();
    }

    // 在环境变量映射中查找给定名称的值
    const auto it = variables.find(name);
    if (it != variables.end()) {
        return it->second;
    }

    // 如果额外的映射不为空，则在额外的映射中查找给定名称的值
    if (!extra.empty()) {
        const auto it = extra.find(name);
        if (it != extra.end()) {
            return it->second;
        }
    }
#   endif

    // 返回系统环境变量中给定名称的值
    return static_cast<const char *>(getenv(name)); // NOLINT(concurrency-mt-unsafe)
}

// 获取主机名
xmrig::String xmrig::Env::hostname()
{
    char buf[UV_MAXHOSTNAMESIZE]{};

    // 获取主机名并存储在缓冲区中
    if (gethostname(buf, sizeof(buf)) == 0) {
        return static_cast<const char *>(buf);
    }

    // 如果获取失败，则返回空字符串
    return {};
}
```