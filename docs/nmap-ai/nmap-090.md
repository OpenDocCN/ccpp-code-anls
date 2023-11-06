# Nmap源码解析 90

# `libssh2/include/libssh2_publickey.h`

This is a copyright notice for the software package libssh2. This package is developed by Sara Golemon and is distributed under the GPL (Gitt performer license) license.

The GPL license allows users to do the following things:

* Redistribute the software by including it in any executable, module, or script that runs on a user's machine.
* Install the software on a user's machine.
* Run the software on a user's machine without modification to the original source code.

The copyright notice also mentions that any third-party libraries or programs that use libssh2 must also be licensed under the GPL license.

Note that the use, distribution, or modification of libssh2 itself is subject to the terms of the GPL license.


```cpp
/* Copyright (c) 2004-2006, Sara Golemon <sarag@libssh2.org>
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

这段代码定义了一个名为 "libssh2_publickey.h" 的头文件，但不需要包含任何实际的代码。这个头文件被包括在 "libssh2.h" 库中，通常与 "libssh2" 一起使用。

通过包含 "libssh2_publickey.h"，你可以在其他源代码中使用 "libssh2" 库中的 "publickey" 子模块，而不是直接使用 "publickey" 子模块。因为 "publickey" 是 "libssh2_publickey_subsystem" 的别名，而这个别名在 "publickey" 和 "publickey_subsystem" 之间存在区别。


```cpp
/* Note: This include file is only needed for using the
 * publickey SUBSYSTEM which is not the same as publickey
 * authentication.  For authentication you only need libssh2.h
 *
 * For more information on the publickey subsystem,
 * refer to IETF draft: secsh-publickey
 */

#ifndef LIBSSH2_PUBLICKEY_H
#define LIBSSH2_PUBLICKEY_H 1

#include "libssh2.h"

typedef struct _LIBSSH2_PUBLICKEY               LIBSSH2_PUBLICKEY;

```

这段代码定义了两个结构体：`libssh2_publickey_attribute` 和 `libssh2_publickey_list`。它们的目的是定义 `libssh2_publickey` 类型的成员变量。

具体来说，`libssh2_publickey_attribute` 结构体定义了六个成员变量：`name`、`name_len`、`value`、`value_len` 和 `mandatory`，它们分别表示公钥属性、属性名称长度、属性值、属性值长度和强制属性。

`libssh2_publickey_list` 结构体定义了四个成员变量：`packet`、`name`、`blob` 和 `blob_len`，它们分别表示公钥列表的发送数据、公钥名称长度、公钥数据和公钥数据长度。

这两个结构体都是 `libssh2_publickey` 的成员变量，因此它们都继承自 `struct libssh2_publickey` 结构体。


```cpp
typedef struct _libssh2_publickey_attribute {
    const char *name;
    unsigned long name_len;
    const char *value;
    unsigned long value_len;
    char mandatory;
} libssh2_publickey_attribute;

typedef struct _libssh2_publickey_list {
    unsigned char *packet; /* For freeing */

    const unsigned char *name;
    unsigned long name_len;
    const unsigned char *blob;
    unsigned long blob_len;
    unsigned long num_attrs;
    libssh2_publickey_attribute *attrs; /* free me */
} libssh2_publickey_list;

```

这段代码定义了两个宏，名为`libssh2_publickey_attribute`和`libssh2_publickey_attribute_fast`。它们都接受`name`和`value`这两个参数，其中`name`可以是字符串 literal，而`value`也可以是字符串 literal。宏定义中包含了一个强制赋值` mandatory`，它会将`name`和`value`的合适长度计算出来，并将计算结果存储回 ` mandatory`。

具体来说，`libssh2_publickey_attribute`会在编译时检查输入是否使用了预处理指令`__fast`，如果是，则使用`_fast()`函数进行计算。否则，按照定义计算。 `libssh2_publickey_attribute_fast`则是对`libssh2_publickey_attribute`进行别名定义，同样会检查输入是否使用了预处理指令`__fast`，如果是，则使用`_fast()`函数进行计算；否则，使用内置函数`strlen()`计算。

此外，代码中还有一句依从性声明，即`extern "C"`。这意味着该代码为 C 语言源代码编写，并且遵循了 C 语言的编码规范。


```cpp
/* Generally use the first macro here, but if both name and value are string
   literals, you can use _fast() to take advantage of preprocessing */
#define libssh2_publickey_attribute(name, value, mandatory) \
  { (name), strlen(name), (value), strlen(value), (mandatory) },
#define libssh2_publickey_attribute_fast(name, value, mandatory) \
  { (name), sizeof(name) - 1, (value), sizeof(value) - 1, (mandatory) },

#ifdef __cplusplus
extern "C" {
#endif

/* Publickey Subsystem */
LIBSSH2_API LIBSSH2_PUBLICKEY *
libssh2_publickey_init(LIBSSH2_SESSION *session);

```

这两段代码定义了两个名为 `libssh2_publickey_add_ex` 和 `libssh2_publickey_remove_ex` 的函数，它们属于 `libssh2_publickey` 类型。这些函数可以用来向 `libssh2_publickey` 结构中添加或删除 `name` 属性的值。

`libssh2_publickey_add_ex` 函数接收一个 `LIBSSH2_PUBLICKEY` 类型的参数 `pkey`，一个 `const unsigned char *` 类型的参数 `name`，一个 `unsigned long` 类型的参数 `name_len`，一个 `const unsigned char *` 类型的参数 `blob`，一个 `unsigned long` 类型的参数 `blob_len`，一个布尔类型的参数 `overwrite`，和一个 `unsigned long` 类型的参数 `num_attrs`。这个函数将 `name` 和 `blob` 添加到 `pkey` 的 `name` 和 `blob` 属性中，如果 `overwrite` 为 `1`，则会覆盖现有的属性值。函数返回 `0`，表示成功添加或修改了属性。

`libssh2_publickey_remove_ex` 函数与 `libssh2_publickey_add_ex` 相反，这个函数接收一个 `LIBSSH2_PUBLICKEY` 类型的参数 `pkey`，一个 `const unsigned char *` 类型的参数 `name`，一个 `unsigned long` 类型的参数 `name_len`，一个 `const unsigned char *` 类型的参数 `blob`，一个 `unsigned long` 类型的参数 `blob_len`。这个函数将从 `pkey` 的 `name` 和 `blob` 属性中删除 `name`，并返回一个值表示成功删除。


```cpp
LIBSSH2_API int
libssh2_publickey_add_ex(LIBSSH2_PUBLICKEY *pkey,
                         const unsigned char *name,
                         unsigned long name_len,
                         const unsigned char *blob,
                         unsigned long blob_len, char overwrite,
                         unsigned long num_attrs,
                         const libssh2_publickey_attribute attrs[]);
#define libssh2_publickey_add(pkey, name, blob, blob_len, overwrite,    \
                              num_attrs, attrs)                         \
  libssh2_publickey_add_ex((pkey), (name), strlen(name), (blob), (blob_len), \
                           (overwrite), (num_attrs), (attrs))

LIBSSH2_API int libssh2_publickey_remove_ex(LIBSSH2_PUBLICKEY *pkey,
                                            const unsigned char *name,
                                            unsigned long name_len,
                                            const unsigned char *blob,
                                            unsigned long blob_len);
```

这段代码定义了三个函数，它们用于与SSH公钥相关的操作。接下来，我将逐一解释每个函数的作用。

1. `libssh2_publickey_remove`：
该函数的实现在`libssh2_publickey_remove_ex`函数中。它的作用是接收一个公钥（`pkey`），一个名称（`name`），一个二进制数据（`blob`），以及一个二进制数据长度（`blob_len`）。它然后执行以下操作：

  a. 通过调用`libssh2_publickey_remove_ex`函数，将公钥和名称转换为相应的数据类型。
  b. 使用`strlen`函数获取名称的长度。
  c. 将二进制数据和长度复制到公钥变量中。
  d. 返回公钥的变化。

2. `libssh2_publickey_list_fetch`：
该函数用于在公钥列表中检索一些公钥。它的作用是接收公钥列表和公钥列表中的元素数量。然后，它将调用`libssh2_publickey_shutdown`函数来关闭公钥列表。

3. `libssh2_publickey_list_free`：
该函数用于释放公钥列表。它的作用是接收公钥列表和公钥列表的指针。然后，它将调用`libssh2_publickey_shutdown`函数来关闭公钥列表。

4. `libssh2_publickey_shutdown`：
该函数的作用是关闭公钥列表。


```cpp
#define libssh2_publickey_remove(pkey, name, blob, blob_len) \
  libssh2_publickey_remove_ex((pkey), (name), strlen(name), (blob), (blob_len))

LIBSSH2_API int
libssh2_publickey_list_fetch(LIBSSH2_PUBLICKEY *pkey,
                             unsigned long *num_keys,
                             libssh2_publickey_list **pkey_list);
LIBSSH2_API void
libssh2_publickey_list_free(LIBSSH2_PUBLICKEY *pkey,
                            libssh2_publickey_list *pkey_list);

LIBSSH2_API int libssh2_publickey_shutdown(LIBSSH2_PUBLICKEY *pkey);

#ifdef __cplusplus
} /* extern "C" */
```

这段代码是在C或C++语言中用来定义一个名为“Libssh2_publickey”的函数指针变量。

其中，`#ifdef`表示先判断是否已经定义过该函数指针变量，如果没有定义过，则定义一个新的函数指针变量；

`#endif`则表示如果已经定义过该函数指针变量，则直接跳过该分支，不再执行下面的语句。

具体来说，如果已经定义过该函数指针变量，则该函数指针变量将不再被链接，因此如果再次定义该函数指针变量，将不再拥有该函数指针指向的函数的内部实现。


