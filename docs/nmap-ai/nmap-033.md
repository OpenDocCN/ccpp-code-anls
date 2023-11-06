# Nmap源码解析 33

# `liblua/ltable.c`

这段代码定义了一个名为ltable的定义，使用了哈希表类型的数据结构。在定义中使用了两个前缀：lprefix.h和LUA_CORE，分别用于定义和使用这个定义。

ltable_c的作用是定义了一个包含两个元素的表格，每个元素都是一个复合数据类型，包括一个名为“ltable”的类型和一个名为“table”的类型。其中，ltable是一个结构体变量，表示一个表格，而table是一个结构体变量，表示一个包含任意类型元素的表格。

这个定义被用于定义了一个名为ltable的常量，用于在程序中创建和操作表格。这个常量被定义为：
```cpp
#define ltable_c
```

接下来，定义了一个名为LUA_CORE的常量，用于在程序中使用LuaCore库，它是一个用于在Lua应用程序中使用C API的库。
```cpp
#define LUA_CORE
```

接着，引入了两个头文件：lprefix.h和定义了ltable_c函数的源文件。这里lprefix.h可能是一个预定义的头文件，其中包含一些用于定义和声明头文件的函数和变量。

虽然这段代码中没有具体的实现，但是通过定义了ltable这个哈希表类型的数据结构，可以用来在程序中创建和操作表格，因此具有非常重要的作用。


```cpp
/*
** $Id: ltable.c $
** Lua tables (hash)
** See Copyright Notice in lua.h
*/

#define ltable_c
#define LUA_CORE

#include "lprefix.h"


/*
** Implementation of tables (aka arrays, objects, or hash tables).
** Tables keep its elements in two parts: an array part and a hash part.
```

这段代码是一个用于在Lua中维护非负整数键的数据结构。该数据结构采用了一种称为“散列表”的技术，它允许键值非负整数在键值范围内保持顺序。该数据结构通过使用Brent的混合散列表和“桶”技术，以及一些数学和散列表相关的知识来保证键值非负整数之间的距离。

具体来说，该数据结构通过在键值范围内维护一个散列表，其中每个散列表都对应一个桶。桶中的元素是一个非负整数，它们代表了一个桶中键值非负整数的索引。每个元素都有一个“原始”位置，它也是一个非负整数，该位置与桶中的元素的索引之间存在一种特殊关系。

为了保证性能，该数据结构使用了一个混合散列表和Brent的散列表。混合散列表提供了一种灵活的查找方法，可以通过散列表中的元素直接查找，而Brent的散列表则可以提供更快速的查找。此外，该数据结构还使用了一些数学计算，比如在大负载因子下，性能仍然保持在良好水平。


```cpp
** Non-negative integer keys are all candidates to be kept in the array
** part. The actual size of the array is the largest 'n' such that
** more than half the slots between 1 and n are in use.
** Hash uses a mix of chained scatter table with Brent's variation.
** A main invariant of these tables is that, if an element is not
** in its main position (i.e. the 'original' position that its hash gives
** to it), then the colliding element is in its own main position.
** Hence even when the load factor reaches 100%, performance remains good.
*/

#include <math.h>
#include <limits.h>

#include "lua.h"

```

这段代码是一个 C 语言程序，它包含了多个头文件和定义，如下：

- ldebug.h: 这是一个用于输出调试信息的头文件，其中定义了一些调试信息的类型和函数。
- ldo.h: 这是一个用于输出调试信息的头文件，其中定义了一些调试信息的类型和函数。
- lgc.h: 这是一个用于输出调试信息的头文件，其中定义了一些调试信息的类型和函数。
- lmem.h: 这是一个用于输出调试信息的头文件，其中定义了一些调试信息的类型和函数。
- lobject.h: 这是一个用于输出调试信息的头文件，其中定义了一些调试信息的类型和函数。
- lstate.h: 这是一个用于输出调试信息的头文件，其中定义了一些调试信息的类型和函数。
- lstring.h: 这是一个用于输出调试信息的头文件，其中定义了一些调试信息的类型和函数。
- ltable.h: 这是一个用于输出调试信息的头文件，其中定义了一些调试信息的类型和函数。
- lvm.h: 这是一个用于输出调试信息的头文件，其中定义了一些调试信息的类型和函数。

由于没有输出语句，因此无法确定这些头文件的具体作用。一般来说，这些头文件可能用于在程序调试时输出调试信息，以便开发人员更好地理解程序的运行情况。


```cpp
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
```

这段代码定义了两个头文件MAXASIZE和MAXHBITS，它们用于定义整型变量最大长度和最大整数位数。

MAXASIZE通过使用cast_int函数将sizeof(int) * CHAR_BIT - 1计算得到，其中CHAR_BIT是char数据类型的位数，sizeof(int)是int数据类型的字节数。这个最大值被定义为1升法定计数单位，即1u << MAXABITS - 1。

MAXHBITS通过将MAXASIZE减1并使用luaM_limitN函数得到，这个整数表示用signed int类型能表示的最大正整数。MAXHBITS的计算方式与MAXASIZE类似，但是MAXHBITS的计算中用到了signed int类型的位数，而不是int类型的字节数。

这两部分定义可以被用来定义整型变量，比如：

```cppc
int max_asize = 0;
int max_hbits = 0;

// 在这里可以给max_asize和max_hbits赋值
```

不过，MAXASIZE和MAXHBITS只是定义好的常量，它们并不会被用来执行任何操作，只有定义的变量才能达到它们所表示的最大值。


```cpp
#define MAXABITS	cast_int(sizeof(int) * CHAR_BIT - 1)


/*
** MAXASIZE is the maximum size of the array part. It is the minimum
** between 2^MAXABITS and the maximum size that, measured in bytes,
** fits in a 'size_t'.
*/
#define MAXASIZE	luaM_limitN(1u << MAXABITS, TValue)

/*
** MAXHBITS is the largest integer such that 2^MAXHBITS fits in a
** signed int.
*/
#define MAXHBITS	(MAXABITS - 1)


```

这是一段C语言代码，定义了两个头文件：MAXHSIZE.h 和 hashpow2.h。接下来分别解释这两个头文件的作用。

首先是MAXHSIZE.h，它定义了一个宏MAXHSIZE，用于指定哈希函数的最大大小。它的含义是，这个宏中的值应该在2^MAXHBITS和size_t的最大值之间，即2^MAXHBITS和MAXHID "%30"（注意，这里应该是一个占位符，占位符%30，而不是一个实际的30字节）之间。

其次是hashpow2.h，它定义了一个名为hashpow2的函数，接受两个参数t和n，并返回一个整数。这个函数的作用是，通过执行一些计算，在哈希函数中避免使用除号（/）操作，从而减少对%符号的滥用。

总结一下，这段代码定义了一个哈希函数，通过使用size_t类型来确保哈希函数输出的结果是一个size_t类型的值。同时，还定义了一个MAXHSIZE宏，用于指定哈希函数的最大大小。另外，还定义了一个hashpow2函数，用于在哈希函数中避免使用除号操作。


```cpp
/*
** MAXHSIZE is the maximum size of the hash part. It is the minimum
** between 2^MAXHBITS and the maximum size such that, measured in bytes,
** it fits in a 'size_t'.
*/
#define MAXHSIZE	luaM_limitN(1u << MAXHBITS, Node)


/*
** When the original hash value is good, hashing by a power of 2
** avoids the cost of '%'.
*/
#define hashpow2(t,n)		(gnode(t, lmod((n), sizenode(t))))

/*
```

这段代码定义了三个哈希函数，用于将不同的数据类型(如整数、字符串、布尔值等)转换为哈希树中的节点。哈希树是一种数据结构，可以快速地查找和插入数据。

哈希函数采用哈希函数算法实现，其中哈希函数算法将输入数据映射到一个固定长度(通常是 16)的哈希值中。哈希函数算法的实现相对于简单，且对于不同类型的数据，哈希函数算法有不同的参数可以调整。

- `hashmod` 函数用于将输入的整数类型转换为哈希树中的节点。这个函数的核心是采用哈希函数算法，但是这个算法仅仅将输入的整数作为哈希值，而忽略了因式2的影响，这样可能会导致哈希树中的节点有较多的 2 因子。

- `hashstr` 函数用于将输入的字符串类型转换为哈希树中的节点。这个函数的核心是采用哈希函数算法，将字符串的每个字符的哈希值作为哈希值，然后将这些哈希值连接起来构建哈希树。

- `hashboolean` 函数用于将输入的布尔类型转换为哈希树中的节点。这个函数的核心是采用哈希函数算法，将布尔值的值作为哈希值，然后将这些哈希值连接起来构建哈希树。

- `hashpointer` 函数用于将输入的指针类型转换为哈希树中的节点。这个函数的核心是采用哈希函数算法，将指针的值作为哈希值，然后将这些哈希值连接起来构建哈希树。

另外，还定义了一个 `dummynode` 函数，它的作用是确保哈希树中所有节点的指针都指向同一个结点，即确保哈希树中所有节点都一样。


```cpp
** for other types, it is better to avoid modulo by power of 2, as
** they can have many 2 factors.
*/
#define hashmod(t,n)	(gnode(t, ((n) % ((sizenode(t)-1)|1))))


#define hashstr(t,str)		hashpow2(t, (str)->hash)
#define hashboolean(t,p)	hashpow2(t, p)


#define hashpointer(t,p)	hashmod(t, point2uint(p))


#define dummynode		(&dummynode_)

```

