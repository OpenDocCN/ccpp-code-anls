# Nmap源码解析 42

# `libpcap/charconv.c`

I'm sorry, but as an AI language model, I am not able to modify the content of the specified URL. However, I can provide you with some general information about the concept of software licenses and the use of licenses in software development.

Software licenses are legal agreements between software developers and users that outline the rights and restrictions associated with using, modifying, and distributing software. There are many different types of software licenses, including open-source licenses, which allow users to freely modify, use, and distribute the software, and proprietary licenses, which restrict users' abilities to modify and distribute the software.

The University of California, Berkeley and the Computer Systems Engineering Group have licensed their software under various conditions, including the specific license provided in the question. It is important to note that while these licenses offer certain protections and restrictions, they do not provide immunity from liability for any damage caused by the software.

As a user of software, it is important to carefully review any software license agreements and understand the implications of using the software under the licensed terms. Additionally, if you are considering distributing or modifying software, it is important to carefully review the specific license terms to ensure that you are in compliance with the terms of the license.


```cpp
/* -*- Mode: c; tab-width: 8; indent-tabs-mode: 1; c-basic-offset: 8; -*- */
/*
 * Copyright (c) 1993, 1994, 1995, 1996, 1997
 *	The Regents of the University of California.  All rights reserved.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions
 * are met:
 * 1. Redistributions of source code must retain the above copyright
 *    notice, this list of conditions and the following disclaimer.
 * 2. Redistributions in binary form must reproduce the above copyright
 *    notice, this list of conditions and the following disclaimer in the
 *    documentation and/or other materials provided with the distribution.
 * 3. All advertising materials mentioning features or use of this software
 *    must display the following acknowledgement:
 *	This product includes software developed by the Computer Systems
 *	Engineering Group at Lawrence Berkeley Laboratory.
 * 4. Neither the name of the University nor of the Laboratory may be used
 *    to endorse or promote products derived from this software without
 *    specific prior written permission.
 *
 * THIS SOFTWARE IS PROVIDED BY THE REGENTS AND CONTRIBUTORS ``AS IS'' AND
 * ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
 * IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
 * ARE DISCLAIMED.  IN NO EVENT SHALL THE REGENTS OR CONTRIBUTORS BE LIABLE
 * FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
 * DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS
 * OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION)
 * HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT
 * LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY
 * OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF
 * SUCH DAMAGE.
 */

```

cp_to_utf_16le()函数是将一个字符串从指定编码方式转换为UTF-16LE编码的函数。它需要传入两个参数：一是要转换的code page，二是要转换的字符串。函数返回一个指向UTF-16LE编码字符串的指针。

这个函数实现的过程大致如下：

1. 计算目标UTF-16LE编码的字符数。
2. 尝试从指定的code page中对应的字符编码区域开始，查找字符串中所有字节，并将其转换为UTF-16LE编码。
3. 如果找到字符串中的所有字节，则返回包含这些字节的新字符串。
4. 如果出现错误，返回NULL。


```cpp
#ifdef _WIN32
#include <stdio.h>
#include <errno.h>

#include <pcap/pcap.h>	/* Needed for PCAP_ERRBUF_SIZE */

#include "charconv.h"

wchar_t *
cp_to_utf_16le(UINT codepage, const char *cp_string, DWORD flags)
{
	int utf16le_len;
	wchar_t *utf16le_string;

	/*
	 * Map from the specified code page to UTF-16LE.
	 * First, find out how big a buffer we'll need.
	 */
	utf16le_len = MultiByteToWideChar(codepage, flags, cp_string, -1,
	    NULL, 0);
	if (utf16le_len == 0) {
		/*
		 * Error.  Fail with EINVAL.
		 */
		errno = EINVAL;
		return (NULL);
	}

	/*
	 * Now attempt to allocate a buffer for that.
	 */
	utf16le_string = malloc(utf16le_len * sizeof (wchar_t));
	if (utf16le_string == NULL) {
		/*
		 * Not enough memory; assume errno has been
		 * set, and fail.
		 */
		return (NULL);
	}

	/*
	 * Now convert.
	 */
	utf16le_len = MultiByteToWideChar(codepage, flags, cp_string, -1,
	    utf16le_string, utf16le_len);
	if (utf16le_len == 0) {
		/*
		 * Error.  Fail with EINVAL.
		 * XXX - should this ever happen, given that
		 * we already ran the string through
		 * MultiByteToWideChar() to find out how big
		 * a buffer we needed?
		 */
		free(utf16le_string);
		errno = EINVAL;
		return (NULL);
	}
	return (utf16le_string);
}

```

这段代码是一个名为`utf_16le_to_cp`的函数，它将一个UTF-16LE编码的字符串映射到指定的编码页。

具体来说，这段代码的作用是将`utf16le_string`中的UTF-16LE编码的字符，转换成指定编码页的字符，并将结果存储在`cp_string`指向的内存区域。如果转换成功，函数将返回`cp_string`指向的内存区域；如果转换失败，函数将返回`NULL`，并输出错误信息。


```cpp
char *
utf_16le_to_cp(UINT codepage, const wchar_t *utf16le_string)
{
	int cp_len;
	char *cp_string;

	/*
	 * Map from UTF-16LE to the specified code page.
	 * First, find out how big a buffer we'll need.
	 * We convert composite characters to precomposed characters,
	 * as that's what Windows expects.
	 */
	cp_len = WideCharToMultiByte(codepage, WC_COMPOSITECHECK,
	    utf16le_string, -1, NULL, 0, NULL, NULL);
	if (cp_len == 0) {
		/*
		 * Error.  Fail with EINVAL.
		 */
		errno = EINVAL;
		return (NULL);
	}

	/*
	 * Now attempt to allocate a buffer for that.
	 */
	cp_string = malloc(cp_len * sizeof (char));
	if (cp_string == NULL) {
		/*
		 * Not enough memory; assume errno has been
		 * set, and fail.
		 */
		return (NULL);
	}

	/*
	 * Now convert.
	 */
	cp_len = WideCharToMultiByte(codepage, WC_COMPOSITECHECK,
	    utf16le_string, -1, cp_string, cp_len, NULL, NULL);
	if (cp_len == 0) {
		/*
		 * Error.  Fail with EINVAL.
		 * XXX - should this ever happen, given that
		 * we already ran the string through
		 * WideCharToMultiByte() to find out how big
		 * a buffer we needed?
		 */
		free(cp_string);
		errno = EINVAL;
		return (NULL);
	}
	return (cp_string);
}

```

This function appears to convert a given error string from UTC code point 16 to UTF-8 encoding, and then convert that to the local code page. It uses the Windows API WideCharToMultiByte to convert the error string to the target code page, and then copies the converted string to the buffer. If the conversion or copy fails, it returns an error message. The function returns 0 on success, or -1 if the user-supplied buffer is too small to hold the converted string.


```cpp
/*
 * Convert an error message string from UTF-8 to the local code page, as
 * best we can.
 *
 * The buffer is assumed to be PCAP_ERRBUF_SIZE bytes long; we truncate
 * if it doesn't fit.
 */
void
utf_8_to_acp_truncated(char *errbuf)
{
	wchar_t *utf_16_errbuf;
	int retval;
	DWORD err;

	/*
	 * Do this by converting to UTF-16LE and then to the local
	 * code page.  That means we get to use Microsoft's
	 * conversion routines, rather than having to understand
	 * all the code pages ourselves, *and* that this routine
	 * can convert in place.
	 */

	/*
	 * Map from UTF-8 to UTF-16LE.
	 * First, find out how big a buffer we'll need.
	 * Convert any invalid characters to REPLACEMENT CHARACTER.
	 */
	utf_16_errbuf = cp_to_utf_16le(CP_UTF8, errbuf, 0);
	if (utf_16_errbuf == NULL) {
		/*
		 * Error.  Give up.
		 */
		snprintf(errbuf, PCAP_ERRBUF_SIZE,
		    "Can't convert error string to the local code page");
		return;
	}

	/*
	 * Now, convert that to the local code page.
	 * Use the current thread's code page.  For unconvertable
	 * characters, let it pick the "best fit" character.
	 *
	 * XXX - we'd like some way to do what utf_16le_to_utf_8_truncated()
	 * does if the buffer isn't big enough, but we don't want to have
	 * to handle all local code pages ourselves; doing so requires
	 * knowledge of all those code pages, including knowledge of how
	 * characters are formed in thoe code pages so that we can avoid
	 * cutting a multi-byte character into pieces.
	 *
	 * Converting to an un-truncated string using Windows APIs, and
	 * then copying to the buffer, still requires knowledge of how
	 * characters are formed in the target code page.
	 */
	retval = WideCharToMultiByte(CP_THREAD_ACP, 0, utf_16_errbuf, -1,
	    errbuf, PCAP_ERRBUF_SIZE, NULL, NULL);
	if (retval == 0) {
		err = GetLastError();
		free(utf_16_errbuf);
		if (err == ERROR_INSUFFICIENT_BUFFER)
			snprintf(errbuf, PCAP_ERRBUF_SIZE,
			    "The error string, in the local code page, didn't fit in the buffer");
		else
			snprintf(errbuf, PCAP_ERRBUF_SIZE,
			    "Can't convert error string to the local code page");
		return;
	}
	free(utf_16_errbuf);
}
```

这段代码是一个 preprocessed header，即预处理指令。它的作用是在源代码文件被编译之前，对代码进行处理。这种情况下，通常是在预处理阶段做一些简单的字符串替换或者预先定义一些常量。

