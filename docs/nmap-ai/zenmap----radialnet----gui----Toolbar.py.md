# `nmap\zenmap\radialnet\gui\Toolbar.py`

```
# 设置文件编码为 UTF-8

# 重要提示：NMAP 许可证条款
# Nmap 安全扫描器由 Nmap 软件有限责任公司（“Nmap 项目”）（C）1996-2023 年开发。Nmap 也是 Nmap 项目的注册商标。
# 本程序根据 Nmap 公共源代码许可证（NPSL）的条款分发。适用于特定 Nmap 版本或源代码控制修订版的确切许可证文本包含在该版本的 Nmap 或源代码控制修订版的 LICENSE 文件中。有关更多 Nmap 版权/法律信息，请访问 https://nmap.org/book/man-legal.html，有关 NPSL 许可证本身的更多信息，请访问 https://nmap.org/npsl/。本标题总结了 Nmap 许可证的一些关键点，但不能替代实际的许可证文本。
# Nmap 通常允许最终用户自行下载和使用，包括商业用途。可从 https://nmap.org 获取。
# Nmap 许可证通常禁止公司在商业产品中使用和重新分发 Nmap，但我们销售具有更宽松许可证和特殊功能的特殊 Nmap OEM 版本。请参阅 https://nmap.org/oem/
# 如果您收到了书面的 Nmap 许可协议或合同，其中规定了与这些条款（例如 Nmap OEM 许可证）不同的条款，您可以选择根据这些条款使用和重新分发 Nmap。
# 官方的 Nmap Windows 构建包括 Npcap 软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特殊许可就重新分发的单独许可条款的约束。因此，官方的 Nmap Windows 构建可能不得未经特殊许可（例如 Nmap OEM 许可证）重新分发。
# 我们提供此软件的源代码，因为我们相信用户有权根据自己的需求修改和重新分发它。但是，我们希望用户尊重我们的劳动成果，遵守我们的许可证条款，并尊重我们的商标。感谢您的合作。
# 导入 gi 模块
import gi

# 要求使用 Gtk 3.0 版本
gi.require_version("Gtk", "3.0")
# 从 gi.repository 中导入 Gtk 模块
from gi.repository import Gtk

# 从 radialnet.bestwidgets.buttons 模块中导入 BWStockButton 和 BWToggleStockButton 类
from radialnet.bestwidgets.buttons import BWStockButton, BWToggleStockButton
# 从 radialnet.gui.SaveDialog 模块中导入 SaveDialog 类
from radialnet.gui.SaveDialog import SaveDialog
# 从 radialnet.gui.Dialogs 模块中导入 AboutDialog 类
from radialnet.gui.Dialogs import AboutDialog
# 从 radialnet.gui.LegendWindow 模块中导入 LegendWindow 类
from radialnet.gui.LegendWindow import LegendWindow
# 从 radialnet.gui.HostsViewer 模块中导入 HostsViewer 类
from radialnet.gui.HostsViewer import HostsViewer
# 从 zenmapGUI.higwidgets.higdialogs 模块中导入 HIGAlertDialog 类
from zenmapGUI.higwidgets.higdialogs import HIGAlertDialog

# 定义常量 SHOW 和 HIDE
SHOW = True
HIDE = False

# 定义常量 REFRESH_RATE
REFRESH_RATE = 500

# 定义 ToolsMenu 类，继承自 Gtk.Menu 类
class ToolsMenu(Gtk.Menu):
    """
    工具菜单类，继承自 Gtk.Menu 类
    """
    # 初始化方法
    def __init__(self, radialnet):
        """
        初始化方法，接受 radialnet 参数
        """
        # 调用父类的初始化方法
        Gtk.Menu.__init__(self)

        # 将 radialnet 参数赋值给 self.radialnet
        self.radialnet = radialnet

        # 调用私有方法 __create_items()
        self.__create_items()
    # 创建项目菜单项
    def __create_items(self):
        # 创建一个带标签的图像菜单项，标签为'Hosts viewer'
        self.__hosts = Gtk.ImageMenuItem.new_with_label(_('Hosts viewer')
        # 连接菜单项的"activate"信号到__hosts_viewer_callback方法
        self.__hosts.connect("activate", self.__hosts_viewer_callback)
        # 创建一个图像对象
        self.__hosts_image = Gtk.Image()
        # 从图标库中设置图像对象的图标为Gtk.STOCK_INDEX，大小为Gtk.IconSize.MENU
        self.__hosts_image.set_from_stock(Gtk.STOCK_INDEX, Gtk.IconSize.MENU)
        # 将图像对象设置为菜单项的图像
        self.__hosts.set_image(self.__hosts_image)

        # 将菜单项添加到菜单
        self.append(self.__hosts)

        # 显示菜单项
        self.__hosts.show_all()

    # 处理"Hosts viewer"菜单项的回调方法
    def __hosts_viewer_callback(self, widget):
        # 创建一个HostsViewer窗口，传入radialnet.get_scanned_nodes()作为参数
        window = HostsViewer(self.radialnet.get_scanned_nodes())
        # 显示窗口
        window.show_all()
        # 将窗口置于最前
        window.set_keep_above(True)

    # 启用依赖项
    def enable_dependents(self):
        # 设置__hosts菜单项为可用状态
        self.__hosts.set_sensitive(True)

    # 禁用依赖项
    def disable_dependents(self):
        # 设置__hosts菜单项为不可用状态
        self.__hosts.set_sensitive(False)
class Toolbar(Gtk.Box):
    """
    工具栏类，继承自 Gtk.Box
    """
    def __init__(self, radialnet, window, control, fisheye):
        """
        初始化方法，接受 radialnet、window、control 和 fisheye 参数
        """
        Gtk.Box.__init__(self, orientation=Gtk.Orientation.HORIZONTAL)
        #self.set_style(gtk.TOOLBAR_BOTH_HORIZ)
        #self.set_tooltips(True)

        self.radialnet = radialnet

        self.__window = window
        self.__control_widget = control
        self.__fisheye_widget = fisheye

        self.__control_widget.show_all()
        self.__control_widget.set_no_show_all(True)
        self.__control_widget.hide()

        self.__fisheye_widget.show_all()
        self.__fisheye_widget.set_no_show_all(True)
        self.__fisheye_widget.hide()

        self.__save_chooser = None

        self.__create_widgets()

    def disable_controls(self):
        """
        禁用控件的方法
        """
        self.__control.set_sensitive(False)
        self.__fisheye.set_sensitive(False)
        self.__hosts_button.set_sensitive(False)
        self.__legend_button.set_sensitive(False)
        #self.__tools_menu.disable_dependents()

    def enable_controls(self):
        """
        启用控件的方法
        """
        self.__control.set_sensitive(True)
        self.__fisheye.set_sensitive(True)
        self.__hosts_button.set_sensitive(True)
        self.__legend_button.set_sensitive(True)
        #self.__tools_menu.enable_dependents()

    def __tools_callback(self, widget):
        """
        工具回调方法
        """
        self.__tools_menu.popup(None, None, None, 1, 0)

    def __hosts_viewer_callback(self, widget):
        """
        主机查看器回调方法
        """
        window = HostsViewer(self.radialnet.get_scanned_nodes())
        window.show_all()
        window.set_keep_above(True)
    # 定义保存图片的回调函数，当按钮被点击时触发
    def __save_image_callback(self, widget):
        # 如果保存对话框对象为空，创建一个新的保存对话框对象
        if self.__save_chooser is None:
            self.__save_chooser = SaveDialog()

        # 运行保存对话框，获取用户的响应
        response = self.__save_chooser.run()

        # 如果用户点击了保存按钮
        if response == Gtk.ResponseType.OK:
            # 获取用户选择的文件名和文件类型
            filename = self.__save_chooser.get_filename()
            filetype = self.__save_chooser.get_filetype()

            # 尝试将绘图保存到文件
            try:
                self.radialnet.save_drawing_to_file(filename, filetype)
            # 如果出现异常，创建一个错误提示对话框
            except Exception as e:
                alert = HIGAlertDialog(parent=self.__save_chooser,
                        type=Gtk.MessageType.ERROR,
                        message_format=_("Error saving snapshot"),
                        secondary_text=str(e))
                alert.run()
                alert.destroy()

        # 隐藏保存对话框
        self.__save_chooser.hide()

    # 控制面板回调函数，当控制按钮被点击时触发
    def __control_callback(self, widget=None):
        # 如果控制按钮被激活，显示控制面板
        if self.__control.get_active():
            self.__control_widget.show()
        # 否则隐藏控制面板
        else:
            self.__control_widget.hide()

    # 鱼眼效果回调函数，当鱼眼按钮被点击时触发
    def __fisheye_callback(self, widget=None):
        # 如果不在动画中
        if not self.radialnet.is_in_animation():
            # 如果鱼眼按钮被激活，激活鱼眼效果并显示鱼眼面板
            if self.__fisheye.get_active():
                self.__fisheye_widget.active_fisheye()
                self.__fisheye_widget.show()
            # 否则取消鱼眼效果并隐藏鱼眼面板
            else:
                self.__fisheye_widget.deactive_fisheye()
                self.__fisheye_widget.hide()

    # 图例回调函数，当图例按钮被点击时触发
    def __legend_callback(self, widget):
        # 创建图例窗口并显示
        self.__legend_window = LegendWindow()
        self.__legend_window.show_all()

    # 关于回调函数，当关于按钮被点击时触发
    def __about_callback(self, widget):
        # 创建关于对话框并显示
        self.__about_dialog = AboutDialog()
        self.__about_dialog.show_all()

    # 全屏回调函数，当全屏按钮被点击时触发
    def __fullscreen_callback(self, widget=None):
        # 如果全屏按钮被激活，将窗口设置为全屏模式
        if self.__fullscreen.get_active():
            self.__window.fullscreen()
        # 否则取消全屏模式
        else:
            self.__window.unfullscreen()
```