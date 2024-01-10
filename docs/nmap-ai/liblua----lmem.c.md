# `nmap\liblua\lmem.c`

```
/*
** $Id: lmem.c $
** 内存管理器接口
** 请参阅lua.h中的版权声明
*/

#define lmem_c
#define LUA_CORE

#include "lprefix.h"


#include <stddef.h>

#include "lua.h"

#include "ldebug.h"
#include "ldo.h"
#include "lgc.h"
#include "lmem.h"
#include "lobject.h"
#include "lstate.h"


#if defined(EMERGENCYGCTESTS)
/*
** 当不构建初始状态时，第一次分配将失败。
** （此失败将触发'tryagain'并在每次分配时触发完整的GC循环。）
*/
static void *firsttry (global_State *g, void *block, size_t os, size_t ns) {
  if (completestate(g) && ns > 0)  /* 释放永远不会失败 */
    return NULL;  /* 失败 */
  else  /* 正常分配 */
    return (*g->frealloc)(g->ud, block, os, ns);
}
#else
#define firsttry(g,block,os,ns)    ((*g->frealloc)(g->ud, block, os, ns))
#endif





/*
** 关于realloc函数：
** void *frealloc (void *ud, void *ptr, size_t osize, size_t nsize);
** （'osize'是旧大小，'nsize'是新大小）
**
** - frealloc(ud, p, x, 0) 释放块'p'并返回NULL。
** 特别地，frealloc(ud, NULL, 0, 0) 什么也不做，
** 这相当于在ISO C中释放(NULL)。
**
** - frealloc(ud, NULL, x, s) 创建大小为's'的新块
** （不管'x'）。如果无法创建新块，则返回NULL。
**
** - 否则，frealloc(ud, b, x, y) 将块'b'从大小'x'重新分配为大小'y'。
** 如果无法将块重新分配到新大小，则返回NULL。
*/




/*
** {==================================================================
** 为解析器分配/释放数组的函数
** ===================================================================
*/

/*
** 在解析期间数组的最小大小，以避免将大小重新分配为1，然后2，然后4的开销。
** 所有这些数组将在解析结束时被重新分配为精确大小或擦除。
*/
#define MINSIZEARRAY    4
/*
** 在 Lua 中用于动态增长内存块的辅助函数
** 参数：
** L：Lua 状态机
** block：内存块的指针
** nelems：当前元素个数
** psize：内存块大小的指针
** size_elems：每个元素的大小
** limit：内存块的限制大小
** what：描述内存块用途的字符串
** 返回值：
** 返回一个新的内存块指针
*/
void *luaM_growaux_ (lua_State *L, void *block, int nelems, int *psize,
                     int size_elems, int limit, const char *what) {
  void *newblock;
  int size = *psize;
  if (nelems + 1 <= size)  /* 是否还能容纳一个额外的元素？ */
    return block;  /* 无需进行任何操作 */
  if (size >= limit / 2) {  /* 无法将大小加倍？ */
    if (l_unlikely(size >= limit))  /* 无法进行任何增长？ */
      luaG_runerror(L, "too many %s (limit is %d)", what, limit);
    size = limit;  /* 仍然至少有一个空位 */
  }
  else {
    size *= 2;
    if (size < MINSIZEARRAY)
      size = MINSIZEARRAY;  /* 最小大小 */
  }
  lua_assert(nelems + 1 <= size && size <= limit);
  /* 'limit' 确保乘法不会溢出 */
  newblock = luaM_saferealloc_(L, block, cast_sizet(*psize) * size_elems,
                                         cast_sizet(size) * size_elems);
  *psize = size;  /* 仅在其他操作都成功时更新 */
  return newblock;
}

/*
** 在函数原型中，数组的大小也是其元素的数量（以节省内存）。因此，如果无法将数组缩小到其元素的数量，唯一的选择就是引发错误。
** 参数：
** L：Lua 状态机
** block：内存块的指针
** size：内存块大小的指针
** final_n：最终的元素数量
** size_elem：每个元素的大小
** 返回值：
** 返回一个新的内存块指针
*/
void *luaM_shrinkvector_ (lua_State *L, void *block, int *size,
                          int final_n, int size_elem) {
  void *newblock;
  size_t oldsize = cast_sizet((*size) * size_elem);
  size_t newsize = cast_sizet(final_n * size_elem);
  lua_assert(newsize <= oldsize);
  newblock = luaM_saferealloc_(L, block, oldsize, newsize);
  *size = final_n;
  return newblock;
}

/*
** 当内存块太大时，引发内存分配错误
** 参数：
** L：Lua 状态机
*/
l_noret luaM_toobig (lua_State *L) {
  luaG_runerror(L, "memory allocation error: block too big");
}

/*
** 释放内存
** 参数：
** L：Lua 状态机
** block：内存块的指针
** osize：原始内存块的大小
*/
void luaM_free_ (lua_State *L, void *block, size_t osize) {
  global_State *g = G(L);
  lua_assert((osize == 0) == (block == NULL));
  (*g->frealloc)(g->ud, block, osize, 0);
  g->GCdebt -= osize;
}
/*
** 在分配失败的情况下，此函数将执行紧急垃圾回收以释放一些内存，然后再尝试分配。
** 在状态未完全构建时，不应调用垃圾回收，因为收集器尚未完全初始化。
** 同样，在 'gcstopem' 为 true 时，也不应调用垃圾回收，因为此时解释器正处于收集步骤的中间。
*/
static void *tryagain (lua_State *L, void *block,
                       size_t osize, size_t nsize) {
  global_State *g = G(L);
  if (completestate(g) && !g->gcstopem) {
    luaC_fullgc(L, 1);  /* 尝试释放一些内存... */
    return (*g->frealloc)(g->ud, block, osize, nsize);  /* 再次尝试分配 */
  }
  else return NULL;  /* 无法在没有完整状态的情况下释放任何内存 */
}


/*
** 通用分配例程。
*/
void *luaM_realloc_ (lua_State *L, void *block, size_t osize, size_t nsize) {
  void *newblock;
  global_State *g = G(L);
  lua_assert((osize == 0) == (block == NULL));
  newblock = firsttry(g, block, osize, nsize);
  if (l_unlikely(newblock == NULL && nsize > 0)) {
    newblock = tryagain(L, block, osize, nsize);
    if (newblock == NULL)  /* 仍然没有内存？ */
      return NULL;  /* 不更新 'GCdebt' */
  }
  lua_assert((nsize == 0) == (newblock == NULL));
  g->GCdebt = (g->GCdebt + nsize) - osize;
  return newblock;
}


void *luaM_saferealloc_ (lua_State *L, void *block, size_t osize,
                                                    size_t nsize) {
  void *newblock = luaM_realloc_(L, block, osize, nsize);
  if (l_unlikely(newblock == NULL && nsize > 0))  /* 分配失败？ */
    luaM_error(L);
  return newblock;
}


void *luaM_malloc_ (lua_State *L, size_t size, int tag) {
  if (size == 0)
    return NULL;  /* 就这样了 */
  else {
    global_State *g = G(L);
    void *newblock = firsttry(g, NULL, tag, size);
    if (l_unlikely(newblock == NULL)) {
      newblock = tryagain(L, NULL, tag, size);
      if (newblock == NULL)
        luaM_error(L);
    }
  }
}
    # 将当前内存块的大小累加到垃圾回收器的债务中
    g->GCdebt += size;
    # 返回新分配的内存块
    return newblock;
  }
# 闭合前面的函数定义
```