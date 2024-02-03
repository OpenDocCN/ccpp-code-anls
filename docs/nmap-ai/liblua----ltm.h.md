# `nmap\liblua\ltm.h`

```cpp
/*
** $Id: ltm.h $
** Tag methods
** See Copyright Notice in lua.h
*/

#ifndef ltm_h
#define ltm_h


#include "lobject.h"


/*
* WARNING: if you change the order of this enumeration,
* grep "ORDER TM" and "ORDER OP"
*/
typedef enum {
  TM_INDEX,  // 索引元方法
  TM_NEWINDEX,  // 新索引元方法
  TM_GC,  // 垃圾回收元方法
  TM_MODE,  // 模式元方法
  TM_LEN,  // 长度元方法
  TM_EQ,  /* last tag method with fast access */  // 等于元方法，最后一个可以快速访问的元方法
  TM_ADD,  // 加法元方法
  TM_SUB,  // 减法元方法
  TM_MUL,  // 乘法元方法
  TM_MOD,  // 取模元方法
  TM_POW,  // 乘方元方法
  TM_DIV,  // 除法元方法
  TM_IDIV,  // 整除元方法
  TM_BAND,  // 按位与元方法
  TM_BOR,  // 按位或元方法
  TM_BXOR,  // 按位异或元方法
  TM_SHL,  // 左移位元方法
  TM_SHR,  // 右移位元方法
  TM_UNM,  // 负号元方法
  TM_BNOT,  // 按位取反元方法
  TM_LT,  // 小于元方法
  TM_LE,  // 小于等于元方法
  TM_CONCAT,  // 连接元方法
  TM_CALL,  // 调用元方法
  TM_CLOSE,  // 关闭元方法
  TM_N        /* number of elements in the enum */  // 枚举中的元素数量
} TMS;


/*
** Mask with 1 in all fast-access methods. A 1 in any of these bits
** in the flag of a (meta)table means the metatable does not have the
** corresponding metamethod field. (Bit 7 of the flag is used for
** 'isrealasize'.)
*/
#define maskflags    (~(~0u << (TM_EQ + 1)))  // 用于快速访问方法的掩码


/*
** Test whether there is no tagmethod.
** (Because tagmethods use raw accesses, the result may be an "empty" nil.)
*/
#define notm(tm)    ttisnil(tm)  // 测试是否没有元方法


#define gfasttm(g,et,e) ((et) == NULL ? NULL : \
  ((et)->flags & (1u<<(e))) ? NULL : luaT_gettm(et, e, (g)->tmname[e]))  // 获取全局快速元方法


#define fasttm(l,et,e)    gfasttm(G(l), et, e)  // 获取快速元方法


#define ttypename(x)    luaT_typenames_[(x) + 1]  // 获取类型名称


LUAI_DDEC(const char *const luaT_typenames_[LUA_TOTALTYPES];)  // 声明类型名称数组


LUAI_FUNC const char *luaT_objtypename (lua_State *L, const TValue *o);  // 获取对象类型名称


LUAI_FUNC const TValue *luaT_gettm (Table *events, TMS event, TString *ename);  // 获取元方法


LUAI_FUNC const TValue *luaT_gettmbyobj (lua_State *L, const TValue *o,
                                                       TMS event);  // 根据对象获取元方法


LUAI_FUNC void luaT_init (lua_State *L);  // 初始化


LUAI_FUNC void luaT_callTM (lua_State *L, const TValue *f, const TValue *p1,
                            const TValue *p2, const TValue *p3);  // 调用元方法


LUAI_FUNC void luaT_callTMres (lua_State *L, const TValue *f,
                            const TValue *p1, const TValue *p2, StkId p3);  // 调用元方法并返回结果
# 尝试调用二元元方法
LUAI_FUNC void luaT_trybinTM (lua_State *L, const TValue *p1, const TValue *p2,
                              StkId res, TMS event);
# 尝试调用连接元方法
LUAI_FUNC void luaT_tryconcatTM (lua_State *L);
# 尝试调用二元关联元方法
LUAI_FUNC void luaT_trybinassocTM (lua_State *L, const TValue *p1,
       const TValue *p2, int inv, StkId res, TMS event);
# 尝试调用整数二元元方法
LUAI_FUNC void luaT_trybiniTM (lua_State *L, const TValue *p1, lua_Integer i2,
                               int inv, StkId res, TMS event);
# 调用比较元方法
LUAI_FUNC int luaT_callorderTM (lua_State *L, const TValue *p1,
                                const TValue *p2, TMS event);
# 调用整数比较元方法
LUAI_FUNC int luaT_callorderiTM (lua_State *L, const TValue *p1, int v2,
                                 int inv, int isfloat, TMS event);

# 调整可变参数
LUAI_FUNC void luaT_adjustvarargs (lua_State *L, int nfixparams,
                                   struct CallInfo *ci, const Proto *p);
# 获取可变参数
LUAI_FUNC void luaT_getvarargs (lua_State *L, struct CallInfo *ci,
                                              StkId where, int wanted);
```