# Nmap源码解析 98

# `libssh2/src/knownhost.c`

这段代码是一个 C 语言的函数，它定义了一个名为 `output` 的函数。然而，该函数在函数声明之前被意外地调用了。

输出函数的实现如下：
```cppc
/*
* Purpose: 
* This function is called when the user specifies
* the file name and/or the number of arguments for
* the `sprintf()` function. It prints the specified
* arguments to the console.
*
* Implementation details:
*
* The function prints the specified arguments
* to the console using `printf()` function.
* If the user specifies a file name, the function prints
* the arguments to the console in the format specified by
* the `printf()` function.
*
* Parameters:
*
* double argc - The number of arguments passed to
* the `sprintf()` function.
*
* double argv[3] - The arguments to print to the
* console.
*
* Return value:
*
* 0 - Successful call to `sprintf()`.
*
* Description:
*
* This function is useful for debugging
* applications that use `sprintf()` to print
* formatted strings to the console.
*/
int output(int argc, double argv[3]);
```
该函数的作用是在 `sprintf()` 函数成功执行时打印给定的 arguments。它通过 `argc` 变量获取 `sprintf()` 函数传递给它的参数数量，并通过 `argv` 数组获取这些参数。函数内部使用 `printf()` 函数打印给定的 arguments 到控制台。如果用户指定了一个文件名，该函数按照指定的格式打印参数。


```cpp
/*
 * Copyright (c) 2009-2019 by Daniel Stenberg
 * All rights reserved.
 *
 * Redistribution and use in source and binary forms,
 * with or without modification, are permitted provided
 * that the following conditions are met:
 *
 *   Redistributions of source code must retain the above
 *   copyright notice, this list of conditions and the
 *   following disclaimer.
 *
 *   Redistributions in binary form must reproduce the above
 *   copyright notice, this list of conditions and the following
 *   disclaimer in the documentation and/or other materials
 *   provided with the distribution.
 *
 *   Neither the name of the copyright holder nor the names
 *   of any other contributors may be used to endorse or
 *   promote products derived from this software without
 *   specific prior written permission.
 *
 * THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND
 * CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES,
 * INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES
 * OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
 * ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR
 * CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
 * SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING,
 * BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
 * SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
 * INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY,
 * WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
 * NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE
 * USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY
 * OF SUCH DAMAGE.
 */

```

这段代码定义了一个名为`known_host`的结构体，用于存储SSH客户端与服务器之间通过libssh2库建立连接时所需了解的客户端主机信息。这个结构体包含了一系列用于标识主机并建立起SSH连接的属性，如主机名称、端口号、类型、盐等。通过在`known_host`结构体中增加一些成员变量，可以方便地使用libssh2库来查找和连接到这些主机。

这个结构体在`libssh2_priv.h`和`misc.h`中被包含，前者是库头文件，后者是一个通用的头文件。在`libssh2_knownhost.h`中，这个结构体的一些成员变量被定义。

这个`known_host`结构体是`libssh2_knownhost`类型的实例化表示，可以在`SSHClient`和`SSHServer`类的实现中中被用来存储客户端和服务器之间的主机信息。


```cpp
#include "libssh2_priv.h"
#include "misc.h"

struct known_host {
    struct list_node node;
    char *name;      /* points to the name or the hash (allocated) */
    size_t name_len; /* needed for hashed data */
    int port;        /* if non-zero, a specific port this key is for on this
                        host */
    int typemask;    /* plain, sha1, custom, ... */
    char *salt;      /* points to binary salt (allocated) */
    size_t salt_len; /* size of salt */
    char *key;       /* the (allocated) associated key. This is kept base64
                        encoded in memory. */
    char *key_type_name; /* the (allocated) key type name */
    size_t key_type_len; /* size of key_type_name */
    char *comment;       /* the (allocated) optional comment text, may be
                            NULL */
    size_t comment_len;  /* the size of comment */

    /* this is the struct we expose externally */
    struct libssh2_knownhost external;
};

```

这段代码定义了一个名为`_LIBSSH2_KNOWNHOSTS`的结构体，它用于存储`known_hosts`数组中的单个元素。

`session`成员表示`known_hosts`数组所属的SSH会话，它用于跟踪当前连接的已知主机。

`head`成员是一个指向`known_hosts`数组开头的指针。

函数`free_host`用于释放包含在`known_hosts`数组中的单个元素（如果有）。函数接收两个参数：`session`和`entry`。函数首先检查`entry`是否为空。如果是，函数会递归地执行以下操作：

1. 如果`entry`包含一个注释，函数会将其free。
2. 如果`entry`包含一个键类型名称，函数会将其free。
3. 如果`entry`包含一个键，函数会将其free。
4. 如果`entry`包含一个 salt，函数会将其free。
5. 如果`entry`包含一个主机名，函数会将其free。
6. 最后，函数会返回`entry`，以便它继续存在于`known_hosts`数组中。


```cpp
struct _LIBSSH2_KNOWNHOSTS
{
    LIBSSH2_SESSION *session;  /* the session this "belongs to" */
    struct list_head head;
};

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

```

这段代码的作用是定义了一个名为`libssh2_knownhost_init`的函数，用于初始化已知的主机。它接受一个名为`session`的`LIBSSH2_SESSION`类型的参数，作为已知主机列表的存储容器。

函数首先在内存中分配一块名为`knh`的已知主机列表结构体变量。如果内存分配失败，它将返回一个错误代码。否则，它将把`session`作为`knh`的目标对象，并将其作为构造函数的第一个参数传递给`_libssh2_list_init`函数，该函数用于初始化列表。

最后，函数返回`knh`，表示已知主机列表已成功初始化。


```cpp
/*
 * libssh2_knownhost_init
 *
 * Init a collection of known hosts. Returns the pointer to a collection.
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

```

这段代码定义了一个常量 KNOWNHOST_MAGIC，其值为 0xdeadcafe。

定义了一个名为 knownhost_to_external 的函数，该函数接收一个 known_host 类型的结构体节点作为参数，并返回一个指向 libssh2_knownhost 类型的结构体。

函数内部首先定义了一个名为 ext 的内部 libssh2_knownhost 结构体变量，然后将其与输入的 known_host 结构体节点中的 external 成员关联。

