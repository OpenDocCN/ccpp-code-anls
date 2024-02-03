# `nmap\zenmap\zenmapGUI\ScanNmapOutputPage.py`

```cpp
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
# 导入gi模块，用于与GTK+ 3.0进行交互
import gi

# 要求使用GTK+ 3.0版本
gi.require_version("Gtk", "3.0")
from gi.repository import Gtk, GObject, GdkPixbuf, Pango

# 导入os模块
import os

# 从zenmapGUI.higwidgets.higboxes模块中导入HIGHBox和HIGVBox类
from zenmapGUI.higwidgets.higboxes import HIGHBox, HIGVBox

# 从zenmapGUI.NmapOutputViewer模块中导入NmapOutputViewer类
from zenmapGUI.NmapOutputViewer import NmapOutputViewer

# 从zenmapGUI.ScanRunDetailsPage模块中导入ScanRunDetailsPage类
from zenmapGUI.ScanRunDetailsPage import ScanRunDetailsPage

# 从zenmapCore.Paths模块中导入Path类
from zenmapCore.Paths import Path

# 从zenmapCore.UmitLogging模块中导入log函数
from zenmapCore.UmitLogging import log

# 导入zenmapCore.I18N模块
import zenmapCore.I18N  # lgtm[py/unused-import]

# 定义一个名为scan_entry_data_func的函数，用于设置扫描条目的单元格渲染器属性
def scan_entry_data_func(widget, cell_renderer, model, iter):
    """Set the properties of a cell renderer for a scan entry."""
    # 设置单元格渲染器的属性
    cell_renderer.set_property("ellipsize", Pango.EllipsizeMode.END)
    cell_renderer.set_property("style", Pango.Style.NORMAL)
    cell_renderer.set_property("strikethrough", False)
    # 从模型中获取条目值
    entry = model.get_value(iter, 0)
    # 如果条目为空，则返回
    if entry is None:
        return
    # 如果任务正在运行
    if entry.running:
        # 设置单元格渲染器的样式为斜体
        cell_renderer.set_property("style", Pango.Style.ITALIC)
    # 如果任务已完成
    elif entry.finished:
        # 什么都不做
        pass
    # 如果任务失败或被取消
    elif entry.failed or entry.canceled:
        # 设置单元格渲染器的样式为删除线
        cell_renderer.set_property("strikethrough", True)
    # 设置单元格渲染器的文本内容为任务的命令字符串
    cell_renderer.set_property("text", entry.get_command_string())
class Throbber(Gtk.Image):
    """This is a little progress indicator that animates while a scan is
    running."""
    # 尝试加载静态和动态图片，如果失败则记录错误并将静态和动态图片设为None
    try:
        still = GdkPixbuf.Pixbuf.new_from_file(
                os.path.join(Path.pixmaps_dir, "throbber.png"))
        anim = GdkPixbuf.PixbufAnimation(
                os.path.join(Path.pixmaps_dir, "throbber.gif"))
    except Exception as e:
        log.debug("Error loading throbber images: %s." % str(e))
        still = None
        anim = None

    def __init__(self):
        Gtk.Image.__init__(self)
        self.set_from_pixbuf(self.still)  # 设置初始显示为静态图片
        self.animating = False

    def go(self):
        # 如果当前没有在动画且动态图片不为空，则设置显示为动态图片
        if not self.animating and self.anim is not None:
            self.set_from_animation(self.anim)
        self.animating = True

    def stop(self):
        # 如果当前在动画且静态图片不为空，则设置显示为静态图片
        if self.animating and self.still is not None:
            self.set_from_pixbuf(self.still)
        self.animating = False


class ScanNmapOutputPage(HIGVBox):
    """This is the "Nmap Output" scan results tab. It holds a text view of Nmap
    output. The constructor takes a ScansListStore, the contents of which are
    made selectable through a combo box. Details for completed scans are
    available and shown in separate windows. It emits the "changed" signal when
    the combo box selection changes."""

    __gsignals__ = {
        "changed": (GObject.SignalFlags.RUN_FIRST, GObject.TYPE_NONE, ())
    }
    # 初始化函数，接受一个 scans_store 参数
    def __init__(self, scans_store):
        # 调用父类 HIGVBox 的初始化函数
        HIGVBox.__init__(self)

        # 这是我们打开的详细窗口的缓存
        self._details_windows = {}

        # 设置控件之间的间距为 0
        self.set_spacing(0)

        # 创建一个水平布局容器
        hbox = HIGHBox()

        # 创建一个下拉列表，使用 scans_store 作为数据模型
        self.scans_list = Gtk.ComboBox.new_with_model(scans_store)
        # 创建一个文本渲染器
        cell = Gtk.CellRendererText()
        # 将文本渲染器添加到下拉列表中
        self.scans_list.pack_start(cell, True)
        # 设置下拉列表的数据函数
        self.scans_list.set_cell_data_func(cell, scan_entry_data_func)
        # 将下拉列表添加到水平布局容器中，并设置为扩展和填充
        hbox._pack_expand_fill(self.scans_list)

        # 监听下拉列表的选择变化事件，触发 _selection_changed 方法
        self.scans_list.connect("changed", self._selection_changed)
        # 监听 scans_store 的行变化事件，触发 _row_changed 方法
        scans_store.connect("row-changed", self._row_changed)
        # 监听 scans_store 的行删除事件，触发 _row_deleted 方法
        scans_store.connect("row-deleted", self._row_deleted)

        # 创建一个 Throbber 控件
        self.throbber = Throbber()
        # 将 Throbber 控件添加到水平布局容器中，并设置为不扩展和不填充
        hbox._pack_noexpand_nofill(self.throbber)

        # 创建一个带有标签 "Details" 的按钮
        self.details_button = Gtk.Button.new_with_label(_("Details"))
        # 监听按钮的点击事件，触发 _show_details 方法
        self.details_button.connect("clicked", self._show_details)
        # 将按钮添加到水平布局容器中，并设置为不扩展和不填充
        hbox._pack_noexpand_nofill(self.details_button)

        # 将水平布局容器添加到当前布局容器中，并设置为不扩展和不填充
        self._pack_noexpand_nofill(hbox)

        # 创建一个 NmapOutputViewer 控件
        self.nmap_output = NmapOutputViewer()
        # 将 NmapOutputViewer 控件添加到当前布局容器中，并设置为扩展和填充
        self._pack_expand_fill(self.nmap_output)

        # 调用 _update 方法
        self._update()

    # 设置活动条目为传入的迭代器
    def set_active_iter(self, i):
        """Set the active entry to an iterator into the ScansListStore
        referred to by this object."""
        self.scans_list.set_active_iter(i)

    # 获取当前活动条目的值
    def get_active_entry(self):
        iter = self.scans_list.get_active_iter()
        if iter is None:
            return None
        return self.scans_list.get_model().get_value(iter, 0)

    # 下拉列表选择变化的回调方法
    def _selection_changed(self, widget):
        """This callback is called when a scan in the list of scans is
        selected."""
        # 调用 _update 方法
        self._update()
        # 触发 "changed" 事件
        self.emit("changed")

    # scans_store 行变化的回调方法
    def _row_changed(self, model, path, i):
        """This callback is called when a row in the underlying scans store is
        changed."""
        # 如果当前选中的条目发生了变化，则更新界面
        if path[0] == self.scans_list.get_active():
            self._update()
    # 当底层扫描存储中的一行被删除时调用此回调函数
    def _row_deleted(self, model, path):
        # 调用_update()函数更新界面
        self._update()

    # 根据当前选择的条目更新界面
    def _update(self):
        # 获取当前选定的条目
        entry = self.get_active_entry()
        # 如果没有选定条目，则清空nmap输出，禁用详情按钮，停止加载动画，然后返回
        if entry is None:
            self.nmap_output.show_nmap_output("")
            self.details_button.set_sensitive(False)
            self.throbber.stop()
            return

        # 如果条目已解析
        if entry.parsed is not None:
            # 清空命令执行，获取nmap输出并显示，启用详情按钮
            self.nmap_output.set_command_execution(None)
            nmap_output = entry.parsed.get_nmap_output()
            if nmap_output:
                self.nmap_output.show_nmap_output(nmap_output)
            self.details_button.set_sensitive(True)
        # 如果条目有命令但未解析
        elif entry.command is not None:
            # 设置命令执行，刷新输出，禁用详情按钮
            self.nmap_output.set_command_execution(entry.command)
            self.nmap_output.refresh_output()
            self.details_button.set_sensitive(False)

        # 如果条目正在运行，则启动加载动画，否则停止加载动画
        if entry.running:
            self.throbber.go()
        else:
            self.throbber.stop()

    # 显示当前选定扫描的详细窗口（如果已完成）
    def _show_details(self, button):
        # 获取当前选定的条目
        entry = self.get_active_entry()
        # 如果没有选定条目，则返回
        if entry is None:
            return
        # 如果条目未完成，则返回
        if not entry.finished:
            return
        # 如果详情窗口不存在，则创建并显示
        if self._details_windows.get(entry) is None:
            window = Gtk.Window()
            window.add(ScanRunDetailsPage(entry.parsed))

            # 定义关闭详情窗口的回调函数
            def close_details(details, event, entry):
                details.destroy()
                del self._details_windows[entry]

            window.connect("delete-event", close_details, entry)
            window.show_all()
            self._details_windows[entry] = window
        # 显示详情窗口
        self._details_windows[entry].present()
```