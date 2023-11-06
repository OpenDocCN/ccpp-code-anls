# Nmap源码解析 100

# `libssh2/src/mac.c`

This is a software license, specifically the SSH (Secure Shell) license. It was first published by Sara Golemon in 2004 and is currently in effect. The license allows users to do whatever they want with the software, as long as they include the copyright notice and a list of conditions disavowing any warranties. The license also specifies that the software may not be used to endorse or promote products derived from it without specific prior written permission.


```cpp
/* Copyright (c) 2004-2007, Sara Golemon <sarag@libssh2.org>
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

这段代码是一个用于在 libssh2_mac 函数中实现 mac_none_MAC 函数的代码。libssh2_mac 和 mac_none_MAC 是在 macOS 平台上使用 libssh2 库时需要调用的函数。

具体来说，这段代码实现了一个 mac_none_MAC 函数，用于在没有 MAC 地址的情况下接收数据。这个函数接收的数据是一个 8 字节的缓冲区（buf），一个包含数据套接字长度的 32 位序列号（seqno），以及一个包含数据分片的长度。然后，函数处理的数据是一个包含 32 个字节（32-byte boundary）的分片，每个分片包含一个数据分片，分片之间没有任何字节间隔。

mac_none_MAC 的函数体中包含一个简单的 if 语句，用于判断是否支持接收数据。如果不支持，函数体返回 0，否则继续执行。如果支持，函数体中的代码会尝试从接收的数据中提取出目标分片，并将其与附加分片的长度相加，以确定目标分片的位置。如果目标分片不在数据中，函数体返回 0。

总之，这段代码的作用是实现了一个在 libssh2_mac 函数中接收没有 MAC 地址的数据的功能。


```cpp
#include "libssh2_priv.h"
#include "mac.h"

#ifdef LIBSSH2_MAC_NONE
/* mac_none_MAC
 * Minimalist MAC: No MAC
 */
static int
mac_none_MAC(LIBSSH2_SESSION * session, unsigned char *buf,
             uint32_t seqno, const unsigned char *packet,
             uint32_t packet_len, const unsigned char *addtl,
             uint32_t addtl_len, void **abstract)
{
    return 0;
}




```

这段代码定义了一个名为 mac_method_none 的结构体，它是一个 Mac 方法，包含一个字符串类型的方法名称以及一个整数类型的计数器。这个结构体定义了 Mac 方法的行为，即使在没有实现具体的方法时，它们也必须被正确地定义。

这个头文件包含一个 mac_method_none 类型的变量，它的值为 "none"，这个值表示这个结构体定义了一个没有实现的方法。

接下来的代码是一个头文件，它定义了一个名为 mac_method_common_init 的函数，用于初始化简单 Mac 方法。

这个函数接收三个参数：LIBSSH2_SESSION 类型的 session，一个指向整数类型的 key 的指针，以及一个指向 void 类型的 abstract 类型的指针。函数内部先将 key 赋值为传入的 key，然后将 free_key 赋值为 0，最后将 session 和 abstract 指向针的值设置为 NULL。函数返回 0，表示 Mac 方法已经成功初始化。


```cpp
static LIBSSH2_MAC_METHOD mac_method_none = {
    "none",
    0,
    0,
    NULL,
    mac_none_MAC,
    NULL
};
#endif /* LIBSSH2_MAC_NONE */

/* mac_method_common_init
 * Initialize simple mac methods
 */
static int
mac_method_common_init(LIBSSH2_SESSION * session, unsigned char *key,
                       int *free_key, void **abstract)
{
    *abstract = key;
    *free_key = 0;
    (void) session;

    return 0;
}



```

这段代码定义了一个名为 mac_method_common_dtor的函数，属于 libssh2 库中的一种方法。它的作用是清理与抽象相关的 mac 方法，然后将抽象指针置为 NULL。

具体来说，函数接收两个参数：一个指向抽象对象的指针，另一个是一个指向 void 类型的指针。如果接收的抽象对象已经被使用过，函数会将其 release，否则将接收的指针置为 NULL，以便在函数结束时释放内存。

函数的实现比较简单，主要目的是为了让 libssh2 库在移植到其他平台或进行修改时，能够更好地支持不同的抽象实现。


```cpp
/* mac_method_common_dtor
 * Cleanup simple mac methods
 */
static int
mac_method_common_dtor(LIBSSH2_SESSION * session, void **abstract)
{
    if(*abstract) {
        LIBSSH2_FREE(session, *abstract);
    }
    *abstract = NULL;

    return 0;
}



```

这段代码是一个名为 `mac_method_hmac_sha2_512_hash` 的函数，它是使用 SHA-256 算法的 HMAC 版本的 `libssh2_hmac_sha2_512` 库函数。该函数接受一个 `LIBSSH2_SESSION` 类型的输入，一个包含数据的字节缓冲区 `buf`，以及一系列 `uint32_t` 类型的输入参数，包括序列号 `seqno`、数据 `packet`、数据长度 `packet_len` 和附加数据 `addtl`。函数的作用是计算数据 `packet` 和附加数据 `addtl` 的哈希值，并将结果存储到字节缓冲区 `buf` 中。

函数的核心部分如下：
```cppc
static int
mac_method_hmac_sha2_512_hash(LIBSSH2_SESSION * session,
                         unsigned char *buf, uint32_t seqno,
                         const unsigned char *packet,
                         uint32_t packet_len,
                         const unsigned char *addtl,
                         uint32_t addtl_len, void **abstract)
```
首先，函数创建了一个 `libssh2_hmac_ctx` 上下文，并将其与传递给 `libssh2_htonu32` 函数的 `seqno` 合并。然后，函数创建了一个 `libssh2_hmac_sha512_ctx` 实例，并将其与 `libssh2_hmac_sha2_512` 函数的输入参数 `abstract` 合并。

接下来，函数使用 `libssh2_hmac_update` 函数对输入参数 `seqno_buf` 和 `packet`，对输入参数 `packet_len` 和 `addtl` 做哈希计算。最后，函数调用 `libssh2_hmac_final` 函数计算哈希值，并将结果存储到输入参数 `buf` 中。最后，函数使用 `libssh2_hmac_cleanup` 函数释放资源。

总体来说，该函数的作用是计算数据包和附加数据的哈希值，并将结果存储到目标数据缓冲区 `buf` 中。


```cpp
#if LIBSSH2_HMAC_SHA512
/* mac_method_hmac_sha512_hash
 * Calculate hash using full sha512 value
 */
static int
mac_method_hmac_sha2_512_hash(LIBSSH2_SESSION * session,
                          unsigned char *buf, uint32_t seqno,
                          const unsigned char *packet,
                          uint32_t packet_len,
                          const unsigned char *addtl,
                          uint32_t addtl_len, void **abstract)
{
    libssh2_hmac_ctx ctx;
    unsigned char seqno_buf[4];
    (void) session;

    _libssh2_htonu32(seqno_buf, seqno);

    libssh2_hmac_ctx_init(ctx);
    libssh2_hmac_sha512_init(&ctx, *abstract, 64);
    libssh2_hmac_update(ctx, seqno_buf, 4);
    libssh2_hmac_update(ctx, packet, packet_len);
    if(addtl && addtl_len) {
        libssh2_hmac_update(ctx, addtl, addtl_len);
    }
    libssh2_hmac_final(ctx, buf);
    libssh2_hmac_cleanup(&ctx);

    return 0;
}



```

这段代码定义了一个名为“mac_method_hmac_sha2_512”的常量，其值为“hmac-sha2-512”。

接着，定义了一个64位的常量“64”。

然后，定义了一个名为“mac_method_common_init”的函数，用于初始化Mac方法链中各个方法的参数。

接着，定义了一个名为“mac_method_hmac_sha2_512_hash”的函数，用于计算Mac方法链中“hmac-sha2-512”方法输出的哈希值。

最后，定义了一个名为“mac_method_common_dtor”的函数，用于清理Mac方法链中“hmac-sha2-512”方法在计算哈希值时的一些辅助函数。

通过这些函数，可以创建一个Mac方法链对象，并使用其中的“hmac-sha2-512”方法来计算输入数据的哈希值。


```cpp
static const LIBSSH2_MAC_METHOD mac_method_hmac_sha2_512 = {
    "hmac-sha2-512",
    64,
    64,
    mac_method_common_init,
    mac_method_hmac_sha2_512_hash,
    mac_method_common_dtor,
};
#endif



#if LIBSSH2_HMAC_SHA256
/* mac_method_hmac_sha256_hash
 * Calculate hash using full sha256 value
 */
```

这段代码是一个名为“mac_method_hmac_sha2_256_hash”的函数，属于名为“libssh2”的库。它接受一个名为“session”的指针，一个包含“unsigned char”类型数据的“buf”数组，以及两个分别表示“uint32_t”类型的“seqno”和“const unsigned char *packet”参数。函数的作用是实现对数据的哈希算法。

具体来说，这段代码执行以下操作：

1. 初始化一个名为“ctx”的“libssh2_hmac_ctx”结构体和一个名为“htonu32”的函数，将“seqno”转换为“unsigned char”并存储在“seqno_buf”数组中。
2. 初始化一个名为“ctx”的“libssh2_hmac_sha256_ctx”结构体，以及一个名为“abstract”的指针。
3. 对于传入的“seqno”和“packet”数据，分别进行哈希算法计算，并将结果存储在“buf”数组中。
4. 如果同时传入了一个名为“addtl”的“const unsigned char *”类型的数据和一个名为“packet_len”的“uint32_t”类型的数据，则对“addtl”进行哈希算法计算，并将结果存储在“buf”数组中。
5. 对计算出的哈希结果进行清理操作，最后返回结果。

这段代码的作用是实现一个安全的哈希算法，可以将输入的数据（包括消息ID和数据）哈希为固定长度的输出，以保护数据不被泄露。


```cpp
static int
mac_method_hmac_sha2_256_hash(LIBSSH2_SESSION * session,
                          unsigned char *buf, uint32_t seqno,
                          const unsigned char *packet,
                          uint32_t packet_len,
                          const unsigned char *addtl,
                          uint32_t addtl_len, void **abstract)
{
    libssh2_hmac_ctx ctx;
    unsigned char seqno_buf[4];
    (void) session;

    _libssh2_htonu32(seqno_buf, seqno);

    libssh2_hmac_ctx_init(ctx);
    libssh2_hmac_sha256_init(&ctx, *abstract, 32);
    libssh2_hmac_update(ctx, seqno_buf, 4);
    libssh2_hmac_update(ctx, packet, packet_len);
    if(addtl && addtl_len) {
        libssh2_hmac_update(ctx, addtl, addtl_len);
    }
    libssh2_hmac_final(ctx, buf);
    libssh2_hmac_cleanup(&ctx);

    return 0;
}



```

这段代码定义了一个名为“mac_method_hmac_sha2_256”的常量，其作用是定义了一个哈希算法——SHA2-256的硬件安全哈希函数。

这个哈希算法使用SHA2-256算法来对输入数据进行哈希计算，其计算结果被封存在一个名为“mac_method_hmac_sha2_256”的常量中。

该哈希算法的实现包括两个函数：mac_method_hmac_sha2_256_hash和mac_method_hmac_sha2_256_dtor。其中，mac_method_hmac_sha2_256_hash函数用于对输入数据计算哈希值，mac_method_hmac_sha2_256_dtor函数则用于清理哈希值，将哈希值恢复为输入数据。

这两个函数的实现的具体细节并没有在代码中给出，因此在实际使用中，需要根据具体需求来调用对应的函数。


```cpp
static const LIBSSH2_MAC_METHOD mac_method_hmac_sha2_256 = {
    "hmac-sha2-256",
    32,
    32,
    mac_method_common_init,
    mac_method_hmac_sha2_256_hash,
    mac_method_common_dtor,
};
#endif




/* mac_method_hmac_sha1_hash
 * Calculate hash using full sha1 value
 */
```

该函数是一个使用自主实现HMAC-SHA1哈希算法的静态函数，其参数包括一个SSH2会话指针、一个输入缓冲区、一个输出缓冲区、一个IP数据报缓冲区和一些与IP数据报相关的参数。

函数的作用是接收一个IP数据报，并在接收到的数据报中执行HMAC-SHA1哈希算法。具体来说，函数将接收到的数据报与一个预先计算好的哈希值进行比较，然后将数据报与哈希值进行一系列的异或操作，直到达到规定的最大哈希值为止。如果输入数据报中包含一个附加的IP数据报，函数将这个附加的IP数据报与哈希值进行哈希运算，并将其结果存储到输出缓冲区中。最后，函数会输出一个表示哈希结果的值。


```cpp
static int
mac_method_hmac_sha1_hash(LIBSSH2_SESSION * session,
                          unsigned char *buf, uint32_t seqno,
                          const unsigned char *packet,
                          uint32_t packet_len,
                          const unsigned char *addtl,
                          uint32_t addtl_len, void **abstract)
{
    libssh2_hmac_ctx ctx;
    unsigned char seqno_buf[4];
    (void) session;

    _libssh2_htonu32(seqno_buf, seqno);

    libssh2_hmac_ctx_init(ctx);
    libssh2_hmac_sha1_init(&ctx, *abstract, 20);
    libssh2_hmac_update(ctx, seqno_buf, 4);
    libssh2_hmac_update(ctx, packet, packet_len);
    if(addtl && addtl_len) {
        libssh2_hmac_update(ctx, addtl, addtl_len);
    }
    libssh2_hmac_final(ctx, buf);
    libssh2_hmac_cleanup(&ctx);

    return 0;
}



