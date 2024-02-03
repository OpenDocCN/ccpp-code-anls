# `nmap\zenmap\zenmapGUI\NmapOutputProperties.py`

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
# 从gi.repository中导入Gtk和Gdk模块
from gi.repository import Gtk, Gdk

# 导入zenmapCore.I18N模块，用于国际化
import zenmapCore.I18N  # lgtm[py/unused-import]
# 从zenmapCore.UmitConf模块中导入NmapOutputHighlight
from zenmapCore.UmitConf import NmapOutputHighlight

# 从zenmapGUI.higwidgets.higdialogs模块中导入HIGDialog
from zenmapGUI.higwidgets.higdialogs import HIGDialog
# 从zenmapGUI.higwidgets.hignotebooks模块中导入HIGNotebook
from zenmapGUI.higwidgets.hignotebooks import HIGNotebook
# 从zenmapGUI.higwidgets.higboxes模块中导入HIGVBox
from zenmapGUI.higwidgets.higboxes import HIGVBox
# 从zenmapGUI.higwidgets.higtables模块中导入HIGTable
from zenmapGUI.higwidgets.higtables import HIGTable
# 从zenmapGUI.higwidgets.higlabels模块中导入HIGEntryLabel
from zenmapGUI.higwidgets.higlabels import HIGEntryLabel
# 从zenmapGUI.higwidgets.higbuttons模块中导入HIGButton和HIGToggleButton
from zenmapGUI.higwidgets.higbuttons import HIGButton, HIGToggleButton

# 定义NmapOutputProperties类，继承自HIGDialog类
class NmapOutputProperties(HIGDialog):
    # 初始化函数，接受一个 nmap_output_view 参数
    def __init__(self, nmap_output_view):
        # 调用父类 HIGDialog 的初始化函数，设置对话框标题为 "Nmap Output Properties"，并添加关闭按钮
        HIGDialog.__init__(self, _("Nmap Output Properties"), buttons=(Gtk.STOCK_CLOSE, Gtk.ResponseType.CLOSE))
    
        # 创建一个 NmapOutputHighlight 对象，用于高亮显示 Nmap 输出
        self.nmap_highlight = NmapOutputHighlight()
    
        # 创建窗口部件
        self.__create_widgets()
        # 将窗口部件添加到窗口中
        self.__pack_widgets()
        # 在窗口中显示高亮标签页
        self.highlight_tab()
    
        # 显示窗口中的所有部件
        self.vbox.show_all()
    
    # 创建窗口部件的私有函数
    def __create_widgets(self):
        # 创建一个 HIGNotebook 对象，用于显示属性信息
        self.properties_notebook = HIGNotebook()
    
    # 将窗口部件添加到窗口中的私有函数
    def __pack_widgets(self):
        # 将属性信息页添加到窗口中
        self.vbox.pack_start(self.properties_notebook, True, True, 0)
