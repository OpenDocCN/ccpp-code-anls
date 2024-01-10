# `nmap\portreasons.h`

```
/*
 * Written by Eddie Bell <ejlbell@gmail.com> 2007
 * Modified by Colin Rice <dah4k0r@gmail.com> 2011
 */

#ifndef REASON_H
#define REASON_H

#include "nbase.h"

#include <sys/types.h>
#include <map>
class Target;
class PortList;

typedef unsigned short reason_t;

/* Holds various string outputs of a reason  *
 * Stored inside a map which maps enum_codes *
 * to reason_strings                     */
class reason_string {
public:
    //Required for map
    reason_string();
    reason_string(const char * singular, const char * plural);
    const char * singular;
    const char * plural;
};

/* stored inside a Port Object and describes
 * why a port is in a specific state */
typedef struct port_reason {
        reason_t reason_id;
        union {
                struct sockaddr_in in;
                struct sockaddr_in6 in6;
                struct sockaddr sockaddr;
        } ip_addr;
        unsigned short ttl;

        int set_ip_addr(const struct sockaddr_storage *ss);
} state_reason_t;

/* used to calculate state reason summaries.
 * I.E 10 ports filter because of 10 no-responses */
typedef struct port_reason_summary {
        reason_t reason_id;
        unsigned int count;
        struct port_reason_summary *next;
        unsigned short proto;
        unsigned short ports[0xffff+1];
} state_reason_summary_t;
# 定义枚举类型 reason_codes，包含各种错误代码
enum reason_codes {
        ER_RESETPEER, ER_CONREFUSED, ER_CONACCEPT,
        ER_SYNACK, ER_SYN, ER_UDPRESPONSE, ER_PROTORESPONSE, ER_ACCES,

        ER_NETUNREACH, ER_HOSTUNREACH, ER_PROTOUNREACH,
        ER_PORTUNREACH, ER_ECHOREPLY,

        ER_DESTUNREACH, ER_SOURCEQUENCH, ER_NETPROHIBITED,
        ER_HOSTPROHIBITED, ER_ADMINPROHIBITED,
        ER_TIMEEXCEEDED, ER_TIMESTAMPREPLY,

        ER_ADDRESSMASKREPLY, ER_NOIPIDCHANGE, ER_IPIDCHANGE,
        ER_ARPRESPONSE, ER_NDRESPONSE, ER_TCPRESPONSE, ER_NORESPONSE,
        ER_INITACK, ER_ABORT,
        ER_LOCALHOST, ER_SCRIPT, ER_UNKNOWN, ER_USER,
        ER_NOROUTE, ER_BEYONDSCOPE, ER_REJECTROUTE, ER_PARAMPROBLEM,
};

# 定义类 reason_map_type，包含一个映射 reason_codes 到 reason_string 的私有成员 reason_map
class reason_map_type{
private:
    std::map<reason_codes,reason_string > reason_map;
public:
    reason_map_type();
    # 定义 find 方法，用于查找 reason_codes 对应的 reason_string
    std::map<reason_codes,reason_string>::const_iterator find(const reason_codes& x) const {
        # 在 reason_map 中查找 reason_codes 对应的 reason_string
        std::map<reason_codes,reason_string>::const_iterator itr = reason_map.find(x);
        # 如果找不到，则返回 ER_UNKNOWN 对应的 reason_string
        if(itr == reason_map.end())
            return reason_map.find(ER_UNKNOWN);
        return itr;
    };
};

# 定义函数 icmp_to_reason，用于将 ICMP 类型和代码转换为 reason_code
reason_codes icmp_to_reason(u8 proto, int icmp_type, int icmp_code);

# 定义宏 SINGULAR 和 PLURAL，用于确定字符串的单数或复数形式
#define SINGULAR 1
#define PLURAL 2

# 定义函数 state_reason_init，用于初始化 state_reason_t 结构
void state_reason_init(state_reason_t *reason);

# 定义函数 reason_str，用于将 reason_id 转换为字符串，根据给定的数量确定使用单数还是复数形式
const char *reason_str(reason_t reason_id, unsigned int number);

# 定义函数 get_state_reason_summary，用于返回给定状态下端口的失败原因的链表
state_reason_summary_t *get_state_reason_summary(const PortList *Ports, int state);
# 释放 get_state_reason_summary 返回的链表
# 声明一个函数，用于初始化状态原因摘要
void state_reason_summary_dinit(state_reason_summary_t *r);

/* 根据原因和源 IP 地址构建输出字符串。
 * 使用静态返回值，以便之前的值将被后续调用覆盖 */
const char *port_reason_str(state_reason_t r);
/* 根据目标对象构建输出字符串 */
const char *target_reason_str(const Target *t);

#endif
```