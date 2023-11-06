# Nmap源码解析 91

# `libssh2/os400/ccsid.c`

Hello! How can I assist you today?


```cpp
/*
 * Copyright (C) 2015 Patrick Monnerat, D+H <patrick.monnerat@dh.com>
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

这段代码定义了两个函数，其主要作用是封装字符编码的封装函数。

函数1：`void libssh2_putenv_utf8(const char *env, const char *value)`
该函数接受两个参数，第一个参数是一个环境名称，第二个参数是一个值，该值使用UTF-8编码。它通过执行下列操作来将该值设置为给定环境中的所有字段的UTF-8编码：

1. 将`value`中的所有字符转换为UTF-8编码。
2. 将`value`中的所有字符转换为正字符字符数组。
3. 将上述步骤得到的正字符字符数组转换为环境名称。
4. 递归调用`libssh2_putenv_utf8`函数，将`value`作为新环境中的参数。

函数2：`void libssh2_putenv_utf16be(const char *env, const char *value)`
与函数1类似，只是使用了UTF-16BE编码。函数1将值`value`中的所有字符转换为UTF-16BE编码，返回一个环境名称，值`value`保留原始编码。


```cpp
/* Character encoding wrappers. */

#include "libssh2_priv.h"
#include "libssh2_ccsid.h"

#include <qtqiconv.h>
#include <iconv.h>
#include <errno.h>
#include <string.h>
#include <stdio.h>



#define CCSID_UTF8      1208
#define CCSID_UTF16BE   13488
```

这段代码定义了一些宏，包括：

- #define STRING_GRANULE  256: 定义了字符串粒度(即字符数组中可以存放的最大字符数)，这个值是256。
- #define MAX_CHAR_SIZE   4: 定义了想在字符串中使用最大4个字符的命名规则，这个值也是4。
- #define OFFSET_OF(t, f) ((size_t) ((char *) &((t *) 0)->f - (char *) 0)): 定义了一个名为OFFSET_OF的函数，接收两个参数t和f，函数实现是将字符t转换为字符串类型，然后计算其偏移量至字符串f的偏移量，这个偏移量是由字符t的下一个字符开始计算的。

- #define MAX_LENGTH    16: 定义了一个名为MAX_LENGTH的常量，其值为16。
- #define MAX_ATTRIBUTES  32: 定义了一个名为MAX_ATTRIBUTES的常量，其值为32。
- #define MAX_MESSAGE    256: 定义了一个名为MAX_MESSAGE的常量，其值为256。

- 还定义了一个名为utf8code的QtqCode类型的变量，其值为：

 { "utf-8": "\\u0-9\\u0-9\\u0-9\\u0-9" }

- 在一个名为libssh2_string_cache的结构体中，定义了一个名为next的成员变量，类型为libssh2_string_cache *，这个结构体还有名为string的成员变量，类型为char。

- 在一个名为utf8 的函数内部，定义了一个名为utf8 的宏，其值为 "\\u0-9\\u0-9\\u0-9\\u0-9" 。

- 在一个名为ssh2_string_cache_init的函数内部，定义了一个名为init 的函数，函数内部使用形参t来初始化字符串缓存器对象，并且使用形参f来指定要初始化的字符串的文件名。函数内部使用OFFSET_OF函数计算字符串中指定的偏移量，然后将计算得到的偏移量存回给形参t所指向的字符数组。函数内部还使用宏MAX_LENGTH和MAX_ATTRIBUTES来获取最大可用的字符串长度和属性数量。函数内部最终返回字符串缓存器对象，使用这个对象来存储字符串。

- 在一个名为ssh2_string_cache_handle的函数内部，定义了一个名为handle 的函数，函数内部接收一个字符串和一个size_t类型的参数来表示要操作的字节数，函数内部使用宏MAX_ATTRIBUTES和MAX_MESSAGE来获取可以设置的最大属性和消息长度，然后使用OFFSET_OF函数计算得到要设置的偏移量，最后将计算得到的偏移量存回给形参t所指向的字符数组。函数内部还使用宏UTF8code来将字符串转换为utf8编码，然后使用这个编码来访问字符串中的数据。函数内部最终返回一个bool类型的值，表示是否成功设置字符串。

- 在一个名为main的函数内部，定义了一个名为string_cache_init的函数，函数内部接收一个指向char类型的指针变量t和一个字符串类型的形参f，函数内部使用OFFSET_OF函数计算出字符串中指定的偏移量，然后将计算得到的偏移量存回给形参t所指向的字符数组。函数内部还使用宏MAX_LENGTH和MAX_ATTRIBUTES来获取最大可用的字符串长度和属性数量。函数内部最终返回string_cache_init返回一个void类型的值，表示成功初始化字符串缓存器。

- 在主函数中，定义了一个名为string_cache的


```cpp
#define STRING_GRANULE  256
#define MAX_CHAR_SIZE   4

#define OFFSET_OF(t, f) ((size_t) ((char *) &((t *) 0)->f - (char *) 0))


struct _libssh2_string_cache {
    libssh2_string_cache *  next;
    char                    string[1];
};


static const QtqCode_T  utf8code = { CCSID_UTF8 };


```

这段代码是一个名为“terminator_size”的函数，属于“sizes”家族，用于返回某个特定编码的终端字符串长度。函数接受一个无符号短整型参数“ccsid”，表示特定的编码服务ID（CCSID）。

具体来说，这段代码执行以下操作：

1. 检查给定的CCSID属于哪种编码服务，如果是UTF-8或0，则直接返回1；如果是UTF-16BE，则返回2。
2. 如果CCSID不正确，执行以下操作：
  1. 将生成的无符号字节码（outcode）初始化为零；
  2. 使用QtqIconvOpen函数将UTF-8编码转换为给定的CCSID，其中使用了一个名为“utf8code”的QtqCode；
  3. 如果转换失败，返回-1；
  4. 检查生成的字节码是否正确，如果是，则返回其长度；
  5. 如果不是，尝试使用iconv函数将其转换为目标编码，其中使用了“cd”表示输入编码，并保存结果于“outp”；
  6. 如果成功，使用iconvClose函数关闭输入编码，然后根据目标编码调整输出缓冲区大小；
  7. 最后返回输出缓冲区长度，如果成功则返回其值，否则返回-1。

这段代码定义了一个名为“terminator_size”的函数，接受一个无符号短整型参数“ccsid”，用于返回指定编码的终端字符串长度。函数根据给定的CCSID属于哪种编码服务进行不同的处理，如果无法正确处理，则返回-1。


```cpp
static ssize_t
terminator_size(unsigned short ccsid)
{
    QtqCode_T outcode;
    iconv_t cd;
    char *inp;
    char *outp;
    size_t ilen;
    size_t olen;
    char buf[MAX_CHAR_SIZE];

    /* Return the null-terminator size for the given CCSID. */

    /* Fast check usual CCSIDs. */
    switch (ccsid) {
    case CCSID_UTF8:
    case 0:                                 /* Job CCSID is SBCS EBCDIC. */
        return 1;
    case CCSID_UTF16BE:
        return 2;
    }

    /* Convert an UTF-8 NUL to the target CCSID: use the converted size as
       result. */
    memset((void *) &outcode, 0, sizeof outcode);
    outcode.CCSID = ccsid;
    cd = QtqIconvOpen(&outcode, (QtqCode_T *) &utf8code);
    if (cd.return_value == -1)
        return -1;
    inp = "";
    ilen = 1;
    outp = buf;
    olen = sizeof buf;
    iconv(cd, &inp, &ilen, &outp, &olen);
    iconv_close(cd);
    olen = sizeof buf - olen;
    return olen? olen: -1;
}

