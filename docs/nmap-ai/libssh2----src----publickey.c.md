# `nmap\libssh2\src\publickey.c`

```
# 版权声明和许可条件
/* Copyright (c) 2004-2007, Sara Golemon <sarag@libssh2.org>
 * Copyright (c) 2010-2014 by Daniel Stenberg
 * All rights reserved.
 *
 * Redistribution and use in source and binary forms,
 * with or without modification, are permitted provided
 * that the following conditions are met:
 *
 *   Redistributions of source code must retain the above
 *   copyright notice, this list of conditions and the
 *   following disclaimer.
 *
 *   Redistributions in binary form must reproduce the above
 *   copyright notice, this list of conditions and the following
 *   disclaimer in the documentation and/or other materials
 *   provided with the distribution.
 *
 *   Neither the name of the copyright holder nor the names
 *   of any other contributors may be used to endorse or
 *   promote products derived from this software without
 *   specific prior written permission.
 *
 * THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND
 * CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES,
 * INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES
 * OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
 * ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR
 * CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
 * SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING,
 * BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
 * SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
 * INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY,
 * WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
 * NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE
 * USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY
 * OF SUCH DAMAGE.
 */

#include "libssh2_priv.h"
#include "libssh2_publickey.h"
#include "channel.h"
#include "session.h"

# 定义 LIBSSH2_PUBLICKEY_VERSION 常量
#define LIBSSH2_PUBLICKEY_VERSION               2

# 定义响应状态码的数字化表示
# 不是 IETF 标准，只是本地表示
#define LIBSSH2_PUBLICKEY_RESPONSE_STATUS       0
# 定义版本响应的数字化表示
#define LIBSSH2_PUBLICKEY_RESPONSE_VERSION      1
// 定义公钥响应中公钥的代码值为2
#define LIBSSH2_PUBLICKEY_RESPONSE_PUBLICKEY    2

// 定义公钥响应代码列表结构体
typedef struct _LIBSSH2_PUBLICKEY_CODE_LIST
{
    int code;           // 响应代码
    const char *name;   // 响应名称
    int name_len;       // 响应名称长度
} LIBSSH2_PUBLICKEY_CODE_LIST;

// 定义公钥响应代码列表
static const LIBSSH2_PUBLICKEY_CODE_LIST publickey_response_codes[] =
{
    {LIBSSH2_PUBLICKEY_RESPONSE_STATUS, "status", sizeof("status") - 1},   // 状态响应
    {LIBSSH2_PUBLICKEY_RESPONSE_VERSION, "version", sizeof("version") - 1}, // 版本响应
    {LIBSSH2_PUBLICKEY_RESPONSE_PUBLICKEY, "publickey", sizeof("publickey") - 1}, // 公钥响应
    {0, NULL, 0}    // 结束标志
};

// 定义公钥状态代码
#define LIBSSH2_PUBLICKEY_SUCCESS               0
#define LIBSSH2_PUBLICKEY_ACCESS_DENIED         1
#define LIBSSH2_PUBLICKEY_STORAGE_EXCEEDED      2
#define LIBSSH2_PUBLICKEY_VERSION_NOT_SUPPORTED 3
#define LIBSSH2_PUBLICKEY_KEY_NOT_FOUND         4
#define LIBSSH2_PUBLICKEY_KEY_NOT_SUPPORTED     5
#define LIBSSH2_PUBLICKEY_KEY_ALREADY_PRESENT   6
#define LIBSSH2_PUBLICKEY_GENERAL_FAILURE       7
#define LIBSSH2_PUBLICKEY_REQUEST_NOT_SUPPORTED 8

// 定义公钥状态代码最大值
#define LIBSSH2_PUBLICKEY_STATUS_CODE_MAX       8

// 定义公钥状态代码列表
static const LIBSSH2_PUBLICKEY_CODE_LIST publickey_status_codes[] = {
    {LIBSSH2_PUBLICKEY_SUCCESS, "success", sizeof("success") - 1},   // 成功
    {LIBSSH2_PUBLICKEY_ACCESS_DENIED, "access denied", sizeof("access denied") - 1},   // 拒绝访问
    {LIBSSH2_PUBLICKEY_STORAGE_EXCEEDED, "storage exceeded", sizeof("storage exceeded") - 1},   // 存储超出
    {LIBSSH2_PUBLICKEY_VERSION_NOT_SUPPORTED, "version not supported", sizeof("version not supported") - 1},   // 版本不支持
    {LIBSSH2_PUBLICKEY_KEY_NOT_FOUND, "key not found", sizeof("key not found") - 1},   // 未找到密钥
    {LIBSSH2_PUBLICKEY_KEY_NOT_SUPPORTED, "key not supported", sizeof("key not supported") - 1},   // 不支持的密钥
    {LIBSSH2_PUBLICKEY_KEY_ALREADY_PRESENT, "key already present", sizeof("key already present") - 1},   // 密钥已存在
    {LIBSSH2_PUBLICKEY_GENERAL_FAILURE, "general failure", sizeof("general failure") - 1},   // 一般失败
    # 定义一个元组，包含错误码、错误消息和消息长度
    {LIBSSH2_PUBLICKEY_REQUEST_NOT_SUPPORTED, "request not supported",
     sizeof("request not supported") - 1},
    # 定义一个元组，包含错误码为0、空指针和消息长度为0
    {0, NULL, 0}
};

/*
 * publickey_status_error
 *
 * 格式化状态码的错误消息
 */
static void
publickey_status_error(const LIBSSH2_PUBLICKEY *pkey,
                       LIBSSH2_SESSION *session, int status)
{
    const char *msg;

    /* 在版本1和2之间重新映射了GENERAL_FAILURE */
    if(status == 6 && pkey && pkey->version == 1) {
        status = 7;
    }

    if(status < 0 || status > LIBSSH2_PUBLICKEY_STATUS_CODE_MAX) {
        msg = "unknown";
    }
    else {
        msg = publickey_status_codes[status].name;
    }

    _libssh2_error(session, LIBSSH2_ERROR_PUBLICKEY_PROTOCOL, msg);
}

/*
 * publickey_packet_receive
 *
 * 从子系统中读取一个数据包
 */
static int
publickey_packet_receive(LIBSSH2_PUBLICKEY * pkey,
                         unsigned char **data, size_t *data_len)
{
    LIBSSH2_CHANNEL *channel = pkey->channel;
    LIBSSH2_SESSION *session = channel->session;
    unsigned char buffer[4];
    int rc;
    *data = NULL; /* 默认情况下不返回任何内容 */
    *data_len = 0;

    if(pkey->receive_state == libssh2_NB_state_idle) {
        rc = _libssh2_channel_read(channel, 0, (char *) buffer, 4);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if(rc != 4) {
            return _libssh2_error(session, LIBSSH2_ERROR_PUBLICKEY_PROTOCOL,
                                  "来自公钥子系统的无效响应");
        }

        pkey->receive_packet_len = _libssh2_ntohu32(buffer);
        pkey->receive_packet =
            LIBSSH2_ALLOC(session, pkey->receive_packet_len);
        if(!pkey->receive_packet) {
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "无法分配公钥响应缓冲区");
        }

        pkey->receive_state = libssh2_NB_state_sent;
    }
    # 如果公钥接收状态为已发送
    if(pkey->receive_state == libssh2_NB_state_sent) {
        # 从通道中读取数据到接收数据包中
        rc = _libssh2_channel_read(channel, 0, (char *) pkey->receive_packet,
                                   pkey->receive_packet_len);
        # 如果返回值为再试一次，则返回该值
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 如果返回值不等于接收数据包长度
        else if(rc != (int)pkey->receive_packet_len) {
            # 释放接收数据包内存
            LIBSSH2_FREE(session, pkey->receive_packet);
            pkey->receive_packet = NULL;
            # 将接收状态设置为闲置
            pkey->receive_state = libssh2_NB_state_idle;
            # 返回超时错误信息
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_TIMEOUT,
                                  "Timeout waiting for publickey subsystem "
                                  "response packet");
        }

        # 将接收数据包和长度赋值给传入的指针
        *data = pkey->receive_packet;
        *data_len = pkey->receive_packet_len;
    }

    # 将接收状态设置为闲置
    pkey->receive_state = libssh2_NB_state_idle;

    # 返回 0 表示成功
    return 0;
/* publickey_response_id
 *
 * 将字符串响应名称转换为数字代码
 * 仅在成功时，数据将增加4 + response_len
 */
static int
publickey_response_id(unsigned char **pdata, size_t data_len)
{
    size_t response_len;
    unsigned char *data = *pdata;
    const LIBSSH2_PUBLICKEY_CODE_LIST *codes = publickey_response_codes;

    if(data_len < 4) {
        /* 响应格式错误 */
        return -1;
    }
    response_len = _libssh2_ntohu32(data);
    data += 4;
    data_len -= 4;
    if(data_len < response_len) {
        /* 响应格式错误 */
        return -1;
    }

    while(codes->name) {
        if((unsigned long)codes->name_len == response_len &&
            strncmp(codes->name, (char *) data, response_len) == 0) {
            *pdata = data + response_len;
            return codes->code;
        }
        codes++;
    }

    return -1;
}

/* publickey_response_success
 *
 * 等待成功响应并且没有其他内容的通用辅助程序
 */
static int
publickey_response_success(LIBSSH2_PUBLICKEY * pkey)
{
    LIBSSH2_SESSION *session = pkey->channel->session;
    unsigned char *data, *s;
    size_t data_len;
    int response;
    # 进入无限循环，等待公钥数据包的到来
    while(1) {
        # 接收公钥数据包，存储在 pkey 中，数据存储在 data 中，数据长度存储在 data_len 中
        int rc = publickey_packet_receive(pkey, &data, &data_len);
        # 如果返回值为 LIBSSH2_ERROR_EAGAIN，表示需要再次尝试接收，返回 rc
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 如果返回值不为 0，表示出现错误，返回错误信息
        else if(rc) {
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_TIMEOUT,
                                  "Timeout waiting for response from "
                                  "publickey subsystem");
        }

        # 如果数据长度小于 4，表示公钥响应过小，返回错误信息
        if(data_len < 4) {
            return _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                                  "Publickey response too small");
        }

        # 将数据指针赋值给 s，获取公钥响应类型
        s = data;
        response = publickey_response_id(&s, data_len);

        # 根据公钥响应类型进行处理
        switch(response) {
        case LIBSSH2_PUBLICKEY_RESPONSE_STATUS:
            # 错误或处理完成
        {
            unsigned long status = 0;

            # 如果数据长度小于 8，表示公钥响应过小，返回错误信息
            if(data_len < 8) {
                return _libssh2_error(session, LIBSSH2_ERROR_BUFFER_TOO_SMALL,
                                      "Publickey response too small");
            }

            # 将数据转换为无符号长整型
            status = _libssh2_ntohu32(s);

            # 释放数据内存
            LIBSSH2_FREE(session, data);

            # 如果状态为 LIBSSH2_PUBLICKEY_SUCCESS，表示公钥操作成功，返回 0
            if(status == LIBSSH2_PUBLICKEY_SUCCESS)
                return 0;

            # 处理公钥状态错误，返回 -1
            publickey_status_error(pkey, session, status);
            return -1;
        }
        default:
            # 释放数据内存
            LIBSSH2_FREE(session, data);
            # 如果响应类型小于 0，表示公钥协议错误，返回错误信息
            if(response < 0) {
                return _libssh2_error(session,
                                      LIBSSH2_ERROR_PUBLICKEY_PROTOCOL,
                                      "Invalid publickey subsystem response");
            }
            # 未知/意外情况，返回错误信息
            _libssh2_error(session, LIBSSH2_ERROR_PUBLICKEY_PROTOCOL,
                           "Unexpected publickey subsystem response");
            data = NULL;
        }
    }
    # 永远不会到达，但包含返回以消除编译器警告
    return -1;
}

/* *****************
 * Publickey API *
 ***************** */

/*
 * publickey_init
 *
 * Startup the publickey subsystem
 */
static LIBSSH2_PUBLICKEY *publickey_init(LIBSSH2_SESSION *session)
{
    int response;
    int rc;

    // 如果公钥初始化状态为闲置，则进行初始化
    if(session->pkeyInit_state == libssh2_NB_state_idle) {
        session->pkeyInit_data = NULL;
        session->pkeyInit_pkey = NULL;
        session->pkeyInit_channel = NULL;

        // 调试信息：初始化公钥子系统
        _libssh2_debug(session, LIBSSH2_TRACE_PUBLICKEY,
                       "Initializing publickey subsystem");

        session->pkeyInit_state = libssh2_NB_state_allocated;
    }

    // 如果公钥初始化状态为已分配，则执行以下代码
    if(session->pkeyInit_state == libssh2_NB_state_allocated) {

        // 打开一个会话通道
        session->pkeyInit_channel =
            _libssh2_channel_open(session, "session",
                                  sizeof("session") - 1,
                                  LIBSSH2_CHANNEL_WINDOW_DEFAULT,
                                  LIBSSH2_CHANNEL_PACKET_DEFAULT, NULL,
                                  0);
        // 如果通道打开失败，则返回空值
        if(!session->pkeyInit_channel) {
            if(libssh2_session_last_errno(session) == LIBSSH2_ERROR_EAGAIN)
                /* The error state is already set, so leave it */
                return NULL;
            // 设置错误信息并跳转到错误退出
            _libssh2_error(session, LIBSSH2_ERROR_CHANNEL_FAILURE,
                           "Unable to startup channel");
            goto err_exit;
        }

        session->pkeyInit_state = libssh2_NB_state_sent;
    }
    // 如果 pkeyInit_state 状态为 libssh2_NB_state_sent
    if(session->pkeyInit_state == libssh2_NB_state_sent) {
        // 处理启动公钥子系统的过程
        rc = _libssh2_channel_process_startup(session->pkeyInit_channel,
                                              "subsystem",
                                              sizeof("subsystem") - 1,
                                              "publickey",
                                              sizeof("publickey") - 1);
        // 如果返回值为 LIBSSH2_ERROR_EAGAIN，表示需要再次调用该函数
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            // 设置错误信息并返回 NULL
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block starting publickey subsystem");
            return NULL;
        }
        // 如果返回值不为 LIBSSH2_ERROR_EAGAIN，但不为 0，表示请求公钥子系统失败
        else if(rc) {
            // 设置错误信息并跳转到错误退出标签
            _libssh2_error(session, LIBSSH2_ERROR_CHANNEL_FAILURE,
                           "Unable to request publickey subsystem");
            goto err_exit;
        }

        // 修改 pkeyInit_state 状态为 libssh2_NB_state_sent1
        session->pkeyInit_state = libssh2_NB_state_sent1;
    }
    # 如果公钥初始化状态为已发送第一步
    if(session->pkeyInit_state == libssh2_NB_state_sent1) {
        # 声明一个指向无符号字符的指针
        unsigned char *s;
        # 调用函数发送扩展数据，忽略扩展数据
        rc = _libssh2_channel_extended_data(session->pkeyInit_channel,
                                         LIBSSH2_CHANNEL_EXTENDED_DATA_IGNORE);
        # 如果返回值为需要再次调用，则返回错误信息并返回空值
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block starting publickey subsystem");
            return NULL;
        }

        # 为公钥结构分配内存空间
        session->pkeyInit_pkey =
            LIBSSH2_CALLOC(session, sizeof(LIBSSH2_PUBLICKEY));
        # 如果内存分配失败，则返回错误信息并跳转到错误退出标签
        if(!session->pkeyInit_pkey) {
            _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                           "Unable to allocate a new publickey structure");
            goto err_exit;
        }
        # 设置公钥结构的通道和版本
        session->pkeyInit_pkey->channel = session->pkeyInit_channel;
        session->pkeyInit_pkey->version = 0;

        # 将数据写入会话的公钥初始化缓冲区
        s = session->pkeyInit_buffer;
        _libssh2_htonu32(s, 4 + (sizeof("version") - 1) + 4);
        s += 4;
        _libssh2_htonu32(s, sizeof("version") - 1);
        s += 4;
        memcpy(s, "version", sizeof("version") - 1);
        s += sizeof("version") - 1;
        _libssh2_htonu32(s, LIBSSH2_PUBLICKEY_VERSION);

        # 设置公钥初始化缓冲区已发送的字节数为0
        session->pkeyInit_buffer_sent = 0;

        # 输出调试信息，发送公钥广告版本支持
        _libssh2_debug(session, LIBSSH2_TRACE_PUBLICKEY,
                       "Sending publickey advertising version %d support",
                       (int) LIBSSH2_PUBLICKEY_VERSION);

        # 设置公钥初始化状态为已发送第二步
        session->pkeyInit_state = libssh2_NB_state_sent2;
    }
    // 如果 pkeyInit_state 等于 libssh2_NB_state_sent2
    if(session->pkeyInit_state == libssh2_NB_state_sent2) {
        // 向 pkeyInit_channel 写入数据，写入长度为 19 - session->pkeyInit_buffer_sent 的数据
        rc = _libssh2_channel_write(session->pkeyInit_channel, 0,
                                    session->pkeyInit_buffer,
                                    19 - session->pkeyInit_buffer_sent);
        // 如果返回值为 LIBSSH2_ERROR_EAGAIN
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            // 报错并返回 NULL
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block sending publickey version packet");
            return NULL;
        }
        // 如果返回值小于 0
        else if(rc < 0) {
            // 报错并跳转到 err_exit 标签
            _libssh2_error(session, rc,
                           "Unable to send publickey version packet");
            goto err_exit;
        }
        // 更新已发送的数据长度
        session->pkeyInit_buffer_sent += rc;
        // 如果已发送的数据长度小于 19
        if(session->pkeyInit_buffer_sent < 19) {
            // 报错并返回 NULL
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Need to be called again to complete this");
            return NULL;
        }

        // 更新状态为 libssh2_NB_state_sent3
        session->pkeyInit_state = libssh2_NB_state_sent3;
    }

    }

    // 标签，直接跳转到此处
  err_exit:
    // 更新状态为 libssh2_NB_state_sent4
    session->pkeyInit_state = libssh2_NB_state_sent4;
    // 如果 pkeyInit_channel 存在
    if(session->pkeyInit_channel) {
        // 关闭 pkeyInit_channel
        rc = _libssh2_channel_close(session->pkeyInit_channel);
        // 如果返回值为 LIBSSH2_ERROR_EAGAIN
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            // 报错并返回 NULL
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block closing channel");
            return NULL;
        }
    }
    // 如果 pkeyInit_pkey 存在
    if(session->pkeyInit_pkey) {
        // 释放 pkeyInit_pkey 的内存
        LIBSSH2_FREE(session, session->pkeyInit_pkey);
        session->pkeyInit_pkey = NULL;
    }
    // 如果 pkeyInit_data 存在
    if(session->pkeyInit_data) {
        // 释放 pkeyInit_data 的内存
        LIBSSH2_FREE(session, session->pkeyInit_data);
        session->pkeyInit_data = NULL;
    }
    // 更新状态为 libssh2_NB_state_idle
    session->pkeyInit_state = libssh2_NB_state_idle;
    // 返回 NULL
    return NULL;
}

/*
 * libssh2_publickey_init
 *
 * 启动公钥子系统
 */
LIBSSH2_API LIBSSH2_PUBLICKEY *
libssh2_publickey_init(LIBSSH2_SESSION *session)
{
    LIBSSH2_PUBLICKEY *ptr;

    BLOCK_ADJUST_ERRNO(ptr, session,
                       publickey_init(session));
    return ptr;
}



/*
 * libssh2_publickey_add_ex
 *
 * 添加新的公钥条目
 */
LIBSSH2_API int
libssh2_publickey_add_ex(LIBSSH2_PUBLICKEY *pkey, const unsigned char *name,
                         unsigned long name_len, const unsigned char *blob,
                         unsigned long blob_len, char overwrite,
                         unsigned long num_attrs,
                         const libssh2_publickey_attribute attrs[])
{
    LIBSSH2_CHANNEL *channel;
    LIBSSH2_SESSION *session;
    /*  19 = packet_len(4) + add_len(4) + "add"(3) + name_len(4) + {name}
        blob_len(4) + {blob} */
    unsigned long i, packet_len = 19 + name_len + blob_len;
    unsigned char *comment = NULL;
    unsigned long comment_len = 0;
    int rc;

    if(!pkey)
        return LIBSSH2_ERROR_BAD_USE;

    channel = pkey->channel;
    session = channel->session;

    }

    if(pkey->add_state == libssh2_NB_state_created) {
        rc = _libssh2_channel_write(channel, 0, pkey->add_packet,
                                    (pkey->add_s - pkey->add_packet));
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        else if((pkey->add_s - pkey->add_packet) != rc) {
            LIBSSH2_FREE(session, pkey->add_packet);
            pkey->add_packet = NULL;
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send publickey add packet");
        }
        LIBSSH2_FREE(session, pkey->add_packet);
        pkey->add_packet = NULL;

        pkey->add_state = libssh2_NB_state_sent;
    }

    rc = publickey_response_success(pkey);
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        return rc;
    }
    # 设置 pkey->add_state 的值为 libssh2_NB_state_idle
    pkey->add_state = libssh2_NB_state_idle;
    
    # 返回 rc 变量的值
    return rc;
/* 
 * libssh2_publickey_remove_ex
 * 移除现有的公钥，使得不能再使用它进行身份验证
 */
LIBSSH2_API int
libssh2_publickey_remove_ex(LIBSSH2_PUBLICKEY * pkey,
                            const unsigned char *name, unsigned long name_len,
                            const unsigned char *blob, unsigned long blob_len)
{
    LIBSSH2_CHANNEL *channel;
    LIBSSH2_SESSION *session;
    /* 22 = packet_len(4) + remove_len(4) + "remove"(6) + name_len(4) + {name}
       + blob_len(4) + {blob} */
    unsigned long packet_len = 22 + name_len + blob_len;
    int rc;

    if(!pkey)
        return LIBSSH2_ERROR_BAD_USE;

    channel = pkey->channel;
    session = channel->session;
    // 返回移除公钥的结果
    # 如果公钥的移除状态为闲置状态
    if(pkey->remove_state == libssh2_NB_state_idle) {
        # 清空移除数据包
        pkey->remove_packet = NULL;

        # 为移除数据包分配内存空间
        pkey->remove_packet = LIBSSH2_ALLOC(session, packet_len);
        # 如果内存分配失败
        if(!pkey->remove_packet) {
            # 返回内存分配错误
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "Unable to allocate memory for "
                                  "publickey \"remove\" packet");
        }

        # 设置移除数据包的指针
        pkey->remove_s = pkey->remove_packet;
        # 将数据包长度转换为网络字节顺序并写入数据包
        _libssh2_htonu32(pkey->remove_s, packet_len - 4);
        pkey->remove_s += 4;
        # 将"remove"字符串的长度转换为网络字节顺序并写入数据包
        _libssh2_htonu32(pkey->remove_s, sizeof("remove") - 1);
        pkey->remove_s += 4;
        # 将"remove"字符串复制到数据包中
        memcpy(pkey->remove_s, "remove", sizeof("remove") - 1);
        pkey->remove_s += sizeof("remove") - 1;
        # 将公钥名称的长度转换为网络字节顺序并写入数据包
        _libssh2_htonu32(pkey->remove_s, name_len);
        pkey->remove_s += 4;
        # 将公钥名称复制到数据包中
        memcpy(pkey->remove_s, name, name_len);
        pkey->remove_s += name_len;
        # 将公钥数据的长度转换为网络字节顺序并写入数据包
        _libssh2_htonu32(pkey->remove_s, blob_len);
        pkey->remove_s += 4;
        # 将公钥数据复制到数据包中
        memcpy(pkey->remove_s, blob, blob_len);
        pkey->remove_s += blob_len;

        # 输出调试信息，发送公钥移除数据包
        _libssh2_debug(session, LIBSSH2_TRACE_PUBLICKEY,
                       "Sending publickey \"remove\" packet: "
                       "type=%s blob_len=%ld",
                       name, blob_len);

        # 设置公钥移除状态为已创建
        pkey->remove_state = libssh2_NB_state_created;
    }
    # 如果公钥删除状态为已创建
    if(pkey->remove_state == libssh2_NB_state_created) {
        # 调用_libssh2_channel_write函数向通道写入数据，写入pkey->remove_packet中的数据直到(pkey->remove_s - pkey->remove_packet)长度
        rc = _libssh2_channel_write(channel, 0, pkey->remove_packet,
                                    (pkey->remove_s - pkey->remove_packet));
        # 如果返回值为LIBSSH2_ERROR_EAGAIN，表示需要再次调用该函数
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            return rc;
        }
        # 如果写入的数据长度不等于实际写入的长度
        else if((pkey->remove_s - pkey->remove_packet) != rc) {
            # 释放pkey->remove_packet的内存
            LIBSSH2_FREE(session, pkey->remove_packet);
            pkey->remove_packet = NULL;
            # 将公钥删除状态设置为闲置
            pkey->remove_state = libssh2_NB_state_idle;
            # 返回发送公钥删除数据包失败的错误信息
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send publickey remove packet");
        }
        # 释放pkey->remove_packet的内存
        LIBSSH2_FREE(session, pkey->remove_packet);
        pkey->remove_packet = NULL;
        # 将公钥删除状态设置为已发送
        pkey->remove_state = libssh2_NB_state_sent;
    }

    # 调用publickey_response_success函数，表示公钥删除成功
    rc = publickey_response_success(pkey);
    # 如果返回值为LIBSSH2_ERROR_EAGAIN，表示需要再次调用该函数
    if(rc == LIBSSH2_ERROR_EAGAIN) {
        return rc;
    }

    # 将公钥删除状态设置为闲置
    pkey->remove_state = libssh2_NB_state_idle;

    # 返回结果
    return rc;
# libssh2_publickey_list_fetch
# 从服务器获取支持的公钥列表
LIBSSH2_API int
libssh2_publickey_list_fetch(LIBSSH2_PUBLICKEY * pkey, unsigned long *num_keys,
                             libssh2_publickey_list ** pkey_list)
{
    LIBSSH2_CHANNEL *channel;
    LIBSSH2_SESSION *session;
    libssh2_publickey_list *list = NULL;
    unsigned long buffer_len = 12, keys = 0, max_keys = 0, i;
    # 12 = packet_len(4) + list_len(4) + "list"(4)
    int response;
    int rc;

    if(!pkey)
        return LIBSSH2_ERROR_BAD_USE;

    channel = pkey->channel;
    session = channel->session;

    if(pkey->listFetch_state == libssh2_NB_state_idle):
        pkey->listFetch_data = NULL;

        pkey->listFetch_s = pkey->listFetch_buffer
        _libssh2_htonu32(pkey->listFetch_s, buffer_len - 4)
        pkey->listFetch_s += 4
        _libssh2_htonu32(pkey->listFetch_s, sizeof("list") - 1)
        pkey->listFetch_s += 4
        memcpy(pkey->listFetch_s, "list", sizeof("list") - 1)
        pkey->listFetch_s += sizeof("list") - 1

        _libssh2_debug(session, LIBSSH2_TRACE_PUBLICKEY,
                       "Sending publickey \"list\" packet")

        pkey->listFetch_state = libssh2_NB_state_created

    if(pkey->listFetch_state == libssh2_NB_state_created):
        rc = _libssh2_channel_write(channel, 0,
                                    pkey->listFetch_buffer,
                                    (pkey->listFetch_s -
                                     pkey->listFetch_buffer))
        if(rc == LIBSSH2_ERROR_EAGAIN):
            return rc
        else if((pkey->listFetch_s - pkey->listFetch_buffer) != rc):
            pkey->listFetch_state = libssh2_NB_state_idle
            return _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "Unable to send publickey list packet")

        pkey->listFetch_state = libssh2_NB_state_sent
    /* 只有通过显式的 goto 语句才会到达这里 */
  err_exit:
    // 如果 pkey->listFetch_data 不为空，则释放其内存并将其置为 NULL
    if(pkey->listFetch_data) {
        LIBSSH2_FREE(session, pkey->listFetch_data);
        pkey->listFetch_data = NULL;
    }
    // 如果 list 不为空，则释放 pkey 和 list 相关的内存
    if(list) {
        libssh2_publickey_list_free(pkey, list);
    }
    // 将 pkey->listFetch_state 置为 libssh2_NB_state_idle
    pkey->listFetch_state = libssh2_NB_state_idle;
    // 返回 -1
    return -1;
/* libssh2_publickey_list_free
 * 释放先前获取的公钥列表
 */
LIBSSH2_API void
libssh2_publickey_list_free(LIBSSH2_PUBLICKEY * pkey,
                            libssh2_publickey_list * pkey_list)
{
    LIBSSH2_SESSION *session;
    libssh2_publickey_list *p = pkey_list;

    if(!pkey || !p)
        return;

    session = pkey->channel->session;

    while(p->packet) {
        if(p->attrs) {
            LIBSSH2_FREE(session, p->attrs);
        }
        LIBSSH2_FREE(session, p->packet);
        p++;
    }

    LIBSSH2_FREE(session, pkey_list);
}

/* libssh2_publickey_shutdown
 * 关闭公钥子系统
 */
LIBSSH2_API int
libssh2_publickey_shutdown(LIBSSH2_PUBLICKEY *pkey)
{
    LIBSSH2_SESSION *session;
    int rc;

    if(!pkey)
        return LIBSSH2_ERROR_BAD_USE;

    session = pkey->channel->session;

    /*
     * 确保状态变量中使用的所有内存都被释放
     */
    if(pkey->receive_packet) {
        LIBSSH2_FREE(session, pkey->receive_packet);
        pkey->receive_packet = NULL;
    }
    if(pkey->add_packet) {
        LIBSSH2_FREE(session, pkey->add_packet);
        pkey->add_packet = NULL;
    }
    if(pkey->remove_packet) {
        LIBSSH2_FREE(session, pkey->remove_packet);
        pkey->remove_packet = NULL;
    }
    if(pkey->listFetch_data) {
        LIBSSH2_FREE(session, pkey->listFetch_data);
        pkey->listFetch_data = NULL;
    }

    rc = _libssh2_channel_free(pkey->channel);
    if(rc == LIBSSH2_ERROR_EAGAIN)
        return rc;

    LIBSSH2_FREE(session, pkey);
    return 0;
}
```