# `xmrig\src\base\api\requests\HttpApiRequest.h`

```
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是基于希望它有用而分发的，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多详情请参见
 *   GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#ifndef XMRIG_HTTPAPIREQUEST_H
#define XMRIG_HTTPAPIREQUEST_H


#include "base/api/requests/ApiRequest.h"
#include "base/net/http/HttpApiResponse.h"
#include "base/tools/String.h"


namespace xmrig {


class HttpData;


class HttpApiRequest : public ApiRequest
{
public:
    // 构造函数，接受 HttpData 对象和一个布尔值作为参数
    HttpApiRequest(const HttpData &req, bool restricted);

protected:
    // 检查是否有解析错误
    inline bool hasParseError() const override           { return m_parsed == 2; }
    // 返回请求的 URL
    inline const String &url() const override            { return m_url; }
    // 返回 rapidjson 文档对象的引用
    inline rapidjson::Document &doc() override           { return m_res.doc(); }
    // 返回 rapidjson 值对象的引用
    inline rapidjson::Value &reply() override            { return m_res.doc(); }

    // 接受请求
    bool accept() override;
    // 返回 JSON 数据
    const rapidjson::Value &json() const override;
    // 返回请求方法
    Method method() const override;
    // 完成请求
    void done(int status) override;
    // 设置 RPC 错误
    void setRpcError(int code, const char *message = nullptr) override;
    // 设置 RPC 结果
    void setRpcResult(rapidjson::Value &result) override;

private:
    // 完成 RPC 请求
    void rpcDone(const char *key, rapidjson::Value &value);

    const HttpData &m_req;
    HttpApiResponse m_res;
    int m_parsed = 0;
    rapidjson::Document m_body;
    String m_url;
};
} // 结束 xmrig 命名空间
#endif // 结束 XMRIG_HTTPAPIREQUEST_H 的条件编译
```