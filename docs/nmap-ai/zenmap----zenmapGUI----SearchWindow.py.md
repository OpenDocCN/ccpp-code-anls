# `nmap\zenmap\zenmapGUI\SearchWindow.py`

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
# 从 zenmapGUI.SearchGUI 中导入 SearchGUI 模块
from zenmapGUI.SearchGUI import SearchGUI
# 从 zenmapCore.I18N 中导入 is_maemo 模块
import zenmapCore.I18N  # lgtm[py/unused-import]
# 从 zenmapCore.UmitConf 中导入 is_maemo 模块
from zenmapCore.UmitConf import is_maemo
# 从 zenmapGUI.higwidgets.higboxes 中导入 HIGVBox 模块
from zenmapGUI.higwidgets.higboxes import HIGVBox
# 从 zenmapGUI.higwidgets.higbuttons 中导入 HIGButton 模块
from zenmapGUI.higwidgets.higbuttons import HIGButton

# 初始化 BaseSearchWindow 和 hildon 为 None
BaseSearchWindow = None
hildon = None

# 如果是 maemo 平台
if is_maemo():
    # 导入 hildon 模块
    import hildon
    # 定义 BaseSearchWindow 类，继承自 hildon.Window
    class BaseSearchWindow(hildon.Window):
        # 初始化方法
        def __init__(self):
            # 调用父类的初始化方法
            hildon.Window.__init__(self)
        # 定义 _pack_widgets 方法
        def _pack_widgets(self):
            # 空方法，暂无实现
            pass
# 如果不是 maemo 平台
else:
    # 创建一个名为 BaseSearchWindow 的类，继承自 Gtk.Window
    class BaseSearchWindow(Gtk.Window):
        # 初始化方法
        def __init__(self):
            # 调用父类的初始化方法
            Gtk.Window.__init__(self)
            # 设置窗口标题为 "Search Scans"
            self.set_title(_("Search Scans"))
            # 设置窗口位置为居中
            self.set_position(Gtk.WindowPosition.CENTER)
    
        # 定义一个私有方法 _pack_widgets
        def _pack_widgets(self):
            # 设置垂直布局容器的边框宽度为 4
            self.vbox.set_border_width(4)
# 定义一个名为 SearchWindow 的类，继承自 BaseSearchWindow
class SearchWindow(BaseSearchWindow):
    # 初始化方法，接受 load_method 和 append_method 两个参数
    def __init__(self, load_method, append_method):
        # 调用父类的初始化方法
        BaseSearchWindow.__init__(self)

        # 设置窗口默认大小为 600x400
        self.set_default_size(600, 400)

        # 将传入的 load_method 和 append_method 分别赋值给实例变量
        self.load_method = load_method
        self.append_method = append_method

        # 创建窗口部件
        self._create_widgets()
        # 将部件添加到窗口中
        self._pack_widgets()
        # 连接部件的信号和槽
        self._connect_widgets()

    # 创建窗口部件的方法
    def _create_widgets(self):
        # 创建垂直布局容器
        self.vbox = HIGVBox()

        # 创建水平布局容器
        self.bottom_hbox = Gtk.Box.new(Gtk.Orientation.HORIZONTAL, 4)
        # 创建标签部件
        self.bottom_label = Gtk.Label()
        # 创建按钮盒子
        self.btn_box = Gtk.ButtonBox.new(Gtk.Orientation.HORIZONTAL)
        # 创建打开按钮
        self.btn_open = HIGButton(stock=Gtk.STOCK_OPEN)
        # 创建追加按钮
        self.btn_append = HIGButton(_("Append"), Gtk.STOCK_ADD)
        # 创建关闭按钮
        self.btn_close = HIGButton(stock=Gtk.STOCK_CLOSE)

        # 创建 SearchGUI 实例
        self.search_gui = SearchGUI(self)

    # 将部件添加到窗口中的方法
    def _pack_widgets(self):
        # 调用父类的将部件添加到窗口中的方法
        BaseSearchWindow._pack_widgets(self)

        # 设置按钮盒子的布局样式和间距
        self.btn_box.set_layout(Gtk.ButtonBoxStyle.END)
        self.btn_box.set_spacing(4)
        # 将关闭、追加、打开按钮添加到按钮盒子中
        self.btn_box.pack_start(self.btn_close, True, True, 0)
        self.btn_box.pack_start(self.btn_append, True, True, 0)
        self.btn_box.pack_start(self.btn_open, True, True, 0)

        # 设置底部标签的对齐方式和使用标记语言
        self.bottom_label.set_alignment(0.0, 0.5)
        self.bottom_label.set_use_markup(True)

        # 将底部标签和按钮盒子添加到底部水平布局容器中
        self.bottom_hbox.pack_start(self.bottom_label, True, True, 0)
        self.bottom_hbox.pack_start(self.btn_box, False, True, 0)

        # 设置垂直布局容器的间距，并将搜索界面和底部水平布局容器添加到其中
        self.vbox.set_spacing(4)
        self.vbox.pack_start(self.search_gui, True, True, 0)
        self.vbox.pack_start(self.bottom_hbox, False, True, 0)

        # 将垂直布局容器添加到窗口中
        self.add(self.vbox)

    # 连接部件的方法
    def _connect_widgets(self):
        # 双击结果时打开选中项
        self.search_gui.result_view.connect(
                "row-activated", self.open_selected)

        # 点击打开按钮时打开选中项
        self.btn_open.connect("clicked", self.open_selected)
        # 点击追加按钮时追加选中项
        self.btn_append.connect("clicked", self.append_selected)
        # 点击关闭按钮时关闭窗口
        self.btn_close.connect("clicked", self.close)
        # 关闭窗口时触发 delete-event 信号，关闭窗口
        self.connect("delete-event", self.close)
    # 关闭窗口的方法，关闭搜索界面并销毁窗口
    def close(self, widget=None, event=None):
        self.search_gui.close()
        self.destroy()

    # 设置底部标签的文本内容
    def set_label_text(self, text):
        self.bottom_label.set_label(text)

    # 打开选定的结果
    def open_selected(self, widget=None, path=None, view_column=None,
            extra=None):
        # 调用加载方法加载结果
        self.load_method(self.results)

        # 关闭搜索窗口
        self.close()

    # 追加选定的结果
    def append_selected(self, widget=None, path=None, view_column=None,
            extra=None):
        # 调用追加方法追加结果
        self.append_method(self.results)

        # 关闭搜索窗口
        self.close()

    # 获取结果列表
    def get_results(self):
        # 从结果列表存储中返回解析后的对象列表
        return self.search_gui.selected_results

    # 将获取结果的方法设置为属性
    results = property(get_results)
# 如果当前模块被直接执行，则执行以下代码
if __name__ == "__main__":
    # 创建一个搜索窗口对象，传入两个回调函数作为参数
    search = SearchWindow(lambda x: Gtk.main_quit(), lambda x: Gtk.main_quit())
    # 显示搜索窗口
    search.show_all()
    # 运行 GTK 主循环
    Gtk.main()
```