# `nmap\zenmap\zenmapGUI\ScriptInterface.py`

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
# 本模块负责处理“脚本”选项卡下的接口。

# 导入 gi 模块
import gi
# 要求使用特定版本的 Gtk
gi.require_version("Gtk", "3.0")
# 从 gi.repository 中导入 Gtk 和 GLib
from gi.repository import Gtk, GLib

# 导入 os 和 tempfile 模块
import os
import tempfile

# 防止加载 PyXML
# 将 xml 模块的 __path__ 属性设置为不包含 "_xmlplus" 的路径
import xml
xml.__path__ = [x for x in xml.__path__ if "_xmlplus" not in x]

# 导入 xml.sax 模块
import xml.sax

# 从 zenmapGUI.higwidgets.higboxes 中导入 HIGVBox 和 HIGHBox
# 从 zenmapGUI.higwidgets.higscrollers 中导入 HIGScrolledWindow
# 从 zenmapGUI.higwidgets.higbuttons 中导入 HIGButton
from zenmapGUI.higwidgets.higboxes import HIGVBox, HIGHBox
from zenmapGUI.higwidgets.higscrollers import HIGScrolledWindow
from zenmapGUI.higwidgets.higbuttons import HIGButton

# 从 zenmapCore.ScriptMetadata 中导入 get_script_entries
# 从 zenmapCore.ScriptArgsParser 中导入 parse_script_args_dict
# 从 zenmapCore.NmapCommand 中导入 NmapCommand
# 从 zenmapCore.NmapOptions 中导入 NmapOptions
# 导入 zenmapCore.NSEDocParser 模块
import zenmapCore.NSEDocParser
# 导入 zenmapGUI.FileChoosers 模块
import zenmapGUI.FileChoosers
# 从 zenmapCore.UmitConf 中导入 PathsConfig
from zenmapCore.UmitConf import PathsConfig
# 从 zenmapCore.UmitLogging 模块中导入 log 函数
from zenmapCore.UmitLogging import log
# 从 zenmapCore.Name 模块中导入 APP_NAME 变量
from zenmapCore.Name import APP_NAME

# 创建 PathsConfig 对象
paths_config = PathsConfig()

# 定义函数 text_buffer_insert_nsedoc，用于在缓冲区末尾插入 NSEDoc，并将标记转换为适当的标签
def text_buffer_insert_nsedoc(buf, nsedoc):
    if not buf.get_tag_table().lookup("NSEDOC_CODE_TAG"):
        # 如果标签表中不存在 NSEDOC_CODE_TAG 标签，则创建该标签
        buf.create_tag("NSEDOC_CODE_TAG", font="Monospace")
    # 遍历 NSEDocParser.nsedoc_parse(nsedoc) 返回的事件
    for event in zenmapCore.NSEDocParser.nsedoc_parse(nsedoc):
        if event.type == "paragraph_start":
            buf.insert(buf.get_end_iter(), "\n")
        elif event.type == "paragraph_end":
            buf.insert(buf.get_end_iter(), "\n")
        elif event.type == "list_start":
            buf.insert(buf.get_end_iter(), "\n")
        elif event.type == "list_end":
            pass
        elif event.type == "list_item_start":
            buf.insert(buf.get_end_iter(), "\u2022\u00a0")  # 插入列表项的标志
        elif event.type == "list_item_end":
            buf.insert(buf.get_end_iter(), "\n")
        elif event.type == "text":
            buf.insert(buf.get_end_iter(), event.text)
        elif event.type == "code":
            # 插入带有 NSEDOC_CODE_TAG 标签的文本
            buf.insert_with_tags_by_name(
                    buf.get_end_iter(), event.text, "NSEDOC_CODE_TAG")

