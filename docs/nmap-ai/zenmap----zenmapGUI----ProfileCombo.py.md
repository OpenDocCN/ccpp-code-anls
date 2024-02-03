# `nmap\zenmap\zenmapGUI\ProfileCombo.py`

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
# 导入gi模块，用于与GTK+库进行交互
import gi

# 要求使用GTK+ 3.0版本
gi.require_version("Gtk", "3.0")
from gi.repository import Gtk

# 导入CommandProfile类
from zenmapCore.UmitConf import CommandProfile
# 导入I18N模块，用于国际化
import zenmapCore.I18N  # lgtm[py/unused-import]

# 创建ProfileCombo类，继承自Gtk.ComboBoxText类
class ProfileCombo(Gtk.ComboBoxText, object):
    # 初始化方法
    def __init__(self):
        # 调用父类的初始化方法
        Gtk.ComboBoxText.__init__(self, has_entry=True)

        # 创建一个EntryCompletion对象
        self.completion = Gtk.EntryCompletion()
        # 将EntryCompletion对象与ComboBoxText的子部件关联
        self.get_child().set_completion(self.completion)
        # 设置EntryCompletion的数据模型
        self.completion.set_model(self.get_model())
        # 设置EntryCompletion显示文本的列
        self.completion.set_text_column(0)

        # 调用update方法
        self.update()

    # 设置profiles的方法
    def set_profiles(self, profiles):
        # 移除所有已有的项
        self.remove_all()

        # 遍历profiles列表，将每个command添加到ComboBoxText中
        for command in profiles:
            self.append_text(command)
    # 定义一个方法用于更新配置文件
    def update(self):
        # 创建一个命令配置文件对象
        profile = CommandProfile()
        # 获取配置文件中的所有部分
        profiles = profile.sections()
        # 对部分进行排序
        profiles.sort()
        # 删除配置文件对象
        del(profile)
        # 调用对象的方法设置配置文件
        self.set_profiles(profiles)
    
    # 定义一个方法用于获取选定的配置文件
    def get_selected_profile(self):
        # 返回子对象的文本内容
        return self.get_child().get_text()
    
    # 定义一个方法用于设置选定的配置文件
    def set_selected_profile(self, profile):
        # 设置子对象的文本内容为指定的配置文件
        self.get_child().set_text(profile)
    
    # 创建一个属性，用于获取和设置选定的配置文件
    selected_profile = property(get_selected_profile, set_selected_profile)
# 如果当前模块被直接执行，则执行以下代码
if __name__ == "__main__":
    # 创建一个 GTK 窗口对象
    w = Gtk.Window()
    # 创建一个 ProfileCombo 对象
    p = ProfileCombo()
    # 更新 ProfileCombo 对象的内容
    p.update()
    # 将 ProfileCombo 对象添加到窗口中
    w.add(p)

    # 连接窗口的 "delete-event" 信号到 Gtk.main_quit() 函数，以便在关闭窗口时退出主循环
    w.connect("delete-event", lambda x, y: Gtk.main_quit())
    # 显示窗口中的所有部件
    w.show_all()
    # 进入 GTK 主循环
    Gtk.main()
```