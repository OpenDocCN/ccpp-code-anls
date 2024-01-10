# `nmap\zenmap\zenmapGUI\ScanHostsView.py`

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
# 导入gi模块，要求使用版本为"Gtk"和"3.0"
import gi
gi.require_version("Gtk", "3.0")
# 从gi.repository中导入Gtk模块
from gi.repository import Gtk
# 从zenmapGUI.higwidgets.higboxes中导入HIGVBox类
from zenmapGUI.higwidgets.higboxes import HIGVBox
# 从zenmapGUI.Icons中导入get_os_icon函数
from zenmapGUI.Icons import get_os_icon
# 导入zenmapCore.I18N模块
import zenmapCore.I18N  # lgtm[py/unused-import]

# 定义函数treemodel_get_addrs_for_sort，用于获取排序后的地址
def treemodel_get_addrs_for_sort(model, iter):
    # 从模型中获取主机
    host = model.get_value(iter, 0)
    # 返回主机的排序后地址
    return host.get_addrs_for_sort()

# 用于按地址排序主机的函数
def cmp_treemodel_addr(model, iter_a, iter_b, *_):
    # 定义内部比较函数cmp
    def cmp(a, b):
        return (a > b) - (a < b)
    # 获取iter_a和iter_b的排序后地址
    addrs_a = treemodel_get_addrs_for_sort(model, iter_a)
    addrs_b = treemodel_get_addrs_for_sort(model, iter_b)
    # 返回排序后地址的比较结果
    return cmp(addrs_a, addrs_b)

