# Nmap源码解析 82

# `libpcre/src/pcre2_intmodedep.h`

这段代码是一个Perl兼容的正则表达式库，它提供了一些与Perl 5语言的语法和语义最为接近的功能。

该代码是一个PCRE库，它包含许多用于支持正则表达式的函数。通过使用PCRE库，用户可以编写正则表达式，并在Perl或Python等编程语言中使用它们。


```cpp
/*************************************************
*      Perl-Compatible Regular Expressions       *
*************************************************/

/* PCRE is a library of functions to support regular expressions whose syntax
and semantics are as close as possible to those of the Perl 5 language.

                       Written by Philip Hazel
     Original API code Copyright (c) 1997-2012 University of Cambridge
          New API code Copyright (c) 2016-2022 University of Cambridge

-----------------------------------------------------------------------------
Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

    * Redistributions of source code must retain the above copyright notice,
      this list of conditions and the following disclaimer.

    * Redistributions in binary form must reproduce the above copyright
      notice, this list of conditions and the following disclaimer in the
      documentation and/or other materials provided with the distribution.

    * Neither the name of the University of Cambridge nor the names of its
      contributors may be used to endorse or promote products derived from
      this software without specific prior written permission.

```

这段代码是一个用于输出文本的 C 语言源代码，其中包括一些表示版权和许可证信息的文本。其中包括以下几行：

```cpp
THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
ARE DISCLAIMED.                                                        
IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE LIABLE FOR ANY
DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING,
BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF
USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
POSSIBILITY OF SUCH DAMAGE.                                              
```

这些行表示该软件的版权和许可证信息，其中一些是隐含的保证，不包括在显著的保证之中。此外，该软件按“仅供参考”的方式提供，即该软件并不是作为某个产品的保证提供的。在任何情况下，该软件的版权或许可证持有人不会承担任何直接、间接、特殊、实体或后果的损害责任，即使如此，也不能保证该软件适用于任何特定的目的。


```cpp
THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE
LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
POSSIBILITY OF SUCH DAMAGE.
-----------------------------------------------------------------------------
*/


```

这段代码定义了一些模式相关的宏和结构定义，用于 PCRE2_CODE_UNIT_WIDTH 是否被定义。如果定义了 PCRE2_CODE_UNIT_WIDTH，那么这些模式相关的宏和结构定义将被保存在一个单独的文件中，以便在不同的 PCRE2_CODE_UNIT 宽度下多次包括。

有些模式相关的宏对于不同 PCRE2_CODE_UNIT 宽度是必需的（尤其是在 pcre_printint.c 文件中）。在这里，我们定义了这些宏，以便在多个 PCRE2_CODE_UNIT 宽度下进行重新定义。

有些模式相关的宏在不同的 PCRE2_CODE_UNIT 宽度下并不需要。在这里，我们统一地否定了这些宏，以便在多个 PCRE2_CODE_UNIT 宽度下进行重新定义。

总之，这段代码定义了一些模式相关的宏和结构定义，用于 PCRE2_CODE_UNIT 宽度，以便在不同的 PCRE2_CODE_UNIT 宽度下进行重新定义。


```cpp
/* This module contains mode-dependent macro and structure definitions. The
file is #included by pcre2_internal.h if PCRE2_CODE_UNIT_WIDTH is defined.
These mode-dependent items are kept in a separate file so that they can also be
#included multiple times for different code unit widths by pcre2test in order
to have access to the hidden structures at all supported widths.

Some of the mode-dependent macros are required at different widths for
different parts of the pcre2test code (in particular, the included
pcre_printint.c file). We undefine them here so that they can be re-defined for
multiple inclusions. Not all of these are used in pcre2test, but it's easier
just to undefine them all. */

#undef ACROSSCHAR
#undef BACKCHAR
#undef BYTES2CU
```

这段代码定义了一系列无符号整型变量，用于表示图像和视频处理中的数据类型。其中包括：

- CHMAX_255：表示最大颜色深度，即255个像素。
- CU2BYTES：表示无符号字节，即字节序列中的数据类型。
- FORWARDCHAR：表示无符号字符，即字符序列中的数据类型。
- FORWARDCHARTEST：表示无符号字符串，即字符串序列中的数据类型。
- GET：表示获取，获取操作的具体实现未知。
- GET2：表示获取第二个参数，获取操作的具体实现未知。
- GETCHAR：表示从文件或设备中读取字符，并返回其ASCII编码值。
- GETCHARINC：表示从文件或设备中读取字符，并返回其ASCII编码值，但不适用于具有两个或更多个缓冲区的文件。
- GETCHARINCTEST：表示从文件或设备中读取字符，并返回其ASCII编码值，但不适用于具有两个或更多个缓冲区的文件。
- GETCHARLEN：表示从文件或设备中读取字符，并返回其ASCII编码值，但不适用于具有两个或更多个缓冲区的文件。
- GETCHARTEST：表示获取图像中的颜色信息，并返回其ASCII编码值。
- GET_EXTRALEN：表示获取图像中的颜色信息，并返回其ASCII编码值，但不适用于具有两个或更多个缓冲区的文件。
- HAS_EXTRALEN：表示判断图像是否具有两个或更多个缓冲区，用于在支持具有两个或更多个缓冲区的图像时进行判断。
- IMM2_SIZE：表示图像和视频处理中的数据类型大小，用于表示图像和视频处理中的数据类型。


```cpp
#undef CHMAX_255
#undef CU2BYTES
#undef FORWARDCHAR
#undef FORWARDCHARTEST
#undef GET
#undef GET2
#undef GETCHAR
#undef GETCHARINC
#undef GETCHARINCTEST
#undef GETCHARLEN
#undef GETCHARLENTEST
#undef GETCHARTEST
#undef GET_EXTRALEN
#undef HAS_EXTRALEN
#undef IMM2_SIZE
```

这段代码定义了多个无符号整数变量MAX_255、MAX_MARK、MAX_PATTERN_SIZE、MAX_UTF_SINGLE_CU、NOT_FIRSTCU，以及一个名为PUT的宏，一个名为PUT2的宏和一个名为PUTINC的宏。还定义了一个名为TABLE_GET的函数。

具体来说，MAX_255是一个256位无符号整数变量，MAX_MARK是一个256位无符号整数变量，MAX_PATTERN_SIZE是一个指针变量，用于保存一个匹配模式的长度，MAX_UTF_SINGLE_CU是一个无符号整数变量，用于保存Unicode字符集中的单个字符，NOT_FIRSTCU是一个布尔变量，用于指示是否是第一个字符，PUT是一个名为put的函数，用于将一个字符串和一个模式字符串连接起来，PUT2是一个名为put2的函数，用于将一个字符串和一个整数连接起来，PUTINC是一个名为putinc的函数，用于将一个整数和一个字符串连接起来。最后，TABLE_GET是一个函数，用于从指定的表中获取数据。


```cpp
#undef MAX_255
#undef MAX_MARK
#undef MAX_PATTERN_SIZE
#undef MAX_UTF_SINGLE_CU
#undef NOT_FIRSTCU
#undef PUT
#undef PUT2
#undef PUT2INC
#undef PUTCHAR
#undef PUTINC
#undef TABLE_GET



/* -------------------------- MACROS ----------------------------- */

```

这段代码定义了一个PCRE类，该类将16位溢出为8位，这意味着PCRE将每个编译代码单元的偏移量存储为至少16位，并在大端模式下以8位为单位存储。这些偏移量用于从子模式的开始到替代物和结束的链接。

具体来说，偏移量用于将子模式与它的替代物链接起来，从而实现更复杂的文本搜索和字符串匹配。如果请求更大的限制，则可以使用16位溢出限制PCRE编译后文件的大小，但是这种情况下需要手动定义和修改相应的PCRE代码。

另外，由于存储和加载偏移量文本模式的大小是一个相对较大且容易出错的概念，因此PCRE定义了一系列控制宏，用于处理编译代码单元的偏移量，从而简化代码并提高可维护性。


```cpp
/* PCRE keeps offsets in its compiled code as at least 16-bit quantities
(always stored in big-endian order in 8-bit mode) by default. These are used,
for example, to link from the start of a subpattern to its alternatives and its
end. The use of 16 bits per offset limits the size of an 8-bit compiled regex
to around 64K, which is big enough for almost everybody. However, I received a
request for an even bigger limit. For this reason, and also to make the code
easier to maintain, the storing and loading of offsets from the compiled code
unit string is now handled by the macros that are defined here.

The macros are controlled by the value of LINK_SIZE. This defaults to 2, but
values of 3 or 4 are also supported. */

/* ------------------- 8-bit support  ------------------ */

#if PCRE2_CODE_UNIT_WIDTH == 8

```



这段代码定义了一系列宏，用于在不同的大小(2或3)下对输入的多态字符串进行处理。具体来说：

- PUT(a,n,d)定义了一个函数，接收三个参数：一个字符串a、一个整数n和一个小整数d。该函数的作用是将字符串a中第n个位置的(小整数d)右移8位，然后将右移后的值存储到a的第n个位置，并将左移后的值存储到a的第(n+1)个位置。
- GET(a,n)定义了一个函数，接收两个参数：一个字符串a和一个整数n。该函数的作用是将字符串a中第n个位置的(小整数d)右移16位，然后将右移后的值存储到a的第n个位置，并将左移后的值存储到a的第(n+1)个位置。
- MAX_PATTERN_SIZE是一个常量，表示可以定义的最大字符数组大小为16进制下的16。


```cpp
#if LINK_SIZE == 2
#define PUT(a,n,d)   \
  (a[n] = (PCRE2_UCHAR)((d) >> 8)), \
  (a[(n)+1] = (PCRE2_UCHAR)((d) & 255))
#define GET(a,n) \
  (unsigned int)(((a)[n] << 8) | (a)[(n)+1])
#define MAX_PATTERN_SIZE (1 << 16)

#elif LINK_SIZE == 3
#define PUT(a,n,d)       \
  (a[n] = (PCRE2_UCHAR)((d) >> 16)),    \
  (a[(n)+1] = (PCRE2_UCHAR)((d) >> 8)), \
  (a[(n)+2] = (PCRE2_UCHAR)((d) & 255))
#define GET(a,n) \
  (unsigned int)(((a)[n] << 16) | ((a)[(n)+1] << 8) | (a)[(n)+2])
```

这段代码定义了一系列宏，用于在 C 语言中处理模式匹配的相关操作。

MAX_PATTERN_SIZE 是一个定义，表示一个字符串模式的最大长度，使用左前缀计数法。

LINK_SIZE 是另一个定义，用于指定模式匹配结果中链接符的数量。当 LINK_SIZE 是 2 时，匹配两个字符串，将匹配结果中的两个字符串连接成一个字符串。当 LINK_SIZE 是 3 时，匹配三个字符串，将匹配结果中的三个字符串连接成一个字符串。当 LINK_SIZE 是 4 时，匹配四个字符串，将匹配结果中的四个字符串连接成一个字符串。当 LINK_SIZE 不等于 2、3 或 4 时，该函数将无法正常工作。

PUT 函数定义了一个字符数组，其中的每个元素都是一个 PCRE2_UCHAR 类型的整数。这个函数的作用是将给定的匹配模式中的每个元素转换成对应的 PCRE2_UCHAR 类型，并在字符数组中进行相应的替换。

GET 函数定义了一个字符数组，其中的每个元素都是一个 PCRE2_UCHAR 类型的整数。这个函数的作用是将给定的模式匹配结果中的每个元素转换成对应的 PCRE2_UCHAR 类型，并将字符数组中的每个元素与对应的 PCRE2_UCHAR 类型进行按位或操作，得到一个新的字符数组。


