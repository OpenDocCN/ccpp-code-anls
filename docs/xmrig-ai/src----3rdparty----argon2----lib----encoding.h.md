# `xmrig\src\3rdparty\argon2\lib\encoding.h`

```cpp
#ifndef ENCODING_H
#define ENCODING_H
#include "3rdparty/argon2.h"

// 定义最大解码通道数
#define ARGON2_MAX_DECODED_LANES UINT32_C(255)
// 定义最小解码盐长度
#define ARGON2_MIN_DECODED_SALT_LEN UINT32_C(8)
// 定义最小解码输出长度
#define ARGON2_MIN_DECODED_OUT_LEN UINT32_C(12)

/*
* 将Argon2哈希字符串编码到提供的缓冲区中。'dst_len'
* 包含'dst'缓冲区的大小（以字符为单位）；如果'dst_len'
* 小于所需字符数（包括终止0），则此函数返回ARGON2_ENCODING_ERROR。
*
* 成功时返回ARGON2_OK。
*/
int encode_string(char *dst, size_t dst_len, argon2_context *ctx,
                  argon2_type type);

/*
* 将Argon2哈希字符串解码到提供的结构'ctx'中。
* 在调用此函数之前必须设置的唯一字段是ctx.saltlen和
* ctx.outlen（必须是允许的最大盐和输出长度值），ctx.salt和ctx.out
* （必须是指定长度的缓冲区），以及ctx.pwd和ctx.pwdlen必须包含有效密码。
*
* 无效的输入字符串会导致错误。成功时，ctx是有效的，并且所有字段都已初始化。
*
* 返回值为ARGON2_OK表示成功，其他ARGON2_代码表示错误。
*/
int decode_string(argon2_context *ctx, const char *str, argon2_type type);

/* 返回具有长度len的编码字节流的长度 */
size_t b64len(uint32_t len);

/* 返回编码数字num的长度 */
size_t numlen(uint32_t num);

#endif
```