```

以下是升级版 man 文件中的实现：

```cpp
man string_convert.h
```

## string_convert.h

该文件定义了一个名为 string_convert.h 的函数，该函数实现了字符串转换为另一个字符串的底层操作。以下是该函数的实现细节：

```cpp
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <iconv.h>
#include <libssh2.h>
#include "libssh2_strings.h"
```

该函数的参数包括源字符串(source)、目标字符串(destination)、源字符串的长度(len source)和目标字符串的字节数(len dest)。函数内部首先定义了一个名为 outp 的指针变量，用于存储目标字符串的起始位置。然后，函数内部定义了一个名为 curlen 的整数变量，用于存储源字符串中除去目标字符串外的字符数量。接着，函数内部使用 iconv() 函数将源字符串转换为目标字符串，并从源字符串中读取字符，存储在 outp 中。如果转换成功，函数内部将 curlen 减去一个字节数，用于存储目标字符串中的字符数量。如果转换失败，函数内部将输出空字符串并返回。函数内部还定义了一个名为 i 的整数变量，用于存储从 input 中读取的字符数量。如果 input 文件不存在或无法打开，函数内部将使用一个固定长度的空字符串作为目标字符串，并使用全部从 input 中读取的字符来填充该字符串。最后，函数内部使用 libssh2_realloc() 函数和 libssh2_string_cache 函数来管理源字符串和目标字符串，并使用 outstring->next 和 libssh2_string_cache 指针变量来获取和设置这些指针变量。

```cpp
int string_convert(const char *source, char *dest, int len dest)
{
   int ret = 0;
   int inlen = 0, outlen = 0;
   const char *input = source;
   const char *output = dest;
   while ((inlen = libssh2_string_get_length(input)) >= 0) {
       int c = libssh2_read_char(input, &outlen);
       if (c == -1) {
           ret = -1;
           break;
       }
       outp = LIBSSH2_REALLOC(session, output, outlen + 1);
       if (!outp) {
           ret = -2;
           break;
       }
       libssh2_string_add(session, input, c, outp, &outlen);
       inlen = outlen - 1;
   }
   if (inlen < 0) {
       ret = -1;
   }
   return ret;
}
```

```cpp
man string_convert_with_realloc.h
```

## string_convert_with_realloc.h

该文件定义了一个名为 string_convert_with_realloc.h 的函数，该函数实现了将一个字符串从一个源字符串转换为另一个字符串，并允许在转换后自动释放内存。

```cpp
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <iconv.h>
#include <libssh2.h>
#include "libssh2_strings.h"
```

该函数的参数包括源字符串(source)、目标字符串(destination)、源字符串的长度(len source)和目标字符串的字节数(len dest)。函数内部首先定义了一个名为 outp 的指针变量，用于存储目标字符串的起始位置。然后，函数内部定义了一个名为 curlen 的整数变量，用于存储源字符串中除去目标字符串外的字符数量。接着，函数内部使用 iconv() 函数将源字符串转换为目标字符串，并从源字符串中读取字符，存储在 outp 中。如果转换成功，函数内部将 curlen 减去一个字节数，用于存储目标字符串中的字符数量。如果转换失败，函数内部将输出空字符串并返回。函数内部还定义了一个名为 i 的整数变量，用于存储从 input 中读取的字符数量。如果 input 文件不存在或无法打开，函数内部将使用一个固定长度的空字符串作为目标字符串，并使用全部从 input 中读取的字符来填充该字符串。最后，函数内部使用 libssh2_realloc() 函数和 libssh2_string_cache 函数来管理源字符串和目标字符串，并使用 outstring->next 和 libssh2_string_cache 指针变量来获取和设置这些指针变量。函数内部还定义了一个名为 malloc_with_free 的函数，用于管理函数内部的内存。

```cpp
int string_convert_with_realloc(const char *source, char *dest, int len dest)
{
   int ret = 0;
   int inlen = 0, outlen = 0;
   const char *input = source;
   const char *output = dest;
   int freed_ memory = 0;
   const char *temp_ptr;
   while ((inlen = libssh2_string_get_length(input)) >= 0) {
       temp_ptr = LIBSSH2_REALLOC(session, output, outlen + 1);
       if (!temp_ptr) {
           ret = -1;
           break;
       }
       outp = LIBSSH2_REALLOC(session, input, input + inlen);
       if (!outp) {
           ret = -2;
           break;
       }
       libssh2_string_add(session, input, c, outp, &outlen);
       inlen = outlen - 1;
       freed_ memory++;
   }
   if (inlen < 0) {
       ret = -1;
   }
   return ret;
}
```

这两个函数的实现要点如下：

1. `string_convert` 函数：

- `ret` 变量表示转换是否成功，初始值为 `0`。
- `inlen` 变量表示从输入源字符串中读取的字符数量，初始值为 0。
- `outlen` 变量表示目标字符串的长度，初始值为 0。
- `c` 表示从输入源字符串中读取的字符


```cpp
static char *
convert_ccsid(LIBSSH2_SESSION *session, libssh2_string_cache **cache,
              unsigned short outccsid, unsigned short inccsid,
              const char *instring, ssize_t inlen, size_t *outlen)
{
    char *inp;
    char *outp;
    size_t olen;
    size_t ilen;
    size_t buflen;
    size_t curlen;
    ssize_t termsize;
    int i;
    char *dst;
    libssh2_string_cache *outstring;
    QtqCode_T incode;
    QtqCode_T outcode;
    iconv_t cd;

    if (!instring) {
        if (outlen)
            *outlen = 0;
        return NULL;
    }
    if (outlen)
        *outlen = -1;
    if (!session || !cache)
        return NULL;

    /* Get terminator size. */
    termsize = terminator_size(outccsid);
    if (termsize < 0)
        return NULL;
 
    /* Prepare conversion parameters. */
    memset((void *) &incode, 0, sizeof incode);
    memset((void *) &outcode, 0, sizeof outcode);
    incode.CCSID = inccsid;
    outcode.CCSID = outccsid;
    curlen = OFFSET_OF(libssh2_string_cache, string);
    inp = (char *) instring;
    ilen = inlen;
    buflen = inlen + curlen;
    if (inlen < 0) {
        incode.length_option = 1;
        buflen = STRING_GRANULE;
        ilen = 0;
    }

    /* Allocate output string buffer and open conversion descriptor. */
    dst = LIBSSH2_ALLOC(session, buflen + termsize);
    if (!dst)
        return NULL;
    cd = QtqIconvOpen(&outcode, &incode);
    if (cd.return_value == -1) {
        LIBSSH2_FREE(session, (char *) dst);
        return NULL;
    }

    /* Convert string. */
    for (;;) {
        outp = dst + curlen;
        olen = buflen - curlen;
        i = iconv(cd, &inp, &ilen, &outp, &olen);
        if (inlen < 0 && olen == buflen - curlen) {
            /* Special case: converted 0-length (sub)strings do not store the
               terminator. */
            if (termsize) {
                memset(outp, 0, termsize);
                olen -= termsize;
            }
        }
        curlen = buflen - olen;
        if (i >= 0 || errno != E2BIG)
            break;
        /* Must expand buffer. */
        buflen += STRING_GRANULE;
        outp = LIBSSH2_REALLOC(session, dst, buflen + termsize);
        if (!outp)
            break;
        dst = outp;
    }

    iconv_close(cd);

    /* Check for error. */
    if (i < 0 || !outp) {
        LIBSSH2_FREE(session, dst);
        return NULL;
    }

    /* Process terminator. */
    if (inlen < 0)
        curlen -= termsize;
    else if (termsize)
        memset(dst + curlen, 0, termsize);

    /* Shorten buffer if possible. */
    if (curlen < buflen)
        dst = LIBSSH2_REALLOC(session, dst, curlen + termsize);

    /* Link to cache. */
    outstring = (libssh2_string_cache *) dst;
    outstring->next = *cache;
    *cache = outstring;

    /* Return length if required. */
    if (outlen)
        *outlen = curlen - OFFSET_OF(libssh2_string_cache, string);

    return outstring->string;
}

```

这两函数是用于将字符串 ccsid 的 UTF-8 编码字符串转换为 CCSID 编码的函数。

在 `libssh2_from_ccsid` 函数中，输入参数 ccsid 是一个 unsigned short 类型的输入参数，表示 UTF-8 编码的字符串。函数的返回值是一个指向字符串常量池中存储的 CCSID 编码字符串的指针。

在 `libssh2_to_ccsid` 函数中，输入参数 ccsid 是一个 unsigned short 类型的输入参数，表示需要转换为 UTF-8 编码的字符串。函数的返回值是一个指向字符串常量池中存储的 CCSID 编码字符串的指针。


```cpp
LIBSSH2_API char *
libssh2_from_ccsid(LIBSSH2_SESSION *session, libssh2_string_cache **cache,
                   unsigned short ccsid, const char *string, ssize_t inlen,
                   size_t *outlen)
{
    return convert_ccsid(session, cache,
                         CCSID_UTF8, ccsid, string, inlen, outlen);
}

LIBSSH2_API char *
libssh2_to_ccsid(LIBSSH2_SESSION *session, libssh2_string_cache **cache,
                 unsigned short ccsid, const char *string, ssize_t inlen,
                 size_t *outlen)
{
    return convert_ccsid(session, cache,
                         ccsid, CCSID_UTF8, string, inlen, outlen);
}

```

这段代码定义了一个名为`libssh2_release_string_cache`的函数，属于`libssh2_api`类别，用于释放由`libssh2_string_cache`结构体组成的链表。

具体来说，这个函数接收两个参数：一个`LIBSSH2_SESSION`类型的`session`表示当前会话，另一个是一个指向`libssh2_string_cache`结构体的指针`cache`，这个结构体存储了所有链表中的节点。函数首先检查`session`和`cache`是否都为有效参数，如果是，就从`cache`指向的链表中逐个释放节点，即将每个节点从`cache`链表中移动到`LIBSSH2_FREE`函数分配的内存空间中，然后将`cache`指向的指针递增，以便下一个释放节点时可以正确访问。

总的来说，这个函数的主要作用是释放由`libssh2_string_cache`结构体组成的链表，使得当前会话结束后不再留下任何链表对象的内存泄漏。


```cpp
LIBSSH2_API void
libssh2_release_string_cache(LIBSSH2_SESSION *session,
                             libssh2_string_cache **cache)
{
    libssh2_string_cache *p;

    if (session && cache)
        while ((p = *cache)) {
            *cache = p->next;
            LIBSSH2_FREE(session, (char *) p);
        }
}

