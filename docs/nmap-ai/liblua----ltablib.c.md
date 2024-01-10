# `nmap\liblua\ltablib.c`

```
/*
** $Id: ltablib.c $
** Library for Table Manipulation
** See Copyright Notice in lua.h
*/

#define ltablib_c
#define LUA_LIB

#include "lprefix.h"


#include <limits.h>
#include <stddef.h>
#include <string.h>

#include "lua.h"

#include "lauxlib.h"
#include "lualib.h"


/*
** Operations that an object must define to mimic a table
** (some functions only need some of them)
*/
#define TAB_R    1            /* read */
#define TAB_W    2            /* write */
#define TAB_L    4            /* length */
#define TAB_RW    (TAB_R | TAB_W)        /* read/write */


#define aux_getn(L,n,w)    (checktab(L, n, (w) | TAB_L), luaL_len(L, n))


static int checkfield (lua_State *L, const char *key, int n) {
  lua_pushstring(L, key);
  return (lua_rawget(L, -n) != LUA_TNIL);
}


/*
** Check that 'arg' either is a table or can behave like one (that is,
** has a metatable with the required metamethods)
*/
static void checktab (lua_State *L, int arg, int what) {
  if (lua_type(L, arg) != LUA_TTABLE) {  /* is it not a table? */
    int n = 1;  /* number of elements to pop */
    if (lua_getmetatable(L, arg) &&  /* must have metatable */
        (!(what & TAB_R) || checkfield(L, "__index", ++n)) &&
        (!(what & TAB_W) || checkfield(L, "__newindex", ++n)) &&
        (!(what & TAB_L) || checkfield(L, "__len", ++n))) {
      lua_pop(L, n);  /* pop metatable and tested metamethods */
    }
    else
      luaL_checktype(L, arg, LUA_TTABLE);  /* force an error */
  }
}


static int tinsert (lua_State *L) {
  lua_Integer pos;  /* where to insert new element */
  lua_Integer e = aux_getn(L, 1, TAB_RW);
  e = luaL_intop(+, e, 1);  /* first empty element */
  switch (lua_gettop(L)) {
    case 2: {  /* called with only 2 arguments */
      pos = e;  /* insert new element at the end */
      break;
    }
    case 3: {
      // 声明变量 i 为整数类型
      lua_Integer i;
      // 获取第二个参数作为位置
      pos = luaL_checkinteger(L, 2);  /* 2nd argument is the position */
      /* 检查 'pos' 是否在 [1, e] 范围内 */
      luaL_argcheck(L, (lua_Unsigned)pos - 1u < (lua_Unsigned)e, 2,
                       "position out of bounds");
      // 从 e 开始向上移动元素
      for (i = e; i > pos; i--) {  /* move up elements */
        // 获取 t[i-1] 的值
        lua_geti(L, 1, i - 1);
        // 设置 t[i] 的值为 t[i-1]
        lua_seti(L, 1, i);  /* t[i] = t[i - 1] */
      }
      break;
    }
    default: {
      // 返回错误信息，参数数量错误
      return luaL_error(L, "wrong number of arguments to 'insert'");
    }
  }
  // 设置 t[pos] 的值为 v
  lua_seti(L, 1, pos);  /* t[pos] = v */
  // 返回 0
  return 0;
static int tremove (lua_State *L) {
  // 获取表的大小
  lua_Integer size = aux_getn(L, 1, TAB_RW);
  // 获取要删除的位置，默认为表的大小
  lua_Integer pos = luaL_optinteger(L, 2, size);
  // 如果给定了位置参数，则验证位置是否在有效范围内
  if (pos != size)  
    luaL_argcheck(L, (lua_Unsigned)pos - 1u <= (lua_Unsigned)size, 1,
                     "position out of bounds");
  // 获取要删除的元素
  lua_geti(L, 1, pos);  
  // 将后续元素向前移动一位
  for ( ; pos < size; pos++) {
    lua_geti(L, 1, pos + 1);
    lua_seti(L, 1, pos);  
  }
  // 删除最后一个元素
  lua_pushnil(L);
  lua_seti(L, 1, pos);  
  return 1;
}


/*
** 将元素从一个表复制到另一个表
*/
static int tmove (lua_State *L) {
  // 获取起始位置、结束位置、目标位置
  lua_Integer f = luaL_checkinteger(L, 2);
  lua_Integer e = luaL_checkinteger(L, 3);
  lua_Integer t = luaL_checkinteger(L, 4);
  // 获取目标表的索引，如果没有指定则默认为1
  int tt = !lua_isnoneornil(L, 5) ? 5 : 1;  
  // 检查源表和目标表
  checktab(L, 1, TAB_R);
  checktab(L, tt, TAB_W);
  // 如果结束位置大于等于起始位置，则进行复制操作
  if (e >= f) {  
    lua_Integer n, i;
    // 检查要复制的元素数量是否超出范围
    luaL_argcheck(L, f > 0 || e < LUA_MAXINTEGER + f, 3,
                  "too many elements to move");
    // 计算要复制的元素数量
    n = e - f + 1;  
    // 检查目标位置是否超出范围
    luaL_argcheck(L, t <= LUA_MAXINTEGER - n + 1, 4,
                  "destination wrap around");
    // 根据情况选择复制顺序
    if (t > e || t <= f || (tt != 1 && !lua_compare(L, 1, tt, LUA_OPEQ))) {
      for (i = 0; i < n; i++) {
        lua_geti(L, 1, f + i);
        lua_seti(L, tt, t + i);
      }
    }
    else {
      for (i = n - 1; i >= 0; i--) {
        lua_geti(L, 1, f + i);
        lua_seti(L, tt, t + i);
      }
    }
  }
  // 返回目标表
  lua_pushvalue(L, tt);  
  return 1;
}


static void addfield (lua_State *L, luaL_Buffer *b, lua_Integer i) {
  // 获取表中指定位置的元素
  lua_geti(L, 1, i);
  // 如果元素不是字符串类型
  if (l_unlikely(!lua_isstring(L, -1)))
    # 在 Lua 中抛出一个错误，指示在表中的索引位置上有一个无效的值
    luaL_error(L, "invalid value (%s) at index %I in table for 'concat'",
                  luaL_typename(L, -1), (LUAI_UACINT)i);
    # 将栈顶的值添加到缓冲区中
    luaL_addvalue(b);
static int tconcat (lua_State *L) {
  // 初始化缓冲区
  luaL_Buffer b;
  // 获取第一个参数的长度
  lua_Integer last = aux_getn(L, 1, TAB_R);
  // 获取分隔符
  size_t lsep;
  const char *sep = luaL_optlstring(L, 2, "", &lsep);
  // 获取起始位置和结束位置
  lua_Integer i = luaL_optinteger(L, 3, 1);
  last = luaL_optinteger(L, 4, last);
  // 初始化缓冲区
  luaL_buffinit(L, &b);
  // 循环拼接字符串
  for (; i < last; i++) {
    addfield(L, &b, i);
    luaL_addlstring(&b, sep, lsep);
  }
  // 如果间隔不为空，添加最后一个值
  if (i == last)  
    addfield(L, &b, i);
  // 将结果放入栈中
  luaL_pushresult(&b);
  return 1;
}

/*
** {======================================================
** Pack/unpack
** =======================================================
*/

static int tpack (lua_State *L) {
  int i;
  // 获取参数个数
  int n = lua_gettop(L);  /* number of elements to pack */
  // 创建结果表
  lua_createtable(L, n, 1);  /* create result table */
  // 将结果表放在索引1处
  lua_insert(L, 1);  /* put it at index 1 */
  // 逆序赋值
  for (i = n; i >= 1; i--)  
    lua_seti(L, 1, i);
  // 设置表的长度字段
  lua_pushinteger(L, n);
  lua_setfield(L, 1, "n");  /* t.n = number of elements */
  // 返回结果表
  return 1;  /* return table */
}

static int tunpack (lua_State *L) {
  lua_Unsigned n;
  // 获取起始位置和结束位置
  lua_Integer i = luaL_optinteger(L, 2, 1);
  lua_Integer e = luaL_opt(L, luaL_checkinteger, 3, luaL_len(L, 1));
  // 如果范围为空，返回0
  if (i > e) return 0;  /* empty range */
  // 计算元素个数
  n = (lua_Unsigned)e - i;  
  // 如果元素个数过多，报错
  if (l_unlikely(n >= (unsigned int)INT_MAX  ||
                 !lua_checkstack(L, (int)(++n))))
    return luaL_error(L, "too many results to unpack");
  // 循环将参数压入栈中
  for (; i < e; i++) {  
    lua_geti(L, 1, i);
  }
  // 压入最后一个元素
  lua_geti(L, 1, e);  
  return (int)n;
}

/* }====================================================== */

/*
** {======================================================
** Quicksort
** (based on 'Algorithms in MODULA-3', Robert Sedgewick;
**  Addison-Wesley, 1993.)
** =======================================================
*/

/* type for array indices */
typedef unsigned int IdxT;
/* 产生一个“随机”的无符号整数来随机选择枢轴。仅当'sort'检测到分区结果严重不平衡时才使用此宏。（如果您不想/不需要这种“随机性”，~0是一个不错的选择。） */
#if !defined(l_randomizePivot)        /* { */

#include <time.h>

/* 'e'的大小以'unsigned int'的数量来衡量 */
#define sof(e)        (sizeof(e) / sizeof(unsigned int))

/*
** 使用'time'和'clock'作为“随机性”的来源。因为我们不知道'time_t'和'clock_t'的类型，所以我们不能将它们转换为任何类型而不会出现溢出的风险。安全使用它们的值的方法是将它们复制到一个已知类型的数组中，并使用数组的值。
*/
static unsigned int l_randomizePivot (void) {
  clock_t c = clock();
  time_t t = time(NULL);
  unsigned int buff[sof(c) + sof(t)];
  unsigned int i, rnd = 0;
  memcpy(buff, &c, sof(c) * sizeof(unsigned int));
  memcpy(buff + sof(c), &t, sof(t) * sizeof(unsigned int));
  for (i = 0; i < sof(buff); i++)
    rnd += buff[i];
  return rnd;
}

#endif                    /* } */


/* 大于'RANLIMIT'的数组可能使用随机枢轴 */
#define RANLIMIT    100u


static void set2 (lua_State *L, IdxT i, IdxT j) {
  lua_seti(L, 1, i);
  lua_seti(L, 1, j);
}


/*
** 如果堆栈索引'a'处的值小于索引'b'处的值（根据排序顺序），则返回true。
*/
static int sort_comp (lua_State *L, int a, int b) {
  if (lua_isnil(L, 2))  /* 没有函数？ */
    return lua_compare(L, a, b, LUA_OPLT);  /* a < b */
  else {  /* 函数 */
    int res;
    lua_pushvalue(L, 2);    /* 压入函数 */
    lua_pushvalue(L, a-1);  /* -1以补偿函数 */
    lua_pushvalue(L, b-2);  /* -2以补偿函数和'a' */
    lua_call(L, 2, 1);      /* 调用函数 */
    res = lua_toboolean(L, -1);  /* 获取结果 */
    lua_pop(L, 1);          /* 弹出结果 */
    return res;
  }
}


/*
** 执行分区：枢轴P位于堆栈顶部。
*/
/* 
** precondition: a[lo] <= P == a[up-1] <= a[up],
** 所以只需要对从 lo + 1 到 up - 2 进行分区。
** Pos-condition: a[lo .. i - 1] <= a[i] == P <= a[i + 1 .. up]
** 返回 'i'。
*/
static IdxT partition (lua_State *L, IdxT lo, IdxT up) {
  IdxT i = lo;  /* 在第一次使用之前将会递增 */
  IdxT j = up - 1;  /* 在第一次使用之前将会递减 */
  /* 循环不变式: a[lo .. i] <= P <= a[j .. up] */
  for (;;) {
    /* 下一个循环: 当 a[i] < P 时重复 ++i */
    while ((void)lua_geti(L, 1, ++i), sort_comp(L, -1, -2)) {
      if (l_unlikely(i == up - 1))  /* a[i] < P  但 a[up - 1] == P  ?? */
        luaL_error(L, "invalid order function for sorting");
      lua_pop(L, 1);  /* 移除 a[i] */
    }
    /* 循环结束后, a[i] >= P 并且 a[lo .. i - 1] < P */
    /* 下一个循环: 当 P < a[j] 时重复 --j */
    while ((void)lua_geti(L, 1, --j), sort_comp(L, -3, -1)) {
      if (l_unlikely(j < i))  /* j < i  但  a[j] > P ?? */
        luaL_error(L, "invalid order function for sorting");
      lua_pop(L, 1);  /* 移除 a[j] */
    }
    /* 循环结束后, a[j] <= P 并且 a[j + 1 .. up] >= P */
    if (j < i) {  /* 没有元素错位? */
      /* a[lo .. i - 1] <= P <= a[j + 1 .. i .. up] */
      lua_pop(L, 1);  /* 弹出 a[j] */
      /* 交换中轴元素 (a[up - 1]) 和 a[i] 以满足 pos-condition */
      set2(L, up - 1, i);
      return i;
    }
    /* 否则, 交换 a[i] - a[j] 以恢复不变式并重复 */
    set2(L, i, j);
  }
}

/*
** 在 [lo,up] 的中间(第二到第三个四分位数)选择一个元素
** 由 'rnd' 随机化
*/
static IdxT choosePivot (IdxT lo, IdxT up, unsigned int rnd) {
  IdxT r4 = (up - lo) / 4;  /* 范围/4 */
  IdxT p = rnd % (r4 * 2) + (lo + r4);
  lua_assert(lo + r4 <= p && p <= up - r4);
  return p;
}

/*
** 快速排序算法 (递归函数)
*/
static void auxsort (lua_State *L, IdxT lo, IdxT up,
                                   unsigned int rnd) {
  while (lo < up) {  /* 尾递归循环 */
    IdxT p;  /* Pivot index */  # 定义变量 p 作为中轴索引
    IdxT n;  /* to be used later */  # 定义变量 n 用于稍后使用
    /* sort elements 'lo', 'p', and 'up' */  # 对 'lo'、'p' 和 'up' 元素进行排序
    lua_geti(L, 1, lo);  # 从 Lua 栈中获取索引为 1 的表中索引为 lo 的元素
    lua_geti(L, 1, up);  # 从 Lua 栈中获取索引为 1 的表中索引为 up 的元素
    if (sort_comp(L, -1, -2))  /* a[up] < a[lo]? */  # 如果 a[up] < a[lo]，则返回 true
      set2(L, lo, up);  /* swap a[lo] - a[up] */  # 交换 a[lo] 和 a[up]
    else
      lua_pop(L, 2);  /* remove both values */  # 从 Lua 栈中移除两个值
    if (up - lo == 1)  /* only 2 elements? */  # 如果只有 2 个元素？
      return;  /* already sorted */  # 直接返回，已经排序好了
    if (up - lo < RANLIMIT || rnd == 0)  /* small interval or no randomize? */  # 如果间隔小于 RANLIMIT 或者 rnd 为 0？
      p = (lo + up)/2;  /* middle element is a good pivot */  # 中间元素是一个很好的中轴
    else  /* for larger intervals, it is worth a random pivot */  # 对于更大的间隔，值得使用随机中轴
      p = choosePivot(lo, up, rnd);  # 选择中轴
    lua_geti(L, 1, p);  # 从 Lua 栈中获取索引为 1 的表中索引为 p 的元素
    lua_geti(L, 1, lo);  # 从 Lua 栈中获取索引为 1 的表中索引为 lo 的元素
    if (sort_comp(L, -2, -1))  /* a[p] < a[lo]? */  # 如果 a[p] < a[lo]？
      set2(L, p, lo);  /* swap a[p] - a[lo] */  # 交换 a[p] 和 a[lo]
    else {
      lua_pop(L, 1);  /* remove a[lo] */  # 移除 a[lo]
      lua_geti(L, 1, up);  # 从 Lua 栈中获取索引为 1 的表中索引为 up 的元素
      if (sort_comp(L, -1, -2))  /* a[up] < a[p]? */  # 如果 a[up] < a[p]？
        set2(L, p, up);  /* swap a[up] - a[p] */  # 交换 a[up] 和 a[p]
      else
        lua_pop(L, 2);  # remove both values  # 移除两个值
    }
    if (up - lo == 2)  /* only 3 elements? */  # 如果只有 3 个元素？
      return;  /* already sorted */  # 直接返回，已经排序好了
    lua_geti(L, 1, p);  /* get middle element (Pivot) */  # 获取中间元素（中轴）
    lua_pushvalue(L, -1);  /* push Pivot */  # 将中轴推入 Lua 栈
    lua_geti(L, 1, up - 1);  /* push a[up - 1] */  # 将 a[up - 1] 推入 Lua 栈
    set2(L, p, up - 1);  /* swap Pivot (a[p]) with a[up - 1] */  # 交换中轴（a[p]）和 a[up - 1]
    p = partition(L, lo, up);  # 调用 partition 函数，返回中轴的最终位置
    /* a[lo .. p - 1] <= a[p] == P <= a[p + 1 .. up] */  # a[lo .. p - 1] <= a[p] == P <= a[p + 1 .. up]
    if (p - lo < up - p) {  /* lower interval is smaller? */  # 较小的间隔？
      auxsort(L, lo, p - 1, rnd);  /* call recursively for lower interval */  # 递归调用较小间隔的排序
      n = p - lo;  /* size of smaller interval */  # 较小间隔的大小
      lo = p + 1;  /* tail call for [p + 1 .. up] (upper interval) */  # 对 [p + 1 .. up]（较大间隔）进行尾递归调用
    }
    else {
      auxsort(L, p + 1, up, rnd);  /* call recursively for upper interval */  # 递归调用较大间隔的排序
      n = up - p;  /* size of smaller interval */  # 较小间隔的大小
      up = p - 1;  /* tail call for [lo .. p - 1]  (lower interval) */  # 对 [lo .. p - 1]（较小间隔）进行尾递归调用
    }
    # 如果分区过于不平衡
    if ((up - lo) / 128 > n) 
      # 重新随机选择一个枢轴值
      rnd = l_randomizePivot();  
  }  # 尾递归调用auxsort(L, lo, up, rnd)
/* }====================================================== */

// 定义一个静态函数sort，用于对Lua表进行排序
static int sort (lua_State *L) {
  // 获取表的长度
  lua_Integer n = aux_getn(L, 1, TAB_RW);
  // 如果表的长度大于1
  if (n > 1) {  /* non-trivial interval? */
    // 检查表的长度是否小于INT_MAX
    luaL_argcheck(L, n < INT_MAX, 1, "array too big");
    // 如果第二个参数不是nil
    if (!lua_isnoneornil(L, 2))  /* is there a 2nd argument? */
      // 检查第二个参数是否为函数类型
      luaL_checktype(L, 2, LUA_TFUNCTION);  /* must be a function */
    // 设置栈顶为第二个参数
    lua_settop(L, 2);  /* make sure there are two arguments */
    // 调用auxsort函数对表进行排序
    auxsort(L, 1, (IdxT)n, 0);
  }
  return 0;
}

// 定义一个包含表操作函数的静态常量数组
static const luaL_Reg tab_funcs[] = {
  {"concat", tconcat},
  {"insert", tinsert},
  {"pack", tpack},
  {"unpack", tunpack},
  {"remove", tremove},
  {"move", tmove},
  {"sort", sort},
  {NULL, NULL}
};

// 定义luaopen_table函数，用于在Lua中打开table库
LUAMOD_API int luaopen_table (lua_State *L) {
  // 创建一个新的库，并将表操作函数数组注册到其中
  luaL_newlib(L, tab_funcs);
  return 1;
}
```