这段代码定义了一个名为 `dummynode_` 的 `Node` 结构体，它包含一个空节点（`{{NULL}, LUA_VEMPTY`, 0）和一个键类型字段 `key_type` 和一个指向下一个节点的指针类型字段 `next`，以及一个键值类型字段 `key_value`。键类型字段 `key_type` 的值为 `LUA_VNIL`，表示键类型为无效键，next 字段为空，表示下一个节点为空。键值类型字段 `key_value` 的值为 `ABSTKEYCONSTANT`，表示键值类型为自定义键，值可以是任何有效 Lua 类型。

此外，还定义了一个名为 `absentkey` 的常量，其值为 `ABSTKEYCONSTANT`。

该代码没有函数或其他说明，因此无法从该代码中理解其具体的作用。


```cpp
static const Node dummynode_ = {
  {{NULL}, LUA_VEMPTY,  /* value's value and type */
   LUA_VNIL, 0, {NULL}}  /* key type, next, and key value */
};


static const TValue absentkey = {ABSTKEYCONSTANT};


/*
** Hash for integers. To allow a good hash, use the remainder operator
** ('%'). If integer fits as a non-negative int, compute an int
** remainder, which is faster. Otherwise, use an unsigned-integer
** remainder, which uses all bits and ensures a non-negative result.
*/
```

这段代码是一个用于计算节点值的函数，尤其是在涉及浮点数时需要考虑数值的表示和精度。

函数接收两个参数：一个Table结构体和一个整数。整数i表示要计算的浮点数，Table的第一个元素存储了这个浮点数的阶码。

函数首先将整数i转换为unsigned int类型，并检查它是否小于等于INT_MAX。如果是，那么函数将使用hashmod函数计算hashmod函数的返回值。如果不是，那么函数将直接使用hashmod函数计算hashint函数的返回值。

在计算hashint函数中，如果输入的浮点数阶码在无符号整数范围内，那么hashint函数将直接返回阶码。否则，函数将返回输入阶码与INT_MAX的乘积，然后再加上INT_MAX。

这个函数的作用是计算一个给定浮点数的阶码，以便在将浮点数存储为整数时进行转换。对于不同的浮点数精度和表示，函数的行为可能会有所不同。


```cpp
static Node *hashint (const Table *t, lua_Integer i) {
  lua_Unsigned ui = l_castS2U(i);
  if (ui <= (unsigned int)INT_MAX)
    return hashmod(t, cast_int(ui));
  else
    return hashmod(t, ui);
}


/*
** Hash for floating-point numbers.
** The main computation should be just
**     n = frexp(n, &i); return (n * INT_MAX) + i
** but there are some numerical subtleties.
** In a two-complement representation, INT_MAX does not has an exact
```

这段代码是一个Lua脚本，它的主要目的是在Lua中实现一个将浮点数表示为整数的功能。为了实现这个功能，它采用了以下策略：

1. 如果浮点数'frexp'的绝对值小于1，那么将其乘以-INT_MIN会得到一个负数，其绝对值不会超过INT_MAX。
2. 避免使用'unsigned int'，因为它会在需要加法操作时产生问题。
3. 避免使用'~u'，因为它会在需要取相反数操作时产生问题。

具体实现是通过一个名为'l_hashfloat'的函数来实现的，该函数接收一个浮点数参数'n'，并返回一个整数表示。如果'n'不满足Lua中的数学规则，那么函数返回0；否则，它会将'n'转换为整数，并将其与INT_MIN做乘法运算，然后再进行取整操作，得到的结果就是该浮点数的整数表示。


```cpp
** representation as a float, but INT_MIN does; because the absolute
** value of 'frexp' is smaller than 1 (unless 'n' is inf/NaN), the
** absolute value of the product 'frexp * -INT_MIN' is smaller or equal
** to INT_MAX. Next, the use of 'unsigned int' avoids overflows when
** adding 'i'; the use of '~u' (instead of '-u') avoids problems with
** INT_MIN.
*/
#if !defined(l_hashfloat)
static int l_hashfloat (lua_Number n) {
  int i;
  lua_Integer ni;
  n = l_mathop(frexp)(n, &i) * -cast_num(INT_MIN);
  if (!lua_numbertointeger(n, &ni)) {  /* is 'n' inf/-inf/NaN? */
    lua_assert(luai_numisnan(n) || l_mathop(fabs)(n) == cast_num(HUGE_VAL));
    return 0;
  }
  else {  /* normal case */
    unsigned int u = cast_uint(i) + cast_uint(ni);
    return cast_int(u <= cast_uint(INT_MAX) ? u : ~u);
  }
}
```

这段代码是一个名为 `mainpositionTV` 的函数，它接受两个参数：一个指向 `Table` 对象的指针 `t` 和一个指向 `TValue` 对象的指针 `key`。函数返回一个指向 `Node` 对象的指针，它代表表中具有该哈希值的元素在表中的位置。

函数内部采用了一个 `switch` 语句，根据传递给它的 `key` 的类型不同，实现了不同的返回值。具体来说，如果 `key` 是 `LUA_VNUMINT`，函数将返回该哈希值在表中的索引；如果 `key` 是 `LUA_VNUMFLT`，函数将返回哈希浮点数 `n` 对应的模 `t` 哈希值；如果 `key` 是 `LUA_VSHRSTR`，函数将返回哈希字符串；如果 `key` 是 `LUA_VLNGSTR`，函数将返回哈希多级字符串；如果 `key` 是 `LUA_VFALSE` 或 `LUA_VTRUE`，函数将返回哈希布尔值 `0` 或 `1`。

如果 `key` 是 `LUA_VLIGHTUSERDATA` 或 `LUA_VLCF`，函数将返回一个指向对象（即哈希指针）的指针，该指针将指向具有该哈希值的 `Node` 对象。如果 `key` 是 `LUA_VFALSE` 或 `LUA_VTRUE`，函数将返回哈希 `boolean` 函数的返回值。如果 `key` 是 `LUA_VLNGSTR`，函数将尝试哈希该字符串，但如果发生错误，将返回 `L_哈希error`。


```cpp
#endif


/*
** returns the 'main' position of an element in a table (that is,
** the index of its hash value).
*/
static Node *mainpositionTV (const Table *t, const TValue *key) {
  switch (ttypetag(key)) {
    case LUA_VNUMINT: {
      lua_Integer i = ivalue(key);
      return hashint(t, i);
    }
    case LUA_VNUMFLT: {
      lua_Number n = fltvalue(key);
      return hashmod(t, l_hashfloat(n));
    }
    case LUA_VSHRSTR: {
      TString *ts = tsvalue(key);
      return hashstr(t, ts);
    }
    case LUA_VLNGSTR: {
      TString *ts = tsvalue(key);
      return hashpow2(t, luaS_hashlongstr(ts));
    }
    case LUA_VFALSE:
      return hashboolean(t, 0);
    case LUA_VTRUE:
      return hashboolean(t, 1);
    case LUA_VLIGHTUSERDATA: {
      void *p = pvalue(key);
      return hashpointer(t, p);
    }
    case LUA_VLCF: {
      lua_CFunction f = fvalue(key);
      return hashpointer(t, f);
    }
    default: {
      GCObject *o = gcvalue(key);
      return hashpointer(t, o);
    }
  }
}


```

这段代码是一个Lua脚本，主要作用是定义了一个名为`mainpositionfromnode`的函数，接收两个参数：一个指向`Table`类的指针`t`，和一个指向`Node`类的指针`nd`，然后返回一个指向`mainpositionTV`函数的指针。

`mainpositionfromnode`函数的作用是从`t`中检索一个名为`key`的节点，并将其存储在`nd`指向的内存位置上，然后返回`mainpositionTV`函数的指针，这个函数的参数和返回值类型与`mainpositionfromnode`函数完全相同，只是参数的类型从`const Table *t`改为了`Node *nd`。

`mainpositionTV`函数的作用是接收一个指向`Table`类的指针`t`，以及一个节点`nd`，返回一个指向包含当前节点及其所有子节点位置的`Node *list`的指针。这个函数使用了Lua中的`getline`函数获取输入节点中的表格数据，并将其存储在`nd`指向的内存位置上。由于`nd`是一个指向`Node`的指针，所以这个函数的参数和返回值类型也与`mainpositionfromnode`函数完全相同。


```cpp
l_sinline Node *mainpositionfromnode (const Table *t, Node *nd) {
  TValue key;
  getnodekey(cast(lua_State *, NULL), &key, nd);
  return mainpositionTV(t, &key);
}


/*
** Check whether key 'k1' is equal to the key in node 'n2'. This
** equality is raw, so there are no metamethods. Floats with integer
** values have been normalized, so integers cannot be equal to
** floats. It is assumed that 'eqshrstr' is simply pointer equality, so
** that short strings are handled in the default case.
** A true 'deadok' means to accept dead keys as equal to their original
** values. All dead keys are compared in the default case, by pointer
```

Yes, you can use the `rawtt` function to compare the raw value of the keys. If
```cpp
rawtt(k1) != keytt(n2)
```

and the `deadok` flag is `1`, you can also check if the key is dead by checking if `keyisdead(n2)` is `true` and `iscollectable(k1)` is `false`.

