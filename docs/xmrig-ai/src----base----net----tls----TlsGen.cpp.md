# `xmrig\src\base\net\tls\TlsGen.cpp`

```
/* XMRig
 * 版权所有（c）2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   许可证的版本为3，或者（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有暗示的保证适用于特定目的。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/net/tls/TlsGen.h"


#include <openssl/ssl.h>
#include <openssl/err.h>
#include <stdexcept>
#include <fstream>


namespace xmrig {


static const char *kLocalhost = "localhost";


static EVP_PKEY *generate_pkey()
{
#   if OPENSSL_VERSION_NUMBER < 0x30000000L || defined(LIBRESSL_VERSION_NUMBER)
    auto pkey = EVP_PKEY_new();
    if (!pkey) {
        return nullptr;
    }

    auto exponent = BN_new();
    auto rsa      = RSA_new();

    // NOLINTNEXTLINE(cppcoreguidelines-pro-type-cstyle-cast)
    if (!exponent || !rsa || !BN_set_word(exponent, RSA_F4) || !RSA_generate_key_ex(rsa, 2048, exponent, nullptr) || !EVP_PKEY_assign_RSA(pkey, rsa)) {
        EVP_PKEY_free(pkey);
        BN_free(exponent);
        RSA_free(rsa);

        return nullptr;
    }

    BN_free(exponent);

    return pkey;
#   else
    return EVP_RSA_gen(2048);
#   endif
}


bool isFileExist(const char *fileName)
{
    std::ifstream in(fileName);

    return in.good();
}


} // namespace xmrig


xmrig::TlsGen::~TlsGen()
{
    EVP_PKEY_free(m_pkey);
    X509_free(m_x509);
}


void xmrig::TlsGen::generate(const char *commonName)
    # 如果证书文件和证书密钥文件都存在，则直接返回，不做任何操作
    if (isFileExist(m_cert) && isFileExist(m_certKey)) {
        return;
    }

    # 生成 RSA 密钥对
    m_pkey = generate_pkey();
    # 如果生成失败，则抛出运行时错误
    if (!m_pkey) {
        throw std::runtime_error("RSA key generation failed.");
    }

    # 生成 x509 证书
    # 如果 commonName 为空或长度为 0，则使用默认的 kLocalhost
    if (!generate_x509(commonName == nullptr || strlen(commonName) == 0 ? kLocalhost : commonName)) {
        throw std::runtime_error("x509 certificate generation failed.");
    }

    # 将生成的证书写入磁盘
    if (!write()) {
        throw std::runtime_error("unable to write certificate to disk.");
    }
# 生成 X.509 证书
bool xmrig::TlsGen::generate_x509(const char *commonName)
{
    # 创建一个新的 X.509 证书对象
    m_x509 = X509_new();
    # 如果对象创建失败，则返回 false
    if (!m_x509) {
        return false;
    }

    # 设置证书的公钥
    if (!X509_set_pubkey(m_x509, m_pkey)) {
        return false;
    }

    # 设置证书的序列号
    ASN1_INTEGER_set(X509_get_serialNumber(m_x509), 1);
    # 设置证书的生效时间为当前时间
    X509_gmtime_adj(X509_get_notBefore(m_x509), 0);
    # 设置证书的过期时间为当前时间后的 315360000 秒（约 10 年）
    X509_gmtime_adj(X509_get_notAfter(m_x509), 315360000L);

    # 获取证书的主题名称
    auto name = X509_get_subject_name(m_x509);
    # 向证书的主题名称中添加 commonName 字段
    X509_NAME_add_entry_by_txt(name, "CN", MBSTRING_ASC, reinterpret_cast<const uint8_t *>(commonName), -1, -1, 0);

    # 设置证书的颁发者名称
    X509_set_issuer_name(m_x509, name);

    # 对证书进行签名
    return X509_sign(m_x509, m_pkey, EVP_sha256());
}

# 将生成的证书写入文件
bool xmrig::TlsGen::write()
{
    # 打开证书私钥文件
    auto pkey_file = fopen(m_certKey, "wb");
    # 如果文件打开失败，则返回 false
    if (!pkey_file) {
        return false;
    }

    # 将私钥写入文件
    bool ret = PEM_write_PrivateKey(pkey_file, m_pkey, nullptr, nullptr, 0, nullptr, nullptr);
    # 关闭文件
    fclose(pkey_file);

    # 如果私钥写入失败，则返回 false
    if (!ret) {
        return false;
    }

    # 打开证书文件
    auto x509_file = fopen(m_cert, "wb");
    # 如果文件打开失败，则返回 false
    if (!x509_file) {
        return false;
    }

    # 将证书写入文件
    ret = PEM_write_X509(x509_file, m_x509);
    # 关闭文件
    fclose(x509_file);

    # 返回写入结果
    return ret;
}
```