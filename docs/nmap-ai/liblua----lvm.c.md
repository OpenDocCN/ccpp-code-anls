# `nmap\liblua\lvm.c`

```cpp
/*
** $Id: lvm.c $
** Lua虚拟机
** 请参阅lua.h中的版权声明
*/

#define lvm_c
#define LUA_CORE

#include "lprefix.h"

#include <float.h>
#include <limits.h>
#include <math.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "lua.h"

#include "ldebug.h"
#include "ldo.h"
#include "lfunc.h"
#include "lgc.h"
#include "lobject.h"
#include "lopcodes.h"
#include "lstate.h"
#include "lstring.h"
#include "ltable.h"
#include "ltm.h"
#include "lvm.h"


/*
** 默认情况下，在gcc和兼容的编译器中，在主解释器循环中使用跳转表。
*/
#if !defined(LUA_USE_JUMPTABLE)
#if defined(__GNUC__)
#define LUA_USE_JUMPTABLE    1
#else
#define LUA_USE_JUMPTABLE    0
#endif
#endif



/* 表标记方法链的限制（以避免无限循环） */
#define MAXTAGLOOP    2000


/*
** 'l_intfitsf'检查给定的整数是否在可以转换为浮点数而不四舍五入的范围内。用于比较。
*/

/* 浮点数尾数的位数 */
#define NBM        (l_floatatt(MANT_DIG))

/*
** 检查一些整数是否可能不适合浮点数，测试（maxinteger >> NBM）> 0。 （这意味着（1 << NBM）<= maxinteger。）
** （移位是分段进行的，以避免移位超过整数的大小。在最坏的情况下，NBM == 113，对于长双和sizeof（long）== 32。）
*/
#if ((((LUA_MAXINTEGER >> (NBM / 4)) >> (NBM / 4)) >> (NBM / 4)) \
    >> (NBM - (3 * (NBM / 4))))  >  0

/* 适合浮点数的整数的限制 */
#define MAXINTFITSF    ((lua_Unsigned)1 << NBM)

/* 检查'i'是否在区间[-MAXINTFITSF，MAXINTFITSF]内 */
#define l_intfitsf(i)    ((MAXINTFITSF + l_castS2U(i)) <= (2 * MAXINTFITSF))

#else  /* 所有整数精确适合浮点数 */

#define l_intfitsf(i)    1

#endif


/*
** 尝试将值从字符串转换为数字值。
** 如果值不是字符串，或者是不表示有效数字的字符串（或者从字符串到数字的强制转换
/*
** 尝试将值转换为浮点数。浮点数情况已经由宏 'tonumber' 处理。
*/
int luaV_tonumber_ (const TValue *obj, lua_Number *n) {
  TValue v;
  if (ttisinteger(obj)) {  /* 如果值是整数 */
    *n = cast_num(ivalue(obj));  /* 将整数值转换为浮点数 */
    return 1;
  }
  else if (l_strton(obj, &v)) {  /* 如果值是可转换为数字的字符串 */
    *n = nvalue(&v);  /* 将 'luaO_str2num' 的结果转换为浮点数 */
    return 1;
  }
  else
    return 0;  /* 转换失败 */
}


/*
** 尝试将浮点数转换为整数，根据 'mode' 进行四舍五入。
*/
int luaV_flttointeger (lua_Number n, lua_Integer *p, F2Imod mode) {
  lua_Number f = l_floor(n);  /* 取浮点数的下界 */
  if (n != f) {  /* 不是整数值？ */
    if (mode == F2Ieq) return 0;  /* 如果模式要求整数值，则失败 */
    else if (mode == F2Iceil)  /* 需要向上取整？ */
      f += 1;  /* 将下界转换为上界（注意：n != f） */
  }
  return lua_numbertointeger(f, p);  /* 将浮点数转换为整数 */
}


/*
** 尝试将值转换为整数，根据 'mode' 进行四舍五入，不进行字符串强制转换。
** （"快速通道" 由宏 'tointegerns' 处理。）
*/
int luaV_tointegerns (const TValue *obj, lua_Integer *p, F2Imod mode) {
  if (ttisfloat(obj))  /* 如果值是浮点数 */
    return luaV_flttointeger(fltvalue(obj), p, mode);  /* 将浮点数转换为整数 */
  else if (ttisinteger(obj)) {  /* 如果值是整数 */
    *p = ivalue(obj);  /* 直接赋值整数值 */
    return 1;
  }
  else
    return 0;  /* 转换失败 */
}


/*
** 尝试将值转换为整数。
*/
int luaV_tointeger (const TValue *obj, lua_Integer *p, F2Imod mode) {
  TValue v;
  if (l_strton(obj, &v))  /* 'obj' 指向数字字符串吗？ */
    obj = &v;  /* 将 'obj' 指向对应的数字 */
  return luaV_tointegerns(obj, p, mode);  /* 尝试将值转换为整数 */
}
# 尝试将'for'循环的限制转换为整数，保持循环的语义。如果循环不应该运行，则返回true；否则，'*p'得到整数限制。
# （以下解释假设步长为正数；对于负数步长也是有效的。）
# 如果限制是整数或可以转换为整数（向下取整），那就是限制。
# 否则，检查限制是否可以转换为浮点数。如果浮点数太大，将其截断为LUA_MAXINTEGER。如果浮点数太小，循环不应该运行，因为任何初始整数值都大于这样的限制；因此，函数返回true来表示这一点。（对于后一种情况，没有整数限制是正确的；即使限制为LUA_MININTEGER，对于初始值等于LUA_MININTEGER的情况也会运行一次循环。）
static int forlimit (lua_State *L, lua_Integer init, const TValue *lim,
                                   lua_Integer *p, lua_Integer step) {
  if (!luaV_tointeger(lim, p, (step < 0 ? F2Iceil : F2Ifloor))) {
    /* 不能强制转换为整数 */
    lua_Number flim;  /* 尝试转换为浮点数 */
    if (!tonumber(lim, &flim)) /* 无法转换为浮点数？ */
      luaG_forerror(L, lim, "limit");
    /* else 'flim' is a float out of integer bounds */
    if (luai_numlt(0, flim)) {  /* 如果是正数，它太大了 */
      if (step < 0) return 1;  /* 初始值必须小于它 */
      *p = LUA_MAXINTEGER;  /* 截断 */
    }
    else {  /* 它小于最小整数 */
      if (step > 0) return 1;  /* 初始值必须大于它 */
      *p = LUA_MININTEGER;  /* 截断 */
    }
  }
  return (step > 0 ? init > *p : init < *p);  /* 不运行？ */
}


/*
** 准备一个数值for循环（操作码OP_FORPREP）。
** 返回true以跳过循环。否则，在准备之后，堆栈将如下所示：
**   ra : 内部索引（控制变量的安全副本）
/*
**   ra + 1 : loop counter (integer loops) or limit (float loops)
**   ra + 2 : step
**   ra + 3 : control variable
*/
static int forprep (lua_State *L, StkId ra) {
  // 获取循环初始值的指针
  TValue *pinit = s2v(ra);
  // 获取循环限制值的指针
  TValue *plimit = s2v(ra + 1);
  // 获取循环步长的指针
  TValue *pstep = s2v(ra + 2);
  // 如果初始值和步长都是整数类型
  if (ttisinteger(pinit) && ttisinteger(pstep)) { /* integer loop? */
    // 获取初始值和步长的整数值
    lua_Integer init = ivalue(pinit);
    lua_Integer step = ivalue(pstep);
    lua_Integer limit;
    // 如果步长为0，抛出运行时错误
    if (step == 0)
      luaG_runerror(L, "'for' step is zero");
    // 设置控制变量的值为初始值
    setivalue(s2v(ra + 3), init);  /* control variable */
    // 如果限制值超出范围，返回1跳过循环
    if (forlimit(L, init, plimit, &limit, step))
      return 1;  /* skip the loop */
    else {  /* prepare loop counter */
      lua_Unsigned count;
      // 如果步长大于0，表示递增循环
      if (step > 0) {  /* ascending loop? */
        // 计算循环次数
        count = l_castS2U(limit) - l_castS2U(init);
        if (step != 1)  /* avoid division in the too common case */
          count /= l_castS2U(step);
      }
      else {  /* step < 0; descending loop */
        // 计算循环次数
        count = l_castS2U(init) - l_castS2U(limit);
        /* 'step+1' avoids negating 'mininteger' */
        count /= l_castS2U(-(step + 1)) + 1u;
      }
      /* store the counter in place of the limit (which won't be
         needed anymore) */
      // 将计数器存储在限制值的位置（之后不再需要限制值）
      setivalue(plimit, l_castU2S(count));
    }
  }
  else {  /* try making all values floats */
    lua_Number init; lua_Number limit; lua_Number step;
    // 尝试将所有值转换为浮点数
    if (l_unlikely(!tonumber(plimit, &limit)))
      luaG_forerror(L, plimit, "limit");
    if (l_unlikely(!tonumber(pstep, &step)))
      luaG_forerror(L, pstep, "step");
    if (l_unlikely(!tonumber(pinit, &init)))
      luaG_forerror(L, pinit, "initial value");
    // 如果步长为0，抛出运行时错误
    if (step == 0)
      luaG_runerror(L, "'for' step is zero");
    // 如果初始值和限制值之间的关系不满足循环条件，返回1跳过循环
    if (luai_numlt(0, step) ? luai_numlt(limit, init)
                            : luai_numlt(init, limit))
      return 1;  /* skip the loop */
    else {
      /* 确保内部数值都是浮点数 */
      setfltvalue(plimit, limit);
      setfltvalue(pstep, step);
      setfltvalue(s2v(ra), init);  /* 内部索引 */
      setfltvalue(s2v(ra + 3), init);  /* 控制变量 */
    }
  }
  return 0;
/*
** 执行一个浮点数数值循环的一步，返回 true 表示循环必须继续。
** （整数情况写在 opcode OP_FORLOOP 中，以提高性能。）
*/
static int floatforloop (StkId ra) {
  lua_Number step = fltvalue(s2v(ra + 2));  // 获取步长
  lua_Number limit = fltvalue(s2v(ra + 1));  // 获取上限
  lua_Number idx = fltvalue(s2v(ra));  /* 内部索引 */
  idx = luai_numadd(L, idx, step);  /* 增加索引 */
  if (luai_numlt(0, step) ? luai_numle(idx, limit)  // 判断是否需要继续循环
                          : luai_numle(limit, idx)) {
    chgfltvalue(s2v(ra), idx);  /* 更新内部索引 */
    setfltvalue(s2v(ra + 3), idx);  /* 更新控制变量 */
    return 1;  /* 跳回循环 */
  }
  else
    return 0;  /* 结束循环 */
}


/*
** 完成表访问 'val = t[key]'。
** 如果 'slot' 为 NULL，则 't' 不是表；否则，'slot' 指向 t[k] 条目（必须为空）。
*/
void luaV_finishget (lua_State *L, const TValue *t, TValue *key, StkId val,
                      const TValue *slot) {
  int loop;  /* 计数器，避免无限循环 */
  const TValue *tm;  /* 元方法 */
  for (loop = 0; loop < MAXTAGLOOP; loop++) {
    if (slot == NULL) {  /* 't' 不是表？ */
      lua_assert(!ttistable(t));
      tm = luaT_gettmbyobj(L, t, TM_INDEX);
      if (l_unlikely(notm(tm)))
        luaG_typeerror(L, t, "index");  /* 没有元方法 */
      /* 否则将尝试元方法 */
    }
    else {  /* 't' 是表 */
      lua_assert(isempty(slot));
      tm = fasttm(L, hvalue(t)->metatable, TM_INDEX);  /* 表的元方法 */
      if (tm == NULL) {  /* 没有元方法？ */
        setnilvalue(s2v(val));  /* 结果为 nil */
        return;
      }
      /* 否则将尝试元方法 */
    }
    if (ttisfunction(tm)) {  /* 元方法是一个函数？ */
      luaT_callTMres(L, tm, t, key, val);  /* 调用它 */
      return;
    }
    t = tm;  /* 否则尝试访问 'tm[key]' */
    # 如果能够快速获取到值，则执行快速路径
    if (luaV_fastget(L, t, key, slot, luaH_get)) {  /* fast track? */
      # 将获取到的值设置到目标位置
      setobj2s(L, val, slot);  /* done */
      # 返回结果
      return;
    }
    # 如果无法快速获取到值，则重复执行 'luaV_finishget' 的尾调用
    /* else repeat (tail call 'luaV_finishget') */
  }
  # 如果 '__index' 链太长，可能存在循环，抛出运行时错误
  luaG_runerror(L, "'__index' chain too long; possible loop");
/*
** Finish a table assignment 't[key] = val'.
** If 'slot' is NULL, 't' is not a table.  Otherwise, 'slot' points
** to the entry 't[key]', or to a value with an absent key if there
** is no such entry.  (The value at 'slot' must be empty, otherwise
** 'luaV_fastget' would have done the job.)
*/
void luaV_finishset (lua_State *L, const TValue *t, TValue *key,
                     TValue *val, const TValue *slot) {
  int loop;  /* counter to avoid infinite loops */
  for (loop = 0; loop < MAXTAGLOOP; loop++) {
    const TValue *tm;  /* '__newindex' metamethod */
    if (slot != NULL) {  /* is 't' a table? */
      Table *h = hvalue(t);  /* save 't' table */
      lua_assert(isempty(slot));  /* slot must be empty */
      tm = fasttm(L, h->metatable, TM_NEWINDEX);  /* get metamethod */
      if (tm == NULL) {  /* no metamethod? */
        luaH_finishset(L, h, key, slot, val);  /* set new value */
        invalidateTMcache(h);
        luaC_barrierback(L, obj2gco(h), val);
        return;
      }
      /* else will try the metamethod */
    }
    else {  /* not a table; check metamethod */
      tm = luaT_gettmbyobj(L, t, TM_NEWINDEX);
      if (l_unlikely(notm(tm)))
        luaG_typeerror(L, t, "index");
    }
    /* try the metamethod */
    if (ttisfunction(tm)) {
      luaT_callTM(L, tm, t, key, val);
      return;
    }
    t = tm;  /* else repeat assignment over 'tm' */
    if (luaV_fastget(L, t, key, slot, luaH_get)) {
      luaV_finishfastset(L, t, slot, val);
      return;  /* done */
    }
    /* else 'return luaV_finishset(L, t, key, val, slot)' (loop) */
  }
  luaG_runerror(L, "'__newindex' chain too long; possible loop");
}
static int l_strcmp (const TString *ls, const TString *rs) {
  // 获取字符串 ls 的指针和长度
  const char *l = getstr(ls);
  size_t ll = tsslen(ls);
  // 获取字符串 rs 的指针和长度
  const char *r = getstr(rs);
  size_t lr = tsslen(rs);
  // 循环比较字符串
  for (;;) {  /* for each segment */
    // 按字典顺序比较字符串
    int temp = strcoll(l, r);
    // 如果不相等，则返回比较结果
    if (temp != 0)  /* not equal? */
      return temp;  /* done */
    else {  /* strings are equal up to a '\0' */
      // 获取两个字符串中第一个 '\0' 的索引
      size_t len = strlen(l);  /* index of first '\0' in both strings */
      // 如果 'rs' 已经结束，则返回比较结果
      if (len == lr)  /* 'rs' is finished? */
        return (len == ll) ? 0 : 1;  /* check 'ls' */
      // 如果 'ls' 已经结束，则返回比较结果
      else if (len == ll)  /* 'ls' is finished? */
        return -1;  /* 'ls' is less than 'rs' ('rs' is not finished) */
      // 如果两个字符串都比 'len' 长，则继续比较 '\0' 后面的部分
      len++;
      l += len; ll -= len; r += len; lr -= len;
    }
  }
}


/*
** Check whether integer 'i' is less than float 'f'. If 'i' has an
** exact representation as a float ('l_intfitsf'), compare numbers as
** floats. Otherwise, use the equivalence 'i < f <=> i < ceil(f)'.
** If 'ceil(f)' is out of integer range, either 'f' is greater than
** all integers or less than all integers.
** (The test with 'l_intfitsf' is only for performance; the else
** case is correct for all values, but it is slow due to the conversion
** from float to int.)
** When 'f' is NaN, comparisons must result in false.
*/
l_sinline int LTintfloat (lua_Integer i, lua_Number f) {
  // 如果 'i' 可以精确表示为浮点数，则将它们作为浮点数比较
  if (l_intfitsf(i))
    return luai_numlt(cast_num(i), f);  /* compare them as floats */
  else {  /* i < f <=> i < ceil(f) */
    // 将浮点数 'f' 向上取整，得到整数 'fi'
    lua_Integer fi;
    if (luaV_flttointeger(f, &fi, F2Iceil))  /* fi = ceil(f) */
      return i < fi;   /* compare them as integers */
    else  /* 'f' is either greater or less than all integers */
      return f > 0;  /* greater? */
  }
}


/*
** Check whether integer 'i' is less than or equal to float 'f'.
** See comments on previous function.
*/
l_sinline int LEintfloat (lua_Integer i, lua_Number f) {
  // 如果 'i' 可以精确表示为浮点数，则将它们作为浮点数比较
  if (l_intfitsf(i))
    return luai_numle(cast_num(i), f);  /* 将它们作为浮点数进行比较 */
  else {  /* i <= f <=> i <= floor(f) */
    lua_Integer fi;
    if (luaV_flttointeger(f, &fi, F2Ifloor))  /* 将 f 转换为整数，存储在 fi 中 */
      return i <= fi;   /* 将它们作为整数进行比较 */
    else  /* 'f' 要么大于所有整数，要么小于所有整数 */
      return f > 0;  /* 大于 0 吗？ */
  }
  /*
  ** 检查浮点数 'f' 是否小于整数 'i'。
  ** 参见前一个函数的注释。
  */
  l_sinline int LTfloatint (lua_Number f, lua_Integer i) {
    if (l_intfitsf(i))
      return luai_numlt(f, cast_num(i));  /* 将它们作为浮点数进行比较 */
    else {  /* f < i <=> floor(f) < i */
      lua_Integer fi;
      if (luaV_flttointeger(f, &fi, F2Ifloor))  /* fi = floor(f) */
        return fi < i;   /* 将它们作为整数进行比较 */
      else  /* 'f' 要么大于所有整数，要么小于所有整数 */
        return f < 0;  /* 小于？ */
    }
  }

  /*
  ** 检查浮点数 'f' 是否小于或等于整数 'i'。
  ** 参见前一个函数的注释。
  */
  l_sinline int LEfloatint (lua_Number f, lua_Integer i) {
    if (l_intfitsf(i))
      return luai_numle(f, cast_num(i));  /* 将它们作为浮点数进行比较 */
    else {  /* f <= i <=> ceil(f) <= i */
      lua_Integer fi;
      if (luaV_flttointeger(f, &fi, F2Iceil))  /* fi = ceil(f) */
        return fi <= i;   /* 将它们作为整数进行比较 */
      else  /* 'f' 要么大于所有整数，要么小于所有整数 */
        return f < 0;  /* 小于？ */
    }
  }

  /*
  ** 返回数字的 'l < r'。
  */
  l_sinline int LTnum (const TValue *l, const TValue *r) {
    lua_assert(ttisnumber(l) && ttisnumber(r));
    if (ttisinteger(l)) {
      lua_Integer li = ivalue(l);
      if (ttisinteger(r))
        return li < ivalue(r);  /* 都是整数 */
      else  /* 'l' 是整数，'r' 是浮点数 */
        return LTintfloat(li, fltvalue(r));  /* l < r ? */
    }
    else {
      lua_Number lf = fltvalue(l);  /* 'l' 必须是浮点数 */
      if (ttisfloat(r))
        return luai_numlt(lf, fltvalue(r));  /* 都是浮点数 */
      else  /* 'l' 是浮点数，'r' 是整数 */
        return LTfloatint(lf, ivalue(r));
    }
  }

  /*
  ** 返回数字的 'l <= r'。
  */
  l_sinline int LEnum (const TValue *l, const TValue *r) {
    lua_assert(ttisnumber(l) && ttisnumber(r));
    if (ttisinteger(l)) {
      lua_Integer li = ivalue(l);
      if (ttisinteger(r))
        return li <= ivalue(r);  /* 都是整数 */
  ```
 
    else  /* 'l' is int and 'r' is float */
      return LEintfloat(li, fltvalue(r));  /* l <= r ? */
  }
  else {
    lua_Number lf = fltvalue(l);  /* 'l' must be float */
    if (ttisfloat(r))
      return luai_numle(lf, fltvalue(r));  /* both are float */
    else  /* 'l' is float and 'r' is int */
      return LEfloatint(lf, ivalue(r));
  }

