# `xmrig\src\base\kernel\config\BaseTransform.h`

```
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有对特定目的的隐含保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_BASETRANSFORM_H
#define XMRIG_BASETRANSFORM_H


#include "3rdparty/rapidjson/document.h"
#include "base/crypto/Coin.h"
#include "base/kernel/interfaces/IConfigTransform.h"


struct option;


namespace xmrig {


class IConfigTransform;
class JsonChain;
class Process;


class BaseTransform : public IConfigTransform
{
public:
    static void load(JsonChain &chain, Process *process, IConfigTransform &transform);

protected:
    void finalize(rapidjson::Document &doc) override;
    void transform(rapidjson::Document &doc, int key, const char *arg) override;


    template<typename T>
    inline void set(rapidjson::Document &doc, const char *key, T value) { set<T>(doc, doc, key, value); }


    template<typename T>
    inline void set(rapidjson::Document &doc, const char *objKey, const char *key, T value)
    {
        if (!doc.HasMember(objKey)) {
            doc.AddMember(rapidjson::StringRef(objKey), rapidjson::kObjectType, doc.GetAllocator());
        }

        set<T>(doc, doc[objKey], key, value);
    }


    template<typename T>
    # 定义一个内联函数，用于向 rapidjson::Document 中的数组类型字段添加值
    inline void add(rapidjson::Document &doc, const char *arrayKey, const char *key, T value, bool force = false)
    {
        # 获取 rapidjson::Document 的分配器
        auto &allocator = doc.GetAllocator();
    
        # 如果 rapidjson::Document 中不存在指定的数组字段
        if (!doc.HasMember(arrayKey)) {
            # 向 rapidjson::Document 中添加指定名称的数组字段
            doc.AddMember(rapidjson::StringRef(arrayKey), rapidjson::kArrayType, allocator);
        }
    
        # 获取指定名称的数组字段
        rapidjson::Value &array = doc[arrayKey];
        # 如果 force 为真或者数组为空
        if (force || array.Size() == 0) {
            # 向数组中添加一个对象
            array.PushBack(rapidjson::kObjectType, allocator);
        }
    
        # 调用 set 函数设置值
        set<T>(doc, array[array.Size() - 1], key, value);
    }
    
    
    # 定义一个模板内联函数，用于设置 rapidjson::Document 中对象类型字段的值
    template<typename T>
    inline void set(rapidjson::Document &doc, rapidjson::Value &obj, const char *key, T value)
    {
        # 如果对象中已经存在指定名称的字段
        if (obj.HasMember(key)) {
            # 直接设置字段的值
            obj[key] = value;
        }
        else {
            # 向对象中添加指定名称的字段和对应的值
            obj.AddMember(rapidjson::StringRef(key), value, doc.GetAllocator());
        }
    }
protected:
    // 保护成员变量，表示算法
    Algorithm m_algorithm;
    // 保护成员变量，表示货币
    Coin m_coin;

private:
    // 私有方法，用于将布尔类型转换为 JSON 对象
    void transformBoolean(rapidjson::Document &doc, int key, bool enable);
    // 私有方法，用于将无符号 64 位整数转换为 JSON 对象
    void transformUint64(rapidjson::Document &doc, int key, uint64_t arg);

    // 私有成员变量，表示是否使用 HTTP
    bool m_http = false;
};


// 模板特化，用于设置字符串类型的 JSON 对象
template<>
inline void BaseTransform::set(rapidjson::Document &doc, rapidjson::Value &obj, const char *key, const char *value)
{
    // 获取 JSON 对象的分配器
    auto &allocator = doc.GetAllocator();

    // 如果 JSON 对象已经包含指定键值，则更新其值
    if (obj.HasMember(key)) {
        obj[key] = rapidjson::Value(value, allocator);
    }
    // 如果 JSON 对象不包含指定键值，则添加新的键值对
    else {
        obj.AddMember(rapidjson::StringRef(key), rapidjson::Value(value, allocator), allocator);
    }
}


} // namespace xmrig

// 结束 if 预处理指令，结束命名空间声明
#endif /* XMRIG_BASETRANSFORM_H */
```