```

这段代码定义了一个名为 mac_method_hmac_sha1 的常量，它表示了一个 HMAC(消息完整性算法)的 SHA-1 哈希函数。这个哈希函数被用于在传输数据时保护数据的完整性。

该函数实现包括两个函数：mac_method_hmac_sha1_96_hash 和 mac_method_hmac_sha1_20。其中，mac_method_hmac_sha1_96_hash函数是计算消息完整性算法 SHA-1 的哈希值，mac_method_hmac_sha1_20函数用于清理函数指针，将函数指针作为参数传递给 mac_method_hmac_sha1_96_hash函数。

函数 mac_method_hmac_sha1_96_hash的具体实现如下：

```cpp
static int
mac_method_hmac_sha1_96_hash(LIBSSH2_SESSION * session,
                            unsigned char *buf, uint32_t seqno,
                            const unsigned char *packet,
                            uint32_t packet_len,
                            const unsigned char *addtl,
                            uint32_t addtl_len, void **abstract)
{
   unsigned char temp[SHA_DIGEST_LENGTH];

   mac_method_hmac_sha1_hash(session, temp, seqno, packet, packet_len,
                             addtl, addtl_len, abstract);
   memcpy(buf, (char *) temp, 96 / 8);

   return 0;
}
```

该函数的参数包括一个指向数据的指针，一个包含序列号和数据长度的整数，一个包含数据和附加数据的整数，一个指向函数指针的指针和一个指向字符指针的指针。函数先调用 mac_method_hmac_sha1_hash 函数，传入数据、序列号、数据和附加数据、函数指针和抽象指针。然后，使用这个函数的输出计算数据的哈希值，实现数据的完整性保护。函数的输出是缓冲区的第二个字节，使用 SHA-1 算法生成一个 96 位的哈希值，取哈希值的前 96 位，然后将哈希值复制到缓冲区的第二个字节中。


```cpp
static const LIBSSH2_MAC_METHOD mac_method_hmac_sha1 = {
    "hmac-sha1",
    20,
    20,
    mac_method_common_init,
    mac_method_hmac_sha1_hash,
    mac_method_common_dtor,
};

/* mac_method_hmac_sha1_96_hash
 * Calculate hash using first 96 bits of sha1 value
 */
static int
mac_method_hmac_sha1_96_hash(LIBSSH2_SESSION * session,
                             unsigned char *buf, uint32_t seqno,
                             const unsigned char *packet,
                             uint32_t packet_len,
                             const unsigned char *addtl,
                             uint32_t addtl_len, void **abstract)
{
    unsigned char temp[SHA_DIGEST_LENGTH];

    mac_method_hmac_sha1_hash(session, temp, seqno, packet, packet_len,
                              addtl, addtl_len, abstract);
    memcpy(buf, (char *) temp, 96 / 8);

    return 0;
}



```

这段代码定义了一个名为“mac_method_hmac_sha1_96”的常量，它表示了一个使用RSA算法和16-字节硬件安全哈希(HSHA1)的哈希函数。该哈希函数使用MAC算法中的“hmac-sha1-96”算法，其硬件安全哈希值为96字节。

该哈希函数实现的主要步骤如下：

1. 定义了一个名为“mac_method_hmac_md5_hash”的函数，它接受一个LIBSSH2_SESSION类型的会话、一个32字节的主机字节、一个包含数据套接字(Routine傳輸)的套接字、一个包含数据套接字源的32字节的套接字、以及一个指向哈希结果的指针。

2. 在函数内部，首先定义了一个名为“ctx”的哈希上下文和一个名为“buf”的字节数组，用于存储哈希结果。

3. 然后定义了一个名为“packet”的32字节的套接字变量和一个名为“addtl”的32字节的套接字变量，用于存储数据源。

4. 接着定义了一个名为“seqno”的4字节整数变量，用于存储数据套接字序列号。

5. 定义了一个名为“packet_len”的32字节的变量，用于存储数据套接字长度。

6. 定义了一个名为“addtl_len”的32字节的变量，用于存储数据源长度。

7. 接着定义了一个名为“abstract”的指针变量，用于存储哈希结果的抽象描述符。

8. 定义了一个名为“mac_method_common_init”的函数，用于初始化哈希函数的参数。

9. 定义了一个名为“mac_method_hmac_sha1_96_hash”的函数，用于执行哈希函数的计算。该函数首先使用输入的哈希算法初始化哈希上下文，然后将输入的序列号、数据套接字和数据源套接字作为参数传递给函数内部的一个执行单元，用于执行哈希计算。最后，将哈希结果存储到输出缓冲区中，并使用libssh2_hmac_cleanup函数释放哈希上下文。

10. 最后定义了该常量的访问权限，即public和const类型的访问权限，以确保该常量只能被const类型的函数进行访问。


```cpp
static const LIBSSH2_MAC_METHOD mac_method_hmac_sha1_96 = {
    "hmac-sha1-96",
    12,
    20,
    mac_method_common_init,
    mac_method_hmac_sha1_96_hash,
    mac_method_common_dtor,
};

#if LIBSSH2_MD5
/* mac_method_hmac_md5_hash
 * Calculate hash using full md5 value
 */
static int
mac_method_hmac_md5_hash(LIBSSH2_SESSION * session, unsigned char *buf,
                         uint32_t seqno,
                         const unsigned char *packet,
                         uint32_t packet_len,
                         const unsigned char *addtl,
                         uint32_t addtl_len, void **abstract)
{
    libssh2_hmac_ctx ctx;
    unsigned char seqno_buf[4];
    (void) session;

    _libssh2_htonu32(seqno_buf, seqno);

    libssh2_hmac_ctx_init(ctx);
    libssh2_hmac_md5_init(&ctx, *abstract, 16);
    libssh2_hmac_update(ctx, seqno_buf, 4);
    libssh2_hmac_update(ctx, packet, packet_len);
    if(addtl && addtl_len) {
        libssh2_hmac_update(ctx, addtl, addtl_len);
    }
    libssh2_hmac_final(ctx, buf);
    libssh2_hmac_cleanup(&ctx);

    return 0;
}



```

这段代码定义了一个名为“mac_method_hmac_md5”的常量，它表示了一个HMAC-MD5算法，其作用是对输入的数据进行摘要计算。

该算法使用一个16字节的哈希函数，通过计算输入数据的MD5摘要来输出结果。这个哈希函数基于输入数据中的前96位，并将其全部转换为字符类型，以便使用中使用。

该算法还定义了一个名为“mac_method_hmac_md5_96_hash”的函数，它是该哈希算法的内部实现。这个函数接受一个LIBSSH2_SESSION类型的会话，一个包含MD5数据缓冲区的字节数组，一个输入数据的字节数，以及一系列的附加元数据。该函数首先通过调用哈希函数自身来初始化哈希状态，然后使用输入数据和附加元数据计算输出数据的摘要，最后将结果拷贝回原来的缓冲区中。

由于这个算法是单例模式，因此每个LIBSSH2_SESSION实例都应该有一个单独的实例，并且这个实例的名称应该与哈希函数名称相同。


```cpp
static const LIBSSH2_MAC_METHOD mac_method_hmac_md5 = {
    "hmac-md5",
    16,
    16,
    mac_method_common_init,
    mac_method_hmac_md5_hash,
    mac_method_common_dtor,
};

/* mac_method_hmac_md5_96_hash
 * Calculate hash using first 96 bits of md5 value
 */
static int
mac_method_hmac_md5_96_hash(LIBSSH2_SESSION * session,
                            unsigned char *buf, uint32_t seqno,
                            const unsigned char *packet,
                            uint32_t packet_len,
                            const unsigned char *addtl,
                            uint32_t addtl_len, void **abstract)
{
    unsigned char temp[MD5_DIGEST_LENGTH];
    mac_method_hmac_md5_hash(session, temp, seqno, packet, packet_len,
                             addtl, addtl_len, abstract);
    memcpy(buf, (char *) temp, 96 / 8);
    return 0;
}



```

mac_method_hmac_ripemd160_hash(const uint8_t *input, size_t length) {
   int hash = 0;
   for (size_t i = 0; i < length; i++) {
       hash = (hash ^ input[i]) * 16777619;
       hash = hash - (hash % 256);
   }
   return hash;
}


```cpp
static const LIBSSH2_MAC_METHOD mac_method_hmac_md5_96 = {
    "hmac-md5-96",
    12,
    16,
    mac_method_common_init,
    mac_method_hmac_md5_96_hash,
    mac_method_common_dtor,
};
#endif /* LIBSSH2_MD5 */

#if LIBSSH2_HMAC_RIPEMD
/* mac_method_hmac_ripemd160_hash
 * Calculate hash using ripemd160 value
 */
static int
```

该函数实现了HMAC-160哈希算法，将输入的密码串和数据作为输入，并输出与输入哈希值相符的哈希值。

具体来说，函数的实现包括以下几个步骤：

1. 初始化哈希函数的输入参数，包括SSH会话、待哈希的密码串、哈希算法版本、哈希数据长度等。
2. 将输入的密码串和数据作为参数传递给哈希函数，并执行哈希算法。
3. 如果待哈希的数据有可用的前缀，则将前缀与数据一起进行哈希计算。
4. 对哈希结果进行清理，包括输出到屏幕外的数据和释放哈希函数的内存。
5. 返回哈希结果，通常是一个与输入相同的指针。


```cpp
mac_method_hmac_ripemd160_hash(LIBSSH2_SESSION * session,
                               unsigned char *buf, uint32_t seqno,
                               const unsigned char *packet,
                               uint32_t packet_len,
                               const unsigned char *addtl,
                               uint32_t addtl_len,
                               void **abstract)
{
    libssh2_hmac_ctx ctx;
    unsigned char seqno_buf[4];
    (void) session;

    _libssh2_htonu32(seqno_buf, seqno);

    libssh2_hmac_ctx_init(ctx);
    libssh2_hmac_ripemd160_init(&ctx, *abstract, 20);
    libssh2_hmac_update(ctx, seqno_buf, 4);
    libssh2_hmac_update(ctx, packet, packet_len);
    if(addtl && addtl_len) {
        libssh2_hmac_update(ctx, addtl, addtl_len);
    }
    libssh2_hmac_final(ctx, buf);
    libssh2_hmac_cleanup(&ctx);

    return 0;
}



```

这段代码定义了一个名为 mac_method_hmac_ripemd160 的 Mac 方法。这个方法属于一个名为 libssh2_macros 的库，它提供了多种 Mac 方法来处理 HMAC-RIPEMD160 算法。

具体来说，这个方法定义了一个名为 mac_method_hmac_ripemd160 的函数，它接受两个参数：一个整数类型的哈希算法参数，和一个字符串类型的数据参数。这个函数首先通过调用 libssh2_macros 库中的 mac_method_common_init 函数来初始化哈希算法和数据，然后执行哈希算法并返回哈希结果。

static const LIBSSH2_MAC_METHOD mac_method_hmac_ripemd160_openssh_com 这个定义是用来给 mac_method_hmac_ripemd160 函数提供打开输出设备(openssh.com)的接口的。它与 mac_method_hmac_ripemd160 函数具有相同的参数和返回值类型。


```cpp
static const LIBSSH2_MAC_METHOD mac_method_hmac_ripemd160 = {
    "hmac-ripemd160",
    20,
    20,
    mac_method_common_init,
    mac_method_hmac_ripemd160_hash,
    mac_method_common_dtor,
};

static const LIBSSH2_MAC_METHOD mac_method_hmac_ripemd160_openssh_com = {
    "hmac-ripemd160@openssh.com",
    20,
    20,
    mac_method_common_init,
    mac_method_hmac_ripemd160_hash,
    mac_method_common_dtor,
};
```

这段代码定义了一个名为“mac_methods”的数组，该数组包含了多种HMAC密码算法的实现。

其中，如果您的编译器支持LIBSSH2_HMAC_SHA256，那么会首先包含“mac_method_hmac_sha2_256”函数；如果支持LIBSSH2_HMAC_SHA512，那么会包含“mac_method_hmac_sha2_512”；如果您的编译器不支持这两个算法，那么会包含“mac_method_hmac_sha1”函数；如果支持LIBSSH2_MD5，那么会包含“mac_method_hmac_md5”函数；如果支持LIBSSH2_MD5_96，那么会包含“mac_method_hmac_md5_96”。

因此，这段代码定义了一个可扩展的接口，用于在支持多种HMAC算法的情况下，对不同的算法进行操作。


```cpp
#endif /* LIBSSH2_HMAC_RIPEMD */