/* vim: set expandtab ts=4 sw=4: */

```

# `libssh2/os400/libssh2_ccsid.h`

Hello! How can I assist you today?


```cpp
/*
 * Copyright (C) 2015 Patrick Monnerat, D+H <patrick.monnerat@dh.com>
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

这段代码定义了一个名为 libssh2_ccsid 的函数，它的作用是支持从给定 CCSID 将字符串转换为 libssh2 协议支持的字符串类型。

具体来说，这段代码包含以下几行：

1.定义了一个名为 libssh2_string_cache 的结构体，它是一个 libssh2_string_cache 类型的指针，用于保存转换后的字符串。

2.定义了一个名为 libssh2_from_ccsid 的函数，它的作用是接收一个 libssh2_session 类型的会话，一个 libssh2_string_cache 类型的指针，以及要转换的 CCSID、要转换的字符串，还有转换后要保存到的输出字符串长度。这个函数首先将 CCSID 转换为 libssh2 协议支持的字符串类型，然后使用 libssh2_string_cache 中的函数将字符串转换为 CCSID 类型，最后将转换后的字符串与输出字符串进行比较，如果相等，则返回转换后的字符串；否则，返回原始字符串。

3.没有定义函数体，因此在代码中没有实现函数的具体行为。


```cpp
/* CCSID conversion support. */

#ifndef LIBSSH2_CCSID_H_
#define LIBSSH2_CCSID_H_

#include "libssh2.h"

typedef struct _libssh2_string_cache    libssh2_string_cache;


LIBSSH2_API char *
libssh2_from_ccsid(LIBSSH2_SESSION *session, libssh2_string_cache **cache,
                   unsigned short ccsid, const char *string, ssize_t inlen,
                   size_t *outlen);
LIBSSH2_API char *
```

这两段代码定义了两个函数，分别是`libssh2_to_ccsid`和`libssh2_release_string_cache`。它们的作用如下：

1. `libssh2_to_ccsid`函数的输入参数是一个`LIBSSH2_SESSION`指针、一个指向`libssh2_string_cache`的指针和一个字符串，表示要转换的字符串的编码类型为`ccsid`，目标编码类型为`string`，转换后的字符串长度为`inlen`，目标字符串长度为`outlen`。这个函数的作用是将输入的字符串编码类型的`ccsid`转换为`string`编码类型，并返回转换后的字符串。

2. `libssh2_release_string_cache`函数的输入参数是一个`LIBSSH2_SESSION`指针和一个指向`libssh2_string_cache`的指针，表示要释放的内存区域是这两个指针指向的内存。这个函数的作用是释放传入的`libssh2_string_cache`结构中的内存，并返回指向这个内存的指针。


```cpp
libssh2_to_ccsid(LIBSSH2_SESSION *session, libssh2_string_cache **cache,
                 unsigned short ccsid, const char *string, ssize_t inlen,
                 size_t *outlen);
LIBSSH2_API void
libssh2_release_string_cache(LIBSSH2_SESSION *session,
                             libssh2_string_cache **cache);

#endif

/* vim: set expandtab ts=4 sw=4: */

```

# `libssh2/os400/libssh2_config.h`

This is a text-based knowledge base starting point for a language model that can answer questions about its construction or any other relevant information. The model is not intended for legal or technical advice and should not be used as such. Please note that any text or data in this knowledge base is provided for informational purposes only and should not be considered legal or professional advice.

In addition, please note that while this text-based knowledge base is a work in progress, it is not complete or fully tested. It may not work correctly or may contain errors. It is provided as-is and should be used at your own risk.

If you have any specific questions or need further assistance, please let me know.


```cpp
/*
 * Copyright (C) 2015 Patrick Monnerat, D+H <patrick.monnerat@dh.com>
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

这段代码定义了一些预处理指令，用于定义和检查目标系统是否支持`alloca`函数。如果目标系统不支持`alloca`，则该函数将无法使用。

具体来说，该代码定义了以下三个定义：

1. `#undef AC_APPLE_UNIVERSAL_BUILD`：如果构建的是 universal库，则该定义表示该库是否支持`AC_APPLE_UNIVERSAL_BUILD`预处理指令。
2. `#undef CRAY_STACKSEG_END`：定义为`_getb67`，`GETB67`或`getb67'`。这个函数是Cray-2和Cray-YMP系统需要的，用于支持`alloca.c'`的定义。
3. `#undef C_ALLOCA`：定义为1，表示是否支持`alloca`函数或宏。

该代码的作用是定义了哪些预处理指令，以确保构建的代码能够在不同操作系统下正确运行，即使目标系统不支持`alloca`函数。


```cpp
#ifndef LIBSSH2_CONFIG_H
#define LIBSSH2_CONFIG_H

/* Define if building universal (internal helper macro) */
#undef AC_APPLE_UNIVERSAL_BUILD

/* Define to one of `_getb67', `GETB67', `getb67' for Cray-2 and Cray-YMP
   systems. This function is required for `alloca.c' support on those systems.
*/
#undef CRAY_STACKSEG_END

/* Define to 1 if using `alloca.c'. */
#undef C_ALLOCA

/* Define to 1 if you have `alloca', as a function or macro. */
```

这段代码定义了一系列头文件，它们定义了系统需要满足的特定条件，如果缺少任何一个头文件，编译器会报错。

具体来说：

1. `#define HAVE_ALLOCA 1`：定义了`HAVE_ALLOCA_H`为1，表示操作系统支持无锁分配内存，如果缺少头文件`<alloca.h>`，则编译器会报错。

2. `#define HAVE_ARPA_INET_H 1`：定义了`HAVE_ARPA_INET_H`为1，表示操作系统支持无状态的Internet套接字，如果缺少头文件`<arpa/inet.h>`，则编译器会报错。

3. `#undef HAVE_DECL_SECUREZEROMEMORY`：定义了`HAVE_DECL_SECUREZEROMEMORY`为1，表示系统支持`SecureZeroMemory`，如果缺少头文件`<alloca.h>`，则编译器会报错。

4. `#undef HAVE_DISABLED_NONBLOCKING`：定义了`HAVE_DISABLED_NONBLOCKING`为1，表示系统支持非阻塞I/O，如果缺少头文件`<sys/types.h>`，则编译器会报错。

如果缺少任何一个头文件，编译器会报错，因此这段代码会定义一些必需的头文件，以保证程序在特定操作系统和硬件上能够正常运行。


```cpp
#define HAVE_ALLOCA 1

/* Define to 1 if you have <alloca.h> and it should be used (not on Ultrix). */
#define HAVE_ALLOCA_H 1

/* Define to 1 if you have the <arpa/inet.h> header file. */
#define HAVE_ARPA_INET_H 1

/* Define to 1 if you have the declaration of `SecureZeroMemory', and to 0 if
   you don't. */
#undef HAVE_DECL_SECUREZEROMEMORY

/* disabled non-blocking sockets */
#undef HAVE_DISABLED_NONBLOCKING

```

这段代码定义了多个宏，用于检查系统头文件中是否定义了相应的函数。如果定义了，则返回1，否则返回0。这些宏如下：

```cpp
#undef HAVE_DLFCN_H
#define HAVE_ERRNO_H 1
#undef HAVE_EVP_AES_128_CTR
#define HAVE_FCNTL_H 1
```

第一个宏 `HAVE_DLFCN_H` 用于检查是否定义了 `dlfcn.h` 头文件，如果定义了，则该宏返回1，否则返回0。

第二个宏 `HAVE_ERRNO_H` 用于检查是否定义了 `errno.h` 头文件，如果定义了，则该宏返回1，否则返回0。

第三个宏 `HAVE_EVP_AES_128_CTR` 用于检查是否定义了 `EVP_aes_128_ctr` 函数，如果定义了，则该宏返回1，否则返回0。

第四个宏 `HAVE_FCNTL_H` 用于检查是否定义了 `fcntl.h` 头文件，如果定义了，则该宏返回1，否则返回0。该宏通过使用 `FIONBIO` 函数来支持非阻塞套接字。


```cpp
/* Define to 1 if you have the <dlfcn.h> header file. */
#undef HAVE_DLFCN_H

/* Define to 1 if you have the <errno.h> header file. */
#define HAVE_ERRNO_H 1

/* Define to 1 if you have the `EVP_aes_128_ctr' function. */
#undef HAVE_EVP_AES_128_CTR

/* Define to 1 if you have the <fcntl.h> header file. */
#define HAVE_FCNTL_H 1