- 如果条件不满足，即'l'是整数而'r'是浮点数，则返回'l'是否小于等于'r'的比较结果
- 否则，如果'l'是浮点数，则将其赋值给变量lf
- 如果'r'也是浮点数，则返回lf是否小于等于'r'的比较结果
- 否则，即'l'是浮点数而'r'是整数，则返回lf是否小于等于'r'的比较结果
/*
** return 'l < r' for non-numbers.
*/
static int lessthanothers (lua_State *L, const TValue *l, const TValue *r) {
  // 断言，确保l和r不是数字
  lua_assert(!ttisnumber(l) || !ttisnumber(r));
  // 如果l和r都是字符串
  if (ttisstring(l) && ttisstring(r))  /* both are strings? */
    // 比较字符串大小，返回结果
    return l_strcmp(tsvalue(l), tsvalue(r)) < 0;
  else
    // 调用元方法进行比较
    return luaT_callorderTM(L, l, r, TM_LT);
}


/*
** Main operation less than; return 'l < r'.
*/
int luaV_lessthan (lua_State *L, const TValue *l, const TValue *r) {
  // 如果l和r都是数字
  if (ttisnumber(l) && ttisnumber(r))  /* both operands are numbers? */
    // 调用LTnum函数进行比较
    return LTnum(l, r);
  else 
    // 调用lessthanothers函数进行比较
    return lessthanothers(L, l, r);
}


