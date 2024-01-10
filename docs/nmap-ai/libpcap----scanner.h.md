# `nmap\libpcap\scanner.h`

```
#ifndef pcap_HEADER_H
#define pcap_HEADER_H 1
#define pcap_IN_HEADER 1

#line 6 "scanner.h"
/* Must come first for _LARGE_FILE_API on AIX. */
#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

/*
 * Must come first to avoid warnings on Windows.
 *
 * Flex-generated scanners may only include <inttypes.h> if __STDC_VERSION__
 * is defined with a value >= 199901, meaning "full C99", and MSVC may not
 * define it with that value, because it isn't 100% C99-compliant, even
 * though it has an <inttypes.h> capable of defining everything the Flex
 * scanner needs.
 *
 * We, however, will include it if we know we have an MSVC version that has
 * it; this means that we may define the INTn_MAX and UINTn_MAX values in
 * scanner.c, and then include <stdint.h>, which may define them differently
 * (same value, but different string of characters), causing compiler warnings.
 *
 * If we include it here, and they're defined, that'll prevent scanner.c
 * from defining them.  So we include <pcap/pcap-inttypes.h>, to get
 * <inttypes.h> if we have it.
 */
#include <pcap/pcap-inttypes.h>

/*
 * grammar.h requires gencode.h and sometimes breaks in a polluted namespace
 * (see ftmacros.h), so include it early.
 */
#include "gencode.h"
#include "grammar.h"

#include "diag-control.h"

#line 41 "scanner.h"

#define  YY_INT_ALIGNED short int

/* A lexical scanner generated by flex */

#define FLEX_SCANNER
#define YY_FLEX_MAJOR_VERSION 2
#define YY_FLEX_MINOR_VERSION 6
#define YY_FLEX_SUBMINOR_VERSION 0
#if YY_FLEX_SUBMINOR_VERSION > 0
#define FLEX_BETA
#endif

/* First, we deal with  platform-specific or compiler-specific issues. */

/* begin standard C headers. */
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <stdlib.h>

/* end standard C headers. */

/* flex integer type definitions */

#ifndef FLEXINT_H
#define FLEXINT_H

/* C99 systems have <inttypes.h>. Non-C99 systems may or may not. */

#if defined (__STDC_VERSION__) && __STDC_VERSION__ >= 199901L
/* C99 says to define __STDC_LIMIT_MACROS before including stdint.h,
 * if you want the limit (max/min) macros for int types. 
 */
#ifndef __STDC_LIMIT_MACROS
#define __STDC_LIMIT_MACROS 1
#endif

#include <inttypes.h>
typedef int8_t flex_int8_t;  // 定义 int8_t 类型的别名 flex_int8_t
typedef uint8_t flex_uint8_t;  // 定义 uint8_t 类型的别名 flex_uint8_t
typedef int16_t flex_int16_t;  // 定义 int16_t 类型的别名 flex_int16_t
typedef uint16_t flex_uint16_t;  // 定义 uint16_t 类型的别名 flex_uint16_t
typedef int32_t flex_int32_t;  // 定义 int32_t 类型的别名 flex_int32_t
typedef uint32_t flex_uint32_t;  // 定义 uint32_t 类型的别名 flex_uint32_t
#else
typedef signed char flex_int8_t;  // 定义 signed char 类型的别名 flex_int8_t
typedef short int flex_int16_t;  // 定义 short int 类型的别名 flex_int16_t
typedef int flex_int32_t;  // 定义 int 类型的别名 flex_int32_t
typedef unsigned char flex_uint8_t;  // 定义 unsigned char 类型的别名 flex_uint8_t
typedef unsigned short int flex_uint16_t;  // 定义 unsigned short int 类型的别名 flex_uint16_t
typedef unsigned int flex_uint32_t;  // 定义 unsigned int 类型的别名 flex_uint32_t

/* Limits of integral types. */
#ifndef INT8_MIN
#define INT8_MIN               (-128)  // 定义 int8_t 类型的最小值
#endif
#ifndef INT16_MIN
#define INT16_MIN              (-32767-1)  // 定义 int16_t 类型的最小值
#endif
#ifndef INT32_MIN
#define INT32_MIN              (-2147483647-1)  // 定义 int32_t 类型的最小值
#endif
#ifndef INT8_MAX
#define INT8_MAX               (127)  // 定义 int8_t 类型的最大值
#endif
#ifndef INT16_MAX
#define INT16_MAX              (32767)  // 定义 int16_t 类型的最大值
#endif
#ifndef INT32_MAX
#define INT32_MAX              (2147483647)  // 定义 int32_t 类型的最大值
#endif
#ifndef UINT8_MAX
#define UINT8_MAX              (255U)  // 定义 uint8_t 类型的最大值
#endif
#ifndef UINT16_MAX
#define UINT16_MAX             (65535U)  // 定义 uint16_t 类型的最大值
#endif
#ifndef UINT32_MAX
#define UINT32_MAX             (4294967295U)  // 定义 uint32_t 类型的最大值
#endif

#endif /* ! C99 */

#endif /* ! FLEXINT_H */

#ifdef __cplusplus

/* The "const" storage-class-modifier is valid. */
#define YY_USE_CONST

#else    /* ! __cplusplus */

/* C99 requires __STDC__ to be defined as 1. */
#if defined (__STDC__)

#define YY_USE_CONST

#endif    /* defined (__STDC__) */
#endif    /* ! __cplusplus */

#ifdef YY_USE_CONST
#define yyconst const
#else
#define yyconst
#endif

/* An opaque pointer. */
#ifndef YY_TYPEDEF_YY_SCANNER_T
#define YY_TYPEDEF_YY_SCANNER_T
typedef void* yyscan_t;  // 定义 void* 类型的别名 yyscan_t
#endif

/* For convenience, these vars (plus the bison vars far below)
   are macros in the reentrant scanner. */
#define yyin yyg->yyin_r  // 定义 yyin 宏
#define yyout yyg->yyout_r  // 定义 yyout 宏
#define yyextra yyg->yyextra_r  // 定义 yyextra 宏
#define yyleng yyg->yyleng_r  // 定义 yyleng 宏
# 定义宏，将 yytext 定义为 yyg->yytext_r
#define yytext yyg->yytext_r
# 定义宏，将 yylineno 定义为 YY_CURRENT_BUFFER_LVALUE->yy_bs_lineno
#define yylineno (YY_CURRENT_BUFFER_LVALUE->yy_bs_lineno)
# 定义宏，将 yycolumn 定义为 YY_CURRENT_BUFFER_LVALUE->yy_bs_column
#define yycolumn (YY_CURRENT_BUFFER_LVALUE->yy_bs_column)
# 定义宏，将 yy_flex_debug 定义为 yyg->yy_flex_debug_r

# 如果未定义 YY_BUF_SIZE，则根据不同的架构定义默认输入缓冲区的大小
#ifndef YY_BUF_SIZE
#ifdef __ia64__
# 在 IA-64 上，缓冲区大小为 16k，而不是 8k。
# 此外，通常情况下，YY_BUF_SIZE 是 2*YY_READ_BUF_SIZE。
# 对应 __ia64__ 情况也是如此。
#define YY_BUF_SIZE 32768
#else
# 定义默认输入缓冲区的大小为 16384
#define YY_BUF_SIZE 16384
#endif /* __ia64__ */
#endif

# 如果未定义 YY_TYPEDEF_YY_BUFFER_STATE，则定义 YY_BUFFER_STATE 结构
#ifndef YY_TYPEDEF_YY_BUFFER_STATE
#define YY_TYPEDEF_YY_BUFFER_STATE
typedef struct yy_buffer_state *YY_BUFFER_STATE;
#endif

# 如果未定义 YY_TYPEDEF_YY_SIZE_T，则定义 yy_size_t 类型
#ifndef YY_TYPEDEF_YY_SIZE_T
#define YY_TYPEDEF_YY_SIZE_T
typedef size_t yy_size_t;
#endif

# 如果未定义 YY_STRUCT_YY_BUFFER_STATE，则定义 YY_BUFFER_STATE 结构
#ifndef YY_STRUCT_YY_BUFFER_STATE
#define YY_STRUCT_YY_BUFFER_STATE
struct yy_buffer_state
    {
    FILE *yy_input_file;

    char *yy_ch_buf;        /* input buffer */
    char *yy_buf_pos;        /* current position in input buffer */

    /* Size of input buffer in bytes, not including room for EOB
     * characters.
     */
    yy_size_t yy_buf_size;

    /* Number of characters read into yy_ch_buf, not including EOB
     * characters.
     */
    int yy_n_chars;

    /* Whether we "own" the buffer - i.e., we know we created it,
     * and can realloc() it to grow it, and should free() it to
     * delete it.
     */
    int yy_is_our_buffer;

    /* Whether this is an "interactive" input source; if so, and
     * if we're using stdio for input, then we want to use getc()
     * instead of fread(), to make sure we stop fetching input after
     * each newline.
     */
    int yy_is_interactive;

    /* Whether we're considered to be at the beginning of a line.
     * If so, '^' rules will be active on the next match, otherwise
     * not.
     */
    int yy_at_bol;

    int yy_bs_lineno; /**< The line count. */
    int yy_bs_column; /**< The column count. */
    # 定义一个变量，用于标识是否在达到输入缓冲区末尾时尝试填充缓冲区
    int yy_fill_buffer;
    
    # 定义一个变量，用于表示缓冲区的状态
    int yy_buffer_status;
#endif /* !YY_STRUCT_YY_BUFFER_STATE */

// 重新开始解析，重置输入文件和词法分析器
void pcap_restart (FILE *input_file ,yyscan_t yyscanner );

// 切换到新的输入缓冲区
void pcap__switch_to_buffer (YY_BUFFER_STATE new_buffer ,yyscan_t yyscanner );

// 创建新的输入缓冲区
YY_BUFFER_STATE pcap__create_buffer (FILE *file,int size ,yyscan_t yyscanner );

// 删除输入缓冲区
void pcap__delete_buffer (YY_BUFFER_STATE b ,yyscan_t yyscanner );

// 刷新输入缓冲区
void pcap__flush_buffer (YY_BUFFER_STATE b ,yyscan_t yyscanner );

// 压入新的输入缓冲区状态
void pcap_push_buffer_state (YY_BUFFER_STATE new_buffer ,yyscan_t yyscanner );

// 弹出输入缓冲区状态
void pcap_pop_buffer_state (yyscan_t yyscanner );

// 扫描指定缓冲区
YY_BUFFER_STATE pcap__scan_buffer (char *base,yy_size_t size ,yyscan_t yyscanner );

// 扫描指定字符串
YY_BUFFER_STATE pcap__scan_string (yyconst char *yy_str ,yyscan_t yyscanner );

// 扫描指定字节序列
YY_BUFFER_STATE pcap__scan_bytes (yyconst char *bytes,yy_size_t len ,yyscan_t yyscanner );

// 分配内存
void *pcap_alloc (yy_size_t ,yyscan_t yyscanner );

// 重新分配内存
void *pcap_realloc (void *,yy_size_t ,yyscan_t yyscanner );

// 释放内存
void pcap_free (void * ,yyscan_t yyscanner );

/* Begin user sect3 */

// 定义词法分析器的包装函数
#define pcap_wrap(yyscanner) (/*CONSTCOND*/1)

// 定义跳过包装函数
#define YY_SKIP_YYWRAP

// 定义 yytext_ptr 为 yytext_r
#define yytext_ptr yytext_r

#ifdef YY_HEADER_EXPORT_START_CONDITIONS
// 定义初始状态为 0
#define INITIAL 0

#endif

#ifndef YY_NO_UNISTD_H
// 包含非 ANSI 的 unistd.h 头文件
#include <unistd.h>
#endif

// 定义额外类型为 compiler_state_t
#define YY_EXTRA_TYPE compiler_state_t *

// 初始化词法分析器
int pcap_lex_init (yyscan_t* scanner);

// 初始化带有额外信息的词法分析器
int pcap_lex_init_extra (YY_EXTRA_TYPE user_defined,yyscan_t* scanner);

// 销毁词法分析器
int pcap_lex_destroy (yyscan_t yyscanner );

// 获取调试标志
int pcap_get_debug (yyscan_t yyscanner );

// 设置调试标志
void pcap_set_debug (int debug_flag ,yyscan_t yyscanner );

// 获取额外信息
YY_EXTRA_TYPE pcap_get_extra (yyscan_t yyscanner );

// 设置额外信息
void pcap_set_extra (YY_EXTRA_TYPE user_defined ,yyscan_t yyscanner );

// 获取输入文件
FILE *pcap_get_in (yyscan_t yyscanner );
# 设置输入流
void pcap_set_in  (FILE * _in_str ,yyscan_t yyscanner );

# 获取输出流
FILE *pcap_get_out (yyscan_t yyscanner );

# 设置输出流
void pcap_set_out  (FILE * _out_str ,yyscan_t yyscanner );

# 获取长度
yy_size_t pcap_get_leng (yyscan_t yyscanner );

# 获取文本
char *pcap_get_text (yyscan_t yyscanner );

# 获取行号
int pcap_get_lineno (yyscan_t yyscanner );

# 设置行号
void pcap_set_lineno (int _line_number ,yyscan_t yyscanner );

# 获取列号
int pcap_get_column  (yyscan_t yyscanner );

# 设置列号
void pcap_set_column (int _column_no ,yyscan_t yyscanner );

# 获取值
YYSTYPE * pcap_get_lval (yyscan_t yyscanner );

# 设置值
void pcap_set_lval (YYSTYPE * yylval_param ,yyscan_t yyscanner );

# 用户定义的宏
/* Macros after this point can all be overridden by user definitions in
 * section 1.
 */

# 跳过 YYWRAP
#ifndef YY_SKIP_YYWRAP
#ifdef __cplusplus
extern "C" int pcap_wrap (yyscan_t yyscanner );
#else
extern int pcap_wrap (yyscan_t yyscanner );
#endif
#endif

# 定义 yytext_ptr
#ifndef yytext_ptr
static void yy_flex_strncpy (char *,yyconst char *,int ,yyscan_t yyscanner);
#endif

# 需要字符串长度
#ifdef YY_NEED_STRLEN
static int yy_flex_strlen (yyconst char * ,yyscan_t yyscanner);
#endif

# 无输入
#ifndef YY_NO_INPUT

#endif

# 每次读取的缓冲区大小
/* Amount of stuff to slurp up with each read. */
#ifndef YY_READ_BUF_SIZE
#ifdef __ia64__
/* On IA-64, the buffer size is 16k, not 8k */
#define YY_READ_BUF_SIZE 16384
#else
#define YY_READ_BUF_SIZE 8192
#endif /* __ia64__ */
#endif

# 开始条件栈增长的条目数
/* Number of entries by which start-condition stack grows. */
#ifndef YY_START_STACK_INCR
#define YY_START_STACK_INCR 25
#endif

# 默认声明生成的扫描器
/* Default declaration of generated scanner - a define so the user can
 * easily add parameters.
 */
#ifndef YY_DECL
#define YY_DECL_IS_OURS 1

extern int pcap_lex \
               (YYSTYPE * yylval_param ,yyscan_t yyscanner);

#define YY_DECL int pcap_lex \
               (YYSTYPE * yylval_param , yyscan_t yyscanner)
#endif /* !YY_DECL */

# 获取前一个状态
/* yy_get_previous_state - get the state just before the EOB char was reached */

# 取消定义
#undef YY_NEW_FILE
#undef YY_FLUSH_BUFFER
#undef yy_set_bol
#undef yy_new_buffer
#undef yy_set_interactive
#undef YY_DO_BEFORE_ACTION
#ifdef YY_DECL_IS_OURS
// 如果 YY_DECL_IS_OURS 已定义，则取消定义
#undef YY_DECL_IS_OURS
// 取消定义 YY_DECL
#undef YY_DECL
#endif

// 在 scanner.l 文件的第 486 行处插入代码

// 在 scanner.h 文件的第 389 行处插入代码
#ifndef pcap_IN_HEADER
// 取消定义 pcap_IN_HEADER
#undef pcap_IN_HEADER
#endif 
// 如果 pcap_HEADER_H 已定义，则取消定义
#endif /* pcap_HEADER_H */
```