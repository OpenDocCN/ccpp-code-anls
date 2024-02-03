# `nmap\ncat\http.h`

```cpp
/* $Id$ */

#ifndef _HTTP_H
#define _HTTP_H

#include "ncat_config.h"
#include "util.h"

#include <stdio.h>
#include <stdlib.h>

/* This is an abstraction over a socket (really a struct fdinfo) that provides
   rudimentary buffering. It is useful for the line-oriented parts of HTTP. */
struct socket_buffer {
    struct fdinfo fdn;  // 定义一个结构体 fdinfo 类型的成员变量 fdn
    char buffer[BUFSIZ];  // 定义一个大小为 BUFSIZ 的字符数组 buffer
    char *p;  // 定义一个字符指针 p
    char *end;  // 定义一个字符指针 end
};

void socket_buffer_init(struct socket_buffer *buf, int sd);  // 初始化 socket_buffer 结构体

int socket_buffer_read(struct socket_buffer *buf, char *out, size_t size);  // 从 socket_buffer 中读取数据到 out 中，最多读取 size 个字节

char *socket_buffer_readline(struct socket_buffer *buf, size_t *n, size_t maxlen);  // 从 socket_buffer 中读取一行数据，存储在 n 中，最多读取 maxlen 个字节

int socket_buffer_readcount(struct socket_buffer *buf, char *out, size_t size);  // 从 socket_buffer 中读取指定数量的数据到 out 中，最多读取 size 个字节

char *socket_buffer_remainder(struct socket_buffer *buf, size_t *len);  // 获取 socket_buffer 中剩余的数据，存储在 len 中

/* A broken-down URI as defined in RFC 3986, except that the query and fragment
   parts are included in the path. */
struct uri {
    char *scheme;  // URI 的协议部分
    char *host;  // URI 的主机部分
    int port;  // URI 的端口部分
    char *path;  // URI 的路径部分
};

void uri_init(struct uri *uri);  // 初始化 URI 结构体

void uri_free(struct uri *uri);  // 释放 URI 结构体的内存

struct uri *uri_parse(struct uri *uri, const char *uri_s);  // 解析 URI 字符串为 URI 结构体

struct uri *uri_parse_authority(struct uri *uri, const char *authority);  // 解析 URI 的权限部分

enum http_version {
    HTTP_09,  // HTTP 版本 0.9
    HTTP_10,  // HTTP 版本 1.0
    HTTP_11,  // HTTP 版本 1.1
    HTTP_UNKNOWN,  // 未知的 HTTP 版本
};

struct http_header {
    char *name;  // HTTP 头部的名称
    char *value;  // HTTP 头部的值
    struct http_header *next;  // 下一个 HTTP 头部
};

struct http_request {
    char *method;  // HTTP 请求的方法
    struct uri uri;  // HTTP 请求的 URI
    enum http_version version;  // HTTP 请求的版本
    struct http_header *header;  // HTTP 请求的头部
    int content_length_set;  // 内容长度是否已设置
    unsigned long content_length;  // 内容长度
    unsigned long bytes_transferred;  // 已传输的字节数
};

struct http_response {
    enum http_version version;  // HTTP 响应的版本
    int code;  // HTTP 响应的状态码
    char *phrase;  // HTTP 响应的短语
    struct http_header *header;  // HTTP 响应的头部
    int content_length_set;  // 内容长度是否已设置
    unsigned long content_length;  // 内容长度
    unsigned long bytes_transferred;  // 已传输的字节数
};

void http_header_free(struct http_header *header);  // 释放 HTTP 头部的内存
char *http_header_get(const struct http_header *header, const char *name);  // 获取指定名称的 HTTP 头部的值
// 返回下一个 HTTP 头部
const struct http_header *http_header_next(const struct http_header *header, const struct http_header *p, const char *name);

// 获取指定名称的 HTTP 头部的第一个值
char *http_header_get_first(const struct http_header *header, const char *name);

// 设置指定名称的 HTTP 头部的值
struct http_header *http_header_set(struct http_header *header, const char *name, const char *value);

// 移除指定名称的 HTTP 头部
struct http_header *http_header_remove(struct http_header *header, const char *name);

// 移除所有与连接相关的 HTTP 头部
int http_header_remove_hop_by_hop(struct http_header **header);

// 将 HTTP 头部转换为字符串
char *http_header_to_string(const struct http_header *header, size_t *n);

// 初始化 HTTP 请求对象
void http_request_init(struct http_request *request);

// 释放 HTTP 请求对象
void http_request_free(struct http_request *request);

// 将 HTTP 请求对象转换为字符串
char *http_request_to_string(const struct http_request *request, size_t *n);

// 初始化 HTTP 响应对象
void http_response_init(struct http_response *response);

// 释放 HTTP 响应对象
void http_response_free(struct http_response *response);

// 将 HTTP 响应对象转换为字符串
char *http_response_to_string(const struct http_response *response, size_t *n);

// 从套接字缓冲区读取 HTTP 头部
int http_read_header(struct socket_buffer *buf, char **result);

// 解析 HTTP 头部
int http_parse_header(struct http_header **result, const char *header);

// 解析 HTTP 请求的头部
int http_request_parse_header(struct http_request *request, const char *header);

// 解析 HTTP 响应的头部
int http_response_parse_header(struct http_response *response, const char *header);

// 从套接字缓冲区读取 HTTP 请求行
int http_read_request_line(struct socket_buffer *buf, char **line);

// 解析 HTTP 请求行
int http_parse_request_line(const char *line, struct http_request *request);

// 从套接字缓冲区读取 HTTP 状态行
int http_read_status_line(struct socket_buffer *buf, char **line);

// 解析 HTTP 状态行
int http_parse_status_line(const char *line, struct http_response *response);

// 解析 HTTP 状态行的状态码
int http_parse_status_line_code(const char *line);

// HTTP 认证方案枚举
enum http_auth_scheme { AUTH_UNKNOWN, AUTH_BASIC, AUTH_DIGEST };

// HTTP 摘要算法枚举
enum http_digest_algorithm { ALGORITHM_MD5, ALGORITHM_UNKNOWN };

// HTTP 摘要 QOP 枚举
enum http_digest_qop { QOP_NONE = 0, QOP_AUTH = 1 << 0, QOP_AUTH_INT = 1 << 1 };

// HTTP 挑战结构体
struct http_challenge {
    enum http_auth_scheme scheme; // 认证方案
    char *realm; // 领域
    # 定义一个结构体，包含nonce、opaque、算法和qop等字段
    struct {
        # 用于存储服务器生成的随机数
        char *nonce;
        # 用于存储服务器生成的不透明字符串
        char *opaque;
        # 用于存储HTTP摘要认证算法的类型
        enum http_digest_algorithm algorithm;
        # 用于存储支持的qop值的位掩码（例如"auth"、"auth-int"等）
        unsigned char qop;
    } digest;
};

// 定义一个结构体，用于存储 HTTP 认证的相关信息
struct http_credentials {
    enum http_auth_scheme scheme;  // 存储 HTTP 认证方案的枚举值
    union {
        char *basic;  // 存储基本认证的信息
        struct {
            char *username;  // 存储用户名
            char *realm;  // 存储领域
            char *nonce;  // 存储随机数
            char *uri;  // 存储 URI
            char *response;  // 存储响应
            enum http_digest_algorithm algorithm;  // 存储摘要认证的算法
            enum http_digest_qop qop;  // 存储摘要认证的质量保证
            char *nc;  // 存储计数器
            char *cnonce;  // 存储客户端随机数
        } digest;  // 存储摘要认证的信息
    } u;  // 联合体，用于存储不同类型的认证信息
};

// 初始化 HTTP 挑战结构体
void http_challenge_init(struct http_challenge *challenge);
// 释放 HTTP 挑战结构体
void http_challenge_free(struct http_challenge *challenge);
// 获取代理服务器挑战的 HTTP 头部信息
struct http_challenge *http_header_get_proxy_challenge(const struct http_header *header, struct http_challenge *challenge);

// 初始化基本认证的 HTTP 凭据结构体
void http_credentials_init_basic(struct http_credentials *credentials);
// 初始化摘要认证的 HTTP 凭据结构体
void http_credentials_init_digest(struct http_credentials *credentials);
// 释放 HTTP 凭据结构体
void http_credentials_free(struct http_credentials *credentials);
// 获取代理服务器的 HTTP 凭据信息
struct http_credentials *http_header_get_proxy_credentials(const struct http_header *header, struct http_credentials *credentials);

#if HAVE_HTTP_DIGEST
// 初始化用于生成随机数的服务器密钥
int http_digest_init_secret(void);
// 获取随机数的时间信息
int http_digest_nonce_time(const char *nonce, struct timeval *tv);
// 返回一个 Proxy-Authenticate 头部信息
char *http_digest_proxy_authenticate(const char *realm, int stale);
// 返回一个回答给定挑战的 Proxy-Authorization 头部信息
char *http_digest_proxy_authorization(const struct http_challenge *challenge,
    const char *username, const char *password,
    const char *method, const char *uri);
// 检查凭据信息是否正确
int http_digest_check_credentials(const char *username, const char *realm,
    const char *password, const char *method,
    const struct http_credentials *credentials);
#endif

#endif
```