# 定义类 ScriptHelpXMLContentHandler，用于解析 --script-help XML 输出
class ScriptHelpXMLContentHandler (xml.sax.handler.ContentHandler):
    def __init__(self):
        xml.sax.handler.ContentHandler.__init__(self)
        # 初始化脚本文件名列表和脚本目录
        self.script_filenames = []
        self.scripts_dir = None
        self.nselib_dir = None
    # 定义处理 XML 元素的方法，接受元素名和属性作为参数
    def startElement(self, name, attrs):
        # 如果元素名为 "directory"
        if name == "directory":
            # 如果属性中没有 "name"，则抛出数值错误
            if "name" not in attrs:
                raise ValueError(
                        '"directory" element did not have "name" attribute')
            # 获取目录名
            dirname = attrs["name"]
            # 如果属性中没有 "path"，则抛出数值错误
            if "path" not in attrs:
                raise ValueError(
                        '"directory" element did not have "path" attribute')
            # 获取路径
            path = attrs["path"]
            # 如果目录名为 "scripts"，则设置脚本目录为路径
            if dirname == "scripts":
                self.scripts_dir = path
            # 如果目录名为 "nselib"，则设置 nselib 目录为路径
            elif dirname == "nselib":
                self.nselib_dir = path
            else:
                # 否则忽略
                # Ignore.
                pass
        # 如果元素名为 "script"
        elif name == "script":
            # 如果属性中没有 "filename"，则抛出数值错误
            if "filename" not in attrs:
                raise ValueError(
                        '"script" element did not have "filename" attribute')
            # 将脚本文件名添加到列表中
            self.script_filenames.append(attrs["filename"])

    # 静态方法，用于解析 nmap 脚本的帮助信息
    @staticmethod
    def parse_nmap_script_help(f):
        # 创建 XML 解析器
        parser = xml.sax.make_parser()
        # 创建自定义的 XML 内容处理器
        handler = ScriptHelpXMLContentHandler()
        # 设置 XML 解析器的内容处理器
        parser.setContentHandler(handler)
        # 解析 XML 文件
        parser.parse(f)
        # 返回处理器
        return handler
