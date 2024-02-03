# `nmap\nping\common_modified.h`

```cpp
// 如果 COMMON_MODIFIED_H 未定义，则定义为 1
#ifndef COMMON_MODIFIED_H
#define COMMON_MODIFIED_H 1

// 包含 nping.h 和 common.h 文件
#include "nping.h"
#include "common.h"

/*****************************************************************************
  * STUFF FROM TargetGroup.h
  ****************************************************************************/

// 定义 TargetGroup 类
class TargetGroup {
 public:
  // 用于 get_target_types 的枚举类型
  enum _targets_types { TYPE_NONE, IPV4_NETMASK, IPV4_RANGES, IPV6_ADDRESS };
  // 用作跳过范围的输入
  enum _octet_nums { FIRST_OCTET, SECOND_OCTET, THIRD_OCTET };
  // 构造函数
  TargetGroup();

  // 用新表达式（如 192.168.0.0/16，10.1.0-5.1-254，或 fe80::202:e3ff:fe14:1102）初始化（或重新初始化）对象。af 参数为 AF_INET 或 AF_INET6。成功返回 0
  int parse_expr(const char * const target_expr, int af);
  // 重置对象而不重新初始化它
  int rewind();
  // 从表达式中获取下一个主机（如果有的话）。成功返回 0 并填充 ss。ss 必须指向预分配的 sockaddr_storage 结构
  int get_next_host(struct sockaddr_storage *ss, size_t *sslen);
  // 返回最后给定的主机，以便下次调用 get_next_host 时再次给出。显然，只有在自从调用 parse_expr() 以来至少获取了 1 个主机时才应调用此函数
  int return_last_host();
  // 返回目标类型
  char get_targets_type() {return targets_type;};
  // 获取子网掩码
  int get_mask() {return netmask;};
  // 当前表达式是否为命名主机
  int get_namedhost() {return namedhost;};
  // 跳过范围数组中的一个八位组
  int skip_range(_octet_nums octet);
 private:
  // 目标类型的枚举类型
  enum _targets_types targets_type;
  // 初始化函数
  void Initialize();

  // 如果有 IPv6，则定义 sockaddr_in6 结构
#if HAVE_IPV6
  struct sockaddr_in6 ip6;
#endif

  /* 用于指定目标网络的'/mask'样式(IPV4_NETMASK) */
  u32 netmask;  // 网络掩码
  struct in_addr startaddr;  // 起始地址
  struct in_addr currentaddr;  // 当前地址
  struct in_addr endaddr;  // 结束地址

  // 用于指定目标表达式的'138.[1-7,16,91-95,200-].12.1'样式(IPV4_RANGES)
  u8 addresses[4][256];  // 地址数组
  unsigned int current[4];  // 当前地址
  u8 last[4];  // 最后一个地址

  /* 结构中剩余的IP数量 -- 如果字段无效，则设置为0 */
  unsigned long long ipsleft;  // 剩余IP数量

  // 当前目标表达式是否为命名主机
  int namedhost;  // 是否为命名主机
};



/*****************************************************************************
  * tcpip.cc中的内容
  ****************************************************************************/
int devname2ipaddr_alt(char *dev, struct sockaddr_storage *addr);  // 将设备名称转换为IP地址
void getpts_aux(const char *origexpr, int nested, u8 *porttbl, int *portwarning);  // 获取端口

#endif
```