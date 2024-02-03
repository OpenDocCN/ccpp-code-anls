# `nmap\liblua\llimits.h`

```cpp
# 定义了一些限制、基本类型和一些其他“与安装相关”的定义
# 包含了一些标准库头文件
#ifndef llimits_h
#define llimits_h

#include <limits.h>
#include <stddef.h>
#include "lua.h"

# 'lu_mem' 和 'l_mem' 是足够大的无符号/有符号整数，用于计算 Lua 使用的总内存（以字节为单位）
# 通常情况下，'size_t' 和 'ptrdiff_t' 应该可以工作，但我们在 16 位机器上使用 'long'
#if defined(LUAI_MEM)        /* { external definitions? */
typedef LUAI_UMEM lu_mem;
typedef LUAI_MEM l_mem;
#elif LUAI_IS32INT    /* }{ */
typedef size_t lu_mem;
typedef ptrdiff_t l_mem;
#else  /* 16-bit ints */    /* }{ */
typedef unsigned long lu_mem;
typedef long l_mem;
#endif                /* } */

# 用作小自然数的字符（以便将 'char' 保留给字符）
typedef unsigned char lu_byte;
typedef signed char ls_byte;

# size_t 的最大值
#define MAX_SIZET    ((size_t)(~(size_t)0))

# Lua 可见的最大大小（必须能够表示为 lua_Integer）
#define MAX_SIZE    (sizeof(size_t) < sizeof(lua_Integer) ? MAX_SIZET \
                          : (size_t)(LUA_MAXINTEGER))

# lu_mem 的最大值
#define MAX_LUMEM    ((lu_mem)(~(lu_mem)0))

# l_mem 的最大值
#define MAX_LMEM    ((l_mem)(MAX_LUMEM >> 1))

# int 的最大值
#define MAX_INT        INT_MAX  /* maximum value of an int */

# 整数类型 't' 的最大有符号值的对数的底
# （即，最大 'n'，使得 '2^n' 可以适应给定的有符号类型）
#define log2maxs(t)    (sizeof(t) * 8 - 2)

# 测试无符号值是否为 2 的幂（或零）
#define ispow2(x)    (((x) & ((x) - 1)) == 0)

# 字面字符串的字符数（不包括结尾的 \0）
#define LL(x)   (sizeof(x)/sizeof(char) - 1)

# 将指针转换为无符号整数：
# 这仅用于哈希；如果整数无法容纳整个指针值，那么就没有问题
#define point2uint(p)    ((unsigned int)((size_t)(p) & UINT_MAX))
/* 定义 lua_Number 和 lua_Integer 的 'usual argument conversions' 类型 */
typedef LUAI_UACNUMBER l_uacNumber;
typedef LUAI_UACINT l_uacInt;

/*
** 用于内部调试的内部断言
*/
#if defined LUAI_ASSERT
#undef NDEBUG
#include <assert.h>
#define lua_assert(c)           assert(c)
#endif

#if defined(lua_assert)
#define check_exp(c,e)        (lua_assert(c), (e))
/* 避免条件过长的问题 */
#define lua_longassert(c)    ((c) ? (void)0 : lua_assert(0))
#else
#define lua_assert(c)        ((void)0)
#define check_exp(c,e)        (e)
#define lua_longassert(c)    ((void)0)
#endif

/*
** 用于检查 API 调用的断言
*/
#if !defined(luai_apicheck)
#define luai_apicheck(l,e)    ((void)l, lua_assert(e))
#endif

#define api_check(l,e,msg)    luai_apicheck(l,(e) && msg)

/* 用于避免关于未使用变量的警告 */
#if !defined(UNUSED)
#define UNUSED(x)    ((void)(x))
#endif

/* 类型转换（宏用于突出代码中的转换） */
#define cast(t, exp)    ((t)(exp))

#define cast_void(i)    cast(void, (i))
#define cast_voidp(i)    cast(void *, (i))
#define cast_num(i)    cast(lua_Number, (i))
#define cast_int(i)    cast(int, (i))
#define cast_uint(i)    cast(unsigned int, (i))
#define cast_byte(i)    cast(lu_byte, (i))
#define cast_uchar(i)    cast(unsigned char, (i))
#define cast_char(i)    cast(char, (i))
#define cast_charp(i)    cast(char *, (i))
#define cast_sizet(i)    cast(size_t, (i))

/* 将有符号的 lua_Integer 转换为 lua_Unsigned */
#if !defined(l_castS2U)
#define l_castS2U(i)    ((lua_Unsigned)(i))
#endif

/*
** 将 lua_Unsigned 转换为有符号的 lua_Integer；这种转换不是严格的 ISO C，但是补码架构应该可以正常工作。
*/
#if !defined(l_castU2S)
#define l_castU2S(i)    ((lua_Integer)(i))
#endif

/*
** 非返回类型
*/
#if !defined(l_noret)

#if defined(__GNUC__)
#define l_noret        void __attribute__((noreturn))
#elif defined(_MSC_VER) && _MSC_VER >= 1200
// 定义了一个宏，用于声明一个不返回值的函数
#define l_noret        void __declspec(noreturn)
#else
#define l_noret        void
#endif

#endif


/*
** 内联函数
*/
#if !defined(LUA_USE_C89)
// 如果不是 C89 标准，则使用 inline 关键字声明内联函数
#define l_inline    inline
// 如果是 GNU 编译器，则使用 __inline__ 关键字声明内联函数
#elif defined(__GNUC__)
#define l_inline    __inline__
// 否则为空
#else
#define l_inline    /* empty */
#endif

// 声明一个静态内联函数
#define l_sinline    static l_inline


/*
** 虚拟机指令的类型；
** 必须是一个至少 4 字节的无符号整数（详细信息见 lopcodes.h）
*/
#if LUAI_IS32INT
// 如果 LUAI_IS32INT 定义了，则使用 unsigned int 类型
typedef unsigned int l_uint32;
#else
// 否则使用 unsigned long 类型
typedef unsigned long l_uint32;
#endif

// 定义了一个类型别名 Instruction，类型为 l_uint32
typedef l_uint32 Instruction;



/*
** 短字符串的最大长度，即内部化的字符串。（不能小于保留字或元方法的标签，因为这些字符串必须内部化；
** #("function") = 8, #("__newindex") = 10.）
*/
#if !defined(LUAI_MAXSHORTLEN)
// 如果 LUAI_MAXSHORTLEN 未定义，则设置为 40
#define LUAI_MAXSHORTLEN    40
#endif


/*
** 字符串表的初始大小（必须是 2 的幂）。
** Lua 核心本身注册了大约 50 个字符串（保留字 + 元事件键 + 几个其他字符串）。
** 库通常会添加几十个以上的字符串。
*/
#if !defined(MINSTRTABSIZE)
// 如果 MINSTRTABSIZE 未定义，则设置为 128
#define MINSTRTABSIZE    128
#endif


/*
** API 中字符串缓存的大小。'N' 是集合的数量（最好是一个素数），"M" 是每个集合的大小（M == 1 表示直接缓存）。
*/
#if !defined(STRCACHE_N)
// 如果 STRCACHE_N 未定义，则设置为 53；如果 STRCACHE_M 未定义，则设置为 2
#define STRCACHE_N        53
#define STRCACHE_M        2
#endif


/* 字符串缓冲区的最小大小 */
#if !defined(LUA_MINBUFFER)
// 如果 LUA_MINBUFFER 未定义，则设置为 32
#define LUA_MINBUFFER    32
#endif


/*
** 嵌套 C 调用、语法嵌套非终结符和其他通过 C 递归实现的功能的最大深度。
** （值必须适合 16 位无符号整数。它还必须与 C 栈的大小兼容。）
*/
#if !defined(LUAI_MAXCCALLS)
// 如果 LUAI_MAXCCALLS 未定义，则设置为 200
#define LUAI_MAXCCALLS        200
#endif


/*
** 每当程序进入 Lua 核心时执行的宏（'lua_lock'）和离开核心时执行的宏（'lua_unlock'）
*/
#if !defined(lua_lock)
# 定义宏，用于在 Lua 函数执行期间的可中断点上执行
#define lua_lock(L)    ((void) 0)
#define lua_unlock(L)    ((void) 0)
#endif

# 如果未定义 luai_threadyield 宏，则定义 luai_threadyield 宏
# 该宏在 Lua 函数中的可中断点处执行
#if !defined(luai_threadyield)
#define luai_threadyield(L)    {lua_unlock(L); lua_lock(L);}
#endif

# 如果未定义 luai_userstateopen 宏，则定义 luai_userstateopen 宏
# 该宏在创建/删除/恢复/暂停线程时执行用户特定操作
#if !defined(luai_userstateopen)
#define luai_userstateopen(L)        ((void)L)
#endif

# 如果未定义 luai_userstateclose 宏，则定义 luai_userstateclose 宏
# 该宏在关闭用户状态时执行用户特定操作
#if !defined(luai_userstateclose)
#define luai_userstateclose(L)        ((void)L)
#endif

# 如果未定义 luai_userstatethread 宏，则定义 luai_userstatethread 宏
# 该宏在创建/删除/恢复/暂停线程时执行用户特定操作
#if !defined(luai_userstatethread)
#define luai_userstatethread(L,L1)    ((void)L)
#endif

# 如果未定义 luai_userstatefree 宏，则定义 luai_userstatefree 宏
# 该宏在释放用户状态时执行用户特定操作
#if !defined(luai_userstatefree)
#define luai_userstatefree(L,L1)    ((void)L)
#endif

# 如果未定义 luai_userstateresume 宏，则定义 luai_userstateresume 宏
# 该宏在恢复线程时执行用户特定操作
#if !defined(luai_userstateresume)
#define luai_userstateresume(L,n)    ((void)L)
#endif

# 如果未定义 luai_userstateyield 宏，则定义 luai_userstateyield 宏
# 该宏在暂停线程时执行用户特定操作
#if !defined(luai_userstateyield)
#define luai_userstateyield(L,n)    ((void)L)
#endif

# 定义 luai_numidiv 宏，用于整数除法
#if !defined(luai_numidiv)
#define luai_numidiv(L,a,b)     ((void)L, l_floor(luai_numdiv(L,a,b)))
#endif

# 定义 luai_numdiv 宏，用于浮点数除法
#if !defined(luai_numdiv)
#define luai_numdiv(L,a,b)      ((a)/(b))
#endif

# 定义 luai_nummod 宏，用于计算取模
#if !defined(luai_nummod)
#define luai_nummod(L,a,b,m)  \
  { (void)L; (m) = l_mathop(fmod)(a,b); \
    if (((m) > 0) ? (b) < 0 : ((m) < 0 && (b) > 0)) (m) += (b); }
#endif
/* exponentiation */
#if !defined(luai_numpow)
#define luai_numpow(L,a,b)  \
  ((void)L, (b == 2) ? (a)*(a) : l_mathop(pow)(a,b))
#endif

/* the others are quite standard operations */
#if !defined(luai_numadd)
#define luai_numadd(L,a,b)      ((a)+(b))  // 定义两个数相加的宏
#define luai_numsub(L,a,b)      ((a)-(b))  // 定义两个数相减的宏
#define luai_nummul(L,a,b)      ((a)*(b))  // 定义两个数相乘的宏
#define luai_numunm(L,a)        (-(a))      // 定义取负数的宏
#define luai_numeq(a,b)         ((a)==(b))  // 定义判断两个数是否相等的宏
#define luai_numlt(a,b)         ((a)<(b))   // 定义判断一个数是否小于另一个数的宏
#define luai_numle(a,b)         ((a)<=(b))  // 定义判断一个数是否小于等于另一个数的宏
#define luai_numgt(a,b)         ((a)>(b))   // 定义判断一个数是否大于另一个数的宏
#define luai_numge(a,b)         ((a)>=(b))  // 定义判断一个数是否大于等于另一个数的宏
#define luai_numisnan(a)        (!luai_numeq((a), (a)))  // 定义判断一个数是否为 NaN 的宏
#endif

/*
** macro to control inclusion of some hard tests on stack reallocation
*/
#if !defined(HARDSTACKTESTS)
#define condmovestack(L,pre,pos)    ((void)0)  // 定义条件移动栈的宏
#else
/* realloc stack keeping its size */
#define condmovestack(L,pre,pos)  \
  { int sz_ = stacksize(L); pre; luaD_reallocstack((L), sz_, 0); pos; }  // 定义条件移动栈并保持其大小的宏
#endif

#if !defined(HARDMEMTESTS)
#define condchangemem(L,pre,pos)    ((void)0)  // 定义条件改变内存的宏
#else
#define condchangemem(L,pre,pos)  \
    { if (gcrunning(G(L))) { pre; luaC_fullgc(L, 0); pos; } }  // 定义条件改变内存并执行完整垃圾回收的宏
#endif

#endif
```