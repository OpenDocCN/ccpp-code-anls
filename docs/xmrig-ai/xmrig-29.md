# xmrig源码解析 29

# `src/3rdparty/hwloc/src/base64.c`

这段代码是一个C语言代码片段，它定义了一个名为"private/private.h"的外部头文件。通过包含这个头文件，程序可以访问定义在这个头文件下的函数和变量。

进一步解释：

* `/*` 和 `*/` 是C语言的注释，用于告诉编译器代码的边界。
* `#include "private/private.h"` 是用来引入头文件的指令，告诉编译器在编译之前需要 include 一下 "private/private.h" 这个头文件。
* `#include "private/private.h"` 是传递给程序的链接参数，告诉它要在编译时链接 "private/private.h" 这个头文件。
* `private/private.h` 是头文件名，它告诉程序这个文件包含了哪些函数和变量。
* `/* copyright */` 是告诉编译器这个代码块是受版权保护的。
* `*/` 是结束注释。
* `/* modifies after import */` 是告诉编译器这个代码块在导入之后。
* `*/` 是结束注释。
* `#include "private/private.h"` 告诉编译器在编译之前需要 include 一下 "private/private.h" 这个头文件。


```cpp
/*
 * Copyright © 2012-2018 Inria.  All rights reserved.
 * See COPYING in top-level directory.
 *
 * Modifications after import:
 * - removed all #if
 * - updated prototypes
 * - updated #include
 */

/* include hwloc's config before anything else
 * so that extensions and features are properly enabled
 */
#include "private/private.h"

```

这段代码是一个C语言的函数，名为"base64.c"。它主要用于将Base64编码的字符串转换为字符串，并返回原始字符串。

在这段注释中，开发者说明了这段代码的版权信息、授权信息和限制。他们允许在遵守上述版权通知和限制的前提下，自由地使用、复制、修改和分发这个软件。

这段代码的具体实现可能有一些不同，但它的作用是相同的：提供一个将Base64编码的字符串转换为原始字符串的方法，以便开发者或其他用户可以使用。


```cpp
/*	$OpenBSD: base64.c,v 1.5 2006/10/21 09:55:03 otto Exp $	*/

/*
 * Copyright (c) 1996 by Internet Software Consortium.
 *
 * Permission to use, copy, modify, and distribute this software for any
 * purpose with or without fee is hereby granted, provided that the above
 * copyright notice and this permission notice appear in all copies.
 *
 * THE SOFTWARE IS PROVIDED "AS IS" AND INTERNET SOFTWARE CONSORTIUM DISCLAIMS
 * ALL WARRANTIES WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES
 * OF MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL INTERNET SOFTWARE
 * CONSORTIUM BE LIABLE FOR ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL
 * DAMAGES OR ANY DAMAGES WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR
 * PROFITS, WHETHER IN AN ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS
 * ACTION, ARISING OUT OF OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS
 * SOFTWARE.
 */

```

这段代码是一个C语言的函数声明，它定义了一个名为`gethostdynamic`的函数。该函数可能会被用于在TCP/IP网络上动态地更新域名系统（DNS）记录。通过引入IBM的版权声明，该代码允许在使用、复制、修改和分发该软件时，无需支付费用，前提是在所有副本中包含上述版权通知以及所有相关说明。

函数声明中包含以下条款：

1. 该软件在IBM的版权下提供，可以在其许可下免费使用、复制、修改和分发。

2. IBM授予在使用、销售或制造包含该软件的产品时，对其提供免疫，前提是在此类产品使用DNS动态更新时执行该软件。

3. 在IBM的版权许可下，该软件可以用于执行DNS动态更新，从而在TCP/IP网络上动态地更新域名系统记录。

4. 该软件是“按现状”提供，IBM不保证其品质，也不会对使用该软件产生的任何损害负责。

5. 在适用情况下，IBM为其在DNS动态更新中所做的事情提供免疫，使其免受诉讼。但该免疫并非普遍适用，具体情况需根据IBM的版权规定进行评估。


```cpp
/*
 * Portions Copyright (c) 1995 by International Business Machines, Inc.
 *
 * International Business Machines, Inc. (hereinafter called IBM) grants
 * permission under its copyrights to use, copy, modify, and distribute this
 * Software with or without fee, provided that the above copyright notice and
 * all paragraphs of this notice appear in all copies, and that the name of IBM
 * not be used in connection with the marketing of any product incorporating
 * the Software or modifications thereof, without specific, written prior
 * permission.
 *
 * To the extent it has a right to do so, IBM grants an immunity from suit
 * under its patents, if any, for the use, sale or manufacture of products to
 * the extent that such products are used for performing Domain Name System
 * dynamic updates in TCP/IP networks by means of the Software.  No immunity is
 * granted for any product per se or for any other function of any product.
 *
 * THE SOFTWARE IS PROVIDED "AS IS", AND IBM DISCLAIMS ALL WARRANTIES,
 * INCLUDING ALL IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A
 * PARTICULAR PURPOSE.  IN NO EVENT SHALL IBM BE LIABLE FOR ANY SPECIAL,
 * DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES WHATSOEVER ARISING
 * OUT OF OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE, EVEN
 * IF IBM IS APPRISED OF THE POSSIBILITY OF SUCH DAMAGES.
 */

```

This is a description of a program called " Base64 Encode" that encodes binary data as a string of printable ASCII characters. The program uses a fixed-length encoding scheme where each 6-bit group of characters is used to represent a single byte of data.

The program takes a single argument, which is the binary data to be encoded. The program first checks if the input data is shorter than 24 bits and, if it is, the program performs a simple padding operation by adding zero bits to form an integral number of 6-bit groups. If the input data is longer than 24 bits, the program performs a full encoding quantum and pads the output with the "=" character.

The program then outputs the encoded data as a string of printable ASCII characters. If the input data is shorter than 8 bits, the program outputs a single character. If the input data is longer than 8 bits but less than 16 bits, the program outputs up to three characters. If the input data is longer than 16 bits, the program outputs up to four characters.

The Base64 Encode program is a useful tool for converting binary data to a printable ASCII string. It is often used in web development applications where binary data needs to be transmitted as a text format.


```cpp
/* OPENBSD ORIGINAL: lib/libc/net/base64.c */

static const char Base64[] =
	"ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";
static const char Pad64 = '=';

/* (From RFC1521 and draft-ietf-dnssec-secext-03.txt)
   The following encoding technique is taken from RFC 1521 by Borenstein
   and Freed.  It is reproduced here in a slightly edited form for
   convenience.

   A 65-character subset of US-ASCII is used, enabling 6 bits to be
   represented per printable character. (The extra 65th character, "=",
   is used to signify a special processing function.)

   The encoding process represents 24-bit groups of input bits as output
   strings of 4 encoded characters. Proceeding from left to right, a
   24-bit input group is formed by concatenating 3 8-bit input groups.
   These 24 bits are then treated as 4 concatenated 6-bit groups, each
   of which is translated into a single digit in the base64 alphabet.

   Each 6-bit group is used as an index into an array of 64 printable
   characters. The character referenced by the index is placed in the
   output string.

                         Table 1: The Base64 Alphabet

      Value Encoding  Value Encoding  Value Encoding  Value Encoding
          0 A            17 R            34 i            51 z
          1 B            18 S            35 j            52 0
          2 C            19 T            36 k            53 1
          3 D            20 U            37 l            54 2
          4 E            21 V            38 m            55 3
          5 F            22 W            39 n            56 4
          6 G            23 X            40 o            57 5
          7 H            24 Y            41 p            58 6
          8 I            25 Z            42 q            59 7
          9 J            26 a            43 r            60 8
         10 K            27 b            44 s            61 9
         11 L            28 c            45 t            62 +
         12 M            29 d            46 u            63 /
         13 N            30 e            47 v
         14 O            31 f            48 w         (pad) =
         15 P            32 g            49 x
         16 Q            33 h            50 y

   Special processing is performed if fewer than 24 bits are available
   at the end of the data being encoded.  A full encoding quantum is
   always completed at the end of a quantity.  When fewer than 24 input
   bits are available in an input group, zero bits are added (on the
   right) to form an integral number of 6-bit groups.  Padding at the
   end of the data is performed using the '=' character.

   Since all base64 input is an integral number of octets, only the
         -------------------------------------------------
   following cases can arise:

       (1) the final quantum of encoding input is an integral
           multiple of 24 bits; here, the final unit of encoded
	   output will be an integral multiple of 4 characters
	   with no "=" padding,
       (2) the final quantum of encoding input is exactly 8 bits;
           here, the final unit of encoded output will be two
	   characters followed by two "=" padding characters, or
       (3) the final quantum of encoding input is exactly 16 bits;
           here, the final unit of encoded output will be three
	   characters followed by one "=" padding character.
   */

```

This looks like a Python implementation of the Base64 decoder. The function takes in a buffer `input` and outputs a coded string `output`. The buffer is padded with padding characters at the end to make it larger, and this function handles the padding properly.

The implementation is recursive and works by first getting the length of the input buffer, then initializing the output buffer with the first four bytes of the input. The output buffer is then initialized with the first byte of the input, followed by the byte that's half-way across the buffer.

If the input buffer is larger than the output buffer, the function returns -1. Otherwise, the function continues by getting the remaining input buffer and padding it with Base64 encoding. This is repeated until the entire input buffer is processed.

The function uses a combination of在校验 and padding to ensure that the output is valid. The output is checked for errors, and if any errors are found the function returns -1.


```cpp
#include <stdlib.h>
#include <string.h>
#include <ctype.h>

int
hwloc_encode_to_base64(const char *src, size_t srclength, char *target, size_t targsize)
{
	size_t datalength = 0;
	unsigned char input[3];
	unsigned char output[4];
	unsigned int i;

	while (2 < srclength) {
		input[0] = *src++;
		input[1] = *src++;
		input[2] = *src++;
		srclength -= 3;

		output[0] = input[0] >> 2;
		output[1] = ((input[0] & 0x03) << 4) + (input[1] >> 4);
		output[2] = ((input[1] & 0x0f) << 2) + (input[2] >> 6);
		output[3] = input[2] & 0x3f;

		if (datalength + 4 > targsize)
			return (-1);
		target[datalength++] = Base64[output[0]];
		target[datalength++] = Base64[output[1]];
		target[datalength++] = Base64[output[2]];
		target[datalength++] = Base64[output[3]];
	}

	/* Now we worry about padding. */
	if (0 != srclength) {
		/* Get what's left. */
		input[0] = input[1] = input[2] = '\0';
		for (i = 0; i < srclength; i++)
			input[i] = *src++;

		output[0] = input[0] >> 2;
		output[1] = ((input[0] & 0x03) << 4) + (input[1] >> 4);
		output[2] = ((input[1] & 0x0f) << 2) + (input[2] >> 6);

		if (datalength + 4 > targsize)
			return (-1);
		target[datalength++] = Base64[output[0]];
		target[datalength++] = Base64[output[1]];
		if (srclength == 1)
			target[datalength++] = Pad64;
		else
			target[datalength++] = Base64[output[2]];
		target[datalength++] = Pad64;
	}
	if (datalength >= targsize)
		return (-1);
	target[datalength] = '\0';	/* Returned value doesn't count \0. */
	return (int)(datalength);
}

```

This is a function that enforces the boundary conditions for a pad character string. The function takes a source string and a target string as input and returns the index of the first non-whitespace character in the target string.

The function first checks the source string and the target string against a set of boundary conditions. If the source string or the target string is not a valid pad character string, the function returns -1.

If the source string is a valid pad character string, the function checks for the presence of a single trailing equal sign (`=`) at the end of the string. If the string is not a valid pad character string, the function returns -1.

If the source string is a valid pad character string and the end of the string is a valid equal sign, the function checks for the presence of any secondary whitespace characters that may be present in the source string. If the function does not find any such characters, the function returns the target index.

If the source string is a valid pad character string and the end of the string is not a valid equal sign, the function checks for the presence of a Pad64 marker at the end of the string. If the function does not find any such markers, the function returns -1.

If the source string is a valid pad character string and the end of the string is not a valid equal sign or a Pad64 marker, the function skips over any number of whitespace characters in the source string and returns the target index.

The function also includes a check for the case where the input string is a valid pad character string but has no valid padding at the end. In this case, the function returns -1.


```cpp
/* skips all whitespace anywhere.
   converts characters, four at a time, starting at (or after)
   src from base - 64 numbers into three 8 bit bytes in the target area.
   it returns the number of data bytes stored at the target, or -1 on error.
 */

int
hwloc_decode_from_base64(char const *src, char *target, size_t targsize)
{
	unsigned int tarindex, state;
	int ch;
	char *pos;

	state = 0;
	tarindex = 0;

	while ((ch = *src++) != '\0') {
		if (isspace(ch))	/* Skip whitespace anywhere. */
			continue;

		if (ch == Pad64)
			break;

		pos = strchr(Base64, ch);
		if (pos == 0) 		/* A non-base64 character. */
			return (-1);

		switch (state) {
		case 0:
			if (target) {
				if (tarindex >= targsize)
					return (-1);
				target[tarindex] = (char)(pos - Base64) << 2;
			}
			state = 1;
			break;
		case 1:
			if (target) {
				if (tarindex + 1 >= targsize)
					return (-1);
				target[tarindex]   |=  (pos - Base64) >> 4;
				target[tarindex+1]  = ((pos - Base64) & 0x0f)
							<< 4 ;
			}
			tarindex++;
			state = 2;
			break;
		case 2:
			if (target) {
				if (tarindex + 1 >= targsize)
					return (-1);
				target[tarindex]   |=  (pos - Base64) >> 2;
				target[tarindex+1]  = ((pos - Base64) & 0x03)
							<< 6;
			}
			tarindex++;
			state = 3;
			break;
		case 3:
			if (target) {
				if (tarindex >= targsize)
					return (-1);
				target[tarindex] |= (pos - Base64);
			}
			tarindex++;
			state = 0;
			break;
		}
	}

	/*
	 * We are done decoding Base-64 chars.  Let's see if we ended
	 * on a byte boundary, and/or with erroneous trailing characters.
	 */

	if (ch == Pad64) {		/* We got a pad char. */
		ch = *src++;		/* Skip it, get next. */
		switch (state) {
		case 0:		/* Invalid = in first position */
		case 1:		/* Invalid = in second position */
			return (-1);

		case 2:		/* Valid, means one byte of info */
			/* Skip any number of spaces. */
			for (; ch != '\0'; ch = *src++)
				if (!isspace(ch))
					break;
			/* Make sure there is another trailing = sign. */
			if (ch != Pad64)
				return (-1);
			ch = *src++;		/* Skip the = */
			/* Fall through to "single trailing =" case. */
			/* FALLTHROUGH */

		case 3:		/* Valid, means two bytes of info */
			/*
			 * We know this char is an =.  Is there anything but
			 * whitespace after it?
			 */
			for (; ch != '\0'; ch = *src++)
				if (!isspace(ch))
					return (-1);

			/*
			 * Now make sure for cases 2 and 3 that the "extra"
			 * bits that slopped past the last full byte were
			 * zeros.  If we don't check them, they become a
			 * subliminal channel.
			 */
			if (target && target[tarindex] != 0)
				return (-1);
		}
	} else {
		/*
		 * We ended by seeing the end of the string.  Make sure we
		 * have no partial bytes lying around.
		 */
		if (state != 0)
			return (-1);
	}

	return (tarindex);
}

```

