# `nmap\zenmap\radialnet\gui\RadialNet.py`

```
# 设置文件编码为 UTF-8
# 重要提示：NMAP许可证条款
# Nmap安全扫描仪是Nmap软件有限责任公司（“Nmap项目”）1996-2023年的版权所有。Nmap也是Nmap项目的注册商标。
# 本程序根据Nmap公共源代码许可证（NPSL）的条款分发。适用于特定Nmap版本或源代码控制修订版的确切许可证文本包含在该版本的Nmap或源代码控制修订版中分发的LICENSE文件中。有关更多Nmap版权/法律信息，请访问https://nmap.org/book/man-legal.html，有关NPSL许可证本身的更多信息，请访问https://nmap.org/npsl/。此标题总结了Nmap许可证的一些关键点，但不能替代实际的许可证文本。
# 一般情况下，用户可以免费下载和使用Nmap，包括商业用途。它可以从https://nmap.org获得。
# Nmap许可证一般禁止公司在商业产品中使用和重新分发Nmap，但我们销售具有更宽松许可证和特殊功能的特殊Nmap OEM版本。请参阅https://nmap.org/oem/
# 如果您收到了书面的Nmap许可协议或合同，其中规定了与这些条款（例如Nmap OEM许可证）不同的条款，您可以选择根据这些条款使用和重新分发Nmap。
# 官方的Nmap Windows版本包括Npcap软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特别许可重新分发的单独许可条款的约束。因此，官方的Nmap Windows版本可能不得在没有特别许可的情况下重新分发（例如Nmap OEM许可证）。
# 我们提供此软件的源代码，因为我们相信用户有权根据自己的需求进行修改和重新分发。因此，我们希望用户能够了解Nmap的许可证条款，并遵守这些条款。
# 导入 gi 模块
import gi
# 要求使用 Gtk 3.0 版本
gi.require_version("Gtk", "3.0")
# 从 gi.repository 中导入 Gtk, GLib, Gdk 模块
from gi.repository import Gtk, GLib, Gdk

# 导入 math, cairo 模块
import math
import cairo

# 从 functools 模块中导入 reduce 函数
from functools import reduce

# 导入 radialnet.util.geometry, radialnet.util.misc 模块
import radialnet.util.geometry as geometry
import radialnet.util.misc as misc

# 从 radialnet.core.Coordinate 中导入 PolarCoordinate, CartesianCoordinate 类
from radialnet.core.Coordinate import PolarCoordinate, CartesianCoordinate
# 从 radialnet.core.Interpolation 中导入 Linear2DInterpolator 类
from radialnet.core.Interpolation import Linear2DInterpolator
# 从 radialnet.core.Graph 中导入 Node 类
from radialnet.core.Graph import Node
# 从 radialnet.gui.NodeWindow 中导入 NodeWindow 类
from radialnet.gui.NodeWindow import NodeWindow
# 从 radialnet.gui.Image 中导入 Icons, get_pixels_for_cairo_image_surface 函数
from radialnet.gui.Image import Icons, get_pixels_for_cairo_image_surface

# 定义 REGION_COLORS 列表
REGION_COLORS = [(1.0, 0.0, 0.0), (1.0, 1.0, 0.0), (0.0, 1.0, 0.0)]
# 定义 REGION_RED, REGION_YELLOW, REGION_GREEN 常量
REGION_RED = 0
REGION_YELLOW = 1
REGION_GREEN = 2

# 定义 SQUARE_TYPES 列表
SQUARE_TYPES = ['router', 'switch', 'wap']
# 定义图标类型的字典
ICON_DICT = {'router': 'router',
             'switch': 'switch',
             'wap': 'wireless',
             'firewall': 'firewall'}

# 定义指针跳转到的位置
POINTER_JUMP_TO = 0
# 定义指针信息的位置
POINTER_INFO = 1
# 定义指针组的位置
POINTER_GROUP = 2
# 定义指针填充的位置
POINTER_FILL = 3

# 定义对称布局
LAYOUT_SYMMETRIC = 0
# 定义加权布局
LAYOUT_WEIGHTED = 1

# 定义笛卡尔插值
INTERPOLATION_CARTESIAN = 0
# 定义极坐标插值
INTERPOLATION_POLAR = 1

# 定义文件类型为 PDF
FILE_TYPE_PDF = 1
# 定义文件类型为 PNG
FILE_TYPE_PNG = 2
# 定义文件类型为 PS
FILE_TYPE_PS = 3
# 定义文件类型为 SVG
FILE_TYPE_SVG = 4

# 定义 RadialNet 类，继承自 Gtk.DrawingArea
class RadialNet(Gtk.DrawingArea):
    """
    Radial network visualization widget
    """
    # 定义装饰器函数，用于防止在图形未设置时执行
    def graph_is_not_empty(function):
        """
        Decorator function to prevent the execution when graph not is set
        @type  function: function
        @param function: Protected function
        """
        # 检查图形状态的函数
        def check_graph_status(*args):
            if args[0].__graph is None:
                return False
            return function(*args)

        return check_graph_status

    # 定义装饰器函数，用于防止在图形动画时执行
    def not_is_in_animation(function):
        """
        Decorator function to prevent the execution when graph is animating
        @type  function: function
        @param function: Protected function
        """
        # 检查动画状态的函数
        def check_animation_status(*args):
            if args[0].__animating:
                return False
            return function(*args)

        return check_animation_status
    # 将绘图保存到文件
    def save_drawing_to_file(self, file, type=FILE_TYPE_PNG):
        # 获取分配的空间
        allocation = self.get_allocation()

        # 根据文件类型创建不同类型的表面
        if type == FILE_TYPE_PDF:
            self.surface = cairo.PDFSurface(file,
                    allocation.width,
                    allocation.height)
        elif type == FILE_TYPE_PNG:
            self.surface = cairo.ImageSurface(cairo.FORMAT_ARGB32,
                    allocation.width,
                    allocation.height)
        elif type == FILE_TYPE_PS:
            self.surface = cairo.PSSurface(file,
                    allocation.width,
                    allocation.height)
        elif type == FILE_TYPE_SVG:
            self.surface = cairo.SVGSurface(file,
                    allocation.width,
                    allocation.height)
        else:
            raise TypeError('unknown surface type')

        # 创建绘图上下文
        context = cairo.Context(self.surface)

        # 绘制矩形并填充白色背景
        context.rectangle(0, 0, allocation.width, allocation.height)
        context.set_source_rgb(1.0, 1.0, 1.0)
        context.fill()

        # 调用私有方法进行绘制
        self.__draw(context)

        # 如果文件类型为 PNG，则将表面内容写入文件
        if type == FILE_TYPE_PNG:
            self.surface.write_to_png(file)

        # 刷新表面并完成绘制
        self.surface.flush()
        self.surface.finish()

        return True

    # 获取缓慢进出的插值
    def get_slow_inout(self):
        return self.__interpolation_slow_in_out

    # 设置缓慢进出的插值
    def set_slow_inout(self, value):
        self.__interpolation_slow_in_out = value

    # 获取区域颜色
    def get_region_color(self):
        return self.__region_color

    # 设置区域颜色
    def set_region_color(self, value):
        self.__region_color = value

    # 获取是否显示区域
    def get_show_region(self):
        return self.__show_region

    # 设置是否显示区域
    def set_show_region(self, value):
        self.__show_region = value
        # 请求重新绘制
        self.queue_draw()

    # 获取指针状态
    def get_pointer_status(self):
        return self.__pointer_status
    # 设置指针状态
    def set_pointer_status(self, pointer_status):
        """
        """
        self.__pointer_status = pointer_status

    # 获取显示地址
    def get_show_address(self):
        """
        """
        return self.__show_address

    # 获取显示主机名
    def get_show_hostname(self):
        """
        """
        return self.__show_hostname

    # 获取显示环
    def get_show_ring(self):
        """
        """
        return self.__show_ring

    # 设置显示地址
    def set_show_address(self, value):
        """
        """
        self.__show_address = value
        # 重新绘制
        self.queue_draw()

    # 设置显示主机名
    def set_show_hostname(self, value):
        """
        """
        self.__show_hostname = value
        # 重新绘制
        self.queue_draw()

    # 设置显示环
    def set_show_ring(self, value):
        """
        """
        self.__show_ring = value
        # 重新绘制
        self.queue_draw()

    # 获取最小环间距
    def get_min_ring_gap(self):
        """
        """
        return self.__min_ring_gap

    # 设置最小环间距
    @graph_is_not_empty
    @not_is_in_animation
    def set_min_ring_gap(self, value):
        """
        """
        self.__min_ring_gap = int(value)

        if self.__ring_gap < self.__min_ring_gap:
            self.__ring_gap = self.__min_ring_gap

        # 更新节点位置
        self.__update_nodes_positions()
        # 重新绘制
        self.queue_draw()

        return True

    # 获取帧数
    def get_number_of_frames(self):
        """
        """
        return self.__number_of_frames

    # 设置帧数
    @not_is_in_animation
    def set_number_of_frames(self, number_of_frames):
        """
        """
        if number_of_frames > 2:
            self.__number_of_frames = int(number_of_frames)
            return True

        self.__number_of_frames = 3
        return False

    # 更新布局
    @not_is_in_animation
    def update_layout(self):
        """
        """
        if self.__graph is None:
            return
        self.__animating = True
        self.__calc_interpolation(self.__graph.get_main_node())
        self.__livens_up()

    @not_is_in_animation
    # 设置布局
    def set_layout(self, layout):
        """
        """
        # 如果当前布局与传入的布局不同
        if self.__layout != layout:
            # 更新当前布局
            self.__layout = layout
            # 如果图形对象不为空
            if self.__graph is not None:
                # 设置动画标志为True
                self.__animating = True
                # 计算插值
                self.__calc_interpolation(self.__graph.get_main_node())
                # 更新显示
                self.__livens_up()
            # 返回True
            return True
        # 返回False
        return False

    # 获取布局
    def get_layout(self):
        """
        """
        return self.__layout

    # 设置插值
    @not_is_in_animation
    def set_interpolation(self, interpolation):
        """
        """
        # 更新插值
        self.__interpolation = interpolation
        # 返回True
        return True

    # 获取插值
    def get_interpolation(self):
        """
        """
        return self.__interpolation

    # 获取环数
    def get_number_of_rings(self):
        """
        """
        return self.__number_of_rings

    # 获取鱼眼环
    def get_fisheye_ring(self):
        """
        """
        return self.__fisheye_ring

    # 获取鱼眼兴趣
    def get_fisheye_interest(self):
        """
        """
        return self.__fisheye_interest

    # 获取鱼眼扩散
    def get_fisheye_spread(self):
        """
        """
        return self.__fisheye_spread

    # 获取鱼眼
    def get_fisheye(self):
        """
        """
        return self.__fisheye

    # 设置鱼眼
    def set_fisheye(self, enable):
        """
        """
        # 更新鱼眼状态
        self.__fisheye = enable
        # 更新节点位置
        self.__update_nodes_positions()
        # 重新绘制
        self.queue_draw()

    # 设置鱼眼环
    def set_fisheye_ring(self, value):
        """
        """
        # 更新鱼眼环
        self.__fisheye_ring = value
        # 检查鱼眼环
        self.__check_fisheye_ring()
        # 更新节点位置
        self.__update_nodes_positions()
        # 重新绘制
        self.queue_draw()

    # 设置鱼眼兴趣
    def set_fisheye_interest(self, value):
        """
        """
        # 更新鱼眼兴趣
        self.__fisheye_interest = value
        # 更新节点位置
        self.__update_nodes_positions()
        # 重新绘制
        self.queue_draw()

    # 设置鱼眼扩散
    def set_fisheye_spread(self, value):
        """
        """
        # 更新鱼眼扩散
        self.__fisheye_spread = value
        # 更新节点位置
        self.__update_nodes_positions()
        # 重新绘制
        self.queue_draw()

    # 获取显示图标
    def get_show_icon(self):
        """
        """
        return self.__show_icon
    # 设置是否显示图标的属性
    def set_show_icon(self, value):
        # 设置私有属性 __show_icon 的数值
        self.__show_icon = value
        # 重新绘制窗口
        self.queue_draw()
    
    # 获取是否显示延迟的属性
    def get_show_latency(self):
        # 返回私有属性 __show_latency 的数值
        return self.__show_latency
    
    # 设置是否显示延迟的属性
    def set_show_latency(self, value):
        # 设置私有属性 __show_latency 的数值
        self.__show_latency = value
        # 重新绘制窗口
        self.queue_draw()
    
    # 获取比例尺属性
    def get_scale(self):
        # 返回私有属性 __scale 的数值
        return self.__scale
    
    # 获取缩放比例属性
    def get_zoom(self):
        # 返回私有属性 __scale 乘以 100 后的整数值
        return int(round(self.__scale * 100))
    
    # 设置比例尺属性
    def set_scale(self, scale):
        # 如果比例尺大于等于 0.01
        if scale >= 0.01:
            # 设置私有属性 __scale 的数值
            self.__scale = scale
            # 重新绘制窗口
            self.queue_draw()
    
    # 设置缩放比例属性
    def set_zoom(self, zoom):
        # 如果缩放比例大于等于 1
        if float(zoom) >= 1:
            # 调用 set_scale 方法，将缩放比例转换为比例尺后重新设置比例尺属性
            self.set_scale(float(zoom) / 100.0)
            # 重新绘制窗口
            self.queue_draw()
    
    # 获取环间距属性
    def get_ring_gap(self):
        # 返回私有属性 __ring_gap 的数值
        return self.__ring_gap
    
    # 设置环间距属性
    @not_is_in_animation
    def set_ring_gap(self, ring_gap):
        # 如果环间距大于等于最小环间距
        if ring_gap >= self.__min_ring_gap:
            # 设置私有属性 __ring_gap 的数值
            self.__ring_gap = ring_gap
            # 更新节点位置
            self.__update_nodes_positions()
            # 重新绘制窗口
            self.queue_draw()
    
    # 处理滚动事件
    def scroll_event(self, widget, event):
        # 如果滚动方向为向上
        if event.direction == Gdk.ScrollDirection.UP:
            # 调用 set_scale 方法，增加比例尺
            self.set_scale(self.__scale + 0.01)
    
        # 如果滚动方向为向下
        if event.direction == Gdk.ScrollDirection.DOWN:
            # 调用 set_scale 方法，减小比例尺
            self.set_scale(self.__scale - 0.01)
    
        # 重新绘制窗口
        self.queue_draw()
    
    # 处理键盘按下事件
    @graph_is_not_empty
    @not_is_in_animation
    def key_press(self, widget, event):
        # 获取按下的键名
        key = Gdk.keyval_name(event.keyval)
    
        # 如果按下的是 KP_Add 键
        if key == 'KP_Add':
            # 调用 set_ring_gap 方法，增加环间距
            self.set_ring_gap(self.__ring_gap + 1)
    
        # 如果按下的是 KP_Subtract 键
        elif key == 'KP_Subtract':
            # 调用 set_ring_gap 方法，减小环间距
            self.set_ring_gap(self.__ring_gap - 1)
    
        # 如果按下的是 Page_Up 键
        elif key == 'Page_Up':
            # 调用 set_scale 方法，增加比例尺
            self.set_scale(self.__scale + 0.01)
    
        # 如果按下的是 Page_Down 键
        elif key == 'Page_Down':
            # 调用 set_scale 方法，减小比例尺
            self.set_scale(self.__scale - 0.01)
    
        # 重新绘制窗口
        self.queue_draw()
    
        # 返回 True
        return True
    # 当按键释放时的回调函数，根据按键的不同执行相应的操作
    @graph_is_not_empty
    def key_release(self, widget, event):
        """
        """
        # 获取释放的按键名称
        key = Gdk.keyval_name(event.keyval)

        # 如果按下的是 'c' 键
        if key == 'c':
            # 重置平移值
            self.__translation = (0, 0)

        # 如果按下的是 'r' 键
        elif key == 'r':
            # 切换是否显示环
            self.__show_ring = not self.__show_ring

        # 如果按下的是 'a' 键
        elif key == 'a':
            # 切换是否显示地址
            self.__show_address = not self.__show_address

        # 如果按下的是 'h' 键
        elif key == 'h':
            # 切换是否显示主机名
            self.__show_hostname = not self.__show_hostname

        # 如果按下的是 'i' 键
        elif key == 'i':
            # 切换是否显示图标
            self.__show_icon = not self.__show_icon

        # 如果按下的是 'l' 键
        elif key == 'l':
            # 切换是否显示延迟
            self.__show_latency = not self.__show_latency

        # 重新绘制图形
        self.queue_draw()

        # 返回事件处理完成
        return True

    # 鼠标进入窗口时的回调函数
    @graph_is_not_empty
    @not_is_in_animation
    def enter_notify(self, widget, event):
        """
        """
        # 获取焦点
        self.grab_focus()
        # 不再传播事件
        return False

    # 鼠标离开窗口时的回调函数
    @graph_is_not_empty
    @not_is_in_animation
    def leave_notify(self, widget, event):
        """
        """
        # 将所有节点的绘制信息中的 'over' 属性设置为 False
        for node in self.__graph.get_nodes():
            node.set_draw_info({'over': False})

        # 重新绘制图形
        self.queue_draw()

        # 不再传播事件
        return False

    # 鼠标释放时的回调函数
    @graph_is_not_empty
    @graph_is_not_empty
    def button_release(self, widget, event):
        """
        Drawing callback
        @type  widget: GtkWidget
        @param widget: Gtk widget superclass
        @type  event: GtkEvent
        @param event: Gtk event of widget
        @rtype: boolean
        @return: Indicator of the event propagation
        """
        # 如果释放的是鼠标左键
        if event.button == 1:
            # 标记鼠标左键未按下
            self.__button1_press = False

        # 如果释放的是鼠标中键
        if event.button == 2:
            # 标记鼠标中键未按下
            self.__button2_press = False

        # 如果释放的是鼠标右键
        if event.button == 3:
            # 标记鼠标右键未按下
            self.__button3_press = False

        # 获取焦点
        self.grab_focus()

        # 不再传播事件
        return False

    # 确保图形不为空
    @graph_is_not_empty
    # 鼠标移动事件回调函数
    def motion_notify(self, widget, event):
        """
        Drawing callback
        @type  widget: GtkWidget
        @param widget: Gtk widget superclass
        @type  event: GtkEvent
        @param event: Gtk event of widget
        @rtype: boolean
        @return: Indicator of the event propagation
        """
        # 获取鼠标指针位置
        pointer = self.get_pointer()

        # 将所有节点的绘制信息中的 'over' 属性设置为 False
        for node in self.__graph.get_nodes():
            node.set_draw_info({'over': False})

        # 获取鼠标指针所在的节点
        result = self.__get_node_by_coordinate(self.get_pointer())

        # 如果鼠标指针在节点上，则将该节点的绘制信息中的 'over' 属性设置为 True
        if result is not None:
            result[0].set_draw_info({'over': True})

        # 如果鼠标左键按下并且上次移动的位置不为空
        elif self.__button1_press and self.__last_motion_point is not None:

            # 计算鼠标移动的距离，并更新图形的平移量
            ax, ay = pointer
            ox, oy = self.__last_motion_point
            tx, ty = self.__translation
            self.__translation = (tx + ax - ox, ty - ay + oy)

        # 更新上次移动的位置
        self.__last_motion_point = pointer

        # 获取焦点并重新绘制
        self.grab_focus()
        self.queue_draw()

        return False

    # 绘制回调函数
    def draw(self, widget, context):
        """
        Drawing callback
        @type  widget: GtkWidget
        @param widget: Gtk widget superclass
        @type  context: cairo.Context
        @param context: cairo context class
        @rtype: boolean
        @return: Indicator of the event propagation
        """
        # 设置绘制上下文的颜色为白色并填充整个绘图区域
        context.set_source_rgb(1.0, 1.0, 1.0)
        context.fill()

        # 调用私有方法进行绘制
        self.__draw(context)

        return False

    # 装饰器，用于检查图形是否为空
    @graph_is_not_empty
    # 绘制两个节点之间的连接
    def __draw_edge(self, context, edge):
        """
        Draw the connection between two nodes
        @type  : Edge
        @param : The second node that will be connected
        """
        # 获取连接的两个节点
        a, b = edge.get_nodes()

        # 获取节点的笛卡尔坐标
        xa, ya = a.get_cartesian_coordinate()
        xb, yb = b.get_cartesian_coordinate()
        xc, yc = self.__center_of_widget

        # 获取节点的子节点信息
        a_children = a.get_draw_info('children')
        b_children = b.get_draw_info('children')

        # 获取连接的延迟时间的平均值
        latency = edge.get_weights_mean()

        # 检查是否不是层次连接
        if a not in b_children and b not in a_children:
            context.set_source_rgba(1.0, 0.6, 0.1, 0.8)

        # 如果节点有无路由信息，则设置连接颜色为黑色
        elif a.get_draw_info('no_route') or b.get_draw_info('no_route'):
            context.set_source_rgba(0.0, 0.0, 0.0, 0.8)

        # 否则设置连接颜色为蓝色
        else:
            context.set_source_rgba(0.1, 0.5, 1.0, 0.8)

        # 根据延迟时间计算线条的粗细
        if latency is not None:

            min = self.__graph.get_min_edge_mean_weight()
            max = self.__graph.get_max_edge_mean_weight()

            if max != min:
                thickness = (latency - min) * 4 / (max - min) + 1

            else:
                thickness = 1

            context.set_line_width(thickness)

        else:
            # 如果没有延迟时间信息，则设置虚线并且线宽为1
            context.set_dash([2, 2])
            context.set_line_width(1)

        # 绘制连接线
        context.move_to(xc + xa, yc - ya)
        context.line_to(xc + xb, yc - yb)
        context.stroke()

        context.set_dash([1, 0])

        # 如果不是在动画中并且显示延迟时间
        if not self.__animating and self.__show_latency:

            if latency is not None:
                # 设置字体大小和线宽
                context.set_font_size(8)
                context.set_line_width(1)
                # 在连接线中间显示延迟时间
                context.move_to(xc + (xa + xb) / 2 + 1,
                                     yc - (ya + yb) / 2 + 4)
                context.show_text(str(round(latency, 2)))
                context.stroke()
    # 检查鱼眼环是否超出环的数量，如果超出则将其设置为环的数量减一
    def __check_fisheye_ring(self):
        if self.__fisheye_ring >= self.__number_of_rings:
            self.__fisheye_ring = self.__number_of_rings - 1

    # 设置环的数量，并调用检查鱼眼环的方法
    def __set_number_of_rings(self, value):
        self.__number_of_rings = value
        self.__check_fisheye_ring()

    # 计算鱼眼效果的函数
    def __fisheye_function(self, ring):
        distance = abs(self.__fisheye_ring - ring)
        level_of_detail = self.__ring_gap * self.__fisheye_interest
        spread_distance = distance - distance * self.__fisheye_spread
        value = level_of_detail / (spread_distance + 1)

        if value < self.__min_ring_gap:
            value = self.__min_ring_gap

        return value

    # 更新节点的位置信息
    @graph_is_not_empty
    @not_is_in_animation
    def __update_nodes_positions(self):
        for node in self.__sorted_nodes:
            if node.get_draw_info('grouped'):
                # 深度分组检查
                group = node.get_draw_info('group_node')
                while group.get_draw_info('group_node') is not None:
                    group = group.get_draw_info('group_node')
                ring = group.get_draw_info('ring')
                node.set_coordinate_radius(self.__calc_radius(ring))
            else:
                ring = node.get_draw_info('ring')
                node.set_coordinate_radius(self.__calc_radius(ring))

    # 检查图是否为空
    @graph_is_not_empty
    # 根据给定的坐标点，获取对应的节点
    def __get_node_by_coordinate(self, point):
        """
        """
        # 获取窗口中心的坐标
        xc, yc = self.__center_of_widget

        # 遍历图中的节点
        for node in self.__graph.get_nodes():

            # 如果节点被分组，则跳过
            if node.get_draw_info('grouped'):
                continue

            # 获取平移量
            ax, ay = self.__translation

            # 获取节点的笛卡尔坐标
            xn, yn = node.get_cartesian_coordinate()
            center = (xc + xn * self.__scale + ax, yc - yn * self.__scale - ay)
            radius = node.get_draw_info('radius') * self.__scale

            # 获取节点的设备类型
            type = node.get_info('device_type')

            # 如果设备类型是方形
            if type in SQUARE_TYPES:
                if geometry.is_in_square(point, radius, center):
                    return node, center

            # 如果设备类型不是方形
            else:
                if geometry.is_in_circle(point, radius, center):
                    return node, center

        # 如果没有找到对应的节点，则返回 None
        return None

    # 计算节点的半径
    def __calc_radius(self, ring):
        """
        """
        # 如果启用了鱼眼效果
        if self.__fisheye:

            radius = 0

            # 根据环数计算半径
            while ring > 0:
                radius += self.__fisheye_function(ring)
                ring -= 1

        # 如果没有启用鱼眼效果
        else:
            radius = ring * self.__ring_gap

        return radius

    # 装饰器，用于检查图是否为空
    @graph_is_not_empty
    # 重新排列节点，使其符合特定规则
    def __arrange_nodes(self):
        """
        """
        # 创建一个新节点集合，包含图的主节点
        new_nodes = set([self.__graph.get_main_node()])
        # 创建一个旧节点集合
        old_nodes = set()

        # 需要的环的数量
        number_of_needed_rings = 1
        ring = 0

        # 当发现新节点时循环
        while len(new_nodes) > 0:

            tmp_nodes = set()

            # 对于每个新节点
            for node in new_nodes:

                # 将节点添加到旧节点集合中
                old_nodes.add(node)

                # 设置环的位置
                node.set_draw_info({'ring': ring})

                # 检查组约束
                if (node.get_draw_info('group') or
                        node.get_draw_info('grouped')):
                    children = node.get_draw_info('children')

                else:

                    # 获取连接并修复多个父节点
                    children = set()
                    for child in self.__graph.get_node_connections(node):
                        if child in old_nodes or child in new_nodes:
                            continue
                        if child.get_draw_info('grouped'):
                            continue
                        children.add(child)

                # 设置父节点
                for child in children:
                    child.set_draw_info({'father': node})

                node.set_draw_info(
                        {'children': misc.sort_children(children, node)})
                tmp_nodes.update(children)

            # 检查组对环数量的影响
            for node in tmp_nodes:

                if not node.get_draw_info('grouped'):

                    number_of_needed_rings += 1
                    break

            # 更新新节点集合
            new_nodes.update(tmp_nodes)
            new_nodes.difference_update(old_nodes)

            ring += 1

        self.__set_number_of_rings(number_of_needed_rings)
    def __weighted_layout(self):
        """
        根据节点权重进行布局
        """
        # 计算每个节点所需的空间
        self.__graph.get_main_node().set_draw_info({'range': (0, 360)})
        new_nodes = set([self.__graph.get_main_node()])

        self.__graph.get_main_node().calc_needed_space()

        while len(new_nodes) > 0:

            node = new_nodes.pop()

            # 只添加非分组节点
            children = set()
            for child in node.get_draw_info('children'):

                if not child.get_draw_info('grouped'):
                    children.add(child)
                    new_nodes.add(child)

            if len(children) > 0:

                min, max = node.get_draw_info('range')

                node_total = max - min
                children_need = node.get_draw_info('children_need')

                for child in children:

                    child_need = child.get_draw_info('space_need')
                    child_total = node_total * child_need / children_need

                    theta = child_total / 2 + min + self.__rotate

                    child.set_coordinate_theta(theta)
                    child.set_draw_info({'range': (min, min + child_total)})

                    min += child_total
    # 对称布局函数，用于设置节点的绘制信息
    def __symmetric_layout(self):
        """
        """
        # 设置主节点的绘制范围为 0 到 360 度
        self.__graph.get_main_node().set_draw_info({'range': (0, 360)})
        # 创建一个新节点集合，初始包含主节点
        new_nodes = set([self.__graph.get_main_node()])

        # 当新节点集合不为空时循环
        while len(new_nodes) > 0:

            # 从新节点集合中取出一个节点
            node = new_nodes.pop()

            # 只添加非分组节点
            children = set()
            for child in node.get_draw_info('children'):

                if not child.get_draw_info('grouped'):
                    children.add(child)
                    new_nodes.add(child)

            # 如果子节点集合不为空
            if len(children) > 0:

                # 计算子节点的角度范围
                min, max = node.get_draw_info('range')
                factor = float(max - min) / len(children)

                # 为每个子节点设置角度和范围
                for child in children:

                    theta = factor / 2 + min + self.__rotate

                    child.set_coordinate_theta(theta)
                    child.set_draw_info({'range': (min, min + factor)})

                    min += factor

    # 根据给定的参考节点计算布局
    @graph_is_not_empty
    def __calc_layout(self, reference):
        """
        """
        # 选择布局算法
        if self.__layout == LAYOUT_SYMMETRIC:
            self.__symmetric_layout()

        elif self.__layout == LAYOUT_WEIGHTED:
            self.__weighted_layout()

        # 旋转焦点的子节点以保持方向
        if reference is not None:

            father, angle = reference
            theta = father.get_coordinate_theta()
            factor = theta - angle

            # 对每个节点进行旋转和范围调整
            for node in self.__graph.get_nodes():

                theta = node.get_coordinate_theta()
                node.set_coordinate_theta(theta - factor)

                a, b = node.get_draw_info('range')
                node.set_draw_info({'range': (a - factor, b - factor)})

    # 确保图形不为空
    @graph_is_not_empty
    # 计算节点的位置
    def __calc_node_positions(self, reference=None):
        """
        """
        # 设置节点的层次结构
        self.__arrange_nodes()
        # 计算排序后的节点
        self.calc_sorted_nodes()
    
        # 设置节点的坐标半径
        for node in self.__graph.get_nodes():
            # 获取节点的绘制信息中的环形属性
            ring = node.get_draw_info('ring')
            # 设置节点的坐标半径为计算得到的半径
            node.set_coordinate_radius(self.__calc_radius(ring))
    
        # 设置节点的极坐标角度
        self.__calc_layout(reference)
    # 私有方法，用于在动画中更新节点的位置
    def __livens_up(self, index=0):
        """
        """
        # 如果图形为空，则退出动画
        if self.__graph is None:
            self.__last_group_node = None
            self.__animating = False
            return False

        # 准备插值点
        if index == 0:
            # 防止不必要的动画
            no_need_to_move = True
            for node in self.__graph.get_nodes():
                ai, bi = node.get_draw_info('start_coordinate')
                af, bf = node.get_draw_info('final_coordinate')
                start_c = round(ai), round(bi)
                final_c = round(af), round(bf)
                if start_c != final_c:
                    no_need_to_move = False
            if no_need_to_move:
                self.__animating = False
                return False

        # 移动所有节点到第 'index' 步
        for node in self.__graph.get_nodes():
            a, b = node.get_draw_info('interpolated_coordinate')[index]
            if self.__interpolation == INTERPOLATION_POLAR:
                node.set_polar_coordinate(a, b)
            elif self.__interpolation == INTERPOLATION_CARTESIAN:
                node.set_cartesian_coordinate(a, b)

        self.queue_draw()

        # 动画继续条件
        if index < self.__number_of_frames - 1:
            GLib.timeout_add(self.__animation_rate,  # time to recall
                                self.__livens_up,       # recursive call
                                index + 1)              # next iteration
        else:
            self.__last_group_node = None
            self.__animating = False

        return False

    @not_is_in_animation
    # 设置要在布局中显示的图形
    def set_graph(self, graph):
        """
        Set graph to be displayed in layout
        @type  : Graph
        @param : Set the graph used in visualization
        """
        # 如果图形中有节点
        if graph.get_number_of_nodes() > 0:
            # 设置私有属性 __graph 为传入的图形
            self.__graph = graph
            # 计算节点位置
            self.__calc_node_positions()
            # 重新绘制
            self.queue_draw()
        else:
            # 如果图形中没有节点，则将 __graph 设置为 None
            self.__graph = None

    # 获取已扫描的节点
    def get_scanned_nodes(self):
        """
        """
        nodes = list()
        # 如果 __graph 为 None，则返回空列表
        if self.__graph is None:
            return nodes
        # 遍历图形中的节点，如果节点的绘制信息中包含'scanned'，则将节点添加到列表中
        for node in self.__graph.get_nodes():
            if node.get_draw_info('scanned'):
                nodes.append(node)
        return nodes

    # 获取图形
    def get_graph(self):
        """
        """
        return self.__graph

    # 设置为空
    def set_empty(self):
        """
        """
        # 删除 __graph，并将其设置为 None
        del(self.__graph)
        self.__graph = None
        # 重新绘制
        self.queue_draw()

    # 获取旋转角度
    def get_rotation(self):
        """
        """
        return self.__rotate

    # 设置旋转角度
    @graph_is_not_empty
    def set_rotation(self, angle):
        """
        """
        # 计算旋转角度的变化量
        delta = angle - self.__rotate
        self.__rotate = angle
        # 遍历图形中的节点，更新节点的坐标角度
        for node in self.__graph.get_nodes():
            theta = node.get_coordinate_theta()
            node.set_coordinate_theta(theta + delta)
        # 重新绘制
        self.queue_draw()

    # 获取平移
    def get_translation(self):
        """
        """
        return self.__translation

    # 设置平移
    @graph_is_not_empty
    def set_translation(self, translation):
        """
        """
        # 设置平移
        self.__translation = translation
        # 重新绘制
        self.queue_draw()

    # 判断是否为空
    def is_empty(self):
        """
        """
        return self.__graph is None

    # 判断是否在动画中
    def is_in_animation(self):
        """
        """
        return self.__animating

    # 计算排序后的节点
    def calc_sorted_nodes(self):
        """
        """
        # 将图形中的节点排序，并存储在私有属性 __sorted_nodes 中
        self.__sorted_nodes = list(self.__graph.get_nodes())
        self.__sorted_nodes.sort(key=lambda n: n.get_draw_info('ring'))
class NetNode(Node):
    """
    Node class for radial network widget
    """
    # 初始化方法，设置节点的绘制信息和极坐标
    def __init__(self):
        """
        """
        # 存储节点的绘制信息的字典
        self.__draw_info = dict()
        """Hash with draw information"""
        # 存储节点的极坐标
        self.__coordinate = PolarCoordinate()

        # 调用父类的初始化方法
        super(NetNode, self).__init__()

    # 获取节点代表的主机信息
    def get_host(self):
        """
        Set the HostInfo that this node represents
        """
        return self.get_data()

    # 设置节点代表的主机信息
    def set_host(self, host):
        """
        Set the HostInfo that this node represents
        """
        self.set_data(host)

    # 获取节点的极坐标角度
    def get_coordinate_theta(self):
        """
        """
        return self.__coordinate.get_theta()

    # 获取节点的极坐标半径
    def get_coordinate_radius(self):
        """
        """
        return self.__coordinate.get_radius()

    # 设置节点的极坐标角度
    def set_coordinate_theta(self, value):
        """
        """
        self.__coordinate.set_theta(value)

    # 设置节点的极坐标半径
    def set_coordinate_radius(self, value):
        """
        """
        self.__coordinate.set_radius(value)

    # 设置节点的极坐标
    def set_polar_coordinate(self, r, t):
        """
        Set polar coordinate
        @type  r: number
        @param r: The radius of coordinate
        @type  t: number
        @param t: The angle (theta) of coordinate in radians
        """
        self.__coordinate.set_coordinate(r, t)

    # 获取节点的极坐标
    def get_polar_coordinate(self):
        """
        Get cartesian coordinate
        @rtype: tuple
        @return: Cartesian coordinates (x, y)
        """
        return self.__coordinate.get_coordinate()

    # 设置节点的直角坐标
    def set_cartesian_coordinate(self, x, y):
        """
        Set cartesian coordinate
        """
        # 将直角坐标转换为极坐标，并设置节点的极坐标
        cartesian = CartesianCoordinate(x, y)
        r, t = cartesian.to_polar()

        self.set_polar_coordinate(r, math.degrees(t))

    # 获取节点的直角坐标
    def get_cartesian_coordinate(self):
        """
        Get cartesian coordinate
        @rtype: tuple
        @return: Cartesian coordinates (x, y)
        """
        # 获取节点的直角坐标
        return self.__coordinate.to_cartesian()
    def get_draw_info(self, info=None):
        """
        Get draw information about node
        @type  : string
        @param : Information name
        @rtype: mixed
        @return: The requested information
        """
        # 如果未指定信息名，则返回节点的绘制信息
        if info is None:
            return self.__draw_info
        # 否则返回指定信息名对应的信息
        return self.__draw_info.get(info)

    def set_draw_info(self, info):
        """
        Set draw information
        @type  : dict
        @param : Draw information dictionary
        """
        # 遍历传入的信息字典，设置节点的绘制信息
        for key in info:
            self.__draw_info[key] = info[key]

    def deep_search_child(self, node):
        """
        """
        # 遍历节点的子节点
        for child in self.get_draw_info('children'):
            # 如果子节点等于指定节点，则返回 True
            if child == node:
                return True
            # 否则递归调用子节点的 deep_search_child 方法
            elif child.deep_search_child(node):
                return True
        # 如果未找到指定节点，则返回 False
        return False

    def set_subtree_info(self, info):
        """
        """
        # 遍历节点的子节点
        for child in self.get_draw_info('children'):
            # 设置子节点的绘制信息
            child.set_draw_info(info)
            # 如果子节点不是分组节点，则递归调用子节点的 set_subtree_info 方法
            if not child.get_draw_info('group'):
                child.set_subtree_info(info)

    def calc_needed_space(self):
        """
        """
        # 获取节点的子节点数量
        number_of_children = len(self.get_draw_info('children'))

        sum_angle = 0
        own_angle = 0

        # 如果节点有子节点且不是分组节点
        if number_of_children > 0 and not self.get_draw_info('group'):
            # 遍历节点的子节点
            for child in self.get_draw_info('children'):
                # 递归调用子节点的 calc_needed_space 方法
                child.calc_needed_space()
                # 计算子节点所需空间的总和
                sum_angle += child.get_draw_info('space_need')

        # 计算节点的距离和大小对应的角度
        distance = self.get_coordinate_radius()
        size = self.get_draw_info('radius') * 2
        own_angle = geometry.angle_from_object(distance, size)

        # 设置节点的绘制信息，包括子节点所需空间和节点自身所需空间的最大值
        self.set_draw_info({'children_need': sum_angle})
        self.set_draw_info({'space_need': max(sum_angle, own_angle)})
```