# `nmap\zenmap\zenmapGUI\OptionBuilder.py`

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
# 导入gi模块，要求版本为"Gtk"的"3.0"
import gi
# 从gi.repository中导入Gtk和GObject
gi.require_version("Gtk", "3.0")
from gi.repository import Gtk, GObject

# 防止加载PyXML
import xml
# 将xml.__path__中不包含"_xmlplus"的路径重新赋值给xml.__path__
xml.__path__ = [x for x in xml.__path__ if "_xmlplus" not in x]

# 从xml.dom中导入minidom
from xml.dom import minidom

# 从zenmapGUI.higwidgets.higlabels中导入HIGEntryLabel
# 从zenmapGUI.higwidgets.higbuttons中导入HIGButton
from zenmapGUI.higwidgets.higlabels import HIGEntryLabel
from zenmapGUI.higwidgets.higbuttons import HIGButton

# 从zenmapGUI.FileChoosers中导入AllFilesFileChooserDialog
# 从zenmapGUI.ProfileHelp中导入ProfileHelp
from zenmapGUI.FileChoosers import AllFilesFileChooserDialog
from zenmapGUI.ProfileHelp import ProfileHelp

# 从zenmapCore.NmapOptions中导入NmapOptions, split_quoted, join_quoted
import zenmapCore.NmapOptions
# 从zenmapGUI.ScriptInterface中导入ScriptInterface
from zenmapGUI.ScriptInterface import ScriptInterface

# 定义函数get_option_check_auxiliary_widget，参数为option, ops, check
def get_option_check_auxiliary_widget(option, ops, check):
    # 如果选项在指定的参数列表中
    if option in ("-sI", "-b", "--script", "--script-args", "--exclude", "-p",
                "-D", "-S", "--source-port", "-e", "--ttl", "-iR", "--max-retries",
                "--host-timeout", "--max-rtt-timeout", "--min-rtt-timeout",
                "--initial-rtt-timeout", "--max-hostgroup", "--min-hostgroup",
                "--max-parallelism", "--min-parallelism", "--max-scan-delay",
                "--scan-delay", "-PA", "-PS", "-PU", "-PO", "-PY"):
        # 返回一个包含选项、操作和检查的 OptionEntry 对象
        return OptionEntry(option, ops, check)
    # 如果选项在指定的参数列表中
    elif option in ("-d", "-v"):
        # 返回一个包含选项、操作和检查的 OptionLevel 对象
        return OptionLevel(option, ops, check)
    # 如果选项在指定的参数列表中
    elif option in ("--excludefile", "-iL"):
        # 返回一个包含选项、操作和检查的 OptionFile 对象
        return OptionFile(option, ops, check)
    # 如果选项在指定的参数列表中
    elif option in ("-A", "-O", "-sV", "-n", "-6", "-Pn", "-PE", "-PP", "-PM",
                "-PB", "-sC", "--script-trace", "-F", "-f", "--packet-trace", "-r",
                "--traceroute"):
        # 返回空对象
        return None
    # 如果选项在指定的参数列表中
    elif option in ("",):
        # 返回一个包含选项、操作和检查的 OptionExtras 对象
        return OptionExtras(option, ops, check)
    # 如果选项不在任何指定的参数列表中
    else:
        # 抛出异常，显示未知选项
        assert False, "Unknown option %s" % option
# 创建一个名为 OptionEntry 的类，继承自 Gtk.Entry
class OptionEntry(Gtk.Entry):
    # 初始化方法，接受 option、ops 和 check 三个参数
    def __init__(self, option, ops, check):
        # 调用父类的初始化方法
        Gtk.Entry.__init__(self)
        # 设置实例变量 option、ops 和 check
        self.option = option
        self.ops = ops
        self.check = check
        # 连接 "changed" 信号到 self.changed_cb 方法
        self.connect("changed", self.changed_cb)
        # 连接 check 的 "toggled" 信号到 self.check_toggled_cb 方法
        self.check.connect("toggled", self.check_toggled_cb)
        # 调用 update 方法
        self.update()

    # 定义 update 方法
    def update(self):
        # 如果 ops 中 option 对应的值不为 None
        if self.ops[self.option] is not None:
            # 设置文本为 ops 中 option 对应的值的字符串形式
            self.set_text(str(self.ops[self.option]))
            # 设置 check 为激活状态
            self.check.set_active(True)
        else:
            # 否则设置文本为空
            self.set_text("")
            # 设置 check 为非激活状态
            self.check.set_active(False)

    # 定义 check_toggled_cb 方法，接受 check 参数
    def check_toggled_cb(self, check):
        # 如果 check 处于激活状态
        if check.get_active():
            # 将 ops 中 option 对应的值设置为当前文本
            self.ops[self.option] = self.get_text()
        else:
            # 否则将 ops 中 option 对应的值设置为 None
            self.ops[self.option] = None

    # 定义 changed_cb 方法，接受 widget 参数
    def changed_cb(self, widget):
        # 设置 check 为激活状态
        self.check.set_active(True)
        # 将 ops 中 option 对应的值设置为当前文本
        self.ops[self.option] = self.get_text()


