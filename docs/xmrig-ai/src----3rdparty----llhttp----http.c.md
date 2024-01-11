# `xmrig\src\3rdparty\llhttp\http.c`

```
#include <stdio.h>
#ifndef LLHTTP__TEST
# include "llhttp.h"
#else
# define llhttp_t llparse_t
#endif  /* */

int llhttp_message_needs_eof(const llhttp_t* parser);
int llhttp_should_keep_alive(const llhttp_t* parser);

int llhttp__before_headers_complete(llhttp_t* parser, const char* p,
                                    const char* endp) {
  /* 在这里设置它，以便 on_headers_complete() 回调可以看到它 */
  if ((parser->flags & F_UPGRADE) &&
      (parser->flags & F_CONNECTION_UPGRADE)) {
    /* 对于响应来说，只有当它是 101 Switching Protocols 响应时，“Upgrade: foo” 和 “Connection: upgrade” 才是必需的，
     * 否则它只是信息性的，用于宣布支持。
     */
    parser->upgrade =
        (parser->type == HTTP_REQUEST || parser->status_code == 101);
  } else {
    parser->upgrade = (parser->method == HTTP_CONNECT);
  }
  return 0;
}


/* 返回值:
 * 0 - 没有消息体，`restart`，message_complete
 * 1 - CONNECT 请求，`restart`，message_complete，并暂停
 * 2 - chunk_size_start
 * 3 - body_identity
 * 4 - body_identity_eof
 * 5 - 请求的传输编码无效
 */
int llhttp__after_headers_complete(llhttp_t* parser, const char* p,
                                   const char* endp) {
  int hasBody;

  hasBody = parser->flags & F_CHUNKED || parser->content_length > 0;
  if (parser->upgrade && (parser->method == HTTP_CONNECT ||
                          (parser->flags & F_SKIPBODY) || !hasBody)) {
    /* 退出，消息的其余部分在不同的协议中 */
    return 1;
  }

  if (parser->flags & F_SKIPBODY) {
    return 0;
  } else if (parser->flags & F_CHUNKED) {
    /* 分块编码 - 忽略 Content-Length 头部，准备一个块 */
    return 2;
  } else if (parser->flags & F_TRANSFER_ENCODING) {
    if (parser->type == HTTP_REQUEST &&
        (parser->lenient_flags & LENIENT_CHUNKED_LENGTH) == 0) {
      /* 如果是 HTTP 请求，并且不是宽松的分块长度 */

      /* 根据 RFC 7230 3.3.3 规定 */
      /* 如果请求中存在 Transfer-Encoding 头字段，并且分块传输编码不是最终编码，
       * 则无法可靠确定消息体长度；服务器必须以 400 (Bad Request) 状态码做出响应，
       * 然后关闭连接。 */
      return 5;
    } else {
      /* 根据 RFC 7230 3.3.3 规定 */

      /* 如果响应中存在 Transfer-Encoding 头字段，并且分块传输编码不是最终编码，
       * 则消息体长度通过读取连接直到服务器关闭来确定。 */
      return 4;
    }
  } else {
    if (!(parser->flags & F_CONTENT_LENGTH)) {
      if (!llhttp_message_needs_eof(parser)) {
        /* 假设内容长度为 0 - 读取下一个 */
        return 0;
      } else {
        /* 读取直到 EOF */
        return 4;
      }
    } else if (parser->content_length == 0) {
      /* 给定 Content-Length 头字段但值为零: Content-Length: 0\r\n */
      return 0;
    } else {
      /* 给定 Content-Length 头字段且值非零 */
      return 3;
    }
  }
# 在消息解析完成后的处理函数
int llhttp__after_message_complete(llhttp_t* parser, const char* p,
                                   const char* endp) {
  int should_keep_alive;

  # 检查是否应该保持连接
  should_keep_alive = llhttp_should_keep_alive(parser);
  # 设置解析器的完成标志为安全完成
  parser->finish = HTTP_FINISH_SAFE;
  # 重置解析器的标志位
  parser->flags = 0;

  /* 注意：在宽松解析模式下，这将被忽略 */
  return should_keep_alive;
}

# 检查消息是否需要 EOF（结束符）
int llhttp_message_needs_eof(const llhttp_t* parser) {
  if (parser->type == HTTP_REQUEST) {
    return 0;
  }

  /* 参见 RFC 2616 第 4.4 节 */
  if (parser->status_code / 100 == 1 || /* 1xx 例如 Continue */
      parser->status_code == 204 ||     /* No Content */
      parser->status_code == 304 ||     /* Not Modified */
      (parser->flags & F_SKIPBODY)) {     /* 对 HEAD 请求的响应 */
    return 0;
  }

  /* RFC 7230 3.3.3，参见 `llhttp__after_headers_complete` */
  if ((parser->flags & F_TRANSFER_ENCODING) &&
      (parser->flags & F_CHUNKED) == 0) {
    return 1;
  }

  if (parser->flags & (F_CHUNKED | F_CONTENT_LENGTH)) {
    return 0;
  }

  return 1;
}

# 检查是否应该保持连接
int llhttp_should_keep_alive(const llhttp_t* parser) {
  if (parser->http_major > 0 && parser->http_minor > 0) {
    /* HTTP/1.1 */
    if (parser->flags & F_CONNECTION_CLOSE) {
      return 0;
    }
  } else {
    /* HTTP/1.0 或更早版本 */
    if (!(parser->flags & F_CONNECTION_KEEP_ALIVE)) {
      return 0;
    }
  }

  return !llhttp_message_needs_eof(parser);
}
```