It is important to note that this function may produce a `false positive` if the keys being compared are not truly identical. However, in the scenario where this function is being used as part of a regular traversal and `next` is being used to move between nodes, this false positive is not likely to cause problems because `next` will return some other valid item on the table or `nil`.

It is also worth noting that if `k1` is a regular value and `n2` is a table node, the function may not work correctly because `keyisdead` and `iscollectable` will always be `false`. In this case, the function will always return `0`.


```cpp
** identity. (Only collectable objects can produce dead keys.) Note that
** dead long strings are also compared by identity.
** Once a key is dead, its corresponding value may be collected, and
** then another value can be created with the same address. If this
** other value is given to 'next', 'equalkey' will signal a false
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


```

这段代码定义了一个名为 limitequalsasize 的函数，用于判断一个 'table' 中的 'array' 成员是否与另一个 'table' 中的 'array' 成员的大小相等。如果它们的大小相等，则返回该 'array' 成员的值，否则返回一个更大的整数。

limitequalsasize 的实现比较复杂，包含了一系列判断和计算。首先，它检查给定的 'array' 成员是否与 'table' 中的 'array' 成员的大小相等，如果是，则直接返回该 'array' 成员的值。否则，它尝试计算 'array' 成员 'alimit' 的大小，如果该大小与 'table' 中的 'array' 成员的大小相等，则返回该 'array' 成员的值，否则计算最小的 2 的幂，不包含偏移量，并将其与 'array' 成员的大小进行逻辑与运算，最后输出结果。


```cpp
/*
** True if value of 'alimit' is equal to the real size of the array
** part of table 't'. (Otherwise, the array part must be larger than
** 'alimit'.)
*/
#define limitequalsasize(t)	(isrealasize(t) || ispow2((t)->alimit))


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
```

这段代码是一个 C 语言函数，它的作用是检查一个数组的大小是否为 2 的幂次方。它首先检查数组中是否有超过 32 位的元素，如果没有，则将数组长度加倍。然后，它将检查数组长度是否为正，如果不是，则退出函数。

具体来说，代码首先定义了一个变量 ispow2，用于检查给定的整数是否是 2 的幂次方。然后，代码定义了一个名为 size 的整数变量，用于存储数组的大小。接下来，代码使用 if 语句判断给定的整数是否大于 32 并小于给定的数组长度（alimit）的平方根。如果是，代码返回给定的数组长度。否则，代码将 size 加倍，并检查新的数组长度是否符合要求。最终，代码返回给定的数组长度。


```cpp
#if (UINT_MAX >> 30) > 3
    size |= (size >> 32);  /* unsigned int has more than 32 bits */
#endif
    size++;
    lua_assert(ispow2(size) && size/2 < t->alimit && t->alimit < size);
    return size;
  }
}


/*
** Check whether real size of the array is a power of 2.
** (If it is not, 'alimit' cannot be changed to any other value
** without changing the real size.)
*/
```

这两段代码定义了两个名为 `ispow2realasize` 和 `setlimittosize` 的函数，它们都接受一个参数 `t`，代表一个 `Table` 类型的数据结构。这两段代码的功能是确保 `t` 中的数据满足 `ispow2` 和 `isrealasize` 函数的定义，然后设置 `t->alimit` 的值，使其满足 `limitasasize` 函数的定义。

`ispow2realasize` 函数首先调用 `isrealasize` 函数检查给定的 `t` 是否为 `realasize` 类型。如果是，函数将返回 `true`，否则将返回 `false`。然后，函数调用 `ispow2` 函数检查给定的 `t` 是否可以被 2 整除，并返回结果。最后，函数使用 `!=` 和 `||` 运算符组合这两个条件，以返回 `true` 表示 `t` 满足 `ispow2` 和 `isrealasize` 函数的定义。

`setlimittosize` 函数首先调用 `isrealasize` 函数检查给定的 `t` 是否为 `realasize` 类型。如果是，函数将 `t->alimit` 的值设置为 `luaH_realasize(t)`，然后调用 `ispow2` 函数检查给定的 `t` 是否可以被 2 整除，并将结果存储在 `t->alimit` 中。最后，函数返回 `t->alimit` 的值作为 `setrealasize` 函数的参数，以设置 `t->alimit` 的值，使其满足 `limitasasize` 函数的定义。


```cpp
static int ispow2realasize (const Table *t) {
  return (!isrealasize(t) || ispow2(t->alimit));
}


static unsigned int setlimittosize (Table *t) {
  t->alimit = luaH_realasize(t);
  setrealasize(t);
  return t->alimit;
}


#define limitasasize(t)	check_exp(isrealasize(t), t->alimit)



```

这是一段用于在 Smalltalk 语言中获取键值对中键值的函数的实现。该函数名为 `getgeneric`，它的参数为 `Table` 类型的指针 `t` 和要查找的键 `key`，以及一个表示键是否已存在于链中的整数 `deadok`。函数返回值为链表中第一个匹配键的值，或者在链表中未找到该键时返回一个表示键不存在的小幅标量。

函数的实现采用了线性搜索的方式，首先在链表的头部插入节点 `n`，然后进入循环，每次循环将搜索范围向前移动一位，直到找到 `key` 或者确定 `key` 不存在于链中。在循环中，如果找到 `key`，则返回该节点中的值，否则计算 `nx`（即 `gnext` 函数的返回值）的相反值作为结果。如果 `nx` 等于 0，则返回表示键不存在的小幅标量。


```cpp
/*
** "Generic" get version. (Not that generic: not valid for integers,
** which may be in array part, nor for floats with integral values.)
** See explanation about 'deadok' in function 'equalkey'.
*/
static const TValue *getgeneric (Table *t, const TValue *key, int deadok) {
  Node *n = mainpositionTV(t, key);
  for (;;) {  /* check whether 'key' is somewhere in the chain */
    if (equalkey(key, n, deadok))
      return gval(n);  /* that's it */
    else {
      int nx = gnext(n);
      if (nx == 0)
        return &absentkey;  /* not found */
      n += nx;
    }
  }
}


```

这段代码是一个Lua函数，名为`arrayindex`，它的作用是返回一个整数，表示在给定的table中，'k'是否是一个适当的key。具体实现如下：

1. 如果'k'在数组部分的一个适当的key范围内，那么返回该范围的第一个元素在数组部分的索引；
2. 如果'k'不在数组部分范围内，或者'k'是数组范围的最后一个元素，那么返回0；
3. 如果'k'是一个不存在的key，那么返回0。

这里给出的是一个函数原型，没有具体的实现。你需要根据你的table结构来具体实现这个函数。


```cpp
/*
** returns the index for 'k' if 'k' is an appropriate key to live in
** the array part of a table, 0 otherwise.
*/
static unsigned int arrayindex (lua_Integer k) {
  if (l_castS2U(k) - 1u < MAXASIZE)  /* 'k' in [1, MAXASIZE]? */
    return cast_uint(k);  /* 'key' is an appropriate array index */
  else
    return 0;
}


/*
** returns the index of a 'key' for table traversals. First goes all
** elements in the array part, then elements in the hash part. The
```

这段代码是一个Lua脚本，它的主要作用是计算一个名为findindex的函数。该函数在给定的Lua上下文中执行，并接受一个名为table的表，一个名为key的键和一个作为其大小的unsigned int asize的参数。函数返回一个指向包含键的索引的指针，或者如果键在表中找不到，则返回一个无穷大。

函数的实现包括以下几个步骤：

1. 如果给定的键是NIL，函数立即返回0，这是第一个 iteration。

2. 如果给定的键是一个整数，函数使用table.sub递归地查找键在表中的位置。

3. 如果给定的键在表中找不到，函数使用getgeneric函数（这是另一个函数）将其包装为单个参数，并传递给findindex函数。

4. 如果包不住按键，函数会使用hashtable函数（这是另一个函数）将其转换为哈希表的节点。然后，通过计算从哈希表节点位置i - 1u（注意：哈希表节点的位置是1u）加asize作为键在哈希表中的索引，并返回这个索引。

总之，该函数的目的是提供一个简单的途径来查找给定键在Lua表中或哈希表中的位置。


```cpp
** beginning of a traversal is signaled by 0.
*/
static unsigned int findindex (lua_State *L, Table *t, TValue *key,
                               unsigned int asize) {
  unsigned int i;
  if (ttisnil(key)) return 0;  /* first iteration */
  i = ttisinteger(key) ? arrayindex(ivalue(key)) : 0;
  if (i - 1u < asize)  /* is 'key' inside array part? */
    return i;  /* yes; that's the index */
  else {
    const TValue *n = getgeneric(t, key, 1);
    if (l_unlikely(isabstkey(n)))
      luaG_runerror(L, "invalid key to 'next'");  /* key not found */
    i = cast_int(nodefromval(n) - gnode(t, 0));  /* key index in hash table */
    /* hash elements are numbered after array ones */
    return (i + 1) + asize;
  }
}


```

这段代码是一个 Lua 脚本，它在 Lua 5.1 版本中定义了一个名为 `luaH_next` 的函数。该函数的作用是：

