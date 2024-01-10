# `nmap\zenmap\zenmapGUI\higwidgets\higbuttons.py`

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
# 从gi.repository中导入Gtk模块
from gi.repository import Gtk

# 定义HIGMixButton类，继承自Gtk.Box
class HIGMixButton(Gtk.Box):
    # 初始化函数，接受标题和库存参数
    def __init__(self, title, stock):
        # 调用父类的初始化函数，设置盒子的方向、是否均匀分布、间距
        Gtk.Box.__init__(self, orientation=Gtk.Orientation.HORIZONTAL,
                         homogeneous=False, spacing=4)
        # 创建一个图像对象
        self.img = Gtk.Image()
        # 从存储中设置图像
        self.img.set_from_stock(stock, Gtk.IconSize.BUTTON)
    
        # 创建一个标签对象
        self.lbl = Gtk.Label.new(title)
    
        # 创建一个新的盒子对象，设置方向和间距
        self.hbox1 = Gtk.Box.new(Gtk.Orientation.HORIZONTAL, 2)
        self.hbox1.set_homogeneous(False)
        # 将图像和标签添加到盒子中
        self.hbox1.pack_start(self.img, False, False, 0)
        self.hbox1.pack_start(self.lbl, False, False, 0)
    
        # 创建一个对齐对象，设置对齐方式和缩进
        self.align = Gtk.Alignment.new(0.5, 0.5, 0, 0)
        # 将对齐对象添加到当前盒子中
        self.pack_start(self.align, True, True, 0)
        # 将之前创建的盒子添加到当前盒子中
        self.pack_start(self.hbox1, True, True, 0)
# 创建一个自定义的按钮类，继承自 Gtk.Button
class HIGButton(Gtk.Button):
    # 初始化方法，接受按钮的标题和图标名称作为参数
    def __init__(self, title="", stock=None):
        # 如果标题和图标名称都存在
        if title and stock:
            # 调用父类的初始化方法
            Gtk.Button.__init__(self)
            # 创建一个 HIGMixButton 对象作为按钮的内容
            content = HIGMixButton(title, stock)
            # 将内容添加到按钮中
            self.add(content)
        # 如果只有标题而没有图标名称
        elif title and not stock:
            # 调用父类的初始化方法，设置按钮的标签为标题
            Gtk.Button.__init__(self, label=title)
        # 如果只有图标名称而没有标题
        elif stock:
            # 调用父类的初始化方法，设置按钮的图标
            Gtk.Button.__init__(self, stock=stock)
        # 如果既没有标题也没有图标名称
        else:
            # 调用父类的初始化方法
            Gtk.Button.__init__(self)

# 创建一个自定义的切换按钮类，继承自 Gtk.ToggleButton
class HIGToggleButton(Gtk.ToggleButton):
    # 初始化方法，接受按钮的标题和图标名称作为参数
    def __init__(self, title="", stock=None):
        # 如果标题和图标名称都存在
        if title and stock:
            # 调用父类的初始化方法
            Gtk.ToggleButton.__init__(self)
            # 创建一个 HIGMixButton 对象作为按钮的内容
            content = HIGMixButton(title, stock)
            # 将内容添加到按钮中
            self.add(content)
        # 如果只有标题而没有图标名称
        elif title and not stock:
            # 调用父类的初始化方法，设置按钮的标签为标题
            Gtk.ToggleButton.__init__(self, label=title)
        # 如果只有图标名称而没有标题
        elif stock:
            # 调用父类的初始化方法，设置按钮的图标
            Gtk.ToggleButton.__init__(self, stock=stock)
            # 设置使用图标名称
            self.set_use_stock(True)
        # 如果既没有标题也没有图标名称
        else:
            # 调用父类的初始化方法
            Gtk.ToggleButton.__init__(self)
```