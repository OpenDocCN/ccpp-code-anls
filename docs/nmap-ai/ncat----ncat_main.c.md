# `nmap\ncat\ncat_main.c`

```
/* $Id$ */

#include "nsock.h"  // 包含网络套接字操作的头文件
#include "ncat.h"   // 包含ncat的头文件
#include "util.h"   // 包含一些工具函数的头文件
#include "sys_wrap.h"  // 包含系统调用的封装函数的头文件

#include <getopt.h>  // 包含命令行参数解析的头文件

#ifndef WIN32
#include <unistd.h>  // 包含UNIX系统调用的头文件
#endif
#include <limits.h>  // 包含一些常量的头文件
#include <stdlib.h>  // 包含标准库函数的头文件
#include <string.h>  // 包含字符串处理函数的头文件
#include <stdio.h>   // 包含输入输出函数的头文件
#include <errno.h>   // 包含错误处理的头文件
#ifndef WIN32
#include <netdb.h>  // 包含网络数据库操作的头文件
#endif
#include <fcntl.h>   // 包含文件控制函数的头文件

#ifdef HAVE_OPENSSL
#include <openssl/ssl.h>  // 包含OpenSSL的SSL协议支持的头文件
#include <openssl/err.h>  // 包含OpenSSL的错误处理的头文件
#endif

#ifdef HAVE_LUA
#include "ncat_lua.h"  // 包含ncat的Lua支持的头文件
#endif

static int ncat_connect_mode(void);  // 声明ncat_connect_mode函数，用于连接模式
static int ncat_listen_mode(void);   // 声明ncat_listen_mode函数，用于监听模式

/* 解析端口号 */
static unsigned int parseport(char *str, unsigned int maxport, char *msg)
{
    unsigned long port;
    char *next;
    errno = 0;
    port = strtoul(str, &next, 10);  // 将字符串转换为无符号长整型
    if (errno || *next || (maxport && port > maxport))  // 如果发生错误或者端口号超出范围
        bye("Invalid %s number \"%s\".", msg, str);  // 输出错误信息并退出程序
    return (unsigned int) port;  // 返回端口号
}

/* 解析代理地址/端口组合 */
static size_t parseproxy(char *str, struct sockaddr_storage *ss,
    size_t *sslen, unsigned short *portno)
{
    char *p = str;
    int rc;

    if (*p == '[') {
        p = strchr(p, ']');
        if (p == NULL)
            bye("Invalid proxy IPv6 address \"%s\".", str);
        ++str;
        *p++ = '\0';
    }

    p = strchr(p, ':');
    if (p != NULL && strchr(p + 1, ':') == NULL) {
        *p++ = '\0';
        *portno = (unsigned short) parseport(p, 0xFFFF, "proxy port");
    }

    rc = resolve(str, *portno, ss, sslen, o.af);  // 解析代理地址
    if (rc != 0) {
        loguser("Could not resolve proxy \"%s\": %s.\n", str, gai_strerror(rc));  // 输出错误信息
        if (o.af == AF_INET6)
            loguser("Did you specify the port number? It's required for IPv6.\n");  // 输出提示信息
        exit(EXIT_FAILURE);  // 退出程序
    }

    return *sslen;  // 返回地址长度
}

static int parse_timespec (const char *const tspec, const char *const optname)
{
    const long l = tval2msecs(tspec);  // 将时间字符串转换为毫秒数
    if (l <= 0 || l > INT_MAX)  // 如果时间小于等于0或者大于INT_MAX
        bye("Invalid %s \"%s\" (must be greater than 0 and less than %ds).",
            optname, tspec, INT_MAX / 1000);  // 输出错误信息并退出程序
    # 如果输入的时间大于等于100000毫秒，并且时间单位为NULL
    if (l >= 100 * 1000 && tval_unit(tspec) == NULL)
        # 输出提示信息并退出程序
        bye("Since April 2010, the default unit for %s is seconds, so your "
            "time of \"%s\" is %.1f minutes. Use \"%sms\" for %s milliseconds.",
            optname, optarg, l / 1000.0 / 60, optarg, optarg);
    # 返回时间的整数部分
    return (int)l;
/* 这些函数实现了一个简单的链表，用于保存允许/拒绝规范，直到选项解析结束。 */

// 定义一个结构体，用于表示链表的节点
struct host_list_node {
    // 如果为 false，则 spec 是包含主机模式的文件名
    int is_filename;
    // 主机模式或文件名
    char *spec;
    // 指向下一个节点的指针
    struct host_list_node *next;
};

// 向链表中添加主机模式
static void host_list_add_spec(struct host_list_node **list, char *spec)
{
    // 分配内存以存储新节点
    struct host_list_node *node = (struct host_list_node *) safe_malloc(sizeof(*node));
    // 设置节点的属性
    node->is_filename = 0;
    node->spec = spec;
    node->next = *list;
    *list = node;
}

// 向链表中添加文件名
static void host_list_add_filename(struct host_list_node **list, char *filename)
{
    // 分配内存以存储新节点
    struct host_list_node *node = (struct host_list_node *) safe_malloc(sizeof(*node));
    // 设置节点的属性
    node->is_filename = 1;
    node->spec = filename;
    node->next = *list;
    *list = node;
}

// 释放链表的内存
static void host_list_free(struct host_list_node *list)
{
    struct host_list_node *next;
    // 遍历链表并释放每个节点的内存
    for ( ; list != NULL; list = next) {
        next = list->next;
        free(list);
    }
}

// 将链表中的主机模式或文件名转换为地址集合
static void host_list_to_set(struct addrset *set, struct host_list_node *list)
{
    struct host_list_node *node;

    // 遍历链表中的节点
    for (node = list; node != NULL; node = node->next) {
        // 如果节点表示文件名
        if (node->is_filename) {
            FILE *fd;

            // 打开文件
            fd = fopen(node->spec, "r");
            // 如果无法打开文件，则输出错误信息并退出程序
            if (fd == NULL)
                bye("can't open %s: %s.", node->spec, strerror(errno));
            // 将文件中的主机模式添加到地址集合中
            if (!addrset_add_file(set, fd, o.af, !o.nodns))
                bye("error in hosts file %s.", node->spec);
            // 关闭文件
            fclose(fd);
        } else {
            char *spec, *commalist;

            commalist = node->spec;
            // 将逗号分隔的主机模式逐个添加到地址集合中
            while ((spec = strtok(commalist, ",")) != NULL) {
                commalist = NULL;
                if (!addrset_add_spec(set, spec, o.af, !o.nodns))
                    bye("error in host specification \"%s\".", node->spec);
            }
        }
    }
}

// 打印程序的版本信息
static void print_banner(void)
{
    loguser("Version %s ( %s )\n", NCAT_VERSION, NCAT_URL);
}
int main(int argc, char *argv[])
{
    /* 为了允许和拒绝的主机列表，我们必须在选项解析完成之后进行缓冲。
       将主机添加到地址集可能需要进行名称解析，这可能会因为像 -n 和 -6 这样的选项而有所不同。 */
    struct host_list_node *allow_host_list = NULL;
    struct host_list_node *deny_host_list = NULL;

    unsigned short proxyport;
    /* vsock 端口是 32 位的，因此端口变量必须至少有这么宽。 */
    unsigned int max_port = 65535;
    long long int srcport = -1;
    char *source = NULL;

    struct option long_options[] = {
        {"4",               no_argument,        NULL,         '4'},
        {"6",               no_argument,        NULL,         '6'},
#if HAVE_SYS_UN_H
        {"unixsock",        no_argument,        NULL,         'U'},
#endif
#if HAVE_LINUX_VM_SOCKETS_H
        {"vsock",           no_argument,        NULL,         0},
#endif
        {"crlf",            no_argument,        NULL,         'C'},
        {"g",               required_argument,  NULL,         'g'},
        {"G",               required_argument,  NULL,         'G'},
        {"exec",            required_argument,  NULL,         'e'},
        {"sh-exec",         required_argument,  NULL,         'c'},
#ifdef HAVE_LUA
        {"lua-exec",        required_argument,  NULL,         0},
        {"lua-exec-internal",required_argument, NULL,         0},
#ifdef HAVE_OPENSSL
        {"ssl-cert",        required_argument,  NULL,         0},
        {"ssl-key",         required_argument,  NULL,         0},
        {"ssl-verify",      no_argument,        NULL,         0},
        {"ssl-trustfile",   required_argument,  NULL,         0},
        {"ssl-ciphers",     required_argument,  NULL,         0},
        {"ssl-servername",  required_argument,  NULL,         0},
        {"ssl-alpn",        required_argument,  NULL,         0},
#else
        {"ssl-cert",        optional_argument,  NULL,         0},  # 定义一个名为 "ssl-cert" 的可选参数
        {"ssl-key",         optional_argument,  NULL,         0},  # 定义一个名为 "ssl-key" 的可选参数
        {"ssl-verify",      no_argument,        NULL,         0},  # 定义一个名为 "ssl-verify" 的无参数选项
        {"ssl-trustfile",   optional_argument,  NULL,         0},  # 定义一个名为 "ssl-trustfile" 的可选参数
        {"ssl-ciphers",     optional_argument,  NULL,         0},  # 定义一个名为 "ssl-ciphers" 的可选参数
        {"ssl-alpn",        optional_argument,  NULL,         0},  # 定义一个名为 "ssl-alpn" 的可选参数
#endif
        {0, 0, 0, 0}  # 结束长选项数组的标记

    };

    gettimeofday(&start_time, NULL);  # 获取当前时间并存储在 start_time 中
    /* Set default options. */
    options_init();  # 初始化默认选项

#ifdef WIN32
    windows_init();  # 在 Windows 系统上进行初始化
#endif

    while (1) {
        /* handle command line arguments */
        int option_index;
        int c = getopt_long(argc, argv, "46UCc:e:g:G:i:km:hp:d:lo:x:ts:uvw:nz",
                            long_options, &option_index);  # 处理命令行参数

        /* That's the end of the options. */
        if (c == -1)  # 如果 c 的值为 -1，表示选项处理结束
            break;

        switch (c) {
        case '4':
            o.af = AF_INET;  # 设置 o.af 为 AF_INET
            break;
        case '6':
#ifdef HAVE_IPV6
            o.af = AF_INET6;  # 如果支持 IPv6，则设置 o.af 为 AF_INET6
#else
            bye("-6 chosen when IPv6 wasn't compiled in.");  # 如果不支持 IPv6，则输出错误信息并退出
#endif
            break;
#if HAVE_SYS_UN_H
        case 'U':
            o.af = AF_UNIX;  # 设置 o.af 为 AF_UNIX
            break;
#ifdef HAVE_OPENSSL
            # 如果编译时包含了 OpenSSL 支持，则执行以下操作
            else if (strcmp(long_options[option_index].name, "ssl-cert") == 0) {
                # 设置 SSL 标志为 1
                o.ssl = 1;
                # 复制 optarg 到 o.sslcert
                o.sslcert = Strdup(optarg);
            } else if (strcmp(long_options[option_index].name, "ssl-key") == 0) {
                # 设置 SSL 标志为 1
                o.ssl = 1;
                # 复制 optarg 到 o.sslkey
                o.sslkey = Strdup(optarg);
            } else if (strcmp(long_options[option_index].name, "ssl-verify") == 0) {
                # 设置 o.sslverify 为 1
                o.sslverify = 1;
                # 设置 SSL 标志为 1
                o.ssl = 1;
            } else if (strcmp(long_options[option_index].name, "ssl-trustfile") == 0) {
                # 设置 SSL 标志为 1
                o.ssl = 1;
                # 如果 o.ssltrustfile 不为空，则报错
                if (o.ssltrustfile != NULL)
                    bye("The --ssl-trustfile option may be given only once.");
                # 复制 optarg 到 o.ssltrustfile
                o.ssltrustfile = Strdup(optarg);
                # 如果他们列出了一个信任文件，则假设他们想要证书验证
                o.sslverify = 1;
            } else if (strcmp(long_options[option_index].name, "ssl-ciphers") == 0) {
                # 设置 SSL 标志为 1
                o.ssl = 1;
                # 复制 optarg 到 o.sslciphers
                o.sslciphers = Strdup(optarg);
            } else if (strcmp(long_options[option_index].name, "ssl-servername") == 0) {
                # 设置 SSL 标志为 1
                o.ssl = 1;
                # 复制 optarg 到 o.sslservername
                o.sslservername = Strdup(optarg);
            }
#ifdef HAVE_ALPN_SUPPORT
            else if (strcmp(long_options[option_index].name, "ssl-alpn") == 0) {
                # 设置 SSL 标志为 1
                o.ssl = 1;
                # 复制 optarg 到 o.sslalpn
                o.sslalpn = Strdup(optarg);
            }
#else
            else if (strcmp(long_options[option_index].name, "ssl-alpn") == 0) {
                # 如果 OpenSSL 没有编译 ALPN 支持，则报错
                bye("OpenSSL does not have ALPN support compiled in. The --ssl-alpn option cannot be chosen.");
            }
#endif
#else
            # 如果选项不是上述列出的任何一个，且 OpenSSL 没有编译进来，抛出错误信息
            else if (strcmp(long_options[option_index].name, "ssl-cert") == 0) {
                bye("OpenSSL isn't compiled in. The --ssl-cert option cannot be chosen.");
            } else if (strcmp(long_options[option_index].name, "ssl-key") == 0) {
                bye("OpenSSL isn't compiled in. The --ssl-key option cannot be chosen.");
            } else if (strcmp(long_options[option_index].name, "ssl-verify") == 0) {
                bye("OpenSSL isn't compiled in. The --ssl-verify option cannot be chosen.");
            } else if (strcmp(long_options[option_index].name, "ssl-trustfile") == 0) {
                bye("OpenSSL isn't compiled in. The --ssl-trustfile option cannot be chosen.");
            } else if (strcmp(long_options[option_index].name, "ssl-ciphers") == 0) {
                bye("OpenSSL isn't compiled in. The --ssl-ciphers option cannot be chosen.");
            } else if (strcmp(long_options[option_index].name, "ssl-servername") == 0) {
                bye("OpenSSL isn't compiled in. The --ssl-servername option cannot be chosen.");
            } else if (strcmp(long_options[option_index].name, "ssl-alpn") == 0) {
                bye("OpenSSL isn't compiled in. The --ssl-alpn option cannot be chosen.");
            }
#endif
#ifdef HAVE_LUA
            // 如果编译时有 Lua 支持，则执行以下代码块
            else if (strcmp(long_options[option_index].name, "lua-exec") == 0) {
                // 如果已经设置了命令执行选项，则报错并退出
                if (o.cmdexec != NULL)
                    bye("Only one of --exec, --sh-exec, and --lua-exec is allowed.");
                // 设置命令执行选项为 optarg
                o.cmdexec = optarg;
                // 设置执行模式为 Lua
                o.execmode = EXEC_LUA;
            } else if (strcmp(long_options[option_index].name, "lua-exec-internal") == 0) {
                /* This command-line switch is undocumented on purpose. Do NOT use it
                   explicitly as its behavior might differ between Ncat releases.

                   Its goal is to switch the Ncat process to the Lua interpreter state
                   so that its standard output and input can be redirected to
                   particular connection's streams. Although it is implemented by
                   forking in POSIX builds, Windows does not have the fork() system
                   call and thus requires this workaround. More info here:
                   http://seclists.org/nmap-dev/2013/q2/492 */
                // 如果是 "lua-exec-internal" 选项
#ifdef WIN32
                // 如果是 Windows 系统，并且开启了调试模式，则记录日志
                if (o.debug)
                    logdebug("Enabling binary stdout for the Lua output.\n");
                // 设置标准输出为二进制模式
                int result = _setmode(_fileno(stdout), _O_BINARY);
                // 如果设置失败，则输出错误信息
                if (result == -1)
                    perror("Cannot set mode");
#endif
                // 确保参数个数为 3
                ncat_assert(argc == 3);
                // 设置命令执行选项为 argv[2]
                o.cmdexec = argv[2];
                // 初始化 Lua 环境
                lua_setup();
                // 执行 Lua 代码
                lua_run();
            }
#endif
#if HAVE_LINUX_VM_SOCKETS_H
            // 如果编译时有 Linux VM Sockets 支持，则执行以下代码块
            else if (strcmp(long_options[option_index].name, "vsock") == 0) {
                // 设置地址族为 AF_VSOCK
                o.af = AF_VSOCK;
            }
#endif
            // 结束 switch 语句
            break;
        case 'h':
            // 打印 Ncat 的版本信息
            printf("%s %s ( %s )\n", NCAT_NAME, NCAT_VERSION, NCAT_URL);
            // 打印 Ncat 的使用说明
            printf(
"Usage: ncat [options] [hostname] [port]\n"
"\n"
"Options taking a time assume seconds. Append 'ms' for milliseconds,\n"
"'s' for seconds, 'm' for minutes, or 'h' for hours (e.g. 500ms).\n"
"  -4                         Use IPv4 only\n"
"  -6                         Use IPv6 only\n"
# 使用 IPv6 地址
#if HAVE_SYS_UN_H
"  -U, --unixsock             Use Unix domain sockets only\n"
# 使用 Unix 域套接字
#endif
#if HAVE_LINUX_VM_SOCKETS_H
"      --vsock                Use vsock sockets only\n"
# 使用 vsock 套接字
#endif
"  -C, --crlf                 Use CRLF for EOL sequence\n"
# 使用 CRLF 作为行尾序列
"  -c, --sh-exec <command>    Executes the given command via /bin/sh\n"
# 通过 /bin/sh 执行给定的命令
#ifdef HAVE_LUA
"      --lua-exec <filename>  Executes the given Lua script\n"
# 执行给定的 Lua 脚本
#endif
"  -g hop1[,hop2,...]         Loose source routing hop points (8 max)\n"
# 设置松散源路由跳点
"  -G <n>                     Loose source routing hop pointer (4, 8, 12, ...)\n"
# 设置松散源路由跳点指针
"  -m, --max-conns <n>        Maximum <n> simultaneous connections\n"
# 设置最大同时连接数
"  -h, --help                 Display this help screen\n"
# 显示帮助屏幕
"  -d, --delay <time>         Wait between read/writes\n"
# 在读/写之间等待一段时间
"  -o, --output <filename>    Dump session data to a file\n"
# 将会话数据转储到文件中
"  -x, --hex-dump <filename>  Dump session data as hex to a file\n"
# 将会话数据以十六进制形式转储到文件中
"  -i, --idle-timeout <time>  Idle read/write timeout\n"
# 空闲读/写超时
"  -p, --source-port port     Specify source port to use\n"
# 指定要使用的源端口
"  -s, --source addr          Specify source address to use (doesn't affect -l)\n"
# 指定要使用的源地址（不影响 -l）
"  -l, --listen               Bind and listen for incoming connections\n"
# 绑定并监听传入连接
"  -k, --keep-open            Accept multiple connections in listen mode\n"
# 在监听模式下接受多个连接
"  -n, --nodns                Do not resolve hostnames via DNS\n"
# 不通过 DNS 解析主机名
"  -t, --telnet               Answer Telnet negotiations\n"
# 回答 Telnet 协商
"  -u, --udp                  Use UDP instead of default TCP\n"
# 使用 UDP 而不是默认的 TCP
"      --sctp                 Use SCTP instead of default TCP\n"
# 使用 SCTP 而不是默认的 TCP
"  -v, --verbose              Set verbosity level (can be used several times)\n"
# 设置详细程度级别（可多次使用）
"  -w, --wait <time>          Connect timeout\n"
# 连接超时
"  -z                         Zero-I/O mode, report connection status only\n"
# 零 I/O 模式，仅报告连接状态
"      --append-output        Append rather than clobber specified output files\n"
# 追加而不是覆盖指定的输出文件
"      --send-only            Only send data, ignoring received; quit on EOF\n"
# 仅发送数据，忽略接收的数据；在 EOF 时退出
# 设置接收数据但不发送数据的选项
"      --recv-only            Only receive data, never send anything\n"
# 当从标准输入接收到 EOF 时，继续半双工通信
"      --no-shutdown          Continue half-duplex when receiving EOF on stdin\n"
# 允许指定的主机连接到 Ncat
"      --allow                Allow only given hosts to connect to Ncat\n"
# 指定包含允许连接到 Ncat 的主机的文件
"      --allowfile            A file of hosts allowed to connect to Ncat\n"
# 拒绝指定的主机连接到 Ncat
"      --deny                 Deny given hosts from connecting to Ncat\n"
# 指定包含被拒绝连接到 Ncat 的主机的文件
"      --denyfile             A file of hosts denied from connecting to Ncat\n"
# 启用 Ncat 的连接代理模式
"      --broker               Enable Ncat's connection brokering mode\n"
# 启动一个简单的 Ncat 聊天服务器
"      --chat                 Start a simple Ncat chat server\n"
# 指定要代理的主机地址和端口
"      --proxy <addr[:port]>  Specify address of host to proxy through\n"
# 指定代理类型（"http", "socks4", "socks5"）
"      --proxy-type <type>    Specify proxy type (\"http\", \"socks4\", \"socks5\")\n"
# 使用 HTTP 或 SOCKS 代理服务器进行身份验证
"      --proxy-auth <auth>    Authenticate with HTTP or SOCKS proxy server\n"
# 指定在代理目的地解析的位置
"      --proxy-dns <type>     Specify where to resolve proxy destination\n"

#ifdef HAVE_OPENSSL
# 使用 SSL 进行连接或监听
"      --ssl                  Connect or listen with SSL\n"
# 指定用于监听的 SSL 证书文件（PEM 格式）
"      --ssl-cert             Specify SSL certificate file (PEM) for listening\n"
# 指定用于监听的 SSL 私钥文件（PEM 格式）
"      --ssl-key              Specify SSL private key (PEM) for listening\n"
# 验证证书的信任和域名
"      --ssl-verify           Verify trust and domain name of certificates\n"
# 包含受信任的 SSL 证书的 PEM 文件
"      --ssl-trustfile        PEM file containing trusted SSL certificates\n"
# 包含要使用的 SSL 密码的密码列表
"      --ssl-ciphers          Cipherlist containing SSL ciphers to use\n"
# 请求不同的服务器名称（SNI）
"      --ssl-servername       Request distinct server name (SNI)\n"
# 使用的 ALPN 协议列表
"      --ssl-alpn             ALPN protocol list to use\n"
#endif
# 显示 Ncat 的版本信息并退出
"      --version              Display Ncat's version information and exit\n"
/*
* 查看 ncat(1) 的 manpage 以获取完整的选项、描述和使用示例
*/
"See the ncat(1) manpage for full options, descriptions and usage examples\n"
            );
            // 退出程序并返回成功状态
            exit(EXIT_SUCCESS);
        case '?':
            /* 将未识别的参数/参数视为致命错误 */
            bye("Try `--help' or man(1) ncat for more information, usage options and help.");
        default:
            /* 我们认为未识别的选项是致命错误 */
            bye("Unrecognised option.");
        }
    }

