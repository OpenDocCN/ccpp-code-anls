# `nmap\zenmap\radialnet\gui\NodeWindow.py`

```
# 设置文件编码为 UTF-8
# 重要提示：NMAP 许可证条款
# Nmap 安全扫描器由 Nmap Software LLC（“Nmap 项目”）（C）1996-2023 年开发。Nmap 也是 Nmap 项目的注册商标。
# 本程序根据 Nmap 公共源代码许可证（NPSL）的条款分发。适用于特定 Nmap 版本或源代码控制修订版的确切许可证文本包含在该版本的 Nmap 或源代码控制修订版的 LICENSE 文件中。有关更多 Nmap 版权/法律信息，请访问 https://nmap.org/book/man-legal.html，有关 NPSL 许可证本身的更多信息，请访问 https://nmap.org/npsl/。此标题总结了 Nmap 许可证的一些关键点，但不能替代实际的许可证文本。
# 一般情况下，用户可以免费下载和使用 Nmap，包括商业用途。可从 https://nmap.org 获取。
# Nmap 许可证一般禁止公司在商业产品中使用和重新分发 Nmap，但我们销售一个特殊的 Nmap OEM 版本，具有更宽松的许可证和专门用于此目的的特殊功能。请参阅 https://nmap.org/oem/
# 如果您收到了一份书面的 Nmap 许可协议或合同，其中规定了与这些条款（例如 Nmap OEM 许可证）不同的条款，您可以选择根据那些条款使用和重新分发 Nmap。
# 官方的 Nmap Windows 构建包括 Npcap 软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特殊许可证重新分发的单独许可证条款的约束。因此，官方的 Nmap Windows 构建可能不得未经特殊许可证（例如 Nmap OEM 许可证）重新分发。
# 我们提供此软件的源代码，因为我们相信用户有权根据其需求进行修改和定制。我们鼓励用户向 Nmap 项目贡献他们的改进。
# 导入gi模块，需要使用其中的Gtk、Gdk和Pango
import gi

# 要求使用的Gtk版本为3.0
gi.require_version("Gtk", "3.0")
# 从gi.repository中导入Gtk、Gdk、Pango
from gi.repository import Gtk, Gdk, Pango

# 导入radialnet.util.drawing模块中的所有内容
import radialnet.util.drawing as drawing

# 从radialnet.bestwidgets.windows模块中导入BWWindow
from radialnet.bestwidgets.windows import BWWindow
# 从radialnet.bestwidgets.boxes模块中导入BWVBox、BWHBox
from radialnet.bestwidgets.boxes import BWVBox, BWHBox
# 从radialnet.bestwidgets.labels模块中导入BWSectionLabel
from radialnet.bestwidgets.labels import BWSectionLabel
# 从radialnet.gui.Image模块中导入Application
from radialnet.gui.Image import Application
# 从radialnet.gui.NodeNotebook模块中导入NodeNotebook
from radialnet.gui.NodeNotebook import NodeNotebook

# 定义常量DIMENSION_NORMAL为(600, 400)
DIMENSION_NORMAL = (600, 400)

# 定义NodeWindow类，继承自BWWindow类
class NodeWindow(BWWindow):
    """
    """
    # 初始化方法，接受节点和位置参数
    def __init__(self, node, position):
        # 调用父类的初始化方法，创建顶层窗口
        BWWindow.__init__(self, Gtk.WindowType.TOPLEVEL)
        # 移动窗口到指定位置
        self.move(position[0], position[1])
        # 设置窗口默认大小
        self.set_default_size(DIMENSION_NORMAL[0], DIMENSION_NORMAL[1])

        # 保存节点信息
        self.__node = node

        # 设置标题字体为 Monospace 加粗
        self.__title_font = Pango.FontDescription('Monospace Bold')

        # 创建应用程序图标对象
        self.__icon = Application()
        # 调用创建小部件的方法
        self.__create_widgets()

    # 创建小部件的方法
    def __create_widgets(self):
        # 创建内容垂直布局容器
        self.__content = BWVBox()
        # 创建头部水平布局容器
        self.__head = BWHBox(spacing=2)

        # 创建节点笔记本对象
        self.__notebook = NodeNotebook(self.__node)

        # 创建头部元素

        # 使用节点评分颜色创建图标
        self.__color_box = Gtk.EventBox()
        self.__color_image = Gtk.Image()
        self.__color_image.set_from_file(self.__icon.get_icon('border'))
        self.__color_box.add(self.__color_image)
        self.__color_box.set_size_request(15, 15)
        r, g, b = drawing.cairo_to_gdk_color(
                self.__node.get_draw_info('color'))
        self.__color_box.modify_bg(Gtk.StateType.NORMAL, Gdk.Color(r, g, b))

        # 使用节点的 IP 和主机名创建标题
        self.__title = self.__node.get_host().get_hostname()

        # 设置窗口标题为节点的主机名
        self.set_title(self.__title)

        # 创建标题标签
        self.__title_label = BWSectionLabel(self.__title)
        self.__title_label.modify_font(self.__title_font)

        # 将头部元素添加到头部布局容器
        self.__head.bw_pack_start_noexpand_nofill(self.__color_box)
        self.__head.bw_pack_start_expand_fill(self.__title_label)

        # 将所有元素添加到内容布局容器
        self.__content.bw_pack_start_noexpand_nofill(self.__head)
        self.__content.bw_pack_start_expand_fill(self.__notebook)

        # 将内容添加到窗口
        self.add(self.__content)
```