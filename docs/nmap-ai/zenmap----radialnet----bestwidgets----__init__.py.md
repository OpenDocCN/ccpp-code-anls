# `nmap\zenmap\radialnet\bestwidgets\__init__.py`

```
# 设置文件编码为 UTF-8

# 重要提示：NMAP 许可证条款
# Nmap 安全扫描器由 Nmap 软件有限责任公司（“Nmap 项目”）（C）1996-2023 年开发。Nmap 也是 Nmap 项目的注册商标。
# 本程序根据 Nmap 公共源代码许可证（NPSL）的条款分发。适用于特定 Nmap 发行版或源代码控制修订版的确切许可证文本包含在该版本的 Nmap 或源代码控制修订版的 LICENSE 文件中。有关更多 Nmap 版权/法律信息，请访问 https://nmap.org/book/man-legal.html，有关 NPSL 许可证本身的更多信息，请访问 https://nmap.org/npsl/。本标题总结了 Nmap 许可证的一些关键点，但不能替代实际的许可证文本。
# Nmap 通常允许最终用户自行下载和使用，包括商业用途。可从 https://nmap.org 获取。
# Nmap 许可证通常禁止公司在商业产品中使用和重新分发 Nmap，但我们销售具有更宽松许可证和特殊功能的特殊 Nmap OEM 版本。请参阅 https://nmap.org/oem/
# 如果您收到了书面的 Nmap 许可协议或合同，其中规定了与这些条款（例如 Nmap OEM 许可证）不同的条款，您可以选择根据那些条款使用和重新分发 Nmap。
# 官方的 Nmap Windows 构建包括 Npcap 软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特殊许可就重新分发的单独许可条款的约束。因此，官方的 Nmap Windows 构建可能不得未经特殊许可（例如 Nmap OEM 许可证）重新分发。
# 我们提供此软件的源代码，因为我们相信用户有权根据自己的需求修改和重新分发它。但是，请务必遵守适用的许可证条款。
# 导入 gi 模块
import gi

# 要求 gi 模块的版本为 "Gtk" 的 "3.0"
gi.require_version("Gtk", "3.0")

# 从 gi.repository 模块中导入 Gtk
from gi.repository import Gtk

# 获取 gi 模块的版本信息
gtk_version_major, gtk_version_minor, gtk_version_release = gi.version_info

# 以下是被注释掉的代码，暂时不需要解释
# from boxes import BWHBox, BWVBox, BWTable, BWStatusbar, BWScrolledWindow
# from buttons import BWStockButton, BWToggleStockButton
# from comboboxes import BWChangeableComboBoxEntry
# from expanders import BWExpander
# from frames import BWFrame
# from labels import BWLabel, BWSectionLabel
# from textview import BWTextView, BWTextEditor
# from windows import BWWindow, BWMainWindow, BWAlertDialog
```