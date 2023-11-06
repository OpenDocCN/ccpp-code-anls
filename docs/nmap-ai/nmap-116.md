# Nmap源码解析 116

# `libz/zutil.c`

这段代码是一个目标依赖的压缩库中的工具函数。它定义了一个名为z_errmsg的数组，用于在压缩过程中捕获错误信息并返回相应的错误码。

具体来说，这段代码包含以下几个部分：

1. 定义了一个名为z_errmsg的数组，包括：
  - Z_NEED_DICT：需要dictionary（即字典）时出现的错误码。
  - Z_STREAM_END：表示输入流已经结束，需要提示用户。
  - Z_OK：表示输入成功，没有错误。
  - Z_ERRNO：表示输入错误，返回相应的错误码。
  - Z_STREAM_ERROR：表示输入流错误，例如编码不正确。
  - Z_DATA_ERROR：表示输出文件错误，例如磁盘读写错误。
  - Z_BUF_ERROR：表示内存错误，例如缓冲区不足或错误。
  - Z_VERSION_ERROR：表示版本不兼容，例如使用不兼容的zlib库。

2. 定义了一个常量z_errmsg[10]，用于表示上述数组。

3. 在z_util.h文件的顶部，引入了zlib.h文件。

4. 在压缩过程中，当遇到错误时，会首先查找z_errmsg数组，根据错误类型返回相应的错误码，然后打印相应的错误信息。


```cpp
/* zutil.c -- target dependent utility functions for the compression library
 * Copyright (C) 1995-2017 Jean-loup Gailly
 * For conditions of distribution and use, see copyright notice in zlib.h
 */

/* @(#) $Id$ */

#include "zutil.h"
#ifndef Z_SOLO
#  include "gzguts.h"
#endif

z_const char * const z_errmsg[10] = {
    (z_const char *)"need dictionary",     /* Z_NEED_DICT       2  */
    (z_const char *)"stream end",          /* Z_STREAM_END      1  */
    (z_const char *)"",                    /* Z_OK              0  */
    (z_const char *)"file error",          /* Z_ERRNO         (-1) */
    (z_const char *)"stream error",        /* Z_STREAM_ERROR  (-2) */
    (z_const char *)"data error",          /* Z_DATA_ERROR    (-3) */
    (z_const char *)"insufficient memory", /* Z_MEM_ERROR     (-4) */
    (z_const char *)"buffer error",        /* Z_BUF_ERROR     (-5) */
    (z_const char *)"incompatible version",/* Z_VERSION_ERROR (-6) */
    (z_const char *)""
};


```

这段代码定义了两个名为`ZEXPORT`的函数：`zlibVersion()`和`zlibCompileFlags()`。它们的作用是返回`zlib_version()`和`zlibCompileFlags()`这两个函数的实现。以下是这两函数的实现：

1. `zlibVersion()`函数返回`ZLIB_VERSION`，它是一个与zlib库版本相关的常量。

2. `zlibCompileFlags()`函数返回一个`uLong`类型的整数，表示编译器的编译标志。它的值取决于编译方式（如共享库或本地库）和目标架构（如32位或64位）。具体来说，函数的实现如下：

a. 处理u整型大小：

  - 如果sizeof(uInt)的值为2，则表示编译为x86架构，不需要进行额外的处理。

  - 如果sizeof(uInt)的值为4，则表示编译为ARM架构。在这种情况下，需要将1作为前缀添加到` flags`中。

  - 如果sizeof(uInt)的值为8，则表示编译为x86架构，但需要将2转换为3，并将4转换为4。因此，` flags`的值将为3。

  - 否则（即sizeof(uInt)的值不可能是2、4或8），则表示编译为x86架构，但需要将8转换为3，因此` flags`的值将为3。

b. 处理uLong型大小：

  - 如果sizeof(uLong)的值为2，则表示编译为x86架构，不需要进行额外的处理。

  - 如果sizeof(uLong)的值为4，则表示编译为ARM架构。在这种情况下，需要将1作为前缀添加到` flags`中。

  - 如果sizeof(uLong)的值为8，则表示编译为x86架构，但需要将2转换为3，并将4转换为4。因此，` flags`的值将为4。

  - 否则（即sizeof(uLong)的值不可能是2、4或8），则表示编译为x86架构，但需要将8转换为3，因此` flags`的值将为3。

c. 处理z_off_t类型大小：

  - 如果sizeof(z_off_t)的值为2，则表示编译为x86架构，不需要进行额外的处理。

  - 如果sizeof(z_off_t)的值为4，则表示编译为ARM架构。在这种情况下，需要将1作为前缀添加到` flags`中。

  - 如果sizeof(z_off_t)的值为8，则表示编译为x86架构，但需要将2转换为3，并将4转换为4。因此，` flags`的值将为3。

  - 否则（即sizeof(z_off_t)的值不可能是2、4或8），则表示编译为x86架构，但需要将8转换为3，因此` flags`的值将为3。


```cpp
const char * ZEXPORT zlibVersion()
{
    return ZLIB_VERSION;
}

uLong ZEXPORT zlibCompileFlags()
{
    uLong flags;

    flags = 0;
    switch ((int)(sizeof(uInt))) {
    case 2:     break;
    case 4:     flags += 1;     break;
    case 8:     flags += 2;     break;
    default:    flags += 3;
    }
    switch ((int)(sizeof(uLong))) {
    case 2:     break;
    case 4:     flags += 1 << 2;        break;
    case 8:     flags += 2 << 2;        break;
    default:    flags += 3 << 2;
    }
    switch ((int)(sizeof(voidpf))) {
    case 2:     break;
    case 4:     flags += 1 << 4;        break;
    case 8:     flags += 2 << 4;        break;
    default:    flags += 3 << 4;
    }
    switch ((int)(sizeof(z_off_t))) {
    case 2:     break;
    case 4:     flags += 1 << 6;        break;
    case 8:     flags += 2 << 6;        break;
    default:    flags += 3 << 6;
    }
```

这段代码是一个C语言中的预处理指令，用于判断是否支持不同的编译选项。它通过检查系统环境变量或者通过检查用户提供的选项来选择是否使用特定的编译选项。

具体来说，这段代码的作用如下：

1. 如果当前系统环境变量或用户提供的选项中包含“-O”选项(表示启用优化编译)，则执行以下操作：将“flags += 1 << 8;”这一行添加到选项中，这样编译器就会在编译时考虑使用“-O”选项编译出的代码。

2. 如果当前系统环境变量或用户提供的选项中包含“-伦”选项(表示启用联合循环)，则执行以下操作：将“flags += 1 << 9;”这一行添加到选项中，这样编译器就会在编译时考虑使用“-伦”选项编译出的代码。

3. 如果当前系统环境变量或用户提供的选项中包含“-z”选项(表示启用零宽断言)，则执行以下操作：将“flags += 1 << 10;”这一行添加到选项中，这样编译器就会在编译时考虑使用“-z”选项编译出的代码。

4. 如果当前系统环境变量或用户提供的选项中包含“-f”选项(表示开启飞空指针)，则执行以下操作：将“flags += 1 << 12;”这一行添加到选项中，这样编译器就会在编译时考虑使用“-f”选项编译出的代码。

5. 如果当前系统环境变量或用户提供的选项中包含“-D”选项(表示开启定义依赖)，则执行以下操作：将“flags += 1 << 13;”这一行添加到选项中，这样编译器就会在编译时考虑使用“-D”选项编译出的代码。

6. 如果当前系统环境变量中包含“-CPU”选项(表示考虑不同CPU架构)，则执行以下操作：将“flags += 1 << 13;”这一行添加到选项中，这样编译器就会在编译时考虑使用“-CPU”选项编译出的代码。

7. 如果当前系统环境变量或用户提供的选项中不包含任何一个上述选项，则执行以下操作：将“flags += 0;”这一行添加到选项中，这样编译器就会忽略该编译选项。


```cpp
#ifdef ZLIB_DEBUG
    flags += 1 << 8;
#endif
    /*
#if defined(ASMV) || defined(ASMINF)
    flags += 1 << 9;
#endif
     */
#ifdef ZLIB_WINAPI
    flags += 1 << 10;
#endif
#ifdef BUILDFIXED
    flags += 1 << 12;
#endif
#ifdef DYNAMIC_CRC_TABLE
    flags += 1 << 13;
```

这段代码是一个C语言预处理指令，用于设置编译器的一些标志选项。具体来说，它包括了以下几个判断条件：

1. NO_GZCOMPRESS：如果这个条件为真，那么编译器应该启用GZ压缩。
2. NO_GZIP：如果这个条件为真，那么编译器应该启用GZ压缩。
3. PKZIP_BUG_WORKAROUND：如果这个条件为真，那么编译器应该启用该补丁中的GZ压缩。
4. FASTEST：如果这个条件为真，那么编译器应该启用快速的编译器。
5. STDC：如果STDC预编译选项没有被定义，那么这个条件为真。
6. Z_HAVE_STDARG_H：如果这个条件为真，那么编译器支持宏定义参数。
7. NO_vsnprintf：如果这个条件为真，那么编译器应该禁用vsnprintf函数。

