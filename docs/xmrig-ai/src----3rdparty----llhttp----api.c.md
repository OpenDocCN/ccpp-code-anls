# `xmrig\src\3rdparty\llhttp\api.c`

```cpp
#include <stdlib.h>  // 包含标准库头文件
#include <stdio.h>   // 包含标准输入输出头文件
#include <string.h>  // 包含字符串处理头文件

#include "llhttp.h"  // 包含自定义的 llhttp 头文件

#define CALLBACK_MAYBE(PARSER, NAME, ...)                                     \  // 定义宏 CALLBACK_MAYBE，用于执行回调函数
  do {                                                                        \
    const llhttp_settings_t* settings;                                        \
    settings = (const llhttp_settings_t*) (PARSER)->settings;                 \
    if (settings == NULL || settings->NAME == NULL) {                         \  // 如果设置为空或者指定的回调函数为空，则跳过
      err = 0;                                                                \
      break;                                                                  \
    }                                                                         \
    err = settings->NAME(__VA_ARGS__);                                        \  // 执行指定的回调函数
  } while (0)

void llhttp_init(llhttp_t* parser, llhttp_type_t type,                        \  // 初始化 llhttp 解析器
                 const llhttp_settings_t* settings) {
  llhttp__internal_init(parser);                                              \  // 调用内部初始化函数

  parser->type = type;                                                        \  // 设置解析器类型
  parser->settings = (void*) settings;                                        \  // 设置解析器的设置
}


#if defined(__wasm__)

extern int wasm_on_message_begin(llhttp_t * p);                                \  // 声明 wasm_on_message_begin 函数
extern int wasm_on_url(llhttp_t* p, const char* at, size_t length);            \  // 声明 wasm_on_url 函数
extern int wasm_on_status(llhttp_t* p, const char* at, size_t length);         \  // 声明 wasm_on_status 函数
extern int wasm_on_header_field(llhttp_t* p, const char* at, size_t length);   \  // 声明 wasm_on_header_field 函数
extern int wasm_on_header_value(llhttp_t* p, const char* at, size_t length);   \  // 声明 wasm_on_header_value 函数
extern int wasm_on_headers_complete(llhttp_t * p);                            \  // 声明 wasm_on_headers_complete 函数
extern int wasm_on_body(llhttp_t* p, const char* at, size_t length);            \  // 声明 wasm_on_body 函数
extern int wasm_on_message_complete(llhttp_t * p);                            \  // 声明 wasm_on_message_complete 函数

const llhttp_settings_t wasm_settings = {                                     \  // 定义 wasm_settings 结构体
  wasm_on_message_begin,                                                      \  // 设置消息开始回调函数
  wasm_on_url,                                                                \  // 设置 URL 回调函数
  wasm_on_status,                                                             \  // 设置状态回调函数
  wasm_on_header_field,                                                       \  // 设置头字段回调函数
  wasm_on_header_value,                                                       \  // 设置头值回调函数
  wasm_on_headers_complete,                                                   \  // 设置头部完成回调函数
  wasm_on_body,                                                               \  // 设置消息体回调函数
  wasm_on_message_complete,                                                   \  // 设置消息完成回调函数
  NULL,                                                                       \  // 设置升级回调函数为 NULL
  NULL,                                                                       \  // 设置解析错误回调函数为 NULL
};

llhttp_t* llhttp_alloc(llhttp_type_t type) {                                  \  // 分配 llhttp 解析器
  llhttp_t* parser = malloc(sizeof(llhttp_t));                                \  // 分配内存
  llhttp_init(parser, type, &wasm_settings);                                  \  // 初始化解析器
  return parser;                                                              \  // 返回解析器
}
// 释放解析器内存
void llhttp_free(llhttp_t* parser) {
  free(parser);
}

// 获取解析器的类型
uint8_t llhttp_get_type(llhttp_t* parser) {
  return parser->type;
}

// 获取解析器的 HTTP 主版本号
uint8_t llhttp_get_http_major(llhttp_t* parser) {
  return parser->http_major;
}

// 获取解析器的 HTTP 次版本号
uint8_t llhttp_get_http_minor(llhttp_t* parser) {
  return parser->http_minor;
}

// 获取解析器的方法
uint8_t llhttp_get_method(llhttp_t* parser) {
  return parser->method;
}

// 获取解析器的状态码
int llhttp_get_status_code(llhttp_t* parser) {
  return parser->status_code;
}

// 获取解析器的升级标志
uint8_t llhttp_get_upgrade(llhttp_t* parser) {
  return parser->upgrade;
}

// 重置解析器
void llhttp_reset(llhttp_t* parser) {
  llhttp_type_t type = parser->type;
  const llhttp_settings_t* settings = parser->settings;
  void* data = parser->data;
  uint8_t lenient_flags = parser->lenient_flags;

  llhttp__internal_init(parser);

  parser->type = type;
  parser->settings = (void*) settings;
  parser->data = data;
  parser->lenient_flags = lenient_flags;
}

// 执行解析器
llhttp_errno_t llhttp_execute(llhttp_t* parser, const char* data, size_t len) {
  return llhttp__internal_execute(parser, data, data + len);
}

// 初始化解析器设置
void llhttp_settings_init(llhttp_settings_t* settings) {
  memset(settings, 0, sizeof(*settings));
}

// 完成解析
llhttp_errno_t llhttp_finish(llhttp_t* parser) {
  int err;

  // 如果解析器处于错误状态，则不执行任何操作
  if (parser->error != 0) {
    return 0;
  }

  switch (parser->finish) {
    case HTTP_FINISH_SAFE_WITH_CB:
      CALLBACK_MAYBE(parser, on_message_complete, parser);
      if (err != HPE_OK) return err;

    /* FALLTHROUGH */
    case HTTP_FINISH_SAFE:
      return HPE_OK;
    case HTTP_FINISH_UNSAFE:
      parser->reason = "Invalid EOF state";
      return HPE_INVALID_EOF_STATE;
    default:
      abort();
  }
}

// 暂停解析器
void llhttp_pause(llhttp_t* parser) {
  if (parser->error != HPE_OK) {
    return;
  }

  parser->error = HPE_PAUSED;
  parser->reason = "Paused";
}
# 恢复解析器的状态，如果解析器的错误状态不是 HPE_PAUSED，则直接返回
void llhttp_resume(llhttp_t* parser) {
  if (parser->error != HPE_PAUSED) {
    return;
  }
  # 将解析器的错误状态重置为0
  parser->error = 0;
}


# 恢复升级后的解析器的状态，如果解析器的错误状态不是 HPE_PAUSED_UPGRADE，则直接返回
void llhttp_resume_after_upgrade(llhttp_t* parser) {
  if (parser->error != HPE_PAUSED_UPGRADE) {
    return;
  }
  # 将解析器的错误状态重置为0
  parser->error = 0;
}


# 获取解析器的错误状态
llhttp_errno_t llhttp_get_errno(const llhttp_t* parser) {
  return parser->error;
}


# 获取解析器的错误原因
const char* llhttp_get_error_reason(const llhttp_t* parser) {
  return parser->reason;
}


# 设置解析器的错误原因
void llhttp_set_error_reason(llhttp_t* parser, const char* reason) {
  parser->reason = reason;
}


# 获取解析器的错误位置
const char* llhttp_get_error_pos(const llhttp_t* parser) {
  return parser->error_pos;
}


# 根据错误状态获取错误名称
const char* llhttp_errno_name(llhttp_errno_t err) {
#define HTTP_ERRNO_GEN(CODE, NAME, _) case HPE_##NAME: return "HPE_" #NAME;
  switch (err) {
    HTTP_ERRNO_MAP(HTTP_ERRNO_GEN)
    default: abort();
  }
#undef HTTP_ERRNO_GEN
}


# 根据方法获取方法名称
const char* llhttp_method_name(llhttp_method_t method) {
#define HTTP_METHOD_GEN(NUM, NAME, STRING) case HTTP_##NAME: return #STRING;
  switch (method) {
    HTTP_METHOD_MAP(HTTP_METHOD_GEN)
    default: abort();
  }
#undef HTTP_METHOD_GEN
}


# 设置解析器是否宽松处理头部
void llhttp_set_lenient_headers(llhttp_t* parser, int enabled) {
  if (enabled) {
    parser->lenient_flags |= LENIENT_HEADERS;
  } else {
    parser->lenient_flags &= ~LENIENT_HEADERS;
  }
}


# 设置解析器是否宽松处理分块长度
void llhttp_set_lenient_chunked_length(llhttp_t* parser, int enabled) {
  if (enabled) {
    parser->lenient_flags |= LENIENT_CHUNKED_LENGTH;
  } else {
    parser->lenient_flags &= ~LENIENT_CHUNKED_LENGTH;
  }
}


# 设置解析器是否宽松处理保持连接
void llhttp_set_lenient_keep_alive(llhttp_t* parser, int enabled) {
  if (enabled) {
    parser->lenient_flags |= LENIENT_KEEP_ALIVE;
  } else {
    parser->lenient_flags &= ~LENIENT_KEEP_ALIVE;
  }
}

# 回调函数

# 开始消息时的回调函数
int llhttp__on_message_begin(llhttp_t* s, const char* p, const char* endp) {
  int err;
  CALLBACK_MAYBE(s, on_message_begin, s);
  return err;
}
# 处理 URL 的回调函数
int llhttp__on_url(llhttp_t* s, const char* p, const char* endp) {
  int err;
  # 调用可能存在的 on_url 回调函数
  CALLBACK_MAYBE(s, on_url, s, p, endp - p);
  return err;
}

# 处理 URL 完成的回调函数
int llhttp__on_url_complete(llhttp_t* s, const char* p, const char* endp) {
  int err;
  # 调用可能存在的 on_url_complete 回调函数
  CALLBACK_MAYBE(s, on_url_complete, s);
  return err;
}

# 处理状态码的回调函数
int llhttp__on_status(llhttp_t* s, const char* p, const char* endp) {
  int err;
  # 调用可能存在的 on_status 回调函数
  CALLBACK_MAYBE(s, on_status, s, p, endp - p);
  return err;
}

# 处理状态码完成的回调函数
int llhttp__on_status_complete(llhttp_t* s, const char* p, const char* endp) {
  int err;
  # 调用可能存在的 on_status_complete 回调函数
  CALLBACK_MAYBE(s, on_status_complete, s);
  return err;
}

# 处理头部字段的回调函数
int llhttp__on_header_field(llhttp_t* s, const char* p, const char* endp) {
  int err;
  # 调用可能存在的 on_header_field 回调函数
  CALLBACK_MAYBE(s, on_header_field, s, p, endp - p);
  return err;
}

# 处理头部字段完成的回调函数
int llhttp__on_header_field_complete(llhttp_t* s, const char* p, const char* endp) {
  int err;
  # 调用可能存在的 on_header_field_complete 回调函数
  CALLBACK_MAYBE(s, on_header_field_complete, s);
  return err;
}

# 处理头部值的回调函数
int llhttp__on_header_value(llhttp_t* s, const char* p, const char* endp) {
  int err;
  # 调用可能存在的 on_header_value 回调函数
  CALLBACK_MAYBE(s, on_header_value, s, p, endp - p);
  return err;
}

# 处理头部值完成的回调函数
int llhttp__on_header_value_complete(llhttp_t* s, const char* p, const char* endp) {
  int err;
  # 调用可能存在的 on_header_value_complete 回调函数
  CALLBACK_MAYBE(s, on_header_value_complete, s);
  return err;
}

# 处理头部完成的回调函数
int llhttp__on_headers_complete(llhttp_t* s, const char* p, const char* endp) {
  int err;
  # 调用可能存在的 on_headers_complete 回调函数
  CALLBACK_MAYBE(s, on_headers_complete, s);
  return err;
}

# 处理消息完成的回调函数
int llhttp__on_message_complete(llhttp_t* s, const char* p, const char* endp) {
  int err;
  # 调用可能存在的 on_message_complete 回调函数
  CALLBACK_MAYBE(s, on_message_complete, s);
  return err;
}

# 处理消息体的回调函数
int llhttp__on_body(llhttp_t* s, const char* p, const char* endp) {
  int err;
  # 调用可能存在的 on_body 回调函数
  CALLBACK_MAYBE(s, on_body, s, p, endp - p);
  return err;
}

# 处理分块头部的回调函数
int llhttp__on_chunk_header(llhttp_t* s, const char* p, const char* endp) {
  int err;
  # 调用可能存在的 on_chunk_header 回调函数
  CALLBACK_MAYBE(s, on_chunk_header, s);
  return err;
}

# 处理分块完成的回调函数
int llhttp__on_chunk_complete(llhttp_t* s, const char* p, const char* endp) {
  int err;
  # 调用可能存在的 on_chunk_complete 回调函数
  CALLBACK_MAYBE(s, on_chunk_complete, s);
  return err;
}

# 私有部分
// 打印调试信息，包括解析器状态、当前字符、标志位等
void llhttp__debug(llhttp_t* s, const char* p, const char* endp,
                   const char* msg) {
  // 如果当前字符指针已经到达结束位置
  if (p == endp) {
    // 打印解析器状态、类型、标志位等信息
    fprintf(stderr, "p=%p type=%d flags=%02x next=null debug=%s\n", s, s->type,
            s->flags, msg);
  } else {
    // 打印解析器状态、类型、标志位以及当前字符信息
    fprintf(stderr, "p=%p type=%d flags=%02x next=%02x   debug=%s\n", s,
            s->type, s->flags, *p, msg);
  }
}
```