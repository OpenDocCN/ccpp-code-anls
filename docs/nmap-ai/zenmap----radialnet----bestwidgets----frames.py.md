# `nmap\zenmap\radialnet\bestwidgets\frames.py`

```
# 设置文件编码为 UTF-8

# 重要提示：NMAP 许可证条款
# Nmap 安全扫描器由 Nmap 软件有限责任公司（“Nmap 项目”）（C）1996-2023 年。Nmap 也是 Nmap 项目的注册商标。
# 本程序根据 Nmap 公共源代码许可证（NPSL）的条款分发。适用于特定 Nmap 发行版或源代码控制修订版的确切许可证文本包含在该版本的 Nmap 或源代码控制修订版的 LICENSE 文件中。有关更多 Nmap 版权/法律信息，请访问 https://nmap.org/book/man-legal.html，有关 NPSL 许可证本身的更多信息，请访问 https://nmap.org/npsl/。本标题总结了 Nmap 许可证的一些关键点，但不能替代实际的许可证文本。
# 一般情况下，用户可以免费下载和使用 Nmap，包括商业用途。可从 https://nmap.org 获取。
# Nmap 许可证一般禁止公司在商业产品中使用和重新分发 Nmap，但我们销售一个特殊的 Nmap OEM 版本，具有更宽松的许可证和特殊功能。请参阅 https://nmap.org/oem/
# 如果您收到了书面的 Nmap 许可协议或合同，其中规定了与这些条款（如 Nmap OEM 许可证）不同的条款，您可以选择根据那些条款使用和重新分发 Nmap。
# 官方的 Nmap Windows 构建包括 Npcap 软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特殊许可证重新分发的单独许可证条款的约束。因此，官方的 Nmap Windows 构建可能不得未经特殊许可证（如 Nmap OEM 许可证）重新分发。
# 我们提供此软件的源代码，因为我们相信用户有权根据其需求进行修改和重新分发。如果您有任何疑问，请联系 legal@nmap.org。
# 导入gi模块，确保使用的是Gtk 3.0版本
import gi

gi.require_version("Gtk", "3.0")
from gi.repository import Gtk, Gdk

# 创建一个名为BWFrame的类，继承自Gtk.Frame
class BWFrame(Gtk.Frame):
    """
    """
    # 初始化方法，设置边框宽度和阴影类型
    def __init__(self, label=''):
        """
        """
        Gtk.Frame.__init__(self)

        self.set_border_width(3)
        self.set_shadow_type(Gtk.ShadowType.NONE)

        # 创建一个对齐对象，并设置填充
        self.__alignment = Gtk.Alignment.new(0, 0, 1, 1)
        self.__alignment.set_padding(12, 0, 24, 0)

        # 将对齐对象添加到框架中
        self.add(self.__alignment)

        # 调用bw_set_label方法设置标签内容
        self.bw_set_label(label)

    # 设置标签内容的方法
    def bw_set_label(self, label):
        """
        """
        # 使用标签内容设置框架的标签，并启用标记语言
        self.set_label("<b>" + label + "</b>")
        self.get_label_widget().set_use_markup(True)
    # 将给定的小部件添加到对齐容器中
    def bw_add(self, widget):
        self.__alignment.add(widget)
    
    # 从对齐容器中移除给定的小部件
    def bw_remove(self, widget):
        self.__alignment.remove(widget)
```