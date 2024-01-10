# `nmap\libssh2\src\agent.h`

```
#ifndef __LIBSSH2_AGENT_H
#define __LIBSSH2_AGENT_H
/*
 * 版权声明
 * 版权所有（C）2009年Daiki Ueno
 * 版权所有（C）2010-2014年Daniel Stenberg
 * 保留所有权利
 *
 * 在源代码和二进制形式下的再发布和使用，无论是否经过修改，都是允许的，只要满足以下条件：
 *
 *   1. 源代码的再发布必须保留上述版权声明、条件列表和以下免责声明。
 *
 *   2. 二进制形式的再发布必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明。
 *
 *   3. 未经特定事先书面许可，不得使用版权所有者的名称或其他贡献者的名称来认可或推广从本软件派生的产品。
 *
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他原因）责任，版权所有者或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责，即使已被告知此类损害的可能性。
 */

#include "libssh2_priv.h"  // 包含私有库头文件
#include "misc.h"  // 包含杂项头文件
#include "session.h"  // 包含会话头文件
#ifdef WIN32
#include <stdlib.h>  // 在 Windows 平台下包含标准库头文件
#endif

/* 非阻塞模式在代理连接上尚未实现，但为将来使用。 */
typedef enum {
    agent_NB_state_init = 0,  // 代理非阻塞状态初始化
    agent_NB_state_request_created,  // 代理非阻塞状态请求已创建
    # 定义了四个变量，分别表示代理的状态请求长度发送、代理的状态请求发送、代理的状态响应长度接收、代理的状态响应接收
    agent_NB_state_request_length_sent,
    agent_NB_state_request_sent,
    agent_NB_state_response_length_received,
    agent_NB_state_response_received
} agent_nonblocking_states;  // 定义了一个名为 agent_nonblocking_states 的结构体

typedef struct agent_transaction_ctx {
    unsigned char *request;  // 代理事务的请求数据
    size_t request_len;  // 请求数据的长度
    unsigned char *response;  // 代理事务的响应数据
    size_t response_len;  // 响应数据的长度
    agent_nonblocking_states state;  // 代理事务的非阻塞状态
    size_t send_recv_total;  // 发送和接收的总量
} *agent_transaction_ctx_t;  // 定义了一个名为 agent_transaction_ctx 的结构体指针类型

typedef int (*agent_connect_func)(LIBSSH2_AGENT *agent);  // 定义了一个名为 agent_connect_func 的函数指针类型
typedef int (*agent_transact_func)(LIBSSH2_AGENT *agent, agent_transaction_ctx_t transctx);  // 定义了一个名为 agent_transact_func 的函数指针类型
typedef int (*agent_disconnect_func)(LIBSSH2_AGENT *agent);  // 定义了一个名为 agent_disconnect_func 的函数指针类型

struct agent_publickey {
    struct list_node node;  // 代理公钥的链表节点

    /* this is the struct we expose externally */
    struct libssh2_agent_publickey external;  // 外部公钥结构体
};

struct agent_ops {
    agent_connect_func connect;  // 代理操作的连接函数
    agent_transact_func transact;  // 代理操作的事务函数
    agent_disconnect_func disconnect;  // 代理操作的断开连接函数
};

struct _LIBSSH2_AGENT
{
    LIBSSH2_SESSION *session;  /* the session this "belongs to" */  // 指向所属会话的指针

    libssh2_socket_t fd;  // 代理的套接字

    struct agent_ops *ops;  // 代理操作的指针

    struct agent_transaction_ctx transctx;  // 代理事务的上下文
    struct agent_publickey *identity;  // 代理的公钥
    struct list_head head;  /* list of public keys */  // 公钥链表的头部

    char *identity_agent_path; /* Path to a custom identity agent socket */  // 自定义身份代理套接字的路径

#ifdef WIN32
    OVERLAPPED overlapped;  // Windows 下的异步 I/O 结构
    HANDLE pipe;  // Windows 下的管道句柄
    BOOL pending_io;  // Windows 下的待处理 I/O
#endif
};

#ifdef WIN32
extern struct agent_ops agent_ops_openssh;  // Windows 下的 Openssh 代理操作
#endif

#endif /* __LIBSSH2_AGENT_H */  // 结束注释
```