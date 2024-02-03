# `nmap\zenmap\radialnet\gui\HostsViewer.py`

```cpp
# 设置文件编码为 UTF-8

# 重要提示：NMAP 许可证条款
# Nmap 安全扫描器由 Nmap 软件有限责任公司（“Nmap 项目”）（C）1996-2023 年开发。Nmap 也是 Nmap 项目的注册商标。
# 本程序根据 Nmap 公共源代码许可证（NPSL）的条款分发。适用于特定 Nmap 版本或源代码控制修订版的确切许可证文本包含在该版本的 Nmap 或源代码控制修订版的 LICENSE 文件中。有关更多 Nmap 版权/法律信息，请访问 https://nmap.org/book/man-legal.html，有关 NPSL 许可证本身的更多信息，请访问 https://nmap.org/npsl/。本标题总结了 Nmap 许可证的一些关键点，但不能替代实际的许可证文本。
# Nmap 通常允许最终用户自行下载和使用，包括商业用途。可从 https://nmap.org 获取。
# Nmap 许可证通常禁止公司在商业产品中使用和重新分发 Nmap，但我们销售具有更宽松许可证和特殊功能的特殊 Nmap OEM 版本。请参阅 https://nmap.org/oem/
# 如果您收到了书面的 Nmap 许可协议或合同，其中规定了与这些条款（例如 Nmap OEM 许可证）不同的条款，您可以选择根据那些条款使用和重新分发 Nmap。
# 官方的 Nmap Windows 构建包括 Npcap 软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特殊许可证重新分发的单独许可证条款的约束。因此，官方的 Nmap Windows 构建可能不得未经特殊许可证（例如 Nmap OEM 许可证）重新分发。
# 我们提供此软件的源代码，因为我们相信用户有权根据 NPSL 条款自行修改和重新分发它。但是，我们希望用户尊重我们的劳动成果，遵守 NPSL 条款，并购买 Nmap OEM 版本以支持我们的工作。
# 导入gi模块，确保使用的是Gtk 3.0版本
import gi

gi.require_version("Gtk", "3.0")
from gi.repository import Gtk

# 导入re模块，用于正则表达式操作
import re

# 从radialnet.bestwidgets.windows模块中导入BWMainWindow类
from radialnet.bestwidgets.windows import BWMainWindow

# 从radialnet.gui.NodeNotebook模块中导入NodeNotebook类
from radialnet.gui.NodeNotebook import NodeNotebook

# 从radialnet.util.misc模块中导入ipv4_compare函数
from radialnet.util.misc import ipv4_compare

# 定义主机颜色列表
HOSTS_COLORS = ['#d5ffd5', '#ffffd5', '#ffd5d5']

# 定义主机表头
HOSTS_HEADER = ['ID', '#', 'Hosts']

# 定义窗口尺寸
DIMENSION = (700, 400)

# 定义IP地址的正则表达式
IP_RE = re.compile(r'^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$')

# 定义HostsViewer类，继承自BWMainWindow类
class HostsViewer(BWMainWindow):
    """
    """
    # 初始化方法，接受节点列表作为参数
    def __init__(self, nodes):
        """
        """
        # 调用父类的初始化方法
        BWMainWindow.__init__(self)
        # 设置窗口标题
        self.set_title(_('Hosts Viewer'))
        # 设置窗口默认大小
        self.set_default_size(DIMENSION[0], DIMENSION[1])

        # 保存传入的节点列表
        self.__nodes = nodes
        # 创建一个默认视图标签
        self.__default_view = Gtk.Label.new(_("No node selected"))
        # 将默认视图标签设置为当前视图
        self.__view = self.__default_view

        # 调用创建小部件的方法
        self.__create_widgets()

    # 创建小部件的方法
    def __create_widgets(self):
        """
        """
        # 创建一个水平分隔窗格
        self.__panel = Gtk.Paned.new(Gtk.Orientation.HORIZONTAL)
        self.__panel.set_border_width(6)

        # 创建一个节点列表
        self.__list = HostsList(self, self.__nodes)

        # 将节点列表添加到分隔窗格的第一个位置
        self.__panel.add1(self.__list)
        # 将当前视图添加到分隔窗格的第二个位置
        self.__panel.add2(self.__view)
        # 设置分隔窗格的位置
        self.__panel.set_position(int(DIMENSION[0] / 5))

        # 将分隔窗格添加到窗口中
        self.add(self.__panel)

    # 切换笔记本的方法，接受节点作为参数
    def change_notebook(self, node):
        """
        """
        # 如果当前视图不为空，则销毁当前视图
        if self.__view is not None:
            self.__view.destroy()

        # 如果节点不为空，则创建一个新的节点笔记本作为当前视图，否则使用默认视图
        if node is not None:
            self.__view = NodeNotebook(node)
        else:
            self.__view = self.__default_view
        # 显示当前视图
        self.__view.show_all()

        # 将当前视图添加到分隔窗格的第二个位置
        self.__panel.add2(self.__view)
# 创建一个名为 HostsList 的类，继承自 Gtk.ScrolledWindow
class HostsList(Gtk.ScrolledWindow):
    """
    """

    # 初始化方法，接受父级窗口和节点列表作为参数
    def __init__(self, parent, nodes):
        """
        """
        # 调用父类的初始化方法
        super(HostsList, self).__init__()
        # 设置滚动窗口的滚动策略为自动
        self.set_policy(Gtk.PolicyType.AUTOMATIC, Gtk.PolicyType.AUTOMATIC)
        # 设置滚动窗口的阴影类型为无
        self.set_shadow_type(Gtk.ShadowType.NONE)

        # 将父级窗口和节点列表保存为对象的属性
        self.__parent = parent
        self.__nodes = nodes

        # 调用私有方法创建窗口部件
        self.__create_widgets()
    # 创建小部件
    def __create_widgets(self):
        # 创建一个文本单元格渲染器
        self.__cell = Gtk.CellRendererText()

        # 创建一个列表存储对象，包含整型、整型、字符串、字符串和布尔类型数据
        self.__hosts_store = Gtk.ListStore.new([int, int, str, str, bool])

        # 创建一个带有模型的树视图对象
        self.__hosts_treeview = Gtk.TreeView.new_with_model(self.__hosts_store)
        # 连接光标变化信号到回调函数
        self.__hosts_treeview.connect('cursor-changed', self.__cursor_callback)

        # 遍历节点列表
        for i in range(len(self.__nodes)):
            # 获取当前节点
            node = self.__nodes[i]
            # 获取节点的开放端口数量
            ports = node.get_info('number_of_open_ports')
            # 获取节点的漏洞分数对应的颜色
            color = HOSTS_COLORS[node.get_info('vulnerability_score')]
            # 获取节点的主机名或 IP 地址，如果为空则赋值为空字符串
            host = node.get_info('hostname') or node.get_info('ip') or ""
            # 将数据添加到列表存储对象中
            self.__hosts_store.append([i, ports, host, color, True])

        # 创建一个空的列列表
        self.__hosts_column = list()

        # 遍历主机头部列表
        for i in range(0, len(HOSTS_HEADER)):
            # 创建一个树视图列对象
            column = Gtk.TreeViewColumn(title=HOSTS_HEADER[i], cell_renderer=self.__cell, text=i)
            # 将列对象添加到列列表中
            self.__hosts_column.append(column)
            # 设置列可重新排序
            self.__hosts_column[i].set_reorderable(True)
            # 设置列可调整大小
            self.__hosts_column[i].set_resizable(True)
            # 设置列的属性
            self.__hosts_column[i].set_attributes(self.__cell, text=i, background=3, editable=4)

        # 将第三列添加到树视图中
        self.__hosts_treeview.append_column(self.__hosts_column[2])

        # 设置列表存储对象的排序函数
        self.__hosts_store.set_sort_func(2, self.__host_sort)

        # 设置第三列为排序列
        self.__hosts_column[2].set_sort_column_id(2)

        # 将树视图添加到窗口中
        self.add_with_viewport(self.__hosts_treeview)

        # 如果树视图模型中有数据，则设置光标到第一行
        if len(self.__hosts_treeview.get_model()) > 0:
            self.__hosts_treeview.set_cursor((0,))
        # 调用光标回调函数
        self.__cursor_callback(self.__hosts_treeview)
    # 定义私有方法，用于处理光标回调事件
    def __cursor_callback(self, widget):
        # 获取光标所在的路径
        path = widget.get_cursor()[0]
        # 如果路径为空，直接返回
        if path is None:
            return
    
        # 根据路径获取迭代器
        iter = self.__hosts_store.get_iter(path)
    
        # 获取节点对象
        node = self.__nodes[self.__hosts_store.get_value(iter, 0)]
    
        # 调用父类的方法，改变笔记本显示的节点
        self.__parent.change_notebook(node)
    
    # 定义私有方法，用于主机排序
    def __host_sort(self, treemodel, iter1, iter2, *_):
        # 获取迭代器对应的值
        value1 = treemodel.get_value(iter1, 2)
        value2 = treemodel.get_value(iter2, 2)
    
        # 判断值是否为 IP 地址
        value1_is_ip = IP_RE.match(value1)
        value2_is_ip = IP_RE.match(value2)
    
        # 如果两个值都是 IP 地址
        if value1_is_ip and value2_is_ip:
            return ipv4_compare(value1, value2)
    
        # 如果只有一个值是 IP 地址
        if value1_is_ip:
            return -1
    
        if value2_is_ip:
            return 1
    
        # 如果都不是 IP 地址，按照字符串大小比较
        if value1 < value2:
            return -1
    
        if value1 > value2:
            return 1
    
        # 如果相等，返回 0
        return 0
```