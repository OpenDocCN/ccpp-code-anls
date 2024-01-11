# `xmrig\src\base\api\interfaces\IApiRequest.h`

```
/*
 * XMRig
 * 版权所有 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据由自由软件基金会发布的 GNU 通用公共许可证的条款
 * 它，无论是许可证的第 3 版，还是
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_IAPIREQUEST_H
#define XMRIG_IAPIREQUEST_H


#include "3rdparty/rapidjson/fwd.h"
#include "base/tools/Object.h"


namespace xmrig {


class String;


class IApiRequest
{
public:
    XMRIG_DISABLE_COPY_MOVE(IApiRequest)

    enum Method {
        METHOD_DELETE,
        METHOD_GET,
        METHOD_HEAD,
        METHOD_POST,
        METHOD_PUT
    };


    enum Source {
        SOURCE_HTTP
    };


    enum RequestType {
        REQ_UNKNOWN,
        REQ_SUMMARY,
        REQ_JSON_RPC
    };


    enum ErrorCode : int {
        RPC_PARSE_ERROR      = -32700,
        RPC_INVALID_REQUEST  = -32600,
        RPC_METHOD_NOT_FOUND = -32601,
        RPC_INVALID_PARAMS   = -32602
    };


    IApiRequest()           = default;
    virtual ~IApiRequest()  = default;

    virtual bool accept()                                               = 0;
    virtual bool hasParseError() const                                  = 0;
    virtual bool isDone() const                                         = 0;
    virtual bool isNew() const                                          = 0;
    virtual bool isRestricted() const                                   = 0;
    # 返回 JSON 值的引用
    virtual const rapidjson::Value &json() const                        = 0;
    # 返回 RPC 方法的名称
    virtual const String &rpcMethod() const                             = 0;
    # 返回 URL 地址
    virtual const String &url() const                                   = 0;
    # 返回版本号
    virtual int version() const                                         = 0;
    # 返回请求方法类型
    virtual Method method() const                                       = 0;
    # 返回 JSON 文档的引用
    virtual rapidjson::Document &doc()                                  = 0;
    # 返回响应的 JSON 值的引用
    virtual rapidjson::Value &reply()                                   = 0;
    # 返回请求类型
    virtual RequestType type() const                                    = 0;
    # 返回请求来源
    virtual Source source() const                                       = 0;
    # 标记请求处理完成，并设置状态码
    virtual void done(int status)                                       = 0;
    # 设置 RPC 错误码和消息
    virtual void setRpcError(int code, const char *message = nullptr)   = 0;
    # 设置 RPC 结果
    virtual void setRpcResult(rapidjson::Value &result)                 = 0;
}; 
// 结束命名空间 xmrig

} /* namespace xmrig */
// 声明命名空间 xmrig 结束

#endif // XMRIG_IAPIREQUEST_H
// 结束宏定义 XMRIG_IAPIREQUEST_H
```