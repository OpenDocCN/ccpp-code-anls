# `nmap\libssh2\src\knownhost.c`

```cpp
/*
 * 版权声明，版权所有
 * 作者：Daniel Stenberg
 * 禁止在未经许可的情况下以源代码或二进制形式重新分发和使用
 * 如果满足以下条件：
 *   1. 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明
 *   2. 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 * 禁止使用版权所有者的名称或其他贡献者的名称，未经特定事先书面许可，不得用于认可或推广从本软件衍生的产品
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途适用性的暗示担保
 * 在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）产生的任何直接、间接、附带、特殊、惩罚性或后果性损害，包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断，版权所有者或贡献者均不承担任何责任
 */

#include "libssh2_priv.h"  // 引入 libssh2_priv.h 头文件
#include "misc.h"  // 引入 misc.h 头文件

struct known_host {
    struct list_node node;  // 已知主机的链表节点
    char *name;      /* 指向名称或哈希值（已分配内存） */
    size_t name_len; /* 哈希数据所需的长度 */
    int port;        /* 如果非零，表示此密钥适用于此主机上的特定端口 */
    int typemask;    /* 普通、sha1、自定义等类型掩码 */
    /* 指向二进制盐值的指针（已分配内存） */
    char *salt;
    /* 盐值的大小 */
    size_t salt_len;
    /* 关联密钥（已分配内存）。在内存中以 base64 编码形式保存 */
    char *key;
    /* 密钥类型名称（已分配内存） */
    char *key_type_name;
    /* 密钥类型名称的大小 */
    size_t key_type_len;
    /* 可选的注释文本（已分配内存），可能为 NULL */
    char *comment;
    /* 注释的大小 */
    size_t comment_len;

    /* 这是我们在外部公开的结构体 */
    struct libssh2_knownhost external;
};

// 定义一个结构体，用于存储已知主机的信息
struct _LIBSSH2_KNOWNHOSTS
{
    LIBSSH2_SESSION *session;  // 指向所属会话的指针
    struct list_head head;  // 用于存储已知主机信息的链表头部
};

// 释放已知主机的内存
static void free_host(LIBSSH2_SESSION *session, struct known_host *entry)
{
    if(entry) {
        if(entry->comment)
            LIBSSH2_FREE(session, entry->comment);
        if(entry->key_type_name)
            LIBSSH2_FREE(session, entry->key_type_name);
        if(entry->key)
            LIBSSH2_FREE(session, entry->key);
        if(entry->salt)
            LIBSSH2_FREE(session, entry->salt);
        if(entry->name)
            LIBSSH2_FREE(session, entry->name);
        LIBSSH2_FREE(session, entry);
    }
}

/*
 * libssh2_knownhost_init
 *
 * 初始化已知主机的集合。返回指向集合的指针。
 *
 */
LIBSSH2_API LIBSSH2_KNOWNHOSTS *
libssh2_knownhost_init(LIBSSH2_SESSION *session)
{
    LIBSSH2_KNOWNHOSTS *knh =
        LIBSSH2_ALLOC(session, sizeof(struct _LIBSSH2_KNOWNHOSTS));

    if(!knh) {
        _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                       "Unable to allocate memory for known-hosts "
                       "collection");
        return NULL;
    }

    knh->session = session;

    _libssh2_list_init(&knh->head);

    return knh;
}

#define KNOWNHOST_MAGIC 0xdeadcafe
/*
 * knownhost_to_external()
 *
 * 从内部复制数据到外部表示的结构体。
 *
 */
static struct libssh2_knownhost *knownhost_to_external(struct known_host *node)
{
    struct libssh2_knownhost *ext = &node->external;

    ext->magic = KNOWNHOST_MAGIC;
    ext->node = node;
    ext->name = ((node->typemask & LIBSSH2_KNOWNHOST_TYPE_MASK) ==
                 LIBSSH2_KNOWNHOST_TYPE_PLAIN)? node->name:NULL;
    ext->key = node->key;
    ext->typemask = node->typemask;

    return ext;
}

static int
def knownhost_add(LIBSSH2_KNOWNHOSTS *hosts,
                  const char *host, const char *salt,
                  const char *key_type_name, size_t key_type_len,
                  const char *key, size_t keylen,
                  const char *comment, size_t commentlen,
                  int typemask, struct libssh2_knownhost **store)
{
    struct known_host *entry;  # 声明一个名为entry的known_host结构体指针
    size_t hostlen = strlen(host);  # 获取host字符串的长度
    int rc;  # 声明一个整型变量rc
    char *ptr;  # 声明一个字符指针ptr
    unsigned int ptrlen;  # 声明一个无符号整型变量ptrlen

    /* make sure we have a key type set */
    if(!(typemask & LIBSSH2_KNOWNHOST_KEY_MASK))  # 检查typemask是否包含LIBSSH2_KNOWNHOST_KEY_MASK
        return _libssh2_error(hosts->session, LIBSSH2_ERROR_INVAL,
                              "No key type set");  # 如果不包含，则返回错误信息

    entry = LIBSSH2_CALLOC(hosts->session, sizeof(struct known_host));  # 分配内存给entry
    if(!entry)  # 如果entry为空
        return _libssh2_error(hosts->session, LIBSSH2_ERROR_ALLOC,
                              "Unable to allocate memory for known host "
                              "entry");  # 返回内存分配错误信息

    entry->typemask = typemask;  # 将typemask赋值给entry的typemask

    switch(entry->typemask  & LIBSSH2_KNOWNHOST_TYPE_MASK) {  # 根据typemask的值进行不同的操作
    case LIBSSH2_KNOWNHOST_TYPE_PLAIN:  # 如果typemask的类型是PLAIN
    case LIBSSH2_KNOWNHOST_TYPE_CUSTOM:  # 或者是CUSTOM
        entry->name = LIBSSH2_ALLOC(hosts->session, hostlen + 1);  # 分配内存给entry的name
        if(!entry->name) {  # 如果entry的name为空
            rc = _libssh2_error(hosts->session, LIBSSH2_ERROR_ALLOC,
                                "Unable to allocate memory for host name");  # 返回内存分配错误信息
            goto error;  # 跳转到error标签
        }
        memcpy(entry->name, host, hostlen + 1);  # 将host的内容复制到entry的name
        entry->name_len = hostlen;  # 设置entry的name_len为hostlen
        break;  # 结束switch语句
    case LIBSSH2_KNOWNHOST_TYPE_SHA1:  # 如果typemask的类型是SHA1
        rc = libssh2_base64_decode(hosts->session, &ptr, &ptrlen,
                                   host, hostlen);  # 对host进行base64解码
        if(rc)  # 如果解码失败
            goto error;  # 跳转到error标签
        entry->name = ptr;  # 将解码后的内容赋值给entry的name
        entry->name_len = ptrlen;  # 设置entry的name_len为ptrlen

        rc = libssh2_base64_decode(hosts->session, &ptr, &ptrlen,
                                   salt, strlen(salt));  # 对salt进行base64解码
        if(rc)  # 如果解码失败
            goto error;  # 跳转到error标签
        entry->salt = ptr;  # 将解码后的内容赋值给entry的salt
        entry->salt_len = ptrlen;  # 设置entry的salt_len为ptrlen
        break;  # 结束switch语句
    # 默认情况下，如果未知主机名类型，则返回未知主机名类型错误
    default:
        rc = _libssh2_error(hosts->session, LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                            "Unknown host name type");
        goto error;
    }

    # 如果类型掩码包含 BASE64 编码，则处理 BASE64 编码的密钥
    if(typemask & LIBSSH2_KNOWNHOST_KEYENC_BASE64) {
        /* the provided key is base64 encoded already */
        # 如果提供的密钥已经是 BASE64 编码，则直接使用
        if(!keylen)
            keylen = strlen(key);
        entry->key = LIBSSH2_ALLOC(hosts->session, keylen + 1);
        if(!entry->key) {
            rc = _libssh2_error(hosts->session, LIBSSH2_ERROR_ALLOC,
                                "Unable to allocate memory for key");
            goto error;
        }
        memcpy(entry->key, key, keylen + 1);
        entry->key[keylen] = 0; /* force a terminating zero trailer */
    }
    else {
        /* key is raw, we base64 encode it and store it as such */
        # 如果密钥是原始的，则进行 BASE64 编码并存储
        size_t nlen = _libssh2_base64_encode(hosts->session, key, keylen,
                                             &ptr);
        if(!nlen) {
            rc = _libssh2_error(hosts->session, LIBSSH2_ERROR_ALLOC,
                                "Unable to allocate memory for "
                                "base64-encoded key");
            goto error;
        }

        entry->key = ptr;
    }

    # 如果存在密钥类型名称且密钥类型未知，则处理密钥类型
    if(key_type_name && ((typemask & LIBSSH2_KNOWNHOST_KEY_MASK) ==
                          LIBSSH2_KNOWNHOST_KEY_UNKNOWN)) {
        entry->key_type_name = LIBSSH2_ALLOC(hosts->session, key_type_len + 1);
        if(!entry->key_type_name) {
            rc = _libssh2_error(hosts->session, LIBSSH2_ERROR_ALLOC,
                                "Unable to allocate memory for key type");
            goto error;
        }
        memcpy(entry->key_type_name, key_type_name, key_type_len);
        entry->key_type_name[key_type_len] = 0;
        entry->key_type_len = key_type_len;
    }
    # 如果有注释内容
    if(comment) {
        # 为注释分配内存空间
        entry->comment = LIBSSH2_ALLOC(hosts->session, commentlen + 1);
        # 如果内存分配失败
        if(!entry->comment) {
            # 设置错误码和错误信息
            rc = _libssh2_error(hosts->session, LIBSSH2_ERROR_ALLOC,
                                "Unable to allocate memory for comment");
            # 跳转到错误处理部分
            goto error;
        }
        # 将注释内容复制到分配的内存空间中
        memcpy(entry->comment, comment, commentlen + 1);
        # 强制添加一个终止的零
        entry->comment[commentlen] = 0; /* force a terminating zero trailer */
        # 记录注释的长度
        entry->comment_len = commentlen;
    }
    # 如果没有注释内容
    else {
        entry->comment = NULL;
    }

    # 将新的主机添加到已知主机的列表中
    _libssh2_list_add(&hosts->head, &entry->node);

    # 如果需要存储
    if(store)
        # 将已知主机转换为外部表示，并存储
        *store = knownhost_to_external(entry);

    # 返回无错误
    return LIBSSH2_ERROR_NONE;
  # 错误处理部分
  error:
    # 释放已知主机的内存
    free_host(hosts->session, entry);
    # 返回错误码
    return rc;
/*
 * libssh2_knownhost_add
 *
 * 将主机及其关联密钥添加到已知主机集合中。
 *
 * 'type' 参数指定给定主机和密钥的格式：
 *
 * plain  - ASCII格式的 "hostname.domain.tld"
 * sha1   - SHA1(<salt> <host>) 的 base64 编码！
 * custom - 其他哈希
 *
 * 如果选择 'sha1' 作为类型，则必须将盐提供给盐参数。这也是 base64 编码的。
 *
 * SHA-1 哈希是 OpenSSH 可以在 known_hosts 文件中使用的内容。如果使用自定义类型，则盐将被忽略，并且在 libssh2_knownhost_check() 函数中检查时，必须提供主机的预先哈希。
 *
 * 如果密钥以 NULL 结尾的 base64 编码字符串提供，则可以省略 keylen 参数（设置为零）。
 */

LIBSSH2_API int
libssh2_knownhost_add(LIBSSH2_KNOWNHOSTS *hosts,
                      const char *host, const char *salt,
                      const char *key, size_t keylen,
                      int typemask, struct libssh2_knownhost **store)
{
    return knownhost_add(hosts, host, salt, NULL, 0, key, keylen, NULL,
                         0, typemask, store);
}
/*
 * libssh2_knownhost_addc
 *
 * Add a host and its associated key to the collection of known hosts.
 *
 * Takes a comment argument that may be NULL.  A NULL comment indicates
 * there is no comment and the entry will end directly after the key
 * when written out to a file.  An empty string "" comment will indicate an
 * empty comment which will cause a single space to be written after the key.
 *
 * The 'type' argument specifies on what format the given host and keys are:
 *
 * plain  - ascii "hostname.domain.tld"
 * sha1   - SHA1(<salt> <host>) base64-encoded!
 * custom - another hash
 *
 * If 'sha1' is selected as type, the salt must be provided to the salt
 * argument. This too base64 encoded.
 *
 * The SHA-1 hash is what OpenSSH can be told to use in known_hosts files.  If
 * a custom type is used, salt is ignored and you must provide the host
 * pre-hashed when checking for it in the libssh2_knownhost_check() function.
 *
 * The keylen parameter may be omitted (zero) if the key is provided as a
 * NULL-terminated base64-encoded string.
 */

LIBSSH2_API int
libssh2_knownhost_addc(LIBSSH2_KNOWNHOSTS *hosts,
                       const char *host, const char *salt,
                       const char *key, size_t keylen,
                       const char *comment, size_t commentlen,
                       int typemask, struct libssh2_knownhost **store)
{
    // 调用 knownhost_add 函数，将主机和其关联的密钥添加到已知主机集合中
    return knownhost_add(hosts, host, salt, NULL, 0, key, keylen,
                         comment, commentlen, typemask, store);
}
/*
 * knownhost_check
 *
 * Check a host and its associated key against the collection of known hosts.
 *
 * The typemask is the type/format of the given host name and key
 *
 * plain  - ascii "hostname.domain.tld"
 * sha1   - NOT SUPPORTED AS INPUT
 * custom - prehashed base64 encoded. Note that this cannot use any salts.
 *
 * Returns:
 *
 * LIBSSH2_KNOWNHOST_CHECK_FAILURE
 * LIBSSH2_KNOWNHOST_CHECK_NOTFOUND
 * LIBSSH2_KNOWNHOST_CHECK_MATCH
 * LIBSSH2_KNOWNHOST_CHECK_MISMATCH
 */
static int
knownhost_check(LIBSSH2_KNOWNHOSTS *hosts,
                const char *hostp, int port,
                const char *key, size_t keylen,
                int typemask,
                struct libssh2_knownhost **ext)
{
    struct known_host *node;  // 定义已知主机节点
    struct known_host *badkey = NULL;  // 定义错误的密钥
    int type = typemask & LIBSSH2_KNOWNHOST_TYPE_MASK;  // 获取主机类型
    char *keyalloc = NULL;  // 初始化密钥分配
    int rc = LIBSSH2_KNOWNHOST_CHECK_NOTFOUND;  // 初始化返回值为未找到
    char hostbuff[270]; /* most host names can't be longer than like 256 */  // 定义主机名缓冲区
    const char *host;  // 定义主机名
    int numcheck; /* number of host combos to check */  // 定义要检查的主机组合数
    int match = 0;  // 初始化匹配值为0

    if(type == LIBSSH2_KNOWNHOST_TYPE_SHA1)
        /* we can't work with a sha1 as given input */
        return LIBSSH2_KNOWNHOST_CHECK_MISMATCH;  // 如果类型为SHA1，则返回不匹配

    /* if a port number is given, check for a '[host]:port' first before the
       plain 'host' */
    if(port >= 0) {
        int len = snprintf(hostbuff, sizeof(hostbuff), "[%s]:%d", hostp, port);  // 格式化主机名和端口号
        if(len < 0 || len >= (int)sizeof(hostbuff)) {
            _libssh2_error(hosts->session,
                           LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                           "Known-host write buffer too small");  // 如果缓冲区太小，则返回错误
            return LIBSSH2_KNOWNHOST_CHECK_FAILURE;  // 返回失败
        }
        host = hostbuff;  // 将格式化后的主机名赋值给host
        numcheck = 2; /* check both combos, start with this */  // 设置要检查的组合数为2
    }
    else {
        host = hostp;  // 如果没有端口号，则直接使用hostp作为主机名
        numcheck = 1; /* only check this host version */  // 设置要检查的组合数为1
    }
    # 如果 typemask 不包含 LIBSSH2_KNOWNHOST_KEYENC_BASE64 标志位
    if(!(typemask & LIBSSH2_KNOWNHOST_KEYENC_BASE64)) {
        # 将原始密钥转换为 base64 格式，以便进行后续的检查
        size_t nlen = _libssh2_base64_encode(hosts->session, key, keylen, &keyalloc);
        # 如果无法分配内存来存储 base64 编码的密钥，则返回检查失败
        if(!nlen) {
            _libssh2_error(hosts->session, LIBSSH2_ERROR_ALLOC, "Unable to allocate memory for base64-encoded key");
            return LIBSSH2_KNOWNHOST_CHECK_FAILURE;
        }

        # 将 key 指向新分配的 base64 编码的密钥
        key = keyalloc;
    }

    # 循环检查是否有匹配的密钥，直到找到匹配或者检查次数用尽
    } while(!match && --numcheck);

    # 如果存在不匹配的密钥
    if(badkey) {
        # 设置外部变量指向不匹配的密钥
        if(ext)
            *ext = knownhost_to_external(badkey);
        # 返回密钥不匹配的结果
        rc = LIBSSH2_KNOWNHOST_CHECK_MISMATCH;
    }

    # 如果 keyalloc 不为空，则释放内存
    if(keyalloc)
        LIBSSH2_FREE(hosts->session, keyalloc);

    # 返回检查结果
    return rc;
/*
 * libssh2_knownhost_check
 *
 * 检查主机及其关联的密钥是否在已知主机集合中
 *
 * typemask 是给定主机名和密钥的类型/格式
 *
 * plain  - ASCII格式的 "hostname.domain.tld"
 * sha1   - 不支持作为输入
 * custom - 预先哈希的Base64编码。注意这不能使用任何盐。
 *
 * 返回值:
 *
 * LIBSSH2_KNOWNHOST_CHECK_FAILURE
 * LIBSSH2_KNOWNHOST_CHECK_NOTFOUND
 * LIBSSH2_KNOWNHOST_CHECK_MATCH
 * LIBSSH2_KNOWNHOST_CHECK_MISMATCH
 */
LIBSSH2_API int
libssh2_knownhost_check(LIBSSH2_KNOWNHOSTS *hosts,
                        const char *hostp, const char *key, size_t keylen,
                        int typemask,
                        struct libssh2_knownhost **ext)
{
    return knownhost_check(hosts, hostp, -1, key, keylen,
                           typemask, ext);
}

/*
 * libssh2_knownhost_checkp
 *
 * 检查主机+端口及其关联的密钥是否在已知主机集合中
 *
 * 注意，如果 'port' 指定为大于零，检查函数将能够检查此特定主机+端口组合的专用密钥，如果 'port' 为负，则仅检查通用主机密钥。
 *
 * typemask 是给定主机名和密钥的类型/格式
 *
 * plain  - ASCII格式的 "hostname.domain.tld"
 * sha1   - 不支持作为输入
 * custom - 预先哈希的Base64编码。注意这不能使用任何盐。
 *
 * 返回值:
 *
 * LIBSSH2_KNOWNHOST_CHECK_FAILURE
 * LIBSSH2_KNOWNHOST_CHECK_NOTFOUND
 * LIBSSH2_KNOWNHOST_CHECK_MATCH
 * LIBSSH2_KNOWNHOST_CHECK_MISMATCH
 */
LIBSSH2_API int
libssh2_knownhost_checkp(LIBSSH2_KNOWNHOSTS *hosts,
                         const char *hostp, int port,
                         const char *key, size_t keylen,
                         int typemask,
                         struct libssh2_knownhost **ext)
{
    return knownhost_check(hosts, hostp, port, key, keylen,
                           typemask, ext);
}
# 删除已知主机集合中的一个主机
LIBSSH2_API int
libssh2_knownhost_del(LIBSSH2_KNOWNHOSTS *hosts,
                      struct libssh2_knownhost *entry)
{
    struct known_host *node;

    # 检查是否正确获取了主机信息，如果没有则返回错误
    if(!entry || (entry->magic != KNOWNHOST_MAGIC))
        return _libssh2_error(hosts->session, LIBSSH2_ERROR_INVAL,
                              "Invalid host information");

    # 获取内部节点指针
    node = entry->node;

    # 从所有主机列表中取消链接
    _libssh2_list_remove(&node->node);

    # 清空结构，因为即将释放内存
    memset(entry, 0, sizeof(struct libssh2_knownhost));

    # 释放所有资源
    free_host(hosts->session, node);

    return 0;
}

# 释放整个已知主机集合
LIBSSH2_API void
libssh2_knownhost_free(LIBSSH2_KNOWNHOSTS *hosts)
{
    struct known_host *node;
    struct known_host *next;

    for(node = _libssh2_list_first(&hosts->head); node; node = next) {
        next = _libssh2_list_next(&node->node);
        free_host(hosts->session, node);
    }
    LIBSSH2_FREE(hosts->session, hosts);
}

# 旧式纯文本格式：[name]([,][name])*

# 为了简单起见，我们将它们作为具有相同密钥的单独主机添加
static int oldstyle_hostline(LIBSSH2_KNOWNHOSTS *hosts,
                             const char *host, size_t hostlen,
                             const char *key_type_name, size_t key_type_len,
                             const char *key, size_t keylen, int key_type,
                             const char *comment, size_t commentlen)
{
    int rc = 0;
    size_t namelen = 0;
    const char *name = host + hostlen;
    # 如果主机名长度小于1，则返回方法不支持的错误
    if(hostlen < 1)
        return _libssh2_error(hosts->session,
                              LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                              "Failed to parse known_hosts line "
                              "(no host names)");

    # 当主机名大于主机时，执行循环
    while(name > host) {
        --name;
        ++namelen;

        # 当我们到达开始或看到逗号时，将主机名添加到集合中
        if((name == host) || (*(name-1) == ',')) {

            char hostbuf[256];

            # 确保我们不会溢出缓冲区
            if(namelen >= sizeof(hostbuf)-1)
                return _libssh2_error(hosts->session,
                                      LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                                      "Failed to parse known_hosts line "
                                      "(unexpected length)");

            # 将主机名复制到临时缓冲区并以零终止
            memcpy(hostbuf, name, namelen);
            hostbuf[namelen] = 0;

            # 添加已知主机到主机列表
            rc = knownhost_add(hosts, hostbuf, NULL,
                               key_type_name, key_type_len,
                               key, keylen,
                               comment, commentlen,
                               key_type | LIBSSH2_KNOWNHOST_TYPE_PLAIN |
                               LIBSSH2_KNOWNHOST_KEYENC_BASE64, NULL);
            if(rc)
                return rc;

            # 如果主机名大于主机，则重置namelen和name
            if(name > host) {
                namelen = 0;
                --name; # 跳过逗号
            }
        }
    }

    # 返回结果
    return rc;
// 定义一个名为hashed_hostline的静态函数，用于处理已经哈希过的主机行
static int hashed_hostline(LIBSSH2_KNOWNHOSTS *hosts,
                           const char *host, size_t hostlen,
                           const char *key_type_name, size_t key_type_len,
                           const char *key, size_t keylen, int key_type,
                           const char *comment, size_t commentlen)
{
    // 声明并初始化变量
    const char *p;
    char saltbuf[32];
    char hostbuf[256];

    // 从主机行中获取盐值
    const char *salt = &host[3]; /* skip the magic marker */
    // 减去魔术标记的长度
    hostlen -= 3;    /* deduct the marker */

    // 找到盐值的结束位置
    for(p = salt; *p && (*p != '|'); p++)
        ;
}
    # 如果当前指针指向的字符是 '|'，则执行以下操作
    if(*p == '|') {
        # 声明一个指向常量字符的指针 hash，并初始化为 NULL
        const char *hash = NULL;
        # 计算盐值的长度
        size_t saltlen = p - salt;
        # 如果盐值长度超过了 saltbuf 数组的长度减一，则返回错误
        if(saltlen >= (sizeof(saltbuf)-1)) /* weird length */
            return _libssh2_error(hosts->session,
                                  LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                                  "Failed to parse known_hosts line "
                                  "(unexpectedly long salt)");
        # 将盐值拷贝到 saltbuf 数组中
        memcpy(saltbuf, salt, saltlen);
        # 在 saltbuf 数组末尾添加空字符，以便作为字符串结尾
        saltbuf[saltlen] = 0; /* zero terminate */
        # 将 salt 指针指向 saltbuf 数组，即指向栈上的缓冲区
        salt = saltbuf; /* point to the stack based buffer */

        # 将 hash 指针指向分隔符 '|' 后面的字符
        hash = p + 1; /* the host hash is after the separator */

        # 将 host 指针指向 hash 指针指向的位置
        host = hash;
        # 减去盐值和分隔符的长度，得到主机名的长度
        hostlen -= saltlen + 1; /* deduct the salt and separator */

        # 检查主机名的长度是否合理
        if(hostlen >= sizeof(hostbuf)-1)
            return _libssh2_error(hosts->session,
                                  LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                                  "Failed to parse known_hosts line "
                                  "(unexpected length)");
        # 将主机名拷贝到 hostbuf 数组中
        memcpy(hostbuf, host, hostlen);
        # 在 hostbuf 数组末尾添加空字符，以便作为字符串结尾
        hostbuf[hostlen] = 0;

        # 调用 knownhost_add 函数，将主机名、盐值、密钥类型、密钥数据、注释等信息添加到已知主机列表中
        return knownhost_add(hosts, hostbuf, salt,
                             key_type_name, key_type_len,
                             key, keylen,
                             comment, commentlen,
                             key_type | LIBSSH2_KNOWNHOST_TYPE_SHA1 |
                             LIBSSH2_KNOWNHOST_KEYENC_BASE64, NULL);
    }
    # 如果当前指针指向的字符不是 '|'，则返回 0，表示出现错误
    else
        return 0; /* XXX: This should be an error, shouldn't it? */
}
/*
 * hostline()
 *
 * Parse a single known_host line pre-split into host and key.
 *
 * The key part may include an optional comment which will be parsed here
 * for ssh-rsa and ssh-dsa keys.  Comments in other key types aren't handled.
 *
 * The function assumes new-lines have already been removed from the arguments.
 */
static int hostline(LIBSSH2_KNOWNHOSTS *hosts,
                    const char *host, size_t hostlen,
                    const char *key, size_t keylen)
{
    const char *comment = NULL;  // 用于存储注释部分
    const char *key_type_name = NULL;  // 用于存储密钥类型名称
    size_t commentlen = 0;  // 注释部分的长度
    size_t key_type_len = 0;  // 密钥类型名称的长度
    int key_type;  // 密钥类型

    /* make some checks that the lengths seem sensible */
    if(keylen < 20)  // 如果密钥长度小于20，返回错误
        return _libssh2_error(hosts->session,
                              LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                              "Failed to parse known_hosts line "
                              "(key too short)");

    switch(key[0]) {  // 根据密钥的第一个字符进行判断
    case '0': case '1': case '2': case '3': case '4':
    case '5': case '6': case '7': case '8': case '9':
        key_type = LIBSSH2_KNOWNHOST_KEY_RSA1;  // 设置密钥类型为 RSA1

        /* Note that the old-style keys (RSA1) aren't truly base64, but we
         * claim it is for now since we can get away with strcmp()ing the
         * entire anything anyway! We need to check and fix these to make them
         * work properly.
         */
        break;
    # 默认情况下
    default:
        # 将 key_type_name 设置为 key，并且去掉开头的空格和制表符
        key_type_name = key;
        while(keylen && *key &&
               (*key != ' ') && (*key != '\t')) {
            key++;
            keylen--;
        }
        key_type_len = key - key_type_name;

        # 根据 key_type_name 的值确定 key_type
        if(!strncmp(key_type_name, "ssh-dss", key_type_len))
            key_type = LIBSSH2_KNOWNHOST_KEY_SSHDSS;
        else if(!strncmp(key_type_name, "ssh-rsa", key_type_len))
            key_type = LIBSSH2_KNOWNHOST_KEY_SSHRSA;
        else if(!strncmp(key_type_name, "ecdsa-sha2-nistp256", key_type_len))
            key_type = LIBSSH2_KNOWNHOST_KEY_ECDSA_256;
        else if(!strncmp(key_type_name, "ecdsa-sha2-nistp384", key_type_len))
            key_type = LIBSSH2_KNOWNHOST_KEY_ECDSA_384;
        else if(!strncmp(key_type_name, "ecdsa-sha2-nistp521", key_type_len))
            key_type = LIBSSH2_KNOWNHOST_KEY_ECDSA_521;
        else if(!strncmp(key_type_name, "ssh-ed25519", key_type_len))
            key_type = LIBSSH2_KNOWNHOST_KEY_ED25519;
        else
            key_type = LIBSSH2_KNOWNHOST_KEY_UNKNOWN;

        /* skip whitespaces */
        # 跳过空格和制表符
        while((*key ==' ') || (*key == '\t')) {
            key++;
            keylen--;
        }

        # 将 comment 设置为 key，并且去掉开头的空格和制表符
        comment = key;
        commentlen = keylen;

        /* move over key */
        # 移动 comment 指针，去掉开头的空格和制表符
        while(commentlen && *comment &&
              (*comment != ' ') && (*comment != '\t')) {
            comment++;
            commentlen--;
        }

        /* reduce key by comment length */
        # 减去 comment 的长度
        keylen -= commentlen;

        /* Distinguish empty comment (a space) from no comment (no space) */
        # 区分空注释（一个空格）和没有注释（没有空格）
        if(commentlen == 0)
            comment = NULL;

        /* skip whitespaces */
        # 跳过空格和制表符
        while(commentlen && *comment &&
              ((*comment ==' ') || (*comment == '\t'))) {
            comment++;
            commentlen--;
        }
        break;
    }

    /* Figure out host format */
    # 确定主机格式
    # 如果主机长度大于2并且主机与"|1|"的前3个字符不匹配
    if((hostlen >2) && memcmp(host, "|1|", 3)) {
        # 旧格式的纯文本：[name]([,][name])*
        # 为了简单起见，我们将它们作为具有相同密钥的单独主机添加
        return oldstyle_hostline(hosts, host, hostlen, key_type_name,
                                 key_type_len, key, keylen, key_type,
                                 comment, commentlen);
    }
    else {
        # |1|[salt]|[hash]
        return hashed_hostline(hosts, host, hostlen, key_type_name,
                               key_type_len, key, keylen, key_type,
                               comment, commentlen);
    }
/*
 * libssh2_knownhost_readline()
 *
 * 传入一个文件的一行内容
 *
 * LIBSSH2_KNOWNHOST_FILE_OPENSSH 是唯一支持的类型
 *
 * OpenSSH 行格式:
 *
 * <host> <key>
 *
 * 其中两部分可以这样创建:
 *
 * <host> 可以是
 * <name> 或 <hash>
 *
 * <name> 由
 * [name] 可选地后跟 [,name] 一次或多次
 *
 * <hash> 由
 * |1|<salt>|hash
 *
 * <key> 可以是以下之一:
 * [RSA bits] [e] [n as a decimal number]
 * 'ssh-dss' [base64-encoded-key]
 * 'ssh-rsa' [base64-encoded-key]
 *
 */
LIBSSH2_API int
libssh2_knownhost_readline(LIBSSH2_KNOWNHOSTS *hosts,
                           const char *line, size_t len, int type)
{
    const char *cp;
    const char *hostp;
    const char *keyp;
    size_t hostlen;
    size_t keylen;
    int rc;

    if(type != LIBSSH2_KNOWNHOST_FILE_OPENSSH)
        return _libssh2_error(hosts->session,
                              LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                              "Unsupported type of known-host information "
                              "store");

    cp = line;

    /* 跳过前导空格 */
    while(len && ((*cp == ' ') || (*cp == '\t'))) {
        cp++;
        len--;
    }

    if(!len || !*cp || (*cp == '#') || (*cp == '\n'))
        /* 注释或空行 */
        return LIBSSH2_ERROR_NONE;

    /* 主机部分从这里开始 */
    hostp = cp;

    /* 移动到分隔符之后的主机 */
    while(len && *cp && (*cp != ' ') && (*cp != '\t')) {
        cp++;
        len--;
    }

    hostlen = cp - hostp;

    /* 密钥从空格后开始 */
    while(len && *cp && ((*cp == ' ') || (*cp == '\t'))) {
        cp++;
        len--;
    }

    if(!*cp || !len) /* 非法行 */
        return _libssh2_error(hosts->session,
                              LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                              "Failed to parse known_hosts line");

    keyp = cp; /* 密钥从这里开始 */
    keylen = len;
}
    /* 检查行（键）是否以换行符结尾，如果是则删除它 */
    while(len && *cp && (*cp != '\n')) {
        cp++;
        len--;
    }

    /* 在换行符位置加上零终止符 */
    if(*cp == '\n')
        keylen--; /* 不包括在计数内 */

    /* 处理这一行主机+键的数据 */
    rc = hostline(hosts, hostp, hostlen, keyp, keylen);
    if(rc)
        return rc; /* 失败 */

    return LIBSSH2_ERROR_NONE; /* 成功 */
}

/*
 * libssh2_knownhost_readfile
 *
 * 从给定文件中读取主机和密钥对。
 *
 * 返回负值表示错误，或者成功添加的主机数量。
 *
 */

LIBSSH2_API int
libssh2_knownhost_readfile(LIBSSH2_KNOWNHOSTS *hosts,
                           const char *filename, int type)
{
    FILE *file;
    int num = 0;
    char buf[4092];

    // 如果类型不是开放SSH类型，则返回不支持该类型的已知主机信息存储
    if(type != LIBSSH2_KNOWNHOST_FILE_OPENSSH)
        return _libssh2_error(hosts->session,
                              LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                              "Unsupported type of known-host information "
                              "store");

    // 打开文件进行读取
    file = fopen(filename, FOPEN_READTEXT);
    if(file) {
        // 逐行读取文件内容
        while(fgets(buf, sizeof(buf), file)) {
            // 读取每行内容并解析
            if(libssh2_knownhost_readline(hosts, buf, strlen(buf), type)) {
                // 如果解析失败，则返回已知主机文件解析失败的错误
                num = _libssh2_error(hosts->session, LIBSSH2_ERROR_KNOWN_HOSTS,
                                     "Failed to parse known hosts file");
                break;
            }
            // 成功解析一行，增加成功添加的主机数量
            num++;
        }
        // 关闭文件
        fclose(file);
    }
    else
        // 如果打开文件失败，则返回文件打开失败的错误
        return _libssh2_error(hosts->session, LIBSSH2_ERROR_FILE,
                              "Failed to open file");

    return num;
}

/*
 * knownhost_writeline()
 *
 * 请求libssh2将已知主机转换为存储的输出行。
 *
 * 请注意，如果给定的输出缓冲区太小而无法容纳所需的输出，则此函数将返回LIBSSH2_ERROR_BUFFER_TOO_SMALL。
 * 'outlen'字段将包含libssh2想要存储的大小，这将是它需要的最小足够的缓冲区。
 *
 */
static int
knownhost_writeline(LIBSSH2_KNOWNHOSTS *hosts,
                    struct known_host *node,
                    char *buf, size_t buflen,
                    size_t *outlen, int type)
{
    size_t required_size;

    const char *key_type_name;
    size_t key_type_len;

    /* 目前我们只支持这种单一文件类型，对所有其他尝试进行退出 */
    # 如果类型不是OpenSSH格式的已知主机文件，则返回不支持该类型的错误
    if(type != LIBSSH2_KNOWNHOST_FILE_OPENSSH)
        return _libssh2_error(hosts->session,
                              LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                              "Unsupported type of known-host information "
                              "store");

    # 根据已知主机节点的密钥类型进行处理
    switch(node->typemask & LIBSSH2_KNOWNHOST_KEY_MASK) {
    case LIBSSH2_KNOWNHOST_KEY_RSA1:
        key_type_name = NULL;
        key_type_len = 0;
        break;
    case LIBSSH2_KNOWNHOST_KEY_SSHRSA:
        key_type_name = "ssh-rsa";
        key_type_len = 7;
        break;
    case LIBSSH2_KNOWNHOST_KEY_SSHDSS:
        key_type_name = "ssh-dss";
        key_type_len = 7;
        break;
    case LIBSSH2_KNOWNHOST_KEY_ECDSA_256:
        key_type_name = "ecdsa-sha2-nistp256";
        key_type_len = 19;
        break;
    case LIBSSH2_KNOWNHOST_KEY_ECDSA_384:
        key_type_name = "ecdsa-sha2-nistp384";
        key_type_len = 19;
        break;
    case LIBSSH2_KNOWNHOST_KEY_ECDSA_521:
        key_type_name = "ecdsa-sha2-nistp521";
        key_type_len = 19;
        break;
    case LIBSSH2_KNOWNHOST_KEY_ED25519:
        key_type_name = "ssh-ed25519";
        key_type_len = 11;
        break;
    case LIBSSH2_KNOWNHOST_KEY_UNKNOWN:
        key_type_name = node->key_type_name;
        # 如果已知主机节点的密钥类型为未知，则使用节点中的密钥类型名称和长度
        if(key_type_name) {
            key_type_len = node->key_type_len;
            break;
        }
        # 否则返回不支持该类型的错误
        /* otherwise fallback to default and error */
        /* FALL-THROUGH */
    default:
        return _libssh2_error(hosts->session,
                              LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                              "Unsupported type of known-host entry");
    }
    /* 当组合主机行时，有三个方面需要考虑：
       - 哈希（SHA1）或未哈希的主机名
       - 密钥名称或无密钥名称（RSA1）
       - 注释或无注释

       这意味着有 2^3 种不同的格式：
       ("|1|%s|%s %s %s %s\n", salt, hashed_host, key_name, key, comment)
       ("|1|%s|%s %s %s\n", salt, hashed_host, key_name, key)
       ("|1|%s|%s %s %s\n", salt, hashed_host, key, comment)
       ("|1|%s|%s %s\n", salt, hashed_host, key)
       ("%s %s %s %s\n", host, key_name, key, comment)
       ("%s %s %s\n", host, key_name, key)
       ("%s %s %s\n", host, key, comment)
       ("%s %s\n", host, key)

       即使缓冲区太小，我们也必须将 outlen 设置为完整行所需的字符数。除非我们确定可以将所有内容写入缓冲区，否则不要向缓冲区写入任何内容。 */

    required_size = strlen(node->key);

    if(key_type_len)
        required_size += key_type_len + 1; /* ' ' = 1 */
    if(node->comment)
        required_size += node->comment_len + 1; /* ' ' = 1 */

    }
    else {
        required_size += node->name_len + 3;
        /* ' ' + '\n' + \0 = 3 */

        if(required_size <= buflen) {
            if(node->comment && key_type_len)
                snprintf(buf, buflen, "%s %s %s %s\n", node->name,
                         key_type_name, node->key, node->comment);
            else if(node->comment)
                snprintf(buf, buflen, "%s %s %s\n", node->name, node->key,
                         node->comment);
            else if(key_type_len)
                snprintf(buf, buflen, "%s %s %s\n", node->name, key_type_name,
                         node->key);
            else
                snprintf(buf, buflen, "%s %s\n", node->name, node->key);
        }
    }

    /* 我们报告数据的完整长度，尾随的零不包括在内 */
    *outlen = required_size-1;
    # 如果所需大小小于等于当前缓冲区大小，则返回无错误
    if(required_size <= buflen)
        return LIBSSH2_ERROR_NONE;
    # 否则，返回已知主机写入缓冲区太小的错误，并将错误信息传递给会话对象
    else
        return _libssh2_error(hosts->session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                              "Known-host write buffer too small");
}

/*
 * libssh2_knownhost_writeline()
 *
 * Ask libssh2 to convert a known host to an output line for storage.
 *
 * Note that this function returns LIBSSH2_ERROR_BUFFER_TOO_SMALL if the given
 * output buffer is too small to hold the desired output.
 */
LIBSSH2_API int
libssh2_knownhost_writeline(LIBSSH2_KNOWNHOSTS *hosts,
                            struct libssh2_knownhost *known,
                            char *buffer, size_t buflen,
                            size_t *outlen, /* the amount of written data */
                            int type)
{
    struct known_host *node; // 定义一个指向已知主机的结构体指针

    if(known->magic != KNOWNHOST_MAGIC) // 如果已知主机的魔术数不等于已知主机的魔术数
        return _libssh2_error(hosts->session, LIBSSH2_ERROR_INVAL,
                              "Invalid host information"); // 返回无效主机信息错误

    node = known->node; // 将已知主机的节点赋值给node

    return knownhost_writeline(hosts, node, buffer, buflen, outlen, type); // 调用knownhost_writeline函数
}

/*
 * libssh2_knownhost_writefile()
 *
 * Write hosts+key pairs to the given file.
 */
LIBSSH2_API int
libssh2_knownhost_writefile(LIBSSH2_KNOWNHOSTS *hosts,
                            const char *filename, int type)
{
    struct known_host *node; // 定义一个指向已知主机的结构体指针
    FILE *file; // 定义一个文件指针
    int rc = LIBSSH2_ERROR_NONE; // 定义一个错误码变量并初始化为无错误
    char buffer[4092]; // 定义一个大小为4092的字符数组

    /* we only support this single file type for now, bail out on all other
       attempts */
    if(type != LIBSSH2_KNOWNHOST_FILE_OPENSSH) // 如果类型不等于LIBSSH2_KNOWNHOST_FILE_OPENSSH
        return _libssh2_error(hosts->session,
                              LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                              "Unsupported type of known-host information "
                              "store"); // 返回不支持的已知主机信息存储类型错误

    file = fopen(filename, FOPEN_WRITETEXT); // 打开文件
    if(!file) // 如果文件指针为空
        return _libssh2_error(hosts->session, LIBSSH2_ERROR_FILE,
                              "Failed to open file"); // 返回打开文件失败错误
    # 遍历 hosts 链表中的节点
    for(node = _libssh2_list_first(&hosts->head);
        node;
        node = _libssh2_list_next(&node->node)) {
        # 初始化已写入的字节数为 0
        size_t wrote = 0;
        size_t nwrote;
        # 调用 knownhost_writeline 函数将节点信息写入 buffer
        rc = knownhost_writeline(hosts, node, buffer, sizeof(buffer), &wrote,
                                 type);
        # 如果写入失败，则跳出循环
        if(rc)
            break;

        # 将 buffer 中的内容写入文件，并记录实际写入的字节数
        nwrote = fwrite(buffer, 1, wrote, file);
        # 如果实际写入的字节数与预期不符，则输出错误信息并跳出循环
        if(nwrote != wrote) {
            /* failed to write the whole thing, bail out */
            rc = _libssh2_error(hosts->session, LIBSSH2_ERROR_FILE,
                                "Write failed");
            break;
        }
    }
    # 关闭文件
    fclose(file);

    # 返回操作结果
    return rc;
/*
 * libssh2_knownhost_get()
 *
 * 遍历已知主机的内部列表。将 'prev' 设为 NULL 以获取第一个已知主机。
 *
 * 返回：
 * 0 - 如果在 'store' 中存储了一个良好的主机
 * 1 - 如果已到达主机列表末尾
 * [负数] - 表示错误
 */
LIBSSH2_API int
libssh2_knownhost_get(LIBSSH2_KNOWNHOSTS *hosts,
                      struct libssh2_knownhost **ext,
                      struct libssh2_knownhost *oprev)
{
    struct known_host *node;
    if(oprev && oprev->node) {
        /* 如果有起始点 */
        struct known_host *prev = oprev->node;

        /* 获取列表中的下一个节点 */
        node = _libssh2_list_next(&prev->node);

    }
    else
        node = _libssh2_list_first(&hosts->head);

    if(!node)
        /* 没有（更多）节点 */
        return 1;

    *ext = knownhost_to_external(node);

    return 0;
}
```