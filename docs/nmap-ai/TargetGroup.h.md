# `nmap\TargetGroup.h`

```cpp
/* $Id$ */

#ifndef TARGETGROUP_H
#define TARGETGROUP_H

#include <list>
#include <cstddef>

class NetBlock;

class TargetGroup {
public:
  NetBlock *netblock;

  // 默认构造函数，将 netblock 初始化为 NULL
  TargetGroup() {
    this->netblock = NULL;
  }

  ~TargetGroup();

  /* 使用新的表达式初始化（或重新初始化）对象，例如 192.168.0.0/16，10.1.0-5.1-254，或 fe80::202:e3ff:fe14:1102。af 参数是 AF_INET 或 AF_INET6，成功返回 0 */
  int parse_expr(const char *target_expr, int af);
  /* 从表达式中获取下一个主机（如果有的话）。成功返回 0，并填充 ss。ss 必须指向预先分配的 sockaddr_storage 结构 */
  int get_next_host(struct sockaddr_storage *ss, std::size_t *sslen);
  /* 如果给定的地址是用于创建此目标组的地址，则返回 true；即不是通过 netmask 派生的地址 */
  bool is_resolved_address(const struct sockaddr_storage *ss) const;
  /* 返回为此组解析的名称或地址的字符串 */
  const char *get_resolved_name(void) const;
  /* 返回名称解析产生的地址列表，但未被扫描的地址列表，如果它来自名称解析 */
  const std::list<struct sockaddr_storage> &get_unscanned_addrs(void) const;
  /* 当前表达式是否为命名主机 */
  int get_namedhost() const;
};

#endif /* TARGETGROUP_H */
```