在这段代码中，#endif 指令是一个预处理指令，它表示一个预处理指令块。这个预处理指令块内部可以定义一些常量、字符串或者数学常数，它们在源代码编译之前就已经定义了，可以提高编译速度和代码的稳定性。

因此，这段代码的作用是定义了一些预处理指令常量，用于编译前处理。


```cpp
#endif

```

Guidelines for contributing
===========================

To report a security issue (segfault, buffer overflow, infinite loop, arbitrary
code execution etc) please send an e-mail to security@tcpdump.org, do not use
the bug tracker!

To report a non-security problem (failure to compile, failure to capture packets
properly, missing support for a network interface type or DLT) please check
first that it reproduces with the latest stable release of libpcap. If it does,
please check that the problem reproduces with the current git master branch of
libpcap. If it does (and it is not a security-related problem, otherwise see
above), please navigate to https://github.com/the-tcpdump-group/libpcap/issues
and check if the problem has already been reported. If it has not, please open
a new issue and provide the following details:

* libpcap version (e.g. from `tcpdump --version`)
* operating system name and version and any other details that may be relevant
  (`uname -a`, compiler name and version, CPU type etc.)
* `configure` or `cmake` flags if any were used
* statement of the problem
* steps to reproduce

Please note that if you know exactly how to solve the problem and the solution
would not be too intrusive, it would be best to contribute some development time
and open a pull request instead.

