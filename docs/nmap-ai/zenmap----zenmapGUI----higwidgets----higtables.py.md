# `nmap\zenmap\zenmapGUI\higwidgets\higtables.py`

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
"""
higwidgets/higlogindialog.py

   a basic login/authentication dialog
"""

# 导入 gi 模块
import gi

# 要求使用 Gtk 3.0 版本
gi.require_version("Gtk", "3.0")
# 从 gi.repository 中导入 Gtk 模块
from gi.repository import Gtk

# 从 higlabels 模块中导入所有内容
#from higlabels import *
# 从 higentries 模块中导入所有内容
#from higentries import *


# 创建 HIGTable 类，继承自 Gtk.Table
class HIGTable(Gtk.Table):
    """
    A HIGFied table
    """

    # 初始化方法
    def __init__(self, rows=1, columns=1, homogeneous=False):
        # 调用父类的初始化方法
        Gtk.Table.__init__(self, n_rows=rows, n_columns=columns, homogeneous=homogeneous)
        # 设置行间距
        self.set_row_spacings(6)
        # 设置列间距
        self.set_col_spacings(12)

        # 记录行数和列数
        self.rows = rows
        self.columns = columns
    # 在表格布局中添加带有标签的小部件，设置填充选项为填充整个空间
    def attach_label(self, widget, x0, x, y0, y):
        self.attach(widget, x0, x, y0, y, xoptions=Gtk.AttachOptions.FILL)
    
    # 在表格布局中添加输入框小部件，设置填充选项为填充整个空间和扩展以填充额外空间
    def attach_entry(self, widget, x0, x, y0, y):
        self.attach(widget, x0, x, y0, y, xoptions=Gtk.AttachOptions.FILL | Gtk.AttachOptions.EXPAND)
```