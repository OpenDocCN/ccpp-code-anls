# `nmap\liblua\lgc.h`

```cpp
/*
** $Id: lgc.h $
** 垃圾收集器
** 请参阅 lua.h 中的版权声明
*/

#ifndef lgc_h
#define lgc_h


#include "lobject.h"
#include "lstate.h"

/*
** 可回收对象可能有三种颜色：白色，表示对象未标记；灰色，表示对象已标记，但其引用可能未标记；黑色，表示对象及其所有引用都已标记。
** 垃圾收集器在标记对象时的主要不变性是，黑色对象永远不会指向白色对象。此外，任何灰色对象必须在“灰色列表”（gray, grayagain, weak, allweak, ephemeron）中，以便在完成收集周期之前可以再次访问它们。（开放的上值是此规则的例外。）这些列表在不执行不变性时（例如，扫描阶段）没有意义。
*/


/*
** 垃圾收集器的可能状态
*/
#define GCSpropagate    0
#define GCSenteratomic    1
#define GCSatomic    2
#define GCSswpallgc    3
#define GCSswpfinobj    4
#define GCSswptobefnz    5
#define GCSswpend    6
#define GCScallfin    7
#define GCSpause    8


#define issweepphase(g)  \
    (GCSswpallgc <= (g)->gcstate && (g)->gcstate <= GCSswpend)


/*
** 用于判断何时必须保持主要不变性（白色对象不能指向黑色对象）。在收集期间，扫描阶段可能会破坏不变性，因为变为白色的对象可能指向仍然是黑色的对象。当扫描结束并且所有对象再次变为白色时，不变性将被恢复。
*/

#define keepinvariant(g)    ((g)->gcstate <= GCSatomic)


/*
** 一些有用的位操作技巧
*/
#define resetbits(x,m)        ((x) &= cast_byte(~(m)))
#define setbits(x,m)        ((x) |= (m))
#define testbits(x,m)        ((x) & (m))
#define bitmask(b)        (1<<(b))
#define bit2mask(b1,b2)        (bitmask(b1) | bitmask(b2))
#define l_setbit(x,b)        setbits(x, bitmask(b))
#define resetbit(x,b)        resetbits(x, bitmask(b))
# 定义一个宏，用于测试指定位是否被设置
#define testbit(x,b)        testbits(x, bitmask(b))

# 定义一些常量，表示对象在'marked'字段中的位布局
# 前三位用于对象在分代模式下的"age"，最后一位用于测试
#define WHITE0BIT    3  # 对象是白色（类型0）
#define WHITE1BIT    4  # 对象是白色（类型1）
#define BLACKBIT    5  # 对象是黑色
#define FINALIZEDBIT    6  # 对象已标记为最终化

# 用于测试位是否被设置
#define TESTBIT        7

# 定义一个常量，表示白色位的组合
#define WHITEBITS    bit2mask(WHITE0BIT, WHITE1BIT)

# 定义宏，用于检查对象是否为白色、黑色、灰色、最终化
#define iswhite(x)      testbits((x)->marked, WHITEBITS)
#define isblack(x)      testbit((x)->marked, BLACKBIT)
#define isgray(x)  /* 既不是白色也不是黑色 */  \
    (!testbits((x)->marked, WHITEBITS | bitmask(BLACKBIT)))
#define tofinalize(x)    testbit((x)->marked, FINALIZEDBIT)

# 定义宏，用于获取另一种白色
#define otherwhite(g)    ((g)->currentwhite ^ WHITEBITS)
# 定义宏，用于检查对象是否为死亡
#define isdeadm(ow,m)    ((m) & (ow))
#define isdead(g,v)    isdeadm(otherwhite(g), (v)->marked)

# 定义宏，用于改变对象的颜色为白色
#define changewhite(x)    ((x)->marked ^= WHITEBITS)
# 定义宏，用于将新白色转换为黑色
#define nw2black(x)  \
    check_exp(!iswhite(x), l_setbit((x)->marked, BLACKBIT))

# 定义宏，用于获取Lua垃圾回收器的白色
#define luaC_white(g)    cast_byte((g)->currentwhite & WHITEBITS)

# 对象在分代模式下的年龄
#define G_NEW        0    # 在当前周期创建
#define G_SURVIVAL    1    # 在上一个周期创建
#define G_OLD0        2    # 在本周期由前向后屏障标记为老对象
#define G_OLD1        3    # 第一个完整周期作为老对象
#define G_OLD        4    # 真正的老对象（不会被访问）
#define G_TOUCHED1    5    # 老对象在本周期被访问
#define G_TOUCHED2    6    # 老对象在上一个周期被访问

# 所有年龄位的常量（111）
#define AGEBITS        7

# 宏，用于获取对象的年龄
#define getage(o)    ((o)->marked & AGEBITS)
# 宏，用于设置对象的年龄
#define setage(o,a)  ((o)->marked = cast_byte(((o)->marked & (~AGEBITS)) | a))
# 宏，用于检查对象是否为老对象
#define isold(o)    (getage(o) > G_SURVIVAL)

# 宏，用于改变对象的年龄
#define changeage(o,f,t)  \
    check_exp(getage(o) == (f), (o)->marked ^= ((f)^(t))

# 垃圾回收参数的默认值
/* 定义主要垃圾回收的倍数 */
#define LUAI_GENMAJORMUL         100
/* 定义次要垃圾回收的倍数 */
#define LUAI_GENMINORMUL         20

/* 在开始新周期之前等待内存翻倍 */
#define LUAI_GCPAUSE    200

/*
** 一些垃圾回收参数被除以4存储，以允许'lu_byte'中的最大值为1023。
*/
#define getgcparam(p)    ((p) * 4)
#define setgcparam(p,v)    ((p) = (v) / 4)

#define LUAI_GCMUL      100

/* 下一次GC步骤之前分配多少内存（log2） */
#define LUAI_GCSTEPSIZE 13      /* 8 KB */


/*
** 检查声明的GC模式是否是分代的。在分代模式下，收集器可以暂时进入增量模式以提高性能。这由'g->lastatomic != 0'表示。
*/
#define isdecGCmodegen(g)    (g->gckind == KGC_GEN || g->lastatomic != 0)


/*
** 控制GC何时运行：
*/
#define GCSTPUSR    1  /* 当用户停止GC时为真 */
#define GCSTPGC        2  /* 当GC自行停止时为真 */
#define GCSTPCLS    4  /* 当关闭Lua状态时为真 */
#define gcrunning(g)    ((g)->gcstp == 0)


/*
** 当债务变得正数时执行一步收集。'pre'/'pos'允许仅在需要时进行一些调整。宏'condchangemem'仅用于重型测试（在每个机会上强制进行完整的GC周期）
*/
#define luaC_condGC(L,pre,pos) \
    { if (G(L)->GCdebt > 0) { pre; luaC_step(L); pos;}; \
      condchangemem(L,pre,pos); }

/* 大多数情况下，'pre'/'pos'为空 */
#define luaC_checkGC(L)        luaC_condGC(L,(void)0,(void)0)


#define luaC_barrier(L,p,v) (  \
    (iscollectable(v) && isblack(p) && iswhite(gcvalue(v))) ?  \
    luaC_barrier_(L,obj2gco(p),gcvalue(v)) : cast_void(0))

#define luaC_barrierback(L,p,v) (  \
    (iscollectable(v) && isblack(p) && iswhite(gcvalue(v))) ? \
    luaC_barrierback_(L,p) : cast_void(0))

#define luaC_objbarrier(L,p,o) (  \
    (isblack(p) && iswhite(o)) ? \
    luaC_barrier_(L,obj2gco(p),obj2gco(o)) : cast_void(0))

LUAI_FUNC void luaC_fix (lua_State *L, GCObject *o);
// 释放 Lua 状态机 L 中的所有对象
LUAI_FUNC void luaC_freeallobjects (lua_State *L);

// 执行一步垃圾回收
LUAI_FUNC void luaC_step (lua_State *L);

// 运行垃圾回收直到状态机 L 的状态满足 statesmask 指定的条件
LUAI_FUNC void luaC_runtilstate (lua_State *L, int statesmask);

// 执行一次完整的垃圾回收
LUAI_FUNC void luaC_fullgc (lua_State *L, int isemergency);

// 在状态机 L 中创建一个新的对象，类型为 tt，大小为 sz
LUAI_FUNC GCObject *luaC_newobj (lua_State *L, int tt, size_t sz);

// 在状态机 L 中设置对象 o 的写屏障，当对象 v 被写入时触发
LUAI_FUNC void luaC_barrier_ (lua_State *L, GCObject *o, GCObject *v);

// 在状态机 L 中设置对象 o 的后向写屏障
LUAI_FUNC void luaC_barrierback_ (lua_State *L, GCObject *o);

// 检查对象 o 是否有 finalizer，如果有则执行
LUAI_FUNC void luaC_checkfinalizer (lua_State *L, GCObject *o, Table *mt);

// 改变状态机 L 的垃圾回收模式为 newmode
LUAI_FUNC void luaC_changemode (lua_State *L, int newmode);
```