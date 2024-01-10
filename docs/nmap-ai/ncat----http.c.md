# `nmap\ncat\http.c`

```
/* $Id$ */

#include <string.h>

#include "base64.h"
#include "ncat.h"
#include "http.h"

/* 限制内存数据结构的大小，以避免某些拒绝服务攻击（试图消耗所有可用内存） */
static const int MAX_REQUEST_LINE_LENGTH = 1024;
static const int MAX_STATUS_LINE_LENGTH = 1024;
static const int MAX_HEADER_LENGTH = 1024 * 10;

void socket_buffer_init(struct socket_buffer *buf, int sd)
{
    buf->fdn.fd = sd;
#ifdef HAVE_OPENSSL
    buf->fdn.ssl = NULL;
#endif
    buf->p = buf->buffer;
    buf->end = buf->p;
}

/* 从有状态的套接字缓冲区中读取数据。如果缓冲区中有任何数据，则返回该数据，否则使用recv读取数据。返回值与recv相同。 */
int socket_buffer_read(struct socket_buffer *buf, char *out, size_t size)
{
    int i;

    /* 如果需要，重新填充缓冲区 */
    if (buf->p >= buf->end) {
        buf->p = buf->buffer;
        do {
            errno = 0;
            i = fdinfo_recv(&buf->fdn, buf->buffer, sizeof(buf->buffer));
        } while (i == -1 && errno == EINTR);
        if (i <= 0)
            return i;
        buf->end = buf->buffer + i;
    }
    i = buf->end - buf->p;
    if (i > size)
        i = size;
    memcpy(out, buf->p, i);
    buf->p += i;

    return i;
}

/* 通过有状态的套接字缓冲区读取一行。该行，包括其'\n'，以动态分配的缓冲区返回。行的长度在*n中返回。如果行的长度超过maxlen，则返回NULL，并且*n大于或等于maxlen。发生错误时，返回NULL，并且*n小于maxlen。如果返回值不为NULL，则返回的缓冲区始终以空字符结尾。 */
char *socket_buffer_readline(struct socket_buffer *buf, size_t *n, size_t maxlen)
{
    char *line;
    char *newline;
    size_t count;

    line = NULL;
    *n = 0;
    /* 如果需要，重新填充缓冲区。 */
    if (buf->p >= buf->end) {
        int i;

        buf->p = buf->buffer;
        do {
            errno = 0;
            i = fdinfo_recv(&buf->fdn, buf->buffer, sizeof(buf->buffer));
        } while (i == -1 && errno == EINTR);
        if (i <= 0) {
            free(line);
            return NULL;
        }
        buf->end = buf->buffer + i;
    }

    newline = (char *) memchr(buf->p, '\n', buf->end - buf->p);
    if (newline == NULL)
        count = buf->end - buf->p;
    else
        count = newline + 1 - buf->p;

    if (*n + count >= maxlen) {
        /* 行超过了最大长度。 */
        free(line);
        *n += count;
        return NULL;
    }

    line = (char *) safe_realloc(line, *n + count + 1);
    memcpy(line + *n, buf->p, count);
    *n += count;
    buf->p += count;
} while (newline == NULL);

line[*n] = '\0';

return line;
}

/* 这类似于 socket_buffer_read，不同之处在于它会阻塞直到能够读取全部 size 字节。如果可用的字节数少于 size 字节，它会读取它们并返回 -1。 */
int socket_buffer_readcount(struct socket_buffer *buf, char *out, size_t size)
{
    size_t n = 0;
    int i;

    while (n < size) {
        /* 如果需要，重新填充缓冲区。 */
        if (buf->p >= buf->end) {
            buf->p = buf->buffer;
            do {
                errno = 0;
                i = fdinfo_recv(&buf->fdn, buf->buffer, sizeof(buf->buffer));
            } while (i == -1 && errno == EINTR);
            if (i <= 0)
                return -1;
            buf->end = buf->buffer + i;
        }
        i = buf->end - buf->p;
        if (i < size - n) {
            memcpy(out + n, buf->p, i);
            buf->p += i;
            n += i;
        } else {
            memcpy(out + n, buf->p, size - n);
            buf->p += size - n;
            n += size - n;
        }
    }

    return n;
}

/* 获取缓冲区中剩余的内容。 */
char *socket_buffer_remainder(struct socket_buffer *buf, size_t *len)
{
    if (len != NULL)
        *len = buf->end - buf->p;

    return buf->p;
}

/* URI 函数在 test/test-uri.c 中有一个测试程序。在进行任何更改后运行测试，并为任何新函数添加测试。 */

void uri_init(struct uri *uri)
{
    uri->scheme = NULL;
    uri->host = NULL;
    uri->port = -1;
    uri->path = NULL;
}

void uri_free(struct uri *uri)
{
    free(uri->scheme);
    free(uri->host);
    free(uri->path);
}

static int hex_digit_value(char digit)
{
    const char *DIGITS = "0123456789abcdef";
    const char *p;

    if ((unsigned char) digit == '\0')
        return -1;
    p = strchr(DIGITS, tolower((int) (unsigned char) digit));
    if (p == NULL)
        return -1;

    return p - DIGITS;
}

/* 不区分大小写的字符串比较。 */
static int str_cmp_i(const char *a, const char *b)
{
    # 当两个字符串都未到达结尾时，执行循环
    while (*a != '\0' && *b != '\0') {
        # 将字符转换为小写并赋值给 ca 和 cb
        int ca, cb;
        ca = tolower((int) (unsigned char) *a);
        cb = tolower((int) (unsigned char) *b);
        # 如果 ca 和 cb 不相等，则返回它们的差值
        if (ca != cb)
            return ca - cb;
        # 指针向后移动一个位置
        a++;
        b++;
    }

    # 如果 a 和 b 都到达结尾，则返回 0
    if (*a == '\0' && *b == '\0')
        return 0;
    # 如果 a 到达结尾，则返回 -1
    else if (*a == '\0')
        return -1;
    # 如果 b 到达结尾，则返回 1
    else
        return 1;
# 比较两个字符串是否相等，返回比较结果
static int str_equal_i(const char *a, const char *b)
{
    return str_cmp_i(a, b) == 0;
}

# 将字符串转换为小写
static int lowercase(char *s)
{
    char *p;

    for (p = s; *p != '\0'; p++)
        *p = tolower((int) (unsigned char) *p);

    return p - s;
}

# 对字符串进行百分号解码
static int percent_decode(char *s)
{
    char *p, *q;

    # 跳过到第一个 '%'。如果没有百分号转义，这样可以在不进行任何复制的情况下返回。
    q = s;
    while (*q != '\0' && *q != '%')
        q++;

    p = q;
    while (*q != '\0') {
        if (*q == '%') {
            int c, d;

            q++;
            c = hex_digit_value(*q);
            if (c == -1)
                return -1;
            q++;
            d = hex_digit_value(*q);
            if (d == -1)
                return -1;

            *p++ = c * 16 + d;
            q++;
        } else {
            *p++ = *q++;
        }
    }
    *p = '\0';

    return p - s;
}

# 使用这些函数是因为 isalpha 和 isdigit 可能会根据区域设置而改变其含义
static int is_alpha_char(int c)
{
    return c != '\0' && strchr("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ", c) != NULL;
}

static int is_digit_char(int c)
{
    return c != '\0' && strchr("0123456789", c) != NULL;
}

# 获取给定 URI 方案的默认端口，如果未识别则返回 -1
static int scheme_default_port(const char *scheme)
{
    if (str_equal_i(scheme, "http"))
        return 80;

    return -1;
}

# 将 URI 字符串解析为 URI 结构体。如果 URI 的任何部分缺失，将在结构中成为 NULL 条目，端口将为 -1。出错时返回 NULL。参见 RFC 3986，第 3 节的语法。
struct uri *uri_parse(struct uri *uri, const char *uri_s)
{
    const char *p, *q;

    uri_init(uri);

    # Scheme，第 3.1 节
    p = uri_s;
    if (!is_alpha_char(*p))
        goto fail;
    # 从当前位置开始，遍历直到找到非字母、数字、加号、减号或者句点的字符
    for (q = p; is_alpha_char(*q) || is_digit_char(*q) || *q == '+' || *q == '-' || *q == '.'; q++)
        ;
    # 如果找到的字符不是冒号，则跳转到失败处理的标签
    if (*q != ':')
        goto fail;
    # 将从 p 到 q 的子串作为 scheme 存储到 uri 对象中
    uri->scheme = mkstr(p, q);
    # 将 scheme 转换为小写，以保证大小写不敏感
    lowercase(uri->scheme);

    # 解析 Authority 部分，即 URI 中的主机和端口部分
    p = q + 1;
    # 如果下一个字符是 '/' 并且紧接着的字符也是 '/'，则表示 Authority 部分存在
    if (*p == '/' && *(p + 1) == '/') {
        char *authority = NULL;

        p += 2;
        # 从 p 开始遍历，直到遇到 '/'、'?'、'#' 或者字符串结束符
        for (q = p; !(*q == '/' || *q == '?' || *q == '#' || *q == '\0'); q++)
            ;
        # 将从 p 到 q 的子串作为 authority 存储到 uri 对象中
        authority = mkstr(p, q);
        # 如果解析 authority 失败，则释放内存并跳转到失败处理的标签
        if (uri_parse_authority(uri, authority) == NULL) {
            free(authority);
            goto fail;
        }
        free(authority);

        p = q;
    }
    # 如果端口未指定，则使用 scheme 的默认端口
    if (uri->port == -1)
        uri->port = scheme_default_port(uri->scheme);

    # 解析 Path 部分，包括查询和片段部分，不进行百分号解码
    q = strchr(p, '\0');
    # 将从 p 到 q 的子串作为 path 存储到 uri 对象中
    uri->path = mkstr(p, q);

    # 返回解析后的 uri 对象
    return uri;
# 释放 URI 对象占用的内存，并返回 NULL
uri_free(uri);
return NULL;
}

/* 解析 URI 的权限部分。不支持用户信息（用户名和密码），如果存在将导致错误。参见 RFC 3986，第 3.2 节。
   出现错误时返回 NULL。 */
struct uri *uri_parse_authority(struct uri *uri, const char *authority)
{
    const char *portsep;
    const char *host_start, *host_end;
    const char *tail;

    /* 我们不支持带有 "user:pass@" 用户信息。代理服务器对此没有用处。 */
    if (strchr(authority, '@') != NULL)
        return NULL;

    /* 查找主机的起始和结束位置。 */
    host_start = authority;
    if (*host_start == '[') {
        /* 方括号中的 IPv6 地址。 */
        host_start++;
        host_end = strchr(host_start, ']');
        if (host_end == NULL)
            return NULL;
        portsep = host_end + 1;
        if (!(*portsep == ':' || *portsep == '\0'))
            return NULL;
    } else {
        portsep = strrchr(authority, ':');
        if (portsep == NULL)
            portsep = strchr(authority, '\0');
        host_end = portsep;
    }

    /* 获取端口号。 */
    if (*portsep == ':' && *(portsep + 1) != '\0') {
        long n;

        errno = 0;
        n = parse_long(portsep + 1, &tail);
        if (errno != 0 || *tail != '\0' || tail == portsep + 1 || n < 1 || n > 65535)
            return NULL;
        uri->port = n;
    } else {
        uri->port = -1;
    }

    /* 获取主机名。 */
    uri->host = mkstr(host_start, host_end);
    if (percent_decode(uri->host) < 0) {
        free(uri->host);
        uri->host = NULL;
        return NULL;
    }

    return uri;
}

static void http_header_node_free(struct http_header *node)
{
    free(node->name);
    free(node->value);
    free(node);
}

void http_header_free(struct http_header *header)
{
    struct http_header *p, *next;

    for (p = header; p != NULL; p = next) {
        next = p->next;
        http_header_node_free(p);
    }
}

/* RFC 2616, 第 2.2 节；参见 LWS。 */
# 判断字符是否为空格或制表符
static int is_space_char(int c)
{
    return c == ' ' || c == '\t';
}

/* RFC 2616, section 2.2. */
# 判断字符是否为控制字符
static int is_ctl_char(int c)
{
    return (c >= 0 && c <= 31) || c == 127;
}

/* RFC 2616, section 2.2. */
# 判断字符是否为分隔符
static int is_sep_char(int c)
{
    return c != '\0' && strchr("()<>@,;:\\\"/[]?={} \t", c) != NULL;
}

/* RFC 2616, section 2.2. */
# 判断字符是否为标记字符
static int is_token_char(char c)
{
    return !iscntrl((int) (unsigned char) c) && !is_sep_char((int) (unsigned char) c);
}

# 判断字符串是否为CRLF（回车换行）
static int is_crlf(const char *s)
{
    return *s == '\n' || (*s == '\r' && *(s + 1) == '\n');
}

# 跳过CRLF（回车换行）并返回新的字符串指针
static const char *skip_crlf(const char *s)
{
    if (*s == '\n')
        return s + 1;
    else if (*s == '\r' && *(s + 1) == '\n')
        return s + 2;

    ncat_assert(0);
    return NULL;
}

# 比较两个字段名是否相等
static int field_name_equal(const char *a, const char *b)
{
    return str_equal_i(a, b);
}

/* Get the value of every header with the given name, separated by commas. If
   you only want the first value for header fields that should not be
   concatenated in this way, use http_header_get_first. The returned string
   must be freed. */
# 获取具有给定名称的每个标头的值，以逗号分隔。如果只想要不应以这种方式连接的标头字段的第一个值，请使用http_header_get_first。返回的字符串必须被释放。
char *http_header_get(const struct http_header *header, const char *name)
{
    const struct http_header *p;
    char *buf = NULL;
    size_t size = 0, offset = 0;
    int count;

    count = 0;
}
    # 遍历链表中的每个节点，直到链表末尾
    for (p = header; p != NULL; p = p->next) {
        # 根据 RFC 2616，section 4.2 的规定，如果消息中存在具有相同字段名的多个消息头字段，则这些字段的整个字段值必须定义为逗号分隔的列表
        if (field_name_equal(p->name, name)) {
            # 如果已经有多个相同字段名的消息头字段，则在追加当前字段值之前，先在缓冲区中追加逗号和空格
            if (count > 0)
                strbuf_append_str(&buf, &size, &offset, ", ");
            # 将当前消息头字段的值追加到缓冲区中
            strbuf_append_str(&buf, &size, &offset, p->value);
            # 计数器加一
            count++;
        }
    }

    # 返回拼接后的消息头字段值
    return buf;
}

/* 获取给定名称的第一个标头的值。返回的字符串必须被释放。 */
const struct http_header *http_header_next(const struct http_header *header,
    const struct http_header *p, const char *name)
{
    if (p == NULL)
        p = header;
    else
        p = p->next;

    for (; p != NULL; p = p->next) {
        if (field_name_equal(p->name, name))
            return p;
    }

    return NULL;
}

/* 获取给定名称的第一个标头的值。返回的字符串必须被释放。 */
char *http_header_get_first(const struct http_header *header, const char *name)
{
    const struct http_header *p;

    p = http_header_next(header, NULL, name);
    if (p != NULL)
        return Strdup(p->value);

    return NULL;
}

/* 设置给定名称的标头的值。 */
struct http_header *http_header_set(struct http_header *header, const char *name, const char *value)
{
    struct http_header *node, **prev;

    header = http_header_remove(header, name);

    node = (struct http_header *) safe_malloc(sizeof(*node));
    node->name = Strdup(name);
    node->value = Strdup(value);
    node->next = NULL;

    /* 将其链接到列表的末尾。 */
    for (prev = &header; *prev != NULL; prev = &(*prev)->next)
        ;
    *prev = node;

    return header;
}

/* 从以空格分隔的字符串中读取一个标记。这只识别空格作为分隔符，因此字符串必须已经进行了 LWS 规范化。http_header_parse 执行此规范化。 */
static const char *read_token(const char *s, char **token)
{
    const char *t;

    while (*s == ' ')
        s++;
    t = s;
    while (is_token_char(*t))
        t++;
    if (s == t)
        return NULL;

    *token = mkstr(s, t);

    return t;
}

/* 从带引号的字符串中读取一个标记。 */
static const char *read_quoted_string(const char *s, char **quoted_string)
{
    char *buf = NULL;
    size_t size = 0, offset = 0;
    const char *t;

    while (is_space_char(*s))
        s++;
    if (*s != '"')
        return NULL;
    s++;
    t = s;
    while (*s != '"') {
        /* 当前字符不是双引号时，进入循环 */
        while (*t != '"' && *t != '\\') {
            /* 获取一块普通字符 */
            /* 这是 qdtext，即除了控制字符以外的文本 */
            if (is_ctl_char(*t)) {
                /* 如果是控制字符，则释放缓冲区并返回空指针 */
                free(buf);
                return NULL;
            }
            t++;
        }
        /* 将普通字符块追加到缓冲区 */
        strbuf_append(&buf, &size, &offset, s, t - s);
        /* 现在可能需要处理转义字符 */
        if (*t == '\\') {
            t++;
            /* 只能转义一个字符，即0-127的八位字符，但我们不允许0 */
            if (*t <= 0) {
                /* 如果是0，则释放缓冲区并返回空指针 */
                free(buf);
                return NULL;
            }
            /* 将转义字符追加到缓冲区 */
            strbuf_append(&buf, &size, &offset, t, 1);
            t++;
        }
        s = t;
    }
    s++;

    /* 将引号内的字符串赋值给指针 */
    *quoted_string = buf;
    return s;
# 从给定字符串中读取一个 token 或者带引号的字符串，返回剩余未读取的字符串和读取到的 token
static const char *read_token_or_quoted_string(const char *s, char **token)
{
    # 跳过空白字符
    while (is_space_char(*s))
        s++;
    # 如果当前字符是双引号，则读取带引号的字符串
    if (*s == '"')
        return read_quoted_string(s, token);
    # 否则读取 token
    else
        return read_token(s, token);
}

# 从给定字符串中读取 token 列表，返回剩余未读取的字符串和读取到的 tokens 数组以及 tokens 数量
static const char *read_token_list(const char *s, char **tokens[], size_t *n)
{
    char *token;

    *tokens = NULL;
    *n = 0;

    # 循环读取 token
    for (;;) {
        s = read_token(s, &token);
        # 如果读取失败，则释放内存并返回空指针
        if (s == NULL) {
            int i;

            for (i = 0; i < *n; i++)
                free((*tokens)[i]);
            free(*tokens);

            return NULL;
        }
        # 重新分配内存并存储读取到的 token
        *tokens = (char **) safe_realloc(*tokens, (*n + 1) * sizeof((*tokens)[0]));
        (*tokens)[(*n)++] = token;
        # 如果当前字符不是逗号，则退出循环
        if (*s != ',')
            break;
        s++;
    }

    return s;
}

# 从给定的 HTTP 头部中移除指定名称的头部，并返回移除后的头部
struct http_header *http_header_remove(struct http_header *header, const char *name)
{
    struct http_header *p, *next, **prev;

    prev = &header;
    # 遍历头部链表
    for (p = header; p != NULL; p = next) {
        next = p->next;
        # 如果找到指定名称的头部，则移除并释放内存
        if (field_name_equal(p->name, name)) {
            *prev = next;
            http_header_node_free(p);
            continue;
        }
        prev = &p->next;
    }

    return header;
}

# 移除 RFC 2616 第 13.5.1 节中列出的 hop-by-hop 头部，并且移除 Connection 头部中列出的任何头部
int http_header_remove_hop_by_hop(struct http_header **header)
{
    static const char *HOP_BY_HOP_HEADERS[] = {
        "Connection",
        "Keep-Alive",
        "Proxy-Authenticate",
        "Proxy-Authorization",
        "TE",
        "Trailers",
        "Transfer-Encoding",
        "Upgrade",
    };
    char *connection;
    char **connection_tokens;
    size_t num_connection_tokens;
    unsigned int i;

    # 从头部中获取 Connection 头部的值
    connection = http_header_get(*header, "Connection");
    # 如果连接存在
    if (connection != NULL) {
        # 声明指向字符常量的指针
        const char *p;
        # 从连接中读取令牌列表，并返回指向令牌列表的指针，以及令牌数量
        p = read_token_list(connection, &connection_tokens, &num_connection_tokens);
        # 如果指针为空，释放连接并返回 400 错误
        if (p == NULL) {
            free(connection);
            return 400;
        }
        # 如果指针指向的字符不为空，释放连接和连接令牌，并返回 400 错误
        if (*p != '\0') {
            free(connection);
            for (i = 0; i < num_connection_tokens; i++)
                free(connection_tokens[i]);
            free(connection_tokens);
            return 400;
        }
        # 释放连接
        free(connection);
    } else {
        # 连接令牌为空
        connection_tokens = NULL;
        num_connection_tokens = 0;
    }

    # 移除头部中的逐跳头部字段
    for (i = 0; i < sizeof(HOP_BY_HOP_HEADERS) / sizeof(HOP_BY_HOP_HEADERS[0]); i++)
        *header = http_header_remove(*header, HOP_BY_HOP_HEADERS[i]);
    # 移除头部中的连接令牌字段
    for (i = 0; i < num_connection_tokens; i++)
        *header = http_header_remove(*header, connection_tokens[i]);

    # 释放连接令牌
    for (i = 0; i < num_connection_tokens; i++)
        free(connection_tokens[i]);
    free(connection_tokens);

    # 返回成功
    return 0;
}

// 将 HTTP 头部结构体转换为字符串
char *http_header_to_string(const struct http_header *header, size_t *n)
{
    const struct http_header *p;
    char *buf = NULL;
    size_t size = 0, offset = 0;

    // 初始化字符串缓冲区
    strbuf_append_str(&buf, &size, &offset, "");

    // 遍历 HTTP 头部链表，将每个头部字段转换为字符串
    for (p = header; p != NULL; p = p->next)
        strbuf_sprintf(&buf, &size, &offset, "%s: %s\r\n", p->name, p->value);

    // 如果传入了指针变量 n，则将 offset 赋值给它
    if (n != NULL)
        *n = offset;

    return buf;
}

// 初始化 HTTP 请求结构体
void http_request_init(struct http_request *request)
{
    request->method = NULL;
    uri_init(&request->uri);
    request->version = HTTP_UNKNOWN;
    request->header = NULL;
    request->content_length_set = 0;
    request->content_length = 0;
    request->bytes_transferred = 0;
}

// 释放 HTTP 请求结构体占用的内存
void http_request_free(struct http_request *request)
{
    free(request->method);
    uri_free(&request->uri);
    http_header_free(request->header);
}

// 将 HTTP 请求结构体转换为字符串
char *http_request_to_string(const struct http_request *request, size_t *n)
{
    const char *path;
    char *buf = NULL;
    size_t size = 0, offset = 0;

    /* RFC 2616, section 5.1.2: "the absolute path cannot be empty; if none is
       present in the original URI, it MUST be given as "/" (the server
       root)." */
    // 获取请求的路径，如果为空则设置为 "/"
    path = request->uri.path;
    if (path[0] == '\0')
        path = "/";

    if (request->version == HTTP_09) {
        /* HTTP/0.9 doesn't have headers. See
           http://www.w3.org/Protocols/HTTP/AsImplemented.html. */
        // 对于 HTTP/0.9 版本的请求，只输出请求方法和路径
        strbuf_sprintf(&buf, &size, &offset, "%s %s\r\n", request->method, path);
    } else {
        const char *version;
        char *header_str;

        if (request->version == HTTP_10)
            version = " HTTP/1.0";
        else
            version = " HTTP/1.1";

        // 将请求方法、路径、版本和头部字段转换为字符串
        header_str = http_header_to_string(request->header, NULL);
        strbuf_sprintf(&buf, &size, &offset, "%s %s%s\r\n%s\r\n",
            request->method, path, version, header_str);
        free(header_str);
    }

    // 如果传入了指针变量 n，则将 offset 赋值给它
    if (n != NULL)
        *n = offset;

    return buf;
}
// 初始化 HTTP 响应结构体
void http_response_init(struct http_response *response)
{
    // 设置 HTTP 版本为未知
    response->version = HTTP_UNKNOWN;
    // 设置状态码为 0
    response->code = 0;
    // 设置状态短语为空
    response->phrase = NULL;
    // 设置头部为空
    response->header = NULL;
    // 设置内容长度未设置
    response->content_length_set = 0;
    // 设置内容长度为 0
    response->content_length = 0;
    // 设置传输字节数为 0
    response->bytes_transferred = 0;
}

// 释放 HTTP 响应结构体
void http_response_free(struct http_response *response)
{
    // 释放状态短语内存
    free(response->phrase);
    // 释放头部内存
    http_header_free(response->header);
}

// 将 HTTP 响应结构体转换为字符串
char *http_response_to_string(const struct http_response *response, size_t *n)
{
    char *buf = NULL;
    size_t size = 0, offset = 0;

    if (response->version == HTTP_09) {
        // 对于 HTTP/0.9，返回空字符串
        return Strdup("");
    } else {
        const char *version;
        char *header_str;

        if (response->version == HTTP_10)
            version = "HTTP/1.0";
        else
            version = "HTTP/1.1";

        // 将头部转换为字符串
        header_str = http_header_to_string(response->header, NULL);
        // 格式化输出 HTTP 响应字符串
        strbuf_sprintf(&buf, &size, &offset, "%s %d %s\r\n%s\r\n",
            version, response->code, response->phrase, header_str);
        // 释放头部字符串内存
        free(header_str);
    }

    // 如果传入了指针，设置其值为偏移量
    if (n != NULL)
        *n = offset;

    return buf;
}

// 读取 HTTP 头部
int http_read_header(struct socket_buffer *buf, char **result)
{
    char *line = NULL;
    char *header;
    size_t n = 0;
    size_t count;
    int blank;

    header = NULL;
}
    // 使用 do-while 循环读取 socket 缓冲区中的一行数据
    do {
        // 从 socket 缓冲区中读取一行数据
        line = socket_buffer_readline(buf, &count, MAX_HEADER_LENGTH);
        // 如果读取失败
        if (line == NULL) {
            // 释放内存
            free(header);
            // 如果读取的数据超过最大长度
            if (count >= MAX_HEADER_LENGTH)
                /* 请求实体太大。*/
                return 413;
            else
                return 400;
        }
        // 判断是否是空行
        blank = is_crlf(line);

        // 如果当前数据长度加上已读取数据长度超过最大长度
        if (n + count >= MAX_HEADER_LENGTH) {
            // 释放内存
            free(line);
            free(header);
            /* 请求实体太大。*/
            return 413;
        }

        // 重新分配内存，将已读取数据拷贝到 header 中
        header = (char *) safe_realloc(header, n + count + 1);
        memcpy(header + n, line, count);
        n += count;
        // 释放内存
        free(line);
    } while (!blank);
    // 将 header 最后一个字符设为结束符
    header[n] = '\0';

    // 将结果赋值给 result
    *result = header;

    // 返回 0 表示成功
    return 0;
# 跳过输入字符串中的空白字符
static const char *skip_lws(const char *s)
{
    for (;;) {
        while (is_space_char(*s))  # 跳过空白字符
            s++;

        if (*s == '\n' && is_space_char(*(s + 1))  # 如果是换行符且下一个字符是空白字符
            s += 1;
        else if (*s == '\r' && *(s + 1) == '\n' && is_space_char(*(s + 2))  # 如果是回车符、换行符且下一个字符是空白字符
            s += 2;
        else
            break;
    }

    return s;
}

/* 查看 RFC 2616 的第 4.2 节以获取头部格式。*/
int http_parse_header(struct http_header **result, const char *header)
{
    const char *p, *q;
    size_t value_len, value_offset;
    struct http_header *node, **prev;

    *result = NULL;
    prev = result;

    p = header;
    while (*p != '\0' && !is_crlf(p)) {  # 当字符串未结束且不是 CRLF 时
        /* 获取字段名。*/
        q = p;
        while (*q != '\0' && is_token_char(*q))  # 获取字段名直到遇到非 token 字符
            q++;
        if (*q != ':') {  # 如果不是冒号，释放内存并返回 400 错误
            http_header_free(*result);
            return 400;
        }

        node = (struct http_header *) safe_malloc(sizeof(*node));  # 分配内存
        node->name = mkstr(p, q);  # 设置字段名
        node->value = NULL;
        node->next = NULL;
        value_len = 0;
        value_offset = 0;

        /* 复制头部字段值直到遇到 CRLF。*/
        p = q + 1;
        p = skip_lws(p);  # 跳过空白字符
        for (;;) {
            q = p;
            while (*q != '\0' && !is_space_char(*q) && !is_crlf(q)) {  # 获取字段值直到遇到空白字符或 CRLF
                /* RFC 2616 的第 2.2 节不允许控制字符。*/
                if (iscntrl((int) (unsigned char) *q)) {  # 如果是控制字符，释放内存并返回 400 错误
                    http_header_node_free(node);
                    return 400;
                }
                q++;
            }
            strbuf_append(&node->value, &value_len, &value_offset, p, q - p);  # 追加字段值
            p = skip_lws(q);  # 跳过空白字符
            if (is_crlf(p))  # 如果是 CRLF，跳出循环
                break;
            /* 用单个空格替换 LWS。*/
            strbuf_append_str(&node->value, &value_len, &value_offset, " ");  # 追加空格
        }
        *prev = node;  # 设置前一个节点
        prev = &node->next;  # 设置前一个节点的下一个节点

        p = skip_crlf(p);  # 跳过 CRLF
    }

    return 0;
}
    # 获取 HTTP 头部中的 Content-Length 字段的值
    static int http_header_get_content_length(const struct http_header *header, int *content_length_set, unsigned long *content_length)
    {
        char *content_length_s;  // 定义指向 Content-Length 字段值的指针
        const char *tail;  // 定义指向字符串末尾的指针
        int code;  // 定义返回码变量

        content_length_s = http_header_get_first(header, "Content-Length");  // 获取 Content-Length 字段的值
        if (content_length_s == NULL) {  // 如果 Content-Length 字段不存在
            *content_length_set = 0;  // 设置 content_length_set 为 0
            *content_length = 0;  // 设置 content_length 为 0
            return 0;  // 返回 0
        }

        code = 0;  // 初始化返回码为 0

        errno = 0;  // 清空错误码
        *content_length_set = 1;  // 设置 content_length_set 为 1
        *content_length = parse_long(content_length_s, &tail);  // 解析 Content-Length 字段的值
        if (errno != 0 || *tail != '\0' || tail == content_length_s)  // 如果解析出错或者解析结果不完整
            code = 400;  // 设置返回码为 400
        free(content_length_s);  // 释放 Content-Length 字段值的内存

        return code;  // 返回返回码
    }

    /* 解析头部并填充请求结构中的相关字段。 */
    int http_request_parse_header(struct http_request *request, const char *header)
    {
        int code;  // 定义返回码变量

        code = http_parse_header(&request->header, header);  // 解析头部
        if (code != 0)  // 如果解析出错
            return code;  // 返回错误码
        code = http_header_get_content_length(request->header, &request->content_length_set, &request->content_length);  // 获取 Content-Length 字段的值
        if (code != 0)  // 如果获取出错
            return code;  // 返回错误码

        return 0;  // 返回 0
    }

    /* 解析头部并填充响应结构中的相关字段。 */
    int http_response_parse_header(struct http_response *response, const char *header)
    {
        int code;  // 定义返回码变量

        code = http_parse_header(&response->header, header);  // 解析头部
        if (code != 0)  // 如果解析出错
            return code;  // 返回错误码
        code = http_header_get_content_length(response->header, &response->content_length_set, &response->content_length);  // 获取 Content-Length 字段的值
        if (code != 0)  // 如果获取出错
            return code;  // 返回错误码

        return 0;  // 返回 0
    }

    int http_read_request_line(struct socket_buffer *buf, char **line)
    {
        size_t n;  // 定义大小变量

        *line = NULL;  // 将 line 指向 NULL

        /* RFC 2616 的第 4.1 节说 "服务器应该忽略在期望请求行的地方接收到的任何空行" */
    # 循环执行以下操作
    do {
        # 释放 *line 指向的内存
        free(*line);
        # 从 socket 缓冲区中读取一行数据，存储到 *line 中，并返回读取的字节数
        *line = socket_buffer_readline(buf, &n, MAX_REQUEST_LINE_LENGTH);
        # 如果 *line 为空，表示读取失败
        if (*line == NULL) {
            # 如果读取的字节数大于等于最大请求行长度，返回 413 错误
            if (n >= MAX_REQUEST_LINE_LENGTH)
                /* Request Entity Too Large. */
                return 413;
            # 否则返回 400 错误
            else
                return 400;
        }
    # 当 *line 不是以 CRLF 结尾时，继续循环
    } while (is_crlf(*line));

    # 循环结束后返回 0，表示读取成功
    return 0;
}

/* 返回 HTTP 版本后的字符指针，如果解析出错则返回 s。*/
static const char *parse_http_version(const char *s, enum http_version *version)
{
    const char *PREFIX = "HTTP/";
    const char *p, *q;
    long major, minor;

    *version = HTTP_UNKNOWN;

    p = s;
    if (memcmp(p, PREFIX, strlen(PREFIX)) != 0)
        return s;
    p += strlen(PREFIX);

    /* 主版本号。*/
    errno = 0;
    major = parse_long(p, &q);
    if (errno != 0 || q == p)
        return s;

    p = q;
    if (*p != '.')
        return s;
    p++;

    /* 次版本号。*/
    errno = 0;
    minor = parse_long(p, &q);
    if (errno != 0 || q == p)
        return s;

    if (major == 1 && minor == 0)
        *version = HTTP_10;
    else if (major == 1 && minor == 1)
        *version = HTTP_11;

    return q;
}

int http_parse_request_line(const char *line, struct http_request *request)
{
    const char *p, *q;
    struct uri *uri;
    char *uri_s;

    http_request_init(request);

    p = line;
    while (*p == ' ')
        p++;

    /* 方法（CONNECT、GET 等）。*/
    q = p;
    while (is_token_char(*q))
        q++;
    if (p == q)
        goto badreq;
    request->method = mkstr(p, q);

    /* URI。*/
    p = q;
    while (*p == ' ')
        p++;
    q = p;
    while (*q != '\0' && *q != ' ')
        q++;
    if (p == q)
        goto badreq;
    uri_s = mkstr(p, q);

    /* RFC 2616，第 5.1.1 节：方法区分大小写。
       RFC 2616，第 5.1.2 节：
         Request-URI    = "*" | absoluteURI | abs_path | authority
       当请求发送给代理时，绝对 URI 形式是必需的...
       授权形式仅被 CONNECT 方法使用。*/
    if (strcmp(request->method, "CONNECT") == 0) {
        uri = uri_parse_authority(&request->uri, uri_s);
    } else {
        uri = uri_parse(&request->uri, uri_s);
    }
    free(uri_s);
    if (uri == NULL)
        /* URI 解析失败。*/
        goto badreq;
    /* 版本号。*/
    p = q;  // 将指针 p 指向指针 q 所指向的位置
    while (*p == ' ')  // 当指针 p 所指向的字符为空格时
        p++;  // 指针 p 向后移动一位
    if (*p == '\0') {  // 如果指针 p 所指向的字符为空
        /* 没有 HTTP/X.X 版本号表示版本为 0.9。*/
        request->version = HTTP_09;  // 请求的版本号为 0.9
    } else {
        q = parse_http_version(p, &request->version);  // 调用函数解析 HTTP 版本号
        if (p == q)  // 如果指针 p 和指针 q 指向同一位置
            goto badreq;  // 跳转到错误请求的处理
    }

    return 0;  // 返回 0，表示执行成功
# 释放 HTTP 请求对象，并返回 400 错误码
badreq:
    http_request_free(request);
    return 400;
}

# 读取 HTTP 状态行，将结果存储在指针 line 指向的位置，返回 0 表示成功，非零表示失败
int http_read_status_line(struct socket_buffer *buf, char **line)
{
    size_t n;

    # 从 RFC 2616 第 6.1 节中引用规则，解释状态行的格式要求
    *line = socket_buffer_readline(buf, &n, MAX_STATUS_LINE_LENGTH);
    if (*line == NULL) {
        if (n >= MAX_STATUS_LINE_LENGTH)
            # 返回 413 错误码，表示请求实体过大
            return 413;
        else
            # 返回 400 错误码，表示请求错误
            return 400;
    }

    return 0;
}

# 解析 HTTP 状态行，将结果存储在结构体 response 中，返回 0 表示成功，-1 表示失败
int http_parse_status_line(const char *line, struct http_response *response)
{
    const char *p, *q;

    # 初始化 HTTP 响应结构体
    http_response_init(response);

    # 解析版本信息
    p = parse_http_version(line, &response->version);
    if (p == line)
        return -1;
    while (*p == ' ')
        p++;

    # 解析状态码
    errno = 0;
    response->code = parse_long(p, &q);
    if (errno != 0 || q == p)
        return -1;
    p = q;

    # 解析原因短语
    while (*p == ' ')
        p++;
    q = p;
    while (!is_crlf(q))
        q++;
    # 检查 CRLF 是否结束字符串
    if (*skip_crlf(q) != '\0')
        return -1;
    response->phrase = mkstr(p, q);

    return 0;
}

# 解析 HTTP 状态行的便捷包装，仅返回状态码，返回状态码表示成功，-1 表示失败
int http_parse_status_line_code(const char *line)
{
    struct http_response resp;
    int code;

    if (http_parse_status_line(line, &resp) != 0)
        return -1;
    code = resp.code;
    http_response_free(&resp);

    return code;
}

# 读取 HTTP 挑战，将结果存储在结构体 challenge 中
static const char *http_read_challenge(const char *s, struct http_challenge *challenge)
{
    const char *p;
    char *scheme;

    # 初始化 HTTP 挑战结构体
    http_challenge_init(challenge);

    scheme = NULL;
    s = read_token(s, &scheme);
    // 如果输入字符串为空，则跳转到错误处理
    if (s == NULL)
        goto bail;
    // 如果认证方案为"Basic"，则设置挑战方案为AUTH_BASIC
    if (str_equal_i(scheme, "Basic")) {
        challenge->scheme = AUTH_BASIC;
    } 
    // 如果认证方案为"Digest"，则设置挑战方案为AUTH_DIGEST
    else if (str_equal_i(scheme, "Digest")) {
        challenge->scheme = AUTH_DIGEST;
    } 
    // 其他情况下，设置挑战方案为AUTH_UNKNOWN
    else {
        challenge->scheme = AUTH_UNKNOWN;
    }
    // 释放scheme指针所指向的内存
    free(scheme);
    scheme = NULL;

    /* 根据 RFC 2617，第1.2节的规定，认证挑战至少需要一个auth-param：
         challenge = auth-scheme 1*SP 1#auth-param
       但是有一些方案（如NTLM和Negotiate）可以没有auth-params，因此在这里允许这种情况。
       逗号表示当前挑战的结束和下一个挑战的开始（参见下面循环中的注释）。 */
    // 跳过空格字符
    while (is_space_char(*s))
        s++;
    // 如果遇到逗号，则表示当前挑战结束，下一个挑战开始
    if (*s == ',') {
        s++;
        // 跳过空格字符
        while (is_space_char(*s))
            s++;
        // 如果字符串结束，则跳转到错误处理
        if (*s == '\0')
            goto bail;
        // 返回下一个挑战的起始位置
        return s;
    }

    }

    // 返回当前挑战的起始位置
    return s;
# 如果 scheme 不为空，则释放其内存
bail:
    if (scheme != NULL)
        free(scheme);
    # 释放 challenge 结构体内存
    http_challenge_free(challenge);

    # 返回空指针
    return NULL;
}

# 从字符串中读取认证信息
static const char *http_read_credentials(const char *s,
    struct http_credentials *credentials)
{
    const char *p;
    char *scheme;

    # 设置 scheme 为未知
    credentials->scheme = AUTH_UNKNOWN;

    # 读取 token，并将结果存入 scheme
    s = read_token(s, &scheme);
    if (s == NULL)
        return NULL;
    # 判断 scheme 类型，初始化相应的 credentials
    if (str_equal_i(scheme, "Basic")) {
        http_credentials_init_basic(credentials);
    } else if (str_equal_i(scheme, "Digest")) {
        http_credentials_init_digest(credentials);
    } else {
        free(scheme);
        return NULL;
    }
    free(scheme);

    # 跳过空格
    while (is_space_char(*s))
        s++;
    # 如果 scheme 是 BASIC，则读取 base64 编码的认证信息
    if (credentials->scheme == AUTH_BASIC) {
        p = s;
        # 读取 base64 编码的字符
        while (is_alpha_char(*p) || is_digit_char(*p) || *p == '+' || *p == '/' || *p == '=')
            p++;
        # 将 base64 编码的认证信息存入 credentials
        credentials->u.basic = mkstr(s, p);
        while (is_space_char(*p))
            p++;
        s = p;
    }

    return s;

# 如果出现错误，释放 credentials 内存并返回空指针
bail:
    http_credentials_free(credentials);

    return NULL;
}

# 判断认证方案的优先级
# 当支持 Digest 时，我们优先选择 Digest 而不是 Basic
static int auth_scheme_is_better(enum http_auth_scheme a,
    enum http_auth_scheme b)
{
    # 如果支持 Digest，则优先选择 Digest
    # 如果 b 是 Basic，则返回 0
    if (b == AUTH_DIGEST)
        return 0;
    # 如果 b 是 Digest，则返回 a 是否为 Digest
    if (b == AUTH_BASIC)
        return a == AUTH_DIGEST;
    # 如果 b 是未知，则返回 a 是否为 Basic 或 Digest
    if (b == AUTH_UNKNOWN)
        return a == AUTH_BASIC || a == AUTH_DIGEST;

    return 0;
}

# 获取代理服务器的认证挑战
struct http_challenge *http_header_get_proxy_challenge(const struct http_header *header, struct http_challenge *challenge)
{
    const struct http_header *p;

    # 初始化 challenge 结构体
    http_challenge_init(challenge);

    p = NULL;
    # 当在 HTTP 头部中找到下一个“Proxy-Authenticate”字段时执行循环
    while ((p = http_header_next(header, p, "Proxy-Authenticate")) != NULL) {
        # 临时存储指针的值
        const char *tmp;

        # 将指针的值赋给临时变量
        tmp = p->value;
        # 当临时变量不为字符串结束符时执行循环
        while (*tmp != '\0') {
            # 创建临时的 HTTP 挑战信息结构体
            struct http_challenge tmp_info;

            # 从临时变量中读取挑战信息，并将结果存储在临时的挑战信息结构体中
            tmp = http_read_challenge(tmp, &tmp_info);
            # 如果读取失败，则释放挑战信息并返回空指针
            if (tmp == NULL) {
                http_challenge_free(challenge);
                return NULL;
            }
            # 如果新的认证方案更好，则释放旧的挑战信息，将新的挑战信息赋给 challenge
            if (auth_scheme_is_better(tmp_info.scheme, challenge->scheme)) {
                http_challenge_free(challenge);
                *challenge = tmp_info;
            } else {
                # 否则释放临时的挑战信息
                http_challenge_free(&tmp_info);
            }
        }
    }

    # 返回挑战信息
    return challenge;
# 获取代理服务器的认证信息
struct http_credentials *http_header_get_proxy_credentials(const struct http_header *header, struct http_credentials *credentials)
{
    # 设置认证方案为未知
    credentials->scheme = AUTH_UNKNOWN;

    # 初始化指针 p
    p = NULL;
    # 遍历代理授权头部，获取代理认证信息
    while ((p = http_header_next(header, p, "Proxy-Authorization")) != NULL) {
        const char *tmp;

        # 临时指针指向头部值
        tmp = p->value;
        # 读取认证信息，存储在临时结构体中
        while (*tmp != '\0') {
            struct http_credentials tmp_info;

            tmp = http_read_credentials(tmp, &tmp_info);
            # 如果读取失败，释放内存并返回空指针
            if (tmp == NULL) {
                http_credentials_free(credentials);
                return NULL;
            }
            # 如果新的认证方案更好，释放原有内存，更新认证信息
            if (auth_scheme_is_better(tmp_info.scheme, credentials->scheme)) {
                http_credentials_free(credentials);
                *credentials = tmp_info;
            } else {
                http_credentials_free(&tmp_info);
            }
        }
    }

    # 返回认证信息
    return credentials;
}

# 初始化挑战结构体
void http_challenge_init(struct http_challenge *challenge)
{
    # 设置认证方案为未知
    challenge->scheme = AUTH_UNKNOWN;
    # 初始化其他字段为 NULL 或默认值
    challenge->realm = NULL;
    challenge->digest.nonce = NULL;
    challenge->digest.opaque = NULL;
    challenge->digest.algorithm = ALGORITHM_MD5;
    challenge->digest.qop = 0;
}

# 释放挑战结构体的内存
void http_challenge_free(struct http_challenge *challenge)
{
    # 释放 realm 字段的内存
    free(challenge->realm);
    # 如果认证方案为摘要认证，释放 nonce 和 opaque 字段的内存
    if (challenge->scheme == AUTH_DIGEST) {
        free(challenge->digest.nonce);
        free(challenge->digest.opaque);
    }
}

# 初始化基本认证结构体
void http_credentials_init_basic(struct http_credentials *credentials)
{
    # 设置认证方案为基本认证
    credentials->scheme = AUTH_BASIC;
    # 初始化其他字段为 NULL
    credentials->u.basic = NULL;
}

# 初始化摘要认证结构体
void http_credentials_init_digest(struct http_credentials *credentials)
{
    # 设置认证方案为摘要认证
    credentials->scheme = AUTH_DIGEST;
    # 初始化其他字段为 NULL 或默认值
    credentials->u.digest.username = NULL;
    credentials->u.digest.realm = NULL;
    credentials->u.digest.nonce = NULL;
    credentials->u.digest.uri = NULL;
    credentials->u.digest.response = NULL;
    credentials->u.digest.algorithm = ALGORITHM_MD5;
    credentials->u.digest.qop = QOP_NONE;
}
    # 将credentials结构体中的u字段中的digest字段中的nc字段设置为NULL
    credentials->u.digest.nc = NULL;
    # 将credentials结构体中的u字段中的digest字段中的cnonce字段设置为NULL
    credentials->u.digest.cnonce = NULL;
# 释放 HTTP 凭据结构体所占用的内存空间
void http_credentials_free(struct http_credentials *credentials)
{
    # 如果凭据方案是基本认证
    if (credentials->scheme == AUTH_BASIC) {
        # 释放基本认证的凭据信息
        free(credentials->u.basic);
    } 
    # 如果凭据方案是摘要认证
    else if (credentials->scheme == AUTH_DIGEST) {
        # 释放摘要认证的用户名
        free(credentials->u.digest.username);
        # 释放摘要认证的领域
        free(credentials->u.digest.realm);
        # 释放摘要认证的随机数
        free(credentials->u.digest.nonce);
        # 释放摘要认证的 URI
        free(credentials->u.digest.uri);
        # 释放摘要认证的响应
        free(credentials->u.digest.response);
        # 释放摘要认证的计数器
        free(credentials->u.digest.nc);
        # 释放摘要认证的客户端随机数
        free(credentials->u.digest.cnonce);
    }
}
```