# `nmap\liblua\lua.c`

```cpp
/*
** $Id: lua.c $
** Lua stand-alone interpreter
** See Copyright Notice in lua.h
*/

// 定义了 lua_c，用于标识这是 Lua 的独立解释器
#define lua_c

// 包含 Lua 的前缀文件
#include "lprefix.h"

// 包含标准输入输出库
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// 包含信号处理库
#include <signal.h>

// 包含 Lua 核心库
#include "lua.h"

// 包含 Lua 辅助库
#include "lauxlib.h"

// 包含 Lua 标准库
#include "lualib.h"

// 如果未定义 LUA_PROGNAME，则定义为 "lua"
#if !defined(LUA_PROGNAME)
#define LUA_PROGNAME        "lua"
#endif

// 如果未定义 LUA_INIT_VAR，则定义为 "LUA_INIT"
#if !defined(LUA_INIT_VAR)
#define LUA_INIT_VAR        "LUA_INIT"
#endif

// 将 LUA_INIT_VAR 和 LUA_VERSUFFIX 连接成 LUA_INITVARVERSION
#define LUA_INITVARVERSION    LUA_INIT_VAR LUA_VERSUFFIX

// 全局 Lua 状态
static lua_State *globalL = NULL;

// 程序名称，默认为 LUA_PROGNAME
static const char *progname = LUA_PROGNAME;

// 如果定义了 LUA_USE_POSIX，则执行以下代码
#if defined(LUA_USE_POSIX)   /* { */

/*
** Use 'sigaction' when available.
*/
// 设置信号处理函数
static void setsignal (int sig, void (*handler)(int)) {
  struct sigaction sa;
  sa.sa_handler = handler;
  sa.sa_flags = 0;
  sigemptyset(&sa.sa_mask);  /* do not mask any signal */
  sigaction(sig, &sa, NULL);
}

#else           /* }{ */

// 否则使用 signal 函数
#define setsignal            signal

#endif                               /* } */

/*
** Hook set by signal function to stop the interpreter.
*/
// 由信号函数设置的钩子，用于停止解释器
static void lstop (lua_State *L, lua_Debug *ar) {
  (void)ar;  /* unused arg. */
  lua_sethook(L, NULL, 0, 0);  /* reset hook */
  luaL_error(L, "interrupted!");
}

/*
** Function to be called at a C signal. Because a C signal cannot
** just change a Lua state (as there is no proper synchronization),
** this function only sets a hook that, when called, will stop the
** interpreter.
*/
// 用于在 C 信号时调用的函数
static void laction (int i) {
  int flag = LUA_MASKCALL | LUA_MASKRET | LUA_MASKLINE | LUA_MASKCOUNT;
  setsignal(i, SIG_DFL); /* if another SIGINT happens, terminate process */
  lua_sethook(globalL, lstop, flag, 1);
}

// 打印用法信息
static void print_usage (const char *badoption) {
  lua_writestringerror("%s: ", progname);
  if (badoption[1] == 'e' || badoption[1] == 'l')
    lua_writestringerror("'%s' needs argument\n", badoption);
  else
    # 使用 lua_writestringerror 函数输出错误信息，包含未识别选项的信息
    lua_writestringerror("unrecognized option '%s'\n", badoption);
    # 使用 lua_writestringerror 函数输出用法信息，包含可用选项的说明
    lua_writestringerror(
    "usage: %s [options] [script [args]]\n"
    "Available options are:\n"
    "  -e stat   execute string 'stat'\n"
    "  -i        enter interactive mode after executing 'script'\n"
    "  -l mod    require library 'mod' into global 'mod'\n"
    "  -l g=mod  require library 'mod' into global 'g'\n"
    "  -v        show version information\n"
    "  -E        ignore environment variables\n"
    "  -W        turn warnings on\n"
    "  --        stop handling options\n"
    "  -         stop handling options and execute stdin\n"
    ,
    progname);
# 打印错误消息，如果程序名存在，则在消息前面添加程序名
static void l_message (const char *pname, const char *msg) {
  if (pname) lua_writestringerror("%s: ", pname);
  lua_writestringerror("%s\n", msg);
}

# 检查'status'是否不是OK，如果是，则打印堆栈顶部的错误消息
static int report (lua_State *L, int status) {
  if (status != LUA_OK) {
    const char *msg = lua_tostring(L, -1);
    l_message(progname, msg);
    lua_pop(L, 1);  /* remove message */
  }
  return status;
}

# 用于运行所有块的消息处理程序
static int msghandler (lua_State *L) {
  const char *msg = lua_tostring(L, 1);
  if (msg == NULL) {  /* is error object not a string? */
    if (luaL_callmeta(L, 1, "__tostring") &&  /* does it have a metamethod */
        lua_type(L, -1) == LUA_TSTRING)  /* that produces a string? */
      return 1;  /* that is the message */
    else
      msg = lua_pushfstring(L, "(error object is a %s value)",
                               luaL_typename(L, 1));
  }
  luaL_traceback(L, L, msg, 1);  /* append a standard traceback */
  return 1;  /* return the traceback */
}

# 'lua_pcall'的接口，设置适当的消息函数和C信号处理程序。用于运行所有块。
static int docall (lua_State *L, int narg, int nres) {
  int status;
  int base = lua_gettop(L) - narg;  /* function index */
  lua_pushcfunction(L, msghandler);  /* push message handler */
  lua_insert(L, base);  /* put it under function and args */
  globalL = L;  /* to be available to 'laction' */
  setsignal(SIGINT, laction);  /* set C-signal handler */
  status = lua_pcall(L, narg, nres, base);
  setsignal(SIGINT, SIG_DFL); /* reset C-signal handler */
  lua_remove(L, base);  /* remove message handler from the stack */
  return status;
}
# 打印 Lua 版本信息
static void print_version (void) {
  lua_writestring(LUA_COPYRIGHT, strlen(LUA_COPYRIGHT));
  lua_writeline();
}

# 创建 'arg' 表，存储从命令行 ('argv') 传入的所有参数。应该对齐，使得在索引 0 处有 'argv[script]'，即脚本名称。脚本的参数（'script' 之后的所有内容）存储在正索引位置；脚本名称之前的参数存储在负索引位置。如果没有脚本名称，则假定解释器的名称为基础。
static void createargtable (lua_State *L, char **argv, int argc, int script) {
  int i, narg;
  if (script == argc) script = 0;  # 没有脚本名称？
  narg = argc - (script + 1);  # 正索引位置的参数数量
  lua_createtable(L, narg, script + 1);
  for (i = 0; i < argc; i++) {
    lua_pushstring(L, argv[i]);
    lua_rawseti(L, -2, i - script);
  }
  lua_setglobal(L, "arg");
}

# 执行 Lua 代码块
static int dochunk (lua_State *L, int status) {
  if (status == LUA_OK) status = docall(L, 0, 0);
  return report(L, status);
}

# 执行 Lua 文件
static int dofile (lua_State *L, const char *name) {
  return dochunk(L, luaL_loadfile(L, name));
}

# 执行 Lua 字符串
static int dostring (lua_State *L, const char *s, const char *name) {
  return dochunk(L, luaL_loadbuffer(L, s, strlen(s), name));
}

# 接收 'globname[=modname]' 并运行 'globname = require(modname)'
static int dolibrary (lua_State *L, char *globname) {
  int status;
  char *modname = strchr(globname, '=');
  if (modname == NULL)  # 没有显式名称？
    modname = globname;  # 模块名称等于全局名称
  else:
    *modname = '\0';  # 全局名称到此结束
    modname++;  # 模块名称从 '=' 后开始
  lua_getglobal(L, "require");
  lua_pushstring(L, modname);
  status = docall(L, 1, 1);  # 调用 'require(modname)'
  if (status == LUA_OK)
    lua_setglobal(L, globname);  # globname = require(modname)
  return report(L, status);
}
/*
** 将表 'arg' 从 1 到 #arg 的内容推入栈中
*/
static int pushargs (lua_State *L) {
  int i, n;
  if (lua_getglobal(L, "arg") != LUA_TTABLE)  // 获取全局变量 'arg'，如果不是表则报错
    luaL_error(L, "'arg' is not a table");
  n = (int)luaL_len(L, -1);  // 获取表 'arg' 的长度
  luaL_checkstack(L, n + 3, "too many arguments to script");  // 检查栈是否有足够的空间
  for (i = 1; i <= n; i++)
    lua_rawgeti(L, -i, i);  // 将表 'arg' 中的元素推入栈中
  lua_remove(L, -i);  /* 从栈中移除表 */
  return n;  // 返回参数个数
}


static int handle_script (lua_State *L, char **argv) {
  int status;
  const char *fname = argv[0];  // 获取脚本文件名
  if (strcmp(fname, "-") == 0 && strcmp(argv[-1], "--") != 0)
    fname = NULL;  /* 如果文件名为 "-" 且前一个参数不是 "--"，则将文件名置为 NULL，表示标准输入 */
  status = luaL_loadfile(L, fname);  // 加载文件
  if (status == LUA_OK) {
    int n = pushargs(L);  /* 将参数推入栈中 */
    status = docall(L, n, LUA_MULTRET);  // 调用 Lua 函数
  }
  return report(L, status);  // 返回执行结果
}


/* 'args' 中各种参数指示位的定义 */
#define has_error    1    /* 错误选项 */
#define has_i        2    /* -i */
#define has_v        4    /* -v */
#define has_e        8    /* -e */
#define has_E        16    /* -E */


/*
** 遍历 'argv' 中的所有参数，返回需要在运行任何 Lua 代码之前的参数掩码
** (如果找到任何无效参数，则返回错误代码)。'first' 返回第一个未处理的参数
** (脚本名称或错误参数)。
*/
static int collectargs (char **argv, int *first) {
  int args = 0;
  int i;
  for (i = 1; argv[i] != NULL; i++) {
    *first = i;  // 记录第一个未处理的参数位置
    if (argv[i][0] != '-')  /* 不是选项? */
        return args;  /* 停止处理选项 */
    switch (argv[i][1]) {  /* 检查选项 */
      case '-':  /* '--' */
        if (argv[i][2] != '\0')  /* '--' 后面有额外字符吗？ */
          return has_error;  /* 无效选项 */
        *first = i + 1;  /* 设置脚本参数的起始位置 */
        return args;  /* 返回参数 */
      case '\0':  /* '-' */
        return args;  /* 脚本“名称”为'-' */
      case 'E':
        if (argv[i][2] != '\0')  /* 有额外字符吗？ */
          return has_error;  /* 无效选项 */
        args |= has_E;  /* 设置标志位 */
        break;
      case 'W':
        if (argv[i][2] != '\0')  /* 有额外字符吗？ */
          return has_error;  /* 无效选项 */
        break;
      case 'i':
        args |= has_i;  /* (-i 意味着 -v) *//* 继续执行下一个case */
      case 'v':
        if (argv[i][2] != '\0')  /* 有额外字符吗？ */
          return has_error;  /* 无效选项 */
        args |= has_v;  /* 设置标志位 */
        break;
      case 'e':
        args |= has_e;  /* 继续执行下一个case */
      case 'l':  /* 两个选项都需要参数 */
        if (argv[i][2] == '\0') {  /* 没有连接的参数吗？ */
          i++;  /* 尝试下一个 'argv' */
          if (argv[i] == NULL || argv[i][0] == '-')
            return has_error;  /* 没有下一个参数或者下一个参数是另一个选项 */
        }
        break;
      default:  /* 无效选项 */
        return has_error;
    }
  }
  *first = i;  /* 没有脚本名称 */
  return args;
/* 
** Processes options 'e' and 'l', which involve running Lua code, and
** 'W', which also affects the state.
** Returns 0 if some code raises an error.
*/
static int runargs (lua_State *L, char **argv, int n) {
  int i;
  for (i = 1; i < n; i++) {
    int option = argv[i][1];
    lua_assert(argv[i][0] == '-');  /* already checked */
    switch (option) {
      case 'e':  case 'l': {
        int status;
        char *extra = argv[i] + 2;  /* both options need an argument */
        if (*extra == '\0') extra = argv[++i];
        lua_assert(extra != NULL);
        status = (option == 'e')
                 ? dostring(L, extra, "=(command line)")
                 : dolibrary(L, extra);
        if (status != LUA_OK) return 0;
        break;
      }
      case 'W':
        lua_warning(L, "@on", 0);  /* warnings on */
        break;
    }
  }
  return 1;
}

static int handle_luainit (lua_State *L) {
  const char *name = "=" LUA_INITVARVERSION;
  const char *init = getenv(name + 1);
  if (init == NULL) {
    name = "=" LUA_INIT_VAR;
    init = getenv(name + 1);  /* try alternative name */
  }
  if (init == NULL) return LUA_OK;
  else if (init[0] == '@')
    return dofile(L, init+1);
  else
    return dostring(L, init, name);
}

/*
** {==================================================================
** Read-Eval-Print Loop (REPL)
** ===================================================================
*/

#if !defined(LUA_PROMPT)
#define LUA_PROMPT        "> "
#define LUA_PROMPT2        ">> "
#endif

#if !defined(LUA_MAXINPUT)
#define LUA_MAXINPUT        512
#endif

/*
** lua_stdin_is_tty detects whether the standard input is a 'tty' (that
** is, whether we're running lua interactively).
*/
#if !defined(lua_stdin_is_tty)    /* { */

#if defined(LUA_USE_POSIX)    /* { */

#include <unistd.h>
#define lua_stdin_is_tty()    isatty(0)

#elif defined(LUA_USE_WINDOWS)    /* }{ */

#include <io.h>
#include <windows.h>

#define lua_stdin_is_tty()    _isatty(_fileno(stdin))
#else                /* }{ */

/* ISO C definition */
#define lua_stdin_is_tty()    1  /* assume stdin is a tty */

#endif                /* } */

#endif                /* } */


/*
** lua_readline defines how to show a prompt and then read a line from
** the standard input.
** lua_saveline defines how to "save" a read line in a "history".
** lua_freeline defines how to free a line read by lua_readline.
*/
#if !defined(lua_readline)    /* { */

#if defined(LUA_USE_READLINE)    /* { */

#include <readline/readline.h>
#include <readline/history.h>
#define lua_initreadline(L)    ((void)L, rl_readline_name="lua")
#define lua_readline(L,b,p)    ((void)L, ((b)=readline(p)) != NULL)
#define lua_saveline(L,line)    ((void)L, add_history(line))
#define lua_freeline(L,b)    ((void)L, free(b))

#else                /* }{ */

#define lua_initreadline(L)  ((void)L)
#define lua_readline(L,b,p) \
        ((void)L, fputs(p, stdout), fflush(stdout),  /* show prompt */ \
        fgets(b, LUA_MAXINPUT, stdin) != NULL)  /* get line */
#define lua_saveline(L,line)    { (void)L; (void)line; }
#define lua_freeline(L,b)    { (void)L; (void)b; }

#endif                /* } */

#endif                /* } */


/*
** Return the string to be used as a prompt by the interpreter. Leave
** the string (or nil, if using the default value) on the stack, to keep
** it anchored.
*/
static const char *get_prompt (lua_State *L, int firstline) {
  if (lua_getglobal(L, firstline ? "_PROMPT" : "_PROMPT2") == LUA_TNIL)
    return (firstline ? LUA_PROMPT : LUA_PROMPT2);  /* use the default */
  else {  /* apply 'tostring' over the value */
    const char *p = luaL_tolstring(L, -1, NULL);
    lua_remove(L, -2);  /* remove original value */
    return p;
  }
}

/* mark in error messages for incomplete statements */
#define EOFMARK        "<eof>"
#define marklen        (sizeof(EOFMARK)/sizeof(char) - 1)


/*
** Check whether 'status' signals a syntax error and the error
/*
** message at the top of the stack ends with the above mark for
** incomplete statements.
*/
static int incomplete (lua_State *L, int status) {
  // 如果解析出错是语法错误
  if (status == LUA_ERRSYNTAX) {
    size_t lmsg;
    const char *msg = lua_tolstring(L, -1, &lmsg);
    // 检查错误信息是否以特定标记结尾
    if (lmsg >= marklen && strcmp(msg + lmsg - marklen, EOFMARK) == 0) {
      lua_pop(L, 1);
      return 1;
    }
  }
  return 0;  /* else... */
}


/*
** Prompt the user, read a line, and push it into the Lua stack.
*/
static int pushline (lua_State *L, int firstline) {
  char buffer[LUA_MAXINPUT];
  char *b = buffer;
  size_t l;
  const char *prmt = get_prompt(L, firstline);
  // 读取用户输入的一行
  int readstatus = lua_readline(L, b, prmt);
  if (readstatus == 0)
    return 0;  /* no input (prompt will be popped by caller) */
  lua_pop(L, 1);  /* remove prompt */
  l = strlen(b);
  // 如果行以换行符结尾，则移除
  if (l > 0 && b[l-1] == '\n')  /* line ends with newline? */
    b[--l] = '\0';  /* remove it */
  // 如果是第一行并且以'='开头，则将其替换为'return'
  if (firstline && b[0] == '=')  /* for compatibility with 5.2, ... */
    lua_pushfstring(L, "return %s", b + 1);  /* change '=' to 'return' */
  else
    lua_pushlstring(L, b, l);
  lua_freeline(L, b);
  return 1;
}


/*
** Try to compile line on the stack as 'return <line>;'; on return, stack
** has either compiled chunk or original line (if compilation failed).
*/
static int addreturn (lua_State *L) {
  const char *line = lua_tostring(L, -1);  /* original line */
  const char *retline = lua_pushfstring(L, "return %s;", line);
  // 尝试编译带有'return'的行
  int status = luaL_loadbuffer(L, retline, strlen(retline), "=stdin");
  if (status == LUA_OK) {
    lua_remove(L, -2);  /* remove modified line */
    if (line[0] != '\0')  /* non empty? */
      lua_saveline(L, line);  /* keep history */
  }
  else
    lua_pop(L, 2);  /* pop result from 'luaL_loadbuffer' and modified line */
  return status;
}


/*
** Read multiple lines until a complete Lua statement
*/
static int multiline (lua_State *L) {
  for (;;) {  /* repeat until gets a complete statement */
    size_t len;
    // 将 Lua 栈中索引为 1 的元素转换为 C 字符串，同时获取其长度
    const char *line = lua_tolstring(L, 1, &len);  /* get what it has */
    // 尝试将字符串内容加载为 Lua 代码块
    int status = luaL_loadbuffer(L, line, len, "=stdin");  /* try it */
    // 如果加载的代码块不是不完整的，或者无法推入新的输入行，则保存当前输入行到历史记录并返回加载状态
    if (!incomplete(L, status) || !pushline(L, 0)) {
      lua_saveline(L, line);  /* keep history */
      return status;  /* cannot or should not try to add continuation line */
    }
    // 在 Lua 栈中推入换行符
    lua_pushliteral(L, "\n");  /* add newline... */
    // 将栈顶元素插入到倒数第二个位置
    lua_insert(L, -2);  /* ...between the two lines */
    // 将栈顶的 3 个元素连接起来
    lua_concat(L, 3);  /* join them */
  }
/*
** 读取一行并尝试首先将其作为表达式（在其前面添加"return "）加载（编译），然后作为语句加载。
** 返回加载/调用的最终状态，并将结果函数（如果有）置于堆栈顶部。
*/
static int loadline (lua_State *L) {
  int status;
  lua_settop(L, 0);  /* 清空堆栈 */
  if (!pushline(L, 1))  /* 将输入行推入堆栈 */
    return -1;  /* 没有输入 */
  if ((status = addreturn(L)) != LUA_OK)  /* 'return ...' 无效？ */
    status = multiline(L);  /* 尝试作为命令，可能包含连续行 */
  lua_remove(L, 1);  /* 从堆栈中移除行 */
  lua_assert(lua_gettop(L) == 1);  /* 确保堆栈中只有一个元素 */
  return status;
}


/*
** 打印（调用 Lua 的 'print' 函数）堆栈上的任何值
*/
static void l_print (lua_State *L) {
  int n = lua_gettop(L);  /* 获取堆栈上的元素数量 */
  if (n > 0) {  /* 有要打印的结果吗？ */
    luaL_checkstack(L, LUA_MINSTACK, "too many results to print");  /* 检查堆栈空间是否足够 */
    lua_getglobal(L, "print");  /* 获取全局的 'print' 函数 */
    lua_insert(L, 1);  /* 将 'print' 函数置于堆栈顶部 */
    if (lua_pcall(L, n, 0, 0) != LUA_OK)  /* 调用 'print' 函数 */
      l_message(progname, lua_pushfstring(L, "error calling 'print' (%s)",
                                             lua_tostring(L, -1)));  /* 打印错误信息 */
  }
}


/*
** 执行 REPL：重复读取（加载）一行，评估（调用）它，并打印任何结果。
*/
static void doREPL (lua_State *L) {
  int status;
  const char *oldprogname = progname;
  progname = NULL;  /* 在交互模式下没有 'progname' */
  lua_initreadline(L);  /* 初始化读取行 */
  while ((status = loadline(L)) != -1) {
    if (status == LUA_OK)
      status = docall(L, 0, LUA_MULTRET);  /* 调用函数 */
    if (status == LUA_OK) l_print(L);  /* 打印结果 */
    else report(L, status);  /* 报告错误 */
  }
  lua_settop(L, 0);  /* 清空堆栈 */
  lua_writeline();  /* 写入换行符 */
  progname = oldprogname;
}

/* }================================================================== */


/*
** 独立解释器的主体（在受保护模式下调用）。
** 读取选项并处理它们。
*/
static int pmain (lua_State *L) {
  // 从 Lua 栈中获取参数个数
  int argc = (int)lua_tointeger(L, 1);
  // 从 Lua 栈中获取参数数组
  char **argv = (char **)lua_touserdata(L, 2);
  int script;
  // 解析命令行参数，获取参数个数
  int args = collectargs(argv, &script);
  luaL_checkversion(L);  /* 检查解释器的版本是否正确 */
  if (argv[0] && argv[0][0]) progname = argv[0];
  if (args == has_error) {  /* 参数错误？ */
    print_usage(argv[script]);  /* 'script' 是错误参数的索引 */
    return 0;
  }
  if (args & has_v)  /* 选项 '-v'? */
    print_version();
  if (args & has_E) {  /* 选项 '-E'? */
    lua_pushboolean(L, 1);  /* 信号让库忽略环境变量 */
    lua_setfield(L, LUA_REGISTRYINDEX, "LUA_NOENV");
  }
  luaL_openlibs(L);  /* 打开标准库 */
  createargtable(L, argv, argc, script);  /* 创建 'arg' 表 */
  lua_gc(L, LUA_GCGEN, 0, 0);  /* 以分代模式进行垃圾回收 */
  if (!(args & has_E)) {  /* 没有选项 '-E'? */
    if (handle_luainit(L) != LUA_OK)  /* 运行 LUA_INIT */
      return 0;  /* 运行 LUA_INIT 出错 */
  }
  if (!runargs(L, argv, script))  /* 执行参数 -e 和 -l */
    return 0;  /* 出现错误 */
  if (script < argc &&  /* 执行主脚本（如果有的话） */
      handle_script(L, argv + script) != LUA_OK)
    return 0;
  if (args & has_i)  /* -i 选项? */
    doREPL(L);  /* 进入交互式模式 */
  else if (script == argc && !(args & (has_e | has_v))) {  /* 没有参数? */
    if (lua_stdin_is_tty()) {  /* 是否在交互模式下运行? */
      print_version();
      doREPL(L);  /* 进入交互式模式 */
    }
    else dofile(L, NULL);  /* 执行标准输入作为文件 */
  }
  lua_pushboolean(L, 1);  /* 信号没有错误 */
  return 1;
}

int main (int argc, char **argv) {
  int status, result;
  lua_State *L = luaL_newstate();  /* 创建状态 */
  if (L == NULL) {
    l_message(argv[0], "cannot create state: not enough memory");
  }
  // ...
}
  // 返回程序退出失败状态
  return EXIT_FAILURE;
  // 将函数 pmain 压入栈中，以便在受保护模式下调用
  lua_pushcfunction(L, &pmain);  /* to call 'pmain' in protected mode */
  // 将 argc 压入栈中作为第一个参数
  lua_pushinteger(L, argc);  /* 1st argument */
  // 将 argv 压入栈中作为第二个参数
  lua_pushlightuserdata(L, argv); /* 2nd argument */
  // 在 Lua 中调用函数，2 个参数，1 个返回值，错误处理模式为 0
  status = lua_pcall(L, 2, 1, 0);  /* do the call */
  // 获取返回值并转换为布尔值
  result = lua_toboolean(L, -1);  /* get result */
  // 输出状态报告
  report(L, status);
  // 关闭 Lua 环境
  lua_close(L);
  // 如果结果为真且状态为 LUA_OK，则返回程序退出成功状态，否则返回失败状态
  return (result && status == LUA_OK) ? EXIT_SUCCESS : EXIT_FAILURE;
# 代码块结束
```