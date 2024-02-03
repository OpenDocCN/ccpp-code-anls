# `nmap\nping\global_structures.h`

```cpp
#ifndef GLOBAL_STRUCTURES_H
#define GLOBAL_STRUCTURES_H

#include "nsock.h"
#include "nbase.h"

// 定义服务查找结构体，包括服务名称、协议类型和端口号
typedef struct service_lookup {
    char *name;
    u8 proto;
    u16 portno;
} service_lookup;

// 定义通用数据结构体，包括网络套接字池、网络套接字信息、状态、协议类型、端口号、IP地址、连接尝试次数、最大连接尝试次数、用户名、密码、缓冲区和缓冲区大小
typedef struct m_data {
    nsock_pool nsp;
    nsock_iod nsi;
    int state;
    int protocol;
    unsigned short port;
    struct in_addr ip;
    int attempts;
    int max_attempts;    /* how many attempts in one connection */
    char *username;
    char *password;
    char *buf;
    int bufsize;
} m_data;

#endif
```