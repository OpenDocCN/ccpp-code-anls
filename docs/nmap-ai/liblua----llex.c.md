# `nmap\liblua\llex.c`

```cpp
/*
** $Id: llex.c $
** Lexical Analyzer
** See Copyright Notice in lua.h
*/

// 定义 llex_c 宏，用于标识 llex.c 文件
#define llex_c
// 定义 LUA_CORE 宏，用于标识 Lua 核心模块
#define LUA_CORE

// 包含预编译头文件 lprefix.h
#include "lprefix.h"

// 包含标准库头文件
#include <locale.h>
#include <string.h>

// 包含 Lua 头文件
#include "lua.h"

// 包含其他 Lua 模块的头文件
#include "lctype.h"
#include "ldebug.h"
#include "ldo.h"
#include "lgc.h"
#include "llex.h"
#include "lobject.h"
#include "lparser.h"
#include "lstate.h"
#include "lstring.h"
#include "ltable.h"
#include "lzio.h"

// 定义宏，用于获取下一个字符
#define next(ls)    (ls->current = zgetc(ls->z))

// 定义宏，用于判断当前字符是否为换行符
#define currIsNewline(ls)    (ls->current == '\n' || ls->current == '\r')

// Lua 保留关键字数组
/* ORDER RESERVED */
static const char *const luaX_tokens [] = {
    "and", "break", "do", "else", "elseif",
    "end", "false", "for", "function", "goto", "if",
    "in", "local", "nil", "not", "or", "repeat",
    "return", "then", "true", "until", "while",
    "//", "..", "...", "==", ">=", "<=", "~=",
    "<<", ">>", "::", "<eof>",
    "<number>", "<integer>", "<name>", "<string>"
};

// 定义宏，用于保存当前字符并获取下一个字符
#define save_and_next(ls) (save(ls, ls->current), next(ls))

// 函数原型声明，用于在词法分析出错时抛出异常
static l_noret lexerror (LexState *ls, const char *msg, int token);

// 函数原型声明，用于保存字符到缓冲区
static void save (LexState *ls, int c) {
  Mbuffer *b = ls->buff;
  if (luaZ_bufflen(b) + 1 > luaZ_sizebuffer(b)) {
    size_t newsize;
    if (luaZ_sizebuffer(b) >= MAX_SIZE/2)
      lexerror(ls, "lexical element too long", 0);
    newsize = luaZ_sizebuffer(b) * 2;
    luaZ_resizebuffer(ls->L, b, newsize);
  }
  b->buffer[luaZ_bufflen(b)++] = cast_char(c);
}

// 初始化词法分析器
void luaX_init (lua_State *L) {
  int i;
  TString *e = luaS_newliteral(L, LUA_ENV);  /* create env name */
  luaC_fix(L, obj2gco(e));  /* never collect this name */
  for (i=0; i<NUM_RESERVED; i++) {
    TString *ts = luaS_new(L, luaX_tokens[i]);
    luaC_fix(L, obj2gco(ts));  /* reserved words are never collected */
    ts->extra = cast_byte(i+1);  /* reserved word */
  }
}

// 将 token 转换为字符串
const char *luaX_token2str (LexState *ls, int token) {
  if (token < FIRST_RESERVED) {  /* single-byte symbols? */
    if (lisprint(token))
      return luaO_pushfstring(ls->L, "'%c'", token);
    else  /* control character */
      return luaO_pushfstring(ls->L, "'<\\%d>'", token);
  }
  else {
    const char *s = luaX_tokens[token - FIRST_RESERVED];
    // 如果是固定格式（符号和保留字）的话，返回格式化后的字符串
    if (token < TK_EOS)  /* fixed format (symbols and reserved words)? */
      return luaO_pushfstring(ls->L, "'%s'", s);
    // 否则，返回字符串本身
    else  /* names, strings, and numerals */
      return s;
  }
# 返回一个字符串表示的 token
static const char *txtToken (LexState *ls, int token) {
  switch (token) {
    # 如果 token 是 TK_NAME、TK_STRING、TK_FLT、TK_INT 中的一个，将当前缓冲区内容保存，并返回格式化后的字符串
    case TK_NAME: case TK_STRING:
    case TK_FLT: case TK_INT:
      save(ls, '\0');
      return luaO_pushfstring(ls->L, "'%s'", luaZ_buffer(ls->buff));
    # 如果 token 不是上述类型之一，将其转换为字符串返回
    default:
      return luaX_token2str(ls, token);
  }
}

# 抛出一个语法错误异常
static l_noret lexerror (LexState *ls, const char *msg, int token) {
  # 将错误信息添加到错误栈中
  msg = luaG_addinfo(ls->L, msg, ls->source, ls->linenumber);
  # 如果 token 存在，将错误信息和 token 转换为字符串后抛出异常
  if (token)
    luaO_pushfstring(ls->L, "%s near %s", msg, txtToken(ls, token));
  luaD_throw(ls->L, LUA_ERRSYNTAX);
}

# 抛出一个语法错误异常
l_noret luaX_syntaxerror (LexState *ls, const char *msg) {
  lexerror(ls, msg, ls->t.token);
}

'''
** 创建一个新的字符串，并将其锚定在扫描器的表中，以便在编译结束之前不会被回收；到那时它应该已经被锚定在某个地方。
** 它还会内部化长字符串，确保每个唯一字符串只有一个副本。这里的表被用作集合：字符串作为键进入，而其值是无关紧要的。
** 我们只是因为 TValue 很容易获得，所以将字符串本身作为值使用。稍后，代码生成可以更改这个值。
'''
TString *luaX_newstring (LexState *ls, const char *str, size_t l) {
  lua_State *L = ls->L;
  # 创建一个新的字符串
  TString *ts = luaS_newlstr(L, str, l);
  # 获取字符串在表中的位置
  const TValue *o = luaH_getstr(ls->h, ts);
  # 如果字符串已经存在，则获取保存的副本
  if (!ttisnil(o))
    ts = keystrval(nodefromval(o));
  else {
    # 如果字符串还未使用
    TValue *stv = s2v(L->top++);
    setsvalue(L, stv, ts);
    luaH_finishset(L, ls->h, stv, o, stv);
    luaC_checkGC(L);
    L->top--;
  }
  return ts;
}

'''
** 增加行号并跳过换行序列（\n、\r、\n\r 或 \r\n 中的任何一个）
'''
static void inclinenumber (LexState *ls) {
  // 保存当前行号
  int old = ls->current;
  // 断言当前字符是换行符
  lua_assert(currIsNewline(ls));
  // 跳过换行符 '\n' 或 '\r'
  next(ls);  /* skip '\n' or '\r' */
  // 如果当前字符是换行符并且不等于之前保存的字符
  if (currIsNewline(ls) && ls->current != old)
    // 跳过换行符 '\n\r' 或 '\r\n'
    next(ls);  /* skip '\n\r' or '\r\n' */
  // 行号加一
  if (++ls->linenumber >= MAX_INT)
    // 如果行号超过最大值，报错
    lexerror(ls, "chunk has too many lines", 0);
}


void luaX_setinput (lua_State *L, LexState *ls, ZIO *z, TString *source,
                    int firstchar) {
  // 初始化词法分析器状态
  ls->t.token = 0;
  ls->L = L;
  ls->current = firstchar;
  ls->lookahead.token = TK_EOS;  /* no look-ahead token */
  ls->z = z;
  ls->fs = NULL;
  ls->linenumber = 1;
  ls->lastline = 1;
  ls->source = source;
  // 获取环境名
  ls->envn = luaS_newliteral(L, LUA_ENV);  /* get env name */
  // 初始化缓冲区
  luaZ_resizebuffer(ls->L, ls->buff, LUA_MINBUFFER);  /* initialize buffer */
}



/*
** =======================================================
** LEXICAL ANALYZER
** =======================================================
*/


static int check_next1 (LexState *ls, int c) {
  // 如果当前字符等于给定字符，移动到下一个字符并返回1
  if (ls->current == c) {
    next(ls);
    return 1;
  }
  else return 0;
}


/*
** Check whether current char is in set 'set' (with two chars) and
** saves it
*/
static int check_next2 (LexState *ls, const char *set) {
  // 断言字符串长度为2
  lua_assert(set[2] == '\0');
  // 如果当前字符等于字符串中的任意一个字符，保存当前字符并返回1
  if (ls->current == set[0] || ls->current == set[1]) {
    save_and_next(ls);
    return 1;
  }
  else return 0;
}


/* LUA_NUMBER */
/*
** This function is quite liberal in what it accepts, as 'luaO_str2num'
** will reject ill-formed numerals. Roughly, it accepts the following
** pattern:
**
**   %d(%x|%.|([Ee][+-]?))* | 0[Xx](%x|%.|([Pp][+-])*
**
** The only tricky part is to accept [+-] only after a valid exponent
** mark, to avoid reading '3-4' or '0xe+1' as a single number.
**
** The caller might have already read an initial dot.
*/
static int read_numeral (LexState *ls, SemInfo *seminfo) {
  TValue obj;  // 声明一个 TValue 类型的变量 obj
  const char *expo = "Ee";  // 初始化一个指向字符串 "Ee" 的指针
  int first = ls->current;  // 获取当前字符的 ASCII 值
  lua_assert(lisdigit(ls->current));  // 断言当前字符是否是数字
  save_and_next(ls);  // 保存当前字符并读取下一个字符
  if (first == '0' && check_next2(ls, "xX"))  /* hexadecimal? */  // 如果第一个字符是 '0' 并且下一个两个字符是 'x' 或 'X'
    expo = "Pp";  // 将 expo 指向字符串 "Pp"
  for (;;) {
    if (check_next2(ls, expo))  /* exponent mark? */  // 如果下一个两个字符是 expo 中的任意一个
      check_next2(ls, "-+");  /* optional exponent sign */  // 检查下一个两个字符是否是 '-' 或 '+'
    else if (lisxdigit(ls->current) || ls->current == '.')  /* '%x|%.' */  // 如果当前字符是十六进制数字或者是 '.'
      save_and_next(ls);  // 保存当前字符并读取下一个字符
    else break;  // 否则跳出循环
  }
  if (lislalpha(ls->current))  /* is numeral touching a letter? */  // 如果当前字符是字母
    save_and_next(ls);  /* force an error */  // 保存当前字符并读取下一个字符，强制报错
  save(ls, '\0');  // 保存空字符到缓冲区
  if (luaO_str2num(luaZ_buffer(ls->buff), &obj) == 0)  /* format error? */  // 如果将缓冲区内容转换为数字失败
    lexerror(ls, "malformed number", TK_FLT);  // 报错，提示数字格式错误
  if (ttisinteger(&obj)) {  // 如果 obj 是整数类型
    seminfo->i = ivalue(&obj);  // 将 obj 转换为整数并赋值给 seminfo->i
    return TK_INT;  // 返回整数类型标记
  }
  else {
    lua_assert(ttisfloat(&obj));  // 断言 obj 是浮点数类型
    seminfo->r = fltvalue(&obj);  // 将 obj 转换为浮点数并赋值给 seminfo->r
    return TK_FLT;  // 返回浮点数类型标记
  }
}


/*
** read a sequence '[=*[' or ']=*]', leaving the last bracket. If
** sequence is well formed, return its number of '='s + 2; otherwise,
** return 1 if it is a single bracket (no '='s and no 2nd bracket);
** otherwise (an unfinished '[==...') return 0.
*/
static size_t skip_sep (LexState *ls) {
  size_t count = 0;  // 初始化计数器为 0
  int s = ls->current;  // 获取当前字符的 ASCII 值
  lua_assert(s == '[' || s == ']');  // 断言当前字符是 '[' 或 ']'
  save_and_next(ls);  // 保存当前字符并读取下一个字符
  while (ls->current == '=') {  // 当当前字符是 '=' 时循环
    save_and_next(ls);  // 保存当前字符并读取下一个字符
    count++;  // 计数器加一
  }
  return (ls->current == s) ? count + 2  // 如果当前字符等于 s，返回 count + 2
         : (count == 0) ? 1  // 如果 count 为 0，返回 1
         : 0;  // 否则返回 0
}


static void read_long_string (LexState *ls, SemInfo *seminfo, size_t sep) {
  int line = ls->linenumber;  /* initial line (for error message) */  // 获取当前行号并赋值给 line
  save_and_next(ls);  /* skip 2nd '[' */  // 保存当前字符并读取下一个字符，跳过第二个 '['
  if (currIsNewline(ls))  /* string starts with a newline? */  // 如果当前字符是换行符
    inclinenumber(ls);  /* skip it */  // 跳过换行符
  for (;;) {  // 无限循环
    switch (ls->current) {  # 根据当前字符进行不同的处理
      case EOZ: {  # 如果当前字符是 EOZ
        const char *what = (seminfo ? "string" : "comment");  # 根据 seminfo 的值确定 what 的取值
        const char *msg = luaO_pushfstring(ls->L, "unfinished long %s (starting at line %d)", what, line);  # 根据 what 和 line 生成错误信息
        lexerror(ls, msg, TK_EOS);  # 抛出词法错误
        break;  # 结束当前 case
      }
      case ']': {  # 如果当前字符是 ']'
        if (skip_sep(ls) == sep) {  # 跳过分隔符，如果和 sep 相等
          save_and_next(ls);  # 保存并移动到下一个字符，跳过第二个 ']'
          goto endloop;  # 跳转到 endloop 标签处
        }
        break;  # 结束当前 case
      }
      case '\n': case '\r': {  # 如果当前字符是换行符
        save(ls, '\n');  # 保存换行符
        inclinenumber(ls);  # 增加行号
        if (!seminfo) luaZ_resetbuffer(ls->buff);  # 如果不是在处理字符串，重置缓冲区
        break;  # 结束当前 case
      }
      default: {  # 默认情况
        if (seminfo) save_and_next(ls);  # 如果在处理字符串，保存并移动到下一个字符
        else next(ls);  # 否则，移动到下一个字符
      }
    }
  } endloop:  # endloop 标签
  if (seminfo)  # 如果在处理字符串
    seminfo->ts = luaX_newstring(ls, luaZ_buffer(ls->buff) + sep, luaZ_bufflen(ls->buff) - 2 * sep);  # 从缓冲区中获取字符串并赋值给 seminfo->ts
# 检查转义字符是否有效，如果无效则报错
static void esccheck (LexState *ls, int c, const char *msg) {
  if (!c) {
    if (ls->current != EOZ)
      save_and_next(ls);  # 将当前字符添加到缓冲区，用于错误消息
    lexerror(ls, msg, TK_STRING);  # 报告词法错误
  }
}

# 获取十六进制转义字符的值
static int gethexa (LexState *ls) {
  save_and_next(ls);  # 保存当前字符并获取下一个字符
  esccheck (ls, lisxdigit(ls->current), "hexadecimal digit expected");  # 检查当前字符是否为十六进制数字，如果不是则报错
  return luaO_hexavalue(ls->current);  # 返回当前字符的十六进制值
}

# 读取十六进制转义字符
static int readhexaesc (LexState *ls) {
  int r = gethexa(ls);  # 获取第一个十六进制字符的值
  r = (r << 4) + gethexa(ls);  # 将第一个十六进制字符的值左移4位，并加上第二个十六进制字符的值
  luaZ_buffremove(ls->buff, 2);  # 从缓冲区中移除2个字符
  return r;  # 返回合并后的十六进制值
}

# 读取UTF-8编码的转义字符
static unsigned long readutf8esc (LexState *ls) {
  unsigned long r;
  int i = 4;  # 需要移除的字符数：'\', 'u', '{', 和第一个数字
  save_and_next(ls);  # 跳过'u'
  esccheck(ls, ls->current == '{', "missing '{'");  # 检查当前字符是否为'{'，如果不是则报错
  r = gethexa(ls);  # 获取第一个十六进制数字
  while (cast_void(save_and_next(ls)), lisxdigit(ls->current)) {
    i++;
    esccheck(ls, r <= (0x7FFFFFFFu >> 4), "UTF-8 value too large");  # 检查UTF-8值是否过大，如果是则报错
    r = (r << 4) + luaO_hexavalue(ls->current);  # 将当前字符的十六进制值左移4位，并加上当前字符的十六进制值
  }
  esccheck(ls, ls->current == '}', "missing '}'");  # 检查当前字符是否为'}'，如果不是则报错
  next(ls);  # 跳过'}'
  luaZ_buffremove(ls->buff, i);  # 从缓冲区中移除i个字符
  return r;  # 返回UTF-8编码的值
}

# 处理UTF-8编码的转义字符
static void utf8esc (LexState *ls) {
  char buff[UTF8BUFFSZ];
  int n = luaO_utf8esc(buff, readutf8esc(ls));  # 将UTF-8编码的值转换为UTF-8字符序列
  for (; n > 0; n--)  # 将字符序列添加到字符串
    save(ls, buff[UTF8BUFFSZ - n]);
}

# 读取十进制转义字符
static int readdecesc (LexState *ls) {
  int i;
  int r = 0;  # 结果累加器
  for (i = 0; i < 3 && lisdigit(ls->current); i++) {  # 读取最多3个数字
    r = 10*r + ls->current - '0';  # 计算十进制值
    save_and_next(ls);  # 保存当前字符并获取下一个字符
  }
  esccheck(ls, r <= UCHAR_MAX, "decimal escape too large");  # 检查十进制转义字符是否过大，如果是则报错
  luaZ_buffremove(ls->buff, i);  # 从缓冲区中移除i个数字
  return r;  # 返回十进制值
}

# 读取字符串
static void read_string (LexState *ls, int del, SemInfo *seminfo) {
  save_and_next(ls);  # 保留定界符（用于错误消息）
  while (ls->current != del) {
    switch (ls->current) {  # 根据当前字符进行判断
      case EOZ:  # 如果当前字符是 EOZ
        lexerror(ls, "unfinished string", TK_EOS);  # 抛出未完成字符串的错误
        break;  /* to avoid warnings */  # 防止出现警告
      case '\n':  # 如果当前字符是换行符
      case '\r':  # 或者是回车符
        lexerror(ls, "unfinished string", TK_STRING);  # 抛出未完成字符串的错误
        break;  /* to avoid warnings */  # 防止出现警告
      case '\\': {  /* escape sequences */  # 如果当前字符是反斜杠，表示转义序列开始
        int c;  /* final character to be saved */  # 定义变量 c，用于保存最终的字符
        save_and_next(ls);  /* keep '\\' for error messages */  # 保存当前字符并获取下一个字符，保留 '\\' 用于错误消息
        switch (ls->current) {  # 根据当前字符进行判断
          case 'a': c = '\a'; goto read_save;  # 如果当前字符是 'a'，则将 c 设置为响铃符，并跳转到 read_save 标签
          case 'b': c = '\b'; goto read_save;  # 如果当前字符是 'b'，则将 c 设置为退格符，并跳转到 read_save 标签
          case 'f': c = '\f'; goto read_save;  # 如果当前字符是 'f'，则将 c 设置为换页符，并跳转到 read_save 标签
          case 'n': c = '\n'; goto read_save;  # 如果当前字符是 'n'，则将 c 设置为换行符，并跳转到 read_save 标签
          case 'r': c = '\r'; goto read_save;  # 如果当前字符是 'r'，则将 c 设置为回车符，并跳转到 read_save 标签
          case 't': c = '\t'; goto read_save;  # 如果当前字符是 't'，则将 c 设置为水平制表符，并跳转到 read_save 标签
          case 'v': c = '\v'; goto read_save;  # 如果当前字符是 'v'，则将 c 设置为垂直制表符，并跳转到 read_save 标签
          case 'x': c = readhexaesc(ls); goto read_save;  # 如果当前字符是 'x'，则调用 readhexaesc 函数获取转义字符，并跳转到 read_save 标签
          case 'u': utf8esc(ls);  goto no_save;  # 如果当前字符是 'u'，则调用 utf8esc 函数，不保存字符
          case '\n': case '\r':  # 如果当前字符是换行符或回车符
            inclinenumber(ls); c = '\n'; goto only_save;  # 增加行号，将 c 设置为换行符，并跳转到 only_save 标签
          case '\\': case '\"': case '\'':  # 如果当前字符是反斜杠、双引号或单引号
            c = ls->current; goto read_save;  # 将 c 设置为当前字符，并跳转到 read_save 标签
          case EOZ: goto no_save;  /* will raise an error next loop */  # 如果当前字符是 EOZ，将不保存字符，并在下一个循环中引发错误
          case 'z': {  /* zap following span of spaces */  # 如果当前字符是 'z'，表示删除后面的空格
            luaZ_buffremove(ls->buff, 1);  /* remove '\\' */  # 从缓冲区中删除一个字符 '\\'
            next(ls);  /* skip the 'z' */  # 跳过字符 'z'
            while (lisspace(ls->current)) {  # 当当前字符是空格时
              if (currIsNewline(ls)) inclinenumber(ls);  # 如果是换行符，则增加行号
              else next(ls);  # 否则获取下一个字符
            }
            goto no_save;  # 跳转到 no_save 标签
          }
          default: {  # 默认情况
            esccheck(ls, lisdigit(ls->current), "invalid escape sequence");  # 检查是否为无效的转义序列
            c = readdecesc(ls);  /* digital escape '\ddd' */  # 读取十进制转义字符
            goto only_save;  # 跳转到 only_save 标签
          }
        }
       read_save:  # read_save 标签
         next(ls);  # 获取下一个字符
         /* go through */  # 继续执行
       only_save:  # only_save 标签
         luaZ_buffremove(ls->buff, 1);  /* remove '\\' */  # 从缓冲区中删除一个字符 '\\'
         save(ls, c);  # 保存字符 c
         /* go through */  # 继续执行
       no_save: break;  # no_save 标签，结束当前循环
      }
      default:  # 默认情况
        save_and_next(ls);  # 保存当前字符并获取下一个字符
    }  // 结束 if 语句块
  }  // 结束 while 语句块
  save_and_next(ls);  // 跳过分隔符
  // 从缓冲区中创建一个新的字符串，存储在语义信息中
  seminfo->ts = luaX_newstring(ls, luaZ_buffer(ls->buff) + 1,
                                   luaZ_bufflen(ls->buff) - 2);
# 从输入流中获取下一个词法单元，将其存储在seminfo中
static int llex (LexState *ls, SemInfo *seminfo) {
  # 重置缓冲区
  luaZ_resetbuffer(ls->buff);
  # 无限循环，直到遇到break或return语句
  for (;;) {
    }
  }
}

# 获取下一个词法单元
void luaX_next (LexState *ls) {
  # 将当前行号保存为上一行的行号
  ls->lastline = ls->linenumber;
  # 如果有预读标记
  if (ls->lookahead.token != TK_EOS) {  /* is there a look-ahead token? */
    # 使用预读标记
    ls->t = ls->lookahead;  /* use this one */
    # 清空预读标记
    ls->lookahead.token = TK_EOS;  /* and discharge it */
  }
  else
    # 读取下一个词法单元
    ls->t.token = llex(ls, &ls->t.seminfo);  /* read next token */
}

# 预读下一个词法单元
int luaX_lookahead (LexState *ls) {
  # 断言预读标记为TK_EOS
  lua_assert(ls->lookahead.token == TK_EOS);
  # 读取下一个词法单元，存储在预读标记中
  ls->lookahead.token = llex(ls, &ls->lookahead.seminfo);
  # 返回预读标记中的词法单元
  return ls->lookahead.token;
}
```