```cpp
#endif

#endif /* ifndef: LIBSSH2_PUBLICKEY_H */

```

# `libssh2/include/libssh2_sftp.h`

Sara Golemon
Excellent, is there anything specific you would like me to do with the software or the information?


```cpp
/* Copyright (c) 2004-2008, Sara Golemon <sarag@libssh2.org>
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

这段代码是一个 C 语言预处理指令，它定义了一个头文件，名为 "libssh2_sftp.h"，它包含了 "libssh2.h" 和 "unistd.h"。

同时，它还包含了一个屏幕输出，#include "libssh2.h" 告诉操作系统应该在屏幕上输出 "libssh2.h" 的内容。

此外，它还包含了一个名为 "__cplusplus" 的预处理指令，这意味着它可以将任何 "const" 类型的参数视为 "char*" 类型，从而使得定义和使用 "const char*" 类型时更方便。

最后，注意在屏幕输出部分，使用了 "#ifdef __cplusplus" 注释，这意味着 "const" 类型的参数可以被当做 "char*" 类型看待。


```cpp
#ifndef LIBSSH2_SFTP_H
#define LIBSSH2_SFTP_H 1

#include "libssh2.h"

#ifndef WIN32
#include <unistd.h>
#endif

#ifdef __cplusplus
extern "C" {
#endif

/* Note: Version 6 was documented at the time of writing
 * However it was marked as "DO NOT IMPLEMENT" due to pending changes
 *
 * Let's start with Version 3 (The version found in OpenSSH) and go from there
 */
```

这段代码定义了一系列头文件和结构体，用于定义 SFTP（Secure Shell Feature）协议的版本、SFTP 文件操作、以及 SFTP 文件属性的枚举类型。

具体来说，这段代码定义了 LIBSSH2_SFTP_VERSION，表示 SFTP 协议的版本号；定义了 LIBSSH2_SFTP、LIBSSH2_SFTP_HANDLE 和 LIBSSH2_SFTP_ATTRIBUTES，分别表示 SFTP 协议的打开文件、打开目录和属性；定义了 LIBSSH2_SFTP_STATVFS，表示 SFTP 协议的统计文件大小。

接下来的代码定义了一系列函数，用于实现 SFTP 文件的打开、关闭和读写等操作。其中，open_ex() 和 rename_ex() 是用于打开和重新命名 SFTP 文件的函数，具体的打开和命名方式可以通过这些函数的参数进行配置。


```cpp
#define LIBSSH2_SFTP_VERSION        3

typedef struct _LIBSSH2_SFTP                LIBSSH2_SFTP;
typedef struct _LIBSSH2_SFTP_HANDLE         LIBSSH2_SFTP_HANDLE;
typedef struct _LIBSSH2_SFTP_ATTRIBUTES     LIBSSH2_SFTP_ATTRIBUTES;
typedef struct _LIBSSH2_SFTP_STATVFS        LIBSSH2_SFTP_STATVFS;

/* Flags for open_ex() */
#define LIBSSH2_SFTP_OPENFILE           0
#define LIBSSH2_SFTP_OPENDIR            1

/* Flags for rename_ex() */
#define LIBSSH2_SFTP_RENAME_OVERWRITE   0x00000001
#define LIBSSH2_SFTP_RENAME_ATOMIC      0x00000002
#define LIBSSH2_SFTP_RENAME_NATIVE      0x00000004

```

这段代码定义了一系列与 Sftp 相关的宏和 flag，用于实现 SFTP 功能。具体来说：

1. LIBSSH2_SFTP_STAT：定义了 SFTP 状态的标志，包括是否成功、是否可读、是否可写等。
2. LIBSSH2_SFTP_LSTAT：定义了 SFTP 状态的标志，包括链接状态、序列号状态等。
3. LIBSSH2_SFTP_SETSTAT：定义了 SFTP 状态的标志，包括设置状态、设置序列号状态等。
4. LIBSSH2_SFTP_SYMLINK：定义了 SFTP 链接的标志，包括链接类型、链接状态等。
5. LIBSSH2_SFTP_READLINK：定义了 SFTP 读取文件的模式，包括是否从远程主机读取、是否递归等。
6. LIBSSH2_SFTP_DEFAULT_MODE：定义了 SFTP 默认的工作模式，包括是否使用标准模式、是否使用 ASAN 等。
7. LIBSSH2_SFTP_ATTR_SIZE：定义了 SFTP 文件属性的最大尺寸，以便在内存中分配空间时进行分配。
8. LIBSSH2_SFTP_PATH_MAX：定义了 SFTP 路径的最大长度，以便在路径过长时进行处理。


```cpp
/* Flags for stat_ex() */
#define LIBSSH2_SFTP_STAT               0
#define LIBSSH2_SFTP_LSTAT              1
#define LIBSSH2_SFTP_SETSTAT            2

/* Flags for symlink_ex() */
#define LIBSSH2_SFTP_SYMLINK            0
#define LIBSSH2_SFTP_READLINK           1
#define LIBSSH2_SFTP_REALPATH           2

/* Flags for sftp_mkdir() */
#define LIBSSH2_SFTP_DEFAULT_MODE      -1

/* SFTP attribute flag bits */
#define LIBSSH2_SFTP_ATTR_SIZE              0x00000001
```

这段代码定义了一个结构体 `LIBSSH2_SFTP_ATTRIBUTES`，用于表示 SFTP 文件系统的属性。

这个结构体包含以下字段：

1. `flags`：一个 32 位无符号整数，定义了 SFTP 文件的权限。具体来说，权限掩码为 `(~LIBSSH2_SFTP_ST_RDONLY) & ~LIBSSH2_SFTP_ST_NOSUID`。
2. `filesize`：文件大小，以字节为单位。
3. `uid`：文件所属的用户 ID，以 0 为前缀，后面跟着一个 16 位十六进制数。
4. `gid`：文件所属的组标识，以 0 为前缀，后面跟着一个 16 位十六进制数。
5. `permissions`：文件权限，包括文件类型、文件大小、是否可执行、是否重写等。具体来说，权限掩码为 `(~LIBSSH2_SFTP_ST_RDONLY) & ~(~LIBSSH2_SFTP_ST_FILETYPE) & ~(~LIBSSH2_SFTP_ST_FILESIZE) & ~(~LIBSSH2_SFTP_ST_READ_ONLY) & ~(~LIBSSH2_SFTP_ST_WRITE_ONLY) & ~(~LIBSSH2_SFTP_ST_EXECUTABLE) & ~(~LIBSSH2_SFTP_ST_SIZE_PREFIX)`。
6. `atime`：文件创建时间，以秒为单位。
7. `mtime`：文件修改时间，以秒为单位。
8. `extended`：文件是否使用了扩展功能，`0` 为未使用，`1` 使用。具体来说，扩展功能包括 `LIBSSH2_SFTP_ST_EXTENDED_REPLACEMENT`、`LIBSSH2_SFTP_ST_EXTENDED_KEYPAD`、`LIBSSH2_SFTP_ST_EXTENDED_TOGUIDE` 等。


```cpp
#define LIBSSH2_SFTP_ATTR_UIDGID            0x00000002
#define LIBSSH2_SFTP_ATTR_PERMISSIONS       0x00000004
#define LIBSSH2_SFTP_ATTR_ACMODTIME         0x00000008
#define LIBSSH2_SFTP_ATTR_EXTENDED          0x80000000

/* SFTP statvfs flag bits */
#define LIBSSH2_SFTP_ST_RDONLY              0x00000001
#define LIBSSH2_SFTP_ST_NOSUID              0x00000002

struct _LIBSSH2_SFTP_ATTRIBUTES {
    /* If flags & ATTR_* bit is set, then the value in this struct will be
     * meaningful Otherwise it should be ignored
     */
    unsigned long flags;

    libssh2_uint64_t filesize;
    unsigned long uid, gid;
    unsigned long permissions;
    unsigned long atime, mtime;
};

```



这是一个结构体定义了 SFTP (Secure File System) 的元数据结构。它包括以下几个成员：

- `f_bsize`：表示文件系统块大小，单位是字节。
- `f_frsize`：表示片段大小，单位是字节。
- `f_blocks`：表示在 `f_frsize` 单位下的文件系统块数量。
- `f_bfree`：表示可用的文件系统块数量。
- `f_bavail`：表示非根目录下的可用的文件系统块数量。
- `f_files`：表示在系统中可读取的文件数量。
- `f_ffree`：表示可用的文件索引节点的数量。
- `f_favail`：表示可用于非根目录的文件数量。
- `f_fsid`：表示文件系统的ID。
- `f_flag`：表示文件系统的挂载标志。
- `f_namemax`：表示文件名最大长度。


```cpp
struct _LIBSSH2_SFTP_STATVFS {
    libssh2_uint64_t  f_bsize;    /* file system block size */
    libssh2_uint64_t  f_frsize;   /* fragment size */
    libssh2_uint64_t  f_blocks;   /* size of fs in f_frsize units */
    libssh2_uint64_t  f_bfree;    /* # free blocks */
    libssh2_uint64_t  f_bavail;   /* # free blocks for non-root */
    libssh2_uint64_t  f_files;    /* # inodes */
    libssh2_uint64_t  f_ffree;    /* # free inodes */
    libssh2_uint64_t  f_favail;   /* # free inodes for non-root */
    libssh2_uint64_t  f_fsid;     /* file system ID */
    libssh2_uint64_t  f_flag;     /* mount flags */
    libssh2_uint64_t  f_namemax;  /* maximum filename length */
};

/* SFTP filetypes */
```

这段代码定义了LIBSSH2_SFTP_TYPE_REGULAR、LIBSSH2_SFTP_TYPE_DIRECTORY、LIBSSH2_SFTP_TYPE_SYMLINK、LIBSSH2_SFTP_TYPE_SPECIAL、LIBSSH2_SFTP_TYPE_UNKNOWN、LIBSSH2_SFTP_TYPE_SOCKET和LIBSSH2_SFTP_TYPE_CHAR_DEVICE、LIBSSH2_SFTP_TYPE_BLOCK_DEVICE和LIBSSH2_SFTP_TYPE_FIFO这8种SFTP文件类型。

其中，LIBSSH2_SFTP_TYPE_REGULAR代表普通文件类型，LIBSSH2_SFTP_TYPE_DIRECTORY代表目录文件类型，LIBSSH2_SFTP_TYPE_SYMLINK代表符号链接文件类型，LIBSSH2_SFTP_TYPE_SPECIAL代表特殊文件类型，LIBSSH2_SFTP_TYPE_UNKNOWN代表未知文件类型，LIBSSH2_SFTP_TYPE_SOCKET代表套接字文件类型，LIBSSH2_SFTP_TYPE_CHAR_DEVICE代表字符设备文件类型，LIBSSH2_SFTP_TYPE_BLOCK_DEVICE代表块设备文件类型，LIBSSH2_SFTP_TYPE_FIFO代表文件设备文件类型。

这些定义用于在代码中使用这些文件类型，以便在程序中正确地处理和操作SFTP文件。


```cpp
#define LIBSSH2_SFTP_TYPE_REGULAR           1
#define LIBSSH2_SFTP_TYPE_DIRECTORY         2
#define LIBSSH2_SFTP_TYPE_SYMLINK           3
#define LIBSSH2_SFTP_TYPE_SPECIAL           4
#define LIBSSH2_SFTP_TYPE_UNKNOWN           5
#define LIBSSH2_SFTP_TYPE_SOCKET            6
#define LIBSSH2_SFTP_TYPE_CHAR_DEVICE       7
#define LIBSSH2_SFTP_TYPE_BLOCK_DEVICE      8
#define LIBSSH2_SFTP_TYPE_FIFO              9

/*
 * Reproduce the POSIX file modes here for systems that are not POSIX
 * compliant.
 *
 * These is used in "permissions" of "struct _LIBSSH2_SFTP_ATTRIBUTES"
 */
```

这段代码定义了一系列标识文件类型的常量，用于标识SFTP文件的类型。包括：LIBSSH2_SFTP_S_IFMT（文件类型）、LIBSSH2_SFTP_S_IFIFO（命名管道）、LIBSSH2_SFTP_S_IFCHR（字符设备）、LIBSSH2_SFTP_S_IFDIR（目录）、LIBSSH2_SFTP_S_IFBLK（块设备）、LIBSSH2_SFTP_S_IFREG（ regular file ）和LIBSSH2_SFTP_S_IFLNK（符号链接文件）。

另外，还定义了一系列文件模式，包括：LIBSSH2_SFTP_S_IRWXU（只读、可写、执行，所有者权限）、LIBSSH2_SFTP_S_IRUSR（只读、所有者权限）、LIBSSH2_SFTP_S_IWUSR（可读、所有者权限）。

这些代码定义了SFTP文件类型标识和文件模式，用于标识文件系统操作。


```cpp
/* File type */
#define LIBSSH2_SFTP_S_IFMT         0170000     /* type of file mask */
#define LIBSSH2_SFTP_S_IFIFO        0010000     /* named pipe (fifo) */
#define LIBSSH2_SFTP_S_IFCHR        0020000     /* character special */
#define LIBSSH2_SFTP_S_IFDIR        0040000     /* directory */
#define LIBSSH2_SFTP_S_IFBLK        0060000     /* block special */
#define LIBSSH2_SFTP_S_IFREG        0100000     /* regular */
#define LIBSSH2_SFTP_S_IFLNK        0120000     /* symbolic link */
#define LIBSSH2_SFTP_S_IFSOCK       0140000     /* socket */

/* File mode */
/* Read, write, execute/search by owner */
#define LIBSSH2_SFTP_S_IRWXU        0000700     /* RWX mask for owner */
#define LIBSSH2_SFTP_S_IRUSR        0000400     /* R for owner */
#define LIBSSH2_SFTP_S_IWUSR        0000200     /* W for owner */
```



这段代码是一个头文件，定义了几个 macro，用于定义 Sftp 文件的权限。

具体来说，这些宏的含义如下：

- LIBSSH2_SFTP_S_IXUSR：表示只读权限，即文件只能被读取，不能修改。
- LIBSSH2_SFTP_S_IRWXG：表示读写权限，包括对其他人的写入权限，但本文件自己不能修改。
- LIBSSH2_SFTP_S_IRGRP：表示对其他组的读写权限，但本文件自己不能修改。
- LIBSSH2_SFTP_S_IWGRP：表示对其他人的读写权限，但本文件自己不能修改。
- LIBSSH2_SFTP_S_IXGRP：表示只写权限，即文件只能被修改，不能读取。
- LIBSSH2_SFTP_S_IRWXO：表示只读权限，包括对其他人的写入权限，但本文件自己不能修改。
- LIBSSH2_SFTP_S_IROTH：表示对其他人的读写权限，但本文件自己不能修改。
- LIBSSH2_SFTP_S_IWOTH：表示对其他人的读写权限，但本文件自己不能修改。
- LIBSSH2_SFTP_S_IXOTH：表示只写权限，即文件只能被修改，不能读取。
- LIBSSH2_SFTP_S_ISLNK：判断文件是否可执行文件(.sln)，如果是，则执行飞文件的所有操作。

这些宏可以通过在编译时选项中指定 libssh2-sftp-dev 库来使用。


```cpp
#define LIBSSH2_SFTP_S_IXUSR        0000100     /* X for owner */
/* Read, write, execute/search by group */
#define LIBSSH2_SFTP_S_IRWXG        0000070     /* RWX mask for group */
#define LIBSSH2_SFTP_S_IRGRP        0000040     /* R for group */
#define LIBSSH2_SFTP_S_IWGRP        0000020     /* W for group */
#define LIBSSH2_SFTP_S_IXGRP        0000010     /* X for group */
/* Read, write, execute/search by others */
#define LIBSSH2_SFTP_S_IRWXO        0000007     /* RWX mask for other */
#define LIBSSH2_SFTP_S_IROTH        0000004     /* R for other */
#define LIBSSH2_SFTP_S_IWOTH        0000002     /* W for other */
#define LIBSSH2_SFTP_S_IXOTH        0000001     /* X for other */

/* macros to check for specific file types, added in 1.2.5 */
#define LIBSSH2_SFTP_S_ISLNK(m) \
  (((m) & LIBSSH2_SFTP_S_IFMT) == LIBSSH2_SFTP_S_IFLNK)
```

这段代码定义了一系列SFTP文件传输标志，用于标识文件在SFTP网络中的传输特性。这些标志是为了在SSH2协议中通过SFTP协议传输文件而制定的。

具体来说，这些标志定义了以下内容：

* LIBSSH2_SFTP_S_ISREG：文件是否为注册文件（即普通文件）
* LIBSSH2_SFTP_S_ISDIR：文件是否为目录文件
* LIBSSH2_SFTP_S_ISCHR：文件是否为字符设备
* LIBSSH2_SFTP_S_ISBLK：文件是否为块设备
* LIBSSH2_SFTP_S_ISFIFO：文件是否为输入/输出设备（如文件）
* LIBSSH2_SFTP_S_ISSOCK：文件是否为套接字（即Socket）

通过使用这些标志，用户可以更方便地管理SFTP文件，并实现不同类型的文件传输。


```cpp
#define LIBSSH2_SFTP_S_ISREG(m) \
  (((m) & LIBSSH2_SFTP_S_IFMT) == LIBSSH2_SFTP_S_IFREG)
#define LIBSSH2_SFTP_S_ISDIR(m) \
  (((m) & LIBSSH2_SFTP_S_IFMT) == LIBSSH2_SFTP_S_IFDIR)
#define LIBSSH2_SFTP_S_ISCHR(m) \
  (((m) & LIBSSH2_SFTP_S_IFMT) == LIBSSH2_SFTP_S_IFCHR)
#define LIBSSH2_SFTP_S_ISBLK(m) \
  (((m) & LIBSSH2_SFTP_S_IFMT) == LIBSSH2_SFTP_S_IFBLK)
#define LIBSSH2_SFTP_S_ISFIFO(m) \
  (((m) & LIBSSH2_SFTP_S_IFMT) == LIBSSH2_SFTP_S_IFIFO)
#define LIBSSH2_SFTP_S_ISSOCK(m) \
  (((m) & LIBSSH2_SFTP_S_IFMT) == LIBSSH2_SFTP_S_IFSOCK)

/* SFTP File Transfer Flags -- (e.g. flags parameter to sftp_open())
 * Danger will robinson... APPEND doesn't have any effect on OpenSSH servers */
```

这段代码是定义了SFTP文件的libssh2_fxf结构体。

libssh2_fxf结构体包含了SFTP文件中各个字段的信息，包括文件类型、文件属性、文件操作等。通过定义这些常量，可以方便地在程序中使用SFTP文件操作。

例如，通过定义LIBSSH2_FXF_READ，可以实现在文件中读取二进制数据；定义LIBSSH2_FXF_WRITE，可以实现在文件中写入二进制数据；定义LIBSSH2_FXF_APPEND，可以实现在文件中追加数据。


```cpp
#define LIBSSH2_FXF_READ                        0x00000001
#define LIBSSH2_FXF_WRITE                       0x00000002
#define LIBSSH2_FXF_APPEND                      0x00000004
#define LIBSSH2_FXF_CREAT                       0x00000008
#define LIBSSH2_FXF_TRUNC                       0x00000010
#define LIBSSH2_FXF_EXCL                        0x00000020

/* SFTP Status Codes (returned by libssh2_sftp_last_error() ) */
#define LIBSSH2_FX_OK                       0UL
#define LIBSSH2_FX_EOF                      1UL
#define LIBSSH2_FX_NO_SUCH_FILE             2UL
#define LIBSSH2_FX_PERMISSION_DENIED        3UL
#define LIBSSH2_FX_FAILURE                  4UL
#define LIBSSH2_FX_BAD_MESSAGE              5UL
#define LIBSSH2_FX_NO_CONNECTION            6UL
```

这段代码定义了一系列常量，用于表示与SSH2文件传输有关的问题。具体解释如下：

1. LIBSSH2_FX_CONNECTION_LOST：表示连接已丢失，通常由于远程主机没有响应或连接超时而引起。
2. LIBSSH2_FX_OP_UNSUPPORTED：表示SSH2操作不支持某种特定的操作，例如无法创建文件或删除文件。
3. LIBSSH2_FX_INVALID_HANDLE：表示无效的文件句柄，可能是因为访问被另一个进程安全地锁定或由于其他原因导致。
4. LIBSSH2_FX_NO_SUCH_PATH：表示指定的路径不存在。
5. LIBSSH2_FX_FILE_ALREADY_EXISTS：表示文件已存在，且这是用户已知的文件名。
6. LIBSSH2_FX_WRITE_PROTECT：表示正在尝试写入受保护的文件，通常由于没有足够的权限而引起。
7. LIBSSH2_FX_NO_MEDIA：表示没有可用的媒体设备(如磁盘驱动器或USB驱动器)。
8. LIBSSH2_FX_NO_SPACE_ON_FILESYSTEM：表示没有可用空间在文件系统上。
9. LIBSSH2_FX_QUOTA_EXCEEDED：表示设置的配额已用尽。
10. LIBSSH2_FX_UNKNOWN_PRINCIPLE：表示SSH2连接在配置过程中出现未知的问题，可能由于用户输入的密码或连接类型不正确而引起。
11. LIBSSH2_FX_UNKNOWN_PRINCIPAL：表示SSH2连接在配置过程中出现未知的问题，可能由于用户输入的密码或连接类型不正确而引起。
12. LIBSSH2_FX_LOCK_CONFLICT：表示在文件系统中发生了锁定冲突，通常由于两个或更多进程尝试在同一时间写入同一个文件而引起。
13. LIBSSH2_FX_LOCK_CONFLICT：表示在文件系统中发生了锁定冲突，通常由于两个或更多进程尝试在同一时间写入同一个文件而引起。
14. LIBSSH2_FX_DIR_NOT_EMPTY：表示指定的目录不空，可能是因为文件已存在。
15. LIBSSH2_FX_NOT_A_DIRECTORY：表示指定的目录不正确，可能是因为文件已存在。


```cpp
#define LIBSSH2_FX_CONNECTION_LOST          7UL
#define LIBSSH2_FX_OP_UNSUPPORTED           8UL
#define LIBSSH2_FX_INVALID_HANDLE           9UL
#define LIBSSH2_FX_NO_SUCH_PATH             10UL
#define LIBSSH2_FX_FILE_ALREADY_EXISTS      11UL
#define LIBSSH2_FX_WRITE_PROTECT            12UL
#define LIBSSH2_FX_NO_MEDIA                 13UL
#define LIBSSH2_FX_NO_SPACE_ON_FILESYSTEM   14UL
#define LIBSSH2_FX_QUOTA_EXCEEDED           15UL
#define LIBSSH2_FX_UNKNOWN_PRINCIPLE        16UL /* Initial mis-spelling */
#define LIBSSH2_FX_UNKNOWN_PRINCIPAL        16UL
#define LIBSSH2_FX_LOCK_CONFlICT            17UL /* Initial mis-spelling */
#define LIBSSH2_FX_LOCK_CONFLICT            17UL
#define LIBSSH2_FX_DIR_NOT_EMPTY            18UL
#define LIBSSH2_FX_NOT_A_DIRECTORY          19UL
```

这段代码定义了一系列头文件和常量，它们在 SFTP（Secure File System）应用程序中定义了 SFTP 数据的结构和函数。以下是代码主要部分的解释：

1.宏定义：
```cppbash
#define LIBSSH2_FX_INVALID_FILENAME        20UL
#define LIBSSH2_FX_LINK_LOOP                21UL
```
这两行定义了两个宏，分别表示文件无效标识（20ul）和链接循环（21ul）。它们在程序中可能被用来检查文件是否成功打开或关闭。

2.常量：
```cppbash
LIBSSH2SFTP_EAGAIN LIBSSH2_ERROR_EAGAIN
```
这两个常量定义了 SFTP 错误代码的两种标识，它们将在函数中用于判断错误类型。

3.函数声明：
```cppc
LIBSSH2_API LIBSSH2_SFTP *libssh2_sftp_init(LIBSSH2_SESSION *session);
LIBSSH2_API int libssh2_sftp_shutdown(LIBSSH2_SFTP *sftp);
LIBSSH2_API unsigned long libssh2_sftp_last_error(LIBSSH2_SFTP *sftp);
LIBSSH2_API LIBSSH2_CHANNEL *libssh2_sftp_get_channel(LIBSSH2_SFTP *sftp);
```
这些函数声明了 SFTP 实例的初始化、关闭和错误处理等功能。

4.函数实现：
```cppscss
LIBSSH2_API LIBSSH2_SFTP *libssh2_sftp_init(LIBSSH2_SESSION *session) {
   int ret；
   char filename[256];
   unsigned long flags = SFTP_READ | SFTP_WRITE;
   unsigned long last_error = LIBSSH2_SFTP_ERROR;
   LIBSSH2_CHANNEL *channel = NULL;

   ret = SFTP_connect("测试文件.txt", session, SFTP_READ | SFTP_WRITE, 0);
   if (ret != 0) {
       last_error = LIBSSH2_SFTP_ERROR;
       goto end;
   }

   // 读取文件并执行其他操作

   ret = SFTP_write(session, "测试文件.txt", sizeof(filename), filename);
   if (ret != 0) {
       last_error = LIBSSH2_SFTP_ERROR;
       goto end;
   }

   ret = SFTP_commit(session, filename, sizeof(filename), 0, 0);
   if (ret != 0) {
       last_error = LIBSSH2_SFTP_ERROR;
       goto end;
   }

   channel = libssh2_sftp_get_channel(session);
   if (channel == NULL) {
       last_error = LIBSSH2_SFTP_ERROR;
       goto end;
   }

   return libssh2_sftp_init(session);
}

LIBSSH2_API int libssh2_sftp_shutdown(LIBSSH2_SFTP *sftp) {
   int ret;
   unsigned long last_error = LIBSSH2_SFTP_ERROR;
   char filename[256];

   ret = SFTP_connect("测试文件.txt", sftp, SFTP_READ | SFTP_WRITE, 0);
   if (ret != 0) {
       last_error = LIBSSH2_SFTP_ERROR;
       goto end;
   }

   // 读取文件并执行其他操作

   ret = SFTP_write(sftp, "测试文件.txt", sizeof(filename), filename);
   if (ret != 0) {
       last_error = LIBSSH2_SFTP_ERROR;
       goto end;
   }

   ret = SFTP_commit(sftp, filename, sizeof(filename), 0, 0);
   if (ret != 0) {
       last_error = LIBSSH2_SFTP_ERROR;
       goto end;
   }

   ret = SFTP_remove(sftp, filename, sizeof(filename));
   if (ret != 0) {
       last_error = LIBSSH2_SFTP_ERROR;
       goto end;
   }

   ret = SFTP_close(sftp);
   if (ret != 0) {
       last_error = LIBSSH2_SFTP_ERROR;
       goto end;
   }

   last_error = 0;
   return 0;
}

LIBSSH2_API unsigned long libssh2_sftp_last_error(LIBSSH2_SFTP *sftp) {
   return sftp->last_error;
}

LIBSSH2_API LIBSSH2_CHANNEL *libssh2_sftp_get_channel(LIBSSH2_SFTP *sftp) {
   return sftp->channel;
}
```
这些函数的实现主要集中在初始化和关闭文件操作上。函数 `libssh2_sftp_init` 负责连接到文件并执行写入、读取等操作，函数 `libssh2_sftp_shutdown` 负责关闭文件操作，并执行一些清理工作，函数 `libssh2_sftp_last_error` 用于返回错误代码，函数 `libssh2_sftp_get_channel` 用于获取当前打开的频道。

这些函数的存在使得 SFTP 能够初始化、执行文件读写操作、关闭文件操作，并且在函数调用出错时能够正确地返回错误代码，从而使得程序更加健壮和可靠。


```cpp
#define LIBSSH2_FX_INVALID_FILENAME         20UL
#define LIBSSH2_FX_LINK_LOOP                21UL

/* Returned by any function that would block during a read/write operation */
#define LIBSSH2SFTP_EAGAIN LIBSSH2_ERROR_EAGAIN

/* SFTP API */
LIBSSH2_API LIBSSH2_SFTP *libssh2_sftp_init(LIBSSH2_SESSION *session);
LIBSSH2_API int libssh2_sftp_shutdown(LIBSSH2_SFTP *sftp);
LIBSSH2_API unsigned long libssh2_sftp_last_error(LIBSSH2_SFTP *sftp);
LIBSSH2_API LIBSSH2_CHANNEL *libssh2_sftp_get_channel(LIBSSH2_SFTP *sftp);

/* File / Directory Ops */
LIBSSH2_API LIBSSH2_SFTP_HANDLE *
libssh2_sftp_open_ex(LIBSSH2_SFTP *sftp,
                     const char *filename,
                     unsigned int filename_len,
                     unsigned long flags,
                     long mode, int open_type);
```

这段代码定义了两个头文件，分别是`libssh2_sftp_open`和`libssh2_sftp_opendir`，以及一个函数`libssh2_sftp_read`和一个函数`libssh2_sftp_readdir_ex`。它们的作用如下：

1. `libssh2_sftp_open`函数：
  - `sftp`：文件系统文件类型，表示打开的文件是块设备（如磁盘）。
  - `filename`：要打开的文件名，包括扩展名。
  - `flags`：额外的文件元数据，包括客户控制的阿斥（ACL）。
  - `mode`：文件模式，包括读/写/追加。
  - `libssh2_sftp_opendir`：打开文件系统的目录。

2. `libssh2_sftp_read`函数：
  - `handle`：文件系统文件句柄，表示已打开的文件系统。
  - `buffer`：用于存储数据缓冲区，可以是输入或输出。
  - `buffer_maxlen`：缓冲区最大长度，从0开始。
  - `LIBSSH2_SFTP_READFILES`：表示从文件系统中读取的文件个数。

3. `libssh2_sftp_readdir_ex`函数：
  - `handle`：文件系统文件句柄，表示已打开的文件系统。
  - `path`：要读取的文件路径，包括目录。
  - `buffer`：用于存储数据缓冲区，可以是输入或输出。
  - `longentry_maxlen`：最大文件路径长度，从0开始。
  - `attrs`：包含文件元数据的结构体，用于传递文件元数据。
  - `LIBSSH2_SFTP_ATTRIBUTES`：表示文件系统的属性。
  - `libssh2_sftp_read`：调用从文件系统中读取文件的函数。


```cpp
#define libssh2_sftp_open(sftp, filename, flags, mode)                  \
    libssh2_sftp_open_ex((sftp), (filename), strlen(filename), (flags), \
                         (mode), LIBSSH2_SFTP_OPENFILE)
#define libssh2_sftp_opendir(sftp, path) \
    libssh2_sftp_open_ex((sftp), (path), strlen(path), 0, 0, \
                         LIBSSH2_SFTP_OPENDIR)

LIBSSH2_API ssize_t libssh2_sftp_read(LIBSSH2_SFTP_HANDLE *handle,
                                      char *buffer, size_t buffer_maxlen);

LIBSSH2_API int libssh2_sftp_readdir_ex(LIBSSH2_SFTP_HANDLE *handle, \
                                        char *buffer, size_t buffer_maxlen,
                                        char *longentry,
                                        size_t longentry_maxlen,
                                        LIBSSH2_SFTP_ATTRIBUTES *attrs);
```

这段代码定义了三个函数，分别是：

1. libssh2_sftp_readdir：用于读取远程服务器上的目录列表，并返回包含目录列表的长度。
2. libssh2_sftp_write：用于向远程服务器写入文件内容。
3. libssh2_sftp_fsync：用于同步文件内容和远程服务器上的文件系统。

以下是每个函数的详细解释：

1. libssh2_sftp_readdir(handle, buffer, buffer_maxlen, attrs)：函数名，参数说明。handle：远程服务器上的handle，用于读取目录列表；buffer：接收目录列表的内存缓冲区；buffer_maxlen：最多可以容纳的缓冲区字节数；attrs：包含目录列表中每个目录对象的属性，如访问权限等。函数实现部分如下：
```cppc
   int libssh2_sftp_readdir_ex(handle, buffer, buffer_maxlen, attrs, sftp_resource *res);
```
其中，handle为远程服务器上的handle，buffer为接收目录列表的内存缓冲区，buffer_maxlen为最多可以容纳的缓冲区字节数，attrs为包含目录列表中每个目录对象的属性，res为目录列表资源的返回值指针。通过调用libssh2_sftp_readdir_ex函数，可以获取远程服务器上的目录列表，并返回包含目录列表的长度。

2. libssh2_sftp_write：函数名，参数说明。handle为远程服务器上的handle，用于写入文件内容；buffer为需要写入的文件内容，大小为count；函数实现部分如下：
```cppc
   int libssh2_sftp_write(handle, buffer, count);
```
通过调用libssh2_sftp_write函数，可以将文件内容写入远程服务器上的handle，size为count。

3. libssh2_sftp_fsync：函数名，参数说明。handle为远程服务器上的handle，用于同步文件内容和远程服务器上的文件系统；buffer为需要同步的文件内容，大小为count；函数实现部分如下：
```cppc
   int libssh2_sftp_fsync(handle, buffer, count);
```
通过调用libssh2_sftp_fsync函数，可以同步文件内容和远程服务器上的文件系统。


```cpp
#define libssh2_sftp_readdir(handle, buffer, buffer_maxlen, attrs)      \
    libssh2_sftp_readdir_ex((handle), (buffer), (buffer_maxlen), NULL, 0, \
                            (attrs))

LIBSSH2_API ssize_t libssh2_sftp_write(LIBSSH2_SFTP_HANDLE *handle,
                                       const char *buffer, size_t count);
LIBSSH2_API int libssh2_sftp_fsync(LIBSSH2_SFTP_HANDLE *handle);

LIBSSH2_API int libssh2_sftp_close_handle(LIBSSH2_SFTP_HANDLE *handle);
#define libssh2_sftp_close(handle) libssh2_sftp_close_handle(handle)
#define libssh2_sftp_closedir(handle) libssh2_sftp_close_handle(handle)

LIBSSH2_API void libssh2_sftp_seek(LIBSSH2_SFTP_HANDLE *handle, size_t offset);
LIBSSH2_API void libssh2_sftp_seek64(LIBSSH2_SFTP_HANDLE *handle,
                                     libssh2_uint64_t offset);
```

这段代码是一个定义，定义了两个名为libssh2_sftp_rewind和libssh2_sftp_tell的函数，以及一个名为libssh2_sftp_fstat_ex的函数。

libssh2_sftp_rewind函数用于将handle文件对象从文件中读取回到开始位置，并返回从文件中读取的起始位置偏移量。

libssh2_sftp_tell函数用于返回文件中从handle文件对象位置开始偏移量，单位是字节。

libssh2_sftp_fstat_ex函数用于获取文件对象的状态，包括文件类型、硬链接数、符号链接数、访问权限、所有者权限、group权限、读权限、写权限、执行权限和时间戳。

libssh2_sftp_fsetstat函数用于设置文件对象的状态，包括文件类型、硬链接数、符号链接数、访问权限、所有者权限、group权限、读权限、写权限、执行权限和时间戳。


```cpp
#define libssh2_sftp_rewind(handle) libssh2_sftp_seek64((handle), 0)

LIBSSH2_API size_t libssh2_sftp_tell(LIBSSH2_SFTP_HANDLE *handle);
LIBSSH2_API libssh2_uint64_t libssh2_sftp_tell64(LIBSSH2_SFTP_HANDLE *handle);

LIBSSH2_API int libssh2_sftp_fstat_ex(LIBSSH2_SFTP_HANDLE *handle,
                                      LIBSSH2_SFTP_ATTRIBUTES *attrs,
                                      int setstat);
#define libssh2_sftp_fstat(handle, attrs) \
    libssh2_sftp_fstat_ex((handle), (attrs), 0)
#define libssh2_sftp_fsetstat(handle, attrs) \
    libssh2_sftp_fstat_ex((handle), (attrs), 1)

/* Miscellaneous Ops */
LIBSSH2_API int libssh2_sftp_rename_ex(LIBSSH2_SFTP *sftp,
                                       const char *source_filename,
                                       unsigned int srouce_filename_len,
                                       const char *dest_filename,
                                       unsigned int dest_filename_len,
                                       long flags);
```

这段代码定义了两个函数，libssh2_sftp_rename和libssh2_sftp_unlink_ex。它们的作用是文件复制和 unlink 函数。

1. libssh2_sftp_rename函数：将一个 SFTP file 中的文件名（sourcefile）复制到另一个 SFTP file 中的目标文件名（destfile）中。可以覆盖目标文件名，原子操作并支持 preserve file metadata。函数输出的整数表示状态变更。

2. libssh2_sftp_unlink_ex函数：从 SFTP file 中删除一个指定的文件名（filename）。如果文件已存在，函数将文件复制到 SFTP file 中的临时文件，并使用 LIBSSH2_SFTP_RENAME_OVERWRITE 标志覆盖文件。如果文件不存在，函数将创建一个新文件，并使用 LIBSSH2_SFTP_RENAME_ATOMIC 标志原子地写入文件。函数输出的整数表示状态变更。

3. libssh2_sftp_fstatvfs函数：返回 SFTP file 对应的文件系统的元数据，包括文件类型、文件大小、文件权限等。


```cpp
#define libssh2_sftp_rename(sftp, sourcefile, destfile) \
    libssh2_sftp_rename_ex((sftp), (sourcefile), strlen(sourcefile), \
                           (destfile), strlen(destfile),                \
                           LIBSSH2_SFTP_RENAME_OVERWRITE | \
                           LIBSSH2_SFTP_RENAME_ATOMIC | \
                           LIBSSH2_SFTP_RENAME_NATIVE)

LIBSSH2_API int libssh2_sftp_unlink_ex(LIBSSH2_SFTP *sftp,
                                       const char *filename,
                                       unsigned int filename_len);
#define libssh2_sftp_unlink(sftp, filename) \
    libssh2_sftp_unlink_ex((sftp), (filename), strlen(filename))

LIBSSH2_API int libssh2_sftp_fstatvfs(LIBSSH2_SFTP_HANDLE *handle,
                                      LIBSSH2_SFTP_STATVFS *st);

```

这段代码定义了两个函数，一个是`libssh2_sftp_statvfs()`，另一个是`libssh2_sftp_mkdir_ex()`和`libssh2_sftp_rmdir_ex()`。

`libssh2_sftp_statvfs()`函数接收一个`LIBSSH2_SFTP`类型的对象一个路径名（`const char *`类型）和一个路径长度（`size_t`类型），然后返回一个指向`LIBSSH2_SFTP_STATVFS`类型的指针，该类型表示文件系统的统计信息，包括文件和目录的个数以及文件和目录的磁盘占用空间。

`libssh2_sftp_mkdir_ex()`函数和`libssh2_sftp_mkdir()`函数类似，但是使用了C库函数`mkdir()`的实现。它接收一个`LIBSSH2_SFTP`类型的对象一个路径名（`const char *`类型）、一个模式（`unsigned int`类型）和一个路径长度（`size_t`类型），然后返回一个整数，表示是否成功创建了一个目录以及创建的目录的路径长度。

`libssh2_sftp_rmdir_ex()`函数和`libssh2_sftp_rmdir()`函数类似，但是使用了C库函数`rmdir()`的实现。它接收一个`LIBSSH2_SFTP`类型的对象一个路径名（`const char *`类型）、一个路径长度（`size_t`类型），然后返回一个整数，表示是否成功删除了一个目录以及删除的目录的路径长度。注意，第二个参数必须是一个非负整数。


```cpp
LIBSSH2_API int libssh2_sftp_statvfs(LIBSSH2_SFTP *sftp,
                                     const char *path,
                                     size_t path_len,
                                     LIBSSH2_SFTP_STATVFS *st);

LIBSSH2_API int libssh2_sftp_mkdir_ex(LIBSSH2_SFTP *sftp,
                                      const char *path,
                                      unsigned int path_len, long mode);
#define libssh2_sftp_mkdir(sftp, path, mode) \
    libssh2_sftp_mkdir_ex((sftp), (path), strlen(path), (mode))

LIBSSH2_API int libssh2_sftp_rmdir_ex(LIBSSH2_SFTP *sftp,
                                      const char *path,
                                      unsigned int path_len);
#define libssh2_sftp_rmdir(sftp, path) \
    libssh2_sftp_rmdir_ex((sftp), (path), strlen(path))

```

这段代码定义了一个名为 libssh2_sftp_stat_ex 的函数，它是 libssh2_sftp_stat 的函数的别名。这个函数接受三个参数：一个 SFTP 对象，一个路径，以及一个表示文件统计信息的数据结构。这个函数返回一个整数，表示文件的统计信息，主要有文件权限、硬链接、符号链接、共享链接、文件系统类型、路径类型、内核路径类型等。

libssh2_sftp_stat_ex 是 libssh2_sftp_stat 的函数，它的参数列表如下：

- sftp：SFTP 对象
- path：文件或目录的路径
- path_len：文件或目录的路径长度
- stat_type：统计类型，0 表示文件，1 表示目录，2 表示 SFTP 对象
- attrs：包含文件属性的数据结构，如下：
 - file_type：文件的类型，0 表示普通文件，1 表示二进制文件，2 表示链接文件
 - file_system_type：文件系统类型，0 表示原始类型，1 表示 BSD 类型，2 表示NTFS 类型
 - file_attributes：包含文件属性的数据结构，如下：
   - file_mode：文件 mode，0 表示读权限，1 表示写权限，2 表示执行权限
   - file_type：文件的类型，0 表示普通文件，1 表示二进制文件，2 表示链接文件
   - file_size：文件大小，单位为字节
   - last_write_time：文件的最后写时间，如果有则返回，否则为 0。

libssh2_sftp_lstat 是 libssh2_sftp_stat_ex 的别名，它与 libssh2_sftp_stat 拥有相同的函数签名。这个函数的参数列表与上面定义的函数的参数列表相同。


```cpp
LIBSSH2_API int libssh2_sftp_stat_ex(LIBSSH2_SFTP *sftp,
                                     const char *path,
                                     unsigned int path_len,
                                     int stat_type,
                                     LIBSSH2_SFTP_ATTRIBUTES *attrs);
#define libssh2_sftp_stat(sftp, path, attrs) \
    libssh2_sftp_stat_ex((sftp), (path), strlen(path), LIBSSH2_SFTP_STAT, \
                         (attrs))
#define libssh2_sftp_lstat(sftp, path, attrs) \
    libssh2_sftp_stat_ex((sftp), (path), strlen(path), LIBSSH2_SFTP_LSTAT, \
                         (attrs))
#define libssh2_sftp_setstat(sftp, path, attrs) \
    libssh2_sftp_stat_ex((sftp), (path), strlen(path), LIBSSH2_SFTP_SETSTAT, \
                         (attrs))

```

这段代码是一个用于在 libssh2_sftp 库中实现符号链接的函数。符号链接是一种特殊的链接，它允许您创建一个指向目标文件的链接，并且这个链接在目标文件被修改时不会丢失。

具体来说，这段代码实现了一个名为 libssh2_sftp_symlink_ex 的函数，它接收一个 SFTP 对象、原始文件路径、目标文件路径长度、目标文件路径、链接类型 和 SFTP 系统调用函数中的 link_type 参数。函数首先调用 SFTP 系统调用函数中的 libssh2_sftp_symlink 函数，并将输入参数传递给该函数。然后，函数根据输入参数中的路径和目标文件路径长度计算出目标文件路径，并将其与原始文件路径和链接类型一起作为参数传递给 libssh2_sftp 函数。

这里定义了三个函数，它们都使用 libssh2_sftp_symlink_ex 函数实现符号链接。这些函数的具体实现类似于在创建符号链接时将原始文件路径和目标文件路径传递给 libssh2_sftp 函数。


```cpp
LIBSSH2_API int libssh2_sftp_symlink_ex(LIBSSH2_SFTP *sftp,
                                        const char *path,
                                        unsigned int path_len,
                                        char *target,
                                        unsigned int target_len,
                                        int link_type);
#define libssh2_sftp_symlink(sftp, orig, linkpath) \
    libssh2_sftp_symlink_ex((sftp), (orig), strlen(orig), (linkpath), \
                            strlen(linkpath), LIBSSH2_SFTP_SYMLINK)
#define libssh2_sftp_readlink(sftp, path, target, maxlen) \
    libssh2_sftp_symlink_ex((sftp), (path), strlen(path), (target), (maxlen), \
    LIBSSH2_SFTP_READLINK)
#define libssh2_sftp_realpath(sftp, path, target, maxlen) \
    libssh2_sftp_symlink_ex((sftp), (path), strlen(path), (target), (maxlen), \
                            LIBSSH2_SFTP_REALPATH)

```

这段代码是一个C语言的预处理指令，用于检查当前源文件是否为C语言编写的。如果当前源文件是C语言编写的，则会编译并嵌入以下代码：
```cpp
#ifdef __cplusplus
/* extern "C" */
#endif
```
这会定义一个名为`__cplusplus`的符号，其值为1。这将允许编译器在编译之前检查`__cplusplus`是否已被定义。如果这个符号已经被定义，则编译器会编译`__cplusplus`而不是简单的`1`，从而提高编译效率。

否则，这段代码并不会被编译，因为它只是一个简单的注释。


```cpp
#ifdef __cplusplus
} /* extern "C" */
#endif

