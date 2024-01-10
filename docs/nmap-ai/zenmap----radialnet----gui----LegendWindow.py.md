# `nmap\zenmap\radialnet\gui\LegendWindow.py`

```
# 设置文件编码为 UTF-8

# ***********************IMPORTANT NMAP LICENSE TERMS************************
# *
# * Nmap安全扫描器由Nmap Software LLC（“Nmap项目”）（C）1996-2023年拥有。Nmap也是Nmap项目的注册商标。
# *
# * 本程序根据Nmap公共源代码许可证（NPSL）的条款分发。适用于特定Nmap版本或源代码控制修订版的确切许可证文本包含在该版本的Nmap或源代码控制修订版的LICENSE文件中。有关更多Nmap版权/法律信息，请访问https://nmap.org/book/man-legal.html，有关NPSL许可证本身的更多信息，请访问https://nmap.org/npsl/。此标题总结了Nmap许可证的一些关键点，但不能替代实际许可证文本。
# *
# * 一般情况下，用户可以免费下载和使用Nmap，包括商业用途。它可以从https://nmap.org获得。
# *
# * Nmap许可证一般禁止公司在商业产品中使用和重新分发Nmap，但我们销售一个特殊的Nmap OEM版本，具有更宽松的许可证和专门用于此目的的特殊功能。请参阅https://nmap.org/oem/
# *
# * 如果您收到了书面的Nmap许可协议或合同，其中规定了与这些条款（例如Nmap OEM许可证）不同的条款，您可以选择根据那些条款使用和重新分发Nmap。
# *
# * 官方的Nmap Windows版本包括Npcap软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特殊许可证重新分发的单独许可证条款的约束。因此，官方的Nmap Windows版本可能不得在没有特殊许可证（例如Nmap OEM许可证）的情况下重新分发。
# *
# * 我们提供此软件的源代码，因为我们相信用户有权根据自己的需求修改和重新分发它。但是，我们希望用户尊重我们的劳动成果，遵守NPSL的条款。
# 导入gi库，确保使用的是Gtk 3.0版本
import gi

gi.require_version("Gtk", "3.0")
from gi.repository import Gtk, Gdk, Pango

# 导入数学和绘图库
import math
import cairo

# 导入国际化模块
import zenmapCore.I18N  # lgtm[py/unused-import]

# 定义常量DIMENSION_NORMAL
DIMENSION_NORMAL = (350, 450)

# 定义绘制图像的函数，传入绘图上下文、坐标、图像名称和标签
def draw_pixmap(context, x, y, name, label):
    # 设置绘图上下文的源为指定图像的像素缓冲区
    Gdk.cairo_set_source_pixbuf(context, Pixmaps().get_pixbuf(name), x, y)
    # 绘制图像
    context.paint()
    # 移动绘图位置到指定坐标
    context.move_to(x + 50, y + 10)
    # 设置文本颜色
    context.set_source_rgb(0, 0, 0)
    # 显示文本
    context.show_text(label)

# 重置字体样式
def reset_font(context):
    # 选择字体样式
    context.select_font_face(
            "Monospace", cairo.FONT_SLANT_NORMAL, cairo.FONT_SLANT_NORMAL)
    # 设置字体大小
    context.set_font_size(11)
# 在给定的上下文中绘制标题
def draw_heading(context, x, y, label):
    # 选择字体样式
    context.select_font_face(
            "Monospace", cairo.FONT_SLANT_NORMAL, cairo.FONT_WEIGHT_BOLD)
    # 设置字体大小
    context.set_font_size(13)
    # 移动到指定位置
    context.move_to(x - 15, y)
    # 设置绘制颜色
    context.set_source_rgb(0, 0, 0)
    # 显示文本
    context.show_text(label)
    # 重置字体
    reset_font(context)


# 在给定的上下文中绘制圆形
def draw_circle(context, x, y, size, color, label):
    # 设置绘制颜色
    context.set_source_rgb(0, 0, 0)
    # 移动到指定位置
    context.move_to(x, y)
    # 绘制圆形
    context.arc(x, y, size, 0, 2 * math.pi)
    # 保留路径并绘制
    context.stroke_preserve()
    # 设置填充颜色
    context.set_source_rgb(*color)
    # 填充圆形
    context.fill()
    # 设置绘制颜色
    context.set_source_rgb(0, 0, 0)
    # 移动到指定位置
    context.move_to(x + 50, y + 5)
    # 显示文本
    context.show_text(label)


# 在给定的上下文中绘制正方形
def draw_square(context, x, y, size, color):
    # 设置绘制颜色
    context.set_source_rgb(0, 0, 0)
    # 绘制矩形
    context.rectangle(x, y - size / 2, size, size)
    # 保留路径并绘制
    context.stroke_preserve()
    # 设置填充颜色
    context.set_source_rgb(*color)
    # 填充矩形
    context.fill()


# 在给定的上下文中绘制线条
def draw_line(context, x, y, dash, color, label):
    # 设置绘制颜色
    context.set_source_rgb(*color)
    # 移动到指定位置
    context.move_to(x - 20, y)
    # 设置虚线样式
    context.set_dash(dash)
    # 绘制直线
    context.line_to(x + 25, y)
    # 绘制线条
    context.stroke()
    # 重置虚线样式
    context.set_dash([])
    # 设置绘制颜色
    context.set_source_rgb(0, 0, 0)
    # 移动到指定位置
    context.move_to(x + 50, y + 5)
    # 显示文本
    context.show_text(label)


# 创建一个名为LegendWindow的类，继承自Gtk.Window
class LegendWindow(Gtk.Window):
    """
    """
    # 初始化窗口对象
    def __init__(self):
        """
        """
        # 调用父类的初始化方法，创建顶层窗口
        Gtk.Window.__init__(self, type=Gtk.WindowType.TOPLEVEL)
        # 设置窗口的默认大小
        self.set_default_size(DIMENSION_NORMAL[0], DIMENSION_NORMAL[1])
        # 设置标题字体为 Monospace Bold
        self.__title_font = Pango.FontDescription("Monospace Bold")
        # 设置窗口标题为 "Topology Legend"
        self.set_title(_("Topology Legend"))

        # 创建垂直布局容器
        self.vbox = Gtk.Box.new(Gtk.Orientation.VERTICAL, 0)
        # 将垂直布局容器添加到窗口中
        self.add(self.vbox)

        # 创建绘图区域
        self.drawing_area = Gtk.DrawingArea()
        # 将绘图区域添加到垂直布局容器中
        self.vbox.pack_start(self.drawing_area, True, True, 0)
        # 连接绘图事件处理器
        self.drawing_area.connect("draw", self.draw_event_handler)
        # 创建带标签的链接按钮，链接到指定网址
        self.more_uri = Gtk.LinkButton.new_with_label(
                "https://nmap.org/book/zenmap-topology.html#zenmap-topology-legend",
                _("View full legend online"))
        # 将链接按钮添加到垂直布局容器中
        self.vbox.pack_start(self.more_uri, False, False, 0)
```