# `nmap\libssh2\src\userauth.c`

```
/*
 * 版权声明，列出了软件的版权归属和使用条件
 * 保留原始代码的版权声明、条件列表和免责声明
 * 在二进制形式中，需要在文档或其他提供的材料中重现版权声明、条件列表和免责声明
 * 未经特定书面许可，不得使用版权所有者或其他贡献者的名称来认可或推广基于此软件的产品
 * 免责声明，声明软件按原样提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的适用性
 * 无论在何种情况下，版权所有者或贡献者均不对任何直接、间接、偶然、特殊、惩罚性或后果性损害承担责任
 * 包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断，无论是合同、严格责任或侵权行为，即使已被告知可能性
 */

#include "libssh2_priv.h"  // 引入 libssh2_priv.h 头文件

#include <ctype.h>  // 引入 ctype.h 头文件
#include <stdio.h>  // 引入 stdio.h 头文件

#include <assert.h>  // 引入 assert.h 头文件

#ifdef HAVE_SYS_UIO_H  // 如果定义了 HAVE_SYS_UIO_H
#include <sys/uio.h>  // 则引入 sys/uio.h 头文件
#endif

#include "transport.h"  // 引入 transport.h 头文件
#include "session.h"  // 引入 session.h 头文件
#include "userauth.h"  // 引入 userauth.h 头文件
# 列出认证方法
# 如果“none”恰好允许该用户，则将产生成功的登录
# 这对于任何 SSH 服务器来说都不是常见的配置
# 用户名应为 NULL，或者为以空字符结尾的字符串
static char *userauth_list(LIBSSH2_SESSION *session, const char *username,
                           unsigned int username_len)
{
    static const unsigned char reply_codes[3] =
        { SSH_MSG_USERAUTH_SUCCESS, SSH_MSG_USERAUTH_FAILURE, 0 };
    # packet_type(1) + username_len(4) + service_len(4) +
    # service(14)"ssh-connection" + method_len(4) = 27
    unsigned long methods_len;
    unsigned char *s;
    int rc;

    if(session->userauth_list_state == libssh2_NB_state_idle) {
        # 将整个结构清零
        memset(&session->userauth_list_packet_requirev_state, 0,
               sizeof(session->userauth_list_packet_requirev_state));

        session->userauth_list_data_len = username_len + 27;

        s = session->userauth_list_data =
            LIBSSH2_ALLOC(session, session->userauth_list_data_len);
        if(!session->userauth_list_data) {
            _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                           "Unable to allocate memory for userauth_list");
            return NULL;
        }

        *(s++) = SSH_MSG_USERAUTH_REQUEST;
        _libssh2_store_str(&s, username, username_len);
        _libssh2_store_str(&s, "ssh-connection", 14);
        _libssh2_store_u32(&s, 4); # 分别发送“none”

        session->userauth_list_state = libssh2_NB_state_created;
    }
}
    // 如果用户认证列表状态为已创建
    if(session->userauth_list_state == libssh2_NB_state_created) {
        // 发送用户认证列表数据
        rc = _libssh2_transport_send(session, session->userauth_list_data,
                                     session->userauth_list_data_len,
                                     (unsigned char *)"none", 4);
        // 如果返回值为 LIBSSH2_ERROR_EAGAIN，表示需要再次发送
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            // 设置错误信息并返回空值
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block requesting userauth list");
            return NULL;
        }
        // 释放已发送的数据包
        LIBSSH2_FREE(session, session->userauth_list_data);
        session->userauth_list_data = NULL;

        // 如果返回值不为 0，表示发送失败
        if(rc) {
            // 设置错误信息并返回空值
            _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                           "Unable to send userauth-none request");
            session->userauth_list_state = libssh2_NB_state_idle;
            return NULL;
        }

        // 设置用户认证列表状态为已发送
        session->userauth_list_state = libssh2_NB_state_sent;
    }

    // 设置用户认证列表状态为空闲
    session->userauth_list_state = libssh2_NB_state_idle;
    // 返回用户认证列表数据
    return (char *) session->userauth_list_data;
}

/* libssh2_userauth_list
 *
 * 列出认证方法
 * 如果“none”恰好适用于此用户，则将产生成功的登录
 * 不过这对于任何 SSH 服务器来说都不是常见的配置
 * 用户名应为 NULL，或者是以空字符结尾的字符串
 */
LIBSSH2_API char *
libssh2_userauth_list(LIBSSH2_SESSION * session, const char *user,
                      unsigned int user_len)
{
    char *ptr;
    BLOCK_ADJUST_ERRNO(ptr, session,
                       userauth_list(session, user, user_len));
    return ptr;
}

/*
 * libssh2_userauth_authenticated
 *
 * 返回：如果尚未经过身份验证，则返回 0
 *       如果已经经过身份验证，则返回 1
 */
LIBSSH2_API int
libssh2_userauth_authenticated(LIBSSH2_SESSION * session)
{
    return (session->state & LIBSSH2_STATE_AUTHENTICATED)?1:0;
}



/* userauth_password
 * 普通的登录
 */
static int
userauth_password(LIBSSH2_SESSION *session,
                  const char *username, unsigned int username_len,
                  const unsigned char *password, unsigned int password_len,
                  LIBSSH2_PASSWD_CHANGEREQ_FUNC((*passwd_change_cb)))
{
    unsigned char *s;
    static const unsigned char reply_codes[4] =
        { SSH_MSG_USERAUTH_SUCCESS, SSH_MSG_USERAUTH_FAILURE,
          SSH_MSG_USERAUTH_PASSWD_CHANGEREQ, 0
        };
    int rc;
    # 如果用户认证密码状态为闲置
    if(session->userauth_pswd_state == libssh2_NB_state_idle) {
        # 将整个结构清零
        memset(&session->userauth_pswd_packet_requirev_state, 0,
               sizeof(session->userauth_pswd_packet_requirev_state));

        # 计算数据长度
        session->userauth_pswd_data_len = username_len + 40;

        # 初始化数据0
        session->userauth_pswd_data0 =
            (unsigned char) ~SSH_MSG_USERAUTH_PASSWD_CHANGEREQ;

        # 分配内存
        s = session->userauth_pswd_data =
            LIBSSH2_ALLOC(session, session->userauth_pswd_data_len);
        # 如果分配内存失败，则返回错误
        if(!session->userauth_pswd_data) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for "
                                  "userauth-password request");
        }

        # 填充数据
        *(s++) = SSH_MSG_USERAUTH_REQUEST;
        _libssh2_store_str(&s, username, username_len);
        _libssh2_store_str(&s, "ssh-connection", sizeof("ssh-connection") - 1);
        _libssh2_store_str(&s, "password", sizeof("password") - 1);
        *s++ = '\0';
        _libssh2_store_u32(&s, password_len);
        # 'password' 被单独发送

        # 调试信息
        _libssh2_debug(session, LIBSSH2_TRACE_AUTH,
                       "Attempting to login using password authentication");

        # 设置用户认证密码状态为已创建
        session->userauth_pswd_state = libssh2_NB_state_created;
    }
    // 如果用户认证密码状态为已创建
    if(session->userauth_pswd_state == libssh2_NB_state_created) {
        // 发送用户认证密码数据
        rc = _libssh2_transport_send(session, session->userauth_pswd_data,
                                     session->userauth_pswd_data_len,
                                     password, password_len);
        // 如果返回值为阻塞状态
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            // 返回阻塞写入密码请求的错误
            return _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                                  "Would block writing password request");
        }

        // 释放已发送的数据包
        LIBSSH2_FREE(session, session->userauth_pswd_data);
        session->userauth_pswd_data = NULL;

        // 如果返回值不为0
        if(rc) {
            // 设置用户认证密码状态为空闲
            session->userauth_pswd_state = libssh2_NB_state_idle;
            // 返回无法发送用户认证密码请求的错误
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send userauth-password request");
        }

        // 设置用户认证密码状态为已发送
        session->userauth_pswd_state = libssh2_NB_state_sent;
    }

  password_response:

    }

    // 释放用户认证密码数据
    LIBSSH2_FREE(session, session->userauth_pswd_data);
    session->userauth_pswd_data = NULL;
    // 设置用户认证密码状态为空闲
    session->userauth_pswd_state = libssh2_NB_state_idle;

    // 返回认证失败的错误
    return _libssh2_error(session, LIBSSH2_ERROR_AUTHENTICATION_FAILED,
                          "Authentication failed");
# 结束 libssh2_userauth_password_ex 函数定义
}

# 定义 libssh2_userauth_password_ex 函数
# 用于通过用户名和密码进行身份验证
LIBSSH2_API int
libssh2_userauth_password_ex(LIBSSH2_SESSION *session, const char *username,
                             unsigned int username_len, const char *password,
                             unsigned int password_len,
                             LIBSSH2_PASSWD_CHANGEREQ_FUNC
                             ((*passwd_change_cb)))
{
    int rc;
    # 调用 userauth_password 函数进行密码验证
    BLOCK_ADJUST(rc, session,
                 userauth_password(session, username, username_len,
                                   (unsigned char *)password, password_len,
                                   passwd_change_cb));
    return rc;
}

# 定义 memory_read_publickey 函数
# 用于读取公钥文件的内容
static int
memory_read_publickey(LIBSSH2_SESSION * session, unsigned char **method,
                      size_t *method_len,
                      unsigned char **pubkeydata,
                      size_t *pubkeydata_len,
                      const char *pubkeyfiledata,
                      size_t pubkeyfiledata_len)
{
    unsigned char *pubkey = NULL, *sp1, *sp2, *tmp;
    size_t pubkey_len = pubkeyfiledata_len;
    unsigned int tmp_len;

    # 如果公钥文件数据长度小于等于1，则返回错误
    if(pubkeyfiledata_len <= 1) {
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                              "Invalid data in public key file");
    }

    # 分配内存用于存储公钥数据
    pubkey = LIBSSH2_ALLOC(session, pubkeyfiledata_len);
    if(!pubkey) {
        return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                              "Unable to allocate memory for public key data");
    }

    # 将公钥文件数据复制到分配的内存中
    memcpy(pubkey, pubkeyfiledata, pubkeyfiledata_len);

    # 移除末尾的空白字符
    while(pubkey_len && isspace(pubkey[pubkey_len - 1]))
        pubkey_len--;

    # 如果公钥数据长度为0，则返回错误
    if(!pubkey_len) {
        LIBSSH2_FREE(session, pubkey);
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                              "Missing public key data");
    }

    # 在公钥数据中查找空格字符
    sp1 = memchr(pubkey, ' ', pubkey_len);

... (以下省略)
    # 如果公钥数据为空，则释放内存并返回错误信息
    if(sp1 == NULL) {
        LIBSSH2_FREE(session, pubkey);
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                              "Invalid public key data");
    }

    # 移动指针到下一个位置
    sp1++;

    # 在公钥数据中查找空格，找到则将 sp2 指向该位置，否则假设 id 字符串缺失
    sp2 = memchr(sp1, ' ', pubkey_len - (sp1 - pubkey));
    if(sp2 == NULL) {
        /* Assume that the id string is missing, but that it's okay */
        sp2 = pubkey + pubkey_len;
    }

    # 使用 base64 解码公钥数据，将结果存储在 tmp 中
    if(libssh2_base64_decode(session, (char **) &tmp, &tmp_len,
                              (char *) sp1, sp2 - sp1)) {
        LIBSSH2_FREE(session, pubkey);
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                                  "Invalid key data, not base64 encoded");
    }

    # 将公钥数据指针和长度存储在 method 和 method_len 中
    *method = pubkey;
    *method_len = sp1 - pubkey - 1;

    # 将解码后的公钥数据指针和长度存储在 pubkeydata 和 pubkeydata_len 中
    *pubkeydata = tmp;
    *pubkeydata_len = tmp_len;

    # 返回成功
    return 0;
# 从一个 id_???.pub 样式的文件中读取公钥
# 在成功时，返回一个包含解码后的密钥的分配字符串 *pubkeydata
# 在成功时，返回一个包含密钥方法（例如 "ssh-dss"）的分配字符串在 method 中
static int
file_read_publickey(LIBSSH2_SESSION * session, unsigned char **method,
                    size_t *method_len,
                    unsigned char **pubkeydata,
                    size_t *pubkeydata_len,
                    const char *pubkeyfile)
{
    FILE *fd;
    char c;
    unsigned char *pubkey = NULL, *sp1, *sp2, *tmp;
    size_t pubkey_len = 0, sp_len;
    unsigned int tmp_len;

    _libssh2_debug(session, LIBSSH2_TRACE_AUTH, "Loading public key file: %s",
                   pubkeyfile);
    # 读取公钥
    fd = fopen(pubkeyfile, FOPEN_READTEXT);
    if(!fd) {
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                              "Unable to open public key file");
    }
    # 计算公钥长度
    while(!feof(fd) && 1 == fread(&c, 1, 1, fd) && c != '\r' && c != '\n') {
        pubkey_len++;
    }
    rewind(fd);

    if(pubkey_len <= 1) {
        fclose(fd);
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                              "Invalid data in public key file");
    }
    # 分配内存并读取公钥数据
    pubkey = LIBSSH2_ALLOC(session, pubkey_len);
    if(!pubkey) {
        fclose(fd);
        return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                              "Unable to allocate memory for public key data");
    }
    if(fread(pubkey, 1, pubkey_len, fd) != pubkey_len) {
        LIBSSH2_FREE(session, pubkey);
        fclose(fd);
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                              "Unable to read public key from file");
    }
    fclose(fd);
    # 移除尾部空白
    while(pubkey_len && isspace(pubkey[pubkey_len - 1])) {
        pubkey_len--;
    }
}
    # 如果公钥长度为0，则释放内存并返回缺少公钥数据的错误
    if(!pubkey_len) {
        LIBSSH2_FREE(session, pubkey);
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                              "Missing public key data");
    }

    # 在公钥数据中查找空格，如果找不到则释放内存并返回无效的公钥数据错误
    sp1 = memchr(pubkey, ' ', pubkey_len);
    if(sp1 == NULL) {
        LIBSSH2_FREE(session, pubkey);
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                              "Invalid public key data");
    }

    # 移动指针到空格后面的位置
    sp1++;

    # 计算空格前面的子串长度
    sp_len = sp1 > pubkey ? (sp1 - pubkey) : 0;
    # 在空格后面的子串中查找下一个空格
    sp2 = memchr(sp1, ' ', pubkey_len - sp_len);
    # 如果找不到，则假设 id 字符串缺失，但仍然返回成功
    if(sp2 == NULL) {
        sp2 = pubkey + pubkey_len;
    }

    # 使用 base64 解码公钥数据，如果失败则释放内存并返回无效的密钥数据，非 base64 编码错误
    if(libssh2_base64_decode(session, (char **) &tmp, &tmp_len,
                              (char *) sp1, sp2 - sp1)) {
        LIBSSH2_FREE(session, pubkey);
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                              "Invalid key data, not base64 encoded");
    }

    # 将公钥数据指针赋值给 method，计算 method 长度
    *method = pubkey;
    *method_len = sp1 - pubkey - 1;

    # 将解码后的公钥数据指针赋值给 pubkeydata，计算 pubkeydata 长度
    *pubkeydata = tmp;
    *pubkeydata_len = tmp_len;

    # 返回成功
    return 0;
# 定义一个静态函数，用于从内存中读取私钥
static int
memory_read_privatekey(LIBSSH2_SESSION * session,
                       const LIBSSH2_HOSTKEY_METHOD ** hostkey_method,
                       void **hostkey_abstract,
                       const unsigned char *method, int method_len,
                       const char *privkeyfiledata, size_t privkeyfiledata_len,
                       const char *passphrase)
{
    # 获取可用的主机密钥方法
    const LIBSSH2_HOSTKEY_METHOD **hostkey_methods_avail =
        libssh2_hostkey_methods();

    # 初始化主机密钥方法和抽象主机密钥
    *hostkey_method = NULL;
    *hostkey_abstract = NULL;
    # 遍历可用的主机密钥方法
    while(*hostkey_methods_avail && (*hostkey_methods_avail)->name) {
        # 如果找到与指定方法匹配的主机密钥方法，则设置主机密钥方法并跳出循环
        if((*hostkey_methods_avail)->initPEMFromMemory
             && strncmp((*hostkey_methods_avail)->name, (const char *) method,
                        method_len) == 0) {
            *hostkey_method = *hostkey_methods_avail;
            break;
        }
        hostkey_methods_avail++;
    }
    # 如果未找到主机密钥方法，则返回错误
    if(!*hostkey_method) {
        return _libssh2_error(session, LIBSSH2_ERROR_METHOD_NONE,
                              "No handler for specified private key");
    }

    # 使用主机密钥方法初始化私钥
    if((*hostkey_method)->
        initPEMFromMemory(session, privkeyfiledata, privkeyfiledata_len,
                          (unsigned char *) passphrase,
                          hostkey_abstract)) {
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                              "Unable to initialize private key from file");
    }

    # 返回成功
    return 0;
}

/* libssh2_file_read_privatekey
 * Read a PEM encoded private key from an id_??? style file
 */
# 定义一个静态函数，用于从文件中读取私钥
static int
file_read_privatekey(LIBSSH2_SESSION * session,
                     const LIBSSH2_HOSTKEY_METHOD ** hostkey_method,
                     void **hostkey_abstract,
                     const unsigned char *method, int method_len,
                     const char *privkeyfile, const char *passphrase)
{
    # 获取可用的主机密钥方法
    const LIBSSH2_HOSTKEY_METHOD **hostkey_methods_avail =
        libssh2_hostkey_methods();
    # 输出调试信息，加载私钥文件
    _libssh2_debug(session, LIBSSH2_TRACE_AUTH, "Loading private key file: %s",
                   privkeyfile);
    # 初始化指针，指向空值
    *hostkey_method = NULL;
    *hostkey_abstract = NULL;
    # 遍历可用的主机密钥方法，直到找到与指定方法匹配的方法
    while(*hostkey_methods_avail && (*hostkey_methods_avail)->name) {
        if((*hostkey_methods_avail)->initPEM
            && strncmp((*hostkey_methods_avail)->name, (const char *) method,
                       method_len) == 0) {
            *hostkey_method = *hostkey_methods_avail;
            break;
        }
        hostkey_methods_avail++;
    }
    # 如果没有找到与指定方法匹配的方法，则返回错误
    if(!*hostkey_method) {
        return _libssh2_error(session, LIBSSH2_ERROR_METHOD_NONE,
                              "No handler for specified private key");
    }
    # 使用指定的方法初始化私钥
    if((*hostkey_method)->
        initPEM(session, privkeyfile, (unsigned char *) passphrase,
                hostkey_abstract)) {
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                              "Unable to initialize private key from file");
    }
    # 返回成功
    return 0;
}

// 定义一个结构体，包含私钥文件名和密码
struct privkey_file {
    const char *filename;
    const char *passphrase;
};

// 从内存中签名
static int
sign_frommemory(LIBSSH2_SESSION *session, unsigned char **sig, size_t *sig_len,
                const unsigned char *data, size_t data_len, void **abstract)
{
    // 将抽象指针转换为私钥文件结构体指针
    struct privkey_file *pk_file = (struct privkey_file *) (*abstract);
    const LIBSSH2_HOSTKEY_METHOD *privkeyobj;
    void *hostkey_abstract;
    struct iovec datavec;
    int rc;

    // 从内存中读取私钥
    rc = memory_read_privatekey(session, &privkeyobj, &hostkey_abstract,
                                session->userauth_pblc_method,
                                session->userauth_pblc_method_len,
                                pk_file->filename,
                                strlen(pk_file->filename),
                                pk_file->passphrase);
    if(rc)
        return rc;

    // 准备数据向量
    libssh2_prepare_iovec(&datavec, 1);
    datavec.iov_base = (void *)data;
    datavec.iov_len  = data_len;

    // 使用私钥对象进行签名
    if(privkeyobj->signv(session, sig, sig_len, 1, &datavec,
                          &hostkey_abstract)) {
        if(privkeyobj->dtor) {
            privkeyobj->dtor(session, &hostkey_abstract);
        }
        return -1;
    }

    if(privkeyobj->dtor) {
        privkeyobj->dtor(session, &hostkey_abstract);
    }
    return 0;
}

// 从文件中签名
static int
sign_fromfile(LIBSSH2_SESSION *session, unsigned char **sig, size_t *sig_len,
              const unsigned char *data, size_t data_len, void **abstract)
{
    // 将抽象指针转换为私钥文件结构体指针
    struct privkey_file *privkey_file = (struct privkey_file *) (*abstract);
    const LIBSSH2_HOSTKEY_METHOD *privkeyobj;
    void *hostkey_abstract;
    struct iovec datavec;
    int rc;

    // 从文件中读取私钥
    rc = file_read_privatekey(session, &privkeyobj, &hostkey_abstract,
                              session->userauth_pblc_method,
                              session->userauth_pblc_method_len,
                              privkey_file->filename,
                              privkey_file->passphrase);
    if(rc)
        return rc;
    # 准备一个 I/O 向量，长度为 1
    libssh2_prepare_iovec(&datavec, 1);
    # 设置 I/O 向量的基地址为数据的地址
    datavec.iov_base = (void *)data;
    # 设置 I/O 向量的长度为数据的长度
    datavec.iov_len  = data_len;
    
    # 如果私钥对象的签名函数返回非零值，表示签名失败
    if(privkeyobj->signv(session, sig, sig_len, 1, &datavec, &hostkey_abstract)) {
        # 如果私钥对象有析构函数，则调用析构函数
        if(privkeyobj->dtor) {
            privkeyobj->dtor(session, &hostkey_abstract);
        }
        # 返回 -1，表示签名失败
        return -1;
    }
    
    # 如果私钥对象有析构函数，则调用析构函数
    if(privkeyobj->dtor) {
        privkeyobj->dtor(session, &hostkey_abstract);
    }
    # 返回 0，表示签名成功
    return 0;
/* userauth_hostbased_fromfile
 * 从指定文件中使用密钥对进行身份验证
 */
static int
userauth_hostbased_fromfile(LIBSSH2_SESSION *session,
                            const char *username, size_t username_len,
                            const char *publickey, const char *privatekey,
                            const char *passphrase, const char *hostname,
                            size_t hostname_len,
                            const char *local_username,
                            size_t local_username_len)
{
    int rc;  // 定义整型变量 rc

    // 检查用户身份验证状态是否为创建状态
    if(session->userauth_host_state == libssh2_NB_state_created) {
        // 发送用户身份验证请求包
        rc = _libssh2_transport_send(session, session->userauth_host_packet,
                                     session->userauth_host_s -
                                     session->userauth_host_packet,
                                     NULL, 0);
        // 如果返回值为 LIBSSH2_ERROR_EAGAIN，表示需要再次尝试
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                                  "Would block");
        }
        // 如果返回值不为 0，表示发送请求失败
        else if(rc) {
            LIBSSH2_FREE(session, session->userauth_host_packet);
            session->userauth_host_packet = NULL;
            session->userauth_host_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send userauth-hostbased request");
        }
        // 释放用户身份验证请求包
        LIBSSH2_FREE(session, session->userauth_host_packet);
        session->userauth_host_packet = NULL;

        // 设置用户身份验证状态为已发送
        session->userauth_host_state = libssh2_NB_state_sent;
    }
}
    # 如果用户认证主机状态为已发送
    if(session->userauth_host_state == libssh2_NB_state_sent) {
        # 静态数组，包含回复代码
        static const unsigned char reply_codes[3] =
            { SSH_MSG_USERAUTH_SUCCESS, SSH_MSG_USERAUTH_FAILURE, 0 };
        # 数据长度
        size_t data_len;
        # 要求接收指定的回复代码
        rc = _libssh2_packet_requirev(session, reply_codes,
                                      &session->userauth_host_data,
                                      &data_len, 0, NULL, 0,
                                      &session->
                                      userauth_host_packet_requirev_state);
        # 如果返回值为 LIBSSH2_ERROR_EAGAIN，表示需要再次调用该函数
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                                  "Would block");
        }

        # 用户认证主机状态设为空闲
        session->userauth_host_state = libssh2_NB_state_idle;
        # 如果返回值不为0，或者数据长度小于1，表示认证失败
        if(rc || data_len < 1) {
            return _libssh2_error(session, LIBSSH2_ERROR_PUBLICKEY_UNVERIFIED,
                                  "Auth failed");
        }

        # 如果回复代码为 SSH_MSG_USERAUTH_SUCCESS，表示主机认证成功
        if(session->userauth_host_data[0] == SSH_MSG_USERAUTH_SUCCESS) {
            _libssh2_debug(session, LIBSSH2_TRACE_AUTH,
                           "Hostbased authentication successful");
            /* We are us and we've proved it. */
            # 释放用户认证主机数据的内存
            LIBSSH2_FREE(session, session->userauth_host_data);
            session->userauth_host_data = NULL;
            # 设置会话状态为已认证
            session->state |= LIBSSH2_STATE_AUTHENTICATED;
            return 0;
        }
    }

    # 释放用户认证主机数据的内存
    LIBSSH2_FREE(session, session->userauth_host_data);
    session->userauth_host_data = NULL;
    # 返回无效签名或用户名/公钥组合错误的错误信息
    return _libssh2_error(session, LIBSSH2_ERROR_PUBLICKEY_UNVERIFIED,
                          "Invalid signature for supplied public key, or bad "
                          "username/public key combination");
# 使用主机名和文件中的密钥对进行身份验证
LIBSSH2_API int
libssh2_userauth_hostbased_fromfile_ex(LIBSSH2_SESSION *session,
                                       const char *user,
                                       unsigned int user_len,
                                       const char *publickey,
                                       const char *privatekey,
                                       const char *passphrase,
                                       const char *host,
                                       unsigned int host_len,
                                       const char *localuser,
                                       unsigned int localuser_len)
{
    int rc;
    # 调整块大小
    BLOCK_ADJUST(rc, session,
                 userauth_hostbased_fromfile(session, user, user_len,
                                             publickey, privatekey,
                                             passphrase, host, host_len,
                                             localuser, localuser_len));
    return rc;
}

# 计算方法的长度
static int plain_method_len(const char *method, size_t method_len)
{
    if(!strncmp("ecdsa-sha2-nistp256-cert-v01@openssh.com",
                method,
                method_len) ||
       !strncmp("ecdsa-sha2-nistp384-cert-v01@openssh.com",
                method,
                method_len) ||
       !strncmp("ecdsa-sha2-nistp521-cert-v01@openssh.com",
                method,
                method_len)) {
        return 19;
    }
    return method_len;
}

# 使用公钥进行身份验证
int
_libssh2_userauth_publickey(LIBSSH2_SESSION *session,
                            const char *username,
                            unsigned int username_len,
                            const unsigned char *pubkeydata,
                            unsigned long pubkeydata_len,
                            LIBSSH2_USERAUTH_PUBLICKEY_SIGN_FUNC
                            ((*sign_callback)),
                            void *abstract)
{
    # 定义一个包含 SSH 回复代码的无符号字符数组
    unsigned char reply_codes[4] =
        { SSH_MSG_USERAUTH_SUCCESS, SSH_MSG_USERAUTH_FAILURE,
          SSH_MSG_USERAUTH_PK_OK, 0
        };
    # 定义变量 rc 为整数类型
    int rc;
    # 定义指针 s 为无符号字符类型

    }

    # 如果用户认证公钥状态为 libssh2_NB_state_created
    if(session->userauth_pblc_state == libssh2_NB_state_created) {
        # 发送用户认证公钥数据包
        rc = _libssh2_transport_send(session, session->userauth_pblc_packet,
                                     session->userauth_pblc_packet_len,
                                     NULL, 0);
        # 如果返回值为 LIBSSH2_ERROR_EAGAIN，则返回错误信息 "Would block"
        if(rc == LIBSSH2_ERROR_EAGAIN)
            return _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                                  "Would block");
        # 如果返回值为非零，则释放相关资源并返回错误信息
        else if(rc) {
            LIBSSH2_FREE(session, session->userauth_pblc_packet);
            session->userauth_pblc_packet = NULL;
            LIBSSH2_FREE(session, session->userauth_pblc_method);
            session->userauth_pblc_method = NULL;
            session->userauth_pblc_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send userauth-publickey request");
        }

        # 设置用户认证公钥状态为 libssh2_NB_state_sent
        session->userauth_pblc_state = libssh2_NB_state_sent;
    }

    }

    }
    # 如果用户认证公钥状态为已发送2，则执行以下代码块
    if(session->userauth_pblc_state == libssh2_NB_state_sent2) {
        # 发送用户认证公钥数据包
        rc = _libssh2_transport_send(session, session->userauth_pblc_packet,
                                     session->userauth_pblc_s -
                                     session->userauth_pblc_packet,
                                     NULL, 0);
        # 如果返回值为LIBSSH2_ERROR_EAGAIN，则表示需要再次发送
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                                  "Would block");
        }
        # 如果返回值不为0，则表示发送失败
        else if(rc) {
            # 释放用户认证公钥数据包的内存
            LIBSSH2_FREE(session, session->userauth_pblc_packet);
            session->userauth_pblc_packet = NULL;
            # 将用户认证公钥状态设置为闲置
            session->userauth_pblc_state = libssh2_NB_state_idle;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send userauth-publickey request");
        }
        # 释放用户认证公钥数据包的内存
        LIBSSH2_FREE(session, session->userauth_pblc_packet);
        session->userauth_pblc_packet = NULL;

        # 将用户认证公钥状态设置为已发送3
        session->userauth_pblc_state = libssh2_NB_state_sent3;
    }

    # 将回复代码数组的第3个元素设置为0
    reply_codes[2] = 0;

    # 请求用户认证公钥数据包
    rc = _libssh2_packet_requirev(session, reply_codes,
                               &session->userauth_pblc_data,
                               &session->userauth_pblc_data_len, 0, NULL, 0,
                               &session->userauth_pblc_packet_requirev_state);
    # 如果返回值为LIBSSH2_ERROR_EAGAIN，则表示需要再次请求
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        return _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                              "Would block requesting userauth list");
    }
    # 如果返回值不为0，或者用户认证公钥数据长度小于1，则表示请求失败
    else if(rc || session->userauth_pblc_data_len < 1) {
        # 将用户认证公钥状态设置为闲置
        session->userauth_pblc_state = libssh2_NB_state_idle;
        return _libssh2_error(session, LIBSSH2_ERROR_PUBLICKEY_UNVERIFIED,
                              "Waiting for publickey USERAUTH response");
    }
    # 如果会话中的用户认证公钥数据的第一个元素等于SSH_MSG_USERAUTH_SUCCESS
    if(session->userauth_pblc_data[0] == SSH_MSG_USERAUTH_SUCCESS) {
        # 在调试日志中记录公钥认证成功的信息
        _libssh2_debug(session, LIBSSH2_TRACE_AUTH,
                       "Publickey authentication successful");
        # 释放用户认证公钥数据的内存空间，并将指针置为空
        LIBSSH2_FREE(session, session->userauth_pblc_data);
        session->userauth_pblc_data = NULL;
        # 将会话状态设置为已认证
        session->state |= LIBSSH2_STATE_AUTHENTICATED;
        # 将用户认证公钥状态设置为空闲状态
        session->userauth_pblc_state = libssh2_NB_state_idle;
        # 返回成功
        return 0;
    }

    # 如果公钥不被允许用于该用户在该服务器上
    # 释放用户认证公钥数据的内存空间，并将指针置为空
    LIBSSH2_FREE(session, session->userauth_pblc_data);
    session->userauth_pblc_data = NULL;
    # 将用户认证公钥状态设置为空闲状态
    session->userauth_pblc_state = libssh2_NB_state_idle;
    # 返回公钥未验证的错误，并提供详细信息
    return _libssh2_error(session, LIBSSH2_ERROR_PUBLICKEY_UNVERIFIED,
                          "Invalid signature for supplied public key, or bad "
                          "username/public key combination");
}

 /*
  * userauth_publickey_frommemory
  * 从内存中使用密钥对进行身份验证
  */
static int
userauth_publickey_frommemory(LIBSSH2_SESSION *session,
                              const char *username,
                              size_t username_len,
                              const char *publickeydata,
                              size_t publickeydata_len,
                              const char *privatekeydata,
                              size_t privatekeydata_len,
                              const char *passphrase)
{
    unsigned char *pubkeydata = NULL;  // 初始化公钥数据为空
    size_t pubkeydata_len = 0;  // 初始化公钥数据长度为0
    struct privkey_file privkey_file;  // 创建私钥文件结构体
    void *abstract = &privkey_file;  // 抽象指针指向私钥文件结构体
    int rc;  // 定义整型变量rc

    privkey_file.filename = privatekeydata;  // 将私钥数据赋值给私钥文件结构体的文件名
    privkey_file.passphrase = passphrase;  // 将密码赋值给私钥文件结构体的密码
    // 如果用户认证公钥状态为闲置
    if(session->userauth_pblc_state == libssh2_NB_state_idle) {
        // 如果有公钥数据
        if(publickeydata_len && publickeydata) {
            // 从内存中读取公钥数据
            rc = memory_read_publickey(session, &session->userauth_pblc_method,
                                       &session->userauth_pblc_method_len,
                                       &pubkeydata, &pubkeydata_len,
                                       publickeydata, publickeydata_len);
            // 如果出现错误，返回错误码
            if(rc)
                return rc;
        }
        // 如果有私钥数据
        else if(privatekeydata_len && privatekeydata) {
            // 从私钥数据中计算公钥
            rc = _libssh2_pub_priv_keyfilememory(session,
                                            &session->userauth_pblc_method,
                                            &session->userauth_pblc_method_len,
                                            &pubkeydata, &pubkeydata_len,
                                            privatekeydata, privatekeydata_len,
                                            passphrase);
            // 如果出现错误，返回错误码
            if(rc)
                return rc;
        }
        // 如果既没有公钥数据也没有私钥数据，返回错误
        else {
            return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                                  "Invalid data in public and private key.");
        }
    }

    // 使用公钥进行用户认证
    rc = _libssh2_userauth_publickey(session, username, username_len,
                                     pubkeydata, pubkeydata_len,
                                     sign_frommemory, &abstract);
    // 如果公钥数据存在，释放内存
    if(pubkeydata)
        LIBSSH2_FREE(session, pubkeydata);

    // 返回用户认证结果
    return rc;
}  // 结束函数 userauth_publickey_fromfile

/*
 * userauth_publickey_fromfile
 * 从指定文件中使用密钥对进行身份验证
 */
static int
userauth_publickey_fromfile(LIBSSH2_SESSION *session,
                            const char *username,
                            size_t username_len,
                            const char *publickey,
                            const char *privatekey,
                            const char *passphrase)
{
    unsigned char *pubkeydata = NULL;  // 初始化公钥数据为空
    size_t pubkeydata_len = 0;  // 初始化公钥数据长度为0
    struct privkey_file privkey_file;  // 创建私钥文件结构体
    void *abstract = &privkey_file;  // 抽象指针指向私钥文件结构体
    int rc;  // 定义整型变量 rc 用于存储返回值

    privkey_file.filename = privatekey;  // 将私钥文件名赋值给私钥文件结构体
    privkey_file.passphrase = passphrase;  // 将私钥密码赋值给私钥文件结构体

    if(session->userauth_pblc_state == libssh2_NB_state_idle) {  // 如果用户身份验证状态为闲置
        if(publickey) {  // 如果存在公钥
            rc = file_read_publickey(session, &session->userauth_pblc_method,
                                     &session->userauth_pblc_method_len,
                                     &pubkeydata, &pubkeydata_len, publickey);  // 读取公钥文件内容
            if(rc)  // 如果读取失败
                return rc;  // 返回错误码
        }
        else {
            /* Compute public key from private key. */
            rc = _libssh2_pub_priv_keyfile(session,
                                           &session->userauth_pblc_method,
                                           &session->userauth_pblc_method_len,
                                           &pubkeydata, &pubkeydata_len,
                                           privatekey, passphrase);  // 从私钥文件计算公钥

            /* _libssh2_pub_priv_keyfile calls _libssh2_error() */
            if(rc)  // 如果计算失败
                return rc;  // 返回错误码
        }
    }

    rc = _libssh2_userauth_publickey(session, username, username_len,
                                     pubkeydata, pubkeydata_len,
                                     sign_fromfile, &abstract);  // 使用公钥进行身份验证
    if(pubkeydata)  // 如果公钥数据存在
        LIBSSH2_FREE(session, pubkeydata);  // 释放公钥数据内存

    return rc;  // 返回身份验证结果
}

/* libssh2_userauth_publickey_frommemory
 * 从内存中使用密钥对进行身份验证
 */
LIBSSH2_API int
# 从内存中的公钥和私钥进行用户认证
libssh2_userauth_publickey_frommemory(LIBSSH2_SESSION *session,
                                      const char *user,
                                      size_t user_len,
                                      const char *publickeyfiledata,
                                      size_t publickeyfiledata_len,
                                      const char *privatekeyfiledata,
                                      size_t privatekeyfiledata_len,
                                      const char *passphrase)
{
    int rc;

    # 如果传入的 passphrase 是 NULL 指针，则将其指向一个长度为零的字符串，以免在多处重复检查
    if(NULL == passphrase)
        passphrase = "";

    # 调整块，调用 userauth_publickey_frommemory 函数进行用户认证
    BLOCK_ADJUST(rc, session,
                 userauth_publickey_frommemory(session, user, user_len,
                                               publickeyfiledata,
                                               publickeyfiledata_len,
                                               privatekeyfiledata,
                                               privatekeyfiledata_len,
                                               passphrase));
    # 返回用户认证结果
    return rc;
}

/* libssh2_userauth_publickey_fromfile_ex
 * 使用指定文件中的密钥对进行认证
 */
LIBSSH2_API int
libssh2_userauth_publickey_fromfile_ex(LIBSSH2_SESSION *session,
                                       const char *user,
                                       unsigned int user_len,
                                       const char *publickey,
                                       const char *privatekey,
                                       const char *passphrase)
{
    int rc;

    # 如果传入的 passphrase 是 NULL 指针，则将其指向一个长度为零的字符串，以免在多处重复检查
    if(NULL == passphrase)
        passphrase = "";
    # 调用 BLOCK_ADJUST 函数，传入参数 rc, session, 调用 userauth_publickey_fromfile 函数并传入参数 session, user, user_len, publickey, privatekey, passphrase
    BLOCK_ADJUST(rc, session,
                 userauth_publickey_fromfile(session, user, user_len,
                                             publickey, privatekey,
                                             passphrase));
    # 返回 rc 变量的值
    return rc;
}

/* libssh2_userauth_publickey_ex
 * 使用外部回调函数进行身份验证
 */
LIBSSH2_API int
libssh2_userauth_publickey(LIBSSH2_SESSION *session,
                           const char *user,
                           const unsigned char *pubkeydata,
                           size_t pubkeydata_len,
                           LIBSSH2_USERAUTH_PUBLICKEY_SIGN_FUNC
                           ((*sign_callback)),
                           void **abstract)
{
    int rc;

    if(!session)
        return LIBSSH2_ERROR_BAD_USE;

    BLOCK_ADJUST(rc, session,
                 _libssh2_userauth_publickey(session, user, strlen(user),
                                             pubkeydata, pubkeydata_len,
                                             sign_callback, abstract));
    return rc;
}



/*
 * userauth_keyboard_interactive
 * 使用挑战-响应身份验证进行身份验证
 */
static int
userauth_keyboard_interactive(LIBSSH2_SESSION * session,
                              const char *username,
                              unsigned int username_len,
                              LIBSSH2_USERAUTH_KBDINT_RESPONSE_FUNC
                              ((*response_callback)))
{
    unsigned char *s;
    int rc;

    static const unsigned char reply_codes[4] = {
        SSH_MSG_USERAUTH_SUCCESS,
        SSH_MSG_USERAUTH_FAILURE, SSH_MSG_USERAUTH_INFO_REQUEST, 0
    };
    unsigned int language_tag_len;
    unsigned int i;

    }
    # 如果用户键盘交互状态为已创建
    if(session->userauth_kybd_state == libssh2_NB_state_created) {
        # 发送用户键盘交互数据包
        rc = _libssh2_transport_send(session, session->userauth_kybd_data,
                                     session->userauth_kybd_packet_len,
                                     NULL, 0);
        # 如果返回值为 LIBSSH2_ERROR_EAGAIN，表示需要再次尝试发送
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            # 返回“Would block”错误
            return _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                                  "Would block");
        }
        # 如果返回值不为 0
        else if(rc) {
            # 释放用户键盘交互数据
            LIBSSH2_FREE(session, session->userauth_kybd_data);
            session->userauth_kybd_data = NULL;
            # 将用户键盘交互状态设置为闲置
            session->userauth_kybd_state = libssh2_NB_state_idle;
            # 返回“Unable to send keyboard-interactive request”错误
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send keyboard-interactive"
                                  " request");
        }
        # 释放用户键盘交互数据
        LIBSSH2_FREE(session, session->userauth_kybd_data);
        session->userauth_kybd_data = NULL;
        # 将用户键盘交互状态设置为已发送
        session->userauth_kybd_state = libssh2_NB_state_sent;
    }
}
/*
 * libssh2_userauth_keyboard_interactive_ex
 *
 * 使用挑战-响应身份验证进行认证
 */
LIBSSH2_API int
libssh2_userauth_keyboard_interactive_ex(LIBSSH2_SESSION *session,
                                         const char *user,
                                         unsigned int user_len,
                                         LIBSSH2_USERAUTH_KBDINT_RESPONSE_FUNC
                                         ((*response_callback)))
{
    int rc;
    BLOCK_ADJUST(rc, session,
                 userauth_keyboard_interactive(session, user, user_len,
                                               response_callback));
    return rc;
}
```