#endif /* LIBSSH2_SFTP_H */

```

# `libssh2/nw/keepscreen.c`

这段代码是一个简单的方式来保持屏幕在NLM结束时保持打开。当NLM结束时，它会调用这个函数，所以你不需要手动调用它。这个函数的行为类似于clib中的行为。

函数体中首先定义了两个变量row和col，用于获取屏幕的行和列。接着，函数使用了GetScreenSize函数来获取屏幕的大小，然后使用gotorowcol函数将行数-1和列数0的屏幕位置标记为当前行和列。最后，函数在屏幕上显示了一个文本框，提示用户按任意键来关闭屏幕，并使用getcharacter函数来接收用户输入的字符。

总的来说，这段代码是为了在NLM结束时保持屏幕的显示，以便用户可以继续使用它而无需手动关闭。


```cpp
/* Simple _NonAppStop() implementation which can be linked to your 
 * NLM in order to keep the screen open when the NLM terminates
 * (the good old clib behaviour).
 * You dont have to call it, its done automatically from LibC.
 *
 * 2004-Aug-11  by Guenter Knauf 
 *
 * URL: http://www.gknw.net/development/mk_nlm/
 */
 
#include <stdio.h>
#include <screen.h>

void _NonAppStop()
{
    uint16_t row, col;
    
    GetScreenSize(&row, &col);
    gotorowcol(row-1, 0);
    /* pressanykey(); */
    printf("<Press any key to close screen> ");
    getcharacter();
}