接着，通过一系列 assignment 操作，将KNOWNHOST_MAGIC 常量值、函数调用者的 node 结构体中的 magic 成员、输入节点中的 internal 成员、输入节点中的 name 成员、输入节点的 typemask 成员设置为 0xdeadcafe，从而将输入节点 internal 的结构体拷贝到输出结构体 ext 的 external 成员中。

最后，通过返回指向 ext 的指针，将 knownhost_to_external 函数返回，以便在需要时调用。


```cpp
#define KNOWNHOST_MAGIC 0xdeadcafe
/*
 * knownhost_to_external()
 *
 * Copies data from the internal to the external representation struct.
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

```

This function appears to be part of the SSH client library, and it is responsible for adding a new "host" to the known hosts list. The function takes in information about the host, such as its hostname, IP address, key type, and key. It also takes in a comment, if provided.

The function first checks if the key type is already known on the host, and if not, it creates a new key type and stores it in the host->session variable.

Then, it checks if the host has any comments, and if it does, it creates a new comment and stores it in the host->session variable.

Finally, the function adds the host to the known hosts list and stores the internal representation of the host in the store variable. If an error occurs, the function returns and frees the host in the hosts->session variable.


```cpp
static int
knownhost_add(LIBSSH2_KNOWNHOSTS *hosts,
              const char *host, const char *salt,
              const char *key_type_name, size_t key_type_len,
              const char *key, size_t keylen,
              const char *comment, size_t commentlen,
              int typemask, struct libssh2_knownhost **store)
{
    struct known_host *entry;
    size_t hostlen = strlen(host);
    int rc;
    char *ptr;
    unsigned int ptrlen;

    /* make sure we have a key type set */
    if(!(typemask & LIBSSH2_KNOWNHOST_KEY_MASK))
        return _libssh2_error(hosts->session, LIBSSH2_ERROR_INVAL,
                              "No key type set");

    entry = LIBSSH2_CALLOC(hosts->session, sizeof(struct known_host));
    if(!entry)
        return _libssh2_error(hosts->session, LIBSSH2_ERROR_ALLOC,
                              "Unable to allocate memory for known host "
                              "entry");

    entry->typemask = typemask;

    switch(entry->typemask  & LIBSSH2_KNOWNHOST_TYPE_MASK) {
    case LIBSSH2_KNOWNHOST_TYPE_PLAIN:
    case LIBSSH2_KNOWNHOST_TYPE_CUSTOM:
        entry->name = LIBSSH2_ALLOC(hosts->session, hostlen + 1);
        if(!entry->name) {
            rc = _libssh2_error(hosts->session, LIBSSH2_ERROR_ALLOC,
                                "Unable to allocate memory for host name");
            goto error;
        }
        memcpy(entry->name, host, hostlen + 1);
        entry->name_len = hostlen;
        break;
    case LIBSSH2_KNOWNHOST_TYPE_SHA1:
        rc = libssh2_base64_decode(hosts->session, &ptr, &ptrlen,
                                   host, hostlen);
        if(rc)
            goto error;
        entry->name = ptr;
        entry->name_len = ptrlen;

        rc = libssh2_base64_decode(hosts->session, &ptr, &ptrlen,
                                   salt, strlen(salt));
        if(rc)
            goto error;
        entry->salt = ptr;
        entry->salt_len = ptrlen;
        break;
    default:
        rc = _libssh2_error(hosts->session, LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                            "Unknown host name type");
        goto error;
    }

    if(typemask & LIBSSH2_KNOWNHOST_KEYENC_BASE64) {
        /* the provided key is base64 encoded already */
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

    if(comment) {
        entry->comment = LIBSSH2_ALLOC(hosts->session, commentlen + 1);
        if(!entry->comment) {
            rc = _libssh2_error(hosts->session, LIBSSH2_ERROR_ALLOC,
                                "Unable to allocate memory for comment");
            goto error;
        }
        memcpy(entry->comment, comment, commentlen + 1);
        entry->comment[commentlen] = 0; /* force a terminating zero trailer */
        entry->comment_len = commentlen;
    }
    else {
        entry->comment = NULL;
    }

    /* add this new host to the big list of known hosts */
    _libssh2_list_add(&hosts->head, &entry->node);

    if(store)
        *store = knownhost_to_external(entry);

    return LIBSSH2_ERROR_NONE;
  error:
    free_host(hosts->session, entry);
    return rc;
}

```

这段代码定义了一个名为 libssh2_knownhost_add 的函数，用于将指定的主机及其关联的密钥添加到已知的主机集合中。

具体来说，这段代码接受一个名为 type 的参数，它指定了要使用哪种格式来输入主机和密钥。如果 type 是 "plain"，则使用 "hostname.domain.tld" 的 ASCII 格式；如果 type 是 "sha1"，则需要提供 salt，并且不能使用 base64 编码的 salt；如果 type 是 "custom"，则可以自定义哈希算法。

在函数内部，首先检查传入的类型是否选择了 "sha1"，如果是，就需要提供 salt，并且不能使用 base64 编码的 salt。如果选择了 "custom"，则使用自定义的哈希算法，并将 salt 设置为零（即可以忽略 salt）。

然后，将主机和密钥添加到已知的主机集合中。最后，函数返回这两个参数的交集（即仅可能是第一个参数，如果不是空的话）。


```cpp
/*
 * libssh2_knownhost_add
 *
 * Add a host and its associated key to the collection of known hosts.
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

```

这段代码定义了一个名为 libssh2_knownhost_add 的函数，它是 LIBSSH2 API 的一部分。它的作用是添加一个已知的 host 和相关的 key 到已知的 host 集合中。

函数接受三个参数：一个整数类型的 libssh2_knownhosts 指针，一个字符串类型的 host 参数，一个字符串类型的 salt 参数，和一个字符串类型的 key 参数。它还接受一个整数类型的 typemask 参数，用于指示如何使用添加的 key。最后，它返回一个指向 libssh2_knownhost 结构的指针，用于将添加的信息存储到已知的 host 集合中。

函数内部包含一个名为 knownhost_add 的函数，这个函数将添加已知的 host 和相关的 key 到已知的 host 集合中。如果已知主机添加类型为 SHA1，那么就需要提供 salt 参数。函数还支持自定义类型，如果使用自定义类型，就需要在 salt 参数中提供主机预先哈希的值。如果使用的已知主机类型为 SHA1，那么函数就需要提供一个字符串类型的 key 参数，用于在存储之前对其进行预先哈希。