根据这些条件，编译器会设置一些编译器选项的标志，包括上述条件的所有选项。这些选项包括：NO_PAGE,NO_ALZ,NO_NE,NO_STACK,NO_STR,NO_UNL,NO_LEX,NO_TRACE,NO_COMPAT,NO_EXEC,NO_CALEND,NO_HEAP,NO_IDLE,NO_DISC,NO_MEM,NO_FETCH,NO_PUSH,NO_PUSH_RAPI,NO_THREAD,NO_FINAL,NO_CONS,NO_DEDENT,NO_MERGE,NO_SLIDE,NO_INSERT,NO_RCC,NO_CONT,NO_CLEAR,NO_JMP,NO_JMP_RAPI,NO_CAPS,NO_HANDLE,NO_TRACE_POST,NO_DBP,NO_DSO,NO_DIRECT,NO_FUNCTION,NO_JIT,NO_PARTIAL,NO_TRANS,NO_LOOP,NO_EXCEPT,NO_BYTEPASS,NO_HALF,NO_FUSION,NO_SYNC,NO_TLS,NO_THREAD_API,NO_PTHREAD,NO_GET_SPACE,NO_PTHREAD_RAPI,NO_ROUTINE,NO_COUNT,NO_SYNC_RWX,NO_LOCK,NO_ALT,NO_SUSPEND,NO_USER,NO_ASYNC,NO_SPINOFF,NO_RAW,NO_OMANGLE,NO_ASYNC_EXIT,NO_ONEJMP,NO_THREAD_ORDER,NO_POLYPROC,NO_MEMORY_ORDER,NO_MEMORY_TAG,NO_MEMORY_SET,NO_POLYPROC_ORDER,NO_ASYNC_POLYPROC,NO_PTHREAD_SET,NO_RTC,NO_RCL,NO_RCN,NO_CMP,NO_ACL,NO_NET,NO_DUPLEX,NO_SNOOPS,NO_TRAILOFF,NO_INIT,NO_FINISH,NO_POSTPONED,NO_CONTINUE,NO_CLEAR_INTO,NO_HEAP_ROOT,NO_WIN_STACK,NO_OVERLAPPED,NO_RPN,NO_RCV,NO_VXID,NO_SYNC_CONT,NO_X86_MCJ,NO_X86_SUB,NO_X86_INVN,NO_X86_FSR,NO_X86_SEC,NO_X86_3DN,NO_X86_IBAK,NO_X86_S1PR,NO_X86_S3PR,NO_X86_T4PR,NO_X86_T8PR,NO_X86_MONO,NO_X86_ALPHA,NO_X86_WORLD,NO_X86_千NOTCH,NO_X86_民API,

因此，这些选项将影响编译器的行为，包括对特定功能的开关以及对特定库的依赖。


```cpp
#endif
#ifdef NO_GZCOMPRESS
    flags += 1L << 16;
#endif
#ifdef NO_GZIP
    flags += 1L << 17;
#endif
#ifdef PKZIP_BUG_WORKAROUND
    flags += 1L << 20;
#endif
#ifdef FASTEST
    flags += 1L << 21;
#endif
#if defined(STDC) || defined(Z_HAVE_STDARG_H)
#  ifdef NO_vsnprintf
    flags += 1L << 25;
```

这段代码是一个条件编译语句，它会根据两个条件是否满足来选择是否使用`snprintf`函数。

具体来说，这段代码会检查两个条件：

1. HAS_vsprintf_void？
2. HAS_vsnprintf_void？

如果条件1和条件2中有一个是真，那么就会执行` flags += 1L << 26;`这个代码块，这个代码块会增加` flags`的值，并且会添加到`|`运算的结果中。这样，` flags`最终的值就是：

```cpp
1L << 26)
```

如果两个条件都不满足，那么就会执行` flags += 1L << 24;`这个代码块，这个代码块会增加` flags`的值，并且会添加到` |`运算的结果中。这样，` flags`最终的值就是：

```cpp
1L << 24)
```

这段代码的作用就是根据`snprintf`函数在不同编译器是否支持来选择使用哪个函数，从而提高程序的兼容性。


```cpp
#    ifdef HAS_vsprintf_void
    flags += 1L << 26;
#    endif
#  else
#    ifdef HAS_vsnprintf_void
    flags += 1L << 26;
#    endif
#  endif
#else
    flags += 1L << 24;
#  ifdef NO_snprintf
    flags += 1L << 25;
#    ifdef HAS_sprintf_void
    flags += 1L << 26;
#    endif
```

这段代码是一个C语言中的函数，它主要作用是检查是否支持输出文件中的压缩和错误检查。具体来说，它实现了以下几个功能：

1. 定义了一个else语句，即在输出文件中不存在压缩和错误检查时执行的代码块。

2. 定义了一个标识位 flag，它的作用是判断是否支持输出文件中的压缩。具体来说，如果 flag 的位 26 被设置为 1，则说明支持输出文件中的压缩；否则不支持。

3. 定义了一个名为 ZLIB_DEBUG 的宏，它的作用是在编译时检查是否支持调试输出。如果宏定义中未指定 verbose，则默认值为 0，表示不输出调试信息。

4. 实现了一个名为 z_verbose 的函数，它接受一个整数参数，表示是否启用 verbose 调试输出。如果这个函数被定义，那么当函数被调用时，会首先检查是否指定了 verbose，如果没有指定，就假设设置了默认值 0。然后，它会根据这个值来决定是否输出调试信息。

这段代码的作用是判断是否支持输出文件中的压缩和是否启用 verbose 调试输出。


```cpp
#  else
#    ifdef HAS_snprintf_void
    flags += 1L << 26;
#    endif
#  endif
#endif
    return flags;
}

#ifdef ZLIB_DEBUG
#include <stdlib.h>
#  ifndef verbose
#    define verbose 0
#  endif
int ZLIB_INTERNAL z_verbose = verbose;

```

这段代码定义了一个名为`z_error`的函数，它的参数`m`是一个字符型指针。这个函数的作用是输出一个错误信息，并使用`fprintf`函数将错误信息输出到`stderr`文件中。同时，它还使用`exit`函数来退出程序，并且参数`1`表示程序出错，通常情况下会输出的错误信息是"程序出错，我已经尽力了"。

该函数被导出为`zError`函数，允许程序在调用时将其错误代码转换为字符串，并使用`err`参数的值作为格式化字符串中的%s部分。这样就可以将错误信息输出到`stderr`文件中了。

`z_error`函数的作用是输出一个错误信息，然后退出程序。`zError`函数的作用则是将`z_error`函数的错误代码输出为字符串，以便在需要时进行格式化输出。


```cpp
void ZLIB_INTERNAL z_error(m)
    char *m;
{
    fprintf(stderr, "%s\n", m);
    exit(1);
}
#endif

/* exported to allow conversion of error code to string for compress() and
 * uncompress()
 */
const char * ZEXPORT zError(err)
    int err;
{
    return ERR_MSG(err);
}

```

这段代码检查两个条件是否都为真，如果是，则执行以下操作：

1. 如果定义了(_WIN32_WCE)并且(_WIN32_WCE)<0x800)，那么执行以下操作：

  ```cpp
  // 定义一个全局变量errno，值为0，表示不存在errno
  ```

2. 如果未定义(HAVE_MEMCPY)，则执行以下操作：

  ```cpp
  // zmemcpy函数定义，接收3个参数：dest、source、len
  ```

  请注意，函数内部的“do-while”循环语句可能存在问题，因为该代码缺少结束条件。


```cpp
#if defined(_WIN32_WCE) && _WIN32_WCE < 0x800
    /* The older Microsoft C Run-Time Library for Windows CE doesn't have
     * errno.  We define it as a global variable to simplify porting.
     * Its value is always 0 and should not be used.
     */
    int errno = 0;
#endif

#ifndef HAVE_MEMCPY

void ZLIB_INTERNAL zmemcpy(dest, source, len)
    Bytef* dest;
    const Bytef* source;
    uInt  len;
{
    if (len == 0) return;
    do {
        *dest++ = *source++; /* ??? to be unrolled */
    } while (--len != 0);
}

```

这两段代码是使用 zlib 库中的两个函数，分别是 `zmemcmp` 和 `zmemzero`。

`zmemcmp` 函数比较两个字节数组 `s1` 和 `s2`，返回它们的差异。它的参数包括三个变量：

* `s1`：要比较的字节数组。
* `s2`：要比较的字节数组。
* `len`：比较的两个字节数组长度。

函数内部首先定义了一个 `uInt` 类型的变量 `j`，用于遍历两个字节数组。对于每个字节，函数比较 `s1[j]` 和 `s2[j]` 是否相等，如果是，则返回 2 * (`s1[j]` 比 `s2[j]` 大) - 1。如果两个字节不相等，则返回 0。

`zmemzero` 函数用于在指定长度的字节数组 `dest` 中执行零拷贝。它的参数包括两个变量：

* `dest`：要拷贝的字节数组。
* `len`：要拷贝的字节数组长度。

函数内部首先定义了一个 `uInt` 类型的变量 `dest`，用于保存要拷贝的字节数组长度。然后使用 `do-while` 循环，将 `dest` 数组长度逐个减 1，直到 `dest` 长度小于 `len` 时退出循环。在循环内部，将 `0` 赋值给 `dest` 数组元素，实现了字节数组的零拷贝。


