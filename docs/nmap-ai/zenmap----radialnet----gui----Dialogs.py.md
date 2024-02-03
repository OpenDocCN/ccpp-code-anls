# `nmap\zenmap\radialnet\gui\Dialogs.py`

```cpp
# 设置文件编码为 UTF-8

# 重要提示：NMAP 许可证条款
# Nmap 安全扫描器是 Nmap 软件有限责任公司（“Nmap 项目”）1996-2023 年的版权所有。
# Nmap 也是 Nmap 项目的注册商标。
# 本程序根据 Nmap 公共源代码许可证（NPSL）的条款分发。适用于特定 Nmap 版本或源代码控制修订版的确切许可证文本包含在该版本的 Nmap 或源代码控制修订版的 LICENSE 文件中。
# 更多关于 Nmap 版权/法律信息，请访问 https://nmap.org/book/man-legal.html，有关 NPSL 许可证本身的更多信息，请访问 https://nmap.org/npsl/。本标题总结了 Nmap 许可证的一些关键点，但不能替代实际的许可证文本。
# Nmap 通常允许最终用户免费下载和使用，包括商业用途。可从 https://nmap.org 获取。
# Nmap 许可证通常禁止公司在商业产品中使用和重新分发 Nmap，但我们销售一个特殊的 Nmap OEM 版本，具有更宽松的许可证和专门用于此目的的特殊功能。请参阅 https://nmap.org/oem/
# 如果您收到了书面的 Nmap 许可协议或合同，其中规定了与这些条款（如 Nmap OEM 许可证）不同的条款，您可以选择根据那些条款使用和重新分发 Nmap。
# 官方的 Nmap Windows 构建包括 Npcap 软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特别许可重新分发的单独许可条款的约束。因此，官方的 Nmap Windows 构建可能不得未经特别许可（如 Nmap OEM 许可证）重新分发。
# 我们提供此软件的源代码，因为我们相信用户有权根据自己的需求修改和重新分发它。
# 导入gi模块，用于使用GTK+ 3.0库
import gi

# 要求使用GTK+ 3.0版本
gi.require_version("Gtk", "3.0")
from gi.repository import Gtk

# 导入自定义的INFO和Pixmaps类
from radialnet.core.Info import INFO
from radialnet.gui.Image import Pixmaps

# 创建关于对话框的类
class AboutDialog(Gtk.AboutDialog):
    """
    """
    # 初始化方法
    def __init__(self):
        """
        """
        # 调用父类的初始化方法
        Gtk.AboutDialog.__init__(self)

        # 设置关于对话框的名称、版本、网站、作者、许可证和版权信息
        self.set_name(INFO['name'])
        self.set_version(INFO['version'])
        self.set_website(INFO['website'])
        self.set_authors(INFO['authors'])
        self.set_license(INFO['license'])
        self.set_copyright(INFO['copyright'])

        # 设置关于对话框的logo
        self.set_logo(Pixmaps().get_pixbuf('logo'))

        # 连接响应事件到销毁方法
        self.connect('response', self.__destroy)
    # 定义一个私有方法，用于销毁对话框
    def __destroy(self, dialog, id):
        # 调用父类的销毁方法，销毁对话框
        self.destroy()
```