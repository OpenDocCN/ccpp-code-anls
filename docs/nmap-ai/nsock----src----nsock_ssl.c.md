# `nmap\nsock\src\nsock_ssl.c`

```
/* $Id$ */

// 包含必要的头文件
#include "nsock.h"
#include "nsock_internal.h"
#include "nsock_log.h"
#include "nsock_ssl.h"
#include "netutils.h"

// 如果使用 OpenSSL
#if HAVE_OPENSSL
// 如果 OpenSSL 版本号大于等于 3.0
#if OPENSSL_VERSION_NUMBER >= 0x30000000L
#include <openssl/provider.h>
#endif

// 安全的密码列表，排除匿名密码、低位强度密码、出口受限密码和 MD5
#define CIPHERS_SECURE "ALL:!aNULL:!eNULL:!LOW:!EXP:!RC4:!MD5:@STRENGTH"

// 快速和兼容性的密码列表，不是为了安全性
#define CIPHERS_FAST "RC4-SHA:RC4-MD5:NULL-SHA:EXP-DES-CBC-SHA:EXP-EDH-RSA-DES-CBC-SHA:EXP-RC4-MD5:NULL-MD5:EDH-RSA-DES-CBC-SHA:EXP-RC2-CBC-MD5:EDH-RSA-DES-CBC3-SHA:EXP-ADH-RC4-MD5:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA:EXP-ADH-DES-CBC-SHA:ADH-AES256-SHA:ADH-DES-CBC-SHA:ADH-RC4-MD5:AES256-SHA:DES-CBC-SHA:DES-CBC3-SHA:ADH-DES-CBC3-SHA:AES128-SHA:ADH-AES128-SHA:eNULL:ALL"

// 定义 nsock_ssl_state 的状态值
extern struct timeval nsock_tod;
#define NSOCK_SSL_STATE_UNINITIALIZED -1
#define NSOCK_SSL_STATE_INITIALIZED 1
#define NSOCK_SSL_STATE_ATEXIT 0
static int nsock_ssl_state = NSOCK_SSL_STATE_UNINITIALIZED;

// 如果 OpenSSL 版本号大于等于 1.1.0，并且不是 LibreSSL
#if OPENSSL_VERSION_NUMBER >= 0x10100000L && !defined LIBRESSL_VERSION_NUMBER
static void nsock_ssl_atexit(void)
{
  nsock_ssl_state = NSOCK_SSL_STATE_ATEXIT;
}
#endif

// 清理 SSL 相关资源
void nsp_ssl_cleanup(struct npool *nsp)
{
  // 如果 SSL 状态不是 ATEXIT
  if (nsock_ssl_state != NSOCK_SSL_STATE_ATEXIT)
  {
    // 释放 SSL 上下文
    if (nsp->sslctx != NULL)
      SSL_CTX_free(nsp->sslctx);
    // 如果支持 DTLS，释放 DTLS 上下文
#ifdef HAVE_DTLS_CLIENT_METHOD
    if (nsp->dtlsctx != NULL)
      SSL_CTX_free(nsp->dtlsctx);
#endif
  }
  // 重置 SSL 上下文和 DTLS 上下文
  nsp->sslctx = NULL;
#ifdef HAVE_DTLS_CLIENT_METHOD
  nsp->dtlsctx = NULL;
#endif
}
# 初始化 SSL 上下文，根据给定的 SSL 方法创建 SSL 上下文对象
static SSL_CTX *ssl_init_helper(const SSL_METHOD *method) {
  SSL_CTX *ctx;

  # 如果 SSL 状态为未初始化，则进行初始化
  if (nsock_ssl_state == NSOCK_SSL_STATE_UNINITIALIZED)
  {
    # 设置 SSL 状态为已初始化
    nsock_ssl_state = NSOCK_SSL_STATE_INITIALIZED;
    # 根据 OpenSSL 版本加载错误字符串和初始化 SSL 库
    # 如果版本低于 1.1.0 或者是 LibreSSL，则执行以下代码
    # 否则执行后续的代码
#if OPENSSL_VERSION_NUMBER < 0x10100000L || defined LIBRESSL_VERSION_NUMBER
    SSL_load_error_strings();
    SSL_library_init();
#else
    # 注册 OpenSSL 退出处理函数
    OPENSSL_atexit(nsock_ssl_atexit);
    # 如果 OpenSSL 版本大于等于 3.0.0，则执行以下代码
    if (NULL == OSSL_PROVIDER_load(NULL, "legacy"))
    {
      nsock_log_info("OpenSSL legacy provider failed to load: %s",
          ERR_error_string(ERR_get_error(), NULL));
    }
    if (NULL == OSSL_PROVIDER_load(NULL, "default"))
    {
      nsock_log_error("OpenSSL default provider failed to load: %s",
          ERR_error_string(ERR_get_error(), NULL));
    }
#endif
  }

  # 根据给定的 SSL 方法创建新的 SSL 上下文对象
  ctx = SSL_CTX_new(method);
  # 如果创建失败，则输出错误信息并终止程序
  if (!ctx) {
    fatal("OpenSSL failed to create a new SSL_CTX: %s",
          ERR_error_string(ERR_get_error(), NULL));
  }

  # 设置 SSL 上下文的会话缓存模式和大小
  SSL_CTX_set_session_cache_mode(ctx, SSL_SESS_CACHE_OFF|SSL_SESS_CACHE_NO_AUTO_CLEAR);
  SSL_CTX_sess_set_cache_size(ctx, 1);
  SSL_CTX_set_timeout(ctx, 3600); /* pretty unnecessary */

  # 返回创建的 SSL 上下文对象
  return ctx;
}

# 创建一个 SSL_CTX 并进行通用的初始化
static SSL_CTX *ssl_init_common() {
  return ssl_init_helper(SSLv23_client_method());
}

# 初始化 Nsock 池以创建 SSL 连接
# 设置内部 SSL_CTX，它类似于为从中创建的所有连接设置选项的模板
# 从此上下文创建的连接将仅使用安全密码，但不进行服务器证书验证
# 返回 SSL_CTX，以便您可以设置自己的选项
# 初始化 SSL 上下文，用于创建 SSL 连接
static nsock_ssl_ctx nsock_pool_ssl_init_helper(SSL_CTX *ctx, int flags) {
  char rndbuf[128];

  /* 获取随机字节，用于增加熵池的随机性，但不增加熵估计值 */
  get_random_bytes(rndbuf, sizeof(rndbuf));
  RAND_add(rndbuf, sizeof(rndbuf), 0);

  # 如果不是最大速度模式，检查 OpenSSL 的随机数生成器状态
  if (!(flags & NSOCK_SSL_MAX_SPEED)) {
    if (!RAND_status())
      fatal("%s: Failed to seed OpenSSL PRNG"
            " (RAND_status returned false).", __func__);
  }

  /* 设置 SSL 上下文的验证选项和加密算法 */
  SSL_CTX_set_verify(ctx, SSL_VERIFY_NONE, NULL);
  SSL_CTX_clear_options(ctx, SSL_OP_NO_SSLv2);
  SSL_CTX_set_options(ctx, flags & NSOCK_SSL_MAX_SPEED ?
                                  SSL_OP_ALL : SSL_OP_ALL|SSL_OP_NO_SSLv2);

  # 设置 SSL 上下文的加密算法列表
  if (!SSL_CTX_set_cipher_list(ctx, flags & NSOCK_SSL_MAX_SPEED ?
                                           CIPHERS_FAST : CIPHERS_SECURE))
    fatal("Unable to set OpenSSL cipher list: %s",
          ERR_error_string(ERR_get_error(), NULL));

  return ctx;
}

# 初始化 Nsock 池，用于创建 SSL 连接
nsock_ssl_ctx nsock_pool_ssl_init(nsock_pool ms_pool, int flags) {
  struct npool *ms = (struct npool *)ms_pool;

  # 如果 SSL 上下文为空，初始化 SSL 上下文
  if (ms->sslctx == NULL)
    ms->sslctx = ssl_init_common();
  return nsock_pool_ssl_init_helper(ms->sslctx, flags);
}

# 创建一个 SSL_CTX 并进行初始化，用于创建 DTLS 客户端
static SSL_CTX *dtls_init_common() {
  return ssl_init_helper(DTLS_client_method());
}

# 初始化 Nsock 池，用于创建 DTLS 连接
/* Initializes an Nsock pool to create DTLS connections. Very much similar to
 * nsock_pool_ssl_init, just with DTLS. */
# 初始化 DTLS 上下文
nsock_ssl_ctx nsock_pool_dtls_init(nsock_pool ms_pool, int flags) {
  SSL_CTX *dtls_ctx = NULL;
  struct npool *ms = (struct npool *)ms_pool;

  # 如果 DTLS 上下文为空，则初始化一个通用的 DTLS 上下文
  if (ms->dtlsctx == NULL)
    ms->dtlsctx = dtls_init_common();
  # 使用通用 DTLS 上下文初始化 SSL 上下文
  dtls_ctx = (SSL_CTX *) nsock_pool_ssl_init_helper(ms->dtlsctx, flags);

  /* 不要添加填充，否则 ClientHello 将会分段，无法正确连接。 */
  SSL_CTX_clear_options(dtls_ctx, SSL_OP_TLSEXT_PADDING);

  # 设置 OpenSSL 密码列表为默认值
  if (!SSL_CTX_set_cipher_list(dtls_ctx, "DEFAULT"))
    fatal("Unable to set OpenSSL cipher list: %s",
          ERR_error_string(ERR_get_error(), NULL));

  return dtls_ctx;
}

#else /* OpenSSL 版本不支持 DTLS */

# 如果没有 OpenSSL DTLS 支持，则报错
nsock_ssl_ctx nsock_pool_dtls_init(nsock_pool ms_pool, int flags) {
  fatal("%s called with no OpenSSL DTLS support", __func__);
}

#endif

/* 检查服务器证书验证，在连接建立后进行。首先检查是否提供了证书，然后调用 SSL_get_verify_result 获取验证的整体状态。
 * （仅调用 SSL_get_verify_result 是不够的，因为当没有提供证书时，该函数返回 X509_V_OK。）如果 SSL 对象的验证模式为 SSL_VERIFY_NONE，
 * 或者 OpenSSL 被禁用，则此函数始终返回 true。 */
int nsi_ssl_post_connect_verify(const nsock_iod nsockiod) {
  struct niod *iod = (struct niod *)nsockiod;

  assert(iod->ssl != NULL);
  # 如果 SSL 对象的验证模式不是 SSL_VERIFY_NONE
  if (SSL_get_verify_mode(iod->ssl) != SSL_VERIFY_NONE) {
    X509 *cert;

    # 获取对等方证书
    cert = SSL_get_peer_certificate(iod->ssl);
    if (cert == NULL)
      /* 未提供证书。 */
      return 0;

    X509_free(cert);

    # 如果验证结果不是 X509_V_OK
    if (SSL_get_verify_result(iod->ssl) != X509_V_OK)
      /* 验证出现问题。 */
      return 0;
  }
  return 1;
}

#else /* 没有 OpenSSL 支持 */

# 如果没有 OpenSSL 支持，则报错
nsock_ssl_ctx nsock_pool_ssl_init(nsock_pool ms_pool, int flags) {
  fatal("%s called with no OpenSSL support", __func__);
}
# 初始化 DTLS 连接的 SSL 上下文
nsock_ssl_ctx nsock_pool_dtls_init(nsock_pool ms_pool, int flags) {
  # 如果没有 OpenSSL 支持，则输出错误信息并终止程序
  fatal("%s called with no OpenSSL support", __func__);
}

# 在 SSL 连接建立后进行验证
int nsi_ssl_post_connect_verify(const nsock_iod nsockiod) {
  # 返回验证结果为通过
  return 1;
}

#endif
```