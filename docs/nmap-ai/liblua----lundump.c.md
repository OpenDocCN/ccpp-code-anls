# `nmap\liblua\lundump.c`

```
/*
** $Id: lundump.c $
** load precompiled Lua chunks
** See Copyright Notice in lua.h
*/

#define lundump_c
#define LUA_CORE

#include "lprefix.h"


#include <limits.h>
#include <string.h>

#include "lua.h"

#include "ldebug.h"
#include "ldo.h"
#include "lfunc.h"
#include "lmem.h"
#include "lobject.h"
#include "lstring.h"
#include "lundump.h"
#include "lzio.h"


#if !defined(luai_verifycode)
#define luai_verifycode(L,f)  /* empty */
#endif


typedef struct {
  lua_State *L;
  ZIO *Z;
  const char *name;
} LoadState;


static l_noret error (LoadState *S, const char *why) {
  luaO_pushfstring(S->L, "%s: bad binary format (%s)", S->name, why);
  luaD_throw(S->L, LUA_ERRSYNTAX);
}


/*
** All high-level loads go through loadVector; you can change it to
** adapt to the endianness of the input
*/
#define loadVector(S,b,n)    loadBlock(S,b,(n)*sizeof((b)[0]))

// 加载数据块
static void loadBlock (LoadState *S, void *b, size_t size) {
  if (luaZ_read(S->Z, b, size) != 0)
    error(S, "truncated chunk");
}


#define loadVar(S,x)        loadVector(S,&x,1)

// 加载一个字节
static lu_byte loadByte (LoadState *S) {
  int b = zgetc(S->Z);
  if (b == EOZ)
    error(S, "truncated chunk");
  return cast_byte(b);
}

// 加载一个无符号整数
static size_t loadUnsigned (LoadState *S, size_t limit) {
  size_t x = 0;
  int b;
  limit >>= 7;
  do {
    b = loadByte(S);
    if (x >= limit)
      error(S, "integer overflow");
    x = (x << 7) | (b & 0x7f);
  } while ((b & 0x80) == 0);
  return x;
}

// 加载一个大小
static size_t loadSize (LoadState *S) {
  return loadUnsigned(S, ~(size_t)0);
}

// 加载一个整数
static int loadInt (LoadState *S) {
  return cast_int(loadUnsigned(S, INT_MAX));
}

// 加载一个浮点数
static lua_Number loadNumber (LoadState *S) {
  lua_Number x;
  loadVar(S, x);
  return x;
}

// 加载一个整数
static lua_Integer loadInteger (LoadState *S) {
  lua_Integer x;
  loadVar(S, x);
  return x;
}

/*
** Load a nullable string into prototype 'p'.
*/
# 从加载状态S中加载一个非空字符串到原型'p'中
static TString *loadStringN (LoadState *S, Proto *p) {
  lua_State *L = S->L;  # 获取加载状态S中的Lua状态机
  TString *ts;  # 定义一个TString类型的指针变量
  size_t size = loadSize(S);  # 从加载状态S中加载字符串的大小
  if (size == 0)  # 如果字符串大小为0
    return NULL;  # 返回空指针
  else if (--size <= LUAI_MAXSHORTLEN) {  # 如果字符串大小小于等于LUAI_MAXSHORTLEN
    char buff[LUAI_MAXSHORTLEN];  # 定义一个大小为LUAI_MAXSHORTLEN的字符数组
    loadVector(S, buff, size);  # 从加载状态S中加载字符串到缓冲区中
    ts = luaS_newlstr(L, buff, size);  # 创建一个新的短字符串
  }
  else {  # 否则，即为长字符串
    ts = luaS_createlngstrobj(L, size);  # 创建一个新的长字符串
    setsvalue2s(L, L->top, ts);  # 将字符串作为栈顶的值（'loadVector'可能会进行垃圾回收）
    luaD_inctop(L);  # 增加栈顶指针
    loadVector(S, getstr(ts), size);  # 直接加载到最终位置
    L->top--;  # 弹出字符串
  }
  luaC_objbarrier(L, p, ts);  # 对象屏障，用于原型'p'和字符串'ts'
  return ts;  # 返回字符串
}

'''
** 加载一个非空字符串到原型'p'中。
'''
static TString *loadString (LoadState *S, Proto *p) {
  TString *st = loadStringN(S, p);  # 调用loadStringN函数加载字符串
  if (st == NULL)  # 如果字符串为空
    error(S, "bad format for constant string");  # 报错，常量字符串格式错误
  return st;  # 返回字符串
}

static void loadCode (LoadState *S, Proto *f) {
  int n = loadInt(S);  # 从加载状态S中加载一个整数
  f->code = luaM_newvectorchecked(S->L, n, Instruction);  # 为原型'f'的代码分配内存空间
  f->sizecode = n;  # 设置代码的大小
  loadVector(S, f->code, n);  # 从加载状态S中加载代码
}

static void loadFunction(LoadState *S, Proto *f, TString *psource);

static void loadConstants (LoadState *S, Proto *f) {
  int i;
  int n = loadInt(S);  # 从加载状态S中加载一个整数
  f->k = luaM_newvectorchecked(S->L, n, TValue);  # 为原型'f'的常量分配内存空间
  f->sizek = n;  # 设置常量的大小
  for (i = 0; i < n; i++)
    setnilvalue(&f->k[i]);  # 将常量数组中的值设置为nil
  for (i = 0; i < n; i++) {
    TValue *o = &f->k[i];  # 定义一个TValue类型的指针变量
    int t = loadByte(S);  # 从加载状态S中加载一个字节
    switch (t) {
      case LUA_VNIL:  # 如果类型为NIL
        setnilvalue(o);  # 设置值为nil
        break;
      case LUA_VFALSE:  # 如果类型为FALSE
        setbfvalue(o);  # 设置值为false
        break;
      case LUA_VTRUE:  # 如果类型为TRUE
        setbtvalue(o);  # 设置值为true
        break;
      case LUA_VNUMFLT:  # 如果类型为NUMFLT
        setfltvalue(o, loadNumber(S));  # 设置值为浮点数
        break;
      case LUA_VNUMINT:  # 如果类型为NUMINT
        setivalue(o, loadInteger(S));  # 设置值为整数
        break;
      case LUA_VSHRSTR:  # 如果类型为SHRSTR
      case LUA_VLNGSTR:  # 如果类型为LNGSTR
        setsvalue2n(S->L, o, loadString(S, f));  # 设置值为加载的字符串
        break;
      default: lua_assert(0);  # 否则，断言失败
    }
  }
}
# 加载函数的原型
static void loadProtos (LoadState *S, Proto *f) {
  # 读取一个整数，表示原型的数量
  int n = loadInt(S);
  # 为原型数组分配内存空间
  f->p = luaM_newvectorchecked(S->L, n, Proto *);
  f->sizep = n;
  # 初始化原型数组
  for (i = 0; i < n; i++)
    f->p[i] = NULL;
  # 为每个原型调用luaF_newproto函数创建新的原型对象
  for (i = 0; i < n; i++) {
    f->p[i] = luaF_newproto(S->L);
    # 设置GC屏障
    luaC_objbarrier(S->L, f, f->p[i]);
    # 调用loadFunction函数加载原型的函数信息
    loadFunction(S, f->p[i], f->source);
  }
}

'''
** 加载函数的upvalues。必须先填充名称，因为填充其他字段可能会引发读取错误，
** 并且创建错误消息可能会调用紧急收集；在这种情况下，所有原型必须对GC保持一致。
'''
static void loadUpvalues (LoadState *S, Proto *f) {
  # 读取一个整数，表示upvalues的数量
  int n = loadInt(S);
  # 为upvalues数组分配内存空间
  f->upvalues = luaM_newvectorchecked(S->L, n, Upvaldesc);
  f->sizeupvalues = n;
  # 初始化upvalues数组
  for (i = 0; i < n; i++)  
    f->upvalues[i].name = NULL;
  # 为每个upvalue设置instack、idx和kind
  for (i = 0; i < n; i++) {  
    f->upvalues[i].instack = loadByte(S);
    f->upvalues[i].idx = loadByte(S);
    f->upvalues[i].kind = loadByte(S);
  }
}

# 加载调试信息
static void loadDebug (LoadState *S, Proto *f) {
  # 读取一个整数，表示lineinfo的数量
  int n = loadInt(S);
  # 为lineinfo数组分配内存空间
  f->lineinfo = luaM_newvectorchecked(S->L, n, ls_byte);
  f->sizelineinfo = n;
  # 加载lineinfo数组的内容
  loadVector(S, f->lineinfo, n);
  # 读取一个整数，表示abslineinfo的数量
  n = loadInt(S);
  # 为abslineinfo数组分配内存空间
  f->abslineinfo = luaM_newvectorchecked(S->L, n, AbsLineInfo);
  f->sizeabslineinfo = n;
  # 为每个abslineinfo设置pc和line
  for (i = 0; i < n; i++) {
    f->abslineinfo[i].pc = loadInt(S);
    f->abslineinfo[i].line = loadInt(S);
  }
  # 读取一个整数，表示locvars的数量
  n = loadInt(S);
  # 为locvars数组分配内存空间
  f->locvars = luaM_newvectorchecked(S->L, n, LocVar);
  f->sizelocvars = n;
  # 初始化locvars数组
  for (i = 0; i < n; i++)
    f->locvars[i].varname = NULL;
  # 为每个locvars设置varname、startpc和endpc
  for (i = 0; i < n; i++) {
    f->locvars[i].varname = loadStringN(S, f);
    f->locvars[i].startpc = loadInt(S);
    f->locvars[i].endpc = loadInt(S);
  }
  # 读取一个整数，表示upvalues的数量
  n = loadInt(S);
  # 为upvalues数组中的每个元素设置name
  for (i = 0; i < n; i++)
    f->upvalues[i].name = loadStringN(S, f);
}
static void loadFunction (LoadState *S, Proto *f, TString *psource) {
  // 加载函数的源码信息
  f->source = loadStringN(S, f);
  // 如果没有源码信息，则复用父级的源码信息
  if (f->source == NULL)  /* no source in dump? */
    f->source = psource;  /* reuse parent's source */
  // 加载函数的起始行号
  f->linedefined = loadInt(S);
  // 加载函数的结束行号
  f->lastlinedefined = loadInt(S);
  // 加载函数的参数个数
  f->numparams = loadByte(S);
  // 加载函数是否具有可变参数
  f->is_vararg = loadByte(S);
  // 加载函数的最大寄存器数量
  f->maxstacksize = loadByte(S);
  // 加载函数的指令集
  loadCode(S, f);
  // 加载函数的常量表
  loadConstants(S, f);
  // 加载函数的上值表
  loadUpvalues(S, f);
  // 加载函数的子函数原型
  loadProtos(S, f);
  // 加载函数的调试信息
  loadDebug(S, f);
}


static void checkliteral (LoadState *S, const char *s, const char *msg) {
  // 创建一个足够大的缓冲区
  char buff[sizeof(LUA_SIGNATURE) + sizeof(LUAC_DATA)]; /* larger than both */
  // 获取字符串的长度
  size_t len = strlen(s);
  // 加载指定长度的数据到缓冲区
  loadVector(S, buff, len);
  // 比较加载的数据和指定字符串是否相等，不相等则报错
  if (memcmp(s, buff, len) != 0)
    error(S, msg);
}


static void fchecksize (LoadState *S, size_t size, const char *tname) {
  // 检查加载的数据是否和指定大小相等，不相等则报错
  if (loadByte(S) != size)
    error(S, luaO_pushfstring(S->L, "%s size mismatch", tname));
}


#define checksize(S,t)    fchecksize(S,sizeof(t),#t)

static void checkHeader (LoadState *S) {
  // 跳过已经读取和检查过的第一个字符
  checkliteral(S, &LUA_SIGNATURE[1], "not a binary chunk");
  // 检查加载的数据是否和指定版本号相等，不相等则报错
  if (loadByte(S) != LUAC_VERSION)
    error(S, "version mismatch");
  // 检查加载的数据是否和指定格式号相等，不相等则报错
  if (loadByte(S) != LUAC_FORMAT)
    error(S, "format mismatch");
  // 检查加载的数据是否和指定数据相等，不相等则报错
  checkliteral(S, LUAC_DATA, "corrupted chunk");
  // 检查加载的数据是否和指定指令集大小相等，不相等则报错
  checksize(S, Instruction);
  // 检查加载的数据是否和指定整数类型大小相等，不相等则报错
  checksize(S, lua_Integer);
  // 检查加载的数据是否和指定浮点数类型大小相等，不相等则报错
  checksize(S, lua_Number);
  // 检查加载的数据是否和指定整数格式相等，不相等则报错
  if (loadInteger(S) != LUAC_INT)
    error(S, "integer format mismatch");
  // 检查加载的数据是否和指定浮点数格式相等，不相等则报错
  if (loadNumber(S) != LUAC_NUM)
    error(S, "float format mismatch");
}


/*
** Load precompiled chunk.
*/
LClosure *luaU_undump(lua_State *L, ZIO *Z, const char *name) {
  // 创建加载状态
  LoadState S;
  // 创建闭包
  LClosure *cl;
  // 根据名称设置加载状态的名称
  if (*name == '@' || *name == '=')
    S.name = name + 1;
  else if (*name == LUA_SIGNATURE[0])
    S.name = "binary string";
  else
    # 设置结构体 S 的 name 属性为传入的 name
    S.name = name;
    # 设置结构体 S 的 L 属性为传入的 L
    S.L = L;
    # 设置结构体 S 的 Z 属性为传入的 Z
    S.Z = Z;
    # 检查结构体 S 的头部信息
    checkHeader(&S);
    # 在 Lua 状态机 L 中创建一个新的闭包 cl
    cl = luaF_newLclosure(L, loadByte(&S));
    # 将闭包 cl 压入 Lua 栈顶
    setclLvalue2s(L, L->top, cl);
    # 增加 Lua 栈顶指针
    luaD_inctop(L);
    # 为闭包 cl 创建一个新的原型对象
    cl->p = luaF_newproto(L);
    # 设置闭包 cl 为原型对象 cl->p 的弱引用
    luaC_objbarrier(L, cl, cl->p);
    # 从结构体 S 中加载函数到闭包 cl 的原型对象 cl->p
    loadFunction(&S, cl->p, NULL);
    # 断言闭包 cl 的上值数量等于原型对象 cl->p 的上值数量
    lua_assert(cl->nupvalues == cl->p->sizeupvalues);
    # 验证闭包 cl 的原型对象 cl->p 的代码
    luai_verifycode(L, cl->p);
    # 返回创建的闭包 cl
    return cl;
# 代码块结束
```