static const LIBSSH2_MAC_METHOD *mac_methods[] = {
#if LIBSSH2_HMAC_SHA256
    &mac_method_hmac_sha2_256,
#endif
#if LIBSSH2_HMAC_SHA512
    &mac_method_hmac_sha2_512,
#endif
    &mac_method_hmac_sha1,
    &mac_method_hmac_sha1_96,
#if LIBSSH2_MD5
    &mac_method_hmac_md5,
    &mac_method_hmac_md5_96,
#endif
```

这段代码是一个C语言代码片段，定义了一个名为`_libssh2_mac_methods`的函数。

首先，它通过`#if`和`#ifdef`条件语句来检查是否支持HMAC RIPEMD160算法。如果支持，那么定义了两个名为`mac_method_hmac_ripemd160`和`mac_method_hmac_ripemd160_openssh_com`的函数指针。如果HMAC RIPEMD160算法不支持，则定义了一个名为`mac_method_none`的函数指针，并将`NULL`作为参数传递给`mac_methods`的函数指针。

然后，它定义了一个名为`_libssh2_mac_methods`的函数，返回`mac_methods`的函数指针。

最后，它定义了一个名为`LIBSSH2_MAC_METHODS`的常量，并将`_libssh2_mac_methods`函数的返回值作为该常量的值返回。


```cpp
#if LIBSSH2_HMAC_RIPEMD
    &mac_method_hmac_ripemd160,
    &mac_method_hmac_ripemd160_openssh_com,
#endif /* LIBSSH2_HMAC_RIPEMD */
#ifdef LIBSSH2_MAC_NONE
    &mac_method_none,
#endif /* LIBSSH2_MAC_NONE */
    NULL
};

const LIBSSH2_MAC_METHOD **
_libssh2_mac_methods(void)
{
    return mac_methods;
}

```

# `libssh2/src/mbedtls.c`

This is a C header file with a copyright notice that allows for distribution and use of the software as long as the specified conditions are met. The conditions include that the software may be redistributed and used in source and binary forms, as long as the copyright notice and a list of dependencies are included, and that any contributions or modifications made to the software must also be distributed under the same copyright notice.


```cpp
/* Copyright (c) 2016, Art <https://github.com/wildart>
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

这段代码是针对mbedtls库编译的，包括了两个函数，用于mbedtls的熵和CTR Drbg上下文的相关设置。

1. _libssh2_mbedtls_entropy函数：该函数用于初始化mbedtls entropy context，包括生成RNG种子、设置计数器等。这些函数在mbedtls的实现中使用，以确保熵随机性的可靠。

2. _libssh2_mbedtls_ctr_drbg函数：该函数用于设置mbedtls CTR Drbg，用于生成安全的随机数。这些随机数可以用于实现SSH客户端和服务器之间的通信，以增加安全性。

由于这段代码仅在支持mbedtls库的编译环境中编译，所以不会输出任何可执行文件。


```cpp
#include "libssh2_priv.h"

#ifdef LIBSSH2_MBEDTLS /* compile only if we build with mbedtls */

/*******************************************************************/
/*
 * mbedTLS backend: Global context handles
 */

static mbedtls_entropy_context  _libssh2_mbedtls_entropy;
static mbedtls_ctr_drbg_context _libssh2_mbedtls_ctr_drbg;

/*******************************************************************/
/*
 * mbedTLS backend: Generic functions
 */

```

这段代码是用于初始化 libssh2-mbedtls 的函数。具体来说，它实现了以下几个主要步骤：

1. 初始化 entropy 函数，用于生成随机数。
2. 初始化时钟上下文，包括时钟周期、秒数、随机数种子等。
3. 通过时钟上下文的设置，让 libssh2-mbedtls 启动随机数生成器。
4. 如果初始化成功，就返回 0，否则输出错误信息并释放相关资源。

由于没有输出函数，因此无法得知具体生成了哪些随机数。libssh2-mbedtls 是一个用于安全地传输 SSH 连接的库，因此这些随机数可能用于生成加密消息或其他安全相关操作。


```cpp
void
_libssh2_mbedtls_init(void)
{
    int ret;

    mbedtls_entropy_init(&_libssh2_mbedtls_entropy);
    mbedtls_ctr_drbg_init(&_libssh2_mbedtls_ctr_drbg);

    ret = mbedtls_ctr_drbg_seed(&_libssh2_mbedtls_ctr_drbg,
                                mbedtls_entropy_func,
                                &_libssh2_mbedtls_entropy, NULL, 0);
    if(ret != 0)
        mbedtls_ctr_drbg_free(&_libssh2_mbedtls_ctr_drbg);
}

```

这段代码是一个用于SSH客户端的安全套接字库（libssh2）中的函数。库中包含两个函数：_libssh2_mbedtls_random() 和 _libssh2_mbedtls_free()。它们分别用于生成随机字节数组和释放内存。

_libssh2_mbedtls_random()函数接收一个长度为len的byte数组和另一个byte数组buf作为参数。它使用mbedtls_ctr_drbg_random()函数生成一个随机数，并将其存储在buf指向的内存区域。函数返回0（成功）或-1（失败）。

_libssh2_mbedtls_free()函数接收一个void类型的指针作为参数。它使用mbedtls_entropy_free()函数释放内存，包括与之前生成的随机数相关的内存。


```cpp
void
_libssh2_mbedtls_free(void)
{
    mbedtls_ctr_drbg_free(&_libssh2_mbedtls_ctr_drbg);
    mbedtls_entropy_free(&_libssh2_mbedtls_entropy);
}

int
_libssh2_mbedtls_random(unsigned char *buf, int len)
{
    int ret;
    ret = mbedtls_ctr_drbg_random(&_libssh2_mbedtls_ctr_drbg, buf, len);
    return ret == 0 ? 0 : -1;
}

```

这段代码定义了一个名为`_libssh2_mbedtls_safe_free`的函数，它的作用是释放一个已经传入的`buf`数组，数组长度为`len`。

函数首先检查`buf`是否为空，如果是，直接返回。然后，根据`libssh2_clear_memory`宏定义（但实际环境中可能没有这个宏），会尝试释放内存。如果是`mbedtls_free`函数，它会尝试直接释放`buf`所指向的内存区域。

总的来说，这段代码的主要目的是释放一个已经传入的`buf`数组，并在释放内存时执行一些额外的清理操作（例如，在`libssh2_clear_memory`宏定义中，明确定义了释放内存时需要执行的一些操作）。


```cpp
static void
_libssh2_mbedtls_safe_free(void *buf, int len)
{
#ifndef LIBSSH2_CLEAR_MEMORY
    (void)len;
#endif

    if(!buf)
        return;

#ifdef LIBSSH2_CLEAR_MEMORY
    if(len > 0)
        _libssh2_explicit_zero(buf, len);
#endif

    mbedtls_free(buf);
}

```

这段代码是一个名为`_libssh2_mbedtls_cipher_init`的函数，属于`libssh2`库的一部分。它的作用是初始化一个SSH2协程的加密配置，包括加密类型、IV、密钥和IV长度。

具体来说，函数接收一个`ctx`参数，它是`mbedtls_cipher_ctx`结构体，表示加密配置的相关信息。然后函数根据传入的加密类型（0表示加密，1表示解密），以及一个可选的IV长度，尝试从内存中初始化一个`mbedtls_cipher_info_t`结构体，用于表示加密配置。如果初始化成功，函数执行实际的加密设置，包括设置加密算法、IV生成和密钥。如果初始化失败，函数返回一个错误代码。

函数的实现基于以下几个步骤：

1. 根据传入的加密类型，定义相应的`mbedtls_cipher_info_t`结构体。
2. 如果加密类型为0，函数首先尝试从内存中初始化一个`mbedtls_cipher_info_t`结构体，用于表示加密配置。
3. 如果初始化成功，函数执行实际的加密设置，包括设置加密算法、IV生成和密钥。
4. 如果初始化失败，函数返回一个错误代码。


```cpp
int
_libssh2_mbedtls_cipher_init(_libssh2_cipher_ctx *ctx,
                             _libssh2_cipher_type(algo),
                             unsigned char *iv,
                             unsigned char *secret,
                             int encrypt)
{
    const mbedtls_cipher_info_t *cipher_info;
    int ret, op;

    if(!ctx)
        return -1;

    op = encrypt == 0 ? MBEDTLS_ENCRYPT : MBEDTLS_DECRYPT;

    cipher_info = mbedtls_cipher_info_from_type(algo);
    if(!cipher_info)
        return -1;

    mbedtls_cipher_init(ctx);
    ret = mbedtls_cipher_setup(ctx, cipher_info);
    if(!ret)
        ret = mbedtls_cipher_setkey(ctx, secret, cipher_info->key_bitlen, op);

    if(!ret)
        ret = mbedtls_cipher_set_iv(ctx, iv, cipher_info->iv_size);

    return ret == 0 ? 0 : -1;
}

```

这段代码是一个名为`_libssh2_mbedtls_cipher_crypt`的函数，它属于名为`libssh2_mbedtls_cipher`的库。它的作用是在使用某种加密算法对数据进行加密时，管理输入和输出数据。

具体来说，这个函数接受一个`_libssh2_cipher_ctx`类型的输入参数，一个表示加密算法的参数，一个要加密的数据块，以及一个数据块的大小。函数内部首先根据算法类型和输入参数计算出需要输入和输出的数据块大小，然后使用`mbedtls_calloc`函数在内存中分配足够的空间来输出数据。接着，函数使用`mbedtls_cipher_reset`和`mbedtls_cipher_update`函数对数据进行加密和解密操作，使用`mbedtls_cipher_finish`函数结束加密操作。最后，如果加密成功，函数将输出数据块的字节数复本。如果分配内存失败或者加密过程中出现错误，函数将返回一个负数。

总的来说，这个函数是一个用于管理加密数据块的函数，它能够在使用多种加密算法的情况下提供一致的接口，使得开发者能够更加方便和安全地使用加密功能。


```cpp
int
_libssh2_mbedtls_cipher_crypt(_libssh2_cipher_ctx *ctx,
                              _libssh2_cipher_type(algo),
                              int encrypt,
                              unsigned char *block,
                              size_t blocklen)
{
    int ret;
    unsigned char *output;
    size_t osize, olen, finish_olen;

    (void) encrypt;
    (void) algo;

    osize = blocklen + mbedtls_cipher_get_block_size(ctx);

    output = (unsigned char *)mbedtls_calloc(osize, sizeof(char));
    if(output) {
        ret = mbedtls_cipher_reset(ctx);

        if(!ret)
            ret = mbedtls_cipher_update(ctx, block, blocklen, output, &olen);

        if(!ret)
            ret = mbedtls_cipher_finish(ctx, output + olen, &finish_olen);

        if(!ret) {
            olen += finish_olen;
            memcpy(block, output, olen);
        }

        _libssh2_mbedtls_safe_free(output, osize);
    }
    else
        ret = -1;

    return ret == 0 ? 0 : -1;
}

```

这两段代码定义了两个名为 `_libssh2_mbedtls_cipher_dtor` 和 `_libssh2_mbedtls_hash_init` 的函数。它们都属于 `libssh2-je乐享库`（libssh2-je乐享库）中的 `_libssh2_cipher_ctx` 和 `_libssh2_mbedtls_md_context_t` 结构体。

1. `_libssh2_mbedtls_cipher_dtor` 函数：

```cppc
void
_libssh2_mbedtls_cipher_dtor(_libssh2_cipher_ctx *ctx)
{
   mbedtls_cipher_free(ctx);
}
```

这个函数接收一个 `_libssh2_cipher_ctx` 类型的参数。函数内部通过调用 `mbedtls_cipher_free` 函数来释放之前创建的加密上下文。

2. `_libssh2_mbedtls_hash_init` 函数：

```cppc
int
_libssh2_mbedtls_hash_init(mbedtls_md_context_t *ctx,
                         mbedtls_md_type_t mdtype,
                         const unsigned char *key, unsigned long keylen)
{
   const mbedtls_md_info_t *md_info;
   int ret, hmac;

   md_info = mbedtls_md_info_from_type(mdtype);
   if(!md_info)
       return 0;

   hmac = key == NULL ? 0 : 1;

   mbedtls_md_init(ctx);
   ret = mbedtls_md_setup(ctx, md_info, hmac);
   if(!ret) {
       if(hmac)
           ret = mbedtls_md_hmac_starts(ctx, key, keylen);
       else
           ret = mbedtls_md_starts(ctx);
   }

   return ret == 0 ? 1 : 0;
}
```

这个函数接收一个 `mbedtls_md_context_t` 类型的参数，一个 `const unsigned char *` 类型的输入密钥，和一个 `unsigned long` 类型的输入密钥长度。函数内部创建一个 `mbedtls_md_info_t` 类型的变量 `md_info`，并将 `mdtype` 和 `key` 字段初始化。

然后，函数调用 `mbedtls_md_init` 函数创建一个 `mbedtls_md_context` 类型的实例，并调用 `mbedtls_md_setup` 函数设置加密参数，包括初始化密钥和加密模式。如果设置成功，函数返回一个非零值表示加密成功。


```cpp
void
_libssh2_mbedtls_cipher_dtor(_libssh2_cipher_ctx *ctx)
{
    mbedtls_cipher_free(ctx);
}


int
_libssh2_mbedtls_hash_init(mbedtls_md_context_t *ctx,
                          mbedtls_md_type_t mdtype,
                          const unsigned char *key, unsigned long keylen)
{
    const mbedtls_md_info_t *md_info;
    int ret, hmac;

    md_info = mbedtls_md_info_from_type(mdtype);
    if(!md_info)
        return 0;

    hmac = key == NULL ? 0 : 1;

    mbedtls_md_init(ctx);
    ret = mbedtls_md_setup(ctx, md_info, hmac);
    if(!ret) {
        if(hmac)
            ret = mbedtls_md_hmac_starts(ctx, key, keylen);
        else
            ret = mbedtls_md_starts(ctx);
    }

    return ret == 0 ? 1 : 0;
}

```

这两段代码是一个在使用libssh2-mbedtls库的哈希函数。

首先，定义了一个名为_libssh2_mbedtls_hash_final的函数，它的作用是传入一个mbedtls_md_context_t类型的上下文和一个unsigned char类型的哈希值，并返回一个整数类型的结果。

实现过程如下：

1. 调用mbedtls_md_finish函数，这个函数会完成输入哈希值的计算，并将结果存储在哈希值中。
2. 调用mbedtls_md_free函数，这个函数用于释放使用完毕的哈希值。
3. 判断哈希值计算结果是否为0，如果是，则返回0，否则返回-1。

接下来，定义了一个名为_libssh2_mbedtls_hash的函数，它的作用和上面那个相反，但是需要一个const unsigned char *类型的数据和一个unsigned long类型的数据长度。

实现过程如下：

1. 定义一个const mbedtls_md_info_t *类型的变量md_info，用于存储输入数据类型对应的md信息。
2. 定义一个int类型的变量ret，用于存储计算哈希值的结果。
3. 调用mbedtls_md函数，这个函数会根据输入的数据类型和长度计算哈希值，并将结果存储在md_info->md_result中。
4. 判断ret是否为0，如果是，则说明计算成功，否则说明有错误，需要返回一个负数。

总的来说，这两段代码实现了一个哈希函数，用于将输入的数据和给定的类型进行哈希运算，并返回计算结果。


```cpp
int
_libssh2_mbedtls_hash_final(mbedtls_md_context_t *ctx, unsigned char *hash)
{
    int ret;

    ret = mbedtls_md_finish(ctx, hash);
    mbedtls_md_free(ctx);

    return ret == 0 ? 0 : -1;
}

int
_libssh2_mbedtls_hash(const unsigned char *data, unsigned long datalen,
                      mbedtls_md_type_t mdtype, unsigned char *hash)
{
    const mbedtls_md_info_t *md_info;
    int ret;

    md_info = mbedtls_md_info_from_type(mdtype);
    if(!md_info)
        return 0;

    ret = mbedtls_md(md_info, data, datalen, hash);

    return ret == 0 ? 0 : -1;
}

```

这段代码是一个用于mbedTLS的BigNumber函数库的初始化函数。具体来说，它做了以下几件事情：

1. 定义了一个名为_libssh2_mbedtls_bignum_init的函数，返回一个指向_libssh2_bn类型的指针变量bignum。
2. 在函数体中，使用mbedtls_calloc函数创建了一个大小为sizeof(_libssh2_bn)的内存区域，并将其赋值给bignum。
3. 如果mbedtls_calloc函数成功创建了内存区域，则调用mbedtls_mpi_init函数对bignum进行初始化。
4. 函数最终返回指向bignum的指针。

由于mbedTLS是一个基于Go的加密库，因此这些函数是在Go语言中定义的。


```cpp
/*******************************************************************/
/*
 * mbedTLS backend: BigNumber functions
 */

_libssh2_bn *
_libssh2_mbedtls_bignum_init(void)
{
    _libssh2_bn *bignum;

    bignum = (_libssh2_bn *)mbedtls_calloc(1, sizeof(_libssh2_bn));
    if(bignum) {
        mbedtls_mpi_init(bignum);
    }

    return bignum;
}

```

This code looks like it is a C implementation of the OpenSSH SSH protocol. It appears to be using the LibSSH2 library to generate random numbers for the purposes of generating random session keys.

The code defines a function `_libssh2_mbedtls_bignum_random` which takes a pointer to a `_libssh2_bn` structure, the number of bits in the random number, the top and bottom indices of the generated random number, and a caller function to be used to generate the random number.

The function first checks that the input `_libssh2_bn` structure is not null before attempting to generate the random number. If the input is not null, it creates a buffer of the required length by calling the `mbedtls_mpi_fill_random` function from the `mbedtls` library and setting it to the random number generated by the `_libssh2_mbedtls_bignum_random` function.

If the `_libssh2_mbedtls_bignum_random` function fails to generate the random number, it returns -1. If the function succeeds, it then checks the usage of the random number and fills in the most significant bit of the random number using the `mbedtls_mpi_set_bit` function from the `mbedtls` library.

Finally, the function compares the remaining of the random number bits to 0 and, if the top index is negative, it sets the most significant bit to 0. This is done using the `for` loop and the `if` statement.


```cpp
void
_libssh2_mbedtls_bignum_free(_libssh2_bn *bn)
{
    if(bn) {
        mbedtls_mpi_free(bn);
        mbedtls_free(bn);
    }
}

static int
_libssh2_mbedtls_bignum_random(_libssh2_bn *bn, int bits, int top, int bottom)
{
    size_t len;
    int err;
    int i;

    if(!bn || bits <= 0)
        return -1;

    len = (bits + 7) >> 3;
    err = mbedtls_mpi_fill_random(bn, len, mbedtls_ctr_drbg_random,
                                  &_libssh2_mbedtls_ctr_drbg);
    if(err)
        return -1;

    /* Zero unused bits above the most significant bit*/
    for(i = len*8 - 1; bits <= i; --i) {
        err = mbedtls_mpi_set_bit(bn, i, 0);
        if(err)
            return -1;
    }

    /* If `top` is -1, the most significant bit of the random number can be
       zero.  If top is 0, the most significant bit of the random number is
       set to 1, and if top is 1, the two most significant bits of the number
       will be set to 1, so that the product of two such random numbers will
       always have 2*bits length.
    */
    for(i = 0; i <= top; ++i) {
        err = mbedtls_mpi_set_bit(bn, bits-i-1, 1);
        if(err)
            return -1;
    }

    /* make odd by setting first bit in least significant byte */
    if(bottom) {
        err = mbedtls_mpi_set_bit(bn, 0, 1);
        if(err)
            return -1;
    }

    return 0;
}


```

This function appears to be a part of a library that provides a secure and efficient way to perform secure key exchange between two parties using RSA and MSE (MPA) protocols.

It appears to handle the tasks of generating RSA keys, checking the server's public key against the client's public key, and checking the client's public key against the server's public key. It also appears to take care of error handling, such as returning -1 if any of the files or keys cannot be found or if there is an error during the key generation process.

The function takes in the RSA and MSE keys for the server and the client, and uses the `mbedtls_rsa_generate_private_key()` function to generate the server's private key. It then returns the server's public key in the format `MbedtlsRSA public_key`.

If the server's public key cannot be generated or the generated key is not valid, the function returns -1. If the client's public key cannot be verified or the client's public key is not valid, the function returns -1.

It also appears to take care of error handling, such as returning -1 if any of the files or keys cannot be found or if there is an error during the key generation process.


```cpp
/*******************************************************************/
/*
 * mbedTLS backend: RSA functions
 */

int
_libssh2_mbedtls_rsa_new(libssh2_rsa_ctx **rsa,
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
                        const unsigned char *coeffdata,
                        unsigned long coefflen)
{
    int ret;
    libssh2_rsa_ctx *ctx;

    ctx = (libssh2_rsa_ctx *) mbedtls_calloc(1, sizeof(libssh2_rsa_ctx));
    if(ctx != NULL) {
        mbedtls_rsa_init(ctx, MBEDTLS_RSA_PKCS_V15, 0);
    }
    else
        return -1;

    /* !checksrc! disable ASSIGNWITHINCONDITION 1 */
    if((ret = mbedtls_mpi_read_binary(&(ctx->E), edata, elen) ) != 0 ||
       (ret = mbedtls_mpi_read_binary(&(ctx->N), ndata, nlen) ) != 0) {
        ret = -1;
    }

    if(!ret) {
        ctx->len = mbedtls_mpi_size(&(ctx->N));
    }

    if(!ret && ddata) {
        /* !checksrc! disable ASSIGNWITHINCONDITION 1 */
        if((ret = mbedtls_mpi_read_binary(&(ctx->D), ddata, dlen) ) != 0 ||
           (ret = mbedtls_mpi_read_binary(&(ctx->P), pdata, plen) ) != 0 ||
           (ret = mbedtls_mpi_read_binary(&(ctx->Q), qdata, qlen) ) != 0 ||
           (ret = mbedtls_mpi_read_binary(&(ctx->DP), e1data, e1len) ) != 0 ||
           (ret = mbedtls_mpi_read_binary(&(ctx->DQ), e2data, e2len) ) != 0 ||
           (ret = mbedtls_mpi_read_binary(&(ctx->QP), coeffdata, coefflen) )
           != 0) {
            ret = -1;
        }
        ret = mbedtls_rsa_check_privkey(ctx);
    }
    else if(!ret) {
        ret = mbedtls_rsa_check_pubkey(ctx);
    }

    if(ret && ctx) {
        _libssh2_mbedtls_rsa_free(ctx);
        ctx = NULL;
    }
    *rsa = ctx;
    return ret;
}

```

这段代码定义了一个名为`_libssh2_mbedtls_rsa_new_private`的函数，它的作用是接收一个RSA加密密钥、一个SSH会话ID和一个私钥文件名，然后尝试使用私钥文件中的私钥内容创建一个新的RSA加密密钥。

具体来说，函数接收三个参数：

1. 一个指向RSA加密密钥上下文的指针`rsa`，它用于存储新创建的RSA加密密钥。
2. 一个指向SSH会话对象的指针`session`，它用于存储SSH会话信息。
3. 一个指向字符串的指针`filename`，它用于存储私钥文件名。
4. 一个指向字节数组的指针`passphrase`，它用于存储私钥文件中的密码。

函数首先尝试从内存中申请一块足够大的空间用于存储RSA加密密钥。如果内存申请失败，函数返回一个负数。

接着，函数调用一个名为`mbedtls_rsa_init`的函数来初始化接收到的RSA加密密钥。这个函数接受三个参数：一个指向RSA加密密钥上下文的指针`pk_rsa`，一个字符串指针`passphrase`，它们用于存储私钥文件中的密码内容，以及一个指向整数的指针`algorithm`，用于指定使用哪种加密算法。

函数接着尝试使用传递给它的私钥文件内容创建一个新的RSA加密密钥。具体来说，它首先调用`mbedtls_pk_parse_keyfile`函数尝试从私钥文件中读取密码内容，并将其存储在`pk_rsa`指向的内存空间中。然后，它调用`mbedtls_pk_get_type`函数检查读取的密码内容是否为RSA类型，如果是，就返回MBEDTLS_PK_RSA。

如果私钥文件读取失败或者读取的密码内容不是RSA类型，函数就释放之前分配的内存空间，并返回一个负数，表示创建RSA加密密钥失败。


```cpp
int
_libssh2_mbedtls_rsa_new_private(libssh2_rsa_ctx **rsa,
                                LIBSSH2_SESSION *session,
                                const char *filename,
                                const unsigned char *passphrase)
{
    int ret;
    mbedtls_pk_context pkey;
    mbedtls_rsa_context *pk_rsa;

    *rsa = (libssh2_rsa_ctx *) LIBSSH2_ALLOC(session, sizeof(libssh2_rsa_ctx));
    if(*rsa == NULL)
        return -1;

    mbedtls_rsa_init(*rsa, MBEDTLS_RSA_PKCS_V15, 0);
    mbedtls_pk_init(&pkey);

    ret = mbedtls_pk_parse_keyfile(&pkey, filename, (char *)passphrase);
    if(ret != 0 || mbedtls_pk_get_type(&pkey) != MBEDTLS_PK_RSA) {
        mbedtls_pk_free(&pkey);
        mbedtls_rsa_free(*rsa);
        LIBSSH2_FREE(session, *rsa);
        *rsa = NULL;
        return -1;
    }

    pk_rsa = mbedtls_pk_rsa(pkey);
    mbedtls_rsa_copy(*rsa, pk_rsa);
    mbedtls_pk_free(&pkey);

    return 0;
}

```

This function appears to be used to handle the key material for an RSA key in an SSH handshake. It takes in the passphrase as a parameter, and is responsible for generating and processing the key material.

It first checks if the passphrase is null or not. If it's null, it creates an empty array and assigns it to the filedata\_nullterm pointer.

Then, it creates a pointer to the key data, and a pointer to the null term of the passphrase.

It then calls the mbedtls\_pk\_init and mbedtls\_pk\_parse\_key functions to generate and parse the key material.

If the key material is generated and parsed successfully, it releases the memory and returns 0.

If the passphrase is invalid, or the key material cannot be generated or parsed successfully, it returns -1.

Finally, it frees the memory and returns 0.


```cpp
int
_libssh2_mbedtls_rsa_new_private_frommemory(libssh2_rsa_ctx **rsa,
                                           LIBSSH2_SESSION *session,
                                           const char *filedata,
                                           size_t filedata_len,
                                           unsigned const char *passphrase)
{
    int ret;
    mbedtls_pk_context pkey;
    mbedtls_rsa_context *pk_rsa;
    void *filedata_nullterm;
    size_t pwd_len;

    *rsa = (libssh2_rsa_ctx *) mbedtls_calloc(1, sizeof(libssh2_rsa_ctx));
    if(*rsa == NULL)
        return -1;

    /*
    mbedtls checks in "mbedtls/pkparse.c:1184" if "key[keylen - 1] != '\0'"
    private-key from memory will fail if the last byte is not a null byte
    */
    filedata_nullterm = mbedtls_calloc(filedata_len + 1, 1);
    if(filedata_nullterm == NULL) {
        return -1;
    }
    memcpy(filedata_nullterm, filedata, filedata_len);

    mbedtls_pk_init(&pkey);

    pwd_len = passphrase != NULL ? strlen((const char *)passphrase) : 0;
    ret = mbedtls_pk_parse_key(&pkey, (unsigned char *)filedata_nullterm,
                               filedata_len + 1,
                               passphrase, pwd_len);
    _libssh2_mbedtls_safe_free(filedata_nullterm, filedata_len);

    if(ret != 0 || mbedtls_pk_get_type(&pkey) != MBEDTLS_PK_RSA) {
        mbedtls_pk_free(&pkey);
        mbedtls_rsa_free(*rsa);
        LIBSSH2_FREE(session, *rsa);
        *rsa = NULL;
        return -1;
    }

    pk_rsa = mbedtls_pk_rsa(pkey);
    mbedtls_rsa_copy(*rsa, pk_rsa);
    mbedtls_pk_free(&pkey);

    return 0;
}

```

这段代码是一个名为`int libssh2_mbedtls_rsa_sha1_verify`的函数，它接受两个参数：一个`int`类型的`rsa`指针和一个`const unsigned char *`类型的`sig`参数，这两个参数都是输入数据。

此外，函数还接受三个参数：一个`const unsigned char *`类型的`m`指针和一个`unsigned long`类型的`m_len`参数，这两个参数用于存储待验证的输入数据。

函数首先调用一个名为`_libssh2_mbedtls_hash`的函数，该函数接受两个参数：一个`const unsigned char *`类型的`m`参数和一个`unsigned long`类型的`m_len`参数，这两个参数用于计算输入数据的摘要。这个摘要算法是使用SHA-1哈希算法。

接着，函数调用一个名为`mbedtls_rsa_pkcs1_verify`的函数，这个函数接受一个`const unsigned char *`类型的`rsa`参数，一个`const unsigned char *`类型的`NULL`参数，和一个`unsigned long`类型的`MBEDTLS_MD_SHA1`参数，以及一个SHA-1哈希函数产生的字节数`MD_SHA1_DIGEST_LENGTH`。此外，这个函数的最后一个参数是一个`const unsigned char *`类型的`hash`参数，用于存储计算得到的哈希值。

最后，函数根据`mbedtls_rsa_pkcs1_verify`函数的返回值来输出一个状态信息，要么输出0表示成功，要么输出-1表示失败。


```cpp
int
_libssh2_mbedtls_rsa_sha1_verify(libssh2_rsa_ctx *rsa,
                                const unsigned char *sig,
                                unsigned long sig_len,
                                const unsigned char *m,
                                unsigned long m_len)
{
    unsigned char hash[SHA_DIGEST_LENGTH];
    int ret;

    ret = _libssh2_mbedtls_hash(m, m_len, MBEDTLS_MD_SHA1, hash);
    if(ret)
        return -1; /* failure */

    ret = mbedtls_rsa_pkcs1_verify(rsa, NULL, NULL, MBEDTLS_RSA_PUBLIC,
                                   MBEDTLS_MD_SHA1, SHA_DIGEST_LENGTH,
                                   hash, sig);

    return (ret == 0) ? 0 : -1;
}

```

这段代码是一个名为`libssh2_mbedtls_rsa_sha1_sign`的函数，它接受一个名为`session`的SSH2会话实例，一个名为`rsa`的RSA加密密钥，一个名为`hash`的哈希值，以及一个名为`signature`的签名输出。

该函数首先检查哈希值的长度是否与传递给它的参数匹配，如果不匹配，函数将返回-1。

然后，函数使用`mbedtls_rsa_pkcs1_sign`函数对RSA加密密钥进行签名，得到一个签名输出`sig`，以及一个签名长度`sig_len`。

接下来，函数检查是否成功地将签名输出中的值复制到`signature`缓冲区中，并检查返回值是否为0。如果是0，则表示签名操作成功，函数将释放所有分配的内存并返回0。如果签名操作失败，函数将返回-1并释放所有分配的内存。


```cpp
int
_libssh2_mbedtls_rsa_sha1_sign(LIBSSH2_SESSION *session,
                              libssh2_rsa_ctx *rsa,
                              const unsigned char *hash,
                              size_t hash_len,
                              unsigned char **signature,
                              size_t *signature_len)
{
    int ret;
    unsigned char *sig;
    unsigned int sig_len;

    (void)hash_len;

    sig_len = rsa->len;
    sig = LIBSSH2_ALLOC(session, sig_len);
    if(!sig) {
        return -1;
    }

    ret = mbedtls_rsa_pkcs1_sign(rsa, NULL, NULL, MBEDTLS_RSA_PRIVATE,
                                 MBEDTLS_MD_SHA1, SHA_DIGEST_LENGTH,
                                 hash, sig);
    if(ret) {
        LIBSSH2_FREE(session, sig);
        return -1;
    }

    *signature = sig;
    *signature_len = sig_len;

    return (ret == 0) ? 0 : -1;
}

```

这段代码定义了一个名为 `_libssh2_mbedtls_rsa_free` 的函数，它的参数是一个名为 `libssh2_rsa_ctx` 的指向 `mbedtls_rsa_context` 的指针，这个函数会在函数内部释放资源。

接着定义了一个名为 `gen_publickey_from_rsa` 的函数，它接受两个参数：`LIBSSH2_SESSION` 类型的 `session`，一个 `mbedtls_rsa_context` 类型的 `rsa`，以及一个指向 `size_t` 类型的 *`keylen` 指针。这个函数的作用是在 `rsa` 和 `session` 变量的基础上生成公钥，并将生成的公钥返回。

具体实现过程如下：

1. 从 `rsa` 变量中提取 RSA 的 E 字节和 N 字节，然后计算出 RSA 模数为 2 字节时的大致长度。
2. 在内存中申请一块长度为计算得到的长度加上 1（为了保证有足够的空间存放 RSA 的参数）的内存，并将 RSA 的参数复制到内存中。
3. 对 RSA 的参数进行 ASCII 编码。
4. 对 RSA 的参数进行 Base64 编码。
5. 通过 `memcpy` 函数将编码后的 RSA 参数复制到申请的内存起始位置，并计算出实际的 RSA 长度。
6. 返回生成的公钥。


```cpp
void
_libssh2_mbedtls_rsa_free(libssh2_rsa_ctx *ctx)
{
    mbedtls_rsa_free(ctx);
    mbedtls_free(ctx);
}

static unsigned char *
gen_publickey_from_rsa(LIBSSH2_SESSION *session,
                      mbedtls_rsa_context *rsa,
                      size_t *keylen)
{
    int            e_bytes, n_bytes;
    unsigned long  len;
    unsigned char *key;
    unsigned char *p;

    e_bytes = mbedtls_mpi_size(&rsa->E);
    n_bytes = mbedtls_mpi_size(&rsa->N);

    /* Key form is "ssh-rsa" + e + n. */
    len = 4 + 7 + 4 + e_bytes + 4 + n_bytes;

    key = LIBSSH2_ALLOC(session, len);
    if(!key) {
        return NULL;
    }

    /* Process key encoding. */
    p = key;

    _libssh2_htonu32(p, 7);  /* Key type. */
    p += 4;
    memcpy(p, "ssh-rsa", 7);
    p += 7;

    _libssh2_htonu32(p, e_bytes);
    p += 4;
    mbedtls_mpi_write_binary(&rsa->E, p, e_bytes);

    _libssh2_htonu32(p, n_bytes);
    p += 4;
    mbedtls_mpi_write_binary(&rsa->N, p, n_bytes);

    *keylen = (size_t)(p - key);
    return key;
}

```

这段代码是一个名为 `_libssh2_mbedtls_pub_priv_key` 的函数，它接受一个 `LIBSSH2_SESSION` 类型的会话，以及一个指向 `unsigned char` 类型的指针和一个指向 `size_t` 类型的指针，分别表示要输出加密方法和输出数据的长度。它同时也接受一个指向 `mbedtls_pk_context` 类型的指针，用于存储公钥。

该函数的作用是：根据传入的参数，首先检查传入的 `pkey` 是否支持 RSA 类型，如果不是，则返回一个错误码。然后，根据输入的 `method` 和 `pubkeydata` 长度，生成一个 RSA 公钥并对输入的 `pubkeydata` 进行拷贝。接着，根据输入的 `method` 和 `pubkeydata_len`，尝试使用生成的公钥对输入的 `pubkeydata` 进行加密，并输出加密后的结果。如果生成公钥或加密失败，则返回相应的错误码。


```cpp
static int
_libssh2_mbedtls_pub_priv_key(LIBSSH2_SESSION *session,
                               unsigned char **method,
                               size_t *method_len,
                               unsigned char **pubkeydata,
                               size_t *pubkeydata_len,
                               mbedtls_pk_context *pkey)
{
    unsigned char *key = NULL, *mth = NULL;
    size_t keylen = 0, mthlen = 0;
    int ret;
    mbedtls_rsa_context *rsa;

    if(mbedtls_pk_get_type(pkey) != MBEDTLS_PK_RSA) {
        mbedtls_pk_free(pkey);
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,
                              "Key type not supported");
    }

    /* write method */
    mthlen = 7;
    mth = LIBSSH2_ALLOC(session, mthlen);
    if(mth) {
        memcpy(mth, "ssh-rsa", mthlen);
    }
    else {
        ret = -1;
    }

    rsa = mbedtls_pk_rsa(*pkey);
    key = gen_publickey_from_rsa(session, rsa, &keylen);
    if(key == NULL) {
        ret = -1;
    }

    /* write output */
    if(ret) {
        if(mth)
            LIBSSH2_FREE(session, mth);
        if(key)
            LIBSSH2_FREE(session, key);
    }
    else {
        *method = mth;
        *method_len = mthlen;
        *pubkeydata = key;
        *pubkeydata_len = keylen;
    }

    return ret;
}

