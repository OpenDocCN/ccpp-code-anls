# `nmap\liblua\lcorolib.c`

```cpp
/*
** $Id: lcorolib.c $
** Coroutine Library
** See Copyright Notice in lua.h
*/

// 定义 lcorolib_c 宏，用于标识该文件是 coroutine 库的一部分
#define lcorolib_c
// 定义 LUA_LIB 宏，用于标识该文件是一个库
#define LUA_LIB

// 包含 lua.h 头文件
#include "lua.h"
// 包含 lauxlib.h 头文件
#include "lauxlib.h"
// 包含 lualib.h 头文件
#include "lualib.h"

// 定义获取协程的函数
static lua_State *getco (lua_State *L) {
  // 将栈上索引为 1 的值转换为线程类型
  lua_State *co = lua_tothread(L, 1);
  // 检查参数类型是否为线程类型，如果不是则抛出错误
  luaL_argexpected(L, co, 1, "thread");
  return co;
}

/*
** Resumes a coroutine. Returns the number of results for non-error
** cases or -1 for errors.
*/
// 辅助函数，用于恢复协程
static int auxresume (lua_State *L, lua_State *co, int narg) {
  int status, nres;
  // 如果栈空间不足以容纳 narg 个参数，则抛出错误
  if (l_unlikely(!lua_checkstack(co, narg))) {
    lua_pushliteral(L, "too many arguments to resume");
    return -1;  /* error flag */
  }
  // 将 L 中的 narg 个参数移动到协程 co 中
  lua_xmove(L, co, narg);
  // 恢复协程，获取状态和结果数量
  status = lua_resume(co, L, narg, &nres);
  // 如果状态为 LUA_OK 或 LUA_YIELD，则表示成功
  if (l_likely(status == LUA_OK || status == LUA_YIELD)) {
    // 如果栈空间不足以容纳 nres + 1 个参数，则抛出错误
    if (l_unlikely(!lua_checkstack(L, nres + 1))) {
      lua_pop(co, nres);  /* remove results anyway */
      lua_pushliteral(L, "too many results to resume");
      return -1;  /* error flag */
    }
    // 将协程 co 中的 nres 个结果移动到 L 中
    lua_xmove(co, L, nres);  /* move yielded values */
    return nres;
  }
  else {
    // 将协程 co 中的错误消息移动到 L 中
    lua_xmove(co, L, 1);  /* move error message */
    return -1;  /* error flag */
  }
}

// 用于实现 lua 中的 coroutine.resume 函数
static int luaB_coresume (lua_State *L) {
  // 获取协程
  lua_State *co = getco(L);
  int r;
  // 调用辅助函数 auxresume 恢复协程
  r = auxresume(L, co, lua_gettop(L) - 1);
  // 如果恢复失败，则返回错误信息
  if (l_unlikely(r < 0)) {
    lua_pushboolean(L, 0);
    lua_insert(L, -2);
    return 2;  /* return false + error message */
  }
  else {
    lua_pushboolean(L, 1);
    lua_insert(L, -(r + 1));
    return r + 1;  /* return true + 'resume' returns */
  }
}

// 用于实现 lua 中的 coroutine.wrap 函数
static int luaB_auxwrap (lua_State *L) {
  // 获取协程
  lua_State *co = lua_tothread(L, lua_upvalueindex(1));
  int r = auxresume(L, co, lua_gettop(L));
  // 如果恢复失败，则返回错误信息
  if (l_unlikely(r < 0)) {  /* error? */
    int stat = lua_status(co);
    # 如果协程中出现错误
    if (stat != LUA_OK && stat != LUA_YIELD) {  /* error in the coroutine? */
      # 重置协程的 tbc 变量
      stat = lua_resetthread(co);  /* close its tbc variables */
      # 确保重置后的状态不是 LUA_OK
      lua_assert(stat != LUA_OK);
      # 将错误消息移动到调用者的栈中
      lua_xmove(co, L, 1);  /* move error message to the caller */
    }
    # 如果不是内存错误并且错误对象是一个字符串
    if (stat != LUA_ERRMEM &&  /* not a memory error and ... */
        lua_type(L, -1) == LUA_TSTRING) {  /* ... error object is a string? */
      # 如果有额外信息，添加到错误消息中
      luaL_where(L, 1);  /* add extra info, if available */
      # 将额外信息插入到错误消息之前
      lua_insert(L, -2);
      # 将额外信息和错误消息连接起来
      lua_concat(L, 2);
    }
    # 传播错误
    return lua_error(L);  /* propagate error */
  }
  # 返回结果
  return r;
static int luaB_cocreate (lua_State *L) {
  // 检查参数1的类型是否为函数
  luaL_checktype(L, 1, LUA_TFUNCTION);
  // 创建一个新的线程
  lua_State *NL = lua_newthread(L);
  // 将参数1的函数移动到栈顶
  lua_pushvalue(L, 1);  /* move function to top */
  // 将参数1的函数从L移动到NL
  lua_xmove(L, NL, 1);  /* move function from L to NL */
  return 1;
}


static int luaB_cowrap (lua_State *L) {
  // 调用luaB_cocreate函数
  luaB_cocreate(L);
  // 将luaB_auxwrap函数作为闭包推入栈顶
  lua_pushcclosure(L, luaB_auxwrap, 1);
  return 1;
}


static int luaB_yield (lua_State *L) {
  // 返回一个yield函数
  return lua_yield(L, lua_gettop(L));
}


#define COS_RUN        0
#define COS_DEAD    1
#define COS_YIELD    2
#define COS_NORM    3


static const char *const statname[] =
  {"running", "dead", "suspended", "normal"};


static int auxstatus (lua_State *L, lua_State *co) {
  // 如果L和co相等，返回COS_RUN
  if (L == co) return COS_RUN;
  else {
    // 获取co的状态
    switch (lua_status(co)) {
      // 如果状态为LUA_YIELD，返回COS_YIELD
      case LUA_YIELD:
        return COS_YIELD;
      // 如果状态为LUA_OK
      case LUA_OK: {
        lua_Debug ar;
        // 是否有调用帧
        if (lua_getstack(co, 0, &ar))  
          return COS_NORM;  // 正在运行
        else if (lua_gettop(co) == 0)
            return COS_DEAD;
        else
          return COS_YIELD;  // 初始状态
      }
      // 其他错误状态，返回COS_DEAD
      default:  
        return COS_DEAD;
    }
  }
}


static int luaB_costatus (lua_State *L) {
  // 获取co状态并推入栈顶
  lua_State *co = getco(L);
  lua_pushstring(L, statname[auxstatus(L, co)]);
  return 1;
}


static int luaB_yieldable (lua_State *L) {
  // 获取co状态并推入栈顶
  lua_State *co = lua_isnone(L, 1) ? L : getco(L);
  lua_pushboolean(L, lua_isyieldable(co));
  return 1;
}


static int luaB_corunning (lua_State *L) {
  // 将当前线程推入栈顶
  int ismain = lua_pushthread(L);
  // 推入布尔值表示是否为主线程
  lua_pushboolean(L, ismain);
  return 2;
}


static int luaB_close (lua_State *L) {
  // 获取co状态
  lua_State *co = getco(L);
  int status = auxstatus(L, co);
  switch (status) {
    // 如果状态为COS_DEAD或COS_YIELD
    case COS_DEAD: case COS_YIELD: {
      // 重置线程状态
      status = lua_resetthread(co);
      if (status == LUA_OK) {
        lua_pushboolean(L, 1);
        return 1;
      }
      else {
        lua_pushboolean(L, 0);
        // 将错误消息从co移动到L
        lua_xmove(co, L, 1);  /* move error message */
        return 2;
      }
    }
    default:  /* normal or running coroutine */
      // 如果状态为正常或运行中的协程，则返回错误信息，说明不能关闭该类型的协程
      return luaL_error(L, "cannot close a %s coroutine", statname[status]);
  }
# 定义 Lua 协程模块的函数列表
static const luaL_Reg co_funcs[] = {
  {"create", luaB_cocreate},  # 创建一个新的协程
  {"resume", luaB_coresume},  # 恢复或启动一个协程
  {"running", luaB_corunning},  # 返回正在运行的协程
  {"status", luaB_costatus},  # 返回协程的状态
  {"wrap", luaB_cowrap},  # 将一个函数包装成一个协程
  {"yield", luaB_yield},  # 挂起当前协程
  {"isyieldable", luaB_yieldable},  # 检查当前协程是否可以被挂起
  {"close", luaB_close},  # 关闭一个协程
  {NULL, NULL}  # 结束函数列表
};

# 打开 Lua 协程模块，将函数列表注册到 Lua 环境中
LUAMOD_API int luaopen_coroutine (lua_State *L) {
  luaL_newlib(L, co_funcs);  # 创建新的 Lua 库，并将函数列表注册到其中
  return 1;  # 返回注册的库
}
```