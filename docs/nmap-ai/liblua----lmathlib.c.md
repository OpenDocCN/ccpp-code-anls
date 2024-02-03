# `nmap\liblua\lmathlib.c`

```cpp
/*
** $Id: lmathlib.c $
** 标准数学库
** 请参阅 lua.h 中的版权声明
*/

#define lmathlib_c
#define LUA_LIB

#include "lprefix.h"


#include <float.h>
#include <limits.h>
#include <math.h>
#include <stdlib.h>
#include <time.h>

#include "lua.h"

#include "lauxlib.h"
#include "lualib.h"


#undef PI
#define PI    (l_mathop(3.141592653589793238462643383279502884))


static int math_abs (lua_State *L) {
  if (lua_isinteger(L, 1)) {
    lua_Integer n = lua_tointeger(L, 1);
    if (n < 0) n = (lua_Integer)(0u - (lua_Unsigned)n);
    lua_pushinteger(L, n);
  }
  else
    lua_pushnumber(L, l_mathop(fabs)(luaL_checknumber(L, 1)));
  return 1;
}

static int math_sin (lua_State *L) {
  lua_pushnumber(L, l_mathop(sin)(luaL_checknumber(L, 1)));
  return 1;
}

static int math_cos (lua_State *L) {
  lua_pushnumber(L, l_mathop(cos)(luaL_checknumber(L, 1)));
  return 1;
}

static int math_tan (lua_State *L) {
  lua_pushnumber(L, l_mathop(tan)(luaL_checknumber(L, 1)));
  return 1;
}

static int math_asin (lua_State *L) {
  lua_pushnumber(L, l_mathop(asin)(luaL_checknumber(L, 1)));
  return 1;
}

static int math_acos (lua_State *L) {
  lua_pushnumber(L, l_mathop(acos)(luaL_checknumber(L, 1)));
  return 1;
}

static int math_atan (lua_State *L) {
  lua_Number y = luaL_checknumber(L, 1);
  lua_Number x = luaL_optnumber(L, 2, 1);
  lua_pushnumber(L, l_mathop(atan2)(y, x));
  return 1;
}


static int math_toint (lua_State *L) {
  int valid;
  lua_Integer n = lua_tointegerx(L, 1, &valid);
  if (l_likely(valid))
    lua_pushinteger(L, n);
  else {
    luaL_checkany(L, 1);
    luaL_pushfail(L);  /* value is not convertible to integer */
  }
  return 1;
}


static void pushnumint (lua_State *L, lua_Number d) {
  lua_Integer n;
  if (lua_numbertointeger(d, &n))  /* does 'd' fit in an integer? */
    lua_pushinteger(L, n);  /* result is integer */
  else
    lua_pushnumber(L, d);  /* result is float */
}


static int math_floor (lua_State *L) {
  if (lua_isinteger(L, 1))

... (此处代码太长，无法完全显示)
    # 将栈顶元素设置为参数1，即将参数1作为整数返回
    lua_settop(L, 1);  /* integer is its own floor */
  else:
    # 如果参数1不是整数，则将其向下取整并压入栈中
    lua_Number d = l_mathop(floor)(luaL_checknumber(L, 1));
    pushnumint(L, d);
  # 返回1，表示成功
  return 1;
# 计算大于或等于给定数字的最小整数
static int math_ceil (lua_State *L) {
  # 如果参数是整数，则将栈顶设置为参数本身，整数的最小整数就是其本身
  if (lua_isinteger(L, 1))
    lua_settop(L, 1);  /* integer is its own ceil */
  else {
    # 否则，将参数转换为 lua_Number 类型，然后向栈中推入其最小整数
    lua_Number d = l_mathop(ceil)(luaL_checknumber(L, 1));
    pushnumint(L, d);
  }
  return 1;
}

# 计算给定两个数的余数
static int math_fmod (lua_State *L) {
  # 如果两个参数都是整数
  if (lua_isinteger(L, 1) && lua_isinteger(L, 2)) {
    # 将第二个参数转换为 lua_Integer 类型
    lua_Integer d = lua_tointeger(L, 2);
    # 处理特殊情况：-1 或 0
    if ((lua_Unsigned)d + 1u <= 1u) {  /* special cases: -1 or 0 */
      luaL_argcheck(L, d != 0, 2, "zero");
      lua_pushinteger(L, 0);  /* avoid overflow with 0x80000... / -1 */
    }
    else
      # 否则，将第一个参数对第二个参数取余数，推入栈中
      lua_pushinteger(L, lua_tointeger(L, 1) % d);
  }
  else
    # 如果参数不全是整数，则将两个参数转换为 lua_Number 类型，然后计算余数，推入栈中
    lua_pushnumber(L, l_mathop(fmod)(luaL_checknumber(L, 1),
                                     luaL_checknumber(L, 2)));
  return 1;
}

# 计算给定数的整数部分和小数部分
static int math_modf (lua_State *L) {
  # 如果参数是整数，则将栈顶设置为参数本身，整数的整数部分就是其本身，小数部分为 0
  if (lua_isinteger(L ,1)) {
    lua_settop(L, 1);  /* number is its own integer part */
    lua_pushnumber(L, 0);  /* no fractional part */
  }
  else {
    # 否则，将参数转换为 lua_Number 类型
    lua_Number n = luaL_checknumber(L, 1);
    # 计算整数部分（向零舍入）
    lua_Number ip = (n < 0) ? l_mathop(ceil)(n) : l_mathop(floor)(n);
    pushnumint(L, ip);
    # 计算小数部分
    lua_pushnumber(L, (n == ip) ? l_mathop(0.0) : (n - ip));
  }
  return 2;
}

# 计算给定数的平方根
static int math_sqrt (lua_State *L) {
  # 将参数转换为 lua_Number 类型，然后计算平方根，推入栈中
  lua_pushnumber(L, l_mathop(sqrt)(luaL_checknumber(L, 1)));
  return 1;
}

# 比较两个整数的无符号小于关系
static int math_ult (lua_State *L) {
  # 将参数转换为 lua_Integer 类型，然后比较大小，推入栈中
  lua_Integer a = luaL_checkinteger(L, 1);
  lua_Integer b = luaL_checkinteger(L, 2);
  lua_pushboolean(L, (lua_Unsigned)a < (lua_Unsigned)b);
  return 1;
}

# 计算给定数的自然对数
static int math_log (lua_State *L) {
  # 将参数转换为 lua_Number 类型
  lua_Number x = luaL_checknumber(L, 1);
  lua_Number res;
  # 如果第二个参数不存在或为 nil，则计算 x 的自然对数
  if (lua_isnoneornil(L, 2))
    res = l_mathop(log)(x);
  else {
    # 否则，将第二个参数转换为 lua_Number 类型，然后计算以该参数为底的对数
    lua_Number base = luaL_checknumber(L, 2);
#if !defined(LUA_USE_C89)
    # 如果 base 等于 2.0，则计算以 2 为底的对数
    if (base == l_mathop(2.0))
      res = l_mathop(log2)(x);
    # 如果 base 不等于 2.0，则执行下面的代码
    else
#endif
    // 如果基数为10，则计算以10为底的对数
    if (base == l_mathop(10.0))
      res = l_mathop(log10)(x);
    // 否则计算以指定基数为底的对数
    else
      res = l_mathop(log)(x)/l_mathop(log)(base);
  }
  // 将结果压入栈中
  lua_pushnumber(L, res);
  // 返回1表示成功
  return 1;
}

// 计算e的x次方
static int math_exp (lua_State *L) {
  lua_pushnumber(L, l_mathop(exp)(luaL_checknumber(L, 1)));
  return 1;
}

// 将角度转换为弧度
static int math_deg (lua_State *L) {
  lua_pushnumber(L, luaL_checknumber(L, 1) * (l_mathop(180.0) / PI));
  return 1;
}

// 将弧度转换为角度
static int math_rad (lua_State *L) {
  lua_pushnumber(L, luaL_checknumber(L, 1) * (PI / l_mathop(180.0)));
  return 1;
}


// 返回参数中的最小值
static int math_min (lua_State *L) {
  int n = lua_gettop(L);  /* 参数个数 */
  int imin = 1;  /* 当前最小值的索引 */
  int i;
  luaL_argcheck(L, n >= 1, 1, "value expected");
  for (i = 2; i <= n; i++) {
    if (lua_compare(L, i, imin, LUA_OPLT))
      imin = i;
  }
  lua_pushvalue(L, imin);
  return 1;
}


// 返回参数中的最大值
static int math_max (lua_State *L) {
  int n = lua_gettop(L);  /* 参数个数 */
  int imax = 1;  /* 当前最大值的索引 */
  int i;
  luaL_argcheck(L, n >= 1, 1, "value expected");
  for (i = 2; i <= n; i++) {
    if (lua_compare(L, imax, i, LUA_OPLT))
      imax = i;
  }
  lua_pushvalue(L, imax);
  return 1;
}


// 返回参数的类型
static int math_type (lua_State *L) {
  if (lua_type(L, 1) == LUA_TNUMBER)
    lua_pushstring(L, (lua_isinteger(L, 1)) ? "integer" : "float");
  else {
    luaL_checkany(L, 1);
    luaL_pushfail(L);
  }
  return 1;
}



/*
** {==================================================================
** 基于'xoshiro256**'的伪随机数生成器。
** ===================================================================
*/

/* 浮点数尾数中的二进制位数 */
#define FIGS    l_floatatt(MANT_DIG)

#if FIGS > 64
/* 只有64位随机位；使用它们全部 */
#undef FIGS
#define FIGS    64
#endif


/*
** LUA_RAND32 强制在PRN生成器的实现中使用32位整数（主要用于测试）。
*/
#if !defined(LUA_RAND32) && !defined(Rand64)
/* 尝试找到至少有 64 位的整数类型 */

#if (ULONG_MAX >> 31 >> 31) >= 3

/* 'long' 至少有 64 位 */
#define Rand64        unsigned long

#elif !defined(LUA_USE_C89) && defined(LLONG_MAX)

/* 存在 'long long' 类型（至少有 64 位） */
#define Rand64        unsigned long long

#elif (LUA_MAXUNSIGNED >> 31 >> 31) >= 3

/* 'lua_Integer' 至少有 64 位 */
#define Rand64        lua_Unsigned

#endif

#endif


#if defined(Rand64)  /* { */

/*
** 标准实现，使用 64 位整数。
** 如果 'Rand64' 有超过 64 位，额外的位不会影响前 64 位，除了右移操作。此外，最终结果必须舍弃额外的位。
*/

/* 在需要时避免使用额外的位 */
#define trim64(x)    ((x) & 0xffffffffffffffffu)


/* 将 'x' 左移 'n' 位 */
static Rand64 rotl (Rand64 x, int n) {
  return (x << n) | (trim64(x) >> (64 - n));
}

static Rand64 nextrand (Rand64 *state) {
  Rand64 state0 = state[0];
  Rand64 state1 = state[1];
  Rand64 state2 = state[2] ^ state0;
  Rand64 state3 = state[3] ^ state1;
  Rand64 res = rotl(state1 * 5, 7) * 9;
  state[0] = state0 ^ state3;
  state[1] = state1 ^ state2;
  state[2] = state2 ^ (state1 << 17);
  state[3] = rotl(state3, 45);
  return res;
}


/* 必须小心不要将东西左移超过 63 位 */


/*
** 将随机整数的位转换为区间 [0,1) 内的浮点数，从随机无符号整数中获取高 FIG 位并将其转换为浮点数。
*/

/* 必须丢弃额外的 (64 - FIGS) 位 */
#define shift64_FIG    (64 - FIGS)

/* 缩放到 [0, 1) 区间，乘以 scaleFIG = 2^(-FIGS) */
#define scaleFIG    (l_mathop(0.5) / ((Rand64)1 << (FIGS - 1)))

static lua_Number I2d (Rand64 x) {
  return (lua_Number)(trim64(x) >> shift64_FIG) * scaleFIG;
}

/* 将 'Rand64' 转换为 'lua_Unsigned' */
#define I2UInt(x)    ((lua_Unsigned)trim64(x))

/* 将 'lua_Unsigned' 转换为 'Rand64' */
#define Int2I(x)    ((Rand64)(x))  # 定义一个宏，将整数转换为Rand64类型

#else    /* no 'Rand64'   }{ */

/* get an integer with at least 32 bits */
#if LUAI_IS32INT
typedef unsigned int lu_int32;  # 如果LUAI_IS32INT为真，则定义无符号整型lu_int32为32位
#else
typedef unsigned long lu_int32;  # 否则定义为无符号长整型lu_int32
#endif


/*
** Use two 32-bit integers to represent a 64-bit quantity.
*/
typedef struct Rand64 {  # 定义一个结构体Rand64，使用两个32位整数表示64位数量
  lu_int32 h;  /* higher half */  # 高32位
  lu_int32 l;  /* lower half */  # 低32位
} Rand64;


/*
** If 'lu_int32' has more than 32 bits, the extra bits do not interfere
** with the 32 initial bits, except in a right shift and comparisons.
** Moreover, the final result has to discard the extra bits.
*/

/* avoid using extra bits when needed */
#define trim32(x)    ((x) & 0xffffffffu)  # 定义一个宏，用于去除多余的位数，只保留32位


/*
** basic operations on 'Rand64' values
*/

/* build a new Rand64 value */
static Rand64 packI (lu_int32 h, lu_int32 l) {  # 定义一个静态函数，用于构建一个新的Rand64值
  Rand64 result;
  result.h = h;  # 设置高32位
  result.l = l;  # 设置低32位
  return result;  # 返回构建的Rand64值
}

/* return i << n */
static Rand64 Ishl (Rand64 i, int n) {  # 定义一个静态函数，用于将i左移n位
  lua_assert(n > 0 && n < 32);  # 断言，确保n大于0且小于32
  return packI((i.h << n) | (trim32(i.l) >> (32 - n)), i.l << n);  # 返回左移后的Rand64值
}

/* i1 ^= i2 */
static void Ixor (Rand64 *i1, Rand64 i2) {  # 定义一个静态函数，用于对i1进行异或操作
  i1->h ^= i2.h;  # 对高32位进行异或操作
  i1->l ^= i2.l;  # 对低32位进行异或操作
}

/* return i1 + i2 */
static Rand64 Iadd (Rand64 i1, Rand64 i2) {  # 定义一个静态函数，用于返回i1和i2的和
  Rand64 result = packI(i1.h + i2.h, i1.l + i2.l);  # 计算和并构建新的Rand64值
  if (trim32(result.l) < trim32(i1.l))  /* carry? */  # 如果低32位的结果小于i1的低32位，需要进位
    result.h++;  # 需要进位
  return result;  # 返回结果
}

/* return i * 5 */
static Rand64 times5 (Rand64 i) {  # 定义一个静态函数，用于返回i乘以5的结果
  return Iadd(Ishl(i, 2), i);  /* i * 5 == (i << 2) + i */  # 返回i左移2位加上i的结果
}

/* return i * 9 */
static Rand64 times9 (Rand64 i) {  # 定义一个静态函数，用于返回i乘以9的结果
  return Iadd(Ishl(i, 3), i);  /* i * 9 == (i << 3) + i */  # 返回i左移3位加上i的结果
}

/* return 'i' rotated left 'n' bits */
static Rand64 rotl (Rand64 i, int n) {  # 定义一个静态函数，用于将i左旋转n位
  lua_assert(n > 0 && n < 32);  # 断言，确保n大于0且小于32
  return packI((i.h << n) | (trim32(i.l) >> (32 - n)),  # 返回左旋转后的Rand64值
               (trim32(i.h) >> (32 - n)) | (i.l << n));
}

/* for offsets larger than 32, rotate right by 64 - offset */
/*
** 对 Rand64 值进行左循环移位
** 参数 i: Rand64 值
** 参数 n: 移位位数
** 返回值: 左循环移位后的 Rand64 值
*/
static Rand64 rotl1 (Rand64 i, int n) {
  // 断言，确保 n 的取值范围在 32 到 64 之间
  lua_assert(n > 32 && n < 64);
  // 计算实际的移位位数
  n = 64 - n;
  // 返回左循环移位后的 Rand64 值
  return packI((trim32(i.h) >> n) | (i.l << (32 - n)),
               (i.h << (32 - n)) | (trim32(i.l) >> n));
}

/*
** 在 Rand64 值上实现 'xoshiro256**' 算法
** 参数 state: Rand64 值的数组
** 返回值: Rand64 值
*/
static Rand64 nextrand (Rand64 *state) {
  // 计算 res
  Rand64 res = times9(rotl(times5(state[1]), 7));
  // 计算 t
  Rand64 t = Ishl(state[1], 17);
  // 对 state[2] 和 state[0] 进行异或操作
  Ixor(&state[2], state[0]);
  // 对 state[3] 和 state[1] 进行异或操作
  Ixor(&state[3], state[1]);
  // 对 state[1] 和 state[2] 进行异或操作
  Ixor(&state[1], state[2]);
  // 对 state[0] 和 state[3] 进行异或操作
  Ixor(&state[0], state[3]);
  // 对 state[2] 和 t 进行异或操作
  Ixor(&state[2], t);
  // 对 state[3] 进行左循环移位
  state[3] = rotl1(state[3], 45);
  // 返回 res
  return res;
}

/*
** 将 Rand64 转换为浮点数
** 参数 x: Rand64 值
** 返回值: 浮点数
*/
static lua_Number I2d (Rand64 x) {
  // 如果 FIGS 小于等于 32
  #if FIGS <= 32
  // 计算 h
  lua_Number h = (lua_Number)(trim32(x.h) >> (32 - FIGS));
  // 返回 h 乘以 scaleFIG
  return h * scaleFIG;
  // 如果 FIGS 大于 32 且小于等于 64
  #else
  // 计算 h
  lua_Number h = (lua_Number)trim32(x.h) * shiftHI;
  // 计算 l
  lua_Number l = (lua_Number)(trim32(x.l) >> shiftLOW);
  // 返回 (h + l) 乘以 scaleFIG
  return (h + l) * scaleFIG;
  #endif
}

/*
** 将 Rand64 转换为 lua_Unsigned
** 参数 x: Rand64 值
** 返回值: lua_Unsigned 值
*/
static lua_Unsigned I2UInt (Rand64 x) {
  // 返回转换后的 lua_Unsigned 值
  return ((lua_Unsigned)trim32(x.h) << 31 << 1) | (lua_Unsigned)trim32(x.l);
}
static Rand64 Int2I (lua_Unsigned n) {
  return packI((lu_int32)(n >> 31 >> 1), (lu_int32)n);
}

#endif  /* } */


/*
** A state uses four 'Rand64' values.
*/
typedef struct {
  Rand64 s[4];
} RanState;


/*
** Project the random integer 'ran' into the interval [0, n].
** Because 'ran' has 2^B possible values, the projection can only be
** uniform when the size of the interval is a power of 2 (exact
** division). Otherwise, to get a uniform projection into [0, n], we
** first compute 'lim', the smallest Mersenne number not smaller than
** 'n'. We then project 'ran' into the interval [0, lim].  If the result
** is inside [0, n], we are done. Otherwise, we try with another 'ran',
** until we have a result inside the interval.
*/
static lua_Unsigned project (lua_Unsigned ran, lua_Unsigned n,
                             RanState *state) {
  if ((n & (n + 1)) == 0)  /* is 'n + 1' a power of 2? */
    return ran & n;  /* no bias */
  else {
    lua_Unsigned lim = n;
    /* compute the smallest (2^b - 1) not smaller than 'n' */
    lim |= (lim >> 1);
    lim |= (lim >> 2);
    lim |= (lim >> 4);
    lim |= (lim >> 8);
    lim |= (lim >> 16);
#if (LUA_MAXUNSIGNED >> 31) >= 3
    lim |= (lim >> 32);  /* integer type has more than 32 bits */
#endif
    lua_assert((lim & (lim + 1)) == 0  /* 'lim + 1' is a power of 2, */
      && lim >= n  /* not smaller than 'n', */
      && (lim >> 1) < n);  /* and it is the smallest one */
    while ((ran &= lim) > n)  /* project 'ran' into [0..lim] */
      ran = I2UInt(nextrand(state->s));  /* not inside [0..n]? try again */
    return ran;
  }
}


static int math_random (lua_State *L) {
  lua_Integer low, up;
  lua_Unsigned p;
  RanState *state = (RanState *)lua_touserdata(L, lua_upvalueindex(1));
  Rand64 rv = nextrand(state->s);  /* next pseudo-random value */
  switch (lua_gettop(L)) {  /* check number of arguments */
    case 0: {  /* no arguments */
      lua_pushnumber(L, I2d(rv));  /* float between 0 and 1 */
      return 1;
    }
    case 1: {  /* 只有上限 */
      low = 1;  // 下限为1
      up = luaL_checkinteger(L, 1);  // 获取参数1作为上限
      if (up == 0) {  /* 参数为单个0？ */
        lua_pushinteger(L, I2UInt(rv));  // 将完整的随机整数压入栈
        return 1;
      }
      break;
    }
    case 2: {  /* 下限和上限 */
      low = luaL_checkinteger(L, 1);  // 获取参数1作为下限
      up = luaL_checkinteger(L, 2);  // 获取参数2作为上限
      break;
    }
    default: return luaL_error(L, "wrong number of arguments");  // 参数数量错误
  }
  /* 在区间[low, up]内生成随机整数 */
  luaL_argcheck(L, low <= up, 1, "interval is empty");  // 检查区间是否为空
  /* 将随机整数投影到区间[0, up - low]内 */
  p = project(I2UInt(rv), (lua_Unsigned)up - (lua_Unsigned)low, state);  // 将随机整数投影到指定区间
  lua_pushinteger(L, p + (lua_Unsigned)low);  // 将投影后的随机整数压入栈
  return 1;
/* 
** 设置随机数种子
** 参数：L - Lua 状态机指针，state - 随机数状态结构体指针，n1 - 无符号整数种子1，n2 - 无符号整数种子2
*/
static void setseed (lua_State *L, Rand64 *state,
                     lua_Unsigned n1, lua_Unsigned n2) {
  int i;
  state[0] = Int2I(n1);
  state[1] = Int2I(0xff);  /* 避免零状态 */
  state[2] = Int2I(n2);
  state[3] = Int2I(0);
  for (i = 0; i < 16; i++)
    nextrand(state);  /* 丢弃初始值以“扩散”种子 */
  lua_pushinteger(L, n1);
  lua_pushinteger(L, n2);
}

/*
** 设置“随机”种子。为了获得一些随机性，使用当前时间和 'L' 的地址（以防机器进行地址空间布局随机化）。
*/
static void randseed (lua_State *L, RanState *state) {
  lua_Unsigned seed1 = (lua_Unsigned)time(NULL);
  lua_Unsigned seed2 = (lua_Unsigned)(size_t)L;
  setseed(L, state->s, seed1, seed2);
}

/*
** math_randomseed 函数
** 参数：L - Lua 状态机指针
** 返回值：2（返回种子）
*/
static int math_randomseed (lua_State *L) {
  RanState *state = (RanState *)lua_touserdata(L, lua_upvalueindex(1));
  if (lua_isnone(L, 1)) {
    randseed(L, state);
  }
  else {
    lua_Integer n1 = luaL_checkinteger(L, 1);
    lua_Integer n2 = luaL_optinteger(L, 2, 0);
    setseed(L, state->s, n1, n2);
  }
  return 2;  /* 返回种子 */
}

/*
** 随机函数数组
*/
static const luaL_Reg randfuncs[] = {
  {"random", math_random},
  {"randomseed", math_randomseed},
  {NULL, NULL}
};

/*
** 注册随机函数并初始化它们的状态
** 参数：L - Lua 状态机指针
*/
static void setrandfunc (lua_State *L) {
  RanState *state = (RanState *)lua_newuserdatauv(L, sizeof(RanState), 0);
  randseed(L, state);  /* 使用“随机”种子进行初始化 */
  lua_pop(L, 2);  /* 移除推送的种子 */
  luaL_setfuncs(L, randfuncs, 1);
}

/*
** Deprecated functions (for compatibility only)
*/
#if defined(LUA_COMPAT_MATHLIB)

/*
** 双曲余弦函数
** 参数：L - Lua 状态机指针
** 返回值：1
*/
static int math_cosh (lua_State *L) {
  lua_pushnumber(L, l_mathop(cosh)(luaL_checknumber(L, 1)));
  return 1;
}
static int math_sinh (lua_State *L) {
  // 将双曲正弦函数应用于栈顶的数字，并将结果推入栈顶
  lua_pushnumber(L, l_mathop(sinh)(luaL_checknumber(L, 1)));
  return 1; // 返回 1 个结果
}

static int math_tanh (lua_State *L) {
  // 将双曲正切函数应用于栈顶的数字，并将结果推入栈顶
  lua_pushnumber(L, l_mathop(tanh)(luaL_checknumber(L, 1)));
  return 1; // 返回 1 个结果
}

static int math_pow (lua_State *L) {
  lua_Number x = luaL_checknumber(L, 1); // 获取栈顶的第一个数字
  lua_Number y = luaL_checknumber(L, 2); // 获取栈顶的第二个数字
  // 将 x 的 y 次幂的结果推入栈顶
  lua_pushnumber(L, l_mathop(pow)(x, y));
  return 1; // 返回 1 个结果
}

static int math_frexp (lua_State *L) {
  int e;
  // 将栈顶的数字进行 frexp 分解，并将结果推入栈顶
  lua_pushnumber(L, l_mathop(frexp)(luaL_checknumber(L, 1), &e));
  lua_pushinteger(L, e); // 将 e 推入栈顶
  return 2; // 返回 2 个结果
}

static int math_ldexp (lua_State *L) {
  lua_Number x = luaL_checknumber(L, 1); // 获取栈顶的第一个数字
  int ep = (int)luaL_checkinteger(L, 2); // 获取栈顶的第二个整数
  // 将 x 乘以 2 的 ep 次幂的结果推入栈顶
  lua_pushnumber(L, l_mathop(ldexp)(x, ep));
  return 1; // 返回 1 个结果
}

static int math_log10 (lua_State *L) {
  // 将以 10 为底的对数应用于栈顶的数字，并将结果推入栈顶
  lua_pushnumber(L, l_mathop(log10)(luaL_checknumber(L, 1)));
  return 1; // 返回 1 个结果
}

#endif
/* }================================================================== */

static const luaL_Reg mathlib[] = {
  // 数学库函数列表
  {"abs",   math_abs},
  {"acos",  math_acos},
  {"asin",  math_asin},
  {"atan",  math_atan},
  {"ceil",  math_ceil},
  {"cos",   math_cos},
  {"deg",   math_deg},
  {"exp",   math_exp},
  {"tointeger", math_toint},
  {"floor", math_floor},
  {"fmod",   math_fmod},
  {"ult",   math_ult},
  {"log",   math_log},
  {"max",   math_max},
  {"min",   math_min},
  {"modf",   math_modf},
  {"rad",   math_rad},
  {"sin",   math_sin},
  {"sqrt",  math_sqrt},
  {"tan",   math_tan},
  {"type", math_type},
#if defined(LUA_COMPAT_MATHLIB)
  {"atan2", math_atan},
  {"cosh",   math_cosh},
  {"sinh",   math_sinh},
  {"tanh",   math_tanh},
  {"pow",   math_pow},
  {"frexp", math_frexp},
  {"ldexp", math_ldexp},
  {"log10", math_log10},
#endif
  /* placeholders */
  {"random", NULL},
  {"randomseed", NULL},
  {"pi", NULL},
  {"huge", NULL},
  {"maxinteger", NULL},
  {"mininteger", NULL},
  {NULL, NULL}
};

/*
** 打开数学库
*/
# 使用 Lua API 打开 math 模块
LUAMOD_API int luaopen_math (lua_State *L) {
  # 创建一个新的库，将 mathlib 中的函数注册到其中
  luaL_newlib(L, mathlib);
  # 将常数 PI 压入栈顶
  lua_pushnumber(L, PI);
  # 设置栈顶的值为字段 "pi"，并弹出栈顶的值
  lua_setfield(L, -2, "pi");
  # 将常数 HUGE_VAL 压入栈顶
  lua_pushnumber(L, (lua_Number)HUGE_VAL);
  # 设置栈顶的值为字段 "huge"，并弹出栈顶的值
  lua_setfield(L, -2, "huge");
  # 将常数 LUA_MAXINTEGER 压入栈顶
  lua_pushinteger(L, LUA_MAXINTEGER);
  # 设置栈顶的值为字段 "maxinteger"，并弹出栈顶的值
  lua_setfield(L, -2, "maxinteger");
  # 将常数 LUA_MININTEGER 压入栈顶
  lua_pushinteger(L, LUA_MININTEGER);
  # 设置栈顶的值为字段 "mininteger"，并弹出栈顶的值
  lua_setfield(L, -2, "mininteger");
  # 设置随机函数
  setrandfunc(L);
  # 返回 1，表示成功加载 math 模块
  return 1;
}
```