```cpp
int ZLIB_INTERNAL zmemcmp(s1, s2, len)
    const Bytef* s1;
    const Bytef* s2;
    uInt  len;
{
    uInt j;

    for (j = 0; j < len; j++) {
        if (s1[j] != s2[j]) return 2*(s1[j] > s2[j])-1;
    }
    return 0;
}

void ZLIB_INTERNAL zmemzero(dest, len)
    Bytef* dest;
    uInt  len;
{
    if (len == 0) return;
    do {
        *dest++ = 0;  /* ??? to be unrolled */
    } while (--len != 0);
}
```

这段代码是一个preprocess指令，它的作用是在编译之前检查特定条件是否成立。如果条件成立，它将生成注释，告诉编译器在编译时进行一些优化或者修改。

具体来说，这段代码的作用是检查两个条件是否成立：

1. 如果当前系统是基于SYS16BIT架构的（即Z系列芯片支持16位寻址），那么需要对代码进行特定修改，这个修改主要是为了在16位模式下正确地使用Turbo C函数。
2. 如果当前系统不基于SYS16BIT架构，那么也需要进行一些预处理，以保证代码的正确性。

具体的实现过程可能会根据不同的编译器和操作系统而有所不同，但是大致上可以分为以下几步：

1. 检查当前系统是否支持SYS16BIT架构。
2. 如果当前系统支持SYS16BIT架构，则检查是否支持__TURBOC__。如果是，那么就需要对代码进行一些修改，以正确地使用Turbo C函数。
3. 如果当前系统不支持SYS16BIT架构，则需要进行一些预处理操作，以保证代码的正确性。这些预处理操作可能会包括：移除一些没有被使用过的符号、检查某些头文件是否定义等。
4. 如果条件1成立，则进行以下操作：

  a. 检查是否支持__TURBOC__。如果是，就需要对代码进行一些修改，以正确地使用Turbo C函数。这些修改可能会包括：在函数声明前添加一些常量、修改函数实现等。
  b. 如果当前系统不支持__TURBOC__，就需要进行一些预处理操作，以保证代码的正确性。这些预处理操作可能会包括：移除一些没有被使用过的符号、检查某些头文件是否定义等。

最后，如果条件1和条件2都成立，那么编译器将会生成注释，告诉编译员进行相应的优化或者修改。


```cpp
#endif

#ifndef Z_SOLO

#ifdef SYS16BIT

#ifdef __TURBOC__
/* Turbo C in 16-bit mode */

#  define MY_ZCALLOC

/* Turbo C malloc() does not allow dynamic allocation of 64K bytes
 * and farmalloc(64K) returns a pointer with an offset of 8, so we
 * must fix the pointer. Warning: the pointer must be put back to its
 * original form in order to free it, use zcfree().
 */

```

这段代码定义了一个名为MAX_PTR的宏，其值为10。然后，定义了一个名为next_ptr的整型变量，并初始化为0。

接下来，定义了一个名为ptr_table的结构体，其中包含两个成员变量：org_ptr和new_ptr。org_ptr是一个指向voidp类型数据的指针，new_ptr是一个指向voidp类型数据的指针。

接着，定义了一个名为table的整型变量，其大小为MAX_PTR，并且包含一个名为next_ptr的整型变量。可以理解为，table是一个用来记住pointer的原始形式的大缓冲区数组，通过next_ptr来追踪table中每个元素的下一个偏移量。

MAX_PTR的值是10*64K，这意味着table最多可以保存10个64K的大缓冲区。


```cpp
#define MAX_PTR 10
/* 10*64K = 640K */

local int next_ptr = 0;

typedef struct ptr_table_s {
    voidpf org_ptr;
    voidpf new_ptr;
} ptr_table;

local ptr_table table[MAX_PTR];
/* This table is used to remember the original form of pointers
 * to large buffers (64K). Such pointers are normalized with a zero offset.
 * Since MSDOS is not a preemptive multitasking OS, this table is not
 * protected from concurrent access. This hack doesn't work anyway on
 * a protected system like OS/2. Use Microsoft C instead.
 */

```

这段代码是一个名为`zcalloc`的函数，属于`zlib`库。它的作用是分配一个指定大小的内存区域，并返回这个内存区的指针。它需要三个参数：一个指向需要分配的内存区的指针、需要分配的内存区的长度，以及需要分配的内存区的字节数。

函数的逻辑如下：

1. 如果分配的内存区大小小于等于65520个字节，就假设分配的内存区是可用的，不需要进行归约，然后返回这个内存区的指针。

2. 如果分配的内存区大小需要大于65520个字节，就分配一个大小为`bsize`个字节加上16个字节的内存区，其中`bsize`是输入参数中的物品数量乘以每个物品的大小，然后将这个内存区的指针赋给`table[next_ptr]`。`next_ptr`是一个计数器，用于记录已经分配的内存区指针的下一个免费指针。

3. 如果函数分配成功，就返回分配的内存区的指针。

函数的实现基于以下几个假设：

1. `farmalloc`函数会返回一个指向内存区的指针，如果分配成功则返回真指针。

2. 如果分配的内存区大小需要进行归约，`zcalloc`会尝试使用`farmalloc`分配一个足够大的内存区，以保证分配的内存区可以容纳所有的对象。如果内存区分配失败，或者分配的内存区大小无法容纳所有对象，`zcalloc`会将输入参数的值传递给`table[next_ptr]`指针，并将计数器`next_ptr`加1，以便在需要时分配更多的内存区。


```cpp
voidpf ZLIB_INTERNAL zcalloc(voidpf opaque, unsigned items, unsigned size)
{
    voidpf buf;
    ulg bsize = (ulg)items*size;

    (void)opaque;

    /* If we allocate less than 65520 bytes, we assume that farmalloc
     * will return a usable pointer which doesn't have to be normalized.
     */
    if (bsize < 65520L) {
        buf = farmalloc(bsize);
        if (*(ush*)&buf != 0) return buf;
    } else {
        buf = farmalloc(bsize + 16L);
    }
    if (buf == NULL || next_ptr >= MAX_PTR) return NULL;
    table[next_ptr].org_ptr = buf;

    /* Normalize the pointer to seg:0 */
    *((ush*)&buf+1) += ((ush)((uch*)buf-0) + 15) >> 4;
    *(ush*)&buf = 0;
    table[next_ptr++].new_ptr = buf;
    return buf;
}

```



这段代码定义了一个名为 zcfree 的函数，它的参数为两个指向 void 类型数据的指针 opaque 和 ptr，分别为需要释放的内存区域和该内存区域的原来的指针。

函数的主要作用是释放被给出的 void 类型数据所占用的内存区域，并确保在释放内存区域后，指针变量 ptr 不再指向该内存区域。具体实现包括以下几个步骤：

1. 检查指针变量 ptr 是否为空，如果是，直接跳过。

2. 如果指针变量 ptr 为空，则需要释放的内存区域至少包含 64K 个字节，因此需要遍历下一个指针，直到找到第一个非空指针或者找到对应的字节数小于 64K。

3. 如果找到对应的字节数小于 64K，需要将对应的字节数从下一个指针中减去，并继续遍历。

4. 如果遍历完成后仍然找到了对应的字节数大于等于 64K，则需要将对应的字节数从当前指针中减去，并继续遍历。

5. 如果仍然无法找到对应的字节数，则函数会输出 "zcfree: ptr not found"，并停止执行。

函数的实现基于 zlib 库中的 farfree 函数，它实现了释放内存区域的功能，该函数可以释放任意指定内存区域，并且不需要保证释放的内存区域可以在调用它的同时被任何进程访问。


```cpp
void ZLIB_INTERNAL zcfree(voidpf opaque, voidpf ptr)
{
    int n;

    (void)opaque;

    if (*(ush*)&ptr != 0) { /* object < 64K */
        farfree(ptr);
        return;
    }
    /* Find the original pointer */
    for (n = 0; n < next_ptr; n++) {
        if (ptr != table[n].new_ptr) continue;

        farfree(table[n].org_ptr);
        while (++n < next_ptr) {
            table[n-1] = table[n];
        }
        next_ptr--;
        return;
    }
    Assert(0, "zcfree: ptr not found");
}

```

这段代码是一个C语言代码中的一个 preprocessor 指令，它通过一种特定的方式定义了一些 macro 函数，从而进一步简化了程序的编写。

具体来说，这段代码定义了一个名为 ZLIB_INTERNAL 的头文件，其中包含了一些与 zlib（一个流行的库，用于处理内存管理）相关的内部函数。其中包括：

* zcalloc：一个名为 ZLIB_INTERNAL 的函数，它的参数表为 voidpf、uInt 和 uInt，用于调用操作系统版本的 halloc 函数来分配内存。
* _halloc：这是一个内部函数，它的参数表为 voidpf 和 uInt，用于定义整个内存分配的函数原型，与操作系统版本无关。
* hfree：这是一个内部函数，它的参数表为 voidpf 和 uInt，用于释放内存，与操作系统版本无关。

通过定义这些函数，这段代码使得开发人员可以使用内部函数而不是全局函数，从而使代码更易于维护和扩展。


```cpp
#endif /* __TURBOC__ */


#ifdef M_I86
/* Microsoft C in 16-bit mode */

#  define MY_ZCALLOC

#if (!defined(_MSC_VER) || (_MSC_VER <= 600))
#  define _halloc  halloc
#  define _hfree   hfree
#endif

voidpf ZLIB_INTERNAL zcalloc(voidpf opaque, uInt items, uInt size)
{
    (void)opaque;
    return _halloc((long)items, size);
}

```



