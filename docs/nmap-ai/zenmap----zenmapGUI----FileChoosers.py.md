# `nmap\zenmap\zenmapGUI\FileChoosers.py`

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

# 导入 os.path 和 sys 模块
import os.path
import sys

# 导入 zenmapCore.I18N 模块，用于国际化
import zenmapCore.I18N  # lgtm[py/unused-import]

# 定义常量 RESPONSE_OPEN_DIRECTORY 的值为 1
RESPONSE_OPEN_DIRECTORY = 1

# 定义 AllFilesFileFilter 类，继承自 Gtk.FileFilter
class AllFilesFileFilter(Gtk.FileFilter):
    # 初始化方法
    def __init__(self):
        # 调用父类的初始化方法
        Gtk.FileFilter.__init__(self)
        # 设置匹配模式为所有文件
        pattern = "*"
        self.add_pattern(pattern)
        # 设置过滤器名称为 "All files (*.)"
        self.set_name(_("All files (%s)") % pattern)

# 定义 ResultsFileFilter 类，继承自 Gtk.FileFilter
class ResultsFileFilter(Gtk.FileFilter):
    # 初始化方法
    def __init__(self):
        # 调用父类的初始化方法
        Gtk.FileFilter.__init__(self)
        # 设置匹配模式为 "*.xml"
        patterns = ["*.xml"]
        for pattern in patterns:
            self.add_pattern(pattern)
        # 设置过滤器名称为 "Nmap XML files (*.xml)"
        self.set_name(_("Nmap XML files (%s)") % ", ".join(patterns))

# 定义 ScriptFileFilter 类，继承自 Gtk.FileFilter
class ScriptFileFilter(Gtk.FileFilter):
    # 初始化函数，用于创建一个新的对象
    def __init__(self):
        # 调用父类的初始化函数
        Gtk.FileFilter.__init__(self)
        # 定义文件名匹配模式列表
        patterns = ["*.nse"]
        # 遍历文件名匹配模式列表
        for pattern in patterns:
            # 将每个匹配模式添加到过滤器中
            self.add_pattern(pattern)
        # 设置过滤器的名称，包括匹配模式列表中的内容
        self.set_name(_("NSE scripts (%s)") % ", ".join(patterns))
# 创建一个自定义的文件选择对话框类，用于选择所有类型的文件
class AllFilesFileChooserDialog(Gtk.FileChooserDialog):
    def __init__(self, title="", parent=None,
                 action=Gtk.FileChooserAction.OPEN,
                 buttons=(Gtk.STOCK_CANCEL, Gtk.ResponseType.CANCEL,
                          Gtk.STOCK_OPEN, Gtk.ResponseType.OK), backend=None):

        # 调用父类的初始化方法，设置对话框的标题、父窗口、操作类型和按钮
        Gtk.FileChooserDialog.__init__(self, title=title, parent=parent,
                                          action=action, buttons=buttons)
        # 设置默认的响应类型为 OK
        self.set_default_response(Gtk.ResponseType.OK)
        # 添加一个过滤器，用于过滤所有类型的文件
        self.add_filter(AllFilesFileFilter())


# 创建一个自定义的文件选择对话框类，用于选择单个结果文件
class ResultsFileSingleChooserDialog(Gtk.FileChooserDialog):
    """This results file choose only allows the selection of single files, not
    directories."""
    def __init__(self, title="", parent=None,
                 action=Gtk.FileChooserAction.OPEN,
                 buttons=(Gtk.STOCK_CANCEL, Gtk.ResponseType.CANCEL,
                          Gtk.STOCK_OPEN, Gtk.ResponseType.OK), backend=None):

        # 调用父类的初始化方法，设置对话框的标题、父窗口、操作类型和按钮
        Gtk.FileChooserDialog.__init__(self, title=title, parent=parent,
                                          action=action, buttons=buttons)
        # 设置默认的响应类型为 OK
        self.set_default_response(Gtk.ResponseType.OK)
        # 添加两个过滤器，一个用于结果文件，一个用于所有类型的文件
        for f in (ResultsFileFilter(), AllFilesFileFilter()):
            self.add_filter(f)


# 创建一个自定义的文件选择对话框类，用于选择结果文件或目录
class ResultsFileChooserDialog(Gtk.FileChooserDialog):
    def __init__(self, title="", parent=None,
                 action=Gtk.FileChooserAction.OPEN,
                 buttons=(Gtk.STOCK_CANCEL, Gtk.ResponseType.CANCEL,
                          "Open Directory", RESPONSE_OPEN_DIRECTORY,
                          Gtk.STOCK_OPEN, Gtk.ResponseType.OK), backend=None):

        # 调用父类的初始化方法，设置对话框的标题、父窗口、操作类型和按钮
        Gtk.FileChooserDialog.__init__(self, title=title, parent=parent,
                                          action=action, buttons=buttons)
        # 设置默认的响应类型为 OK
        self.set_default_response(Gtk.ResponseType.OK)
        # 添加两个过滤器，一个用于结果文件，一个用于所有类型的文件
        for f in (ResultsFileFilter(), AllFilesFileFilter()):
            self.add_filter(f)


# 创建一个自定义的文件选择对话框类，用于选择脚本文件
class ScriptFileChooserDialog(Gtk.FileChooserDialog):
    # 初始化函数，用于创建文件选择对话框
    def __init__(self, title="", parent=None,
                     action=Gtk.FileChooserAction.OPEN,
                     buttons=(Gtk.STOCK_CANCEL, Gtk.ResponseType.CANCEL,
                              Gtk.STOCK_OPEN, Gtk.ResponseType.OK), backend=None):
        # 调用父类的初始化函数，设置对话框的标题、父窗口、操作类型和按钮
        Gtk.FileChooserDialog.__init__(self, title=title, parent=parent,
                                              action=action, buttons=buttons)
        # 设置默认的响应类型为 OK
        self.set_default_response(Gtk.ResponseType.OK)
        # 允许选择多个文件
        self.set_select_multiple(True)
        # 添加两种文件过滤器：ScriptFileFilter 和 AllFilesFileFilter
        for f in (ScriptFileFilter(), AllFilesFileFilter()):
            self.add_filter(f)
