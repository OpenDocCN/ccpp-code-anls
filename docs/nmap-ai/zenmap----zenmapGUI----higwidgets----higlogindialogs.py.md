# `nmap\zenmap\zenmapGUI\higwidgets\higlogindialogs.py`

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
# 导入gi库，确保使用的是Gtk 3.0版本
import gi

# 从gi库的Gtk模块中导入Gtk
gi.require_version("Gtk", "3.0")
from gi.repository import Gtk

# 从当前目录下的higdialogs模块中导入HIGDialog类
# 从当前目录下的higlabels模块中导入HIGEntryLabel类
# 从当前目录下的higtables模块中导入HIGTable类
# 从当前目录下的higentries模块中导入HIGTextEntry类和HIGPasswordEntry类
from .higdialogs import HIGDialog
from .higlabels import HIGEntryLabel
from .higtables import HIGTable
from .higentries import HIGTextEntry, HIGPasswordEntry

# 定义HIGLoginDialog类，继承自HIGDialog类
class HIGLoginDialog(HIGDialog):
    """
    A dialog that asks for basic login information (username / password)
    """
    # 初始化函数，设置对话框的标题和按钮
    def __init__(self, title='Login',
                 buttons=(Gtk.STOCK_CANCEL, Gtk.ResponseType.CANCEL,
                          Gtk.STOCK_OK, Gtk.ResponseType.ACCEPT)):
        # 调用父类的初始化函数，设置对话框的标题和按钮
        HIGDialog.__init__(self, title, buttons=buttons)

        # 创建用户名标签和输入框
        self.username_label = HIGEntryLabel("Username:")
        self.username_entry = HIGTextEntry()
        # 创建密码标签和输入框
        self.password_label = HIGEntryLabel("Password:")
        self.password_entry = HIGPasswordEntry()

        # 创建一个2x2的表格布局
        self.username_password_table = HIGTable(2, 2)
        # 将用户名标签和输入框添加到表格布局中
        self.username_password_table.attach_label(self.username_label,
                                                  0, 1, 0, 1)
        self.username_password_table.attach_entry(self.username_entry,
                                                  1, 2, 0, 1)
        # 将密码标签和输入框添加到表格布局中
        self.username_password_table.attach_label(self.password_label,
                                                  0, 1, 1, 2)
        self.username_password_table.attach_entry(self.password_entry,
                                                  1, 2, 1, 2)

        # 将表格布局添加到垂直布局中
        self.vbox.pack_start(self.username_password_table, False, False, 0)
        # 设置默认的响应类型为接受
        self.set_default_response(Gtk.ResponseType.ACCEPT)

    # 运行对话框
    def run(self):
        # 显示所有部件
        self.show_all()
        # 调用父类的运行函数
        return HIGDialog.run(self)
# 如果当前模块是主程序，则执行以下代码
if __name__ == '__main__':

    # 从 gtkutils 模块中导入 gtk_constant_name 函数
    from gtkutils import gtk_constant_name

    # 创建 HIGLoginDialog 对象
    d = HIGLoginDialog()
    # 运行对话框并获取返回值
    response_value = d.run()
    # 打印返回值对应的 GTK 常量名
    print(gtk_constant_name('response', response_value))
    # 销毁对话框
    d.destroy()
```