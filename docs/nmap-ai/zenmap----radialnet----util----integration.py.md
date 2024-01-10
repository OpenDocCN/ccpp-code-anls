# `nmap\zenmap\radialnet\util\integration.py`

```
# 设置文件编码为 UTF-8

# 重要提示：NMAP 许可证条款
# Nmap 安全扫描器由 Nmap Software LLC（“Nmap 项目”）（C）1996-2023 年开发。Nmap 也是 Nmap 项目的注册商标。
# 本程序根据 Nmap 公共源代码许可证（NPSL）的条款分发。适用于特定 Nmap 版本或源代码控制修订版本的确切许可证文本包含在该版本的 Nmap 或源代码控制修订版本的 LICENSE 文件中。有关更多 Nmap 版权/法律信息，请访问 https://nmap.org/book/man-legal.html，并且有关 NPSL 许可证本身的更多信息，请访问 https://nmap.org/npsl/。本标题总结了 Nmap 许可证的一些关键点，但不能替代实际的许可证文本。
# Nmap 通常允许最终用户自行下载和使用，包括商业用途。可从 https://nmap.org 获取。
# Nmap 许可证通常禁止公司在商业产品中使用和重新分发 Nmap，但我们销售具有更宽松许可证和特殊功能的特殊 Nmap OEM 版本。请参阅 https://nmap.org/oem/
# 如果您收到了书面的 Nmap 许可协议或合同，其中规定了与这些条款（例如 Nmap OEM 许可证）不同的条款，您可以选择根据那些条款使用和重新分发 Nmap。
# 官方的 Nmap Windows 构建包括 Npcap 软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特殊许可就重新分发的单独许可条款的约束。因此，官方的 Nmap Windows 构建可能不得未经特殊许可（例如 Nmap OEM 许可证）重新分发。
# 我们提供此软件的源代码，因为我们相信用户有权根据自己的需求进行修改和定制。
# 导入所需的模块
from radialnet.core.Graph import Graph
from radialnet.gui.RadialNet import NetNode

# 导入数学模块
import math

# 颜色列表
COLORS = [(0.0, 1.0, 0.0),
          (1.0, 1.0, 0.0),
          (1.0, 0.0, 0.0)]

# 基础半径
BASE_RADIUS = 5.5
# 无半径
NONE_RADIUS = 4.5

# 设置节点信息的函数
def set_node_info(node, host):
    """
    设置节点的主机信息
    """
    node.set_host(host)

    # 计算节点的半径
    radius = BASE_RADIUS + 2 * math.log(
            node.get_info("number_of_open_ports") + 1)

    # 设置节点的绘制信息
    node.set_draw_info({"color": COLORS[node.get_info("vulnerability_score")],
                        "radius": radius})


# 跟踪路由主机信息类
class TracerouteHostInfo(object):
    """This is a minimal implementation of HostInfo, sufficient to
    represent the information in an intermediate traceroute hop."""
    # 初始化函数，初始化对象的各个属性
    def __init__(self):
        # IP 地址
        self.ip = None
        # IPv6 地址
        self.ipv6 = None
        # MAC 地址
        self.mac = None
        # 主机名
        self.hostname = None
        # 端口列表
        self.ports = []
        # 额外端口列表
        self.extraports = []
        # 操作系统匹配列表
        self.osmatches = []
    
    # 获取主机名的方法
    def get_hostname(self):
        return self.hostname
    
    # 获取最佳操作系统匹配的方法
    def get_best_osmatch(self):
        # 如果没有操作系统匹配结果，则返回 None
        if not self.osmatches:
            return None
    
        # 定义操作系统匹配结果的排序规则
        def osmatch_key(osmatch):
            try:
                return -float(osmatch["accuracy"])
            except ValueError:
                return 0
    
        # 返回排序后的操作系统匹配结果列表中的第一个元素
        return sorted(self.osmatches, key=osmatch_key)[0]
    
    # 主机名的属性，使用 lambda 表达式定义
    hostnames = property(lambda self: self.hostname and [self.hostname] or [])
# 根据跳数和 TTL 值查找对应的跳数信息
def find_hop_by_ttl(hops, ttl):
    # 断言 TTL 值必须是非负数
    assert ttl >= 0, "ttl must be non-negative"
    # 如果 TTL 为 0，表示在同一台机器上（即本地主机）
    if ttl == 0:
        return {"ipaddr": "127.0.0.1/8"}
    # 遍历跳数列表，查找对应 TTL 值的跳数信息
    for h in hops:
        if ttl == int(h["ttl"]):
            return h
    # 如果未找到对应 TTL 值的跳数信息，则返回 None
    return None


# 根据主机信息创建图形
def make_graph_from_hosts(hosts):
    # 创建图形对象
    graph = Graph()
    # 创建节点列表
    nodes = list()
    # 创建节点缓存
    node_cache = {}
    # 创建祖先节点缓存
    ancestor_node_cache = {}
    # 创建后代节点缓存
    descendant_node_cache = {}

    # 设置初始参考主机
    main_node = NetNode()
    nodes.append(main_node)

    # 设置本地主机信息
    localhost = TracerouteHostInfo()
    localhost.ip = {"addr": "127.0.0.1/8", "type": "ipv4"}
    localhost.hostname = "localhost"
    main_node.set_host(localhost)
    main_node.set_draw_info(
            {"valid": True, "color": (0, 0, 0), "radius": NONE_RADIUS})

    # 保存端点以便连接扫描到的主机
    endpoints = {}
    # 对于每个主机，将其挂载到图形上
    # 对于每个完全扫描的主机
    for host in hosts:
        ip = host.ip
        if ip is None:
            ip = host.ipv6

        # 从节点缓存中获取节点信息
        node = node_cache.get(ip["addr"])
        if node is None:
            node = NetNode()
            nodes.append(node)

            node.set_draw_info({"no_route": True})

            graph.set_connection(node, endpoints[host])

        node.set_draw_info({"valid": True})
        node.set_draw_info({"scanned": True})
        set_node_info(node, host)
        node_cache[node.get_info("ip")] = node

    # 设置图形的节点列表
    graph.set_nodes(nodes)
    # 设置图形的主节点
    graph.set_main_node(main_node)

    return graph


# 根据 nmap 解析器创建图形
def make_graph_from_nmap_parser(parser):
    return make_graph_from_hosts(
            parser.get_root().search_children('host', deep=True))
```