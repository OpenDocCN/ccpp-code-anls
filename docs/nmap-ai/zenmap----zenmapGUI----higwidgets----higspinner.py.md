# `nmap\zenmap\zenmapGUI\higwidgets\higspinner.py`

```
# 指定脚本解释器为 Python 3
#!/usr/bin/env python3

# ***********************IMPORTANT NMAP LICENSE TERMS************************
# *
# * The Nmap Security Scanner is (C) 1996-2023 Nmap Software LLC ("The Nmap
# * Project"). Nmap is also a registered trademark of the Nmap Project.
# *
# * This program is distributed under the terms of the Nmap Public Source
# * License (NPSL). The exact license text applying to a particular Nmap
# * release or source code control revision is contained in the LICENSE
# * file distributed with that version of Nmap or source code control
# * revision. More Nmap copyright/legal information is available from
# * https://nmap.org/book/man-legal.html, and further information on the
# * NPSL license itself can be found at https://nmap.org/npsl/ . This
# * header summarizes some key points from the Nmap license, but is no
# * substitute for the actual license text.
# *
# * Nmap is generally free for end users to download and use themselves,
# * including commercial use. It is available from https://nmap.org.
# *
# * The Nmap license generally prohibits companies from using and
# * redistributing Nmap in commercial products, but we sell a special Nmap
# * OEM Edition with a more permissive license and special features for
# * this purpose. See https://nmap.org/oem/
# *
# * If you have received a written Nmap license agreement or contract
# * stating terms other than these (such as an Nmap OEM license), you may
# * choose to use and redistribute Nmap under those terms instead.
# *
# * The official Nmap Windows builds include the Npcap software
# * (https://npcap.com) for packet capture and transmission. It is under
# * separate license terms which forbid redistribution without special
# * permission. So the official Nmap Windows builds may not be redistributed
# * without special permission (such as an Nmap OEM license).
# *
# * Source is provided to this software because we believe users have a
# * right to know exactly what a program is going to do before they run it.
# 该部分是关于 Nmap 软件的一些版权和许可信息，以及对开发者提交代码的一些要求和说明
# 该文件是 higwidgets/higspinner.py，包含了一个基于 epiphany/nautilus 实现的 pygtk spinner
# 导入 gi 模块
import gi
# 要求导入 Gtk 版本 3.0
gi.require_version("Gtk", "3.0")
# 从 gi.repository 中导入 Gtk, GLib, Gdk, GdkPixbuf 模块
from gi.repository import Gtk, GLib, Gdk, GdkPixbuf

# 定义 HIGSpinnerImages 类
class HIGSpinnerImages:
    # 初始化方法，用于创建一个包含 GDK Pixbuffers 列表的类
    def __init__(self):
        """This class holds list of GDK Pixbuffers.

        - static_pixbufs is used for multiple static pixbuffers
        - self.animated_pixbufs is used for the pixbuffers that make up the
          animation
        """

        # 打印初始化信息
        dprint('HIGSpinnerImages::__init__')

        # 用于存储多个静态 pixbuffer 的字典
        self.static_pixbufs = {}

        # 用于存储默认的静态 pixbuffer
        self.rest_pixbuf = None

        # 用于存储动画中的 pixbuffer 列表
        self.animated_pixbufs = []

    # 添加静态 pixbuffer 的方法
    def add_static_pixbuf(self, name, pixbuf, default_on_rest=False):
        """Add a static pixbuf.

        If this is the first one, make it the default pixbuffer on rest.
        The user can make some other pixbuf the new default on rest, by setting
        default_on_rest to True.
        """

        # 打印添加静态 pixbuffer 的信息
        dprint('HIGSpinnerImages::add_static_pixbuf')

        # 将静态 pixbuffer 存储到字典中
        self.static_pixbufs[name] = pixbuf
        # 如果这是第一个静态 pixbuffer，或者设置了 default_on_rest 为 True，则将其设置为默认的静态 pixbuffer
        if (len(self.static_pixbufs) == 1) or default_on_rest:
            self.set_rest_pixbuf(name)

    # 添加动画 pixbuffer 的方法
    def add_animated_pixbuf(self, pixbuf):

        # 打印添加动画 pixbuffer 的信息
        dprint('HIGSpinnerImages::add_animated_pixbuf')

        # 将动画 pixbuffer 添加到列表中
        self.animated_pixbufs.append(pixbuf)

    # 设置默认的静态 pixbuffer 的方法
    def set_rest_pixbuf(self, name):
        """Sets the pixbuf that will be used on the default, 'rest' state. """

        # 打印设置默认的静态 pixbuffer 的信息
        dprint('HIGSpinnerImages::set_rest_pixbuf')

        # 如果指定的静态 pixbuffer 名称不存在，则抛出异常
        if name not in self.static_pixbufs:
            raise StaticPixbufNotFound

        # 将指定的静态 pixbuffer 设置为默认的静态 pixbuffer
        self.rest_pixbuf = self.static_pixbufs[name]
    # 设置每个 pixbuf（静态和动画）的大小
    def set_size(self, width, height):
        """Sets the size of each pixbuf (static and animated)"""
        # 创建一个新的动画 pixbuf 列表
        new_animated = []
        # 遍历动画 pixbuf 列表，将每个 pixbuf 缩放到指定的宽度和高度，并添加到新列表中
        for p in self.animated_pixbufs:
            new_animated.append(p.scale_simple(width, height, GdkPixbuf.InterpType.BILINEAR))
        # 将新的动画 pixbuf 列表赋值给原始的动画 pixbuf 列表
        self.animated_pixbufs = new_animated

        # 遍历静态 pixbuf 字典，将每个 pixbuf 缩放到指定的宽度和高度
        for k in self.static_pixbufs:
            self.static_pixbufs[k] = self.static_pixbufs[k].scale_simple(
                    width, height, GdkPixbuf.InterpType.BILINEAR)

        # 将 rest_pixbuf 缩放到指定的宽度和高度
        self.rest_pixbuf = self.rest_pixbuf.scale_simple(
                width, height, GdkPixbuf.InterpType.BILINEAR)

        # 更新图片的宽度和高度属性
        self.images_width = width
        self.images_height = height
# 定义一个名为 HIGSpinnerCache 的类，用于保存 HIGSpinners 实例中使用的图像副本
class HIGSpinnerCache:
    """This hols a copy of the images used on the HIGSpinners instances."""
    # 初始化方法
    def __init__(self):

        # 打印调试信息
        dprint('HIGSpinnerCache::__init__')

        # 创建一个 HIGSpinnerImages 的实例，用于保存图像
        self.spinner_images = HIGSpinnerImages()

        # 创建一个 Gtk.IconTheme 的实例，用于管理图标主题
        self.icon_theme = Gtk.IconTheme()
        # 初始化原始图像和图像为空
        self.originals = None
        self.images = None

        # 可能有一个“默认”的动画图标
        # 例如，如果我们在 GNOME 桌面上，并安装了“gnome-icon-theme”包，可能会有访问“gnome-spinner”的权限
        # 在使用之前检查一下
        if (self.icon_theme.lookup_icon("gnome-spinner", -1, 0)):
            # 如果存在“gnome-spinner”图标，则将其设置为默认动画图标名称
            self.default_animated_icon_name = "gnome-spinner"
        else:
            # 否则将默认动画图标名称设置为 None
            self.default_animated_icon_name = None
    # 通过在图标主题上进行查找来加载动画图标
    def load_animated_from_lookup(self, icon_name=None):
        """Loads an animated icon by doing a lookup on the icon theme."""

        # 如果用户没有选择图标名称，则使用默认图标名称
        if icon_name is None:
            icon_name = self.default_animated_icon_name

        # 即使默认图标（现在在icon_name上）可能不可用
        if icon_name is None:
            raise AnimatedIconNotFound

        # 尝试查找图标
        icon_info = self.icon_theme.lookup_icon(icon_name, -1, 0)
        # 即使icon_name存在，也可能找不到查找结果
        if icon_info is None:
            raise AnimatedIconNotFound

        # 基本大小是根据PyGTK文档：
        # "图标主题创建者指定的图标大小，这可能与图像的实际大小不同。"
        # 哎呀！我们在这里是盲目相信...
        size = icon_info.get_base_size()

        # 注意：如果图标是内置的，它将没有文件名，参见：
        # http://www.pygtk.org/pygtk2reference/class-gtkicontheme.html
        # 但是，我们没有使用gtk.ICON_LOOKUP_USE_BUILTIN标志，GTK+也没有内置动画，所以我们是安全的 ;-)
        filename = icon_info.get_filename()

        # 现在我们有了一个文件名，调用load_animated_from_filename()
        self.load_animated_from_filename(filename, size)

    def load_animated_from_filename(self, filename, size):
        # grid_pixbuf是一个包含整个图像的pixbuf
        grid_pixbuf = GdkPixbuf.Pixbuf.new_from_file(filename)
        grid_width = grid_pixbuf.get_width()
        grid_height = grid_pixbuf.get_height()

        for x in range(0, grid_width, size):
            for y in range(0, grid_height, size):
                self.spinner_images.add_animated_pixbuf(
                        self.__extract_frame(grid_pixbuf, x, y, size, size))
    # 从图标主题中查找指定图标的信息
    def load_static_from_lookup(self, icon_name="gnome-spinner-rest",
                                key_name=None):
        # 查找指定图标的信息
        icon_info = self.icon_theme.lookup_icon(icon_name, -1, 0)
        # 获取图标文件名
        filename = icon_info.get_filename()

        # 现在我们有了文件名，调用load_static_from_filename()方法
        self.load_static_from_filename(filename)

    # 从文件名加载静态图像
    def load_static_from_filename(self, filename, key_name=None):
        # 从文件创建图像像素缓冲
        icon_pixbuf = GdkPixbuf.Pixbuf.new_from_file(filename)

        # 如果键名为空，则使用文件名的第一个部分作为键名
        if key_name is None:
            key_name = filename.split(".")[0]

        # 将静态图像添加到静态图像字典中
        self.spinner_images.add_static_pixbuf(key_name, icon_pixbuf)

    # 提取图像的子图像，通常是动画的一帧
    def __extract_frame(self, pixbuf, x, y, w, h):
        """Cuts a sub pixbuffer, usually a frame of an animation.

        - pixbuf is the complete pixbuf, from which a frame will be cut off
        - x/y are the position
        - w (width) is the is the number of pixels to move right
        - h (height) is the is the number of pixels to move down
        """
        # 如果要提取的图像超出了原图像的范围，则抛出异常
        if (x + w > pixbuf.get_width()) or (y + h > pixbuf.get_height()):
            raise PixbufSmallerThanRequiredError
        # 返回提取的子图像
        return pixbuf.subpixbuf(x, y, w, h)

    # 将动画图像写入文件
    def _write_animated_pixbuf_to_files(self, path_format, image_format):
        """Writes image files from self.spinner_images.animated_pixbufs

        - path_format should be a format string with one occurrence of a
          string substitution, such as '/tmp/animation_%s.png'
        - image_format can be either 'png' or 'jpeg'
        """
        counter = 0
        # 遍历动画图像列表
        for i in self.spinner_images.animated_pixbufs:
            # 将图像保存为文件
            i.save(path_format % counter, "png")
            counter += 1

    # 将静态图像写入文件
    def _write_static_pixbuf_to_file(self, key_name, path_name, image_format):
        # 将静态图像保存为文件
        self.spinner_images.static_pixbufs[key_name].save(path_name,
                                                          image_format)
class HIGSpinner(Gtk.EventBox):
    """Simple spinner, such as the one found in webbrowsers and file managers.

    You can construct it with the optional parameters:
    * images, a list of images that will make up the animation
    * width, the width that will be set for the images
    * height, the height that will be set for the images
    """

    #__gsignals__ = {'expose-event': 'override',
    #                'size-request': 'override'}

    def __init__(self):
        # 调用父类的构造函数
        Gtk.EventBox.__init__(self)

        #self.set_events(self.get_events())

        # This holds a GDK Graphic Context
        # 初始化一个 GDK 图形上下文
        self.gc = None

        # These are sane defaults, but should really come from the images
        # 这些是合理的默认值，但实际上应该来自图像
        self.images_width = 32
        self.images_height = 32

        # Timeout set to 100 milliseconds per frame, just as the
        # Nautilus/Epiphany implementation
        # 设置每帧的超时时间为100毫秒，与 Nautilus/Epiphany 实现一致
        self.timeout = 120

        # Initialize a cache for ourselves
        # 初始化一个缓存对象
        self.cache = HIGSpinnerCache()
        self.cache.load_static_from_lookup()
        self.cache.load_animated_from_lookup()

        # timer_task it the gobject.timeout_add identifier (when the animation
        # is in progress, and __bump_frame is being continually called). If the
        # spinner is static, timer_task is 0
        # timer_task 是 gobject.timeout_add 的标识符（当动画正在进行时，__bump_frame 被不断调用）。如果 spinner 是静态的，timer_task 为0
        self.timer_task = 0
        # animated_pixbuf_index is a index on
        # animated_pixbuf_index 是一个索引
        self.animated_pixbuf_index = 0
        # current_pixbuf is initially the default rest_pixbuf
        # current_pixbuf 最初是默认的 rest_pixbuf
        self.current_pixbuf = self.cache.spinner_images.rest_pixbuf
    # 这个函数将动画帧移动到下一个帧，如果当前是最后一个帧，则返回到第一个帧
    def __bump_frame(self):
        # 获取动画帧列表
        animated_list = self.cache.spinner_images.animated_pixbufs
        # 如果当前帧是最后一个帧，则返回到第一个帧
        if self.animated_pixbuf_index == (len(animated_list) - 1):
            self.animated_pixbuf_index = 0
        else:
            # 否则，移动到下一个帧
            self.animated_pixbuf_index += 1

        # 重新绘制窗口
        self.queue_draw()
        return True

    # 这个函数根据 timer_task 的状态选择静态图像或动画帧
    def __select_pixbuf(self):
        # 如果 timer_task 为 0，则选择静态图像
        if self.timer_task == 0:
            self.current_pixbuf = self.cache.spinner_images.rest_pixbuf
        else:
            # 否则，选择当前动画帧
            self.current_pixbuf = self.cache.spinner_images.animated_pixbufs[
                    self.animated_pixbuf_index]

    # 启动动画
    def start(self):
        # 如果 timer_task 为 0，则添加定时任务来切换帧
        if self.timer_task == 0:
            self.timer_task = GLib.timeout_add(self.timeout,
                                                  self.__bump_frame)

    # 暂停动画
    def pause(self):
        # 如果 timer_task 不为 0，则移除定时任务
        if self.timer_task != 0:
            GLib.source_remove(self.timer_task)

        # 重置 timer_task 为 0，并重新绘制窗口
        self.timer_task = 0
        self.queue_draw()

    # 停止动画
    def stop(self):
        # 停止动画，同时将动画帧索引重置为 0
        self.pause()
        self.animated_pixbuf_index = 0

    # 设置动画速度
    def set_speed(speed_in_milliseconds):
        # 设置定时器的时间间隔，并暂停动画，然后重新启动动画
        self.timeout = speed_in_milliseconds
        self.pause()
        self.start()
    # 处理窗口曝光事件，event为事件对象
    def do_expose_event(self, event):
        # 调用chain方法，处理事件
        #self.chain(event)

        # 如果缓存中的spinner_images的rest_pixbuf为空，则抛出RestPixbufNotFound异常
        if self.cache.spinner_images.rest_pixbuf is None:
            raise RestPixbufNotFound

        # 选择当前的pixbuf
        self.__select_pixbuf()

        # 获取当前pixbuf的宽度和高度
        width = self.current_pixbuf.get_width()
        height = self.current_pixbuf.get_height()

        # 计算x和y的偏移量，使得pixbuf居中显示在allocation中
        x_offset = (self.allocation.width - width) // 2
        y_offset = (self.allocation.height - height) // 2

        # 创建一个Gdk.Rectangle对象，表示pixbuf在窗口中的位置和大小
        pix_area = Gdk.Rectangle(x_offset + self.allocation.x,
                                 y_offset + self.allocation.y,
                                 width, height)

        # 计算pix_area和event.area的交集，得到最终需要绘制的区域
        dest = event.area.intersect(pix_area)
# 如果图形上下文还不存在，则创建一个
if self.gc is None:
    self.gc = gtk.gdk.GC(self.window)
# 使用窗口创建 cairo 上下文
cairo = self.window.cairo_create()

# 在窗口上绘制 pixbuf
self.window.draw_pixbuf(self.gc,
                        self.current_pixbuf,
                        dest.x - x_offset - self.allocation.x,
                        dest.y - y_offset - self.allocation.y,
                        dest.x, dest.y,
                        dest.width, dest.height)

# 定义 size_request 方法，设置组件的大小需求
def do_size_request(self, requisition):
    # 设置组件的宽度需求为缓存中 spinner_images 的图片宽度
    requisition.width = self.cache.spinner_images.images_width
    # 设置组件的高度需求为缓存中 spinner_images 的图片高度
    requisition.height = self.cache.spinner_images.images_height
```