```

# `libssh2/nw/nwlib.c`

这段代码是一个名为"stub.h"的文件头文件，属于"Universal NetWare library"库。它包含了一些定义和声明，用于定义该库中通用的函数和数据结构。

首先，通过#ifdef NETWARE和#define NETWARE来定义是否支持NetWare库。接着，通过#include <stdlib.h>来引入标准库中的stdlib.h函数。

然后，通过#ifdef __NOVELL_LIBC__来检查是否支持Novell库。如果支持，通过#include <errno.h>、#include <string.h>和#include <library.h>来引入Novell库中的errno.h、string.h和library.h函数。

接下来，通过#include "netware_api.h"来引入NetWare库中的全局函数和数据结构。最后，通过#include "stub_api.h"来引入自己定义的函数和数据结构。


```cpp
/*********************************************************************
 * Universal NetWare library stub.                                   *
 * written by Ulrich Neuman and given to OpenSource copyright-free.  *
 * Extended for CLIB support by Guenter Knauf.                       *
 *********************************************************************/

#ifdef NETWARE /* Novell NetWare */

#include <stdlib.h>

#ifdef __NOVELL_LIBC__
/* For native LibC-based NLM we need to register as a real lib. */
#include <errno.h>
#include <string.h>
#include <library.h>
```

这段代码定义了一个名为libthreaddata_t的 struct，该 struct 用于存储线程数据。线程数据是在创建线程时通过调用socket()函数创建的，socket()函数用于创建一个套接字并返回其套接字指针，而线程数据则保存在该套接字指针所指向的内存区域。

线程数据结构体定义了6个成员变量，包括 _errno、twentybytes、x、y、z、tenbytes、perthreadkey和lock。其中，_errno是一个int类型的成员变量，用于记录线程在创建或销毁时遇到的错误码；twentybytes是一个void类型的成员变量，用于存储线程需要分配的20字节空间；x、y、z和tenbytes也是int类型的成员变量，用于记录线程的偏移量和方向；perthreadkey是一个NXKey_t类型的成员变量，用于存储当前线程的唯一标识符；lock是一个NXMutex_t类型的成员变量，用于存储当前线程锁。

另外，该代码还引入了两个头文件，netware.h和screen.h，netware.h是用于定义网络套接字的相关操作，screen.h是用于定义屏幕输出操作的头文件。


```cpp
#include <netware.h>
#include <screen.h>
#include <nks/thread.h>
#include <nks/synch.h>

