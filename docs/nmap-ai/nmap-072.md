# Nmap源码解析 72

# `libpcap/lbl/os-ultrix4.h`

这段代码是一个C语言的函数声明，它定义了一些名为“my_function”的函数。这些函数没有定义具体的参数列表，也没有定义返回类型。根据声明中的描述，这些函数将“my_func”作为参数，但未提供任何上下文。你需要在调用这些函数时，根据需要提供相应的参数。


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

这段代码是一个名为"bcmp"的函数，缺少在Ultrix 4中的prototypes。根据函数名称和参数，我猜测它可能是用来比较两个字符串是否相等的。

"bcopy"函数是一个名为void的函数，没有参数，返回值类型为void。根据函数名称，我猜测它可能是用来复制一个void类型的变量的。

"bzero"函数是一个名为void的函数，没有参数，返回值类型为void。根据函数名称，我猜测它可能是用来设置一个void类型的变量为0。

"getopt"函数是一个名为int的函数，返回值为int。根据函数名称，我猜测它是一个获取命令行参数的函数。

"gettimeofday"函数是一个名为int的函数，返回值为int。根据函数名称，我猜测它是一个获取当前时间的函数。

"ioctl"函数是一个名为int的函数，返回值为int。根据函数名称，我猜测它是一个向系统I/O控制的函数。

"pfopen"函数是一个名为int的函数，返回值为int。根据函数名称，我猜测它是一个打开文件的函数。

"setlinebuf"函数是一个名为FILE的函数，没有参数，返回值类型为FILE。根据函数名称，我猜测它是一个设置缓冲区的函数。

"socket"函数是一个名为int的函数，返回值为int。根据函数名称，我猜测它是一个创建套接字的函数。

"strcasecmp"函数是一个名为const char *的函数，没有参数，返回值类型为const char *。根据函数名称，我猜测它是一个比较两个字符串是否相等的函数。


```cpp
/* Prototypes missing in Ultrix 4 */
int	bcmp(const char *, const char *, u_int);
void	bcopy(const void *, void *, u_int);
void	bzero(void *, u_int);
int	getopt(int, char * const *, const char *);
#ifdef __STDC__
struct timeval;
struct timezone;
#endif
int	gettimeofday(struct timeval *, struct timezone *);
int	ioctl(int, int, caddr_t);
int	pfopen(char *, int);
int	setlinebuf(FILE *);
int	socket(int, int, int);
int	strcasecmp(const char *, const char *);

```

# `libpcap/missing/asprintf.c`

This is a function definition for `vsprintf()` which takes a format string, a variable number of arguments, and a buffer. The function returns the number of characters successfully written to the buffer. If the returned value is negative, the function returns -1.

The function first checks if the returned value is negative. If it is, the function sets the first argument to `NULL` and returns -1.

If the returned value is non-negative, the function mallows the buffer and assigns it to the first argument. It then initializes a pointer `str` to the newly mallowed buffer and assigns it to the first argument.

The function then calls `sprintf()` formatting function with the format string, the first argument, and the variable number of arguments. The returned value of `sprintf()` is stored in a variable `ret`.

Finally, the function checks if `ret` is non-negative. If it is, the function returns the return value. If it is not, the function frees the buffer and sets the first argument to `NULL`. The function then returns -1.


```cpp
#include <stdio.h>
#include <stdlib.h>
#include <stdarg.h>

#include "portability.h"

/*
 * vasprintf() and asprintf() for platforms with a C99-compliant
 * snprintf() - so that, if you format into a 1-byte buffer, it
 * will return how many characters it would have produced had
 * it been given an infinite-sized buffer.
 */
int
pcap_vasprintf(char **strp, const char *format, va_list args)
{
	char buf;
	int len;
	size_t str_size;
	char *str;
	int ret;

	/*
	 * XXX - the C99 standard says, in section 7.19.6.5 "The
	 * nprintf function":
	 *
	 *    The snprintf function is equivalent to fprintf, except that
	 *    the output is written into an array (specified by argument s)
	 *    rather than to a stream.  If n is zero, nothing is written,
	 *    and s may be a null pointer.  Otherwise, output characters
	 *    beyond the n-1st are discarded rather than being written
	 *    to the array, and a null character is written at the end
	 *    of the characters actually written into the array.
	 *
	 *        ...
	 *
	 *    The snprintf function returns the number of characters that
	 *    would have been written had n been sufficiently large, not
	 *    counting the terminating null character, or a negative value
	 *    if an encoding error occurred. Thus, the null-terminated
	 *    output has been completely written if and only if the returned
	 *    value is nonnegative and less than n.
	 *
	 * That doesn't make it entirely clear whether, if a null buffer
	 * pointer and a zero count are passed, it will return the number
	 * of characters that would have been written had a buffer been
	 * passed.
	 *
	 * And, even if C99 *does*, in fact, say it has to work, it
	 * doesn't work in Solaris 8, for example - it returns -1 for
	 * NULL/0, but returns the correct character count for a 1-byte
	 * buffer.
	 *
	 * So we pass a one-character pointer in order to find out how
	 * many characters this format and those arguments will need
	 * without actually generating any more of those characters
	 * than we need.
	 *
	 * (The fact that it might happen to work with GNU libc or with
	 * various BSD libcs is completely uninteresting, as those tend
	 * to have asprintf() already and thus don't even *need* this
	 * code; this is for use in those UN*Xes that *don't* have
	 * asprintf().)
	 */
	len = vsnprintf(&buf, sizeof buf, format, args);
	if (len == -1) {
		*strp = NULL;
		return (-1);
	}
	str_size = len + 1;
	str = malloc(str_size);
	if (str == NULL) {
		*strp = NULL;
		return (-1);
	}
	ret = vsnprintf(str, str_size, format, args);
	if (ret == -1) {
		free(str);
		*strp = NULL;
		return (-1);
	}
	*strp = str;
	/*
	 * vsnprintf() shouldn't truncate the string, as we have
	 * allocated a buffer large enough to hold the string, so its
	 * return value should be the number of characters written.
	 */
	return (ret);
}

```

这段代码是一个名为 `pcap_asprintf` 的函数，它是用来将一个 `const char*` 类型的格式字符串和多个 `int` 类型的参数进行字符串拼接，并将结果存储到一个指向 `char*` 类型的指针变量 `strp` 中。

具体来说，这段代码的作用如下：

1. 接受一个 `const char*` 类型的格式字符串 `format` 和多个 `int` 类型的参数；
2. 将 `format` 中的字符串和参数列表（也就是 `args` 变量）拼接成一个 `const char*` 类型的字符数组；
3. 使用 `pcap_vasprintf` 函数将格式字符串和参数列表进行字符串拼接，并将结果存储到 `strp` 指向的内存区域；
4. 返回 `ret` 变量，即拼接的结果；
5. `va_start` 函数用于将参数列表传递给 `pcap_vasprintf` 函数，以便正确地拼接格式字符串和参数列表；
6. `va_end` 函数用于将 `va_list` 中的参数传递给 `pcap_vasprintf` 函数，以便正确地将格式字符串和参数列表进行拼接。


```cpp
int
pcap_asprintf(char **strp, const char *format, ...)
{
	va_list args;
	int ret;

	va_start(args, format);
	ret = pcap_vasprintf(strp, format, args);
	va_end(args);
	return (ret);
}


```

# `libpcap/missing/getopt.c`

This is a copy of the "Company [name]


```cpp
/*
 * Copyright (c) 1987, 1993, 1994
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
 *	This product includes software developed by the University of
 *	California, Berkeley and its contributors.
 * 4. Neither the name of the University nor the names of its contributors
 *    may be used to endorse or promote products derived from this software
 *    without specific prior written permission.
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

这段代码是一个条件判断语句，它判断两个条件是否同时为真，如果是，就执行sccsid[0]的值，如果不是，则不做任何操作。具体来说，它首先定义了一个名为LIBC_SCCS的预设标识符，如果没有定义该标识符，那么就会执行它内部的getopt函数。然后，它包含了一个header文件getopt.h，该文件包含函数getopt的一些定义。在包含getopt.h的文件中，可能还包含其他头文件和函数定义。

该代码的作用是判断用户是否定义了LIBC_SCCS标识符，如果定义了，就获取getopt函数的源代码文件名和版本信息，如果没有定义该标识符，就执行默认的getopt函数。然后，该代码还包含一个名为optarg的参数，用于获取用户输入的选项字符串。


```cpp
#if defined(LIBC_SCCS) && !defined(lint)
static char sccsid[] = "@(#)getopt.c	8.3 (Berkeley) 4/27/95";
#endif /* LIBC_SCCS and not lint */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "getopt.h"

int	opterr = 1,		/* if error message should be printed */
	optind = 1,		/* index into parent argv vector */
	optopt,			/* character checked for validity */
	optreset;		/* reset getopt */
char	*optarg;		/* argument associated with option */

```

This is a C program that processes the `-option` format. The `-option` format is used for options in command-line programs. It consists of a verb followed by zero or more position-independent arguments, where the verb is either `-` or some other option followed by a space.

The program first checks if the user has specified a verb using the `-` option. If the user has specified a verb, the program then checks if the user has provided position-independent arguments. If the user has not specified position-independent arguments, the program defaults to `-1`.

The program then checks if the user has specified the `-option` format. If the user has specified the `-option` format, the program then checks if the user has provided position-independent arguments. If the user has not specified position-independent arguments, the program prints an error message and returns a value indicating failure.

If the user has specified position-independent arguments, the program then processes the arguments. The program checks if the user has specified a `-` option followed by a space. If the user has specified a `-` option, the program checks if the user has provided position-independent arguments. If the user has not specified position-independent arguments, the program prints an error message and returns a value indicating failure.

If the user has specified position-independent arguments, the program then processes the arguments. The program checks if the user has specified a `-` option followed by a space. If the user has specified a `-` option, the program checks if the user has provided position-independent arguments. If the user has not specified position-independent arguments, the program prints an error message and returns a value indicating failure.

If the user has specified position-independent arguments, the program then returns the option letter. If the user has not specified position-independent arguments or if the user has specified a `-option` format without position-independent arguments, the program prints an error message and returns a value indicating failure.


```cpp
#define	BADCH	(int)'?'
#define	BADARG	(int)':'
#define	EMSG	""

/*
 * getopt --
 *	Parse argc/argv argument vector.
 */
int
getopt(int nargc, char * const *nargv, const char *ostr)
{
	char *cp;
	static char *__progname;
	static char *place = EMSG;		/* option letter processing */
	char *oli;				/* option letter list index */

	if (__progname == NULL) {
		if ((cp = strrchr(nargv[0], '/')) != NULL)
			__progname = cp + 1;
		else
			__progname = nargv[0];
	}
	if (optreset || !*place) {		/* update scanning pointer */
		optreset = 0;
		if (optind >= nargc || *(place = nargv[optind]) != '-') {
			place = EMSG;
			return (-1);
		}
		if (place[1] && *++place == '-') {	/* found "--" */
			++optind;
			place = EMSG;
			return (-1);
		}
	}
	optopt = (int)*place++;
	if (optopt == (int)':') {		/* option letter okay? */
		if (!*place)
			++optind;
		if (opterr && *ostr != ':')
			(void)fprintf(stderr,
			    "%s: illegal option -- %c\n", __progname, optopt);
		return (BADCH);
	}
	oli = strchr(ostr, optopt);
	if (!oli) {
		/*
		 * if the user didn't specify '-' as an option,
		 * assume it means -1.
		 */
		if (optopt == (int)'-')
			return (-1);
		if (!*place)
			++optind;
		if (opterr && *ostr != ':')
			(void)fprintf(stderr,
			    "%s: illegal option -- %c\n", __progname, optopt);
		return (BADCH);
	}
	if (*++oli != ':') {			/* don't need argument */
		optarg = NULL;
		if (!*place)
			++optind;
	}
	else {					/* need an argument */
		if (*place)			/* no white space */
			optarg = place;
		else if (nargc <= ++optind) {	/* no arg */
			place = EMSG;
			if (*ostr == ':')
				return (BADARG);
			if (opterr)
				(void)fprintf(stderr,
				    "%s: option requires an argument -- %c\n",
				    __progname, optopt);
			return (BADCH);
		}
		else				/* white space */
			optarg = nargv[optind];
		place = EMSG;
		++optind;
	}
	return (optopt);			/* dump back option letter */
}

```

# `libpcap/missing/strlcat.c`

这段代码是一个C语言的函数，名为`pcap_strlcat.c`。它是一个用于打印字符串的函数，其目的是将两个字符串连接起来并存储在`pcap_strlcat`数组中。

函数的第一个参数是一个字符串，第二个参数是一个字符串，这两个字符串将通过连接起来并存储到`pcap_strlcat`数组中。函数的返回值是连接后的字符串。

由于该函数使用了`开放式宏布卢 Strings`库，因此它可以在不需要`#include <stdio.h>`和其他库头文件的情况下使用。


```cpp
/*	$OpenBSD: pcap_strlcat.c,v 1.15 2015/03/02 21:41:08 millert Exp $	*/

/*
 * Copyright (c) 1998, 2015 Todd C. Miller <Todd.Miller@courtesan.com>
 *
 * Permission to use, copy, modify, and distribute this software for any
 * purpose with or without fee is hereby granted, provided that the above
 * copyright notice and this permission notice appear in all copies.
 *
 * THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES
 * WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF
 * MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR
 * ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES
 * WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN
 * ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF
 * OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.
 */

```

这段代码是一个C语言中的函数，名为“append_src_to_string”。它作用于将一个源字符串（src）附加到另一个字符串（dst）中，并返回新的字符串（strlen(src) + MIN(dsize, strlen(initial dst))）。

具体来说，这段代码以下午示例：

1. 如果当前系统已经定义了“config.h”头文件，那么会包含包含在函数中的“#include <config.h>”。

2. 如果当前系统没有定义“config.h”头文件，那么需要预先包含“<config.h>”。

3. 在函数内部，首先检查dst是否已经定义过。如果已经定义过，那么可能会从头文件中包含一个特定于当前系统的头文件（“portability.h”）。

4. 如果已经定义过，那么“portability.h”中的函数将会被包含。否则，编译器将会报错。

5. 在函数主体部分，首先检查dst是否已经定义过。如果是，那么使用strlen()函数获取dst当前的长度，并将其存储在一个变量中。

6. 接下来，使用NUL（空字符串）来确保我们可以将源字符串尽可能多地复制到dst中。这可能意味着我们需要复制尽可能多的字符，但不会超过dst的实际长度。

7. 最后，使用NUL来处理可能在第4步中产生的溢出，并返回源字符串的长度与dst字符串长度的最小值。


```cpp
#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <stddef.h>
#include <string.h>

#include "portability.h"

/*
 * Appends src to string dst of size dsize (unlike strncat, dsize is the
 * full size of dst, not space left).  At most dsize-1 characters
 * will be copied.  Always NUL terminates (unless dsize <= strlen(dst)).
 * Returns strlen(src) + MIN(dsize, strlen(initial dst)).
 * If retval >= dsize, truncation occurred.
 */
```

这段代码是一个名为`pcap_strlcat`的函数，其作用是将从`const char *restrict src`连接到`char *restrict dst`，并返回连接后的字符串长度。

具体实现过程如下：

1. 首先将`dst`指向`const char *restrict src`，将`src`指向需要连接的字符串，将`dsize`初始化为需要连接的字符串长度。
2. 定义一个变量`odst`指向`dst`，定义一个变量`osrc`指向`src`，定义一个变量`n`用于记录需要连接的字符串长度。
3. 遍历`dst`，直到字符串末尾存在字符`'\0'`，此时记录`n`的值。
4. 如果`n`为0，说明连接的字符串长度为0，直接返回`dsize`加上`strlen(src)`，即`odst`。
5. 遍历`src`，直到字符串末尾存在字符`'\0'`，或者遍历完成后`n`仍然不为0。
6. 在遍历过程中，将`odst`的下一个字符与`src`中的字符连接，并将`n`减1。
7. 如果遍历完成后`n`仍然为0，将`odst`赋值为`'\0'`。
8. 返回`dsize`加上`strlen(src)`，即连接后的字符串长度。

注意，由于在函数中使用了`const char *restrict`，因此`dst`和`src`不能被修改为`char *`或`const char *`类型。


```cpp
size_t
pcap_strlcat(char * restrict dst, const char * restrict src, size_t dsize)
{
	const char *odst = dst;
	const char *osrc = src;
	size_t n = dsize;
	size_t dlen;

	/* Find the end of dst and adjust bytes left but don't go past end. */
	while (n-- != 0 && *dst != '\0')
		dst++;
	dlen = dst - odst;
	n = dsize - dlen;

	if (n-- == 0)
		return(dlen + strlen(src));
	while (*src != '\0') {
		if (n != 0) {
			*dst++ = *src;
			n--;
		}
		src++;
	}
	*dst = '\0';

	return(dlen + (src - osrc));	/* count does not include NUL */
}

```

# `libpcap/missing/strlcpy.c`

这段代码是一个C语言的函数，名为`pcap_strlcpy`，它用于将`pcap`格式的数据字符串复制到`strlcpy`函数中。

`pcap`是一个网络数据包抓取器（嗅探器），可以捕获网络流量并保存为字符串。`strlcpy`是一个标准库函数，用于将一个字符串从一个源字符串中复制到另一个目标字符串中。通过将`pcap`捕获到的数据包字符串复制到`strlcpy`函数中，可以得到一个与原始数据包相同但更短（只包含实际可用数据）的字符串。这样就可以将数据包中的敏感信息（如IP地址、端口号等）从数据包中提取并复制到另一个字符串中，而不必暴露这些敏感信息。