```cpp
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

```

这段代码是一个名为`libssh2_knownhost_addc`的函数，属于名为`libssh2_knownhost`的库。它实现了将已知主机添加到已知的库中的功能。

具体来说，这段代码接受四个参数：

1. `hosts`：一个`LIBSSH2_KNOWNHOSTS`类型的数组，用于存储已知主机。
2. `host`：一个字符串，包含主机名称。
3. `salt`：一个字符串，包含主机salt（用于标识主机）。
4. `key`：一个字符串，包含主机密钥。
5. `keylen`：一个整数，密钥的长度。
6. `comment`：一个字符串，包含主机描述。
7. `commentlen`：一个整数，描述的长度。
8. `typemask`：一个整数，用于指定输入和输出数据类型。
9. `store`：一个`struct libssh2_knownhost2`类型的指针，用于将添加的主机添加到已知的库中。

函数首先调用`knownhost_add`函数，将输入的主机添加到已知的库中。如果已知主机添加失败，函数将返回`LIBSSH2_KNOWNHOST_CHECK_FAILURE`。如果输入的主机不在已知的库中，函数将返回`LIBSSH2_KNOWNHOST_CHECK_NOTFOUND`。如果输入的主机与已知的库中的主机匹配，或者输入的密钥与已知的库中的密钥匹配，函数将返回`LIBSSH2_KNOWNHOST_CHECK_MATCH`。如果输入的主机与已知的库中的主机不匹配，或者输入的密钥与已知的库中的密钥不匹配，函数将返回`LIBSSH2_KNOWNHOST_CHECK_MISMATCH`。

函数的实现主要依赖于已知的库函数`knownhost_add`和`knownhost_remove`，以及一些显式地转换为字符串类型的函数，如`strdup`和`strcpy`。


```cpp
LIBSSH2_API int
libssh2_knownhost_addc(LIBSSH2_KNOWNHOSTS *hosts,
                       const char *host, const char *salt,
                       const char *key, size_t keylen,
                       const char *comment, size_t commentlen,
                       int typemask, struct libssh2_knownhost **store)
{
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
```

这段代码的作用是判断输入的文件是否和已知的主机密钥匹配。首先，它读取文件内容并获取文件名和文件类型。然后，它创建一个HSF object，并使用文件名和文件类型来搜索已知主机密钥。如果找到了匹配的密钥，它会对密钥进行类型检查，并在类型检查通过后，检查文件内容是否和密钥匹配。如果匹配成功，它将创建一个匹配的HSF object，并从文件中删除已知主机密钥。如果匹配失败，它将返回一个错误码。


```cpp
static int
knownhost_check(LIBSSH2_KNOWNHOSTS *hosts,
                const char *hostp, int port,
                const char *key, size_t keylen,
                int typemask,
                struct libssh2_knownhost **ext)
{
    struct known_host *node;
    struct known_host *badkey = NULL;
    int type = typemask & LIBSSH2_KNOWNHOST_TYPE_MASK;
    char *keyalloc = NULL;
    int rc = LIBSSH2_KNOWNHOST_CHECK_NOTFOUND;
    char hostbuff[270]; /* most host names can't be longer than like 256 */
    const char *host;
    int numcheck; /* number of host combos to check */
    int match = 0;

    if(type == LIBSSH2_KNOWNHOST_TYPE_SHA1)
        /* we can't work with a sha1 as given input */
        return LIBSSH2_KNOWNHOST_CHECK_MISMATCH;

    /* if a port number is given, check for a '[host]:port' first before the
       plain 'host' */
    if(port >= 0) {
        int len = snprintf(hostbuff, sizeof(hostbuff), "[%s]:%d", hostp, port);
        if(len < 0 || len >= (int)sizeof(hostbuff)) {
            _libssh2_error(hosts->session,
                           LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                           "Known-host write buffer too small");
            return LIBSSH2_KNOWNHOST_CHECK_FAILURE;
        }
        host = hostbuff;
        numcheck = 2; /* check both combos, start with this */
    }
    else {
        host = hostp;
        numcheck = 1; /* only check this host version */
    }

    if(!(typemask & LIBSSH2_KNOWNHOST_KEYENC_BASE64)) {
        /* we got a raw key input, convert it to base64 for the checks below */
        size_t nlen = _libssh2_base64_encode(hosts->session, key, keylen,
                                             &keyalloc);
        if(!nlen) {
            _libssh2_error(hosts->session, LIBSSH2_ERROR_ALLOC,
                           "Unable to allocate memory for base64-encoded "
                           "key");
            return LIBSSH2_KNOWNHOST_CHECK_FAILURE;
        }

        /* make the key point to this */
        key = keyalloc;
    }

    do {
        node = _libssh2_list_first(&hosts->head);
        while(node) {
            switch(node->typemask & LIBSSH2_KNOWNHOST_TYPE_MASK) {
            case LIBSSH2_KNOWNHOST_TYPE_PLAIN:
                if(type == LIBSSH2_KNOWNHOST_TYPE_PLAIN)
                    match = !strcmp(host, node->name);
                break;
            case LIBSSH2_KNOWNHOST_TYPE_CUSTOM:
                if(type == LIBSSH2_KNOWNHOST_TYPE_CUSTOM)
                    match = !strcmp(host, node->name);
                break;
            case LIBSSH2_KNOWNHOST_TYPE_SHA1:
                if(type == LIBSSH2_KNOWNHOST_TYPE_PLAIN) {
                    /* when we have the sha1 version stored, we can use a
                       plain input to produce a hash to compare with the
                       stored hash.
                    */
                    unsigned char hash[SHA_DIGEST_LENGTH];
                    libssh2_hmac_ctx ctx;
                    libssh2_hmac_ctx_init(ctx);

                    if(SHA_DIGEST_LENGTH != node->name_len) {
                        /* the name hash length must be the sha1 size or
                           we can't match it */
                        break;
                    }
                    libssh2_hmac_sha1_init(&ctx, (unsigned char *)node->salt,
                                           node->salt_len);
                    libssh2_hmac_update(ctx, (unsigned char *)host,
                                        strlen(host));
                    libssh2_hmac_final(ctx, hash);
                    libssh2_hmac_cleanup(&ctx);

                    if(!memcmp(hash, node->name, SHA_DIGEST_LENGTH))
                        /* this is a node we're interested in */
                        match = 1;
                }
                break;
            default: /* unsupported type */
                break;
            }
            if(match) {
                int host_key_type = typemask & LIBSSH2_KNOWNHOST_KEY_MASK;
                int known_key_type =
                    node->typemask & LIBSSH2_KNOWNHOST_KEY_MASK;
                /* match on key type as follows:
                   - never match on an unknown key type
                   - if key_type is set to zero, ignore it an match always
                   - otherwise match when both key types are equal
                */
                if(host_key_type != LIBSSH2_KNOWNHOST_KEY_UNKNOWN &&
                     (host_key_type == 0 ||
                      host_key_type == known_key_type)) {
                    /* host name and key type match, now compare the keys */
                    if(!strcmp(key, node->key)) {
                        /* they match! */
                        if(ext)
                            *ext = knownhost_to_external(node);
                        badkey = NULL;
                        rc = LIBSSH2_KNOWNHOST_CHECK_MATCH;
                        break;
                    }
                    else {
                        /* remember the first node that had a host match but a
                           failed key match since we continue our search from
                           here */
                        if(!badkey)
                            badkey = node;
                    }
                }
                match = 0; /* don't count this as a match anymore */
            }
            node = _libssh2_list_next(&node->node);
        }
        host = hostp;
    } while(!match && --numcheck);

    if(badkey) {
        /* key mismatch */
        if(ext)
            *ext = knownhost_to_external(badkey);
        rc = LIBSSH2_KNOWNHOST_CHECK_MISMATCH;
    }

    if(keyalloc)
        LIBSSH2_FREE(hosts->session, keyalloc);

    return rc;
}

```

