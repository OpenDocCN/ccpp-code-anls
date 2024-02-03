# `nmap\libssh2\src\pem.c`

```cpp
/* 版权声明和许可协议 */
/* 版权声明和许可协议 */

#include "libssh2_priv.h"  // 包含 libssh2_priv.h 头文件

static int  // 定义静态函数 readline，返回整型值
readline(char *line, int line_size, FILE * fp)  // 函数参数包括字符指针 line，整型 line_size，文件指针 fp
{
    size_t len;  // 声明变量 len，表示长度

    if(!line) {  // 如果 line 为空
        return -1;  // 返回 -1
    }
    if(!fgets(line, line_size, fp)) {  // 如果从文件中读取一行到 line 中失败
        return -1;  // 返回 -1
    }

    if(*line) {  // 如果 line 不为空
        len = strlen(line);  // 获取 line 的长度
        if(len > 0 && line[len - 1] == '\n') {  // 如果长度大于 0 且最后一个字符是换行符
            line[len - 1] = '\0';  // 将最后一个字符设置为字符串结束符
        }
    }
    # 如果行不为空
    if(*line) {
        # 计算行的长度
        len = strlen(line);
        # 如果长度大于0且最后一个字符是回车符
        if(len > 0 && line[len - 1] == '\r') {
            # 将回车符替换为空字符
            line[len - 1] = '\0';
        }
    }
    # 返回0
    return 0;
# 读取内存中的一行数据，将其存储到指定的缓冲区中
static int
readline_memory(char *line, size_t line_size,
                const char *filedata, size_t filedata_len,
                size_t *filedata_offset)
{
    # 定义变量 off 和 len
    size_t off, len;

    # 将文件数据偏移量赋值给 off
    off = *filedata_offset;

    # 遍历文件数据，直到遇到换行符或者达到行缓冲区的最大长度
    for(len = 0; off + len < filedata_len && len < line_size - 1; len++) {
        if(filedata[off + len] == '\n' ||
            filedata[off + len] == '\r') {
                break;
        }
    }

    # 如果读取到了数据
    if(len) {
        # 将文件数据中的内容复制到行缓冲区中
        memcpy(line, filedata + off, len);
        # 更新文件数据偏移量
        *filedata_offset += len;
    }

    # 在行缓冲区的末尾添加字符串结束符
    line[len] = '\0';
    # 更新文件数据偏移量
    *filedata_offset += 1;

    # 返回 0 表示成功
    return 0;
}

# 定义行缓冲区的大小
#define LINE_SIZE 128

# 定义加密注释的字符串
static const char *crypt_annotation = "Proc-Type: 4,ENCRYPTED";

# 定义十六进制解码函数
static unsigned char hex_decode(char digit)
{
    # 如果数字大于等于 'A'，则返回对应的十六进制值
    return (digit >= 'A') ? 0xA + (digit - 'A') : (digit - '0');
}

# 解析 PEM 格式的数据
int
_libssh2_pem_parse(LIBSSH2_SESSION * session,
                   const char *headerbegin,
                   const char *headerend,
                   const unsigned char *passphrase,
                   FILE * fp, unsigned char **data, unsigned int *datalen)
{
    # 定义行缓冲区和初始化向量
    char line[LINE_SIZE];
    unsigned char iv[LINE_SIZE];
    # 定义 base64 编码的数据和长度
    char *b64data = NULL;
    unsigned int b64datalen = 0;
    # 定义返回值和加密方法
    int ret;
    const LIBSSH2_CRYPT_METHOD *method = NULL;

    # 循环读取文件中的每一行数据
    do {
        *line = '\0';

        # 调用 readline 函数读取一行数据
        if(readline(line, LINE_SIZE, fp)) {
            return -1;
        }
    }
    # 直到读取到 headerbegin 所指定的字符串
    while(strcmp(line, headerbegin) != 0);

    # 读取下一行数据
    if(readline(line, LINE_SIZE, fp)) {
        return -1;
    }
}
    # 如果存在密码并且当前行与加密注释相匹配
    if(passphrase &&
            memcmp(line, crypt_annotation, strlen(crypt_annotation)) == 0) {
        # 定义变量
        const LIBSSH2_CRYPT_METHOD **all_methods, *cur_method;
        int i;

        # 读取下一行
        if(readline(line, LINE_SIZE, fp)) {
            ret = -1;
            goto out;
        }

        # 获取所有加密方法
        all_methods = libssh2_crypt_methods();
        # 遍历所有加密方法
        while((cur_method = *all_methods++)) {
            # 如果当前方法的 PEM 注释存在并且与当前行匹配
            if(*cur_method->pem_annotation &&
                    memcmp(line, cur_method->pem_annotation,
                           strlen(cur_method->pem_annotation)) == 0) {
                # 设置当前方法
                method = cur_method;
                # 从行中复制 IV 数据
                memcpy(iv, line + strlen(method->pem_annotation) + 1,
                       2*method->iv_len);
            }
        }

        # 如果没有可用的加密方法能够解密密钥
        if(method == NULL)
            return -1;

        # 从十六进制解码 IV
        for(i = 0; i < method->iv_len; ++i) {
            iv[i]  = hex_decode(iv[2*i]) << 4;
            iv[i] |= hex_decode(iv[2*i + 1]);
        }

        # 跳过下一行
        if(readline(line, LINE_SIZE, fp)) {
            ret = -1;
            goto out;
        }
    }

    # 循环直到遇到结束标记
    do {
        # 如果当前行不为空
        if(*line) {
            # 临时变量和行长度
            char *tmp;
            size_t linelen;

            linelen = strlen(line);
            # 重新分配内存并拷贝数据
            tmp = LIBSSH2_REALLOC(session, b64data, b64datalen + linelen);
            if(!tmp) {
                _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                               "Unable to allocate memory for PEM parsing");
                ret = -1;
                goto out;
            }
            memcpy(tmp + b64datalen, line, linelen);
            b64data = tmp;
            b64datalen += linelen;
        }

        # 将当前行置为空
        *line = '\0';

        # 读取下一行
        if(readline(line, LINE_SIZE, fp)) {
            ret = -1;
            goto out;
        }
    } while(strcmp(line, headerend) != 0);

    # 如果没有 b64 数据，则返回错误
    if(!b64data) {
        return -1;
    }
    # 使用libssh2_base64_decode函数对base64编码的数据进行解码，并将解码后的数据和长度存储在data和datalen中
    if(libssh2_base64_decode(session, (char **) data, datalen,
                              b64data, b64datalen)) {
        # 如果解码失败，则将ret设为-1，并跳转到out标签处
        ret = -1;
        goto out;
    }

    }

    # 如果解码成功，则将ret设为0
    ret = 0;
  out:
    # 如果b64data不为空，则调用_libssh2_explicit_zero函数将其清零，并释放内存
    if(b64data) {
        _libssh2_explicit_zero(b64data, b64datalen);
        LIBSSH2_FREE(session, b64data);
    }
    # 返回ret的值
    return ret;
}
// 结束函数定义

int
_libssh2_pem_parse_memory(LIBSSH2_SESSION * session,
                          const char *headerbegin,
                          const char *headerend,
                          const char *filedata, size_t filedata_len,
                          unsigned char **data, unsigned int *datalen)
{
    // 定义变量
    char line[LINE_SIZE];
    char *b64data = NULL;
    unsigned int b64datalen = 0;
    size_t off = 0;
    int ret;

    // 循环读取文件数据，直到找到 headerbegin 为止
    do {
        *line = '\0';

        // 从内存中读取一行数据
        if(readline_memory(line, LINE_SIZE, filedata, filedata_len, &off)) {
            return -1;
        }
    }
    while(strcmp(line, headerbegin) != 0);

    *line = '\0';

    // 循环读取文件数据，直到找到 headerend 为止
    do {
        if(*line) {
            char *tmp;
            size_t linelen;

            linelen = strlen(line);
            tmp = LIBSSH2_REALLOC(session, b64data, b64datalen + linelen);
            if(!tmp) {
                _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                               "Unable to allocate memory for PEM parsing");
                ret = -1;
                goto out;
            }
            memcpy(tmp + b64datalen, line, linelen);
            b64data = tmp;
            b64datalen += linelen;
        }

        *line = '\0';

        // 从内存中读取一行数据
        if(readline_memory(line, LINE_SIZE, filedata, filedata_len, &off)) {
            ret = -1;
            goto out;
        }
    } while(strcmp(line, headerend) != 0);

    // 如果 b64data 为空，则返回错误
    if(!b64data) {
        return -1;
    }

    // 解码 base64 数据
    if(libssh2_base64_decode(session, (char **) data, datalen,
                              b64data, b64datalen)) {
        ret = -1;
        goto out;
    }

    ret = 0;
  out:
    // 释放内存并返回结果
    if(b64data) {
        _libssh2_explicit_zero(b64data, b64datalen);
        LIBSSH2_FREE(session, b64data);
    }
    return ret;
}

/* OpenSSH formatted keys */
// 定义常量
#define AUTH_MAGIC "openssh-key-v1"
#define OPENSSH_HEADER_BEGIN "-----BEGIN OPENSSH PRIVATE KEY-----"
#define OPENSSH_HEADER_END "-----END OPENSSH PRIVATE KEY-----"

static int
# 解析 PEM 格式的数据并进行解密
_libssh2_openssh_pem_parse_data(LIBSSH2_SESSION * session,
                                const unsigned char *passphrase,
                                const char *b64data, size_t b64datalen,
                                struct string_buf **decrypted_buf)
{
    const LIBSSH2_CRYPT_METHOD *method = NULL;  # 定义加密方法
    struct string_buf decoded, decrypted, kdf_buf;  # 定义字符串缓冲区
    unsigned char *ciphername = NULL;  # 定义加密算法名称
    unsigned char *kdfname = NULL;  # 定义密钥派生函数名称
    unsigned char *kdf = NULL;  # 定义密钥派生函数
    unsigned char *buf = NULL;  # 定义缓冲区
    unsigned char *salt = NULL;  # 定义盐值
    uint32_t nkeys, check1, check2;  # 定义整型变量
    uint32_t rounds = 0;  # 定义轮数
    unsigned char *key = NULL;  # 定义密钥
    unsigned char *key_part = NULL;  # 定义密钥部分
    unsigned char *iv_part = NULL;  # 定义初始化向量部分
    unsigned char *f = NULL;  # 定义临时变量
    unsigned int f_len = 0;  # 定义临时变量的长度
    int ret = 0, keylen = 0, ivlen = 0, total_len = 0;  # 定义整型变量
    size_t kdf_len = 0, tmp_len = 0, salt_len = 0;  # 定义大小变量

    if(decrypted_buf)
        *decrypted_buf = NULL;  # 如果解密缓冲区存在，则置空

    /* decode file */
    if(libssh2_base64_decode(session, (char **)&f, &f_len,
                             b64data, b64datalen)) {  # 解码文件
       ret = -1;  # 设置返回值为-1
       goto out;  # 跳转到标签 out
    }

    /* Parse the file */
    decoded.data = (unsigned char *)f;  # 设置解码后的数据
    decoded.dataptr = (unsigned char *)f;  # 设置解码后的数据指针
    decoded.len = f_len;  # 设置解码后的数据长度

    if(decoded.len < strlen(AUTH_MAGIC)) {  # 如果解码后的数据长度小于认证魔术值的长度
        ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO, "key too short");  # 设置错误码和错误信息
        goto out;  # 跳转到标签 out
    }

    if(strncmp((char *) decoded.dataptr, AUTH_MAGIC,
               strlen(AUTH_MAGIC)) != 0) {  # 如果解码后的数据与认证魔术值不匹配
        ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                             "key auth magic mismatch");  # 设置错误码和错误信息
        goto out;  # 跳转到标签 out
    }

    decoded.dataptr += strlen(AUTH_MAGIC) + 1;  # 移动解码后的数据指针

    if(_libssh2_get_string(&decoded, &ciphername, &tmp_len) ||  # 获取加密算法名称
       tmp_len == 0) {  # 如果加密算法名称长度为0
        ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                             "ciphername is missing");  # 设置错误码和错误信息
        goto out;  # 跳转到标签 out
    }
    # 如果解析 kdfname 失败或者长度为0，则返回错误信息并跳转到 out 标签
    if(_libssh2_get_string(&decoded, &kdfname, &tmp_len) ||
       tmp_len == 0) {
        ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "kdfname is missing");
        goto out;
    }

    # 如果解析 kdf 失败，则返回错误信息并跳转到 out 标签；否则设置 kdf_buf 的数据和长度
    if(_libssh2_get_string(&decoded, &kdf, &kdf_len)) {
        ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                             "kdf is missing");
        goto out;
    }
    else {
        kdf_buf.data = kdf;
        kdf_buf.dataptr = kdf;
        kdf_buf.len = kdf_len;
    }

    # 如果 passphrase 为空或者长度为0，并且 ciphername 不为 "none"，则返回密钥文件认证失败的错误信息并跳转到 out 标签
    if((passphrase == NULL || strlen((const char *)passphrase) == 0) &&
        strcmp((const char *)ciphername, "none") != 0) {
        /* passphrase required */
        ret = LIBSSH2_ERROR_KEYFILE_AUTH_FAILED;
        goto out;
    }

    # 如果 kdfname 不为 "none" 且不为 "bcrypt"，则返回未知密码错误信息并跳转到 out 标签
    if(strcmp((const char *)kdfname, "none") != 0 &&
       strcmp((const char *)kdfname, "bcrypt") != 0) {
        ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                             "unknown cipher");
        goto out;
    }

    # 如果 kdfname 为 "none" 且 ciphername 不为 "none"，则返回无效格式错误信息并跳转到 out 标签
    if(!strcmp((const char *)kdfname, "none") &&
       strcmp((const char *)ciphername, "none") != 0) {
        ret =_libssh2_error(session, LIBSSH2_ERROR_PROTO,
                            "invalid format");
        goto out;
    }

    # 如果解析 nkeys 失败或者 nkeys 不为1，则返回不支持多个密钥的错误信息并跳转到 out 标签
    if(_libssh2_get_u32(&decoded, &nkeys) != 0 || nkeys != 1) {
        ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                             "Multiple keys are unsupported");
        goto out;
    }

    # 如果解析 buf 失败或者长度为0，则返回无效私钥错误信息并跳转到 out 标签
    if(_libssh2_get_string(&decoded, &buf, &tmp_len) || tmp_len == 0) {
        ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                             "Invalid private key; "
                             "expect embedded public key");
        goto out;
    }

    # 如果解析 buf 失败或者长度为0，则返回私钥数据未找到的错误信息并跳转到 out 标签
    if(_libssh2_get_string(&decoded, &buf, &tmp_len) || tmp_len == 0) {
        ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "Private key data not found");
        goto out;
    }

    # 解码加密的私钥
    # 设置解密后的数据指针和数据缓冲区
    decrypted.data = decrypted.dataptr = buf;
    # 设置解密后的数据长度
    decrypted.len = tmp_len;

    # 如果加密算法名称存在且不是"none"
    if(ciphername && strcmp((const char *)ciphername, "none") != 0) {
        # 获取所有可用的加密方法
        const LIBSSH2_CRYPT_METHOD **all_methods, *cur_method;
        all_methods = libssh2_crypt_methods();
        # 遍历所有加密方法
        while((cur_method = *all_methods++)) {
            # 如果当前方法的名称不为空且与给定的加密算法名称匹配
            if(*cur_method->name &&
                memcmp(ciphername, cur_method->name,
                       strlen(cur_method->name)) == 0) {
                    # 设置当前加密方法
                    method = cur_method;
                }
        }

        # 如果没有找到匹配的加密方法
        if(method == NULL) {
            # 返回错误信息
            ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                "No supported cipher found");
            # 跳转到结束位置
            goto out;
        }
    }

    # 检查随机字节是否匹配
    if(_libssh2_get_u32(&decrypted, &check1) != 0 ||
       _libssh2_get_u32(&decrypted, &check2) != 0 ||
       check1 != check2) {
       # 返回私钥解包失败的错误信息
       _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                      "Private key unpack failed (correct password?)");
       # 设置返回值为私钥文件认证失败
       ret = LIBSSH2_ERROR_KEYFILE_AUTH_FAILED;
       # 跳转到结束位置
       goto out;
    }
    # 如果解密后的缓冲区不为空
    if(decrypted_buf != NULL) {
        # 创建一个新的字符串缓冲区用于输出
        struct string_buf *out_buf = _libssh2_string_buf_new(session);
        # 如果无法分配内存给新的缓冲区
        if(!out_buf) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                 "Unable to allocate memory for "
                                 "decrypted struct");
            # 转到标签out
            goto out;
        }

        # 为新缓冲区的数据分配内存
        out_buf->data = LIBSSH2_CALLOC(session, decrypted.len);
        # 如果无法分配内存给新缓冲区的数据
        if(out_buf->data == NULL) {
            ret = _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                 "Unable to allocate memory for "
                                 "decrypted struct");
            # 释放新缓冲区的内存
            _libssh2_string_buf_free(session, out_buf);
            # 转到标签out
            goto out;
        }
        # 将解密后的数据复制到新缓冲区
        memcpy(out_buf->data, decrypted.data, decrypted.len);
        # 设置新缓冲区的数据指针
        out_buf->dataptr = out_buf->data +
            (decrypted.dataptr - decrypted.data);
        # 设置新缓冲区的长度
        out_buf->len = decrypted.len;

        # 将新缓冲区赋值给解密后的缓冲区
        *decrypted_buf = out_buf;
    }
    /* 清理 */
    if(key) {
        _libssh2_explicit_zero(key, total_len);
        LIBSSH2_FREE(session, key);
    }
    if(key_part) {
        _libssh2_explicit_zero(key_part, keylen);
        LIBSSH2_FREE(session, key_part);
    }
    if(iv_part) {
        _libssh2_explicit_zero(iv_part, ivlen);
        LIBSSH2_FREE(session, iv_part);
    }
    if(f) {
        _libssh2_explicit_zero(f, f_len);
        LIBSSH2_FREE(session, f);
    }

    返回结果
    return ret;
}

int
_libssh2_openssh_pem_parse(LIBSSH2_SESSION * session,
                           const unsigned char *passphrase,
                           FILE * fp, struct string_buf **decrypted_buf)
{
    char line[LINE_SIZE];
    char *b64data = NULL;
    unsigned int b64datalen = 0;
    int ret = 0;

    /* 读取文件 */

    do {
        *line = '\0';

        如果读取行失败，则返回-1
        if(readline(line, LINE_SIZE, fp)) {
            return -1;
        }
    }
    当读取到的行不是 OPENSSH_HEADER_BEGIN 时，继续循环
    while(strcmp(line, OPENSSH_HEADER_BEGIN) != 0);

    如果读取行失败，则返回-1
    if(readline(line, LINE_SIZE, fp)) {
        return -1;
    }

    do {
        如果行不为空
        if(*line) {
            char *tmp;
            size_t linelen;

            计算行的长度
            linelen = strlen(line);
            重新分配内存以存储 base64 数据
            tmp = LIBSSH2_REALLOC(session, b64data, b64datalen + linelen);
            如果分配内存失败
            if(!tmp) {
                报错并返回-1
                _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                               "Unable to allocate memory for PEM parsing");
                ret = -1;
                跳转到清理部分
                goto out;
            }
            将行数据拷贝到 b64data 中
            memcpy(tmp + b64datalen, line, linelen);
            更新 b64data 和 b64datalen
            b64data = tmp;
            b64datalen += linelen;
        }

        *line = '\0';

        如果读取行失败，则返回-1
        if(readline(line, LINE_SIZE, fp)) {
            ret = -1;
            跳转到清理部分
            goto out;
        }
    当读取到的行不是 OPENSSH_HEADER_END 时，继续循环
    } while(strcmp(line, OPENSSH_HEADER_END) != 0);

    如果 b64data 为空，则返回-1
    if(!b64data) {
        return -1;
    }
    # 调用_libssh2_openssh_pem_parse_data函数，解析数据并返回结果
    ret = _libssh2_openssh_pem_parse_data(session,
                                          passphrase,
                                          (const char *)b64data,
                                          (size_t)b64datalen,
                                          decrypted_buf);
    # 如果b64data存在
    if(b64data) {
        # 清空b64data中的数据
        _libssh2_explicit_zero(b64data, b64datalen);
        # 释放b64data所占用的内存
        LIBSSH2_FREE(session, b64data);
    }
    return ret;
}

int
_libssh2_openssh_pem_parse_memory(LIBSSH2_SESSION * session,
                                  const unsigned char *passphrase,
                                  const char *filedata, size_t filedata_len,
                                  struct string_buf **decrypted_buf)
{
    char line[LINE_SIZE]; // 定义一个字符数组用于存储每行数据
    char *b64data = NULL; // 定义一个指针用于存储base64编码的数据
    unsigned int b64datalen = 0; // 存储base64编码数据的长度
    size_t off = 0; // 偏移量
    int ret; // 存储返回值

    if(filedata == NULL || filedata_len <= 0) // 如果文件数据为空或长度小于等于0
        return _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                              "Error parsing PEM: filedata missing"); // 返回解析PEM错误

    do {

        *line = '\0'; // 将line数组的第一个元素置为空字符

        if(off >= filedata_len) // 如果偏移量大于等于文件数据长度
            return _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                  "Error parsing PEM: offset out of bounds"); // 返回解析PEM错误

        if(readline_memory(line, LINE_SIZE, filedata, filedata_len, &off)) { // 调用readline_memory函数读取一行数据
            return -1; // 如果读取失败，返回-1
        }
    }
    while(strcmp(line, OPENSSH_HEADER_BEGIN) != 0); // 当读取的行不是"-----BEGIN OPENSSH PRIVATE KEY-----"时循环

    *line = '\0'; // 将line数组的第一个元素置为空字符

    do {
        if (*line) { // 如果line不为空
            char *tmp; // 定义一个临时指针
            size_t linelen; // 存储行的长度

            linelen = strlen(line); // 获取行的长度
            tmp = LIBSSH2_REALLOC(session, b64data, b64datalen + linelen); // 重新分配内存用于存储base64编码数据
            if(!tmp) { // 如果分配内存失败
                ret = _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                     "Unable to allocate memory for "
                                     "PEM parsing"); // 返回分配内存错误
                goto out; // 跳转到out标签
            }
            memcpy(tmp + b64datalen, line, linelen); // 将行数据拷贝到新分配的内存中
            b64data = tmp; // 更新base64编码数据的指针
            b64datalen += linelen; // 更新base64编码数据的长度
        }

        *line = '\0'; // 将line数组的第一个元素置为空字符

        if(off >= filedata_len) { // 如果偏移量大于等于文件数据长度
            ret = _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                 "Error parsing PEM: offset out of bounds"); // 返回解析PEM错误
            goto out; // 跳转到out标签
        }

        if(readline_memory(line, LINE_SIZE, filedata, filedata_len, &off)) { // 调用readline_memory函数读取一行数据
            ret = -1; // 如果读取失败，返回-1
            goto out; // 跳转到out标签
        }
    } while(strcmp(line, OPENSSH_HEADER_END) != 0); // 当读取的行不是"-----END OPENSSH PRIVATE KEY-----"时循环
    # 如果b64data为空，则返回解析PEM错误
    if(!b64data)
        return _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                              "Error parsing PEM: base 64 data missing");
    # 调用_libssh2_openssh_pem_parse_data函数，解析PEM数据并存储到decrypted_buf中
    ret = _libssh2_openssh_pem_parse_data(session, passphrase, b64data,
                                          b64datalen, decrypted_buf);
# 如果 b64data 存在
if(b64data) {
    # 明确将 b64data 的内容清零
    _libssh2_explicit_zero(b64data, b64datalen);
    # 释放 b64data 的内存
    LIBSSH2_FREE(session, b64data);
}
# 返回结果
return ret;
}

# 读取 ASN.1 长度
static int
read_asn1_length(const unsigned char *data,
                 unsigned int datalen, unsigned int *len)
{
    unsigned int lenlen;
    int nextpos;

    # 如果数据长度小于 1，返回错误
    if(datalen < 1) {
        return -1;
    }
    # 读取长度信息
    *len = data[0];

    # 如果长度大于等于 0x80
    if(*len >= 0x80) {
        # 读取长度的长度
        lenlen = *len & 0x7F;
        *len = data[1];
        # 如果长度信息的长度超过了数据长度，返回错误
        if(1 + lenlen > datalen) {
            return -1;
        }
        # 如果长度信息的长度大于 1
        if(lenlen > 1) {
            # 读取更多的长度信息
            *len <<= 8;
            *len |= data[2];
        }
    }
    else {
        lenlen = 0;
    }

    # 计算下一个位置
    nextpos = 1 + lenlen;
    # 如果长度信息的长度大于 2，或者 1 + 长度信息的长度 + 长度值 大于数据长度，返回错误
    if(lenlen > 2 || 1 + lenlen + *len > datalen) {
        return -1;
    }

    return nextpos;
}

# 解码 PEM 序列
int
_libssh2_pem_decode_sequence(unsigned char **data, unsigned int *datalen)
{
    unsigned int len;
    int lenlen;

    # 如果数据长度小于 1，返回错误
    if(*datalen < 1) {
        return -1;
    }

    # 如果数据的第一个字节不是 '\x30'，返回错误
    if((*data)[0] != '\x30') {
        return -1;
    }

    # 移动数据指针和减少数据长度
    (*data)++;
    (*datalen)--;

    # 读取 ASN.1 长度
    lenlen = read_asn1_length(*data, *datalen, &len);
    # 如果读取长度失败，或者长度信息的长度加上长度值不等于数据长度，返回错误
    if(lenlen < 0 || lenlen + len != *datalen) {
        return -1;
    }

    # 移动数据指针和减少数据长度
    *data += lenlen;
    *datalen -= lenlen;

    return 0;
}

# 解码 PEM 整数
int
_libssh2_pem_decode_integer(unsigned char **data, unsigned int *datalen,
                            unsigned char **i, unsigned int *ilen)
{
    unsigned int len;
    int lenlen;

    # 如果数据长度小于 1，返回错误
    if(*datalen < 1) {
        return -1;
    }

    # 如果数据的第一个字节不是 '\x02'，返回错误
    if((*data)[0] != '\x02') {
        return -1;
    }

    # 移动数据指针和减少数据长度
    (*data)++;
    (*datalen)--;

    # 读取 ASN.1 长度
    lenlen = read_asn1_length(*data, *datalen, &len);
    # 如果读取长度失败，或者长度信息的长度加上长度值大于数据长度，返回错误
    if(lenlen < 0 || lenlen + len > *datalen) {
        return -1;
    }

    # 移动数据指针和减少数据长度
    *data += lenlen;
    *datalen -= lenlen;

    # 设置整数指针和长度
    *i = *data;
    *ilen = len;

    # 移动数据指针和减少数据长度
    *data += len;
    *datalen -= len;

    return 0;
}
```