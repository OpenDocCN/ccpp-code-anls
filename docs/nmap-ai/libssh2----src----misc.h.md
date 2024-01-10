# `nmap\libssh2\src\misc.h`

```
#ifndef __LIBSSH2_MISC_H
#define __LIBSSH2_MISC_H
/* 定义宏，用于避免重复包含该头文件 */

/* 版权声明，版权归 Daniel Stenberg 所有 */

/* 在源代码和二进制形式下，允许以原样或经过修改的形式进行再发布和使用，但需要满足以下条件：
 *   1. 源代码的再发布必须保留上述版权声明、条件列表和以下免责声明
 *   2. 二进制形式的再发布必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 *   3. 未经特定事先书面许可，不得使用版权所有者的名称或其他贡献者的名称来认可或推广从本软件派生的产品
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的适用性担保。无论在任何情况下，版权所有者或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失，或业务中断）负责，无论是在合同、严格责任或侵权行为（包括疏忽或其他方式）引起的，即使已被告知可能发生此类损害的情况下，使用本软件的任何方式都不承担任何责任。 */

/* 定义双向链表的结构体 */
struct list_head {
    struct list_node *last;  // 指向链表的最后一个节点
    struct list_node *first;  // 指向链表的第一个节点
};

/* 定义链表节点的结构体 */
struct list_node {
    struct list_node *next;  // 指向下一个节点
    struct list_node *prev;  // 指向上一个节点
    struct list_head *head;  // 指向链表头部
};

/* 定义字符串缓冲区的结构体 */
struct string_buf {
    unsigned char *data;  // 数据指针
    unsigned char *dataptr;  // 数据指针
    size_t len;  // 数据长度
};

#endif
# 返回错误标志
int _libssh2_error_flags(LIBSSH2_SESSION* session, int errcode,
                         const char *errmsg, int errflags);
# 返回错误
int _libssh2_error(LIBSSH2_SESSION* session, int errcode, const char *errmsg);

# 初始化链表
void _libssh2_list_init(struct list_head *head);

# 在链表末尾添加节点
void _libssh2_list_add(struct list_head *head,
                       struct list_node *entry);

# 返回链表中第一个节点
void *_libssh2_list_first(struct list_head *head);

# 返回链表中下一个节点
void *_libssh2_list_next(struct list_node *node);

# 返回链表中上一个节点
void *_libssh2_list_prev(struct list_node *node);

# 从链表中移除节点
void _libssh2_list_remove(struct list_node *entry);

# 对输入进行 Base64 编码
size_t _libssh2_base64_encode(LIBSSH2_SESSION *session,
                              const char *inp, size_t insize, char **outptr);

# 将网络字节序的 32 位整数转换为主机字节序
unsigned int _libssh2_ntohu32(const unsigned char *buf);
# 将网络字节序的 64 位整数转换为主机字节序
libssh2_uint64_t _libssh2_ntohu64(const unsigned char *buf);
# 将主机字节序的 32 位整数转换为网络字节序
void _libssh2_htonu32(unsigned char *buf, uint32_t val);
# 存储 32 位整数到缓冲区
void _libssh2_store_u32(unsigned char **buf, uint32_t value);
# 存储字符串到缓冲区
void _libssh2_store_str(unsigned char **buf, const char *str, size_t len);
# 分配内存并初始化为零
void *_libssh2_calloc(LIBSSH2_SESSION *session, size_t size);
# 显式将缓冲区清零
void _libssh2_explicit_zero(void *buf, size_t size);

# 创建新的字符串缓冲区
struct string_buf* _libssh2_string_buf_new(LIBSSH2_SESSION *session);
# 释放字符串缓冲区
void _libssh2_string_buf_free(LIBSSH2_SESSION *session,
                              struct string_buf *buf);
# 从字符串缓冲区中获取 32 位整数
int _libssh2_get_u32(struct string_buf *buf, uint32_t *out);
# 从字符串缓冲区中获取 64 位整数
int _libssh2_get_u64(struct string_buf *buf, libssh2_uint64_t *out);
# 从字符串缓冲区中匹配字符串
int _libssh2_match_string(struct string_buf *buf, const char *match);
# 从字符串缓冲区中获取字符串
int _libssh2_get_string(struct string_buf *buf, unsigned char **outbuf,
                        size_t *outlen);
# 从字符串缓冲区中复制字符串
int _libssh2_copy_string(LIBSSH2_SESSION* session, struct string_buf *buf,
                         unsigned char **outbuf, size_t *outlen);
# 定义一个函数，用于获取大整数的字节数
int _libssh2_get_bignum_bytes(struct string_buf *buf, unsigned char **outbuf,
                              size_t *outlen);
# 定义一个函数，用于检查缓冲区的长度是否满足请求的长度
int _libssh2_check_length(struct string_buf *buf, size_t requested_len);

# 如果在 Windows 平台且不是 MinGW 或 Cygwin 环境下，则提供一个私有的 gettimeofday 函数
# 取消 HAVE_GETTIMEOFDAY 宏定义
int __cdecl _libssh2_gettimeofday(struct timeval *tp, void *tzp);
# 定义 HAVE_LIBSSH2_GETTIMEOFDAY 宏
#define HAVE_LIBSSH2_GETTIMEOFDAY
# 启用 win32 实现的 gettimeofday 函数
#define LIBSSH2_GETTIMEOFDAY_WIN32
#else
# 如果有 gettimeofday 函数，则定义 _libssh2_gettimeofday 为 gettimeofday
#define _libssh2_gettimeofday(x,y) gettimeofday(x,y)
# 定义 HAVE_LIBSSH2_GETTIMEOFDAY 宏
#define HAVE_LIBSSH2_GETTIMEOFDAY
#endif

# 定义一个函数，用于对数据进行异或操作
void _libssh2_xor_data(unsigned char *output,
                       const unsigned char *input1,
                       const unsigned char *input2,
                       size_t length);

# 定义一个函数，用于对 AES 计数器进行增量操作
void _libssh2_aes_ctr_increment(unsigned char *ctr, size_t length);

# 结束 _LIBSSH2_MISC_H 宏定义
#endif /* _LIBSSH2_MISC_H */
```