```

这段代码是一个名为`_libssh2_mbedtls_pub_priv_keyfile`的函数，属于SSH2库的一部分。它的作用是公开一个SSH2客户端的私钥数据。

具体来说，该函数接受一个SSH2会话对象（LIBSSH2_SESSION），一个方法名称（unsigned char **method）和一个方法长度（size_t *method_len），一个公钥数据（unsigned char **pubkeydata）和一个公钥数据长度（size_t *pubkeydata_len），还有两个密码（const char *privatekey 和 const char *passphrase）。

首先，函数使用`mbedtls_pk_init`函数创建一个名为`pkey`的`mbedtls_pk_context`结构体。然后，使用`mbedtls_pk_parse_keyfile`函数将私钥文件（const char *privatekey 和 const char *passphrase）解析为字节数组。如果解析失败，函数将返回一个错误码，并使用`mbedtls_strerror`函数打印错误信息。

接下来，函数调用另一个名为`_libssh2_mbedtls_pub_priv_key`的函数，将私钥数据公开给客户端。函数的第一个参数是一个方法名称（unsigned char *method），第二个参数是一个方法长度（size_t *method_len），第三个参数是一个公钥数据缓冲区（unsigned char *pubkeydata）。这两个参数将在函数返回前被保留。

最后，函数使用`mbedtls_pk_free`函数释放私钥数据和`mbedtls_strerror`函数的错误码。如果函数成功执行并且所有参数都正确，它将返回0。否则，它将返回一个错误码，并打印错误信息。


```cpp
int
_libssh2_mbedtls_pub_priv_keyfile(LIBSSH2_SESSION *session,
                                 unsigned char **method,
                                 size_t *method_len,
                                 unsigned char **pubkeydata,
                                 size_t *pubkeydata_len,
                                 const char *privatekey,
                                 const char *passphrase)
{
    mbedtls_pk_context pkey;
    char buf[1024];
    int ret;

    mbedtls_pk_init(&pkey);
    ret = mbedtls_pk_parse_keyfile(&pkey, privatekey, passphrase);
    if(ret != 0) {
        mbedtls_strerror(ret, (char *)buf, sizeof(buf));
        mbedtls_pk_free(&pkey);
        return _libssh2_error(session, LIBSSH2_ERROR_FILE, buf);
    }

    ret = _libssh2_mbedtls_pub_priv_key(session, method, method_len,
                                       pubkeydata, pubkeydata_len, &pkey);

    mbedtls_pk_free(&pkey);

    return ret;
}