```cpp
/*	$OpenBSD: pcap_strlcpy.c,v 1.12 2015/01/15 03:54:12 millert Exp $	*/

/*
 * Copyright (c) 1998, 2015 Todd C. Miller <Todd.Miller@courtesan.com>
 *
 * Permission to use, copy, modify, and distribute this software for any
 * purpose with or without fee is hereby granted, provided that the above
 * copyright notice and this permission notice appear in all copies.
 *
 * THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES
 * WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF
 * MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR
 * ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES
 * WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN
 * ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF
 * OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.
 */

```

这段代码是一个C语言中的预处理指令，名为“#ifdef HAVE_CONFIG_H”。它的作用是在源代码中定义了一个名为“config.h”的文件。如果这个文件已经存在，则包含在该文件中的所有内容都会被包含并编译。否则，该文件和包含它的源文件都不会被编译。

接下来的两个包含头文件的指令包含了一个名为“portability.h”的文件。这个文件很可能是从portability.h.h文件中继承而来的，因为它们都包含了一个名为“portability”。

接下来的三个包含头文件的指令包含了一个名为“stddef.h”的文件。这个文件也很可能是从stddef.h.h文件中继承而来的，因为它们都包含了一个名为“stddef”。

最后，该代码定义了一个名为“sizeof”的函数，它是一个取整函数，用于计算一个给定的变量或类型占用的字节数。它的实现如下：

```cpp
size_t
sizeof(char * str, size_t size)
{
   if (size <= 0) {
       return 0;
   }

   return strlen(str);
}
```

综上所述，这段代码的作用是定义了一个预处理指令，用于检查一个名为“config.h”的文件是否存在，并包含的内容是否已经定义。如果这个文件存在，则编译时将会包含该文件中定义的所有内容。否则，该文件和包含它的源文件都不会被编译。


```cpp
#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <stddef.h>
#include <string.h>

#include "portability.h"

/*
 * Copy string src to buffer dst of size dsize.  At most dsize-1
 * chars will be copied.  Always NUL terminates (unless dsize == 0).
 * Returns strlen(src); if retval >= dsize, truncation occurred.
 */
size_t
```

这段代码是一个名为`pcap_strlcpy`的函数，它的作用是将从`const char *`类型的参数`src`中复制尽可能多的字节到`char *`类型的参数`dst`中，同时将`src`中的字节数减一，如果`dst`已满，则将`dst`的最后一个字节设置为`'\0'`，然后从`src`中继续复制字节，直到`src`为空。

该函数的实现中包含两个条件判断，当`src`长度为0时，判断`dst`是否已满，如果是，则将`dst`的最后一个字节设置为`'\0'`，然后停止复制；如果不是，则说明`dst`还有空间可以复制，从`src`中复制尽可能多的字节到`dst`中，并将`src`中剩余的`nleft`个字节复制到`dst`中。复制完成后，函数返回从`src`到`dst`的复制长度，但不包括NUL结尾的字符。


```cpp
pcap_strlcpy(char * restrict dst, const char * restrict src, size_t dsize)
{
	const char *osrc = src;
	size_t nleft = dsize;

	/* Copy as many bytes as will fit. */
	if (nleft != 0) {
		while (--nleft != 0) {
			if ((*dst++ = *src++) == '\0')
				break;
		}
	}

	/* Not enough room in dst, add NUL and traverse rest of src. */
	if (nleft == 0) {
		if (dsize != 0)
			*dst = '\0';		/* NUL-terminate dst */
		while (*src++)
			;
	}

	return(src - osrc - 1);	/* count does not include NUL */
}

```

# `libpcap/missing/strtok_r.c`

The Regents of the University of California have permission to distribute and modify this software as long as the copyright notifications and disclaimer are included and the name of the University and its contributors are not used to promote the software without specific prior written permission. This software is provided "AS IS" with no warranty of any kind, including but not limited to the implied warranties of merchantability and fitness for a particular purpose. The contributors of this software agree to the terms of this software license.



```cpp
/*-
 * Copyright (c) 1998 Softweyr LLC.  All rights reserved.
 *
 * strtok_r, from Berkeley strtok
 * Oct 13, 1998 by Wes Peters <wes@softweyr.com>
 *
 * Copyright (c) 1988, 1993
 *	The Regents of the University of California.  All rights reserved.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions
 * are met:
 * 1. Redistributions of source code must retain the above copyright
 *    notices, this list of conditions and the following disclaimer.
 * 2. Redistributions in binary form must reproduce the above copyright
 *    notices, this list of conditions and the following disclaimer in the
 *    documentation and/or other materials provided with the distribution.
 * 3. Neither the name of the University nor the names of its contributors
 *    may be used to endorse or promote products derived from this software
 *    without specific prior written permission.
 *
 * THIS SOFTWARE IS PROVIDED BY SOFTWEYR LLC, THE REGENTS AND CONTRIBUTORS
 * ``AS IS'' AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
 * LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A
 * PARTICULAR PURPOSE ARE DISCLAIMED.  IN NO EVENT SHALL SOFTWEYR LLC, THE
 * REGENTS, OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
 * SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED
 * TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
 * PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF
 * LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
 * NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
 * SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
 *
 * From: @(#)strtok.c	8.1 (Berkeley) 6/4/93
 */

```

这段代码是一个C语言程序，它定义了一个名为`pcap_strtok_r`的函数。这个函数的作用是用于对一个字符串`s`按照给定的模式和分隔符进行分割，返回分割出来的字符串。

函数的实现包括以下步骤：

1. 定义两个字符指针变量`spanp`和`tok`，初始化为`NULL`，并定义一个指向整数的变量`last`。

2. 使用if语句判断`s`是否为`NULL`，如果是，则说明已经到达了字符串的结尾，不需要进行分割，将`NULL`作为返回值。

3. 如果`s`不为`NULL`，则执行以下步骤：

  a. 如果`last`所指向的地址已经到达了字符串的结尾，执行以下语句：

   ```cpp
   spanp = last;
   last = last + strlen(last) - 1;
   ```

   b. 如果`last`所指向的地址还有剩余，执行以下语句：

   ```cpp
   c = strcspn(s, delim);
   tokens[c] = last + c;
   last = last + strlen(last) - 1;
   ```

   c. 将分割得到的字符串存储在`tokens`数组中，并更新`last`指针。

4. 函数返回`tokens`数组，即分割后的字符串。

该函数的作用是将给定的字符串`s`按照指定的模式和分隔符进行分割，并将分割得到的字符串存储在`tokens`数组中。由于分隔符可以出现在字符串的任意位置，因此这个函数可以用于解析网络数据包、文本文件等数据。


```cpp
#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include "portability.h"

char *
pcap_strtok_r(char *s, const char *delim, char **last)
{
	char *spanp, *tok;
	int c, sc;

	if (s == NULL && (s = *last) == NULL)
		return (NULL);

	/*
	 * Skip (span) leading delimiters (s += strspn(s, delim), sort of).
	 */
```

这段代码的作用是读取一个字符串，其中的逗号作为分隔符，分隔出一个个字符。程序会从开始位置开始遍历，每当找到一个逗号，就将之前扫描到的字符移动到后一个位置，直到整个字符串结束或者遇到空字符串。

具体来说，程序的作用可以分解为以下几个步骤：

1. 初始化变量 c 和 spanp 为字符串的第一个字符和第一个分隔符。
2. 遍历整个字符串，从开始位置开始。
3. 如果找到字符逗号，就将 spanp 移动到后一个位置，继续遍历。
4. 如果循环到字符串的结尾或者遇到空格，就将 c 设为 0，并将 last 指向 null。
5. 在遍历过程中，如果找到字符逗号，就将之前扫描到的字符串中的最后一个字符赋值给 c，并返回该逗号对应的下一个字符。
6. 如果循环过程中不达到字符串的结尾或者遇到空格，则说明字符串中不包含逗号，将 last 指向 null。
7. 如果循环过程中正常结束，则输出结果并返回 NULL。


```cpp
cont:
	c = *s++;
	for (spanp = (char *)delim; (sc = *spanp++) != 0;) {
		if (c == sc)
			goto cont;
	}

	if (c == 0) {		/* no non-delimiter characters */
		*last = NULL;
		return (NULL);
	}
	tok = s - 1;

	/*
	 * Scan token (scan for delimiters: s += strcspn(s, delim), sort of).
	 * Note that delim must have one NUL; we stop if we see that, too.
	 */
	for (;;) {
		c = *s++;
		spanp = (char *)delim;
		do {
			if ((sc = *spanp++) == c) {
				if (c == 0)
					s = NULL;
				else
					s[-1] = '\0';
				*last = s;
				return (tok);
			}
		} while (sc != 0);
	}
	/* NOTREACHED */
}

```

# `libpcap/missing/win_asprintf.c`

这段代码是一个C语言的函数，名为`pcap_vasprintf`，它用于将一个格式字符串和多个参数进行字符串格式化。

该函数的参数包括：

- `strp`：一个指向字符数组的指针，用于存储格式化后的字符串；
- `format`：一个格式字符串，用于指定要格式化的字符串；
- `args`：一个包含多个可变参数的指针，用于存储各个参数；

函数实现的内容如下：

1. 首先定义了一个名为`pcap_vasprintf`的函数，该函数接受三个参数：`strp`、`format`和`args`。
2. 在函数体中，首先定义了一个名为`len`的整型变量，用于存储输入的格式字符串的长度。
3. 定义了一个名为`str`的指针，用于存储格式化后的字符串，该指针被初始化为`malloc`函数返回的一个内存区域。
4. 调用`_vscprintf`函数来将输入的格式字符串和各个参数进行字符串格式化。如果`format`的长度大于输入参数的长度，函数将返回负数，表示无法完成格式化。
5. 在格式化成功后，将`str`指向的字符数组中的字符复制到`strp`指向的字符数组中，并返回成功返回的返回值。

该函数的作用是帮助用户在输入参数中格式化字符串，以便将输入参数的字符串按照指定的格式进行格式化，并在完成格式化后返回成功返回的返回值。


```cpp
#include <stdio.h>
#include <stdlib.h>
#include <stdarg.h>

#include "portability.h"

int
pcap_vasprintf(char **strp, const char *format, va_list args)
{
	int len;
	size_t str_size;
	char *str;
	int ret;

	len = _vscprintf(format, args);
	if (len == -1) {
		*strp = NULL;
		return (-1);
	}
	str_size = len + 1;
	str = malloc(str_size);
	if (str == NULL) {
		*strp = NULL;
		return (-1);
	}
	ret = vsnprintf(str, str_size, format, args);
	if (ret == -1) {
		free(str);
		*strp = NULL;
		return (-1);
	}
	*strp = str;
	/*
	 * vsnprintf() shouldn't truncate the string, as we have
	 * allocated a buffer large enough to hold the string, so its
	 * return value should be the number of characters printed.
	 */
	return (ret);
}

```

这段代码是一个名为 `pcap_asprintf` 的函数，它是 pcap-lib 库中的一个函数，用于将格式化字符串中的格式参数（如时间戳、人数等）打印到一个字符指针变量 `strp` 中。

函数接受两个参数，一个是 `const char *format`，另一个是 `...` 表示可变参数列表，这些参数将在格式化字符串中被替换。第一个参数是一个指向字符指针的指针，第二个参数是一个格式化字符串，用于将格式化参数的值替换到指针所指向的字符串中。

函数内部首先定义了一个名为 `va_list` 的变量，它用于存储可变参数列表。然后，定义了一个名为 `ret` 的整数变量，用于存储将格式化字符串打印到 `strp` 的结果。接着，定义了一个名为 `va_end` 的函数指针，用于指向格式化参数列表的结束位置。最后，定义了 `pcap_vasprintf` 函数，用于将格式化字符串打印到 `strp` 中，并将 `strp` 和格式化字符串作为参数传入。

整个函数的作用是将传入的格式化字符串中的参数打印到一个字符指针变量 `strp` 中，并返回打印结果。


```cpp
int
pcap_asprintf(char **strp, const char *format, ...)
{
	va_list args;
	int ret;

	va_start(args, format);
	ret = pcap_vasprintf(strp, format, args);
	va_end(args);
	return (ret);
}

```

# `libpcap/msdos/bin2c.c`

这段代码的主要作用是读取一个名为 "bin-file" 的文件，并将其内容打印到标准输出或一个名为 "result" 的文件中。它包含以下主要函数：

1. `Abort()` 函数，用于打印错误消息并终止程序。它的参数是一个格式化字符串和多个可变参数。在函数内部，使用 `vfprintf()` 和 `va_end()` 函数将格式化字符串和可变参数分别打印到标准输出和标准错误中。最后，程序使用 `exit()` 函数终止，并打印一个错误消息。

2. `main()` 函数，程序的主要入口点。它包含以下步骤：

 2. 如果命令行参数数目小于2，函数将打印错误消息并终止。

 3. 如果 `bin-file` 文件的输入和输出文件都存在，函数将打开输入文件并读取其内容。

 4. 函数将内容打印到标准输出或 "result" 文件中。为此，它首先获取输入文件中的下一行，并使用可变参数 `fmt` 来打印该行。然后，它尝试使用 `fputs()` 函数将内容打印到标准输出或 "result" 文件中。如果任何一次尝试失败，函数将打印错误消息并终止。

 5. 最后，函数使用 `fclose()` 函数关闭输入文件。


```cpp
#include <stdio.h>
#include <stdlib.h>
#include <stdarg.h>
#include <time.h>

static void Abort (const char *fmt,...)
{
  va_list args;
  va_start (args, fmt);
  vfprintf (stderr, fmt, args);
  va_end (args);
  exit (1);
}

int main (int argc, char **argv)
{
  FILE  *inFile;
  FILE  *outFile = stdout;
  time_t now     = time (NULL);
  int    ch, i;

  if (argc != 2)
     Abort ("Usage: %s bin-file [> result]", argv[0]);

  if ((inFile = fopen(argv[1],"rb")) == NULL)
     Abort ("Cannot open %s\n", argv[1]);

  fprintf (outFile,
           "/* data statements for file %s at %.24s */\n"
           "/* Generated by BIN2C, G. Vanem 1995 */\n",
           argv[1], ctime(&now));

  i = 0;
  while ((ch = fgetc(inFile)) != EOF)
  {
    if (i++ % 12 == 0)
       fputs ("\n  ", outFile);
    fprintf (outFile, "0x%02X,", ch);
  }
  fputc ('\n', outFile);
  fclose (inFile);
  return (0);
}

```

# `libpcap/msdos/pktdrvr.c`

这段代码定义了一个名为"pktdrvr.c"的C文件，它是一个用于实现数据传输的接口。这个接口定义了16/32位C和Borland C/C++ 3.0+ small/large模型的数据传输特性。它还兼容于Watcom C/C++ 11+，DOS4GW flat模型，Metaware HighC 3.1+和PharLap 386|DosX，GNU C/C++ 2.7+以及 Djgpp 2.x扩展器。

pktdrvr.c的作用是为这些操作系统和平台提供一种标准的数据传输接口，使得开发人员可以使用单一的接口来实现不同平台的代码。这个接口还定义了各种数据传输方式，包括读取和写入串行端口，并支持多种数据传输模式，如主设备/从设备，主设备/从设备主机到主机，存储设备到存储设备等。


```cpp
/*
 *  File.........: pktdrvr.c
 *
 *  Responsible..: Gisle Vanem,  giva@bgnett.no
 *
 *  Created......: 26.Sept 1995
 *
 *  Description..: Packet-driver interface for 16/32-bit C :
 *                 Borland C/C++ 3.0+ small/large model
 *                 Watcom C/C++ 11+, DOS4GW flat model
 *                 Metaware HighC 3.1+ and PharLap 386|DosX
 *                 GNU C/C++ 2.7+ and djgpp 2.x extender
 *
 *  References...: PC/TCP Packet driver Specification. rev 1.09
 *                 FTP Software Inc.
 *
 */

```



该代码是一个用C语言编写的DOS程序，用于读取DOS类型数据包（pktdrvr.sys）的网络接口卡配置信息。主要作用是读取配置文件中的IP地址和子网掩码，并将它们存储到内存中。以下是程序的主要步骤：

1. 引入标准输入输出库和DOS库；
2. 引入pktdrvr库；
3. 初始化DOS程序的一些成员变量，如当前盘符、当前配置文件中的IP地址和子网掩码等；
4. 如果当前操作系统是DOSX，则定义Rx FIFO队列的大小为32个缓冲区；
5. 循环读取配置文件中的配置字段；
6. 将配置字中的IP地址和子网掩码拷贝到内存中的IP地址变量和子网掩码变量中；
7. 调用pktdrvr库中的get_ip_info函数，设置IP驱动程序的api_class为msdos.pktdrvr.Pkg，api_subclass为pktdrvr.Dls，param_len为2，param_val为&ip_info；
8. 循环遍历配置文件中的子网掩码字段，将子网掩码转换为二进制形式，并将其作为参数传递给pktdrvr库中的get_net_info函数，获取网络接口卡的地址信息，并将其存储到内存中的子网掩码变量中；
9. 循环读取配置文件中的配置字段，并设置当前盘符为0；
10. 如果当前操作系统是DOSX，则将Rx FIFO队列的缓冲区大小设置为32；
11. 循环从内存中读取Rx FIFO队列中的数据，并输出到控制台；
12. 如果不包含任何数据，显示当前盘符为当前配置文件中的IP地址和子网掩码。