```cpp
#define MAX_PATTERN_SIZE (1 << 24)

#elif LINK_SIZE == 4
#define PUT(a,n,d)        \
  (a[n] = (PCRE2_UCHAR)((d) >> 24)),     \
  (a[(n)+1] = (PCRE2_UCHAR)((d) >> 16)), \
  (a[(n)+2] = (PCRE2_UCHAR)((d) >> 8)),  \
  (a[(n)+3] = (PCRE2_UCHAR)((d) & 255))
#define GET(a,n) \
  (unsigned int)(((a)[n] << 24) | ((a)[(n)+1] << 16) | ((a)[(n)+2] << 8) | (a)[(n)+3])
#define MAX_PATTERN_SIZE (1 << 30)   /* Keep it positive */

#else
#error LINK_SIZE must be 2, 3, or 4
#endif


```

这段代码定义了一个PCRE2_CODE_UNIT_WIDTH为16的定义，并且在相应的条件下起作用。下面是具体的实现：

1. 如果PCRE2_CODE_UNIT_WIDTH为16，那么整个代码将不起作用，因为该定义没有被实现。

2. 如果PCRE2_CODE_UNIT_WIDTH为16，并且LINK_SIZE为2，那么代码将使用以下公式计算一个PCRE2_UCHAR类型的参数d:
```cpp
a[n] = (PCRE2_UCHAR)(d)
```
3. 如果PCRE2_CODE_UNIT_WIDTH为16，并且LINK_SIZE为3或4，那么代码将不会起作用，因为该定义没有被实现。


```cpp
/* ------------------- 16-bit support  ------------------ */

#elif PCRE2_CODE_UNIT_WIDTH == 16

#if LINK_SIZE == 2
#undef LINK_SIZE
#define LINK_SIZE 1
#define PUT(a,n,d)   \
  (a[n] = (PCRE2_UCHAR)(d))
#define GET(a,n) \
  (a[n])
#define MAX_PATTERN_SIZE (1 << 16)

#elif LINK_SIZE == 3 || LINK_SIZE == 4
#undef LINK_SIZE
```

这段代码定义了一系列宏，用于定义有限长度字符串中的模式，以及用于查找和修改字符串中特定位置的字符的函数。以下是每个宏的说明：

1. `LINK_SIZE`：定义一个名为`LINK_SIZE`的宏，其值为2。
2. `PUT`：定义一个名为`PUT`的函数，其接收三个参数：一个字符串`a`、一个整数`n`和一个整数`d`。该函数将`a`中第`n`个位置的左移16位后的内容修改为`PCRE2_UCHAR`类型的数值，并将`d`的低16位和该数值的左移16位后的内容修改为`PCRE2_UCHAR`类型的数值。
3. `GET`：定义一个名为`GET`的函数，其接收两个参数：一个字符串`a`和一个整数`n`。该函数将`a`中第`n`个位置的左移16位后的内容转换为`unsigned int`类型的整数，并将`a`中第`(n)+1`个位置的左移16位后的内容转换为`unsigned int`类型的整数。
4. `MAX_PATTERN_SIZE`：定义一个名为`MAX_PATTERN_SIZE`的常量，其值为`(1 << 30)`。该常量用于确保`PUT`和`GET`函数不会对超过32个字符的输入字符串进行操作。


```cpp
#define LINK_SIZE 2
#define PUT(a,n,d)   \
  (a[n] = (PCRE2_UCHAR)((d) >> 16)), \
  (a[(n)+1] = (PCRE2_UCHAR)((d) & 65535))
#define GET(a,n) \
  (unsigned int)(((a)[n] << 16) | (a)[(n)+1])
#define MAX_PATTERN_SIZE (1 << 30)  /* Keep it positive */

#else
#error LINK_SIZE must be 2, 3, or 4
#endif


/* ------------------- 32-bit support  ------------------ */

```

这段代码是一个C语言的if语句，根据PCRE2_CODE_UNIT_WIDTH是否为32来执行不同的代码。

当PCRE2_CODE_UNIT_WIDTH等于32时，代码将跳过以下定义：

```cpp
#undef LINK_SIZE
#define LINK_SIZE 1
```

并且将定义的LINK_SIZE的值设为1。

```cpp
#define PUT(a,n,d)   \
 (a[n] = (d))
#define GET(a,n) \
 (a[n])
```

还将MAX_PATTERN_SIZE的值设为(1 << 30)，用于匹配输入中的最大长度。

```cpp
#else
#error Unsupported compiling mode
#endif
```

当PCRE2_CODE_UNIT_WIDTH不等于32时，代码将执行以下定义：

```cpp
#include "pcre2-codeunit.h"
```

然后根据PCRE2_CODE_UNIT_WIDTH的值执行相应的宏定义。


```cpp
#elif PCRE2_CODE_UNIT_WIDTH == 32
#undef LINK_SIZE
#define LINK_SIZE 1
#define PUT(a,n,d)   \
  (a[n] = (d))
#define GET(a,n) \
  (a[n])
#define MAX_PATTERN_SIZE (1 << 30)  /* Keep it positive */

#else
#error Unsupported compiling mode
#endif


/* --------------- Other mode-specific macros ----------------- */

```

这段代码定义了PCRE在使用计数值时需要的一些16位数量，以及用于存储和获取这些数量的宏。

首先定义了需要多少个代码单元来存储一个16位计数值，以及用于存储和获取计数值的宏。

然后定义了一个8位的GET2函数，用于将一个8位计数值转换成一个16位计数值。这个函数将左移8位并右移8位，得到一个24位的计数值。

接着定义了一个9位的PUT2函数，用于将一个16位计数值转换成一个8位计数值。这个函数首先将右移8位并将值存储到第一个参数中，然后将值左移8位。

最后定义了一个PCRE2_CODE_UNIT_WIDTH宏，用于在代码中设置16位计数值的宽度为8位。


```cpp
/* PCRE uses some other (at least) 16-bit quantities that do not change when
the size of offsets changes. There are used for repeat counts and for other
things such as capturing parenthesis numbers in back references.

Define the number of code units required to hold a 16-bit count/offset, and
macros to load and store such a value. For reasons that I do not understand,
the expression in the 8-bit GET2 macro is treated by gcc as a signed
expression, even when a is declared as unsigned. It seems that any kind of
arithmetic results in a signed value. Hence the cast. */

#if PCRE2_CODE_UNIT_WIDTH == 8
#define IMM2_SIZE 2
#define GET2(a,n) (unsigned int)(((a)[n] << 8) | (a)[(n)+1])
#define PUT2(a,n,d) a[n] = (d) >> 8, a[(n)+1] = (d) & 255

```

这段代码定义了一些宏，用于在8位模式下去定义宏变量IMM2_SIZE，并在需要时将8位模式下的宏变量应用到IMM2_SIZE上。

IMM2_SIZE是一个宏定义，其值为1。

GET2和PUT2是宏定义，用于在需要时将8位模式下的宏变量应用到IMM2_SIZE上。GET2将8位模式下的宏变量a[n]应用到IMM2_SIZE上，并将其名为d的宏定义，而PUT2则是相反的，它将IMM2_SIZE上的宏变量d应用到a[n]上，并将其名为d的宏定义。

MAX_255是一个宏定义，用于定义MARK名字符串的最大长度。它用于在8位模式下去检查MARK名字符串是否小于256个字符，如果是，则执行MAX_255宏，否则执行CHMAX_255宏。

TABLE_GET是一个宏定义，用于在8位模式下去访问包含恰好256个元素的表。它的实现在代码中未定义，因此无法确定它的具体行为。


```cpp
#else  /* Code units are 16 or 32 bits */
#define IMM2_SIZE 1
#define GET2(a,n) a[n]
#define PUT2(a,n,d) a[n] = d
#endif

/* Other macros that are different for 8-bit mode. The MAX_255 macro checks
whether its argument, which is assumed to be one code unit, is less than 256.
The CHMAX_255 macro does not assume one code unit. The maximum length of a MARK
name must fit in one code unit; currently it is set to 255 or 65535. The
TABLE_GET macro is used to access elements of tables containing exactly 256
items. Its argument is a code unit. When code points can be greater than 255, a
check is needed before accessing these tables. */

#if PCRE2_CODE_UNIT_WIDTH == 8
```



这段代码定义了一系列头文件和常量，用于定义C语言中的宏和标识符。以下是每个部分的解释：

```cpp
#define MAX_255(c) TRUE          /* macro to check if an integer is 255 or not */
```

这个宏定义了一个名为`MAX_255`的函数，它的参数是一个整数`c`。如果`c`是255或更大的整数，函数返回`TRUE`，否则返回`FALSE`。

```cpp
#define MAX_MARK((1u << 8) - 1)    /* macro to define MAX_MARK */
```

这个宏定义了一个名为`MAX_MARK`的函数，它的参数是一个整数`c`，它表示一个16位无符号整数的最大值。具体来说，它通过计算`1u << 8`来获取一个16位的最大值，然后将其减去1以获取一个8位的最大值。

```cpp
#define TABLE_GET(c, table, default) ((table)[c])    /* macro to define TABLE_GET */
```

这个宏定义了一个名为`TABLE_GET`的函数，它的参数包括两个部分：一个整数`c`，一个整数`table`，和一个默认值`default`。它从`table`中取出一个值，如果`c`与`table`中的某个值相等，则返回相应的`table`中的值，否则返回`default`。

```cpp
#ifdef SUPPORT_UNICODE                       /* macro to define MAX_255 with support for UNICODE */
```

这个条件语句用于检查是否支持C语言中的Unicode字符集。如果支持，它定义了一个名为`SUPPORT_WIDE_CHARS`的函数，它返回`TRUE`，否则返回`FALSE`。

```cpp
#define CHMAX_255(c) ((c) <= 255u)             /* macro to define CHMAX_255 */
```

这个宏定义了一个名为`CHMAX_255`的函数，它的参数是一个整数`c`。它通过检查`c`是否小于或等于255来定义一个布尔值，如果是，则返回`true`，否则返回`false`。

```cpp
#define MAX_255(c) ((c) <= 255u)             /* macro to define MAX_255 */
```

这个宏定义了一个名为`MAX_255`的函数，它的参数是一个整数`c`。它通过检查`c`是否小于或等于255来定义一个布尔值，如果是，则返回`true`，否则返回`false`。

```cpp
#define MAX_MARK((1u << 16) - 1)    /* macro to define MAX_MARK */
```

这个宏定义了一个名为`MAX_MARK`的函数，它的参数是一个整数`c`，它表示一个16位无符号整数的最大值。具体来说，它通过计算`1u << 8`来获取一个16位的最大值，然后将其减去1以获取一个8位的最大值。

```cpp
#elif defined( __GNUC__) || defined( __STDC__)         /* code block for UNIX-style C code */
```

这是一个代码块，用于在UNIX风格C代码中定义宏。如果`__GNUC__`或`__STDC__`定义了这个函数，那么它将使用GNUC语法定义宏。否则，它将使用C语言语法定义宏。


```cpp
#define MAX_255(c) TRUE
#define MAX_MARK ((1u << 8) - 1)
#define TABLE_GET(c, table, default) ((table)[c])
#ifdef SUPPORT_UNICODE
#define SUPPORT_WIDE_CHARS
#define CHMAX_255(c) ((c) <= 255u)
#else
#define CHMAX_255(c) TRUE
#endif  /* SUPPORT_UNICODE */

#else  /* Code units are 16 or 32 bits */
#define CHMAX_255(c) ((c) <= 255u)
#define MAX_255(c) ((c) <= 255u)
#define MAX_MARK ((1u << 16) - 1)
#define SUPPORT_WIDE_CHARS
```

这段代码定义了一个名为“TABLE_GET”的宏，用于从table变量中获取c列的值，如果c列的值小于255，则返回table[c]的值，否则返回default。这个宏是在#define TABLE_GET(c, table, default) ... endif这一行注释中定义的。

该代码中定义了一系列以“UCHAR21”为前缀的宏，用于在程序中处理字符数据类型。其中包括MAX_255(c)的宏，它会在编译时判断c列的值是否小于255，如果是，则返回((table)[c])的值，否则返回default。这些宏都有简单的定义，用于在不同的模式下去操作。

