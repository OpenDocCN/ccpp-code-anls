# `nmap\nsock\src\nsock_proxy.h`

```
/* $Id$ */

#ifndef NSOCK_PROXY_H
#define NSOCK_PROXY_H

#include "gh_list.h"

#if HAVE_NETDB_H
#include <netdb.h>
#endif

#include <nsock.h>
#include <errno.h>


/* ------------------- CONSTANTS ------------------- */
// 代理类型枚举
enum nsock_proxy_type {
  PROXY_TYPE_HTTP = 0,  // HTTP 代理
  PROXY_TYPE_SOCKS4,     // SOCKS4 代理
  PROXY_TYPE_COUNT,      // 代理类型数量
};

// 代理状态枚举
enum nsock_proxy_state {
  /* Common initial state for all proxy types. */
  PROXY_STATE_INITIAL,  // 所有代理类型的初始状态

  /* HTTP proxy states. */
  PROXY_STATE_HTTP_TCP_CONNECTED,    // HTTP 代理 TCP 连接建立
  PROXY_STATE_HTTP_TUNNEL_ESTABLISHED,  // HTTP 代理隧道建立

  /* SOCKS 4 proxy states. */
  PROXY_STATE_SOCKS4_TCP_CONNECTED,  // SOCKS4 代理 TCP 连接建立
  PROXY_STATE_SOCKS4_TUNNEL_ESTABLISHED,  // SOCKS4 代理隧道建立
};


/* ------------------- STRUCTURES ------------------- */

// URI 结构体
struct uri {
  char *scheme;  // 协议
  char *user;    // 用户名
  char *pass;    // 密码
  char *host;    // 主机
  char *path;    // 路径
  int port;      // 端口
};

// 代理节点结构体
struct proxy_node {
  const struct proxy_spec *spec;  // 代理规范

  struct sockaddr_storage ss;  // 地址存储
  size_t sslen;  // 地址长度
  unsigned short port;  // 端口
  char *nodestr;  // 用于日志消息的节点字符串
  gh_lnode_t nodeq;  // 节点队列
};

// 代理链结构体
struct proxy_chain {
  gh_list_t nodes;  // 节点列表
};

// 代理链上下文结构体
struct proxy_chain_context {
  const struct proxy_chain *px_chain;  // 代理链
  struct proxy_node *px_current;  // 当前节点迭代器
  enum nsock_proxy_state px_state;  // 当前节点连接状态
  enum nse_type target_ev_type;  // 目标事件类型
  struct sockaddr_storage target_ss;  // 目标地址存储
  size_t target_sslen;  // 目标地址长度
  unsigned short target_port;  // 目标端口
  nsock_ev_handler target_handler;  // 目标事件处理器
};

#endif  // NSOCK_PROXY_H
# 定义了一个结构体 proxy_op，包含了三个函数指针，用于创建节点、删除节点和处理事件
struct proxy_op {
  int (*node_new)(struct proxy_node **node, const struct uri *uri);
  void (*node_delete)(struct proxy_node *node);
  void (*handler)(nsock_pool nspool, nsock_event nsevent, void *udata);
};

# 定义了一个结构体 proxy_spec，包含了前缀、代理类型和操作函数指针
struct proxy_spec {
  const char *prefix;
  enum nsock_proxy_type type;
  const struct proxy_op *ops;
};


/* ------------------- UTIL FUNCTIONS ------------------- */
# 定义了一个名为 proxy_resolve 的函数，用于解析主机名并返回地址信息
int proxy_resolve(const char *host, struct sockaddr *addr, size_t *addrlen, int ai_family);

# 定义了一个静态内联函数 proxy_ctx_node_next，用于获取代理链上下文中的下一个节点
static inline struct proxy_node *proxy_ctx_node_next(struct proxy_chain_context *ctx) {
  gh_lnode_t *next;

  # 断言代理链上下文和当前节点不为空
  assert(ctx);
  assert(ctx->px_current);

  # 获取下一个节点
  next = gh_lnode_next(&ctx->px_current->nodeq);
  if (!next)
    return NULL;

  # 将下一个节点转换为 proxy_node 结构体并返回
  return container_of(next, struct proxy_node, nodeq);
}


/* ------------------- PROTOTYPES ------------------- */
# 声明了一个函数，用于创建代理链上下文
struct proxy_chain_context *proxy_chain_context_new(nsock_pool nspool);
# 声明了一个函数，用于删除代理链上下文
void proxy_chain_context_delete(struct proxy_chain_context *ctx);
# 声明了一个函数，用于处理代理事件的分发
void nsock_proxy_ev_dispatch(nsock_pool nspool, nsock_event nsevent, void *udata);
# 声明了一个函数，用于转发事件
void forward_event(nsock_pool nspool, nsock_event nse, void *udata);

# 结束 NSOCK_PROXY_H 的声明
#endif /* NSOCK_PROXY_H */
```