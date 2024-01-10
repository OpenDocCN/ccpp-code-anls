# `nmap\zenmap\zenmapGUI\FilterBar.py`

```
# 导入 gi 模块
import gi
# 要求使用 Gtk 3.0 版本
gi.require_version("Gtk", "3.0")
# 从 gi.repository 中导入 Gtk 和 GObject 模块
from gi.repository import Gtk, GObject

# 从 zenmapGUI.higwidgets.higboxes 模块中导入 HIGHBox 类
from zenmapGUI.higwidgets.higboxes import HIGHBox
# 从 zenmapGUI.higwidgets.higlabels 模块中导入 HintWindow 类
from zenmapGUI.higwidgets.higlabels import HintWindow

# 创建 FilterBar 类，继承自 HIGHBox 类
class FilterBar(HIGHBox):
    """This is the bar that appears while the host filter is active. It allows
    entering a string that restricts the set of visible hosts."""

    # 定义信号 "changed"
    __gsignals__ = {
        "changed": (GObject.SignalFlags.RUN_FIRST, GObject.TYPE_NONE, ())
    }

    # 初始化方法
    def __init__(self):
        # 调用父类的初始化方法
        HIGHBox.__init__(self)
        # 创建 Gtk.Label 对象
        self.information_label = Gtk.Label()
        # 创建 Gtk.Entry 对象
        self.entry = Gtk.Entry()

        # 将信息标签添加到容器中
        self.pack_start(self.information_label, False, True, 0)
        self.information_label.show()

        # 创建标签对象
        label = Gtk.Label.new(_("Host Filter:"))
        # 将标签添加到容器中
        self.pack_start(label, False, True, 0)
        label.show()

        # 将输入框添加到容器中
        self.pack_start(self.entry, True, True, 0)
        self.entry.show()

        # 创建帮助按钮
        help_button = Gtk.Button()
        # 创建图标对象
        icon = Gtk.Image()
        # 设置图标来源为 Gtk.STOCK_INFO
        icon.set_from_stock(Gtk.STOCK_INFO, Gtk.IconSize.BUTTON)
        # 将图标添加到帮助按钮中
        help_button.add(icon)
        # 连接按钮的 "clicked" 信号到 _help_button_clicked 方法
        help_button.connect("clicked", self._help_button_clicked)
        # 将帮助按钮添加到容器中
        self.pack_start(help_button, False, True, 0)
        help_button.show_all()

        # 连接输入框的 "changed" 信号到 lambda 函数，触发 "changed" 信号
        self.entry.connect("changed", lambda x: self.emit("changed"))

    # 获取焦点的方法
    def grab_focus(self):
        self.entry.grab_focus()

    # 获取过滤字符串的方法
    def get_filter_string(self):
        return self.entry.get_text()

    # 设置过滤字符串的方法
    def set_filter_string(self, filter_string):
        return self.entry.set_text(filter_string)

    # 设置信息文本的方法
    def set_information_text(self, text):
        self.information_label.set_text(text)

    # 帮助按钮点击事件的方法
    def _help_button_clicked(self, button):
        # 创建 HintWindow 对象，显示帮助文本
        hint_window = HintWindow(HELP_TEXT)
        hint_window.show_all()

# 帮助文本
HELP_TEXT = _("""\
Entering the text into the search performs a <b>keyword search</b> - the \
search string is matched against every aspect of the host.

To refine the search, you can use <b>operators</b> to search only \
# 定义一些特定字段及其缩写的含义
<b>target: (t:)</b> - 用户提供的目标，或者是一个反向DNS结果。
<b>os:</b> - 所有与操作系统相关的字段。
<b>open: (op:)</b> - 在扫描中发现的开放端口。
<b>closed: (cp:)</b> - 在扫描中发现的关闭端口。
<b>filtered: (fp:)</b> - 在扫描中发现的被过滤的端口。
<b>unfiltered: (ufp:)</b> - 在扫描中发现的未被过滤的端口（例如使用ACK扫描）。
<b>open|filtered: (ofp:)</b> - 处于“开放|被过滤”状态的端口。
<b>closed|filtered: (cfp:)</b> - 处于“关闭|被过滤”状态的端口。
<b>service: (s:)</b> - 所有与服务相关的字段。
<b>inroute: (ir:)</b> - 匹配扫描的路由器在路由跟踪输出中的情况。
```