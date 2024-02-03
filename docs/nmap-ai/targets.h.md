# `nmap\targets.h`

```cpp
/* $Id$ */

#ifndef TARGETS_H
#define TARGETS_H

#include "TargetGroup.h"
#include <list>
#include <nbase.h>
class Target;

class HostGroupState {
public:
  /* The maximum number of entries we want to allow storing in defer_buffer. */
  static const unsigned int DEFER_LIMIT = 64;

  // 构造函数，初始化 HostGroupState 对象
  HostGroupState(int lookahead, int randomize, int argc, const char *argv[]);
  // 析构函数，释放 HostGroupState 对象
  ~HostGroupState();
  // 存储 Target 对象指针的数组
  Target **hostbatch;

  /* The defer_buffer is a place to store targets that have previously been
     returned but that can't be used right now. They wait in defer_buffer until
     HostGroupState::undefer is called, at which point they all move to the end
     of the undeferred list. HostGroupState::next_target always pulls from the
     undeferred list before returning anything new. */
  // 存储先前返回但当前不能使用的目标的列表
  std::list<Target *> defer_buffer;
  // 存储已经可以使用的目标的列表
  std::list<Target *> undeferred;

  // 命令行参数的数量
  int argc;
  // 命令行参数的数组
  const char **argv;
  // hostbatch[] 数组的最大大小
  int max_batch_sz; 
  // hostbatch[] 数组当前有效成员的数量
  int current_batch_sz; 
  // 下一个要返回给用户的 hostbatch[] 成员的索引
  int next_batch_no; 
  // 是否对每个批次进行“洗牌”以进行 ping 扫描
  int randomize; 
  // 当前组的目标列表
  TargetGroup current_group; 

  // 如果 defer_buffer 还没有满，则返回 true
  bool defer(Target *t);
  // 将 defer_buffer 中的目标移动到 undeferred 列表的末尾
  void undefer();
  // 返回下一个表达式
  const char *next_expression();
  // 返回下一个目标
  Target *next_target();
};

// 用于传递有关主机发现使用的端口信息
Target *nexthost(HostGroupState *hs, struct addrset *exclude_group,
                 const struct scan_lists *ports, int pingtype);
// 从文件中加载要排除的地址集
int load_exclude_file(struct addrset *exclude_group, FILE *fp);
// 从字符串中加载要排除的地址集
int load_exclude_string(struct addrset *exclude_group, const char *s);
// 用于将排除列表转储到标准输出的调试程序
// 声明一个函数，该函数接受一个指向addrset结构的指针参数，并返回一个整数值
int dumpExclude(const struct addrset *exclude_group);

/* 返回由nexthost获取的最后一个主机。下次调用nexthost()时将再次给出该主机。 */
void returnhost(HostGroupState *hs);

// 声明一个函数，该函数接受一个指向指针数组的指针、一个整数值和一个指向Target结构的指针参数，并返回一个布尔值
bool target_needs_new_hostgroup(Target **targets, int targets_sz, const Target *target);

#endif /* TARGETS_H */
```