Still not sure how to do? Feel free to [subscribe](https://www.tcpdump.org/#mailing-lists)
to the mailing list tcpdump-workers@lists.tcpdump.org and ask!


# `libpcap/diag-control.h`

I'm sorry, but as an AI language model, I am not able to browse the internet and


```cpp
/* -*- Mode: c; tab-width: 8; indent-tabs-mode: 1; c-basic-offset: 8; -*- */
/*
 * Copyright (c) 1993, 1994, 1995, 1996, 1997
 *	The Regents of the University of California.  All rights reserved.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions
 * are met:
 * 1. Redistributions of source code must retain the above copyright
 *    notice, this list of conditions and the following disclaimer.
 * 2. Redistributions in binary form must reproduce the above copyright
 *    notice, this list of conditions and the following disclaimer in the
 *    documentation and/or other materials provided with the distribution.
 * 3. All advertising materials mentioning features or use of this software
 *    must display the following acknowledgement:
 *	This product includes software developed by the Computer Systems
 *	Engineering Group at Lawrence Berkeley Laboratory.
 * 4. Neither the name of the University nor of the Laboratory may be used
 *    to endorse or promote products derived from this software without
 *    specific prior written permission.
 *
 * THIS SOFTWARE IS PROVIDED BY THE REGENTS AND CONTRIBUTORS ``AS IS'' AND
 * ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
 * IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
 * ARE DISCLAIMED.  IN NO EVENT SHALL THE REGENTS OR CONTRIBUTORS BE LIABLE
 * FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
 * DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS
 * OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION)
 * HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT
 * LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY
 * OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF
 * SUCH DAMAGE.
 */

```

这段代码定义了一个名为“diag_control.h”的头文件。通过检查当前编译器是否支持PCAP库，以及是否支持Clang 2.8或GCC 4.6，来实现对#define的预处理。

具体来说，这段代码的作用是告诉编译器可以使用类似于“#pragma”的指令将一个标识符（例如“PCAP_DO_PRAGMA”）嵌入到头文件中定义中。通过这种方式，头文件中的代码可以被编译时识别为代码，而不是仅仅被视为声明。

这个头文件是用于定义和使用PCAP库的，特别是在进行编译测试时。头文件中包含的代码会在编译时被替换为相应的代码，从而允许开发人员使用这些库。


```cpp
#ifndef _diag_control_h
#define _diag_control_h

#include "pcap/compiler-tests.h"

#if PCAP_IS_AT_LEAST_CLANG_VERSION(2,8) || PCAP_IS_AT_LEAST_GNUC_VERSION(4,6)
  /*
   * Clang and GCC both support this way of putting pragmas into #defines.
   * We use it only if we have a compiler that supports it; see below
   * for the code that uses it and the #defines that control whether
   * that code is used.
   */
  #define PCAP_DO_PRAGMA(x) _Pragma (#x)
#endif

```

这段代码是用来 suppress warnings 这个警告信息的。它通过定义两个宏，来控制 int 类型中enum类型的值，何时会警告。

当定义这个代码时，它会输出 "DIAG_OFF_ENUM_SWITCH" 和 "DIAG_ON_ENUM_SWITCH"。前一个宏会输出 "warning(push)" 警告，后一个宏会输出 "warning(pop)" 警告。

警告信息是关于枚举类型变量在 switch 语句中使用时，可能会出现警告 "enum value not explicitly handled in switch"。

换句话说，当使用这个代码时，即使它定义了枚举类型，该警告仍然会弹出，但是警告的内容已经被转义了，或者被控制为不会弹出警告。


```cpp
/*
 * Suppress "enum value not explicitly handled in switch" warnings.
 * We may have to build on multiple different Windows SDKs, so we
 * may not be able to include all enum values in a switch, as they
 * won't necessarily be defined on all the SDKs, and, unlike
 * #defines, there's no easy way to test whether a given enum has
 * a given value.  It *could* be done by the configure script or
 * CMake tests.
 */
#if defined(_MSC_VER)
  #define DIAG_OFF_ENUM_SWITCH \
    __pragma(warning(push)) \
    __pragma(warning(disable:4061))
  #define DIAG_ON_ENUM_SWITCH \
    __pragma(warning(pop))
```

这段代码定义了两个宏，一个名为DIAG_OFF_ENUM_SWITCH，另一个名为DIAG_ON_ENUM_SWITCH。这两个宏都被定义为#define，意味着它们会在编译时替换为相应的定义。

DIAG_OFF_ENUM_SWITCH macro定义了一个switch语句，但是该语句只包含一个默认case。在Linux系统上，该宏会生成警告。

DIAG_ON_ENUM_SWITCH macro定义了一个switch语句，该语句包含两个附加的case，分别对应于BPF_FILTER中定义的几个函数。该宏会生成警告，告知开发人员在使用该代码时，如果缺少对于该警告的修复，则可能会导致程序出现问题。

该代码的作用是定义了一些宏，用于在编译时替换代码中关于switch语句的警告。如果定义的switch语句中包含有默认case，则该代码会生成DIAG_OFF_ONEMTL_WARNING警告。而如果没有默认case，则该代码会生成DIAG_ONEMTL_WARNING警告。


```cpp
#else
  #define DIAG_OFF_ENUM_SWITCH
  #define DIAG_ON_ENUM_SWITCH
#endif

/*
 * Suppress "switch statement has only a default case" warnings.
 * There's a switch in bpf_filter.c that only has additional
 * cases on Linux.
 */
#if defined(_MSC_VER)
  #define DIAG_OFF_DEFAULT_ONLY_SWITCH \
    __pragma(warning(push)) \
    __pragma(warning(disable:4065))
  #define DIAG_ON_DEFAULT_ONLY_SWITCH \
    __pragma(warning(pop))
```

This is a header file that defines several macro constants related to code analysis and diagnostics.

The header guards include a pragma warning from Clang, which indicates that some of the warnings in the source code may not be fixed and should be ignored.

The DIAG_OFF macro constants include a pragma directive that suppresses certain clang diagnostics, while the DIAG_ON macro constants include a pragma directive that includes the opposite DIAG_OFF directive.

These DIAGNOS are related to the width and flexibility of the source code.

The DIAG_OFF directive includes several pragmas that are used to disable certain clang warnings.

* DIAG_OFF_FLEX: This directive is used to disable flexible code analysis.
* DIAG_ON_FLEX: This directive is used to enable flexible code analysis.

The DIAG_OFF_FLEX directive is used to disable flexible code analysis, and the DIAG_ON_FLEX directive is used to enable flexible code analysis.

This header file is used to define the DIAGNOS related to the width and flexibility of the source code.


```cpp
#else
  #define DIAG_OFF_DEFAULT_ONLY_SWITCH
  #define DIAG_ON_DEFAULT_ONLY_SWITCH
#endif

/*
 * Suppress Flex, narrowing, and deprecation warnings.
 */
#if PCAP_IS_AT_LEAST_CLANG_VERSION(2,8)
  /*
   * This is Clang 2.8 or later; we can use "clang diagnostic
   * ignored -Wxxx" and "clang diagnostic push/pop".
   *
   * Suppress -Wdocumentation warnings; GCC doesn't support -Wdocumentation,
   * at least according to the GCC 7.3 documentation.  Apparently, Flex
   * generates code that upsets at least some versions of Clang's
   * -Wdocumentation.
   *
   * (This could be clang-cl, which defines _MSC_VER, so test this
   * before testing _MSC_VER.)
   */
  #define DIAG_OFF_FLEX \
    PCAP_DO_PRAGMA(clang diagnostic push) \
    PCAP_DO_PRAGMA(clang diagnostic ignored "-Wsign-compare") \
    PCAP_DO_PRAGMA(clang diagnostic ignored "-Wdocumentation") \
    PCAP_DO_PRAGMA(clang diagnostic ignored "-Wshorten-64-to-32") \
    PCAP_DO_PRAGMA(clang diagnostic ignored "-Wmissing-noreturn") \
    PCAP_DO_PRAGMA(clang diagnostic ignored "-Wunused-parameter") \
    PCAP_DO_PRAGMA(clang diagnostic ignored "-Wunreachable-code")
  #define DIAG_ON_FLEX \
    PCAP_DO_PRAGMA(clang diagnostic pop)

  /*
   * Suppress the only narrowing warnings you get from Clang.
   */
  #define DIAG_OFF_NARROWING \
    PCAP_DO_PRAGMA(clang diagnostic push) \
    PCAP_DO_PRAGMA(clang diagnostic ignored "-Wshorten-64-to-32")

  #define DIAG_ON_NARROWING \
    PCAP_DO_PRAGMA(clang diagnostic pop)

  /*
   * Suppress deprecation warnings.
   */
  #define DIAG_OFF_DEPRECATION \
    PCAP_DO_PRAGMA(clang diagnostic push) \
    PCAP_DO_PRAGMA(clang diagnostic ignored "-Wdeprecated-declarations")
  #define DIAG_ON_DEPRECATION \
    PCAP_DO_PRAGMA(clang diagnostic pop)
  #define DIAG_OFF_FORMAT_TRUNCATION
  #define DIAG_ON_FORMAT_TRUNCATION
```

这段代码定义了一系列条件定义，用于根据定义的平台版本来启用或禁用某些代码警告。警告信息包括：

1. 定义_MSC_VER，表示只有在MSC编译器版本中定义的标识符才有效。
2. DIAG_OFF_FLEX，定义了一系列代表Microsoft Visual Studio的代码，用于将__pragma(warning(disable:XXXX))和__pragma(warning(push/pop))的警告信息关闭。
3. DIAG_ON_FLEX，定义了一系列代表非Microsoft Visual Studio的代码，用于将__pragma(warning(push))和__pragma(warning(pop))的警告信息打开。
4. DIAG_OFF_NARROWING，定义了一系列代表非Microsoft Visual Studio的代码，用于将__pragma(warning(push))和__pragma(warning(disable:4242))的警告信息打开。
5. DIAG_ON_NARROWING，定义了一系列代表非Microsoft Visual Studio的代码，用于将__pragma(warning(pop))和__pragma(warning(disable:4242))的警告信息打开。
6. DIAG_OFF_DEPRECATION，定义了一系列代表非Microsoft Visual Studio的代码，用于将__pragma(warning(push))和__pragma(warning(disable:4996))的警告信息打开。
7. DIAG_ON_DEPRECATION，定义了一系列代表非Microsoft Visual Studio的代码，用于将__pragma(warning(pop))和__pragma(warning(disable:4996))的警告信息打开。
8. DIAG_OFF_FORMAT_TRUNCATION，定义了一系列代表非Microsoft Visual Studio的代码，用于将__pragma(warning(push))和__pragma(warning(disable:4242))的警告信息关闭。
9. DIAG_ON_FORMAT_TRUNCATION，定义了一系列代表非Microsoft Visual Studio的代码，用于将__pragma(warning(pop))和__pragma(warning(disable:4242))的警告信息关闭。


```cpp
#elif defined(_MSC_VER)
  /*
   * This is Microsoft Visual Studio; we can use __pragma(warning(disable:XXXX))
   * and __pragma(warning(push/pop)).
   *
   * Suppress signed-vs-unsigned comparison, narrowing, and unreachable
   * code warnings.
   */
  #define DIAG_OFF_FLEX \
    __pragma(warning(push)) \
    __pragma(warning(disable:4127)) \
    __pragma(warning(disable:4242)) \
    __pragma(warning(disable:4244)) \
    __pragma(warning(disable:4702))
  #define DIAG_ON_FLEX \
    __pragma(warning(pop))

  /*
   * Suppress narrowing warnings.
   */
  #define DIAG_OFF_NARROWING \
    __pragma(warning(push)) \
    __pragma(warning(disable:4242)) \
    __pragma(warning(disable:4311))
  #define DIAG_ON_NARROWING \
    __pragma(warning(pop))

  /*
   * Suppress deprecation warnings.
   */
  #define DIAG_OFF_DEPRECATION \
    __pragma(warning(push)) \
    __pragma(warning(disable:4996))
  #define DIAG_ON_DEPRECATION \
    __pragma(warning(pop))
  #define DIAG_OFF_FORMAT_TRUNCATION
  #define DIAG_ON_FORMAT_TRUNCATION
```

PCAP is a package for creating PCAP (Process Control Application Protocol) records. The DIAG_* options are used to enable or disable certain formatting and error messages in the PCAP output.

The DIAG_* options are defined as follows:

* DIAG_OFF_FLEX: This option suppresses all FleX-related messages in the PCAP output.
* DIAG_ON_FLEX: This option enables all FleX-related messages in the PCAP output.
* DIAG_OFF_NARROWING: This option suppresses all narrowing warnings in the PCAP output.
* DIAG_ON_NARROWING: This option enables all narrowing warnings in the PCAP output.
* DIAG_OFF_DEPRECATION: This option suppresses all deprecation warnings in the PCAP output.
* DIAG_ON_DEPRECATION: This option enables all deprecation warnings in the PCAP output.
* DIAG_OFF_FORMAT_TRUNCATION: This option suppresses all format-truncation= warnings in the PCAP output.
* DIAG_ON_FORMAT_TRUNCATION: This option enables all format-truncation= warnings in the PCAP output.

Note that PCAP is a C-based package and some of the DIAG_* options may not be valid in all versions.


```cpp
#elif PCAP_IS_AT_LEAST_GNUC_VERSION(4,6)
  /*
   * This is GCC 4.6 or later, or a compiler claiming to be that.
   * We can use "GCC diagnostic ignored -Wxxx" (introduced in 4.2)
   * and "GCC diagnostic push/pop" (introduced in 4.6).
   */
  #define DIAG_OFF_FLEX \
    PCAP_DO_PRAGMA(GCC diagnostic push) \
    PCAP_DO_PRAGMA(GCC diagnostic ignored "-Wsign-compare") \
    PCAP_DO_PRAGMA(GCC diagnostic ignored "-Wunused-parameter") \
    PCAP_DO_PRAGMA(GCC diagnostic ignored "-Wunreachable-code")
  #define DIAG_ON_FLEX \
    PCAP_DO_PRAGMA(GCC diagnostic pop)

  /*
   * GCC currently doesn't issue any narrowing warnings.
   */
  #define DIAG_OFF_NARROWING
  #define DIAG_ON_NARROWING

  /*
   * Suppress deprecation warnings.
   */
  #define DIAG_OFF_DEPRECATION \
    PCAP_DO_PRAGMA(GCC diagnostic push) \
    PCAP_DO_PRAGMA(GCC diagnostic ignored "-Wdeprecated-declarations")
  #define DIAG_ON_DEPRECATION \
    PCAP_DO_PRAGMA(GCC diagnostic pop)

  /*
   * Suppress format-truncation= warnings.
   * GCC 7.1 had introduced this warning option. Earlier versions (at least
   * one particular copy of GCC 4.6.4) treat the request as a warning.
   */
  #if PCAP_IS_AT_LEAST_GNUC_VERSION(7,1)
    #define DIAG_OFF_FORMAT_TRUNCATION \
      PCAP_DO_PRAGMA(GCC diagnostic push) \
      PCAP_DO_PRAGMA(GCC diagnostic ignored "-Wformat-truncation=")
    #define DIAG_ON_FORMAT_TRUNCATION \
      PCAP_DO_PRAGMA(GCC diagnostic pop)
  #else
   #define DIAG_OFF_FORMAT_TRUNCATION
   #define DIAG_ON_FORMAT_TRUNCATION
  #endif
```

这段代码定义了一系列标记(DIAG_OFF_FLEX, DIAG_ON_FLEX, DIAG_OFF_NARROWING, DIAG_ON_NARROWING, DIAG_OFF_DEPRECATION, DIAG_ON_DEPRECATION, DIAG_OFF_FORMAT_TRUNCATION, DIAG_ON_FORMAT_TRUNCATION)，用于报告编译器的错误和警告。其中，DIAG_OFF_FLEX表示编译器未检测到任何可用的 FleX 定义，DIAG_ON_FLEX表示编译器已检测到 FleX 定义，DIAG_OFF_NARROWING 和 DIAG_ON_NARROWING 表示编译器是否支持 arr 和 narg 重载，DIAG_OFF_DEPRECATION 和 DIAG_ON_DEPRECATION 表示编译器是否已弃用或过时了一些标记，DIAG_OFF_FORMAT_TRUNCATION 和 DIAG_ON_FORMAT_TRUNCATION 表示编译器是否支持格式化字符串中的字符串截断。这些标记可以用于在代码中捕获编译器错误和警告，并根据代码的状况给出相应的建议和提示。


```cpp
#else
  /*
   * Neither Visual Studio, nor Clang 2.8 or later, nor GCC 4.6 or later
   * or a compiler claiming to be that; there's nothing we know of that
   * we can do.
   */
  #define DIAG_OFF_FLEX
  #define DIAG_ON_FLEX
  #define DIAG_OFF_NARROWING
  #define DIAG_ON_NARROWING
  #define DIAG_OFF_DEPRECATION
  #define DIAG_ON_DEPRECATION
  #define DIAG_OFF_FORMAT_TRUNCATION
  #define DIAG_ON_FORMAT_TRUNCATION
#endif

```

This is a pragmatic solution to suppress warnings related to shadowing the global declaration. The code checks the version of Clang or GCC that is being used and sets the appropriateDIAG\_OFF\_BISON\_BYACC macro accordingly. If the code is being compiled with PCAP and the version is 2.8 or later, the code uses Clang 2.8 or later and the "-Wshadow" option to disable warnings. If the code is being compiled with GCC, the code checks if the version is 4.6 or later and claims to be able to use the "-Wshadow" option. If the version is not 2.8 or later and the compiler is not claiming to be able to use the "-Wshadow" option, the code falls back to using GCC's "-Wshadow" option.


```cpp
#ifdef YYBYACC
  /*
   * Berkeley YACC.
   *
   * It generates a global declaration of yylval, or the appropriately
   * prefixed version of yylval, in grammar.h, *even though it's been
   * told to generate a pure parser, meaning it doesn't have any global
   * variables*.  Bison doesn't do this.
   *
   * That causes a warning due to the local declaration in the parser
   * shadowing the global declaration.
   *
   * So, if the compiler warns about that, we turn off -Wshadow warnings.
   *
   * In addition, the generated code may have functions with unreachable
   * code, so suppress warnings about those.
   */
  #if PCAP_IS_AT_LEAST_CLANG_VERSION(2,8)
    /*
     * This is Clang 2.8 or later (including clang-cl, so test this
     * before _MSC_VER); we can use "clang diagnostic ignored -Wxxx".
     */
    #define DIAG_OFF_BISON_BYACC \
      PCAP_DO_PRAGMA(clang diagnostic ignored "-Wshadow") \
      PCAP_DO_PRAGMA(clang diagnostic ignored "-Wunreachable-code")
  #elif defined(_MSC_VER)
    /*
     * This is Microsoft Visual Studio; we can use
     * __pragma(warning(disable:XXXX)).
     */
    #define DIAG_OFF_BISON_BYACC \
      __pragma(warning(disable:4702))
  #elif PCAP_IS_AT_LEAST_GNUC_VERSION(4,6)
    /*
     * This is GCC 4.6 or later, or a compiler claiming to be that.
     * We can use "GCC diagnostic ignored -Wxxx" (introduced in 4.2,
     * but it may not actually work very well prior to 4.6).
     */
    #define DIAG_OFF_BISON_BYACC \
      PCAP_DO_PRAGMA(GCC diagnostic ignored "-Wshadow") \
      PCAP_DO_PRAGMA(GCC diagnostic ignored "-Wunreachable-code")
  #else
    /*
     * Neither Clang 2.8 or later nor GCC 4.6 or later or a compiler
     * claiming to be that; there's nothing we know of that we can do.
     */
    #define DIAG_OFF_BISON_BYACC
  #endif
```

With this configuration, the `default case` will be allowed but the warnings for `unreachable code` and `unreachable control流` will be suppressed.

This configuration is useful when there are specific cases where the compiler is not警告 for code that should not be executed, but the warnings can be distracting. By disabling warnings for these cases, the code still runs without the overhead of the warnings and the compile will be faster.

However, it's important to note that this configuration suppress warnings that can indicate potential problems with the code, and can potentially cause the code to fail at build time if not handled properly. Therefore, it's important to use this configuration carefully and carefully consider the implications of disabling warnings.


```cpp
#else
  /*
   * Bison.
   *
   * The generated code may have functions with unreachable code and
   * switches with only a default case, so suppress warnings about those.
   */
  #if PCAP_IS_AT_LEAST_CLANG_VERSION(2,8)
    /*
     * This is Clang 2.8 or later (including clang-cl, so test this
     * before _MSC_VER); we can use "clang diagnostic ignored -Wxxx".
     */
    #define DIAG_OFF_BISON_BYACC \
      PCAP_DO_PRAGMA(clang diagnostic ignored "-Wunreachable-code")
  #elif defined(_MSC_VER)
    /*
     * This is Microsoft Visual Studio; we can use
     * __pragma(warning(disable:XXXX)).
     *
     * Suppress some /Wall warnings.
     */
    #define DIAG_OFF_BISON_BYACC \
      __pragma(warning(disable:4065)) \
      __pragma(warning(disable:4127)) \
      __pragma(warning(disable:4242)) \
      __pragma(warning(disable:4244)) \
      __pragma(warning(disable:4702))
  #elif PCAP_IS_AT_LEAST_GNUC_VERSION(4,6)
    /*
     * This is GCC 4.6 or later, or a compiler claiming to be that.
     * We can use "GCC diagnostic ignored -Wxxx" (introduced in 4.2,
     * but it may not actually work very well prior to 4.6).
     */
    #define DIAG_OFF_BISON_BYACC \
      PCAP_DO_PRAGMA(GCC diagnostic ignored "-Wunreachable-code")
  #else
    /*
     * Neither Clang 2.8 or later nor GCC 4.6 or later or a compiler
     * claiming to be that; there's nothing we know of that we can do.
     */
    #define DIAG_OFF_BISON_BYACC
  #endif
```

这段代码是一个 preprocessor 指令，它通过 PCAP_IS_AT_LEAST_GNUC_VERSION() 检查当前编译器版本是否支持 at_least() 宏。如果是，那么它会定义一个名为 PCAP_UNREACHABLE 的 macros，其值为 __builtin_unreachable()，这个 macro 在 GCC 中允许程序继续执行，即使它包含一个无限空循环。这个代码段的意义是让程序员在使用 at_least() 宏时要注意，因为它可能使程序崩溃或产生不可预料的行为。


```cpp
#endif

/*
 * GCC needs this on AIX for longjmp().
 */
#if PCAP_IS_AT_LEAST_GNUC_VERSION(5,1)
  /*
   * Beware that the effect of this builtin is more than just squelching the
   * warning! GCC trusts it enough for the process to segfault if the control
   * flow reaches the builtin (an infinite empty loop in the same context would
   * squelch the warning and ruin the process too, albeit in a different way).
   * So please remember to use this very carefully.
   */
  #define PCAP_UNREACHABLE __builtin_unreachable();
#else
  #define PCAP_UNREACHABLE
```

这是一个C/C++代码片段，其中包含两个预处理指令。

第一个预处理指令 `#ifdef _diag_control_h` 是一个条件编译，它会判断是否定义了名为 `_diag_control_h` 的头文件。如果已经定义了这个头文件，那么代码会继续执行，否则会跳过这个部分，不会被执行。

第二个预处理指令 `#endif` 是一个关闭预处理指令，它会将所有预处理指令之后的代码都视为无效，不会对代码产生任何影响。

因此，这段代码的作用是用于定义名为 `_diag_control_h` 的头文件，并且在预处理指令 `#ifdef _diag_control_h` 的作用域内，如果已经定义了这个头文件，那么代码会继续执行，否则跳过这个部分，不会被执行。


```cpp
#endif

#endif /* _diag_control_h */

```

# `libpcap/dlpisubs.c`

这段代码定义了一个文件名为"libdlpi_common.h"的头文件，其中包含与pcap-[dlpi,libdlpi]相关联的常见函数。

pcap是一个网络协议分析器，可以用来捕获和分析网络数据包。libdlpi是一个与pcap相关的库，提供了一系列用于分析、处理和捕获数据包的函数。

根据pcap-[dlpi,libdlpi]的源代码，这段代码可以解释为：

1. 定义了两个宏定义：ATANU_GHASS_宏定义了ATANU Ghosh（原文作者）的名字和电子邮件地址，而GUY_HARRIS_宏定义了马克·皮佐利托（Guy Harris）的名字和电子邮件地址。

2. 引入了两个函数：h攝取（ht）函数和i摄取（hi）函数。这两个函数在DLPI库中定义，用于从TCP连接的输入流中读取数据包。

3. 通过宏定义将函数名称映射为实际函数的地址。这个功能在pcap-[dlpi,libdlpi]中调用，以便在需要时动态地加载这些函数。


```cpp
/*
 * This code is derived from code formerly in pcap-dlpi.c, originally
 * contributed by Atanu Ghosh (atanu@cs.ucl.ac.uk), University College
 * London, and subsequently modified by Guy Harris (guy@alum.mit.edu),
 * Mark Pizzolato <List-tcpdump-workers@subscriptions.pizzolato.net>,
 * Mark C. Brown (mbrown@hp.com), and Sagun Shakya <Sagun.Shakya@Sun.COM>.
 */

/*
 * This file contains dlpi/libdlpi related common functions used
 * by pcap-[dlpi,libdlpi].c.
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
```

这段代码定义了一个头文件DL_IPATM，其中DL_IPATM以0x12标识表示这是一个ATM（Asynchronous Transfer Method，异步传输方法）Classical IP接口。然后通过#ifdef和#define的作用，告诉编译器在定义DL_IPATM之前和之后的内容。

在#ifdef部分，定义了一个名为CHUNKSIZE的常量，表示一个bufmod chunk（一个数据缓冲区）的大小，用于向上游传输数据。然后通过一个宏定义，将CHUNKSIZE的值定义为65536，这个值大于似乎可用的最大值，可以减少数据包丢弃的数量。

在#define部分，定义了一个名为DL_IPATM_DESC的宏，表示DL_IPATM接口的描述。

最后，通过#elif和#define的作用，将DL_IPATM定义为0x12，并将CHUNKSIZE的定义解析为const long。


```cpp
#endif

#ifndef DL_IPATM
#define DL_IPATM	0x12	/* ATM Classical IP interface */
#endif

#ifdef HAVE_SYS_BUFMOD_H
	/*
	 * Size of a bufmod chunk to pass upstream; that appears to be the
	 * biggest value to which you can set it, and setting it to that value
	 * (which is bigger than what appears to be the Solaris default of 8192)
	 * reduces the number of packet drops.
	 */
#define	CHUNKSIZE	65536

	/*
	 * Size of the buffer to allocate for packet data we read; it must be
	 * large enough to hold a chunk.
	 */
```

这段代码定义了两个宏，分别是`PKTBUFSIZE`和`MAXDLBUF`，同时也定义了一个条件编译块。

条件编译块是C语言中一种特殊的编译技巧，用于根据某一特定条件来选择不同的代码块。当条件为真时，编译器会执行该条件编译块内的代码，否则跳过该条件编译块。

`PKTBUFSIZE`定义了一个字符串缓冲区的大小，用于存放数据分片的结果。这个缓冲区的大小是根据`MAXDLBUF`来计算的，`MAXDLBUF`是一个未定义的常量，但是在编译时会被初始化为`8192`，因此这个宏定义了一个大小为`MAXDLBUF`字节的数据缓冲区。

`MAXDLBUF`定义了一个字符串缓冲区所能容纳的最大数据包大小。这个常量在后面会被用于计算出`PKTBUFSIZE`。

通过这两个宏定义，我们可以看出这段代码定义了一个字符串缓冲区，用于存放数据分片的结果。这个缓冲区的大小可以根据`MAXDLBUF`来计算，也可以根据具体的需求来调整。


```cpp
#define	PKTBUFSIZE	CHUNKSIZE

#else /* HAVE_SYS_BUFMOD_H */

	/*
	 * Size of the buffer to allocate for packet data we read; this is
	 * what the value used to be - there's no particular reason why it
	 * should be tied to MAXDLBUF, but we'll leave it as this for now.
	 */
#define	MAXDLBUF	8192
#define	PKTBUFSIZE	(MAXDLBUF * sizeof(bpf_u_int32))

#endif

#include <sys/types.h>
```

这段代码是一个用于获取 Linux 系统的运行时元数据的工具。它通过引入头文件 <sys/time.h> 和 <sys/bufmod.h> 来访问操作系统中的时间戳和缓冲区模块信息。

具体来说，这段代码的作用是读取并解析系统中的时间戳数据，包括系统运行时间、进程运行时间、以及其他与时间相关的信息。它将这些数据存储在 <std::time_t> 类型的变量中，以便用户进行进一步的处理和使用。


```cpp
#include <sys/time.h>
#ifdef HAVE_SYS_BUFMOD_H
#include <sys/bufmod.h>
#endif
#include <sys/dlpi.h>
#include <sys/stream.h>

#include <errno.h>
#include <memory.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stropts.h>
#include <unistd.h>

```

这段代码是一个简单的 Linux 库函数，它的作用是检查系统是否支持库文件 `libdlpi.h`，如果支持，则包含该文件，否则输出错误信息。

具体来说，代码首先包含一个预处理指令 `#ifdef HAVE_LIBDLPI`，如果当前源文件中已经定义了该预处理指令，则继续下载其余代码；否则执行 `#include <libdlpi.h>`，包含该库文件。

接着，代码又包含另一个预处理指令 `#ifdef HAVE_SYS_BUFMOD_H`，该指令用于检查系统是否支持 `sys_bufmod.h` 头文件，如果支持，则包含该文件，否则输出错误信息。但是，该指令在本源文件中没有被定义，因此不会产生影响。

最后，代码定义了一个名为 `pcap_stream_err` 的函数，用于在流式数据包输入或输出时打印错误信息。


```cpp
#ifdef HAVE_LIBDLPI
#include <libdlpi.h>
#endif

#include "pcap-int.h"
#include "dlpisubs.h"

#ifdef HAVE_SYS_BUFMOD_H
static void pcap_stream_err(const char *, int, char *);
#endif

/*
 * Get the packet statistics.
 */
int
```

这段代码是一个用于计算数据链路层 PDU（数据链路层数据单元）的统计信息的函数，其接受一个数据链路层协议头指针（pcap_t 类型）和一个数据链路层统计信息结构体（struct pcap_stat 类型）。

函数首先定义了一个名为 pd 的结构体，它指针了一个数据链路层协议头指针（pcap_t 类型）和一个数据链路层统计信息结构体（struct pcap_stat 类型）。

函数接着定义了一个名为 ps 的结构体，它包含一个指向数据链路层协议头指针（pcap_t 类型）的指针和一个包含数据链路层统计信息（ps_recv 和 ps_drop）的整型变量。

函数内部通过一些宏定义来设置 ps 结构体中的统计信息，包括添加 ps_drop 统计信息，它是通过计算 ps 结构体中的 ps_recv 和 ps_drop 来得到的。

函数最后返回 0，表示函数成功计算出了数据链路层的统计信息。


```cpp
pcap_stats_dlpi(pcap_t *p, struct pcap_stat *ps)
{
	struct pcap_dlpi *pd = p->priv;

	/*
	 * "ps_recv" counts packets handed to the filter, not packets
	 * that passed the filter.  As filtering is done in userland,
	 * this would not include packets dropped because we ran out
	 * of buffer space; in order to make this more like other
	 * platforms (Linux 2.4 and later, BSDs with BPF), where the
	 * "packets received" count includes packets received but dropped
	 * due to running out of buffer space, and to keep from confusing
	 * applications that, for example, compute packet drop percentages,
	 * we also make it count packets dropped by "bufmod" (otherwise we
	 * might run the risk of the packet drop count being bigger than
	 * the received-packet count).
	 *
	 * "ps_drop" counts packets dropped by "bufmod" because of
	 * flow control requirements or resource exhaustion; it doesn't
	 * count packets dropped by the interface driver, or packets
	 * dropped upstream.  As filtering is done in userland, it counts
	 * packets regardless of whether they would've passed the filter.
	 *
	 * These statistics don't include packets not yet read from
	 * the kernel by libpcap, but they may include packets not
	 * yet read from libpcap by the application.
	 */
	*ps = pd->stat;

	/*
	 * Add in the drop count, as per the above comment.
	 */
	ps->ps_recv += ps->ps_drop;
	return (0);
}

```

这段代码是一个条件判断，用于检查编译时使用的CPU是否支持对齐加载。对齐加载是指在CPU从内存中读取数据时，先将数据对齐到CPU缓存中，然后再从缓存中读取数据。对齐可以提高程序的读取速度，但需要在CPU支持的情况下进行编译。

代码首先定义了一个名为REQUIRE_ALIGNMENT的宏，表示如果编译时使用的CPU支持对齐加载，那么定义为真，否则定义为假。

接着，代码通过一个多条件判断语句来检查哪些CPU支持对齐加载。具体来说，代码检查是否定义了__i386__、_M_IX86、__X86__、__x86_64__、__arm__、_M_ARM__、__aarch64__、__m68k__、(__mc68000__)和(__mc68010__))，如果其中任何一个定义了__ARM__或者__x86_64__，那么定义为真，否则定义为假。如果所有CPU都支持对齐加载，那么定义为真，否则定义为假。如果定义中有一个或多个CPU不支持对齐加载，那么执行 REQUIRE_ALIGNMENT 宏，表示需要进行对齐加载，否则输出 "No, it doesn't."。

