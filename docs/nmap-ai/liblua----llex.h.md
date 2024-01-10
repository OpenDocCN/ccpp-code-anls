# `nmap\liblua\llex.h`

```
/*
** $Id: llex.h $
** 词法分析器
** 请参阅 lua.h 中的版权声明
*/

#ifndef llex_h
#define llex_h

#include <limits.h>

#include "lobject.h"
#include "lzio.h"


/*
** 单字符标记（终结符）由它们自己的数字代码表示。其他标记从以下值开始。
*/
#define FIRST_RESERVED    (UCHAR_MAX + 1)


#if !defined(LUA_ENV)
#define LUA_ENV        "_ENV"
#endif


/*
* 警告：如果更改此枚举的顺序，
* 使用 grep "ORDER RESERVED"
*/
enum RESERVED {
  /* 由保留字表示的终结符 */
  TK_AND = FIRST_RESERVED, TK_BREAK,
  TK_DO, TK_ELSE, TK_ELSEIF, TK_END, TK_FALSE, TK_FOR, TK_FUNCTION,
  TK_GOTO, TK_IF, TK_IN, TK_LOCAL, TK_NIL, TK_NOT, TK_OR, TK_REPEAT,
  TK_RETURN, TK_THEN, TK_TRUE, TK_UNTIL, TK_WHILE,
  /* 其他终结符 */
  TK_IDIV, TK_CONCAT, TK_DOTS, TK_EQ, TK_GE, TK_LE, TK_NE,
  TK_SHL, TK_SHR,
  TK_DBCOLON, TK_EOS,
  TK_FLT, TK_INT, TK_NAME, TK_STRING
};

/* 保留字的数量 */
#define NUM_RESERVED    (cast_int(TK_WHILE-FIRST_RESERVED + 1))


typedef union {
  lua_Number r;
  lua_Integer i;
  TString *ts;
} SemInfo;  /* 语义信息 */


typedef struct Token {
  int token;
  SemInfo seminfo;
} Token;


/* 词法分析器的状态以及当被所有函数共享时解析器的状态 */
typedef struct LexState {
  int current;  /* 当前字符（charint） */
  int linenumber;  /* 输入行计数器 */
  int lastline;  /* '消耗'的最后一个标记的行 */
  Token t;  /* 当前标记 */
  Token lookahead;  /* 向前看标记 */
  struct FuncState *fs;  /* 当前函数（解析器） */
  struct lua_State *L;
  ZIO *z;  /* 输入流 */
  Mbuffer *buff;  /* 标记缓冲区 */
  Table *h;  /* 避免收集/重用字符串 */
  struct Dyndata *dyd;  /* 解析器使用的动态结构 */
  TString *source;  /* 当前源名称 */
  TString *envn;  /* 环境变量名称 */
} LexState;


LUAI_FUNC void luaX_init (lua_State *L);
// 设置输入流，初始化词法分析器
LUAI_FUNC void luaX_setinput (lua_State *L, LexState *ls, ZIO *z,
                              TString *source, int firstchar);

// 创建新的字符串对象
LUAI_FUNC TString *luaX_newstring (LexState *ls, const char *str, size_t l);

// 获取下一个词法单元
LUAI_FUNC void luaX_next (LexState *ls);

// 预读下一个词法单元，但不移动指针
LUAI_FUNC int luaX_lookahead (LexState *ls);

// 抛出语法错误异常
LUAI_FUNC l_noret luaX_syntaxerror (LexState *ls, const char *s);

// 将词法单元转换为字符串
LUAI_FUNC const char *luaX_token2str (LexState *ls, int token);
```