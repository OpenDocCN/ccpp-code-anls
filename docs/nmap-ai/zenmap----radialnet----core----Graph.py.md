# `nmap\zenmap\radialnet\core\Graph.py`

```cpp
# 设置文件编码为 UTF-8

# ***********************IMPORTANT NMAP LICENSE TERMS************************
# *
# * Nmap安全扫描器由Nmap Software LLC（“Nmap项目”）（C）1996-2023年开发。Nmap也是Nmap项目的注册商标。
# *
# * 本程序根据Nmap公共源代码许可证（NPSL）的条款分发。适用于特定Nmap版本或源代码控制修订版的确切许可证文本包含在该版本的Nmap或源代码控制修订版的LICENSE文件中。有关更多Nmap版权/法律信息，请访问https://nmap.org/book/man-legal.html，有关NPSL许可证本身的更多信息，请访问https://nmap.org/npsl/。此标题总结了Nmap许可证的一些关键点，但不能替代实际许可证文本。
# *
# * 一般情况下，用户可以免费下载和使用Nmap，包括商业用途。可从https://nmap.org获取。
# *
# * Nmap许可证一般禁止公司在商业产品中使用和重新分发Nmap，但我们销售具有更宽松许可证和特殊功能的特殊Nmap OEM版本。请参阅https://nmap.org/oem/
# *
# * 如果您收到了书面的Nmap许可协议或合同，其中规定了与这些条款（例如Nmap OEM许可证）不同的条款，您可以选择根据那些条款使用和重新分发Nmap。
# *
# * 官方的Nmap Windows版本包括Npcap软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特殊许可就重新分发的单独许可条款的约束。因此，官方的Nmap Windows版本可能不得未经特殊许可（例如Nmap OEM许可证）重新分发。
# *
# * 我们提供此软件的源代码，因为我们相信用户有权根据自己的需求进行修改和定制。
class Node(object):
    """
    Node class
    """
    # Node 类的构造方法
    def __init__(self):
        """
        Constructor method of Node class
        @type  : integer
        @param : Node identifier
        """
        # 用户控制的数据指针
        self.__data = None
        # 到其他节点的边的列表
        self.__edges = []

    # 获取数据
    def get_data(self):
        return self.__data

    # 设置数据
    def set_data(self, data):
        self.__data = data

    # 获取连接到目标节点的边，如果没有则返回 None
    def get_edge(self, dest):
        """
        Return the edge connecting to dest, or None if none
        """
        for edge in self.__edges:
            if dest in edge.get_nodes():
                return edge
        return None
    # 返回边的列表
    def get_edges(self):
        """
        Return the list of edges
        """
        return self.__edges

    # 添加一条边到边的列表中
    def add_edge(self, edge):
        self.__edges.append(edge)
class Edge:
    """
    Edge class for representing edges in a graph
    """
    def __init__(self, nodes):
        """
        Constructor method for Edge class
        @param nodes: List of nodes connected by the edge
        """
        self.__weights = []  # Initialize an empty list to store edge weights
        self.__nodes = nodes  # Store the nodes connected by the edge
        self.__weights_mean = None  # Initialize the mean of edge weights to None

    def get_nodes(self):
        """
        Get the nodes connected by the edge
        @return: List of nodes connected by the edge
        """
        return self.__nodes

    def get_weights(self):
        """
        Get the weights of the edge
        @return: List of edge weights
        """
        return self.__weights

    def set_weights(self, weights):
        """
        Set the weights of the edge
        @param weights: List of edge weights
        """
        self.__weights = weights  # Set the edge weights
        self.__weights_mean = sum(self.__weights) / len(self.__weights)  # Calculate the mean of edge weights

    def add_weight(self, weight):
        """
        Add a weight to the edge
        @param weight: The weight to be added
        """
        self.__weights.append(weight)  # Add the weight to the list of edge weights
        self.__weights_mean = sum(self.__weights) / len(self.__weights)  # Recalculate the mean of edge weights

    def get_weights_mean(self):
        """
        Get the mean of edge weights
        @return: The mean of edge weights
        """
        return self.__weights_mean


class Graph:
    """
    Network Graph class for representing a graph
    """

    def __init__(self):
        """
        Constructor method of Graph class
        """
        self.__main_node = None  # Initialize the main node to None
        self.__nodes = []  # Initialize an empty list to store nodes
        self.__max_edge_mean_value = None  # Initialize the maximum edge mean value to None
        self.__min_edge_mean_value = None  # Initialize the minimum edge mean value to None

    def set_nodes(self, nodes):
        """
        Set the nodes of the graph
        @param nodes: List of nodes to be set
        """
        self.__nodes = nodes  # Set the nodes of the graph

    def get_nodes(self):
        """
        Get the nodes of the graph
        @return: List of nodes in the graph
        """
        return self.__nodes

    def get_number_of_nodes(self):
        """
        Get the number of nodes in the graph
        @return: The number of nodes in the graph
        """
        return len(self.__nodes)

    def set_main_node(self, node):
        """
        Set the main node of the graph
        @param node: The main node to be set
        """
        self.__main_node = node  # Set the main node of the graph

    def get_main_node(self):
        """
        Get the main node of the graph
        @return: The main node of the graph
        """
        return self.__main_node
    # 设置节点之间的连接关系
    def set_connection(self, a, b, weight=None):
        """
        Set node connections
        @type  : list
        @param : List of connections
        """

        # 如果是新的连接，则创建
        edge = a.get_edge(b)
        if edge is None:
            edge = Edge((a, b))
            a.add_edge(edge)
            b.add_edge(edge)

        # 添加新的权重值
        if weight is not None:

            edge.add_weight(weight)

            mean_weight = edge.get_weights_mean()
            if (self.__min_edge_mean_value is None or
                    mean_weight < self.__min_edge_mean_value):
                self.__min_edge_mean_value = mean_weight
            if (self.__max_edge_mean_value is None or
                    mean_weight > self.__max_edge_mean_value):
                self.__max_edge_mean_value = mean_weight

    # 返回所有边的迭代器
    def get_edges(self):
        """
        An iterator that yields all edges
        """
        for node in self.__nodes:
            for edge in node.get_edges():
                if edge.get_nodes()[0] == node:
                    yield edge

    # 获取节点的连接
    def get_node_connections(self, node):
        """
        """
        connections = []

        for edge in node.get_edges():

            (a, b) = edge.get_nodes()

            if a == node:
                connections.append(b)
            if b == node:
                connections.append(a)

        return connections

    # 获取最大边的平均权重
    def get_max_edge_mean_weight(self):
        """
        """
        return self.__max_edge_mean_value

    # 获取最小边的平均权重
    def get_min_edge_mean_weight(self):
        """
        """
        return self.__min_edge_mean_value
```