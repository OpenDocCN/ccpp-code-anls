# `nmap\libpcap\lbl\os-sunos4.h`

```
/*
 * 版权声明，保留所有权利
 */
/* 在源代码和二进制形式中允许修改和重新分发，但需要保留版权声明和许可声明 */
/* 所有包含二进制代码的分发都需要在文档或其他材料中包含版权声明和许可声明 */
/* 所有宣传材料都需要包含以下声明："本产品包含由加利福尼亚大学，劳伦斯伯克利实验室及其贡献者开发的软件" */
/* 未经特定书面许可，不得使用大学的名称或其贡献者的名称来认可或推广从本软件衍生的产品 */
/* 本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保 */
 */

/* SunOS 4 中缺少的函数原型 */
#ifdef FILE
int    _filbuf(FILE *);
int    _flsbuf(u_char, FILE *);
int    fclose(FILE *);
int    fflush(FILE *);
int    fgetc(FILE *);
int    fprintf(FILE *, const char *, ...);
int    fputc(int, FILE *);
int    fputs(const char *, FILE *);
u_int    fread(void *, u_int, u_int, FILE *);
int    fseek(FILE *, long, int);
u_int    fwrite(const void *, u_int, u_int, FILE *);
int    pclose(FILE *);
void    rewind(FILE *);
void    setbuf(FILE *, char *);
int    setlinebuf(FILE *);
int    ungetc(int, FILE *);
int    vfprintf(FILE *, const char *, ...);
int    vprintf(const char *, ...);
#endif

#if __GNUC__ <= 1
int    read(int, char *, u_int);
int    write(int, char *, u_int);
#endif

long    a64l(const char *);
#ifdef __STDC__
struct    sockaddr;
#endif
# 定义接受连接的函数，参数为套接字描述符、指向存放客户端地址信息的结构体指针、客户端地址信息结构体的长度指针
int    accept(int, struct sockaddr *, int *);
# 将套接字绑定到指定地址，参数为套接字描述符、指向存放地址信息的结构体指针、地址信息结构体的长度
int    bind(int, struct sockaddr *, int);
# 比较两个内存区域的内容，参数为两个内存区域的指针和比较的长度
int    bcmp(const void *, const void *, u_int);
# 复制内存内容，参数为源内存区域的指针、目标内存区域的指针、复制的长度
void    bcopy(const void *, void *, u_int);
# 将内存区域清零，参数为内存区域的指针和长度
void    bzero(void *, int);
# 改变根目录，参数为新的根目录路径
int    chroot(const char *);
# 关闭文件描述符，参数为文件描述符
int    close(int);
# 关闭系统日志
void    closelog(void);
# 连接到指定地址，参数为套接字描述符、指向存放目标地址信息的结构体指针、地址信息结构体的长度
int    connect(int, struct sockaddr *, int);
# 加密字符串，参数为需要加密的字符串和加密盐
char    *crypt(const char *, const char *);
# 创建守护进程，参数为是否关闭标准输入输出和错误输出
int    daemon(int, int);
# 改变文件的权限，参数为文件描述符和新的权限
int    fchmod(int, int);
# 改变文件的所有者和所属组，参数为文件描述符、新的所有者和新的所属组
int    fchown(int, int, int);
# 结束组文件扫描
void    endgrent(void);
# 结束密码文件扫描
void    endpwent(void);
# 将字符串转换为以太网地址结构体指针
struct    ether_addr *ether_aton(const char *);
# 对文件加锁，参数为文件描述符和加锁方式
int    flock(int, int);
# 获取文件状态，参数为文件描述符和存放文件状态信息的结构体指针
int    fstat(int, struct stat *);
# 获取文件系统状态，参数为文件描述符和存放文件系统状态信息的结构体指针
int    fstatfs(int, struct statfs *);
# 同步文件数据到磁盘
int    fsync(int);
# 获取当前时间，参数为存放时间信息的结构体指针
int    ftime(struct timeb *);
# 改变文件大小，参数为文件描述符和新的大小
int    ftruncate(int, off_t);
# 获取文件描述符的最大数量
int    getdtablesize(void);
# 获取主机标识符
long    gethostid(void);
# 获取主机名，参数为存放主机名的字符串指针和长度
int    gethostname(char *, int);
# 解析命令行参数，参数为参数个数、参数数组和选项字符串
int    getopt(int, char * const *, const char *);
# 获取页面大小
int    getpagesize(void);
# 获取密码，参数为提示字符串
char    *getpass(char *);
# 获取对端套接字地址，参数为套接字描述符、指向存放对端地址信息的结构体指针、对端地址信息结构体的长度指针
int    getpeername(int, struct sockaddr *, int *);
# 获取进程优先级，参数为进程标识符和进程类型
int    getpriority(int, int);
# 获取资源限制，参数为资源类型和存放资源限制信息的结构体指针
int    getrlimit(int, struct rlimit *);
# 获取套接字地址，参数为套接字描述符、指向存放套接字地址信息的结构体指针、套接字地址信息结构体的长度指针
int    getsockname(int, struct sockaddr *, int *);
# 获取套接字选项，参数为套接字描述符、选项级别、选项名、存放选项值的缓冲区指针和选项值的长度指针
int    getsockopt(int, int, int, char *, int *);
# 获取当前时间和时区，参数为存放时间信息的结构体指针和存放时区信息的结构体指针
int    gettimeofday(struct timeval *, struct timezone *);
# 获取用户 shell
char    *getusershell(void);
# 获取当前工作目录，参数为存放路径的字符串指针
char    *getwd(char *);
# 初始化用户组，参数为用户名和组标识符
int    initgroups(const char *, int);
# 控制设备，参数为文件描述符、控制命令和控制参数
int    ioctl(int, int, caddr_t);
# 检查远程用户是否合法，参数为主机标识符、远程用户标识符、本地用户名和本地主机名
int    iruserok(u_long, int, char *, char *);
# 判断是否为终端设备，参数为文件描述符
int    isatty(int);
# 杀死进程组，参数为进程组标识符和信号
int    killpg(int, int);
# 监听套接字，参数为套接字描述符和最大连接队列长度
int    listen(int, int);
# 登录系统，参数为用户登录信息结构体指针
void    login(struct utmp *);
# 注销系统，参数为用户名
int    logout(const char *);
# 移动文件指针，参数为文件描述符、偏移量和起始位置
off_t    lseek(int, off_t, int);
# 获取文件状态，参数为文件路径和存放文件状态信息的结构体指针
int    lstat(const char *, struct stat *);
# 创建临时文件，参数为文件名模板
int    mkstemp(char *);
# 创建临时文件名，参数为文件名模板
char    *mktemp(char *);
# 解除内存映射，参数为内存区域的指针和长度
int    munmap(caddr_t, int);
# 打开系统日志，参数为标识符字符串、日志选项和日志设施
void    openlog(const char *, int, int);
# 输出错误信息
void    perror(const char *);

# 格式化输出
int    printf(const char *, ...);

# 输出字符串
int    puts(const char *);

# 生成随机数
long    random(void);

# 读取符号链接
int    readlink(const char *, char *, int);

# 读取多个缓冲区的数据
int    readv(int, struct iovec *, int);

# 从套接字接收数据
int    recv(int, char *, u_int, int);

# 从套接字接收数据，并获取发送方地址
int    recvfrom(int, char *, u_int, int, struct sockaddr *, int *);

# 重命名文件
int    rename(const char *, const char *);

# 远程命令执行
int    rcmd(char **, u_short, char *, char *, char *, int *);

# 申请一个保留端口
int    rresvport(int *);

# 发送数据
int    send(int, char *, u_int, int);

# 发送数据，并指定目标地址
int    sendto(int, char *, u_int, int, struct sockaddr *, int);

# 设置环境变量
int    setenv(const char *, const char *, int);

# 设置有效用户 ID
int    seteuid(int);

# 设置进程优先级
int    setpriority(int, int, int);

# I/O 复用
int    select(int, fd_set *, fd_set *, fd_set *, struct timeval *);

# 设置进程组 ID
int    setpgrp(int, int);

# 打开密码文件
void    setpwent(void);

# 设置资源限制
int    setrlimit(int, struct rlimit *);

# 设置套接字选项
int    setsockopt(int, int, int, char *, int);

# 关闭套接字
int    shutdown(int, int);

# 阻塞指定信号
int    sigblock(int);

# 设置信号处理函数
void    (*signal (int, void (*) (int))) (int);

# 暂停进程，直到收到信号
int    sigpause(int);

# 设置信号屏蔽字
int    sigsetmask(int);

# 设置信号处理函数
int    sigvec(int, struct sigvec *, struct sigvec*);

# 格式化输出到字符串
int    snprintf(char *, size_t, const char *, ...);

# 创建套接字
int    socket(int, int, int);

# 创建一对连接的套接字
int    socketpair(int, int, int, int *);

# 创建符号链接
int    symlink(const char *, const char *);

# 设置随机数种子
void    srandom(int);

# 从字符串中读取格式化输入
int    sscanf(char *, const char *, ...);

# 获取文件状态
int    stat(const char *, struct stat *);

# 获取文件系统状态
int    statfs(char *, struct statfs *);

# 获取错误信息字符串
char    *strerror(int);

# 比较字符串，不区分大小写
int    strcasecmp(const char *, const char *);

# 格式化时间
int    strftime(char *, int, char *, struct tm *);

# 比较字符串，不区分大小写
int    strncasecmp(const char *, const char *, int);

# 将字符串转换为长整型
long    strtol(const char *, char **, int);

# 同步文件系统
void    sync(void);

# 系统日志
void    syslog(int, const char *, ...);

# 执行系统命令
int    system(const char *);

# 获取文件当前位置
long    tell(int);

# 获取时钟时间
time_t    time(time_t *);

# 获取时区信息
char    *timezone(int, int);

# 将字符转换为小写
int    tolower(int);

# 将字符转换为大写
int    toupper(int);

# 截断文件
int    truncate(char *, off_t);

# 删除环境变量
void    unsetenv(const char *);

# 创建子进程
int    vfork(void);
/* 声明 vsprintf 函数，接受一个格式化字符串和可变数量的参数，将结果输出到字符数组中 */
int vsprintf(char *, const char *, ...);
/* 声明 writev 函数，将数据写入文件描述符中 */
int writev(int, struct iovec *, int);
/* 如果是标准 C，声明 rusage 结构体 */
#ifdef __STDC__
struct rusage;
#endif
/* 声明 utimes 函数，修改文件的访问和修改时间 */
int utimes(const char *, struct timeval *);
/* 如果编译器版本小于等于 1，声明 wait 函数和 wait3 函数 */
#if __GNUC__ <= 1
int wait(int *);
pid_t wait3(int *, int, struct rusage *);
#endif

/* 信号处理的丑陋破解 */
#ifdef SIG_ERR
/* 重定义 SIG_ERR 常量 */
#undef SIG_ERR
#define SIG_ERR (void (*)(int))-1
/* 重定义 SIG_DFL 常量 */
#undef SIG_DFL
#define SIG_DFL (void (*)(int))0
/* 重定义 SIG_IGN 常量 */
#undef SIG_IGN
#define SIG_IGN (void (*)(int))1

#ifdef KERNEL
/* 如果是内核代码，重定义 SIG_CATCH 常量 */
#undef SIG_CATCH
#define SIG_CATCH (void (*)(int))2
#endif
/* 重定义 SIG_HOLD 常量 */
#undef SIG_HOLD
#define SIG_HOLD (void (*)(int))3
#endif
```