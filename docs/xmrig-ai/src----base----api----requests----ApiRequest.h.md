# `xmrig\src\base\api\requests\ApiRequest.h`

```cpp
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2019 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2019 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据 GNU 通用公共许可证的条款发布，
 * 由自由软件基金会发布，无论是许可证的第3版
 * （在您的选择下）或任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。更多细节请参见
 * GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_APIREQUEST_H
#define XMRIG_APIREQUEST_H


#include "base/api/interfaces/IApiRequest.h"
#include "base/tools/String.h"
#include "base/tools/Object.h"


namespace xmrig {


class ApiRequest : public IApiRequest
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(ApiRequest)

    ApiRequest(Source source, bool restricted);
    ~ApiRequest() override;

protected:
    enum State {
        STATE_NEW,
        STATE_ACCEPTED,
        STATE_DONE
    };

    inline bool accept() override                   { m_state = STATE_ACCEPTED; return true; }
    inline bool isDone() const override             { return m_state == STATE_DONE; }
    inline bool isNew() const override              { return m_state == STATE_NEW; }
    # 返回是否受限制的标志
    inline bool isRestricted() const override       { return m_restricted; }
    # 返回 RPC 方法的名称
    inline const String &rpcMethod() const override { return m_rpcMethod; }
    # 返回版本号
    inline int version() const override             { return m_version; }
    # 返回请求类型
    inline RequestType type() const override        { return m_type; }
    # 返回请求的来源
    inline Source source() const override           { return m_source; }
    # 标记请求已完成
    inline void done(int) override                  { m_state = STATE_DONE; }
    
    # 初始化版本号为1
    int m_version       = 1;
    # 初始化请求类型为未知
    RequestType m_type  = REQ_UNKNOWN;
    # 初始化状态为新建
    State m_state       = STATE_NEW;
    # 初始化 RPC 方法名称为空
    String m_rpcMethod;
// 声明私有成员变量 m_restricted，类型为布尔值，表示是否受限制
const bool m_restricted;
// 声明私有成员变量 m_source，类型为 Source，表示数据来源
const Source m_source;
} // namespace xmrig
#endif // XMRIG_APIREQUEST_H
```