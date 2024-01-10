# `nmap\zenmap\zenmapGUI\higwidgets\higwindows.py`

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
gi.require_version("Gtk", "3.0")
from gi.repository import Gtk

# 定义模块可导出的类
__all__ = ('HIGWindow', 'HIGMainWindow')

# 创建HIGWindow类，继承自Gtk.Window类
class HIGWindow(Gtk.Window):
    """HIGFied Window"""
    # 初始化方法
    def __init__(self, type=Gtk.WindowType.TOPLEVEL):
        # 调用父类的初始化方法
        Gtk.Window.__init__(self, type=type)
        # 设置窗口边框宽度为5
        self.set_border_width(5)

# 定义HIGMainWindow为Gtk.Window类
# 应用程序的主窗口不应该有边框，所以应该是一个真正的gtk.Window
HIGMainWindow = Gtk.Window
```