这段代码是一个名为 `libssh2_knownhost_check` 的函数，它的作用是检查一个主机及其关联的密钥是否与预定义的主机密钥集合中的主机和密钥匹配。

该函数接受两个参数：一个字符串类型的主机名和一个字符串类型的密钥。函数使用 `libssh2_strings_parse` 函数将主机名解析为字符串，然后使用哈希函数（如 `hcrypt`）将密钥哈希为字符串。

函数的返回值包括：

* LIBSSH2_KNOWNHOST_CHECK_FAILURE：表示输入的主机名和密钥不匹配任何一个预定义的主机密钥。
* LIBSSH2_KNOWNHOST_CHECK_NOTFOUND：表示在预定义主机密钥集合中找不到与输入的主机名和密钥匹配的主机。
* LIBSSH2_KNOWNHOST_CHECK_MATCH：表示输入的主机名和密钥与预定义的主机密钥集合中的主机完全匹配。
* LIBSSH2_KNOWNHOST_CHECK_MISMATCH：表示输入的主机名和密钥与预定义的主机密钥集合中的主机不完全匹配，但哈希函数没有问题。

需要注意的是，这段代码不支持使用任何盐（salt）来保护哈希函数，因此它不能保证密钥的完整性和安全性。


```cpp
/*
 * libssh2_knownhost_check
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
```

这段代码是一个名为`libssh2_knownhost_check`的函数，属于`libssh2_api`库。它的作用是检查给定的`hosts`数组中，是否存在与给定`hostp`和`key`拼接的`host`和`port`，并返回相应的结果。

函数首先通过`knownhost_check`函数检查给定的`hosts`数组，如果找到了匹配的`host`和`key`，则返回`LIBSSH2_KNOWNHOST_CHECK_MATCH`。如果`port`参数为正，则函数将检查是否存在针对该主机和端口的`key`，如果存在，则返回`LIBSSH2_KNOWNHOST_CHECK_MATCH`。如果`port`参数为负，则函数将仅检查给定`host`的`key`。

另外，如果给定的`key`格式不正确（如`plain`或`custom`），函数将返回`LIBSSH2_KNOWNHOST_CHECK_FAILURE`。


```cpp
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
 * Check a host+port and its associated key against the collection of known
 * hosts.
 *
 * Note that if 'port' is specified as greater than zero, the check function
 * will be able to check for a dedicated key for this particular host+port
 * combo, and if 'port' is negative it only checks for the generic host key.
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
```

这段代码是一个名为libssh2_knownhost_checkp的函数，属于libssh2库。它接受一个名为hosts的已知主机结构体指针，一个要检查的主机名称hostp，一个目标端口号port，一个密钥key，以及一个typemask，表示要检查的关键字掩码。该函数返回一个指向已知主机的指针，如果匹配则返回，否则返回NULL。

libssh2_knownhost_del是一个名为libssh2_knownhost_del的函数，属于libssh2库。它接受一个名为ext的已知主机结构体指针，用于存储已知主机信息，一个要删除的已知主机名称hostp，用于指定要删除的已知主机。该函数用于从已知主机列表中删除指定的主机，并将结果存储在ext指向的指针中。


```cpp
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


/*
 * libssh2_knownhost_del
 *
 * Remove a host from the collection of known hosts.
 *
 */
```

这段代码是一个名为`libssh2_knownhost_del`的函数，属于名为`libssh2_api`的函数。它的作用是移除已知主机列表中的一个已知主机，并将其从已知的所有主机列表中移除。以下是具体的实现步骤：

1. 首先检查传入的`hosts`参数是否为空，以及`entry`参数是否符合`libssh2_knownhost`结构体的定义。如果是，函数返回一个错误码，并输出错误信息。
2. 如果`hosts`和`entry`参数都正确，函数开始执行实际的移除操作。
3. 首先获取一个指向已知主机的内部节点指针`node`。
4. 然后使用`_libssh2_list_remove`函数从所有已知主机列表中移除节点`node`。
5. 接着，清除`entry`参数，以便将已知主机列表中的所有信息都丢失。
6. 最后，释放所有资源，并返回0，表示成功移除已知主机。