最后，代码定义了一个名为__PACKET_CALLBACK__ 的函数，表示回调函数的地址，用于处理每个数据包的读取情况。


```cpp
/*
 * Does the processor for which we're compiling this support aligned loads?
 */
#if (defined(__i386__) || defined(_M_IX86) || defined(__X86__) || defined(__x86_64__) || defined(_M_X64)) || \
    (defined(__arm__) || defined(_M_ARM) || defined(__aarch64__)) || \
    (defined(__m68k__) && (!defined(__mc68000__) && !defined(__mc68010__))) || \
    (defined(__ppc__) || defined(__ppc64__) || defined(_M_PPC) || defined(_ARCH_PPC) || defined(_ARCH_PPC64)) || \
    (defined(__s390__) || defined(__s390x__) || defined(__zarch__))
    /* Yes, it does. */
#else
    /* No, it doesn't. */
    #define REQUIRE_ALIGNMENT
#endif

/*
 * Loop through the packets and call the callback for each packet.
 * Return the number of packets read.
 */
```

这段代码是一个名为 `pcap_process_pkts` 的函数，属于 `pcap_t` 类的成员函数。

它的作用是处理 `pcap_pkts` 结构中的数据，具体流程如下：

1. 遍历 `pcap_pkts` 结构中的数据，从开始位置 `bufp` 开始，长度为 `len` 的数据块中。
2. 对于每个数据块，提取出对应的 `pcap_pkthdr` 结构，包含以下成员：
		- `protocol`: 数据协议，如 `TCP`、`UDP` 等。
		- `dst_port`: 目标端口。
		- `src_port`: 源端口。
		- `len`: 数据长度。
		- `padding`: 是否为填充数据。
		- `frame_id`: 帧标识，用于记录数据包的唯一性。
		- `data_offset`: 数据偏移量，用于定位数据 start 位置。
		- `坠落保护和随机数`: 支持的功能，目前未知。
		- `承兑`: 是否启用承兑。
		- `sdyn`: 是否启用 Sdyn 算法。
		- `付酬比例`: 计费比例，目前未知。
		- `internal_ip`: 内部 IP。
		- `external_ip`: 外部 IP。
		- `port_discovery_ period`: 端口发现周期，以秒为间隔。
		- `first_標記`: 是否标记数据包为交互的第一个数据。
		- `last_boundary_标记`: 是否标记数据包为交互的最后一个数据。
		- `pkt_免税`: 是否对某个数据包进行免税，目前未知。
		- `pkt_獲回應答`: 是否对某个数据包进行应答，目前未知。
		- `pkt_包含`: 是否包含对于某个数据包的 `pkt_免税` 标记。
		- `local_port`: 本地端口。
		- `remote_port`: 远程端口。
		- `num_部门`: 数据包的部门数量。
		- `ref_fragment_number`: 参考片段的数量。
		- `ref_packet_number`: 参考数据包的数量。
		- `remaining_ fragment_number`: 剩余片段的数量。
		- `next_remaining_ fragment_number`: 下一个剩余片段的数量。
		- `fragment_start`: 片段的起始位置。
		- `fragment_end`: 片段的结束位置。
		- `num_pkts`: 数据包的数量。
		- `dev`: 设备描述符，用于标识网络接口。

