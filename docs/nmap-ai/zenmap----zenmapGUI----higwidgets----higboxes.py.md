# `nmap\zenmap\zenmapGUI\higwidgets\higboxes.py`

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
higwidgets/higboxes.py

   box related classes
"""

# 导入 gi 模块
import gi

# 要求使用指定版本的 Gtk
gi.require_version("Gtk", "3.0")
# 从 gi.repository 中导入 Gtk 模块
from gi.repository import Gtk

# 定义 HIGBox 类，继承自 Gtk.Box
class HIGBox(Gtk.Box):
    # 定义 _pack_noexpand_nofill 方法，用于向容器中添加不可扩展、不填充的部件
    def _pack_noexpand_nofill(self, widget):
        self.pack_start(widget, False, False, 0)

    # 定义 _pack_expand_fill 方法，用于向容器中添加可扩展、填充的部件
    def _pack_expand_fill(self, widget):
        self.pack_start(widget, True, True, 0)

    # 重写 add 方法，使默认的 packing 参数与之前相同
    def add(self, widget):
        self.pack_start(widget, True, True, 0)

# 定义 HIGHBox 类，继承自 HIGBox
class HIGHBox(HIGBox):
    # 初始化方法，设置方向为水平，是否均匀分布为非均匀，间距为12
    def __init__(self, homogeneous=False, spacing=12):
        Gtk.Box.__init__(self, orientation=Gtk.Orientation.HORIZONTAL,
                         homogeneous=homogeneous, spacing=spacing)
    # 将 HIGBox 类的 _pack_noexpand_nofill 属性赋值给 pack_section_label
    pack_section_label = HIGBox._pack_noexpand_nofill
    # 将 HIGBox 类的 _pack_noexpand_nofill 属性赋值给 pack_label
    pack_label = HIGBox._pack_noexpand_nofill
    # 将 HIGBox 类的 _pack_expand_fill 属性赋值给 pack_entry
    pack_entry = HIGBox._pack_expand_fill
# 创建一个垂直布局的 HIGVBox 类，继承自 HIGBox 类
class HIGVBox(HIGBox):
    # 初始化方法，设置是否均匀分布和间距
    def __init__(self, homogeneous=False, spacing=12):
        # 调用父类的初始化方法，设置布局为垂直，传入是否均匀分布和间距
        Gtk.Box.__init__(self, orientation=Gtk.Orientation.VERTICAL,
                         homogeneous=homogeneous, spacing=spacing)

    # 将一个小部件打包成一行，使其在垂直方向上不扩展
    pack_line = HIGBox._pack_noexpand_nofill


# 创建一个 HIGSpacer 类，继承自 HIGHBox 类
class HIGSpacer(HIGHBox):
    # 初始化方法，接受一个小部件作为参数
    def __init__(self, widget=None):
        # 调用父类的初始化方法
        HIGHBox.__init__(self)
        # 设置间距为6
        self.set_spacing(6)

        # 在布局中添加一个空白标签
        self._pack_noexpand_nofill(hig_box_space_holder())

        # 如果有传入小部件，则将其扩展并填充布局，并将其设置为子部件
        if widget:
            self._pack_expand_fill(widget)
            self.child = widget

    # 获取子部件的方法
    def get_child(self):
        return self.child


# 创建一个函数 hig_box_space_holder，返回一个带有空格的标签
def hig_box_space_holder():
    return Gtk.Label.new("    ")
```