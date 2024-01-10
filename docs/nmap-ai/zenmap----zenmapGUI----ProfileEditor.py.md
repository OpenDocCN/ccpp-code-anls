# `nmap\zenmap\zenmapGUI\ProfileEditor.py`

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
# 导入 gi 模块
import gi
# 要求使用 Gtk 3.0 版本
gi.require_version("Gtk", "3.0")
# 从 gi.repository 中导入 Gtk 模块
from gi.repository import Gtk

# 从 zenmapGUI.higwidgets.higwindows 模块中导入 HIGWindow 类
from zenmapGUI.higwidgets.higwindows import HIGWindow
# 从 zenmapGUI.higwidgets.higboxes 模块中导入 HIGVBox, HIGHBox, HIGSpacer, hig_box_space_holder 类
from zenmapGUI.higwidgets.higboxes import HIGVBox, HIGHBox, HIGSpacer, \
        hig_box_space_holder
# 从 zenmapGUI.higwidgets.higlabels 模块中导入 HIGSectionLabel, HIGEntryLabel 类
from zenmapGUI.higwidgets.higlabels import HIGSectionLabel, HIGEntryLabel
# 从 zenmapGUI.higwidgets.higscrollers 模块中导入 HIGScrolledWindow 类
from zenmapGUI.higwidgets.higscrollers import HIGScrolledWindow
# 从 zenmapGUI.higwidgets.higtextviewers 模块中导入 HIGTextView 类
from zenmapGUI.higwidgets.higtextviewers import HIGTextView
# 从 zenmapGUI.higwidgets.higbuttons 模块中导入 HIGButton 类
from zenmapGUI.higwidgets.higbuttons import HIGButton
# 从 zenmapGUI.higwidgets.higtables 模块中导入 HIGTable 类
from zenmapGUI.higwidgets.higtables import HIGTable
# 从 zenmapGUI.higwidgets.higdialogs 模块中导入 HIGAlertDialog, HIGDialog 类
from zenmapGUI.higwidgets.higdialogs import HIGAlertDialog, HIGDialog
# 从 zenmapGUI.OptionBuilder 模块中导入 OptionBuilder 类
from zenmapGUI.OptionBuilder import OptionBuilder
# 从 zenmapCore.Paths 模块中导入 Path 类
from zenmapCore.Paths import Path
# 从 zenmapCore.UmitConf 模块中导入 CommandProfile 类
from zenmapCore.UmitConf import CommandProfile
# 从 zenmapCore.UmitLogging 模块中导入 log 函数
from zenmapCore.UmitLogging import log
# 导入zenmapCore.I18N模块，用于国际化
import zenmapCore.I18N  # lgtm[py/unused-import]
# 从zenmapCore.NmapOptions模块中导入NmapOptions类
from zenmapCore.NmapOptions import NmapOptions


