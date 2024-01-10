# `nmap\nsock\src\proxy_http.c`

```
/* $Id $ */

#define _GNU_SOURCE
#include <stdio.h>

#include "nsock.h"
#include "nsock_internal.h"
#include "nsock_log.h"
#include <string.h>

#define DEFAULT_PROXY_PORT_HTTP 8080


extern struct timeval nsock_tod;
extern const struct proxy_spec ProxySpecHttp;


static int proxy_http_node_new(struct proxy_node **node, const struct uri *uri) {
  int rc;
  struct proxy_node *proxy;

  // 为代理节点分配内存空间
  proxy = (struct proxy_node *)safe_zalloc(sizeof(struct proxy_node));
  // 设置代理节点的规范为 HTTP 代理规范
  proxy->spec = &ProxySpecHttp;

  // 解析代理节点的主机名和地址
  rc = proxy_resolve(uri->host, (struct sockaddr *)&proxy->ss, &proxy->sslen, AF_UNSPEC);
  if (rc < 0) {
    free(proxy);
    *node = NULL;
    return -1;
  }

  // 如果 URI 中的端口为 -1，则使用默认的 HTTP 代理端口
  if (uri->port == -1)
    proxy->port = DEFAULT_PROXY_PORT_HTTP;
  else
    proxy->port = (unsigned short)uri->port;

  // 根据主机名和端口号生成代理节点的字符串表示
  rc = asprintf(&proxy->nodestr, "http://%s:%d", uri->host, proxy->port);
  if (rc < 0) {
    /* asprintf() failed for some reason but this is not a disaster (yet).
     * Set nodestr to NULL and try to keep on going. */
    proxy->nodestr = NULL;
  }

  *node = proxy;

  return 1;
}

static void proxy_http_node_delete(struct proxy_node *node) {
  if (!node)
    return;

  // 释放代理节点的字符串表示
  free(node->nodestr);

  // 释放代理节点的内存空间
  free(node);
}

static int handle_state_initial(struct npool *nsp, struct nevent *nse, void *udata) {
  struct proxy_chain_context *px_ctx = nse->iod->px_ctx;
  struct sockaddr_storage *ss;
  size_t sslen;
  unsigned short port;
  struct proxy_node *next;
  int timeout;

  // 设置代理状态为 HTTP TCP 连接已建立
  px_ctx->px_state = PROXY_STATE_HTTP_TCP_CONNECTED;

  // 获取下一个代理节点
  next = proxy_ctx_node_next(px_ctx);
  if (next) {
    ss    = &next->ss;
    sslen = next->sslen;
    port  = next->port;
  } else {
    ss    = &px_ctx->target_ss;
    sslen = px_ctx->target_sslen;
  }
    # 获取目标端口号
    port  = px_ctx->target_port;
  }

  # 计算超时时间
  timeout = TIMEVAL_MSEC_SUBTRACT(nse->timeout, nsock_tod);

  # 发送连接请求并指定超时时间
  nsock_printf(nsp, (nsock_iod)nse->iod, nsock_proxy_ev_dispatch,
               timeout, udata, "CONNECT %s:%d HTTP/1.1\r\n\r\n",
               inet_ntop_ez(ss, sslen), (int)port);

  # 读取连接响应
  nsock_readlines(nsp, (nsock_iod)nse->iod, nsock_proxy_ev_dispatch,
                  timeout, udata, 1);

  # 返回成功
  return 0;
# 处理 TCP 连接已建立状态的回调函数
static int handle_state_tcp_connected(struct npool *nsp, struct nevent *nse, void *udata) {
  # 获取当前事件的代理链上下文
  struct proxy_chain_context *px_ctx = nse->iod->px_ctx;
  # 定义响应字符串和长度
  char *res;
  int reslen;

  # 从事件中读取缓冲区的内容
  res = nse_readbuf(nse, &reslen);

  # 检查响应字符串是否包含 "200 OK"，如果不包含则拒绝连接
  if (!((reslen >= 15) && strstr(res, "200 OK"))) {
    # 获取当前代理节点并记录连接被拒绝的日志
    struct proxy_node *node = px_ctx->px_current;
    nsock_log_debug("Connection refused from proxy %s", node->nodestr);
    return -EINVAL;
  }

  # 设置代理链状态为 HTTP 隧道已建立
  px_ctx->px_state = PROXY_STATE_HTTP_TUNNEL_ESTABLISHED;

  # 如果没有下一个代理节点，则转发事件
  if (proxy_ctx_node_next(px_ctx) == NULL) {
    forward_event(nsp, nse, udata);
  } else {
    # 否则设置当前代理节点为下一个节点，并将代理链状态设置为初始状态，然后调度代理事件
    px_ctx->px_current = proxy_ctx_node_next(px_ctx);
    px_ctx->px_state   = PROXY_STATE_INITIAL;
    nsock_proxy_ev_dispatch(nsp, nse, udata);
  }
  return 0;
}

# 处理 HTTP 代理事件的回调函数
static void proxy_http_handler(nsock_pool nspool, nsock_event nsevent, void *udata) {
  int rc = 0;
  # 将 nspool 和 nsevent 转换为结构体指针
  struct npool *nsp = (struct npool *)nspool;
  struct nevent *nse = (struct nevent *)nsevent;

  # 根据代理链状态进行不同的处理
  switch (nse->iod->px_ctx->px_state) {
    # 如果代理链状态为初始状态，则调用处理初始状态的函数
    case PROXY_STATE_INITIAL:
      rc = handle_state_initial(nsp, nse, udata);
      break;

    # 如果代理链状态为 HTTP TCP 连接已建立，则根据事件类型调用处理 TCP 连接已建立状态的函数
    case PROXY_STATE_HTTP_TCP_CONNECTED:
      if (nse->type == NSE_TYPE_READ)
        rc = handle_state_tcp_connected(nsp, nse, udata);
      break;

    # 如果代理链状态为 HTTP 隧道已建立，则直接转发事件
    case PROXY_STATE_HTTP_TUNNEL_ESTABLISHED:
      forward_event(nsp, nse, udata);
      break;

    # 如果代理链状态为其他状态，则记录错误日志
    default:
      fatal("Invalid proxy state!");
  }

  # 如果处理函数返回错误，则将事件状态设置为代理错误，并转发事件
  if (rc) {
    nse->status = NSE_STATUS_PROXYERROR;
    forward_event(nsp, nse, udata);
  }
}

# HTTP 代理操作的定义
static const struct proxy_op ProxyOpsHttp = {
  proxy_http_node_new,  # 创建新的 HTTP 代理节点
  proxy_http_node_delete,  # 删除 HTTP 代理节点
  proxy_http_handler,  # 处理 HTTP 代理事件的回调函数
};

# HTTP 代理规范的定义
const struct proxy_spec ProxySpecHttp = {
  "http://",  # 代理规范的标识符
  PROXY_TYPE_HTTP,  # 代理类型为 HTTP
  &ProxyOpsHttp,  # 指向 HTTP 代理操作的指针
};
```