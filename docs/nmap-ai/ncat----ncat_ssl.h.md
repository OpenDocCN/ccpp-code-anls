# `nmap\ncat\ncat_ssl.h`

```cpp
/* $Id$ */
#ifndef NCAT_SSL_H
#define NCAT_SSL_H

#include "ncat_config.h"

#ifdef HAVE_OPENSSL
#include <openssl/ssl.h>
#include <openssl/err.h>

#define NCAT_CA_CERTS_FILE "ca-bundle.crt"

enum {
    SHA1_BYTES = 160 / 8,
    /* 40 bytes for hex digits and 9 bytes for ' '. */
    SHA1_STRING_LENGTH = SHA1_BYTES * 2 + (SHA1_BYTES / 2 - 1)
};

/* 定义 SSL 握手状态变量，用于描述非阻塞 SSL 握手(SSL_accept())的状态 */
enum {
    NCAT_SSL_HANDSHAKE_COMPLETED      = 0,
    NCAT_SSL_HANDSHAKE_PENDING_READ   = 1,
    NCAT_SSL_HANDSHAKE_PENDING_WRITE  = 2,
    NCAT_SSL_HANDSHAKE_FAILED         = 3
};

extern SSL_CTX *setup_ssl_listen(const SSL_METHOD *method);

extern SSL *new_ssl(int fd);

extern int ssl_post_connect_check(SSL *ssl, const char *hostname);

extern char *ssl_cert_fp_str_sha1(const X509 *cert, char *strbuf, size_t len);

extern int ssl_load_default_ca_certs(SSL_CTX *ctx);

/* 尝试以非阻塞方式完成套接字 sinfo 的 SSL 握手，如果尚未完成套接字的初始化，则使用 new_ssl() 进行初始化 */
extern int ssl_handshake(struct fdinfo *sinfo);

#endif
#endif
```