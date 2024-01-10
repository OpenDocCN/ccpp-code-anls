# `nmap\liblua\lparser.h`

```
/*
** $Id: lparser.h $
** Lua Parser
** See Copyright Notice in lua.h
*/

#ifndef lparser_h
#define lparser_h

#include "llimits.h"
#include "lobject.h"
#include "lzio.h"


/*
** Expression and variable descriptor.
** Code generation for variables and expressions can be delayed to allow
** optimizations; An 'expdesc' structure describes a potentially-delayed
** variable/expression. It has a description of its "main" value plus a
** list of conditional jumps that can also produce its value (generated
** by short-circuit operators 'and'/'or').
*/

/* kinds of variables/expressions */
# 定义枚举类型，表示表达式的种类
typedef enum {
  VVOID,  /* 当 'expdesc' 描述列表的最后一个表达式时，这种类型表示空列表（即没有表达式） */
  VNIL,  /* 常量 nil */
  VTRUE,  /* 常量 true */
  VFALSE,  /* 常量 false */
  VK,  /* 'k' 中的常量；info = 'k' 中常量的索引 */
  VKFLT,  /* 浮点常量；nval = 数值浮点值 */
  VKINT,  /* 整数常量；ival = 数值整数值 */
  VKSTR,  /* 字符串常量；strval = TString 地址；（字符串由词法分析器固定） */
  VNONRELOC,  /* 表达式的值在固定寄存器中；info = 结果寄存器 */
  VLOCAL,  /* 局部变量；var.ridx = 寄存器索引；var.vidx = 'actvar.arr' 中的相对索引 */
  VUPVAL,  /* 上值变量；info = 'upvalues' 中的上值索引 */
  VCONST,  /* 编译时 <const> 变量；info = 'actvar.arr' 中的绝对索引 */
  VINDEXED,  /* 索引变量；ind.t = 表寄存器；ind.idx = 键的 R 索引 */
  VINDEXUP,  /* 索引上值；ind.t = 表上值；ind.idx = 键的 K 索引 */
  VINDEXI, /* 具有常量整数的索引变量；
                ind.t = 表寄存器；
                ind.idx = 键的值 */
  VINDEXSTR, /* 具有字面字符串的索引变量；
                ind.t = 表寄存器；
                ind.idx = 键的 K 索引 */
  VJMP,  /* 表达式是测试/比较；info = 相应跳转指令的程序计数 */
  VRELOC,  /* 表达式可以将结果放入任何寄存器；info = 指令程序计数 */
  VCALL,  /* 表达式是函数调用；info = 指令程序计数 */
  VVARARG  /* 可变参数表达式；info = 指令程序计数 */
} expkind;

# 定义宏，用于判断表达式种类是否为变量
#define vkisvar(k)    (VLOCAL <= (k) && (k) <= VINDEXSTR)
# 定义宏，用于判断表达式种类是否为索引变量
#define vkisindexed(k)    (VINDEXED <= (k) && (k) <= VINDEXSTR)

# 定义结构体，描述表达式
typedef struct expdesc {
  expkind k;  # 表达式种类
  union {
    lua_Integer ival;    /* 用于 VKINT */
    lua_Number nval;  /* 用于 VKFLT 类型的数值变量 */
    TString *strval;  /* 用于 VKSTR 类型的字符串变量 */
    int info;  /* 用于通用目的的整型变量 */
    struct {  /* 用于索引变量 */
      short idx;  /* 索引值 (R 或者 "long" K) */
      lu_byte t;  /* 表格 (寄存器或者上值) */
    } ind;
    struct {  /* 用于局部变量 */
      lu_byte ridx;  /* 存储变量的寄存器 */
      unsigned short vidx;  /* 编译器索引 (在 'actvar.arr' 中的索引)  */
    } var;
  } u;
  int t;  /* '条件为真' 时的跳转列表 */
  int f;  /* '条件为假' 时的跳转列表 */
/* 结构体，描述表达式的类型和值 */
} expdesc;

/* 变量的类型 */
#define VDKREG        0   /* 普通变量 */
#define RDKCONST    1   /* 常量 */
#define RDKTOCLOSE    2   /* 待关闭的变量 */
#define RDKCTC        3   /* 编译时常量 */

/* 活动局部变量的描述 */
typedef union Vardesc {
  struct {
    TValuefields;  /* 常量值（如果是编译时常量） */
    lu_byte kind;
    lu_byte ridx;  /* 存储变量的寄存器 */
    short pidx;  /* 变量在 Proto 的 'locvars' 数组中的索引 */
    TString *name;  /* 变量名 */
  } vd;
  TValue k;  /* 常量值（如果有的话） */
} Vardesc;

/* 待处理的跳转语句和标签语句的描述 */
typedef struct Labeldesc {
  TString *name;  /* 标签标识符 */
  int pc;  /* 代码中的位置 */
  int line;  /* 出现的行数 */
  lu_byte nactvar;  /* 该位置的活动变量数 */
  lu_byte close;  /* 跳转语句是否涉及闭包 */
} Labeldesc;

/* 标签或跳转语句的列表 */
typedef struct Labellist {
  Labeldesc *arr;  /* 数组 */
  int n;  /* 使用中的条目数 */
  int size;  /* 数组大小 */
} Labellist;

/* 解析器使用的动态结构 */
typedef struct Dyndata {
  struct {  /* 所有活动局部变量的列表 */
    Vardesc *arr;
    int n;
    int size;
  } actvar;
  Labellist gt;  /* 待处理的跳转语句列表 */
  Labellist label;   /* 活动标签列表 */
} Dyndata;

/* 控制块的状态 */
struct BlockCnt;  /* 在 lparser.c 中定义 */

/* 为给定函数生成代码所需的状态 */
// 定义了表示函数状态的结构体
typedef struct FuncState {
  Proto *f;  /* 当前函数头部 */
  struct FuncState *prev;  /* 包围的函数 */
  struct LexState *ls;  /* 词法状态 */
  struct BlockCnt *bl;  /* 当前块的链 */
  int pc;  /* 下一个代码位置（相当于 'ncode'） */
  int lasttarget;   /* 上一个 '跳转标签' 的 '标签' */
  int previousline;  /* 上次保存在 'lineinfo' 中的行 */
  int nk;  /* 'k' 中元素的数量 */
  int np;  /* 'p' 中元素的数量 */
  int nabslineinfo;  /* 'abslineinfo' 中元素的数量 */
  int firstlocal;  /* 第一个本地变量的索引（在 Dyndata 数组中） */
  int firstlabel;  /* 第一个标签的索引（在 'dyd->label->arr' 中） */
  short ndebugvars;  /* 'f->locvars' 中元素的数量 */
  lu_byte nactvar;  /* 活动本地变量的数量 */
  lu_byte nups;  /* upvalue 的数量 */
  lu_byte freereg;  /* 第一个空闲寄存器 */
  lu_byte iwthabs;  /* 自上次绝对行信息以来发出的指令 */
  lu_byte needclose;  /* 函数在返回时需要关闭 upvalue */
} FuncState;

// 声明了一个函数，返回变量堆栈的数量
LUAI_FUNC int luaY_nvarstack (FuncState *fs);
// 声明了一个函数，解析 Lua 代码
LUAI_FUNC LClosure *luaY_parser (lua_State *L, ZIO *z, Mbuffer *buff,
                                 Dyndata *dyd, const char *name, int firstchar);
```