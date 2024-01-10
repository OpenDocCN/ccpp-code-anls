# `nmap\liblua\lapi.h`

```
/*
** $Id: lapi.h $
** 辅助函数来自 Lua API
** 请参阅 lua.h 中的版权声明
*/

#ifndef lapi_h
#define lapi_h

#include "llimits.h"  // 包含 llmits.h 文件
#include "lstate.h"   // 包含 lstate.h 文件

/* 增加 'L->top'，检查堆栈溢出 */
#define api_incr_top(L)   {L->top++; api_check(L, L->top <= L->ci->top, \
                "stack overflow");}

/*
** 如果调用返回太多的多重返回值，被调用方可能没有堆栈空间来容纳所有结果。在这种情况下，这个宏会增加堆栈空间 ('L->ci->top')。
*/
#define adjustresults(L,nres) \
    { if ((nres) <= LUA_MULTRET && L->ci->top < L->top) L->ci->top = L->top; }

/* 确保堆栈至少有 'n' 个元素 */
#define api_checknelems(L,n)    api_check(L, (n) < (L->top - L->ci->func), \
                  "not enough elements in the stack")

/*
** 为了减少从 C 函数返回的开销，这些函数中待关闭变量的存在被编码在 CallInfo 的 'nresults' 字段中，以便没有待关闭变量的函数，希望返回零个、一个或 "all" 结果的函数没有额外开销。其他数量的期望结果的函数，以及有待关闭变量的函数，都有额外的检查。
*/
#define hastocloseCfunc(n)    ((n) < LUA_MULTRET)

/* 将 [-1, inf)（'nresults' 的范围）映射为 (-inf, -2] */
#define codeNresults(n)        (-(n) - 3)
#define decodeNresults(n)    (-(n) - 3)

#endif
```