当新的UTF-21模式被实现时，该代码中的这些宏将根据#ifdef来编译库。当支持该模式时，编译器将根据#ifdef来编译该库，从而可以使用这些宏。


```cpp
#define TABLE_GET(c, table, default) (MAX_255(c)? ((table)[c]):(default))
#endif


/* ----------------- Character-handling macros ----------------- */

/* There is a proposed future special "UTF-21" mode, in which only the lowest
21 bits of a 32-bit character are interpreted as UTF, with the remaining 11
high-order bits available to the application for other uses. In preparation for
the future implementation of this mode, there are macros that load a data item
and, if in this special mode, mask it to 21 bits. These macros all have names
starting with UCHAR21. In all other modes, including the normal 32-bit
library, the macros all have the same simple definitions. When the new mode is
implemented, it is expected that these definitions will be varied appropriately
using #ifdef when compiling the library that supports the special mode. */

```

这段代码定义了一系列宏，用于在支持UTF编码的情况下对字符进行操作。

首先，定义了一个名为"UCHAR21"的宏，它的作用是获取一个字节（UCHAR）类型的指针（eptr）的下一个字节。

接下来定义了一个名为"UCHAR21TEST"的宏，它的作用与上面宏的作用相同，但是它是非空的，即它返回的指针（eptr）至少包含一个字节（UCHAR）。

然后定义了一个名为"UCHAR21INC"的宏，它的作用是获取一个字节（UCHAR）类型的指针（eptr）的值加1。

最后定义了一个名为"UCHAR21INCTEST"的宏，它的作用与上面宏的作用相同，但是它是非空的，即它返回的指针（eptr）至少包含一个字节（UCHAR）。

通过这些宏，可以方便地操作字符，例如获取字符串中的下一个字符、获取字符串的长度等等。当不支持UTF编码时，这些宏将不再被使用。


```cpp
#define UCHAR21(eptr)        (*(eptr))
#define UCHAR21TEST(eptr)    (*(eptr))
#define UCHAR21INC(eptr)     (*(eptr)++)
#define UCHAR21INCTEST(eptr) (*(eptr)++)

/* When UTF encoding is being used, a character is no longer just a single
byte in 8-bit mode or a single short in 16-bit mode. The macros for character
handling generate simple sequences when used in the basic mode, and more
complicated ones for UTF characters. GETCHARLENTEST and other macros are not
used when UTF is not supported. To make sure they can never even appear when
UTF support is omitted, we don't even define them. */

#ifndef SUPPORT_UNICODE

/* #define MAX_UTF_SINGLE_CU */
```

这段代码定义了一系列宏定义，用于在编译时检查输入的字符串是否符合特定的格式。

具体来说，宏定义的内容如下：

* `HAS_EXTRALEN(c)`：表示该符号是否定义了`EXTRALEN`这个名字。如果没有定义，则此宏不会产生任何作用，可以将其忽略。
* `GET_EXTRALEN(c)`：表示如何获取`EXTRALEN`这个名字。如果已经定义过了，那么此宏返回`EXTRALEN`的值；否则，返回`NULL`。
* `NOT_FIRSTCU(c)`：表示该符号是否定义了`NOT_FIRSTCU`这个名字。如果没有定义，则此宏不会产生任何作用，可以将其忽略。
* `GETCHAR(c, eptr)`：表示如何从给定的输入字符串中获取第一个字符的位置。它的参数包括：输入字符串`c`和指向字符及其下一个字符的指针`eptr`。它返回的是字符`c`的下一个字符的值，或者如果`eptr`指向的字符不是字符串的第一个字符，则返回该字符的值。
* `GETCHARTEST(c, eptr)`：表示如何从给定的输入字符串中获取第一个字符的位置，但允许在字符串中查找字符。它的参数与`GETCHAR`类似，但允许在字符串中查找第一个字符。
* `GETCHARINC(c, eptr)`：表示如何从给定的输入字符串中获取第一个字符的位置，并递增该位置的字符。它的参数与`GETCHAR`类似，但允许在字符串中查找第一个字符，并递增该位置的字符。
* `GETCHARINCTEST(c, eptr)`：表示如何从给定的输入字符串中获取第一个字符的位置，并允许在字符串中查找第一个字符，但不允许递增该位置的字符。它的参数与`GETCHARINCTEST`类似，但允许在字符串中查找第一个字符，但不允许递增该位置的字符。
* `GETCHARLEN(c, eptr, len)`：表示如何获取字符串中指定位置的字符和该位置的字符计数。它的参数包括：输入字符串`c`、指向字符及其下一个字符的指针`eptr`和字符计数值`len`。它返回的是字符`c`的下一个字符的值，以及该位置的字符计数。
* `PUTCHAR(c, p)`：表示如何将给定的字符串中的字符及其下标存储到指定的指针中。它的参数包括：输入字符串`c`、指定指针`p`和字符计数值`1`，用于指示是否将字符及其下标存储到指针中。它将字符`c`的值存储到指定指针中，并将字符及其下标的计数递增。
* `GETCHARLENTEST(c, eptr, len)`：表示如何获取字符串中指定位置的字符和该位置的字符计数，允许在字符串中查找第一个字符。它的参数包括：输入字符串`c`、指向字符及其下一个字符的指针`eptr`和字符计数值`len`。它返回的是字符`c`的下一个字符的值，以及该位置的字符计数。
* `BACKCHAR(eptr)`：表示如何返回指定位置的字符串的最后一个字符。它的参数包括：指向字符及其下一个字符的指针`eptr`。它返回的是字符串的最后一个字符。
* `FORWARDCHAR(eptr)`：表示如何返回指定位置的字符串的第一个字符。它的参数包括：指向字符及其下一个字符的指针`eptr`。它返回的是字符串的第一个字符。
* `FORWARCCHARTEST(eptr,end)`：表示如何从给定的输入字符串中获取指定位置的字符，允许在字符串中查找第一个字符。它的参数包括：输入字符串`c`、指向字符及其下一个字符的指针`eptr`和指针`end`，用于指示字符串的结束位置。它返回的是字符`c`的第一个字符，或者如果`end`指向的字符不是字符串的第一个字符，则返回该字符的值。
* `ACROSSCHAR(condition, eptr, action)`：表示如何从给定的输入字符串中获取指定位置的字符，允许在字符串中查找多个字符。它的参数包括：条件`condition`、指向字符及其下一个字符的指针`eptr`和指针`action`，用于指示是否允许在字符串中查找多个字符。它返回的是指定位置的字符及其下标的计数，或者如果`action`


```cpp
/* #define HAS_EXTRALEN(c) */
/* #define GET_EXTRALEN(c) */
/* #define NOT_FIRSTCU(c) */
#define GETCHAR(c, eptr) c = *eptr;
#define GETCHARTEST(c, eptr) c = *eptr;
#define GETCHARINC(c, eptr) c = *eptr++;
#define GETCHARINCTEST(c, eptr) c = *eptr++;
#define GETCHARLEN(c, eptr, len) c = *eptr;
#define PUTCHAR(c, p) (*p = c, 1)
/* #define GETCHARLENTEST(c, eptr, len) */
/* #define BACKCHAR(eptr) */
/* #define FORWARDCHAR(eptr) */
/* #define FORWARCCHARTEST(eptr,end) */
/* #define ACROSSCHAR(condition, eptr, action) */

```

这段代码定义了一个条件判断块，名为`else`。在这个条件判断块内部，定义了一系列与UTF-8编码有关的常量和定义。

首先，代码中定义了一个名为`MAYBE_UTF_MULTI`的宏，它表示UTF-8字符是否可以使用多个字节编码。接下来，定义了一个名为`MAX_UTF_SINGLE_CU`的宏，表示UTF-8编码中单个字节的最大值。

然后，定义了一个名为`HAS_EXTRALEN`的宏，它表示一个字符是否使用了扩展字符。最后，定义了一个名为`PCRE2_CODE_UNIT_WIDTH`的宏，它表示PCRE2编码单元的最大宽度。

该代码的作用是定义了一些与UTF-8编码有关的常量和定义，用于支持多字节编码的字符串和测试字符是否使用了扩展字符。


```cpp
#else   /* SUPPORT_UNICODE */

/* ------------------- 8-bit support  ------------------ */

#if PCRE2_CODE_UNIT_WIDTH == 8
#define MAYBE_UTF_MULTI          /* UTF chars may use multiple code units */

/* The largest UTF code point that can be encoded as a single code unit. */

#define MAX_UTF_SINGLE_CU 127

/* Tests whether the code point needs extra characters to decode. */

#define HAS_EXTRALEN(c) HASUTF8EXTRALEN(c)

```

这段代码定义了两个函数，分别用于计算字符串中的附加字符数和判断字符串是否为UTF-8编码。

第一个函数名为`GET_EXTRALEN`，它接收一个字符（`c`）并尝试使用`utf8_table4`数组中的一个元素来存储该字符的UTF-8编码。如果没有找到相应的元素，函数将返回`undefined`。该函数的作用是获取给定字符的附加字符数。

第二个函数名为`NOT_FIRSTCU`，它判断给定字符是否是UTF-8编码的第一个字符单元。如果是，函数返回`true`；否则返回`false`。

第三个函数名为`GETCHAR`，它接收一个字符（`c`）和一个字符指针（`eptr`），尝试从该字符串的下一个UTF-8编码字符开始获取字符。如果字符越界或者字符串未正确建立UTF-8编码，函数将崩溃。


```cpp
/* Returns with the additional number of characters if IS_MULTICHAR(c) is TRUE.
Otherwise it has an undefined behaviour. */

#define GET_EXTRALEN(c) (PRIV(utf8_table4)[(c) & 0x3fu])

/* Returns TRUE, if the given value is not the first code unit of a UTF
sequence. */

#define NOT_FIRSTCU(c) (((c) & 0xc0u) == 0x80u)

/* Get the next UTF-8 character, not advancing the pointer. This is called when
we know we are in UTF-8 mode. */

#define GETCHAR(c, eptr) \
  c = *eptr; \
  if (c >= 0xc0u) GETUTF8(c, eptr);

```

这段代码定义了两个UTF-8模式下的函数，用于获取下一个UTF-8字符。其中一个函数是“GETCHARTEST”，另一个是“GETCHARINC”。

这两个函数的主要区别在于，在函数体内部，对于UTF-8模式，是从第一个UTF-8字符的第一个字节开始测试的，还是在UTF-8模式下从第二个字节开始测试的。

对于“GETCHARTEST”函数，如果当前已经处于UTF-8模式，则不做任何处理，直接返回当前的字符码即可。

而对于“GETCHARINC”函数，如果当前已经处于UTF-8模式，则从当前位置开始向前推一个字节，然后继续测试。如果当前位置不处于UTF-8模式，则需要先将当前位置的字符码转换为UTF-8编码，然后从当前位置开始向前推一个字节。

总的来说，这两个函数的主要作用是用于在UTF-8模式和ASCII模式下获取下一个UTF-8字符，并决定如何处理已经进入的UTF-8模式。


```cpp
/* Get the next UTF-8 character, testing for UTF-8 mode, and not advancing the
pointer. */

#define GETCHARTEST(c, eptr) \
  c = *eptr; \
  if (utf && c >= 0xc0u) GETUTF8(c, eptr);

/* Get the next UTF-8 character, advancing the pointer. This is called when we
know we are in UTF-8 mode. */

#define GETCHARINC(c, eptr) \
  c = *eptr++; \
  if (c >= 0xc0u) GETUTF8INC(c, eptr);

/* Get the next character, testing for UTF-8 mode, and advancing the pointer.
```

当程序员在编写代码时，可能会遇到一个情况，即不知道当前是否处于 UTF-8 编码模式。为了解决这个问题，我们可以使用以下代码：

```cppc
#define GETCHARINCTEST(c, eptr)
 c = *eptr++;
 if (utf && c >= 0xc0u) GETUTF8INC(c, eptr);

/* Get the next UTF-8 character, not advancing the pointer, incrementing
length if there are extra bytes. This is called when we know we are in UTF-8 mode. */

#define GETCHARLEN(c, eptr, len)
 c = *eptr;
 if (c >= 0xc0u) GETUTF8LEN(c, eptr, len);
```

