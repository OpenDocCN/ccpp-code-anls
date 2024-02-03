# `nmap\nsock\src\nsock_engines.c`

```cpp
/* $Id$ */

#ifdef HAVE_CONFIG_H
#include "nsock_config.h"
#endif

#include "nsock_internal.h"

#if HAVE_IOCP
  extern struct io_engine engine_iocp;
  #define ENGINE_IOCP &engine_iocp,
#else
  #define ENGINE_IOCP
#endif /* HAVE_IOCP */

#if HAVE_EPOLL
  extern struct io_engine engine_epoll;
  #define ENGINE_EPOLL &engine_epoll,
#else
  #define ENGINE_EPOLL
#endif /* HAVE_EPOLL */

#if HAVE_KQUEUE
  extern struct io_engine engine_kqueue;
  #define ENGINE_KQUEUE &engine_kqueue,
#else
  #define ENGINE_KQUEUE
#endif /* HAVE_KQUEUE */

#if HAVE_POLL
  extern struct io_engine engine_poll;
  #define ENGINE_POLL &engine_poll,
#else
  #define ENGINE_POLL
#endif /* HAVE_POLL */

/* select() based engine is the fallback engine, we assume it's always available */
extern struct io_engine engine_select;
#define ENGINE_SELECT &engine_select,

/* Available IO engines. This depends on which IO management interfaces are
 * available on your system. Engines must be sorted by order of preference */
static struct io_engine *available_engines[] = {
  ENGINE_EPOLL  // 添加 epoll 引擎到可用引擎列表
  ENGINE_KQUEUE  // 添加 kqueue 引擎到可用引擎列表
  ENGINE_POLL  // 添加 poll 引擎到可用引擎列表
  ENGINE_IOCP  // 添加 iocp 引擎到可用引擎列表
  ENGINE_SELECT  // 添加 select 引擎到可用引擎列表
  NULL  // 结束可用引擎列表
};

static char *engine_hint;  // 引擎提示信息

int posix_iod_connect(struct npool *nsp, int sockfd, const struct sockaddr *addr, socklen_t addrlen) {
  return connect(sockfd, addr, addrlen);  // 执行连接操作
}

int posix_iod_read(struct npool *nsp, int sockfd, void *buf, size_t len, int flags, struct sockaddr *src_addr, socklen_t *addrlen) {
  return recvfrom(sockfd, (char *)buf, len, flags, src_addr, addrlen);  // 执行读取操作
}

int posix_iod_write(struct npool *nsp, int sockfd, const void *buf, size_t len, int flags, const struct sockaddr *dest_addr, socklen_t addrlen) {
  struct sockaddr_storage *dest = (struct sockaddr_storage *)dest_addr;
  if (dest->ss_family == AF_UNSPEC)
    return send(sockfd, (char *)buf, len, flags);  // 如果目标地址未指定，执行发送操作
  else
    return sendto(sockfd, (char *)buf, len, flags, dest_addr, addrlen);  // 如果目标地址已指定，执行发送操作
}

struct io_operations posix_io_operations = {
  posix_iod_connect,  // 连接操作
  posix_iod_read,  // 读取操作
  posix_iod_write  // 写入操作
};

// 获取默认的 I/O 引擎
struct io_engine *get_io_engine(void) {
  struct io_engine *engine = NULL;
  int i;

  // 如果没有指定引擎提示，则选择第一个可用的引擎
  if (!engine_hint) {
    engine = available_engines[0];
  } else {
    // 遍历可用引擎列表，查找与提示相符的引擎
    for (i = 0; available_engines[i] != NULL; i++)
      if (strcmp(engine_hint, available_engines[i]->name) == 0) {
        engine = available_engines[i];
        break;
      }
  }

  // 如果没有找到合适的引擎，则输出错误信息并退出
  if (!engine)
    fatal("No suitable IO engine found! (%s)\n",
          engine_hint ? engine_hint : "no hint");

  return engine;
}

// 设置默认引擎
int nsock_set_default_engine(char *engine) {
  // 释放之前的引擎提示
  if (engine_hint)
    free(engine_hint);

  if (engine) {
    int i;

    // 遍历可用引擎列表，查找与指定引擎相符的引擎
    for (i = 0; available_engines[i] != NULL; i++) {
      if (strcmp(engine, available_engines[i]->name) == 0) {
        engine_hint = strdup(engine);
        return 0;
      }
    }
    return -1;
  }
  /* 如果 engine = NULL，表示使用默认引擎，返回 0 */
  engine_hint = NULL;
  return 0;
}

// 列出可用的引擎
const char *nsock_list_engines(void) {
  return
#if HAVE_IOCP
  "iocp "
#endif
#if HAVE_EPOLL
  "epoll "
#endif
#if HAVE_KQUEUE
  "kqueue "
#endif
#if HAVE_POLL
  "poll "
#endif
  "select";
}
```