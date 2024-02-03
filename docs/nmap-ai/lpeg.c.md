# `nmap\lpeg.c`

```cpp
/*
** $Id: lptypes.h,v 1.8 2013/04/12 16:26:38 roberto Exp $
** LPeg - PEG pattern matching for Lua
** Copyright 2007, Lua.org & PUC-Rio  (see 'lpeg.html' for license)
** written by Roberto Ierusalimschy
*/

// 如果未定义 lptypes_h，则定义 lptypes_h
#if !defined(lptypes_h)
#define lptypes_h

// 如果未定义 LPEG_DEBUG，则定义 NDEBUG
#if !defined(LPEG_DEBUG)
#define NDEBUG
#endif

// 包含头文件
#include <assert.h>
#include <limits.h>

// 定义版本号
#define VERSION         "0.12"

// 定义模式类型
#define PATTERN_T    "lpeg-pattern"
// 定义最大堆栈索引
#define MAXSTACKIDX    "lpeg-maxstack"

/*
** compatibility with Lua 5.2
*/
#if (LUA_VERSION_NUM == 502)

// 重新定义 lua_equal
#undef lua_equal
#define lua_equal(L,idx1,idx2)  lua_compare(L,(idx1),(idx2),LUA_OPEQ)

// 重新定义 lua_getfenv 和 lua_setfenv
#undef lua_getfenv
#define lua_getfenv    lua_getuservalue
#undef lua_setfenv
#define lua_setfenv    lua_setuservalue

// 重新定义 lua_objlen
#undef lua_objlen
#define lua_objlen    lua_rawlen

// 重新定义 luaL_register
#undef luaL_register
#define luaL_register(L,n,f) \
    { if ((n) == NULL) luaL_setfuncs(L,f,0); else luaL_newlib(L,f); }

#endif

// 默认调用/回溯堆栈的最大大小
#if !defined(MAXBACK)
#define MAXBACK         100
#endif

// 语法中规则的最大数量
#define MAXRULES        200

// 捕获列表的初始大小
#define INITCAPSIZE    32

// 主题在 Lua 堆栈上的索引
#define SUBJIDX        2

// 'match' 的固定参数数量（捕获参数之前）
#define FIXEDARGS    3

// 捕获列表在 Lua 堆栈上的索引
#define caplistidx(ptop)    ((ptop) + 2)

// 模式的 ktable 在 Lua 堆栈上的索引
#define ktableidx(ptop)        ((ptop) + 3)

// 回溯堆栈在 Lua 堆栈上的索引
#define stackidx(ptop)    ((ptop) + 4)

// 无符号字符类型
typedef unsigned char byte;

// 每个字符的位数
#define BITSPERCHAR        8

// 字符集的大小
#define CHARSETSIZE        ((UCHAR_MAX/BITSPERCHAR) + 1)

// 字符集结构
typedef struct Charset {
  byte cs[CHARSETSIZE];
} Charset;

// 循环设置字符集
#define loopset(v,b)    { int v; for (v = 0; v < CHARSETSIZE; v++) {b;} }

// 访问字符集
#define treebuffer(t)      ((byte *)((t) + 1))

// 需要 'n' 字节的插槽数量
/* 定义宏，将字节数转换为 TTree 结构体的数量 */
#define bytes2slots(n)  (((n) - 1) / sizeof(TTree) + 1)

/* 在字符集 'cs' 中设置 'b' 位 */
#define setchar(cs,b)   ((cs)[(b) >> 3] |= (1 << ((b) & 7)))

/*
** 在捕获指令中，'kind' 和其偏移量被打包在 'aux' 字段中，每个占 4 位
*/
#define getkind(op)        ((op)->i.aux & 0xF)
#define getoff(op)        (((op)->i.aux >> 4) & 0xF)
#define joinkindoff(k,o)    ((k) | ((o) << 4))

#define MAXOFF        0xF  // 最大偏移量
#define MAXAUX        0xFF  // 最大辅助值

/* 最大向后查看的字节数 */
#define MAXBEHIND    MAXAUX

/* 模式的最大大小（以元素为单位） */
#define MAXPATTSIZE    (SHRT_MAX - 10)

/* 指令加额外 l 字节的大小（以元素为单位） */
#define instsize(l)  (((l) + sizeof(Instruction) - 1)/sizeof(Instruction) + 1)

/* ISet 指令的大小（以元素为单位） */
#define CHARSETINSTSIZE        instsize(CHARSETSIZE)

/* IFunc 指令的大小（以元素为单位） */
#define funcinstsize(p)        ((p)->i.aux + 2)

#define testchar(st,c)    (((int)(st)[((c) >> 3)] & (1 << ((c) & 7)))

#endif

/*
** $Id: lptree.h,v 1.2 2013/03/24 13:51:12 roberto Exp $
*/

#if !defined(lptree_h)
#define lptree_h

/*
** 树的类型
*/
typedef enum TTag {
  TChar = 0, TSet, TAny,  /* 标准 PEG 元素 */
  TTrue, TFalse,
  TRep,
  TSeq, TChoice,
  TNot, TAnd,
  TCall,
  TOpenCall,
  TRule,  /* sib1 是规则的模式，sib2 是 'next' 规则 */
  TGrammar,  /* sib1 是初始（也是第一个）规则 */
  TBehind,  /* 向后匹配 */
  TCapture,  /* 常规捕获 */
  TRunTime  /* 运行时捕获 */
} TTag;

/* 每个树的兄弟节点数 */
extern const byte numsiblings[];

/*
** 树的树
** 如果树有第一个兄弟节点（如果有的话），则紧跟在树的后面。
** 对第二个兄弟节点（ps）的引用是相对于树本身的位置。
** ktable 中的键使用创建该条目的原始树的（唯一）地址。NULL 表示没有数据。
*/
/* 定义了一个名为 TTree 的结构体，用于表示正则表达式的语法树节点 */
typedef struct TTree {
  byte tag;  /* 节点类型 */
  byte cap;  /* 捕获类型（如果是捕获的话） */
  unsigned short key;  /* Lua 数据在 ktable 中的键（如果没有键则为 0） */
  union {
    int ps;  /* 偶尔出现的第二个兄弟节点 */
    int n;  /* 偶尔出现的计数器 */
  } u;
} TTree;


/*
** 完整的模式包括其语法树以及（如果已经编译）其对应的代码
*/
typedef struct Pattern {
  union Instruction *code;  /* 对应的指令数组 */
  int codesize;  /* 指令数组的大小 */
  TTree tree[1];  /* 语法树数组（长度为 1，实际长度根据需要动态分配） */
} Pattern;


/* 每个语法树节点的兄弟节点数量 */
extern const byte numsiblings[];

/* 访问兄弟节点 */
#define sib1(t)         ((t) + 1)  /* 第一个兄弟节点 */
#define sib2(t)         ((t) + (t)->u.ps)  /* 第二个兄弟节点 */




#endif

/*
** $Id: lpcap.h,v 1.1 2013/03/21 20:25:12 roberto Exp $
*/

#if !defined(lpcap_h)
#define lpcap_h




/* 捕获类型 */
typedef enum CapKind {
  Cclose, Cposition, Cconst, Cbackref, Carg, Csimple, Ctable, Cfunction,
  Cquery, Cstring, Cnum, Csubst, Cfold, Cruntime, Cgroup
} CapKind;


typedef struct Capture {
  const char *s;  /* 主题位置 */
  short idx;  /* 关于捕获的额外信息（组名、参数索引等） */
  byte kind;  /* 捕获类型 */
  byte siz;  /* 完整捕获的大小 + 1（0 = 不是完整捕获） */
} Capture;


typedef struct CapState {
  Capture *cap;  /* 当前捕获 */
  Capture *ocap;  /* 原始捕获列表 */
  lua_State *L;
  int ptop;  /* 'match' 的最后一个参数的索引 */
  const char *s;  /* 原始字符串 */
  int valuecached;  /* 存储在缓存槽中的值 */
} CapState;


int runtimecap (CapState *cs, Capture *close, const char *s, int *rem);  /* 运行时捕获 */
int getcaptures (lua_State *L, const char *s, const char *r, int ptop);  /* 获取捕获 */
int finddyncap (Capture *cap, Capture *last);  /* 查找动态捕获 */

#endif


/*
** $Id: lpvm.h,v 1.2 2013/04/03 20:37:18 roberto Exp $
*/

#if !defined(lpvm_h)
#define lpvm_h



/* 虚拟机指令 */
typedef enum Opcode {
  IAny, /* if no char, fail */  // 如果没有字符，匹配失败
  IChar,  /* if char != aux, fail */  // 如果字符不等于辅助字符，匹配失败
  ISet,  /* if char not in buff, fail */  // 如果字符不在缓冲区中，匹配失败
  ITestAny,  /* in no char, jump to 'offset' */  // 如果没有字符，跳转到指定偏移量
  ITestChar,  /* if char != aux, jump to 'offset' */  // 如果字符不等于辅助字符，跳转到指定偏移量
  ITestSet,  /* if char not in buff, jump to 'offset' */  // 如果字符不在缓冲区中，跳转到指定偏移量
  ISpan,  /* read a span of chars in buff */  // 从缓冲区中读取一段字符
  IBehind,  /* walk back 'aux' characters (fail if not possible) */  // 向后移动 'aux' 个字符（如果不可能则匹配失败）
  IRet,  /* return from a rule */  // 从规则中返回
  IEnd,  /* end of pattern */  // 模式结束
  IChoice,  /* stack a choice; next fail will jump to 'offset' */  // 堆栈一个选择；下一个失败将跳转到指定偏移量
  IJmp,  /* jump to 'offset' */  // 跳转到指定偏移量
  ICall,  /* call rule at 'offset' */  // 调用指定偏移量处的规则
  IOpenCall,  /* call rule number 'key' (must be closed to a ICall) */  // 调用规则号为 'key' 的规则（必须与 ICall 配对）
  ICommit,  /* pop choice and jump to 'offset' */  // 弹出选择并跳转到指定偏移量
  IPartialCommit,  /* update top choice to current position and jump */  // 更新顶部选择到当前位置并跳转
  IBackCommit,  /* "fails" but jump to its own 'offset' */  // “失败”但跳转到自己的偏移量
  IFailTwice,  /* pop one choice and then fail */  // 弹出一个选择然后失败
  IFail,  /* go back to saved state on choice and jump to saved offset */  // 返回到选择时保存的状态并跳转到保存的偏移量
  IGiveup,  /* internal use */  // 内部使用
  IFullCapture,  /* complete capture of last 'off' chars */  // 完全捕获最后 'off' 个字符
  IOpenCapture,  /* start a capture */  // 开始捕获
  ICloseCapture,  // 关闭捕获
  ICloseRunTime  // 运行时关闭
} Opcode;



typedef union Instruction {
  struct Inst {
    byte code;
    byte aux;
    short key;
  } i;
  int offset;
  byte buff[1];
} Instruction;


int getposition (lua_State *L, int t, int i);
void printpatt (Instruction *p, int n);
const char *match (lua_State *L, const char *o, const char *s, const char *e,
                   Instruction *op, Capture *capture, int ptop);
int verify (lua_State *L, Instruction *op, const Instruction *p,
            Instruction *e, int postable, int rule);
void checkrule (lua_State *L, Instruction *op, int from, int to,
                int postable, int rule);


#endif

/*
** $Id: lpcode.h,v 1.5 2013/04/04 21:24:45 roberto Exp $
*/

#if !defined(lpcode_h)
#define lpcode_h


int tocharset (TTree *tree, Charset *cs);
int checkaux (TTree *tree, int pred);
int fixedlenx (TTree *tree, int count, int len);
# 检查给定的 TTree 是否有捕获
int hascaptures (TTree *tree);

# 垃圾回收函数
int lp_gc (lua_State *L);

# 编译函数，将模式编译成指令序列
Instruction *compile (lua_State *L, Pattern *p);

# 重新分配模式对象的指令数组大小
void reallocprog (lua_State *L, Pattern *p, int nsize);

# 返回指令的大小
int sizei (const Instruction *i);

# 定义 PEnullable 和 PEnofail 的值
#define PEnullable      0
#define PEnofail        1

# 检查给定的 t 是否为 PEnofail
#define nofail(t)    checkaux(t, PEnofail)

# 检查给定的 t 是否为 PEnullable
#define nullable(t)    checkaux(t, PEnullable)

# 获取固定长度
#define fixedlen(t)     fixedlenx(t, 0, 0)

# 结束宏定义部分

# 如果未定义 lpprint_h，则定义 lpprint_h
#if !defined(lpprint_h)
#define lpprint_h

# 如果定义了 LPEG_DEBUG，则定义以下函数
#if defined(LPEG_DEBUG)

# 打印指令序列
void printpatt (Instruction *p, int n);

# 打印语法树
void printtree (TTree *tree, int ident);

# 打印 Lua 表
void printktable (lua_State *L, int idx);

# 打印字符集
void printcharset (const byte *st);

# 打印捕获列表
void printcaplist (Capture *cap, Capture *limit);

# 否则，定义以下宏
#else

# 抛出错误，表示函数只在调试模式下实现
#define printktable(L,idx)  \
    luaL_error(L, "function only implemented in debug mode")
#define printtree(tree,i)  \
    luaL_error(L, "function only implemented in debug mode")
#define printpatt(p,n)  \
    luaL_error(L, "function only implemented in debug mode")

#endif

# 结束宏定义部分

# 结束 lpprint_h 的定义
#endif

# 定义 captype 宏
#define captype(cap)    ((cap)->kind)

# 判断是否为闭合捕获
#define isclosecap(cap)    (captype(cap) == Cclose)

# 获取闭合捕获的地址
#define closeaddr(c)    ((c)->s + (c)->siz - 1)

# 判断是否为完整捕获
#define isfullcap(cap)    ((cap)->siz != 0)

# 从 ktable 中获取 Lua 值
#define getfromktable(cs,v)    lua_rawgeti((cs)->L, ktableidx((cs)->ptop), v)

# 将 Lua 值推入栈中
#define pushluaval(cs)        getfromktable(cs, (cs)->cap->idx)

# 更新缓存中的 Lua 值，如果不存在则将其放入缓存并返回其索引
static int updatecache (CapState *cs, int v) {
  int idx = cs->ptop + 1;  /* stack index of cache for Lua values */
  if (v != cs->valuecached) {  /* not there? */
    getfromktable(cs, v);  /* get value from 'ktable' */
    lua_replace(cs->L, idx);  /* put it at reserved stack position */
    cs->valuecached = v;  /* 将变量v的值赋给cs->valuecached，用于跟踪记录当前的数值 */
  }
  return idx;  /* 返回变量idx的值 */
# 静态函数，用于在捕获列表中查找对应的开放捕获
static Capture *findopen (Capture *cap) {
  int n = 0;  /* 等待开放捕获的关闭捕获数量 */
  for (;;) {
    cap--;  # 向前遍历捕获列表
    if (isclosecap(cap)) n++;  # 如果是关闭捕获，则等待开放捕获数量加一
    else if (!isfullcap(cap))  # 如果不是完整捕获
      if (n-- == 0) return cap;  # 如果等待开放捕获数量减一后为零，则返回当前捕获
  }
}

# 移动到下一个捕获
static void nextcap (CapState *cs) {
  Capture *cap = cs->cap;  # 获取当前捕获
  if (!isfullcap(cap)) {  /* 不是单个捕获？ */
    int n = 0;  /* 等待关闭捕获的开放捕获数量 */
    for (;;) {  /* 寻找对应的关闭捕获 */
      cap++;  # 向后遍历捕获列表
      if (isclosecap(cap)) {
        if (n-- == 0) break;  # 如果等待关闭捕获的开放捕获数量减一后为零，则跳出循环
      }
      else if (!isfullcap(cap)) n++;  # 如果不是完整捕获，则等待关闭捕获的开放捕获数量加一
    }
  }
  cs->cap = cap + 1;  /* + 1 用于跳过最后一个关闭捕获（或整个单个捕获） */
}

# 将当前捕获内部生成的所有值推送到 Lua 栈上
static int pushnestedvalues (CapState *cs, int addextra) {
  Capture *co = cs->cap;  # 获取当前捕获
  if (isfullcap(cs->cap++)) {  /* 没有嵌套捕获？ */
    lua_pushlstring(cs->L, co->s, co->siz - 1);  # 推送整个匹配
    return 1;  # 返回推送的值数量
  }
  else {
    int n = 0;
    while (!isclosecap(cs->cap))  # 重复推送所有嵌套模式的值
      n += pushcapture(cs);
    if (addextra || n == 0) {  /* 需要额外的值？ */
      lua_pushlstring(cs->L, co->s, cs->cap->s - co->s);  # 推送整个匹配
      n++;
    }
    cs->cap++;  # 跳过关闭捕获条目
    return n;
  }
}

# 仅推送嵌套捕获生成的第一个值
static void pushonenestedvalue (CapState *cs) {
  int n = pushnestedvalues(cs, 0);  # 推送嵌套值，不需要额外的值
  if (n > 1)
    lua_pop(cs->L, n - 1);  # 弹出额外的值
}
# 尝试在堆栈顶部给定的名称中查找命名组捕获；从'cap'向后查找
static Capture *findback (CapState *cs, Capture *cap) {
  lua_State *L = cs->L;
  while (cap-- > cs->ocap) {  /* 重复直到列表结束 */
    if (isclosecap(cap))
      cap = findopen(cap);  /* 跳过嵌套捕获 */
    else if (!isfullcap(cap))
      continue; /* 开启一个封闭捕获：跳过并获取前一个 */
    if (captype(cap) == Cgroup) {
      getfromktable(cs, cap->idx);  /* 获取组名 */
      if (lua_equal(L, -2, -1)) {  /* 正确的组？ */
        lua_pop(L, 2);  /* 移除引用名称和组名 */
        return cap;
      }
      else lua_pop(L, 1);  /* 移除组名 */
    }
  }
  luaL_error(L, "back reference '%s' not found", lua_tostring(L, -1));
  return NULL;  /* 为了避免警告 */
}


/*
** 回溯引用捕获。返回推送的值数量。
*/
static int backrefcap (CapState *cs) {
  int n;
  Capture *curr = cs->cap;
  pushluaval(cs);  /* 引用名称 */
  cs->cap = findback(cs, curr);  /* 查找对应的组 */
  n = pushnestedvalues(cs, 0);  /* 推送组的值 */
  cs->cap = curr + 1;
  return n;
}


/*
** 表捕获：创建一个新表并用嵌套捕获填充它。
*/
static int tablecap (CapState *cs) {
  lua_State *L = cs->L;
  int n = 0;
  lua_newtable(L);
  if (isfullcap(cs->cap++))
    return 1;  /* 表为空 */
  while (!isclosecap(cs->cap)) {
    if (captype(cs->cap) == Cgroup && cs->cap->idx != 0) {  /* 命名组？ */
      pushluaval(cs);  /* 推送组名 */
      pushonenestedvalue(cs);
      lua_settable(L, -3);
    }
    else {  /* 不是命名组 */
      int i;
      int k = pushcapture(cs);
      for (i = k; i > 0; i--)  /* 将所有值存储到表中 */
        lua_rawseti(L, -(i + 1), n + i);
      n += k;
    }
  }
  cs->cap++;  /* 跳过关闭条目 */
  return 1;  /* 推送的值数量（只有表） */
}


/*
** 表查询捕获
*/
# 查询捕获值
static int querycap (CapState *cs) {
  int idx = cs->cap->idx;  /* 获取捕获的索引 */
  pushonenestedvalue(cs);  /* 获取嵌套捕获 */
  lua_gettable(cs->L, updatecache(cs, idx));  /* 在表中查询捕获值 */
  if (!lua_isnil(cs->L, -1))  /* 如果不是 nil 值 */
    return 1;  /* 返回 1 */
  else {  /* 没有值 */
    lua_pop(cs->L, 1);  /* 移除 nil 值 */
    return 0;  /* 返回 0 */
  }
}


/*
** 折叠捕获
*/
static int foldcap (CapState *cs) {
  int n;
  lua_State *L = cs->L;
  int idx = cs->cap->idx;  /* 获取捕获的索引 */
  if (isfullcap(cs->cap++) ||  /* 没有嵌套捕获？ */
      isclosecap(cs->cap) ||  /* 没有嵌套捕获（大主题）？ */
      (n = pushcapture(cs)) == 0)  /* 嵌套捕获没有值？ */
    return luaL_error(L, "no initial value for fold capture");  /* 报错：折叠捕获没有初始值 */
  if (n > 1)
    lua_pop(L, n - 1);  /* 只留下一个结果作为累加器 */
  while (!isclosecap(cs->cap)) {
    lua_pushvalue(L, updatecache(cs, idx));  /* 获取折叠函数 */
    lua_insert(L, -2);  /* 将其放在累加器之前 */
    n = pushcapture(cs);  /* 获取下一个捕获的值 */
    lua_call(L, n + 1, 1);  /* 调用折叠函数 */
  }
  cs->cap++;  /* 跳过关闭条目 */
  return 1;  /* 栈上只剩下累加器 */
}


/*
** 函数捕获
*/
static int functioncap (CapState *cs) {
  int n;
  int top = lua_gettop(cs->L);
  pushluaval(cs);  /* 推入函数 */
  n = pushnestedvalues(cs, 0);  /* 推入嵌套捕获 */
  lua_call(cs->L, n, LUA_MULTRET);  /* 调用函数 */
  return lua_gettop(cs->L) - top;  /* 返回函数的结果 */
}


/*
** 选择捕获
*/
static int numcap (CapState *cs) {
  int idx = cs->cap->idx;  /* 要选择的值 */
  if (idx == 0) {  /* 没有值？ */
    nextcap(cs);  /* 跳过整个捕获 */
    return 0;  /* 没有产生值 */
  }
  else {
    int n = pushnestedvalues(cs, 0);
    if (n < idx)  /* 无效的索引？ */
      return luaL_error(cs->L, "no capture '%d'", idx);  /* 报错：没有捕获 '%d' */
  }
}
    else {
      // 将栈顶的第 n - idx + 1 个值复制一份压入栈顶
      lua_pushvalue(cs->L, -(n - idx + 1));  /* get selected capture */
      // 将栈顶的值替换掉栈顶的第 n + 1 个值
      lua_replace(cs->L, -(n + 1));  /* put it in place of 1st capture */
      // 弹出栈顶的 n - 1 个值
      lua_pop(cs->L, n - 1);  /* remove other captures */
      // 返回 1
      return 1;
    }
  }
# 返回给定捕获列表中第一个运行时捕获的堆栈索引（如果没有运行时捕获，则返回零）
int finddyncap (Capture *cap, Capture *last) {
  # 遍历捕获列表，查找第一个运行时捕获
  for (; cap < last; cap++) {
    if (cap->kind == Cruntime)
      return cap->idx;  # 返回第一个捕获的堆栈位置
  }
  return 0;  # 在此段中没有动态捕获
}


# 调用运行时捕获。返回调用移除的捕获数量，包括初始的 Cgroup。（要添加的捕获在 Lua 栈上。）
int runtimecap (CapState *cs, Capture *close, const char *s, int *rem) {
  int n, id;
  lua_State *L = cs->L;
  int otop = lua_gettop(L);
  Capture *open = findopen(close);
  assert(captype(open) == Cgroup);
  id = finddyncap(open, close);  # 获取第一个动态捕获参数
  close->kind = Cclose;  # 关闭组
  close->s = s;
  cs->cap = open; cs->valuecached = 0;  # 准备捕获状态
  luaL_checkstack(L, 4, "too many runtime captures");
  pushluaval(cs);  # 推送要调用的函数
  lua_pushvalue(L, SUBJIDX);  # 推送原始主题
  lua_pushinteger(L, s - cs->s + 1);  # 推送当前位置
  n = pushnestedvalues(cs, 0);  # 推送嵌套捕获
  lua_call(L, n + 2, LUA_MULTRET);  # 调用动态函数
  if (id > 0) {  # 是否有旧的动态捕获需要移除？
    int i;
    for (i = id; i <= otop; i++)
      lua_remove(L, id);  # 移除旧的动态捕获
    *rem = otop - id + 1;  # 移除的动态捕获总数
  }
  else
    *rem = 0;  # 没有移除动态捕获
  return close - open;  # 移除的所有类型的捕获数量
}


# 用于替换和字符串捕获的辅助结构：保留关于嵌套捕获的信息，以便将来使用，避免将字符串结果推送到 Lua 中
typedef struct StrAux {
  int isstring;  # 捕获是否为字符串
  union {
    Capture *cp;  # 如果不是字符串，则是相应的捕获
    // 定义一个结构体，用于存储字符串的起始和结束位置
    struct {  /* if it is a string... */
      const char *s;  /* ... starts here */
      const char *e;  /* ... ends here */
    } s;
  } u;
} StrAux;

#define MAXSTRCAPS    10

/*
** Collect values from current capture into array 'cps'. Current
** capture must be Cstring (first call) or Csimple (recursive calls).
** (In first call, fills %0 with whole match for Cstring.)
** Returns number of elements in the array that were filled.
*/
static int getstrcaps (CapState *cs, StrAux *cps, int n) {
  int k = n++;
  cps[k].isstring = 1;  /* get string value */
  cps[k].u.s.s = cs->cap->s;  /* starts here */
  if (!isfullcap(cs->cap++)) {  /* nested captures? */
    while (!isclosecap(cs->cap)) {  /* traverse them */
      if (n >= MAXSTRCAPS)  /* too many captures? */
        nextcap(cs);  /* skip extra captures (will not need them) */
      else if (captype(cs->cap) == Csimple)  /* string? */
        n = getstrcaps(cs, cps, n);  /* put info. into array */
      else {
        cps[n].isstring = 0;  /* not a string */
        cps[n].u.cp = cs->cap;  /* keep original capture */
        nextcap(cs);
        n++;
      }
    }
    cs->cap++;  /* skip close */
  }
  cps[k].u.s.e = closeaddr(cs->cap - 1);  /* ends here */
  return n;
}


/*
** add next capture value (which should be a string) to buffer 'b'
*/
static int addonestring (luaL_Buffer *b, CapState *cs, const char *what);


/*
** String capture: add result to buffer 'b' (instead of pushing
** it into the stack)
*/
static void stringcap (luaL_Buffer *b, CapState *cs) {
  StrAux cps[MAXSTRCAPS];  // 创建一个 StrAux 类型的数组，用于存储捕获的值
  int n;  // 定义一个整型变量 n
  size_t len, i;  // 定义两个 size_t 类型的变量 len 和 i
  const char *fmt;  // 定义一个指向字符常量的指针 fmt
  fmt = lua_tolstring(cs->L, updatecache(cs, cs->cap->idx), &len);  // 获取格式字符串和长度
  n = getstrcaps(cs, cps, 0) - 1;  // 收集嵌套捕获的值，并将结果减去 1 赋给 n
  for (i = 0; i < len; i++) {  // 遍历格式字符串
    if (fmt[i] != '%')  // 如果不是转义字符
      luaL_addchar(b, fmt[i]);  // 将字符添加到缓冲区
    else if (fmt[++i] < '0' || fmt[i] > '9')  // 如果后面不是数字
      luaL_addchar(b, fmt[i]);  // 将字符添加到缓冲区
    else {
      int l = fmt[i] - '0';  /* 捕获索引 */
      if (l > n)
        luaL_error(cs->L, "无效的捕获索引 (%d)", l);
      else if (cps[l].isstring)
        luaL_addlstring(b, cps[l].u.s.s, cps[l].u.s.e - cps[l].u.s.s);
      else {
        Capture *curr = cs->cap;
        cs->cap = cps[l].u.cp;  /* 回到评估嵌套捕获的位置 */
        if (!addonestring(b, cs, "capture"))
          luaL_error(cs->L, "捕获索引 %d 中没有值", l);
        cs->cap = curr;  /* 从停止的地方继续 */
      }
    }
  }
/*
** Substitution capture: add result to buffer 'b'
*/
static void substcap (luaL_Buffer *b, CapState *cs) {
  // 获取当前捕获的字符串起始位置
  const char *curr = cs->cap->s;
  // 如果没有嵌套捕获
  if (isfullcap(cs->cap))  /* no nested captures? */
    // 将原始文本添加到缓冲区
    luaL_addlstring(b, curr, cs->cap->siz - 1);  /* keep original text */
  else {
    cs->cap++;  /* skip open entry */
    // 遍历嵌套捕获
    while (!isclosecap(cs->cap)) {  /* traverse nested captures */
      // 获取下一个捕获的字符串起始位置
      const char *next = cs->cap->s;
      // 将当前捕获到下一个捕获之间的文本添加到缓冲区
      luaL_addlstring(b, curr, next - curr);  /* add text up to capture */
      // 如果添加替换字符串成功
      if (addonestring(b, cs, "replacement"))
        // 继续在匹配后的位置添加文本
        curr = closeaddr(cs->cap - 1);  /* continue after match */
      else  /* no capture value */
        // 保持原始文本在最终结果中
        curr = next;  /* keep original text in final result */
    }
    // 添加最后一段文本到缓冲区
    luaL_addlstring(b, curr, cs->cap->s - curr);  /* add last piece of text */
  }
  // 转到下一个捕获
  cs->cap++;  /* go to next capture */
}


/*
** Evaluates a capture and adds its first value to buffer 'b'; returns
** whether there was a value
*/
static int addonestring (luaL_Buffer *b, CapState *cs, const char *what) {
  switch (captype(cs->cap)) {
    case Cstring:
      // 直接将捕获的字符串添加到缓冲区
      stringcap(b, cs);  /* add capture directly to buffer */
      return 1;
    case Csubst:
      // 直接将捕获的字符串添加到缓冲区
      substcap(b, cs);  /* add capture directly to buffer */
      return 1;
    default: {
      lua_State *L = cs->L;
      // 将当前捕获的所有值推入栈中
      int n = pushcapture(cs);
      if (n > 0) {
        if (n > 1) lua_pop(L, n - 1);  /* only one result */
        // 如果栈顶不是字符串类型，则抛出错误
        if (!lua_isstring(L, -1))
          luaL_error(L, "invalid %s value (a %s)", what, luaL_typename(L, -1));
        // 将栈顶值添加到缓冲区
        luaL_addvalue(b);
      }
      return n;
    }
  }
}


/*
** Push all values of the current capture into the stack; returns
** number of values pushed
*/
static int pushcapture (CapState *cs) {
  lua_State *L = cs->L;
  luaL_checkstack(L, 4, "too many captures");
  switch (captype(cs->cap)) {
    case Cposition: {
      // 将当前捕获的位置推入栈中
      lua_pushinteger(L, cs->cap->s - cs->s + 1);
      cs->cap++;
      return 1;
    }
    case Cconst: {
      // 将当前捕获的常量值推入栈中
      pushluaval(cs);
      cs->cap++;
      return 1;
      // 以下代码未完整，无法确定具体作用
    }
    case Carg: {
      // 获取参数索引
      int arg = (cs->cap++)->idx;
      // 检查参数索引是否有效
      if (arg + FIXEDARGS > cs->ptop)
        return luaL_error(L, "reference to absent argument #%d", arg);
      // 将参数值压入 Lua 栈
      lua_pushvalue(L, arg + FIXEDARGS);
      return 1;
    }
    case Csimple: {
      // 将嵌套值压入 Lua 栈
      int k = pushnestedvalues(cs, 1);
      // 将整个匹配结果放在第一个位置
      lua_insert(L, -k);  /* make whole match be first result */
      return k;
    }
    case Cruntime: {
      // 将栈中的值压入 Lua 栈
      lua_pushvalue(L, (cs->cap++)->idx);  /* value is in the stack */
      return 1;
    }
    case Cstring: {
      // 初始化缓冲区
      luaL_Buffer b;
      luaL_buffinit(L, &b);
      // 将字符串捕获结果放入缓冲区
      stringcap(&b, cs);
      // 将缓冲区中的内容压入 Lua 栈
      luaL_pushresult(&b);
      return 1;
    }
    case Csubst: {
      // 初始化缓冲区
      luaL_Buffer b;
      luaL_buffinit(L, &b);
      // 将替换捕获结果放入缓冲区
      substcap(&b, cs);
      // 将缓冲区中的内容压入 Lua 栈
      luaL_pushresult(&b);
      return 1;
    }
    case Cgroup: {
      // 检查是否为匿名组
      if (cs->cap->idx == 0)  /* anonymous group? */
        // 添加所有嵌套值
        return pushnestedvalues(cs, 0);  /* add all nested values */
      else {  /* named group: add no values */
        // 跳过捕获
        nextcap(cs);  /* skip capture */
        return 0;
      }
    }
    case Cbackref: 
      // 返回反向引用捕获结果
      return backrefcap(cs);
    case Ctable: 
      // 返回表捕获结果
      return tablecap(cs);
    case Cfunction: 
      // 返回函数捕获结果
      return functioncap(cs);
    case Cnum: 
      // 返回数字捕获结果
      return numcap(cs);
    case Cquery: 
      // 返回查询捕获结果
      return querycap(cs);
    case Cfold: 
      // 返回折叠捕获结果
      return foldcap(cs);
    default: 
      // 断言，不应该执行到这里
      assert(0); 
      return 0;
  }
}
/*
** 准备一个 CapState 结构并遍历堆栈中的所有捕获，推送其结果。
** 's' 是主题字符串，'r' 是匹配的最终位置，'ptop' 是堆栈中推送一些有用值的索引。
** 返回推送的结果数量。（如果列表没有产生结果，则推送匹配的最终位置。）
*/
int getcaptures (lua_State *L, const char *s, const char *r, int ptop) {
  // 从堆栈中获取 Capture 结构的用户数据
  Capture *capture = (Capture *)lua_touserdata(L, caplistidx(ptop));
  int n = 0;
  // 检查是否有捕获
  if (!isclosecap(capture)) {  /* is there any capture? */
    CapState cs;
    cs.ocap = cs.cap = capture; cs.L = L;
    cs.s = s; cs.valuecached = 0; cs.ptop = ptop;
    // 收集捕获的值
    do {
      n += pushcapture(&cs);
    } while (!isclosecap(cs.cap));
  }
  // 如果没有捕获值
  if (n == 0) {
    // 只返回结束位置
    lua_pushinteger(L, r - s + 1);
    n = 1;
  }
  return n;
}

/*
** $Id: lpcode.c,v 1.18 2013/04/12 16:30:33 roberto Exp $
** 版权所有 2007, Lua.org & PUC-Rio  (查看 'lpeg.html' 获取许可证)
*/

#include <limits.h>

/* 表示“无指令” */
#define NOINST        -1

// 完整字符集
static const Charset fullset_ =
  {{0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF,
    0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF,
    0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF,
    0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF}};

static const Charset *fullset = &fullset_;

/*
** {======================================================
** 分析和一些优化
** =======================================================
*/

/*
** 检查字符集是否为空（IFail）、单字符（IChar）、全集（IAny），或者都不是（ISet）。
*/
static Opcode charsettype (const byte *cs, int *c) {
  int count = 0;
  int i;
  int candidate = -1;  /* 候选字符的位置 */
  for (i = 0; i < CHARSETSIZE; i++) {
    int b = cs[i];
    # 如果字节为0
    if (b == 0) {
      # 如果计数大于1，则返回ISet；否则集合仍然为空
      if (count > 1) return ISet;  
    }
    # 如果字节为0xFF
    else if (b == 0xFF) {
      # 如果计数小于（i * BITSPERCHAR），则返回ISet；否则计数增加BITSPERCHAR，集合仍然是满的
      if (count < (i * BITSPERCHAR))
        return ISet;
      else count += BITSPERCHAR;  
    }
    # 如果字节只有一个位被设置
    else if ((b & (b - 1)) == 0) {  
      # 如果计数大于0，则返回ISet；否则，集合既不是满的也不是空的
      if (count > 0)
        return ISet;  
      else {  
        # 集合目前只有一个字符；跟踪它
        count++;
        candidate = i;
      }
    }
    else return ISet;  # 字节既不是空的、满的，也不是单字符的
  }
  # 切换计数
  switch (count) {
    # 空集
    case 0: return IFail;  
    # 单字符；在字节中找到字符位
    case 1: {  
      int b = cs[candidate];
      *c = candidate * BITSPERCHAR;
      if ((b & 0xF0) != 0) { *c += 4; b >>= 4; }
      if ((b & 0x0C) != 0) { *c += 2; b >>= 2; }
      if ((b & 0x02) != 0) { *c += 1; }
      return IChar;
    }
    # 完整集合
    default: {
       assert(count == CHARSETSIZE * BITSPERCHAR);  
       return IAny;
    }
  }
/* 
** A few basic operations on Charsets
*/
// 对字符集进行几个基本操作

// 对字符集取反
static void cs_complement (Charset *cs) {
  loopset(i, cs->cs[i] = ~cs->cs[i]);
}

// 判断两个字符集是否相等
static int cs_equal (const byte *cs1, const byte *cs2) {
  loopset(i, if (cs1[i] != cs2[i]) return 0);
  return 1;
}

// 计算字符集 cs1 和 cs2 是否不相交
static int cs_disjoint (const Charset *cs1, const Charset *cs2) {
  loopset(i, if ((cs1->cs[i] & cs2->cs[i]) != 0) return 0;)
  return 1;
}

// 将 'char' 模式（TSet、TChar、TAny）转换为字符集
int tocharset (TTree *tree, Charset *cs) {
  switch (tree->tag) {
    case TSet: {  /* copy set */
      loopset(i, cs->cs[i] = treebuffer(tree)[i]);
      return 1;
    }
    case TChar: {  /* only one char */
      assert(0 <= tree->u.n && tree->u.n <= UCHAR_MAX);
      loopset(i, cs->cs[i] = 0);  /* erase all chars */
      setchar(cs->cs, tree->u.n);  /* add that one */
      return 1;
    }
    case TAny: {
      loopset(i, cs->cs[i] = 0xFF);  /* add all to the set */
      return 1;
    }
    default: return 0;
  }
}

// 检查模式是否具有捕获
int hascaptures (TTree *tree) {
 tailcall:
  switch (tree->tag) {
    case TCapture: case TRunTime:
      return 1;
    default: {
      switch (numsiblings[tree->tag]) {
        case 1:  /* return hascaptures(sib1(tree)); */
          tree = sib1(tree); goto tailcall;
        case 2:
          if (hascaptures(sib1(tree))) return 1;
          /* else return hascaptures(sib2(tree)); */
          tree = sib2(tree); goto tailcall;
        default: assert(numsiblings[tree->tag] == 0); return 0;
      }
    }
  }
}

// 检查模式在空字符串方面的行为
// 一种是 *nullable*，如果它可以匹配而不消耗任何字符
// 一种是 *nofail*，如果它对任何字符串（包括空字符串）都不会失败
// 这两种属性对于谓词和运行时捕获来说是不同的
// 对于其他模式，这两种属性是等价的
/*
** (With predicates, &'a' is nullable but not nofail. Of course,
** nofail => nullable.)
** These functions are all convervative in the following way:
**    p is nullable => nullable(p)
**    nofail(p) => p cannot fail
** The function assumes that TOpenCall is not nullable;
** this will be checked again when the grammar is fixed.)
** Run-time captures can do whatever they want, so the result
** is conservative.
*/
int checkaux (TTree *tree, int pred) {
 tailcall:
  switch (tree->tag) {
    case TChar: case TSet: case TAny:
    case TFalse: case TOpenCall:
      return 0;  /* not nullable */
    case TRep: case TTrue:
      return 1;  /* no fail */
    case TNot: case TBehind:  /* can match empty, but can fail */
      if (pred == PEnofail) return 0;
      else return 1;  /* PEnullable */
    case TAnd:  /* can match empty; fail iff body does */
      if (pred == PEnullable) return 1;
      /* else return checkaux(sib1(tree), pred); */
      tree = sib1(tree); goto tailcall;
    case TRunTime:  /* can fail; match empty iff body does */
      if (pred == PEnofail) return 0;
      /* else return checkaux(sib1(tree), pred); */
      tree = sib1(tree); goto tailcall;
    case TSeq:
      if (!checkaux(sib1(tree), pred)) return 0;
      /* else return checkaux(sib2(tree), pred); */
      tree = sib2(tree); goto tailcall;
    case TChoice:
      if (checkaux(sib2(tree), pred)) return 1;
      /* else return checkaux(sib1(tree), pred); */
      tree = sib1(tree); goto tailcall;
    case TCapture: case TGrammar: case TRule:
      /* return checkaux(sib1(tree), pred); */
      tree = sib1(tree); goto tailcall;
    case TCall:  /* return checkaux(sib2(tree), pred); */
      tree = sib2(tree); goto tailcall;
    default: assert(0); return 0;
  };
}


/*
** number of characters to match a pattern (or -1 if variable)
** ('count' avoids infinite loops for grammars)
*/
int fixedlenx (TTree *tree, int count, int len) {
 tailcall:
  switch (tree->tag) {
    # 对于字符、字符集、任意字符，返回长度加1
    case TChar: case TSet: case TAny:
      return len + 1;
    # 对于假、真、非、与、回溯，返回长度
    case TFalse: case TTrue: case TNot: case TAnd: case TBehind:
      return len;
    # 对于重复、运行时、开放调用，返回-1
    case TRep: case TRunTime: case TOpenCall:
      return -1;
    # 对于捕获、规则、语法，跳转到相应的子节点并进行尾递归
    case TCapture: case TRule: case TGrammar:
      /* return fixedlenx(sib1(tree), count); */
      tree = sib1(tree); goto tailcall;
    # 对于调用，如果计数超过最大规则数，则返回-1，否则跳转到相应的子节点并进行尾递归
    case TCall:
      if (count++ >= MAXRULES)
        return -1;  /* may be a loop */
      /* else return fixedlenx(sib2(tree), count); */
      tree = sib2(tree); goto tailcall;
    # 对于序列，计算第一个子节点的长度，如果小于0则返回-1，否则计算第二个子节点的长度
    case TSeq: {
      len = fixedlenx(sib1(tree), count, len);
      if (len < 0) return -1;
      /* else return fixedlenx(sib2(tree), count, len); */
      tree = sib2(tree); goto tailcall;
    }
    # 对于选择，计算两个子节点的长度，如果相等则返回长度，否则返回-1
    case TChoice: {
      int n1, n2;
      n1 = fixedlenx(sib1(tree), count, len);
      if (n1 < 0) return -1;
      n2 = fixedlenx(sib2(tree), count, len);
      if (n1 == n2) return n1;
      else return -1;
    }
    # 默认情况下断言失败，返回0
    default: assert(0); return 0;
  };
/*
** 计算模式的“first set”。
** 结果是一个保守的近似：
**   匹配 p ax -> x' 对于某个 x ==> a 在 first(p) 中。
** 集合 'follow' 是模式后面的内容的 first set（如果后面没有内容，则是完整的集合）。
** 当这个集合可以用于避免模式时，函数返回 0。
** 非零返回有两种情况：
** 1）匹配 p '' -> ''            ==> 返回 1。
** （测试不能用于空输入，因为它们总是失败的）
** 2）存在匹配时间捕获 ==> 返回 2。
** （匹配时间捕获不应该被优化避免）
*/
static int getfirst (TTree *tree, const Charset *follow, Charset *firstset) {
 tailcall:
  switch (tree->tag) {
    case TChar: case TSet: case TAny: {
      tocharset(tree, firstset);
      return 0;
    }
    case TTrue: {
      loopset(i, firstset->cs[i] = follow->cs[i]);
      return 1;
    }
    case TFalse: {
      loopset(i, firstset->cs[i] = 0);
      return 0;
    }
    case TChoice: {
      Charset csaux;
      int e1 = getfirst(sib1(tree), follow, firstset);
      int e2 = getfirst(sib2(tree), follow, &csaux);
      loopset(i, firstset->cs[i] |= csaux.cs[i]);
      return e1 | e2;
    }
    case TSeq: {
      if (!nullable(sib1(tree))) {
        /* return getfirst(sib1(tree), fullset, firstset); */
        tree = sib1(tree); follow = fullset; goto tailcall;
      }
      else {  /* FIRST(p1 p2, fl) = FIRST(p1, FIRST(p2, fl)) */
        Charset csaux;
        int e2 = getfirst(sib2(tree), follow, &csaux);
        int e1 = getfirst(sib1(tree), &csaux, firstset);
        if (e1 == 0) return 0;  /* 'e1' ensures that first can be used */
        else if ((e1 | e2) & 2)  /* one of the children has a matchtime? */
          return 2;  /* pattern has a matchtime capture */
        else return e2;  /* else depends on 'e2' */
      }
    }
    # 对于 TRep 类型的节点
    case TRep: {
      # 获取子节点的 first 集合，并将其与 follow 集合合并
      getfirst(sib1(tree), follow, firstset);
      loopset(i, firstset->cs[i] |= follow->cs[i]);
      # 返回 1，表示接受空字符串
      return 1;  /* accept the empty string */
    }
    # 对于 TCapture、TGrammar、TRule 类型的节点
    case TCapture: case TGrammar: case TRule: {
      # 返回 sib1(tree) 的 first 集合
      /* return getfirst(sib1(tree), follow, firstset); */
      # 转到尾递归
      tree = sib1(tree); goto tailcall;
    }
    # 对于 TRunTime 类型的节点
    case TRunTime: {  /* function invalidates any follow info. */
      # 获取 sib1(tree) 的 first 集合，并将其与 fullset 合并
      int e = getfirst(sib1(tree), fullset, firstset);
      # 如果 e 不为 0，则返回 2，表示函数不是 "protected"
      if (e) return 2;  /* function is not "protected"? */
      # 否则返回 0，表示捕获内的模式确保可以使用 first 集合
      else return 0;  /* pattern inside capture ensures first can be used */
    }
    # 对于 TCall 类型的节点
    case TCall: {
      # 返回 sib2(tree) 的 first 集合
      /* return getfirst(sib2(tree), follow, firstset); */
      # 转到尾递归
      tree = sib2(tree); goto tailcall;
    }
    # 对于 TAnd 类型的节点
    case TAnd: {
      # 获取 sib1(tree) 的 first 集合，并将其与 follow 集合交集
      int e = getfirst(sib1(tree), follow, firstset);
      loopset(i, firstset->cs[i] &= follow->cs[i]);
      return e;
    }
    # 对于 TNot 类型的节点
    case TNot: {
      # 如果 sib1(tree) 转换为字符集成功
      if (tocharset(sib1(tree), firstset)) {
        # 对 firstset 取补集
        cs_complement(firstset);
        return 1;
      }
      # 否则继续执行
      /* else go through */
    }
    # 对于 TBehind 类型的节点
    case TBehind: {  /* instruction gives no new information */
      # 调用 'getfirst' 检查数学运行时捕获
      int e = getfirst(sib1(tree), follow, firstset);
      loopset(i, firstset->cs[i] = follow->cs[i]);  /* 使用 follow 集合 */
      return e | 1;  /* 总是可以接受空字符串 */
    }
    # 默认情况，断言失败
    default: assert(0); return 0;
  }
/* 
** 如果返回 true，则模式只能在主题的下一个字符上失败
*/
static int headfail (TTree *tree) {
 tailcall:
  switch (tree->tag) {
    case TChar: case TSet: case TAny: case TFalse:
      return 1;
    case TTrue: case TRep: case TRunTime: case TNot:
    case TBehind:
      return 0;
    case TCapture: case TGrammar: case TRule: case TAnd:
      tree = sib1(tree); goto tailcall;  /* return headfail(sib1(tree)); */
    case TCall:
      tree = sib2(tree); goto tailcall;  /* return headfail(sib2(tree)); */
    case TSeq:
      if (!nofail(sib2(tree))) return 0;
      /* else return headfail(sib1(tree)); */
      tree = sib1(tree); goto tailcall;
    case TChoice:
      if (!headfail(sib1(tree))) return 0;
      /* else return headfail(sib2(tree)); */
      tree = sib2(tree); goto tailcall;
    default: assert(0); return 0;
  }
}


/*
** 检查给定树的代码生成是否可以受益于跟随集（以避免在不需要时计算跟随集）
*/
static int needfollow (TTree *tree) {
 tailcall:
  switch (tree->tag) {
    case TChar: case TSet: case TAny:
    case TFalse: case TTrue: case TAnd: case TNot:
    case TRunTime: case TGrammar: case TCall: case TBehind:
      return 0;
    case TChoice: case TRep:
      return 1;
    case TCapture:
      tree = sib1(tree); goto tailcall;
    case TSeq:
      tree = sib2(tree); goto tailcall;
    default: assert(0); return 0;
  }
}

/* }====================================================== */



/*
** {======================================================
** 代码生成
** =======================================================
*/


/*
** 指令的大小
*/
int sizei (const Instruction *i) {
  switch((Opcode)i->i.code) {
    case ISet: case ISpan: return CHARSETINSTSIZE;
    case ITestSet: return CHARSETINSTSIZE + 1;
    case ITestChar: case ITestAny: case IChoice: case IJmp:
    # 如果语句是ICall、IOpenCall、ICommit、IPartialCommit、IBackCommit中的任意一个，返回2
    case ICall: case IOpenCall: case ICommit: case IPartialCommit:
    case IBackCommit: return 2;
    # 如果不是以上情况，返回1
    default: return 1;
  }
/*
** 编译器的状态
*/
typedef struct CompileState {
  Pattern *p;  /* 正在编译的模式 */
  int ncode;  /* p->code 中下一个要填充的位置 */
  lua_State *L;
} CompileState;


/*
** 代码生成是递归的；'opt' 表示代码是在 'IChoice' 操作符下生成的，跳转到其结尾。
** 'tt' 指向保护此代码的先前测试。'fl' 是模式的跟随集。
*/
static void codegen (CompileState *compst, TTree *tree, int opt, int tt,
                     const Charset *fl);


void reallocprog (lua_State *L, Pattern *p, int nsize) {
  void *ud;
  lua_Alloc f = lua_getallocf(L, &ud);
  void *newblock = f(ud, p->code, p->codesize * sizeof(Instruction),
                                  nsize * sizeof(Instruction));
  if (newblock == NULL && nsize > 0)
    luaL_error(L, "not enough memory");
  p->code = (Instruction *)newblock;
  p->codesize = nsize;
}


static int nextinstruction (CompileState *compst) {
  int size = compst->p->codesize;
  if (compst->ncode >= size)
    reallocprog(compst->L, compst->p, size * 2);
  return compst->ncode++;
}


#define getinstr(cs,i)        ((cs)->p->code[i])


static int addinstruction (CompileState *compst, Opcode op, int aux) {
  int i = nextinstruction(compst);
  getinstr(compst, i).i.code = op;
  getinstr(compst, i).i.aux = aux;
  return i;
}


static int addoffsetinst (CompileState *compst, Opcode op) {
  int i = addinstruction(compst, op, 0);  /* 指令 */
  addinstruction(compst, (Opcode)0, 0);  /* 为偏移量留出空间 */
  assert(op == ITestSet || sizei(&getinstr(compst, i)) == 2);
  return i;
}


static void setoffset (CompileState *compst, int instruction, int offset) {
  getinstr(compst, instruction + 1).offset = offset;
}


/*
** 添加捕获指令：
** 'op' 是捕获指令；'cap' 是捕获类型；
** 'key' 是 ktable 中的键；'aux' 是可选的偏移量
**
*/
# 在指定位置插入一条指令，包括操作码、捕获、关键字和辅助信息
static int addinstcap (CompileState *compst, Opcode op, int cap, int key,
                       int aux) {
  int i = addinstruction(compst, op, joinkindoff(cap, aux));
  getinstr(compst, i).i.key = key;
  return i;
}

# 获取当前位置的指令索引
#define gethere(compst)     ((compst)->ncode)

# 获取指定位置的目标指令索引
#define target(code,i)        ((i) + code[i + 1].offset)

# 跳转到指定位置
static void jumptothere (CompileState *compst, int instruction, int target) {
  if (instruction >= 0)
    setoffset(compst, instruction, target - instruction);
}

# 跳转到当前位置
static void jumptohere (CompileState *compst, int instruction) {
  jumptothere(compst, instruction, gethere(compst));
}

# 生成 IChar 指令，如果存在等效的测试指令，则生成 IAny 指令
static void codechar (CompileState *compst, int c, int tt) {
  if (tt >= 0 and getinstr(compst, tt).i.code == ITestChar and
                 getinstr(compst, tt).i.aux == c)
    addinstruction(compst, IAny, 0);
  else
    addinstruction(compst, IChar, c);
}

# 为指令添加字符集后缀
static void addcharset (CompileState *compst, const byte *cs) {
  int p = gethere(compst);
  int i;
  for (i = 0; i < (int)CHARSETINSTSIZE - 1; i++)
    nextinstruction(compst);  # 为缓冲区预留空间
  # 使用字符集填充缓冲区
  loopset(j, getinstr(compst, p).buff[j] = cs[j]);
}

# 生成字符集指令，优化单元集合为 IChar，完整集合为 IAny，空集合为 IFail
static void codecharset (CompileState *compst, const byte *cs, int tt) {
  int c = 0;  # 避免警告
  Opcode op = charsettype(cs, &c);
  switch (op) {
    case IChar: codechar(compst, c, tt); break;
    case ISet: {  /* 如果是非平凡的集合？ */
      // 如果指令类型大于等于0，并且下一个指令是ITestSet，并且字符集相等
      if (tt >= 0 && getinstr(compst, tt).i.code == ITestSet &&
          cs_equal(cs, getinstr(compst, tt + 2).buff))
        // 添加一个IAny指令，参数为0
        addinstruction(compst, IAny, 0);
      else {
        // 否则添加一个ISet指令，参数为0
        addinstruction(compst, ISet, 0);
        // 添加字符集
        addcharset(compst, cs);
      }
      // 跳出switch语句
      break;
    }
    // 默认情况下，添加指令和参数
    default: addinstruction(compst, op, c); break;
  }
}

/*
** code a test set, optimizing unit sets for ITestChar, "complete"
** sets for ITestAny, and empty sets for IJmp (always fails).
** 'e' is true iff test should accept the empty string. (Test
** instructions in the current VM never accept the empty string.)
*/
static int codetestset (CompileState *compst, Charset *cs, int e) {
  if (e) return NOINST;  /* 如果 e 为真，表示不需要测试，直接返回 NOINST */
  else {
    int c = 0;  /* 初始化 c 为 0 */
    Opcode op = charsettype(cs->cs, &c);  /* 获取字符集的类型和字符 */
    switch (op) {
      case IFail: return addoffsetinst(compst, IJmp);  /* 如果字符集类型为失败，则添加跳转指令 */
      case IAny: return addoffsetinst(compst, ITestAny);  /* 如果字符集类型为任意字符，则添加测试任意字符指令 */
      case IChar: {
        int i = addoffsetinst(compst, ITestChar);  /* 如果字符集类型为字符，则添加测试字符指令 */
        getinstr(compst, i).i.aux = c;  /* 获取指令并设置辅助值为字符 c */
        return i;  /* 返回指令索引 */
      }
      case ISet: {
        int i = addoffsetinst(compst, ITestSet);  /* 如果字符集类型为集合，则添加测试集合指令 */
        addcharset(compst, cs->cs);  /* 添加字符集 */
        return i;  /* 返回指令索引 */
      }
      default: assert(0); return 0;  /* 默认情况下断言失败，返回 0 */
    }
  }
}


/*
** Find the final destination of a sequence of jumps
*/
static int finaltarget (Instruction *code, int i) {
  while (code[i].i.code == IJmp)  /* 当指令为跳转指令时 */
    i = target(code, i);  /* 获取跳转目标的索引 */
  return i;  /* 返回最终目标的索引 */
}


/*
** final label (after traversing any jumps)
*/
static int finallabel (Instruction *code, int i) {
  return finaltarget(code, target(code, i));  /* 返回最终目标的索引 */
}


/*
** <behind(p)> == behind n; <p>   (where n = fixedlen(p))
*/
static void codebehind (CompileState *compst, TTree *tree) {
  if (tree->u.n > 0)  /* 如果树的 u.n 大于 0 */
    addinstruction(compst, IBehind, tree->u.n);  /* 添加指令，表示在 n 个字符之后匹配树 */
  codegen(compst, sib1(tree), 0, NOINST, fullset);  /* 生成代码 */
}


/*
** Choice; optimizations:
** - when p1 is headfail
** - when first(p1) and first(p2) are disjoint; than
** a character not in first(p1) cannot go to p1, and a character
** in first(p1) cannot go to p2 (at it is not in first(p2)).
** (The optimization is not valid if p1 accepts the empty string,
** as then there is no character at all...)
** - when p2 is empty and opt is true; a IPartialCommit can resuse
** the Choice already active in the stack.
*/
# 定义函数 codechoice，接受编译状态、两个树节点、选项和字符集作为参数
static void codechoice (CompileState *compst, TTree *p1, TTree *p2, int opt,
                        const Charset *fl) {
  # 判断 p2 是否为空
  int emptyp2 = (p2->tag == TTrue);
  # 定义字符集 cs1 和 cs2，以及整数 e1
  Charset cs1, cs2;
  int e1 = getfirst(p1, fullset, &cs1);
  # 判断 p1 是否以 headfail 开头，或者 p1 是否为空且 p2 与 fl 的字符集不相交
  if (headfail(p1) ||
      (!e1 && (getfirst(p2, fl, &cs2), cs_disjoint(&cs1, &cs2)))) {
    # 如果条件成立，执行以下代码块
    /* <p1 / p2> == test (fail(p1)) -> L1 ; p1 ; jmp L2; L1: p2; L2: */
    # 定义整数 test 和 jmp
    int test = codetestset(compst, &cs1, 0);
    int jmp = NOINST;
    # 生成 p1 的代码
    codegen(compst, p1, 0, test, fl);
    # 如果 p2 不为空，添加跳转指令
    if (!emptyp2)
      jmp = addoffsetinst(compst, IJmp);
    # 跳转到 test 处
    jumptohere(compst, test);
    # 生成 p2 的代码
    codegen(compst, p2, opt, NOINST, fl);
    # 跳转到 jmp 处
    jumptohere(compst, jmp);
  }
  # 如果条件不成立，执行以下代码块
  else if (opt && emptyp2) {
    /* p1? == IPartialCommit; p1 */
    # 跳转到部分匹配处
    jumptohere(compst, addoffsetinst(compst, IPartialCommit));
    # 生成 p1 的代码
    codegen(compst, p1, 1, NOINST, fullset);
  }
  # 如果条件都不成立，执行以下代码块
  else {
    /* <p1 / p2> ==
        test(fail(p1)) -> L1; choice L1; <p1>; commit L2; L1: <p2>; L2: */
    # 定义整数 pcommit 和 test
    int pcommit;
    int test = codetestset(compst, &cs1, e1);
    # 添加选择指令
    int pchoice = addoffsetinst(compst, IChoice);
    # 生成 p1 的代码
    codegen(compst, p1, emptyp2, test, fullset);
    # 添加提交指令
    pcommit = addoffsetinst(compst, ICommit);
    # 跳转到 pchoice 处
    jumptohere(compst, pchoice);
    # 跳转到 test 处
    jumptohere(compst, test);
    # 生成 p2 的代码
    codegen(compst, p2, opt, NOINST, fl);
    # 跳转到 pcommit 处
    jumptohere(compst, pcommit);
  }
}


/*
** And predicate
** optimization: fixedlen(p) = n ==> <&p> == <p>; behind n
** (valid only when 'p' has no captures)
*/
# 定义函数 codeand，接受编译状态、树节点和整数作为参数
static void codeand (CompileState *compst, TTree *tree, int tt) {
  # 获取树节点的固定长度
  int n = fixedlen(tree);
  # 如果固定长度大于等于 0 且小于等于 MAXBEHIND，并且树节点没有捕获
  if (n >= 0 && n <= MAXBEHIND && !hascaptures(tree)) {
    # 生成树节点的代码
    codegen(compst, tree, 0, tt, fullset);
    # 如果固定长度大于 0，添加向后匹配指令
    if (n > 0)
      addinstruction(compst, IBehind, n);
  }
  # 如果条件不成立，执行以下代码块
  else {  /* default: Choice L1; p1; BackCommit L2; L1: Fail; L2: */
    # 定义整数 pcommit
    int pcommit;
    # 添加选择指令
    int pchoice = addoffsetinst(compst, IChoice);
    # 生成树节点的代码
    codegen(compst, tree, 0, tt, fullset);
    # 添加回溯提交指令
    pcommit = addoffsetinst(compst, IBackCommit);
    # 跳转到 pchoice 处
    jumptohere(compst, pchoice);
    # 添加失败指令
    addinstruction(compst, IFail, 0);
    # 跳转到 pcommit 处
    jumptohere(compst, pcommit);
  }
}
/*
** Captures: if pattern has fixed (and not too big) length, use
** a single IFullCapture instruction after the match; otherwise,
** enclose the pattern with OpenCapture - CloseCapture.
*/
static void codecapture (CompileState *compst, TTree *tree, int tt,
                         const Charset *fl) {
  // 获取子树的固定长度
  int len = fixedlen(sib1(tree));
  // 如果长度在有效范围内且没有捕获
  if (len >= 0 && len <= MAXOFF && !hascaptures(sib1(tree))) {
    // 生成代码
    codegen(compst, sib1(tree), 0, tt, fl);
    // 添加完整捕获指令
    addinstcap(compst, IFullCapture, tree->cap, tree->key, len);
  }
  else {
    // 添加打开捕获指令
    addinstcap(compst, IOpenCapture, tree->cap, tree->key, 0);
    // 生成代码
    codegen(compst, sib1(tree), 0, tt, fl);
    // 添加关闭捕获指令
    addinstcap(compst, ICloseCapture, Cclose, 0, 0);
  }
}

static void coderuntime (CompileState *compst, TTree *tree, int tt) {
  // 添加打开捕获指令
  addinstcap(compst, IOpenCapture, Cgroup, tree->key, 0);
  // 生成代码
  codegen(compst, sib1(tree), 0, tt, fullset);
  // 添加关闭运行时捕获指令
  addinstcap(compst, ICloseRunTime, Cclose, 0, 0);
}

/*
** Repetion; optimizations:
** When pattern is a charset, can use special instruction ISpan.
** When pattern is head fail, or if it starts with characters that
** are disjoint from what follows the repetions, a simple test
** is enough (a fail inside the repetition would backtrack to fail
** again in the following pattern, so there is no need for a choice).
** When 'opt' is true, the repetion can reuse the Choice already
** active in the stack.
*/
static void coderep (CompileState *compst, TTree *tree, int opt,
                     const Charset *fl) {
  Charset st;
  // 如果模式是字符集
  if (tocharset(tree, &st)) {
    // 添加指令
    addinstruction(compst, ISpan, 0);
    // 添加字符集
    addcharset(compst, st.cs);
  }
  else {
    int e1 = getfirst(tree, fullset, &st);
    // 如果是头部失败，或者如果它以与重复后面不同的字符开头
    if (headfail(tree) || (!e1 && cs_disjoint(&st, fl))) {
      /* L1: test (fail(p1)) -> L2; <p>; jmp L1; L2: */
      int jmp;
      // 添加测试集指令
      int test = codetestset(compst, &st, 0);
      // 生成代码
      codegen(compst, tree, opt, test, fullset);
      // 添加跳转指令
      jmp = addoffsetinst(compst, IJmp);
      jumptohere(compst, test);
      jumptothere(compst, jmp, test);
    }
  }
}
    else {
      /* 如果条件为真，则执行以下代码块 */
      /* test(fail(p1)) -> L2; choice L2; L1: <p>; partialcommit L1; L2: */
      /* or (if 'opt'): partialcommit L1; L1: <p>; partialcommit L1; */
      // 定义变量 commit 和 l2
      int commit, l2;
      // 调用 codetestset 函数，对表达式 e1 进行测试
      int test = codetestset(compst, &st, e1);
      // 定义变量 pchoice，并初始化为 NOINST
      int pchoice = NOINST;
      // 如果 opt 为真，则在当前位置添加 IPartialCommit 指令
      if (opt)
        jumptohere(compst, addoffsetinst(compst, IPartialCommit));
      // 否则，在当前位置添加 IChoice 指令，并将指令地址赋给 pchoice
      else
        pchoice = addoffsetinst(compst, IChoice);
      // 获取当前位置的指令地址，并赋给 l2
      l2 = gethere(compst);
      // 生成代码，对树进行编码
      codegen(compst, tree, 0, NOINST, fullset);
      // 在当前位置添加 IPartialCommit 指令，并将指令地址赋给 commit
      commit = addoffsetinst(compst, IPartialCommit);
      // 跳转到指定位置
      jumptothere(compst, commit, l2);
      // 在当前位置添加 pchoice 指令
      jumptohere(compst, pchoice);
      // 在当前位置添加 test 指令
      jumptohere(compst, test);
    }
  }
/*
** Not predicate; optimizations:
** In any case, if first test fails, 'not' succeeds, so it can jump to
** the end. If pattern is headfail, that is all (it cannot fail
** in other parts); this case includes 'not' of simple sets. Otherwise,
** use the default code (a choice plus a failtwice).
*/
static void codenot (CompileState *compst, TTree *tree) {
  Charset st;  // 定义字符集结构体变量 st
  int e = getfirst(tree, fullset, &st);  // 调用 getfirst 函数获取树的第一个字符集，存储在 st 中，并返回字符集的结束位置
  int test = codetestset(compst, &st, e);  // 调用 codetestset 函数生成测试字符集的指令，并返回指令的位置
  if (headfail(tree))  /* test (fail(p1)) -> L1; fail; L1:  */
    addinstruction(compst, IFail, 0);  // 如果树的头部匹配失败，则添加失败指令
  else {
    /* test(fail(p))-> L1; choice L1; <p>; failtwice; L1:  */
    int pchoice = addoffsetinst(compst, IChoice);  // 添加偏移指令，选择失败后的跳转位置
    codegen(compst, tree, 0, NOINST, fullset);  // 生成代码
    addinstruction(compst, IFailTwice, 0);  // 添加失败两次的指令
    jumptohere(compst, pchoice);  // 跳转到指定位置
  }
  jumptohere(compst, test);  // 跳转到指定位置
}


/*
** change open calls to calls, using list 'positions' to find
** correct offsets; also optimize tail calls
*/
static void correctcalls (CompileState *compst, int *positions,
                          int from, int to) {
  int i;
  Instruction *code = compst->p->code;  // 定义指令数组指针 code，指向编译状态的代码
  for (i = from; i < to; i += sizei(&code[i])) {  // 循环遍历指令数组
    if (code[i].i.code == IOpenCall) {  // 如果指令是打开调用
      int n = code[i].i.key;  /* rule number */  // 获取规则编号
      int rule = positions[n];  /* rule position */  // 获取规则位置
      assert(rule == from || code[rule - 1].i.code == IRet);  // 断言规则位置等于起始位置或者规则位置的前一个指令是返回指令
      if (code[finaltarget(code, i + 2)].i.code == IRet)  /* call; ret ? */  // 如果调用后是返回指令
        code[i].i.code = IJmp;  /* tail call */  // 将指令改为跳转指令，进行尾调用优化
      else
        code[i].i.code = ICall;  // 否则将指令改为调用指令
      jumptothere(compst, i, rule);  /* call jumps to respective rule */  // 调用跳转到相应的规则
    }
  }
  assert(i == to);  // 断言 i 等于结束位置
}


/*
** Code for a grammar:
** call L1; jmp L2; L1: rule 1; ret; rule 2; ret; ...; L2:
*/
# 为给定的语法树生成代码
static void codegrammar (CompileState *compst, TTree *grammar) {
  int positions[MAXRULES];  # 用于保存规则位置的数组
  int rulenumber = 0;  # 规则编号初始化为0
  TTree *rule;  # 规则节点
  int firstcall = addoffsetinst(compst, ICall);  # 添加调用初始规则的指令
  int jumptoend = addoffsetinst(compst, IJmp);  # 添加跳转到结尾的指令
  int start = gethere(compst);  # 获取初始规则的起始位置
  jumptohere(compst, firstcall);  # 跳转到初始规则的位置
  for (rule = sib1(grammar); rule->tag == TRule; rule = sib2(rule)) {
    positions[rulenumber++] = gethere(compst);  # 保存规则的位置
    codegen(compst, sib1(rule), 0, NOINST, fullset);  # 生成规则的代码
    addinstruction(compst, IRet, 0);  # 添加返回指令
  }
  assert(rule->tag == TTrue);  # 断言规则为真
  jumptohere(compst, jumptoend);  # 跳转到结尾的位置
  correctcalls(compst, positions, start, gethere(compst));  # 修正调用
}

# 为给定的调用生成代码
static void codecall (CompileState *compst, TTree *call) {
  int c = addoffsetinst(compst, IOpenCall);  # 添加打开调用的指令，稍后进行修正
  getinstr(compst, c).i.key = sib2(call)->cap;  # 获取规则编号并赋值给指令的关键字
  assert(sib2(call)->tag == TRule);  # 断言调用的标签为规则
}

# 为序列的第一个子节点生成代码
# （第二个子节点在原地调用以允许尾递归）
# 返回第二个子节点的'tt'
static int codeseq1 (CompileState *compst, TTree *p1, TTree *p2,
                     int tt, const Charset *fl) {
  if (needfollow(p1)) {
    Charset fl1;
    getfirst(p2, fl, &fl1);  # 获取p2的第一个字符集作为p1的跟随集
    codegen(compst, p1, 0, tt, &fl1);  # 生成p1的代码
  }
  else  # 使用'fullset'作为跟随集
    codegen(compst, p1, 0, tt, fullset);  # 生成p1的代码
  if (fixedlen(p1) != 0)  # 'p1'能否匹配任何内容？
    return  NOINST;  # 使测试无效
  else return tt;  # 否则'tt'仍然保护sib2
}

# 主代码生成函数：根据树的类型分派到辅助函数
static void codegen (CompileState *compst, TTree *tree, int opt, int tt,
                     const Charset *fl) {
 tailcall:  # 尾递归标签
  switch (tree->tag) {
    case TChar: codechar(compst, tree->u.n, tt); break;  # 生成字符匹配的代码
    case TAny: addinstruction(compst, IAny, 0); break;  # 添加匹配任意字符的指令
    # 如果节点类型是集合，则根据编译状态、树缓冲区和目标类型生成代码字符集
    case TSet: codecharset(compst, treebuffer(tree), tt); break;
    # 如果节点类型是真，则直接跳过
    case TTrue: break;
    # 如果节点类型是假，则添加一个失败指令
    case TFalse: addinstruction(compst, IFail, 0); break;
    # 如果节点类型是选择，则根据编译状态、树的两个子节点、选项和标志生成选择代码
    case TChoice: codechoice(compst, sib1(tree), sib2(tree), opt, fl); break;
    # 如果节点类型是重复，则根据编译状态、树的第一个子节点、选项和标志生成重复代码
    case TRep: coderep(compst, sib1(tree), opt, fl); break;
    # 如果节点类型是回溯，则根据编译状态和树生成回溯代码
    case TBehind: codebehind(compst, tree); break;
    # 如果节点类型是非，则根据编译状态和树的第一个子节点生成非代码
    case TNot: codenot(compst, sib1(tree)); break;
    # 如果节点类型是与，则根据编译状态、树的第一个子节点和目标类型生成与代码
    case TAnd: codeand(compst, sib1(tree), tt); break;
    # 如果节点类型是捕获，则根据编译状态、树、目标类型和标志生成捕获代码
    case TCapture: codecapture(compst, tree, tt, fl); break;
    # 如果节点类型是运行时，则根据编译状态、树和目标类型生成运行时代码
    case TRunTime: coderuntime(compst, tree, tt); break;
    # 如果节点类型是语法规则，则根据编译状态和树生成语法规则代码
    case TGrammar: codegrammar(compst, tree); break;
    # 如果节点类型是调用，则根据编译状态和树生成调用代码
    case TCall: codecall(compst, tree); break;
    # 如果节点类型是序列，则根据编译状态、树的两个子节点、目标类型和标志生成序列代码
    case TSeq: {
      # 生成第一个子节点的代码，并更新目标类型
      tt = codeseq1(compst, sib1(tree), sib2(tree), tt, fl);  /* code 'p1' */
      # 生成第二个子节点的代码（注释掉的代码）
      # codegen(compst, p2, opt, tt, fl);
      # 更新树为第二个子节点，并跳转到尾部调用
      tree = sib2(tree); goto tailcall;
    }
    # 默认情况下，断言失败
    default: assert(0);
  }
/*
** 优化跳转和其他类似跳转的指令。
** * 更新具有标签的指令的标签到其最终目的地（例如，choice L1; ... L1: jmp L2: 变成 choice L2）
** * 跳转到其他执行跳转的指令变成那些指令（例如，跳转到返回变成返回；跳转到提交变成提交）
*/
static void peephole (CompileState *compst) {
  Instruction *code = compst->p->code;
  int i;
  for (i = 0; i < compst->ncode; i += sizei(&code[i])) {
    switch (code[i].i.code) {
      case IChoice: case ICall: case ICommit: case IPartialCommit:
      case IBackCommit: case ITestChar: case ITestSet:
      case ITestAny: {  /* 具有标签的指令 */
        jumptothere(compst, i, finallabel(code, i));  /* 优化标签 */
        break;
      }
      case IJmp: {
        int ft = finaltarget(code, i);
        switch (code[ft].i.code) {  /* 跳转到哪里？ */
          case IRet: case IFail: case IFailTwice:
          case IEnd: {  /* 具有无条件隐式跳转的指令 */
            code[i] = code[ft];  /* 跳转变成那个指令 */
            code[i + 1].i.code = IAny;  /* 目标位置的“空操作” */
            break;
          }
          case ICommit: case IPartialCommit:
          case IBackCommit: {  /* 具有无条件显式跳转的指令 */
            int fft = finallabel(code, ft);
            code[i] = code[ft];  /* 跳转变成那个指令... */
            jumptothere(compst, i, fft);  /* 但必须修正其偏移量 */
            i--;  /* 重新优化其标签 */
            break;
          }
          default: {
            jumptothere(compst, i, ft);  /* 优化标签 */
            break;
          }
        }
        break;
      }
      default: break;
    }
  }
  assert(code[i - 1].i.code == IEnd);
}
Instruction *compile (lua_State *L, Pattern *p) {
  // 创建编译状态结构体
  CompileState compst;
  // 设置编译状态结构体的 p 字段为给定的 Pattern 指针
  compst.p = p;  compst.ncode = 0;  compst.L = L;
  // 重新分配 Pattern 对象的指令数组的内存空间，初始大小为 2
  reallocprog(L, p, 2);  /* minimum initial size */
  // 生成指令
  codegen(&compst, p->tree, 0, NOINST, fullset);
  // 添加结束指令
  addinstruction(&compst, IEnd, 0);
  // 重新分配 Pattern 对象的指令数组的内存空间，设置最终大小
  reallocprog(L, p, compst.ncode);  /* set final size */
  // 对指令进行优化
  peephole(&compst);
  // 返回 Pattern 对象的指令数组
  return p->code;
}


/* }====================================================== */

/*
** $Id: lpprint.c,v 1.7 2013/04/12 16:29:49 roberto Exp $
** Copyright 2007, Lua.org & PUC-Rio  (see 'lpeg.html' for license)
*/

#include <ctype.h>
#include <limits.h>
#include <stdio.h>




#if defined(LPEG_DEBUG)

/*
** {======================================================
** Printing patterns (for debugging)
** =======================================================
*/


void printcharset (const byte *st) {
  int i;
  printf("[");
  for (i = 0; i <= UCHAR_MAX; i++) {
    int first = i;
    while (testchar(st, i) && i <= UCHAR_MAX) i++;
    if (i - 1 == first)  /* unary range? */
      printf("(%02x)", first);
    else if (i - 1 > first)  /* non-empty range? */
      printf("(%02x-%02x)", first, i - 1);
  }
  printf("]");
}


static void printcapkind (int kind) {
  const char *const modes[] = {
    "close", "position", "constant", "backref",
    "argument", "simple", "table", "function",
    "query", "string", "num", "substitution", "fold",
    "runtime", "group"};
  printf("%s", modes[kind]);
}


static void printjmp (const Instruction *op, const Instruction *p) {
  printf("-> %d", (int)(p + (p + 1)->offset - op));
}


static void printinst (const Instruction *op, const Instruction *p) {
  const char *const names[] = {
    "any", "char", "set",
    "testany", "testchar", "testset",
    "span", "behind",
    "ret", "end",
    "choice", "jmp", "call", "open_call",
  # 定义一个包含操作码名称的字符串数组
  "commit", "partial_commit", "back_commit", "failtwice", "fail", "giveup",
  "fullcapture", "opencapture", "closecapture", "closeruntime"
};
# 打印当前操作码的位置和名称
printf("%02ld: %s ", (long)(p - op), names[p->i.code]);
# 根据操作码类型进行不同的处理
switch ((Opcode)p->i.code) {
  # 如果是匹配单个字符的操作码
  case IChar: {
    # 打印匹配的字符
    printf("'%c'", p->i.aux);
    break;
  }
  # 如果是测试匹配单个字符的操作码
  case ITestChar: {
    # 打印匹配的字符，并打印跳转位置
    printf("'%c'", p->i.aux); printjmp(op, p);
    break;
  }
  # 如果是完全捕获的操作码
  case IFullCapture: {
    # 打印捕获类型和大小，以及索引
    printcapkind(getkind(p));
    printf(" (size = %d)  (idx = %d)", getoff(p), p->i.key);
    break;
  }
  # 如果是开启捕获的操作码
  case IOpenCapture: {
    # 打印捕获类型和索引
    printcapkind(getkind(p));
    printf(" (idx = %d)", p->i.key);
    break;
  }
  # 如果是字符集匹配的操作码
  case ISet: {
    # 打印字符集
    printcharset((p+1)->buff);
    break;
  }
  # 如果是测试字符集匹配的操作码
  case ITestSet: {
    # 打印字符集，并打印跳转位置
    printcharset((p+2)->buff); printjmp(op, p);
    break;
  }
  # 如果是跨度匹配的操作码
  case ISpan: {
    # 打印字符集
    printcharset((p+1)->buff);
    break;
  }
  # 如果是开启调用的操作码
  case IOpenCall: {
    # 打印跳转位置
    printf("-> %d", (p + 1)->offset);
    break;
  }
  # 如果是回溯的操作码
  case IBehind: {
    # 打印回溯的偏移量
    printf("%d", p->i.aux);
    break;
  }
  # 如果是跳转、调用、提交、选择、部分提交、回溯、测试任意字符的操作码
  case IJmp: case ICall: case ICommit: case IChoice:
  case IPartialCommit: case IBackCommit: case ITestAny: {
    # 打印跳转位置
    printjmp(op, p);
    break;
  }
  # 如果是其他操作码，不做处理
  default: break;
}
# 打印换行符
printf("\n");
# 打印指令的模式
void printpatt (Instruction *p, int n) {
  # 保存指令的起始地址
  Instruction *op = p;
  # 遍历指令数组，打印每条指令
  while (p < op + n) {
    printinst(op, p);
    # 移动到下一条指令的地址
    p += sizei(p);
  }
}

# 如果定义了 LPEG_DEBUG，则打印捕获列表
#if defined(LPEG_DEBUG)
static void printcap (Capture *cap) {
  # 打印捕获类型
  printcapkind(cap->kind);
  # 打印捕获的索引和大小，以及地址
  printf(" (idx: %d - size: %d) -> %p\n", cap->idx, cap->siz, cap->s);
}

# 打印捕获列表
void printcaplist (Capture *cap, Capture *limit) {
  # 打印分隔符
  printf(">======\n");
  # 遍历捕获列表，打印每个捕获
  for (; cap->s && (limit == NULL || cap < limit); cap++)
    printcap(cap);
  # 打印结束符
  printf("=======\n");
}
#endif

/* }====================================================== */

# 打印树结构（用于调试）
/*
** {======================================================
** Printing trees (for debugging)
** =======================================================
*/

# 定义标签名称数组
static const char *tagnames[] = {
  "char", "set", "any",
  "true", "false",
  "rep",
  "seq", "choice",
  "not", "and",
  "call", "opencall", "rule", "grammar",
  "behind",
  "capture", "run-time"
};

# 打印树结构
void printtree (TTree *tree, int ident) {
  int i;
  # 打印缩进
  for (i = 0; i < ident; i++) printf(" ");
  # 打印节点类型
  printf("%s", tagnames[tree->tag]);
  # 根据节点类型打印不同的信息
  switch (tree->tag) {
    case TChar: {
      int c = tree->u.n;
      # 如果是可打印字符，则打印字符，否则打印十六进制
      if (isprint(c))
        printf(" '%c'\n", c);
      else
        printf(" (%02X)\n", c);
      break;
    }
    case TSet: {
      # 打印字符集
      printcharset(treebuffer(tree));
      printf("\n");
      break;
    }
    case TOpenCall: case TCall: {
      # 打印调用的键值
      printf(" key: %d\n", tree->key);
      break;
    }
    case TBehind: {
      # 打印偏移值
      printf(" %d\n", tree->u.n);
      # 递归打印子树
      printtree(sib1(tree), ident + 2);
      break;
    }
    case TCapture: {
      # 打印捕获信息
      printf(" cap: %d  key: %d  n: %d\n", tree->cap, tree->key, tree->u.n);
      # 递归打印子树
      printtree(sib1(tree), ident + 2);
      break;
    }
    case TRule: {
      # 打印规则信息
      printf(" n: %d  key: %d\n", tree->cap, tree->key);
      # 递归打印子树
      printtree(sib1(tree), ident + 2);
      break;  /* do not print next rule as a sibling */
    }
    # 如果当前节点是语法规则节点
    case TGrammar: {
      # 获取当前节点的第一个子节点，即规则节点
      TTree *rule = sib1(tree);
      # 打印规则的数量
      printf(" %d\n", tree->u.n);  /* number of rules */
      # 遍历每个规则节点并打印
      for (i = 0; i < tree->u.n; i++) {
        printtree(rule, ident + 2);
        # 获取下一个规则节点
        rule = sib2(rule);
      }
      # 确保最后一个节点是TTrue类型，作为结束标志
      assert(rule->tag == TTrue);  /* sentinel */
      break;
    }
    # 如果当前节点不是语法规则节点
    default: {
      # 获取当前节点的兄弟节点数量
      int sibs = numsiblings[tree->tag];
      # 打印换行
      printf("\n");
      # 如果有至少一个兄弟节点，则打印第一个兄弟节点
      if (sibs >= 1) {
        printtree(sib1(tree), ident + 2);
        # 如果有至少两个兄弟节点，则打印第二个兄弟节点
        if (sibs >= 2)
          printtree(sib2(tree), ident + 2);
      }
      break;
    }
  }
}
// 打印指定索引处的 Lua 环境中的 ktable
void printktable (lua_State *L, int idx) {
  int n, i;
  // 获取指定索引处的 Lua 环境
  lua_getfenv(L, idx);
  // 如果 ktable 不存在，则返回
  if (lua_isnil(L, -1))  /* no ktable? */
    return;
  // 获取 ktable 的长度
  n = lua_objlen(L, -1);
  // 打印左括号
  printf("[");
  // 遍历 ktable
  for (i = 1; i <= n; i++) {
    // 打印索引
    printf("%d = ", i);
    // 获取索引处的值
    lua_rawgeti(L, -1, i);
    // 如果值是字符串，则打印字符串，否则打印类型名
    if (lua_isstring(L, -1))
      printf("%s  ", lua_tostring(L, -1));
    else
      printf("%s  ", lua_typename(L, lua_type(L, -1)));
    // 弹出栈顶的值
    lua_pop(L, 1);
  }
  // 打印右括号
  printf("]\n");
  /* leave ktable at the stack */
}

/* }====================================================== */

#endif
/*
** $Id: lptree.c,v 1.10 2013/04/12 16:30:33 roberto Exp $
** Copyright 2013, Lua.org & PUC-Rio  (see 'lpeg.html' for license)
*/

#include <ctype.h>
#include <limits.h>
#include <string.h>

// 每个树的兄弟节点数
const byte numsiblings[] = {
  0, 0, 0,    /* char, set, any */
  0, 0,        /* true, false */
  1,        /* rep */
  2, 2,        /* seq, choice */
  1, 1,        /* not, and */
  0, 0, 2, 1,  /* call, opencall, rule, grammar */
  1,  /* behind */
  1, 1  /* capture, runtime capture */
};

// 创建新的语法树
static TTree *newgrammar (lua_State *L, int arg);

// 返回栈上索引为 'idx' 的值的合理名称
static const char *val2str (lua_State *L, int idx) {
  const char *k = lua_tostring(L, idx);
  if (k != NULL)
    return lua_pushfstring(L, "%s", k);
  else
    return lua_pushfstring(L, "(a %s)", luaL_typename(L, idx));
}

// 将 TOpenCall 修复为 TCall 节点
static void fixonecall (lua_State *L, int postable, TTree *g, TTree *t) {
  int n;
  // 获取规则名称
  lua_rawgeti(L, -1, t->key);  /* get rule's name */
  // 查询名称在位置表中的位置
  lua_gettable(L, postable);  /* query name in position table */
  // 获取（绝对）位置
  n = (int)lua_tonumber(L, -1);  /* get (absolute) position */
  // 移除位置
  lua_pop(L, 1);  /* remove position */
  // 如果没有位置，则获取规则名称
  if (n == 0) {  /* no position? */
    lua_rawgeti(L, -1, t->key);  /* get rule's name again */
    // 如果给定语法中未定义规则，则抛出 Lua 错误
    luaL_error(L, "rule '%s' undefined in given grammar", val2str(L, -1));
  }
  // 设置节点标签为 TCall
  t->tag = TCall;
  // 设置节点的位置相对于节点的偏移量
  t->u.ps = n - (t - g);  /* position relative to node */
  // 断言节点的下一个兄弟节点的标签为 TRule
  assert(sib2(t)->tag == TRule);
  // 将节点的下一个兄弟节点的关键字设置为与当前节点相同
  sib2(t)->key = t->key;
/*
** Transform left associative constructions into right
** associative ones, for sequence and choice; that is:
** (t11 + t12) + t2  =>  t11 + (t12 + t2)
** (t11 * t12) * t2  =>  t11 * (t12 * t2)
** (that is, Op (Op t11 t12) t2 => Op t11 (Op t12 t2))
*/
static void correctassociativity (TTree *tree) {
  TTree *t1 = sib1(tree);
  assert(tree->tag == TChoice || tree->tag == TSeq);
  while (t1->tag == tree->tag) {
    int n1size = tree->u.ps - 1;  /* t1 == Op t11 t12 */
    int n11size = t1->u.ps - 1;
    int n12size = n1size - n11size - 1;
    memmove(sib1(tree), sib1(t1), n11size * sizeof(TTree)); /* move t11 */
    tree->u.ps = n11size + 1;
    sib2(tree)->tag = tree->tag;
    sib2(tree)->u.ps = n12size + 1;
  }
}
*/

/*
** Make final adjustments in a tree. Fix open calls in tree 't',
** making them refer to their respective rules or raising appropriate
** errors (if not inside a grammar). Correct associativity of associative
** constructions (making them right associative). Assume that tree's
** ktable is at the top of the stack (for error messages).
*/
static void finalfix (lua_State *L, int postable, TTree *g, TTree *t) {
 tailcall:
  switch (t->tag) {
    case TGrammar:  /* subgrammars were already fixed */
      return;
    case TOpenCall: {
      if (g != NULL)  /* inside a grammar? */
        fixonecall(L, postable, g, t);
      else {  /* open call outside grammar */
        lua_rawgeti(L, -1, t->key);
        luaL_error(L, "rule '%s' used outside a grammar", val2str(L, -1));
      }
      break;
    }
    case TSeq: case TChoice:
      correctassociativity(t);
      break;
  }
  switch (numsiblings[t->tag]) {
    case 1: /* finalfix(L, postable, g, sib1(t)); */
      t = sib1(t); goto tailcall;
    case 2:
      finalfix(L, postable, g, sib1(t));
      t = sib2(t); goto tailcall;  /* finalfix(L, postable, g, sib2(t)); */
    default: assert(numsiblings[t->tag] == 0); break;
  }
}
*/

/*
** {======================================================
*/
/*
** 生成语法树
** =======================================================
*/

/*
** 在 5.2 版本中，可以使用 'luaL_testudata'...
*/
static int testpattern (lua_State *L, int idx) {
  // 检查值是否为一个用户数据
  if (lua_touserdata(L, idx)) {  /* value is a userdata? */
    // 如果有元表
    if (lua_getmetatable(L, idx)) {  /* does it have a metatable? */
      // 获取元表
      luaL_getmetatable(L, PATTERN_T);
      // 检查是否具有正确的元表
      if (lua_rawequal(L, -1, -2)) {  /* does it have the correct mt? */
        // 移除两个元表
        lua_pop(L, 2);  /* remove both metatables */
        return 1;
      }
    }
  }
  return 0;
}


static Pattern *getpattern (lua_State *L, int idx) {
  return (Pattern *)luaL_checkudata(L, idx, PATTERN_T);
}


static int getsize (lua_State *L, int idx) {
  return (lua_objlen(L, idx) - sizeof(Pattern)) / sizeof(TTree) + 1;
}


static TTree *gettree (lua_State *L, int idx, int *len) {
  // 获取模式
  Pattern *p = getpattern(L, idx);
  if (len)
    *len = getsize(L, idx);
  return p->tree;
}


/*
** 创建一个模式
*/
static TTree *newtree (lua_State *L, int len) {
  // 计算所需的内存大小
  size_t size = (len - 1) * sizeof(TTree) + sizeof(Pattern);
  // 分配内存
  Pattern *p = (Pattern *)lua_newuserdata(L, size);
  // 获取元表
  luaL_getmetatable(L, PATTERN_T);
  // 设置元表
  lua_setmetatable(L, -2);
  p->code = NULL;  p->codesize = 0;
  return p->tree;
}


static TTree *newleaf (lua_State *L, int tag) {
  // 创建一个叶子节点
  TTree *tree = newtree(L, 1);
  tree->tag = tag;
  return tree;
}


static TTree *newcharset (lua_State *L) {
  // 创建一个字符集节点
  TTree *tree = newtree(L, bytes2slots(CHARSETSIZE) + 1);
  tree->tag = TSet;
  loopset(i, treebuffer(tree)[i] = 0);
  return tree;
}


/*
** 向树中添加一个序列，其中第一个兄弟是 'sib'（大小为 'sibsize'）；返回第二个兄弟的位置
*/
static TTree *seqaux (TTree *tree, TTree *sib, int sibsize) {
  tree->tag = TSeq; tree->u.ps = sibsize + 1;
  memcpy(sib1(tree), sib, sibsize * sizeof(TTree));
  return sib2(tree);
}


/*
** 将元素 'idx' 添加到栈顶模式的 'ktable' 中；如有必要，创建新的 'ktable'。返回新元素的索引。
*/
static int addtoktable (lua_State *L, int idx) {
  // 如果索引为0或者对应的值为nil，则不需要插入实际数值
  if (idx == 0 || lua_isnil(L, idx))  
    return 0;
  else {
    int n;
    // 从模式中获取ktable
    lua_getfenv(L, -1);  
    // 获取ktable的长度
    n = lua_objlen(L, -1);
    // 如果ktable为空或不存在
    if (n == 0) {  
      // 移除ktable
      lua_pop(L, 1);  
      // 创建一个新的table
      lua_createtable(L, 1, 0);  
    }
    // 将要添加的元素压入栈中
    lua_pushvalue(L, idx);  
    // 将元素添加到ktable中
    lua_rawseti(L, -2, n + 1);
    // 将ktable设置为模式的ktable
    lua_setfenv(L, -2);  
    return n + 1;
  }
}


/*
** Build a sequence of 'n' nodes, each with tag 'tag' and 'u.n' got
** from the array 's' (or 0 if array is NULL). (TSeq is binary, so it
** must build a sequence of sequence of sequence...)
*/
static void fillseq (TTree *tree, int tag, int n, const char *s) {
  int i;
  // 初始化n-1个Seq标签的节点
  for (i = 0; i < n - 1; i++) {  
    // 设置节点的标签为TSeq，u.ps为2
    tree->tag = TSeq; tree->u.ps = 2;
    // 设置sib1节点的标签为tag，u.n为s[i]或0
    sib1(tree)->tag = tag;
    sib1(tree)->u.n = s ? (byte)s[i] : 0;
    // 移动到sib2节点
    tree = sib2(tree);
  }
  // 最后一个节点的标签为tag，u.n为s[i]或0
  tree->tag = tag;  
  tree->u.n = s ? (byte)s[i] : 0;
}


/*
** Numbers as patterns:
** 0 == true (always match); n == TAny repeated 'n' times;
** -n == not (TAny repeated 'n' times)
*/
static TTree *numtree (lua_State *L, int n) {
  // 如果n为0，则返回TTrue叶子节点
  if (n == 0)
    return newleaf(L, TTrue);
  else {
    TTree *tree, *nd;
    // 如果n大于0
    if (n > 0)
      // 创建2*n-1个节点
      tree = nd = newtree(L, 2 * n - 1);
    else {  
      // 负数：编码为!(-n)
      n = -n;
      // 创建2*n个节点
      tree = newtree(L, 2 * n);
      // 设置节点标签为TNot
      tree->tag = TNot;
      // nd指向sib1节点
      nd = sib1(tree);
    }
    // 填充nd节点，标签为TAny，重复n次，s为NULL
    fillseq(nd, TAny, n, NULL);  
    return tree;
  }
}


/*
** Convert value at index 'idx' to a pattern
*/
static TTree *getpatt (lua_State *L, int idx, int *len) {
  TTree *tree;
  switch (lua_type(L, idx)) {
    # 如果 Lua 值是字符串类型
    case LUA_TSTRING: {
      # 获取字符串和其长度
      size_t slen;
      const char *s = lua_tolstring(L, idx, &slen);  /* get string */
      # 如果字符串为空
      if (slen == 0)  /* empty? */
        # 创建一个始终匹配的叶子节点
        tree = newleaf(L, TTrue);  /* always match */
      else {
        # 创建一个新的树节点，长度为字符串长度的两倍减一
        tree = newtree(L, 2 * (slen - 1) + 1);
        # 填充树节点，内容为字符串中的字符序列
        fillseq(tree, TChar, slen, s);  /* sequence of 'slen' chars */
      }
      # 结束当前 case
      break;
    }
    # 如果 Lua 值是数字类型
    case LUA_TNUMBER: {
      # 获取整数值
      int n = lua_tointeger(L, idx);
      # 创建一个表示数字的树节点
      tree = numtree(L, n);
      # 结束当前 case
      break;
    }
    # 如果 Lua 值是布尔类型
    case LUA_TBOOLEAN: {
      # 根据布尔值创建一个表示真或假的叶子节点
      tree = (lua_toboolean(L, idx) ? newleaf(L, TTrue) : newleaf(L, TFalse));
      # 结束当前 case
      break;
    }
    # 如果 Lua 值是表类型
    case LUA_TTABLE: {
      # 创建一个新的语法树
      tree = newgrammar(L, idx);
      # 结束当前 case
      break;
    }
    # 如果 Lua 值是函数类型
    case LUA_TFUNCTION: {
      # 创建一个新的树节点，标记为运行时
      tree = newtree(L, 2);
      tree->tag = TRunTime;
      # 将函数添加到令牌表中，并将其作为树节点的键
      tree->key = addtoktable(L, idx);
      # 将树节点的第一个子节点标记为真
      sib1(tree)->tag = TTrue;
      # 结束当前 case
      break;
    }
    # 如果 Lua 值不是上述类型
    default: {
      # 递归调用 gettree 函数，处理其他类型的 Lua 值
      return gettree(L, idx, len);
    }
  }
  # 将新创建的树节点替换到原来 Lua 值的位置
  lua_replace(L, idx);  /* put new tree into 'idx' slot */
  # 如果传入了长度参数
  if (len)
    # 获取树节点的大小，并赋值给 len
    *len = getsize(L, idx);
  # 返回创建的树节点
  return tree;
/*
** 返回模式的ktable在'idx'处的元素数目。
** 在Lua 5.2中，默认模式的“环境”是nil，而不是一个表。将其视为一个空表。
** 在Lua 5.1中，假设环境没有数字索引（长度为0）。
*/
static int ktablelen (lua_State *L, int idx) {
  if (!lua_istable(L, idx)) return 0;  // 如果在'idx'处不是表，则返回0
  else return lua_objlen(L, idx);  // 否则返回表的长度
}


/*
** 将表'idx1'的内容连接到表'idx2'中。
** （假设两个索引都是负数。）
** 返回表'idx2'的原始长度
*/
static int concattable (lua_State *L, int idx1, int idx2) {
  int i;
  int n1 = ktablelen(L, idx1);  // 获取表'idx1'的长度
  int n2 = ktablelen(L, idx2);  // 获取表'idx2'的长度
  if (n1 == 0) return 0;  // 如果没有需要修正的内容，则返回0
  for (i = 1; i <= n1; i++) {
    lua_rawgeti(L, idx1, i);  // 获取表'idx1'中第i个元素
    lua_rawseti(L, idx2 - 1, n2 + i);  // 将其放入表'idx2'中，修正'idx2'
  }
  return n2;  // 返回表'idx2'的长度
}


/*
** 从p1和p2的ktable中创建一个新模式的ktable，放在栈顶。
*/
static int joinktables (lua_State *L, int p1, int p2) {
  int n1, n2;
  lua_getfenv(L, p1);  // 获取p1的ktable
  lua_getfenv(L, p2);  // 获取p2的ktable
  n1 = ktablelen(L, -2);  // 获取p1的ktable的长度
  n2 = ktablelen(L, -1);  // 获取p2的ktable的长度
  if (n1 == 0 && n2 == 0) {  // 两个表都为空吗？
    lua_pop(L, 2);  // 什么也不做；弹出表
    return 0;  // 无需修正
  }
  if (n2 == 0 || lua_equal(L, -2, -1)) {  // 第二个表为空或相等吗？
    lua_pop(L, 1);  // 弹出第二个表
    lua_setfenv(L, -2);  // 将第一个ktable设置为新模式的ktable
    return 0;  // 无需修正
  }
  if (n1 == 0) {  // 第一个表为空吗？
    lua_setfenv(L, -3);  // 将第二个表设置为新模式的ktable
    lua_pop(L, 1);  // 弹出第一个表
    return 0;  // 无需修正
  }
  else {
    lua_createtable(L, n1 + n2, 0);  // 为新模式创建ktable
    /* 栈顶: 新p; ktable p1; ktable p2; 新ktable */
    concattable(L, -3, -1);  // 从p1中放入新ktable
    concattable(L, -2, -1);  // 从p2中放入新ktable
    # 将栈顶的值作为新的环境表，替换掉栈顶的函数
    lua_setfenv(L, -4);  /* new ktable becomes p env */
    # 弹出栈顶的两个值
    lua_pop(L, 2);  /* pop other ktables */
    # 返回 n1 作为修正值，用于 p2 的索引
    return n1;  /* correction for indices from p2 */
  }
static void correctkeys (TTree *tree, int n) {
  if (n == 0) return;  /* 如果 n 为 0，则不需要修正，直接返回 */
 tailcall:
  switch (tree->tag) {
    case TOpenCall: case TCall: case TRunTime: case TRule: {  /* 根据不同的标签类型进行处理 */
      if (tree->key > 0)  /* 如果节点的 key 大于 0 */
        tree->key += n;  /* 则将 key 值加上 n */
      break;
    }
    case TCapture: {  /* 如果节点是捕获类型 */
      if (tree->key > 0 && tree->cap != Carg && tree->cap != Cnum)  /* 并且 key 大于 0 且不是 Carg 或 Cnum 类型 */
        tree->key += n;  /* 则将 key 值加上 n */
      break;
    }
    default: break;
  }
  switch (numsiblings[tree->tag]) {  /* 根据节点的标签类型进行处理 */
    case 1:  /* 如果只有一个子节点 */
      tree = sib1(tree); goto tailcall;  /* 则递归处理子节点 */
    case 2:
      correctkeys(sib1(tree), n);  /* 递归处理第一个子节点 */
      tree = sib2(tree); goto tailcall;  /* 递归处理第二个子节点 */
    default: assert(numsiblings[tree->tag] == 0); break;
  }
}


/*
** 将索引为 'idx' 的元素的 'ktable' 复制到新树上（位于栈顶）
*/
static void copyktable (lua_State *L, int idx) {
  lua_getfenv(L, idx);  /* 获取索引为 'idx' 的元素的环境表 */
  lua_setfenv(L, -2);  /* 设置栈顶元素的环境表为获取到的环境表 */
}


/*
** 将栈顶的树的 'ktable' 与栈索引 'idx' 处的规则的 'ktable' 合并，并修正相应的树
*/
static void mergektable (lua_State *L, int idx, TTree *rule) {
  int n;
  lua_getfenv(L, -1);  /* 获取栈顶元素的环境表 */
  lua_getfenv(L, idx);  /* 获取索引为 'idx' 的元素的环境表 */
  n = concattable(L, -1, -2);  /* 合并两个环境表，并返回合并后的元素个数 */
  lua_pop(L, 2);  /* 弹出两个环境表 */
  correctkeys(rule, n);  /* 修正规则的 key 值 */
}


/*
** 创建一个新的树，带有新的根节点和一个子节点
** 子节点必须在 Lua 栈上，索引为 1
*/
static TTree *newroot1sib (lua_State *L, int tag) {
  int s1;
  TTree *tree1 = getpatt(L, 1, &s1);  /* 获取栈索引为 1 的模式 */
  TTree *tree = newtree(L, 1 + s1);  /* 创建新的树 */
  tree->tag = tag;  /* 设置根节点的标签类型 */
  memcpy(sib1(tree), tree1, s1 * sizeof(TTree));  /* 复制子节点的内容 */
  copyktable(L, 1);  /* 复制环境表 */
  return tree;
}


/*
** 创建一个新的树，带有新的根节点和两个子节点
** 子节点必须在 Lua 栈上，第一个子节点的索引为 1
*/
static TTree *newroot2sib (lua_State *L, int tag) {
  // 获取第一个模式的指针和大小
  int s1, s2;
  TTree *tree1 = getpatt(L, 1, &s1);
  // 获取第二个模式的指针和大小
  TTree *tree2 = getpatt(L, 2, &s2);
  // 创建新的树
  TTree *tree = newtree(L, 1 + s1 + s2);  /* create new tree */
  // 设置标签
  tree->tag = tag;
  // 设置指针
  tree->u.ps =  1 + s1;
  // 复制第一个模式的内容到新树
  memcpy(sib1(tree), tree1, s1 * sizeof(TTree));
  // 复制第二个模式的内容到新树
  memcpy(sib2(tree), tree2, s2 * sizeof(TTree));
  // 修正键
  correctkeys(sib2(tree), joinktables(L, 1, 2));
  // 返回新树
  return tree;
}


static int lp_P (lua_State *L) {
  // 检查参数是否为任意类型
  luaL_checkany(L, 1);
  // 获取第一个模式的指针
  getpatt(L, 1, NULL);
  // 设置栈顶为1
  lua_settop(L, 1);
  // 返回1
  return 1;
}


/*
** sequence operator; optimizations:
** false x => false, x true => x, true x => x
** (cannot do x . false => false because x may have runtime captures)
*/
static int lp_seq (lua_State *L) {
  // 获取第一个模式的指针
  TTree *tree1 = getpatt(L, 1, NULL);
  // 获取第二个模式的指针
  TTree *tree2 = getpatt(L, 2, NULL);
  // 如果第一个模式的标签为假 或者 第二个模式的标签为真
  if (tree1->tag == TFalse || tree2->tag == TTrue)
    // 将第一个模式压入栈中
    lua_pushvalue(L, 1);  /* false . x == false, x . true = x */
  // 如果第一个模式的标签为真
  else if (tree1->tag == TTrue)
    // 将第二个模式压入栈中
    lua_pushvalue(L, 2);  /* true . x = x */
  else
    // 创建新的树
    newroot2sib(L, TSeq);
  // 返回1
  return 1;
}


/*
** choice operator; optimizations:
** charset / charset => charset
** true / x => true, x / false => x, false / x => x
** (x / true is not equivalent to true)
*/
static int lp_choice (lua_State *L) {
  Charset st1, st2;
  // 获取第一个模式的指针
  TTree *t1 = getpatt(L, 1, NULL);
  // 获取第二个模式的指针
  TTree *t2 = getpatt(L, 2, NULL);
  // 如果第一个模式可以转换为字符集 并且 第二个模式也可以转换为字符集
  if (tocharset(t1, &st1) && tocharset(t2, &st2)) {
    // 创建新的字符集树
    TTree *t = newcharset(L);
    // 循环设置字符集
    loopset(i, treebuffer(t)[i] = st1.cs[i] | st2.cs[i]);
  }
  // 如果第一个模式不会失败 或者 第二个模式的标签为假
  else if (nofail(t1) || t2->tag == TFalse)
    // 将第一个模式压入栈中
    lua_pushvalue(L, 1);  /* true / x => true, x / false => x */
  // 如果第一个模式的标签为假
  else if (t1->tag == TFalse)
    // 将第二个模式压入栈中
    lua_pushvalue(L, 2);  /* false / x => x */
  else
    // 创建新的树
    newroot2sib(L, TChoice);
  // 返回1
  return 1;
}


/*
** p^n
*/
static int lp_star (lua_State *L) {
  int size1;
  // 获取第二个参数
  int n = luaL_checkint(L, 2);
  // 获取第一个模式的指针和大小
  TTree *tree1 = gettree(L, 1, &size1);
  // 如果n大于等于0
  if (n >= 0) {  /* seq tree1 (seq tree1 ... (seq tree1 (rep tree1))) */
    // 创建新的树
    TTree *tree = newtree(L, (n + 1) * (size1 + 1));
    # 如果 tree1 可为空，则抛出 Lua 错误
    if (nullable(tree1))
      luaL_error(L, "loop body may accept empty string");
    # 循环执行 n 次
    while (n--)  /* repeat 'n' times */
      # 将 tree1 连接到 tree 上，重复 n 次
      tree = seqaux(tree, tree1, size1);
    # 将 tree 的标签设置为 TRep
    tree->tag = TRep;
    # 复制 size1 个 TTree 到 tree1 后面
    memcpy(sib1(tree), tree1, size1 * sizeof(TTree));
  }
  else {  /* choice (seq tree1 ... choice tree1 true ...) true */
    TTree *tree;
    # 取 n 的相反数
    n = -n;
    # 计算新建 tree 的大小
    tree = newtree(L, n * (size1 + 3) - 1);
    # 循环执行 n-1 次
    for (; n > 1; n--) {  /* repeat (n - 1) times */
      # 将 tree 的标签设置为 TChoice，设置 u.ps 值
      tree->tag = TChoice; tree->u.ps = n * (size1 + 3) - 2;
      # 将 sib2(tree) 的标签设置为 TTrue
      sib2(tree)->tag = TTrue;
      # 将 tree1 连接到 tree 上
      tree = sib1(tree);
      tree = seqaux(tree, tree1, size1);
    }
    # 将 tree 的标签设置为 TChoice，设置 u.ps 值
    tree->tag = TChoice; tree->u.ps = size1 + 1;
    # 将 sib2(tree) 的标签设置为 TTrue
    sib2(tree)->tag = TTrue;
    # 复制 size1 个 TTree 到 tree1 后面
    memcpy(sib1(tree), tree1, size1 * sizeof(TTree));
  }
  # 复制 ktable
  copyktable(L, 1);
  # 返回 1
  return 1;
/*
** #p == &p
** 创建一个新的 And 节点，并返回 1
*/
static int lp_and (lua_State *L) {
  newroot1sib(L, TAnd);
  return 1;
}


/*
** -p == !p
** 创建一个新的 Not 节点，并返回 1
*/
static int lp_not (lua_State *L) {
  newroot1sib(L, TNot);
  return 1;
}


/*
** [t1 - t2] == Seq (Not t2) t1
** 如果 t1 和 t2 是字符集，那么做它们的差集
** 否则，创建一个新的序列节点，返回 1
*/
static int lp_sub (lua_State *L) {
  Charset st1, st2;
  int s1, s2;
  TTree *t1 = getpatt(L, 1, &s1);
  TTree *t2 = getpatt(L, 2, &s2);
  if (tocharset(t1, &st1) && tocharset(t2, &st2)) {
    TTree *t = newcharset(L);
    loopset(i, treebuffer(t)[i] = st1.cs[i] & ~st2.cs[i]);
  }
  else {
    TTree *tree = newtree(L, 2 + s1 + s2);
    tree->tag = TSeq;  /* sequence of... */
    tree->u.ps =  2 + s2;
    sib1(tree)->tag = TNot;  /* ...not... */
    memcpy(sib1(sib1(tree)), t2, s2 * sizeof(TTree));  /* ...t2 */
    memcpy(sib2(tree), t1, s1 * sizeof(TTree));  /* ... and t1 */
    correctkeys(sib1(tree), joinktables(L, 1, 2));
  }
  return 1;
}


/*
** 创建一个新的字符集节点，并返回 1
*/
static int lp_set (lua_State *L) {
  size_t l;
  const char *s = luaL_checklstring(L, 1, &l);
  TTree *tree = newcharset(L);
  while (l--) {
    setchar(treebuffer(tree), (byte)(*s));
    s++;
  }
  return 1;
}


/*
** 创建一个新的字符集节点，表示一个范围，并返回 1
*/
static int lp_range (lua_State *L) {
  int arg;
  int top = lua_gettop(L);
  TTree *tree = newcharset(L);
  for (arg = 1; arg <= top; arg++) {
    int c;
    size_t l;
    const char *r = luaL_checklstring(L, arg, &l);
    luaL_argcheck(L, l == 2, arg, "range must have two characters");
    for (c = (byte)r[0]; c <= (byte)r[1]; c++)
      setchar(treebuffer(tree), c);
  }
  return 1;
}


/*
** Look-behind predicate
** 创建一个新的 Look-behind 节点，并返回 1
*/
static int lp_behind (lua_State *L) {
  TTree *tree;
  TTree *tree1 = getpatt(L, 1, NULL);
  int n = fixedlen(tree1);
  luaL_argcheck(L, !hascaptures(tree1), 1, "pattern have captures");
  luaL_argcheck(L, n > 0, 1, "pattern may not have fixed length");
  luaL_argcheck(L, n <= MAXBEHIND, 1, "pattern too long to look behind");
  tree = newroot1sib(L, TBehind);
  tree->u.n = n;
  return 1;
}
// 定义一个名为 lp_V 的静态函数，接受一个 Lua 状态机指针作为参数
static int lp_V (lua_State *L) {
  // 创建一个新的叶子节点，类型为 TOpenCall，并返回指向该节点的指针
  TTree *tree = newleaf(L, TOpenCall);
  // 检查参数 1 是否为非空非 nil 值，如果是则抛出错误信息
  luaL_argcheck(L, !lua_isnoneornil(L, 1), 1, "non-nil value expected");
  // 将参数 1 添加到关键字表中，并将其索引赋值给 tree 的 key 属性
  tree->key = addtoktable(L, 1);
  // 返回 1，表示函数执行成功
  return 1;
}


/*
** 创建一个非空捕获的树，包含一个主体和可选的关联 Lua 值（在堆栈中的索引 'labelidx' 处）
*/
static int capture_aux (lua_State *L, int cap, int labelidx) {
  // 创建一个新的根节点，类型为 TCapture，并返回指向该节点的指针
  TTree *tree = newroot1sib(L, TCapture);
  // 设置捕获类型
  tree->cap = cap;
  // 将索引为 labelidx 的值添加到关键字表中，并将其索引赋值给 tree 的 key 属性
  tree->key = addtoktable(L, labelidx);
  // 返回 1，表示函数执行成功
  return 1;
}


/*
** 用空捕获填充树，使用一个空的（TTrue）兄弟节点
*/
static TTree *auxemptycap (lua_State *L, TTree *tree, int cap, int idx) {
  // 将树的标签设置为 TCapture
  tree->tag = TCapture;
  // 设置捕获类型
  tree->cap = cap;
  // 将索引为 idx 的值添加到关键字表中，并将其索引赋值给 tree 的 key 属性
  tree->key = addtoktable(L, idx);
  // 设置树的第一个子节点的标签为 TTrue
  sib1(tree)->tag = TTrue;
  // 返回树
  return tree;
}


/*
** 创建一个空捕获的树
*/
static TTree *newemptycap (lua_State *L, int cap, int idx) {
  // 调用 auxemptycap 函数创建一个新的树，并返回该树
  return auxemptycap(L, newtree(L, 2), cap, idx);
}


/*
** 语法为 p / v 的捕获（函数捕获、查询捕获、字符串捕获或数字捕获）
*/
static int lp_divcapture (lua_State *L) {
  // 根据参数 2 的类型进行不同的捕获操作
  switch (lua_type(L, 2)) {
    // 如果参数 2 的类型为 LUA_TFUNCTION，则进行函数捕获
    case LUA_TFUNCTION: return capture_aux(L, Cfunction, 2);
    // 如果参数 2 的类型为 LUA_TTABLE，则进行查询捕获
    case LUA_TTABLE: return capture_aux(L, Cquery, 2);
    // 如果参数 2 的类型为 LUA_TSTRING，则进行字符串捕获
    case LUA_TSTRING: return capture_aux(L, Cstring, 2);
    // 如果参数 2 的类型为 LUA_TNUMBER，则进行数字捕获
    case LUA_TNUMBER: {
      // 将参数 2 转换为整数
      int n = lua_tointeger(L, 2);
      // 创建一个新的根节点，类型为 TCapture
      TTree *tree = newroot1sib(L, TCapture);
      // 检查参数 2 是否在有效范围内
      luaL_argcheck(L, 0 <= n && n <= SHRT_MAX, 1, "invalid number");
      // 设置捕获类型为 Cnum
      tree->cap = Cnum;
      // 将参数 2 赋值给 tree 的 key 属性
      tree->key = n;
      // 返回 1，表示函数执行成功
      return 1;
    }
    // 如果参数 2 的类型不是上述类型之一，则抛出错误信息
    default: return luaL_argerror(L, 2, "invalid replacement value");
  }
}


// 函数捕获替换
static int lp_substcapture (lua_State *L) {
  // 进行捕获替换操作
  return capture_aux(L, Csubst, 0);
}


// 表捕获
static int lp_tablecapture (lua_State *L) {
  // 进行表捕获操作
  return capture_aux(L, Ctable, 0);
}


// 分组捕获
static int lp_groupcapture (lua_State *L) {
  // 如果参数 2 为 nil 或者不存在，则进行分组捕获
  if (lua_isnoneornil(L, 2))
    return capture_aux(L, Cgroup, 0);
  // 否则，检查参数 2 是否为字符串，然后进行分组捕获
  else {
    luaL_checkstring(L, 2);
    return capture_aux(L, Cgroup, 2);
  }
}
static int lp_foldcapture (lua_State *L) {
  // 检查参数2的类型是否为函数，如果不是则抛出错误
  luaL_checktype(L, 2, LUA_TFUNCTION);
  // 调用辅助函数，传入L和Cfold，返回结果
  return capture_aux(L, Cfold, 2);
}


static int lp_simplecapture (lua_State *L) {
  // 调用辅助函数，传入L和Csimple，返回结果
  return capture_aux(L, Csimple, 0);
}


static int lp_poscapture (lua_State *L) {
  // 在栈上创建一个空的位置捕获，返回1
  newemptycap(L, Cposition, 0);
  return 1;
}


static int lp_argcapture (lua_State *L) {
  // 检查参数1的类型是否为整数，如果不是则抛出错误
  int n = luaL_checkint(L, 1);
  // 在栈上创建一个空的参数捕获，返回该捕获的指针
  TTree *tree = newemptycap(L, Carg, 0);
  // 设置捕获的键为n
  tree->key = n;
  // 检查n是否在有效范围内，如果不是则抛出错误
  luaL_argcheck(L, 0 < n && n <= SHRT_MAX, 1, "invalid argument index");
  return 1;
}


static int lp_backref (lua_State *L) {
  // 检查参数1的类型是否为字符串，如果不是则抛出错误
  luaL_checkstring(L, 1);
  // 在栈上创建一个空的反向引用捕获，返回1
  newemptycap(L, Cbackref, 1);
  return 1;
}


/*
** Constant capture
*/
static int lp_constcapture (lua_State *L) {
  int i;
  int n = lua_gettop(L);  /* number of values */
  // 如果参数个数为0，创建一个空的叶子节点，返回1
  if (n == 0)  
    newleaf(L, TTrue);  
  // 如果参数个数为1，创建一个空的常量捕获，返回1
  else if (n == 1)
    newemptycap(L, Cconst, 1);  
  // 如果参数个数大于1，创建一个包含所有值的组捕获
  else {  
    TTree *tree = newtree(L, 1 + 3 * (n - 1) + 2);
    tree->tag = TCapture;
    tree->cap = Cgroup;
    tree->key = 0;
    tree = sib1(tree);
    for (i = 1; i <= n - 1; i++) {
      tree->tag = TSeq;
      tree->u.ps = 3;  /* skip TCapture and its sibling */
      auxemptycap(L, sib1(tree), Cconst, i);
      tree = sib2(tree);
    }
    auxemptycap(L, tree, Cconst, i);
  }
  return 1;
}


static int lp_matchtime (lua_State *L) {
  TTree *tree;
  // 检查参数2的类型是否为函数，如果不是则抛出错误
  luaL_checktype(L, 2, LUA_TFUNCTION);
  // 在栈上创建一个新的运行时捕获，返回1
  tree = newroot1sib(L, TRunTime);
  tree->key = addtoktable(L, 2);
  return 1;
}

/* }====================================================== */


/*
** {======================================================
** Grammar - Tree generation
** =======================================================
*/

/*
** push on the stack the index and the pattern for the
** initial rule of grammar at index 'arg' in the stack;
** also add that index into position table.
*/
static void getfirstrule (lua_State *L, int arg, int postab) {
  lua_rawgeti(L, arg, 1);  /* access first element */  // 从参数 arg 中获取第一个元素
  if (lua_isstring(L, -1)) {  /* is it the name of initial rule? */  // 判断是否为初始规则的名称
    lua_pushvalue(L, -1);  /* duplicate it to use as key */  // 复制该值以便用作键
    lua_gettable(L, arg);  /* get associated rule */  // 获取关联的规则
  }
  else {
    lua_pushinteger(L, 1);  /* key for initial rule */  // 初始规则的键
    lua_insert(L, -2);  /* put it before rule */  // 将其放在规则之前
  }
  if (!testpattern(L, -1)) {  /* initial rule not a pattern? */  // 初始规则不是模式？
    if (lua_isnil(L, -1))
      luaL_error(L, "grammar has no initial rule");  // 抛出错误，语法没有初始规则
    else
      luaL_error(L, "initial rule '%s' is not a pattern", lua_tostring(L, -2));  // 抛出错误，初始规则不是模式
  }
  lua_pushvalue(L, -2);  /* push key */  // 压入键
  lua_pushinteger(L, 1);  /* push rule position (after TGrammar) */  // 压入规则位置（在 TGrammar 之后）
  lua_settable(L, postab);  /* insert pair at position table */  // 在位置表中插入键值对
}

/*
** traverse grammar at index 'arg', pushing all its keys and patterns
** into the stack. Create a new table (before all pairs key-pattern) to
** collect all keys and their associated positions in the final tree
** (the "position table").
** Return the number of rules and (in 'totalsize') the total size
** for the new tree.
*/
static int collectrules (lua_State *L, int arg, int *totalsize) {
  int n = 1;  /* to count number of rules */  // 计算规则数量
  int postab = lua_gettop(L) + 1;  /* index of position table */  // 位置表的索引
  int size;  /* accumulator for total size */  // 总大小的累加器
  lua_newtable(L);  /* create position table */  // 创建位置表
  getfirstrule(L, arg, postab);  // 获取第一个规则
  size = 2 + getsize(L, postab + 2);  /* TGrammar + TRule + rule */  // 计算总大小
  lua_pushnil(L);  /* prepare to traverse grammar table */  // 准备遍历语法表
  while (lua_next(L, arg) != 0) {
    if (lua_tonumber(L, -2) == 1 || lua_equal(L, -2, postab + 1)) {  /* initial rule? */  // 初始规则？
      lua_pop(L, 1);  /* remove value (keep key for lua_next) */  // 移除值（保留键以便进行 lua_next）
      continue;
    }
    if (!testpattern(L, -1))  /* value is not a pattern? */  // 值不是模式？
      luaL_error(L, "rule '%s' is not a pattern", val2str(L, -2));  // 抛出错误，规则不是模式
  }
}
    # 检查 Lua 栈的空间是否足够，如果不够则报错
    luaL_checkstack(L, LUA_MINSTACK, "grammar has too many rules");
    # 将栈顶元素复制一份压入栈中，用于插入到位置表中
    lua_pushvalue(L, -2);  /* push key (to insert into position table) */
    # 将整数 size 压入栈中
    lua_pushinteger(L, size);
    # 将栈顶两个元素插入到 postab 表中
    lua_settable(L, postab);
    # 更新 size 的值
    size += 1 + getsize(L, -1);  /* update size */
    # 再次将栈顶元素复制一份压入栈中，用于下一次 lua_next
    lua_pushvalue(L, -2);  /* push key (for next lua_next) */
    # n 自增
    n++;
  }
  # 更新 totalsize 的值，用于结束规则列表
  *totalsize = size + 1;  /* TTrue to finish list of rules */
  # 返回 n 的值
  return n;
}
# 构建语法树
static void buildgrammar (lua_State *L, TTree *grammar, int frule, int n) {
  int i;
  TTree *nd = sib1(grammar);  /* 辅助指针用于遍历树 */
  for (i = 0; i < n; i++) {  /* 将每个规则添加到新树中 */
    int ridx = frule + 2*i + 1;  /* 第i个规则的索引 */
    int rulesize;
    TTree *rn = gettree(L, ridx, &rulesize);
    nd->tag = TRule;
    nd->key = 0;
    nd->cap = i;  /* 规则编号 */
    nd->u.ps = rulesize + 1;  /* 指向下一个规则 */
    memcpy(sib1(nd), rn, rulesize * sizeof(TTree));  /* 复制规则 */
    mergektable(L, ridx, sib1(nd));  /* 将其ktable合并到新的ktable中 */
    nd = sib2(nd);  /* 移动到下一个规则 */
  }
  nd->tag = TTrue;  /* 完成规则列表 */
}


/*
** 检查树是否具有潜在的无限循环
*/
static int checkloops (TTree *tree) {
 tailcall:
  if (tree->tag == TRep && nullable(sib1(tree)))
    return 1;
  else if (tree->tag == TGrammar)
    return 0;  /* 子语法已经检查过 */
  else {
    switch (numsiblings[tree->tag]) {
      case 1:  /* return checkloops(sib1(tree)); */
        tree = sib1(tree); goto tailcall;
      case 2:
        if (checkloops(sib1(tree))) return 1;
        /* else return checkloops(sib2(tree)); */
        tree = sib2(tree); goto tailcall;
      default: assert(numsiblings[tree->tag] == 0); return 0;
    }
  }
}


static int verifyerror (lua_State *L, int *passed, int npassed) {
  int i, j;
  for (i = npassed - 1; i >= 0; i--) {  /* 搜索重复项 */
    for (j = i - 1; j >= 0; j--) {
      if (passed[i] == passed[j]) {
        lua_rawgeti(L, -1, passed[i]);  /* 获取规则的键 */
        return luaL_error(L, "rule '%s' may be left recursive", val2str(L, -1));
      }
    }
  }
  return luaL_error(L, "too many left calls in grammar");
}


/*
** 检查规则是否可以左递归；在这种情况下引发错误；否则返回1（如果模式可为空）。假设堆栈顶部是ktable。
*/
static int verifyrule (lua_State *L, TTree *tree, int *passed, int npassed,
                       int nullable) {
 tailcall:
  switch (tree->tag) {
    case TChar: case TSet: case TAny:
    case TFalse:
      return nullable;  /* 从这里无法通过 */
    case TTrue:
    case TBehind:  /* 回顾不能有调用 */
      return 1;
    case TNot: case TAnd: case TRep:
      /* return verifyrule(L, sib1(tree), passed, npassed, 1); */
      tree = sib1(tree); nullable = 1; goto tailcall;
    case TCapture: case TRunTime:
      /* return verifyrule(L, sib1(tree), passed, npassed); */
      tree = sib1(tree); goto tailcall;
    case TCall:
      /* return verifyrule(L, sib2(tree), passed, npassed); */
      tree = sib2(tree); goto tailcall;
    case TSeq:  /* 只有在第一个可为空的情况下才检查第二个子节点 */
      if (!verifyrule(L, sib1(tree), passed, npassed, 0))
        return nullable;
      /* else return verifyrule(L, sib2(tree), passed, npassed); */
      tree = sib2(tree); goto tailcall;
    case TChoice:  /* 必须检查两个子节点 */
      nullable = verifyrule(L, sib1(tree), passed, npassed, nullable);
      /* return verifyrule(L, sib2(tree), passed, npassed, nullable); */
      tree = sib2(tree); goto tailcall;
    case TRule:
      if (npassed >= MAXRULES)
        return verifyerror(L, passed, npassed);
      else {
        passed[npassed++] = tree->key;
        /* return verifyrule(L, sib1(tree), passed, npassed); */
        tree = sib1(tree); goto tailcall;
      }
    case TGrammar:
      return nullable(tree);  /* 子语法不能是左递归的 */
    default: assert(0); return 0;
  }
}


static void verifygrammar (lua_State *L, TTree *grammar) {
  int passed[MAXRULES];
  TTree *rule;
  /* 检查左递归规则 */
  for (rule = sib1(grammar); rule->tag == TRule; rule = sib2(rule)) {
    if (rule->key == 0) continue;  /* 未使用的规则 */
    # 验证规则，检查是否通过，传入参数为 L, sib1(rule), passed, 0, 0
    verifyrule(L, sib1(rule), passed, 0, 0);
  }
  # 断言规则的标签为真
  assert(rule->tag == TTrue);
  # 检查规则内部的无限循环
  for (rule = sib1(grammar); rule->tag == TRule; rule = sib2(rule)) {
    # 如果规则的键为0，则跳过，表示未使用的规则
    if (rule->key == 0) continue;  
    # 如果检测到循环，则抛出 Lua 错误
    if (checkloops(sib1(rule))) {
      lua_rawgeti(L, -1, rule->key);  # 获取规则的键
      luaL_error(L, "empty loop in rule '%s'", val2str(L, -1));
    }
  }
  # 断言规则的标签为真
  assert(rule->tag == TTrue);
/*
** 如果初始规则没有被引用，则为其命名
*/
static void initialrulename (lua_State *L, TTree *grammar, int frule) {
  if (sib1(grammar)->key == 0) {  /* 初始规则没有被引用？ */
    int n = lua_objlen(L, -1) + 1;  /* 名称的索引 */
    lua_pushvalue(L, frule);  /* 规则的名称 */
    lua_rawseti(L, -2, n);  /* ktable 在栈顶 */
    sib1(grammar)->key = n;
  }
}


static TTree *newgrammar (lua_State *L, int arg) {
  int treesize;
  int frule = lua_gettop(L) + 2;  /* 第一个规则的键的位置 */
  int n = collectrules(L, arg, &treesize);
  TTree *g = newtree(L, treesize);
  luaL_argcheck(L, n <= MAXRULES, arg, "grammar has too many rules");
  g->tag = TGrammar;  g->u.n = n;
  lua_newtable(L);  /* 创建 'ktable' */
  lua_setfenv(L, -2);
  buildgrammar(L, g, frule, n);
  lua_getfenv(L, -1);  /* 获取新树的 'ktable' */
  finalfix(L, frule - 1, g, sib1(g));
  initialrulename(L, g, frule);
  verifygrammar(L, g);
  lua_pop(L, 1);  /* 移除 'ktable' */
  lua_insert(L, -(n * 2 + 2));  /* 将新表移动到正确的位置 */
  lua_pop(L, n * 2 + 1);  /* 移除位置表 + 规则对 */
  return g;  /* 新表在栈顶 */
}

/* }====================================================== */


static Instruction *prepcompile (lua_State *L, Pattern *p, int idx) {
  lua_getfenv(L, idx);  /* 推入 'ktable'（可能被 'finalfix' 使用） */
  finalfix(L, 0, NULL, p->tree);
  lua_pop(L, 1);  /* 移除 'ktable' */
  return compile(L, p);
}


static int lp_printtree (lua_State *L) {
  TTree *tree = getpatt(L, 1, NULL);
  int c = lua_toboolean(L, 2);
  if (c) {
    lua_getfenv(L, 1);  /* 推入 'ktable'（可能被 'finalfix' 使用） */
    finalfix(L, 0, NULL, tree);
    lua_pop(L, 1);  /* 移除 'ktable' */
  }
  printktable(L, 1);
  printtree(tree, 0);
  return 0;
}


static int lp_printcode (lua_State *L) {
  Pattern *p = getpattern(L, 1);
  printktable(L, 1);
  if (p->code == NULL)  /* 还未编译？ */
    # 对模式进行预编译，生成匹配模式的代码
    prepcompile(L, p, 1);
    # 打印模式的代码和代码大小
    printpatt(p->code, p->codesize);
    # 返回0，表示执行成功
    return 0;
}


/*
** Get the initial position for the match, interpreting negative
** values from the end of the subject
*/
static size_t initposition (lua_State *L, size_t len) {
  // 从第三个参数获取初始位置，如果没有则默认为1
  lua_Integer ii = luaL_optinteger(L, 3, 1);
  if (ii > 0) {  /* positive index? */
    if ((size_t)ii <= len)  /* inside the string? */
      return (size_t)ii - 1;  /* return it (corrected to 0-base) */
    else return len;  /* crop at the end */
  }
  else {  /* negative index */
    if ((size_t)(-ii) <= len)  /* inside the string? */
      return len - ((size_t)(-ii));  /* return position from the end */
    else return 0;  /* crop at the beginning */
  }
}


/*
** Main match function
*/
static int lp_match (lua_State *L) {
  // 初始化捕获数组
  Capture capture[INITCAPSIZE];
  const char *r;
  size_t l;
  // 获取模式
  Pattern *p = (getpatt(L, 1, NULL), getpattern(L, 1));
  // 获取指令集
  Instruction *code = (p->code != NULL) ? p->code : prepcompile(L, p, 1);
  // 获取主题字符串
  const char *s = luaL_checklstring(L, SUBJIDX, &l);
  // 获取初始位置
  size_t i = initposition(L, l);
  int ptop = lua_gettop(L);
  lua_pushnil(L);  /* initialize subscache */
  lua_pushlightuserdata(L, capture);  /* initialize caplistidx */
  lua_getfenv(L, 1);  /* initialize penvidx */
  // 进行匹配
  r = match(L, s, s + i, s + l, code, capture, ptop);
  if (r == NULL) {
    lua_pushnil(L);
    return 1;
  }
  // 获取捕获结果
  return getcaptures(L, s, r, ptop);
}



/*
** {======================================================
** Library creation and functions not related to matching
** =======================================================
*/

// 设置最大栈空间
static int lp_setmax (lua_State *L) {
  luaL_optinteger(L, 1, -1);
  lua_settop(L, 1);
  lua_setfield(L, LUA_REGISTRYINDEX, MAXSTACKIDX);
  return 0;
}


// 获取版本号
static int lp_version (lua_State *L) {
  lua_pushstring(L, VERSION);
  return 1;
}


// 获取模式类型
static int lp_type (lua_State *L) {
  if (testpattern(L, 1))
    lua_pushliteral(L, "pattern");
  else
    lua_pushnil(L);
  return 1;
}


// 垃圾回收函数
int lp_gc (lua_State *L) {
  Pattern *p = getpattern(L, 1);
  if (p->codesize > 0)
    # 调用reallocprog函数，将L、p和0作为参数传递
    reallocprog(L, p, 0);
    # 返回0
    return 0;
# 创建一个名为 createcat 的静态函数，接受 Lua 状态机指针、分类名称和分类函数作为参数
static void createcat (lua_State *L, const char *catname, int (catf) (int)) {
  # 创建一个新的字符集
  TTree *t = newcharset(L);
  int i;
  # 遍历所有可能的字符
  for (i = 0; i <= UCHAR_MAX; i++)
    # 如果字符符合分类函数的条件，则将其添加到字符集中
    if (catf(i)) setchar(treebuffer(t), i);
  # 将字符集添加到 Lua 栈顶的表中，表的索引为 -2，键为 catname
  lua_setfield(L, -2, catname);
}

# 创建一个名为 lp_locale 的静态函数，接受 Lua 状态机指针作为参数
static int lp_locale (lua_State *L) {
  # 如果传入的参数为空或为 nil，则将栈清空并创建一个新的空表
  if (lua_isnoneornil(L, 1)) {
    lua_settop(L, 0);
    lua_createtable(L, 0, 12);
  }
  else {
    # 否则，检查参数类型是否为表，将栈顶设置为参数
    luaL_checktype(L, 1, LUA_TTABLE);
    lua_settop(L, 1);
  }
  # 分别创建各种字符分类，并将其添加到表中
  createcat(L, "alnum", isalnum);
  createcat(L, "alpha", isalpha);
  createcat(L, "cntrl", iscntrl);
  createcat(L, "digit", isdigit);
  createcat(L, "graph", isgraph);
  createcat(L, "lower", islower);
  createcat(L, "print", isprint);
  createcat(L, "punct", ispunct);
  createcat(L, "space", isspace);
  createcat(L, "upper", isupper);
  createcat(L, "xdigit", isxdigit);
  # 返回 1，表示成功
  return 1;
}

# 定义一个名为 pattreg 的结构体数组，包含了各种模式和对应的处理函数
static struct luaL_Reg pattreg[] = {
  {"ptree", lp_printtree},
  {"pcode", lp_printcode},
  {"match", lp_match},
  {"B", lp_behind},
  {"V", lp_V},
  {"C", lp_simplecapture},
  {"Cc", lp_constcapture},
  {"Cmt", lp_matchtime},
  {"Cb", lp_backref},
  {"Carg", lp_argcapture},
  {"Cp", lp_poscapture},
  {"Cs", lp_substcapture},
  {"Ct", lp_tablecapture},
  {"Cf", lp_foldcapture},
  {"Cg", lp_groupcapture},
  {"P", lp_P},
  {"S", lp_set},
  {"R", lp_range},
  {"locale", lp_locale},
  {"version", lp_version},
  {"setmaxstack", lp_setmax},
  {"type", lp_type},
  {NULL, NULL}
};

# 定义一个名为 metareg 的结构体数组，包含了各种元方法和对应的处理函数
static struct luaL_Reg metareg[] = {
  {"__mul", lp_seq},
  {"__add", lp_choice},
  {"__pow", lp_star},
  {"__gc", lp_gc},
  {"__len", lp_and},
  {"__div", lp_divcapture},
  {"__unm", lp_not},
  {"__sub", lp_sub},
  {NULL, NULL}
};

# 定义一个名为 luaopen_lpeg 的函数，接受 Lua 状态机指针作为参数，返回一个整型值
LUALIB_API int luaopen_lpeg (lua_State *L);
LUALIB_API int luaopen_lpeg (lua_State *L) {
  // 创建一个新的元表，用于存储模式对象
  luaL_newmetatable(L, PATTERN_T);
  // 将最大回溯值初始化为一个数字，并存储在全局注册表中
  lua_pushnumber(L, MAXBACK);  /* initialize maximum backtracking */
  lua_setfield(L, LUA_REGISTRYINDEX, MAXSTACKIDX);
  // 注册元方法和函数到 Lua 环境中
  luaL_register(L, NULL, metareg);
  luaL_register(L, "lpeg", pattreg);
  // 将元表设置为自身的索引
  lua_pushvalue(L, -1);
  lua_setfield(L, -3, "__index");
  // 返回 1 表示成功
  return 1;
}

/* }====================================================== */
/*
** $Id: lpvm.c,v 1.5 2013/04/12 16:29:49 roberto Exp $
** Copyright 2007, Lua.org & PUC-Rio  (see 'lpeg.html' for license)
*/

#include <limits.h>
#include <string.h>

// 初始化调用/回溯栈的初始大小
#if !defined(INITBACK)
#define INITBACK    100
#endif

// 获取指令的偏移量
#define getoffset(p)    (((p) + 1)->offset)

// 定义放弃指令
static const Instruction giveup = {{IGiveup, 0, 0}};

/*
** {======================================================
** Virtual Machine
** =======================================================
*/

// 定义栈结构
typedef struct Stack {
  const char *s;  /* saved position (or NULL for calls) */
  const Instruction *p;  /* next instruction */
  int caplevel;
} Stack;

// 获取栈的基地址
#define getstackbase(L, ptop)    ((Stack *)lua_touserdata(L, stackidx(ptop)))

// 扩大捕获数组的大小
static Capture *doublecap (lua_State *L, Capture *cap, int captop, int ptop) {
  Capture *newc;
  if (captop >= INT_MAX/((int)sizeof(Capture) * 2))
    luaL_error(L, "too many captures");
  newc = (Capture *)lua_newuserdata(L, captop * 2 * sizeof(Capture));
  memcpy(newc, cap, captop * sizeof(Capture));
  lua_replace(L, caplistidx(ptop));
  return newc;
}

// 扩大栈的大小
static Stack *doublestack (lua_State *L, Stack **stacklimit, int ptop) {
  Stack *stack = getstackbase(L, ptop);
  Stack *newstack;
  int n = *stacklimit - stack;  /* current stack size */
  int max, newn;
  lua_getfield(L, LUA_REGISTRYINDEX, MAXSTACKIDX);
  max = lua_tointeger(L, -1);  /* maximum allowed size */
  lua_pop(L, 1);
  if (n >= max)  /* already at maximum size? */
    # 如果有太多的待处理调用/选择，则报错
    luaL_error(L, "too many pending calls/choices");
  # 计算新的大小
  newn = 2 * n;  /* new size */
  # 如果新的大小超过最大限制，则将新的大小设为最大限制
  if (newn > max) newn = max;
  # 使用lua_newuserdata在Lua栈上创建一个新的用户数据，大小为newn * sizeof(Stack)，并返回其指针
  newstack = (Stack *)lua_newuserdata(L, newn * sizeof(Stack));
  # 将stack指向的n个Stack结构体的数据拷贝到newstack指向的内存中
  memcpy(newstack, stack, n * sizeof(Stack));
  # 将栈顶的元素替换为指定索引处的值
  lua_replace(L, stackidx(ptop));
  # 设置stacklimit指向newstack + newn
  *stacklimit = newstack + newn;
  # 返回下一个位置的指针
  return newstack + n;  /* return next position */
/*
** 解释动态捕获的结果：false -> 失败；
** true -> 保持当前位置；number -> 下一个位置。
** 返回新的主题位置。'fr' 是结果的堆栈索引；
** 'curr' 是当前主题位置；'limit' 是主题的大小。
*/
static int resdyncaptures (lua_State *L, int fr, int curr, int limit) {
  lua_Integer res;
  if (!lua_toboolean(L, fr)) {  /* false value? */
    lua_settop(L, fr - 1);  /* 移除结果 */
    return -1;  /* 并失败 */
  }
  else if (lua_isboolean(L, fr))  /* true? */
    res = curr;  /* 保持当前位置 */
  else {
    res = lua_tointeger(L, fr) - 1;  /* 新位置 */
    if (res < curr || res > limit)
      luaL_error(L, "invalid position returned by match-time capture");
  }
  lua_remove(L, fr);  /* 移除第一个结果（偏移量） */
  return res;
}


/*
** 将动态捕获返回的捕获值添加到捕获列表'base'中，嵌套在一个组捕获内。'fd' 索引第一个捕获值，'n' 是值的数量（至少为1）。
*/
static void adddyncaptures (const char *s, Capture *base, int n, int fd) {
  int i;
  /* Cgroup 捕获已经存在 */
  assert(base[0].kind == Cgroup && base[0].siz == 0);
  base[0].idx = 0;  /* 使其成为匿名组 */
  for (i = 1; i <= n; i++) {  /* 添加运行时捕获 */
    base[i].kind = Cruntime;
    base[i].siz = 1;  /* 标记为关闭 */
    base[i].idx = fd + i - 1;  /* 捕获值的堆栈索引 */
    base[i].s = s;
  }
  base[i].kind = Cclose;  /* 关闭组 */
  base[i].siz = 1;
  base[i].s = s;
}


/*
** 从 Lua 栈中移除动态捕获（在失败的情况下调用）
*/
static int removedyncap (lua_State *L, Capture *capture,
                         int level, int last) {
  // 查找动态捕获的索引
  int id = finddyncap(capture + level, capture + last);  /* index of 1st cap. */
  // 获取 Lua 栈的栈顶
  int top = lua_gettop(L);
  // 如果没有动态捕获，返回 0
  if (id == 0) return 0;  /* no dynamic captures? */
  // 移除捕获
  lua_settop(L, id - 1);  /* remove captures */
  // 返回移除的值的数量
  return top - id + 1;  /* number of values removed */
}


/*
** Opcode interpreter
*/
const char *match (lua_State *L, const char *o, const char *s, const char *e,
                   Instruction *op, Capture *capture, int ptop) {
  // 初始化堆栈
  Stack stackbase[INITBACK];
  Stack *stacklimit = stackbase + INITBACK;
  Stack *stack = stackbase;  /* point to first empty slot in stack */
  // 初始化捕获大小和索引
  int capsize = INITCAPSIZE;
  int captop = 0;  /* point to first empty slot in captures */
  int ndyncap = 0;  /* number of dynamic captures (in Lua stack) */
  const Instruction *p = op;  /* current instruction */
  // 初始化堆栈
  stack->p = &giveup; stack->s = s; stack->caplevel = 0; stack++;
  // 将堆栈的地址作为轻量级用户数据压入 Lua 栈
  lua_pushlightuserdata(L, stackbase);
  for (;;) {
#if defined(DEBUG)
      // 打印调试信息
      printf("s: |%s| stck:%d, dyncaps:%d, caps:%d  ",
             s, stack - getstackbase(L, ptop), ndyncap, captop);
      printinst(op, p);
      printcaplist(capture, capture + captop);
#endif
    // 断言 Lua 栈的索引和动态捕获的数量
    assert(stackidx(ptop) + ndyncap == lua_gettop(L) && ndyncap <= captop);
    }
  }
}

/* }====================================================== */
```