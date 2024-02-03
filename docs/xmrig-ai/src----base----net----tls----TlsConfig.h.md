# `xmrig\src\base\net\tls\TlsConfig.h`

```cpp
/*
 * XMRig
 * 版权所有（C）2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有（C）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（C）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发或修改它，
 * 由自由软件基金会发布，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，但没有任何保证；甚至没有暗示的保证适用于特定目的。
 * 有关更多详细信息，请参见GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本。
 * 如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_TLSCONFIG_H
#define XMRIG_TLSCONFIG_H


#include "3rdparty/rapidjson/fwd.h"
#include "base/tools/String.h"


namespace xmrig {


class TlsConfig
{
public:
    static const char *kCert;
    static const char *kCertKey;
    static const char *kCiphers;
    static const char *kCipherSuites;
    static const char *kDhparam;
    static const char *kEnabled;
    static const char *kGen;
    static const char *kProtocols;

    enum Versions {
        TLSv1   = 1,
        TLSv1_1 = 2,
        TLSv1_2 = 4,
        TLSv1_3 = 8
    };

    // 默认构造函数
    TlsConfig() = default;
    // 从rapidjson::Value对象构造TlsConfig对象
    TlsConfig(const rapidjson::Value &value);

    // 返回是否启用TLS
    inline bool isEnabled() const                    { return m_enabled && isValid(); }
    // 返回TLS配置是否有效
    inline bool isValid() const                      { return !m_cert.isEmpty() && !m_key.isEmpty(); }
    // 返回证书数据
    inline const char *cert() const                  { return m_cert.data(); }
    // 返回密码数据
    inline const char *ciphers() const               { return m_ciphers.isEmpty() ? nullptr : m_ciphers.data(); }
    // 返回加密套件，如果为空则返回空指针
    inline const char *cipherSuites() const          { return m_cipherSuites.isEmpty() ? nullptr : m_cipherSuites.data(); }
    // 返回 DH 参数，如果为空则返回空指针
    inline const char *dhparam() const               { return m_dhparam.isEmpty() ? nullptr : m_dhparam.data(); }
    // 返回密钥
    inline const char *key() const                   { return m_key.data(); }
    // 返回协议
    inline uint32_t protocols() const                { return m_protocols; }
    // 设置证书
    inline void setCert(const char *cert)            { m_cert = cert; }
    // 设置加密算法
    inline void setCiphers(const char *ciphers)      { m_ciphers = ciphers; }
    // 设置加密套件
    inline void setCipherSuites(const char *ciphers) { m_cipherSuites = ciphers; }
    // 设置 DH 参数
    inline void setDH(const char *dhparam)           { m_dhparam = dhparam; }
    // 设置密钥
    inline void setKey(const char *key)              { m_key = key; }
    // 设置协议
    inline void setProtocols(uint32_t protocols)     { m_protocols = protocols; }
    
    // 生成证书，commonName 默认为空
    bool generate(const char *commonName = nullptr);
    // 将对象转换为 JSON 格式
    rapidjson::Value toJSON(rapidjson::Document &doc) const;
    // 设置协议
    void setProtocols(const char *protocols);
    // 设置协议
    void setProtocols(const rapidjson::Value &protocols);
# 私有成员变量部分开始
private:
    # 布尔类型的成员变量，表示是否启用
    bool m_enabled       = true;
    # 32位无符号整数类型的成员变量，表示协议
    uint32_t m_protocols = 0;
    # 字符串类型的成员变量，表示证书
    String m_cert;
    # 字符串类型的成员变量，表示密码
    String m_ciphers;
    # 字符串类型的成员变量，表示密码套件
    String m_cipherSuites;
    # 字符串类型的成员变量，表示DH参数
    String m_dhparam;
    # 字符串类型的成员变量，表示密钥
    String m_key;
# 私有成员变量部分结束
} /* namespace xmrig */
# 结束命名空间声明
#endif /* XMRIG_TLSCONFIG_H */
# 结束头文件条件编译指令
```