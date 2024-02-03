# `nmap\zenmap\zenmapGUI\higwidgets\higlabels.py`

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
# 从 gi.repository 中导入 Gtk 和 Gdk 模块
from gi.repository import Gtk, Gdk

# 定义 HIGSectionLabel 类，继承自 Gtk.Label
class HIGSectionLabel(Gtk.Label):
    """
    Bold label, used to define sections
    """
    # 初始化方法
    def __init__(self, text=None):
        # 调用父类的初始化方法
        Gtk.Label.__init__(self)
        # 如果传入了文本参数
        if text:
            # 设置标签的文本为加粗格式
            self.set_markup("<b>%s</b>" % (text))
            # 设置文本左对齐
            self.set_justify(Gtk.Justification.LEFT)
            # 设置 x 轴对齐方式
            self.props.xalign = 0
            # 设置 y 轴对齐方式
            self.props.yalign = 0.5
            # 设置文本自动换行
            self.set_line_wrap(True)

# 定义 HIGHintSectionLabel 类，继承自 Gtk.Box
class HIGHintSectionLabel(Gtk.Box, object):
    """
    Bold label used to define sections, with a little icon that shows up a hint
    when mouse is over it.
    """
    # 初始化方法，用于创建对象实例
    def __init__(self, text=None, hint=None):
        # 调用父类的初始化方法，设置布局为水平方向
        Gtk.Box.__init__(self, orientation=Gtk.Orientation.HORIZONTAL)
    
        # 创建一个 HIGSectionLabel 对象，并将文本赋给 label 属性
        self.label = HIGSectionLabel(text)
        # 创建一个 Hint 对象，并将提示信息赋给 hint 属性
        self.hint = Hint(hint)
    
        # 将 label 对象添加到布局中，设置为不填充，不扩展，间距为 0
        self.pack_start(self.label, False, False, 0)
        # 将 hint 对象添加到布局中，设置为不填充，不扩展，间距为 5
        self.pack_start(self.hint, False, False, 5)
# 创建一个名为 Hint 的类，继承自 Gtk.EventBox 和 object
class Hint(Gtk.EventBox, object):
    # 初始化方法，接受一个提示参数
    def __init__(self, hint):
        # 调用父类的初始化方法
        Gtk.EventBox.__init__(self)
        # 将提示参数赋给实例变量
        self.hint = hint

        # 创建一个 Gtk.Image 对象作为提示图标
        self.hint_image = Gtk.Image()
        # 设置图标的来源为 "dialog-information"，大小为 SMALL_TOOLBAR
        self.hint_image.set_from_icon_name(
                "dialog-information", Gtk.IconSize.SMALL_TOOLBAR)

        # 将图标添加到事件盒子中
        self.add(self.hint_image)

        # 连接按钮按下事件到显示提示的方法
        self.connect("button-press-event", self.show_hint)

    # 显示提示的方法，接受一个小部件和事件参数
    def show_hint(self, widget, event=None):
        # 创建一个 HintWindow 对象，传入提示参数，并显示出来
        hint_window = HintWindow(self.hint)
        hint_window.show_all()


# 创建一个名为 HintWindow 的类，继承自 Gtk.Window
class HintWindow(Gtk.Window):
    # 初始化方法，接受一个提示参数
    def __init__(self, hint):
        # 调用父类的初始化方法，窗口类型为 POPUP
        Gtk.Window.__init__(self, type=Gtk.WindowType.POPUP)
        # 设置窗口位置为 MOUSE
        self.set_position(Gtk.WindowPosition.MOUSE)
        # 禁止窗口大小调整
        self.set_resizable(False)

        # 创建一个 Gdk.RGBA 对象，解析颜色值为 "#fbff99"
        bg_color = Gdk.RGBA()
        bg_color.parse("#fbff99")
        # 覆盖窗口的背景颜色为指定颜色
        self.override_background_color(Gtk.StateFlags.NORMAL, bg_color)

        # 创建一个事件盒子，覆盖背景颜色为指定颜色，设置边框宽度为 10
        self.event = Gtk.EventBox()
        self.event.override_background_color(Gtk.StateFlags.NORMAL, bg_color)
        self.event.set_border_width(10)
        # 连接按钮按下事件到关闭方法
        self.event.connect("button-press-event", self.close)

        # 创建一个标签，内容为提示参数，支持使用标记语言，自动换行，最大宽度为 52 个字符
        self.hint_label = Gtk.Label.new(hint)
        self.hint_label.set_use_markup(True)
        self.hint_label.set_line_wrap(True)
        self.hint_label.set_max_width_chars(52)
        # 设置水平对齐方式为左对齐，垂直对齐方式为居中
        self.hint_label.props.xalign = 0
        self.hint_label.props.yalign = 0.5

        # 将标签添加到事件盒子中，再将事件盒子添加到窗口中
        self.event.add(self.hint_label)
        self.add(self.event)

    # 关闭窗口的方法，接受一个小部件和事件参数
    def close(self, widget, event=None):
        # 销毁窗口
        self.destroy()


# 创建一个名为 HIGEntryLabel 的类，继承自 Gtk.Label
class HIGEntryLabel(Gtk.Label):
    """
    Simple label, like the ones used to label entries
    """
    # 初始化方法，接受一个文本参数
    def __init__(self, text=None):
        # 调用父类的初始化方法，内容为文本参数
        Gtk.Label.__init__(self, label=text)
        # 设置文本左对齐，水平对齐方式为左对齐，垂直对齐方式为居中
        self.set_justify(Gtk.Justification.LEFT)
        self.props.xalign = 0
        self.props.yalign = 0.5
        # 支持使用标记语言，自动换行
        self.set_use_markup(True)
        self.set_line_wrap(True)


# 创建一个名为 HIGDialogLabel 的类，继承自 Gtk.Label
class HIGDialogLabel(Gtk.Label):
    """
    Centered, line-wrappable label, usually used on dialogs.
    """
    # 初始化方法，用于创建对象实例
    def __init__(self, text=None):
        # 调用父类的初始化方法，传入文本内容作为标签
        Gtk.Label.__init__(self, label=text)
        # 设置文本内容居中显示
        self.set_justify(Gtk.Justification.CENTER)
        # 启用文本内容的标记语言解析
        self.set_use_markup(True)
        # 设置文本内容自动换行
        self.set_line_wrap(True)
# 如果当前模块被直接执行，则执行以下代码
if __name__ == "__main__":
    # 创建一个 GTK 窗口对象
    w = Gtk.Window()
    # 创建一个高亮的标签对象，包含标签文本和提示文本
    h = HIGHintSectionLabel("Label", "Hint")
    # 将高亮标签添加到窗口中
    w.add(h)
    # 连接窗口的 "delete-event" 信号到 Gtk.main_quit() 函数，以便关闭窗口
    w.connect("delete-event", lambda x, y: Gtk.main_quit())
    # 显示窗口中的所有部件
    w.show_all()
    # 运行 GTK 主循环，等待用户交互
    Gtk.main()
```