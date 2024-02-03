# `nmap\zenmap\radialnet\bestwidgets\windows.py`

```cpp
# 设置文件编码为 UTF-8

# ***********************IMPORTANT NMAP LICENSE TERMS************************
# *
# * Nmap安全扫描器由Nmap Software LLC（“Nmap项目”）（C）1996-2023年开发。Nmap也是Nmap项目的注册商标。
# *
# * 本程序根据Nmap公共源代码许可证（NPSL）的条款分发。适用于特定Nmap版本或源代码控制修订版的确切许可证文本包含在该版本的Nmap或源代码控制修订版的LICENSE文件中。有关更多Nmap版权/法律信息，请访问https://nmap.org/book/man-legal.html，有关NPSL许可证本身的更多信息，请访问https://nmap.org/npsl/。此标题总结了Nmap许可证的一些关键点，但不能替代实际许可证文本。
# *
# * 一般情况下，用户可以免费下载和使用Nmap，包括商业用途。可从https://nmap.org获取。
# *
# * Nmap许可证一般禁止公司在商业产品中使用和重新分发Nmap，但我们销售具有更宽松许可证和特殊功能的特殊Nmap OEM版本。请参阅https://nmap.org/oem/
# *
# * 如果您收到了书面的Nmap许可协议或合同，其中规定了与这些条款（例如Nmap OEM许可证）不同的条款，您可以选择根据那些条款使用和重新分发Nmap。
# *
# * 官方的Nmap Windows版本包括Npcap软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特殊许可证重新分发的单独许可证条款的约束。因此，官方的Nmap Windows版本可能不得在没有特殊许可证（例如Nmap OEM许可证）的情况下重新分发。
# *
# * 我们提供此软件的源代码，因为我们相信用户有权根据自己的需求修改和重新分发软件。我们希望通过这种方式促进网络安全和自由软件的发展。如果您对此有任何疑问，请联系legal@nmap.org。
# 导入 gi 模块
import gi
# 要求使用 Gtk 版本 3.0
gi.require_version("Gtk", "3.0")
# 从 gi.repository 中导入 Gtk 模块
from gi.repository import Gtk

# 定义一个常量，用于设置主要文本的标记
PRIMARY_TEXT_MARKUP = '<span weight="bold" size="larger">%s</span>'

# 定义一个名为 BWAlertDialog 的类，继承自 Gtk.MessageDialog
class BWAlertDialog(Gtk.MessageDialog):
    """
    """
    # 初始化函数，设置消息对话框的类型、按钮类型、主要文本和次要文本
    def __init__(self, parent=None, flags=0, type=Gtk.MessageType.INFO,
                 buttons=Gtk.ButtonsType.OK,
                 primary_text=None,
                 secondary_text=None):
    
        # 调用父类的初始化函数，设置消息对话框的父窗口、标志、消息类型和按钮类型
        Gtk.MessageDialog.__init__(self, parent=parent, flags=flags,
                                   message_type=type, buttons=buttons)
    
        # 连接 'response' 信号到 __destroy 方法
        self.connect('response', self.__destroy)
    
        # 设置消息对话框不可调整大小
        self.set_resizable(False)
    
        # 设置消息对话框的标题为 "Alert"
        self.set_title(_("Alert"))
        # 使用标记语言设置主要文本
        self.set_markup(PRIMARY_TEXT_MARKUP % primary_text)
    
        # 如果有次要文本，则格式化设置次要文本
        if secondary_text:
            self.format_secondary_text(secondary_text)
    
    # 销毁方法，关闭消息对话框
    def __destroy(self, dialog, id):
        """
        """
        self.destroy()
# 创建一个名为BWWindow的类，继承自Gtk.Window类
class BWWindow(Gtk.Window):
    """
    """

    # 初始化方法，接受一个type参数，默认为Gtk.WindowType.TOPLEVEL
    def __init__(self, type=Gtk.WindowType.TOPLEVEL):
        """
        """
        # 调用父类的初始化方法，传入type参数
        Gtk.Window.__init__(self, type=type)
        # 设置窗口的边框宽度为5
        self.set_border_width(5)

# 创建一个名为BWMainWindow的变量，指向Gtk.Window类
BWMainWindow = Gtk.Window
```