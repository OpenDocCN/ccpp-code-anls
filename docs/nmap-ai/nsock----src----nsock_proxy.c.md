# `nmap\nsock\src\nsock_proxy.c`

```cpp
/* $Id$ */

#include "nsock.h"  // 包含自定义的头文件 nsock.h
#include "nsock_internal.h"  // 包含自定义的头文件 nsock_internal.h
#include "nsock_log.h"  // 包含自定义的头文件 nsock_log.h
#include <string.h>  // 包含标准库头文件 string.h

#define IN_RANGE(x, min, max) ((x) >= (min) && (x) <= (max))  // 定义一个宏，用于判断 x 是否在 min 和 max 范围内

static struct proxy_node *proxy_node_new(const char *proxystr, const char *end);  // 声明一个静态函数 proxy_node_new，返回类型为 struct proxy_node*

/* --- Implemented proxy backends --- */
extern const struct proxy_spec ProxySpecHttp;  // 声明一个外部的结构体变量 ProxySpecHttp
extern const struct proxy_spec ProxySpecSocks4;  // 声明一个外部的结构体变量 ProxySpecSocks4

static const struct proxy_spec *ProxyBackends[] = {  // 声明一个静态的结构体指针数组 ProxyBackends
  &ProxySpecHttp,  // 将 ProxySpecHttp 的地址存入数组
  &ProxySpecSocks4,  // 将 ProxySpecSocks4 的地址存入数组
  NULL  // 数组结束标志
};

/* A proxy chain is a comma-separated list of proxy specification strings:
 * proto://[user:pass@]host[:port] */
int nsock_proxychain_new(const char *proxystr, nsock_proxychain *chain, nsock_pool nspool) {  // 定义函数 nsock_proxychain_new，接受 proxystr、chain 和 nspool 三个参数
  struct npool *nsp = (struct npool *)nspool;  // 将 nspool 转换为结构体指针类型 npool*

  struct proxy_chain *pxc, **pchain = (struct proxy_chain **)chain;  // 声明结构体指针 pxc 和 pchain，将 chain 转换为结构体指针类型 proxy_chain*

  *pchain = NULL;  // 将 pchain 指向的地址设置为 NULL

  pxc = (struct proxy_chain *)safe_malloc(sizeof(struct proxy_chain));  // 分配内存给 pxc，大小为 proxy_chain 结构体的大小
  gh_list_init(&pxc->nodes);  // 初始化 pxc 的 nodes 列表

  if (proxystr) {  // 如果 proxystr 不为空
    const char *next = proxystr;  // 声明指针 next，指向 proxystr
    const char *end = strchr(proxystr, ',');  // 声明指针 end，指向 proxystr 中第一个逗号的位置
    struct proxy_node *proxy;  // 声明结构体指针 proxy

    while (end != NULL) {  // 当 end 不为空时循环
      proxy = proxy_node_new(next, end);  // 调用 proxy_node_new 函数创建一个新的 proxy_node
      if (!proxy)  // 如果 proxy 为空
        return -1;  // 返回 -1
      gh_list_append(&pxc->nodes, &proxy->nodeq);  // 将新创建的 proxy_node 添加到 pxc 的 nodes 列表中
      next = end + 1;  // 更新 next 指针的位置
      end = strchr(next, ',');  // 更新 end 指针的位置
    }
    proxy = proxy_node_new(next, strchr(next, '\0'));  // 调用 proxy_node_new 函数创建最后一个 proxy_node
    if (!proxy)  // 如果 proxy 为空
      return -1;  // 返回 -1
    gh_list_append(&pxc->nodes, &proxy->nodeq);  // 将最后一个 proxy_node 添加到 pxc 的 nodes 列表中
  }

  if (nsp) {  // 如果 nsp 不为空
    if (nsock_pool_set_proxychain(nspool, pxc) < 0) {  // 调用 nsock_pool_set_proxychain 函数设置代理链
      nsock_proxychain_delete(pxc);  // 删除代理链
      return -1;  // 返回 -1
    }
  }

  *pchain = pxc;  // 将 pchain 指向的地址设置为 pxc
  return 1;  // 返回 1
}

void nsock_proxychain_delete(nsock_proxychain chain) {  // 定义函数 nsock_proxychain_delete，接受 chain 作为参数
  struct proxy_chain *pchain = (struct proxy_chain *)chain;  // 将 chain 转换为结构体指针类型 proxy_chain*

  gh_lnode_t *lnode;  // 声明链表节点指针 lnode

  if (!pchain)  // 如果 pchain 为空
    return;  // 直接返回

  while ((lnode = gh_list_pop(&pchain->nodes)) != NULL) {  // 当链表不为空时循环
    struct proxy_node *node;  // 声明结构体指针 node

    node = container_of(lnode, struct proxy_node, nodeq);  // 将 lnode 转换为 proxy_node 结构体指针类型
    node->spec->ops->node_delete(node);  // 调用 node_delete 函数删除节点
  }

  gh_list_free(&pchain->nodes);  // 释放链表内存
  free(pchain);  // 释放 pchain 内存
}
# 设置代理链
int nsock_pool_set_proxychain(nsock_pool nspool, nsock_proxychain chain) {
  # 将 nspool 转换为结构体指针
  struct npool *nsp = (struct npool *)nspool;
  # 断言 nsp 不为空
  assert(nsp != NULL);

  # 如果 nsp 不为空且已经存在代理链，则返回错误
  if (nsp && nsp->px_chain) {
    nsock_log_error("Invalid call. Existing proxychain on this nsock_pool");
    return -1;
  }

  # 如果代理链中的节点数小于1，则返回错误
  if (gh_list_count(&chain->nodes) < 1) {
    nsock_log_error("Invalid call. No proxies in chain");
    return -1;
  }

  # 将代理链设置到 nspool 中
  nsp->px_chain = (struct proxy_chain *)chain;
  return 1;
}

# 创建代理链上下文
struct proxy_chain_context *proxy_chain_context_new(nsock_pool nspool) {
  # 将 nspool 转换为结构体指针
  struct npool *nsp = (struct npool *)nspool;
  struct proxy_chain_context *ctx;

  # 分配内存给代理链上下文
  ctx = (struct proxy_chain_context *)safe_malloc(sizeof(struct proxy_chain_context));
  # 将代理链设置到上下文中
  ctx->px_chain = nsp->px_chain;
  # 设置代理状态为初始状态
  ctx->px_state = PROXY_STATE_INITIAL;
  # 设置当前代理节点为代理链的第一个节点
  ctx->px_current = container_of(gh_list_first_elem(&nsp->px_chain->nodes),
                                 struct proxy_node,
                                 nodeq);
  return ctx;
}

# 删除代理链上下文
void proxy_chain_context_delete(struct proxy_chain_context *ctx) {
  free(ctx);
}

# 释放 URI 结构体内存
static void uri_free(struct uri *uri) {
  free(uri->scheme);
  free(uri->user);
  free(uri->pass);
  free(uri->host);
  free(uri->path);
}

# 将字符串转换为小写
static int lowercase(char *s) {
  char *p;

  for (p = s; *p != '\0'; p++)
    *p = tolower((int) (unsigned char) *p);

  return p - s;
}

# 获取十六进制数字的值
static int hex_digit_value(char digit) {
  static const char DIGITS[] = "0123456789abcdef";
  const char *p;

  if ((unsigned char)digit == '\0')
    return -1;

  p = strchr(DIGITS, tolower((int)(unsigned char)digit));
  if (p == NULL)
    return -1;

  return p - DIGITS;
}

# 对字符串进行百分号解码
static int percent_decode(char *s) {
  char *p, *q;

  # 跳过第一个 '%'，如果没有百分号转义，则直接返回
  q = s;
  while (*q != '\0' && *q != '%')
    q++;

  p = q;
  while (*q != '\0') {
    # 如果当前字符是 '%'，则进行 URL 编码解析
    if (*q == '%') {
      int c, d;

      # 跳过 '%'，获取十六进制字符对应的数值
      q++;
      c = hex_digit_value(*q);
      # 如果获取失败，返回错误
      if (c == -1)
        return -1;
      # 继续获取下一个十六进制字符对应的数值
      q++;
      d = hex_digit_value(*q);
      # 如果获取失败，返回错误
      if (d == -1)
        return -1;

      # 将两个十六进制字符对应的数值合并成一个字节，并存入目标字符串
      *p++ = c * 16 + d;
      # 继续处理下一个字符
      q++;
      } else {
        # 如果当前字符不是 '%'，直接将其存入目标字符串
        *p++ = *q++;
      }
  }
  # 在目标字符串末尾添加结束符
  *p = '\0';

  # 返回解析后的字符串长度
  return p - s;
# 解析 URI 中的 authority 部分，将解析结果存入 uri 结构体中
static int uri_parse_authority(const char *authority, const char *end, struct uri *uri) {
  const char *portsep;  # 定义端口分隔符的指针
  const char *host_start, *host_end;  # 定义主机起始和结束位置的指针
  const char *tail;  # 定义尾部指针

  # 检查是否存在 "user:pass@" userinfo，如果存在则不支持，返回错误
  if (strchr_p(authority, end, '@') != NULL)
    return -1;

  # 查找主机的起始和结束位置
  host_start = authority;  # 主机起始位置为 authority 的起始位置

  if (*host_start == '[') {  # 如果主机起始位置为 '['，则表示是 IPv6 地址
    host_start++;  # 移动主机起始位置指针
    if (host_start >= end || NULL == (host_end = strchr_p(host_start, end, ']'))) {  # 如果主机起始位置超出范围，或者找不到 ']'，则返回错误
      nsock_log_error("Invalid IPv6 address: %s", authority);  # 记录错误日志
      return -1;
    }

    portsep = host_end + 1;  # 端口分隔符为主机结束位置的下一个位置

  } else {  # 如果主机起始位置不是 '['，则表示是普通主机地址
    for (portsep = end; portsep > host_start && *portsep != ':'; portsep--);  # 从末尾向前查找端口分隔符 ':'

    if (portsep == host_start)  # 如果找不到端口分隔符，则将端口分隔符指向末尾
      portsep = end;
    host_end = portsep;  # 主机结束位置为端口分隔符位置
  }

  # 获取端口号
  if (portsep + 1 < end && *portsep == ':') {  # 如果端口分隔符后还有内容，并且端口分隔符为 ':'
    long n;  # 定义长整型变量

    errno = 0;  # 清空错误标志
    n = parse_long(portsep + 1, &tail);  # 解析端口号
    if (errno || tail != end || !IN_RANGE(n, 1, 65535)) {  # 如果解析出错，或者尾部位置不正确，或者端口号不在范围内
      nsock_log_error("Invalid port number (%ld), errno = %d", n, errno);  # 记录错误日志
      return -1;
    }
    uri->port = n;  # 将解析出的端口号存入 uri 结构体
  } else {
    uri->port = -1;  # 如果没有端口号，则将端口号设为 -1
  }

  # 获取主机地址
  uri->host = mkstr(host_start, host_end);  # 将主机地址存入 uri 结构体
  if (percent_decode(uri->host) < 0) {  # 如果主机地址的 URL 编码解析出错
    nsock_log_error("Invalid URL encoding in host: %s", uri->host);  # 记录错误日志
    free(uri->host);  # 释放主机地址的内存
    uri->host = NULL;  # 将主机地址指针设为 NULL
    return -1;
  }

  return 1;  # 返回解析结果
}

# 解析 URI，将解析结果存入 uri 结构体中
static int parse_uri(const char *proxystr, const char *end, struct uri *uri) {
  const char *p, *q;  # 定义指针变量

  # Scheme, section 3.1.
  p = proxystr;  # 指针 p 指向 URI 字符串的起始位置
  if (!isalpha(*p))  # 如果起始位置不是字母，则跳转到失败标签
    goto fail;

  q = p;  # 指针 q 指向指针 p 的位置
  while (isalpha(*q) || isdigit(*q) || *q == '+' || *q == '-' || *q == '.') {  # 循环直到遇到非字母、数字、加号、减号、点号的字符
    q++;  # 移动指针 q
    # 如果当前位置大于等于结束位置，则跳转到失败标签
    if (q >= end)
      goto fail;
  }

  # 如果当前位置不是冒号，则跳转到失败标签
  if (*q != ':')
      goto fail;

  # 将 URI 的 scheme 部分设置为从 p 到 q 的字符串
  uri->scheme = mkstr(p, q);

  # 将 scheme 部分转换为小写，以保证大小写不敏感
  lowercase(uri->scheme);

  # Authority, section 3.2.
  # 移动 p 到下一个位置，如果超出结束位置，则跳转到失败标签
  p = q + 1;
  if (p >= end)
    goto fail;
  # 如果 p+1 位置小于结束位置且 p 位置是 '/' 且 p+1 位置是 '/'，则执行以下操作
  if (p + 1 < end && *p == '/' && *(p + 1) == '/') {

    # 移动 p 到下一个位置，q 设置为 p
    p += 2;
    q = p;
    # 当 q 位置不是 '/' 且不是 '?' 且不是 '#' 且不是结束位置时，移动 q
    while (q < end && !(*q == '/' || *q == '?' || *q == '#' || *q == '\0'))
      q++;

    # 如果解析 authority 部分失败，则跳转到失败标签
    if (uri_parse_authority(p, q, uri) < 0) {
      goto fail;
    }

    # 将 p 设置为 q
    p = q;
  }

  # Path, section 3.3. We include the query and fragment in the path. The
  # path is also not percent-decoded because we just pass it on to the origin
  # server.
  # 将 URI 的 path 部分设置为从 p 到结束位置的字符串
  uri->path = mkstr(p, end);

  # 返回成功
  return 1;
# 释放 URI 对象占用的内存，并返回 -1 表示失败
uri_free(uri);
return -1;
}

# 创建新的代理节点
static struct proxy_node *proxy_node_new(const char *proxystr, const char *end) {
  int i;

  # 遍历代理后端列表
  for (i = 0; ProxyBackends[i] != NULL; i++) {
    const struct proxy_spec *pspec;
    size_t prefix_len;

    pspec = ProxyBackends[i];
    prefix_len = strlen(pspec->prefix);
    # 检查代理字符串是否以指定前缀开头
    if (end - proxystr > prefix_len && strncasecmp(proxystr, pspec->prefix, prefix_len) == 0) {
      struct proxy_node *proxy = NULL;
      struct uri uri;

      # 初始化 URI 结构体
      memset(&uri, 0x00, sizeof(struct uri));

      # 解析代理字符串为 URI 对象
      if (parse_uri(proxystr, end, &uri) < 0)
        break;

      # 使用代理后端的 node_new 方法初始化代理节点
      if (pspec->ops->node_new(&proxy, &uri) < 0) {
        nsock_log_error("Cannot initialize proxy node %s", proxystr);
        uri_free(&uri);
        break;
      }

      # 释放 URI 对象占用的内存
      uri_free(&uri);

      return proxy;
    }
  }
  nsock_log_error("Invalid protocol in proxy specification string: %s", proxystr);
  return NULL;
}

# 转发事件到上游
void forward_event(nsock_pool nspool, nsock_event nsevent, void *udata) {
  struct npool *nsp = (struct npool *)nspool;
  struct nevent *nse = (struct nevent *)nsevent;
  enum nse_type cached_type;
  enum nse_status cached_status;

  # 缓存事件类型和状态
  cached_type = nse->type;
  cached_status = nse->status;

  # 设置事件类型为目标事件类型
  nse->type = nse->iod->px_ctx->target_ev_type;

  # 如果事件状态不是成功，则设置状态为代理错误
  if (nse->status != NSE_STATUS_SUCCESS)
    nse->status = NSE_STATUS_PROXYERROR;

  # 记录转发事件的信息
  nsock_log_info("Forwarding event upstream: TCP connect %s (IOD #%li) EID %li",
                 nse_status2str(nse->status), nse->iod->id, nse->id);

  # 调用目标处理程序处理事件
  nse->iod->px_ctx->target_handler(nsp, nse, udata);

  # 恢复事件类型和状态
  nse->type = cached_type;
  nse->status = cached_status;
}

# 代理事件分发函数
void nsock_proxy_ev_dispatch(nsock_pool nspool, nsock_event nsevent, void *udata) {
  struct nevent *nse = (struct nevent *)nsevent;

  # 如果事件状态是成功，则调用当前代理节点的处理程序，否则转发事件到上游
  if (nse->status == NSE_STATUS_SUCCESS) {
    struct proxy_node *current;

    current = nse->iod->px_ctx->px_current;
    assert(current);
    current->spec->ops->handler(nspool, nsevent, udata);
  } else {
    forward_event(nspool, nsevent, udata);
  }
}
int proxy_resolve(const char *host, struct sockaddr *addr, size_t *addrlen, int ai_family) {
  struct addrinfo hints;  // 创建 addrinfo 结构体变量 hints
  struct addrinfo *res;  // 创建指向 addrinfo 结构体的指针 res
  int rc;  // 定义整型变量 rc

  memset(&hints, 0, sizeof(hints));  // 将 hints 结构体变量的内存清零

  hints.ai_family = ai_family;  // 设置 hints 结构体变量的 ai_family 成员为传入的 ai_family 参数
  /* All proxy types are TCP-only at the moment */
  hints.ai_socktype = SOCK_STREAM;  // 设置 hints 结构体变量的 ai_socktype 成员为 SOCK_STREAM，表示所有代理类型目前都是仅支持 TCP

  rc = getaddrinfo(host, NULL, &hints, &res);  // 调用 getaddrinfo 函数，根据 host 和 hints 获取地址信息，存储在 res 中，返回值存储在 rc 中
  if (rc) {
    nsock_log_info("getaddrinfo error: %s", gai_strerror(rc));  // 如果 getaddrinfo 返回错误，记录错误信息
    return -abs(rc);  // 返回错误码的绝对值的负数
  }

  *addr = *res->ai_addr;  // 将 res 中的地址信息拷贝到 addr 中
  *addrlen = res->ai_addrlen;  // 将 res 中的地址长度赋值给 addrlen
  freeaddrinfo(res);  // 释放 res 占用的内存
  return 1;  // 返回 1，表示解析成功
}
```