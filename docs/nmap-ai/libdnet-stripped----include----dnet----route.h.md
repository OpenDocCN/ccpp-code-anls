# `nmap\libdnet-stripped\include\dnet\route.h`

```cpp
/*
 * route.c
 *
 * Kernel route table operations.
 *
 * Copyright (c) 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: route.h 260 2002-02-04 04:03:45Z dugsong $
 */

#ifndef DNET_ROUTE_H
#define DNET_ROUTE_H

/*
 * Routing table entry
 */
struct route_entry {
    char        intf_name[INTF_NAME_LEN];    /* interface name */
    struct addr    route_dst;    /* destination address */
    struct addr    route_gw;    /* gateway address */
    int        metric;        /* per-route metric */
};

typedef struct route_handle route_t;

typedef int (*route_handler)(const struct route_entry *entry, void *arg);

__BEGIN_DECLS
route_t    *route_open(void);    // 打开路由表
int     route_add(route_t *r, const struct route_entry *entry);    // 添加路由表项
int     route_delete(route_t *r, const struct route_entry *entry);    // 删除路由表项
int     route_get(route_t *r, struct route_entry *entry);    // 获取路由表项
int     route_loop(route_t *r, route_handler callback, void *arg);    // 遍历路由表
route_t    *route_close(route_t *r);    // 关闭路由表
__END_DECLS

#endif /* DNET_ROUTE_H */
```