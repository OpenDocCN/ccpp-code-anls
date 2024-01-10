# `nmap\ncat\ncat_proxy.h`

```
# 定义了一个宏，用于指定发出的随机数的有效期限，单位为秒
# 如果客户端收到一个带有 stale="true" 的 407 响应，表示随机数仍然有效但已过期
# 随机数只能使用一次，因此这个宏实际上是限制我们在将随机数放入“已使用”列表之前需要保留它们的时间限制
#define HTTP_DIGEST_NONCE_EXPIRY 10

# 定义了一个简单的分叉式 HTTP 代理服务器
# 声明了一个函数 ncat_http_server，用于启动 HTTP 代理服务器
extern int ncat_http_server(void);
```