/*
** return 'l <= r' for non-numbers.
*/
static int lessequalothers (lua_State *L, const TValue *l, const TValue *r) {
  // 断言，确保l和r不是数字
  lua_assert(!ttisnumber(l) || !ttisnumber(r));
  // 如果l和r都是字符串
  if (ttisstring(l) && ttisstring(r))  /* both are strings? */
    // 比较字符串大小，返回结果
    return l_strcmp(tsvalue(l), tsvalue(r)) <= 0;
  else
    // 调用元方法进行比较
    return luaT_callorderTM(L, l, r, TM_LE);
}


/*
** Main operation less than or equal to; return 'l <= r'.
*/
int luaV_lessequal (lua_State *L, const TValue *l, const TValue *r) {
  // 如果l和r都是数字
  if (ttisnumber(l) && ttisnumber(r))  /* both operands are numbers? */
    // 调用LEnum函数进行比较
    return LEnum(l, r);
  else 
    // 调用lessequalothers函数进行比较
    return lessequalothers(L, l, r);
}


/*
** Main operation for equality of Lua values; return 't1 == t2'.
** L == NULL means raw equality (no metamethods)
*/
int luaV_equalobj (lua_State *L, const TValue *t1, const TValue *t2) {
  const TValue *tm;
  // 如果t1和t2的类型标签不同
  if (ttypetag(t1) != ttypetag(t2)) {  /* not the same variant? */
    // 如果t1和t2的类型不同，或者类型不是数字
    if (ttype(t1) != ttype(t2) || ttype(t1) != LUA_TNUMBER)
      // 返回0，表示不相等
      return 0;  /* only numbers can be equal with different variants */
    else {  /* two numbers with different variants */
      /* One of them is an integer. If the other does not have an
         integer value, they cannot be equal; otherwise, compare their
         integer values. */
      lua_Integer i1, i2;
      // 将t1转换为整数，存储在i1中
      // 如果t1不是整数，返回0
      // 将t2转换为整数，存储在i2中
      // 如果t2不是整数，返回0
      // 比较i1和i2的值，如果相等返回1，否则返回0
      return (luaV_tointegerns(t1, &i1, F2Ieq) &&
              luaV_tointegerns(t2, &i2, F2Ieq) &&
              i1 == i2);
  }
  }
  /* values have same type and same variant */
  switch (ttypetag(t1)) {
    case LUA_VNIL: case LUA_VFALSE: case LUA_VTRUE: return 1;  // 如果值为 nil、false 或 true，则返回 1
    case LUA_VNUMINT: return (ivalue(t1) == ivalue(t2));  // 如果是整数类型，则比较两个值是否相等
    case LUA_VNUMFLT: return luai_numeq(fltvalue(t1), fltvalue(t2));  // 如果是浮点数类型，则调用 luai_numeq 函数比较两个值是否相等
    case LUA_VLIGHTUSERDATA: return pvalue(t1) == pvalue(t2);  // 如果是轻量级用户数据类型，则比较两个指针是否相等
    case LUA_VLCF: return fvalue(t1) == fvalue(t2);  // 如果是 C 函数类型，则比较两个函数指针是否相等
    case LUA_VSHRSTR: return eqshrstr(tsvalue(t1), tsvalue(t2));  // 如果是短字符串类型，则调用 eqshrstr 函数比较两个字符串是否相等
    case LUA_VLNGSTR: return luaS_eqlngstr(tsvalue(t1), tsvalue(t2));  // 如果是长字符串类型，则调用 luaS_eqlngstr 函数比较两个字符串是否相等
    case LUA_VUSERDATA: {
      if (uvalue(t1) == uvalue(t2)) return 1;  // 如果是用户数据类型且指针相等，则返回 1
      else if (L == NULL) return 0;  // 如果 Lua 环境为空，则返回 0
      tm = fasttm(L, uvalue(t1)->metatable, TM_EQ);  // 否则获取元表中的 TM_EQ 元方法
      if (tm == NULL)
        tm = fasttm(L, uvalue(t2)->metatable, TM_EQ);
      break;  /* will try TM */
    }
    case LUA_VTABLE: {
      if (hvalue(t1) == hvalue(t2)) return 1;  // 如果是表类型且指针相等，则返回 1
      else if (L == NULL) return 0;  // 如果 Lua 环境为空，则返回 0
      tm = fasttm(L, hvalue(t1)->metatable, TM_EQ);  // 否则获取元表中的 TM_EQ 元方法
      if (tm == NULL)
        tm = fasttm(L, hvalue(t2)->metatable, TM_EQ);
      break;  /* will try TM */
    }
    default:
      return gcvalue(t1) == gcvalue(t2);  // 其他类型则比较两个值的指针是否相等
  }
  if (tm == NULL)  /* no TM? */
    return 0;  /* objects are different */  // 如果没有元方法，则返回 0，表示对象不相等
  else {
    luaT_callTMres(L, tm, t1, t2, L->top);  /* call TM */  // 否则调用元方法
    return !l_isfalse(s2v(L->top));  // 返回调用结果
  }
/* 宏用于 'luaV_concat' 中，确保 'o' 处的元素是一个字符串 */
#define tostring(L,o)  \
    (ttisstring(o) || (cvt2str(o) && (luaO_tostring(L, o), 1)))

/* 检查 'o' 是否是空字符串 */
#define isemptystr(o)    (ttisshrstring(o) && tsvalue(o)->shrlen == 0)

/* 从栈顶 - n 到栈顶 - 1 复制字符串到缓冲区 */
static void copy2buff (StkId top, int n, char *buff) {
  size_t tl = 0;  /* 已经复制的大小 */
  do {
    size_t l = vslen(s2v(top - n));  /* 正在被复制的字符串的长度 */
    memcpy(buff + tl, svalue(s2v(top - n)), l * sizeof(char));
    tl += l;
  } while (--n > 0);
}

/*
** 连接的主要操作：在栈中连接 'total' 个值，从 'L->top - total' 到 'L->top - 1'。
*/
void luaV_concat (lua_State *L, int total) {
  if (total == 1)
    return;  /* "all" values already concatenated */
  do {
    StkId top = L->top;
    int n = 2;  /* 在这一轮处理的元素数量（至少为 2） */
    if (!(ttisstring(s2v(top - 2)) || cvt2str(s2v(top - 2))) ||
        !tostring(L, s2v(top - 1)))
      luaT_tryconcatTM(L);
    else if (isemptystr(s2v(top - 1)))  /* 第二个操作数是空的？ */
      cast_void(tostring(L, s2v(top - 2)));  /* 结果是第一个操作数 */
    else if (isemptystr(s2v(top - 2))) {  /* 第一个操作数是空字符串？ */
      setobjs2s(L, top - 2, top - 1);  /* 结果是第二个操作数 */
    }
    else {
      /* 如果栈顶有至少两个非空字符串值；尽可能获取更多 */
      size_t tl = vslen(s2v(top - 1));
      TString *ts;
      /* 收集总长度和字符串数量 */
      for (n = 1; n < total && tostring(L, s2v(top - n - 1)); n++) {
        size_t l = vslen(s2v(top - n - 1));
        if (l_unlikely(l >= (MAX_SIZE/sizeof(char)) - tl))
          luaG_runerror(L, "string length overflow");
        tl += l;
      }
      if (tl <= LUAI_MAXSHORTLEN) {  /* 结果是短字符串吗？ */
        char buff[LUAI_MAXSHORTLEN];
        copy2buff(top, n, buff);  /* 将字符串复制到缓冲区 */
        ts = luaS_newlstr(L, buff, tl);
      }
      else {  /* 长字符串；直接将字符串复制到最终结果 */
        ts = luaS_createlngstrobj(L, tl);
        copy2buff(top, n, getstr(ts));
      }
      setsvalue2s(L, top - n, ts);  /* 创建结果 */
    }
    total -= n-1;  /* 获取了'n'个字符串来创建1个新字符串 */
    L->top -= n-1;  /* 弹出'n'个字符串并推入一个新字符串 */
  } while (total > 1);  /* 重复直到只剩下1个结果 */
}

/*
** 主要操作 'ra = #rb'。
*/
void luaV_objlen (lua_State *L, StkId ra, const TValue *rb) {
  const TValue *tm;
  switch (ttypetag(rb)) {
    case LUA_VTABLE: {
      Table *h = hvalue(rb);
      tm = fasttm(L, h->metatable, TM_LEN);
      if (tm) break;  /* 如果存在元方法，则跳出 switch 调用它 */
      setivalue(s2v(ra), luaH_getn(h));  /* 否则使用原始长度 */
      return;
    }
    case LUA_VSHRSTR: {
      setivalue(s2v(ra), tsvalue(rb)->shrlen);
      return;
    }
    case LUA_VLNGSTR: {
      setivalue(s2v(ra), tsvalue(rb)->u.lnglen);
      return;
    }
    default: {  /* 尝试元方法 */
      tm = luaT_gettmbyobj(L, rb, TM_LEN);
      if (l_unlikely(notm(tm)))  /* 没有元方法？ */
        luaG_typeerror(L, rb, "get length of");
      break;
    }
  }
  luaT_callTMres(L, tm, rb, rb, ra);
}


/*
** 整数除法；返回 'm // n'，即 floor(m/n)。
** C 除法会截断其结果（朝零方向舍入）。
** 当 'q >= 0' 或 'q' 为整数时，'floor(q) == trunc(q)'，
** 否则 'floor(q) == trunc(q) - 1'。
*/
lua_Integer luaV_idiv (lua_State *L, lua_Integer m, lua_Integer n) {
  if (l_unlikely(l_castS2U(n) + 1u <= 1u)) {  /* 特殊情况：-1 或 0 */
    if (n == 0)
      luaG_runerror(L, "attempt to divide by zero");
    return intop(-, 0, m);   /* n==-1；避免 0x80000...//-1 溢出 */
  }
  else {
    lua_Integer q = m / n;  /* 执行 C 除法 */
    if ((m ^ n) < 0 && m % n != 0)  /* 'm/n' 会是负非整数？ */
      q -= 1;  /* 对不同舍入方式进行修正 */
    return q;
  }
}


/*
** 整数取模；返回 'm % n'。（假设 C '%' 对负数操作数遵循 C99 行为。参见 luaV_idiv 的前面注释。）
*/
lua_Integer luaV_mod (lua_State *L, lua_Integer m, lua_Integer n) {
  if (l_unlikely(l_castS2U(n) + 1u <= 1u)) {  /* 特殊情况：-1 或 0 */
    if (n == 0)
      luaG_runerror(L, "attempt to perform 'n%%0'");
    return 0;   /* 如果 n 为 -1，避免溢出，返回 0 */
  }
  else {
    lua_Integer r = m % n;  /* 计算 m 除以 n 的余数 */
    if (r != 0 && (r ^ n) < 0)  /* 如果余数不为 0 且结果为负数 */
      r += n;  /* 对不同的四舍五入方式进行修正 */
    return r;  /* 返回计算结果 */
  }
/*
** Float modulus
*/
# 计算浮点数取模
lua_Number luaV_modf (lua_State *L, lua_Number m, lua_Number n) {
  lua_Number r;
  luai_nummod(L, m, n, r);  // 调用内部函数计算浮点数取模
  return r;  // 返回结果
}


/* number of bits in an integer */
#define NBITS    cast_int(sizeof(lua_Integer) * CHAR_BIT)

/*
** Shift left operation. (Shift right just negates 'y'.)
*/
#define luaV_shiftr(x,y)    luaV_shiftl(x,intop(-, 0, y))  // 定义左移操作，右移操作只需对'y'取反


lua_Integer luaV_shiftl (lua_Integer x, lua_Integer y) {
  if (y < 0) {  /* shift right? */
    if (y <= -NBITS) return 0;  // 如果右移位数超过整数位数，则返回0
    else return intop(>>, x, -y);  // 否则进行右移操作
  }
  else {  /* shift left */
    if (y >= NBITS) return 0;  // 如果左移位数超过整数位数，则返回0
    else return intop(<<, x, y);  // 否则进行左移操作
  }
}


/*
** create a new Lua closure, push it in the stack, and initialize
** its upvalues.
*/
// 创建一个新的 Lua 闭包，将其推入堆栈，并初始化其上值
static void pushclosure (lua_State *L, Proto *p, UpVal **encup, StkId base,
                         StkId ra) {
  int nup = p->sizeupvalues;  // 获取上值的数量
  Upvaldesc *uv = p->upvalues;  // 获取上值描述数组
  int i;
  LClosure *ncl = luaF_newLclosure(L, nup);  // 创建一个新的 Lua 闭包
  ncl->p = p;  // 设置闭包的原型
  setclLvalue2s(L, ra, ncl);  /* anchor new closure in stack */  // 将新闭包放入栈中
  for (i = 0; i < nup; i++) {  /* fill in its upvalues */  // 遍历上值数组
    if (uv[i].instack)  /* upvalue refers to local variable? */  // 如果上值引用了局部变量
      ncl->upvals[i] = luaF_findupval(L, base + uv[i].idx);  // 则将上值设置为对应的局部变量
    else  /* get upvalue from enclosing function */  // 否则从封闭函数获取上值
      ncl->upvals[i] = encup[uv[i].idx];  // 设置上值为封闭函数的上值
    luaC_objbarrier(L, ncl, ncl->upvals[i]);  // 对象屏障
  }
}


/*
** finish execution of an opcode interrupted by a yield
*/
// 完成被 yield 中断的操作码的执行
void luaV_finishOp (lua_State *L) {
  CallInfo *ci = L->ci;  // 获取当前调用信息
  StkId base = ci->func + 1;  // 获取基地址
  Instruction inst = *(ci->u.l.savedpc - 1);  /* interrupted instruction */  // 获取被中断的指令
  OpCode op = GET_OPCODE(inst);  // 获取指令操作码
  switch (op) {  /* finish its execution */  // 根据操作码完成执行
    case OP_MMBIN: case OP_MMBINI: case OP_MMBINK: {
      setobjs2s(L, base + GETARG_A(*(ci->u.l.savedpc - 2)), --L->top);  // 设置对象
      break;
    }
    case OP_UNM: case OP_BNOT: case OP_LEN:
    case OP_GETTABUP: case OP_GETTABLE: case OP_GETI:
    case OP_GETFIELD: case OP_SELF: {
      setobjs2s(L, base + GETARG_A(inst), --L->top);  // 设置对象
      break;
    }
    // 处理小于、小于等于、大于、大于等于、等于操作
    case OP_LT: case OP_LE:
    case OP_LTI: case OP_LEI:
    case OP_GTI: case OP_GEI:
    case OP_EQ: {  /* 注意 'OP_EQI'/'OP_EQK' 不能产生结果 */
      // 判断栈顶元素是否为假，将结果保存在 res 中
      int res = !l_isfalse(s2v(L->top - 1));
      // 栈顶指针减一
      L->top--;
#if defined(LUA_COMPAT_LT_LE)
      // 如果定义了 LUA_COMPAT_LT_LE，则执行以下代码块
      if (ci->callstatus & CIST_LEQ) {  /* "<=" using "<" instead? */
        // 如果调用状态中包含 CIST_LEQ 标记，则执行以下代码块
        ci->callstatus ^= CIST_LEQ;  /* clear mark */
        // 清除调用状态中的 CIST_LEQ 标记
        res = !res;  /* negate result */
        // 对结果取反
      }
#endif
      // 断言当前指令的操作码为 OP_JMP
      lua_assert(GET_OPCODE(*ci->u.l.savedpc) == OP_JMP);
      // 如果条件不成立，则跳过跳转指令
      if (res != GETARG_k(inst))  /* condition failed? */
        ci->u.l.savedpc++;  /* skip jump instruction */
      break;
    }
    case OP_CONCAT: {
      // 调用 'luaT_tryconcatTM' 时的栈顶位置
      StkId top = L->top - 1;  /* top when 'luaT_tryconcatTM' was called */
      // 获取第一个要连接的元素
      int a = GETARG_A(inst);      /* first element to concatenate */
      // 尚未连接的元素数量
      int total = cast_int(top - 1 - (base + a));  /* yet to concatenate */
      // 将 TM 结果放在正确的位置
      setobjs2s(L, top - 2, top);  /* put TM result in proper position */
      // 栈顶位置调整为最后一个元素之后的位置
      L->top = top - 1;  /* top is one after last element (at top-2) */
      // 连接它们（可能会再次产生结果）
      luaV_concat(L, total);  /* concat them (may yield again) */
      break;
    }
    case OP_CLOSE: {  /* yielded closing variables */
      // 重复指令以关闭其他变量
      ci->u.l.savedpc--;  /* repeat instruction to close other vars. */
      break;
    }
    case OP_RETURN: {  /* yielded closing variables */
      // 返回指令
      StkId ra = base + GETARG_A(inst);
      /* adjust top to signal correct number of returns, in case the
         return is "up to top" ('isIT') */
      // 调整栈顶位置以表示正确的返回数量，以防返回是 "up to top" ('isIT')
      L->top = ra + ci->u2.nres;
      // 重复指令以关闭其他变量并完成返回
      ci->u.l.savedpc--;
      break;
    }
    default: {
      // 只有这些其他操作码可以产生结果
      lua_assert(op == OP_TFORCALL || op == OP_CALL ||
           op == OP_TAILCALL || op == OP_SETTABUP || op == OP_SETTABLE ||
           op == OP_SETI || op == OP_SETFIELD);
      break;
    }
  }
}




/*
** {==================================================================
** Macros for arithmetic/bitwise/comparison opcodes in 'luaV_execute'
** ===================================================================
*/

#define l_addi(L,a,b)    intop(+, a, b)
#define l_subi(L,a,b)    intop(-, a, b)
#define l_muli(L,a,b)    intop(*, a, b)
#define l_band(a,b)    intop(&, a, b)
# 定义按位与操作，将 a 和 b 按位与，并返回结果

#define l_bor(a,b)    intop(|, a, b)
# 定义按位或操作，将 a 和 b 按位或，并返回结果

#define l_bxor(a,b)    intop(^, a, b)
# 定义按位异或操作，将 a 和 b 按位异或，并返回结果

#define l_lti(a,b)    (a < b)
# 定义小于比较操作，判断 a 是否小于 b

#define l_lei(a,b)    (a <= b)
# 定义小于等于比较操作，判断 a 是否小于等于 b

#define l_gti(a,b)    (a > b)
# 定义大于比较操作，判断 a 是否大于 b

#define l_gei(a,b)    (a >= b)
# 定义大于等于比较操作，判断 a 是否大于等于 b

/*
** Arithmetic operations with immediate operands. 'iop' is the integer
** operation, 'fop' is the float operation.
*/
#define op_arithI(L,iop,fop) {  \
  TValue *v1 = vRB(i);  \
  int imm = GETARG_sC(i);  \
  if (ttisinteger(v1)) {  \
    lua_Integer iv1 = ivalue(v1);  \
    pc++; setivalue(s2v(ra), iop(L, iv1, imm));  \
  }  \
  else if (ttisfloat(v1)) {  \
    lua_Number nb = fltvalue(v1);  \
    lua_Number fimm = cast_num(imm);  \
    pc++; setfltvalue(s2v(ra), fop(L, nb, fimm)); \
  }}

/*
** Auxiliary function for arithmetic operations over floats and others
** with two register operands.
*/
#define op_arithf_aux(L,v1,v2,fop) {  \
  lua_Number n1; lua_Number n2;  \
  if (tonumberns(v1, n1) && tonumberns(v2, n2)) {  \
    pc++; setfltvalue(s2v(ra), fop(L, n1, n2));  \
  }}