/* use FIONBIO for non-blocking sockets */
#undef HAVE_FIONBIO

```

这段代码定义了一些宏，用于检查是否支持不同的功能。以下是每个宏的上下文和含义：

```cpp
#define HAVE_GETTIMEOFDAY 1

#define HAVE_INTTYPES_H 1

/* use ioctlsocket() for non-blocking sockets */
#undef HAVE_IOCTLSOCKET

/* use Ioctlsocket() for non-blocking sockets */
#undef HAVE_IOCTLSOCKET_CASE

/* Define if you have the bcrypt library. */
#undef HAVE_LIBBCRYPT
```

这些宏根据定义是否包含特定的库或头文件进行判断，并为该库或头文件添加了注释。如果定义了宏，则说明该库或头文件已经被引入或定义。否则，将不输出任何信息，表明对应的库或头文件不存在。


```cpp
/* Define to 1 if you have the `gettimeofday' function. */
#define HAVE_GETTIMEOFDAY 1

/* Define to 1 if you have the <inttypes.h> header file. */
#define HAVE_INTTYPES_H 1

/* use ioctlsocket() for non-blocking sockets */
#undef HAVE_IOCTLSOCKET

/* use Ioctlsocket() for non-blocking sockets */
#undef HAVE_IOCTLSOCKET_CASE

/* Define if you have the bcrypt library. */
#undef HAVE_LIBBCRYPT

```

这段代码定义了一些编译器特定的变量，用于检查用户是否安装了crypt32，libgcrpt和ssl库。如果其中任何一个库没有安装，那么定义为0。

同时，它还定义了一个名为HAVE_LONGLONG的变量，用于表示编译器是否支持long long数据类型。如果定义为1，则表示编译器支持long long数据类型。

最后，它通过#undef重置了HAVE_LIBCRYPT32，HAVE_LIBGCRYPT和HAVE_LIBSSL这三个变量，这意味着在编译时它们都将被视为无定义。


```cpp
/* Define if you have the crypt32 library. */
#undef HAVE_LIBCRYPT32

/* Define if you have the gcrypt library. */
#undef HAVE_LIBGCRYPT

/* Define if you have the ssl library. */
#undef HAVE_LIBSSL

/* Define if you have the z library. */
/* #undef HAVE_LIBZ */

/* Define to 1 if the compiler supports the 'long long' data type. */
#define HAVE_LONGLONG 1

```

这段代码定义了多个宏，用于检查系统头文件中是否定义了相应的头文件。如果头文件存在，则这些宏的值被定义为1，否则为0。这些宏的作用是用来检查系统是否支持指定的头文件，以便在程序开发过程中提高代码的可移植性和兼容性。

具体来说，这段代码定义了以下四个宏：

1.Hashtmp( )：定义为1表示成功定义了<memory.h>头文件，否则为0。

2.H业netinet( )：定义为1表示成功定义了<netinet/in.h>头文件，否则为0。

3.Hntdef( )：定义为1表示成功定义了<ntdef.h>头文件，否则为0。

4.Hntstatus( )：定义为1表示成功定义了<ntstatus.h>头文件，否则为0。

5.O0非块( )：定义为1表示成功定义了O_NONBLOCK头文件，以便在非阻塞套接字中使用。


```cpp
/* Define to 1 if you have the <memory.h> header file. */
#undef HAVE_MEMORY_H

/* Define to 1 if you have the <netinet/in.h> header file. */
#define HAVE_NETINET_IN_H 1

/* Define to 1 if you have the <ntdef.h> header file. */
#undef HAVE_NTDEF_H

/* Define to 1 if you have the <ntstatus.h> header file. */
#undef HAVE_NTSTATUS_H

/* use O_NONBLOCK for non-blocking sockets */
#define HAVE_O_NONBLOCK 1

```

这段代码定义了一些宏，用于检查系统是否支持特定的函数或库。如果系统支持指定的函数或库，则定义为1，否则定义为0。这些宏的定义如下：
```cpp
#define DO_POLL              1
#define DO_SELECT            1
#define SO_NONBLOCK            1
#define DO_SO_NONBLOCK        0
#define HAS_STDINT_H        1
#define HAS_STDIO_H        1
```
首先定义了一些符号常量，然后在其他部分使用它们。这些符号常量被定义为1时，表示函数或库存在，否则为0。接下来定义了一些宏，用于在编译时检查系统是否支持指定的函数或库。如果系统支持指定的函数或库，则定义为1，否则定义为0。


```cpp
/* Define to 1 if you have the `poll' function. */
#undef HAVE_POLL

/* Define to 1 if you have the select function. */
#define HAVE_SELECT 1

/* use SO_NONBLOCK for non-blocking sockets */
#undef HAVE_SO_NONBLOCK

/* Define to 1 if you have the <stdint.h> header file. */
#define HAVE_STDINT_H 1

/* Define to 1 if you have the <stdio.h> header file. */
#define HAVE_STDIO_H 1

```

这段代码定义了多个宏，用于检查系统头文件中是否定义了相应的函数。如果头文件中定义了这些函数，则对应的宏的值为1，否则为0。

具体来说，这些宏的含义如下：

1. CANNOT_BE_DETERMINATED：如果定义了<stdlib.h>头文件，但是没有定义`哈希查找函数`或者`串查找函数`，那么这个宏的含义是未定义的。
2. HAS_STANDARD_LIBRARY：如果定义了`<stdlib.h>`头文件，并且定义了`哈希查找函数`或者`串查找函数`，那么这个宏的含义是已定义的。
3. HAS_STRINGS_H：如果定义了`<strings.h>`头文件，那么这个宏的含义是已定义的。
4. HAS_STRING_H：如果定义了`<string.h>`头文件，那么这个宏的含义是已定义的。
5. HAS_STRTOL_H：如果定义了`<strtol.h>`头文件，那么这个宏的含义是已定义的。
6. HAS_SYS_IOCTL_H：如果定义了`<sys/ioctl.h>`头文件，那么这个宏的含义是已定义的。


```cpp
/* Define to 1 if you have the <stdlib.h> header file. */
#define HAVE_STDLIB_H 1

/* Define to 1 if you have the <strings.h> header file. */
#define HAVE_STRINGS_H 1

/* Define to 1 if you have the <string.h> header file. */
#define HAVE_STRING_H 1

/* Define to 1 if you have the `strtoll' function. */
#define HAVE_STRTOLL 1

/* Define to 1 if you have the <sys/ioctl.h> header file. */
#define HAVE_SYS_IOCTL_H 1

```

这段代码定义了几个宏，根据你是否有特定的头文件，宏的值会被定义为1或0。

具体来说：

- 如果已经定义了<sys/select.h>头文件，宏定义为1，否则定义为0。
- 如果已经定义了<sys/socket.h>头文件，宏定义为1，否则定义为0。
- 如果已经定义了<sys/stat.h>头文件，宏定义为1，否则定义为0。
- 如果已经定义了<sys/time.h>头文件，宏定义为1，否则定义为0。
- 如果已经定义了<sys/types.h>头文件，宏定义为1，否则定义为0。

这些宏的作用是定义了几个宏，根据你是否有特定的头文件，宏的值会被定义为1或0。


```cpp
/* Define to 1 if you have the <sys/select.h> header file. */
#undef HAVE_SYS_SELECT_H

/* Define to 1 if you have the <sys/socket.h> header file. */
#define HAVE_SYS_SOCKET_H 1

/* Define to 1 if you have the <sys/stat.h> header file. */
#define HAVE_SYS_STAT_H 1

/* Define to 1 if you have the <sys/time.h> header file. */
#define HAVE_SYS_TIME_H 1

/* Define to 1 if you have the <sys/types.h> header file. */
#define HAVE_SYS_TYPES_H 1

```

这段代码定义了几个宏，根据是否包含特定的头文件，判断是否支持特定的库函数。

如果引入了<sys/uio.h>头文件，则定义了HAVE_SYS_UIO_H宏，表示支持库函数：uio_ioctl()。
如果引入了<sys/un.h>头文件，则定义了HAVE_SYS_UN_H宏，表示支持库函数：uacll()。
如果引入了<unistd.h>头文件，则定义了HAVE_UNISTD_H宏，表示支持库函数：std::<函数名>()。
如果引入了<windows.h>头文件，则定义了HAVE_WINDOWS_H宏，表示支持库函数：<函数名>()。
如果引入了<winsock2.h>头文件，则定义了HAVE_WINSOCK2_H宏，表示支持库函数：<函数名>()。
如果没有指定任何一个头文件，则定义了HAVE_WINDOWS_H和HAVE_WINSOCK2_H宏，表示都不支持库函数。


```cpp
/* Define to 1 if you have the <sys/uio.h> header file. */
#define HAVE_SYS_UIO_H 1

/* Define to 1 if you have the <sys/un.h> header file. */
#define HAVE_SYS_UN_H 1