```

0

This function appears to be part of the OpenSSH library, and it appears to handle the process of generating a private key from a passphrase and a private key data file.

It takes three arguments:

- `privatekeydata`: a pointer to a buffer containing the private key data
- `privatekeydata_len`: the length of the `privatekeydata` buffer
- `passphrase`: a pointer to a buffer containing the passphrase

It returns either a success or failure result, along with an error message if the private key cannot be generated.

It looks like the function uses the `mbedtls_pk_parse_key` function to generate the private key from the passphrase and the private key data file. This function expects the passphrase to be a null-terminated C-style string, and the private key data file to be a byte array containing the passphrase.

If the private key can be generated successfully, it is returned. Otherwise, the function returns an error code and a strerror message for failure.


```cpp
int
_libssh2_mbedtls_pub_priv_keyfilememory(LIBSSH2_SESSION *session,
                                       unsigned char **method,
                                       size_t *method_len,
                                       unsigned char **pubkeydata,
                                       size_t *pubkeydata_len,
                                       const char *privatekeydata,
                                       size_t privatekeydata_len,
                                       const char *passphrase)
{
    mbedtls_pk_context pkey;
    char buf[1024];
    int ret;
    void *privatekeydata_nullterm;
    size_t pwd_len;

    /*
    mbedtls checks in "mbedtls/pkparse.c:1184" if "key[keylen - 1] != '\0'"
    private-key from memory will fail if the last byte is not a null byte
    */
    privatekeydata_nullterm = mbedtls_calloc(privatekeydata_len + 1, 1);
    if(privatekeydata_nullterm == NULL) {
        return -1;
    }
    memcpy(privatekeydata_nullterm, privatekeydata, privatekeydata_len);

    mbedtls_pk_init(&pkey);

    pwd_len = passphrase != NULL ? strlen((const char *)passphrase) : 0;
    ret = mbedtls_pk_parse_key(&pkey,
                               (unsigned char *)privatekeydata_nullterm,
                               privatekeydata_len + 1,
                               (const unsigned char *)passphrase, pwd_len);
    _libssh2_mbedtls_safe_free(privatekeydata_nullterm, privatekeydata_len);

    if(ret != 0) {
        mbedtls_strerror(ret, (char *)buf, sizeof(buf));
        mbedtls_pk_free(&pkey);
        return _libssh2_error(session, LIBSSH2_ERROR_FILE, buf);
    }

    ret = _libssh2_mbedtls_pub_priv_key(session, method, method_len,
                                       pubkeydata, pubkeydata_len, &pkey);

    mbedtls_pk_free(&pkey);

    return ret;
}

