# `nmap\libssh2\src\agent.c`

```
/*
 * 版权声明和许可声明
 * 版权所有，禁止未经许可的源代码和二进制形式的再分发和使用
 * 在满足以下条件的情况下，允许进行修改后的再分发和使用：
 *   1. 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明
 *   2. 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 *   3. 未经特定事先书面许可，不得使用版权所有者的名称或其他贡献者的名称来认可或推广从本软件派生的产品
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保
 * 在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）责任，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责，即使已被告知可能发生此类损害
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
/* 包含 session.h 头文件 */
#include "session.h"
/* 如果是在 WIN32 平台，包含 stdlib.h 头文件 */
#ifdef WIN32
#include <stdlib.h>
#endif

/* 定义客户端向代理请求协议 1 密钥操作的请求 */
#define SSH_AGENTC_REQUEST_RSA_IDENTITIES 1
#define SSH_AGENTC_RSA_CHALLENGE 3
#define SSH_AGENTC_ADD_RSA_IDENTITY 7
#define SSH_AGENTC_REMOVE_RSA_IDENTITY 8
#define SSH_AGENTC_REMOVE_ALL_RSA_IDENTITIES 9
#define SSH_AGENTC_ADD_RSA_ID_CONSTRAINED 24

/* 定义客户端向代理请求协议 2 密钥操作的请求 */
#define SSH2_AGENTC_REQUEST_IDENTITIES 11
#define SSH2_AGENTC_SIGN_REQUEST 13
#define SSH2_AGENTC_ADD_IDENTITY 17
#define SSH2_AGENTC_REMOVE_IDENTITY 18
#define SSH2_AGENTC_REMOVE_ALL_IDENTITIES 19
#define SSH2_AGENTC_ADD_ID_CONSTRAINED 25

/* 定义客户端向代理请求与密钥类型无关的请求 */
#define SSH_AGENTC_ADD_SMARTCARD_KEY 20
#define SSH_AGENTC_REMOVE_SMARTCARD_KEY 21
#define SSH_AGENTC_LOCK 22
#define SSH_AGENTC_UNLOCK 23
#define SSH_AGENTC_ADD_SMARTCARD_KEY_CONSTRAINED 26

/* 代理向客户端的通用回复 */
#define SSH_AGENT_FAILURE 5
#define SSH_AGENT_SUCCESS 6

/* 代理向客户端的协议 1 密钥操作的回复 */
#define SSH_AGENT_RSA_IDENTITIES_ANSWER 2
#define SSH_AGENT_RSA_RESPONSE 4

/* 代理向客户端的协议 2 密钥操作的回复 */
#define SSH2_AGENT_IDENTITIES_ANSWER 12
#define SSH2_AGENT_SIGN_RESPONSE 14

/* 密钥约束标识符 */
#define SSH_AGENT_CONSTRAIN_LIFETIME 1
#define SSH_AGENT_CONSTRAIN_CONFIRM 2

#ifdef PF_UNIX
/* 在 UNIX 平台下，建立与代理的连接 */
static int
agent_connect_unix(LIBSSH2_AGENT *agent)
{
    const char *path;
    struct sockaddr_un s_un;

    /* 获取代理的身份验证路径 */
    path = agent->identity_agent_path;
    if(!path) {
        /* 如果没有指定身份验证路径，则尝试从环境变量中获取 */
        path = getenv("SSH_AUTH_SOCK");
        if(!path)
            /* 如果环境变量中也没有，则返回错误 */
            return _libssh2_error(agent->session, LIBSSH2_ERROR_BAD_USE,
                                  "no auth sock variable");
    }

    /* 创建 UNIX 套接字并连接代理 */
    agent->fd = socket(PF_UNIX, SOCK_STREAM, 0);
    # 如果代理的文件描述符小于0，则表示创建套接字失败，返回套接字错误
    if(agent->fd < 0)
        return _libssh2_error(agent->session, LIBSSH2_ERROR_BAD_SOCKET,
                              "failed creating socket");

    # 设置UNIX域套接字的地址家族为AF_UNIX
    s_un.sun_family = AF_UNIX;
    # 将路径复制到UNIX域套接字的路径中，确保不超过路径长度
    strncpy(s_un.sun_path, path, sizeof s_un.sun_path);
    # 确保路径末尾有一个空字符
    s_un.sun_path[sizeof(s_un.sun_path)-1] = 0; /* make sure there's a trailing
                                                   zero */
    # 尝试连接代理的文件描述符和UNIX域套接字
    if(connect(agent->fd, (struct sockaddr*)(&s_un), sizeof s_un) != 0) {
        # 连接失败时关闭文件描述符，并返回代理协议错误
        close(agent->fd);
        return _libssh2_error(agent->session, LIBSSH2_ERROR_AGENT_PROTOCOL,
                              "failed connecting with agent");
    }

    # 连接成功，返回无错误
    return LIBSSH2_ERROR_NONE;
# 定义一个宏，用于接收或发送所有数据
#define RECV_SEND_ALL(func, socket, buffer, length, flags, abstract) \
    int rc;                                                          \  # 声明一个整型变量 rc
    size_t finished = 0;                                             \  # 声明一个 size_t 类型的变量 finished，并初始化为 0
                                                                     \
    while(finished < length) {                                       \  # 循环，直到 finished 大于等于 length
        rc = func(socket,                                            \  # 调用指定的函数 func，接收或发送数据
                  (char *)buffer + finished, length - finished,      \  # 传入 socket、buffer、剩余长度和标志位
                  flags, abstract);                                  \
        if(rc < 0)                                                   \  # 如果返回值小于 0
            return rc;                                               \  # 直接返回返回值
                                                                     \
        finished += rc;                                              \  # 将返回值累加到 finished
    }                                                                \
                                                                     \
    return finished;                                                 \  # 返回累加后的 finished

# 定义一个静态函数，用于发送所有数据
static ssize_t _send_all(LIBSSH2_SEND_FUNC(func), libssh2_socket_t socket,  \  # 声明一个静态函数，接收函数指针、socket、数据指针、长度、标志位和抽象指针
                         const void *buffer, size_t length,              \
                         int flags, void **abstract)
{
    RECV_SEND_ALL(func, socket, buffer, length, flags, abstract);       \  # 调用宏，发送所有数据
}

# 定义一个静态函数，用于接收所有数据
static ssize_t _recv_all(LIBSSH2_RECV_FUNC(func), libssh2_socket_t socket,  \  # 声明一个静态函数，接收函数指针、socket、数据指针、长度、标志位和抽象指针
                         void *buffer, size_t length,                   \
                         int flags, void **abstract)
{
    RECV_SEND_ALL(func, socket, buffer, length, flags, abstract);       \  # 调用宏，接收所有数据
}

# 取消宏的定义
#undef RECV_SEND_ALL

# 定义一个函数，用于在 agent 之间传输数据
static int
agent_transact_unix(LIBSSH2_AGENT *agent, agent_transaction_ctx_t transctx)  \  # 声明一个函数，接收 agent 和 transaction 上下文
{
    unsigned char buf[4];                                             \  # 声明一个长度为 4 的无符号字符数组
    int rc;                                                           \  # 声明一个整型变量 rc

    /* Send the length of the request */                              \  # 发送请求的长度
    # 如果传输上下文的状态为请求已创建
    if(transctx->state == agent_NB_state_request_created) {
        # 将请求长度转换为网络字节顺序，并存入缓冲区
        _libssh2_htonu32(buf, transctx->request_len);
        # 发送缓冲区中的数据到指定的文件描述符
        rc = _send_all(agent->session->send, agent->fd,
                       buf, sizeof buf, 0, &agent->session->abstract);
        # 如果返回值为-EAGAIN，表示需要重试
        if(rc == -EAGAIN)
            return LIBSSH2_ERROR_EAGAIN;
        # 如果返回值小于0，表示发送失败
        else if(rc < 0)
            return _libssh2_error(agent->session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "agent send failed");
        # 修改传输上下文的状态为请求长度已发送
        transctx->state = agent_NB_state_request_length_sent;
    }

    # 发送请求体
    if(transctx->state == agent_NB_state_request_length_sent) {
        # 发送请求体到指定的文件描述符
        rc = _send_all(agent->session->send, agent->fd, transctx->request,
                       transctx->request_len, 0, &agent->session->abstract);
        # 如果返回值为-EAGAIN，表示需要重试
        if(rc == -EAGAIN)
            return LIBSSH2_ERROR_EAGAIN;
        # 如果返回值小于0，表示发送失败
        else if(rc < 0)
            return _libssh2_error(agent->session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "agent send failed");
        # 修改传输上下文的状态为请求已发送
        transctx->state = agent_NB_state_request_sent;
    }

    # 接收响应的长度
    if(transctx->state == agent_NB_state_request_sent) {
        # 从指定的文件描述符接收数据到缓冲区
        rc = _recv_all(agent->session->recv, agent->fd,
                       buf, sizeof buf, 0, &agent->session->abstract);
        # 如果返回值小于0
        if(rc < 0) {
            # 如果返回值为-EAGAIN，表示需要重试
            if(rc == -EAGAIN)
                return LIBSSH2_ERROR_EAGAIN;
            # 否则表示接收失败
            return _libssh2_error(agent->session, LIBSSH2_ERROR_SOCKET_RECV,
                                  "agent recv failed");
        }
        # 将缓冲区中的数据转换为主机字节顺序的无符号32位整数，存入响应长度中
        transctx->response_len = _libssh2_ntohu32(buf);
        # 为响应分配指定长度的内存空间
        transctx->response = LIBSSH2_ALLOC(agent->session,
                                           transctx->response_len);
        # 如果分配失败，返回内存分配错误
        if(!transctx->response)
            return LIBSSH2_ERROR_ALLOC;
        # 修改传输上下文的状态为响应长度已接收
        transctx->state = agent_NB_state_response_length_received;
    }

    # 接收响应体
    # 如果传输上下文的状态为代理服务器响应长度已接收
    if(transctx->state == agent_NB_state_response_length_received) {
        # 从套接字接收指定长度的数据到响应缓冲区中
        rc = _recv_all(agent->session->recv, agent->fd, transctx->response,
                       transctx->response_len, 0, &agent->session->abstract);
        # 如果接收失败
        if(rc < 0) {
            # 如果返回错误为暂时无法完成
            if(rc == -EAGAIN)
                # 返回暂时无法完成错误
                return LIBSSH2_ERROR_EAGAIN;
            # 返回套接字发送失败错误
            return _libssh2_error(agent->session, LIBSSH2_ERROR_SOCKET_SEND,
                                  "agent recv failed");
        }
        # 修改传输上下文的状态为代理服务器响应已接收
        transctx->state = agent_NB_state_response_received;
    }

    # 返回成功
    return 0;
# 关闭与 UNIX 代理的连接
static int
agent_disconnect_unix(LIBSSH2_AGENT *agent)
{
    # 关闭代理的文件描述符
    int ret;
    ret = close(agent->fd);
    # 如果关闭成功，则将文件描述符标记为无效
    if(ret != -1)
        agent->fd = LIBSSH2_INVALID_SOCKET;
    # 如果关闭失败，则返回错误信息
    else
        return _libssh2_error(agent->session, LIBSSH2_ERROR_SOCKET_DISCONNECT,
                              "failed closing the agent socket");
    # 返回无错误
    return LIBSSH2_ERROR_NONE;
}

# 定义与 UNIX 代理通信的操作
struct agent_ops agent_ops_unix = {
    agent_connect_unix,  # 连接 UNIX 代理的函数
    agent_transact_unix,  # 与 UNIX 代理交互的函数
    agent_disconnect_unix  # 断开与 UNIX 代理的连接的函数
};
#endif  /* PF_UNIX */

#ifdef WIN32
# 与 Pageant 通信的代码取自 PuTTY
# 版权归属 Robert de Bath, Joris van Rantwijk, Delian Delchev, Andreas Schultz, Jeroen Massar, Wez Furlong, Nicolas Barry, Justin Bradford, Ben Harris, Malcolm Smith, Ahmad Khalifa, Markus Kuhn, Colin Watson 和 CORE SDI S.A.
# 定义 Pageant 通信的相关常量
#define PAGEANT_COPYDATA_ID 0x804e50ba   /* 随机值 */
#define PAGEANT_MAX_MSGLEN  8192

# 连接到 Pageant 代理
static int
agent_connect_pageant(LIBSSH2_AGENT *agent)
{
    HWND hwnd;
    # 查找 Pageant 窗口
    hwnd = FindWindowA("Pageant", "Pageant");
    # 如果未找到窗口，则返回连接代理失败的错误信息
    if(!hwnd)
        return _libssh2_error(agent->session, LIBSSH2_ERROR_AGENT_PROTOCOL,
                              "failed connecting agent");
    # 将文件描述符标记为已建立连接
    agent->fd = 0;
    # 返回无错误
    return LIBSSH2_ERROR_NONE;
}

# 与 Pageant 代理进行交互
static int
agent_transact_pageant(LIBSSH2_AGENT *agent, agent_transaction_ctx_t transctx)
{
    HWND hwnd;
    char mapname[23];
    HANDLE filemap;
    unsigned char *p;
    unsigned char *p2;
    int id;
    COPYDATASTRUCT cds;

    # 如果交易上下文为空或请求长度超过最大消息长度，则返回非法输入的错误信息
    if(!transctx || 4 + transctx->request_len > PAGEANT_MAX_MSGLEN)
        return _libssh2_error(agent->session, LIBSSH2_ERROR_INVAL,
                              "illegal input");

    # 查找 Pageant 窗口
    hwnd = FindWindowA("Pageant", "Pageant");
    # 如果未找到窗口，则返回未找到 Pageant 代理的错误信息
    if(!hwnd)
        return _libssh2_error(agent->session, LIBSSH2_ERROR_AGENT_PROTOCOL,
                              "found no pageant");

    # 格式化共享内存的名称
    snprintf(mapname, sizeof(mapname),
             "PageantRequest%08x%c", (unsigned)GetCurrentThreadId(), '\0');
    # 创建一个文件映射对象，用于将文件映射到进程的地址空间
    filemap = CreateFileMappingA(INVALID_HANDLE_VALUE, NULL, PAGE_READWRITE,
                                 0, PAGEANT_MAX_MSGLEN, mapname);

    # 检查文件映射对象是否创建成功，如果失败则返回错误信息
    if(filemap == NULL || filemap == INVALID_HANDLE_VALUE)
        return _libssh2_error(agent->session, LIBSSH2_ERROR_AGENT_PROTOCOL,
                              "failed setting up pageant filemap");

    # 将文件映射对象映射到进程的地址空间，并返回指向映射区域的指针
    p2 = p = MapViewOfFile(filemap, FILE_MAP_WRITE, 0, 0, 0);
    # 检查映射是否成功，如果失败则关闭文件映射对象并返回错误信息
    if(p == NULL || p2 == NULL) {
        CloseHandle(filemap);
        return _libssh2_error(agent->session, LIBSSH2_ERROR_AGENT_PROTOCOL,
                              "failed to open pageant filemap for writing");
    }

    # 将数据写入映射区域
    _libssh2_store_str(&p2, (const char *)transctx->request,
                       transctx->request_len);

    # 设置剪贴板数据结构的成员
    cds.dwData = PAGEANT_COPYDATA_ID;
    cds.cbData = 1 + strlen(mapname);
    cds.lpData = mapname;

    # 发送消息到指定窗口，并返回接收消息的窗口的标识符
    id = SendMessage(hwnd, WM_COPYDATA, (WPARAM) NULL, (LPARAM) &cds);
    # 如果发送消息成功，则处理接收到的数据
    if(id > 0) {
        transctx->response_len = _libssh2_ntohu32(p);
        # 检查接收到的数据长度是否超出最大限制，如果是则释放资源并返回错误信息
        if(transctx->response_len > PAGEANT_MAX_MSGLEN) {
            UnmapViewOfFile(p);
            CloseHandle(filemap);
            return _libssh2_error(agent->session, LIBSSH2_ERROR_AGENT_PROTOCOL,
                                  "agent setup fail");
        }
        # 分配内存用于存储接收到的数据，并将数据复制到分配的内存中
        transctx->response = LIBSSH2_ALLOC(agent->session,
                                           transctx->response_len);
        if(!transctx->response) {
            UnmapViewOfFile(p);
            CloseHandle(filemap);
            return _libssh2_error(agent->session, LIBSSH2_ERROR_ALLOC,
                                  "agent malloc");
        }
        memcpy(transctx->response, p + 4, transctx->response_len);
    }

    # 关闭文件映射对象
    UnmapViewOfFile(p);
    CloseHandle(filemap);
    # 返回成功状态
    return 0;
}
# 断开与 Pageant 代理的连接
static int
agent_disconnect_pageant(LIBSSH2_AGENT *agent) {
    # 将代理的文件描述符设置为无效
    agent->fd = LIBSSH2_INVALID_SOCKET;
    # 返回 0 表示成功
    return 0;
}

# 定义 Pageant 代理的操作函数
struct agent_ops agent_ops_pageant = {
    agent_connect_pageant,
    agent_transact_pageant,
    agent_disconnect_pageant
};
#endif  /* WIN32 */

# 定义支持的代理后端
static struct {
    const char *name;
    struct agent_ops *ops;
} supported_backends[] = {
#ifdef WIN32
    {"Pageant", &agent_ops_pageant},
    {"OpenSSH", &agent_ops_openssh},
#endif  /* WIN32 */
#ifdef PF_UNIX
    {"Unix", &agent_ops_unix},
#endif  /* PF_UNIX */
    {NULL, NULL}
};

# 代理签名函数
static int
agent_sign(LIBSSH2_SESSION *session, unsigned char **sig, size_t *sig_len,
           const unsigned char *data, size_t data_len, void **abstract) {
    LIBSSH2_AGENT *agent = (LIBSSH2_AGENT *) (*abstract);
    agent_transaction_ctx_t transctx = &agent->transctx;
    struct agent_publickey *identity = agent->identity;
    ssize_t len = 1 + 4 + identity->external.blob_len + 4 + data_len + 4;
    ssize_t method_len;
    unsigned char *s;
    int rc;

    # 创建一个请求来签名数据
    if(transctx->state == agent_NB_state_init) {
        s = transctx->request = LIBSSH2_ALLOC(session, len);
        if(!transctx->request)
            return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                                  "out of memory");

        *s++ = SSH2_AGENTC_SIGN_REQUEST;
        # 密钥 blob
        _libssh2_store_str(&s, (const char *)identity->external.blob,
                           identity->external.blob_len);
        # 数据
        _libssh2_store_str(&s, (const char *)data, data_len);

        # 标志
        _libssh2_store_u32(&s, 0);

        transctx->request_len = s - transctx->request;
        transctx->send_recv_total = 0;
        transctx->state = agent_NB_state_request_created;
    }

    # 确保在 EAGAIN 的情况下重新调用
    # 检查传入的请求是否为SSH2_AGENTC_SIGN_REQUEST，如果不是则返回错误
    if(*transctx->request != SSH2_AGENTC_SIGN_REQUEST)
        return _libssh2_error(session, LIBSSH2_ERROR_BAD_USE,
                              "illegal request");

    # 检查代理操作是否存在，如果不存在则返回错误
    if(!agent->ops)
        /* if no agent has been connected, bail out */
        return _libssh2_error(session, LIBSSH2_ERROR_BAD_USE,
                              "agent not connected");

    # 调用代理操作的transact方法，处理传入的transctx，如果返回错误则跳转到error标签
    rc = agent->ops->transact(agent, transctx);
    if(rc) {
        goto error;
    }
    LIBSSH2_FREE(session, transctx->request);
    transctx->request = NULL;

    # 获取响应数据的长度和指针
    len = transctx->response_len;
    s = transctx->response;
    len--;
    if(len < 0) {
        rc = LIBSSH2_ERROR_AGENT_PROTOCOL;
        goto error;
    }
    # 检查响应数据的第一个字节是否为SSH2_AGENT_SIGN_RESPONSE，如果不是则返回错误
    if(*s != SSH2_AGENT_SIGN_RESPONSE) {
        rc = LIBSSH2_ERROR_AGENT_PROTOCOL;
        goto error;
    }
    s++;

    /* Skip the entire length of the signature */
    # 跳过签名的整个长度
    len -= 4;
    if(len < 0) {
        rc = LIBSSH2_ERROR_AGENT_PROTOCOL;
        goto error;
    }
    s += 4;

    /* Skip signing method */
    # 跳过签名方法的长度
    len -= 4;
    if(len < 0) {
        rc = LIBSSH2_ERROR_AGENT_PROTOCOL;
        goto error;
    }
    method_len = _libssh2_ntohu32(s);
    s += 4;
    len -= method_len;
    if(len < 0) {
        rc = LIBSSH2_ERROR_AGENT_PROTOCOL;
        goto error;
    }
    s += method_len;

    /* Read the signature */
    # 读取签名数据的长度
    len -= 4;
    if(len < 0) {
        rc = LIBSSH2_ERROR_AGENT_PROTOCOL;
        goto error;
    }
    *sig_len = _libssh2_ntohu32(s);
    s += 4;
    len -= *sig_len;
    if(len < 0) {
        rc = LIBSSH2_ERROR_AGENT_PROTOCOL;
        goto error;
    }

    # 分配内存存储签名数据，如果分配失败则返回错误
    *sig = LIBSSH2_ALLOC(session, *sig_len);
    if(!*sig) {
        rc = LIBSSH2_ERROR_ALLOC;
        goto error;
    }
    # 将签名数据拷贝到分配的内存中
    memcpy(*sig, s, *sig_len);

  error:
    # 释放transctx的请求和响应数据
    LIBSSH2_FREE(session, transctx->request);
    transctx->request = NULL;

    LIBSSH2_FREE(session, transctx->response);
    transctx->response = NULL;

    # 返回代理签名失败的错误信息
    return _libssh2_error(session, rc, "agent sign failure");
# 列出代理的身份信息
static int
agent_list_identities(LIBSSH2_AGENT *agent)
{
    # 获取代理的事务上下文
    agent_transaction_ctx_t transctx = &agent->transctx;
    ssize_t len, num_identities;
    unsigned char *s;
    int rc;
    unsigned char c = SSH2_AGENTC_REQUEST_IDENTITIES;

    # 创建一个列出身份信息的请求
    if(transctx->state == agent_NB_state_init) {
        transctx->request = &c;
        transctx->request_len = 1;
        transctx->send_recv_total = 0;
        transctx->state = agent_NB_state_request_created;
    }

    # 确保在 EAGAIN 的情况下重新调用
    if(*transctx->request != SSH2_AGENTC_REQUEST_IDENTITIES)
        return _libssh2_error(agent->session, LIBSSH2_ERROR_BAD_USE,
                              "illegal agent request");

    if(!agent->ops)
        # 如果没有连接代理，则退出
        return _libssh2_error(agent->session, LIBSSH2_ERROR_BAD_USE,
                              "agent not connected");

    rc = agent->ops->transact(agent, transctx);
    if(rc) {
        LIBSSH2_FREE(agent->session, transctx->response);
        transctx->response = NULL;
        return rc;
    }
    transctx->request = NULL;

    len = transctx->response_len;
    s = transctx->response;
    len--;
    if(len < 0) {
        rc = LIBSSH2_ERROR_AGENT_PROTOCOL;
        goto error;
    }
    if(*s != SSH2_AGENT_IDENTITIES_ANSWER) {
        rc = LIBSSH2_ERROR_AGENT_PROTOCOL;
        goto error;
    }
    s++;

    # 读取身份信息的长度
    len -= 4;
    if(len < 0) {
        rc = LIBSSH2_ERROR_AGENT_PROTOCOL;
        goto error;
    }
    num_identities = _libssh2_ntohu32(s);
    s += 4;

    }
 error:
    LIBSSH2_FREE(agent->session, transctx->response);
    transctx->response = NULL;

    return _libssh2_error(agent->session, rc,
                          "agent list id failed");
}

# 释放代理的身份信息
static void
agent_free_identities(LIBSSH2_AGENT *agent)
{
    struct agent_publickey *node;
    struct agent_publickey *next;
    # 遍历代理对象的链表，从链表头开始
    for(node = _libssh2_list_first(&agent->head); node; node = next) {
        # 获取下一个节点的指针
        next = _libssh2_list_next(&node->node);
        # 释放节点中外部数据的内存
        LIBSSH2_FREE(agent->session, node->external.blob);
        LIBSSH2_FREE(agent->session, node->external.comment);
        # 释放节点的内存
        LIBSSH2_FREE(agent->session, node);
    }
    # 初始化代理对象的链表
    _libssh2_list_init(&agent->head);
}

#define AGENT_PUBLICKEY_MAGIC 0x3bdefed2
/*
 * agent_publickey_to_external()
 *
 * Copies data from the internal to the external representation struct.
 *
 */
static struct libssh2_agent_publickey *
agent_publickey_to_external(struct agent_publickey *node)
{
    // 将内部表示的数据复制到外部表示的结构体中
    struct libssh2_agent_publickey *ext = &node->external;

    // 设置外部表示的结构体的魔数
    ext->magic = AGENT_PUBLICKEY_MAGIC;
    // 将内部表示的结构体赋值给外部表示的结构体
    ext->node = node;

    return ext;
}

/*
 * libssh2_agent_init
 *
 * Init an ssh-agent handle. Returns the pointer to the handle.
 *
 */
LIBSSH2_API LIBSSH2_AGENT *
libssh2_agent_init(LIBSSH2_SESSION *session)
{
    LIBSSH2_AGENT *agent;

    // 分配内存并初始化 ssh-agent 句柄
    agent = LIBSSH2_CALLOC(session, sizeof *agent);
    if(!agent) {
        // 如果分配失败，返回错误信息
        _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                       "Unable to allocate space for agent connection");
        return NULL;
    }
    // 初始化句柄的一些属性
    agent->fd = LIBSSH2_INVALID_SOCKET;
    agent->session = session;
    agent->identity_agent_path = NULL;
    _libssh2_list_init(&agent->head);

#ifdef WIN32
    // 如果是在 Windows 下，进行一些额外的初始化
    agent->pipe = INVALID_HANDLE_VALUE;
    memset(&agent->overlapped, 0, sizeof(OVERLAPPED));
    agent->pending_io = FALSE;
#endif

    return agent;
}

/*
 * libssh2_agent_connect()
 *
 * Connect to an ssh-agent.
 *
 * Returns 0 if succeeded, or a negative value for error.
 */
LIBSSH2_API int
libssh2_agent_connect(LIBSSH2_AGENT *agent)
{
    int i, rc = -1;
    // 遍历支持的后端，尝试连接
    for(i = 0; supported_backends[i].name; i++) {
        agent->ops = supported_backends[i].ops;
        rc = (agent->ops->connect)(agent);
        if(!rc)
            return 0;
    }
    return rc;
}

/*
 * libssh2_agent_list_identities()
 *
 * Request ssh-agent to list identities.
 *
 * Returns 0 if succeeded, or a negative value for error.
 */
LIBSSH2_API int
libssh2_agent_list_identities(LIBSSH2_AGENT *agent)
{
    // 清空传输上下文
    memset(&agent->transctx, 0, sizeof agent->transctx);
    /* Abandon the last fetched identities */
    // 放弃上次获取的身份信息
    agent_free_identities(agent);
    return agent_list_identities(agent);
}
# 获取 SSH 代理中的公钥身份
LIBSSH2_API int
libssh2_agent_get_identity(LIBSSH2_AGENT *agent,
                           struct libssh2_agent_publickey **ext,
                           struct libssh2_agent_publickey *oprev)
{
    struct agent_publickey *node;
    # 如果存在上一个公钥身份，则获取下一个
    if(oprev && oprev->node) {
        /* we have a starting point */
        struct agent_publickey *prev = oprev->node;

        /* get the next node in the list */
        node = _libssh2_list_next(&prev->node);
    }
    # 否则获取第一个公钥身份
    else
        node = _libssh2_list_first(&agent->head);

    # 如果没有公钥身份，则返回 1
    if(!node)
        /* no (more) node */
        return 1;

    # 将内部的公钥身份转换为外部的公钥身份
    *ext = agent_publickey_to_external(node);

    # 返回成功
    return 0;
}

# 使用 SSH 代理进行公钥用户认证
LIBSSH2_API int
libssh2_agent_userauth(LIBSSH2_AGENT *agent,
                       const char *username,
                       struct libssh2_agent_publickey *identity)
{
    void *abstract = agent;
    int rc;

    # 如果用户认证状态为空闲，则进行初始化
    if(agent->session->userauth_pblc_state == libssh2_NB_state_idle) {
        memset(&agent->transctx, 0, sizeof agent->transctx);
        agent->identity = identity->node;
    }

    # 调整阻塞状态，进行用户认证
    BLOCK_ADJUST(rc, agent->session,
                 _libssh2_userauth_publickey(agent->session, username,
                                             strlen(username),
                                             identity->blob,
                                             identity->blob_len,
                                             agent_sign,
                                             &abstract));
    return rc;
}
/*
 * libssh2_agent_disconnect()
 *
 * 关闭与 ssh-agent 的连接。
 *
 * 如果成功则返回 0，否则返回负值表示错误。
 */
LIBSSH2_API int
libssh2_agent_disconnect(LIBSSH2_AGENT *agent)
{
    // 如果 agent 的操作存在并且文件描述符不是无效的套接字
    if(agent->ops && agent->fd != LIBSSH2_INVALID_SOCKET)
        // 调用操作中的 disconnect 方法
        return agent->ops->disconnect(agent);
    // 返回 0
    return 0;
}

/*
 * libssh2_agent_free()
 *
 * 释放 ssh-agent 句柄。此函数还会释放内部的公钥集合。
 */
LIBSSH2_API void
libssh2_agent_free(LIBSSH2_AGENT *agent)
{
    /* 当套接字失去连接时允许连接释放 */
    if(agent->fd != LIBSSH2_INVALID_SOCKET) {
        // 断开与 ssh-agent 的连接
        libssh2_agent_disconnect(agent);
    }

    // 如果 identity_agent_path 不为空
    if(agent->identity_agent_path != NULL)
        // 释放内存
        LIBSSH2_FREE(agent->session, agent->identity_agent_path);

    // 释放身份集合
    agent_free_identities(agent);
    // 释放 agent
    LIBSSH2_FREE(agent->session, agent);
}

/*
 * libssh2_agent_set_identity_path()
 *
 * 允许设置自定义的代理套接字路径，超出 SSH_AUTH_SOCK 环境变量
 *
 */
LIBSSH2_API void
libssh2_agent_set_identity_path(LIBSSH2_AGENT *agent, const char *path)
{
    // 如果 identity_agent_path 存在
    if(agent->identity_agent_path) {
        // 释放内存
        LIBSSH2_FREE(agent->session, agent->identity_agent_path);
        agent->identity_agent_path = NULL;
    }

    // 如果 path 存在
    if(path) {
        size_t path_len = strlen(path);
        // 如果路径长度小于 SIZE_MAX - 1
        if(path_len < SIZE_MAX - 1) {
            // 分配内存并复制路径
            char *path_buf = LIBSSH2_ALLOC(agent->session, path_len + 1);
            memcpy(path_buf, path, path_len);
            path_buf[path_len] = '\0';
            agent->identity_agent_path = path_buf;
        }
    }
}

/*
 * libssh2_agent_get_identity_path()
 *
 * 如果设置了自定义代理套接字路径，则返回该路径
 *
 */
LIBSSH2_API const char *libssh2_agent_get_identity_path(LIBSSH2_AGENT *agent)
{
    return agent->identity_agent_path;
}
*/
```