# Nmap源码解析 27

# `liblua/linit.c`

这段代码定义了一个名为init_c的函数，它用于初始化Lua库文件。它通过定义宏变量LINIT_C和LUA_LIB来达到这个目的。

在代码中，首先定义了一个名为init_c的函数，它被定义为函数名。接着，定义了一个名为LINIT_C的宏变量，它被定义为：
```cpparduino
#define LINIT_C
```
这个宏定义的作用是告诉编译器在编译过程中，将LINIT_C宏变量中的内容替换为定义的函数体。

接下来，定义了一个名为LUA_LIB的宏变量，它被定义为：
```cpparduino
#define LUA_LIB
```
这个宏定义的作用是告诉编译器，在编译过程中，将LUA_LIB宏变量中的内容替换为定义的库文件名。

在init_c函数体中，调用了luaL_openlibs函数，这个函数的作用是打开指定的Lua库文件，如果指定的库文件不存在，则会自动创建一个名为"lib"的目录，并将库文件名放入其中。

初始化完成后，代码会输出两行，第一行是说明一下这段代码的作用，第二行是告诉编译器这是一段注释。


```cpp
/*
** $Id: linit.c $
** Initialization of libraries for lua.c and other clients
** See Copyright Notice in lua.h
*/


#define linit_c
#define LUA_LIB

/*
** If you embed Lua in your program and need to open the standard
** libraries, call luaL_openlibs in your program. If you need a
** different set of libraries, copy this file to your project and edit
** it to suit your needs.
```

这段代码是一个Lua脚本，它的作用是加载一个名为"libargon.so"的动态库，并将其链接到应用程序中。它还允许在以后需要使用这个库时，使用以下代码打开库：
```cpp
```
```cpp
*preload* libraries, so that a later 'require' can
```
具体来说，代码首先通过`luaL_getsubtable`函数获取了`libargon.so`库在`LUA_REGISTRYINDEX`中的偏移量，然后使用`luaopen_modname`函数打开了这个库。接着，使用`lua_pushcfunction`和`lua_setfield`函数将`modname`名称存储到`modname`变量中，然后使用`lua_pop`函数移除了`PRELOAD`表。

这个代码片段是在`lprefix.h`头文件中定义的，它告诉Lua加载库时可以自动加载`libargon.so`库的预加载模块。这样，在稍后的`require`语句中，Lua就可以自动加载这个模块了。


```cpp
**
** You can also *preload* libraries, so that a later 'require' can
** open the library, which is already linked to the application.
** For that, do the following code:
**
**  luaL_getsubtable(L, LUA_REGISTRYINDEX, LUA_PRELOAD_TABLE);
**  lua_pushcfunction(L, luaopen_modname);
**  lua_setfield(L, -2, modname);
**  lua_pop(L, 1);  // remove PRELOAD table
*/

#include "lprefix.h"


#include <stddef.h>

```

这段代码是一个Lua编程语言的包含文件，它定义了一个常量数组loadedlibs，该数组包含了Lua中可用的许多不同类型的库函数。

具体来说，这段代码包含了一个Lua文件名列表数组loadedlibs，其中每个数组元素都是一个指向Lua库函数的LuaL_Reg结构体指针。这些函数允许程序在运行时使用来自不同包的函数，从而使程序能够实现代码分离和模块化。

loadedlibs数组中的每个函数都使用了一个特定的函数名称前缀，用于标识它们属于哪个Lua库。例如，LUA_GNAME对应于luaopen_base函数，它从luaopen库中打开一个包装器函数；LUA_LOADLIBNAME对应于luaopen_package函数，它从luaopen_package库中加载一个包函数。

通过包含这些函数，程序员可以使用Lua编程语言中的许多标准库函数，而无需编写自己的代码。


```cpp
#include "lua.h"

#include "lualib.h"
#include "lauxlib.h"


/*
** these libs are loaded by lua.c and are readily available to any Lua
** program
*/
static const luaL_Reg loadedlibs[] = {
  {LUA_GNAME, luaopen_base},
  {LUA_LOADLIBNAME, luaopen_package},
  {LUA_COLIBNAME, luaopen_coroutine},
  {LUA_TABLIBNAME, luaopen_table},
  {LUA_IOLIBNAME, luaopen_io},
  {LUA_OSLIBNAME, luaopen_os},
  {LUA_STRLIBNAME, luaopen_string},
  {LUA_MATHLIBNAME, luaopen_math},
  {LUA_UTF8LIBNAME, luaopen_utf8},
  {LUA_DBLIBNAME, luaopen_debug},
  {NULL, NULL}
};


```

这段代码是一个Lua脚本中的函数，名为`lualib_openlibs`。"lualib_openlibs"函数的作用是在开始执行脚本之前，检查系统上可用的库文件，并设置一个名为` loadedlibs`的函数指针数组。该函数会遍历数组` loadedlibs`中的每个函数指针，然后验证函数是否成功加载。如果函数成功加载，函数指针将传递给`luaL_requiref`函数，该函数会从函数表中删除函数指针，并将结果存储在当前的`L`状态中的` lib`命名空间中。


```cpp
LUALIB_API void luaL_openlibs (lua_State *L) {
  const luaL_Reg *lib;
  /* "require" functions from 'loadedlibs' and set results to global table */
  for (lib = loadedlibs; lib->func; lib++) {
    luaL_requiref(L, lib->name, lib->func, 1);
    lua_pop(L, 1);  /* remove lib */
  }
}


```

# `liblua/liolib.c`

这段代码是一个 C 语言代码，定义了一些宏，使用了预处理指令 #define。

具体来说，这段代码定义了一个名为 liolib_c 的函数，通过定义宏 LUA_LIB，使得输入的 Lua 代码可以使用 liolib 库。接着，代码通过 include 函数引入了 lpreet.h 头文件，可能是为了使用 lint 库进行代码检查。

然后，代码通过 include 函数引入了 errno.h 和 locale.h 头文件，可能是为了在程序运行时处理输入的异常和本地化支持。

最后，代码使用 lint 库检查了输入的 Lua 代码，如果出现潜在问题，会输出详细的错误信息。


```cpp
/*
** $Id: liolib.c $
** Standard I/O (and system) library
** See Copyright Notice in lua.h
*/

#define liolib_c
#define LUA_LIB

#include "lprefix.h"


#include <ctype.h>
#include <errno.h>
#include <locale.h>
```

**这个宏现在可以接受除了标准'fopen'模式以外的任何'fopen'调用模式。**

**注意：**

- 如果使用的是C语言规范，则'fopen'的第一个参数必须是文件名。
- 如果使用的是C语言非标准库，则需要使用较长的字符串来表示文件名，并且需要使用'l'来转义。
- 如果使用的是其他编程语言，则需要使用相应的库函数来获取文件名。
***/

int l_open(const char *filename, int mode);

```cpp

这个代码是一个Lua脚本，它实现了Linux系统中的文件操作。下面是具体的代码解析：

```java
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
```cpp

这个代码头文件包含了标准输入输出库头文件和字符串库头文件。

```c
#include "lua.h"
#include "lauxlib.h"
```cpp

这个代码中包含了一个名为'lua'的外部库，以及一个名为'lauxlib'的外部库。这两个库与Lua脚本交互，负责将Lua脚本与C库进行绑定，并执行C库函数。

```
#include "lualib.h"
```cpp

这个代码中包含了一个名为'lualib'的外部库，它很可能是为了提供Lua与C库之间的接口而编写的。

```
const char *filename;          /**< 文件名。使用这个参数指定文件名。如果使用的是C语言规范，则文件名必须是第一个字符串。否则，使用这个参数指定文件名。 */
int mode;                      /**< 文件访问模式。使用这个参数指定文件访问模式。对于大多数文件，这个值将是一个整数，表示文件类型。对于二进制文件，这个值将是一个负的整数，表示以二进制模式打开文件。对于其他类型的文件，这个值将是一个有效的文件访问模式。 */
```cpp

```
int l_open(const char *filename, int mode)
{
   int ret;
   const char *filenameptr;
   
   ret = luaL_open(filename, mode, &filenameptr);
   if (ret != LPC_OK) {
       return -1;
   }
   
   ret = luaL_getFromWin(filenameptr, filename, &filenameptr);
   if (ret != LPC_OK) {
       return -1;
   }
   
   return ret;
}
```cpp

这个函数是一个Lua内部函数，它接受一个文件名和一个文件访问模式，然后使用这些参数来打开或关闭文件。函数的第一个参数是一个指向字符串的指针，用于指定文件名。第二个参数是一个整数，用于指定文件访问模式。函数返回一个逻辑成功返回码，如果是LPC_OK，则表示文件成功打开，否则返回一个负数。

文件访问模式可以用"r"（读写模式）或"w"（写模式）加上一个可选的文件操作类型来指定，例如"rw"。函数的第三个参数返回文件对象，可以用来读取或写入文件内容。第四个参数是一个指向字符串的指针，用于指定二进制文件模式。


```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "lua.h"