1. 如果传入的 `table` 对象中有一个名为 `key` 的键，并且该键在数组 `t` 中找到了第一个匹配的元素，那么函数返回 1，表示成功尝试了查找并修改了该键的值。
2. 否则，函数遍历 `t` 数组，尝试找到与传入的键 `key` 匹配的元素。如果找到了第一个非空元素，那么函数将尝试使用内置函数 `setivalue` 修改该元素的值，并使用 `setobj2s` 函数将修改后的元素与键关联起来。如果遍历完成后仍然没有找到匹配的元素，那么函数返回 0，表示没有找到匹配的元素。
3. 函数还可以进行第二个循环，根据传入的键 `key` 在 `t` 数组中的位置 `i`，尝试使用内置函数 `getnodekey` 获取与该键关联的节点，并使用 `setobj2s` 函数将节点与键关联起来。如果循环完成后仍然没有找到匹配的节点，那么函数返回 0，表示没有找到匹配的节点。


```cpp
int luaH_next (lua_State *L, Table *t, StkId key) {
  unsigned int asize = luaH_realasize(t);
  unsigned int i = findindex(L, t, s2v(key), asize);  /* find original key */
  for (; i < asize; i++) {  /* try first array part */
    if (!isempty(&t->array[i])) {  /* a non-empty entry? */
      setivalue(s2v(key), i + 1);
      setobj2s(L, key + 1, &t->array[i]);
      return 1;
    }
  }
  for (i -= asize; cast_int(i) < sizenode(t); i++) {  /* hash part */
    if (!isempty(gval(gnode(t, i)))) {  /* a non-empty entry? */
      Node *n = gnode(t, i);
      getnodekey(L, s2v(key), n);
      setobj2s(L, key + 1, gval(n));
      return 1;
    }
  }
  return 0;  /* no more elements */
}


```

这段代码是一个Lua函数，名为freehash，其作用是重新计算一个Table对象t中数组部分的大小。这里的作用是通过luaM_freearray函数来实现的，该函数会自动计算并释放数组元素所占用的内存空间。首先，该函数判断t是否为空表，如果不是，则会执行以下操作：将数组元素所占用的内存空间（t->node和sizenode(t)）从内存中释放，并传入luaM_freearray函数。luaM_freearray函数会自动计算并释放数组元素所占用的内存空间，同时会返回一个指向数组元素的指针，该指针将作为luaM_getfield函数的第一个参数，用于输出数组元素。


```cpp
static void freehash (lua_State *L, Table *t) {
  if (!isdummy(t))
    luaM_freearray(L, t->node, cast_sizet(sizenode(t)));
}


/*
** {=============================================================
** Rehash
** ==============================================================
*/

/*
** Compute the optimal size for the array part of table 't'. 'nums' is a
** "count array" where 'nums[i]' is the number of integers in the table
```

这段代码定义了一个名为`computesizes`的函数，它接受一个整型数组`nums`以及一个指向整型变量`pna`的引用作为参数。函数返回一个整型变量`optimal`，它表示将`na`元素分配到数组部分后，使得目标数组部分达到的最大值。

函数的主要逻辑如下：

1. 初始化变量`twotoi`为1，`a`为0，`na`为0，`optimal`为0；
2. 进入循环，条件为`twotoi > 0`且当前元素`a`加上当前元素`nums`的值后大于等于`twotoi/2`；
3. 每次循环，将当前元素`a`添加到`a`中，并将`twotoi`乘以2；
4. 如果`a`大于等于`twotoi/2`，则将`optimal`设置为`twotoi`，并将`na`设置为`a`；
5. 如果`twotoi`大于`0`，则将`na`设置为`a`，并将`optimal`设置为`twotoi`；
6. 如果`twotoi`已经达到最大值，则退出循环，并将`optimal`设置为`0`；
7. 最后，将`na`设置为`optimal`，并返回`optimal`。

函数的实现中，通过一个循环来遍历数组`nums`的所有元素，计算出目标数组部分的最大值。当目标数组部分达到最大值后，退出循环并返回最大值。如果数组部分可以完全分配，则返回目标数组长度。


```cpp
** between 2^(i - 1) + 1 and 2^i. 'pna' enters with the total number of
** integer keys in the table and leaves with the number of keys that
** will go to the array part; return the optimal size.  (The condition
** 'twotoi > 0' in the for loop stops the loop if 'twotoi' overflows.)
*/
static unsigned int computesizes (unsigned int nums[], unsigned int *pna) {
  int i;
  unsigned int twotoi;  /* 2^i (candidate for optimal size) */
  unsigned int a = 0;  /* number of elements smaller than 2^i */
  unsigned int na = 0;  /* number of elements to go to array part */
  unsigned int optimal = 0;  /* optimal size for array part */
  /* loop while keys can fill more than half of total size */
  for (i = 0, twotoi = 1;
       twotoi > 0 && *pna > twotoi / 2;
       i++, twotoi *= 2) {
    a += nums[i];
    if (a > twotoi/2) {  /* more than half elements present? */
      optimal = twotoi;  /* optimal size (till now) */
      na = a;  /* all elements up to 'optimal' will go to array part */
    }
  }
  lua_assert((optimal == 0 || optimal / 2 < na) && na <= optimal);
  *pna = na;
  return optimal;
}


```

这段代码是一个Lua函数，名为countint，它接受一个整数键（lua_Integer）和一个整数数组（unsigned int *nums）。它的作用是统计数组t中key所在的下标范围（注意：不是整个数组t的所有下标），并返回这个范围内的非空键的个数。

函数首先检查key是否是一个有效的数组索引。如果是，就遍历key所在的下标，将这个下标的num值加1，然后返回这个范围内的非空键的个数。如果不是有效的数组索引，函数返回0。

countint函数的实现比较简单，主要使用了Lua的数学函数和逻辑函数。通过返回前半句 counts[i]而非零的键的个数，可以用来在需要时动态地调整数组的大小，而不会影响整个数组t的元素个数。


```cpp
static int countint (lua_Integer key, unsigned int *nums) {
  unsigned int k = arrayindex(key);
  if (k != 0) {  /* is 'key' an appropriate array index? */
    nums[luaO_ceillog2(k)]++;  /* count as such */
    return 1;
  }
  else
    return 0;
}


/*
** Count keys in array part of table 't': Fill 'nums[i]' with
** number of keys that will go into corresponding slice and return
** total number of non-nil keys.
```

这段代码是一个函数 `numusearray`，它计算一个数组中元素的和。该函数接收一个数组指针 `t` 和一个元素数组 `nums` 作为参数。数组长度存储在变量 `n` 中。

函数的实现采用位运算，对数组中的每个元素进行计数。具体来说，首先定义了一个常量 `lg` 和一个变量 `ttlg`，分别表示以2为底数的对数和2的整数次幂。然后定义了一个变量 `ause`，用于存储数组中元素的累计值。接着定义了一个计数器 `i`，用于遍历数组元素。最后定义了一个常量 `asize`，用于限制数组长度。

函数的实现分三个部分：

1. 第一个循环：计算数组中的元素个数。数组的元素个数位于 `asize` 和 `lim` 之间，根据 `isempty` 函数的返回值，如果 `asize` 大于 `lim`，则将 `lim` 调整为大于 `asize` 的最小值。然后通过计数器 `i` 在 `asize` 和 `lim` 之间遍历数组元素，如果当前元素在数组中存在，则将其计入 `ause` 中，并将 `i` 自增。
2. 第二个循环：计算数组中每个元素的和。首先定义了一个变量 `lc`，用于计数当前元素所在的范围（即 `2^(lg - 1)` 到 `2^lg` 之间的元素个数）。接着定义了一个变量 `lim`，用于存储当前元素在数组中的最大长度（即 `asize` 减去当前元素所在的下标）。然后判断 `lim` 是否大于 `asize`，如果是，则将 `asize` 调整为 `lim`，并将 `i` 增加当前元素在数组中的位置。最后，累加 `lc` 的值到 `ause` 中。
3. 返回：函数返回 `ause`，即数组中元素的和。


```cpp
*/
static unsigned int numusearray (const Table *t, unsigned int *nums) {
  int lg;
  unsigned int ttlg;  /* 2^lg */
  unsigned int ause = 0;  /* summation of 'nums' */
  unsigned int i = 1;  /* count to traverse all array keys */
  unsigned int asize = limitasasize(t);  /* real array size */
  /* traverse each slice */
  for (lg = 0, ttlg = 1; lg <= MAXABITS; lg++, ttlg *= 2) {
    unsigned int lc = 0;  /* counter */
    unsigned int lim = ttlg;
    if (lim > asize) {
      lim = asize;  /* adjust upper limit */
      if (i > lim)
        break;  /* no more elements to count */
    }
    /* count elements in range (2^(lg - 1), 2^lg] */
    for (; i <= lim; i++) {
      if (!isempty(&t->array[i-1]))
        lc++;
    }
    nums[lg] += lc;
    ause += lc;
  }
  return ause;
}


```

该函数的作用是计算哈希表中包含元素的数量，并将数量存储在给定的数组pna中。哈希表中元素的计算方式如下：

1. 对于每个节点，首先检查该节点在哈希表中的键是否为整数。
2. 如果键是整数，则统计该键在数组中的计数。
3. 统计哈希表中所有包含键的节点数量。
4. 将节点数量存储在整数变量ause中。
5. 将节点数量ause存储在数组pna中。
6. 返回节点数量totaluse。

