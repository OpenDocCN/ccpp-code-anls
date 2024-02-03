# `nmap\libssh2\src\libgcrypt.c`

```cpp
/*
 * 版权声明，版权所有
 * 允许在源代码和二进制形式下重新分发和使用，需满足以下条件：
 *   1. 源代码的再分发必须保留上述版权声明、条件列表和下面的免责声明
 *   2. 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和下面的免责声明
 *   3. 未经特定事先书面许可，不得使用版权所有者的名称或其他贡献者的名称来认可或推广从本软件派生的产品
 * 本软件由版权所有者和贡献者"按原样"提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途适用性的暗示担保。无论在任何情况下，版权所有者或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责，无论是在合同、严格责任或侵权行为（包括疏忽或其他方式）的任何理论下产生的，即使已被告知可能发生此类损害。
 */

#include "libssh2_priv.h"

#ifdef LIBSSH2_LIBGCRYPT /* 只有在构建时使用libgcrypt时才编译 */

#include <string.h>

int
# 创建一个新的 RSA 密钥对
_libssh2_rsa_new(libssh2_rsa_ctx ** rsa,
                 const unsigned char *edata,
                 unsigned long elen,
                 const unsigned char *ndata,
                 unsigned long nlen,
                 const unsigned char *ddata,
                 unsigned long dlen,
                 const unsigned char *pdata,
                 unsigned long plen,
                 const unsigned char *qdata,
                 unsigned long qlen,
                 const unsigned char *e1data,
                 unsigned long e1len,
                 const unsigned char *e2data,
                 unsigned long e2len,
                 const unsigned char *coeffdata, unsigned long coefflen)
{
    int rc;
    (void) e1data;  # 忽略参数 e1data
    (void) e1len;   # 忽略参数 e1len
    (void) e2data;  # 忽略参数 e2data
    (void) e2len;   # 忽略参数 e2len

    # 如果存在私钥数据，则构建包含私钥的 RSA 密钥对
    if(ddata) {
        rc = gcry_sexp_build
            (rsa, NULL,
             "(private-key(rsa(n%b)(e%b)(d%b)(q%b)(p%b)(u%b)))",
             nlen, ndata, elen, edata, dlen, ddata, plen, pdata,
             qlen, qdata, coefflen, coeffdata);
    }
    # 否则，构建只包含公钥的 RSA 密钥对
    else {
        rc = gcry_sexp_build(rsa, NULL, "(public-key(rsa(n%b)(e%b)))",
                             nlen, ndata, elen, edata);
    }
    # 如果构建密钥对失败，则返回错误
    if(rc) {
        *rsa = NULL;
        return -1;
    }

    return 0;
}

# 使用 RSA 公钥验证 SHA1 签名
int
_libssh2_rsa_sha1_verify(libssh2_rsa_ctx * rsa,
                         const unsigned char *sig,
                         unsigned long sig_len,
                         const unsigned char *m, unsigned long m_len)
{
    unsigned char hash[SHA_DIGEST_LENGTH];
    gcry_sexp_t s_sig, s_hash;
    int rc = -1;

    # 计算消息 m 的 SHA1 哈希值
    libssh2_sha1(m, m_len, hash);

    # 构建包含 SHA1 哈希值的数据结构
    rc = gcry_sexp_build(&s_hash, NULL,
                         "(data (flags pkcs1) (hash sha1 %b))",
                         SHA_DIGEST_LENGTH, hash);
    # 如果构建失败，则返回错误
    if(rc != 0) {
        return -1;
    }

    # 构建包含 RSA 签名的数据结构
    rc = gcry_sexp_build(&s_sig, NULL, "(sig-val(rsa(s %b)))", sig_len, sig);
    # 如果构建失败，则释放之前构建的数据结构并返回错误
    if(rc != 0) {
        gcry_sexp_release(s_hash);
        return -1;
    }
    # 使用公钥验证签名
    rc = gcry_pk_verify(s_sig, s_hash, rsa);
    # 释放签名数据的 S 表达式
    gcry_sexp_release(s_sig);
    # 释放哈希数据的 S 表达式
    gcry_sexp_release(s_hash);

    # 如果验证结果为 0，则返回 0；否则返回 -1
    return (rc == 0) ? 0 : -1;
# 定义一个函数，用于创建一个新的 DSA 上下文
int
_libssh2_dsa_new(libssh2_dsa_ctx ** dsactx,
                 const unsigned char *p,
                 unsigned long p_len,
                 const unsigned char *q,
                 unsigned long q_len,
                 const unsigned char *g,
                 unsigned long g_len,
                 const unsigned char *y,
                 unsigned long y_len,
                 const unsigned char *x, unsigned long x_len)
{
    int rc;

    # 如果私钥长度不为0，则使用私钥创建 DSA 上下文
    if(x_len) {
        rc = gcry_sexp_build
            (dsactx, NULL,
             "(private-key(dsa(p%b)(q%b)(g%b)(y%b)(x%b)))",
             p_len, p, q_len, q, g_len, g, y_len, y, x_len, x);
    }
    # 否则使用公钥创建 DSA 上下文
    else {
        rc = gcry_sexp_build(dsactx, NULL,
                             "(public-key(dsa(p%b)(q%b)(g%b)(y%b)))",
                             p_len, p, q_len, q, g_len, g, y_len, y);
    }

    # 如果创建上下文失败，则将上下文置为NULL并返回-1
    if(rc) {
        *dsactx = NULL;
        return -1;
    }

    # 创建成功则返回0
    return 0;
}

# 从内存中创建一个新的 RSA 私钥
int
_libssh2_rsa_new_private_frommemory(libssh2_rsa_ctx ** rsa,
                                    LIBSSH2_SESSION * session,
                                    const char *filedata, size_t filedata_len,
                                    unsigned const char *passphrase)
{
    # 返回不支持的错误信息
    return _libssh2_error(session, LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                         "Unable to extract private key from memory: "
                         "Method unimplemented in libgcrypt backend");
}

# 从文件中创建一个新的 RSA 私钥
int
_libssh2_rsa_new_private(libssh2_rsa_ctx ** rsa,
                         LIBSSH2_SESSION * session,
                         const char *filename, unsigned const char *passphrase)
{
    FILE *fp;
    unsigned char *data, *save_data;
    unsigned int datalen;
    int ret;
    unsigned char *n, *e, *d, *p, *q, *e1, *e2, *coeff;
    unsigned int nlen, elen, dlen, plen, qlen, e1len, e2len, coefflen;

    # 打开文件
    fp = fopen(filename, FOPEN_READTEXT);
    # 如果文件打开失败，则返回-1
    if(!fp) {
        return -1;
    }
}
    # 调用_libssh2_pem_parse函数解析PEM格式的RSA私钥
    ret = _libssh2_pem_parse(session,
                             "-----BEGIN RSA PRIVATE KEY-----",
                             "-----END RSA PRIVATE KEY-----",
                             passphrase,
                             fp, &data, &datalen);
    # 关闭文件指针
    fclose(fp);
    # 如果解析失败，返回-1
    if(ret) {
        return -1;
    }

    # 保存解析后的数据
    save_data = data;

    # 解码PEM格式的数据
    if(_libssh2_pem_decode_sequence(&data, &datalen)) {
        # 如果解码失败，返回-1，并跳转到fail标签处
        ret = -1;
        goto fail;
    }
# 首先读取版本字段（应该为0）
ret = _libssh2_pem_decode_integer(&data, &datalen, &n, &nlen);
# 如果返回值不为0，或者nlen不为1且n不为'\0'，则将ret设为-1，跳转到fail标签处
if(ret != 0 || (nlen != 1 && *n != '\0')) {
    ret = -1;
    goto fail;
}

# 依次解析密钥的各个部分，如果返回值不为0，则将ret设为-1，跳转到fail标签处
ret = _libssh2_pem_decode_integer(&data, &datalen, &n, &nlen);
if(ret != 0) {
    ret = -1;
    goto fail;
}

ret = _libssh2_pem_decode_integer(&data, &datalen, &e, &elen);
if(ret != 0) {
    ret = -1;
    goto fail;
}

ret = _libssh2_pem_decode_integer(&data, &datalen, &d, &dlen);
if(ret != 0) {
    ret = -1;
    goto fail;
}

ret = _libssh2_pem_decode_integer(&data, &datalen, &p, &plen);
if(ret != 0) {
    ret = -1;
    goto fail;
}

ret = _libssh2_pem_decode_integer(&data, &datalen, &q, &qlen);
if(ret != 0) {
    ret = -1;
    goto fail;
}

ret = _libssh2_pem_decode_integer(&data, &datalen, &e1, &e1len);
if(ret != 0) {
    ret = -1;
    goto fail;
}

ret = _libssh2_pem_decode_integer(&data, &datalen, &e2, &e2len);
if(ret != 0) {
    ret = -1;
    goto fail;
}

ret = _libssh2_pem_decode_integer(&data, &datalen, &coeff, &coefflen);
if(ret != 0) {
    ret = -1;
    goto fail;
}

# 使用解析出的各个部分创建RSA密钥，如果失败则将ret设为-1，跳转到fail标签处
if(_libssh2_rsa_new(rsa, e, elen, n, nlen, d, dlen, p, plen,
                     q, qlen, e1, e1len, e2, e2len, coeff, coefflen)) {
    ret = -1;
    goto fail;
}

# 执行到这里表示成功，将ret设为0
ret = 0;

# fail标签处，释放内存并返回ret
fail:
LIBSSH2_FREE(session, save_data);
return ret;
}

# 从内存中读取DSA私钥
int
_libssh2_dsa_new_private_frommemory(libssh2_dsa_ctx ** dsa,
                                    LIBSSH2_SESSION * session,
                                    const char *filedata, size_t filedata_len,
                                    unsigned const char *passphrase)
    # 返回一个表示无法从内存中提取私钥的错误信息
    return _libssh2_error(session, LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                         "Unable to extract private key from memory: "
                         "Method unimplemented in libgcrypt backend");
}

int
_libssh2_dsa_new_private(libssh2_dsa_ctx ** dsa,
                         LIBSSH2_SESSION * session,
                         const char *filename, unsigned const char *passphrase)
{
    FILE *fp;  // 声明文件指针
    unsigned char *data, *save_data;  // 声明存储数据的指针
    unsigned int datalen;  // 声明数据长度
    int ret;  // 声明返回值
    unsigned char *p, *q, *g, *y, *x;  // 声明用于存储整数的指针
    unsigned int plen, qlen, glen, ylen, xlen;  // 声明整数长度

    fp = fopen(filename, FOPEN_READTEXT);  // 打开文件
    if(!fp) {  // 如果文件指针为空
        return -1;  // 返回错误值
    }

    ret = _libssh2_pem_parse(session,  // 调用_libssh2_pem_parse函数
                             "-----BEGIN DSA PRIVATE KEY-----",  // 指定开始标记
                             "-----END DSA PRIVATE KEY-----",  // 指定结束标记
                             passphrase,  // 传入密码
                             fp, &data, &datalen);  // 传入文件指针和数据指针
    fclose(fp);  // 关闭文件
    if(ret) {  // 如果返回值不为0
        return -1;  // 返回错误值
    }

    save_data = data;  // 保存数据指针

    if(_libssh2_pem_decode_sequence(&data, &datalen)) {  // 调用_libssh2_pem_decode_sequence函数
        ret = -1;  // 返回错误值
        goto fail;  // 跳转到失败标签
    }

    /* First read Version field (should be 0). */
    ret = _libssh2_pem_decode_integer(&data, &datalen, &p, &plen);  // 调用_libssh2_pem_decode_integer函数
    if(ret != 0 || (plen != 1 && *p != '\0')) {  // 如果返回值不为0或者长度不为1且值不为0
        ret = -1;  // 返回错误值
        goto fail;  // 跳转到失败标签
    }

    ret = _libssh2_pem_decode_integer(&data, &datalen, &p, &plen);  // 调用_libssh2_pem_decode_integer函数
    if(ret != 0) {  // 如果返回值不为0
        ret = -1;  // 返回错误值
        goto fail;  // 跳转到失败标签
    }

    ret = _libssh2_pem_decode_integer(&data, &datalen, &q, &qlen);  // 调用_libssh2_pem_decode_integer函数
    if(ret != 0) {  // 如果返回值不为0
        ret = -1;  // 返回错误值
        goto fail;  // 跳转到失败标签
    }

    ret = _libssh2_pem_decode_integer(&data, &datalen, &g, &glen);  // 调用_libssh2_pem_decode_integer函数
    if(ret != 0) {  // 如果返回值不为0
        ret = -1;  // 返回错误值
        goto fail;  // 跳转到失败标签
    }

    ret = _libssh2_pem_decode_integer(&data, &datalen, &y, &ylen);  // 调用_libssh2_pem_decode_integer函数
    if(ret != 0) {  // 如果返回值不为0
        ret = -1;  // 返回错误值
        goto fail;  // 跳转到失败标签
    }

    ret = _libssh2_pem_decode_integer(&data, &datalen, &x, &xlen);  // 调用_libssh2_pem_decode_integer函数
    if(ret != 0) {  // 如果返回值不为0
        ret = -1;  // 返回错误值
        goto fail;  // 跳转到失败标签
    }

    if(datalen != 0) {  // 如果数据长度不为0
        ret = -1;  // 返回错误值
        goto fail;  // 跳转到失败标签
    }

    if(_libssh2_dsa_new(dsa, p, plen, q, qlen, g, glen, y, ylen, x, xlen)) {  // 调用_libssh2_dsa_new函数
        ret = -1;  // 返回错误值
        goto fail;  // 跳转到失败标签
    }

    ret = 0;  // 返回成功值

  fail:
    LIBSSH2_FREE(session, save_data);  // 释放数据内存
    # 返回变量 ret 的值
    return ret;
}

int
_libssh2_rsa_sha1_sign(LIBSSH2_SESSION * session,
                       libssh2_rsa_ctx * rsactx,
                       const unsigned char *hash,
                       size_t hash_len,
                       unsigned char **signature, size_t *signature_len)
{
    gcry_sexp_t sig_sexp;  // 定义一个用于存储签名数据的变量
    gcry_sexp_t data;  // 定义一个用于存储数据的变量
    int rc;  // 定义一个用于存储返回状态的变量
    const char *tmp;  // 定义一个临时存储字符串的变量
    size_t size;  // 定义一个用于存储大小的变量

    if(hash_len != SHA_DIGEST_LENGTH) {  // 如果哈希长度不等于 SHA_DIGEST_LENGTH
        return -1;  // 返回错误状态
    }

    if(gcry_sexp_build(&data, NULL,  // 构建数据的 S 表达式
                        "(data (flags pkcs1) (hash sha1 %b))",
                        hash_len, hash)) {
        return -1;  // 构建失败则返回错误状态
    }

    rc = gcry_pk_sign(&sig_sexp, data, rsactx);  // 使用 RSA 上下文对数据进行签名

    gcry_sexp_release(data);  // 释放数据变量

    if(rc != 0) {  // 如果签名失败
        return -1;  // 返回错误状态
    }

    data = gcry_sexp_find_token(sig_sexp, "s", 0);  // 在签名数据中查找 token
    if(!data) {  // 如果未找到
        return -1;  // 返回错误状态
    }

    tmp = gcry_sexp_nth_data(data, 1, &size);  // 获取 token 数据
    if(!tmp) {  // 如果未找到
        gcry_sexp_release(data);  // 释放数据变量
        return -1;  // 返回错误状态
    }

    if(tmp[0] == '\0') {  // 如果 token 数据的第一个字符为 '\0'
        tmp++;  // 移动指针
        size--;  // 减小大小
    }

    *signature = LIBSSH2_ALLOC(session, size);  // 分配空间给签名变量
    if(!*signature) {  // 如果分配失败
        gcry_sexp_release(data);  // 释放数据变量
        return -1;  // 返回错误状态
    }
    memcpy(*signature, tmp, size);  // 复制签名数据
    *signature_len = size;  // 设置签名长度

    gcry_sexp_release(data);  // 释放数据变量

    return rc;  // 返回状态
}

int
_libssh2_dsa_sha1_sign(libssh2_dsa_ctx * dsactx,
                       const unsigned char *hash,
                       unsigned long hash_len, unsigned char *sig)
{
    unsigned char zhash[SHA_DIGEST_LENGTH + 1];  // 定义一个用于存储哈希数据的变量
    gcry_sexp_t sig_sexp;  // 定义一个用于存储签名数据的变量
    gcry_sexp_t data;  // 定义一个用于存储数据的变量
    int ret;  // 定义一个用于存储返回状态的变量
    const char *tmp;  // 定义一个临时存储字符串的变量
    size_t size;  // 定义一个用于存储大小的变量

    if(hash_len != SHA_DIGEST_LENGTH) {  // 如果哈希长度不等于 SHA_DIGEST_LENGTH
        return -1;  // 返回错误状态
    }

    memcpy(zhash + 1, hash, hash_len);  // 复制哈希数据到 zhash 变量
    zhash[0] = 0;  // 设置 zhash 变量的第一个字符为 '\0'

    if(gcry_sexp_build(&data, NULL, "(data (value %b))",  // 构建数据的 S 表达式
                       hash_len + 1, zhash)) {
        return -1;  // 构建失败则返回错误状态
    }

    ret = gcry_pk_sign(&sig_sexp, data, dsactx);  // 使用 DSA 上下文对数据进行签名

    gcry_sexp_release(data);  // 释放数据变量

    if(ret != 0) {  // 如果签名失败
        return -1;  // 返回错误状态
    }

    memset(sig, 0, 40);  // 将签名数据初始化为 0

/* Extract R. */
    # 从 sig_sexp 中查找包含 "r" 的数据
    data = gcry_sexp_find_token(sig_sexp, "r", 0);
    # 如果找不到数据，则跳转到 err 标签
    if(!data)
        goto err;

    # 从 data 中获取第一个数据，并返回数据的大小
    tmp = gcry_sexp_nth_data(data, 1, &size);
    # 如果获取不到数据，则跳转到 err 标签
    if(!tmp)
        goto err;

    # 如果 tmp 的第一个字符是 '\0'，则移动指针和减小大小
    if(tmp[0] == '\0') {
        tmp++;
        size--;
    }

    # 如果大小小于 1 或者大于 20，则跳转到 err 标签
    if(size < 1 || size > 20)
        goto err;

    # 将 tmp 中的数据拷贝到 sig 中，从 (20 - size) 处开始拷贝，拷贝大小为 size
    memcpy(sig + (20 - size), tmp, size);

    # 释放 data 占用的内存
    gcry_sexp_release(data);
/* Extract S. */

    // 从签名数据中提取 S 值
    data = gcry_sexp_find_token(sig_sexp, "s", 0);
    // 如果提取失败，则跳转到错误处理
    if(!data)
        goto err;

    // 从 S 值中提取数据
    tmp = gcry_sexp_nth_data(data, 1, &size);
    // 如果提取失败，则跳转到错误处理
    if(!tmp)
        goto err;

    // 如果 S 值的第一个字节为 '\0'，则移动指针和减小大小
    if(tmp[0] == '\0') {
        tmp++;
        size--;
    }

    // 如果 S 值的大小不在 1 到 20 之间，则跳转到错误处理
    if(size < 1 || size > 20)
        goto err;

    // 将 S 值拷贝到签名数据中
    memcpy(sig + 20 + (20 - size), tmp, size);
    // 跳转到结束处理
    goto out;

  // 错误处理
  err:
    ret = -1;

  // 结束处理
  out:
    // 释放签名数据
    if(sig_sexp) {
        gcry_sexp_release(sig_sexp);
    }
    // 释放 S 值数据
    if(data) {
        gcry_sexp_release(data);
    }
    // 返回结果
    return ret;
}

// DSA 签名验证
int
_libssh2_dsa_sha1_verify(libssh2_dsa_ctx * dsactx,
                         const unsigned char *sig,
                         const unsigned char *m, unsigned long m_len)
{
    unsigned char hash[SHA_DIGEST_LENGTH + 1];
    gcry_sexp_t s_sig, s_hash;
    int rc = -1;

    // 计算消息的 SHA1 哈希值
    libssh2_sha1(m, m_len, hash + 1);
    hash[0] = 0;

    // 构建哈希值的 S 表达式
    if(gcry_sexp_build(&s_hash, NULL, "(data(flags raw)(value %b))",
                        SHA_DIGEST_LENGTH + 1, hash)) {
        return -1;
    }

    // 构建签名的 S 表达式
    if(gcry_sexp_build(&s_sig, NULL, "(sig-val(dsa(r %b)(s %b)))",
                        20, sig, 20, sig + 20)) {
        gcry_sexp_release(s_hash);
        return -1;
    }

    // 使用 DSA 上下文验证签名
    rc = gcry_pk_verify(s_sig, s_hash, dsactx);
    gcry_sexp_release(s_sig);
    gcry_sexp_release(s_hash);

    // 返回验证结果
    return (rc == 0) ? 0 : -1;
}

// 初始化加密算法
int
_libssh2_cipher_init(_libssh2_cipher_ctx * h,
                     _libssh2_cipher_type(algo),
                     unsigned char *iv, unsigned char *secret, int encrypt)
{
    int ret;
    int cipher = _libssh2_gcry_cipher(algo);
    int mode = _libssh2_gcry_mode(algo);
    int keylen = gcry_cipher_get_algo_keylen(cipher);

    (void) encrypt;

    // 打开加密算法上下文
    ret = gcry_cipher_open(h, cipher, mode, 0);
    // 如果打开失败，则返回错误
    if(ret) {
        return -1;
    }

    // 设置密钥
    ret = gcry_cipher_setkey(*h, secret, keylen);
    // 如果设置失败，则关闭加密算法上下文并返回错误
    if(ret) {
        gcry_cipher_close(*h);
        return -1;
    }
    # 如果加密模式不是流模式
    if(mode != GCRY_CIPHER_MODE_STREAM) {
        # 获取加密算法的块长度
        int blklen = gcry_cipher_get_algo_blklen(cipher);
        # 如果加密模式是计数器模式，设置计数器
        if(mode == GCRY_CIPHER_MODE_CTR)
            ret = gcry_cipher_setctr(*h, iv, blklen);
        # 否则，设置初始化向量
        else
            ret = gcry_cipher_setiv(*h, iv, blklen);
        # 如果设置失败，关闭加密句柄并返回错误
        if(ret) {
            gcry_cipher_close(*h);
            return -1;
        }
    }
    # 返回成功
    return 0;
}

int
_libssh2_cipher_crypt(_libssh2_cipher_ctx * ctx,
                      _libssh2_cipher_type(algo),
                      int encrypt, unsigned char *block, size_t blklen)
{
    // 获取加密算法对应的密码
    int cipher = _libssh2_gcry_cipher(algo);
    int ret;

    // 如果是加密操作
    if(encrypt) {
        // 使用密码对数据进行加密
        ret = gcry_cipher_encrypt(*ctx, block, blklen, block, blklen);
    }
    // 如果是解密操作
    else {
        // 使用密码对数据进行解密
        ret = gcry_cipher_decrypt(*ctx, block, blklen, block, blklen);
    }
    return ret;
}

int
_libssh2_pub_priv_keyfilememory(LIBSSH2_SESSION *session,
                                unsigned char **method,
                                size_t *method_len,
                                unsigned char **pubkeydata,
                                size_t *pubkeydata_len,
                                const char *privatekeydata,
                                size_t privatekeydata_len,
                                const char *passphrase)
{
    // 返回不支持从内存中提取公钥的错误信息
    return _libssh2_error(session, LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                          "Unable to extract public key from private "
                          "key in memory: "
                          "Method unimplemented in libgcrypt backend");
}

int
_libssh2_pub_priv_keyfile(LIBSSH2_SESSION *session,
                          unsigned char **method,
                          size_t *method_len,
                          unsigned char **pubkeydata,
                          size_t *pubkeydata_len,
                          const char *privatekey,
                          const char *passphrase)
{
    // 返回不支持从私钥文件中提取公钥的错误信息
    return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                         "Unable to extract public key from private key file: "
                         "Method unimplemented in libgcrypt backend");
}

void _libssh2_init_aes_ctr(void)
{
    /* no implementation */
}

void
_libssh2_dh_init(_libssh2_dh_ctx *dhctx)
{
    *dhctx = gcry_mpi_new(0);                   /* Random from client */
}

int
# 生成 Diffie-Hellman 密钥对
_libssh2_dh_key_pair(_libssh2_dh_ctx *dhctx, _libssh2_bn *public,
                     _libssh2_bn *g, _libssh2_bn *p, int group_order)
{
    # 生成 x 和 e
    gcry_mpi_randomize(*dhctx, group_order * 8 - 1, GCRY_WEAK_RANDOM);
    # 计算公钥
    gcry_mpi_powm(public, g, *dhctx, p);
    # 返回 0 表示成功
    return 0;
}

# 计算共享密钥
int
_libssh2_dh_secret(_libssh2_dh_ctx *dhctx, _libssh2_bn *secret,
                   _libssh2_bn *f, _libssh2_bn *p)
{
    # 计算共享密钥
    gcry_mpi_powm(secret, f, *dhctx, p);
    # 返回 0 表示成功
    return 0;
}

# 释放 Diffie-Hellman 上下文
void
_libssh2_dh_dtor(_libssh2_dh_ctx *dhctx)
{
    # 释放内存
    gcry_mpi_release(*dhctx);
    # 将指针置为空
    *dhctx = NULL;
}

# 结束 LIBSSH2_LIBGCRYPT 的条件编译
#endif /* LIBSSH2_LIBGCRYPT */
```