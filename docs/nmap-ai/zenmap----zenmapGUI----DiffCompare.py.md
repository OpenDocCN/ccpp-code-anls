# `nmap\zenmap\zenmapGUI\DiffCompare.py`

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
# 导入gi模块，确保使用的是Gtk 3.0版本
import gi
gi.require_version("Gtk", "3.0")
from gi.repository import Gtk, GObject, GLib

# 导入os、os.path和sys模块
import os
import os.path
import sys

# 防止加载PyXML
import xml
xml.__path__ = [x for x in xml.__path__ if "_xmlplus" not in x]

# 导入xml.sax模块
import xml.sax

# 导入zenmapGUI.higwidgets.higdialogs、zenmapGUI.higwidgets.higboxes、zenmapGUI.higwidgets.higlabels、
# zenmapGUI.higwidgets.higtables和zenmapGUI.higwidgets.higbuttons模块
from zenmapGUI.higwidgets.higdialogs import HIGAlertDialog
from zenmapGUI.higwidgets.higboxes import HIGVBox, HIGHBox, hig_box_space_holder
from zenmapGUI.higwidgets.higlabels import HIGSectionLabel
from zenmapGUI.higwidgets.higtables import HIGTable
from zenmapGUI.higwidgets.higbuttons import HIGButton

# 导入zenmapCore.NmapParser和zenmapCore.UmitLogging模块
from zenmapCore.NmapParser import NmapParser
from zenmapCore.UmitLogging import log
import zenmapCore.I18N  # lgtm[py/unused-import]
import zenmapCore.Diff

# 导入zenmapGUI.FileChoosers模块中的ResultsFileSingleChooserDialog类
from zenmapGUI.FileChoosers import ResultsFileSingleChooserDialog

# 设置超时时间为毫秒
# In milliseconds.
# 设置超时时间为200毫秒
NDIFF_CHECK_TIMEOUT = 200


