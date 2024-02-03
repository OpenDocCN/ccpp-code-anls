# `nmap\liblua\luaconf.h`

```cpp
/*
** $Id: luaconf.h $
** Configuration file for Lua
** See Copyright Notice in lua.h
*/


#ifndef luaconf_h
#define luaconf_h

#include <limits.h>
#include <stddef.h>


/*
** ===================================================================
** General Configuration File for Lua
**
** Some definitions here can be changed externally, through the compiler
** (e.g., with '-D' options): They are commented out or protected
** by '#if !defined' guards. However, several other definitions
** should be changed directly here, either because they affect the
** Lua ABI (by making the changes here, you ensure that all software
** connected to Lua, such as C libraries, will be compiled with the same
** configuration); or because they are seldom changed.
**
** Search for "@@" to find all configurable definitions.
** ===================================================================
*/


/*
** {====================================================================
** System Configuration: macros to adapt (if needed) Lua to some
** particular platform, for instance restricting it to C89.
** =====================================================================
*/

/*
@@ LUA_USE_C89 controls the use of non-ISO-C89 features.
** Define it if you want Lua to avoid the use of a few C99 features
** or Windows-specific features on Windows.
*/
/* #define LUA_USE_C89 */


/*
** By default, Lua on Windows use (some) specific Windows features
*/
#if !defined(LUA_USE_C89) && defined(_WIN32) && !defined(_WIN32_WCE)
#define LUA_USE_WINDOWS  /* enable goodies for regular Windows */
#endif


#if defined(LUA_USE_WINDOWS)
#define LUA_DL_DLL    /* enable support for DLL */
#define LUA_USE_C89    /* broadly, Windows is C89 */
#endif


#if defined(LUA_USE_LINUX)
#define LUA_USE_POSIX
#define LUA_USE_DLOPEN        /* needs an extra library: -ldl */
#endif


#if defined(LUA_USE_MACOSX)
#define LUA_USE_POSIX
#define LUA_USE_DLOPEN        /* MacOS does not need -ldl */
#endif
/*
** LUAI_IS32INT is true iff 'int' has (at least) 32 bits.
*/
#define LUAI_IS32INT    ((UINT_MAX >> 30) >= 3)

/* }================================================================== */



/*
** {==================================================================
** Configuration for Number types. These options should not be
** set externally, because any other code connected to Lua must
** use the same configuration.
** ===================================================================
*/

/*
@@ LUA_INT_TYPE defines the type for Lua integers.
@@ LUA_FLOAT_TYPE defines the type for Lua floats.
** Lua should work fine with any mix of these options supported
** by your C compiler. The usual configurations are 64-bit integers
** and 'double' (the default), 32-bit integers and 'float' (for
** restricted platforms), and 'long'/'double' (for C compilers not
** compliant with C99, which may not have support for 'long long').
*/

/* predefined options for LUA_INT_TYPE */
#define LUA_INT_INT        1
#define LUA_INT_LONG        2
#define LUA_INT_LONGLONG    3

/* predefined options for LUA_FLOAT_TYPE */
#define LUA_FLOAT_FLOAT        1
#define LUA_FLOAT_DOUBLE    2
#define LUA_FLOAT_LONGDOUBLE    3


/* Default configuration ('long long' and 'double', for 64-bit Lua) */
#define LUA_INT_DEFAULT        LUA_INT_LONGLONG
#define LUA_FLOAT_DEFAULT    LUA_FLOAT_DOUBLE


/*
@@ LUA_32BITS enables Lua with 32-bit integers and 32-bit floats.
*/
#define LUA_32BITS    0


/*
@@ LUA_C89_NUMBERS ensures that Lua uses the largest types available for
** C89 ('long' and 'double'); Windows always has '__int64', so it does
** not need to use this case.
*/
#if defined(LUA_USE_C89) && !defined(LUA_USE_WINDOWS)
#define LUA_C89_NUMBERS        1
#else
#define LUA_C89_NUMBERS        0
#endif


#if LUA_32BITS        /* { */
/*
** 32-bit integers and 'float'
*/
#if LUAI_IS32INT  /* use 'int' if big enough */
#define LUA_INT_TYPE    LUA_INT_INT
#else  /* otherwise use 'long' */
/*
** Configuration for Paths.
** ===================================================================
*/

/*
** LUA_PATH_SEP is the character that separates templates in a path.
** LUA_PATH_MARK is the string that marks the substitution points in a
** template.
** LUA_EXEC_DIR in a Windows path is replaced by the executable's
** directory.
*/
#define LUA_PATH_SEP            ";"  // 定义路径中模板之间的分隔符
#define LUA_PATH_MARK           "?"  // 定义模板中的替换点标记
#define LUA_EXEC_DIR            "!"  // 在 Windows 路径中，被可执行文件的目录替换


/*
@@ LUA_PATH_DEFAULT is the default path that Lua uses to look for
** Lua libraries.
@@ LUA_CPATH_DEFAULT is the default path that Lua uses to look for
** C libraries.
** CHANGE them if your machine has a non-conventional directory
** hierarchy or if you want to install your libraries in
** non-conventional directories.
*/

#define LUA_VDIR    LUA_VERSION_MAJOR "." LUA_VERSION_MINOR  // 定义 Lua 版本目录
#if defined(_WIN32)    /* { */
/*
** In Windows, any exclamation mark ('!') in the path is replaced by the
** path of the directory of the executable file of the current process.
*/
#define LUA_LDIR    "!\\lua\\"  // 在 Windows 中，路径中的感叹号 ('!') 被当前进程的可执行文件目录替换
#define LUA_CDIR    "!\\"  // 在 Windows 中，路径中的感叹号 ('!') 被当前进程的可执行文件目录替换
#define LUA_SHRDIR    "!\\..\\share\\lua\\" LUA_VDIR "\\"  // 在 Windows 中，路径中的感叹号 ('!') 被当前进程的可执行文件目录替换

#if !defined(LUA_PATH_DEFAULT)
#define LUA_PATH_DEFAULT  \
        LUA_LDIR"?.lua;"  LUA_LDIR"?\\init.lua;" \
        LUA_CDIR"?.lua;"  LUA_CDIR"?\\init.lua;" \
        LUA_SHRDIR"?.lua;" LUA_SHRDIR"?\\init.lua;" \
        ".\\?.lua;" ".\\?\\init.lua"  // 如果未定义默认路径，则设置默认路径
#endif
#if !defined(LUA_CPATH_DEFAULT)
#define LUA_CPATH_DEFAULT \
        LUA_CDIR"?.dll;" \
        LUA_CDIR"..\\lib\\lua\\" LUA_VDIR "\\?.dll;" \
        LUA_CDIR"loadall.dll;" ".\\?.dll"
#endif

这段代码定义了一个名为LUA_CPATH_DEFAULT的宏，用于设置默认的Lua模块搜索路径。


#else            /* }{ */

#define LUA_ROOT    "/usr/local/"
#define LUA_LDIR    LUA_ROOT "share/lua/" LUA_VDIR "/"
#define LUA_CDIR    LUA_ROOT "lib/lua/" LUA_VDIR "/"

#if !defined(LUA_PATH_DEFAULT)
#define LUA_PATH_DEFAULT  \
        LUA_LDIR"?.lua;"  LUA_LDIR"?/init.lua;" \
        LUA_CDIR"?.lua;"  LUA_CDIR"?/init.lua;" \
        "./?.lua;" "./?/init.lua"
#endif

这段代码定义了LUA_ROOT、LUA_LDIR和LUA_CDIR这几个宏，用于设置Lua的根目录和相关路径。同时也定义了LUA_PATH_DEFAULT宏，用于设置默认的Lua模块搜索路径。


#if !defined(LUA_CPATH_DEFAULT)
#define LUA_CPATH_DEFAULT \
        LUA_CDIR"?.so;" LUA_CDIR"loadall.so;" "./?.so"
#endif

这段代码定义了LUA_CPATH_DEFAULT宏，用于设置默认的Lua模块搜索路径。


#if !defined(LUA_DIRSEP)

#if defined(_WIN32)
#define LUA_DIRSEP    "\\"
#else
#define LUA_DIRSEP    "/"
#endif

#endif

这段代码定义了LUA_DIRSEP宏，用于表示目录分隔符。


#if defined(LUA_BUILD_AS_DLL)    /* { */

#if defined(LUA_CORE) || defined(LUA_LIB)    /* { */
#define LUA_API __declspec(dllexport)
#else                        /* }{ */
#define LUA_API __declspec(dllimport)
#endif                        /* } */

这段代码定义了LUA_API宏，用于标记所有核心API函数。根据LUA_BUILD_AS_DLL的定义，决定是导出函数还是导入函数。
/*
** 如果不是 C++ 编译器，定义 LUA_API 为 extern
*/
#else                /* }{ */

#define LUA_API        extern

#endif                /* } */


/*
** 大多数情况下，库和核心一起使用
*/
#define LUALIB_API    LUA_API
#define LUAMOD_API    LUA_API


/*
@@ LUAI_FUNC 是一个标记，用于所有不导出到外部模块的外部函数
@@ LUAI_DDEF 和 LUAI_DDEC 是标记，用于所有不导出到外部模块的外部（常量）变量
** （LUAI_DDEF 用于定义，LUAI_DDEC 用于声明）
** 如果需要以某种特殊方式标记它们，请更改它们。Elf/gcc
** （3.2 版本及更高版本）将它们标记为“hidden”，以优化 Lua 编译为共享库时的访问。
** 并非所有 elf 目标都支持此属性。不幸的是，gcc 没有提供一种方法来检查目标是否支持该属性，
** 那些不支持的目标会发出警告。为了避免这些警告，更改为默认定义。
*/
#if defined(__GNUC__) && ((__GNUC__*100 + __GNUC_MINOR__) >= 302) && \
    defined(__ELF__)        /* { */
#define LUAI_FUNC    __attribute__((visibility("internal"))) extern
#else                /* }{ */
#define LUAI_FUNC    extern
#endif                /* } */

#define LUAI_DDEC(dec)    LUAI_FUNC dec
#define LUAI_DDEF    /* empty */

/* }================================================================== */


/*
** {==================================================================
** 与以前版本的兼容性
** ===================================================================
*/

/*
@@ LUA_COMPAT_5_3 控制与 Lua 5.3 版本兼容性的其他宏
** 您可以定义它以获取所有选项，或更改特定选项以适应您的特定需求。
*/
#if defined(LUA_COMPAT_5_3)    /* { */

/*
@@ LUA_COMPAT_MATHLIB 控制数学库中几个已弃用函数的存在
** （这些函数在 5.3 版本中已被正式移除；尽管如此，它们仍然在这里可用。）
*/
#define LUA_COMPAT_MATHLIB

/*
@@ LUA_COMPAT_APIINTCASTS controls the presence of macros for
** manipulating other integer types (lua_pushunsigned, lua_tounsigned,
** luaL_checkint, luaL_checklong, etc.)
** (These macros were also officially removed in 5.3, but they are still
** available here.)
*/
#define LUA_COMPAT_APIINTCASTS


/*
@@ LUA_COMPAT_LT_LE controls the emulation of the '__le' metamethod
** using '__lt'.
*/
#define LUA_COMPAT_LT_LE


/*
@@ The following macros supply trivial compatibility for some
** changes in the API. The macros themselves document how to
** change your code to avoid using them.
** (Once more, these macros were officially removed in 5.3, but they are
** still available here.)
*/
#define lua_strlen(L,i)        lua_rawlen(L, (i))

#define lua_objlen(L,i)        lua_rawlen(L, (i))

#define lua_equal(L,idx1,idx2)        lua_compare(L,(idx1),(idx2),LUA_OPEQ)
#define lua_lessthan(L,idx1,idx2)    lua_compare(L,(idx1),(idx2),LUA_OPLT)

#endif                /* } */

/* }================================================================== */



/*
** {==================================================================
** Configuration for Numbers (low-level part).
** Change these definitions if no predefined LUA_FLOAT_* / LUA_INT_*
** satisfy your needs.
** ===================================================================
*/

/*
@@ LUAI_UACNUMBER is the result of a 'default argument promotion'
@@ over a floating number.
@@ l_floatatt(x) corrects float attribute 'x' to the proper float type
** by prefixing it with one of FLT/DBL/LDBL.
@@ LUA_NUMBER_FRMLEN is the length modifier for writing floats.
@@ LUA_NUMBER_FMT is the format for writing floats.
@@ lua_number2str converts a float to a string.
@@ l_mathop allows the addition of an 'l' or 'f' to all math operations.
@@ l_floor takes the floor of a float.
@@ lua_str2number converts a decimal numeral to a number.
*/


/* The following definitions are good for most cases here */
#define l_floor(x)        (l_mathop(floor)(x))
// 定义宏 l_floor，用于对 x 取下限

#define lua_number2str(s,sz,n)  \
    l_sprintf((s), sz, LUA_NUMBER_FMT, (LUAI_UACNUMBER)(n))
// 定义宏 lua_number2str，用于将数字 n 转换为字符串格式并存储到 s 中

/*
@@ lua_numbertointeger converts a float number with an integral value
** to an integer, or returns 0 if float is not within the range of
** a lua_Integer.  (The range comparisons are tricky because of
** rounding. The tests here assume a two-complement representation,
** where MININTEGER always has an exact representation as a float;
** MAXINTEGER may not have one, and therefore its conversion to float
** may have an ill-defined value.)
*/
#define lua_numbertointeger(n,p) \
  ((n) >= (LUA_NUMBER)(LUA_MININTEGER) && \
   (n) < -(LUA_NUMBER)(LUA_MININTEGER) && \
      (*(p) = (LUA_INTEGER)(n), 1))
// 定义宏 lua_numbertointeger，用于将浮点数转换为整数，或者如果浮点数不在 lua_Integer 范围内，则返回 0

/* now the variable definitions */

#if LUA_FLOAT_TYPE == LUA_FLOAT_FLOAT        /* { single float */

#define LUA_NUMBER    float
// 如果 LUA_FLOAT_TYPE 为 LUA_FLOAT_FLOAT，则定义宏 LUA_NUMBER 为 float

#define l_floatatt(n)        (FLT_##n)
// 如果 LUA_FLOAT_TYPE 为 LUA_FLOAT_FLOAT，则定义宏 l_floatatt，用于获取 float 类型的属性

#define LUAI_UACNUMBER    double
// 如果 LUA_FLOAT_TYPE 为 LUA_FLOAT_FLOAT，则定义宏 LUAI_UACNUMBER 为 double

#define LUA_NUMBER_FRMLEN    ""
#define LUA_NUMBER_FMT        "%.7g"
// 如果 LUA_FLOAT_TYPE 为 LUA_FLOAT_FLOAT，则定义宏 LUA_NUMBER_FRMLEN 为空字符串，定义宏 LUA_NUMBER_FMT 为格式化字符串 "%.7g"

#define l_mathop(op)        op##f
// 如果 LUA_FLOAT_TYPE 为 LUA_FLOAT_FLOAT，则定义宏 l_mathop，用于对操作符 op 进行 float 类型的操作

#define lua_str2number(s,p)    strtof((s), (p))
// 如果 LUA_FLOAT_TYPE 为 LUA_FLOAT_FLOAT，则定义宏 lua_str2number，用于将字符串转换为 float 类型的数字


#elif LUA_FLOAT_TYPE == LUA_FLOAT_LONGDOUBLE    /* }{ long double */

#define LUA_NUMBER    long double
// 如果 LUA_FLOAT_TYPE 为 LUA_FLOAT_LONGDOUBLE，则定义宏 LUA_NUMBER 为 long double

#define l_floatatt(n)        (LDBL_##n)
// 如果 LUA_FLOAT_TYPE 为 LUA_FLOAT_LONGDOUBLE，则定义宏 l_floatatt，用于获取 long double 类型的属性

#define LUAI_UACNUMBER    long double
// 如果 LUA_FLOAT_TYPE 为 LUA_FLOAT_LONGDOUBLE，则定义宏 LUAI_UACNUMBER 为 long double

#define LUA_NUMBER_FRMLEN    "L"
#define LUA_NUMBER_FMT        "%.19Lg"
// 如果 LUA_FLOAT_TYPE 为 LUA_FLOAT_LONGDOUBLE，则定义宏 LUA_NUMBER_FRMLEN 为 "L"，定义宏 LUA_NUMBER_FMT 为格式化字符串 "%.19Lg"

#define l_mathop(op)        op##l
// 如果 LUA_FLOAT_TYPE 为 LUA_FLOAT_LONGDOUBLE，则定义宏 l_mathop，用于对操作符 op 进行 long double 类型的操作

#define lua_str2number(s,p)    strtold((s), (p))
// 如果 LUA_FLOAT_TYPE 为 LUA_FLOAT_LONGDOUBLE，则定义宏 lua_str2number，用于将字符串转换为 long double 类型的数字

#elif LUA_FLOAT_TYPE == LUA_FLOAT_DOUBLE    /* }{ double */

#define LUA_NUMBER    double
// 如果 LUA_FLOAT_TYPE 为 LUA_FLOAT_DOUBLE，则定义宏 LUA_NUMBER 为 double

#define l_floatatt(n)        (DBL_##n)
// 如果 LUA_FLOAT_TYPE 为 LUA_FLOAT_DOUBLE，则定义宏 l_floatatt，用于获取 double 类型的属性

#define LUAI_UACNUMBER    double
// 如果 LUA_FLOAT_TYPE 为 LUA_FLOAT_DOUBLE，则定义宏 LUAI_UACNUMBER 为 double

#define LUA_NUMBER_FRMLEN    ""
#define LUA_NUMBER_FMT        "%.14g"
// 如果 LUA_FLOAT_TYPE 为 LUA_FLOAT_DOUBLE，则定义宏 LUA_NUMBER_FRMLEN 为空字符串，定义宏 LUA_NUMBER_FMT 为格式化字符串 "%.14g"

#define l_mathop(op)        op
// 如果 LUA_FLOAT_TYPE 为 LUA_FLOAT_DOUBLE，则定义宏 l_mathop，用于对操作符 op 进行 double 类型的操作

#define lua_str2number(s,p)    strtod((s), (p))
// 如果 LUA_FLOAT_TYPE 为 LUA_FLOAT_DOUBLE，则定义宏 lua_str2number，用于将字符串转换为 double 类型的数字

#else                        /* }{ */

#error "numeric float type not defined"
// 如果 LUA_FLOAT_TYPE 未定义，则输出错误信息

#endif                    /* } */

/*
@@ LUA_UNSIGNED is the unsigned version of LUA_INTEGER.
*/
/* LUAI_UACINT 是 'default argument promotion' 的结果，是一个 LUA_INTEGER 的默认参数提升 */
/* LUA_INTEGER_FRMLEN 是读取/写入整数的长度修饰符 */
/* LUA_INTEGER_FMT 是写入整数的格式 */
/* LUA_MAXINTEGER 是 LUA_INTEGER 的最大值 */
/* LUA_MININTEGER 是 LUA_INTEGER 的最小值 */
/* LUA_MAXUNSIGNED 是 LUA_UNSIGNED 的最大值 */
/* lua_integer2str 将整数转换为字符串 */

/* 以下的定义对大多数情况都适用 */

/* 使用 LUA_INTEGER_FRMLEN 定义 LUA_INTEGER_FMT */
#define LUA_INTEGER_FMT        "%" LUA_INTEGER_FRMLEN "d"

/* LUAI_UACINT 在这里使用，以避免提升问题（可以将无符号比较转换为有符号比较） */
#define LUAI_UACINT        LUA_INTEGER

/* 将整数定义为无符号 LUAI_UACINT */
#define LUA_UNSIGNED        unsigned LUAI_UACINT

/* 现在是变量定义 */

#if LUA_INT_TYPE == LUA_INT_INT        /* { int */

/* 如果是 int 类型，则定义以下变量 */
#define LUA_INTEGER        int
#define LUA_INTEGER_FRMLEN    ""

#define LUA_MAXINTEGER        INT_MAX
#define LUA_MININTEGER        INT_MIN

#define LUA_MAXUNSIGNED        UINT_MAX

#elif LUA_INT_TYPE == LUA_INT_LONG    /* }{ long */

/* 如果是 long 类型，则定义以下变量 */
#define LUA_INTEGER        long
#define LUA_INTEGER_FRMLEN    "l"

#define LUA_MAXINTEGER        LONG_MAX
#define LUA_MININTEGER        LONG_MIN

#define LUA_MAXUNSIGNED        ULONG_MAX

#elif LUA_INT_TYPE == LUA_INT_LONGLONG    /* }{ long long */

/* 使用宏 LLONG_MAX 作为 C99 兼容性的代理 */
#if defined(LLONG_MAX)        /* { */
/* 使用 ISO C99 的内容 */

#define LUA_INTEGER        long long
#define LUA_INTEGER_FRMLEN    "ll"

#define LUA_MAXINTEGER        LLONG_MAX
#define LUA_MININTEGER        LLONG_MIN

#define LUA_MAXUNSIGNED        ULLONG_MAX

#elif defined(LUA_USE_WINDOWS) /* }{ */
/* 在 Windows 中，可以使用特定的 Windows 类型 */

#define LUA_INTEGER        __int64
#define LUA_INTEGER_FRMLEN    "I64"

#define LUA_MAXINTEGER        _I64_MAX
#define LUA_MININTEGER        _I64_MIN
// 定义 LUA_MININTEGER 为 _I64_MIN

#define LUA_MAXUNSIGNED        _UI64_MAX
// 定义 LUA_MAXUNSIGNED 为 _UI64_MAX

#else                /* }{ */
// 如果不支持 long long 类型

#error "Compiler does not support 'long long'. Use option '-DLUA_32BITS' \
  or '-DLUA_C89_NUMBERS' (see file 'luaconf.h' for details)"
// 报错，编译器不支持 'long long'。使用选项 '-DLUA_32BITS' 或 '-DLUA_C89_NUMBERS'（详见 'luaconf.h' 文件）

#endif                /* }{ */

#else                /* }{ */
// 如果没有定义整数类型

#error "numeric integer type not defined"
// 报错，未定义整数类型

#endif                /* } */

/* }================================================================== */


/*
** {==================================================================
** Dependencies with C99 and other C details
** ===================================================================
*/

/*
@@ l_sprintf is equivalent to 'snprintf' or 'sprintf' in C89.
** (All uses in Lua have only one format item.)
*/
#if !defined(LUA_USE_C89)
#define l_sprintf(s,sz,f,i)    snprintf(s,sz,f,i)
// 如果不是 C89 标准，定义 l_sprintf 为 snprintf
#else
#define l_sprintf(s,sz,f,i)    ((void)(sz), sprintf(s,f,i))
// 否则定义 l_sprintf 为 sprintf
#endif


/*
@@ lua_strx2number converts a hexadecimal numeral to a number.
** In C99, 'strtod' does that conversion. Otherwise, you can
** leave 'lua_strx2number' undefined and Lua will provide its own
** implementation.
*/
#if !defined(LUA_USE_C89)
#define lua_strx2number(s,p)        lua_str2number(s,p)
// 如果不是 C89 标准，定义 lua_strx2number 为 lua_str2number
#endif


/*
@@ lua_pointer2str converts a pointer to a readable string in a
** non-specified way.
*/
#define lua_pointer2str(buff,sz,p)    l_sprintf(buff,sz,"%p",p)
// 定义 lua_pointer2str 为 l_sprintf，将指针转换为可读字符串


/*
@@ lua_number2strx converts a float to a hexadecimal numeral.
** In C99, 'sprintf' (with format specifiers '%a'/'%A') does that.
** Otherwise, you can leave 'lua_number2strx' undefined and Lua will
** provide its own implementation.
*/
#if !defined(LUA_USE_C89)
#define lua_number2strx(L,b,sz,f,n)  \
    ((void)L, l_sprintf(b,sz,f,(LUAI_UACNUMBER)(n)))
// 如果不是 C89 标准，定义 lua_number2strx 为 l_sprintf
#endif


/*
** 'strtof' and 'opf' variants for math functions are not valid in
** C89. Otherwise, the macro 'HUGE_VALF' is a good proxy for testing the
** availability of these variants. ('math.h' is already included in
/*
** all files that use these macros.)
*/
#if defined(LUA_USE_C89) || (defined(HUGE_VAL) && !defined(HUGE_VALF))
#undef l_mathop  /* variants not available */
#undef lua_str2number
#define l_mathop(op)        (lua_Number)op  /* no variant */
#define lua_str2number(s,p)    ((lua_Number)strtod((s), (p)))
#endif


/*
@@ LUA_KCONTEXT is the type of the context ('ctx') for continuation
** functions.  It must be a numerical type; Lua will use 'intptr_t' if
** available, otherwise it will use 'ptrdiff_t' (the nearest thing to
** 'intptr_t' in C89)
*/
#define LUA_KCONTEXT    ptrdiff_t

#if !defined(LUA_USE_C89) && defined(__STDC_VERSION__) && \
    __STDC_VERSION__ >= 199901L
#include <stdint.h>
#if defined(INTPTR_MAX)  /* even in C99 this type is optional */
#undef LUA_KCONTEXT
#define LUA_KCONTEXT    intptr_t
#endif
#endif


/*
@@ lua_getlocaledecpoint gets the locale "radix character" (decimal point).
** Change that if you do not want to use C locales. (Code using this
** macro must include the header 'locale.h'.)
*/
#if !defined(lua_getlocaledecpoint)
#define lua_getlocaledecpoint()        (localeconv()->decimal_point[0])
#endif


/*
** macros to improve jump prediction, used mostly for error handling
** and debug facilities. (Some macros in the Lua API use these macros.
** Define LUA_NOBUILTIN if you do not want '__builtin_expect' in your
** code.)
*/
#if !defined(luai_likely)

#if defined(__GNUC__) && !defined(LUA_NOBUILTIN)
#define luai_likely(x)        (__builtin_expect(((x) != 0), 1))
#define luai_unlikely(x)    (__builtin_expect(((x) != 0), 0))
#else
#define luai_likely(x)        (x)
#define luai_unlikely(x)    (x)
#endif

#endif


#if defined(LUA_CORE) || defined(LUA_LIB)
/* shorter names for Lua's own use */
#define l_likely(x)    luai_likely(x)
#define l_unlikely(x)    luai_unlikely(x)
#endif



/* }================================================================== */


/*
** {==================================================================
/*
** Language Variations
** =====================================================================
*/

/*
@@ LUA_NOCVTN2S/LUA_NOCVTS2N control how Lua performs some
** coercions. Define LUA_NOCVTN2S to turn off automatic coercion from
** numbers to strings. Define LUA_NOCVTS2N to turn off automatic
** coercion from strings to numbers.
*/
/* #define LUA_NOCVTN2S */
/* #define LUA_NOCVTS2N */


/*
@@ LUA_USE_APICHECK turns on several consistency checks on the C API.
** Define it as a help when debugging C code.
*/
#if defined(LUA_USE_APICHECK)
#include <assert.h>
#define luai_apicheck(l,e)    assert(e)
#endif

/* }================================================================== */


/*
** {==================================================================
** Macros that affect the API and must be stable (that is, must be the
** same when you compile Lua and when you compile code that links to
** Lua).
** =====================================================================
*/

/*
@@ LUAI_MAXSTACK limits the size of the Lua stack.
** CHANGE it if you need a different limit. This limit is arbitrary;
** its only purpose is to stop Lua from consuming unlimited stack
** space (and to reserve some numbers for pseudo-indices).
** (It must fit into max(size_t)/32.)
*/
#if LUAI_IS32INT
#define LUAI_MAXSTACK        1000000
#else
#define LUAI_MAXSTACK        15000
#endif


/*
@@ LUA_EXTRASPACE defines the size of a raw memory area associated with
** a Lua state with very fast access.
** CHANGE it if you need a different size.
*/
#define LUA_EXTRASPACE        (sizeof(void *))


/*
@@ LUA_IDSIZE gives the maximum size for the description of the source
@@ of a function in debug information.
** CHANGE it if you want a different size.
*/
#define LUA_IDSIZE    60


/*
@@ LUAL_BUFFERSIZE is the buffer size used by the lauxlib buffer system.
*/
#define LUAL_BUFFERSIZE   ((int)(16 * sizeof(void*) * sizeof(lua_Number)))


/*
# 定义 LUAI_MAXALIGN，用于在联合中确保其他项目的最大对齐。
#define LUAI_MAXALIGN  lua_Number n; double u; void *s; lua_Integer i; long l

/* }================================================================== */

/* =================================================================== */

/*
** 本地配置。您可以使用此空间添加您的重新定义，而无需修改文件的主要部分。
*/

#ifndef

#endif
```