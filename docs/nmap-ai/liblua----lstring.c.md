# `nmap\liblua\lstring.c`

```cpp
/*
** $Id: lstring.c $
** String table (keeps all strings handled by Lua)
** See Copyright Notice in lua.h
*/

// 定义 lstring_c 宏，用于标识 lstring.c 文件
#define lstring_c
// 定义 LUA_CORE 宏，用于标识 Lua 核心模块
#define LUA_CORE

// 包含预编译的头文件 lprefix.h
#include "lprefix.h"

// 包含标准库头文件 string.h
#include <string.h>

// 包含 Lua 头文件 lua.h
#include "lua.h"

// 包含调试模块头文件 ldebug.h
#include "ldebug.h"
// 包含执行模块头文件 ldo.h
#include "ldo.h"
// 包含内存模块头文件 lmem.h
#include "lmem.h"
// 包含对象模块头文件 lobject.h
#include "lobject.h"
// 包含状态模块头文件 lstate.h
#include "lstate.h"
// 包含字符串模块头文件 lstring.h
#include "lstring.h"

// 定义字符串表的最大大小
#define MAXSTRTB    cast_int(luaM_limitN(MAX_INT, TString*))

// 比较长字符串的相等性
int luaS_eqlngstr (TString *a, TString *b) {
  size_t len = a->u.lnglen;
  lua_assert(a->tt == LUA_VLNGSTR && b->tt == LUA_VLNGSTR);
  return (a == b) ||  /* 同一实例或... */
    ((len == b->u.lnglen) &&  /* 相等长度且... */
     (memcmp(getstr(a), getstr(b), len) == 0));  /* 相等内容 */
}

// 计算字符串的哈希值
unsigned int luaS_hash (const char *str, size_t l, unsigned int seed) {
  unsigned int h = seed ^ cast_uint(l);
  for (; l > 0; l--)
    h ^= ((h<<5) + (h>>2) + cast_byte(str[l - 1]));
  return h;
}

// 计算长字符串的哈希值
unsigned int luaS_hashlongstr (TString *ts) {
  lua_assert(ts->tt == LUA_VLNGSTR);
  if (ts->extra == 0) {  /* 没有哈希值？ */
    size_t len = ts->u.lnglen;
    ts->hash = luaS_hash(getstr(ts), len, ts->hash);
    ts->extra = 1;  /* 现在有了哈希值 */
  }
  return ts->hash;
}

// 重新哈希字符串表
static void tablerehash (TString **vect, int osize, int nsize) {
  int i;
  for (i = osize; i < nsize; i++)  /* 清空新元素 */
    vect[i] = NULL;
  for (i = 0; i < osize; i++) {  /* 重新哈希数组的旧部分 */
    TString *p = vect[i];
    vect[i] = NULL;
    while (p) {  /* 对于列表中的每个字符串 */
      TString *hnext = p->u.hnext;  /* 保存下一个 */
      unsigned int h = lmod(p->hash, nsize);  /* 新位置 */
      p->u.hnext = vect[h];  /* 链接到数组中 */
      vect[h] = p;
      p = hnext;
    }
  }
}

/*
** 调整字符串表的大小。如果分配失败，则保持当前大小。
** （这可能会降低性能，但任何非零大小都应该能正常工作。）
*/
void luaS_resize (lua_State *L, int nsize) {
  // 获取全局状态中的字符串表
  stringtable *tb = &G(L)->strt;
  // 保存当前字符串表的大小
  int osize = tb->size;
  TString **newvect;
  if (nsize < osize)  /* 是否需要缩小表? */
    // 重新哈希，将缩小的部分移除
    tablerehash(tb->hash, osize, nsize);
  // 重新分配字符串表的哈希数组
  newvect = luaM_reallocvector(L, tb->hash, osize, nsize, TString*);
  if (l_unlikely(newvect == NULL)) {  /* 重新分配失败? */
    if (nsize < osize)  /* 是否是缩小表? */
      // 恢复到原始大小
      tablerehash(tb->hash, nsize, osize);
    /* 保持表不变 */
  }
  else {  /* 分配成功 */
    tb->hash = newvect;
    tb->size = nsize;
    if (nsize > osize)
      // 为新的大小重新哈希
      tablerehash(newvect, osize, nsize);
  }
}

/*
** 清空 API 字符串缓存。 (条目不能为空，因此用不可收集的字符串填充它们。)
*/
void luaS_clearcache (global_State *g) {
  int i, j;
  for (i = 0; i < STRCACHE_N; i++)
    for (j = 0; j < STRCACHE_M; j++) {
      if (iswhite(g->strcache[i][j]))  /* 条目是否会被回收? */
        g->strcache[i][j] = g->memerrmsg;  /* 用固定的内容替换它 */
    }
}

/*
** 初始化字符串表和字符串缓存
*/
void luaS_init (lua_State *L) {
  global_State *g = G(L);
  int i, j;
  // 获取全局状态中的字符串表
  stringtable *tb = &G(L)->strt;
  // 为字符串表的哈希数组分配内存
  tb->hash = luaM_newvector(L, MINSTRTABSIZE, TString*);
  // 清空数组
  tablerehash(tb->hash, 0, MINSTRTABSIZE);
  // 设置字符串表的大小
  tb->size = MINSTRTABSIZE;
  // 预先创建内存错误消息
  g->memerrmsg = luaS_newliteral(L, MEMERRMSG);
  // 将内存错误消息标记为不可回收
  luaC_fix(L, obj2gco(g->memerrmsg));
  for (i = 0; i < STRCACHE_N; i++)  /* 用有效的字符串填充缓存 */
    for (j = 0; j < STRCACHE_M; j++)
      g->strcache[i][j] = g->memerrmsg;
}

/*
** 创建一个新的字符串对象
*/
static TString *createstrobj (lua_State *L, size_t l, int tag, unsigned int h) {
  // 声明变量
  TString *ts;
  GCObject *o;
  size_t totalsize;  /* total size of TString object */
  // 计算字符串对象的总大小
  totalsize = sizelstring(l);
  // 创建新的对象
  o = luaC_newobj(L, tag, totalsize);
  // 将对象转换为字符串对象
  ts = gco2ts(o);
  // 设置字符串对象的哈希值和额外信息
  ts->hash = h;
  ts->extra = 0;
  // 在字符串末尾添加结束符
  getstr(ts)[l] = '\0';  /* ending 0 */
  // 返回字符串对象
  return ts;
}


TString *luaS_createlngstrobj (lua_State *L, size_t l) {
  // 调用createstrobj函数创建长字符串对象
  TString *ts = createstrobj(L, l, LUA_VLNGSTR, G(L)->seed);
  // 设置长字符串对象的长度
  ts->u.lnglen = l;
  // 返回长字符串对象
  return ts;
}


void luaS_remove (lua_State *L, TString *ts) {
  // 获取全局字符串表
  stringtable *tb = &G(L)->strt;
  // 获取哈希桶
  TString **p = &tb->hash[lmod(ts->hash, tb->size)];
  // 查找要删除的字符串对象
  while (*p != ts)  /* find previous element */
    p = &(*p)->u.hnext;
  // 从哈希桶中移除字符串对象
  *p = (*p)->u.hnext;  /* remove element from its list */
  // 减少字符串表中字符串对象的数量
  tb->nuse--;
}


static void growstrtab (lua_State *L, stringtable *tb) {
  // 如果字符串对象数量过多，则进行垃圾回收
  if (l_unlikely(tb->nuse == MAX_INT)) {  /* too many strings? */
    luaC_fullgc(L, 1);  /* try to free some... */
    // 如果依然过多，则报错
    if (tb->nuse == MAX_INT)  /* still too many? */
      luaM_error(L);  /* cannot even create a message... */
  }
  // 如果字符串表的大小小于最大值的一半，则扩大字符串表
  if (tb->size <= MAXSTRTB / 2)  /* can grow string table? */
    luaS_resize(L, tb->size * 2);
}


/*
** Checks whether short string exists and reuses it or creates a new one.
*/
static TString *internshrstr (lua_State *L, const char *str, size_t l) {
  // 声明变量
  TString *ts;
  global_State *g = G(L);
  stringtable *tb = &g->strt;
  // 计算字符串的哈希值
  unsigned int h = luaS_hash(str, l, g->seed);
  // 获取哈希桶
  TString **list = &tb->hash[lmod(h, tb->size)];
  // 断言字符串不为空
  lua_assert(str != NULL);  /* otherwise 'memcmp'/'memcpy' are undefined */
  // 遍历哈希桶中的字符串对象
  for (ts = *list; ts != NULL; ts = ts->u.hnext) {
    // 如果找到相同的短字符串，则返回该字符串对象
    if (l == ts->shrlen && (memcmp(str, getstr(ts), l * sizeof(char)) == 0)) {
      /* found! */
      // 如果字符串对象已经死亡，则重新激活它
      if (isdead(g, ts))  /* dead (but not collected yet)? */
        changewhite(ts);  /* resurrect it */
      return ts;
    }
  }
  // 如果没有找到相同的短字符串，则需要创建一个新的字符串对象
  // 如果字符串表中的字符串对象数量已经达到上限，则需要扩大字符串表
  if (tb->nuse >= tb->size) {  /* need to grow string table? */
    growstrtab(L, tb);
    list = &tb->hash[lmod(h, tb->size)];  /* 使用新的大小重新计算哈希值 */
  }
  ts = createstrobj(L, l, LUA_VSHRSTR, h);  /* 创建一个新的字符串对象 */
  memcpy(getstr(ts), str, l * sizeof(char));  /* 将字符串内容复制到新创建的字符串对象中 */
  ts->shrlen = cast_byte(l);  /* 设置新字符串对象的长度 */
  ts->u.hnext = *list;  /* 将新字符串对象插入哈希表中 */
  *list = ts;  /* 更新哈希表中的链表指针 */
  tb->nuse++;  /* 更新哈希表中的元素数量 */
  return ts;  /* 返回新创建的字符串对象 */
/*
** 创建一个新的字符串（带有显式长度）
*/
TString *luaS_newlstr (lua_State *L, const char *str, size_t l) {
  // 如果字符串长度小于等于 LUAI_MAXSHORTLEN，则返回一个短字符串
  if (l <= LUAI_MAXSHORTLEN)  /* short string? */
    return internshrstr(L, str, l);
  else {
    TString *ts;
    // 如果长度超过了最大限制，则抛出异常
    if (l_unlikely(l >= (MAX_SIZE - sizeof(TString))/sizeof(char)))
      luaM_toobig(L);
    // 创建一个指定长度的字符串对象
    ts = luaS_createlngstrobj(L, l);
    // 将字符串内容复制到新创建的字符串对象中
    memcpy(getstr(ts), str, l * sizeof(char));
    return ts;
  }
}


/*
** 创建或重用一个以零结尾的字符串，首先在缓存中检查（使用字符串地址作为键）。
** 缓存只能包含以零结尾的字符串，因此可以安全地使用 'strcmp' 来检查命中。
*/
TString *luaS_new (lua_State *L, const char *str) {
  // 计算字符串地址的哈希值
  unsigned int i = point2uint(str) % STRCACHE_N;  /* hash */
  int j;
  TString **p = G(L)->strcache[i];
  // 遍历缓存中的字符串，检查是否命中
  for (j = 0; j < STRCACHE_M; j++) {
    if (strcmp(str, getstr(p[j])) == 0)  /* hit? */
      return p[j];  /* that is it */
  }
  /* normal route */
  // 将最后一个元素移出列表
  for (j = STRCACHE_M - 1; j > 0; j--)
    p[j] = p[j - 1];  /* move out last element */
  // 新元素位于列表的第一个位置
  p[0] = luaS_newlstr(L, str, strlen(str));
  return p[0];
}


// 创建一个新的用户数据对象
Udata *luaS_newudata (lua_State *L, size_t s, int nuvalue) {
  Udata *u;
  int i;
  GCObject *o;
  // 如果大小超过最大限制，则抛出异常
  if (l_unlikely(s > MAX_SIZE - udatamemoffset(nuvalue)))
    luaM_toobig(L);
  // 创建一个新的用户数据对象
  o = luaC_newobj(L, LUA_VUSERDATA, sizeudata(nuvalue, s));
  u = gco2u(o);
  u->len = s;
  u->nuvalue = nuvalue;
  u->metatable = NULL;
  // 初始化用户数据对象中的值
  for (i = 0; i < nuvalue; i++)
    setnilvalue(&u->uv[i].uv);
  return u;
}
```