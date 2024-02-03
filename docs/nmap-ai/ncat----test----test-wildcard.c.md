# `nmap\ncat\test\test-wildcard.c`

```cpp
/*
Usage: ./test-wildcard

This is a test program for the ssl_post_connect_check function. It generates
certificates with a variety of different combinations of commonNames and
dNSNames, then checks that matching names are accepted and non-matching names
are rejected. The SSL transactions happen over OpenSSL BIO pairs.
*/

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#include <openssl/bio.h>
#include <openssl/ssl.h>
#include <openssl/err.h>
#include <openssl/rsa.h>
#include <openssl/x509.h>
#include <openssl/x509v3.h>

#include "ncat_core.h"
#include "ncat_ssl.h"
#if OPENSSL_VERSION_NUMBER < 0x30000000L
#include <openssl/bn.h>
#endif

#define KEY_BITS 2048

static int tests_run = 0, tests_passed = 0;

/* A length-delimited string. */
struct lstr {
    size_t len;  // 字符串长度
    const char *s;  // 字符串内容
};

/* Make an anonymous struct lstr. */
#define LSTR(s) { sizeof(s) - 1, (s) }  // 定义匿名结构体 lstr

/* Variable-length arrays of struct lstr are terminated with a special sentinel
   value. */
#define LSTR_SENTINEL { -1, NULL }  // 结束标志
const struct lstr lstr_sentinel = LSTR_SENTINEL;  // 结束标志

int is_sentinel(const struct lstr *name) {
    return name->len == -1;  // 判断是否为结束标志
}

int ssl_post_connect_check(SSL *ssl, const char *hostname);  // SSL 连接后检查

static struct lstr *check(SSL *ssl, const struct lstr names[]);  // 检查 SSL 连接
static int ssl_ctx_trust_cert(SSL_CTX *ctx, X509 *cert);  // SSL 上下文信任证书
static int gen_cert(X509 **cert, EVP_PKEY **key,
    const struct lstr commonNames[], const struct lstr dNSNames[]);  // 生成证书
static void print_escaped(const char *s, size_t len);  // 打印转义后的字符串
static void print_array(const struct lstr array[]);  // 打印字符串数组
static int arrays_equal(const struct lstr a[], const struct lstr b[]);  // 判断两个字符串数组是否相等

/* Returns positive on success, 0 on failure. The various arrays must be
   NULL-terminated. */
static int test(const struct lstr commonNames[], const struct lstr dNSNames[],
    const struct lstr test_names[], const struct lstr expected[])
{
    SSL_CTX *server_ctx, *client_ctx;  // 服务器和客户端 SSL 上下文
    SSL *server_ssl, *client_ssl;  // 服务器和客户端 SSL 对象
    BIO *server_bio, *client_bio;  // 服务器和客户端 BIO 对象
    X509 *cert;  // X509 证书
    # 声明一个指向 EVP_PKEY 结构的指针变量 key
    EVP_PKEY *key;
    # 声明一个指向 struct lstr 结构的指针变量 results
    struct lstr *results;
    # 声明整型变量 need_accept, need_connect, passed
    int need_accept, need_connect;
    int passed;

    # 测试用例总数加一
    tests_run++;

    # 调用 gen_cert 函数生成证书和密钥，并进行断言检查返回值
    ncat_assert(gen_cert(&cert, &key, commonNames, dNSNames) == 1);

    # 创建一个双向的 BIO 对象，并进行断言检查返回值
    ncat_assert(BIO_new_bio_pair(&server_bio, 0, &client_bio, 0) == 1);

    # 创建一个 SSL 上下文对象，使用 SSLv23_server_method，并进行断言检查返回值
    server_ctx = SSL_CTX_new(SSLv23_server_method());
    ncat_assert(server_ctx != NULL);

    # 创建一个 SSL 上下文对象，使用 SSLv23_client_method，并进行断言检查返回值
    client_ctx = SSL_CTX_new(SSLv23_client_method());
    ncat_assert(client_ctx != NULL);
    # 设置客户端 SSL 上下文对象的验证模式和深度
    SSL_CTX_set_verify(client_ctx, SSL_VERIFY_PEER, NULL);
    SSL_CTX_set_verify_depth(client_ctx, 1);
    # 将证书添加到客户端 SSL 上下文对象的信任证书列表中
    ssl_ctx_trust_cert(client_ctx, cert);

    # 创建一个服务器端 SSL 对象，并进行断言检查返回值
    server_ssl = SSL_new(server_ctx);
    ncat_assert(server_ssl != NULL);
    # 设置服务器端 SSL 对象的接受状态和 BIO 对象
    SSL_set_accept_state(server_ssl);
    SSL_set_bio(server_ssl, server_bio, server_bio);
    # 使用证书和私钥配置服务器端 SSL 对象
    ncat_assert(SSL_use_certificate(server_ssl, cert) == 1);
    ncat_assert(SSL_use_PrivateKey(server_ssl, key) == 1);

    # 创建一个客户端 SSL 对象，并进行断言检查返回值
    client_ssl = SSL_new(client_ctx);
    ncat_assert(client_ssl != NULL);
    # 设置客户端 SSL 对象的连接状态和 BIO 对象
    SSL_set_connect_state(client_ssl);
    SSL_set_bio(client_ssl, client_bio, client_bio);

    # 初始化 passed 变量为 0

    # 初始化 need_accept 和 need_connect 变量为 1
    need_accept = 1;
    need_connect = 1;
    // 使用 do-while 循环，直到 need_accept 和 need_connect 都为 false
    do {
        int rc, err;

        // 如果需要进行 SSL 握手
        if (need_accept) {
            // 执行 SSL 握手
            rc = SSL_accept(server_ssl);
            // 获取 SSL 错误码
            err = SSL_get_error(server_ssl, rc);
            // 如果握手成功，将 need_accept 置为 0
            if (rc == 1) {
                need_accept = 0;
            } else {
                // 如果握手失败且错误码不是 SSL_ERROR_WANT_READ 或 SSL_ERROR_WANT_WRITE
                if (err != SSL_ERROR_WANT_READ && err != SSL_ERROR_WANT_WRITE) {
                    // 打印错误信息并跳转到 end 标签处
                    printf("SSL_accept: %s \n",
                        ERR_error_string(ERR_get_error(), NULL));
                    goto end;
                }
            }
        }
        // 如果需要进行 SSL 连接
        if (need_connect) {
            // 执行 SSL 连接
            rc = SSL_connect(client_ssl);
            // 获取 SSL 错误码
            err = SSL_get_error(client_ssl, rc);
            // 如果连接成功，将 need_connect 置为 0
            if (rc == 1) {
                need_connect = 0;
            } else {
                // 如果连接失败且错误码不是 SSL_ERROR_WANT_READ 或 SSL_ERROR_WANT_WRITE
                if (err != SSL_ERROR_WANT_READ && err != SSL_ERROR_WANT_WRITE) {
                    // 打印错误信息并跳转到 end 标签处
                    printf("SSL_connect: %s \n",
                        ERR_error_string(ERR_get_error(), NULL));
                    goto end;
                }
            }
        }
    } while (need_accept || need_connect);

    // 调用 check 函数检查 SSL 连接结果
    results = check(client_ssl, test_names);
    // 如果结果与期望值相等
    if (arrays_equal(results, expected)) {
        // 增加测试通过数，设置 passed 为 1，并打印 PASS 信息和相关数组
        tests_passed++;
        passed = 1;
        printf("PASS CN");
        print_array(commonNames);
        printf(" DNS");
        print_array(dNSNames);
        printf("\n");
    } else {
        // 否则打印 FAIL 信息和相关数组，以及实际结果和期望结果
        printf("FAIL CN");
        print_array(commonNames);
        printf(" DNS");
        print_array(dNSNames);
        printf("\n");
        printf("     got ");
        print_array(results);
        printf("\n");
        printf("expected ");
        print_array(expected);
        printf("\n");
    }
    // 释放结果数组的内存
    free(results);
end:
    // 释放证书对象占用的内存
    X509_free(cert);
    // 释放密钥对象占用的内存
    EVP_PKEY_free(key);

    // 销毁 BIO 对象
    (void) BIO_destroy_bio_pair(server_bio);

    // 释放服务器 SSL 上下文对象占用的内存
    SSL_CTX_free(server_ctx);
    // 释放客户端 SSL 上下文对象占用的内存
    SSL_CTX_free(client_ctx);

    // 释放服务器 SSL 对象占用的内存
    SSL_free(server_ssl);
    // 释放客户端 SSL 对象占用的内存
    SSL_free(client_ssl);

    // 返回测试结果
    return passed;
}

/* 返回一个以哨兵结尾的 malloc 分配的与 ssl 和 ssl_post_connect_check 匹配的名称数组 */
static struct lstr *check(SSL *ssl, const struct lstr names[])
{
    const struct lstr *name;
    struct lstr *results = NULL;
    size_t size = 0, capacity = 0;

    // 如果名称数组为空，则返回空指针
    if (names == NULL)
        return NULL;

    // 遍历名称数组
    for (name = names; !is_sentinel(name); name++) {
        // 使用 ssl_post_connect_check 函数检查 SSL 对象的连接是否匹配名称
        if (ssl_post_connect_check(ssl, name->s)) {
            // 如果结果数组大小超过容量，则重新分配内存
            if (size >= capacity) {
                capacity = (size + 1) * 2;
                results = safe_realloc(results, (capacity + 1) * sizeof(results[0]));
            }
            // 将匹配的名称添加到结果数组中
            results[size++] = *name;
        }
    }
    // 重新分配内存以适应实际大小
    results = safe_realloc(results, (size + 1) * sizeof(results[0]));
    // 在结果数组末尾添加哨兵
    results[size] = lstr_sentinel;

    return results;
}

/* 使证书对象受 SSL_CTX 信任。我找不到直接的方法来做到这一点，所以证书以 PEM 格式写入临时文件，然后使用 SSL_CTX_load_verify_locations 加载。成功返回 1，失败返回 0。 */
static int ssl_ctx_trust_cert(SSL_CTX *ctx, X509 *cert)
{
    char name[] = "ncat-test-XXXXXX";
    int fd;
    FILE *fp;
    int rc;

    // 创建临时文件
    fd = mkstemp(name);
    if (fd == -1)
        return 0;
    // 打开文件流
    fp = fdopen(fd, "w");
    if (fp == NULL) {
        close(fd);
        return 0;
    }
    // 将证书以 PEM 格式写入临时文件
    if (PEM_write_X509(fp, cert) == 0) {
        fclose(fp);
        return 0;
    }
    fclose(fp);

    // 使用 SSL_CTX_load_verify_locations 加载临时文件中的证书
    rc = SSL_CTX_load_verify_locations(ctx, name, NULL);
    if (rc == 0) {
        fprintf(stderr, "SSL_CTX_load_verify_locations: %s \n",
            ERR_error_string(ERR_get_error(), NULL));
    }
    // 删除临时文件
    if (unlink(name) == -1)
        fprintf(stderr, "unlink(\"%s\"): %s\n", name, strerror(errno));

    return rc;
}
static int set_dNSNames(X509 *cert, const struct lstr dNSNames[])
{
    STACK_OF(GENERAL_NAME) *gen_names;  // 创建一个存储GENERAL_NAME结构的栈
    GENERAL_NAME *gen_name;  // 创建一个GENERAL_NAME结构指针
    X509_EXTENSION *ext;  // 创建一个X509_EXTENSION结构指针
    const struct lstr *name;  // 创建一个指向struct lstr结构的指针

    if (dNSNames == NULL || is_sentinel(&dNSNames[0]))  // 如果dNSNames为空或者第一个元素是哨兵，则返回1
        return 1;

    /* We break the abstraction here a bit because the normal way of setting
       a list of values, using an i2v method, uses a stack of CONF_VALUE that
       doesn't contain the length of each value. We rely on the fact that
       the internal representation (the "i" in "i2d") for
       NID_subject_alt_name is STACK_OF(GENERAL_NAME). */
    // 这里我们稍微打破了抽象，因为使用i2v方法设置值的正常方式使用一个不包含每个值长度的CONF_VALUE栈。我们依赖于NID_subject_alt_name的内部表示（"i2d"中的"i"）是STACK_OF(GENERAL_NAME)。

    gen_names = sk_GENERAL_NAME_new_null();  // 创建一个空的GENERAL_NAME结构栈
    if (gen_names == NULL)  // 如果创建失败，返回0
        return 0;

    for (name = dNSNames; !is_sentinel(name); name++) {  // 遍历dNSNames数组
        gen_name = GENERAL_NAME_new();  // 创建一个新的GENERAL_NAME结构
        if (gen_name == NULL)  // 如果创建失败，跳转到stack_err标签
            goto stack_err;
        gen_name->type = GEN_DNS;  // 设置GENERAL_NAME结构的类型为GEN_DNS
#if (OPENSSL_VERSION_NUMBER >= 0x10100000L) && !defined LIBRESSL_VERSION_NUMBER
        gen_name->d.dNSName = ASN1_IA5STRING_new();  // 如果是OpenSSL版本大于等于1.1.0并且不是LibreSSL版本，创建一个新的ASN1_IA5STRING结构
#else
        gen_name->d.dNSName = M_ASN1_IA5STRING_new();  // 否则，创建一个新的M_ASN1_IA5STRING结构
#endif
        if (gen_name->d.dNSName == NULL)  // 如果创建失败，跳转到name_err标签
            goto name_err;
        if (ASN1_STRING_set(gen_name->d.dNSName, name->s, name->len) == 0)  // 设置GENERAL_NAME结构的dNSName字段
            goto name_err;
        if (sk_GENERAL_NAME_push(gen_names, gen_name) == 0)  // 将GENERAL_NAME结构压入栈中
            goto name_err;
    }
    ext = X509V3_EXT_i2d(NID_subject_alt_name, 0, gen_names);  // 将GENERAL_NAME结构栈转换为X509_EXTENSION结构
    if (ext == NULL)  // 如果转换失败，跳转到stack_err标签
        goto stack_err;
    if (X509_add_ext(cert, ext, -1) == 0) {  // 将X509_EXTENSION结构添加到证书中
        X509_EXTENSION_free(ext);
        goto stack_err;
    }
    X509_EXTENSION_free(ext);
    sk_GENERAL_NAME_pop_free(gen_names, GENERAL_NAME_free);  // 释放GENERAL_NAME结构栈

    return 1;

name_err:
    GENERAL_NAME_free(gen_name);  // 释放GENERAL_NAME结构

stack_err:
    sk_GENERAL_NAME_pop_free(gen_names, GENERAL_NAME_free);  // 释放GENERAL_NAME结构栈

    return 0;
}

static int gen_cert(X509 **cert, EVP_PKEY **key,
    const struct lstr commonNames[], const struct lstr dNSNames[])
{
#if OPENSSL_VERSION_NUMBER < 0x30000000L
    int rc, ret=0;  // 定义变量rc和ret，初始化ret为0
    # 声明 RSA 结构体指针和 BIGNUM 结构体指针，并初始化为 NULL
    RSA *rsa = NULL;
    BIGNUM *bne = NULL;

    # 初始化证书和密钥指针为 NULL
    *cert = NULL;
    *key = NULL;

    /* 生成一个私钥。*/
    # 创建一个新的 EVP_PKEY 对象
    *key = EVP_PKEY_new();
    # 如果创建失败，则跳转到错误处理
    if (*key == NULL)
        goto err;
    do {
        /* 生成 RSA 密钥。*/
        # 创建一个新的 BIGNUM 对象
        bne = BN_new();
        # 设置 BIGNUM 对象的值为 RSA_F4
        ret = BN_set_word(bne, RSA_F4);
        # 如果设置失败，则跳转到错误处理
        if (ret != 1)
            goto err;

        # 创建一个新的 RSA 对象
        rsa = RSA_new();
        # 生成指定位数的 RSA 密钥对
        ret = RSA_generate_key_ex(rsa, KEY_BITS, bne, NULL);
        # 如果生成失败，则跳转到错误处理
        if (ret != 1)
            goto err;
        /* 检查 RSA 密钥。*/
        # 检查生成的 RSA 密钥对是否有效
        rc = RSA_check_key(rsa);
    } while (rc == 0);
    # 如果检查失败，则跳转到错误处理
    if (rc == -1)
        goto err;
    # 将生成的 RSA 密钥对赋值给 EVP_PKEY 对象
    if (EVP_PKEY_assign_RSA(*key, rsa) == 0) {
        # 如果赋值失败，则释放 RSA 对象并跳转到错误处理
        RSA_free(rsa);
        goto err;
    }
#else
    // 如果条件不成立，将证书指针和密钥指针置为空
    *cert = NULL;
    *key = EVP_RSA_gen(KEY_BITS);
    // 如果生成的密钥为空，跳转到错误处理
    if (*key == NULL)
        goto err;
#endif

    /* 生成证书 */
    *cert = X509_new();
    // 如果生成的证书为空，跳转到错误处理
    if (*cert == NULL)
        goto err;
    // 设置证书版本为3
    if (X509_set_version(*cert, 2) == 0) /* Version 3. */
        goto err;
    // 设置证书序列号
    ASN1_INTEGER_set(X509_get_serialNumber(*cert), get_random_u32() & 0x7FFFFFFF);

    /* 设置 commonNames */
    if (commonNames != NULL) {
        X509_NAME *subj;
        const struct lstr *name;

        subj = X509_get_subject_name(*cert);
        // 遍历 commonNames，逐个添加到证书主题中
        for (name = commonNames; !is_sentinel(name); name++) {
            if (X509_NAME_add_entry_by_txt(subj, "commonName", MBSTRING_ASC,
                (unsigned char *) name->s, name->len, -1, 0) == 0) {
                goto err;
            }
        }
    }

    /* 设置 dNSNames */
    // 调用 set_dNSNames 函数设置 dNSNames
    if (set_dNSNames(*cert, dNSNames) == 0)
        goto err;

#if (OPENSSL_VERSION_NUMBER >= 0x10100000L) && !defined LIBRESSL_VERSION_NUMBER
    {
        ASN1_TIME *tb, *ta;
        tb = NULL;
        ta = NULL;

        // 设置颁发者名称、证书生效时间和过期时间，并设置公钥
        if (X509_set_issuer_name(*cert, X509_get_subject_name(*cert)) == 0
            || (tb = ASN1_STRING_dup(X509_get0_notBefore(*cert))) == 0
            || X509_gmtime_adj(tb, 0) == 0
            || X509_set1_notBefore(*cert, tb) == 0
            || (ta = ASN1_STRING_dup(X509_get0_notAfter(*cert))) == 0
            || X509_gmtime_adj(ta, 60) == 0
            || X509_set1_notAfter(*cert, ta) == 0
            || X509_set_pubkey(*cert, *key) == 0) {
            ASN1_STRING_free(tb);
            ASN1_STRING_free(ta);
            goto err;
        }
        ASN1_STRING_free(tb);
        ASN1_STRING_free(ta);
    }
#else
    // 设置颁发者名称、证书生效时间和过期时间，并设置公钥
    if (X509_set_issuer_name(*cert, X509_get_subject_name(*cert)) == 0
        || X509_gmtime_adj(X509_get_notBefore(*cert), 0) == 0
        || X509_gmtime_adj(X509_get_notAfter(*cert), 60) == 0
        || X509_set_pubkey(*cert, *key) == 0) {
        goto err;
    }
#endif

    /* 签名 */
    # 如果使用 X509 证书和密钥对进行签名，使用 SHA1 算法，如果签名失败则跳转到 err 标签
    if (X509_sign(*cert, *key, EVP_sha1()) == 0)
        goto err;
    # 返回签名成功的标志
    return 1;
// 如果证书指针不为空，则释放证书
err:
    if (*cert != NULL)
        X509_free(*cert);
    // 如果密钥指针不为空，则释放密钥
    if (*key != NULL)
        EVP_PKEY_free(*key);

    // 返回 0
    return 0;
}

// 打印转义后的字符串
static void print_escaped(const char *s, size_t len)
{
    int c;
    // 遍历字符串
    for ( ; len > 0; len--) {
        c = (unsigned char) *s++;
        // 如果是可打印字符且不是空白字符，则直接打印
        if (isprint(c) && !isspace(c))
            putchar(c);
        // 否则以八进制格式打印
        else
            printf("\\%03o", c);
    }
}

// 打印字符串数组
static void print_array(const struct lstr array[])
{
    const struct lstr *p;

    // 如果数组为空，则打印空数组
    if (array == NULL) {
        printf("[]");
        return;
    }
    // 打印数组起始标记
    printf("[");
    // 遍历数组
    for (p = array; !is_sentinel(p); p++) {
        // 如果不是第一个元素，则打印空格
        if (p != array)
            printf(" ");
        // 打印转义后的字符串
        print_escaped(p->s, p->len);
    }
    // 打印数组结束标记
    printf("]");
}

// 比较两个 lstr 结构是否相等
static int lstr_equal(const struct lstr *a, const struct lstr *b)
{
    // 长度和内容都相等则返回 1，否则返回 0
    return a->len == b->len && memcmp(a->s, b->s, a->len) == 0;
}

// 比较两个字符串数组是否相等
static int arrays_equal(const struct lstr a[], const struct lstr b[])
{
    // 如果其中一个数组为空，则判断另一个数组是否也为空
    if (a == NULL)
        return b == NULL;
    if (b == NULL)
        return a == NULL;
    // 遍历两个数组，逐个比较元素
    while (!is_sentinel(a) && !is_sentinel(b)) {
        // 如果有不相等的元素，则返回 0
        if (!lstr_equal(a, b))
            return 0;
        a++;
        b++;
    }

    // 如果两个数组都到达结尾标记，则返回 1，否则返回 0
    return is_sentinel(a) && is_sentinel(b);
}

/* 这只是一个常量，用于给测试用例中概念上可变长度的数组指定固定长度。如果某个数组增长过大，则增加它的长度。 */
#define ARR_LEN 10

// 测试用的字符串数组
const struct lstr test_names[] = {
    LSTR("a.com"), LSTR("www.a.com"), LSTR("sub.www.a.com"),
    LSTR("www.example.com"), LSTR("example.co.uk"), LSTR("*.*.com"),
    LSTR_SENTINEL
};

/* 这些测试只是检查匹配单个字符串是否正常工作。*/
struct {
    const struct lstr name[ARR_LEN];
    const struct lstr expected[ARR_LEN];
} single_tests[] = {
    { { LSTR_SENTINEL },
      { LSTR_SENTINEL } },
    { { LSTR("a.com"), LSTR_SENTINEL },
      { LSTR("a.com"), LSTR_SENTINEL } },
    { { LSTR("www.a.com"), LSTR_SENTINEL },
      { LSTR("www.a.com"), LSTR_SENTINEL } },
    # 匹配以 *.a.com 结尾的字符串
    { { LSTR("*.a.com"), LSTR_SENTINEL },
      { LSTR("www.a.com"), LSTR_SENTINEL } },
    # 匹配以 w 开头，以 .a.com 结尾的字符串
    { { LSTR("w*.a.com"), LSTR_SENTINEL },
      { LSTR_SENTINEL } },
    # 匹配以 *w.a.com 结尾的字符串
    { { LSTR("*w.a.com"), LSTR_SENTINEL },
      { LSTR_SENTINEL } },
    # 匹配以 www.*.com 结尾的字符串
    { { LSTR("www.*.com"), LSTR_SENTINEL },
      { LSTR_SENTINEL } },
    # 匹配以 *.com 结尾的字符串
    { { LSTR("*.com"), LSTR_SENTINEL },
      { LSTR_SENTINEL } },
    # 匹配以 *.com. 结尾的字符串
    { { LSTR("*.com."), LSTR_SENTINEL },
      { LSTR_SENTINEL } },
    # 匹配以 *.*.com 结尾的字符串
    { { LSTR("*.*.com"), LSTR_SENTINEL },
      { LSTR_SENTINEL } },
    # 匹配以 a.com 结尾或者以 evil.com 结尾的字符串
    { { LSTR("a.com\0evil.com"), LSTR_SENTINEL },
      { LSTR_SENTINEL } },
/* 这些测试不同的 commonName 和 dNSName 的组合。 */
struct {
    const struct lstr common[ARR_LEN];  // 存储 commonName 的数组
    const struct lstr dns[ARR_LEN];  // 存储 dNSName 的数组
    const struct lstr expected[ARR_LEN];  // 存储预期结果的数组
} double_tests[] = {
    /* 如果存在任何 dNSNames，则不应匹配任何 commonName。 */
    { { LSTR("a.com"), LSTR_SENTINEL },  // commonName 数组
      { LSTR("example.co.uk"), LSTR_SENTINEL },  // dNSName 数组
      { LSTR("example.co.uk"), LSTR_SENTINEL } },  // 预期结果数组
    { { LSTR("a.com"), LSTR_SENTINEL },  // commonName 数组
      { LSTR("b.com"), LSTR_SENTINEL },  // dNSName 数组
      { LSTR_SENTINEL } },  // 预期结果数组
    /* 应该针对所有的 dNSNames 进行检查。 */
    { { LSTR_SENTINEL },  // commonName 数组
      { LSTR("a.com"), LSTR("example.co.uk"), LSTR("b.com"), LSTR_SENTINEL },  // dNSName 数组
      { LSTR("a.com"), LSTR("example.co.uk"), LSTR_SENTINEL } },  // 预期结果数组
};

const struct lstr specificity_test_names[] = {
    LSTR("a.com"),  // 存储特定性测试名称的数组
    LSTR("sub.b.com"), LSTR("sub.c.com"), LSTR("sub.d.com"),  // 特定性测试名称数组的其他元素
    LSTR("sub.sub.e.com"), LSTR("sub.sub.f.com"), LSTR("sub.sub.g.com"),  // 特定性测试名称数组的其他元素
    LSTR_SENTINEL  // 特定性测试名称数组的结束标志
};

/* 验证应该只检查“最具体”的 commonName（如果存在多个）。在 RFCs 2818、4261 和 5018 中至少使用了“最具体”这个术语，但我找不到它的定义。让我们将其解释为具有最多名称元素，通配符名称被认为比所有非通配符名称更不具体。对于并列的情况，后面出现的名称被认为更具体。 */
struct {
    const struct lstr patterns[ARR_LEN];  // 存储模式的数组
    const struct lstr expected_forward;  // 存储预期前向结果
    const struct lstr expected_backward;  // 存储预期后向结果
} specificity_tests[] = {
    { { LSTR("a.com"), LSTR("*.b.com"), LSTR("sub.c.com"), LSTR("sub.d.com"), LSTR("*.sub.e.com"), LSTR("*.sub.f.com"), LSTR("sub.sub.g.com"), LSTR_SENTINEL },  // 模式数组
      LSTR("sub.sub.g.com"), LSTR("sub.sub.g.com") },  // 预期前向结果和预期后向结果
    { { LSTR("a.com"), LSTR("*.b.com"), LSTR("sub.c.com"), LSTR("sub.d.com"), LSTR("*.sub.e.com"), LSTR("*.sub.f.com"), LSTR_SENTINEL },  // 模式数组
      LSTR("sub.d.com"), LSTR("sub.c.com") },  // 预期前向结果和预期后向结果
    # 创建包含域名的列表，以及对应的通配符域名，以及特定的子域名
    { { LSTR("a.com"), LSTR("*.b.com"), LSTR("sub.c.com"), LSTR("*.sub.e.com"), LSTR("*.sub.f.com"), LSTR_SENTINEL },
      LSTR("sub.c.com"), LSTR("sub.c.com") },
    # 创建包含域名的列表，以及对应的通配符域名
    { { LSTR("a.com"), LSTR("*.b.com"), LSTR("*.sub.e.com"), LSTR("*.sub.f.com"), LSTR_SENTINEL },
      LSTR("a.com"), LSTR("a.com") },
    # 创建包含域名的列表，以及对应的通配符域名
    { { LSTR("*.b.com"), LSTR("*.sub.e.com"), LSTR("*.sub.f.com"), LSTR_SENTINEL },
      LSTR("sub.sub.f.com"), LSTR("sub.sub.e.com") },
    # 创建包含域名的列表，以及对应的通配符域名
    { { LSTR("*.b.com"), LSTR("*.sub.e.com"), LSTR_SENTINEL },
      LSTR("sub.sub.e.com"), LSTR("sub.sub.e.com") },
};

// 定义一个宏，用于计算数组元素个数
#define NELEMS(a) (sizeof(a) / sizeof(a[0]))

// 反转字符串数组
void reverse(struct lstr a[])
{
    struct lstr tmp;
    unsigned int i, j;

    i = 0;
    // 找到字符串数组中的结束标志
    for (j = 0; !is_sentinel(&a[j]); j++)
        ;
    // 如果数组为空，直接返回
    if (j == 0)
        return;
    j--;
    // 反转字符串数组
    while (i < j) {
        tmp = a[i];
        a[i] = a[j];
        a[j] = tmp;
        i++;
        j--;
    }
}

// 测试特异性
void test_specificity(const struct lstr patterns[],
    const struct lstr test_names[],
    const struct lstr expected_forward[],
    const struct lstr expected_backward[])
{
    struct lstr scratch[ARR_LEN];
    unsigned int i;

    // 将模式复制到临时数组中
    for (i = 0; i < ARR_LEN && !is_sentinel(&patterns[i]); i++)
        scratch[i] = patterns[i];
    // 断言数组未满
    ncat_assert(i < ARR_LEN);
    scratch[i] = lstr_sentinel;

    // 测试正向匹配
    test(scratch, NULL, test_names, expected_forward);
    // 反转临时数组并测试反向匹配
    reverse(scratch);
    test(scratch, NULL, test_names, expected_backward);

    return;
}

// 主函数
int main(void)
{
    unsigned int i;

    // 根据 OpenSSL 版本号初始化 SSL 库
#if OPENSSL_VERSION_NUMBER < 0x10100000L || defined LIBRESSL_VERSION_NUMBER
    SSL_library_init();
    ERR_load_crypto_strings();
    SSL_load_error_strings();
#endif

    /* Test single pattens in both the commonName and dNSName positions. */
    // 测试单个模式在 commonName 和 dNSName 位置
    for (i = 0; i < NELEMS(single_tests); i++)
        test(single_tests[i].name, NULL, test_names, single_tests[i].expected);
    for (i = 0; i < NELEMS(single_tests); i++)
        test(NULL, single_tests[i].name, test_names, single_tests[i].expected);

    // 测试双模式在 common 和 dns 位置
    for (i = 0; i < NELEMS(double_tests); i++) {
        test(double_tests[i].common, double_tests[i].dns,
            test_names, double_tests[i].expected);
    }
}
    # 遍历 specificity_tests 数组
    for (i = 0; i < NELEMS(specificity_tests); i++) {
        # 声明两个 lstr 结构体数组，用于存放期望的前向和后向名称
        struct lstr expected_forward[2], expected_backward[2];

        # 将期望的名称放入数组中，用于测试
        expected_forward[0] = specificity_tests[i].expected_forward;
        expected_forward[1] = lstr_sentinel;
        expected_backward[0] = specificity_tests[i].expected_backward;
        expected_backward[1] = lstr_sentinel;
        # 调用 test_specificity 函数进行测试
        test_specificity(specificity_tests[i].patterns,
            specificity_test_names, expected_forward, expected_backward);
    }

    # 输出测试通过的数量和总测试数量
    printf("%d / %d tests passed.\n", tests_passed, tests_run);

    # 返回测试结果，如果通过的测试数量等于总测试数量则返回 0，否则返回 1
    return tests_passed == tests_run ? 0 : 1;
# 闭合前面的函数定义
```