这段代码定义了一个名为`zcfree`的函数，其作用是释放指定参数`opaque`所指向的内存区域，并将其返回地址赋给`ptr`参数。

进一步分析可以发现，该函数与`malloc`函数相似，但仅在`my_zcalloc`函数中定义，说明该函数并不适用于所有系统。而`malloc`函数则可以在大多数操作系统中使用，因为它定义了一个通用的内存分配函数。

由于该函数未进行明确的错误检查，因此它的安全性可能存在一定的问题。


```cpp
void ZLIB_INTERNAL zcfree(voidpf opaque, voidpf ptr)
{
    (void)opaque;
    _hfree(ptr);
}

#endif /* M_I86 */

#endif /* SYS16BIT */


#ifndef MY_ZCALLOC /* Any system without a special alloc function */

#ifndef STDC
extern voidp  malloc OF((uInt size));
```

这段代码定义了两个名为`calloc`和`free`的函数，以及一个名为`zcalloc`的函数指针。

`calloc`函数接受两个参数：`items`和`size`。它返回一个指向动态内存分配的函数的指针。如果`items`大于`size`的两倍，函数将返回一个指向内存分配的函数的指针，否则函数将返回一个指向`calloc`函数的指针。这个函数指针将作为`opaque`参数传递给函数实现。

`free`函数接受一个参数`ptr`，它是一个指向`voidp`类型数据的指针。函数将释放内存 pointed to by `ptr`，并返回一个空指针。这个函数指针将作为`opaque`参数传递给函数实现。

`zcalloc`函数是`calloc`函数的别名。它与`calloc`函数的实现完全相同，只是使用了更短的函数名称。

这些函数指针以及`zcalloc`函数的实现将在使用这些函数时决定如何分配和释放内存，从而允许用户自定义内存管理。


```cpp
extern voidp  calloc OF((uInt items, uInt size));
extern void   free   OF((voidpf ptr));
#endif

voidpf ZLIB_INTERNAL zcalloc(opaque, items, size)
    voidpf opaque;
    unsigned items;
    unsigned size;
{
    (void)opaque;
    return sizeof(uInt) > 2 ? (voidpf)malloc(items * size) :
                              (voidpf)calloc(items, size);
}

void ZLIB_INTERNAL zcfree(opaque, ptr)
    voidpf opaque;
    voidpf ptr;
{
    (void)opaque;
    free(ptr);
}

```

这两行代码是C预处理指令，它们检查两个条件，并根据这些条件是否满足来选择是否输出特定的代码块。

第一条语句检查是否定义了名为“MY_ZCALLOC”的函数。如果定义了该函数，则预处理器会编译此代码块，否则不会编译。

第二条语句检查是否定义了名为“Z_SOLO”的函数。如果定义了该函数，则预处理器会编译此代码块，否则不会编译。

这两行代码的作用是选择是否输出特定代码块，只有在定义了相应的函数时才会编译此代码块。


```cpp
#endif /* MY_ZCALLOC */

#endif /* !Z_SOLO */

```

# `libz/contrib/blast/blast.c`

这段代码是一个名为“blast.c”的 C 语言源文件，它是由 Mark Adler 编写的。它用于解压缩被 PKWare 压缩库压缩过的数据。

具体来说，这个代码将接收一个数据压缩后的流（通常是二进制数据），并将其解码为原始数据。为了实现这个目标，它实现了一个类似于 PKWare library 中 explode() 函数的函数。这个函数将输入数据中的每个块（通常是 4 字节）解码为一个字符串，并输出这个字符串。

然而，需要注意的是，这个代码的实现与 Ben Rudiak-Gould 在 comp.compression 上的描述略有不同。因此，在使用这个函数时，需要参考原始数据在 PKWare 库中的编码形式，以确保它能够正确地解码。


```cpp
/* blast.c
 * Copyright (C) 2003, 2012, 2013 Mark Adler
 * For conditions of distribution and use, see copyright notice in blast.h
 * version 1.3, 24 Aug 2013
 *
 * blast.c decompresses data compressed by the PKWare Compression Library.
 * This function provides functionality similar to the explode() function of
 * the PKWare library, hence the name "blast".
 *
 * This decompressor is based on the excellent format description provided by
 * Ben Rudiak-Gould in comp.compression on August 13, 2001.  Interestingly, the
 * example Ben provided in the post is incorrect.  The distance 110001 should
 * instead be 111000.  When corrected, the example byte stream becomes:
 *
 *    00 04 82 24 25 8f 80 7f
 *
 * which decompresses to "AIAIAIAIAIAIA" (without the quotes).
 */

```

这段代码是一个C语言程序，它主要用于修改Blast库中的代码，以解决距离检查和二进制模式的问题。

具体来说，这段代码实现了以下功能：

1. 更改历史记录：

a. 第一次发布版本，时间为2003年12月12日。

b. 修复了当输入数据大于4GB时，距离检查会失效的问题。

c. 在CRH文件中添加了使用二进制模式读取文件的说明。

d. 修复了对于不同符号的整数，比较结果可能会出现错误的问题。

e. 从blast()函数中返回未使用的输入。

f. 修复了测试代码，以正确地报告未使用的输入。

g. 使程序能够初始化输入到blast()函数中。

另外，还添加了一些注释来解释代码的功能。


```cpp
/*
 * Change history:
 *
 * 1.0  12 Feb 2003     - First version
 * 1.1  16 Feb 2003     - Fixed distance check for > 4 GB uncompressed data
 * 1.2  24 Oct 2012     - Add note about using binary mode in stdio
 *                      - Fix comparisons of differently signed integers
 * 1.3  24 Aug 2013     - Return unused input from blast()
 *                      - Fix test code to correctly report unused input
 *                      - Enable the provision of initial input to blast()
 */

#include <stddef.h>             /* for NULL */
#include <setjmp.h>             /* for setjmp(), longjmp(), and jmp_buf */
#include "blast.h"              /* prototype for blast() */

```

这段代码定义了一些用于函数定义的宏和常量，包括 `local`, `MAXBITS`, `MAXWIN`, 以及 `state` 和 `outfun` 结构体。

宏定义是预处理指令，用于定义符号名称和定义，宏可以替换为相应的语句。

`local` 定义了一个内联函数 `infun`，该函数没有参数，返回值类型为 `void`。

`MAXBITS` 定义了一个整型变量 `MAXBITS`，值为 13。

`MAXWIN` 定义了一个整型变量 `MAXWIN`，值为 4096。

`state` 定义了一个 `state` 结构体，该结构体定义了输入和输出状态的变量。

`infun` 是一个函数指针，指针类型为 `void*`。

`inhow` 被定义为 `opaque` 类型的变量，但它的作用域被限定为 `infun` 函数内部。

`in` 指针变量，指针类型为 `unsigned char*`。

`left` 被定义为 `unsigned` 类型的变量，它的作用域被限定为 `in` 指针变量内部。

`bitbuf` 被定义为 `unsigned char` 类型的变量，它的作用域被限定为 `in` 指针变量内部。

`bitcnt` 被定义为 `unsigned int` 类型的变量，它的作用域被限定为 `bitbuf` 指针变量内部。

`env` 被定义为 `jmp_buf` 类型的变量，它的作用域被限定为 `outfun` 函数内部。

`outfun` 是一个函数指针，指针类型为 `void*`。

`outhow` 被定义为 `void` 类型。

`next` 被定义为 `unsigned int` 类型的变量，它的作用域被限定为 `outfun` 函数内部。

`first` 被定义为 `bool` 类型的变量，它的作用域被限定为 `outfun` 函数内部。

`out` 是一个 `unsigned char` 类型的数组，它的作用域被限定为 `outfun` 函数内部。

`maxwins` 是一个 `unsigned int` 类型的常量，它的值为 4096。

最后，这段代码定义了 `state` 结构体，用于保存输入和输出状态的变量，以及 `outfun` 函数指针，用于提供用户定义的输出函数。


```cpp
#define local static            /* for local function definitions */
#define MAXBITS 13              /* maximum code length */
#define MAXWIN 4096             /* maximum window size */

/* input and output state */
struct state {
    /* input state */
    blast_in infun;             /* input function provided by user */
    void *inhow;                /* opaque information passed to infun() */
    unsigned char *in;          /* next input location */
    unsigned left;              /* available input at in */
    int bitbuf;                 /* bit buffer */
    int bitcnt;                 /* number of bits in bit buffer */

    /* input limit error return state for bits() and decode() */
    jmp_buf env;

    /* output state */
    blast_out outfun;           /* output function provided by user */
    void *outhow;               /* opaque information passed to outfun() */
    unsigned next;              /* index of next write location in out[] */
    int first;                  /* true to check distances (for first 4K) */
    unsigned char out[MAXWIN];  /* output buffer and sliding window */
};

```

这段代码是一个函数 `bits()`，它的作用是从输入流中返回需要的二进制数据。函数的实现采用了位运算，将输入的二进制数据中的需要的二进制数据提取出来，同时保留不足8位的数据。

