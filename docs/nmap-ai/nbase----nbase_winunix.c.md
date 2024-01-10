# `nmap\nbase\nbase_winunix.c`

```
/*
   $Id$ 
   代码版本标识

   包含必要的头文件
*/
#include <assert.h>
#include "nbase.h"
#include "nbase_winunix.h"

/*
   该代码使得在 Windows 上可以检查 stdin 的输入而不会阻塞。有两个障碍需要克服。
   第一个是在 Windows 上，select 仅适用于套接字，而不适用于 stdin。
   另一个是 Windows 命令 shell 除非程序主动从 stdin 读取（通常会导致阻塞），否则不会将键入的字符回显到屏幕上。

   策略是创建一个后台线程，不断从 stdin 读取。线程在读取时会阻塞，这样字符就可以被回显。
   线程将每个数据块写入一个匿名管道。我们通过操作文件描述符和 Windows 文件句柄，使程序的其余部分认为管道的另一端是 stdin。
   只有线程保留对真正的 stdin 的引用。Windows 有一个 PeekNamedPipe 函数，我们用它来检查管道中的输入而不会阻塞。

   调用 win_stdin_start_thread 来启动线程，调用 win_stdin_ready 来进行非阻塞输入检查。
   对 stdin 的任何其他操作（read、scanf 等）应该是透明的，但我注意到 eof(0) 在管道中没有任何内容时返回 1，但如果在管道中写入更多内容，它会再次返回 0。
   在启动后台线程之前缓冲但未传递给程序的任何数据可能在启动线程时丢失。
*/

/* 读取和缓冲真正的 stdin 的后台线程 */
static HANDLE stdin_thread = NULL;

/* 这是在任何重定向之前真正的 stdin 文件句柄的副本。线程读取它。 */
static HANDLE thread_stdin_handle = NULL;

/* 线程写入这个管道，标准输入被重新分配为它的读端。 */
static HANDLE stdin_pipe_r = NULL, stdin_pipe_w = NULL;
/* 这是一个从真正的标准输入（thread_stdin_handle）读取并写入到stdin_pipe_w的线程，stdin_pipe_w被重新分配为程序的其余部分看到的标准输入。一旦启动，除非发生错误，它永远不会结束。win_stdin_start_thread负责设置thread_stdin_handle。 */
static DWORD WINAPI win_stdin_thread_func(void *data) {
    DWORD n, nwritten;
    char buffer[BUFSIZ];

    for (;;) {
        if (ReadFile(thread_stdin_handle, buffer, sizeof(buffer), &n, NULL) == 0)
            break;
        if (n == -1 || n == 0)
            break;

        if (WriteFile(stdin_pipe_w, buffer, n, &nwritten, NULL) == 0)
            break;
        if (nwritten != n)
            break;
    }
    CloseHandle(thread_stdin_handle);
    CloseHandle(stdin_pipe_w);

    return 0;
}

/* 获取文件描述符的换行符转换模式（_O_TEXT或_O_BINARY）。_O_TEXT执行CRLF-LF转换，_O_BINARY不执行转换。与_setmode相对。 */
static int _getmode(int fd)
{
    int mode;

    /* 没有标准的_getmode函数，但_setmode返回先前的值。将其设置为一个虚拟值，然后将其设置回去。 */
    mode = _setmode(fd, _O_BINARY);
    _setmode(fd, mode);

    return mode;
}

/* 启动读取线程并进行所有文件句柄/描述符重定向。成功返回非零，错误返回零。 */
int win_stdin_start_thread(void) {
    int stdin_fd;
    int stdin_fmode;

    assert(stdin_thread == NULL);
    assert(stdin_pipe_r == NULL);
    assert(stdin_pipe_w == NULL);
    assert(thread_stdin_handle == NULL);

    /* 创建win_stdin_thread_func写入的管道。我们重新分配读端以成为程序的其余部分看到的新stdin。 */
    if (CreatePipe(&stdin_pipe_r, &stdin_pipe_w, NULL, 0) == 0)
        return 0;
    /* 复制标准输入句柄，用于 win_stdin_thread_func 使用。在我们伪造标准输入以从管道中读取之前，它将保持对真正标准输入的引用。 */
    if (DuplicateHandle(GetCurrentProcess(), GetStdHandle(STD_INPUT_HANDLE),
                        GetCurrentProcess(), &thread_stdin_handle,
                        0, FALSE, DUPLICATE_SAME_ACCESS) == 0) {
        CloseHandle(stdin_pipe_r);
        CloseHandle(stdin_pipe_w);
        return 0;
    }

    /* 设置标准输入句柄以从管道中读取。 */
    if (SetStdHandle(STD_INPUT_HANDLE, stdin_pipe_r) == 0) {
        CloseHandle(stdin_pipe_r);
        CloseHandle(stdin_pipe_w);
        CloseHandle(thread_stdin_handle);
        return 0;
    }
    /* 需要重定向文件描述符 0。_open_osfhandle 从现有句柄创建一个新的文件描述符。 */
    /* 记住换行符转换模式（_O_TEXT 或 _O_BINARY），并在新文件描述符中恢复它。 */
    stdin_fmode = _getmode(STDIN_FILENO);
    stdin_fd = _open_osfhandle((intptr_t) GetStdHandle(STD_INPUT_HANDLE), _O_RDONLY | stdin_fmode);
    if (stdin_fd == -1) {
        CloseHandle(stdin_pipe_r);
        CloseHandle(stdin_pipe_w);
        CloseHandle(thread_stdin_handle);
        return 0;
    }
    dup2(stdin_fd, STDIN_FILENO);

    /* 最后，启动线程。我们不需要保留对它的引用，因为它会一直运行直到程序终止。从现在开始，所有从标准输入句柄或文件描述符 0 读取的内容都将来自由线程提供的匿名管道。 */
    stdin_thread = CreateThread(NULL, 0, win_stdin_thread_func, NULL, 0, NULL);
    if (stdin_thread == NULL) {
        CloseHandle(stdin_pipe_r);
        CloseHandle(stdin_pipe_w);
        CloseHandle(thread_stdin_handle);
        return 0;
    }

    return 1;
// 检查标准输入是否可用，一旦上述所有操作都已完成。
int win_stdin_ready(void) {
    // 定义一个变量n，用于存储管道中的字节数
    DWORD n;
    // 断言标准输入的读取管道不为空
    assert(stdin_pipe_r != NULL);
    // 如果标准输入管道中有数据，则返回1
    if (!PeekNamedPipe(stdin_pipe_r, NULL, 0, NULL, &n, NULL))
        return 1;
    // 如果管道中的字节数大于0，则返回1，表示标准输入可用
    return n > 0;
}
```