/* Define to 1 if you have the <unistd.h> header file. */
#define HAVE_UNISTD_H 1

/* Define to 1 if you have the <windows.h> header file. */
#undef HAVE_WINDOWS_H

/* Define to 1 if you have the <winsock2.h> header file. */
#undef HAVE_WINSOCK2_H

```

这段代码定义了一些预处理指令，以使代码在编译时生成可移植的提示。以下是每个指令的作用：

1. `#define`：定义了一个预处理指令，如果当前头文件中包含`<ws2tcpip.h>`头文件，则将`define`指令的参数设置为1，否则将其设置为0。

2. `#undef`：定义了一个预处理指令，如果当前头文件中包含`<ws2tcpip.h>`头文件，则将`undefine`指令的参数设置为`1`，否则将其设置为`0`。

3. `#define`：定义了一个预处理指令，将`LIBSSH2_API`定义为`1`，使得编译器能够识别该符号。

4. `#undef`：定义了一个预处理指令，将`LIBSSH2_CLEAR_MEMORY`定义为`1`，使得编译器能够识别该符号。

5. `#undef`：定义了一个预处理指令，将`LIBSSH2_CRYPT_NONE`定义为`1`，使得编译器能够识别该符号。

6. `#define`：定义了一个预处理指令，将`LIBSSH2_DH_GEX_NEW`定义为`1`，使得编译器能够识别该符号。

由于每个预处理指令都包含反斜杠`/`，因此这些指令不会对代码产生实际的输出。


```cpp
/* Define to 1 if you have the <ws2tcpip.h> header file. */
#undef HAVE_WS2TCPIP_H

/* to make a symbol visible */
#undef LIBSSH2_API

/* Enable clearing of memory before being freed */
#define LIBSSH2_CLEAR_MEMORY 1

/* Enable "none" cipher -- NOT RECOMMENDED */
#undef LIBSSH2_CRYPT_NONE

/* Enable newer diffie-hellman-group-exchange-sha1 syntax */
#define LIBSSH2_DH_GEX_NEW 1

```

这段代码是一个用于编译 zlib 支持的安全 shell（SSH）库的示例。它实现了以下主要功能：

1. 引入 zlib 库。
2. 定义了 zlib 库的路径。
3. 移除了 LIBSSH2_HAVE_ZLIB，因为当前代码已经引用了 zlib。
4. 引入了 libgcrypt 和 OpenSSL，以便在需要时动态地加载这些库。
5. 禁用了 "none" MAC，因为这不是一个安全的设计。
6. 激活了 Windows CNG。

总之，这段代码允许在 zlib 库存在的情况下使用 OpenSSL 和 Windows CNG，从而可以编译并运行更广泛的 SSH 应用程序。


```cpp
/* Compile in zlib support */
/* #undef LIBSSH2_HAVE_ZLIB */

/* Use libgcrypt */
#undef LIBSSH2_LIBGCRYPT

/* Enable "none" MAC -- NOT RECOMMENDED */
#undef LIBSSH2_MAC_NONE

/* Use OpenSSL */
#undef LIBSSH2_OPENSSL

/* Use Windows CNG */
#undef LIBSSH2_WINCNG

```

这段代码定义了一个名为"libssh2"的软件包，其中包括了SSH2协议的实现。它作用于OS/400系统，并使用了Qc3库。

以下是代码的详细解释：

```cpp
/* Use OS/400 Qc3 */
#define LIBSSH2_OS400QC3

/* Define to the sub-directory in which libtool stores uninstalled libraries. */
#define LT_OBJDIR ".libs/"

/* Define to 1 if _REENTRANT preprocessor symbol must be defined. */
#undef NEED_REENTRANT

/* Name of package */
#define PACKAGE "libssh2"

/* Define to the address where bug reports for this package should be sent. */
#define PACKAGE_BUGREPORT "libssh2-devel@cool.haxx.se"
```

* `LIBSSH2_OS400QC3`是一个预定义宏，表示要使用Qc3库。
* `LT_OBJDIR`是一个宏定义，表示库tool存储未安装库的子目录。
* `NEED_REENTRANT`是一个宏定义，表示要定义_REENTRANT预处理函数，但是实际上没有需要定义它。
* `PACKAGE`是一个宏定义，表示软件包的名称。
* `PACKAGE_BUGREPORT`是一个宏定义，表示软件包的bug报告地址。


```cpp
/* Use OS/400 Qc3 */
#define LIBSSH2_OS400QC3

/* Define to the sub-directory in which libtool stores uninstalled libraries.
*/
#define LT_OBJDIR ".libs/"

/* Define to 1 if _REENTRANT preprocessor symbol must be defined. */
#undef NEED_REENTRANT

/* Name of package */
#define PACKAGE "libssh2"

/* Define to the address where bug reports for this package should be sent. */
#define PACKAGE_BUGREPORT "libssh2-devel@cool.haxx.se"

```

这段代码定义了一系列头文件，包括：

1. `#define PACKAGE_NAME "libssh2"`：定义了一个名为`PACKAGE_NAME`的宏，将`libssh2`作为它的值。这个宏定义了该包的名称。

2. `#define PACKAGE_STRING "libssh2 -"`：定义了一个名为`PACKAGE_STRING`的宏，将`libssh2`作为它的值。这个宏定义了该包的文本描述符。

3. `#define PACKAGE_TARNAME "libssh2"`：定义了一个名为`PACKAGE_TARNAME`的宏，将`libssh2`作为它的值。这个宏定义了该包的简称。

4. `#define PACKAGE_URL ""`：定义了一个名为`PACKAGE_URL`的宏，将` ""`作为它的值。这个宏定义了该包的URL，也就是网站主页。

5. `#define PACKAGE_VERSION "-"`：定义了一个名为`PACKAGE_VERSION`的宏，将`-`作为它的值。这个宏定义了该包的版本号。

这些宏定义了该包的各种元素，包括名称、描述、简称、URL和版本号。这些元素在不同的上下文中被使用，例如在编译器和终端应用程序中的人类可读的名称。


```cpp
/* Define to the full name of this package. */
#define PACKAGE_NAME "libssh2"

/* Define to the full name and version of this package. */
#define PACKAGE_STRING "libssh2 -"

/* Define to the one symbol short name of this package. */
#define PACKAGE_TARNAME "libssh2"

/* Define to the home page for this package. */
#define PACKAGE_URL ""

/* Define to the version of this package. */
#define PACKAGE_VERSION "-"

```

这段代码定义了一个头文件 STACK_DIRECTION，用于描述栈增长的方向。它包含两个条件判断，判断系统是否使用的是栈全貌实现，如果是，则使用 STACK_DIRECTION > 0 表示栈将 grows toward higher 地址；如果是，则使用 STACK_DIRECTION < 0 表示栈将 grows toward lower 地址；如果是，则使用 STACK_DIRECTION = 0 来定义栈增长的方向未知。

接下来的两行代码定义了两个宏定义 STDC_HEADERS 和 VERSION，用于表示 C 语言标准库是否包含头文件 STL。如果是，则定义 STDC_HEADERS = 1，否则定义 STDC_HEADERS = 0。

接下来一行代码定义了一个名为 ANSI_C_HEADERS 的宏，用于表示是否包含 ANSI C 标准库。如果是，则定义 ANSI_C_HEADERS = 1，否则定义 ANSI_C_HEADERS = 0。

最后一行代码定义了一个名为 WORDS_BIGENDIAN 的宏，用于表示系统是否使用的是最具有代表性的数据类型。如果是系统支持多字节字符串（如 Motorola 和 SPARC），则定义 WORDS_BIGENDIAN = 1；否则定义 WORDS_BIGENDIAN = 0。


```cpp
/* If using the C implementation of alloca, define if you know the
   direction of stack growth for your system; otherwise it will be
   automatically deduced at runtime.
	STACK_DIRECTION > 0 => grows toward higher addresses
	STACK_DIRECTION < 0 => grows toward lower addresses
	STACK_DIRECTION = 0 => direction of growth unknown */
#undef STACK_DIRECTION

/* Define to 1 if you have the ANSI C header files. */
#define STDC_HEADERS 1

/* Version number of package */
#define VERSION "-"

/* Define WORDS_BIGENDIAN to 1 if your processor stores words with the most
   significant byte first (like Motorola and SPARC, unlike Intel). */
```

这段代码定义了一些预处理指令，用于在源代码中更方便地使用特定的 macOS 操作系统特性。以下是每个指令的作用：

1. #define WORDS_BIGENDIAN 1

这是一个定义，表示为宏，后面会详细解释。

1. /* Enable large inode numbers on Mac OS X 10.5.  */

这是一个注释，告诉编译器在 Mac OS X 10.5 上启用大的 inode 数。

1. #ifndef _DARWIN_USE_64_BIT_INODE
# define _DARWIN_USE_64_BIT_INODE 1
#endif

这是一个带井井头符号的注释，表示这是一个以 #include 开头的源代码块。后面会详细解释。

