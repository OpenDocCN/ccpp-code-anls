# `nmap\zenmap\zenmapGUI\MainWindow.py`

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
# 导入gi模块，确保使用的是Gtk 3.0版本
import gi
gi.require_version("Gtk", "3.0")
from gi.repository import Gtk

# 导入系统和路径相关的模块
import sys
import os
from os.path import split, isfile, join, abspath

# 防止加载PyXML
import xml
xml.__path__ = [x for x in xml.__path__ if "_xmlplus" not in x]

# 导入XML工具相关的模块
import xml.sax.saxutils

# 导入zenmapGUI相关的模块
from zenmapGUI.higwidgets.higwindows import HIGMainWindow
from zenmapGUI.higwidgets.higdialogs import HIGDialog, HIGAlertDialog
from zenmapGUI.higwidgets.higlabels import HIGEntryLabel
from zenmapGUI.higwidgets.higboxes import HIGHBox, HIGVBox

# 导入zenmapGUI的App模块
import zenmapGUI.App

# 导入zenmapGUI的FileChoosers模块
from zenmapGUI.FileChoosers import RESPONSE_OPEN_DIRECTORY, \
        ResultsFileChooserDialog, SaveResultsFileChooserDialog, \
        SaveToDirectoryChooserDialog

# 导入zenmapGUI的ScanInterface模块
from zenmapGUI.ScanInterface import ScanInterface

# 导入zenmapGUI的ProfileEditor模块
from zenmapGUI.ProfileEditor import ProfileEditor
# 从 zenmapGUI.About 模块中导入 About 类
from zenmapGUI.About import About
# 从 zenmapGUI.DiffCompare 模块中导入 DiffWindow 类
from zenmapGUI.DiffCompare import DiffWindow
# 从 zenmapGUI.SearchWindow 模块中导入 SearchWindow 类
from zenmapGUI.SearchWindow import SearchWindow
# 从 zenmapGUI.BugReport 模块中导入 BugReport 类
from zenmapGUI.BugReport import BugReport

# 从 zenmapCore.Name 模块中导入 APP_DISPLAY_NAME, APP_DOCUMENTATION_SITE
from zenmapCore.Name import APP_DISPLAY_NAME, APP_DOCUMENTATION_SITE
# 从 zenmapCore.Paths 模块中导入 Path
from zenmapCore.Paths import Path
# 从 zenmapCore.RecentScans 模块中导入 recent_scans
from zenmapCore.RecentScans import recent_scans
# 从 zenmapCore.UmitLogging 模块中导入 log
from zenmapCore.UmitLogging import log
# 导入 zenmapCore.I18N 模块，但未使用
import zenmapCore.I18N  # lgtm[py/unused-import]
# 从 zenmapGUI.Print 模块中导入所有内容
import zenmapGUI.Print
# 从 zenmapCore.UmitConf 模块中导入 SearchConfig, is_maemo, WindowConfig, config_parser
from zenmapCore.UmitConf import SearchConfig, is_maemo, WindowConfig, config_parser

# 初始化 UmitScanWindow 和 hildon 为 None
UmitScanWindow = None
hildon = None

# 如果是 maemo 平台
if is_maemo():
    # 导入 hildon 模块
    import hildon
    # 定义 UmitScanWindow 类，继承自 hildon.Window
    class UmitScanWindow(hildon.Window):
        # 初始化方法
        def __init__(self):
            # 调用父类的初始化方法
            hildon.Window.__init__(self)
            # 设置窗口不可调整大小
            self.set_resizable(False)
            # 设置窗口边框宽度为 0
            self.set_border_width(0)
            # 创建一个垂直方向的 Gtk.Box 容器
            self.vbox = Gtk.Box.new(Gtk.Orientation.VERTICAL, 0)
            # 设置容器边框宽度为 0