#include "lauxlib.h"
#include "lualib.h"




/*
** Change this macro to accept other modes for 'fopen' besides
** the standard ones.
```cpp

这段代码是一个 C 语言中的函数，它对 `mode` 参数进行了检查，以确定它是否符合某种特定的 `mode`。

首先，它检查 `mode` 是否已经被定义为 `"rwa"` 扩展形式。如果是，函数会定义一个新的 `L_MODEEXT` 宏，表示 `"rwa"` 扩展。否则，函数直接跳过 `mode` 参数，因为它不包含 `+` 符号，并且 `mode` 后面也不是 `L_MODEEXT` 扩展形式。

接下来，函数会对 `mode` 进行扩展检查。如果 `mode` 中包含 `+` 符号，则会执行以下操作：

1. 如果 `mode` 已经扩展过，函数会从 `mode` 后面提取 `L_MODEEXT` 扩展，并检查它是否与 `strlen` 函数返回的 `L_MODEEXT` 相等。如果不是，函数就返回 `0`，表示 `mode` 不符合要求。
2. 如果 `mode` 中包含 `+` 符号，并且已经执行了第 1 步操作，函数会计算 `mode` 左移一位后的长度，并检查它是否与 `strlen` 函数返回的 `L_MODEEXT` 相等。如果是，函数就返回 `0`，表示 `mode` 不符合要求。
3. 如果 `mode` 中包含 `+` 符号，并且已经执行了第 1 和第 2 步操作，函数会尝试从 `mode` 后面提取 `L_MODEEXT` 扩展，并检查它是否与 `strlen` 函数返回的 `L_MODEEXT` 相等。如果是，函数就返回 `0`，表示 `mode` 不符合要求。

如果 `mode` 没有问题，函数就会返回 `0`，否则就会返回 `1`。


```
*/
#if !defined(l_checkmode)

/* accepted extensions to 'mode' in 'fopen' */
#if !defined(L_MODEEXT)
#define L_MODEEXT	"b"
#endif

/* Check whether 'mode' matches '[rwa]%+?[L_MODEEXT]*' */
static int l_checkmode (const char *mode) {
  return (*mode != '\0' && strchr("rwa", *(mode++)) != NULL &&
         (*mode != '+' || ((void)(++mode), 1)) &&  /* skip if char is '+' */
         (strspn(mode, L_MODEEXT) == strlen(mode)));  /* check extensions */
}

```cpp

这段代码是一个用于在Linux系统上spawn新进程的C函数。函数接收三个参数：

1. `L`：当前进程ID，用于输出spawn的新进程ID。
2. `c`：新进程的STDOUT文件描述符，用于输出新进程的stdout数据。
3. `m`：新进程的STDOUT文件大小，用于输出新进程的stdout数据大小。

函数内部的实现比较复杂，大致可以分为以下几步：

1. 调用`fflush`函数将当前进程的内存中的所有输出内容刷写到磁盘。
2. 调用`popen`函数以新进程的父子进程方式打开一个新进程。新进程的文件描述符为`c`，大小为`m`。
3. 通过`pclose`函数关闭新进程的文件描述符。
4. 将新进程的返回码（一般为0）存储在`L`参数中，并返回新进程的ID。

由于在函数内部没有其他文件操作，因此这段代码不会输出任何文件内容。


```
#endif

/*
** {======================================================
** l_popen spawns a new process connected to the current
** one through the file streams.
** =======================================================
*/

#if !defined(l_popen)		/* { */

#if defined(LUA_USE_POSIX)	/* { */

#define l_popen(L,c,m)		(fflush(NULL), popen(c,m))
#define l_pclose(L,file)	(pclose(file))

```cpp

这段代码是一个Lua脚本，主要作用是定义了两个函数，l_popen()和l_pclose()函数，用于打开和关闭文件。

l_popen()函数接受三个参数，第一个参数是一个Lua脚本执行上下文，第二个参数是一个文件描述符，第三个参数是Lua定义的文件模式(mode)，该模式可以是'r'或'w'，分别表示读写模式。函数返回一个FILE类型的指针，代表打开的文件句柄。如果函数无法打开文件，Lua会抛出错误。

l_pclose()函数接受两个参数，第一个参数是一个FILE类型的文件描述符，第二个参数是一个Lua脚本执行上下文。函数关闭文件并返回Lua的错误信息。如果函数无法关闭文件，Lua同样会抛出错误。

另外，该代码还包含一个if语句，判断Lua是否支持某种文件模式。如果文件模式不指定，则表示Lua支持所有文件模式。


```
#elif defined(LUA_USE_WINDOWS)	/* }{ */

#define l_popen(L,c,m)		(_popen(c,m))
#define l_pclose(L,file)	(_pclose(file))

#if !defined(l_checkmodep)
/* Windows accepts "[rw][bt]?" as valid modes */
#define l_checkmodep(m)	((m[0] == 'r' || m[0] == 'w') && \
  (m[1] == '\0' || ((m[1] == 'b' || m[1] == 't') && m[2] == '\0')))
#endif

#else				/* }{ */

/* ISO C definitions */
#define l_popen(L,c,m)  \
	  ((void)c, (void)m, \
	  luaL_error(L, "'popen' not supported"), \
	  (FILE*)0)
```cpp

这段代码定义了一些预处理指令和宏定义。

首先是一个定义，定义了一个名为 l_pclose 的函数，其参数为 L 和 file 类型。函数的作用是关闭文件并返回一个 void 类型的值。函数的实现被定义为 ((void)L, (void)file, -1)。这个实现是在函数声明前定义的，因此这个函数可以被任何程序调用。

接着是另一个预处理指令，以 l_checkmodep 为前缀的指令。这个指令的作用是检查给定的模块是否支持指定模式。如果支持，就返回 true，否则返回 false。模式必须以 'r' 或 'w' 作为前缀，并且后面跟着一个空字符 '\0'，模式的格式如下： rm '\r' 或 wm '\w'。这个指令的实现被定义为 l_checkmodep(m)。

最后是另一个预处理指令，以 l_abstract 和 l_finishp 为前缀的指令。这个指令的作用是在 Lua 程序中插入抽象函数，用于在程序结束时执行一些操作。这个指令的实现被定义为 l_abstract 和 l_finishp。


```
#define l_pclose(L,file)		((void)L, (void)file, -1)

#endif				/* } */

#endif				/* } */


#if !defined(l_checkmodep)
/* By default, Lua accepts only "r" or "w" as valid modes */
#define l_checkmodep(m)        ((m[0] == 'r' || m[0] == 'w') && m[1] == '\0')
#endif

/* }====================================================== */


```cpp

这段代码是一个条件编译语句，用于判断当前 Lua 脚本是否定义了 `l_getc()` 函数。如果没有定义，则执行下面的代码。如果定义了，则跳过该部分代码，否则执行 `l_getc()` 函数。

具体来说，代码分为两部分。第一部分是一个条件编译语句，用于检查 `l_getc()` 函数是否被定义。如果这个函数已经被定义，那么第二部分代码会被执行，否则直接跳过。第二部分代码是一个三元表达式，用于在 `l_getc()` 函数被定义和 `l_getc()` 函数没有被定义的情况下，分别调用不同的函数或者输出不同的值。

因此，该代码的作用是检查 `l_getc()` 函数是否被定义，并在函数定义和函数未定义的情况下输出不同的值。


```
#if !defined(l_getc)		/* { */

#if defined(LUA_USE_POSIX)
#define l_getc(f)		getc_unlocked(f)
#define l_lockfile(f)		flockfile(f)
#define l_unlockfile(f)		funlockfile(f)
#else
#define l_getc(f)		getc(f)
#define l_lockfile(f)		((void)0)
#define l_unlockfile(f)		((void)0)
#endif

#endif				/* } */


```cpp

这段代码定义了一个名为 l_fseek 的函数，用于在 Lua 解释器中进行文件偏移量的设置。函数有三个参数，分别表示文件类型、偏移量和偏移量类型。函数的作用是帮助用户在 Lua 解释器中更方便地设置文件偏移量。

具体来说，l_fseek 函数接受三个参数：

1. f：文件指针，代表要设置偏移量的文件指针。
2. o：偏移量，代表文件偏移的目标位置，可以是文件偏移量的零度（0）或者偏移量的正数。
3. w：偏移量类型，可以是 LARGE_NUMBER、LARGE_INTEGER、SCALAR 或者 STDINT。这个参数决定了 Lua 是否将偏移量转换为整数类型。

l_fseek 函数首先检查是否定义了函数本身，如果没有定义，则定义一个名为 l_fseek 的函数，这个函数包含在 l_方案（l_方案是 C传给 Lua 的一个类，继承自钾超类）内。如果已经定义，那么 l_fseek 函数将返回。


```
/*
** {======================================================
** l_fseek: configuration for longer offsets
** =======================================================
*/