#if HAVE_LINUX_VM_SOCKETS_H
    if (o.af == AF_VSOCK)
        max_port = UINT32_MAX;
#endif

    if (srcport > max_port)
        bye("Invalid source port %lld.", srcport);

#ifndef HAVE_OPENSSL
    if (o.ssl)
        bye("OpenSSL isn't compiled in. The --ssl option cannot be chosen.");
#endif

    if (o.normlog)
        o.normlogfd = ncat_openlog(o.normlog, o.append);
    if (o.hexlog)
        o.hexlogfd = ncat_openlog(o.hexlog, o.append);

    if (o.verbose)
        print_banner();

    if (o.debug)
        nbase_set_log(loguser, logdebug);
    else
        nbase_set_log(loguser, NULL);

#if HAVE_SYS_UN_H
    /* 使用 Unix 域套接字，因此现在进行检查 */
    if (o.af == AF_UNIX) {
        if (o.proxyaddr || o.proxytype)
            bye("Proxy option not supported when using Unix domain sockets.");
#ifdef HAVE_OPENSSL
        if (o.ssl)
            bye("SSL option not supported when using Unix domain sockets.");
#endif
        if (o.broker)
            bye("Connection brokering not supported when using Unix domain sockets.");
        if (srcport != -1)
            bye("Specifying source port when using Unix domain sockets doesn't make sense.");
        if (o.numsrcrtes > 0)
            bye("Loose source routing not allowed when using Unix domain sockets.");
    }
