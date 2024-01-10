# `nmap\liblua\ldebug.h`

```
/*
** $Id: ldebug.h $
** 辅助函数来自调试接口模块
** 请参阅lua.h中的版权声明
*/

#ifndef ldebug_h
#define ldebug_h


#include "lstate.h"


#define pcRel(pc, p)    (cast_int((pc) - (p)->code) - 1)  // 计算指令在代码中的相对位置


/* 活动的 Lua 函数（给定调用信息） */
#define ci_func(ci)        (clLvalue(s2v((ci)->func)))  // 获取调用信息中的函数


#define resethookcount(L)    (L->hookcount = L->basehookcount)  // 重置钩子计数器


/*
** 标记'lineinfo'数组中具有'abslineinfo'数组中绝对信息的条目
*/
#define ABSLINEINFO    (-0x80)


/*
** 连续指令没有绝对行信息的最大数量。（2的幂可以快速进行除法运算。）
*/
#if !defined(MAXIWTHABS)
#define MAXIWTHABS    128
#endif


LUAI_FUNC int luaG_getfuncline (const Proto *f, int pc);  // 获取函数中指定指令的行号
LUAI_FUNC const char *luaG_findlocal (lua_State *L, CallInfo *ci, int n, StkId *pos);  // 查找本地变量
LUAI_FUNC l_noret luaG_typeerror (lua_State *L, const TValue *o, const char *opname);  // 类型错误处理
LUAI_FUNC l_noret luaG_callerror (lua_State *L, const TValue *o);  // 调用错误处理
LUAI_FUNC l_noret luaG_forerror (lua_State *L, const TValue *o, const char *what);  // for 循环错误处理
LUAI_FUNC l_noret luaG_concaterror (lua_State *L, const TValue *p1, const TValue *p2);  // 连接错误处理
LUAI_FUNC l_noret luaG_opinterror (lua_State *L, const TValue *p1, const TValue *p2, const char *msg);  // 操作数为整数错误处理
LUAI_FUNC l_noret luaG_tointerror (lua_State *L, const TValue *p1, const TValue *p2);  // 转换为整数错误处理
LUAI_FUNC l_noret luaG_ordererror (lua_State *L, const TValue *p1, const TValue *p2);  // 顺序比较错误处理
LUAI_FUNC l_noret luaG_runerror (lua_State *L, const char *fmt, ...);  // 运行时错误处理
// 声明一个名为 luaG_addinfo 的函数，返回类型为 const char*，参数为 lua_State*、const char*、TString*、int
LUAI_FUNC const char *luaG_addinfo (lua_State *L, const char *msg, TString *src, int line);
// 声明一个名为 luaG_errormsg 的函数，返回类型为 l_noret，参数为 lua_State*
LUAI_FUNC l_noret luaG_errormsg (lua_State *L);
// 声明一个名为 luaG_traceexec 的函数，返回类型为 int，参数为 lua_State*、const Instruction*
LUAI_FUNC int luaG_traceexec (lua_State *L, const Instruction *pc);
// 结束条件编译指令
#endif
```