# `src/3rdparty/hwloc/src/bind.c`

这段代码是一个通用的头文件，它定义了一些通用的函数和变量。

*在版权声明中提到了对以下源代码的版权：
```cpp
Copyright © 2009 CNRS
Copyright © 2009-2020 Inria.  All rights reserved.
Copyright © 2009-2010, 2012 Université Bordeaux
Copyright © 2011-2015 Cisco Systems, Inc.  All rights reserved.
See COPYING in top-level directory.
```
表明了这些库的版权所有人。

*`#include "private/autogen/config.h"`和`#include "hwloc.h"`以及`#include "private/private.h"`和`#include "hwloc/helper.h"`是一个导入导入了的外部头文件，它们定义了`hwloc`库中与内存管理，网络配置和定位相关的一些函数和结构体。
*`#ifdef HAVE_SYS_MMAN_H`是一个条件编译语句，如果`HAVE_SYS_MMAN_H`是`#define`或者是一个`#ifdef`或者`#ifndef`，那么它将编译成`<sys/mman.h>`。
*`sys/mman.h`是一个系统调用接口，它提供了一组用于管理内存的函数。通过使用`sys/mman.h`可以访问操作系统提供的内存映射，并可以实现自己的内存池等功能。


```cpp
/*
 * Copyright © 2009 CNRS
 * Copyright © 2009-2020 Inria.  All rights reserved.
 * Copyright © 2009-2010, 2012 Université Bordeaux
 * Copyright © 2011-2015 Cisco Systems, Inc.  All rights reserved.
 * See COPYING in top-level directory.
 */

#include "private/autogen/config.h"
#include "hwloc.h"
#include "private/private.h"
#include "hwloc/helper.h"

#ifdef HAVE_SYS_MMAN_H
#  include <sys/mman.h>
```

这段代码是一个 conditional compilation header，它包含了一系列条件判断，并根据这些判断结果决定是否包含某些头文件和函数。

具体来说，这段代码的作用如下：

1. 如果已经定义了 `hwloc_getpagesize()` 函数，并且没有定义 `HAVE_POSIX_MEMALIGN` 函数，也没有定义 `HAVE_MEMALIGN` 函数，同时定义了 `HAVE_MALLOC_H` 函数，那么它将包含 `<malloc.h>` 头文件。

2. 如果定义了 `HAVE_UNISTD_H` 函数，那么它将包含该函数。

3. 如果定义了 `posix_memalign()` 函数，并且没有定义 `HAVE_POSIX_MEMALIGN` 函数，那么它将包含 `<malloc.h>` 头文件。

4. 如果定义了 `memalign()` 函数，并且定义了 `HAVE_MEMALIGN` 函数，那么它将包含该函数。

5. 如果定义了 `malloc_盲重载()` 函数，那么它将包含该函数。

6. 如果定义了 `hwloc_sched_affine()` 函数，那么它将包含该函数。

由于该代码使用了许多来自头文件和函数的 `#ifdef` 和 `#include` 指令，因此它的可读性非常高，同时它也不会在保存路径中包含任何库文件或库路径，因此也不会对代码可读性造成影响。


```cpp
#endif
/* <malloc.h> is only needed if we don't have posix_memalign() */
#if defined(hwloc_getpagesize) && !defined(HAVE_POSIX_MEMALIGN) && defined(HAVE_MEMALIGN) && defined(HAVE_MALLOC_H)
#include <malloc.h>
#endif
#ifdef HAVE_UNISTD_H
#include <unistd.h>
#endif
#include <stdlib.h>
#include <errno.h>

/* TODO: HWLOC_GNU_SYS,
 *
 * We could use glibc's sched_setaffinity generically when it is available
 *
 * Darwin and OpenBSD don't seem to have binding facilities.
 */

```

这段代码定义了一个名为 `hwloc_fix_cpubind` 的函数，它接受一个 `hwloc_topology_t` 的 topology 参数和一个 `hwloc_const_bitmap_t` 的 set 参数。

这个函数的作用是在给定的 topology 架构中查找与给定 set 最为相关的可用 CPU 绑定（CPU绑定、线程绑定、内存绑定）设置，然后返回该设置。

具体来说，函数首先通过 `hwloc_topology_get_topology_cpuset` 函数获取 topology 架构的 CPU 绑定集合，接着通过 `hwloc_topology_get_complete_cpuset` 函数获取与给定 topology 架构完全相关的 CPU 绑定集合。

如果给定的 set 已经包含了完整的 topology 架构，函数将返回设置。否则，函数将检查给定的 set 是否与完整的 topology 架构中的某个 CPU 绑定相关。如果是，函数将更新设置为完整的 topology 架构。最后，函数返回设置。


```cpp
#define HWLOC_CPUBIND_ALLFLAGS (HWLOC_CPUBIND_PROCESS|HWLOC_CPUBIND_THREAD|HWLOC_CPUBIND_STRICT|HWLOC_CPUBIND_NOMEMBIND)

static hwloc_const_bitmap_t
hwloc_fix_cpubind(hwloc_topology_t topology, hwloc_const_bitmap_t set)
{
  hwloc_const_bitmap_t topology_set = hwloc_topology_get_topology_cpuset(topology);
  hwloc_const_bitmap_t complete_set = hwloc_topology_get_complete_cpuset(topology);

  if (hwloc_bitmap_iszero(set)) {
    errno = EINVAL;
    return NULL;
  }

  if (!hwloc_bitmap_isincluded(set, complete_set)) {
    errno = EINVAL;
    return NULL;
  }

  if (hwloc_bitmap_isincluded(topology_set, set))
    set = complete_set;

  return set;
}

```

这段代码定义了一个名为 `hwloc_set_cpubind` 的函数，属于 `hwloc_topology_t` 结构体的成员函数。它的作用是设置 CPU  binding。

函数接受三个参数：

- `topology`：要设置 CPU 绑定的 topology 结构体。
- `set`：已知的 CPU  binding 设置。
- `flags`：设置 CPU 绑定的标志，包括 `HWLOC_CPUBIND_ALLFLAGS`。

函数首先检查 flags，如果它不包括 `HWLOC_CPUBIND_ALLFLAGS`，那么就返回 `EINVAL` 和 `-1`。

接下来，如果 flags 中包括 `HWLOC_CPUBIND_PROCESS`，那么尝试调用 `hwloc_fix_cpubind` 来修复设置。如果 `hwloc_fix_cpubind` 成功，那么就返回它。如果 `hwloc_fix_cpubind` 失败，那么就返回 `-1`。

如果 flags 中包括 `HWLOC_CPUBIND_THREAD`，那么尝试调用 `hwloc_bind_this_process_cpubind` 或 `hwloc_bind_this_thread_cpubind` 来设置 CPU binding。如果 `hwloc_bind_this_process_cpubind` 或 `hwloc_bind_this_thread_cpubind` 成功，那么就返回它。如果 `hwloc_bind_this_process_cpubind` 或 `hwloc_bind_this_thread_cpubind` 失败，那么就返回 `-1`。

如果 flags 中不包括 `HWLOC_CPUBIND_ALLFLAGS`，那么调用 `hwloc_setup_cpulist` 来设置 CPU binding。如果 `hwloc_setup_cpulist` 成功，那么就返回 `0`。如果 `hwloc_setup_cpulist` 失败，那么就返回 `-1`。

如果任何除 `EINVAL` 和 `-1` 外的错误发生，那么将错误码作为返回值。


```cpp
int
hwloc_set_cpubind(hwloc_topology_t topology, hwloc_const_bitmap_t set, int flags)
{
  if (flags & ~HWLOC_CPUBIND_ALLFLAGS) {
    errno = EINVAL;
    return -1;
  }

  set = hwloc_fix_cpubind(topology, set);
  if (!set)
    return -1;

  if (flags & HWLOC_CPUBIND_PROCESS) {
    if (topology->binding_hooks.set_thisproc_cpubind)
      return topology->binding_hooks.set_thisproc_cpubind(topology, set, flags);
  } else if (flags & HWLOC_CPUBIND_THREAD) {
    if (topology->binding_hooks.set_thisthread_cpubind)
      return topology->binding_hooks.set_thisthread_cpubind(topology, set, flags);
  } else {
    if (topology->binding_hooks.set_thisproc_cpubind) {
      int err = topology->binding_hooks.set_thisproc_cpubind(topology, set, flags);
      if (err >= 0 || errno != ENOSYS)
        return err;
      /* ENOSYS, fallback */
    }
    if (topology->binding_hooks.set_thisthread_cpubind)
      return topology->binding_hooks.set_thisthread_cpubind(topology, set, flags);
  }

  errno = ENOSYS;
  return -1;
}

```

这段代码定义了一个名为 `hwloc_get_cpubind` 的函数，属于 `hwloc_topology_t` 结构体中的一个成员函数。它的作用是获取绑定到指定硬件设备的 CPU 内存映射，并返回相应的结果。

具体来说，函数接受三个参数：`topology` 是表示硬件设备拓扑结构的一个 `hwloc_topology_t` 类型的变量，`set` 是表示要查询的 CPU 内存映射集合的一个 `hwloc_bitmap_t` 类型的变量，`flags` 是用于设置查询方式的二进制位，其中 `HWLOC_CPUBIND_ALLFLAGS` 表示查询所有与指定硬件设备相关的 CPU 内存映射。

函数首先检查 `flags` 是否包含 `HWLOC_CPUBIND_ALLFLAGS`，如果是，则函数返回一个错误码并销毁。然后，函数依次检查 `flags` 是否包含 `HWLOC_CPUBIND_PROCESS` 和 `HWLOC_CPUBIND_THREAD`，如果是，则函数分别尝试通过 `topology->binding_hooks.get_thisproc_cpubind` 和 `topology->binding_hooks.get_thisthread_cpubind` 查询指定进程或线程的 CPU 内存映射，如果函数成功返回它们的值，否则返回一个错误码。最后，如果 `flags` 中任何一个是 `HWLOC_CPUBIND_ALLFLAGS`，函数将抛出错误码并返回一个错误信息。


```cpp
int
hwloc_get_cpubind(hwloc_topology_t topology, hwloc_bitmap_t set, int flags)
{
  if (flags & ~HWLOC_CPUBIND_ALLFLAGS) {
    errno = EINVAL;
    return -1;
  }

  if (flags & HWLOC_CPUBIND_PROCESS) {
    if (topology->binding_hooks.get_thisproc_cpubind)
      return topology->binding_hooks.get_thisproc_cpubind(topology, set, flags);
  } else if (flags & HWLOC_CPUBIND_THREAD) {
    if (topology->binding_hooks.get_thisthread_cpubind)
      return topology->binding_hooks.get_thisthread_cpubind(topology, set, flags);
  } else {
    if (topology->binding_hooks.get_thisproc_cpubind) {
      int err = topology->binding_hooks.get_thisproc_cpubind(topology, set, flags);
      if (err >= 0 || errno != ENOSYS)
        return err;
      /* ENOSYS, fallback */
    }
    if (topology->binding_hooks.get_thisthread_cpubind)
      return topology->binding_hooks.get_thisthread_cpubind(topology, set, flags);
  }

  errno = ENOSYS;
  return -1;
}

```

这段代码定义了一个名为 `hwloc_set_proc_cpubind` 的函数，属于 `hwloc_base_server` 库。它的作用是设置一个进程的 `CPU` 绑定，以便将某些与 `CPU` 绑定相关的数据传递给绑定到 `hwloc_topology_t` 结构中的其他函数。

函数接受四个参数：

1. `topology`：要设置的 `hwloc_topology_t` 结构体，用于指定要绑定的目标。
2. `pid`：要设置的 `hwloc_pid_t` 结构体，用于指定要绑定的进程 ID。
3. `set`：一个 `hwloc_const_bitmap_t` 结构体，用于指定要设置的 CPU 绑定。在这里，我们通过 `hwloc_fix_cpubind` 函数将 `set` 转换为正确的 CPU 绑定，如果转换失败，就返回一个错误码。
4. `flags`：一个 `int` 结构体，用于指定其他设置的标志，如 `HWLOC_CPUBIND_ALLFLAGS`，如果没有这个标志，就表示 `set` 中的所有设置都是错误的，函数的行为将类似于没有设置任何 CPU 绑定。

函数首先检查传递给它的 `flags`，如果 `flags` 中不包括 `HWLOC_CPUBIND_ALLFLAGS`，就表示 `set` 中的所有设置都是错误的，函数的行为将类似于没有设置任何 CPU 绑定。在这种情况下，函数返回一个错误码，否则就执行 `hwloc_fix_cpubind` 函数将 `set` 转换为正确的 CPU 绑定。