#endif  /* HAVE_SYS_UN_H */

#if HAVE_LINUX_VM_SOCKETS_H
    if (o.af == AF_VSOCK) {
        if (o.proxyaddr || o.proxytype)
            bye("Proxy option not supported when using vsock sockets.");
#ifdef HAVE_OPENSSL
        # 如果编译时支持 OpenSSL，则检查是否使用了 SSL 选项，如果是则报错
        if (o.ssl)
            bye("SSL option not supported when using vsock sockets.");
#endif
        # 如果使用了连接代理选项，则报错
        if (o.broker)
            bye("Connection brokering not supported when using vsock sockets.");
        # 如果使用了松散源路由选项，则报错
        if (o.numsrcrtes > 0)
            bye("Loose source routing not allowed when using vsock sockets.");
    }
#endif  /* HAVE_LINUX_VM_SOCKETS_H */

    /* 创建一个静态目标地址，因为至少必须始终分配一个目标地址 */
    targetaddrs = (struct sockaddr_list *)safe_zalloc(sizeof(struct sockaddr_list));

    /* 将 srcaddr.storage 初始化为 0，并设置地址族为 AF_UNSPEC */
    /* 当有效时，将是 AF_INET 或 AF_INET6 或 AF_UNIX */
    memset(&srcaddr.storage, 0, sizeof(srcaddr.storage));
    srcaddr.storage.ss_family = AF_UNSPEC;
    targetaddrs->addr.storage = srcaddr.storage;

    /* 清空 listenaddrs 数组 */
    int i;
    for (i = 0; i < NUM_LISTEN_ADDRS; i++) {
        listenaddrs[i].storage = srcaddr.storage;
    }
    # 如果存在代理地址
    if (o.proxyaddr) {
        # 如果代理类型为空，则设置默认为"http"
        if (!o.proxytype)
            o.proxytype = Strdup("http");

        # 验证代理类型并配置其默认端口
        if (!strcmp(o.proxytype, "http"))
            proxyport = DEFAULT_PROXY_PORT;
        else if (!strcmp(o.proxytype, "socks4") || !strcmp(o.proxytype, "4"))
            proxyport = DEFAULT_SOCKS4_PORT;
        else if (!strcmp(o.proxytype, "socks5") || !strcmp(o.proxytype, "5"))
            proxyport = DEFAULT_SOCKS5_PORT;
        else
            bye("Invalid proxy type \"%s\".", o.proxytype);

        # 解析HTTP/SOCKS代理地址并将其存储在targetss中
        # 如果代理服务器以IPv6地址（而不是主机名）给出，则必须指定端口号，否则解析将会出错
        # （因为IPv6地址中的冒号和主机:端口分隔符）
        targetaddrs->addrlen = parseproxy(o.proxyaddr,
            &targetaddrs->addr.storage, &targetaddrs->addrlen, &proxyport);
        # 如果地址族为AF_INET，则设置端口号
        if (o.af == AF_INET) {
            targetaddrs->addr.in.sin_port = htons(proxyport);
        } else { // 可能修改为else if并测试AF_{INET6|UNIX|UNSPEC}
            targetaddrs->addr.in6.sin6_port = htons(proxyport);
        }

        # 如果存在监听选项，则报错
        if (o.listen)
            bye("Invalid option combination: --proxy and -l.");
    } else {
        # 如果不存在代理地址但存在代理类型，则报错
        if (o.proxytype) {
            if (!o.listen)
                bye("Proxy type (--proxy-type) specified without proxy address (--proxy).");
            if (strcmp(o.proxytype, "http"))
                bye("Invalid proxy type \"%s\"; Ncat proxy server only supports \"http\".", o.proxytype);
        }
    }

    # 如果不存在代理认证信息，则从环境变量中获取
    if (!o.proxy_auth)
        o.proxy_auth = getenv("NCAT_PROXY_AUTH");
    # 如果设置了zerobyte选项
    if (o.zerobyte) {
        # 如果设置了listen选项，则不能和-z一起使用
        if (o.listen)
            bye("Services designed for LISTENING can't be used with -z");
        # 如果设置了telnet选项，则不能和-z一起使用
        if (o.telnet)
            bye("Invalid option combination: -z and -t.");
        # 如果设置了execmode或cmdexec选项，则不能和-z一起使用
        if (o.execmode||o.cmdexec)
            bye("Command execution can't be done along with option -z.");
        # 如果没有设置idletimeout并且协议是UDP，则设置idletimeout为2秒
        if (!o.idletimeout && o.proto == IPPROTO_UDP)
            o.idletimeout = 2 * 1000;
    }
    # 默认端口
    if (o.listen && o.proxytype && !o.portno && srcport == -1)
        o.portno = DEFAULT_PROXY_PORT;
    else
        o.portno = DEFAULT_NCAT_PORT;

    # 解析给定的源地址
    if (source) {
        int rc = 0;

        # 如果设置了listen选项，则不能和-s一起使用
        if (o.listen)
            bye("-l and -s are incompatible.  Specify the address and port to bind to like you would a host to connect to.");
    }
