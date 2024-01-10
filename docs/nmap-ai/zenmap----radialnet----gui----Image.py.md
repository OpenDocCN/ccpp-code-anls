# `nmap\zenmap\radialnet\gui\Image.py`

```
# 设置文件编码为 UTF-8

# 重要提示：NMAP 许可证条款
# Nmap 安全扫描器由 Nmap 软件有限责任公司（“Nmap 项目”）（C）1996-2023 年开发。Nmap 也是 Nmap 项目的注册商标。
# 本程序根据 Nmap 公共源代码许可证（NPSL）的条款分发。适用于特定 Nmap 版本或源代码控制修订版的确切许可证文本包含在该版本或源代码控制修订版的 LICENSE 文件中。有关更多 Nmap 版权/法律信息，请访问 https://nmap.org/book/man-legal.html，有关 NPSL 许可证本身的更多信息，请访问 https://nmap.org/npsl/。本标题总结了 Nmap 许可证的一些关键点，但不能替代实际的许可证文本。
# Nmap 通常允许最终用户自行下载和使用，包括商业用途。可从 https://nmap.org 获取。
# Nmap 许可证通常禁止公司在商业产品中使用和重新分发 Nmap，但我们销售一个特殊的 Nmap OEM 版本，具有更宽松的许可证和专门用于此目的的特殊功能。请参阅 https://nmap.org/oem/
# 如果您收到了书面的 Nmap 许可协议或合同，其中规定了与这些条款（例如 Nmap OEM 许可证）不同的条款，您可以选择根据那些条款使用和重新分发 Nmap。
# 官方的 Nmap Windows 构建包括 Npcap 软件（https://npcap.com）用于数据包捕获和传输。它受到禁止未经特殊许可证重新分发的单独许可证条款的约束。因此，官方的 Nmap Windows 构建可能不得未经特殊许可证（例如 Nmap OEM 许可证）重新分发。
# 我们提供此软件的源代码，因为我们相信用户有权根据自己的需求修改和重新分发它。但是，我们希望用户尊重我们的劳动成果，遵守我们的许可证条款。
# 导入gi库，确保使用的是Gtk 3.0版本
import gi

gi.require_version("Gtk", "3.0")
# 从gi.repository中导入GdkPixbuf模块
from gi.repository import GdkPixbuf

# 导入os和array模块
import os
import array

# 从zenmapCore.Paths中导入Path
from zenmapCore.Paths import Path

# 定义RGBA和RGB格式的常量
FORMAT_RGBA = 4
FORMAT_RGB = 3

# 定义一个函数，用于获取cairo图像表面的像素
def get_pixels_for_cairo_image_surface(pixbuf):
    """
    This method return the image stride and a python array.ArrayType
    containing the icon pixels of a gtk.gdk.Pixbuf that can be used by
    cairo.ImageSurface.create_for_data() method.
    """
    # 创建一个字节类型的数组
    data = array.array('B')
    # 获取图像的格式
    image_format = pixbuf.get_rowstride() // pixbuf.get_width()

    # 初始化变量i和j
    i = 0
    j = 0
    # 当 i 小于像素缓冲区中像素数量时，执行循环
    while i < len(pixbuf.get_pixels()):
    
        # 从像素缓冲区中获取像素的蓝色、绿色和红色值
        b, g, r = pixbuf.get_pixels()[i:i + FORMAT_RGB]
    
        # 如果图像格式为 RGBA，则获取像素的 alpha 值
        if image_format == FORMAT_RGBA:
            a = pixbuf.get_pixels()[i + FORMAT_RGBA - 1]
        # 如果图像格式为 RGB，则设置 alpha 值为 255
        elif image_format == FORMAT_RGB:
            a = 255
        # 如果图像格式不是 RGBA 或 RGB，则抛出类型错误异常
        else:
            raise TypeError('unknown image format')
    
        # 将像素的颜色值和 alpha 值组成数组，并存入数据中
        data[j:j + FORMAT_RGBA] = array.array('B', [r, g, b, a])
    
        # 更新 i 和 j 的值
        i += image_format
        j += FORMAT_RGBA
    
    # 返回图像的宽度和数据
    return (FORMAT_RGBA * pixbuf.get_width(), data)
class Image:
    """
    图像类
    """
    def __init__(self, path=None):
        """
        初始化方法，设置路径和缓存字典
        """
        self.__path = path
        self.__cache = dict()

    def set_path(self, path):
        """
        设置路径的方法
        """
        self.__path = path

    def get_pixbuf(self, icon, image_type='png'):
        """
        获取像素缓冲区的方法
        """
        if self.__path is None:
            return False

        if icon + image_type not in self.__cache.keys():
            file = self.get_icon(icon, image_type)
            self.__cache[icon + image_type] = \
                    GdkPixbuf.Pixbuf.new_from_file(file)

        return self.__cache[icon + image_type]

    def get_icon(self, icon, image_type='png'):
        """
        获取图标的方法
        """
        if self.__path is None:
            return False

        return os.path.join(self.__path, icon + "." + image_type)


class Pixmaps(Image):
    """
    图像类的子类，用于处理像素图
    """
    def __init__(self):
        """
        初始化方法，设置路径为像素图目录下的radialnet
        """
        Image.__init__(self, os.path.join(Path.pixmaps_dir, "radialnet"))


class Icons(Image):
    """
    图像类的子类，用于处理图标
    """
    def __init__(self):
        """
        初始化方法，设置路径为像素图目录下的radialnet
        """
        Image.__init__(self, os.path.join(Path.pixmaps_dir, "radialnet"))


class Application(Image):
    """
    图像类的子类，用于处理应用程序图标
    """
    def __init__(self):
        """
        初始化方法，设置路径为像素图目录下的radialnet
        """
        Image.__init__(self, os.path.join(Path.pixmaps_dir, "radialnet"))
```