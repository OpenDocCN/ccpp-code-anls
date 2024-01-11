# `xmrig\src\base\io\json\JsonChain.cpp`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。更多详情请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/io/json/JsonChain.h"
#include "3rdparty/rapidjson/error/en.h"
#include "base/io/json/Json.h"
#include "base/io/log/Log.h"


namespace xmrig {

static const rapidjson::Value kNullValue;

} // namespace xmrig


// 默认构造函数
xmrig::JsonChain::JsonChain() = default;


// 添加 JSON 文档到链中
bool xmrig::JsonChain::add(rapidjson::Document &&doc)
{
    // 如果文档有解析错误，或者不是对象，或者是空对象，则返回false
    if (doc.HasParseError() || !doc.IsObject() || doc.ObjectEmpty()) {
        return false;
    }

    // 将文档移动到链的末尾
    m_chain.push_back(std::move(doc));

    return true;
}


// 从文件中添加 JSON 文档到链中
bool xmrig::JsonChain::addFile(const char *fileName)
{
    using namespace rapidjson;
    Document doc;
    // 从文件中获取 JSON 文档
    if (Json::get(fileName, doc)) {
        m_fileName = fileName;

        return add(std::move(doc));
    }
}
    // 检查文档是否有解析错误
    if (doc.HasParseError()) {
        // 获取解析错误的偏移量
        const size_t offset = doc.GetErrorOffset();

        // 初始化行号、位置和字符串向量
        size_t line = 0;
        size_t pos  = 0;
        std::vector<std::string> s;

        // 将偏移量转换为行号、位置和字符串向量
        if (Json::convertOffset(fileName, offset, line, pos, s)) {
            // 遍历字符串向量并输出错误信息
            for (const auto& t : s) {
                LOG_ERR("%s", t.c_str());
            }

            // 创建指示错误位置的箭头字符串
            std::string t;
            if (pos > 0) {
                t.assign(pos - 1, ' ');
            }
            t += '^';
            LOG_ERR("%s", t.c_str());

            // 输出包含行号、位置和错误信息的完整错误信息
            LOG_ERR("%s<line:%zu, position:%zu>: \"%s\"", fileName, line, pos, GetParseError_En(doc.GetParseError()));
        }
        else {
            // 输出包含偏移量和错误信息的完整错误信息
            LOG_ERR("%s<offset:%zu>: \"%s\"", fileName, offset, GetParseError_En(doc.GetParseError()));
        }
    }
    else {
        // 输出无法打开文件的错误信息
        LOG_ERR("unable to open \"%s\".", fileName);
    }

    // 返回 false
    return false;
}

// 向 JSON 链中添加原始 JSON 数据
bool xmrig::JsonChain::addRaw(const char *json)
{
    // 使用 rapidjson 解析 JSON 数据，允许注释和尾随逗号
    using namespace rapidjson;
    Document doc;
    doc.Parse<kParseCommentsFlag | kParseTrailingCommasFlag>(json);

    // 调用 add 方法，将解析后的 JSON 数据移动到链中
    return add(std::move(doc));
}

// 将 JSON 链中的数据保存到文件
void xmrig::JsonChain::dump(const char *fileName)
{
    // 创建一个 rapidjson 的数组类型的 Document
    rapidjson::Document doc(rapidjson::kArrayType);

    // 遍历 JSON 链中的数据，将每个值添加到 doc 中
    for (rapidjson::Document &value : m_chain) {
        doc.PushBack(value, doc.GetAllocator());
    }

    // 调用 Json 的 save 方法，将 doc 中的数据保存到文件
    Json::save(fileName, doc);
}

// 从 JSON 链中获取布尔类型的值
bool xmrig::JsonChain::getBool(const char *key, bool defaultValue) const
{
    // 从链的末尾开始遍历，查找指定键的布尔值
    for (auto it = m_chain.rbegin(); it != m_chain.rend(); ++it) {
        auto i = it->FindMember(key);
        if (i != it->MemberEnd() && i->value.IsBool()) {
            return i->value.GetBool();
        }
    }

    // 如果未找到指定键的布尔值，则返回默认值
    return defaultValue;
}

// 从 JSON 链中获取字符串类型的值
const char *xmrig::JsonChain::getString(const char *key, const char *defaultValue) const
{
    // 从链的末尾开始遍历，查找指定键的字符串值
    for (auto it = m_chain.rbegin(); it != m_chain.rend(); ++it) {
        auto i = it->FindMember(key);
        if (i != it->MemberEnd() && i->value.IsString()) {
            return i->value.GetString();
        }
    }

    // 如果未找到指定键的字符串值，则返回默认值
    return defaultValue;
}

// 从 JSON 链中获取数组类型的值
const rapidjson::Value &xmrig::JsonChain::getArray(const char *key) const
{
    // 从链的末尾开始遍历，查找指定键的数组值
    for (auto it = m_chain.rbegin(); it != m_chain.rend(); ++it) {
        auto i = it->FindMember(key);
        if (i != it->MemberEnd() && i->value.IsArray()) {
            return i->value;
        }
    }

    // 如果未找到指定键的数组值，则返回空值
    return kNullValue;
}

