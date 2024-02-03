# `nmap\libpcap\pcap-sita.c`

```cpp
/*
 *  pcap-sita.c: SITA ACN 设备的数据包捕获接口补充
 *
 *  版权所有 (c) 2007 Fulko Hew, SITA INC Canada, Inc <fulko.hew@sita.aero>
 *
 *  许可证: BSD
 *
 *  在源代码和二进制形式下的再发布和使用，无论是否经过修改，都是允许的，只要满足以下条件:
 *
 *  1. 源代码的再发布必须保留上述版权声明、此条件列表和以下免责声明。
 *  2. 二进制形式的再发布必须在文档和/或其他提供的材料中重现上述版权声明、此条件列表和以下免责声明。
 *  3. 未经特定书面许可，不得使用作者的名称来认可或推广从本软件衍生的产品。
 *
 *  本软件按原样提供，没有任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <sys/time.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include "pcap-int.h"

#include "pcap-sita.h"

    /* 不可配置的常量定义 */

#define IOP_SNIFFER_PORT    49152            /* IOP 上用于 '分布式 pcap' 使用的 TCP 端口 */
#define MAX_LINE_SIZE        255                /* /etc/hosts 中缓冲区/行的最大大小 */
#define MAX_CHASSIS            8                /* ACN 站点中机箱的最大数量 */
#define MAX_GEOSLOT            8                /* ACN 站点中访问单元的最大数量 */

#define FIND            0
#define LIVE            1

typedef struct iface {
    struct iface    *next;        /* 指向下一个接口的指针 */
    # 定义一个指向字符串的指针，用于存储接口的名称
    char        *name;        /* this interface's name */
    # 定义一个指向字符串的指针，用于存储接口在IOP上的名称
    char        *IOPname;    /* this interface's name on an IOP */
    # 定义一个32位的整数，用于存储接口的类型（DLT值）
    uint32_t    iftype;        /* the type of interface (DLT values) */
} iface_t;

typedef struct unit {
    char            *ip;        /* this unit's IP address (as extracted from /etc/hosts) */
    int            fd;        /* the connection to this unit (if it exists) */
    int            find_fd;    /* a big kludge to avoid my programming limitations since I could have this unit open for findalldevs purposes */
    int            first_time;    /* 0 = just opened via acn_open_live(),  ie. the first time, NZ = nth time */
    struct sockaddr_in    *serv_addr;    /* the address control block for comms to this unit */
    int            chassis;
    int            geoslot;
    iface_t            *iface;        /* a pointer to a linked list of interface structures */
    char            *imsg;        /* a pointer to an inbound message */
    int            len;        /* the current size of the inbound message */
} unit_t;

/*
 * Private data.
 * Currently contains nothing.
 */
struct pcap_sita {
    int    dummy;
};

static unit_t        units[MAX_CHASSIS+1][MAX_GEOSLOT+1];    /* we use indexes of 1 through 8, but we reserve/waste index 0 */
static fd_set        readfds;                /* a place to store the file descriptors for the connections to the IOPs */
static int        max_fs;

pcap_if_t        *acn_if_list;        /* pcap's list of available interfaces */

static void dump_interface_list(void) {
    pcap_if_t        *iff;
    pcap_addr_t        *addr;
    int            longest_name_len = 0;
    char            *n, *d, *f;
    int            if_number = 0;

    iff = acn_if_list;
    while (iff) {
        if (iff->name && (strlen(iff->name) > longest_name_len)) longest_name_len = strlen(iff->name);  // 计算最长接口名的长度
        iff = iff->next;
    }
    iff = acn_if_list;
    printf("Interface List:\n");  // 打印接口列表
    // 当 iff 不为空时，执行循环
    while (iff) {
        // 如果 iff 的 name 不为空，则将其赋给 n，否则赋空字符串
        n = (iff->name)                            ? iff->name            : "";
        // 如果 iff 的 description 不为空，则将其赋给 d，否则赋空字符串
        d = (iff->description)                    ? iff->description    : "";
        // 如果 iff 的 flags 等于 PCAP_IF_LOOPBACK，则将 "L" 赋给 f，否则赋空字符串
        f = (iff->flags == PCAP_IF_LOOPBACK)    ? "L"                : "";
        // 打印接口的编号、名称、标志和描述
        printf("%3d: %*s %s '%s'\n", if_number++, longest_name_len, n, f, d);
        // 获取接口的地址
        addr = iff->addresses;
        // 遍历接口的地址
        while (addr) {
            // 添加一些缩进
            printf("%*s ", (5 + longest_name_len), "");
            // 打印地址、子网掩码、广播地址和目的地址
            printf("%15s  ", (addr->addr)        ? inet_ntoa(((struct sockaddr_in *)addr->addr)->sin_addr)        : "");
            printf("%15s  ", (addr->netmask)    ? inet_ntoa(((struct sockaddr_in *)addr->netmask)->sin_addr)    : "");
            printf("%15s  ", (addr->broadaddr)    ? inet_ntoa(((struct sockaddr_in *)addr->broadaddr)->sin_addr)    : "");
            printf("%15s  ", (addr->dstaddr)    ? inet_ntoa(((struct sockaddr_in *)addr->dstaddr)->sin_addr)    : "");
            printf("\n");
            // 移动到下一个地址
            addr = addr->next;
        }
        // 移动到下一个接口
        iff = iff->next;
    }
# 打印十六进制数据
static void dump(unsigned char *ptr, int i, int indent) {
    # 打印指定缩进的空格
    fprintf(stderr, "%*s", indent, " ");
    # 遍历指定长度的数据，以十六进制形式打印
    for (; i > 0; i--) {
        fprintf(stderr, "%2.2x ", *ptr++);
    }
    # 打印换行符
    fprintf(stderr, "\n");
}

# 打印接口列表信息
static void dump_interface_list_p(void) {
    pcap_if_t        *iff;
    pcap_addr_t        *addr;
    int                if_number = 0;

    # 初始化接口指针
    iff = acn_if_list;
    # 打印接口指针的地址和值
    printf("Interface Pointer @ %p is %p:\n", &acn_if_list, iff);
    # 遍历接口列表
    while (iff) {
        # 打印接口的名称、描述和下一个接口的地址
        printf("%3d: %p %p next: %p\n", if_number++, iff->name, iff->description, iff->next);
        # 调用 dump 函数打印接口的数据
        dump((unsigned char *)iff, sizeof(pcap_if_t), 5);
        # 初始化地址指针
        addr = iff->addresses;
        # 遍历接口的地址列表
        while (addr) {
            # 打印地址、子网掩码、广播地址、目标地址和下一个地址的地址
            printf("          %p %p %p %p, next: %p\n", addr->addr, addr->netmask, addr->broadaddr, addr->dstaddr, addr->next);
            # 调用 dump 函数打印地址的数据
            dump((unsigned char *)addr, sizeof(pcap_addr_t), 10);
            # 移动到下一个地址
            addr = addr->next;
        }
        # 移动到下一个接口
        iff = iff->next;
    }
}

# 打印单元表信息
static void dump_unit_table(void) {
    int        chassis, geoslot;
    iface_t    *p;

    # 打印表头
    printf("%c:%c %s %s\n", 'C', 'S', "fd", "IP Address");
    # 遍历单元表
    for (chassis = 0; chassis <= MAX_CHASSIS; chassis++) {
        for (geoslot = 0; geoslot <= MAX_GEOSLOT; geoslot++) {
            # 如果单元的 IP 地址不为空，打印单元的信息
            if (units[chassis][geoslot].ip != NULL)
                printf("%d:%d %2d %s\n", chassis, geoslot, units[chassis][geoslot].fd, units[chassis][geoslot].ip);
            # 初始化接口指针
            p = units[chassis][geoslot].iface;
            # 遍历接口列表
            while (p) {
                # 获取接口的名称和 IOP 名称，打印接口的连接关系
                char *n = (p->name)            ? p->name            : "";
                char *i = (p->IOPname)        ? p->IOPname        : "";
                p = p->next;
                printf("   %12s    -> %12s\n", i, n);
            }
        }
    }
}

# 根据文件描述符查找单元
static int find_unit_by_fd(int fd, int *chassis, int *geoslot, unit_t **unit_ptr) {
    int        c, s;
    # 遍历所有的底盘和槽位
    for (c = 0; c <= MAX_CHASSIS; c++) {
        for (s = 0; s <= MAX_GEOSLOT; s++) {
            # 如果当前单元的文件描述符等于给定的文件描述符，或者查找到的文件描述符等于给定的文件描述符
            if (units[c][s].fd == fd || units[c][s].find_fd == fd) {
                # 如果传入了底盘指针，将当前底盘的值赋给它
                if (chassis)    *chassis = c;
                # 如果传入了槽位指针，将当前槽位的值赋给它
                if (geoslot)    *geoslot = s;
                # 如果传入了单元指针，将当前单元的地址赋给它
                if (unit_ptr)    *unit_ptr = &units[c][s];
                # 返回 1，表示找到了对应的底盘、槽位和单元
                return 1;
            }
        }
    }
    # 如果没有找到对应的底盘、槽位和单元，返回 0
    return 0;
static int read_client_nbytes(int fd, int count, unsigned char *buf) {
    unit_t            *u;  // 定义指向 unit_t 结构体的指针 u
    int                chassis, geoslot;  // 定义整型变量 chassis 和 geoslot
    int                len;  // 定义整型变量 len

    find_unit_by_fd(fd, &chassis, &geoslot, &u);  // 根据文件描述符查找对应的单元，并将 chassis、geoslot 和 u 的值设置为对应的结果
    while (count) {  // 进入循环，条件为 count 不为 0
        if ((len = recv(fd, buf, count, 0)) <= 0)    return -1;    /* read in whatever data was sent to us */  // 如果接收到的数据长度小于等于 0，则返回 -1；否则将接收到的数据存入 buf 中
        count -= len;  // 将 count 减去已经接收到的数据长度
        buf += len;  // 将 buf 指针向后移动 len 个位置
    }                                                            /* till we have everything we are looking for */  // 直到我们获得了我们要找的所有数据
    return 0;  // 返回 0
}

static void empty_unit_iface(unit_t *u) {
    iface_t    *p, *cur;  // 定义指向 iface_t 结构体的指针 p 和 cur

    cur = u->iface;  // 将 u 的 iface 成员赋值给 cur
    while (cur) {                                            /* loop over all the interface entries */  // 循环遍历所有接口条目
        if (cur->name)            free(cur->name);            /* throwing away the contents if they exist */  // 如果 cur 的 name 成员存在，则释放其内存
        if (cur->IOPname)        free(cur->IOPname);  // 如果 cur 的 IOPname 成员存在，则释放其内存
        p = cur->next;  // 将 cur 的 next 成员赋值给 p
        free(cur);                                            /* then throw away the structure itself */  // 释放 cur 结构体本身的内存
        cur = p;  // 将 p 的值赋给 cur
    }
    u->iface = 0;                                            /* and finally remember that there are no remaining structure */  // 最后记住没有剩余的结构
}

static void empty_unit(int chassis, int geoslot) {
    unit_t    *u = &units[chassis][geoslot];  // 定义指向 unit_t 结构体的指针 u，并将其指向 units 数组中指定位置的元素

    empty_unit_iface(u);  // 调用 empty_unit_iface 函数，传入 u 指针
    if (u->imsg) {                                            /* then if an inbound message buffer exists */  // 如果存在入站消息缓冲区
        void *bigger_buffer;

        bigger_buffer = (char *)realloc(u->imsg, 1);                /* and re-allocate the old large buffer into a new small one */  // 将旧的大缓冲区重新分配为新的小缓冲区
        if (bigger_buffer == NULL) {    /* oops, realloc call failed */  // 如果重新分配内存失败
            fprintf(stderr, "Warning...call to realloc() failed, value of errno is %d\n", errno);  // 输出警告信息和错误码
            return;  // 返回
        }
        u->imsg = bigger_buffer;  // 将 imsg 指针指向新的缓冲区
    }
}

static void empty_unit_table(void) {
    int        chassis, geoslot;
    # 遍历所有机箱和槽位
    for (chassis = 0; chassis <= MAX_CHASSIS; chassis++) {
        for (geoslot = 0; geoslot <= MAX_GEOSLOT; geoslot++) {
            # 如果单元的 IP 地址不为空
            if (units[chassis][geoslot].ip != NULL) {
                # 释放存储 IP 地址的内存空间
                free(units[chassis][geoslot].ip);            /* get rid of the malloc'ed space that holds the IP address */
                # 将指针设置为 NULL
                units[chassis][geoslot].ip = 0;                /* then set the pointer to NULL */
            }
            # 清空机箱和槽位上的单元
            empty_unit(chassis, geoslot);
        }
    }
# 查找第n个接口的名称
static char *find_nth_interface_name(int n) {
    int        chassis, geoslot;  # 定义变量chassis和geoslot
    iface_t    *p;  # 定义指向iface_t类型的指针p
    char    *last_name = 0;  # 定义指向字符类型的指针last_name，并初始化为0

    if (n < 0) n = 0;  # 如果n小于0，则将n设置为0，确保使用有效的数字
    for (chassis = 0; chassis <= MAX_CHASSIS; chassis++) {  # 循环遍历chassis
        for (geoslot = 0; geoslot <= MAX_GEOSLOT; geoslot++) {  # 循环遍历geoslot
            if (units[chassis][geoslot].ip != NULL) {  # 如果units[chassis][geoslot].ip不为空
                p = units[chassis][geoslot].iface;  # 将units[chassis][geoslot].iface赋值给p
                while (p) {  # 进入循环，遍历所有接口
                    if (p->IOPname) last_name = p->name;  # 如果p->IOPname不为空，则将p->name赋值给last_name，记住找到的最后一个名称
                    if (n-- == 0) return last_name;  # 如果n减到0，则返回last_name，表示找到了请求的实例
                    p = p->next;  # 将p指向下一个接口
                }
            }
        }
    }
    if (last_name)    return last_name;  # 如果找不到选定的条目，但至少有一个条目，则返回找到的最后一个条目
    return "";  # 如果没有任何条目，则返回空字符串
}

# 解析hosts文件
int acn_parse_hosts_file(char *errbuf) {  # 返回值：-1 = 错误，0 = 正常
    FILE    *fp;  # 定义文件指针fp
    char    buf[MAX_LINE_SIZE];  # 定义字符数组buf
    char    *ptr, *ptr2;  # 定义指向字符类型的指针ptr和ptr2
    int        pos;  # 定义变量pos
    int        chassis, geoslot;  # 定义变量chassis和geoslot
    unit_t    *u;  # 定义指向unit_t类型的指针u

    empty_unit_table();  # 清空unit表
    if ((fp = fopen("/etc/hosts", "r")) == NULL) {  # 尝试打开hosts文件，如果失败
        snprintf(errbuf, PCAP_ERRBUF_SIZE, "Cannot open '/etc/hosts' for reading.");  # 返回nohostsfile错误响应
        return -1;
    }
    fclose(fp);  # 关闭文件
    if (*errbuf)    return -1;  # 如果errbuf不为空，则返回-1
    else            return 0;  # 否则返回0
}

# 使用IOP打开
static int open_with_IOP(unit_t  *u, int flag) {
    int                    sockfd;  # 定义变量sockfd
    char                *ip;  # 定义指向字符类型的指针ip
    # 如果服务地址为空
    if (u->serv_addr == NULL) {
        # 分配内存给服务地址结构体
        u->serv_addr = malloc(sizeof(struct sockaddr_in));

        # 检查是否成功分配内存
        if (u->serv_addr == NULL) {    # 如果内存分配失败
            # 输出错误信息和错误码
            fprintf(stderr, "malloc() request for u->serv_addr failed, value of errno is: %d\n", errno);
            # 返回失败
            return 0;
        }

    }
    # 获取 IP 地址
    ip = u->ip;
    # 使用 memset() 初始化服务地址结构体
    memset((char *)u->serv_addr, 0, sizeof(struct sockaddr_in));
    # 设置服务地址结构体的属性
    u->serv_addr->sin_family        = AF_INET;
    u->serv_addr->sin_addr.s_addr    = inet_addr(ip);
    u->serv_addr->sin_port            = htons(IOP_SNIFFER_PORT);

    # 创建套接字
    if ((sockfd = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
        # 输出错误信息
        fprintf(stderr, "pcap can't open a socket for connecting to IOP at %s\n", ip);
        # 返回失败
        return 0;
    }
    # 连接套接字
    if (connect(sockfd, (struct sockaddr *)u->serv_addr, sizeof(struct sockaddr_in)) < 0) {
        # 输出错误信息
        fprintf(stderr, "pcap can't connect to IOP at %s\n", ip);
        # 返回失败
        return 0;
    }
    # 根据标志设置文件描述符
    if (flag == LIVE)    u->fd = sockfd;
    else                u->find_fd = sockfd;
    # 设置第一次连接标志
    u->first_time = 0;
    # 返回套接字文件描述符作为成功指示
    return sockfd;            
# 关闭具有IOP的连接
static void close_with_IOP(int chassis, int geoslot, int flag) {
    int        *id;

    if (flag == LIVE)    id = &units[chassis][geoslot].fd;  # 如果标志是LIVE，则将id指向对应单元的文件描述符
    else                id = &units[chassis][geoslot].find_fd;  # 否则将id指向对应单元的查找文件描述符

    if (*id) {                                        /* 如果存在文件描述符... */
        close(*id);                                    /* 关闭文件描述符 */
        *id = 0;                                    /* 将文件描述符置为0，表示连接已关闭 */
    }
}

# 清理ACN
static void pcap_cleanup_acn(pcap_t *handle) {
    int        chassis, geoslot;
    unit_t    *u;

    if (find_unit_by_fd(handle->fd, &chassis, &geoslot, &u) == 0)  # 如果找不到对应的单元，则返回
        return;
    close_with_IOP(chassis, geoslot, LIVE);  # 关闭具有IOP的连接
    if (u)
        u->first_time = 0;  # 将u的first_time属性置为0
    pcap_cleanup_live_common(handle);  # 清理live common
}

# 发送到文件描述符
static void send_to_fd(int fd, int len, unsigned char *str) {
    int        nwritten;
    int        chassis, geoslot;

    while (len > 0) {
        if ((nwritten = write(fd, str, len)) <= 0) {  # 如果写入失败
            find_unit_by_fd(fd, &chassis, &geoslot, NULL);  # 查找对应的单元
            if (units[chassis][geoslot].fd == fd)            close_with_IOP(chassis, geoslot, LIVE);  # 如果文件描述符是fd，则关闭具有IOP的连接
            else if (units[chassis][geoslot].find_fd == fd)    close_with_IOP(chassis, geoslot, FIND);  # 如果查找文件描述符是fd，则关闭具有IOP的连接
            empty_unit(chassis, geoslot);  # 清空对应的单元
            return;
        }
        len -= nwritten;  # 减去已写入的长度
        str += nwritten;  # 更新字符串指针
    }
}

# 释放所有设备
static void acn_freealldevs(void) {

    pcap_if_t    *iff, *next_iff;
    pcap_addr_t    *addr, *next_addr;
    # 遍历网络接口列表
    for (iff = acn_if_list; iff != NULL; iff = next_iff) {
        # 保存下一个网络接口的指针，以便在释放当前接口后继续遍历
        next_iff = iff->next;
        # 遍历当前网络接口的地址列表
        for (addr = iff->addresses; addr != NULL; addr = next_addr) {
            # 保存下一个地址的指针，以便在释放当前地址后继续遍历
            next_addr = addr->next;
            # 释放当前地址的 IP 地址
            if (addr->addr)            free(addr->addr);
            # 释放当前地址的子网掩码
            if (addr->netmask)        free(addr->netmask);
            # 释放当前地址的广播地址
            if (addr->broadaddr)    free(addr->broadaddr);
            # 释放当前地址的目标地址
            if (addr->dstaddr)        free(addr->dstaddr);
            # 释放当前地址结构体
            free(addr);
        }
        # 释放当前网络接口的名称
        if (iff->name)            free(iff->name);
        # 释放当前网络接口的描述
        if (iff->description)    free(iff->description);
        # 释放当前网络接口结构体
        free(iff);
    }
    # 为非统一 IOP 端口生成端口名
    static void nonUnified_IOP_port_name(char *buf, size_t bufsize, const char *proto, unit_t *u) {
        # 使用 snprintf 格式化生成端口名
        snprintf(buf, bufsize, "%s_%d_%d", proto, u->chassis, u->geoslot);
    }

    # 为统一 IOP 端口生成端口名
    static void unified_IOP_port_name(char *buf, size_t bufsize, const char *proto, unit_t *u, int IOPportnum) {
        int            portnum;

        # 计算端口号
        portnum = ((u->chassis - 1) * 64) + ((u->geoslot - 1) * 8) + IOPportnum + 1;
        # 使用 snprintf 格式化生成端口名
        snprintf(buf, bufsize, "%s_%d", proto, portnum);
    }

    # 将 IOP 名称转换为 pcap 名称
    static char *translate_IOP_to_pcap_name(unit_t *u, char *IOPname, bpf_u_int32 iftype) {
        iface_t        *iface_ptr, *iface;
        char        buf[32];
        char        *proto;
        char        *port;
        int            IOPportnum = 0;

        # 为 iface 结构分配内存
        iface = malloc(sizeof(iface_t));
        # 检查内存分配是否成功
        if (iface == NULL) {
            # 打印错误信息并返回空指针
            fprintf(stderr, "Error...couldn't allocate memory for interface structure...value of errno is: %d\n", errno);
            return NULL;
        }
        # 使用 memset 清空分配的内存
        memset((char *)iface, 0, sizeof(iface_t));

        # 记录接口类型
        iface->iftype = iftype;

        # 复制 IOP 名称到 iface 结构
        iface->IOPname = strdup(IOPname);
        # 检查内存分配是否成功
        if (iface->IOPname == NULL) {
            # 打印错误信息并返回空指针
            fprintf(stderr, "Error...couldn't allocate memory for IOPname...value of errno is: %d\n", errno);
            return NULL;
        }

        # 检查 IOP 名称是否为 "lo"
        if (strncmp(IOPname, "lo", 2) == 0) {
            # 提取端口号
            IOPportnum = atoi(&IOPname[2]);
            # 根据接口类型选择生成端口名的函数
            switch (iftype) {
                case DLT_EN10MB:
                    nonUnified_IOP_port_name(buf, sizeof buf, "lo", u);
                    break;
                default:
                    unified_IOP_port_name(buf, sizeof buf, "???", u, IOPportnum);
                    break;
            }
        }
    }
    } else if (strncmp(IOPname, "eth", 3) == 0) {  # 如果 IOP 名称以 "eth" 开头
        IOPportnum = atoi(&IOPname[3]);  # 获取端口号
        switch (iftype) {  # 根据网络接口类型进行判断
            case DLT_EN10MB:  # 如果是以太网类型
                nonUnified_IOP_port_name(buf, sizeof buf, "eth", u);  # 调用非统一的 IOP 端口名称函数
                break;
            default:
                unified_IOP_port_name(buf, sizeof buf, "???", u, IOPportnum);  # 调用统一的 IOP 端口名称函数
                break;
        }
    } else if (strncmp(IOPname, "wan", 3) == 0) {  # 如果 IOP 名称以 "wan" 开头
        IOPportnum = atoi(&IOPname[3]);  # 获取端口号
        switch (iftype) {  # 根据网络接口类型进行判断
            case DLT_SITA:  # 如果是 SITA 类型
                unified_IOP_port_name(buf, sizeof buf, "wan", u, IOPportnum);  # 调用统一的 IOP 端口名称函数
                break;
            default:
                unified_IOP_port_name(buf, sizeof buf, "???", u, IOPportnum);  # 调用统一的 IOP 端口名称函数
                break;
        }
    } else {  # 如果不是以上情况
        fprintf(stderr, "Error... invalid IOP name %s\n", IOPname);  # 输出错误信息
        return NULL;  # 返回空指针
    }

    iface->name = strdup(buf);  /* 复制字符串并将其放入结构体中 */
        if (iface->name == NULL) {  /* 如果内存分配失败 */
                fprintf(stderr, "Error...couldn't allocate memory for IOP port name...value of errno is: %d\n", errno);  # 输出错误信息
                return NULL;  # 返回空指针
        }

    if (u->iface == 0) {  /* 如果这是第一个名称 */
        u->iface = iface;  /* 将此条目放在列表的开头 */
    } else {
        iface_ptr = u->iface;
        while (iface_ptr->next) {  /* 否则扫描列表 */
            iface_ptr = iface_ptr->next;  /* 直到到达最后一个条目 */
        }
        iface_ptr->next = iface;  /* 然后将此条目附加到列表的末尾 */
    }
    return iface->name;  # 返回接口名称
# 比较两个字符串是否需要排序的函数
static int if_sort(char *s1, char *s2) {
    char    *s1_p2, *s2_p2;  # 定义指向字符串中下划线后部分的指针
    char    str1[MAX_LINE_SIZE], str2[MAX_LINE_SIZE];  # 定义存储字符串的数组
    int        s1_p1_len, s2_p1_len;  # 定义字符串前半部分的长度
    int        retval;  # 定义返回值

    if ((s1_p2 = strchr(s1, '_'))) {    # 如果在字符串中找到下划线...
        s1_p1_len = s1_p2 - s1;            # 计算前半部分的长度
        s1_p2++;                        # 下划线后部分的指针指向下一个字符
    } else {                            # 否则...
        s1_p1_len = strlen(s1);            # 前半部分的长度为整个字符串的长度
        s1_p2 = 0;                        # 后半部分为空
    }
    if ((s2_p2 = strchr(s2, '_'))) {    # 对第二个字符串做同样的处理
        s2_p1_len = s2_p2 - s2;
        s2_p2++;
    } else {
        s2_p1_len = strlen(s2);
        s2_p2 = 0;
    }
    # 将字符串复制到数组中，并在末尾添加结束符
    strncpy(str1, s1, (s1_p1_len > sizeof(str1)) ? s1_p1_len : sizeof(str1));   *(str1 + s1_p1_len) = 0;
    strncpy(str2, s2, (s2_p1_len > sizeof(str2)) ? s2_p1_len : sizeof(str2));   *(str2 + s2_p1_len) = 0;
    retval = strcmp(str1, str2);  # 比较两个前半部分的字符串
    if (retval != 0) return retval;        # 如果它们不相同，则返回比较结果
    return strcmp(s1_p2, s2_p2);        # 否则返回比较后半部分的结果
}

# 对接口表进行排序的函数
static void sort_if_table(void) {
    pcap_if_t    *p1, *p2, *prev, *temp;  # 定义指向接口的指针
    int            has_swapped;  # 定义交换标志

    if (!acn_if_list) return;                # 如果列表为空，则无需排序
    # 进入无限循环，直到条件为假
    while (1) {
        # 设置 p1 指向 acn_if_list 的头部
        p1 = acn_if_list;                    /* start at the head of the list */
        # 初始化 prev 为 0
        prev = 0;
        # 初始化标志位 has_swapped 为 0
        has_swapped = 0;
        # 进入内层循环，直到 p1 指向的节点没有下一个节点
        while ((p2 = p1->next)) {
            # 如果 p1 的名称比 p2 的名称大
            if (if_sort(p1->name, p2->name) > 0) {
                # 如果 prev 不为空，表示交换的节点不在链表头部
                if (prev) {                    /* we are swapping things that are _not_ at the head of the list */
                    # 交换 p1 和 p2 节点的位置
                    temp = p2->next;
                    prev->next = p2;
                    p2->next = p1;
                    p1->next = temp;
                } else {                    /* special treatment if we are swapping with the head of the list */
                    # 如果交换的节点在链表头部，特殊处理
                    temp = p2->next;
                    acn_if_list= p2;
                    p2->next = p1;
                    p1->next = temp;
                }
                # 更新 p1 和 prev 的值
                p1 = p2;
                prev = p1;
                # 设置标志位表示已经交换过
                has_swapped = 1;
            }
            # 更新 prev 和 p1 的值
            prev = p1;
            p1 = p1->next;
        }
        # 如果没有发生交换，退出循环
        if (has_swapped == 0)
            return;
    }
    # 返回空
    return;
# 处理客户端数据的函数，返回-1表示错误，返回0表示正常
static int process_client_data (char *errbuf) {                                /* returns: -1 = error, 0 = OK */
    # 定义变量：chassis, geoslot
    int                    chassis, geoslot;
    # 定义指针变量：u, iff, prev_iff, addr, prev_addr, ptr, s, newname, bigger_buffer
    unit_t                *u;
    pcap_if_t            *iff, *prev_iff;
    pcap_addr_t            *addr, *prev_addr;
    char                *ptr;
    int                    address_count;
    struct sockaddr_in    *s;
    char                *newname;
    bpf_u_int32                interfaceType;
    unsigned char        flags;
    void *bigger_buffer;

    # 初始化 prev_iff
    prev_iff = 0;
    # 返回0表示正常
    return 0;
}

# 读取客户端数据的函数
static int read_client_data (int fd) {
    # 定义变量：buf, chassis, geoslot, u, len
    unsigned char    buf[256];
    int                chassis, geoslot;
    unit_t            *u;
    int                len;

    # 根据文件描述符查找单元
    find_unit_by_fd(fd, &chassis, &geoslot, &u);

    # 从套接字中接收数据
    if ((len = recv(fd, buf, sizeof(buf), 0)) <= 0)    return 0;    /* read in whatever data was sent to us */

    # 重新分配内存，扩展缓冲区
    if ((u->imsg = realloc(u->imsg, (u->len + len))) == NULL)    /* extend the buffer for the new data */
        return 0;
    # 将新数据追加到缓冲区
    memcpy((u->imsg + u->len), buf, len);                        /* append the new data */
    u->len += len;
    return 1;
}

# 等待所有答复的函数
static void wait_for_all_answers(void) {
    # 定义变量：retval, tv, fd, chassis, geoslot
    int        retval;
    struct    timeval tv;
    int        fd;
    int        chassis, geoslot;

    # 设置等待时间
    tv.tv_sec = 2;
    tv.tv_usec = 0;
    # 进入无限循环，直到条件不满足时退出
    while (1) {
        # 初始化标志位为0
        int flag = 0;
        # 创建文件描述符集合
        fd_set working_set;

        # 遍历文件描述符列表，检查是否有文件描述符仍然处于监听状态
        for (fd = 0; fd <= max_fs; fd++) {                                /* scan the list of descriptors we may be listening to */
            if (FD_ISSET(fd, &readfds)) flag = 1;                        /* and see if there are any still set */
        }
        # 如果所有文件描述符都不再监听状态，则退出循环
        if (flag == 0) return;                                            /* we are done, when they are all gone */

        # 复制监听文件描述符集合
        memcpy(&working_set, &readfds, sizeof(readfds));                /* otherwise, we still have to listen for more stuff, till we timeout */
        # 调用 select 函数，等待文件描述符状态改变
        retval = select(max_fs + 1, &working_set, NULL, NULL, &tv);
        # 如果 select 函数返回-1，表示发生错误，直接返回
        if (retval == -1) {                                                /* an error occurred !!!!! */
            return;
        } else if (retval == 0) {                                        /* timeout occurred, so process what we've got sofar and return */
            # 如果 select 函数返回0，表示超时，打印超时信息并返回
            printf("timeout\n");
            return;
        } else {
            # 遍历文件描述符列表，处理发生变化的文件描述符
            for (fd = 0; fd <= max_fs; fd++) {                            /* scan the list of things to do, and do them */
                if (FD_ISSET(fd, &working_set)) {
                    # 调用 read_client_data 函数，如果返回0表示套接字已关闭
                    if (read_client_data(fd) == 0) {                    /* if the socket has closed */
                        # 清除文件描述符的监听状态
                        FD_CLR(fd, &readfds);                            /* and descriptors we listen to for errors */
                        # 根据文件描述符查找相关信息
                        find_unit_by_fd(fd, &chassis, &geoslot, NULL);
                        # 关闭与该文件描述符相关的连接
                        close_with_IOP(chassis, geoslot, FIND);            /* and close out connection to him */
                    }
                }
            }
        }
    }
}

# 获取错误响应的函数，如果有错误则返回指针，没有错误则返回 NULL
static char *get_error_response(int fd, char *errbuf) {
    char    byte;  # 定义一个字符变量
    int        len = 0;  # 定义一个整型变量

    while (1):  # 进入无限循环
        recv(fd, &byte, 1, 0);  # 从套接字中读取一个字节
        if (errbuf && (len++ < PCAP_ERRBUF_SIZE)):  # 如果错误缓冲区存在且还有空间
            *errbuf++ = byte;  # 将字节放入缓冲区
            *errbuf = '\0';  # 确保字符串以空字符结尾，以防超出缓冲区大小
        if (byte == '\0'):  # 如果字节为零
            if (len > 1)    { return errbuf;    }  # 如果长度大于1，则返回错误缓冲区
            else            { return NULL;        }  # 否则返回 NULL
        }
    }

# 查找所有设备的函数，返回值：-1 = 错误，0 = 正常
int acn_findalldevs(char *errbuf) {
    int        chassis, geoslot;  # 定义整型变量
    unit_t    *u;  # 定义指向 unit_t 结构的指针

    FD_ZERO(&readfds);  # 清空文件描述符集合
    max_fs = 0;  # 将最大文件描述符数置为0
    for (chassis = 0; chassis <= MAX_CHASSIS; chassis++):  # 遍历机箱
        for (geoslot = 0; geoslot <= MAX_GEOSLOT; geoslot++):  # 遍历地理槽位
            u = &units[chassis][geoslot];  # 获取指向单元的指针
            if (u->ip && (open_with_IOP(u, FIND))):  # 如果 IP 存在且成功连接到远程 IOP
                send_to_fd(u->find_fd, 1, (unsigned char *)"\0");  # 发送数据到文件描述符
                if (get_error_response(u->find_fd, errbuf)):  # 获取错误响应
                    close_with_IOP(chassis, geoslot, FIND);  # 关闭与 IOP 的连接
                else:
                    if (u->find_fd > max_fs):  # 如果文件描述符大于最大文件描述符数
                        max_fs = u->find_fd;  # 更新最大文件描述符数
                    FD_SET(u->find_fd, &readfds);  # 将文件描述符添加到集合中
                    u->len = 0;  # 将长度置为0
                    send_to_fd(u->find_fd, 1, (unsigned char *)"Q");  # 发送接口查询请求
            }
        }
    }
    wait_for_all_answers();  # 等待所有答复
    if (process_client_data(errbuf)):  # 处理客户端数据，如果有错误
        return -1;  # 返回错误
    # 调用sort_if_table()函数对表进行排序
    sort_if_table();
    # 返回0，表示函数执行成功
    return 0;
# 结束静态函数 pcap_stats_acn
}

# 定义静态函数 pcap_stats_acn，接受一个 pcap_t 结构体指针和一个 pcap_stat 结构体指针作为参数
static int pcap_stats_acn(pcap_t *handle, struct pcap_stat *ps) {
    # 声明一个长度为12的无符号字符数组
    unsigned char    buf[12];

    # 向文件描述符发送一个字节的数据，发送获取统计信息的命令给 IOP
    send_to_fd(handle->fd, 1, (unsigned char *)"S");

    # 如果读取指定长度的字节失败，则返回-1
    if (read_client_nbytes(handle->fd, sizeof(buf), buf) == -1) return -1;

    # 将缓冲区的前4个字节转换为网络字节顺序的32位整数，赋值给 ps_recv
    ps->ps_recv        = ntohl(*(uint32_t *)&buf[0]);
    # 将缓冲区的第5到第8个字节转换为网络字节顺序的32位整数，赋值给 ps_drop
    ps->ps_drop        = ntohl(*(uint32_t *)&buf[4]);
    # 将缓冲区的最后4个字节转换为网络字节顺序的32位整数，赋值给 ps_ifdrop
    ps->ps_ifdrop    = ntohl(*(uint32_t *)&buf[8]);

    # 返回0表示成功
    return 0;
}

# 定义静态函数 acn_open_live，接受一个字符串指针、一个字符指针和一个整型指针作为参数
static int acn_open_live(const char *name, char *errbuf, int *linktype) {
    # 声明整型变量 chassis 和 geoslot
    int            chassis, geoslot;
    # 声明一个指向 unit_t 结构体的指针 u
    unit_t        *u;
    # 声明一个指向 iface_t 结构体的指针 p
    iface_t        *p;
    # 声明一个 pcap_if_list_t 结构体 devlist
    pcap_if_list_t    devlist;

    # 调用 pcap_platform_finddevs 函数，传入 devlist 和 errbuf 作为参数
    pcap_platform_finddevs(&devlist, errbuf);
    for (chassis = 0; chassis <= MAX_CHASSIS; chassis++) {                                        /* 逐个扫描表格... */
        for (geoslot = 0; geoslot <= MAX_GEOSLOT; geoslot++) {
            u = &units[chassis][geoslot];
            if (u->ip != NULL) {
                p = u->iface;
                while (p) {                                                                        /* 遍历所有接口... */
                    if (p->IOPname && p->name && (strcmp(p->name, name) == 0)) {                /* 如果找到了想要的接口... */
                        *linktype = p->iftype;
                        open_with_IOP(u, LIVE);                                                    /* 与该 IOP 建立连接 */
                        send_to_fd(u->fd, strlen(p->IOPname)+1, (unsigned char *)p->IOPname);    /* 发送 IOP 的接口名称和终止符 */
                        if (get_error_response(u->fd, errbuf)) {
                            return -1;
                        }
                        return u->fd;                                                            /* 返回打开的描述符 */
                    }
                    p = p->next;
                }
            }
        }
    }
    return -1;                                                                                /* 如果未找到接口，返回错误 */
static void acn_start_monitor(int fd, int snaplen, int timeout, int promiscuous, int direction) {
    unsigned char    buf[8];
    unit_t            *u;

    //printf("acn_start_monitor()\n");                // fulko
    // 通过文件描述符查找对应的单元
    find_unit_by_fd(fd, NULL, NULL, &u);
    // 如果单元的第一次标志为0
    if (u->first_time == 0) {
        // 设置命令字节流的各个字段
        buf[0]                    = 'M';
        *(uint32_t *)&buf[1]    = htonl(snaplen);
        buf[5]                    = timeout;
        buf[6]                    = promiscuous;
        buf[7]                    = direction;
        //printf("acn_start_monitor() first time\n");                // fulko
        // 发送启动监视器命令及其参数到IOP
        send_to_fd(fd, 8, buf);
        // 设置单元的第一次标志为1
        u->first_time = 1;
    }
    //printf("acn_start_monitor() complete\n");                // fulko
}

static int pcap_inject_acn(pcap_t *p, const void *buf _U_, int size _U_) {
    // 设置错误信息
    pcap_strlcpy(p->errbuf, "Sending packets isn't supported on ACN adapters",
        PCAP_ERRBUF_SIZE);
    return (-1);
}

static int pcap_setfilter_acn(pcap_t *handle, struct bpf_program *bpf) {
    int                fd = handle->fd;
    int                count;
    struct bpf_insn    *p;
    uint16_t        shortInt;
    uint32_t        longInt;

    // 发送BPF过滤器跟随命令
    send_to_fd(fd, 1, (unsigned char *)"F");
    // 获取指令序列计数
    count = bpf->bf_len;
    longInt = htonl(count);
    // 发送指令序列计数
    send_to_fd(fd, 4, (unsigned char *)&longInt);
    p = bpf->bf_insns;
    // 遍历指令序列
    while (count--) {
        shortInt = htons(p->code);
        longInt = htonl(p->k);
        // 发送指令的操作码
        send_to_fd(fd, 2, (unsigned char *)&shortInt);
        // 发送指令的jt字段
        send_to_fd(fd, 1, (unsigned char *)&p->jt);
        // 发送指令的jf字段
        send_to_fd(fd, 1, (unsigned char *)&p->jf);
        // 发送指令的偏移值
        send_to_fd(fd, 4, (unsigned char *)&longInt);
        p++;
    }
    // 如果有错误响应，则返回-1
    if (get_error_response(fd, NULL))
        return -1;
    return 0;
}
# 以指定数量的字节读取数据，并设置超时时间
static int acn_read_n_bytes_with_timeout(pcap_t *handle, int count) {
    struct        timeval tv;  // 定义时间结构体
    int            retval, fd;  // 定义返回值、文件描述符
    fd_set        r_fds;  // 定义读文件描述符集合
    fd_set        w_fds;  // 定义写文件描述符集合
    u_char        *bp;  // 定义字节指针
    int            len = 0;  // 初始化长度为0
    int            offset = 0;  // 初始化偏移量为0

    tv.tv_sec = 5;  // 设置超时时间秒
    tv.tv_usec = 0;  // 设置超时时间微秒

    fd = handle->fd;  // 获取文件描述符
    FD_ZERO(&r_fds);  // 清空读文件描述符集合
    FD_SET(fd, &r_fds);  // 将文件描述符加入读文件描述符集合
    memcpy(&w_fds, &r_fds, sizeof(r_fds));  // 复制读文件描述符集合到写文件描述符集合
    bp = handle->bp;  // 获取字节指针
    while (count) {  // 循环读取指定数量的字节
        retval = select(fd + 1, &w_fds, NULL, NULL, &tv);  // 使用select函数等待文件描述符准备好
        if (retval == -1) {  // 如果返回值为-1，表示发生错误
            return -1;  // 返回-1表示出错
        } else if (retval == 0) {  // 如果返回值为0，表示超时
            return -1;  // 返回-1表示超时
        } else {
            if ((len = recv(fd, (bp + offset), count, 0)) <= 0) {  // 接收数据到指定缓冲区
                return -1;  // 返回-1表示接收数据出错
            }
            count -= len;  // 更新剩余需要读取的字节数
            offset += len;  // 更新偏移量
        }
    }
    return 0;  // 返回0表示成功
}

static int pcap_read_acn(pcap_t *handle, int max_packets, pcap_handler callback, u_char *user) {
    #define HEADER_SIZE (4 * 4)  // 定义报文头部大小
    unsigned char        packet_header[HEADER_SIZE];  // 定义报文头部缓冲区
    struct pcap_pkthdr    pcap_header;  // 定义pcap报文头部结构体

    acn_start_monitor(handle->fd, handle->snapshot, handle->opt.timeout, handle->opt.promisc, handle->direction);  // 开始监视网络流量
    handle->bp = packet_header;  // 设置报文头部缓冲区
    # 如果读取指定字节数超时，则返回 0
    if (acn_read_n_bytes_with_timeout(handle, HEADER_SIZE) == -1) return 0;            /* try to read a packet header in so we can get the sizeof the packet data */

    # 从数据包头部获取时间戳的秒部分，并转换为主机字节顺序
    pcap_header.ts.tv_sec    = ntohl(*(uint32_t *)&packet_header[0]);                /* tv_sec */
    # 从数据包头部获取时间戳的微秒部分，并转换为主机字节顺序
    pcap_header.ts.tv_usec    = ntohl(*(uint32_t *)&packet_header[4]);                /* tv_usec */
    # 从数据包头部获取数据包的捕获长度，并转换为主机字节顺序
    pcap_header.caplen        = ntohl(*(uint32_t *)&packet_header[8]);                /* caplen */
    # 从数据包头部获取数据包的实际长度，并转换为主机字节顺序
    pcap_header.len            = ntohl(*(uint32_t *)&packet_header[12]);                /* len */

    # 设置接收指针的起始位置
    handle->bp = (u_char *)handle->buffer + handle->offset;                                    /* start off the receive pointer at the right spot */
    # 如果读取指定长度的数据超时，则返回 0
    if (acn_read_n_bytes_with_timeout(handle, pcap_header.caplen) == -1) return 0;    /* then try to read in the rest of the data */

    # 调用用户提供的回调函数，传入用户数据、数据包头部和数据包数据
    callback(user, &pcap_header, handle->bp);                                        /* call the user supplied callback function */
    # 返回 1 表示成功
    return 1;
}

static int pcap_activate_sita(pcap_t *handle) {
    int        fd;

    if (handle->opt.rfmon) {
        /*
         * 如果设置了监控模式，SITA 设备不支持监控模式（它们不是 Wi-Fi 设备）。
         */
        return PCAP_ERROR_RFMON_NOTSUP;
    }

    /* 初始化 pcap 结构的一些组件。 */

    handle->inject_op = pcap_inject_acn;
    handle->setfilter_op = pcap_setfilter_acn;
    handle->setdirection_op = NULL; /* 未实现 */
    handle->set_datalink_op = NULL;    /* 无法更改数据链路类型 */
    handle->getnonblock_op = pcap_getnonblock_fd;
    handle->setnonblock_op = pcap_setnonblock_fd;
    handle->cleanup_op = pcap_cleanup_acn;
    handle->read_op = pcap_read_acn;
    handle->stats_op = pcap_stats_acn;

    fd = acn_open_live(handle->opt.device, handle->errbuf,
        &handle->linktype);
    if (fd == -1)
        return PCAP_ERROR;

    /*
     * 将负的快照值（无效）、快照值为 0（未指定）或大于正常最大值的值，转换为最大允许的值。
     *
     * 如果某些应用程序确实 *需要* 更大的快照长度，我们应该增加 MAXIMUM_SNAPLEN。
     */
    if (handle->snapshot <= 0 || handle->snapshot > MAXIMUM_SNAPLEN)
        handle->snapshot = MAXIMUM_SNAPLEN;

    handle->fd = fd;
    handle->bufsize = handle->snapshot;

    /* 分配缓冲区 */

    handle->buffer     = malloc(handle->bufsize + handle->offset);
    if (!handle->buffer) {
        pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
            errno, "malloc");
        pcap_cleanup_acn(handle);
        return PCAP_ERROR;
    }

    /*
     * "handle->fd" 是一个套接字，因此 "select()" 和 "poll()"
     * 应该在其上工作。
     */
    handle->selectable_fd = handle->fd;

    return 0;
}

pcap_t *pcap_create_interface(const char *device _U_, char *ebuf) {
    pcap_t *p;

    p = PCAP_CREATE_COMMON(ebuf, struct pcap_sita);
    # 如果指针 p 为空，则返回空指针
    if (p == NULL)
        return (NULL);

    # 设置指针 p 的 activate_op 属性为 pcap_activate_sita
    p->activate_op = pcap_activate_sita;
    # 返回指针 p
    return (p);
// 查找网络设备列表并返回结果
int pcap_platform_finddevs(pcap_if_list_t *devlistp, char *errbuf) {

    //printf("pcap_findalldevs()\n");                // fulko

    *alldevsp = 0;                                                /* 在执行任何操作之前初始化返回的变量 */
    strcpy(errbuf, "");
    if (acn_parse_hosts_file(errbuf))                            /* 扫描主机文件以获取潜在的IOPs */
        {
        //printf("pcap_findalldevs() returning BAD after parsehosts\n");                // fulko
        return -1;
        }
    //printf("pcap_findalldevs() got hostlist now finding devs\n");                // fulko
    if (acn_findalldevs(errbuf))                                /* 然后询问IOPs他们的可监控设备 */
        {
        //printf("pcap_findalldevs() returning BAD after findalldevs\n");                // fulko
        return -1;
        }
    devlistp->beginning = acn_if_list;
    acn_if_list = 0;                                            /* 然后忘记我们的列表头，因为有人会调用pcap_freealldevs()来清空malloc的内容 */
    //printf("pcap_findalldevs() returning ZERO OK\n");                // fulko
    return 0;
}

/*
 * Libpcap版本字符串。
 */
const char *
pcap_lib_version(void)
{
    return PCAP_VERSION_STRING " (SITA-only)";
}
```