#if !defined(l_fseek)		/* { */

#if defined(LUA_USE_POSIX)	/* { */

#include <sys/types.h>

#define l_fseek(f,o,w)		fseeko(f,o,w)
#define l_ftell(f)		ftello(f)
#define l_seeknum		off_t

```cpp

这段代码是一个条件分支语句，它会根据定义的条件来选择是否使用 Windows 特定的实现，还是使用 ISO C 定义的实现。

具体来说，如果定义了 `LUA_USE_WINDOWS` 并且没有定义 `_CRTIMP_TYPEINFO`，而且定义了 `_MSC_VER` 并且 `_MSC_VER` 大于或等于 1400，那么就会选择 Windows 特定的实现，直接使用 `_fseeki64` 和 `_ftelli64` 函数来实现文件输入输出。否则，就会选择 ISO C 定义的实现，使用 `fseek` 和 `ftell` 函数来实现文件输入输出。

这里 `l_fseek` 和 `l_ftell` 函数分别对应 Windows 和 ISO C 定义的 `_fseeki64` 和 `_ftelli64` 函数，而 `l_seeknum` 对应的是 `__int64` 类型，用于表示文件输入输出的整数类型。


```
#elif defined(LUA_USE_WINDOWS) && !defined(_CRTIMP_TYPEINFO) \
   && defined(_MSC_VER) && (_MSC_VER >= 1400)	/* }{ */

/* Windows (but not DDK) and Visual C++ 2005 or higher */
#define l_fseek(f,o,w)		_fseeki64(f,o,w)
#define l_ftell(f)		_ftelli64(f)
#define l_seeknum		__int64

#else				/* }{ */

/* ISO C definitions */
#define l_fseek(f,o,w)		fseek(f,o,w)
#define l_ftell(f)		ftell(f)
#define l_seeknum		long

```cpp

这是一个Lua预处理头文件，它定义了一些常量和宏，用于定义编译器和运行时使用的符号名称。

具体来说，这个文件的作用是定义了一些输出、输入和输入输出预处理规则，包括IO前缀、IOPREF_LEN、IO_INPUT、IO_OUTPUT等。

IO前缀指的是在Lua脚本中使用IO符号时，它们会被自动转换成相应的预处理规则，比如IO_INPUT和IO_OUTPUT。

IOPREF_LEN是一个宏，它计算了IO前缀的长度，使用sizeof函数得到一个包含IO前缀长度和结束标志的字符串。

LStream是一个用户定义的符号名称，它表示一个Lua脚本中的输入输出流。


```
#endif				/* } */

#endif				/* } */

/* }====================================================== */



#define IO_PREFIX	"_IO_"
#define IOPREF_LEN	(sizeof(IO_PREFIX)/sizeof(char) - 1)
#define IO_INPUT	(IO_PREFIX "input")
#define IO_OUTPUT	(IO_PREFIX "output")


typedef luaL_Stream LStream;


```cpp

这段代码定义了两个C函数：`tolstream` 和 `isclosed`。它们用于将Lua的`LStream`数据类型与C的`FILE`数据类型进行转换。

`tolstream`函数接受一个参数`L`，它是一个`FILE`指针。这个函数的作用是获取一个`FILE`指针，如果`L`是一个`FILE`指针，那么它会被返回。否则，它会抛出一个`lua_漱败`错误。

`isclosed`函数接受一个参数`p`，它是一个`LStream`指针。这个函数的作用是判断一个`LStream`是否关闭。如果`p`是一个关闭的`LStream`，它会返回`1`；否则，它会返回`0`。

这两个函数与Lua的接口紧密相连，用于在C和Lua之间进行数据类型的转换和关闭。


```
#define tolstream(L)	((LStream *)luaL_checkudata(L, 1, LUA_FILEHANDLE))

#define isclosed(p)	((p)->closef == NULL)


static int io_type (lua_State *L) {
  LStream *p;
  luaL_checkany(L, 1);
  p = (LStream *)luaL_testudata(L, 1, LUA_FILEHANDLE);
  if (p == NULL)
    luaL_pushfail(L);  /* not a file */
  else if (isclosed(p))
    lua_pushliteral(L, "closed file");
  else
    lua_pushliteral(L, "file");
  return 1;
}


```cpp

这两段代码是Lua脚本中的一部分，主要目的是将文件输出为字符串。

首先，这两段代码定义了两个名为`f_tostring`和`tofile`的函数。这些函数的作用是在Lua脚本中以字符串形式输出文件名。

`f_tostring`函数接受一个指向`lua_State`结构的变量`L`，并返回一个整数。它首先通过调用`tolstream`函数将文件输入到`L`对象中，然后判断是否已经关闭了文件。如果是，函数将返回字符串"file (closed)"；否则，函数使用`lua_pushfstring`函数将文件名以字符串形式存储到`L`对象中，并返回1。

`tofile`函数与`f_tostring`函数类似，但它将文件输出为文件名。它也接受一个指向`lua_State`结构的变量`L`，并返回一个指向文件的`FILE`结构。如果文件已经关闭，函数将返回`NULL`；否则，函数返回文件对象。

总的来说，这两个函数的主要作用是将文件输出为字符串或文件名，以便在Lua脚本中进行使用。


```
static int f_tostring (lua_State *L) {
  LStream *p = tolstream(L);
  if (isclosed(p))
    lua_pushliteral(L, "file (closed)");
  else
    lua_pushfstring(L, "file (%p)", p->f);
  return 1;
}


static FILE *tofile (lua_State *L) {
  LStream *p = tolstream(L);
  if (l_unlikely(isclosed(p)))
    luaL_error(L, "attempt to use a closed file");
  lua_assert(p->f);
  return p->f;
}


```cpp

这段代码定义了一个名为newprefile的函数，用于创建文件句柄。该函数在创建文件句柄之前先创建一个关闭的文件句柄，以确保在内存错误发生时，文件句柄处于一致的状态。

函数参数为lua_State *L表示当前Lua脚本的主循环句柄。函数返回一个LStream类型的指针，用于返回新创建的文件句柄。

函数实现中，首先创建一个名为p的LStream类型的变量，该变量在创建文件句柄之前设置为NULL，并标记为“关闭”的文件句柄。然后，将p存储在Lua中通过调用lua_newuserdatauv函数创建的LObject类型的变量中。luaL_setmetatable函数将Lua中的文件句柄设置为'filehandle'，从而可以调用Lua中的close函数。

最后，函数返回新创建的文件句柄。


```
/*
** When creating file handles, always creates a 'closed' file handle
** before opening the actual file; so, if there is a memory error, the
** handle is in a consistent state.
*/
static LStream *newprefile (lua_State *L) {
  LStream *p = (LStream *)lua_newuserdatauv(L, sizeof(LStream), 0);
  p->closef = NULL;  /* mark file handle as 'closed' */
  luaL_setmetatable(L, LUA_FILEHANDLE);
  return p;
}


/*
** Calls the 'close' function from a file handle. The 'volatile' avoids
```cpp

这段代码是一个Lua脚本，它的作用是修复Clang编译器中32位版本存在的一个错误。具体来说，这个错误会导致在某些情况下，编译器无法正确关闭已经打开的输入或输出流。

L脚本中定义了两个函数：`f_close`和`aux_close`。这两个函数都是用`lua_CFunction`定义的，表示关闭输入或输出流时需要执行的Lua函数。

`f_close`函数的作用是确保输入或输出流是已经打开的，然后调用`aux_close`函数关闭它。

`aux_close`函数的作用是在`f_close`函数返回后执行关闭操作。它首先获取输入或输出流的LStream对象，然后调用`closef`函数关闭流。由于`closef`函数是一个 volatile 函数，即对流的状态有可能会对后面的代码产生不可预期的结果，因此在函数内部会标记流为关闭，以确保在后面的代码中正确地关闭了流。最后，`aux_close`函数返回了`f_close`函数返回的返回值，以确保`f_close`函数可以正常地返回结果。


```
** a bug in some versions of the Clang compiler (e.g., clang 3.0 for
** 32 bits).
*/
static int aux_close (lua_State *L) {
  LStream *p = tolstream(L);
  volatile lua_CFunction cf = p->closef;
  p->closef = NULL;  /* mark stream as closed */
  return (*cf)(L);  /* close it */
}


static int f_close (lua_State *L) {
  tofile(L);  /* make sure argument is an open stream */
  return aux_close(L);
}