class ScanChooser(HIGVBox):
    """This class allows the selection of scan results from the list of open
    tabs or from a file. It emits the "changed" signal when the scan selection
    has changed."""

    # 定义信号"changed"
    __gsignals__ = {
        "changed": (GObject.SignalFlags.RUN_FIRST, GObject.TYPE_NONE, ())
    }

    # 初始化方法
    def __init__(self, scans, title):
        # 调用父类的初始化方法
        HIGVBox.__init__(self)

        # 设置标题和扫描结果字典
        self.title = title
        self.scan_dict = {}

        # 设置HIGVBox的边框宽度和间距
        self.set_border_width(5)
        self.set_spacing(6)

        # 创建各种小部件
        self._create_widgets()
        self._pack_hbox()
        self._attaching_widgets()
        self._set_scrolled()
        self._set_text_view()
        self._set_open_button()

        # 遍历扫描结果列表，添加到界面中
        for scan in scans:
            self.add_scan(scan.scan_name or scan.get_nmap_command(), scan)

        # 连接下拉框的changed信号到show_scan方法
        self.combo_scan.connect('changed', self.show_scan)
        # 连接下拉框的changed信号到发射"changed"信号
        self.combo_scan.connect('changed', lambda x: self.emit('changed'))

        # 将标签和水平盒子添加到界面中
        self._pack_noexpand_nofill(self.lbl_scan)
        self._pack_expand_fill(self.hbox)

    # 创建各种小部件的方法
    def _create_widgets(self):
        self.lbl_scan = HIGSectionLabel(self.title)
        self.hbox = HIGHBox()
        self.table = HIGTable()
        self.combo_scan = Gtk.ComboBoxText.new_with_entry()
        self.btn_open_scan = Gtk.Button.new_from_stock(Gtk.STOCK_OPEN)
        self.exp_scan = Gtk.Expander.new(_("Scan Output"))
        self.scrolled = Gtk.ScrolledWindow()
        self.txt_scan_result = Gtk.TextView()
        self.txg_tag = Gtk.TextTag.new("scan_style")

    # 获取文本视图的缓冲区
    def get_buffer(self):
        return self.txt_scan_result.get_buffer()

    # 显示扫描结果的方法
    def show_scan(self, widget):
        nmap_output = self.get_nmap_output()
        if nmap_output:
            self.txt_scan_result.get_buffer().set_text(nmap_output)

    # 标准化输出的方法
    def normalize_output(self, output):
        return "\n".join(output.split("\\n"))
    # 将 hbox 中的控件进行布局，不扩展也不填充
    def _pack_hbox(self):
        self.hbox._pack_noexpand_nofill(hig_box_space_holder())
        self.hbox._pack_expand_fill(self.table)
    
    # 将控件添加到表格中
    def _attaching_widgets(self):
        self.table.attach(self.combo_scan, 0, 1, 0, 1, yoptions=0)
        self.table.attach(
            self.btn_open_scan, 1, 2, 0, 1, yoptions=0, xoptions=0)
        self.table.attach(self.exp_scan, 0, 2, 1, 2)
    
    # 设置滚动窗口的边框宽度和大小
    def _set_scrolled(self):
        self.scrolled.set_border_width(5)
        self.scrolled.set_size_request(-1, 130)
    
        # 将滚动窗口放入扩展器中
        self.exp_scan.add(self.scrolled)
    
        # 将文本视图放入滚动窗口
        self.scrolled.add_with_viewport(self.txt_scan_result)
    
        # 设置滚动窗口的滚动策略
        self.scrolled.set_policy(Gtk.PolicyType.AUTOMATIC, Gtk.PolicyType.AUTOMATIC)
    
    # 设置文本视图的属性
    def _set_text_view(self):
        self.txg_table = self.txt_scan_result.get_buffer().get_tag_table()
        self.txg_table.add(self.txg_tag)
        self.txg_tag.set_property("family", "Monospace")
    
        self.txt_scan_result.set_wrap_mode(Gtk.WrapMode.WORD)
        self.txt_scan_result.set_editable(False)
        self.txt_scan_result.get_buffer().connect(
            "changed", self._text_changed_cb)
    
    # 设置打开按钮的点击事件
    def _set_open_button(self):
        self.btn_open_scan.connect('clicked', self.open_file)
    # 打开文件的回调函数，用于选择扫描结果文件
    def open_file(self, widget):
        # 创建文件选择对话框
        file_chooser = ResultsFileSingleChooserDialog(_("Select Scan Result"))

        # 运行文件选择对话框并获取用户的响应
        response = file_chooser.run()
        # 获取用户选择的文件名
        file_chosen = file_chooser.get_filename()
        # 销毁文件选择对话框
        file_chooser.destroy()
        # 如果用户选择了文件
        if response == Gtk.ResponseType.OK:
            try:
                # 创建 NmapParser 对象并解析选定的文件
                parser = NmapParser()
                parser.parse_file(file_chosen)
            # 捕获 XML 解析异常
            except xml.sax.SAXParseException as e:
                # 显示解析文件错误的警告对话框
                alert = HIGAlertDialog(
                    message_format='<b>%s</b>' % _('Error parsing file'),
                    secondary_text=_(
                        "The file is not an Nmap XML output file. "
                        "The parsing error that occurred was\n%s") % str(e))
                alert.run()
                alert.destroy()
                return False
            # 捕获其他异常
            except Exception as e:
                # 显示无法打开文件的警告对话框
                alert = HIGAlertDialog(
                    message_format='<b>%s</b>' % _(
                        'Cannot open selected file'),
                    secondary_text=_("""\
                        This error occurred while trying to open the file:
                        %s""") % str(e))
                alert.run()
                alert.destroy()
                return False

            # 获取扫描文件的名称
            scan_name = os.path.split(file_chosen)[-1]
            # 将扫描结果添加到界面中
            self.add_scan(scan_name, parser)

            # 设置下拉框中最新添加的扫描结果为当前选中项
            self.combo_scan.set_active(len(self.combo_scan.get_model()) - 1)

    # 添加扫描结果到界面中
    def add_scan(self, scan_name, parser):
        # 初始化扫描 ID 和新的扫描名称
        scan_id = 1
        new_scan_name = scan_name
        # 如果新的扫描名称已经存在于扫描字典中，则添加序号
        while new_scan_name in self.scan_dict.keys():
            new_scan_name = "%s (%s)" % (scan_name, scan_id)
            scan_id += 1

        # 在下拉框中添加新的扫描名称
        self.combo_scan.append_text(new_scan_name)
        # 将新的扫描结果添加到扫描字典中
        self.scan_dict[new_scan_name] = parser

    # 文本内容改变的回调函数，用于应用文本标签
    def _text_changed_cb(self, widget):
        # 获取文本缓冲区
        buff = self.txt_scan_result.get_buffer()
        # 应用文本标签到整个文本内容
        buff.apply_tag(
            self.txg_tag, buff.get_start_iter(), buff.get_end_iter())
    # 返回当前选择的扫描的解析输出作为 NmapParser 对象，如果没有有效的扫描被选择则返回 None
    def get_parsed_scan(self):
        # 获取当前选择的扫描
        selected_scan = self.combo_scan.get_active_text()
        # 返回对应扫描的解析输出
        return self.scan_dict.get(selected_scan)
    
    # 返回当前选择的扫描的输出作为字符串，如果没有有效的扫描被选择则返回 None
    def get_nmap_output(self):
        # 如果解析的扫描不为 None，则返回对应的 nmap 输出
        if self.parsed_scan is not None:
            return self.parsed_scan.get_nmap_output()
        else:
            return None
    
    # 创建 nmap_output 属性，用于获取当前选择的扫描的输出
    nmap_output = property(get_nmap_output)
    # 创建 parsed_scan 属性，用于获取当前选择的扫描的解析输出
    parsed_scan = property(get_parsed_scan)