该程序的作用是读取DOS类型数据包的网络接口卡配置信息，包括IP地址和子网掩码，并将其存储到内存中的ip_info变量和sub_net_info变量中。通过调用pktdrvr库中的get_ip_info函数，可以方便地获取网络接口卡的地址信息。


```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <dos.h>

#include "pcap-dos.h"
#include "pcap-int.h"
#include "msdos/pktdrvr.h"

#if (DOSX)
#define NUM_RX_BUF  32      /* # of buffers in Rx FIFO queue */
#else
#define NUM_RX_BUF  10
#endif

```

这段代码定义了两个宏，分别是 `DIM` 和 `PUTS`。

`DIM` 宏的定义如下：
```cpparduino
#define DIM(x)   (sizeof((x)) / sizeof(x[0]))
```
它的作用是计算一个数组 `x` 的大小，然后将结果赋值给宏名 `DIM`。这个宏可以被用来定义其他宏，例如：
```cppc
#define DIMS(x)   DIM(x)
#define DUMP(x)  DIM(x)
```
`PUTS` 宏的定义如下：
```cpparduino
#define PUTS(s)  do {                                           \
                  if (!pktInfo.quiet)                          \
                     pktInfo.error ?                           \
                       printf ("%s: %s\n", s, pktInfo.error) : \
                       printf ("%s\n", pktInfo.error = s);     \
                } while (0)
```
它的作用是打印一个字符串 `s`，并检查 `pktInfo.quiet` 是否为 `0`。如果是，则输出 `s` 错误的信息，否则输出 `s` 没有错误的信息。这个宏可以被用来输出错误信息，例如：
```cpparduino
#define PUTS(s)  PUTS(s)
#define PUTS_WERROR(s) PUTS(s)
```
这里 `PUTS_WERROR` 宏就是对 `PUTS` 宏的扩展，使用了 `WERROR` 参数，可以输出错误信息。


```cpp
#define DIM(x)   (sizeof((x)) / sizeof(x[0]))
#define PUTS(s)  do {                                           \
                   if (!pktInfo.quiet)                          \
                      pktInfo.error ?                           \
                        printf ("%s: %s\n", s, pktInfo.error) : \
                        printf ("%s\n", pktInfo.error = s);     \
                 } while (0)

#if defined(__HIGHC__)
  extern UINT _mwenv;

#elif defined(__DJGPP__)
  #include <stddef.h>
  #include <dpmi.h>
  #include <go32.h>
  #include <pc.h>
  #include <sys/farptr.h>

```

I'm sorry, but I don't have enough information to


```cpp
#elif defined(__WATCOMC__)
  #include <i86.h>
  #include <stddef.h>
  extern char _Extender;

#else
  extern void far PktReceiver (void);
#endif


#if (DOSX & (DJGPP|DOS4GW))
  #include <sys/pack_on.h>

  struct DPMI_regs {
         DWORD  r_di;
         DWORD  r_si;
         DWORD  r_bp;
         DWORD  reserved;
         DWORD  r_bx;
         DWORD  r_dx;
         DWORD  r_cx;
         DWORD  r_ax;
         WORD   r_flags;
         WORD   r_es, r_ds, r_fs, r_gs;
         WORD   r_ip, r_cs, r_sp, r_ss;
       };

  /* Data located in a real-mode segment. This becomes far at runtime
   */
  typedef struct  {          /* must match data/code in pkt_rx1.s */
          WORD       _rxOutOfs;
          WORD       _rxInOfs;
          DWORD      _pktDrop;
          BYTE       _pktTemp [20];
          TX_ELEMENT _pktTxBuf[1];
          RX_ELEMENT _pktRxBuf[NUM_RX_BUF];
          WORD       _dummy[2];        /* screenSeg,newInOffset */
          BYTE       _fanChars[4];
          WORD       _fanIndex;
          BYTE       _PktReceiver[15]; /* starts on a paragraph (16byte) */
        } PktRealStub;
  #include <sys/pack_off.h>

  static BYTE real_stub_array [] = {
         #include "pkt_stub.inc"       /* generated opcode array */
       };

  #define rxOutOfs      offsetof (PktRealStub,_rxOutOfs)
  #define rxInOfs       offsetof (PktRealStub,_rxInOfs)
  #define PktReceiver   offsetof (PktRealStub,_PktReceiver [para_skip])
  #define pktDrop       offsetof (PktRealStub,_pktDrop)
  #define pktTemp       offsetof (PktRealStub,_pktTemp)
  #define pktTxBuf      offsetof (PktRealStub,_pktTxBuf)
  #define FIRST_RX_BUF  offsetof (PktRealStub,_pktRxBuf [0])
  #define LAST_RX_BUF   offsetof (PktRealStub,_pktRxBuf [NUM_RX_BUF-1])

```

这段代码定义了一个名为RX_ELEMENT的局部变量，它是一个4字节的整型变量数组，可以存储在用户提供的输入数据中。这个数组有两个元素，分别命名为pktRxBuf和pktTxBuf，用来存储接收机接收到的数据包和发送的数据包。

接着，代码定义了一个名为pktDrop的整型变量，用来记录在PktReceiver()函数中由用户丢弃的数据包数量。

接下来的代码定义了一个名为pktRxEnd的布尔型变量，用来标记接收机是否处于Rx模式代码/数据结束的状态。如果这个变量为TRUE，那么表示接收机已经接收到了所有的数据包，可以停止接收，否则继续接收。

最后，代码定义了一个名为pktTemp的20字节长的整型变量，用来存储临时缓冲区，用于在RX_ELEMENT数组中接收数据包。


```cpp
#else
  extern WORD       rxOutOfs;    /* offsets into pktRxBuf FIFO queue   */
  extern WORD       rxInOfs;
  extern DWORD      pktDrop;     /* # packets dropped in PktReceiver() */
  extern BYTE       pktRxEnd;    /* marks the end of r-mode code/data  */

  extern RX_ELEMENT pktRxBuf [NUM_RX_BUF];       /* PktDrvr Rx buffers */
  extern TX_ELEMENT pktTxBuf;                    /* PktDrvr Tx buffer  */
  extern char       pktTemp[20];                 /* PktDrvr temp area  */

  #define FIRST_RX_BUF (WORD) &pktRxBuf [0]
  #define LAST_RX_BUF  (WORD) &pktRxBuf [NUM_RX_BUF-1]
#endif


```

这段代码定义了一系列与内存相关的头文件和函数指针，旨在定义 Borland 中内联函数的名称和格式。

具体来说，这段代码定义了以下函数：

- memcpy：将两个实参按位复制到目标实参中。
- memcmp：比较两个实参，返回它们的差异。
- memset：将目标实参中的一个或多个字节设置为特定的值，具体值由实参指定。

这些函数是通过定义头文件 #ifdef __BORLANDC__ 来引入的。如果当前编译器支持 Borland 汇编语言，就定义了一系列函数指针。

此外，代码中还有一段注释说明，指出这段代码是在 DOSX 且 PHARLAP 条件下编译的。


```cpp
#ifdef __BORLANDC__           /* Use Borland's inline functions */
  #define memcpy  __memcpy__
  #define memcmp  __memcmp__
  #define memset  __memset__
#endif


#if (DOSX & PHARLAP)
  extern void PktReceiver (void);     /* in pkt_rx0.asm */
  static int  RealCopy    (ULONG, ULONG, REALPTR*, FARPTR*, USHORT*);

  #undef  FP_SEG
  #undef  FP_OFF
  #define FP_OFF(x)     ((WORD)(x))
  #define FP_SEG(x)     ((WORD)(realBase >> 16))
  #define DOS_ADDR(s,o) (((DWORD)(s) << 16) + (WORD)(o))
  #define r_ax          eax
  #define r_bx          ebx
  #define r_dx          edx
  #define r_cx          ecx
  #define r_si          esi
  #define r_di          edi
  #define r_ds          ds
  #define r_es          es
  LOCAL FARPTR          protBase;
  LOCAL REALPTR         realBase;
  LOCAL WORD            realSeg;   /* DOS para-address of allocated area */
  LOCAL SWI_REGS        reg;

  static WORD _far *rxOutOfsFp, *rxInOfsFp;

```

这段代码是在if语句中，根据DOSX和DJGPP平台的某些标志位是否为真来执行不同的操作。

具体来说，如果DOSX和DJGPP均为真，则执行以下操作：

1. 定义一个名为rm_mem的静态变量，其值为0；
2. 定义一个名为reg的静态变量，其值为0；
3. 定义一个名为realBase的静态变量，其值为0；
4. 定义一个名为para_skip的静态变量，其值为0；
5. 定义一个名为DOS_ADDR的函数，其作用是接收两个参数s和o，计算出DOS_ADDR函数的返回值；
6. 在DOS_ADDR函数中，根据DOSX和DJGPP平台来计算出参数s和o的地址；
7. 根据计算出的地址，执行以下操作：
	* 读取指定位置的内存，并将其存储在rm_mem中；
	* 读取指定位置的内存，并将其存储在reg中；
	* 将realBase的值设置为当前内存位置的偏移量；
	* 如果para_skip为1，则跳过当前参数；
	* 否则，将para_skip设置为1。

这段代码的作用是检查DOSX和DJGPP平台是否支持执行特定操作，并根据结果执行不同的操作。


```cpp
#elif (DOSX & DJGPP)
  static _go32_dpmi_seginfo rm_mem;
  static __dpmi_regs        reg;
  static DWORD              realBase;
  static int                para_skip = 0;

  #define DOS_ADDR(s,o)     (((WORD)(s) << 4) + (o))
  #define r_ax              x.ax
  #define r_bx              x.bx
  #define r_dx              x.dx
  #define r_cx              x.cx
  #define r_si              x.si
  #define r_di              x.di
  #define r_ds              x.ds
  #define r_es              x.es

```

这段代码是一个条件判断，判断DOSX和DOS4GW两个条件是否为真。如果是，则执行以下操作：

1. 定义一个名为struct DPMI_regs的结构体变量reg，以及一个名为rm_base_seg和rm_base_sel的WORD变量。
2. 定义一个名为realBase的DWORD变量。
3. 定义一个名为para_skip的int变量，并将其初始化为0。
4. 调用dpmi_get_real_vector函数，并将其结果存储在para_skip中。
5. 如果调用成功，那么根据DOS_ADDR函数，将成功存储的int值偏移4个字节，并将其与rm_base_sel相加得到新的base_sel。
6. 如果没有调用dpmi_get_real_vector函数，那么直接跳过para_skip，并将rm_base_sel赋值为0。
7. 根据DOSX和DOS4GW判断结果，如果为真，则执行以下操作：
  a. 定义一个名为reg的结构体变量，包含r_ax、r_bx、r_cx、r_dx和r_si五个成员变量。
  b. 定义一个名为selector的WORD变量，用于选择要分配给rm_base_sel的内存区域。
  c. 调用dpmi_real_malloc函数，并将size、selector和rm_base_sel作为参数传入，成功后将结果存储到reg中，rm_base_sel的值也随之赋为selector。
  d. 调用dpmi_real_free函数，将rm_base_sel作为参数传入，成功后rm_base_sel的值被更新为0。


```cpp
#elif (DOSX & DOS4GW)
  LOCAL struct DPMI_regs    reg;
  LOCAL WORD                rm_base_seg, rm_base_sel;
  LOCAL DWORD               realBase;
  LOCAL int                 para_skip = 0;

  LOCAL DWORD dpmi_get_real_vector (int intr);
  LOCAL WORD  dpmi_real_malloc     (int size, WORD *selector);
  LOCAL void  dpmi_real_free       (WORD selector);
  #define DOS_ADDR(s,o) (((DWORD)(s) << 4) + (WORD)(o))

#else              /* real-mode Borland etc. */
  static struct  {
         WORD r_ax, r_bx, r_cx, r_dx, r_bp;
         WORD r_si, r_di, r_ds, r_es, r_flags;
       } reg;
```

这段代码定义了一个名为 `pktStat` 的 `PKT_STAT` 类型的变量，并对其进行初始化。同时，通过 `#ifdef` 和 `#endif` 预处理指令，当源代码中包含 `__HIGHC__` 时，会启用某些 `#pragma` 指令。

具体来说，这段代码定义了一个 `PKT_STAT` 类型的变量，用于存储数据包统计信息。通过 `#pragma Alias` 指令，对变量进行了别名定义，以简化代码。这里定义的别名包括：

- `pktDrop`：数据包丢失统计；
- `_pktDrop`：数据包丢失的定义；
- `pktRxBuf`：接收到的数据包缓冲区统计；
- `_pktRxBuf`：接收到的数据包缓冲区的定义；
- `pktTemp`： temporary 缓存的数据包统计；
- `_pktTemp`：temp 缓存的数据包统计的定义；
- `rxOutOfs`：输出到 RX 输出节点的统计；
- `_rxOutOfs`：输出到 RX 输出节点的定义；
- `pktRxEnd`：接收到的数据包结束的统计；
- `_pktRxEnd`：接收到的数据包结束的定义；
- `PktReceiver`：接收到的数据包的接收者。

最后，在 `PUBLIC` 修饰下，可以被任何使用 `pktStat` 的函数访问和修改。


```cpp
#endif

#ifdef __HIGHC__
  #pragma Alias (pktDrop,    "_pktDrop")
  #pragma Alias (pktRxBuf,   "_pktRxBuf")
  #pragma Alias (pktTxBuf,   "_pktTxBuf")
  #pragma Alias (pktTemp,    "_pktTemp")
  #pragma Alias (rxOutOfs,   "_rxOutOfs")
  #pragma Alias (rxInOfs,    "_rxInOfs")
  #pragma Alias (pktRxEnd,   "_pktRxEnd")
  #pragma Alias (PktReceiver,"_PktReceiver")
#endif


PUBLIC PKT_STAT    pktStat;    /* statistics for packets    */
```

这段代码定义了一个名为 pktInfo 的包信息结构体，其中包含了一些与数据包相关的信息。下面是各个成员的说明：

1. pktInfo: 包信息结构体，用于存储接收到的数据包的相关信息。
 - receiveMode: 接收模式，可以是 PDRX_DIRECT 或者 PDRX_MEM。
 - myAddress: 我的 MAC 地址，是一个二进制字符数组，用于标识发送的数据包是属于发送者的哪一个 MAC 地址。
 - ethBroadcast: 发送者的广播 MAC 地址，是一个二进制字符数组，用于标识所有接收者都能够接收到数据包的 MAC 地址。

2. receiveMode: 接收模式，可以是 PDRX_DIRECT 或者 PDRX_MEM。这个成员用于指示数据包是以哪种方式接收的，是代码的一部分，但不会输出到函数外。

3. myAddress: 我的 MAC 地址，是一个二进制字符数组，用于标识发送的数据包是属于发送者的哪一个 MAC 地址。这个成员可能会在某些数据包传输过程中发生变化，从而导致数据包的丢失或延迟，需要进行特殊处理。

4. ethBroadcast: 发送者的广播 MAC 地址，是一个二进制字符数组，用于标识所有接收者都能够接收到数据包的 MAC 地址。这个成员可能会在某些数据包传输过程中发生变化，从而导致数据包的丢失或延迟，需要进行特殊处理。

5. internalStat: 内部统计信息，包括 tooSm small、tooLarge 和 wrongHandle 等成员。这些统计信息可能与数据包的传输有关，用于在必要时回滚或重发数据包。

6. PDRX_MEM: PDRX_MEM 函数的别名，可能是一个内存-to-screen 类型的函数，用于将数据包发送到内存中的 MAC 地址。这个函数需要提供数据包的头部信息，包括 source 和 destination MAC addresses，以及数据包的长度。这个函数在接收模式下不会被使用。


```cpp
PUBLIC PKT_INFO    pktInfo;    /* packet-driver information */

PUBLIC PKT_RX_MODE receiveMode  = PDRX_DIRECT;
PUBLIC ETHER       myAddress    = {   0,  0,  0,  0,  0,  0 };
PUBLIC ETHER       ethBroadcast = { 255,255,255,255,255,255 };

LOCAL  struct {             /* internal statistics */
       DWORD  tooSmall;     /* size < ETH_MIN */
       DWORD  tooLarge;     /* size > ETH_MAX */
       DWORD  badSync;      /* count_1 != count_2 */
       DWORD  wrongHandle;  /* upcall to wrong handle */
     } intStat;

/***************************************************************************/

```

这段代码定义了一个名为 `PktGetErrorStr` 的函数，用于获取错误信息。函数接收一个整数参数 `errNum`，表示出了错的天数。函数内部定义了一个静态的数组 `errStr`，其中包含了 11 个字符串元素，用于存储各种不同的错误信息。函数内部使用 `const` 关键字定义了这个数组，以确保数组中的字符串是常量，而不是可变的。

函数的作用是获取错误代码，即将 `errNum` 错误号与数组 `errStr` 中相应的字符串进行比较，返回错误信息。具体的实现过程如下：

1. 定义了一个名为 `errStr` 的数组，用于存储错误信息。数组长度为 `DIM(errStr)` 即 11，其中 `DIM` 函数返回的是数组的大小，而不是数组中元素的个数。

2. 对于给定的 `errNum` 错误号，首先检查它是否小于 0。如果是，则说明出了严重错误，将返回字符串 `"Unknown driver error"`。

