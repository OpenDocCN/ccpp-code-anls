# `nmap\zenmap\zenmapGUI\higwidgets\hignotebooks.py`

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
# 导入 gi 模块
import gi
# 要求使用 Gtk 3.0 版本
gi.require_version("Gtk", "3.0")
# 从 gi.repository 中导入 Gtk 和 GObject
from gi.repository import Gtk, GObject
# 从当前目录下的 higboxes 模块中导入 HIGHBox 类
from .higboxes import HIGHBox
# 从当前目录下的 higbuttons 模块中导入 HIGButton 类
from .higbuttons import HIGButton

# 创建 HIGNotebook 类，继承自 Gtk.Notebook
class HIGNotebook(Gtk.Notebook):
    # 初始化方法
    def __init__(self):
        # 调用父类 Gtk.Notebook 的初始化方法
        Gtk.Notebook.__init__(self)
        # 启用弹出菜单
        self.popup_enable()

# 创建 HIGClosableTabLabel 类，继承自 HIGHBox
class HIGClosableTabLabel(HIGHBox):
    # 定义信号 close-clicked
    __gsignals__ = {
            'close-clicked': (GObject.SignalFlags.RUN_LAST, GObject.TYPE_NONE, ())
            }

    # 初始化方法
    def __init__(self, label_text=""):
        # 调用 GObject 的初始化方法
        GObject.GObject.__init__(self)
        # 设置标签文本
        self.label_text = label_text
        # 创建小部件
        self.__create_widgets()

        # 设置属性映射
        #self.property_map = {"label_text" : self.label.get_label}
    # 创建小部件
    def __create_widgets(self):
        # 创建一个标签并设置文本
        self.label = Gtk.Label.new(self.label_text)
        # 创建一个关闭按钮的图标
        self.close_image = Gtk.Image()
        # 从存储中设置关闭按钮的图标
        self.close_image.set_from_stock(Gtk.STOCK_CLOSE, Gtk.IconSize.BUTTON)
        # 创建一个关闭按钮
        self.close_button = HIGButton()
        # 设置关闭按钮的大小
        self.close_button.set_size_request(20, 20)
        # 设置关闭按钮的样式
        self.close_button.set_relief(Gtk.ReliefStyle.NONE)
        # 设置关闭按钮点击时不获取焦点
        self.close_button.set_focus_on_click(False)
        # 将关闭按钮添加关闭图标
        self.close_button.add(self.close_image)

        # 连接关闭按钮的点击事件到相应的处理函数
        self.close_button.connect('clicked', self.__close_button_clicked)

        # 将标签和关闭按钮添加到容器中
        for w in (self.label, self.close_button):
            self.pack_start(w, False, False, 0)

        # 显示所有小部件
        self.show_all()

    # 关闭按钮点击事件的处理函数
    def __close_button_clicked(self, data):
        # 发射关闭按钮点击的信号
        self.emit('close-clicked')

    # 获取标签的文本
    def get_text(self):
        return self.label.get_text()

    # 设置标签的文本
    def set_text(self, text):
        self.label.set_text(text)

    # 获取标签的标签
    def get_label(self):
        return self.label.get_label()

    # 设置标签的标签
    def set_label(self, label):
        self.label.set_text(label)
# 注册 HIGClosableTabLabel 类型到 GObject 类型系统中
GObject.type_register(HIGClosableTabLabel)

# 将 HIGClosableTabLabel 类型赋值给 HIGAnimatedTabLabel，即 HIGAnimatedTabLabel 类型和 HIGClosableTabLabel 类型相同
HIGAnimatedTabLabel = HIGClosableTabLabel
```