class HighlightProperty(object):
    # 定义 HighlightProperty 类
    def __init__(self, property_name, property):
        # 初始化方法，创建小部件
        self.__create_widgets()

        # 设置属性名称
        self.property_name = property_name

        # 设置属性标签
        self.property_label = property[0].capitalize()
        # 设置示例
        self.example = property[1]
        # 设置是否加粗
        self.bold = property[2]
        # 设置是否斜体
        self.italic = property[3]
        # 设置是否下划线
        self.underline = property[4]

        # 设置文本颜色
        self.text_color = property[5]
        # 设置高亮颜色
        self.highlight_color = property[6]

        # 连接按钮信号
        self.__connect_buttons()

    def __create_widgets(self):
        # 创建小部件
        self.property_name_label = HIGEntryLabel("")
        self.example_label = HIGEntryLabel("")
        self.bold_tg_button = HIGToggleButton("", Gtk.STOCK_BOLD)
        self.italic_tg_button = HIGToggleButton("", Gtk.STOCK_ITALIC)
        self.underline_tg_button = HIGToggleButton("", Gtk.STOCK_UNDERLINE)
        self.text_color_button = HIGButton(
                _("Text"), stock=Gtk.STOCK_SELECT_COLOR)
        self.highlight_color_button = HIGButton(
                _("Highlight"), stock=Gtk.STOCK_SELECT_COLOR)

    def __connect_buttons(self):
        # 连接按钮信号
        self.bold_tg_button.connect("toggled", self.update_example)
        self.italic_tg_button.connect("toggled", self.update_example)
        self.underline_tg_button.connect("toggled", self.update_example)

        self.text_color_button.connect("clicked", self.text_color_dialog)
        self.highlight_color_button.connect(
                "clicked", self.highlight_color_dialog)

    ####################################
    # Text color dialog
    # 文本颜色对话框
    # 创建文本颜色选择对话框
    def text_color_dialog(self, widget):
        color_dialog = Gtk.ColorSelectionDialog.new(
                "%s %s" % (self.label, _("text color")))
        # 设置文本颜色选择对话框的当前颜色
        color_dialog.get_color_selection().set_current_color(self.text_color)

        # 连接确定按钮的点击事件到文本颜色对话框的确定方法
        color_dialog.props.ok_button.connect(
                "clicked", self.text_color_dialog_ok, color_dialog)
        # 连接取消按钮的点击事件到文本颜色对话框的取消方法
        color_dialog.props.cancel_button.connect(
                "clicked", self.text_color_dialog_cancel, color_dialog)
        # 连接关闭事件到文本颜色对话框的关闭方法
        color_dialog.connect(
                "delete-event", self.text_color_dialog_close, color_dialog)

        # 运行文本颜色选择对话框
        color_dialog.run()

    # 文本颜色对话框的确定方法
    def text_color_dialog_ok(self, widget, color_dialog):
        # 获取选择的文本颜色并关闭对话框
        self.text_color = color_dialog.get_color_selection().get_current_color()
        color_dialog.destroy()
        # 更新示例
        self.update_example()

    # 文本颜色对话框的取消方法
    def text_color_dialog_cancel(self, widget, color_dialog):
        # 关闭文本颜色对话框
        color_dialog.destroy()

    # 文本颜色对话框的关闭方法
    def text_color_dialog_close(self, widget, extra, color_dialog):
        # 关闭文本颜色对话框
        color_dialog.destroy()

    #########################################
    # 创建高亮颜色选择对话框
    def highlight_color_dialog(self, widget):
        color_dialog = Gtk.ColorSelectionDialog.new(
                "%s %s" % (self.property_name, _("highlight color")))
        # 设置高亮颜色选择对话框的当前颜色
        color_dialog.get_color_selection().set_current_color(self.highlight_color)

        # 连接确定按钮的点击事件到高亮颜色对话框的确定方法
        color_dialog.props.ok_button.connect(
                "clicked", self.highlight_color_dialog_ok, color_dialog)
        # 连接取消按钮的点击事件到高亮颜色对话框的取消方法
        color_dialog.props.cancel_button.connect(
                "clicked", self.highlight_color_dialog_cancel,
                color_dialog)
        # 连接关闭事件到高亮颜色对话框的关闭方法
        color_dialog.connect(
                "delete-event", self.highlight_color_dialog_close,
                color_dialog)

        # 运行高亮颜色选择对话框
        color_dialog.run()

    # 高亮颜色对话框的确定方法
    def highlight_color_dialog_ok(self, widget, color_dialog):
        # 获取选择的高亮颜色并关闭对话框
        self.highlight_color = color_dialog.get_color_selection().get_current_color()
        color_dialog.destroy()
        # 更新示例
        self.update_example()
    # 取消高亮颜色对话框
    def highlight_color_dialog_cancel(self, widget, color_dialog):
        color_dialog.destroy()
    
    # 关闭高亮颜色对话框
    def highlight_color_dialog_close(self, widget, extra, color_dialog):
        color_dialog.destroy()
    
    # 更新示例标签内容
    def update_example(self, widget=None):
        # 获取示例标签的文本内容
        label = self.example_label.get_text()
    
        # 检查是否需要加粗
        if self.bold_tg_button.get_active():
            label = "<b>" + label + "</b>"
    
        # 检查是否需要斜体
        if self.italic_tg_button.get_active():
            label = "<i>" + label + "</i>"
    
        # 检查是否需要下划线
        if self.underline_tg_button.get_active():
            label = "<u>" + label + "</u>"
    
        # 修改示例标签的前景色和背景色，并设置标记文本
        self.example_label.modify_fg(Gtk.StateType.NORMAL, self.text_color)
        self.example_label.modify_bg(Gtk.StateType.NORMAL, self.highlight_color)
        self.example_label.set_markup(label)
    
    # 显示加粗效果
    def show_bold(self, widget):
        self.example_label.set_markup("<>")
    
    # 获取示例标签的内容
    def get_example(self):
        return self.example_label.get_text()
    
    # 设置示例标签的内容
    def set_example(self, example):
        self.example_label.set_text(example)
    
    # 获取加粗状态
    def get_bold(self):
        if self.bold_tg_button.get_active():
            return 1
        return 0
    
    # 设置加粗状态
    def set_bold(self, bold):
        self.bold_tg_button.set_active(bold)
    
    # 获取斜体状态
    def get_italic(self):
        if self.italic_tg_button.get_active():
            return 1
        return 0
    
    # 设置斜体状态
    def set_italic(self, italic):
        self.italic_tg_button.set_active(italic)
    
    # 获取下划线状态
    def get_underline(self):
        if self.underline_tg_button.get_active():
            return 1
        return 0
    
    # 设置下划线状态
    def set_underline(self, underline):
        self.underline_tg_button.set_active(underline)
    
    # 获取标签内容
    def get_label(self):
        return self.property_name_label.get_text()
    
    # 设置标签内容
    def set_label(self, label):
        self.property_name_label.set_text(label)
    
    # 创建属性标签
    label = property(get_label, set_label)
    example = property(get_example, set_example)
    bold = property(get_bold, set_bold)
    # 创建一个名为italic的属性，其getter方法为get_italic，setter方法为set_italic
    italic = property(get_italic, set_italic)
    # 创建一个名为underline的属性，其getter方法为get_underline，setter方法为set_underline
    underline = property(get_underline, set_underline)
# 如果当前模块被直接执行，而不是被导入到其他模块中
if __name__ == "__main__":
    # 创建一个NmapOutputProperties对象，传入None作为参数
    n = NmapOutputProperties(None)
    # 运行NmapOutputProperties对象的run方法
    n.run()
    # 运行GTK的主循环，开始处理事件
    Gtk.main()
```