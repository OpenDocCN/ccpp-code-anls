# `nmap\zenmap\radialnet\bestwidgets\expanders.py`

```cpp
# 设置文件编码为 UTF-8

# 重要提示：NMAP 许可证条款
# Nmap 安全扫描器是 Nmap 软件有限责任公司（“Nmap 项目”）1996-2023 年的版权所有。
# Nmap 也是 Nmap 项目的注册商标。
# 本程序根据 Nmap 公共源代码许可证（NPSL）的条款分发。适用于特定 Nmap 版本或源代码控制修订版的确切许可证文本包含在该版本的 Nmap 或源代码控制修订版的 LICENSE 文件中。
# 更多关于 Nmap 版权/法律信息，请访问 https://nmap.org/book/man-legal.html，有关 NPSL 许可证本身的更多信息，请访问 https://nmap.org/npsl/。本页概述了 Nmap 许可证的一些关键点，但不能替代实际的许可证文本。
# Nmap 通常允许最终用户自由下载和使用，包括商业用途。可从 https://nmap.org 获取。
# Nmap 许可证通常禁止公司在商业产品中使用和重新分发 Nmap，但我们销售一个特殊的 Nmap OEM 版本，具有更宽松的许可证和专门用于此目的的特殊功能。请参阅 https://nmap.org/oem/
# 如果您收到了书面的 Nmap 许可协议或合同，其中规定了与这些条款（如 Nmap OEM 许可证）不同的条款，您可以选择根据那些条款使用和重新分发 Nmap。
# 官方的 Nmap Windows 构建包括 Npcap 软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特殊许可证重新分发的单独许可证条款的约束。因此，官方的 Nmap Windows 构建可能不得未经特殊许可证（如 Nmap OEM 许可证）重新分发。
# 我们提供此软件的源代码，因为我们相信用户有权根据自己的需求修改和重新分发它。
# 导入 gi 模块，确保使用的是 Gtk 3.0 版本
import gi

# 要求使用 Gtk 3.0 版本
gi.require_version("Gtk", "3.0")
from gi.repository import Gtk
from radialnet.bestwidgets.labels import BWSectionLabel

# 创建一个名为 BWExpander 的类，继承自 Gtk.Expander
class BWExpander(Gtk.Expander):
    """
    """
    # 初始化方法，接受一个标签参数
    def __init__(self, label=''):
        """
        """
        # 调用父类的初始化方法
        Gtk.Expander.__init__(self)

        # 创建一个 BWSectionLabel 对象作为标签
        self.__label = BWSectionLabel(label)
        # 将标签设置为 Expander 的标签部件
        self.set_label_widget(self.__label)

        # 创建一个对齐对象
        self.__alignment = Gtk.Alignment.new(0, 0, 1, 1)
        # 设置对齐对象的填充
        self.__alignment.set_padding(12, 0, 24, 0)

        # 将对齐对象添加到 Expander 中
        self.add(self.__alignment)

    # 设置标签文本的方法
    def bw_set_label_text(self, text):
        """
        """
        # 调用 BWSectionLabel 对象的设置文本方法
        self.__label.bw_set_text(text)
    # 添加一个子部件到对齐容器中
    def bw_add(self, widget):
        # 检查对齐容器中是否已经有子部件，如果有则移除
        if len(self.__alignment.get_children()) > 0:
            self.__alignment.remove(self.__alignment.get_children()[0])
    
        # 向对齐容器中添加子部件
        self.__alignment.add(widget)
    
    # 设置对齐容器的填充值为0，即无填充
    def bw_no_padding(self):
        self.__alignment.set_padding(0, 0, 0, 0)
```