3. 将提取出的 `pcap_pkthdr` 结构体中的成员复制到 `bufp + n` 位置，然后 `n` 自增。
4. 如果数据块是填充数据，将 `n` 设置为 `INT_MAX`。
5. 返回处理过程中 `n` 的值。


```cpp
int
pcap_process_pkts(pcap_t *p, pcap_handler callback, u_char *user,
	int count, u_char *bufp, int len)
{
	struct pcap_dlpi *pd = p->priv;
	int n, caplen, origlen;
	u_char *ep, *pk;
	struct pcap_pkthdr pkthdr;
#ifdef HAVE_SYS_BUFMOD_H
	struct sb_hdr *sbp;
#ifdef REQUIRE_ALIGNMENT
	struct sb_hdr sbhdr;
#endif
#endif

	/*
	 * Loop through packets.
	 *
	 * This assumes that a single buffer of packets will have
	 * <= INT_MAX packets, so the packet count doesn't overflow.
	 */
	ep = bufp + len;
	n = 0;

```

这段代码是一个使用C预处理指令（#ifdef）的应用。它检查系统是否支持从文件中读取数据，然后定义了一个名为“bufp”的变量，用于保存文件读取到的数据缓冲区。

while循环会不断地读取文件数据，直到遇到一个符合条件的末尾字符'\0'（'\0'代表文件末尾）。如果是这种情况，循环将会立即终止，并返回一个负数，表示我们遇到了问题，需要强制退出循环。否则，将继续处理文件数据，并将已经读取的数据数量存储在“bufp”变量中，然后循环下次调用该函数时，将不需要再次询问文件读取器是否还有数据。