# 创建一个名为 OptionExtras 的类，继承自 Gtk.Entry
class OptionExtras(Gtk.Entry):
    # 初始化方法，接受 option、ops 和 check 三个参数
    def __init__(self, option, ops, check):
        # 调用父类的初始化方法
        Gtk.Entry.__init__(self)
        # 设置实例变量 ops 和 check
        self.ops = ops
        self.check = check
        # 连接 "changed" 信号到 self.changed_cb 方法
        self.connect("changed", self.changed_cb)
        # 连接 check 的 "toggled" 信号到 self.check_toggled_cb 方法
        self.check.connect("toggled", self.check_toggled_cb)
        # 调用 update 方法
        self.update()

    # 定义 update 方法
    def update(self):
        # 如果 ops 中 extras 的长度大于 0
        if len(self.ops.extras) > 0:
            # 设置文本为 ops 中 extras 的元素用空格连接起来的字符串
            self.set_text(" ".join(self.ops.extras))
            # 设置 check 为激活状态
            self.check.set_active(True)
        else:
            # 否则设置文本为空
            self.set_text("")
            # 设置 check 为非激活状态
            self.check.set_active(False)

    # 定义 check_toggled_cb 方法，接受 check 参数
    def check_toggled_cb(self, check):
        # 如果 check 处于激活状态
        if check.get_active():
            # 将 ops 中 extras 设置为包含当前文本的列表
            self.ops.extras = [self.get_text()]
        else:
            # 否则将 ops 中 extras 设置为空列表
            self.ops.extras = []

    # 定义 changed_cb 方法，接受 widget 参数
    def changed_cb(self, widget):
        # 设置 check 为激活状态
        self.check.set_active(True)
        # 将 ops 中 extras 设置为包含当前文本的列表
        self.ops.extras = [self.get_text()]


# 创建一个名为 OptionLevel 的类，继承自 Gtk.SpinButton
    # 初始化方法，接受选项、操作和检查参数
    def __init__(self, option, ops, check):
        # 创建一个调整对象，初始值为0，最小值为0，最大值为10，步进为1，页面大小为0，页面步进为0
        adjustment = Gtk.Adjustment.new(0, 0, 10, 1, 0, 0)
        # 调用父类的初始化方法，传入调整对象和其他参数
        Gtk.SpinButton.__init__(self, adjustment=adjustment, climb_rate=0.0, digits=0)
        # 设置实例的option属性为传入的option参数
        self.option = option
        # 设置实例的ops属性为传入的ops参数
        self.ops = ops
        # 设置实例的check属性为传入的check参数
        self.check = check
        # 连接"changed"信号到self.changed_cb方法
        self.connect("changed", self.changed_cb)
        # 连接"toggled"信号到self.check_toggled_cb方法
        self.check.connect("toggled", self.check_toggled_cb)
        # 调用update方法
        self.update()

    # 更新方法
    def update(self):
        # 获取选项对应的操作
        level = self.ops[self.option]
        # 如果操作不为None且大于0
        if level is not None and level > 0:
            # 设置调整对象的值为操作的整数值
            self.get_adjustment().set_value(int(level))
            # 设置检查按钮为激活状态
            self.check.set_active(True)
        else:
            # 否则设置调整对象的值为0
            self.get_adjustment().set_value(0)
            # 设置检查按钮为非激活状态
            self.check.set_active(False)

    # 检查按钮状态改变的回调方法
    def check_toggled_cb(self, check):
        # 如果检查按钮为激活状态
        if check.get_active():
            # 设置选项对应的操作为调整对象的整数值
            self.ops[self.option] = int(self.get_adjustment().get_value())
        else:
            # 否则设置选项对应的操作为0
            self.ops[self.option] = 0

    # 调整对象值改变的回调方法
    def changed_cb(self, widget):
        # 设置检查按钮为激活状态
        self.check.set_active(True)
        # 设置选项对应的操作为调整对象的整数值
        self.ops[self.option] = int(self.get_adjustment().get_value())
