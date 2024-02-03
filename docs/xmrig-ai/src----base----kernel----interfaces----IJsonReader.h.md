# `xmrig\src\base\kernel\interfaces\IJsonReader.h`

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
 *   但没有任何保证；甚至没有对特定目的的隐含保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_IJSONREADER_H
#define XMRIG_IJSONREADER_H


#include "3rdparty/rapidjson/fwd.h"
#include "base/tools/Object.h"
#include "base/tools/String.h"


namespace xmrig {


class IJsonReader
{
public:
    XMRIG_DISABLE_COPY_MOVE(IJsonReader)

    // 默认构造函数
    IJsonReader()           = default;
    // 虚析构函数
    virtual ~IJsonReader()  = default;

    // 获取布尔值
    virtual bool getBool(const char *key, bool defaultValue = false) const                       = 0;
    // 判断是否为空
    virtual bool isEmpty() const                                                                 = 0;
    // 获取字符串
    virtual const char *getString(const char *key, const char *defaultValue = nullptr) const     = 0;
    // 获取数组
    virtual const rapidjson::Value &getArray(const char *key) const                              = 0;
    // 获取对象
    virtual const rapidjson::Value &getObject(const char *key) const                             = 0;
    // 获取值
    virtual const rapidjson::Value &getValue(const char *key) const                              = 0;
    // 获取对象
    virtual const rapidjson::Value &object() const                                               = 0;
    // 获取双精度浮点数
    virtual double getDouble(const char *key, double defaultValue = 0) const                     = 0;
    # 获取指定键对应的整数值，如果键不存在则返回默认值
    virtual int getInt(const char *key, int defaultValue = 0) const                              = 0;
    # 获取指定键对应的64位整数值，如果键不存在则返回默认值
    virtual int64_t getInt64(const char *key, int64_t defaultValue = 0) const                    = 0;
    # 获取指定键对应的字符串值，限定最大长度为maxSize
    virtual String getString(const char *key, size_t maxSize) const                              = 0;
    # 获取指定键对应的64位无符号整数值，如果键不存在则返回默认值
    virtual uint64_t getUint64(const char *key, uint64_t defaultValue = 0) const                 = 0;
    # 获取指定键对应的无符号整数值，如果键不存在则返回默认值
    virtual unsigned getUint(const char *key, unsigned defaultValue = 0) const                   = 0;
}; 
// 结束了 xmrig 命名空间的定义

} /* namespace xmrig */
// 声明当前代码块在 xmrig 命名空间中

#endif // XMRIG_IJSONREADER_H
// 结束了对 XMRIG_IJSONREADER_H 的条件编译
```