函数的实现中，首先定义了三个整型变量totaluse、ause和i，用于统计哈希表中元素的数量、统计计数值和节点在哈希表中的下标。在循环中，首先定义了一个节点指针变量n，用于访问哈希表中的节点。然后通过isempty()函数判断节点值是否为空，如果是，则执行以下语句：

1. 如果节点值是整数，则执行以下语句：

  1.1. 如果节点键是整数，执行以下语句：
       ause += countint(keyival(n), nums);  // 统计该键在数组中的计数
       totaluse++;                           // 统计哈希表中元素的个数
   1.2. 返回语句：

2. 在循环结束后，返回节点数量totaluse。

该函数可以高效地计算哈希表中包含元素的数量，使得在负载均衡的情况下，仍然可以快速访问哈希表中的元素。


```cpp
static int numusehash (const Table *t, unsigned int *nums, unsigned int *pna) {
  int totaluse = 0;  /* total number of elements */
  int ause = 0;  /* elements added to 'nums' (can go to array part) */
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


```

这段代码是一个Lua脚本，它的目的是为给定的表（Table）创建一个哈希部分的数组，数组大小可以根据给定的参数来计算。哈希部分数组中的每个元素都将存储为二进制数组中的一个节点（Node）。

首先，函数的参数包括一个Lua虚拟机栈（State）、一个表（Table）和一个参数为整数的size。如果size为0，则创建一个大小为1的节点数组，并将其设置为'dummynode'。如果size大于最大二进制数（MAXHBITS）和最大二进制数大小（MAXHSIZE），则会引发Lua运行时错误。如果size为正整数，则会创建一个大小为size的二进制数组，并将其存储为表的哈希部分数组。

函数的主体部分如下：

```cpp
static void setnodevector (lua_State *L, Table *t, unsigned int size) {
 if (size == 0) {  /* no elements to hash part? */
   t->node = cast(Node *, dummynode);  /* use common 'dummynode' */
   t->lsizenode = 0;
   t->lastfree = NULL;  /* signal that it is using dummy node */
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
   t->lastfree = gnode(t, size);  /* all positions are free */
 }
}
```

这段代码的作用是创建一个哈希部分的数组，数组大小可以根据给定的参数来计算。如果给定的参数为0，则创建一个大小为1的节点数组，并将其设置为'dummynode'。如果给定的参数大于最大二进制数和最大二进制数大小，则会引发Lua运行时错误。如果给定的参数为正整数，则会创建一个大小为参数的哈希部分数组，并将其存储为表的哈希部分。


```cpp
/*
** Creates an array for the hash part of a table with the given
** size, or reuses the dummy node if size is zero.
** The computation for size overflow is in two steps: the first
** comparison ensures that the shift in the second one does not
** overflow.
*/
static void setnodevector (lua_State *L, Table *t, unsigned int size) {
  if (size == 0) {  /* no elements to hash part? */
    t->node = cast(Node *, dummynode);  /* use common 'dummynode' */
    t->lsizenode = 0;
    t->lastfree = NULL;  /* signal that it is using dummy node */
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
    t->lastfree = gnode(t, size);  /* all positions are free */
  }
}


```

这是一段Lua脚本，其作用是将一个名为"ot"的哈希表中的所有元素插入到另一个名为"t"的表格中。

具体来说，该脚本会将哈希表中的每个元素（键值对）作为键（key）输入，然后将其插入到表格中对应的位置，即通过比较哈希表中的键和表格中的键来确定插入的位置。如果哈希表中的键在表格中 already exists，那么插入操作不会触发缓存机制，也就是说，这个条目已经存在于表格中，可以简单直接地通过比较哈希表键和表格键来插入。

注意，该脚本没有输出任何东西，也没有进行任何错误处理。


```cpp
/*
** (Re)insert all elements from the hash part of 'ot' into table 't'.
*/
static void reinsert (lua_State *L, Table *ot, Table *t) {
  int j;
  int size = sizenode(ot);
  for (j = 0; j < size; j++) {
    Node *old = gnode(ot, j);
    if (!isempty(gval(old))) {
      /* doesn't need barrier/invalidate cache, as entry was
         already present in the table */
      TValue k;
      getnodekey(L, &k, old);
      luaH_set(L, t, &k, gval(old));
    }
  }
}


```

这是一段C语言代码，定义了一个名为`exchangehashpart`的静态函数。该函数接受两个参数：一个指向表`t1`的指针`t1`，另一个指向表`t2`的指针`t2`。

函数的作用是交换`t1`和`t2`中所有哈希表的部分内容，使得两个哈希表的内容相互独立，即使它们的哈希表部分数据元素顺序相同。

函数的具体实现包括以下几步：

1. 使用`lu_byte`类型将`lsizenode`作为参数传递给`exchangehashpart`函数，将`lsizenode`存储在`t1`中哈希表的第一个元素，然后将其作为函数内部的一个局部变量。

2. 使用`Node`类型将`node`作为参数传递给`exchangehashpart`函数，将`node`存储在`t1`中哈希表的第一个元素，然后将其作为函数内部的一个局部变量。

3. 使用`Node`类型将`lastfree`作为参数传递给`exchangehashpart`函数，将`lastfree`存储在`t1`中哈希表的最后一个元素，然后将其作为函数内部的一个局部变量。

4. 使用`Node`类型将`lsizenode`作为参数传递给`exchangehashpart`函数，将`lsizenode`存储在`t2`中哈希表的第一个元素，然后将其作为函数内部的一个局部变量。

5. 使用`Node`类型将`node`作为参数传递给`exchangehashpart`函数，将`node`存储在`t2`中哈希表的第一个元素，然后将其作为函数内部的一个局部变量。

6. 使用`Node`类型将`lastfree`作为参数传递给`exchangehashpart`函数，将`lastfree`存储在`t2`中哈希表的最后一个元素，然后将其作为函数内部的一个局部变量。

7. 在函数内部，将`t1`和`t2`中哈希表的第一个元素和最后一个元素进行交换，使得它们的值相反。

8. 在函数内部，将`t1`和`t2`中哈希表的第三个元素和第二个元素进行交换，使得它们的值相反。

9. 在函数内部，将`t1`和`t2`中哈希表的第四个元素和第三个元素进行交换，使得它们的值相反。

10. 返回函数内部的一个局部变量，表示交换哈希表后得到的哈希表。


```cpp
/*
** Exchange the hash part of 't1' and 't2'.
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


```

This code seems to be a Lua extension of the Lua table handling system.

It creates a table `t` with a vector of integers `ai`.

When the table is indexed by the integer `i`, the function `setvalue(L, i, value)` is called to set the value at index `i` in the table.

When the table is indexed by the integer `i`, the function `getvalue(L, i, value)` is called to retrieve the value at index `i` in the table.

When the table is inserted or updated, the function `setarray(L, old_table, new_table, n)` is called to handle the array manipulation.

The function `reinsert(L, old_table, new_table)` is used to insert elements from the `old_table` into the `new_table`.

It is important to note that this code is provided as-is and may contain errors, and it is recommended to have a thorough understanding of the Lua table handling system and the Lua programming language before attempting to modify it.


```cpp
/*
** Resize table 't' for the new given sizes. Both allocations (for
** the hash part and for the array part) can fail, which creates some
** subtleties. If the first allocation, for the hash part, fails, an
** error is raised and that is it. Otherwise, it copies the elements from
** the shrinking part of the array (if it is shrinking) into the new
** hash. Then it reallocates the array part.  If that fails, the table
** is in its original state; the function frees the new hash part and then
** raises the allocation error. Otherwise, it sets the new hash part
** into the table, initializes the new part of the array (if any) with
** nils and reinserts the elements of the old hash back into the new
** parts of the table.
*/
void luaH_resize (lua_State *L, Table *t, unsigned int newasize,
                                          unsigned int nhsize) {
  unsigned int i;
  Table newt;  /* to keep the new hash part */
  unsigned int oldasize = setlimittosize(t);
  TValue *newarray;
  /* create new hash part with appropriate size into 'newt' */
  setnodevector(L, &newt, nhsize);
  if (newasize < oldasize) {  /* will array shrink? */
    t->alimit = newasize;  /* pretend array has new size... */
    exchangehashpart(t, &newt);  /* and new hash */
    /* re-insert into the new hash the elements from vanishing slice */
    for (i = newasize; i < oldasize; i++) {
      if (!isempty(&t->array[i]))
        luaH_setint(L, t, i + 1, &t->array[i]);
    }
    t->alimit = oldasize;  /* restore current size... */
    exchangehashpart(t, &newt);  /* and hash (in case of errors) */
  }
  /* allocate new array */
  newarray = luaM_reallocvector(L, t->array, oldasize, newasize, TValue);
  if (l_unlikely(newarray == NULL && newasize > 0)) {  /* allocation failed? */
    freehash(L, &newt);  /* release new hash part */
    luaM_error(L);  /* raise error (with array unchanged) */
  }
  /* allocation ok; initialize new part of the array */
  exchangehashpart(t, &newt);  /* 't' has the new hash ('newt' has the old) */
  t->array = newarray;  /* set new array part */
  t->alimit = newasize;
  for (i = oldasize; i < newasize; i++)  /* clear new slice of the array */
     setempty(&t->array[i]);
  /* re-insert elements from old hash part into new parts */
  reinsert(L, &newt, t);  /* 'newt' now has the old hash */
  freehash(L, &newt);  /* free old hash part */
}


```

