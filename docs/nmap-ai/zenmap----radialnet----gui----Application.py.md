# `nmap\zenmap\radialnet\gui\Application.py`

```
# 设置文件编码为 UTF-8

# 重要提示：NMAP 许可证条款
# Nmap 安全扫描器由 Nmap 软件有限责任公司（“Nmap 项目”）（C）1996-2023 年开发。Nmap 也是 Nmap 项目的注册商标。
# 本程序根据 Nmap 公共源代码许可证（NPSL）的条款分发。适用于特定 Nmap 版本或源代码控制修订版的确切许可证文本包含在该版本的 Nmap 或源代码控制修订版的 LICENSE 文件中。有关更多 Nmap 版权/法律信息，请访问 https://nmap.org/book/man-legal.html，有关 NPSL 许可证本身的更多信息，请访问 https://nmap.org/npsl/。本标题总结了 Nmap 许可证的一些关键点，但不能替代实际的许可证文本。
# Nmap 通常允许最终用户免费下载和使用，包括商业用途。可从 https://nmap.org 获取。
# Nmap 许可证通常禁止公司在商业产品中使用和重新分发 Nmap，但我们销售一个特殊的 Nmap OEM 版本，具有更宽松的许可证和专门用于此目的的特殊功能。请参阅 https://nmap.org/oem/
# 如果您收到了书面的 Nmap 许可协议或合同，其中规定了与这些条款（如 Nmap OEM 许可证）不同的条款，您可以选择根据那些条款使用和重新分发 Nmap。
# 官方的 Nmap Windows 构建包括 Npcap 软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特殊许可证重新分发的单独许可证条款的约束。因此，官方的 Nmap Windows 构建可能不得未经特殊许可证（如 Nmap OEM 许可证）重新分发。
# 我们提供此软件的源代码，因为我们相信用户有权根据其需求进行修改和定制。
# 导入gi模块，用于与GTK+库进行集成
import gi
# 要求使用GTK+ 3.0版本
gi.require_version("Gtk", "3.0")
# 从gi.repository中导入Gtk模块
from gi.repository import Gtk

# 从radialnet.util.integration模块中导入make_graph_from_nmap_parser函数
from radialnet.util.integration import make_graph_from_nmap_parser
# 从radialnet.core.Info模块中导入INFO对象
from radialnet.core.Info import INFO
# 从radialnet.core.XMLHandler模块中导入XMLReader类
from radialnet.core.XMLHandler import XMLReader
# 从radialnet.gui.ControlWidget模块中导入ControlWidget和ControlFisheye类
from radialnet.gui.ControlWidget import ControlWidget, ControlFisheye
# 从radialnet.gui.Toolbar模块中导入Toolbar类
from radialnet.gui.Toolbar import Toolbar
# 从radialnet.gui.Image模块中导入Pixmaps类
from radialnet.gui.Image import Pixmaps
# 从radialnet包中导入RadialNet模块
import radialnet.gui.RadialNet as RadialNet
# 从radialnet.bestwidgets.windows模块中导入BWMainWindow和BWAlertDialog类
from radialnet.bestwidgets.windows import BWMainWindow, BWAlertDialog
# 从radialnet.bestwidgets.boxes模块中导入BWHBox、BWVBox和BWStatusbar类
from radialnet.bestwidgets.boxes import BWHBox, BWVBox, BWStatusbar

# 定义DIMENSION常量，表示窗口尺寸
DIMENSION = (640, 480)

# 定义Application类，继承自BWMainWindow类
class Application(BWMainWindow):
    """
    """
    # 初始化方法，用于创建窗口对象
    def __init__(self):
        """
        """
        # 调用父类的初始化方法
        BWMainWindow.__init__(self)
        # 设置窗口的默认大小
        self.set_default_size(DIMENSION[0], DIMENSION[1])

        # 设置窗口的图标
        self.set_icon(Pixmaps().get_pixbuf('logo'))

        # 调用私有方法创建窗口的各个部件
        self.__create_widgets()

    # 创建窗口的各个部件
    def __create_widgets(self):
        """
        """
        # 创建水平布局容器
        self.__hbox = BWHBox(spacing=0)
        # 创建垂直布局容器
        self.__vbox = BWVBox(spacing=0)

        # 创建 RadialNet 对象
        self.__radialnet = RadialNet.RadidalNet(RadialNet.LAYOUT_WEIGHTED)
        # 创建 ControlWidget 对象
        self.__control = ControlWidget(self.__radialnet)
        # 创建 ControlFisheye 对象
        self.__fisheye = ControlFisheye(self.__radialnet)
        # 创建 Toolbar 对象
        self.__toolbar = Toolbar(self.__radialnet, self, self.__control, self.__fisheye)
        # 创建状态栏对象
        self.__statusbar = BWStatusbar()

        # 将 RadialNet 对象添加到水平布局容器中并设置为扩展填充
        self.__hbox.bw_pack_start_expand_fill(self.__radialnet)
        # 将 ControlWidget 对象添加到水平布局容器中并设置为不扩展不填充
        self.__hbox.bw_pack_start_noexpand_nofill(self.__control)

        # 将 Toolbar 对象添加到垂直布局容器中并设置为不扩展不填充
        self.__vbox.bw_pack_start_noexpand_nofill(self.__toolbar)
        # 将水平布局容器添加到垂直布局容器中并设置为扩展填充
        self.__vbox.bw_pack_start_expand_fill(self.__hbox)
        # 将 ControlFisheye 对象添加到垂直布局容器中并设置为不扩展不填充
        self.__vbox.bw_pack_start_noexpand_nofill(self.__fisheye)
        # 将状态栏对象添加到垂直布局容器中并设置为不扩展不填充
        self.__vbox.bw_pack_start_noexpand_nofill(self.__statusbar)

        # 将垂直布局容器添加到窗口中
        self.add(self.__vbox)
        # 设置窗口的标题
        self.set_title(" ".join([INFO['name'], INFO['version']]))
        # 设置窗口的位置为居中
        self.set_position(Gtk.WindowPosition.CENTER)
        # 显示窗口中的所有部件
        self.show_all()
        # 关联窗口的关闭事件到 Gtk 的主循环退出
        self.connect('destroy', Gtk.main_quit)

        # 设置 RadialNet、ControlWidget 和 ControlFisheye 对象不在显示所有部件
        self.__radialnet.set_no_show_all(True)
        self.__control.set_no_show_all(True)
        self.__fisheye.set_no_show_all(True)

        # 隐藏 RadialNet、ControlWidget 和 ControlFisheye 对象
        self.__radialnet.hide()
        self.__control.hide()
        self.__fisheye.hide()
        # 禁用 Toolbar 中的控件
        self.__toolbar.disable_controls()
    # 解析给定的 Nmap XML 文件
    def parse_nmap_xml_file(self, file):
        """
        """
        try:
            # 使用 XMLReader 对象解析文件
            self.__parser = XMLReader(file)
            self.__parser.parse()
        except Exception as e:
            # 如果出现异常，显示错误对话框并返回 False
            text = 'It is not possible open file %s: %s' % (file, e)
            alert = BWAlertDialog(self,
                                  primary_text='Error opening file.',
                                  secondary_text=text)
            alert.show_all()
            return False
        # 清空 radialnet
        self.__radialnet.set_empty()
        # 从解析器创建图形并设置给 radialnet
        self.__radialnet.set_graph(make_graph_from_nmap_parser(self.__parser))
        # 显示 radialnet
        self.__radialnet.show()
        # 启用工具栏控件
        self.__toolbar.enable_controls()
        # 返回 True
        return True

    # 启动应用程序的主循环
    def start(self):
        """
        """
        Gtk.main()
```