# 定义ProfileEditor类，继承自HIGWindow类
class ProfileEditor(HIGWindow):
    # 初始化函数，设置命令、配置文件名、是否可删除、是否覆盖等参数
    def __init__(self, command=None, profile_name=None,
            deletable=True, overwrite=False):
        # 调用父类的初始化函数
        HIGWindow.__init__(self)
        # 连接窗口的删除事件到退出函数
        self.connect("delete_event", self.exit)
        # 设置窗口标题为'Profile Editor'
        self.set_title(_('Profile Editor'))
        # 设置窗口位置为居中
        self.set_position(Gtk.WindowPosition.CENTER)

        # 设置是否可删除、配置文件名、是否覆盖等属性
        self.deletable = deletable
        self.profile_name = profile_name
        self.overwrite = overwrite

        # 用于阻止命令输入框递归更新时的标志
        self.inhibit_command_update = False

        # 创建窗口部件
        self.__create_widgets()
        # 将窗口部件打包
        self.__pack_widgets()

        # 创建命令配置文件对象
        self.profile = CommandProfile()

        # 创建 NmapOptions 对象
        self.ops = NmapOptions()
        # 如果有配置文件名，则显示配置文件内容
        if profile_name:
            log.debug("Showing profile %s" % profile_name)
            prof = self.profile.get_profile(profile_name)

            # 设置配置文件名输入框的文本为配置文件名
            self.profile_name_entry.set_text(profile_name)
            # 设置配置文件描述文本框的内容为配置文件描述
            self.profile_description_text.get_buffer().set_text(
                    prof['description'])

            # 获取配置文件中的命令字符串
            command_string = prof['command']
            # 解析命令字符串为选项对象
            self.ops.parse_string(command_string)
        # 如果有命令，则解析命令字符串为选项对象
        if command:
            self.ops.parse_string(command)

        # 创建 OptionBuilder 对象，用于构建选项界面
        self.option_builder = OptionBuilder(
                Path.profile_editor, self.ops,
                self.update_command, self.help_field.get_buffer())
        # 打印选项组
        log.debug("Option groups: %s" % str(self.option_builder.groups))
        # 打印选项部分名称
        log.debug("Option section names: %s" % str(
            self.option_builder.section_names))
        # 打印选项标签
        #log.debug("Option tabs: %s" % str(self.option_builder.tabs))

        # 遍历选项组，创建选项卡
        for tab in self.option_builder.groups:
            self.__create_tab(
                    _(tab),
                    _(self.option_builder.section_names[tab]),
                    self.option_builder.tabs[tab])

        # 更新命令显示
        self.update_command()
    # 当命令行输入框内容改变时的回调函数
    def command_entry_changed_cb(self, widget):
        # 获取命令行输入框中的文本内容
        command_string = self.command_entry.get_text()
        # 解析命令行字符串
        self.ops.parse_string(command_string)
        # 阻止命令更新
        self.inhibit_command_update = True
        # 更新选项构建器
        self.option_builder.update()
        # 取消阻止命令更新
        self.inhibit_command_update = False

    # 更新命令
    def update_command(self):
        """重新生成并显示命令。"""
        if not self.inhibit_command_update:
            # 阻止选项构建器小部件递归更新，当它们导致命令输入框发生变化时
            self.command_entry.handler_block(self.command_entry_changed_cb_id)
            self.command_entry.set_text(self.ops.render_string())
            self.command_entry.handler_unblock(
                    self.command_entry_changed_cb_id)

    # 更新帮助名称
    def update_help_name(self, widget, extra):
        self.help_field.get_buffer().set_text(
                "Profile name\n\nThis is how the profile will be identified "
                "in the drop-down combo box in the scan tab.")

    # 更新帮助描述
    def update_help_desc(self, widget, extra):
        self.help_field.get_buffer().set_text(
                "Description\n\nThe description is a full description of what "
                "the scan does, which may be long.")

    # 创建选项卡
    def __create_tab(self, tab_name, section_name, tab):
        log.debug(">>> Tab name: %s" % tab_name)
        log.debug(">>>Creating profile editor section: %s" % section_name)
        # 创建垂直布局
        vbox = HIGVBox()
        # 如果不是脚本选项卡
        if tab.notscripttab:  
            # 创建表格
            table = HIGTable()
            table.set_row_spacings(2)
            # 创建部分标签
            section = HIGSectionLabel(section_name)
            vbox._pack_noexpand_nofill(section)
            vbox._pack_noexpand_nofill(HIGSpacer(table))
            vbox.set_border_width(5)
            # 填充表格
            tab.fill_table(table, True)
        else:
            # 获取水平主框
            hbox = tab.get_hmain_box()
            vbox.pack_start(hbox, True, True, 0)
        # 将垂直布局添加到笔记本
        self.notebook.append_page(vbox, Gtk.Label.new(tab_name))
    # 保存配置文件
    def save_profile(self, widget):
        # 如果允许覆盖，则删除同名配置文件
        if self.overwrite:
            self.profile.remove_profile(self.profile_name)
        # 获取配置文件名
        profile_name = self.profile_name_entry.get_text()
        # 如果配置文件名为空，弹出警告对话框
        if profile_name == '':
            alert = HIGAlertDialog(
                    message_format=_('Unnamed profile'),
                    secondary_text=_(
                        'You must provide a name for this profile.'))
            alert.run()
            alert.destroy()
            # 将焦点设置到配置文件名输入框
            self.profile_name_entry.grab_focus()
            return None

        # 获取命令
        command = self.ops.render_string()

        # 获取配置文件描述
        buf = self.profile_description_text.get_buffer()
        description = buf.get_text(
                buf.get_start_iter(), buf.get_end_iter(), include_hidden_chars=True)

        # 尝试添加配置文件
        try:
            self.profile.add_profile(
                    profile_name,
                    command=command,
                    description=description)
        # 如果配置文件名不合法，弹出警告对话框
        except ValueError:
            alert = HIGAlertDialog(
                    message_format=_('Disallowed profile name'),
                    secondary_text=_('Sorry, the name "%s" is not allowed due '
                        'to technical limitations. (The underlying '
                        'ConfigParser used to store profiles does not allow '
                        'it.) Choose a different name.' % profile_name))
            alert.run()
            alert.destroy()
            return

        # 更新扫描界面的工具栏
        self.scan_interface.toolbar.profile_entry.update()
        # 销毁当前窗口
        self.destroy()

    # 清空配置文件信息
    def clean_profile_info(self):
        self.profile_name_entry.set_text('')
        self.profile_description_text.get_buffer().set_text('')

    # 设置扫描界面
    def set_scan_interface(self, interface):
        self.scan_interface = interface

    # 退出函数
    def exit(self, *args):
        self.destroy()
    # 删除用户配置文件的方法
    def delete_profile(self, widget=None, extra=None):
        # 如果可删除
        if self.deletable:
            # 创建一个对话框
            dialog = HIGDialog(buttons=(Gtk.STOCK_OK, Gtk.ResponseType.OK,
                                        Gtk.STOCK_CANCEL, Gtk.ResponseType.CANCEL))
            # 创建一个警告标签
            alert = HIGEntryLabel('<b>' + _("Deleting Profile") + '</b>')
            # 创建一个文本标签
            text = HIGEntryLabel(_(
                'Your profile is going to be deleted! Click Ok to continue, '
                'or Cancel to go back to Profile Editor.'))
            # 创建水平盒子
            hbox = HIGHBox()
            hbox.set_border_width(5)
            hbox.set_spacing(12)

            # 创建垂直盒子
            vbox = HIGVBox()
            vbox.set_border_width(5)
            vbox.set_spacing(12)

            # 创建一个图像
            image = Gtk.Image()
            image.set_from_stock(
                    Gtk.STOCK_DIALOG_WARNING, Gtk.IconSize.DIALOG)

            # 将警告标签和文本标签添加到垂直盒子中
            vbox.pack_start(alert, True, True, 0)
            vbox.pack_start(text, True, True, 0)
            # 将图像和垂直盒子添加到水平盒子中
            hbox.pack_start(image, True, True, 0)
            hbox.pack_start(vbox, True, True, 0)

            # 将水平盒子添加到对话框中并显示所有部件
            dialog.vbox.pack_start(hbox, True, True, 0)
            dialog.vbox.show_all()

            # 运行对话框并销毁
            response = dialog.run()
            dialog.destroy()
            # 如果用户选择取消，则返回True
            if response == Gtk.ResponseType.CANCEL:
                return True
            # 否则删除用户配置文件
            self.profile.remove_profile(self.profile_name)

        # 更新用户配置文件条目并销毁窗口
        self.update_profile_entry()
        self.destroy()

    # 运行扫描的方法
    def run_scan(self, widget=None):
        # 获取命令行输入的字符串
        command_string = self.command_entry.get_text()
        # 设置扫描接口的命令
        self.scan_interface.command_toolbar.command = command_string
        # 调用扫描接口的开始扫描方法
        self.scan_interface.start_scan_cb()
        # 退出窗口
        self.exit()

    # 更新用户配置文件条目的方法
    def update_profile_entry(self, widget=None, extra=None):
        # 更新扫描接口的工具栏中的用户配置文件条目
        self.scan_interface.toolbar.profile_entry.update()
        # 获取用户配置文件列表
        list = self.scan_interface.toolbar.profile_entry.get_model()
        # 获取用户配置文件列表的长度
        length = len(list)
        # 如果列表长度大于0，则设置扫描接口的工具栏中的用户配置文件条目为第一个
        if length > 0:
            self.scan_interface.toolbar.profile_entry.set_active(0)
# 如果当前模块被直接执行，而不是被导入到其他模块中
if __name__ == '__main__':
    # 创建一个ProfileEditor对象
    p = ProfileEditor()
    # 显示ProfileEditor对象的所有内容
    p.show_all()
    # 进入GTK的主循环，等待用户交互
    Gtk.main()
```