```cpp
LIBSSH2_API int
libssh2_knownhost_del(LIBSSH2_KNOWNHOSTS *hosts,
                      struct libssh2_knownhost *entry)
{
    struct known_host *node;

    /* check that this was retrieved the right way or get out */
    if(!entry || (entry->magic != KNOWNHOST_MAGIC))
        return _libssh2_error(hosts->session, LIBSSH2_ERROR_INVAL,
                              "Invalid host information");

    /* get the internal node pointer */
    node = entry->node;

    /* unlink from the list of all hosts */
    _libssh2_list_remove(&node->node);

    /* clear the struct now since the memory in which it is allocated is
       about to be freed! */
    memset(entry, 0, sizeof(struct libssh2_knownhost));

    /* free all resources */
    free_host(hosts->session, node);

    return 0;
}

```

这段代码定义了一个名为`libssh2_knownhost_free`的函数，其作用是释放一个已知主机的集合。这个函数接受一个已知主机结构体数组`hosts`作为参数。

函数内部先通过一个循环遍历已知主机的每一个节点，然后执行每个节点下面的free_host函数，这个函数会根据需要释放已知主机。最后，函数再释放整个已知主机数组。

通过这个函数，你可以释放已知主机数组`hosts`中的所有元素，释放后，这个函数就不会再对已知主机进行修改了。


```cpp
/*
 * libssh2_knownhost_free
 *
 * Free an entire collection of known hosts.
 *
 */
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


```



This function appears to be part of the knownhosts utility for the OpenSSH protocol, which allows you to add known hosts to a client session. It takes a single parameter, `keylen`, which is the length of the key used to authenticate with the server, and a variable number of additional parameters representing the known host database.

The function first checks if the known host database is loaded, and if not, it returns an error. If the database is loaded, it then parses the known host file into a format that can be used by the `knownhost_add` function. This involves parsing the file line by line, adding the host to the database, and returning the success or failure of the operation.

If the function encounters a comma, it skips to the next line in the file. If the function encounters the end of the file or a bad format, it returns an error.

The function also takes two optional parameters, `key_type` and `comment`, which specify the type of key and the comment associated with the host, respectively.


```cpp
/* old style plain text: [name]([,][name])*

   for the sake of simplicity, we add them as separate hosts with the same
   key
*/
static int oldstyle_hostline(LIBSSH2_KNOWNHOSTS *hosts,
                             const char *host, size_t hostlen,
                             const char *key_type_name, size_t key_type_len,
                             const char *key, size_t keylen, int key_type,
                             const char *comment, size_t commentlen)
{
    int rc = 0;
    size_t namelen = 0;
    const char *name = host + hostlen;

    if(hostlen < 1)
        return _libssh2_error(hosts->session,
                              LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                              "Failed to parse known_hosts line "
                              "(no host names)");

    while(name > host) {
        --name;
        ++namelen;

        /* when we get the the start or see a comma coming up, add the host
           name to the collection */
        if((name == host) || (*(name-1) == ',')) {

            char hostbuf[256];

            /* make sure we don't overflow the buffer */
            if(namelen >= sizeof(hostbuf)-1)
                return _libssh2_error(hosts->session,
                                      LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                                      "Failed to parse known_hosts line "
                                      "(unexpected length)");

            /* copy host name to the temp buffer and zero terminate */
            memcpy(hostbuf, name, namelen);
            hostbuf[namelen] = 0;

            rc = knownhost_add(hosts, hostbuf, NULL,
                               key_type_name, key_type_len,
                               key, keylen,
                               comment, commentlen,
                               key_type | LIBSSH2_KNOWNHOST_TYPE_PLAIN |
                               LIBSSH2_KNOWNHOST_KEYENC_BASE64, NULL);
            if(rc)
                return rc;

            if(name > host) {
                namelen = 0;
                --name; /* skip comma */
            }
        }
    }

    return rc;
}

```

This is a function that appears to parse a "known\_hosts" line from a file and adds a user to the known hosts on the local machine. It takes as input a "known\_hosts" line in the format:
```cppmakefile
<hostname> <commit> < sshuser > <sshpassphrase > <许可证> <泡牌> <不信任的 CA > <quiet> <yes > <使这个主机为已知 > <算法> <散列算法>
```
It returns an error code and a pointer to the added user.

The function first checks that the salt at the beginning of the "known\_hosts" line is a valid base64-encoded salt value. Then, it loops through the rest of the "known\_hosts" line until it finds the end of the salt.

If it finds the end of the salt, it checks that the salt length is valid and that it ends with a separator character. If the salt length is valid and it ends with a separator character, it makes the host point to the hash by updating the `host` and `hostlen` variables.

Finally, it checks that the `hostlen` seems to be a reasonable length and compares it to the expected length. If the `hostlen` is valid and it seems to be a reasonable length, it adds the user to the known hosts and returns 0. If any of the conditions fail, it returns an error code.


```cpp
/* |1|[salt]|[hash] */
static int hashed_hostline(LIBSSH2_KNOWNHOSTS *hosts,
                           const char *host, size_t hostlen,
                           const char *key_type_name, size_t key_type_len,
                           const char *key, size_t keylen, int key_type,
                           const char *comment, size_t commentlen)
{
    const char *p;
    char saltbuf[32];
    char hostbuf[256];

    const char *salt = &host[3]; /* skip the magic marker */
    hostlen -= 3;    /* deduct the marker */

    /* this is where the salt starts, find the end of it */
    for(p = salt; *p && (*p != '|'); p++)
        ;

    if(*p == '|') {
        const char *hash = NULL;
        size_t saltlen = p - salt;
        if(saltlen >= (sizeof(saltbuf)-1)) /* weird length */
            return _libssh2_error(hosts->session,
                                  LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                                  "Failed to parse known_hosts line "
                                  "(unexpectedly long salt)");

        memcpy(saltbuf, salt, saltlen);
        saltbuf[saltlen] = 0; /* zero terminate */
        salt = saltbuf; /* point to the stack based buffer */

        hash = p + 1; /* the host hash is after the separator */

        /* now make the host point to the hash */
        host = hash;
        hostlen -= saltlen + 1; /* deduct the salt and separator */

        /* check that the lengths seem sensible */
        if(hostlen >= sizeof(hostbuf)-1)
            return _libssh2_error(hosts->session,
                                  LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                                  "Failed to parse known_hosts line "
                                  "(unexpected length)");

        memcpy(hostbuf, host, hostlen);
        hostbuf[hostlen] = 0;

        return knownhost_add(hosts, hostbuf, salt,
                             key_type_name, key_type_len,
                             key, keylen,
                             comment, commentlen,
                             key_type | LIBSSH2_KNOWNHOST_TYPE_SHA1 |
                             LIBSSH2_KNOWNHOST_KEYENC_BASE64, NULL);
    }
    else
        return 0; /* XXX: This should be an error, shouldn't it? */
}

```