#if HAVE_SYS_UN_H
        /* 如果使用 UNIX sockets，只需复制路径。
         * 如果路径无效，将在后面失败！ */
        if (o.af == AF_UNIX) {
            if (o.proto == IPPROTO_UDP) {
                NCAT_INIT_SUN(&srcaddr, source);
                srcaddrlen = SUN_LEN(&srcaddr.un);
            }
            else
                if (o.verbose)
                    loguser("Specifying source socket for other than DATAGRAM Unix domain sockets have no effect.\n");
        } else
#endif
#if HAVE_LINUX_VM_SOCKETS_H
        if (o.af == AF_VSOCK) {
            long long_cid;

            srcaddr.vm.svm_family = AF_VSOCK;

            errno = 0;
            long_cid = strtol(source, NULL, 10);
            if (errno != 0 || long_cid <= 0 || long_cid > UINT32_MAX)
                bye("Invalid source address CID \"%s\".", source);
            srcaddr.vm.svm_cid = long_cid;

            srcaddrlen = sizeof(srcaddr.vm);
        } else
#endif
            rc = resolve(source, 0, &srcaddr.storage, &srcaddrlen, o.af);
        if (rc != 0)
            bye("Could not resolve source address \"%s\": %s.", source, gai_strerror(rc));
    }

    host_list_to_set(o.allowset, allow_host_list);
    host_list_free(allow_host_list);
    host_list_to_set(o.denyset, deny_host_list);
    host_list_free(deny_host_list);

    int rc;
    int num_ports = 0;
    if (srcport != -1 && o.listen) {
        /* 对于 nc 兼容性，将 "ncat -l -p <port>" 视为 "ncat -l <port>"。 */
        o.portno = (unsigned int) srcport;
        num_ports++;
    }
    /* 剩下多少个参数？ */
    ncat_assert(optind <= argc);
    switch (argc - optind) {
      case 2:
#if HAVE_SYS_UN_H
        /* 我们不使用 Unix 域套接字的端口。 */
        if (o.af == AF_UNIX) {
            bye("使用 Unix 域套接字并指定端口没有意义。");
        }
#endif
        // 如果端口数量为0，则解析端口号并赋值给o.portno
        if (num_ports == 0)
          o.portno = parseport(argv[optind + 1], max_port, "port");
        // 增加端口数量
        num_ports++;
        /* fall through: */
      case 1:
#if HAVE_SYS_UN_H
        // 如果定义了HAVE_SYS_UN_H并且o.af为AF_UNIX，则执行以下代码块
        if (o.af == AF_UNIX) {
            // 初始化UNIX域套接字地址结构
            NCAT_INIT_SUN(&targetaddrs->addr, argv[optind]);
            // 设置地址长度
            targetaddrs->addrlen = SUN_LEN(&targetaddrs->addr.un);
            // 设置SSL服务器名称和目标地址
            o.sslservername = o.target = argv[optind];
            break;
        }
#endif
#if HAVE_LINUX_VM_SOCKETS_H
        // 如果定义了HAVE_LINUX_VM_SOCKETS_H并且o.af为AF_VSOCK，则执行以下代码块
        if (o.af == AF_VSOCK) {
            long long_cid;

            // 初始化VSOCK套接字地址结构
            memset(&targetaddrs->addr.storage, 0, sizeof(struct sockaddr_vm));
            targetaddrs->addr.vm.svm_family = AF_VSOCK;

            errno = 0;
            // 将字符串转换为长整型
            long_cid = strtol(argv[optind], NULL, 10);
            // 如果转换失败或者超出范围，则报错并退出
            if (errno != 0 || long_cid <= 0 || long_cid > UINT32_MAX)
                bye("Invalid CID \"%s\".", argv[optind]);
            // 设置VSOCK套接字的CID
            targetaddrs->addr.vm.svm_cid = long_cid;

            // 设置地址长度
            targetaddrs->addrlen = sizeof(targetaddrs->addr.vm);
            // 设置SSL服务器名称和目标地址
            o.sslservername = o.target = argv[optind];
            break;
        }
#endif
        // 如果端口数量为0且o.listen为真，则执行以下代码块
        /* Support ncat -l <port>, but otherwise assume ncat <target> */
        if (num_ports == 0 && o.listen) {
            // 计算字符串中数字字符的长度
            rc = strspn(argv[optind], "1234567890");
            /* If the last arg is 5 or fewer digits, assume it's a port number */
            // 如果最后一个参数是5位或更少的数字，则假设它是一个端口号
            if (argv[optind][rc] == '\0' && rc <= 5) {
                // 解析端口号并赋值给o.portno
                o.portno = parseport(argv[optind], max_port, "port");
                // 增加端口数量
                num_ports++;
                break;
            }
        }
        // 设置目标地址
        o.target = argv[optind];
        // 如果o.proxytype为NULL，则解析主机名并填充targetaddrs，如果解析失败则报错并退出
        /* resolve hostname only if o.proxytype == NULL
         * targetss contains data already and you don't want remove them
         */
        if( !o.proxytype
                && (rc = resolve_multi(o.target, 0, targetaddrs, o.af)) != 0)

            bye("Could not resolve hostname \"%s\": %s.", o.target, gai_strerror(rc));
        // 如果SSL服务器名称为空，则设置为目标地址
        if (!o.sslservername)
            o.sslservername = o.target;
        break;
      case 0:
#if HAVE_SYS_UN_H
        # 如果系统支持 UNIX 域套接字
        if (o.af == AF_UNIX) {
            # 如果地址族是 AF_UNIX，则必须指定一个套接字的路径来监听或连接
            bye("You must specify a path to a socket to %s.",
                    o.listen ? "listen on" : "connect to");
        }
#endif
        /* Listen defaults to any address and DEFAULT_NCAT_PORT */
        # 如果不是监听模式，则必须指定要连接的主机
        if (!o.listen)
            bye("You must specify a host to connect to.");
        break;
      default:
        # 如果没有指定端口号，则解析命令行参数中的端口号
        if (num_ports == 0)
            o.portno = parseport(argv[optind + 1], max_port, "port");
        num_ports += argc - optind - 1;
        break;
    }

    # 如果指定了多个端口号，则记录日志并退出
    if (num_ports > 1) {
        loguser("Got more than one port specification: %u", o.portno);
        for (rc = argc - num_ports + 1; rc < argc; rc++)
            loguser_noprefix(" %s", argv[rc]);
        loguser_noprefix(". QUITTING.\n");
        exit(2);
    }

    # 如果使用代理类型并且不是监听模式，则不做任何操作，端口号已经设置为代理端口
    if (o.proxytype && !o.listen)
        ; /* Do nothing - port is already set to proxyport  */
    else {
        # 遍历目标地址列表，根据地址族设置端口号
        struct sockaddr_list *targetaddrs_item = targetaddrs;
        while (targetaddrs_item != NULL)
        {
            # 如果地址族是 AF_INET，则设置端口号
            if (targetaddrs_item->addr.storage.ss_family == AF_INET)
                targetaddrs_item->addr.in.sin_port = htons(o.portno);
#ifdef HAVE_IPV6
            # 如果地址族是 AF_INET6，则设置端口号
            else if (targetaddrs_item->addr.storage.ss_family == AF_INET6)
                targetaddrs_item->addr.in6.sin6_port = htons(o.portno);
#endif
#if HAVE_SYS_UN_H
            /* If we use Unix domain sockets, we have to count with them. */
            # 如果使用 UNIX 域套接字，则不做任何操作
            else if (targetaddrs_item->addr.storage.ss_family == AF_UNIX)
                ; /* Do nothing. */
#endif
#if HAVE_LINUX_VM_SOCKETS_H
            # 如果地址族是 AF_VSOCK，则设置端口号
            else if (targetaddrs_item->addr.storage.ss_family == AF_VSOCK)
                targetaddrs_item->addr.vm.svm_port = o.portno;
#else
            else if (targetaddrs_item->addr.storage.ss_family == AF_UNSPEC)
                ; /* Leave unspecified. */
            else
                bye("Unknown address family %d.", targetaddrs_item->addr.storage.ss_family);
            targetaddrs_item = targetaddrs_item->next;
        }
    }

    if (srcport != -1 && !o.listen) {
            if (srcaddr.storage.ss_family == AF_UNSPEC) {
                /* We have a source port but not an explicit source address;
                   fill in an unspecified address of the same family as the
                   target. */
                srcaddr.storage.ss_family = targetaddrs->addr.storage.ss_family;
                if (srcaddr.storage.ss_family == AF_INET)
                    srcaddr.in.sin_addr.s_addr = INADDR_ANY;
                else if (srcaddr.storage.ss_family == AF_INET6)
                    srcaddr.in6.sin6_addr = in6addr_any;
            }
            if (srcaddr.storage.ss_family == AF_INET)
                srcaddr.in.sin_port = htons((unsigned int) srcport);
#ifdef HAVE_IPV6
            else if (srcaddr.storage.ss_family == AF_INET6)
                srcaddr.in6.sin6_port = htons((unsigned int) srcport);
#endif
#ifdef HAVE_LINUX_VM_SOCKETS_H
            else if (srcaddr.storage.ss_family == AF_VSOCK)
                srcaddr.vm.svm_port = (unsigned int) srcport;
#endif
    }

    if (o.proto == IPPROTO_UDP) {

#ifndef HAVE_DTLS_CLIENT_METHOD
        if (o.ssl)
            bye("OpenSSL does not have DTLS support compiled in.");
#endif
    }

    /* Do whatever is necessary to receive \n for line endings on input from
       the console. A no-op on Unix. */
    set_lf_mode();

