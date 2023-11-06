# Nmap源码解析 86

# `libpcre/src/pcre2_substitute.c`

这段代码是一个Perl兼容的正则表达式库，提供了几个常用的正则表达式函数，支持Perl 5语言的语法和语义。它被用于支持各种文本处理任务，如数据提取、数据清洗和数据验证等。


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

这段代码是一个名为`express_warnings.py`的Python文件，它是一个用于包含一些已知但可能会在代码中产生的潜在问题的警告。

该文件的作用是告知用户，这个软件是“通过copyrright持有者和贡献者”提供的，并且可能会包含“在任何情况下”产生的任何间接、特殊、示例或后果的损害。此外，该文件也明确表示不会对因为使用该软件而产生的任何责任或费用，包括商业化的违约或因使用该软件而导致的利润损失。

总结起来，这段代码是为了告知用户，该软件可能会在某些情况下产生问题，但这些问题并不是由copyright持有者和贡献者造成的，并且用户不需要承担任何责任。


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

这段代码包括以下几行：

1. `#ifdef HAVE_CONFIG_H`：这是一个预处理指令，它检查是否已经定义了`config.h`这个头文件。如果没有定义，那么代码会从`pcre2_internal.h`开始包含这个头文件。

2. `#include "config.h"`：这也是一个预处理指令，它从`config.h`这个头文件中导入了一个全局变量`config`.

3. `#include "pcre2_internal.h"`：这是引入`pcre2_internal.h`头文件的位置。

4. `#define PTR_STACK_SIZE 20`：定义了一个常量`PTR_STACK_SIZE`，它的值为20。

5. `#define SUBSTITUTE_OPTIONS`：定义了一个枚举类型`SUBSTITUTE_OPTIONS`，其中包含了一些常量，它们的值分别为：

   - `PCRE2_SUBSTITUTE_EXTENDED`
   - `PCRE2_SUBSTITUTE_GLOBAL`
   - `PCRE2_SUBSTITUTE_LITERAL`
   - `PCRE2_SUBSTITUTE_MATCHED`
   - `PCRE2_SUBSTITUTE_OVERFLOW_LENGTH`
   - `PCRE2_SUBSTITUTE_REPLACEMENT_ONLY`
   - `PCRE2_SUBSTITUTE_UNKNOWN_UNSET`
   - `PCRE2_SUBSTITUTE_UNSET_EMPTY`

6. `#define SUBSTITUTE_SUBSTITUTE`：定义了一个函数`SUBSTITUTE_SUBSTITUTE`，它接受两个参数：`PCRE2_STACK_VAR`和`PCRE2_SUBSTITUTE_OPTIONSTable`。函数的作用是，根据`PCRE2_SUBSTITUTE_OPTIONSTable`的值，用`PCRE2_SUBSTITUTE_DEFAULT`替换`PCRE2_STACK_VAR`中的字符串，直到字符串完全相同或者达到`PCRE2_SUBSTITUTE_LENGTH`。最后，返回完全替换后的字符串。

7. `#define DATA_STACK_SIZE`：定义了一个常量`DATA_STACK_SIZE`，它的值是`PTR_STACK_SIZE`的两倍，即40。

8. `#define MAX_BUFSET_SIZE`：定义了一个常量`MAX_BUFSET_SIZE`，它的值是`PCRE2_MAX_BUFSET_SIZE`的两倍，即4096。


```cpp
#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include "pcre2_internal.h"

#define PTR_STACK_SIZE 20

#define SUBSTITUTE_OPTIONS \
  (PCRE2_SUBSTITUTE_EXTENDED|PCRE2_SUBSTITUTE_GLOBAL| \
   PCRE2_SUBSTITUTE_LITERAL|PCRE2_SUBSTITUTE_MATCHED| \
   PCRE2_SUBSTITUTE_OVERFLOW_LENGTH|PCRE2_SUBSTITUTE_REPLACEMENT_ONLY| \
   PCRE2_SUBSTITUTE_UNKNOWN_UNSET|PCRE2_SUBSTITUTE_UNSET_EMPTY)



```

这段代码的作用是查找一个字符串中的结束字符，特别是在扩展模式下，它将解析包含${name:-set text:unset text}等形式的构造。该函数需要识别未转义的字符，并且扫描字符串中的嵌套构造。如果找到这样的构造，将更新指针变量code的值，以便指向新的字符串结束的位置，否则将更新指针变量ptrptr指向错误位置。

具体来说，该函数接受两个参数：一个指向编译表达式的指针code，以及一个指向字符串开始位置的指针ptrptr。函数内部使用嵌套循环遍历字符串中的所有构造，并记录下每个构造的结束位置last。如果last为TRUE，则函数已经找到了最后一个预期字符串的结束位置，可以停止遍历。否则，函数将继续遍历字符串中的所有构造，并更新code的值，以便指向下一个构造的结束位置。最后，函数返回code的值，作为参数返回。


```cpp
/*************************************************
*           Find end of substitute text          *
*************************************************/

/* In extended mode, we recognize ${name:+set text:unset text} and similar
constructions. This requires the identification of unescaped : and }
characters. This function scans for such. It must deal with nested ${
constructions. The pointer to the text is updated, either to the required end
character, or to where an error was detected.

Arguments:
  code      points to the compiled expression (for options)
  ptrptr    points to the pointer to the start of the text (updated)
  ptrend    end of the whole string
  last      TRUE if the last expected string (only } recognized)

```

This is a PHP function that appears to check a string for various escaping characters. The function takes a single parameter, `$str`, which is the string to be checked, and an optional second parameter, `$esc`. The function returns a boolean value indicating whether the string contains an escape sequence.

The function first checks if the current character is a valid escape character. If it is, the function checks the escape sequence. If it is not, the function returns the error code. If the sequence is not valid (e.g. an escape sequence ends with an invalid character), the function returns `FALSE`.

Here is the full function:
```cppphp
function ptr_has_escape($str, $esc = FALSE)
{
 static $allow_esc = [
   '"','\\','b','n','r','\n','f','\r','"','t','\t','\n','\r','\f','\r','\\','\','
   'u','\u','i','\i','j','\j','k','\k','l','\l','m','\m','n','\n','\p','\p','
   '`','\p','[','\ ]','\\',''','0','1','2','3','4','5','6','7','8','9',''
 ];

 $esc = is_array($esc);

 if (!$esc)
   return FALSE;

 $result = '';

 foreach ($esc as $key)
 {
   if (in_array($key, $allow_esc))
   {
     $result .= $key;
   }
 }

 return $result == 'F' ? FALSE : TRUE;
}
```


```cpp
Returns:    0 on success
            negative error code on failure
*/

static int
find_text_end(const pcre2_code *code, PCRE2_SPTR *ptrptr, PCRE2_SPTR ptrend,
  BOOL last)
{
int rc = 0;
uint32_t nestlevel = 0;
BOOL literal = FALSE;
PCRE2_SPTR ptr = *ptrptr;

for (; ptr < ptrend; ptr++)
  {
  if (literal)
    {
    if (ptr[0] == CHAR_BACKSLASH && ptr < ptrend - 1 && ptr[1] == CHAR_E)
      {
      literal = FALSE;
      ptr += 1;
      }
    }

  else if (*ptr == CHAR_RIGHT_CURLY_BRACKET)
    {
    if (nestlevel == 0) goto EXIT;
    nestlevel--;
    }

  else if (*ptr == CHAR_COLON && !last && nestlevel == 0) goto EXIT;

  else if (*ptr == CHAR_DOLLAR_SIGN)
    {
    if (ptr < ptrend - 1 && ptr[1] == CHAR_LEFT_CURLY_BRACKET)
      {
      nestlevel++;
      ptr += 1;
      }
    }

  else if (*ptr == CHAR_BACKSLASH)
    {
    int erc;
    int errorcode;
    uint32_t ch;

    if (ptr < ptrend - 1) switch (ptr[1])
      {
      case CHAR_L:
      case CHAR_l:
      case CHAR_U:
      case CHAR_u:
      ptr += 1;
      continue;
      }

    ptr += 1;  /* Must point after \ */
    erc = PRIV(check_escape)(&ptr, ptrend, &ch, &errorcode,
      code->overall_options, code->extra_options, FALSE, NULL);
    ptr -= 1;  /* Back to last code unit of escape */
    if (errorcode != 0)
      {
      rc = errorcode;
      goto EXIT;
      }

    switch(erc)
      {
      case 0:      /* Data character */
      case ESC_E:  /* Isolated \E is ignored */
      break;

      case ESC_Q:
      literal = TRUE;
      break;

      default:
      rc = PCRE2_ERROR_BADREPESCAPE;
      goto EXIT;
      }
    }
  }

```

the previous function, and the last two arguments are specific to this function.

This function takes a pointer to a compile-time regular expression (PCRE2) and a pointer to a character pointer (ptr), and returns the result of matching and substituting the subject string against the PCRE2 re.

The first six arguments are the same as before, and they correspond to the following:

* PCRE2\_CANDIDATE\_EXPR: This is the PCRE2 re expression that is used for the match.
* PCRE2\_CONTROL\_FLAGS: These are flags that control the matching process, such as whether to perform a byte-for-byte或者block-based match.
* PCRE2\_NAMER\_FLAGS: These are flags that control the naming of the matches.
* PCRE2\_SPTR\_FLAGS: These are flags that control the tracking of variable-length substrings.
* PCRE2\_TRACT\_FROM\_LICENSE: This flag indicates whether to subtract the PCRE2 license from the result.
* PCRE2\_ACTIVITY: This flag indicates the activity of the PCRE2 re, and whether it is active or not.

The last two arguments are the result of returning the new string with the substitutions. These last two arguments are used to create a new string with the substitutions made to the original string by the PCRE2 re.

EXIT:

This is a simple function that returns an exit code (usually 0) indicating the success or failure of the matching and substitution operation.


```cpp
rc = PCRE2_ERROR_REPMISSINGBRACE;   /* Terminator not found */

EXIT:
*ptrptr = ptr;
return rc;
}



/*************************************************
*              Match and substitute              *
*************************************************/

/* This function applies a compiled re to a subject string and creates a new
string with substitutions. The first 7 arguments are the same as for
```

这段代码是用于在给定的编译表达式中查找字符串匹配的 PCRE2 函数。函数接受一个指向编译表达式的指针，一个指向主题字符串的指针，主题字符串的长度，以及一个指向字符串开始偏移量的指针。函数内部使用了 points to a match\_data block 或者 is NULL 来检查匹配数据是否有效，然后使用替换字符串的长度来更新匹配数据。最后，将替换后的字符串存储到缓冲区中。


```cpp
pcre2_match(). Either string length may be PCRE2_ZERO_TERMINATED.

Arguments:
  code            points to the compiled expression
  subject         points to the subject string
  length          length of subject string (may contain binary zeros)
  start_offset    where to start in the subject string
  options         option bits
  match_data      points to a match_data block, or is NULL
  context         points a PCRE2 context
  replacement     points to the replacement string
  rlength         length of replacement string
  buffer          where to put the substituted string
  blength         points to length of buffer; updated to length of string

```

这段代码是一个C语言中的一个宏定义，名为“CHECKMEMCPY”。它的作用是检查一个缓冲区（变量from）中是否可以复制它所指定的长度（变量length）的字符，并在必要时输出错误信息。

该宏定义内部包含多个条件判断，用于判断缓冲区是否可以复制到目标缓冲区。具体来说，它首先检查缓冲区是否已经被溢出，如果是，就直接返回，否则继续判断。

接着，它会检查目标缓冲区是否可以容纳缓冲区中的所有字符（即length是否大于等于0），如果是，那么尝试将缓冲区中的所有字符复制到目标缓冲区，并更新缓冲区的偏移量和长度。最后，如果缓冲区仍然小于目标缓冲区，那么会累积超过缓冲区长度的字符，并尝试在下一次复制前将其计算出来。


```cpp
Returns:          >= 0 number of substitutions made
                  < 0 an error code
                  PCRE2_ERROR_BADREPLACEMENT means invalid use of $
*/

/* This macro checks for space in the buffer before copying into it. On
overflow, either give an error immediately, or keep on, accumulating the
length. */

#define CHECKMEMCPY(from,length) \
  { \
  if (!overflowed && lengthleft < length) \
    { \
    if ((suboptions & PCRE2_SUBSTITUTE_OVERFLOW_LENGTH) == 0) goto NOROOM; \
    overflowed = TRUE; \
    extra_needed = length - lengthleft; \
    } \
  else if (overflowed) \
    { \
    extra_needed += length; \
    }  \
  else \
    {  \
    memcpy(buffer + buff_offset, from, CU2BYTES(length)); \
    buff_offset += length; \
    lengthleft -= length; \
    } \
  }