以上代码的主要功能是实现了 SSH 协议中的两种认证方式：ED25519 和 UNKnownHost。ED25519 是一种基于 RSA 证书的认证方式，而 UNKnownHost 则是一种基于受理客户端发送的明文消息的认证方式。在使用这两种认证方式时，用户需要输入不同的密钥和不同的认证信息。ED25519 密钥类型用于客户端登录，UNKnownHost 密钥类型用于服务器之间的登录。




```cpp
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
    const char *comment = NULL;
    const char *key_type_name = NULL;
    size_t commentlen = 0;
    size_t key_type_len = 0;
    int key_type;

    /* make some checks that the lengths seem sensible */
    if(keylen < 20)
        return _libssh2_error(hosts->session,
                              LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                              "Failed to parse known_hosts line "
                              "(key too short)");

    switch(key[0]) {
    case '0': case '1': case '2': case '3': case '4':
    case '5': case '6': case '7': case '8': case '9':
        key_type = LIBSSH2_KNOWNHOST_KEY_RSA1;

        /* Note that the old-style keys (RSA1) aren't truly base64, but we
         * claim it is for now since we can get away with strcmp()ing the
         * entire anything anyway! We need to check and fix these to make them
         * work properly.
         */
        break;

    default:
        key_type_name = key;
        while(keylen && *key &&
               (*key != ' ') && (*key != '\t')) {
            key++;
            keylen--;
        }
        key_type_len = key - key_type_name;

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
        while((*key ==' ') || (*key == '\t')) {
            key++;
            keylen--;
        }

        comment = key;
        commentlen = keylen;

        /* move over key */
        while(commentlen && *comment &&
              (*comment != ' ') && (*comment != '\t')) {
            comment++;
            commentlen--;
        }

        /* reduce key by comment length */
        keylen -= commentlen;

        /* Distinguish empty comment (a space) from no comment (no space) */
        if(commentlen == 0)
            comment = NULL;

        /* skip whitespaces */
        while(commentlen && *comment &&
              ((*comment ==' ') || (*comment == '\t'))) {
            comment++;
            commentlen--;
        }
        break;
    }

    /* Figure out host format */
    if((hostlen >2) && memcmp(host, "|1|", 3)) {
        /* old style plain text: [name]([,][name])*

           for the sake of simplicity, we add them as separate hosts with the
           same key
        */
        return oldstyle_hostline(hosts, host, hostlen, key_type_name,
                                 key_type_len, key, keylen, key_type,
                                 comment, commentlen);
    }
    else {
        /* |1|[salt]|[hash] */
        return hashed_hostline(hosts, host, hostlen, key_type_name,
                               key_type_len, key, keylen, key_type,
                               comment, commentlen);
    }
}

```

这段代码是一个名为 `libssh2_knownhost_readline()` 的函数，它接受一个文件名作为参数，并从该文件中读取一行或多行数据。它通过 `libssh2_knownhost_file_openssh` 函数打开文件并读取其内容，这个函数将文件内容中的每一行都当作一个 `SSHLine` 对象处理。

`SSHLine` 是一个 `SSH` 类的数据结构，它包含了在 `libssh2_ssh` 库中定义的所有 `SSHLine` 成员。这个函数的作用就是将文件中的每一行 SSH 格式化并存储到一个 `SSHLine` 对象中。

函数的具体实现可能还需要结合具体的使用场景来解释，比如将文件中的数据解析为具体的命令行参数，或者将 SSH 数据打包成 `SSHPolicy` 并传递给 `SSHClient` 类等。


```cpp
/*
 * libssh2_knownhost_readline()
 *
 * Pass in a line of a file of 'type'.
 *
 * LIBSSH2_KNOWNHOST_FILE_OPENSSH is the only supported type.
 *
 * OpenSSH line format:
 *
 * <host> <key>
 *
 * Where the two parts can be created like:
 *
 * <host> can be either
 * <name> or <hash>
 *
 * <name> consists of
 * [name] optionally followed by [,name] one or more times
 *
 * <hash> consists of
 * |1|<salt>|hash
 *
 * <key> can be one of:
 * [RSA bits] [e] [n as a decimal number]
 * 'ssh-dss' [base64-encoded-key]
 * 'ssh-rsa' [base64-encoded-key]
 *
 */
```

The 'known\_hosts' configuration option is used to store the host names and IP addresses of trusted hosts
in the SSH connection.
It's used to verify the identity of the server.
The function 'known\_hosts' is responsible for parsing the 'known\_hosts' configuration option
and it returns an error code depending on the result.
The error codes are:

* LIBSSH2_ERROR_NONE: the feature is not supported
* LIBSSH2_ERROR_METHOD_NOT_SUPPORTED: the method is not supported
* the host may not be known or not responding
* the line (key) starts with a newline and it's not a valid line.


```cpp
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

    /* skip leading whitespaces */
    while(len && ((*cp == ' ') || (*cp == '\t'))) {
        cp++;
        len--;
    }

    if(!len || !*cp || (*cp == '#') || (*cp == '\n'))
        /* comment or empty line */
        return LIBSSH2_ERROR_NONE;

    /* the host part starts here */
    hostp = cp;

    /* move over the host to the separator */
    while(len && *cp && (*cp != ' ') && (*cp != '\t')) {
        cp++;
        len--;
    }

    hostlen = cp - hostp;

    /* the key starts after the whitespaces */
    while(len && *cp && ((*cp == ' ') || (*cp == '\t'))) {
        cp++;
        len--;
    }

    if(!*cp || !len) /* illegal line */
        return _libssh2_error(hosts->session,
                              LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                              "Failed to parse known_hosts line");

    keyp = cp; /* the key starts here */
    keylen = len;

    /* check if the line (key) ends with a newline and if so kill it */
    while(len && *cp && (*cp != '\n')) {
        cp++;
        len--;
    }

    /* zero terminate where the newline is */
    if(*cp == '\n')
        keylen--; /* don't include this in the count */

    /* deal with this one host+key line */
    rc = hostline(hosts, hostp, hostlen, keyp, keylen);
    if(rc)
        return rc; /* failed */

    return LIBSSH2_ERROR_NONE; /* success */
}

```