# 定义一个名为 OptionFile 的类，继承自 Gtk.Box
class OptionFile(Gtk.Box):
    # 定义信号 "changed"，当发生变化时会触发
    __gsignals__ = {
        "changed": (GObject.SignalFlags.RUN_FIRST, GObject.TYPE_NONE, ())
    }

    # 初始化方法，接受 option、ops 和 check 三个参数
    def __init__(self, option, ops, check):
        # 调用父类的初始化方法，设置布局为水平方向
        Gtk.Box.__init__(self, orientation=Gtk.Orientation.HORIZONTAL)

        # 设置实例变量 option、ops 和 check
        self.option = option
        self.ops = ops
        self.check = check

        # 创建一个文本输入框
        self.entry = Gtk.Entry()
        # 将文本输入框添加到布局中
        self.pack_start(self.entry, True, True, 0)
        # 创建一个按钮，按钮的图标为打开文件的图标
        button = HIGButton(stock=Gtk.STOCK_OPEN)
        # 将按钮添加到布局中
        self.pack_start(button, False, True, 0)

        # 连接按钮的点击事件到 clicked_cb 方法
        button.connect("clicked", self.clicked_cb)

        # 连接文本输入框的内容变化事件到 emit 方法
        self.entry.connect("changed", lambda x: self.emit("changed"))
        # 连接文本输入框的内容变化事件到 changed_cb 方法
        self.entry.connect("changed", self.changed_cb)
        # 连接复选框的状态变化事件到 check_toggled_cb 方法
        self.check.connect("toggled", self.check_toggled_cb)
        # 调用 update 方法
        self.update()

    # 更新方法
    def update(self):
        # 如果 ops 中 option 对应的值不为 None
        if self.ops[self.option] is not None:
            # 设置文本输入框的内容为 ops 中 option 对应的值
            self.entry.set_text(self.ops[self.option])
            # 设置复选框为选中状态
            self.check.set_active(True)
        else:
            # 否则，清空文本输入框的内容
            self.entry.set_text("")
            # 设置复选框为未选中状态
            self.check.set_active(False)

    # 复选框状态变化的回调方法
    def check_toggled_cb(self, check):
        # 如果复选框为选中状态
        if check.get_active():
            # 将 ops 中 option 对应的值设置为文本输入框的内容
            self.ops[self.option] = self.entry.get_text()
        else:
            # 否则，将 ops 中 option 对应的值设置为 None
            self.ops[self.option] = None

    # 文本输入框内容变化的回调方法
    def changed_cb(self, widget):
        # 设置复选框为选中状态
        self.check.set_active(True)
        # 将 ops 中 option 对应的值设置为文本输入框的内容
        self.ops[self.option] = self.entry.get_text()

    # 按钮点击的回调方法
    def clicked_cb(self, button):
        # 创建一个文件选择对话框
        dialog = AllFilesFileChooserDialog(_("Choose file"))
        # 如果对话框返回结果为 OK
        if dialog.run() == Gtk.ResponseType.OK:
            # 设置文本输入框的内容为对话框选择的文件名
            self.entry.set_text(dialog.get_filename())
        # 销毁对话框
        dialog.destroy()


# 定义一个名为 TargetEntry 的类，继承自 Gtk.Entry
class TargetEntry(Gtk.Entry):
    # 初始化方法，接受 ops 一个参数
    def __init__(self, ops):
        # 调用父类的初始化方法
        Gtk.Entry.__init__(self)
        # 设置实例变量 ops
        self.ops = ops
        # 连接文本输入框的内容变化事件到 changed_cb 方法
        self.connect("changed", self.changed_cb)
        # 调用 update 方法
        self.update()

    # 更新方法
    def update(self):
        # 设置文本输入框的内容为 ops 中 target_specs 的值，用空格连接
        self.set_text(" ".join(self.ops.target_specs))

    # 文本输入框内容变化的回调方法
    def changed_cb(self, widget):
        # 将 ops 中 target_specs 的值设置为文本输入框内容的分割结果
        self.ops.target_specs = self.get_targets()

    # 获取目标的方法
    def get_targets(self):
        # 返回文本输入框内容按空格分割的结果
        return split_quoted(self.get_text())
