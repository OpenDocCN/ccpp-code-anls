# `nmap\liblua\lparser.c`

```
/*
** $Id: lparser.c $
** Lua Parser
** See Copyright Notice in lua.h
*/

// 定义 lparser_c，用于标识 lparser.c 文件
#define lparser_c
// 定义 LUA_CORE，用于标识 Lua 核心模块
#define LUA_CORE

// 包含预编译头文件 lprefix.h
#include "lprefix.h"

// 包含标准库头文件
#include <limits.h>
#include <string.h>

// 包含 Lua 头文件
#include "lua.h"

// 包含 Lua 解释器的相关头文件
#include "lcode.h"
#include "ldebug.h"
#include "ldo.h"
#include "lfunc.h"
#include "llex.h"
#include "lmem.h"
#include "lobject.h"
#include "lopcodes.h"
#include "lparser.h"
#include "lstate.h"
#include "lstring.h"
#include "ltable.h"

// 定义每个函数的最大局部变量数量
#define MAXVARS        200

// 定义宏函数，用于判断表达式是否具有多个返回值
#define hasmultret(k)        ((k) == VCALL || (k) == VVARARG)

// 定义宏函数，用于判断两个字符串是否相等
#define eqstr(a,b)    ((a) == (b))

// 定义结构体，用于表示活动块列表中的节点
typedef struct BlockCnt {
  struct BlockCnt *previous;  // 链表
  int firstlabel;  // 该块中第一个标签的索引
  int firstgoto;  // 该块中第一个待处理的跳转语句的索引
  lu_byte nactvar;  // 该块外部活动局部变量的数量
  lu_byte upval;  // 如果该块中的某个变量是上值，则为 true
  lu_byte isloop;  // 如果 'block' 是一个循环，则为 true
  lu_byte insidetbc;  // 如果在待关闭变量的作用域内，则为 true
} BlockCnt;

// 递归非终结符函数的原型声明
static void statement (LexState *ls);
static void expr (LexState *ls, expdesc *v);

// 定义错误处理函数，用于报告预期的语法错误
static l_noret error_expected (LexState *ls, int token) {
  luaX_syntaxerror(ls,
      luaO_pushfstring(ls->L, "%s expected", luaX_token2str(ls, token)));
}
/*
** 设置错误限制，如果超出限制则报错
*/
static l_noret errorlimit (FuncState *fs, int limit, const char *what) {
  lua_State *L = fs->ls->L;  // 获取 Lua 状态机
  const char *msg;  // 错误信息
  int line = fs->f->linedefined;  // 获取函数定义的行号
  const char *where = (line == 0)  // 判断是否为主函数
                      ? "main function"
                      : luaO_pushfstring(L, "function at line %d", line);  // 根据行号生成函数位置信息
  msg = luaO_pushfstring(L, "too many %s (limit is %d) in %s",  // 生成错误信息
                             what, limit, where);
  luaX_syntaxerror(fs->ls, msg);  // 抛出语法错误
}


/*
** 检查是否超出限制，如果超出则调用 errorlimit 报错
*/
static void checklimit (FuncState *fs, int v, int l, const char *what) {
  if (v > l) errorlimit(fs, l, what);  // 如果超出限制则报错
}


/*
** 测试下一个标记是否为 'c'，如果是则跳过
*/
static int testnext (LexState *ls, int c) {
  if (ls->t.token == c) {  // 判断下一个标记是否为 'c'
    luaX_next(ls);  // 跳过当前标记
    return 1;  // 返回 1 表示成功
  }
  else return 0;  // 返回 0 表示失败
}


/*
** 检查下一个标记是否为 'c'
*/
static void check (LexState *ls, int c) {
  if (ls->t.token != c)  // 判断下一个标记是否为 'c'
    error_expected(ls, c);  // 抛出预期错误
}


/*
** 检查下一个标记是否为 'c'，如果是则跳过
*/
static void checknext (LexState *ls, int c) {
  check(ls, c);  // 检查下一个标记是否为 'c'
  luaX_next(ls);  // 跳过当前标记
}


#define check_condition(ls,c,msg)    { if (!(c)) luaX_syntaxerror(ls, msg); }  // 定义宏，检查条件是否满足，否则抛出语法错误


/*
** 检查下一个标记是否为 'what'，如果不是则报错
** 如果出现错误，抛出一个错误，指出预期的 'what' 应该匹配 'who' 在 'where' 行（如果不是当前行）
*/
static void check_match (LexState *ls, int what, int who, int where) {
  if (l_unlikely(!testnext(ls, what))) {  // 如果下一个标记不是 'what'
    if (where == ls->linenumber)  // 如果都在同一行
      error_expected(ls, what);  // 抛出预期错误
    else {
      luaX_syntaxerror(ls, luaO_pushfstring(ls->L,
             "%s expected (to close %s at line %d)",
              luaX_token2str(ls, what), luaX_token2str(ls, who), where));  // 抛出语法错误，指出预期的标记
    }
  }
}


/*
** 检查下一个标记是否为名称，如果是则返回该名称
*/
static TString *str_checkname (LexState *ls) {
  TString *ts;
  check(ls, TK_NAME);  // 检查下一个标记是否为名称
  ts = ls->t.seminfo.ts;  // 获取名称的字符串
  luaX_next(ls);  // 跳过当前标记
  return ts;  // 返回名称
}


/*
** 初始化表达式，设置表达式的类型和信息
*/
static void init_exp (expdesc *e, expkind k, int i) {
  e->f = e->t = NO_JUMP;  // 初始化跳转位置
  e->k = k;  // 设置表达式类型
  e->u.info = i;  // 设置表达式信息
}
static void codestring (expdesc *e, TString *s) {
  e->f = e->t = NO_JUMP;  // 初始化表达式的跳转位置为 NO_JUMP
  e->k = VKSTR;  // 表达式类型为字符串
  e->u.strval = s;  // 表达式的字符串值为 s
}


static void codename (LexState *ls, expdesc *e) {
  codestring(e, str_checkname(ls));  // 为表达式生成一个字符串
}


/*
** Register a new local variable in the active 'Proto' (for debug
** information).
*/
static int registerlocalvar (LexState *ls, FuncState *fs, TString *varname) {
  Proto *f = fs->f;  // 获取函数状态的 Proto 结构
  int oldsize = f->sizelocvars;  // 保存之前的局部变量数量
  luaM_growvector(ls->L, f->locvars, fs->ndebugvars, f->sizelocvars,
                  LocVar, SHRT_MAX, "local variables");  // 分配或重新分配局部变量数组的内存空间
  while (oldsize < f->sizelocvars)  // 遍历之前的局部变量数组
    f->locvars[oldsize++].varname = NULL;  // 将之前的局部变量名置为 NULL
  f->locvars[fs->ndebugvars].varname = varname;  // 设置新的局部变量名
  f->locvars[fs->ndebugvars].startpc = fs->pc;  // 设置新的局部变量的起始指令位置
  luaC_objbarrier(ls->L, f, varname);  // 设置 Lua 垃圾回收的屏障
  return fs->ndebugvars++;  // 返回新的局部变量数量
}


/*
** Create a new local variable with the given 'name'. Return its index
** in the function.
*/
static int new_localvar (LexState *ls, TString *name) {
  lua_State *L = ls->L;  // 获取 Lua 状态
  FuncState *fs = ls->fs;  // 获取函数状态
  Dyndata *dyd = ls->dyd;  // 获取动态数据
  Vardesc *var;  // 定义变量描述符
  checklimit(fs, dyd->actvar.n + 1 - fs->firstlocal,
                 MAXVARS, "local variables");  // 检查局部变量数量是否超出限制
  luaM_growvector(L, dyd->actvar.arr, dyd->actvar.n + 1,
                  dyd->actvar.size, Vardesc, USHRT_MAX, "local variables");  // 分配或重新分配局部变量数组的内存空间
  var = &dyd->actvar.arr[dyd->actvar.n++];  // 获取新的局部变量描述符
  var->vd.kind = VDKREG;  /* default */  // 设置局部变量描述符的类型为寄存器
  var->vd.name = name;  // 设置局部变量描述符的名称
  return dyd->actvar.n - 1 - fs->firstlocal;  // 返回新的局部变量索引
}

#define new_localvarliteral(ls,v) \
    new_localvar(ls,  \
      luaX_newstring(ls, "" v, (sizeof(v)/sizeof(char)) - 1));


/*
** Return the "variable description" (Vardesc) of a given variable.
** (Unless noted otherwise, all variables are referred to by their
** compiler indices.)
*/
static Vardesc *getlocalvardesc (FuncState *fs, int vidx) {
  return &fs->ls->dyd->actvar.arr[fs->firstlocal + vidx];  // 返回给定变量的变量描述符
}


/*
** Convert 'nvar', a compiler index level, to its corresponding
** register. For that, search for the highest variable below that level
/*
** 根据变量索引（'nvar'）返回其在寄存器中的级别
*/
static int reglevel (FuncState *fs, int nvar) {
  while (nvar-- > 0) {
    // 获取先前变量的描述信息
    Vardesc *vd = getlocalvardesc(fs, nvar);  /* get previous variable */
    // 如果变量不在寄存器中，则返回其寄存器索引加一
    if (vd->vd.kind != RDKCTC)  /* is in a register? */
      return vd->vd.ridx + 1;
  }
  // 如果没有变量在寄存器中，则返回0
  return 0;  /* no variables in registers */
}


/*
** 返回给定函数中寄存器堆栈中的变量数量
*/
int luaY_nvarstack (FuncState *fs) {
  return reglevel(fs, fs->nactvar);
}


/*
** 获取当前变量 'vidx' 的调试信息条目
*/
static LocVar *localdebuginfo (FuncState *fs, int vidx) {
  Vardesc *vd = getlocalvardesc(fs,  vidx);
  // 如果变量是常量，则返回空
  if (vd->vd.kind == RDKCTC)
    return NULL;  /* no debug info. for constants */
  else {
    int idx = vd->vd.pidx;
    lua_assert(idx < fs->ndebugvars);
    return &fs->f->locvars[idx];
  }
}


/*
** 创建表示变量 'vidx' 的表达式
*/
static void init_var (FuncState *fs, expdesc *e, int vidx) {
  e->f = e->t = NO_JUMP;
  e->k = VLOCAL;
  e->u.var.vidx = vidx;
  e->u.var.ridx = getlocalvardesc(fs, vidx)->vd.ridx;
}


/*
** 如果 'e' 描述的变量是只读的，则引发错误
*/
static void check_readonly (LexState *ls, expdesc *e) {
  FuncState *fs = ls->fs;
  TString *varname = NULL;  /* to be set if variable is const */
  switch (e->k) {
    case VCONST: {
      varname = ls->dyd->actvar.arr[e->u.info].vd.name;
      break;
    }
    case VLOCAL: {
      Vardesc *vardesc = getlocalvardesc(fs, e->u.var.vidx);
      // 如果不是常规变量，则设置变量名
      if (vardesc->vd.kind != VDKREG)  /* not a regular variable? */
        varname = vardesc->vd.name;
      break;
    }
    case VUPVAL: {
      Upvaldesc *up = &fs->f->upvalues[e->u.info];
      // 如果不是寄存器变量，则设置变量名
      if (up->kind != VDKREG)
        varname = up->name;
      break;
    }
    default:
      return;  /* other cases cannot be read-only */
  }
  // 如果变量名存在，则引发错误
  if (varname) {
    // 声明一个指向常量字符的指针msg，并使用luaO_pushfstring函数将格式化的错误信息推入Lua栈中
    const char *msg = luaO_pushfstring(ls->L, "attempt to assign to const variable '%s'", getstr(varname));
    // 调用luaK_semerror函数，传入错误信息msg，用于报告语法分析错误
    luaK_semerror(ls, msg);  /* error */
  }
/*
** Start the scope for the last 'nvars' created variables.
*/
static void adjustlocalvars (LexState *ls, int nvars) {
  // 获取当前函数状态
  FuncState *fs = ls->fs;
  // 获取当前变量栈的长度
  int reglevel = luaY_nvarstack(fs);
  int i;
  // 遍历最后创建的 'nvars' 个变量
  for (i = 0; i < nvars; i++) {
    // 获取下一个活跃变量的索引
    int vidx = fs->nactvar++;
    // 获取当前变量的描述
    Vardesc *var = getlocalvardesc(fs, vidx);
    // 设置变量的寄存器索引
    var->vd.ridx = reglevel++;
    // 注册局部变量
    var->vd.pidx = registerlocalvar(ls, fs, var->vd.name);
  }
}


/*
** Close the scope for all variables up to level 'tolevel'.
** (debug info.)
*/
static void removevars (FuncState *fs, int tolevel) {
  // 减少活跃变量的数量
  fs->ls->dyd->actvar.n -= (fs->nactvar - tolevel);
  // 逐个关闭变量的作用域
  while (fs->nactvar > tolevel) {
    LocVar *var = localdebuginfo(fs, --fs->nactvar);
    if (var)  /* does it have debug information? */
      // 设置变量的结束指令位置
      var->endpc = fs->pc;
  }
}


/*
** Search the upvalues of the function 'fs' for one
** with the given 'name'.
*/
static int searchupvalue (FuncState *fs, TString *name) {
  int i;
  Upvaldesc *up = fs->f->upvalues;
  // 遍历查找函数的上值
  for (i = 0; i < fs->nups; i++) {
    if (eqstr(up[i].name, name)) return i;
  }
  return -1;  /* not found */
}


// 分配一个新的上值描述
static Upvaldesc *allocupvalue (FuncState *fs) {
  Proto *f = fs->f;
  int oldsize = f->sizeupvalues;
  // 检查上值的数量是否超出限制
  checklimit(fs, fs->nups + 1, MAXUPVAL, "upvalues");
  // 分配新的上值数组
  luaM_growvector(fs->ls->L, f->upvalues, fs->nups, f->sizeupvalues,
                  Upvaldesc, MAXUPVAL, "upvalues");
  while (oldsize < f->sizeupvalues)
    f->upvalues[oldsize++].name = NULL;
  return &f->upvalues[fs->nups++];
}


// 创建一个新的上值
static int newupvalue (FuncState *fs, TString *name, expdesc *v) {
  // 分配一个新的上值描述
  Upvaldesc *up = allocupvalue(fs);
  FuncState *prev = fs->prev;
  if (v->k == VLOCAL) {
    // 上值在栈内
    up->instack = 1;
    up->idx = v->u.var.ridx;
    up->kind = getlocalvardesc(prev, v->u.var.vidx)->vd.kind;
    lua_assert(eqstr(name, getlocalvardesc(prev, v->u.var.vidx)->vd.name));
  }
  else {
    // 上值在栈外
    up->instack = 0;
    up->idx = cast_byte(v->u.info);
    up->kind = prev->f->upvalues[v->u.info].kind;
  }
}
    # 使用lua_assert函数判断name和prev->f->upvalues[v->u.info].name是否相等
    lua_assert(eqstr(name, prev->f->upvalues[v->u.info].name));
  }
  # 将up->name赋值为name
  up->name = name;
  # 对name进行内存屏障处理
  luaC_objbarrier(fs->ls->L, fs->f, name);
  # 返回fs->nups - 1
  return fs->nups - 1;
/*
** 在函数 'fs' 中查找名为 'n' 的活动局部变量。如果找到，用它初始化 'var' 并返回其表达式类型；否则返回 -1。
*/
static int searchvar (FuncState *fs, TString *n, expdesc *var) {
  int i;
  for (i = cast_int(fs->nactvar) - 1; i >= 0; i--) {
    Vardesc *vd = getlocalvardesc(fs, i);
    if (eqstr(n, vd->vd.name)) {  /* 找到了？ */
      if (vd->vd.kind == RDKCTC)  /* 编译时常量？ */
        init_exp(var, VCONST, fs->firstlocal + i);
      else  /* 真实变量 */
        init_var(fs, var, i);
      return var->k;
    }
  }
  return -1;  /* 没找到 */
}


/*
** 标记定义了给定级别变量的块（以便稍后发出关闭指令）。
*/
static void markupval (FuncState *fs, int level) {
  BlockCnt *bl = fs->bl;
  while (bl->nactvar > level)
    bl = bl->previous;
  bl->upval = 1;
  fs->needclose = 1;
}


/*
** 标记当前块有一个待关闭的变量。
*/
static void marktobeclosed (FuncState *fs) {
  BlockCnt *bl = fs->bl;
  bl->upval = 1;
  bl->insidetbc = 1;
  fs->needclose = 1;
}


/*
** 查找名为 'n' 的变量。如果它是一个上值，将此上值添加到所有中间函数中。如果它是一个全局变量，将 'var' 设置为 'void' 作为标志。
*/
static void singlevaraux (FuncState *fs, TString *n, expdesc *var, int base) {
  if (fs == NULL)  /* 没有更多的级别了？ */
    init_exp(var, VVOID, 0);  /* 默认是全局的 */
  else {
    int v = searchvar(fs, n, var);  /* 查找当前级别的局部变量 */
    if (v >= 0) {  /* 找到了？ */
      if (v == VLOCAL && !base)
        markupval(fs, var->u.var.vidx);  /* 将局部变量用作上值 */
    }
  }
}
    else {  /* 如果在当前级别找不到变量，则尝试上值 */
      int idx = searchupvalue(fs, n);  /* 尝试现有的上值 */
      if (idx < 0) {  /* 没找到？ */
        singlevaraux(fs->prev, n, var, 0);  /* 尝试上一级 */
        if (var->k == VLOCAL || var->k == VUPVAL)  /* 是局部变量还是上值？ */
          idx  = newupvalue(fs, n, var);  /* 将会是一个新的上值 */
        else  /* 是全局变量或常量 */
          return;  /* 在这个级别不需要做任何事情 */
      }
      init_exp(var, VUPVAL, idx);  /* 新的或旧的上值 */
    }
  }
/*
** Find a variable with the given name 'n', handling global variables
** too.
*/
static void singlevar (LexState *ls, expdesc *var) {
  // 检查变量名是否合法，并返回变量名字符串
  TString *varname = str_checkname(ls);
  // 获取当前函数状态
  FuncState *fs = ls->fs;
  // 调用辅助函数，处理变量名，将结果存储在var中
  singlevaraux(fs, varname, var, 1);
  // 如果var的类型是VVOID，表示全局变量
  if (var->k == VVOID) {  /* global name? */
    expdesc key;
    // 获取环境变量
    singlevaraux(fs, ls->envn, var, 1);  /* get environment variable */
    // 断言变量一定存在
    lua_assert(var->k != VVOID);  /* this one must exist */
    // 将变量名存储在key中
    codestring(&key, varname);  /* key is variable name */
    // 将env[varname]存储在var中
    luaK_indexed(fs, var, &key);  /* env[varname] */
  }
}


/*
** Adjust the number of results from an expression list 'e' with 'nexps'
** expressions to 'nvars' values.
*/
static void adjust_assign (LexState *ls, int nvars, int nexps, expdesc *e) {
  // 获取当前函数状态
  FuncState *fs = ls->fs;
  // 计算需要的额外值
  int needed = nvars - nexps;  /* extra values needed */
  // 如果最后一个表达式有多个返回值
  if (hasmultret(e->k)) {  /* last expression has multiple returns? */
    // 计算额外的值
    int extra = needed + 1;  /* discount last expression itself */
    if (extra < 0)
      extra = 0;
    // 设置返回值
    luaK_setreturns(fs, e, extra);  /* last exp. provides the difference */
  }
  else {
    // 如果表达式类型不是VVOID
    if (e->k != VVOID)  /* at least one expression? */
      // 关闭最后一个表达式
      luaK_exp2nextreg(fs, e);  /* close last expression */
    // 如果需要额外的值
    if (needed > 0)  /* missing values? */
      // 用nil填充
      luaK_nil(fs, fs->freereg, needed);  /* complete with nils */
  }
  // 如果需要额外的值
  if (needed > 0)
    // 为额外的值保留寄存器
    luaK_reserveregs(fs, needed);  /* registers for extra values */
  else  /* adding 'needed' is actually a subtraction */
    // 移除额外的值
    fs->freereg += needed;  /* remove extra values */
}


// 定义宏，增加C栈的深度
#define enterlevel(ls)    luaE_incCstack(ls->L)


// 定义宏，减少C栈的深度
#define leavelevel(ls) ((ls)->L->nCcalls--)


/*
** Generates an error that a goto jumps into the scope of some
** local variable.
*/
# 当跳转到一个标签的作用域范围内时，抛出错误
static l_noret jumpscopeerror (LexState *ls, Labeldesc *gt) {
  # 获取变量名
  const char *varname = getstr(getlocalvardesc(ls->fs, gt->nactvar)->vd.name);
  # 设置错误信息
  const char *msg = "<goto %s> at line %d jumps into the scope of local '%s'";
  # 将标签名、行号和变量名格式化为错误信息
  msg = luaO_pushfstring(ls->L, msg, getstr(gt->name), gt->line, varname);
  # 抛出错误
  luaK_semerror(ls, msg);
}

/*
** 解决给定标签的跳转，并将其从待解决的跳转列表中移除
** 如果跳转到某个变量的作用域范围内，抛出错误
*/
static void solvegoto (LexState *ls, int g, Labeldesc *label) {
  int i;
  Labellist *gl = &ls->dyd->gt;  /* 跳转列表 */
  Labeldesc *gt = &gl->arr[g];  /* 待解决的跳转 */
  lua_assert(eqstr(gt->name, label->name));
  if (l_unlikely(gt->nactvar < label->nactvar))  /* 进入某个作用域？ */
    jumpscopeerror(ls, gt);
  luaK_patchlist(ls->fs, gt->pc, label->pc);
  for (i = g; i < gl->n - 1; i++)  /* 从待解决列表中移除跳转 */
    gl->arr[i] = gl->arr[i + 1];
  gl->n--;
}

/*
** 查找具有给定名称的活动标签
*/
static Labeldesc *findlabel (LexState *ls, TString *name) {
  int i;
  Dyndata *dyd = ls->dyd;
  /* 检查当前函数中的标签是否匹配 */
  for (i = ls->fs->firstlabel; i < dyd->label.n; i++) {
    Labeldesc *lb = &dyd->label.arr[i];
    if (eqstr(lb->name, name))  /* 正确的标签？ */
      return lb;
  }
  return NULL;  /* 未找到标签 */
}

/*
** 在相应的列表中添加一个新的标签/跳转
*/
static int newlabelentry (LexState *ls, Labellist *l, TString *name,
                          int line, int pc) {
  int n = l->n;
  luaM_growvector(ls->L, l->arr, n, l->size,
                  Labeldesc, SHRT_MAX, "labels/gotos");
  l->arr[n].name = name;
  l->arr[n].line = line;
  l->arr[n].nactvar = ls->fs->nactvar;
  l->arr[n].close = 0;
  l->arr[n].pc = pc;
  l->n = n + 1;
  return n;
}
static int newgotoentry (LexState *ls, TString *name, int line, int pc) {
  // 创建一个新的跳转目标，并将其添加到当前函数状态的跳转目标列表中
  return newlabelentry(ls, &ls->dyd->gt, name, line, pc);
}


/*
** 解决向前跳转。检查新标签 'lb' 是否与当前块中的任何待定跳转匹配并解决它们。如果任何跳转需要关闭 upvalue，则返回 true。
*/
static int solvegotos (LexState *ls, Labeldesc *lb) {
  Labellist *gl = &ls->dyd->gt;
  int i = ls->fs->bl->firstgoto;
  int needsclose = 0;
  while (i < gl->n) {
    if (eqstr(gl->arr[i].name, lb->name)) {
      needsclose |= gl->arr[i].close;
      solvegoto(ls, i, lb);  /* 将从列表中移除 'i' */
    }
    else
      i++;
  }
  return needsclose;
}


/*
** 在给定 'line' 处创建一个具有给定 'name' 的新标签。
** 'last' 告诉标签是否是其块中最后一个非操作语句。解决所有待定的跳转到这个新标签，并在必要时添加一个 close 指令。
** 如果添加了 close 指令，则返回 true。
*/
static int createlabel (LexState *ls, TString *name, int line,
                        int last) {
  FuncState *fs = ls->fs;
  Labellist *ll = &ls->dyd->label;
  int l = newlabelentry(ls, ll, name, line, luaK_getlabel(fs));
  if (last) {  /* 标签是块中最后一个非操作语句？ */
    /* 假设局部变量已经超出范围 */
    ll->arr[l].nactvar = fs->bl->nactvar;
  }
  if (solvegotos(ls, &ll->arr[l])) {  /* 需要 close 吗？ */
    luaK_codeABC(fs, OP_CLOSE, luaY_nvarstack(fs), 0, 0);
    return 1;
  }
  return 0;
}


/*
** 调整块的待定跳转到外部级别。
*/
static void movegotosout (FuncState *fs, BlockCnt *bl) {
  int i;
  Labellist *gl = &fs->ls->dyd->gt;
  /* 修正当前块的待定跳转 */
  for (i = bl->firstgoto; i < gl->n; i++) {  /* 对于每个待定跳转 */
    Labeldesc *gt = &gl->arr[i];
    /* 离开一个变量作用域？ */
    if (reglevel(fs, gt->nactvar) > reglevel(fs, bl->nactvar))
      gt->close |= bl->upval;  /* 跳转可能需要一个 close */
    gt->nactvar = bl->nactvar;  /* 更新跳转级别的活跃变量数 */
  }
}

// 进入一个新的代码块
static void enterblock (FuncState *fs, BlockCnt *bl, lu_byte isloop) {
  bl->isloop = isloop;  // 设置当前代码块是否为循环
  bl->nactvar = fs->nactvar;  // 记录当前活跃变量的数量
  bl->firstlabel = fs->ls->dyd->label.n;  // 记录当前函数中的标签数量
  bl->firstgoto = fs->ls->dyd->gt.n;  // 记录当前函数中的goto语句数量
  bl->upval = 0;  // 初始化上值数量为0
  bl->insidetbc = (fs->bl != NULL && fs->bl->insidetbc);  // 判断是否在tbc内部
  bl->previous = fs->bl;  // 记录前一个代码块
  fs->bl = bl;  // 设置当前代码块为新的代码块
  lua_assert(fs->freereg == luaY_nvarstack(fs));  // 断言当前空闲寄存器数量等于变量栈的数量
}


/*
** 为未定义的 'goto' 生成一个错误
*/
static l_noret undefgoto (LexState *ls, Labeldesc *gt) {
  const char *msg;  // 错误消息
  if (eqstr(gt->name, luaS_newliteral(ls->L, "break"))) {  // 如果是'break'语句
    msg = "break outside loop at line %d";  // 设置错误消息
    msg = luaO_pushfstring(ls->L, msg, gt->line);  // 将行号插入错误消息
  }
  else {
    msg = "no visible label '%s' for <goto> at line %d";  // 设置错误消息
    msg = luaO_pushfstring(ls->L, msg, getstr(gt->name), gt->line);  // 将标签名和行号插入错误消息
  }
  luaK_semerror(ls, msg);  // 抛出语法错误
}


// 离开当前代码块
static void leaveblock (FuncState *fs) {
  BlockCnt *bl = fs->bl;  // 获取当前代码块
  LexState *ls = fs->ls;  // 获取词法状态
  int hasclose = 0;  // 是否有关闭
  int stklevel = reglevel(fs, bl->nactvar);  // 获取代码块外部的级别
  if (bl->isloop)  // 如果是循环
    hasclose = createlabel(ls, luaS_newliteral(ls->L, "break"), 0, 0);  // 创建一个'break'标签
  if (!hasclose && bl->previous && bl->upval)  // 如果没有关闭且有前一个代码块和上值
    luaK_codeABC(fs, OP_CLOSE, stklevel, 0, 0);  // 生成OP_CLOSE指令
  fs->bl = bl->previous;  // 设置当前代码块为前一个代码块
  removevars(fs, bl->nactvar);  // 移除变量
  lua_assert(bl->nactvar == fs->nactvar);  // 断言当前活跃变量数量等于函数的活跃变量数量
  fs->freereg = stklevel;  // 释放寄存器
  ls->dyd->label.n = bl->firstlabel;  // 移除本地标签
  if (bl->previous)  // 如果有内部代码块
    movegotosout(fs, bl);  // 更新到外部代码块的未决goto
  else {
    if (bl->firstgoto < ls->dyd->gt.n)  // 如果外部代码块有未决goto
      undefgoto(ls, &ls->dyd->gt.arr[bl->firstgoto]);  // 抛出未定义的goto错误
  }
}


/*
** 将一个新的原型添加到原型列表中
*/
static Proto *addprototype (LexState *ls) {
  Proto *clp;  // 新的原型
  lua_State *L = ls->L;  // Lua状态
  FuncState *fs = ls->fs;  // 函数状态
  Proto *f = fs->f;  // 当前函数的原型
  if (fs->np >= f->sizep) {  // 如果原型数量超过了当前函数原型的大小
    int oldsize = f->sizep;  // 保存旧的大小
    # 调用 luaM_growvector 函数，用于增长函数原型数组的大小
    luaM_growvector(L, f->p, fs->np, f->sizep, Proto *, MAXARG_Bx, "functions");
    # 循环，将原来大小小于新大小的函数原型数组中的元素赋值为 NULL
    while (oldsize < f->sizep)
      f->p[oldsize++] = NULL;
  }
  # 将新的函数原型对象添加到函数原型数组中，并赋值给 clp
  f->p[fs->np++] = clp = luaF_newproto(L);
  # 对象屏障，用于标记 f 对象和 clp 对象之间的关系
  luaC_objbarrier(L, f, clp);
  # 返回新创建的函数原型对象
  return clp;
/*
** codes instruction to create new closure in parent function.
** The OP_CLOSURE instruction uses the last available register,
** so that, if it invokes the GC, the GC knows which registers
** are in use at that time.
*/
static void codeclosure (LexState *ls, expdesc *v) {
  // 获取父函数的函数状态
  FuncState *fs = ls->fs->prev;
  // 初始化表达式描述符，设置为可重定位，生成 OP_CLOSURE 指令
  init_exp(v, VRELOC, luaK_codeABx(fs, OP_CLOSURE, 0, fs->np - 1));
  // 将表达式描述符转换为下一个寄存器
  luaK_exp2nextreg(fs, v);  /* fix it at the last register */
}

// 打开函数
static void open_func (LexState *ls, FuncState *fs, BlockCnt *bl) {
  // 获取函数原型
  Proto *f = fs->f;
  // 将当前函数状态添加到函数状态链表中
  fs->prev = ls->fs;  /* linked list of funcstates */
  fs->ls = ls;
  ls->fs = fs;
  fs->pc = 0;
  fs->previousline = f->linedefined;
  fs->iwthabs = 0;
  fs->lasttarget = 0;
  fs->freereg = 0;
  fs->nk = 0;
  fs->nabslineinfo = 0;
  fs->np = 0;
  fs->nups = 0;
  fs->ndebugvars = 0;
  fs->nactvar = 0;
  fs->needclose = 0;
  fs->firstlocal = ls->dyd->actvar.n;
  fs->firstlabel = ls->dyd->label.n;
  fs->bl = NULL;
  f->source = ls->source;
  luaC_objbarrier(ls->L, f, f->source);
  f->maxstacksize = 2;  /* registers 0/1 are always valid */
  enterblock(fs, bl, 0);
}

// 关闭函数
static void close_func (LexState *ls) {
  lua_State *L = ls->L;
  FuncState *fs = ls->fs;
  Proto *f = fs->f;
  // 生成最终的返回指令
  luaK_ret(fs, luaY_nvarstack(fs), 0);  /* final return */
  leaveblock(fs);
  lua_assert(fs->bl == NULL);
  luaK_finish(fs);
  // 释放多余的空间
  luaM_shrinkvector(L, f->code, f->sizecode, fs->pc, Instruction);
  luaM_shrinkvector(L, f->lineinfo, f->sizelineinfo, fs->pc, ls_byte);
  luaM_shrinkvector(L, f->abslineinfo, f->sizeabslineinfo,
                       fs->nabslineinfo, AbsLineInfo);
  luaM_shrinkvector(L, f->k, f->sizek, fs->nk, TValue);
  luaM_shrinkvector(L, f->p, f->sizep, fs->np, Proto *);
  luaM_shrinkvector(L, f->locvars, f->sizelocvars, fs->ndebugvars, LocVar);
  luaM_shrinkvector(L, f->upvalues, f->sizeupvalues, fs->nups, Upvaldesc);
  ls->fs = fs->prev;
  luaC_checkGC(L);
}

/*============================================================*/
/* GRAMMAR RULES */
/*============================================================*/

/*
** check whether current token is in the follow set of a block.
** 'until' closes syntactical blocks, but do not close scope,
** so it is handled in separate.
*/
static int block_follow (LexState *ls, int withuntil) {
  switch (ls->t.token) {
    case TK_ELSE: case TK_ELSEIF:
    case TK_END: case TK_EOS:
      return 1;
    case TK_UNTIL: return withuntil;
    default: return 0;
  }
}

static void statlist (LexState *ls) {
  /* statlist -> { stat [';'] } */
  while (!block_follow(ls, 1)) {
    if (ls->t.token == TK_RETURN) {
      statement(ls);
      return;  /* 'return' must be last statement */
    }
    statement(ls);
  }
}

static void fieldsel (LexState *ls, expdesc *v) {
  /* fieldsel -> ['.' | ':'] NAME */
  FuncState *fs = ls->fs;
  expdesc key;
  luaK_exp2anyregup(fs, v);
  luaX_next(ls);  /* skip the dot or colon */
  codename(ls, &key);
  luaK_indexed(fs, v, &key);
}

static void yindex (LexState *ls, expdesc *v) {
  /* index -> '[' expr ']' */
  luaX_next(ls);  /* skip the '[' */
  expr(ls, v);
  luaK_exp2val(ls->fs, v);
  checknext(ls, ']');
}

/*
** {======================================================================
** Rules for Constructors
** =======================================================================
*/

typedef struct ConsControl {
  expdesc v;  /* last list item read */
  expdesc *t;  /* table descriptor */
  int nh;  /* total number of 'record' elements */
  int na;  /* number of array elements already stored */
  int tostore;  /* number of array elements pending to be stored */
} ConsControl;

static void recfield (LexState *ls, ConsControl *cc) {
  /* recfield -> (NAME | '['exp']') = exp */
  FuncState *fs = ls->fs;
  int reg = ls->fs->freereg;
  expdesc tab, key, val;
  if (ls->t.token == TK_NAME) {
    checklimit(fs, cc->nh, MAX_INT, "items in a constructor");
    codename(ls, &key);
  }
  else  /* ls->t.token == '[' */
    # 调用函数 yindex，传入 ls 和 key 的地址
    yindex(ls, &key);
  # cc 结构体中的 nh 属性自增1
  cc->nh++;
  # 检查下一个标记是否为 '='
  checknext(ls, '=');
  # 将 cc 结构体中的 t 指针所指向的表赋值给 tab
  tab = *cc->t;
  # 调用函数 luaK_indexed，传入 fs、tab 和 key 的地址
  luaK_indexed(fs, &tab, &key);
  # 调用函数 expr，传入 ls 和 val 的地址
  expr(ls, &val);
  # 调用函数 luaK_storevar，传入 fs、tab 和 val 的地址
  luaK_storevar(fs, &tab, &val);
  # 设置 fs 结构体中的 freereg 属性为 reg，释放寄存器
  fs->freereg = reg;  /* free registers */
static void closelistfield (FuncState *fs, ConsControl *cc) {
  // 如果当前列表项为空，则直接返回
  if (cc->v.k == VVOID) return;  /* there is no list item */
  // 将列表项的值存储到下一个寄存器中
  luaK_exp2nextreg(fs, &cc->v);
  // 将列表项的值置为无效
  cc->v.k = VVOID;
  // 如果待存储的列表项数量达到一定值，则执行一次列表项的存储操作
  if (cc->tostore == LFIELDS_PER_FLUSH) {
    luaK_setlist(fs, cc->t->u.info, cc->na, cc->tostore);  /* flush */
    // 更新已存储的列表项数量
    cc->na += cc->tostore;
    // 重置待存储的列表项数量
    cc->tostore = 0;  /* no more items pending */
  }
}


static void lastlistfield (FuncState *fs, ConsControl *cc) {
  // 如果待存储的列表项数量为0，则直接返回
  if (cc->tostore == 0) return;
  // 如果列表项的值为多返回值，则设置多返回值标记
  if (hasmultret(cc->v.k)) {
    luaK_setmultret(fs, &cc->v);
    // 执行一次多返回值的列表项存储操作
    luaK_setlist(fs, cc->t->u.info, cc->na, LUA_MULTRET);
    // 更新已存储的列表项数量，不计算最后一个表达式（元素数量未知）
    cc->na--;  /* do not count last expression (unknown number of elements) */
  }
  else {
    // 如果列表项的值不为空，则将其存储到下一个寄存器中
    if (cc->v.k != VVOID)
      luaK_exp2nextreg(fs, &cc->v);
    // 执行一次列表项的存储操作
    luaK_setlist(fs, cc->t->u.info, cc->na, cc->tostore);
  }
  // 更新已存储的列表项数量
  cc->na += cc->tostore;
}


static void listfield (LexState *ls, ConsControl *cc) {
  // 列表项 -> 表达式
  // 解析表达式
  expr(ls, &cc->v);
  // 增加待存储的列表项数量
  cc->tostore++;
}


static void field (LexState *ls, ConsControl *cc) {
  // 字段 -> 列表项 | 记录字段
  switch(ls->t.token) {
    case TK_NAME: {  /* 可能是 '列表项' 或 '记录字段' */
      // 如果下一个标记不是'='，则为表达式
      if (luaX_lookahead(ls) != '=')  /* expression? */
        listfield(ls, cc);
      else
        recfield(ls, cc);
      break;
    }
    case '[': {
      // 解析记录字段
      recfield(ls, cc);
      break;
    }
    default: {
      // 解析列表项
      listfield(ls, cc);
      break;
    }
  }
}


static void constructor (LexState *ls, expdesc *t) {
  // 构造器 -> '{' [ 字段 { 分隔符 字段 } [分隔符] ] '}'
  // 分隔符 -> ',' | ';'
  FuncState *fs = ls->fs;
  int line = ls->linenumber;
  // 生成一个新的表指令，并返回其索引
  int pc = luaK_codeABC(fs, OP_NEWTABLE, 0, 0, 0);
  ConsControl cc;
  // 生成一个空的指令，用于存储额外的参数
  luaK_code(fs, 0);  /* space for extra arg. */
  // 初始化列表控制结构
  cc.na = cc.nh = cc.tostore = 0;
  cc.t = t;
  // 初始化表达式，将表放置在栈顶
  init_exp(t, VNONRELOC, fs->freereg);  /* table will be at stack top */
  // 预留一个寄存器
  luaK_reserveregs(fs, 1);
  // 初始化表达式，值为空（尚未赋值）
  init_exp(&cc.v, VVOID, 0);  /* no value (yet) */
  // 检查下一个标记是否为'{'
  checknext(ls, '{');
  do {
    // 断言表达式的值为空或待存储的列表项数量大于0
    lua_assert(cc.v.k == VVOID || cc.tostore > 0);
    # 如果当前标记为 '}'，则跳出循环
    if (ls->t.token == '}') break;
    # 关闭列表字段
    closelistfield(fs, &cc);
    # 解析字段
    field(ls, &cc);
  # 循环直到遇到 ',' 或 ';'，或者直到遇到 '}'
  } while (testnext(ls, ',') || testnext(ls, ';'));
  # 检查匹配的符号是否为 '}'
  check_match(ls, '}', '{', line);
  # 最后一个列表字段
  lastlistfield(fs, &cc);
  # 设置表大小
  luaK_settablesize(fs, pc, t->u.info, cc.na, cc.nh);
/* }====================================================================== */

// 设置函数的可变参数标志，并生成相应的指令
static void setvararg (FuncState *fs, int nparams) {
  fs->f->is_vararg = 1;
  luaK_codeABC(fs, OP_VARARGPREP, nparams, 0, 0);
}

// 解析参数列表
static void parlist (LexState *ls) {
  /* parlist -> [ {NAME ','} (NAME | '...') ] */
  FuncState *fs = ls->fs;
  Proto *f = fs->f;
  int nparams = 0;
  int isvararg = 0;
  if (ls->t.token != ')') {  /* is 'parlist' not empty? */
    do {
      switch (ls->t.token) {
        case TK_NAME: {
          new_localvar(ls, str_checkname(ls));
          nparams++;
          break;
        }
        case TK_DOTS: {
          luaX_next(ls);
          isvararg = 1;
          break;
        }
        default: luaX_syntaxerror(ls, "<name> or '...' expected");
      }
    } while (!isvararg && testnext(ls, ','));
  }
  adjustlocalvars(ls, nparams);
  f->numparams = cast_byte(fs->nactvar);
  if (isvararg)
    setvararg(fs, f->numparams);  /* declared vararg */
  luaK_reserveregs(fs, fs->nactvar);  /* reserve registers for parameters */
}

// 解析函数体
static void body (LexState *ls, expdesc *e, int ismethod, int line) {
  /* body ->  '(' parlist ')' block END */
  FuncState new_fs;
  BlockCnt bl;
  new_fs.f = addprototype(ls);
  new_fs.f->linedefined = line;
  open_func(ls, &new_fs, &bl);
  checknext(ls, '(');
  if (ismethod) {
    new_localvarliteral(ls, "self");  /* create 'self' parameter */
    adjustlocalvars(ls, 1);
  }
  parlist(ls);
  checknext(ls, ')');
  statlist(ls);
  new_fs.f->lastlinedefined = ls->linenumber;
  check_match(ls, TK_END, TK_FUNCTION, line);
  codeclosure(ls, e);
  close_func(ls);
}

// 解析表达式列表
static int explist (LexState *ls, expdesc *v) {
  /* explist -> expr { ',' expr } */
  int n = 1;  /* at least one expression */
  expr(ls, v);
  while (testnext(ls, ',')) {
    luaK_exp2nextreg(ls->fs, v);
    expr(ls, v);
    n++;
  }
  return n;
}
# 解析函数参数
static void funcargs (LexState *ls, expdesc *f, int line) {
  # 获取当前函数状态
  FuncState *fs = ls->fs;
  # 创建表达式描述对象
  expdesc args;
  int base, nparams;
  # 根据不同情况进行处理
  switch (ls->t.token) {
    case '(': {  /* funcargs -> '(' [ explist ] ')' */
      # 移动到下一个 token
      luaX_next(ls);
      # 如果参数列表为空
      if (ls->t.token == ')')  
        args.k = VVOID;
      else {
        # 解析参数列表
        explist(ls, &args);
        # 如果参数列表包含多个返回值
        if (hasmultret(args.k))
          luaK_setmultret(fs, &args);
      }
      # 检查括号匹配
      check_match(ls, ')', '(', line);
      break;
    }
    case '{': {  /* funcargs -> constructor */
      # 解析构造器
      constructor(ls, &args);
      break;
    }
    case TK_STRING: {  /* funcargs -> STRING */
      # 将字符串转换为常量
      codestring(&args, ls->t.seminfo.ts);
      # 移动到下一个 token
      luaX_next(ls);  /* must use 'seminfo' before 'next' */
      break;
    }
    default: {
      # 报错，期望函数参数
      luaX_syntaxerror(ls, "function arguments expected");
    }
  }
  # 断言函数描述对象的类型为非重定位
  lua_assert(f->k == VNONRELOC);
  # 获取调用的基本寄存器
  base = f->u.info;  /* base register for call */
  # 如果参数列表包含多个返回值
  if (hasmultret(args.k))
    nparams = LUA_MULTRET;  /* open call */
  else {
    # 如果参数列表不为空
    if (args.k != VVOID)
      # 将参数列表中的表达式转换为下一个寄存器
      luaK_exp2nextreg(fs, &args);  /* close last argument */
    # 计算参数个数
    nparams = fs->freereg - (base+1);
  }
  # 初始化表达式描述对象
  init_exp(f, VCALL, luaK_codeABC(fs, OP_CALL, base, nparams+1, 2));
  # 固定行号
  luaK_fixline(fs, line);
  # 释放寄存器
  fs->freereg = base+1;  /* call remove function and arguments and leaves
                            (unless changed) one result */
}

# 表达式解析
static void primaryexp (LexState *ls, expdesc *v) {
  # 根据不同情况进行处理
  switch (ls->t.token) {
    case '(': {
      # 记录行号
      int line = ls->linenumber;
      # 移动到下一个 token
      luaX_next(ls);
      # 解析表达式
      expr(ls, v);
      # 检查括号匹配
      check_match(ls, ')', '(', line);
      # 释放变量
      luaK_dischargevars(ls->fs, v);
      return;
    }
    case TK_NAME: {
      # 解析单变量
      singlevar(ls, v);
      return;
    }
    default: {
      # 报错，遇到意外符号
      luaX_syntaxerror(ls, "unexpected symbol");
    }
  }
}
static void suffixedexp (LexState *ls, expdesc *v) {
  /* 定义一个函数，用于处理带后缀的表达式
     其中后缀可以是：primaryexp { '.' NAME | '[' exp ']' | ':' NAME funcargs | funcargs } */
  FuncState *fs = ls->fs;  // 获取词法状态中的函数状态
  int line = ls->linenumber;  // 获取当前行号
  primaryexp(ls, v);  // 调用 primaryexp 函数处理表达式的主体部分
  for (;;) {
    switch (ls->t.token) {
      case '.': {  /* fieldsel */
        fieldsel(ls, v);  // 处理字段选择
        break;
      }
      case '[': {  /* '[' exp ']' */
        expdesc key;  // 定义一个表达式描述符用于存储键
        luaK_exp2anyregup(fs, v);  // 将表达式转换为寄存器索引
        yindex(ls, &key);  // 处理索引
        luaK_indexed(fs, v, &key);  // 处理索引表达式
        break;
      }
      case ':': {  /* ':' NAME funcargs */
        expdesc key;  // 定义一个表达式描述符用于存储键
        luaX_next(ls);  // 获取下一个词法单元
        codename(ls, &key);  // 处理名称
        luaK_self(fs, v, &key);  // 处理 self 表达式
        funcargs(ls, v, line);  // 处理函数参数
        break;
      }
      case '(': case TK_STRING: case '{': {  /* funcargs */
        luaK_exp2nextreg(fs, v);  // 将表达式转换为下一个寄存器索引
        funcargs(ls, v, line);  // 处理函数参数
        break;
      }
      default: return;  // 默认情况下返回
    }
  }
}


static void simpleexp (LexState *ls, expdesc *v) {
  /* 定义一个函数，用于处理简单表达式
     其中简单表达式可以是：FLT | INT | STRING | NIL | TRUE | FALSE | ... | constructor | FUNCTION body | suffixedexp */
  switch (ls->t.token) {
    case TK_FLT: {
      init_exp(v, VKFLT, 0);  // 初始化表达式描述符为浮点数类型
      v->u.nval = ls->t.seminfo.r;  // 将词法状态中的浮点数值赋给表达式描述符
      break;
    }
    case TK_INT: {
      init_exp(v, VKINT, 0);  // 初始化表达式描述符为整数类型
      v->u.ival = ls->t.seminfo.i;  // 将词法状态中的整数值赋给表达式描述符
      break;
    }
    case TK_STRING: {
      codestring(v, ls->t.seminfo.ts);  // 处理字符串
      break;
    }
    case TK_NIL: {
      init_exp(v, VNIL, 0);  // 初始化表达式描述符为 nil 类型
      break;
    }
    case TK_TRUE: {
      init_exp(v, VTRUE, 0);  // 初始化表达式描述符为 true 类型
      break;
    }
    case TK_FALSE: {
      init_exp(v, VFALSE, 0);  // 初始化表达式描述符为 false 类型
      break;
    }
    case TK_DOTS: {  /* vararg */
      FuncState *fs = ls->fs;  // 获取词法状态中的函数状态
      check_condition(ls, fs->f->is_vararg, "cannot use '...' outside a vararg function");  // 检查是否在可变参数函数之外使用 '...'
      init_exp(v, VVARARG, luaK_codeABC(fs, OP_VARARG, 0, 0, 1));  // 初始化表达式描述符为可变参数类型
      break;
    }
    case '{': {  /* constructor */
      constructor(ls, v);  // 处理构造器
      return;
    }
    # 如果当前 token 是函数关键字
    case TK_FUNCTION: {
      # 获取下一个 token
      luaX_next(ls);
      # 解析函数体
      body(ls, v, 0, ls->linenumber);
      # 返回
      return;
    }
    # 如果不是函数关键字
    default: {
      # 解析后缀表达式
      suffixedexp(ls, v);
      # 返回
      return;
    }
  }
  # 获取下一个 token
  luaX_next(ls);
# 获取一元操作符对应的操作码
static UnOpr getunopr (int op) {
  switch (op) {
    case TK_NOT: return OPR_NOT;  # 如果操作符是 TK_NOT，则返回 OPR_NOT
    case '-': return OPR_MINUS;   # 如果操作符是 '-'，则返回 OPR_MINUS
    case '~': return OPR_BNOT;    # 如果操作符是 '~'，则返回 OPR_BNOT
    case '#': return OPR_LEN;     # 如果操作符是 '#'，则返回 OPR_LEN
    default: return OPR_NOUNOPR;  # 如果操作符不在以上情况，则返回 OPR_NOUNOPR
  }
}

# 获取二元操作符对应的操作码
static BinOpr getbinopr (int op) {
  switch (op) {
    case '+': return OPR_ADD;     # 如果操作符是 '+'，则返回 OPR_ADD
    case '-': return OPR_SUB;     # 如果操作符是 '-'，则返回 OPR_SUB
    case '*': return OPR_MUL;     # 如果操作符是 '*'，则返回 OPR_MUL
    case '%': return OPR_MOD;     # 如果操作符是 '%'，则返回 OPR_MOD
    case '^': return OPR_POW;     # 如果操作符是 '^'，则返回 OPR_POW
    case '/': return OPR_DIV;     # 如果操作符是 '/'，则返回 OPR_DIV
    case TK_IDIV: return OPR_IDIV; # 如果操作符是 TK_IDIV，则返回 OPR_IDIV
    case '&': return OPR_BAND;     # 如果操作符是 '&'，则返回 OPR_BAND
    case '|': return OPR_BOR;      # 如果操作符是 '|'，则返回 OPR_BOR
    case '~': return OPR_BXOR;     # 如果操作符是 '~'，则返回 OPR_BXOR
    case TK_SHL: return OPR_SHL;   # 如果操作符是 TK_SHL，则返回 OPR_SHL
    case TK_SHR: return OPR_SHR;   # 如果操作符是 TK_SHR，则返回 OPR_SHR
    case TK_CONCAT: return OPR_CONCAT;  # 如果操作符是 TK_CONCAT，则返回 OPR_CONCAT
    case TK_NE: return OPR_NE;     # 如果操作符是 TK_NE，则返回 OPR_NE
    case TK_EQ: return OPR_EQ;     # 如果操作符是 TK_EQ，则返回 OPR_EQ
    case '<': return OPR_LT;      # 如果操作符是 '<'，则返回 OPR_LT
    case TK_LE: return OPR_LE;     # 如果操作符是 TK_LE，则返回 OPR_LE
    case '>': return OPR_GT;      # 如果操作符是 '>'，则返回 OPR_GT
    case TK_GE: return OPR_GE;     # 如果操作符是 TK_GE，则返回 OPR_GE
    case TK_AND: return OPR_AND;   # 如果操作符是 TK_AND，则返回 OPR_AND
    case TK_OR: return OPR_OR;     # 如果操作符是 TK_OR，则返回 OPR_OR
    default: return OPR_NOBINOPR;  # 如果操作符不在以上情况，则返回 OPR_NOBINOPR
  }
}

# 二元操作符的优先级表
static const struct {
  lu_byte left;  /* 每个二元操作符的左优先级 */
  lu_byte right; /* 右优先级 */
} priority[] = {  /* ORDER OPR */
   {10, 10}, {10, 10},           /* '+' '-' */
   {11, 11}, {11, 11},           /* '*' '%' */
   {14, 13},                  /* '^' (右结合) */
   {11, 11}, {11, 11},           /* '/' '//' */
   {6, 6}, {4, 4}, {5, 5},   /* '&' '|' '~' */
   {7, 7}, {7, 7},           /* '<<' '>>' */
   {9, 8},                   /* '..' (右结合) */
   {3, 3}, {3, 3}, {3, 3},   /* ==, <, <= */
   {3, 3}, {3, 3}, {3, 3},   /* ~=, >, >= */
   {2, 2}, {1, 1}            /* and, or */
};

# 一元操作符的优先级
#define UNARY_PRIORITY    12  # 一元操作符的优先级为 12

# 子表达式 -> (simpleexp | unop subexpr) { binop subexpr }
# 其中 'binop' 是任何优先级高于 'limit' 的二元操作符
static BinOpr subexpr (LexState *ls, expdesc *v, int limit) {
  // 进入新的表达式级别
  enterlevel(ls);
  // 获取一元操作符
  uop = getunopr(ls->t.token);
  // 如果是一元操作符
  if (uop != OPR_NOUNOPR) {  /* prefix (unary) operator? */
    // 记录当前行号
    int line = ls->linenumber;
    // 跳过操作符
    luaX_next(ls);  /* skip operator */
    // 递归处理子表达式
    subexpr(ls, v, UNARY_PRIORITY);
    // 生成前缀表达式的指令
    luaK_prefix(ls->fs, uop, v, line);
  }
  else simpleexp(ls, v);
  // 当操作符的优先级高于给定的限制时，展开表达式
  op = getbinopr(ls->t.token);
  while (op != OPR_NOBINOPR && priority[op].left > limit) {
    expdesc v2;
    BinOpr nextop;
    // 记录当前行号
    int line = ls->linenumber;
    // 跳过操作符
    luaX_next(ls);  /* skip operator */
    // 生成中缀表达式的指令
    luaK_infix(ls->fs, op, v);
    // 读取优先级更高的子表达式
    nextop = subexpr(ls, &v2, priority[op].right);
    // 生成后缀表达式的指令
    luaK_posfix(ls->fs, op, v, &v2, line);
    op = nextop;
  }
  // 离开表达式级别
  leavelevel(ls);
  // 返回第一个未处理的操作符
  return op;
}


static void expr (LexState *ls, expdesc *v) {
  // 处理表达式
  subexpr(ls, v, 0);
}

/* }==================================================================== */



/*
** {======================================================================
** Rules for Statements
** =======================================================================
*/


static void block (LexState *ls) {
  /* block -> statlist */
  // 获取函数状态
  FuncState *fs = ls->fs;
  // 进入新的块级别
  BlockCnt bl;
  enterblock(fs, &bl, 0);
  // 处理语句列表
  statlist(ls);
  // 离开块级别
  leaveblock(fs);
}


/*
** structure to chain all variables in the left-hand side of an
** assignment
*/
struct LHS_assign {
  struct LHS_assign *prev;
  expdesc v;  /* variable (global, local, upvalue, or indexed) */
};


/*
** check whether, in an assignment to an upvalue/local variable, the
** upvalue/local variable is begin used in a previous assignment to a
** table. If so, save original upvalue/local value in a safe place and
** use this safe copy in the previous assignment.
*/
static void check_conflict (LexState *ls, struct LHS_assign *lh, expdesc *v) {
  FuncState *fs = ls->fs;  /* 获取当前词法状态的函数状态 */
  int extra = fs->freereg;  /* 保存局部变量的最终位置 */
  int conflict = 0;  /* 冲突标志位初始化为0 */
  for (; lh; lh = lh->prev) {  /* 遍历所有先前的赋值操作 */
    if (vkisindexed(lh->v.k)) {  /* 是否为对表字段的赋值操作？ */
      if (lh->v.k == VINDEXUP) {  /* 表是否为上值？ */
        if (v->k == VUPVAL && lh->v.u.ind.t == v->u.info) {
          conflict = 1;  /* 表是当前正在赋值的上值 */
          lh->v.k = VINDEXSTR;
          lh->v.u.ind.t = extra;  /* 赋值将使用安全副本 */
        }
      }
      else {  /* 表是一个寄存器 */
        if (v->k == VLOCAL && lh->v.u.ind.t == v->u.var.ridx) {
          conflict = 1;  /* 表是当前正在赋值的局部变量 */
          lh->v.u.ind.t = extra;  /* 赋值将使用安全副本 */
        }
        /* 索引是否为正在赋值的局部变量？ */
        if (lh->v.k == VINDEXED && v->k == VLOCAL &&
            lh->v.u.ind.idx == v->u.var.ridx) {
          conflict = 1;
          lh->v.u.ind.idx = extra;  /* 先前的赋值将使用安全副本 */
        }
      }
    }
  }
  if (conflict) {
    /* 将上值/局部值复制到临时位置（在位置 'extra'） */
    if (v->k == VLOCAL)
      luaK_codeABC(fs, OP_MOVE, extra, v->u.var.ridx, 0);
    else
      luaK_codeABC(fs, OP_GETUPVAL, extra, v->u.info, 0);
    luaK_reserveregs(fs, 1);  /* 预留一个寄存器 */
  }
}

/*
** 解析和编译多重赋值。第一个“变量”（一个'suffixedexp'）已经被调用者读取。
**
** 赋值 -> suffixedexp restassign
** restassign -> ',' suffixedexp restassign | '=' explist
*/
static void restassign (LexState *ls, struct LHS_assign *lh, int nvars) {
  expdesc e;  /* 表达式描述符 */
  check_condition(ls, vkisvar(lh->v.k), "syntax error");  /* 检查条件，如果不是变量则报错 */
  check_readonly(ls, &lh->v);  /* 检查是否为只读变量 */
  if (testnext(ls, ',')) {  /* restassign -> ',' suffixedexp restassign */
    struct LHS_assign nv;  /* 新的左值列表 */
    nv.prev = lh;  /* 设置新的左值列表的前一个元素为lh */
    // 对表达式进行后缀处理
    suffixedexp(ls, &nv.v);
    // 如果表达式不是索引表达式，则检查是否存在冲突
    if (!vkisindexed(nv.v.k))
      check_conflict(ls, lh, &nv.v);
    // 进入一个新的递归深度
    enterlevel(ls);  /* 控制递归深度 */
    // 对剩余的赋值语句进行处理
    restassign(ls, &nv, nvars+1);
    // 离开当前递归深度
    leavelevel(ls);
  }
  else {  /* restassign -> '=' explist */
    int nexps;
    // 检查下一个字符是否为 '='
    checknext(ls, '=');
    // 获取表达式列表的长度
    nexps = explist(ls, &e);
    // 如果表达式列表的长度不等于变量列表的长度，则调整赋值
    if (nexps != nvars)
      adjust_assign(ls, nvars, nexps, &e);
    else {
      // 关闭最后一个表达式
      luaK_setoneret(ls->fs, &e);
      // 存储变量
      luaK_storevar(ls->fs, &lh->v, &e);
      // 避免默认情况
      return;
    }
  }
  // 初始化表达式
  init_exp(&e, VNONRELOC, ls->fs->freereg-1);  /* 默认赋值 */
  // 存储变量
  luaK_storevar(ls->fs, &lh->v, &e);
# 检查条件语句，返回条件表达式的假出口
static int cond (LexState *ls) {
  # 创建一个表达式描述符
  expdesc v;
  # 读取条件表达式
  expr(ls, &v);  /* read condition */
  # 如果条件表达式为nil，则将其转换为false
  if (v.k == VNIL) v.k = VFALSE;  /* 'falses' are all equal here */
  # 生成条件为真时的跳转指令
  luaK_goiftrue(ls->fs, &v);
  # 返回条件表达式的假出口
  return v.f;
}

# 处理goto语句
static void gotostat (LexState *ls) {
  # 获取函数状态
  FuncState *fs = ls->fs;
  # 获取当前行号
  int line = ls->linenumber;
  # 获取标签名
  TString *name = str_checkname(ls);  /* label's name */
  # 查找标签
  Labeldesc *lb = findlabel(ls, name);
  # 如果标签不存在
  if (lb == NULL)  /* no label? */
    # 创建一个新的跳转入口
    newgotoentry(ls, name, line, luaK_jump(fs));
  else {  # 如果找到了标签
    # 获取标签的级别
    int lblevel = reglevel(fs, lb->nactvar);  /* label level */
    # 如果离开了变量的作用域
    if (luaY_nvarstack(fs) > lblevel)  /* leaving the scope of a variable? */
      # 生成关闭指令
      luaK_codeABC(fs, OP_CLOSE, lblevel, 0, 0);
    # 创建跳转并将其链接到标签
    luaK_patchlist(fs, luaK_jump(fs), lb->pc);
  }
}

# 处理break语句
static void breakstat (LexState *ls) {
  # 获取当前行号
  int line = ls->linenumber;
  # 跳过break关键字
  luaX_next(ls);  /* skip break */
  # 创建一个新的跳转入口
  newgotoentry(ls, luaS_newliteral(ls->L, "break"), line, luaK_jump(ls->fs));
}

# 检查是否已经存在具有给定名称的标签
static void checkrepeated (LexState *ls, TString *name) {
  # 查找标签
  Labeldesc *lb = findlabel(ls, name);
  # 如果标签已经定义
  if (l_unlikely(lb != NULL)) {  /* already defined? */
    # 抛出错误信息
    const char *msg = "label '%s' already defined on line %d";
    msg = luaO_pushfstring(ls->L, msg, getstr(name), lb->line);
    luaK_semerror(ls, msg);  /* error */
  }
}

# 处理标签语句
static void labelstat (LexState *ls, TString *name, int line) {
  # 跳过双冒号
  checknext(ls, TK_DBCOLON);  /* skip double colon */
  # 跳过其他无操作语句
  while (ls->t.token == ';' || ls->t.token == TK_DBCOLON)
    statement(ls);  /* skip other no-op statements */
  # 检查是否存在重复的标签
  checkrepeated(ls, name);  /* check for repeated labels */
  # 创建标签
  createlabel(ls, name, line, block_follow(ls, 0));
}
static void whilestat (LexState *ls, int line) {
  /* whilestat -> WHILE cond DO block END */
  // 定义一个指向词法状态的指针
  FuncState *fs = ls->fs;
  // 初始化 while 循环的标签
  int whileinit;
  // 条件退出标签
  int condexit;
  // 定义一个块计数器
  BlockCnt bl;
  // 跳过 WHILE 关键字
  luaX_next(ls);  /* skip WHILE */
  // 获取 while 循环的起始标签
  whileinit = luaK_getlabel(fs);
  // 获取条件退出标签
  condexit = cond(ls);
  // 进入一个新的块
  enterblock(fs, &bl, 1);
  // 检查并跳过 DO 关键字
  checknext(ls, TK_DO);
  // 解析并执行 block
  block(ls);
  // 在条件退出标签处跳转到 whileinit 标签
  luaK_jumpto(fs, whileinit);
  // 检查并匹配 END 关键字
  check_match(ls, TK_END, TK_WHILE, line);
  // 离开当前块
  leaveblock(fs);
  // 在条件退出标签处打补丁，用于结束循环
  luaK_patchtohere(fs, condexit);  /* false conditions finish the loop */
}


static void repeatstat (LexState *ls, int line) {
  /* repeatstat -> REPEAT block UNTIL cond */
  // 条件退出标签
  int condexit;
  // 定义一个指向词法状态的指针
  FuncState *fs = ls->fs;
  // 获取 repeat 循环的起始标签
  int repeat_init = luaK_getlabel(fs);
  // 定义两个块计数器
  BlockCnt bl1, bl2;
  // 进入一个新的块，用于循环
  enterblock(fs, &bl1, 1);  /* loop block */
  // 进入一个新的块，用于作用域
  enterblock(fs, &bl2, 0);  /* scope block */
  // 跳过 REPEAT 关键字
  luaX_next(ls);  /* skip REPEAT */
  // 解析并执行 block
  statlist(ls);
  // 检查并匹配 UNTIL 关键字
  check_match(ls, TK_UNTIL, TK_REPEAT, line);
  // 读取条件（在作用域块内）
  condexit = cond(ls);
  // 结束作用域块
  leaveblock(fs);
  // 如果存在上值
  if (bl2.upval) {  /* upvalues? */
    // 正常退出必须跳过修复
    int exit = luaK_jump(fs);  /* normal exit must jump over fix */
    // 重复必须关闭上值
    luaK_patchtohere(fs, condexit);  /* repetition must close upvalues */
    // 生成关闭上值的指令
    luaK_codeABC(fs, OP_CLOSE, reglevel(fs, bl2.nactvar), 0, 0);
    // 重复关闭上值后跳转
    condexit = luaK_jump(fs);  /* repeat after closing upvalues */
    // 正常退出跳转到这里
    luaK_patchtohere(fs, exit);  /* normal exit comes to here */
  }
  // 关闭循环
  luaK_patchlist(fs, condexit, repeat_init);  /* close the loop */
  // 结束循环
  leaveblock(fs);  /* finish loop */
}


/*
** 读取一个表达式并生成代码，将其结果放入下一个堆栈位置
**
*/
static void exp1 (LexState *ls) {
  // 定义一个表达式描述符
  expdesc e;
  // 解析表达式
  expr(ls, &e);
  // 将表达式转换为下一个寄存器
  luaK_exp2nextreg(ls->fs, &e);
  // 断言表达式类型为 VNONRELOC
  lua_assert(e.k == VNONRELOC);
}


/*
** 修复位置 'pc' 处的指令以跳转到 'dest' 处
** （Lua 中跳转地址是相对的）。'back' 为 true 表示是一个回跳。
*/
static void fixforjump (FuncState *fs, int pc, int dest, int back) {
  // 获取指令的地址
  Instruction *jmp = &fs->f->code[pc];
  // 计算偏移量
  int offset = dest - (pc + 1);
  // 如果是回跳
  if (back)
    # 将偏移量取反
    offset = -offset;
  # 如果偏移量大于最大的无符号整数表示范围
  if (l_unlikely(offset > MAXARG_Bx))
    # 抛出语法错误，控制结构太长
    luaX_syntaxerror(fs->ls, "control structure too long");
  # 设置指令参数为偏移量
  SETARG_Bx(*jmp, offset);
# 生成一个 'for' 循环的代码
static void forbody (LexState *ls, int base, int line, int nvars, int isgen) {
  # forbody -> DO block
  # 定义两种不同类型的 for 循环的预处理和循环操作码
  static const OpCode forprep[2] = {OP_FORPREP, OP_TFORPREP};
  static const OpCode forloop[2] = {OP_FORLOOP, OP_TFORLOOP};
  # 定义一个块结构体
  BlockCnt bl;
  # 获取函数状态
  FuncState *fs = ls->fs;
  int prep, endfor;
  # 检查下一个标记是否为 DO
  checknext(ls, TK_DO);
  # 生成 for 循环的预处理操作码
  prep = luaK_codeABx(fs, forprep[isgen], base, 0);
  # 进入一个新的块
  enterblock(fs, &bl, 0);  # scope for declared variables
  # 调整局部变量
  adjustlocalvars(ls, nvars);
  # 预留寄存器
  luaK_reserveregs(fs, nvars);
  # 进入块
  block(ls);
  # 离开块，结束声明变量的作用域
  leaveblock(fs);  # end of scope for declared variables
  # 修正 for 循环的跳转
  fixforjump(fs, prep, luaK_getlabel(fs), 0);
  if (isgen) {  # generic for?
    # 生成通用 for 循环的调用操作码
    luaK_codeABC(fs, OP_TFORCALL, base, 0, nvars);
    # 修正行号
    luaK_fixline(fs, line);
  }
  # 生成 for 循环的循环操作码
  endfor = luaK_codeABx(fs, forloop[isgen], base, 0);
  # 修正 for 循环的跳转
  fixforjump(fs, endfor, prep + 1, 1);
  # 修正行号
  luaK_fixline(fs, line);
}

# 生成一个数值型的 'for' 循环的代码
static void fornum (LexState *ls, TString *varname, int line) {
  # fornum -> NAME = exp,exp[,exp] forbody
  # 获取函数状态
  FuncState *fs = ls->fs;
  # 获取基础寄存器
  int base = fs->freereg;
  # 创建一个新的局部变量
  new_localvarliteral(ls, "(for state)");
  new_localvarliteral(ls, "(for state)");
  new_localvarliteral(ls, "(for state)");
  new_localvar(ls, varname);
  # 检查下一个标记是否为等号
  checknext(ls, '=');
  # 解析表达式1
  exp1(ls);  # initial value
  # 检查下一个标记是否为逗号
  checknext(ls, ',');
  # 解析表达式2
  exp1(ls);  # limit
  # 如果下一个标记是逗号
  if (testnext(ls, ','))
    # 解析表达式3
    exp1(ls);  # optional step
  else {  # default step = 1
    # 将默认步长设置为1
    luaK_int(fs, fs->freereg, 1);
    # 预留寄存器
    luaK_reserveregs(fs, 1);
  }
  # 调整局部变量
  adjustlocalvars(ls, 3);  # control variables
  # 生成 for 循环的代码
  forbody(ls, base, line, 1, 0);
}
static void forlist (LexState *ls, TString *indexname) {
  /* forlist -> NAME {,NAME} IN explist forbody */
  # 定义一个名为 forlist 的静态函数，接受 LexState 和 TString 类型的参数
  FuncState *fs = ls->fs;
  # 从 LexState 中获取 FuncState 对象
  expdesc e;
  int nvars = 5;  /* gen, state, control, toclose, 'indexname' */
  # 初始化变量 nvars 为 5
  int line;
  int base = fs->freereg;
  # 从 fs 中获取 freereg 属性，赋值给 base
  /* create control variables */
  new_localvarliteral(ls, "(for state)");
  new_localvarliteral(ls, "(for state)");
  new_localvarliteral(ls, "(for state)");
  new_localvarliteral(ls, "(for state)");
  # 创建四个控制变量
  /* create declared variables */
  new_localvar(ls, indexname);
  # 创建声明的变量 indexname
  while (testnext(ls, ',')) {
    new_localvar(ls, str_checkname(ls));
    nvars++;
  }
  # 当遇到逗号时，创建新的本地变量，并递增 nvars
  checknext(ls, TK_IN);
  # 检查下一个 token 是否为 TK_IN
  line = ls->linenumber;
  # 获取当前行号，赋值给 line
  adjust_assign(ls, 4, explist(ls, &e), &e);
  # 调整赋值，传入参数 ls, 4, explist(ls, &e), &e
  adjustlocalvars(ls, 4);  /* control variables */
  # 调整本地变量，传入参数 ls, 4
  marktobeclosed(fs);  /* last control var. must be closed */
  # 标记为即将关闭的变量
  luaK_checkstack(fs, 3);  /* extra space to call generator */
  # 检查栈空间是否足够，传入参数 fs, 3
  forbody(ls, base, line, nvars - 4, 1);
  # 调用 forbody 函数，传入参数 ls, base, line, nvars - 4, 1
}


static void forstat (LexState *ls, int line) {
  /* forstat -> FOR (fornum | forlist) END */
  # 定义一个名为 forstat 的静态函数，接受 LexState 和 int 类型的参数
  FuncState *fs = ls->fs;
  # 从 LexState 中获取 FuncState 对象
  TString *varname;
  BlockCnt bl;
  enterblock(fs, &bl, 1);  /* scope for loop and control variables */
  # 进入一个新的代码块，传入参数 fs, &bl, 1
  luaX_next(ls);  /* skip 'for' */
  # 跳过 'for' 关键字
  varname = str_checkname(ls);  /* first variable name */
  # 获取第一个变量名
  switch (ls->t.token) {
    case '=': fornum(ls, varname, line); break;
    # 如果 token 为 '=', 调用 fornum 函数，传入参数 ls, varname, line
    case ',': case TK_IN: forlist(ls, varname); break;
    # 如果 token 为 ',' 或 TK_IN，调用 forlist 函数，传入参数 ls, varname
    default: luaX_syntaxerror(ls, "'=' or 'in' expected");
    # 否则，抛出语法错误
  }
  check_match(ls, TK_END, TK_FOR, line);
  # 检查匹配，传入参数 ls, TK_END, TK_FOR, line
  leaveblock(fs);  /* loop scope ('break' jumps to this point) */
  # 离开当前代码块
}


static void test_then_block (LexState *ls, int *escapelist) {
  /* test_then_block -> [IF | ELSEIF] cond THEN block */
  # 定义一个名为 test_then_block 的静态函数，接受 LexState 和 int* 类型的参数
  BlockCnt bl;
  FuncState *fs = ls->fs;
  expdesc v;
  int jf;  /* instruction to skip 'then' code (if condition is false) */
  # 定义一个整型变量 jf
  luaX_next(ls);  /* skip IF or ELSEIF */
  # 跳过 IF 或 ELSEIF 关键字
  expr(ls, &v);  /* read condition */
  # 读取条件表达式
  checknext(ls, TK_THEN);
  # 检查下一个 token 是否为 TK_THEN
  if (ls->t.token == TK_BREAK) {  /* 'if x then break' ? */
    int line = ls->linenumber;
    # 如果 token 为 TK_BREAK，获取当前行号，赋值给 line
    luaK_goiffalse(ls->fs, &v);  /* 如果条件为真，则跳转 */
    luaX_next(ls);  /* 跳过 'break' */
    enterblock(fs, &bl, 0);  /* 必须在 'goto' 之前进入块 */
    newgotoentry(ls, luaS_newliteral(ls->L, "break"), line, v.t);  /* 创建新的 'break' 条目 */
    while (testnext(ls, ';')) {}  /* 跳过分号 */
    if (block_follow(ls, 0)) {  /* 跳转到整个块？ */
      leaveblock(fs);
      return;  /* 就是这样 */
    }
    else  /* 如果条件为假，必须跳过 'then' 部分 */
      jf = luaK_jump(fs);
  }
  else {  /* 常规情况（不是 'break'） */
    luaK_goiftrue(ls->fs, &v);  /* 如果条件为假，则跳过块 */
    enterblock(fs, &bl, 0);
    jf = v.f;
  }
  statlist(ls);  /* 'then' 部分 */
  leaveblock(fs);
  if (ls->t.token == TK_ELSE ||
      ls->t.token == TK_ELSEIF)  /* 后面跟着 'else'/'elseif'？ */
    luaK_concat(fs, escapelist, luaK_jump(fs));  /* 必须跳过它 */
  luaK_patchtohere(fs, jf);
# 解析 if 语句，包括条件、执行块和可能的 else if 和 else 块
static void ifstat (LexState *ls, int line) {
  # 获取当前函数状态
  FuncState *fs = ls->fs;
  # 用于记录已完成部分的退出列表
  int escapelist = NO_JUMP;
  # 解析并生成条件判断和执行块
  test_then_block(ls, &escapelist);  /* IF cond THEN block */
  # 解析并生成可能的 else if 条件和执行块
  while (ls->t.token == TK_ELSEIF)
    test_then_block(ls, &escapelist);  /* ELSEIF cond THEN block */
  # 解析并生成 else 块
  if (testnext(ls, TK_ELSE))
    block(ls);  /* 'else' part */
  # 检查 if 语句结束
  check_match(ls, TK_END, TK_IF, line);
  # 将退出列表链接到 if 语句结束处
  luaK_patchtohere(fs, escapelist);  /* patch escape list to 'if' end */
}


# 解析局部函数
static void localfunc (LexState *ls) {
  # 临时表达式描述符
  expdesc b;
  # 获取当前函数状态
  FuncState *fs = ls->fs;
  # 函数的变量索引
  int fvar = fs->nactvar;
  # 创建新的局部变量
  new_localvar(ls, str_checkname(ls));
  # 调整局部变量
  adjustlocalvars(ls, 1);
  # 进入函数作用域，创建函数在下一个寄存器中
  body(ls, &b, 0, ls->linenumber);
  # 调试信息只能在此处看到变量
  localdebuginfo(fs, fvar)->startpc = fs->pc;
}


# 获取局部变量属性
static int getlocalattribute (LexState *ls) {
  # 解析属性，包括 <Name> 形式
  if (testnext(ls, '<')) {
    const char *attr = getstr(str_checkname(ls));
    checknext(ls, '>');
    # 检查属性并返回相应的标识
    if (strcmp(attr, "const") == 0)
      return RDKCONST;  /* read-only variable */
    else if (strcmp(attr, "close") == 0)
      return RDKTOCLOSE;  /* to-be-closed variable */
    else
      luaK_semerror(ls,
        luaO_pushfstring(ls->L, "unknown attribute '%s'", attr));
  }
  # 返回常规变量标识
  return VDKREG;  /* regular variable */
}


# 检查是否需要关闭变量
static void checktoclose (FuncState *fs, int level) {
  # 是否存在需要关闭的变量
  if (level != -1) {
    # 标记需要关闭的变量
    marktobeclosed(fs);
    # 生成关闭指令
    luaK_codeABC(fs, OP_TBC, reglevel(fs, level), 0, 0);
  }
}
# 定义一个函数，用于处理局部变量声明
static void localstat (LexState *ls) {
  FuncState *fs = ls->fs;  # 获取词法状态中的函数状态
  int toclose = -1;  # 将要关闭的变量的索引（如果有的话）
  Vardesc *var;  # 最后一个变量
  int vidx, kind;  # 最后一个变量的索引和类型
  int nvars = 0;  # 变量数量
  int nexps;  # 表达式数量
  expdesc e;  # 表达式描述
  do {
    vidx = new_localvar(ls, str_checkname(ls));  # 创建一个新的局部变量
    kind = getlocalattribute(ls);  # 获取局部变量的属性
    getlocalvardesc(fs, vidx)->vd.kind = kind;  # 设置局部变量的类型
    if (kind == RDKTOCLOSE) {  # 是否需要关闭？
      if (toclose != -1)  # 已经有一个需要关闭的变量？
        luaK_semerror(ls, "multiple to-be-closed variables in local list");  # 报错，局部列表中有多个需要关闭的变量
      toclose = fs->nactvar + nvars;  # 设置需要关闭的变量的索引
    }
    nvars++;  # 变量数量加一
  } while (testnext(ls, ','));  # 循环直到没有逗号
  if (testnext(ls, '='))  # 是否有赋值操作
    nexps = explist(ls, &e);  # 获取表达式列表
  else {
    e.k = VVOID;  # 表达式为空
    nexps = 0;  # 表达式数量为0
  }
  var = getlocalvardesc(fs, vidx);  # 获取最后一个变量
  if (nvars == nexps &&  # 没有调整？
      var->vd.kind == RDKCONST &&  # 最后一个变量是常量？
      luaK_exp2const(fs, &e, &var->k)) {  # 编译时常量？
    var->vd.kind = RDKCTC;  # 变量是编译时常量
    adjustlocalvars(ls, nvars - 1);  # 排除最后一个变量
    fs->nactvar++;  # 但是计数
  }
  else {
    adjust_assign(ls, nvars, nexps, &e);  # 调整赋值
    adjustlocalvars(ls, nvars);  # 调整局部变量
  }
  checktoclose(fs, toclose);  # 检查需要关闭的变量
}

# 定义一个函数，用于处理函数名
static int funcname (LexState *ls, expdesc *v) {
  int ismethod = 0;  # 是否是方法
  singlevar(ls, v);  # 获取单个变量
  while (ls->t.token == '.')  # 如果是点操作符
    fieldsel(ls, v);  # 处理字段选择
  if (ls->t.token == ':') {  # 如果是冒号操作符
    ismethod = 1;  # 是方法
    fieldsel(ls, v);  # 处理字段选择
  }
  return ismethod;  # 返回是否是方法
}
static void funcstat (LexState *ls, int line) {
  /* 定义函数语句，包括函数名和函数体 */
  int ismethod;
  expdesc v, b;
  luaX_next(ls);  /* 跳过关键字 FUNCTION */
  ismethod = funcname(ls, &v);  // 获取函数名
  body(ls, &b, ismethod, line);  // 解析函数体
  check_readonly(ls, &v);  // 检查函数名是否只读
  luaK_storevar(ls->fs, &v, &b);  // 存储函数名和函数体
  luaK_fixline(ls->fs, line);  /* 函数定义发生在第一行 */
}


static void exprstat (LexState *ls) {
  /* 表达式语句 -> 函数调用 | 赋值语句 */
  FuncState *fs = ls->fs;
  struct LHS_assign v;
  suffixedexp(ls, &v.v);  // 解析表达式
  if (ls->t.token == '=' || ls->t.token == ',') { /* 是否为赋值语句？ */
    v.prev = NULL;
    restassign(ls, &v, 1);  // 解析赋值语句
  }
  else {  /* 函数调用 */
    Instruction *inst;
    check_condition(ls, v.v.k == VCALL, "syntax error");  // 检查条件
    inst = &getinstruction(fs, &v.v);  // 获取指令
    SETARG_C(*inst, 1);  /* 调用语句不使用结果 */
  }
}


static void retstat (LexState *ls) {
  /* 返回语句 -> RETURN [表达式列表] [';'] */
  FuncState *fs = ls->fs;
  expdesc e;
  int nret;  /* 返回的值的数量 */
  int first = luaY_nvarstack(fs);  /* 第一个要返回的槽位 */
  if (block_follow(ls, 1) || ls->t.token == ';')
    nret = 0;  /* 不返回任何值 */
  else {
    nret = explist(ls, &e);  /* 可选的返回值 */
    if (hasmultret(e.k)) {
      luaK_setmultret(fs, &e);
      if (e.k == VCALL && nret == 1 && !fs->bl->insidetbc) {  /* 尾调用？ */
        SET_OPCODE(getinstruction(fs,&e), OP_TAILCALL);
        lua_assert(GETARG_A(getinstruction(fs,&e)) == luaY_nvarstack(fs));
      }
      nret = LUA_MULTRET;  /* 返回所有值 */
    }
    else {
      if (nret == 1)  /* 只有一个返回值？ */
        first = luaK_exp2anyreg(fs, &e);  /* 可以使用原始槽位 */
      else {  /* 值必须放在栈顶 */
        luaK_exp2nextreg(fs, &e);
        lua_assert(nret == fs->freereg - first);
      }
    }
  }
  luaK_ret(fs, first, nret);  // 返回值
  testnext(ls, ';');  /* 跳过可选的分号 */
}
static void statement (LexState *ls) {
  int line = ls->linenumber;  /* 保存当前行号，可能用于错误消息 */
  enterlevel(ls);  /* 进入一个新的语句块 */
  switch (ls->t.token) {  /* 根据当前 token 进行不同的处理 */
    case ';': {  /* 如果是分号，表示空语句 */
      luaX_next(ls);  /* 跳过分号 */
      break;
    }
    case TK_IF: {  /* 如果是 if 语句 */
      ifstat(ls, line);  /* 调用 if 语句处理函数 */
      break;
    }
    case TK_WHILE: {  /* 如果是 while 语句 */
      whilestat(ls, line);  /* 调用 while 语句处理函数 */
      break;
    }
    case TK_DO: {  /* 如果是 do 语句 */
      luaX_next(ls);  /* 跳过 DO */
      block(ls);  /* 处理代码块 */
      check_match(ls, TK_END, TK_DO, line);  /* 检查是否匹配 END 和 DO */
      break;
    }
    case TK_FOR: {  /* 如果是 for 语句 */
      forstat(ls, line);  /* 调用 for 语句处理函数 */
      break;
    }
    case TK_REPEAT: {  /* 如果是 repeat 语句 */
      repeatstat(ls, line);  /* 调用 repeat 语句处理函数 */
      break;
    }
    case TK_FUNCTION: {  /* 如果是 function 语句 */
      funcstat(ls, line);  /* 调用 function 语句处理函数 */
      break;
    }
    case TK_LOCAL: {  /* 如果是 local 语句 */
      luaX_next(ls);  /* 跳过 LOCAL */
      if (testnext(ls, TK_FUNCTION))  /* 如果是 local function */
        localfunc(ls);  /* 调用 local function 处理函数 */
      else
        localstat(ls);  /* 否则调用 local 语句处理函数 */
      break;
    }
    case TK_DBCOLON: {  /* 如果是双冒号，表示标签 */
      luaX_next(ls);  /* 跳过双冒号 */
      labelstat(ls, str_checkname(ls), line);  /* 调用标签处理函数 */
      break;
    }
    case TK_RETURN: {  /* 如果是 return 语句 */
      luaX_next(ls);  /* 跳过 RETURN */
      retstat(ls);  /* 调用 return 语句处理函数 */
      break;
    }
    case TK_BREAK: {  /* 如果是 break 语句 */
      breakstat(ls);  /* 调用 break 语句处理函数 */
      break;
    }
    case TK_GOTO: {  /* 如果是 goto 语句 */
      luaX_next(ls);  /* 跳过 'goto' */
      gotostat(ls);  /* 调用 goto 语句处理函数 */
      break;
    }
    default: {  /* 如果是其他情况，可能是函数调用或赋值语句 */
      exprstat(ls);  /* 调用表达式语句处理函数 */
      break;
    }
  }
  lua_assert(ls->fs->f->maxstacksize >= ls->fs->freereg &&
             ls->fs->freereg >= luaY_nvarstack(ls->fs));  /* 断言，检查栈的最大大小和空闲寄存器数 */
  ls->fs->freereg = luaY_nvarstack(ls->fs);  /* 释放寄存器 */
  leavelevel(ls);  /* 离开当前语句块 */
}

/* }====================================================================== */
/*
** 编译主函数，这是一个常规的可变参数函数，其中包含一个名为 LUA_ENV 的上值
*/
static void mainfunc (LexState *ls, FuncState *fs) {
  BlockCnt bl;  // 定义一个块计数器
  Upvaldesc *env;  // 定义一个上值描述符指针
  open_func(ls, fs, &bl);  // 打开函数
  setvararg(fs, 0);  /* 主函数总是声明为可变参数 */
  env = allocupvalue(fs);  /* ...设置环境上值 */
  env->instack = 1;  // 上值在堆栈中
  env->idx = 0;  // 上值索引为0
  env->kind = VDKREG;  // 上值类型为寄存器
  env->name = ls->envn;  // 上值名称为环境名
  luaC_objbarrier(ls->L, fs->f, env->name);  // 对象屏障
  luaX_next(ls);  /* 读取第一个标记 */
  statlist(ls);  /* 解析主体 */
  check(ls, TK_EOS);  // 检查结束符
  close_func(ls);  // 关闭函数
}


LClosure *luaY_parser (lua_State *L, ZIO *z, Mbuffer *buff,
                       Dyndata *dyd, const char *name, int firstchar) {
  LexState lexstate;  // 定义词法状态
  FuncState funcstate;  // 定义函数状态
  LClosure *cl = luaF_newLclosure(L, 1);  /* 创建主闭包 */
  setclLvalue2s(L, L->top, cl);  /* 锚定它（避免被回收） */
  luaD_inctop(L);  // 栈顶指针增加
  lexstate.h = luaH_new(L);  /* 创建扫描器的表 */
  sethvalue2s(L, L->top, lexstate.h);  /* 锚定它 */
  luaD_inctop(L);  // 栈顶指针增加
  funcstate.f = cl->p = luaF_newproto(L);
  luaC_objbarrier(L, cl, cl->p);
  funcstate.f->source = luaS_new(L, name);  /* 创建并锚定字符串 */
  luaC_objbarrier(L, funcstate.f, funcstate.f->source);
  lexstate.buff = buff;
  lexstate.dyd = dyd;
  dyd->actvar.n = dyd->gt.n = dyd->label.n = 0;
  luaX_setinput(L, &lexstate, z, funcstate.f->source, firstchar);
  mainfunc(&lexstate, &funcstate);
  lua_assert(!funcstate.prev && funcstate.nups == 1 && !lexstate.fs);
  /* 所有作用域应正确结束 */
  lua_assert(dyd->actvar.n == 0 && dyd->gt.n == 0 && dyd->label.n == 0);
  L->top--;  /* 移除扫描器的表 */
  return cl;  /* 闭包也在栈上 */
}
```