该代码的作用是读取一个名为 "hosts.key" 的文件中的主机列表，并返回已成功添加的主机数量。它支持使用 SSH 协议中的 known-host 信息存储。

具体来说，代码首先会检查要使用的文件类型是否支持 known-host 信息存储，如果是 not_supported 错误，则会返回一个错误信息并打印错误消息。然后，它打开文件并读取所有内容，使用 known-host 函数来解析每一行并添加主机。如果解析失败或文件读取失败，则会返回一个错误信息。最终，它返回已成功添加的主机数量。


```cpp
/*
 * libssh2_knownhost_readfile
 *
 * Read hosts+key pairs from a given file.
 *
 * Returns a negative value for error or number of successfully added hosts.
 *
 */

LIBSSH2_API int
libssh2_knownhost_readfile(LIBSSH2_KNOWNHOSTS *hosts,
                           const char *filename, int type)
{
    FILE *file;
    int num = 0;
    char buf[4092];

    if(type != LIBSSH2_KNOWNHOST_FILE_OPENSSH)
        return _libssh2_error(hosts->session,
                              LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                              "Unsupported type of known-host information "
                              "store");

    file = fopen(filename, FOPEN_READTEXT);
    if(file) {
        while(fgets(buf, sizeof(buf), file)) {
            if(libssh2_knownhost_readline(hosts, buf, strlen(buf), type)) {
                num = _libssh2_error(hosts->session, LIBSSH2_ERROR_KNOWN_HOSTS,
                                     "Failed to parse known hosts file");
                break;
            }
            num++;
        }
        fclose(file);
    }
    else
        return _libssh2_error(hosts->session, LIBSSH2_ERROR_FILE,
                              "Failed to open file");

    return num;
}

```

This function appears to write data to a file, either for a host or for a user. It takes in data that has been extracted from theknown-hosts file, as well as some additional information that specifies the type of data and whether it contains comments.

The function first checks the type of the data and then decide how to write it. If the data is a file, it writes the data to the specified file. If the data is a user, it writes the data to the specified process.

The function also handles comments in the data. If the data contains comments, the function will write the comments to the specified file or process, if possible.

The function returns an error code and an optional output length. If the function completes successfully, it returns LIBSSH2_ERROR_NONE. If an error occurs, it returns a different error code, such as LIBSSH2_ERROR_BUFFER_TOO_SMALL.


```cpp
/*
 * knownhost_writeline()
 *
 * Ask libssh2 to convert a known host to an output line for storage.
 *
 * Note that this function returns LIBSSH2_ERROR_BUFFER_TOO_SMALL if the given
 * output buffer is too small to hold the desired output. The 'outlen' field
 * will then contain the size libssh2 wanted to store, which then is the
 * smallest sufficient buffer it would require.
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

    /* we only support this single file type for now, bail out on all other
       attempts */
    if(type != LIBSSH2_KNOWNHOST_FILE_OPENSSH)
        return _libssh2_error(hosts->session,
                              LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                              "Unsupported type of known-host information "
                              "store");

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
        if(key_type_name) {
            key_type_len = node->key_type_len;
            break;
        }
        /* otherwise fallback to default and error */
        /* FALL-THROUGH */
    default:
        return _libssh2_error(hosts->session,
                              LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                              "Unsupported type of known-host entry");
    }

    /* When putting together the host line there are three aspects to consider:
       - Hashed (SHA1) or unhashed hostname
       - key name or no key name (RSA1)
       - comment or no comment

       This means there are 2^3 different formats:
       ("|1|%s|%s %s %s %s\n", salt, hashed_host, key_name, key, comment)
       ("|1|%s|%s %s %s\n", salt, hashed_host, key_name, key)
       ("|1|%s|%s %s %s\n", salt, hashed_host, key, comment)
       ("|1|%s|%s %s\n", salt, hashed_host, key)
       ("%s %s %s %s\n", host, key_name, key, comment)
       ("%s %s %s\n", host, key_name, key)
       ("%s %s %s\n", host, key, comment)
       ("%s %s\n", host, key)

       Even if the buffer is too small, we have to set outlen to the number of
       characters the complete line would have taken.  We also don't write
       anything to the buffer unless we are sure we can write everything to the
       buffer. */

    required_size = strlen(node->key);

    if(key_type_len)
        required_size += key_type_len + 1; /* ' ' = 1 */
    if(node->comment)
        required_size += node->comment_len + 1; /* ' ' = 1 */

    if((node->typemask & LIBSSH2_KNOWNHOST_TYPE_MASK) ==
       LIBSSH2_KNOWNHOST_TYPE_SHA1) {
        char *namealloc;
        size_t name_base64_len;
        char *saltalloc;
        size_t salt_base64_len;

        name_base64_len = _libssh2_base64_encode(hosts->session, node->name,
                                                 node->name_len, &namealloc);
        if(!name_base64_len)
            return _libssh2_error(hosts->session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for "
                                  "base64-encoded host name");

        salt_base64_len = _libssh2_base64_encode(hosts->session,
                                                 node->salt, node->salt_len,
                                                 &saltalloc);
        if(!salt_base64_len) {
            LIBSSH2_FREE(hosts->session, namealloc);
            return _libssh2_error(hosts->session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for "
                                  "base64-encoded salt");
        }

        required_size += salt_base64_len + name_base64_len + 7;
        /* |1| + | + ' ' + \n + \0 = 7 */

        if(required_size <= buflen) {
            if(node->comment && key_type_len)
                snprintf(buf, buflen, "|1|%s|%s %s %s %s\n", saltalloc,
                         namealloc, key_type_name, node->key, node->comment);
            else if(node->comment)
                snprintf(buf, buflen, "|1|%s|%s %s %s\n", saltalloc, namealloc,
                         node->key, node->comment);
            else if(key_type_len)
                snprintf(buf, buflen, "|1|%s|%s %s %s\n", saltalloc, namealloc,
                         key_type_name, node->key);
            else
                snprintf(buf, buflen, "|1|%s|%s %s\n", saltalloc, namealloc,
                         node->key);
        }

        LIBSSH2_FREE(hosts->session, namealloc);
        LIBSSH2_FREE(hosts->session, saltalloc);
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

    /* we report the full length of the data with the trailing zero excluded */
    *outlen = required_size-1;

    if(required_size <= buflen)
        return LIBSSH2_ERROR_NONE;
    else
        return _libssh2_error(hosts->session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                              "Known-host write buffer too small");
}

```