如果 `topology->binding_hooks.set_proc_cpubind` 函数仍然无法正确设置 CPU 绑定，函数将返回一个错误码。


```cpp
int
hwloc_set_proc_cpubind(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_const_bitmap_t set, int flags)
{
  if (flags & ~HWLOC_CPUBIND_ALLFLAGS) {
    errno = EINVAL;
    return -1;
  }

  set = hwloc_fix_cpubind(topology, set);
  if (!set)
    return -1;

  if (topology->binding_hooks.set_proc_cpubind)
    return topology->binding_hooks.set_proc_cpubind(topology, pid, set, flags);

  errno = ENOSYS;
  return -1;
}

```

这段代码定义了一个名为 `hwloc_get_proc_cpubind` 的函数，属于 `hwloc_topology_ex` 库。它用于获取在给定拓扑结构中的一个指定 PID 对应的 CPU  binding 设置。

函数接收四个参数：

1. `topology`：指定了拓扑结构的句柄。
2. `pid`：要获取的 PID。
3. `set`：用于设置 CPU 绑定的二进制位图像。
4. `flags`：附加的标志，以决定是否使用其他绑定的设置。

函数首先检查是否有禁止 CPU 绑定所有标志位的设置，如果没有，则返回 EINVAL 和 -1。否则，将调用 `topology->binding_hooks.get_proc_cpubind` 函数，如果该函数成功返回，则返回成功，否则返回 ENOSYS 和 -1。


```cpp
int
hwloc_get_proc_cpubind(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_bitmap_t set, int flags)
{
  if (flags & ~HWLOC_CPUBIND_ALLFLAGS) {
    errno = EINVAL;
    return -1;
  }

  if (topology->binding_hooks.get_proc_cpubind)
    return topology->binding_hooks.get_proc_cpubind(topology, pid, set, flags);

  errno = ENOSYS;
  return -1;
}

```

这段代码定义了一个名为`hwloc_set_thread_cpubind`的函数，属于`hwloc_topology_t`类型的内部函数。

这个函数接受四个参数：

1. `topology`：要操作的硬件布局结构，可以是`hwloc_topology_t`、`hwloc_device_t`或者`hwloc_struct_layout_t`。
2. `tid`：线程的ID，这个线程属于哪个物理设备（CPU、GPU等）上的哪个CPU核心呢？
3. `set`：用于指定哪些硬件资源要设置为绑定状态的位掩，这个位掩可以是`hwloc_const_bitmap_t`，它会输出一个以`HWLOC_CPUBIND_ALLFLAGS`为尾的掩码，表示对应的CPU核心和缓存等可以设置为绑定状态。这个掩码的求反得到的就是`HWLOC_CPUBIND_ANYFLAGS`，表示对应的CPU核心和缓存等禁止设置为绑定状态。
4. `flags`：用于指定`hwloc_set_thread_cpubind`函数需要执行的指令列，包括了`HWLOC_CPUBIND_ALLFLAGS`、`HWLOC_CPUBIND_ANYFLAGS`、`HWLOC_CPUBIND_FOREGROUNDFLAGS`和`HWLOC_CPUBIND_POSTBACKFLAGS`，分别表示：
	* `HWLOC_CPUBIND_ALLFLAGS`：设置为绑定状态的所有CPU核心和缓存。
	* `HWLOC_CPUBIND_ANYFLAGS`：设置为绑定状态的CPU核心，但不包括缓存。
	* `HWLOC_CPUBIND_FOREGROUNDFLAGS`：设置为当前线程的主机CPU核心。
	* `HWLOC_CPUBIND_POSTBACKFLAGS`：设置为当前线程的协程CPU核心。

函数的作用是帮助用户设置硬件资源的绑定状态，使得`hwloc_topology_t`中的硬件资源可以按需绑定到线程上。首先，检查是否有`HWLOC_CPUBIND_ALLFLAGS`，如果没有，就设置一下。然后，尝试使用`hwloc_fix_cpubind`函数设置设置，如果失败就返回-1。接着，检查`topology->binding_hooks.set_thread_cpubind`函数是否支持设置，如果支持，就使用它设置设置，否则直接返回`ENOSYS`。最后，如果设置了绑定状态，就返回设置状态的操作系统返回值，否则返回-1。


```cpp
#ifdef hwloc_thread_t
int
hwloc_set_thread_cpubind(hwloc_topology_t topology, hwloc_thread_t tid, hwloc_const_bitmap_t set, int flags)
{
  if (flags & ~HWLOC_CPUBIND_ALLFLAGS) {
    errno = EINVAL;
    return -1;
  }

  set = hwloc_fix_cpubind(topology, set);
  if (!set)
    return -1;

  if (topology->binding_hooks.set_thread_cpubind)
    return topology->binding_hooks.set_thread_cpubind(topology, tid, set, flags);

  errno = ENOSYS;
  return -1;
}

```

这段代码定义了一个名为 `hwloc_get_thread_cpubind` 的函数，属于 `hwloc_lowlevel_t` 类型。它的作用是获取在给定的 `hwloc_topology_t` 结构中，具有指定 `hwloc_bitmap_t` 设置的 `hwloc_thread_t` 实例的 `CPU 绑定`。

函数首先检查是否有 `HWLOC_CPUBIND_ALLFLAGS` 标志位，如果没有，则表示有误，返回 `-1`。接着，函数会尝试使用 `topology->binding_hooks.get_thread_cpubind` 函数，如果这个函数有效，则使用它获取绑定。如果 `get_thread_cpubind` 函数无效，则返回 `ENOSYS`。

如果 `hwloc_get_thread_cpubind` 函数自身也无效，则输出错误并返回 `-1`。


```cpp
int
hwloc_get_thread_cpubind(hwloc_topology_t topology, hwloc_thread_t tid, hwloc_bitmap_t set, int flags)
{
  if (flags & ~HWLOC_CPUBIND_ALLFLAGS) {
    errno = EINVAL;
    return -1;
  }

  if (topology->binding_hooks.get_thread_cpubind)
    return topology->binding_hooks.get_thread_cpubind(topology, tid, set, flags);

  errno = ENOSYS;
  return -1;
}
#endif

```

这段代码定义了一个名为 `hwloc_get_last_cpu_location` 的函数，属于 `hwloc_topology_t` 结构体及其 `hwloc_bitmap_t` 成员的函数。

它的作用是返回一个整数，表示在给定的 `hwloc_topology_t` 结构体和 `hwloc_bitmap_t` 掩码中，最近与 `hwloc_cpubinding_t` 相关的 `hwloc_binding_t` 结构体中，用于指定处理器或线程的最后 CPU 位置的函数的返回值。

函数实现中，首先检查给定的 `flags` 是否包括 `HWLOC_CPUBIND_ALLFLAGS`，如果是，则返回 `-1`，否则继续执行。接着，按照不同的 `hwloc_cpubinding_t` 类型进行判断。

如果是 `hwloc_cpubinding_t.CPUBIND_ALL`，那么函数首先调用 `topology->binding_hooks.get_thisproc_last_cpu_location`，如果已经调用成功，则返回它的返回值。

如果是 `hwloc_cpubinding_t.CPUBIND_PROCESS` 或 `hwloc_cpubinding_t.CPUBIND_THREAD`，那么函数首先调用 `topology->binding_hooks.get_thisthread_last_cpu_location`，如果已经调用成功，则返回它的返回值。

如果是 `hwloc_cpubinding_t.CPUBIND_ALLFLAGS`，那么直接返回 `topology->binding_hooks.get_thisproc_last_cpu_location` 的返回值，不过需要检查给定的 `hwloc_topology_t` 和 `hwloc_bitmap_t` 是否支持此种 binding。


```cpp
int
hwloc_get_last_cpu_location(hwloc_topology_t topology, hwloc_bitmap_t set, int flags)
{
  if (flags & ~HWLOC_CPUBIND_ALLFLAGS) {
    errno = EINVAL;
    return -1;
  }

  if (flags & HWLOC_CPUBIND_PROCESS) {
    if (topology->binding_hooks.get_thisproc_last_cpu_location)
      return topology->binding_hooks.get_thisproc_last_cpu_location(topology, set, flags);
  } else if (flags & HWLOC_CPUBIND_THREAD) {
    if (topology->binding_hooks.get_thisthread_last_cpu_location)
      return topology->binding_hooks.get_thisthread_last_cpu_location(topology, set, flags);
  } else {
    if (topology->binding_hooks.get_thisproc_last_cpu_location) {
      int err = topology->binding_hooks.get_thisproc_last_cpu_location(topology, set, flags);
      if (err >= 0 || errno != ENOSYS)
        return err;
      /* ENOSYS, fallback */
    }
    if (topology->binding_hooks.get_thisthread_last_cpu_location)
      return topology->binding_hooks.get_thisthread_last_cpu_location(topology, set, flags);
  }

  errno = ENOSYS;
  return -1;
}

```

这段代码定义了一个名为`hwloc_get_proc_last_cpu_location`的函数，属于`hwloc_topology_t`类的成员函数。

它的作用是获取一个进程（`hwloc_pid_t`）在当前操作系统中最近的CPU位置，用于定位程序的性能瓶颈。它可以在`hwloc_topology_t`的`binding_hooks`成员函数中调用，或者在`hwloc_device_t`的`proc_path_lookup_avail`函数中使用`hwloc_topology_t`的`get_proc_last_cpu_location`函数调用。

函数的参数包括：

- `topology`：`hwloc_topology_t`类型的数据结构，用于指定分析的硬件布局。
- `pid`：要分析的进程ID。
- `set`：用于指定哪些CPU位置需要返回，可以通过设置`HWLOC_CPUBIND_ALLFLAGS`标志位来获取所有的CPU位置。
- `flags`：设置为`HWLOC_CPUBIND_ALLFLAGS`标志位，用于指定需要返回的CPU位置。

函数返回：

- 如果`topology->binding_hooks.get_proc_last_cpu_location`函数成功调用，返回它返回的CPU位置。
- 如果任何函数失败，返回-1，并设置`errno`为EINVAL。


```cpp
int
hwloc_get_proc_last_cpu_location(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_bitmap_t set, int flags)
{
  if (flags & ~HWLOC_CPUBIND_ALLFLAGS) {
    errno = EINVAL;
    return -1;
  }

  if (topology->binding_hooks.get_proc_last_cpu_location)
    return topology->binding_hooks.get_proc_last_cpu_location(topology, pid, set, flags);

  errno = ENOSYS;
  return -1;
}

```

这段代码定义了一个名为 `hwloc_fix_membind` 的函数，属于 `hwloc_membind` 系列的函数。它的作用是处理一个 `hwloc_topology_t` 类型的顶部和 `hwloc_const_nodeset_t` 类型的节点集合，输出一个 `hwloc_const_nodeset_t` 类型的子节点集合。

具体来说，这段代码首先获取 `topology` 的节点集合和 `complete_nodeset` 的节点集合，然后检查输入的 `nodeset` 节点集合是否与 `complete_nodeset` 一致。如果不一致，函数将返回 `NULL`，并打印错误信息。如果 `nodeset` 与 `complete_nodeset` 一致，则函数返回 `complete_nodeset`，否则返回 `nodeset`。


```cpp
#define HWLOC_MEMBIND_ALLFLAGS (HWLOC_MEMBIND_PROCESS|HWLOC_MEMBIND_THREAD|HWLOC_MEMBIND_STRICT|HWLOC_MEMBIND_MIGRATE|HWLOC_MEMBIND_NOCPUBIND|HWLOC_MEMBIND_BYNODESET)

static hwloc_const_nodeset_t
hwloc_fix_membind(hwloc_topology_t topology, hwloc_const_nodeset_t nodeset)
{
  hwloc_const_bitmap_t topology_nodeset = hwloc_topology_get_topology_nodeset(topology);
  hwloc_const_bitmap_t complete_nodeset = hwloc_topology_get_complete_nodeset(topology);

  if (hwloc_bitmap_iszero(nodeset)) {
    errno = EINVAL;
    return NULL;
  }

  if (!hwloc_bitmap_isincluded(nodeset, complete_nodeset)) {
    errno = EINVAL;
    return NULL;
  }

  if (hwloc_bitmap_isincluded(topology_nodeset, nodeset))
    return complete_nodeset;

  return nodeset;
}

```

这段代码定义了一个名为 `hwloc_fix_membind_cpuset` 的函数，属于 `hwloc_core` 库。它的作用是处理在 `hwloc_topology` 类型的 topology、`hwloc_nodeset` 类型的 nodeset 和 `hwloc_const_cpuset_t` 类型的 cpuset 中的一个或多个输入参数。

具体来说，函数接受三个参数：

1. `topology`：表示要处理的 topology 类型；
2. `nodeset`：表示要在其中创建的 nodeset 类型；
3. `cpuset`：表示一个或多个要设置的 cpuset。

函数首先根据传递的 topology 类型获取相应的 topology cpuset，并从 topology 中剔除所有不在 complete set 中的节点，然后检查 cpuset 是否为空，如果是，则表示操作成功，返回 0；如果不是，则继续处理。

接着，函数检查 cpuset 是否在 complete set 中，如果是，则将 corresponding nodeset 复制到 nodeset 中；如果不是，则使用 cpuset 将 topology 中的节点映射到 nodeset 中，并将结果返回。


```cpp
static int
hwloc_fix_membind_cpuset(hwloc_topology_t topology, hwloc_nodeset_t nodeset, hwloc_const_cpuset_t cpuset)
{
  hwloc_const_bitmap_t topology_set = hwloc_topology_get_topology_cpuset(topology);
  hwloc_const_bitmap_t complete_set = hwloc_topology_get_complete_cpuset(topology);
  hwloc_const_bitmap_t complete_nodeset = hwloc_topology_get_complete_nodeset(topology);

  if (hwloc_bitmap_iszero(cpuset)) {
    errno = EINVAL;
    return -1;
  }

  if (!hwloc_bitmap_isincluded(cpuset, complete_set)) {
    errno = EINVAL;
    return -1;
  }

  if (hwloc_bitmap_isincluded(topology_set, cpuset)) {
    hwloc_bitmap_copy(nodeset, complete_nodeset);
    return 0;
  }

  hwloc_cpuset_to_nodeset(topology, cpuset, nodeset);
  return 0;
}

```