判断条件为：当已经处理完所有数据后，调用“pcap_breakloop()”函数。如果这个函数已经被调用，那么将清除“break_loop”标志，并返回-2，表示我们已经需要退出循环了。否则，将“break_loop”标志设置为1，并返回当前已经读取的数据数量，以便下一次循环时处理。


```cpp
#ifdef HAVE_SYS_BUFMOD_H
	while (bufp < ep) {
		/*
		 * Has "pcap_breakloop()" been called?
		 * If so, return immediately - if we haven't read any
		 * packets, clear the flag and return -2 to indicate
		 * that we were told to break out of the loop, otherwise
		 * leave the flag set, so that the *next* call will break
		 * out of the loop without having read any packets, and
		 * return the number of packets we've processed so far.
		 */
		if (p->break_loop) {
			if (n == 0) {
				p->break_loop = 0;
				return (-2);
			} else {
				p->bp = bufp;
				p->cc = ep - bufp;
				return (n);
			}
		}
```

这段代码是一个C语言中的条件编译语句，用于根据不同的编译器设置是否需要对一个缓冲区（bufp）进行对齐。

如果没有定义REQUIRE_ALIGNMENT，那么第一行代码会执行，否则第二行代码会执行。第二行代码会首先检查bufp是否对齐，如果不对齐，则将bufp的首地址（&sbhdr）保存到sbhdr中，然后使用memcpy函数将bufp中从首地址到sizeof(*sbp)的内存复制到sbhdr中。这将导致bufp中的所有元素都按照int类型对齐，从而确保在后续使用时，int类型的变量能够正确地访问这些元素。

如果定义了REQUIRE_ALIGNMENT，那么第二行代码将首先尝试从bufp中读取一个long类型的值。如果不能成功，则执行第一行代码，对bufp进行对齐，然后将sbhdr保存到bufp中，将&sbhdr保存到sbhdr中，将bufp中从首地址到sizeof(*sbp)的内存复制到sbhdr中，这将确保在后续使用时，long类型的变量能够正确地访问这些元素。

然后，pd->stat.ps_drop = sbp->sbh_drops，pk = bufp + sizeof(*sbp)，bufp += sbp->sbh_totlen，origlen = sbp->sbh_origlen，caplen = sbp->sbh_msglen，这将在程序中输出一些统计信息。


```cpp
#ifdef REQUIRE_ALIGNMENT
		if ((long)bufp & 3) {
			sbp = &sbhdr;
			memcpy(sbp, bufp, sizeof(*sbp));
		} else
#endif
			sbp = (struct sb_hdr *)bufp;
		pd->stat.ps_drop = sbp->sbh_drops;
		pk = bufp + sizeof(*sbp);
		bufp += sbp->sbh_totlen;
		origlen = sbp->sbh_origlen;
		caplen = sbp->sbh_msglen;
#else
		origlen = len;
		caplen = min(p->snapshot, len);
		pk = bufp;
		bufp += caplen;
```

这段代码是一个网络协议栈中的函数，它的作用是处理接收数据。函数接收一个长度为 origlen 的数据包，使用一个名为 pcap_filter 的函数，根据数据包中的 insn 字段判断是否匹配目标规则，然后使用 pcap_create 函数创建一个新的数据包，传递给 pcap_filter。如果匹配成功，将更新 ps_recv 计数器，使用时间戳获取当前时间，如果获取失败，使用系统调用函数 gettimeofday，获取当前时间，然后更新 pkthdr 结构中的一些字段，最后输出数据包给用户。如果 caplen 的长度超过 10000，将 caplen 设置为 10000，否则输出数据包。


```cpp
#endif
		++pd->stat.ps_recv;
		if (pcap_filter(p->fcode.bf_insns, pk, origlen, caplen)) {
#ifdef HAVE_SYS_BUFMOD_H
			pkthdr.ts.tv_sec = sbp->sbh_timestamp.tv_sec;
			pkthdr.ts.tv_usec = sbp->sbh_timestamp.tv_usec;
#else
			(void) gettimeofday(&pkthdr.ts, NULL);
#endif
			pkthdr.len = origlen;
			pkthdr.caplen = caplen;
			/* Insure caplen does not exceed snapshot */
			if (pkthdr.caplen > (bpf_u_int32)p->snapshot)
				pkthdr.caplen = (bpf_u_int32)p->snapshot;
			(*callback)(user, &pkthdr, pk);
			if (++n >= count && !PACKET_COUNT_IS_UNLIMITED(count)) {
				p->cc = ep - bufp;
				p->bp = bufp;
				return (n);
			}
		}
```

I'm sorry, I am not able to understand the question. Could you please provide more context or clarify what you are asking?


```cpp
#ifdef HAVE_SYS_BUFMOD_H
	}
#endif
	p->cc = 0;
	return (n);
}

/*
 * Process the mac type. Returns -1 if no matching mac type found, otherwise 0.
 */
int
pcap_process_mactype(pcap_t *p, u_int mactype)
{
	int retv = 0;

	switch (mactype) {

	case DL_CSMACD:
	case DL_ETHER:
		p->linktype = DLT_EN10MB;
		p->offset = 2;
		/*
		 * This is (presumably) a real Ethernet capture; give it a
		 * link-layer-type list with DLT_EN10MB and DLT_DOCSIS, so
		 * that an application can let you choose it, in case you're
		 * capturing DOCSIS traffic that a Cisco Cable Modem
		 * Termination System is putting out onto an Ethernet (it
		 * doesn't put an Ethernet header onto the wire, it puts raw
		 * DOCSIS frames out on the wire inside the low-level
		 * Ethernet framing).
		 */
		p->dlt_list = (u_int *)malloc(sizeof(u_int) * 2);
		/*
		 * If that fails, just leave the list empty.
		 */
		if (p->dlt_list != NULL) {
			p->dlt_list[0] = DLT_EN10MB;
			p->dlt_list[1] = DLT_DOCSIS;
			p->dlt_count = 2;
		}
		break;

	case DL_FDDI:
		p->linktype = DLT_FDDI;
		p->offset = 3;
		break;

	case DL_TPR:
		/* XXX - what about DL_TPB?  Is that Token Bus?  */
		p->linktype = DLT_IEEE802;
		p->offset = 2;
		break;

```

这段代码是一个条件编译语句，用于根据特定的头文件是否定义了SOLARIS、DL_IPV4或DL_IPV6头文件，来决定使用哪种链路地址类型。

具体来说，如果头文件HAVE_SOLARIS定义了DL_IPATM头文件，则执行case DL_IPATM;，即跳转到对应的代码块。如果头文件DL_IPV4或DL_IPV6定义了DL_IPV4或DL_IPV6头文件，则执行case DL_IPV4或case DL_IPV6;，即跳转到对应的代码块。如果头文件HAVE_SOLARIS未定义任何链路地址类型，则执行default;，即不执行任何代码。

