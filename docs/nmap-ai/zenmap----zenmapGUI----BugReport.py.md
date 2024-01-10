# `nmap\zenmap\zenmapGUI\BugReport.py`

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
from gi.repository import Gtk

# 从zenmapGUI.higwidgets.higboxes模块中导入HIGVBox
from zenmapGUI.higwidgets.higboxes import HIGVBox

# 从zenmapCore.Name模块中导入APP_DISPLAY_NAME, NMAP_DISPLAY_NAME, NMAP_WEB_SITE
from zenmapCore.Name import APP_DISPLAY_NAME, NMAP_DISPLAY_NAME, NMAP_WEB_SITE

# 导入zenmapCore.I18N模块，用于国际化，但未被使用
import zenmapCore.I18N

# 防止加载PyXML
import xml
# 从xml.sax.saxutils模块中导入escape，用于在标记的标签中转义文本
xml.__path__ = [x for x in xml.__path__ if "_xmlplus" not in x]
from xml.sax.saxutils import escape

# 定义BugReport类，继承自Gtk.Window
class BugReport(Gtk.Window, object):
    def __init__(self):
        # 调用父类的初始化方法
        Gtk.Window.__init__(self)
        # 设置窗口标题为'How to Report a Bug'
        self.set_title(_('How to Report a Bug'))
        # 设置窗口位置为居中
        self.set_position(Gtk.WindowPosition.CENTER_ALWAYS)
        # 禁止调整窗口大小
        self.set_resizable(False)

        # 创建窗口部件
        self._create_widgets()
        # 将窗口部件打包
        self._pack_widgets()
        # 连接窗口部件
        self._connect_widgets()
    # 创建窗口部件
    def _create_widgets(self):
        # 创建垂直布局容器
        self.vbox = HIGVBox()
        # 创建水平按钮容器
        self.button_box = Gtk.ButtonBox.new(Gtk.Orientation.HORIZONTAL)
    
        # 创建文本标签
        self.text = Gtk.Label()
    
        # 创建“确定”按钮
        self.btn_ok = Gtk.Button.new_from_stock(Gtk.STOCK_OK)
    
    # 将部件添加到布局中
    def _pack_widgets(self):
        # 设置垂直布局容器的边框宽度
        self.vbox.set_border_width(6)
    
        # 设置文本标签自动换行
        self.text.set_line_wrap(True)
        # 设置文本标签最大宽度字符数
        self.text.set_max_width_chars(50)
        # 设置文本标签的标记文本
        self.text.set_markup(_("""\
# 定义一个类，用于报告 bug
class BugReport(Gtk.Dialog):

    # 初始化方法，设置对话框的标题和按钮
    def __init__(self):
        Gtk.Dialog.__init__(self, title="How to report a bug")

        # 创建一个文本框，显示 bug 报告的内容
        self.text = Gtk.Label(
            # 使用字符串格式化，插入 app 和 nmap 的显示名称和网站
            ("""Like their author, %(nmap)s and %(app)s aren't perfect. But you can help \
make it better by sending bug reports or even writing patches. If \
%(nmap)s doesn't behave the way you expect, first upgrade to the latest \
version available from <b>%(nmap_web)s</b>. If the problem persists, do \
some research to determine whether it has already been discovered and \
addressed. Try Googling the error message or browsing the nmap-dev \
archives at http://seclists.org/. Read the full manual page as well. If \
nothing comes of this, mail a bug report to \
<b>&lt;dev@nmap.org&gt;</b>. Please include everything you have \
learned about the problem, as well as what version of Nmap you are \
running and what operating system version it is running on. Problem \
reports and %(nmap)s usage questions sent to dev@nmap.org are \
far more likely to be answered than those sent to Fyodor directly.

Code patches to fix bugs are even better than bug reports. Basic \
instructions for creating patch files with your changes are available at \
https://nmap.org/data/HACKING. Patches may be sent to nmap-dev \
(recommended) or to Fyodor directly.
""") % {
            "app": escape(APP_DISPLAY_NAME),
            "nmap": escape(NMAP_DISPLAY_NAME),
            "nmap_web": escape(NMAP_WEB_SITE)
            })
        # 将文本框添加到对话框中
        self.vbox.add(self.text)

        # 设置按钮框的布局样式
        self.button_box.set_layout(Gtk.ButtonBoxStyle.END)
        # 将确认按钮添加到按钮框中
        self.button_box.pack_start(self.btn_ok, True, True, 0)

        # 将按钮框添加到对话框中
        self.vbox._pack_noexpand_nofill(self.button_box)
        self.add(self.vbox)

    # 连接各个部件的信号
    def _connect_widgets(self):
        # 点击确认按钮时关闭对话框
        self.btn_ok.connect("clicked", self.close)
        # 关闭对话框的事件
        self.connect("delete-event", self.close)

    # 关闭对话框的方法
    def close(self, widget=None, event=None):
        self.destroy()

# 如果是主程序入口
if __name__ == "__main__":
    # 创建 BugReport 对象
    w = BugReport()
    # 显示对话框
    w.show_all()
    # 关联关闭事件，关闭主循环
    w.connect("delete-event", lambda x, y: Gtk.main_quit())

    # 进入 GTK 主循环
    Gtk.main()
```