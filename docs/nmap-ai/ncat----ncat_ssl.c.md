# `nmap\ncat\ncat_ssl.c`

```
/* $Id$ */

#include "nbase.h"  // 包含自定义的基础库头文件
#include "ncat_config.h"  // 包含自定义的ncat配置头文件
#ifdef HAVE_OPENSSL
#include "nsock.h"  // 如果有OpenSSL，则包含nsock头文件
#include "ncat.h"  // 如果有OpenSSL，则包含ncat头文件

#include <stdio.h>  // 包含标准输入输出头文件
#include <openssl/ssl.h>  // 包含OpenSSL的SSL头文件
#include <openssl/err.h>  // 包含OpenSSL的错误处理头文件
#include <openssl/rsa.h>  // 包含OpenSSL的RSA头文件
#include <openssl/rand.h>  // 包含OpenSSL的随机数生成头文件
#include <openssl/x509.h>  // 包含OpenSSL的X.509证书头文件
#include <openssl/x509v3.h>  // 包含OpenSSL的X.509证书扩展头文件

#if (OPENSSL_VERSION_NUMBER >= 0x10100000L) && !defined LIBRESSL_VERSION_NUMBER
#define HAVE_OPAQUE_STRUCTS 1  // 如果OpenSSL版本大于等于1.1.0并且不是LibreSSL，则定义HAVE_OPAQUE_STRUCTS为1
#define FUNC_ASN1_STRING_data ASN1_STRING_get0_data  // 定义FUNC_ASN1_STRING_data为ASN1_STRING_get0_data
#else
#define FUNC_ASN1_STRING_data ASN1_STRING_data  // 否则定义FUNC_ASN1_STRING_data为ASN1_STRING_data
#endif

#if OPENSSL_VERSION_NUMBER >= 0x30000000L
#include <openssl/provider.h>  // 如果OpenSSL版本大于等于3.0，则包含OpenSSL的provider头文件
/* Deprecated in OpenSSL 3.0 */
#define SSL_get_peer_certificate SSL_get1_peer_certificate  // 在OpenSSL 3.0中已弃用，定义SSL_get_peer_certificate为SSL_get1_peer_certificate
#else
#include <openssl/bn.h>  // 否则包含OpenSSL的大数头文件
#endif

/* Required for windows compilation to Eliminate APPLINK errors.
   See http://www.openssl.org/support/faq.html#PROG2 */
#ifdef WIN32
#include <openssl/applink.c>  // 如果是Windows编译，则包含OpenSSL的applink.c文件
#endif

static SSL_CTX *sslctx;  // 定义静态的SSL上下文指针sslctx

static int ssl_gen_cert(X509 **cert, EVP_PKEY **key);  // 声明静态的SSL生成证书函数，接受X509证书指针和EVP_PKEY密钥指针作为参数

/* Parameters for automatic key and certificate generation. */
enum {
    DEFAULT_KEY_BITS = 2048,  // 默认密钥位数为2048
    DEFAULT_CERT_DURATION = 60 * 60 * 24 * 365,  // 默认证书有效期为365天
};
#define CERTIFICATE_COMMENT "Automatically generated by Ncat. See https://nmap.org/ncat/."  // 定义证书注释信息

SSL_CTX *setup_ssl_listen(const SSL_METHOD *method)  // 定义SSL监听设置函数，接受SSL方法指针作为参数
{
    if (sslctx)  // 如果sslctx已经存在，则跳转到done标签
        goto done;

#if OPENSSL_VERSION_NUMBER < 0x10100000L || defined LIBRESSL_VERSION_NUMBER
    SSL_library_init();  // 初始化SSL库
    OpenSSL_add_all_algorithms();  // 添加所有算法
    ERR_load_crypto_strings();  // 加载加密字符串
    SSL_load_error_strings();  // 加载SSL错误字符串
#elif OPENSSL_VERSION_NUMBER >= 0x30000000L
  if (NULL == OSSL_PROVIDER_load(NULL, "legacy") && o.debug)  // 如果加载旧版provider失败且o.debug为真
  {
    loguser("OpenSSL legacy provider failed to load: %s",  // 记录用户日志，显示加载旧版provider失败的信息
        ERR_error_string(ERR_get_error(), NULL));
  }
  if (NULL == OSSL_PROVIDER_load(NULL, "default"))  // 如果加载默认provider失败
  {
    loguser("OpenSSL default provider failed to load: %s",  // 记录用户日志，显示加载默认provider失败的信息
        ERR_error_string(ERR_get_error(), NULL));
  }
#endif
    /* 通过各种平台相关的方法初始化随机数生成器，然后返回1（如果有足够的熵）或0（如果没有足够的熵）。这似乎是一种很好的平台无关的种子生成方法，同时也是拒绝在没有足够熵的情况下继续的方法。 */
    if (!RAND_status())
        bye("Failed to seed OpenSSL PRNG (RAND_status returned false).");

    // 如果 SSL 方法无效，则退出并打印错误信息
    if (!method)
        bye("Invalid SSL method: %s.", ERR_error_string(ERR_get_error(), NULL));
    // 创建新的 SSL 上下文对象
    if (!(sslctx = SSL_CTX_new(method)))
        bye("SSL_CTX_new(): %s.", ERR_error_string(ERR_get_error(), NULL));

    // 设置 SSL 上下文对象的选项
    SSL_CTX_set_options(sslctx, SSL_OP_ALL | SSL_OP_NO_SSLv2);

    /* 从 Nsock 中获取安全密码列表。 */
    // 如果没有指定 SSL 密码列表，则使用默认的安全密码列表
    if (o.sslciphers == NULL) {
      if (!SSL_CTX_set_cipher_list(sslctx, "ALL:!aNULL:!eNULL:!LOW:!EXP:!RC4:!MD5:@STRENGTH"))
        bye("Unable to set OpenSSL cipher list: %s", ERR_error_string(ERR_get_error(), NULL));
    }
    // 如果指定了 SSL 密码列表，则使用指定的密码列表
    else {
      if (!SSL_CTX_set_cipher_list(sslctx, o.sslciphers))
        bye("Unable to set OpenSSL cipher list: %s", ERR_error_string(ERR_get_error(), NULL));
    }
    # 如果 SSL 证书和 SSL 私钥都为空
    if (o.sslcert == NULL && o.sslkey == NULL) {
        # 定义 X509 证书和 EVP 私钥
        X509 *cert;
        EVP_PKEY *key;
        # 定义 SHA1 摘要缓冲区
        char digest_buf[SHA1_STRING_LENGTH + 1];

        # 如果设置了详细输出，打印生成临时的默认位数的 RSA 密钥的消息
        if (o.verbose)
            loguser("Generating a temporary %d-bit RSA key. Use --ssl-key and --ssl-cert to use a permanent one.\n", DEFAULT_KEY_BITS);
        # 生成 SSL 证书和私钥
        if (ssl_gen_cert(&cert, &key) == 0)
            bye("ssl_gen_cert(): %s.", ERR_error_string(ERR_get_error(), NULL));
        # 如果设置了详细输出，计算并打印 SSL 证书的 SHA-1 指纹
        if (o.verbose) {
            char *fp;
            fp = ssl_cert_fp_str_sha1(cert, digest_buf, sizeof(digest_buf));
            ncat_assert(fp == digest_buf);
            loguser("SHA-1 fingerprint: %s\n", digest_buf);
        }
        # 将 SSL 证书添加到 SSL 上下文
        if (SSL_CTX_use_certificate(sslctx, cert) != 1)
            bye("SSL_CTX_use_certificate(): %s.", ERR_error_string(ERR_get_error(), NULL));
        # 将 SSL 私钥添加到 SSL 上下文
        if (SSL_CTX_use_PrivateKey(sslctx, key) != 1)
            bye("SSL_CTX_use_PrivateKey(): %s.", ERR_error_string(ERR_get_error(), NULL));
        # 释放 SSL 证书和私钥的内存
        X509_free(cert);
        EVP_PKEY_free(key);
    } 
    # 如果 SSL 证书和 SSL 私钥不都为空
    else {
        # 如果 SSL 证书或 SSL 私钥为空，报错
        if (o.sslcert == NULL || o.sslkey == NULL)
            bye("The --ssl-key and --ssl-cert options must be used together.");
        # 将 SSL 证书链文件添加到 SSL 上下文
        if (SSL_CTX_use_certificate_chain_file(sslctx, o.sslcert) != 1)
            bye("SSL_CTX_use_certificate_chain_file(): %s.", ERR_error_string(ERR_get_error(), NULL));
        # 将 SSL 私钥文件添加到 SSL 上下文
        if (SSL_CTX_use_PrivateKey_file(sslctx, o.sslkey, SSL_FILETYPE_PEM) != 1)
            bye("SSL_CTX_use_Privatekey_file(): %s.", ERR_error_string(ERR_get_error(), NULL));
    }
# 返回 SSL 上下文
done:
    return sslctx;
}

# 创建一个新的 SSL 对象
SSL *new_ssl(int fd)
{
    SSL *ssl;

    # 如果 SSL 对象创建失败，则输出错误信息并退出
    if (!(ssl = SSL_new(sslctx)))
        bye("SSL_new(): %s.", ERR_error_string(ERR_get_error(), NULL));
    # 将 SSL 对象与文件描述符关联
    if (!SSL_set_fd(ssl, fd))
        bye("SSL_set_fd(): %s.", ERR_error_string(ERR_get_error(), NULL));

    return ssl;
}

# 匹配主机名与证书提供的名称，可能包含通配符
static int wildcard_match(const char *pattern, const char *hostname, int len)
{
    const char *p = pattern;
    const char *h = hostname;
    int remaining = len;
    # 如果长度大于1且第一个字符是*且第二个字符是.，则为通配符模式
    if (len > 1 && pattern[0] == '*' && pattern[1] == '.') {
        /* A wildcard pattern. */
        const char *dot;

        /* 跳过通配符组件 */
        p += 2;
        remaining -= 2;

        /* 确保没有更多的通配符字符 */
        if (memchr(p, '*', remaining) != NULL)
            return 0;

        /* 确保至少还有一个点，不包括末尾的点 */
        dot = (const char *) memchr(p, '.', remaining);
        if (dot == NULL /* not found */
          || dot - p == remaining /* dot in last position */
          || *(dot + 1) == '\0') /* dot immediately before null terminator */
          {
            if (o.debug > 1) {
                logdebug("Wildcard name \"%.*s\" doesn't have at least two"
                    " components after the wildcard; rejecting.\n", len, pattern);
            }
            return 0;
        }

        /* 跳过最左边的主机名组件 */
        h = strchr(hostname, '.');
        if (h == NULL)
            return 0;
        h++;

    }
    /* 比较模式和主机名的剩余部分。*/
    /* 普通的字符串比较。检查名称长度，因为我担心有人在主题中嵌入 '\0' 并与较短的名称匹配。*/
    返回剩余部分等于主机名长度并且 strncmp(p, h, remaining) == 0;
/* 匹配主机名与subjectAltName扩展字段的dNSName字段内容，如果存在的话。这是证书存储其域名的首选位置，而不是commonName字段。
   它的优势在于可以存储多个名称，因此一个证书可以匹配"example.com"和"www.example.com"。 */

static int cert_match_dnsname(X509 *cert, const char *hostname,
    unsigned int *num_checked)
{
    X509_EXTENSION *ext;  // 定义X509_EXTENSION指针ext
    STACK_OF(GENERAL_NAME) *gen_names;  // 定义GENERAL_NAME指针gen_names
    const X509V3_EXT_METHOD *method;  // 定义X509V3_EXT_METHOD指针method
    unsigned char *data;  // 定义unsigned char指针data
    int i;  // 定义整型变量i

    if (num_checked != NULL)  // 如果num_checked不为空
        *num_checked = 0;  // 将num_checked的值设为0

    i = X509_get_ext_by_NID(cert, NID_subject_alt_name, -1);  // 获取证书中NID_subject_alt_name的扩展
    if (i < 0)  // 如果获取失败
        return 0;  // 返回0
    /* 如果有多个subjectAltName扩展，则忽略 */
    if (X509_get_ext_by_NID(cert, NID_subject_alt_name, i) >= 0)  // 如果有多个subjectAltName扩展
        return 0;  // 返回0
    ext = X509_get_ext(cert, i);  // 获取证书中第i个扩展

    /* 请参阅OpenSSL源代码中的X509V3_EXT_print函数，了解从扩展中获取字符串值的方法。 */
    method = X509V3_EXT_get(ext);  // 获取扩展的方法
    if (method == NULL)  // 如果方法为空
        return 0;  // 返回0

    /* 我们必须将此地址复制到临时变量中，因为ASN1_item_d2i会对其进行递增。我们不希望它损坏ext->value->data。 */
    ASN1_OCTET_STRING* asn1_str = X509_EXTENSION_get_data(ext);  // 获取扩展数据
    data = asn1_str->data;  // 将数据赋给data
    /* 这里我们依赖于一个事实，即 NID_subject_alt_name 的内部表示（在“i2d”中）是 STACK_OF(GENERAL_NAME)。将其转换为具有 i2v 方法的 CONF_VALUE 堆栈是不令人满意的，因为 CONF_VALUE 不包含值的长度，所以无法知道空字节的存在。 */
#if (OPENSSL_VERSION_NUMBER > 0x00907000L)
    // 如果 OpenSSL 版本号大于 0x00907000L
    if (method->it != NULL) {
        // 如果方法的 it 成员不为空
        ASN1_OCTET_STRING* asn1_str_a = X509_EXTENSION_get_data(ext);
        // 获取 X509_EXTENSION 对象的数据
        gen_names = (STACK_OF(GENERAL_NAME) *) ASN1_item_d2i(NULL,
            (const unsigned char **) &data,
            asn1_str_a->length, ASN1_ITEM_ptr(method->it));
        // 使用 ASN1_item_d2i 函数解码数据并赋值给 gen_names
    } else {
        // 如果方法的 it 成员为空
        ASN1_OCTET_STRING* asn1_str_b = X509_EXTENSION_get_data(ext);
        // 获取 X509_EXTENSION 对象的数据
        gen_names = (STACK_OF(GENERAL_NAME) *) method->d2i(NULL,
            (const unsigned char **) &data,
            asn1_str_b->length);
        // 使用 method 的 d2i 函数解码数据并赋值给 gen_names
    }
#else
    // 如果 OpenSSL 版本号小于等于 0x00907000L
    gen_names = (STACK_OF(GENERAL_NAME) *) method->d2i(NULL,
        (const unsigned char **) &data,
        ext->value->length);
    // 使用 method 的 d2i 函数解码数据并赋值给 gen_names
#endif
    if (gen_names == NULL)
        // 如果 gen_names 为空，返回 0
        return 0;

    /* Look for a dNSName field with a matching hostname. There may be more than
       one dNSName field. */
    // 查找具有匹配主机名的 dNSName 字段。可能有多个 dNSName 字段。
    for (i = 0; i < sk_GENERAL_NAME_num(gen_names); i++) {
        // 遍历 gen_names 中的 GENERAL_NAME
        GENERAL_NAME *gen_name;
        // 定义 GENERAL_NAME 指针 gen_name

        gen_name = sk_GENERAL_NAME_value(gen_names, i);
        // 获取 gen_names 中第 i 个 GENERAL_NAME 的值
        if (gen_name->type == GEN_DNS) {
            // 如果 GENERAL_NAME 的类型是 GEN_DNS
            const char *dnsname = (const char *) FUNC_ASN1_STRING_data(gen_name->d.dNSName);
            // 获取 dNSName 字段的数据
            int dnslen = ASN1_STRING_length(gen_name->d.dNSName);
            // 获取 dNSName 字段的长度
            if (o.debug > 1)
                // 如果调试级别大于 1
                logdebug("Checking certificate DNS name \"%.*s\" against \"%s\".\n", dnslen, dnsname, hostname);
                // 记录日志，检查证书的 DNS 名称与主机名是否匹配
            if (num_checked != NULL)
                // 如果 num_checked 不为空
                (*num_checked)++;
                // 递增 num_checked
            if (wildcard_match(dnsname, hostname, dnslen))
                // 如果通配符匹配成功
                return 1;
                // 返回 1
        }
    }

    return 0;
    // 返回 0
}

/* Returns the number of contiguous blocks of bytes in pattern that do not
   contain the '.' byte. */
// 返回模式中不包含 '.' 字节的连续字节块的数量
static unsigned int num_components(const unsigned char *pattern, size_t len)
{
    // 定义函数 num_components，参数为 pattern 和 len
    const unsigned char *p;
    // 定义无符号字符指针 p
    unsigned int count;
    // 定义无符号整数 count

    count = 0;
    // 将 count 初始化为 0
    p = pattern;
    // 将 p 指向 pattern 的起始位置
    # 无限循环，直到条件不满足
    for (;;) {
        # 当指针p与pattern的差小于len并且指针p指向的字符为'.'时，指针p向后移动
        while (p - pattern < len && *p == '.')
            p++;
        # 如果指针p与pattern的差大于等于len，则跳出循环
        if (p - pattern >= len)
            break;
        # 当指针p与pattern的差小于len并且指针p指向的字符不为'.'时，指针p向后移动
        while (p - pattern < len && *p != '.')
            p++;
        # 计数器加一
        count++;
    }

    # 返回计数器的值
    return count;
/* 返回 true 如果模式 a 比模式 b 更严格 */
static int less_specific(const unsigned char *a, size_t a_len,
    const unsigned char *b, size_t b_len)
{
    /* 如果模式 a 包含通配符而模式 b 不包含，则模式 a 比模式 b 更不具体 */
    if (memchr(a, '*', a_len) != NULL && memchr(b, '*', b_len) == NULL)
        return 1;
    /* 如果模式 a 不包含通配符而模式 b 包含，则模式 a 比模式 b 更不具体 */
    if (memchr(a, '*', a_len) == NULL && memchr(b, '*', b_len) != NULL)
        return 0;

    return num_components(a, a_len) < num_components(b, b_len);
}

static int most_specific_commonname(X509_NAME *subject, const char **result)
{
    ASN1_STRING *best, *cur;
    int i;

    i = -1;
    best = NULL;
    while ((i = X509_NAME_get_index_by_NID(subject, NID_commonName, i)) != -1) {
        cur = X509_NAME_ENTRY_get_data(X509_NAME_get_entry(subject, i));
        /* 我们使用 "not less specific" 而不是 "more specific"，以允许后面的条目取代前面的条目 */
        if (best == NULL
            || !less_specific(FUNC_ASN1_STRING_data(cur), ASN1_STRING_length(cur),
                              FUNC_ASN1_STRING_data(best), ASN1_STRING_length(best))) {
            best = cur;
        }
    }

    if (best == NULL) {
        *result = NULL;
        return -1;
    } else {
        *result = (char *) FUNC_ASN1_STRING_data(best);
        return ASN1_STRING_length(best);
    }
}

/* 匹配主机名与证书的 "most specific" commonName 字段的内容。
   "most specific" 一词在 RFC 2818 中使用，但在我（David Fifield）找不到任何定义。
   在 Ncat 中，这是指通配符模式始终比非通配符模式更不具体。
   如果两个模式都是通配符或都是非通配符，则具有更多名称组件的模式更具体。
   如果两个名称具有相同数量的组件，则在证书中后出现的名称更具体。 */
static int cert_match_commonname(X509 *cert, const char *hostname)
    # 定义 X509_NAME 结构体指针变量 subject
    X509_NAME *subject;
    # 定义常量指针变量 commonname
    const char *commonname;
    # 定义整型变量 n
    int n;

    # 获取证书的主题信息，赋值给 subject
    subject = X509_get_subject_name(cert);
    # 如果主题信息为空，返回 0
    if (subject == NULL)
        return 0;

    # 调用函数找到最具体的 commonName，并返回找到的字符数
    n = most_specific_commonname(subject, &commonname);
    # 如果返回值小于 0 或者 commonname 为空，表示未找到 commonName，返回 0
    if (n < 0 || commonname == NULL)
        /* 未找到 commonName */
        return 0;
    # 如果 commonname 和 hostname 匹配，返回 1
    if (wildcard_match(commonname, hostname, n))
        return 1;

    # 如果 o.verbose 为真，记录证书验证错误信息
    if (o.verbose)
        loguser("Certificate verification error: Connected to \"%s\", but certificate is for \"%s\".\n", hostname, commonname);

    # 返回 0
    return 0;
}

/* 验证连接后主机的名称与其证书中的名称是否匹配。
   如果验证模式为SSL_VERIFY_NONE，则始终返回true。成功时返回非零值。 */
int ssl_post_connect_check(SSL *ssl, const char *hostname)
{
    X509 *cert = NULL;
    unsigned int num_checked;

    if (SSL_get_verify_mode(ssl) == SSL_VERIFY_NONE)
        return 1;

    if (hostname == NULL)
        return 0;

    cert = SSL_get_peer_certificate(ssl);
    if (cert == NULL)
        return 0;

    /* RFC 2818 (HTTP Over TLS): 如果存在类型为dNSName的subjectAltName扩展，则必须将其用作标识。
       否则，必须使用证书主体字段中的（最具体的）通用名称字段。尽管使用通用名称是现有的做法，但已被弃用，
       并鼓励证书颁发机构使用dNSName。 */
    if (!cert_match_dnsname(cert, hostname, &num_checked)) {
        /* 如果存在dNSNames，则完成。如果没有，则尝试commonNames。 */
        if (num_checked > 0 || !cert_match_commonname(cert, hostname)) {
            X509_free(cert);
            return 0;
        }
    }

    X509_free(cert);

    return SSL_get_verify_result(ssl) == X509_V_OK;
}

/* 生成自签名证书和匹配的RSA密钥对。此代码的参考资料是《使用OpenSSL进行网络编程》一书，第10章，第"制作证书"节；以及OpenSSL源代码中的apps/req.c。 */
static int ssl_gen_cert(X509 **cert, EVP_PKEY **key)
{
    X509_NAME *subj;
    X509_EXTENSION *ext;
    X509V3_CTX ctx;
    const char *commonName = "localhost";
    char dNSName[128];
    int rc;
#if OPENSSL_VERSION_NUMBER < 0x30000000L
    int ret = 0;
    RSA *rsa = NULL;
    BIGNUM *bne = NULL;

    *cert = NULL;
    *key = NULL;

    /* 生成私钥。 */
    *key = EVP_PKEY_new();
    if (*key == NULL)
        goto err;
    // 生成 RSA 密钥
    do {
        // 创建大数对象并设置为 RSA_F4
        bne = BN_new();
        ret = BN_set_word(bne, RSA_F4);
        // 如果设置失败，则跳转到错误处理
        if (ret != 1)
            goto err;

        // 创建 RSA 对象
        rsa = RSA_new();
        // 生成 RSA 密钥对
        ret = RSA_generate_key_ex(rsa, DEFAULT_KEY_BITS, bne, NULL);
        // 如果生成失败，则跳转到错误处理
        if (ret != 1)
            goto err;

        // 检查生成的 RSA 密钥对是否有效
        rc = RSA_check_key(rsa);
    } while (rc == 0);
    // 如果检查失败，则输出错误信息并退出
    if (rc == -1)
        bye("Error generating RSA key: %s", ERR_error_string(ERR_get_error(), NULL));
    // 将生成的 RSA 密钥对分配给 EVP_PKEY 对象
    if (EVP_PKEY_assign_RSA(*key, rsa) == 0) {
        // 如果分配失败，则释放 RSA 对象并跳转到错误处理
        RSA_free(rsa);
        goto err;
    }
#else
    // 如果条件不成立，则将指针 *cert 设置为 NULL
    *cert = NULL;
    // 生成一个默认位数的 RSA 密钥，并将其指针赋值给 *key
    *key = EVP_RSA_gen(DEFAULT_KEY_BITS);
    // 如果生成的密钥为空，则跳转到错误处理部分
    if (*key == NULL)
        goto err;
#endif

    /* 生成一个证书。*/
    // 创建一个新的 X509 证书对象，并将其指针赋值给 *cert
    *cert = X509_new();
    // 如果创建的证书为空，则跳转到错误处理部分
    if (*cert == NULL)
        goto err;
    // 设置证书版本为 3
    if (X509_set_version(*cert, 2) == 0) /* Version 3. */
        goto err;
    // 设置证书的序列号为随机生成的 31 位整数
    ASN1_INTEGER_set(X509_get_serialNumber(*cert), get_random_u32() & 0x7FFFFFFF);

    /* 设置 commonName。*/
    // 获取证书的主题信息
    subj = X509_get_subject_name(*cert);
    // 如果 o.target 不为空，则将其赋值给 commonName
    if (o.target != NULL)
        commonName = o.target;
    // 向证书的主题信息中添加 commonName 条目
    if (X509_NAME_add_entry_by_txt(subj, "commonName", MBSTRING_ASC,
        (unsigned char *) commonName, -1, -1, 0) == 0) {
        goto err;
    }

    /* 设置 dNSName。*/
    // 将 commonName 格式化为 "DNS:commonName"，并存储到 dNSName 中
    rc = Snprintf(dNSName, sizeof(dNSName), "DNS:%s", commonName);
    // 如果格式化失败或超出长度，则跳转到错误处理部分
    if (rc < 0 || rc >= sizeof(dNSName))
        goto err;
    // 设置 X509V3 上下文
    X509V3_set_ctx(&ctx, *cert, *cert, NULL, NULL, 0);
    // 使用 dNSName 创建 X509V3 扩展
    ext = X509V3_EXT_conf(NULL, &ctx, "subjectAltName", dNSName);
    // 如果创建的扩展为空，则跳转到错误处理部分
    if (ext == NULL)
        goto err;
    // 向证书添加扩展
    if (X509_add_ext(*cert, ext, -1) == 0)
        goto err;

    /* 设置一个注释。*/
    // 使用 CERTIFICATE_COMMENT 创建 X509V3 扩展
    ext = X509V3_EXT_conf(NULL, &ctx, "nsComment", CERTIFICATE_COMMENT);
    // 如果创建的扩展为空，则跳转到错误处理部分
    if (ext == NULL)
        goto err;
    // 向证书添加扩展
    if (X509_add_ext(*cert, ext, -1) == 0)
        goto err;

#if (OPENSSL_VERSION_NUMBER >= 0x10100000L) && !defined LIBRESSL_VERSION_NUMBER
    {
        // 声明两个 ASN1_TIME 类型的指针，并初始化为 NULL
        ASN1_TIME *tb, *ta;
        tb = NULL;
        ta = NULL;

        // 如果以下任一条件成立，则执行错误处理并跳转到 err 标签处
        if (X509_set_issuer_name(*cert, X509_get_subject_name(*cert)) == 0
            || (tb = ASN1_STRING_dup(X509_get0_notBefore(*cert))) == 0
            || X509_gmtime_adj(tb, 0) == 0
            || X509_set1_notBefore(*cert, tb) == 0
            || (ta = ASN1_STRING_dup(X509_get0_notAfter(*cert))) == 0
            || X509_gmtime_adj(ta, DEFAULT_CERT_DURATION) == 0
            || X509_set1_notAfter(*cert, ta) == 0
            || X509_set_pubkey(*cert, *key) == 0) {
            // 释放内存并跳转到 err 标签处
            ASN1_STRING_free(tb);
            ASN1_STRING_free(ta);
            goto err;
        }
        // 释放内存
        ASN1_STRING_free(tb);
        ASN1_STRING_free(ta);
    }
#else
    // 如果不满足条件，则执行以下代码块
    if (X509_set_issuer_name(*cert, X509_get_subject_name(*cert)) == 0
        || X509_gmtime_adj(X509_get_notBefore(*cert), 0) == 0
        || X509_gmtime_adj(X509_get_notAfter(*cert), DEFAULT_CERT_DURATION) == 0
        || X509_set_pubkey(*cert, *key) == 0) {
        // 如果上述条件有任何一个不满足，则跳转到错误处理部分
        goto err;
    }
#endif

    /* Sign it. */
    // 对证书进行签名
    if (X509_sign(*cert, *key, EVP_sha1()) == 0)
        // 如果签名失败，则跳转到错误处理部分
        goto err;

    // 返回成功标志
    return 1;

err:
    // 错误处理部分，释放资源
    if (*cert != NULL)
        X509_free(*cert);
    if (*key != NULL)
        EVP_PKEY_free(*key);

    // 返回失败标志
    return 0;
}

/* Calculate a SHA-1 fingerprint of a certificate and format it as a
   human-readable string. Returns strbuf or NULL on error. */
// 计算证书的 SHA-1 指纹并格式化为人类可读的字符串
char *ssl_cert_fp_str_sha1(const X509 *cert, char *strbuf, size_t len)
{
    unsigned char binbuf[SHA1_BYTES];
    unsigned int n;
    char *p;
    unsigned int i;

    if (len < SHA1_STRING_LENGTH + 1)
        // 如果长度不足，则返回 NULL
        return NULL;
    n = sizeof(binbuf);
    if (X509_digest(cert, EVP_sha1(), binbuf, &n) != 1)
        // 如果计算摘要失败，则返回 NULL
        return NULL;

    p = strbuf;
    for (i = 0; i < n; i++) {
        if (i > 0 && i % 2 == 0)
            *p++ = ' ';
        Snprintf(p, 3, "%02X", binbuf[i]);
        p += 2;
    }
    ncat_assert(p - strbuf <= len);
    *p = '\0';

    // 返回格式化后的字符串
    return strbuf;
}

/* Tries to complete an ssl handshake on the socket received by fdinfo struct
   if ssl is enabled on that socket. */
// 尝试在 fdinfo 结构接收到的套接字上完成 SSL 握手，如果该套接字启用了 SSL
int ssl_handshake(struct fdinfo *sinfo)
{
    int ret = 0;
    int sslerr = 0;

    if (sinfo == NULL) {
        if (o.debug)
           logdebug("ncat_ssl.c: Invoking ssl_handshake() with a NULL parameter "
                    "is a serious bug. Please fix it.\n");
        return -1;
    }

    if (!o.ssl)
        return -1;

    /* Initialize the socket too if it isn't.  */
    // 如果套接字未初始化，则进行初始化
    if (!sinfo->ssl)
        sinfo->ssl = new_ssl(sinfo->fd);

    // 尝试完成 SSL 握手
    ret = SSL_accept(sinfo->ssl);

    if (ret == 1)
        // 如果握手成功，则返回握手完成标志
        return NCAT_SSL_HANDSHAKE_COMPLETED;

    // 获取 SSL 错误码
    sslerr = SSL_get_error(sinfo->ssl, ret);
    # 如果 SSL 握手返回 -1，表示出现错误
    if (ret == -1) {
        # 如果 SSL 错误为 SSL_ERROR_WANT_READ，表示 SSL 握手需要读操作
        if (sslerr == SSL_ERROR_WANT_READ)
            # 返回 SSL 握手需要进行读操作的状态
            return NCAT_SSL_HANDSHAKE_PENDING_READ;
        # 如果 SSL 错误为 SSL_ERROR_WANT_WRITE，表示 SSL 握手需要写操作
        if (sslerr == SSL_ERROR_WANT_WRITE)
            # 返回 SSL 握手需要进行写操作的状态
            return NCAT_SSL_HANDSHAKE_PENDING_WRITE;
    }

    # 如果设置了详细输出
    if (o.verbose) {
        # 记录 SSL 连接失败的详细信息，包括远程地址和错误信息
        loguser("Failed SSL connection from %s: %s\n",
        inet_socktop(&sinfo->remoteaddr),
                     ERR_error_string(ERR_get_error(), NULL));
    }
    # 返回 SSL 握手失败的状态
    return NCAT_SSL_HANDSHAKE_FAILED;
# 结束一个条件编译指令的标记
#endif
```