# `xmrig\src\base\net\tls\TlsGen.h`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款
 *   其版本为 3 或 (在您的选择下) 任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多细节请参见
 *   GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_TLSGEN_H
#define XMRIG_TLSGEN_H


#include "base/tools/Object.h"
#include "base/tools/String.h"


using EVP_PKEY  = struct evp_pkey_st;
using X509      = struct x509_st;


namespace xmrig {


class TlsGen
{
public:
    XMRIG_DISABLE_COPY_MOVE(TlsGen)

    // 构造函数，初始化证书和证书密钥的文件名
    TlsGen() : m_cert("cert.pem"), m_certKey("cert_key.pem") {}
    // 析构函数
    ~TlsGen();

    // 返回证书文件名
    inline const String &cert() const       { return m_cert; }
    // 返回证书密钥文件名
    inline const String &certKey() const    { return m_certKey; }

    // 生成证书和证书密钥
    void generate(const char *commonName = nullptr);

private:
    // 生成 X509 证书
    bool generate_x509(const char *commonName);
    // 写入证书和证书密钥到文件
    bool write();

    const String m_cert; // 证书文件名
    const String m_certKey; // 证书密钥文件名
    EVP_PKEY *m_pkey    = nullptr; // EVP 密钥
    X509 *m_x509        = nullptr; // X509 证书
};


} // namespace xmrig


#endif // XMRIG_TLSGEN_H
```