The function `hwloc_set_membind_by_nodeset` sets the global memory-bindings for a given nodeset in the specified topology. It takes a `hwloc_const_nodeset_t` that identifies the nodeset, a `hwloc_membind_policy_t` that specifies the policy to use for the memory binding, and a `int` flag or a combination of flags indicating the memory binding flags (e.g., `HWLOC_MEMBIND_ALLFLAGS`, `HWLOC_MEMBIND_THREAD`, `HWLOC_MEMBIND_PROCESS`, `HWLOC_MEMBIND_THREAD`, `HWLOC_MEMBIND_PROCESS`, `HWLOC_MEMBIND_FIFO`, `HWLOC_MEMBIND_FIFO_BUF)).

The function first checks if the given policy is valid and sets the nodeset if it's not. If the nodeset is set, the function attempts to set the global memory bindings for the nodeset according to the specified policy. If the function cannot set the memory bindings, it returns `ENOSYS`.

If the `hwloc__check_membind_policy` function fails with an error, the function will return `-1`.


```cpp
static __hwloc_inline int hwloc__check_membind_policy(hwloc_membind_policy_t policy)
{
  if (policy == HWLOC_MEMBIND_DEFAULT
      || policy == HWLOC_MEMBIND_FIRSTTOUCH
      || policy == HWLOC_MEMBIND_BIND
      || policy == HWLOC_MEMBIND_INTERLEAVE
      || policy == HWLOC_MEMBIND_NEXTTOUCH)
    return 0;
  return -1;
}

static int
hwloc_set_membind_by_nodeset(hwloc_topology_t topology, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags)
{
  if ((flags & ~HWLOC_MEMBIND_ALLFLAGS) || hwloc__check_membind_policy(policy) < 0) {
    errno = EINVAL;
    return -1;
  }

  nodeset = hwloc_fix_membind(topology, nodeset);
  if (!nodeset)
    return -1;

  if (flags & HWLOC_MEMBIND_PROCESS) {
    if (topology->binding_hooks.set_thisproc_membind)
      return topology->binding_hooks.set_thisproc_membind(topology, nodeset, policy, flags);
  } else if (flags & HWLOC_MEMBIND_THREAD) {
    if (topology->binding_hooks.set_thisthread_membind)
      return topology->binding_hooks.set_thisthread_membind(topology, nodeset, policy, flags);
  } else {
    if (topology->binding_hooks.set_thisproc_membind) {
      int err = topology->binding_hooks.set_thisproc_membind(topology, nodeset, policy, flags);
      if (err >= 0 || errno != ENOSYS)
        return err;
      /* ENOSYS, fallback */
    }
    if (topology->binding_hooks.set_thisthread_membind)
      return topology->binding_hooks.set_thisthread_membind(topology, nodeset, policy, flags);
  }

  errno = ENOSYS;
  return -1;
}

```

这段代码定义了一个名为`hwloc_set_membind`的函数，属于`hwloc_membind_policy_t`类的成员函数。

它的作用是设置硬件布局（hwloc_topology_t）中的内存绑定（hwloc_const_bitmap_t set），以及控制内存绑定的策略（hwloc_membind_policy_t policy）和标志位（int flags）。

具体来说，函数可以实现以下操作：

1. 如果设置的标志位`flags`中包含`HWLOC_MEMBIND_BYNODESET`，则执行`hwloc_set_membind_by_nodeset`函数，对指定的节点集（hwloc_nodeset_t）进行内存绑定；
2. 如果`flags`中包含`HWLOC_MEMBIND_SET_CPUSET`，则执行`hwloc_fix_membind_cpuset`函数，对指定的CPU集（hwloc_const_bitmap_t）进行内存绑定；
3. 如果`flags`中包含`HWLOC_MEMBIND_BY_NODESET`，则按照设置的节点集进行内存绑定；
4. 如果`flags`中包含`HWLOC_MEMBIND_SET_FLAGS`，则根据设置的标志位执行相应的内存绑定策略。

函数的返回值是一个整数类型的`ret`，用于表示设置内存绑定的成功或失败。


```cpp
int
hwloc_set_membind(hwloc_topology_t topology, hwloc_const_bitmap_t set, hwloc_membind_policy_t policy, int flags)
{
  int ret;

  if (flags & HWLOC_MEMBIND_BYNODESET) {
    ret = hwloc_set_membind_by_nodeset(topology, set, policy, flags);
  } else {
    hwloc_nodeset_t nodeset = hwloc_bitmap_alloc();
    if (hwloc_fix_membind_cpuset(topology, nodeset, set))
      ret = -1;
    else
      ret = hwloc_set_membind_by_nodeset(topology, nodeset, policy, flags);
    hwloc_bitmap_free(nodeset);
  }
  return ret;
}

```

这段代码定义了一个名为 `hwloc_get_membind_by_nodeset` 的函数，属于 `hwloc_topology_t` 系列的成员函数。它的作用是返回一个整数，表示与给定 `hwloc_nodeset_t` 和 `hwloc_membind_policy_t` 相关的内存绑定位置。

函数首先检查给定的 `flags` 是否包含 `HWLOC_MEMBIND_ALLFLAGS`，如果是，函数返回 `-1`，否则允许函数继续执行。

接着，函数根据 `flags` 中的某些子标量来判断如何获取与 `hwloc_nodeset_t` 绑定的内存。如果 `flags` 中包含 `HWLOC_MEMBIND_PROCESS` 或 `HWLOC_MEMBIND_THREAD`，函数会尝试调用对应的过程的 `get_thisproc_membind` 函数。如果 `flags` 中包含 `HWLOC_MEMBIND_ALLFLAGS`，函数会直接尝试调用 `get_thisproc_membind` 函数。

如果 `topology->binding_hooks.get_thisproc_membind` 和 `topology->binding_hooks.get_thisthread_membind` 函数都存在，函数将返回它们的返回值。否则，函数调用 `topology->binding_hooks.get_enOSYS_error` 函数，并将结果加入 `errno` 变量中。最后，函数返回 `-1`，表示函数执行失败。


```cpp
static int
hwloc_get_membind_by_nodeset(hwloc_topology_t topology, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags)
{
  if (flags & ~HWLOC_MEMBIND_ALLFLAGS) {
    errno = EINVAL;
    return -1;
  }

  if (flags & HWLOC_MEMBIND_PROCESS) {
    if (topology->binding_hooks.get_thisproc_membind)
      return topology->binding_hooks.get_thisproc_membind(topology, nodeset, policy, flags);
  } else if (flags & HWLOC_MEMBIND_THREAD) {
    if (topology->binding_hooks.get_thisthread_membind)
      return topology->binding_hooks.get_thisthread_membind(topology, nodeset, policy, flags);
  } else {
    if (topology->binding_hooks.get_thisproc_membind) {
      int err = topology->binding_hooks.get_thisproc_membind(topology, nodeset, policy, flags);
      if (err >= 0 || errno != ENOSYS)
        return err;
      /* ENOSYS, fallback */
    }
    if (topology->binding_hooks.get_thisthread_membind)
      return topology->binding_hooks.get_thisthread_membind(topology, nodeset, policy, flags);
  }

  errno = ENOSYS;
  return -1;
}

```

这段代码定义了一个名为 `hwloc_get_membind` 的函数，用于在 HWLOC 系统中获取内存绑定。

函数接受四个参数：

- `topology`：表示要获取内存绑定的拓扑结构。
- `set`：表示要设置的内存绑定集合。这个参数可以是单个内存绑定，也可以是一个内存绑定的集合。
- `policy`：表示内存绑定的策略，可以是以下几种策略之一：
 - `HWLOC_MEMBIND_ACTIVE_EXEC`
 - `HWLOC_MEMBIND_DEFAULT`
 - `HWLOC_MEMBIND_FREE_EXEC`
 - `HWLOC_MEMBIND_HWNODE`
 - `HWLOC_MEMBIND_NON_ZERO_EXEC`
 - `HWLOC_MEMBIND_NON_ZERO_TORQUE`
 - `HWLOC_MEMBIND_PARTIAL_EXEC`
 - `HWLOC_MEMBIND_PARTIAL_TORQUE`
 - `HWLOC_MEMBIND_SINGLE_EXEC`
 - `HWLOC_MEMBIND_SINGLE_TORQUE`

- `flags`：设置为以下值时，使用这个策略：
 - `HWLOC_MEMBIND_BYNODESET`：返回每个节点设置的内存绑定。
 - `HWLOC_MEMBIND_SET_EXEC`：在所有节点上都设置的内存绑定。
 - `HWLOC_MEMBIND_SET_NON_ZERO_EXEC`：设置非零内存绑定的节点上。
 - `HWLOC_MEMBIND_SET_PARTIAL_EXEC`：设置部分节点上。
 - `HWLOC_MEMBIND_SET_PARTIAL_TORQUE`：设置部分节点上的内存绑定。
 - `HWLOC_MEMBIND_SET_SINGLE_EXEC`：设置单个节点上的内存绑定。
 - `HWLOC_MEMBIND_SET_SINGLE_TORQUE`：设置单个节点上的内存绑定。

函数首先检查 `flags` 是否与 `HWLOC_MEMBIND_BYNODESET` 相关联，如果是，就直接使用 `hwloc_get_membind_by_nodeset` 函数，否则会尝试使用 `hwloc_nodeset_get_membind` 函数。如果仍然失败，说明有节点设置非零的内存绑定，那么就需要遍历设置节点，并使用 `hwloc_get_membind_by_nodeset` 函数获取每个节点的内存绑定，最后再将整个节点设置的内存绑定输出。


```cpp
int
hwloc_get_membind(hwloc_topology_t topology, hwloc_bitmap_t set, hwloc_membind_policy_t * policy, int flags)
{
  int ret;

  if (flags & HWLOC_MEMBIND_BYNODESET) {
    ret = hwloc_get_membind_by_nodeset(topology, set, policy, flags);
  } else {
    hwloc_nodeset_t nodeset = hwloc_bitmap_alloc();
    ret = hwloc_get_membind_by_nodeset(topology, nodeset, policy, flags);
    if (!ret)
      hwloc_cpuset_from_nodeset(topology, set, nodeset);
    hwloc_bitmap_free(nodeset);
  }

  return ret;
}

```

该函数定义了一个名为 `hwloc_set_proc_membind_by_nodeset` 的函数，属于 `hwloc_syscall` 函数家族。它的作用是设置一个进程的内存绑定。

它的参数包括：

- `topology`：表示拓扑结构，可以是 `hwloc_topology_t` 或 `hwloc_topology_e` 类型。
- `pid`：表示要绑定的进程 ID。
- `nodeset`：表示要绑定的节点集。
- `policy`：表示内存绑定策略，可以是 `hwloc_membind_policy_t` 类型。
- `flags`：表示标志位，可以包含 `HWLOC_MEMBIND_ALLFLAGS`，它的含义如下：

 - `HWLOC_MEMBIND_ALLFLAGS`：表示所有可用的内存绑定策略。
 - `～HWLOC_MEMBIND_ALLFLAGS`：表示保留的内存绑定策略。

它的返回值是一个整数，表示是否成功设置内存绑定。如果设置成功，返回 0；如果失败，返回一个负数，可能的错误码包括：

- `EINVAL`：表示 invalid argument。
- `ENOSYS`：表示 out of system error。

注意：如果 `policy` 的值为 `hwloc_membind_policy_t::HWLOC_MEMBIND_DEFAULT`，则调用 `hwloc__check_membind_policy` 函数进行默认检查，否则不会调用这个函数。


```cpp
static int
hwloc_set_proc_membind_by_nodeset(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags)
{
  if ((flags & ~HWLOC_MEMBIND_ALLFLAGS) || hwloc__check_membind_policy(policy) < 0) {
    errno = EINVAL;
    return -1;
  }

  nodeset = hwloc_fix_membind(topology, nodeset);
  if (!nodeset)
    return -1;

  if (topology->binding_hooks.set_proc_membind)
    return topology->binding_hooks.set_proc_membind(topology, pid, nodeset, policy, flags);

  errno = ENOSYS;
  return -1;
}


```

这段代码定义了一个名为 `hwloc_set_proc_membind` 的函数，属于 `hwloc_membind_policy_t` 类的成员函数。

它的作用是设置一个进程的内存绑定，用于指定在哪些内存区域绑定内存。它接受四个参数：

1. `topology`：指网络硬件设备的 topology 结构，用于定义设备在网络中的位置和类型。
2. `pid`：要绑定的进程 ID。
3. `set`：用于指定内存区域设置的位图。
4. `policy`：用于指定内存区域绑定的策略，可以是 `hwloc_membind_policy_t` 类型的成员函数指针。
5. `flags`：设置内存区域绑定的标志，可以包括多个标志，如 `HWLOC_MEMBIND_BYNODESET`，`HWLOC_MEMBIND_SET_CPU_SET` 等。

函数实现中，首先检查 `flags` 是否包括 `HWLOC_MEMBIND_BYNODESET`，如果是，则执行以下操作：

1. 设置 `topology` 中的 `pid` 和 `set`，然后设置绑定的策略。
2. 如果 `set` 是 `hwloc_membind_policy_t` 类型的指针，则直接使用策略的值，否则执行下一个操作。
3. 如果所有设置都正确，则返回 0，否则返回 `-1`，并设置返回值为 `-1`。

如果 `flags` 中不包括 `HWLOC_MEMBIND_BYNODESET`，函数会执行以下操作：

1. 创建一个空节点集 `nodeset`，并尝试使用 `hwloc_fix_membind_cpuset` 设置每个 CPU 设置的内存区域。
2. 如果设置成功，则执行设置策略的函数，否则执行下一个操作。
3. 使用 `hwloc_set_proc_membind_by_nodeset` 设置绑定的内存区域。
4. 释放内存区域，并将 `nodeset` 释放。

最后，返回函数的整数表示，如果设置成功则返回 0，否则返回 `-1`。


```cpp
int
hwloc_set_proc_membind(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_const_bitmap_t set, hwloc_membind_policy_t policy, int flags)
{
  int ret;

  if (flags & HWLOC_MEMBIND_BYNODESET) {
    ret = hwloc_set_proc_membind_by_nodeset(topology, pid, set, policy, flags);
  } else {
    hwloc_nodeset_t nodeset = hwloc_bitmap_alloc();
    if (hwloc_fix_membind_cpuset(topology, nodeset, set))
      ret = -1;
    else
      ret = hwloc_set_proc_membind_by_nodeset(topology, pid, nodeset, policy, flags);
    hwloc_bitmap_free(nodeset);
  }

  return ret;
}

```

这段代码定义了一个名为 `hwloc_get_proc_membind_by_nodeset` 的函数，属于 `hwloc_membind_policy_t` 类的成员函数。

它的作用是：通过 `hwloc_topology_t` 类型的顶类和 `hwloc_pid_t` 类型的参数 `pid` 和 `hwloc_nodeset_t` 类型的参数 `nodeset`，查找并返回 `hwloc_membind_policy_t` 类型的 `policy` 变量所指的内存绑定策略的实例。如果 `flags` 中包含了 `HWLOC_MEMBIND_ALLFLAGS`，则会抛出 `errno`，否则返回结果。

函数的实现中，首先检查 `flags` 是否为 `0`，如果是，则表示需要返回 `-1`，否则继续执行。接着，如果 `topology` 对象中存在 `binding_hooks.get_proc_membind` 函数，则直接调用该函数，并将 `topology`、`pid`、`nodeset`、`policy` 和 `flags` 作为参数传入。如果 `topology` 对象中不存在 `binding_hooks.get_proc_membind` 函数，则执行错误操作并返回 `-1`。


```cpp
static int
hwloc_get_proc_membind_by_nodeset(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags)
{
  if (flags & ~HWLOC_MEMBIND_ALLFLAGS) {
    errno = EINVAL;
    return -1;
  }

  if (topology->binding_hooks.get_proc_membind)
    return topology->binding_hooks.get_proc_membind(topology, pid, nodeset, policy, flags);

  errno = ENOSYS;
  return -1;
}

```

这段代码定义了一个名为 `hwloc_get_proc_membind` 的函数，它属于 `hwloc_membind_policy_t` 类的成员函数。

它的作用是获取一个 `hwloc_membind_policy_t` 类型的数据，通过 `hwloc_topology_t` 和 `hwloc_pid_t` 参数，然后传递给 `hwloc_get_proc_membind_by_nodeset` 函数进行处理，最后返回相应的结果。

函数可以分为两部分：

1. 如果 `flags` 中包含了 `HWLOC_MEMBIND_BYNODESET`，那么调用 `hwloc_get_proc_membind_by_nodeset` 函数，传递 `topology`、`pid`、`set` 和 `policy` 参数，并将 `flags` 作为参数传递。

2. 否则，创建一个 `hwloc_nodeset_t` 类型的节点集，然后调用 `hwloc_get_proc_membind_by_nodeset` 函数，传递 `topology`、`pid` 和 `nodeset` 参数，并将 `policy` 和 `flags` 作为参数传递。

如果第一次调用 `hwloc_get_proc_membind_by_nodeset` 函数失败，那么就需要在创建节点集之后，将当前节点集映射到对应的 `hwloc_membind_policy_t` 数据上。


```cpp
int
hwloc_get_proc_membind(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_bitmap_t set, hwloc_membind_policy_t * policy, int flags)
{
  int ret;

  if (flags & HWLOC_MEMBIND_BYNODESET) {
    ret = hwloc_get_proc_membind_by_nodeset(topology, pid, set, policy, flags);
  } else {
    hwloc_nodeset_t nodeset = hwloc_bitmap_alloc();
    ret = hwloc_get_proc_membind_by_nodeset(topology, pid, nodeset, policy, flags);
    if (!ret)
      hwloc_cpuset_from_nodeset(topology, set, nodeset);
    hwloc_bitmap_free(nodeset);
  }

  return ret;
}

```

这段代码定义了一个名为 `hwloc_set_area_membind_by_nodeset` 的函数，属于 `hwloc_bindings` 系列的函数。它的作用是设置一个 `HWLOC_MEMBIND_NONE` 类型的内存区域，将其绑定到指定的 `HWLOC_CONST_NODESET` 上，并设置相应的 `HWLOC_MEMBIND_POLICY` 和 `HWLOC_FLAGS` 标志。以下是具体的实现步骤：

1. 首先检查传入的 `flags` 是否包含 `HWLOC_MEMBIND_ALLFLAGS`，如果是，函数返回 `EINVAL` 并输出 `-1`，否则继续执行。
2. 如果 `len` 为 0，则不做任何操作，返回 `0`。
3. 创建一个 `hwloc_const_nodeset_t` 类型的变量 `nodeset`，如果创建失败则返回 `-1`。
4. 如果 `topology` 对应的 `binding_hooks.set_area_membind` 函数有效，函数调用 `topology->binding_hooks.set_area_membind` 函数，并将传入的参数传递给它，得到一个 `HWLOC_MEMBIND_NONE` 类型的结果。
5. 如果以上所有步骤都成功，函数返回 `0`。


```cpp
static int
hwloc_set_area_membind_by_nodeset(hwloc_topology_t topology, const void *addr, size_t len, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags)
{
  if ((flags & ~HWLOC_MEMBIND_ALLFLAGS) || hwloc__check_membind_policy(policy) < 0) {
    errno = EINVAL;
    return -1;
  }

  if (!len)
    /* nothing to do */
    return 0;

  nodeset = hwloc_fix_membind(topology, nodeset);
  if (!nodeset)
    return -1;

  if (topology->binding_hooks.set_area_membind)
    return topology->binding_hooks.set_area_membind(topology, addr, len, nodeset, policy, flags);

  errno = ENOSYS;
  return -1;
}

```

这段代码定义了一个名为 `hwloc_set_area_membind` 的函数，属于 `hwloc_membind` 系列的函数。其作用是设置一个内存区域，将其绑定到指定的硬件设备上，以便于后续的 HWLOC 配置。以下是这个函数的实现细节：

1. 首先定义了一个名为 `topology` 的变量，其类型为 `hwloc_topology_t`，这个变量在后续的函数调用中可能会用到，所以需要定义。
2. 然后定义了一个名为 `addr` 的参数，其类型为 `const void *`，用于指定要绑定的内存区域的起始地址。
3. 定义了一个名为 `len` 的参数，其类型为 `size_t`，用于指定要绑定的内存区域的长度。
4. 接下来定义了一个名为 `set` 的参数，其类型为 `hwloc_const_bitmap_t`，用于指定用于设置内存区域的位图。
5. 定义了一个名为 `policy` 的参数，其类型为 `hwloc_membind_policy_t`，用于指定内存区域绑定的策略。
6. 定义了一个名为 `flags` 的参数，其类型为 `int`，用于指定与 `HWLOC_MEMBIND_BYNODESET` 相关的标志位。如果这个标志位为 1，则表示使用基于节点集的设置策略。
7. 接着定义了一个名为 `nodeset` 的变量，其类型为 `hwloc_nodeset_t`，用于保存用于设置内存区域的节点集。
8. 如果 `hwloc_fix_membind_cpuset` 函数成功，则执行以下操作：设置内存区域为绑定到指定 CPU 的节点集，并返回 -1。否则，继续执行以下操作：设置内存区域为绑定到指定 CPU 的节点集，并尝试返回之前设置的 -1，如果设置成功则返回 0，否则继续执行。
9. 最后返回设置内存区域的操作结果。


```cpp
int
hwloc_set_area_membind(hwloc_topology_t topology, const void *addr, size_t len, hwloc_const_bitmap_t set, hwloc_membind_policy_t policy, int flags)
{
  int ret;

  if (flags & HWLOC_MEMBIND_BYNODESET) {
    ret = hwloc_set_area_membind_by_nodeset(topology, addr, len, set, policy, flags);
  } else {
    hwloc_nodeset_t nodeset = hwloc_bitmap_alloc();
    if (hwloc_fix_membind_cpuset(topology, nodeset, set))
      ret = -1;
    else
      ret = hwloc_set_area_membind_by_nodeset(topology, addr, len, nodeset, policy, flags);
    hwloc_bitmap_free(nodeset);
  }

  return ret;
}

```

这段代码定义了一个名为 `hwloc_get_area_membind_by_nodeset` 的函数，属于 `hwloc_topology_ex` 类的成员函数。它的作用是获取绑定到指定 `hwloc_nodeset_t` 上的 `hwloc_membind_policy_t` 类型的内存区域，并返回区域的有效性（0 或 返回 -1）。

函数首先检查传递给它的 `flags` 是否包含 `HWLOC_MEMBIND_ALLFLAGS`，如果是，函数将返回 `EINVAL`，否则函数将返回 `0`。

接下来，函数检查传递给它的 `len` 参数是否为 0，如果是，函数将返回 `EINVAL`，否则函数将返回 `0`。

然后，函数通过 `topology->binding_hooks.get_area_membind` 函数，或者调用者自身提供的函数，获取绑定到指定 `hwloc_nodeset_t` 上的 `hwloc_membind_policy_t` 类型的内存区域。

如果 `get_area_membind` 函数成功，函数将返回 0，否则函数将返回 `EINVAL`。


```cpp
static int
hwloc_get_area_membind_by_nodeset(hwloc_topology_t topology, const void *addr, size_t len, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags)
{
  if (flags & ~HWLOC_MEMBIND_ALLFLAGS) {
    errno = EINVAL;
    return -1;
  }

  if (!len) {
    /* nothing to query */
    errno = EINVAL;
    return -1;
  }

  if (topology->binding_hooks.get_area_membind)
    return topology->binding_hooks.get_area_membind(topology, addr, len, nodeset, policy, flags);

  errno = ENOSYS;
  return -1;
}

```

这段代码定义了一个名为 `hwloc_get_area_membind` 的函数，它属于 `hwloc_membind_policy_t` 类的成员函数。它的作用是获取一个特定高度的共享内存区域，并设置或取消设置的内存区域。

具体来说，函数接受四个参数：

- `hwloc_topology_t`：表示 topology 数据结构的类型，它是由 `hwloc_layer_type_t` 和 `hwloc_device_t` 组成的结构体，用于描述要操作的硬件层。
- `const void *` `addr`：要操作的共享内存区域的目标地址。
- `size_t` `len`：目标地址区的大小，以字节计。
- `hwloc_bitmap_t` `set`：用于设置或取消内存区域的掩码。
- `hwloc_membind_policy_t *` `policy`：指向用于控制内存区域的政策的指针。
- `int` `flags`：设置内存区域的方式，可以使用以下设置值的索引：
 - HWLOC_MEMBIND_BYNODESET：按节点设置内存区域
 - HWLOC_MEMBIND_SET_CPU：按 CPU 设置内存区域
 - HWLOC_MEMBIND_SET：允许设置内存区域的值

函数首先检查设置的内存区域是否使用节点设置，如果是，就执行该操作。否则，它创建一个节点集，设置为允许设置，然后尝试使用节点集设置内存区域，如果失败就返回。

最后，函数返回成功设置或取消内存区域的结果。


```cpp
int
hwloc_get_area_membind(hwloc_topology_t topology, const void *addr, size_t len, hwloc_bitmap_t set, hwloc_membind_policy_t * policy, int flags)
{
  int ret;

  if (flags & HWLOC_MEMBIND_BYNODESET) {
    ret = hwloc_get_area_membind_by_nodeset(topology, addr, len, set, policy, flags);
  } else {
    hwloc_nodeset_t nodeset = hwloc_bitmap_alloc();
    ret = hwloc_get_area_membind_by_nodeset(topology, addr, len, nodeset, policy, flags);
    if (!ret)
      hwloc_cpuset_from_nodeset(topology, set, nodeset);
    hwloc_bitmap_free(nodeset);
  }

  return ret;
}

```

这段代码定义了一个名为 `hwloc_get_area_memlocation_by_nodeset` 的函数，属于 `hwloc_topology_t` 类的成员函数。

它的作用是：通过 `topology->binding_hooks.get_area_memlocation` 函数，获取指定 `addr` 在 `nodeset` 中的内存位置，并返回它。如果 `len` 参数不正确，函数返回 0；如果 `topology` 对象中 `binding_hooks` 成员函数返回 `-1`，函数也返回 `-1`。

函数的参数包括：

- `topology`：表示输入的 `hwloc_topology_t` 对象。
- `addr`：要查找的内存位置的地址。
- `len`：存放要查找的内存位置长度的变量。
- `nodeset`：表示用于存储内存位置的节点集的指针。
- `flags`：用于传递给 `topology->binding_hooks.get_area_memlocation` 的标志，具体作用参考下文。

函数返回：

- 如果 `topology` 对象中 `binding_hooks.get_area_memlocation` 函数返回 `-1`，函数将返回该 `-1`。
- 如果 `topology` 对象中 `binding_hooks.get_area_memlocation` 函数返回 `0`，函数将返回该 `0`。
- 如果 `topology` 对象中 `binding_hooks.get_area_memlocation` 函数返回 `-1` 且 `flags` 中不包括 `HWLOC_MEMBIND_ALLFLAGS`，函数将不返回任何值。


```cpp
static int
hwloc_get_area_memlocation_by_nodeset(hwloc_topology_t topology, const void *addr, size_t len, hwloc_nodeset_t nodeset, int flags)
{
  if (flags & ~HWLOC_MEMBIND_ALLFLAGS) {
    errno = EINVAL;
    return -1;
  }

  if (!len)
    /* nothing to do */
    return 0;

  if (topology->binding_hooks.get_area_memlocation)
    return topology->binding_hooks.get_area_memlocation(topology, addr, len, nodeset, flags);

  errno = ENOSYS;
  return -1;
}

```

这段代码定义了一个名为 `hwloc_get_area_memlocation` 的函数，它用于获取定制的内存区域的定位信息。

函数接受四个参数：

- `topology`：输入 topology 类型，用于指定要获取布局范围的硬件设备。
- `addr`：目标内存地址。
- `len`：目标内存长度。
- `set`：设置，用于指定要考虑的芯片集。
- `flags`：设置，用于指定获取内存区域时使用的标志，具体来说，`HWLOC_MEMBIND_BYNODESET` 可以用于指定每个节点设置。

函数内部包含以下步骤：

1. 如果设置了 `HWLOC_MEMBIND_BYNODESET`，函数首先尝试使用 `hwloc_get_area_memlocation_by_nodeset` 函数，传递 `topology`、`addr`、`len` 和 `set` 参数，以及任何传递给 `hwloc_get_area_memlocation_by_nodeset` 的自定义标志 `flags`。
2. 如果 `hwloc_get_area_memlocation_by_nodeset` 函数成功，函数将返回结果。
3. 如果 `hwloc_get_area_memlocation_by_nodeset` 函数失败，函数将创建一个节点集，并尝试使用 `hwloc_get_area_memlocation_by_nodeset` 函数，传递 `topology`、`addr`、`len`、`nodeset` 和 `flags` 参数，以及传递给 `hwloc_get_area_memlocation_by_nodeset` 的自定义标志 `flags`。如果 `hwloc_get_area_memlocation_by_nodeset` 函数成功，则使用 `hwloc_bitmap_alloc` 函数分配内存，并返回结果。如果 `hwloc_get_area_memlocation_by_nodeset` 函数失败，函数将释放内存并尝试使用 `hwloc_cpuset_from_nodeset` 函数，传递 `topology`、`set` 和 `nodeset` 参数，以及传递给 `hwloc_get_area_memlocation_by_nodeset` 的自定义标志 `flags`。如果 `hwloc_cpuset_from_nodeset` 函数成功，则使用 `hwloc_bitmap_free` 函数释放内存，并返回结果。


```cpp
int
hwloc_get_area_memlocation(hwloc_topology_t topology, const void *addr, size_t len, hwloc_cpuset_t set, int flags)
{
  int ret;

  if (flags & HWLOC_MEMBIND_BYNODESET) {
    ret = hwloc_get_area_memlocation_by_nodeset(topology, addr, len, set, flags);
  } else {
    hwloc_nodeset_t nodeset = hwloc_bitmap_alloc();
    ret = hwloc_get_area_memlocation_by_nodeset(topology, addr, len, nodeset, flags);
    if (!ret)
      hwloc_cpuset_from_nodeset(topology, set, nodeset);
    hwloc_bitmap_free(nodeset);
  }

  return ret;
}

```

这段代码定义了一个名为 `hwloc_alloc_heap` 的函数，它属于 `hwloc_topology_t` 类型的参数 `topology` 和 `len`。

函数的作用是在给定的 `topology` 和 `len` 参数下，从堆内存中分配一块内存并返回该内存的指针。

函数实现的主要步骤如下：

1. 初始化 `p` 变量为 `NULL`。
2. 如果 `topology` 参数中定义了 `hwloc_getpagesize` 函数，并且 `HAVE_POSIX_MEMALIGN` 条件也满足，那么函数将尝试从页内存布局中分配一块指定大小的内存，并返回该内存的地址。
3. 如果 `topology` 参数中定义了 `hwloc_getpagesize` 函数，但是 `HAVE_MEMALIGN` 条件不满足，那么函数将尝试从堆内存中分配一块指定大小的内存，并返回该内存的地址。
4. 如果 `topology` 参数中没有定义 `hwloc_getpagesize` 函数，或者 `HAVE_MEMALIGN` 条件不满足，那么函数将尝试从堆内存中分配一块指定大小的内存，并返回该内存的地址。
5. 无论哪种情况，如果函数成功分配内存并返回指针，则将其返回。如果分配内存失败，则返回 `NULL`。


```cpp
void *
hwloc_alloc_heap(hwloc_topology_t topology __hwloc_attribute_unused, size_t len)
{
  void *p = NULL;
#if defined(hwloc_getpagesize) && defined(HAVE_POSIX_MEMALIGN)
  errno = posix_memalign(&p, hwloc_getpagesize(), len);
  if (errno)
    p = NULL;
#elif defined(hwloc_getpagesize) && defined(HAVE_MEMALIGN)
  p = memalign(hwloc_getpagesize(), len);
#else
  p = malloc(len);
#endif
  return p;
}

```

这段代码定义了两个函数，名为`hwloc_alloc_mmap`和`hwloc_free_heap`。它们属于`hwloc_topology_t`类型的函数，可能用于在`hwloc_topology_t`结构中进行内存分配和释放。

`hwloc_alloc_mmap`函数的实现如下：
```cppc
void *
hwloc_alloc_mmap(hwloc_topology_t topology __hwloc_attribute_unused, size_t len)
{
 void * buffer = mmap(NULL, len, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
 return buffer == MAP_FAILED ? NULL : buffer;
}
```
这个函数接收一个`hwloc_topology_t`类型的参数`topology`，以及一个表示内存分配的整数`len`。它首先调用`mmap`函数，这个函数将一个空内存区域映射到指定的内存区域，并返回一个指向新分配内存的指针`buffer`。如果`mmap`函数失败，函数将返回一个`NULL`值。如果`mmap`函数成功，函数将返回`buffer`，这个指针现在可以被用来访问新分配的内存区域。

`hwloc_free_heap`函数的实现如下：
```cppc
int
hwloc_free_heap(hwloc_topology_t topology __hwloc_attribute_unused, void *addr, size_t len __hwloc_attribute_unused)
{
 free(addr);
 return 0;
}
```
这个函数接收一个`hwloc_topology_t`类型的参数`topology`，以及一个指向内存分配的指针`addr`和一个表示内存分配的整数`len`。它首先调用`free`函数，这个函数释放分配的内存区域。然后，函数返回一个`0`，表示`free`函数成功释放内存。

这两个函数可以用于管理内存分配和释放，它们在`hwloc_topology_t`结构中的作用需要根据具体情况进行分析。


```cpp
#ifdef MAP_ANONYMOUS
void *
hwloc_alloc_mmap(hwloc_topology_t topology __hwloc_attribute_unused, size_t len)
{
  void * buffer = mmap(NULL, len, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
  return buffer == MAP_FAILED ? NULL : buffer;
}
#endif

int
hwloc_free_heap(hwloc_topology_t topology __hwloc_attribute_unused, void *addr, size_t len __hwloc_attribute_unused)
{
  free(addr);
  return 0;
}

```

这段代码定义了两个函数：`hwloc_free_mmap` 和 `hwloc_alloc`。它们属于一个名为 `hwloc_topology_device_t` 的数据结构类型，可能用于在嵌入式系统中的硬件资源上分配内存。

这两个函数的具体作用如下：

1. `hwloc_free_mmap` 函数的参数 `topology` 是一个 `hwloc_topology_t` 类型的数据结构，它表示嵌入式系统中的硬件资源，但它在函数内部没有被使用。这个函数的作用是检查传入的 `addr` 参数是否为 `NULL`，如果是，就返回 0。否则，这个函数调用另一个函数 `munmap`，它将 `addr` 指向的内存区域取消映射。

2. `hwloc_alloc` 函数的参数 `topology` 是一个 `hwloc_topology_t` 类型的数据结构，它表示嵌入式系统中的硬件资源，并且它包含一个名为 `binding_hooks.alloc` 的函数。这个函数的作用是在不使用 `binding_hooks.alloc` 函数的情况下，从系统内存中分配一块指定长度的内存，并返回它。如果 `topology` 中的 `binding_hooks.alloc` 函数可用，那么它将首先尝试从硬件资源（如内存）分配内存，如果失败，就尝试从系统内存分配内存。

这两个函数一起工作，使得在 `hwloc_topology_device_t` 类型的数据结构中，可以使用 `hwloc_free_mmap` 函数来释放使用 `hwloc_topology_device_t` 分配的内存，并使用 `hwloc_alloc` 函数来分配未被分配的内存。


```cpp
#ifdef MAP_ANONYMOUS
int
hwloc_free_mmap(hwloc_topology_t topology __hwloc_attribute_unused, void *addr, size_t len)
{
  if (!addr)
    return 0;
  return munmap(addr, len);
}
#endif

void *
hwloc_alloc(hwloc_topology_t topology, size_t len)
{
  if (topology->binding_hooks.alloc)
    return topology->binding_hooks.alloc(topology, len);
  return hwloc_alloc_heap(topology, len);
}

```

这段代码定义了一个名为 `hwloc_alloc_membind_by_nodeset` 的函数，它是 `hwloc_topology_t` 结构体的指针，用于分配内存并绑定到指定的 `HWLOC_MEMBIND_POLICY` 策略上。

函数的参数包括：

- `topology`：要绑定的 `HWLOC_topology_t` 结构体；
- `len`：要分配的内存长度；
- `nodeset`：要绑定的 `HWLOC_const_nodeset_t` 结构体；
- `policy`：用于内存绑定的策略，可以是 `HWLOC_MEMBIND_POLICY_TRIVILEG`、`HWLOC_MEMBIND_POLICY_SET_AREA` 或 `HWLOC_MEMBIND_POLICY_SET_PARTITION` 之一；
- `flags`：用于标志的值，其中 `HWLOC_MEMBIND_ALLFLAGS` 是保留标志，用于指示是否包括所有分配和释放操作。

函数首先检查 flags 和 `hwloc__check_membind_policy` 函数的返回值，如果它们都返回负数，则错误处理。否则，函数将尝试使用 `hwloc_fix_membind` 函数将指定的 `nodeset` 节点设置为绑定策略的内存范围。如果 `nodeset` 不能分配内存，函数跳转到 `fallback` 标签，尝试执行下一次内存分配。

如果 `topology->binding_hooks.alloc_membind` 函数仍然不能分配内存，或者 `topology->binding_hooks.set_area_membind` 函数尝试设置内存区域，则函数继续执行内存分配操作，并设置相应的策略标志。如果设置的区域策略标记 `HWLOC_MEMBIND_SET_AREA`，则函数在分配内存后，尝试设置 `HWLOC_MEMBIND_SET_PARTITION` 策略标志，以便在需要时可以设置区域。如果设置的所有策略标志都为真，则函数错误处理并返回 NULL。


```cpp
static void *
hwloc_alloc_membind_by_nodeset(hwloc_topology_t topology, size_t len, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags)
{
  void *p;

  if ((flags & ~HWLOC_MEMBIND_ALLFLAGS) || hwloc__check_membind_policy(policy) < 0) {
    errno = EINVAL;
    return NULL;
  }

  nodeset = hwloc_fix_membind(topology, nodeset);
  if (!nodeset)
    goto fallback;
  if (flags & HWLOC_MEMBIND_MIGRATE) {
    errno = EINVAL;
    goto fallback;
  }

  if (topology->binding_hooks.alloc_membind)
    return topology->binding_hooks.alloc_membind(topology, len, nodeset, policy, flags);
  else if (topology->binding_hooks.set_area_membind) {
    p = hwloc_alloc(topology, len);
    if (!p)
      return NULL;
    if (topology->binding_hooks.set_area_membind(topology, p, len, nodeset, policy, flags) && flags & HWLOC_MEMBIND_STRICT) {
      int error = errno;
      free(p);
      errno = error;
      return NULL;
    }
    return p;
  } else {
    errno = ENOSYS;
  }

```



这段代码定义了 `hwloc_alloc_membind` 函数，用于将 `hwloc_membind_policy_t` 类型的标志与 `HWLOC_MEMBIND_STRICT` 标志组合起来，然后根据不同的标志来决定如何分配内存。

函数接收四个参数：

- `topology`：表示底层硬件平台上的 `hwloc_topology_t` 类型，用于指定要使用的硬件资源。
- `len`：要分配的内存大小。
- `set`：指定内存中哪些硬件资源被绑定，是一个 `hwloc_const_bitmap_t` 类型的标志，用于指定哪些硬件资源不被绑定。
- `policy`:`hwloc_membind_policy_t` 类型的标志，用于指定分配内存时使用的策略。可以包括 `HWLOC_MEMBIND_BYNODESET` 和 `HWLOC_MEMBIND_STRICT` 标志。
- `flags`：一个额外的标志，用于指定其他需要报告的错误。

函数首先检查 `flags` 中是否包含 `HWLOC_MEMBIND_STRICT` 标志，如果是，那么函数将返回 `NULL`，表示无法分配内存。否则，函数将根据 `policy` 指定的人口径来尝试分配内存。如果 `policy` 中包含 `HWLOC_MEMBIND_BYNODESET` 标志，那么函数将尝试使用 `hwloc_alloc_membind_by_nodeset` 函数来分配内存。如果 `policy` 中包含 `HWLOC_MEMBIND_STRICT` 标志，那么函数将使用 `hwloc_alloc` 函数来分配内存。无论哪种情况，如果分配内存成功，函数将返回分配的内存的地址。如果出现错误，函数将返回 `NULL`。


```cpp
fallback:
  if (flags & HWLOC_MEMBIND_STRICT)
    /* Report error */
    return NULL;
  /* Never mind, allocate anyway */
  return hwloc_alloc(topology, len);
}

void *
hwloc_alloc_membind(hwloc_topology_t topology, size_t len, hwloc_const_bitmap_t set, hwloc_membind_policy_t policy, int flags)
{
  void *ret;

  if (flags & HWLOC_MEMBIND_BYNODESET) {
    ret = hwloc_alloc_membind_by_nodeset(topology, len, set, policy, flags);
  } else {
    hwloc_nodeset_t nodeset = hwloc_bitmap_alloc();
    if (hwloc_fix_membind_cpuset(topology, nodeset, set)) {
      if (flags & HWLOC_MEMBIND_STRICT)
	ret = NULL;
      else
	ret = hwloc_alloc(topology, len);
    } else
      ret = hwloc_alloc_membind_by_nodeset(topology, len, nodeset, policy, flags);
    hwloc_bitmap_free(nodeset);
  }

  return ret;
}

```



这段代码定义了两个函数，第一个函数 `hwloc_free` 和第二个函数 `dontset_return_complete_cpuset`。

函数 `hwloc_free` 的作用是释放一个指向内存的指针 `addr` 所指向的内存区域，如果 `topology` 结构中定义了 `binding_hooks.free_membind` 函数，则会调用这个函数，否则会调用 `hwloc_free_heap` 函数。`hwloc_free_heap` 函数从 `topology` 结构中绑定的堆中释放内存。

函数 `dontset_return_complete_cpuset` 的作用是设置 `set` 所指向的 CPU 集为 `topology` 结构中定义的完全 CPU 集，并将 `set` 所指向的 CPU 集与 `topology_get_complete_cpuset` 函数返回的 CPU 集进行复制。如果 `topology` 结构中没有定义 `binding_hooks.complete_cpuset_hook` 函数，则不会执行该函数。

这两个函数都是 HWLOC 库中的绑定钩子函数，用于管理在 DMA 操作中指定哪些 CPU 集应该被用于绑定到内存上。`hwloc_free` 函数用于释放指定 CPU 集上的内存，`dontset_return_complete_cpuset` 函数用于设置指定 CPU 集中的 CPU 集，以便在 DMA 操作中使用。


```cpp
int
hwloc_free(hwloc_topology_t topology, void *addr, size_t len)
{
  if (topology->binding_hooks.free_membind)
    return topology->binding_hooks.free_membind(topology, addr, len);
  return hwloc_free_heap(topology, addr, len);
}

/*
 * Empty binding hooks always returning success
 */

static int dontset_return_complete_cpuset(hwloc_topology_t topology, hwloc_cpuset_t set)
{
  hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
  return 0;
}

```

这三段代码定义了三个名为 "dontset_thisthread_cpubind" 和 "dontget_thisthread_cpubind" 的函数，它们的函数名都为 "int"，返回类型也都为 "int"。

这些函数的主要作用是执行与 "set" 函数相关的设置或获取操作。"set" 函数的参数是一个 "hwloc_const_bitmap_t" 类型的 "set"，表示要设置的共享内存区域，可以通过输出该参数来指定哪些核心对应的内存区域需要被设置为该值。而 "dontset_thisthread_cpubind" 和 "dontget_thisthread_cpubind" 函数的作用，就是分别执行设置和获取操作，并返回相应的结果。

在 "dontset_thisthread_cpubind" 函数中，如果通过 "set" 函数设置的共享内存区域被其他进程或线程访问，那么这个函数会直接返回 0，不会执行 "dontset_thisproc_cpubind" 函数中可能存在的逻辑。

在 "dontget_thisthread_cpubind" 函数中，如果通过 "get" 函数获取的共享内存区域被其他进程或线程访问，那么这个函数会执行 "dontset_thisproc_cpubind" 函数中可能存在的逻辑，以避免进一步的并发问题。不过，需要注意的是，在这个函数中，如果设置的共享内存区域没有被其他进程或线程访问，那么它的返回值也不会有影响。


```cpp
static int dontset_thisthread_cpubind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_const_bitmap_t set __hwloc_attribute_unused, int flags __hwloc_attribute_unused)
{
  return 0;
}
static int dontget_thisthread_cpubind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_bitmap_t set, int flags __hwloc_attribute_unused)
{
  return dontset_return_complete_cpuset(topology, set);
}
static int dontset_thisproc_cpubind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_const_bitmap_t set __hwloc_attribute_unused, int flags __hwloc_attribute_unused)
{
  return 0;
}
static int dontget_thisproc_cpubind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_bitmap_t set, int flags __hwloc_attribute_unused)
{
  return dontset_return_complete_cpuset(topology, set);
}
```

这两个函数是 `dontset_proc_cpubind` 和 `dontget_proc_cpubind` 的别名，它们都接受一个 `hwloc_topology_t` 类型的局部总线布局和一个 `hwloc_pid_t` 或 `hwloc_const_bitmap_t` 类型的设置，并返回一个表示操作结果的整数。这两个函数的主要作用是设置或获取 `cpu` 集合的值，并将设置的结果返回。

`dontset_proc_cpubind` 函数接受一个 `hwloc_topology_t`，设置了一个 `hwloc_const_bitmap_t`，然后返回 0。如果设置成功，则返回 0，否则返回一个非零值。 `dontget_proc_cpubind` 函数与 `dontset_proc_cpubind` 相反，它接受一个 `hwloc_topology_t`，设置了一个 `hwloc_const_bitmap_t`，然后返回 `dontset_proc_cpubind` 的返回值。 `dontget_thread_cpubind` 函数也与 `dontset_proc_cpubind` 相反，它接受一个 `hwloc_topology_t`，设置了一个 `hwloc_const_bitmap_t`，并返回 `dontget_proc_cpubind` 的返回值。


```cpp
static int dontset_proc_cpubind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_pid_t pid __hwloc_attribute_unused, hwloc_const_bitmap_t set __hwloc_attribute_unused, int flags __hwloc_attribute_unused)
{
  return 0;
}
static int dontget_proc_cpubind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_pid_t pid __hwloc_attribute_unused, hwloc_bitmap_t cpuset, int flags __hwloc_attribute_unused)
{
  return dontset_return_complete_cpuset(topology, cpuset);
}
#ifdef hwloc_thread_t
static int dontset_thread_cpubind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_thread_t tid __hwloc_attribute_unused, hwloc_const_bitmap_t set __hwloc_attribute_unused, int flags __hwloc_attribute_unused)
{
  return 0;
}
static int dontget_thread_cpubind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_thread_t tid __hwloc_attribute_unused, hwloc_bitmap_t cpuset, int flags __hwloc_attribute_unused)
{
  return dontset_return_complete_cpuset(topology, cpuset);
}
```

这段代码定义了两个函数：dontset_return_complete_nodeset和dontset_thisproc_membind。它们都属于一个名为dontset_complete_nodeset的函数。函数名都包含“dontset_”前缀，表明它们是用于设置或获取内存绑定设置的函数。

dontset_return_complete_nodeset函数的实现要点如下：

1. 首先，将传入的节点集与topology的完整节点集进行复制，使用hwloc_bitmap_copy函数实现。

2. 将policy设置为HWLOC_MEMBIND_MIXED，这个设置表示允许函数指针引用到的内存区域。

3. 函数返回0，表示成功完成了设置。

dontset_thisproc_membind函数的实现要点如下：

1. 首先，使用hwloc_topology_get_complete_nodeset函数获取topology的完整节点集，并将其与传入的节点集进行比较，使用hwloc_bitmap_compare函数。

2. 如果两个节点集相等，则说明函数设置成功，返回0。

3. 如果两个节点集不相等，则设置失败，返回-1，需要进行错误处理。

4. 在函数内部，使用dontset_return_complete_nodeset函数处理这个问题，具体是传入topology和set，返回0。


```cpp
#endif

