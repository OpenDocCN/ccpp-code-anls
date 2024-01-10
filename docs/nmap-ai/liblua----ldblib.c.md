# `nmap\liblua\ldblib.c`

```
/*
** $Id: ldblib.c $
** Interface from Lua to its debug API
** See Copyright Notice in lua.h
*/

// 定义宏 ldblib_c，用于标识 ldblib.c 文件
#define ldblib_c
// 定义宏 LUA_LIB，用于标识为 Lua 库
#define LUA_LIB

// 包含 Lua 的前缀文件
#include "lprefix.h"

// 包含标准输入输出库
#include <stdio.h>
// 包含标准库
#include <stdlib.h>
// 包含字符串处理库
#include <string.h>

// 包含 Lua 头文件
#include "lua.h"

// 包含 Lua 辅助库
#include "lauxlib.h"
// 包含 Lua 库
#include "lualib.h"

// 定义钩子表在注册表中的键值
static const char *const HOOKKEY = "_HOOKKEY";

/*
** If L1 != L, L1 can be in any state, and therefore there are no
** guarantees about its stack space; any push in L1 must be
** checked.
*/
// 检查栈空间，确保在不同状态的 Lua 状态机中进行栈操作时不会溢出
static void checkstack (lua_State *L, lua_State *L1, int n) {
  if (l_unlikely(L != L1 && !lua_checkstack(L1, n)))
    luaL_error(L, "stack overflow");
}

// 获取注册表
static int db_getregistry (lua_State *L) {
  lua_pushvalue(L, LUA_REGISTRYINDEX);
  return 1;
}

// 获取元表
static int db_getmetatable (lua_State *L) {
  luaL_checkany(L, 1);
  if (!lua_getmetatable(L, 1)) {
    lua_pushnil(L);  /* no metatable */
  }
  return 1;
}

// 设置元表
static int db_setmetatable (lua_State *L) {
  int t = lua_type(L, 2);
  luaL_argexpected(L, t == LUA_TNIL || t == LUA_TTABLE, 2, "nil or table");
  lua_settop(L, 2);
  lua_setmetatable(L, 1);
  return 1;  /* return 1st argument */
}

// 获取用户数据的值
static int db_getuservalue (lua_State *L) {
  int n = (int)luaL_optinteger(L, 2, 1);
  if (lua_type(L, 1) != LUA_TUSERDATA)
    luaL_pushfail(L);
  else if (lua_getiuservalue(L, 1, n) != LUA_TNONE) {
    lua_pushboolean(L, 1);
    return 2;
  }
  return 1;
}

// 设置用户数据的值
static int db_setuservalue (lua_State *L) {
  int n = (int)luaL_optinteger(L, 3, 1);
  luaL_checktype(L, 1, LUA_TUSERDATA);
  luaL_checkany(L, 2);
  lua_settop(L, 2);
  if (!lua_setiuservalue(L, 1, n))
    luaL_pushfail(L);
  return 1;
}

/*
** Auxiliary function used by several library functions: check for
** an optional thread as function's first argument and set 'arg' with
** 1 if this argument is present (so that functions can skip it to
** access their other arguments)
*/
// 辅助函数，用于检查可选的线程作为函数的第一个参数，并在存在该参数时设置 'arg' 为 1（以便函数可以跳过它来访问其他参数）
-- 获取线程，如果参数是线程，则返回该线程，否则返回当前线程
static lua_State *getthread (lua_State *L, int *arg) {
  if (lua_isthread(L, 1)) {
    *arg = 1;
    return lua_tothread(L, 1);
  }
  else {
    *arg = 0;
    return L;  -- 函数将在当前线程上操作
  }
}


-- 'lua_settable'的变体，用于在'db_getinfo'中将'lua_getinfo'的结果放入结果表中。键始终是字符串；值可以是字符串、整数或布尔值。
static void settabss (lua_State *L, const char *k, const char *v) {
  lua_pushstring(L, v);
  lua_setfield(L, -2, k);
}

static void settabsi (lua_State *L, const char *k, int v) {
  lua_pushinteger(L, v);
  lua_setfield(L, -2, k);
}

static void settabsb (lua_State *L, const char *k, int v) {
  lua_pushboolean(L, v);
  lua_setfield(L, -2, k);
}


-- 在函数'db_getinfo'中，调用'lua_getinfo'可能会将结果推送到堆栈上；稍后它会创建结果表以放置这些对象。函数'treatstackoption'将'lua_getinfo'的结果放在结果表的顶部，以便它可以调用'lua_setfield'。
static void treatstackoption (lua_State *L, lua_State *L1, const char *fname) {
  if (L == L1)
    lua_rotate(L, -2, 1);  -- 交换对象和表
  else
    lua_xmove(L1, L, 1);  -- 将对象移动到“主”堆栈
  lua_setfield(L, -2, fname);  -- 将对象放入表中
}


-- 调用'lua_getinfo'并将所有结果收集到一个新表中。L1需要堆栈空间来存放一个可选输入（函数）以及来自函数'lua_getinfo'的两个可选输出（函数和行表）。
static int db_getinfo (lua_State *L) {
  lua_Debug ar;
  int arg;
  lua_State *L1 = getthread(L, &arg);
  const char *options = luaL_optstring(L, arg+2, "flnSrtu");
  checkstack(L, L1, 3);
  luaL_argcheck(L, options[0] != '>', arg + 2, "invalid option '>'");
  if (lua_isfunction(L, arg + 1)) {  -- 关于函数的信息？
    options = lua_pushfstring(L, ">%s", options);  -- 在'options'中添加'>'
    lua_pushvalue(L, arg + 1);  /* 将栈中索引为 arg + 1 的值复制一份到 'L1' 栈中 */
    lua_xmove(L, L1, 1);  /* 将栈 L 中的一个值移动到栈 L1 中 */
  }
  else {  /* 如果不是函数 */
    if (!lua_getstack(L1, (int)luaL_checkinteger(L, arg + 1), &ar)) {  /* 获取调用栈信息 */
      luaL_pushfail(L);  /* 将失败信息推入栈 L */
      return 1;  /* 返回 1 */
    }
  }
  if (!lua_getinfo(L1, options, &ar))  /* 获取调用栈信息 */
    return luaL_argerror(L, arg+2, "invalid option");  /* 返回参数错误信息 */
  lua_newtable(L);  /* 创建一个新的表用于收集结果 */
  if (strchr(options, 'S')) {  /* 如果选项中包含 'S' */
    lua_pushlstring(L, ar.source, ar.srclen);  /* 将字符串推入栈 L */
    lua_setfield(L, -2, "source");  /* 设置表中字段的值 */
    settabss(L, "short_src", ar.short_src);  /* 设置表中字段的值 */
    settabsi(L, "linedefined", ar.linedefined);  /* 设置表中字段的值 */
    settabsi(L, "lastlinedefined", ar.lastlinedefined);  /* 设置表中字段的值 */
    settabss(L, "what", ar.what);  /* 设置表中字段的值 */
  }
  if (strchr(options, 'l'))  /* 如果选项中包含 'l' */
    settabsi(L, "currentline", ar.currentline);  /* 设置表中字段的值 */
  if (strchr(options, 'u')) {  /* 如果选项中包含 'u' */
    settabsi(L, "nups", ar.nups);  /* 设置表中字段的值 */
    settabsi(L, "nparams", ar.nparams);  /* 设置表中字段的值 */
    settabsb(L, "isvararg", ar.isvararg);  /* 设置表中字段的值 */
  }
  if (strchr(options, 'n')) {  /* 如果选项中包含 'n' */
    settabss(L, "name", ar.name);  /* 设置表中字段的值 */
    settabss(L, "namewhat", ar.namewhat);  /* 设置表中字段的值 */
  }
  if (strchr(options, 'r')) {  /* 如果选项中包含 'r' */
    settabsi(L, "ftransfer", ar.ftransfer);  /* 设置表中字段的值 */
    settabsi(L, "ntransfer", ar.ntransfer);  /* 设置表中字段的值 */
  }
  if (strchr(options, 't'))  /* 如果选项中包含 't' */
    settabsb(L, "istailcall", ar.istailcall);  /* 设置表中字段的值 */
  if (strchr(options, 'L'))  /* 如果选项中包含 'L' */
    treatstackoption(L, L1, "activelines");  /* 处理栈选项 */
  if (strchr(options, 'f'))  /* 如果选项中包含 'f' */
    treatstackoption(L, L1, "func");  /* 处理栈选项 */
  return 1;  /* 返回 1，即返回表 */
/*
** 从闭包中获取（如果 'get' 为真）或设置一个上值
*/

static int db_getlocal (lua_State *L) {
  int arg;
  lua_State *L1 = getthread(L, &arg);  // 获取线程
  int nvar = (int)luaL_checkinteger(L, arg + 2);  // 获取局部变量索引
  if (lua_isfunction(L, arg + 1)) {  // 函数参数？
    lua_pushvalue(L, arg + 1);  // 压入函数
    lua_pushstring(L, lua_getlocal(L, NULL, nvar));  // 压入局部变量名
    return 1;  // 只返回名称（没有值）
  }
  else {  // 堆栈级别参数
    lua_Debug ar;
    const char *name;
    int level = (int)luaL_checkinteger(L, arg + 1);
    if (l_unlikely(!lua_getstack(L1, level, &ar)))  // 超出范围？
      return luaL_argerror(L, arg+1, "level out of range");
    checkstack(L, L1, 1);
    name = lua_getlocal(L1, &ar, nvar);
    if (name) {
      lua_xmove(L1, L, 1);  // 移动局部值
      lua_pushstring(L, name);  // 压入名称
      lua_rotate(L, -2, 1);  // 重新排序
      return 2;
    }
    else {
      luaL_pushfail(L);  // 没有名称（也没有值）
      return 1;
    }
  }
}

static int db_setlocal (lua_State *L) {
  int arg;
  const char *name;
  lua_State *L1 = getthread(L, &arg);  // 获取线程
  lua_Debug ar;
  int level = (int)luaL_checkinteger(L, arg + 1);
  int nvar = (int)luaL_checkinteger(L, arg + 2);
  if (l_unlikely(!lua_getstack(L1, level, &ar)))  // 超出范围？
    return luaL_argerror(L, arg+1, "level out of range");
  luaL_checkany(L, arg+3);
  lua_settop(L, arg+3);
  checkstack(L, L1, 1);
  lua_xmove(L, L1, 1);
  name = lua_setlocal(L1, &ar, nvar);
  if (name == NULL)
    lua_pop(L1, 1);  // 弹出值（如果 'lua_setlocal' 没有弹出）
  lua_pushstring(L, name);
  return 1;
}
static int auxupvalue (lua_State *L, int get) {
  const char *name;
  int n = (int)luaL_checkinteger(L, 2);  /* upvalue index */  // 获取第二个参数作为闭包的上值索引
  luaL_checktype(L, 1, LUA_TFUNCTION);  /* closure */  // 检查第一个参数是否为函数类型
  name = get ? lua_getupvalue(L, 1, n) : lua_setupvalue(L, 1, n);  // 根据get参数调用不同的函数获取或设置闭包的上值，并将结果赋给name
  if (name == NULL) return 0;  // 如果name为空，返回0
  lua_pushstring(L, name);  // 将name压入栈中
  lua_insert(L, -(get+1));  /* no-op if get is false */  // 如果get为false，则不进行任何操作
  return get + 1;  // 返回get加1的值
}


static int db_getupvalue (lua_State *L) {
  return auxupvalue(L, 1);  // 调用auxupvalue函数，传入参数1
}


static int db_setupvalue (lua_State *L) {
  luaL_checkany(L, 3);  // 检查第三个参数是否为任意类型
  return auxupvalue(L, 0);  // 调用auxupvalue函数，传入参数0
}


/*
** Check whether a given upvalue from a given closure exists and
** returns its index
*/
static void *checkupval (lua_State *L, int argf, int argnup, int *pnup) {
  void *id;
  int nup = (int)luaL_checkinteger(L, argnup);  /* upvalue index */  // 获取第argnup个参数作为闭包的上值索引
  luaL_checktype(L, argf, LUA_TFUNCTION);  /* closure */  // 检查第argf个参数是否为函数类型
  id = lua_upvalueid(L, argf, nup);  // 获取闭包的上值的唯一标识
  if (pnup) {
    luaL_argcheck(L, id != NULL, argnup, "invalid upvalue index");  // 如果pnup不为空，检查id是否为空，如果为空则抛出错误
    *pnup = nup;  // 将nup的值赋给pnup指向的变量
  }
  return id;  // 返回闭包的上值的唯一标识
}


static int db_upvalueid (lua_State *L) {
  void *id = checkupval(L, 1, 2, NULL);  // 调用checkupval函数，传入参数1、2和NULL
  if (id != NULL)
    lua_pushlightuserdata(L, id);  // 如果id不为空，将id作为轻量级用户数据压入栈中
  else
    luaL_pushfail(L);  // 如果id为空，调用lual_pushfail函数
  return 1;  // 返回1
}


static int db_upvaluejoin (lua_State *L) {
  int n1, n2;
  checkupval(L, 1, 2, &n1);  // 调用checkupval函数，传入参数1、2和n1的地址
  checkupval(L, 3, 4, &n2);  // 调用checkupval函数，传入参数3、4和n2的地址
  luaL_argcheck(L, !lua_iscfunction(L, 1), 1, "Lua function expected");  // 检查第一个参数是否为Lua函数类型
  luaL_argcheck(L, !lua_iscfunction(L, 3), 3, "Lua function expected");  // 检查第三个参数是否为Lua函数类型
  lua_upvaluejoin(L, 1, n1, 3, n2);  // 将两个闭包的上值进行连接
  return 0;  // 返回0
}


/*
** Call hook function registered at hook table for the current
** thread (if there is one)
*/
static void hookf (lua_State *L, lua_Debug *ar) {
  static const char *const hooknames[] =
    {"call", "return", "line", "count", "tail call"};  // 定义hooknames数组
  lua_getfield(L, LUA_REGISTRYINDEX, HOOKKEY);  // 获取注册表中HOOKKEY对应的值
  lua_pushthread(L);  // 将当前线程压入栈中
  if (lua_rawget(L, -2) == LUA_TFUNCTION) {  /* is there a hook function? */  // 判断栈中的值是否为函数类型
    lua_pushstring(L, hooknames[(int)ar->event]);  /* push event name */  // 将事件名压入栈中
    # 如果当前行号大于等于0，则将当前行号压入栈中，否则压入nil
    if (ar->currentline >= 0)
      lua_pushinteger(L, ar->currentline);  /* push current line */
    # 获取当前函数的信息，包括行号和源文件名
    else lua_pushnil(L);
    lua_assert(lua_getinfo(L, "lS", ar));
    # 调用Lua函数，参数为2个，返回值为0个
    lua_call(L, 2, 0);  /* call hook function */
/* 
** 将字符串掩码（用于 'sethook'）转换为位掩码
*/
static int makemask (const char *smask, int count) {
  int mask = 0;
  if (strchr(smask, 'c')) mask |= LUA_MASKCALL;  // 如果字符串中包含 'c'，则将 LUA_MASKCALL 加入掩码
  if (strchr(smask, 'r')) mask |= LUA_MASKRET;   // 如果字符串中包含 'r'，则将 LUA_MASKRET 加入掩码
  if (strchr(smask, 'l')) mask |= LUA_MASKLINE;  // 如果字符串中包含 'l'，则将 LUA_MASKLINE 加入掩码
  if (count > 0) mask |= LUA_MASKCOUNT;          // 如果 count 大于 0，则将 LUA_MASKCOUNT 加入掩码
  return mask;  // 返回位掩码
}


/* 
** 将位掩码（用于 'gethook'）转换为字符串掩码
*/
static char *unmakemask (int mask, char *smask) {
  int i = 0;
  if (mask & LUA_MASKCALL) smask[i++] = 'c';  // 如果掩码中包含 LUA_MASKCALL，则将 'c' 加入字符串掩码
  if (mask & LUA_MASKRET) smask[i++] = 'r';   // 如果掩码中包含 LUA_MASKRET，则将 'r' 加入字符串掩码
  if (mask & LUA_MASKLINE) smask[i++] = 'l';  // 如果掩码中包含 LUA_MASKLINE，则将 'l' 加入字符串掩码
  smask[i] = '\0';  // 在字符串掩码的末尾添加结束符
  return smask;  // 返回字符串掩码
}


static int db_sethook (lua_State *L) {
  int arg, mask, count;
  lua_Hook func;
  lua_State *L1 = getthread(L, &arg);  // 获取线程
  if (lua_isnoneornil(L, arg+1)) {  /* no hook? */  // 如果没有设置 hook
    lua_settop(L, arg+1);  // 设置栈顶
    func = NULL; mask = 0; count = 0;  /* turn off hooks */  // 将 hook 关闭
  }
  else {
    const char *smask = luaL_checkstring(L, arg+2);  // 获取字符串掩码
    luaL_checktype(L, arg+1, LUA_TFUNCTION);  // 检查类型是否为函数
    count = (int)luaL_optinteger(L, arg + 3, 0);  // 获取可选的计数参数
    func = hookf; mask = makemask(smask, count);  // 设置函数和掩码
  }
  if (!luaL_getsubtable(L, LUA_REGISTRYINDEX, HOOKKEY)) {  // 如果没有获取到子表
    /* table just created; initialize it */  // 表刚刚创建；初始化它
    lua_pushliteral(L, "k");  // 压入字符串 "k"
    lua_setfield(L, -2, "__mode");  /** hooktable.__mode = "k" */  // 设置字段 __mode 为 "k"
    lua_pushvalue(L, -1);  // 复制栈顶元素
    lua_setmetatable(L, -2);  /* metatable(hooktable) = hooktable */  // 设置元表
  }
  checkstack(L, L1, 1);  // 检查栈
  lua_pushthread(L1); lua_xmove(L1, L, 1);  /* key (thread) */  // 将线程推入栈，并在两个线程之间移动值
  lua_pushvalue(L, arg + 1);  /* value (hook function) */  // 将 hook 函数推入栈
  lua_rawset(L, -3);  /* hooktable[L1] = new Lua hook */  // 设置 hooktable[L1] 为新的 Lua hook
  lua_sethook(L1, func, mask, count);  // 设置 hook
  return 0;  // 返回 0
}


static int db_gethook (lua_State *L) {
  int arg;
  lua_State *L1 = getthread(L, &arg);  // 获取线程
  char buff[5];  // 创建字符数组
  int mask = lua_gethookmask(L1);  // 获取 hook 掩码
  lua_Hook hook = lua_gethook(L1);  // 获取 hook 函数
  if (hook == NULL) {  /* no hook? */  // 如果没有 hook
    luaL_pushfail(L);  // 推入失败
    return 1;  // 返回 1
  }
  else if (hook != hookf)  /* external hook? */  // 如果 hook 不等于 hookf
    # 将字符串"external hook"压入 Lua 栈中
    lua_pushliteral(L, "external hook");
  else {  # hook table must exist
    # 获取全局表中键为 HOOKKEY 的值，将其压入 Lua 栈中
    lua_getfield(L, LUA_REGISTRYINDEX, HOOKKEY);
    # 检查 Lua 栈的大小，确保有足够的空间
    checkstack(L, L1, 1);
    # 将当前线程的状态机 L1 压入 Lua 栈中，然后从 L1 移动到 L 中
    lua_pushthread(L1); lua_xmove(L1, L, 1);
    # 从栈顶弹出一个值作为表的键，然后获取表中对应键的值，将其压入栈中
    lua_rawget(L, -2);   # 1st result = hooktable[L1]
    # 从栈中移除一个值
    lua_remove(L, -2);  # remove hook table
  }
  # 将解析后的掩码字符串压入 Lua 栈中作为第二个返回值
  lua_pushstring(L, unmakemask(mask, buff));  # 2nd result = mask
  # 将当前线程的钩子计数值压入 Lua 栈中作为第三个返回值
  lua_pushinteger(L, lua_gethookcount(L1));  # 3rd result = count
  # 返回 3 个结果
  return 3;
# 定义一个静态函数，用于调试
static int db_debug (lua_State *L) {
  # 进入无限循环，等待用户输入调试命令
  for (;;) {
    # 定义一个字符数组作为输入缓冲区
    char buffer[250];
    # 输出调试提示符
    lua_writestringerror("%s", "lua_debug> ");
    # 从标准输入读取用户输入的调试命令
    if (fgets(buffer, sizeof(buffer), stdin) == NULL ||
        strcmp(buffer, "cont\n") == 0)
      return 0;
    # 将用户输入的命令加载为 Lua 代码块，并执行
    if (luaL_loadbuffer(L, buffer, strlen(buffer), "=(debug command)") ||
        lua_pcall(L, 0, 0, 0))
      lua_writestringerror("%s\n", luaL_tolstring(L, -1, NULL));
    # 清空栈顶的返回值
    lua_settop(L, 0);  /* remove eventual returns */
  }
}

# 定义一个静态函数，用于生成调用栈信息
static int db_traceback (lua_State *L) {
  # 获取函数参数
  int arg;
  lua_State *L1 = getthread(L, &arg);
  const char *msg = lua_tostring(L, arg + 1);
  # 如果消息为空且不是空值，则将其原样返回
  if (msg == NULL && !lua_isnoneornil(L, arg + 1))  
    lua_pushvalue(L, arg + 1);  /* return it untouched */
  else {
    # 获取调用栈的层级
    int level = (int)luaL_optinteger(L, arg + 2, (L == L1) ? 1 : 0);
    # 生成调用栈信息
    luaL_traceback(L, L1, msg, level);
  }
  return 1;
}

# 定义一个静态函数，用于设置 C 栈的限制
static int db_setcstacklimit (lua_State *L) {
  # 获取参数中的限制值
  int limit = (int)luaL_checkinteger(L, 1);
  # 设置 C 栈的限制，并返回设置结果
  int res = lua_setcstacklimit(L, limit);
  lua_pushinteger(L, res);
  return 1;
}

# 定义一个 Lua 库，包含调试相关的函数
static const luaL_Reg dblib[] = {
  {"debug", db_debug},
  {"getuservalue", db_getuservalue},
  {"gethook", db_gethook},
  {"getinfo", db_getinfo},
  {"getlocal", db_getlocal},
  {"getregistry", db_getregistry},
  {"getmetatable", db_getmetatable},
  {"getupvalue", db_getupvalue},
  {"upvaluejoin", db_upvaluejoin},
  {"upvalueid", db_upvalueid},
  {"setuservalue", db_setuservalue},
  {"sethook", db_sethook},
  {"setlocal", db_setlocal},
  {"setmetatable", db_setmetatable},
  {"setupvalue", db_setupvalue},
  {"traceback", db_traceback},
  {"setcstacklimit", db_setcstacklimit},
  {NULL, NULL}
};

# 定义 Lua 模块，用于打开调试库
LUAMOD_API int luaopen_debug (lua_State *L) {
  # 创建新的 Lua 库，并将调试函数注册进去
  luaL_newlib(L, dblib);
  return 1;
}
```