typedef struct
{
  int     _errno;
  void    *twentybytes;
} libthreaddata_t;

typedef struct
{
  int         x;
  int         y;
  int         z;
  void        *tenbytes;
  NXKey_t     perthreadkey;   /* if -1, no key obtained... */
  NXMutex_t   *lock;
} libdata_t;

```

这段代码定义了几个变量，以及它们的初始化和函数指针。它们可能用于在应用程序中管理共享数据。

- `gLibId` 是一个整数类型的变量，用于标识全局变量。它被初始化为 -1，表示还没有分配到内存。

- `gLibHandle` 是一个指向全局变量的指针。它被初始化为 `(void *) NULL`，表示全局变量在内存中的位置为零。由于全局变量不能在函数内部直接访问，因此这个指针被用来存放全局变量的地址，而不是实际的数据。

- `gAllocTag` 是一个整数类型的变量，用于标识全局变量。它被初始化为 (RTAG_OBJECT_INITIALIZER老)，表示全局变量在运行时分配。

- `gLibLock` 是一个指向全局锁的指针。它被初始化为 `(NXMutex_t) NULL`，表示全局锁在运行时为空。

- `DisposeLibraryData` 是内部库函数，用于释放内存中的数据。它接收两个参数：一个指向数据结构的指针，以及一个内部函数，用于在数据结构中释放数据。

- `DisposeThreadData` 是内部库函数，用于释放线程数据。它接收一个指向数据结构的指针，以及一个内部函数，用于在数据结构中释放数据。

- `GetOrSetUpData` 是内部库函数，用于根据给定的 ID 获取或设置全局变量。它接收两个参数：一个整数和一个指向数据结构的指针，用于存储要读取或设置的数据。函数首先检查给定的 ID 是否存在于全局变量中，如果是，则返回全局变量的值；否则，函数会尝试将新数据存储到全局变量中。


```cpp
int         gLibId      = -1;
void        *gLibHandle = (void *) NULL;
rtag_t      gAllocTag   = (rtag_t) NULL;
NXMutex_t   *gLibLock   = (NXMutex_t *) NULL;

