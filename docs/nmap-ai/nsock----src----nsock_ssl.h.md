# `nmap\nsock\src\nsock_ssl.h`

```cpp
/* $Id$ */

#ifndef NSOCK_SSL_H
#define NSOCK_SSL_H

#ifdef HAVE_CONFIG_H
#include "nsock_config.h"
#endif
#include "nsock_internal.h"

#if HAVE_OPENSSL
#include <openssl/ssl.h>
#include <openssl/err.h>
#include <openssl/rand.h>

#if OPENSSL_VERSION_NUMBER >= 0x30000000L
/* 在 OpenSSL 3.0 中已弃用 */
#define SSL_get_peer_certificate SSL_get1_peer_certificate
#endif

struct sslinfo {
  /* SSL_ERROR_NONE, SSL_ERROR_WANT_CONNECT, SSL_ERROR_WANT_READ, or
   * SSL_ERROR_WANT_WRITE */
  int ssl_desire;  // 用于存储 SSL 连接状态的变量
};

int nsi_ssl_post_connect_verify(const nsock_iod nsockiod);  // SSL 连接后的验证函数声明

void nsp_ssl_cleanup(struct npool *nsp);  // SSL 清理函数声明
#endif /* HAVE_OPENSSL */
#endif /* NSOCK_SSL_H */
```