这两行代码是在判断当前是否处于 UTF-8 编码模式。如果没有确定，我们可以先将字符读取到一个临时变量中，然后检查它是否位于 UTF-8 编码的起点（即 ASCII 码 0x200B）。如果是，那么将 UTF-8 编码的逻辑过程跳过，否则执行 UTF-8 编码的逻辑过程。

如果已经确定当前处于 UTF-8 编码模式，那么这两行代码就是我们要找的答案。


```cpp
This is called when we don't know if we are in UTF-8 mode. */

#define GETCHARINCTEST(c, eptr) \
  c = *eptr++; \
  if (utf && c >= 0xc0u) GETUTF8INC(c, eptr);

/* Get the next UTF-8 character, not advancing the pointer, incrementing length
if there are extra bytes. This is called when we know we are in UTF-8 mode. */

#define GETCHARLEN(c, eptr, len) \
  c = *eptr; \
  if (c >= 0xc0u) GETUTF8LEN(c, eptr, len);

/* Get the next UTF-8 character, testing for UTF-8 mode, not advancing the
pointer, incrementing length if there are extra bytes. This is called when we
```

这段代码定义了三个宏：GETCHARLENTEST，BACKCHAR和FORWARDCHAR。同时定义了一个宏DOCTYPE。

GETCHARLENTEST的作用是获取输入字符串中的最长连续的UTF-8编码字符，并将其存储在变量c中。它首先读取一个字符eptr，然后检查当前字符是否在UTF-8编码模式中，如果是，则执行GETUTF8LEN函数，该函数将输入的c字符及其后续的字符直到字符结束的UTF-8编码长度。如果不是UTF-8编码模式，则移动字符eptr直到它位于字符串的起始位置。

BACKCHAR的作用与DOCTYPE相同，但它是将输入的字符eptr向后移动一个字符，直到它位于字符串的起始位置或者遇到了字符0。

FORWARDCHAR的作用与DOCTYPE相同，但它是将输入的字符eptr向前移动一个字符，直到它位于字符串的结束位置或者遇到了字符0。

DOCTYPE的作用是定义了一个宏，该宏在不同的输入模式下对输入的字符eptr进行移动。这个宏是在输入字符串中查找UTF-8编码模式的关键字，然后在需要时将其转换为UTF-8编码模式。


```cpp
do not know if we are in UTF-8 mode. */

#define GETCHARLENTEST(c, eptr, len) \
  c = *eptr; \
  if (utf && c >= 0xc0u) GETUTF8LEN(c, eptr, len);

/* If the pointer is not at the start of a character, move it back until
it is. This is called only in UTF-8 mode - we don't put a test within the macro
because almost all calls are already within a block of UTF-8 only code. */

#define BACKCHAR(eptr) while((*eptr & 0xc0u) == 0x80u) eptr--

/* Same as above, just in the other direction. */
#define FORWARDCHAR(eptr) while((*eptr & 0xc0u) == 0x80u) eptr++
#define FORWARDCHARTEST(eptr,end) while(eptr < end && (*eptr & 0xc0u) == 0x80u) eptr++

```

这段代码定义了两个头文件，一个是`ACROSSCHAR`，另一个是`PUTCHAR`。这两个头文件的作用如下：

1. `ACROSSCHAR`头文件：定义了一个带条件的函数，用于在满足特定条件下执行一次具体的操作。该函数接收三个参数：一个条件（`condition`），一个增强型指针（`eptr`）和一个动作（`action`）。函数内部使用了一个无条件循环，在循环条件为真时执行`action`，并将结果返回。

2. `PUTCHAR`头文件：定义了一个将字符`c`存储到内存中，并将返回的代码单元数存储在`p`指针中的函数。函数使用了`utf`标志，表示可以处理UTF-16编码的字符。如果`c`超过了UTF-16编码的字符最大长度，函数将返回4。

3. `MAYBE_UTF_MULTI`定义了一个条件，用于检查在UTF-16编码下是否可以使用多个字符编码。如果`utf`标志为真，则说明可以支持多个字符编码。


```cpp
/* Same as above, but it allows a fully customizable form. */
#define ACROSSCHAR(condition, eptr, action) \
  while((condition) && ((*eptr) & 0xc0u) == 0x80u) action

/* Deposit a character into memory, returning the number of code units. */

#define PUTCHAR(c, p) ((utf && c > MAX_UTF_SINGLE_CU)? \
  PRIV(ord2utf)(c,p) : (*p = c, 1))


/* ------------------- 16-bit support  ------------------ */

#elif PCRE2_CODE_UNIT_WIDTH == 16
#define MAYBE_UTF_MULTI          /* UTF chars may use multiple code units */

```

这段代码定义了一些宏，用于表示UTF-8编码中的字符。

首先定义了一个常量MAX_UTF_SINGLE_CU，表示一个UTF-8编码单元中最大可以表示的字符数量。

接着定义了一个宏HAS_EXTRALEN，用于检查给定的代码点是否需要额外的字符进行编码。这个宏使用了位运算，将给定的代码点与0xfc00u进行按位与操作，如果有进位，那么这个字符就是一个附加的UTF-8编码单元。

然后定义了一个宏GET_EXTRALEN，用于在给定的代码点后面添加额外的字符，如果MAX_UTF_SINGLE_CU变量为真，否则会导致 undefined behavior。这个宏使用了逻辑运算，将MAX_UTF_SINGLE_CU减去1，因为如果有附加的UTF-8编码单元，那么就需要从MAX_UTF_SINGLE_CU中减去1。

最后定义了一个测试宏IS_MULTICHAR，用于检查给定的代码点是否是UTF-8编码序列中的第一个字符。如果是，则该宏返回1，否则会导致 undefined behavior。

由于所有的宏都使用了宏定义，因此需要包含一个包含所有宏定义的定义，才能在代码中使用它们。


```cpp
/* The largest UTF code point that can be encoded as a single code unit. */

#define MAX_UTF_SINGLE_CU 65535

/* Tests whether the code point needs extra characters to decode. */

#define HAS_EXTRALEN(c) (((c) & 0xfc00u) == 0xd800u)

/* Returns with the additional number of characters if IS_MULTICHAR(c) is TRUE.
Otherwise it has an undefined behaviour. */

#define GET_EXTRALEN(c) 1

/* Returns TRUE, if the given value is not the first code unit of a UTF
sequence. */

```

这段代码定义了两个宏，分别是NOT_FIRSTCU和GETCHAR。NOT_FIRSTCU的作用是获取一个UTF-16编码的字符的低8位作为它的补码，然后判断补码是否为0xdc00u。如果是，则说明这个字符的编码是正确的，否则会将eptr指向的下一个UTF-16编码的字符的低8位。

而GETCHAR的作用是在我们已经知道我们处于UTF-16模式的情况下，获取一个UTF-16编码的字符，并将其补码赋值给c，然后判断补码是否为0xd800u。如果是，则继续获取下一个UTF-16编码的字符，否则就不再获取下一个UTF-16编码的字符了。注意，如果已经获取到了一个字符，那么GETCHAR就会返回它，而不是继续获取下一个字符。


```cpp
#define NOT_FIRSTCU(c) (((c) & 0xfc00u) == 0xdc00u)

/* Base macro to pick up the low surrogate of a UTF-16 character, not
advancing the pointer. */

#define GETUTF16(c, eptr) \
   { c = (((c & 0x3ffu) << 10) | (eptr[1] & 0x3ffu)) + 0x10000u; }

/* Get the next UTF-16 character, not advancing the pointer. This is called when
we know we are in UTF-16 mode. */

#define GETCHAR(c, eptr) \
  c = *eptr; \
  if ((c & 0xfc00u) == 0xd800u) GETUTF16(c, eptr);

```

这段代码的主要作用是测试一个字符是否为UTF-16编码。如果是UTF-16编码，那么程序会将字符的 surrogate 字段设置为0xd800，如果不是UTF-16编码，则不会移动指针。

具体来说，代码首先定义了一个名为`GETCHARTEST`的macro，该macro会获取字符的下一个UTF-16编码的 characters，同时不移动指针。

然后定义了一个名为`GETUTF16INC`的macro，该macro会在字符的低8位二进制数上递增10，并将此值与高16位的UTF-16编码合并，再加上0x10000u，最后将结果赋值给字符变量`c`，同时将`eptr`指向下一个字符的低8位。

最后，在`utf`条件语句中，使用`GETCHARTEST`macro获取字符的下一个UTF-16编码的 characters，并判断该字符是否为UTF-16编码。如果是，则执行`GETUTF16INC`macro，否则不移动指针继续下一个字符的读取。


```cpp
/* Get the next UTF-16 character, testing for UTF-16 mode, and not advancing the
pointer. */

#define GETCHARTEST(c, eptr) \
  c = *eptr; \
  if (utf && (c & 0xfc00u) == 0xd800u) GETUTF16(c, eptr);

/* Base macro to pick up the low surrogate of a UTF-16 character, advancing
the pointer. */

#define GETUTF16INC(c, eptr) \
   { c = (((c & 0x3ffu) << 10) | (*eptr++ & 0x3ffu)) + 0x10000u; }

/* Get the next UTF-16 character, advancing the pointer. This is called when we
know we are in UTF-16 mode. */

```



这段代码定义了两个宏，GETCHARINC和GETCHARINCTEST，以及一个函数，UTF16LEN。

GETCHARINC的作用是在当前已经读取一个UTF-16编码的字符中，获取下一个字符，并测试是否处于UTF-16模式。如果是UTF-16模式，则会执行GETUTF16INC函数，如果不是，则执行GETCHARINCTEST函数。

UTF16LEN是一个辅助函数，用于计算UTF-16编码的字符长度。它接受三个参数：一个已经读取到的UTF-16编码的字符c，一个指向字符eptr的指针，和一个已经读取到的字符数组长度len。函数首先将c和eptr的最高位合并，然后将c和len一起左移16位，将结果与0x10000u相加，最后将结果和len拼接在一起。

最后，定义的宏可以被其他程序或库函数使用，它们可以在编译时进行替换，而不需要修改代码。


```cpp
#define GETCHARINC(c, eptr) \
  c = *eptr++; \
  if ((c & 0xfc00u) == 0xd800u) GETUTF16INC(c, eptr);

/* Get the next character, testing for UTF-16 mode, and advancing the pointer.
This is called when we don't know if we are in UTF-16 mode. */

#define GETCHARINCTEST(c, eptr) \
  c = *eptr++; \
  if (utf && (c & 0xfc00u) == 0xd800u) GETUTF16INC(c, eptr);

/* Base macro to pick up the low surrogate of a UTF-16 character, not
advancing the pointer, incrementing the length. */

#define GETUTF16LEN(c, eptr, len) \
   { c = (((c & 0x3ffu) << 10) | (eptr[1] & 0x3ffu)) + 0x10000u; len++; }

```

这两行代码是用于检测在多字节编码模式（UTF-8或UTF-16）中是否存在低级辅助字符（ surrogate）。

当确定我们处于多字节编码模式时，这两行代码会执行以下操作：

1. 将目标字符 'c' 的一个字节（即第一个高字节）存储在变量 'eptr' 中。
2. 如果目标字符 'c' 的第一个高字节是 0xd800，即两个字节都是低级辅助字符，则调用另一个函数 `GETUTF16LEN`。
3. 如果目标字符 'c' 的第一个高字节是 0xfc00，即第一个高字节是低级辅助字符，但第二个高字节是 0xfe00 或更高，则不执行函数 `GETUTF16LEN`，而是直接返回第一个高字节的长度。

当不确定是否处于多字节编码模式时，这两行代码会执行以下操作：