/* internal library function prototypes... */
int     DisposeLibraryData ( void * );
void    DisposeThreadData ( void * );
int     GetOrSetUpData ( int id, libdata_t **data, libthreaddata_t **threaddata );


int _NonAppStart( void        *NLMHandle,
                  void        *errorScreen,
                  const char  *cmdLine,
                  const char  *loadDirPath,
                  size_t      uninitializedDataLength,
                  void        *NLMFileHandle,
                  int         (*readRoutineP)( int conn,
                                               void *fileHandle, size_t offset,
                                               size_t nbytes,
                                               size_t *bytesRead,
                                               void *buffer ),
                  size_t      customDataOffset,
                  size_t      customDataSize,
                  int         messageCount,
                  const char  **messages )
{
  NX_LOCK_INFO_ALLOC(liblock, "Per-Application Data Lock", 0);

```

这段代码是一个名为“__GNUC__”的预处理指令声明，其中包含多个未使用的声明。这些未使用的声明由“#pragma”预处理指令指定，表示它们在编译时可以被忽略。

具体来说，这些未使用的声明包括：

- cmdLine：表示命令行参数的名称，但在此处没有定义。
- loadDirPath：表示要加载的可执行文件所在的目录路径，但在此处没有定义。
- uninitializedDataLength：表示一个未初始化的数据缓冲区的长度，但在此处没有定义。
- NLMFileHandle：表示一个文本文件指针，但在此处没有定义。
- readRoutineP：表示一个读取计划的指针，但在此处没有定义。
- customDataOffset：表示一个自定义数据偏移量的指针，但在此处没有定义。
- customDataSize：表示一个自定义数据的大小，但在此处没有定义。
- messageCount：表示一个要打印的消息的数量，但在此处没有定义。
- messages：表示一个存储消息的数组的指针，但在此处没有定义。

如果这段代码出现在一个C或C++源文件中，那么这些未使用的声明都不会对代码的编译产生影响，因为编译器会忽略它们。但是，在某些脚本或框架中，这些未使用的声明可能会有实际的作用，需要特别注意。


```cpp
#ifndef __GNUC__
#pragma unused(cmdLine)
#pragma unused(loadDirPath)
#pragma unused(uninitializedDataLength)
#pragma unused(NLMFileHandle)
#pragma unused(readRoutineP)
#pragma unused(customDataOffset)
#pragma unused(customDataSize)
#pragma unused(messageCount)
#pragma unused(messages)
#endif

/*
** Here we process our command line, post errors (to the error screen),
** perform initializations and anything else we need to do before being able
```

这段代码是一个在NetWare 系统中的库函数，它的作用是接受调用，如果成功，就返回非零值，否则会失败并返回一个错误码。

具体来说，它首先尝试从内存分配器分配一个名为"<library-name> memory allocations"的资源类型，如果分配失败，就输出错误信息并返回-1。然后，它尝试注册一个名为"<library-name>"的库到内核中，如果注册失败，就输出错误信息并返回-1。接着，它尝试获取一个名为"library data lock"的同步锁，如果获取失败，就输出错误信息并返回-1。最后，如果所有尝试都成功，它返回0。


```cpp
** to accept calls into us. If we succeed, we return non-zero and the NetWare
** Loader will leave us up, otherwise we fail to load and get dumped.
*/
  gAllocTag = AllocateResourceTag(NLMHandle,
                   "<library-name> memory allocations", AllocSignature);

  if (!gAllocTag) {
    OutputToScreen(errorScreen, "Unable to allocate resource tag for "
                     "library memory allocations.\n");
    return -1;
  }

  gLibId = register_library(DisposeLibraryData);

  if (gLibId < -1) {
    OutputToScreen(errorScreen, "Unable to register library with kernel.\n");
    return -1;
  }

  gLibHandle = NLMHandle;

  gLibLock = NXMutexAlloc(0, 0, &liblock);

  if (!gLibLock) {
    OutputToScreen(errorScreen, "Unable to allocate library data lock.\n");
    return -1;
  }

  return 0;
}

