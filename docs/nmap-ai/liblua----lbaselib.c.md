# `nmap\liblua\lbaselib.c`

```cpp
/*
** $Id: lbaselib.c $
** 基本库
** 请参阅 lua.h 中的版权声明
*/

#define lbaselib_c
#define LUA_LIB

#include "lprefix.h"


#include <ctype.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "lua.h"

#include "lauxlib.h"
#include "lualib.h"


static int luaB_print (lua_State *L) {
  int n = lua_gettop(L);  /* 参数个数 */
  int i;
  for (i = 1; i <= n; i++) {  /* 遍历每个参数 */
    size_t l;
    const char *s = luaL_tolstring(L, i, &l);  /* 转换为字符串 */
    if (i > 1)  /* 不是第一个元素？ */
      lua_writestring("\t", 1);  /* 在前面加一个制表符 */
    lua_writestring(s, l);  /* 打印参数 */
    lua_pop(L, 1);  /* 弹出结果 */
  }
  lua_writeline();
  return 0;
}


/*
** 使用所有给定的参数创建一个警告。
** 首先检查错误；否则，错误可能会中断警告的组成，使其无法完成。
*/
static int luaB_warn (lua_State *L) {
  int n = lua_gettop(L);  /* 参数个数 */
  int i;
  luaL_checkstring(L, 1);  /* 至少一个参数 */
  for (i = 2; i <= n; i++)
    luaL_checkstring(L, i);  /* 确保所有参数都是字符串 */
  for (i = 1; i < n; i++)  /* 组成警告 */
    lua_warning(L, lua_tostring(L, i), 1);
  lua_warning(L, lua_tostring(L, n), 0);  /* 关闭警告 */
  return 0;
}


#define SPACECHARS    " \f\n\r\t\v"

static const char *b_str2int (const char *s, int base, lua_Integer *pn) {
  lua_Unsigned n = 0;
  int neg = 0;
  s += strspn(s, SPACECHARS);  /* 跳过初始空格 */
  if (*s == '-') { s++; neg = 1; }  /* 处理符号 */
  else if (*s == '+') s++;
  if (!isalnum((unsigned char)*s))  /* 没有数字？ */
    return NULL;
  do {
    int digit = (isdigit((unsigned char)*s)) ? *s - '0'
                   : (toupper((unsigned char)*s) - 'A') + 10;
    if (digit >= base) return NULL;  /* 无效的数字 */
    n = n * base + digit;
    s++;  # 将指针 s 向后移动一位
  } while (isalnum((unsigned char)*s));  # 当指针 s 指向的字符是字母或数字时，继续循环
  s += strspn(s, SPACECHARS);  # 跳过字符串末尾的空格
  *pn = (lua_Integer)((neg) ? (0u - n) : n);  # 根据 neg 的值确定 *pn 的取值
  return s;  # 返回指针 s 的值
# 将 Lua 值转换为数字
static int luaB_tonumber (lua_State *L) {
  # 如果参数为空或为 nil，则进行标准转换
  if (lua_isnoneornil(L, 2)) {  
    # 如果第一个参数已经是数字，则直接返回
    if (lua_type(L, 1) == LUA_TNUMBER) {  
      lua_settop(L, 1);  # 将栈顶设置为第一个参数
      return 1;  # 返回 1
    }
    else {
      size_t l;
      const char *s = lua_tolstring(L, 1, &l);  # 将第一个参数转换为字符串
      # 如果成功将字符串转换为数字，则返回 1
      if (s != NULL && lua_stringtonumber(L, s) == l + 1)
        return 1;  
      # 否则，参数不是数字
      luaL_checkany(L, 1);  # 检查第一个参数是否为任意类型
    }
  }
  else {
    size_t l;
    const char *s;
    lua_Integer n = 0;  # 为了避免警告而定义的变量
    lua_Integer base = luaL_checkinteger(L, 2);  # 获取第二个参数作为基数
    luaL_checktype(L, 1, LUA_TSTRING);  # 第一个参数必须是字符串，不能是数字
    s = lua_tolstring(L, 1, &l);  # 将第一个参数转换为字符串
    luaL_argcheck(L, 2 <= base && base <= 36, 2, "base out of range");  # 检查基数是否在范围内
    # 如果成功将字符串转换为整数，则将整数压入栈顶并返回 1
    if (b_str2int(s, (int)base, &n) == s + l) {
      lua_pushinteger(L, n);
      return 1;
    }  # 否则，参数不是数字
  }  # 否则，参数不是数字
  luaL_pushfail(L);  # 将失败信息压入栈顶
  return 1;  # 返回 1
}


# 抛出 Lua 错误
static int luaB_error (lua_State *L) {
  int level = (int)luaL_optinteger(L, 2, 1);  # 获取第二个参数作为错误级别
  lua_settop(L, 1);  # 将栈顶设置为第一个参数
  # 如果第一个参数是字符串且错误级别大于 0，则添加额外信息
  if (lua_type(L, 1) == LUA_TSTRING && level > 0) {
    luaL_where(L, level);   # 添加额外信息
    lua_pushvalue(L, 1);  # 将第一个参数压入栈顶
    lua_concat(L, 2);  # 连接栈顶的两个值
  }
  return lua_error(L);  # 抛出 Lua 错误
}


# 获取元表
static int luaB_getmetatable (lua_State *L) {
  luaL_checkany(L, 1);  # 检查第一个参数是否为任意类型
  # 如果第一个参数没有元表，则将 nil 压入栈顶并返回 1
  if (!lua_getmetatable(L, 1)) {
    lua_pushnil(L);
    return 1;  
  }
  luaL_getmetafield(L, 1, "__metatable");  # 获取元表中的 __metatable 字段
  return 1;  # 返回 1，要么返回 __metatable 字段，要么返回元表
}


# 设置元表
static int luaB_setmetatable (lua_State *L) {
  int t = lua_type(L, 2);  # 获取第二个参数的类型
  luaL_checktype(L, 1, LUA_TTABLE);  # 第一个参数必须是表
  luaL_argexpected(L, t == LUA_TNIL || t == LUA_TTABLE, 2, "nil or table");  # 第二个参数必须是 nil 或表
  # 如果元表中存在 __metatable 字段，则返回错误
  if (l_unlikely(luaL_getmetafield(L, 1, "__metatable") != LUA_TNIL))
    # 返回 Lua 中的错误信息，指示无法更改受保护的元表
    return luaL_error(L, "cannot change a protected metatable");
    # 设置栈顶元素数量为 2
    lua_settop(L, 2);
    # 设置第一个参数的元表为第二个参数
    lua_setmetatable(L, 1);
    # 返回 1，表示成功
    return 1;
}

// 检查两个值是否严格相等
static int luaB_rawequal (lua_State *L) {
  luaL_checkany(L, 1);  // 检查参数1是否为有效值
  luaL_checkany(L, 2);  // 检查参数2是否为有效值
  lua_pushboolean(L, lua_rawequal(L, 1, 2));  // 将两个值进行严格比较，并将结果推入栈中
  return 1;  // 返回结果数量
}

// 获取表或字符串的长度
static int luaB_rawlen (lua_State *L) {
  int t = lua_type(L, 1);  // 获取参数1的类型
  luaL_argexpected(L, t == LUA_TTABLE || t == LUA_TSTRING, 1, "table or string");  // 检查参数1是否为表或字符串
  lua_pushinteger(L, lua_rawlen(L, 1));  // 将表或字符串的长度推入栈中
  return 1;  // 返回结果数量
}

// 获取表中指定键的值
static int luaB_rawget (lua_State *L) {
  luaL_checktype(L, 1, LUA_TTABLE);  // 检查参数1是否为表
  luaL_checkany(L, 2);  // 检查参数2是否为有效值
  lua_settop(L, 2);  // 设置栈顶为参数2
  lua_rawget(L, 1);  // 获取表中指定键的值
  return 1;  // 返回结果数量
}

// 设置表中指定键的值
static int luaB_rawset (lua_State *L) {
  luaL_checktype(L, 1, LUA_TTABLE);  // 检查参数1是否为表
  luaL_checkany(L, 2);  // 检查参数2是否为有效值
  luaL_checkany(L, 3);  // 检查参数3是否为有效值
  lua_settop(L, 3);  // 设置栈顶为参数3
  lua_rawset(L, 1);  // 设置表中指定键的值
  return 1;  // 返回结果数量
}

// 将旧的垃圾回收模式推入栈中
static int pushmode (lua_State *L, int oldmode) {
  if (oldmode == -1)
    luaL_pushfail(L);  // 如果旧模式为-1，则推入失败信息
  else
    lua_pushstring(L, (oldmode == LUA_GCINC) ? "incremental" : "generational");  // 根据旧模式推入相应的字符串
  return 1;  // 返回结果数量
}

/*
** 检查调用 'lua_gc' 是否有效（不在最终器中）
*/
#define checkvalres(res) { if (res == -1) break; }  // 检查结果是否为-1，如果是则跳出循环

// 执行垃圾回收操作
static int luaB_collectgarbage (lua_State *L) {
  static const char *const opts[] = {"stop", "restart", "collect", "count", "step", "setpause", "setstepmul", "isrunning", "generational", "incremental", NULL};  // 垃圾回收操作的选项
  static const int optsnum[] = {LUA_GCSTOP, LUA_GCRESTART, LUA_GCCOLLECT, LUA_GCCOUNT, LUA_GCSTEP, LUA_GCSETPAUSE, LUA_GCSETSTEPMUL, LUA_GCISRUNNING, LUA_GCGEN, LUA_GCINC};  // 垃圾回收操作的选项对应的值
  int o = optsnum[luaL_checkoption(L, 1, "collect", opts)];  // 获取选项对应的值
  switch (o) {
    case LUA_GCCOUNT: {
      int k = lua_gc(L, o);  // 获取垃圾回收的数量
      int b = lua_gc(L, LUA_GCCOUNTB);  // 获取垃圾回收的字节数
      checkvalres(k);  // 检查结果是否有效
      lua_pushnumber(L, (lua_Number)k + ((lua_Number)b/1024));  // 将垃圾回收的数量和字节数推入栈中
      return 1;  // 返回结果数量
    }
    case LUA_GCSTEP: {
      int step = (int)luaL_optinteger(L, 2, 0);  // 获取步长
      int res = lua_gc(L, o, step);  // 执行垃圾回收操作
      checkvalres(res);  // 检查结果是否有效
      lua_pushboolean(L, res);  // 将结果推入栈中
      return 1;  // 返回结果数量
    }
    case LUA_GCSETPAUSE:
    # 如果是 LUA_GCSETSTEPMUL 操作
    case LUA_GCSETSTEPMUL: {
      # 从栈中获取第二个参数，如果没有则默认为0，设置为整数类型
      int p = (int)luaL_optinteger(L, 2, 0);
      # 调用 lua_gc 函数执行垃圾回收，返回上一次的值
      int previous = lua_gc(L, o, p);
      # 检查返回值是否有效
      checkvalres(previous);
      # 将上一次的值压入栈中
      lua_pushinteger(L, previous);
      # 返回1表示成功
      return 1;
    }
    # 如果是 LUA_GCISRUNNING 操作
    case LUA_GCISRUNNING: {
      # 调用 lua_gc 函数执行垃圾回收，返回结果
      int res = lua_gc(L, o);
      # 检查返回值是否有效
      checkvalres(res);
      # 将结果转换为布尔值并压入栈中
      lua_pushboolean(L, res);
      # 返回1表示成功
      return 1;
    }
    # 如果是 LUA_GCGEN 操作
    case LUA_GCGEN: {
      # 从栈中获取第二个和第三个参数，如果没有则默认为0，设置为整数类型
      int minormul = (int)luaL_optinteger(L, 2, 0);
      int majormul = (int)luaL_optinteger(L, 3, 0);
      # 调用 lua_gc 函数执行垃圾回收，返回结果
      return pushmode(L, lua_gc(L, o, minormul, majormul));
    }
    # 如果是 LUA_GCINC 操作
    case LUA_GCINC: {
      # 从栈中获取第二个、第三个和第四个参数，如果没有则默认为0，设置为整数类型
      int pause = (int)luaL_optinteger(L, 2, 0);
      int stepmul = (int)luaL_optinteger(L, 3, 0);
      int stepsize = (int)luaL_optinteger(L, 4, 0);
      # 调用 lua_gc 函数执行垃圾回收，返回结果
      return pushmode(L, lua_gc(L, o, pause, stepmul, stepsize));
    }
    # 如果是其他操作
    default: {
      # 调用 lua_gc 函数执行垃圾回收，返回结果
      int res = lua_gc(L, o);
      # 检查返回值是否有效
      checkvalres(res);
      # 将结果压入栈中
      lua_pushinteger(L, res);
      # 返回1表示成功
      return 1;
    }
  }
  # 如果以上情况都不符合，则压入失败信息并返回1
  luaL_pushfail(L);  /* invalid call (inside a finalizer) */
  return 1;
static int luaB_type (lua_State *L) {
  // 获取参数的类型
  int t = lua_type(L, 1);
  // 检查参数是否为有效值
  luaL_argcheck(L, t != LUA_TNONE, 1, "value expected");
  // 将参数的类型名压入栈中
  lua_pushstring(L, lua_typename(L, t));
  // 返回1表示成功
  return 1;
}


static int luaB_next (lua_State *L) {
  // 检查参数是否为表
  luaL_checktype(L, 1, LUA_TTABLE);
  // 设置栈顶为第二个参数，如果没有则创建一个
  lua_settop(L, 2);  /* create a 2nd argument if there isn't one */
  // 遍历表，返回2表示找到下一个键值对，返回1表示遍历结束
  if (lua_next(L, 1))
    return 2;
  else {
    lua_pushnil(L);
    return 1;
  }
}


static int pairscont (lua_State *L, int status, lua_KContext k) {
  (void)L; (void)status; (void)k;  /* unused */
  // 返回3
  return 3;
}

static int luaB_pairs (lua_State *L) {
  // 检查参数是否为任意类型
  luaL_checkany(L, 1);
  // 获取元表中的__pairs字段，如果不存在则返回nil
  if (luaL_getmetafield(L, 1, "__pairs") == LUA_TNIL) {  /* no metamethod? */
    // 压入luaB_next函数
    lua_pushcfunction(L, luaB_next);  /* will return generator, */
    // 压入参数表
    lua_pushvalue(L, 1);  /* state, */
    // 压入nil
    lua_pushnil(L);  /* and initial value */
  }
  else {
    // 压入参数表
    lua_pushvalue(L, 1);  /* argument 'self' to metamethod */
    // 调用元方法
    lua_callk(L, 1, 3, 0, pairscont);  /* get 3 values from metamethod */
  }
  // 返回3
  return 3;
}


/*
** Traversal function for 'ipairs'
*/
static int ipairsaux (lua_State *L) {
  // 获取第二个参数作为索引
  lua_Integer i = luaL_checkinteger(L, 2);
  // 索引加1
  i = luaL_intop(+, i, 1);
  // 将新的索引压入栈中
  lua_pushinteger(L, i);
  // 获取表中索引为i的值，如果为nil则返回1，否则返回2
  return (lua_geti(L, 1, i) == LUA_TNIL) ? 1 : 2;
}


/*
** 'ipairs' function. Returns 'ipairsaux', given "table", 0.
** (The given "table" may not be a table.)
*/
static int luaB_ipairs (lua_State *L) {
  // 检查参数是否为任意类型
  luaL_checkany(L, 1);
  // 压入ipairsaux函数
  lua_pushcfunction(L, ipairsaux);  /* iteration function */
  // 压入参数表
  lua_pushvalue(L, 1);  /* state */
  // 压入初始值0
  lua_pushinteger(L, 0);  /* initial value */
  // 返回3
  return 3;
}


static int load_aux (lua_State *L, int status, int envidx) {
  // 如果状态为OK
  if (l_likely(status == LUA_OK)) {
    // 如果envidx不为0，将env作为加载函数的第一个upvalue
    if (envidx != 0) {  /* 'env' parameter? */
      // 将env压入栈中
      lua_pushvalue(L, envidx);  /* environment for loaded function */
      // 如果设置upvalue失败，则移除env
      if (!lua_setupvalue(L, -2, 1))  /* set it as 1st upvalue */
        lua_pop(L, 1);  /* remove 'env' if not used by previous call */
    }
    // 返回1表示成功
    return 1;
  }
  else {  /* error (message is on top of the stack) */
    # 将一个失败标记推入 Lua 栈顶
    luaL_pushfail(L);
    # 将栈顶的值插入到指定位置，这里是将失败标记放在错误消息之前
    lua_insert(L, -2);  /* put before error message */
    # 返回两个值，一个是失败标记，一个是错误消息
    return 2;  /* return fail plus error message */
  }
}


static int luaB_loadfile (lua_State *L) {
  // 从栈中获取第一个参数作为文件名，如果没有则为NULL
  const char *fname = luaL_optstring(L, 1, NULL);
  // 从栈中获取第二个参数作为模式，如果没有则为NULL
  const char *mode = luaL_optstring(L, 2, NULL);
  // 如果第三个参数不为空，则将env设为3，否则为0
  int env = (!lua_isnone(L, 3) ? 3 : 0);  /* 'env' index or 0 if no 'env' */
  // 调用luaL_loadfilex函数加载文件
  int status = luaL_loadfilex(L, fname, mode);
  // 调用load_aux函数处理加载结果
  return load_aux(L, status, env);
}


/*
** {======================================================
** Generic Read function
** =======================================================
*/


/*
** reserved slot, above all arguments, to hold a copy of the returned
** string to avoid it being collected while parsed. 'load' has four
** optional arguments (chunk, source name, mode, and environment).
*/
// 定义一个保留槽，用于保存返回的字符串，避免在解析时被回收
#define RESERVEDSLOT    5


/*
** Reader for generic 'load' function: 'lua_load' uses the
** stack for internal stuff, so the reader cannot change the
** stack top. Instead, it keeps its resulting string in a
** reserved slot inside the stack.
*/
// 通用的读取函数，用于'load'函数
static const char *generic_reader (lua_State *L, void *ud, size_t *size) {
  (void)(ud);  /* not used */
  // 检查栈是否有足够的空间
  luaL_checkstack(L, 2, "too many nested functions");
  // 将栈中的第一个参数（函数）压入栈顶
  lua_pushvalue(L, 1);  /* get function */
  // 调用函数
  lua_call(L, 0, 1);  /* call it */
  // 如果返回值为nil，则弹出结果，返回NULL
  if (lua_isnil(L, -1)) {
    lua_pop(L, 1);  /* pop result */
    *size = 0;
    return NULL;
  }
  // 如果返回值不是字符串，则报错
  else if (l_unlikely(!lua_isstring(L, -1)))
    luaL_error(L, "reader function must return a string");
  // 将返回的字符串保存在保留槽中
  lua_replace(L, RESERVEDSLOT);  /* save string in reserved slot */
  // 返回保留槽中的字符串
  return lua_tolstring(L, RESERVEDSLOT, size);
}


static int luaB_load (lua_State *L) {
  int status;
  size_t l;
  // 从栈中获取第一个参数作为字符串
  const char *s = lua_tolstring(L, 1, &l);
  // 从栈中获取第三个参数作为模式，如果没有则为"bt"
  const char *mode = luaL_optstring(L, 3, "bt");
  // 如果第四个参数不为空，则将env设为4，否则为0
  int env = (!lua_isnone(L, 4) ? 4 : 0);  /* 'env' index or 0 if no 'env' */
  // 如果s不为空，则加载字符串
  if (s != NULL) {  /* loading a string? */
    // 从栈中获取第二个参数作为chunkname，如果没有则为s
    const char *chunkname = luaL_optstring(L, 2, s);
    // 调用luaL_loadbufferx函数加载字符串
    status = luaL_loadbufferx(L, s, l, chunkname, mode);
  }
  // 否则从读取函数中加载
  else {  /* loading from a reader function */
    // 从栈中获取第二个参数作为chunkname，如果没有则为"=(load)"
    const char *chunkname = luaL_optstring(L, 2, "=(load)");
    # 检查栈中第一个参数的类型是否为函数，如果不是则抛出错误
    luaL_checktype(L, 1, LUA_TFUNCTION);
    # 设置栈顶元素的位置为预留槽的位置，用于创建预留槽
    lua_settop(L, RESERVEDSLOT);  /* create reserved slot */
    # 载入 Lua 代码块，使用指定的读取器函数和模式
    status = lua_load(L, generic_reader, NULL, chunkname, mode);
  }
  # 调用辅助函数进行加载操作的后续处理
  return load_aux(L, status, env);
/* }====================================================== */

/* 定义一个静态函数，用于处理文件内容，返回栈顶元素数量减一 */
static int dofilecont (lua_State *L, int d1, lua_KContext d2) {
  (void)d1;  (void)d2;  /* 只是为了匹配 'lua_Kfunction' 原型 */
  return lua_gettop(L) - 1;
}

/* 定义 Lua 函数，用于执行指定文件 */
static int luaB_dofile (lua_State *L) {
  const char *fname = luaL_optstring(L, 1, NULL);  /* 获取文件名参数 */
  lua_settop(L, 1);  /* 设置栈顶为 1 */
  if (l_unlikely(luaL_loadfile(L, fname) != LUA_OK))  /* 如果加载文件失败 */
    return lua_error(L);  /* 返回错误 */
  lua_callk(L, 0, LUA_MULTRET, 0, dofilecont);  /* 调用函数，使用指定的 continuation 函数 */
  return dofilecont(L, 0, 0);  /* 返回 continuation 函数的结果 */
}

/* 定义 Lua 函数，用于断言条件是否为真 */
static int luaB_assert (lua_State *L) {
  if (l_likely(lua_toboolean(L, 1)))  /* 条件为真 */
    return lua_gettop(L);  /* 返回所有参数 */
  else {  /* 错误 */
    luaL_checkany(L, 1);  /* 必须有一个条件 */
    lua_remove(L, 1);  /* 移除条件 */
    lua_pushliteral(L, "assertion failed!");  /* 默认消息 */
    lua_settop(L, 1);  /* 只保留消息（如果没有其他消息，则使用默认消息） */
    return luaB_error(L);  /* 调用 'error' 函数 */
  }
}

/* 定义 Lua 函数，用于选择参数 */
static int luaB_select (lua_State *L) {
  int n = lua_gettop(L);  /* 获取参数数量 */
  if (lua_type(L, 1) == LUA_TSTRING && *lua_tostring(L, 1) == '#') {  /* 如果第一个参数是字符串且以 '#' 开头 */
    lua_pushinteger(L, n-1);  /* 压入参数数量减一 */
    return 1;  /* 返回 1 */
  }
  else {
    lua_Integer i = luaL_checkinteger(L, 1);  /* 获取整数参数 */
    if (i < 0) i = n + i;  /* 如果参数小于 0，则加上参数数量 */
    else if (i > n) i = n;  /* 如果参数大于数量，则取参数数量 */
    luaL_argcheck(L, 1 <= i, 1, "index out of range");  /* 检查参数范围 */
    return n - (int)i;  /* 返回参数数量减去整数参数 */
  }
}

/*
** 'pcall' 和 'xpcall' 的 continuation 函数。这两个函数在调用之前已经压入了一个 'true'，
** 所以在成功的情况下，'finishpcall' 只需要返回栈中的所有内容减去 'extra' 个值
** （其中 'extra' 正好是要忽略的项目数量）。
*/
static int finishpcall (lua_State *L, int status, lua_KContext extra) {
  if (l_unlikely(status != LUA_OK && status != LUA_YIELD)) {  /* 错误？ */
    lua_pushboolean(L, 0);  /* 第一个结果（false） */
    lua_pushvalue(L, -2);  /* 错误消息 */
    return 2;  /* 返回 false, msg */
  }
  else
    # 返回 Lua 栈顶索引减去额外参数的数量，即返回所有结果
    return lua_gettop(L) - (int)extra;  /* return all results */
# 执行受保护的函数调用，处理错误
static int luaB_pcall (lua_State *L) {
  int status;
  luaL_checkany(L, 1);  # 检查第一个参数是否存在
  lua_pushboolean(L, 1);  # 如果没有错误，将第一个结果设为 true
  lua_insert(L, 1);  # 将第一个结果放到正确的位置
  status = lua_pcallk(L, lua_gettop(L) - 2, LUA_MULTRET, 0, 0, finishpcall);  # 调用受保护的函数，返回状态
  return finishpcall(L, status, 0);  # 调用完成函数，返回结果
}

'''
** 执行带有错误处理的受保护调用。在 'lua_rotate' 之后，堆栈将具有 <f, err, true, f, [args...]>；因此，当返回结果时，函数将传递 2 给 'finishpcall' 来跳过前两个值。
'''
static int luaB_xpcall (lua_State *L) {
  int status;
  int n = lua_gettop(L);
  luaL_checktype(L, 2, LUA_TFUNCTION);  # 检查错误函数的类型
  lua_pushboolean(L, 1);  # 第一个结果
  lua_pushvalue(L, 1);  # 函数
  lua_rotate(L, 3, 2);  # 将它们移动到函数参数的下面
  status = lua_pcallk(L, n - 2, LUA_MULTRET, 2, 2, finishpcall);  # 调用受保护的函数，返回状态
  return finishpcall(L, status, 2);  # 调用完成函数，返回结果
}

# 将参数转换为字符串
static int luaB_tostring (lua_State *L) {
  luaL_checkany(L, 1);  # 检查第一个参数是否存在
  luaL_tolstring(L, 1, NULL);  # 将参数转换为字符串
  return 1;  # 返回结果
}

# 基本函数的注册表
static const luaL_Reg base_funcs[] = {
  {"assert", luaB_assert},
  {"collectgarbage", luaB_collectgarbage},
  {"dofile", luaB_dofile},
  {"error", luaB_error},
  {"getmetatable", luaB_getmetatable},
  {"ipairs", luaB_ipairs},
  {"loadfile", luaB_loadfile},
  {"load", luaB_load},
  {"next", luaB_next},
  {"pairs", luaB_pairs},
  {"pcall", luaB_pcall},
  {"print", luaB_print},
  {"warn", luaB_warn},
  {"rawequal", luaB_rawequal},
  {"rawlen", luaB_rawlen},
  {"rawget", luaB_rawget},
  {"rawset", luaB_rawset},
  {"select", luaB_select},
  {"setmetatable", luaB_setmetatable},
  {"tonumber", luaB_tonumber},
  {"tostring", luaB_tostring},
  {"type", luaB_type},
  {"xpcall", luaB_xpcall},
  /* placeholders */
  {LUA_GNAME, NULL},
  {"_VERSION", NULL},
  {NULL, NULL}
};
# 使用 Lua API 打开基础库，将其加载到全局表中
LUAMOD_API int luaopen_base (lua_State *L) {
  # 将全局表压入栈顶
  lua_pushglobaltable(L);
  # 将基础库函数设置到全局表中
  luaL_setfuncs(L, base_funcs, 0);
  # 设置全局变量 _G
  lua_pushvalue(L, -1);
  lua_setfield(L, -2, LUA_GNAME);
  # 设置全局变量 _VERSION
  lua_pushliteral(L, LUA_VERSION);
  lua_setfield(L, -2, "_VERSION");
  # 返回 1 表示加载成功
  return 1;
}
```