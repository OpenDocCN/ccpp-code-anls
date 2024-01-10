# `nmap\libssh2\src\agent_win.c`

```
/*
 * 版权声明和许可声明
 * 版权所有，禁止未经许可的源代码和二进制形式的再分发和使用
 * 在满足以下条件的情况下，允许进行修改后的再分发和使用：
 *   1. 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明
 *   2. 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 * 禁止使用版权所有者的名称或任何其他贡献者的名称，未经特定事先书面许可，不得用于认可或推广从本软件衍生的产品
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的适用性的担保
 * 在任何情况下，无论是在合同、严格责任还是侵权（包括疏忽或其他方式）的情况下，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责，即使已被告知可能发生此类损害
 */

#include "libssh2_priv.h"  // 包含 libssh2 私有头文件
#include "agent.h"  // 包含 agent 头文件
#include "misc.h"  // 包含 misc 头文件
#include <errno.h>  // 包含 errno 头文件
#ifdef HAVE_SYS_UN_H
#include <sys/un.h>  // 如果支持 Unix 域套接字，则包含 sys/un.h 头文件
#else
/* Use the existence of sys/un.h as a test if Unix domain socket is
   supported.  winsock*.h define PF_UNIX/AF_UNIX but do not actually
   support them. */
#undef PF_UNIX  // 如果不支持 Unix 域套接字，则取消定义 PF_UNIX
#endif
#include "userauth.h"  // 包含 userauth 头文件
#include "session.h"
#ifdef WIN32
#include <stdlib.h>
#endif

#ifdef WIN32
#define WIN32_OPENSSH_AGENT_SOCK "\\\\.\\pipe\\openssh-ssh-agent"

static int
agent_connect_openssh(LIBSSH2_AGENT *agent)
{
    int ret = LIBSSH2_ERROR_NONE;
    const char *path;
    HANDLE pipe = INVALID_HANDLE_VALUE;
    HANDLE event = NULL;

    // 获取身份代理路径，如果不存在则使用默认路径
    path = agent->identity_agent_path;
    if(!path) {
        path = getenv("SSH_AUTH_SOCK");
        if(!path)
            path = WIN32_OPENSSH_AGENT_SOCK;
    }

    // 循环尝试连接代理
    for(;;) {
        // 创建代理管道
        pipe = CreateFileA(
            path,
            GENERIC_READ | GENERIC_WRITE,
            0,
            NULL,
            OPEN_EXISTING,
            /* Non-blocking mode for agent connections is not implemented at
             * the point this was implemented. The code for Win32 OpenSSH
             * should support non-blocking IO, but the code calling it doesn't
             * support it as of yet.
             * When non-blocking IO is implemented for the surrounding code,
             * uncomment the following line to enable support within the Win32
             * OpenSSH code.
             */
            /* FILE_FLAG_OVERLAPPED | */
            SECURITY_SQOS_PRESENT |
            SECURITY_IDENTIFICATION,
            NULL
        );

        // 如果成功创建管道，则跳出循环
        if(pipe != INVALID_HANDLE_VALUE)
            break;
        // 如果管道忙，则等待
        if(GetLastError() != ERROR_PIPE_BUSY)
            break;

        /* Wait up to 1 second for a pipe instance to become available */
        if(!WaitNamedPipeA(path, 1000))
            break;
    }

    // 如果无法创建管道，则返回错误
    if(pipe == INVALID_HANDLE_VALUE) {
        ret = _libssh2_error(agent->session, LIBSSH2_ERROR_AGENT_PROTOCOL,
                             "unable to connect to agent pipe");
        goto cleanup;
    }

    // 设置管道信息
    if(SetHandleInformation(pipe, HANDLE_FLAG_INHERIT, 0) == FALSE) {
        ret = _libssh2_error(agent->session, LIBSSH2_ERROR_AGENT_PROTOCOL,
                             "unable to set handle information of agent pipe");
        goto cleanup;
    }
    # 创建一个事件对象，用于异步 I/O
    event = CreateEventA(NULL, TRUE, FALSE, NULL);
    # 如果事件对象创建失败
    if(event == NULL) {
        # 设置错误码并跳转到清理代码的部分
        ret = _libssh2_error(agent->session, LIBSSH2_ERROR_AGENT_PROTOCOL,
                             "unable to create async I/O event");
        goto cleanup;
    }

    # 将管道对象赋给代理对象的管道属性
    agent->pipe = pipe;
    # 将管道对象设置为无效值
    pipe = INVALID_HANDLE_VALUE;
    # 将事件对象赋给代理对象的重叠属性
    agent->overlapped.hEvent = event;
    # 将事件对象设置为NULL
    event = NULL;
    # 将文件描述符设置为0，表示连接已建立
    agent->fd = 0; /* Mark as the connection has been established */
# 清理操作的标签，用于在函数结束时执行清理操作
cleanup:
    # 如果事件句柄不为空，则关闭事件句柄
    if(event != NULL)
        CloseHandle(event);
    # 如果管道句柄不是无效句柄，则关闭管道句柄
    if(pipe != INVALID_HANDLE_VALUE)
        CloseHandle(pipe);
    # 返回函数的结果
    return ret;
}

# 定义一个宏，用于接收或发送所有数据
#define RECV_SEND_ALL(func, agent, buffer, length, total)              \
    # 定义变量，用于存储传输的字节数
    DWORD bytes_transfered;                                            \
    # 定义变量，用于存储函数的返回值
    BOOL ret;                                                          \
    # 定义变量，用于存储错误码
    DWORD err;                                                         \
    # 定义变量，用于存储函数的返回值
    int rc;                                                            \
    // 当总传输字节数小于指定长度时循环执行以下操作
    while(*total < length) {                                           \
        // 如果没有挂起的 I/O 操作
        if(!agent->pending_io)                                         \
            // 调用指定的函数进行数据传输
            ret = func(agent->pipe, (char *)buffer + *total,           \
                       (DWORD)(length - *total), &bytes_transfered,    \
                       &agent->overlapped);                            \
        else                                                           \
            // 获取挂起的 I/O 操作的结果
            ret = GetOverlappedResult(agent->pipe, &agent->overlapped, \
                                      &bytes_transfered, FALSE);       \
                                                                       \
        // 更新总传输字节数
        *total += bytes_transfered;                                    \
        // 如果操作失败
        if(!ret) {                                                     \
            // 获取错误码
            err = GetLastError();                                      \
            // 如果没有挂起的 I/O 操作且错误码为 ERROR_IO_PENDING
            // 或有挂起的 I/O 操作且错误码为 ERROR_IO_INCOMPLETE
            if((!agent->pending_io && ERROR_IO_PENDING == err)         \
               || (agent->pending_io && ERROR_IO_INCOMPLETE == err)) { \
                // 设置有挂起的 I/O 操作标志，返回 LIBSSH2_ERROR_EAGAIN
                agent->pending_io = TRUE;                              \
                return LIBSSH2_ERROR_EAGAIN;                           \
            }                                                          \
                                                                       \
            // 返回 LIBSSH2_ERROR_SOCKET_NONE
            return LIBSSH2_ERROR_SOCKET_NONE;                          \
        }                                                              \
        // 清除挂起的 I/O 操作标志
        agent->pending_io = FALSE;                                     \
    }                                                                  \
                                                                       \
    // 将总传输字节数转换为整数，重置总传输字节数，返回结果
    rc = (int)*total;                                                  \
    *total = 0;                                                        \
    return rc;
# 发送所有数据到 Win32 Openssh 代理
static int
win32_openssh_send_all(LIBSSH2_AGENT *agent, void *buffer, size_t length,
                       size_t *send_recv_total)
{
    RECV_SEND_ALL(WriteFile, agent, buffer, length, send_recv_total)
}

# 从 Win32 Openssh 代理接收所有数据
static int
win32_openssh_recv_all(LIBSSH2_AGENT *agent, void *buffer, size_t length,
                       size_t *send_recv_total)
{
    RECV_SEND_ALL(ReadFile, agent, buffer, length, send_recv_total)
}

# 取消 RECV_SEND_ALL 宏定义
#undef RECV_SEND_ALL

# 与 Openssh 代理进行交互
static int
agent_transact_openssh(LIBSSH2_AGENT *agent, agent_transaction_ctx_t transctx)
{
    unsigned char buf[4];
    int rc;

    # 发送请求的长度
    if(transctx->state == agent_NB_state_request_created) {
        _libssh2_htonu32(buf, (uint32_t)transctx->request_len);
        rc = win32_openssh_send_all(agent, buf, sizeof buf,
                                    &transctx->send_recv_total);
        if(rc == LIBSSH2_ERROR_EAGAIN)
            return LIBSSH2_ERROR_EAGAIN;
        else if(rc < 0)
            return _libssh2_error(agent->session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "agent send failed");
        transctx->state = agent_NB_state_request_length_sent;
    }

    # 发送请求体
    if(transctx->state == agent_NB_state_request_length_sent) {
        rc = win32_openssh_send_all(agent, transctx->request,
                                    transctx->request_len,
                                    &transctx->send_recv_total);
        if(rc == LIBSSH2_ERROR_EAGAIN)
            return LIBSSH2_ERROR_EAGAIN;
        else if(rc < 0)
            return _libssh2_error(agent->session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "agent send failed");
        transctx->state = agent_NB_state_request_sent;
    }

    # 接收请求体的长度
    # 如果传输上下文的状态为请求已发送
    if(transctx->state == agent_NB_state_request_sent) {
        # 从代理接收所有数据，并存储在buf中，同时更新发送接收总量
        rc = win32_openssh_recv_all(agent, buf, sizeof buf,
                                    &transctx->send_recv_total);
        # 如果返回值为LIBSSH2_ERROR_EAGAIN，则表示需要再次调用该函数
        if(rc == LIBSSH2_ERROR_EAGAIN)
            return LIBSSH2_ERROR_EAGAIN;
        # 如果返回值小于0，则表示接收失败，返回相应错误信息
        else if(rc < 0)
            return _libssh2_error(agent->session, LIBSSH2_ERROR_SOCKET_RECV,
                                  "agent recv failed");

        # 将接收到的响应长度转换为主机字节顺序，并存储在transctx->response_len中
        transctx->response_len = _libssh2_ntohu32(buf);
        # 为响应分配内存空间，大小为transctx->response_len
        transctx->response = LIBSSH2_ALLOC(agent->session,
                                           transctx->response_len);
        # 如果分配内存失败，则返回内存分配错误
        if(!transctx->response)
            return LIBSSH2_ERROR_ALLOC;

        # 将传输上下文的状态更新为响应长度已接收
        transctx->state = agent_NB_state_response_length_received;
    }

    # 接收响应体
    if(transctx->state == agent_NB_state_response_length_received) {
        # 从代理接收所有数据，并存储在transctx->response中，同时更新发送接收总量
        rc = win32_openssh_recv_all(agent, transctx->response,
                                    transctx->response_len,
                                    &transctx->send_recv_total);
        # 如果返回值为LIBSSH2_ERROR_EAGAIN，则表示需要再次调用该函数
        if(rc == LIBSSH2_ERROR_EAGAIN)
            return LIBSSH2_ERROR_EAGAIN;
        # 如果返回值小于0，则表示接收失败，返回相应错误信息
        else if(rc < 0)
            return _libssh2_error(agent->session, LIBSSH2_ERROR_SOCKET_RECV,
                                  "agent recv failed");
        # 将传输上下文的状态更新为响应已接收
        transctx->state = agent_NB_state_response_received;
    }

    # 返回无错误
    return LIBSSH2_ERROR_NONE;
}

static int
agent_disconnect_openssh(LIBSSH2_AGENT *agent)
{
    // 如果无法取消代理管道的挂起IO操作，则返回错误
    if(!CancelIo(agent->pipe))
        return _libssh2_error(agent->session, LIBSSH2_ERROR_SOCKET_DISCONNECT,
                              "failed to cancel pending IO of agent pipe");
    // 如果无法关闭异步IO事件句柄，则返回错误
    if(!CloseHandle(agent->overlapped.hEvent))
        return _libssh2_error(agent->session, LIBSSH2_ERROR_SOCKET_DISCONNECT,
                              "failed to close handle to async I/O event");
    agent->overlapped.hEvent = NULL;
    // 让排队的异步过程调用（如果有）执行完毕
    SleepEx(0, TRUE);
    // 如果无法关闭代理管道句柄，则返回错误
    if(!CloseHandle(agent->pipe))
        return _libssh2_error(agent->session, LIBSSH2_ERROR_SOCKET_DISCONNECT,
                              "failed to close handle to agent pipe");

    agent->pipe = INVALID_HANDLE_VALUE;
    agent->fd = LIBSSH2_INVALID_SOCKET;

    return LIBSSH2_ERROR_NONE;
}

// 定义代理操作的结构体，包括连接、交互和断开连接的函数指针
struct agent_ops agent_ops_openssh = {
    agent_connect_openssh,
    agent_transact_openssh,
    agent_disconnect_openssh
};
#endif  /* WIN32 */
```