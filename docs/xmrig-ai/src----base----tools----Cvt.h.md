# `xmrig\src\base\tools\Cvt.h`

```
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有适销性或特定用途的暗示保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参阅 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CVT_H
#define XMRIG_CVT_H


#include "3rdparty/rapidjson/fwd.h"
#include "base/tools/Buffer.h"
#include "base/tools/Span.h"
#include "base/tools/String.h"


#include <string>


namespace xmrig {


class Cvt
{
public:
    // 从十六进制字符串转换为字节流
    inline static bool fromHex(Buffer &buf, const String &hex)                  { return fromHex(buf, hex.data(), hex.size()); }
    // 从十六进制字符串转换为字节流
    inline static Buffer fromHex(const std::string &hex)                        { return fromHex(hex.data(), hex.size()); }
    // 从十六进制字符串转换为字节流
    inline static Buffer fromHex(const String &hex)                             { return fromHex(hex.data(), hex.size()); }
    // 将数据转换为十六进制字符串
    inline static String toHex(const std::string &data)                         { return toHex(reinterpret_cast<const uint8_t *>(data.data()), data.size()); }

    // 将数据转换为十六进制字符串
    template<typename T>
    inline static String toHex(const T &data)                                   { return toHex(data.data(), data.size()); }

    // 从十六进制字符串转换为字节流
    static bool fromHex(Buffer &buf, const char *in, size_t size);
    // 从 JSON 值的十六进制字符串转换为字节流
    static bool fromHex(Buffer &buf, const rapidjson::Value &value);
    // 从十六进制字符串转换为字符串
    static bool fromHex(std::string &buf, const char *in, size_t size);
    # 将十六进制字符串转换为二进制数据，存储到指定的数组中
    static bool fromHex(uint8_t *bin, size_t bin_maxlen, const char *hex, size_t hex_len);
    
    # 将 JSON 值中的十六进制字符串转换为二进制数据，存储到指定的数组中
    static bool fromHex(uint8_t *bin, size_t bin_maxlen, const rapidjson::Value &value);
    
    # 将二进制数据转换为对应的十六进制字符串，存储到指定的数组中
    static bool toHex(char *hex, size_t hex_maxlen, const uint8_t *bin, size_t bin_len);
    
    # 将十六进制字符串转换为二进制数据，返回对应的二进制数据对象
    static Buffer fromHex(const char *in, size_t size);
    
    # 生成指定长度的随机二进制数据
    static Buffer randomBytes(size_t size);
    
    # 将二进制数据转换为对应的十六进制字符串，存储到 JSON 文档中
    static rapidjson::Value toHex(const Buffer &data, rapidjson::Document &doc);
    
    # 将二进制数据转换为对应的十六进制字符串，存储到 JSON 文档中
    static rapidjson::Value toHex(const Span &data, rapidjson::Document &doc);
    
    # 将字符串转换为对应的十六进制字符串，存储到 JSON 文档中
    static rapidjson::Value toHex(const std::string &data, rapidjson::Document &doc);
    
    # 将十六进制字符串转换为二进制数据，存储到 JSON 文档中
    static rapidjson::Value toHex(const uint8_t *in, size_t size, rapidjson::Document &doc);
    
    # 将十六进制字符串转换为对应的十六进制字符串
    static String toHex(const uint8_t *in, size_t size);
    
    # 生成指定长度的随机二进制数据，存储到指定的缓冲区中
    static void randomBytes(void *buf, size_t size);
}; 
// 结束了 xmrig 命名空间的定义

} /* namespace xmrig */
// 声明 xmrig 命名空间的结束

#endif /* XMRIG_CVT_H */
// 结束了 XMRIG_CVT_H 文件的条件编译
```