```cpp

该代码是一个Lua脚本，定义了两个静态函数，分别为`io_close`和`f_gc`。这两个函数都作用于Lua脚本的主循环（通常是主函数）。

`io_close`函数接受一个指向Lua状态的引用（通常是`lua_State *L`），并在没有任何参数的情况下使用默认输出，并返回`f_close`函数的返回值。

`f_gc`函数是一个Lua函数，它接受一个指向Lua流（通常是`LStream *p`）的引用，并检查当前是否关闭流。如果是关闭的，或者是由于函数调用导致流已经关闭，函数将调用辅助关闭（通常使用`aux_close`函数）。如果流没有关闭，并且`f`成员变量仍然有效，函数返回0。

两个函数的作用是关闭输入或输出流，这对于使用Lua作为编程工具的人来说是很有用的，尤其是当需要在运行时处理输入或输出时。


```
static int io_close (lua_State *L) {
  if (lua_isnone(L, 1))  /* no argument? */
    lua_getfield(L, LUA_REGISTRYINDEX, IO_OUTPUT);  /* use default output */
  return f_close(L);
}


static int f_gc (lua_State *L) {
  LStream *p = tolstream(L);
  if (!isclosed(p) && p->f != NULL)
    aux_close(L);  /* ignore closed and incompletely open files */
  return 0;
}


```cpp

这段代码定义了两个函数，分别是`io_fclose`和`newfile`。这两个函数的作用是关闭定期文件和创建一个新的文件读取流。

`io_fclose`函数接受一个`lua_State`结构体，并返回一个整数。它使用`fclose`函数关闭已经打开的文件描述符，并将其返回值作为参数传递给`luaL_fileresult`函数，如果函数成功返回0，否则返回错误代码。这里使用了Lua的`lua_差点`特性，可以在运行时查看函数的内部实现。

`newfile`函数接受一个`lua_State`结构体，并返回一个新的`LStream`对象的引用。它创建一个新的文件读取流，将`fopen`函数的返回值赋给`p->f`，并将`io_fclose`函数的地址赋给`p->closef`。它返回这个新的`LStream`对象。

这两个函数可以组合使用，例如：
```
int main() {
 lua_init(L);
 int res = io_fclose(L);
 printf("IO closed successfully\n");
 res = newfile(L);
 printf("IO opened successfully\n");
 res = io_fclose(L);
 printf("IO closed successfully\n");
 return 0;
}
```cpp
这段代码会在`main`函数中初始化Lua，然后使用`io_fclose`函数关闭文件，并使用`newfile`函数创建一个新的文件读取流。最后，它再次使用`io_fclose`函数关闭文件，并打印成功消息。


```
/*
** function to close regular files
*/
static int io_fclose (lua_State *L) {
  LStream *p = tolstream(L);
  int res = fclose(p->f);
  return luaL_fileresult(L, (res == 0), NULL);
}


static LStream *newfile (lua_State *L) {
  LStream *p = newprefile(L);
  p->f = NULL;
  p->closef = &io_fclose;
  return p;
}


```cpp

这两段代码是Lua L源代码中的函数，opencheck和io_open函数。

opencheck函数的作用是打开一个文件并返回其文件描述符。它接收三个参数：lua_State *L、要打开的文件名和文件模式。函数内部使用newfile函数创建一个新的文件对象 p，然后使用fopen函数打开文件，并使用fclose函数关闭文件。如果打开文件失败，函数将抛出luaL_error函数。

io_open函数的作用是打开一个文件并返回其返回值。它接收两个参数：lua_State *L 和要打开的文件名和文件模式。函数内部使用luaL_checkstring函数检查文件名是否正确，然后使用luaL_optstring函数检查文件模式是否正确。接着，使用newfile函数创建一个新的文件对象 p，并使用fopen函数打开文件，最后使用fclose函数关闭文件。函数将返回打开文件的状态，如果打开文件失败，函数将抛出luaL_error函数。


```
static void opencheck (lua_State *L, const char *fname, const char *mode) {
  LStream *p = newfile(L);
  p->f = fopen(fname, mode);
  if (l_unlikely(p->f == NULL))
    luaL_error(L, "cannot open file '%s' (%s)", fname, strerror(errno));
}


static int io_open (lua_State *L) {
  const char *filename = luaL_checkstring(L, 1);
  const char *mode = luaL_optstring(L, 2, "r");
  LStream *p = newfile(L);
  const char *md = mode;  /* to traverse/check mode */
  luaL_argcheck(L, l_checkmode(md), 2, "invalid mode");
  p->f = fopen(filename, mode);
  return (p->f == NULL) ? luaL_fileresult(L, 0, filename) : 1;
}


```cpp

这两段代码是一个Lua脚本，用于关闭打开的文件。第一个函数是在Lua中使用的`popen`函数，第二个函数是在C文件中使用`lua_popen`函数的封装。

`io_pclose`函数的作用是关闭通过`l_pclose`函数打开的文件。它将返回一个整数，如果文件成功关闭，则返回0，否则返回一个错误码。

`io_popen`函数接受两个参数：要打开的文件的文件名和打开模式（读或写）。它使用`newprefile`函数创建一个`LStream`对象，然后使用`l_checkmodep`函数检查所选模式是否正确。接着使用`l_popen`函数打开文件，并将其存储在`p`变量中。然后将`p->f`存储为`l_pclose`函数的指针，以便在文件关闭时调用`io_pclose`函数。最后，返回`p->f`是否为`NULL`，如果是，则返回0，否则返回错误码。


```
/*
** function to close 'popen' files
*/
static int io_pclose (lua_State *L) {
  LStream *p = tolstream(L);
  errno = 0;
  return luaL_execresult(L, l_pclose(L, p->f));
}


static int io_popen (lua_State *L) {
  const char *filename = luaL_checkstring(L, 1);
  const char *mode = luaL_optstring(L, 2, "r");
  LStream *p = newprefile(L);
  luaL_argcheck(L, l_checkmodep(mode), 2, "invalid mode");
  p->f = l_popen(L, filename, mode);
  p->closef = &io_pclose;
  return (p->f == NULL) ? luaL_fileresult(L, 0, filename) : 1;
}


```cpp

这两段代码是Lua Lua脚本中的函数，它们的作用如下：

1. io_tmpfile函数的作用是返回一个文件句柄，它使用了操作系统中的tmpfile函数，将生成的文件句柄返回给Lua脚本，并返回其关闭文件句柄的编号。

2. getiofile函数的作用是返回一个FILE类型的对象，它使用lua_open函数打开一个给定的文件索引，并返回一个LStream类型的文件对象。如果打开的文件失败，函数将返回一个L Failure类型的错误，否则函数返回一个FILE类型的对象，该对象包含一个指向FILE对象的指针。

这两段代码都定义在Lua Lua脚本中，并且使用了Lua的文件I/O功能，允许用户在脚本中读取或写入文件。使用这些函数，用户可以方便地在Lua脚本中读取或写入文件，而无需关心底层的文件操作细节。


```
static int io_tmpfile (lua_State *L) {
  LStream *p = newfile(L);
  p->f = tmpfile();
  return (p->f == NULL) ? luaL_fileresult(L, 0, NULL) : 1;
}


static FILE *getiofile (lua_State *L, const char *findex) {
  LStream *p;
  lua_getfield(L, LUA_REGISTRYINDEX, findex);
  p = (LStream *)lua_touserdata(L, -1);
  if (l_unlikely(isclosed(p)))
    luaL_error(L, "default %s file is closed", findex + IOPREF_LEN);
  return p->f;
}


```cpp

该代码是一个Lua脚本中的函数声明，名为"g_iofile"。函数参数包括一个Lua状态对象(L)、一个文件名(f)和一个文件模式(mode)。函数的作用是在给定的文件模式下检查文件是否存在，如果不存在，则创建并返回文件对象，否则返回文件对象。

函数的实现主要分两个步骤：

1. 如果文件不存在，则使用`lua_tostring`函数将文件名参数转换为字符串，并检查是否成功。如果成功，则调用`opencheck`函数以打开文件。如果失败，则使用`towfile`函数检查文件是否存在。如果文件存在，则将文件名作为参数传递给`tfile`函数，用于将文件内容写入到输出流中。然后，通过`lua_setfield`函数将文件名作为参数传递给Lua函数，以便在需要时可以访问文件名。

2. 如果文件存在，则使用`lua_getfield`函数读取文件内容，并将其存储在Lua状态对象中。然后，通过`lua_setfield`函数将文件名作为参数传递给Lua函数，以便在需要时可以访问文件名。

函数的返回值是一个整数，表示文件操作的的成功或失败。


```
static int g_iofile (lua_State *L, const char *f, const char *mode) {
  if (!lua_isnoneornil(L, 1)) {
    const char *filename = lua_tostring(L, 1);
    if (filename)
      opencheck(L, filename, mode);
    else {
      tofile(L);  /* check that it's a valid file handle */
      lua_pushvalue(L, 1);
    }
    lua_setfield(L, LUA_REGISTRYINDEX, f);
  }
  /* return current value */
  lua_getfield(L, LUA_REGISTRYINDEX, f);
  return 1;
}