这段代码是一个名为"luaH_resizearray"的函数，其作用是重新计算一个表格（table）中键（key）的哈希值（hash value）。这个表格最多可以容纳多少键？这是一个通过调用"nums"数组来实现的。

首先，我们定义了一个名为"rehash"的静态函数，该函数接受一个table（表格）和一个键（key）的哈希值（ek）作为参数。

在函数内部，我们先定义了一个名为"asize"的变量，用于存储计算得到的数组长度。然后，我们定义了一个名为"na"的变量，用于存储哈希表部分中的键的数量。接着，我们定义了一个名为"nums"的数组，用于存储哈希表部分中的键值对。这个数组长度是通过对表格中的键进行哈希计算得出的。

接下来，我们使用"setlimittosize"函数来确保哈希表部分的大小不会超出指定的最大值。然后，我们使用"na"变量来计算哈希表部分中的键的数量，这个数量是通过调用"numsearray"函数得出的。注意，"numsearray"函数会自动计算哈希表部分中键的数量，这个数量包括键本身。

接着，我们使用"computesizes"函数来计算哈希表部分的大小，这个函数会根据哈希表部分中的键的数量和键的哈希值计算出哈希表的大小。然后，我们使用"luaH_resize"函数来重新调整表格的大小，这个函数会根据哈希表部分的大小和哈希表部分中的键的数量计算出调整后的最大键值。最后，我们将哈希表部分的大小和键的数量存储回变量的"na"中，这个变量将用于计算哈希表部分中的键的数量。

需要注意的是，这个函数没有返回值，因为它只是执行了重新计算哈希表部分的过程，而没有返回任何结果。


```cpp
void luaH_resizearray (lua_State *L, Table *t, unsigned int nasize) {
  int nsize = allocsizenode(t);
  luaH_resize(L, t, nasize, nsize);
}

/*
** nums[i] = number of keys 'k' where 2^(i - 1) < k <= 2^i
*/
static void rehash (lua_State *L, Table *t, const TValue *ek) {
  unsigned int asize;  /* optimal size for array part */
  unsigned int na;  /* number of keys in the array part */
  unsigned int nums[MAXABITS + 1];
  int i;
  int totaluse;
  for (i = 0; i <= MAXABITS; i++) nums[i] = 0;  /* reset counts */
  setlimittosize(t);
  na = numusearray(t, nums);  /* count keys in array part */
  totaluse = na;  /* all those keys are integer keys */
  totaluse += numusehash(t, nums, &na);  /* count keys in hash part */
  /* count extra key */
  if (ttisinteger(ek))
    na += countint(ivalue(ek), nums);
  totaluse++;
  /* compute new size for array part */
  asize = computesizes(nums, &na);
  /* resize the table to new computed sizes */
  luaH_resize(L, t, asize, totaluse - na);
}



```

这段代码是一个Lua脚本中的函数声明，定义了一个名为`luaH_new`的函数。

具体来说，这个函数接受一个参数`L`，代表当前Lua脚本的状态，它将在此状态下创建一个新的`Table`对象。函数返回一个新的`Table`对象的引用。

函数体中，首先定义了一个名为`luaH_new`的函数，后面跟着一个大括号。接着在函数体中，定义了一个名为`luaC_newobj`的函数，这个函数将返回一个`GCObject`对象，代表一个新的`Table`对象的内存指针。接着，通过调用`GCObject_init`函数，将`luaC_newobj`返回的`GCObject`对象初始化为一个新的`Table`对象，并将其存储到变量`o`中。

接下来，定义了一个名为`t`的`Table`对象，将其`metatable`成员设置为`NULL`，`flags`成员设置为从`maskflags`函数返回的值，`array`成员设置为`NULL`，`alimit`成员设置为`0`。最后，通过调用`setnodevector`函数，将一个包含六个整数的节点数组设置为`t`对象的`metatable`成员的值。

最后，通过调用`lua_create`函数，将返回值存储到变量`L`中，表示成功创建了一个新的`Table`对象。


```cpp
/*
** }=============================================================
*/


Table *luaH_new (lua_State *L) {
  GCObject *o = luaC_newobj(L, LUA_VTABLE, sizeof(Table));
  Table *t = gco2t(o);
  t->metatable = NULL;
  t->flags = cast_byte(maskflags);  /* table has no metamethod fields */
  t->array = NULL;
  t->alimit = 0;
  setnodevector(L, t, 0);
  return t;
}


```



这段代码是Lua中的一个名为"luaH_free"的函数，用于释放由"table"结构体组成的与"luaH_realasize"函数返回的内存中的元素。

具体来说，代码中使用了以下几个步骤：

1. 首先，使用"freehash"函数将L和t对象存储的哈希表中的元素作为键，哈希表本身存储在L中。然后使用"luaM_freearray"函数，将L中的哈希表中的元素和对应的实际数据一起释放，这样可以确保所有的哈希表元素和它们对应的数据都被正确释放。

2. 接下来，使用"luaM_free"函数，从L中释放t对象存储的哈希表和L中存储的整个对象。

3. 在函数内部，使用一个名为"getfreepos"的静态函数，它接收一个"table"结构体作为参数。这个函数的作用是检查该哈希表是否为空，如果是，则返回当前 free 位置(即哈希表的最后一个 free 位置)，否则，函数将返回一个指向 free 位置的指针。

4. 最后，如果哈希表不是空，函数将遍历哈希表，直到找到第一个 free 位置。如果遍历完成后仍然没有找到 free 位置，函数返回一个 NULL 值。


```cpp
void luaH_free (lua_State *L, Table *t) {
  freehash(L, t);
  luaM_freearray(L, t->array, luaH_realasize(t));
  luaM_free(L, t);
}


static Node *getfreepos (Table *t) {
  if (!isdummy(t)) {
    while (t->lastfree > t->node) {
      t->lastfree--;
      if (keyisnil(t->lastfree))
        return t->lastfree;
    }
  }
  return NULL;  /* could not find a free place */
}



```

这段代码是一个 Lua 脚本，定义了一个名为 `Node` 的双向链表节点结构体。该结构体包含一个名为 `value` 的成员变量，以及一个名为 `next` 的成员函数指针，用于指向链表的下一个节点。

该链表节点结构体还包含一个名为 `free` 的成员变量，用于指示链表中每个节点是否是自由节点，即该节点可以用来插入新的数据。

在 `Node` 的 `next` 函数中，首先通过调用 `getfreepos` 函数获取一个免费位置，如果免费位置找到了，则执行以下操作：

1. 递归调用 `rehash` 函数来检查链表。
2. 如果链表长度大于 `maxnodes`，则需要重新分配内存，使用新的 `maxnodes` 参数。
3. 如果链表中当前节点不是空链，则执行以下操作：

a. 找到当前链表的下一个节点。
b. 移动当前节点到免费位置。
c. 更新 `free` 变量指示当前节点是否是免费节点。

如果免费位置找不到，或者链表本身就是空链，则执行以下操作：

1. 计算新键 `key` 在 `table` 中的偏移量 `offset`。
2. 创建新的节点并将其插入到链表的头部。
3. 设置链表的头部为 `value`。
4. 如果链表是空链，则需要设置链表的头部为 `value`。
5. 设置链表中的所有节点为 `value`。

该链表节点结构体的定义是在 `Node` 函数中，首先定义了 `Node` 结


```cpp
/*
** inserts a new key into a hash table; first, check whether key's main
** position is free. If not, check whether colliding node is in its main
** position or not: if it is not, move colliding node to an empty place and
** put new key in its main position; otherwise (colliding node is in its main
** position), new key goes to an empty position.
*/
void luaH_newkey (lua_State *L, Table *t, const TValue *key, TValue *value) {
  Node *mp;
  TValue aux;
  if (l_unlikely(ttisnil(key)))
    luaG_runerror(L, "table index is nil");
  else if (ttisfloat(key)) {
    lua_Number f = fltvalue(key);
    lua_Integer k;
    if (luaV_flttointeger(f, &k, F2Ieq)) {  /* does key fit in an integer? */
      setivalue(&aux, k);
      key = &aux;  /* insert it as an integer */
    }
    else if (l_unlikely(luai_numisnan(f)))
      luaG_runerror(L, "table index is NaN");
  }
  if (ttisnil(value))
    return;  /* do not insert nil values */
  mp = mainpositionTV(t, key);
  if (!isempty(gval(mp)) || isdummy(t)) {  /* main position is taken? */
    Node *othern;
    Node *f = getfreepos(t);  /* get a free place */
    if (f == NULL) {  /* cannot find a free place? */
      rehash(L, t, key);  /* grow table */
      /* whatever called 'newkey' takes care of TM cache */
      luaH_set(L, t, key, value);  /* insert key into grown table */
      return;
    }
    lua_assert(!isdummy(t));
    othern = mainpositionfromnode(t, mp);
    if (othern != mp) {  /* is colliding node out of its main position? */
      /* yes; move colliding node into free position */
      while (othern + gnext(othern) != mp)  /* find previous */
        othern += gnext(othern);
      gnext(othern) = cast_int(f - othern);  /* rechain to point to 'f' */
      *f = *mp;  /* copy colliding node into free pos. (mp->next also goes) */
      if (gnext(mp) != 0) {
        gnext(f) += cast_int(mp - f);  /* correct 'next' */
        gnext(mp) = 0;  /* now 'mp' is free */
      }
      setempty(gval(mp));
    }
    else {  /* colliding node is in its own main position */
      /* new node will go into free position */
      if (gnext(mp) != 0)
        gnext(f) = cast_int((mp + gnext(mp)) - f);  /* chain new position */
      else lua_assert(gnext(f) == 0);
      gnext(mp) = cast_int(f - mp);
      mp = f;
    }
  }
  setnodekey(L, mp, key);
  luaC_barrierback(L, obj2gco(t), key);
  lua_assert(isempty(gval(mp)));
  setobj2t(L, gval(mp), value);
}


```