#ifdef HAVE_LUA
    if (o.execmode == EXEC_LUA)
        lua_setup();
#endif

    if (o.listen)
        return ncat_listen_mode();
    else
        return ncat_connect_mode();
}

/* connect error handling and operations. */
static int ncat_connect_mode(void)
    /*
     * 允许/拒绝使用 connect 命令毫无意义。如果你不想连接到主机，就不要尝试连接。
     */
    如果 (o.allow || o.deny)
        bye("Invalid option combination: allow/deny with connect.");

    /* o.conn_limit 与 'connect' 没有任何意义。 */
    如果 (o.conn_limit != -1)
        bye("Invalid option combination: `--max-conns' with connect.");

    如果 (o.chat)
        bye("Invalid option combination: `--chat' with connect.");

    如果 (o.keepopen)
        bye("Invalid option combination: `--keep-open' with connect.");

    返回 ncat_connect();
static int ncat_listen_mode(void)
{
    /* 不能同时在代理服务器上“监听”和“连接” */
    if (o.proxyaddr != NULL)
        bye("Invalid option combination: --proxy and -l.");

    /* 不能同时使用--broker和-e选项 */
    if (o.broker && o.cmdexec != NULL)
        bye("Invalid option combination: --broker and -e.");

    /* 如果使用--proxytype选项并且使用--telnet选项，则--telnet选项无效 */
    if (o.proxytype != NULL && o.telnet)
        bye("Invalid option combination: --telnet has no effect with --proxy-type.");

    /* 如果设置了最大连接数限制并且没有使用-k或--broker选项，则忽略最大连接数限制 */
    if (o.conn_limit != -1 && !(o.keepopen || o.broker))
        loguser("Warning: Maximum connections ignored, since it does not take "
                "effect without -k or --broker.\n");

    /* 设置默认的最大同时TCP连接限制 */
    if (o.conn_limit == -1)
        o.conn_limit = DEFAULT_MAX_CONNS;

    /* 在深入处理之前，查看shell是否可执行 */
#ifndef WIN32
    if (o.execmode == EXEC_SHELL && access("/bin/sh", X_OK) == -1)
        bye("/bin/sh is not executable, so `-c' won't work.");
#endif

    /* 如果目标地址的地址族不是AF_UNSPEC，则将其添加到监听地址列表中 */
    if (targetaddrs->addr.storage.ss_family != AF_UNSPEC) {
        listenaddrs[num_listenaddrs++] = targetaddrs->addr;
    } else {
        size_t ss_len;
        int rc;

        /* 没有命令行地址。监听IPv4或IPv6或两者都监听 */
        /* 首先尝试绑定到IPv6；在AIX上，绑定的IPv4套接字会阻塞相同端口上的IPv6套接字，尽管设置了IPV6_V6ONLY。 */
#ifdef HAVE_IPV6
        if (o.af == AF_INET6 || o.af == AF_UNSPEC) {
            ss_len = sizeof(listenaddrs[num_listenaddrs]);
            rc = resolve("::", o.portno, &listenaddrs[num_listenaddrs].storage, &ss_len, AF_INET6);
            if (rc == 0)
                num_listenaddrs++;
            else if (o.debug > 0)
                logdebug("Failed to resolve default IPv6 address: %s\n", gai_strerror(rc));
        }
#endif
        // 如果地址族是 IPv4 或未指定，则解析默认的 IPv4 地址并添加到监听地址列表中
        if (o.af == AF_INET || o.af == AF_UNSPEC) {
            ss_len = sizeof(listenaddrs[num_listenaddrs]);
            rc = resolve("0.0.0.0", o.portno, &listenaddrs[num_listenaddrs].storage, &ss_len, AF_INET);
            if (rc != 0)
                bye("Failed to resolve default IPv4 address: %s.", gai_strerror(rc));
            num_listenaddrs++;
        }
#ifdef HAVE_LINUX_VM_SOCKETS_H
        // 如果地址族是 VSOCK，则设置 VSOCK 地址并添加到监听地址列表中
        if (o.af == AF_VSOCK) {
            listenaddrs[num_listenaddrs].vm.svm_family = AF_VSOCK;
            listenaddrs[num_listenaddrs].vm.svm_cid = VMADDR_CID_ANY;
            listenaddrs[num_listenaddrs].vm.svm_port = o.portno;
            num_listenaddrs++;
        }
#endif
    }

    // 如果设置了代理类型，并且代理类型是 "http"，则将 httpserver 标记设置为 1
    if (o.proxytype) {
        if (strcmp(o.proxytype, "http") == 0)
            o.httpserver = 1;
    }

    /* 触发监听/选择调度程序以进行标准的监听操作。 */
    // 调用 ncat_listen() 函数进行监听操作
    return ncat_listen();
}
```