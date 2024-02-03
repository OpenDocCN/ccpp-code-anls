# `nmap\libssh2\src\transport.h`

```cpp
#ifndef __LIBSSH2_TRANSPORT_H
#define __LIBSSH2_TRANSPORT_H
/* 定义宏，用于避免重复包含该头文件 */
/* 版权声明 */
/* 作者信息 */
/* 条款和条件 */
/* 二进制形式的再分发和使用条件 */
/* 代码形式的再分发和使用条件 */
/* 不得使用版权持有者的名称或其他贡献者的名称来认可或推广从本软件衍生的产品 */
/* 免责声明 */
/* 该文件处理 SECSH 传输层的读写操作，遵循 RFC4253 标准 */
#include "libssh2_priv.h"
#include "packet.h"
/*
 * libssh2_transport_send
 *
 * 发送一个数据包，如果需要的话进行加密并添加 MAC 码
 * 成功返回 0，失败返回非零值
 *
 * 该函数接收两个数据区域，这两个区域会被合并。'data' 部分会在 'data2' 之前立即发送。'data2' 可以设置为 NULL（或者 data2_len 为 0）以只使用一个部分。
 *
 * 如果该函数会阻塞或者整个数据包还没有发送完，则返回 LIBSSH2_ERROR_EAGAIN。如果是这样，调用者应该在更多数据可以发送时尽快再次调用该函数，并且该函数必须使用相同的参数集（相同的数据指针和相同的数据长度）调用，直到返回 ERROR_NONE 或者失败为止。
 *
 * 该函数在任何错误情况下都不会调用 _libssh2_error()。
 */
int _libssh2_transport_send(LIBSSH2_SESSION *session,
                            const unsigned char *data, size_t data_len,
                            const unsigned char *data2, size_t data2_len);

/*
 * _libssh2_transport_read
 *
 * 将一个数据包收集到输入 brigade 块中，block 控制是否等待数据包开始。
 *
 * 返回添加到输入 brigade 的数据包类型（如果没有添加任何内容则返回 PACKET_NONE），或者在失败时返回 PACKET_FAIL，在无法处理完整数据包时返回 PACKET_EAGAIN。
 */

/*
 * 该函数按照 RFC4253 第 6 章中指定的二进制流格式读取数据
 * "The Secure Shell (SSH) Transport Layer Protocol"
 */
int _libssh2_transport_read(LIBSSH2_SESSION * session);

#endif /* __LIBSSH2_TRANSPORT_H */
```