# `nmap\liblua\lctype.h`

```
# 定义了一个头文件保护宏，避免重复包含
#ifndef lctype_h
#define lctype_h

# 包含 lua.h 头文件
#include "lua.h"

# 如果没有定义 LUA_USE_CTYPE
#if !defined(LUA_USE_CTYPE)

# 如果 'A' 的 ASCII 码是 65，'0' 的 ASCII 码是 48
# 则可以使用自己的表格，更快且固定
#define LUA_USE_CTYPE    0
# 否则必须使用标准的 C ctype
#else
#define LUA_USE_CTYPE    1
#endif

# 如果不使用标准的 C ctype
#if !LUA_USE_CTYPE    /* { */

# 包含 limits.h 和 llimits.h 头文件
#include <limits.h>
#include "llimits.h"

# 定义一些位掩码
#define ALPHABIT    0
#define DIGITBIT    1
#define PRINTBIT    2
#define SPACEBIT    3
#define XDIGITBIT    4

# 定义一个宏，用于生成位掩码
#define MASK(B)        (1 << (B))

# 将字符加 1，以允许索引 -1 (EOZ)
#define testprop(c,p)    (luai_ctype_[(c)+1] & (p))

# 定义一些宏，用于判断字符的属性
#define lislalpha(c)    testprop(c, MASK(ALPHABIT))
#define lislalnum(c)    testprop(c, (MASK(ALPHABIT) | MASK(DIGITBIT)))
#define lisdigit(c)    testprop(c, MASK(DIGITBIT))
#define lisspace(c)    testprop(c, MASK(SPACEBIT))
#define lisprint(c)    testprop(c, MASK(PRINTBIT))
#define lisxdigit(c)    testprop(c, MASK(XDIGITBIT))

# 定义一个宏，用于将字符转换为小写
#define ltolower(c)  \
  check_exp(('A' <= (c) && (c) <= 'Z') || (c) == ((c) | ('A' ^ 'a')),  \
            (c) | ('A' ^ 'a'))

# 为每个字符和 -1 (EOZ) 定义了一个 luai_ctype_ 数组
LUAI_DDEC(const lu_byte luai_ctype_[UCHAR_MAX + 2];)

# 否则使用标准的 C ctype
#else            /* }{ */

# 包含 ctype.h 头文件
#include <ctype.h>

# 定义一些宏，用于判断字符的属性
#define lislalpha(c)    (isalpha(c) || (c) == '_')
#define lislalnum(c)    (isalnum(c) || (c) == '_')
# 定义宏函数，用于判断字符是否为数字
#define lisdigit(c)    (isdigit(c))
# 定义宏函数，用于判断字符是否为空白字符
#define lisspace(c)    (isspace(c))
# 定义宏函数，用于判断字符是否可打印
#define lisprint(c)    (isprint(c))
# 定义宏函数，用于判断字符是否为十六进制数字
#define lisxdigit(c)    (isxdigit(c))

# 定义宏函数，将字符转换为小写
#define ltolower(c)    (tolower(c))

# 结束条件编译指令，结束对应的条件编译块
#endif            /* } */

# 结束条件编译指令，结束对应的条件编译块
#endif
```