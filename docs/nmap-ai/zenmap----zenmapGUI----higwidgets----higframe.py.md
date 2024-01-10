# `nmap\zenmap\zenmapGUI\higwidgets\higframe.py`

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
# 从gi库中导入Gtk
from gi.repository import Gtk

# 定义HIGFrame类，继承自Gtk.Frame
class HIGFrame(Gtk.Frame):
    """
    Frame without border with bold label.
    """
    # 初始化方法
    def __init__(self, label=None):
        # 调用父类的初始化方法
        Gtk.Frame.__init__(self)

        # 设置边框类型为无
        self.set_shadow_type(Gtk.ShadowType.NONE)
        # 创建一个Gtk.Label对象
        self._flabel = Gtk.Label()
        # 设置标签内容
        self._set_label(label)
        # 设置标签部件
        self.set_label_widget(self._flabel)

    # 设置标签内容的方法
    def _set_label(self, label):
        self._flabel.set_markup("<b>%s</b>" % label)

# 如果作为独立程序运行
if __name__ == "__main__":
    # 创建一个Gtk.Window对象
    w = Gtk.Window()

    # 创建一个HIGFrame对象
    hframe = HIGFrame("Sample HIGFrame")
    # 创建一个对齐对象
    aalign = Gtk.Alignment.new(0, 0, 0, 0)
    # 设置对齐对象的填充
    aalign.set_padding(12, 0, 24, 0)
    # 创建一个垂直方向的Gtk.Box对象
    abox = Gtk.Box.new(Gtk.Orientation.VERTICAL, 0)
    # 将Gtk.Box对象添加到对齐对象中
    aalign.add(abox)
    # 将aalign添加到hframe中
    hframe.add(aalign)
    # 将hframe添加到w中
    w.add(hframe)

    # 循环5次，将带有相应文本的标签添加到abox中
    for i in range(5):
        abox.pack_start(Gtk.Label.new("Sample %d" % i), False, False, 3)

    # 连接窗口关闭事件，触发Gtk.main_quit()函数
    w.connect('destroy', lambda d: Gtk.main_quit())
    # 显示窗口中的所有部件
    w.show_all()

    # 运行GTK主循环
    Gtk.main()
```