```

这段代码定义了一个名为 _NonAppStop 的函数。它是一个无返回值的函数，但会在函数内部执行一些操作。

首先，它使用 unregister_library 函数来卸载分配的资源。unregister_library 函数会将所注册的图书馆中所有命名的资源名称、所分配的内存以及所有对资源分配的引用等所有资源信息返回给调用它的函数。

接着，它使用 NXMutexFree 函数释放了所分配的互斥锁。这个互斥锁在函数中没有明确的说明，但可以推测它是用于确保在多进程或多线程环境下对资源的安全访问。

最后，由于 NetWare 没有要求释放资源，所以该函数不会输出任何错误信息。

整体来看，这段代码的作用是释放之前分配的资源，包括注册的图书馆、互斥锁以及可能定义的其他资源。


```cpp
/*
** Here we clean up any resources we allocated. Resource tags is a big part
** of what we created, but NetWare doesn't ask us to free those.
*/
void _NonAppStop( void )
{
  (void) unregister_library(gLibId);
  NXMutexFree(gLibLock);
}

/*
** This function cannot be the first in the file for if the file is linked
** first, then the check-unload function's offset will be nlmname.nlm+0
** which is how to tell that there isn't one. When the check function is
** first in the linked objects, it is ambiguous. For this reason, we will
```

这段代码定义了一个名为 `_NonAppCheckUnload` 的函数，它的作用是检查是否可以卸载应用程序的数据。如果应用程序的数据被卸载，函数返回非零值，否则返回 0。

在 `GetOrSetUpData` 函数中，它检查两个指针变量 `appData` 和 `threadData` 是否为空，然后尝试获取或设置应用程序的数据。如果这两个指针为空，函数使用 `NX_LOCK_INFO_ALLOC` 函数尝试获取或设置数据锁，如果锁不能被获取或设置，则返回一个非零值。如果锁可以被获取或设置，函数将 `app_data` 和 `thread_data` 指向应用程序的数据。


```cpp
** put it inside this file after the stop function.
**
** Here we check to see if it's alright to ourselves to be unloaded. If not,
** we return a non-zero value. Right now, there isn't any reason not to allow
** it.
*/
int _NonAppCheckUnload( void )
{
  return 0;
}

