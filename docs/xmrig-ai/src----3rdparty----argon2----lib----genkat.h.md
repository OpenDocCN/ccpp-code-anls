# `xmrig\src\3rdparty\argon2\lib\genkat.h`

```cpp
/*
 * Argon2源代码包
 *
 * 由Daniel Dinu和Dmitry Khovratovich编写，2015年
 *
 * 本作品根据知识共享CC0 1.0许可证/豁免授权发布。
 *
 * 您应该已经收到了CC0公共领域奉献的副本
 * 这个软件。如果没有，请参见
 * <http://creativecommons.org/publicdomain/zero/1.0/>。
 */

#ifndef ARGON2_KAT_H
#define ARGON2_KAT_H

#include "core.h"

/*
 * 打印输入到文件的初始KAT函数
 * @param  blockhash  包含预哈希摘要的数组
 * @param  context 保存输入
 * @param  type Argon2类型
 * @pre blockhash必须指向INPUT_INITIAL_HASH_LENGTH字节
 * @pre 上下文成员指针必须指向根据长度值分配的内存
 */
void initial_kat(const uint8_t *blockhash, const argon2_context *context,
                 argon2_type type);

/*
 * 打印输出标签的函数
 * @param  out  输出数组指针
 * @param  outlen 摘要长度
 * @pre out必须指向@a outlen字节
 **/
void print_tag(const void *out, uint32_t outlen);

/*
 * 打印给定时刻的内部状态的函数
 * @param  instance 指向当前实例的指针
 * @param  pass 当前传递号
 * @pre 实例必须具有必要的内存分配
 **/
void internal_kat(const argon2_instance_t *instance, uint32_t pass);

#endif
```