3. 如果 `errNum` 在 `DIM(errStr)` 范围内，则使用 `strcpy` 函数将 `errStr` 中相应的错误信息字符串复制给 `errNum` 错误号，并返回该错误信息。

4. 如果 `errNum` 不在 `DIM(errStr)` 范围内，或者 `errNum` 是负数，则将 `errStr` 中第一个元素 `"Unknown driver error"` 复制给 `errNum` 错误号，并返回该错误信息。

这段代码的主要作用是获取出错信息，对于不同的错误情况，返回不同的字符串。这个函数对于开发驱动程序非常重要，可以帮助开发人员快速定位和解决问题。


```cpp
PUBLIC const char *PktGetErrorStr (int errNum)
{
  static const char *errStr[] = {
                    "",
                    "Invalid handle number",
                    "No interfaces of specified class found",
                    "No interfaces of specified type found",
                    "No interfaces of specified number found",
                    "Bad packet type specified",
                    "Interface does not support multicast",
                    "Packet driver cannot terminate",
                    "Invalid receiver mode specified",
                    "Insufficient memory space",
                    "Type previously accessed, and not released",
                    "Command out of range, or not implemented",
                    "Cannot send packet (usually hardware error)",
                    "Cannot change hardware address ( > 1 handle open)",
                    "Hardware address has bad length or format",
                    "Cannot reset interface (more than 1 handle open)",
                    "Bad Check-sum",
                    "Bad size",
                    "Bad sync" ,
                    "Source hit"
                  };

  if (errNum < 0 || errNum >= DIM(errStr))
     return ("Unknown driver error.");
  return (errStr [errNum]);
}

```



这段代码定义了一个名为 `PktGetClassName` 的函数，用于获取特定数据包协议(PD)类别的名称。

函数接收一个 `class` 参数，它是一个整数，表示要获取的 PD 类别。函数的逻辑根据 PD 类别执行不同的操作，并返回相应的名称。

具体来说，函数在不同 PD 类别下执行以下操作：

- 如果在 PD 类别为 PD_ETHER 时，返回 "DIX-Ether" 的名称。
- 如果在 PD 类别为 PD_PRONET10 时，返回 "ProNET-10" 的名称。
- 如果在 PD 类别为 PD_IEEE8025 时，返回 "IEEE 802.5" 的名称。
- 如果在 PD 类别为 PD_OMNINET 时，返回 "OmniNet" 的名称。
- 如果在 PD 类别为 PD_APPLETALK 时，返回 "AppleTalk" 的名称。
- 如果在 PD 类别为 PD_SLIP 时，返回 "SLIP" 的名称。
- 如果在 PD 类别为 PD_STARTLAN 时，返回 "StartLAN" 的名称。
- 如果在 PD 类别为 PD_ARCNET 时，返回 "ArcNet" 的名称。
- 如果在 PD 类别为 PD_AX25 时，返回 "AX.25" 的名称。
- 如果在 PD 类别为 PD_KISS 时，返回 "KISS" 的名称。
- 如果在 PD 类别为 PD_IEEE8023_2 时，返回 "IEEE 802.3 w/802.2 hdr" 的名称。
- 如果在 PD 类别为 PD_FDDI8022 时，返回 "FDDI w/802.2 hdr" 的名称。
- 如果在 PD 类别为 PD_X25 时，返回 "X.25" 的名称。
- 如果在 PD 类别为 PD_LANstar 时，返回 "LANstar" 的名称。
- 如果在 PD 类别为 PD_PPP 时，返回 "PPP" 的名称。

如果在 PD 类别不匹配任何上述 PD 类别，函数将返回 "unknown" 的名称。


```cpp
/**************************************************************************/

PUBLIC const char *PktGetClassName (WORD class)
{
  switch (class)
  {
    case PD_ETHER:
         return ("DIX-Ether");
    case PD_PRONET10:
         return ("ProNET-10");
    case PD_IEEE8025:
         return ("IEEE 802.5");
    case PD_OMNINET:
         return ("OmniNet");
    case PD_APPLETALK:
         return ("AppleTalk");
    case PD_SLIP:
         return ("SLIP");
    case PD_STARTLAN:
         return ("StartLAN");
    case PD_ARCNET:
         return ("ArcNet");
    case PD_AX25:
         return ("AX.25");
    case PD_KISS:
         return ("KISS");
    case PD_IEEE8023_2:
         return ("IEEE 802.3 w/802.2 hdr");
    case PD_FDDI8022:
         return ("FDDI w/802.2 hdr");
    case PD_X25:
         return ("X.25");
    case PD_LANstar:
         return ("LANstar");
    case PD_PPP:
         return ("PPP");
    default:
         return ("unknown");
  }
}

```

这段代码定义了一个名为 `PktRXmodeStr` 的公共函数，它接收一个名为 `mode` 的整数参数，并返回一个字符串，描述了指定模式的含义。

该函数的实现采用了下列步骤：

1. 定义了一个名为 `modeStr` 的静态字符数组，包含 6 个字符串元素，分别表示不同的模式的含义。
2. 如果传递给函数的 `mode` 参数超出了 `modeStr` 数组长度，函数会返回一个通用的字符串 "??"，表示无法确定模式的意义。
3. 使用 `modeStr` 数组长度减去 `mode` 参数，得到一个整数 `index`，它指向了与 `mode` 参数最接近的元素，并将其作为字符串返回，以表示该模式。
4. 如果 `mode` 参数不在 `modeStr` 数组中，函数会默认返回 "Receiver turned off"，表示无法确定模式的意义。

因此，该函数的作用是定义了一个字符串，根据传入的模式参数，输出相应的字符串，以表示不同的数据接收模式。


```cpp
/**************************************************************************/

PUBLIC char const *PktRXmodeStr (PKT_RX_MODE mode)
{
  static const char *modeStr [] = {
                    "Receiver turned off",
                    "Receive only directly addressed packets",
                    "Receive direct & broadcast packets",
                    "Receive direct,broadcast and limited multicast packets",
                    "Receive direct,broadcast and all multicast packets",
                    "Receive all packets (promiscuouos mode)"
                  };

  if (mode > DIM(modeStr))
     return ("??");
  return (modeStr [mode-1]);
}

```

这段代码是一个静态的中断服务函数，称为 "PktInterrupt"。它用于处理硬件中断请求，如 USB 控制中断、网络中断等。函数声明了一个名为 "BOY" 的局部变量，用于跟踪当前的中断状态。

函数根据 DOSX 标志和具体的硬件平台来选择如何处理中断。如果 DOSX 标志为 0 且该平台支持并行操作，那么函数将使用 32 位模式，类似地，如果 DOSX 标志为 0 且该平台不支持并行操作，则使用 64 位模式。

如果支持的硬件平台是 DJGPP，则函数使用 32 位模式，否则使用 64 位模式。函数的实现包括以下几个步骤：

1. 通过调用函数 __dpmi_int() 处理中断请求。
2. 如果中断请求是 USB 控制中断，那么函数需要判断 carry 是否为 0，如果是，那么函数返回真，表示处理成功。
3. 如果中断请求是网络中断，那么函数使用 32 位模式，并按照一系列预定义的中断向量表中的函数实现网络中断处理。
4. 如果支持的硬件平台是 32 位模式，函数需要使用 32 位模式来访问寄存器。如果使用的硬件平台是 64 位模式，函数需要使用 64 位模式来访问寄存器。


```cpp
/**************************************************************************/

LOCAL __inline BOOL PktInterrupt (void)
{
  BOOL okay;

#if (DOSX & PHARLAP)
  _dx_real_int ((UINT)pktInfo.intr, &reg);
  okay = ((reg.flags & 1) == 0);  /* OK if carry clear */

#elif (DOSX & DJGPP)
  __dpmi_int ((int)pktInfo.intr, &reg);
  okay = ((reg.x.flags & 1) == 0);

#elif (DOSX & DOS4GW)
  union  REGS  r;
  struct SREGS s;

  memset (&r, 0, sizeof(r));
  segread (&s);
  r.w.ax  = 0x300;
  r.x.ebx = pktInfo.intr;
  r.w.cx  = 0;
  s.es    = FP_SEG (&reg);
  r.x.edi = FP_OFF (&reg);
  reg.r_flags = 0;
  reg.r_ss = reg.r_sp = 0;     /* DPMI host provides stack */

  int386x (0x31, &r, &r, &s);
  okay = (!r.w.cflag);

```

这段代码是一个 C 语言函数，名为 "intr_handler"，作用是在设备驱动程序（reg）的中断 60h 到 80h（不包括 60h 和 80h）时执行。函数的主要作用是检查设备是否发生了中断，如果是，则根据中断类型设置寄存器（reg) 的某些位（r_flags），然后执行内部函数 (intr)，并将结果存储回寄存器（reg)。

具体来说，代码首先检查给定的寄存器（reg）中 r_flags 位的最低位（&reg)。如果是 0，表示设备没有发生中断，那么函数将返回 0；如果是 1，表示设备发生了中断，那么函数将执行内部函数 (intr)，并将中断的结果存储回寄存器 (reg)。

内部函数 (intr) 接收一个名为 pktInfo 的参数，该参数是一个指向整型结构体 REGPACK 的指针。函数通过 pktInfo.intr 参数获得中断信息，然后将其存储在整型结构体变量 reg 中。函数还返回一个布尔值，表示是否发生了中断，如果发生了，则表示有错误信息，否则表示成功完成了中断处理。

最后，函数根据结果返回一个整型值，表示是否发生了中断。


```cpp
#else
  reg.r_flags = 0;
  intr (pktInfo.intr, (struct REGPACK*)&reg);
  okay = ((reg.r_flags & 1) == 0);
#endif

  if (okay)
       pktInfo.error = NULL;
  else pktInfo.error = PktGetErrorStr (reg.r_dx >> 8);
  return (okay);
}

/**************************************************************************/

/*
 * Search for packet driver at interrupt 60h through 80h. If ASCIIZ
 * string "PKT DRVR" found at offset 3 in the interrupt handler, return
 * interrupt number, else return zero in pktInfo.intr
 */
```

这段代码是一个名为"PktSearchDriver"的函数，用于在 interrupt 0x20（中断 0x20）发生时查找名为"PKT DRVR"的程序。

它首先定义了一个名为"intr"的局部变量，其值为0x20。接着定义了一个名为"found"的布尔变量，其初始值为假。

进入一个无限 while 循环，只要中断没有发生，就会执行其中的一个循环。

在循环中，定义了一个名为"str"的静态字符数组，该数组包含一个名为"PKT DRVR"的字符串，该字符串中共有9个字符。然后定义了一个名为"pktStr"的静态字符数组，该数组与"str"数组长度相同，但仅仅初始化为"PKT DRVR"。

接着定义了一个名为"rp"的静态整型变量，用于存储一个 interrupt 0x20 发生时对应的实参。

然后是一个 if 语句，判断 DOSX 是否为真（即操作系统支持 DOS 风格）。如果是，那么通过调用 _dx_rmiv_get 函数，获取中断 0x20 对应的实参，并将其存储到 rp 变量中。

接着是 ReadingRealMem 函数，用于读取实参，即上面 _dx_rmiv_get 函数返回的值，并将其存储到 str 数组的起始地址处。

最后，在 while 循环中，通过比较 "intr" 和 0xFF，如果 "intr" 不等于 0xFF，说明找到了目标程序，则将"found" 设置为真，跳出循环。否则继续等待中断发生。


```cpp
PUBLIC BOOL PktSearchDriver (void)
{
  BYTE intr  = 0x20;
  BOOL found = FALSE;

  while (!found && intr < 0xFF)
  {
    static char str[12];                 /* 3 + strlen("PKT DRVR") */
    static char pktStr[9] = "PKT DRVR";  /* ASCIIZ string at ofs 3 */
    DWORD  rp;                           /* in interrupt  routine  */

#if (DOSX & PHARLAP)
    _dx_rmiv_get (intr, &rp);
    ReadRealMem (&str, (REALPTR)rp, sizeof(str));

```

这段代码的作用是判断当前处理器是支持DOSX或者DJGPP架构的DPMI（硬件描述符信息）获取中断向量函数。如果是，那么就会执行以下操作：首先获取一个实模式的中断向量，然后将实模式的中断向量与segment和offset16成员组合成一个实模式内存地址，最后通过调用dpmi_get_real_vector函数获取到该实模式内存地址中的字符串。如果当前处理器不支持DOSX或者DJGPP架构，那么就会执行下一个if语句块，通过调用_fmemcpy函数将当前中断向量与getvect函数返回的字符串进行复制，并将其存储到实模式内存地址的str变量中。这样，就可以判断当前处理器支持哪种架构，并根据架构选择相应的操作。


```cpp
#elif (DOSX & DJGPP)
    __dpmi_raddr realAdr;
    __dpmi_get_real_mode_interrupt_vector (intr, &realAdr);
    rp = (realAdr.segment << 4) + realAdr.offset16;
    dosmemget (rp, sizeof(str), &str);

#elif (DOSX & DOS4GW)
    rp = dpmi_get_real_vector (intr);
    memcpy (&str, (void*)rp, sizeof(str));

#else
    _fmemcpy (&str, getvect(intr), sizeof(str));
#endif

    found = memcmp (&str[3],&pktStr,sizeof(pktStr)) == 0;
    intr++;
  }
  pktInfo.intr = (found ? intr-1 : 0);
  return (found);
}


```

这段代码是一个静态函数，名为PktSetAccess。函数内部对传入参数进行处理，然后输出结果。具体解释如下：

1. 函数开始于一个静态段，并包含一个名为PktSetAccess的函数。
2. 函数内部使用reg标签来定义四个寄存器，分别为r_ax，r_bx，r_dx和r_cx。这些寄存器在接下来的代码中被使用。
3. 函数首先将r_ax的值设置为0x0200加上传递给函数的PktInfo.class，即PktInfo.class + 0x200。
4. 函数然后将r_bx的值设置为FFFF，即一个32位的FFFF的值。
5. 函数接下来将r_dx设置为0，可能是为了某些输入数据做好准备。
6. 函数最后设置r_cx为0，同样可能是为了某些输入数据做好准备。
7. 函数接下来的部分包含一个if语句，判断是否为DOSX或PHARLAP版本。如果是DOSX版本，则执行以下操作：
  1. 将rs的值设置为0，rsi的值设置为0，es的值设置为实基的RP_SEG(realBase)。
  2. 将edi的值设置为(WORD)&PktReceiver，即PktReceiver的地址。
  3. 然后执行接下来的代码。
8. 函数最终的输出结果未给出，但可以推测它返回TRUE或FALSE。


```cpp
/**************************************************************************/

static BOOL PktSetAccess (void)
{
  reg.r_ax = 0x0200 + pktInfo.class;
  reg.r_bx = 0xFFFF;
  reg.r_dx = 0;
  reg.r_cx = 0;

#if (DOSX & PHARLAP)
  reg.ds  = 0;
  reg.esi = 0;
  reg.es  = RP_SEG (realBase);
  reg.edi = (WORD) &PktReceiver;

```

这段代码是用来判断当前处理器系列是否支持DOSX和DJGPP。如果支持，则执行以下操作：

1. 将es寄存器中的值设置为rm_mem.rm_segment，并将di寄存器设置为PktReceiver，这会选择一个可用的内存区域用于存储数据。
2. 如果当前处理器系列只支持DOSX，则执行以下操作：
  将ds和si寄存器中的值设置为0，并将es寄存器中的值设置为rm_base_seg，这将在内存中选择一个起始位置。
  将di寄存器设置为PktReceiver，这将成为数据接收器的下一个指针。
3. 如果当前处理器系列只支持DJGPP，则执行以下操作：
  将ds和si寄存器中的值设置为0，并将es寄存器中的值设置为rm_segment，这将在内存中选择一个起始位置。
  将di寄存器设置为PktReceiver，这将成为数据接收器的下一个指针。

如果当前处理器系列不支持DOSX和DJGPP中的任何一个，则执行以下操作：

1. 将ds和si寄存器中的值设置为0，这将清除内存中所有分配的内存区域。
2. 如果当前处理器系列支持FP_SEG，则执行以下操作：
  将es寄存器中的值设置为FP_SEG(&PktReceiver)，这将从内存中选择一个FP_SEG指针，并将其设置为PktReceiver的下一个指针。
3. 如果当前处理器系列不支持FP_SEG，则执行以下操作：
  将es寄存器中的值设置为&PktReceiver，这将在内存中选择一个FP_OFF指针，并将其设置为PktReceiver的下一个指针。


```cpp
#elif (DOSX & DJGPP)
  reg.x.ds = 0;
  reg.x.si = 0;
  reg.x.es = rm_mem.rm_segment;
  reg.x.di = PktReceiver;

#elif (DOSX & DOS4GW)
  reg.r_ds = 0;
  reg.r_si = 0;
  reg.r_es = rm_base_seg;
  reg.r_di = PktReceiver;

#else
  reg.r_ds = 0;
  reg.r_si = 0;
  reg.r_es = FP_SEG (&PktReceiver);
  reg.r_di = FP_OFF (&PktReceiver);
```

这段代码是一个 C 语言程序，它实现了异常处理和中断函数。

该程序的主要作用是在发生异常情况时返回一个布尔值，用于指示程序是否在异常中断中。

程序首先定义了一个名为 PktInterrupt 的函数，但这个函数没有具体的功能，只是一个局部变量。

接下来，程序定义了一个名为 PktReleaseHandle 的函数，它的参数为 handle，表示一个中断请求标记(Windows中的 INT 类型)。

