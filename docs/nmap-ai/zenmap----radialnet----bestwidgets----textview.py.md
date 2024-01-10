# `nmap\zenmap\radialnet\bestwidgets\textview.py`

```
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
# * 我们提供此软件的源代码，因为我们相信用户有权根据自己的需求修改和重新分发它。但是，我们希望用户尊重我们的劳动成果，遵守NPSL的条款，并尊重其他软件的许可证。
# 导入gi模块，确保使用的是Gtk 3.0版本
import gi

gi.require_version("Gtk", "3.0")
from gi.repository import Gtk
from radialnet.bestwidgets.boxes import *

# 创建一个名为BWTextView的类，继承自BWScrolledWindow
class BWTextView(BWScrolledWindow):
    """
    """

    # 初始化方法
    def __init__(self):
        """
        """
        # 调用父类的初始化方法
        BWScrolledWindow.__init__(self)

        # 设置私有属性__auto_scroll的初始值为False
        self.__auto_scroll = False

        # 调用私有方法__create_widgets
        self.__create_widgets()

    # 创建私有方法__create_widgets
    def __create_widgets(self):
        """
        """
        # 创建一个Gtk.TextBuffer对象
        self.__textbuffer = Gtk.TextBuffer()
        # 创建一个带有缓冲区的Gtk.TextView对象
        self.__textview = Gtk.TextView.new_with_buffer(self.__textbuffer)

        # 将文本视图添加到滚动窗口中
        self.add_with_viewport(self.__textview)

    # 创建公有方法bw_set_auto_scroll
    def bw_set_auto_scroll(self, value):
        """
        """
        # 设置私有属性__auto_scroll的值
        self.__auto_scroll = value
    # 设置文本视图是否可编辑
    def bw_set_editable(self, editable):
        # 调用私有属性__textview的方法，设置是否可编辑
        self.__textview.set_editable(False)
    
    # 修改文本视图的字体
    def bw_modify_font(self, font):
        # 调用私有属性__textview的方法，修改字体
        self.__textview.modify_font(font)
    
    # 设置文本内容
    def bw_set_text(self, text):
        # 调用私有属性__textbuffer的方法，设置文本内容
        self.__textbuffer.set_text(text)
        # 如果自动滚动开启，则调用bw_set_scroll_down方法
        if self.__auto_scroll:
            self.bw_set_scroll_down()
    
    # 获取文本内容
    def bw_get_text(self):
        # 调用私有属性__textbuffer的方法，获取文本内容
        return self.__textbuffer.get_text(self.__textbuffer.get_start_iter(),
                                          self.__textbuffer.get_end_iter())
    
    # 设置滚动条到底部
    def bw_set_scroll_down(self):
        # 调用get_vadjustment方法获取垂直滚动条对象，设置其值为最大值
        self.get_vadjustment().set_value(self.get_vadjustment().upper)
    
    # 获取文本缓冲区
    def bw_get_textbuffer(self):
        # 返回私有属性__textbuffer
        return self.__textbuffer
class BWTextEditor(BWScrolledWindow):
    """
    文本编辑器类，继承自带滚动条的窗口类
    """
    def __init__(self):
        """
        初始化方法
        """
        # 调用父类的初始化方法
        BWScrolledWindow.__init__(self)
        # 连接绘制事件和自定义的绘制方法
        self.connect('draw', self.__draw)

        # 初始化自动滚动属性为 False
        self.__auto_scroll = False

        # 创建编辑器的各个部件
        self.__create_widgets()

    def __create_widgets(self):
        """
        创建编辑器的各个部件
        """
        # 创建水平布局
        self.__hbox = BWHBox(spacing=6)

        # 创建文本缓冲区和文本视图
        self.__textbuffer = Gtk.TextBuffer()
        self.__textview = Gtk.TextView.new_with_buffer(self.__textbuffer)

        # 创建行号缓冲区和行号视图
        self.__linebuffer = Gtk.TextBuffer()
        self.__lineview = Gtk.TextView.new_with_buffer(self.__linebuffer)
        self.__lineview.set_justification(Gtk.Justification.RIGHT)
        self.__lineview.set_editable(False)
        self.__lineview.set_sensitive(False)

        # 将行号视图和文本视图添加到水平布局中
        self.__hbox.bw_pack_start_noexpand_nofill(self.__lineview)
        self.__hbox.bw_pack_start_expand_fill(self.__textview)

        # 将水平布局添加到带滚动条的窗口中
        self.add_with_viewport(self.__hbox)

    def __draw(self, widget, event):
        """
        绘制方法，用于修复 GTK 显示文本不正确的问题
        """
        # 修复 GTK 显示文本不正确的问题
        self.__hbox.check_resize()

    def bw_set_auto_scroll(self, value):
        """
        设置自动滚动属性的方法
        """
        self.__auto_scroll = value

    def bw_set_editable(self, editable):
        """
        设置文本视图是否可编辑的方法
        """
        self.__textview.set_editable(False)

    def bw_modify_font(self, font):
        """
        修改文本视图和行号视图的字体的方法
        """
        self.__textview.modify_font(font)
        self.__lineview.modify_font(font)

    def bw_set_text(self, text):
        """
        设置文本内容的方法
        """
        if text != "":
            # 计算文本中的行数
            count = text.count('\n') + text.count('\r')

            # 生成行号列表
            lines = range(1, count + 2)
            lines = [str(i).strip() for i in lines]

            # 设置文本缓冲区和行号缓冲区的内容
            self.__textbuffer.set_text(text)
            self.__linebuffer.set_text('\n'.join(lines))

            # 如果自动滚动属性为 True，则滚动到底部
            if self.__auto_scroll:
                self.bw_set_scroll_down()

        else:
            # 如果文本为空，则清空文本缓冲区和行号缓冲区的内容
            self.__textbuffer.set_text("")
            self.__linebuffer.set_text("")
    # 获取文本缓冲区中的文本内容
    def bw_get_text(self):
        """
        """
        # 从文本缓冲区中获取文本内容，从起始位置到结束位置
        return self.__textbuffer.get_text(self.__textbuffer.get_start_iter(),
                                          self.__textbuffer.get_end_iter())
    
    # 设置滚动条向下滚动
    def bw_set_scroll_down(self):
        """
        """
        # 获取垂直调整对象，并设置其值为最大值，即向下滚动
        self.get_vadjustment().set_value(self.get_vadjustment().upper)
```