具体来说，这段代码实现了一个 while 循环，该循环会在输入流中的每个字节中提取出需要的二进制数据，将其存储在一个叫做 `val` 的变量中。在循环的过程中，需要使用左移运算符 `<<` 将二进制数据向左移动一定的位数，以便实现位运算。

在每次循环中，首先判断 `s->left` 是否等于0，如果是，则表示输入流中已经没有足够多的二进制数据，需要提前退出循环，否则继续执行循环体内的代码。在循环体内，首先将需要的信息读取到一个叫做 `s->in` 的变量中，并将其赋值给 `val`。然后，计算需要多少个字节，并将这个值除以8，得到每个字节需要包含多少个位，再将这个值左移一定的位置，将需要的信息存储到 `val` 中。最后，使用 `val >>` 实现右移运算，将需要的信息转换为二进制数据并存储到 `s->bitbuf` 中，同时将 `s->bitcnt` 减去需要的信息的长度。

最终，函数返回的就是 `val` 对 2 的 `need` 次幂减 1，即需要的信息的位数。


```cpp
/*
 * Return need bits from the input stream.  This always leaves less than
 * eight bits in the buffer.  bits() works properly for need == 0.
 *
 * Format notes:
 *
 * - Bits are stored in bytes from the least significant bit to the most
 *   significant bit.  Therefore bits are dropped from the bottom of the bit
 *   buffer, using shift right, and new bytes are appended to the top of the
 *   bit buffer, using shift left.
 */
local int bits(struct state *s, int need)
{
    int val;            /* bit accumulator */

    /* load at least need bits into val */
    val = s->bitbuf;
    while (s->bitcnt < need) {
        if (s->left == 0) {
            s->left = s->infun(s->inhow, &(s->in));
            if (s->left == 0) longjmp(s->env, 1);       /* out of input */
        }
        val |= (int)(*(s->in)++) << s->bitcnt;          /* load eight bits */
        s->left--;
        s->bitcnt += 8;
    }

    /* drop need bits and update buffer, always zero to seven bits left */
    s->bitbuf = val >> need;
    s->bitcnt -= need;

    /* return need bits, zeroing the bits above that */
    return val & ((1 << need) - 1);
}

```

这段代码定义了一个名为 `huffman` 的结构体，用于存储 Huffman 代码的计数信息。这些计数信息包括每个长度的符号数量，以及符号值。

此外，这段代码还定义了一个名为 `decode` 的函数，该函数接受一个 `huffman` 结构体和一个字符流 `s`，尝试从 `s` 中编码一个给定的代码，并返回一个符号值（-1 如果出错，0 表示代码是完整的）。

函数 `decode` 首先将字符流 `s` 中的符号值存储在一个名为 `count` 的数组中，然后逐个从 `count` 中检索符号值，将它们存储在名为 `symbol` 的数组中。接着，函数将输入的代码中的符号值与计数数组中的符号值进行比较，并将符号值的正确值存储在 `symbol` 数组中。如果输入的代码长度不正确，函数将返回一个负数。

对于存储在 `count` 数组中的符号计数信息，该代码将使用哈夫曼编码将输入的符号计数信息编码成一个压缩后的表示。这种编码方式将字符的计数信息构建成一个二进制序列，使得输入的符号计数信息在存储和传输过程中可以更加高效地使用。


```cpp
/*
 * Huffman code decoding tables.  count[1..MAXBITS] is the number of symbols of
 * each length, which for a canonical code are stepped through in order.
 * symbol[] are the symbol values in canonical order, where the number of
 * entries is the sum of the counts in count[].  The decoding process can be
 * seen in the function decode() below.
 */
struct huffman {
    short *count;       /* number of symbols of each length */
    short *symbol;      /* canonically ordered symbols */
};

/*
 * Decode a code from the stream s using huffman table h.  Return the symbol or
 * a negative value if there is an error.  If all of the lengths are zero, i.e.
 * an empty code, or if the code is incomplete and an invalid code is received,
 * then -9 is returned after reading MAXBITS bits.
 *
 * Format notes:
 *
 * - The codes as stored in the compressed data are bit-reversed relative to
 *   a simple integer ordering of codes of the same lengths.  Hence below the
 *   bits are pulled from the compressed data one at a time and used to
 *   build the code value reversed from what is in the stream in order to
 *   permit simple integer comparisons for decoding.
 *
 * - The first code for the shortest length is all ones.  Subsequent codes of
 *   the same length are simply integer decrements of the previous code.  When
 *   moving up a length, a one bit is appended to the code.  For a complete
 *   code, the last code of the longest length will be all zeros.  To support
 *   this ordering, the bits pulled during decoding are inverted to apply the
 *   more "natural" ordering starting with all zeros and incrementing.
 */
```

这段代码定义了一个名为decode的函数，其功能是将输入的编码字符串解码成对应的ASCII字符。函数接收两个参数：一个指向struct state结构体的指针h和一个指向struct huffman结构体的指针h，分别用于处理解码过程中需要用到的信息。函数内部定义了一系列整型变量，用于表示解码过程中的各种信息，如当前字符串长度、正在解码的字符数目、已处理过的字符数目等等。

函数主要分以下几步：

1. 定义了一个len变量，用于表示当前字符串长度，初始化为1。

2. 定义了一个first变量，用于记录当前正在处理的字符所处的编码位置，初始化为0。

3. 定义了一个count变量，用于记录已经处理过的字符数目，初始化为0。

4. 定义了一个next变量，用于存储下一个需要处理的编码字符，初始化为h的count加上1，即当前的编码字符的下一个编码字符。

5. 定义了一个code变量，用于表示当前正在解码的字符的编码，初始化为0。

6. 定义了一个bitbuf变量，用于存储当前输入字符串的比特缓冲区，每次读取一个比特并将其存储到bitbuf中。

7. 定义了一个left变量，用于记录当前正在处理的字符列的左端点，初始化为0。

8. 定义了一个while循环，用于不断处理输入字符串中的比特，直到处理完整个字符串。

9. 在循环内部，定义了一个while循环，用于计算剩余的可以处理的字符数目，然后更新next变量，用于在循环结束后获取下一个编码字符。

10. 在while循环内部，首先将bitbuf的当前位设置为输入字符串中当前比特的逆码，然后将bitbuf的值移除一个字节并将其赋值给code变量，用于更新当前正在处理的编码字符。

11. 然后更新left变量为当前比特缓冲区中的位向量与下一个需要处理的字符的ASCII码之差，保证left的值在-8到8之间。

12. 如果当前代码长度的字符串已经处理完毕，将left变量设置为0，跳出while循环，返回-9，表示无法继续处理这个字符串。

13. 否则，将bitbuf的值设置为输入字符串中当前比特，将s->inhow的值更新为字符串长度减1，然后尝试获取当前比特字符的ASCII码，如果成功则更新s->bitcnt并从h->symbol数组中返回对应的ASCII码，否则继续尝试下一个字符的ASCII码。

14. 循环结束后，如果left变量仍然等于0，说明已经处理完了所有的字符，此时将next变量赋值为下一个需要处理的编码字符，然后跳出while循环。

15. 如果循环结束后left变量仍然大于8，说明已经遍历了整个字符串，此时将while循环中的while语句跳出。

16. 最后返回-9，表示无法继续处理这个字符串。


```cpp
local int decode(struct state *s, struct huffman *h)
{
    int len;            /* current number of bits in code */
    int code;           /* len bits being decoded */
    int first;          /* first code of length len */
    int count;          /* number of codes of length len */
    int index;          /* index of first code of length len in symbol table */
    int bitbuf;         /* bits from stream */
    int left;           /* bits left in next or left to process */
    short *next;        /* next number of codes */

    bitbuf = s->bitbuf;
    left = s->bitcnt;
    code = first = index = 0;
    len = 1;
    next = h->count + 1;
    while (1) {
        while (left--) {
            code |= (bitbuf & 1) ^ 1;   /* invert code */
            bitbuf >>= 1;
            count = *next++;
            if (code < first + count) { /* if length len, return symbol */
                s->bitbuf = bitbuf;
                s->bitcnt = (s->bitcnt - len) & 7;
                return h->symbol[index + (code - first)];
            }
            index += count;             /* else update for next length */
            first += count;
            first <<= 1;
            code <<= 1;
            len++;
        }
        left = (MAXBITS+1) - len;
        if (left == 0) break;
        if (s->left == 0) {
            s->left = s->infun(s->inhow, &(s->in));
            if (s->left == 0) longjmp(s->env, 1);       /* out of input */
        }
        bitbuf = *(s->in)++;
        s->left--;
        if (left > 8) left = 8;
    }
    return -9;                          /* ran out of codes */
}

```

这段代码的主要作用是实现一个名为“compaction”的压缩算法，用于减少一个给定长度的列表中重复代码的影响。通过计算每个字节中的计数值，以及为每个代码长度创建一个独立的计数器，最终可以压缩出一部分代码。

在给定的长度表示的代码中，每个长度都有自己的计数器。当遇到一个重复的长度时，该计数器会增加该长度的计数。通过遍历整个列表，可以为每个长度创建一个计数器，并在遍历过程中计算出每个长度的出现次数。

在构建完计数器之后，给定长度的列表中包含了每个长度的计数。为了在解码时使用这些计数，需要将这些计数转换为实际的数量。这个过程可以通过取绝对值来完成，这样即使某个长度对应的计数值为负数，解码器也可以处理它。