```cpp

这段代码是C语言编写的，它涉及到GLib库中的文件输入输出操作。`lua_State`是GNU轻量级LuaState的封装接口，用于在不同操作系统上运行Lua脚本。

`io_input()`函数用于从文件中读取行，并返回一个整数。它接受一个`lua_State`切片，代表当前正在运行的Lua脚本的环境，以及一个文件名。函数使用`g_iofile()`函数从文件中读取行，并将读取的行返回给调用者。函数以`r`模式读取文件，这意味着只读取文件中的行而不是二进制内容。

`io_output()`函数用于向文件中写入数据。它同样接受一个`lua_State`切片，代表当前正在运行的Lua脚本的环境，以及一个文件名。函数使用`g_iofile()`函数从文件中写入数据，并将写入的数据返回给调用者。函数以`w`模式写入文件，这意味着向文件中写入数据。

`io_readline()`函数用于从文件中读取行，并返回一个字符串。它同样接受一个`lua_State`切片，代表当前正在运行的Lua脚本的环境，以及一个文件名。函数使用`g_iofile()`函数从文件中读取行，并将读取的行返回给调用者。函数以`r`模式读取文件，这意味着只读取文件中的行而不是二进制内容。

总结一下，这段代码定义了三个函数，用于从文件中读取、写入和读取行。这些函数接受一个`lua_State`切片和一个文件名作为参数，并将读取或写入的数据返回给调用者。


```
static int io_input (lua_State *L) {
  return g_iofile(L, IO_INPUT, "r");
}


static int io_output (lua_State *L) {
  return g_iofile(L, IO_OUTPUT, "w");
}


static int io_readline (lua_State *L);


/*
** maximum number of arguments to 'f:lines'/'io.lines' (it + 3 must fit
```cpp



这是一段用于创建一个迭代函数的定义，该函数接受一个文件名和一个布尔参数，用于控制是否在文件结束时关闭文件。迭代函数通过将文件名和文件读取作为参数传递给函数，并将它们存储在函数内部。函数内部使用了 luaL_argcheck 函数来检查函数是否接收到了符合预期数量的参数。

具体来说，该函数的作用如下：

1. 如果函数接收到了小于 MAXARGLINE (250) 的参数数量，那么函数会抛出一个 lua_桃花错误。MAXARGLINE 是一个定义在恶人大小(MAXARGLINE)之上的常量，它用于指定在函数内部创建的迭代函数可以接受的最大参数数量。

2. 函数会尝试从函数外部读取一个文件名，并将其存储在参数中，然后将其传递给 io_readline 函数，以便从文件中读取数据。

3. 函数会判断文件是否需要关闭，如果文件需要关闭，那么函数会在函数内部设置一个 toclose 参数为真。

4. 函数会使用 lua_rotate 函数来交换参数的位置，以便将参数从开始的顺序移动到它们在函数内部的位置。

5. 函数会使用 io_readline 函数来读取文件中的数据，并将它们存储在从函数内部创建的迭代函数中。该迭代函数的第一个参数是一个包含剩余参数的指针，第二个参数是一个 lua_State 类型的对象，用于保存文件读取的状态信息。

6. 最后，函数会将创建的迭代函数返回，以便可以将其用于 lua_iostream 等文件 I/O 类的接口中。


```
** in the limit for upvalues of a closure)
*/
#define MAXARGLINE	250

/*
** Auxiliary function to create the iteration function for 'lines'.
** The iteration function is a closure over 'io_readline', with
** the following upvalues:
** 1) The file being read (first value in the stack)
** 2) the number of arguments to read
** 3) a boolean, true iff file has to be closed when finished ('toclose')
** *) a variable number of format arguments (rest of the stack)
*/
static void aux_lines (lua_State *L, int toclose) {
  int n = lua_gettop(L) - 1;  /* number of arguments to read */
  luaL_argcheck(L, n <= MAXARGLINE, MAXARGLINE + 2, "too many arguments");
  lua_pushvalue(L, 1);  /* file */
  lua_pushinteger(L, n);  /* number of arguments to read */
  lua_pushboolean(L, toclose);  /* close/not close file when finished */
  lua_rotate(L, 2, 3);  /* move the three values to their positions */
  lua_pushcclosure(L, io_readline, 3 + n);
}


```cpp

该代码为Lua脚本中的两个函数，分别为`f_lines`和`io_lines`。它们的目的是用于在Lua脚本中读取或写入文件。以下是这两个函数的详细解释：

1. `f_lines`函数的作用是：如果给定的文件句柄不是一个有效的文件句柄，则执行以下操作：关闭文件并返回`1`，否则返回`1`。然后调用辅助函数`aux_lines`。

2. `io_lines`函数的作用是：如果给定的文件名不是一个有效的文件名，则执行以下操作：关闭文件并返回`1`，否则执行以下操作：打开一个新文件并返回`1`。然后调用辅助函数`aux_lines`。

辅助函数 `aux_lines` 的作用是接受一个文件句柄、文件名和是否关闭的参数。如果给定的文件句柄不是一个有效的文件句柄，函数将关闭文件并返回`1`。如果给定的文件名不是一个有效的文件名，函数将关闭文件并返回`1`。如果给定的文件句柄是一个有效的文件句柄，并且文件已打开，函数将执行文件读取或写入操作并返回`1`。


```
static int f_lines (lua_State *L) {
  tofile(L);  /* check that it's a valid file handle */
  aux_lines(L, 0);
  return 1;
}


/*
** Return an iteration function for 'io.lines'. If file has to be
** closed, also returns the file itself as a second result (to be
** closed as the state at the exit of a generic for).
*/
static int io_lines (lua_State *L) {
  int toclose;
  if (lua_isnone(L, 1)) lua_pushnil(L);  /* at least one argument */
  if (lua_isnil(L, 1)) {  /* no file name? */
    lua_getfield(L, LUA_REGISTRYINDEX, IO_INPUT);  /* get default input */
    lua_replace(L, 1);  /* put it at index 1 */
    tofile(L);  /* check that it's a valid file handle */
    toclose = 0;  /* do not close it after iteration */
  }
  else {  /* open a new file */
    const char *filename = luaL_checkstring(L, 1);
    opencheck(L, filename, "r");
    lua_replace(L, 1);  /* put file at index 1 */
    toclose = 1;  /* close it after iteration */
  }
  aux_lines(L, toclose);  /* push iteration function */
  if (toclose) {
    lua_pushnil(L);  /* state */
    lua_pushnil(L);  /* control */
    lua_pushvalue(L, 1);  /* file is the to-be-closed variable (4th result) */
    return 4;
  }
  else
    return 1;
}


```cpp

这段代码定义了一个名为 `RN` 的结构体，用于支持 `read_number` 函数的输入数据。具体来说，这个结构体包含以下成员：

1. 一个指向 `FILE` 类型的成员 `f`，用于指定输入文件的文件名和位置。
2. 一个指向 `char` 类型的成员 `c`，用于当前读取的字符。
3. 一个指向 `int` 类型的成员 `n`，用于指示 buffer 中当前读取的元素个数。
4. 一个字符数组 `buff`，用于存储当前读取的数字及其位数，其中 `L_MAXLENNUM` 是一个定义好的常量，指定了最大字符串长度。
5. 一个名为 ` buff` 的字符数组，用于存储 buffer 中当前读取的元素及其位数，其中 `L_MAXLENNUM` 是一个定义好的常量，指定了最大字符串长度。该数组在 `RN` 结构体定义中被声明为 `int`，但其实是 `char` 类型，因为 `buff` 数组中保存的是字符而非整数。

`read_number` 函数的输入数据需要包含一个整数，因此 `c` 和 `n` 成员分别用于指示输入数字的位数和当前已经读取的数字个数。而 `f` 成员用于指定输入文件的文件名和位置，以及告诉 `read_number` 函数从哪里开始读取输入数据。


```
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


```cpp

这段代码是一个 C 语言函数，名为 `nextc`，属于 `RNA`（RNA 代表 RNA，即核苷酸）类型。其作用是向缓冲区（如果 buffer 空间不足，则从输入流中读取一个字符）添加当前字符，并从输入流中读取下一个字符。

具体来说，函数在以下两种情况下行为不同：

1. 如果缓冲区满，或者读取字符超出了缓冲区可以容纳的字符数（即 `l_unlikely(rn->n >= L_MAXLENNUM)` 成立），那么函数将返回一个错误码，例如用 `rn->buff[0] = '\0'` 替换结果，或者直接返回 0。
2. 如果缓冲区为空，或者读取字符超出了可以容纳的字符数，那么函数将从输入流中读取下一个字符，并将结果存储在 `rn->c` 变量中。然后，函数将从输入流中读取下一个字符，并将其存储在 `rn->buff` 数组的下一个位置。最后，函数返回 1，表示成功完成了操作。


```
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