1. 将目标字符 'c' 的一个字节（即第一个高字节）存储在变量 'eptr' 中。
2. 如果目标字符 'c' 的第一个高字节是 0xd800，即两个字节都是低级辅助字符，则调用另一个函数 `GETCHARLENTEST`。
3. 如果目标字符 'c' 的第一个高字节是 0xfc00，即第一个高字节是低级辅助字符，但第二个高字节是 0xfe00 或更高，则不执行函数 `GETCHARLENTEST`，而是直接返回第一个高字节的长度。


```cpp
/* Get the next UTF-16 character, not advancing the pointer, incrementing
length if there is a low surrogate. This is called when we know we are in
UTF-16 mode. */

#define GETCHARLEN(c, eptr, len) \
  c = *eptr; \
  if ((c & 0xfc00u) == 0xd800u) GETUTF16LEN(c, eptr, len);

/* Get the next UTF-816character, testing for UTF-16 mode, not advancing the
pointer, incrementing length if there is a low surrogate. This is called when
we do not know if we are in UTF-16 mode. */

#define GETCHARLENTEST(c, eptr, len) \
  c = *eptr; \
  if (utf && (c & 0xfc00u) == 0xd800u) GETUTF16LEN(c, eptr, len);

```

这段代码定义了三个宏，用于在 UTF-16 编码下对字符 pointers 进行移动。

第一个宏是 BACKCHAR，它的作用是如果指针 *eptr 不是以 0xfc00u 为开头的字符，就将其移动回直到以 0xfc00u 为开头的字符。在 UTF-16 编码模式下，这个宏只在 UTF-16 模式内部使用，因为几乎所有调用都会在 UTF-16 模式内部执行。

第二个宏是 FORWARDCHAR，它的作用与 BACKCHAR 类似，但是是从字符的结尾向前移动。

第三个宏是 FORWARDCHARTEST，它的作用与 FORWARDCHAR 类似，但是它允许在定义该宏的时候指定一个可变的结束点 eend。这个宏在 UTF-16 编码模式下只在跨越字符的结尾处移动，只有在跨越字符的结尾时才会考虑当前指针的位置。

最后一个宏是 ACROSSCHAR，它的作用是通过一个条件判断，如果指针 *eptr 在一个已经定义好的条件中为真，并且当前指针位置的 UTF-16 编码值与已经定义好的条件值匹配，则执行 ACROSSCHAR 定义的 action。这个宏在 UTF-16 编码模式下启用跨越字符的移动操作。


```cpp
/* If the pointer is not at the start of a character, move it back until
it is. This is called only in UTF-16 mode - we don't put a test within the
macro because almost all calls are already within a block of UTF-16 only
code. */

#define BACKCHAR(eptr) if ((*eptr & 0xfc00u) == 0xdc00u) eptr--

/* Same as above, just in the other direction. */
#define FORWARDCHAR(eptr) if ((*eptr & 0xfc00u) == 0xdc00u) eptr++
#define FORWARDCHARTEST(eptr,end) if (eptr < end && (*eptr & 0xfc00u) == 0xdc00u) eptr++

/* Same as above, but it allows a fully customizable form. */
#define ACROSSCHAR(condition, eptr, action) \
  if ((condition) && ((*eptr) & 0xfc00u) == 0xdc00u) action

```

这段代码定义了一个名为PUTCHAR的函数，它的作用是将一个字符c存储到内存中，并返回存储该c的代码单元数量。这个函数的条件是在32位库中，如果c大于最大可用字符单元数（即UTF-32字符集中的所有字符），则调用PRIV函数将c转换为UTF-32编码并返回。否则，将c转换为单个字符并将其赋值给p，然后返回1。

在这段注释中，还定义了一个名为MAX_UTF_SINGLE_CU的宏，表示32位库中可以存储的最大UTF-32字符单元数。另外，HAS_EXTRALEN函数用于判断一个给定的字符c是否属于UTF-32字符集中的所有字符。


```cpp
/* Deposit a character into memory, returning the number of code units. */

#define PUTCHAR(c, p) ((utf && c > MAX_UTF_SINGLE_CU)? \
  PRIV(ord2utf)(c,p) : (*p = c, 1))


/* ------------------- 32-bit support  ------------------ */

#else

/* These are trivial for the 32-bit library, since all UTF-32 characters fit
into one PCRE2_UCHAR unit. */

#define MAX_UTF_SINGLE_CU (0x10ffffu)
#define HAS_EXTRALEN(c) (0)
```

这两行代码定义了两个宏，GET_EXTRALEN和NOT_FIRSTCU，以及一个函数GETCHAR和GETCHAEST。

GET_EXTRALEN macro的作用是获取一个 UTF-32 编码的字符的下一个字符，但不将指针向前移动。即，当知道当前处于 UTF-32 编码模式时，该宏返回一个 UTF-32 编码的字符的下一个字符。

NOT_FIRSTCU macro的作用是获取一个 UTF-32 编码的字符的下一个字符，同时不将指针向前移动。即，当知道当前处于 UTF-32 编码模式时，该宏返回一个 UTF-32 编码的字符的下一个字符，但同时不返回字符本身。

GETCHAR函数的作用是获取一个 UTF-32 编码的字符的下一个字符，同时不将指针向前移动。即，当知道当前处于 UTF-32 编码模式时，该函数返回一个 UTF-32 编码的字符的下一个字符。

GETCHAEST函数的作用是获取一个 UTF-32 编码的字符的下一个字符，同时不将指针向前移动。即，当知道当前处于 UTF-32 编码模式时，该函数返回一个 UTF-32 编码的字符的下一个字符，但同时不返回字符本身。


```cpp
#define GET_EXTRALEN(c) (0)
#define NOT_FIRSTCU(c) (0)

/* Get the next UTF-32 character, not advancing the pointer. This is called when
we know we are in UTF-32 mode. */

#define GETCHAR(c, eptr) \
  c = *(eptr);

/* Get the next UTF-32 character, testing for UTF-32 mode, and not advancing the
pointer. */

#define GETCHARTEST(c, eptr) \
  c = *(eptr);

```

这段代码定义了两个函数，GETCHARINC和GETCHARINCTEST。这两个函数的作用是获取字符，并检查当前是否处于UTF-32模式。

GETCHARINC函数的作用是在当前位置获取一个UTF-32编码的字符，并将其存储在c变量中。然后，通过将eptr指向当前字符的前一个字节， 并使用自增操作符将eptr的值递增， 使得我们可以在需要获取下一个字符的时候， 通过eptr获取到下一个有效字符的位置。

而GETCHARINCTEST函数的作用是在当前位置获取一个UTF-32编码的字符，并将其存储在c变量中。然后，通过将eptr指向当前字符的前一个字节， 并使用自增操作符将eptr的值递增， 使得我们可以在需要获取下一个字符的时候， 通过eptr获取到下一个有效字符的位置。不过， GETCHARINCTEST函数并不会像GETCHARINC函数一样， 对当前字符的长度进行递增，因为此时我们并不知道当前是否处于UTF-32模式，所以该函数在此处的实现仅能获取到一个字符，而不能获取到字符的长度信息。


```cpp
/* Get the next UTF-32 character, advancing the pointer. This is called when we
know we are in UTF-32 mode. */

#define GETCHARINC(c, eptr) \
  c = *((eptr)++);

/* Get the next character, testing for UTF-32 mode, and advancing the pointer.
This is called when we don't know if we are in UTF-32 mode. */

#define GETCHARINCTEST(c, eptr) \
  c = *((eptr)++);

/* Get the next UTF-32 character, not advancing the pointer, not incrementing
length (since all UTF-32 is of length 1). This is called when we know we are in
UTF-32 mode. */

```

这段代码定义了两个 macro，GETCHARLEN和GETCHARLENTEST，它们都接受三个参数：c、eptr和len。GETCHARLEN macro 作用是获取字符到指针所经过的字节数，并将结果存储在len中。GETCHARLENTEST macro 作用是获取字符到指针所经过的字节数，并在指针不在字符串的起始位置时将指针移回至字符串的起始位置。

在GETCHARLEN macro中，首先使用GETCHAR函数获取字符c和字符eptr所组成的字符节，然后通过比较这个字符节和UTF-32模式中字符串的起始位置（0）来判断是否处于UTF-32模式。如果是，那么就直接返回这个字符节所经过的字节数。如果不是，那么就需要将字符eptr向后移动，直到字符节0出现。

在GETCHARLENTEST macro中，首先使用GETCHARTEST函数获取字符c和字符eptr所组成的字符节，然后通过比较这个字符节和UTF-32模式中字符串的起始位置（0）来判断是否处于UTF-32模式。如果是，那么就直接返回这个字符节所经过的字节数。如果不是，那么就需要将字符eptr向后移动，并记录指针所经过的字节数len。


```cpp
#define GETCHARLEN(c, eptr, len) \
  GETCHAR(c, eptr)

/* Get the next UTF-32character, testing for UTF-32 mode, not advancing the
pointer, not incrementing the length (since all UTF-32 is of length 1).
This is called when we do not know if we are in UTF-32 mode. */

#define GETCHARLENTEST(c, eptr, len) \
  GETCHARTEST(c, eptr)

/* If the pointer is not at the start of a character, move it back until
it is. This is called only in UTF-32 mode - we don't put a test within the
macro because almost all calls are already within a block of UTF-32 only
code.

```

这段代码定义了一系列无副作用的宏，用于在C语言中进行字符操作。这些宏中，所有的无副作用函数都使用了PCRE2_UCHAR类型的参数。

首先，定义了一个名为BACKCHAR的宏。它的作用是：若eptr指向了一个PCRE2_UCHAR类型的参数，则不做任何操作，直接返回。

接着，定义了一个名为FORWARDCHAR的宏。它的作用与BACKCHAR相反，即如果eptr指向了一个PCRE2_UCHAR类型的参数，则执行相应的操作，并将结果存储在eptr中。

然后，定义了一个名为FORWARDCHARTEST的宏。它的作用与FORWARDCHAR和BACKCHAR类似，但是它允许函数调用者通过重载函数的方式来定义操作的具体逻辑。具体来说，它接受三个参数：条件（作为PCRE2_UCHAR类型的参数）、目标字符串（作为PCRE2_UCHAR类型的参数）和一个可选的动作（作为PCRE2_UCHAR类型的参数）。

接下来，定义了一个名为ACROSSCHAR的宏。它的作用与FORWARDCHAR和FORWARDCHARTEST类似，但是它的操作可以在源代码中进行定义，而不仅仅是通过定义宏来实现的。它的具体实现是通过一个PCRE2_UCHAR类型的参数eptr来实现的。在执行操作时，它会检查给定的字符串是否与给定的条件字符串一致，如果一致则不做任何操作，否则执行相应的操作并将结果存储在eptr中。

最后，还有一组通用的无副作用的宏，包括eptr类型，可以用于执行各种字符操作。


```cpp
These are all no-ops since all UTF-32 characters fit into one PCRE2_UCHAR. */

#define BACKCHAR(eptr) do { } while (0)

/* Same as above, just in the other direction. */

#define FORWARDCHAR(eptr) do { } while (0)
#define FORWARDCHARTEST(eptr,end) do { } while (0)

/* Same as above, but it allows a fully customizable form. */

#define ACROSSCHAR(condition, eptr, action) do { } while (0)

/* Deposit a character into memory, returning the number of code units. */

```

这段代码定义了一系列宏，用于在代码中输出字符及其相关信息。以下是每个宏的作用：

1. `PUTCHAR`：定义了一个名为`PUTCHAR`的函数，其参数为两个字符型指针`c`和`p`。该函数的作用是将字符`c`的ASCII值存储在`p`指向的内存单元中，并将ASCII值加1。然后，通过将`c`的ASCII值存储在`p`指向的内存单元中，使得`c`仍然保留原来的ASCII值。

2. `#ifdef`和`#ifndef`：这两个宏可以看做是预处理指令。当定义这两个宏时，编译器会检查当前源文件是否已经定义过它们，如果没有定义过，则直接定义。当定义过后，如果当前源文件在定义`#ifdef`块的代码中再次定义了`#ifdef`，则将之前定义的代码作为注释，不会执行它们。而当定义过后，如果当前源文件在定义`#ifndef`块的代码中再次定义了`#ifndef`，则编译器会将之前定义的代码执行。