最后，根据给定长度的列表中代码出现次数构建代码表。这些代码表可以用于解码器，以处理从足够长的代码中提取出长度的过程。如果返回值为零，则表示解码器可以处理给定的输入，否则表示已处理过的长度范围内有错误，需要返回一个负值或正值。


```cpp
/*
 * Given a list of repeated code lengths rep[0..n-1], where each byte is a
 * count (high four bits + 1) and a code length (low four bits), generate the
 * list of code lengths.  This compaction reduces the size of the object code.
 * Then given the list of code lengths length[0..n-1] representing a canonical
 * Huffman code for n symbols, construct the tables required to decode those
 * codes.  Those tables are the number of codes of each length, and the symbols
 * sorted by length, retaining their original order within each length.  The
 * return value is zero for a complete code set, negative for an over-
 * subscribed code set, and positive for an incomplete code set.  The tables
 * can be used if the return value is zero or positive, but they cannot be used
 * if the return value is negative.  If the return value is zero, it is not
 * possible for decode() using that table to return an error--any stream of
 * enough bits will resolve to a symbol.  If the return value is positive, then
 * it is possible for decode() using that table to return an error for received
 * codes past the end of the incomplete lengths.
 */
```

This is a function called `decode()` which takes an array of code lengths `length` and an array of counts `rep` in both the little-endian and big-endian forms.

It first converts the `rep` array into a symbol-based bit-length array by counting the number of occurrences of each symbol in the `rep` array, and storing the counts in the `length` array.

Then, it checks the length of the input `length` array and sets the initial offset of the symbol table to `0`.

It then loops through each symbol in the `length` array, counting the number of occurrences of each symbol and storing the counts in the `h->count` array.

If the length of a symbol is non-zero, it checks whether the symbol has been used to represent any code of any length, and if so, increases the offset of the symbol table in that direction.

Finally, it sorts the symbols according to their length, and if there are no symbols with a length greater than or equal to the maximum symbol bit length, it returns 0. Otherwise, it returns the offset for the smallest non-zero symbol bit length, or the negative value if the input is incomplete.


```cpp
local int construct(struct huffman *h, const unsigned char *rep, int n)
{
    int symbol;         /* current symbol when stepping through length[] */
    int len;            /* current length when stepping through h->count[] */
    int left;           /* number of possible codes left of current length */
    short offs[MAXBITS+1];      /* offsets in symbol table for each length */
    short length[256];  /* code lengths */

    /* convert compact repeat counts into symbol bit length list */
    symbol = 0;
    do {
        len = *rep++;
        left = (len >> 4) + 1;
        len &= 15;
        do {
            length[symbol++] = len;
        } while (--left);
    } while (--n);
    n = symbol;

    /* count number of codes of each length */
    for (len = 0; len <= MAXBITS; len++)
        h->count[len] = 0;
    for (symbol = 0; symbol < n; symbol++)
        (h->count[length[symbol]])++;   /* assumes lengths are within bounds */
    if (h->count[0] == n)               /* no codes! */
        return 0;                       /* complete, but decode() will fail */

    /* check for an over-subscribed or incomplete set of lengths */
    left = 1;                           /* one possible code of zero length */
    for (len = 1; len <= MAXBITS; len++) {
        left <<= 1;                     /* one more bit, double codes left */
        left -= h->count[len];          /* deduct count from possible codes */
        if (left < 0) return left;      /* over-subscribed--return negative */
    }                                   /* left > 0 means incomplete */

    /* generate offsets into symbol table for each length for sorting */
    offs[1] = 0;
    for (len = 1; len < MAXBITS; len++)
        offs[len + 1] = offs[len] + h->count[len];

    /*
     * put symbols in table sorted by length, by symbol order within each
     * length
     */
    for (symbol = 0; symbol < n; symbol++)
        if (length[symbol] != 0)
            h->symbol[offs[length[symbol]]++] = symbol;

    /* return zero for complete set, positive for incomplete set */
    return left;
}

```

这段代码是一个名为“decode_pkware”的函数，其作用是解码PKware压缩库中的流。

具体来说，这段代码将接收一个字节数组，里面包含了数据压缩中的键值对。对于每个键值对，函数会根据键的类型（0表示未编码，1表示编码）进行不同的处理。对于第二个字节，函数将其转换为4、5或6，表示距离代码。

函数的主要作用是将压缩数据中的键值对进行解码，并输出对应的原始数据。在解码的过程中，函数会根据不同的键类型进行不同的操作，对于未编码的键，函数会将下一个8个字节输出；对于编码的键，函数会将对应距离的编码后的数据复制到输出中。

此外，函数还支持一些特殊的操作，如双倍距离复制和重叠复制。双倍距离复制是指将某个距离的键值对复制到输出中，而重叠复制则是指将某个距离的键值对复制到输出中，同时将相同距离的键值对进行覆盖。


```cpp
/*
 * Decode PKWare Compression Library stream.
 *
 * Format notes:
 *
 * - First byte is 0 if literals are uncoded or 1 if they are coded.  Second
 *   byte is 4, 5, or 6 for the number of extra bits in the distance code.
 *   This is the base-2 logarithm of the dictionary size minus six.
 *
 * - Compressed data is a combination of literals and length/distance pairs
 *   terminated by an end code.  Literals are either Huffman coded or
 *   uncoded bytes.  A length/distance pair is a coded length followed by a
 *   coded distance to represent a string that occurs earlier in the
 *   uncompressed data that occurs again at the current location.
 *
 * - A bit preceding a literal or length/distance pair indicates which comes
 *   next, 0 for literals, 1 for length/distance.
 *
 * - If literals are uncoded, then the next eight bits are the literal, in the
 *   normal bit order in the stream, i.e. no bit-reversal is needed. Similarly,
 *   no bit reversal is needed for either the length extra bits or the distance
 *   extra bits.
 *
 * - Literal bytes are simply written to the output.  A length/distance pair is
 *   an instruction to copy previously uncompressed bytes to the output.  The
 *   copy is from distance bytes back in the output stream, copying for length
 *   bytes.
 *
 * - Distances pointing before the beginning of the output data are not
 *   permitted.
 *
 * - Overlapped copies, where the length is greater than the distance, are
 *   allowed and common.  For example, a distance of one and a length of 518
 *   simply copies the last byte 518 times.  A distance of four and a length of
 *   twelve copies the last four bytes three times.  A simple forward copy
 *   ignoring whether the length is greater than the distance or not implements
 *   this correctly.
 */
```

This is a JavaScript function that performs a text search in a JSON file. It takes in a JSON string `s` and a JSON object `dict` and returns the search result (0 or -2).

The function first attempts to decode any literal text by looking for a specific ASCII symbol. If it finds such a symbol, it then gets the length of the literal and uses it to calculate the distance to the next occurrence of the symbol in the `dict`. If the distance is too far back, the function returns -2.

If the function does not find any literal text, it then attempts to search for the search term by looking for specific keywords in the JSON string. It does this by first decoding any keywords using a lookup table, and then replacing any match with the ASCII code of the keyword. This is done in a way that avoids consuming too much memory space by searching for keywords multiple times.

The function returns 0 if the search term is found or -2 if it cannot find any matching text or the search term is too far back.