# 创建一个名为 DiffWindow 的类，继承自 Gtk.Window
class DiffWindow(Gtk.Window):
    # 初始化方法
    def __init__(self, scans):
        # 调用父类的初始化方法
        Gtk.Window.__init__(self)
        # 设置窗口标题为 "Compare Results"
        self.set_title(_("Compare Results"))
        # 初始化一个变量用于存储旧的进程
        self.old_processes = []
        # 初始化一个变量用于存储定时器的 ID
        self.timer_id = None

        # 创建主垂直布局容器
        self.main_vbox = HIGVBox()
        # 创建一个名为 diff_view 的 DiffView 对象
        self.diff_view = DiffView()
        # 设置 diff_view 的大小请求为宽度不限制，高度为 100
        self.diff_view.set_size_request(-1, 100)
        # 创建一个水平布局容器
        self.hbox_buttons = HIGHBox()
        # 创建一个进度条对象
        self.progress = Gtk.ProgressBar()
        # 创建一个名为 btn_close 的关闭按钮
        self.btn_close = HIGButton(stock=Gtk.STOCK_CLOSE)
        # 创建一个水平布局容器
        self.hbox_selection = HIGHBox()
        # 创建一个名为 scan_chooser_a 的 ScanChooser 对象，用于选择扫描A
        self.scan_chooser_a = ScanChooser(scans, _("A Scan"))
        # 创建一个名为 scan_chooser_b 的 ScanChooser 对象，用于选择扫描B
        self.scan_chooser_b = ScanChooser(scans, _("B Scan"))

        # 调用内部方法，将各个部件添加到窗口中
        self._pack_widgets()
        # 调用内部方法，连接各个部件的信号和槽
        self._connect_widgets()

        # 设置窗口的默认大小，宽度不限制，高度为 500
        self.set_default_size(-1, 500)

        # 记录窗口的初始大小
        self.initial_size = self.get_size()

    # 内部方法，用于将各个部件添加到窗口中
    def _pack_widgets(self):
        # 设置主垂直布局容器的边框宽度为 6
        self.main_vbox.set_border_width(6)

        # 将扫描A选择器和扫描B选择器添加到水平布局容器中
        self.hbox_selection.pack_start(self.scan_chooser_a, True, True, 0)
        self.hbox_selection.pack_start(self.scan_chooser_b, True, True, 0)

        # 将水平布局容器添加到主垂直布局容器中
        self.main_vbox.pack_start(self.hbox_selection, False, True, 0)

        # 创建一个滚动窗口，设置滚动策略为自动，将 diff_view 添加到滚动窗口中
        scroll = Gtk.ScrolledWindow()
        scroll.set_policy(Gtk.PolicyType.AUTOMATIC, Gtk.PolicyType.AUTOMATIC)
        scroll.add(self.diff_view)
        # 将滚动窗口添加到主垂直布局容器中
        self.main_vbox.pack_start(scroll, True, True, 0)

        # 隐藏进度条，并设置不在所有部件中显示
        self.progress.hide()
        self.progress.set_no_show_all(True)
        # 将进度条和关闭按钮添加到水平布局容器中
        self.hbox_buttons.pack_start(self.progress, False, True, 0)
        self.hbox_buttons.pack_end(self.btn_close, False, True, 0)

        # 将水平布局容器添加到主垂直布局容器中
        self.main_vbox._pack_noexpand_nofill(self.hbox_buttons)

        # 将主垂直布局容器添加到窗口中
        self.add(self.main_vbox)
    # 连接窗口的关闭事件到 close 方法
    def _connect_widgets(self):
        self.connect("delete-event", self.close)
        # 连接关闭按钮的点击事件到 close 方法
        self.btn_close.connect("clicked", self.close)
        # 连接扫描选择器 A 的改变事件到 refresh_diff 方法
        self.scan_chooser_a.connect('changed', self.refresh_diff)
        # 连接扫描选择器 B 的改变事件到 refresh_diff 方法
        self.scan_chooser_b.connect('changed', self.refresh_diff)

    # 刷新 diff 输出的方法，当选择器中的扫描发生改变时调用
    def refresh_diff(self, widget):
        """This method is called whenever the diff output might have changed,
        such as when a different scan was selected in one of the choosers."""
        # 记录日志
        log.debug("Refresh diff.")

        # 如果 ndiff 进程存在且未结束，则将其加入旧进程列表，并置为 None
        if (self.ndiff_process is not None and
                self.ndiff_process.poll() is None):
            self.old_processes.append(self.ndiff_process)
            self.ndiff_process = None

        # 获取扫描选择器 A 和 B 中的扫描数据
        scan_a = self.scan_chooser_a.parsed_scan
        scan_b = self.scan_chooser_b.parsed_scan

        # 如果扫描数据为空，则清空 diff 视图
        if scan_a is None or scan_b is None:
            self.diff_view.clear()
        else:
            try:
                # 使用 zenmapCore.Diff.ndiff 方法比较两个扫描数据，获取差异
                self.ndiff_process = zenmapCore.Diff.ndiff(scan_a, scan_b)
            except OSError as e:
                # 如果出现异常，显示错误对话框
                alert = HIGAlertDialog(
                    message_format=_("Error running ndiff"),
                    secondary_text=_(
                        "There was an error running the ndiff program.\n\n"
                        ) + str(e))
                alert.run()
                alert.destroy()
            else:
                # 显示进度条
                self.progress.show()
                # 如果定时器 ID 为空，则添加定时器，定时检查 ndiff 进程
                if self.timer_id is None:
                    self.timer_id = GLib.timeout_add(
                        NDIFF_CHECK_TIMEOUT, self.check_ndiff_process)

    # 关闭窗口的方法
    def close(self, widget=None, extra=None):
        self.destroy()
