# `nmap\zenmap\radialnet\gui\NodeNotebook.py`

```cpp
# 设置文件编码为 UTF-8

# 重要提示：NMAP 许可证条款
# Nmap 安全扫描器由 Nmap Software LLC（“Nmap 项目”）（C）1996-2023 年开发。Nmap 也是 Nmap 项目的注册商标。
# 本程序根据 Nmap 公共源代码许可证（NPSL）的条款分发。适用于特定 Nmap 版本或源代码控制修订版的确切许可证文本包含在该版本或源代码控制修订版的 LICENSE 文件中。有关更多 Nmap 版权/法律信息，请访问 https://nmap.org/book/man-legal.html，有关 NPSL 许可证本身的更多信息，请访问 https://nmap.org/npsl/。本标题总结了 Nmap 许可证的一些关键点，但不能替代实际的许可证文本。
# Nmap 通常允许最终用户免费下载和使用，包括商业用途。可从 https://nmap.org 获取。
# Nmap 许可证通常禁止公司在商业产品中使用和重新分发 Nmap，但我们销售具有更宽松许可证和特殊功能的特殊 Nmap OEM 版本。请参阅 https://nmap.org/oem/
# 如果您收到了书面的 Nmap 许可协议或合同，其中规定了与这些条款（例如 Nmap OEM 许可证）不同的条款，您可以选择根据那些条款使用和重新分发 Nmap。
# 官方的 Nmap Windows 构建包括 Npcap 软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特殊许可证重新分发的单独许可证条款的约束。因此，官方的 Nmap Windows 构建可能不得未经特殊许可证（例如 Nmap OEM 许可证）重新分发。
# 我们提供此软件的源代码，因为我们相信用户有权根据自己的需求修改和重新分发它。
# 导入 gi 模块，确保使用的是 Gtk 3.0 版本
import gi

# 要求使用 Gtk 3.0 版本
gi.require_version("Gtk", "3.0")
# 从 gi.repository 中导入 Gtk, GObject, Pango 模块
from gi.repository import Gtk, GObject, Pango

# 从 radialnet.bestwidgets.boxes 模块中导入 BWVBox, BWHBox, BWScrolledWindow, BWTable 类
from radialnet.bestwidgets.boxes import BWVBox, BWHBox, BWScrolledWindow, BWTable
# 从 radialnet.bestwidgets.expanders 模块中导入 BWExpander 类
from radialnet.bestwidgets.expanders import BWExpander
# 从 radialnet.bestwidgets.labels 模块中导入 BWLabel, BWSectionLabel 类
from radialnet.bestwidgets.labels import BWLabel, BWSectionLabel
# 从 radialnet.bestwidgets.textview 模块中导入 BWTextEditor 类
from radialnet.bestwidgets.textview import BWTextEditor
# 导入 zenmapCore.I18N 模块，但未使用，可能是未使用的导入
import zenmapCore.I18N  # lgtm[py/unused-import]

# 定义常量 PORTS_HEADER，包含端口相关的标题
PORTS_HEADER = [
        _('Port'), _('Protocol'), _('State'), _('Service'), _('Method')]
# 定义常量 EXTRAPORTS_HEADER，包含额外端口相关的标题
EXTRAPORTS_HEADER = [_('Count'), _('State'), _('Reasons')]
# 定义服务状态对应的颜色
SERVICE_COLORS = {'open':            '#ffd5d5',  # noqa
                  'closed':          '#d5ffd5',  # noqa
                  'filtered':        '#ffffd5',  # noqa
                  'unfiltered':      '#ffd5d5',  # noqa
                  'open|filtered':   '#ffd5d5',  # noqa
                  'closed|filtered': '#d5ffd5'}  # noqa
# 未知服务状态的颜色
UNKNOWN_SERVICE_COLOR = '#d5d5d5'

# 定义跟踪信息的表头
TRACE_HEADER = ['TTL', 'RTT', 'IP', _('Hostname')]

# 定义跟踪信息的文本
TRACE_TEXT = _(
    "Traceroute on port <b>%s/%s</b> totalized <b>%d</b> known hops.")
# 无跟踪信息的文本
NO_TRACE_TEXT = _("No traceroute information available.")

# 定义跳数的颜色
HOP_COLOR = {'known':   '#ffffff',  # noqa
             'unknown': '#cccccc'}  # noqa

# 系统地址的文本格式
SYSTEM_ADDRESS_TEXT = "[%s] %s"

# 操作系统匹配的表头
OSMATCH_HEADER = ['%', _('Name'), _('DB Line')]
# 操作系统类别的表头
OSCLASS_HEADER = ['%', _('Vendor'), _('Type'), _('Family'), _('Version')]

# 使用的端口信息的文本格式
USED_PORTS_TEXT = "%d/%s %s"

# TCP序列的注释文本格式
TCP_SEQ_NOTE = _("""\
<b>*</b> TCP sequence <i>index</i> equal to %d and <i>difficulty</i> is "%s".\
""")

# 根据服务状态获取对应的颜色
def get_service_color(state):
    color = SERVICE_COLORS.get(state)
    if color is None:
        color = UNKNOWN_SERVICE_COLOR
    return color

# 定义一个继承自Gtk.Notebook的类NodeNotebook
class NodeNotebook(Gtk.Notebook):
    """
    """
    # 初始化方法
    def __init__(self, node):
        """
        """
        Gtk.Notebook.__init__(self)
        self.set_tab_pos(Gtk.PositionType.TOP)

        self.__node = node

        self.__create_widgets()

    # 创建小部件的方法
    def __create_widgets(self):
        """
        """
        # 创建服务页面
        self.__services_page = ServicesPage(self.__node)
        # 创建系统页面
        self.__system_page = SystemPage(self.__node)
        # 创建跟踪页面
        self.__trace_page = TraceroutePage(self.__node)

        # 将页面添加到Notebook中
        self.append_page(self.__system_page, BWLabel(_('General')))
        self.append_page(self.__services_page, BWLabel(_('Services')))
        self.append_page(self.__trace_page, BWLabel(_('Traceroute')))

# 定义一个继承自Gtk.Notebook的类ServicesPage
class ServicesPage(Gtk.Notebook):
    """
    """
    # 初始化方法，接受一个节点参数
    def __init__(self, node):
        # 调用父类的初始化方法
        Gtk.Notebook.__init__(self)
        # 设置边框宽度
        self.set_border_width(6)
        # 设置标签位置
        self.set_tab_pos(Gtk.PositionType.TOP)
    
        # 保存传入的节点参数
        self.__node = node
        # 创建一个等宽字体的描述对象
        self.__font = Pango.FontDescription('Monospace')
    
        # 调用私有方法创建小部件
        self.__create_widgets()
    
    # 私有方法，用于改变文本值
    def __change_text_value(self, widget):
        # 获取下拉框的当前选中项
        id = self.__select_combobox.get_active()
        # 根据选中项的索引，设置文本编辑器的文本内容
        self.__texteditor.bw_set_text(self.__text[id])
# 创建 SystemPage 类，继承自 BWScrolledWindow
class SystemPage(BWScrolledWindow):
    """
    """

    # 初始化方法，接受一个节点参数
    def __init__(self, node):
        """
        """
        # 调用父类的初始化方法
        BWScrolledWindow.__init__(self)

        # 设置私有属性 __node 为传入的节点参数
        self.__node = node
        # 设置私有属性 __font 为 Monospace 字体描述
        self.__font = Pango.FontDescription('Monospace')

        # 调用私有方法创建小部件
        self.__create_widgets()

# 创建 TraceroutePage 类，继承自 BWVBox
class TraceroutePage(BWVBox):
    """
    """

    # 初始化方法，接受一个节点参数
    def __init__(self, node):
        """
        """
        # 调用父类的初始化方法
        BWVBox.__init__(self)
        # 设置边框宽度为 6
        self.set_border_width(6)

        # 设置私有属性 __node 为传入的节点参数
        self.__node = node

        # 调用私有方法创建小部件
        self.__create_widgets()
```