```cpp

这两段代码一起作用于名为 `test2` 的函数中。函数接收两个参数：一个 `RN` 类型的指针 `rn` 和一个字符串 `set`，其中 `set` 包含一个字符。

这两段代码的作用是：

1. 如果 `rn` 所接收的字符在 `set` 中，则返回 `nextc(rn)` 函数的返回值。
2. 如果 `rn` 所接收的字符不在 `set` 中，则返回 0。

`test2` 函数的实现依赖于另一个名为 `nextc` 的函数。


```
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


```cpp

39615219  292329839  334551379  180728613211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211211


```
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
    else count = 1;  /* count initial '0' as a valid digit */
  }
  count += readdigits(&rn, hex);  /* integral part */
  if (test2(&rn, decp))  /* decimal point? */
    count += readdigits(&rn, hex);  /* fractional part */
  if (count > 0 && test2(&rn, (hex ? "pP" : "eE"))) {  /* exponent mark? */
    test2(&rn, "-+");  /* exponent sign */
    readdigits(&rn, 0);  /* exponent digits */
  }
  ungetc(rn.c, rn.f);  /* unread look-ahead char */
  l_unlockfile(rn.f);
  rn.buff[rn.n] = '\0';  /* finish string */
  if (l_likely(lua_stringtonumber(L, rn.buff)))
    return 1;  /* ok, it is a valid number */
  else {  /* invalid format */
   lua_pushnil(L);  /* "result" to be removed */
   return 0;  /* read fails */
  }
}


```cpp

这段代码是一个 Lua 函数，名为 `test_eof`，它用于在 Lua 脚本中检测 EOF（End-of-File） marker 的位置，并返回相应的结果。

具体来说，这段代码做以下几件事情：

1. 读取文件中的一个字符，并将其存储在整数变量 `c` 中。
2. 如果读取到的字符是 EOF，则跳过整个函数并返回 `c`，否则继续执行。
3. 如果读取到了文件末尾（包括换行符），则返回 `true`，否则返回 `false`。
4. 返回读取结果。

代码中包含两个函数，一个名为 `read_line`，另一个名为 `test_eof`，它们都是 Lua 函数。`read_line` 函数的作用是读取文件中的一个字符，并返回其是否为换行符。`test_eof` 函数的作用是读取文件中的一个字符，并检测给定的 EOF 标记的位置，并返回相应的结果。这两个函数都是使用 `lua_file` 函数提供的功能，它们在 Lua 脚本中直接操作文件，而不需要使用 Lua 的内置函数。


```
static int test_eof (lua_State *L, FILE *f) {
  int c = getc(f);
  ungetc(c, f);  /* no-op when c == EOF */
  lua_pushliteral(L, "");
  return (c != EOF);
}


static int read_line (lua_State *L, FILE *f, int chop) {
  luaL_Buffer b;
  int c;
  luaL_buffinit(L, &b);
  do {  /* may need to read several chunks to get whole line */
    char *buff = luaL_prepbuffer(&b);  /* preallocate buffer space */
    int i = 0;
    l_lockfile(f);  /* no memory errors can happen inside the lock */
    while (i < LUAL_BUFFERSIZE && (c = l_getc(f)) != EOF && c != '\n')
      buff[i++] = c;  /* read up to end of line or buffer limit */
    l_unlockfile(f);
    luaL_addsize(&b, i);
  } while (c != EOF && c != '\n');  /* repeat until end of line */
  if (!chop && c == '\n')  /* want a newline and have one? */
    luaL_addchar(&b, c);  /* add ending newline to result */
  luaL_pushresult(&b);  /* close buffer */
  /* return ok if read something (either a newline or something else) */
  return (c == '\n' || lua_rawlen(L, -1) > 0);
}


```cpp

这两段代码是Lua编写的函数，用于从文件中读取字符。主要目的是实现在不使用无限循环的情况下，从文件中读取尽可能多字符，并返回给调用函数。

1. `read_all`函数的作用是读取文件中的所有字符，并返回给主函数（通常玩游戏时使用）。它将文件读取为Lua定义的`luaL_Buffer`类型，通过调用`luaL_prepbuffer`和`luaL_addsize`函数来逐步读取文件内容，并最终返回给调用者。主函数需要在此函数返回`true`，表示已成功从文件中读取到尽可能多的字符。

2. `read_chars`函数的作用是读取文件中的若干字符，并返回给主函数。它与`read_all`函数不同，因为它仅尝试读取指定字符数目的字符，而不是整个文件内容。`read_chars`函数接受一个`FILE`句柄和字符数组`chars`作为参数。它首先使用`luaL_prepbuffer`函数为`b`变量分配足够空间，然后使用`fread`函数从文件中读取字符。接下来，它使用`luaL_addsize`函数将读取的字节数加入`b`缓冲区。最后，它使用`luaL_pushresult`函数返回读取的字节数，并调用主函数。主函数需要在此函数返回`true`，表示已成功从文件中读取到指定字符数目的字符。


```
static void read_all (lua_State *L, FILE *f) {
  size_t nr;
  luaL_Buffer b;
  luaL_buffinit(L, &b);
  do {  /* read file in chunks of LUAL_BUFFERSIZE bytes */
    char *p = luaL_prepbuffer(&b);
    nr = fread(p, sizeof(char), LUAL_BUFFERSIZE, f);
    luaL_addsize(&b, nr);
  } while (nr == LUAL_BUFFERSIZE);
  luaL_pushresult(&b);  /* close buffer */
}


static int read_chars (lua_State *L, FILE *f, size_t n) {
  size_t nr;  /* number of chars actually read */
  char *p;
  luaL_Buffer b;
  luaL_buffinit(L, &b);
  p = luaL_prepbuffsize(&b, n);  /* prepare buffer to read whole block */
  nr = fread(p, sizeof(char), n, f);  /* try to read 'n' chars */
  luaL_addsize(&b, nr);
  luaL_pushresult(&b);  /* close buffer */
  return (nr > 0);  /* true iff read something */
}


```cpp

这段代码是一个 Lua 脚本，它在命令行中接受零或多个参数。它主要实现了以下功能：

1. 如果命令行没有提供任何参数，脚本将读取第一个文件并输出。
2. 如果命令行提供了文件名，脚本将读取该文件并输出。
3. 如果命令行提供了带参数的文件名，脚本将尝试读取该文件并输出。
4. 如果命令行读取过程中出现错误，脚本将返回 1 并打印错误消息。
5. 如果所有参数和文件读取都成功，脚本将返回文件长度。

脚本的基本结构如下：

```lua
if (nargs == 0) {    /* no arguments? */
   success = read_line(L, f, 1);
   n = first + 1;  /* to return 1 result */
}
else {
   /* ensure stack space for all results and for auxlib's buffer */
   luaL_checkstack(L, nargs+LUA_MINSTACK, "too many arguments");
   success = 1;
   for (n = first; nargs-- && success; n++) {
     if (lua_type(L, n) == LUA_TNUMBER) {
       size_t l = (size_t)luaL_checkinteger(L, n);
       success = (l == 0) ? test_eof(L, f) : read_chars(L, f, l);
     }
     else {
       const char *p = luaL_checkstring(L, n);
       if (*p == '*') p++;  /* skip optional '*' (for compatibility) */
       switch (*p) {
         case 'n':  /* number */
           success = read_number(L, f);
           break;
         case 'l':  /* line */
           success = read_line(L, f, 1);
           break;
         case 'L':  /* line with end-of-line */
           success = read_line(L, f, 0);
           break;
         case 'a':  /* file */
           read_all(L, f);  /* read entire file */
           success = 1; /* always success */
           break;
         default:
           return luaL_argerror(L, n, "invalid format");
       }
     }
   }
 }
 if (ferror(f))
   return luaL_fileresult(L, 0, NULL);
 if (!success) {
   lua_pop(L, 1);  /* remove last result */
   luaL_pushfail(L);  /* push nil instead */
 }
 return n - first;
}
```cpp

为了实现这些功能，脚本还定义了一系列函数，如下：

```lua
read_line(L, f, 1) 是一个函数，它从文件 `f` 和标准输入 `L` 中读取一行字符，并返回成功的结果（`true` 或 `false`）。函数的实现如下：

```cpplua
size_t read_line(L, f, n) {
 const char *p = fgets(f, n);
 if (!p) {
   return 0;
 }
 return strtol(p, L, 10);  /* convert to decimal number */
}
```

`test_eof(L, f)` 是一个函数，它尝试从文件 `f` 中读取并打印一个 EOF（结束文件）字符。函数的实现如下：

