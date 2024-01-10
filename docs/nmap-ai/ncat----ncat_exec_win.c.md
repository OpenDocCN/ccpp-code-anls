# `nmap\ncat\ncat_exec_win.c`

```
/* $Id$ */

#include "ncat.h"

/* This structure holds information about a subprocess with redirected input
   and output handles. */
struct subprocess_info {
    HANDLE proc;
    struct fdinfo fdn;
    HANDLE child_in_r;
    HANDLE child_in_w;
    HANDLE child_out_r;
    HANDLE child_out_w;
};

/* A list of subprocesses, so we can kill them when the program exits. */
static HANDLE subprocesses[DEFAULT_MAX_CONNS];
static int subprocess_max_index = 0;
/* Prevent concurrent access to the subprocesses table by the main process and
   a thread. Protects subprocesses and subprocesses_max_index. */
static HANDLE subprocesses_mutex = NULL;

static int start_subprocess(char *cmdexec, struct subprocess_info *info);
static DWORD WINAPI subprocess_thread_func(void *data);

static int register_subprocess(HANDLE proc);
static int unregister_subprocess(HANDLE proc);
static int get_subprocess_slot(void);

/* Have we registered the termination handler yet? */
static int atexit_registered = 0;
static void terminate_subprocesses(void);
static void sigint_handler(int s);

/* This may be set with set_pseudo_sigchld_handler. It is called when a thread
   representing a child process ends. */
static void (*pseudo_sigchld_handler)(void) = NULL;
/* Simulates blocking of SIGCHLD while the handler runs. Also prevents
   concurrent modification of pseudo_sigchld_handler. */
static HANDLE pseudo_sigchld_mutex = NULL;

/* Run a child process, redirecting its standard file handles to a socket
   descriptor. Return the child's PID or -1 on error. */
int netrun(struct fdinfo *fdn, char *cmdexec)
{
    struct subprocess_info *info;
    HANDLE thread;
    int pid;

    info = (struct subprocess_info *) safe_malloc(sizeof(*info));
    info->fdn = *fdn;

    pid = start_subprocess(cmdexec, info);
    if (pid == -1) {
        close(info->fdn.fd);
        free(info);
        return -1;
    }

    /* Start up the thread to handle process I/O. */
    # 创建一个新线程，使用指定的函数和参数
    thread = CreateThread(NULL, 0, subprocess_thread_func, info, 0, NULL);
    # 如果线程创建失败
    if (thread == NULL) {
        # 如果设置了详细输出，记录创建线程错误信息
        if (o.verbose)
            logdebug("Error in CreateThread: %d\n", GetLastError());
        # 释放 info 内存
        free(info);
        # 返回错误代码 -1
        return -1;
    }
    # 关闭线程句柄
    CloseHandle(thread);

    # 返回新线程的进程标识符
    return pid;
}

/* Run the given command line as if by exec. Doesn't return. */
void netexec(struct fdinfo *fdn, char *cmdexec)
{
    // 分配内存以存储子进程信息
    struct subprocess_info *info;
    int pid;
    DWORD ret;

    // 分配内存以存储子进程信息
    info = (struct subprocess_info *) safe_malloc(sizeof(*info));
    info->fdn = *fdn;

    // 启动子进程
    pid = start_subprocess(cmdexec, info);
    if (pid == -1)
        ExitProcess(2);

    /* Run the subprocess thread function, but don't put it in a thread. Just
       run it and exit with its return value because we're simulating exec. */
    // 运行子进程线程函数，但不将其放入线程中，直接运行并退出，返回其返回值，因为我们在模拟 exec
    ExitProcess(subprocess_thread_func(info));
}

/* Set a pseudo-signal handler that is called when a thread representing a
   child process dies. This is only used on Windows. */
// 设置一个伪信号处理程序，当代表子进程的线程终止时调用。仅在 Windows 上使用。
extern void set_pseudo_sigchld_handler(void (*handler)(void))
{
    DWORD rc;

    // 如果伪信号处理程序互斥量为空，则创建互斥量
    if (pseudo_sigchld_mutex == NULL) {
        pseudo_sigchld_mutex = CreateMutex(NULL, FALSE, NULL);
        ncat_assert(pseudo_sigchld_mutex != NULL);
    }
    // 等待伪信号处理程序互斥量
    rc = WaitForSingleObject(pseudo_sigchld_mutex, INFINITE);
    ncat_assert(rc == WAIT_OBJECT_0);
    // 设置伪信号处理程序
    pseudo_sigchld_handler = handler;
    // 释放伪信号处理程序互斥量
    rc = ReleaseMutex(pseudo_sigchld_mutex);
    ncat_assert(rc != 0);
}

// 设置环境变量，可移植性版本
int setenv_portable(const char *name, const char *value)
{
    char *var;
    int ret;
    size_t len;
    // 计算变量的长度
    len = strlen(name) + strlen(value) + 2; /* 1 for '\0', 1 for =. */
    // 分配内存以存储变量
    var = (char *) safe_malloc(len);
    // 格式化变量
    Snprintf(var, len, "%s=%s", name, value);
    /* _putenv was chosen over SetEnvironmentVariable because variables set
       with the latter seem to be invisible to getenv() calls and Lua uses
       these in the 'os' module. */
    // 使用 _putenv 覆盖环境变量，而不是 SetEnvironmentVariable，因为后者设置的变量似乎对 getenv() 调用不可见，而 Lua 在 'os' 模块中使用这些变量。
    ret = _putenv(var) == 0;
    // 释放变量内存
    free(var);
    return ret;
}

/* Run a command and redirect its input and output handles to a pair of
   anonymous pipes.  The process handle and pipe handles are returned in the
   info struct. Returns the PID of the new process, or -1 on error. */
// 运行命令并将其输入和输出句柄重定向到一对匿名管道。进程句柄和管道句柄在 info 结构中返回。返回新进程的 PID，或在出错时返回 -1。
static int run_command_redirected(char *cmdexec, struct subprocess_info *info)
{
    /* 每个创建的命名管道都必须有一个唯一的名称。 */
    static int pipe_serial_no = 0;  // 创建命名管道的序列号
    char pipe_name[32];  // 命名管道的名称
    SECURITY_ATTRIBUTES sa;  // 安全属性
    STARTUPINFO si;  // 启动信息
    PROCESS_INFORMATION pi;  // 进程信息

    setup_environment(&info->fdn);  // 设置环境变量

    /* 使管道句柄可继承。 */
    sa.nLength = sizeof(sa);  // 设置安全属性的长度
    sa.bInheritHandle = TRUE;  // 设置句柄可继承
    sa.lpSecurityDescriptor = NULL;  // 设置安全描述符为空

    /* 子进程的输入管道是普通的阻塞管道。 */
    if (CreatePipe(&info->child_in_r, &info->child_in_w, &sa, 0) == 0) {  // 创建管道
        if (o.verbose)
            logdebug("Error in CreatePipe: %d\n", GetLastError());  // 输出错误信息
        return -1;  // 返回错误
    }

    /* 管道名称必须具有这种特殊形式。 */
    Snprintf(pipe_name, sizeof(pipe_name), "\\\\.\\pipe\\ncat-%d-%d",
        GetCurrentProcessId(), pipe_serial_no);  // 格式化管道名称
    if (o.debug > 1)
        logdebug("Creating named pipe \"%s\"\n", pipe_name);  // 输出创建的命名管道名称

    /* 输出管道必须是非阻塞的，这需要进行复杂的设置。 */
    info->child_out_r = CreateNamedPipe(pipe_name,
        PIPE_ACCESS_INBOUND | FILE_FLAG_OVERLAPPED,
        PIPE_TYPE_BYTE, 1, 4096, 4096, 1000, &sa);  // 创建命名管道
    if (info->child_out_r == 0) {  // 如果创建失败
        if (o.verbose)
            logdebug("Error in CreateNamedPipe: %d\n", GetLastError());  // 输出错误信息
        CloseHandle(info->child_in_r);  // 关闭输入管道句柄
        CloseHandle(info->child_in_w);  // 关闭输出管道句柄
        return -1;  // 返回错误
    }
    info->child_out_w = CreateFile(pipe_name,
        GENERIC_WRITE, 0, &sa, OPEN_EXISTING,
        FILE_ATTRIBUTE_NORMAL | FILE_FLAG_OVERLAPPED, NULL);  // 创建文件
    if (info->child_out_w == 0) {  // 如果创建失败
        CloseHandle(info->child_in_r);  // 关闭输入管道句柄
        CloseHandle(info->child_in_w);  // 关闭输出管道句柄
        CloseHandle(info->child_out_r);  // 关闭输出管道句柄
        return -1;  // 返回错误
    }
    pipe_serial_no++;  // 管道序列号加一

    /* 不继承管道的一端。 */
    SetHandleInformation(info->child_in_w, HANDLE_FLAG_INHERIT, 0);  // 设置输入管道句柄不可继承
    SetHandleInformation(info->child_out_r, HANDLE_FLAG_INHERIT, 0);  // 设置输出管道句柄不可继承

    memset(&si, 0, sizeof(si));  // 清空启动信息
    si.cb = sizeof(si);  // 设置启动信息的大小
    si.hStdInput = info->child_in_r;  // 设置标准输入为输入管道
    # 将标准输出重定向到子进程的输出句柄
    si.hStdOutput = info->child_out_w;
    # 将标准错误输出重定向到标准错误句柄
    si.hStdError = GetStdHandle(STD_ERROR_HANDLE);
    # 启用标准输入、输出和错误输出句柄
    si.dwFlags |= STARTF_USESTDHANDLES;

    # 将 pi 结构体清零
    memset(&pi, 0, sizeof(pi));

    # 创建子进程
    if (CreateProcess(NULL, cmdexec, NULL, NULL, TRUE, 0, NULL, NULL, &si, &pi) == 0) {
        # 如果创建进程失败，输出错误信息
        if (o.verbose) {
            LPVOID lpMsgBuf;
            FormatMessage(
            FORMAT_MESSAGE_ALLOCATE_BUFFER |
            FORMAT_MESSAGE_FROM_SYSTEM |
            FORMAT_MESSAGE_IGNORE_INSERTS,
            NULL,
            GetLastError(),
            MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT),
            (LPTSTR)&lpMsgBuf,
            0, NULL);

            logdebug("Error in CreateProcess: %s\nCommand was: %s\n", (lpMsgBuf), cmdexec);
        }
        # 关闭子进程的输入、输出句柄
        CloseHandle(info->child_in_r);
        CloseHandle(info->child_in_w);
        CloseHandle(info->child_out_r);
        CloseHandle(info->child_out_w);
        # 返回错误码
        return -1;
    }

    # 关闭子进程的线程句柄
    /* Close hThread here because we have no use for it. hProcess is closed in
       subprocess_info_close. */
    CloseHandle(pi.hThread);

    # 将子进程的进程句柄保存到 info 结构体中
    info->proc = pi.hProcess;

    # 返回子进程的进程 ID
    return pi.dwProcessId;
# 关闭子进程信息结构体，包括 SSL 连接（如果有）、套接字、进程句柄和管道句柄
static void subprocess_info_close(struct subprocess_info *info)
{
#ifdef HAVE_OPENSSL
    # 如果存在 SSL 连接，则关闭 SSL 连接并释放资源
    if (info->fdn.ssl != NULL) {
        SSL_shutdown(info->fdn.ssl);
        SSL_free(info->fdn.ssl);
    }
#endif
    # 关闭套接字
    closesocket(info->fdn.fd);
    # 关闭进程句柄和管道句柄
    CloseHandle(info->proc);
    CloseHandle(info->child_in_r);
    CloseHandle(info->child_in_w);
    CloseHandle(info->child_out_r);
    CloseHandle(info->child_out_w);
}

/* 启动子进程，使用 run_command_redirected 并注册终止处理程序。处理 o.shellexec。返回子进程的 PID，出错时返回 -1。 */
static int start_subprocess(char *cmdexec, struct subprocess_info *info)
{
    char *cmdbuf;
    int pid;

    # 如果执行模式为 EXEC_SHELL
    if (o.execmode == EXEC_SHELL) {
        /* 使用 cmd.exe 运行 */
        const char *shell;
        size_t cmdlen;

        # 获取 shell 的路径
        shell = get_shell();
        # 计算命令长度
        cmdlen = strlen(shell) + strlen(cmdexec) + 32;
        # 分配内存存储命令
        cmdbuf = (char *) safe_malloc(cmdlen);
        # 格式化命令字符串
        Snprintf(cmdbuf, cmdlen, "%s /C %s", shell, cmdexec);
#ifdef HAVE_LUA
    } else if (o.execmode == EXEC_LUA) {
        char exepath[8192];
        char *cmdexec_escaped, *exepath_escaped;
        int n;

        # 获取当前可执行文件的路径
        n = GetModuleFileName(GetModuleHandle(0), exepath, sizeof(exepath));
        # 如果获取失败，则返回 -1
        if (n == 0 || n == sizeof(exepath))
            return -1;

        # 对命令参数进行转义
        cmdexec_escaped = escape_windows_command_arg(cmdexec);
        # 如果转义失败，则返回 -1
        if (cmdexec_escaped == NULL)
            return -1;

        # 对可执行文件路径进行转义
        exepath_escaped = escape_windows_command_arg(exepath);
        # 如果转义失败，则释放内存并返回 -1
        if (exepath_escaped == NULL) {
            free(cmdexec_escaped);
            return -1;
        }

        # 格式化命令字符串
        n = asprintf(&cmdbuf, "%s --lua-exec-internal %s", exepath_escaped, cmdexec_escaped);
        free(cmdexec_escaped);
        free(exepath_escaped);
        # 如果格式化失败，则返回 -1
        if (n < 0)
            return -1;
#endif
    } else {
        // 如果 o.debug 为假，将 cmdbuf 设置为 cmdexec
        cmdbuf = cmdexec;
    }

    // 如果 o.debug 为真，记录执行的命令
    if (o.debug)
        logdebug("Executing: %s\n", cmdbuf);

    // 运行命令，并将输出重定向到 info
    pid = run_command_redirected(cmdbuf, info);

    // 如果 cmdbuf 不等于 cmdexec，释放 cmdbuf 的内存
    if (cmdbuf != cmdexec)
        free(cmdbuf);

    // 如果 pid 为 -1，返回 -1
    if (pid == -1)
        return -1;

    // 如果注册子进程失败
    if (register_subprocess(info->proc) == -1) {
        // 如果 o.verbose 为真，记录注册子进程失败的信息
        if (o.verbose)
            logdebug("Couldn't register subprocess with termination handler; not executing.\n");
        // 终止进程
        TerminateProcess(info->proc, 2);
        // 关闭子进程信息
        subprocess_info_close(info);
        // 返回 -1
        return -1;
    }

    // 返回进程 ID
    return pid;
/* 通过套接字和进程之间中继数据，直到进程终止或停止发送或接收数据。数据参数中包含套接字描述符和进程管道句柄，必须是指向struct subprocess_info的指针。

   这个函数是为了解决一个问题，即我们不能简单地运行一个进程，然后将其输入句柄重定向到套接字。如果进程，例如，重定向了自己的标准输入，它会以某种方式混淆套接字，导致标准输出停止工作。这正是ncat所做的（作为Windows标准输入的解决方法的一部分），因此不能忽视它。

   这个函数可以通过CreateThread调用来模拟fork+exec，或者直接调用来模拟exec。在返回之前，它会释放subprocess_info结构并关闭套接字和管道句柄。返回子进程的退出码。 */
static DWORD WINAPI subprocess_thread_func(void *data)
{
    struct subprocess_info *info;
    char pipe_buffer[BUFSIZ];
    OVERLAPPED overlap = { 0 };
    HANDLE events[3];
    DWORD ret, rc;
    int crlf_state = 0;

    info = (struct subprocess_info *) data;

    /* 我们要监视的三个事件：套接字读取、管道读取和进程结束。 */
    events[0] = (HANDLE) WSACreateEvent();
    WSAEventSelect(info->fdn.fd, events[0], FD_READ | FD_CLOSE);
    events[1] = info->child_out_r;
    events[2] = info->proc;

    /* 为了避免阻塞或轮询，我们在进程管道上使用异步I/O，或者微软所谓的"重叠" I/O。WaitForMultipleObjects会报告读取操作何时完成。 */
    ReadFile(info->child_out_r, pipe_buffer, sizeof(pipe_buffer), NULL, &overlap);

    /* 循环直到EOF或错误。 */
    }

loop_end:

#ifdef HAVE_OPENSSL
    if (o.ssl && info->fdn.ssl) {
        SSL_shutdown(info->fdn.ssl);
        SSL_free(info->fdn.ssl);
        /* 避免在subprocess_info_close中再次关闭和释放 */
        info->fdn.ssl = NULL;
    }
#endif

    WSACloseEvent(events[0]);
}
    # 取消注册子进程
    rc = unregister_subprocess(info->proc);
    ncat_assert(rc != -1);

    # 获取子进程的退出代码
    GetExitCodeProcess(info->proc, &ret);
    # 如果子进程仍在运行
    if (ret == STILL_ACTIVE) {
        # 如果调试级别大于1，记录调试信息
        if (o.debug > 1)
            logdebug("Subprocess still running, terminating it.\n");
        # 终止子进程
        rc = TerminateProcess(info->proc, 0);
        # 如果终止失败
        if (rc == 0) {
            # 如果调试级别大于1，记录终止失败的信息
            if (o.debug > 1)
                logdebug("TerminateProcess failed with code %d.\n", rc);
        }
    }
    # 再次获取子进程的退出代码
    GetExitCodeProcess(info->proc, &ret);
    # 如果调试级别大于1，记录子进程的退出代码
    if (o.debug > 1)
        logdebug("Subprocess ended with exit code %d.\n", ret);

    # 关闭文件描述符
    shutdown(info->fdn.fd, 2);
    # 关闭子进程信息
    subprocess_info_close(info);
    # 释放内存
    free(info);

    # 等待伪SIGCHLD互斥对象
    rc = WaitForSingleObject(pseudo_sigchld_mutex, INFINITE);
    ncat_assert(rc == WAIT_OBJECT_0);
    # 如果伪SIGCHLD处理程序不为空，调用它
    if (pseudo_sigchld_handler != NULL)
        pseudo_sigchld_handler();
    # 释放伪SIGCHLD互斥对象
    rc = ReleaseMutex(pseudo_sigchld_mutex);
    ncat_assert(rc != 0);

    # 返回子进程的退出代码
    return ret;
# 查找子进程表中的空闲槽位。更新 subprocesses_max_index 为包含非空句柄的最大索引加一。（假设此函数返回的索引将被句柄填充。）
static int get_subprocess_slot(void)
{
    int i, free_index, max_index;
    DWORD rc;

    rc = WaitForSingleObject(subprocesses_mutex, INFINITE);
    ncat_assert(rc == WAIT_OBJECT_0);

    free_index = -1;
    max_index = 0;
    for (i = 0; i < subprocess_max_index; i++) {
        HANDLE proc = subprocesses[i];

        if (proc == NULL) {
            if (free_index == -1)
                free_index = i;
        } else {
            max_index = i + 1;
        }
    }
    if ((free_index == -1 || free_index == max_index)
        && max_index < sizeof(subprocesses) / sizeof(subprocesses[0]))
        free_index = max_index++;
    subprocess_max_index = max_index;

    rc = ReleaseMutex(subprocesses_mutex);
    ncat_assert(rc != 0);

    return free_index;
}

# 将进程添加到程序退出时需要终止的进程列表中。一旦调用此函数，进程句柄“属于”它，直到调用 unregister_subprocess 之前不应修改句柄。出错时返回-1。
static int register_subprocess(HANDLE proc)
{
    int i;
    DWORD rc;

    if (subprocesses_mutex == NULL) {
        subprocesses_mutex = CreateMutex(NULL, FALSE, NULL);
        ncat_assert(subprocesses_mutex != NULL);
    }
    if (pseudo_sigchld_mutex == NULL) {
        pseudo_sigchld_mutex = CreateMutex(NULL, FALSE, NULL);
        ncat_assert(pseudo_sigchld_mutex != NULL);
    }

    rc = WaitForSingleObject(subprocesses_mutex, INFINITE);
    ncat_assert(rc == WAIT_OBJECT_0);

    i = get_subprocess_slot();
    if (i == -1) {
        if (o.verbose)
            logdebug("No free process slots for termination handler.\n");
    } else {
        // 将子进程注册到subprocesses数组中的索引i处
        subprocesses[i] = proc;

        // 如果调试级别大于1，则记录调试信息
        if (o.debug > 1)
            logdebug("Register subprocess %p at index %d.\n", proc, i);

        // 如果还没有注册atexit处理程序，则注册一个atexit和SIGINT处理程序
        if (!atexit_registered) {
            /* We register both an atexit and a SIGINT handler because ^C
               doesn't seem to cause atexit handlers to be called. */
            atexit(terminate_subprocesses);
            signal(SIGINT, sigint_handler);
            atexit_registered = 1;
        }
    }

    // 释放互斥锁
    rc = ReleaseMutex(subprocesses_mutex);
    // 断言释放互斥锁的返回值不为0
    ncat_assert(rc != 0);

    // 返回子进程在subprocesses数组中的索引
    return i;
# 从终止处理程序列表中移除一个进程句柄。如果进程尚未注册，则返回-1。
static int unregister_subprocess(HANDLE proc)
{
    int i;
    DWORD rc;

    # 等待子进程互斥锁
    rc = WaitForSingleObject(subprocesses_mutex, INFINITE);
    ncat_assert(rc == WAIT_OBJECT_0);

    # 遍历子进程列表，查找要移除的进程句柄
    for (i = 0; i < subprocess_max_index; i++) {
        if (proc == subprocesses[i])
            break;
    }
    # 如果找到要移除的进程句柄
    if (i < subprocess_max_index) {
        subprocesses[i] = NULL;
        # 如果调试级别大于1，则记录调试信息
        if (o.debug > 1)
            logdebug("Unregister subprocess %p from index %d.\n", proc, i);
    } else {
        i = -1;
    }

    # 释放子进程互斥锁
    rc = ReleaseMutex(subprocesses_mutex);
    ncat_assert(rc != 0);

    # 返回结果
    return i;
}

# 终止子进程
static void terminate_subprocesses(void)
{
    int i;
    DWORD rc;

    # 如果调试级别大于0，则记录调试信息
    if (o.debug)
        logdebug("Terminating subprocesses\n");

    # 等待子进程互斥锁
    rc = WaitForSingleObject(subprocesses_mutex, INFINITE);
    ncat_assert(rc == WAIT_OBJECT_0);

    # 如果调试级别大于1，则记录调试信息
    if (o.debug > 1)
        logdebug("max_index %d\n", subprocess_max_index);
    # 遍历子进程列表
    for (i = 0; i < subprocess_max_index; i++) {
        HANDLE proc = subprocesses[i];
        DWORD ret;

        # 如果进程句柄为空，则继续下一次循环
        if (proc == NULL)
            continue;
        # 获取进程退出码
        GetExitCodeProcess(proc, &ret);
        # 如果进程仍处于活动状态
        if (ret == STILL_ACTIVE) {
            # 如果调试级别大于1，则记录调试信息
            if (o.debug > 1)
                logdebug("kill index %d\n", i);
            # 终止进程
            TerminateProcess(proc, 0);
        }
        subprocesses[i] = NULL;
    }

    # 释放子进程互斥锁
    rc = ReleaseMutex(subprocesses_mutex);
    ncat_assert(rc != 0);
}

# SIGINT信号处理程序
static void sigint_handler(int s)
{
    # 终止子进程
    terminate_subprocesses();
    # 退出进程
    ExitProcess(0);
}
```