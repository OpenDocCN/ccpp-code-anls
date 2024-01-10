# `nmap\zenmap\zenmapGUI\NmapOutputViewer.py`

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
# 导入 gi 模块，用于与 GTK+ 库进行交互
import gi

# 要求使用 GTK+ 3.0 版本
gi.require_version("Gtk", "3.0")
# 从 gi.repository 中导入 Gtk、Gdk、Pango、GLib 模块
from gi.repository import Gtk, Gdk, Pango, GLib

# 导入 gobject 模块
import gobject
# 导入 re 模块，用于正则表达式操作
import re

# 导入 zenmapCore.I18N 模块，用于国际化
import zenmapCore.I18N  # lgtm[py/unused-import]
# 导入 zenmapCore.UmitLogging 模块中的 log 函数
from zenmapCore.UmitLogging import log
# 导入 zenmapCore.UmitConf 模块中的 NmapOutputHighlight 和 NmapOutputProperties 类
from zenmapCore.UmitConf import NmapOutputHighlight
from zenmapGUI.NmapOutputProperties import NmapOutputProperties

# 定义 NmapOutputViewer 类，继承自 Gtk.Box
class NmapOutputViewer(Gtk.Box):
    # 定义 HIGHLIGHT_PROPERTIES 常量，包含需要高亮显示的属性列表
    HIGHLIGHT_PROPERTIES = ["details", "date", "hostname", "ip", "port_list",
            "open_port", "closed_port", "filtered_port"]
    # 初始化函数，设置默认参数refresh和stop
    def __init__(self, refresh=1, stop=1):
        # 创建NmapOutputHighlight对象并赋值给self.nmap_highlight
        self.nmap_highlight = NmapOutputHighlight()
        # 调用父类的初始化函数，设置布局为垂直方向
        Gtk.Box.__init__(self, orientation=Gtk.Orientation.VERTICAL)

        # 创建小部件
        self.__create_widgets()

        # 设置滚动窗口
        self.__set_scrolled_window()

        # 设置文本视图
        self.__set_text_view()

        # 获取文本视图的缓冲区
        buffer = self.text_view.get_buffer()
        # 创建一个标记，用于滚动到显示区域的底部
        self.end_mark = buffer.create_mark(None, buffer.get_end_iter(), False)

        # 设置刷新标志
        self.refreshing = True

        # 将小部件添加到垂直布局中
        self.pack_start(self.scrolled, True, True, 0)

        # 与此显示相关的NmapCommand实例（如果有的话）
        self.command_execution = None
        # 上次从输出流中读取的位置
        self.output_file_pointer = None

    def __create_widgets(self):
        # 创建小部件
        self.scrolled = Gtk.ScrolledWindow()
        self.text_view = Gtk.TextView()

    def __set_scrolled_window(self):
        # 设置滚动窗口
        self.scrolled.set_border_width(5)
        self.scrolled.add(self.text_view)
        self.scrolled.set_policy(Gtk.PolicyType.AUTOMATIC, Gtk.PolicyType.AUTOMATIC)
    # 设置文本视图的换行模式为单词换行
    def __set_text_view(self):
        self.text_view.set_wrap_mode(Gtk.WrapMode.WORD)
        # 设置文本视图为不可编辑状态
        self.text_view.set_editable(False)

        # 创建一个标签用于设置字体样式
        self.tag_font = self.text_view.get_buffer().create_tag(None)
        self.tag_font.set_property("family", "Monospace")
        # 遍历高亮属性列表，为每个属性创建对应的标签
        for property in self.HIGHLIGHT_PROPERTIES:
            settings = self.nmap_highlight.__getattribute__(property)
            tag = self.text_view.get_buffer().create_tag(property)

            # 根据属性设置标签的字体粗细
            if settings[0]:
                tag.set_property("weight", Pango.Weight.HEAVY)
            else:
                tag.set_property("weight", Pango.Weight.NORMAL)

            # 根据属性设置标签的字体样式
            if settings[1]:
                tag.set_property("style", Pango.Style.ITALIC)
            else:
                tag.set_property("style", Pango.Style.NORMAL)

            # 根据属性设置标签的下划线样式
            if settings[2]:
                tag.set_property("underline", Pango.Underline.SINGLE)
            else:
                tag.set_property("underline", Pango.Underline.NONE)

            # 根据属性设置标签的文本颜色和高亮颜色
            text_color = settings[3]
            highlight_color = settings[4]
            tag.set_property("foreground", Gdk.Color(*text_color).to_string())
            tag.set_property("background", Gdk.Color(*highlight_color).to_string())

    # 跳转到指定主机在 nmap 输出结果中的行
    def go_to_host(self, host):
        buff = self.text_view.get_buffer()
        start_iter = buff.get_start_iter()

        # 在文本缓冲区中查找指定主机的行
        found_tuple = start_iter.forward_search(
                "\nNmap scan report for %s\n" % host, Gtk.TextSearchFlags.TEXT_ONLY
                )
        # 如果未找到指定主机的行，则返回
        if found_tuple is None:
            return

        found = found_tuple[0]
        # 如果找到指定主机的行，则滚动文本视图到该行
        if not found.forward_line():
            return
        self.text_view.scroll_to_iter(found, 0, True, 0, 0)
    # 定义一个方法来展示输出属性，接受一个 widget 参数
    def show_output_properties(self, widget):
        # 创建一个 NmapOutputProperties 对象，传入文本视图参数
        nmap_out_prop = NmapOutputProperties(self.text_view)

        # 运行 NmapOutputProperties 对象的 run 方法
        nmap_out_prop.run()

        # 遍历 NmapOutputProperties 对象的属性名称列表
        for prop in nmap_out_prop.property_names:
            # 获取属性对应的 widget
            widget = nmap_out_prop.property_names[prop][8]

            # 创建一个空列表来存储 widget 的属性
            wid_props = []

            # 检查 widget 是否为粗体，是则添加 1，否则添加 0
            if widget.bold:
                wid_props.append(1)
            else:
                wid_props.append(0)

            # 检查 widget 是否为斜体，是则添加 1，否则添加 0
            if widget.italic:
                wid_props.append(1)
            else:
                wid_props.append(0)

            # 检查 widget 是否有下划线，是则添加 1，否则添加 0
            if widget.underline:
                wid_props.append(1)
            else:
                wid_props.append(0)

            # 将 widget 的文本颜色转换为字符串并添加到属性列表中
            wid_props.append("(%s, %s, %s)" % (widget.text_color.red,
                                               widget.text_color.green,
                                               widget.text_color.blue))
            # 将 widget 的高亮颜色转换为字符串并添加到属性列表中
            wid_props.append("(%s, %s, %s)" % (widget.highlight_color.red,
                                               widget.highlight_color.green,
                                               widget.highlight_color.blue))

            # 将 widget 的属性名和属性列表添加到 nmap_highlight 对象中
            self.nmap_highlight.__setattr__(widget.property_name, wid_props)

        # 销毁 NmapOutputProperties 对象
        nmap_out_prop.destroy()
        # 保存更改到 nmap_highlight 对象
        self.nmap_highlight.save_changes()
        # 应用高亮
        self.apply_highlighting()
    # 应用高亮显示到文本视图中的指定范围
    def apply_highlighting(self, start_iter=None, end_iter=None):
        # 获取文本视图的缓冲区
        buf = self.text_view.get_buffer()

        # 如果未指定起始迭代器，则使用缓冲区的起始迭代器
        if start_iter is None:
            start_iter = buf.get_start_iter()
        else:
            # 模式是以行为单位的；从行边界开始
            start_iter.backward_line()
        # 如果未指定结束迭代器，则使用缓冲区的结束迭代器
        if end_iter is None:
            end_iter = buf.get_end_iter()

        # 应用字体标签到指定范围
        buf.apply_tag(self.tag_font, start_iter, end_iter)

        # 如果未启用 NMAP 高亮显示，则直接返回
        if not self.nmap_highlight.enable:
            return

        # 获取指定范围内的文本
        text = buf.get_text(start_iter, end_iter, include_hidden_chars=True)

        # 遍历高亮属性列表
        for property in self.HIGHLIGHT_PROPERTIES:
            # 获取当前属性的设置
            settings = self.nmap_highlight.__getattribute__(property)
            # 在文本中查找匹配的正则表达式
            for m in re.finditer(settings[5], text, re.M):
                # 创建匹配的起始迭代器和结束迭代器
                m_start_iter = start_iter.copy()
                m_start_iter.forward_chars(m.start())
                m_end_iter = start_iter.copy()
                m_end_iter.forward_chars(m.end())
                # 应用标签到匹配的范围
                buf.apply_tag_by_name(property, m_start_iter, m_end_iter)

    # 在输出显示中展示字符串（或unicode）输出
    def show_nmap_output(self, output):
        try:
            # 设置文本视图的缓冲区内容为输出
            self.text_view.get_buffer().set_text(output)
            # 应用高亮显示
            self.apply_highlighting()
        except MemoryError:
            # 如果内存错误，则显示大输出消息
            self.show_large_output_message(self.command_execution)

    # 设置正在运行的命令，其输出显示在该显示器中
    # 当前输出从命令对象中提取
    def set_command_execution(self, command):
        self.command_execution = command
        if command is not None:
            # 如果命令不为空，则设置缓冲区内容为空
            self.text_view.get_buffer().set_text("")
            self.output_file_pointer = 0
        else:
            self.output_file_pointer = None
        # 刷新输出
        self.refresh_output()
    # 定义一个方法，用于在界面上显示大型输出的警告信息
    def show_large_output_message(self, command=None):
        # 获取文本视图的缓冲区
        buf = self.text_view.get_buffer()
        try:
            # 检查是否有命令正在运行
            running = (command is not None and command.scan_state() is True)
        except Exception:
            # 如果出现异常，则将运行状态设置为 False
            running = False
            complete = False
        else:
            # 如果没有异常，则判断是否运行完成
            complete = not running
        # 根据运行状态设置不同的警告信息
        if running:
            # 如果正在运行，则显示警告信息
            buf.set_text("Warning: You have insufficient resources for Zenmap "
                "to be able to display the complete output from Nmap here. \n"
                "Zenmap will continue to run the scan to completion. However,"
                " some features of Zenmap might not work as expected.")
        elif complete:
            # 如果运行完成，则显示警告信息
            buf.set_text("Warning: You have insufficient resources for Zenmap "
                "to be able to display the complete output from Nmap here. \n"
                "The scan has completed. However, some features of Zenmap "
                "might not work as expected.")
        else:
            # 如果运行被停止，则显示警告信息
            buf.set_text("Warning: You have insufficient resources for Zenmap "
                "to be able to display the complete output from Nmap here. \n"
                "The scan has been stopped. Some features of Zenmap might not "
                "work as expected.")
```