```



这段代码定义了一个名为 `_libssh2_init_aes_ctr` 的函数，它是一个名为 `void` 的函数，但并没有具体的函数体，即该函数没有实现任何功能。

接下来，定义了一个名为 `_libssh2_dh_init` 的函数，它接受一个名为 `dhctx` 的参数，并调用了名为 `_libssh2_mbedtls_bignum_init` 的函数，生成了随机数。通过调用这个函数，初始化了一位板卡 - 数据库(DH)的参数。

总结起来，这段代码定义了一个函数 `_libssh2_dh_init`，用于初始化一位板卡 - 数据库(DH)，但该函数没有具体的实现，只是一个头文件，需要通过其他函数来完成具体的初始化操作。


```cpp
void _libssh2_init_aes_ctr(void)
{
    /* no implementation */
}


/*******************************************************************/
/*
 * mbedTLS backend: Diffie-Hellman functions
 */

void
_libssh2_dh_init(_libssh2_dh_ctx *dhctx)
{
    *dhctx = _libssh2_mbedtls_bignum_init();    /* Random from client */
}

```

这两函数是用来生成SSH客户端和服务器之间的密钥对。具体来说，这两个函数分别负责生成公钥、私钥，以及使用这两个密钥生成SSH客户端和服务器之间的交换。

*_libssh2_dh_key_pair函数的作用是生成公钥和私钥，并且公钥用于与服务器建立安全连接，私钥用于加密数据以发送给服务器。
*_libssh2_dh_secret函数的作用是使用公钥和私钥生成SSH客户端和服务器之间的交换。

这两个函数的具体实现主要依赖于libssh2库，它可以提供安全地建立SSH连接的功能，并支持加密和验证数据等功能。


```cpp
int
_libssh2_dh_key_pair(_libssh2_dh_ctx *dhctx, _libssh2_bn *public,
                     _libssh2_bn *g, _libssh2_bn *p, int group_order)
{
    /* Generate x and e */
    _libssh2_mbedtls_bignum_random(*dhctx, group_order * 8 - 1, 0, -1);
    mbedtls_mpi_exp_mod(public, g, *dhctx, p, NULL);
    return 0;
}

int
_libssh2_dh_secret(_libssh2_dh_ctx *dhctx, _libssh2_bn *secret,
                   _libssh2_bn *f, _libssh2_bn *p)
{
    /* Compute the shared secret */
    mbedtls_mpi_exp_mod(secret, f, *dhctx, p, NULL);
    return 0;
}

```

这段代码是使用libssh2_dh库实现对SSH客户端的ECDSA密钥进行操作的函数。

具体来说，代码中定义了一个名为_libssh2_dh_dtor的函数，该函数接收一个SSH客户端的ECDSA密钥上下文(公钥和私钥)作为参数。在函数内部，首先通过调用*dhctx指针所指向的内存区域，释放了之前使用过的ECDSA密钥数据。然后，将*dhctx指向的内存区域置为空指针，表示没有更多的ECDSA密钥数据可以分配。

接下来的两行代码是在ECDSA库中定义的函数，用于创建本地私钥。这两行代码的具体实现可能因使用的ECDSA库而异，但通常会涉及从输入曲线中计算出一个随机数，然后使用这个随机数作为私钥。

最后，函数没有返回值，但有两个void类型的参数被定义为output，这意味着如果有错误发生，这些参数将作为错误信息返回。


```cpp
void
_libssh2_dh_dtor(_libssh2_dh_ctx *dhctx)
{
    _libssh2_mbedtls_bignum_free(*dhctx);
    *dhctx = NULL;
}

#if LIBSSH2_ECDSA

/*******************************************************************/
/*
 * mbedTLS backend: ECDSA functions
 */

/*
 * _libssh2_ecdsa_create_key
 *
 * Creates a local private key based on input curve
 * and returns octal value and octal length
 *
 */

```

这段代码是一个名为 `_libssh2_mbedtls_ecdsa_create_key` 的函数，它用于创建一个名为 `privkey` 的 `ec_key` 变量和一个名为 `pubkey_oct` 的 `unsigned char` 变量，同时分配一个包含 `pubkey_oct` 大小 的内存。

该函数的主要作用是创建一个新的 `ec_key` 并将其赋值给 `privkey`，然后使用 `mbedtls_ecdsa_genkey` 函数生成新的 `ec_key`，再将生成的 `ec_key` 以及随机生成的密钥信息存回 `privkey`。接着，使用 `mbedtls_mpi_point_write_binary` 函数将 `ec_key` 中的公钥信息写入 `pubkey_oct` 数组中，并将数组长度设置为 `pubkey_oct_len`。最后，函数返回 0，表示创建成功。


```cpp
int
_libssh2_mbedtls_ecdsa_create_key(LIBSSH2_SESSION *session,
                                  _libssh2_ec_key **privkey,
                                  unsigned char **pubkey_oct,
                                  size_t *pubkey_oct_len,
                                  libssh2_curve_type curve)
{
    size_t plen = 0;

    *privkey = LIBSSH2_ALLOC(session, sizeof(mbedtls_ecp_keypair));

    if(*privkey == NULL)
        goto failed;

    mbedtls_ecdsa_init(*privkey);

    if(mbedtls_ecdsa_genkey(*privkey, (mbedtls_ecp_group_id)curve,
                            mbedtls_ctr_drbg_random,
                            &_libssh2_mbedtls_ctr_drbg) != 0)
        goto failed;

    plen = 2 * mbedtls_mpi_size(&(*privkey)->grp.P) + 1;
    *pubkey_oct = LIBSSH2_ALLOC(session, plen);

    if(*pubkey_oct == NULL)
        goto failed;

    if(mbedtls_ecp_point_write_binary(&(*privkey)->grp, &(*privkey)->Q,
                                      MBEDTLS_ECP_PF_UNCOMPRESSED,
                                      pubkey_oct_len, *pubkey_oct, plen) == 0)
        return 0;

```

这段代码是关于 SSH 密钥的创建和释放。它具体实现了以下功能：

1. `failed:` 是一个错误处理标志，如果在函数执行过程中出现错误，将输出错误信息并返回 `-1`，否则不执行任何操作。

2. `_libssh2_mbedtls_ecdsa_free(*privkey)` 函数接收一个用户提供的私钥指针作为输入，然后释放该私钥指针。

3. `_libssh2_mbedtls_safe_free(*pubkey_oct, plen)` 函数接收一个用户提供的公钥的八进制字符串作为输入，以及一个用户指定的公钥字符数组长度。然后，它尝试从内存中释放公钥指针和长度，如果释放失败，函数将返回 `-1`，否则不执行任何操作。

4. `*privkey = NULL` 将私钥指针初始化为 `NULL`，以便在需要时进行释放。

5. `return -1;` 如果以上操作成功，函数返回 `0`，否则返回 `-1`。


```cpp
failed:

    _libssh2_mbedtls_ecdsa_free(*privkey);
    _libssh2_mbedtls_safe_free(*pubkey_oct, plen);
    *privkey = NULL;

    return -1;
}

/* _libssh2_ecdsa_curve_name_with_octal_new
 *
 * Creates a new public key given an octal string, length and type
 *
 */

```

这段代码是一个名为`int libssh2_mbedtls_ecdsa_curve_name_with_octal_new`的函数，它接受一个指向`libssh2_ecdsa_ctx`类型的指针参数`ctx`，一个包含`const unsigned char *k`和`size_t k_len`的整数参数，以及一个`libssh2_curve_type`类型的参数`curve`。

函数首先通过`mbedtls_calloc`函数创建一个包含`mbedtls_ecp_keypair`结构体的指针变量`ctx`，如果分配失败，则直接跳转到`goto failed`标签。

接下来，函数调用了`mbedtls_ecdsa_init`函数，该函数的第一个参数`ctx`指的就是上面创建的指针变量`ctx`，第二个参数是一个`mbedtls_ecp_group_id`类型的整数，表示要加载的ECP组。如果这个函数调用失败，则直接跳转到`goto failed`标签。

接着，函数使用`mbedtls_ecp_point_read_binary`函数从ECP组中读取点，并将其存储在`Q`结构体中。这个点是使用Binary格式读取的，因此需要提供ECP组的私钥。

最后，函数使用`mbedtls_ecp_check_pubkey`函数检查ECP组的公钥是否与传入的私钥匹配。如果匹配成功，则返回0，否则返回一个负值表示函数执行失败。

该函数的作用是创建一个名为`libssh2_mbedtls_ecdsa_curve_name_with_octal_new`的函数，它接受一个指向`libssh2_ecdsa_ctx`类型的指针参数`ctx`，一个包含`const unsigned char *k`和`size_t k_len`的整数参数，以及一个`libssh2_curve_type`类型的参数`curve`。它通过调用`mbedtls_ecdsa_init`和`mbedtls_ecp_point_read_binary`函数来加载ECP组并验证私钥，然后使用`mbedtls_ecp_check_pubkey`函数检查匹配程度。如果匹配成功，则返回0，否则返回一个负值表示函数执行失败。


```cpp
int
_libssh2_mbedtls_ecdsa_curve_name_with_octal_new(libssh2_ecdsa_ctx **ctx,
                                                 const unsigned char *k,
                                                 size_t k_len,
                                                 libssh2_curve_type curve)
{
    *ctx = mbedtls_calloc(1, sizeof(mbedtls_ecp_keypair));

    if(*ctx == NULL)
        goto failed;

    mbedtls_ecdsa_init(*ctx);

    if(mbedtls_ecp_group_load(&(*ctx)->grp, (mbedtls_ecp_group_id)curve) != 0)
        goto failed;

    if(mbedtls_ecp_point_read_binary(&(*ctx)->grp, &(*ctx)->Q, k, k_len) != 0)
        goto failed;

    if(mbedtls_ecp_check_pubkey(&(*ctx)->grp, &(*ctx)->Q) == 0)
        return 0;

```

这段代码是针对SSH客户端库中的一部分函数，具体来说，它包括了两个函数：`failed:` 和 `_libssh2_mbedtls_ecdsa_free()`。

首先来看 `failed:` 函数。这个函数的作用是输出一个错误信息，然后将`ctx`指针设置为`NULL`。这个函数通常是在发生错误时被调用，比如在使用SSH客户端库时，如果出现了异常情况，就可以调用这个函数来输出错误信息，然后释放内存。

接下来是 `_libssh2_mbedtls_ecdsa_free()` 函数。这个函数的作用是释放一个`ECDSAResolver`结构的指针变量`ctx`所指向的内存。这个函数通常是在`ECDSAResolver`结构被使用完后，需要释放内存时调用。如果释放失败，可能会导致内存泄漏，从而引发严重的安全问题。

总结一下，这两个函数都是SSH客户端库中的一部分，用于处理错误和释放资源。


```cpp
failed:

    _libssh2_mbedtls_ecdsa_free(*ctx);
    *ctx = NULL;

    return -1;
}