这段代码定义了一个名为 `libssh2_knownhost_writeline()` 的函数，它接受一个名为 `hosts` 的 `libssh2_knownhosts` 类型的数据结构，该数据结构包含一个已知主机列表。函数的作用是将已知主机列表中的每个主机转换为输出行并存储到缓冲区中，如果缓冲区不足以存储所有主机信息，函数将返回 LIBSSH2_ERROR_BUFFER_TOO_SMALL，否则函数将成功执行。

具体实现中，函数首先检查给定的已知主机列表是否包含有效的已知主机，如果不是，函数将返回 LIBSSH2_ERROR_INVAL，并指出原因。接着，函数将遍历已知主机列表中的每个主机，将其作为 `known_host` 结构体的一部分，并调用一个名为 `knownhost_writeline()` 的函数，该函数将从 `known_host` 结构体中读取所有必要的信息，并将其写入缓冲区。如果缓冲区不足以存储所有主机信息，函数将返回 LIBSSH2_ERROR_BUFFER_TOO_SMALL，否则函数将成功执行。


```cpp
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
    struct known_host *node;

    if(known->magic != KNOWNHOST_MAGIC)
        return _libssh2_error(hosts->session, LIBSSH2_ERROR_INVAL,
                              "Invalid host information");

    node = known->node;

    return knownhost_writeline(hosts, node, buffer, buflen, outlen, type);
}

```

这段代码的作用是实现将已知主机列表中的所有主机键值对写入到一个给定的文件中。已知主机列表是通过遍历 knownhost 结构体来获得的，每个已知主机对象包含 host 属性和 key 属性。writefile() 函数接受一个已知主机列表、要写入的文件名和一个表示文件类型（以 LIBSSH2_KNOWNHOST_FILE_OPENSSH 为分界点）的参数。函数首先检查要使用的文件类型是否支持，如果不支持，则输出 LIBSSH2_ERROR_METHOD_NOT_SUPPORTED。

如果已配置为支持文件类型，函数打开一个给定的文件，并逐行将已知主机列表中的每个主机键值对写入到文件中。在写入数据之前，函数首先调用 knownhost_writeline() 函数尝试在文件中写入数据。如果写入数据失败，函数输出 LIBSSH2_ERROR_FILE 并关闭文件。写入数据完成后，函数关闭文件并返回 LIBSSH2_ERROR_NONE。


```cpp
/*
 * libssh2_knownhost_writefile()
 *
 * Write hosts+key pairs to the given file.
 */
LIBSSH2_API int
libssh2_knownhost_writefile(LIBSSH2_KNOWNHOSTS *hosts,
                            const char *filename, int type)
{
    struct known_host *node;
    FILE *file;
    int rc = LIBSSH2_ERROR_NONE;
    char buffer[4092];

    /* we only support this single file type for now, bail out on all other
       attempts */
    if(type != LIBSSH2_KNOWNHOST_FILE_OPENSSH)
        return _libssh2_error(hosts->session,
                              LIBSSH2_ERROR_METHOD_NOT_SUPPORTED,
                              "Unsupported type of known-host information "
                              "store");

    file = fopen(filename, FOPEN_WRITETEXT);
    if(!file)
        return _libssh2_error(hosts->session, LIBSSH2_ERROR_FILE,
                              "Failed to open file");

    for(node = _libssh2_list_first(&hosts->head);
        node;
        node = _libssh2_list_next(&node->node)) {
        size_t wrote = 0;
        size_t nwrote;
        rc = knownhost_writeline(hosts, node, buffer, sizeof(buffer), &wrote,
                                 type);
        if(rc)
            break;

        nwrote = fwrite(buffer, 1, wrote, file);
        if(nwrote != wrote) {
            /* failed to write the whole thing, bail out */
            rc = _libssh2_error(hosts->session, LIBSSH2_ERROR_FILE,
                                "Write failed");
            break;
        }
    }
    fclose(file);

    return rc;
}


```

这段代码定义了一个名为 libssh2_knownhost_get 的函数，它用于获取内部列表中的已知主机，并允许用户传递一个指向已知主机的指针以及一个指向已知主机前一个元素的指针。

函数首先定义了一个名为 hosts 的已知主机结构体数组，该数组包含一系列已知主机。然后定义了一个名为 ext 的已知主机结构体指针，用于存储当前已知主机，以及一个名为 oprev 的已知主机结构体指针，用于存储前一个已知主机。

函数中有一个 if 语句，用于确定前一个已知主机是否为空。如果是，则直接返回 0，表示没有前一个已知主机。否则，函数将遍历已知主机的列表，并返回 1，表示找到了一个已知主机，然后将 knownhost_to_external 函数的返回值存储到 ext 指针中，并返回 0。

函数的作用是帮助用户遍历已知主机列表，并在已知主机列表中找到了一个或多个已知主机时返回 1，否则返回 0。


```cpp
/*
 * libssh2_knownhost_get()
 *
 * Traverse the internal list of known hosts. Pass NULL to 'prev' to get
 * the first one.
 *
 * Returns:
 * 0 if a fine host was stored in 'store'
 * 1 if end of hosts
 * [negative] on errors
 */
LIBSSH2_API int
libssh2_knownhost_get(LIBSSH2_KNOWNHOSTS *hosts,
                      struct libssh2_knownhost **ext,
                      struct libssh2_knownhost *oprev)
{
    struct known_host *node;
    if(oprev && oprev->node) {
        /* we have a starting point */
        struct known_host *prev = oprev->node;

        /* get the next node in the list */
        node = _libssh2_list_next(&prev->node);

    }
    else
        node = _libssh2_list_first(&hosts->head);

    if(!node)
        /* no (more) node */
        return 1;

    *ext = knownhost_to_external(node);

    return 0;
}

```