/*
** Arithmetic operations over floats and others with register operands.
*/
#define op_arithf(L,fop) {  \
  TValue *v1 = vRB(i);  \
  TValue *v2 = vRC(i);  \
  op_arithf_aux(L, v1, v2, fop); }

/*
** Arithmetic operations with K operands for floats.
*/
#define op_arithfK(L,fop) {  \
  TValue *v1 = vRB(i);  \
  TValue *v2 = KC(i); lua_assert(ttisnumber(v2));  \
  op_arithf_aux(L, v1, v2, fop); }

/*
** Arithmetic operations over integers and floats.
*/
#define op_arith_aux(L,v1,v2,iop,fop) {  \
  if (ttisinteger(v1) && ttisinteger(v2)) {  \
    lua_Integer i1 = ivalue(v1); lua_Integer i2 = ivalue(v2);  \
    pc++; setivalue(s2v(ra), iop(L, i1, i2));  \
  }  \
  else op_arithf_aux(L, v1, v2, fop); }

/*
** Arithmetic operations with register operands.
*/
#define op_arith(L,iop,fop) {  \
  TValue *v1 = vRB(i);  \
  TValue *v2 = vRC(i);  \
  op_arith_aux(L, v1, v2, iop, fop); }
/*
** 使用常量操作数进行算术运算
*/
#define op_arithK(L,iop,fop) {  \
  // 获取寄存器中的值v1
  TValue *v1 = vRB(i);  \
  // 获取常量操作数v2，并确保其为数字类型
  TValue *v2 = KC(i); lua_assert(ttisnumber(v2));  \
  // 调用辅助函数进行算术运算
  op_arith_aux(L, v1, v2, iop, fop); }