/* _libssh2_ecdh_gen_k
 *
 * Computes the shared secret K given a local private key,
 * remote public key and length
 */

int
```

这段代码的作用是实现了一个SSH客户端与服务器之间的交互，实现了客户端尝试连接到服务器，并在连接成功后，客户端可以发送公钥给服务器，并且服务器验证客户端公钥是否正确。

具体来说，代码中定义了两个指针变量：一个指向SSH客户端的指针k，一个指向SSH客户端的私钥的指针privkey。同时，定义了一个字符指针server_pubkey，用于存储服务器公示的public key。

接下来，代码使用mbedtls_bc等函数实现了一系列加密和验证操作。首先，定义了一个EC public point pubkey，然后尝试使用mbedtls_ecp_point_init函数，将pubkey初始化为从服务器公示的public key中提取的值。接着，使用mbedtls_ecp_point_read_binary函数，将pubkey从服务器公示的public key中读取并初始化。

然后，定义了一个EC point变量d，用于存储服务器公示的public key和私钥之间的差异化。接着，使用mbedtls_ecdh_compute_shared函数，将d与server_pubkey一起作为输入，计算出一个新的EC point，并将其与服务器公示的public key一起作为输出，同时使用mbedtls_ctr_drbg_random函数生成一个随机数作为种子，用于计算新的EC point的值。

最后，使用mbedtls_ecp_check_privkey函数，检验客户端私钥是否正确，如果错误则返回-1。


```cpp
_libssh2_mbedtls_ecdh_gen_k(_libssh2_bn **k,
                            _libssh2_ec_key *privkey,
                            const unsigned char *server_pubkey,
                            size_t server_pubkey_len)
{
    mbedtls_ecp_point pubkey;
    int rc = 0;

    if(*k == NULL)
        return -1;

    mbedtls_ecp_point_init(&pubkey);

    if(mbedtls_ecp_point_read_binary(&privkey->grp, &pubkey,
                                     server_pubkey, server_pubkey_len) != 0) {
        rc = -1;
        goto cleanup;
    }

    if(mbedtls_ecdh_compute_shared(&privkey->grp, *k,
                                   &pubkey, &privkey->d,
                                   mbedtls_ctr_drbg_random,
                                   &_libssh2_mbedtls_ctr_drbg) != 0) {
        rc = -1;
        goto cleanup;
    }

    if(mbedtls_ecp_check_privkey(&privkey->grp, *k) != 0)
        rc = -1;

```

这段代码定义了一个名为 "cleanup" 的函数，其作用是释放之前分配的Pubkey。然后，通过返回一个整数 rc，表示SSH2和ECC格式的MD5摘要是否有效。

这里还定义了一个名为 "libssh2_mbedtls_ECDSA_VERIFY" 的函数，用于使用MD5摘要验证SSH2客户端与远程服务器之间的ECDSA(ECDSA)身份验证。该函数需要一个名为 "digest_type" 的参数，它的值可以是 "libssh2_sha256" 或 "libssh2_sha512"。函数通过使用libssh2_sha256或libssh2_sha512函数来验证输入的MD5摘要，并将其与定义在LIBSSH2_MBEDTLS_ECDSA_VERIFY函数中的哈希值进行比较。如果两个函数都成功，则返回一个表示MD5摘要有效性的整数，rc将设置为0。否则，rc将设置为非零值。


```cpp
cleanup:

    mbedtls_ecp_point_free(&pubkey);

    return rc;
}

