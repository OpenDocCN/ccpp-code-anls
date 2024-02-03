# `xmrig\src\base\io\json\Json_unix.cpp`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据GNU通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版，或者
 *   （在您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <fstream>


#include "base/io/json/Json.h"
#include "3rdparty/rapidjson/document.h"
#include "3rdparty/rapidjson/istreamwrapper.h"
#include "3rdparty/rapidjson/ostreamwrapper.h"
#include "3rdparty/rapidjson/prettywriter.h"


// 从文件中读取JSON数据到rapidjson::Document对象中
bool xmrig::Json::get(const char *fileName, rapidjson::Document &doc)
{
    // 以二进制只读模式打开文件流
    std::ifstream ifs(fileName, std::ios_base::in | std::ios_base::binary);
    // 如果文件流未成功打开，则返回false
    if (!ifs.is_open()) {
        return false;
    }

    // 使用文件流创建rapidjson::IStreamWrapper对象
    rapidjson::IStreamWrapper isw(ifs);
    // 从流中解析JSON数据到rapidjson::Document对象中，允许注释和尾随逗号
    doc.ParseStream<rapidjson::kParseCommentsFlag | rapidjson::kParseTrailingCommasFlag>(isw);

    // 返回解析是否成功，并且Document对象是对象或数组
    return !doc.HasParseError() && (doc.IsObject() || doc.IsArray());
}


// 将rapidjson::Document对象保存为JSON格式到文件中
bool xmrig::Json::save(const char *fileName, const rapidjson::Document &doc)
{
    // 以二进制写入模式打开文件流，如果文件不存在则创建，如果文件已存在则清空内容
    std::ofstream ofs(fileName, std::ios_base::out | std::ios_base::binary | std::ios_base::trunc);
    // 如果文件流未成功打开，则返回false
    if (!ofs.is_open()) {
        return false;
    }

    // 使用文件流创建rapidjson::OStreamWrapper对象
    rapidjson::OStreamWrapper osw(ofs);
    // 创建rapidjson::PrettyWriter对象，用于将Document对象以美观的格式写入到流中
    rapidjson::PrettyWriter<rapidjson::OStreamWrapper> writer(osw);

    // 如果定义了XMRIG_JSON_SINGLE_LINE_ARRAY，则设置格式选项为单行数组
#   ifdef XMRIG_JSON_SINGLE_LINE_ARRAY
    writer.SetFormatOptions(rapidjson::kFormatSingleLineArray);
#   endif

    // 将Document对象写入到流中
    doc.Accept(writer);

    // 返回保存是否成功
    return true;
}
# 将文件名、偏移量和字符串向量转换为行号和位置
bool xmrig::Json::convertOffset(const char* fileName, size_t offset, size_t& line, size_t& pos, std::vector<std::string>& s)
{
    # 以二进制只读模式打开文件流
    std::ifstream ifs(fileName, std::ios_base::in | std::ios_base::binary);
    # 如果文件流未成功打开，则返回 false
    if (!ifs.is_open()) {
        return false;
    }
    # 调用另一个重载的 convertOffset 函数，传入文件流、偏移量、行号、位置和字符串向量
    return convertOffset(ifs, offset, line, pos, s);
}
```