/*
** 使用常量操作数进行位运算
*/
#define op_bitwiseK(L,op) {  \
  // 获取寄存器中的值v1
  TValue *v1 = vRB(i);  \
  // 获取常量操作数v2
  TValue *v2 = KC(i);  \
  // 定义整数i1，并获取常量操作数v2的整数值i2
  lua_Integer i1;  \
  lua_Integer i2 = ivalue(v2);  \
  // 如果v1可以转换为整数类型，则进行位运算并设置结果到寄存器ra中
  if (tointegerns(v1, &i1)) {  \
    pc++; setivalue(s2v(ra), op(i1, i2));  \
  }}


/*
** 使用寄存器操作数进行位运算
*/
#define op_bitwise(L,op) {  \
  // 获取寄存器中的值v1
  TValue *v1 = vRB(i);  \
  // 获取寄存器中的值v2
  TValue *v2 = vRC(i);  \
  // 定义整数i1和i2，并将v1和v2转换为整数类型
  lua_Integer i1; lua_Integer i2;  \
  // 如果v1和v2都可以转换为整数类型，则进行位运算并设置结果到寄存器ra中
  if (tointegerns(v1, &i1) && tointegerns(v2, &i2)) {  \
    pc++; setivalue(s2v(ra), op(i1, i2));  \
  }}


