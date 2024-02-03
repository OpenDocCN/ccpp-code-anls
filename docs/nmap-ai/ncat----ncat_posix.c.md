# `nmap\ncat\ncat_posix.c`

```cpp
/* $Id$ */

#include "ncat.h"

#ifdef HAVE_LUA
#include "ncat_lua.h"
#endif

// 将命令行字符串拆分成参数数组
char **cmdline_split(const char *cmdexec);

/* fork and exec a child process with netexec. Close the given file descriptor
   in the parent process. Return the child's PID or -1 on error. */
int netrun(struct fdinfo *info, char *cmdexec)
{
    int pid;

    errno = 0;
    // 创建子进程
    pid = fork();
    if (pid == 0) {
        /* In the child process. */
        // 在子进程中执行命令
        netexec(info, cmdexec);
    }

    // 关闭父进程中的文件描述符
    Close(info->fd);

    if (pid == -1 && o.verbose)
        // 如果出错且设置了详细输出，则记录错误信息
        logdebug("Error in fork: %s\n", strerror(errno));

    return pid;
}

/* Call write in a loop until all the data is written or an error occurs. The
   return value is the number of bytes written. If it is less than size, then
   there was an error. */
// 循环调用 write 函数，直到所有数据被写入或发生错误
static int write_loop(int fd, char *buf, size_t size)
{
    char *p;
    int n;

    p = buf;
    while (p - buf < size) {
        // 写入数据
        n = write(fd, p, size - (p - buf));
        if (n == -1) {
            if (errno == EINTR)
                continue;
            else
                break;
        }
        p += n;
    }

    return p - buf;
}

/* Run the given command line as if with exec. What we actually do is fork the
   command line as a subprocess, then loop, relaying data between the socket and
   the subprocess. This allows Ncat to handle SSL from the socket and give plain
   text to the subprocess, and also allows things like logging and line delays.
   Never returns. */
// 以类似 exec 的方式运行给定的命令行
// 实际上是将命令行作为子进程 fork，然后循环在套接字和子进程之间中继数据
// 这允许 Ncat 处理来自套接字的 SSL 并将纯文本传递给子进程，还允许记录和行延迟等功能
// 该函数永远不会返回
void netexec(struct fdinfo *info, char *cmdexec)
{
    int child_stdin[2];
    int child_stdout[2];
    int pid;
    int crlf_state;

    char buf[DEFAULT_TCP_BUF_LEN];
    int maxfd;

    if (o.debug) {
        switch (o.execmode) {
        case EXEC_SHELL:
            // 如果设置了调试模式并且执行模式为 shell，则记录执行信息
            logdebug("Executing with shell: %s\n", cmdexec);
            break;
#ifdef HAVE_LUA
        case EXEC_LUA:
            // 如果设置了调试模式并且执行模式为 Lua，则记录执行信息
            logdebug("Executing as lua script: %s\n", cmdexec);
            break;
#endif
        default:
            logdebug("Executing: %s\n", cmdexec);
            break;
        }
    }

    if (pipe(child_stdin) == -1 || pipe(child_stdout) == -1)
        bye("Can't create child pipes: %s", strerror(errno));

    pid = fork();
    if (pid == -1)
        bye("Error in fork: %s", strerror(errno));
    if (pid == 0) {
        /* This is the child process. Exec the command. */
        close(child_stdin[1]);
        close(child_stdout[0]);

        /* We might have turned off SIGPIPE handling in ncat_listen.c. Since
           the child process SIGPIPE might mean that the connection got broken,
           ignoring it could result in an infinite loop if the code here
           ignores the error codes of read()/write() calls. So, just in case,
           let's restore SIGPIPE so that writing to a broken pipe results in
           killing the child process. */
        Signal(SIGPIPE, SIG_DFL);

        /* rearrange stdin and stdout */
        Dup2(child_stdin[0], STDIN_FILENO);
        Dup2(child_stdout[1], STDOUT_FILENO);

        setup_environment(info);

        switch (o.execmode) {
        char **cmdargs;

        case EXEC_SHELL:
            execl("/bin/sh", "sh", "-c", cmdexec, (void *) NULL);
            break;
#ifdef HAVE_LUA
        case EXEC_LUA:
            lua_run();
            break;
#endif
        default:
            cmdargs = cmdline_split(cmdexec);
            execv(cmdargs[0], cmdargs);
            break;
        }

        /* exec failed. */
        die("exec");
    }

    close(child_stdin[0]);
    close(child_stdout[1]);

    maxfd = child_stdout[0];
    if (info->fd > maxfd)
        maxfd = info->fd;

    /* This is the parent process. Enter a "caretaker" loop that reads from the
       socket and writes to the subprocess, and reads from the subprocess and
       writes to the socket. We exit the loop on any read error (or EOF). On a
       write error we just close the opposite side of the conversation. */
    # 初始化回车换行状态为0
    crlf_state = 0;
    # 无限循环
    for (;;) {
        # 定义文件描述符集合
        fd_set fds;
        # 定义变量r和n_r
        int r, n_r;

        # 清空文件描述符集合
        FD_ZERO(&fds);
        # 将info->fd添加到文件描述符集合中
        checked_fd_set(info->fd, &fds);
        # 将child_stdout[0]添加到文件描述符集合中
        checked_fd_set(child_stdout[0], &fds);

        # 使用fselect函数等待文件描述符集合中的文件就绪
        r = fselect(maxfd + 1, &fds, NULL, NULL, NULL);
        # 如果fselect返回-1，处理错误情况
        if (r == -1) {
            # 如果是中断错误，继续循环
            if (errno == EINTR)
                continue;
            # 否则跳出循环
            else
                break;
        }
        # 如果info->fd在文件描述符集合中就绪
        if (checked_fd_isset(info->fd, &fds)) {
            # 定义变量pending
            int pending;

            # 循环接收数据，直到没有数据可接收
            do {
                # 调用ncat_recv函数接收数据
                n_r = ncat_recv(info, buf, sizeof(buf), &pending);
                # 如果接收到的数据小于等于0
                if (n_r <= 0) {
                    # 如果返回值为0且没有错误，继续循环
                    if(n_r == 0 && info->lasterr == 0) {
                        continue; # 检查是否有待处理数据
                    }
                    # 跳转到循环结束处
                    goto loop_end;
                }
                # 写入数据到child_stdin[1]中
                r = write_loop(child_stdin[1], buf, n_r);
                # 如果写入的数据不完整，跳转到循环结束处
                if (r != n_r)
                  goto loop_end;
            } while (pending);
        }
        # 如果child_stdout[0]在文件描述符集合中就绪
        if (checked_fd_isset(child_stdout[0], &fds)) {
            # 定义变量crlf和wbuf
            char *crlf = NULL, *wbuf;
            # 从child_stdout[0]中读取数据
            n_r = read(child_stdout[0], buf, sizeof(buf));
            # 如果读取到的数据小于等于0，跳出循环
            if (n_r <= 0)
                break;
            # 将wbuf指向buf
            wbuf = buf;
            # 如果o.crlf为真
            if (o.crlf) {
                # 如果修复行结束符成功，更新wbuf指向crlf
                if (fix_line_endings((char *) buf, &n_r, &crlf, &crlf_state))
                    wbuf = crlf;
            }
            # 发送数据到info中
            r = ncat_send(info, wbuf, n_r);
            # 如果crlf不为空，释放内存
            if (crlf != NULL)
                free(crlf);
            # 如果发送数据不完整，跳转到循环结束处
            if (r <= 0)
                goto loop_end;
        }
    }
loop_end:

#ifdef HAVE_OPENSSL
    // 如果支持 OpenSSL，则关闭 SSL 连接并释放资源
    if (info->ssl != NULL) {
        SSL_shutdown(info->ssl);
        SSL_free(info->ssl);
    }
#endif
    // 关闭文件描述符
    close(info->fd);

    // 退出程序
    exit(0);
}

/*
 * 将命令行拆分成适合传递给 execv 的数组
 *
 * 语法说明：单词以空格和 '\' 转义字符分割。'\\' 将显示为 '\'，'\ ' 将留下一个空格，合并两个单词。例如：
 * "ncat\ experiment -l -k" 将被解析为以下标记："ncat experiment", "-l", "-k"。
 * "ncat\\ -l -k" 将被解析为 "ncat\", "-l", "-k"
 * 请参阅测试程序 test/test-cmdline-split 以查看其他情况。
 */
char **cmdline_split(const char *cmdexec)
{
    const char *ptr;
    char *cur_arg, **cmd_args;
    int max_tokens = 0, arg_idx = 0, ptr_idx = 0;

    /* 计算所需的最大标记数 */
    ptr = cmdexec;
    while (*ptr) {
        // 找到标记的起始位置
        while (('\0' != *ptr) && isspace((int) (unsigned char) *ptr))
            ptr++;
        if ('\0' == *ptr)
            break;
        max_tokens++;
        // 再次找到空白的起始位置
        while (('\0' != *ptr) && !isspace((int) (unsigned char) *ptr))
            ptr++;
    }

    /* 行不为空，所以有东西要处理 */
    cmd_args = (char **) safe_malloc(sizeof(char *) * (max_tokens + 1));
    cur_arg = (char *) Calloc(sizeof(char), strlen(cmdexec) + 1);

    /* 获取并复制标记 */
    ptr = cmdexec;
    # 当指针指向的字符不是空字符时，进入循环
    while (*ptr) {
        # 跳过空白字符
        while (('\0' != *ptr) && isspace((int) (unsigned char) *ptr))
            ptr++;
        # 如果指针指向空字符，则跳出循环
        if ('\0' == *ptr)
            break;

        # 当指针指向的字符不是空字符时，进入循环
        while (('\0' != *ptr) && !isspace((int) (unsigned char) *ptr)) {
            # 如果指针指向的字符是反斜杠
            if ('\\' == *ptr) {
                # 移动指针到下一个字符
                ptr++;
                # 如果指针指向空字符，则跳出循环
                if ('\0' == *ptr)
                    break;

                # 将当前字符存入当前参数数组中
                cur_arg[ptr_idx] = *ptr;
                # 更新当前参数数组的索引
                ptr_idx++;
                # 移动指针到下一个字符
                ptr++;

                # 如果前一个字符不是反斜杠
                if ('\\' != *(ptr - 1)) {
                    # 跳过空白字符
                    while (('\0' != *ptr) && isspace((int) (unsigned char) *ptr))
                        ptr++;
                }
            } else {
                # 将当前字符存入当前参数数组中
                cur_arg[ptr_idx] = *ptr;
                # 更新当前参数数组的索引
                ptr_idx++;
                # 移动指针到下一个字符
                ptr++;
            }
        }
        # 在当前参数数组的末尾添加空字符
        cur_arg[ptr_idx] = '\0';

        # 将当前参数数组的副本存入命令参数数组中
        cmd_args[arg_idx] = strdup(cur_arg);
        # 清空当前参数数组
        cur_arg[0] = '\0';
        # 重置当前参数数组的索引
        ptr_idx = 0;
        # 更新命令参数数组的索引
        arg_idx++;
    }

    # 在命令参数数组的末尾添加空指针
    cmd_args[arg_idx] = NULL;

    # 释放当前参数数组的内存
    /* Clean up */
    free(cur_arg);

    # 返回命令参数数组
    return cmd_args;
# 关闭大括号，结束函数或代码块
}

# 打开指定日志文件，并返回文件描述符
int ncat_openlog(const char *logfile, int append)
{
    # 如果需要追加内容，则以追加模式打开文件
    if (append)
        return Open(logfile, O_WRONLY | O_CREAT | O_APPEND, 0664);
    # 否则以覆盖模式打开文件
    else
        return Open(logfile, O_WRONLY | O_CREAT | O_TRUNC, 0664);
}

# 设置换行模式
void set_lf_mode(void)
{
    # 无需执行任何操作
    /* Nothing needed. */
}

# 如果支持 OpenSSL
#ifdef HAVE_OPENSSL

# 默认的 CA 证书路径
#define NCAT_CA_CERTS_PATH (NCAT_DATADIR "/" NCAT_CA_CERTS_FILE)

# 加载默认的 CA 证书
int ssl_load_default_ca_certs(SSL_CTX *ctx)
{
    int rc;

    # 如果开启了调试模式，则记录使用系统默认的受信任 CA 证书和 NCAT_CA_CERTS_PATH 中的证书
    if (o.debug)
        logdebug("Using system default trusted CA certificates and those in %s.\n", NCAT_CA_CERTS_PATH);

    # 加载系统默认的 CA 证书路径
    rc = SSL_CTX_set_default_verify_paths(ctx);
    ncat_assert(rc > 0);

    # 加载我们提供的受信任证书
    rc = SSL_CTX_load_verify_locations(ctx, NCAT_CA_CERTS_PATH, NULL);
    # 如果加载失败，则记录错误信息并返回 -1
    if (rc != 1) {
        if (o.debug)
            logdebug("Unable to load trusted CA certificates from %s: %s\n",
                NCAT_CA_CERTS_PATH, ERR_error_string(ERR_get_error(), NULL));
        return -1;
    }

    # 加载成功，返回 0
    return 0;
}
#endif

# 设置环境变量，兼容不同系统
int setenv_portable(const char *name, const char *value)
{
    return setenv(name, value, 1);
}
```