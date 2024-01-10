# `nmap\zenmap\zenmapGUI\App.py`

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
# 导入所需的模块
import os  # 导入操作系统模块
import signal  # 导入信号处理模块
import sys  # 导入系统相关的模块
import configparser  # 导入配置文件解析模块
import shutil  # 导入文件操作模块

# 如果 PyGTK 无法打开显示器，则引发异常。通常这只会产生警告，但缺少显示器最终会导致分段错误。
# 详见 http://live.gnome.org/PyGTK/WhatsNew210。
# 'append = "True"' 是为了解决在使用 Python 2.7 时，PyGTK 出现错误导致断言失败的问题。
# 详见 https://bugzilla.redhat.com/show_bug.cgi?id=620216#c10。
import warnings  # 导入警告模块
warnings.filterwarnings("error", module="gtk", append="True")  # 设置警告过滤器，当模块为 gtk 时，将错误追加到现有警告列表中

try:
    import gi  # 尝试导入 gi 模块
    gi.require_version("Gtk", "3.0")  # 要求 gi 版本为 3.0
    from gi.repository import Gtk, Gdk  # 从 gi 模块的 repository 中导入 Gtk 和 Gdk
except Exception:
    # 在 Mac OS X 10.5 上，X11 应该会在需要时自动启动。
    pass  # 如果出现异常，则不做任何操作
    # 设置 DISPLAY 环境变量并拦截该套接字上的流量来实现功能；参见 http://homepage.mac.com/sao1/X11/#four。
    # 但是，如果在 .profile 等 shell 启动文件中设置了 DISPLAY，这种方法会以一种奇怪的方式中断。
    # 这些文件只对 shell 有影响，而不对图形环境有影响，因此 X11 会如预期地启动，但在这样做时会读取启动脚本，
    # 由于某种原因，第一个连接（导致启动的连接）被拒绝。但是不知何故，随后的连接却正常工作！
    # 因此，如果导入失败，请再试一次。
    import gi
    # 要求使用指定版本的 Gtk
    gi.require_version("Gtk", "3.0")
    # 从 gi.repository 中导入 Gtk 和 Gdk
    from gi.repository import Gtk, Gdk
# 重置警告，清除所有警告状态
warnings.resetwarnings()

# 导入 HIGAlertDialog 类
from zenmapGUI.higwidgets.higdialogs import HIGAlertDialog

# 导入 is_maemo 和 SearchConfig 函数，以及 zenmapCore.UmitConf 模块
from zenmapCore.UmitConf import is_maemo, SearchConfig
import zenmapCore.UmitConf

# 导入 log 函数，以及 zenmapCore.UmitOptionParser 模块
from zenmapCore.UmitLogging import log
from zenmapCore.UmitOptionParser import option_parser

# 导入 APP_NAME, APP_DISPLAY_NAME, NMAP_DISPLAY_NAME 变量，以及 zenmapCore.I18N 模块
from zenmapCore.Name import APP_NAME, APP_DISPLAY_NAME, NMAP_DISPLAY_NAME
import zenmapCore.I18N  # lgtm[py/unused-import]

# 导入 Path, create_user_config_dir 函数，以及 zenmapCore.Paths 模块
from zenmapCore.Paths import Path, create_user_config_dir

# 重新导入 APP_DISPLAY_NAME 变量
from zenmapCore.Name import APP_DISPLAY_NAME

# 再次导入 HIGAlertDialog 类

# 一个全局的打开扫描窗口的列表。当最后一个窗口被销毁时，调用 Gtk.main_quit。
open_windows = []

# 当窗口被销毁时的回调函数
def _destroy_callback(window):
    open_windows.remove(window)
    if len(open_windows) == 0:
        Gtk.main_quit()
    try:
        # 尝试导入 UmitDB 类
        from zenmapCore.UmitDB import UmitDB
    except ImportError as e:
        # 如果导入失败，记录错误信息
        log.debug(">>> Not cleaning up database: %s." % str(e))
    else:
        # 清理数据库
        UmitDB().cleanup(SearchConfig().converted_save_time)

# 创建新窗口的函数
def new_window():
    # 导入 ScanWindow 类
    from zenmapGUI.MainWindow import ScanWindow
    # 创建 ScanWindow 实例
    w = ScanWindow()
    # 将销毁窗口的回调函数与窗口连接
    w.connect("destroy", _destroy_callback)
    # 如果是 maemo 平台，导入 hildon 模块，并将窗口添加到 hildon 程序中
    if is_maemo():
        import hildon
        hildon_app = hildon.Program()
        hildon_app.add_window(w)
    # 将窗口添加到打开窗口列表中
    open_windows.append(w)
    return w

