# `nmap\libssh2\src\packet.h`

```cpp
#ifndef __LIBSSH2_PACKET_H
#define __LIBSSH2_PACKET_H
/*
 * 以下为版权声明和许可条件
 * Copyright (C) 2010 by Daniel Stenberg
 * Author: Daniel Stenberg <daniel@haxx.se>
 *
 * 在源代码和二进制形式下允许重新分发和使用，无论是否经过修改，只要满足以下条件：
 *
 *   1. 源代码的重新分发必须保留上述版权声明、条件列表和以下免责声明。
 *   2. 二进制形式的重新分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和免责声明。
 *   3. 未经特定事先书面许可，不得使用版权所有者的名称或其他贡献者的名称来认可或推广从本软件衍生的产品。
 *
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的适用性担保。在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）产生的任何损害，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责，即使已被告知可能发生此类损害的可能性。
 *
 */

// 从会话中读取数据包
int _libssh2_packet_read(LIBSSH2_SESSION * session);
# 定义函数 _libssh2_packet_ask，用于向服务器发送请求并获取响应数据
int _libssh2_packet_ask(LIBSSH2_SESSION * session, unsigned char packet_type,
                        unsigned char **data, size_t *data_len,
                        int match_ofs,
                        const unsigned char *match_buf,
                        size_t match_len);

# 定义函数 _libssh2_packet_askv，用于向服务器发送多种请求并获取响应数据
int _libssh2_packet_askv(LIBSSH2_SESSION * session,
                         const unsigned char *packet_types,
                         unsigned char **data, size_t *data_len,
                         int match_ofs,
                         const unsigned char *match_buf,
                         size_t match_len);

# 定义函数 _libssh2_packet_require，用于向服务器发送请求并要求特定响应数据
int _libssh2_packet_require(LIBSSH2_SESSION * session,
                            unsigned char packet_type, unsigned char **data,
                            size_t *data_len, int match_ofs,
                            const unsigned char *match_buf,
                            size_t match_len,
                            packet_require_state_t * state);

# 定义函数 _libssh2_packet_requirev，用于向服务器发送多种请求并要求特定响应数据
int _libssh2_packet_requirev(LIBSSH2_SESSION *session,
                             const unsigned char *packet_types,
                             unsigned char **data, size_t *data_len,
                             int match_ofs,
                             const unsigned char *match_buf,
                             size_t match_len,
                             packet_requirev_state_t * state);

# 定义函数 _libssh2_packet_burn，用于清空非阻塞状态下的数据包
int _libssh2_packet_burn(LIBSSH2_SESSION * session,
                         libssh2_nonblocking_states * state);

# 定义函数 _libssh2_packet_write，用于向服务器发送数据包
int _libssh2_packet_write(LIBSSH2_SESSION * session, unsigned char *data,
                          unsigned long data_len);

# 定义函数 _libssh2_packet_add，用于向会话添加数据包
int _libssh2_packet_add(LIBSSH2_SESSION * session, unsigned char *data,
                        size_t datalen, int macstate);

# 结束条件编译指令
#endif /* __LIBSSH2_PACKET_H */
```