# 定义ScanHostsView类，继承自HIGVBox类
class ScanHostsView(HIGVBox):
    # 定义常量HOST_MODE和SERVICE_MODE，分别为0和1
    HOST_MODE, SERVICE_MODE = list(range(2))
    # 初始化函数，接受一个扫描接口参数
    def __init__(self, scan_interface):
        # 调用父类的初始化函数
        HIGVBox.__init__(self)

        # 保存扫描接口参数
        self._scan_interface = scan_interface
        # 创建窗口部件
        self._create_widgets()
        # 连接窗口部件
        self._connect_widgets()
        # 将窗口部件打包
        self._pack_widgets()
        # 设置滚动
        self._set_scrolled()
        # 设置主机列表
        self._set_host_list()
        # 设置服务列表
        self._set_service_list()

        # 将主窗口vbox部件打包并填充
        self._pack_expand_fill(self.main_vbox)

        # 默认模式为空
        self.mode = None

        # 默认模式为主机模式
        self.host_mode(self.host_mode_button)

        # 显示主机视图
        self.host_view.show_all()
        # 显示服务视图
        self.service_view.show_all()

    # 创建窗口部件函数
    def _create_widgets(self):
        # 模式按钮
        self.host_mode_button = Gtk.ToggleButton.new_with_label(_("Hosts"))
        self.service_mode_button = Gtk.ToggleButton.new_with_label(_("Services"))
        self.buttons_box = Gtk.Box.new(Gtk.Orientation.HORIZONTAL, 0)

        # 主窗口vbox
        self.main_vbox = HIGVBox()

        # 主机列表
        self.host_list = Gtk.ListStore.new([object, str, str])
        self.host_list.set_sort_func(1000, cmp_treemodel_addr)
        self.host_list.set_sort_column_id(1000, Gtk.SortType.ASCENDING)
        self.host_view = Gtk.TreeView.new_with_model(self.host_list)
        self.pic_column = Gtk.TreeViewColumn(title=_('OS'))
        self.host_column = Gtk.TreeViewColumn(title=_('Host'))
        self.os_cell = Gtk.CellRendererPixbuf()
        self.host_cell = Gtk.CellRendererText()

        # 服务列表
        self.service_list = Gtk.ListStore.new([str])
        self.service_list.set_sort_column_id(0, Gtk.SortType.ASCENDING)
        self.service_view = Gtk.TreeView.new_with_model(self.service_list)
        self.service_column = Gtk.TreeViewColumn(title=_('Service'))
        self.service_cell = Gtk.CellRendererText()

        # 滚动窗口
        self.scrolled = Gtk.ScrolledWindow()
    # 封装小部件
    def _pack_widgets(self):
        # 设置主垂直框的间距为0
        self.main_vbox.set_spacing(0)
        # 设置主垂直框的边框宽度为0
        self.main_vbox.set_border_width(0)
        # 将按钮框添加到主垂直框中，不扩展，不填充
        self.main_vbox._pack_noexpand_nofill(self.buttons_box)
        # 将滚动条添加到主垂直框中，扩展，填充
        self.main_vbox._pack_expand_fill(self.scrolled)
    
        # 设置主机模式按钮为激活状态
        self.host_mode_button.set_active(True)
    
        # 设置按钮框的边框宽度为5
        self.buttons_box.set_border_width(5)
        # 在按钮框中添加主机模式按钮，扩展，填充，间距为0
        self.buttons_box.pack_start(self.host_mode_button, True, True, 0)
        # 在按钮框中添加服务模式按钮，扩展，填充，间距为0
        self.buttons_box.pack_start(self.service_mode_button, True, True, 0)
    
    # 连接小部件
    def _connect_widgets(self):
        # 当主机模式按钮被切换时，调用主机模式方法
        self.host_mode_button.connect("toggled", self.host_mode)
        # 当服务模式按钮被切换时，调用服务模式方法
        self.service_mode_button.connect("toggled", self.service_mode)
    
    # 主机模式方法
    def host_mode(self, widget):
        # 移除滚动条中的子元素
        self._remove_scrolled_child()
        # 如果主机模式按钮被激活
        if widget.get_active():
            # 设置模式为主机模式
            self.mode = self.HOST_MODE
            # 关闭服务模式按钮
            self.service_mode_button.set_active(False)
            # 在滚动条中添加主机视图
            self.scrolled.add(self.host_view)
        else:
            # 打开服务模式按钮
            self.service_mode_button.set_active(True)
    
    # 服务模式方法
    def service_mode(self, widget):
        # 移除滚动条中的子元素
        self._remove_scrolled_child()
        # 如果服务模式按钮被激活
        if widget.get_active():
            # 设置模式为服务模式
            self.mode = self.SERVICE_MODE
            # 关闭主机模式按钮
            self.host_mode_button.set_active(False)
            # 在滚动条中添加服务视图
            self.scrolled.add(self.service_view)
        else:
            # 打开主机模式按钮
            self.host_mode_button.set_active(True)
    
    # 移除滚动条中的子元素
    def _remove_scrolled_child(self):
        try:
            # 获取滚动条中的子元素
            child = self.scrolled.get_child()
            # 移除子元素
            self.scrolled.remove(child)
        except Exception:
            # 如果出现异常，则忽略
            pass
    
    # 设置滚动条
    def _set_scrolled(self):
        # 设置滚动条的边框宽度为5
        self.scrolled.set_border_width(5)
        # 设置滚动条的大小请求为150，高度不限制
        self.scrolled.set_size_request(150, -1)
        # 设置滚动条的滚动策略为自动
        self.scrolled.set_policy(Gtk.PolicyType.AUTOMATIC, Gtk.PolicyType.AUTOMATIC)
    # 设置服务列表的属性
    def _set_service_list(self):
        # 设置服务列表可搜索
        self.service_view.set_enable_search(True)
        # 设置搜索列为第一列
        self.service_view.set_search_column(0)
    
        # 获取服务列表的选择对象
        selection = self.service_view.get_selection()
        # 设置选择模式为多选
        selection.set_mode(Gtk.SelectionMode.MULTIPLE)
        # 在服务列表中添加服务列
        self.service_view.append_column(self.service_column)
    
        # 设置服务列可调整大小
        self.service_column.set_resizable(True)
        # 设置按第一列排序
        self.service_column.set_sort_column_id(0)
        # 设置列可重新排序
        self.service_column.set_reorderable(True)
        # 在服务列中添加服务单元格
        self.service_column.pack_start(self.service_cell, True)
        # 设置服务单元格的属性为显示第一列的文本
        self.service_column.set_attributes(self.service_cell, text=0)
    
    # 设置主机列表的属性
    def _set_host_list(self):
        # 设置主机列表可搜索
        self.host_view.set_enable_search(True)
        # 设置搜索列为第二列
        self.host_view.set_search_column(1)
    
        # 获取主机列表的选择对象
        selection = self.host_view.get_selection()
        # 设置选择模式为多选
        selection.set_mode(Gtk.SelectionMode.MULTIPLE)
    
        # 在主机列表中添加图片列和主机列
        self.host_view.append_column(self.pic_column)
        self.host_view.append_column(self.host_column)
    
        # 设置主机列和图片列可调整大小
        self.host_column.set_resizable(True)
        self.pic_column.set_resizable(True)
    
        # 设置主机列和图片列的排序列ID
        self.host_column.set_sort_column_id(1000)
        self.pic_column.set_sort_column_id(1)
    
        # 设置主机列和图片列可重新排序
        self.host_column.set_reorderable(True)
        self.pic_column.set_reorderable(True)
    
        # 在图片列中添加操作系统单元格
        self.pic_column.pack_start(self.os_cell, True)
        # 在主机列中添加主机名单元格
        self.host_column.pack_start(self.host_cell, True)
    
        # 设置图片列的最小宽度
        self.pic_column.set_min_width(35)
        # 设置图片列的属性为显示第一列的图片
        self.pic_column.set_attributes(self.os_cell, stock_id=1)
        # 设置主机列的属性为显示第二列的文本
        self.host_column.set_attributes(self.host_cell, text=2)
    
    # 添加主机到主机列表
    def add_host(self, host):
        # 向主机列表中添加主机、操作系统图标和主机名
        self.host_list.append([host, get_os_icon(host), host.get_hostname()])
    
    # 添加服务到服务列表
    def add_service(self, service):
        # 向服务列表中添加服务
        self.service_list.append([service])
# 如果当前模块被直接执行，则执行以下代码
if __name__ == "__main__":
    # 创建一个 GTK 窗口对象
    w = Gtk.Window()
    # 创建一个 ScanHostsView 对象
    h = ScanHostsView(None)
    # 将 ScanHostsView 对象添加到窗口中
    w.add(h)

    # 连接窗口的 "delete-event" 信号，当窗口关闭时退出 GTK 主循环
    w.connect("delete-event", lambda x, y: Gtk.main_quit())
    # 显示窗口及其所有子部件
    w.show_all()
    # 进入 GTK 主循环
    Gtk.main()
```