/*
** 使用寄存器操作数进行比较操作。'opn'实际上适用于所有数字，但快速跟踪可以提高整数的性能。
*/
#define op_order(L,opi,opn,other) {  \
        // 定义条件变量cond
        int cond;  \
        // 获取寄存器rb中的值
        TValue *rb = vRB(i);  \
        // 如果寄存器ra和rb都是整数类型，则进行整数比较
        if (ttisinteger(s2v(ra)) && ttisinteger(rb)) {  \
          lua_Integer ia = ivalue(s2v(ra));  \
          lua_Integer ib = ivalue(rb);  \
          cond = opi(ia, ib);  \
        }  \
        // 如果寄存器ra和rb都是数字类型，则进行数字比较
        else if (ttisnumber(s2v(ra)) && ttisnumber(rb))  \
          cond = opn(s2v(ra), rb);  \
        // 否则调用其他函数进行比较操作
        else  \
          Protect(cond = other(L, s2v(ra), rb));  \
        // 执行条件跳转
        docondjump(); }
#define op_orderI(L,opi,opf,inv,tm) {  \  # 定义一个宏，用于执行比较操作
        int cond;  \  # 声明一个整型变量 cond
        int im = GETARG_sB(i);  \  # 获取指令参数 i 的 sB 部分，赋值给整型变量 im
        if (ttisinteger(s2v(ra)))  \  # 如果寄存器 ra 中的值是整数类型
          cond = opi(ivalue(s2v(ra)), im);  \  # 执行整数比较操作，将结果赋值给 cond
        else if (ttisfloat(s2v(ra))) {  \  # 如果寄存器 ra 中的值是浮点数类型
          lua_Number fa = fltvalue(s2v(ra));  \  # 获取寄存器 ra 中的浮点数值，赋值给 fa
          lua_Number fim = cast_num(im);  \  # 将整型变量 im 转换为浮点数类型，赋值给 fim
          cond = opf(fa, fim);  \  # 执行浮点数比较操作，将结果赋值给 cond
        }  \  
        else {  \  # 如果寄存器 ra 中的值不是整数也不是浮点数
          int isf = GETARG_C(i);  \  # 获取指令参数 i 的 C 部分，赋值给整型变量 isf
          Protect(cond = luaT_callorderiTM(L, s2v(ra), im, inv, isf, tm));  \  # 调用 luaT_callorderiTM 函数执行比较操作，将结果赋值给 cond
        }  \  
        docondjump(); }  \  # 执行条件跳转