3. `#define PUTINC(a,n,d) PUT(a,n,d), a += LINK_SIZE`：定义了一个名为`PUTINC`的函数，其参数为两个字符型指针`a`和`n`，以及一个整型参数`d`。该函数的作用是将参数`a`中的字符及其ASCII值存储在`n`指向的内存单元中，然后将`a`向后移动`d`个字节的长度，并将移动后的`a`保留原来的ASCII值。

4. `#define PUT2INC(a,n,d) PUT2(a,n,d), a += IMM2_SIZE`：定义了一个名为`PUT2INC`的函数，其参数为两个字符型指针`a`和`n`，以及一个整型参数`d`。该函数的作用是将参数`a`中的字符及其ASCII值存储在`n`指向的内存单元中，然后将`a`向后移动`d`个字节的长度，并将移动后的`a`保留原来的ASCII值。


```cpp
#define PUTCHAR(c, p) (*p = c, 1)

#endif  /* UTF-32 character handling */
#endif  /* SUPPORT_UNICODE */


/* Mode-dependent macros that have the same definition in all modes. */

#define CU2BYTES(x)     ((x)*((PCRE2_CODE_UNIT_WIDTH/8)))
#define BYTES2CU(x)     ((x)/((PCRE2_CODE_UNIT_WIDTH/8)))
#define PUTINC(a,n,d)   PUT(a,n,d), a += LINK_SIZE
#define PUT2INC(a,n,d)  PUT2(a,n,d), a += IMM2_SIZE


/* ----------------------- HIDDEN STRUCTURES ----------------------------- */

```

这段代码定义了两个结构体，pcre2_real_general_context 和 pcre2_real_compile_context。

pcre2_real_general_context结构体包含了一个pcre2_memctl结构体，用于在内存中申请空间、设置内存保护以及获取上下文信息等。

pcre2_real_compile_context结构体包含了上述的内存控制结构体，还包含了一些 compile-time configuration settings，如最大pattern长度、BSR convention、Newline convention、Parens Nesting Limit等。

这两个结构体都是基于pcre2_memctl结构体，通过不同的方式来控制内存。其中，pcre2_real_compile_context结构体用于编译-time configuration，而pcre2_real_general_context结构体用于内存控制。


```cpp
/* NOTE: All these structures *must* start with a pcre2_memctl structure. The
code that uses them is simpler because it assumes this. */

/* The real general context structure. At present it holds only data for custom
memory control. */

typedef struct pcre2_real_general_context {
  pcre2_memctl memctl;
} pcre2_real_general_context;

/* The real compile context structure */

typedef struct pcre2_real_compile_context {
  pcre2_memctl memctl;
  int (*stack_guard)(uint32_t, void *);
  void *stack_guard_data;
  const uint8_t *tables;
  PCRE2_SIZE max_pattern_length;
  uint16_t bsr_convention;
  uint16_t newline_convention;
  uint32_t parens_nest_limit;
  uint32_t extra_options;
} pcre2_real_compile_context;

```

这段代码定义了一个名为`pcre2_real_match_context`的结构体，表示匹配上下文。该结构体包含以下成员：

1. `pcre2_memctl`是一个指向`pcre2_memctl`结构体的指针，用于控制内存映射。
2. `jit_callback`是一个指向`pcre2_jit_callback`的指针，用于在匹配过程中执行自定义的JavaScript代码。
3. `jit_callback_data`是一个指向`void *`的指针，用于存储自定义JavaScript代码的数据。
4. `callout`是一个指针类型，用于传递给`pcre2_callout_block`结构体，它是一个函数指针，需要一个输出参数和一个数据参数。
5. `callout_data`是一个指针类型，用于存储给定`callout`所要传递的数据。
6. `substitute_callout`是一个指针类型，用于传递给`pcre2_substitute_callout_block`结构体，它是一个函数指针，需要一个输入参数和一个数据参数。
7. `substitute_callout_data`是一个指针类型，用于存储给定`substitute_callout`所要传递的数据。
8. `offset_limit`是一个32位无符号整数，用于限制堆栈偏移量。
9. `heap_limit`是一个32位无符号整数，用于限制堆内存的大小。
10. `match_limit`是一个32位无符号整数，用于限制匹配 input 的大小。
11. `depth_limit`是一个32位无符号整数，用于限制匹配的深度。

这个结构体表示一个可以执行JIT 编码的PCRE2 Match上下文，可以设置一些匹配参数，如输出字符串、匹配模式、匹配优先级等。


```cpp
/* The real match context structure. */

typedef struct pcre2_real_match_context {
  pcre2_memctl memctl;
#ifdef SUPPORT_JIT
  pcre2_jit_callback jit_callback;
  void *jit_callback_data;
#endif
  int    (*callout)(pcre2_callout_block *, void *);
  void    *callout_data;
  int    (*substitute_callout)(pcre2_substitute_callout_block *, void *);
  void    *substitute_callout_data;
  PCRE2_SIZE offset_limit;
  uint32_t heap_limit;
  uint32_t match_limit;
  uint32_t depth_limit;
} pcre2_real_match_context;

```

This code defines a structure for a PCRE2-real converted context that is used for encoding PCRE2 patterns into another data type.

The structure contains three fields:

1. `memctl`: a pointer to an instance of `pcre2_memctl` that is used to manage memory when converting patterns to this data type.
2. `glob_separator`: a 32-bit field that is used to indicate the separated glue ballad and escape point in the converted pattern.
3. `glob_escape`: a 32-bit field that is used to indicate the escape point of the converted pattern.

The `pcre2_real_convert_context` structure is defined to provide the template for an instance of the `pcre2_real_convert_context` struct.

This structure is used to serialize and convert PCRE2 patterns into the `pcre2_real` data type. When converting from PCRE2-format to `pcre2_real`-format, the context is copied from the source pattern to the converted context, and then the context is converted to the `pcre2_real` format.


```cpp
/* The real convert context structure. */

typedef struct pcre2_real_convert_context {
  pcre2_memctl memctl;
  uint32_t glob_separator;
  uint32_t glob_escape;
} pcre2_real_convert_context;

/* The real compiled code structure. The type for the blocksize field is
defined specially because it is required in pcre2_serialize_decode() when
copying the size from possibly unaligned memory into a variable of the same
type. Use a macro rather than a typedef to avoid compiler warnings when this
file is included multiple times by pcre2test. LOOKBEHIND_MAX specifies the
largest lookbehind that is supported. (OP_REVERSE in a pattern has a 16-bit
argument in 8-bit and 16-bit modes, so we need no more than a 16-bit field
```

This is a struct that represents the PCRE2 Real-code fragment. It contains various fields that describe the code fragment, such as the memory control fields, the tables used to access regular expressions, the location of the executable JIT code, the code blocksize, the magic number, and various flags and options that can be passed to the PCRE2 Compile or other functions.

Here is a description of the fields in the PCRE2 Real-code struct:

- memctl: This field is a pointer to an instance of the PCRE2 Mem-control-fl盂， which is a structure that allows for more fine-grained control over memory usage than a simple memory-controlled buffer.

- tables: This field is a pointer to an array of uint8_t that holds the character tables used to look up special characters, such as Multi-byte characters or Charset Compatible Unicode characters.

- executable_jit: This field is a pointer to a location in the code where the JIT code for this code fragment is located.

- start_bitmap: This field is a pointer to an array of uint8_t that holds a bitmap starting at the given start index in the code. This allows for fast lookup of the code for a given region of the pattern.

- blocksize: This field is a pointer to an integer that represents the code blocksize in bytes. This allows for efficient memory allocation and deallocation.

- magic_number: This field is a pointer to an integer that holds a magical number used for endianness and paranoidness checks.

- compile_options: This field is a pointer to an array of uint32_t that holds the compile options used when the code is compiled, such as the level of debugging to enable.

- overall_options: This field is a pointer to an array of uint32_t that holds the post-processing options, such as whether to include debug information.

- extra_options: This field is a pointer to an array of uint32_t that holds additional options passed to the PCRE2 Compile() function.

- flags: This field is a pointer to an array of uint32_t that holds various flags for tracking the state of the code fragment, such as whether it has been processed previously or not.

- limit_heap: This field is a pointer to an integer that represents the maximum heap limit for the code fragment, and is used to prevent memory exhaustion.

- limit_match: This field is a pointer to an integer that represents the maximum number of matching characters in the pattern, and is used to prevent performance degradation.

- limit_depth: This field is a pointer to an integer that represents the maximum depth of the pattern, and is used to prevent unexpected behavior.

- first_codeunit: This field is a pointer to an integer that represents the starting code unit in the pattern, and is used to track the order of the code.

- last_codeunit: This field is a pointer to an integer that represents the ending code unit in the pattern, and is used to track the order of the code.

- bsr_convention: This field is a pointer to an integer that represents the byte order report (BSR) convention used for the magical number, and is used to track the order of the pattern.

- newline_convention: This field is a pointer to an integer that represents the newline convention, and is used to track the presence or absence of a newline character in the pattern.

- max_lookbehind: This field is a pointer to an integer that represents the maximum number of lookahead characters that can be used to prevent unexpected behavior.

- minlength: This field is a pointer to an integer that represents the minimum length of a match, and is used to prevent unexpected behavior.

- top_bracket: This field is a pointer to an integer that represents the highest numbered group in the pattern.

- top_backref: This field is a pointer to an integer that represents the highest numbered backreference in the pattern.

- name_entry_size: This field is a pointer to an integer that represents the size of each table entry in


```cpp
here.) */

#undef  CODE_BLOCKSIZE_TYPE
#define CODE_BLOCKSIZE_TYPE size_t

#undef  LOOKBEHIND_MAX
#define LOOKBEHIND_MAX UINT16_MAX

typedef struct pcre2_real_code {
  pcre2_memctl memctl;            /* Memory control fields */
  const uint8_t *tables;          /* The character tables */
  void    *executable_jit;        /* Pointer to JIT code */
  uint8_t  start_bitmap[32];      /* Bitmap for starting code unit < 256 */
  CODE_BLOCKSIZE_TYPE blocksize;  /* Total (bytes) that was malloc-ed */
  uint32_t magic_number;          /* Paranoid and endianness check */
  uint32_t compile_options;       /* Options passed to pcre2_compile() */
  uint32_t overall_options;       /* Options after processing the pattern */
  uint32_t extra_options;         /* Taken from compile_context */
  uint32_t flags;                 /* Various state flags */
  uint32_t limit_heap;            /* Limit set in the pattern */
  uint32_t limit_match;           /* Limit set in the pattern */
  uint32_t limit_depth;           /* Limit set in the pattern */
  uint32_t first_codeunit;        /* Starting code unit */
  uint32_t last_codeunit;         /* This codeunit must be seen */
  uint16_t bsr_convention;        /* What \R matches */
  uint16_t newline_convention;    /* What is a newline? */
  uint16_t max_lookbehind;        /* Longest lookbehind (characters) */
  uint16_t minlength;             /* Minimum length of match */
  uint16_t top_bracket;           /* Highest numbered group */
  uint16_t top_backref;           /* Highest numbered back reference */
  uint16_t name_entry_size;       /* Size (code units) of table entries */
  uint16_t name_count;            /* Number of name entries in the table */
} pcre2_real_code;

```

这段代码定义了一个名为`pcre2_real_match_data`的结构体，用于存储捕获字符串的中缀表达式匹配数据。

该结构体定义了一个名为`memctl`的内存控制字段，以及一个指向`pcre2_real_code`类型的指针，用于存储匹配的模板。

该结构体还定义了一个名为`subject`的指针，用于存储被匹配的字符串的主语，以及一个名为`mark`的指针，用于存储字符串中最后一个匹配的标记位置。

该结构体还定义了一个名为`heapframes`的指针，用于存储用于后退跟踪的堆栈帧。

