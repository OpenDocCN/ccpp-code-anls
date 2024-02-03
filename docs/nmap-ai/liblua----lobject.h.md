# `nmap\liblua\lobject.h`

```cpp
/*
** $Id: lobject.h $
** Type definitions for Lua objects
** See Copyright Notice in lua.h
*/


#ifndef lobject_h
#define lobject_h


#include <stdarg.h>


#include "llimits.h"
#include "lua.h"


/*
** Extra types for collectable non-values
*/
#define LUA_TUPVAL    LUA_NUMTYPES  /* upvalues */
#define LUA_TPROTO    (LUA_NUMTYPES+1)  /* function prototypes */
#define LUA_TDEADKEY    (LUA_NUMTYPES+2)  /* removed keys in tables */



/*
** number of all possible types (including LUA_TNONE but excluding DEADKEY)
*/
#define LUA_TOTALTYPES        (LUA_TPROTO + 2)


/*
** tags for Tagged Values have the following use of bits:
** bits 0-3: actual tag (a LUA_T* constant)
** bits 4-5: variant bits
** bit 6: whether value is collectable
*/

/* add variant bits to a type */
#define makevariant(t,v)    ((t) | ((v) << 4))



/*
** Union of all Lua values
*/
typedef union Value {
  struct GCObject *gc;    /* collectable objects */
  void *p;         /* light userdata */
  lua_CFunction f; /* light C functions */
  lua_Integer i;   /* integer numbers */
  lua_Number n;    /* float numbers */
} Value;


/*
** Tagged Values. This is the basic representation of values in Lua:
** an actual value plus a tag with its type.
*/

#define TValuefields    Value value_; lu_byte tt_

typedef struct TValue {
  TValuefields;
} TValue;


#define val_(o)        ((o)->value_)
#define valraw(o)    (val_(o))


/* raw type tag of a TValue */
#define rawtt(o)    ((o)->tt_)

/* tag with no variants (bits 0-3) */
#define novariant(t)    ((t) & 0x0F)

/* type tag of a TValue (bits 0-3 for tags + variant bits 4-5) */
#define withvariant(t)    ((t) & 0x3F)
#define ttypetag(o)    withvariant(rawtt(o))

/* type of a TValue */
#define ttype(o)    (novariant(rawtt(o)))


/* Macros to test type */
#define checktag(o,t)        (rawtt(o) == (t))
#define checktype(o,t)        (ttype(o) == (t))


/* Macros for internal tests */

/* collectable object has the same tag as the original value */
/* 定义宏，用于检查对象的类型标签是否正确 */
#define righttt(obj)        (ttypetag(obj) == gcvalue(obj)->tt)

/*
** 程序中操作的任何值都是不可回收的，或者可回收对象具有正确的标签
** 并且不是死对象。选项 'L == NULL' 允许其他宏在没有 L 可用的情况下使用该宏。
*/
#define checkliveness(L,obj) \
    ((void)L, lua_longassert(!iscollectable(obj) || \
        (righttt(obj) && (L == NULL || !isdead(G(L),gcvalue(obj)))))

/* 宏用于设置值的标签 */
#define settt_(o,t)    ((o)->tt_=(t))

/* 主要宏，用于复制值（从 'obj2' 到 'obj1'） */
#define setobj(L,obj1,obj2) \
    { TValue *io1=(obj1); const TValue *io2=(obj2); \
          io1->value_ = io2->value_; settt_(io1, io2->tt_); \
      checkliveness(L,io1); lua_assert(!isnonstrictnil(io1)); }

/*
** 根据源和目标的不同类型，有不同类型的赋值。
** （它们现在大部分相等，但将来可能会有所不同。）
*/

/* 从堆栈到堆栈 */
#define setobjs2s(L,o1,o2)    setobj(L,s2v(o1),s2v(o2))
/* 到堆栈（不是从同一个堆栈） */
#define setobj2s(L,o1,o2)    setobj(L,s2v(o1),o2)
/* 从表到相同的表 */
#define setobjt2t    setobj
/* 到新对象 */
#define setobj2n    setobj
/* 到表 */
#define setobj2t    setobj

/*
** Lua 堆栈中的条目。字段 'tbclist' 形成此堆栈中所有活动的待关闭变量的列表。
** 当两个 tbc 变量之间的距离不适合无符号短时，使用虚拟条目。
** 它们由 delta==0 表示，它们的真实 delta 始终是适合该字段的最大值。
*/
typedef union StackValue {
  TValue val;
  struct {
    TValuefields;
    unsigned short delta;
  } tbclist;
} StackValue;

/* 堆栈元素的索引 */
typedef StackValue *StkId;

/* 将 'StackValue' 转换为 'TValue' */
#define s2v(o)    (&(o)->val)

/*
** {==================================================================
** Nil
/* Standard nil */
#define LUA_VNIL    makevariant(LUA_TNIL, 0)
// 定义标准的 nil，使用 makevariant 函数将类型和值组合成一个整数

/* Empty slot (which might be different from a slot containing nil) */
#define LUA_VEMPTY    makevariant(LUA_TNIL, 1)
// 定义空槽，可能与包含 nil 的槽不同

/* Value returned for a key not found in a table (absent key) */
#define LUA_VABSTKEY    makevariant(LUA_TNIL, 2)
// 定义在表中找不到的键的返回值（缺失的键）

/* macro to test for (any kind of) nil */
#define ttisnil(v)        checktype((v), LUA_TNIL)
// 用于测试（任何类型的）nil 的宏

/* macro to test for a standard nil */
#define ttisstrictnil(o)    checktag((o), LUA_VNIL)
// 用于测试标准 nil 的宏

#define setnilvalue(obj) settt_(obj, LUA_VNIL)
// 将对象设置为标准 nil 的宏

#define isabstkey(v)        checktag((v), LUA_VABSTKEY)
// 用于检测是否为缺失键的宏

/*
** macro to detect non-standard nils (used only in assertions)
*/
#define isnonstrictnil(v)    (ttisnil(v) && !ttisstrictnil(v))
// 用于检测非标准 nil 的宏（仅用于断言）

/*
** By default, entries with any kind of nil are considered empty.
** (In any definition, values associated with absent keys must also
** be accepted as empty.)
*/
#define isempty(v)        ttisnil(v)
// 默认情况下，任何类型的 nil 都被视为空的条目

/* macro defining a value corresponding to an absent key */
#define ABSTKEYCONSTANT        {NULL}, LUA_VABSTKEY
// 定义与缺失键对应的值的宏

/* mark an entry as empty */
#define setempty(v)        settt_(v, LUA_VEMPTY)
// 将条目标记为空的宏

/* }================================================================== */

/*
** {==================================================================
** Booleans
** ===================================================================
*/

#define LUA_VFALSE    makevariant(LUA_TBOOLEAN, 0)
#define LUA_VTRUE    makevariant(LUA_TBOOLEAN, 1)
// 定义 false 和 true 的值

#define ttisboolean(o)        checktype((o), LUA_TBOOLEAN)
#define ttisfalse(o)        checktag((o), LUA_VFALSE)
#define ttistrue(o)        checktag((o), LUA_VTRUE)
// 用于测试布尔类型的宏

#define l_isfalse(o)    (ttisfalse(o) || ttisnil(o))
// 用于判断是否为 false 的宏

#define setbfvalue(obj)        settt_(obj, LUA_VFALSE)
#define setbtvalue(obj)        settt_(obj, LUA_VTRUE)
// 将对象设置为 false 和 true 的宏

/* }================================================================== */
/*
** {==================================================================
** Threads
** ===================================================================
*/

/* 定义 LUA 线程类型的变体 */
#define LUA_VTHREAD        makevariant(LUA_TTHREAD, 0)

/* 检查对象是否为线程类型 */
#define ttisthread(o)        checktag((o), ctb(LUA_VTHREAD))

/* 获取线程对象的值 */
#define thvalue(o)    check_exp(ttisthread(o), gco2th(val_(o).gc))

/* 设置对象为线程类型的值 */
#define setthvalue(L,obj,x) \
  { TValue *io = (obj); lua_State *x_ = (x); \
    val_(io).gc = obj2gco(x_); settt_(io, ctb(LUA_VTHREAD)); \
    checkliveness(L,io); }

/* 设置对象为线程类型的值，使用 s2v 宏 */
#define setthvalue2s(L,o,t)    setthvalue(L,s2v(o),t)

/* }================================================================== */


/*
** {==================================================================
** Collectable Objects
** ===================================================================
*/

/*
** 所有可收集对象的通用头部（以宏形式存在，用于包含在其他对象中）
*/
#define CommonHeader    struct GCObject *next; lu_byte tt; lu_byte marked


/* 所有可收集对象的通用类型 */
typedef struct GCObject {
  CommonHeader;
} GCObject;


/* 可收集类型的位标记 */
#define BIT_ISCOLLECTABLE    (1 << 6)

/* 检查对象是否为可收集类型 */
#define iscollectable(o)    (rawtt(o) & BIT_ISCOLLECTABLE)

/* 将标记设置为可收集类型 */
#define ctb(t)            ((t) | BIT_ISCOLLECTABLE)

/* 获取对象的 GC 值 */
#define gcvalue(o)    check_exp(iscollectable(o), val_(o).gc)

/* 获取对象的原始 GC 值 */
#define gcvalueraw(v)    ((v).gc)

/* 设置对象的 GC 值 */
#define setgcovalue(L,obj,x) \
  { TValue *io = (obj); GCObject *i_g=(x); \
    val_(io).gc = i_g; settt_(io, ctb(i_g->tt)); }

/* }================================================================== */


/*
** {==================================================================
** Numbers
** ===================================================================
*/

/* 数字的变体标记 */
#define LUA_VNUMINT    makevariant(LUA_TNUMBER, 0)  /* 整数数字 */
#define LUA_VNUMFLT    makevariant(LUA_TNUMBER, 1)  /* 浮点数数字 */
#define ttisnumber(o)        checktype((o), LUA_TNUMBER)  // 检查对象 o 的类型是否为数字
#define ttisfloat(o)        checktag((o), LUA_VNUMFLT)  // 检查对象 o 的标签是否为浮点数
#define ttisinteger(o)        checktag((o), LUA_VNUMINT)  // 检查对象 o 的标签是否为整数

#define nvalue(o)    check_exp(ttisnumber(o), \  // 如果对象 o 是数字，返回其值；如果是整数，返回其值；如果是浮点数，返回其值
    (ttisinteger(o) ? cast_num(ivalue(o)) : fltvalue(o)))
#define fltvalue(o)    check_exp(ttisfloat(o), val_(o).n)  // 如果对象 o 是浮点数，返回其值
#define ivalue(o)    check_exp(ttisinteger(o), val_(o).i)  // 如果对象 o 是整数，返回其值

#define fltvalueraw(v)    ((v).n)  // 返回浮点数值
#define ivalueraw(v)    ((v).i)  // 返回整数值

#define setfltvalue(obj,x) \  // 设置对象 obj 的值为浮点数 x
  { TValue *io=(obj); val_(io).n=(x); settt_(io, LUA_VNUMFLT); }

#define chgfltvalue(obj,x) \  // 修改对象 obj 的值为浮点数 x
  { TValue *io=(obj); lua_assert(ttisfloat(io)); val_(io).n=(x); }

#define setivalue(obj,x) \  // 设置对象 obj 的值为整数 x
  { TValue *io=(obj); val_(io).i=(x); settt_(io, LUA_VNUMINT); }

#define chgivalue(obj,x) \  // 修改对象 obj 的值为整数 x
  { TValue *io=(obj); lua_assert(ttisinteger(io)); val_(io).i=(x); }

/* }================================================================== */


/*
** {==================================================================
** Strings
** ===================================================================
*/

/* Variant tags for strings */
#define LUA_VSHRSTR    makevariant(LUA_TSTRING, 0)  // 创建短字符串的变体标签
#define LUA_VLNGSTR    makevariant(LUA_TSTRING, 1)  // 创建长字符串的变体标签

#define ttisstring(o)        checktype((o), LUA_TSTRING)  // 检查对象 o 的类型是否为字符串
#define ttisshrstring(o)    checktag((o), ctb(LUA_VSHRSTR))  // 检查对象 o 的标签是否为短字符串
#define ttislngstring(o)    checktag((o), ctb(LUA_VLNGSTR))  // 检查对象 o 的标签是否为长字符串

#define tsvalueraw(v)    (gco2ts((v).gc))  // 返回字符串值的原始值

#define tsvalue(o)    check_exp(ttisstring(o), gco2ts(val_(o).gc))  // 如果对象 o 是字符串，返回其字符串值

#define setsvalue(L,obj,x) \  // 设置对象 obj 的值为字符串 x
  { TValue *io = (obj); TString *x_ = (x); \
    val_(io).gc = obj2gco(x_); settt_(io, ctb(x_->tt)); \
    checkliveness(L,io); }

/* set a string to the stack */
#define setsvalue2s(L,o,s)    setsvalue(L,s2v(o),s)  // 将字符串设置到堆栈

/* set a string to a new object */
#define setsvalue2n    setsvalue  // 设置一个新对象的字符串值


/*
** Header for a string value.
*/
/* 定义了一个名为 TString 的结构体，用于表示字符串对象 */
typedef struct TString {
  CommonHeader;  /* 通用头部信息 */
  lu_byte extra;  /* 用于短字符串的保留字段；对于长字符串则表示是否有哈希值 */
  lu_byte shrlen;  /* 短字符串的长度 */
  unsigned int hash;  /* 哈希值 */
  union {
    size_t lnglen;  /* 长字符串的长度 */
    struct TString *hnext;  /* 哈希表中的链表指针 */
  } u;
  char contents[1];  /* 字符串内容 */
} TString;

/* 从 TString 中获取实际的字符串（字节数组） */
#define getstr(ts)  ((ts)->contents)

/* 从 Lua 值中获取实际的字符串（字节数组） */
#define svalue(o)       getstr(tsvalue(o))

/* 从 'TString *s' 中获取字符串长度 */
#define tsslen(s)    ((s)->tt == LUA_VSHRSTR ? (s)->shrlen : (s)->u.lnglen)

/* 从 'TValue *o' 中获取字符串长度 */
#define vslen(o)    tsslen(tsvalue(o))

/* }================================================================== */

/* 用户数据 */

/* 轻量级用户数据应该是用户数据的一种变体，但出于兼容性的考虑，它们也是不同的类型 */
#define LUA_VLIGHTUSERDATA    makevariant(LUA_TLIGHTUSERDATA, 0)

#define LUA_VUSERDATA        makevariant(LUA_TUSERDATA, 0)

#define ttislightuserdata(o)    checktag((o), LUA_VLIGHTUSERDATA)
#define ttisfulluserdata(o)    checktag((o), ctb(LUA_VUSERDATA))

#define pvalue(o)    check_exp(ttislightuserdata(o), val_(o).p)
#define uvalue(o)    check_exp(ttisfulluserdata(o), gco2u(val_(o).gc))

#define pvalueraw(v)    ((v).p)

#define setpvalue(obj,x) \
  { TValue *io=(obj); val_(io).p=(x); settt_(io, LUA_VLIGHTUSERDATA); }  /* 设置轻量级用户数据的值 */

#define setuvalue(L,obj,x) \
  { TValue *io = (obj); Udata *x_ = (x); \
    val_(io).gc = obj2gco(x_); settt_(io, ctb(LUA_VUSERDATA)); \
    checkliveness(L,io); }  /* 设置用户数据的值 */

/* 确保此类型后的地址始终完全对齐 */
typedef union UValue {
  TValue uv;
  LUAI_MAXALIGN;  /* 确保用户数据字节的最大对齐 */
/* 结构体，用于表示带有用户值的用户数据；内存区域跟随此结构的末尾 */
typedef struct Udata {
  CommonHeader;  // 公共头部
  unsigned short nuvalue;  // 用户值的数量
  size_t len;  // 字节数
  struct Table *metatable;  // 元表
  GCObject *gclist;  // GC 列表
  UValue uv[1];  // 用户值数组
} Udata;


/* 结构体，用于表示没有用户值的用户数据。这些用户数据在 GC 过程中不需要变灰，因此不需要 'gclist' 字段。
为了简化，代码总是使用 'Udata' 来表示两种类型的用户数据，确保在没有用户值的用户数据上永远不会访问 'gclist'。
此结构仅用于计算此表示的正确大小。（其末尾的 'bindata' 字段确保了接下来的二进制数据的正确对齐。） */
typedef struct Udata0 {
  CommonHeader;  // 公共头部
  unsigned short nuvalue;  // 用户值的数量
  size_t len;  // 字节数
  struct Table *metatable;  // 元表
  union {LUAI_MAXALIGN;} bindata;  // 二进制数据
} Udata0;


/* 计算用户数据内存区域的偏移量 */
#define udatamemoffset(nuv) \
    ((nuv) == 0 ? offsetof(Udata0, bindata)  \
                    : offsetof(Udata, uv) + (sizeof(UValue) * (nuv)))

/* 获取 'Udata' 内存块的地址 */
#define getudatamem(u)    (cast_charp(u) + udatamemoffset((u)->nuvalue))

/* 计算用户数据的大小 */
#define sizeudata(nuv,nb)    (udatamemoffset(nuv) + (nb))

/* }================================================================== */


/*
** {==================================================================
** Prototypes
** ===================================================================
*/

#define LUA_VPROTO    makevariant(LUA_TPROTO, 0)


/*
** 函数原型的上值描述
*/
# 定义了 Upvaldesc 结构体，用于描述闭包中的 upvalue 变量
typedef struct Upvaldesc {
  TString *name;  /* upvalue 的名称（用于调试信息） */
  lu_byte instack;  /* 是否在堆栈中（寄存器） */
  lu_byte idx;  /* upvalue 的索引（在堆栈中或外部函数的列表中） */
  lu_byte kind;  /* 对应变量的类型 */
} Upvaldesc;


/*
** 函数原型中本地变量的描述（用于调试信息）
*/
typedef struct LocVar {
  TString *varname;
  int startpc;  /* 变量活跃的第一个点 */
  int endpc;    /* 变量失效的第一个点 */
} LocVar;


/*
** 为给定指令（'pc'）关联绝对行号的结构体。
** 数组 'lineinfo' 给出了每个指令与前一个指令的行号差异。
** 当这个差异不能用一个字节表示时，Lua 会保存该指令的绝对行号。
** （Lua 也会定期保存绝对行号，以加快行号的计算：我们可以在绝对行号数组中使用二分查找，
** 但必须线性遍历 'lineinfo' 数组来计算行号。）
*/
typedef struct AbsLineInfo {
  int pc;
  int line;
} AbsLineInfo;

/*
** 函数原型
*/
typedef struct Proto {
  CommonHeader;  // 公共头部信息
  lu_byte numparams;  // 固定（命名）参数的数量
  lu_byte is_vararg;  // 是否是可变参数
  lu_byte maxstacksize;  // 函数需要的寄存器数量
  int sizeupvalues;  // 'upvalues' 的大小
  int sizek;  // 'k' 的大小
  int sizecode;  // 'code' 的大小
  int sizelineinfo;  // 'lineinfo' 的大小
  int sizep;  // 'p' 的大小
  int sizelocvars;  // 'locvars' 的大小
  int sizeabslineinfo;  // 'abslineinfo' 的大小
  int linedefined;  // 调试信息
  int lastlinedefined;  // 调试信息
  TValue *k;  // 函数使用的常量
  Instruction *code;  // 操作码
  struct Proto **p;  // 函数内部定义的函数
  Upvaldesc *upvalues;  // upvalue 信息
  ls_byte *lineinfo;  // 源代码行信息（调试信息）
  AbsLineInfo *abslineinfo;  // 同上
  LocVar *locvars;  // 局部变量信息（调试信息）
  TString  *source;  // 用于调试信息
  GCObject *gclist;  // 垃圾回收链表
} Proto;

/* }================================================================== */


/*
** {==================================================================
** Functions
** ===================================================================
*/

#define LUA_VUPVAL    makevariant(LUA_TUPVAL, 0)  // 创建 LUA_TUPVAL 类型的变体标签


/* Variant tags for functions */
#define LUA_VLCL    makevariant(LUA_TFUNCTION, 0)  // Lua 闭包
#define LUA_VLCF    makevariant(LUA_TFUNCTION, 1)  // 轻量级 C 函数
#define LUA_VCCL    makevariant(LUA_TFUNCTION, 2)  // C 闭包

#define ttisfunction(o)        checktype(o, LUA_TFUNCTION)  // 检查对象是否是 LUA_TFUNCTION 类型
#define ttisLclosure(o)        checktag((o), ctb(LUA_VLCL))  // 检查对象是否是 LUA_VLCL 类型
#define ttislcf(o)        checktag((o), LUA_VLCF)  // 检查对象是否是 LUA_VLCF 类型
#define ttisCclosure(o)        checktag((o), ctb(LUA_VCCL))  // 检查对象是否是 LUA_VCCL 类型
#define ttisclosure(o)         (ttisLclosure(o) || ttisCclosure(o))  // 检查对象是否是闭包类型


#define isLfunction(o)    ttisLclosure(o)  // 检查对象是否是 Lua 闭包类型

#define clvalue(o)    check_exp(ttisclosure(o), gco2cl(val_(o).gc))  // 检查对象是否是闭包类型，返回闭包对象
#define clLvalue(o)    check_exp(ttisLclosure(o), gco2lcl(val_(o).gc))
// 宏定义，用于检查对象是否为闭包类型，并返回其值

#define fvalue(o)    check_exp(ttislcf(o), val_(o).f)
// 宏定义，用于检查对象是否为 C 函数类型，并返回其值

#define clCvalue(o)    check_exp(ttisCclosure(o), gco2ccl(val_(o).gc))
// 宏定义，用于检查对象是否为 C 闭包类型，并返回其值

#define fvalueraw(v)    ((v).f)
// 宏定义，用于返回 C 函数类型的值

#define setclLvalue(L,obj,x) \
  { TValue *io = (obj); LClosure *x_ = (x); \
    val_(io).gc = obj2gco(x_); settt_(io, ctb(LUA_VLCL)); \
    checkliveness(L,io); }
// 宏定义，用于设置闭包类型的值

#define setclLvalue2s(L,o,cl)    setclLvalue(L,s2v(o),cl)
// 宏定义，用于设置闭包类型的值

#define setfvalue(obj,x) \
  { TValue *io=(obj); val_(io).f=(x); settt_(io, LUA_VLCF); }
// 宏定义，用于设置 C 函数类型的值

#define setclCvalue(L,obj,x) \
  { TValue *io = (obj); CClosure *x_ = (x); \
    val_(io).gc = obj2gco(x_); settt_(io, ctb(LUA_VCCL)); \
    checkliveness(L,io); }
// 宏定义，用于设置 C 闭包类型的值

/*
** Upvalues for Lua closures
*/
typedef struct UpVal {
  CommonHeader;
  lu_byte tbc;  /* true if it represents a to-be-closed variable */
  TValue *v;  /* points to stack or to its own value */
  union {
    struct {  /* (when open) */
      struct UpVal *next;  /* linked list */
      struct UpVal **previous;
    } open;
    TValue value;  /* the value (when closed) */
  } u;
} UpVal;
// 定义 Lua 闭包的 Upvalues 结构体

#define ClosureHeader \
    CommonHeader; lu_byte nupvalues; GCObject *gclist
// 宏定义，用于定义闭包头部

typedef struct CClosure {
  ClosureHeader;
  lua_CFunction f;
  TValue upvalue[1];  /* list of upvalues */
} CClosure;
// 定义 C 闭包结构体

typedef struct LClosure {
  ClosureHeader;
  struct Proto *p;
  UpVal *upvals[1];  /* list of upvalues */
} LClosure;
// 定义 Lua 闭包结构体

typedef union Closure {
  CClosure c;
  LClosure l;
} Closure;
// 定义闭包联合体

#define getproto(o)    (clLvalue(o)->p)
// 宏定义，用于获取闭包的原型

/* }================================================================== */


/*
** {==================================================================
** Tables
** ===================================================================
*/

#define LUA_VTABLE    makevariant(LUA_TTABLE, 0)
// 宏定义，用于创建表类型的变体

#define ttistable(o)        checktag((o), ctb(LUA_VTABLE))
// 宏定义，用于检查对象是否为表类型

#define hvalue(o)    check_exp(ttistable(o), gco2t(val_(o).gc))
// 宏定义，用于获取表类型的值
# 定义宏，用于设置哈希表中值的宏
#define sethvalue(L,obj,x) \
  { TValue *io = (obj); Table *x_ = (x); \  # 设置值的指针和表的指针
    val_(io).gc = obj2gco(x_); settt_(io, ctb(LUA_VTABLE)); \  # 设置值的垃圾回收对象和类型标记
    checkliveness(L,io); }  # 检查值的存活状态

# 定义宏，用于将值设置为哈希表中的键
#define sethvalue2s(L,o,h)    sethvalue(L,s2v(o),h)  # 设置值为哈希表中的键

'''
** 哈希表的节点：两个TValue（键-值对）的组合，加上一个'next'字段用于链接冲突的条目。
** 键的字段分布（'key_tt'和'key_val'）不构成一个合适的'TValue'，允许'Node'在4字节和8字节对齐中都有更小的大小。
'''
typedef union Node {
  struct NodeKey {
    TValuefields;  /* value的字段 */
    lu_byte key_tt;  /* 键的类型 */
    int next;  /* 用于链接 */
    Value key_val;  /* 键的值 */
  } u;
  TValue i_val;  /* 直接访问节点值作为合适的'TValue' */
} Node;  # 定义哈希表的节点

# 将值复制到键中
#define setnodekey(L,node,obj) \
    { Node *n_=(node); const TValue *io_=(obj); \  # 设置节点和值的指针
      n_->u.key_val = io_->value_; n_->u.key_tt = io_->tt_; \  # 设置键的值和类型
      checkliveness(L,io_); }  # 检查值的存活状态

# 从键中复制值
#define getnodekey(L,obj,node) \
    { TValue *io_=(obj); const Node *n_=(node); \  # 设置值和节点的指针
      io_->value_ = n_->u.key_val; io_->tt_ = n_->u.key_tt; \  # 设置值为键的值，类型为键的类型
      checkliveness(L,io_); }  # 检查值的存活状态

'''
** 关于'alimit'：如果'isrealasize(t)'为真，则'alimit'是'array'的实际大小。
** 否则，'array'的实际大小是不小于'alimit'的最小2的幂（如果'alimit'为零，则为零）；
** 然后'alimit'用作#t的提示。
'''
#define BITRAS        (1 << 7)  # 位掩码，用于判断是否为实际大小
#define isrealasize(t)        (!((t)->flags & BITRAS))  # 判断是否为实际大小
#define setrealasize(t)        ((t)->flags &= cast_byte(~BITRAS))  # 设置为实际大小
#define setnorealasize(t)    ((t)->flags |= BITRAS)  # 设置为非实际大小
# 定义一个名为 Table 的结构体，包含一系列字段
typedef struct Table {
  CommonHeader;
  lu_byte flags;  /* 1<<p means tagmethod(p) is not present */  # 标志位，用于表示 tagmethod(p) 是否存在
  lu_byte lsizenode;  /* log2 of size of 'node' array */  # 'node' 数组大小的对数
  unsigned int alimit;  /* "limit" of 'array' array */  # 'array' 数组的限制
  TValue *array;  /* array part */  # 数组部分
  Node *node;  # 节点
  Node *lastfree;  /* any free position is before this position */  # 最后一个空闲位置之前的任何空闲位置
  struct Table *metatable;  # 元表
  GCObject *gclist;  # GC 对象列表
} Table;


/*
** 用于操作插入节点的键的宏
*/
#define keytt(node)        ((node)->u.key_tt)  # 获取节点的键类型
#define keyval(node)        ((node)->u.key_val)  # 获取节点的键值

#define keyisnil(node)        (keytt(node) == LUA_TNIL)  # 判断节点的键是否为 nil
#define keyisinteger(node)    (keytt(node) == LUA_VNUMINT)  # 判断节点的键是否为整数
#define keyival(node)        (keyval(node).i)  # 获取节点的整数键值
#define keyisshrstr(node)    (keytt(node) == ctb(LUA_VSHRSTR))  # 判断节点的键是否为短字符串
#define keystrval(node)        (gco2ts(keyval(node).gc))  # 获取节点的字符串键值

#define setnilkey(node)        (keytt(node) = LUA_TNIL)  # 设置节点的键为 nil

#define keyiscollectable(n)    (keytt(n) & BIT_ISCOLLECTABLE)  # 判断节点的键是否可收集

#define gckey(n)    (keyval(n).gc)  # 获取节点的 GC 键值
#define gckeyN(n)    (keyiscollectable(n) ? gckey(n) : NULL)  # 如果节点的键可收集，则返回 GC 键值，否则返回 NULL


/*
** 表中的死键具有 DEADKEY 标记，但保留其原始 gcvalue。这将它们与常规键区分开，但允许在特殊方式搜索时找到它们。('next' 需要这样做才能在遍历期间找到从表中移除的键。)
*/
#define setdeadkey(node)    (keytt(node) = LUA_TDEADKEY)  # 设置节点的键为死键
#define keyisdead(node)        (keytt(node) == LUA_TDEADKEY)  # 判断节点的键是否为死键

/* }================================================================== */



/*
** 用于哈希运算的 'module' 操作（大小始终是 2 的幂）
*/
#define lmod(s,size) \
    (check_exp((size&(size-1))==0, (cast_int((s) & ((size)-1)))))


#define twoto(x)    (1<<(x))  # 计算 2 的 x 次方
#define sizenode(t)    (twoto((t)->lsizenode))  # 计算节点数组的大小


/* 'luaO_utf8esc' 函数的缓冲区大小 */
#define UTF8BUFFSZ    8

LUAI_FUNC int luaO_utf8esc (char *buff, unsigned long x);  # 声明 utf8esc 函数
LUAI_FUNC int luaO_ceillog2 (unsigned int x);  # 声明 ceillog2 函数
// 执行原始的算术操作，将结果存储在 res 中
LUAI_FUNC int luaO_rawarith (lua_State *L, int op, const TValue *p1,
                             const TValue *p2, TValue *res);

// 执行算术操作，将结果存储在 res 中
LUAI_FUNC void luaO_arith (lua_State *L, int op, const TValue *p1,
                           const TValue *p2, StkId res);

// 将字符串转换为数字，存储在 o 中，并返回字符串的长度
LUAI_FUNC size_t luaO_str2num (const char *s, TValue *o);

// 将十六进制字符转换为对应的数值
LUAI_FUNC int luaO_hexavalue (int c);

// 将对象 obj 转换为字符串
LUAI_FUNC void luaO_tostring (lua_State *L, TValue *obj);

// 将格式化字符串推入栈中，使用可变参数列表
LUAI_FUNC const char *luaO_pushvfstring (lua_State *L, const char *fmt,
                                                       va_list argp);

// 将格式化字符串推入栈中，使用可变参数列表
LUAI_FUNC const char *luaO_pushfstring (lua_State *L, const char *fmt, ...);

// 将源字符串的前 srclen 个字符复制到 out 中，并添加省略号
LUAI_FUNC void luaO_chunkid (char *out, const char *source, size_t srclen);
```