static int dontset_return_complete_nodeset(hwloc_topology_t topology, hwloc_nodeset_t set, hwloc_membind_policy_t *policy)
{
  hwloc_bitmap_copy(set, hwloc_topology_get_complete_nodeset(topology));
  *policy = HWLOC_MEMBIND_MIXED;
  return 0;
}

static int dontset_thisproc_membind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_const_bitmap_t set __hwloc_attribute_unused, hwloc_membind_policy_t policy __hwloc_attribute_unused, int flags __hwloc_attribute_unused)
{
  return 0;
}
static int dontget_thisproc_membind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_bitmap_t set, hwloc_membind_policy_t * policy, int flags __hwloc_attribute_unused)
{
  return dontset_return_complete_nodeset(topology, set, policy);
}

```

这三段代码定义了三个名为 "dontset_thisthread_membind" 的函数，它们的参数列表相同，但返回值不同。函数声明使用 "hwloc_topology_t"、"hwloc_const_bitmap_t" 和 "hwloc_membind_policy_t" 作为参数，这表明它们的使用者需要定义 "hwloc" 系列的函数。

"dontset_thisthread_membind" 函数表示在 "dontset_thisthread_membind" 函数中，任何线程级的 "hwloc_topology_t" 对象和内存布局以及 "hwloc_const_bitmap_t" 和 "hwloc_membind_policy_t" 都未被定义时，返回 0。

"dontget_thisthread_membind" 函数与 "dontset_thisthread_membind" 函数具有相同的签名，但它们的实现是相反的：它表示在 "dontget_thisthread_membind" 函数中，任何线程级的 "hwloc_topology_t" 对象和内存布局以及 "hwloc_const_bitmap_t" 和 "hwloc_membind_policy_t" 都被定义时，返回值是成功设置的 bitmap 对应的线程级 "hwloc_pid_t" 对象的 ID。

"dontset_proc_membind" 和 "dontget_proc_membind" 函数与 "dontset_thisthread_membind" 和 "dontget_thisthread_membind" 函数的实现完全相同，只是它们的输入参数列表略有不同：它们分别需要输入 "hwloc_topology_t"、"hwloc_const_bitmap_t" 和 "hwloc_membind_policy_t"，以及输入和输出各不相同。


```cpp
static int dontset_thisthread_membind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_const_bitmap_t set __hwloc_attribute_unused, hwloc_membind_policy_t policy __hwloc_attribute_unused, int flags __hwloc_attribute_unused)
{
  return 0;
}
static int dontget_thisthread_membind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_bitmap_t set, hwloc_membind_policy_t * policy, int flags __hwloc_attribute_unused)
{
  return dontset_return_complete_nodeset(topology, set, policy);
}