该结构体定义了一个名为`heapframes_size`的整型变量，用于存储分配给堆栈帧的内存大小，以及一个名为`leftchar`和`rightchar`的整型变量，用于存储字符串中左端和右端代码单元的偏移量。

该结构体定义了一个名为`matchedby`的整型变量，用于存储匹配类型，以及一个名为`flags`的整型变量，包含各种匹配标志。

该结构体定义了一个名为`oveccount`的整型变量，用于存储匹配字符串中匹配项的数量，以及一个名为`rc`的整型变量，用于存储匹配返回代码。

最后，该结构体定义了一个名为`pcre2_memctl_replace`的函数，用于将`memctl`字段的值替换为字符串中指定的表达式，并将替换后的结果存储回该字段。


```cpp
/* The real match data structure. Define ovector as large as it can ever
actually be so that array bound checkers don't grumble. Memory for this
structure is obtained by calling pcre2_match_data_create(), which sets the size
as the offset of ovector plus a pair of elements for each capturable string, so
the size varies from call to call. As the maximum number of capturing
subpatterns is 65535 we must allow for 65536 strings to include the overall
match. (See also the heapframe structure below.) */

struct heapframe;  /* Forward reference */

typedef struct pcre2_real_match_data {
  pcre2_memctl     memctl;           /* Memory control fields */
  const pcre2_real_code *code;       /* The pattern used for the match */
  PCRE2_SPTR       subject;          /* The subject that was matched */
  PCRE2_SPTR       mark;             /* Pointer to last mark */
  struct heapframe *heapframes;      /* Backtracking frames heap memory */
  PCRE2_SIZE       heapframes_size;  /* Malloc-ed size */
  PCRE2_SIZE       leftchar;         /* Offset to leftmost code unit */
  PCRE2_SIZE       rightchar;        /* Offset to rightmost code unit */
  PCRE2_SIZE       startchar;        /* Offset to starting code unit */
  uint8_t          matchedby;        /* Type of match (normal, JIT, DFA) */
  uint8_t          flags;            /* Various flags */
  uint16_t         oveccount;        /* Number of pairs */
  int              rc;               /* The return code from the match */
  PCRE2_SIZE       ovector[131072];  /* Must be last in the structure */
} pcre2_real_match_data;


```

这段代码定义了两个结构体：recurse_check和parsed_recurse_check。它们都是用于在代码扫描或解析过程中检查相互递归的工具。

recurse_check结构体有一个指向前一个递归结构体的指针（prev）和一个指向要检查的组（group）。这个结构体在函数中被声明，但并没有函数体，因此它的成员变量都不会被使用。

parsed_recurse_check结构体有一个指向前一个递归结构体的指针（prev）和一个指向要检查的组指针的指针（groupptr）。这个结构体在函数中被声明，但同样没有函数体。它的成员变量中，groupptr成员表示指向要检查的组的指针，prev成员表示指向前一个递归结构体的指针。

这段代码的作用是定义两个结构体，用于在代码扫描或解析过程中检查相互递归。这些结构体可能在某些地方被依赖，但它们并没有在函数中被使用。


```cpp
/* ----------------------- PRIVATE STRUCTURES ----------------------------- */

/* These structures are not needed for pcre2test. */

#ifndef PCRE2_PCRE2TEST

/* Structures for checking for mutual recursion when scanning compiled or
parsed code. */

typedef struct recurse_check {
  struct recurse_check *prev;
  PCRE2_SPTR group;
} recurse_check;

typedef struct parsed_recurse_check {
  struct parsed_recurse_check *prev;
  uint32_t *groupptr;
} parsed_recurse_check;

```

这段代码定义了两个结构体，一个是用于在递归调用过程中构建缓存结构，另一个是用于管理递归树链中的指针。

首先，定义了一个名为`recurse_cache`的结构体，它包含两个成员变量：`PCRE2_SPTR`类型的`group`和`int`类型的`groupnumber`。`group`成员表示当前递归调用所属于的树链分组，`groupnumber`用于表示该分组中节点的数量。

接着，定义了一个名为`branch_chain`的结构体，它包含一个`struct branch_chain`类型的`outer`成员和一个`PCRE2_UCHAR`类型的`current_branch`成员。`outer`成员表示该链链的`PCRE2_SPTR`类型的指针，`current_branch`成员表示当前节点在链链中的位置。

`recurse_cache`结构体可以用于在递归调用过程中构建缓存结构。在递归调用中，每个节点都会将其`PCRE2_SPTR`类型的指针存储在`recurse_cache`中，然后在链链中插入当前节点的位置，最后将`PCRE2_SPTR`类型的指针存储为当前节点的`PCRE2_SPTR`类型的指针。这样，在递归调用结束后，我们可以通过`recurse_cache`中的指针来获取链链中所有节点的引用。

`branch_chain`结构体可以用于管理递归树链中的指针。`outer`成员表示该链链的`PCRE2_SPTR`类型的指针，`current_branch`成员表示当前节点在链链中的位置。这种结构体可以用于在编译时测试是否出现左递归，即当前节点是否在链链的左半部分。

总的来说，这段代码定义了两个结构体，一个是用于在递归调用过程中构建缓存结构，另一个是用于管理递归树链中的指针。


```cpp
/* Structure for building a cache when filling in recursion offsets. */

typedef struct recurse_cache {
  PCRE2_SPTR group;
  int groupnumber;
} recurse_cache;

/* Structure for maintaining a chain of pointers to the currently incomplete
branches, for testing for left recursion while compiling. */

typedef struct branch_chain {
  struct branch_chain *outer;
  PCRE2_UCHAR *current_branch;
} branch_chain;

```

This is a C/C++ language definition that defines the `PCRE2_DEFINE` macro which is used to define the structure of a PCRE2 pattern. The `PCRE2_DEFINE` macro takes a single argument, which is an `PCRE2_CAPSET` structure that specifies the properties of the pattern, such as the maximum backreference allowed, the backreference map, and the maximum lookbehind. The structure has several fields, including the depth of nested parentheses, the number of open capture items, and the named groups.



```cpp
/* Structure for building a list of named groups during the first pass of
compiling. */

typedef struct named_group {
  PCRE2_SPTR   name;          /* Points to the name in the pattern */
  uint32_t     number;        /* Group number */
  uint16_t     length;        /* Length of the name */
  uint16_t     isdup;         /* TRUE if a duplicate */
} named_group;

/* Structure for passing "static" information around between the functions
doing the compiling, so that they are thread-safe. */

typedef struct compile_block {
  pcre2_real_compile_context *cx;  /* Points to the compile context */
  const uint8_t *lcc;              /* Points to lower casing table */
  const uint8_t *fcc;              /* Points to case-flipping table */
  const uint8_t *cbits;            /* Points to character type table */
  const uint8_t *ctypes;           /* Points to table of type maps */
  PCRE2_SPTR start_workspace;      /* The start of working space */
  PCRE2_SPTR start_code;           /* The start of the compiled code */
  PCRE2_SPTR start_pattern;        /* The start of the pattern */
  PCRE2_SPTR end_pattern;          /* The end of the pattern */
  PCRE2_UCHAR *name_table;         /* The name/number table */
  PCRE2_SIZE workspace_size;       /* Size of workspace */
  PCRE2_SIZE small_ref_offset[10]; /* Offsets for \1 to \9 */
  PCRE2_SIZE erroroffset;          /* Offset of error in pattern */
  uint16_t names_found;            /* Number of entries so far */
  uint16_t name_entry_size;        /* Size of each entry */
  uint16_t parens_depth;           /* Depth of nested parentheses */
  uint16_t assert_depth;           /* Depth of nested assertions */
  open_capitem *open_caps;         /* Chain of open capture items */
  named_group *named_groups;       /* Points to vector in pre-compile */
  uint32_t named_group_list_size;  /* Number of entries in the list */
  uint32_t external_options;       /* External (initial) options */
  uint32_t external_flags;         /* External flag bits to be set */
  uint32_t bracount;               /* Count of capturing parentheses */
  uint32_t lastcapture;            /* Last capture encountered */
  uint32_t *parsed_pattern;        /* Parsed pattern buffer */
  uint32_t *parsed_pattern_end;    /* Parsed pattern should not get here */
  uint32_t *groupinfo;             /* Group info vector */
  uint32_t top_backref;            /* Maximum back reference */
  uint32_t backref_map;            /* Bitmap of low back refs */
  uint32_t nltype;                 /* Newline type */
  uint32_t nllen;                  /* Newline string length */
  uint32_t class_range_start;      /* Overall class range start */
  uint32_t class_range_end;        /* Overall class range end */
  PCRE2_UCHAR nl[4];               /* Newline string when fixed length */
  uint32_t req_varyopt;            /* "After variable item" flag for reqbyte */
  int  max_lookbehind;             /* Maximum lookbehind (characters) */
  BOOL had_accept;                 /* (*ACCEPT) encountered */
  BOOL had_pruneorskip;            /* (*PRUNE) or (*SKIP) encountered */
  BOOL had_recurse;                /* Had a recursion or subroutine call */
  BOOL dupnames;                   /* Duplicate names exist */
} compile_block;

```

这段代码定义了两个结构体，一个是用于在内存栈上保存赵氏栈（JIT）匹配器所需的数据结构的结构体，另一个是一个表示在递归调用过程中会出现的元组的结构体。

* `pcre2_real_jit_stack`结构体：该结构体描述了一个内存栈，用于在JIT匹配器中保存计算过程中的实参、形参、局部变量等信息。该实参和形参被保存在一个`pcre2_memctl`结构体中，它提供了一个统一的接口，用于访问和操作内存中的数据。
* `dfa_recursion_info`结构体：该结构体描述了一个在递归调用过程中出现的信息。它包含一个指向前一个递归调用信息的指针（`prevrec`），以及当前递归的上下文信息（包括一个指向正在匹配的元组的指针，以及一个表示当前匹配组编号的整数）。

这段代码的主要目的是定义两个结构体，以便在代码中更方便地使用它们。这些结构体可以在赵氏栈匹配器中使用，也可以在递归调用过程中传递上下文信息。


```cpp
/* Structure for keeping the properties of the in-memory stack used
by the JIT matcher. */

typedef struct pcre2_real_jit_stack {
  pcre2_memctl memctl;
  void* stack;
} pcre2_real_jit_stack;

/* Structure for items in a linked list that represents an explicit recursive
call within the pattern when running pcre2_dfa_match(). */

typedef struct dfa_recursion_info {
  struct dfa_recursion_info *prevrec;
  PCRE2_SPTR subject_position;
  uint32_t group_num;
} dfa_recursion_info;

```

I am not sure what you are asking. Could you please provide more context or clarify your question?


```cpp
/* Structure for "stack" frames that are used for remembering backtracking
positions during matching. As these are used in a vector, with the ovector item
being extended, the size of the structure must be a multiple of PCRE2_SIZE. The
only way to check this at compile time is to force an error by generating an
array with a negative size. By putting this in a typedef (which is never used),
we don't generate any code when all is well. */

typedef struct heapframe {

  /* The first set of fields are variables that have to be preserved over calls
  to RRMATCH(), but which do not need to be copied to new frames. */

  PCRE2_SPTR ecode;          /* The current position in the pattern */
  PCRE2_SPTR temp_sptr[2];   /* Used for short-term PCRE_SPTR values */
  PCRE2_SIZE length;         /* Used for character, string, or code lengths */
  PCRE2_SIZE back_frame;     /* Amount to subtract on RRETURN */
  PCRE2_SIZE temp_size;      /* Used for short-term PCRE2_SIZE values */
  uint32_t rdepth;           /* "Recursion" depth */
  uint32_t group_frame_type; /* Type information for group frames */
  uint32_t temp_32[4];       /* Used for short-term 32-bit or BOOL values */
  uint8_t return_id;         /* Where to go on in internal "return" */
  uint8_t op;                /* Processing opcode */

  /* At this point, the structure is 16-bit aligned. On most architectures
  the alignment requirement for a pointer will ensure that the eptr field below
  is 32-bit or 64-bit aligned. However, on m68k it is fine to have a pointer
  that is 16-bit aligned. We must therefore ensure that what comes between here
  and eptr is an odd multiple of 16 bits so as to get back into 32-bit
  alignment. This happens naturally when PCRE2_UCHAR is 8 bits wide, but needs
  fudges in the other cases. In the 32-bit case the padding comes first so that
  the occu field itself is 32-bit aligned. Without the padding, this structure
  is no longer a multiple of PCRE2_SIZE on m68k, and the check below fails. */

```

