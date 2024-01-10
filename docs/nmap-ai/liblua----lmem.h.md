# `nmap\liblua\lmem.h`

```
/*
** $Id: lmem.h $
** 内存管理器接口
** 请参阅lua.h中的版权声明
*/

#ifndef lmem_h
#define lmem_h

#include <stddef.h>

#include "llimits.h"
#include "lua.h"

#define luaM_error(L)    luaD_throw(L, LUA_ERRMEM)
// 如果可以安全地将'n'乘以类型't'的大小而不会溢出，则返回true
#define luaM_testsize(n,e)  \
    (sizeof(n) >= sizeof(size_t) && cast_sizet((n)) + 1 > MAX_SIZET/(e))

#define luaM_checksize(L,n,e)  \
    (luaM_testsize(n,e) ? luaM_toobig(L) : cast_void(0))
// 计算'n'和'MAX_SIZET/sizeof(t)'之间的最小值，以便结果不大于'n'，并且在乘以类型't'的大小时不会溢出'size_t'。
#define luaM_limitN(n,t)  \
  ((cast_sizet(n) <= MAX_SIZET/sizeof(t)) ? (n) :  \
     cast_uint((MAX_SIZET/sizeof(t))))
// 字符数组不需要任何测试
#define luaM_reallocvchar(L,b,on,n)  \
  cast_charp(luaM_saferealloc_(L, (b), (on)*sizeof(char), (n)*sizeof(char)))
// 释放内存
#define luaM_freemem(L, b, s)    luaM_free_(L, (b), (s))
#define luaM_free(L, b)        luaM_free_(L, (b), sizeof(*(b)))
#define luaM_freearray(L, b, n)   luaM_free_(L, (b), (n)*sizeof(*(b)))
// 分配新的内存
#define luaM_new(L,t)        cast(t*, luaM_malloc_(L, sizeof(t), 0))
#define luaM_newvector(L,n,t)    cast(t*, luaM_malloc_(L, (n)*sizeof(t), 0))
/* 定义一个宏，用于创建一个新的大小为 n 的类型为 t 的向量，并进行大小检查 */
#define luaM_newvectorchecked(L,n,t) \
  (luaM_checksize(L,n,sizeof(t)), luaM_newvector(L,n,t))

/* 定义一个宏，用于在 Lua 状态机 L 中分配一个大小为 s 的 tag 类型的对象 */
#define luaM_newobject(L,tag,s)    luaM_malloc_(L, (s), tag)

/* 定义一个宏，用于在 Lua 状态机 L 中增长一个向量的大小 */
#define luaM_growvector(L,v,nelems,size,t,limit,e) \
    ((v)=cast(t *, luaM_growaux_(L,v,nelems,&(size),sizeof(t), \
                         luaM_limitN(limit,t),e)))

/* 定义一个宏，用于在 Lua 状态机 L 中重新分配一个向量的大小 */
#define luaM_reallocvector(L, v,oldn,n,t) \
   (cast(t *, luaM_realloc_(L, v, cast_sizet(oldn) * sizeof(t), \
                                  cast_sizet(n) * sizeof(t))))

/* 定义一个宏，用于在 Lua 状态机 L 中缩小一个向量的大小 */
#define luaM_shrinkvector(L,v,size,fs,t) \
   ((v)=cast(t *, luaM_shrinkvector_(L, v, &(size), fs, sizeof(t))))

/* 声明一个函数，用于处理内存分配过大的情况 */
LUAI_FUNC l_noret luaM_toobig (lua_State *L);

/* 声明一个函数，用于在 Lua 状态机 L 中重新分配内存 */
/* 不要直接调用 */
LUAI_FUNC void *luaM_realloc_ (lua_State *L, void *block, size_t oldsize,
                                                          size_t size);
/* 声明一个函数，用于在 Lua 状态机 L 中安全地重新分配内存 */
LUAI_FUNC void *luaM_saferealloc_ (lua_State *L, void *block, size_t oldsize,
                                                              size_t size);
/* 声明一个函数，用于在 Lua 状态机 L 中释放内存 */
LUAI_FUNC void luaM_free_ (lua_State *L, void *block, size_t osize);
/* 声明一个函数，用于在 Lua 状态机 L 中增长向量的辅助函数 */
LUAI_FUNC void *luaM_growaux_ (lua_State *L, void *block, int nelems,
                               int *size, int size_elem, int limit,
                               const char *what);
/* 声明一个函数，用于在 Lua 状态机 L 中缩小向量的辅助函数 */
LUAI_FUNC void *luaM_shrinkvector_ (lua_State *L, void *block, int *nelem,
                                    int final_n, int size_elem);
/* 声明一个函数，用于在 Lua 状态机 L 中分配内存 */
LUAI_FUNC void *luaM_malloc_ (lua_State *L, size_t size, int tag);

#endif
```