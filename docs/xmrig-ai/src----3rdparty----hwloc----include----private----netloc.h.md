# `xmrig\src\3rdparty\hwloc\include\private\netloc.h`

```
/*
 * 版权声明
 * 版权所有 © 2014 Cisco Systems, Inc.
 * 版权所有 © 2013-2014 University of Wisconsin-La Crosse.
 * 版权所有 © 2015-2017 Inria.
 *
 * $COPYRIGHT$
 *
 * 可能存在其他版权声明，请参见顶层目录中的 COPYING 文件。
 *
 * $HEADER$
 */

#ifndef _NETLOC_PRIVATE_H_
#define _NETLOC_PRIVATE_H_

#include <hwloc.h>  // 包含 hwloc 库的头文件
#include <netloc.h>  // 包含 netloc 库的头文件
#include <netloc/uthash.h>  // 包含 netloc 库的 uthash 头文件
#include <netloc/utarray.h>  // 包含 netloc 库的 utarray 头文件
#include <private/autogen/config.h>  // 包含私有的自动生成配置文件

#define NETLOCFILE_VERSION 1  // 定义 NETLOC 文件版本号为 1

#ifdef NETLOC_SCOTCH
#include <stdint.h>  // 包含标准整数类型的头文件
#include <scotch.h>  // 包含 scotch 库的头文件
#define NETLOC_int SCOTCH_Num  // 如果定义了 NETLOC_SCOTCH，则将 NETLOC_int 定义为 SCOTCH_Num
#else
#define NETLOC_int int  // 否则将 NETLOC_int 定义为 int
#endif

/*
 * 从 hwloc 中 "导入" 一些内容
 */
#define __netloc_attribute_unused __hwloc_attribute_unused  // 定义 __netloc_attribute_unused 为 __hwloc_attribute_unused
#define __netloc_attribute_malloc __hwloc_attribute_malloc  // 定义 __netloc_attribute_malloc 为 __hwloc_attribute_malloc
#define __netloc_attribute_const __hwloc_attribute_const  // 定义 __netloc_attribute_const 为 __hwloc_attribute_const
#define __netloc_attribute_pure __hwloc_attribute_pure  // 定义 __netloc_attribute_pure 为 __hwloc_attribute_pure
#define __netloc_attribute_deprecated __hwloc_attribute_deprecated  // 定义 __netloc_attribute_deprecated 为 __hwloc_attribute_deprecated
#define __netloc_attribute_may_alias __hwloc_attribute_may_alias  // 定义 __netloc_attribute_may_alias 为 __hwloc_attribute_may_alias
#define NETLOC_DECLSPEC HWLOC_DECLSPEC  // 定义 NETLOC_DECLSPEC 为 HWLOC_DECLSPEC

/**********************************************************************
 * 类型
 **********************************************************************/

/**
 * 比较器的定义
 * \sa 这些是以下函数的返回值:
 *     netloc_network_compare, netloc_dt_edge_t_compare, netloc_dt_node_t_compare
 */
typedef enum {
    NETLOC_CMP_SAME    =  0,  /**< 比较结果相同 */
    NETLOC_CMP_SIMILAR = -1,  /**< 比较结果相似，但不相同 */
    NETLOC_CMP_DIFF    = -2   /**< 比较结果不同 */
} netloc_compare_type_t;

/**
 * 支持网络的各种类型的枚举类型
 */
typedef enum {
    NETLOC_NETWORK_TYPE_ETHERNET    = 1, /**< 以太网网络 */
    NETLOC_NETWORK_TYPE_INFINIBAND  = 2, /**< InfiniBand 网络 */
    NETLOC_NETWORK_TYPE_INVALID     = 3  /**< 无效网络 */
} netloc_network_type_t;

/**
 * Enumerated type for the various types of supported topologies
 */
typedef enum {
    NETLOC_TOPOLOGY_TYPE_INVALID = -1, /**< Invalid */
    NETLOC_TOPOLOGY_TYPE_TREE    = 1,  /**< Tree */
} netloc_topology_type_t;

/**
 * Enumerated type for the various types of nodes
 */
typedef enum {
    NETLOC_NODE_TYPE_HOST    = 0, /**< Host (a.k.a., network addressable endpoint - e.g., MAC Address) node */
    NETLOC_NODE_TYPE_SWITCH  = 1, /**< Switch node */
    NETLOC_NODE_TYPE_INVALID = 2  /**< Invalid node */
} netloc_node_type_t;

typedef enum {
    NETLOC_ARCH_TREE    =  0,  /* Fat tree */
} netloc_arch_type_t;


/* Pre declarations to avoid inter dependency problems */
/** \cond IGNORE */
struct netloc_topology_t;
typedef struct netloc_topology_t netloc_topology_t;
struct netloc_node_t;
typedef struct netloc_node_t netloc_node_t;
struct netloc_edge_t;
typedef struct netloc_edge_t netloc_edge_t;
struct netloc_physical_link_t;
typedef struct netloc_physical_link_t netloc_physical_link_t;
struct netloc_path_t;
typedef struct netloc_path_t netloc_path_t;

struct netloc_arch_tree_t;
typedef struct netloc_arch_tree_t netloc_arch_tree_t;
struct netloc_arch_node_t;
typedef struct netloc_arch_node_t netloc_arch_node_t;
struct netloc_arch_node_slot_t;
typedef struct netloc_arch_node_slot_t netloc_arch_node_slot_t;
struct netloc_arch_t;
typedef struct netloc_arch_t netloc_arch_t;
/** \endcond */

/**
 * \struct netloc_topology_t
 * \brief Netloc Topology Context
 *
 * An opaque data structure used to reference a network topology.
 *
 * \note Must be initialized with \ref netloc_topology_construct()
 */
struct netloc_topology_t {
    /** Topology path */
    char *topopath;
    /** Subnet ID */
    char *subnet_id;

    /** Node List */
    netloc_node_t *nodes; /* Hash table of nodes by physical_id */
    netloc_node_t *nodesByHostname; /* Hash table of nodes by hostname */
    # 声明一个指向 netloc_physical_link_t 结构体的指针，表示物理链接的哈希表
    netloc_physical_link_t *physical_links; 

    # 声明一个 UT_array 结构体指针，表示分区列表
    UT_array *partitions;

    # 声明一个 char 类型指针，表示 Hwloc 拓扑路径
    char *hwlocpath;
    # 声明一个 UT_array 结构体指针，表示 Hwloc 拓扑列表
    UT_array *topos;
    # 声明一个指向 hwloc_topology_t 结构体指针的指针，表示 Hwloc 拓扑
    hwloc_topology_t *hwloc_topos;

    # 声明一个 netloc_topology_type_t 类型的变量，表示图的类型
    netloc_topology_type_t type;
/**
 * \brief Netloc Node Type
 *
 * Represents the concept of a node (a.k.a., vertex, endpoint) within a network
 * graph. This could be a server or a network switch. The \ref node_type parameter
 * will distinguish the exact type of node this represents in the graph.
 */
struct netloc_node_t {
    UT_hash_handle hh;       /* makes this structure hashable with physical_id */
    UT_hash_handle hh2;      /* makes this structure hashable with hostname */

    /** Physical ID of the node */
    char physical_id[20];

    /** Logical ID of the node (if any) */
    int logical_id;

    /** Type of the node */
    netloc_node_type_t type;

    /* Pointer to physical_links */
    UT_array *physical_links;

    /** Description information from discovery (if any) */
    char *description;

    /**
     * Application-given private data pointer.
     * Initialized to NULL, and not used by the netloc library.
     */
    void * userdata;

    /** Outgoing edges from this node */
    netloc_edge_t *edges;

    UT_array *subnodes; /* the group of nodes for the virtual nodes */

    netloc_path_t *paths;

    char *hostname;

    UT_array *partitions; /* index in the list from the topology */

    hwloc_topology_t hwlocTopo;
    int hwlocTopoIdx;
};

/**
 * \brief Netloc Edge Type
 *
 * Represents the concept of a directed edge within a network graph.
 *
 * \note We do not point to the netloc_node_t structure directly to
 * simplify the representation, and allow the information to more easily
 * be entered into the data store without circular references.
 * \todo JJH Is the note above still true?
 */
struct netloc_edge_t {
    UT_hash_handle hh;       /* makes this structure hashable */

    netloc_node_t *dest;

    int id;

    /** Pointers to the parent node */
    netloc_node_t *node;

    /* Pointer to physical_links */
    UT_array *physical_links;

    /** total gbits of the links */
    float total_gbits;
    # 用于存储拓扑结构中分区的索引
    UT_array *partitions; /* index in the list from the topology */

    # 用于存储指向虚拟节点的边
    UT_array *subnode_edges; /* for edges going to virtual nodes */

    # 指向另一条边的指针
    struct netloc_edge_t *other_way;

    /**
     * 应用程序提供的私有数据指针。
     * 初始化为 NULL，并且 netloc 库不使用它。
     */
    void * userdata;
};

// 定义网络位置物理连接结构体
struct netloc_physical_link_t {
    UT_hash_handle hh;       /* makes this structure hashable */  // 使该结构体可哈希化

    int id; // TODO long long  // 连接的唯一标识符
    netloc_node_t *src;  // 连接的源节点
    netloc_node_t *dest;  // 连接的目标节点
    int ports[2];  // 连接的端口
    char *width;  // 连接的宽度
    char *speed;  // 连接的速度

    netloc_edge_t *edge;  // 连接的边

    int other_way_id;  // 另一条连接的唯一标识符
    struct netloc_physical_link_t *other_way;  // 另一条连接的指针

    UT_array *partitions; /* index in the list from the topology */  // 分区的索引

    /** gbits of the link from speed and width */
    float gbits;  // 连接的速度和宽度对应的 gbits

    /** Description information from discovery (if any) */
    char *description;  // 从发现中获取的描述信息
};

// 定义网络位置路径结构体
struct netloc_path_t {
    UT_hash_handle hh;       /* makes this structure hashable */  // 使该结构体可哈希化
    char dest_id[20];  // 目标节点的唯一标识符
    UT_array *links;  // 路径上的连接
};


/**********************************************************************
 *        Architecture structures
 **********************************************************************/

// 定义网络位置架构树结构体
struct netloc_arch_tree_t {
    NETLOC_int num_levels;  // 层级数
    NETLOC_int *degrees;  // 每层的度数
    NETLOC_int *cost;  // 成本
};

// 定义网络位置架构节点结构体
struct netloc_arch_node_t {
    UT_hash_handle hh;       /* makes this structure hashable */  // 使该结构体可哈希化
    char *name; /* Hash key */  // 哈希键
    netloc_node_t *node; /* Corresponding node */  // 对应的节点
    int idx_in_topo; /* idx with ghost hosts to have complete topo */  // 在拓扑中的索引
    int num_slots; /* it is not the real number of slots but the maximum slot idx */  // 插槽的数量
    int *slot_idx; /* corresponding idx in slot_tree */  // 在插槽树中的对应索引
    int *slot_os_idx; /* corresponding os index for each leaf in tree */  // 对应树中每个叶子的操作系统索引
    netloc_arch_tree_t *slot_tree; /* Tree built from hwloc */  // 从 hwloc 构建的树
    int num_current_slots; /* Number of PUs */  // 当前插槽数量
    NETLOC_int *current_slots; /* indices in the complete tree */  // 在完整树中的索引
    int *slot_ranks; /* corresponding MPI rank for each leaf in tree */  // 对应树中每个叶子的 MPI 等级
};

// 定义网络位置架构节点插槽结构体
struct netloc_arch_node_slot_t {
    netloc_arch_node_t *node;  // 架构节点
    int slot;  // 插槽
};

// 定义网络位置架构结构体
struct netloc_arch_t {
    netloc_topology_t *topology;  // 拓扑
    int has_slots; /* if slots are included in the architecture */  // 是否在架构中包含插槽
    netloc_arch_type_t type;  // 架构类型
    # 定义一个联合体，包含两个指向netloc_arch_tree_t类型的指针，用于存储节点树或全局树
    union {
        netloc_arch_tree_t *node_tree;
        netloc_arch_tree_t *global_tree;
    } arch;
    # 指向节点的名称的指针
    netloc_arch_node_t *nodes_by_name;
    # 指向节点槽的指针，按照完整拓扑结构的索引
    netloc_arch_node_slot_t *node_slot_by_idx; 
    # 当前主机的数量，如果有槽位，则主机是一个槽位，否则主机是一个节点
    NETLOC_int num_current_hosts; 
    # 当前主机的索引，按照完整拓扑结构的索引
    NETLOC_int *current_hosts; 
};

/**********************************************************************
 * Topology Functions
 **********************************************************************/

/**
 * Allocate a topology handle.
 *
 * User is responsible for calling \ref netloc_detach on the topology handle.
 * The network parameter information is deep copied into the topology handle, so the
 * user may destruct the network handle after calling this function and/or reuse
 * the network handle.
 *
 * \returns NETLOC_SUCCESS on success
 * \returns NETLOC_ERROR upon an error.
 */
netloc_topology_t *netloc_topology_construct(char *path);

/**
 * Destruct a topology handle
 *
 * \param topology A valid pointer to a \ref netloc_topology_t handle created
 * from a prior call to \ref netloc_topology_construct.
 *
 * \returns NETLOC_SUCCESS on success
 * \returns NETLOC_ERROR upon an error.
 */
int netloc_topology_destruct(netloc_topology_t *topology);

int netloc_topology_find_partition_idx(netloc_topology_t *topology, char *partition_name);

int netloc_topology_read_hwloc(netloc_topology_t *topology, int num_nodes,
        netloc_node_t **node_list);

#define netloc_topology_iter_partitions(topology,partition) \
    for ((partition) = (char **)utarray_front(topology->partitions); \
            (partition) != NULL; \
            (partition) = (char **)utarray_next(topology->partitions, partition))

#define netloc_topology_iter_hwloctopos(topology,hwloctopo) \
    for ((hwloctopo) = (char **)utarray_front(topology->topos); \
            (hwloctopo) != NULL; \
            (hwloctopo) = (char **)utarray_next(topology->topos, hwloctopo))

#define netloc_topology_find_node(topology,node_id,node) \
    HASH_FIND_STR(topology->nodes, node_id, node)

#define netloc_topology_iter_nodes(topology,node,_tmp) \
    HASH_ITER(hh, topology->nodes, node, _tmp)

#define netloc_topology_num_nodes(topology) \
    HASH_COUNT(topology->nodes)

/*************************************************/
/**
 * Constructor for netloc_node_t
 *
 * User is responsible for calling the destructor on the handle.
 *
 * Returns
 *   A newly allocated pointer to the network information.
 */
netloc_node_t *netloc_node_construct(void);

/**
 * Destructor for netloc_node_t
 *
 * \param node A valid node handle
 *
 * Returns
 *   NETLOC_SUCCESS on success
 *   NETLOC_ERROR on error
 */
int netloc_node_destruct(netloc_node_t *node);

char *netloc_node_pretty_print(netloc_node_t* node);

#define netloc_node_get_num_subnodes(node) \
    utarray_len((node)->subnodes)

#define netloc_node_get_subnode(node,i) \
    (*(netloc_node_t **)utarray_eltptr((node)->subnodes, (i)))

#define netloc_node_get_num_edges(node) \
    utarray_len((node)->edges)

#define netloc_node_get_edge(node,i) \
    (*(netloc_edge_t **)utarray_eltptr((node)->edges, (i)))

#define netloc_node_iter_edges(node,edge,_tmp) \
    HASH_ITER(hh, node->edges, edge, _tmp)

#define netloc_node_iter_paths(node,path,_tmp) \
    HASH_ITER(hh, node->paths, path, _tmp)

#define netloc_node_is_host(node) \
    (node->type == NETLOC_NODE_TYPE_HOST)

#define netloc_node_is_switch(node) \
    (node->type == NETLOC_NODE_TYPE_SWITCH)

#define netloc_node_iter_paths(node, path,_tmp) \
    HASH_ITER(hh, node->paths, path, _tmp)

int netloc_node_is_in_partition(netloc_node_t *node, int partition);

/*************************************************/


/**
 * Constructor for netloc_edge_t
 *
 * User is responsible for calling the destructor on the handle.
 *
 * Returns
 *   A newly allocated pointer to the edge information.
 */
netloc_edge_t *netloc_edge_construct(void);

/**
 * Destructor for netloc_edge_t
 *
 * \param edge A valid edge handle
 *
 * Returns
 *   NETLOC_SUCCESS on success
 *   NETLOC_ERROR on error
 */
int netloc_edge_destruct(netloc_edge_t *edge);

char * netloc_edge_pretty_print(netloc_edge_t* edge);

void netloc_edge_reset_uid(void);

int netloc_edge_is_in_partition(netloc_edge_t *edge, int partition);
#define netloc_edge_get_num_links(edge) \
    utarray_len((edge)->physical_links)  // 获取边的物理链接数量

#define netloc_edge_get_link(edge,i) \
    (*(netloc_physical_link_t **)utarray_eltptr((edge)->physical_links, (i)))  // 获取边上第 i 个物理链接的指针

#define netloc_edge_get_num_subedges(edge) \
    utarray_len((edge)->subnode_edges)  // 获取边的子边数量

#define netloc_edge_get_subedge(edge,i) \
    (*(netloc_edge_t **)utarray_eltptr((edge)->subnode_edges, (i)))  // 获取边上第 i 个子边的指针

/*************************************************/

/**
 * Constructor for netloc_physical_link_t
 *
 * User is responsible for calling the destructor on the handle.
 *
 * Returns
 *   A newly allocated pointer to the physical link information.
 */
netloc_physical_link_t * netloc_physical_link_construct(void);  // 构造函数，返回一个新分配的 netloc_physical_link_t 指针

/**
 * Destructor for netloc_physical_link_t
 *
 * Returns
 *   NETLOC_SUCCESS on success
 *   NETLOC_ERROR on error
 */
int netloc_physical_link_destruct(netloc_physical_link_t *link);  // 析构函数，返回成功或错误状态

char * netloc_link_pretty_print(netloc_physical_link_t* link);  // 打印 netloc_physical_link_t 的信息

/*************************************************/

netloc_path_t *netloc_path_construct(void);  // 构造函数，返回一个新分配的 netloc_path_t 指针
int netloc_path_destruct(netloc_path_t *path);  // 析构函数，返回成功或错误状态

/**********************************************************************
 *        Architecture functions
 **********************************************************************/

netloc_arch_t * netloc_arch_construct(void);  // 构造函数，返回一个新分配的 netloc_arch_t 指针

int netloc_arch_destruct(netloc_arch_t *arch);  // 析构函数，返回成功或错误状态

int netloc_arch_build(netloc_arch_t *arch, int add_slots);  // 构建架构，返回成功或错误状态

int netloc_arch_set_current_resources(netloc_arch_t *arch);  // 设置当前资源，返回成功或错误状态

int netloc_arch_set_global_resources(netloc_arch_t *arch);  // 设置全局资源，返回成功或错误状态

int netloc_arch_node_get_hwloc_info(netloc_arch_node_t *arch);  // 获取 hwloc 信息，返回成功或错误状态

void netloc_arch_tree_complete(netloc_arch_tree_t *tree, UT_array **down_degrees_by_level,
        int num_hosts, int **parch_idx);  // 完成架构树

NETLOC_int netloc_arch_tree_num_leaves(netloc_arch_tree_t *tree);  // 获取架构树的叶子节点数量
/**********************************************************************
 *        Access functions of various elements of the topology
 **********************************************************************/

// 获取对象的分区数量
#define netloc_get_num_partitions(object) \
    utarray_len((object)->partitions)

// 获取对象的第 i 个分区
#define netloc_get_partition(object,i) \
    (*(int *)utarray_eltptr((object)->partitions, (i)))


// 迭代路径中的链接
#define netloc_path_iter_links(path,link) \
    for ((link) = (netloc_physical_link_t **)utarray_front(path->links); \
            (link) != NULL; \
            (link) = (netloc_physical_link_t **)utarray_next(path->links, link))

/**********************************************************************
 *        Misc functions
 **********************************************************************/

/**
 * 解码网络类型
 *
 * \param net_type \ref netloc_network_type_t 类型的有效成员
 *
 * \returns 如果类型无效，则返回 NULL
 * \returns 该 \ref netloc_network_type_t 类型的字符串
 */
static inline const char * netloc_network_type_decode(netloc_network_type_t net_type) {
    if( NETLOC_NETWORK_TYPE_ETHERNET == net_type ) {
        return "ETH";
    }
    else if( NETLOC_NETWORK_TYPE_INFINIBAND == net_type ) {
        return "IB";
    }
    else {
        return NULL;
    }
}

/**
 * 解码节点类型
 *
 * \param node_type \ref netloc_node_type_t 类型的有效成员
 *
 * \returns 如果类型无效，则返回 NULL
 * \returns 该 \ref netloc_node_type_t 类型的字符串
 */
static inline const char * netloc_node_type_decode(netloc_node_type_t node_type) {
    if( NETLOC_NODE_TYPE_SWITCH == node_type ) {
        return "SW";
    }
    else if( NETLOC_NODE_TYPE_HOST == node_type ) {
        return "CA";
    }
    else {
        return NULL;
    }
}

// 获取一行内容
ssize_t netloc_line_get(char **lineptr, size_t *n, FILE *stream);

// 获取下一个标记
char *netloc_line_get_next_token(char **string, char c);

// 构建通信矩阵
int netloc_build_comm_mat(char *filename, int *pn, double ***pmat);
# 如果输入的字符串为空，则返回空指针；否则返回输入字符串的副本
#define STRDUP_IF_NOT_NULL(str) (NULL == str ? NULL : strdup(str))
# 如果输入的字符串为空，则返回空字符串；否则返回输入字符串
#define STR_EMPTY_IF_NULL(str) (NULL == str ? "" : str)

# 结束 _NETLOC_PRIVATE_H_ 文件的定义
#endif // _NETLOC_PRIVATE_H_
```