1. /* Number of bits in a file offset, on hosts where this is settable. */

这是一个定义，定义了一个名为 _FILE_OFFSET_BITS 的整数，表示文件偏移中可设置比特数的位数。

1. /* Define for large files, on AIX-style hosts. */

这是一个定义，定义了一个名为 _LARGE_FILES 的整数，表示在 AIX 风格的主机上是否适用于大型文件。

1. /* Define to empty if `const' does not conform to ANSI C. */

这是一个定义，定义了一个名为 const 的整数，表示如果 `const'` 不符合 ANSI C 的规范，则该整数为 `0`。

总之，这段代码定义了一些用于在特定的 macOS 操作系统特性上更方便地使用特定指令的预处理指令。


```cpp
#define WORDS_BIGENDIAN 1

/* Enable large inode numbers on Mac OS X 10.5.  */
#ifndef _DARWIN_USE_64_BIT_INODE
# define _DARWIN_USE_64_BIT_INODE 1
#endif

/* Number of bits in a file offset, on hosts where this is settable. */
#undef _FILE_OFFSET_BITS

/* Define for large files, on AIX-style hosts. */
#undef _LARGE_FILES

/* Define to empty if `const' does not conform to ANSI C. */
#undef const

```

这段代码定义了一些变量和函数，其中包含对C语言中`inline`和`__cplusplus`的支持情况。

如果C编译器支持`inline`，则定义为`__inline__'`或`__inline'`。否则，定义为空字符串(`''`)。此外，定义了一个名为`size_t`的常量，其类型为`unsigned int`(`未定义`)。

此外，定义了一个名为`LIBSSH2_DISABLE_QADRT_EXT`的宏，它通过`#pragma map`指令 remap了`zlib`库中的`inflateInit_`和`deflateInit_`函数，使其可以被 ASCII 版本调用。


```cpp
/* Define to `__inline__' or `__inline' if that's what the C compiler
   calls it, or to nothing if 'inline' is not supported under any name.  */
#ifndef __cplusplus
#define inline
#endif

/* Define to `unsigned int' if <sys/types.h> does not define. */
#undef size_t


#ifndef LIBSSH2_DISABLE_QADRT_EXT
/* Remap zlib procedures to ASCII versions. */
#pragma map(inflateInit_, "_libssh2_os400_inflateInit_")
#pragma map(deflateInit_, "_libssh2_os400_deflateInit_")
#endif

```

这段代码是一个自包含的 Vim 配置文件。它的作用是设置 Vim 编辑器的一些设置，包括打开两行模式（即 "tw"）和设置文本自动完成、自动缩进等。

具体来说，这段代码执行以下操作：

1. 打开两行模式：在 Vim 中，按下 "Esc" 键，再输入 "tw"，就可以进入两行模式。
2. 设置自动完成文本：当你在编辑器中输入一个 "?" 的时候，Vim 会自动去查找这个 "?" 定义，并完成文本的自动完成。
3. 设置自动缩进：当你在编辑器中输入一个 "+" 或者 "=" 的时候，Vim 会自动缩进你的代码块。

注意，这段代码中的 "vim: set expandtab ts=4 sw=4:" 是一个快捷键，你可以使用 "vim+" 命令来创建一个新的配置文件，它的作用和这段代码一样。


```cpp
#endif
/* vim: set expandtab ts=4 sw=4: */

```

# `libssh2/os400/macros.h`

This is a text-based copy of the Apache License 2.0, which is a widely used open-source software license. The specific version of the Apache License 2.0 being copied here is version 2.0 with cumulative changes, as of January 1, 2023.

This software license allows users to do just about anything with the software, for example, to copy, modify, distribute, and publicly perform the software, as long as they include the copyright notice and a license from the Apache Software Foundation.

The Apache Software Foundation requires users to provide attribution to the copyright holder and to include a copy of this license in all copies of the software, except for the少部分想单项许可或躬身使用，或以任何方式与服务或产品结合使用。


```cpp
/*
 * Copyright (C) 2015 Patrick Monnerat, D+H <patrick.monnerat@dh.com>
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

这段代码定义了一个头文件，名为 "libssh2_macros.h"，定义了在 libssh2_session_init 和 libssh2_session_disconnect 函数中使用的 macros。

具体来说，这段代码包含以下内容：

1. 定义了两个函数：

  LIBSSH2_API LIBSSH2_SESSION * libssh2_session_init(void);
  LIBSSH2_API int libssh2_session_disconnect(LIBSSH2_SESSION *session,
                                          const char *description);

2. 在函数头定义了一些预处理指令，用于生成 C 语言源代码中的 macros：

  #ifdef DEBUG
  #define DEBUG
  #endif

3. 在函数 body中包含了一些头文件：

  #include "libssh2.h"
  #include "libssh2_publickey.h"
  #include "libssh2_sftp.h"

4. 通过 `libssh2_session_init` 和 `libssh2_session_disconnect` 函数，实现了对 libssh2 库的 session 功能的封装，包括会话初始化和会话关闭。

总之，这段代码定义了两个函数，分别是会话初始化和会话关闭，通过定义了一些预处理指令，使得在 libssh2 库的 session 功能的封装更加简单和明确。


```cpp
#ifndef LIBSSH2_MACROS_H_
#define LIBSSH2_MACROS_H_

#include "libssh2.h"
#include "libssh2_publickey.h"
#include "libssh2_sftp.h"

/*
 * Dummy prototypes to generate wrapper procedures to C macros.
 * This is a helper for languages without a clever preprocessor (ILE/RPG).
 */

LIBSSH2_API LIBSSH2_SESSION * libssh2_session_init(void);
LIBSSH2_API int libssh2_session_disconnect(LIBSSH2_SESSION *session,
                                           const char *description);
```

这段代码是一个用于设置用户密码、从文件中读取公钥和私钥的库函数。

具体来说，`libssh2_userauth_password`函数用于设置用户密码，它接收两个指针参数，第一个参数是一个SSH会话，第二个参数是用户名和密码。该函数将密码存储在第二个指针中，然后返回修改后的会话。

`libssh2_userauth_publickey_fromfile`函数用于从文件中读取公钥。它接收四个指针参数，第一个参数是一个SSH会话，第二个参数是用户名，第三个参数是公钥和私钥，第四个参数是密码。该函数将从文件中读取公钥并将其存储在第三个指针中，然后尝试使用密码进行身份验证，如果成功，将尝试使用公钥进行身份验证。

`libssh2_userauth_hostbased_fromfile`函数用于从文件中读取身份验证信息。它与`libssh2_userauth_publickey_fromfile`类似，但仅读取文件中包含的公共密钥。它接收六个指针参数，第一个参数是一个SSH会话，第二个参数是用户名、公共密钥、私钥、密码和文件名。该函数将从文件中读取公共密钥，并尝试使用密码进行身份验证，如果成功，将尝试使用公共密钥进行身份验证。


```cpp
LIBSSH2_API int libssh2_userauth_password(LIBSSH2_SESSION *session,
                                          const char *username,
                                          const char *password);
LIBSSH2_API int
libssh2_userauth_publickey_fromfile(LIBSSH2_SESSION *session,
                                    const char *username,
                                    const char *publickey,
                                    const char *privatekey,
                                    const char *passphrase);
LIBSSH2_API int
libssh2_userauth_hostbased_fromfile(LIBSSH2_SESSION *session,
                                    const char *username,
                                    const char *publickey,
                                    const char *privatekey,
                                    const char *passphrase,
                                    const char *hostname);
```

这段代码是一个用于SSH客户端的标准库函数，实现了与服务器进行交互的过程。具体解释如下：

1. `int libssh2_userauth_keyboard_interactive`函数，用于在客户端与服务器之间的交互过程中获取用户输入。函数内部使用`LIBSSH2_USERAUTH_KBDINT_RESPONSE_FUNC`类型作为回调函数，该函数在获取用户输入后返回一个`LIBSSH2_USERAUTH_KBDINT_RESPONSE`类型的数据，代表用户输入的用户名和密码。

2. `LIBSSH2_API LIBSSH2_CHANNEL * libssh2_channel_open_session`函数，用于打开一个SSH会话。函数内部使用`LIBSSH2_SESSION *session`作为参数，使用`libssh2_channel_direct_tcpip`函数作为底层通道连接函数，然后使用`libssh2_channel_open`函数打开会话。

3. `LIBSSH2_API LIBSSH2_CHANNEL * libssh2_channel_direct_tcpip`函数，用于使用底层SSH协议连接到服务器。函数内部使用`const char *host`作为参数，用于指定服务器的主机名，使用`int port`作为参数，用于指定服务器监听的端口号。函数使用`libssh2_channel_direct_tcpip`函数作为底层连接函数，然后使用`LIBSSH2_API LIBSSH2_CHANNEL * libssh2_channel_open_session`函数打开会话。

4. `LIBSSH2_API LIBSSH2_LISTENER * libssh2_channel_forward_listen`函数，用于在服务器端监听客户端连接并转发数据。函数内部使用`LIBSSH2_SESSION *session`作为参数，使用`int port`作为参数，用于指定服务器监听的端口号。函数使用`libssh2_channel_forward_listen`函数作为底层监听函数，然后使用`LIBSSH2_API int libssh2_channel_setenv`函数设置服务器环境变量。

5. `LIBSSH2_API int libssh2_channel_setenv`函数，用于设置服务器环境变量。函数内部使用`const char *varname`作为参数，用于指定要设置的环境变量，使用`const char *value`作为参数，用于指定要设置的变量值。函数返回一个整数表示设置的环境变量ID，可以用来获取设置的环境变量值。


```cpp
LIBSSH2_API int
libssh2_userauth_keyboard_interactive(LIBSSH2_SESSION* session,
                                      const char *username,
                                      LIBSSH2_USERAUTH_KBDINT_RESPONSE_FUNC(
                                                       (*response_callback)));
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_channel_open_session(LIBSSH2_SESSION *session);
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_channel_direct_tcpip(LIBSSH2_SESSION *session, const char *host,
                             int port);
