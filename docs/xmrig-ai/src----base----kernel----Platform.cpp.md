# `xmrig\src\base\kernel\Platform.cpp`

```cpp
/*
 * XMRig
 * 版权所有 (c) 2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据GNU通用公共许可证的条款发布
 *   由自由软件基金会发布的许可证的第3版，或者
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/kernel/Platform.h"


#include <cstring>
#include <uv.h>


#ifdef XMRIG_FEATURE_TLS
#   include <openssl/ssl.h>
#   include <openssl/err.h>
#endif


namespace xmrig {

String Platform::m_userAgent;

} // namespace xmrig


void xmrig::Platform::init(const char *userAgent)
{
#   ifdef XMRIG_FEATURE_TLS
    // 初始化SSL库
    SSL_library_init();
    SSL_load_error_strings();

#   if OPENSSL_VERSION_NUMBER < 0x30000000L || defined(LIBRESSL_VERSION_NUMBER)
    ERR_load_BIO_strings();
    ERR_load_crypto_strings();
#   endif

    OpenSSL_add_all_digests();
#   endif

    // 如果提供了用户代理，则使用提供的用户代理，否则创建一个用户代理
    if (userAgent) {
        m_userAgent = userAgent;
    }
    else {
        m_userAgent = createUserAgent();
    }
}
```