这段代码是一个C语言中的if语句，用于判断当前所使用的PCRE2编码单元的宽度是否为8或16。如果是8，则定义了一个长度为6的临时数组occu，可以用于其他情况。如果是16，则定义了一个长度为2的临时数组unused，以及一个长度为2的数组align_array，用于确保内存的32位对齐。在块注释中还定义了一个PCRE2结构体变量eptr，start_match，mark，以及一些与嵌套调用pcre2_match()有关的变量和常量。最后还定义了一个PCRE2数组ocuu，该数组长度为131072，用于存储在嵌套调用中捕获到的PCRE2数据。


```cpp
#if PCRE2_CODE_UNIT_WIDTH == 8
  PCRE2_UCHAR occu[6];       /* Used for other case code units */
#elif PCRE2_CODE_UNIT_WIDTH == 16
  PCRE2_UCHAR occu[2];       /* Used for other case code units */
  uint8_t unused[2];         /* Ensure 32-bit alignment (see above) */
#else
  uint8_t unused[2];         /* Ensure 32-bit alignment (see above) */
  PCRE2_UCHAR occu[1];       /* Used for other case code units */
#endif

  /* The rest have to be copied from the previous frame whenever a new frame
  becomes current. The final field is specified as a large vector so that
  runtime array bound checks don't catch references to it. However, for any
  specific call to pcre2_match() the memory allocated for each frame structure
  allows for exactly the right size ovector for the number of capturing
  parentheses. (See also the comment for pcre2_real_match_data above.) */

  PCRE2_SPTR eptr;           /* MUST BE FIRST */
  PCRE2_SPTR start_match;    /* Can be adjusted by \K */
  PCRE2_SPTR mark;           /* Most recent mark on the success path */
  uint32_t current_recurse;  /* Current (deepest) recursion number */
  uint32_t capture_last;     /* Most recent capture */
  PCRE2_SIZE last_group_offset;  /* Saved offset to most recent group frame */
  PCRE2_SIZE offset_top;     /* Offset after highest capture */
  PCRE2_SIZE ovector[131072]; /* Must be last in the structure */
} heapframe;

```

这段代码定义了一个自定义类型 check_heapframe_size，用于检查堆框架的大小是否是 PCRE2_SIZE 的倍数。如果大小是 PCRE2_SIZE 的倍数，则加 1，否则减 1。接着定义了一个结构体 heapframe_align，用于计算堆框架的对齐。最后定义了一个整型变量 align，表示最小对齐要求的结果。


```cpp
/* This typedef is a check that the size of the heapframe structure is a
multiple of PCRE2_SIZE. See various comments above. */

typedef char check_heapframe_size[
  ((sizeof(heapframe) % sizeof(PCRE2_SIZE)) == 0)? (+1):(-1)];

/* Structure for computing the alignment of heapframe. */

typedef struct heapframe_align {
  char unalign;    /* Completely unalign the current offset */
  heapframe frame; /* Offset is its alignment */
} heapframe_align;

/* This define is the minimum alignment required for a heapframe, in bytes. */

```

A match block is a data structure used in the PCRE2 (Parent-Child Runtime) library to store information about a match operation. It contains information about the match operation, such as the start and end positions of the subject, the name of the group, and the result of the match (success or failure).

The fields of a match block include:

- `size`: The size of the match block in the names table.
- `name_table`: A pointer to a table of group names.
- `start_code`: A pointer to the start position of the subject string.
- `start_subject`: A pointer to the start position of the subject string.
- `check_subject`: A pointer to the end position of the subject string.
- `end_subject`: A pointer to the end position of the subject string.
- `end_match_ptr`: A pointer to the end position of the match.
- `start_used_ptr`: A pointer to the earliest consulted character.
- `last_used_ptr`: A pointer to the latest consulted character.
- `mark`: A pointer to mark the next occurrence of a particular character in the subject string.
- `nomatch_mark`: A pointer to mark the next occurrence of the group name that caused the match to fail.
- `verb_ecode_ptr`: A pointer to the end code of the verb for the match.
- `verb_skip_ptr`: A pointer to the location of the next occurrence of the verb that caused the match to fail.
- `mark_ptr`: A pointer to mark the next occurrence of a particular character in the subject string.
- `callout_data`: A pointer to the data passed back to the callout.
- `cb`: A pointer to a callout block.

The `cb` field should be set to null when the match block is used to pass back data to the callout.

The `callout` field is a function or pointer to the callback function or the address of the function that will receive the data passed back.

The `callout` function should be called with the match block as an argument. It will be passed the data from the match block and can be used to perform any actions that are required after the match operation, such as updating the subject or returning a error code.


```cpp
#define HEAPFRAME_ALIGNMENT offsetof(heapframe_align, frame)

/* Structure for passing "static" information around between the functions
doing traditional NFA matching (pcre2_match() and friends). */

typedef struct match_block {
  pcre2_memctl memctl;            /* For general use */
  PCRE2_SIZE heap_limit;          /* As it says */
  uint32_t match_limit;           /* As it says */
  uint32_t match_limit_depth;     /* As it says */
  uint32_t match_call_count;      /* Number of times a new frame is created */
  BOOL hitend;                    /* Hit the end of the subject at some point */
  BOOL hasthen;                   /* Pattern contains (*THEN) */
  BOOL allowemptypartial;         /* Allow empty hard partial */
  const uint8_t *lcc;             /* Points to lower casing table */
  const uint8_t *fcc;             /* Points to case-flipping table */
  const uint8_t *ctypes;          /* Points to table of type maps */
  PCRE2_SIZE start_offset;        /* The start offset value */
  PCRE2_SIZE end_offset_top;      /* Highwater mark at end of match */
  uint16_t partial;               /* PARTIAL options */
  uint16_t bsr_convention;        /* \R interpretation */
  uint16_t name_count;            /* Number of names in name table */
  uint16_t name_entry_size;       /* Size of entry in names table */
  PCRE2_SPTR name_table;          /* Table of group names */
  PCRE2_SPTR start_code;          /* For use when recursing */
  PCRE2_SPTR start_subject;       /* Start of the subject string */
  PCRE2_SPTR check_subject;       /* Where UTF-checked from */
  PCRE2_SPTR end_subject;         /* End of the subject string */
  PCRE2_SPTR end_match_ptr;       /* Subject position at end match */
  PCRE2_SPTR start_used_ptr;      /* Earliest consulted character */
  PCRE2_SPTR last_used_ptr;       /* Latest consulted character */
  PCRE2_SPTR mark;                /* Mark pointer to pass back on success */
  PCRE2_SPTR nomatch_mark;        /* Mark pointer to pass back on failure */
  PCRE2_SPTR verb_ecode_ptr;      /* For passing back info */
  PCRE2_SPTR verb_skip_ptr;       /* For passing back a (*SKIP) name */
  uint32_t verb_current_recurse;  /* Current recurse when (*VERB) happens */
  uint32_t moptions;              /* Match options */
  uint32_t poptions;              /* Pattern options */
  uint32_t skip_arg_count;        /* For counting SKIP_ARGs */
  uint32_t ignore_skip_arg;       /* For re-run when SKIP arg name not found */
  uint32_t nltype;                /* Newline type */
  uint32_t nllen;                 /* Newline string length */
  PCRE2_UCHAR nl[4];              /* Newline string when fixed */
  pcre2_callout_block *cb;        /* Points to a callout block */
  void  *callout_data;            /* To pass back to callouts */
  int (*callout)(pcre2_callout_block *,void *);  /* Callout function or NULL */
} match_block;

```

This is a struct definition for a `dfa_match_block` that is used for similar purposes as the `DFA matching functions`. This struct contains the following fields:

* `memctl`: a `PCRE2_MemCTL` structure that can be used for general use.
* `start_code`: a `PCRE2_SPTR` that points to the start of the compiled pattern.
* `start_subject`: a `PCRE2_SPTR` that points to the start of the subject string.
* `end_subject`: a `PCRE2_SPTR` that points to the end of the subject string.
* `start_used_ptr`: a `PCRE2_SPTR` that points to the earliest consulted character.
* `last_used_ptr`: a `PCRE2_SPTR` that points to the latest consulted character.
* `tables`: a pointer to a array of characters that can be used to look up subsequent characters in the subject string.
* `start_offset`: a `PCRE2_SIZE` that specifies the start offset value for the `tables` array.
* `heap_limit`: a `PCRE2_SIZE` that specifies the limit of the heap memory used by this struct.
* `heap_used`: a `PCRE2_SIZE` that specifies the amount of heap memory used by this struct.
* `match_limit`: a `PCRE2_SIZE` that specifies the limit of the number of matches that can be performed against the compiled pattern.
* `match_limit_depth`: a `PCRE2_SIZE` that specifies the maximum depth of the match.
* `match_call_count`: a `PCRE2_SIZE` that specifies the number of calls made by the function of this struct.
* `moptions`: a `PCRE2_SIZE` that specifies the options used for matching the compiled pattern.
* `poptions`: a `PCRE2_SIZE` that specifies the options used for matching the subject string.
* `nltype`: a `PCRE2_SIZE` that specifies the type of the Newline string.
* `nlen`: a `PCRE2_SIZE` that specifies the length of the Newline string.
* `allowemptypartial`: a `PCRE2_BOOL` that specifies whether the empty and partial matches should be allowed.
* `cb`: a pointer to a callout block.
* `callout_data`: a pointer to a variable that can be passed back to the callout function.
* `int`: a return type of the callout function.
* `dfa_recursion_info`: a linked list of recursion data.


```cpp
/* A similar structure is used for the same purpose by the DFA matching
functions. */

typedef struct dfa_match_block {
  pcre2_memctl memctl;            /* For general use */
  PCRE2_SPTR start_code;          /* Start of the compiled pattern */
  PCRE2_SPTR start_subject ;      /* Start of the subject string */
  PCRE2_SPTR end_subject;         /* End of subject string */
  PCRE2_SPTR start_used_ptr;      /* Earliest consulted character */
  PCRE2_SPTR last_used_ptr;       /* Latest consulted character */
  const uint8_t *tables;          /* Character tables */
  PCRE2_SIZE start_offset;        /* The start offset value */
  PCRE2_SIZE heap_limit;          /* As it says */
  PCRE2_SIZE heap_used;           /* As it says */
  uint32_t match_limit;           /* As it says */
  uint32_t match_limit_depth;     /* As it says */
  uint32_t match_call_count;      /* Number of calls of internal function */
  uint32_t moptions;              /* Match options */
  uint32_t poptions;              /* Pattern options */
  uint32_t nltype;                /* Newline type */
  uint32_t nllen;                 /* Newline string length */
  BOOL allowemptypartial;         /* Allow empty hard partial */
  PCRE2_UCHAR nl[4];              /* Newline string when fixed */
  uint16_t bsr_convention;        /* \R interpretation */
  pcre2_callout_block *cb;        /* Points to a callout block */
  void *callout_data;             /* To pass back to callouts */
  int (*callout)(pcre2_callout_block *,void *);  /* Callout function or NULL */
  dfa_recursion_info *recursive;  /* Linked list of recursion data */
} dfa_match_block;

```

这段代码是一个PCRE2库的头文件，它定义了一个名为“pcre2_intmodedep”的函数。然而，目前没有给这个函数提供任何定义。所以，这个头文件暂时不能被使用。


```cpp
#endif  /* PCRE2_PCRE2TEST */

/* End of pcre2_intmodedep.h */

```