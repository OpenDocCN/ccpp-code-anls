# `nmap\zenmap\zenmapGUI\higwidgets\higexpanders.py`

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
"""
higwidgets/higexpanders.py

   expanders related classes
"""

# 导入gi库
import gi

# 要求使用特定版本的Gtk
gi.require_version("Gtk", "3.0")
# 从gi库的repository模块中导入Gtk
from gi.repository import Gtk

# 从当前目录下的higboxes模块中导入HIGHBox和hig_box_space_holder
from .higboxes import HIGHBox, hig_box_space_holder

# 定义HIGExpander类，继承自Gtk.Expander
class HIGExpander(Gtk.Expander):
    # 初始化方法
    def __init__(self, label):
        # 调用父类的初始化方法
        Gtk.Expander.__init__(self)

        # 设置使用标记语言
        self.set_use_markup(True)
        # 设置标签
        self.set_label(label)

        # 创建一个HIGHBox对象
        self.hbox = HIGHBox()
        # 设置边框宽度
        self.hbox.set_border_width(5)
        # 将一个占位符添加到HIGHBox中
        self.hbox._pack_noexpand_nofill(hig_box_space_holder())

        # 将HIGHBox添加到当前的Expander中
        self.add(self.hbox)

    # 获取容器方法
    def get_container(self):
        return self.hbox
```