# 定义一个名为 ScriptInterface 的类
class ScriptInterface:
    # 用户停止输入后，从 --script 更新界面的超时时间，单位为毫秒
    SCRIPT_LIST_DELAY = 500
    # 在轮询 Nmap 子进程之间的超时时间，单位为毫秒
    NMAP_DELAY = 200

    # 定义一个名为 get_script_list 的方法，接受规则和回调函数作为参数
    def get_script_list(self, rules, callback):
        """Start an Nmap subprocess in the background with
        "--script-help=<rules> -oX -", and set it up to call the given callback
        when finished."""
        # 创建 NmapOptions 对象
        ops = NmapOptions()
        # 设置 Nmap 可执行文件路径
        ops.executable = paths_config.nmap_command_path
        # 设置 Nmap 命令行参数 "--script-help"
        ops["--script-help"] = rules
        # 设置 Nmap 命令行参数 "-oX"
        ops["-oX"] = "-"
        # 生成 Nmap 命令行字符串
        command_string = ops.render_string()
        # 创建临时文件用于存储标准错误输出
        stderr = tempfile.TemporaryFile(
                mode="r", prefix=APP_NAME + "-script-help-stderr-")
        # 输出调试信息
        log.debug("Script interface: running %s" % repr(command_string))
        # 创建 NmapCommand 对象
        nmap_process = NmapCommand(command_string)
        try:
            # 运行 Nmap 扫描，并将标准错误输出重定向到临时文件
            nmap_process.run_scan(stderr=stderr)
        except Exception as e:
            # 如果出现异常，调用回调函数并关闭临时文件
            callback(False, None)
            stderr.close()
            return
        # 关闭临时文件
        stderr.close()

        # 禁用脚本列表小部件
        self.script_list_widget.set_sensitive(False)

        # 添加定时器，用于轮询 Nmap 子进程，并调用回调函数
        GLib.timeout_add(
                self.NMAP_DELAY, self.script_list_timer_callback,
                nmap_process, callback)
    # 定义一个函数，用于定时检查脚本列表的状态并调用回调函数
    def script_list_timer_callback(self, process, callback):
        # 尝试获取进程的扫描状态
        try:
            status = process.scan_state()
        except Exception:
            status = None
        # 记录调试信息
        log.debug("Script interface: script_list_timer_callback %s" %
                repr(status))

        # 如果状态为 True，表示仍在运行，需要再次调度定时器来检查
        if status is True:
            return True

        # 设置脚本列表部件为可用状态
        self.script_list_widget.set_sensitive(True)

        # 如果状态为 False，表示成功完成
        if status is False:
            # 调用回调函数，传递 True 和进程对象
            callback(True, process)
        else:
            # 如果状态为其他值，表示出现错误
            # 调用回调函数，传递 False 和进程对象
            callback(False, process)

    # 定义一个回调函数，用于处理初始脚本列表的状态
    def initial_script_list_cb(self, status, process):
        # 记录调试信息
        log.debug("Script interface: initial_script_list_cb %s" % repr(status))
        # 移除脚本列表容器中的所有子部件
        for child in self.script_list_container.get_children():
            self.script_list_container.remove(child)
        # 如果状态为 True 并且成功处理了初始脚本列表的输出
        if status and self.handle_initial_script_list_output(process):
            # 将脚本列表部件添加到脚本列表容器中
            self.script_list_container.pack_start(self.script_list_widget, True, True, 0)
        else:
            # 否则，将 Nmap 错误部件添加到脚本列表容器中
            self.script_list_container.pack_start(self.nmap_error_widget, True, True, 0)
    # 处理初始脚本列表输出的方法
    def handle_initial_script_list_output(self, process):
        # 将文件指针移动到文件开头
        process.stdout_file.seek(0)
        try:
            # 解析 nmap 脚本帮助的 XML 内容
            handler = ScriptHelpXMLContentHandler.parse_nmap_script_help(
                    process.stdout_file)
        except (ValueError, xml.sax.SAXParseException) as e:
            # 如果解析出现异常，记录错误信息并返回 False
            log.debug("--script-help parse exception: %s" % str(e))
            return False

        # 检查是否有脚本输出；如果没有，可能是 Nmap 版本过旧
        if len(handler.script_filenames) == 0:
            return False

        # 如果没有脚本目录，记录错误信息并返回 False
        if not handler.scripts_dir:
            log.debug("--script-help error: no scripts directory")
            return False
        # 如果没有 nselib 目录，记录错误信息并返回 False
        if not handler.nselib_dir:
            log.debug("--script-help error: no nselib directory")
            return False

        # 记录脚本接口的脚本目录信息
        log.debug("Script interface: scripts dir %s" % repr(
            handler.scripts_dir))
        # 记录脚本接口的 nselib 目录信息
        log.debug("Script interface: nselib dir %s" % repr(handler.nselib_dir))

        # 创建脚本元数据条目的字典
        entries = {}
        # 遍历脚本条目，将文件名作为键，条目对象作为值存入字典
        for entry in get_script_entries(
                handler.scripts_dir, handler.nselib_dir):
            entries[entry.filename] = entry

        # 清空列表存储
        self.liststore.clear()
        # 遍历脚本文件名列表
        for filename in handler.script_filenames:
            # 获取文件名的基本名称
            basename = os.path.basename(filename)
            # 根据基本名称从字典中获取条目对象
            entry = entries.get(basename)
            if entry:
                # 如果条目对象存在，将脚本 ID、False 和条目对象添加到列表存储中
                script_id = self.strip_file_name(basename)
                self.liststore.append([script_id, False, entry])
            else:
                # 如果找不到脚本元数据，将文件名和 False 添加到文件列表存储中
                self.file_liststore.append([filename, False])

        # 现在确定哪些脚本被选中
        self.update_script_list_from_spec(self.ops["--script"])
        return True
    # 从规范中更新脚本列表的回调方法
    def update_script_list_from_spec(self, spec):
        """Callback method for user edit delay."""
        # 记录调试信息
        log.debug("Script interface: update_script_list_from_spec %s" % repr(spec))
        # 如果规范存在，则获取脚本列表
        if spec:
            self.get_script_list(spec, self.update_script_list_cb)
        else:
            # 否则刷新脚本列表为空
            self.refresh_list_scripts([])

    # 更新脚本列表的回调方法
    def update_script_list_cb(self, status, process):
        # 记录调试信息
        log.debug("Script interface: update_script_list_cb %s" % repr(status))
        # 如果状态为真，则处理更新脚本列表的输出
        if status:
            self.handle_update_script_list_output(process)
        else:
            # 否则刷新脚本列表为空
            self.refresh_list_scripts([])

    # 处理更新脚本列表的输出
    def handle_update_script_list_output(self, process):
        # 将进程的标准输出文件指针移动到开头
        process.stdout_file.seek(0)
        try:
            # 尝试解析 Nmap 脚本帮助的 XML 内容
            handler = ScriptHelpXMLContentHandler.parse_nmap_script_help(
                    process.stdout_file)
        except (ValueError, xml.sax.SAXParseException) as e:
            # 如果出现异常，则记录调试信息并返回假
            log.debug("--script-help parse exception: %s" % str(e))
            return False

        # 刷新脚本列表
        self.refresh_list_scripts(handler.script_filenames)

    # 获取主 Hbox 到 ProfileEditor 的方法
    def get_hmain_box(self):
        """Returns main Hbox to ProfileEditor."""
        return self.hmainbox

    # 更新命令输入时的接口
    def update(self):
        """Updates the interface when the command entry is changed."""
        # 更新脚本列表
        rules = self.ops["--script"]
        if (self.prev_script_spec != rules):
            self.renew_script_list_timer(rules)
        self.prev_script_spec = rules
        # 更新参数
        raw_argument = self.ops["--script-args"]
        if raw_argument is not None:
            self.parse_script_args(raw_argument)
        self.arg_liststore.clear()
        for arg in self.current_arguments:
            arg_name, arg_desc = arg
            value = self.arg_values.get(arg_name)
            if not value:
                self.arg_liststore.append([arg_name, None, arg])
            else:
                self.arg_liststore.append([arg_name, value, arg])
    # 重新启动定时器以更新脚本列表，当用户编辑命令时。因为更新脚本列表是一个昂贵的操作，涉及到创建一个子进程，所以我们不会在每次输入一个字符时都执行。
    def renew_script_list_timer(self, spec):
        if self.script_list_timeout_id:
            # 移除之前的定时器
            GLib.source_remove(self.script_list_timeout_id)
        # 添加一个新的定时器，用于在一定延迟后执行更新脚本列表的操作
        self.script_list_timeout_id = GLib.timeout_add(
                self.SCRIPT_LIST_DELAY,
                self.update_script_list_from_spec, spec)

    # 当命令行被编辑时，调用此函数来根据--script-args的值更新脚本参数的显示
    def parse_script_args(self, raw_argument):
        # 解析原始参数字符串，将其转换为参数字典
        arg_dict = parse_script_args_dict(raw_argument)
        if arg_dict is None:  # 如果解析出错，args_dict 为 None
            # 清空参数值
            self.arg_values.clear()
        else:
            # 更新参数值
            for key in arg_dict.keys():
                self.arg_values[key] = arg_dict[key]

    # 当脚本标签页启动时，更新参数值
    def update_argument_values(self, raw_argument):
        if raw_argument is not None:
            # 调用 parse_script_args 函数更新参数值
            self.parse_script_args(raw_argument)

    # 设置要显示的帮助文本
    def set_help_texts(self):
        self.list_scripts_help = _("""List of scripts
# 列出所有已安装脚本的列表。通过点击脚本名称旁边的框来激活或停用脚本
A list of all installed scripts. Activate or deactivate a script \
by clicking the box next to the script name.

# 描述
# 此框显示脚本所属的类别。此外，它提供了脚本的详细描述，该描述位于脚本中。URL 指向在线 NSEDoc 文档。
Description

This box shows the categories a script belongs to. In addition, it gives a \
detailed description of the script which is present in script. A URL points \
to online NSEDoc documentation.

# 参数
# 影响所选脚本的参数列表。通过点击参数名称旁边的值字段来输入值。
Arguments

A list of arguments that affect the selected script. Enter a value by \
clicking in the value field beside the argument name.

# 创建“请等待”小部件
def make_please_wait_widget(self):
    # 创建垂直方向的 Gtk.Box
    vbox = Gtk.Box.new(Gtk.Orientation.VERTICAL, 0)
    # 创建标签并设置文本为“Please wait.”
    label = Gtk.Label.new(_("Please wait."))
    # 设置标签自动换行
    label.set_line_wrap(True)
    # 将标签添加到垂直 Box 中
    vbox.pack_start(label, True, True, 0)
    # 返回垂直 Box
    return vbox

# 刷新脚本列表
def refresh_list_scripts(self, selected_scripts):
    """The list of selected scripts is refreshed in the list store."""
    # 将列表存储中所有行的第二列设置为 False
    for row in self.liststore:
        row[1] = False
    # 将文件列表存储中所有行的第二列设置为 False
    for row in self.file_liststore:
        row[1] = False
    # 遍历选定的脚本列表
    for filename in selected_scripts:
        # 遍历列表存储中的行
        for row in self.liststore:
            # 如果行的第一列等于文件名的基本名称，则将第二列设置为 True，并跳出循环
            if row[0] == self.strip_file_name(os.path.basename(filename)):
                row[1] = True
                break
        else:
            # 如果未找到匹配的行，则遍历文件列表存储中的行
            for row in self.file_liststore:
                # 如果行的第一列等于文件名，则将第二列设置为 True，并跳出循环
                if row[0] == filename:
                    row[1] = True
                    break
            else:
                # 如果未找到匹配的行，则将文件名和 True 添加到文件列表存储中
                self.file_liststore.append([filename, True])

# 去除文件名的“.nse”扩展名（如果存在）
def strip_file_name(self, filename):
    """Removes a ".nse" extension from filename if present."""
    # 如果文件名以“.nse”结尾，则去除后缀并返回
    if(filename.endswith(".nse")):
        return filename[:-4]
    else:
        # 否则直接返回文件名
        return filename
    # 从选定的脚本中设置脚本
    def set_script_from_selection(self):
        # 创建一个空列表来存储脚本名称
        scriptsname = []
        # 遍历 self.liststore 中的条目
        for entry in self.liststore:
            # 如果条目的第二个元素为真
            if entry[1]:
                # 将条目的文件名添加到脚本名称列表中
                scriptsname.append(self.strip_file_name(entry[0]))
        # 遍历 self.file_liststore 中的条目
        for entry in self.file_liststore:
            # 如果条目的第二个元素为真
            if entry[1]:
                # 将条目的文件名添加到脚本名称列表中
                scriptsname.append(entry[0])
        # 如果脚本名称列表为空
        if len(scriptsname) == 0:
            # 将 self.ops["--script"] 设置为 None
            self.ops["--script"] = None
        else:
            # 将 self.ops["--script"] 设置为脚本名称列表的逗号分隔字符串
            self.ops["--script"] = ",".join(scriptsname)
        # 更新命令
        self.update_command()

    # 切换回调方法，当脚本列表中的复选框被切换时调用
    def toggled_cb(self, cell, path, model):
        """Callback method, called when the check box in list of scripts is
        toggled."""
        # 切换条目的第二个元素的值
        model[path][1] = not model[path][1]
        # 从选定的脚本中设置脚本
        self.set_script_from_selection()

    # 创建并打包与显示描述框相关的小部件
    def make_description_widget(self):
        """Creates and packs widgets related to displaying the description
        box."""
        # 创建一个可滚动的窗口
        sw = HIGScrolledWindow()
        # 设置滚动窗口的策略
        sw.set_policy(Gtk.PolicyType.AUTOMATIC, Gtk.PolicyType.ALWAYS)
        # 设置滚动窗口的阴影类型
        sw.set_shadow_type(Gtk.ShadowType.OUT)
        # 设置滚动窗口的边框宽度
        sw.set_border_width(5)
        # 创建一个文本视图
        text_view = Gtk.TextView()
        # 连接 "enter-notify-event" 信号到更新帮助描述回调方法
        text_view.connect("enter-notify-event", self.update_help_desc_cb)
        # 获取文本缓冲区
        self.text_buffer = text_view.get_buffer()
        # 创建标签 "Usage" 和 "Output"
        self.text_buffer.create_tag("Usage", font="Monospace")
        self.text_buffer.create_tag("Output", font="Monospace")
        # 设置文本视图的换行模式
        text_view.set_wrap_mode(Gtk.WrapMode.WORD)
        # 设置文本视图不可编辑
        text_view.set_editable(False)
        # 设置文本视图的对齐方式
        text_view.set_justification(Gtk.Justification.LEFT)
        # 将文本视图添加到滚动窗口中
        sw.add(text_view)
        # 返回滚动窗口
        return sw
    # 创建并打包与参数框相关的小部件
    def make_arguments_widget(self):
        # 创建垂直方向的 Gtk.Box 容器
        vbox = Gtk.Box.new(Gtk.Orientation.VERTICAL, 0)
        # 在垂直容器中添加一个标签，显示 "Arguments"
        vbox.pack_start(Gtk.Label.new(_("Arguments")), False, False, 0)
        # 创建一个可滚动的窗口
        arg_window = HIGScrolledWindow()
        # 设置滚动窗口的滚动策略
        arg_window.set_policy(Gtk.PolicyType.AUTOMATIC, Gtk.PolicyType.ALWAYS)
        # 设置滚动窗口的阴影类型
        arg_window.set_shadow_type(Gtk.ShadowType.OUT)

        # 创建一个带有模型的 Gtk.TreeView
        arg_listview = Gtk.TreeView.new_with_model(self.arg_liststore)
        # 连接 "motion-notify-event" 信号到更新帮助参数的回调函数
        arg_listview.connect("motion-notify-event", self.update_help_arg_cb)
        # 创建一个文本单元格渲染器
        argument = Gtk.CellRendererText()
        self.value = Gtk.CellRendererText()
        # 连接 "edited" 信号到值编辑的回调函数，并传递参数列表存储模型
        self.value.connect("edited", self.value_edited_cb, self.arg_liststore)
        # 创建两个列，一个用于参数，一个用于值
        arg_col = Gtk.TreeViewColumn(title="Arguments\t")
        val_col = Gtk.TreeViewColumn(title="values")
        # 将两列添加到树视图中
        arg_listview.append_column(arg_col)
        arg_listview.append_column(val_col)
        # 在参数列中添加参数单元格渲染器，并设置属性为文本
        arg_col.pack_start(argument, True)
        arg_col.add_attribute(argument, "text", 0)
        # 在值列中添加值单元格渲染器，并设置属性为文本
        val_col.pack_start(self.value, True)
        val_col.add_attribute(self.value, "text", 1)

        # 将树视图添加到参数窗口中
        arg_window.add(arg_listview)
        # 将参数窗口添加到垂直容器中
        vbox.pack_start(arg_window, True, True, 0)

        # 返回垂直容器
        return vbox

    # 当参数单元格被编辑时调用的回调函数
    def value_edited_cb(self, cell, path, new_text, model):
        # 清空参数列表
        self.arg_list = []
        # 更新模型中指定路径的值为新文本
        model[path][1] = new_text
        # 获取参数名
        argument_name = model[path][0]
        # 更新参数值字典
        self.arg_values[argument_name] = new_text
        # 更新参数值
        self.update_arg_values()
    # 更新参数值时，相应地更新命令行
    def update_arg_values(self):
        # 遍历参数值的键
        for key in self.arg_values.keys():
            # 如果参数值为空，则删除该键
            if len(self.arg_values[key]) == 0:
                del self.arg_values[key]
            else:
                # 否则将参数名和参数值添加到参数列表中
                self.arg_list.append(key + "=" + self.arg_values[key])
        # 如果参数列表为空，则将命令行参数设置为 None，并清空参数值
        if len(self.arg_list) == 0:
            self.ops["--script-args"] = None
            self.arg_values.clear()
        else:
            # 否则将参数列表连接成字符串，作为命令行参数
            self.ops["--script-args"] = ",".join(self.arg_list)
        # 更新命令行
        self.update_command()
    
    # 当脚本列表被选中时调用的回调函数
    def selection_changed_cb(self, selection):
        model, selection = selection.get_selected_rows()
        # 遍历选中的脚本
        for path in selection:
            # 获取选中脚本的描述并设置
            entry = model.get_value(model.get_iter(path), 2)
            self.set_description(entry)
            # 填充参数列表
            self.populate_arg_list(entry)
            # 记住当前指向的脚本条目
            self.focusedentry = entry
    
    # 更新脚本列表帮助信息的回调方法
    def update_help_ls_cb(self, widget, extra):  # list of scripts
        # 设置帮助文本为脚本列表的帮助信息
        self.help_buf.set_text(self.list_scripts_help)
    
    # 更新描述帮助信息的回调方法
    def update_help_desc_cb(self, widget, extra):
        # 设置帮助文本为描述的帮助信息
        self.help_buf.set_text(self.description_help)
    # 更新帮助参数回调方法，用于显示参数帮助
    def update_help_arg_cb(self, treeview, event):
        # 获取鼠标指针位置
        wx, wy = treeview.get_pointer()
        # 将鼠标指针位置转换为二进制窗口坐标
        x, y = treeview.convert_widget_to_bin_window_coords(wx, wy)
        # 获取指定位置的路径
        path = treeview.get_path_at_pos(x, y)
        # 如果路径不存在或者没有焦点输入，则清空帮助文本并返回
        if not path or not self.focusedentry:
            self.help_buf.set_text("")
            return
        # 获取选中的模型和值
        path = path[0]
        model, selected = treeview.get_selection().get_selected()
        # 获取参数名和参数描述
        arg_name, arg_desc = model.get_value(model.get_iter(path), 2)
        # 如果参数描述不为空，则清空帮助文本并插入参数名和参数描述
        if arg_desc is not None:
            self.help_buf.set_text("")
            self.help_buf.insert(
                    self.help_buf.get_end_iter(), text="%s\n" % arg_name)
            text_buffer_insert_nsedoc(self.help_buf, arg_desc)
        # 否则，清空帮助文本
        else:
            self.help_buf.set_text("")

    # 添加文件按钮点击回调方法
    def add_file_button_clicked_cb(self, button):
        # 如果脚本文件选择器为空，则创建一个新的脚本文件选择器对话框
        if self.script_file_chooser is None:
            self.script_file_chooser = \
                    zenmapGUI.FileChoosers.ScriptFileChooserDialog(
                            title=_("Select script files"))
        # 运行脚本文件选择器对话框
        response = self.script_file_chooser.run()
        # 获取选择的文件名
        filenames = self.script_file_chooser.get_filenames()
        # 隐藏脚本文件选择器对话框
        self.script_file_chooser.hide()
        # 如果响应不是 OK，则返回
        if response != Gtk.ResponseType.OK:
            return
        # 遍历文件名列表，将文件名和 True 添加到文件列表存储中
        for filename in filenames:
            self.file_liststore.append([filename, True])
        # 如果文件列表存储中有文件，则显示文件滚动窗口并启用移除文件按钮
        if len(self.file_liststore) > 0:
            self.file_scrolled_window.show()
            self.remove_file_button.set_sensitive(True)
        # 根据选择设置脚本
        self.set_script_from_selection()
    # 当移除文件按钮被点击时的回调函数
    def remove_file_button_clicked_cb(self, button):
        # 获取文件列表视图的选择
        selection = self.file_listview.get_selection()
        # 获取选择的行
        model, selection = selection.get_selected_rows()
        # 遍历选择的行
        for path in selection:
            # 从文件列表中移除选中的行
            self.file_liststore.remove(model.get_iter(path))
        # 如果文件列表为空
        if len(self.file_liststore) == 0:
            # 隐藏文件滚动窗口
            self.file_scrolled_window.hide()
            # 禁用移除文件按钮
            self.remove_file_button.set_sensitive(False)
        # 根据选择设置脚本
        self.set_script_from_selection()
    
    # 设置描述内容的方法
    def set_description(self, entry):
        """Sets the content that is to be displayed in the description box."""
        # 清空文本缓冲区
        self.text_buffer.set_text("")
    
        # 插入内容到文本缓冲区
        self.text_buffer.insert(self.text_buffer.get_end_iter(), """\
# 将 %(cats)s 替换为 entry.categories 中的所有元素，并以逗号分隔的形式插入到文本缓冲区中
""" % {"cats": ", ".join(entry.categories)})
# 将 entry.description 插入到文本缓冲区中
        text_buffer_insert_nsedoc(self.text_buffer, entry.description)
# 如果 entry.usage 存在，则在文本缓冲区中插入 "Usage"，并使用 "Usage" 标签插入 entry.usage
        if entry.usage:
            self.text_buffer.insert(
                    self.text_buffer.get_end_iter(), "\nUsage\n")
            self.text_buffer.insert_with_tags_by_name(
                    self.text_buffer.get_end_iter(), entry.usage, "Usage")
# 如果 entry.output 存在，则在文本缓冲区中插入 "Output"，并使用 "Output" 标签插入 entry.output
        if entry.output:
            self.text_buffer.insert(
                    self.text_buffer.get_end_iter(), "\nOutput\n")
            self.text_buffer.insert_with_tags_by_name(
                    self.text_buffer.get_end_iter(), entry.output, "Output")
# 如果 entry.url 存在，则在文本缓冲区中插入 entry.url
        if entry.url:
            self.text_buffer.insert(
                    self.text_buffer.get_end_iter(), "\n" + entry.url)

# 当悬停在特定脚本上时，调用此函数以显示其参数和值（如果有）
    def populate_arg_list(self, entry):
        # 清空参数列表存储
        self.arg_liststore.clear()
        # 重置当前参数列表和值
        self.current_arguments = []
        self.value.set_property('editable', True)
        # 遍历 entry.arguments 中的参数和描述
        for arg in entry.arguments:
            arg_name, arg_desc = arg
            self.current_arguments.append(arg)
            # 获取参数值
            value = self.arg_values.get(arg_name)
            # 如果值不存在，则将参数名和描述插入到参数列表存储中
            if not value:
                self.arg_liststore.append([arg_name, None, arg])
            # 如果值存在，则将参数名、值和描述插入到参数列表存储中
            else:
                self.arg_liststore.append([arg_name, value, arg])
```