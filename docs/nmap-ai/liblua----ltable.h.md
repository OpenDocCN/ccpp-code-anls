# `nmap\liblua\ltable.h`

```
/*
** $Id: ltable.h $
** Lua tables (hash)
** See Copyright Notice in lua.h
*/

#ifndef ltable_h
#define ltable_h

#include "lobject.h"


#define gnode(t,i)    (&(t)->node[i])  // 定义获取表 t 的第 i 个节点的宏
#define gval(n)        (&(n)->i_val)   // 定义获取节点 n 的值的宏
#define gnext(n)    ((n)->u.next)      // 定义获取节点 n 的下一个节点的宏


/*
** Clear all bits of fast-access metamethods, which means that the table
** may have any of these metamethods. (First access that fails after the
** clearing will set the bit again.)
*/
#define invalidateTMcache(t)    ((t)->flags &= ~maskflags)  // 清除快速访问元方法的所有位，使表 t 可以有任何元方法


/* true when 't' is using 'dummynode' as its hash part */
#define isdummy(t)        ((t)->lastfree == NULL)  // 当表 t 使用 'dummynode' 作为其哈希部分时返回 true


/* allocated size for hash nodes */
#define allocsizenode(t)    (isdummy(t) ? 0 : sizenode(t))  // 为哈希节点分配的大小


/* returns the Node, given the value of a table entry */
#define nodefromval(v)    cast(Node *, (v))  // 根据表条目的值返回节点


LUAI_FUNC const TValue *luaH_getint (Table *t, lua_Integer key);  // 获取整数键对应的值
LUAI_FUNC void luaH_setint (lua_State *L, Table *t, lua_Integer key, TValue *value);  // 设置整数键对应的值
LUAI_FUNC const TValue *luaH_getshortstr (Table *t, TString *key);  // 获取短字符串键对应的值
LUAI_FUNC const TValue *luaH_getstr (Table *t, TString *key);  // 获取字符串键对应的值
LUAI_FUNC const TValue *luaH_get (Table *t, const TValue *key);  // 获取键对应的值
LUAI_FUNC void luaH_newkey (lua_State *L, Table *t, const TValue *key, TValue *value);  // 创建新的键值对
LUAI_FUNC void luaH_set (lua_State *L, Table *t, const TValue *key, TValue *value);  // 设置键值对
LUAI_FUNC void luaH_finishset (lua_State *L, Table *t, const TValue *key, const TValue *slot, TValue *value);  // 完成键值对的设置
LUAI_FUNC Table *luaH_new (lua_State *L);  // 创建新的表
LUAI_FUNC void luaH_resize (lua_State *L, Table *t, unsigned int nasize, unsigned int nhsize);  // 调整表的大小
LUAI_FUNC void luaH_resizearray (lua_State *L, Table *t, unsigned int nasize);  // 调整表的数组部分的大小
LUAI_FUNC void luaH_free (lua_State *L, Table *t);  // 释放表的内存
# 定义 luaH_next 函数，用于在表 t 中查找键 key 的下一个键值对
LUAI_FUNC int luaH_next (lua_State *L, Table *t, StkId key);

# 定义 luaH_getn 函数，用于获取表 t 的数组部分的大小
LUAI_FUNC lua_Unsigned luaH_getn (Table *t);

# 定义 luaH_realasize 函数，用于获取表 t 的实际大小
LUAI_FUNC unsigned int luaH_realasize (const Table *t);

# 如果定义了 LUA_DEBUG，则定义 luaH_mainposition 函数，用于获取键 key 在表 t 中的主位置
LUAI_FUNC Node *luaH_mainposition (const Table *t, const TValue *key);
# 如果定义了 LUA_DEBUG，则定义 luaH_isdummy 函数，用于判断表 t 是否为虚拟表
LUAI_FUNC int luaH_isdummy (const Table *t);
```