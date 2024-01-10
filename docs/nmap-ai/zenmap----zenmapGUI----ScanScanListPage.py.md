# `nmap\zenmap\zenmapGUI\ScanScanListPage.py`

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
# 导入gi模块，需要使用其中的功能
import gi

# 要求使用GTK版本3.0
gi.require_version("Gtk", "3.0")
# 从gi.repository中导入Gtk和Pango模块
from gi.repository import Gtk, Pango

# 从zenmapGUI.higwidgets.higboxes模块中导入HIGHBox和HIGVBox类
from zenmapGUI.higwidgets.higboxes import HIGHBox, HIGVBox
# 从zenmapGUI.higwidgets.higbuttons模块中导入HIGButton类
from zenmapGUI.higwidgets.higbuttons import HIGButton
# 从zenmapGUI.higwidgets.higscrollers模块中导入HIGScrolledWindow类
from zenmapGUI.higwidgets.higscrollers import HIGScrolledWindow
# 导入zenmapCore.I18N模块，但未使用，可能是未使用的导入
import zenmapCore.I18N

# 定义一个函数，用于在列表中显示状态数据
def status_data_func(widget, cell_renderer, model, iter, data):
    # 从模型中获取条目
    entry = model.get_value(iter, 0)
    # 根据条目的状态设置对应的文本
    if entry.running:
        status = _("Running")
    elif entry.finished:
        if entry.parsed is not None and entry.parsed.unsaved:
            status = _("Unsaved")
        else:
            status = ""
    elif entry.failed:
        status = _("Failed")
    elif entry.canceled:
        status = _("Canceled")
    # 设置单元格渲染器的文本属性
    cell_renderer.set_property("text", status)
# 定义一个函数，用于设置单元格渲染器的数据
def command_data_func(widget, cell_renderer, model, iter, data):
    # 从模型中获取指定行的数值
    entry = model.get_value(iter, 0)
    # 设置单元格渲染器的文本截断模式为末尾省略
    cell_renderer.set_property("ellipsize", Pango.EllipsizeMode.END)
    # 设置单元格渲染器的文本内容为命令字符串
    cell_renderer.set_property("text", entry.get_command_string())

# 定义一个类，表示“扫描”扫描结果选项卡
class ScanScanListPage(HIGVBox):
    """This is the "Scans" scan results tab. It the list of running and
    finished scans contained in the ScansListStore passed to the
    constructor."""
    # 初始化函数，接受一个 scans_store 参数
    def __init__(self, scans_store):
        # 调用父类 HIGVBox 的初始化函数
        HIGVBox.__init__(self)

        # 设置子部件之间的间距
        self.set_spacing(4)

        # 连接 scans_store 的 "row-changed" 信号到 _row_changed 方法
        scans_store.connect("row-changed", self._row_changed)

        # 创建一个包含 scans_store 数据模型的树视图
        self.scans_list = Gtk.TreeView.new_with_model(scans_store)
        # 连接树视图的选择变化信号到 _selection_changed 方法
        self.scans_list.get_selection().connect(
                "changed", self._selection_changed)

        # 创建一个标题为 "Status" 的树视图列
        status_col = Gtk.TreeViewColumn(title=_("Status"))
        # 创建一个文本单元格渲染器
        cell = Gtk.CellRendererText()
        # 将渲染器添加到列中
        status_col.pack_start(cell, True)
        # 设置单元格数据函数为 status_data_func
        status_col.set_cell_data_func(cell, status_data_func)
        # 将列添加到树视图中
        self.scans_list.append_column(status_col)

        # 创建一个标题为 "Command" 的树视图列
        command_col = Gtk.TreeViewColumn(title=_("Command"))
        # 创建一个文本单元格渲染器
        cell = Gtk.CellRendererText()
        # 将渲染器添加到列中
        command_col.pack_start(cell, True)
        # 设置单元格数据函数为 command_data_func
        command_col.set_cell_data_func(cell, command_data_func)
        # 将列添加到树视图中
        self.scans_list.append_column(command_col)

        # 创建一个滚动窗口
        scrolled_window = HIGScrolledWindow()
        scrolled_window.set_border_width(0)
        scrolled_window.add(self.scans_list)

        # 将滚动窗口添加到当前 Vbox 中
        self.pack_start(scrolled_window, True, True, 0)

        # 创建一个水平布局
        hbox = HIGHBox()
        # 创建一个按钮盒子
        buttonbox = Gtk.ButtonBox.new(Gtk.Orientation.HORIZONTAL)
        buttonbox.set_layout(Gtk.ButtonBoxStyle.START)
        buttonbox.set_spacing(4)

        # 创建一个标题为 "Append Scan" 的按钮
        self.append_button = HIGButton(_("Append Scan"), Gtk.STOCK_ADD)
        # 将按钮添加到按钮盒子中
        buttonbox.pack_start(self.append_button, False, True, 0)

        # 创建一个标题为 "Remove Scan" 的按钮
        self.remove_button = HIGButton(_("Remove Scan"), Gtk.STOCK_REMOVE)
        # 将按钮添加到按钮盒子中
        buttonbox.pack_start(self.remove_button, False, True, 0)

        # 创建一个标题为 "Cancel Scan" 的按钮
        self.cancel_button = HIGButton(_("Cancel Scan"), Gtk.STOCK_CANCEL)
        # 将按钮添加到按钮盒子中
        buttonbox.pack_start(self.cancel_button, False, True, 0)

        # 将按钮盒子添加到水平布局中
        hbox.pack_start(buttonbox, True, True, 4)

        # 将水平布局添加到当前 Vbox 中
        self.pack_start(hbox, False, True, 4)

        # 调用 _update 方法
        self._update()

    # 当数据模型的行发生变化时调用的方法
    def _row_changed(self, model, path, i):
        # 调用 _update 方法
        self._update()

    # 当选择发生变化时调用的方法
    def _selection_changed(self, selection):
        # 调用 _update 方法
        self._update()
    # 更新界面状态，根据当前选中的运行扫描来设置取消按钮的可用性
    tree_selection = self.scans_list.get_selection()
    # 如果没有选中任何扫描，则将 model 和 selection 设置为 None 和空列表
    if tree_selection is None:
        model, selection = None, []
    else:
        # 获取选中的行的 model 和 selection
        model, selection = tree_selection.get_selected_rows()

    # 遍历选中的行
    for path in selection:
        # 获取选中行的 entry 对象
        entry = model.get_value(model.get_iter(path), 0)
        # 如果选中的扫描正在运行，则设置取消按钮为可用，并跳出循环
        if entry.running:
            self.cancel_button.set_sensitive(True)
            break
    # 如果没有选中正在运行的扫描，则设置取消按钮为不可用
    else:
        self.cancel_button.set_sensitive(False)

    # 如果没有选中任何行，则设置移除按钮为不可用
    if len(selection) == 0:
        self.remove_button.set_sensitive(False)
    # 如果选中了行，则设置移除按钮为可用
    else:
        self.remove_button.set_sensitive(True)
```