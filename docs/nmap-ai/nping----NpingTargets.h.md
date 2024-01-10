# `nmap\nping\NpingTargets.h`

```
#ifndef NPINGTARGETS_H
#define NPINGTARGETS_H


/* TODO: Needs to be changed if we move TargetGroup to another source file */
#include "common_modified.h"
#include "NpingTarget.h"
#include <vector>

#define MAX_NPING_HOSTNAME_LEN 512    /**< Max length for named hosts */

class NpingTargets {

  private:

    char *specs[1024];  // 存储目标规范的字符指针数组
    bool skipspec[1024];  // 存储是否跳过目标规范的布尔数组
    int speccount;  // 目标规范的数量
    int current_spec;  // 当前目标规范的索引
    bool lastwaslastingroup;  // 上一个目标是否是最后一个目标组中的最后一个
    bool finished;  // 是否已完成目标处理
    TargetGroup current_group;  // 当前目标组

    bool ready;  // 是否准备好
    unsigned long int targets_fetched;  // 已获取的目标数量
    unsigned long int current_target;  // 当前目标的索引

  public:

    NpingTargets();  // 构造函数
    ~NpingTargets();  // 析构函数
    int addSpec(char *spec);  // 添加目标规范
    int getNextTargetSockAddr(struct sockaddr_storage *t, size_t *tlen);  // 获取下一个目标的套接字地址
    NpingTarget *getNextTarget();  // 获取下一个目标
    int rewind();  // 重置目标处理状态
    int getNextTargetAddressAndName(struct sockaddr_storage *t, size_t *tlen, char *hname, size_t hlen);  // 获取下一个目标的地址和名称
    int getNextIPv4Address(u32 *addr);  // 获取下一个 IPv4 地址
    int rewindSpecs();  // 重置目标规范
    unsigned long int getTargetsFetched();  // 获取已获取的目标数量
    int getTargetSpecCount();  // 获取目标规范数量
    int processSpecs();  // 处理目标规范
    unsigned long int freeTargets();  // 释放目标
    NpingTarget *findTarget(struct sockaddr_storage *tt);  // 查找目标

    /* TODO: Make private */
    NpingTarget *currenths;  // 当前目标
    std::vector<NpingTarget *> Targets;  // 目标列表

}; /* End of class NpingTargets */

#endif
```