在这个函数中，程序先将 reg.r_ax 赋值为 0x0300，并将 handle 存储在 reg.r_bx 中。

接着，程序调用 PktInterrupt 函数，并将返回值存储在 tr条款控制寄存器中。

如果返回值为 TRUE，则说明程序成功执行了 PktInterrupt 函数，即在异常中断中。此时，程序将返回 TRUE。

否则，程序将返回 FALSE。在这种情况下，程序将执行 PktReleaseHandle 函数，即将 handle 返回给系统，并将 handle 存储在 reg.r_ax 中。

注意，这段代码中只有一行代码使用了 `#ifdef` 和 `#endif` 预处理指令，用于定义了一个名为 PktInterrupt 的函数。


```cpp
#endif

  if (!PktInterrupt())
     return (FALSE);

  pktInfo.handle = reg.r_ax;
  return (TRUE);
}

/**************************************************************************/

PUBLIC BOOL PktReleaseHandle (WORD handle)
{
  reg.r_ax = 0x0300;
  reg.r_bx = handle;
  return PktInterrupt();
}

```

这段代码是一个名为 "PktTransmit" 的函数，它接受一个名为 "eth" 的指针和一个名为 "len" 的整数参数。它的作用是检查输入的长度是否大于数据帧的最大传输单元（MTU）并返回 FALSE。

如果输入长度小于或等于 MTU，函数将返回 TRUE。否则，函数会执行以下操作：

1. 将函数参数 "len" 存储在 "reg.r_cx" 寄存器中，以便在函数内使用。

2. 设置 "reg.r_ax" 寄存器为 0x0400，这是功能 4，表示发送数据包。

3. 如果操作系统支持 DOS（磁盘操作系统），函数将调用 "dosmemput" 函数，将输入数据 "eth" 和 "len" 存储在内存 "eth" 中，并从内存中分配一个名为 "pktTxBuf" 的缓冲区。

4. 将 "rm_mem.rm_segment" 成员点的地址（即 "pktTxBuf" 的起始地址）存储在 "reg.x.si" 寄存器中，并将输入数据 "eth" 的偏移量设置为 "len"。

5. 如果 "DOSX" 和 "DJGPP" 设置为 1，函数将使用 "rm_mem.rm_sys" 函数将内存中的数据写入到 "eth" 变量中。

6. 函数返回 TRUE，表示数据包发送成功。


```cpp
/**************************************************************************/

PUBLIC BOOL PktTransmit (const void *eth, int len)
{
  if (len > ETH_MTU)
     return (FALSE);

  reg.r_ax = 0x0400;             /* Function 4, send pkt */
  reg.r_cx = len;                /* total size of frame  */

#if (DOSX & DJGPP)
  dosmemput (eth, len, realBase+pktTxBuf);
  reg.x.ds = rm_mem.rm_segment;  /* DOS data segment and */
  reg.x.si = pktTxBuf;           /* DOS offset to buffer */

```

这段代码是一个条件分支语句，根据DOSX和DOS4GW以及DOSX和PHARLAP中的任意一个为真，执行不同的内存复制操作。具体来说：

1. 如果DOSX和DOS4GW都为真，那么执行memcpy操作，将ethereal_pkt结构中的内容复制到内存中的&pktTxBuf，同时将eth缓冲区中的长度作为参数传递给memcpy函数。此时，reg变量中的ds和si指向的是内存中的起始地址和结束地址，以便于之后的操作。

2. 如果DOSX和PHARLAP都为真，那么执行memcpy操作，将ethereal_pkt结构中的内容复制到&pktTxBuf，同时将PHARLAP中获取到的以偏移量为长度的&pktTxBuf的偏移量作为参数传递给memcpy函数。

3. 如果DOSX和DOS4GW都为假，或者PHARLAP为假，或者DOSX和PHARLAP都为假，那么执行reg变量中的操作，将内存中的起始地址和结束地址作为参数传递给reg.r_ds和reg.r_si，即将ethereal_pkt结构中的内容复制到内存中的起始地址和结束地址。

4. 在函数结束处，通过调用PktInterrupt()函数来使程序挂起当前中断，以便于系统进行一些操作，如关闭设备驱动程序等。


```cpp
#elif (DOSX & DOS4GW)
  memcpy ((void*)(realBase+pktTxBuf), eth, len);
  reg.r_ds = rm_base_seg;
  reg.r_si = pktTxBuf;

#elif (DOSX & PHARLAP)
  memcpy (&pktTxBuf, eth, len);
  reg.r_ds = FP_SEG (&pktTxBuf);
  reg.r_si = FP_OFF (&pktTxBuf);

#else
  reg.r_ds = FP_SEG (eth);
  reg.r_si = FP_OFF (eth);
#endif

  return PktInterrupt();
}

```



这段代码是一个条件判断，根据DOSX系统的版本来判断是否支持大端或小端下标。如果DOSX系统版本为13.0或更高版本，则只支持小端下标；否则，支持大端下标。

具体来说，代码中定义了一个名为CheckElement的函数，该函数接收一个指向RX_ELEMENT结构体的指针变量rx，并检查该指针指向的元素是否符合某些条件。如果元素不符合条件，函数将返回FALSE，否则返回TRUE。

如果DOSX系统版本为13.0或更高版本，则定义了一个名为CheckElement的函数，该函数与上述函数同名但参数类型不同，需要使用_far关键字修饰。这个函数与上面定义的函数不同之处在于，它接收的是一个名为RX_ELEMENT的结构体类型，而不是一个名为RX_ELEMENT的指针类型。这个函数的作用与上面定义的函数相同，但可能会对不同的系统版本产生不同的行为。

函数的具体实现包括两个判断：首先判断元素是否符合某些条件，如果不符合条件，则返回FALSE并返回一个错误码。如果符合条件，则继续执行下面的判断。

第一个判断是检查元素是否与pktInfo.handle指向的元素的firstCount和secondCount是否相同。如果这两个元素的计数值不同，则返回FALSE并返回一个错误码。

第二个判断是检查元素的大小是否超过了ETH_MAX，如果是，则返回FALSE并返回一个错误码。

如果上述两个判断都符合条件，则函数将返回TRUE。


```cpp
/**************************************************************************/

#if (DOSX & (DJGPP|DOS4GW))
LOCAL __inline BOOL CheckElement (RX_ELEMENT *rx)
#else
LOCAL __inline BOOL CheckElement (RX_ELEMENT _far *rx)
#endif
{
  WORD count_1, count_2;

  /*
   * We got an upcall to the same RMCB with wrong handle.
   * This can happen if we failed to release handle at program exit
   */
  if (rx->handle != pktInfo.handle)
  {
    pktInfo.error = "Wrong handle";
    intStat.wrongHandle++;
    PktReleaseHandle (rx->handle);
    return (FALSE);
  }
  count_1 = rx->firstCount;
  count_2 = rx->secondCount;

  if (count_1 != count_2)
  {
    pktInfo.error = "Bad sync";
    intStat.badSync++;
    return (FALSE);
  }
  if (count_1 > ETH_MAX)
  {
    pktInfo.error = "Large esize";
    intStat.tooLarge++;
    return (FALSE);
  }
```

这段代码的作用是判断一个数据包是否成功接收。数据包接收成功后，会执行以下操作：

1. 检查数据包的大小是否小于期望的最小大小，如果是，将错误信息添加到 pktInfo.error 字段，并将 tooSmall 计数器加 1。
2. 如果数据包大小符合期望，则返回 TRUE，否则返回 FALSE。

简而言之，这段代码会检查数据包的大小是否符合期望，如果不符合，会添加错误信息并增加计数器。如果数据包成功接收，则返回 TRUE，否则返回 FALSE。


```cpp
#if 0
  if (count_1 < ETH_MIN)
  {
    pktInfo.error = "Small esize";
    intStat.tooSmall++;
    return (FALSE);
  }
#endif
  return (TRUE);
}

/**************************************************************************/

PUBLIC BOOL PktTerminHandle (WORD handle)
{
  reg.r_ax = 0x0500;
  reg.r_bx = handle;
  return PktInterrupt();
}

```

这段代码定义了两个公共函数，名为`PktResetInterface`和`PktSetReceiverMode`。

`PktResetInterface`函数的作用是重置接收设备（设备类为PD_SLIP或PD_PPP）时的寄存器设置。具体来说，该函数会将`reg.r_ax`设置为`0x0700`，将`reg.r_bx`设置为接收设备的硬件ID，然后调用`PktInterrupt()`函数，如果成功则返回`TRUE`，否则返回`FALSE`。

`PktSetReceiverMode`函数的作用是在接收设备设置为PD_SLIP或PD_PPP时，设置接收模式的函数。函数首先检查设备类，如果是PD_SLIP或PD_PPP，则设置`reg.r_cx`为要设置的模式的硬件ID，然后调用`PktInterrupt()`函数，如果成功则返回`TRUE`，否则返回`FALSE`。最后，将`receiveMode`变量设置为设置的模式，返回`TRUE`或`FALSE`。


```cpp
/**************************************************************************/

PUBLIC BOOL PktResetInterface (WORD handle)
{
  reg.r_ax = 0x0700;
  reg.r_bx = handle;
  return PktInterrupt();
}

/**************************************************************************/

PUBLIC BOOL PktSetReceiverMode (PKT_RX_MODE mode)
{
  if (pktInfo.class == PD_SLIP || pktInfo.class == PD_PPP)
     return (TRUE);

  reg.r_ax = 0x1400;
  reg.r_bx = pktInfo.handle;
  reg.r_cx = (WORD)mode;

  if (!PktInterrupt())
     return (FALSE);

  receiveMode = mode;
  return (TRUE);
}

```

这段代码是一个用于获取接收模式的函数，接收模式存储在变量 `mode` 中。函数的实现过程如下：

1. 初始化寄存器 `reg`，将 `r_ax` 寄存器设置为 0x1500，将 `r_bx` 寄存器设置为 `pktInfo.handle`，即接收机的数据缓冲区的起始地址。
2. 调用一个名为 `PktInterrupt` 的函数，如果这个函数返回 FALSE，说明发生了中断，可以获取到更多的信息。
3. 将返回值 `TRUE` 存储在 `mode` 变量中，意味着函数成功，返回了一个可接受的模式。
4. 返回值类型说明 `mode` 变量是 `PKT_RX_MODE` 类型，表示接收机支持的模式。


```cpp
/**************************************************************************/

PUBLIC BOOL PktGetReceiverMode (PKT_RX_MODE *mode)
{
  reg.r_ax = 0x1500;
  reg.r_bx = pktInfo.handle;

  if (!PktInterrupt())
     return (FALSE);

  *mode = reg.r_ax;
  return (TRUE);
}

/**************************************************************************/

```

这段代码是一个C语言程序，它定义了两个静态成员变量和一个静态函数。以下是它的作用和使用：

1. `initialStat`：是一个静态成员变量，用于在程序启动时记录一些统计信息。它没有被定义为`READ_ONLY`或`WRITE_ONLY`类型，因此可以被任何函数或变量访问和修改。

2. `resetStat`：是一个静态成员变量，用于记录是否已经重置了统计信息。它被定义为`BOOL`类型，并且没有被定义为`READ_ONLY`或`WRITE_ONLY`类型，因此也可以被任何函数或变量访问和修改。

3. `PktGetStatistics`函数：它是一个静态函数，用于获取当前的统计信息并将其返回。它使用`PktInterrupt()`函数来暂停当前正在进行的I/O操作，然后执行一些操作来获取统计信息。如果当前正在进行的I/O操作没有挂起，它将调用`ReadRealMem()`函数来从设备驱动程序或系统内存中读取统计信息。如果当前正在进行的I/O操作挂起了硬件或系统事件，它将调用`dosmemget()`函数来读取统计信息，这个函数在DOS X和Java虚拟机中提供。

该程序的作用是获取系统统计信息，例如CPU和内存的使用情况，并将其显示在控制台上。


```cpp
static PKT_STAT initialStat;         /* statistics at startup */
static BOOL     resetStat = FALSE;   /* statistics reset ? */

PUBLIC BOOL PktGetStatistics (WORD handle)
{
  reg.r_ax = 0x1800;
  reg.r_bx = handle;

  if (!PktInterrupt())
     return (FALSE);

#if (DOSX & PHARLAP)
  ReadRealMem (&pktStat, DOS_ADDR(reg.ds,reg.esi), sizeof(pktStat));

#elif (DOSX & DJGPP)
  dosmemget (DOS_ADDR(reg.x.ds,reg.x.si), sizeof(pktStat), &pktStat);

```

这段代码是一个C语言函数，名为`PktSessStatistics`。它对传输包统计信息进行计算和更新，以满足DOSX和DOS4GW设备。以下是代码的功能解释：

1. 首先检查当前处理器是DOSX还是DOS4GW。如果是DOSX，则执行以下代码：

```cppc
#elif (DOSX & DOS4GW)
 memcpy (&pktStat, (void*)DOS_ADDR(reg.r_ds,reg.r_si), sizeof(pktStat));
```
这段代码首先检查当前处理器是否支持DOSX和DOS4GW。如果是DOSX，则使用`memcpy`函数将`pktStat`向量与`DOS_ADDR`函数的返回值进行内存对齐，并将其复制到`pktStat`中。

2. 如果当前处理器不是DOSX，则执行以下代码：

```cppc
 _fmemcpy (&pktStat, MK_FP(reg.r_ds,reg.r_si), sizeof(pktStat));
```
这段代码首先检查当前处理器是否支持内存对齐。如果是，则使用`_fmemcpy`函数将`pktStat`向量与`MK_FP`函数的返回值进行内存对齐，并将其复制到`pktStat`中。

3. 调用`PktGetStatistics`函数，并根据需要更新`pktStat`中的统计信息。

4. 如果需要重置统计信息，则执行以下操作：

```cppc
 pktStat.inPackets  -= initialStat.inPackets;
 pktStat.outPackets -= initialStat.outPackets;
 pktStat.inBytes    -= initialStat.inBytes;
 pktStat.outBytes   -= initialStat.outBytes;
 pktStat.inErrors   -= initialStat.inErrors;
 pktStat.outErrors  -= initialStat.outErrors;
 pktStat.outErrors  -= initialStat.outErrors;
 pktStat.lost       -= initialStat.lost;
```
这段代码首先调用`PktGetStatistics`函数获取原始统计信息。然后，根据需要将`pktStat`中的某些成员变量设置为初始值，最后返回更新后的统计信息。

这段代码的作用是对传输包统计信息进行计算和更新，以满足DOSX和DOS4GW设备。


```cpp
#elif (DOSX & DOS4GW)
  memcpy (&pktStat, (void*)DOS_ADDR(reg.r_ds,reg.r_si), sizeof(pktStat));

#else
  _fmemcpy (&pktStat, MK_FP(reg.r_ds,reg.r_si), sizeof(pktStat));
#endif

  return (TRUE);
}

/**************************************************************************/

PUBLIC BOOL PktSessStatistics (WORD handle)
{
  if (!PktGetStatistics(pktInfo.handle))
     return (FALSE);

  if (resetStat)
  {
    pktStat.inPackets  -= initialStat.inPackets;
    pktStat.outPackets -= initialStat.outPackets;
    pktStat.inBytes    -= initialStat.inBytes;
    pktStat.outBytes   -= initialStat.outBytes;
    pktStat.inErrors   -= initialStat.inErrors;
    pktStat.outErrors  -= initialStat.outErrors;
    pktStat.outErrors  -= initialStat.outErrors;
    pktStat.lost       -= initialStat.lost;
  }
  return (TRUE);
}

```

这段代码定义了两个函数，然后在另外两个函数中进行了调用。现在我将解释这两个函数的作用。

1. PktResetStatistics函数

该函数的目的是在使用给定的处理程序（PktInfo）之前，确保该程序已经准备好。为此，该函数首先调用PktGetStatistics函数，如果该函数返回FALSE，则表示处理程序没有准备好，从而返回TRUE。接下来，该函数将默认的统计信息（initialStat）复制到目标统计信息（pktStat）中，并将resetStat设置为TRUE。最后，该函数返回TRUE，表示成功执行了操作。

2. PktGetAddress函数

该函数的目的是从给定的以太网地址（addr）中获取物理地址。为此，该函数使用三个寄存器（reg）中的一个来设置一个累加器（reg.r_ax）的值，该值将包含从以太网接口中读取的物理地址。然后，该函数将一个指向以太网地址的指针（addr）存储在变量中。

请注意，该函数的实现非常简单，仅使用两个硬件寄存器和一个用于存储目的的指针。这表明它的功能非常有限，可能只适用于一些非常简单的场景。


```cpp
/**************************************************************************/

PUBLIC BOOL PktResetStatistics (WORD handle)
{
  if (!PktGetStatistics(pktInfo.handle))
     return (FALSE);

  memcpy (&initialStat, &pktStat, sizeof(initialStat));
  resetStat = TRUE;
  return (TRUE);
}

/**************************************************************************/

PUBLIC BOOL PktGetAddress (ETHER *addr)
{
  reg.r_ax = 0x0600;
  reg.r_bx = pktInfo.handle;
  reg.r_cx = sizeof (*addr);

```

这段代码是一个if语句，根据DOSX和DJGPP操作系统版本是否支持，执行不同的操作。