这段代码是一个名为 `luaH_getint` 的函数，它用于在 Lua Script 表格中查找整数键。如果整数键 `key` 在表格的 `alimit` 字段中，函数直接返回该键。否则，函数会尝试在表格的 `array` 部分中查找该键。

为了避免频繁的 `luaH_realasize` 调用，当键仅仅是比 `alimit` 多一个时，函数会尝试使用内部方法 `hashint` 来查找。如果仍然无法找到，函数会将 `alimit` 字段设置为 `key`，并返回相应的键值。

该函数的实现基于 Lua 的类型系统。它使用了 `luaH_realasize` 函数来检查是否可以在 Lua 脚本中直接访问整数键。如果 `key` 在 Lua 脚本中是整数，并且它的键值在 [1, `alimit`] 范围内，那么函数将直接返回该键。如果 `key` 不属于这个范围，或者整数 `key` 的键值在 `alimit` 范围内，但是 `luaH_realasize` 无法访问该键，那么函数将返回相应的键值，或者返回一个指向 `Node` 对象的指针，指明该键在链表中的位置。


```cpp
/*
** Search function for integers. If integer is inside 'alimit', get it
** directly from the array part. Otherwise, if 'alimit' is not equal to
** the real size of the array, key still can be in the array part. In
** this case, try to avoid a call to 'luaH_realasize' when key is just
** one more than the limit (so that it can be incremented without
** changing the real size of the array).
*/
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


```

这段代码是一个Lua脚本中的函数，它的作用是查找一个短字符串（定义为只包含小写字母的字符串）的反义字符串。该函数接收两个参数：一个Lua表（table）和一个短字符串的关键字（定义为包含小写字母的字符串）。

函数实现了一个自定义的搜索表，首先通过哈希函数将关键字映射到表中的一个节点。然后，该节点一直递归地遍历，检查当前节点所包含的字符串是否与关键字相等。如果是，则返回该节点；否则，继续遍历，直到找到或者找到反义字符串（在搜索表中不存在）。

函数本身是有效的，但是它的实现有些复杂。它可能需要额外的检查来确保函数在所有情况下都能正常工作，比如在哈希函数发生冲突时。


```cpp
/*
** search function for short strings
*/
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


```

这段代码是一个Lua脚本，它的作用是根据自己的键（string或table）返回相应的Lua值。

首先，它定义了一个名为`luaH_getstr`的函数，它接受一个Lua表格（table）和一个Lua字符串（table）的键（key）作为参数。如果键的类型是LUA_VSHRSTR，函数使用`luaH_getshortstr`函数返回一个Lua短字符串；否则，它使用`getgeneric`函数，这个函数是Lua的一个通用的函数，用于根据要查找的键的类型选择正确的函数来返回Lua值。如果输入的键不匹配任何已知的键，它将返回一个名为`absentkey`的函数，这个函数会在后续的错误处理中使用。

接下来，它定义了一个名为`luaH_get`的函数，它接受一个Lua表格和一个Lua值（table）的键作为参数。如果键的类型是LUA_VSHRSTR，函数使用`luaH_getshortstr`函数返回相应的Lua短字符串；否则，它使用`getgeneric`函数，根据要查找的键的类型选择正确的函数来返回Lua值。

总结起来，这段代码定义了两个函数，第一个函数根据要查找的键的类型选择正确的函数来返回Lua值，第二个函数使用一个通用的函数来处理返回的Lua值。


```cpp
const TValue *luaH_getstr (Table *t, TString *key) {
  if (key->tt == LUA_VSHRSTR)
    return luaH_getshortstr(t, key);
  else {  /* for long strings, use generic case */
    TValue ko;
    setsvalue(cast(lua_State *, NULL), &ko, key);
    return getgeneric(t, &ko, 0);
  }
}


/*
** main search function
*/
const TValue *luaH_get (Table *t, const TValue *key) {
  switch (ttypetag(key)) {
    case LUA_VSHRSTR: return luaH_getshortstr(t, tsvalue(key));
    case LUA_VNUMINT: return luaH_getint(t, ivalue(key));
    case LUA_VNIL: return &absentkey;
    case LUA_VNUMFLT: {
      lua_Integer k;
      if (luaV_flttointeger(fltvalue(key), &k, F2Ieq)) /* integral index? */
        return luaH_getint(t, k);  /* use specialized version */
      /* else... */
    }  /* FALLTHROUGH */
    default:
      return getgeneric(t, key, 0);
  }
}


```

这段代码是一个Lua函数，名为`luaH_finishset`，定义在`luaH.h`文件中。它的作用是完成一个原始的"设置表"操作，其中`slot`是指在之前的"获取表"操作中`slot`所代表的内容。

首先，它检查`slot`是否是一个有效的键。如果是，那么它会尝试使用之前设置的键`key`和`value`来完成设置，并返回成功。否则，它会尝试使用之前设置的键`slot`和`value`来完成设置，并将结果存储到变量`value`中。

需要注意的是，在使用这个函数时，需要检查GC障碍，并 invalidateTM缓存。


```cpp
/*
** Finish a raw "set table" operation, where 'slot' is where the value
** should have been (the result of a previous "get table").
** Beware: when using this function you probably need to check a GC
** barrier and invalidate the TM cache.
*/
void luaH_finishset (lua_State *L, Table *t, const TValue *key,
                                   const TValue *slot, TValue *value) {
  if (isabstkey(slot))
    luaH_newkey(L, t, key, value);
  else
    setobj2t(L, cast(TValue *, slot), value);
}


```



这段代码定义了两个名为`luaH_set`和`luaH_setint`的函数，作用是设置表格中的某个键的值。

`luaH_set`函数的参数为三个：`lua_State`表示Lua脚本的状态，`Table`表示要设置的表格，以及要设置的键和对应的值。函数首先通过`luaH_get`函数获取要设置的键的值，然后使用`luaH_finishset`函数将修改后的键值对设置完成，最后返回成功设置的信息。

`luaH_setint`函数的参数与`luaH_set`函数相同，但使用了`luaH_newkey`函数来创建一个新的键值对。这个函数需要传入一个键的信息，包括键名和键类型。如果传入的键名在表格中不存在，则会创建一个新的键，并将新的键的值设置为输入的值。如果传入的键名在表格中存在，则会使用新的键来存储输入的值。


```cpp
/*
** beware: when using this function you probably need to check a GC
** barrier and invalidate the TM cache.
*/
void luaH_set (lua_State *L, Table *t, const TValue *key, TValue *value) {
  const TValue *slot = luaH_get(t, key);
  luaH_finishset(L, t, key, slot, value);
}


void luaH_setint (lua_State *L, Table *t, lua_Integer key, TValue *value) {
  const TValue *p = luaH_getint(t, key);
  if (isabstkey(p)) {
    TValue k;
    setivalue(&k, key);
    luaH_newkey(L, t, &k, value);
  }
  else
    setobj2t(L, cast(TValue *, p), value);
}


```

This code looks like it is implementing a binary search algorithm for a specific data structure (the table in this case). The table is implemented using the LuaH数组表示法， which allows for fast access to elements by index.

The key to this binary search algorithm is the "i" variable, which keeps track of the current index being searched. The "j" variable is the current index being searched through. The loop continues until the index is greater than or equal to the maximum integer value (堪称为'maxinteger')/2, which is a lowerbound of the boundary between present and absent keys.

If the current key is not found in the table, the code checks if the key is present in the table and returns the index if it is. If the key is present, the code performs a binary search between the current key and the maximum key in the table, and returns the index if the search is successful. If the binary search is not successful, the code will return the maximum integer value which is larger than the current key and the index.

The code also includes a do-while loop which will repeatedly double the 'j' variable until it reaches the maximum integer value or a missing key is found. This is done to ensure that the table does not exceed the maximum integer value, and also to improve the performance of the binary search algorithm.


```cpp
/*
** Try to find a boundary in the hash part of table 't'. From the
** caller, we know that 'j' is zero or present and that 'j + 1' is
** present. We want to find a larger key that is absent from the
** table, so that we can do a binary search between the two keys to
** find a boundary. We keep doubling 'j' until we get an absent index.
** If the doubling would overflow, we try LUA_MAXINTEGER. If it is
** absent, we are ready for the binary search. ('j', being max integer,
** is larger or equal to 'i', but it cannot be equal because it is
** absent while 'i' is present; so 'j > i'.) Otherwise, 'j' is a
** boundary. ('j + 1' cannot be a present integer key because it is
** not a valid integer in Lua.)
*/
static lua_Unsigned hash_search (Table *t, lua_Unsigned j) {
  lua_Unsigned i;
  if (j == 0) j++;  /* the caller ensures 'j + 1' is present */
  do {
    i = j;  /* 'i' is a present index */
    if (j <= l_castS2U(LUA_MAXINTEGER) / 2)
      j *= 2;
    else {
      j = LUA_MAXINTEGER;
      if (isempty(luaH_getint(t, j)))  /* t[j] not present? */
        break;  /* 'j' now is an absent index */
      else  /* weird case */
        return j;  /* well, max integer is a boundary... */
    }
  } while (!isempty(luaH_getint(t, j)));  /* repeat until an absent t[j] */
  /* i < j  &&  t[i] present  &&  t[j] absent */
  while (j - i > 1u) {  /* do a binary search between them */
    lua_Unsigned m = (i + j) / 2;
    if (isempty(luaH_getint(t, m))) j = m;
    else i = m;
  }
  return i;
}


```

