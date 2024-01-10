# `nmap\zenmap\zenmapGUI\TopologyPage.py`

```
# 指定脚本解释器为 Python 3
#!/usr/bin/env python3

# ***********************IMPORTANT NMAP LICENSE TERMS************************
# *
# * The Nmap Security Scanner is (C) 1996-2023 Nmap Software LLC ("The Nmap
# * Project"). Nmap is also a registered trademark of the Nmap Project.
# *
# * This program is distributed under the terms of the Nmap Public Source
# * License (NPSL). The exact license text applying to a particular Nmap
# * release or source code control revision is contained in the LICENSE
# * file distributed with that version of Nmap or source code control
# * revision. More Nmap copyright/legal information is available from
# * https://nmap.org/book/man-legal.html, and further information on the
# * NPSL license itself can be found at https://nmap.org/npsl/ . This
# * header summarizes some key points from the Nmap license, but is no
# * substitute for the actual license text.
# *
# * Nmap is generally free for end users to download and use themselves,
# * including commercial use. It is available from https://nmap.org.
# *
# * The Nmap license generally prohibits companies from using and
# * redistributing Nmap in commercial products, but we sell a special Nmap
# * OEM Edition with a more permissive license and special features for
# * this purpose. See https://nmap.org/oem/
# *
# * If you have received a written Nmap license agreement or contract
# * stating terms other than these (such as an Nmap OEM license), you may
# * choose to use and redistribute Nmap under those terms instead.
# *
# * The official Nmap Windows builds include the Npcap software
# * (https://npcap.com) for packet capture and transmission. It is under
# * separate license terms which forbid redistribution without special
# * permission. So the official Nmap Windows builds may not be redistributed
# * without special permission (such as an Nmap OEM license).
# *
# * Source is provided to this software because we believe users have a
# * right to know exactly what a program is going to do before they run it.
# 导入 gi 模块
import gi

# 要求使用 Gtk 3.0 版本
gi.require_version("Gtk", "3.0")
# 从 gi.repository 中导入 Gtk 模块
from gi.repository import Gtk

# 从 zenmapGUI.higwidgets.higboxes 模块中导入 HIGVBox 类
from zenmapGUI.higwidgets.higboxes import HIGVBox

# 从 radialnet.gui.RadialNet 模块中导入 RadialNet 类
import radialnet.gui.RadialNet as RadialNet
# 从 radialnet.gui.ControlWidget 模块中导入 ControlWidget, ControlFisheye 类
from radialnet.gui.ControlWidget import ControlWidget, ControlFisheye
# 从 radialnet.gui.Toolbar 模块中导入 Toolbar 类
from radialnet.gui.Toolbar import Toolbar
# 从 radialnet.util.integration 模块中导入 make_graph_from_hosts 函数
from radialnet.util.integration import make_graph_from_hosts

# 设置 SLOW_LIMIT 常量值为 1000
SLOW_LIMIT = 1000

# 定义 TopologyPage 类，继承自 HIGVBox 类
class TopologyPage(HIGVBox):
    # 初始化方法
    def __init__(self, inventory):
        # 调用父类的初始化方法
        HIGVBox.__init__(self)

        # 设置边框宽度为 6
        self.set_border_width(6)
        # 设置间距为 4
        self.set_spacing(4)

        # 将传入的 inventory 参数赋值给 self.network_inventory 属性
        self.network_inventory = inventory

        # 调用 _create_widgets 方法
        self._create_widgets()
        # 调用 _pack_widgets 方法
        self._pack_widgets()
    # 创建小部件
    def _create_widgets(self):
        # 创建水平方向的 Box 容器
        self.rn_hbox = Gtk.Box.new(Gtk.Orientation.HORIZONTAL, 4)
        # 创建垂直方向的 Box 容器
        self.rn_vbox = Gtk.Box.new(Gtk.Orientation.VERTICAL, 0)

        # RadialNet 的小部件
        self.radialnet = RadialNet.RadialNet(RadialNet.LAYOUT_WEIGHTED)
        # 创建控制小部件
        self.control = ControlWidget(self.radialnet)
        # 创建鱼眼效果控制小部件
        self.fisheye = ControlFisheye(self.radialnet)
        # 创建工具栏
        self.rn_toolbar = Toolbar(self.radialnet,
                               self,
                               self.control,
                               self.fisheye)

        # 创建显示面板
        self.display_panel = HIGVBox()

        # 设置 RadialNet 不立即显示
        self.radialnet.set_no_show_all(True)

        # 创建垂直方向的 Box 容器
        self.slow_vbox = HIGVBox()
        # 创建标签
        self.slow_label = Gtk.Label()
        self.slow_vbox.pack_start(self.slow_label, False, False, 0)
        # 创建按钮并连接点击事件
        show_button = Gtk.Button.new_with_label(_("Show the topology anyway"))
        show_button.connect("clicked", self.show_anyway)
        self.slow_vbox.pack_start(show_button, False, False, 0)
        self.slow_vbox.show_all()
        self.slow_vbox.set_no_show_all(True)
        self.slow_vbox.hide()

        # 显示 RadialNet
        self.radialnet.show()

    # 将小部件打包
    def _pack_widgets(self):
        self.rn_hbox.pack_start(self.display_panel, True, True, 0)
        self.rn_hbox.pack_start(self.control, False, True, 0)

        self.rn_vbox.pack_start(self.rn_hbox, True, True, 0)
        self.rn_vbox.pack_start(self.fisheye, False, True, 0)

        self.pack_start(self.rn_toolbar, False, False, 0)
        self.pack_start(self.rn_vbox, True, True, 0)

        self.display_panel.pack_start(self.slow_vbox, True, False, 0)
        self.display_panel.pack_start(self.radialnet, True, True, 0)

    # 添加扫描
    def add_scan(self, scan):
        """解析给定的 XML 文件并将解析结果添加到网络清单中。"""
        self.network_inventory.add_scan(scan)
        self.update_radialnet()
        # 更新径向网络图
        """从网络清单的主机列表创建图形并显示"""
        # 获取网络清单中处于上线状态的主机列表
        hosts_up = self.network_inventory.get_hosts_up()
        
        # 设置慢速标签的文本内容
        self.slow_label.set_text(_("""\
# 如果主机数量过多，会导致拓扑图运行缓慢，因此禁用拓扑图
# 输出拓扑图运行缓慢的限制主机数量和当前主机数量
""" % (SLOW_LIMIT, len(hosts_up))))

# 如果当前主机数量小于等于限制数量，显示拓扑图并隐藏慢速提示框，然后更新拓扑图
if len(hosts_up) <= SLOW_LIMIT:
    self.radialnet.show()
    self.slow_vbox.hide()
    self.update_radialnet_unchecked()
# 如果当前主机数量大于限制数量，隐藏拓扑图并显示慢速提示框
else:
    self.radialnet.hide()
    self.slow_vbox.show()

# 更新未选中的拓扑图
def update_radialnet_unchecked(self):
    # 获取当前在线的主机
    hosts_up = self.network_inventory.get_hosts_up()
    # 根据在线主机创建图形
    graph = make_graph_from_hosts(hosts_up)
    # 设置拓扑图为空
    self.radialnet.set_empty()
    # 设置拓扑图的图形
    self.radialnet.set_graph(graph)
    # 显示拓扑图
    self.radialnet.show()

# 无论如何显示拓扑图
def show_anyway(self, widget):
    # 显示拓扑图并隐藏慢速提示框，然后更新拓扑图
    self.radialnet.show()
    self.slow_vbox.hide()
    self.update_radialnet_unchecked()
```