```cpplua
void test_eof(L, f) {
 char c;
 while ((c = fgetc(f)) != EOF);
 printf("读取了一个 EOF character: %c\n", c);
}
```

`read_number(L, f)` 是一个函数，它尝试从文件 `f` 中读取一个浮点数并返回它的值（`double` 类型）。函数的实现如下：

```cpplua
double read_number(L, f) {
 char buffer[10];
 double result = 0.0;
 while (fgets(buffer, 10, f) != NULL) {
   result = (double)result * (double)strtod(buffer, L);
 }
 return result;
}
```

`read_all(L, f)` 是一个函数，它尝试从文件 `f` 中读取并返回文件内容（包括换行符）。函数的实现如下：

```cpplua
void read_all(L, f) {
 size_t length;
 char buffer[1000];
 double result = 0.0;
 while ((length = fgets(buffer, 1000, f)) != NULL) {
   result = (double)result * (double)strtod(buffer, L);
 }
 return result;
}
```

```cpplua
luaL_checkinteger(L, n)` 是一个函数，它尝试从标准输入 `L` 中读取一个整数并返回它的值。函数的实现如下：

```lua
int luaL_checkinteger(L, n) {
 int result = (int)n;
 if (n < 0) result = -result;  /* negate the result if the number is negative */
 return result;
}
```cpp

这些函数的实现主要依赖于 Lua 的 `lua_bootstrap` 函数，它为 Lua 脚本提供了辅助函数和变量。


```
static int g_read (lua_State *L, FILE *f, int first) {
  int nargs = lua_gettop(L) - 1;
  int n, success;
  clearerr(f);
  if (nargs == 0) {  /* no arguments? */
    success = read_line(L, f, 1);
    n = first + 1;  /* to return 1 result */
  }
  else {
    /* ensure stack space for all results and for auxlib's buffer */
    luaL_checkstack(L, nargs+LUA_MINSTACK, "too many arguments");
    success = 1;
    for (n = first; nargs-- && success; n++) {
      if (lua_type(L, n) == LUA_TNUMBER) {
        size_t l = (size_t)luaL_checkinteger(L, n);
        success = (l == 0) ? test_eof(L, f) : read_chars(L, f, l);
      }
      else {
        const char *p = luaL_checkstring(L, n);
        if (*p == '*') p++;  /* skip optional '*' (for compatibility) */
        switch (*p) {
          case 'n':  /* number */
            success = read_number(L, f);
            break;
          case 'l':  /* line */
            success = read_line(L, f, 1);
            break;
          case 'L':  /* line with end-of-line */
            success = read_line(L, f, 0);
            break;
          case 'a':  /* file */
            read_all(L, f);  /* read entire file */
            success = 1; /* always success */
            break;
          default:
            return luaL_argerror(L, n, "invalid format");
        }
      }
    }
  }
  if (ferror(f))
    return luaL_fileresult(L, 0, NULL);
  if (!success) {
    lua_pop(L, 1);  /* remove last result */
    luaL_pushfail(L);  /* push nil instead */
  }
  return n - first;
}


```cpp



This is a Lua script that includes two functions, `f_read` and `io_readline`. `f_read` is a file I/O function that reads the contents of a file and returns it as a Lua value. It takes two arguments, one of which is a file pointer and the other of which is the maximum number of columns to read from the file. `io_readline` is an iterative file I/O function that reads the contents of a file and returns them in a Lua table. It takes one argument, which is a Lua table that maps file handles to Lua functions.

The `f_read` function first checks if the file handle is already closed and returns an error if it is. It then reads the contents of the file and returns it as a Lua value. If the file handle is not closed or the file does not contain any contents, the function returns `nil`.

The `io_readline` function reads the contents of the file and returns them in a Lua table. It populates the table with the contents of the file and returns it. If there is an error while reading the file, the function returns an error message. If the file handle is not closed, the function clears the stack and returns.


```
static int io_read (lua_State *L) {
  return g_read(L, getiofile(L, IO_INPUT), 1);
}


static int f_read (lua_State *L) {
  return g_read(L, tofile(L), 2);
}


/*
** Iteration function for 'lines'.
*/
static int io_readline (lua_State *L) {
  LStream *p = (LStream *)lua_touserdata(L, lua_upvalueindex(1));
  int i;
  int n = (int)lua_tointeger(L, lua_upvalueindex(2));
  if (isclosed(p))  /* file is already closed? */
    return luaL_error(L, "file is already closed");
  lua_settop(L , 1);
  luaL_checkstack(L, n, "too many arguments");
  for (i = 1; i <= n; i++)  /* push arguments to 'g_read' */
    lua_pushvalue(L, lua_upvalueindex(3 + i));
  n = g_read(L, p->f, 2);  /* 'n' is number of results */
  lua_assert(n > 0);  /* should return at least a nil */
  if (lua_toboolean(L, -n))  /* read at least one value? */
    return n;  /* return them */
  else {  /* first result is false: EOF or error */
    if (n > 1) {  /* is there error information? */
      /* 2nd result is error message */
      return luaL_error(L, "%s", lua_tostring(L, -n + 1));
    }
    if (lua_toboolean(L, lua_upvalueindex(3))) {  /* generator created file? */
      lua_settop(L, 0);  /* clear stack */
      lua_pushvalue(L, lua_upvalueindex(1));  /* push file at index 1 */
      aux_close(L);  /* close it */
    }
    return 0;
  }
}

```cpp

这段代码是一个Lua脚本，定义了一个名为`g_write`的函数，它的参数是一个`lua_State`指针、一个`FILE`指针和一个整数参数。函数的作用是在一个文件中写入一个字符串，并返回一个整数表示文件是否成功写入。

具体来说，函数的实现包括以下几个步骤：

1. 读取文件指针和整数参数；
2. 如果参数是一个整数，则进行字符串处理，也可以省略；
3. 如果参数不是一个整数，则以文件指针的读写模式读写文件内容；
4. 判断文件是否成功写入，如果成功则返回1，否则返回 LuaL_fileresult；
5. 在函数体内部，先判断文件指针是否在栈上，如果是，则说明文件已经成功写入，返回1；如果不是，则执行文件写入操作并返回文件状态。

这段代码定义的`g_write`函数主要用于在Lua脚本中读取或写入文件内容，对于不同的文件类型和参数，需要进行不同的处理。


```
/* }====================================================== */


static int g_write (lua_State *L, FILE *f, int arg) {
  int nargs = lua_gettop(L) - arg;
  int status = 1;
  for (; nargs--; arg++) {
    if (lua_type(L, arg) == LUA_TNUMBER) {
      /* optimization: could be done exactly as for strings */
      int len = lua_isinteger(L, arg)
                ? fprintf(f, LUA_INTEGER_FMT,
                             (LUAI_UACINT)lua_tointeger(L, arg))
                : fprintf(f, LUA_NUMBER_FMT,
                             (LUAI_UACNUMBER)lua_tonumber(L, arg));
      status = status && (len > 0);
    }
    else {
      size_t l;
      const char *s = luaL_checklstring(L, arg, &l);
      status = status && (fwrite(s, sizeof(char), l, f) == l);
    }
  }
  if (l_likely(status))
    return 1;  /* file handle already on stack top */
  else return luaL_fileresult(L, status, NULL);
}


```cpp

这段代码是一个 Lua 函数，用于在 Lua 脚本中输出文件内容。函数包括三个函数：io_write、f_write 和 f_seek。

1. io_write 函数接受一个 Lua 状态（L）和一个输出文件指针（FILE *）。函数的作用是返回一个整数，表示写入文件的字节数。具体来说，函数首先通过调用 g_write 函数从输入文件中读取数据，然后将读取的数据写入到输出文件中，最后返回成功返回的写入字节数。

2. f_write 函数接受一个 Lua 状态（L）和一个文件指针（FILE *）。函数的作用是返回一个整数，表示成功写入文件的字符数。具体来说，函数首先调用 tofile 函数获取输出文件指针，然后调用 g_write 函数将文件内容写入到指定位置。最后，函数将写入的字节数存储在返回值中。

3. f_seek 函数接受一个 Lua 状态（L）和一个文件指针（FILE *）。函数的作用是返回一个整数，表示成功从指定位置读取文件的字节数。具体来说，函数首先获取输出文件指针，然后调用 l_fseek 函数设置文件指针的文件类型为 Lua 指定模式（cur 或 end），并指定文件指针的初始偏移量。接着，函数调用 l_fseek 函数从文件指针处读取指定字节数并设置文件指针，最后检查文件是否成功读取。如果成功，函数将文件读取的字节数存储在返回值中。