// 从 JSON 链中获取对象类型的值
const rapidjson::Value &xmrig::JsonChain::getObject(const char *key) const
{
    // 从链的末尾开始遍历，查找指定键的对象值
    for (auto it = m_chain.rbegin(); it != m_chain.rend(); ++it) {
        auto i = it->FindMember(key);
        if (i != it->MemberEnd() && i->value.IsObject()) {
            return i->value;
        }
    }

    // 如果未找到指定键的对象值，则返回空值
    return kNullValue;
}

// 从 JSON 链中获取任意类型的值
const rapidjson::Value &xmrig::JsonChain::getValue(const char *key) const
{
    // 从链的末尾开始遍历，查找指定键的值
    for (auto it = m_chain.rbegin(); it != m_chain.rend(); ++it) {
        auto i = it->FindMember(key);
        if (i != it->MemberEnd()) {
            return i->value;
        }
    }

    // 如果未找到指定键的值，则返回空值
    return kNullValue;
}
    # 结束当前的函数或代码块，并返回一个空值
    }
    # 返回一个空值
    return kNullValue;
// 返回 JSON 链中最后一个元素
const rapidjson::Value &xmrig::JsonChain::object() const
{
    // 断言，如果为假则终止程序
    assert(false);

    // 返回 JSON 链中最后一个元素
    return m_chain.back();
}


// 获取指定键对应的双精度浮点数值
double xmrig::JsonChain::getDouble(const char *key, double defaultValue) const
{
    // 从 JSON 链的末尾开始遍历
    for (auto it = m_chain.rbegin(); it != m_chain.rend(); ++it) {
        // 查找指定键
        auto i = it->FindMember(key);
        // 如果找到并且值为双精度浮点数，则返回该值
        if (i != it->MemberEnd() && (i->value.IsDouble() || i->value.IsLosslessDouble())) {
            return i->value.GetDouble();
        }
    }

    // 如果未找到指定键，则返回默认值
    return defaultValue;
}


// 获取指定键对应的整数值
int xmrig::JsonChain::getInt(const char *key, int defaultValue) const
{
    // 从 JSON 链的末尾开始遍历
    for (auto it = m_chain.rbegin(); it != m_chain.rend(); ++it) {
        // 查找指定键
        auto i = it->FindMember(key);
        // 如果找到并且值为整数，则返回该值
        if (i != it->MemberEnd() && i->value.IsInt()) {
            return i->value.GetInt();
        }
    }

    // 如果未找到指定键，则返回默认值
    return defaultValue;
}


// 获取指定键对应的 64 位整数值
int64_t xmrig::JsonChain::getInt64(const char *key, int64_t defaultValue) const
{
    // 从 JSON 链的末尾开始遍历
    for (auto it = m_chain.rbegin(); it != m_chain.rend(); ++it) {
        // 查找指定键
        auto i = it->FindMember(key);
        // 如果找到并且值为 64 位整数，则返回该值
        if (i != it->MemberEnd() && i->value.IsInt64()) {
            return i->value.GetInt64();
        }
    }

    // 如果未找到指定键，则返回默认值
    return defaultValue;
}


// 获取指定键对应的字符串值
xmrig::String xmrig::JsonChain::getString(const char *key, size_t maxSize) const
{
    // 从 JSON 链的末尾开始遍历
    for (auto it = m_chain.rbegin(); it != m_chain.rend(); ++it) {
        // 查找指定键
        auto i = it->FindMember(key);
        // 如果找到并且值为字符串，则返回该值
        if (i != it->MemberEnd() && i->value.IsString()) {
            // 如果限制了最大长度且超过了最大长度，则返回截断后的字符串
            if (maxSize == 0 || i->value.GetStringLength() <= maxSize) {
                return i->value.GetString();
            }
            return { i->value.GetString(), maxSize };
        }
    }

    // 如果未找到指定键，则返回空字符串
    return {};
}


// 获取指定键对应的 64 位无符号整数值
uint64_t xmrig::JsonChain::getUint64(const char *key, uint64_t defaultValue) const
{
    // 从 JSON 链的末尾开始遍历
    for (auto it = m_chain.rbegin(); it != m_chain.rend(); ++it) {
        // 查找指定键
        auto i = it->FindMember(key);
        // 如果找到并且值为 64 位无符号整数，则返回该值
        if (i != it->MemberEnd() && i->value.IsUint64()) {
            return i->value.GetUint64();
        }
    }

    // 如果未找到指定键，则返回默认值
    return defaultValue;
}
# 从 JSON 链中获取指定键的无符号整数值，如果不存在则返回默认值
unsigned xmrig::JsonChain::getUint(const char *key, unsigned defaultValue) const
{
    # 从 JSON 链的末尾开始遍历
    for (auto it = m_chain.rbegin(); it != m_chain.rend(); ++it) {
        # 查找指定键的成员
        auto i = it->FindMember(key);
        # 如果找到指定键的成员，并且其值是无符号整数类型
        if (i != it->MemberEnd() && i->value.IsUint()) {
            # 返回该键对应的无符号整数值
            return i->value.GetUint();
        }
    }
    # 如果未找到指定键的成员，或者值不是无符号整数类型，则返回默认值
    return defaultValue;
}
```