具体来说，如果DOSX和DJGPP都支持，那么将reg.x.es赋值为rm_mem.rm_segment，reg.x.di赋值为pktTemp；如果DOSX和DJGPP不支持，那么将reg.r_es赋值为rm_base_seg，reg.r_di赋值为pktTemp。然后判断是否为真，如果是，则执行以下操作：读取指定位置的内存，然后跳转到result的位置。

最后，如果PktInterrupt为真，则退出程序。


```cpp
#if (DOSX & DJGPP)
  reg.x.es = rm_mem.rm_segment;
  reg.x.di = pktTemp;
#elif (DOSX & DOS4GW)
  reg.r_es = rm_base_seg;
  reg.r_di = pktTemp;
#else
  reg.r_es = FP_SEG (&pktTemp);
  reg.r_di = FP_OFF (&pktTemp);  /* ES:DI = address for result */
#endif

  if (!PktInterrupt())
     return (FALSE);

#if (DOSX & PHARLAP)
  ReadRealMem (addr, realBase + (WORD)&pktTemp, sizeof(*addr));

```

这段代码是一个C语言函数，名为“memcpy_ex”。它根据DOSX和DJGPP软件环境来执行不同的内存复制操作。

在代码开始部分，定义了一个名为“DOSX”和“DJGPP”的布尔变量。这些变量可能代表某个与软件版本相关的位，但具体表示意义取决于上下文。在接下来的代码中，根据这两个变量的值，使用“#elif”为每个情况分支。

第一个分支包含两行代码，根据DOSX是否为真以及DJGPP是否为真来决定是否执行相应的memcpy函数。如果DOSX为真且DJGPP为真，那么将实Base加上一个大小为sizeof(*addr)的内存区域，并使用memcpy函数将其复制到内存中的地址。

第二个分支包含两行代码，根据DOSX是否为真以及DJGPP是否为真来决定是否执行相应的memcpy函数。如果DOSX为真，那么将实Base加上一个大小为sizeof(*addr)的内存区域，并使用memcpy函数将其复制到内存中的地址。如果DJGPP为真，那么将实Base加上一个大小为sizeof(*addr)的内存区域，并使用memcpy函数将其复制到内存中的地址。

如果DOSX和DJGPP都不为真，那么将实Base上的内存区域复制到内存中的地址，使用memcpy函数将其复制到实Base上的地址。

最后，函数返回TRUE，表明复制操作成功。


```cpp
#elif (DOSX & DJGPP)
  dosmemget (realBase+pktTemp, sizeof(*addr), addr);

#elif (DOSX & DOS4GW)
  memcpy (addr, (void*)(realBase+pktTemp), sizeof(*addr));

#else
  memcpy ((void*)addr, &pktTemp, sizeof(*addr));
#endif

  return (TRUE);
}

/**************************************************************************/

```

这段代码是一个名为 "PktSetAddress" 的函数，它的作用是设置一个以太数据包（ETH）的发送地址。该函数接受一个指针变量 "addr"，该变量必须是一个指向 Ether 类型数据的指针。

以下是可能的调用者：

1. 如果你使用的是 DOSX 架构，并且定义了 "DOSX & PHARLAP"，那么将调用 "WriteRealMem" 函数，并将添加的地址复制到 "realBase" 加上该地址的偏移量。
2. 如果你使用的是 DOSX 架构，且定义了 "DOSX & DJGPP"，那么将调用 "dosmemput" 函数，并将添加的地址复制到 "pktTemp" 变量上。
3. 如果你使用的是 DJGPP 架构，那么将调用 "dosmemput" 函数，并将添加的地址复制到 "pktTemp" 变量上。
4. 如果你使用的是更高版本的 DOSX 架构，或者定义了 "DOSX & DOS4GW"，那么将调用 "memcpy" 函数，并将添加的地址复制到 "pktTemp" 变量上。


```cpp
PUBLIC BOOL PktSetAddress (const ETHER *addr)
{
  /* copy addr to real-mode scrath area */

#if (DOSX & PHARLAP)
  WriteRealMem (realBase + (WORD)&pktTemp, (void*)addr, sizeof(*addr));

#elif (DOSX & DJGPP)
  dosmemput (addr, sizeof(*addr), realBase+pktTemp);

#elif (DOSX & DOS4GW)
  memcpy ((void*)(realBase+pktTemp), addr, sizeof(*addr));

#else
  memcpy (&pktTemp, (void*)addr, sizeof(*addr));
```

这段代码是一个ARM汇编代码，它定义了一个硬件寄存器reg和一个指向内存地址的指针reg。然后，它根据DOSX和DJGPP架构设置了不同的段寄存器和DI寄存器，以便在不同的操作系统环境中执行代码。

具体来说，如果当前正在运行DOSX，则将rm_mem.rm_segment作为ES寄存器的DOS偏移量，并将pktTemp作为DI寄存器的DOS偏移量。如果正在运行DOS4GW，则将rm_base_seg作为ES寄存器的DOS偏移量，并将pktTemp作为DI寄存器的DOS偏移量。如果正在运行FP_SEG或FP_OFF，则根据所提供的函数指针执行相应的函数，并将返回值作为ARM汇编代码的最后一个操作数。

最后，该代码通过调用PktInterrupt函数来输出中断标志，以便通知操作系统暂停当前正在执行的任务。


```cpp
#endif

  reg.r_ax = 0x1900;
  reg.r_cx = sizeof (*addr);      /* address length       */

#if (DOSX & DJGPP)
  reg.x.es = rm_mem.rm_segment;   /* DOS offset to param  */
  reg.x.di = pktTemp;             /* DOS segment to param */
#elif (DOSX & DOS4GW)
  reg.r_es = rm_base_seg;
  reg.r_di = pktTemp;
#else
  reg.r_es = FP_SEG (&pktTemp);
  reg.r_di = FP_OFF (&pktTemp);
#endif

  return PktInterrupt();
}

```



这段代码是一个名为 "PktGetDriverInfo" 的函数，其作用是获取平台设备驱动程序的版本信息并返回。以下是该函数的步骤解释：

1. 定义变量 pktInfo，包括 majVer、minVer、name 三个整型变量以及一个字符串指针变量 name。

2. 定义 reg 变量，包括 r_ax 和 r_bx 两个寄存器，reg 指向的是 01FF。

3. 判断是否发生了 pktInterrupt 事件，如果没有，则继续执行下一次。

4. 读取 reg 寄存器中的值，并将其与 0xFF 进行按位与操作，得到 pktInfo.number。

5. 读取 reg 寄存器中的低 8 位，并将其与 0xFF 进行按位与操作，得到 pktInfo.class。

6. 将 pktInfo.number 和 pktInfo.class 存储到 pktInfo 变量中。

7. 如果发生了 pktInterrupt 事件，函数返 FALSE。

综上所述，该函数的作用是获取平台设备驱动程序的版本信息并返回，其中包括设备类型和版本号。


```cpp
/**************************************************************************/

PUBLIC BOOL PktGetDriverInfo (void)
{
  pktInfo.majVer = 0;
  pktInfo.minVer = 0;
  memset (&pktInfo.name, 0, sizeof(pktInfo.name));
  reg.r_ax = 0x01FF;
  reg.r_bx = 0;

  if (!PktInterrupt())
     return (FALSE);

  pktInfo.number = reg.r_cx & 0xFF;
  pktInfo.class  = reg.r_cx >> 8;
```

这段代码的作用是定义一个名为 `pktInfo` 的结构体，其中包含以下字段：

1. `minVer`：用 `reg.r_bx % 10` 得到的8位整数，取最低位作为 `pktInfo.minVer`。
2. `majVer`：用 `reg.r_bx / 10` 得到的16位整数，作为 `pktInfo.majVer`。
3. `funcs`：用 `reg.r_ax & 0xFF` 获取的8位二进制数，取低8位作为 `pktInfo.funcs`。
4. `type`：用 `reg.r_dx & 0xFF` 获取的8位二进制数，取低8位作为 `pktInfo.type`。
5. `name`：用 `ReadRealMem` 函数从内存中读取的串，以 `pktInfo.name` 命名并存储。

具体地，如果 `DOSX` 和 `PHARLAP` 至少有一个为真，则会执行以下操作：

1. 如果 `DOSX` 为真，则会执行 `ReadRealMem` 函数，从内存中读取一个串，以 `pktInfo.name` 命名并存储。
2. 如果 `PHARLAP` 为真，则并不会执行 `ReadRealMem` 函数，因为 `DOSX` 和 `PHARLAP` 同时为真时，执行的是第一个操作，而不会再次执行。

如果 `DOSX` 和 `DJGPP` 中至少有一个为真，则会执行以下操作：

1. 如果 `DOSX` 为真，则会执行 `dosmemget` 函数，从内存中读取一个串，以 `pktInfo.name` 命名并存储。
2. 如果 `DJGPP` 为真，则并不会执行 `dosmemget` 函数，因为 `DOSX` 和 `DJGPP` 同时为真时，执行的是第一个操作，而不会再次执行。


```cpp
#if 0
  pktInfo.minVer = reg.r_bx % 10;
  pktInfo.majVer = reg.r_bx / 10;
#else
  pktInfo.majVer = reg.r_bx;  // !!
#endif
  pktInfo.funcs  = reg.r_ax & 0xFF;
  pktInfo.type   = reg.r_dx & 0xFF;

#if (DOSX & PHARLAP)
  ReadRealMem (&pktInfo.name, DOS_ADDR(reg.ds,reg.esi), sizeof(pktInfo.name));

#elif (DOSX & DJGPP)
  dosmemget (DOS_ADDR(reg.x.ds,reg.x.si), sizeof(pktInfo.name), &pktInfo.name);

```

这段代码是一个C语言的if语句，针对两种不同的DOS版本（DOSX和DOS4GW）进行了检查。

如果当前系统支持DOSX或者DOS4GW，那么就会执行第一个else子句，该子句使用_fmemcpy函数从内存中读取数据并将其复制到reg.r_ds所指向的内存区域，然后使用memcpy函数将数据写回到内存中的pktInfo.name中，最后返回TRUE表示函数执行成功。

否则，如果当前系统不支持DOSX或者DOS4GW，那么就会执行if语句判断，此时不执行else子句，而是直接使用memcpy函数将数据直接复制到pktInfo.name中，同样需要返回TRUE表示函数执行成功。

总结：该代码的作用是检查当前系统是否支持DOSX或者DOS4GW，并返回相应的TRUE或FALSE。


```cpp
#elif (DOSX & DOS4GW)
  memcpy (&pktInfo.name, (void*)DOS_ADDR(reg.r_ds,reg.r_si), sizeof(pktInfo.name));

#else
  _fmemcpy (&pktInfo.name, MK_FP(reg.r_ds,reg.r_si), sizeof(pktInfo.name));
#endif
  return (TRUE);
}

/**************************************************************************/

PUBLIC BOOL PktGetDriverParam (void)
{
  reg.r_ax = 0x0A00;

  if (!PktInterrupt())
     return (FALSE);

```

这段代码是一个if语句，根据DOSX和PHARLAP的组合情况执行不同的内存读写操作。

具体来说：

1. 如果DOSX和PHARLAP同时为真，那么执行第3个判断，从ES寄存器的一个连续内存区域读取一个字节并将其复制到pktInfo.majVer中，然后返回TRUE。
2. 如果DOSX和DJGPP同时为真，那么执行第2个判断，使用dosmemget函数从ES寄存器的另一个连续内存区域读取一个字节并将其复制到pktInfo.majVer中，然后返回TRUE。
3. 如果DOSX和DOS4GW同时为真，那么执行第1个判断，从ES寄存器的连续内存区域中读取一个字节并将其复制到pktInfo.majVer中，然后返回TRUE。
4. 如果DOSX的任何组合都不是真，那么执行第4个判断，将ES寄存器中指定位置的值复制到pktInfo.majVer中，然后返回TRUE。

这段代码的作用是读取或写入一个字节的值，并将其复制到目标结构体中，根据DOSX和PHARLAP的组合情况来决定使用哪种内存读写操作。


```cpp
#if (DOSX & PHARLAP)
  ReadRealMem (&pktInfo.majVer, DOS_ADDR(reg.es,reg.edi), PKT_PARAM_SIZE);

#elif (DOSX & DJGPP)
  dosmemget (DOS_ADDR(reg.x.es,reg.x.di), PKT_PARAM_SIZE, &pktInfo.majVer);

#elif (DOSX & DOS4GW)
  memcpy (&pktInfo.majVer, (void*)DOS_ADDR(reg.r_es,reg.r_di), PKT_PARAM_SIZE);

#else
  _fmemcpy (&pktInfo.majVer, MK_FP(reg.r_es,reg.r_di), PKT_PARAM_SIZE);
#endif
  return (TRUE);
}

```

0 是否在传输过程中丢包？答案是肯定的。从代码中可以看出，当 `PktQueueBusy()` 函数返回时，表示在传输过程中丢包。在函数中，我们发送了一个 `RX_ELEMENT` 类型的数据包，但是在实际传输中可能会丢失数据包，因此函数需要返回一个表示数据包丢包数量的结果。


```cpp
/**************************************************************************/

#if (DOSX & PHARLAP)
  PUBLIC int PktReceive (BYTE *buf, int max)
  {
    WORD inOfs  = *rxInOfsFp;
    WORD outOfs = *rxOutOfsFp;

    if (outOfs != inOfs)
    {
      RX_ELEMENT _far *head = (RX_ELEMENT _far*)(protBase+outOfs);
      int size, len = max;

      if (CheckElement(head))
      {
        size = min (head->firstCount, sizeof(RX_ELEMENT));
        len  = min (size, max);
        _fmemcpy (buf, &head->destin, len);
      }
      else
        size = -1;

      outOfs += sizeof (RX_ELEMENT);
      if (outOfs > LAST_RX_BUF)
          outOfs = FIRST_RX_BUF;
      *rxOutOfsFp = outOfs;
      return (size);
    }
    return (0);
  }

  PUBLIC void PktQueueBusy (BOOL busy)
  {
    *rxOutOfsFp = busy ? (*rxInOfsFp + sizeof(RX_ELEMENT)) : *rxInOfsFp;
    if (*rxOutOfsFp > LAST_RX_BUF)
        *rxOutOfsFp = FIRST_RX_BUF;
    *(DWORD _far*)(protBase + (WORD)&pktDrop) = 0;
  }

  PUBLIC WORD PktBuffersUsed (void)
  {
    WORD inOfs  = *rxInOfsFp;
    WORD outOfs = *rxOutOfsFp;

    if (inOfs >= outOfs)
       return (inOfs - outOfs) / sizeof(RX_ELEMENT);
    return (NUM_RX_BUF - (outOfs - inOfs) / sizeof(RX_ELEMENT));
  }

  PUBLIC DWORD PktRxDropped (void)
  {
    return (*(DWORD _far*)(protBase + (WORD)&pktDrop));
  }

```

This is a module for the Personal computer以前的DOS中断 0x411, also known as the Private Bytecode (PBC) DOS中断 0x411. It appears to be related to the handling of packets being sent over a network.

The module defines several functions and variables:

* `PktQueueBusy()`: This function appears to be used to queue a packet for transport over a network. It takes a single argument, `busy`, which is a boolean value indicating whether the packet should be queued or not. The function returns the size of the packet queue (number of packets) if the packet was queued, or `0` if the queue was not full.
* `PktBuffersUsed()`: This function returns the number of packets that have been queued for transport over the network. It seems to be based on the number of packets that have been queued, and the number of packets that have been successfully dropped.
* `PktDrop()`: This function returns the index of the first packet that has been dropped by the program, if any.
* `RxInOfs()`: This function returns the current index of the first packet that has been received by the program, if any.
* `RxOutOfs()`: This function returns the current index of the last packet that is being sent by the program, if any.
* `RxElems()`: This function returns the number of elements in the current packet, if any.
* `send_packet()`: This function appears to be the main function for sending a packet over a network. It takes two arguments: the packet data and the index of the packet to send. It returns the success status (0 for success, 1 for failure).
* `receive_packet()`: This function appears to be the main function for receiving a packet over a network. It takes two arguments: the packet data and the index of the packet to receive. It returns the success status (0 for success, 1 for failure).
* `send_successful()`: This function appears to be used to keep track of the number of packets that have been successfully sent over a network. It takes no arguments and returns the success status (0 for success, 1 for failure).
* `send_failed()`: This function appears to be used to keep track of the number of packets that have failed to be sent over a network. It takes no arguments and returns the success status (0 for success, 1 for failure).


