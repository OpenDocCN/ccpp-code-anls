# `nmap\liblua\lzio.h`

```
/*
** $Id: lzio.h $
** Buffered streams
** See Copyright Notice in lua.h
*/

// 防止重复包含头文件
#ifndef lzio_h
#define lzio_h

// 包含 lua.h 头文件
#include "lua.h"

// 包含 lmem.h 头文件
#include "lmem.h"

// 定义流的结束标志
#define EOZ    (-1)            /* end of stream */

// 定义 ZIO 结构体
typedef struct Zio ZIO;

// 从流中读取一个字符
#define zgetc(z)  (((z)->n--)>0 ?  cast_uchar(*(z)->p++) : luaZ_fill(z))

// 定义 Mbuffer 结构体
typedef struct Mbuffer {
  char *buffer;
  size_t n;
  size_t buffsize;
} Mbuffer;

// 初始化缓冲区
#define luaZ_initbuffer(L, buff) ((buff)->buffer = NULL, (buff)->buffsize = 0)

// 返回缓冲区的内容
#define luaZ_buffer(buff)    ((buff)->buffer)

// 返回缓冲区的大小
#define luaZ_sizebuffer(buff)    ((buff)->buffsize)

// 返回缓冲区中当前的数据长度
#define luaZ_bufflen(buff)    ((buff)->n)

// 从缓冲区中移除指定长度的数据
#define luaZ_buffremove(buff,i)    ((buff)->n -= (i))

// 重置缓冲区
#define luaZ_resetbuffer(buff) ((buff)->n = 0)

// 调整缓冲区的大小
#define luaZ_resizebuffer(L, buff, size) \
    ((buff)->buffer = luaM_reallocvchar(L, (buff)->buffer, \
                (buff)->buffsize, size), \
    (buff)->buffsize = size)

// 释放缓冲区
#define luaZ_freebuffer(L, buff)    luaZ_resizebuffer(L, buff, 0)

// 初始化流
LUAI_FUNC void luaZ_init (lua_State *L, ZIO *z, lua_Reader reader,
                                        void *data);

// 读取下一个 n 个字节
LUAI_FUNC size_t luaZ_read (ZIO* z, void *b, size_t n);    /* read next n bytes */

/* --------- Private Part ------------------ */

// 定义 Zio 结构体
struct Zio {
  size_t n;            /* bytes still unread */
  const char *p;        /* current position in buffer */
  lua_Reader reader;        /* reader function */
  void *data;            /* additional data */
  lua_State *L;            /* Lua state (for reader) */
};

// 填充流
LUAI_FUNC int luaZ_fill (ZIO *z);

#endif
```