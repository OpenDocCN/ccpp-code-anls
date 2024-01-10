# `nmap\zenmap\zenmapGUI\ScanToolbar.py`

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
# 导入gi模块，用于与GTK+库进行交互
import gi

# 要求使用GTK+ 3.0版本
gi.require_version("Gtk", "3.0")
from gi.repository import Gtk

# 导入自定义模块
from zenmapGUI.higwidgets.higboxes import HIGHBox
from zenmapGUI.higwidgets.higlabels import HIGEntryLabel

# 导入国际化模块
import zenmapCore.I18N  # lgtm[py/unused-import]

# 导入自定义模块
from zenmapGUI.ProfileCombo import ProfileCombo
from zenmapGUI.TargetCombo import TargetCombo

# 定义一个名为ScanCommandToolbar的类，继承自HIGHBox
class ScanCommandToolbar(HIGHBox):
    """This class builds the toolbar devoted to Command entry. It allows you to
    retrieve and edit the current command entered."""
    # 初始化方法
    def __init__(self):
        """Initialize command toolbar"""
        # 调用父类的初始化方法
        HIGHBox.__init__(self)

        # 创建一个标签和一个文本输入框
        self.command_label = HIGEntryLabel(_("Command:"))
        self.command_entry = Gtk.Entry()

        # 将标签添加到工具栏中
        self._pack_noexpand_nofill(self.command_label)
        # 将文本输入框添加到工具栏中
        self._pack_expand_fill(self.command_entry)
    # 定义一个方法，用于获取命令输入框中的内容
    def get_command(self):
        """Retrieve command entry"""
        return self.command_entry.get_text()
    
    # 定义一个方法，用于设置命令输入框中的内容
    def set_command(self, command):
        """Set a command entry"""
        self.command_entry.set_text(command)
    
    # 创建一个属性，将获取命令输入框内容的方法和设置命令输入框内容的方法绑定到一起
    command = property(get_command, set_command)
# 创建一个名为ScanToolbar的类，继承自HIGHBox
class ScanToolbar(HIGHBox):
    """
    This function regards the Scanning Toolbar, which includes
    the Target and Profile editable fields/dropdown boxes, as well as
    the Scan button and assigns events and and actions associated with
    each.
    """
    # 初始化Scan Toolbar，包括事件和布局中的所有GUI元素
    def __init__(self):
        """Initialize Scan Toolbar, including Events, and packing all
        of the GUI elements in layout"""
        # 调用父类的初始化方法
        HIGHBox.__init__(self)

        # 创建目标字段和更新列表
        self._create_target()
        # 创建配置字段和更新列表
        self._create_profile()

        # 创建一个名为Scan的按钮
        self.scan_button = Gtk.Button.new_with_label(_("Scan"))
        # 创建一个名为Cancel的按钮
        self.cancel_button = Gtk.Button.new_with_label(_("Cancel"))

        # 将目标标签添加到布局中
        self._pack_noexpand_nofill(self.target_label)
        # 将目标输入框添加到布局中
        self._pack_expand_fill(self.target_entry)

        # 将配置标签添加到布局中
        self._pack_noexpand_nofill(self.profile_label)
        # 将配置输入框添加到布局中
        self._pack_expand_fill(self.profile_entry)

        # 将Scan按钮添加到布局中
        self._pack_noexpand_nofill(self.scan_button)
        # 将Cancel按钮添加到布局中
        self._pack_noexpand_nofill(self.cancel_button)

        # 跳过下拉箭头，以便可以通过Tab键到达配置输入框
        self.target_entry.set_focus_chain((self.target_entry.get_child(),))

        # 当目标输入框中的内容被激活时，将焦点转移到配置输入框
        self.target_entry.get_child().connect('activate',
                        lambda x: self.profile_entry.grab_focus())
        # 当配置输入框中的内容被激活时，模拟点击Scan按钮
        self.profile_entry.get_child().connect('activate',
                        lambda x: self.scan_button.clicked())

    # 创建一个名为_create_target的私有方法
    def _create_target(self):
        """Create a target and update the list"""
        # 创建一个名为Target的标签
        self.target_label = HIGEntryLabel(_("Target:"))
        # 创建一个名为TargetCombo的目标组合框
        self.target_entry = TargetCombo()

        # 更新目标列表
        self.update_target_list()

    # 创建一个名为_create_profile的私有方法
    def _create_profile(self):
        """Create new profile and update list"""
        # 创建一个名为Profile的标签
        self.profile_label = HIGEntryLabel(_('Profile:'))
        # 创建一个名为ProfileCombo的配置组合框
        self.profile_entry = ProfileCombo()

        # 更新配置列表
        self.update()

    # 更新目标列表的方法
    def update_target_list(self):
        self.target_entry.update()

    # 添加新目标的方法
    def add_new_target(self, target):
        self.target_entry.add_new_target(target)
    # 返回当前选定的目标
    def get_selected_target(self):
        return self.target_entry.selected_target
    
    # 修改当前选定的目标
    def set_selected_target(self, target):
        self.target_entry.selected_target = target
    
    # 更新界面
    def update(self):
        self.profile_entry.update()
    
    # 修改配置文件
    def set_profiles(self, profiles):
        self.profile_entry.set_profiles(profiles)
    
    # 返回当前选定的配置文件
    def get_selected_profile(self):
        return self.profile_entry.selected_profile
    
    # 修改当前选定的配置文件
    def set_selected_profile(self, profile):
        self.profile_entry.selected_profile = profile
    
    # 使用属性装饰器创建属性，使得可以通过 selected_profile 来访问和修改当前选定的配置文件
    selected_profile = property(get_selected_profile, set_selected_profile)
    
    # 使用属性装饰器创建属性，使得可以通过 selected_target 来访问和修改当前选定的目标
    selected_target = property(get_selected_target, set_selected_target)
# 如果当前模块被直接执行，则执行以下代码
if __name__ == "__main__":
    # 创建一个 GTK 窗口对象
    w = Gtk.Window()
    # 创建一个垂直方向的 GTK 盒子对象
    box = Gtk.Box.new(Gtk.Orientation.VERTICAL, 0)
    # 将盒子对象添加到窗口中
    w.add(box)

    # 创建一个扫描工具栏对象
    stool = ScanToolbar()
    # 创建一个扫描命令工具栏对象
    sctool = ScanCommandToolbar()

    # 将扫描工具栏对象添加到盒子中
    box.pack_start(stool, True, True, 0)
    # 将扫描命令工具栏对象添加到盒子中
    box.pack_start(sctool, True, True, 0)

    # 连接窗口的 "delete-event" 信号，当窗口关闭时退出 GTK 主循环
    w.connect("delete-event", lambda x, y: Gtk.main_quit())
    # 显示窗口中的所有控件
    w.show_all()
    # 进入 GTK 主循环
    Gtk.main()
```