```cpp
#elif (DOSX & DJGPP)
  PUBLIC int PktReceive (BYTE *buf, int max)
  {
    WORD ofs = _farpeekw (_dos_ds, realBase+rxOutOfs);

    if (ofs != _farpeekw (_dos_ds, realBase+rxInOfs))
    {
      RX_ELEMENT head;
      int  size, len = max;

      head.firstCount  = _farpeekw (_dos_ds, realBase+ofs);
      head.secondCount = _farpeekw (_dos_ds, realBase+ofs+2);
      head.handle      = _farpeekw (_dos_ds, realBase+ofs+4);

      if (CheckElement(&head))
      {
        size = min (head.firstCount, sizeof(RX_ELEMENT));
        len  = min (size, max);
        dosmemget (realBase+ofs+6, len, buf);
      }
      else
        size = -1;

      ofs += sizeof (RX_ELEMENT);
      if (ofs > LAST_RX_BUF)
           _farpokew (_dos_ds, realBase+rxOutOfs, FIRST_RX_BUF);
      else _farpokew (_dos_ds, realBase+rxOutOfs, ofs);
      return (size);
    }
    return (0);
  }

  PUBLIC void PktQueueBusy (BOOL busy)
  {
    WORD ofs;

    disable();
    ofs = _farpeekw (_dos_ds, realBase+rxInOfs);
    if (busy)
       ofs += sizeof (RX_ELEMENT);

    if (ofs > LAST_RX_BUF)
         _farpokew (_dos_ds, realBase+rxOutOfs, FIRST_RX_BUF);
    else _farpokew (_dos_ds, realBase+rxOutOfs, ofs);
    _farpokel (_dos_ds, realBase+pktDrop, 0UL);
    enable();
  }

  PUBLIC WORD PktBuffersUsed (void)
  {
    WORD inOfs, outOfs;

    disable();
    inOfs  = _farpeekw (_dos_ds, realBase+rxInOfs);
    outOfs = _farpeekw (_dos_ds, realBase+rxOutOfs);
    enable();
    if (inOfs >= outOfs)
       return (inOfs - outOfs) / sizeof(RX_ELEMENT);
    return (NUM_RX_BUF - (outOfs - inOfs) / sizeof(RX_ELEMENT));
  }

  PUBLIC DWORD PktRxDropped (void)
  {
    return _farpeekl (_dos_ds, realBase+pktDrop);
  }

```

0 无法完成此请求。这个功能需要使用特殊的头文件和函数，但是缺少这些文件。请确认已经定义了这些文件，或者正在按照说明书中说明的方式编写代码。


```cpp
#elif (DOSX & DOS4GW)
  PUBLIC int PktReceive (BYTE *buf, int max)
  {
    WORD ofs = *(WORD*) (realBase+rxOutOfs);

    if (ofs != *(WORD*) (realBase+rxInOfs))
    {
      RX_ELEMENT head;
      int  size, len = max;

      head.firstCount  = *(WORD*) (realBase+ofs);
      head.secondCount = *(WORD*) (realBase+ofs+2);
      head.handle      = *(WORD*) (realBase+ofs+4);

      if (CheckElement(&head))
      {
        size = min (head.firstCount, sizeof(RX_ELEMENT));
        len  = min (size, max);
        memcpy (buf, (const void*)(realBase+ofs+6), len);
      }
      else
        size = -1;

      ofs += sizeof (RX_ELEMENT);
      if (ofs > LAST_RX_BUF)
           *(WORD*) (realBase+rxOutOfs) = FIRST_RX_BUF;
      else *(WORD*) (realBase+rxOutOfs) = ofs;
      return (size);
    }
    return (0);
  }

  PUBLIC void PktQueueBusy (BOOL busy)
  {
    WORD ofs;

    _disable();
    ofs = *(WORD*) (realBase+rxInOfs);
    if (busy)
       ofs += sizeof (RX_ELEMENT);

    if (ofs > LAST_RX_BUF)
         *(WORD*) (realBase+rxOutOfs) = FIRST_RX_BUF;
    else *(WORD*) (realBase+rxOutOfs) = ofs;
    *(DWORD*) (realBase+pktDrop) = 0UL;
    _enable();
  }

  PUBLIC WORD PktBuffersUsed (void)
  {
    WORD inOfs, outOfs;

    _disable();
    inOfs  = *(WORD*) (realBase+rxInOfs);
    outOfs = *(WORD*) (realBase+rxOutOfs);
    _enable();
    if (inOfs >= outOfs)
       return (inOfs - outOfs) / sizeof(RX_ELEMENT);
    return (NUM_RX_BUF - (outOfs - inOfs) / sizeof(RX_ELEMENT));
  }

  PUBLIC DWORD PktRxDropped (void)
  {
    return *(DWORD*) (realBase+pktDrop);
  }

```

这段代码定义了 three 个函数，以及一个名为 PktBuffersUsed 的宏。它们的作用如下：

1. PktReceive：该函数接收一行数据并返回其长度。它接收的数据是一行字节数组 `buf`，最大长度为 `max` 字节。函数首先检查接收到的数据是否发生偏移，如果是，则将其移回到正确的位置，然后计算并复制该数据。最后，函数返回发生偏移的字节数。

2. PktQueueBusy：该函数用于在数据传输过程中标记缓冲区为忙碌状态。当函数被调用时，它会更新接收端的偏移，并检查缓冲区是否为空。如果是，则会将缓冲区标记为忙碌状态，并将 `pktDrop` 设置为 0。

3. PktBuffersUsed：该函数返回在给定时间内使用的物理缓冲区数量。它通过计算已使用的缓冲区数量和剩余缓冲区数量来得出。函数的实现基于 PktReceive 函数，用于在数据传输过程中计算接收端的偏移并更新缓冲区。

4. PktRxDropped：该函数用于在数据传输过程中丢弃数据。当函数被调用时，它会返回正在传输的数据字节数。

PktReceive, PktQueueBusy 和 PktBuffersUsed 函数用于数据传输过程中的数据接收和标记，而 PktRxDropped 函数用于在数据传输过程中丢弃数据。


```cpp
#else     /* real-mode small/large model */

  PUBLIC int PktReceive (BYTE *buf, int max)
  {
    if (rxOutOfs != rxInOfs)
    {
      RX_ELEMENT far *head = (RX_ELEMENT far*) MK_FP (_DS,rxOutOfs);
      int  size, len = max;

      if (CheckElement(head))
      {
        size = min (head->firstCount, sizeof(RX_ELEMENT));
        len  = min (size, max);
        _fmemcpy (buf, &head->destin, len);
      }
      else
        size = -1;

      rxOutOfs += sizeof (RX_ELEMENT);
      if (rxOutOfs > LAST_RX_BUF)
          rxOutOfs = FIRST_RX_BUF;
      return (size);
    }
    return (0);
  }

  PUBLIC void PktQueueBusy (BOOL busy)
  {
    rxOutOfs = busy ? (rxInOfs + sizeof(RX_ELEMENT)) : rxInOfs;
    if (rxOutOfs > LAST_RX_BUF)
        rxOutOfs = FIRST_RX_BUF;
    pktDrop = 0L;
  }

  PUBLIC WORD PktBuffersUsed (void)
  {
    WORD inOfs  = rxInOfs;
    WORD outOfs = rxOutOfs;

    if (inOfs >= outOfs)
       return ((inOfs - outOfs) / sizeof(RX_ELEMENT));
    return (NUM_RX_BUF - (outOfs - inOfs) / sizeof(RX_ELEMENT));
  }

  PUBLIC DWORD PktRxDropped (void)
  {
    return (pktDrop);
  }
```

这段代码是一个无意义的注释，没有函数体，也不返回任何值。然而，它告诉编译器在编译之前需要包含这个定义。所以，这段代码的意义是告诉编译器在编译之前需要定义这个函数。


```cpp
#endif

/**************************************************************************/

LOCAL __inline void PktFreeMem (void)
{
#if (DOSX & PHARLAP)
  if (realSeg)
  {
    _dx_real_free (realSeg);
    realSeg = 0;
  }
#elif (DOSX & DJGPP)
  if (rm_mem.rm_segment)
  {
    unsigned ofs;  /* clear the DOS-mem to prevent further upcalls */

    for (ofs = 0; ofs < 16 * rm_mem.size / 4; ofs += 4)
       _farpokel (_dos_ds, realBase + ofs, 0);
    _go32_dpmi_free_dos_memory (&rm_mem);
    rm_mem.rm_segment = 0;
  }
```

这段代码的作用是检查 DOSX 和 DOS4GW 是否同时存在，如果存在，则执行以下操作：

1. 如果已经执行了 rm_base_sel，则将其取消，即释放之前分配的内存。
2. 否则，执行以下操作：

a. 检查 PKT-DRVR 是否被绑定，如果不是，则执行 PUTS 输出一个错误消息。

b. 如果 PKT-DRVR 已经被绑定，则执行以下操作：

i. 尝试释放已经绑定的 handle。
ii. 如果 handle 仍然被绑定，则继续执行以下操作：
  a. 尝试打印出一些内部统计信息。
  b. 如果 pcap_pkt_debug 至少是 2，则执行以下操作：
     1. 打印出一些统计信息。
     2. 输出一个错误消息，指出错误的 handle。

注意：这段代码的作用是检查 DOSX 和 DOS4GW 是否同时存在，并执行一系列操作，如果执行失败，则输出一个错误消息。但是，实际情况下，DOSX 和 DOS4GW 不一定同时存在，因此，如果检测到这种情况，代码会直接退出，并不会执行 PUTS 或者打印错误消息。


```cpp
#elif (DOSX & DOS4GW)
  if (rm_base_sel)
  {
    dpmi_real_free (rm_base_sel);
    rm_base_sel = 0;
  }
#endif
}

/**************************************************************************/

PUBLIC BOOL PktExitDriver (void)
{
  if (pktInfo.handle)
  {
    if (!PktSetReceiverMode(PDRX_BROADCAST))
       PUTS ("Error restoring receiver mode.");

    if (!PktReleaseHandle(pktInfo.handle))
       PUTS ("Error releasing PKT-DRVR handle.");

    PktFreeMem();
    pktInfo.handle = 0;
  }

  if (pcap_pkt_debug >= 1)
     printf ("Internal stats: too-small %lu, too-large %lu, bad-sync %lu, "
             "wrong-handle %lu\n",
             intStat.tooSmall, intStat.tooLarge,
             intStat.badSync, intStat.wrongHandle);
  return (TRUE);
}

```

这段代码是一个C语言中的一个函数，它对名为"dump_pkt_stub"的函数进行注释。它判断当前系统是否为DOSX或DJGPP或DOS4GW，如果是，则执行名为"dump_pkt_stub"的函数，并在函数内输出接收到的数据（pktReceiver）的每个分片（stub）的真正存储位置。

dump_pkt_stub函数的具体实现如下：
```cppc
static void dump_pkt_stub (void)
{
 int i;

 fprintf (stderr, "PktReceiver %lu, pkt_stub[PktReceiver] =\n",
          PktReceiver);
 for (i = 0; i < 15; i++)
     fprintf (stderr, "%02X, ", real_stub_array[i+PktReceiver]);
 fputs ("\n", stderr);
}
```
dump_pkt_stub函数首先定义了一个内部变量"i"，并使用fprintf函数输出一个字符串，其中包含接收者（PktReceiver）的值，以及一个格式为"%02X "的输出，其中%02X表示输出两个字节，并使用"_"填充宽度。然后，它用一个for循环遍历数组real_stub_array，输出每个分片（stub）的真正存储位置，并输出一个换行符。

这段代码的作用是输出一个实现在内存中的数据结构（如一个二进制数组），以便在调试和测试过程中进行观察。


```cpp
#if (DOSX & (DJGPP|DOS4GW))
static void dump_pkt_stub (void)
{
  int i;

  fprintf (stderr, "PktReceiver %lu, pkt_stub[PktReceiver] =\n",
           PktReceiver);
  for (i = 0; i < 15; i++)
      fprintf (stderr, "%02X, ", real_stub_array[i+PktReceiver]);
  fputs ("\n", stderr);
}
#endif

/*
 * Front end initialization routine
 */
```

这段代码是一个名为 `PKT_RX_MODE` 的函数，参数为 `mode`，用于初始化接收模式。函数内部先定义了一个名为 `writeInfo` 的布尔变量，表示是否输出 pktInfo 中的调试信息。然后，定义了一个名为 `pktInfo` 的结构体，其中包含一个 `quiet` 字段，表示在输出信息时是否显示。

接下来，代码通过 `pcap_pkt_debug` 获取当前调试级别，并判断是否为大于等于 3 且定义了 `__HIGHC__`。如果是，则输出一些关于调试信息的内容，否则不会输出。最后，代码根据操作系统和处理器架构进行了一些检查，返回一个布尔值表示函数是否成功初始化。


```cpp
PUBLIC BOOL PktInitDriver (PKT_RX_MODE mode)
{
  PKT_RX_MODE rxMode;
  BOOL   writeInfo = (pcap_pkt_debug >= 3);

  pktInfo.quiet = (pcap_pkt_debug < 3);

#if (DOSX & PHARLAP) && defined(__HIGHC__)
  if (_mwenv != 2)
  {
    fprintf (stderr, "Only Pharlap DOS extender supported.\n");
    return (FALSE);
  }
#endif

```

这段代码是一个条件判断语句，它会根据两个条件来决定是否输出错误信息。

第一个条件是：(DOSX & PHARLAP) && defined(__WATCOMC__)。这个条件判断中包含了一个位运算，如果这个位运算的结果为真，那么第一个条件（DOSX & PHARLAP）和第二个条件（__WATCOMC__）为真时，执行第一个内部的if语句。

第二个条件是：if (_Extender != 1) { fprintf(stderr, "Only DOS4GW style extenders supported.\n"); }。这个if语句的作用是判断是否支持4GW扩展，如果支持4GW扩展，那么输出"Only DOS4GW style extenders supported."，否则执行if语句内部的语句。

综合来看，这段代码的作用是判断是否支持4GW扩展，如果支持4GW扩展，就会输出"Only DOS4GW style extenders supported."，否则执行if语句内部的语句。


```cpp
#if (DOSX & PHARLAP) && defined(__WATCOMC__)
  if (_Extender != 1)
  {
    fprintf (stderr, "Only DOS4GW style extenders supported.\n");
    return (FALSE);
  }
#endif

  if (!PktSearchDriver())
  {
    PUTS ("Packet driver not found.");
    PktFreeMem();
    return (FALSE);
  }

  if (!PktGetDriverInfo())
  {
    PUTS ("Error getting pkt-drvr information.");
    PktFreeMem();
    return (FALSE);
  }

```

这段代码的作用是判断输入的文件是否为 DOSX 文件，如果是，则尝试将收到的数据复制到输出文件中。

代码首先检查文件是否为 DOSX 文件(使用条件判断中的 &DOSX & PHARLAP)。如果是 DOSX 文件，则尝试从文件中读取数据并将其复制到输出文件中。具体实现是通过调用函数 RealCopy() 和 protBase，并将读取到的数据存储到 realBase 和 protBase 指向的内存区域中。

接着，代码检查是否成功将数据复制到输出文件中。如果是，则将 first_rx_buf 赋值给 rxOutOfsFp 和 rxInOfsFp，并将输出文件指针 protBase 指向的内存区域设为 first_rx_bufsz。最后，如果数据复制失败，则输出 "Cannot allocate real-mode stub." 并返回 FALSE，程序结束。


```cpp
#if (DOSX & PHARLAP)
  if (RealCopy((ULONG)&rxOutOfs, (ULONG)&pktRxEnd,
               &realBase, &protBase, (USHORT*)&realSeg))
  {
    rxOutOfsFp  = (WORD _far *) (protBase + (WORD) &rxOutOfs);
    rxInOfsFp   = (WORD _far *) (protBase + (WORD) &rxInOfs);
    *rxOutOfsFp = FIRST_RX_BUF;
    *rxInOfsFp  = FIRST_RX_BUF;
  }
  else
  {
    PUTS ("Cannot allocate real-mode stub.");
    return (FALSE);
  }

```

这段代码检查 DOSX 是否为 DJGPP 或 DOS4GW，如果是，则执行以下操作：

1. 如果实模式栈（real_stub_array）大小超过了 2^32-1（FFFF），则输出错误并返回 FALSE。
2. 如果 DOSX 包含 DJGPP，则执行以下操作：
a. 计算出 2^32-1（FFFF）到 2^32+1（JMP_EXEC） 的实际大小为 15（JMP_EXEC_PARTIAL）。
b. 将实际大小除以 16，得到 9683.125。
c. 使用 _go32_dpmi_allocate_dos_memory 函数分配 9683.125 大小的内存。
d. 判断分配是否成功，如果成功，则执行以下操作：
e. 将实模式栈（real_stub_array）的起始地址（rm_mem.rm_offset）偏移到新分配的内存上。
f. 调用 dosmemput 函数将大实模式内存（rm_mem.rm_segment+sizeof(real_stub_array)）复制到起始地址对应的小实模式内存上。
g. 使用 _farpokel 和 _farpoke 函数，将实模式内存上对应的小实模式内存（FIRST_RX_BUF）和（RXOutOfs）以及大实模式内存上对应的小实模式内存（RXInOfs）和（RXOutOfs）对齐。
h. 调用 _dos_ds 函数，发送 SSE 和 SAF 指令，以便从实模式内存开始执行。


```cpp
#elif (DOSX & (DJGPP|DOS4GW))
  if (sizeof(real_stub_array) > 0xFFFF)
  {
    fprintf (stderr, "`real_stub_array[]' too big.\n");
    return (FALSE);
  }
#if (DOSX & DJGPP)
  rm_mem.size = (sizeof(real_stub_array) + 15) / 16;

  if (_go32_dpmi_allocate_dos_memory(&rm_mem) || rm_mem.rm_offset != 0)
  {
    PUTS ("real-mode init failed.");
    return (FALSE);
  }
  realBase = (rm_mem.rm_segment << 4);
  dosmemput (&real_stub_array, sizeof(real_stub_array), realBase);
  _farpokel (_dos_ds, realBase+rxOutOfs, FIRST_RX_BUF);
  _farpokel (_dos_ds, realBase+rxInOfs,  FIRST_RX_BUF);

