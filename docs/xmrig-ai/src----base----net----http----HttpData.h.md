# `xmrig\src\base\net\http\HttpData.h`

```cpp
/* XMRig
 * 版权所有 (c) 2014-2019 heapwolf    <https://github.com/heapwolf>
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它，
 * 其发布由自由软件基金会发布，无论是许可证的第3版还是
 * （在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有暗示的保证适用于特定目的。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#ifndef XMRIG_HTTPDATA_H
#define XMRIG_HTTPDATA_H


#include "3rdparty/rapidjson/document.h"
#include "base/tools/Object.h"


#include <map>
#include <string>


namespace xmrig {


class HttpData
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(HttpData)

    static const std::string kApplicationJson;
    static const std::string kContentType;
    static const std::string kContentTypeL;
    static const std::string kTextPlain;


    inline HttpData(uint64_t id) : m_id(id) {}  // 构造函数，初始化 m_id
    virtual ~HttpData() = default;  // 虚析构函数

    inline const char *statusName() const   { return statusName(status); }  // 返回状态名称
    inline uint64_t id() const              { return m_id; }  // 返回 m_id

    virtual bool isRequest() const                      = 0;  // 虚函数，判断是否为请求
    virtual const char *host() const                    = 0;  // 虚函数，返回主机
    virtual const char *tlsFingerprint() const          = 0;  // 虚函数，返回 TLS 指纹
    virtual const char *tlsVersion() const              = 0;  // 虚函数，返回 TLS 版本
    virtual std::string ip() const                      = 0;  // 虚函数，返回 IP 地址
    virtual uint16_t port() const                       = 0;  // 虚函数，返回端口号
    virtual void write(std::string &&data, bool close)  = 0;  // 虚函数，写入数据并关闭
    # 检查当前对象是否为 JSON 格式
    bool isJSON() const;
    
    # 返回当前对象的方法名
    const char *methodName() const;
    
    # 返回当前对象的 JSON 文档
    rapidjson::Document json() const;
    
    # 返回指定状态码对应的状态名称
    static const char *statusName(int status);
    
    # 初始化 method 变量为 0
    int method      = 0;
    
    # 初始化 status 变量为 0
    int status      = 0;
    
    # 初始化 userType 变量为 0
    int userType    = 0;
    
    # 初始化 headers 变量为一个空的字符串到字符串的映射
    std::map<const std::string, const std::string> headers;
    
    # 初始化 body 变量为空字符串
    std::string body;
    
    # 初始化 url 变量为空字符串
    std::string url;
    
    # 初始化 rpcId 变量为 0
    uint64_t rpcId  = 0;
# 私有成员变量，表示对象的唯一标识符
private:
    const uint64_t m_id;
};


# 结束命名空间 xmrig
} // namespace xmrig


# 结束条件编译指令，关闭 XMRIG_HTTPDATA_H 文件的定义
#endif // XMRIG_HTTPDATA_H
```