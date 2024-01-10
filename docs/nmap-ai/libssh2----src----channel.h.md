# `nmap\libssh2\src\channel.h`

```
#ifndef __LIBSSH2_CHANNEL_H
#define __LIBSSH2_CHANNEL_H
/* 定义 LIBSSH2_CHANNEL_H 宏，用于条件编译，避免重复包含 */

/* 版权声明，版权所有，禁止未经许可的复制和使用 */

/* 定义在源码和二进制形式下的再分发和修改的条件 */

/* 如果再分发源代码，必须保留版权声明、条件列表和下面的免责声明 */

/* 如果再分发二进制形式，必须在文档和/或其他提供的材料中重现版权声明、条件列表和下面的免责声明 */

/* 未经特定书面许可，不得使用版权所有者的名称或其他贡献者的名称来认可或推广从本软件派生的产品 */

/* 免责声明，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保 */

/* 版权所有者和贡献者不对任何直接、间接、偶然、特殊、惩罚性或后果性损害负责，无论是合同责任、严格责任还是侵权行为，即使已被告知可能发生此类损害 */

/*
 * _libssh2_channel_receive_window_adjust
 *
 * 调整通道的接收窗口，通过调整字节。如果要调整的量小于 LIBSSH2_CHANNEL_MINADJUST，并且 force 为 0，则调整量将被排队等待后续数据包处理。
 *
 * 总是非阻塞的。
 */
# 调整通道接收窗口大小
int _libssh2_channel_receive_window_adjust(LIBSSH2_CHANNEL * channel,
                                           uint32_t adjustment,
                                           unsigned char force,
                                           unsigned int *store);

# 刷新一个（或所有）流中的数据，返回刷新的字节数，失败时返回负数
int _libssh2_channel_flush(LIBSSH2_CHANNEL *channel, int streamid);

# 确保通道关闭，然后从会话中移除通道并释放其资源，成功返回0，失败返回负数
int _libssh2_channel_free(LIBSSH2_CHANNEL *channel);

# 扩展数据
int
_libssh2_channel_extended_data(LIBSSH2_CHANNEL *channel, int ignore_mode);

# 向通道发送数据
ssize_t
_libssh2_channel_write(LIBSSH2_CHANNEL *channel, int stream_id,
                       const unsigned char *buf, size_t buflen);

# 建立一个通用会话通道
LIBSSH2_CHANNEL *
_libssh2_channel_open(LIBSSH2_SESSION * session, const char *channel_type,
                      uint32_t channel_type_len,
                      uint32_t window_size,
                      uint32_t packet_size,
                      const unsigned char *message, size_t message_len);

# 用于处理启动过程，用于libssh2_channel_(shell|exec|subsystem)
int
_libssh2_channel_process_startup(LIBSSH2_CHANNEL *channel,
                                 const char *request, size_t request_len,
                                 const char *message, size_t message_len);

# 从通道读取数据
# 重要提示：在当前读取的通道完成之前不要返回0。如果从网络中读取数据但没有有效载荷数据填充缓冲区，则必须确保返回PACKET_EAGAIN。
int
_libssh2_channel_read(LIBSSH2_CHANNEL *channel);
# 从通道中读取数据到缓冲区，返回读取的字节数
ssize_t _libssh2_channel_read(LIBSSH2_CHANNEL *channel, int stream_id,
                              char *buf, size_t buflen);

# 获取下一个通道的ID
uint32_t _libssh2_channel_nextid(LIBSSH2_SESSION * session);

# 根据通道ID查找通道
LIBSSH2_CHANNEL *_libssh2_channel_locate(LIBSSH2_SESSION * session,
                                         uint32_t channel_id);

# 获取通道数据包的长度
size_t _libssh2_channel_packet_data_len(LIBSSH2_CHANNEL * channel,
                                        int stream_id);

# 关闭通道
int _libssh2_channel_close(LIBSSH2_CHANNEL * channel);

/*
 * _libssh2_channel_forward_cancel
 *
 * 停止监听远程端口并释放监听器
 * 丢弃任何未处理（未接受）的连接
 *
 * 成功返回0，如果会阻塞则返回LIBSSH2_ERROR_EAGAIN，出错返回-1
 */
int _libssh2_channel_forward_cancel(LIBSSH2_LISTENER *listener);

#endif /* __LIBSSH2_CHANNEL_H */
```