```
static int io_write (lua_State *L) {
  return g_write(L, getiofile(L, IO_OUTPUT), 1);
}


static int f_write (lua_State *L) {
  FILE *f = tofile(L);
  lua_pushvalue(L, 1);  /* push file at the stack top (to be returned) */
  return g_write(L, f, 2);
}


static int f_seek (lua_State *L) {
  static const int mode[] = {SEEK_SET, SEEK_CUR, SEEK_END};
  static const char *const modenames[] = {"set", "cur", "end", NULL};
  FILE *f = tofile(L);
  int op = luaL_checkoption(L, 2, "cur", modenames);
  lua_Integer p3 = luaL_optinteger(L, 3, 0);
  l_seeknum offset = (l_seeknum)p3;
  luaL_argcheck(L, (lua_Integer)offset == p3, 3,
                  "not an integer in proper range");
  op = l_fseek(f, offset, mode[op]);
  if (l_unlikely(op))
    return luaL_fileresult(L, 0, NULL);  /* error */
  else {
    lua_pushinteger(L, (lua_Integer)l_ftell(f));
    return 1;
  }
}


```cpp

这两段代码是 C 语言与 Lua 语言的交互，主要目的是实现将 Lua 缓冲区与 C 语言的文件 I/O 函数进行绑定。

1. f_setvbuf 函数的作用是，当创建一个新的 Lua 上下文时，将一个文件描述符与相应的文件 I/O 模式绑定，以便在后续的 Lua 函数中使用。

L 文件描述符：当调用 f_setvbuf 时，需要提供一个 L 文件描述符。这个描述符将在函数内部用于设置文件 I/O 模式。

I/O 模式：通过 f_setvbuf 函数，可以指定文件的 I/O 模式，包括 "no"、"full" 或 "line"。这些模式分别表示不输入、输入或输出文件内容。

L 文件描述符：文件描述符是一个整数，用于跟踪文件操作的 Lua 上下文。

C 文件 I/O 函数：这两段代码分别对应了 C 语言中的 file I/O 函数，包括 fopen、fclose 和 fflush。

2. io_flush 函数的作用是，在 Lua 上下文结束时，调用 C 语言中的这三个文件 I/O 函数，将 Lua 缓冲区的内容输出到文件中。

这两个函数的作用是将 Lua 缓冲区与文件 I/O 函数进行绑定，以便在 Lua 上下文结束时，将 Lua 缓冲区的内容输出到文件中。


```
static int f_setvbuf (lua_State *L) {
  static const int mode[] = {_IONBF, _IOFBF, _IOLBF};
  static const char *const modenames[] = {"no", "full", "line", NULL};
  FILE *f = tofile(L);
  int op = luaL_checkoption(L, 2, NULL, modenames);
  lua_Integer sz = luaL_optinteger(L, 3, LUAL_BUFFERSIZE);
  int res = setvbuf(f, NULL, mode[op], (size_t)sz);
  return luaL_fileresult(L, res == 0, NULL);
}



static int io_flush (lua_State *L) {
  return luaL_fileresult(L, fflush(getiofile(L, IO_OUTPUT)) == 0, NULL);
}


```cpp

这段代码是一个Lua函数，名为`fflush`，属于`io`库。它的作用是执行与`fflush`函数相同的操作，并返回一个integer类型的结果。

`fflush`函数接受一个指向`lua_State`结构体的参数`L`，并返回一个integer类型的结果。它使用`fflush`函数来执行一个Lua文件操作，并检查其结果。如果结果为0，那么函数将返回`NULL`，否则返回L的当前状态。

`fflush`函数的实现与`luaL_fileresult`函数结合使用，这个函数会在Lua文件操作成功或失败时返回一个integer类型的值。如果文件操作成功，那么`fflush`会返回`0`，否则返回负数。


```
static int f_flush (lua_State *L) {
  return luaL_fileresult(L, fflush(tofile(L)) == 0, NULL);
}


/*
** functions for 'io' library
*/
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


```cpp

这段代码定义了一个名为`meth`的数组，包含了针对文件 handle的多种方法。这是一个Lua脚本，可能会被嵌入到其他Lua脚本中，也可以被Lua程序直接使用。

这个LuaLuaReg结构体数组`meth`存储了所有的文件 handle方法。每个方法都由一个Lua函数名`f_`开头，后面跟着一个参数表。例如，`f_read`表示文件读取方法，其参数包括一个文件描述符和读取缓冲区的位置。

这个数组方法的定义可以在应用程序的Lua脚本中使用，只要将这个数组复制到应用程序的Lua栈中。例如，当需要在应用程序中读取文件内容时，程序可以调用`file.open("example.txt")`来打开文件，然后使用`file.read`方法读取文件内容。

这个LuaLuaReg结构体数组`meth`是一个非常有用的Lua工具，因为它定义了与文件 handle相关的多种方法，使得Lua程序可以更轻松地读取、写入、设置文件内容等。


```
/*
** methods for file handles
*/
static const luaL_Reg meth[] = {
  {"read", f_read},
  {"write", f_write},
  {"lines", f_lines},
  {"flush", f_flush},
  {"seek", f_seek},
  {"close", f_close},
  {"setvbuf", f_setvbuf},
  {NULL, NULL}
};


```cpp

这段代码定义了一个名为 "metameth" 的函数数组，用于提供文件句柄的元方法。

每个元方法都对应一个 Lua 函数，通过调用这个函数可以实现与文件句柄相关的操作。这些元方法包括：__index，__gc，__close，__tostring。

函数原型定义了每个元方法对应的原型函数，以及这些元方法的名称。在调用这些元方法时，会先执行元方法定义的原型函数，如果没有定义的原型函数，则会执行 Lua 中的默认实现。

在代码的最后，定义了一个名为 "createmeta" 的函数，该函数会创建一个只读的元组（metatable），并将元方法数组和元方法函数指针作为参数传递给该函数。这个元组用于存储文件句柄的相关信息，例如：文件句柄的类型、关闭模式、是否以二进制格式输出等。


```
/*
** metamethods for file handles
*/
static const luaL_Reg metameth[] = {
  {"__index", NULL},  /* place holder */
  {"__gc", f_gc},
  {"__close", f_gc},
  {"__tostring", f_tostring},
  {NULL, NULL}
};


static void createmeta (lua_State *L) {
  luaL_newmetatable(L, LUA_FILEHANDLE);  /* metatable for file handles */
  luaL_setfuncs(L, metameth, 0);  /* add metamethods to new metatable */
  luaL_newlibtable(L, meth);  /* create method table */
  luaL_setfuncs(L, meth, 0);  /* add file methods to method table */
  lua_setfield(L, -2, "__index");  /* metatable.__index = method table */
  lua_pop(L, 1);  /* pop metatable */
}


```cpp

这段代码定义了两个名为"io_noclose"和"createstdfile"的函数。函数的作用是关闭标准文件stdin、stdout和stderr。具体来说，函数"io_noclose"接受一个指向Lua脚本对象的引用（L），并返回一个整数。如果调用此函数成功，整数将返回0；否则，将返回2。函数"createstdfile"接受一个指向标准文件（FILE）的引用、一个将作为文件名的字符串和一个将作为模块名的字符串。它将创建一个新文件，并将文件名添加到注册表中，然后将文件名添加到模块中。


```
/*
** function to (not) close the standard files stdin, stdout, and stderr
*/
static int io_noclose (lua_State *L) {
  LStream *p = tolstream(L);
  p->closef = &io_noclose;  /* keep file opened */
  luaL_pushfail(L);
  lua_pushliteral(L, "cannot close standard file");
  return 2;
}


static void createstdfile (lua_State *L, FILE *f, const char *k,
                           const char *fname) {
  LStream *p = newprefile(L);
  p->f = f;
  p->closef = &io_noclose;
  if (k != NULL) {
    lua_pushvalue(L, -1);
    lua_setfield(L, LUA_REGISTRYINDEX, k);  /* add file to registry */
  }
  lua_setfield(L, -2, fname);  /* add file to module */
}


```cpp

这段代码是一个Lua脚本，它实现了打开I/O设备的操作。具体来说，它做了以下几件事情：

1. 创建了一个名为"luaopen_io"的Lua模块。
2. 创建了一个名为"iolib"的Lua模块，这个模块可能包含了用于文件操作的函数。
3. 创建了4个标准的I/O文件，分别位于"stdin"、"stdout"、"stderr"和空目录"/dev/null"（表示标准输入、标准输出、标准错误、空输出）。
4. 通过调用LuaL_newlib函数，Lua获得了上下文，并可以调用实现了LuaIO接口的函数。
5. 返回一个表示成功打开I/O设备的整数，通常是0。


```
LUAMOD_API int luaopen_io (lua_State *L) {
  luaL_newlib(L, iolib);  /* new module */
  createmeta(L);
  /* create (and set) default files */
  createstdfile(L, stdin, IO_INPUT, "stdin");
  createstdfile(L, stdout, IO_OUTPUT, "stdout");
  createstdfile(L, stderr, NULL, "stderr");
  return 1;
}


```