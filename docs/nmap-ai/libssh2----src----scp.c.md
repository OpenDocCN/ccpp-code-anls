# `nmap\libssh2\src\scp.c`

```
/*
 * 作者声明和版权信息
 * 该部分声明了代码的版权信息和使用许可
 * 保留了作者和贡献者的权利声明
 * 规定了源代码和二进制形式的再分发条件
 * 限制了使用者对该软件的背书和推广
 * 声明了对软件的免责条款
 */

#include "libssh2_priv.h"
#include <errno.h>
#include <stdlib.h>

#include "channel.h"
#include "session.h"

/* libssh2_shell_quotedsize() 函数的宏定义，用于计算经过 libssh2_shell_quotearg() 处理后的引用字符串的最大长度 */
#define _libssh2_shell_quotedsize(s)     (3 * strlen(s) + 2)
*/

static unsigned
# 将路径中的特殊字符进行转义处理，将处理后的结果存储到缓冲区中
shell_quotearg(const char *path, unsigned char *buf,
               unsigned bufsize)
{
    const char *src;  # 原始路径字符串的指针
    unsigned char *dst, *endp;  # 目标缓冲区的指针，以及缓冲区的末尾指针

    """
    处理状态:
    UQSTRING: 未引用的字符串: ... -- 用于引用感叹号。这是初始状态
    SQSTRING: 单引号字符串: '... -- 任何字符都可以跟随
    QSTRING: 引用字符串: "... -- 只有撇号可以跟随
    """
    enum { UQSTRING, SQSTRING, QSTRING } state = UQSTRING;  # 定义处理状态，默认为未引用的字符串

    endp = &buf[bufsize];  # 缓冲区的末尾指针
    src = path;  # 原始路径字符串的指针
    dst = buf;  # 目标缓冲区的指针

    }

    switch(state) {  # 根据处理状态进行不同的处理
    case UQSTRING:  # 未引用的字符串
        break;
    case QSTRING:           /* Close quoted string */  # 引用字符串
        if(dst + 1 >= endp)  # 如果目标缓冲区剩余空间不足1个字节
            return 0;  # 返回错误
        *dst++ = '"';  # 在目标缓冲区中添加双引号
        break;
    case SQSTRING:          /* Close single quoted string */  # 单引号字符串
        if(dst + 1 >= endp)  # 如果目标缓冲区剩余空间不足1个字节
            return 0;  # 返回错误
        *dst++ = '\'';  # 在目标缓冲区中添加单引号
        break;
    default:  # 默认情况
        break;
    }

    if(dst + 1 >= endp)  # 如果目标缓冲区剩余空间不足1个字节
        return 0;  # 返回错误
    *dst = '\0';  # 在目标缓冲区末尾添加字符串结束符

    /* 结果不能大于3 * strlen(path) + 2 */
    /* assert((dst - buf) <= (3 * (src - path) + 2)); */

    return dst - buf;  # 返回处理后的字符串长度
}

/*
 * scp_recv
 *
 * 打开一个通道并通过SCP请求远程文件
 *
 */
static LIBSSH2_CHANNEL *
scp_recv(LIBSSH2_SESSION * session, const char *path, libssh2_struct_stat * sb)
{
    int cmd_len;  # 命令长度
    int rc;  # 返回值
    int tmp_err_code;  # 临时错误代码
    const char *tmp_err_msg;  # 临时错误消息
    # 如果会话的scpRecv_state为libssh2_NB_state_idle，则执行以下操作
    if(session->scpRecv_state == libssh2_NB_state_idle) {
        # 设置会话的scpRecv_mode、scpRecv_size、scpRecv_mtime、scpRecv_atime为0
        session->scpRecv_mode = 0;
        session->scpRecv_size = 0;
        session->scpRecv_mtime = 0;
        session->scpRecv_atime = 0;

        # 计算scp命令的长度
        session->scpRecv_command_len =
            _libssh2_shell_quotedsize(path) + sizeof("scp -f ") + (sb?1:0);

        # 为scp命令分配内存
        session->scpRecv_command =
            LIBSSH2_ALLOC(session, session->scpRecv_command_len);

        # 如果内存分配失败，则返回NULL
        if(!session->scpRecv_command) {
            _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                           "Unable to allocate a command buffer for "
                           "SCP session");
            return NULL;
        }

        # 格式化scp命令
        snprintf((char *)session->scpRecv_command,
                 session->scpRecv_command_len,
                 "scp -%sf ", sb?"p":"");

        # 计算命令长度
        cmd_len = strlen((char *)session->scpRecv_command);
        cmd_len += shell_quotearg(path,
                                  &session->scpRecv_command[cmd_len],
                                  session->scpRecv_command_len - cmd_len);

        # 命令不应该以NUL结尾
        session->scpRecv_command_len = cmd_len;

        # 调试信息：打开通道以接收SCP
        _libssh2_debug(session, LIBSSH2_TRACE_SCP,
                       "Opening channel for SCP receive");

        # 设置会话的scpRecv_state为libssh2_NB_state_created
        session->scpRecv_state = libssh2_NB_state_created;
    }
    # 如果会话的接收状态为已创建
    if(session->scpRecv_state == libssh2_NB_state_created) {
        # 分配一个通道
        session->scpRecv_channel =
            _libssh2_channel_open(session, "session",
                                  sizeof("session") - 1,
                                  LIBSSH2_CHANNEL_WINDOW_DEFAULT,
                                  LIBSSH2_CHANNEL_PACKET_DEFAULT, NULL,
                                  0);
        # 如果通道未能成功创建
        if(!session->scpRecv_channel) {
            # 如果最近的会话错误码不是再试一次错误
            if(libssh2_session_last_errno(session) !=
                LIBSSH2_ERROR_EAGAIN) {
                # 释放会话的接收命令
                LIBSSH2_FREE(session, session->scpRecv_command);
                session->scpRecv_command = NULL;
                # 将会话的接收状态设置为闲置
                session->scpRecv_state = libssh2_NB_state_idle;
            }
            # 如果是再试一次错误
            else {
                # 报告错误，表示启动通道时会阻塞
                _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                               "Would block starting up channel");
            }
            # 返回空值
            return NULL;
        }

        # 将会话的接收状态设置为已发送
        session->scpRecv_state = libssh2_NB_state_sent;
    }
    # 如果会话的接收状态为已发送，则执行以下代码块
    if(session->scpRecv_state == libssh2_NB_state_sent) {
        # 请求SCP获取所需的文件
        rc = _libssh2_channel_process_startup(session->scpRecv_channel, "exec",
                                              sizeof("exec") - 1,
                                              (char *)session->scpRecv_command,
                                              session->scpRecv_command_len);
        # 如果返回值为LIBSSH2_ERROR_EAGAIN，则表示需要再次调用该函数
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block requesting SCP startup");
            return NULL;
        }
        # 如果返回值不为0，则释放内存并跳转到错误处理代码块
        else if(rc) {
            LIBSSH2_FREE(session, session->scpRecv_command);
            session->scpRecv_command = NULL;
            goto scp_recv_error;
        }
        # 释放内存并将指针置为NULL
        LIBSSH2_FREE(session, session->scpRecv_command);
        session->scpRecv_command = NULL;

        # 输出SCP的初始唤醒信息
        _libssh2_debug(session, LIBSSH2_TRACE_SCP, "Sending initial wakeup");
        # 将SCP响应的第一个字符置为空
        session->scpRecv_response[0] = '\0';

        # 将接收状态置为已发送1
        session->scpRecv_state = libssh2_NB_state_sent1;
    }

    # 如果会话的接收状态为已发送1，则执行以下代码块
    if(session->scpRecv_state == libssh2_NB_state_sent1) {
        # 向SCP通道写入初始唤醒信息
        rc = _libssh2_channel_write(session->scpRecv_channel, 0,
                                    session->scpRecv_response, 1);
        # 如果返回值为LIBSSH2_ERROR_EAGAIN，则表示需要再次调用该函数
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block sending initial wakeup");
            return NULL;
        }
        # 如果返回值不为1，则跳转到错误处理代码块
        else if(rc != 1) {
            goto scp_recv_error;
        }

        # 将SCP响应的长度置为0
        session->scpRecv_response_len = 0;

        # 将接收状态置为已发送2
        session->scpRecv_state = libssh2_NB_state_sent2;
    }

    # 如果会话的接收状态为已发送4，则执行以下代码块
    if(session->scpRecv_state == libssh2_NB_state_sent4) {
        # 将SCP响应的长度置为0
        session->scpRecv_response_len = 0;

        # 将接收状态置为已发送5
        session->scpRecv_state = libssh2_NB_state_sent5;
    }
    # 如果传入的 sb 不为空，则将其内容清零，大小为 libssh2_struct_stat 的大小
    if(sb) {
        memset(sb, 0, sizeof(libssh2_struct_stat));

        # 将传入的时间戳赋值给 sb 的 st_mtime
        sb->st_mtime = session->scpRecv_mtime;
        # 将传入的访问时间赋值给 sb 的 st_atime
        sb->st_atime = session->scpRecv_atime;
        # 将传入的文件大小赋值给 sb 的 st_size
        sb->st_size = session->scpRecv_size;
        # 将传入的文件权限赋值给 sb 的 st_mode
        sb->st_mode = (unsigned short)session->scpRecv_mode;
    }

    # 将 session 的 scpRecv_state 设置为 libssh2_NB_state_idle
    session->scpRecv_state = libssh2_NB_state_idle;
    # 返回 session 的 scpRecv_channel
    return session->scpRecv_channel;

  # 如果 scp_recv_empty_channel 标签被调用，则执行以下代码
  scp_recv_empty_channel:
    # 代码只有在从 channel_read() 中得到零读取时才会跳转到这里，因此我们检查 EOF 状态以避免陷入循环
    if(libssh2_channel_eof(session->scpRecv_channel))
        # 如果通道已关闭，则抛出异常
        _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                       "Unexpected channel close");
    else
        # 否则返回 session 的 scpRecv_channel
        return session->scpRecv_channel;
    # 继续执行下面的代码
    /* fall-through */
  # 如果 scp_recv_error 标签被调用，则执行以下代码
  scp_recv_error:
    # 保存临时的错误代码和错误消息
    tmp_err_code = session->err_code;
    tmp_err_msg = session->err_msg;
    # 释放 scpRecv_channel 直到不再返回 LIBSSH2_ERROR_EAGAIN
    while(libssh2_channel_free(session->scpRecv_channel) ==
           LIBSSH2_ERROR_EAGAIN);
    # 恢复错误代码和错误消息
    session->err_code = tmp_err_code;
    session->err_msg = tmp_err_msg;
    # 将 scpRecv_channel 和 scpRecv_state 设置为 NULL 和 libssh2_NB_state_idle
    session->scpRecv_channel = NULL;
    session->scpRecv_state = libssh2_NB_state_idle;
    # 返回 NULL
    return NULL;
}

/*
 * libssh2_scp_recv
 *
 * DEPRECATED
 *
 * Open a channel and request a remote file via SCP.  This receives files
 * larger than 2 GB, but is unable to report the proper size on platforms
 * where the st_size member of struct stat is limited to 2 GB (e.g. windows).
 *
 */
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_scp_recv(LIBSSH2_SESSION *session, const char *path, struct stat * sb)
{
    LIBSSH2_CHANNEL *ptr;

    /* scp_recv uses libssh2_struct_stat, so pass one if the caller gave us a
       struct to populate... */
    libssh2_struct_stat sb_intl;
    libssh2_struct_stat *sb_ptr;
    memset(&sb_intl, 0, sizeof(sb_intl));
    sb_ptr = sb ? &sb_intl : NULL;

    BLOCK_ADJUST_ERRNO(ptr, session, scp_recv(session, path, sb_ptr));

    /* ...and populate the caller's with as much info as fits. */
    if(sb) {
        memset(sb, 0, sizeof(struct stat));

        sb->st_mtime = sb_intl.st_mtime;  // 设置文件的修改时间
        sb->st_atime = sb_intl.st_atime;  // 设置文件的访问时间
        sb->st_size = (off_t)sb_intl.st_size;  // 设置文件的大小
        sb->st_mode = sb_intl.st_mode;  // 设置文件的权限
    }

    return ptr;  // 返回通道指针
}

/*
 * libssh2_scp_recv2
 *
 * Open a channel and request a remote file via SCP.  This supports files > 2GB
 * on platforms that support it.
 *
 */
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_scp_recv2(LIBSSH2_SESSION *session, const char *path,
                  libssh2_struct_stat *sb)
{
    LIBSSH2_CHANNEL *ptr;
    BLOCK_ADJUST_ERRNO(ptr, session, scp_recv(session, path, sb));  // 调整错误码并接收远程文件
    return ptr;  // 返回通道指针
}

/*
 * scp_send()
 *
 * Send a file using SCP
 *
 */
static LIBSSH2_CHANNEL *
scp_send(LIBSSH2_SESSION * session, const char *path, int mode,
         libssh2_int64_t size, time_t mtime, time_t atime)
{
    int cmd_len;
    int rc;
    int tmp_err_code;
    const char *tmp_err_msg;
    # 如果会话的scpSend_state为libssh2_NB_state_idle，则执行以下操作
    if(session->scpSend_state == libssh2_NB_state_idle) {
        # 计算发送到远程主机的scp命令的长度
        session->scpSend_command_len =
            _libssh2_shell_quotedsize(path) + sizeof("scp -t ") +
            ((mtime || atime)?1:0);

        # 为发送到远程主机的scp命令分配内存空间
        session->scpSend_command =
            LIBSSH2_ALLOC(session, session->scpSend_command_len);

        # 如果内存分配失败，则返回NULL并报错
        if(!session->scpSend_command) {
            _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                           "Unable to allocate a command buffer for "
                           "SCP session");
            return NULL;
        }

        # 格式化scp命令字符串
        snprintf((char *)session->scpSend_command,
                 session->scpSend_command_len,
                 "scp -%st ", (mtime || atime)?"p":"");

        # 计算命令字符串的长度
        cmd_len = strlen((char *)session->scpSend_command);
        cmd_len += shell_quotearg(path,
                                  &session->scpSend_command[cmd_len],
                                  session->scpSend_command_len - cmd_len);

        # 命令字符串的长度不应该包括NUL终止符
        session->scpSend_command_len = cmd_len;

        # 调试信息：为SCP发送打开通道
        _libssh2_debug(session, LIBSSH2_TRACE_SCP,
                       "Opening channel for SCP send");
        # 分配一个通道
        session->scpSend_state = libssh2_NB_state_created;
    }
    # 如果发送状态为已创建
    if(session->scpSend_state == libssh2_NB_state_created) {
        # 打开一个新的通道，用于发送数据
        session->scpSend_channel =
            _libssh2_channel_open(session, "session", sizeof("session") - 1,
                                  LIBSSH2_CHANNEL_WINDOW_DEFAULT,
                                  LIBSSH2_CHANNEL_PACKET_DEFAULT, NULL, 0);
        # 如果通道打开失败
        if(!session->scpSend_channel) {
            # 如果错误码不是 LIBSSH2_ERROR_EAGAIN
            if(libssh2_session_last_errno(session) != LIBSSH2_ERROR_EAGAIN) {
                # 之前的调用设置了 libssh2_session_last_error()，传递它
                LIBSSH2_FREE(session, session->scpSend_command);
                session->scpSend_command = NULL;
                session->scpSend_state = libssh2_NB_state_idle;
            }
            # 如果错误码是 LIBSSH2_ERROR_EAGAIN
            else {
                # 报告错误，表示需要阻塞来启动通道
                _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                               "Would block starting up channel");
            }
            # 返回空指针
            return NULL;
        }

        # 设置发送状态为已发送
        session->scpSend_state = libssh2_NB_state_sent;
    }
    # 如果会话的SCP发送状态为已发送
    if(session->scpSend_state == libssh2_NB_state_sent) {
        # 请求SCP发送所需的文件
        rc = _libssh2_channel_process_startup(session->scpSend_channel, "exec",
                                              sizeof("exec") - 1,
                                              (char *)session->scpSend_command,
                                              session->scpSend_command_len);
        # 如果返回值为LIBSSH2_ERROR_EAGAIN，表示需要再次调用才能完成请求
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block requesting SCP startup");
            return NULL;
        }
        # 如果返回值不为0，表示出现错误
        else if(rc) {
            # 前一个调用设置了libssh2_session_last_error()，将其传递
            LIBSSH2_FREE(session, session->scpSend_command);
            session->scpSend_command = NULL;
            _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                           "Unknown error while getting error string");
            # 跳转到错误处理标签
            goto scp_send_error;
        }
        # 释放SCP发送命令的内存
        LIBSSH2_FREE(session, session->scpSend_command);
        session->scpSend_command = NULL;

        # 设置SCP发送状态为已发送1
        session->scpSend_state = libssh2_NB_state_sent1;
    }
    # 如果会话的scpSend_state为libssh2_NB_state_sent1，则等待ACK
    if(session->scpSend_state == libssh2_NB_state_sent1) {
        # 从scpSend_channel通道读取1个字节的数据到scpSend_response中
        rc = _libssh2_channel_read(session->scpSend_channel, 0,
                                   (char *) session->scpSend_response, 1);
        # 如果返回值为LIBSSH2_ERROR_EAGAIN，则表示需要等待远程响应
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block waiting for response from remote");
            return NULL;
        }
        # 如果返回值小于0，则表示SCP失败
        else if(rc < 0) {
            _libssh2_error(session, rc, "SCP failure");
            goto scp_send_error;
        }
        # 如果返回值为0，则保持在相同的状态
        else if(!rc)
            goto scp_send_empty_channel;
        # 如果收到的响应不为0，则表示远程响应无效
        else if(session->scpSend_response[0] != 0) {
            _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                           "Invalid ACK response from remote");
            goto scp_send_error;
        }
        # 如果mtime或atime不为0，则发送mtime和atime用于文件
        if(mtime || atime) {
            # 将mtime和atime格式化成字符串，存储到scpSend_response中
            session->scpSend_response_len =
                snprintf((char *) session->scpSend_response,
                         LIBSSH2_SCP_RESPONSE_BUFLEN, "T%ld 0 %ld 0\n",
                         (long)mtime, (long)atime);
            # 调试信息，打印已发送的内容
            _libssh2_debug(session, LIBSSH2_TRACE_SCP, "Sent %s",
                           session->scpSend_response);
        }

        # 设置scpSend_state为libssh2_NB_state_sent2
        session->scpSend_state = libssh2_NB_state_sent2;
    }

    # 发送mtime和atime用于文件
    # 如果修改时间或访问时间存在
    if(mtime || atime) {
        # 如果会话的SCP发送状态为libssh2_NB_state_sent2
        if(session->scpSend_state == libssh2_NB_state_sent2) {
            # 写入SCP发送通道的时间数据
            rc = _libssh2_channel_write(session->scpSend_channel, 0,
                                        session->scpSend_response,
                                        session->scpSend_response_len);
            # 如果返回值为LIBSSH2_ERROR_EAGAIN，表示需要再次尝试发送
            if(rc == LIBSSH2_ERROR_EAGAIN) {
                _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                               "Would block sending time data for SCP file");
                return NULL;
            }
            # 如果返回值不等于session->scpSend_response_len，表示发送时间数据失败
            else if(rc != (int)session->scpSend_response_len) {
                _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                               "Unable to send time data for SCP file");
                # 跳转到scp_send_error标签处
                goto scp_send_error;
            }

            # 将会话的SCP发送状态设置为libssh2_NB_state_sent3
            session->scpSend_state = libssh2_NB_state_sent3;
        }

        # 如果会话的SCP发送状态为libssh2_NB_state_sent3
        if(session->scpSend_state == libssh2_NB_state_sent3) {
            # 等待ACK响应
            rc = _libssh2_channel_read(session->scpSend_channel, 0,
                                       (char *) session->scpSend_response, 1);
            # 如果返回值为LIBSSH2_ERROR_EAGAIN，表示需要再次尝试等待响应
            if(rc == LIBSSH2_ERROR_EAGAIN) {
                _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                               "Would block waiting for response");
                return NULL;
            }
            # 如果返回值小于0，表示SCP失败
            else if(rc < 0) {
                _libssh2_error(session, rc, "SCP failure");
                # 跳转到scp_send_error标签处
                goto scp_send_error;
            }
            # 如果返回值为0，表示保持在相同状态
            else if(!rc)
                # 跳转到scp_send_empty_channel标签处
                goto scp_send_empty_channel;
            # 如果响应不为0，表示收到无效的SCP ACK响应
            else if(session->scpSend_response[0] != 0) {
                _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                               "Invalid SCP ACK response");
                # 跳转到scp_send_error标签处
                goto scp_send_error;
            }

            # 将会话的SCP发送状态设置为libssh2_NB_state_sent4
            session->scpSend_state = libssh2_NB_state_sent4;
        }
    }
    else {
        // 如果会话的发送状态为libssh2_NB_state_sent2，则将其设置为libssh2_NB_state_sent4
        if(session->scpSend_state == libssh2_NB_state_sent2) {
            session->scpSend_state = libssh2_NB_state_sent4;
        }
    }

    // 如果会话的发送状态为libssh2_NB_state_sent4
    if(session->scpSend_state == libssh2_NB_state_sent4) {
        /* 发送模式、大小和基本名称 */
        const char *base = strrchr(path, '/');  // 获取路径中最后一个'/'之后的字符串
        if(base)
            base++;
        else
            base = path;

        // 格式化发送响应字符串
        session->scpSend_response_len =
            snprintf((char *) session->scpSend_response,
                     LIBSSH2_SCP_RESPONSE_BUFLEN, "C0%o %"
                     LIBSSH2_INT64_T_FORMAT " %s\n", mode,
                     size, base);
        _libssh2_debug(session, LIBSSH2_TRACE_SCP, "Sent %s",
                       session->scpSend_response);

        session->scpSend_state = libssh2_NB_state_sent5;
    }

    // 如果会话的发送状态为libssh2_NB_state_sent5
    if(session->scpSend_state == libssh2_NB_state_sent5) {
        // 将数据写入通道
        rc = _libssh2_channel_write(session->scpSend_channel, 0,
                                    session->scpSend_response,
                                    session->scpSend_response_len);
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block send core file data for SCP file");
            return NULL;
        }
        else if(rc != (int)session->scpSend_response_len) {
            _libssh2_error(session, LIBSSH2_ERROR_SOCKET_SEND,
                           "Unable to send core file data for SCP file");
            goto scp_send_error;
        }

        session->scpSend_state = libssh2_NB_state_sent6;
    }
    if(session->scpSend_state == libssh2_NB_state_sent6) {
        /* 检查是否已经发送了数据，等待对方的确认 */
        rc = _libssh2_channel_read(session->scpSend_channel, 0,
                                   (char *) session->scpSend_response, 1);
        /* 如果返回值是 LIBSSH2_ERROR_EAGAIN，表示需要等待对方的响应 */
        if(rc == LIBSSH2_ERROR_EAGAIN) {
            _libssh2_error(session, LIBSSH2_ERROR_EAGAIN,
                           "Would block waiting for response");
            return NULL;
        }
        /* 如果返回值小于 0，表示收到了无效的确认响应 */
        else if(rc < 0) {
            _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                           "Invalid ACK response from remote");
            goto scp_send_error;
        }
        /* 如果返回值为 0，表示通道为空 */
        else if(rc == 0)
            goto scp_send_empty_channel;

        /* 如果收到了非零的确认响应 */
        else if(session->scpSend_response[0] != 0) {
            size_t err_len;
            char *err_msg;

            /* 获取远程错误消息的长度 */
            err_len =
                _libssh2_channel_packet_data_len(session->scpSend_channel, 0);
            /* 分配内存来存储错误消息 */
            err_msg = LIBSSH2_ALLOC(session, err_len + 1);
            if(!err_msg) {
                _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                               "failed to get memory");
                goto scp_send_error;
            }

            /* 读取远程错误消息 */
            rc = _libssh2_channel_read(session->scpSend_channel, 0,
                                       err_msg, err_len);
            if(rc > 0) {
                err_msg[err_len] = 0;
                _libssh2_debug(session, LIBSSH2_TRACE_SCP,
                               "got %02x %s", session->scpSend_response[0],
                               err_msg);
            }
            LIBSSH2_FREE(session, err_msg);
            _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                           "failed to send file");
            goto scp_send_error;
        }
    }

    /* 将状态设置为闲置状态，并返回通道 */
    session->scpSend_state = libssh2_NB_state_idle;
    return session->scpSend_channel;

  scp_send_empty_channel:
    /* 如果从 channel_read() 函数读取到零字节，则跳转到这里，因此我们检查 EOF 状态以避免陷入循环 */
    if(libssh2_channel_eof(session->scpSend_channel)) {
        _libssh2_error(session, LIBSSH2_ERROR_SCP_PROTOCOL,
                       "Unexpected channel close");
    }
    else
        return session->scpSend_channel;
    /* 继续执行 */
  scp_send_error:
    tmp_err_code = session->err_code;
    tmp_err_msg = session->err_msg;
    while(libssh2_channel_free(session->scpSend_channel) ==
          LIBSSH2_ERROR_EAGAIN);
    session->err_code = tmp_err_code;
    session->err_msg = tmp_err_msg;
    session->scpSend_channel = NULL;
    session->scpSend_state = libssh2_NB_state_idle;
    return NULL;
/*
 * libssh2_scp_send_ex
 *
 * 使用 SCP 发送文件。旧的 API。
 */
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_scp_send_ex(LIBSSH2_SESSION *session, const char *path, int mode,
                    size_t size, long mtime, long atime)
{
    // 声明指针变量
    LIBSSH2_CHANNEL *ptr;
    // 调用 scp_send 函数发送文件，并将结果赋给 ptr
    BLOCK_ADJUST_ERRNO(ptr, session,
                       scp_send(session, path, mode, size,
                                (time_t)mtime, (time_t)atime));
    // 返回指针
    return ptr;
}

/*
 * libssh2_scp_send64
 *
 * 使用 SCP 发送文件
 */
LIBSSH2_API LIBSSH2_CHANNEL *
libssh2_scp_send64(LIBSSH2_SESSION *session, const char *path, int mode,
                   libssh2_int64_t size, time_t mtime, time_t atime)
{
    // 声明指针变量
    LIBSSH2_CHANNEL *ptr;
    // 调用 scp_send 函数发送文件，并将结果赋给 ptr
    BLOCK_ADJUST_ERRNO(ptr, session,
                       scp_send(session, path, mode, size, mtime, atime));
    // 返回指针
    return ptr;
}
```