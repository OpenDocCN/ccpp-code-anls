# `xmrig\src\base\net\http\Http.h`

```
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它能有用，
 *   但没有任何保证；甚至没有对适销性或特定用途的隐含保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参阅 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_HTTP_H
#define XMRIG_HTTP_H


#include "base/tools/String.h"


namespace xmrig {


class Http
{
public:
    static const char *kEnabled;  // 静态成员变量，表示是否启用
    static const char *kHost;     // 静态成员变量，表示主机
    static const char *kLocalhost; // 静态成员变量，表示本地主机
    static const char *kPort;     // 静态成员变量，表示端口
    static const char *kRestricted;  // 静态成员变量，表示是否受限
    static const char *kToken;    // 静态成员变量，表示令牌

    Http();  // 构造函数

    inline bool isAuthRequired() const         { return !m_restricted || !m_token.isNull(); }  // 返回是否需要身份验证
    inline bool isEnabled() const              { return m_enabled; }  // 返回是否启用
    inline bool isRestricted() const           { return m_restricted; }  // 返回是否受限
    inline const String &host() const          { return m_host; }  // 返回主机
    inline const String &token() const         { return m_token; }  // 返回令牌
    inline uint16_t port() const               { return m_port; }  // 返回端口
    inline void setEnabled(bool enabled)       { m_enabled = enabled; }  // 设置是否启用
    inline void setHost(const char *host)      { m_host = host; }  // 设置主机
    inline void setRestricted(bool restricted) { m_restricted = restricted; }  // 设置是否受限
    inline void setToken(const char *token)    { m_token = token; }  // 设置令牌

    inline bool operator!=(const Http &other) const    { return !isEqual(other); }  // 重载不等于运算符
    # 定义重载运算符，用于比较两个Http对象是否相等
    inline bool operator==(const Http &other) const    { return isEqual(other); }

    # 比较当前Http对象与另一个Http对象是否相等
    bool isEqual(const Http &other) const;

    # 将当前Http对象转换为JSON格式，并存入rapidjson::Document对象中
    rapidjson::Value toJSON(rapidjson::Document &doc) const;

    # 从JSON格式的数据中加载并设置Http对象的内容
    void load(const rapidjson::Value &http);

    # 设置Http对象的端口号
    void setPort(int port);
# 私有成员变量，表示是否启用，初始值为false
bool m_enabled      = false;
# 私有成员变量，表示是否受限制，初始值为true
bool m_restricted   = true;
# 私有成员变量，表示主机名
String m_host;
# 私有成员变量，表示令牌
String m_token;
# 私有成员变量，表示端口号，初始值为0
uint16_t m_port     = 0;
# 结束私有成员变量的定义
} // namespace xmrig
# 结束 xmrig 命名空间的定义
#endif // XMRIG_HTTP_H
# 结束头文件的定义
```