LIBSSH2_API LIBSSH2_LISTENER *
libssh2_channel_forward_listen(LIBSSH2_SESSION *session, int port);
LIBSSH2_API int
libssh2_channel_setenv(LIBSSH2_CHANNEL *channel,
                       const char *varname, const char *value);
```

这段代码是一个名为`libssh2_channel_request_pty`的函数，它是`libssh2_channel_request_pty_size`的函数内部实现。它的作用是接受一个`LIBSSH2_CHANNEL`类型的channel和一个字符串参数`term`，然后返回一个整数类型的`channel`，其中包含一个指向字符串`term`的指针。

具体来说，这段代码实现了一个基于libssh2库的通道操作，通过调用libssh2库中`LIBSSH2_channel_request_pty`、`LIBSSH2_channel_request_pty_size`、`LIBSSH2_channel_x11_req`、`LIBSSH2_channel_shell`和`LIBSSH2_channel_exec`函数来实现。通过这些函数，可以实现对通道的输入、输出、读取、执行等操作，从而实现一个完整的通道操作流程。


```cpp
LIBSSH2_API int
libssh2_channel_request_pty(LIBSSH2_CHANNEL *channel, const char *term);
LIBSSH2_API int
libssh2_channel_request_pty_size(LIBSSH2_CHANNEL *channel,
                                 int width, int height);
LIBSSH2_API int
libssh2_channel_x11_req(LIBSSH2_CHANNEL *channel, int screen_number);
LIBSSH2_API int
libssh2_channel_shell(LIBSSH2_CHANNEL *channel);
LIBSSH2_API int
libssh2_channel_exec(LIBSSH2_CHANNEL *channel, const char *command);
LIBSSH2_API int
libssh2_channel_subsystem(LIBSSH2_CHANNEL *channel, const char *subsystem);
LIBSSH2_API ssize_t
libssh2_channel_read(LIBSSH2_CHANNEL *channel, char *buf, size_t buflen);
```

该代码是一个名为libssh2_api的函数，它是SSH2协议的API。以下是该代码的作用和各函数的说明：

1. libssh2_channel_read_stderr函数的作用是读取channel缓冲区中的数据并将其存储在buf中。同时将buflen作为参数传入。

2. libssh2_channel_window_read函数的作用是从channel缓冲区中逐个读取数据，并将其存储在buf中。同时返回libssh2_channel_window结构中当前已经读取的数据个数。

3. libssh2_channel_write函数的作用是将BUF中的数据写入到channel缓冲区中。同时，根据第二个参数的不同，分别实现为写入buf中数据或者buflen的到channel缓冲区中。

4. libssh2_channel_write_stderr函数的作用与libssh2_channel_write函数类似，但同时将错误信息添加到buf中。

5. libssh2_channel_window_write函数的作用是从channel缓冲区中逐个写入数据，并将其存储在window结构中。

6. libssh2_channel_flush函数的作用是刷新channel缓冲区，将缓冲区中的所有数据写入到channel缓冲区中。

7. libssh2_channel_flush_stderr函数的作用与libssh2_channel_flush函数类似，但同时将错误信息添加到buf中。

8. libssh2_scp_send函数的作用是发送文件到远程主机，接收文件路径，文件模式和发送的文件大小。


```cpp
LIBSSH2_API ssize_t
libssh2_channel_read_stderr(LIBSSH2_CHANNEL *channel, char *buf, size_t buflen);
LIBSSH2_API unsigned long
libssh2_channel_window_read(LIBSSH2_CHANNEL *channel);
LIBSSH2_API ssize_t
libssh2_channel_write(LIBSSH2_CHANNEL *channel, const char *buf, size_t buflen);
LIBSSH2_API ssize_t
libssh2_channel_write_stderr(LIBSSH2_CHANNEL *channel,
                             const char *buf, size_t buflen);
LIBSSH2_API unsigned long
libssh2_channel_window_write(LIBSSH2_CHANNEL *channel);
LIBSSH2_API int libssh2_channel_flush(LIBSSH2_CHANNEL *channel);
LIBSSH2_API int libssh2_channel_flush_stderr(LIBSSH2_CHANNEL *channel);
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_scp_send(LIBSSH2_SESSION *session,
                 const char *path, int mode, libssh2_int64_t size);

```

该代码是一个名为libssh2_publickey_add的函数，属于libssh2库，用于添加公钥到库中。

具体来说，该函数接受一个publickey指针pkey，一个name参数，一个blob参数（包含公钥数据），以及一些用于指定是否覆盖现有公钥的选项，如overwrite（是否覆盖）。接着，函数将这些选项传递给名为attrs的数组，然后返回0（成功）或一个负数（失败）。

以下是函数的更详细的说明：

- pkey：公钥指针，指向要添加的publickey。
- name：要添加的公钥名称。
- blob：公钥数据，以字节数组形式传递。
- blob_len：公钥数据的字节数。
- overwrite：是否覆盖现有公钥（设置为0时，不覆盖，设置为1时，覆盖）。
- num_attrs：公钥属性的数量。
- attrs：公钥属性的数组，指向要添加的属性。

以下是函数的实现：
```cpp
int libssh2_publickey_add(LIBSSH2_PUBLICKEY *pkey, const unsigned char *name,
                     const unsigned char *blob, unsigned long blob_len,
                     char overwrite, unsigned long num_attrs,
                     const libssh2_publickey_attribute attrs[])
{
   int ret;
   ret = libssh2_myssh2_umark(overwrite);
   if (ret != 0) {
       return ret;
   }

   ret = libssh2_myssh2_putsctxt(pkey, name, blob, blob_len);
   if (ret != 0) {
       return ret;
   }

   ret = libssh2_myssh2_putpublickey(pkey, overwrite, num_attrs, attrs);
   if (ret != 0) {
       return ret;
   }

   return 0;
}
```
以下是函数的定义：
```cpp
int LIBSSH2_SFTP_S_ISLNK(unsigned long permissions);
int LIBSSH2_SFTP_S_ISREG(unsigned long permissions);
int LIBSSH2_SFTP_S_ISDIR(unsigned long permissions);
int LIBSSH2_SFTP_S_ISCHR(unsigned long permissions);
int LIBSSH2_SFTP_S_ISBLK(unsigned long permissions);
int LIBSSH2_SFTP_S_ISFIFO(unsigned long permissions);
```
这些是函数libssh2_publickey_add所支持的库函数，用于与SFTP相关的事务。


```cpp
LIBSSH2_API int
libssh2_publickey_add(LIBSSH2_PUBLICKEY *pkey, const unsigned char *name,
		      const unsigned char *blob, unsigned long blob_len,
                      char overwrite, unsigned long num_attrs,
		      const libssh2_publickey_attribute attrs[]);
LIBSSH2_API int
libssh2_publickey_remove(LIBSSH2_PUBLICKEY *pkey, const unsigned char *name,
                         const unsigned char *blob, unsigned long blob_len);

