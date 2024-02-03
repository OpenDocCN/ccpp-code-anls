# `xmrig\src\3rdparty\llhttp\llhttp.h`

```cpp
#ifndef INCLUDE_LLHTTP_H_
#define INCLUDE_LLHTTP_H_

#define LLHTTP_VERSION_MAJOR 5  // 定义LLHTTP的主版本号为5
#define LLHTTP_VERSION_MINOR 1  // 定义LLHTTP的次版本号为1
#define LLHTTP_VERSION_PATCH 0  // 定义LLHTTP的修订版本号为0

#ifndef LLHTTP_STRICT_MODE
# define LLHTTP_STRICT_MODE 0  // 如果未定义LLHTTP_STRICT_MODE，则设置为0
#endif

#ifndef INCLUDE_LLHTTP_ITSELF_H_
#define INCLUDE_LLHTTP_ITSELF_H_
#ifdef __cplusplus
extern "C" {
#endif

#include <stdint.h>

typedef struct llhttp__internal_s llhttp__internal_t;  // 定义llhttp__internal_t结构体
struct llhttp__internal_s {
  int32_t _index;
  void* _span_pos0;
  void* _span_cb0;
  int32_t error;
  const char* reason;
  const char* error_pos;
  void* data;
  void* _current;
  uint64_t content_length;
  uint8_t type;
  uint8_t method;
  uint8_t http_major;
  uint8_t http_minor;
  uint8_t header_state;
  uint8_t lenient_flags;
  uint8_t upgrade;
  uint8_t finish;
  uint16_t flags;
  uint16_t status_code;
  void* settings;
};

int llhttp__internal_init(llhttp__internal_t* s);  // 初始化llhttp__internal_t结构体
int llhttp__internal_execute(llhttp__internal_t* s, const char* p, const char* endp);  // 执行llhttp__internal_t结构体

#ifdef __cplusplus
}  /* extern "C" */
#endif
#endif  /* INCLUDE_LLHTTP_ITSELF_H_ */

#ifndef LLLLHTTP_C_HEADERS_
#define LLLLHTTP_C_HEADERS_
#ifdef __cplusplus
extern "C" {
#endif

enum llhttp_errno {  // 定义llhttp_errno枚举类型
  HPE_OK = 0,
  HPE_INTERNAL = 1,
  HPE_STRICT = 2,
  HPE_LF_EXPECTED = 3,
  HPE_UNEXPECTED_CONTENT_LENGTH = 4,
  HPE_CLOSED_CONNECTION = 5,
  HPE_INVALID_METHOD = 6,
  HPE_INVALID_URL = 7,
  HPE_INVALID_CONSTANT = 8,
  HPE_INVALID_VERSION = 9,
  HPE_INVALID_HEADER_TOKEN = 10,
  HPE_INVALID_CONTENT_LENGTH = 11,
  HPE_INVALID_CHUNK_SIZE = 12,
  HPE_INVALID_STATUS = 13,
  HPE_INVALID_EOF_STATE = 14,
  HPE_INVALID_TRANSFER_ENCODING = 15,
  HPE_CB_MESSAGE_BEGIN = 16,
  HPE_CB_HEADERS_COMPLETE = 17,
  HPE_CB_MESSAGE_COMPLETE = 18,
  HPE_CB_CHUNK_HEADER = 19,
  HPE_CB_CHUNK_COMPLETE = 20,
  HPE_PAUSED = 21,
  HPE_PAUSED_UPGRADE = 22,
  HPE_PAUSED_H2_UPGRADE = 23,
  HPE_USER = 24
};
typedef enum llhttp_errno llhttp_errno_t;  // 定义llhttp_errno_t为llhttp_errno枚举类型
# 定义枚举类型 llhttp_flags，表示 HTTP 解析器的标志位
enum llhttp_flags {
  F_CONNECTION_KEEP_ALIVE = 0x1,  # 表示连接保持活跃
  F_CONNECTION_CLOSE = 0x2,       # 表示关闭连接
  F_CONNECTION_UPGRADE = 0x4,     # 表示升级连接
  F_CHUNKED = 0x8,                 # 表示分块传输编码
  F_UPGRADE = 0x10,                # 表示升级协议
  F_CONTENT_LENGTH = 0x20,         # 表示内容长度
  F_SKIPBODY = 0x40,               # 表示跳过消息体
  F_TRAILING = 0x80,               # 表示尾部字段
  F_TRANSFER_ENCODING = 0x200      # 表示传输编码
};
typedef enum llhttp_flags llhttp_flags_t;

# 定义枚举类型 llhttp_lenient_flags，表示 HTTP 解析器的宽松标志位
enum llhttp_lenient_flags {
  LENIENT_HEADERS = 0x1,           # 表示宽松的头部解析
  LENIENT_CHUNKED_LENGTH = 0x2,    # 表示宽松的分块长度解析
  LENIENT_KEEP_ALIVE = 0x4         # 表示宽松的保持活跃解析
};
typedef enum llhttp_lenient_flags llhttp_lenient_flags_t;

# 定义枚举类型 llhttp_type，表示 HTTP 解析器的类型
enum llhttp_type {
  HTTP_BOTH = 0,                   # 表示 HTTP 请求和响应都支持
  HTTP_REQUEST = 1,                # 表示仅支持 HTTP 请求
  HTTP_RESPONSE = 2                # 表示仅支持 HTTP 响应
};
typedef enum llhttp_type llhttp_type_t;

# 定义枚举类型 llhttp_finish，表示 HTTP 解析器的结束类型
enum llhttp_finish {
  HTTP_FINISH_SAFE = 0,            # 表示安全的结束
  HTTP_FINISH_SAFE_WITH_CB = 1,    # 表示带回调的安全结束
  HTTP_FINISH_UNSAFE = 2           # 表示不安全的结束
};
typedef enum llhttp_finish llhttp_finish_t;

# 定义枚举类型 llhttp_method，表示 HTTP 方法
enum llhttp_method {
  HTTP_DELETE = 0,                 # 表示 DELETE 方法
  HTTP_GET = 1,                    # 表示 GET 方法
  HTTP_HEAD = 2,                   # 表示 HEAD 方法
  HTTP_POST = 3,                   # 表示 POST 方法
  ...                              # 其他 HTTP 方法的枚举值
};
typedef enum llhttp_method llhttp_method_t;
# 定义一个宏，用于将 HTTP 错误码映射为对应的错误信息
#define HTTP_ERRNO_MAP(XX) \
  # 错误码 0，表示正常
  XX(0, OK, OK) \
  # 错误码 1，表示内部错误
  XX(1, INTERNAL, INTERNAL) \
  # 错误码 2，表示严格模式错误
  XX(2, STRICT, STRICT) \
  # 错误码 3，表示缺少换行符错误
  XX(3, LF_EXPECTED, LF_EXPECTED) \
  # 错误码 4，表示意外的内容长度错误
  XX(4, UNEXPECTED_CONTENT_LENGTH, UNEXPECTED_CONTENT_LENGTH) \
  # 错误码 5，表示连接已关闭错误
  XX(5, CLOSED_CONNECTION, CLOSED_CONNECTION) \
  # 错误码 6，表示无效的方法错误
  XX(6, INVALID_METHOD, INVALID_METHOD) \
  # 错误码 7，表示无效的 URL 错误
  XX(7, INVALID_URL, INVALID_URL) \
  # 错误码 8，表示无效的常量错误
  XX(8, INVALID_CONSTANT, INVALID_CONSTANT) \
  # 错误码 9，表示无效的版本错误
  XX(9, INVALID_VERSION, INVALID_VERSION) \
  # 错误码 10，表示无效的头部令牌错误
  XX(10, INVALID_HEADER_TOKEN, INVALID_HEADER_TOKEN) \
  # 错误码 11，表示无效的内容长度错误
  XX(11, INVALID_CONTENT_LENGTH, INVALID_CONTENT_LENGTH) \
  # 错误码 12，表示无效的块大小错误
  XX(12, INVALID_CHUNK_SIZE, INVALID_CHUNK_SIZE) \
  # 错误码 13，表示无效的状态错误
  XX(13, INVALID_STATUS, INVALID_STATUS) \
  # 错误码 14，表示无效的 EOF 状态错误
  XX(14, INVALID_EOF_STATE, INVALID_EOF_STATE) \
  # 错误码 15，表示无效的传输编码错误
  XX(15, INVALID_TRANSFER_ENCODING, INVALID_TRANSFER_ENCODING) \
  # 错误码 16，表示消息开始回调
  XX(16, CB_MESSAGE_BEGIN, CB_MESSAGE_BEGIN) \
  # 错误码 17，表示头部完成回调
  XX(17, CB_HEADERS_COMPLETE, CB_HEADERS_COMPLETE) \
  # 错误码 18，表示消息完成回调
  XX(18, CB_MESSAGE_COMPLETE, CB_MESSAGE_COMPLETE) \
  # 错误码 19，表示块头回调
  XX(19, CB_CHUNK_HEADER, CB_CHUNK_HEADER) \
  # 错误码 20，表示块完成回调
  XX(20, CB_CHUNK_COMPLETE, CB_CHUNK_COMPLETE) \
  # 错误码 21，表示暂停
  XX(21, PAUSED, PAUSED) \
  # 错误码 22，表示升级暂停
  XX(22, PAUSED_UPGRADE, PAUSED_UPGRADE) \
  # 错误码 23，表示 H2 升级暂停
  XX(23, PAUSED_H2_UPGRADE, PAUSED_H2_UPGRADE) \
  # 错误码 24，表示用户自定义错误
  XX(24, USER, USER) \
// 定义一个宏，用于映射 HTTP 方法的枚举值和字符串表示
#define HTTP_METHOD_MAP(XX) \
  XX(0, DELETE, DELETE) \
  XX(1, GET, GET) \
  XX(2, HEAD, HEAD) \
  // ... 其他 HTTP 方法的枚举值和字符串表示

// 如果是 C++ 环境，则使用 extern "C" 包裹代码
#ifdef __cplusplus
}  /* extern "C" */
#endif

// 结束当前文件的条件编译
#endif  /* LLLLHTTP_C_HEADERS_ */

// 如果没有包含过 LLHTTP_API_H_，则包含该头文件
#ifndef INCLUDE_LLHTTP_API_H_
#define INCLUDE_LLHTTP_API_H_
// 如果是 C++ 环境，则使用 extern "C" 包裹代码
#ifdef __cplusplus
extern "C" {
#endif
// 包含标准库的头文件
#include <stddef.h>

// 根据环境定义 LLHTTP_EXPORT 宏
#if defined(__wasm__)
#define LLHTTP_EXPORT __attribute__((visibility("default")))
#else
#define LLHTTP_EXPORT
#endif

// 定义 llhttp_t 和 llhttp_settings_t 的别名
typedef llhttp__internal_t llhttp_t;
typedef struct llhttp_settings_s llhttp_settings_t;

// 定义回调函数类型 llhttp_data_cb 和 llhttp_cb
typedef int (*llhttp_data_cb)(llhttp_t*, const char *at, size_t length);
typedef int (*llhttp_cb)(llhttp_t*);
# 定义了一个结构体 llhttp_settings_s，用于存储解析 HTTP 请求/响应时的回调函数
struct llhttp_settings_s {
  # 当解析开始时的回调函数，可能返回值为 0, -1, `HPE_PAUSED`
  llhttp_cb      on_message_begin;

  # 当解析 URL 时的回调函数
  llhttp_data_cb on_url;
  # 当解析状态码时的回调函数
  llhttp_data_cb on_status;
  # 当解析头部字段时的回调函数
  llhttp_data_cb on_header_field;
  # 当解析头部值时的回调函数
  llhttp_data_cb on_header_value;

  # 当头部解析完成时的回调函数，可能返回值为 0, 1, 2, -1, `HPE_PAUSED`
  llhttp_cb      on_headers_complete;

  # 当解析请求/响应体时的回调函数
  llhttp_data_cb on_body;

  # 当消息解析完成时的回调函数，可能返回值为 0, -1, `HPE_PAUSED`
  llhttp_cb      on_message_complete;

  # 当解析分块头部时的回调函数，可能返回值为 0, -1, `HPE_PAUSED`
  llhttp_cb      on_chunk_header;
  # 当解析分块完成时的回调函数
  llhttp_cb      on_chunk_complete;

  # 当解析 URL 完成时的回调函数
  llhttp_cb      on_url_complete;
  # 当解析状态码完成时的回调函数
  llhttp_cb      on_status_complete;
  # 当解析头部字段完成时的回调函数
  llhttp_cb      on_header_field_complete;
  # 当解析头部值完成时的回调函数
  llhttp_cb      on_header_value_complete;
};

# 初始化解析器，设置解析类型和用户定义的回调函数
LLHTTP_EXPORT
void llhttp_init(llhttp_t* parser, llhttp_type_t type,
                 const llhttp_settings_t* settings);

# 如果是在 WebAssembly 环境下编译
# 分配一个解析器对象
LLHTTP_EXPORT
llhttp_t* llhttp_alloc(llhttp_type_t type);

# 释放解析器对象
LLHTTP_EXPORT
void llhttp_free(llhttp_t* parser);

# 获取解析器对象的类型
LLHTTP_EXPORT
uint8_t llhttp_get_type(llhttp_t* parser);

# 获取解析器对象的 HTTP 主版本号
LLHTTP_EXPORT
uint8_t llhttp_get_http_major(llhttp_t* parser);

# 获取解析器对象的 HTTP 次版本号
LLHTTP_EXPORT
uint8_t llhttp_get_http_minor(llhttp_t* parser);

# 获取解析器对象的请求方法
LLHTTP_EXPORT
uint8_t llhttp_get_method(llhttp_t* parser);

# 获取解析器对象的状态码
LLHTTP_EXPORT
int llhttp_get_status_code(llhttp_t* parser);

# 获取解析器对象的升级标志
LLHTTP_EXPORT
uint8_t llhttp_get_upgrade(llhttp_t* parser);
// 如果定义了 __wasm__，则结束代码块
#endif  // defined(__wasm__)

/* 重置已初始化的解析器回到起始状态，保留解析器类型、回调设置、用户数据和宽松标志 */
LLHTTP_EXPORT
void llhttp_reset(llhttp_t* parser);

/* 初始化设置对象 */
LLHTTP_EXPORT
void llhttp_settings_init(llhttp_settings_t* settings);

/* 解析完整或部分请求/响应，沿途调用用户回调函数。
   如果任何 `llhttp_data_cb` 返回的 errno 不等于 `HPE_OK` - 解析中断，并且该 errno 从 `llhttp_execute()` 返回。
   如果 `HPE_PAUSED` 作为 errno 使用，可以通过调用 `llhttp_resume()` 来恢复执行。
   在 CONNECT/Upgrade 请求/响应的特殊情况下，在完全解析请求/响应后返回 `HPE_PAUSED_UPGRADE`。
   如果用户希望继续解析，需要调用 `llhttp_resume_after_upgrade()`。
   注意：如果此函数返回非暂停类型错误，它将继续在每次连续调用中返回相同的错误，直到调用 `llhttp_init()`。
*/
LLHTTP_EXPORT
llhttp_errno_t llhttp_execute(llhttp_t* parser, const char* data, size_t len);

/* 当另一端没有更多字节要发送时（例如关闭了 TCP 连接的可读端），应调用此方法。
   没有 `Content-Length` 的请求和其他消息可能需要将所有传入的字节视为主体的一部分，直到连接的最后一个字节。
   如果请求安全终止，此方法将调用 `on_message_complete()` 回调。否则将返回错误代码。
*/
LLHTTP_EXPORT
llhttp_errno_t llhttp_finish(llhttp_t* parser);

/* 如果传入消息解析到最后一个字节，并且必须在 EOF 上调用 `llhttp_finish()` 完成 */
LLHTTP_EXPORT
int llhttp_message_needs_eof(const llhttp_t* parser);
/* 返回 `1` 如果可能会有跟在成功解析的最后一条消息之后的其他消息 */
LLHTTP_EXPORT
int llhttp_should_keep_alive(const llhttp_t* parser);

/* 使进一步调用 `llhttp_execute()` 返回 `HPE_PAUSED` 并设置适当的错误原因。

重要提示：不要从用户回调中调用此函数！用户回调必须在需要暂停时返回 `HPE_PAUSED`。
*/
LLHTTP_EXPORT
void llhttp_pause(llhttp_t* parser);

/* 可能在用户回调暂停后调用以恢复执行。有关详细信息，请参阅上面的 `llhttp_execute()`。

仅在 `llhttp_execute()` 返回 `HPE_PAUSED` 时调用此函数。
*/
LLHTTP_EXPORT
void llhttp_resume(llhttp_t* parser);

/* 可能在用户回调暂停后调用以恢复执行。有关详细信息，请参阅上面的 `llhttp_execute()`。

仅在 `llhttp_execute()` 返回 `HPE_PAUSED_UPGRADE` 时调用此函数。
*/
LLHTTP_EXPORT
void llhttp_resume_after_upgrade(llhttp_t* parser);

/* 返回最新的错误代码 */
LLHTTP_EXPORT
llhttp_errno_t llhttp_get_errno(const llhttp_t* parser);

/* 返回最新返回错误的文本解释。

注意：用户回调在返回错误时应设置错误原因。有关详细信息，请参阅 `llhttp_set_error_reason()`。
*/
LLHTTP_EXPORT
const char* llhttp_get_error_reason(const llhttp_t* parser);

/* 为返回的错误分配文本描述。必须在用户回调中在返回错误码之前调用。

注意：`HPE_USER` 错误代码在用户回调中可能会有用。
*/
LLHTTP_EXPORT
void llhttp_set_error_reason(llhttp_t* parser, const char* reason);

/* 返回返回错误之前解析的最后一个字节的指针。该指针是相对于 `llhttp_execute()` 的 `data` 参数的。

注意：此方法可能对计算解析字节数有用。
*/
LLHTTP_EXPORT
const char* llhttp_get_error_pos(const llhttp_t* parser);

/* 返回错误代码的文本名称 */
LLHTTP_EXPORT
/* 返回 HTTP 错误码对应的文本描述 */
const char* llhttp_errno_name(llhttp_errno_t err);

/* 返回 HTTP 方法对应的文本描述 */
LLHTTP_EXPORT
const char* llhttp_method_name(llhttp_method_t method);


/* 启用/禁用宽松的头部值解析（默认禁用）。
 *
 * 宽松解析禁用头部值的令牌检查，扩展了 llhttp 对高度不兼容的客户端/服务器的协议支持。
 * 当宽松解析开启时，不会因为不正确的头部值而引发 `HPE_INVALID_HEADER_TOKEN` 错误。
 *
 * **（使用需谨慎）**
 */
LLHTTP_EXPORT
void llhttp_set_lenient_headers(llhttp_t* parser, int enabled);


/* 启用/禁用对冲突的 `Transfer-Encoding` 和 `Content-Length` 头部的宽松处理（默认禁用）。
 *
 * 通常情况下，当 `Transfer-Encoding` 与 `Content-Length` 同时存在时，`llhttp` 会报错。
 * 这个错误对于防止 HTTP 请求走私非常重要，但在涉及传统服务器的少数情况下可能不太理想。
 *
 * **（使用需谨慎）**
 */
LLHTTP_EXPORT
void llhttp_set_lenient_chunked_length(llhttp_t* parser, int enabled);


/* 启用/禁用对 `Connection: close` 和 HTTP/1.0 请求响应的宽松处理。
 *
 * 通常情况下，`llhttp` 会在严格模式下报错，或在宽松模式下丢弃带有 `Connection: close` 和 `Content-Length` 的请求/响应。
 * 这对于防止缓存污染攻击非常重要，但可能与过时和不安全的客户端产生不良交互。
 * 使用此标志，额外的请求/响应将被正常解析。
 *
 * **（使用需谨慎）**
 */
void llhttp_set_lenient_keep_alive(llhttp_t* parser, int enabled);

#ifdef __cplusplus
}  /* extern "C" */
#endif
#endif  /* INCLUDE_LLHTTP_API_H_ */

#endif  /* INCLUDE_LLHTTP_H_ */
```