/* }================================================================== */


/*
** {==================================================================
** Function 'luaV_execute': main interpreter loop
** ===================================================================
*/

/*
** some macros for common tasks in 'luaV_execute'
*/


#define RA(i)    (base+GETARG_A(i))  \  # 定义宏 RA，用于获取指令参数 i 的 A 部分
#define RB(i)    (base+GETARG_B(i))  \  # 定义宏 RB，用于获取指令参数 i 的 B 部分
#define vRB(i)    s2v(RB(i))  \  # 定义宏 vRB，用于获取寄存器 RB(i) 中的值
#define KB(i)    (k+GETARG_B(i))  \  # 定义宏 KB，用于获取常量表 k 中的值
#define RC(i)    (base+GETARG_C(i))  \  # 定义宏 RC，用于获取指令参数 i 的 C 部分
#define vRC(i)    s2v(RC(i))  \  # 定义宏 vRC，用于获取寄存器 RC(i) 中的值
#define KC(i)    (k+GETARG_C(i))  \  # 定义宏 KC，用于获取常量表 k 中的值
#define RKC(i)    ((TESTARG_k(i)) ? k + GETARG_C(i) : s2v(base + GETARG_C(i)))  \  # 定义宏 RKC，根据指令参数 i 的 k 部分获取常量表 k 中的值或者寄存器 base + GETARG_C(i) 中的值