class DiffView(Gtk.TextView):
    REMOVE_COLOR = "#ffaaaa"  # 定义删除行的颜色
    ADD_COLOR = "#ccffcc"  # 定义新增行的颜色

    """A widget displaying a zenmapCore.Diff.ScanDiff."""
    def __init__(self):
        Gtk.TextView.__init__(self)  # 调用父类的初始化方法
        self.set_editable(False)  # 设置文本视图为不可编辑

        buff = self.get_buffer()  # 获取文本视图的缓冲区
        # 创建文本标记
        buff.create_tag("=", font="Monospace")  # 创建普通行的标记
        buff.create_tag(
            "-", font="Monospace", background=self.REMOVE_COLOR)  # 创建删除行的标记
        buff.create_tag("+", font="Monospace", background=self.ADD_COLOR)  # 创建新增行的标记

    def clear(self):
        self.get_buffer().set_text("")  # 清空文本视图的内容

    def show_diff(self, diff):
        self.clear()  # 清空文本视图的内容
        buff = self.get_buffer()  # 获取文本视图的缓冲区
        for line in diff.splitlines(True):  # 遍历diff的每一行
            if line.startswith("-"):  # 如果是删除行
                tags = ["-"]  # 使用删除行的标记
            elif line.startswith("+"):  # 如果是新增行
                tags = ["+"]  # 使用新增行的标记
            else:  # 如果是普通行
                tags = ["="]  # 使用普通行的标记
            buff.insert_with_tags_by_name(buff.get_end_iter(), line, *tags)  # 在文本视图中插入带有标记的行

if __name__ == "__main__":
    from zenmapCore.NmapParser import NmapParser  # 导入NmapParser类

    parsed1 = NmapParser()  # 创建NmapParser对象
    parsed2 = NmapParser()  # 创建NmapParser对象
    parsed3 = NmapParser()  # 创建NmapParser对象
    parsed4 = NmapParser()  # 创建NmapParser对象

    parsed1.parse_file("test/xml_test1.xml")  # 解析xml文件
    parsed2.parse_file("test/xml_test2.xml")  # 解析xml文件
    parsed3.parse_file("test/xml_test3.xml")  # 解析xml文件
    parsed4.parse_file("test/xml_test4.xml")  # 解析xml文件

    dw = DiffWindow({"Parsed 1": parsed1,
                     "Parsed 2": parsed2,
                     "Parsed 3": parsed3,
                     "Parsed 4": parsed4})  # 创建DiffWindow对象并传入解析后的数据

    dw.show_all()  # 显示DiffWindow对象
    dw.connect("delete-event", lambda x, y: Gtk.main_quit())  # 连接关闭窗口事件，退出Gtk主循环

    Gtk.main()  # 运行Gtk主循环
```