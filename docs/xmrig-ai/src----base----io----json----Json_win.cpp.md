# `xmrig\src\base\io\json\Json_win.cpp`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有对适销性或特定用途的隐含保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include <windows.h>


#ifdef __GNUC__
#   include <fcntl.h>
#   include <sys/stat.h>
#   include <ext/stdio_filebuf.h>
#endif


#include <fstream>


#include "base/io/json/Json.h"
#include "3rdparty/rapidjson/document.h"
#include "3rdparty/rapidjson/istreamwrapper.h"
#include "3rdparty/rapidjson/ostreamwrapper.h"
#include "3rdparty/rapidjson/prettywriter.h"


namespace xmrig {


#if defined(_MSC_VER) || defined (__GNUC__)
// 将 UTF-8 字符串转换为 UTF-16 字符串
static std::wstring toUtf16(const char *str)
{
    const int size = static_cast<int>(strlen(str));
    std::wstring ret;

    int len = MultiByteToWideChar(CP_UTF8, 0, str, size, nullptr, 0);
    if (len > 0) {
        ret.resize(static_cast<size_t>(len));
        MultiByteToWideChar(CP_UTF8, 0, str, size, &ret[0], len);
    }

    return ret;
}
#endif


#if defined(_MSC_VER)
// 打开并读取文件，将文件名转换为 UTF-16 格式
#   define OPEN_IFS(name)                                                               \
    std::ifstream ifs(toUtf16(name), std::ios_base::in | std::ios_base::binary);        \
    # 如果输入文件流未成功打开
    if (!ifs.is_open()) {                                                               \
        # 返回 false
        return false;                                                                   \
    }
#elif defined(__GNUC__)
#   define OPEN_IFS(name)                                                               \
    // 如果是 GNU 编译器，则使用_wopen函数打开文件，并将文件描述符赋值给fd
    const int fd = _wopen(toUtf16(name).c_str(), _O_RDONLY | _O_BINARY);                \
    // 如果文件描述符为-1，表示打开文件失败，返回false
    if (fd == -1) {                                                                     \
        return false;                                                                   \
    }                                                                                   \
    // 使用文件描述符创建stdio_filebuf对象，并将其赋值给buf
    __gnu_cxx::stdio_filebuf<char> buf(fd, std::ios_base::in | std::ios_base::binary);  \
    // 使用buf创建输入流ifs
    std::istream ifs(&buf);
#else
#   define OPEN_IFS(name)                                                               \
    // 如果不是GNU编译器，则使用ifstream打开文件，并将其赋值给ifs
    std::ifstream ifs(name, std::ios_base::in | std::ios_base::binary);                 \
    // 如果文件未成功打开，则返回false
    if (!ifs.is_open()) {                                                               \
        return false;                                                                   \
    }
#endif

} // namespace xmrig


bool xmrig::Json::get(const char *fileName, rapidjson::Document &doc)
{
    // 使用OPEN_IFS宏打开文件
    OPEN_IFS(fileName)

    using namespace rapidjson;
    // 使用输入流ifs创建IStreamWrapper对象isw
    IStreamWrapper isw(ifs);
    // 使用isw解析JSON文档，并将结果赋值给doc
    doc.ParseStream<kParseCommentsFlag | kParseTrailingCommasFlag>(isw);

    // 返回解析结果是否成功
    return !doc.HasParseError() && (doc.IsObject() || doc.IsArray());
}


bool xmrig::Json::save(const char *fileName, const rapidjson::Document &doc)
{
    using namespace rapidjson;
    // 定义文件打开模式
    constexpr const std::ios_base::openmode mode = std::ios_base::out | std::ios_base::binary | std::ios_base::trunc;

#   if defined(_MSC_VER)
    // 如果是MSVC编译器，则使用toUtf16函数转换文件名，并使用ofstream打开文件
    std::ofstream ofs(toUtf16(fileName), mode);
    // 如果文件未成功打开，则返回false
    if (!ofs.is_open()) {
        return false;
    }
#   elif defined(__GNUC__)
    // 如果是GNU编译器，则使用_wopen函数打开文件，并将文件描述符赋值给fd
    const int fd = _wopen(toUtf16(fileName).c_str(), _O_WRONLY | _O_BINARY | _O_CREAT | _O_TRUNC, _S_IWRITE);
    // 如果文件描述符为-1，表示打开文件失败，返回false
    if (fd == -1) {
        return false;
    }
    // 使用文件描述符创建stdio_filebuf对象，并将其赋值给buf
    __gnu_cxx::stdio_filebuf<char> buf(fd, mode);
    // 使用buf创建输出流ofs
    std::ostream ofs(&buf);
#   else
    // 如果不是MSVC或GNU编译器，则使用ofstream打开文件
    std::ofstream ofs(fileName, mode);
    # 如果输出文件流没有成功打开
    if (!ofs.is_open()) {
        # 返回 false
        return false;
    }
#   endif
    // 创建一个输出流包装器，将数据写入到文件中
    OStreamWrapper osw(ofs);
    // 创建一个 JSON 格式化的写入器
    PrettyWriter<OStreamWrapper> writer(osw);
#   ifdef XMRIG_JSON_SINGLE_LINE_ARRAY
    // 设置 JSON 格式化选项为单行数组
    writer.SetFormatOptions(kFormatSingleLineArray);
#   endif
    // 将 JSON 文档写入到输出流中
    doc.Accept(writer);
    // 返回 true 表示成功
    return true;
}

// 根据文件名、偏移量和输出参数，将文件内容转换为行号、位置和字符串数组
bool xmrig::Json::convertOffset(const char *fileName, size_t offset, size_t &line, size_t &pos, std::vector<std::string> &s)
{
    // 打开文件输入流
    OPEN_IFS(fileName)
    // 调用另一个函数，将输入流、偏移量和输出参数传入，进行转换
    return convertOffset(ifs, offset, line, pos, s);
}
```