LIBSSH2_API int LIBSSH2_SFTP_S_ISLNK(unsigned long permissions);
LIBSSH2_API int LIBSSH2_SFTP_S_ISREG(unsigned long permissions);
LIBSSH2_API int LIBSSH2_SFTP_S_ISDIR(unsigned long permissions);
LIBSSH2_API int LIBSSH2_SFTP_S_ISCHR(unsigned long permissions);
LIBSSH2_API int LIBSSH2_SFTP_S_ISBLK(unsigned long permissions);
LIBSSH2_API int LIBSSH2_SFTP_S_ISFIFO(unsigned long permissions);
```

这段代码是一个用于SSH2 SFTP文件操作的库函数。它允许在SSH2 SFTP中打开、关闭和读取目录。以下是每个函数的简要说明：

1. `LIBSSH2_API int LIBSSH2_SFTP_S_ISSOCK(unsigned long permissions)`：检查给定的权限是否足够在SSH2 SFTP中创建套接字。
2. `LIBSSH2_API LIBSSH2_SFTP_HANDLE * libssh2_sftp_open(LIBSSH2_SFTP *sftp, const char *filename, unsigned long flags, long mode)`：打开一个SSH2 SFTP文件到给定文件。函数的第一个参数是一个指向SSH2 SFTP对象的句柄，第二个参数是要读取或写入的文件名，第三个参数是SSH2 SFTP文件操作的标志，第四个参数是文件权限，第五个参数是文件描述符。
6. `LIBSSH2_API LIBSSH2_SFTP_HANDLE * libssh2_sftp_opendir(LIBSSH2_SFTP *sftp, const char *path)`：打开一个SSH2 SFTP目录到给定路径。函数的第一个参数是一个指向SSH2 SFTP对象的句柄，第二个参数是要打开的目录名，第三个参数是SSH2 SFTP文件操作的标志，第四个参数是文件描述符。
10. `LIBSSH2_API int libssh2_sftp_readdir(LIBSSH2_SFTP_HANDLE *handle, char *buffer, size_t buffer_maxlen, LIBSSH2_SFTP_ATTRIBUTES *attrs)`：读取给定handle的目录中的文件。函数的第一个参数是一个指向SSH2 SFTP对象的句柄，第二个参数是要读取的文件名，第三个参数是文件描述符，第四个参数是文件属性，第五个参数是int，表示文件描述符中包含的文件个数。
13. `LIBSSH2_API int libssh2_sftp_close(LIBSSH2_SFTP_HANDLE *handle)`：关闭一个SSH2 SFTP文件到给定句柄。函数的第一个参数是一个指向SSH2 SFTP对象的句柄，第二个参数是关闭的文件描述符。
16. `LIBSSH2_API int libssh2_sftp_closedir(LIBSSH2_SFTP_HANDLE *handle)`：关闭一个SSH2 SFTP目录到给定句柄。函数的第一个参数是一个指向SSH2 SFTP对象的句柄，第二个参数是关闭的文件描述符。
18. `void libssh2_sftp_rewind(LIBSSH2_SFTP_HANDLE *handle)`：在SSH2 SFTP中恢复文件。函数的第一个参数是一个指向SSH2 SFTP对象的句柄，用于指定要恢复的文件。
21. `LIBSSH2_API int libssh2_sftp_fstat(LIBSSH2_SFTP_HANDLE *handle, LIBSSH2_SFTP_ATTRIBUTES *attrs)`：返回给定文件描述符的文件属性。函数的第一个参数是一个指向SSH2 SFTP对象的句柄，第二个参数是一个指向LIBSSH2 SFTP文件属性的结构体，用于指定要返回的文件属性。
24. `LIBSSH2_API int libssh2_sftp_fsetstat(LIBSSH2_SFTP_HANDLE *handle, LIBSSH2_SFTP_ATTRIBUTES *attrs)`：设置给定文件描述符的文件属性。函数的第一个参数是一个指向SSH2 SFTP对象的句柄，第二个参数是一个指向LIBSSH2 SFTP文件属性的结构体，用于指定要设置的文件属性。


```cpp
LIBSSH2_API int LIBSSH2_SFTP_S_ISSOCK(unsigned long permissions);
LIBSSH2_API LIBSSH2_SFTP_HANDLE *
libssh2_sftp_open(LIBSSH2_SFTP *sftp, const char *filename,
                  unsigned long flags, long mode);
LIBSSH2_API LIBSSH2_SFTP_HANDLE *
libssh2_sftp_opendir(LIBSSH2_SFTP *sftp, const char *path);
LIBSSH2_API int libssh2_sftp_readdir(LIBSSH2_SFTP_HANDLE *handle,
                                     char *buffer, size_t buffer_maxlen,
                                     LIBSSH2_SFTP_ATTRIBUTES *attrs);
LIBSSH2_API int libssh2_sftp_close(LIBSSH2_SFTP_HANDLE *handle);
LIBSSH2_API int libssh2_sftp_closedir(LIBSSH2_SFTP_HANDLE *handle);
LIBSSH2_API void libssh2_sftp_rewind(LIBSSH2_SFTP_HANDLE *handle);
LIBSSH2_API int libssh2_sftp_fstat(LIBSSH2_SFTP_HANDLE *handle,
                                   LIBSSH2_SFTP_ATTRIBUTES *attrs);
LIBSSH2_API int libssh2_sftp_fsetstat(LIBSSH2_SFTP_HANDLE *handle,
                                      LIBSSH2_SFTP_ATTRIBUTES *attrs);
```

这段代码是一个名为libssh2_sftp_rename、libssh2_sftp_unlink、libssh2_sftp_mkdir、libssh2_sftp_rmdir、libssh2_sftp_stat和libssh2_sftp_lstat的函数，它们是libssh2_sftp的几个重要功能。

libssh2_sftp_rename函数用于将 SFTP 上的文件或目录重命名为另一个文件或目录，可以通过 source_filename 和 dest_filename 参数指定要重命名的文件或目录。

libssh2_sftp_unlink函数用于在 SFTP 上删除指定的文件或目录，可以通过 filename 参数指定要删除的文件或目录。

libssh2_sftp_mkdir函数用于在 SFTP 上创建一个新的目录，可以通过 path 和 mode 参数指定要创建的目录的路径和权限模式。

libssh2_sftp_rmdir函数用于在 SFTP 上删除指定的目录，可以通过 path 参数指定要删除的目录。

libssh2_sftp_stat函数用于返回 SFTP 上的文件或目录的元数据信息，包括文件类型、文件大小、文件权限、文件所有者、文件所属组等。

libssh2_sftp_lstat函数与 libssh2_sftp_stat 类似，但返回文件或目录的元数据信息中缺少文件所有者和所属组。

libssh2_sftp_setstat函数用于设置 SFTP 上文件或目录的元数据信息，包括文件类型、文件大小、文件权限、文件所有者、文件所属组等。

libssh2_sftp_symlink函数用于在 SFTP 上创建一个新的符号链接，可以指定原名为符号链接的目标路径。


```cpp
LIBSSH2_API int libssh2_sftp_rename(LIBSSH2_SFTP *sftp,
                                    const char *source_filename,
                                    const char *dest_filename);
LIBSSH2_API int libssh2_sftp_unlink(LIBSSH2_SFTP *sftp, const char *filename);
LIBSSH2_API int libssh2_sftp_mkdir(LIBSSH2_SFTP *sftp,
                                   const char *path, long mode);
LIBSSH2_API int libssh2_sftp_rmdir(LIBSSH2_SFTP *sftp, const char *path);
LIBSSH2_API int libssh2_sftp_stat(LIBSSH2_SFTP *sftp, const char *path,
                                  LIBSSH2_SFTP_ATTRIBUTES *attrs);
LIBSSH2_API int libssh2_sftp_lstat(LIBSSH2_SFTP *sftp, const char *path,
                                   LIBSSH2_SFTP_ATTRIBUTES *attrs);
LIBSSH2_API int libssh2_sftp_setstat(LIBSSH2_SFTP *sftp, const char *path,
                                     LIBSSH2_SFTP_ATTRIBUTES *attrs);
LIBSSH2_API int libssh2_sftp_symlink(LIBSSH2_SFTP *sftp, const char *orig,
                                     char *linkpath);
```

这两段代码定义了两个名为libssh2_sftp_readlink和libssh2_sftp_realpath的函数，用于读取和获取SFTP文件中的链接文件名。

libssh2_sftp_readlink函数接收三个参数：SFTP指针、路径和目标字符串，以及一个最大允许读取的字节数。函数使用SFTP的getcwd函数获取当前工作目录，并使用SFTP的link文件属性中的base64编码名取得链接文件名。然后，函数将目标字符串和当前工作目录一起作为参数传递给函数realpath，函数使用base64解码和len取得链接文件实际路径名，并返回该路径名。

libssh2_sftp_realpath函数与libssh2_sftp_readlink类似，但使用了libssh2_sftp_getenter函数取得链接文件名，而不是使用SFTP的属性。函数同样接收三个参数，包括当前工作目录、路径和目标字符串，以及一个最大允许读取的字节数。函数使用SFTP的getcwd和linkfile属性的组合，获取链接文件名并解码，然后使用base64解码和len取得链接文件实际路径名，并将其存储到目标字符串中。函数使用libssh2_sftp_readlink函数的第二个参数作为传递给函数的路径参数，以实现对链接文件的读取。


```cpp
LIBSSH2_API int libssh2_sftp_readlink(LIBSSH2_SFTP *sftp, const char *path,
                                      char *target, unsigned int maxlen);
LIBSSH2_API int libssh2_sftp_realpath(LIBSSH2_SFTP *sftp, const char *path,
                                      char *target, unsigned int maxlen);

#endif

```