#define LIBSSH2_MBEDTLS_ECDSA_VERIFY(digest_type)                   \
{                                                                   \
    unsigned char hsh[SHA##digest_type##_DIGEST_LENGTH];            \
                                                                    \
    if(libssh2_sha##digest_type(m, m_len, hsh) == 0) {              \
        rc = mbedtls_ecdsa_verify(&ctx->grp, hsh,                   \
                                  SHA##digest_type##_DIGEST_LENGTH, \
                                  &ctx->Q, &pr, &ps);               \
    }                                                               \
                                                                    \
}

```

这段代码是一个名为`_libssh2_ecdsa_verify`的函数，它用于验证消息的ECDSA签名。函数接受四个参数：`ctx`是一个ECDSACryptContext实例，包含了验证所需的私钥信息；`r`是一个消息哈希，用于比较ECDSA签名；`s`是一个待签名的消息，也是哈希；`m`是一个已知哈希的消息，同样用于比较ECDSA签名。函数实现了一个switch语句，根据输入的曲线类型来选择验证方式，然后分别对每个曲线类型进行验证。如果验证通过，函数返回0，否则返回一个负数。


```cpp
/* _libssh2_ecdsa_sign
 *
 * Verifies the ECDSA signature of a hashed message
 *
 */

int
_libssh2_mbedtls_ecdsa_verify(libssh2_ecdsa_ctx *ctx,
                              const unsigned char *r, size_t r_len,
                              const unsigned char *s, size_t s_len,
                              const unsigned char *m, size_t m_len)
{
    mbedtls_mpi pr, ps;
    int rc = -1;

    mbedtls_mpi_init(&pr);
    mbedtls_mpi_init(&ps);

    if(mbedtls_mpi_read_binary(&pr, r, r_len) != 0)
        goto cleanup;

    if(mbedtls_mpi_read_binary(&ps, s, s_len) != 0)
        goto cleanup;

    switch(_libssh2_ecdsa_get_curve_type(ctx)) {
    case LIBSSH2_EC_CURVE_NISTP256:
        LIBSSH2_MBEDTLS_ECDSA_VERIFY(256);
        break;
    case LIBSSH2_EC_CURVE_NISTP384:
        LIBSSH2_MBEDTLS_ECDSA_VERIFY(384);
        break;
    case LIBSSH2_EC_CURVE_NISTP521:
        LIBSSH2_MBEDTLS_ECDSA_VERIFY(512);
        break;
    default:
        rc = -1;
    }

```

这段代码是针对libssh2_ecdsa_parse_eckey函数进行实现的。该函数的作用是解析包含ECC key对数据的公钥。以下是具体解释：

1. 首先，使用mbedtls_mpi_free函数释放public和private keys的内存。
2. 返回0表示成功，-1表示失败。
3. _libssh2_mbedtls_parse_eckey函数接受三个参数：ecdsa context、public key和private key对。
4. 首先，获取输入数据的长度，并计算得到pwd的长度。
5. 然后，根据不同的输入数据类型，实现不同的函数。
6. 如果实现成功，则分配内存并设置ecdsa上下文。
7. 调用ecdsa_init函数初始化ecdsa上下文。
8. 调用ecdsa_from_keypair函数，将public key对和private key对作为输入，并获取返回值。
9. 最后，根据返回值，输出成功或失败的信息。


```cpp
cleanup:

    mbedtls_mpi_free(&pr);
    mbedtls_mpi_free(&ps);

    return (rc == 0) ? 0 : -1;
}

static int
_libssh2_mbedtls_parse_eckey(libssh2_ecdsa_ctx **ctx,
                             mbedtls_pk_context *pkey,
                             LIBSSH2_SESSION *session,
                             const unsigned char *data,
                             size_t data_len,
                             const unsigned char *pwd)
{
    size_t pwd_len;

    pwd_len = pwd ? strlen((const char *) pwd) : 0;

    if(mbedtls_pk_parse_key(pkey, data, data_len, pwd, pwd_len) != 0)
        goto failed;

    if(mbedtls_pk_get_type(pkey) != MBEDTLS_PK_ECKEY)
        goto failed;

    *ctx = LIBSSH2_ALLOC(session, sizeof(libssh2_ecdsa_ctx));

    if(*ctx == NULL)
        goto failed;

    mbedtls_ecdsa_init(*ctx);

    if(mbedtls_ecdsa_from_keypair(*ctx, mbedtls_pk_ec(*pkey)) == 0)
        return 0;

```

This is a C function that takes a pointer to an SSH client data object, a pointer to a buffer containing the data, and a pointer to a boolean indicating whether the data has been decrypted. It performs the following tasks:

1. Allocates memory for an SSH2 ECDSA context and initializes it with the data and the randomly generated point using the data and the exponent.
2. Adds the point to the ECDSA context.
3. Reads the data from the buffer and converts it to a binary format.
4. Encrypts the data using the ECDSA context.
5. Checks if the data has been successfully decrypted.

If any errors occur, it prints them and jumps to the "failed" label. If the encryption is successful, it prints nothing and jumps to the "cleanup" label.


```cpp
failed:

    _libssh2_mbedtls_ecdsa_free(*ctx);
    *ctx = NULL;

    return -1;
}

static int
_libssh2_mbedtls_parse_openssh_key(libssh2_ecdsa_ctx **ctx,
                                   LIBSSH2_SESSION *session,
                                   const unsigned char *data,
                                   size_t data_len,
                                   const unsigned char *pwd)
{
    libssh2_curve_type type;
    unsigned char *name = NULL;
    struct string_buf *decrypted = NULL;
    size_t curvelen, exponentlen, pointlen;
    unsigned char *curve, *exponent, *point_buf;

    if(_libssh2_openssh_pem_parse_memory(session, pwd,
                                         (const char *)data, data_len,
                                         &decrypted) != 0)
        goto failed;

    if(_libssh2_get_string(decrypted, &name, NULL) != 0)
        goto failed;

    if(_libssh2_mbedtls_ecdsa_curve_type_from_name((const char *)name,
                                                   &type) != 0)
        goto failed;

    if(_libssh2_get_string(decrypted, &curve, &curvelen) != 0)
        goto failed;

    if(_libssh2_get_string(decrypted, &point_buf, &pointlen) != 0)
        goto failed;

    if(_libssh2_get_bignum_bytes(decrypted, &exponent, &exponentlen) != 0)
        goto failed;

    *ctx = LIBSSH2_ALLOC(session, sizeof(libssh2_ecdsa_ctx));

    if(*ctx == NULL)
        goto failed;

    mbedtls_ecdsa_init(*ctx);

    if(mbedtls_ecp_group_load(&(*ctx)->grp, (mbedtls_ecp_group_id)type) != 0)
        goto failed;

    if(mbedtls_mpi_read_binary(&(*ctx)->d, exponent, exponentlen) != 0)
        goto failed;

    if(mbedtls_ecp_mul(&(*ctx)->grp, &(*ctx)->Q,
                       &(*ctx)->d, &(*ctx)->grp.G,
                       mbedtls_ctr_drbg_random,
                       &_libssh2_mbedtls_ctr_drbg) != 0)
        goto failed;

    if(mbedtls_ecp_check_privkey(&(*ctx)->grp, &(*ctx)->d) == 0)
        goto cleanup;

```

这段代码是用于在SSH客户端（libssh2）中执行ECDSA（Elliptic Curve Digital Signature Algorithm）私钥的创建。其具体作用如下：

1. 首先，通过调用函数`failed:`，成功地将之前创建的ECDSA私钥的上下文（ctx）释放。

2. 接着，通过调用函数`cleanup:`，对已经创建的ECDSA私钥进行清理。如果之前已经解除了ECDSA私钥，这将确保所有内存都被释放。

3. 在`cleanup:`函数中，首先尝试通过调用`_libssh2_string_buf_free`函数来释放已经分配的、基于文件路径和密码的密钥数据。如果该函数成功，说明私钥数据已经被正确地编码并且可以被释放，于是将`decrypted`变量设置为`NULL`。

4. 最后，返回一个值，指示ECDSA私钥的创建结果。如果`failed:`尚未发生，则表示创建失败，返回`-1`；如果私钥成功创建，则返回`0`。


```cpp
failed:

    _libssh2_mbedtls_ecdsa_free(*ctx);
    *ctx = NULL;

cleanup:

    if(decrypted) {
        _libssh2_string_buf_free(session, decrypted);
    }

    return (*ctx == NULL) ? -1 : 0;
}

/* _libssh2_ecdsa_new_private
 *
 * Creates a new private key given a file path and password
 *
 */

```

这段代码的作用是创建一个新的SSH客户端私钥对ECDSA签名算法，并将其存储在内存中。它实现了以下主要步骤：

1. 从传入的文件中读取ECDSA签名密钥数据。
2. 对密钥数据进行初始化。
3. 对密钥数据进行签名。
4. 将签名后的密钥数据存储在输入的ctx指向的内存空间中。


```cpp
int
_libssh2_mbedtls_ecdsa_new_private(libssh2_ecdsa_ctx **ctx,
                                   LIBSSH2_SESSION *session,
                                   const char *filename,
                                   const unsigned char *pwd)
{
    mbedtls_pk_context pkey;
    unsigned char *data;
    size_t data_len;

    if(mbedtls_pk_load_file(filename, &data, &data_len) != 0)
        goto cleanup;

    mbedtls_pk_init(&pkey);

    if(_libssh2_mbedtls_parse_eckey(ctx, &pkey, session,
                                    data, data_len, pwd) == 0)
        goto cleanup;

    _libssh2_mbedtls_parse_openssh_key(ctx, session, data, data_len, pwd);

```

这段代码是一个 C 语言函数，名为 "cleanup"，属于 "libssh2" 库。它的作用是释放之前分配的资源和关闭套接字。

具体来说，这段代码以下几个步骤：

1. 释放 PKey 资源的指针：
```cpp
mbedtls_pk_free(&pkey);
```
2. 释放由 data_len 指向的内存：
```cpp
_libssh2_mbedtls_safe_free(data, data_len);
```
3. 返回前两个参数的值，如果前两个参数为空，则返回 -1：
```cpp
return (*ctx == NULL) ? -1 : 0;
```
4. 通过调用 "libssh2_ecdsa_new_private" 函数创建新私钥。但是，这个函数没有返回新私钥的存储位置，因此这个位置是固定的，并不影响 "cleanup" 函数的实现。


```cpp
cleanup:

    mbedtls_pk_free(&pkey);

    _libssh2_mbedtls_safe_free(data, data_len);

    return (*ctx == NULL) ? -1 : 0;
}

/* _libssh2_ecdsa_new_private
 *
 * Creates a new private key given a file data and password
 *
 */

```

这段代码的作用是创建一个新的PrivateKey对ecdsa类型的密钥，并将其存储在内存中。

具体来说，代码中调用了两个函数：LIBSSH2_ALLOC和LIBSSH2_MBEDTLS_ECDSA_parse_eckey。

LIBSSH2_ALLOC函数的作用是分配一块内存，用于存储输入的数据。这个函数的返回值是一个指针，如果分配失败，则返回NULL。

LIBSSH2_MBEDTLS_ECDSA_parse_eckey函数的作用是解析输入的ECKey数据。这个函数需要一个PrivateKey对和一个数据缓冲区。在函数内部，首先用从libssh2_ecdsa_ctx结构中获取的PrivateKey初始化了一个mbedtls_pk_context对象，然后调用了LIBSSH2_MBEDTLS_ECDSA_parse_openssh_key函数，将数据缓冲区解析为了openssh_key结构，最终将openssh_key结构存储到了PrivateKey中。

整个函数的实现要点如下：

1. 分配一块内存，并将其存储在ctx指向的变量ntdata中。

2. 从session指向的内存中复制数据缓冲区中的数据。

3. 用从libssh2_ecdsa_ctx结构中获取的PrivateKey初始化了一个mbedtls_pk_context对象，并调用了LIBSSH2_MBEDTLS_ECDSA_parse_openssh_key函数，将数据缓冲区解析为了openssh_key结构，最终将openssh_key结构存储到了PrivateKey中。


```cpp
int
_libssh2_mbedtls_ecdsa_new_private_frommemory(libssh2_ecdsa_ctx **ctx,
                                              LIBSSH2_SESSION *session,
                                              const char *data,
                                              size_t data_len,
                                              const unsigned char *pwd)
{
    unsigned char *ntdata;
    mbedtls_pk_context pkey;

    mbedtls_pk_init(&pkey);

    ntdata = LIBSSH2_ALLOC(session, data_len + 1);

    if(ntdata == NULL)
        goto cleanup;

    memcpy(ntdata, data, data_len);

    if(_libssh2_mbedtls_parse_eckey(ctx, &pkey, session,
                                    ntdata, data_len + 1, pwd) == 0)
        goto cleanup;

    _libssh2_mbedtls_parse_openssh_key(ctx, session,
                                       ntdata, data_len + 1, pwd);

```

这段代码是针对libssh2_mbedtls_mpi_write_binary函数的实现。

这段代码的主要作用是释放之前分配的内存，并输出一个整数，表示write_binary函数的执行结果。

具体来说，代码首先通过mbedtls_pk_free函数释放了pkey变量所指向的加密密钥。

接着，通过_libssh2_mbedtls_safe_free函数释放了长整型变量ntdata所指向的数据，该数据用于存储libssh2_mbedtls_mpi_write_binary函数需要写入的目标数据。

然后，函数返回了一个整数，表示write_binary函数的执行结果。如果执行结果为NULL，则表示函数在执行过程中失败，可以输出一个负数或者不输出。如果执行结果为0，则表示write_binary函数成功执行，并将返回值输出为0。

_libssh2_mbedtls_mpi_write_binary函数的实现主要分为以下几个步骤：

1. 定义了一个宏，名为_ZNK免费订阅，该宏用于计算数据缓冲区中可用内存的大小。

2. 通过移动指针p，将数据缓冲区从start指向end，并设置end为当前可用的最大内存大小。

3. 如果可用的内存大小小于4，则说明数据缓冲区足够大，可以开始输出数据。

4. 如果可用的内存大小大于0，则先通过mbedtls_mpi_write_binary函数将剩余的数据写入数据缓冲区。

5. 如果可用的内存大小大于0且数据缓冲区中已有的数据行以4字节为单位组，则通过memmove函数将多餘的4字节数据从内存中拷贝到数据缓冲区中。

6. 最后，将数据缓冲区中所有元素的长度转换为无符号32位整数，并将其存储回变量p-4。

7. 通过_libssh2_htonu32函数将8个字节的高字节转换为无符号整数，存储回变量p-4。

8. 最终，通过？:a：免费输出一个整数，表示write_binary函数的执行结果，如果函数失败则返回-1。


```cpp
cleanup:

    mbedtls_pk_free(&pkey);

    _libssh2_mbedtls_safe_free(ntdata, data_len);

    return (*ctx == NULL) ? -1 : 0;
}

static unsigned char *
_libssh2_mbedtls_mpi_write_binary(unsigned char *buf,
                                  const mbedtls_mpi *mpi,
                                  size_t bytes)
{
    unsigned char *p = buf;

    if(sizeof(&p) / sizeof(p[0]) < 4) {
        goto done;
    }

    p += 4;
    *p = 0;

    if(bytes > 0) {
        mbedtls_mpi_write_binary(mpi, p + 1, bytes - 1);
    }

    if(bytes > 0 && !(*(p + 1) & 0x80)) {
        memmove(p, p + 1, --bytes);
    }

    _libssh2_htonu32(p - 4, bytes);

```

This function appears to be part of the SSH2 protocol, specifically the ECDSA signature method. It takes as input a session object, an ECDSA context, a hashalabic structure, and is responsible for generating an electronic signature for the specified hash. The signature is then returned in the form of a byte array.


```cpp
done:

    return p + bytes;
}

/* _libssh2_ecdsa_sign
 *
 * Computes the ECDSA signature of a previously-hashed message
 *
 */

int
_libssh2_mbedtls_ecdsa_sign(LIBSSH2_SESSION *session,
                            libssh2_ecdsa_ctx *ctx,
                            const unsigned char *hash,
                            unsigned long hash_len,
                            unsigned char **sign,
                            size_t *sign_len)
{
    size_t r_len, s_len, tmp_sign_len = 0;
    unsigned char *sp, *tmp_sign = NULL;
    mbedtls_mpi pr, ps;

    mbedtls_mpi_init(&pr);
    mbedtls_mpi_init(&ps);

    if(mbedtls_ecdsa_sign(&ctx->grp, &pr, &ps, &ctx->d,
                          hash, hash_len,
                          mbedtls_ctr_drbg_random,
                          &_libssh2_mbedtls_ctr_drbg) != 0)
        goto cleanup;

    r_len = mbedtls_mpi_size(&pr) + 1;
    s_len = mbedtls_mpi_size(&ps) + 1;
    tmp_sign_len = r_len + s_len + 8;

    tmp_sign = LIBSSH2_CALLOC(session, tmp_sign_len);

    if(tmp_sign == NULL)
        goto cleanup;

    sp = tmp_sign;
    sp = _libssh2_mbedtls_mpi_write_binary(sp, &pr, r_len);
    sp = _libssh2_mbedtls_mpi_write_binary(sp, &ps, s_len);

    *sign_len = (size_t)(sp - tmp_sign);

    *sign = LIBSSH2_CALLOC(session, *sign_len);

    if(*sign == NULL)
        goto cleanup;

    memcpy(*sign, tmp_sign, *sign_len);

```

这段代码是一个 C 语言函数，名为 "cleanup"，属于 "libssh2" 库。它的作用是释放之前分配的内存，并检查输入的 signs 是否为空。

具体来说，以下是代码的主要步骤：

1. 释放 mbedtls_mpi_sign 结构，它可能是通过 mbedtls_mpi_create 函数分配的。这个结构可能用于在 MPI 环境中进行信号操作。
2. 释放 mbedtls_mpi_sign_len 变量，这个变量可能用于在 MPI环境中传递签名信息。
3. 从 libssh2_ecdsa_get_curve_type 函数中获取签名的类型，并将其存储到 signs 变量中。
4. 如果 signs 为空，函数返回 -1，这可能意味着签名无效。
5. 函数最终返回 0，表示成功释放资源。


```cpp
cleanup:

    mbedtls_mpi_free(&pr);
    mbedtls_mpi_free(&ps);

    _libssh2_mbedtls_safe_free(tmp_sign, tmp_sign_len);

    return (*sign == NULL) ? -1 : 0;
}

/* _libssh2_ecdsa_get_curve_type
 *
 * returns key curve type that maps to libssh2_curve_type
 *
 */

```

这两函数用于从给定的名义字符串中确定公钥曲线类型并返回相应的 libssh2_curve_type 类型。

_libssh2_mbedtls_ecdsa_curve_type_from_name() 函数接受一个名为 name 的常量，然后将其与给定的四个名义字符串进行比较，以查找匹配的曲线类型。如果找到了匹配的类型，函数将返回该类型，否则返回 -1。

_libssh2_ecdsa_curve_type_from_name() 函数接受一个名为 name 的常量，然后将其与给定的四个名义字符串进行比较，以查找匹配的曲线类型。如果找到了匹配的类型，函数将返回该类型，否则返回 0，表示无法确定该曲线类型。

这两个函数一起使用时，可以从给定的名义字符串中确定公钥曲线类型，并将其存储在 libssh2_curve_type 类型的变量 out_type 中。


```cpp
libssh2_curve_type
_libssh2_mbedtls_ecdsa_get_curve_type(libssh2_ecdsa_ctx *ctx)
{
    return (libssh2_curve_type) ctx->grp.id;
}

/* _libssh2_ecdsa_curve_type_from_name
 *
 * returns 0 for success, key curve type that maps to libssh2_curve_type
 *
 */

int
_libssh2_mbedtls_ecdsa_curve_type_from_name(const char *name,
                                            libssh2_curve_type *out_type)
{
    int ret = 0;
    libssh2_curve_type type;

    if(name == NULL || strlen(name) != 19)
        return -1;

    if(strcmp(name, "ecdsa-sha2-nistp256") == 0)
        type = LIBSSH2_EC_CURVE_NISTP256;
    else if(strcmp(name, "ecdsa-sha2-nistp384") == 0)
        type = LIBSSH2_EC_CURVE_NISTP384;
    else if(strcmp(name, "ecdsa-sha2-nistp521") == 0)
        type = LIBSSH2_EC_CURVE_NISTP521;
    else {
        ret = -1;
    }

    if(ret == 0 && out_type) {
        *out_type = type;
    }

    return ret;
}

```

这段代码定义了一个名为`_libssh2_mbedtls_ecdsa_free`的函数，属于`libssh2_ecdsa_lib`库。它的作用是释放`libssh2_ecdsa_ctx`类型的内存。

首先，函数内部调用一个名为`mbedtls_ecdsa_free`的函数，这个函数释放`mbedtls_ecdsa`类型的内存。然后，函数内部再调用一个名为`mbedtls_free`的函数，这个函数释放`mbedtls`类型的内存。这两个函数分别从包含在`libssh2_mbedtls_ecdsa_lib`库中的函数表和`mbedtls`库中获取。

总的来说，这段代码的作用是释放`libssh2_ecdsa_ctx`类型的内存，它属于`libssh2_ecdsa_lib`库。


```cpp
void
_libssh2_mbedtls_ecdsa_free(libssh2_ecdsa_ctx *ctx)
{
    mbedtls_ecdsa_free(ctx);
    mbedtls_free(ctx);
}

#endif /* LIBSSH2_ECDSA */
#endif /* LIBSSH2_MBEDTLS */

```