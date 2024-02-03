# `nmap\zenmap\radialnet\bestwidgets\comboboxes.py`

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
# * 官方的Nmap Windows构建包括Npcap软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特殊许可就重新分发的单独许可条款的约束。因此，官方的Nmap Windows构建可能不得未经特殊许可（例如Nmap OEM许可证）重新分发。
# *
# * 我们提供此软件的源代码，因为我们相信用户有权根据自己的需求进行修改和重新分发。但是，我们希望用户尊重Nmap许可证的条款。
# 导入gi模块，要求使用版本3.0的Gtk
import gi

# 要求使用版本3.0的Gtk，从gi.repository中导入Gtk和GObject
gi.require_version("Gtk", "3.0")
from gi.repository import Gtk, GObject

# 创建一个继承自Gtk.ComboBoxText的类BWChangeableComboBoxEntry
class BWChangeableComboBoxEntry(Gtk.ComboBoxText):
    """
    """
    # 初始化方法
    def __init__(self):
        """
        """
        # 创建一个新的ListStore，用于存储字符串
        self.__liststore = Gtk.ListStore.new([str])

        # 调用父类的初始化方法，传入ListStore作为model，并设置has_entry为True
        Gtk.ComboBoxText.__init__(self, model=self.__liststore, has_entry=True)

        # 连接"changed"信号到self.__changed方法
        self.connect("changed", self.__changed)
        # 连接子组件的"changed"信号到self.__entry_changed方法
        self.get_child().connect("changed", self.__entry_changed)

        # 初始化self.__last_active为None
        self.__last_active = None

    # "changed"信号的处理方法
    def __changed(self, widget):
        """
        """
        # 如果当前选中的项不是-1（即有选中项），则将当前选中项保存到self.__last_active
        if self.get_active() != -1:
            self.__last_active = self.get_active()

    # 返回ListStore中存储的字符串数量的方法
    def bw_get_length(self):
        """
        """
        return len(self.__liststore)
    # 当下拉框内容改变时触发的函数
    def __entry_changed(self, widget):
        # 如果下拉框内容不为空，并且上一个活动项不为空，并且当前选中项为-1
        if len(self.__liststore) > 0 and\
           self.__last_active is not None and\
           self.get_active() == -1:
            # 获取上一个活动项的迭代器，设置其值为文本框内容的去除空格后的值
            iter = self.get_model().get_iter((self.__last_active,))
            self.__liststore.set_value(iter, 0, widget.get_text().strip())
    
    # 获取下拉框当前选中项的函数
    def bw_get_active(self):
        # 如果当前选中项为-1，则返回上一个活动项
        if self.get_active() == -1:
            return self.__last_active
        # 否则返回当前选中项
        return self.get_active()
# 如果当前脚本被直接执行，则执行以下代码
if __name__ == "__main__":

    # 定义一个按钮点击事件的回调函数
    def button_clicked(widget, combo):
        """
        """
        # 在下拉框中添加文本 'New'
        combo.append_text('New')

    # 创建一个 GTK 窗口
    window = Gtk.Window()
    # 连接窗口关闭事件，关闭窗口时退出 GTK 主循环
    window.connect("destroy", lambda w: Gtk.main_quit())

    # 创建一个水平方向的 GTK 容器
    box = Gtk.Box.new(Gtk.Orientation.HORIZONTAL, 0)

    # 创建一个可变的下拉框输入框组件
    combo = BWChangeableComboBoxEntry()
    # 在下拉框中添加文本 'New'
    combo.append_text('New')
    # 设置下拉框的默认选中项为第一个
    combo.set_active(0)

    # 创建一个带有标签的按钮
    button = Gtk.Button.new_with_label('More')
    # 连接按钮点击事件，点击按钮时调用 button_clicked 函数，并传入 combo 参数
    button.connect("clicked", button_clicked, combo)

    # 将按钮添加到 GTK 容器中，水平方向，不填充，不扩展，间距为 0
    box.pack_start(button, False, False, 0)
    # 将下拉框添加到 GTK 容器中，水平方向，填充，扩展，间距为 0
    box.pack_start(combo, True, True, 0)

    # 将 GTK 容器添加到窗口中
    window.add(box)
    # 显示窗口中的所有组件
    window.show_all()

    # 进入 GTK 主循环
    Gtk.main()
```