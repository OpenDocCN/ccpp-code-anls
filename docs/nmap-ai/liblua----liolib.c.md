# `nmap\liblua\liolib.c`

```cpp
/*
** $Id: liolib.c $
** Standard I/O (and system) library
** See Copyright Notice in lua.h
*/

// 定义 liolib.c，标准 I/O（和系统）库
#define liolib_c
// 定义 LUA_LIB
#define LUA_LIB

// 包含 lprefix.h
#include "lprefix.h"

// 包含标准库头文件
#include <ctype.h>
#include <errno.h>
#include <locale.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// 包含 lua.h
#include "lua.h"

// 包含 lauxlib.h
#include "lauxlib.h"
// 包含 lualib.h
#include "lualib.h"

// 定义 l_checkmode 宏
#if !defined(l_checkmode)

// 定义 L_MODEEXT 宏
#if !defined(L_MODEEXT)
#define L_MODEEXT    "b"
#endif

// 检查 mode 是否匹配 '[rwa]%+?[L_MODEEXT]*'
static int l_checkmode (const char *mode) {
  return (*mode != '\0' && strchr("rwa", *(mode++)) != NULL &&
         (*mode != '+' || ((void)(++mode), 1)) &&  /* skip if char is '+' */
         (strspn(mode, L_MODEEXT) == strlen(mode)));  /* check extensions */
}

#endif

// l_popen 生成一个新的进程，通过文件流与当前进程连接
#if !defined(l_popen)        /* { */

#if defined(LUA_USE_POSIX)    /* { */

#define l_popen(L,c,m)        (fflush(NULL), popen(c,m))
#define l_pclose(L,file)    (pclose(file))

#elif defined(LUA_USE_WINDOWS)    /* }{ */

#define l_popen(L,c,m)        (_popen(c,m))
#define l_pclose(L,file)    (_pclose(file))

#if !defined(l_checkmodep)
// Windows 接受 "[rw][bt]?" 作为有效的模式
#define l_checkmodep(m)    ((m[0] == 'r' || m[0] == 'w') && \
  (m[1] == '\0' || ((m[1] == 'b' || m[1] == 't') && m[2] == '\0')))
#endif

#else                /* }{ */

// ISO C 定义
#define l_popen(L,c,m)  \
      ((void)c, (void)m, \
      luaL_error(L, "'popen' not supported"), \
      (FILE*)0)
#define l_pclose(L,file)        ((void)L, (void)file, -1)

#endif                /* } */

#endif                /* } */


#if !defined(l_checkmodep)
/* By default, Lua accepts only "r" or "w" as valid modes */
// 定义宏，检查模式是否为 "r" 或 "w"
#define l_checkmodep(m)        ((m[0] == 'r' || m[0] == 'w') && m[1] == '\0')
#endif

/* }====================================================== */


#if !defined(l_getc)        /* { */

#if defined(LUA_USE_POSIX)
// 如果使用 POSIX，定义获取字符的宏
#define l_getc(f)        getc_unlocked(f)
// 定义文件加锁的宏
#define l_lockfile(f)        flockfile(f)
// 定义文件解锁的宏
#define l_unlockfile(f)        funlockfile(f)
#else
// 如果不使用 POSIX，定义获取字符的宏
#define l_getc(f)        getc(f)
// 定义空的文件加锁宏
#define l_lockfile(f)        ((void)0)
// 定义空的文件解锁宏
#define l_unlockfile(f)        ((void)0)
#endif

#endif                /* } */


/*
** {======================================================
** l_fseek: configuration for longer offsets
** =======================================================
*/

#if !defined(l_fseek)        /* { */

#if defined(LUA_USE_POSIX)    /* { */

#include <sys/types.h>

// 如果使用 POSIX，定义文件定位的宏
#define l_fseek(f,o,w)        fseeko(f,o,w)
// 定义文件当前位置的宏
#define l_ftell(f)        ftello(f)
// 定义文件偏移量的类型
#define l_seeknum        off_t

#elif defined(LUA_USE_WINDOWS) && !defined(_CRTIMP_TYPEINFO) \
   && defined(_MSC_VER) && (_MSC_VER >= 1400)    /* }{ */

/* Windows (but not DDK) and Visual C++ 2005 or higher */
// 如果使用 Windows 并且是 Visual C++ 2005 或更高版本，定义文件定位的宏
#define l_fseek(f,o,w)        _fseeki64(f,o,w)
// 定义文件当前位置的宏
#define l_ftell(f)        _ftelli64(f)
// 定义文件偏移量的类型
#define l_seeknum        __int64

#else                /* }{ */

/* ISO C definitions */
// 如果不满足上述条件，使用 ISO C 定义文件定位的宏
#define l_fseek(f,o,w)        fseek(f,o,w)
// 定义文件当前位置的宏
#define l_ftell(f)        ftell(f)
// 定义文件偏移量的类型
#define l_seeknum        long

#endif                /* } */

#endif                /* } */

/* }====================================================== */



// 定义文件流的前缀和长度
#define IO_PREFIX    "_IO_"
#define IOPREF_LEN    (sizeof(IO_PREFIX)/sizeof(char) - 1)
// 定义输入文件流的标识符
#define IO_INPUT    (IO_PREFIX "input")
// 定义输出文件流的标识符
#define IO_OUTPUT    (IO_PREFIX "output")


// 定义 Lua 文件流类型
typedef luaL_Stream LStream;


// 将 Lua 对象转换为文件流类型
#define tolstream(L)    ((LStream *)luaL_checkudata(L, 1, LUA_FILEHANDLE))
// 检查文件流是否关闭
#define isclosed(p)    ((p)->closef == NULL)
static int io_type (lua_State *L) {
  // 声明一个指向 LStream 结构体的指针 p
  LStream *p;
  // 检查参数是否为任意类型
  luaL_checkany(L, 1);
  // 尝试将参数转换为 LUA_FILEHANDLE 类型的用户数据，如果失败返回 NULL
  p = (LStream *)luaL_testudata(L, 1, LUA_FILEHANDLE);
  // 如果转换失败，压入一个失败标记
  if (p == NULL)
    luaL_pushfail(L);  /* not a file */
  // 如果转换成功，判断文件是否已关闭，分别压入不同的字符串
  else if (isclosed(p))
    lua_pushliteral(L, "closed file");
  else
    lua_pushliteral(L, "file");
  // 返回 1 个结果
  return 1;
}


static int f_tostring (lua_State *L) {
  // 将参数转换为 LStream 结构体指针
  LStream *p = tolstream(L);
  // 如果文件已关闭，压入字符串 "file (closed)"
  if (isclosed(p))
    lua_pushliteral(L, "file (closed)");
  // 如果文件未关闭，压入格式化后的文件指针字符串
  else
    lua_pushfstring(L, "file (%p)", p->f);
  // 返回 1 个结果
  return 1;
}


static FILE *tofile (lua_State *L) {
  // 将参数转换为 LStream 结构体指针
  LStream *p = tolstream(L);
  // 如果文件已关闭，抛出错误
  if (l_unlikely(isclosed(p)))
    luaL_error(L, "attempt to use a closed file");
  // 断言文件指针不为空，然后返回文件指针
  lua_assert(p->f);
  return p->f;
}


/*
** When creating file handles, always creates a 'closed' file handle
** before opening the actual file; so, if there is a memory error, the
** handle is in a consistent state.
*/
static LStream *newprefile (lua_State *L) {
  // 分配 LStream 结构体大小的内存，并返回指向该内存的指针
  LStream *p = (LStream *)lua_newuserdatauv(L, sizeof(LStream), 0);
  // 将 closef 字段置为 NULL，标记文件句柄为 'closed'
  p->closef = NULL;  /* mark file handle as 'closed' */
  // 设置元表为 LUA_FILEHANDLE
  luaL_setmetatable(L, LUA_FILEHANDLE);
  // 返回指向 LStream 结构体的指针
  return p;
}


/*
** Calls the 'close' function from a file handle. The 'volatile' avoids
** a bug in some versions of the Clang compiler (e.g., clang 3.0 for
** 32 bits).
*/
static int aux_close (lua_State *L) {
  // 将参数转换为 LStream 结构体指针
  LStream *p = tolstream(L);
  // 声明一个 volatile lua_CFunction 类型的变量 cf，并赋值为 p->closef
  volatile lua_CFunction cf = p->closef;
  // 将 p->closef 置为 NULL，标记流为关闭状态
  p->closef = NULL;  /* mark stream as closed */
  // 调用 cf 函数指针，关闭流
  return (*cf)(L);  /* close it */
}


static int f_close (lua_State *L) {
  // 确保参数是一个打开的流
  tofile(L);  /* make sure argument is an open stream */
  // 调用辅助关闭函数
  return aux_close(L);
}


static int io_close (lua_State *L) {
  // 如果没有参数，获取默认输出流
  if (lua_isnone(L, 1))  /* no argument? */
    lua_getfield(L, LUA_REGISTRYINDEX, IO_OUTPUT);  /* use default output */
  // 调用文件关闭函数
  return f_close(L);
}


static int f_gc (lua_State *L) {
  // 将参数转换为 LStream 结构体指针
  LStream *p = tolstream(L);
  // 如果文件未关闭且文件指针不为空，调用辅助关闭函数
  if (!isclosed(p) && p->f != NULL)
    aux_close(L);  /* ignore closed and incompletely open files */
  // 返回 0 个结果
  return 0;
}


/*
** function to close regular files
*/
static int io_fclose (lua_State *L) {
  // 从 Lua 状态中获取文件流对象
  LStream *p = tolstream(L);
  // 关闭文件流
  int res = fclose(p->f);
  // 返回文件关闭结果
  return luaL_fileresult(L, (res == 0), NULL);
}


static LStream *newfile (lua_State *L) {
  // 创建新的文件流对象
  LStream *p = newprefile(L);
  // 初始化文件流对象的文件指针为空
  p->f = NULL;
  // 设置文件流对象的关闭函数为 io_fclose
  p->closef = &io_fclose;
  // 返回文件流对象
  return p;
}


static void opencheck (lua_State *L, const char *fname, const char *mode) {
  // 创建新的文件流对象
  LStream *p = newfile(L);
  // 以指定模式打开文件
  p->f = fopen(fname, mode);
  // 如果文件打开失败，则抛出错误
  if (l_unlikely(p->f == NULL))
    luaL_error(L, "cannot open file '%s' (%s)", fname, strerror(errno));
}


static int io_open (lua_State *L) {
  // 获取文件名和模式参数
  const char *filename = luaL_checkstring(L, 1);
  const char *mode = luaL_optstring(L, 2, "r");
  // 创建新的文件流对象
  LStream *p = newfile(L);
  const char *md = mode;  /* to traverse/check mode */
  // 检查模式是否有效
  luaL_argcheck(L, l_checkmode(md), 2, "invalid mode");
  // 以指定模式打开文件
  p->f = fopen(filename, mode);
  // 返回文件打开结果
  return (p->f == NULL) ? luaL_fileresult(L, 0, filename) : 1;
}


/*
** function to close 'popen' files
*/
static int io_pclose (lua_State *L) {
  // 从 Lua 状态中获取文件流对象
  LStream *p = tolstream(L);
  // 清空错误标志
  errno = 0;
  // 返回执行结果
  return luaL_execresult(L, l_pclose(L, p->f));
}


static int io_popen (lua_State *L) {
  // 获取文件名和模式参数
  const char *filename = luaL_checkstring(L, 1);
  const char *mode = luaL_optstring(L, 2, "r");
  // 创建新的文件流对象
  LStream *p = newprefile(L);
  // 检查模式是否有效
  luaL_argcheck(L, l_checkmodep(mode), 2, "invalid mode");
  // 打开文件流
  p->f = l_popen(L, filename, mode);
  // 设置文件流对象的关闭函数为 io_pclose
  p->closef = &io_pclose;
  // 返回文件打开结果
  return (p->f == NULL) ? luaL_fileresult(L, 0, filename) : 1;
}


static int io_tmpfile (lua_State *L) {
  // 创建新的文件流对象
  LStream *p = newfile(L);
  // 创建临时文件
  p->f = tmpfile();
  // 返回文件打开结果
  return (p->f == NULL) ? luaL_fileresult(L, 0, NULL) : 1;
}


static FILE *getiofile (lua_State *L, const char *findex) {
  LStream *p;
  // 从全局注册表中获取文件流对象
  lua_getfield(L, LUA_REGISTRYINDEX, findex);
  // 将文件流对象转换为指针类型
  p = (LStream *)lua_touserdata(L, -1);
  // 如果文件流对象已关闭，则抛出错误
  if (l_unlikely(isclosed(p)))
    luaL_error(L, "default %s file is closed", findex + IOPREF_LEN);
  // 返回文件指针
  return p->f;
}


static int g_iofile (lua_State *L, const char *f, const char *mode) {
  // 如果第一个参数不为 nil，则获取文件名
  if (!lua_isnoneornil(L, 1)) {
    const char *filename = lua_tostring(L, 1);
    # 如果文件名存在，则打开文件并检查模式
    if (filename)
      opencheck(L, filename, mode);
    # 如果文件名不存在，则检查它是否是有效的文件句柄
    else {
      tofile(L);  # 检查它是否是有效的文件句柄
      lua_pushvalue(L, 1);  # 将文件句柄推入栈顶
    }
    # 将栈顶的值设置为指定注册表中的字段
    lua_setfield(L, LUA_REGISTRYINDEX, f);
  }
  # 返回当前值
  lua_getfield(L, LUA_REGISTRYINDEX, f);
  return 1;
# 静态函数，用于处理输入流，返回一个文件句柄
static int io_input (lua_State *L) {
  return g_iofile(L, IO_INPUT, "r");
}

# 静态函数，用于处理输出流，返回一个文件句柄
static int io_output (lua_State *L) {
  return g_iofile(L, IO_OUTPUT, "w");
}

# 读取一行数据的函数
static int io_readline (lua_State *L);

# 定义最大参数数量为250
#define MAXARGLINE    250

# 辅助函数，用于创建'lines'的迭代函数
static void aux_lines (lua_State *L, int toclose) {
  int n = lua_gettop(L) - 1;  /* 读取参数的数量 */
  luaL_argcheck(L, n <= MAXARGLINE, MAXARGLINE + 2, "too many arguments");  # 检查参数数量是否超过最大值
  lua_pushvalue(L, 1);  /* 文件 */
  lua_pushinteger(L, n);  /* 参数数量 */
  lua_pushboolean(L, toclose);  /* 是否在结束时关闭文件 */
  lua_rotate(L, 2, 3);  /* 移动三个值到它们的位置 */
  lua_pushcclosure(L, io_readline, 3 + n);
}

# 返回'lines'的迭代函数
static int f_lines (lua_State *L) {
  tofile(L);  /* 检查是否为有效的文件句柄 */
  aux_lines(L, 0);
  return 1;
}

# 返回'io.lines'的迭代函数，如果文件需要关闭，也返回文件本身作为第二个结果
static int io_lines (lua_State *L) {
  int toclose;
  if (lua_isnone(L, 1)) lua_pushnil(L);  /* 至少一个参数 */
  if (lua_isnil(L, 1)) {  /* 没有文件名？ */
    lua_getfield(L, LUA_REGISTRYINDEX, IO_INPUT);  /* 获取默认输入 */
    lua_replace(L, 1);  /* 将其放在索引1处 */
    tofile(L);  /* 检查是否为有效的文件句柄 */
    toclose = 0;  /* 迭代结束后不关闭文件 */
  }
  else {  /* 打开一个新文件 */
    // 从 Lua 栈中获取第一个参数作为文件名，并将其转换为 C 字符串
    const char *filename = luaL_checkstring(L, 1);
    // 检查文件是否成功打开，如果失败则抛出错误
    opencheck(L, filename, "r");
    // 将文件放置在索引 1 的位置
    lua_replace(L, 1);  /* put file at index 1 */
    // 在迭代结束后关闭文件
    toclose = 1;  /* close it after iteration */
  }
  // 调用辅助函数 aux_lines，将迭代函数推入 Lua 栈
  aux_lines(L, toclose);  /* push iteration function */
  // 如果需要关闭文件
  if (toclose) {
    // 将状态和控制值推入 Lua 栈
    lua_pushnil(L);  /* state */
    lua_pushnil(L);  /* control */
    // 将文件推入 Lua 栈，作为第四个返回值
    lua_pushvalue(L, 1);  /* file is the to-be-closed variable (4th result) */
    // 返回 4 个结果
    return 4;
  }
  // 如果不需要关闭文件
  else
    // 返回 1 个结果
    return 1;
/*
** {======================================================
** READ
** =======================================================
*/

/* maximum length of a numeral */
#if !defined (L_MAXLENNUM)
#define L_MAXLENNUM     200
#endif

/* auxiliary structure used by 'read_number' */
typedef struct {
  FILE *f;  /* file being read */
  int c;  /* current character (look ahead) */
  int n;  /* number of elements in buffer 'buff' */
  char buff[L_MAXLENNUM + 1];  /* +1 for ending '\0' */
} RN;

/*
** Add current char to buffer (if not out of space) and read next one
*/
static int nextc (RN *rn) {
  if (l_unlikely(rn->n >= L_MAXLENNUM)) {  /* buffer overflow? */
    rn->buff[0] = '\0';  /* invalidate result */
    return 0;  /* fail */
  }
  else {
    rn->buff[rn->n++] = rn->c;  /* save current char */
    rn->c = l_getc(rn->f);  /* read next one */
    return 1;
  }
}

/*
** Accept current char if it is in 'set' (of size 2)
*/
static int test2 (RN *rn, const char *set) {
  if (rn->c == set[0] || rn->c == set[1])
    return nextc(rn);
  else return 0;
}

/*
** Read a sequence of (hex)digits
*/
static int readdigits (RN *rn, int hex) {
  int count = 0;
  while ((hex ? isxdigit(rn->c) : isdigit(rn->c)) && nextc(rn))
    count++;
  return count;
}

/*
** Read a number: first reads a valid prefix of a numeral into a buffer.
** Then it calls 'lua_stringtonumber' to check whether the format is
** correct and to convert it to a Lua number.
*/
static int read_number (lua_State *L, FILE *f) {
  RN rn;
  int count = 0;
  int hex = 0;
  char decp[2];
  rn.f = f; rn.n = 0;
  decp[0] = lua_getlocaledecpoint();  /* get decimal point from locale */
  decp[1] = '.';  /* always accept a dot */
  l_lockfile(rn.f);
  do { rn.c = l_getc(rn.f); } while (isspace(rn.c));  /* skip spaces */
  test2(&rn, "-+");  /* optional sign */
  if (test2(&rn, "00")) {
    if (test2(&rn, "xX")) hex = 1;  /* numeral is hexadecimal */
    else count = 1;  /* 如果不是数字，则将 count 设置为 1，表示初始的 '0' 是有效数字 */
  }
  count += readdigits(&rn, hex);  /* 读取整数部分 */
  if (test2(&rn, decp))  /* 是否有小数点？ */
    count += readdigits(&rn, hex);  /* 读取小数部分 */
  if (count > 0 && test2(&rn, (hex ? "pP" : "eE"))) {  /* 是否有指数标记？ */
    test2(&rn, "-+");  /* 指数的正负号 */
    readdigits(&rn, 0);  /* 读取指数部分的数字 */
  }
  ungetc(rn.c, rn.f);  /* 撤销预读取的字符 */
  l_unlockfile(rn.f);
  rn.buff[rn.n] = '\0';  /* 结束字符串 */
  if (l_likely(lua_stringtonumber(L, rn.buff)))
    return 1;  /* 是有效数字 */
  else {  /* 格式无效 */
   lua_pushnil(L);  /* 将 "result" 推入栈中，稍后移除 */
   return 0;  /* 读取失败 */
  }
}
// 检测文件是否已经到达末尾
static int test_eof (lua_State *L, FILE *f) {
  int c = getc(f);  // 读取一个字符
  ungetc(c, f);  // 将字符放回流中，当 c == EOF 时无效
  lua_pushliteral(L, "");  // 将空字符串压入栈中
  return (c != EOF);  // 返回是否已经到达文件末尾
}

// 读取一行内容
static int read_line (lua_State *L, FILE *f, int chop) {
  luaL_Buffer b;  // 创建缓冲区
  int c;  // 用于存储读取的字符
  luaL_buffinit(L, &b);  // 初始化缓冲区
  do {  // 可能需要读取多个块才能得到整行内容
    char *buff = luaL_prepbuffer(&b);  // 预分配缓冲区空间
    int i = 0;  // 初始化索引
    l_lockfile(f);  // 锁定文件，确保在锁定期间不会发生内存错误
    while (i < LUAL_BUFFERSIZE && (c = l_getc(f)) != EOF && c != '\n')
      buff[i++] = c;  // 读取直到行尾或缓冲区限制
    l_unlockfile(f);  // 解锁文件
    luaL_addsize(&b, i);  // 更新缓冲区大小
  } while (c != EOF && c != '\n');  // 重复直到行尾或文件末尾
  if (!chop && c == '\n')  // 需要换行符且存在？
    luaL_addchar(&b, c);  // 将换行符添加到结果中
  luaL_pushresult(&b);  // 关闭缓冲区
  // 如果读取到内容（换行符或其他内容），则返回成功
  return (c == '\n' || lua_rawlen(L, -1) > 0);
}

// 读取整个文件内容
static void read_all (lua_State *L, FILE *f) {
  size_t nr;  // 实际读取的字节数
  luaL_Buffer b;  // 创建缓冲区
  luaL_buffinit(L, &b);  // 初始化缓冲区
  do {  // 每次读取 LUAL_BUFFERSIZE 字节的文件内容
    char *p = luaL_prepbuffer(&b);  // 预分配缓冲区空间
    nr = fread(p, sizeof(char), LUAL_BUFFERSIZE, f);  // 读取文件内容
    luaL_addsize(&b, nr);  // 更新缓冲区大小
  } while (nr == LUAL_BUFFERSIZE);  // 当读取的字节数等于缓冲区大小时继续读取
  luaL_pushresult(&b);  // 关闭缓冲区
}

// 读取指定数量的字符
static int read_chars (lua_State *L, FILE *f, size_t n) {
  size_t nr;  // 实际读取的字符数
  char *p;  // 用于存储读取的字符
  luaL_Buffer b;  // 创建缓冲区
  luaL_buffinit(L, &b);  // 初始化缓冲区
  p = luaL_prepbuffsize(&b, n);  // 准备缓冲区以读取整个块
  nr = fread(p, sizeof(char), n, f);  // 尝试读取 n 个字符
  luaL_addsize(&b, nr);  // 更新缓冲区大小
  luaL_pushresult(&b);  // 关闭缓冲区
  return (nr > 0);  // 如果读取到内容则返回 true
}

// 读取文件内容
static int g_read (lua_State *L, FILE *f, int first) {
  int nargs = lua_gettop(L) - 1;  // 获取参数个数
  int n, success;  // 定义变量
  clearerr(f);  // 清除文件错误标志
  if (nargs == 0) {  // 没有参数？
    success = read_line(L, f, 1);  # 调用 read_line 函数，读取一行数据并返回成功与否
    n = first + 1;  /* to return 1 result */  # 设置 n 的值为 first + 1，以返回一个结果
  }
  else {
    /* ensure stack space for all results and for auxlib's buffer */
    luaL_checkstack(L, nargs+LUA_MINSTACK, "too many arguments");  # 检查堆栈空间是否足够存储所有结果和辅助库的缓冲区
    success = 1;  # 设置 success 的值为 1
    for (n = first; nargs-- && success; n++) {  # 循环遍历参数列表
      if (lua_type(L, n) == LUA_TNUMBER) {  # 如果参数类型为数字
        size_t l = (size_t)luaL_checkinteger(L, n);  # 将参数转换为整数
        success = (l == 0) ? test_eof(L, f) : read_chars(L, f, l);  # 如果 l 为 0，则调用 test_eof 函数，否则调用 read_chars 函数
      }
      else {
        const char *p = luaL_checkstring(L, n);  # 将参数转换为字符串
        if (*p == '*') p++;  /* skip optional '*' (for compatibility) */  # 如果字符串以 '*' 开头，则跳过 '*' 字符
        switch (*p) {  # 根据字符串的第一个字符进行判断
          case 'n':  # 如果是 'n'
            success = read_number(L, f);  # 调用 read_number 函数
            break;
          case 'l':  # 如果是 'l'
            success = read_line(L, f, 1);  # 调用 read_line 函数，读取一行数据
            break;
          case 'L':  # 如果是 'L'
            success = read_line(L, f, 0);  # 调用 read_line 函数，读取一行数据（包括换行符）
            break;
          case 'a':  # 如果是 'a'
            read_all(L, f);  /* read entire file */  # 调用 read_all 函数，读取整个文件
            success = 1;  # 设置 success 的值为 1
            break;
          default:
            return luaL_argerror(L, n, "invalid format");  # 返回参数错误
        }
      }
    }
  }
  if (ferror(f))  # 如果文件发生错误
    return luaL_fileresult(L, 0, NULL);  # 返回文件操作结果
  if (!success) {
    lua_pop(L, 1);  /* remove last result */  # 移除最后一个结果
    luaL_pushfail(L);  /* push nil instead */  # 推入 nil 代替
  }
  return n - first;  # 返回 n 与 first 的差值
/* 读取函数，接受 Lua 状态机和输入输出文件对象作为参数 */
static int io_read (lua_State *L) {
  return g_read(L, getiofile(L, IO_INPUT), 1);  /* 调用 g_read 函数，传入 Lua 状态机、IO_INPUT 对应的文件对象和参数 1 */
}


/* 文件读取函数，接受 Lua 状态机作为参数 */
static int f_read (lua_State *L) {
  return g_read(L, tofile(L), 2);  /* 调用 g_read 函数，传入 Lua 状态机、tofile(L) 返回的文件对象和参数 2 */
}


/*
** 'lines' 的迭代函数
*/
static int io_readline (lua_State *L) {
  LStream *p = (LStream *)lua_touserdata(L, lua_upvalueindex(1));  /* 获取上值索引 1 对应的用户数据，并转换为 LStream 结构体指针 */
  int i;
  int n = (int)lua_tointeger(L, lua_upvalueindex(2));  /* 获取上值索引 2 对应的整数值 */
  if (isclosed(p))  /* 文件已关闭？ */
    return luaL_error(L, "file is already closed");  /* 抛出 Lua 错误，文件已关闭 */
  lua_settop(L , 1);  /* 设置栈顶为 1 */
  luaL_checkstack(L, n, "too many arguments");  /* 检查栈空间是否足够，否则抛出错误 */
  for (i = 1; i <= n; i++)  /* 将参数推入 'g_read' 函数 */
    lua_pushvalue(L, lua_upvalueindex(3 + i));
  n = g_read(L, p->f, 2);  /* 调用 g_read 函数，传入 Lua 状态机、文件指针和参数 2，返回结果数量 */
  lua_assert(n > 0);  /* 断言结果数量大于 0 */
  if (lua_toboolean(L, -n))  /* 是否至少读取到一个值？ */
    return n;  /* 返回结果数量 */
  else {  /* 第一个结果为 false: EOF 或错误 */
    if (n > 1) {  /* 是否有错误信息？ */
      /* 第二个结果为错误消息 */
      return luaL_error(L, "%s", lua_tostring(L, -n + 1));  /* 抛出 Lua 错误，错误消息为栈顶第 -n + 1 个值 */
    }
    if (lua_toboolean(L, lua_upvalueindex(3))) {  /* 生成器创建了文件？ */
      lua_settop(L, 0);  /* 清空栈 */
      lua_pushvalue(L, lua_upvalueindex(1));  /* 将文件推入栈顶 */
      aux_close(L);  /* 关闭文件 */
    }
    return 0;  /* 返回 0 */
  }
}

/* }====================================================== */


/* 写入函数，接受 Lua 状态机、文件指针和参数作为参数 */
static int g_write (lua_State *L, FILE *f, int arg) {
  int nargs = lua_gettop(L) - arg;  /* 获取参数数量 */
  int status = 1;  /* 状态标志，默认为 1 */
  for (; nargs--; arg++) {  /* 遍历参数 */
    if (lua_type(L, arg) == LUA_TNUMBER) {  /* 参数类型为数字 */
      /* 优化：可以像字符串一样精确处理 */
      int len = lua_isinteger(L, arg)  /* 判断是否为整数 */
                ? fprintf(f, LUA_INTEGER_FMT,  /* 格式化输出整数 */
                             (LUAI_UACINT)lua_tointeger(L, arg))
                : fprintf(f, LUA_NUMBER_FMT,  /* 格式化输出数字 */
                             (LUAI_UACNUMBER)lua_tonumber(L, arg));
      status = status && (len > 0);  /* 更新状态标志 */
    }
    else {
      // 声明变量l，用于存储字符串的长度
      size_t l;
      // 从Lua栈中获取指定位置的字符串，并返回其指针和长度
      const char *s = luaL_checklstring(L, arg, &l);
      // 将字符串s写入文件f，返回值与字符串长度比较，更新status状态
      status = status && (fwrite(s, sizeof(char), l, f) == l);
    }
  }
  // 如果status为真，说明文件句柄已经在栈顶，返回1
  if (l_likely(status))
    return 1;  /* file handle already on stack top */
  // 否则调用luaL_fileresult函数，返回处理结果
  else return luaL_fileresult(L, status, NULL);
# 写入函数，接收 Lua 状态机和文件对象作为参数
static int io_write (lua_State *L) {
  # 调用 g_write 函数，传入 Lua 状态机、获取到的输出文件对象和参数数量
  return g_write(L, getiofile(L, IO_OUTPUT), 1);
}

# 文件写入函数，接收 Lua 状态机作为参数
static int f_write (lua_State *L) {
  # 获取文件指针
  FILE *f = tofile(L);
  # 将文件对象推入栈顶，以便返回
  lua_pushvalue(L, 1);
  # 调用 g_write 函数，传入 Lua 状态机、文件指针和参数数量
  return g_write(L, f, 2);
}

# 文件定位函数，接收 Lua 状态机作为参数
static int f_seek (lua_State *L) {
  # 定义文件定位模式和模式名称
  static const int mode[] = {SEEK_SET, SEEK_CUR, SEEK_END};
  static const char *const modenames[] = {"set", "cur", "end", NULL};
  # 获取文件指针
  FILE *f = tofile(L);
  # 检查并获取定位模式
  int op = luaL_checkoption(L, 2, "cur", modenames);
  # 获取第三个参数，如果不存在则默认为 0
  lua_Integer p3 = luaL_optinteger(L, 3, 0);
  # 将第三个参数转换为文件定位数值
  l_seeknum offset = (l_seeknum)p3;
  # 检查第三个参数是否为整数且在合适的范围内
  luaL_argcheck(L, (lua_Integer)offset == p3, 3, "not an integer in proper range");
  # 调用 l_fseek 函数进行文件定位
  op = l_fseek(f, offset, mode[op]);
  # 如果文件定位失败，则返回错误信息
  if (l_unlikely(op))
    return luaL_fileresult(L, 0, NULL);  /* error */
  # 否则，将当前文件位置推入栈顶并返回
  else {
    lua_pushinteger(L, (lua_Integer)l_ftell(f));
    return 1;
  }
}

# 设置文件缓冲区函数，接收 Lua 状态机作为参数
static int f_setvbuf (lua_State *L) {
  # 定义缓冲区模式和模式名称
  static const int mode[] = {_IONBF, _IOFBF, _IOLBF};
  static const char *const modenames[] = {"no", "full", "line", NULL};
  # 获取文件指针
  FILE *f = tofile(L);
  # 检查并获取缓冲区模式
  int op = luaL_checkoption(L, 2, NULL, modenames);
  # 获取第三个参数，如果不存在则默认为 LUAL_BUFFERSIZE
  lua_Integer sz = luaL_optinteger(L, 3, LUAL_BUFFERSIZE);
  # 调用 setvbuf 函数设置文件缓冲区
  int res = setvbuf(f, NULL, mode[op], (size_t)sz);
  # 返回设置结果
  return luaL_fileresult(L, res == 0, NULL);
}

# 刷新输出流函数，接收 Lua 状态机作为参数
static int io_flush (lua_State *L) {
  # 返回刷新输出流的结果
  return luaL_fileresult(L, fflush(getiofile(L, IO_OUTPUT)) == 0, NULL);
}

# 刷新文件流函数，接收 Lua 状态机作为参数
static int f_flush (lua_State *L) {
  # 返回刷新文件流的结果
  return luaL_fileresult(L, fflush(tofile(L)) == 0, NULL);
}

# 'io' 库的函数列表
static const luaL_Reg iolib[] = {
  {"close", io_close},
  {"flush", io_flush},
  {"input", io_input},
  {"lines", io_lines},
  {"open", io_open},
  {"output", io_output},
  {"popen", io_popen},
  {"read", io_read},
  {"tmpfile", io_tmpfile},
  {"type", io_type},
  {"write", io_write},
  {NULL, NULL}
};

# 文件句柄的方法
# 定义文件操作方法的映射表
static const luaL_Reg meth[] = {
  {"read", f_read},  # 读取文件内容的方法
  {"write", f_write},  # 写入文件内容的方法
  {"lines", f_lines},  # 逐行读取文件内容的方法
  {"flush", f_flush},  # 刷新文件缓冲区的方法
  {"seek", f_seek},  # 移动文件指针的方法
  {"close", f_close},  # 关闭文件的方法
  {"setvbuf", f_setvbuf},  # 设置文件缓冲区的方法
  {NULL, NULL}  # 结束标记
};

# 文件句柄的元方法
static const luaL_Reg metameth[] = {
  {"__index", NULL},  # 占位符
  {"__gc", f_gc},  # 垃圾回收方法
  {"__close", f_gc},  # 关闭文件的方法
  {"__tostring", f_tostring},  # 转换为字符串的方法
  {NULL, NULL}  # 结束标记
};

# 创建文件句柄的元表
static void createmeta (lua_State *L) {
  luaL_newmetatable(L, LUA_FILEHANDLE);  # 创建文件句柄的元表
  luaL_setfuncs(L, metameth, 0);  # 将元方法添加到新的元表中
  luaL_newlibtable(L, meth);  # 创建方法表
  luaL_setfuncs(L, meth, 0);  # 将文件操作方法添加到方法表中
  lua_setfield(L, -2, "__index");  # 元表.__index = 方法表
  lua_pop(L, 1);  # 弹出元表
}

# 不关闭标准输入、输出、错误文件的函数
static int io_noclose (lua_State *L) {
  LStream *p = tolstream(L);
  p->closef = &io_noclose;  # 保持文件打开状态
  luaL_pushfail(L);  # 推入失败标记
  lua_pushliteral(L, "cannot close standard file");  # 推入错误信息
  return 2;  # 返回值个数
}

# 创建标准文件
static void createstdfile (lua_State *L, FILE *f, const char *k,
                           const char *fname) {
  LStream *p = newprefile(L);  # 创建文件流
  p->f = f;  # 设置文件指针
  p->closef = &io_noclose;  # 设置关闭文件的函数
  if (k != NULL) {
    lua_pushvalue(L, -1);
    lua_setfield(L, LUA_REGISTRYINDEX, k);  # 将文件添加到注册表
  }
  lua_setfield(L, -2, fname);  # 将文件添加到模块中
}

# 打开 io 模块
LUAMOD_API int luaopen_io (lua_State *L) {
  luaL_newlib(L, iolib);  # 创建新的模块
  createmeta(L);  # 创建元表
  # 创建（并设置）默认文件
  createstdfile(L, stdin, IO_INPUT, "stdin");  # 创建标准输入文件
  createstdfile(L, stdout, IO_OUTPUT, "stdout");  # 创建标准输出文件
  createstdfile(L, stderr, NULL, "stderr");  # 创建标准错误文件
  return 1;  # 返回值个数
}
```