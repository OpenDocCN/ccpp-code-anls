# `nmap\ncat\ncat_lua.c`

```cpp
/* $Id$ */

#include "ncat.h"
#include "ncat_lua.h"

static lua_State *L;  // 创建 Lua 状态对象指针
static int last_function_number;  // 存储最后一个函数的索引

static void report(char *prefix)
{
    const char *errormsg;
    errormsg = lua_tostring(L, -1);  // 获取 Lua 栈顶的错误信息
    if (errormsg == NULL)
        errormsg = "(error object is not a string)";
    bye("%s: %s.", prefix, errormsg);  // 输出错误信息
}

static int traceback (lua_State *LL)
{
    const char *msg;
    msg = lua_tostring(LL, 1);  // 获取 Lua 栈顶的消息
    if (msg) {
        luaL_traceback(LL, LL, msg, 1);  // 获取 Lua 调用堆栈的回溯信息
    } else {
        if (!lua_isnoneornil(LL, 1)) {  /* 是否有错误对象? */
            if (!luaL_callmeta(LL, 1, "__tostring"))  /* 尝试调用 'tostring' 元方法 */
                lua_pushliteral(LL, "(no error message)");  // 如果没有错误消息，则推入默认消息
        }
    }
    return 1;
}

void lua_setup(void)
{
    ncat_assert(o.cmdexec != NULL);  // 确保命令执行不为空

    L = luaL_newstate();  // 创建 Lua 状态对象
    luaL_openlibs(L);  // 打开 Lua 标准库

    if (luaL_loadfile(L, o.cmdexec) != 0)  // 加载 Lua 脚本文件
        report("Error loading the Lua script");  // 报告加载脚本错误

    /* 安装回溯函数 */
    last_function_number = lua_gettop(L);  // 获取 Lua 栈顶索引
    lua_pushcfunction(L, traceback);  // 将回溯函数推入 Lua 栈
    lua_insert(L, last_function_number);  // 将回溯函数插入到指定位置
}

void lua_run(void)
{
    if (lua_pcall(L, 0, 0, last_function_number) != LUA_OK && !lua_isnil(L, -1)) {
        /* 处理错误; 下面的代码取自 lua.c，Lua 源代码 */
        lua_remove(L, last_function_number);  // 移除最后一个函数
        report("Error running the Lua script");  // 报告运行脚本错误
    } else {
        if (o.debug)
            logdebug("%s returned successfully.\n", o.cmdexec);  // 如果调试模式开启，则输出成功信息
        lua_close(L);  // 关闭 Lua 状态对象
        exit(EXIT_SUCCESS);  // 退出程序
    }
}
```