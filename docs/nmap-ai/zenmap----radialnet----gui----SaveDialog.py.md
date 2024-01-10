# `nmap\zenmap\radialnet\gui\SaveDialog.py`

```
# 设置文件编码为 UTF-8
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
# 导入 gi 模块
import gi
# 要求使用 Gtk 版本 3.0
gi.require_version("Gtk", "3.0")
# 从 gi.repository 中导入 Gtk 模块
from gi.repository import Gtk

# 导入 os.path 模块
import os.path
# 导入 radialnet.gui.RadialNet 模块
import radialnet.gui.RadialNet as RadialNet
# 导入 zenmapGUI.FileChoosers 模块
import zenmapGUI.FileChoosers

# 从 zenmapGUI.higwidgets.higboxes 中导入 HIGHBox 模块
from zenmapGUI.higwidgets.higboxes import HIGHBox
# 从 zenmapGUI.higwidgets.higdialogs 中导入 HIGAlertDialog 模块

# 定义 TYPES 元组
TYPES = ((_("By extension"), None, None),
         ("PDF", RadialNet.FILE_TYPE_PDF, ".pdf"),
         ("PNG", RadialNet.FILE_TYPE_PNG, ".png"),
         ("PostScript", RadialNet.FILE_TYPE_PS, ".ps"),
         ("SVG", RadialNet.FILE_TYPE_SVG, ".svg"))
# 为 "By extension" 文件类型构建一个反向索引，将扩展名映射到文件类型
EXTENSIONS = {}
for type in TYPES:
    if type[2] is not None:
        EXTENSIONS[type[2]] = type[1]
class SaveDialog(Gtk.FileChooserDialog):
    # 定义 SaveDialog 类，继承自 Gtk.FileChooserDialog
    def __init__(self):
        """
        """
        # 初始化 SaveDialog 对象
        super(SaveDialog, self).__init__(title=_("Save Topology"),
                action=Gtk.FileChooserAction.SAVE,
                buttons=(Gtk.STOCK_CANCEL, Gtk.ResponseType.CANCEL,
                    Gtk.STOCK_SAVE, Gtk.ResponseType.OK))

        # 创建一个存储类型的列表
        types_store = Gtk.ListStore.new([str, object, str])
        # 遍历 TYPES 列表，将每个类型添加到 types_store 中
        for type in TYPES:
            types_store.append(type)

        # 创建一个下拉框，并将 types_store 作为其数据模型
        self.__combo = Gtk.ComboBox.new_with_model(types_store)
        cell = Gtk.CellRendererText()
        self.__combo.pack_start(cell, True)
        self.__combo.add_attribute(cell, "text", 0)

        # 监听下拉框的选择变化事件
        self.__combo.connect("changed", self.__combo_changed_cb)
        # 设置下拉框的默认选项
        self.__combo.set_active(0)

        # 监听对话框的响应事件
        self.connect("response", self.__response_cb)

        # 创建一个水平布局
        hbox = HIGHBox()
        # 创建一个标签
        label = Gtk.Label.new(_("Select File Type:"))
        # 将下拉框和标签添加到水平布局中
        hbox.pack_end(self.__combo, False, True, 0)
        hbox.pack_end(label, False, True, 0)

        # 将水平布局设置为对话框的额外部件
        self.set_extra_widget(hbox)
        # 设置是否进行覆盖确认
        self.set_do_overwrite_confirmation(True)

        # 显示所有部件
        hbox.show_all()

    def __combo_changed_cb(self, widget):
        # 获取当前文件名，如果不存在则为空字符串
        filename = self.get_filename() or ""
        # 获取文件路径和文件名
        dir, basename = os.path.split(filename)
        # 如果路径不等于当前文件夹，则设置当前文件夹为路径
        if dir != self.get_current_folder():
            self.set_current_folder(dir)

        # 查找推荐的扩展名
        new_ext = self.__combo.get_model().get_value(
                self.__combo.get_active_iter(), 2)
        # 如果存在推荐的扩展名
        if new_ext is not None:
            # 更改文件名以使用推荐的扩展名
            root, ext = os.path.splitext(basename)
            if len(ext) == 0 and root.startswith("."):
                root = ""
            self.set_current_name(root + new_ext)
    # 定义一个回调函数，用于拦截“response”信号，检查是否有人使用了未知扩展名的“按扩展名”文件类型
    def __response_cb(self, widget, response_id):
        # 如果响应 ID 为 OK，并且文件类型为 None，则执行以下操作
        if response_id == Gtk.ResponseType.OK and self.get_filetype() is None:
            # 获取文件的扩展名
            ext = self.__get_extension()
            # 如果扩展名为空
            if ext == "":
                # 获取文件名，并分割成目录和文件名
                filename = self.get_filename() or ""
                dir, basename = os.path.split(filename)
                # 创建一个警告对话框
                alert = HIGAlertDialog(
                        message_format=_("No filename extension"),
                        secondary_text=_("""\
# 如果文件名没有扩展名，并且没有选择特定的文件类型，则提示用户输入已知的扩展名或从列表中选择文件类型
The filename "%s" does not have an extension, \
and no specific file type was chosen.
Enter a known extension or select the file type from the list.""" % basename))

            else:
                # 创建警告对话框，提示用户未知的文件名扩展名
                alert = HIGAlertDialog(
                        message_format=_("Unknown filename extension"),
                        secondary_text=_("""\
There is no file type known for the filename extension "%s".
Enter a known extension or select the file type from the list.\
""") % self.__get_extension())
            # 运行警告对话框
            alert.run()
            # 销毁警告对话框
            alert.destroy()
            # 返回到对话框
            self.emit_stop_by_name("response")

    # 获取文件扩展名
    def __get_extension(self):
        return os.path.splitext(self.get_filename())[1]

    # 获取文件类型
    def get_filetype(self):
        # 从下拉框中获取用户选择的文件类型
        filetype = self.__combo.get_model().get_value(
                self.__combo.get_active_iter(), 1)
        if filetype is None:
            # 如果用户没有选择文件类型，则根据扩展名猜测文件类型
            return EXTENSIONS.get(self.__get_extension())
        return filetype
```