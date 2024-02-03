# `xmrig\src\3rdparty\llhttp\api.h`

```cpp
#ifndef INCLUDE_LLHTTP_API_H_
#define INCLUDE_LLHTTP_API_H_
#ifdef __cplusplus
extern "C" {
#endif
#include <stddef.h>

#if defined(__wasm__)
#define LLHTTP_EXPORT __attribute__((visibility("default")))
#else
#define LLHTTP_EXPORT
#endif

typedef llhttp__internal_t llhttp_t;  // 定义 llhttp_t 类型为 llhttp__internal_t 类型
typedef struct llhttp_settings_s llhttp_settings_t;  // 定义 llhttp_settings_t 结构体

typedef int (*llhttp_data_cb)(llhttp_t*, const char *at, size_t length);  // 定义 llhttp_data_cb 函数指针类型
typedef int (*llhttp_cb)(llhttp_t*);  // 定义 llhttp_cb 函数指针类型

struct llhttp_settings_s {
  /* Possible return values 0, -1, `HPE_PAUSED` */
  llhttp_cb      on_message_begin;  // 消息开始时的回调函数

  llhttp_data_cb on_url;  // URL 数据的回调函数
  llhttp_data_cb on_status;  // 状态数据的回调函数
  llhttp_data_cb on_header_field;  // 头部字段数据的回调函数
  llhttp_data_cb on_header_value;  // 头部值数据的回调函数

  /* Possible return values:
   * 0  - Proceed normally
   * 1  - Assume that request/response has no body, and proceed to parsing the
   *      next message
   * 2  - Assume absence of body (as above) and make `llhttp_execute()` return
   *      `HPE_PAUSED_UPGRADE`
   * -1 - Error
   * `HPE_PAUSED`
   */
  llhttp_cb      on_headers_complete;  // 头部解析完成时的回调函数

  llhttp_data_cb on_body;  // 消息体数据的回调函数

  /* Possible return values 0, -1, `HPE_PAUSED` */
  llhttp_cb      on_message_complete;  // 消息解析完成时的回调函数

  /* When on_chunk_header is called, the current chunk length is stored
   * in parser->content_length.
   * Possible return values 0, -1, `HPE_PAUSED`
   */
  llhttp_cb      on_chunk_header;  // 分块头部解析完成时的回调函数
  llhttp_cb      on_chunk_complete;  // 分块解析完成时的回调函数

  llhttp_cb      on_url_complete;  // URL 解析完成时的回调函数
  llhttp_cb      on_status_complete;  // 状态解析完成时的回调函数
  llhttp_cb      on_header_field_complete;  // 头部字段解析完成时的回调函数
  llhttp_cb      on_header_value_complete;  // 头部值解析完成时的回调函数
};

/* Initialize the parser with specific type and user settings.
 *
 * NOTE: lifetime of `settings` has to be at least the same as the lifetime of
 * the `parser` here. In practice, `settings` has to be either a static
 * variable or be allocated with `malloc`, `new`, etc.
 */
LLHTTP_EXPORT
void llhttp_init(llhttp_t* parser, llhttp_type_t type,
                 const llhttp_settings_t* settings);  // 初始化解析器，设置类型和用户设置

#if defined(__wasm__)

LLHTTP_EXPORT
// 分配一个新的 llhttp_t 对象，根据给定的类型
llhttp_t* llhttp_alloc(llhttp_type_t type);

// 释放一个已经初始化的 llhttp_t 对象
void llhttp_free(llhttp_t* parser);

// 获取解析器的类型
uint8_t llhttp_get_type(llhttp_t* parser);

// 获取 HTTP 协议的主版本号
uint8_t llhttp_get_http_major(llhttp_t* parser);

// 获取 HTTP 协议的次版本号
uint8_t llhttp_get_http_minor(llhttp_t* parser);

// 获取请求方法
uint8_t llhttp_get_method(llhttp_t* parser);

// 获取状态码
int llhttp_get_status_code(llhttp_t* parser);

// 获取是否为升级请求
uint8_t llhttp_get_upgrade(llhttp_t* parser);

// 重置已经初始化的解析器到初始状态，保留解析器类型、回调设置、用户数据和宽松标志
void llhttp_reset(llhttp_t* parser);

// 初始化设置对象
void llhttp_settings_init(llhttp_settings_t* settings);

/* 解析完整或部分请求/响应，沿途调用用户回调。
   如果任何 `llhttp_data_cb` 返回的 errno 不等于 `HPE_OK` - 解析中断，并且该 errno 从 `llhttp_execute()` 返回。
   如果 `HPE_PAUSED` 作为 errno 使用，可以通过调用 `llhttp_resume()` 来恢复执行。
   在 CONNECT/Upgrade 请求/响应的特殊情况下，完全解析请求/响应后返回 `HPE_PAUSED_UPGRADE`。
   如果用户希望继续解析，需要调用 `llhttp_resume_after_upgrade()`。
   注意：如果此函数返回非暂停类型错误，它将继续在每次后续调用中返回相同的错误，直到调用 `llhttp_init()`。
*/
llhttp_errno_t llhttp_execute(llhttp_t* parser, const char* data, size_t len);
# 当另一端没有更多字节要发送时，应调用此方法（例如，关闭 TCP 连接的可读一侧）。
# 没有 `Content-Length` 的请求和其他消息可能需要将所有传入的字节视为消息体的一部分，直到连接的最后一个字节。
# 如果请求安全终止，此方法将调用 `on_message_complete()` 回调。否则将返回错误代码。
LLHTTP_EXPORT
llhttp_errno_t llhttp_finish(llhttp_t* parser);

# 如果传入消息被解析到最后一个字节，并且必须在 EOF 上调用 `llhttp_finish()` 来完成，则返回 `1`。
LLHTTP_EXPORT
int llhttp_message_needs_eof(const llhttp_t* parser);

# 如果可能有任何其他消息跟随成功解析的最后一个消息，则返回 `1`。
LLHTTP_EXPORT
int llhttp_should_keep_alive(const llhttp_t* parser);

# 使进一步调用 `llhttp_execute()` 返回 `HPE_PAUSED` 并设置适当的错误原因。
# 重要提示：不要从用户回调中调用此方法！用户回调必须在需要暂停时返回 `HPE_PAUSED`。
LLHTTP_EXPORT
void llhttp_pause(llhttp_t* parser);

# 可能在用户回调中暂停后恢复执行。
# 详细信息请参见上面的 `llhttp_execute()`。
# 仅当 `llhttp_execute()` 返回 `HPE_PAUSED` 时才调用此方法。
LLHTTP_EXPORT
void llhttp_resume(llhttp_t* parser);

# 可能在用户回调中暂停后恢复执行。
# 详细信息请参见上面的 `llhttp_execute()`。
# 仅当 `llhttp_execute()` 返回 `HPE_PAUSED_UPGRADE` 时才调用此方法。
LLHTTP_EXPORT
void llhttp_resume_after_upgrade(llhttp_t* parser);

# 返回最新的错误代码
LLHTTP_EXPORT
llhttp_errno_t llhttp_get_errno(const llhttp_t* parser);

# 返回最新返回错误的文本解释。
# 注意：用户回调在返回错误时应设置错误原因。有关详细信息，请参见 `llhttp_set_error_reason()`。
# 获取解析错误的原因
LLHTTP_EXPORT
const char* llhttp_get_error_reason(const llhttp_t* parser);

# 为返回的错误分配口头描述。必须在用户回调中调用，就在返回错误码之前。
#
# 注意：在用户回调中，`HPE_USER` 错误码可能会很有用。
LLHTTP_EXPORT
void llhttp_set_error_reason(llhttp_t* parser, const char* reason);

# 返回返回的错误之前解析的最后一个字节的指针。该指针是相对于 `llhttp_execute()` 的 `data` 参数的。
#
# 注意：这个方法可能对计算解析字节数很有用。
LLHTTP_EXPORT
const char* llhttp_get_error_pos(const llhttp_t* parser);

# 返回错误码的文本名称
LLHTTP_EXPORT
const char* llhttp_errno_name(llhttp_errno_t err);

# 返回 HTTP 方法的文本名称
LLHTTP_EXPORT
const char* llhttp_method_name(llhttp_method_t method);

# 启用/禁用宽松的头部值解析（默认情况下禁用）。
#
# 宽松解析禁用头部值令牌检查，扩展了 llhttp 对高度不兼容的客户端/服务器的协议支持。当宽松解析为“开启”时，不会因为不正确的头部值而引发 `HPE_INVALID_HEADER_TOKEN` 错误。
#
# **（自行承担风险）**
LLHTTP_EXPORT
void llhttp_set_lenient_headers(llhttp_t* parser, int enabled);

# 启用/禁用对冲突的 `Transfer-Encoding` 和 `Content-Length` 头部的宽松处理（默认情况下禁用）。
#
# 通常情况下，当 `Transfer-Encoding` 与 `Content-Length` 同时存在时，`llhttp` 会引发错误。这个错误对于防止 HTTP 请求走私很重要，但在涉及传统服务器的少数情况下可能不太理想。
#
# **（自行承担风险）**
LLHTTP_EXPORT
void llhttp_set_lenient_chunked_length(llhttp_t* parser, int enabled);
/* 启用/禁用对`Connection: close`和HTTP/1.0请求响应的宽松处理。
 *
 * 通常情况下，`llhttp`会在严格模式下出错，或在宽松模式下丢弃带有`Connection: close`和`Content-Length`的HTTP请求/响应。这对于防止缓存污染攻击非常重要，但可能与过时和不安全的客户端产生不良交互。使用此标志，额外的请求/响应将被正常解析。
 *
 * **（使用需谨慎）**
 */
void llhttp_set_lenient_keep_alive(llhttp_t* parser, int enabled);

#ifdef __cplusplus
}  /* extern "C" */
#endif
#endif  /* INCLUDE_LLHTTP_API_H_ */
```