int GetOrSetUpData(int id, libdata_t **appData, libthreaddata_t **threadData)
{
  int              err;
  libdata_t        *app_data;
  libthreaddata_t  *thread_data;
  NXKey_t          key;
  NX_LOCK_INFO_ALLOC(liblock, "Application Data Lock", 0);

  err         = 0;
  thread_data = (libthreaddata_t *) NULL;

```

这段代码的作用是获取一个名为"id"的应用程序的特定数据，并将其存储为名为"app_data"的整型变量中。如果尝试获取数据时出现错误，该代码将尝试获取并存储应用程序特定信息以支持应用程序。

具体来说，代码首先通过调用函数`get_app_data(id)`获取应用程序数据。如果该函数返回值为零，说明这个应用程序第一次调用，此时代码将尝试创建应用程序特定数据，并将它存储在`app_data`中。如果该函数返回值不为零，则说明这个应用程序已经调用过我们，此时我们需要确保在获取数据之前先获取并存储数据，避免多个线程同时调用应用程序特定代码。因此，代码会尝试获取并存储数据，同时使用一个互斥锁来确保在获取数据之前先获取并存储数据。


```cpp
/*
** Attempt to get our data for the application calling us. This is where we
** store whatever application-specific information we need to carry in support
** of calling applications.
*/
  app_data = (libdata_t *) get_app_data(id);

  if (!app_data) {
/*
** This application hasn't called us before; set up application AND per-thread
** data. Of course, just in case a thread from this same application is calling
** us simultaneously, we better lock our application data-creation mutex. We
** also need to recheck for data after we acquire the lock because WE might be
** that other thread that was too late to create the data and the first thread
** in will have created it.
```

这段代码是一个 C 语言程序，主要作用是确保一个名为 "app_data" 的内存区域被正确初始化和释放，同时创建一个名为 "liblock" 的互斥锁，以确保在 "app_data" 所占据的内存区域中可以进行互斥锁操作。

具体来说，代码首先通过调用函数 `get_app_data(id)` 获取应用程序的数据，如果该函数返回的结果为空，则说明 "app_data" 区域的内存可能已经被释放，需要重新分配内存并初始化。在 `malloc()` 函数调用之后，如果 "app_data" 区域分配成功，则说明该内存区域已经被正确初始化，可以进行互斥锁操作。

代码接下来创建了一个名为 "liblock" 的互斥锁，并使用 `NXMutexAlloc()` 函数将互斥锁分配给调用者，以确保在 "app_data" 所占据的内存区域中可以进行互斥锁操作。如果 `app_data` 区域的内存初始化成功并且分配的内存空间可以用于创建互斥锁，则说明互斥锁已经创建成功。

最后，代码还定义了一个名为 "err" 的变量，用于记录错误信息。如果在函数执行过程中出现错误，可以在代码中通过 `err` 变量进行记录，并输出相关错误信息。


```cpp
*/
    NXLock(gLibLock);

    if (!(app_data = (libdata_t *) get_app_data(id))) {
      app_data = (libdata_t *) malloc(sizeof(libdata_t));

      if (app_data) {
        memset(app_data, 0, sizeof(libdata_t));

        app_data->tenbytes = malloc(10);
        app_data->lock     = NXMutexAlloc(0, 0, &liblock);

        if (!app_data->tenbytes || !app_data->lock) {
          if (app_data->lock)
            NXMutexFree(app_data->lock);

          free(app_data);
          app_data = (libdata_t *) NULL;
          err      = ENOMEM;
        }

        if (app_data) {
```

这段代码的主要作用是获取应用程序数据并将其保存到堆内存中，以便在下次调用函数时可以恢复。首先，通过调用get_app_data()函数获取数据，并将其存储到变量app_data中。然后，如果获取数据时出现错误，则释放已分配的内存并将其设置为空。如果获取数据成功，则创建一个线程特定的数据键，并将键的值存储为app_data中的perthreadkey变量。这样，在下次函数调用时，就可以通过键来获取线程特定的数据了。


```cpp
/*
** Here we burn in the application data that we were trying to get by calling
** get_app_data(). Next time we call the first function, we'll get this data
** we're just now setting. We also go on here to establish the per-thread data
** for the calling thread, something we'll have to do on each application
** thread the first time it calls us.
*/
          err = set_app_data(gLibId, app_data);

          if (err) {
            free(app_data);
            app_data = (libdata_t *) NULL;
            err      = ENOMEM;
          }
          else {
            /* create key for thread-specific data... */
            err = NXKeyCreate(DisposeThreadData, (void *) NULL, &key);

            if (err)                /* (no more keys left?) */
              key = -1;

            app_data->perthreadkey = key;
          }
        }
      }
    }

    NXUnlock(gLibLock);
  }

  if (app_data) {
    key = app_data->perthreadkey;

    if (key != -1 /* couldn't create a key? no thread data */
        && !(err = NXKeyGetValue(key, (void **) &thread_data))
        && !thread_data) {
```

这段代码的主要作用是分配一个20字节的消息，并为调用该函数的线程分配相应的数据。

首先，在函数头部，我们定义了一个名为`thread_data`的变量，并使用`malloc`函数为其分配了20字节的空间。这部分代码是为了确保无论调用该函数的线程是否之前有应用数据，我们都为该线程分配了独立的数据。

接下来，我们检查`thread_data`是否被成功分配。如果是，我们为`twentybytes`变量分配内存，并检查`twentybytes`是否成功分配。如果这两步都成功，我们将`libthreaddata_t`类型的变量标记为`true`，从而为该线程设置了一个有效的`libthreaddata_t`类型的数据结构。

然后，在函数主体中，我们检查是否有成功分配的内存。如果是，我们将其复制到`appData`指向的内存位置，并将`thread_data`指向我们刚刚分配的内存位置。最后，我们检查是否有任何错误发生。如果是，我们释放分配的内存并返回错误码。


```cpp
/*
** Allocate the per-thread data for the calling thread. Regardless of whether
** there was already application data or not, this may be the first call by a
** a new thread. The fact that we allocation 20 bytes on a pointer is not very
** important, this just helps to demonstrate that we can have arbitrarily
** complex per-thread data.
*/
      thread_data = (libthreaddata_t *) malloc(sizeof(libthreaddata_t));

      if (thread_data) {
        thread_data->_errno      = 0;
        thread_data->twentybytes = malloc(20);

        if (!thread_data->twentybytes) {
          free(thread_data);
          thread_data = (libthreaddata_t *) NULL;
          err         = ENOMEM;
        }

        if ((err = NXKeySetValue(key, thread_data))) {
          free(thread_data->twentybytes);
          free(thread_data);
          thread_data = (libthreaddata_t *) NULL;
        }
      }
    }
  }

  if (appData)
    *appData = app_data;

  if (threadData)
    *threadData = thread_data;

  return err;
}

```

这两段代码是在定义两个函数，名为 DisposeLibraryData 和 DisposeThreadData。这两个函数有一个共同点，它们都是接受一个 void 类型的指针参数 data，并在函数内部对 data 指向的内存进行释放。

DisposeLibraryData 是函数，接收一个 void 类型的指针 data，然后获取一个名为 tenbytes 的 void 类型的指针，接着判断 tenbytes 是否为非空，如果是，则释放 tenbytes 指向的内存，最后释放 data 指向的内存。

DisposeThreadData 是另一个函数，与 DisposeLibraryData 类似，但是它接收的是一个 void 类型的指针 data，而不是一个 libdata_t 或 libthreaddata_t 类型的数据结构。这个函数与 DisposeLibraryData 不同的是，它需要手动释放 data 指向的内存。


```cpp
int DisposeLibraryData( void *data )
{
  if (data) {
    void *tenbytes = ((libdata_t *) data)->tenbytes;

    if (tenbytes)
      free(tenbytes);

    free(data);
  }

  return 0;
}

void DisposeThreadData( void *data )
{
  if (data) {
    void *twentybytes = ((libthreaddata_t *) data)->twentybytes;

    if (twentybytes)
      free(twentybytes);

    free(data);
  }
}

```

这段代码是一个C语言程序，主要作用是定义了一个名为“main”的函数。在这个函数中，首先引入了两个头文件：<nwthread.h>和<stdio.h>，然后定义了一个整型变量“ExitThread”，接着对<stdio.h>中的“0”进行整型赋值。最后， ExitThread函数被调用，并使用它的参数“TSR_THREAD”和“0”来表示异常条件。整型变量“ExitThread”的作用是在程序遇到异常情况时，返回一个整型值，通常为0表示程序运行成功，非0表示程序运行失败或出现异常。


```cpp
#else /* __NOVELL_LIBC__ */
/* For native CLib-based NLM seems we can do a bit more simple. */
#include <nwthread.h>

int main ( void )
{
  /* initialize any globals here... */

  /* do this if any global initializing was done
  SynchronizeStart();
  */
  ExitThread (TSR_THREAD, 0);
  return 0;
}

```

这两行代码是预处理指令，用于检查源文件是否定义了特定头文件或定义。

如果没有定义，则编译器会抛出错误。

这里，第一个预处理指令检查名为"__NOVELL_LIBC__"的头文件是否定义，第二个预处理指令检查名为"NETWARE"的头文件是否定义。如果两个预处理指令都检查失败，则编译器不会输出任何错误，继续编译并生成可执行文件。

如果只有一个预处理指令检查失败，则编译器会输出错误并停止编译。

这里，`#ifdef` 是第一个预处理指令，`#ifndef` 是第二个预处理指令。


```cpp
#endif /* __NOVELL_LIBC__ */

#endif /* NETWARE */



```