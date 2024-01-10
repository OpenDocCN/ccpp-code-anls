# `nmap\ncat\http_digest.c`

```
/* $Id$ */
/* 定义了一个标识符 */

/* Nonces returned by make_nonce have the form
        timestamp-MD5(secret:timestamp)
   using representative values, this may look like
        1263929285.015273-a8e75fae174fc0e6a5df47bf9900beb6
   Sending a timestamp in the clear allows us to compute how long ago the nonce
   was issued without local state. Including microseconds reduces the chance
   that the same nonce will be issued for two different requests. When a nonce
   is received from a client, the time is extracted and then the nonce is
   recalculated locally to make sure they match. This is similar to the strategy
   recommended in section 3.2.1 of RFC 2617.
   make_nonce返回的 Nonce 的形式为 timestamp-MD5(secret:timestamp)，
   使用代表性值，可能看起来像
   1263929285.015273-a8e75fae174fc0e6a5df47bf9900beb6
   在明文中发送时间戳允许我们计算 Nonce 发行多久之前，而无需本地状态。
   包括微秒减少了相同 Nonce 被用于两个不同请求的机会。
   当从客户端接收到 Nonce 时，提取时间，然后在本地重新计算 Nonce 以确保它们匹配。
   这类似于 RFC 2617 第 3.2.1 节推荐的策略。 */

   When Ncat does Digest authentication as a client, it only does so to make a
   single CONNECT request to a proxy server. Therefore we don't use a differing
   nc (nonce count) but always the constant 00000001.
   当 Ncat 作为客户端进行摘要认证时，它只是为了向代理服务器发出单个 CONNECT 请求而这样做。
   因此，我们不使用不同的 nc (Nonce 计数)，而总是使用常量 00000001。 */

#include "ncat.h"
#include "http.h"

#include <openssl/evp.h>
#include <openssl/rand.h>

/* What's a good length for this? I think it exists only to prevent us from
   hashing known plaintext from the server. */
#define CNONCE_LENGTH 8
/* 这个长度多少合适？我认为它只是为了防止我们从服务器散列已知的明文。 */
#define SECRET_LENGTH 16
/* 密钥的长度 */

static unsigned char secret[SECRET_LENGTH];
static int secret_initialized = 0;
/* 初始化密钥和标志 */

static int append_quoted_string(char **buf, size_t *size, size_t *offset, const char *s)
{
    const char *t;

    strbuf_append_str(buf, size, offset, "\"");
    for (;;) {
        t = s;
        while (!((*t >= 0 && *t <= 31) || *t == 127 || *t == '\\'))
            t++;
        strbuf_append(buf, size, offset, s, t - s);
        if (*t == '\0')
            break;
        strbuf_sprintf(buf, size, offset, "\\%c", *t);
        s = t + 1;
    }
    strbuf_append_str(buf, size, offset, "\"");

    return *size;
}
/* 添加带引号的字符串 */

/* n is the size of src. dest must have at least n * 2 + 1 allocated bytes. */
static char *enhex(char *dest, const unsigned char *src, size_t n)
{
    unsigned int i;

    for (i = 0; i < n; i++)
        Snprintf(dest + i * 2, 3, "%02x", src[i]);

    return dest;
}
/* 将二进制数据转换为十六进制字符串 */
/* 初始化用于生成随机数的服务器密钥。在失败时返回-1。 */
int http_digest_init_secret(void)
{
    // 检查随机数生成器是否可用
    if (!RAND_status())
        return -1;
    // 生成随机的密钥
    if (RAND_bytes(secret, sizeof(secret)) != 1)
        return -1;
    // 标记密钥已初始化
    secret_initialized = 1;

    return 0;
}

#if OPENSSL_VERSION_NUMBER < 0x10100000L
#define EVP_MD_CTX_new EVP_MD_CTX_create
#define EVP_MD_CTX_free EVP_MD_CTX_destroy
#endif

// 生成一个随机数
static char *make_nonce(const struct timeval *tv)
{
    char *buf = NULL;
    size_t size = 0, offset = 0;
    EVP_MD_CTX *md5;
    unsigned char hashbuf[EVP_MAX_MD_SIZE];
    char hash_hex[EVP_MAX_MD_SIZE * 2 + 1];
    char time_buf[32];
    unsigned int hash_size = 0;

    /* 如果忘记调用http_digest_init_secret，则会崩溃。 */
    if (!secret_initialized)
        bye("Server secret not initialized for Digest authentication. Call http_digest_init_secret.");

    // 格式化时间
    Snprintf(time_buf, sizeof(time_buf), "%lu.%06lu",
        (long unsigned) tv->tv_sec, (long unsigned) tv->tv_usec);
    // 创建MD5上下文
    md5 = EVP_MD_CTX_new();
    // 初始化MD5上下文
    EVP_DigestInit_ex(md5, EVP_md5(), NULL);
    // 更新MD5上下文的数据
    EVP_DigestUpdate(md5, secret, sizeof(secret));
    EVP_DigestUpdate(md5, ":", 1);
    EVP_DigestUpdate(md5, time_buf, strlen(time_buf));
    // 完成MD5哈希计算
    EVP_DigestFinal_ex(md5, hashbuf, &hash_size);
    // 将哈希值转换为十六进制字符串
    enhex(hash_hex, hashbuf, hash_size);

    // 格式化随机数
    strbuf_sprintf(&buf, &size, &offset, "%s-%s", time_buf, hash_hex);
    // 释放MD5上下文
    EVP_MD_CTX_free(md5);

    return buf;
}

/* 假设参数都不为空，除了nc和cnonce，如果qop == QOP_NONE，则它们可以是垃圾值。 */
static void make_response(char buf[EVP_MAX_MD_SIZE * 2 + 1],
    const char *username, const char *realm, const char *password,
    const char *method, const char *uri, const char *nonce,
    enum http_digest_qop qop, const char *nc, const char *cnonce)
{
    char HA1_hex[EVP_MAX_MD_SIZE * 2 + 1], HA2_hex[EVP_MAX_MD_SIZE * 2 + 1];
    unsigned char hashbuf[EVP_MAX_MD_SIZE];
    EVP_MD_CTX *md5;
    unsigned int hash_size = 0;
    # 使用 EVP_md5() 函数获取 MD5 摘要算法的指针
    const EVP_MD *md = EVP_md5();

    # 计算 H(A1) 的摘要
    md5 = EVP_MD_CTX_new();
    EVP_DigestInit_ex(md5, md, NULL);
    EVP_DigestUpdate(md5, username, strlen(username));
    EVP_DigestUpdate(md5, ":", 1);
    EVP_DigestUpdate(md5, realm, strlen(realm));
    EVP_DigestUpdate(md5, ":", 1);
    EVP_DigestUpdate(md5, password, strlen(password));
    EVP_DigestFinal_ex(md5, hashbuf, &hash_size);
    enhex(HA1_hex, hashbuf, hash_size);

    # 计算 H(A2) 的摘要
    EVP_DigestInit_ex(md5, md, NULL);
    EVP_DigestUpdate(md5, method, strlen(method));
    EVP_DigestUpdate(md5, ":", 1);
    EVP_DigestUpdate(md5, uri, strlen(uri));
    EVP_DigestFinal_ex(md5, hashbuf, &hash_size);
    enhex(HA2_hex, hashbuf, hash_size);

    # 计算响应的摘要
    EVP_DigestInit_ex(md5, md, NULL);
    EVP_DigestUpdate(md5, HA1_hex, strlen(HA1_hex));
    EVP_DigestUpdate(md5, ":", 1);
    EVP_DigestUpdate(md5, nonce, strlen(nonce));
    if (qop == QOP_AUTH) {
        EVP_DigestUpdate(md5, ":", 1);
        EVP_DigestUpdate(md5, nc, strlen(nc));
        EVP_DigestUpdate(md5, ":", 1);
        EVP_DigestUpdate(md5, cnonce, strlen(cnonce));
        EVP_DigestUpdate(md5, ":", 1);
        EVP_DigestUpdate(md5, "auth", strlen("auth"));
    }
    EVP_DigestUpdate(md5, ":", 1);
    EVP_DigestUpdate(md5, HA2_hex, strlen(HA2_hex));
    EVP_DigestFinal_ex(md5, hashbuf, &hash_size);

    # 将摘要转换为十六进制字符串
    enhex(buf, hashbuf, hash_size);
    # 释放 MD5 上下文
    EVP_MD_CTX_free(md5);
/* 从一个随机数中提取发行时间（不检查有效性的其他方面）。如果无法提取时间，则返回-1，否则返回0。 */
int http_digest_nonce_time(const char *nonce, struct timeval *tv)
{
    unsigned long sec, usec;

    // 从随机数中提取秒和微秒，如果无法提取则返回-1
    if (sscanf(nonce, "%lu.%lu", &sec, &usec) != 2)
        return -1;

    // 将提取的秒和微秒赋值给tv结构体
    tv->tv_sec = sec;
    tv->tv_usec = usec;

    return 0;
}

// 生成代理身份验证的头部信息
char *http_digest_proxy_authenticate(const char *realm, int stale)
{
    char *buf = NULL;
    size_t size = 0, offset = 0;
    struct timeval tv;
    char *nonce;

    // 获取当前时间
    if (gettimeofday(&tv, NULL) == -1)
        return NULL;

    // 添加认证领域到buf中
    strbuf_append_str(&buf, &size, &offset, "Digest realm=");
    append_quoted_string(&buf, &size, &offset, realm);

    // 生成随机数
    nonce = make_nonce(&tv);
    // 添加随机数到buf中
    strbuf_append_str(&buf, &size, &offset, ", nonce=");
    append_quoted_string(&buf, &size, &offset, nonce);
    free(nonce);
    // 添加qop到buf中
    strbuf_append_str(&buf, &size, &offset, ", qop=\"auth\"");

    // 如果stale为真，添加stale到buf中
    if (stale)
        strbuf_append_str(&buf, &size, &offset, ", stale=true");

    return buf;
}

// 生成代理授权头部信息
char *http_digest_proxy_authorization(const struct http_challenge *challenge,
    const char *username, const char *password,
    const char *method, const char *uri)
{
    /* For now we authenticate successfully at most once, so we don't need a
       varying client nonce count. */
    static const u32 nc = 0x00000001;

    char response_hex[EVP_MAX_MD_SIZE * 2 + 1];
    unsigned char cnonce[CNONCE_LENGTH];
    char cnonce_buf[CNONCE_LENGTH * 2 + 1];
    char nc_buf[8 + 1];
    char *buf = NULL;
    size_t size = 0, offset = 0;
    enum http_digest_qop qop;

    // 检查挑战的方案、领域、随机数和算法是否满足条件，如果不满足则返回NULL
    if (challenge->scheme != AUTH_DIGEST
        || challenge->realm == NULL
        || challenge->digest.nonce == NULL
        || challenge->digest.algorithm != ALGORITHM_MD5)
        return NULL;
}
    # 如果挑战的摘要包含 QOP_AUTH，则执行以下操作
    if (challenge->digest.qop & QOP_AUTH) {
        # 将 nc 转换为 8 位十六进制字符串
        Snprintf(nc_buf, sizeof(nc_buf), "%08x", nc);
        # 检查随机数发生器状态，如果状态不足够强，则返回 NULL
        if (!RAND_status())
            return NULL;
        # 生成随机数填充 cnonce 缓冲区，如果生成失败则返回 NULL
        if (RAND_bytes(cnonce, sizeof(cnonce)) != 1)
            return NULL;
        # 将 cnonce 转换为十六进制字符串
        enhex(cnonce_buf, cnonce, sizeof(cnonce));
        # 设置 qop 为 QOP_AUTH
        qop = QOP_AUTH;
    } else {
        # 如果挑战的摘要不包含 QOP_AUTH，则将 qop 设置为 QOP_NONE
        qop = QOP_NONE;
    }

    # 将 " Digest" 添加到 buf 中
    strbuf_append_str(&buf, &size, &offset, " Digest");
    # 将 " username=" 添加到 buf 中
    strbuf_append_str(&buf, &size, &offset, " username=");
    # 将 username 以引号括起来添加到 buf 中
    append_quoted_string(&buf, &size, &offset, username);
    # 将 " realm=" 添加到 buf 中
    strbuf_append_str(&buf, &size, &offset, ", realm=");
    # 将挑战的 realm 以引号括起来添加到 buf 中
    append_quoted_string(&buf, &size, &offset, challenge->realm);
    # 将 " nonce=" 添加到 buf 中
    strbuf_append_str(&buf, &size, &offset, ", nonce=");
    # 将挑战的摘要中的 nonce 以引号括起来添加到 buf 中
    append_quoted_string(&buf, &size, &offset, challenge->digest.nonce);
    # 将 " uri=" 添加到 buf 中
    strbuf_append_str(&buf, &size, &offset, ", uri=");
    # 将 uri 以引号括起来添加到 buf 中
    append_quoted_string(&buf, &size, &offset, uri);

    # 如果 qop 为 QOP_AUTH，则执行以下操作
    if (qop == QOP_AUTH) {
        # 将 " qop=auth" 添加到 buf 中
        strbuf_append_str(&buf, &size, &offset, ", qop=auth");
        # 将 " cnonce=" 添加到 buf 中
        strbuf_append_str(&buf, &size, &offset, ", cnonce=");
        # 将 cnonce_buf 以引号括起来添加到 buf 中
        append_quoted_string(&buf, &size, &offset, cnonce_buf);
        # 将 nc_buf 格式化后添加到 buf 中
        strbuf_sprintf(&buf, &size, &offset, ", nc=%s", nc_buf);
    }

    # 生成响应的十六进制字符串
    make_response(response_hex, username, challenge->realm, password,
        method, uri, challenge->digest.nonce, qop, nc_buf, cnonce_buf);
    # 将 " response=" 添加到 buf 中
    strbuf_append_str(&buf, &size, &offset, ", response=");
    # 将 response_hex 以引号括起来添加到 buf 中
    append_quoted_string(&buf, &size, &offset, response_hex);

    # 如果挑战的摘要中包含 opaque，则执行以下操作
    if (challenge->digest.opaque != NULL) {
        # 将 " opaque=" 添加到 buf 中
        strbuf_append_str(&buf, &size, &offset, ", opaque=");
        # 将挑战的摘要中的 opaque 以引号括起来添加到 buf 中
        append_quoted_string(&buf, &size, &offset, challenge->digest.opaque);
    }

    # 将换行符添加到 buf 中
    strbuf_append_str(&buf, &size, &offset, "\r\n");

    # 返回填充好数据的 buf
    return buf;
/* 检查一个随机数是否是我们发出的，并且响应是否是预期的。这不会检查随机数的生命周期。 */
int http_digest_check_credentials(const char *username, const char *realm,
    const char *password, const char *method,
    const struct http_credentials *credentials)
{
    char response_hex[EVP_MAX_MD_SIZE * 2 + 1];  // 用于存储响应的十六进制字符串
    struct timeval tv;  // 用于存储时间信息的结构体
    char *nonce;  // 用于存储随机数的指针

    // 检查认证方案是否为摘要认证，以及各个字段是否为空或算法是否为MD5
    if (credentials->scheme != AUTH_DIGEST
        || credentials->u.digest.username == NULL
        || credentials->u.digest.realm == NULL
        || credentials->u.digest.nonce == NULL
        || credentials->u.digest.uri == NULL
        || credentials->u.digest.response == NULL
        || credentials->u.digest.algorithm != ALGORITHM_MD5) {
        return 0;  // 返回0表示认证失败
    }
    // 如果qop不是NONE或AUTH，则返回0
    if (credentials->u.digest.qop != QOP_NONE && credentials->u.digest.qop != QOP_AUTH)
        return 0;
    // 如果qop是AUTH且nc或cnonce为空，则返回0
    if (credentials->u.digest.qop == QOP_AUTH
        && (credentials->u.digest.nc == NULL
            || credentials->u.digest.cnonce == NULL)) {
        return 0;
    }

    // 如果用户名和领域与凭据中的不匹配，则返回0
    if (strcmp(username, credentials->u.digest.username) != 0)
        return 0;
    if (strcmp(realm, credentials->u.digest.realm) != 0)
        return 0;

    // 如果随机数的时间不合法，则返回0
    if (http_digest_nonce_time(credentials->u.digest.nonce, &tv) == -1)
        return 0;

    // 生成一个新的随机数，并与凭据中的随机数进行比较
    nonce = make_nonce(&tv);
    if (strcmp(nonce, credentials->u.digest.nonce) != 0) {
        /* 我们不可能分发这个随机数。 */
        free(nonce);
        return 0;
    }
    free(nonce);

    // 生成响应的十六进制字符串，并与凭据中的响应进行比较
    make_response(response_hex, credentials->u.digest.username, realm,
        password, method, credentials->u.digest.uri,
        credentials->u.digest.nonce, credentials->u.digest.qop,
        credentials->u.digest.nc, credentials->u.digest.cnonce);

    // 返回响应是否与凭据中的响应相同
    return strcmp(response_hex, credentials->u.digest.response) == 0;
}
```