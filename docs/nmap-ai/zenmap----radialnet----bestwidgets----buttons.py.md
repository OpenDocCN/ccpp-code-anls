# `nmap\zenmap\radialnet\bestwidgets\buttons.py`

```cpp
# 设置文件编码为 UTF-8

# 重要提示：NMAP 许可证条款
# Nmap 安全扫描器由 Nmap 软件有限责任公司（“Nmap 项目”）（C）1996-2023 年开发。Nmap 也是 Nmap 项目的注册商标。
# 本程序根据 Nmap 公共源代码许可证（NPSL）的条款分发。适用于特定 Nmap 发行版或源代码控制修订版的确切许可证文本包含在该版本的 Nmap 或源代码控制修订版的 LICENSE 文件中。有关更多 Nmap 版权/法律信息，请访问 https://nmap.org/book/man-legal.html，有关 NPSL 许可证本身的更多信息，请访问 https://nmap.org/npsl/。本标题总结了 Nmap 许可证的一些关键点，但不能替代实际的许可证文本。
# Nmap 通常允许最终用户自由下载和使用，包括商业用途。可从 https://nmap.org 获取。
# Nmap 许可证通常禁止公司在商业产品中使用和重新分发 Nmap，但我们销售具有更宽松许可证和特殊功能的特殊 Nmap OEM 版本。请参阅 https://nmap.org/oem/
# 如果您收到了书面的 Nmap 许可协议或合同，其中规定了与这些条款（例如 Nmap OEM 许可证）不同的条款，您可以选择根据那些条款使用和重新分发 Nmap。
# 官方的 Nmap Windows 构建包括 Npcap 软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特殊许可证重新分发的单独许可证条款的约束。因此，官方的 Nmap Windows 构建可能不得在没有特殊许可证（例如 Nmap OEM 许可证）的情况下重新分发。
# 我们提供此软件的源代码，因为我们相信用户有权根据自己的需求修改和重新分发它。但是，请注意，这并不意味着您可以随意修改和重新分发 Nmap。请务必详细阅读 NPSL 许可证文本以了解您的权利和义务。
# 导入gi模块，确保使用的是Gtk 3.0版本
import gi

gi.require_version("Gtk", "3.0")
from gi.repository import Gtk

# 创建一个名为BWStockButton的类，继承自Gtk.Button
class BWStockButton(Gtk.Button):
    """
    """
    # 初始化方法，接受stock、text和size三个参数
    def __init__(self, stock, text=None, size=Gtk.IconSize.BUTTON):
        """
        """
        # 调用父类的初始化方法，设置按钮的标签为text
        Gtk.Button.__init__(self, label=text)

        # 设置私有属性__size为传入的size参数
        self.__size = size

        # 创建一个Gtk.Image对象，并使用stock和size设置图标
        self.__image = Gtk.Image()
        self.__image.set_from_stock(stock, self.__size)
        # 将图标设置为按钮的图像
        self.set_image(self.__image)

# 创建一个名为BWToggleStockButton的类，继承自Gtk.ToggleButton
class BWToggleStockButton(Gtk.ToggleButton):
    """
    """
    # 初始化方法，接受股票名称、文本和图标大小作为参数
    def __init__(self, stock, text=None, size=Gtk.IconSize.BUTTON):
        # 调用父类的初始化方法，设置按钮的标签为传入的文本
        Gtk.ToggleButton.__init__(self, label=text)
    
        # 设置私有属性__size为传入的图标大小
        self.__size = size
    
        # 创建一个图像对象
        self.__image = Gtk.Image()
        # 从图标主题中获取图标，并设置图标大小
        self.__image.set_from_stock(stock, self.__size)
        # 将图像对象设置为按钮的图标
        self.set_image(self.__image)
```