static int dontset_proc_membind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_pid_t pid __hwloc_attribute_unused, hwloc_const_bitmap_t set __hwloc_attribute_unused, hwloc_membind_policy_t policy __hwloc_attribute_unused, int flags __hwloc_attribute_unused)
{
  return 0;
}
static int dontget_proc_membind(hwloc_topology_t topology __hwloc_attribute_unused, hwloc_pid_t pid __hwloc_attribute_unused, hwloc_bitmap_t set, hwloc_membind_policy_t * policy, int flags __hwloc_attribute_unused)
{
  return dontset_return_complete_nodeset(topology, set, policy);
}

```



这段代码定义了两个名为 `dontset_area_membind` 和 `dontget_area_membind` 的函数，它们属于 `hwloc_membind_policy_t` 类型，用于管理内存绑定策略。

这两个函数的参数列表几乎完全相同，只是在参数类型上有一些微调，主要是将 `hwloc_topology_t` 类型的参数转换为 `const void *` 类型，以便在函数内部进行类型检查。

函数实现主要分为两部分：

1. 在函数声明之前，定义了一系列常量，包括 `dontset_area_membind` 和 `dontget_area_membind` 函数名称，以及 `hwloc_membind_policy_t` 类型的参数。

2. 在函数体内部，通过调用 `dontset_return_complete_nodeset` 函数，将指定的 `hwloc_topology_t` 和 `hwloc_const_bitmap_t` 对象设置为membind策略，并将结果返回。

这里需要指出的是， `dontset_area_membind` 和 `dontget_area_membind` 函数的实现几乎完全相同，只是在参数列表上稍微调整了一些。由于它们的参数列表几乎完全相同，因此只需要定义其中一个函数，另一个将 `dontget_area_membind` 作为 `dontset_area_membind` 的别名来实现即可。


```cpp
static int dontset_area_membind(hwloc_topology_t topology __hwloc_attribute_unused, const void *addr __hwloc_attribute_unused, size_t size __hwloc_attribute_unused, hwloc_const_bitmap_t set __hwloc_attribute_unused, hwloc_membind_policy_t policy __hwloc_attribute_unused, int flags __hwloc_attribute_unused)
{
  return 0;
}
static int dontget_area_membind(hwloc_topology_t topology __hwloc_attribute_unused, const void *addr __hwloc_attribute_unused, size_t size __hwloc_attribute_unused, hwloc_bitmap_t set, hwloc_membind_policy_t * policy, int flags __hwloc_attribute_unused)
{
  return dontset_return_complete_nodeset(topology, set, policy);
}
static int dontget_area_memlocation(hwloc_topology_t topology __hwloc_attribute_unused, const void *addr __hwloc_attribute_unused, size_t size __hwloc_attribute_unused, hwloc_bitmap_t set, int flags __hwloc_attribute_unused)
{
  hwloc_membind_policy_t policy;
  return dontset_return_complete_nodeset(topology, set, &policy);
}