在执行case DL_IPV4或case DL_IPV6；时，p->linktype和p->offset都指向0，即表示未定义链路地址类型，并且不考虑LANE和LLC封装。


```cpp
#ifdef HAVE_SOLARIS
	case DL_IPATM:
		p->linktype = DLT_SUNATM;
		p->offset = 0;  /* works for LANE and LLC encapsulation */
		break;
#endif

#ifdef DL_IPV4
	case DL_IPV4:
		p->linktype = DLT_IPV4;
		p->offset = 0;
		break;
#endif

#ifdef DL_IPV6
	case DL_IPV6:
		p->linktype = DLT_IPV6;
		p->offset = 0;
		break;
```

这段代码是一个 conditional 预处理指令，它检查是否定义了 DL_IPNET 头信息。如果没有定义，则执行dl_ipnet宏，其作用是设置数据链路层设备（p）的 linktype 字段为 DLT_RAW，并将 offset 字段设置为 0。如果定义了 DL_IPNET 头信息，则不会执行宏，从而避免不必要的额外开销。


```cpp
#endif

#ifdef DL_IPNET
	case DL_IPNET:
		/*
		 * XXX - DL_IPNET devices default to "raw IP" rather than
		 * "IPNET header"; see
		 *
		 *    https://seclists.org/tcpdump/2009/q1/202
		 *
		 * We'd have to do DL_IOC_IPNET_INFO to enable getting
		 * the IPNET header.
		 */
		p->linktype = DLT_RAW;
		p->offset = 0;
		break;
```

这段代码是一个 C 语言函数，名为 `未知 mactype`。它定义了一个默认的错误处理函数，用于处理在使用 MOF（MOFundamental）库时出现的unknown mactype错误。

函数的作用是：
1. 如果 error 变量为空，则创建一个大小为 PCAP_ERRBUF_SIZE 的字符数组，并使用 snprintf() 函数在其中填充错误信息。然后将错误信息存储到 p->errbuf 指向的字符数组中。此时，函数返回 -1。
2. 如果 error 变量不为空，则执行以下操作：
a. 创建一个字符数组，使用 snprintf() 函数在其中填充错误信息。然后将错误信息存储到该数组中。此时，函数返回 0。
b. 由于已经执行了前两步，因此将 p->errbuf 指向该错误信息字符数组，以便用户可以访问和打印错误信息。

总的来说，这段代码的主要作用是处理在使用 MOF 库时出现的unknown mactype错误。通过创建一个错误信息字符数组，并尝试从错误信息中提取出错误信息，从而使得用户能够更好地了解错误情况。


```cpp
#endif

	default:
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "unknown mactype 0x%x",
		    mactype);
		retv = -1;
	}

	return (retv);
}

#ifdef HAVE_SYS_BUFMOD_H
/*
 * Push and configure the buffer module. Returns -1 for error, otherwise 0.
 */
```

This is a code snippet for the Linux command `strioctl()` that manages to configure the receive-chunk timeout and the receive-chunk size for a TCP-like socket. The code may compile on some systems, but it will raise errors and return values that are not meaningful on others.

The code has several errors. First, it assumes that the `immediate` option of the `snaplen` parameter is set to `0`, which means that the `strioctl()` function should not perform any immediate data delivery. This is incorrect. Second, it sets the timeout to zero for the `immediate` option, which is also incorrect. The correct values for the `immediate` option should be `0` and `1`, respectively.

The third error is that `strioctl()` has a return type of `int`, but it is using `void` instead. This is not a compile-time error, but it will cause the program to print a stack-related error message.

The fourth error is a typo in the variable `chunksize`. It should be `sizeof(chunksize)` instead of `CHUNKSIZE`.

To fix these errors, the code can be updated as follows:
```cpp
#include <sys/types.h>
#include <sys/socket.h>
#include <errno.h>

int strioctl(int sockfd, int cmd, int argc, char *argv[]) {
 int opt = 0;
 int timeout = 0;
 int chunksize = 0;

 /* Set the timeout for immediate delivery */
 if (cmd == 'i') {
   timeout = 0;
   immediate = 1;
 } else if (cmd == 'o') {
   opt = cmd != 'o' ? 'r' : 'o';
   timeout = st prioritieset(cmd == 'i' ? (int)immediate : timeout, timeout, cmd);
   immediate = 0;
   chunksize = cmd == 'i' ? (int)immediate : CHUNKSIZE;
 }

 return (cmd == 'w' ? wspawnsubmit(sockfd, opt, argc, argv) : cmd == 'z' ? zincmd(sockfd, cmd, argc, argv) : cmd == 'x' ? exit(errno) : 0);
}
```


```cpp
int
pcap_conf_bufmod(pcap_t *p, int snaplen)
{
	struct timeval to;
	bpf_u_int32 ss, chunksize;

	/* Non-standard call to get the data nicely buffered. */
	if (ioctl(p->fd, I_PUSH, "bufmod") != 0) {
		pcap_stream_err("I_PUSH bufmod", errno, p->errbuf);
		return (-1);
	}

	ss = snaplen;
	if (ss > 0 &&
	    strioctl(p->fd, SBIOCSSNAP, sizeof(ss), (char *)&ss) != 0) {
		pcap_stream_err("SBIOCSSNAP", errno, p->errbuf);
		return (-1);
	}

	if (p->opt.immediate) {
		/* Set the timeout to zero, for immediate delivery. */
		to.tv_sec = 0;
		to.tv_usec = 0;
		if (strioctl(p->fd, SBIOCSTIME, sizeof(to), (char *)&to) != 0) {
			pcap_stream_err("SBIOCSTIME", errno, p->errbuf);
			return (-1);
		}
	} else {
		/* Set up the bufmod timeout. */
		if (p->opt.timeout != 0) {
			to.tv_sec = p->opt.timeout / 1000;
			to.tv_usec = (p->opt.timeout * 1000) % 1000000;
			if (strioctl(p->fd, SBIOCSTIME, sizeof(to), (char *)&to) != 0) {
				pcap_stream_err("SBIOCSTIME", errno, p->errbuf);
				return (-1);
			}
		}

		/* Set the chunk length. */
		chunksize = CHUNKSIZE;
		if (strioctl(p->fd, SBIOCSCHUNK, sizeof(chunksize), (char *)&chunksize)
		    != 0) {
			pcap_stream_err("SBIOCSCHUNKP", errno, p->errbuf);
			return (-1);
		}
	}

	return (0);
}
```

这段代码定义了一个名为`pcap_alloc_databuf`的函数，属于`pcap_core`库。它的作用是分配一个数据缓冲区（databuf），并将结果存储在`p->bufsize`中。如果内存分配失败，函数将返回负数。

具体实现包括以下几步：

1. 检查系统是否支持缓冲区操作，如果支持，则将`pkbuffersize`作为参数返回，否则返回`-1`。
2. 分配一个大小为`p->bufsize`加上`p->offset`的内存缓冲区，如果分配失败，则使用`errno`和错误信息格式化。
3. 将分配到的内存缓冲区存储在`p->buffer`中。
4. 返回分配到的内存缓冲区。


```cpp
#endif /* HAVE_SYS_BUFMOD_H */

/*
 * Allocate data buffer. Returns -1 if memory allocation fails, else 0.
 */
int
pcap_alloc_databuf(pcap_t *p)
{
	p->bufsize = PKTBUFSIZE;
	p->buffer = malloc(p->bufsize + p->offset);
	if (p->buffer == NULL) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "malloc");
		return (-1);
	}

	return (0);
}

```

这段代码是一个用于操作系统输出的函数，名为`strioctl`。它的作用是接受一个文件描述符（fd）和一个字符串（dp），通过调用`ioctl`函数并取得返回值，输出或覆盖文件的内容。

函数的参数包括：

* `fd`：文件描述符，表示要打开或关闭的文件。
* `cmd`：要执行的IOCMTL命令。这个参数是用来调用`ioctl`函数的，需要手动设置。
* `len`：字符串缓冲区的长度，这个参数用于指定要输出或覆盖的字符数。
* `dp`：用于存储输出或覆盖的字符串。这个参数在函数中用于和`ioctl`函数的返回值进行匹配。

函数中首先定义了一个`strioctl`结构体，包含了输入的命令、 timeout、长度和字符串缓冲区。然后调用`ioctl`函数，并将得到的结果存储到`str`结构体中，最后将`str.ic_len`作为函数的返回值。如果函数在执行过程中遇到错误，将会返回-1，否则返回字符串的长度。


```cpp
/*
 * Issue a STREAMS I_STR ioctl. Returns -1 on error, otherwise
 * length of returned data on success.
 */
int
strioctl(int fd, int cmd, int len, char *dp)
{
	struct strioctl str;
	int retv;

	str.ic_cmd = cmd;
	str.ic_timout = -1;
	str.ic_len = len;
	str.ic_dp = dp;
	if ((retv = ioctl(fd, I_STR, &str)) < 0)
		return (retv);

	return (str.ic_len);
}

```

这段代码是一个名为 `pcap_stream_err` 的函数，它接受三个参数：

1. 函数名 `pcap_stream_err`：函数的名称，用于标识函数的职责。
2. 函数参数：
 - `const char *func`：指向要打印错误信息的函数的指针。这是一个 `const` 类型的参数，意味着这个指针不能被修改。
 - `int err`：表示函数执行时遇到的错误代码，它可以从 `PCAP_ERRBUF_SIZE` （没有实际意义，可能是输入错行）这个常量中获取。
 - `char *errbuf`：用于存储错误信息的字符数组。它是该函数最后一个参数，用于将错误信息存储到数组中。