class SaveResultsFileChooserDialog(Gtk.FileChooserDialog):
    # 定义文件类型选项
    TYPES = (
        (_("By extension"), None, None),  # 按扩展名选择
        (_("Nmap XML format (.xml)"), "xml", ".xml"),  # Nmap XML 格式
        (_("Nmap text format (.nmap)"), "text", ".nmap"),  # Nmap 文本格式
    )
    # 为“按扩展名选择”准备的扩展名字典
    EXTENSIONS = {
        ".xml": "xml",
        ".nmap": "text",
        ".txt": "text",
    }

    def __init__(self, title="", parent=None,
                 action=Gtk.FileChooserAction.SAVE,
                 buttons=(Gtk.STOCK_CANCEL, Gtk.ResponseType.CANCEL,
                          Gtk.STOCK_SAVE, Gtk.ResponseType.OK), backend=None):
        # 调用父类的初始化方法
        Gtk.FileChooserDialog.__init__(self, title=title, parent=parent,
                                          action=action, buttons=buttons)

        # 创建文件类型选项的列表存储
        types_store = Gtk.ListStore.new([str, str, str])
        # 将文件类型选项添加到列表存储中
        for type in self.TYPES:
            types_store.append(type)

        # 创建下拉框并绑定列表存储
        self.combo = Gtk.ComboBox.new_with_model(types_store)
        cell = Gtk.CellRendererText()
        self.combo.pack_start(cell, True)
        self.combo.add_attribute(cell, "text", 0)
        self.combo.connect("changed", self.combo_changed_cb)  # 绑定下拉框变化事件
        self.combo.set_active(1)  # 设置默认选中项

        # 创建水平布局盒子
        hbox = Gtk.Box.new(Gtk.Orientation.HORIZONTAL, 6)
        hbox.pack_end(self.combo, False, True, 0)
        hbox.pack_end(Gtk.Label.new(_("Select File Type:")), False, True, 0)
        hbox.show_all()

        # 设置额外的小部件
        self.set_extra_widget(hbox)
        # 设置是否进行覆盖确认
        self.set_do_overwrite_confirmation(True)
        # 设置默认响应
        self.set_default_response(Gtk.ResponseType.OK)
    # 当组合框内容改变时的回调函数
    def combo_changed_cb(self, combo):
        # 获取当前文件名，如果不存在则为空字符串
        filename = self.get_filename() or ""
        # 分离路径和文件名
        dir, basename = os.path.split(filename)
        # 如果路径不等于当前文件夹，则设置当前文件夹为路径
        if dir != self.get_current_folder():
            self.set_current_folder(dir)
    
        # 查找推荐的扩展名
        new_ext = combo.get_model().get_value(combo.get_active_iter(), 2)
        if new_ext is not None:
            # 将文件名更改为推荐的扩展名
            root, ext = os.path.splitext(basename)
            if len(ext) == 0 and root.startswith("."):
                root = ""
            self.set_current_name(root + new_ext)
    
    # 获取文件扩展名
    def get_extension(self):
        return os.path.splitext(self.get_filename())[1]
    
    # 获取用户选择的保存格式，返回字符串，可以是"text"或"xml"
    def get_format(self):
        filetype = self.combo.get_model().get_value(
                self.combo.get_active_iter(), 1)
        if filetype is None:
            # 根据扩展名猜测，默认为"xml"
            return self.EXTENSIONS.get(self.get_extension(), "xml")
        return filetype
# 创建一个自定义的目录选择对话框类，继承自 Gtk.FileChooserDialog
class DirectoryChooserDialog(Gtk.FileChooserDialog):
    # 初始化方法，接受标题、父窗口、操作类型、按钮和后端参数
    def __init__(self, title="", parent=None,
                 action=Gtk.FileChooserAction.SELECT_FOLDER,
                 buttons=(Gtk.STOCK_CANCEL, Gtk.ResponseType.CANCEL,
                          Gtk.STOCK_OPEN, Gtk.ResponseType.OK), backend=None):
        # 调用父类的初始化方法，设置对话框的标题、父窗口、操作类型和按钮
        Gtk.FileChooserDialog.__init__(self, title=title, parent=parent,
                                          action=action, buttons=buttons)
        # 设置默认的响应类型为 OK
        self.set_default_response(Gtk.ResponseType.OK)

# 创建一个自定义的保存到目录选择对话框类，继承自 Gtk.FileChooserDialog
class SaveToDirectoryChooserDialog(Gtk.FileChooserDialog):
    # 初始化方法，接受标题、父窗口、操作类型、按钮和后端参数
    def __init__(self, title="", parent=None,
                 action=Gtk.FileChooserAction.SELECT_FOLDER,
                 buttons=(Gtk.STOCK_CANCEL, Gtk.ResponseType.CANCEL,
                          Gtk.STOCK_SAVE, Gtk.ResponseType.OK), backend=None):
        # 调用父类的初始化方法，设置对话框的标题、父窗口、操作类型和按钮
        Gtk.FileChooserDialog.__init__(self, title=title, parent=parent,
                                          action=action, buttons=buttons)
        # 设置默认的响应类型为 OK
        self.set_default_response(Gtk.ResponseType.OK)
```