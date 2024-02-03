# `nmap\liblua\ltable.c`

```cpp
/*
** $Id: ltable.c $
** Lua tables (hash)
** See Copyright Notice in lua.h
*/

// 定义 ltable.c，表示这是 Lua 中的表（哈希表）实现文件
#define ltable_c
// 定义 LUA_CORE，表示这是 Lua 核心模块
#define LUA_CORE

// 包含 lua.h 中的版权声明
#include "lprefix.h"


/*
** Implementation of tables (aka arrays, objects, or hash tables).
** Tables keep its elements in two parts: an array part and a hash part.
** Non-negative integer keys are all candidates to be kept in the array
** part. The actual size of the array is the largest 'n' such that
** more than half the slots between 1 and n are in use.
** Hash uses a mix of chained scatter table with Brent's variation.
** A main invariant of these tables is that, if an element is not
** in its main position (i.e. the 'original' position that its hash gives
** to it), then the colliding element is in its own main position.
** Hence even when the load factor reaches 100%, performance remains good.
*/

// 包含一些头文件
#include <math.h>
#include <limits.h>

#include "lua.h"

#include "ldebug.h"
#include "ldo.h"
#include "lgc.h"
#include "lmem.h"
#include "lobject.h"
#include "lstate.h"
#include "lstring.h"
#include "ltable.h"
#include "lvm.h"


/*
** MAXABITS is the largest integer such that MAXASIZE fits in an
** unsigned int.
*/
// 定义 MAXABITS 为一个整数，使得 MAXASIZE 能够适应无符号整数
#define MAXABITS    cast_int(sizeof(int) * CHAR_BIT - 1)


/*
** MAXASIZE is the maximum size of the array part. It is the minimum
** between 2^MAXABITS and the maximum size that, measured in bytes,
** fits in a 'size_t'.
*/
// 定义 MAXASIZE 为数组部分的最大大小，是 2^MAXABITS 和能够适应 size_t 类型的最大大小的最小值
#define MAXASIZE    luaM_limitN(1u << MAXABITS, TValue)

/*
** MAXHBITS is the largest integer such that 2^MAXHBITS fits in a
** signed int.
*/
// 定义 MAXHBITS 为一个整数，使得 2^MAXHBITS 能够适应有符号整数
#define MAXHBITS    (MAXABITS - 1)


/*
** MAXHSIZE is the maximum size of the hash part. It is the minimum
** between 2^MAXHBITS and the maximum size such that, measured in bytes,
** it fits in a 'size_t'.
*/
// 定义 MAXHSIZE 为哈希部分的最大大小，是 2^MAXHBITS 和能够适应 size_t 类型的最大大小的最小值
#define MAXHSIZE    luaM_limitN(1u << MAXHBITS, Node)


/*
** When the original hash value is good, hashing by a power of 2
** avoids the cost of '%'.
*/
// 如果原始哈希值很好，通过 2 的幂次方进行哈希可以避免 '%' 运算的开销
#define hashpow2(t,n)        (gnode(t, lmod((n), sizenode(t))))
/* 
** 对于其他类型，最好避免使用2的幂进行取模运算，因为它们可能有很多2因子。
*/
#define hashmod(t,n)    (gnode(t, ((n) % ((sizenode(t)-1)|1)))

/*
** 根据字符串的哈希值进行取模运算
*/
#define hashstr(t,str)        hashpow2(t, (str)->hash)

/*
** 根据布尔值的哈希值进行取模运算
*/
#define hashboolean(t,p)    hashpow2(t, p)

/*
** 根据指针的哈希值进行取模运算
*/
#define hashpointer(t,p)    hashmod(t, point2uint(p))

/*
** 定义一个虚拟节点
*/
#define dummynode        (&dummynode_)

/*
** 定义一个虚拟节点的结构体
*/
static const Node dummynode_ = {
  {{NULL}, LUA_VEMPTY,  /* value's value and type */
   LUA_VNIL, 0, {NULL}}  /* key type, next, and key value */
};

/*
** 整数的哈希函数。为了获得一个好的哈希值，使用取余运算符（'%'）。
** 如果整数适合作为非负整数，计算一个整数余数，这样更快。
** 否则，使用无符号整数余数，它使用所有位并确保得到一个非负结果。
*/
static Node *hashint (const Table *t, lua_Integer i) {
  lua_Unsigned ui = l_castS2U(i);
  if (ui <= (unsigned int)INT_MAX)
    return hashmod(t, cast_int(ui));
  else
    return hashmod(t, ui);
}

/*
** 浮点数的哈希函数。
** 主要计算应该是
**     n = frexp(n, &i); return (n * INT_MAX) + i
** 但是有一些数值上的微妙之处。
** 在二进制补码表示中，INT_MAX没有一个精确的浮点表示，但INT_MIN有；
** 因为'frexp'的绝对值小于1（除非'n'是inf/NaN），'frexp * -INT_MIN'的绝对值小于或等于INT_MAX。
** 接下来，使用'unsigned int'避免在添加'i'时溢出；使用'~u'（而不是'-u'）避免INT_MIN的问题。
*/
#if !defined(l_hashfloat)
static int l_hashfloat (lua_Number n) {
  int i;
  lua_Integer ni;
  n = l_mathop(frexp)(n, &i) * -cast_num(INT_MIN);
  if (!lua_numbertointeger(n, &ni)) {  /* 'n'是否为inf/-inf/NaN？ */
    lua_assert(luai_numisnan(n) || l_mathop(fabs)(n) == cast_num(HUGE_VAL));
    return 0;
  }
  else {  /* 正常情况 */
    # 将变量 i 和 ni 转换为无符号整数并相加
    unsigned int u = cast_uint(i) + cast_uint(ni);
    # 如果 u 小于等于 INT_MAX，则将 u 转换为有符号整数返回，否则返回 ~u（按位取反）
    return cast_int(u <= cast_uint(INT_MAX) ? u : ~u);
  }
}
#endif


/*
** 返回表中元素的“主”位置（即其哈希值的索引）。
*/
static Node *mainpositionTV (const Table *t, const TValue *key) {
  switch (ttypetag(key)) {
    case LUA_VNUMINT: {
      // 如果键是整数，返回整数的哈希位置
      lua_Integer i = ivalue(key);
      return hashint(t, i);
    }
    case LUA_VNUMFLT: {
      // 如果键是浮点数，返回浮点数的哈希位置
      lua_Number n = fltvalue(key);
      return hashmod(t, l_hashfloat(n));
    }
    case LUA_VSHRSTR: {
      // 如果键是短字符串，返回短字符串的哈希位置
      TString *ts = tsvalue(key);
      return hashstr(t, ts);
    }
    case LUA_VLNGSTR: {
      // 如果键是长字符串，返回长字符串的哈希位置
      TString *ts = tsvalue(key);
      return hashpow2(t, luaS_hashlongstr(ts));
    }
    case LUA_VFALSE:
      // 如果键是假值，返回假值的哈希位置
      return hashboolean(t, 0);
    case LUA_VTRUE:
      // 如果键是真值，返回真值的哈希位置
      return hashboolean(t, 1);
    case LUA_VLIGHTUSERDATA: {
      // 如果键是轻量用户数据，返回用户数据的哈希位置
      void *p = pvalue(key);
      return hashpointer(t, p);
    }
    case LUA_VLCF: {
      // 如果键是 C 函数，返回 C 函数的哈希位置
      lua_CFunction f = fvalue(key);
      return hashpointer(t, f);
    }
    default: {
      // 如果键是其他类型的对象，返回对象的哈希位置
      GCObject *o = gcvalue(key);
      return hashpointer(t, o);
    }
  }
}


l_sinline Node *mainpositionfromnode (const Table *t, Node *nd) {
  TValue key;
  getnodekey(cast(lua_State *, NULL), &key, nd);
  return mainpositionTV(t, &key);
}


/*
** 检查键 'k1' 是否等于节点 'n2' 中的键。这种相等性是原始的，所以没有元方法。
** 具有整数值的浮点数已被标准化，因此整数不能等于浮点数。假定 'eqshrstr' 只是指针相等性，
** 因此短字符串在默认情况下处理。
** 真正的 'deadok' 意味着接受死键等于其原始值。所有死键在默认情况下都进行比较，通过指针标识。
** （只有可收集的对象才能产生死键。）请注意，死长字符串也通过标识进行比较。
** 一旦键死亡，其对应的值可能被收集，然后可以使用相同地址创建另一个值。如果将此其他值传递给 'next'，
** 'equalkey' 将发出错误
*/
/*
** positive. In a regular traversal, this situation should never happen,
** as all keys given to 'next' came from the table itself, and therefore
** could not have been collected. Outside a regular traversal, we
** have garbage in, garbage out. What is relevant is that this false
** positive does not break anything.  (In particular, 'next' will return
** some other valid item on the table or nil.)
*/
static int equalkey (const TValue *k1, const Node *n2, int deadok) {
  if ((rawtt(k1) != keytt(n2)) &&  /* not the same variants? */
       !(deadok && keyisdead(n2) && iscollectable(k1)))
   return 0;  /* cannot be same key */
  switch (keytt(n2)) {
    case LUA_VNIL: case LUA_VFALSE: case LUA_VTRUE:
      return 1;
    case LUA_VNUMINT:
      return (ivalue(k1) == keyival(n2));
    case LUA_VNUMFLT:
      return luai_numeq(fltvalue(k1), fltvalueraw(keyval(n2)));
    case LUA_VLIGHTUSERDATA:
      return pvalue(k1) == pvalueraw(keyval(n2));
    case LUA_VLCF:
      return fvalue(k1) == fvalueraw(keyval(n2));
    case ctb(LUA_VLNGSTR):
      return luaS_eqlngstr(tsvalue(k1), keystrval(n2));
    default:
      return gcvalue(k1) == gcvalueraw(keyval(n2));
  }
}


/*
** True if value of 'alimit' is equal to the real size of the array
** part of table 't'. (Otherwise, the array part must be larger than
** 'alimit'.)
*/
#define limitequalsasize(t)    (isrealasize(t) || ispow2((t)->alimit))


/*
** Returns the real size of the 'array' array
*/
LUAI_FUNC unsigned int luaH_realasize (const Table *t) {
  if (limitequalsasize(t))
    return t->alimit;  /* this is the size */
  else {
    unsigned int size = t->alimit;
    /* compute the smallest power of 2 not smaller than 'n' */
    size |= (size >> 1);
    size |= (size >> 2);
    size |= (size >> 4);
    size |= (size >> 8);
    size |= (size >> 16);
#if (UINT_MAX >> 30) > 3
    size |= (size >> 32);  /* unsigned int has more than 32 bits */
#endif
    size++;
    # 使用lua_assert函数来确保size是2的幂，并且size/2小于t->alimit并且t->alimit小于size
    lua_assert(ispow2(size) && size/2 < t->alimit && t->alimit < size);
    # 返回size的值
    return size;
  }
# 检查数组的实际大小是否是2的幂。
# （如果不是，'alimit' 不能在不改变实际大小的情况下更改为其他值。）
static int ispow2realasize (const Table *t) {
  return (!isrealasize(t) || ispow2(t->alimit));
}

# 将 'alimit' 设置为表的实际大小
static unsigned int setlimittosize (Table *t) {
  t->alimit = luaH_realasize(t);
  setrealasize(t);
  return t->alimit;
}

# 如果表的实际大小是真实大小，则将 'alimit' 作为真实大小返回
#define limitasasize(t)    check_exp(isrealasize(t), t->alimit)

# "通用" 获取版本。（并非真正通用：对于整数来说无效，因为它们可能在数组部分，对于具有整数值的浮点数也是如此。）
# 有关 'deadok' 在函数 'equalkey' 中的解释，请参见说明。
static const TValue *getgeneric (Table *t, const TValue *key, int deadok) {
  Node *n = mainpositionTV(t, key);
  for (;;) {  /* 检查 'key' 是否在链中的某处 */
    if (equalkey(key, n, deadok))
      return gval(n);  /* 找到了 */
    else {
      int nx = gnext(n);
      if (nx == 0)
        return &absentkey;  /* 未找到 */
      n += nx;
    }
  }
}

# 如果 'k' 是适合存储在表的数组部分的键，则返回 'k' 的索引，否则返回0。
static unsigned int arrayindex (lua_Integer k) {
  if (l_castS2U(k) - 1u < MAXASIZE)  /* 'k' 在 [1, MAXASIZE] 中？ */
    return cast_uint(k);  /* 'key' 是适合的数组索引 */
  else
    return 0;
}

# 返回表遍历的 'key' 的索引。首先遍历数组部分的所有元素，然后遍历哈希部分的元素。
# 遍历的开始由0表示。
static unsigned int findindex (lua_State *L, Table *t, TValue *key,
                               unsigned int asize) {
  unsigned int i;
  if (ttisnil(key)) return 0;  /* 第一次迭代 */
  i = ttisinteger(key) ? arrayindex(ivalue(key)) : 0;
  if (i - 1u < asize)  /* 'key' 是否在数组部分内？ */
    return i;  /* 是的；这就是索引 */
  else {
    const TValue *n = getgeneric(t, key, 1);
    # 如果给定的键是抽象键，则执行以下代码
    if (l_unlikely(isabstkey(n)))
      # 抛出运行时错误，指示给定键无效
      luaG_runerror(L, "invalid key to 'next'");  /* key not found */
    # 计算给定键在哈希表中的索引
    i = cast_int(nodefromval(n) - gnode(t, 0);  /* key index in hash table */
    # 返回键在数组元素之后的哈希元素中的索引
    # 并加上数组的大小，得到最终的索引值
    return (i + 1) + asize;
  }
}

int luaH_next (lua_State *L, Table *t, StkId key) {
  // 计算实际数组大小
  unsigned int asize = luaH_realasize(t);
  // 查找原始键值
  unsigned int i = findindex(L, t, s2v(key), asize);  /* find original key */
  // 遍历数组部分
  for (; i < asize; i++) {  /* try first array part */
    // 如果数组元素非空
    if (!isempty(&t->array[i])) {  /* a non-empty entry? */
      // 设置键值为 i+1
      setivalue(s2v(key), i + 1);
      // 设置下一个键值对应的值
      setobj2s(L, key + 1, &t->array[i]);
      return 1;
    }
  }
  // 遍历哈希部分
  for (i -= asize; cast_int(i) < sizenode(t); i++) {  /* hash part */
    // 如果哈希元素非空
    if (!isempty(gval(gnode(t, i)))) {  /* a non-empty entry? */
      // 获取节点的键值
      Node *n = gnode(t, i);
      getnodekey(L, s2v(key), n);
      // 设置下一个键值对应的值
      setobj2s(L, key + 1, gval(n));
      return 1;
    }
  }
  return 0;  /* no more elements */
}

static void freehash (lua_State *L, Table *t) {
  // 如果不是虚拟表，则释放哈希表
  if (!isdummy(t))
    luaM_freearray(L, t->node, cast_sizet(sizenode(t)));
}

/*
** {=============================================================
** Rehash
** ==============================================================
*/

/*
** 计算表 't' 的数组部分的最佳大小。'nums' 是一个“计数数组”，其中 'nums[i]' 是表中在 2^(i - 1) + 1 和 2^i 之间的整数的数量。
** 'pna' 输入为表中整数键的总数，输出为将进入数组部分的键的数量；返回最佳大小。
** （for 循环中的 'twotoi > 0' 条件在 'twotoi' 溢出时停止循环。）
*/
static unsigned int computesizes (unsigned int nums[], unsigned int *pna) {
  int i;
  unsigned int twotoi;  /* 2^i (candidate for optimal size) */
  unsigned int a = 0;  /* number of elements smaller than 2^i */
  unsigned int na = 0;  /* number of elements to go to array part */
  unsigned int optimal = 0;  /* optimal size for array part */
  /* 当键可以填满总大小的一半以上时循环 */
  for (i = 0, twotoi = 1;
       twotoi > 0 && *pna > twotoi / 2;
       i++, twotoi *= 2) {
    a += nums[i];
    # 如果元素数量大于二分之一的2的幂次方
    if (a > twotoi/2) {  /* more than half elements present? */
      # 将 optimal 设为当前的2的幂次方
      optimal = twotoi;  /* optimal size (till now) */
      # 将 na 设为当前元素数量
      na = a;  /* all elements up to 'optimal' will go to array part */
    }
  }
  # 断言 optimal 为0或者optimal的二分之一小于na，并且na小于等于optimal
  lua_assert((optimal == 0 || optimal / 2 < na) && na <= optimal);
  # 将na的值赋给*pna
  *pna = na;
  # 返回optimal的值
  return optimal;
# 计算整数键在哈希表中的使用情况，更新 nums 数组
static int countint (lua_Integer key, unsigned int *nums) {
  # 将 lua_Integer 转换为适当的数组索引
  unsigned int k = arrayindex(key);
  # 如果 k 不为 0，表示 key 是一个合适的数组索引
  if (k != 0) {  /* is 'key' an appropriate array index? */
    # 更新 nums 数组中对应索引的值
    nums[luaO_ceillog2(k)]++;  /* count as such */
    # 返回 1，表示找到了合适的数组索引
    return 1;
  }
  else
    # 返回 0，表示未找到合适的数组索引
    return 0;
}


'''
** 计算表 't' 的数组部分中的键数：将 'nums[i]' 填充为相应切片中的键数，并返回非空键的总数。
'''
static unsigned int numusearray (const Table *t, unsigned int *nums) {
  int lg;
  unsigned int ttlg;  /* 2^lg */
  unsigned int ause = 0;  /* 'nums' 的总和 */
  unsigned int i = 1;  /* 用于遍历所有数组键的计数 */
  unsigned int asize = limitasasize(t);  /* 实际数组大小 */
  /* 遍历每个切片 */
  for (lg = 0, ttlg = 1; lg <= MAXABITS; lg++, ttlg *= 2) {
    unsigned int lc = 0;  /* 计数器 */
    unsigned int lim = ttlg;
    if (lim > asize) {
      lim = asize;  /* 调整上限 */
      if (i > lim)
        break;  /* 没有更多元素可计数 */
    }
    /* 计算范围 (2^(lg - 1), 2^lg] 中的元素数 */
    for (; i <= lim; i++) {
      if (!isempty(&t->array[i-1]))
        lc++;
    }
    nums[lg] += lc;
    ause += lc;
  }
  return ause;
}


# 计算哈希表中的键的使用情况，更新 nums 数组，并返回总的键数
static int numusehash (const Table *t, unsigned int *nums, unsigned int *pna) {
  int totaluse = 0;  /* 元素的总数 */
  int ause = 0;  /* 添加到 'nums' 中的元素数（可以放入数组部分） */
  int i = sizenode(t);
  while (i--) {
    Node *n = &t->node[i];
    if (!isempty(gval(n))) {
      if (keyisinteger(n))
        ause += countint(keyival(n), nums);
      totaluse++;
    }
  }
  *pna += ause;
  return totaluse;
}


'''
** 为给定大小创建哈希表的数组部分，或者如果大小为零则重用虚拟节点。
** 大小溢出的计算分为两步：第一步的比较确保第二步的移位不会溢出。
'''
/*
** 设置表 't' 的节点向量，大小为 'size'
*/
static void setnodevector (lua_State *L, Table *t, unsigned int size) {
  if (size == 0) {  /* 没有元素需要哈希部分？ */
    t->node = cast(Node *, dummynode);  /* 使用公共的 'dummynode' */
    t->lsizenode = 0;
    t->lastfree = NULL;  /* 表示正在使用虚拟节点 */
  }
  else {
    int i;
    int lsize = luaO_ceillog2(size);
    if (lsize > MAXHBITS || (1u << lsize) > MAXHSIZE)
      luaG_runerror(L, "table overflow");
    size = twoto(lsize);
    t->node = luaM_newvector(L, size, Node);
    for (i = 0; i < (int)size; i++) {
      Node *n = gnode(t, i);
      gnext(n) = 0;
      setnilkey(n);
      setempty(gval(n));
    }
    t->lsizenode = cast_byte(lsize);
    t->lastfree = gnode(t, size);  /* 所有位置都是空闲的 */
  }
}


/*
** (Re)insert all elements from the hash part of 'ot' into table 't'.
*/
static void reinsert (lua_State *L, Table *ot, Table *t) {
  int j;
  int size = sizenode(ot);
  for (j = 0; j < size; j++) {
    Node *old = gnode(ot, j);
    if (!isempty(gval(old))) {
      /* 不需要屏障/使缓存失效，因为条目已经存在于表中 */
      TValue k;
      getnodekey(L, &k, old);
      luaH_set(L, t, &k, gval(old));
    }
  }
}


/*
** 交换 't1' 和 't2' 的哈希部分。
*/
static void exchangehashpart (Table *t1, Table *t2) {
  lu_byte lsizenode = t1->lsizenode;
  Node *node = t1->node;
  Node *lastfree = t1->lastfree;
  t1->lsizenode = t2->lsizenode;
  t1->node = t2->node;
  t1->lastfree = t2->lastfree;
  t2->lsizenode = lsizenode;
  t2->node = node;
  t2->lastfree = lastfree;
}


/*
** 为给定的大小调整表 't' 的大小。两个分配（哈希部分和数组部分）都可能失败，这会产生一些微妙之处。如果第一个分配，即哈希部分的分配失败，将引发错误，然后结束。否则，它将从数组部分的缩小部分复制元素（如果正在缩小）
*/
/*
** 根据新的数组大小和哈希表大小重新调整表 t 的大小
*/
void luaH_resize (lua_State *L, Table *t, unsigned int newasize,
                                          unsigned int nhsize) {
  unsigned int i;
  Table newt;  /* 用于保存新的哈希部分 */
  unsigned int oldasize = setlimittosize(t);
  TValue *newarray;
  /* 创建具有适当大小的新哈希部分到 'newt' 中 */
  setnodevector(L, &newt, nhsize);
  if (newasize < oldasize) {  /* 数组会缩小吗？ */
    t->alimit = newasize;  /* 假装数组有新的大小... */
    exchangehashpart(t, &newt);  /* 以及新的哈希部分 */
    /* 将消失的切片中的元素重新插入到新哈希中 */
    for (i = newasize; i < oldasize; i++) {
      if (!isempty(&t->array[i]))
        luaH_setint(L, t, i + 1, &t->array[i]);
    }
    t->alimit = oldasize;  /* 恢复当前大小... */
    exchangehashpart(t, &newt);  /* 以及哈希（如果出现错误） */
  }
  /* 分配新数组 */
  newarray = luaM_reallocvector(L, t->array, oldasize, newasize, TValue);
  if (l_unlikely(newarray == NULL && newasize > 0)) {  /* 分配失败？ */
    freehash(L, &newt);  /* 释放新的哈希部分 */
    luaM_error(L);  /* 调用 Lua 中的错误处理函数，抛出错误（数组不变） */
  }
  /* 分配成功；初始化数组的新部分 */
  exchangehashpart(t, &newt);  /* 交换哈希表的部分，将新的哈希表赋给 't'（旧的哈希表赋给 'newt'） */
  t->array = newarray;  /* 设置新的数组部分 */
  t->alimit = newasize;  /* 设置新的数组大小上限 */
  for (i = oldasize; i < newasize; i++)  /* 清空数组的新切片 */
     setempty(&t->array[i]);  /* 将数组的新切片置为空 */
  /* 将旧哈希表中的元素重新插入到新的部分中 */
  reinsert(L, &newt, t);  /* 将旧哈希表中的元素重新插入到新的部分中，'newt' 现在包含旧的哈希表 */
  freehash(L, &newt);  /* 释放旧的哈希表部分 */
/*
** luaH_resizearray函数的作用是重新调整数组部分的大小
** 参数L是Lua状态，t是表，nasize是新的数组部分大小
*/
void luaH_resizearray (lua_State *L, Table *t, unsigned int nasize) {
  int nsize = allocsizenode(t);  /* 计算节点分配大小 */
  luaH_resize(L, t, nasize, nsize);  /* 调整表的大小 */
}

/*
** rehash函数的作用是重新计算哈希表的大小
** 参数L是Lua状态，t是表，ek是额外的键
*/
static void rehash (lua_State *L, Table *t, const TValue *ek) {
  unsigned int asize;  /* 数组部分的最佳大小 */
  unsigned int na;  /* 数组部分中的键的数量 */
  unsigned int nums[MAXABITS + 1];  /* 存储每个区间的键的数量 */
  int i;
  int totaluse;
  for (i = 0; i <= MAXABITS; i++) nums[i] = 0;  /* 重置计数 */
  setlimittosize(t);  /* 设置表的大小限制 */
  na = numusearray(t, nums);  /* 计算数组部分中的键的数量 */
  totaluse = na;  /* 所有这些键都是整数键 */
  totaluse += numusehash(t, nums, &na);  /* 计算哈希部分中的键的数量 */
  /* 计算额外的键 */
  if (ttisinteger(ek))
    na += countint(ivalue(ek), nums);
  totaluse++;
  /* 计算数组部分的新大小 */
  asize = computesizes(nums, &na);
  /* 调整表的大小为新计算的大小 */
  luaH_resize(L, t, asize, totaluse - na);
}

/*
** luaH_new函数的作用是创建一个新的表
** 参数L是Lua状态
*/
Table *luaH_new (lua_State *L) {
  GCObject *o = luaC_newobj(L, LUA_VTABLE, sizeof(Table));  /* 创建一个新的对象 */
  Table *t = gco2t(o);  /* 将对象转换为表 */
  t->metatable = NULL;  /* 元表为空 */
  t->flags = cast_byte(maskflags);  /* 表没有元方法字段 */
  t->array = NULL;  /* 数组部分为空 */
  t->alimit = 0;  /* 数组部分的大小限制为0 */
  setnodevector(L, t, 0);  /* 设置节点向量 */
  return t;  /* 返回新创建的表 */
}

/*
** luaH_free函数的作用是释放表的内存
** 参数L是Lua状态，t是表
*/
void luaH_free (lua_State *L, Table *t) {
  freehash(L, t);  /* 释放哈希部分的内存 */
  luaM_freearray(L, t->array, luaH_realasize(t));  /* 释放数组部分的内存 */
  luaM_free(L, t);  /* 释放表的内存 */
}

/*
** getfreepos函数的作用是获取一个空闲位置的节点
** 参数t是表
*/
static Node *getfreepos (Table *t) {
  if (!isdummy(t)) {
    while (t->lastfree > t->node) {
      t->lastfree--;
      if (keyisnil(t->lastfree))
        return t->lastfree;
    }
  }
  return NULL;  /* 找不到空闲位置 */
}

/*
** insert函数的作用是将一个新的键插入哈希表中
** 首先，检查键的主位置是否为空。如果不是，则检查碰撞节点是否在其主位置上
** 如果不是，则将碰撞节点移动到一个空位置
*/
/*
** 将新键放在其主位置；否则（冲突节点在其主位置），新键放在空位置。
*/
void luaH_newkey (lua_State *L, Table *t, const TValue *key, TValue *value) {
  Node *mp;  // 指向主位置的节点指针
  TValue aux;  // 辅助值
  if (l_unlikely(ttisnil(key)))  // 如果键为nil
    luaG_runerror(L, "table index is nil");  // 抛出错误
  else if (ttisfloat(key)) {  // 如果键为浮点数
    lua_Number f = fltvalue(key);  // 获取浮点数值
    lua_Integer k;  // 整数值
    if (luaV_flttointeger(f, &k, F2Ieq)) {  /* key是否适合转换为整数？ */
      setivalue(&aux, k);  // 将整数值存入aux
      key = &aux;  // 将整数值作为键
    }
    else if (l_unlikely(luai_numisnan(f)))  // 如果键为NaN
      luaG_runerror(L, "table index is NaN");  // 抛出错误
  }
  if (ttisnil(value))
    return;  // 不插入nil值
  mp = mainpositionTV(t, key);  // 获取键的主位置
  if (!isempty(gval(mp)) || isdummy(t)) {  /* 主位置被占用？ */
    Node *othern;  // 其他位置的节点指针
    Node *f = getfreepos(t);  /* 获取一个空闲位置 */
    if (f == NULL) {  /* 找不到空闲位置？ */
      rehash(L, t, key);  /* 扩大表 */
      /* 调用'newkey'的函数负责TM缓存 */
      luaH_set(L, t, key, value);  /* 在扩大的表中插入键 */
      return;
    }
    lua_assert(!isdummy(t));
    othern = mainpositionfromnode(t, mp);  // 获取冲突节点的位置
    if (othern != mp) {  /* 冲突节点是否在其主位置之外？ */
      /* 是；将冲突节点移动到空闲位置 */
      while (othern + gnext(othern) != mp)  /* 找到前一个节点 */
        othern += gnext(othern);
      gnext(othern) = cast_int(f - othern);  /* 重新链接到'f' */
      *f = *mp;  /* 将冲突节点复制到空闲位置（mp->next也会被复制） */
      if (gnext(mp) != 0) {
        gnext(f) += cast_int(mp - f);  /* 修正'next' */
        gnext(mp) = 0;  /* 现在'mp'是空闲的 */
      }
      setempty(gval(mp));  // 设置mp为empty
    }
    else {  /* colliding node is in its own main position */
      /* 如果发生碰撞的节点在自己的主位置上，执行以下操作 */
      /* new node will go into free position */
      /* 新节点将进入空闲位置 */
      if (gnext(mp) != 0)
        /* 如果碰撞节点的下一个节点不为空，则将新节点链接到新位置 */
        gnext(f) = cast_int((mp + gnext(mp)) - f);  /* chain new position */
      else lua_assert(gnext(f) == 0);
      /* 否则，断言碰撞节点的下一个节点为空 */
      gnext(mp) = cast_int(f - mp);
      /* 将碰撞节点的下一个节点指向新节点 */
      mp = f;
      /* 将 mp 指向新节点 */
    }
  }
  /* 设置节点的键值 */
  setnodekey(L, mp, key);
  /* 执行垃圾回收屏障 */
  luaC_barrierback(L, obj2gco(t), key);
  /* 断言碰撞节点的值为空 */
  lua_assert(isempty(gval(mp)));
  /* 设置碰撞节点的值为新值 */
  setobj2t(L, gval(mp), value);
  /* 将新值设置到碰撞节点的值中 */
'''
** Search function for integers. If integer is inside 'alimit', get it
** directly from the array part. Otherwise, if 'alimit' is not equal to
** the real size of the array, key still can be in the array part. In
** this case, try to avoid a call to 'luaH_realasize' when key is just
** one more than the limit (so that it can be incremented without
** changing the real size of the array).
'''
# 用于查找整数的搜索函数。如果整数在'alimit'内，则直接从数组部分获取。否则，如果'alimit'不等于数组的实际大小，则键仍然可以在数组部分中。在这种情况下，当键刚好比限制多一个时（这样可以增加而不改变数组的实际大小），尽量避免调用'luaH_realasize'。

const TValue *luaH_getint (Table *t, lua_Integer key) {
  if (l_castS2U(key) - 1u < t->alimit)  /* 'key' in [1, t->alimit]? */
    return &t->array[key - 1];
  else if (!limitequalsasize(t) &&  /* key still may be in the array part? */
           (l_castS2U(key) == t->alimit + 1 ||
            l_castS2U(key) - 1u < luaH_realasize(t))) {
    t->alimit = cast_uint(key);  /* probably '#t' is here now */
    return &t->array[key - 1];
  }
  else {
    Node *n = hashint(t, key);
    for (;;) {  /* check whether 'key' is somewhere in the chain */
      if (keyisinteger(n) && keyival(n) == key)
        return gval(n);  /* that's it */
      else {
        int nx = gnext(n);
        if (nx == 0) break;
        n += nx;
      }
    }
    return &absentkey;
  }
}


'''
** search function for short strings
'''
# 用于短字符串的搜索函数

const TValue *luaH_getshortstr (Table *t, TString *key) {
  Node *n = hashstr(t, key);
  lua_assert(key->tt == LUA_VSHRSTR);
  for (;;) {  /* check whether 'key' is somewhere in the chain */
    if (keyisshrstr(n) && eqshrstr(keystrval(n), key))
      return gval(n);  /* that's it */
    else {
      int nx = gnext(n);
      if (nx == 0)
        return &absentkey;  /* not found */
      n += nx;
    }
  }
}


const TValue *luaH_getstr (Table *t, TString *key) {
  if (key->tt == LUA_VSHRSTR)
    return luaH_getshortstr(t, key);
  else {  /* for long strings, use generic case */
    TValue ko;
    setsvalue(cast(lua_State *, NULL), &ko, key);
    return getgeneric(t, &ko, 0);
  }
}


'''
** main search function
'''
# 主搜索函数
/*
** 从表 't' 的哈希部分中尝试找到一个边界。
** 从调用者来看，我们知道 'j' 要么是零，要么存在，'j + 1' 也存在。
** 我们想要找到一个更大的键，它在表中不存在，这样我们就可以在这两个键之间进行二分查找，找到一个边界。
** 我们不断将 'j' 加倍，直到得到一个不存在的索引。
** 如果加倍会溢出，我们尝试 LUA_MAXINTEGER。
** 如果 'j' 是 LUA_MAXINTEGER，我们返回 NULL，表示没有找到边界。
*/
/*
** 在表 't' 中尝试找到一个边界。 （'边界' 是一个整数索引，使得 t[i] 存在且 t[i+1] 不存在，或者如果 t[1] 不存在则为 0，如果 t[maxinteger] 存在则为 'maxinteger'。）
** （在下面的解释中，我们使用 Lua 索引，即基于 1 的索引。代码本身在索引表的数组部分时使用基于 0 的索引。）
** 代码从 'limit = t->alimit' 开始，这是数组部分中可能是边界的位置。
**
** （1）如果 't[limit]' 为空，那么它之前一定有一个边界。
** 作为一个常见情况（例如，在 't[#t]=nil' 之后），检查 'limit-1' 是否为空，如果是，则 'limit' 是一个边界。
** 否则，我们准备进行二分查找。 （'j'，作为最大整数，大于或等于 'i'，但它不能等于 'i'，因为它不存在，而 'i' 存在；所以 'j > i'。）否则，'j' 是一个边界。 （'j + 1' 不能是一个存在的整数键，因为它不是 Lua 中的有效整数。）
*/
static lua_Unsigned hash_search (Table *t, lua_Unsigned j) {
  lua_Unsigned i;
  if (j == 0) j++;  /* 调用者确保 'j + 1' 存在 */
  do {
    i = j;  /* 'i' 是一个存在的索引 */
    if (j <= l_castS2U(LUA_MAXINTEGER) / 2)
      j *= 2;
    else {
      j = LUA_MAXINTEGER;
      if (isempty(luaH_getint(t, j)))  /* t[j] 不存在？ */
        break;  /* 'j' 现在是一个不存在的索引 */
      else  /* 奇怪的情况 */
        return j;  /* 好吧，最大整数是一个边界... */
    }
  } while (!isempty(luaH_getint(t, j)));  /* 重复直到 t[j] 不存在 */
  /* i < j  &&  t[i] 存在  &&  t[j] 不存在 */
  while (j - i > 1u) {  /* 在它们之间进行二分查找 */
    lua_Unsigned m = (i + j) / 2;
    if (isempty(luaH_getint(t, m))) j = m;
    else i = m;
  }
  return i;
}


static unsigned int binsearch (const TValue *array, unsigned int i,
                                                    unsigned int j) {
  while (j - i > 1u) {  /* 二分查找 */
    unsigned int m = (i + j) / 2;
    if (isempty(&array[m - 1])) j = m;
    else i = m;
  }
  return i;
}
/*
** 获取表的长度
** 
** (1) 如果 't[limit]' 不为空，并且数组还有更多元素在 'limit' 之后，尝试在那里找到一个边界。
**     首先尝试特殊情况（应该很常见），即 'limit+1' 为空，这样 'limit' 就是一个边界。
**     否则，检查数组部分的最后一个元素。如果它为空，则必须在旧的限制（存在）和最后一个元素（不存在）之间找到一个边界，使用二分搜索找到。
**     （这个边界总是可以成为一个新的限制。）
**
** (2) 如果 't[limit]' 不存在，并且数组还有更多元素在 'limit' 之后，尝试在那里找到一个边界。
**     再次，首先尝试特殊情况（应该很常见），即 'limit+1' 为空，这样 'limit' 就是一个边界。
**     否则，检查数组部分的最后一个元素。如果它为空，则必须在旧的限制（存在）和最后一个元素（不存在）之间找到一个边界，使用二分搜索找到。
**     （这个边界总是可以成为一个新的限制。）
**
** (3) 最后一种情况是数组部分没有元素（limit == 0）或者它的最后一个元素（新的限制）存在。
**     在这种情况下，必须检查哈希部分。如果没有哈希部分或者 'limit+1' 不存在，'limit' 就是一个边界。
**     否则，调用 'hash_search' 在表的哈希部分找到一个边界。
**     （在这些情况下，边界不在数组部分内，因此不能用作新的限制。）
*/
lua_Unsigned luaH_getn (Table *t) {
  unsigned int limit = t->alimit;
  if (limit > 0 && isempty(&t->array[limit - 1])) {  /* (1)? */
    /* 在 'limit' 之前必须有一个边界 */
    if (limit >= 2 && !isempty(&t->array[limit - 2])) {
      /* 'limit - 1' 是一个边界；它可以成为一个新的限制吗？ */
      if (ispow2realasize(t) && !ispow2(limit - 1)) {
        t->alimit = limit - 1;
        setnorealasize(t);  /* 现在 'alimit' 不是真实大小 */
      }
      return limit - 1;
    }
    else {  /* must search for a boundary in [0, limit] */
      // 在 [0, limit] 范围内搜索边界
      unsigned int boundary = binsearch(t->array, 0, limit);
      // 这个边界能否代表数组的真实大小？
      if (ispow2realasize(t) && boundary > luaH_realasize(t) / 2) {
        t->alimit = boundary;  // 将其用作新的限制
        setnorealasize(t);
      }
      return boundary;
    }
  }
  // 'limit' 为零或存在于表中
  if (!limitequalsasize(t)) {  /* (2)? */
    // 'limit' > 0 并且数组在 'limit' 之后还有更多元素
    if (isempty(&t->array[limit]))  // 'limit + 1' 是否为空？
      return limit;  // 这就是边界
    // 否则，尝试数组中的最后一个元素
    limit = luaH_realasize(t);
    if (isempty(&t->array[limit - 1])) {  // 为空？
      // 在旧限制之后数组中必须存在一个边界，并且它必须是一个有效的新限制
      unsigned int boundary = binsearch(t->array, t->alimit, limit);
      t->alimit = boundary;
      return boundary;
    }
    // 否则，新的限制存在于表中；检查哈希部分
  }
  // (3) 'limit' 是最后一个元素，要么为零，要么存在于表中
  lua_assert(limit == luaH_realasize(t) &&
             (limit == 0 || !isempty(&t->array[limit - 1])));
  if (isdummy(t) || isempty(luaH_getint(t, cast(lua_Integer, limit + 1))))
    return limit;  // 'limit + 1' 不存在
  else  // 'limit + 1' 也存在
    return hash_search(t, limit);
# 如果定义了 LUA_DEBUG，则导出以下函数给测试库使用

# 返回给定键在表中的主位置
Node *luaH_mainposition (const Table *t, const TValue *key) {
  return mainpositionTV(t, key);
}

# 检查表是否为虚拟表
int luaH_isdummy (const Table *t) { return isdummy(t); }
```