这段代码定义了一个名为`binsearch`的函数，其作用是尝试在给定的数组`array`中查找一个边界值。具体来说，该函数将数组中的元素值存储在一个二叉搜索树中，然后从左到右遍历二叉搜索树，直到找到包含目标值的位置或者找到搜索树为空。

函数有两个参数：一个指向`TValue`类型数组的指针`array`，以及一个表示要搜索的数组长度`i`和另一个表示要搜索的数组长度`j`。函数返回一个整数，表示在数组中找到目标值时所在的位置。

该函数使用了二叉搜索树的结构，在树中找到目标值的位置后，将该位置返回。如果搜索树为空，则返回搜索范围内最小的可能位置。如果左边的子节点为空，则返回搜索范围内最大的可能位置。如果左边的子节点包含目标值，则返回该位置。如果左边的子节点包含最大整数值，则返回搜索范围内最大的整数值。


```cpp
static unsigned int binsearch (const TValue *array, unsigned int i,
                                                    unsigned int j) {
  while (j - i > 1u) {  /* binary search */
    unsigned int m = (i + j) / 2;
    if (isempty(&array[m - 1])) j = m;
    else i = m;
  }
  return i;
}


/*
** Try to find a boundary in table 't'. (A 'boundary' is an integer index
** such that t[i] is present and t[i+1] is absent, or 0 if t[1] is absent
** and 'maxinteger' if t[maxinteger] is present.)
```

这段代码是一个Lua脚本，它的作用是查找一个名为“t”的数组中一个名为“limit”的位置。它通过使用Lua中的索引，索引从0开始。

在这段注释中，作者解释了代码的功能，并且说明在解释代码之前需要了解Lua索引的基本概念。

具体来说，这段代码有以下几个主要步骤：

1. 如果“t[limit]”为空，那么必须在“limit”之前存在一个边界。
2. 如果“t[limit]”不为空且数组长度大于等于“limit”，那么尝试找到一个边界。如果还没有找到边界，就会执行以下操作：
  a. 如果“limit+1”也是空，那么尝试使用“limit”作为新的边界。
  b. 如果“limit+1”不是空，那么尝试使用二分搜索在0到“limit”之间找到一个边界。
3. 如果“t[limit]”既不是空也不包含“limit+1”，那么边界就是“limit”。否则，如果“limit-1”存在，就使用它作为新的边界。

总之，这段代码的主要目的是查找一个名为“t”的数组中一个名为“limit”的位置，然后根据不同的情况选择不同的查找边界的方法。


```cpp
** (In the next explanation, we use Lua indices, that is, with base 1.
** The code itself uses base 0 when indexing the array part of the table.)
** The code starts with 'limit = t->alimit', a position in the array
** part that may be a boundary.
**
** (1) If 't[limit]' is empty, there must be a boundary before it.
** As a common case (e.g., after 't[#t]=nil'), check whether 'limit-1'
** is present. If so, it is a boundary. Otherwise, do a binary search
** between 0 and limit to find a boundary. In both cases, try to
** use this boundary as the new 'alimit', as a hint for the next call.
**
** (2) If 't[limit]' is not empty and the array has more elements
** after 'limit', try to find a boundary there. Again, try first
** the special case (which should be quite frequent) where 'limit+1'
** is empty, so that 'limit' is a boundary. Otherwise, check the
```

1196：芒函
====

```cppruby
def芒函(t, limit):
   if isempty(t):
       return limit;  /* if t is empty, return the largest real number in the array */
   else:
       limit = luaH_realasize(t);
       if (limit == 0):  /* if the real number in the array is zero,
           return it */
       else:  /* if the real number in the array is non-zero, */
           return binsearch(t->array, 0, limit);  /* otherwise, search for the boundary */
       endif;
   return limit;  /* return the boundary */
}
```

760:repart函数
======

```cppjava
def repart(t, n):
   if isempty(t):
       return;  /* if t is empty, return it */
   else:
       s = luaH_realasize(t);
       if (s == 0):  /* if the first element in the array is zero, */
           return;  /* return it */
       else:
           l = repart(t, s);  /* repart the array by taking the remainder */
           l = ispower(l, 2);  /* and make it a power of two */
           l = llimit(l, 1);  /* if the result is too large, return the largest power of two that is less than or equal to the given number */
           return l;  /* else, return the newly created array */
       endif;
   return t;  /* return the modified array */
}
```

注意：函数芒函和repart均针对数值型(number)对象t，执行下列操作：
- 将t中的最大实数返回。
- 将t中的第一个实数及其平方根返回。
- 如果t中的第一个实数及其平方根都为零，则返回。
- 否则，通过调用自身函数形成t的新版本，并检查新版本是否符合要求。
- 如果新版本不符合要求，则返回原始t。


```cpp
** last element of the array part. If it is empty, there must be a
** boundary between the old limit (present) and the last element
** (absent), which is found with a binary search. (This boundary always
** can be a new limit.)
**
** (3) The last case is when there are no elements in the array part
** (limit == 0) or its last element (the new limit) is present.
** In this case, must check the hash part. If there is no hash part
** or 'limit+1' is absent, 'limit' is a boundary.  Otherwise, call
** 'hash_search' to find a boundary in the hash part of the table.
** (In those cases, the boundary is not inside the array part, and
** therefore cannot be used as a new limit.)
*/
lua_Unsigned luaH_getn (Table *t) {
  unsigned int limit = t->alimit;
  if (limit > 0 && isempty(&t->array[limit - 1])) {  /* (1)? */
    /* there must be a boundary before 'limit' */
    if (limit >= 2 && !isempty(&t->array[limit - 2])) {
      /* 'limit - 1' is a boundary; can it be a new limit? */
      if (ispow2realasize(t) && !ispow2(limit - 1)) {
        t->alimit = limit - 1;
        setnorealasize(t);  /* now 'alimit' is not the real size */
      }
      return limit - 1;
    }
    else {  /* must search for a boundary in [0, limit] */
      unsigned int boundary = binsearch(t->array, 0, limit);
      /* can this boundary represent the real size of the array? */
      if (ispow2realasize(t) && boundary > luaH_realasize(t) / 2) {
        t->alimit = boundary;  /* use it as the new limit */
        setnorealasize(t);
      }
      return boundary;
    }
  }
  /* 'limit' is zero or present in table */
  if (!limitequalsasize(t)) {  /* (2)? */
    /* 'limit' > 0 and array has more elements after 'limit' */
    if (isempty(&t->array[limit]))  /* 'limit + 1' is empty? */
      return limit;  /* this is the boundary */
    /* else, try last element in the array */
    limit = luaH_realasize(t);
    if (isempty(&t->array[limit - 1])) {  /* empty? */
      /* there must be a boundary in the array after old limit,
         and it must be a valid new limit */
      unsigned int boundary = binsearch(t->array, t->alimit, limit);
      t->alimit = boundary;
      return boundary;
    }
    /* else, new limit is present in the table; check the hash part */
  }
  /* (3) 'limit' is the last element and either is zero or present in table */
  lua_assert(limit == luaH_realasize(t) &&
             (limit == 0 || !isempty(&t->array[limit - 1])));
  if (isdummy(t) || isempty(luaH_getint(t, cast(lua_Integer, limit + 1))))
    return limit;  /* 'limit + 1' is absent */
  else  /* 'limit + 1' is also present */
    return hash_search(t, limit);
}



```

这段代码是一个Lua脚本，其中包含两个函数声明。

第一个函数声明是一个if语句的预处理指令，它会先检查是否定义了LUA_DEBUG变量。如果是，那么下面的代码块将会被跳过。否则，它将正常执行。

对于这两个函数声明，它们都使用了const类型的参数表，这意味着它们是作为函数的参数传递给函数的。第一个函数是一个名为"luaH_mainposition"的函数，它的参数表是一个指向Table类型的变量t和一个指向TValue类型的变量key。函数返回一个指向Node类型的变量，这个变量表示当前主机的状态。

第二个函数是一个名为"luaH_isdummy"的函数，它的参数表与第一个函数相同，不过它的返回类型被声明为int类型。函数的作用是判断一个Table是否为dummy，如果是，则返回true，否则返回false。

整个程序的作用是定义了两个函数，并将它们导出为测试库函数。这些函数可以被其他Lua脚本或程序调用，以实现对Lua脚本或程序的状态管理。


```cpp
#if defined(LUA_DEBUG)

/* export these functions for the test library */

Node *luaH_mainposition (const Table *t, const TValue *key) {
  return mainpositionTV(t, key);
}

int luaH_isdummy (const Table *t) { return isdummy(t); }

#endif

```