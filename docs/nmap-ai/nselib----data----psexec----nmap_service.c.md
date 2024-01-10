# `nmap\nselib\data\psexec\nmap_service.c`

```
#include <stdio.h>
#include <windows.h>

FILE *outfile;  // 定义一个文件指针变量

SERVICE_STATUS ServiceStatus;  // 定义一个服务状态变量
SERVICE_STATUS_HANDLE hStatus;  // 定义一个服务状态句柄变量

static char *enc_key;  // 定义一个静态的字符指针变量，用于存储加密密钥
static int   enc_key_loc;  // 定义一个静态的整型变量，用于存储加密密钥的位置

static void log_message(char *format, ...)  // 定义一个静态的日志函数，可变参数
{
    static int enabled = 1;  // 定义一个静态的整型变量，用于表示日志是否启用

    if(!format)  // 如果格式为空
    {
        enabled = 0;  // 禁用日志
        DeleteFile("c:\\nmap-log.txt");  // 删除日志文件
    }

    if(enabled)  // 如果日志启用
    {
        va_list argp;  // 定义一个可变参数列表
        FILE *file;  // 定义一个文件指针变量

        fopen_s(&file, "c:\\nmap-log.txt", "a");  // 打开日志文件，追加模式

        if(file != NULL)  // 如果文件指针不为空
        {
            va_start(argp, format);  // 初始化可变参数列表
            vfprintf(file, format, argp);  // 将格式化输出到文件
            va_end(argp);  // 结束可变参数列表
            fprintf(file, "\n");  // 输出换行符
            fclose(file);  // 关闭文件
        }
    }
}

static char cipher(char c)  // 定义一个静态的加密函数，接收一个字符参数
{
    if(strlen(enc_key) == 0)  // 如果加密密钥长度为0
        return c;  // 返回原字符

    c = c ^ enc_key[enc_key_loc];  // 对字符进行异或加密
    enc_key_loc = (enc_key_loc + 1) % strlen(enc_key);  // 更新加密密钥位置

    return c;  // 返回加密后的字符
}

static void output(int num, char *str, int length)  // 定义一个静态的输出函数，接收一个整型参数、一个字符指针参数和一个整型参数
{
    int i;  // 定义一个整型变量

    if(length == -1)  // 如果长度为-1
        length = strlen(str);  // 获取字符串的长度

    for(i = 0; i < length; i++)  // 遍历字符串
    {
        if(str[i] == '\n')  // 如果是换行符
        {
            fprintf(outfile, "%c", cipher('\n'));  // 输出加密后的换行符
            fprintf(outfile, "%c", cipher('0' + (num % 10)));  // 输出加密后的数字
        }
        else  // 如果不是换行符
        {
            fprintf(outfile, "%c", cipher(str[i]));  // 输出加密后的字符
        }
    }
}

static void go(int num, char *lpAppPath, char *env, int headless, int include_stderr, char *readfile)  // 定义一个静态的执行函数，接收一个整型参数、三个字符指针参数和两个整型参数
{
    STARTUPINFO         startupInfo;  // 定义一个启动信息结构体
    PROCESS_INFORMATION processInformation;  // 定义一个进程信息结构体
    SECURITY_ATTRIBUTES sa;  // 定义一个安全属性结构体
    HANDLE              stdout_read, stdout_write;  // 定义两个句柄变量
    DWORD               creation_flags;  // 定义一个双字型变量

    int bytes_read;  // 定义一个整型变量
    char buffer[1024];  // 定义一个大小为1024的字符数组

    /* Create a security attributes structure. This is required to inherit handles. */
    ZeroMemory(&sa, sizeof(SECURITY_ATTRIBUTES));  // 将安全属性结构体清零
    sa.nLength              = sizeof(SECURITY_ATTRIBUTES);  // 设置结构体长度
    sa.lpSecurityDescriptor = NULL;  // 设置安全描述符为空

    if(!headless)  // 如果不是无头模式
        sa.bInheritHandle       = TRUE;  // 继承句柄

    /* Create a pipe that'll be used for stdout and stderr. */
    # 如果不是无头模式，创建一个管道用于标准输出
    if(!headless)
        CreatePipe(&stdout_read, &stdout_write, &sa, 1);

    # 初始化启动信息结构体，最重要的是将标准输出/标准错误句柄设置为我们的管道
    ZeroMemory(&startupInfo, sizeof(STARTUPINFO));
    startupInfo.cb         = sizeof(STARTUPINFO);

    # 如果不是无头模式
    if(!headless)
    {
        startupInfo.dwFlags    = STARTF_USESTDHANDLES;
        startupInfo.hStdOutput = stdout_write;
        if(include_stderr)
            startupInfo.hStdError  = stdout_write;
    }

    # 记录一些消息
    log_message("Attempting to load the program: %s", lpAppPath);

    # 初始化进程信息结构体
    ZeroMemory(&processInformation, sizeof(PROCESS_INFORMATION));

    # 用于分隔一个程序的输出和下一个程序的输入
    output(num, "\n", -1);

    # 决定创建标志
    creation_flags = CREATE_NO_WINDOW;
    if(headless)
        creation_flags = DETACHED_PROCESS;

    # 使用复杂的CreateProcess函数创建实际进程
    if(!CreateProcess(NULL, lpAppPath, 0, &sa, sa.bInheritHandle, CREATE_NO_WINDOW, env, 0, &startupInfo, &processInformation))
    {
        output(num, "Failed to create the process", -1);

        # 如果不是无头模式，关闭句柄
        if(!headless)
        {
            CloseHandle(stdout_write);
            CloseHandle(stdout_read);
        }
    }
    else
    {
        // 记录日志信息，表示成功创建了进程
        log_message("Successfully created the process!");

        /* 如果不是无头模式，则读取管道 */
        if(!headless)
        {
            /* 关闭写句柄 -- 如果不这样做，接下来的 ReadFile() 会被卡住 */
            CloseHandle(stdout_write);

            /* 从管道中读取数据 */
            log_message("Attempting to read from the pipe");
            while(ReadFile(stdout_read, buffer, 1023, &bytes_read, NULL))
            {
                if(strlen(readfile) == 0)
                    output(num, buffer, bytes_read);
            }
            CloseHandle(stdout_read);

            /* 如果要读取的是输出文件而不是标准输出，就在这里处理 */
            if(strlen(readfile) > 0)
            {
                FILE *read;
                errno_t err;

                log_message("Trying to open output file: %s", readfile);
                err = fopen_s(&read, readfile, "rb");

                if(err)
                {
                    log_message("Couldn't open the readfile: %d", err);
                    output(num, "Couldn't open the output file", -1);
                }
                else
                {
                    char buf[1024];
                    int count;

                    count = fread(buf, 1, 1024, read);
                    while(count)
                    {
                        output(num, buf, count);
                        count = fread(buf, 1, 1024, read);
                    }

                    fclose(read);
                }
            }
        }
        else
        {
            output(num, "Process has been created", -1);
        }

        // 记录日志信息，表示处理完成
        log_message("Done!");
    }
}

// 控制处理程序函数
static void ControlHandler(DWORD request)
{
    switch(request)
    {
        // 如果收到停止服务的请求
        case SERVICE_CONTROL_STOP:

            // 设置退出代码为0，当前状态为停止
            ServiceStatus.dwWin32ExitCode = 0;
            ServiceStatus.dwCurrentState  = SERVICE_STOPPED;
            // 更新服务状态
            SetServiceStatus (hStatus, &ServiceStatus);
            return;

        // 如果收到关闭计算机的请求
        case SERVICE_CONTROL_SHUTDOWN:

            // 设置退出代码为0，当前状态为停止
            ServiceStatus.dwWin32ExitCode = 0;
            ServiceStatus.dwCurrentState  = SERVICE_STOPPED;
            // 更新服务状态
            SetServiceStatus (hStatus, &ServiceStatus);
            return;

        // 默认情况
        default:
            break;
    }

    // 更新服务状态
    SetServiceStatus(hStatus,  &ServiceStatus);
}

// 终止服务的函数
static void die(int err)
{
    // 参数不足
    ServiceStatus.dwCurrentState  = SERVICE_STOPPED;
    ServiceStatus.dwWin32ExitCode = err;
    // 更新服务状态
    SetServiceStatus(hStatus, &ServiceStatus);
}

// 服务主函数
static void ServiceMain(int argc, char** argv)
{
    char   *outfile_name;
    char   *tempfile_name;
    int     count;
    int     logging;
    int     result;
    int     i;
    char   *current_directory;

    /* 确保我们得到了最少数量的参数。 */
    if(argc < 6)
        return;

    /* 读取参数。 */
    outfile_name      = argv[1];
    tempfile_name     = argv[2];
    count             = atoi(argv[3]);
    logging           = atoi(argv[4]);
    enc_key           = argv[5];
    enc_key_loc       = 0;
    current_directory = argv[6];

    /* 如果他们没有打开日志记录，禁用它。 */
    if(logging != 1)
        log_message(NULL);

    /* 记录状态。 */
    log_message("");
    log_message("-----------------------");
    log_message("STARTING");

    /* 记录所有参数。 */
    log_message("Arguments: %d\n", argc);
    for(i = 0; i < argc; i++)
        log_message("Argument %d: %s", i, argv[i]);

    /* 设置服务。对于我们正在做的事情可能是不必要的，但也没有坏处。 */
    ServiceStatus.dwServiceType             = SERVICE_WIN32;
}
    # 设置服务的当前状态为运行中
    ServiceStatus.dwCurrentState            = SERVICE_RUNNING;
    # 设置服务接受停止和关闭命令
    ServiceStatus.dwControlsAccepted        = SERVICE_ACCEPT_STOP | SERVICE_ACCEPT_SHUTDOWN;
    # 设置服务的退出码为0
    ServiceStatus.dwWin32ExitCode           = 0;
    # 设置服务的特定退出码为0
    ServiceStatus.dwServiceSpecificExitCode = 0;
    # 设置服务的检查点为0
    ServiceStatus.dwCheckPoint              = 0;
    # 设置服务的等待提示为0
    ServiceStatus.dwWaitHint                = 0;
    # 注册服务控制处理程序
    hStatus = RegisterServiceCtrlHandler("", (LPHANDLER_FUNCTION)ControlHandler);
    # 设置服务状态
    SetServiceStatus(hStatus, &ServiceStatus);

    # 如果注册控制处理程序失败
    if(hStatus == (SERVICE_STATUS_HANDLE)0)
    {
        # 记录日志信息
        log_message("Service failed to start");
        # 终止程序
        die(-1);
        # 返回
        return;
    }

    # 设置当前目录
    SetCurrentDirectory(current_directory);

    # 打开临时输出文件
    log_message("Opening temporary output file: %s", tempfile_name);

    # 打开输出文件
    if(result = fopen_s(&outfile, tempfile_name, "wb"))
    {
        # 记录日志信息
        log_message("Couldn't open output file: %d", result);
        # 终止程序
        die(-1);
    }
    else
    {
        /* 运行我们得到的程序。 */
        for(i = 0; i < count; i++)
        {
            // 获取程序、环境变量、是否无头模式、是否包含标准错误、读取文件参数
            char *program        = argv[(i*5) + 7];
            char *env            = argv[(i*5) + 8];
            char *headless       = argv[(i*5) + 9];
            char *include_stderr = argv[(i*5) + 10];
            char *read_file      = argv[(i*5) + 11];

            // 调用函数执行程序
            go(i, program, env, !strcmp(headless, "true"), !strcmp(include_stderr, "true"), read_file);
        }

        /* 关闭输出文件。 */
        if(fclose(outfile))
            log_message("无法关闭文件：%d", errno);

        /* 重命名输出文件（这告诉 Nmap 我们已经完成了）。 */
        log_message("重命名文件 %s => %s", tempfile_name, outfile_name);

        /* 我注意到有时候，程序会继承文件句柄（或其他什么东西），所以我不能立即更改它。因此，允许大约 10 秒来移动它。 */
        for(i = 0; i < 10; i++)
        {
            if(rename(tempfile_name, outfile_name))
            {
                log_message("无法重命名文件：%d（将再尝试 %d 次）", errno, 10 - i - 1);
            }
            else
            {
                log_message("文件成功重命名！");
                break;
            }

            Sleep(1000);
        }

        /* 清理并停止服务。 */
        die(0);
    }
// 主函数，程序入口
int main(int argc, char *argv[])
{
    // 创建服务表，包含两个条目
    SERVICE_TABLE_ENTRY ServiceTable[2];
    // 设置第一个条目的服务名为空字符串
    ServiceTable[0].lpServiceName = "";
    // 设置第一个条目的服务处理函数为ServiceMain
    ServiceTable[0].lpServiceProc = (LPSERVICE_MAIN_FUNCTION)ServiceMain;

    // 设置第二个条目的服务名为NULL
    ServiceTable[1].lpServiceName = NULL;
    // 设置第二个条目的服务处理函数为NULL
    ServiceTable[1].lpServiceProc = NULL;
    // 启动服务的控制分发器线程
    StartServiceCtrlDispatcher(ServiceTable);
}
```