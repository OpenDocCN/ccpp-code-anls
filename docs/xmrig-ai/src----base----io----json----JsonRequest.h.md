# `xmrig\src\base\io\json\JsonRequest.h`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，版本为3或
 *   （根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_JSONREQUEST_H
#define XMRIG_JSONREQUEST_H


#include "3rdparty/rapidjson/fwd.h"


namespace xmrig {


class JsonRequest
{
public:
    static const char *k2_0;
    static const char *kId;
    static const char *kJsonRPC;
    static const char *kMethod;
    static const char *kOK;
    static const char *kParams;
    static const char *kResult;
    static const char *kStatus;

    static const char *kParseError;
    static const char *kInvalidRequest;
    static const char *kMethodNotFound;
    static const char *kInvalidParams;
    static const char *kInternalError;

    constexpr static int kParseErrorCode        = -32700;
    constexpr static int kInvalidRequestCode    = -32600;
    constexpr static int kMethodNotFoundCode    = -32601;
    constexpr static int kInvalidParamsCode     = -32602;
    constexpr static int kInternalErrorCode     = -32603;

    static rapidjson::Document create(const char *method);
    static rapidjson::Document create(int64_t id, const char *method);
    static uint64_t create(rapidjson::Document &doc, const char *method, rapidjson::Value &params);
    // 创建一个 JSON 文档对象，传入参数为整型 id、字符指针 method 和 JSON 值 params，返回一个 64 位无符号整型数
    static uint64_t create(rapidjson::Document &doc, int64_t id, const char *method, rapidjson::Value &params);
}; 
// 结束了一个命名空间的定义

} /* namespace xmrig */
// 命名空间结束

#endif /* XMRIG_JSONREQUEST_H */
// 结束了对头文件的引用声明
```