```cpp
local int decomp(struct state *s)
{
    int lit;            /* true if literals are coded */
    int dict;           /* log2(dictionary size) - 6 */
    int symbol;         /* decoded symbol, extra bits for distance */
    int len;            /* length for copy */
    unsigned dist;      /* distance for copy */
    int copy;           /* copy counter */
    unsigned char *from, *to;   /* copy pointers */
    static int virgin = 1;                              /* build tables once */
    static short litcnt[MAXBITS+1], litsym[256];        /* litcode memory */
    static short lencnt[MAXBITS+1], lensym[16];         /* lencode memory */
    static short distcnt[MAXBITS+1], distsym[64];       /* distcode memory */
    static struct huffman litcode = {litcnt, litsym};   /* length code */
    static struct huffman lencode = {lencnt, lensym};   /* length code */
    static struct huffman distcode = {distcnt, distsym};/* distance code */
        /* bit lengths of literal codes */
    static const unsigned char litlen[] = {
        11, 124, 8, 7, 28, 7, 188, 13, 76, 4, 10, 8, 12, 10, 12, 10, 8, 23, 8,
        9, 7, 6, 7, 8, 7, 6, 55, 8, 23, 24, 12, 11, 7, 9, 11, 12, 6, 7, 22, 5,
        7, 24, 6, 11, 9, 6, 7, 22, 7, 11, 38, 7, 9, 8, 25, 11, 8, 11, 9, 12,
        8, 12, 5, 38, 5, 38, 5, 11, 7, 5, 6, 21, 6, 10, 53, 8, 7, 24, 10, 27,
        44, 253, 253, 253, 252, 252, 252, 13, 12, 45, 12, 45, 12, 61, 12, 45,
        44, 173};
        /* bit lengths of length codes 0..15 */
    static const unsigned char lenlen[] = {2, 35, 36, 53, 38, 23};
        /* bit lengths of distance codes 0..63 */
    static const unsigned char distlen[] = {2, 20, 53, 230, 247, 151, 248};
    static const short base[16] = {     /* base for length codes */
        3, 2, 4, 5, 6, 7, 8, 9, 10, 12, 16, 24, 40, 72, 136, 264};
    static const char extra[16] = {     /* extra bits for length codes */
        0, 0, 0, 0, 0, 0, 0, 0, 1, 2, 3, 4, 5, 6, 7, 8};

    /* set up decoding tables (once--might not be thread-safe) */
    if (virgin) {
        construct(&litcode, litlen, sizeof(litlen));
        construct(&lencode, lenlen, sizeof(lenlen));
        construct(&distcode, distlen, sizeof(distlen));
        virgin = 0;
    }

    /* read header */
    lit = bits(s, 8);
    if (lit > 1) return -1;
    dict = bits(s, 8);
    if (dict < 4 || dict > 6) return -2;

    /* decode literals and length/distance pairs */
    do {
        if (bits(s, 1)) {
            /* get length */
            symbol = decode(s, &lencode);
            len = base[symbol] + bits(s, extra[symbol]);
            if (len == 519) break;              /* end code */

            /* get distance */
            symbol = len == 2 ? 2 : dict;
            dist = decode(s, &distcode) << symbol;
            dist += bits(s, symbol);
            dist++;
            if (s->first && dist > s->next)
                return -3;              /* distance too far back */

            /* copy length bytes from distance bytes back */
            do {
                to = s->out + s->next;
                from = to - dist;
                copy = MAXWIN;
                if (s->next < dist) {
                    from += copy;
                    copy = dist;
                }
                copy -= s->next;
                if (copy > len) copy = len;
                len -= copy;
                s->next += copy;
                do {
                    *to++ = *from++;
                } while (--copy);
                if (s->next == MAXWIN) {
                    if (s->outfun(s->outhow, s->out, s->next)) return 1;
                    s->next = 0;
                    s->first = 0;
                }
            } while (len != 0);
        }
        else {
            /* get literal and write it */
            symbol = lit ? decode(s, &litcode) : bits(s, 8);
            s->out[s->next++] = symbol;
            if (s->next == MAXWIN) {
                if (s->outfun(s->outhow, s->out, s->next)) return 1;
                s->next = 0;
                s->first = 0;
            }
        }
    } while (1);
    return 0;
}

```



这段代码是一个名为`blast`的函数，其作用是执行字节序列的解码操作。

具体来说，它接受四个参数：

- `infun`是一个输入函数，用于读取输入序列中的比特流；
- `inhow`是一个输入参数，用于指定输入序列的读取方式，可以是`BLAST_从头读`或`BLAST_从中间读取`;
- `outfun`是一个输出函数，用于写入解码后的结果比特流；
- `outhow`是一个输出参数，用于指定输出序列的写入方式，可以是`BLAST_从头写入`或`BLAST_从中间写入`。

函数内部首先初始化输入和输出状态，包括输入序列的左右边界、输入比特流和输出比特流等。

然后，函数开始执行解码操作。首先将输入序列中的比特流左移，如果输入序列中还有比特流，就将它们添加到`bitbuf`数组中。接着，函数使用`decom`函数对输入序列进行解码，如果解码过程中出现错误，函数将返回错误码。

最后，函数将解码后的结果比特流写入输出函数，并更新错误码。如果错误码为0，说明解码成功，返回0；否则，返回错误码。


```cpp
/* See comments in blast.h */
int blast(blast_in infun, void *inhow, blast_out outfun, void *outhow,
          unsigned *left, unsigned char **in)
{
    struct state s;             /* input/output state */
    int err;                    /* return value */

    /* initialize input state */
    s.infun = infun;
    s.inhow = inhow;
    if (left != NULL && *left) {
        s.left = *left;
        s.in = *in;
    }
    else
        s.left = 0;
    s.bitbuf = 0;
    s.bitcnt = 0;

    /* initialize output state */
    s.outfun = outfun;
    s.outhow = outhow;
    s.next = 0;
    s.first = 1;

    /* return if bits() or decode() tries to read past available input */
    if (setjmp(s.env) != 0)             /* if came back here via longjmp(), */
        err = 2;                        /*  then skip decomp(), return error */
    else
        err = decomp(&s);               /* decompress */

    /* return unused input */
    if (left != NULL)
        *left = s.left;
    if (in != NULL)
        *in = s.left ? s.in : NULL;

    /* write any leftover output and update the error code if needed */
    if (err != 1 && s.next && s.outfun(s.outhow, s.out, s.next) && err == 0)
        err = 1;
    return err;
}

```

这段代码包括一个名为“TEST”的预处理指令，以及一个名为“inf”的函数。

“TEST”指令是一个示例，告诉编译器在编译之前进行自定义的检查，检查是否支持某种特定功能。如果这个功能是启用Blast库，那么就会编译这个函数，否则就会忽略。

“inf”函数是一个局部函数，作用是读取一个指定大小的块（这里定义为16384个块），并将其存储在静态变量hold中。然后，函数将hold中的所有内容复制到一个动态变量buf中。最后，函数返回了开始从how文件中读取的块的起始地址。

通过创建一个名为“inf”的函数，用户可以方便地在程序中添加自己的代码，以检查程序是否支持Blast库。


```cpp
#ifdef TEST
/* Example of how to use blast() */
#include <stdio.h>
#include <stdlib.h>

#define CHUNK 16384

local unsigned inf(void *how, unsigned char **buf)
{
    static unsigned char hold[CHUNK];

    *buf = hold;
    return fread(hold, 1, CHUNK, (FILE *)how);
}

```

这段代码定义了一个名为`outf`的函数，它的作用是将从标准输入（stdin）传递给它的一个可变参数`how`以及一个可变长度缓冲区`buf`，然后将其写入到一个指向`how`类型的指针所指向的文件中。函数的返回值是异或操作的结果，即`fwrite`函数返回的文件写入长度减去`buf`中实际被写入的元素个数。

该函数来自于一个名为`blast`的函数，该函数的参数为：

- `inf`：输入文件描述符，指向输入文件流的起始地址
- `how`：输出文件描述符，指向输出文件流的起始地址
- `buf`：输入缓冲区，包含输入数据
- `len`：输入缓冲区的长度，字节数
- `how`：输出缓冲区，包含输出数据
- `stdout`：输出文件描述符，指向输出文件流的结束地址

该函数的实现基于Linux系统调用`fwrite`，`fread`和`getchar`函数，`blast`函数的实现则依赖于具体的压缩库。


```cpp
local int outf(void *how, unsigned char *buf, unsigned len)
{
    return fwrite(buf, 1, len, (FILE *)how) != len;
}

/* Decompress a PKWare Compression Library stream from stdin to stdout */
int main(void)
{
    int ret;
    unsigned left;

    /* decompress to stdout */
    left = 0;
    ret = blast(inf, stdin, outf, stdout, &left, NULL);
    if (ret != 0)
        fprintf(stderr, "blast error: %d\n", ret);

    /* count any leftover bytes */
    while (getchar() != EOF)
        left++;
    if (left)
        fprintf(stderr, "blast warning: %u unused bytes of input\n", left);

    /* return blast() error code */
    return ret;
}
```

这段代码是一个 preprocessor 头文件，主要用于在编译时检查特定符号是否定义。在这里，`#ifdef` 和 `#ifndef` 分别表示如果定义了这个头文件，那么编译器会检查 `#ifdef` 后面的代码，而如果未定义，则不检查。

简单来说，这段代码的作用是检查编译器是否定义了特定符号，如果没有定义，则编译器会提示开发者进行定义。这个功能在开发中非常重要，因为它可以避免在编译时出现未定义的符号，从而保证代码的编译质量和稳定性。


```cpp
#endif

```

# `libz/contrib/dotzlib/DotZLib/AssemblyInfo.cs`

该代码是一个 .NET 库的元数据(metadata)文件，其中包含有关该库的名称、描述、配置、作者、产品、版权和 trademark等信息。它是由 .NET 框架编译器服务提供程序编写的，用于描述该库的元数据信息。

具体来说，该代码定义了一个名为 "DotZLib" 的 .NET  assembly，元数据中包含了以下信息：

- AssemblTitle: "DotZLib"：该库的标题，显示为 "DotZLib"
- AssemblDescription: ".Net bindings for ZLib compression dll 1.2.x"：该库的描述，指出该库是为 Zlib 压缩 DLL 版本 1.2.x 编写的。
- AssemblConfiguration: ""：该列目前为空。
- AssemblCompany: "Henrik Ravn"：该列的值是一个公司或组织名称。
- AssemblProduct: ""：该列的值是一个产品名称。
- AssemblCopyright: "Copyright (c) 2004 by Henrik Ravn"：该列的值是版权通知，指出该库的版权属于 "Henrik Ravn"。
- AssemblTrademark: "DotZLib"：该列的值是一个商标名称。
- As您就可以UseDescription = "This library provides .NET bindings for ZLib compression dll 1.2.x"：该列的值是一个描述，指出该库用于什么目的。
- As您就可以AgeDescription = "1.2.x"：该列的值是一个描述，指出该库支持哪些版本。
- As您就可以License: "Software Foundation"：该列的值是一个许可证，指出该库使用哪种软件基金会许可证。
- As您就可以Remarks: "This library provides .NET bindings for ZLib compression dll 1.2.x。可以从 https://github.com/zlib/zlib 获取更多信息。"：该列的值是一些其他信息，指向一个 GitHub 页面的链接，以获取有关该库的更多信息。


