# `nmap\zenmap\zenmapGUI\higwidgets\higprogressbars.py`

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
# 导入gi库，确保使用的是Gtk 3.0版本
import gi

gi.require_version("Gtk", "3.0")
from gi.repository import Gtk

# 从higboxes模块中导入HIGHBox类
from .higboxes import HIGHBox

# 定义HIGLabeledProgressBar类，继承自HIGHBox类
class HIGLabeledProgressBar(HIGHBox):
    # 初始化方法
    def __init__(self, label=None):
        # 调用父类的初始化方法
        HIGHBox.__init__(self)
        # 如果有标签，创建HIGEntryLabel对象，并添加到布局中
        if label:
            self.label = HIGEntryLabel(label)
            self.pack_label(self.label)
            # 创建进度条对象，并设置大小
            self.progress_bar = Gtk.ProgressBar()
            self.progress_bar.set_size_request(80, 16)
            # 将进度条添加到布局中
            self.pack_label(self.progress_bar)

        # 定义show方法，显示所有组件
        def show(self):
            HIGHBox.show_all(self)
```