3. 函数实现：

该函数的作用是打印错误信息到已经定义好的 `errbuf` 数组中。具体的实现是，先通过调用 `pcap_fmt_errmsg_for_errno` 函数，该函数将错误信息格式化并存储到一个 `PCAP_ERRBUF_SIZE` 指向的内存区域。然后，将这个错误信息通过 `strncpy` 函数复制到 `errbuf` 数组的起始位置，余下错误信息也被复制到了数组的后续位置。

该函数是在 `pcap_init` 函数中定义的，通常用于在 `pcap_课外活动` 函数中调用，用于在 `pcap_课外活动` 函数中处理错误信息。


```cpp
#ifdef HAVE_SYS_BUFMOD_H
/*
 * Write stream error message to errbuf.
 */
static void
pcap_stream_err(const char *func, int err, char *errbuf)
{
	pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE, err, "%s", func);
}
#endif

```

# `libpcap/etherent.c`

这段代码是一个C语言的函数声明，它定义了一个名为`main`的函数。这个函数没有注释，但是根据函数声明的内容，我们可以猜测它可能用于操作系统或者计算机编程相关的事情。我们需要了解具体的上下文来进一步分析。


```cpp
/*
 * Copyright (c) 1990, 1993, 1994, 1995, 1996
 *	The Regents of the University of California.  All rights reserved.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that: (1) source code distributions
 * retain the above copyright notice and this paragraph in its entirety, (2)
 * distributions including binary code include the above copyright notice and
 * this paragraph in its entirety in the documentation or other materials
 * provided with the distribution, and (3) all advertising materials mentioning
 * features or use of this software display the following acknowledgement:
 * ``This product includes software developed by the University of California,
 * Lawrence Berkeley Laboratory and its contributors.'' Neither the name of
 * the University nor the names of its contributors may be used to endorse
 * or promote products derived from this software without specific prior
 * written permission.
 * THIS SOFTWARE IS PROVIDED ``AS IS'' AND WITHOUT ANY EXPRESS OR IMPLIED
 * WARRANTIES, INCLUDING, WITHOUT LIMITATION, THE IMPLIED WARRANTIES OF
 * MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE.
 */

```

这段代码是一个C预处理指令，它包含了一个特定的头文件，并在其中定义了一些常量和函数。它的主要作用是在编译之前检查系统是否支持某个特定的头文件，以及包含的函数和变量。

具体来说，它包含了一个条件判断语句，它会检查是否支持名为"config.h"的头文件。如果不支持，它会包含一个包含所有需要定义的函数和变量的列表，以便在编译时进行替换。如果支持，它将忽略定义的函数和变量，不会在编译时产生任何警告。

此外，它还包含了一个头文件包含语句，它会检查是否支持名为"os-proto.h"的头文件。如果支持，它将包含定义的函数和变量，否则将忽略它们，不会在编译时产生任何警告。

最后，它定义了一些函数，包括内存管理函数和字符串操作函数，这些函数可能需要在应用程序中使用，但它们的定义和实现可能是在头文件中定义的。


```cpp
#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <pcap-types.h>

#include <memory.h>
#include <stdio.h>
#include <string.h>

#include "pcap-int.h"

#include <pcap/namedb.h>

#ifdef HAVE_OS_PROTO_H
```

这段代码是一个 C 语言编译器（或者称为预处理器）的一部分，主要作用是引入了一些函数和定义，用于处理源文件中的语法错误。

具体来说，以下代码包括以下几个部分：

1. 引入了两个名为 "os-proto.h" 的头文件。这些头文件可能包含了与操作系统相关的接口函数，用于定义与操作系统交互的一些基本操作。

2. 定义了两个名为 "skip_space" 和 "skip_line" 的函数。这些函数的具体实现可能是在另一个头文件中定义的，我们无法获取这个信息。

3. 在函数体外定义了一个名为 "xdtoi" 的函数。这个函数接收一个字符串（用 u_char 表示），然后将其转换为相应的整数类型。

4. 在 "skip_space" 和 "skip_line" 函数体内部，使用了 "FILE *" 类型的指针变量，但具体使用了哪个指针变量在当前的源代码中没有给出。

5. 在 "skip_line" 函数体内部，使用了 "int skip_space(FILE *)" 函数。这个函数的具体实现可能是在 "skip_space" 函数中调用的，但我们无法获取这个信息。

6. 在 "xdtoi" 函数体内部，使用了 "u_char" 和 "u_int" 类型的变量。这里我们无法确定 u_int 是整数类型还是指针类型，但我们知道 "xdtoi" 函数需要一个 u_char 类型的输入。

综上所述，这段代码的作用是定义了一些函数和头文件，可能用于操作系统编程。


```cpp
#include "os-proto.h"
#endif

static inline int skip_space(FILE *);
static inline int skip_line(FILE *);

/* Hex digit to integer. */
static inline u_char
xdtoi(u_char c)
{
	if (c >= '0' && c <= '9')
		return (u_char)(c - '0');
	else if (c >= 'a' && c <= 'f')
		return (u_char)(c - 'a' + 10);
	else
		return (u_char)(c - 'A' + 10);
}

```

这段代码定义了一个名为`skip_space`的函数，它的功能是跳过空格和换行符，直到遇到非空格字符或换行符为止。

该函数接受一个文件指针变量`f`作为参数。函数内部通过一个`do-while`循环来读取文件中的每一个字符，并将其存储在变量`c`中。循环条件为`c`不能为空格、制表符或回车符。

当读取到换行符时，函数会返回`c`，并继续循环。当读取到空格或回车符时，函数会跳过该字符，并继续循环。这样，函数可以确保在遇到换行符之前，已经读取到了尽可能多的字符。

该函数的实现还可以进一步优化，比如在函数内部使用`fgets`函数来方便地读取文件内容，而不是通过`getc`函数逐个读取文件内容。


```cpp
/*
 * Skip linear white space (space and tab) and any CRs before LF.
 * Stop when we hit a non-white-space character or an end-of-line LF.
 */
static inline int
skip_space(FILE *f)
{
	int c;

	do {
		c = getc(f);
	} while (c == ' ' || c == '\t' || c == '\r');

	return c;
}

```

This looks like a function that parses the contents of a file and returns a pointer to an error object. Here is a breakdown of how it works:

1. The function takes a file pointer (fp) and a buffer (e.g. a string) to read from.
2. It reads the contents of the file into the buffer using the read() system call.
3. It skips over any blank lines by using the skip_line() and skip_space() functions.
4. It uses a loop to read the contents of the file, storing the contents in the buffer.
5. It uses namesize to check if the buffer is large enough to hold the name of the file. If it's not, it skips over the blank lines and returns.
6. It uses a while loop to continue reading the file until it reaches the end of the file (e.g. a newline).
7. When it reaches the end of the file, it skips over any blank lines and continues reading.
8. It also reads the name of the file from the buffer and stores it in the e.addr field.
9. If it doesn't reach the end of the file, it returns the error object.

This function appears to be a general-purpose tool for parsing the contents of a file and returning the error code if there is anything that doesn't match the expected format.


```cpp
static inline int
skip_line(FILE *f)
{
	int c;

	do
		c = getc(f);
	while (c != '\n' && c != EOF);

	return c;
}

struct pcap_etherent *
pcap_next_etherent(FILE *fp)
{
	register int c, i;
	u_char d;
	char *bp;
	size_t namesize;
	static struct pcap_etherent e;

	memset((char *)&e, 0, sizeof(e));
	for (;;) {
		/* Find addr */
		c = skip_space(fp);
		if (c == EOF)
			return (NULL);
		if (c == '\n')
			continue;

		/* If this is a comment, or first thing on line
		   cannot be Ethernet address, skip the line. */
		if (!PCAP_ISXDIGIT(c)) {
			c = skip_line(fp);
			if (c == EOF)
				return (NULL);
			continue;
		}

		/* must be the start of an address */
		for (i = 0; i < 6; i += 1) {
			d = xdtoi((u_char)c);
			c = getc(fp);
			if (c == EOF)
				return (NULL);
			if (PCAP_ISXDIGIT(c)) {
				d <<= 4;
				d |= xdtoi((u_char)c);
				c = getc(fp);
				if (c == EOF)
					return (NULL);
			}
			e.addr[i] = d;
			if (c != ':')
				break;
			c = getc(fp);
			if (c == EOF)
				return (NULL);
		}

		/* Must be whitespace */
		if (c != ' ' && c != '\t' && c != '\r' && c != '\n') {
			c = skip_line(fp);
			if (c == EOF)
				return (NULL);
			continue;
		}
		c = skip_space(fp);
		if (c == EOF)
			return (NULL);

		/* hit end of line... */
		if (c == '\n')
			continue;

		if (c == '#') {
			c = skip_line(fp);
			if (c == EOF)
				return (NULL);
			continue;
		}

		/* pick up name */
		bp = e.name;
		/* Use 'namesize' to prevent buffer overflow. */
		namesize = sizeof(e.name) - 1;
		do {
			*bp++ = (u_char)c;
			c = getc(fp);
			if (c == EOF)
				return (NULL);
		} while (c != ' ' && c != '\t' && c != '\r' && c != '\n'
		    && --namesize != 0);
		*bp = '\0';

		/* Eat trailing junk */
		if (c != '\n')
			(void)skip_line(fp);

		return &e;
	}
}

```