# 定义 OptionTab 类
class OptionTab(object):
    # 初始化方法
    def __init__(self, root_tab, ops, update_command, help_buf):
        # 定义操作字典
        actions = {'target': self.__parse_target,
                   'option_list': self.__parse_option_list,
                   'option_check': self.__parse_option_check}

        # 初始化实例变量
        self.ops = ops
        self.update_command = update_command
        self.help_buf = help_buf

        # 创建 ProfileHelp 实例
        self.profilehelp = ProfileHelp()
        # 假设每个选项卡都不是脚本选项卡
        self.notscripttab = False
        # 初始化小部件列表
        self.widgets_list = []
        # 遍历根选项卡的子节点
        for option_element in root_tab.childNodes:
            # 如果节点有标签属性且标签在操作字典中
            if (hasattr(option_element, "tagName") and
                    option_element.tagName in actions.keys()):
                # 获取对应的解析方法
                parse_func = actions[option_element.tagName]
                # 解析节点并添加到小部件列表
                widget = parse_func(option_element)
                self.widgets_list.append(widget)

    # 解析目标节点
    def __parse_target(self, target_element):
        # 获取标签文本
        label = _(target_element.getAttribute('label'))
        # 创建标签小部件
        label_widget = HIGEntryLabel(label)
        # 创建目标输入小部件
        target_widget = TargetEntry(self.ops)
        # 连接目标输入小部件的 "changed" 信号到更新目标方法
        target_widget.connect("changed", self.update_target)
        # 返回标签小部件和目标输入小部件
        return label_widget, target_widget
    # 解析选项列表元素，获取其子元素列表
    def __parse_option_list(self, option_list_element):
        children = option_list_element.getElementsByTagName('option')

        # 创建标签部件，显示选项列表的标签
        label_widget = HIGEntryLabel(
                _(option_list_element.getAttribute('label')))
        # 创建选项列表部件
        option_list_widget = OptionList(self.ops)

        # 遍历选项列表的子元素
        for child in children:
            # 获取选项、参数、标签等属性
            option = child.getAttribute('option')
            argument = child.getAttribute('argument')
            label = _(child.getAttribute('label'))
            # 将选项、参数、标签添加到选项列表部件中
            option_list_widget.append(option, argument, label)
            # 将选项和标签添加到帮助信息中
            self.profilehelp.add_label(option, label)
            # 将选项和短描述添加到帮助信息中
            self.profilehelp.add_shortdesc(
                    option, _(child.getAttribute('short_desc')))
            # 将选项和示例添加到帮助信息中
            self.profilehelp.add_example(
                    option, child.getAttribute('example'))

        # 更新选项列表部件
        option_list_widget.update()

        # 连接选项列表部件的"changed"信号到更新选项列表的方法
        option_list_widget.connect("changed", self.update_list_option)

        # 返回标签部件和选项列表部件
        return label_widget, option_list_widget
    # 解析选项检查，获取选项、标签、简短描述和示例
    def __parse_option_check(self, option_check):
        option = option_check.getAttribute('option')  # 获取选项
        label = _(option_check.getAttribute('label'))  # 获取标签
        short_desc = _(option_check.getAttribute('short_desc'))  # 获取简短描述
        example = option_check.getAttribute('example')  # 获取示例

        # 将标签、简短描述和示例添加到帮助信息中
        self.profilehelp.add_label(option, label)
        self.profilehelp.add_shortdesc(option, short_desc)
        self.profilehelp.add_example(option, example)

        # 创建选项检查对象
        check = OptionCheck(option, label)
        # 获取辅助部件
        auxiliary_widget = get_option_check_auxiliary_widget(
                option, self.ops, check)
        # 如果辅助部件不为空，则连接相应的信号
        if auxiliary_widget is not None:
            auxiliary_widget.connect("changed", self.update_auxiliary_widget)
            auxiliary_widget.connect(
                    'enter-notify-event', self.enter_notify_event_cb, option)
        # 否则，设置选项检查的激活状态
        else:
            check.set_active(not not self.ops[option])

        # 连接选项检查的信号
        check.connect('toggled', self.update_check, auxiliary_widget)
        check.connect('enter-notify-event', self.enter_notify_event_cb, option)

        # 返回选项检查对象和辅助部件
        return check, auxiliary_widget

    # 填充表格
    def fill_table(self, table, expand_fill=True):
        yopt = (0, Gtk.AttachOptions.EXPAND | Gtk.AttachOptions.FILL)[expand_fill]
        # 遍历部件列表，根据是否有辅助部件来决定填充方式
        for y, widget in enumerate(self.widgets_list):
            if widget[1] is None:
                table.attach(widget[0], 0, 2, y, y + 1, yoptions=yopt)
            else:
                table.attach(widget[0], 0, 1, y, y + 1, yoptions=yopt)
                table.attach(widget[1], 1, 2, y, y + 1, yoptions=yopt)

    # 更新辅助部件
    def update_auxiliary_widget(self, auxiliary_widget):
        self.update_command()

    # 更新部件
    def update(self):
        # 遍历部件列表，更新辅助部件或设置选项检查的激活状态
        for check, auxiliary_widget in self.widgets_list:
            if auxiliary_widget is not None:
                auxiliary_widget.update()
            else:
                check.set_active(not not self.ops[check.option])

    # 更新目标
    def update_target(self, entry):
        self.ops.target_specs = entry.get_targets()
        self.update_command()
    # 更新检查框的状态，并根据状态更新操作字典
    def update_check(self, check, auxiliary_widget):
        # 如果辅助小部件为空
        if auxiliary_widget is None:
            # 如果检查框被选中
            if check.get_active():
                # 将选项设置为True
                self.ops[check.option] = True
            else:
                # 将选项设置为False
                self.ops[check.option] = False
        # 更新命令
        self.update_command()
    
    # 更新列表选项的状态
    def update_list_option(self, widget):
        # 如果小部件有上次选择的值
        if widget.last_selected:
            # 将操作字典中上次选择的值设置为None
            self.ops[widget.last_selected] = None
    
        # 获取选中的选项、参数和标签
        opt, arg, label = widget.list[widget.get_active()]
        # 如果选项存在
        if opt:
            # 如果参数存在
            if arg:
                # 将选项设置为参数值
                self.ops[opt] = arg
            else:
                # 将选项设置为True
                self.ops[opt] = True
    
        # 将上次选择的值设置为当前选项
        widget.last_selected = opt
    
        # 显示选项的帮助信息
        self.show_help_for_option(opt)
    
        # 更新命令
        self.update_command()
    
    # 显示选项的帮助信息
    def show_help_for_option(self, option):
        # 调用profilehelp对象的处理方法
        self.profilehelp.handler(option)
        text = ""
        # 如果当前状态为“默认”
        if self.profilehelp.get_currentstate() == "Default":
            text = ""
        else:
            # 获取标签和简短描述
            text += self.profilehelp.get_label()
            text += "\n\n"
            text += self.profilehelp.get_shortdesc()
            # 如果有示例输入
            if self.profilehelp.get_example():
                text += "\n\nExample input:\n"
                text += self.profilehelp.get_example()
        # 设置帮助文本
        self.help_buf.set_text(text)
    
    # 进入通知事件回调
    def enter_notify_event_cb(self, event, widget, option):
        # 显示选项的帮助信息
        self.show_help_for_option(option)
