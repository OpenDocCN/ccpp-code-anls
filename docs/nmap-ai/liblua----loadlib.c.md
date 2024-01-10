# `nmap\liblua\loadlib.c`

```
/*
** $Id: loadlib.c $
** Dynamic library loader for Lua
** See Copyright Notice in lua.h
**
** This module contains an implementation of loadlib for Unix systems
** that have dlfcn, an implementation for Windows, and a stub for other
** systems.
*/

#define loadlib_c
#define LUA_LIB

#include "lprefix.h"


#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "lua.h"

#include "lauxlib.h"
#include "lualib.h"


/*
** LUA_IGMARK is a mark to ignore all before it when building the
** luaopen_ function name.
*/
#if !defined (LUA_IGMARK)
#define LUA_IGMARK        "-"
#endif


/*
** LUA_CSUBSEP is the character that replaces dots in submodule names
** when searching for a C loader.
** LUA_LSUBSEP is the character that replaces dots in submodule names
** when searching for a Lua loader.
*/
#if !defined(LUA_CSUBSEP)
#define LUA_CSUBSEP        LUA_DIRSEP
#endif

#if !defined(LUA_LSUBSEP)
#define LUA_LSUBSEP        LUA_DIRSEP
#endif


/* prefix for open functions in C libraries */
#define LUA_POF        "luaopen_"

/* separator for open functions in C libraries */
#define LUA_OFSEP    "_"


/*
** key for table in the registry that keeps handles
** for all loaded C libraries
*/
static const char *const CLIBS = "_CLIBS";

#define LIB_FAIL    "open"


#define setprogdir(L)           ((void)0)


/*
** Special type equivalent to '(void*)' for functions in gcc
** (to suppress warnings when converting function pointers)
*/
typedef void (*voidf)(void);


/*
** system-dependent functions
*/

/*
** unload library 'lib'
*/
static void lsys_unloadlib (void *lib);

/*
** load C library in file 'path'. If 'seeglb', load with all names in
** the library global.
** Returns the library; in case of error, returns NULL plus an
** error string in the stack.
*/
static void *lsys_load (lua_State *L, const char *path, int seeglb);

/*
** Try to find a function named 'sym' in library 'lib'.
** Returns the function; in case of error, returns NULL plus
** an error string in the stack.
*/
static lua_CFunction lsys_sym (lua_State *L, void *lib, const char *sym);
/*
** 设置程序目录为可执行文件的路径
*/
static void setprogdir (lua_State *L) {
  char buff[MAX_PATH + 1];  // 定义一个字符数组用于存储路径
  char *lb;  // 定义一个字符指针
  DWORD nsize = sizeof(buff)/sizeof(char);  // 获取字符数组的大小
  DWORD n = GetModuleFileNameA(NULL, buff, nsize);  // 获取可执行文件的名称
  if (n == 0 || n == nsize || (lb = strrchr(buff, '\\')) == NULL)  // 判断获取文件名是否成功
    luaL_error(L, "unable to get ModuleFileName");  // 抛出 Lua 错误
  else {
    *lb = '\0';  // 截取最后一个 '\\' 之前的路径
    luaL_gsub(L, lua_tostring(L, -1), LUA_EXEC_DIR, buff);  // 替换字符串中的 LUA_EXEC_DIR 为 buff
    lua_remove(L, -2);  // 移除原始字符串
  }
}

/*
** 将系统错误信息推入 Lua 栈
*/
static void pusherror (lua_State *L) {
  int error = GetLastError();  // 获取系统错误码
  char buffer[128];  // 定义一个字符数组用于存储错误信息
  if (FormatMessageA(FORMAT_MESSAGE_IGNORE_INSERTS | FORMAT_MESSAGE_FROM_SYSTEM,
      NULL, error, 0, buffer, sizeof(buffer)/sizeof(char), NULL))  // 获取系统错误信息
    lua_pushstring(L, buffer);  // 将错误信息推入 Lua 栈
  else
    lua_pushfstring(L, "system error %d\n", error);  // 将错误信息推入 Lua 栈
}

/*
** 释放动态链接库
*/
static void lsys_unloadlib (void *lib) {
  FreeLibrary((HMODULE)lib);  // 释放动态链接库
}

/*
** 加载动态链接库
*/
static void *lsys_load (lua_State *L, const char *path, int seeglb) {
  HMODULE lib = LoadLibraryExA(path, NULL, LUA_LLE_FLAGS);  // 加载动态链接库
  (void)(seeglb);  // 不使用：符号默认为 'global'
  if (lib == NULL) pusherror(L);  // 如果加载失败，推入错误信息到 Lua 栈
  return lib;  // 返回加载的动态链接库
}

/*
** 获取动态链接库中的符号
*/
static lua_CFunction lsys_sym (lua_State *L, void *lib, const char *sym) {
  lua_CFunction f = (lua_CFunction)(voidf)GetProcAddress((HMODULE)lib, sym);  // 获取动态链接库中的符号
  if (f == NULL) pusherror(L);  // 如果获取失败，推入错误信息到 Lua 栈
  return f;  // 返回获取的符号
}

/* }====================================================== */

#else                /* }{ */
/*
** {======================================================
** 其他系统的回退方案
** =======================================================
*/

#undef LIB_FAIL
#define LIB_FAIL    "absent"  // 定义动态链接库加载失败的提示信息

#define DLMSG    "dynamic libraries not enabled; check your Lua installation"  // 定义动态链接库未启用的提示信息

/*
** 释放动态链接库
*/
static void lsys_unloadlib (void *lib) {
  (void)(lib);  // 不使用
}
static void *lsys_load (lua_State *L, const char *path, int seeglb) {
  (void)(path); (void)(seeglb);  /* not used */
  lua_pushliteral(L, DLMSG);
  return NULL;
}


static lua_CFunction lsys_sym (lua_State *L, void *lib, const char *sym) {
  (void)(lib); (void)(sym);  /* not used */
  lua_pushliteral(L, DLMSG);
  return NULL;
}


/* }====================================================== */
#endif                /* } */


/*
** {==================================================================
** Set Paths
** ===================================================================
*/


/*
** LUA_PATH_VAR and LUA_CPATH_VAR are the names of the environment
** variables that Lua check to set its paths.
*/
#if !defined(LUA_PATH_VAR)
#define LUA_PATH_VAR    "LUA_PATH"
#endif

#if !defined(LUA_CPATH_VAR)
#define LUA_CPATH_VAR   "LUA_CPATH"
#endif


/*
** return registry.LUA_NOENV as a boolean
*/
static int noenv (lua_State *L) {
  int b;
  lua_getfield(L, LUA_REGISTRYINDEX, "LUA_NOENV");
  b = lua_toboolean(L, -1);
  lua_pop(L, 1);  /* remove value */
  return b;
}


/*
** Set a path
*/
static void setpath (lua_State *L, const char *fieldname,
                                   const char *envname,
                                   const char *dft) {
  const char *dftmark;
  const char *nver = lua_pushfstring(L, "%s%s", envname, LUA_VERSUFFIX);
  const char *path = getenv(nver);  /* try versioned name */
  if (path == NULL)  /* no versioned environment variable? */
    path = getenv(envname);  /* try unversioned name */
  if (path == NULL || noenv(L))  /* no environment variable? */
    lua_pushstring(L, dft);  /* use default */
  else if ((dftmark = strstr(path, LUA_PATH_SEP LUA_PATH_SEP)) == NULL)
    lua_pushstring(L, path);  /* nothing to change */
  else {  /* path contains a ";;": insert default path in its place */
    size_t len = strlen(path);
    luaL_Buffer b;
    luaL_buffinit(L, &b);
    if (path < dftmark) {  /* 检查是否存在';;'之前的前缀 */
      luaL_addlstring(&b, path, dftmark - path);  /* 将前缀添加到缓冲区 */
      luaL_addchar(&b, *LUA_PATH_SEP);  /* 在缓冲区中添加路径分隔符 */
    }
    luaL_addstring(&b, dft);  /* 将默认路径添加到缓冲区 */
    if (dftmark < path + len - 2) {  /* 检查是否存在';;'之后的后缀 */
      luaL_addchar(&b, *LUA_PATH_SEP);  /* 在缓冲区中添加路径分隔符 */
      luaL_addlstring(&b, dftmark + 2, (path + len - 2) - dftmark);  /* 将后缀添加到缓冲区 */
    }
    luaL_pushresult(&b);  /* 将缓冲区中的内容推入栈中 */
  }
  setprogdir(L);  /* 设置程序目录 */
  lua_setfield(L, -3, fieldname);  /* 设置 package[fieldname] 的值为路径 */
  lua_pop(L, 1);  /* 弹出版本化的变量名 ('nver') */
/* }================================================================== */

/*
** return registry.CLIBS[path]
*/
static void *checkclib (lua_State *L, const char *path) {
  void *plib;
  lua_getfield(L, LUA_REGISTRYINDEX, CLIBS);  /* 获取注册表中的CLIBS */
  lua_getfield(L, -1, path);  /* 获取CLIBS[path] */
  plib = lua_touserdata(L, -1);  /* 将CLIBS[path]转换为用户数据类型 */
  lua_pop(L, 2);  /* 弹出CLIBS表和'plib' */
  return plib;  /* 返回plib */
}

/*
** registry.CLIBS[path] = plib        -- for queries
** registry.CLIBS[#CLIBS + 1] = plib  -- also keep a list of all libraries
*/
static void addtoclib (lua_State *L, const char *path, void *plib) {
  lua_getfield(L, LUA_REGISTRYINDEX, CLIBS);  /* 获取注册表中的CLIBS */
  lua_pushlightuserdata(L, plib);  /* 将plib作为轻量用户数据压入栈中 */
  lua_pushvalue(L, -1);  /* 复制栈顶元素 */
  lua_setfield(L, -3, path);  /* 设置CLIBS[path] = plib */
  lua_rawseti(L, -2, luaL_len(L, -2) + 1);  /* 设置CLIBS[#CLIBS + 1] = plib */
  lua_pop(L, 1);  /* 弹出CLIBS表 */
}

/*
** __gc tag method for CLIBS table: calls 'lsys_unloadlib' for all lib
** handles in list CLIBS
*/
static int gctm (lua_State *L) {
  lua_Integer n = luaL_len(L, 1);  /* 获取表的长度 */
  for (; n >= 1; n--) {  /* 对于每个句柄，倒序遍历 */
    lua_rawgeti(L, 1, n);  /* 获取句柄CLIBS[n] */
    lsys_unloadlib(lua_touserdata(L, -1));  /* 调用lsys_unloadlib卸载库 */
    lua_pop(L, 1);  /* 弹出句柄 */
  }
  return 0;
}

/* error codes for 'lookforfunc' */
#define ERRLIB        1  /* 库错误代码 */
#define ERRFUNC        2  /* 函数错误代码 */

/*
** Look for a C function named 'sym' in a dynamically loaded library
** 'path'.
** First, check whether the library is already loaded; if not, try
** to load it.
** Then, if 'sym' is '*', return true (as library has been loaded).
** Otherwise, look for symbol 'sym' in the library and push a
** C function with that symbol.
** Return 0 and 'true' or a function in the stack; in case of
** errors, return an error code and an error message in the stack.
*/
static int lookforfunc (lua_State *L, const char *path, const char *sym) {
  void *reg = checkclib(L, path);  /* 检查已加载的C库 */
  if (reg == NULL) {  /* 必须加载库？ */
    // 载入动态链接库中的函数或全局符号
    reg = lsys_load(L, path, *sym == '*');  /* global symbols if 'sym'=='*' */
    // 如果无法载入库，则返回错误码
    if (reg == NULL) return ERRLIB;  /* unable to load library */
    // 将载入的库添加到全局库列表中
    addtoclib(L, path, reg);
  }
  // 如果要载入的符号为'*'，表示只载入库而不载入函数
  if (*sym == '*') {  /* loading only library (no function)? */
    // 将布尔值true压入栈中
    lua_pushboolean(L, 1);  /* return 'true' */
    // 返回0表示没有错误
    return 0;  /* no errors */
  }
  else {
    // 在载入的库中查找指定的函数
    lua_CFunction f = lsys_sym(L, reg, sym);
    // 如果找不到函数，则返回错误码
    if (f == NULL)
      return ERRFUNC;  /* unable to find function */
    // 否则将找到的函数压入栈中
    lua_pushcfunction(L, f);  /* else create new function */
    // 返回0表示没有错误
    return 0;  /* no errors */
  }
}

// 加载动态链接库的函数
static int ll_loadlib (lua_State *L) {
  // 获取第一个参数作为路径
  const char *path = luaL_checkstring(L, 1);
  // 获取第二个参数作为初始化函数名
  const char *init = luaL_checkstring(L, 2);
  // 在指定路径中查找初始化函数
  int stat = lookforfunc(L, path, init);
  // 如果查找成功
  if (l_likely(stat == 0))  /* no errors? */
    // 返回加载的函数
    return 1;  /* return the loaded function */
  else {  /* error; error message is on stack top */
    // 将失败信息推入栈顶
    luaL_pushfail(L);
    // 将失败信息插入到初始化函数之前
    lua_insert(L, -2);
    // 根据错误类型推入相应的错误信息
    lua_pushstring(L, (stat == ERRLIB) ?  LIB_FAIL : "init");
    // 返回失败信息、错误消息和错误位置
    return 3;  /* return fail, error message, and where */
  }
}

/*
** {======================================================
** 'require' function
** =======================================================
*/

// 检查文件是否可读
static int readable (const char *filename) {
  // 尝试打开文件
  FILE *f = fopen(filename, "r");
  // 如果打开失败
  if (f == NULL) return 0;  /* open failed */
  // 关闭文件
  fclose(f);
  // 返回文件是否可读
  return 1;
}

// 获取路径中的下一个文件名
static const char *getnextfilename (char **path, char *end) {
  char *sep;
  char *name = *path;
  // 如果已经到达路径末尾
  if (name == end)
    return NULL;  /* no more names */
  // 如果是上一次迭代的剩余部分
  else if (*name == '\0') {  /* from previous iteration? */
    *name = *LUA_PATH_SEP;  /* restore separator */
    name++;  /* skip it */
  }
  // 查找下一个分隔符
  sep = strchr(name, *LUA_PATH_SEP);  /* find next separator */
  // 如果没有找到分隔符
  if (sep == NULL)  /* separator not found? */
    sep = end;  /* name goes until the end */
  // 结束文件名
  *sep = '\0';  /* finish file name */
  // 更新路径指针
  *path = sep;  /* will start next search from here */
  // 返回文件名
  return name;
}

// 推入文件未找到的错误信息
static void pusherrornotfound (lua_State *L, const char *path) {
  luaL_Buffer b;
  luaL_buffinit(L, &b);
  luaL_addstring(&b, "no file '");
  luaL_addgsub(&b, path, LUA_PATH_SEP, "'\n\tno file '");
  luaL_addstring(&b, "'");
  luaL_pushresult(&b);
}
static const char *searchpath (lua_State *L, const char *name,
                                             const char *path,
                                             const char *sep,
                                             const char *dirsep) {
  luaL_Buffer buff;  /* 创建缓冲区对象 */
  char *pathname;  /* 插入名称的路径 */
  char *endpathname;  /* 路径的末尾 */
  const char *filename;  /* 文件名 */
  /* 分隔符不为空且出现在 'name' 中？ */
  if (*sep != '\0' && strchr(name, *sep) != NULL)
    name = luaL_gsub(L, name, sep, dirsep);  /* 用 'dirsep' 替换分隔符 */
  luaL_buffinit(L, &buff);  /* 初始化缓冲区 */
  /* 将路径添加到缓冲区，用文件名替换标记（'?'） */
  luaL_addgsub(&buff, path, LUA_PATH_MARK, name);
  luaL_addchar(&buff, '\0');  /* 添加字符串结束符 */
  pathname = luaL_buffaddr(&buff);  /* 可写的文件名列表 */
  endpathname = pathname + luaL_bufflen(&buff) - 1;  /* 路径的末尾 */
  while ((filename = getnextfilename(&pathname, endpathname)) != NULL) {
    if (readable(filename))  /* 文件是否存在且可读？ */
      return lua_pushstring(L, filename);  /* 保存并返回文件名 */
  }
  luaL_pushresult(&buff);  /* 推送路径以创建错误消息 */
  pusherrornotfound(L, lua_tostring(L, -1));  /* 创建错误消息 */
  return NULL;  /* 未找到 */
}


static int ll_searchpath (lua_State *L) {
  const char *f = searchpath(L, luaL_checkstring(L, 1),
                                luaL_checkstring(L, 2),
                                luaL_optstring(L, 3, "."),
                                luaL_optstring(L, 4, LUA_DIRSEP));
  if (f != NULL) return 1;  /* 找到文件 */
  else {  /* 错误消息在栈顶 */
    luaL_pushfail(L);  /* 推送失败标记 */
    lua_insert(L, -2);  /* 将失败标记插入错误消息之前 */
    return 2;  /* 返回失败标记 + 错误消息 */
  }
}
/*
** 在给定的 Lua 状态机中查找文件名为 'name' 的模块文件
** pname 为 package 表中的字段名
** dirsep 为目录分隔符
*/
static const char *findfile (lua_State *L, const char *name,
                                           const char *pname,
                                           const char *dirsep) {
  const char *path;
  lua_getfield(L, lua_upvalueindex(1), pname);  // 获取 package 表中字段名为 pname 的值
  path = lua_tostring(L, -1);  // 将字段值转换为字符串
  if (l_unlikely(path == NULL))  // 如果路径为空，抛出错误
    luaL_error(L, "'package.%s' must be a string", pname);
  return searchpath(L, name, path, ".", dirsep);  // 调用 searchpath 函数查找模块文件
}


/*
** 检查模块是否成功加载
** 如果成功加载，将文件名和打开函数推入栈中，返回 2
** 如果加载失败，抛出错误
*/
static int checkload (lua_State *L, int stat, const char *filename) {
  if (l_likely(stat)) {  // 如果模块成功加载
    lua_pushstring(L, filename);  // 将文件名推入栈中作为第二个参数
    return 2;  // 返回打开函数和文件名
  }
  else
    return luaL_error(L, "error loading module '%s' from file '%s':\n\t%s",
                          lua_tostring(L, 1), filename, lua_tostring(L, -1));  // 加载失败，抛出错误
}


/*
** Lua 搜索器函数
** 查找模块文件并加载
*/
static int searcher_Lua (lua_State *L) {
  const char *filename;
  const char *name = luaL_checkstring(L, 1);  // 获取参数中的模块名
  filename = findfile(L, name, "path", LUA_LSUBSEP);  // 查找模块文件
  if (filename == NULL) return 1;  // 如果模块文件未找到，返回 1
  return checkload(L, (luaL_loadfile(L, filename) == LUA_OK), filename);  // 检查模块加载情况
}


/*
** 尝试在文件 'filename' 中找到模块 'modname' 的加载函数
** 首先将 'modname' 中的 '.' 替换为 '_'，然后检查是否有忽略标记
** 如果有忽略标记，构建函数名 "luaopen_X" 并查找，如果失败，也尝试 "luaopen_Y"
** 如果没有忽略标记，直接查找函数名 "luaopen_modname"
*/
static int loadfunc (lua_State *L, const char *filename, const char *modname) {
  const char *openfunc;
  const char *mark;
  modname = luaL_gsub(L, modname, ".", LUA_OFSEP);  // 将 modname 中的 '.' 替换为 '_'
  mark = strchr(modname, *LUA_IGMARK);  // 查找忽略标记
  if (mark) {
    int stat;
    openfunc = lua_pushlstring(L, modname, mark - modname);  // 构建函数名
    openfunc = lua_pushfstring(L, LUA_POF"%s", openfunc);  // 构建函数名
    stat = lookforfunc(L, filename, openfunc);  // 查找函数
    # 如果状态不是 ERRFUNC，则返回状态值
    if (stat != ERRFUNC) return stat;
    # 将 modname 指向 mark 后面一个位置，即指向旧式名称
    modname = mark + 1;  /* else go ahead and try old-style name */
  }
  # 将 LUA_POF 和 modname 格式化为字符串并压入栈中
  openfunc = lua_pushfstring(L, LUA_POF"%s", modname);
  # 调用 lookforfunc 函数查找函数并返回结果
  return lookforfunc(L, filename, openfunc);
  # 定义一个名为 searcher_C 的静态函数，接收 Lua 状态机和一个参数
  static int searcher_C (lua_State *L) {
    # 从 Lua 栈中获取第一个参数作为模块名
    const char *name = luaL_checkstring(L, 1);
    # 在 cpath 中查找指定模块的文件名
    const char *filename = findfile(L, name, "cpath", LUA_CSUBSEP);
    # 如果找不到文件名，则返回 1
    if (filename == NULL) return 1;  /* module not found in this path */
    # 调用 checkload 函数，检查加载模块是否成功
    return checkload(L, (loadfunc(L, filename, name) == 0), filename);
  }

  # 定义一个名为 searcher_Croot 的静态函数，接收 Lua 状态机和一个参数
  static int searcher_Croot (lua_State *L) {
    # 从 Lua 栈中获取第一个参数作为模块名
    const char *name = luaL_checkstring(L, 1);
    # 在模块名中查找第一个点的位置
    const char *p = strchr(name, '.');
    int stat;
    # 如果找不到点，则返回 0
    if (p == NULL) return 0;  /* is root */
    # 将模块名的第一个点之前的部分推入 Lua 栈
    lua_pushlstring(L, name, p - name);
    # 在 cpath 中查找指定模块的文件名
    const char *filename = findfile(L, lua_tostring(L, -1), "cpath", LUA_CSUBSEP);
    # 如果找不到文件名，则返回 1
    if (filename == NULL) return 1;  /* root not found */
    # 调用 loadfunc 函数加载模块，并检查加载是否成功
    if ((stat = loadfunc(L, filename, name)) != 0) {
      # 如果加载失败，则返回相应的错误信息
      if (stat != ERRFUNC)
        return checkload(L, 0, filename);  /* real error */
      else {  /* open function not found */
        lua_pushfstring(L, "no module '%s' in file '%s'", name, filename);
        return 1;
      }
    }
    # 将文件名推入 Lua 栈
    lua_pushstring(L, filename);  /* will be 2nd argument to module */
    return 2;
  }

  # 定义一个名为 searcher_preload 的静态函数，接收 Lua 状态机和一个参数
  static int searcher_preload (lua_State *L) {
    # 从 Lua 栈中获取第一个参数作为模块名
    const char *name = luaL_checkstring(L, 1);
    # 获取 package.preload 表
    lua_getfield(L, LUA_REGISTRYINDEX, LUA_PRELOAD_TABLE);
    # 如果在 package.preload 表中找不到指定模块，则返回相应的错误信息
    if (lua_getfield(L, -1, name) == LUA_TNIL) {  /* not found? */
      lua_pushfstring(L, "no field package.preload['%s']", name);
      return 1;
    }
    else {
      # 否则返回 ":preload:"
      lua_pushliteral(L, ":preload:");
      return 2;
    }
  }

  # 定义一个名为 findloader 的静态函数，接收 Lua 状态机和一个参数
  static void findloader (lua_State *L, const char *name) {
    int i;
    luaL_Buffer msg;  /* to build error message */
    # 从闭包中获取 package.searchers 表
    if (l_unlikely(lua_getfield(L, lua_upvalueindex(1), "searchers")
                   != LUA_TTABLE))
      luaL_error(L, "'package.searchers' must be a table");
    # 初始化错误消息缓冲区
    luaL_buffinit(L, &msg);
    # 遍历 package.searchers 表，查找合适的加载器
    for (i = 1; ; i++) {
      luaL_addstring(&msg, "\n\t");  /* error-message prefix */
    # 如果搜索器返回的值是空值，则表示没有更多的搜索器了
    if (l_unlikely(lua_rawgeti(L, 3, i) == LUA_TNIL)) {  /* no more searchers? */
      # 弹出栈顶的空值
      lua_pop(L, 1);  /* remove nil */
      # 从消息缓冲区中移除前缀
      luaL_buffsub(&msg, 2);  /* remove prefix */
      # 将消息缓冲区中的内容作为错误消息推入栈中
      luaL_pushresult(&msg);  /* create error message */
      # 抛出 Lua 错误，指定模块未找到的错误消息
      luaL_error(L, "module '%s' not found:%s", name, lua_tostring(L, -1));
    }
    # 将模块名推入栈中
    lua_pushstring(L, name);
    # 调用 Lua 函数，参数为 1 个，返回值为 2 个
    lua_call(L, 1, 2);  /* call it */
    # 判断栈顶的值是否为函数
    if (lua_isfunction(L, -2))  /* did it find a loader? */
      # 如果是函数，则表示找到了模块加载器，直接返回
      return;  /* module loader found */
    # 如果栈顶的值是字符串，则表示搜索器返回了错误消息
    else if (lua_isstring(L, -2)) {  /* searcher returned error message? */
      # 弹出栈顶的额外返回值
      lua_pop(L, 1);  /* remove extra return */
      # 将错误消息添加到消息缓冲区中
      luaL_addvalue(&msg);  /* concatenate error message */
    }
    # 如果栈顶的值既不是函数也不是字符串，则表示没有错误消息
    else {  /* no error message */
      # 移除栈顶的两个返回值
      lua_pop(L, 2);  /* remove both returns */
      # 从消息缓冲区中移除前缀
      luaL_buffsub(&msg, 2);  /* remove prefix */
    }
  }
# 定义一个静态函数，用于加载模块
static int ll_require (lua_State *L) {
  # 获取传入参数的字符串类型
  const char *name = luaL_checkstring(L, 1);
  # 设置栈顶为1
  lua_settop(L, 1);  /* LOADED table will be at index 2 */
  # 获取全局表中的 LOADED_TABLE
  lua_getfield(L, LUA_REGISTRYINDEX, LUA_LOADED_TABLE);
  # 获取 LOADED_TABLE 中的 name
  lua_getfield(L, 2, name);  /* LOADED[name] */
  # 判断 LOADED[name] 是否存在
  if (lua_toboolean(L, -1))  /* is it there? */
    return 1;  /* package is already loaded */
  # 如果 LOADED[name] 不存在，则加载包
  /* else must load package */
  lua_pop(L, 1);  /* remove 'getfield' result */
  # 查找加载器
  findloader(L, name);
  # 旋转栈中的元素
  lua_rotate(L, -2, 1);  /* function <-> loader data */
  # 将 name 作为第一个参数推入栈中
  lua_pushvalue(L, 1);  /* name is 1st argument to module loader */
  # 将加载器数据作为第二个参数推入栈中
  lua_pushvalue(L, -3);  /* loader data is 2nd argument */
  # 调用加载器加载模块
  lua_call(L, 2, 1);  /* run loader to load module */
  # 判断加载器返回值是否为非空
  if (!lua_isnil(L, -1))  /* non-nil return? */
    # 将加载器返回值设置为 LOADED[name]
    lua_setfield(L, 2, name);  /* LOADED[name] = returned value */
  else
    # 弹出栈顶的 nil
    lua_pop(L, 1);  /* pop nil */
  # 如果 LOADED[name] 不存在
  if (lua_getfield(L, 2, name) == LUA_TNIL) {   /* module set no value? */
    # 将 true 推入栈中
    lua_pushboolean(L, 1);  /* use true as result */
    # 替换加载器返回值
    lua_copy(L, -1, -2);  /* replace loader result */
    # 将 true 设置为 LOADED[name]
    lua_setfield(L, 2, name);  /* LOADED[name] = true */
  }
  # 旋转栈中的元素
  lua_rotate(L, -2, 1);  /* loader data <-> module result  */
  # 返回模块结果和加载器数据
  return 2;  /* return module result and loader data */
}

/* }====================================================== */

# 定义一个静态常量数组，包含加载模块所需的函数
static const luaL_Reg pk_funcs[] = {
  {"loadlib", ll_loadlib},
  {"searchpath", ll_searchpath},
  /* placeholders */
  {"preload", NULL},
  {"cpath", NULL},
  {"path", NULL},
  {"searchers", NULL},
  {"loaded", NULL},
  {NULL, NULL}
};

# 定义一个静态常量数组，包含加载模块所需的函数
static const luaL_Reg ll_funcs[] = {
  {"require", ll_require},
  {NULL, NULL}
};

# 创建一个搜索器表
static void createsearcherstable (lua_State *L) {
  # 定义一个静态常量数组，包含搜索器函数
  static const lua_CFunction searchers[] =
    // 定义一个包含预加载搜索器和特定语言搜索器的数组
    {searcher_preload, searcher_Lua, searcher_C, searcher_Croot, NULL};
  int i;
  // 创建一个空的表格，用于存储搜索器
  lua_createtable(L, sizeof(searchers)/sizeof(searchers[0]) - 1, 0);
  // 将预定义的搜索器填充到表格中
  for (i=0; searchers[i] != NULL; i++) {
    // 将 'package' 设置为所有搜索器的上值
    lua_pushvalue(L, -2);
    // 将搜索器封装成闭包，并将其存储到表格中
    lua_pushcclosure(L, searchers[i], 1);
    lua_rawseti(L, -2, i+1);
  }
  // 将存储搜索器的表格放入 'searchers' 字段中
  lua_setfield(L, -2, "searchers");
/*
** create table CLIBS to keep track of loaded C libraries,
** setting a finalizer to close all libraries when closing state.
*/
static void createclibstable (lua_State *L) {
  // 获取或创建 LUA_REGISTRYINDEX 中的 CLIBS 子表
  luaL_getsubtable(L, LUA_REGISTRYINDEX, CLIBS);  /* create CLIBS table */
  // 创建 CLIBS 元表
  lua_createtable(L, 0, 1);  /* create metatable for CLIBS */
  // 将 gctm 函数设置为 CLIBS 元表的 __gc 元方法
  lua_pushcfunction(L, gctm);
  lua_setfield(L, -2, "__gc");  /* set finalizer for CLIBS table */
  // 将 CLIBS 元表设置为 CLIBS 表的元表
  lua_setmetatable(L, -2);
}


LUAMOD_API int luaopen_package (lua_State *L) {
  // 创建 CLIBS 表
  createclibstable(L);
  // 创建 'package' 表
  luaL_newlib(L, pk_funcs);  /* create 'package' table */
  // 创建搜索器表
  createsearcherstable(L);
  // 设置路径
  setpath(L, "path", LUA_PATH_VAR, LUA_PATH_DEFAULT);
  setpath(L, "cpath", LUA_CPATH_VAR, LUA_CPATH_DEFAULT);
  // 存储配置信息
  lua_pushliteral(L, LUA_DIRSEP "\n" LUA_PATH_SEP "\n" LUA_PATH_MARK "\n"
                     LUA_EXEC_DIR "\n" LUA_IGMARK "\n");
  lua_setfield(L, -2, "config");
  // 设置 'loaded' 字段
  luaL_getsubtable(L, LUA_REGISTRYINDEX, LUA_LOADED_TABLE);
  lua_setfield(L, -2, "loaded");
  // 设置 'preload' 字段
  luaL_getsubtable(L, LUA_REGISTRYINDEX, LUA_PRELOAD_TABLE);
  lua_setfield(L, -2, "preload");
  // 将全局表压栈
  lua_pushglobaltable(L);
  // 将 'package' 表作为下一个库的上值
  lua_pushvalue(L, -2);
  // 将 ll_funcs 中的函数设置为全局表的方法
  luaL_setfuncs(L, ll_funcs, 1);  /* open lib into global table */
  // 弹出全局表
  lua_pop(L, 1);  /* pop global table */
  // 返回 'package' 表
  return 1;  /* return 'package' table */
}
```