# 检查是否是 root 用户
def is_root():
    if 'NMAP_PRIVILEGED' in os.environ:
        return True
    elif 'NMAP_UNPRIVILEGED' in os.environ:
        return False
    else:
        return sys.platform == "win32" or os.getuid() == 0 or is_maemo()

# 安装异常钩子，捕获异常并发送到 bugzilla
def install_excepthook():
    # 这将捕获异常并发送到 bugzilla
    # 定义一个异常钩子函数，用于处理未捕获的异常
    def excepthook(type, value, tb):
        # 导入 traceback 模块，用于打印异常信息
        import traceback

        # 打印异常信息
        traceback.print_exception(type, value, tb)

        # 设置警告过滤器，如果 PyGTK 无法打开显示器，则引发异常
        warnings.filterwarnings("error", module="gtk")
        # 导入 gi 模块，要求使用 Gtk 3.0 版本
        import gi
        gi.require_version("Gtk", "3.0")
        # 从 gi.repository 导入 Gtk 和 Gdk 模块
        from gi.repository import Gtk, Gdk
        # 重置警告过滤器
        warnings.resetwarnings()

        # 进入 GDK 线程
        Gdk.threads_enter()

        # 从 zenmapGUI.higwidgets.higdialogs 模块导入 HIGAlertDialog 类
        from zenmapGUI.higwidgets.higdialogs import HIGAlertDialog
        # 从 zenmapGUI.CrashReport 模块导入 CrashReport 类
        from zenmapGUI.CrashReport import CrashReport
        # 如果异常类型为 ImportError
        if type == ImportError:
            # 创建一个错误消息对话框
            d = HIGAlertDialog(type=Gtk.MessageType.ERROR,
                message_format=_("Import error"),
                secondary_text=_("""A required module was not found.
# 定义一个函数，用于处理程序异常
def excepthook(type, value, tb):
    # 进入 GTK 主循环
    Gdk.threads_enter()

    # 如果是开发模式，打印异常信息
    if DEVELOPMENT:
        d = Gtk.MessageDialog(None, 0, Gtk.MessageType.ERROR,
                              Gtk.ButtonsType.CANCEL, str(type) + "\n\n" + str(value))
        d.run()
        d.destroy()
    else:
        # 创建一个崩溃报告窗口，并显示
        c = CrashReport(type, value, tb)
        c.show_all()
        Gtk.main()

    # 离开 GTK 主循环
    Gdk.threads_leave()

    # 退出 GTK 主循环
    Gtk.main_quit()

# 定义一个函数，用于安全关闭程序
def safe_shutdown(signum, stack):
    # 打印安全关闭信息
    log.debug("\n\n%s\nSAFE SHUTDOWN!\n%s\n" % ("#" * 30, "#" * 30))
    log.debug("SIGNUM: %s" % signum)

    # 关闭所有窗口的扫描任务
    for window in open_windows:
        window.scan_interface.kill_all_scans()

    # 退出程序
    sys.exit(signum)

# 运行程序
def run():
    # 如果是 POSIX 系统，注册 SIGHUP 信号处理函数
    if os.name == "posix":
        signal.signal(signal.SIGHUP, safe_shutdown)
    # 注册 SIGTERM 和 SIGINT 信号处理函数
    signal.signal(signal.SIGTERM, safe_shutdown)
    signal.signal(signal.SIGINT, safe_shutdown)

    # 获取开发模式标志
    DEVELOPMENT = os.environ.get(APP_NAME.upper() + "_DEVELOPMENT", False)
    # 如果不是开发模式，安装异常处理钩子
    if not DEVELOPMENT:
        install_excepthook()

    # 安装国际化支持
    zenmapCore.I18N.install_gettext(Path.locale_dir)

    try:
        # 创建用户配置目录
        create_user_config_dir(
                Path.user_config_dir, Path.config_dir)
    except (IOError, OSError) as e:
        # 处理创建用户配置目录时的异常
        error_dialog = HIGAlertDialog(
                message_format=_(
                    "Error creating the per-user configuration directory"),
                secondary_text=_("""\
There was an error creating the directory %s or one of the files in it. \
The directory is created by copying the contents of %s. \
The specific error was

%s

%s needs to create this directory to store information such as the list of \
scan profiles. Check for access to the directory and try again.""") % (
                    repr(Path.user_config_dir), repr(Path.config_dir),
                    repr(str(e)), APP_DISPLAY_NAME
                    )
                )
        error_dialog.run()
        error_dialog.destroy()
        sys.exit(1)
    # 尝试读取 ~/.zenmap/zenmap.conf 配置文件
    zenmapCore.UmitConf.config_parser.read(Path.user_config_file)
except configparser.ParsingError as e:
    # 如果出现 ParsingError 错误，将一些值留在列表中而不是字符串中。如果出现这个问题，就将配置文件全部清除。
    zenmapCore.UmitConf.config_parser = zenmapCore.UmitConf.config_parser.__class__()
    # 创建一个错误对话框，显示错误消息和辅助文本
    error_dialog = HIGAlertDialog(
            message_format=_("Error parsing the configuration file"),
            secondary_text=_("""\
# 如果解析配置文件出现错误，则显示错误对话框
There was an error parsing the configuration file %s. \
The specific error was

%s

%s can continue without this file but any information in it will be ignored \
until it is repaired.""") % (Path.user_config_file, str(e), APP_DISPLAY_NAME)
                )
        error_dialog.run()
        error_dialog.destroy()
        # 获取全局配置文件路径
        global_config_path = os.path.join(Path.config_dir, APP_NAME + '.conf')
        # 显示恢复默认配置的对话框
        repair_dialog = HIGAlertDialog(
                type=Gtk.MessageType.QUESTION,
                message_format=_("Restore default configuration?"),
                secondary_text=_("""\
To avoid further errors parsing the configuration file %s, \
you can copy the default configuration from %s.

Do this now? \
""") % (Path.user_config_file, global_config_path),
                )
        repair_dialog.add_button(Gtk.STOCK_CANCEL, Gtk.ResponseType.CANCEL)
        repair_dialog.set_default_response(Gtk.ResponseType.CANCEL)
        # 如果用户选择恢复默认配置，则复制全局配置文件到用户配置文件
        if repair_dialog.run() == Gtk.ResponseType.OK:
            shutil.copyfile(global_config_path, Path.user_config_file)
            log.debug(">>> Copy %s to %s." % (global_config_path, Path.user_config_file))
        repair_dialog.destroy()

    # 如果用户不是 root 用户，则显示警告
    if not is_root():
        non_root = NonRootWarning()
        non_root.run()
        non_root.destroy()

    # 加载作为命令行参数给定的文件
    filenames = option_parser.get_open_results()
    # 如果没有给定文件，则打开一个空白窗口
    if len(filenames) == 0:
        window = new_window()
        window.show_all()
    else:
        # 否则，遍历文件名列表，加载文件或目录到窗口中
        for filename in filenames:
            window = new_window()
            if os.path.isdir(filename):
                window._load_directory(window.scan_interface, filename)
            else:
                window._load(window.scan_interface, filename)
            window.show_all()

    # 获取 nmap、target 和 profile 参数
    nmap = option_parser.get_nmap()
    target = option_parser.get_target()
    profile = option_parser.get_profile()
    # 如果给定了 nmap 参数，则开始运行扫描
    if nmap:
        # 获取一个空的界面页面
        page = window.get_empty_interface()
        # 设置命令工具栏的命令为 nmap 参数的字符串表示
        page.command_toolbar.command = " ".join(nmap)
        # 调用开始扫描的回调函数
        page.start_scan_cb()
    # 如果给定了 target 或 profile 参数
    elif target or profile:
        # 获取一个空的界面页面
        page = window.get_empty_interface()
        # 如果给定了 target 参数，则设置工具栏的选定目标为 target
        if target:
            page.toolbar.selected_target = target
        # 如果给定了 profile 参数，则设置工具栏的选定配置文件为 profile
        if profile:
            page.toolbar.selected_profile = profile
        # 如果同时给定了 target 和 profile 参数，则调用开始扫描的回调函数
        if target and profile:
            page.start_scan_cb()

    # 运行 GTK 主循环
    Gtk.main()
# 定义一个名为 NonRootWarning 的类，继承自 HIGAlertDialog
class NonRootWarning (HIGAlertDialog):
    # 初始化方法
    def __init__(self):
        # 定义警告文本，使用 _() 函数进行国际化处理
        warning_text = _('''You are trying to run %s with a non-root user!

Some %s options need root privileges to work.''') % (
            APP_DISPLAY_NAME, NMAP_DISPLAY_NAME)
        
        # 调用父类的初始化方法，设置消息格式和次要文本
        HIGAlertDialog.__init__(self, message_format=_('Non-root user'),
                                secondary_text=warning_text)

# 如果当前脚本作为主程序运行，则调用 run() 函数
if __name__ == "__main__":
    run()
```