# `nmap\nsock\src\proxy_socks4.c`

```
/* $Id $ */

#define _GNU_SOURCE
#include <stdio.h>

#include "nsock.h"
#include "nsock_internal.h"
#include "nsock_log.h"

#include <string.h>

#define DEFAULT_PROXY_PORT_SOCKS4 1080


extern struct timeval nsock_tod;
extern const struct proxy_spec ProxySpecSocks4;


struct socks4_data {
    uint8_t  version;  // 版本号
    uint8_t  type;  // 类型
    uint16_t port;  // 端口号
    uint32_t address;  // 地址
    uint8_t  null;  // 空值
} __attribute__((packed));


static int proxy_socks4_node_new(struct proxy_node **node, const struct uri *uri) {
  int rc;
  struct proxy_node *proxy;

  proxy = (struct proxy_node *)safe_zalloc(sizeof(struct proxy_node));  // 分配内存
  proxy->spec = &ProxySpecSocks4;  // 设置代理规范为Socks4

  rc = proxy_resolve(uri->host, (struct sockaddr *)&proxy->ss, &proxy->sslen, AF_INET);  // 解析代理地址
  if (rc < 0)
    goto err_out;

  if (proxy->ss.ss_family != AF_INET) {  // 如果地址族不是IPv4
    rc = -1;
    goto err_out;
  }

  if (uri->port == -1)
    proxy->port = DEFAULT_PROXY_PORT_SOCKS4;  // 如果端口号未指定，则使用默认端口号
  else
    proxy->port = (unsigned short)uri->port;  // 否则使用指定的端口号

  rc = asprintf(&proxy->nodestr, "socks4://%s:%d", uri->host, proxy->port);  // 格式化代理节点字符串
  if (rc < 0) {
    /* asprintf() failed for some reason but this is not a disaster (yet).
     * Set nodestr to NULL and try to keep on going. */
    proxy->nodestr = NULL;  // 如果格式化失败，则将nodestr设置为NULL
  }

  rc = 1;

err_out:
  if (rc < 0) {
    free(proxy);  // 释放内存
    proxy = NULL;
  }
  *node = proxy;  // 将代理节点赋值给node
  return rc;
}

static void proxy_socks4_node_delete(struct proxy_node *node) {
  if (!node)
    return;

  free(node->nodestr);  // 释放nodestr内存

  free(node);  // 释放代理节点内存
}

static inline void socks4_data_init(struct socks4_data *socks4,
                                    struct sockaddr_storage *ss, size_t sslen,
                                    unsigned short port) {
  struct sockaddr_in *sin = (struct sockaddr_in *)ss;

  memset(socks4, 0x00, sizeof(struct socks4_data));  // 初始化socks4_data结构体
  socks4->version = 4;  // 设置版本号为4
  socks4->type = 1;  // 设置类型为1
  socks4->port = htons(port);  // 设置端口号，并转换为网络字节序
  assert(ss->ss_family == AF_INET);  // 断言地址族为IPv4
  socks4->address = sin->sin_addr.s_addr;  // 设置地址
}
  // 处理初始状态的函数
static int handle_state_initial(struct npool *nsp, struct nevent *nse, void *udata) {
  // 获取代理链上下文
  struct proxy_chain_context *px_ctx = nse->iod->px_ctx;
  struct sockaddr_storage *ss;
  size_t sslen;
  unsigned short port;
  struct proxy_node *next;
  struct socks4_data socks4;
  int timeout;

  // 设置代理状态为 SOCKS4_TCP_CONNECTED
  px_ctx->px_state = PROXY_STATE_SOCKS4_TCP_CONNECTED;

  // 获取下一个代理节点的信息
  next = proxy_ctx_node_next(px_ctx);
  if (next) {
    ss    = &next->ss;
    sslen = next->sslen;
    port  = next->port;
  } else {
    ss    = &px_ctx->target_ss;
    sslen = px_ctx->target_sslen;
    port  = px_ctx->target_port;
  }

  // 初始化 SOCKS4 数据
  socks4_data_init(&socks4, ss, sslen, port);

  // 计算超时时间
  timeout = TIMEVAL_MSEC_SUBTRACT(nse->timeout, nsock_tod);

  // 发送 SOCKS4 数据
  nsock_write(nsp, (nsock_iod)nse->iod, nsock_proxy_ev_dispatch, timeout, udata,
              (char *)&socks4, sizeof(socks4));

  // 读取数据
  nsock_readbytes(nsp, (nsock_iod)nse->iod, nsock_proxy_ev_dispatch, timeout,
                  udata, 8);
  return 0;
}

// 处理 TCP 连接状态的函数
static int handle_state_tcp_connected(struct npool *nsp, struct nevent *nse, void *udata) {
  // 获取代理链上下文
  struct proxy_chain_context *px_ctx = nse->iod->px_ctx;
  char *res;
  int reslen;

  // 读取缓冲区中的数据
  res = nse_readbuf(nse, &reslen);

  // 检查 SOCKS4 回复是否有效
  if (!(reslen == 8 && res[1] == 90)) {
    struct proxy_node *node = px_ctx->px_current;

    nsock_log_debug("Ignoring invalid socks4 reply from proxy %s",
                    node->nodestr);
    return -EINVAL;
  }

  // 设置代理状态为 SOCKS4_TUNNEL_ESTABLISHED
  px_ctx->px_state = PROXY_STATE_SOCKS4_TUNNEL_ESTABLISHED;

  // 如果没有下一个代理节点，则转发事件
  if (proxy_ctx_node_next(px_ctx) == NULL) {
    forward_event(nsp, nse, udata);
  } else {
    // 否则，设置当前代理节点为下一个节点，并将代理状态设置为初始状态
    px_ctx->px_current = proxy_ctx_node_next(px_ctx);
    px_ctx->px_state   = PROXY_STATE_INITIAL;
    nsock_proxy_ev_dispatch(nsp, nse, udata);
  }
  return 0;
}

// SOCKS4 代理处理函数
static void proxy_socks4_handler(nsock_pool nspool, nsock_event nsevent, void *udata) {
  int rc = 0;
  struct npool *nsp = (struct npool *)nspool;
  struct nevent *nse = (struct nevent *)nsevent;

  // 根据代理状态进行处理
  switch (nse->iod->px_ctx->px_state) {
    # 根据代理状态进行不同的处理
    case PROXY_STATE_INITIAL:
      # 处理初始状态
      rc = handle_state_initial(nsp, nse, udata);
      break;

    case PROXY_STATE_SOCKS4_TCP_CONNECTED:
      # 如果是 SOCKS4 TCP 连接状态，并且是读事件，则处理 TCP 连接状态
      if (nse->type == NSE_TYPE_READ)
        rc = handle_state_tcp_connected(nsp, nse, udata);
      break;

    case PROXY_STATE_SOCKS4_TUNNEL_ESTABLISHED:
      # 如果是 SOCKS4 隧道建立状态，则转发事件
      forward_event(nsp, nse, udata);
      break;

    default:
      # 如果是其他状态，则输出错误信息
      fatal("Invalid proxy state!");
  }

  # 如果处理结果不为 0，则设置状态为代理错误，并转发事件
  if (rc) {
    nse->status = NSE_STATUS_PROXYERROR;
    forward_event(nsp, nse, udata);
  }
// 结构体 proxy_op 的实例化，包含了 SOCKS4 代理的操作函数
static const struct proxy_op ProxyOpsSocks4 = {
  proxy_socks4_node_new, // 创建 SOCKS4 代理节点的函数
  proxy_socks4_node_delete, // 删除 SOCKS4 代理节点的函数
  proxy_socks4_handler, // 处理 SOCKS4 代理请求的函数
};

// SOCKS4 代理的规范定义
const struct proxy_spec ProxySpecSocks4 = {
  "socks4://", // SOCKS4 代理的标识符
  PROXY_TYPE_SOCKS4, // 代理类型为 SOCKS4
  &ProxyOpsSocks4, // 指向 SOCKS4 代理操作函数的指针
};
```