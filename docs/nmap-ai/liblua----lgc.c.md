# `nmap\liblua\lgc.c`

```cpp
/*
** $Id: lgc.c $
** 垃圾回收器
** 请参见 lua.h 中的版权声明
*/

#define lgc_c
#define LUA_CORE

#include "lprefix.h"

#include <stdio.h>
#include <string.h>


#include "lua.h"

#include "ldebug.h"
#include "ldo.h"
#include "lfunc.h"
#include "lgc.h"
#include "lmem.h"
#include "lobject.h"
#include "lstate.h"
#include "lstring.h"
#include "ltable.h"
#include "ltm.h"


/*
** 每个单步骤中要扫描的最大元素数量。
** （足够大以消散固定开销，但足够小以允许垃圾回收器的小步骤。）
*/
#define GCSWEEPMAX    100

/*
** 每个单步骤中要调用的最大终结器数量。
*/
#define GCFINMAX    10


/*
** 调用一个终结器的成本。
*/
#define GCFINALIZECOST    50


/*
** 一个“工作”单位的等效字节数（访问一个槽，扫描一个对象等）。
*/
#define WORK2MEM    sizeof(TValue)


/*
** 调整 'pause' 的宏：'pause' 实际上被使用为 'pause / PAUSEADJ'（由测试选择的值）
*/
#define PAUSEADJ        100


/* 所有颜色位的掩码 */
#define maskcolors    (bitmask(BLACKBIT) | WHITEBITS)

/* 所有 GC 位的掩码 */
#define maskgcbits      (maskcolors | AGEBITS)


/* 擦除所有颜色位，然后只设置当前的白色位的宏 */
#define makewhite(g,x)    \
  (x->marked = cast_byte((x->marked & ~maskcolors) | luaC_white(g)))

/* 使对象变灰色（既不是白色也不是黑色） */
#define set2gray(x)    resetbits(x->marked, maskcolors)


/* 使对象变黑色（从任何颜色变为黑色） */
#define set2black(x)  \
  (x->marked = cast_byte((x->marked & ~WHITEBITS) | bitmask(BLACKBIT)))


#define valiswhite(x)   (iscollectable(x) && iswhite(gcvalue(x)))

#define keyiswhite(n)   (keyiscollectable(n) && iswhite(gckey(n)))


/*
** 对值中对象的受保护访问
*/
#define gcvalueN(o)     (iscollectable(o) ? gcvalue(o) : NULL)


#define markvalue(g,o) { checkliveness(g->mainthread,o); \
  if (valiswhite(o)) reallymarkobject(g,gcvalue(o)); }
# 定义宏，用于标记键
#define markkey(g, n)    { if keyiswhite(n) reallymarkobject(g,gckey(n)); }

# 定义宏，用于标记对象
#define markobject(g,t)    { if (iswhite(t)) reallymarkobject(g, obj2gco(t)); }

# 定义宏，用于标记可为空的对象
#define markobjectN(g,t)    { if (t) markobject(g,t); }

# 声明函数，用于真正标记对象
static void reallymarkobject (global_State *g, GCObject *o);

# 声明函数，用于原子操作
static lu_mem atomic (lua_State *L);

# 声明函数，用于进入扫描阶段
static void entersweep (lua_State *L);


# 通用函数
#======================================================
#


# 在哈希数组中获取最后一个元素的下一个位置
#define gnodelast(h)    gnode(h, cast_sizet(sizenode(h)))


# 获取对象的灰色列表
static GCObject **getgclist (GCObject *o) {
  switch (o->tt) {
    case LUA_VTABLE: return &gco2t(o)->gclist;
    case LUA_VLCL: return &gco2lcl(o)->gclist;
    case LUA_VCCL: return &gco2ccl(o)->gclist;
    case LUA_VTHREAD: return &gco2th(o)->gclist;
    case LUA_VPROTO: return &gco2p(o)->gclist;
    case LUA_VUSERDATA: {
      Udata *u = gco2u(o);
      lua_assert(u->nuvalue > 0);
      return &u->gclist;
    }
    default: lua_assert(0); return 0;
  }
}


# 将一个可收集对象'o'链接到已知类型的列表'p'中
#（必须是一个宏，以便访问不同类型中的'gclist'字段）
#define linkgclist(o,p)    linkgclist_(obj2gco(o), &(o)->gclist, &(p))

# 将一个可收集对象'o'链接到列表'p'中
#define linkobjgclist(o,p) linkgclist_(obj2gco(o), getgclist(o), &(p))


# 清除表中空条目的键。如果条目为空，则将其标记为死亡。这允许收集键，但保留其条目在表中：删除它可能会破坏链，并可能会破坏
/*
** 清除键的函数，用于清除表中的无用键
** 在其他地方不会操作无用键，因为其关联的空值足以表示该条目在逻辑上为空
*/
static void clearkey (Node *n) {
  lua_assert(isempty(gval(n)));  // 断言值为空
  if (keyiscollectable(n))  // 如果键是可收集的
    setdeadkey(n);  // 将键标记为无用的，即移除它
}


/*
** 判断键或值是否可以从弱表中清除
** 不可收集的对象永远不会从弱表中移除
** 字符串被视为“值”，因此也永远不会被移除
** 对于其他对象：如果真的被收集了，就不能保留它们；对于正在被终结的对象，保留它们作为键，但不作为值
*/
static int iscleared (global_State *g, const GCObject *o) {
  if (o == NULL) return 0;  // 非可收集的值
  else if (novariant(o->tt) == LUA_TSTRING) {
    markobject(g, o);  // 字符串被视为“值”，因此永远不会是弱引用
    return 0;
  }
  else return iswhite(o);  // 返回对象是否为白色
}


/*
** 将收集器向前移动的屏障，即标记黑色对象'o'指向的白色对象'v'
** 在分代模式下，如果'o'是老对象，'v'也必须变成老对象；但是，它不能直接被标记为OLD，因为它可能仍然指向非老对象
** 因此，它被标记为OLD0。在下一个周期中，它将变为OLD1，在下一个周期中它最终将变为OLD（正常的老对象）。到那时，它指向的任何对象也将是老对象
** 如果在增量扫描阶段调用它，它会将黑色对象清除为白色（扫描它），以避免对同一对象进行其他屏障调用。（在分代模式下无法这样做，因为它的扫描不区分白色和死亡对象）
*/
void luaC_barrier_ (lua_State *L, GCObject *o, GCObject *v) {
  global_State *g = G(L);
  lua_assert(isblack(o) && iswhite(v) && !isdead(g, v) && !isdead(g, o));  // 断言'o'为黑色，'v'为白色，且不是死亡对象
  if (keepinvariant(g)) {  // 必须保持不变性？
    reallymarkobject(g, v);  // 恢复不变性
    # 如果对象 o 是老对象
    if (isold(o)) {
      # 断言：白色对象不可能是老对象
      lua_assert(!isold(v));  /* white object could not be old */
      # 设置对象 v 的年龄为 G_OLD0，恢复世代不变性
      setage(v, G_OLD0);  /* restore generational invariant */
    }
  }
  else {  /* sweep phase */
    # 断言：处于扫描阶段
    lua_assert(issweepphase(g));
    # 如果垃圾收集器的类型是增量模式
    if (g->gckind == KGC_INC)  /* incremental mode? */
      # 将对象 o 标记为白色，以避免其他屏障
      makewhite(g, o);  /* mark 'o' as white to avoid other barriers */
  }
/*
** barrier that moves collector backward, that is, mark the black object
** pointing to a white object as gray again.
*/
void luaC_barrierback_ (lua_State *L, GCObject *o) {
  global_State *g = G(L);
  lua_assert(isblack(o) && !isdead(g, o));  // 检查对象是否为黑色且不是死对象
  lua_assert((g->gckind == KGC_GEN) == (isold(o) && getage(o) != G_TOUCHED1));  // 检查是否为分代模式
  if (getage(o) == G_TOUCHED2)  /* already in gray list? */  // 如果对象已经在灰色列表中
    set2gray(o);  /* make it gray to become touched1 */  // 将对象标记为灰色以变为touched1
  else  /* link it in 'grayagain' and paint it gray */  // 否则将对象链接到'grayagain'并将其标记为灰色
    linkobjgclist(o, g->grayagain);  // 将对象链接到'grayagain'并标记为灰色
  if (isold(o))  /* generational mode? */  // 如果是分代模式
    setage(o, G_TOUCHED1);  /* touched in current cycle */  // 在当前周期中标记为touched1
}


void luaC_fix (lua_State *L, GCObject *o) {
  global_State *g = G(L);
  lua_assert(g->allgc == o);  /* object must be 1st in 'allgc' list! */  // 对象必须是'allgc'列表中的第一个
  set2gray(o);  /* they will be gray forever */  // 将对象永远标记为灰色
  setage(o, G_OLD);  /* and old forever */  // 并永远标记为老对象
  g->allgc = o->next;  /* remove object from 'allgc' list */  // 从'allgc'列表中移除对象
  o->next = g->fixedgc;  /* link it to 'fixedgc' list */  // 将对象链接到'fixedgc'列表
  g->fixedgc = o;  // 将对象链接到'fixedgc'列表
}


/*
** create a new collectable object (with given type and size) and link
** it to 'allgc' list.
*/
GCObject *luaC_newobj (lua_State *L, int tt, size_t sz) {
  global_State *g = G(L);
  GCObject *o = cast(GCObject *, luaM_newobject(L, novariant(tt), sz));  // 创建一个新的可收集对象并将其链接到'allgc'列表
  o->marked = luaC_white(g);  // 将对象标记为白色
  o->tt = tt;  // 设置对象的类型
  o->next = g->allgc;  // 将对象链接到'allgc'列表
  g->allgc = o;  // 将对象链接到'allgc'列表
  return o;  // 返回对象
}

/* }====================================================== */



/*
** {======================================================
** Mark functions
** =======================================================
*/


/*
** Mark an object.  Userdata with no user values, strings, and closed upvalues are visited and turned black here.  Open upvalues are
** already indirectly linked through their respective threads in the 'twups' list, so they don't go to the gray list; nevertheless, they
** are kept gray to avoid barriers, as their values will be revisited by the thread or by 'remarkupvals'.  Other objects are added to the
/*
** 将对象标记为黑色，以便稍后访问（并将其变为黑色）。userdata 和 upvalues 都可以递归调用此函数，但递归的层级最多为两级：upvalue 不能引用另一个 upvalue（只有闭包可以），而 userdata 的元表必须是一个表。
*/
static void reallymarkobject (global_State *g, GCObject *o) {
  switch (o->tt) {
    case LUA_VSHRSTR:
    case LUA_VLNGSTR: {
      set2black(o);  /* 没有需要访问的内容 */
      break;
    }
    case LUA_VUPVAL: {
      UpVal *uv = gco2upv(o);
      if (upisopen(uv))
        set2gray(uv);  /* 开放的 upvalue 保持灰色 */
      else
        set2black(uv);  /* 已关闭的 upvalue 在此处访问 */
      markvalue(g, uv->v);  /* 标记其内容 */
      break;
    }
    case LUA_VUSERDATA: {
      Udata *u = gco2u(o);
      if (u->nuvalue == 0) {  /* 没有用户值？ */
        markobjectN(g, u->metatable);  /* 标记其元表 */
        set2black(u);  /* 没有其他需要标记的内容 */
        break;
      }
      /* else... */
    }  /* FALLTHROUGH */
    case LUA_VLCL: case LUA_VCCL: case LUA_VTABLE:
    case LUA_VTHREAD: case LUA_VPROTO: {
      linkobjgclist(o, g->gray);  /* 稍后访问 */
      break;
    }
    default: lua_assert(0); break;
  }
}


/*
** 标记基本类型的元方法
*/
static void markmt (global_State *g) {
  int i;
  for (i=0; i < LUA_NUMTAGS; i++)
    markobjectN(g, g->mt[i]);
}


/*
** 标记正在被最终化的对象列表中的所有对象
*/
static lu_mem markbeingfnz (global_State *g) {
  GCObject *o;
  lu_mem count = 0;
  for (o = g->tobefnz; o != NULL; o = o->next) {
    count++;
    markobject(g, o);
  }
  return count;
}


/*
** 对于每个未标记的线程，模拟每个开放的 upvalue 与其值之间的屏障。（如果线程被收集，值将被分配给 upvalue，但这时对屏障来说可能太晚了。"屏障" 不需要检查颜色：未标记的
*/
# 标记并处理线程的上值
static int remarkupvals (global_State *g) {
  lua_State *thread;  // 定义 lua_State 类型的变量 thread
  lua_State **p = &g->twups;  // 定义 lua_State 类型的指针变量 p，指向全局状态中的 twups
  int work = 0;  // 估计在此处完成的工作量
  while ((thread = *p) != NULL) {  // 当指针变量 p 指向的线程不为空时循环
    work++;  // 增加工作量计数
    if (!iswhite(thread) && thread->openupval != NULL)  // 如果线程不是白色且具有打开的上值
      p = &thread->twups;  // 更新指针变量 p，指向线程的 twups
    else {  // 线程未标记或没有上值
      UpVal *uv;  // 定义 UpVal 类型的变量 uv
      lua_assert(!isold(thread) || thread->openupval == NULL);  // 断言线程不是旧的或没有打开的上值
      *p = thread->twups;  // 从列表中移除线程
      thread->twups = thread;  // 标记线程已经不在列表中
      for (uv = thread->openupval; uv != NULL; uv = uv->u.open.next) {  // 遍历线程的打开上值
        lua_assert(getage(uv) <= getage(thread));  // 断言上值的年龄小于等于线程的年龄
        work++;  // 增加工作量计数
        if (!iswhite(uv)) {  // 上值已经被访问过？
          lua_assert(upisopen(uv) && isgray(uv));  // 断言上值是打开的且是灰色的
          markvalue(g, uv->v);  // 标记其值
        }
      }
    }
  }
  return work;  // 返回工作量
}

// 清除所有灰色列表
static void cleargraylists (global_State *g) {
  g->gray = g->grayagain = NULL;  // 重置灰色列表
  g->weak = g->allweak = g->ephemeron = NULL;  // 重置弱引用列表
}

// 标记根集并重置所有灰色列表，以开始新的收集
static void restartcollection (global_State *g) {
  cleargraylists(g);  // 清除灰色列表
  markobject(g, g->mainthread);  // 标记主线程
  markvalue(g, &g->l_registry);  // 标记注册表
  markmt(g);  // 标记元表
  markbeingfnz(g);  // 标记上一个周期中剩下的任何正在结束的对象
}
/*
** 检查对象'o'是否应该保留在'grayagain'列表中，以供'correctgraylist'后续处理。
** （它可以将所有旧对象放在列表中，然后将所有工作留给'correctgraylist'，
** 但是避免添加将被移除的元素更有效。）只有 TOUCHED1 对象需要在列表中。
** TOUCHED2 不需要返回到灰色列表，但它必须变为 OLD。（当'correctgraylist'发现 TOUCHED2 对象时会这样做。）
*/
static void genlink (global_State *g, GCObject *o) {
  lua_assert(isblack(o));
  if (getage(o) == G_TOUCHED1) {  /* 在本周期中被标记？ */
    linkobjgclist(o, g->grayagain);  /* 将其链接回'grayagain'列表 */
  }  /* 其他所有对象都不需要链接回去 */
  else if (getage(o) == G_TOUCHED2)
    changeage(o, G_TOUCHED2, G_OLD);  /* 提升对象年龄 */
}


/*
** 遍历具有弱值的表，并将其链接到适当的列表中。在传播阶段，将其保留在'grayagain'列表中，以便在原子阶段重新访问。
** 在原子阶段，如果表具有任何白色值，则将其放入'weak'列表中，以便清除。
*/
static void traverseweakvalue (global_State *g, Table *h) {
  Node *n, *limit = gnodelast(h);
  /* 如果有数组部分，则假定它可能具有白色值（现在遍历它以检查不值得） */
  int hasclears = (h->alimit > 0);
  for (n = gnode(h, 0); n < limit; n++) {  /* 遍历哈希部分 */
    if (isempty(gval(n)))  /* 条目为空？ */
      clearkey(n);  /* 清除其键 */
    else {
      lua_assert(!keyisnil(n));
      markkey(g, n);
      if (!hasclears && iscleared(g, gcvalueN(gval(n))))  /* 白色值？ */
        hasclears = 1;  /* 表将需要被清除 */
    }
  }
  if (g->gcstate == GCSatomic && hasclears)
    linkgclist(h, g->weak);  /* 稍后必须清除 */
  else
    linkgclist(h, g->grayagain);  /* 必须在原子阶段重新遍历 */
}


/*
** 遍历一个短命表并将其链接到适当的列表中。返回true
*/
/* 如果在这次遍历中有任何对象被标记（这意味着收敛必须继续）。在传播阶段，保持表在'grayagain'列表中，以便在原子阶段再次访问。在原子阶段，如果表中有任何白色->白色的条目，它在瞬变收敛期间必须被重新访问（因为该键可能变为黑色）。否则，如果它有任何白色键，表必须被清除（在原子阶段）。在分代模式下，一些表必须保留在一些灰色列表中进行后处理；这是通过'genlink'完成的。*/
static int traverseephemeron (global_State *g, Table *h, int inv) {
  int marked = 0;  /* 如果在这次遍历中有对象被标记，则为真 */
  int hasclears = 0;  /* 如果表中有白色键，则为真 */
  int hasww = 0;  /* 如果表中有条目"白色键->白色值"，则为真 */
  unsigned int i;
  unsigned int asize = luaH_realasize(h);
  unsigned int nsize = sizenode(h);
  /* 遍历数组部分 */
  for (i = 0; i < asize; i++) {
    if (valiswhite(&h->array[i])) {
      marked = 1;
      reallymarkobject(g, gcvalue(&h->array[i]));
    }
  }
  /* 遍历哈希部分；如果'inv'为真，则遍历降序（参见'convergeephemerons'） */
  for (i = 0; i < nsize; i++) {
    Node *n = inv ? gnode(h, nsize - 1 - i) : gnode(h, i);
    if (isempty(gval(n)))  /* 条目为空？ */
      clearkey(n);  /* 清除其键 */
    else if (iscleared(g, gckeyN(n))) {  /* 键未标记（尚未）？ */
      hasclears = 1;  /* 表必须被清除 */
      if (valiswhite(gval(n)))  /* 值尚未标记？ */
        hasww = 1;  /* 白色-白色条目 */
    }
    else if (valiswhite(gval(n))) {  /* 值尚未标记？ */
      marked = 1;
      reallymarkobject(g, gcvalue(gval(n)));  /* 现在标记它 */
    }
  }
  /* 将表链接到适当的列表中 */
  if (g->gcstate == GCSpropagate)
    linkgclist(h, g->grayagain);  /* 必须在原子阶段重新遍历它 */
  else if (hasww)  /* 表中有白色->白色的条目？ */
    linkgclist(h, g->ephemeron);  /* 将对象 h 添加到弱表链表中，需要再次传播 */
  else if (hasclears)  /* 表中是否有白色键？ */
    linkgclist(h, g->allweak);  /* 可能需要清理白色键 */
  else
    genlink(g, obj2gco(h));  /* 检查收集器是否仍然需要查看它 */
  return marked;
}
# 遍历强引用表
static void traversestrongtable (global_State *g, Table *h) {
  Node *n, *limit = gnodelast(h);  # 定义节点指针n和limit，limit指向哈希表的最后一个节点
  unsigned int i;  # 定义无符号整型变量i
  unsigned int asize = luaH_realasize(h);  # 获取哈希表的实际数组部分大小
  for (i = 0; i < asize; i++)  # 遍历数组部分
    markvalue(g, &h->array[i]);  # 标记数组部分的值
  for (n = gnode(h, 0); n < limit; n++) {  # 遍历哈希部分
    if (isempty(gval(n)))  # 判断节点是否为空
      clearkey(n);  # 清除节点的键
    else {
      lua_assert(!keyisnil(n));  # 断言节点的键不为空
      markkey(g, n);  # 标记节点的键
      markvalue(g, gval(n));  # 标记节点的值
    }
  }
  genlink(g, obj2gco(h));  # 生成链接
}

# 遍历表
static lu_mem traversetable (global_State *g, Table *h) {
  const char *weakkey, *weakvalue;  # 定义弱键和弱值
  const TValue *mode = gfasttm(g, h->metatable, TM_MODE);  # 获取元表的模式
  markobjectN(g, h->metatable);  # 标记元表
  if (mode && ttisstring(mode) &&  # 判断是否存在弱模式
      (cast_void(weakkey = strchr(svalue(mode), 'k')),  # 获取弱键
       cast_void(weakvalue = strchr(svalue(mode), 'v')),  # 获取弱值
       (weakkey || weakvalue))) {  # 判断是否真的是弱引用
    if (!weakkey)  # 如果没有弱键
      traverseweakvalue(g, h);  # 遍历弱值
    else if (!weakvalue)  # 如果没有弱值
      traverseephemeron(g, h, 0);  # 遍历短暂值
    else  # 如果都是弱引用
      linkgclist(h, g->allweak);  # 链接到弱引用链表
  }
  else  # 如果不是弱引用
    traversestrongtable(g, h);  # 遍历强引用表
  return 1 + h->alimit + 2 * allocsizenode(h);  # 返回值
}

# 遍历用户数据
static int traverseudata (global_State *g, Udata *u) {
  int i;  # 定义整型变量i
  markobjectN(g, u->metatable);  # 标记用户数据的元表
  for (i = 0; i < u->nuvalue; i++)  # 遍历用户数据的值
    markvalue(g, &u->uv[i].uv);  # 标记用户数据的值
  genlink(g, obj2gco(u));  # 生成链接
  return 1 + u->nuvalue;  # 返回值
}

# 遍历原型
static int traverseproto (global_State *g, Proto *f) {
  int i;  # 定义整型变量i
  markobjectN(g, f->source);  # 标记原型的源
  for (i = 0; i < f->sizek; i++)  # 遍历字面量
    markvalue(g, &f->k[i]);  # 标记字面量的值
  for (i = 0; i < f->sizeupvalues; i++)  # 遍历上值名
    // 标记 upvalues 中的名称对象
    markobjectN(g, f->upvalues[i].name);
  // 遍历嵌套的函数原型，标记对象
  for (i = 0; i < f->sizep; i++)  /* mark nested protos */
    markobjectN(g, f->p[i]);
  // 遍历本地变量名，标记对象
  for (i = 0; i < f->sizelocvars; i++)  /* mark local-variable names */
    markobjectN(g, f->locvars[i].varname);
  // 返回标记对象的数量
  return 1 + f->sizek + f->sizeupvalues + f->sizep + f->sizelocvars;
}

// 遍历 C 闭包，标记其上值
static int traverseCclosure (global_State *g, CClosure *cl) {
  int i;
  for (i = 0; i < cl->nupvalues; i++)  // 标记其上值
    markvalue(g, &cl->upvalue[i]);
  return 1 + cl->nupvalues;
}

/*
** 遍历 Lua 闭包，标记其原型和上值。
** （在闭包创建时，原型和上值都可以为 NULL。）
*/
static int traverseLclosure (global_State *g, LClosure *cl) {
  int i;
  markobjectN(g, cl->p);  // 标记其原型
  for (i = 0; i < cl->nupvalues; i++) {  // 访问其上值
    UpVal *uv = cl->upvals[i];
    markobjectN(g, uv);  // 标记上值
  }
  return 1 + cl->nupvalues;
}

/*
** 遍历线程，标记栈中直到其顶部的元素，并在最终遍历中清理栈的其余部分。
** 这确保整个栈都有有效（非死亡）对象。
** 线程没有屏障。在一般模式下，旧线程必须在每个周期被访问，因为它们可能指向年轻对象。
** 在增量模式下，线程在周期结束前仍然可以被修改，因此必须在原子阶段再次被访问。
** 为了确保这些访问，如果线程不是新的（只能在生成模式下发生）或者遍历处于传播阶段（只能在增量模式下发生），
** 线程必须返回到灰色列表中。
*/
static int traversethread (global_State *g, lua_State *th) {
  UpVal *uv;
  StkId o = th->stack;
  if (isold(th) || g->gcstate == GCSpropagate)
    linkgclist(th, g->grayagain);  // 插入到 'grayagain' 列表中
  if (o == NULL)
    return 1;  // 栈尚未完全构建
  lua_assert(g->gcstate == GCSatomic ||
             th->openupval == NULL || isintwups(th));
  for (; o < th->top; o++)  // 标记栈中的活跃元素
    markvalue(g, s2v(o));
  for (uv = th->openupval; uv != NULL; uv = uv->u.open.next)
    markobject(g, uv);  // 开放的上值不能被收集
  if (g->gcstate == GCSatomic) {  // 最终遍历？
    # 遍历从栈顶到额外栈空间的每个元素
    for (; o < th->stack_last + EXTRA_STACK; o++)
      # 将栈中已经不再使用的部分设置为 nil 值，清理无用的栈片段
      setnilvalue(s2v(o));  /* clear dead stack slice */
    # 如果线程不在 'twups' 列表中，并且有未关闭的 upvalue
    if (!isintwups(th) && th->openupval != NULL) {
      # 将线程重新链接到 'twups' 列表中
      th->twups = g->twups;  /* link it back to the list */
      g->twups = th;
    }
  }
  # 如果不是处于紧急循环状态
  else if (!g->gcemergency)
    # 在紧急循环状态下不改变栈的大小
    luaD_shrinkstack(th); /* do not change stack in emergency cycle */
  # 返回栈的大小加一
  return 1 + stacksize(th);
/*
** traverse one gray object, turning it to black.
*/
static lu_mem propagatemark (global_State *g) {
  GCObject *o = g->gray;  // 获取全局状态中的灰色对象
  nw2black(o);  // 将灰色对象转换为黑色
  g->gray = *getgclist(o);  /* remove from 'gray' list */  // 从灰色列表中移除该对象
  switch (o->tt) {  // 根据对象的类型进行不同的处理
    case LUA_VTABLE: return traversetable(g, gco2t(o));  // 如果是表类型，则调用traversetable函数
    case LUA_VUSERDATA: return traverseudata(g, gco2u(o));  // 如果是用户数据类型，则调用traverseudata函数
    case LUA_VLCL: return traverseLclosure(g, gco2lcl(o));  // 如果是Lua闭包类型，则调用traverseLclosure函数
    case LUA_VCCL: return traverseCclosure(g, gco2ccl(o));  // 如果是C闭包类型，则调用traverseCclosure函数
    case LUA_VPROTO: return traverseproto(g, gco2p(o));  // 如果是原型类型，则调用traverseproto函数
    case LUA_VTHREAD: return traversethread(g, gco2th(o));  // 如果是线程类型，则调用traversethread函数
    default: lua_assert(0); return 0;  // 默认情况下，断言失败并返回0
  }
}


static lu_mem propagateall (global_State *g) {
  lu_mem tot = 0;  // 初始化总数为0
  while (g->gray)  // 当灰色列表不为空时
    tot += propagatemark(g);  // 调用propagatemark函数，并累加返回值到总数中
  return tot;  // 返回总数
}


/*
** Traverse all ephemeron tables propagating marks from keys to values.
** Repeat until it converges, that is, nothing new is marked. 'dir'
** inverts the direction of the traversals, trying to speed up
** convergence on chains in the same table.
**
*/
static void convergeephemerons (global_State *g) {
  int changed;  // 定义变量changed
  int dir = 0;  // 初始化方向为0
  do {
    GCObject *w;  // 定义GCObject指针w
    GCObject *next = g->ephemeron;  /* get ephemeron list */  // 获取瞬变表列表
    g->ephemeron = NULL;  /* tables may return to this list when traversed */  // 当遍历时，表可能会返回到这个列表中
    changed = 0;  // 初始化变量changed为0
    while ((w = next) != NULL) {  /* for each ephemeron table */  // 遍历每个瞬变表
      Table *h = gco2t(w);  // 将GCObject转换为Table类型
      next = h->gclist;  /* list is rebuilt during loop */  // 在循环期间重建列表
      nw2black(h);  /* out of the list (for now) */  // 将表从列表中移除（暂时）
      if (traverseephemeron(g, h, dir)) {  /* marked some value? */  // 如果标记了一些值
        propagateall(g);  /* propagate changes */  // 传播更改
        changed = 1;  /* will have to revisit all ephemeron tables */  // 将不得不重新访问所有瞬变表
      }
    }
    dir = !dir;  /* invert direction next time */  // 下一次反转方向
  } while (changed);  /* repeat until no more changes */  // 重复直到没有更改为止
}

/* }====================================================== */


/*
** {======================================================
** Sweep Functions
** =======================================================
*/
/*
** 从列表 'l' 中的所有弱引用表中清除具有未标记键的条目
*/
static void clearbykeys (global_State *g, GCObject *l) {
  for (; l; l = gco2t(l)->gclist) {
    Table *h = gco2t(l);
    Node *limit = gnodelast(h);
    Node *n;
    for (n = gnode(h, 0); n < limit; n++) {
      if (iscleared(g, gckeyN(n)))  /* 未标记的键？ */
        setempty(gval(n));  /* 移除条目 */
      if (isempty(gval(n)))  /* 条目为空？ */
        clearkey(n);  /* 清除其键 */
    }
  }
}


/*
** 从列表 'l' 中的所有弱引用表中清除具有未标记值的条目，直到元素 'f'
*/
static void clearbyvalues (global_State *g, GCObject *l, GCObject *f) {
  for (; l != f; l = gco2t(l)->gclist) {
    Table *h = gco2t(l);
    Node *n, *limit = gnodelast(h);
    unsigned int i;
    unsigned int asize = luaH_realasize(h);
    for (i = 0; i < asize; i++) {
      TValue *o = &h->array[i];
      if (iscleared(g, gcvalueN(o)))  /* 值已被回收？ */
        setempty(o);  /* 移除条目 */
    }
    for (n = gnode(h, 0); n < limit; n++) {
      if (iscleared(g, gcvalueN(gval(n))))  /* 未标记的值？ */
        setempty(gval(n));  /* 移除条目 */
      if (isempty(gval(n)))  /* 条目为空？ */
        clearkey(n);  /* 清除其键 */
    }
  }
}


static void freeupval (lua_State *L, UpVal *uv) {
  if (upisopen(uv))
    luaF_unlinkupval(uv);
  luaM_free(L, uv);
}


static void freeobj (lua_State *L, GCObject *o) {
  switch (o->tt) {
    case LUA_VPROTO:
      luaF_freeproto(L, gco2p(o));
      break;
    case LUA_VUPVAL:
      freeupval(L, gco2upv(o));
      break;
    case LUA_VLCL: {
      LClosure *cl = gco2lcl(o);
      luaM_freemem(L, cl, sizeLclosure(cl->nupvalues));
      break;
    }
    case LUA_VCCL: {
      CClosure *cl = gco2ccl(o);
      luaM_freemem(L, cl, sizeCclosure(cl->nupvalues));
      break;
    }
    case LUA_VTABLE:
      luaH_free(L, gco2t(o));
      break;
    case LUA_VTHREAD:
      luaE_freethread(L, gco2th(o));
      break;
    # 如果对象类型是用户数据
    case LUA_VUSERDATA: {
      # 将对象转换为用户数据类型
      Udata *u = gco2u(o);
      # 释放用户数据占用的内存
      luaM_freemem(L, o, sizeudata(u->nuvalue, u->len));
      # 退出 switch 语句
      break;
    }
    # 如果对象类型是短字符串
    case LUA_VSHRSTR: {
      # 将对象转换为短字符串类型
      TString *ts = gco2ts(o);
      # 从哈希表中移除该短字符串
      luaS_remove(L, ts);  /* remove it from hash table */
      # 释放短字符串占用的内存
      luaM_freemem(L, ts, sizelstring(ts->shrlen));
      # 退出 switch 语句
      break;
    }
    # 如果对象类型是长字符串
    case LUA_VLNGSTR: {
      # 将对象转换为长字符串类型
      TString *ts = gco2ts(o);
      # 释放长字符串占用的内存
      luaM_freemem(L, ts, sizelstring(ts->u.lnglen));
      # 退出 switch 语句
      break;
    }
    # 如果对象类型不是上述任何一种，断言失败
    default: lua_assert(0);
  }
/*
** sweep at most 'countin' elements from a list of GCObjects erasing dead
** objects, where a dead object is one marked with the old (non current)
** white; change all non-dead objects back to white, preparing for next
** collection cycle. Return where to continue the traversal or NULL if
** list is finished. ('*countout' gets the number of elements traversed.)
*/
static GCObject **sweeplist (lua_State *L, GCObject **p, int countin,
                             int *countout) {
  global_State *g = G(L);  // 获取全局状态
  int ow = otherwhite(g);  // 获取非当前白色的标记
  int i;
  int white = luaC_white(g);  // 当前白色标记
  for (i = 0; *p != NULL && i < countin; i++) {  // 遍历列表中的元素
    GCObject *curr = *p;  // 获取当前元素
    int marked = curr->marked;  // 获取当前元素的标记
    if (isdeadm(ow, marked)) {  // 判断当前元素是否为死对象
      *p = curr->next;  // 从列表中移除当前元素
      freeobj(L, curr);  // 释放当前元素
    }
    else {  // 将非死对象的标记改回白色
      curr->marked = cast_byte((marked & ~maskgcbits) | white);
      p = &curr->next;  // 移动到下一个元素
    }
  }
  if (countout)
    *countout = i;  // 设置遍历的元素数量
  return (*p == NULL) ? NULL : p;  // 如果列表遍历完成则返回 NULL，否则返回继续遍历的位置
}


/*
** sweep a list until a live object (or end of list)
*/
static GCObject **sweeptolive (lua_State *L, GCObject **p) {
  GCObject **old = p;  // 保存初始位置
  do {
    p = sweeplist(L, p, 1, NULL);  // 从当前位置开始遍历列表
  } while (p == old);  // 直到遍历到活对象或者列表结束
  return p;  // 返回遍历结束的位置
}

/* }====================================================== */


/*
** {======================================================
** Finalization
** =======================================================
*/

/*
** If possible, shrink string table.
*/
static void checkSizes (lua_State *L, global_State *g) {
  if (!g->gcemergency) {  // 如果不是在紧急模式下
    if (g->strt.nuse < g->strt.size / 4) {  // 如果字符串表太大
      l_mem olddebt = g->GCdebt;  // 保存旧的 GC 负债
      luaS_resize(L, g->strt.size / 2);  // 缩小字符串表
      g->GCestimate += g->GCdebt - olddebt;  // 修正估计值
    }
  }
}


/*
** Get the next udata to be finalized from the 'tobefnz' list, and
*/
/*
** 将对象链接回 'allgc' 列表中。
*/
static GCObject *udata2finalize (global_State *g) {
  GCObject *o = g->tobefnz;  /* 获取第一个元素 */
  lua_assert(tofinalize(o));
  g->tobefnz = o->next;  /* 从 'tobefnz' 列表中移除 */
  o->next = g->allgc;  /* 将其返回到 'allgc' 列表中 */
  g->allgc = o;
  resetbit(o->marked, FINALIZEDBIT);  /* 对象恢复为“正常”状态 */
  if (issweepphase(g))
    makewhite(g, o);  /* “扫描”对象 */
  else if (getage(o) == G_OLD1)
    g->firstold1 = o;  /* 它是列表中第一个 OLD1 对象 */
  return o;
}

static void dothecall (lua_State *L, void *ud) {
  UNUSED(ud);
  luaD_callnoyield(L, L->top - 2, 0);
}

static void GCTM (lua_State *L) {
  global_State *g = G(L);
  const TValue *tm;
  TValue v;
  lua_assert(!g->gcemergency);
  setgcovalue(L, &v, udata2finalize(g));
  tm = luaT_gettmbyobj(L, &v, TM_GC);
  if (!notm(tm)) {  /* 是否有终结器？ */
    int status;
    lu_byte oldah = L->allowhook;
    int oldgcstp  = g->gcstp;
    g->gcstp |= GCSTPGC;  /* 避免 GC 步骤 */
    L->allowhook = 0;  /* 在 GC 元方法执行期间停止调试钩子 */
    setobj2s(L, L->top++, tm);  /* 压入终结器... */
    setobj2s(L, L->top++, &v);  /* ... 和其参数 */
    L->ci->callstatus |= CIST_FIN;  /* 将运行终结器 */
    status = luaD_pcall(L, dothecall, NULL, savestack(L, L->top - 2), 0);
    L->ci->callstatus &= ~CIST_FIN;  /* 不再运行终结器 */
    L->allowhook = oldah;  /* 恢复钩子 */
    g->gcstp = oldgcstp;  /* 恢复状态 */
    if (l_unlikely(status != LUA_OK)) {  /* 运行 __gc 时出错？ */
      luaE_warnerror(L, "__gc");
      L->top--;  /* 弹出错误对象 */
    }
  }
}

/*
** 调用一些终结器
*/
static int runafewfinalizers (lua_State *L, int n) {
  global_State *g = G(L);
  int i;
  for (i = 0; i < n && g->tobefnz; i++)
    GCTM(L);  /* 调用一个终结器 */
  return i;
}

/*
** 调用所有待处理的终结器
*/
/*
** 调用所有待处理的 finalizers
*/
static void callallpendingfinalizers (lua_State *L) {
  global_State *g = G(L);
  while (g->tobefnz)
    GCTM(L);
}


/*
** 查找列表 'p' 中最后一个 'next' 字段（用于在其末尾添加元素）
*/
static GCObject **findlast (GCObject **p) {
  while (*p != NULL)
    p = &(*p)->next;
  return p;
}


/*
** 将所有不可达对象（或者所有对象）从 'finobj' 列表移动到 'tobefnz' 列表（以进行 finalization）
** （注意，'finobjold1' 之后的对象不可能是白色，因此不需要遍历它们。在增量模式下，'finobjold1' 为 NULL，因此整个列表都会被遍历。）
*/
static void separatetobefnz (global_State *g, int all) {
  GCObject *curr;
  GCObject **p = &g->finobj;
  GCObject **lastnext = findlast(&g->tobefnz);
  while ((curr = *p) != g->finobjold1) {  /* 遍历所有可 finalization 的对象 */
    lua_assert(tofinalize(curr));
    if (!(iswhite(curr) || all))  /* 不在被收集？ */
      p = &curr->next;  /* 不需要处理它 */
    else {
      if (curr == g->finobjsur)  /* 移除 'finobjsur'？ */
        g->finobjsur = curr->next;  /* 修正它 */
      *p = curr->next;  /* 从 'finobj' 列表中移除 'curr' */
      curr->next = *lastnext;  /* 连接到 'tobefnz' 列表的末尾 */
      *lastnext = curr;
      lastnext = &curr->next;
    }
  }
}


/*
** 如果指针 'p' 指向 'o'，将其移动到下一个元素
*/
static void checkpointer (GCObject **p, GCObject *o) {
  if (o == *p)
    *p = o->next;
}


/*
** 当从列表 'allgc' 中移除对象 'o' 时，修正指向对象内部的指针
*/
static void correctpointers (global_State *g, GCObject *o) {
  checkpointer(&g->survival, o);
  checkpointer(&g->old1, o);
  checkpointer(&g->reallyold, o);
  checkpointer(&g->firstold1, o);
}


/*
** 如果对象 'o' 有 finalizer，从 'allgc' 列表中移除它（必须搜索列表找到它），并将其链接到 'finobj' 列表中
*/
void luaC_checkfinalizer (lua_State *L, GCObject *o, Table *mt) {
  global_State *g = G(L);  // 获取全局状态
  if (tofinalize(o) ||                 /* obj. is already marked... */
      gfasttm(g, mt, TM_GC) == NULL ||    /* or has no finalizer... */
      (g->gcstp & GCSTPCLS))                   /* or closing state? */
    return;  /* nothing to be done */
  else {  /* move 'o' to 'finobj' list */
    GCObject **p;  // 定义 GCObject 指针
    if (issweepphase(g)) {
      makewhite(g, o);  /* "sweep" object 'o' */
      if (g->sweepgc == &o->next)  /* should not remove 'sweepgc' object */
        g->sweepgc = sweeptolive(L, g->sweepgc);  /* change 'sweepgc' */
    }
    else
      correctpointers(g, o);  // 修正指针
    /* search for pointer pointing to 'o' */
    for (p = &g->allgc; *p != o; p = &(*p)->next) { /* empty */ }  // 查找指向 'o' 的指针
    *p = o->next;  /* remove 'o' from 'allgc' list */  // 从 'allgc' 列表中移除 'o'
    o->next = g->finobj;  /* link it in 'finobj' list */  // 将 'o' 链接到 'finobj' 列表中
    g->finobj = o;  // 更新 'finobj' 列表
    l_setbit(o->marked, FINALIZEDBIT);  /* mark it as such */  // 将其标记为已完成
  }
}

/* }====================================================== */


/*
** {======================================================
** Generational Collector
** =======================================================
*/

static void setpause (global_State *g);  // 设置暂停

/*
** Sweep a list of objects to enter generational mode.  Deletes dead
** objects and turns the non dead to old. All non-dead threads---which
** are now old---must be in a gray list. Everything else is not in a
** gray list. Open upvalues are also kept gray.
*/
static void sweep2old (lua_State *L, GCObject **p) {
  GCObject *curr;  // 当前对象
  global_State *g = G(L);  // 获取全局状态
  while ((curr = *p) != NULL) {  // 遍历对象列表
    if (iswhite(curr)) {  /* is 'curr' dead? */  // 判断 'curr' 是否已死亡
      lua_assert(isdead(g, curr));  // 断言 'curr' 已死亡
      *p = curr->next;  /* remove 'curr' from list */  // 从列表中移除 'curr'
      freeobj(L, curr);  /* erase 'curr' */  // 释放 'curr'
    }
    else {  /* 所有幸存的对象都变成老对象 */
      setage(curr, G_OLD);  // 将当前对象的年龄设置为老对象
      if (curr->tt == LUA_VTHREAD) {  /* 线程必须被监视 */
        lua_State *th = gco2th(curr);
        linkgclist(th, g->grayagain);  // 将线程插入到'grayagain'列表中
      }
      else if (curr->tt == LUA_VUPVAL && upisopen(gco2upv(curr)))
        set2gray(curr);  // 开放的 upvalue 总是灰色
      else  /* 其他所有对象都是黑色 */
        nw2black(curr);
      p = &curr->next;  /* 转到下一个元素 */
    }
  }
/*
** Sweep for generational mode. Delete dead objects. (Because the
** collection is not incremental, there are no "new white" objects
** during the sweep. So, any white object must be dead.) For
** non-dead objects, advance their ages and clear the color of
** new objects. (Old objects keep their colors.)
** The ages of G_TOUCHED1 and G_TOUCHED2 objects cannot be advanced
** here, because these old-generation objects are usually not swept
** here. They will all be advanced in 'correctgraylist'. That function
** will also remove objects turned white here from any gray list.
*/
static GCObject **sweepgen (lua_State *L, global_State *g, GCObject **p,
                            GCObject *limit, GCObject **pfirstold1) {
  static const lu_byte nextage[] = {
    G_SURVIVAL,  /* from G_NEW */
    G_OLD1,      /* from G_SURVIVAL */
    G_OLD1,      /* from G_OLD0 */
    G_OLD,       /* from G_OLD1 */
    G_OLD,       /* from G_OLD (do not change) */
    G_TOUCHED1,  /* from G_TOUCHED1 (do not change) */
    G_TOUCHED2   /* from G_TOUCHED2 (do not change) */
  };
  int white = luaC_white(g);  // 获取白色对象的标记
  GCObject *curr;  // 当前对象
  while ((curr = *p) != limit) {  // 遍历对象链表
    if (iswhite(curr)) {  // 判断当前对象是否为白色（即是否已死亡）
      lua_assert(!isold(curr) && isdead(g, curr));  // 断言当前对象不是老对象且已死亡
      *p = curr->next;  // 从链表中移除当前对象
      freeobj(L, curr);  // 释放当前对象
    }
    else {  // 如果对象不是白色
      if (getage(curr) == G_NEW) {  // 如果对象是新对象，则重新标记为白色
        int marked = curr->marked & ~maskgcbits;  // 消除GC位
        curr->marked = cast_byte(marked | G_SURVIVAL | white);  // 重新标记为白色
      }
      else {  // 如果对象不是新对象
        setage(curr, nextage[getage(curr)]);  // 更新对象的年龄
        if (getage(curr) == G_OLD1 && *pfirstold1 == NULL)
          *pfirstold1 = curr;  // 如果是第一个OLD1对象，则记录下来
      }
      p = &curr->next;  // 移动到下一个元素
    }
  }
  return p;  // 返回下一个元素的指针
}
/*
** Traverse a list making all its elements white and clearing their
** age. In incremental mode, all objects are 'new' all the time,
** except for fixed strings (which are always old).
*/
static void whitelist (global_State *g, GCObject *p) {
  int white = luaC_white(g);  // 获取白色标记
  for (; p != NULL; p = p->next)  // 遍历链表
    p->marked = cast_byte((p->marked & ~maskgcbits) | white);  // 将元素标记为白色并清除年龄
}


/*
** Correct a list of gray objects. Return pointer to where rest of the
** list should be linked.
** Because this correction is done after sweeping, young objects might
** be turned white and still be in the list. They are only removed.
** 'TOUCHED1' objects are advanced to 'TOUCHED2' and remain on the list;
** Non-white threads also remain on the list; 'TOUCHED2' objects become
** regular old; they and anything else are removed from the list.
*/
static GCObject **correctgraylist (GCObject **p) {
  GCObject *curr;
  while ((curr = *p) != NULL) {  // 遍历灰色对象列表
    GCObject **next = getgclist(curr);  // 获取下一个元素的指针
    if (iswhite(curr))  // 如果当前对象是白色
      goto remove;  // 移除所有白色对象
    else if (getage(curr) == G_TOUCHED1) {  // 如果当前对象在本次循环中被访问
      lua_assert(isgray(curr));  // 断言当前对象是灰色
      nw2black(curr);  // 将对象标记为黑色，为下一个屏障做准备
      changeage(curr, G_TOUCHED1, G_TOUCHED2);  // 将对象的年龄从TOUCHED1改为TOUCHED2
      goto remain;  // 保留对象在列表中，并继续处理下一个元素
    }
    else if (curr->tt == LUA_VTHREAD) {  // 如果当前对象是非白色线程
      lua_assert(isgray(curr));  // 断言当前对象是灰色
      goto remain;  // 保留非白色线程在列表中
    }
    else {  // 其他情况下移除对象
      lua_assert(isold(curr));  // 年轻对象应该是白色的
      if (getage(curr) == G_TOUCHED2)  // 从TOUCHED2状态转为OLD状态
        changeage(curr, G_TOUCHED2, G_OLD);  // 将对象的年龄从TOUCHED2改为OLD
      nw2black(curr);  // 将对象标记为黑色（即将被移除）
      goto remove;  // 移除对象
    }
    remove: *p = *next; continue;  // 移除当前对象并继续处理下一个元素
    remain: p = next; continue;  // 保留当前对象并继续处理下一个元素
  }
  return p;  // 返回处理后的列表
}


/*
** Correct all gray lists, coalescing them into 'grayagain'.
*/
static void correctgraylists (global_State *g) {
  // 修正灰色对象列表，返回修正后的列表
  GCObject **list = correctgraylist(&g->grayagain);
  // 将 g->weak 对象添加到修正后的列表中，并将 g->weak 置空
  *list = g->weak; g->weak = NULL;
  // 继续修正灰色对象列表
  list = correctgraylist(list);
  // 将 g->allweak 对象添加到修正后的列表中，并将 g->allweak 置空
  *list = g->allweak; g->allweak = NULL;
  // 继续修正灰色对象列表
  list = correctgraylist(list);
  // 将 g->ephemeron 对象添加到修正后的列表中，并将 g->ephemeron 置空
  *list = g->ephemeron; g->ephemeron = NULL;
  // 继续修正灰色对象列表
  correctgraylist(list);
}


/*
** Mark black 'OLD1' objects when starting a new young collection.
** Gray objects are already in some gray list, and so will be visited
** in the atomic step.
*/
static void markold (global_State *g, GCObject *from, GCObject *to) {
  // 遍历 from 到 to 之间的对象
  GCObject *p;
  for (p = from; p != to; p = p->next) {
    // 如果对象的年龄为 G_OLD1
    if (getage(p) == G_OLD1) {
      // 断言对象不是白色
      lua_assert(!iswhite(p));
      // 将对象的年龄从 G_OLD1 改为 G_OLD
      changeage(p, G_OLD1, G_OLD);  /* now they are old */
      // 如果对象是黑色，则标记该对象
      if (isblack(p))
        reallymarkobject(g, p);
    }
  }
}


/*
** Finish a young-generation collection.
*/
static void finishgencycle (lua_State *L, global_State *g) {
  // 修正灰色对象列表
  correctgraylists(g);
  // 检查大小
  checkSizes(L, g);
  // 设置垃圾回收状态为 GCSpropagate，跳过重新启动
  g->gcstate = GCSpropagate;  /* skip restart */
  // 如果不是紧急垃圾回收，则调用所有待处理的终结器
  if (!g->gcemergency)
    callallpendingfinalizers(L);
}


/*
** Does a young collection. First, mark 'OLD1' objects. Then does the
** atomic step. Then, sweep all lists and advance pointers. Finally,
** finish the collection.
*/
static void youngcollection (lua_State *L, global_State *g) {
  // 指向第一个非死亡存活对象的指针
  GCObject **psurvival;
  // 'sweepgen' 的虚拟输出参数
  GCObject *dummy;
  // 断言垃圾回收状态为 GCSpropagate
  lua_assert(g->gcstate == GCSpropagate);
  // 如果存在 regular OLD1 对象
  if (g->firstold1) {
    // 标记 regular OLD1 对象
    markold(g, g->firstold1, g->reallyold);
    g->firstold1 = NULL;  /* no more OLD1 objects (for now) */
  }
  // 标记 g->finobj 和 g->finobjrold 为老对象
  markold(g, g->finobj, g->finobjrold);
  // 标记 g->tobefnz 为老对象
  markold(g, g->tobefnz, NULL);
  // 执行原子操作
  atomic(L);

  /* sweep nursery and get a pointer to its last live element */
  // 设置垃圾回收状态为 GCSswpallgc
  g->gcstate = GCSswpallgc;
  // 执行 sweepgen 函数，清扫所有的新生代对象，并获取最后一个存活对象的指针
  psurvival = sweepgen(L, g, &g->allgc, g->survival, &g->firstold1);
  /* sweep 'survival' */
  // 执行 sweepgen 函数，清扫 'survival' 列表
  sweepgen(L, g, psurvival, g->old1, &g->firstold1);
  // 设置 g->reallyold 为 g->old1
  g->reallyold = g->old1;
  // 设置 g->old1 为 *psurvival，即 'survival' 中的存活对象现在都是老对象
  g->old1 = *psurvival;  /* 'survival' survivals are old now */
  // 设置 g->survival 为 g->allgc，即所有新对象都是存活对象

  /* repeat for 'finobj' lists */
  // 设置 dummy 为 NULL，为 'finobj' 列表禁用 'firstold1' 优化
  dummy = NULL;  /* no 'firstold1' optimization for 'finobj' lists */
  // 执行 sweepgen 函数，清扫 'finobj' 列表，并获取最后一个存活对象的指针
  psurvival = sweepgen(L, g, &g->finobj, g->finobjsur, &dummy);
  /* sweep 'survival' */
  // 执行 sweepgen 函数，清扫 'survival' 列表
  sweepgen(L, g, psurvival, g->finobjold1, &dummy);
  // 设置 g->finobjrold 为 g->finobjold1
  g->finobjrold = g->finobjold1;
  // 设置 g->finobjold1 为 *psurvival，即 'survival' 中的存活对象现在都是老对象
  g->finobjold1 = *psurvival;  /* 'survival' survivals are old now */
  // 设置 g->finobjsur 为 g->finobj，即所有新对象都是存活对象

  // 执行 sweepgen 函数，清扫 'tobefnz' 列表
  sweepgen(L, g, &g->tobefnz, NULL, &dummy);
  // 执行 finishgencycle 函数，完成一轮垃圾回收
  finishgencycle(L, g);
/*
** 清除所有灰色列表，扫描对象，并准备子列表进入分代模式。扫描会移除死对象并将所有存活对象转为老对象。线程回到 'grayagain' 状态；其它所有对象都被标记为黑色（不在任何灰色列表中）。
*/
static void atomic2gen (lua_State *L, global_State *g) {
  cleargraylists(g);  // 清除灰色列表
  /* 扫描所有元素将它们标记为老对象 */
  g->gcstate = GCSswpallgc;  // 设置 GC 状态为扫描所有对象
  sweep2old(L, &g->allgc);  // 扫描所有对象并将它们标记为老对象
  /* 现在所有存活对象都是老对象 */
  g->reallyold = g->old1 = g->survival = g->allgc;
  g->firstold1 = NULL;  /* 任何地方都没有 OLD1 对象 */

  /* 对 'finobj' 列表重复相同操作 */
  sweep2old(L, &g->finobj);
  g->finobjrold = g->finobjold1 = g->finobjsur = g->finobj;

  sweep2old(L, &g->tobefnz);

  g->gckind = KGC_GEN;
  g->lastatomic = 0;
  g->GCestimate = gettotalbytes(g);  /* 内存控制的基础 */
  finishgencycle(L, g);
}


/*
** 进入分代模式。必须一直执行直到一个原子周期结束，以确保所有对象都被正确标记并且弱表被清除。然后，将所有对象标记为老对象并完成收集。
*/
static lu_mem entergen (lua_State *L, global_State *g) {
  lu_mem numobjs;
  luaC_runtilstate(L, bitmask(GCSpause));  /* 准备开始一个新周期 */
  luaC_runtilstate(L, bitmask(GCSpropagate));  /* 开始新周期 */
  numobjs = atomic(L);  /* 传播所有对象然后执行原子操作 */
  atomic2gen(L, g);
  return numobjs;
}


/*
** 进入增量模式。将所有对象标记为白色，使所有中间列表指向 NULL（避免无效指针），并进入暂停状态。
*/
static void enterinc (global_State *g) {
  whitelist(g, g->allgc);  // 将所有对象标记为白色
  g->reallyold = g->old1 = g->survival = NULL;
  whitelist(g, g->finobj);
  whitelist(g, g->tobefnz);
  g->finobjrold = g->finobjold1 = g->finobjsur = NULL;
  g->gcstate = GCSpause;
  g->gckind = KGC_INC;
  g->lastatomic = 0;
}


/*
** 将收集器模式更改为 'newmode'。
*/
/*
** 修改垃圾收集器的模式
*/
void luaC_changemode (lua_State *L, int newmode) {
  // 获取全局状态
  global_State *g = G(L);
  // 如果新模式不等于当前模式
  if (newmode != g->gckind) {
    // 如果新模式是 KGC_GEN（进入分代模式）
    if (newmode == KGC_GEN)  /* entering generational mode? */
      // 进入分代模式
      entergen(L, g);
    else
      // 否则进入增量模式
      enterinc(g);  /* entering incremental mode */
  }
  // 重置上次的原子操作计数
  g->lastatomic = 0;
}


/*
** 在分代模式下进行完整的垃圾收集
*/
static lu_mem fullgen (lua_State *L, global_State *g) {
  // 进入增量模式
  enterinc(g);
  // 返回进入分代模式的结果
  return entergen(L, g);
}


/*
** 设置下一次次要垃圾收集的负债，当内存增长到 'genminormul'% 时进行
*/
static void setminordebt (global_State *g) {
  // 设置负债
  luaE_setdebt(g, -(cast(l_mem, (gettotalbytes(g) / 100)) * g->genminormul));
}


/*
** 在上次收集是“坏收集”后进行主要收集
**
** 当程序构建一个大型结构时，它分配了大量内存，但生成了很少的垃圾。
** 在这些情况下，分代模式只会浪费时间进行小型收集，而主要收集经常被称为“坏收集”，
** 即释放的对象太少。为了避免在坏收集后在分代模式和全（主要）收集所需的增量模式之间切换的成本，
** 收集器尝试在坏收集后保持增量模式，并且仅在“好”收集（遍历少于前一个收集的 9/8 对象）后才切换回分代模式。
** 收集器必须在扫描之前选择是保持增量模式还是返回分代模式。此时，它不知道实际使用的内存，因此不能使用内存来决定是否返回分代模式。
** 相反，它使用遍历的对象数（由 'atomic' 返回）作为代理。字段 'g->lastatomic' 保留了上次收集的计数。（'g->lastatomic != 0' 也表示上次收集是坏的。）
*/
static void stepgenfull (lua_State *L, global_State *g) {
  lu_mem newatomic;  /* count of traversed objects */  // 定义变量 newatomic，用于记录遍历对象的数量
  lu_mem lastatomic = g->lastatomic;  /* count from last collection */  // 获取上一次垃圾回收时的遍历对象数量
  if (g->gckind == KGC_GEN)  /* still in generational mode? */  // 如果仍处于分代模式
    enterinc(g);  /* enter incremental mode */  // 进入增量模式
  luaC_runtilstate(L, bitmask(GCSpropagate));  /* start new cycle */  // 运行垃圾回收器，开始新的循环
  newatomic = atomic(L);  /* mark everybody */  // 标记所有对象
  if (newatomic < lastatomic + (lastatomic >> 3)) {  /* good collection? */  // 如果遍历对象数量小于上次回收数量的 1/8
    atomic2gen(L, g);  /* return to generational mode */  // 返回到分代模式
    setminordebt(g);  // 设置小型债务
  }
  else {  /* another bad collection; stay in incremental mode */  // 另一次坏的回收；保持在增量模式
    g->GCestimate = gettotalbytes(g);  /* first estimate */  // 获取内存总量的初始估计
    entersweep(L);  // 进入扫描状态
    luaC_runtilstate(L, bitmask(GCSpause));  /* finish collection */  // 运行垃圾回收器，直到暂停状态
    setpause(g);  // 设置暂停状态
    g->lastatomic = newatomic;  // 更新上次回收的遍历对象数量
  }
}

/*
** Does a generational "step".
** Usually, this means doing a minor collection and setting the debt to
** make another collection when memory grows 'genminormul'% larger.
**
** However, there are exceptions.  If memory grows 'genmajormul'%
** larger than it was at the end of the last major collection (kept
** in 'g->GCestimate'), the function does a major collection. At the
** end, it checks whether the major collection was able to free a
** decent amount of memory (at least half the growth in memory since
** previous major collection). If so, the collector keeps its state,
** and the next collection will probably be minor again. Otherwise,
** we have what we call a "bad collection". In that case, set the field
** 'g->lastatomic' to signal that fact, so that the next collection will
** go to 'stepgenfull'.
**
** 'GCdebt <= 0' means an explicit call to GC step with "size" zero;
** in that case, do a minor collection.
*/
static void genstep (lua_State *L, global_State *g) {
  if (g->lastatomic != 0)  /* last collection was a bad one? */  // 上次回收是坏的吗？
    stepgenfull(L, g);  /* do a full step */  // 进行完整的步骤
  else {
    lu_mem majorbase = g->GCestimate;  /* 记录上次主要垃圾回收后的内存 */
    lu_mem majorinc = (majorbase / 100) * getgcparam(g->genmajormul);  /* 计算主要垃圾回收的增量 */
    if (g->GCdebt > 0 && gettotalbytes(g) > majorbase + majorinc) {  /* 如果有垃圾回收债务且内存超过了上次主要回收后的值加上增量 */
      lu_mem numobjs = fullgen(L, g);  /* 进行一次主要垃圾回收 */
      if (gettotalbytes(g) < majorbase + (majorinc / 2)) {  /* 如果内存增长超过上次主要回收后的一半 */
        /* 收集了至少上次主要回收以来内存增长的一半；继续进行次要回收 */
        setminordebt(g);  /* 设置次要垃圾回收债务 */
      }
      else {  /* 垃圾回收效果不好 */
        g->lastatomic = numobjs;  /* 表示上次回收效果不好 */
        setpause(g);  /* 等待下一次（主要）垃圾回收 */
      }
    }
    else {  /* 常规情况；进行次要垃圾回收 */
      youngcollection(L, g);  /* 进行次要垃圾回收 */
      setminordebt(g);  /* 设置次要垃圾回收债务 */
      g->GCestimate = majorbase;  /* 保留基础值 */
    }
  }
  lua_assert(isdecGCmodegen(g));  /* 断言当前为减少垃圾回收模式 */
/* }====================================================== */
/* 结束标记 */

/*
** {======================================================
** GC control
** 垃圾回收控制
** =======================================================
*/

/*
** Set the "time" to wait before starting a new GC cycle; cycle will
** start when memory use hits the threshold of ('estimate' * pause /
** PAUSEADJ). (Division by 'estimate' should be OK: it cannot be zero,
** because Lua cannot even start with less than PAUSEADJ bytes).
** 设置在开始新的GC周期之前等待的“时间”；当内存使用量达到（'estimate' * pause / PAUSEADJ）的阈值时，循环将开始。
** （通过 'estimate' 的除法应该是可以的：它不可能为零，因为Lua甚至不能以少于PAUSEADJ字节开始）。
*/
static void setpause (global_State *g) {
  l_mem threshold, debt;
  int pause = getgcparam(g->gcpause);
  l_mem estimate = g->GCestimate / PAUSEADJ;  /* adjust 'estimate' */  // 调整 'estimate'
  lua_assert(estimate > 0);  // 断言 'estimate' 大于0
  threshold = (pause < MAX_LMEM / estimate)  /* overflow? */  // 溢出？
            ? estimate * pause  /* no overflow */  // 没有溢出
            : MAX_LMEM;  /* overflow; truncate to maximum */  // 溢出；截断到最大值
  debt = gettotalbytes(g) - threshold;  // 计算内存使用量与阈值之间的差值
  if (debt > 0) debt = 0;  // 如果差值大于0，则设为0
  luaE_setdebt(g, debt);  // 设置内存使用的差值
}

/*
** Enter first sweep phase.
** The call to 'sweeptolive' makes the pointer point to an object
** inside the list (instead of to the header), so that the real sweep do
** not need to skip objects created between "now" and the start of the
** real sweep.
** 进入第一次扫描阶段。
** 对 'sweeptolive' 的调用使指针指向列表中的一个对象（而不是指向头部），这样真正的扫描就不需要跳过在“现在”和真正扫描开始之间创建的对象。
*/
static void entersweep (lua_State *L) {
  global_State *g = G(L);
  g->gcstate = GCSswpallgc;  // 设置垃圾回收状态为全局扫描
  lua_assert(g->sweepgc == NULL);  // 断言全局扫描的对象为空
  g->sweepgc = sweeptolive(L, &g->allgc);  // 设置全局扫描的对象为活跃对象
}

/*
** Delete all objects in list 'p' until (but not including) object 'limit'.
** 删除列表 'p' 中的所有对象，直到（但不包括）对象 'limit'。
*/
static void deletelist (lua_State *L, GCObject *p, GCObject *limit) {
  while (p != limit) {  // 当 p 不等于 limit 时循环
    GCObject *next = p->next;  // 获取下一个对象
    freeobj(L, p);  // 释放对象
    p = next;  // 将 p 指向下一个对象
  }
}

/*
** Call all finalizers of the objects in the given Lua state, and
** then free all objects, except for the main thread.
** 调用给定Lua状态中对象的所有终结器，然后释放所有对象，除了主线程。
*/
void luaC_freeallobjects (lua_State *L) {
  global_State *g = G(L);  // 获取全局状态
  g->gcstp = GCSTPCLS;  /* no extra finalizers after here */  // 设置垃圾收集器状态，表示此后没有额外的终结器
  luaC_changemode(L, KGC_INC);  // 改变垃圾收集器模式为增量模式
  separatetobefnz(g, 1);  /* separate all objects with finalizers */  // 将所有带有终结器的对象分离出来
  lua_assert(g->finobj == NULL);  // 断言确保没有带有终结器的对象
  callallpendingfinalizers(L);  // 调用所有待处理的终结器
  deletelist(L, g->allgc, obj2gco(g->mainthread));  // 删除对象链表中的对象，从全局对象链表中删除主线程对象
  lua_assert(g->finobj == NULL);  /* no new finalizers */  // 断言确保没有新的终结器
  deletelist(L, g->fixedgc, NULL);  /* collect fixed objects */  // 删除固定对象链表中的对象，收集固定对象
  lua_assert(g->strt.nuse == 0);  // 断言确保字符串表中没有被使用的字符串
}
  // 获取全局状态
  global_State *g = G(L);
  // 初始化工作量
  lu_mem work = 0;
  // 保存原始的弱引用表和全部对象表
  GCObject *origweak, *origall;
  // 保存原始的灰色对象列表
  GCObject *grayagain = g->grayagain;  /* save original list */
  g->grayagain = NULL;
  // 断言确保弱引用表和临时对象表为空
  lua_assert(g->ephemeron == NULL && g->weak == NULL);
  // 断言确保主线程不是白色
  lua_assert(!iswhite(g->mainthread));
  // 设置垃圾回收状态为原子状态
  g->gcstate = GCSatomic;
  // 标记当前运行的线程
  markobject(g, L);  /* mark running thread */
  // 标记全局注册表
  markvalue(g, &g->l_registry);
  // 标记全局元表
  markmt(g);  /* mark global metatables */
  // 传播所有标记，清空灰色列表
  work += propagateall(g);  /* empties 'gray' list */
  // 标记可能已经死亡的线程的偶发上值
  work += remarkupvals(g);
  // 传播所有标记，传播变化
  work += propagateall(g);  /* propagate changes */
  // 恢复原始的灰色对象列表
  g->gray = grayagain;
  // 传播所有标记，遍历 'grayagain' 列表
  work += propagateall(g);  /* traverse 'grayagain' list */
  // 收敛弱引用表
  convergeephemerons(g);
  // 清除弱表中的值，然后检查终结器
  clearbyvalues(g, g->weak, NULL);
  clearbyvalues(g, g->allweak, NULL);
  // 保存原始的弱引用表和全部对象表
  origweak = g->weak; origall = g->allweak;
  // 将待终结的对象分离出来
  separatetobefnz(g, 0);  /* separate objects to be finalized */
  // 标记将要终结的对象
  work += markbeingfnz(g);  /* mark objects that will be finalized */
  // 传播所有标记，标记以传播 'resurrection'
  work += propagateall(g);  /* remark, to propagate 'resurrection' */
  // 收敛弱引用表
  convergeephemerons(g);
  // 清除弱表中的死对象
  clearbykeys(g, g->ephemeron);  /* clear keys from all ephemeron tables */
  clearbykeys(g, g->allweak);  /* clear keys from all 'allweak' tables */
  // 清除复活的弱表中的值
  clearbyvalues(g, g->weak, origweak);
  clearbyvalues(g, g->allweak, origall);
  // 清除字符串缓存
  luaS_clearcache(g);
  // 切换当前白色
  g->currentwhite = cast_byte(otherwhite(g));  /* flip current white */
  // 断言确保灰色列表为空
  lua_assert(g->gray == NULL);
  // 返回估计被 'atomic' 标记的槽位数量
  return work;  /* estimate of slots marked by 'atomic' */
}

// 执行垃圾回收的一步
static int sweepstep (lua_State *L, global_State *g,
                      int nextstate, GCObject **nextlist) {
  // 如果需要进行垃圾回收
  if (g->sweepgc) {
    # 保存旧的 GC 债务
    l_mem olddebt = g->GCdebt;
    # 定义变量 count
    int count;
    # 设置 sweepgc 指向经过一次垃圾回收后的新列表，并返回新列表的头部
    g->sweepgc = sweeplist(L, g->sweepgc, GCSWEEPMAX, &count);
    # 更新 GC 估算值
    g->GCestimate += g->GCdebt - olddebt;  /* update estimate */
    # 返回垃圾回收的对象数量
    return count;
  }
  else {  /* 进入下一个状态 */
    # 设置 gcstate 为下一个状态
    g->gcstate = nextstate;
    # 设置 sweepgc 为下一个列表
    g->sweepgc = nextlist;
    # 没有进行任何工作，返回 0
    return 0;  /* no work done */
  }
static lu_mem singlestep (lua_State *L) {
  // 获取全局状态
  global_State *g = G(L);
  // 断言垃圾收集器不可重入
  lua_assert(!g->gcstopem);  /* collector is not reentrant */
  // 设置垃圾收集器为不可紧急收集状态
  g->gcstopem = 1;  /* no emergency collections while collecting */
  // 根据垃圾收集器的状态进行不同的操作
  switch (g->gcstate) {
    // 如果是暂停状态
    case GCSpause: {
      // 重新启动垃圾收集
      restartcollection(g);
      // 设置垃圾收集状态为传播状态
      g->gcstate = GCSpropagate;
      // 设置工作量为1
      work = 1;
      break;
    }
    // 如果是传播状态
    case GCSpropagate: {
      // 如果没有灰色对象了
      if (g->gray == NULL) {  /* no more gray objects? */
        // 设置垃圾收集状态为进入原子状态
        g->gcstate = GCSenteratomic;  /* finish propagate phase */
        // 设置工作量为0
        work = 0;
      }
      else
        // 传播标记
        work = propagatemark(g);  /* traverse one gray object */
      break;
    }
    // 如果是进入原子状态
    case GCSenteratomic: {
      // 进行原子操作
      work = atomic(L);  /* work is what was traversed by 'atomic' */
      // 进入扫描阶段
      entersweep(L);
      // 估算总字节数
      g->GCestimate = gettotalbytes(g);  /* first estimate */;
      break;
    }
    // 如果是扫描所有"常规"对象状态
    case GCSswpallgc: {  /* sweep "regular" objects */
      // 执行一步扫描
      work = sweepstep(L, g, GCSswpfinobj, &g->finobj);
      break;
    }
    // 如果是扫描带有终结器的对象状态
    case GCSswpfinobj: {  /* sweep objects with finalizers */
      // 执行一步扫描
      work = sweepstep(L, g, GCSswptobefnz, &g->tobefnz);
      break;
    }
    // 如果是扫描待终结对象状态
    case GCSswptobefnz: {  /* sweep objects to be finalized */
      // 执行一步扫描
      work = sweepstep(L, g, GCSswpend, NULL);
      break;
    }
    // 如果是扫描结束状态
    case GCSswpend: {  /* finish sweeps */
      // 检查大小
      checkSizes(L, g);
      // 设置垃圾收集状态为调用终结器状态
      g->gcstate = GCScallfin;
      // 设置工作量为0
      work = 0;
      break;
    }
    // 如果是调用剩余终结器状态
    case GCScallfin: {  /* call remaining finalizers */
      // 如果有待终结对象且不是紧急模式
      if (g->tobefnz && !g->gcemergency) {
        // 设置垃圾收集器为可进行紧急收集
        g->gcstopem = 0;  /* ok collections during finalizers */
        // 运行一些剩余的终结器
        work = runafewfinalizers(L, GCFINMAX) * GCFINALIZECOST;
      }
      else {  /* 紧急模式或没有剩余终结器 */
        // 设置垃圾收集状态为暂停状态
        g->gcstate = GCSpause;  /* finish collection */
        // 设置工作量为0
        work = 0;
      }
      break;
    }
    // 默认情况下断言失败
    default: lua_assert(0); return 0;
  }
  // 设置垃圾收集器为可进行紧急收集
  g->gcstopem = 0;
  // 返回工作量
  return work;
}


/*
** 推进垃圾收集器，直到达到 'statemask' 允许的状态
*/
# 在给定状态掩码下运行 Lua 程序，直到达到指定状态
def luaC_runtilstate (lua_State *L, int statesmask):
    # 获取全局状态
    global_State *g = G(L);
    # 当全局状态的 gcstate 不满足给定状态掩码时，执行单步操作
    while (!testbit(statesmask, g->gcstate))
        singlestep(L);

# 执行基本的增量步骤
static void incstep (lua_State *L, global_State *g):
    # 避免除以 0，获取步骤倍数
    int stepmul = (getgcparam(g->gcstepmul) | 1);
    # 将债务和步骤大小从字节转换为“工作单位”，然后循环运行单步操作，直到添加了这么多工作单位或完成一个周期（暂停状态）
    l_mem debt = (g->GCdebt / WORK2MEM) * stepmul;
    l_mem stepsize = (g->gcstepsize <= log2maxs(l_mem))
                 ? ((cast(l_mem, 1) << g->gcstepsize) / WORK2MEM) * stepmul
                 : MAX_LMEM;  # 溢出；保持最大值
    do {  # 重复直到暂停或足够的“信用”（负债）
        lu_mem work = singlestep(L);  # 执行一个单步操作
        debt -= work;
    } while (debt > -stepsize && g->gcstate != GCSpause);
    if (g->gcstate == GCSpause)
        setpause(g);  # 暂停直到下一个周期
    else:
        debt = (debt / stepmul) * WORK2MEM;  # 将“工作单位”转换为字节
        luaE_setdebt(g, debt);

# 如果收集器正在运行，则执行基本的 GC 步骤
void luaC_step (lua_State *L):
    # 获取全局状态
    global_State *g = G(L);
    # 断言不处于紧急 GC 模式
    lua_assert(!g->gcemergency);
    if (gcrunning(g)):  # 正在运行？
        if(isdecGCmodegen(g)):
            genstep(L, g);
        else:
            incstep(L, g);

# 在增量模式下执行完整的收集
static void fullinc (lua_State *L, global_State *g):
    # 在运行收集之前，检查 'keepinvariant'；如果为真，则可能有一些对象标记为黑色，因此收集器必须扫描所有对象将它们转换回白色（因为白色没有改变，不会被收集）
    if (keepinvariant(g)):  # 黑色对象？
    # 执行垃圾回收，将所有对象标记为白色
    entersweep(L); /* sweep everything to turn them back to white */
    # 完成任何挂起的扫描阶段，以开始新的循环
    luaC_runtilstate(L, bitmask(GCSpause));
    # 运行直到调用终结器
    luaC_runtilstate(L, bitmask(GCScallfin));  /* run up to finalizers */
    # 在完整的垃圾回收周期后，估计值必须是正确的
    lua_assert(g->GCestimate == gettotalbytes(g));
    # 完成收集
    luaC_runtilstate(L, bitmask(GCSpause));  /* finish collection */
    # 设置暂停时间
    setpause(g);
/*
** 执行完整的垃圾回收循环；如果 'isemergency' 为真，设置一个标志以避免
** 一些操作，这些操作可能以一些意想不到的方式改变解释器状态（运行终结器和缩小一些结构）。
*/
void luaC_fullgc (lua_State *L, int isemergency) {
  global_State *g = G(L);
  lua_assert(!g->gcemergency);
  g->gcemergency = isemergency;  /* 设置标志 */
  if (g->gckind == KGC_INC)
    fullinc(L, g);
  else
    fullgen(L, g);
  g->gcemergency = 0;
}
/* }====================================================== */
```