static void * dontalloc_membind(hwloc_topology_t topology __hwloc_attribute_unused, size_t size __hwloc_attribute_unused, hwloc_const_bitmap_t set __hwloc_attribute_unused, hwloc_membind_policy_t policy __hwloc_attribute_unused, int flags __hwloc_attribute_unused)
{
  return malloc(size);
}
```



这段代码定义了一个名为 `dontfree_membind` 的函数，其作用是释放传入的 `addr` 所指向的内存，并返回 0。

该函数被用于 `hwloc_topology_t` 类型的 `topology` 和 `void` 类型的 `addr` 和 `size_t` 类型的 `size` 作为参数。这意味着该函数可以被用于任何 `hwloc_topology_t` 类型的 topology 对象中，只要该对象中包含了一个 `void` 类型的指针和一个大小为 `size_t` 的成员变量。

函数体中首先通过调用 `free` 函数释放了 `addr` 所指向的内存，然后返回 0。

接下来，定义了一个名为 `hwloc_set_dummy_hooks` 的函数，该函数接受一个 `struct hwloc_binding_hooks` 类型的参数，一个 `struct hwloc_topology_support` 类型的 `support` 参数和一个空括号。函数的作用是设置钩子，以使 `set_thisproc_cpubind`, `get_thisproc_cpubind`, `set_thisthread_cpubind`, `get_thisthread_cpubind`, `set_proc_cpubind` 和 `get_proc_cpubind` 函数不再接受前缀 `__hwloc_attribute_unused` 的参数。

函数首先设置 `hooks->set_thisproc_cpubind` 和 `hooks->get_thisproc_cpubind` 函数不再接受前缀 `__hwloc_attribute_unused` 的参数，这意味着该函数将不再接受这两个函数的参数。

然后设置 `hooks->set_thisthread_cpubind` 和 `hooks->get_thisthread_cpubind` 函数不再接受前缀 `__hwloc_attribute_unused` 的参数。

接下来设置 `hooks->set_proc_cpubind` 和 `hooks->get_proc_cpubind` 函数不再接受前缀 `__hwloc_attribute_unused` 的参数。

最后设置 `hooks->get_thisthread_cpubind` 和 `hooks->get_proc_cpubind` 函数接受前缀 `__hwloc_attribute_unused` 的参数。


```cpp
static int dontfree_membind(hwloc_topology_t topology __hwloc_attribute_unused, void *addr __hwloc_attribute_unused, size_t size __hwloc_attribute_unused)
{
  free(addr);
  return 0;
}