```cpp
using System.Reflection;
using System.Runtime.CompilerServices;

//
// General Information about an assembly is controlled through the following
// set of attributes. Change these attribute values to modify the information
// associated with an assembly.
//
[assembly: AssemblyTitle("DotZLib")]
[assembly: AssemblyDescription(".Net bindings for ZLib compression dll 1.2.x")]
[assembly: AssemblyConfiguration("")]
[assembly: AssemblyCompany("Henrik Ravn")]
[assembly: AssemblyProduct("")]
[assembly: AssemblyCopyright("(c) 2004 by Henrik Ravn")]
[assembly: AssemblyTrademark("")]
[assembly: AssemblyCulture("")]

```

这段代码是一个名为"AssemblyVersion"的 assembly 版本信息。它包含四个值：

1. Major 版本号
2. Minor 版本号
3. 构建号
4. 修订号

这段代码的作用是定义了一个名为"AssemblyVersion"的 assembly 版本信息类，并在需要签名时使用自定义的密钥进行签名。通过使用 "*" 占位符，可以指定所有四个值，或者默认使用修订号和构建号。


```cpp
//
// Version information for an assembly consists of the following four values:
//
//      Major Version
//      Minor Version
//      Build Number
//      Revision
//
// You can specify all the values or you can default the Revision and Build Numbers
// by using the '*' as shown below:

[assembly: AssemblyVersion("1.0.*")]

//
// In order to sign your assembly you must specify a key to use. Refer to the
```

这段代码是一个用于在C#应用程序中选择用于签名本地 assembly的Crypto Service Provider (CSP) 密钥的代码。它允许开发人员在没有指定密钥的情况下使用默认密钥进行签名。

具体来说，这段代码使用两个整型属性：KeyName 和 KeyFile。如果这两个属性中指定的任意一个存在，那么就会使用该属性中指定的密钥进行签名。如果两个属性中指定的密钥都不存在，那么就会在当前目录下创建一个新的密钥文件并将其安装到CSP中，然后使用该密钥文件进行签名。

另外，这段代码还提供了一个名为 sn.exe的命令行工具，用于创建新的密钥文件。要使用这个工具，开发人员需要将其安装到项目中并指定具体的密钥文件和该文件的名称。


```cpp
// Microsoft .NET Framework documentation for more information on assembly signing.
//
// Use the attributes below to control which key is used for signing.
//
// Notes:
//   (*) If no key is specified, the assembly is not signed.
//   (*) KeyName refers to a key that has been installed in the Crypto Service
//       Provider (CSP) on your machine. KeyFile refers to a file which contains
//       a key.
//   (*) If the KeyFile and the KeyName values are both specified, the
//       following processing occurs:
//       (1) If the KeyName can be found in the CSP, that key is used.
//       (2) If the KeyName does not exist and the KeyFile does exist, the key
//           in the KeyFile is installed into the CSP and used.
//   (*) In order to create a KeyFile, you can use the sn.exe (Strong Name) utility.
```

这段代码是一个 Assembler  Delay Signer，用于延迟签名。它通过 `[assembly: AssemblyDelaySign(false)]` 属性表明这是一个 Delay Signer，并通过 `[assembly: AssemblyKeyFile("")]` 和 `[assembly: AssemblyKeyName("")]` 属性指定要签名的代码单元的引用和名称。

对于 `[assembly: AssemblyDelaySign(false)]`，它指示 Assembler 是否启用延迟签名。如果启用延迟签名，则每次代码单元的延迟时间将被限制，以确保不会导致延迟签名导致的安全问题。如果不启用延迟签名，则允许每个代码单元立即执行。

对于 `[assembly: AssemblyKeyFile("")]` 和 `[assembly: AssemblyKeyName("")]` 属性，它们指定了要签名的代码单元的引用和名称。

因此，这段代码的作用是定义了一个 Assembler 延迟签名，允许在代码单元的延迟时间超过预设限制之前延迟签名。


```cpp
//       When specifying the KeyFile, the location of the KeyFile should be
//       relative to the project output directory which is
//       %Project Directory%\obj\<configuration>. For example, if your KeyFile is
//       located in the project directory, you would specify the AssemblyKeyFile
//       attribute as [assembly: AssemblyKeyFile("..\\..\\mykey.snk")]
//   (*) Delay Signing is an advanced option - see the Microsoft .NET Framework
//       documentation for more information on this.
//
[assembly: AssemblyDelaySign(false)]
[assembly: AssemblyKeyFile("")]
[assembly: AssemblyKeyName("")]

```

# Table of Contents
---
   
 * [Jhbuild](#jhbuild)
 	* Possible error
 * [gtk-mac-bundler](#bundler)
 * [How to use](#howto)
 	* Prerequisite
 	* Usage

## <a name="jhbuild"></a>Jhbuild

In order to set up Jhbuild properly before building Nmap suite, follow the tutorial at [https://wiki.gnome.org/Projects/GTK%2B/OSX/Building](https://wiki.gnome.org/Projects/GTK%2B/OSX/Building), but keep reading this file if you encounter any error...

If you had any error, just type the following command to delete jhbuild,

	$ rm -rf ~/.local ~/.new_local ~/.cache ~/.config ~/Source/jhbuild ~/Source/pyenv ~/Library/Caches/pip* ~/gtk

And we'll start over together:

1.	First, simply download the following script in your _$HOME_ directory ([https://git.gnome.org/browse/gtk-osx/plain/gtk-osx-build-setup.sh](https://git.gnome.org/browse/gtk-osx/plain/gtk-osx-build-setup.sh)). Edit it to make sure that `MACOSX_DEPLOYMENT_TARGET` exists and is set to the lowest supported version of OS X, e.g. "10.11". Then run it:

	~~~~
	$ sh gtk-osx-build-setup.sh
	~~~~
	
	And add it to your _$PATH_, so you can run jhbuild without the absolute path:
	
	~~~~
	$ export PATH=$HOME/.local/bin:$PATH
	~~~~
	
2.	In `~/.jhbuildrc-custom`, make sure that this line is setup properly and matches `MACOSX_DEPLOYMENT_TARGET` from step 1:

	~~~~
	setup_sdk(target="10.11")
	~~~~
	
3.	Now do,

	~~~~
	$ jhbuild bootstrap-gtk-osx
	~~~~
	
	To install missing dependencies (with **--force** option to force rebuilding).<br/>
	
4.	And,

	~~~~
	$ jhbuild build meta-gtk-osx-bootstrap
	$ jhbuild build meta-gtk-osx-core

5. Now we need Python2 and the GTK2 bindings for it, but gtk-osx has built
Python3, and the bindings will prefer that even though the dev headers aren't
present. Specifically, we need pycairo prior to 1.19 (when they dropped Python2
support) and gtk-integration-python. There's got to be a better way, but what I
did was first install python2:

	$ jhbuild build python

Then install pycairo. This is necessary because if it's missing for Python 2,
the other bindings won't build for Python 2 either. Make sure version is less
than 1.19 in ~/.cache/jhbuild/gtk-osx-python.modules. This may "succeed" but it
will have built the Python3 bindings. Clear out the build tree and make sure
the source will prefer python2:

	$ jhbuild build pycairo
	$ rm -rf ~/.cache/jhbuild/build/pycairo-*
	$ sed -i 's/python3/python2/' ~/gtk/source/pycairo-*/meson_options.txt
	$ jhbuild build pycairo

Now build the rest of the python bindings. Some of these will fail (and maybe
they failed as prereqs for pycairo earlier), so hang on and I'll tell you how
to fix those:

	$ jhbuild build meta-gtk-osx-python

Ok, when you get a failure, that's your chance to reconfigure with python2.
Jhbuild will give you some options; choose "4. start a shell" and then check
for the proper configuration command (may be visible in scrollback, otherwise
check config.log) and copy it. Clear out the build directory (probably the
current directory, ~/.cache/jhbuild/build/package-name-version/*) then from
there run the configuration command with PYTHON variable overridden, e.g.:

	$ PYTHON=$(which python2) ~/gtk/source/package-name-version/configure --some-options

Now exit that shell and go to the build step. This might mean "ignore error and
continue with build" or it might mean "rerun step build" depending on when the
error happened.

### Possible error

For those of you who have this error while trying to make,

~~~~
svn: E155021: This client is too old to work with the working copy at...
~~~~

You need to **update SVN**.<br/>
Go to [http://www.wandisco.com/subversion/download#osx](http://www.wandisco.com/subversion/download#osx) and download and install the approriate version for your OS.

Now, add the path for the new SVN version to your _$PATH_:

~~~~
$ export PATH=/opt/subversion/bin:$PATH
~~~~

## <a name="bundler"></a>gtk-mac-bundler

Now that Jhbuild is properly configured, we need to install **gtk-mac-bundler** in order to render the bundle file:

~~~~
$ git clone git://git.gnome.org/gtk-mac-bundler
$ cd gtk-mac-bundler
$ make install
~~~~

## <a name="howto"></a>How to use
#### Prerequisite:
—`openssl.modules`:

This is a jhbuild moduleset that can be used to build/update openssl.

#### Usage:

Now use it like this:
    
~~~~
$ jhbuild -m openssl.modules build nmap-deps
~~~~
