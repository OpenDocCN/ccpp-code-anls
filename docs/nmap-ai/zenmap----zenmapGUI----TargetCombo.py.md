# `nmap\zenmap\zenmapGUI\TargetCombo.py`

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
# 导入gi模块，要求使用版本为"Gtk"和"3.0"
import gi

# 要求使用版本为"Gtk"和"3.0"的模块
gi.require_version("Gtk", "3.0")
# 从gi.repository中导入Gtk模块
from gi.repository import Gtk

# 从zenmapCore.TargetList模块中导入target_list
from zenmapCore.TargetList import target_list

# 创建TargetCombo类，继承自Gtk.ComboBoxText类
class TargetCombo(Gtk.ComboBoxText):
    # 初始化方法
    def __init__(self):
        # 调用父类的初始化方法，设置has_entry为True
        Gtk.ComboBoxText.__init__(self, has_entry=True)

        # 创建一个EntryCompletion对象
        self.completion = Gtk.EntryCompletion()
        # 设置子对象的自动完成为创建的EntryCompletion对象
        self.get_child().set_completion(self.completion)
        # 设置自动完成的模型为当前对象的模型
        self.completion.set_model(self.get_model())
        # 设置自动完成的文本列为第一列
        self.completion.set_text_column(0)

        # 调用update方法
        self.update()

    # 更新方法
    def update(self):
        # 移除所有选项
        self.remove_all()

        # 获取目标列表
        t_list = target_list.get_target_list()
        # 遍历目标列表的前15个目标
        for target in t_list[:15]:
            # 将目标添加到下拉列表中，替换换行符为空字符
            self.append_text(target.replace('\n', ''))

    # 添加新目标方法
    def add_new_target(self, target):
        # 向目标列表中添加新目标
        target_list.add_target(target)
        # 调用update方法
        self.update()
    # 获取选定的目标，通过调用子元素的获取文本方法
    def get_selected_target(self):
        return self.get_child().get_text()
    
    # 设置选定的目标，通过调用子元素的设置文本方法
    def set_selected_target(self, target):
        self.get_child().set_text(target)
    
    # 创建属性 selected_target，使其可以通过 property 方法获取和设置选定的目标
    selected_target = property(get_selected_target, set_selected_target)
# 如果当前模块被直接执行，则执行以下代码
if __name__ == "__main__":
    # 创建一个 GTK 窗口对象
    w = Gtk.Window()
    # 创建一个目标选择下拉框对象
    t = TargetCombo()
    # 将目标选择下拉框对象添加到窗口中
    w.add(t)

    # 连接窗口的 "delete-event" 信号到 Gtk.main_quit() 函数，以便在关闭窗口时退出主循环
    w.connect("delete-event", lambda x, y: Gtk.main_quit())
    # 显示窗口及其所有子部件
    w.show_all()
    # 进入 GTK 主循环
    Gtk.main()
```