```

这段代码是一个名为"PCRE2_CALL_CONVENTION"的函数，属于PCRE2库。它实现了PCRE2函数的调用转换。

函数接收一个PCRE2编码指针code，一个指向subject（匹配短语）的PCRE2_SPTR，一个匹配长度offset，以及一些选项（如match_data、mcontext、replacement等）。函数内部根据传入的选项执行不同的操作，然后返回匹配到的数据。

具体来说，这段代码实现以下操作：

1. 将PCRE2编码指针code转换为PCRE2编码表示，并将其赋值给PCRE2_SPTR subject。
2. 将offset和匹配长度offset设置为0，并将forcecase设置为0。
3. 读取选项中包含的vector_count，并根据vector_count执行相应的操作。
4. 执行forcecasereset后，如果vector_count为0，则将forcecase设置为1，并重置为0。
5. 执行option，并根据option设置goptions和suboptions。
6. 调用PCRE2库中的函数，传入相应的参数，并将其赋值给PCRE2_SPTR replacement 和PCRE2_SPTR buffer。
7. 返回PCRE2_MATCH_DATA类型的match_data。
8. 返回PCRE2_MATCH_CONTEXT类型的mcontext。
9. 将PCRE2_SPTR subject 和replacement作为PCRE2_SPTR返回。

最后，需要注意的是，该函数是在PCRE2_EXP_DEFN宏定义的基础上实现的，因此在函数内部声明的变量和参数可能与该定义不符。


```cpp
/* Here's the function */

PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_substitute(const pcre2_code *code, PCRE2_SPTR subject, PCRE2_SIZE length,
  PCRE2_SIZE start_offset, uint32_t options, pcre2_match_data *match_data,
  pcre2_match_context *mcontext, PCRE2_SPTR replacement, PCRE2_SIZE rlength,
  PCRE2_UCHAR *buffer, PCRE2_SIZE *blength)
{
int rc;
int subs;
int forcecase = 0;
int forcecasereset = 0;
uint32_t ovector_count;
uint32_t goptions = 0;
uint32_t suboptions;
```

这段代码定义了一个名为 internal_match_data 的 pointer变量，用于存储 PCRE2_MATCH_DATA 结构体。

这个结构体可能用于存储匹配到的文本数据，例如在替换操作中。

接下来代码定义了一些布尔变量，用于控制是否溢出以及是否使用现有匹配。

然后定义了一个名为 use_existing_match 的布尔变量，表示是否使用已有的匹配结果。

接着定义了一个名为 utf 的布尔变量，表示是否支持 Unicode 编码。

然后定义了一个名为 ucp 的布尔变量，表示是否支持合规编码（UTF-8或UTF-16）。

接下来定义了一个名为 temp 的字符数组，用于存储匹配到的数据。

接着定义了一个名为 ptr 的字符串指针变量，用于存储匹配到的数据。

接着定义了一个名为 repend 的字符串指针变量，用于将匹配到的数据添加到 repend 中。

接下来定义了一些变量用于计算需要多少 extra 字节，以及如何计算长度、偏移量等。

最后，使用 internal_match_data 变量，存储了匹配到的数据，它可以被用于替换操作。


```cpp
pcre2_match_data *internal_match_data = NULL;
BOOL escaped_literal = FALSE;
BOOL overflowed = FALSE;
BOOL use_existing_match;
BOOL replacement_only;
#ifdef SUPPORT_UNICODE
BOOL utf = (code->overall_options & PCRE2_UTF) != 0;
BOOL ucp = (code->overall_options & PCRE2_UCP) != 0;
#endif
PCRE2_UCHAR temp[6];
PCRE2_SPTR ptr;
PCRE2_SPTR repend;
PCRE2_SIZE extra_needed = 0;
PCRE2_SIZE buff_offset, buff_length, lengthleft, fraglength;
PCRE2_SIZE *ovector;
```

这段代码定义了一个名为"PCRE2_SIZE"的整型变量和一个名为"scb"的名为"pcre2_substitute_callout_block"的结构体。接下来，定义了一些整型变量和两个整型指针，分别命名为"buff_offset"、"lengthleft"和"ovecsave"。

紧接着，对选项进行了判断，如果选项中包含了"PCRE2_PARTIAL_HARD"或"PCRE2_PARTIAL_SOFT"，则执行以下操作：

1. 如果PCRE2_PARTIAL_HARD为1并且当前已经设置了"blength"，则认为部分匹配有效，返回PCRE2_ERROR_BADOPTION。

2. 否则，直接跳过部分匹配，不会执行任何操作。

3. 最后，定义了三个整型变量分别命名为"PCRE2_PARTIAL_HARD"、"PCRE2_PARTIAL_SOFT"和PCRE2_UNSET，分别对应着选项中的"PCRE2_PARTIAL_HARD"、"PCRE2_PARTIAL_SOFT"和"PCRE2_UNSET"。


```cpp
PCRE2_SIZE ovecsave[3];
pcre2_substitute_callout_block scb;

/* General initialization */

buff_offset = 0;
lengthleft = buff_length = *blength;
*blength = PCRE2_UNSET;
ovecsave[0] = ovecsave[1] = ovecsave[2] = PCRE2_UNSET;

/* Partial matching is not valid. This must come after setting *blength to
PCRE2_UNSET, so as not to imply an offset in the replacement. */

if ((options & (PCRE2_PARTIAL_HARD|PCRE2_PARTIAL_SOFT)) != 0)
  return PCRE2_ERROR_BADOPTION;

```

这段代码的主要作用是检查字符串中是否包含某种长度的子串，并找到该子串的起始和结束位置，如果找到该子串，则将其复制到目标字符串中。如果该子串的长度为0，则被认为是空字符串，代码将返回PCRE2_ERROR_NULL错误。


```cpp
/* Validate length and find the end of the replacement. A NULL replacement of
zero length is interpreted as an empty string. */

if (replacement == NULL)
  {
  if (rlength != 0) return PCRE2_ERROR_NULL;
  replacement = (PCRE2_SPTR)"";
  }

if (rlength == PCRE2_ZERO_TERMINATED) rlength = PRIV(strlen)(replacement);
repend = replacement + rlength;

/* Check for using a match that has already happened. Note that the subject
pointer in the match data may be NULL after a no-match. */

```

这段代码是用来判断两个PCRE2函数选项中是否包含“使用现有匹配”和“仅替换匹配”。如果包含，则表示要使用现有的匹配数据，否则表示仅替换匹配数据。

具体来说，如果使用现有匹配数据，则需要创建一个内部匹配数据结构。这个内部匹配数据结构包含一个指向现有匹配数据块的指针，以及一个包含从现有匹配数据中复制出来的数据的所有其他选项。如果使用的是仅替换匹配数据，则只需要对现有的匹配数据进行复制，不需要进行额外的处理。

如果从现有匹配数据中无法获取匹配数据，则返回PCRE2_ERROR_NULL。如果使用现有匹配数据，但选项中包含“仅替换匹配”，则返回PCRE2_ERROR_NOMEMORY。


```cpp
use_existing_match = ((options & PCRE2_SUBSTITUTE_MATCHED) != 0);
replacement_only = ((options & PCRE2_SUBSTITUTE_REPLACEMENT_ONLY) != 0);

/* If starting from an existing match, there must be an externally provided
match data block. We create an internal match_data block in two cases: (a) an
external one is not supplied (and we are not starting from an existing match);
(b) an existing match is to be used for the first substitution. In the latter
case, we copy the existing match into the internal block, except for any cached
heap frame size and pointer. This ensures that no changes are made to the
external match data block. */

if (match_data == NULL)
  {
  pcre2_general_context *gcontext;
  if (use_existing_match) return PCRE2_ERROR_NULL;
  gcontext = (mcontext == NULL)?
    (pcre2_general_context *)code :
    (pcre2_general_context *)mcontext;
  match_data = internal_match_data =
    pcre2_match_data_create_from_pattern(code, gcontext);
  if (internal_match_data == NULL) return PCRE2_ERROR_NOMEMORY;
  }

```

这段代码的作用是检查给定的匹配数据中，是否存在使用现有匹配数据覆盖现有匹配数据的情况。如果存在，则执行特定的操作，否则返回一个错误码。

具体来说，代码首先检查给定的 `use_existing_match` 是否为真。如果是，代码将使用现有匹配数据，否则将创建一个新的匹配数据。

如果使用现有匹配数据，代码将根据匹配数据中的 `oveccount` 计算出要查找的两端括号位置 `pairs`，并将该位置对匹配数据进行翻转。然后，代码将使用 `pcre2_match_data_create` 函数创建一个新的内部匹配数据，该数据与给定的匹配数据拥有相同的 `oveccount` 且包含两端括号经过翻转后的匹配数据。

接下来，代码将复制匹配数据中两端括号之间的所有内容，并将其存储到内部匹配数据中。此外，内部匹配数据还将包含一个指向堆内存分配区域的指针 `heapframes`，以及一个指示 `heapframes` 的大小关系的计数器 `heapframes_size`。

最后，如果内部匹配数据包含翻转后的匹配数据，则代码将跳回匹配数据中，否则将返回复合错误码。


```cpp
else if (use_existing_match)
  {
  pcre2_general_context *gcontext = (mcontext == NULL)?
    (pcre2_general_context *)code :
    (pcre2_general_context *)mcontext;
  int pairs = (code->top_bracket + 1 < match_data->oveccount)?
    code->top_bracket + 1 : match_data->oveccount;
  internal_match_data = pcre2_match_data_create(match_data->oveccount,
    gcontext);
  if (internal_match_data == NULL) return PCRE2_ERROR_NOMEMORY;
  memcpy(internal_match_data, match_data, offsetof(pcre2_match_data, ovector)
    + 2*pairs*sizeof(PCRE2_SIZE));
  internal_match_data->heapframes = NULL;
  internal_match_data->heapframes_size = 0;
  match_data = internal_match_data;
  }

```

这段代码的作用是实现PCRE2库中的一部分函数，其主要目的是从给定的匹配数据中提取CORE动机(CVRE2)的输入矢量，并将其存储在名为“ovector”的指针中。以下是代码的主要步骤：

1.从PCRE2库中获取匹配数据对象的指针变量“match_data”，并将其存储在名为“ovector”的指针中。

2.从PCRE2库中获取匹配数据对象中CORE动机输入矢量的长度，并将其存储在名为“ovector_count”的变量中。

3.定义一个名为“scb”的结构体变量，其包含了一些描述性常量，包括匹配算法版本号、输入数据指针、输出数据指针和CORE动机输入指针。

4.使用PCRE2库中的函数“pcre2_get_ovector_pointer”和“pcre2_get_ovector_count”提取CORE动机输入矢量和长度，并将它们存储在变量“ovector”和“ovector_count”中，从而使结构体变量中的“scb.input”和“scb.output”和“scb.ovector”和“scb.ivector_count”都指向了匹配数据对象。

5.将提取的CORE动机输入矢量存储在名为“buffer”的指针中，从而使结构体变量中的“scb.output”指针指向了匹配数据对象的输出。

6.通过“PCRE2_ERROR_ACTIVITY”检查匹配数据对象是否为空字符串。如果是，则表明算法没有找到任何CORE动机数据，此时将返回PCRE2_ERROR_ACTIVITY。否则，将更新结构体变量中的“scb.version”为0，以便记住该匹配未被激活。

7.最后，将提取的CORE动机输入矢量返回给用户。


```cpp
/* Remember ovector details */

ovector = pcre2_get_ovector_pointer(match_data);
ovector_count = pcre2_get_ovector_count(match_data);

/* Fixed things in the callout block */

scb.version = 0;
scb.input = subject;
scb.output = (PCRE2_SPTR)buffer;
scb.ovector = ovector;

/* A NULL subject of zero length is treated as an empty string. */

if (subject == NULL)
  {
  if (length != 0) return PCRE2_ERROR_NULL;
  subject = (PCRE2_SPTR)"";
  }

```

这段代码的作用是检查给定的字符串是否为零结尾，并替换掉UTF-8编码中不在UTF-8编码中的字符。

具体来说，代码首先检查给定的字符串是否为零结尾，如果是则执行下面的操作：

```cpp
length = subject? PRIV(strlen)(subject) : 0;
```

如果给定的字符串不是零结尾，则执行下面的操作：

```cpp
if (length == PCRE2_ZERO_TERMINATED)
 length = subject? PRIV(strlen)(subject) : 0;
```

这里使用了PCRE2库中的一个名为`PCRE2_ZERO_TERMINATED`的函数，它会判断一个字符串是否为零结尾。如果是，则将`length`的值设置为该字符串的长度。否则，将`length`的值设置为0。

接下来，代码检查是否需要替换UTF-8编码中的字符。具体来说，先检查给定的字符串是否使用了UTF-8编码，如果不是，则执行下面的操作：

```cpp
#ifdef SUPPORT_UNICODE
if (utf && (options & PCRE2_NO_UTF_CHECK))
 {
   if (!UTF8_QUAL(utf, "utf-8"))
     rc = PRIV(valid_utf)(replacement, rlength, &(match_data->startchar));
   if (rc != 0)
     {
       match_data->leftchar = 0;
       goto EXIT;
     }
 }
```

这里使用了两个条件判断，第一个条件判断是否支持Unicode编码，如果是，则执行下面的操作。第二个条件判断给定的字符串是否使用UTF-8编码，如果不是，则使用`PRIV(valid_utf)`函数将输入的字符串转换为UTF-8编码，并使用`PCRE2_NO_UTF_CHECK`选项去除对UTF-8编码的检查。然后使用`PRIV(valid_utf)`函数将输入的字符串转换为UTF-8编码，并使用`match_data->startchar`变量替换掉UTF-8编码中的字符。最后，如果替换后的字符串仍然有效，则将`match_data->leftchar`变量设置为0，并跳转到`EXIT`标签。否则，继续执行之前的操作。

综上所述，这段代码的作用是检查给定的字符串是否为零结尾，并替换掉UTF-8编码中不在UTF-8编码中的字符，只有使用UTF-8编码时才替换字符。


```cpp
/* Find length of zero-terminated subject */

if (length == PCRE2_ZERO_TERMINATED)
  length = subject? PRIV(strlen)(subject) : 0;

/* Check UTF replacement string if necessary. */

#ifdef SUPPORT_UNICODE
if (utf && (options & PCRE2_NO_UTF_CHECK) == 0)
  {
  rc = PRIV(valid_utf)(replacement, rlength, &(match_data->startchar));
  if (rc != 0)
    {
    match_data->leftchar = 0;
    goto EXIT;
    }
  }
```

这段代码的主要作用是检查一个正则表达式模式（$options）中是否定义了 "utf8" 选项（#define SUPPORT_UNICODE），如果没有定义，则定义。同时，它还实现了从 "options" 复制 "utf8" 选项，并从 "match options" 中移除 "utf8" 选项。


```cpp
#endif  /* SUPPORT_UNICODE */

/* Save the substitute options and remove them from the match options. */

suboptions = options & SUBSTITUTE_OPTIONS;
options &= ~SUBSTITUTE_OPTIONS;

/* Error if the start match offset is greater than the length of the subject. */

if (start_offset > length)
  {
  match_data->leftchar = 0;
  rc = PCRE2_ERROR_BADOFFSET;
  goto EXIT;
  }

```

这段代码的主要作用是执行一个字符串替换操作，其中替换操作将在给定的起始偏移处开始，除非只需要替换字符。

具体来说，代码首先检查是否只需要替换字符，如果是，那么将起始偏移量设置为给定的偏移量，然后使用CHECKMEMCPY函数将该字符串的起始偏移量及之后的字符复制到subject缓冲区中。

否则，代码将执行一个循环，该循环将使用PCRE2_SUBSTITUTE_MATCHED功能进行字符串替换。该函数将在传递给它的 match_data 变量中指定的起始偏移量和终止偏移量进行匹配。如果使用了现有的匹配，那么将 `use_existing_match` 设置为假，并使用 `match_data` 中的起始偏移量开始匹配。如果匹配成功，则代码将更新 `subs` 计数器，以便在循环结束时计算总替换字符数。


```cpp
/* Copy up to the start offset, unless only the replacement is required. */

if (!replacement_only) CHECKMEMCPY(subject, start_offset);

/* Loop for global substituting. If PCRE2_SUBSTITUTE_MATCHED is set, the first
match is taken from the match_data that was passed in. */

subs = 0;
do
  {
  PCRE2_SPTR ptrstack[PTR_STACK_SIZE];
  uint32_t ptrstackptr = 0;

  if (use_existing_match)
    {
    rc = match_data->rc;
    use_existing_match = FALSE;
    }
  else rc = pcre2_match(code, subject, length, start_offset, options|goptions,
    match_data, mcontext);

```

这段代码是一个PCRE2编程库中的一个函数，它的作用是检查输入字符串是否支持Unicode编码。以下是这段代码的功能解释：

1. 首先定义了一个预处理指令，当使用支持Unicode编码的PCRE2库时，将`utf`选项的二进制值设置为1，从而只在输入字符串的第一次Unicode编码检查时执行PCRE2库中的检查。

2. 接下来定义了一个错误处理函数，当输入字符串中没有找到匹配的规则时，函数将返回PCRE2库中的error码，如果没有错误则继续处理。

3. 在函数内部，首先检查输入字符串是否为空，如果是，函数将结束。否则，将开始计数器初始化为字符串长度减1，将当前字符作为输出并将计数器加1，尝试匹配下一个字符。

4. 如果得到的结果不满足条件，则函数将执行以下操作：首先，将之前的结果保存到一个名为`save_start`的PCRE2变量中。然后，检查当前是否已经到达字符串的结尾，如果是，函数将跳过当前的循环。

5. 如果当前已经到达字符串的结尾，但仍然没有找到匹配的规则，函数将继续执行以下操作：将`save_start`的值从当前计数器中减1，并将字符`code->newline_convention`的值（如果是`PCRE2_NEWLINE_CR`或`PCRE2_NEWLINE_LF`）与当前计数器中的值进行比较。如果是`PCRE2_NEWLINE_CR`，则将计数器加1并尝试继续后退；如果是`PCRE2_NEWLINE_LF`，则说明已经到达字符串的结尾，且仍然需要继续尝试，因此将计数器继续加1。

6. 如果`code->newline_convention`为`PCRE2_NEWLINE_CR`，则说明在当前字符的后面有特殊空格序列或者Unicode编码的字符，函数需要将计数器继续后退。否则，如果当前字符是UTF-8编码的字符，则无论是否匹配字符串的Unicode编码，函数都需要继续执行以下操作：将`save_start`的值从当前计数器中减1，并将`code->newline_convention`的值（如果是`PCRE2_NEWLINE_CR`或`PCRE2_NEWLINE_LF`）与当前计数器中的值进行比较。如果是`PCRE2_NEWLINE_CR`，则将计数器加1并尝试继续后退；如果是`PCRE2_NEWLINE_LF`，则说明已经到达字符串的结尾，且仍然需要继续尝试，因此将计数器继续加1。

7. 如果当前已经到达字符串的结尾，且仍然没有找到匹配的规则，函数将结束。


```cpp
#ifdef SUPPORT_UNICODE
  if (utf) options |= PCRE2_NO_UTF_CHECK;  /* Only need to check once */
#endif

  /* Any error other than no match returns the error code. No match when not
  doing the special after-empty-match global rematch, or when at the end of the
  subject, breaks the global loop. Otherwise, advance the starting point by one
  character, copying it to the output, and try again. */

  if (rc < 0)
    {
    PCRE2_SIZE save_start;

    if (rc != PCRE2_ERROR_NOMATCH) goto EXIT;
    if (goptions == 0 || start_offset >= length) break;

    /* Advance by one code point. Then, if CRLF is a valid newline sequence and
    we have advanced into the middle of it, advance one more code point. In
    other words, do not start in the middle of CRLF, even if CR and LF on their
    own are valid newlines. */

    save_start = start_offset++;
    if (subject[start_offset-1] == CHAR_CR &&
        code->newline_convention != PCRE2_NEWLINE_CR &&
        code->newline_convention != PCRE2_NEWLINE_LF &&
        start_offset < length &&
        subject[start_offset] == CHAR_LF)
      start_offset++;

    /* Otherwise, in UTF mode, advance past any secondary code points. */

    else if ((code->overall_options & PCRE2_UTF) != 0)
      {
```

It looks like this is a C-style program that performs a pattern search and replacement. The program is using the PCRE2 library, which is a C-compatible regex library. The `PCRE2_ERROR_UNSET` and `PCRE2_ERROR_SETTLE` constants are likely defined by this library, and they indicate that an error has occurred or that the集玛() have been entered. The `rc` variable is likely used to keep track of the current error status.


```cpp
#if PCRE2_CODE_UNIT_WIDTH == 8
      while (start_offset < length && (subject[start_offset] & 0xc0) == 0x80)
        start_offset++;
#elif PCRE2_CODE_UNIT_WIDTH == 16
      while (start_offset < length &&
            (subject[start_offset] & 0xfc00) == 0xdc00)
        start_offset++;
#endif
      }

    /* Copy what we have advanced past (unless not required), reset the special
    global options, and continue to the next match. */

    fraglength = start_offset - save_start;
    if (!replacement_only) CHECKMEMCPY(subject + save_start, fraglength);
    goptions = 0;
    continue;
    }

  /* Handle a successful match. Matches that use \K to end before they start
  or start before the current point in the subject are not supported. */

  if (ovector[1] < ovector[0] || ovector[0] < start_offset)
    {
    rc = PCRE2_ERROR_BADSUBSPATTERN;
    goto EXIT;
    }

  /* Check for the same match as previous. This is legitimate after matching an
  empty string that starts after the initial match offset. We have tried again
  at the match point in case the pattern is one like /(?<=\G.)/ which can never
  match at its starting point, so running the match achieves the bumpalong. If
  we do get the same (null) match at the original match point, it isn't such a
  pattern, so we now do the empty string magic. In all other cases, a repeat
  match should never occur. */

  if (ovecsave[0] == ovector[0] && ovecsave[1] == ovector[1])
    {
    if (ovector[0] == ovector[1] && ovecsave[2] != start_offset)
      {
      goptions = PCRE2_NOTEMPTY_ATSTART | PCRE2_ANCHORED;
      ovecsave[2] = start_offset;
      continue;    /* Back to the top of the loop */
      }
    rc = PCRE2_ERROR_INTERNAL_DUPMATCH;
    goto EXIT;
    }

  /* Count substitutions with a paranoid check for integer overflow; surely no
  real call to this function would ever hit this! */

  if (subs == INT_MAX)
    {
    rc = PCRE2_ERROR_TOOMANYREPLACE;
    goto EXIT;
    }
  subs++;

  /* Copy the text leading up to the match (unless not required), and remember
  where the insert begins and how many ovector pairs are set. */

  if (rc == 0) rc = ovector_count;
  fraglength = ovector[0] - start_offset;
  if (!replacement_only) CHECKMEMCPY(subject + start_offset, fraglength);
  scb.output_offsets[0] = buff_offset;
  scb.oveccount = rc;

  /* Process the replacement string. If the entire replacement is literal, just
  copy it with length check. */

  ptr = replacement;
  if ((suboptions & PCRE2_SUBSTITUTE_LITERAL) != 0)
    {
    CHECKMEMCPY(ptr, rlength);
    }

  /* Within a non-literal replacement, which must be scanned character by
  character, local literal mode can be set by \Q, but only in extended mode
  when backslashes are being interpreted. In extended mode we must handle
  nested substrings that are to be reprocessed. */

  else for (;;)
    {
    uint32_t ch;
    unsigned int chlen;

    /* If at the end of a nested substring, pop the stack. */

    if (ptr >= repend)
      {
      if (ptrstackptr == 0) break;       /* End of replacement string */
      repend = ptrstack[--ptrstackptr];
      ptr = ptrstack[--ptrstackptr];
      continue;
      }

    /* Handle the next character */

    if (escaped_literal)
      {
      if (ptr[0] == CHAR_BACKSLASH && ptr < repend - 1 && ptr[1] == CHAR_E)
        {
        escaped_literal = FALSE;
        ptr += 2;
        continue;
        }
      goto LOADLITERAL;
      }

    /* Not in literal mode. */

    if (*ptr == CHAR_DOLLAR_SIGN)
      {
      int group, n;
      uint32_t special = 0;
      BOOL inparens;
      BOOL star;
      PCRE2_SIZE sublength;
      PCRE2_SPTR text1_start = NULL;
      PCRE2_SPTR text1_end = NULL;
      PCRE2_SPTR text2_start = NULL;
      PCRE2_SPTR text2_end = NULL;
      PCRE2_UCHAR next;
      PCRE2_UCHAR name[33];

      if (++ptr >= repend) goto BAD;
      if ((next = *ptr) == CHAR_DOLLAR_SIGN) goto LOADLITERAL;

      group = -1;
      n = 0;
      inparens = FALSE;
      star = FALSE;

      if (next == CHAR_LEFT_CURLY_BRACKET)
        {
        if (++ptr >= repend) goto BAD;
        next = *ptr;
        inparens = TRUE;
        }

      if (next == CHAR_ASTERISK)
        {
        if (++ptr >= repend) goto BAD;
        next = *ptr;
        star = TRUE;
        }

      if (!star && next >= CHAR_0 && next <= CHAR_9)
        {
        group = next - CHAR_0;
        while (++ptr < repend)
          {
          next = *ptr;
          if (next < CHAR_0 || next > CHAR_9) break;
          group = group * 10 + next - CHAR_0;

          /* A check for a number greater than the hightest captured group
          is sufficient here; no need for a separate overflow check. If unknown
          groups are to be treated as unset, just skip over any remaining
          digits and carry on. */

          if (group > code->top_bracket)
            {
            if ((suboptions & PCRE2_SUBSTITUTE_UNKNOWN_UNSET) != 0)
              {
              while (++ptr < repend && *ptr >= CHAR_0 && *ptr <= CHAR_9);
              break;
              }
            else
              {
              rc = PCRE2_ERROR_NOSUBSTRING;
              goto PTREXIT;
              }
            }
          }
        }
      else
        {
        const uint8_t *ctypes = code->tables + ctypes_offset;
        while (MAX_255(next) && (ctypes[next] & ctype_word) != 0)
          {
          name[n++] = next;
          if (n > 32) goto BAD;
          if (++ptr >= repend) break;
          next = *ptr;
          }
        if (n == 0) goto BAD;
        name[n] = 0;
        }

      /* In extended mode we recognize ${name:+set text:unset text} and
      ${name:-default text}. */

      if (inparens)
        {
        if ((suboptions & PCRE2_SUBSTITUTE_EXTENDED) != 0 &&
             !star && ptr < repend - 2 && next == CHAR_COLON)
          {
          special = *(++ptr);
          if (special != CHAR_PLUS && special != CHAR_MINUS)
            {
            rc = PCRE2_ERROR_BADSUBSTITUTION;
            goto PTREXIT;
            }

          text1_start = ++ptr;
          rc = find_text_end(code, &ptr, repend, special == CHAR_MINUS);
          if (rc != 0) goto PTREXIT;
          text1_end = ptr;

          if (special == CHAR_PLUS && *ptr == CHAR_COLON)
            {
            text2_start = ++ptr;
            rc = find_text_end(code, &ptr, repend, TRUE);
            if (rc != 0) goto PTREXIT;
            text2_end = ptr;
            }
          }

        else
          {
          if (ptr >= repend || *ptr != CHAR_RIGHT_CURLY_BRACKET)
            {
            rc = PCRE2_ERROR_REPMISSINGBRACE;
            goto PTREXIT;
            }
          }

        ptr++;
        }

      /* Have found a syntactically correct group number or name, or *name.
      Only *MARK is currently recognized. */

      if (star)
        {
        if (PRIV(strcmp_c8)(name, STRING_MARK) == 0)
          {
          PCRE2_SPTR mark = pcre2_get_mark(match_data);
          if (mark != NULL)
            {
            PCRE2_SPTR mark_start = mark;
            while (*mark != 0) mark++;
            fraglength = mark - mark_start;
            CHECKMEMCPY(mark_start, fraglength);
            }
          }
        else goto BAD;
        }

      /* Substitute the contents of a group. We don't use substring_copy
      functions any more, in order to support case forcing. */

      else
        {
        PCRE2_SPTR subptr, subptrend;

        /* Find a number for a named group. In case there are duplicate names,
        search for the first one that is set. If the name is not found when
        PCRE2_SUBSTITUTE_UNKNOWN_EMPTY is set, set the group number to a
        non-existent group. */

        if (group < 0)
          {
          PCRE2_SPTR first, last, entry;
          rc = pcre2_substring_nametable_scan(code, name, &first, &last);
          if (rc == PCRE2_ERROR_NOSUBSTRING &&
              (suboptions & PCRE2_SUBSTITUTE_UNKNOWN_UNSET) != 0)
            {
            group = code->top_bracket + 1;
            }
          else
            {
            if (rc < 0) goto PTREXIT;
            for (entry = first; entry <= last; entry += rc)
              {
              uint32_t ng = GET2(entry, 0);
              if (ng < ovector_count)
                {
                if (group < 0) group = ng;          /* First in ovector */
                if (ovector[ng*2] != PCRE2_UNSET)
                  {
                  group = ng;                       /* First that is set */
                  break;
                  }
                }
              }

            /* If group is still negative, it means we did not find a group
            that is in the ovector. Just set the first group. */

            if (group < 0) group = GET2(first, 0);
            }
          }

        /* We now have a group that is identified by number. Find the length of
        the captured string. If a group in a non-special substitution is unset
        when PCRE2_SUBSTITUTE_UNSET_EMPTY is set, substitute nothing. */

        rc = pcre2_substring_length_bynumber(match_data, group, &sublength);
        if (rc < 0)
          {
          if (rc == PCRE2_ERROR_NOSUBSTRING &&
              (suboptions & PCRE2_SUBSTITUTE_UNKNOWN_UNSET) != 0)
            {
            rc = PCRE2_ERROR_UNSET;
            }
          if (rc != PCRE2_ERROR_UNSET) goto PTREXIT;  /* Non-unset errors */
          if (special == 0)                           /* Plain substitution */
            {
            if ((suboptions & PCRE2_SUBSTITUTE_UNSET_EMPTY) != 0) continue;
            goto PTREXIT;                             /* Else error */
            }
          }

        /* If special is '+' we have a 'set' and possibly an 'unset' text,
        both of which are reprocessed when used. If special is '-' we have a
        default text for when the group is unset; it must be reprocessed. */

        if (special != 0)
          {
          if (special == CHAR_MINUS)
            {
            if (rc == 0) goto LITERAL_SUBSTITUTE;
            text2_start = text1_start;
            text2_end = text1_end;
            }

          if (ptrstackptr >= PTR_STACK_SIZE) goto BAD;
          ptrstack[ptrstackptr++] = ptr;
          ptrstack[ptrstackptr++] = repend;

          if (rc == 0)
            {
            ptr = text1_start;
            repend = text1_end;
            }
          else
            {
            ptr = text2_start;
            repend = text2_end;
            }
          continue;
          }

        /* Otherwise we have a literal substitution of a group's contents. */

        LITERAL_SUBSTITUTE:
        subptr = subject + ovector[group*2];
        subptrend = subject + ovector[group*2 + 1];

        /* Substitute a literal string, possibly forcing alphabetic case. */

        while (subptr < subptrend)
          {
          GETCHARINCTEST(ch, subptr);
          if (forcecase != 0)
            {
```

这段代码是用来处理Unicode字符编码的。主要作用是判断当前正在处理的字符是否为Unicode字符，如果是，则根据当前处理方式，从代码中查找相应的处理函数，并对其进行处理。

具体来说，代码分为两部分。第一部分是判断当前字符是否为Unicode字符，如果是，则执行第二部分代码。第二部分代码根据当前处理方式，即从高位开始处理还是从低位开始处理，来决定具体的处理方式。

如果当前处理方式是从高位开始处理，那么先判断当前字符是否为左l附录A类型(即代码中用粗体打印的类型)。如果是，那么就执行代码中用粗体打印的处理函数，并对其进行处理。如果不是从高位开始处理的，那么就需要执行代码中用粗体打印的处理函数，并对其进行处理。

如果当前处理方式是从低位开始处理，那么就需要判断当前字符是否可以原生支持，如果是，就直接返回Unicode编码；如果不是，就需要根据当前处理方式，从代码中查找相应的处理函数，并对其进行处理。在处理函数中，可能会对Unicode编码进行进一步的处理，比如将某些Unicode字符转换为高字节字符，或者将其转换为低字节字符。


```cpp
#ifdef SUPPORT_UNICODE
            if (utf || ucp)
              {
              uint32_t type = UCD_CHARTYPE(ch);
              if (PRIV(ucp_gentype)[type] == ucp_L &&
                  type != ((forcecase > 0)? ucp_Lu : ucp_Ll))
                ch = UCD_OTHERCASE(ch);
              }
            else
#endif
              {
              if (((code->tables + cbits_offset +
                  ((forcecase > 0)? cbit_upper:cbit_lower)
                  )[ch/8] & (1u << (ch%8))) == 0)
                ch = (code->tables + fcc_offset)[ch];
              }
            forcecase = forcecasereset;
            }

```

Based on the given information, it seems that the PCRE2 encoding may not support certain sub options. Specifically, when using PCRE22 on Windows, PCRE2_SUBSTITUTE_EXTENDED sub option is not supported if it contains a non-alphanumeric character. Additionally, when a non-alphanumeric character is present in the sub option string, the case-forcing escape must be recognized manually.

However, if a specific sub option is PCRE2_SUBSTITUTE_EXTENDED and a non-alphanumeric character is present, it's important to keep in mind that this behavior is not officially supported by PCRE22, and it may cause issues with the encoding. As a result, it's recommended to avoid using this sub option when possible.

If a non-alphanumeric character is found in the sub option string, it's crucial to recognize that the case-forcing escape is not supported in PCRE22. To handle this case, you can simply escape the non-alphanumeric character as you would with a backslash. For more information, you may refer to the PCRE2 documentation or use the PCRE22 tool to see how to handle such cases.


```cpp
#ifdef SUPPORT_UNICODE
          if (utf) chlen = PRIV(ord2utf)(ch, temp); else
#endif
            {
            temp[0] = ch;
            chlen = 1;
            }
          CHECKMEMCPY(temp, chlen);
          }
        }
      }

    /* Handle an escape sequence in extended mode. We can use check_escape()
    to process \Q, \E, \c, \o, \x and \ followed by non-alphanumerics, but
    the case-forcing escapes are not supported in pcre2_compile() so must be
    recognized here. */

    else if ((suboptions & PCRE2_SUBSTITUTE_EXTENDED) != 0 &&
              *ptr == CHAR_BACKSLASH)
      {
      int errorcode;

      if (ptr < repend - 1) switch (ptr[1])
        {
        case CHAR_L:
        forcecase = forcecasereset = -1;
        ptr += 2;
        continue;

        case CHAR_l:
        forcecase = -1;
        forcecasereset = 0;
        ptr += 2;
        continue;

        case CHAR_U:
        forcecase = forcecasereset = 1;
        ptr += 2;
        continue;

        case CHAR_u:
        forcecase = 1;
        forcecasereset = 0;
        ptr += 2;
        continue;

        default:
        break;
        }

      ptr++;  /* Point after \ */
      rc = PRIV(check_escape)(&ptr, repend, &ch, &errorcode,
        code->overall_options, code->extra_options, FALSE, NULL);
      if (errorcode != 0) goto BADESCAPE;

      switch(rc)
        {
        case ESC_E:
        forcecase = forcecasereset = 0;
        continue;

        case ESC_Q:
        escaped_literal = TRUE;
        continue;

        case 0:      /* Data character */
        goto LITERAL;

        default:
        goto BADESCAPE;
        }
      }

    /* Handle a literal code unit */

    else
      {
      LOADLITERAL:
      GETCHARINCTEST(ch, ptr);    /* Get character value, increment pointer */

      LITERAL:
      if (forcecase != 0)
        {
```

这段代码是关于支持Unicode字符类型的switch语句。它检查当前正在处理的字符串是否已经使用了Unicode字符类型。如果是，那么它将检查当前正在处理的字符的类型是否为升序排列的Unicode字符类型（即L或l）。如果是，则将当前字符的类型更改为Unicode字符类型。如果不是，那么它将尝试使用已经定义好的Unicode字符类型来处理当前字符。

在代码中，首先使用一个if语句，如果当前正在处理的字符串使用了Unicode字符类型，那么将使用第一个分支语句。否则，将使用第二个分支语句。如果当前正在处理的字符使用了Unicode字符类型，那么将使用第一个分支语句。否则，将使用第二个分支语句。


```cpp
#ifdef SUPPORT_UNICODE
        if (utf || ucp)
          {
          uint32_t type = UCD_CHARTYPE(ch);
          if (PRIV(ucp_gentype)[type] == ucp_L &&
              type != ((forcecase > 0)? ucp_Lu : ucp_Ll))
            ch = UCD_OTHERCASE(ch);
          }
        else
#endif
          {
          if (((code->tables + cbits_offset +
              ((forcecase > 0)? cbit_upper:cbit_lower)
              )[ch/8] & (1u << (ch%8))) == 0)
            ch = (code->tables + fcc_offset)[ch];
          }
        forcecase = forcecasereset;
        }

```

This is a JavaScript function that performs a replacement of a substitution block in PCRE2 format. It takes a pointer to a `PCRE2_SUBSTITUTE_DATA` structure as its argument.

Here is what the function does:

1. It checks if the given substitution block has not been tampered with and if the `mcontext` pointer is not `NULL`.
2. If the block has not been tampered with and the `mcontext` pointer is not `NULL`, it copies the matched substring to the output buffer.
3. If the block has been tampered with or the `mcontext` pointer is `NULL`, it creates a new substitution block and copies the matched substring to it.
4. It creates an array called `buff_offset` to store the offset of the current output buffer.
5. It creates an array called `ovecsave` to store the current output buffer.
6. It sets the first element of the `ovecsave` array to the first element of the `buff_offset` array.
7. It sets the second element of the `ovecsave` array to the current start offset of the substitution block.
8. It sets the `goptions` variable to the contents of the `ivector` array.
9. It sets the output offset to the current start offset.
10. It repeat the "do" loop until it copies the entire substitution block or the block has been tampered with.

This function is useful in cases where you need to replace a PCRE2 substitution block with new content.


```cpp
#ifdef SUPPORT_UNICODE
      if (utf) chlen = PRIV(ord2utf)(ch, temp); else
#endif
        {
        temp[0] = ch;
        chlen = 1;
        }
      CHECKMEMCPY(temp, chlen);
      } /* End handling a literal code unit */
    }   /* End of loop for scanning the replacement. */

  /* The replacement has been copied to the output, or its size has been
  remembered. Do the callout if there is one and we have done an actual
  replacement. */

  if (!overflowed && mcontext != NULL && mcontext->substitute_callout != NULL)
    {
    scb.subscount = subs;
    scb.output_offsets[1] = buff_offset;
    rc = mcontext->substitute_callout(&scb, mcontext->substitute_callout_data);

    /* A non-zero return means cancel this substitution. Instead, copy the
    matched string fragment. */

    if (rc != 0)
      {
      PCRE2_SIZE newlength = scb.output_offsets[1] - scb.output_offsets[0];
      PCRE2_SIZE oldlength = ovector[1] - ovector[0];

      buff_offset -= newlength;
      lengthleft += newlength;
      if (!replacement_only) CHECKMEMCPY(subject + ovector[0], oldlength);

      /* A negative return means do not do any more. */

      if (rc < 0) suboptions &= (~PCRE2_SUBSTITUTE_GLOBAL);
      }
    }

  /* Save the details of this match. See above for how this data is used. If we
  matched an empty string, do the magic for global matches. Update the start
  offset to point to the rest of the subject string. If we re-used an existing
  match for the first match, switch to the internal match data block. */

  ovecsave[0] = ovector[0];
  ovecsave[1] = ovector[1];
  ovecsave[2] = start_offset;

  goptions = (ovector[0] != ovector[1] || ovector[0] > start_offset)? 0 :
    PCRE2_ANCHORED|PCRE2_NOTEMPTY_ATSTART;
  start_offset = ovector[1];
  } while ((suboptions & PCRE2_SUBSTITUTE_GLOBAL) != 0);  /* Repeat "do" loop */

```

这段代码的主要目的是在给定一个字符串（可能是输入数据）时复制其剩余部分，并将其传递给一个名为“fraglength”的变量。然后，它检查一个名为“start_offset”的偏移量是否为零，如果是，则计算出“fraglength”变量实际需要复制的长度。接下来，它使用“CHECKMEMCPY”函数将字符串“subject”中从偏移量“start_offset”开始到“fraglength”长度的字符复制到“fraglength”变量中。

接下来，代码创建一个名为“temp”的数组，并将其赋值为一个包含1个字节（即一个字节中的字符）的数组。然后，它使用另一个名为“temp”的数组（可能是之前复制得到的）进行复制，以确保复制过程正确完成。

最后，代码检查是否已设置“OVERFLOW”标志，如果是，则表示已计算出匹配行中的字符数量。如果是，则表示已经处理完了该部分字符，可以开始输出字符。如果不是，则表示有溢出发生，应该立即返回一个错误码。


```cpp
/* Copy the rest of the subject unless not required, and terminate the output
with a binary zero. */

if (!replacement_only)
  {
  fraglength = length - start_offset;
  CHECKMEMCPY(subject + start_offset, fraglength);
  }

temp[0] = 0;
CHECKMEMCPY(temp, 1);

/* If overflowed is set it means the PCRE2_SUBSTITUTE_OVERFLOW_LENGTH is set,
and matching has carried on after a full buffer, in order to compute the length
needed. Otherwise, an overflow generates an immediate error return. */

```

这段代码是一个if-else语句，用于在不同的溢出示例中执行不同的操作。

如果在输入数据中存在溢出示例，那么代码将执行第一个if语句块，其中会设置一个名为rc的变量并将其设置为PCRE2_ERROR_NOMEMORY，然后将名为blength的变量设置为输入缓冲区的长度加上需要额外分配的内存大小。

如果在输入数据中没有溢出示例，那么代码将执行第二个if语句块，其中会设置一个名为rc的变量并将其设置为substrings的数量，然后将名为blength的变量设置为输入缓冲区的偏移量减1，不包括尾随零。

最后，在if-else语句块之外，还有一条语句，它是一个printf函数，用于将统计信息输出到控制台。


```cpp
if (overflowed)
  {
  rc = PCRE2_ERROR_NOMEMORY;
  *blength = buff_length + extra_needed;
  }

/* After a successful execution, return the number of substitutions and set the
length of buffer used, excluding the trailing zero. */

else
  {
  rc = subs;
  *blength = buff_offset - 1;
  }

```

这段代码是针对PCRE2库中的一种错误处理方式。其作用是当PCRE2在执行匹配操作时遇到无法找到匹配数据的情况下，生成不同的错误代码并跳转到相应的处理函数。以下是代码的作用解释：

1. 如果内部匹配数据不为空，则释放内部匹配数据，并将rc变量赋值为当前匹配数据的返回值。
2. 如果内部匹配数据为空，则设置rc变量为当前匹配数据的返回值，并执行goto EXIT语句跳转到EXIT函数。
3. 如果生成的匹配数据不符合预期，则会执行BADESCAPE错误处理函数，并将rc变量设置为当前匹配数据的返回值，跳转到PTREXIT函数。
4. 如果生成的匹配数据无法访问，则会执行BAD错误处理函数，并将rc变量设置为当前匹配数据的返回值，继续执行匹配操作。


```cpp
EXIT:
if (internal_match_data != NULL) pcre2_match_data_free(internal_match_data);
  else match_data->rc = rc;
return rc;

NOROOM:
rc = PCRE2_ERROR_NOMEMORY;
goto EXIT;

BAD:
rc = PCRE2_ERROR_BADREPLACEMENT;
goto PTREXIT;

BADESCAPE:
rc = PCRE2_ERROR_BADREPESCAPE;

```

这段代码是一个C语言函数，名为“pcre2_substitute”，属于PCRE2库。函数的作用是实现PCRE2库中的函数“pcre2_substitute”，该函数在某些需要替换字符串的情况下非常有用。

具体来说，这段代码定义了一个名为“blength”的变量，并计算出了一个名为“ptr”的指针与一个名为“replacement”的指针之间的差值（即：ptr - replacement）。然后，将计算得到的差值存储到了“blength”变量中。

最后，判断当前所处的位置是否在函数内，如果是，跳转到“EXIT”标签处，如果不是，则继续执行后续的代码。


```cpp
PTREXIT:
*blength = (PCRE2_SIZE)(ptr - replacement);
goto EXIT;
}

