# `nmap\zenmap\zenmapGUI\higwidgets\higdialogs.py`

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
# 导入gi库
import gi
# 要求使用特定版本的Gtk
gi.require_version("Gtk", "3.0")
# 从gi库的repository模块中导入Gtk
from gi.repository import Gtk

# 定义HIGDialog类，继承自Gtk.Dialog
class HIGDialog(Gtk.Dialog):
    """
    HIGFied Dialog
    """
    # 初始化方法
    def __init__(self, title='', parent=None, flags=0, buttons=()):
        # 调用父类的初始化方法
        Gtk.Dialog.__init__(self, title=title, parent=parent, flags=flags)
        # 设置边框宽度
        self.set_border_width(5)
        # 设置vbox的边框宽度和间距
        self.vbox.set_border_width(2)
        self.vbox.set_spacing(6)

        # 如果有按钮参数，则添加按钮
        if buttons:
            self.add_buttons(*buttons)

# 定义HIGAlertDialog类，继承自Gtk.MessageDialog
class HIGAlertDialog(Gtk.MessageDialog):
    """
    HIGfied Alert Dialog.

    Implements the suggestions documented in:
    http://developer.gnome.org/projects/gup/hig/2.0/windows-alert.html
    """
    # 初始化函数，用于创建一个消息对话框
    def __init__(self, parent=None, flags=0, type=Gtk.MessageType.INFO,
                 # HIG mandates that every Alert should have an "affirmative
                 # button that dismisses the alert and performs the action
                 # suggested"
                 buttons=Gtk.ButtonsType.OK,
                 message_format=None,
                 secondary_text=None):
        # 调用父类的初始化函数，设置父窗口、标志、消息类型和按钮类型
        Gtk.MessageDialog.__init__(self, parent=parent, flags=flags,
                                   message_type=type, buttons=buttons)
    
        # 设置对话框不可调整大小
        self.set_resizable(False)
    
        # HIG mandates that Message Dialogs should have no title:
        # "Alert windows have no titles, as the title would usually
        # unnecessarily duplicate the alert's primary text"
        # 设置对话框标题为空
        self.set_title("")
        # 设置消息格式为粗体和较大字号
        self.set_markup(
                "<span weight='bold'size='larger'>%s</span>" % message_format)
        # 如果有次要文本，则格式化设置次要文本
        if secondary_text:
            self.format_secondary_text(secondary_text)
# 如果当前脚本被直接执行
if __name__ == '__main__':

    # 从higlabels模块中导入HIGEntryLabel和HIGDialogLabel类
    from higlabels import HIGEntryLabel, HIGDialogLabel

    # 创建一个HIGDialog对象，设置标题为'HIGDialog'，按钮为'确定'，响应类型为ACCEPT
    d = HIGDialog(title='HIGDialog',
                  buttons=(Gtk.STOCK_OK, Gtk.ResponseType.ACCEPT))
    
    # 创建一个HIGDialogLabel对象，设置标签内容为'A HIGDialogLabel on a HIGDialog'，并显示
    dialog_label = HIGDialogLabel('A HIGDialogLabel on a HIGDialog')
    dialog_label.show()
    
    # 将HIGDialogLabel对象添加到HIGDialog对象的垂直布局中
    d.vbox.pack_start(dialog_label, True, True, 0)

    # 创建一个HIGEntryLabel对象，设置标签内容为'A HIGEntryLabel on a HIGDialog'，并显示
    entry_label = HIGEntryLabel('A HIGEntryLabel on a HIGDialog')
    entry_label.show()
    
    # 将HIGEntryLabel对象添加到HIGDialog对象的垂直布局中
    d.vbox.pack_start(entry_label, True, True, 0)

    # 运行HIGDialog对象，等待用户操作
    d.run()
    
    # 销毁HIGDialog对象
    d.destroy()

    # 创建一个HIGAlertDialog对象，设置消息格式为'You Have and Appointment in 15 minutes'，次要文本为'You shouldn't be late this time. Oh, and there's a huge traffic jam on your way!' 
    d = HIGAlertDialog(message_format="You Have and Appointment in 15 minutes",
                       secondary_text="You shouldn't be late this time. "
                       "Oh, and there's a huge traffic jam on your way!")
    
    # 运行HIGAlertDialog对象，等待用户操作
    d.run()
    
    # 销毁HIGAlertDialog对象
    d.destroy()
```