```

这段代码是一个条件判断，判断语句中的条件为DOSX & DOS4GW是否成立，如果不成立，则执行失败并输出“real-mode init failed.”，否则进行以下操作：

1. 分配内存并初始化，将rm_base_sel保存到rm_base_seg中。
2. 将real_stub_array的地址复制到rm_base_seg所指向的内存地址上。
3. 修改第一个和最后一个输出缓冲区为FIRST_RX_BUF，以便在接收数据时可以输出。
4. 在循环中，检查输入数据是否为ASCII字符0x9C或ASCII字符0xFA，如果不是，则继续循环。
5. 如果当前循环参数pushf的值已经大于16，则输出错误并跳回16。
6. 如果二进制数据中包含偏移量为0xB800的ASCII字符，则输出错误并跳回2。
7. 最后，如果pcap_pkt_debug的值大于2，则输出更多的调试信息并调试。

具体来说，该代码是一个实模式以太网启动代码。在实模式模式下，该代码初始化数据链路，创建接收者，并等待输入数据。当收到数据时，将其解析为ASCII字符，并输出缓冲区中的数据。如果输入数据包含非ASCII字符或者输入缓冲区已满，则会输出错误信息并调试。


```cpp
#elif (DOSX & DOS4GW)
  rm_base_seg = dpmi_real_malloc (sizeof(real_stub_array), &rm_base_sel);
  if (!rm_base_seg)
  {
    PUTS ("real-mode init failed.");
    return (FALSE);
  }
  realBase = (rm_base_seg << 4);
  memcpy ((void*)realBase, &real_stub_array, sizeof(real_stub_array));
  *(WORD*) (realBase+rxOutOfs) = FIRST_RX_BUF;
  *(WORD*) (realBase+rxInOfs)  = FIRST_RX_BUF;

#endif
  {
    int pushf = PktReceiver;

    while (real_stub_array[pushf++] != 0x9C &&    /* pushf */
           real_stub_array[pushf]   != 0xFA)      /* cli   */
    {
      if (++para_skip > 16)
      {
        fprintf (stderr, "Something wrong with `pkt_stub.inc'.\n");
        para_skip = 0;
        dump_pkt_stub();
        return (FALSE);
      }
    }
    if (*(WORD*)(real_stub_array + offsetof(PktRealStub,_dummy)) != 0xB800)
    {
      fprintf (stderr, "`real_stub_array[]' is misaligned.\n");
      return (FALSE);
    }
  }

  if (pcap_pkt_debug > 2)
      dump_pkt_stub();

```

It looks like there is a problem with the adapter's address being set. Are you sure that the adapter is properly connected and that the address is correct? If the problem persists, it might be helpful to check the adapter's documentation or seek assistance from the manufacturer or a technical support specialist.



```cpp
#else
  rxOutOfs = FIRST_RX_BUF;
  rxInOfs  = FIRST_RX_BUF;
#endif

  if (!PktSetAccess())
  {
    PUTS ("Error setting pkt-drvr access.");
    PktFreeMem();
    return (FALSE);
  }

  if (!PktGetAddress(&myAddress))
  {
    PUTS ("Error fetching adapter address.");
    PktFreeMem();
    return (FALSE);
  }

  if (!PktSetReceiverMode(mode))
  {
    PUTS ("Error setting receiver mode.");
    PktFreeMem();
    return (FALSE);
  }

  if (!PktGetReceiverMode(&rxMode))
  {
    PUTS ("Error getting receiver mode.");
    PktFreeMem();
    return (FALSE);
  }

  if (writeInfo)
     printf ("Pkt-driver information:\n"
             "  Version  : %d.%d\n"
             "  Name     : %.15s\n"
             "  Class    : %u (%s)\n"
             "  Type     : %u\n"
             "  Number   : %u\n"
             "  Funcs    : %u\n"
             "  Intr     : %Xh\n"
             "  Handle   : %u\n"
             "  Extended : %s\n"
             "  Hi-perf  : %s\n"
             "  RX mode  : %s\n"
             "  Eth-addr : %02X:%02X:%02X:%02X:%02X:%02X\n",

             pktInfo.majVer, pktInfo.minVer, pktInfo.name,
             pktInfo.class,  PktGetClassName(pktInfo.class),
             pktInfo.type,   pktInfo.number,
             pktInfo.funcs,  pktInfo.intr,   pktInfo.handle,
             pktInfo.funcs == 2 || pktInfo.funcs == 6 ? "Yes" : "No",
             pktInfo.funcs == 5 || pktInfo.funcs == 6 ? "Yes" : "No",
             PktRXmodeStr(rxMode),
             myAddress[0], myAddress[1], myAddress[2],
             myAddress[3], myAddress[4], myAddress[5]);

```

这段代码的作用是检查定义的DEBUG变量是否为真，以及检查DOSX和PHARLAP是否为真。如果是，那么执行下面的代码。

if (DEBUG) && (DOSX & PHARLAP)

; 检查DEBUG变量是否为真

if (writeInfo)

; 如果writeInfo为真，执行下面的代码

DWORD    rAdr;
unsigned sel, ofs;

printf ("\nReceiver at   %04X:%04X\n", RP_SEG(rAdr), RP_OFF(rAdr));
printf ("Realbase    = %04X:%04X\n", RP_SEG(realBase), RP_OFF(realBase));

sel = _FP_SEG (protBase);
ofs = _FP_OFF (protBase);
printf ("Protbase    = %04X:%08X\n", Sel,ofs);
printf ("RealSeg     = %04X\n", realSeg);

sel = _FP_SEG (rxOutOfsFp);
ofs = _FP_OFF (rxOutOfsFp);
printf ("rxOutOfsFp  = %04X:%08X\n", Sel,ofs);

sel = _FP_SEG (rxInOfsFp);
ofs = _FP_OFF (rxInOfsFp);
printf ("rxInOfsFp   = %04X:%08X\n", Sel,ofs);

printf ("Ready: *rxOutOfsFp = %04X *rxInOfsFp = %04X\n",
           *rxOutOfsFp, *rxInOfsFp);

PktQueueBusy (TRUE);
printf ("Busy:  *rxOutOfsFp = %04X *rxInOfsFp = %04X\n",
           *rxOutOfsFp, *rxInOfsFp);


```cpp
#if defined(DEBUG) && (DOSX & PHARLAP)
  if (writeInfo)
  {
    DWORD    rAdr = realBase + (WORD)&PktReceiver;
    unsigned sel, ofs;

    printf ("\nReceiver at   %04X:%04X\n", RP_SEG(rAdr),    RP_OFF(rAdr));
    printf ("Realbase    = %04X:%04X\n",   RP_SEG(realBase),RP_OFF(realBase));

    sel = _FP_SEG (protBase);
    ofs = _FP_OFF (protBase);
    printf ("Protbase    = %04X:%08X\n", sel,ofs);
    printf ("RealSeg     = %04X\n", realSeg);

    sel = _FP_SEG (rxOutOfsFp);
    ofs = _FP_OFF (rxOutOfsFp);
    printf ("rxOutOfsFp  = %04X:%08X\n", sel,ofs);

    sel = _FP_SEG (rxInOfsFp);
    ofs = _FP_OFF (rxInOfsFp);
    printf ("rxInOfsFp   = %04X:%08X\n", sel,ofs);

    printf ("Ready: *rxOutOfsFp = %04X *rxInOfsFp = %04X\n",
            *rxOutOfsFp, *rxInOfsFp);

    PktQueueBusy (TRUE);
    printf ("Busy:  *rxOutOfsFp = %04X *rxInOfsFp = %04X\n",
            *rxOutOfsFp, *rxInOfsFp);
  }
```

这段代码是一个 if 语句，它会判断当前所处的 DOS 版本是否支持 DPMI 函数。如果不支持，则退出函数，否则继续执行。

if (DOSX & DOS4GW) 表示：

* DOSX 和 DOS4GW 指的是当前正在运行的 DOS 版本，具体可以参考 x86 和 current_thdw_fault 输出。
* if 语句会执行 if 块内的代码，如果该代码块中的语句为假，则会执行 if 块外的语句。
* 否则退出函数，即直接退出 if 语句。
* DPMI 函数是针对 Watcom 和 DOS4GW 扩展而设计的，用于读取和设置硬盘设备的状态。
* 这个函数有两个参数，intr 和 pointer，其中 intr 是 0x766，表示硬盘设备的状态，而 pointer 则是函数的返回值，也就是 DPMI 函数的执行结果。
* 这个函数会返回一个名为 "dpmi_get_real_vector" 的函数，它接受一个整数参数 intr，并返回一个名为 "w" 的 register 的值，其中 w.cx 是 0x31 字节，表示对 DPMI 函数的控制，而 w.dx 是自加运算后的结果。


```cpp
#endif

  memset (&pktStat, 0, sizeof(pktStat));  /* clear statistics */
  PktQueueBusy (TRUE);
  return (TRUE);
}


/*
 * DPMI functions only for Watcom + DOS4GW extenders
 */
#if (DOSX & DOS4GW)
LOCAL DWORD dpmi_get_real_vector (int intr)
{
  union REGS r;

  r.x.eax = 0x200;
  r.x.ebx = (DWORD) intr;
  int386 (0x31, &r, &r);
  return ((r.w.cx << 4) + r.w.dx);
}

```

这段代码定义了两个名为`dpmi_real_malloc`和`dpmi_real_free`的函数，用于管理DPMI内存分配和释放。这两个函数的具体实现如下：

```cpp
LOCAL WORD dpmi_real_malloc (int size, WORD *selector)
{
   union REGS r;

   r.x.eax = 0x0100;             /* DPMI allocate DOS memory */
   r.x.ebx = (size + 15) / 16;   /* Number of paragraphs requested */
   int386 (0x31, &r, &r);
   if (r.w.cflag & 1)
   return (0);

   *selector = r.w.dx;
   return (r.w.ax);              /* Return segment address */
}

LOCAL void dpmi_real_free (WORD selector)
{
   union REGS r;

   r.x.eax = 0x101;              /* DPMI free DOS memory */
   r.x.ebx = selector;           /* Selector to free */
   int386 (0x31, &r, &r);
}
```

上述代码中，`dpmi_real_malloc`函数的作用是分配一段DPMI内存，指定要分配的段ID，然后返回该段的起始地址。如果分配失败，函数返回0。

`dpmi_real_free`函数的作用是释放指定段的DPMI内存，传入的段ID由函数实参传递给函数。函数先将当前段ID清零，然后将目标段ID设置为传入的段ID，最后返回分配的内存地址。


```cpp
LOCAL WORD dpmi_real_malloc (int size, WORD *selector)
{
  union REGS r;

  r.x.eax = 0x0100;             /* DPMI allocate DOS memory */
  r.x.ebx = (size + 15) / 16;   /* Number of paragraphs requested */
  int386 (0x31, &r, &r);
  if (r.w.cflag & 1)
     return (0);

  *selector = r.w.dx;
  return (r.w.ax);              /* Return segment address */
}

LOCAL void dpmi_real_free (WORD selector)
{
  union REGS r;

  r.x.eax = 0x101;              /* DPMI free DOS memory */
  r.x.ebx = selector;           /* Selector to free */
  int386 (0x31, &r, &r);
}
```

这段代码是一个用于在保护模式下分配指定代码块的内存分配和复制代码的定义。它使用了 DOSX 宏，如果在支持 DOSX 并且定义了 PHARLAP，则这个函数将被用来分配 conventional（非缓存）内存，并将指定的代码块复制到内存中。

该函数需要实模式指针（Real Mode Pointer）和 conventional 内存指针（Conventional Memory Pointer）作为参数。它首先检查是否支持 DOSX 并且是否定义了 PHARLAP, 如果满足，则函数开始执行。

函数的主要作用是在保护模式下分配一块 conventional 的内存，并将指定的代码块复制到该内存区域。这个函数需要实模式指针和 conventional 内存指针作为参数。它主要用于在程序执行完毕后释放 conventional 内存区域，需要确保已配置好的实模式指针与 conventional 内存指针的对应关系。


```cpp
#endif


#if defined(DOSX) && (DOSX & PHARLAP)
/*
 * Description:
 *     This routine allocates conventional memory for the specified block
 *     of code (which must be within the first 64K of the protected mode
 *     program segment) and copies the code to it.
 *
 *     The caller should free up the conventional memory block when it
 *     is done with the conventional memory.
 *
 *     NOTE THIS ROUTINE REQUIRES 386|DOS-EXTENDER 3.0 OR LATER.
 *
 * Calling arguments:
 *     start_offs      start of real mode code in program segment
 *     end_offs        1 byte past end of real mode code in program segment
 *     real_basep      returned;  real mode ptr to use as a base for the
 *                        real mode code (eg, to get the real mode FAR
 *                        addr of a function foo(), take
 *                        real_basep + (ULONG) foo).
 *                        This pointer is constructed such that
 *                        offsets within the real mode segment are
 *                        the same as the link-time offsets in the
 *                        protected mode program segment
 *     prot_basep      returned;  prot mode ptr to use as a base for getting
 *                        to the conventional memory, also constructed
 *                        so that adding the prot mode offset of a
 *                        function or variable to the base gets you a
 *                        ptr to the function or variable in the
 *                        conventional memory block.
 *     rmem_adrp       returned;  real mode para addr of allocated
 *                        conventional memory block, to be used to free
 *                        up the conventional memory when done.  DO NOT
 *                        USE THIS TO CONSTRUCT A REAL MODE PTR, USE
 *                        REAL_BASEP INSTEAD SO THAT OFFSETS WORK OUT
 *                        CORRECTLY.
 *
 * Returned values:
 *     0      if error
 *     1      if success
 */
```



This function appears to be a part of a larger software suite for managing memory in DOS/FAT files. It appears to be used to allocate a block of memory for a DOS file and then copy the contents of that block to the allocated memory.

The function takes in several parameters, including the endianness of the memory, the base address of the allocated memory, and the memory alignment flag. It returns a boolean value indicating whether the allocation was successful.

The function first calculates the length of the allocated memory by subtracting the start address from the end address and adding 15 to the result. It then checks whether the memory endianness is correct and whether the allocation was successful. If the allocation was successful, the function copies the contents of the allocated memory to the base address and returns a TRUE value.


```cpp
int RealCopy (ULONG    start_offs,
              ULONG    end_offs,
              REALPTR *real_basep,
              FARPTR  *prot_basep,
              USHORT  *rmem_adrp)
{
  ULONG   rm_base;    /* base real mode para addr for accessing */
                      /* allocated conventional memory          */
  UCHAR  *source;     /* source pointer for copy                */
  FARPTR  destin;     /* destination pointer for copy           */
  ULONG   len;        /* number of bytes to copy                */
  ULONG   temp;
  USHORT  stemp;

  /* First check for valid inputs
   */
  if (start_offs >= end_offs || end_offs > 0x10000)
     return (FALSE);

  /* Round start_offs down to a paragraph (16-byte) boundary so we can set up
   * the real mode pointer easily. Round up end_offs to make sure we allocate
   * enough paragraphs
   */
  start_offs &= ~15;
  end_offs = (15 + (end_offs << 4)) >> 4;

  /* Allocate the conventional memory for our real mode code.  Remember to
   * round byte count UP to 16-byte paragraph size.  We alloc it
   * above the DOS data buffer so both the DOS data buffer and the appl
   * conventional mem block can still be resized.
   *
   * First just try to alloc it;  if we can't get it, shrink the appl mem
   * block down to the minimum, try to alloc the memory again, then grow the
   * appl mem block back to the maximum.  (Don't try to shrink the DOS data
   * buffer to free conventional memory;  it wouldn't be good for this routine
   * to have the possible side effect of making file I/O run slower.)
   */
  len = ((end_offs - start_offs) + 15) >> 4;
  if (_dx_real_above(len, rmem_adrp, &stemp) != _DOSE_NONE)
  {
    if (_dx_cmem_usage(0, 0, &temp, &temp) != _DOSE_NONE)
       return (FALSE);

    if (_dx_real_above(len, rmem_adrp, &stemp) != _DOSE_NONE)
       *rmem_adrp = 0;

    if (_dx_cmem_usage(0, 1, &temp, &temp) != _DOSE_NONE)
    {
      if (*rmem_adrp != 0)
         _dx_real_free (*rmem_adrp);
      return (FALSE);
    }

    if (*rmem_adrp == 0)
       return (FALSE);
  }

  /* Construct real mode & protected mode pointers to access the allocated
   * memory.  Note we know start_offs is aligned on a paragraph (16-byte)
   * boundary, because we rounded it down.
   *
   * We make the offsets come out rights by backing off the real mode selector
   * by start_offs.
   */
  rm_base = ((ULONG) *rmem_adrp) - (start_offs >> 4);
  RP_SET (*real_basep, 0, rm_base);
  FP_SET (*prot_basep, rm_base << 4, SS_DOSMEM);

  /* Copy the real mode code/data to the allocated memory
   */
  source = (UCHAR *) start_offs;
  destin = *prot_basep;
  FP_SET (destin, FP_OFF(*prot_basep) + start_offs, FP_SEL(*prot_basep));
  len = end_offs - start_offs;
  WriteFarMem (destin, source, len);

  return (TRUE);
}
```

这段代码是一个 preprocessed 预处理指令，它会在源代码文件被编译之前对代码进行处理。在这段注释中，说明该代码是在 DOSX 和 PHARLAP 环境下一个编译器所必需的。

换句话说，这段代码会确保在编译之前，你已经明确指定了要使用的编译器，并且在编译过程中需要使用这种编译器。这个功能在某些编程语言中（如 C 和 C++）非常重要，以确保代码在不同的操作系统和计算机上都能正常运行。


```cpp
#endif /* DOSX && (DOSX & PHARLAP) */

```