static void hwloc_set_dummy_hooks(struct hwloc_binding_hooks *hooks,
				  struct hwloc_topology_support *support __hwloc_attribute_unused)
{
  hooks->set_thisproc_cpubind = dontset_thisproc_cpubind;
  hooks->get_thisproc_cpubind = dontget_thisproc_cpubind;
  hooks->set_thisthread_cpubind = dontset_thisthread_cpubind;
  hooks->get_thisthread_cpubind = dontget_thisthread_cpubind;
  hooks->set_proc_cpubind = dontset_proc_cpubind;
  hooks->get_proc_cpubind = dontget_proc_cpubind;
```

这段代码是用来定义 `hooks` 结构体中的几个成员变量的。具体来说：

1. `#ifdef hwloc_thread_t` 是一个预处理指令，它会在编译时将 `hwloc_thread_t` 定义展开为以下内容： `hooks->set_thread_cpubind = 0; hooks->get_thread_cpubind = 0;`  
2. `#endif` 是一个预处理指令，它会在编译时将 `#ifdef` 和 `#endif` 之间的内容作为注释，不会被执行。
3. `hooks->get_thisproc_last_cpu_location` 和 `hooks->get_thisthread_last_cpu_location` 函数用于获取当前进程或线程的最后一个 CPU 位置。其中，`hooks->get_thisproc_last_cpu_location` 是获取进程的最后一个 CPU 位置，而 `hooks->get_thisthread_last_cpu_location` 是获取线程的最后一个 CPU 位置。这两个函数都使用了 `dontget_<cpu_position>` 函数作为代替，其中 `<cpu_position>` 是 `hooks` 结构体中要获取的 CPU 位置。
4. `hooks->get_proc_last_cpu_location` 函数用于获取进程的最后一个 CPU 位置。它使用了与 `hooks->get_thisproc_last_cpu_location` 类似的函数 `hooks->get_thisproc_cpubind` 的别称 `hooks->get_proc_last_cpu_location`。
5. `hooks->get_thisthread_membind` 和 `hooks->get_thisthread_memlocation` 函数用于获取线程的内存绑定位置。其中，`hooks->get_thisthread_membind` 是获取线程的第一个内存绑定位置，而 `hooks->get_thisthread_memlocation` 是获取线程的最后一个内存绑定位置。这两个函数都使用了 `dontget_<membind_position>` 函数作为代替，其中 `<membind_position>` 是 `hooks` 结构体中要获取的内存绑定位置。
6. `hooks->free_membind` 和 `hooks->allocate_membind` 函数用于设置或取消内存绑定。其中，`hooks->free_membind` 是释放内存绑定，而 `hooks->allocate_membind` 是分配内存绑定。


```cpp
#ifdef hwloc_thread_t
  hooks->set_thread_cpubind = dontset_thread_cpubind;
  hooks->get_thread_cpubind = dontget_thread_cpubind;
#endif
  hooks->get_thisproc_last_cpu_location = dontget_thisproc_cpubind; /* cpubind instead of last_cpu_location is ok */
  hooks->get_thisthread_last_cpu_location = dontget_thisthread_cpubind; /* cpubind instead of last_cpu_location is ok */
  hooks->get_proc_last_cpu_location = dontget_proc_cpubind; /* cpubind instead of last_cpu_location is ok */
  /* TODO: get_thread_last_cpu_location */
  hooks->set_thisproc_membind = dontset_thisproc_membind;
  hooks->get_thisproc_membind = dontget_thisproc_membind;
  hooks->set_thisthread_membind = dontset_thisthread_membind;
  hooks->get_thisthread_membind = dontget_thisthread_membind;
  hooks->set_proc_membind = dontset_proc_membind;
  hooks->get_proc_membind = dontget_proc_membind;
  hooks->set_area_membind = dontset_area_membind;
  hooks->get_area_membind = dontget_area_membind;
  hooks->get_area_memlocation = dontget_area_memlocation;
  hooks->alloc_membind = dontalloc_membind;
  hooks->free_membind = dontfree_membind;
}

```

这段代码定义了一个名为 `hwloc_set_native_binding_hooks` 的函数，其作用是设置特定的钩子(hooks)以支持特定操作系统(如 Linux、BGQ 或 AIX)。

具体来说，函数首先检查操作系统是否为 Linux，如果是，则执行 `hwloc_set_linuxfs_hooks` 函数，如果不是，则根据操作系统类型执行其他钩子设置函数。接下来，函数根据需要设置针对 BGQ 和 AIX 的钩子。

这段代码的作用是确保在 hwloc 中定义的所有钩子(hooks)都可以在相应的操作系统上正常工作，并为特定的操作系统定义了钩子函数，使得 hwloc 可以正确地初始化和配置系统。


```cpp
void
hwloc_set_native_binding_hooks(struct hwloc_binding_hooks *hooks, struct hwloc_topology_support *support)
{
#    ifdef HWLOC_LINUX_SYS
    hwloc_set_linuxfs_hooks(hooks, support);
#    endif /* HWLOC_LINUX_SYS */

#    ifdef HWLOC_BGQ_SYS
    hwloc_set_bgq_hooks(hooks, support);
#    endif /* HWLOC_BGQ_SYS */

#    ifdef HWLOC_AIX_SYS
    hwloc_set_aix_hooks(hooks, support);
#    endif /* HWLOC_AIX_SYS */

```

这段代码的作用是检查系统是否支持HWLOC_SOLARIS_SYS、HWLOC_WIN_SYS、HWLOC_DARWIN_SYS和HWLOC_FREEBSD_SYS。如果系统支持指定的SYS，则将`hooks`和`support`设置为真，否则将它们设置为假。

具体来说，这段代码的作用如下：

1. 如果系统支持HWLOC_SOLARIS_SYS，则设置`hooks`为真，并将`support`设置为真。
2. 如果系统不支持HWLOC_SOLARIS_SYS，则将`hooks`和`support`都设置为假。
3. 如果系统支持HWLOC_WIN_SYS，则设置`hooks`为真，并将`support`设置为真。
4. 如果系统不支持HWLOC_WIN_SYS，则将`hooks`和`support`都设置为假。
5. 如果系统支持HWLOC_DARWIN_SYS，则设置`hooks`为真，并将`support`设置为真。
6. 如果系统不支持HWLOC_DARWIN_SYS，则将`hooks`和`support`都设置为假。
7. 如果系统支持HWLOC_FREEBSD_SYS，则设置`hooks`为真，并将`support`设置为真。
8. 如果系统不支持HWLOC_FREEBSD_SYS，则将`hooks`和`support`都设置为假。


```cpp
#    ifdef HWLOC_SOLARIS_SYS
    hwloc_set_solaris_hooks(hooks, support);
#    endif /* HWLOC_SOLARIS_SYS */

#    ifdef HWLOC_WIN_SYS
    hwloc_set_windows_hooks(hooks, support);
#    endif /* HWLOC_WIN_SYS */

#    ifdef HWLOC_DARWIN_SYS
    hwloc_set_darwin_hooks(hooks, support);
#    endif /* HWLOC_DARWIN_SYS */

#    ifdef HWLOC_FREEBSD_SYS
    hwloc_set_freebsd_hooks(hooks, support);
#    endif /* HWLOC_FREEBSD_SYS */

```

这段代码的作用是设置硬件抽象层（HWLOC）中的网络适配器（netbsd）和硬件处理器单元（HPUX）的钩子（hooks），以便在系统初始化时进行自检并提供支持。

具体来说，如果代表当前系统的操作系统支持HWLOC_NETBSD_SYS和HWLOC_HPUX_SYS，那么代码会设置这两个操作系统对应的netbsd和HPUX钩子。如果当前系统既不支持这两个操作系统，也不会创建它们，那么就会设置一些dummy binding hooks，这些钩子不会做任何实际的检查，但可以让代码通过编译。

另外，如果当前系统是HWLOC_NETBSD_SYS或HWLOC_HPUX_SYS，但系统本身并不是这两个操作系统之一，那么就会设置一些dummy binding hooks来完成设置。


```cpp
#    ifdef HWLOC_NETBSD_SYS
    hwloc_set_netbsd_hooks(hooks, support);
#    endif /* HWLOC_NETBSD_SYS */

#    ifdef HWLOC_HPUX_SYS
    hwloc_set_hpux_hooks(hooks, support);
#    endif /* HWLOC_HPUX_SYS */
}

/* If the represented system is actually not this system, use dummy binding hooks. */
void
hwloc_set_binding_hooks(struct hwloc_topology *topology)
{
  if (topology->is_thissystem) {
    hwloc_set_native_binding_hooks(&topology->binding_hooks, &topology->support);
    /* every hook not set above will return ENOSYS */
  } else {
    /* not this system, use dummy binding hooks that do nothing (but don't return ENOSYS) */
    hwloc_set_dummy_hooks(&topology->binding_hooks, &topology->support);

    /* Linux has some hooks that also work in this case, but they are not strictly needed yet. */
  }

  /* if not is_thissystem, set_cpubind is fake
   * and get_cpubind returns the whole system cpuset,
   * so don't report that set/get_cpubind as supported
   */
  if (topology->is_thissystem) {
```

这段代码定义了一个名为 DO 的函数，它接受两个参数：一个可选项前缀kind，表示要定义的某个特定功能，以及一个字符串，表示输出过程中用到的钩子函数名称。

DO 函数的作用是在程序运行时，根据传入的kind选择性地执行相应的钩子函数，设置或获取系统中与kind 相关的 binding 上下文。

具体来说，DO 函数的功能如下：

1. 如果传入的kind 存在，则执行设置上下文钩子函数 topology->binding_hooks.kind 到 topology->support.which##bind->kind，这将在程序启动时初始化 CPU 线程对应的 binding 上下文。

2. 如果传入的kind 不存在，则依次执行以下操作：设置 CPU 线程对应的 binding 上下层函数为 1，设置进程对应的 binding 上下层函数为 @hwloc_set_proc_cpubind，设置上下文钩子函数为 @hwloc_set_thisthread_cpubind，设置线程对应的 binding 上下层函数为 @hwloc_set_thread_cpubind，获取线程对应的上下文钩子函数，然后执行这些钩子函数。

3. 如果上下文钩子函数均不执行，则执行以下操作：获取 CPU 线程对应的最后一个 CPU 位置，获取进程对应的最后一个 CPU 位置，获取线程对应的最后一个 CPU 位置，然后设置上下文钩子函数为 @hwloc_set_proc_last_cpu_location，设置上下文钩子函数为 @hwloc_set_proc_last_cpu_location，设置上下文钩子函数为 @hwloc_set_thisthread_last_cpu_location，设置上下文钩子函数为 @hwloc_set_thisproc_last_cpu_location，设置上下文钩子函数为 @hwloc_set_proc_last_cpu_location，设置上下文钩子函数为 @hwloc_set_thisthread_membind，设置上下文钩子函数为 @hwloc_set_membind，设置上下文钩子函数为 @hwloc_set_thisproc_membind，设置上下文钩子函数为 @hwloc_set_proc_membind，设置上下文钩子函数为 @hwloc_set_area_membind，设置上下文钩子函数为 @hwloc_alloc_membind，然后输出最后一个 CPU 位置。


```cpp
#define DO(which,kind) \
    if (topology->binding_hooks.kind) \
      topology->support.which##bind->kind = 1;
    DO(cpu,set_thisproc_cpubind);
    DO(cpu,get_thisproc_cpubind);
    DO(cpu,set_proc_cpubind);
    DO(cpu,get_proc_cpubind);
    DO(cpu,set_thisthread_cpubind);
    DO(cpu,get_thisthread_cpubind);
#ifdef hwloc_thread_t
    DO(cpu,set_thread_cpubind);
    DO(cpu,get_thread_cpubind);
#endif
    DO(cpu,get_thisproc_last_cpu_location);
    DO(cpu,get_proc_last_cpu_location);
    DO(cpu,get_thisthread_last_cpu_location);
    DO(mem,set_thisproc_membind);
    DO(mem,get_thisproc_membind);
    DO(mem,set_thisthread_membind);
    DO(mem,get_thisthread_membind);
    DO(mem,set_proc_membind);
    DO(mem,get_proc_membind);
    DO(mem,set_area_membind);
    DO(mem,get_area_membind);
    DO(mem,get_area_memlocation);
    DO(mem,alloc_membind);
```

这段代码的作用是定义了一个名为“DO”的函数，但同时也对它进行了否定定义，即未定义（undefined）。这个函数没有 body（即没有对它进行任何操作），因此它不能被调用或访问。


```cpp
#undef DO
  }
}

```