/* End of pcre2_substitute.c */

```

# `libpcre/src/pcre2_substring.c`

这段代码是一个Perl-Compatible的正则表达式库，它提供了一些与Perl 5语言的语法和语义尽可能相似的功能。

正则表达式是一种用于描述文本模式的字符集，它可以用来搜索、替换或验证文本。这个库在处理正则表达式时，主要提供了以下功能：

1. 支持Perl 5修饰的正则表达式，包括诸如`@`、`^`、`$`等表示式的支持；
2. 支持C风格的可变参数函数，允许用户使用`?`、`{`、`}`等特殊符号指定参数；
3. 正则表达式的解析算法是使用王二熵算法，并支持Lang bigrams；
4. 对Perl中的负向前瞻序列`^(-1)`做了对应的Java实现，即`@(-1)`；
5. 通过`usePCRE_exceptions`函数实现对PCRE异常的检测和处理。


```cpp
/*************************************************
*      Perl-Compatible Regular Expressions       *
*************************************************/

/* PCRE is a library of functions to support regular expressions whose syntax
and semantics are as close as possible to those of the Perl 5 language.

                       Written by Philip Hazel
     Original API code Copyright (c) 1997-2012 University of Cambridge
          New API code Copyright (c) 2016-2018 University of Cambridge

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

这段代码是一个名为`THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"`的声明。它表明这段软件是版权持有者和贡献者自愿提供给您的，但没有包括任何隐含的保证或警告。它还明确表示，在任何情况下， copyright 持有者或贡献者都不会对任何直接、间接、特殊、示例或后果性的损失承担责任，这些损失可能包括采购 substitute 商品或服务、数据丢失或商业中断。最后，它指出，如果您使用这段软件，无论何种原因，包括但不仅限于疏忽或重大过失， copyright 持有者或贡献者都不会对您承担任何责任。


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

这段代码是一个名为“config_copy_captured_string”的函数，它用于将一个命名捕获的字符串复制到给定的缓冲区中。它通过检查是否存在一个名为“HAVE_CONFIG_H”的配置头来确定是否支持这个名字定义。如果支持，那么它将包含在函数体中。否则，它将包含在函数的外部。

具体来说，这段代码包含以下几行：

1. `#ifdef HAVE_CONFIG_H`：这是一个条件编译语句，只有在存在“HAVE_CONFIG_H”配置头时才会编译。这个配置头可能是一个预定义的名称，或者是一个包含多个头的配置头。

2. `#include "config.h"`：这是引入自“config.h”头文件的行。这个头文件可能包含一些与配置头和给定函数相关的定义。

3. `#include "pcre2_internal.h"`：这是引入自“pcre2_internal.h”头文件的行。这个头文件可能包含与字符串处理函数相关的定义。

4. `/*************************************************`：这是一个注释，说明这个函数可能实现的目的是什么。

5. `int config_copy_captured_string(const char *regex, char *buffer, size_t max_len);`：这是函数的定义，说明它接受一个命名捕获的字符串和一个缓冲区作为参数，并将它们返回。函数实现可能将捕获到的字符串与缓冲区的前缀进行比较，并在找到第一个匹配的子串时将其复制到缓冲区中。如果捕获到的字符串与缓冲区的前缀不同，或者捕获字符串包含多个相同的子串，那么第一个设置的子串将被复制到缓冲区中。

6. `/*************************************************/`：这是另一个注释，说明这个函数可能实现的目的是什么。

7. `void config_print_captured_string(const char *regex, const char *format, const char *捕获_str);`：这是函数的定义，说明它接受一个命名捕获的字符串、一个格式字符串和一个输出字符串作为参数，并将它们用于格式化输出捕获到的字符串。函数实现可能将捕获到的字符串与输出字符串进行格式化，并将格式字符串的值使用PCRE2_PCMDiag格式化。


```cpp
#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include "pcre2_internal.h"



