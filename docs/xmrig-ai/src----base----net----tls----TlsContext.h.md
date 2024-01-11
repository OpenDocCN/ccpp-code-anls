# `xmrig\src\base\net\tls\TlsContext.h`

```
/* XMRig
 * 版权所有（C）2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有（C）2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有（C）2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发或修改它，
 * 由自由软件基金会发布，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，但没有任何保证；甚至没有适销性或特定用途的暗示保证。有关更多详细信息，请参见GNU通用公共许可证。
 *
 * 您应该已经收到GNU通用公共许可证的副本。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_TLSCONTEXT_H
#define XMRIG_TLSCONTEXT_H


#include "base/tools/Object.h"


using SSL_CTX = struct ssl_ctx_st;


namespace xmrig {


class TlsConfig;


class TlsContext
{
public:
    XMRIG_DISABLE_COPY_MOVE(TlsContext)

    ~TlsContext();

    static TlsContext *create(const TlsConfig &config);

    inline SSL_CTX *ctx() const { return m_ctx; }

private:
    TlsContext() = default;

    bool load(const TlsConfig &config);  // 加载TLS配置
    bool setCiphers(const char *ciphers);  // 设置加密算法
    bool setCipherSuites(const char *ciphersuites);  // 设置密码套件
    bool setDH(const char *dhparam);  // 设置DH参数
    void setProtocols(uint32_t protocols);  // 设置协议

    SSL_CTX *m_ctx = nullptr;
};


} // namespace xmrig


#endif // XMRIG_TLSCONTEXT_H
```