class OptionBuilder(object):
    def __init__(self, xml_file, ops, update_func, help_buf):
        """
        xml_file is a UI description xml-file
        ops is an NmapOptions instance
        """
        # 打开 UI 描述的 XML 文件
        xml_desc = open(xml_file)
        # 解析 XML 文件内容
        self.xml = minidom.parse(xml_desc)
        # 关闭文件以避免文件描述符问题
        xml_desc.close()

        # 初始化实例变量
        self.ops = ops
        self.help_buf = help_buf
        self.update_func = update_func

        # 设置根标签
        self.root_tag = "interface"

        # 获取 XML 文件中的根标签
        self.xml = self.xml.getElementsByTagName(self.root_tag)[0]

        # 解析组信息
        self.groups = self.__parse_groups()
        # 解析部分名称
        self.section_names = self.__parse_section_names()
        # 解析选项卡
        self.tabs = self.__parse_tabs()

    # 更新方法
    def update(self):
        # 遍历所有选项卡并更新
        for tab in self.tabs.values():
            tab.update()

    # 解析部分名称的私有方法
    def __parse_section_names(self):
        dic = {}
        # 遍历组并获取标签属性
        for group in self.groups:
            grp = self.xml.getElementsByTagName(group)[0]
            dic[group] = grp.getAttribute('label')
        return dic

    # 解析组的私有方法
    def __parse_groups(self):
        # 获取所有组的名称
        return [g_name.getAttribute('name') for g_name in
                self.xml.getElementsByTagName('groups')[0].getElementsByTagName('group')]  # noqa

    # 解析选项卡的私有方法
    def __parse_tabs(self):
        dic = {}
        # 遍历所有组并创建选项卡对象
        for tab_name in self.groups:
            if tab_name != "Scripting":
                dic[tab_name] = OptionTab(
                        self.xml.getElementsByTagName(tab_name)[0], self.ops,
                        self.update_func, self.help_buf)
                dic[tab_name].notscripttab = True
            else:
                dic[tab_name] = ScriptInterface(
                        None, self.ops, self.update_func, self.help_buf)
        return dic


