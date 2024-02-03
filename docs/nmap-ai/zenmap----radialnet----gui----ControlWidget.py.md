# `nmap\zenmap\radialnet\gui\ControlWidget.py`

```cpp
# 设置文件编码为 UTF-8

# 重要提示：NMAP 许可证条款
# Nmap 安全扫描器由 Nmap 软件有限责任公司（“Nmap 项目”）（C）1996-2023 年开发。Nmap 也是 Nmap 项目的注册商标。
# 本程序根据 Nmap 公共源代码许可证（NPSL）的条款分发。适用于特定 Nmap 发行版或源代码控制修订版本的确切许可证文本包含在该版本的 Nmap 或源代码控制修订版本的 LICENSE 文件中。有关更多 Nmap 版权/法律信息，请访问 https://nmap.org/book/man-legal.html，有关 NPSL 许可证本身的更多信息，请访问 https://nmap.org/npsl/。此标题总结了 Nmap 许可证的一些关键点，但不能替代实际的许可证文本。
# Nmap 通常允许最终用户自行下载和使用，包括商业用途。可从 https://nmap.org 获取。
# Nmap 许可证通常禁止公司在商业产品中使用和重新分发 Nmap，但我们销售具有更宽松许可证和特殊功能的特殊 Nmap OEM 版本。请参阅 https://nmap.org/oem/
# 如果您收到了书面的 Nmap 许可协议或合同，其中规定了与这些条款（例如 Nmap OEM 许可证）不同的条款，您可以选择根据这些条款使用和重新分发 Nmap。
# 官方的 Nmap Windows 构建包括 Npcap 软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特殊许可就重新分发的单独许可条款的约束。因此，官方的 Nmap Windows 构建可能不得未经特殊许可（例如 Nmap OEM 许可证）重新分发。
# 我们提供此软件的源代码，因为我们相信用户有权根据自己的需求修改和重新分发它。因此，我们希望用户能够了解并遵守 Nmap 许可证的条款。
# 导入gi模块，确保使用的是Gtk 3.0版本
import gi

gi.require_version("Gtk", "3.0")
# 从gi.repository中导入Gtk、GLib和Gdk
from gi.repository import Gtk, GLib, Gdk

# 导入math模块
import math

# 导入radialnet.util.geometry模块中的所有内容
import radialnet.util.geometry as geometry

# 从radialnet.bestwidgets.boxes中导入所有内容
from radialnet.bestwidgets.boxes import *

# 从radialnet.core.Coordinate中导入PolarCoordinate
from radialnet.core.Coordinate import PolarCoordinate

# 从radialnet.gui.RadialNet中导入所有内容
import radialnet.gui.RadialNet as RadialNet

# 从radialnet.bestwidgets.expanders中导入BWExpander
from radialnet.bestwidgets.expanders import BWExpander

# 定义一个OPTIONS列表
OPTIONS = ['address',
           'hostname',
           'icon',
           'latency',
           'ring',
           'region',
           'slow in/out']

# 定义刷新频率为500
REFRESH_RATE = 500

# 定义一个名为ControlWidget的类，继承自BWVBox
class ControlWidget(BWVBox):
    """
    """
    # 初始化方法，接受 radialnet 参数
    def __init__(self, radialnet):
        # 调用父类的初始化方法
        BWVBox.__init__(self)
        # 设置边框宽度为6
        self.set_border_width(6)
    
        # 将 radialnet 参数赋给实例变量 radialnet
        self.radialnet = radialnet
    
        # 调用私有方法创建小部件
        self.__create_widgets()
    
    # 创建小部件的私有方法
    def __create_widgets(self):
        # 创建 ControlAction 实例并传入 radialnet 参数
        self.__action = ControlAction(self.radialnet)
        # 创建 ControlInterpolation 实例并传入 radialnet 参数
        self.__interpolation = ControlInterpolation(self.radialnet)
        # 创建 ControlLayout 实例并传入 radialnet 参数
        self.__layout = ControlLayout(self.radialnet)
        # 创建 ControlView 实例并传入 radialnet 参数
        self.__view = ControlView(self.radialnet)
    
        # 将创建的小部件添加到布局中
        self.bw_pack_start_noexpand_nofill(self.__action)
        self.bw_pack_start_noexpand_nofill(self.__interpolation)
        self.bw_pack_start_noexpand_nofill(self.__layout)
        self.bw_pack_start_noexpand_nofill(self.__view)
class ControlAction(BWExpander):
    """
    控制动作类，继承自BWExpander
    """
    def __init__(self, radialnet):
        """
        控制动作类的初始化方法
        """
        BWExpander.__init__(self, _('Action'))
        # 设置控制动作类为展开状态
        self.set_expanded(True)

        self.radialnet = radialnet

        # 创建控件
        self.__create_widgets()

    def __change_pointer(self, widget, pointer):
        """
        改变指针状态的私有方法
        """
        if pointer != self.radialnet.get_pointer_status():
            self.radialnet.set_pointer_status(pointer)

        if pointer == RadialNet.POINTER_FILL:
            self.__region_color.show()
        else:
            self.__region_color.hide()

    def __change_region(self, widget):
        """
        改变区域颜色的私有方法
        """
        self.radialnet.set_region_color(self.__region_color.get_active())


class ControlVariableWidget(Gtk.DrawingArea):
    """
    控制变量小部件类，继承自Gtk.DrawingArea
    """
    def __init__(self, name, value, update, increment=1):
        """
        控制变量小部件类的初始化方法
        """
        Gtk.DrawingArea.__init__(self)

        self.__variable_name = name
        self.__value = value
        self.__update = update
        self.__increment_pass = increment

        self.__radius = 6
        self.__increment_time = 100

        self.__pointer_position = 0
        self.__active_increment = False

        self.__last_value = self.__value()

        # 连接绘制事件、鼠标按下事件、鼠标释放事件、鼠标移动事件
        self.connect('draw', self.draw)
        self.connect('button_press_event', self.button_press)
        self.connect('button_release_event', self.button_release)
        self.connect('motion_notify_event', self.motion_notify)

        # 添加事件掩码
        self.add_events(Gdk.EventMask.BUTTON_PRESS_MASK |
                        Gdk.EventMask.BUTTON_RELEASE_MASK |
                        Gdk.EventMask.POINTER_MOTION_HINT_MASK |
                        Gdk.EventMask.POINTER_MOTION_MASK)

        # 定时检查数值变化
        GLib.timeout_add(REFRESH_RATE, self.verify_value)

    def verify_value(self):
        """
        验证数值变化的方法
        """
        if self.__value() != self.__last_value:
            self.__last_value = self.__value()

        self.queue_draw()

        return True
    # 按钮按下事件处理函数
    def button_press(self, widget, event):
        """
        """
        # 禁用增量标志
        self.__active_increment = False
        # 获取鼠标指针位置
        pointer = self.get_pointer()

        # 如果鼠标指针在按钮上点击并且是左键
        if self.__button_is_clicked(pointer) and event.button == 1:
            # 设置鼠标指针样式为手型
            event.window.set_cursor(Gdk.Cursor(Gdk.CursorType.HAND2))
            # 启用增量标志
            self.__active_increment = True
            # 增加值
            self.__increment_value()

    # 按钮释放事件处理函数
    def button_release(self, widget, event):
        """
        """
        # 设置鼠标指针样式为默认
        event.window.set_cursor(Gdk.Cursor(Gdk.CursorType.LEFT_PTR))
        # 禁用增量标志
        self.__active_increment = False
        # 重置指针位置
        self.__pointer_position = 0
        # 请求重新绘制
        self.queue_draw()

    # 鼠标移动事件处理函数
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
        # 如果启用了增量标志
        if self.__active_increment:
            # 获取窗口中心坐标
            xc, yc = self.__center_of_widget
            # 获取鼠标指针位置
            x, _ = self.get_pointer()

            # 如果鼠标指针在有效范围内
            if x - self.__radius > 0 and x + self.__radius < 2 * xc:
                # 更新指针位置
                self.__pointer_position = x - xc

        # 请求重新绘制
        self.queue_draw()

    # 绘制函数
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
        # 设置组件大小
        self.set_size_request(100, 30)
        # 调用绘制方法
        self.__draw(context)
        # 返回事件传播指示
        return True
    # 在给定上下文中绘制图形
    def __draw(self, context):
        """
        """
        # 获取分配给部件的空间
        allocation = self.get_allocation()

        # 计算部件中心点的坐标
        self.__center_of_widget = (allocation.width // 2,
                                   allocation.height // 2)

        xc, yc = self.__center_of_widget

        # 绘制线条
        context.set_line_width(1)
        context.set_dash([1, 2])
        context.move_to(self.__radius,
                             yc + self.__radius)
        context.line_to(2 * xc - 5,
                             yc + self.__radius)
        context.stroke()

        # 绘制文本
        context.set_dash([1, 0])
        context.set_font_size(10)

        context.move_to(5, yc - self.__radius)
        context.show_text(self.__variable_name)

        width = context.text_extents(str(self.__value()))[2]
        context.move_to(2 * xc - width - 5, yc - self.__radius)
        context.show_text(str(self.__value()))

        context.set_line_width(1)
        context.stroke()

        # 绘制节点
        context.arc(xc + self.__pointer_position,
                         yc + self.__radius,
                         self.__radius, 0, 2 * math.pi)
        if self.__active_increment:
            context.set_source_rgb(0.0, 0.0, 0.0)
        else:
            context.set_source_rgb(1.0, 1.0, 1.0)
        context.fill_preserve()
        context.set_source_rgb(0.0, 0.0, 0.0)
        context.stroke()

    # 检查按钮是否被点击
    def __button_is_clicked(self, pointer):
        """
        """
        xc, yc = self.__center_of_widget
        center = (xc, yc + self.__radius)

        return geometry.is_in_circle(pointer, 6, center)

    # 增加数值
    def __increment_value(self):
        """
        """
        self.__update(self.__value() + self.__pointer_position / 4)

        self.queue_draw()

        if self.__active_increment:
            # 设置定时器，以便在一定时间后再次调用增加数值的函数
            GLib.timeout_add(self.__increment_time,
                                self.__increment_value)

    # 设置数值函数
    def set_value_function(self, value):
        """
        """
        self.__value = value
    # 设置更新函数，将传入的更新函数赋值给对象的私有属性__update
    def set_update_function(self, update):
        self.__update = update
# 创建一个名为 ControlVariable 的类，继承自 BWHBox
class ControlVariable(BWHBox):
    """
    """
    # 初始化方法，接受参数 name, get_function, set_function, increment，默认值为1
    def __init__(self, name, get_function, set_function, increment=1):
        """
        """
        # 调用父类 BWHBox 的初始化方法，设置间距为0
        BWHBox.__init__(self, spacing=0)

        # 设置增量传递值为参数 increment
        self.__increment_pass = increment
        # 设置增量时间为200
        self.__increment_time = 200
        # 设置增量为 False
        self.__increment = False

        # 设置名称为参数 name
        self.__name = name
        # 设置获取函数为参数 get_function
        self.__get_function = get_function
        # 设置设置函数为参数 set_function
        self.__set_function = set_function

        # 调用私有方法 __create_widgets
        self.__create_widgets()
    # 创建控件和按钮
    def __create_widgets(self):
        """
        """
        # 创建控件对象
        self.__control = ControlVariableWidget(self.__name,
                                               self.__get_function,
                                               self.__set_function,
                                               self.__increment_pass)

        # 创建左侧按钮
        self.__left_button = Gtk.Button()
        self.__left_button.set_size_request(20, 20)
        # 创建左箭头图标
        self.__left_arrow = Gtk.Image.new_from_icon_name("pan-start-symbolic",
                                                         Gtk.IconSize.BUTTON);
        self.__left_button.add(self.__left_arrow)
        # 绑定左按钮的按下和释放事件
        self.__left_button.connect('pressed',
                                   self.__pressed,
                                   -self.__increment_pass)
        self.__left_button.connect('released', self.__released)

        # 创建右侧按钮
        self.__right_button = Gtk.Button()
        self.__right_button.set_size_request(20, 20)
        # 创建右箭头图标
        self.__right_arrow = Gtk.Image.new_from_icon_name("pan-end-symbolic",
                                                          Gtk.IconSize.BUTTON);
        self.__right_button.add(self.__right_arrow)
        # 绑定右按钮的按下和释放事件
        self.__right_button.connect('pressed',
                                    self.__pressed,
                                    self.__increment_pass)
        self.__right_button.connect('released', self.__released)

        # 将左按钮、控件和右按钮添加到布局中
        self.bw_pack_start_noexpand_nofill(self.__left_button)
        self.bw_pack_start_expand_fill(self.__control)
        self.bw_pack_start_noexpand_nofill(self.__right_button)

    # 处理按钮按下事件
    def __pressed(self, widget, increment):
        """
        """
        # 设置增量标志为True，并调用增量函数
        self.__increment = True
        self.__increment_function(increment)
    # 定义一个私有方法，用于增加函数的值
    def __increment_function(self, increment):
        """
        """
        # 如果增量标志为真
        if self.__increment:
            # 调用私有方法设置函数的值为原值加上增量
            self.__set_function(self.__get_function() + increment)
            # 调用控制方法验证数值
            self.__control.verify_value()
            # 使用GLib定时器，在一定时间后调用增量函数
            GLib.timeout_add(self.__increment_time,
                                self.__increment_function,
                                increment)
    
    # 定义一个私有方法，用于处理释放事件
    def __released(self, widget):
        """
        """
        # 将增量标志设置为假
        self.__increment = False
class ControlFisheye(BWVBox):
    """
    控制鱼眼效果的类，继承自BWVBox
    """
    def __init__(self, radialnet):
        """
        初始化方法，接受radialnet参数
        """
        BWVBox.__init__(self)  # 调用父类的初始化方法
        self.set_border_width(6)  # 设置边框宽度为6

        self.radialnet = radialnet  # 初始化radialnet属性
        self.__ring_max_value = self.radialnet.get_number_of_rings()  # 获取环的最大值

        self.__create_widgets()  # 调用私有方法创建小部件

    def __update_fisheye(self):
        """
        更新鱼眼效果的私有方法
        """
        # 调整环的比例以适应radialnet节点的数量
        ring_max_value = self.radialnet.get_number_of_rings() - 1

        if ring_max_value != self.__ring_max_value:

            value = self.__ring.get_value()

            if value == 0 and ring_max_value != 0:
                value = 1

            elif value > ring_max_value:
                value = ring_max_value

            self.__ring.configure(value, 1, ring_max_value, 0.01, 0.01, 0)
            self.__ring_max_value = ring_max_value

            self.__ring_scale.queue_draw()

        # 检查环的值
        ring_value = self.radialnet.get_fisheye_ring()

        if self.__ring.get_value() != ring_value:
            self.__ring.set_value(ring_value)

        # 检查兴趣值
        interest_value = self.radialnet.get_fisheye_interest()

        if self.__interest.get_value() != interest_value:
            self.__interest.set_value(interest_value)

        # 检查扩散值
        spread_value = self.radialnet.get_fisheye_spread()

        if self.__spread.get_value() != spread_value:
            self.__spread.set_value(spread_value)

        return True

    def active_fisheye(self):
        """
        激活鱼眼效果的方法
        """
        self.radialnet.set_fisheye(True)  # 设置radialnet的鱼眼效果为True
        self.__change_ring()  # 调用私有方法改变环
        self.__change_interest()  # 调用私有方法改变兴趣

    def deactive_fisheye(self):
        """
        停用鱼眼效果的方法
        """
        self.radialnet.set_fisheye(False)  # 设置radialnet的鱼眼效果为False
    # 改变环形网络的参数，根据传入的小部件值
    def __change_ring(self, widget=None):
        """
        """
        # 如果环形网络不在动画中
        if not self.radialnet.is_in_animation():
            # 设置鱼眼效果的环形参数
            self.radialnet.set_fisheye_ring(self.__ring.get_value())
        else:
            # 否则，将小部件值设置为环形网络的鱼眼效果环形参数
            self.__ring.set_value(self.radialnet.get_fisheye_ring())
    
    # 改变环形网络的参数，根据传入的小部件值
    def __change_interest(self, widget=None):
        """
        """
        # 如果环形网络不在动画中
        if not self.radialnet.is_in_animation():
            # 设置鱼眼效果的兴趣点参数
            self.radialnet.set_fisheye_interest(self.__interest.get_value())
        else:
            # 否则，将小部件值设置为环形网络的鱼眼效果兴趣点参数
            self.__interest.set_value(self.radialnet.get_fisheye_interest())
    
    # 改变环形网络的参数，根据传入的小部件值
    def __change_spread(self, widget=None):
        """
        """
        # 如果环形网络不在动画中
        if not self.radialnet.is_in_animation():
            # 设置鱼眼效果的扩散参数
            self.radialnet.set_fisheye_spread(self.__spread.get_value())
        else:
            # 否则，将小部件值设置为环形网络的鱼眼效果扩散参数
            self.__spread.set_value(self.radialnet.get_fisheye_spread())
class ControlInterpolation(BWExpander):
    """
    控制插值的类，继承自BWExpander
    """
    def __init__(self, radialnet):
        """
        初始化方法，接受radialnet参数
        """
        BWExpander.__init__(self, _('Interpolation'))

        self.radialnet = radialnet

        self.__create_widgets()

    def __create_widgets(self):
        """
        创建小部件的私有方法
        """
        self.__vbox = BWVBox()

        self.__cartesian_radio = Gtk.RadioButton(group=None,
                                                 label=_('Cartesian'))
        self.__polar_radio = Gtk.RadioButton(group=self.__cartesian_radio,
                                             label=_('Polar'))
        self.__cartesian_radio.connect('toggled',
                                       self.__change_system,
                                       RadialNet.INTERPOLATION_CARTESIAN)
        self.__polar_radio.connect('toggled',
                                   self.__change_system,
                                   RadialNet.INTERPOLATION_POLAR)

        self.__system_box = BWHBox()
        self.__system_box.bw_pack_start_noexpand_nofill(self.__polar_radio)
        self.__system_box.bw_pack_start_noexpand_nofill(self.__cartesian_radio)

        self.__frames_box = BWHBox()
        self.__frames_label = Gtk.Label.new(_('Frames'))
        self.__frames_label.set_alignment(0.0, 0.5)
        self.__frames = Gtk.Adjustment.new(
            self.radialnet.get_number_of_frames(), 1, 1000, 1, 0, 0)
        self.__frames.connect('value_changed', self.__change_frames)
        self.__frames_spin = Gtk.SpinButton(adjustment=self.__frames)
        self.__frames_box.bw_pack_start_expand_fill(self.__frames_label)
        self.__frames_box.bw_pack_start_noexpand_nofill(self.__frames_spin)

        self.__vbox.bw_pack_start_noexpand_nofill(self.__frames_box)
        self.__vbox.bw_pack_start_noexpand_nofill(self.__system_box)

        self.bw_add(self.__vbox)

        GLib.timeout_add(REFRESH_RATE, self.__update_animation)
    # 更新动画状态
    def __update_animation(self):
        """
        """
        # 获取插值方式
        active = self.radialnet.get_interpolation()

        # 如果插值方式为笛卡尔坐标系
        if active == RadialNet.INTERPOLATION_CARTESIAN:
            # 设置笛卡尔坐标系单选按钮为选中状态
            self.__cartesian_radio.set_active(True)

        # 如果插值方式为极坐标系
        else:
            # 设置极坐标系单选按钮为选中状态
            self.__polar_radio.set_active(True)

        # 返回 True
        return True

    # 改变系统
    def __change_system(self, widget, value):
        """
        """
        # 如果设置插值方式失败
        if not self.radialnet.set_interpolation(value):

            # 获取当前插值方式
            active = self.radialnet.get_interpolation()

            # 如果插值方式为笛卡尔坐标系
            if active == RadialNet.INTERPOLATION_CARTESIAN:
                # 设置笛卡尔坐标系单选按钮为选中状态
                self.__cartesian_radio.set_active(True)

            # 如果插值方式为极坐标系
            else:
                # 设置极坐标系单选按钮为选中状态
                self.__polar_radio.set_active(True)

    # 改变帧数
    def __change_frames(self, widget):
        """
        """
        # 如果设置帧数失败
        if not self.radialnet.set_number_of_frames(self.__frames.get_value()):
            # 重置帧数为当前系统的帧数
            self.__frames.set_value(self.radialnet.get_number_of_frames())
# 创建一个名为 ControlLayout 的类，继承自 BWExpander
class ControlLayout(BWExpander):
    """
    """
    # 初始化方法，接受 radialnet 参数
    def __init__(self, radialnet):
        """
        """
        # 调用父类的初始化方法，传入字符串 'Layout'
        BWExpander.__init__(self, _('Layout'))

        # 将 radialnet 参数赋值给实例变量 radialnet
        self.radialnet = radialnet

        # 调用私有方法 __create_widgets()
        self.__create_widgets()

    # 创建私有方法 __create_widgets()
    def __create_widgets(self):
        """
        """
        # 创建一个水平布局对象 BWHBox
        self.__hbox = BWHBox()

        # 创建一个下拉框对象 Gtk.ComboBoxText
        self.__layout = Gtk.ComboBoxText()
        # 向下拉框添加选项 'Symmetric' 和 'Weighted'
        self.__layout.append_text(_('Symmetric')
        self.__layout.append_text(_('Weighted'))
        # 设置下拉框的默认选项为 radialnet 的布局类型
        self.__layout.set_active(self.radialnet.get_layout())
        # 连接下拉框的 'changed' 信号到私有方法 __change_layout
        self.__layout.connect('changed', self.__change_layout)
        # 创建一个工具按钮对象 Gtk.ToolButton，使用 Gtk.STOCK_REFRESH 图标
        self.__force = Gtk.ToolButton(stock_id=Gtk.STOCK_REFRESH)
        # 连接工具按钮的 'clicked' 信号到私有方法 __force_update
        self.__force.connect('clicked', self.__force_update)

        # 将下拉框添加到水平布局中，填充并扩展
        self.__hbox.bw_pack_start_expand_fill(self.__layout)
        # 将工具按钮添加到水平布局中，不扩展也不填充
        self.__hbox.bw_pack_start_noexpand_nofill(self.__force)

        # 将水平布局添加到当前对象中
        self.bw_add(self.__hbox)

        # 调用私有方法 __check_layout()
        self.__check_layout()

    # 创建私有方法 __check_layout()
    def __check_layout(self):
        """
        """
        # 如果下拉框的当前选项是 RadialNet.LAYOUT_WEIGHTED，则使工具按钮可用
        if self.__layout.get_active() == RadialNet.LAYOUT_WEIGHTED:
            self.__force.set_sensitive(True)
        # 否则使工具按钮不可用
        else:
            self.__force.set_sensitive(False)

        # 返回 True
        return True

    # 创建私有方法 __force_update，接受 widget 参数
    def __force_update(self, widget):
        """
        """
        # 将 radialnet 的 fisheye_ring 赋值给实例变量 __fisheye_ring
        self.__fisheye_ring = self.radialnet.get_fisheye_ring()
        # 调用 radialnet 的 update_layout 方法
        self.radialnet.update_layout()

    # 创建私有方法 __change_layout，接受 widget 参数
    def __change_layout(self, widget):
        """
        """
        # 如果 radialnet 的 set_layout 方法返回 False，则将下拉框的选项设置为 radialnet 的布局类型
        if not self.radialnet.set_layout(self.__layout.get_active()):
            self.__layout.set_active(self.radialnet.get_layout())
        # 否则调用私有方法 __check_layout()
        else:
            self.__check_layout()


# 创建一个名为 ControlRingGap 的类，继承自 BWVBox
class ControlRingGap(BWVBox):
    """
    """
    # 初始化方法，接受 radialnet 参数
    def __init__(self, radialnet):
        """
        """
        # 调用父类的初始化方法，传入字符串 'Layout'
        BWVBox.__init__(self)

        # 将 radialnet 参数赋值给实例变量 radialnet
        self.radialnet = radialnet

        # 调用私有方法 __create_widgets()
        self.__create_widgets()
    # 创建小部件
    def __create_widgets(self):
        # 创建控制变量对象，用于控制环形网络的环间距
        self.__radius = ControlVariable(_('Ring gap'),
                                        self.radialnet.get_ring_gap,
                                        self.radialnet.set_ring_gap)

        # 创建标签对象，显示“Lower ring gap”
        self.__label = Gtk.Label.new(_('Lower ring gap'))
        # 设置标签对象的对齐方式
        self.__label.set_alignment(0.0, 0.5)
        # 创建调整对象，用于调整环形网络的最小环间距
        self.__adjustment = Gtk.Adjustment.new(
            self.radialnet.get_min_ring_gap(), 0, 50, 1, 0, 0)
        # 创建微调按钮对象，与调整对象关联
        self.__spin = Gtk.SpinButton(adjustment=self.__adjustment)
        # 连接微调按钮的数值变化信号与指定的回调函数
        self.__spin.connect('value_changed', self.__change_lower)

        # 创建水平盒子对象
        self.__lower_hbox = BWHBox()
        # 将标签对象添加到水平盒子中，可扩展并填充
        self.__lower_hbox.bw_pack_start_expand_fill(self.__label)
        # 将微调按钮对象添加到水平盒子中，不可扩展且不填充
        self.__lower_hbox.bw_pack_start_noexpand_nofill(self.__spin)

        # 将控制变量对象添加到当前对象中，不可扩展且不填充
        self.bw_pack_start_noexpand_nofill(self.__radius)
        # 将水平盒子对象添加到当前对象中，不可扩展且不填充
        self.bw_pack_start_noexpand_nofill(self.__lower_hbox)

    # 修改最小环间距的回调函数
    def __change_lower(self, widget):
        # 如果无法设置最小环间距，则将调整对象的值设置为环形网络的最小环间距
        if not self.radialnet.set_min_ring_gap(self.__adjustment.get_value()):
            self.__adjustment.set_value(self.radialnet.get_min_ring_gap())
# 创建名为 ControlOptions 的类，继承自 BWScrolledWindow
class ControlOptions(BWScrolledWindow):
    """
    """

    # 初始化方法，接受 radialnet 参数
    def __init__(self, radialnet):
        """
        """
        # 调用父类的初始化方法
        BWScrolledWindow.__init__(self)

        # 设置滚动窗口的滚动条策略
        self.set_policy(Gtk.PolicyType.AUTOMATIC, Gtk.PolicyType.ALWAYS)
        # 设置滚动窗口的阴影类型
        self.set_shadow_type(Gtk.ShadowType.NONE)

        # 将 radialnet 参数赋值给实例变量 radialnet
        self.radialnet = radialnet

        # 设置实例变量 enable_labels 为 True
        self.enable_labels = True

        # 调用私有方法 __create_widgets()
        self.__create_widgets()
    # 创建小部件
    def __create_widgets(self):
        # 创建一个新的列表存储对象，包含布尔值和字符串类型
        self.__liststore = Gtk.ListStore.new([bool, str])

        # 向列表存储对象中添加数据
        self.__liststore.append([None, OPTIONS[0]])
        self.__liststore.append([None, OPTIONS[1]])
        self.__liststore.append([None, OPTIONS[2]])
        self.__liststore.append([None, OPTIONS[3]])
        self.__liststore.append([None, OPTIONS[4]])
        self.__liststore.append([None, OPTIONS[5]])
        self.__liststore.append([None, OPTIONS[6])

        # 创建一个可切换的单元格渲染器
        self.__cell_toggle = Gtk.CellRendererToggle()
        self.__cell_toggle.set_property('activatable', True)
        self.__cell_toggle.connect('toggled',
                                   self.__change_option,
                                   self.__liststore)

        # 创建一个树视图列，使用单元格渲染器
        self.__column_toggle = Gtk.TreeViewColumn(cell_renderer=self.__cell_toggle)
        self.__column_toggle.add_attribute(self.__cell_toggle, 'active', 0)
        self.__column_toggle.set_cell_data_func(self.__cell_toggle, self.__cell_toggle_data_method)

        # 创建一个文本单元格渲染器
        self.__cell_text = Gtk.CellRendererText()

        # 创建一个树视图列，使用文本单元格渲染器
        self.__column_text = Gtk.TreeViewColumn(title=None,
                                                cell_renderer=self.__cell_text,
                                                text=1)
        self.__column_text.set_cell_data_func(self.__cell_text, self.__cell_text_data_method)

        # 创建一个带有模型的树视图
        self.__treeview = Gtk.TreeView.new_with_model(self.__liststore)
        self.__treeview.set_enable_search(True)
        self.__treeview.set_search_column(1)
        self.__treeview.set_headers_visible(False)
        self.__treeview.append_column(self.__column_toggle)
        self.__treeview.append_column(self.__column_text)

        # 将树视图添加到视口中
        self.add_with_viewport(self.__treeview)

        # 添加一个定时器，用于更新选项
        GLib.timeout_add(REFRESH_RATE, self.__update_options)
    # 定义一个私有方法，用于切换单元格数据的方法
    def __cell_toggle_data_method(self, column, cell, model, it, data):
        # 如果不启用标签，并且模型中的值为'hostname'，则设置单元格不可激活
        if not self.enable_labels and model.get_value(it, 1) == 'hostname':
            cell.set_property('activatable', False)
        else:
            # 否则设置单元格可激活
            cell.set_property('activatable', True)

    # 定义一个私有方法，用于设置单元格文本数据的方法
    def __cell_text_data_method(self, column, cell, model, it, data):
        # 如果不启用标签，并且模型中的值为'hostname'，则设置单元格文本为删除线
        if not self.enable_labels and model.get_value(it, 1) == 'hostname':
            cell.set_property('strikethrough', True)
        else:
            # 否则取消单元格文本的删除线
            cell.set_property('strikethrough', False)

    # 定义一个私有方法，用于更新选项
    def __update_options(self):
        """
        """
        # 获取列表存储模型
        model = self.__liststore

        # 更新各个选项的值
        model[OPTIONS.index('address')][0] = self.radialnet.get_show_address()
        model[OPTIONS.index('hostname')][0] = self.enable_labels and \
                self.radialnet.get_show_hostname()
        model[OPTIONS.index('icon')][0] = self.radialnet.get_show_icon()
        model[OPTIONS.index('latency')][0] = self.radialnet.get_show_latency()
        model[OPTIONS.index('ring')][0] = self.radialnet.get_show_ring()
        model[OPTIONS.index('region')][0] = self.radialnet.get_show_region()
        model[OPTIONS.index('slow in/out')][0] = \
                self.radialnet.get_slow_inout()

        # 返回 True
        return True
    # 定义一个私有方法，用于改变选项的状态
    def __change_option(self, cell, option, model):
        """
        """
        # 将选项转换为整数类型
        option = int(option)
        # 更新模型中对应选项的状态
        model[option][0] = not model[option][0]

        # 根据选项类型进行不同的处理
        if OPTIONS[option] == 'address':
            # 设置 radialnet 对象是否显示地址
            self.radialnet.set_show_address(model[option][0])
            # 如果显示地址，则更新模型中的 hostname 状态
            if model[option][0]:
                model[OPTIONS.index('hostname')][0] = self.radialnet.get_show_hostname()
            else:
                model[OPTIONS.index('hostname')][0] = False
            # 更新标签的可见性
            self.enable_labels = model[option][0]

        elif OPTIONS[option] == 'hostname':
            # 设置 radialnet 对象是否显示主机名
            self.radialnet.set_show_hostname(model[option][0])

        elif OPTIONS[option] == 'icon':
            # 设置 radialnet 对象是否显示图标
            self.radialnet.set_show_icon(model[option][0])

        elif OPTIONS[option] == 'latency':
            # 设置 radialnet 对象是否显示延迟
            self.radialnet.set_show_latency(model[option][0])

        elif OPTIONS[option] == 'ring':
            # 设置 radialnet 对象是否显示环
            self.radialnet.set_show_ring(model[option][0])

        elif OPTIONS[option] == 'region':
            # 设置 radialnet 对象是否显示区域
            self.radialnet.set_show_region(model[option][0])

        elif OPTIONS[option] == 'slow in/out':
            # 设置 radialnet 对象是否显示慢速输入/输出
            self.radialnet.set_slow_inout(model[option][0])
# 创建一个名为 ControlView 的类，继承自 BWExpander
class ControlView(BWExpander):
    """
    """
    # 初始化方法，接受 radialnet 参数
    def __init__(self, radialnet):
        """
        """
        # 调用父类的初始化方法，设置标题为 'View'，并展开视图
        BWExpander.__init__(self, _('View'))
        self.set_expanded(True)

        # 将 radialnet 参数赋给实例变量 radialnet
        self.radialnet = radialnet

        # 调用私有方法 __create_widgets()
        self.__create_widgets()

    # 创建小部件的私有方法
    def __create_widgets(self):
        """
        """
        # 创建一个垂直布局的容器
        self.__vbox = BWVBox(spacing=0)

        # 创建一个名为 Zoom 的控制变量，传入标题、获取缩放值的方法和设置缩放值的方法
        self.__zoom = ControlVariable(_('Zoom'),
                                      self.radialnet.get_zoom,
                                      self.radialnet.set_zoom)

        # 创建一个 ControlRingGap 对象，传入 radialnet 参数
        self.__ring_gap = ControlRingGap(self.radialnet)
        # 创建一个 ControlNavigation 对象，传入 radialnet 参数
        self.__navigation = ControlNavigation(self.radialnet)

        # 创建一个 ControlOptions 对象，传入 radialnet 参数，并设置边框宽度为 0
        self.__options = ControlOptions(self.radialnet)
        self.__options.set_border_width(0)

        # 将各个小部件添加到垂直布局容器中
        self.__vbox.bw_pack_start_expand_nofill(self.__options)
        self.__vbox.bw_pack_start_noexpand_nofill(self.__navigation)
        self.__vbox.bw_pack_start_noexpand_nofill(self.__zoom)
        self.__vbox.bw_pack_start_noexpand_nofill(self.__ring_gap)

        # 将垂直布局容器添加到当前对象中
        self.bw_add(self.__vbox)


# 创建一个名为 ControlNavigation 的类，继承自 Gtk.DrawingArea
class ControlNavigation(Gtk.DrawingArea):
    """
    """
    # 初始化方法，接受一个 radialnet 参数
    def __init__(self, radialnet):
        """
        """
        # 调用父类的初始化方法
        Gtk.DrawingArea.__init__(self)

        # 将 radialnet 参数赋值给实例变量 radialnet
        self.radialnet = radialnet

        # 初始化一个 PolarCoordinate 对象，并设置坐标为 (40, 90)
        self.__rotate_node = PolarCoordinate()
        self.__rotate_node.set_coordinate(40, 90)
        # 设置小部件的中心点坐标为 (50, 50)
        self.__center_of_widget = (50, 50)
        # 初始化移动、居中、旋转等标志
        self.__moving = None
        self.__centering = False
        self.__rotating = False
        # 移动步长
        self.__move_pass = 100

        # 初始化移动位置、移动增量
        self.__move_position = (0, 0)
        self.__move_addition = [(-1, 0),
                                (-1, -1),
                                (0, -1),
                                (1, -1),
                                (1, 0),
                                (1, 1),
                                (0, 1),
                                (-1, 1)]
        # 移动因子和移动因子限制
        self.__move_factor = 1
        self.__move_factor_limit = 20

        # 旋转半径和移动半径
        self.__rotate_radius = 6
        self.__move_radius = 6

        # 初始化旋转和移动点击标志
        self.__rotate_clicked = False
        self.__move_clicked = None

        # 连接绘制、按钮按下、按钮释放、鼠标移动、鼠标进入、鼠标离开、键盘按下、键盘释放事件
        self.connect('draw', self.draw)
        self.connect('button_press_event', self.button_press)
        self.connect('button_release_event', self.button_release)
        self.connect('motion_notify_event', self.motion_notify)
        self.connect('enter_notify_event', self.enter_notify)
        self.connect('leave_notify_event', self.leave_notify)
        self.connect('key_press_event', self.key_press)
        self.connect('key_release_event', self.key_release)

        # 添加事件掩码
        self.add_events(Gdk.EventMask.BUTTON_PRESS_MASK |
                        Gdk.EventMask.BUTTON_RELEASE_MASK |
                        Gdk.EventMask.ENTER_NOTIFY_MASK |
                        Gdk.EventMask.LEAVE_NOTIFY_MASK  |
                        Gdk.EventMask.KEY_PRESS_MASK |
                        Gdk.EventMask.KEY_RELEASE_MASK |
                        Gdk.EventMask.POINTER_MOTION_HINT_MASK |
                        Gdk.EventMask.POINTER_MOTION_MASK)

        # 设置旋转节点的坐标为 (40, radialnet.get_rotation())
        self.__rotate_node.set_coordinate(40, self.radialnet.get_rotation())
    # 处理键盘按下事件的回调函数
    def key_press(self, widget, event):
        """
        """
        # 获取按下的键名
        # key = Gdk.keyval_name(event.keyval)

        # 重新绘制窗口
        self.queue_draw()

        # 返回 True 表示事件已处理
        return True

    # 处理键盘释放事件的回调函数
    def key_release(self, widget, event):
        """
        """
        # 获取释放的键名
        # key = Gdk.keyval_name(event.keyval)

        # 重新绘制窗口
        self.queue_draw()

        # 返回 True 表示事件已处理
        return True

    # 处理鼠标进入窗口事件的回调函数
    def enter_notify(self, widget, event):
        """
        """
        # 返回 False 表示事件未处理
        return False

    # 处理鼠标离开窗口事件的回调函数
    def leave_notify(self, widget, event):
        """
        """
        # 重新绘制窗口
        self.queue_draw()

        # 返回 False 表示事件未处理
        return False

    # 处理鼠标按下事件的回调函数
    def button_press(self, widget, event):
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

        # 初始化方向变量
        direction = False

        # 如果点击了旋转按钮
        if self.__rotate_is_clicked(pointer):

            # 设置鼠标样式为手型
            event.window.set_cursor(Gdk.Cursor(Gdk.CursorType.HAND2))
            self.__rotating = True

        # 判断是否点击了移动按钮
        direction = self.__move_is_clicked(pointer)

        # 如果点击了移动按钮且当前未处于移动状态
        if direction is not None and self.__moving is None:

            # 设置鼠标样式为手型
            event.window.set_cursor(Gdk.Cursor(Gdk.CursorType.HAND2))
            self.__moving = direction
            self.__move_in_direction(direction)

        # 如果点击了中心按钮
        if self.__center_is_clicked(pointer):

            # 设置鼠标样式为手型
            event.window.set_cursor(Gdk.Cursor(Gdk.CursorType.HAND2))
            self.__centering = True
            self.__move_position = (0, 0)
            self.radialnet.set_translation(self.__move_position)

        # 重新绘制窗口
        self.queue_draw()

        # 返回 False 表示事件未处理
        return False
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
        self.__moving = None        # 停止移动
        self.__centering = False
        self.__rotating = False     # 停止旋转
        self.__move_factor = 1

        event.window.set_cursor(Gdk.Cursor(Gdk.CursorType.LEFT_PTR))

        self.queue_draw()

        return False

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
        xc, yc = self.__center_of_widget
        x, y = self.get_pointer()

        status = not self.radialnet.is_in_animation()
        status = status and not self.radialnet.is_empty()

        if self.__rotating and status:

            r, t = self.__rotate_node.get_coordinate()
            t = math.degrees(math.atan2(yc - y, x - xc))

            if t < 0:
                t = 360 + t

            self.radialnet.set_rotation(t)
            self.__rotate_node.set_coordinate(r, t)

            self.queue_draw()

        return False

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
        self.set_size_request(120, 130)

        self.__draw(context)

        return False
    # 绘制旋转控制器
    def __draw_rotate_control(self, context):
        """
        """
        # 获取小部件中心坐标
        xc, yc = self.__center_of_widget
        # 获取旋转节点的极坐标
        r, t = self.__rotate_node.get_coordinate()
        # 将极坐标转换为笛卡尔坐标
        x, y = self.__rotate_node.to_cartesian()

        # 绘制文本
        context.set_font_size(10)
        context.move_to(xc - 49, yc - 48)
        context.show_text(_("Navigation"))

        # 计算文本宽度并绘制旋转角度
        width = context.text_extents(str(int(t)))[2]
        context.move_to(xc + 49 - width - 2, yc - 48)
        context.show_text(str(round(t, 1)))
        context.set_line_width(1)
        context.stroke()

        # 绘制圆弧
        context.set_dash([1, 2])
        context.arc(xc, yc, 40, 0, 2 * math.pi)
        context.set_source_rgb(0.0, 0.0, 0.0)
        context.set_line_width(1)
        context.stroke()

        # 绘制节点
        context.set_dash([1, 0])
        context.arc(xc + x, yc - y, self.__rotate_radius, 0, 2 * math.pi)

        # 根据旋转状态设置颜色并填充节点
        if self.__rotating:
            context.set_source_rgb(0.0, 0.0, 0.0)
        else:
            context.set_source_rgb(1.0, 1.0, 1.0)

        context.fill_preserve()
        context.set_source_rgb(0.0, 0.0, 0.0)
        context.set_line_width(1)
        context.stroke()

        # 返回 False
        return False
    # 绘制移动控制器
    def __draw_move_control(self, context):
        """
        """
        # 获取小部件中心坐标
        xc, yc = self.__center_of_widget
        # 创建极坐标对象
        pc = PolarCoordinate()

        # 设置虚线样式
        context.set_dash([1, 1])
        # 绘制圆弧
        context.arc(xc, yc, 23, 0, 2 * math.pi)
        # 设置线条颜色
        context.set_source_rgb(0.0, 0.0, 0.0)
        # 设置线条宽度
        context.set_line_width(1)
        # 绘制线条
        context.stroke()

        # 循环绘制8个移动控制器
        for i in range(8):
            # 设置极坐标的坐标和角度
            pc.set_coordinate(23, 45 * i)
            x, y = pc.to_cartesian()

            # 设置虚线样式
            context.set_dash([1, 1])
            # 移动到指定坐标
            context.move_to(xc, yc)
            # 绘制线条
            context.line_to(xc + x, yc - y)
            context.stroke()

            # 设置虚线样式
            context.set_dash([1, 0])
            # 绘制圆形
            context.arc(xc + x, yc - y, self.__move_radius, 0, 2 * math.pi)

            # 根据移动状态设置颜色
            if i == self.__moving:
                context.set_source_rgb(0.0, 0.0, 0.0)
            else:
                context.set_source_rgb(1.0, 1.0, 1.0)
            # 填充圆形
            context.fill_preserve()
            context.set_source_rgb(0.0, 0.0, 0.0)
            context.set_line_width(1)
            context.stroke()

        # 绘制中心圆形
        context.arc(xc, yc, 6, 0, 2 * math.pi)

        # 根据中心状态设置颜色
        if self.__centering:
            context.set_source_rgb(0.0, 0.0, 0.0)
        else:
            context.set_source_rgb(1.0, 1.0, 1.0)
        # 填充中心圆形
        context.fill_preserve()
        context.set_source_rgb(0.0, 0.0, 0.0)
        context.set_line_width(1)
        context.stroke()

        # 返回 False
        return False

    # 绘制方法
    def __draw(self, context):
        """
        Drawing method
        """
        # 获取分配引用
        allocation = self.get_allocation()

        # 计算小部件中心坐标
        self.__center_of_widget = (allocation.width // 2,
                                   allocation.height // 2)

        # 绘制旋转控制器
        self.__draw_rotate_control(context)
        # 绘制移动控制器
        self.__draw_move_control(context)

        # 返回 False
        return False
    # 在给定方向上移动对象
    def __move_in_direction(self, direction):
        """
        """
        # 如果对象正在移动
        if self.__moving is not None:
            # 获取当前位置
            bx, by = self.__move_position
            # 获取给定方向上的增量
            ax, ay = self.__move_addition[direction]
            # 更新位置
            self.__move_position = (bx + self.__move_factor * ax,
                                    by + self.__move_factor * ay)
            # 设置对象的新位置
            self.radialnet.set_translation(self.__move_position)

            # 如果移动因子小于限制值，则增加移动因子
            if self.__move_factor < self.__move_factor_limit:
                self.__move_factor += 1

            # 在一定时间后再次调用该函数，实现连续移动
            GLib.timeout_add(self.__move_pass,
                                self.__move_in_direction,
                                direction)

        return False

    # 检查旋转按钮是否被点击
    def __rotate_is_clicked(self, pointer):
        """
        """
        # 获取旋转节点的笛卡尔坐标
        xn, yn = self.__rotate_node.to_cartesian()
        xc, yc = self.__center_of_widget

        # 计算旋转按钮的中心点
        center = (xc + xn, yc - yn)
        return geometry.is_in_circle(pointer, self.__rotate_radius, center)

    # 检查中心按钮是否被点击
    def __center_is_clicked(self, pointer):
        """
        """
        # 检查点击位置是否在中心按钮内
        return geometry.is_in_circle(pointer, self.__move_radius,
                self.__center_of_widget)

    # 检查移动按钮是否被点击
    def __move_is_clicked(self, pointer):
        """
        """
        # 获取中心点坐标
        xc, yc = self.__center_of_widget
        pc = PolarCoordinate()

        # 遍历8个方向
        for i in range(8):
            # 设置极坐标
            pc.set_coordinate(23, 45 * i)
            x, y = pc.to_cartesian()

            # 计算每个方向按钮的中心点
            center = (xc + x, yc - y)
            # 检查点击位置是否在移动按钮内
            if geometry.is_in_circle(pointer, self.__move_radius, center):
                return i

        return None
```