# 如果不是 maemo 平台
else:
    # 导入 HIGMainWindow 类
    class UmitScanWindow(HIGMainWindow):
        # 初始化方法
        def __init__(self):
            # 调用父类的初始化方法
            HIGMainWindow.__init__(self)
            # 创建一个垂直方向的 Gtk.Box 容器

# 定义 can_print 函数
def can_print():
    """Return true if we have printing operations (PyGTK 2.10 or later) or
    false otherwise."""
    # 尝试导入 Gtk.PrintOperation 类
    try:
        Gtk.PrintOperation
    # 如果出现 AttributeError 异常
    except AttributeError:
        # 返回 False
        return False
    # 如果没有出现异常
    else:
        # 返回 True
        return True

# 定义 ScanWindow 类，继承自 UmitScanWindow
class ScanWindow(UmitScanWindow):
    # 初始化函数，继承自 UmitScanWindow 类
    def __init__(self):
        # 调用父类的初始化函数
        UmitScanWindow.__init__(self)

        # 创建窗口配置对象
        window = WindowConfig()

        # 设置窗口标题
        self.set_title(_(APP_DISPLAY_NAME))
        # 移动窗口到指定位置
        self.move(window.x, window.y)
        # 设置窗口默认大小
        self.set_default_size(window.width, window.height)

        # 创建扫描界面对象
        self.scan_interface = ScanInterface()

        # 创建主加速组
        self.main_accel_group = Gtk.AccelGroup()
        # 将主加速组添加到窗口
        self.add_accel_group(self.main_accel_group)

        # self.vbox 是菜单栏和扫描界面的容器
        self.add(self.vbox)

        # 监听窗口关闭事件，调用 _exit_cb 方法
        self.connect('delete-event', self._exit_cb)
        # 创建 UI 管理器
        self._create_ui_manager()
        # 创建菜单栏
        self._create_menubar()
        # 创建扫描界面
        self._create_scan_interface()

        # 初始化结果文件选择对话框和关于对话框
        self._results_filechooser_dialog = None
        self._about_dialog = None

    # 显示“如何报告 Bug”窗口
    def _show_bug_report(self, widget):
        bug = BugReport()
        bug.show_all()

    # 显示搜索窗口
    def _search_scan_result(self, widget):
        search_window = SearchWindow(
                self._load_search_result, self._append_search_result)
        search_window.show_all()

    # 过滤回调函数
    def _filter_cb(self, widget):
        self.scan_interface.toggle_filter_bar()

    # 加载搜索结果的回调函数
    def _load_search_result(self, results):
        """This function is passed as an argument to the SearchWindow.__init__
        method.  When the user selects scans in the search window and clicks on
        "Open", this function is called to load each of the selected scans into
        a new window."""
        for result in results:
            self._load(self.get_empty_interface(),
                    parsed_result=results[result][1])
    # 将搜索结果追加到当前窗口的私有方法
    def _append_search_result(self, results):
        """This function is passed as an argument to the SearchWindow.__init__
        method.  When the user selects scans in the search window and clicks on
        "Append", this function is called to append the selected scans into the
        current window."""
        # 遍历搜索结果
        for result in results:
            # 调用_load方法加载选定的扫描结果到扫描界面
            self._load(self.scan_interface, parsed_result=results[result][1])

    # 将网络清单存储到数据库中的方法
    def store_result(self, scan_interface):
        """Stores the network inventory into the database."""
        # 记录调试信息
        log.debug(">>> Saving result into database...")
        try:
            # 将网络清单存储到数据库中
            scan_interface.inventory.save_to_db()
        except Exception as e:
            # 弹出警告对话框，提示无法存储到数据库
            alert = HIGAlertDialog(
                    message_format=_("Can't save to database"),
                    secondary_text=_("Can't store unsaved scans to the "
                        "recent scans database:\n%s") % str(e))
            alert.run()
            alert.destroy()
            # 记录无法存储到数据库的错误信息
            log.debug(">>> Can't save result to database: %s." % str(e))

    # 获取最近的七次扫描并将它们追加到默认的UI定义中
    def get_recent_scans(self):
        """Gets seven most recent scans and appends them to the default UI
        definition."""
        # 获取最近的扫描列表
        r_scans = recent_scans.get_recent_scans_list()
        new_rscan_xml = ''

        # 遍历最近的七次扫描
        for scan in r_scans[:7]:
            scan = scan.replace('\n', '')
            # 检查扫描文件是否可读且存在
            if os.access(split(scan)[0], os.R_OK) and isfile(scan):
                scan = scan.replace('\n', '')
                # 创建新的扫描项，并将其添加到主操作列表中
                new_rscan = (
                        scan, None, scan, None, scan, self._load_recent_scan)
                new_rscan_xml += "<menuitem action=%s/>\n" % (
                        xml.sax.saxutils.quoteattr(scan))

                self.main_actions.append(new_rscan)

        # 添加分隔符和新的扫描项到默认UI定义中
        new_rscan_xml += "<separator />\n"
        self.default_ui %= new_rscan_xml
    # 创建菜单栏
    def _create_menubar(self):
        # 获取并添加菜单栏
        menubar = self.ui_manager.get_widget('/menubar')

        # 如果是 Maemo 平台
        if is_maemo():
            # 创建一个菜单
            menu = Gtk.Menu()
            # 将菜单栏的子项添加到菜单中
            for child in menubar.get_children():
                child.reparent(menu)
            # 设置菜单
            self.set_menu(menu)
            # 销毁原菜单栏
            menubar.destroy()
            self.menubar = menu
        else:
            self.menubar = menubar
            self.vbox.pack_start(self.menubar, False, False, 0)

        # 显示菜单栏
        self.menubar.show_all()

    # 创建扫描界面
    def _create_scan_interface(self):
        # 获取扫描结果的笔记本
        notebook = self.scan_interface.scan_result.scan_result_notebook
        # 连接“追加扫描结果”按钮的点击事件
        notebook.scans_list.append_button.connect(
                "clicked", self._append_scan_results_cb)
        # 连接 Nmap 输出的改变事件
        notebook.nmap_output.connect("changed", self._displayed_scan_change_cb)
        # 调用显示扫描结果改变的回调函数
        self._displayed_scan_change_cb(None)
        # 显示扫描界面
        self.scan_interface.show_all()
        self.vbox.pack_start(self.scan_interface, True, True, 0)

    # 显示打开文件对话框
    def show_open_dialog(self, title=None):
        """Show a load file chooser and return the filename chosen."""
        # 如果结果文件选择对话框为空，则创建一个
        if self._results_filechooser_dialog is None:
            self._results_filechooser_dialog = ResultsFileChooserDialog(
                    title=title)

        filename = None
        # 运行结果文件选择对话框
        response = self._results_filechooser_dialog.run()
        # 如果选择了文件
        if response == Gtk.ResponseType.OK:
            filename = self._results_filechooser_dialog.get_filename()
        # 如果选择了打开目录
        elif response == RESPONSE_OPEN_DIRECTORY:
            filename = self._results_filechooser_dialog.get_filename()

            # 检查所选文件名是否是一个目录，如果不是，则只取路径的目录部分，省略所选文件的实际名称
            if filename is not None and not os.path.isdir(filename):
                filename = os.path.dirname(filename)

        # 隐藏结果文件选择对话框
        self._results_filechooser_dialog.hide()
        # 返回所选文件名
        return filename
    # 加载扫描结果的回调函数，显示文件选择对话框并从所选文件或所选目录加载扫描结果
    def _load_scan_results_cb(self, p):
        """'Open Scan' callback function. Displays a file chooser dialog and
        loads the scan from the selected file or from the selected
        directory."""
        # 获取文件名
        filename = self.show_open_dialog(p.get_name())
        # 如果文件名不为空
        if filename is not None:
            # 获取一个空的接口
            scan_interface = self.get_empty_interface()
            # 如果是目录
            if os.path.isdir(filename):
                # 加载目录
                self._load_directory(scan_interface, filename)
            else:
                # 加载文件
                self._load(scan_interface, filename)

    # 'Append Scan' 回调函数，显示文件选择对话框并将所选文件的扫描结果追加到当前窗口
    def _append_scan_results_cb(self, p):
        """'Append Scan' callback function. Displays a file chooser dialog and
        appends the scan from the selected file into the current window."""
        # 获取文件名
        filename = self.show_open_dialog(p.get_name())
        # 如果文件名不为空
        if filename is not None:
            # 如果是目录
            if os.path.isdir(filename):
                # 加载目录
                self._load_directory(self.scan_interface, filename)
            else:
                # 加载文件
                self._load(self.scan_interface, filename)

    # 当当前显示的扫描输出发生变化时调用
    def _displayed_scan_change_cb(self, widget):
        """Called when the currently shown scan output is changed."""
        # 如果有要打印的内容，设置“打印...”菜单项为可用状态
        widget = self.ui_manager.get_widget("/ui/menubar/Scan/Print...")
        if widget is None:
            # 如果缺乏支持，没有打印菜单项
            return
        # 获取当前活动的条目
        entry = self.scan_interface.scan_result.scan_result_notebook.nmap_output.get_active_entry()  # noqa
        widget.set_sensitive(entry is not None)

    # 从菜单直接加载最近的扫描结果的辅助函数
    def _load_recent_scan(self, widget):
        """A helper function for loading a recent scan directly from the
        menu."""
        # 加载一个空的接口
        self._load(self.get_empty_interface(), widget.get_name())
    # 从文件或解析结果加载扫描数据到给定的扫描接口
    def _load(self, scan_interface, filename=None, parsed_result=None):
        # 如果既没有文件名也没有解析结果，则返回空
        if not (filename or parsed_result):
            return None

        if filename:
            # 从文件加载扫描结果
            log.debug(">>> Loading file: %s" % filename)
            try:
                # 解析结果
                scan_interface.load_from_file(filename)
            except Exception as e:
                # 弹出错误对话框
                alert = HIGAlertDialog(message_format=_('Error loading file'),
                                       secondary_text=str(e))
                alert.run()
                alert.destroy()
                return
            # 保存文件名到扫描接口
            scan_interface.saved_filename = filename
        elif parsed_result:
            # 从解析对象加载扫描结果
            scan_interface.load_from_parsed_result(parsed_result)

    # 从目录加载扫描结果到给定的扫描接口
    def _load_directory(self, scan_interface, directory):
        # 遍历目录下的文件
        for file in os.listdir(directory):
            # 如果是目录则跳过
            if os.path.isdir(os.path.join(directory, file)):
                continue
            # 调用_load方法加载文件到扫描接口
            self._load(scan_interface, filename=os.path.join(directory, file))

    # 保存到目录的回调函数
    def _save_to_directory_cb(self, widget):
        # 如果扫描接口为空，则弹出提示框
        if self.scan_interface.empty:
            alert = HIGAlertDialog(message_format=_('Nothing to save'),
                                   secondary_text=_('\
        # 如果扫描尚未运行，则弹出警告窗口并返回
        if not self.scan_interface.has_run():
            alert = HIGAlertDialog(message_format=_('This scan has not been run yet. Start the scan with the "Scan" button first.'))
            alert.run()
            alert.destroy()
            return
        
        # 获取当前正在运行的扫描数量
        num_scans_running = self.scan_interface.num_scans_running()
        
        # 如果有正在运行的扫描，则弹出警告窗口并返回
        if num_scans_running > 0:
            if num_scans_running == 1:
                text = _("There is a scan still running. Wait until it finishes and then save.")
            else:
                text = _("There are %u scans still running. Wait until they finish and then save.") % num_scans_running
            alert = HIGAlertDialog(message_format=_('Scan is running'), secondary_text=text)
            alert.run()
            alert.destroy()
            return
        
        # 如果有多个扫描在网络清单中，则显示一个目录选择对话框
        dir_chooser = SaveToDirectoryChooserDialog(title=_("Choose a directory to save scans into"))
        
        # 如果用户选择了目录，则保存所有扫描结果到该目录
        if dir_chooser.run() == Gtk.ResponseType.OK:
            self._save_all(self.scan_interface, dir_chooser.get_filename())
        dir_chooser.destroy()

    # 关于对话框的响应函数
    def _about_cb_response(self, dialog, response_id):
        if response_id == Gtk.ResponseType.DELETE_EVENT:
            self._about_dialog = None
        else:
            self._about_dialog.hide()

    # 显示关于对话框的回调函数
    def _show_about_cb(self, widget):
        if self._about_dialog is None:
            self._about_dialog = About()
            self._about_dialog.connect("response", self._about_cb_response)
        self._about_dialog.present()
    # 保存所有扫描到给定目录中
    def _save_all(self, scan_interface, directory):
        """Saves all scans in saving_page's inventory to a given directory.
        Displays an alert dialog if the save fails."""
        try:
            # 将扫描页面的库存保存到指定目录，并返回文件名列表
            filenames = scan_interface.inventory.save_to_dir(directory)
            # 将所有扫描标记为已保存
            for scan in scan_interface.inventory.get_scans():
                scan.unsaved = False
        except Exception as ex:
            # 如果保存失败，显示警告对话框
            alert = HIGAlertDialog(message_format=_('Can\'t save file'),
                        secondary_text=str(ex))
            alert.run()
            alert.destroy()
        else:
            # 如果保存成功，设置扫描页面的已保存文件名为目录名
            scan_interface.saved_filename = directory

            # 保存最近扫描信息
            try:
                # 将每个文件名添加到最近扫描列表中
                for filename in filenames:
                    recent_scans.add_recent_scan(filename)
                # 保存最近扫描信息
                recent_scans.save()
            except (OSError, IOError) as e:
                # 如果保存最近扫描信息失败，显示警告对话框
                alert = HIGAlertDialog(
                        message_format=_(
                            "Can't save recent scan information"),
                        secondary_text=_(
                            "Can't open file to write.\n%s") % str(e))
                alert.run()
                alert.destroy()
    # 将扫描保存到指定文件名，如果保存失败则显示警告对话框
    def _save(self, scan_interface, saved_filename, selected_index,
            format="xml"):
        """Saves the scan into a file with a given filename. Displays an alert
        dialog if the save fails."""
        # 打印保存的文件名
        log.debug(">>> File being saved: %s" % saved_filename)
        try:
            # 将扫描保存到文件
            scan_interface.inventory.save_to_file(
                    saved_filename, selected_index, format)
            # 将选定的扫描标记为已保存
            scan_interface.inventory.get_scans()[selected_index].unsaved = False  # noqa
        except (OSError, IOError) as e:
            # 如果保存失败，显示警告对话框
            alert = HIGAlertDialog(
                    message_format=_("Can't save file"),
                    secondary_text=_("Can't open file to write.\n%s") % str(e))
            alert.run()
            alert.destroy()
        else:
            # 更新保存的文件名
            scan_interface.saved_filename = saved_filename

            # 打印页面是否有更改
            log.debug(">>> Changes on page? %s" % scan_interface.changed)
            # 打印文件保存的位置
            log.debug(">>> File saved at: %s" % scan_interface.saved_filename)

            if format == "xml":
                # 保存最近的扫描信息
                try:
                    recent_scans.add_recent_scan(saved_filename)
                    recent_scans.save()
                except (OSError, IOError) as e:
                    # 如果保存最近的扫描信息失败，显示警告对话框
                    alert = HIGAlertDialog(
                            message_format=_(
                                "Can't save recent scan information"),
                            secondary_text=_(
                                "Can't open file to write.\n%s") % str(e))
                    alert.run()
                    alert.destroy()

    # 返回空的扫描界面，如果不为空则创建并返回一个新的界面
    def get_empty_interface(self):
        """Return this window if it is empty, otherwise create and return a new
        one."""
        if self.scan_interface.empty:
            return self.scan_interface
        return self._new_scan_cb().scan_interface
    # 创建一个新的扫描窗口
    def _new_scan_cb(self, widget=None, data=None):
        # 调用 zenmapGUI.App 的 new_window 方法创建一个新窗口
        w = zenmapGUI.App.new_window()
        # 显示新窗口
        w.show_all()
        # 返回新窗口对象
        return w
    
    # 创建一个新的扫描配置窗口
    def _new_scan_profile_cb(self, p):
        # 创建一个 ProfileEditor 对象，设置扫描命令和不可删除属性
        pe = ProfileEditor(
                command=self.scan_interface.command_toolbar.command,
                deletable=False)
        # 设置扫描接口
        pe.set_scan_interface(self.scan_interface)
        # 显示窗口
        pe.show_all()
    
    # 编辑扫描配置窗口
    def _edit_scan_profile_cb(self, p):
        # 创建一个 ProfileEditor 对象，设置配置名称、可删除和覆盖属性
        pe = ProfileEditor(
                profile_name=self.scan_interface.toolbar.selected_profile,
                deletable=True, overwrite=True)
        # 设置扫描接口
        pe.set_scan_interface(self.scan_interface)
        # 显示窗口
        pe.show_all()
    
    # 帮助回调函数
    def _help_cb(self, action):
        # 调用 show_help 方法显示帮助信息
        self.show_help()
    
    # 打印回调函数
    def _print_cb(self, *args):
        # 显示打印对话框
        entry = self.scan_interface.scan_result.scan_result_notebook.nmap_output.get_active_entry()  # noqa
        if entry is None:
            return False
        # 运行打印操作
        zenmapGUI.Print.run_print_operation(
                self.scan_interface.inventory, entry)
    
    # 退出回调函数
    def _quit_cb(self, *args):
        # 关闭所有打开的窗口
        for window in zenmapGUI.App.open_windows[:]:
            # 激活窗口
            window.present()
            # 调用窗口的退出回调函数
            if window._exit_cb():
                break
    
    # 加载差异比较回调函数
    def _load_diff_compare_cb(self, widget=None, extra=None):
        # 将所有活动扫描加载到字典中，传递给 DiffWindow 构造函数，然后显示“比较结果”窗口
        self.diff_window = DiffWindow(
                self.scan_interface.inventory.get_scans())
        self.diff_window.show_all()
        # 定义一个方法用于显示帮助文档
        def show_help(self):
            # 导入 urllib.request 模块和 webbrowser 模块
            import urllib.request
            import webbrowser

            # 获取帮助文档的绝对路径
            doc_path = abspath(join(Path.docs_dir, "help.html"))
            # 将文件路径转换为 URL 格式
            url = "file:" + urllib.request.pathname2url(doc_path)

            # 尝试在浏览器中打开帮助文档
            try:
                webbrowser.open(url, new=2)
            # 如果出现 OSError 异常，显示警告对话框
            except OSError as e:
                d = HIGAlertDialog(parent=self,
                                   message_format=_("Can't find documentation files"),
                                   secondary_text=_("""\
# 如果加载文档文件出现错误，则显示错误信息，并提供在线文档的链接
There was an error loading the documentation file %s (%s). See the \
online documentation at %s.\
""") % (doc_path, str(e), APP_DOCUMENTATION_SITE))
# 运行错误信息对话框
            d.run()
# 销毁错误信息对话框
            d.destroy()

# 如果作为主程序执行，则创建扫描窗口对象
if __name__ == '__main__':
    w = ScanWindow()
    # 显示扫描窗口
    w.show_all()
    # 运行 GTK 主循环
    Gtk.main()
```