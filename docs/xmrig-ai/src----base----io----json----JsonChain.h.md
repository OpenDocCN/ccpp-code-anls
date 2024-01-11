# `xmrig\src\base\io\json\JsonChain.h`

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
 * 适销性或特定用途的保证。有关更多详细信息，请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_JSONCHAIN_H
#define XMRIG_JSONCHAIN_H


#include <vector>


#include "3rdparty/rapidjson/document.h"
#include "base/kernel/interfaces/IJsonReader.h"
#include "base/tools/String.h"


namespace xmrig {


class JsonChain : public IJsonReader
{
public:
    // 构造函数
    JsonChain();

    // 添加 JSON 文档
    bool add(rapidjson::Document &&doc);
    // 从文件添加 JSON 文档
    bool addFile(const char *fileName);
    // 直接添加原始 JSON 字符串
    bool addRaw(const char *json);

    // 将 JSON 链写入文件
    void dump(const char *fileName);

    // 返回文件名
    inline const String &fileName() const { return m_fileName; }
    // 返回 JSON 链的大小
    inline size_t size() const            { return m_chain.size(); }

protected:
    // 重写的方法，判断 JSON 链是否为空
    inline bool isEmpty() const override  { return m_chain.empty(); }

    // 重写的方法，获取布尔值
    bool getBool(const char *key, bool defaultValue = false) const override;
    // 重写的方法，获取字符串
    const char *getString(const char *key, const char *defaultValue = nullptr) const override;
    // 重写的方法，获取数组
    const rapidjson::Value &getArray(const char *key) const override;
    // 重写的方法，获取对象
    const rapidjson::Value &getObject(const char *key) const override;
    // 重写的方法，获取值
    const rapidjson::Value &getValue(const char *key) const override;
    // 重写的方法，获取整个 JSON 对象
    const rapidjson::Value &object() const override;
    # 重写父类方法，根据给定的键获取对应的双精度浮点数，如果键不存在则返回默认值
    double getDouble(const char *key, double defaultValue = 0) const override;
    
    # 重写父类方法，根据给定的键获取对应的整数，如果键不存在则返回默认值
    int getInt(const char *key, int defaultValue = 0) const override;
    
    # 重写父类方法，根据给定的键获取对应的64位整数，如果键不存在则返回默认值
    int64_t getInt64(const char *key, int64_t defaultValue = 0) const override;
    
    # 重写父类方法，根据给定的键获取对应的字符串，限定最大长度，如果键不存在则返回空字符串
    String getString(const char *key, size_t maxSize) const override;
    
    # 重写父类方法，根据给定的键获取对应的64位无符号整数，如果键不存在则返回默认值
    uint64_t getUint64(const char *key, uint64_t defaultValue = 0) const override;
    
    # 重写父类方法，根据给定的键获取对应的无符号整数，如果键不存在则返回默认值
    unsigned getUint(const char *key, unsigned defaultValue = 0) const override;
# 声明私有成员变量，用于存储 rapidjson::Document 对象的向量
std::vector<rapidjson::Document> m_chain;
# 声明私有成员变量，用于存储文件名
String m_fileName;
# 结束命名空间 xmrig
} /* namespace xmrig */
# 结束条件编译指令，关闭 XMRIG_JSONCHAIN_H 文件的定义
#endif /* XMRIG_JSONCHAIN_H */
```