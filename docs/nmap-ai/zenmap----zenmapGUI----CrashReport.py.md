# `nmap\zenmap\zenmapGUI\CrashReport.py`

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
from gi.repository import Gtk, Gdk

# 导入sys和traceback模块
import sys
import traceback

# 从zenmapGUI.higwidgets.higdialogs模块中导入HIGDialog类
# 从zenmapGUI.higwidgets.higboxes模块中导入HIGHBox类
from zenmapGUI.higwidgets.higdialogs import HIGDialog
from zenmapGUI.higwidgets.higboxes import HIGHBox

# 从zenmapCore.Name模块中导入APP_DISPLAY_NAME变量
# 从zenmapCore.Version模块中导入VERSION变量
# 导入zenmapCore.I18N模块（虽然未使用）
import zenmapCore.Name
import zenmapCore.Version
import zenmapCore.I18N  # lgtm[py/unused-import]

# 防止加载PyXML
# 将xml.__path__中不包含"_xmlplus"的路径重新赋值给xml.__path__
import xml
xml.__path__ = [x for x in xml.__path__ if "_xmlplus" not in x]

# 用于在标记的标签中转义文本
from xml.sax.saxutils import escape

# 定义CrashReport类，继承自HIGDialog类
class CrashReport(HIGDialog):
    # 初始化方法，接受异常类型、值和回溯信息
    def __init__(self, type, value, tb):
        # 调用父类的初始化方法
        HIGDialog.__init__(self)
        # 调用 Gtk 窗口的初始化方法
        Gtk.Window.__init__(self)
        # 设置窗口标题为 'Crash Report'
        self.set_title(_('Crash Report'))
        # 设置窗口位置为居中
        self.set_position(Gtk.WindowPosition.CENTER_ALWAYS)

        # 创建窗口中的各个部件
        self._create_widgets()
        # 将各个部件添加到窗口中
        self._pack_widgets()
        # 连接各个部件的信号和槽
        self._connect_widgets()

        # 将异常的回溯信息格式化为字符串
        trace = "".join(traceback.format_exception(type, value, tb))
        # 拼接版本信息和回溯信息，设置为文本框的内容
        text = "Version: " + VERSION + "\n" + trace
        self.description_text.get_buffer().set_text(text)

    # 创建窗口中的各个部件
    def _create_widgets(self):
        # 创建水平按钮盒子
        self.button_box = Gtk.ButtonBox.new(Gtk.Orientation.HORIZONTAL)
        # 创建水平按钮盒子
        self.button_box_ok = Gtk.ButtonBox.new(Gtk.Orientation.HORIZONTAL)

        # 创建可滚动的文本框
        self.description_scrolled = Gtk.ScrolledWindow()
        # 创建文本视图
        self.description_text = Gtk.TextView()
        # 设置文本视图不可编辑
        self.description_text.set_editable(False)

        # 创建标签，显示错误信息
        self.bug_text = Gtk.Label()
        self.bug_text.set_markup(_('An unexpected error has crashed '
            '%(app_name)s. Please copy the stack trace below and send it to '
            'the <a href="mailto:dev@nmap.org">dev@nmap.org</a> mailing list. '
            '(<a href="http://seclists.org/nmap-dev/">More about the list.</a>'
            ') The developers will see your report and try to fix the problem.'
            ) % {"app_name": escape(APP_DISPLAY_NAME)})
        # 创建框架
        self.email_frame = Gtk.Frame()
        # 创建标签，显示邮件相关信息
        self.email_label = Gtk.Label()
        self.email_label.set_markup(_('<b>Copy and email to '
            '<a href="mailto:dev@nmap.org">dev@nmap.org</a>:</b>'))
        # 创建复制按钮
        self.btn_copy = Gtk.Button.new_from_stock(Gtk.STOCK_COPY)
        # 创建确定按钮
        self.btn_ok = Gtk.Button.new_from_stock(Gtk.STOCK_OK)

        # 创建水平盒子
        self.hbox = HIGHBox()
    # 将描述文本添加到可滚动窗口中
        self.description_scrolled.add(self.description_text)
    # 设置描述文本的滚动策略为自动
        self.description_scrolled.set_policy(
                Gtk.PolicyType.AUTOMATIC, Gtk.PolicyType.AUTOMATIC)
    # 设置描述文本滚动窗口的大小
        self.description_scrolled.set_size_request(400, 150)
    # 设置描述文本的换行模式为单词换行
        self.description_text.set_wrap_mode(Gtk.WrapMode.WORD)

    # 设置 bug 文本的最大宽度字符数
        self.bug_text.set_max_width_chars(60)
    # 设置 bug 文本为自动换行
        self.bug_text.set_line_wrap(True)
    # 设置 email 标签为自动换行
        self.email_label.set_line_wrap(True)

    # 设置 email 框架的标签部件为 email 标签
        self.email_frame.set_label_widget(self.email_label)
    # 设置 email 框架的阴影类型为无
        self.email_frame.set_shadow_type(Gtk.ShadowType.NONE)

    # 设置水平盒子的边框宽度
        self.hbox.set_border_width(6)
    # 设置垂直盒子的边框宽度
        self.vbox.set_border_width(6)

    # 将 bug 文本添加到水平盒子中并设置为扩展和填充
        self.hbox._pack_expand_fill(self.bug_text)

    # 设置按钮盒子的布局样式为开始
        self.button_box.set_layout(Gtk.ButtonBoxStyle.START)
    # 设置确定按钮盒子的布局样式为结束
        self.button_box_ok.set_layout(Gtk.ButtonBoxStyle.END)

    # 在按钮盒子中添加复制按钮，并设置为可扩展和填充
        self.button_box.pack_start(self.btn_copy, True, True, 0)
    # 在确定按钮盒子中添加确定按钮，并设置为可扩展和填充
        self.button_box_ok.pack_start(self.btn_ok, True, True, 0)

    # 在垂直盒子中添加水平盒子，并设置为可扩展和填充
        self.vbox.pack_start(self.hbox, True, True, 0)
    # 在垂直盒子中添加 email 框架，并设置为可扩展和填充
        self.vbox.pack_start(self.email_frame, True, True, 0)
    # 在垂直盒子中添加描述文本滚动窗口，并设置为可扩展和填充
        self.vbox.pack_start(self.description_scrolled, True, True, 0)
    # 在垂直盒子中添加按钮盒子，并设置为可扩展和填充
        self.vbox.pack_start(self.button_box, True, True, 0)
    # 在动作区域中添加确定按钮盒子，并设置为可扩展和填充
        self.action_area.pack_start(self.button_box_ok, True, True, 0)

    # 连接确定按钮的点击事件到关闭方法
        self.btn_ok.connect("clicked", self.close)
    # 连接复制按钮的点击事件到复制方法
        self.btn_copy.connect("clicked", self.copy)
    # 连接窗口的删除事件到关闭方法
        self.connect("delete-event", self.close)

    # 获取描述文本的缓冲区，并返回其中的文本
        buff = self.description_text.get_buffer()
        return buff.get_text(buff.get_start_iter(), buff.get_end_iter(), include_hidden_chars=True)

    # 复制描述文本到剪贴板
        clipboard = Gtk.Clipboard.get(Gdk.SELECTION_CLIPBOARD)
        clipboard.set_text(self.get_description(), -1)
        clipboard.store()

    # 关闭窗口并退出程序
        self.destroy()
        Gtk.main_quit()
        sys.exit(0)
# 如果当前模块被直接执行，则执行以下代码
if __name__ == "__main__":
    # 创建一个CrashReport对象，传入None作为参数
    c = CrashReport(None, None, None)
    # 显示CrashReport对象的所有内容
    c.show_all()
    # 连接CrashReport对象的"delete-event"信号，当触发时退出Gtk主循环
    c.connect("delete-event", lambda x, y: Gtk.main_quit())
    # 进入Gtk主循环，等待事件处理
    Gtk.main()
```