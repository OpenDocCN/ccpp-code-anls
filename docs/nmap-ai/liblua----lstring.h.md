# `nmap\liblua\lstring.h`

```
/*
** $Id: lstring.h $
** 字符串表（保存 Lua 处理的所有字符串）
** 请参阅 lua.h 中的版权声明
*/

#ifndef lstring_h
#define lstring_h

#include "lgc.h"
#include "lobject.h"
#include "lstate.h"


/*
** 内存分配错误消息必须预先分配（在内存耗尽后无法创建）
*/
#define MEMERRMSG       "not enough memory"


/*
** TString 的大小：头部大小加上字符串本身的空间（包括最后的 '\0'）
*/
#define sizelstring(l)  (offsetof(TString, contents) + ((l) + 1) * sizeof(char))

#define luaS_newliteral(L, s)    (luaS_newlstr(L, "" s, \
                                 (sizeof(s)/sizeof(char))-1))


/*
** 测试字符串是否为保留字
*/
#define isreserved(s)    ((s)->tt == LUA_VSHRSTR && (s)->extra > 0)


/*
** 短字符串的相等性，它们总是内部化的
*/
#define eqshrstr(a,b)    check_exp((a)->tt == LUA_VSHRSTR, (a) == (b))


LUAI_FUNC unsigned int luaS_hash (const char *str, size_t l, unsigned int seed);
LUAI_FUNC unsigned int luaS_hashlongstr (TString *ts);
LUAI_FUNC int luaS_eqlngstr (TString *a, TString *b);
LUAI_FUNC void luaS_resize (lua_State *L, int newsize);
LUAI_FUNC void luaS_clearcache (global_State *g);
LUAI_FUNC void luaS_init (lua_State *L);
LUAI_FUNC void luaS_remove (lua_State *L, TString *ts);
LUAI_FUNC Udata *luaS_newudata (lua_State *L, size_t s, int nuvalue);
LUAI_FUNC TString *luaS_newlstr (lua_State *L, const char *str, size_t l);
LUAI_FUNC TString *luaS_new (lua_State *L, const char *str);
LUAI_FUNC TString *luaS_createlngstrobj (lua_State *L, size_t l);


#endif
```