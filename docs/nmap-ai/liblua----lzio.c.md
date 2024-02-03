# `nmap\liblua\lzio.c`

```cpp
/*
** $Id: lzio.c $
** Buffered streams
** See Copyright Notice in lua.h
*/

// 定义宏 lzio_c
#define lzio_c
// 定义宏 LUA_CORE
#define LUA_CORE

// 包含预编译的头文件 lprefix.h
#include "lprefix.h"

// 包含标准库头文件
#include <string.h>

// 包含 Lua 解释器头文件
#include "lua.h"

// 包含限制头文件
#include "llimits.h"
// 包含内存头文件
#include "lmem.h"
// 包含状态头文件
#include "lstate.h"
// 包含 lzio 头文件
#include "lzio.h"

// 从输入流中读取数据填充缓冲区
int luaZ_fill (ZIO *z) {
  size_t size;
  lua_State *L = z->L;
  const char *buff;
  lua_unlock(L);  // 解锁 Lua 状态
  buff = z->reader(L, z->data, &size);  // 从输入流中读取数据
  lua_lock(L);  // 锁定 Lua 状态
  if (buff == NULL || size == 0)  // 如果读取的数据为空或者大小为0
    return EOZ;  // 返回 EOZ 表示没有更多的输入
  z->n = size - 1;  // 减去一个字符，作为返回的折扣
  z->p = buff;  // 设置指针指向读取的数据
  return cast_uchar(*(z->p++));  // 返回读取的字符
}

// 初始化输入流
void luaZ_init (lua_State *L, ZIO *z, lua_Reader reader, void *data) {
  z->L = L;  // 设置 Lua 状态
  z->reader = reader;  // 设置读取函数
  z->data = data;  // 设置数据
  z->n = 0;  // 初始化缓冲区大小为0
  z->p = NULL;  // 初始化指针为空
}

/* --------------------------------------------------------------- read --- */

// 从输入流中读取数据
size_t luaZ_read (ZIO *z, void *b, size_t n) {
  while (n) {
    size_t m;
    if (z->n == 0) {  // 如果缓冲区中没有字节
      if (luaZ_fill(z) == EOZ)  // 尝试读取更多数据
        return n;  // 没有更多的输入，返回缺少的字节数
      else {
        z->n++;  // luaZ_fill 消耗了第一个字节，将其放回
        z->p--;
      }
    }
    m = (n <= z->n) ? n : z->n;  // 取 n 和 z->n 中的最小值
    memcpy(b, z->p, m);  // 将数据从缓冲区复制到目标内存
    z->n -= m;  // 更新缓冲区大小
    z->p += m;  // 更新指针位置
    b = (char *)b + m;  // 更新目标内存位置
    n -= m;  // 更新剩余字节数
  }
  return 0;  // 返回读取成功
}
```