#define updatetrap(ci)  (trap = ci->u.l.trap)  \  # 定义宏 updatetrap，用于更新 trap 变量
#define updatebase(ci)    (base = ci->func + 1)  \  # 定义宏 updatebase，用于更新 base 变量


#define updatestack(ci)  \  # 定义宏 updatestack
    { if (l_unlikely(trap)) { updatebase(ci); ra = RA(i); } }  \  # 如果 trap 变量为真，则更新 base 变量，并将寄存器 ra 的值更新为指令参数 i 的 A 部分


/*
** Execute a jump instruction. The 'updatetrap' allows signals to stop
** tight loops. (Without it, the local copy of 'trap' could never change.)
*/
#define dojump(ci,i,e)    { pc += GETARG_sJ(i) + e; updatetrap(ci); }  \  # 定义宏 dojump，用于执行跳转指令


/* for test instructions, execute the jump instruction that follows it */
#define donextjump(ci)    { Instruction ni = *pc; dojump(ci, ni, 1); }  \  # 定义宏 donextjump，用于执行测试指令后面的跳转指令

/*
** do a conditional jump: skip next instruction if 'cond' is not what
** was expected (parameter 'k'), else do next instruction, which must
** be a jump.
*/
#define docondjump()    if (cond != GETARG_k(i)) pc++; else donextjump(ci);  \  # 定义宏 docondjump，用于执行条件跳转
/*
** 保存当前指令的位置到当前调用信息的 savedpc 字段
*/
#define savepc(L)    (ci->u.l.savedpc = pc)


/*
** 每当代码可能引发错误时，全局变量 'pc' 和全局变量 'top' 必须正确，以报告偶发错误。
*/
#define savestate(L,ci)        (savepc(L), L->top = ci->top)


/*
** 保护可能引发错误的代码，一般情况下，重新分配堆栈并更改钩子。
*/
#define Protect(exp)  (savestate(L,ci), (exp), updatetrap(ci))

/* 不改变 top 的特殊版本 */
#define ProtectNT(exp)  (savepc(L), (exp), updatetrap(ci))

/*
** 保护只能引发错误的代码。（也就是说，它不能改变堆栈或钩子。）
*/
#define halfProtect(exp)  (savestate(L,ci), (exp))

/* 'c' 是堆栈中活跃值的限制 */
#define checkGC(L,c)  \
    { luaC_condGC(L, (savepc(L), L->top = (c)), \
                         updatetrap(ci)); \
           luai_threadyield(L); }


/* 获取一条指令并准备执行 */
#define vmfetch()    { \
  if (l_unlikely(trap)) {  /* 堆栈重新分配或钩子？ */ \
    trap = luaG_traceexec(L, pc);  /* 处理钩子 */ \
    updatebase(ci);  /* 正确的堆栈 */ \
  } \
  i = *(pc++); \
  ra = RA(i); /* 警告：任何堆栈重新分配都会使 'ra' 无效 */ \
}

#define vmdispatch(o)    switch(o)
#define vmcase(l)    case l:
#define vmbreak        break
    ci->u.l.trap = 1;  /* 假设陷阱当前是打开的 */
  }
  base = ci->func + 1;
  /* 解释器的主循环 */
  for (;;) {
    Instruction i;  /* 正在执行的指令 */
    StkId ra;  /* 指令的 A 寄存器 */
    vmfetch();
    #if 0
      /* 用于调试 Lua 的低级行跟踪 */
      printf("line: %d\n", luaG_getfuncline(cl->p, pcRel(pc, cl->p)));
    #endif
    lua_assert(base == ci->func + 1);
    lua_assert(base <= L->top && L->top < L->stack_last);
    /* 对于不期望顶部的指令，使顶部无效 */
    lua_assert(isIT(i) || (cast_void(L->top = base), 1));
    }
  }
# 闭合大括号，表示代码块的结束
}

# 分隔线，表示代码块的结束
/* }================================================================== */
```