/*************************************************
*   Copy named captured string to given buffer   *
*************************************************/

/* This function copies a single captured substring into a given buffer,
identifying it by name. If the regex permits duplicate names, the first
substring that is set is chosen.

```

这段代码是一个PCRE2库中的函数，用于将给定的正则表达式模式（match_data）中的查找点（stringname）与缓冲区（buffer）中的子串（substring）进行匹配。

函数接收三个参数：匹配数据（match_data，指向匹配数据结构的指针），模式名称（stringname，正则表达式中指定要查找的模式），以及缓冲区（buffer，用于存储子串的位置）。函数返回一个整数：

- 0：表示匹配成功
- 非0：表示匹配失败，可以输出一个错误码，如(1) an error from nametable_scan()，(2) an error from copy_bynumber()，(3) PCRE2_ERROR_UNAVAILABLE: no group is in ovector，(4) PCRE2_ERROR_UNSET: all named groups in ovector are unset。


```cpp
Arguments:
  match_data     points to the match data
  stringname     the name of the required substring
  buffer         where to put the substring
  sizeptr        the size of the buffer, updated to the size of the substring

Returns:         if successful: zero
                 if not successful, a negative error code:
                   (1) an error from nametable_scan()
                   (2) an error from copy_bynumber()
                   (3) PCRE2_ERROR_UNAVAILABLE: no group is in ovector
                   (4) PCRE2_ERROR_UNSET: all named groups in ovector are unset
*/

PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
```

这段代码的作用是实现一个名为 `pcre2_substring_copy_byname` 的函数，它接收一个 `PCRE2_match_data` 结构体，一个 `PCRE2_stringname` 指针和一个 `PCRE2_UCHAR` 类型的缓冲区和一个 `PCRE2_SIZE` 类型的指针数组。

函数的具体实现如下：

1. 首先判断 `match_data` 的 `matchedby` 是否为 `PCRE2_MATCHEDBY_DFA_INTERPRETER`，如果是，则直接返回 `PCRE2_ERROR_DFA_UFUNC`，否则继续执行。

2. 调用 `pcre2_substring_nametable_scan` 函数，将 `match_data` 的 `code` 域和 `stringname` 指针作为参数，返回一个包含字符串名称和匹配子串的数组。如果函数返回负数，则抛出 `PCRE2_ERROR_UNAVAILABLE`。

3. 遍历匹配子串，从 `first` 开始，直到 `last` 结束（注意：`last` 是 `PCRE2_SUSTRING_BACKEND` 函数返回的，可能比 `PCRE2_SUSTRING_COUNT` 函数返回的更小）。在遍历过程中，如果当前字符串长度小于 `match_data` 的 `oveccount`，则执行以下操作：

  a. 从 `entry` 所指向的匹配子串的第二个元素开始，遍历整个字符串，查找是否有匹配的 `PCRE2_UNSET` 标记。

  b. 如果找到了匹配的标记，则说明该标记对应的数值可能存在，可以直接将 `buffer` 中的元素复制到匹配子串中，或者执行后续的 `PCRE2_substring_copy_bynumber` 函数。注意，由于 `PCRE2_SUSTRING_BACKEND` 和 `PCRE2_SUSTRING_COUNT` 的实现方式不同，这里需要根据实际情况进行选择。

4. 如果所有匹配的子串的 `PCRE2_UNSET` 标记都已经被处理完毕，仍然无法继续前驱匹配，则抛出 `PCRE2_ERROR_UNSET`。


```cpp
pcre2_substring_copy_byname(pcre2_match_data *match_data, PCRE2_SPTR stringname,
  PCRE2_UCHAR *buffer, PCRE2_SIZE *sizeptr)
{
PCRE2_SPTR first, last, entry;
int failrc, entrysize;
if (match_data->matchedby == PCRE2_MATCHEDBY_DFA_INTERPRETER)
  return PCRE2_ERROR_DFA_UFUNC;
entrysize = pcre2_substring_nametable_scan(match_data->code, stringname,
  &first, &last);
if (entrysize < 0) return entrysize;
failrc = PCRE2_ERROR_UNAVAILABLE;
for (entry = first; entry <= last; entry += entrysize)
  {
  uint32_t n = GET2(entry, 0);
  if (n < match_data->oveccount)
    {
    if (match_data->ovector[n*2] != PCRE2_UNSET)
      return pcre2_substring_copy_bynumber(match_data, n, buffer, sizeptr);
    failrc = PCRE2_ERROR_UNSET;
    }
  }
```

这段代码是一个函数，名为`copy_captured_string`。它将一个整数匹配数据中的某个子串，并将其复制到给定的缓冲区中。这个子串是通過arguments参数传递给函数的。函数需要两个整数参数：`match_data` 指向要匹配的匹配数据，`stringnumber` 是要复制的子串的编号。函数还需要一个参数 `buffer`，用于放置子串。最后一个参数 `sizeptr` 用於将子串的大小调整到缓冲区的大小。

函数的作用是，当函数被调用时，它会从 `match_data` 缓冲区中读取一个字符，然后将其中的子串复制到 `stringnumber` 缓冲区中，最后将子串复制到给定的 `buffer` 缓冲区中。


```cpp
return failrc;
}



/*************************************************
*  Copy numbered captured string to given buffer *
*************************************************/

/* This function copies a single captured substring into a given buffer,
identifying it by number.

Arguments:
  match_data     points to the match data
  stringnumber   the number of the required substring
  buffer         where to put the substring
  sizeptr        the size of the buffer, updated to the size of the substring

```

这段代码是一个名为"PCRE2_CALL_CONVENTION"的函数，它接受两个参数：匹配数据和目标字符串。它返回一个整数，表示字符串是否成功匹配或者是否发生了错误。

函数内部首先定义了一个名为"buffer"的局部变量，用于存储目标字符串，然后定义了一个名为"sizeptr"的局部变量，用于存储目标字符串的存储空间大小指针。

接着函数使用"pcre2_substring_copy_bynumber"函数将匹配数据中的目标字符串复制到一个名为"buffer"的数组中，同时记录下该字符串在匹配数据中的偏移量和长度。

接下来，函数使用"rc"变量来获取字符串复制过程中发生错误的代码。如果rc小于零，说明发生了错误，代码会尝试通过错误代码进行错误提示。

最后，函数返回一个整数，表示字符串是否成功匹配，如果成功，则返回0，否则返回一个负的错误码。


```cpp
Returns:         if successful: 0
                 if not successful, a negative error code:
                   PCRE2_ERROR_NOMEMORY: buffer too small
                   PCRE2_ERROR_NOSUBSTRING: no such substring
                   PCRE2_ERROR_UNAVAILABLE: ovector too small
                   PCRE2_ERROR_UNSET: substring is not set
*/

PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_substring_copy_bynumber(pcre2_match_data *match_data,
  uint32_t stringnumber, PCRE2_UCHAR *buffer, PCRE2_SIZE *sizeptr)
{
int rc;
PCRE2_SIZE size;
rc = pcre2_substring_length_bynumber(match_data, stringnumber, &size);
```

这段代码是一个 C 语言函数，名为 `extract_named_captured_string`。它实现了一个名为 `extract_string` 的函数，该函数用于提取一个命名捕获字符串（Named Captured String）中的字符串。

具体来说，这段代码的功能如下：

1. 如果rc小于0，函数返回rc。
2. 如果size加上1大于*sizeptr指向的内存区域大小，函数返回PCRE2_ERROR_NOMEMORY。
3. 函数接受一个名为 match_data 的结构体，其中包含一个名为 subject 的字符串和一个名为 ovector 的字符数组。
4. 函数首先计算出 subject 加上 ovector 中第二个字符 *stringnumber，然后从 memory 缓冲区中读取这个字符串，并将其复制到 buffer 中。
5. 函数将 buffer 中字符串的最后一个字符设置为0，并更新 sizeptr 指向的内存区域大小为当前 buffer 的大小。
6. 函数返回0，表示成功提取了命名捕获字符串中的字符串。

这段代码定义的函数接受一个命名捕获字符串，并将其复制到缓冲区中。通过这个函数，用户可以轻松地将命名捕获字符串中的字符串提取出来。


```cpp
if (rc < 0) return rc;
if (size + 1 > *sizeptr) return PCRE2_ERROR_NOMEMORY;
memcpy(buffer, match_data->subject + match_data->ovector[stringnumber*2],
  CU2BYTES(size));
buffer[size] = 0;
*sizeptr = size;
return 0;
}



/*************************************************
*          Extract named captured string         *
*************************************************/

```

这段代码是一个名为“copy_captured_substring”的函数，它从指定名称的输入参数中复制一个匹配的子字符串，并将其复制到新的内存位置。如果正则表达式允许相同的名称，则第一个设置名称的子字符串将被选择。

函数参数包括：

* match_data：指向匹配数据的指针。
* stringname：要复制的字符串的名称。
* stringptr：新内存位置的指针。
* sizeptr：子字符串的长度的指针。

函数返回：

* 成功：零。
* 失败：负值：
	+ 1：错误来自命名表扫描。
	+ 2：错误来自 get_bynumber。
	+ 3：PCRE2_ERROR_UNAVAILABLE：无法使用 ovector。
	+ 4：PCRE2_ERROR_UNSET：所有命名组中未设置的名称的值都是未知的。


```cpp
/* This function copies a single captured substring, identified by name, into
new memory. If the regex permits duplicate names, the first substring that is
set is chosen.

Arguments:
  match_data     pointer to match_data
  stringname     the name of the required substring
  stringptr      where to put the pointer to the new memory
  sizeptr        where to put the length of the substring

Returns:         if successful: zero
                 if not successful, a negative value:
                   (1) an error from nametable_scan()
                   (2) an error from get_bynumber()
                   (3) PCRE2_ERROR_UNAVAILABLE: no group is in ovector
                   (4) PCRE2_ERROR_UNSET: all named groups in ovector are unset
```

这段代码是一个名为`PCRE2_CALL_CONVENTION`的函数，属于PCRE2库。它实现了从PCRE2匹配数据中提取一个字符串，并返回该字符串的起始和结束位置。以下是具体的实现步骤：

1. 从`match_data`中获取字符串名称`stringname`，以及该字符串开始和结束的位置`start`和`end`。
2. 如果`match_data`中已经包含匹配子字符串的索引，则直接返回`PCRE2_SUCCESS`，否则执行以下操作：

3. 从`PCRE2_MATCHEDBY_DFA_INTERPRETER`中确定当前所执行的字符串提取操作类型为DFA（DFA是一种高级语法），则执行以下操作：

4. 从`match_data->code`中获取匹配子字符串的起始位置，在`match_data->ovector`中查找该起始位置，并获取匹配子字符串的长度`length`。

5. 如果`match_data->ovector`中包含该起始位置，则执行以下操作：

6. 通过`pcre2_substring_nametable_scan`函数获取与`stringname`相同的字符串，并从匹配子字符串的起始位置开始，向后查找与`end`位置的字符，如果找到了，则返回该子字符串作为`stringptr`指向的值，否则返回`PCRE2_ERROR_UNSET`并返回`sizeptr`指向的值。
7. 否则，执行以下操作：

8. 遍历`match_data->code`中从`first`到`end`的子字符串，使用`GET2`函数获取匹配子字符串的值，如果该值小于`match_data->oveccount`，则执行以下操作：

9. 如果`match_data->ovector`中包含该值，则返回`PCRE2_SUCCESS`，否则执行以下操作：

10. 如果步骤9成功，则返回`PCRE2_CALL_CONVENTION`，否则执行以下操作：

11. 创建一个`PCRE2_SPTR`类型的变量`first`，用于存储匹配子字符串的起始位置，`last`用于存储匹配子字符串的结束位置，`entry`用于存储当前子字符串的起始位置。

12. 创建一个`PCRE2_SPTR`类型的变量`entrysize`，用于存储匹配子字符串的长度。

13. 执行`PCRE2_CALL_CONVENTION`函数，将`match_data`、`stringname`、`entry`和`entrysize`作为参数传入，该函数将返回匹配子字符串的起始和结束位置，以及`PCRE2_CALL_CONVENTION`的返回码。


```cpp
*/

PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_substring_get_byname(pcre2_match_data *match_data,
  PCRE2_SPTR stringname, PCRE2_UCHAR **stringptr, PCRE2_SIZE *sizeptr)
{
PCRE2_SPTR first, last, entry;
int failrc, entrysize;
if (match_data->matchedby == PCRE2_MATCHEDBY_DFA_INTERPRETER)
  return PCRE2_ERROR_DFA_UFUNC;
entrysize = pcre2_substring_nametable_scan(match_data->code, stringname,
  &first, &last);
if (entrysize < 0) return entrysize;
failrc = PCRE2_ERROR_UNAVAILABLE;
for (entry = first; entry <= last; entry += entrysize)
  {
  uint32_t n = GET2(entry, 0);
  if (n < match_data->oveccount)
    {
    if (match_data->ovector[n*2] != PCRE2_UNSET)
      return pcre2_substring_get_bynumber(match_data, n, stringptr, sizeptr);
    failrc = PCRE2_ERROR_UNSET;
    }
  }
```

这段代码是一个 C 语言函数，名为 "failrc"。函数返回一个整数类型的 failrc，但没有说明这个整数的具体含义。从代码中无法确定它返回的是一个错误码、一个状态信息，还是其他信息，这取决于它被加入到程序中的上下文。

在这段注释中，作者简要说明了这个函数的作用是复制一个匹配数据中的一部分到一个名为 new_memory 的内存区域，并返回该内存区域的大小。然而，具体实现可能会根据上下文的不同而有所不同。


```cpp
return failrc;
}



/*************************************************
*      Extract captured string to new memory     *
*************************************************/

/* This function copies a single captured substring into a piece of new
memory.

Arguments:
  match_data     points to match data
  stringnumber   the number of the required substring
  stringptr      where to put a pointer to the new memory
  sizeptr        where to put the size of the substring

```

这段代码是一个名为"PCRE2_CALL_CONVENTION"的函数，它的作用是获取给定的PCRE2_match_data结构中的一个串，并返回该串的字符数。

具体来说，函数接收两个参数：pcre2_match_data结构和一个表示要获取的字符数的uint32_t类型的变量。函数将这些参数传递给PCRE2_UCHAR类型的变量yield，然后使用rc variable来存储获取到的字符数。

函数接著检查rc variable的值是否为0，如果是，说明函数成功，返回0；如果不是，说明函数失败，返回一个错误代码。如果函数失败，可能的错误代码包括PCRE2_ERROR_NOMEMORY、PCRE2_ERROR_NOSUBSTRING、PCRE2_ERROR_UNAVAILABLE和PCRE2_ERROR_UNSET。


```cpp
Returns:         if successful: 0
                 if not successful, a negative error code:
                   PCRE2_ERROR_NOMEMORY: failed to get memory
                   PCRE2_ERROR_NOSUBSTRING: no such substring
                   PCRE2_ERROR_UNAVAILABLE: ovector too small
                   PCRE2_ERROR_UNSET: substring is not set
*/

PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_substring_get_bynumber(pcre2_match_data *match_data,
  uint32_t stringnumber, PCRE2_UCHAR **stringptr, PCRE2_SIZE *sizeptr)
{
int rc;
PCRE2_SIZE size;
PCRE2_UCHAR *yield;
```

这段代码是一个名为 `pcre2_substring_length_bynumber` 的函数，它接受两个参数：`match_data` 和 `stringnumber`，分别表示匹配数据和匹配模式中包含子字符串的起始位置。它返回一个整数，表示匹配到的子字符串的长度。

函数内部首先定义了一个名为 `rc` 的变量，用于存储匹配到的子字符串的起始位置。然后，通过调用 `pcre2_substring_length_bynumber` 函数，将子字符串的起始位置存储到了 `rc` 中。

接着，函数尝试通过调用 `PRIV(memctl_malloc)` 函数，分配内存并将其存储到了 `yield` 变量中。这个函数的作用是确保 `yield` 变量有足够的内存来存储子字符串的起始位置。

接下来，函数将子字符串的起始位置通过 `memcpy` 函数复制到了 `yield` 变量中。

最后，函数将 `yield` 变量的最后一个元素设置为 0，并更新了 `stringptr` 和 `sizeptr` 两个指向子字符串起始位置和子字符串长度的指针。

函数返回 0，表示成功执行。


```cpp
rc = pcre2_substring_length_bynumber(match_data, stringnumber, &size);
if (rc < 0) return rc;
yield = PRIV(memctl_malloc)(sizeof(pcre2_memctl) +
  (size + 1)*PCRE2_CODE_UNIT_WIDTH, (pcre2_memctl *)match_data);
if (yield == NULL) return PCRE2_ERROR_NOMEMORY;
yield = (PCRE2_UCHAR *)(((char *)yield) + sizeof(pcre2_memctl));
memcpy(yield, match_data->subject + match_data->ovector[stringnumber*2],
  CU2BYTES(size));
yield[size] = 0;
*stringptr = yield;
*sizeptr = size;
return 0;
}



```

这段代码定义了一个名为PCRE2_CALL_CONVENTION的函数，用于释放通过pcre2_substring_get_byxxx()函数获取的子字符串内存。

该函数接收一个PCRE2_UCHAR类型的参数，代表上一次获取子字符串的返回值。函数内部先检查传入的参数是否不为 NULL，如果是，则执行以下操作：

1. 获取子字符串的内存地址，将其存储在pcre2_memctl结构体中。
2. 通过memctl结构体的free()函数，释放内存地址所对应的内存空间。

最后，函数返回void类型，表明没有执行任何操作，需要其他函数来确保释放内存。


```cpp
/*************************************************
*       Free memory obtained by get_substring    *
*************************************************/

/*
Argument:     the result of a previous pcre2_substring_get_byxxx()
Returns:      nothing
*/

PCRE2_EXP_DEFN void PCRE2_CALL_CONVENTION
pcre2_substring_free(PCRE2_UCHAR *string)
{
if (string != NULL)
  {
  pcre2_memctl *memctl = (pcre2_memctl *)((char *)string - sizeof(pcre2_memctl));
  memctl->free(memctl, memctl->memory_data);
  }
}



```

这段代码是一个名为`get_captured_substrings`的函数，它用于获取一个命名子字符串的长度。它接受两个参数：匹配数据和命名的字符串。它返回一个整数表示是否成功，或者一个负的错误码。

函数的实现可能是在一个正则表达式中使用，以确保正则表达式允许指定多个匹配名称。如果允许，那么它将返回第一个设置的名称作为答案。如果不允许，它将抛出 an error。


```cpp
/*************************************************
*         Get length of a named substring        *
*************************************************/

/* This function returns the length of a named captured substring. If the regex
permits duplicate names, the first substring that is set is chosen.

Arguments:
  match_data      pointer to match data
  stringname      the name of the required substring
  sizeptr         where to put the length

Returns:          0 if successful, else a negative error number
*/

```

这段代码是一个名为"PCRE2_CALL_CONVENTION"的函数，它接受两个参数：match_data和stringname，并返回一个名为first的指针，表示匹配到的字符串在代码中的起始位置。

首先，函数检查match_data是否使用了DFA（DFA也可以称为全文算法）接口，如果是，则函数将直接返回PCRE2_ERROR_DFA_UFUNC，表示无法继续执行。如果不是，函数将调用一个名为"pcre2_substring_nametable_scan"的函数，该函数将搜索stringname子串在代码中出现的位置，并返回其结果。如果这个函数返回 negative 值，则表示无法继续执行，函数将返回PCRE2_ERROR_UNAVAILABLE，表示函数执行失败。

接下来，函数会遍历匹配到的字符串的起始位置，每个起始位置计算其长度的字节数，并将这个长度存储到sizeptr指向的内存区域。如果字符串长度小于oveccount，则说明有部分匹配到的字符没有被计算在内，函数会返回这个长度，表示成功处理了字符串。如果所有字符都被正确处理了，函数就不需要返回任何值。

最后，函数会检查输入参数中的first和stringname参数是否正确，如果它们的有效性得到确认，函数就可以正常工作。


```cpp
PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_substring_length_byname(pcre2_match_data *match_data,
  PCRE2_SPTR stringname, PCRE2_SIZE *sizeptr)
{
PCRE2_SPTR first, last, entry;
int failrc, entrysize;
if (match_data->matchedby == PCRE2_MATCHEDBY_DFA_INTERPRETER)
  return PCRE2_ERROR_DFA_UFUNC;
entrysize = pcre2_substring_nametable_scan(match_data->code, stringname,
  &first, &last);
if (entrysize < 0) return entrysize;
failrc = PCRE2_ERROR_UNAVAILABLE;
for (entry = first; entry <= last; entry += entrysize)
  {
  uint32_t n = GET2(entry, 0);
  if (n < match_data->oveccount)
    {
    if (match_data->ovector[n*2] != PCRE2_UNSET)
      return pcre2_substring_length_bynumber(match_data, n, sizeptr);
    failrc = PCRE2_ERROR_UNSET;
    }
  }
```



这段代码是一个 C 语言函数，名为 `get_substring_length`，它的作用是获取一个数字子字符串的长度，如果子字符串的起始位置超出了结束位置，函数将返回零长度。

函数的参数包括三个参数：

- `match_data` 是一个指向匹配数据的指针。
- `stringnumber` 是数字子字符串的数量。
- `sizeptr` 是用于存放子字符串长度的指针，如果该指针为 NULL，则不会使用该参数。

函数内部首先检查 `sizeptr` 是否为 NULL，如果是，则表示没有提供子字符串的长度信息，无法计算长度，函数将返回。如果 `sizeptr` 不是 NULL，则使用 `stringnumber` 获取子字符串的长度，并将其存储到 `sizeptr` 指向的内存位置。最后，函数返回子字符串的长度。


```cpp
return failrc;
}



/*************************************************
*        Get length of a numbered substring      *
*************************************************/

/* This function returns the length of a captured substring. If the start is
beyond the end (which can happen when \K is used in an assertion), it sets the
length to zero.

Arguments:
  match_data      pointer to match data
  stringnumber    the number of the required substring
  sizeptr         where to put the length, if not NULL

```

该函数的作用是检查给定的PCRE2匹配数据中包含子串的子串长度的有效性，并返回相应的错误码。

具体来说，函数接收两个参数：匹配数据和子串编号。函数首先检查给定的子串在匹配数据中是否存在，如果不存在，则返回PCRE2_ERROR_NOSUPPORT。如果存在子串，函数将计算子串在匹配数据中的长度，并将其存储到计数器中。最后，函数将计数器的值返回，如果子串包含整个匹配数据，则返回0。

如果给定的子串编号为0，则表示整个匹配数据包含该子串，这种情况下函数将返回PCRE2_ERROR_PARTIAL。如果给定的子串编号为负数，则表示该子串不存在于匹配数据中，这种情况下函数将返回PCRE2_ERROR_UNAVAILABLE。


```cpp
Returns:         if successful: 0
                 if not successful, a negative error code:
                   PCRE2_ERROR_NOSUBSTRING: no such substring
                   PCRE2_ERROR_UNAVAILABLE: ovector is too small
                   PCRE2_ERROR_UNSET: substring is not set
*/

PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_substring_length_bynumber(pcre2_match_data *match_data,
  uint32_t stringnumber, PCRE2_SIZE *sizeptr)
{
PCRE2_SIZE left, right;
int count = match_data->rc;
if (count == PCRE2_ERROR_PARTIAL)
  {
  if (stringnumber > 0) return PCRE2_ERROR_PARTIAL;
  count = 0;
  }
```

这段代码是一个if-else语句，判断是否能够成功匹配一个PCRE2_MATCHEDBY_DFA_INTERPROPRI 不能不说非常重大。如果匹配失败，将返回count，否则将继续判断。

if语句的第一部分匹配条件为匹配数据结构中type是否为DFA。如果是，程序将执行if语句的第二部分。在if语句的第二部分，程序将检查给定的字符串是否匹配给定的锣计数器中的前括号。如果是，程序将返回PCRE2_ERROR_NOSUBSTRING错误码。

如果字符串数组长度大于匹配计数器中的字符数组长度，程序将返回PCRE2_ERROR_UNAVAILABLE错误码。

如果匹配计数器中当前字符串与给定字符串的匹配度小于0，程序将返回count，并输出count作为错误码。

否则，如果匹配成功，程序将继续执行if语句的第一部分，并检查给定的字符串是否在给定的锣计数器中。


```cpp
else if (count < 0) return count;            /* Match failed */

if (match_data->matchedby != PCRE2_MATCHEDBY_DFA_INTERPRETER)
  {
  if (stringnumber > match_data->code->top_bracket)
    return PCRE2_ERROR_NOSUBSTRING;
  if (stringnumber >= match_data->oveccount)
    return PCRE2_ERROR_UNAVAILABLE;
  if (match_data->ovector[stringnumber*2] == PCRE2_UNSET)
    return PCRE2_ERROR_UNSET;
  }
else  /* Matched using pcre2_dfa_match() */
  {
  if (stringnumber >= match_data->oveccount) return PCRE2_ERROR_UNAVAILABLE;
  if (count != 0 && stringnumber >= (uint32_t)count) return PCRE2_ERROR_UNSET;
  }

```

这段代码的作用是获取一个匹配数据结构中的所有捕捉到的字符串，并将它们存储在一个字符数组中。

首先，它定义了两个变量left和right，它们都指向一个int类型的变量match_data->ovector，即匹配数据结构中的一个ovector数组。然后，它比较了这两个变量的大小，如果左边的值大于右边的值，那么将0赋值给*sizeptr，否则将右边的值减去左边的值并将其赋值给*sizeptr。最后，它返回0。

具体来说，这段代码的作用是获取一个字符数组匹配数据结构中的所有捕捉到的字符串，并将它们存储在一个字符数组中。如果捕捉到的字符串有一个结束标志'\0'，则该数组将包含从匹配数据结构中第2个元素到字符串结尾的所有字符。如果捕捉到的字符串不包括结束标志'\0'，则该数组将包含从匹配数据结构中第2个元素到字符串结尾的所有字符，包括结束标志'\0'。


```cpp
left = match_data->ovector[stringnumber*2];
right = match_data->ovector[stringnumber*2+1];
if (sizeptr != NULL) *sizeptr = (left > right)? 0 : right - left;
return 0;
}



/*************************************************
*    Extract all captured strings to new memory  *
*************************************************/

/* This function gets one chunk of memory and builds a list of pointers and all
the captured substrings in it. A NULL pointer is put on the end of the list.
The substrings are zero-terminated, but also, if the final argument is
```

这段代码的作用是实现一个名为`PCRE2_CALL_CONVENTION`的函数，它接受两个参数：`match_data`表示匹配数据，`listptr`表示指向匹配数据中 pointer 的指针，`lengthsptr`表示指向匹配数据中 length 的指针（可能为 NULL）。

函数实现了一个简单的二进制数据匹配功能，当`match_data`中的数据与`listptr`中的指针匹配时，函数返回 0；否则，函数返回一个负的错误码。


```cpp
non-NULL, a list of lengths is also returned. This allows binary data to be
handled.

Arguments:
  match_data     points to the match data
  listptr        set to point to the list of pointers
  lengthsptr     set to point to the list of lengths (may be NULL)

Returns:         if successful: 0
                 if not successful, a negative error code:
                   PCRE2_ERROR_NOMEMORY: failed to get memory,
                   or a match failure code
*/

PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
```

这段代码的作用是获取PCRE2_SUBSTRING_LIST_CONTAINS函数返回的匹配数据中，所有包含目标字符串的子串数量。并将结果存储在PCRE2_UCHAR数组中。

具体来说，代码首先判断输入的匹配数据是否成功，如果失败，则返回0。接着判断输入的ovec数是否足够大，如果不够，则将输入的ovec数乘以2并加1，以便正确处理最长公共子序列的情况。

接着，代码定义了一个PCRE2_SIZE类型的变量size，用于存储匹配数据中字符串的长度信息。然后定义了一个PCRE2_SIZE类型的变量lensp，用于存储每个子串的起始和长度信息。接下来，定义了一个PCRE2_UCHAR类型的变量listp，用于存储匹配数据中包含目标字符串的子串。最后，定义了一个PCRE2_UCHAR类型的变量sp，用于存储目标字符串本身。

在if语句的内部，首先计算了输入的count大小，用于后续的遍历。接着计算了最长公共子序列的长度，然后遍历所有包含目标字符串的子串，并将它们存储到listp中。最后，返回包含目标字符串的子串数量。


```cpp
pcre2_substring_list_get(pcre2_match_data *match_data, PCRE2_UCHAR ***listptr,
  PCRE2_SIZE **lengthsptr)
{
int i, count, count2;
PCRE2_SIZE size;
PCRE2_SIZE *lensp;
pcre2_memctl *memp;
PCRE2_UCHAR **listp;
PCRE2_UCHAR *sp;
PCRE2_SIZE *ovector;

if ((count = match_data->rc) < 0) return count;   /* Match failed */
if (count == 0) count = match_data->oveccount;    /* Ovector too small */

count2 = 2*count;
```

这段代码的主要目的是在给定一个`PCRE2_Ovector`类型的变量`ovector`和一个指向`PCRE2_MemCTL`结构物的指针`match_data`的基础上，对数据进行处理，将其存储在一个动态内存分配的数组中，并返回一个指向该数组的指针。

具体来说，代码首先将`size`的值设置为`sizeof(pcre2_memctl)`和`sizeof(PCRE2_UCHAR *)"，因为这两个结构物的成员数相同，都是3个字节。然后，代码将`lengthsptr`的值初始化为`match_data->lengthsptr`，因为如果没有这个指针，则意味着数据没有给出长度信息，需要计算出它的长度。

接下来，代码使用嵌套的循环遍历数据，每次将`PCRE2_UCHAR *`类型的内存中的一个元素增加`sizeof(PCRE2_UCHAR *)`个字节，然后将当前元素与另一个元素比较，如果当前元素大于另一个元素，则将前一个元素加上两个元素的大小。这样，就可以在每次遍历过程中，将当前元素与其左右邻居元素的长度信息加入动态内存分配的数组中。

最后，代码使用`PRIV(memctl_malloc)`函数为新分配的内存分配内存，并返回一个指向该内存数组的指针。如果分配失败，则返回一个状态码。


```cpp
ovector = match_data->ovector;
size = sizeof(pcre2_memctl) + sizeof(PCRE2_UCHAR *);      /* For final NULL */
if (lengthsptr != NULL) size += sizeof(PCRE2_SIZE) * count;  /* For lengths */

for (i = 0; i < count2; i += 2)
  {
  size += sizeof(PCRE2_UCHAR *) + CU2BYTES(1);
  if (ovector[i+1] > ovector[i]) size += CU2BYTES(ovector[i+1] - ovector[i]);
  }

memp = PRIV(memctl_malloc)(size, (pcre2_memctl *)match_data);
if (memp == NULL) return PCRE2_ERROR_NOMEMORY;

*listptr = listp = (PCRE2_UCHAR **)((char *)memp + sizeof(pcre2_memctl));
lensp = (PCRE2_SIZE *)((char *)listp + sizeof(PCRE2_UCHAR *) * (count + 1));

```

这段代码的作用是检查给定的字符串是否以 "PCRE2_SIZE" 为结束标记，如果是，则分析该标记之前的字符，否则将其复制到标头为 "sp" 的位置，同时更新标头为 "lensp" 的位置。

具体而言，代码首先检查给定的字符串是否为空字符串，如果是，则将标头为 "lensp" 的位置初始化为NULL，将标头为 "lengthsptr" 的位置指向当前字符串的结尾位置。

如果字符串不是空字符串，则将标头为 "lensp" 的位置指向该字符串的起始位置，并将标头为 "lengthsptr" 的位置指向该字符串的结尾位置。接着，代码遍历给定的字符串，计算出每个字符 "ovector[i]" 减去前一个字符 "ovector[i-1]" 的大小，如果该字符大于0，则将其复制到标头为 "sp" 的位置，同时将 "size" 设置为该字符的大小。如果字符小于0，则将其忽略，防止在匹配数据中产生溢出。最后，代码将标头为 "sp" 和 "lensp" 的位置进行指向，以便在遍历过程中更新它们的位置。


```cpp
if (lengthsptr == NULL)
  {
  sp = (PCRE2_UCHAR *)lensp;
  lensp = NULL;
  }
else
  {
  *lengthsptr = lensp;
  sp = (PCRE2_UCHAR *)((char *)lensp + sizeof(PCRE2_SIZE) * count);
  }

for (i = 0; i < count2; i += 2)
  {
  size = (ovector[i+1] > ovector[i])? (ovector[i+1] - ovector[i]) : 0;

  /* Size == 0 includes the case when the capture is unset. Avoid adding
  PCRE2_UNSET to match_data->subject because it overflows, even though with
  zero size calling memcpy() is harmless. */

  if (size != 0) memcpy(sp, match_data->subject + ovector[i], CU2BYTES(size));
  *listp++ = sp;
  if (lensp != NULL) *lensp++ = size;
  sp += size;
  *sp++ = 0;
  }

```

这段代码定义了一个名为`listp`的指针变量，并将其初始化为`NULL`。然后，它返回了一个整数`0`。

接下来是一段注释，说明代码的功能。

然后，代码实现了一个名为`substring_list_get`的函数，但并没有为其定义参数。因此，这个函数的使用者需要自己定义好函数的参数。

由于没有具体的函数实现，所以无法确定这个函数是否合法，以及它会发生什么行为。


```cpp
*listp = NULL;
return 0;
}



/*************************************************
*   Free memory obtained by substring_list_get   *
*************************************************/

/*
Argument:     the result of a previous pcre2_substring_list_get()
Returns:      nothing
*/

```

这段代码是用于处理PCRE2函数中的参数列表，属于PCRE2_EXP_DEFN类型。

具体来说，这段代码定义了一个名为"PCRE2_CALL_CONVENTION"的函数，该函数接受一个字符指针列表参数。如果该列表参数不为空，则代表列表中存在匹配给定名称的PCRE2_SPTR类型的数据，函数内部通过一个名为"memctl"的指针访问列表的第一个元素，然后使用该指针的"free"函数释放内存，从而释放列表中的所有数据。

代码的目的是在PCRE2函数中处理一个字符指针列表，通过释放内存来释放列表中的所有数据，以避免内存泄漏。


```cpp
PCRE2_EXP_DEFN void PCRE2_CALL_CONVENTION
pcre2_substring_list_free(PCRE2_SPTR *list)
{
if (list != NULL)
  {
  pcre2_memctl *memctl = (pcre2_memctl *)((char *)list - sizeof(pcre2_memctl));
  memctl->free(memctl, memctl->memory_data);
  }
}



/*************************************************
*     Find (multiple) entries for named string   *
*************************************************/

```

这段代码定义了一个名为 `scrape_nametable` 的函数，它用于扫描给定名称的命名表，使用二进制剪枝。函数接受三个参数：

1. `code`：要扫描的命名表中的一行字符串，表示成 regular expression 形式；
2. `stringname`：要扫描的名称，作为 `scrape_nametable` 函数内部使用的一行字符串；
3. `firstptr` 和 `lastptr`：指向命名表中第一行和最后一行位置的指针，如果不需要这两个指针，函数将返回一个数字，表示给定名称的独一无二子串的数量；如果存在重复名称，函数将返回一个非负整数，表示错误。

函数的作用是：

1. 如果给定的名称在命名表中找不到，函数将返回 PCRE2_ERROR_NOSUBSTRING，否则；
2. 如果 `firstptr` 和 `lastptr` 都为空，函数将返回一个独特的子串编号；
3. 如果给定的名称在命名表中存在，函数将返回包含每个条目的长度的一组指针，或者只返回一组长度，表示给定名称的独一无二子串的数量。


```cpp
/* This function scans the nametable for a given name, using binary chop. It
returns either two pointers to the entries in the table, or, if no pointers are
given, the number of a unique group with the given name. If duplicate names are
permitted, and the name is not unique, an error is generated.

Arguments:
  code        the compiled regex
  stringname  the name whose entries required
  firstptr    where to put the pointer to the first entry
  lastptr     where to put the pointer to the last entry

Returns:      PCRE2_ERROR_NOSUBSTRING if the name is not found
              otherwise, if firstptr and lastptr are NULL:
                a group number for a unique substring
                else PCRE2_ERROR_NOUNIQUESUBSTRING
              otherwise:
                the length of each entry, having set firstptr and lastptr
```

This is a function definition for `pcre2_substring_nametable_scan` that takes a pointer to a PCRE2 code, a name table, and pointers to the first and last pointers to a PCRE2 substring that matches the name table.

It first initializes the variable `bot` to 0 and `top` to the number of characters in the name table.

Then it sets the variable `entrysize` to the number of bytes in each substring in the name table.

Next, it sets the variable `nametable` to the memory location of the name table in the PCRE2 code.

It then enters a while loop that iterates through the name table.

For each substring, it compares the substring to the name table using `strcmp` and sets the `first` and `last` pointers to the first and last matching characters, respectively.

Then, in a second while loop, it checks if the substring starts from the middle of the substring. If it does, it sets the `bot` and `top` variables to the midpoint and updates the `first` and `last` pointers to the appropriate characters.

If the substring does not match, the function returns an error code and sets the `first` and `last` pointers to the last and first matching characters, respectively.

If the substring starts from the middle of the substring, the function returns the length of the substring and updates the `first` and `last` pointers to the appropriate characters.

Finally, the function returns 0 on success and the error code on failure.


```cpp
*/

PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_substring_nametable_scan(const pcre2_code *code, PCRE2_SPTR stringname,
  PCRE2_SPTR *firstptr, PCRE2_SPTR *lastptr)
{
uint16_t bot = 0;
uint16_t top = code->name_count;
uint16_t entrysize = code->name_entry_size;
PCRE2_SPTR nametable = (PCRE2_SPTR)((char *)code + sizeof(pcre2_real_code));

while (top > bot)
  {
  uint16_t mid = (top + bot) / 2;
  PCRE2_SPTR entry = nametable + entrysize*mid;
  int c = PRIV(strcmp)(stringname, entry + IMM2_SIZE);
  if (c == 0)
    {
    PCRE2_SPTR first;
    PCRE2_SPTR last;
    PCRE2_SPTR lastentry;
    lastentry = nametable + entrysize * (code->name_count - 1);
    first = last = entry;
    while (first > nametable)
      {
      if (PRIV(strcmp)(stringname, (first - entrysize + IMM2_SIZE)) != 0) break;
      first -= entrysize;
      }
    while (last < lastentry)
      {
      if (PRIV(strcmp)(stringname, (last + entrysize + IMM2_SIZE)) != 0) break;
      last += entrysize;
      }
    if (firstptr == NULL) return (first == last)?
      (int)GET2(entry, 0) : PCRE2_ERROR_NOUNIQUESUBSTRING;
    *firstptr = first;
    *lastptr = last;
    return entrysize;
    }
  if (c > 0) bot = mid + 1; else top = mid;
  }

```

这段代码是一个名为 `find_number_for_named_string` 的函数，它的作用是返回一个名为 `PCRE2_ERROR_NOSUBSTRING` 的错误码。这个函数的作用是方便地调用 `pcre2_substring_nametable_scan()` 函数，当已知命名字符串时，这个函数会返回一个唯一的数字，否则返回 `PCRE2_ERROR_NOSUBSTRING`。


```cpp
return PCRE2_ERROR_NOSUBSTRING;
}


/*************************************************
*           Find number for named string         *
*************************************************/

/* This function is a convenience wrapper for pcre2_substring_nametable_scan()
when it is known that names are unique. If there are duplicate names, it is not
defined which number is returned.

Arguments:
  code        the compiled regex
  stringname  the name whose number is required

```

这段代码定义了一个名为 `PCRE2_CALL_CONVENTION` 的函数，它接受一个名为 `code` 的 `pcre2_code` 类型的参数，和一个名为 `stringname` 的 `pcre2_sptr` 类型的参数。

函数的作用是返回一个整数，表示给定的 `code` 中的命名父字符串的数量。如果给定的 `code` 和 `stringname` 都不存在，函数将返回 `PCRE2_ERROR_NOSUBSTRING`，否则将返回 `PCRE2_ERROR_NOUNIQUESSUBSTRING`。

这里使用了 `pcre2_substring_nametable_scan` 函数来实现字符串的扫描，该函数将给定的 `code` 中的字符串名映射到一个包含所有给定字符串的子字符串数组的指针上，该指针作为第二个参数传递给 `pcre2_substring_nametable_scan` 函数，该函数将该子字符串数组的起始和长度作为第一个和第二个参数返回。


```cpp
Returns:      the number of the named parenthesis, or a negative number
                PCRE2_ERROR_NOSUBSTRING if not found
                PCRE2_ERROR_NOUNIQUESUBSTRING if not unique
*/

PCRE2_EXP_DEFN int PCRE2_CALL_CONVENTION
pcre2_substring_number_from_name(const pcre2_code *code,
  PCRE2_SPTR stringname)
{
return pcre2_substring_nametable_scan(code, stringname, NULL, NULL);
}

/* End of pcre2_substring.c */

```