class OptionList(Gtk.ComboBox):
    # 这里是 OptionList 类的定义，暂时没有需要注释的代码
    # 初始化方法，接受参数 ops
    def __init__(self, ops):
        # 将参数 ops 赋值给实例变量 self.ops
        self.ops = ops
    
        # 创建一个包含三个字符串类型的列表存储对象，并赋值给实例变量 self.list
        self.list = Gtk.ListStore.new([str, str, str])
        # 调用父类 Gtk.ComboBox 的初始化方法，传入参数 model=self.list
        Gtk.ComboBox.__init__(self, model=self.list)
    
        # 创建一个文本单元格渲染器对象
        cell = Gtk.CellRendererText()
        # 将文本单元格渲染器对象添加到组合框中
        self.pack_start(cell, True)
        # 将文本单元格渲染器对象的 'text' 属性与列表的第二列绑定
        self.add_attribute(cell, 'text', 2)
    
        # 初始化实例变量 self.last_selected 为 None
        self.last_selected = None
        # 初始化实例变量 self.options 为空列表
        self.options = []
    
    # 更新方法
    def update(self):
        # 初始化选中索引为 0
        selected = 0
        # 遍历列表中的每一行
        for i, row in enumerate(self.list):
            # 获取每行的第一列和第二列的值
            opt, arg = row[0], row[1]
            # 如果选项为空，则跳过本次循环
            if opt == "":
                continue
            # 如果参数为空且 ops 中对应选项为真，或者参数不为空且 ops 中对应选项的字符串值等于参数值
            if ((not arg and self.ops[opt]) or
                    (arg and str(self.ops[opt]) == arg)):
                # 更新选中索引为当前行索引
                selected = i
        # 设置组合框的选中项为选中索引
        self.set_active(selected)
    
    # 添加方法，接受选项、参数和标签作为参数
    def append(self, option, argument, label):
        # 将标签赋值给 opt
        opt = label
        # 创建 NmapOptions 对象 ops
        ops = NmapOptions()
        # 如果选项不为空且不为 None
        if option is not None and option != "":
            # 如果参数不为空
            if argument:
                # 将选项和参数添加到 ops 对象中
                ops[option] = argument
            else:
                # 将选项添加到 ops 对象中
                ops[option] = True
            # 在标签后面添加 ops 对象的渲染结果（去掉第一个字符）
            opt += " (%s)" % join_quoted(ops.render()[1:])
    
        # 将选项、参数、标签组成的列表添加到列表存储对象中
        self.list.append([option, argument, opt])
        # 将选项添加到实例变量 self.options 中
        self.options.append(option)
# 定义一个名为 OptionCheck 的类，继承自 Gtk.CheckButton
class OptionCheck(Gtk.CheckButton):
    # 初始化方法，接受 option 和 label 两个参数
    def __init__(self, option, label):
        # 如果 option 不为空且不为 None，则在 label 后面加上 option 的内容
        opt = label
        if option is not None and option != "":
            opt += " (%s)" % option

        # 调用父类 Gtk.CheckButton 的初始化方法，传入参数 label=opt 和 use_underline=False
        Gtk.CheckButton.__init__(self, label=opt, use_underline=False)

        # 将 option 赋值给实例属性 option
        self.option = option
```