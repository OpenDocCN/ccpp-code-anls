# `xmrig\src\base\net\tls\TlsConfig.cpp`

```
/* XMRig
 * 版权所有（c）2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发或修改它，
 * 该许可证由自由软件基金会发布，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是基于希望它将是有用的分发的，但没有任何保证；甚至没有暗示的保证适销性或特定用途的保证。有关更多详细信息，请参见GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/net/tls/TlsConfig.h"
#include "3rdparty/rapidjson/document.h"
#include "base/io/json/Json.h"
#include "base/io/log/Log.h"
#include "base/net/tls/TlsGen.h"


namespace xmrig {


const char *TlsConfig::kCert            = "cert";
const char *TlsConfig::kEnabled         = "enabled";
const char *TlsConfig::kCertKey         = "cert_key";
const char *TlsConfig::kCiphers         = "ciphers";
const char *TlsConfig::kCipherSuites    = "ciphersuites";
const char *TlsConfig::kDhparam         = "dhparam";
const char *TlsConfig::kGen             = "gen";
const char *TlsConfig::kProtocols       = "protocols";

static const char *kTLSv1               = "TLSv1";
static const char *kTLSv1_1             = "TLSv1.1";
static const char *kTLSv1_2             = "TLSv1.2";
static const char *kTLSv1_3             = "TLSv1.3";


} // namespace xmrig
/**
 * "cert"         load TLS certificate chain from file.
 * "cert_key"     load TLS private key from file.
 * "ciphers"      set list of available ciphers (TLSv1.2 and below).
 * "ciphersuites" set list of available TLSv1.3 ciphersuites.
 * "dhparam"      load DH parameters for DHE ciphers from file.
 */
xmrig::TlsConfig::TlsConfig(const rapidjson::Value &value)
{
    // 如果传入的值是对象
    if (value.IsObject()) {
        // 从 JSON 中获取布尔值，如果不存在则使用默认值
        m_enabled = Json::getBool(value, kEnabled, m_enabled);

        // 设置协议
        setProtocols(Json::getString(value, kProtocols));
        // 加载 TLS 证书链
        setCert(Json::getString(value, kCert));
        // 加载 TLS 私钥
        setKey(Json::getString(value, kCertKey));
        // 设置可用的加密算法
        setCiphers(Json::getString(value, kCiphers));
        // 设置可用的 TLSv1.3 加密套件
        setCipherSuites(Json::getString(value, kCipherSuites));
        // 加载用于 DHE 加密的 DH 参数
        setDH(Json::getString(value, kDhparam));

        // 如果私钥为空，则使用 "cert-key" 字段的值
        if (m_key.isNull()) {
            setKey(Json::getString(value, "cert-key"));
        }

        // 如果启用了 TLS 并且配置无效，则生成新的证书
        if (m_enabled && !isValid()) {
            generate(Json::getString(value, kGen));
        }
    }
    // 如果传入的值是布尔值
    else if (value.IsBool()) {
        // 获取布尔值并设置启用状态
        m_enabled = value.GetBool();

        // 如果启用了 TLS，则生成新的证书
        if (m_enabled) {
            generate();
        }
    }
#   ifdef XMRIG_FORCE_TLS
    // 如果传入的值为空
    else if (value.IsNull()) {
        // 生成新的证书
        generate();
    }
#   endif
    // 如果传入的值是字符串
    else if (value.IsString()) {
        // 生成新的证书
        generate(value.GetString());
    }
    // 如果传入的值不是对象、布尔值或空值，则禁用 TLS
    else {
        m_enabled = false;
    }
}


bool xmrig::TlsConfig::generate(const char *commonName)
{
    // 创建 TlsGen 对象
    TlsGen gen;

    try {
        // 生成证书
        gen.generate(commonName);
    }
    catch (std::exception &ex) {
        // 捕获异常并记录错误日志
        LOG_ERR("%s", ex.what());

        return false;
    }

    // 设置证书和私钥
    setCert(gen.cert());
    setKey(gen.certKey());

    // 设置启用状态为 true
    m_enabled = true;

    return true;
}


rapidjson::Value xmrig::TlsConfig::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;

    // 获取分配器
    auto &allocator = doc.GetAllocator();
    // 创建 JSON 对象
    Value obj(kObjectType);
    // 添加启用状态到 JSON 对象
    obj.AddMember(StringRef(kEnabled), m_enabled, allocator);
    # 如果支持的协议数量大于0
    if (m_protocols > 0) {
        # 创建一个字符串向量来存储支持的协议
        std::vector<String> protocols;

        # 如果支持 TLSv1 协议
        if (m_protocols & TLSv1) {
            # 将 TLSv1 协议添加到协议列表中
            protocols.emplace_back(kTLSv1);
        }

        # 如果支持 TLSv1_1 协议
        if (m_protocols & TLSv1_1) {
            # 将 TLSv1_1 协议添加到协议列表中
            protocols.emplace_back(kTLSv1_1);
        }

        # 如果支持 TLSv1_2 协议
        if (m_protocols & TLSv1_2) {
            # 将 TLSv1_2 协议添加到协议列表中
            protocols.emplace_back(kTLSv1_2);
        }

        # 如果支持 TLSv1_3 协议
        if (m_protocols & TLSv1_3) {
            # 将 TLSv1_3 协议添加到协议列表中
            protocols.emplace_back(kTLSv1_3);
        }

        # 将协议列表添加到 JSON 对象中
        obj.AddMember(StringRef(kProtocols), String::join(protocols, ' ').toJSON(doc), allocator);
    }
    # 如果支持的协议数量为0
    else {
        # 将协议设置为 null
        obj.AddMember(StringRef(kProtocols), kNullType, allocator);
    }

    # 将证书添加到 JSON 对象中
    obj.AddMember(StringRef(kCert),         m_cert.toJSON(), allocator);
    # 将证书密钥添加到 JSON 对象中
    obj.AddMember(StringRef(kCertKey),      m_key.toJSON(), allocator);
    # 将密码添加到 JSON 对象中
    obj.AddMember(StringRef(kCiphers),      m_ciphers.toJSON(), allocator);
    # 将密码套件添加到 JSON 对象中
    obj.AddMember(StringRef(kCipherSuites), m_cipherSuites.toJSON(), allocator);
    # 将 DH 参数添加到 JSON 对象中
    obj.AddMember(StringRef(kDhparam),      m_dhparam.toJSON(), allocator);

    # 返回 JSON 对象
    return obj;
# 设置TLS协议
void xmrig::TlsConfig::setProtocols(const char *protocols)
{
    # 将传入的协议字符串以空格分隔，存储到vector中
    const std::vector<String> vec = String(protocols).split(' ');

    # 遍历vector中的每个协议
    for (const String &value : vec) {
        # 如果协议为TLSv1，则将m_protocols设置为TLSv1
        if (value == kTLSv1) {
            m_protocols |= TLSv1;
        }
        # 如果协议为TLSv1_1，则将m_protocols设置为TLSv1_1
        else if (value == kTLSv1_1) {
            m_protocols |= TLSv1_1;
        }
        # 如果协议为TLSv1_2，则将m_protocols设置为TLSv1_2
        else if (value == kTLSv1_2) {
            m_protocols |= TLSv1_2;
        }
        # 如果协议为TLSv1_3，则将m_protocols设置为TLSv1_3
        else if (value == kTLSv1_3) {
            m_protocols |= TLSv1_3;
        }
    }
}

# 设置TLS协议
void xmrig::TlsConfig::setProtocols(const rapidjson::Value &protocols)
{
    # 将m_protocols初始化为0
    m_protocols = 0;

    # 如果传入的协议为无符号整数，则调用setProtocols(protocols.GetUint())进行处理
    if (protocols.IsUint()) {
        return setProtocols(protocols.GetUint());
    }

    # 如果传入的协议为字符串，则调用setProtocols(protocols.GetString())进行处理
    if (protocols.IsString()) {
        return setProtocols(protocols.GetString());
    }
}
```